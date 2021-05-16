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
tuples easier to write and compose better with the rest of the language. The
specific list of proposed changes are:

a. The addition of sugar for declaring large homogenous tuples. (1), (2)
b. When parsing tuple types, stashing a bit in the tuple type that states if we
   parsed the tuple as a homogenous tuple to allow for improved type checking
   performance and to enable printing of large homogenous tuples with the sugar
   defined by (a). (1), (2)
c. Adding new initializers for homogenous tuples:
   a. A repeating initializer that initializes all elements of the tuple to the same value. (1)
   b. An unsafe uninitialized memory based initializer similar to Array's. (1)
d. Adding `RandomAccessCollection` and `MutableCollection` conformances to enable usage
   as a collection and accessing as contiguous storage. (1) (2)
e. Changing the Clang Importer to import fixed size arrays as sugared homogenous
   tuples and remove the arbitrary limitation on the number of elements (4096
   elements) that an imported fixed size array can have now that type checking
   homogenous tuples is fast. (2)
f. Eliminating the need to use unsafe type pointer punning to pass imported C
   fixed size arrays to related imported C APIs. (2)

NOTE: (1), (2) is used to notate which use case a specific alpha-numeric list
element above corresponds to.

NOTE: This proposal is specifically not attempting to implement a fixed size
"Swifty" array for all Swift programmers. Instead, we are attempting to extend
Swift in a minimal, composable way that helps system/stdlib programmers get
their job done today in the context where these tuples are already used today.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

Today in Swift, system programmers use homogenous tuples to represent a fixed
buffer of bytes of a certain type. An example of this is the SmallString
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
tuple elements increase, using homogenous tuples in this manner does not scale
from a usability and compile time perspective. We explore these difficulties
below:

### Problem 1: Large homogenous tuples result in linear type checking behavior

The first issue that naturally comes up is type checker performance. Today, when
performing type checking, the type checker must type check each tuple element
specifically to check properties such as if every element of a tuple obeys a
protocol when inferring if a tuple type is Comparable or Hashable due to
[SE-0283](0283-tuples-are-equatable-comparable-hashable.md). This can make large
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

If we want to define an initializer that initializes this tuple to nil, we
either have to use a Swift source generator or use unsafe code:

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
generator or use unsafe code,

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
Swift, the for-in loop and instead must iterate by using an index range and
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

In all of these cases, the lack of language support add unnecessary complexity
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
   programmer: we have made the problem more difficult than it need to be by
   forcing the expression of redundant information in the language.

2. Imported fixed size arrays can not be concisely iterated over due to
   homogenous tuples lacking a Collection conformance. This makes an operation
   that is easy to write in C much harder to perform in Swift since one must
   define a nominal type wrapper and use one of the techniques above from our
   PointerCache example. As a quick reminder using our imported `MyData_t`, this
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

This is just type sugar that is expanded before type checking, so beyond the
homogenous bit that we set in the tuple type itself, the rest of the compiler
just sees a normal tuple and thus will be minimally effected.

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

### Type Checker: Use homogenous tuple bit to decrease time needed to TypeCheck homogenous tuples

Today the type checker has to perform a linear amount of work when type checking
homogenous tuples. This is exascerbated by the comparable/equatable/hashable
conformances added in
[SE-0283](0283-tuples-are-equatable-comparable-hashable.md). We can eliminate
this for large homogenous tuples since when we typecheck we will be able to
infer that all elements of a tuple that has the homogenous tuple bit set is the
same, allowing the type checker can just type check the first element of the
tuple type and use only that for type checking purposes.

### Standard Library/Runtime Improvements:

We propose the following changes to the stdlib/runtime:

1. Add builtin `RandomAccessCollection` and `MutableCollection` Collection conformance. These will be implemented in the same manner as [SE-0283](0283-tuples-are-equatable-comparable-hashable.md):

2. Add the below init helpers for initializing homogenous tuples. These could be implemented 
  * `init(repeating: repeatedValue: Element)`: This will allow for users to easily initialize a
    large tuple all with the same value such as nil or a sentinel value:
    ```swift
    let x = (128 x Int)(repeating: 0)
    let y = (128 x UnsafePointer<MyDataType>)(repeating: sentinelValue)
    ```
    NOTE: Since the actual number of elements in the homogenous tuple is fixed,
    we do not need to pass in the count.
  
  * `init(initializingWith initializer: (UnsafeMutableBufferPointer<Element>, inout Optional<Int>) throws -> Void) rethrows`:
    This method allows for one to either initialize all elements of a tuple with
    pre-known values avoiding the need to first zero initialize such a tuple:
  
    ```swift
    // Fill tuple with integral data.
    let x = (1024 x Int) { (buffer: UnsafeMutableBufferPointer<Int>, count: inout Int) in
      for i in 0..<1024 { x[i] = i }
      count = 1024
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
  
  * `init(unsafeUninitializedCapacity: Int, initializingWith initializer: (inout UnsafeMutableBufferPointer<Element>, inout Int) throws -> Void) rethrows`:

     This method allows for one to initialize all elements of a tuple with
     pre-known values, placing the number of elements actually written to in
     count. After returning, the routine will fill the remaining uninitialized
     memory with a bad pointer pattern and when asan is enabled will poison the
     memory. We will follow the same condition's of [Array's version of this method](https://developer.apple.com/documentation/swift/array/3200717-init)
     around the behavior of the initializedCount parameter. Consider the
     following examples below of this initializer in action:

     ```swift
     // Fill tuple with integral data.
     let x = (1024 x Int) { (buffer: UnsafeMutableBufferPointer<Int>, initializedCount: inout Int) in
       for i in 0..<1000 { x[i] = i }
       initializedCount = 1000
     }
     precondition(x[1001] == 0xDEADBEEF, "Beef!") // For arguments sake
     // More succintly:
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

### Clang Importer Changes:

We propose changing the Clang Importer so that it sets the homogenous bit on all
fixed size arrays that it imports as homogenous tuples. This will allow for all
of the benefits from the work above to apply to these types when used in Swift.

## Detailed design

In more detail, the specific chains we are suggesting are:

* Homogenous Tuple Span Parsing. Changing the tuple
  element grammar as follows:
  ```
  parseTypeTupleBody
    type-tuple:
      '(' type-tuple-body? ')'
    type-tuple-body:
      type-tuple-element (',' type-tuple-element)* '...'?
    type-tuple-element:
      identifier? identifier ':' type
      type
      type-homogenous-tuple-span
    type-homogenous-tuple-span:
      integer_literal 'x' type
    type-pure-homogenous-tuple:
      // with all tuple-span-elements having same base type.
      '(' type-homogenous-tuple-span (',' type-homogenous-tuple-span)* ')'
  ```
  
  In terms of implementation, we recognize the count and the set product symbol
  `x` when parsing tuple elements as signaling a homogenous tuple span
  element. For our purposes here, lets assume we are parsing such a homogenous
  tuple element of the form `N x Type`. To convert this into its AST form, we
  add `N` elements of `Type` to the current parent Tuple Type. We also while
  parsing maintain a bit if all elements of a tuple are the same type. This
  enables us to use this information to speed up type checking for homogenous
  tuples written with the new syntax or any tuples that use the old syntax.

* In order to change the 

* Homogenous Tuple Types and the ObjC Importer

  Once we have changed the parsing, 
  To change the manner in which we import tuple types, we simply change in
  ImportType.cpp SwiftTypeConverter::VisitConstantArrayType to create a new
  tuple type with the appropriate number of elements and set the is homogenous
  tuple bit.

* Homogenous Tuple Types and extra methods

  Options: Could do a protocol (like Collection). Or maybe something in Type
  Checker. Need to see where one can shim this in.

* Homogenous Tuple Types and Collection conformance

  We build upon the work in equatable, hashable for tuple

* Improved Tuple Type ABI? Could rely on 4096 limit? What one /could/ do is add
  new loadable thing that fuses them if larger than a certain size and right
  deployment target? 

## Source compatibility

Relative to the Swift 3 evolution process, the source compatibility
requirements for Swift 4 are *much* more stringent: we should only
break source compatibility if the Swift 3 constructs were actively
harmful in some way, the volume of affected Swift 3 code is relatively
small, and we can provide source compatibility (in Swift 3
compatibility mode) and migration.

Will existing correct Swift 3 or Swift 4 applications stop compiling
due to this change? Will applications still compile but produce
different behavior than they used to? If "yes" to either of these, is
it possible for the Swift 4 compiler to accept the old syntax in its
Swift 3 compatibility mode? Is it possible to automatically migrate
from the old syntax to the new syntax? Can Swift applications be
written in a common subset that works both with Swift 3 and Swift 4 to
aid in migration?

## Effect on ABI stability

Does the proposal change the ABI of existing language features? The
ABI comprises all aspects of the code generation model and interaction
with the Swift runtime, including such things as calling conventions,
the layout of data types, and the behavior of dynamic features in the
language (reflection, dynamic dispatch, dynamic casting via `as?`,
etc.). Purely syntactic changes rarely change existing ABI. Additive
features may extend the ABI but, unless they extend some fundamental
runtime behavior (such as the aforementioned dynamic features), they
won't change the existing ABI.

Features that don't change the existing ABI are considered out of
scope for [Swift 4 stage 1](README.md). However, additive features
that would reshape the standard library in a way that changes its ABI,
such as [where clauses for associated
types](https://github.com/apple/swift-evolution/blob/master/proposals/0142-associated-types-constraints.md),
can be in scope. If this proposal could be used to improve the
standard library in ways that would affect its ABI, describe them
here.

## Effect on API resilience

API resilience describes the changes one can make to a public API
without breaking its ABI. Does this proposal introduce features that
would become part of a public API? If so, what kinds of changes can be
made without breaking ABI? Can this feature be added/removed without
breaking ABI? For more information about the resilience model, see the
[library evolution
document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst)
in the Swift repository.

## Alternatives considered

The main alternative considered is whether or not to use a different syntax.

## Acknowledgments

Thanks to Joe Groff for suggesting going down this path!
