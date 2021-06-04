# Improved Language Support for Homogeneous Tuples

* Proposal: [SE-NNNN](NNNN-improved-homogenous-tuple-language-support.md)
* Author: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Awaiting implementation**

* Reference: [SE-0283](0283-tuples-are-equatable-comparable-hashable.md)
* Pitch Thread 1: [https://forums.swift.org/t/pitch-improved-compiler-support-for-large-homogenous-tuple](https://forums.swift.org/t/pitch-improved-compiler-support-for-large-homogenous-tuple)

## Introduction

Swift programmers frequently encounter homogenous tuples (e.x.: `(T, T, ...,
T)`) in contexts such as:

1. Writing code that uses an imported C fixed size array in Swift.
2. Defining a Swift nominal type that uses a homogenous tuples to represent a
   field with a fixed layout of bytes (e.x.: SmallString)

In this proposal, we attempt to improve language support for such tuples in a
manner that makes homogenous tuples easier to write and compose better with the
rest of the language. The specific list of proposed changes are:

+ The addition of sugar for declaring large homogenous tuples.
+ Introducing a new `HomogeneousTuple` protocol. This protocol will
  extend `RandomAccessCollection` and `MutableCollection` allowing for
  homogenous tuples to be used as collections and access contiguous storage. It
  will also provide a place for us to declare new helper init methods for
  initializing homogenous tuple memory, allow us to expose the count of a tuple
  type, and also allow us to implement using default protocol implementations.
  We will only allow for homogenous tuples to conform to this, not user types.
+ Changing the Clang Importer to import fixed size arrays as sugared homogenous
  tuples and remove the arbitrary limitation on the number of elements (4096
  elements) that an imported fixed size array can have now that type checking
  homogenous tuples is fast.
+ Eliminating the need to use unsafe type pointer punning to pass imported C
  fixed size arrays to related imported C APIs.
+ Changing the Swift calling convention ABI of sufficiently large tuples to
  allow for them to be passed as an aggregate rather than aggressively
  destructuring the tuple into individual arguments.

NOTE: This proposal is specifically not attempting to implement a fixed size
"Swifty" array for all Swift programmers. Instead, we are attempting to extend
Swift in a minimal, composable way that helps system/stdlib programmers get
their job done today in the context where these tuples are already used today.

Swift-evolution thread: **Still a Pitch**.

## Motivation

Today in Swift, system programmers use homogenous tuples when working with
imported fixed size C-arrays and when defining a fixed buffer of bytes of a
certain type. As a quick example, consider the following C struct,

```swift
typedef struct {
  int versionDigits[3];
} MyVersionInfo_t;
```

This would be imported as the following struct in Swift,

```swift
struct MyVersionInfo_t {
  var versionDigits: (Int, Int, Int)
}
```

This works in the small for such small fixed size arrays but as the number of
tuple elements increase, using homogenous tuples in this manner does not scale
from a usability and compile time perspective. We explore these difficulties
below:

### Problem 1: Basic operations on large homogenous tuples require use of a source generator or unsafe code

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

### Problem 2: Imported C fixed size arrays do not compose well with other Swift language constructs

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

### Problem 3: Large homogenous tuples result in linear type checking behavior

[SE-0283](0283-tuples-are-equatable-comparable-hashable.md) while enabling
tuples to become equatable, hashable and comparable also introduced new code
paths that result in linear work by the type checker since the type checker
needs to prove that each individual tuple element obeys that protocol. This is
exascerbated by the length of homogenous tuples. Luckily, We can eliminate that
issue for homogenous tuples by taking advantage of the extra semantic
information we have from the syntax to just use the type of the first tuple
element to perform the type checking.

### Problem 4: Passing/returning large homogenous tuples is inefficient

Today Swift's ABI possesses a rule that all tuples are eagerly destructured at
all costs. This means that if one were to pass a 1024 element tuple as an
argument, the resulting function would have 1024 arguments resulting in poor
performance. This ABI is an artifact of Swift's early evolution where there was
an attempt to enable tuples to be passed as a function argument list. In such a
case, destructuring the tuple into arguments makes sense. Moving forward in
time, that functionality was eliminated from the language so now we are just
left with an ABI rule that is actively harmful to performance. This performance
cost is significant enough that users generally avoid large tuples.

## Proposed solution

### Syntax Change: Homogeneous Tuple Type Sugar

We propose a new sugar syntax for a "homogenous tuple span" element. The
grammar of tuple elements in Swift would be extended as follows:

```swift
(5 * Int) -> (Int, Int, Int, Int, Int)
```

This sugar is expanded before type checking, so beyond the homogenous bit that
we set in the tuple type itself, the rest of the compiler just sees a normal
tuple and thus will be minimally effected. We would restrict ones ability to
only use homogenous tuple spans in tuples that are otherwise single element
lists.

### Standard Library/Runtime Improvements:

We propose the addition of a new protocol called `HomogeneousTuple` that all
homogenous tuples implicitly conform to:

```swift
protocol HomogeneousTuple : RandomAccessCollection, MutableCollection
  where
    // NOTE: This is the default subsequence implementation. We are just making
    // it explicit in the proposal.
    SubSequence == Slice<Self>
  {

  /// Used to initialize a tuple with a repeating element. All elements of the tuple will be initialized.
  ///
  /// Example:
  ///
  /// let * = (128 * Int)(repeating: 0)
  /// let y = (128 * UnsafePointer<MyDataType>)(repeating: sentinelValue)
  init(repeating: repeatedValue: Element)

  /// Initialize all elements of a tuple using a callback that maps an index to
  /// the value the index must take.
  ///
  /// DISCUSSION: This entrypoint allows for the conformance to use in place
  /// initialization of the tuple memory.
  ///
  /// Example:
  ///
  /// // Fill tuple with integral data.
  /// let * = (1024 * Int) { (index: Index) -> Int in
  ///   return index
  /// }
  /// // Or more succintly:
  /// let * = (1024 * Int) { $0 }
  init(initializingWith initializer: (Index) throws -> Element) rethrows

  /// Initialize all elements of a tuple with pre-known values, placing the
  /// number of elements actually written to in count. After returning, the
  /// routine will fill the remaining uninitialized memory with either a zero
  /// fill or a bad pointer pattern and when asan is enabled will poison
  /// the memory. We will follow the same condition's of [Array's version of this method](https://developer.apple.com/documentation/swift/array/3200717-init)
  /// around the behavior of the initializedCount parameter.
  ///
  /// Example:
  ///
  ///   // Fill tuple with integral data.
  ///   let * = (1024 * Int) { (buffer: UnsafeMutableBufferPointer<Int>, initializedCount: inout Int) in
  ///     for i in 0..<1000 { x[i] = i }
  ///     initializedCount = 1000
  ///   }
  ///   precondition(x[1001] == 0, "Zero init!") // For arguments sake
  init(unsafeUninitializedCapacity: Int, initializingWith initializer: (inout UnsafeMutableBufferPointer<Element>, inout Int) throws -> Void) rethrows
}
```

### Clang Importer Changes:

We propose changing the Clang Importer so that it sets the homogenous bit on all
fixed size arrays that it imports as homogenous tuples. This will allow for all
of the benefits from the work above to apply to these types when used in Swift.

## Detailed design

### Grammar Change: Add support for homogenous tuple span

In order to prase homogenous tuple spans, we modify the swift grammar as follows:
```
  type-tuple:
    '(' type-tuple-body? ')'
  type-tuple-body:
    type-tuple-element (',' type-tuple-element)* '...'?
  type-tuple-element:
    identifier? identifier ':' type
    type
    type-homogenous-tuple-span
  integer_literal_gt_one:
    [2-9][0-9]*
  type-homogenous-tuple-span:
    integer_literal_gt_one '*' type
  type-pure-homogenous-tuple:
    '(' type-homogenous-tuple-span ')'
```

When we parse a tuple that consists of a single homogenous tuple span element,
we set in the tuple type a homogenous tuple bit using spare bits already
available in the tuple type.

NOTE: We specifically ban homogenous tuple spans that provide an integer literal
of 0 or 1. The reason for this is that given that Swift's generics do not
support variadics or type level integers, having homogeneous tuples of size 0, 1
are not useful. Instead we require our users to use an empty tuple or just an
element.

### Type Checker: Reduce number of Type variables for Homogeneous Tuples

The type checker today creates a type variable for all tuple elements when
solving constraints since any element of the tuple could be different. But if
the user provides us with the information ahead of time that they explicitly
have a homogenous tuple, we can introduce a join constraint on all of the tuple
elements allowing the constraint solver to be more efficient via the use of its
merging heuristic:

```
// We know that all of these must be the same type, so we can provide the type checker with that constraint info.
let x: (3 * Int) = (1, 2, 3)
```

### Type Checker: Use homogenous tuple bit to decrease type checking when comparing, equating, and hashing homogenous tuples

Today the type checker has to perform a linear amount of work when type checking
homogenous tuples. This is exascerbated by the comparable/equatable/hashable
conformances added in
[SE-0283](0283-tuples-are-equatable-comparable-hashable.md). We can eliminate
this for large homogenous tuples since when we typecheck we will be able to
infer that all elements of a tuple that has the homogenous tuple bit set is the
same, allowing the type checker can just type check the first element of the
tuple type and use only that for type checking purposes.

### HomogeneousTuple protocol

This will be implemented following along the lines of the
``ExpressibleBy*Literal`` protocols except that we will not allow for user
defined types to conform to this protocol. The only conformances are for types
with the homogenous tuple bit set preventing bad type checker performance. This
will additionally let us implement the protocol's functionality by using a
default implementation and using that default implementation for all
tuples. This default implementation in practice will just be a trampoline into
the c++ runtime.

### Clang Importer Changes

The Clang Importer will be modified so that we set the homogenous tuple bit on
imported fixed size arrays. This will ensure that we print out the imported
tuples in homogenous form and will allow us to lift the 4096 element limit since
we will no longer have type checker slow down issues.

### Improved Tuple ABI of Swift's Calling Convention

We propose that we change Swift's calling convention so that a function that
takes a homogenous tuple will have the new ABI. Easily, if we pass the
homogenous tuple to a function with the normal ABI we can just destructure it
and can provide a thunk for the old tuple ABI if we ever pass a non-homogenous
tuple as an argument and or result of a homogenous tuple API.

## Source compatibility

This is purely additive from a source perspective. The only possible concern is
from the ClangImporter changing how it prints such headers. This is not actually
a concern since this is not used by any Swift programs for compilation as an
artifact. Rather the ClangImporter in this case is instead only intended to show
programmers what the ClangImporter imported.

## Effect on ABI stability

The main effect on ABI stability would be additive since the change in tuple ABI
would only affect functions that have explicit homogenous tuples as arguments,
results and we can provide thunks when needed to convert in between the
representations.

## Effect on API resilience

This is additive at the source level so should not effect API resilience.

## Alternatives considered

The main alternative approach considered is the introduction of a new fixed size
array type rather than extending the tuple type grammar. For the purpose of this
proposal, we call such a type a NewFixedSizeArray and use the following syntax
as a straw man: `[5 * Int]`. The main advantage of using NewFixedSizeArray is
that it allows us to fix some ABI issues around tail padding (see footnote 1
below). That being said, we trade this ABI improvement for the following
problems:

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

Footnote 1: Tail Padding Issues. An additional ABI issue that a new fixed size
array type would fix is around tail padding. Specifically if one has a type with
tail padding due to alignment, then Swift considers its size (the number of
bytes used to store the data for the value) to be smaller than its stride (the
space in between consecutive elements in an array. This contrasts with the C ABI
where it is always assumed that size and stride are the same.

As an example, `MyState` in the following has a size of 9 but a stride of 16 b/c
of the 8-byte alignment on an Int64:

```swift
// Size of 9, but stride of 16
struct MyState {
  var a: Int64 // 8 byte and 8 byte aligned
  var b: Int8  // 1 byte
  // var padding: Int56 (7 bytes of padding)
}
```

This means that `(2 * MyState)` would have a size of `16 + 9 = 25` instead of
`16 + 16 = 32` like one may expect if one were considering a fixed size
array from C. As another example of this, consider if we made a `MyPair` that
contains a `MyState`:

```swift
struct MyPair {
  var state: MyState
  var b: Int8
}
```

By the rules of C, we would have that `MyPair` has a size of `24` since
`MyState` has an alignment of `8` and a stride of 16 meaning that `b` causes us
to need a whole additional byte to satisfy the alignment. In contrast in Swift,
`MyPair` will have a size of `10` and a stride of `16`.

Now lets take a look at how this plays with homogenous tuples stored in structs:

```swift
struct MyHomogeneousTuplePair {
  var states: (2 * MyState)
  var b: Int8
}
```

If we wanted to load from `states.1`, we would need to load bytes `(16,32]` and
then mask out bytes `(25,32]`.

Luckily for us we import all C types (including tuples) as having the same
size/stride. So if `(2 * MyState)` was a fixed size array imported from C, we
would not have these issues. The issue is only for Swift types.

## Acknowledgments

Thanks to Joe Groff for suggesting going down this path! Also thanks to Andrew
Trick, Michael Ilseman, David Smith, Tim Kientzle, John McCall, Doug Gregor, and
many others for feedback on this!
