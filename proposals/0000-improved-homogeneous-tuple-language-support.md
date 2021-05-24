# Improved Language Support for Homogenous Tuples

* Proposal: [SE-NNNN](NNNN-improved-homogenous-tuple-language-support.md)
  * Authors: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Awaiting implementation**

* Reference: [SE-0283](0283-tuples-are-equatable-comparable-hashable.md)

## Introduction

Define a pure homogenous tuple of type `T` as a non-empty tuple that only
contains elements of type `T`, e.x.: `(T, T, ..., T)`. Swift System programmers
use these tuples in (non-exclusively) the following contexts:

1. Defining a Swift nominal type that uses a homogenous tuples to represent a field with a fixed layout of bytes (e.x.: SmallString)
2. Writing code that uses an imported C fixed size array in Swift.

There has historically not been much language support specifically for
homogenous tuples or even tuples in general. In this proposal, we attempt to
incrementally back fill such language support in a manner that makes homogenous
tuples easier to write and composes better with the rest of the language. The
specific proposed changes are:

+ The addition of the sugar for declaring large homogenous tuples. (1), (2).
+ When parsing tuple types, stashing a bit in the tuple type that states if we
   parsed the tuple as a homogenous tuple to allow for improved type checking
   performance and to enable printing of large homogenous tuples with the sugar
   defined by (a). (1), (2).
+ Adding new initializers for homogenous tuples to ease initialization of
  homogenous tuple fields.
+ Adding `RandomAccessCollection` and `MutableCollection` conformances to enable use
  of tuple as a collection and access them as contiguous storage. (1) (2)
+ Changing the Clang Importer to import fixed size arrays as sugared homogenous
   tuples and removing the arbitrary limitation on the number of elements (4096
   elements) that an imported fixed size array can have now that type checking
   homogenous tuples is fast. (2)
+ Eliminating the need to use unsafe type pointer punning to pass imported C
   fixed size arrays to related imported C APIs. (2)

NOTE: (1), (2) is used to notate which use case a specific alpha-numeric list
element above corresponds to.

NOTE: This proposal is specifically not attempting to implement a fixed size
"Swifty" array for all Swift programmers. Instead, we are attempting to extend
Swift in a minimal, composable way that helps system/stdlib programmers get
their job done in the context where these tuples are already used today.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

Today in Swift, system programmers use homogenous tuples to represent a fixed
buffer of bytes of a certain type. An example of this is the `SmallString`
implementation in Swift's standard library:

```swift
@frozen @usableFromInline
internal struct _SmallString {
  @usableFromInline
  internal var _storage: (UInt64, UInt64)
}

// Taken from:
// https://github.com/apple/swift/blob/833a453c8ad6e9982e849229d6f91532717cd354/stdlib/public/core/SmallString.swift#L33
```

By declaring `_SmallString` as frozen and `_storage` as a homogenous tuple of
type `UInt64`, we are able to guarantee that `_SmallString` when laid out in
memory is exactly 128 bits and can be treated as layout compatible with other
128 bit values, which is totally awesome! That being said, as the number of
tuple elements increases, using homogenous tuples in this manner does not scale
from a usability and compile time perspective. We explore these difficulties
below:

### Problem 1: Large homogenous tuples result in linear type checking behavior

The first issue that naturally comes up is type checker performance. Today, 
the type checker considers a tuple (as a whole) to be a sum of its parts, which 
means that to verify specific properties of a tuple e.g. if it is `Comparable`
or `Hashable` it has to verify that every element conforms to a required protocol
due to [SE-0283](0283-tuples-are-equatable-comparable-hashable.md). This can make large
homogenous tuples incur a significant type checking overhead when being
used. Tuple in it of themselves are expensive enough already to typecheck as
shown by the clang importer posessing an artifical limit of 4096 of the number
of elements of an importable C fixed size array.

### Problem 2: Basic operations on large homogenous tuples require use of a source generator or unsafe code

To explore the implications of the current homogenous tuple model, imagine that
we are defining a System API that wants to maintain a cache of 128 pointers to
improve performance. Since we need to have enough storage for 128 pointers, we
use a homogenous tuple of `UnsafeMutablePointer` that point at objects of type
`T`:

```swift
@frozen
struct PointerCache<T> {
  typealias PointerType = UnsafeMutablePointer<T>

  /// 128 pointers that we use to cache the first pointers that are passed
  /// to our API.
  var _storage: (PointerType?, /*126 more pointers*/, PointerType?)
}
```

If we want to define an initializer that initializes this tuple to `nil`, we
either have to use a Swift source generator or write unsafe code:

```swift
extension PointerCache {
  init(repeating value: PointerType) {
    _storage = (value, /* value 125 times */, value)
  }

  init(unsafe: ()) {
    withUnsafeMutableBytes(of: &_storage) { (buffer: UnsafeMutableRawBufferPointer) in
      for i in 0..<128 {
        buffer.storeBytes(of: nil, toByteOffset: i*MemoryLayout<PointerType?>.stride, as: PointerType?.self)
      }
    }
  }

  init(unsafe2: ()) {
    withUnsafeMutableBytes(of: &_storage) { (buffer: UnsafeMutableRawBufferPointer) in
      memset(buffer.baseAddress!, 0, 128*MemoryLayout<PointerType?>.stride)
    }
  }
}
```

If we want to define a Collection conformance for our type, we must wrap our
tuple in a nominal type and are forced to again use either a Swift source code
generator or write unsafe code:

```swift
extension PointerCache {
  subscript(index: Int) -> PointerType? {
    switch index {
    case 0:
      return _storage.0
    case 1:
      return _storage.1
    /* ... 125 more cases ... */
    case 127:
      return _storage.127
    default:
      fatalError("...")
  }
}

extension PointerCache {
  subscript(unsafe index: Int) -> PointerType? {
    withUnsafeBytes(of: x) { (buffer: UnsafeRawBufferPointer) -> PointerType? in
        buffer.load(fromByteOffset: index*MemoryLayout<PointerType?>.stride, as: PointerType?)
    }
  }
}
```

Even with this, we still are forced to avoid the natural manner of iterating in
Swift - the `for-in` loop, and instead must iterate by using an index range and
subscript into the type:

```swift
func printCache(_ p: PointerCache) {
  for i in 0..<1024 {
    print(p[i])
  }
}
// Instead of
func printCache(_ p : PointerCache) {
  for elt in p {
    print(elt)
  }
}
```

In all of these cases, the lack of language support adds unnecessary complexity
to the program for what should be simple primitive operations that should scale
naturally to larger types without needing us to use unsafe code. Even if we say
that the unsafe code example is ok, we would be relying on the optimizer to
eliminate overhead. Relying on the optimizer is ok in the large, but when system
programming praying is insufficient since the programmer must at compile time be
able to guarantee certain performance constraints. This generally forces the
programmer to look at the assembly to guarantee that the optimizer optimized the
unsafe code as the programmer expected, an unfortunate outcome.

### Problem 3: Imported C fixed size arrays do not compose well with other Swift language constructs

The Clang Importer today imports C fixed size arrays into Swift as homogenous
tuples. For instance, the following C:

```c
typedef struct {
  float dataBuffer[1024];
} MyData_t;
void printFloatData(const float *buffer);
```

would be imported as:

```swift
struct MyData_t {
  dataBuffer: (Float, ... /* 1022 Floats */, ..., Float)
}
void printFloatData(UnsafePointer<Float>);
```

The lack of language support for tuples results in these imported arrays not
being as easy to work with as they should be:

1. The ClangImporter due to the bad type checker performance of large homogenous
   tuples will not import a fixed size array if the array would be imported as a
   tuple with more than 4096 elements (see
   [ImportType.cpp](https://github.com/apple/swift/blob/e91b305b940362238c0b63b27fd3cccdbecadbaa/lib/ClangImporter/ImportType.cpp#L571)). At
   a high level, the type checker is running into the same problem of the
   programmer: we have made the problem more difficult than it needs to be by
   forcing the expression of redundant information in the language.

2. Imported fixed size arrays can not be concisely iterated over due to
   homogenous tuples lacking a `Collection` conformance. This makes an operation
   that is easy to write in C much harder to perform in Swift since one must
   define a nominal type wrapper and use one of the techniques above from our
   `PointerCache` example. As a quick reminder using our imported `MyData_t`, this
   is how we could use unsafe code to write our print method:

   ```swift
   extension MyData_t {
     func print() {
       withUnsafeBytes(of: dataBuffer) { (buffer: UnsafeRawBufferPointer) -> PointerType? in
         for i in 0..<1024 {
            let f = buffer.load(fromByteOffset: i*MemoryLayout<Float>.stride, as: Float)
            print(f)
         }
       }
     }
   }
   ```

3. Imported fixed size arrays from C no longer compose with associated C apis
   when working with C code in Swift. As an example, if we wanted to use the
   imported C API `printFloatData` with our C imported type, we would be unable
   to write the natural code due to the types not lining up:
   ```swift
      extension MyData_t {
        mutating func cPrint1() {
          // We get a type error here since printFloatData accepts an
          // UnsafePointer<Float> as its first argument and
          // '&dataBuffer' is an UnsafePointer<(Float, ..., Float)>.
          printFloatData(&dataBuffer) // Error! Types don't line up!
        }
      }
   ```
   but instead must write the following verbose code to satisfy the type checker:
   ```swift
     extension MyData_t {
       func cPrint() {
         withUnsafeBytes(of: x?.dataBuffer) {
           printFloatData($0.baseAddress!.assumingMemoryBound(to: Float.self))
         }
       }
     }
   ```

## Proposed solution

In order to make the life easier for System Programmers, we suggest adding the
following language support:

### Syntax Change: Homogenous Tuple Type Sugar

We propose a new sugar syntax for a "homogenous tuple span" element. The
grammar of tuple elements in Swift would be extended as follows:

```swift
(5 x Int) -> (Int, Int, Int, Int, Int)
```

This sugar is expanded before type checking, so beyond the homogenous bit that
we set in the tuple type itself, the rest of the compiler just sees a normal
tuple and thus will be minimally effected.

As an additional unnecessary extension, a homogenous tuple span element can be mix/matched
with other elements to create more complex layout compatible data structures,
e.x.:
```swift
(Float, 5 x Int, String, 2 x AnyObject)
// --> expands to
(Float, Int, Int, Int, Int, Int, String, AnyObject, AnyObject)
```

This capability is not integral to the proposal and if necessary can be sliced off and
we can allow only for tuples to only have a single homogenous tuple element.

### Type Checker: Use homogenous tuple bit to decrease time needed to type-check homogenous tuples

Today the type checker has to perform a linear amount of work when type checking
homogenous tuples. This is exascerbated by the comparable/equatable/hashable
conformances added in
[SE-0283](0283-tuples-are-equatable-comparable-hashable.md). We can eliminate
this for large homogenous tuples since when we typecheck we will be able to
infer that all elements of a tuple that has the homogenous tuple bit set are the
same, allowing the type checker to just type-check the first element of the
tuple type and skip the rest.

### Standard Library/Runtime Improvements:

We propose the following changes to the stdlib/runtime:

1. Add builtin `RandomAccessCollection` and `MutableCollection` Collection conformance. These will be implemented in the same manner as [SE-0283](0283-tuples-are-equatable-comparable-hashable.md):

2. Add init helpers for initializing homogenous tuple memory:
  * `init(repeating: repeatedValue: Element)`: This will allow for users to easily initialize a
    large tuple all with the same value such as nil or a sentinel value:
    ```swift
    let x = (128 x Int)(repeating: 0)
    let y = (128 x UnsafePointer<MyDataType>)(repeating: sentinelValue)
    ```
    NOTE: Since the actual number of elements in the homogenous tuple is fixed,
    we do not need to pass in the count.

  * `init(initializingWith initializer: (UnsafeMutableBufferPointer<Element>) throws -> Void) rethrows`:
    This method allows for one to either initialize all elements of a tuple with
    pre-known values avoiding the need to first zero initialize such a tuple:

    ```swift
    // Fill tuple with integral data.
    let x = (1024 x Int) { (buffer: UnsafeMutableBufferPointer<Int>) in
      for i in 0..<1024 { x[i] = i }
    }
    // Or more succintly:
    let x = (1024 x Int) { for i in 0..<1024 { $0[i] = i } }

    // memcpy data from a data buffer into a tuple after being called via a callback from C.
    var _cache: (1024 x Int) = ...
    func callBackForC(_ src: UnsafeMutableBufferPointer<Int>) {
      precondition(src.size == 1024)
      _cache = (1024 x Int) { dst: UnsafeMutableBufferPointer<Int> in
        memcpy(dst.baseAddress!, src.baseAddress!, src.size * MemoryLayout<Int>.stride)
      }
    }
    ```

    The user in this case is stating that they will initialize all elements of
    the buffer pointer and it is undefined behavior to not do so.

  * `init(unsafeUninitializedCapacity: Int, initializingWith initializer: (inout UnsafeMutableBufferPointer<Element>, inout Int) throws -> Void) rethrows`:

     This method allows to initialize all elements of a tuple with
     pre-known values, placing the number of elements actually written to in
     count. After returning, the routine will fill the remaining uninitialized
     memory with either a zero fill or a bad pointer pattern and when asan is enabled will poison the
     memory. We will follow the same condition's of [Array's version of this method](https://developer.apple.com/documentation/swift/array/3200717-init)
     around the behavior of the initializedCount parameter. Consider the
     following examples below of this initializer in action:

     ```swift
     // Fill tuple with integral data.
     let x = (1024 x Int) { (buffer: UnsafeMutableBufferPointer<Int>, initializedCount: inout Int) in
       for i in 0..<1000 { x[i] = i }
       initializedCount = 1000
     }
     precondition(x[1001] == 0, "Zero init!") // For arguments sake
     ```

### Clang Importer Changes:

We propose changing the Clang Importer so that it sets the homogenous bit on all
fixed size arrays that it imports as homogenous tuples. This will allow for all
of the benefits from the work above to apply to these types when used in Swift.

## Detailed design

In more detail, the specific changes we are suggesting are:

* Changing the tuple element grammar as follows:

  ```
    set-product: 'x'
    type-tuple:
      '(' type-tuple-body? ')'
    type-tuple-body:
      type-tuple-element (',' type-tuple-element)* '...'?
    type-tuple-element:
      identifier? identifier ':' type
      type
      type-homogenous-tuple-span
    type-homogenous-tuple-span:
      integer_literal set-product type
    type-pure-homogenous-tuple:
      // with all tuple-span-elements having same base type.
      '(' type-homogenous-tuple-span (',' type-homogenous-tuple-span)* ')'
  ```

  In terms of implementation, we recognize we represent count and the set
  product symbol `x` when parsing tuple elements as signaling a homogenous tuple
  span element. For our purposes here, let's assume we are parsing such a
  homogenous tuple element of the form `N x Type`. To convert this into its AST
  form, we add `N` elements of `Type` to the current parent Tuple Type. We also
  while parsing maintain a bit if all elements of a tuple are the same
  type. This enables us to use this information to speed up type checking for
  homogenous tuples written with the new syntax or any tuples that use the old
  syntax.

* We will implement the initializers by introducing new routines in the runtime
  to implement the initializers and teaching name lookup in the type checker how
  to resolve the methods for our tuples.

* The Clang Importer will be modified so that it sets the "homogenous tuple" bit on
  an imported fixed size array. This will ensure that imported tuples are printed out
  in homogenous form and will allow us to lift the 4096 element limit
  since we will no longer have type checker performance issues.

* We will implement `RandomAccessCollection` and `MutableCollection`
  conformances for tuples by building upon the builtin conformance work in
  [SE-0283](0283-tuples-are-equatable-comparable-hashable.md).

* In order to improve the performance of Tuples, we break the tuple ABI for
  large tuples and no longer expand large tuples into multiple arguments/return
  values. Noting that most ABIs are able to pass up to 8 arguments in registers
  without needing to spill, we either should make the change for tuples with N >
  8 if we are ok with binary functions of smaller tuples having an inefficient
  ABI. If we are worried about binary functions, we instead could instead choose
  N > 4 which would then ensure that binary functions of tuples with size 4 or
  less are passed in registers and those larger will be passed by pointer giving
  a more efficient ABI.

## Source compatibility

This is purely additive from a source perspective. The only possible concern is
from the ClangImporter changing how it prints such headers. This is not actually
a concern since this is not used by any Swift programs for compilation as an
artifact. Rather the ClangImporter in this case is instead only intended to show
programmers what the ClangImporter imported.

## Effect on ABI stability

The only effect on ABI stability will be changing the calling convention of
large tuples. The authors believe that the passing of large tuples (given
sufficiently large N) is rare enough (due to bad performance) that breaking ABI
this way will not effect large bodies of code and is worth it to improve perf in
the language.

## Effect on API resilience

This is additive at the source level so should not effect API resilience.

## Alternatives considered

The main alternative approach considered is the introduction of a new fixed size
array type rather than extending the tuple type grammar. For the purpose of this
proposal, we call such a type a `NewFixedSizeArray` and use the following syntax
as a straw man: `[5 x Int]`. The main advantage of using `NewFixedSizeArray` is
that it allows us to avoid breaking ABI for large tuples since we are just
introducing new ABI instead. That being said, we trade the lack of an ABI break
for the following problems:

1. The language surface area is being expanded to add a new type/concept rather
   than building on something that already exists in the language. This is a
   real concern given that the language is already relatively large and this is
   another concept for Swift programmers to learn.

2. Since we are introducing a whole new type into the language and into the
   compiler, we will need to update a far larger part of the compiler to add
   this type and additionally will need to do work all over the optimizer to
   ensure that we can handle these new types. That being said, we /may/ be able
   to build on top of `alloc_box`/`alloc_stack` and thus avoid needing to touch
   the optimizer.

3. Any code written assuming that the ClangImporter imports fixed size arrays as
   homogenous tuples will either need to be modified (a source compatibility
   break) or we will have to ensure that where-ever one could use one of those
   tuples we can pass a NewFixedSizeArray. In practice this means adding a new
   implicit conversion and adding support in the type checker for accessing
   elements of NewFixedSizeArray like a tuple using `x.0`, `x.1` syntax.

## Acknowledgments

Thanks to Joe Groff for suggesting going down this path! Also thanks to Andrew
Trick, Michael Ilseman, David Smith, Tim Kientzle, John McCall, Doug Gregor, and
many others for feedback on this!
