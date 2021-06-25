# Improved Language Support for Homogeneous Tuples

* Proposal: [SE-NNNN](NNNN-improved-homogeneous-tuple-language-support.md)
* Author: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Awaiting implementation**

* Reference: [SE-0283](0283-tuples-are-equatable-comparable-hashable.md)
* Pitch Thread 1: [https://forums.swift.org/t/pitch-improved-compiler-support-for-large-homogeneous-tuples/49023](https://forums.swift.org/t/pitch-improved-compiler-support-for-large-homogeneous-tuples/49023)

## Introduction

Swift programmers frequently encounter homogeneous tuples (`(T, T, ..., T)`) in
contexts such as:

1. Writing code that uses an imported C fixed size array in Swift.
2. Defining a Swift nominal type that uses a homogeneous tuples to represent a
   field with a fixed layout of bytes (e.x.: `SmallString`)

In this proposal, we attempt to improve language support for such tuples in a
manner that makes homogeneous tuples easier to write and compose better with the
rest of the language. The specific list of proposed changes are:

+ The addition of syntactic sugar for declaring large homogeneous tuples.
+ Introducing a new `HomogeneousTuple` protocol. This protocol will
  extend `RandomAccessCollection` and `MutableCollection` allowing for
  homogeneous tuples to be used as collections and access contiguous storage. It
  will also provide a place for us to declare new helper init methods for
  initializing homogeneous tuple memory, allow us to expose the count of a tuple
  type, and also allow us to implement using default protocol implementations.
  We will only allow for homogeneous tuples to conform to this, not user types.
+ Changing the Clang Importer to import fixed size arrays as sugared homogeneous
  tuples and remove the arbitrary limitation on the number of elements (4096
  elements) that an imported fixed size array can have now that type checking
  homogeneous tuples is fast.
+ Eliminating the need to use unsafe type pointer punning to pass imported C
  fixed size arrays to imported C APIs using the `&` operator.
+ Changing the Swift calling convention ABI to pass tuples with 7 or more
  elements as aggregates instead of destructuring the tuple into separate
  arguments and return values. The calling convention ABI for tuples with less
  than 7 elements will not be modified.

NOTE: This proposal is specifically not attempting to implement a fixed size
"Swifty" array for all Swift programmers. Instead, we are attempting to extend
Swift in a minimal, composable way that helps system/stdlib programmers get
their job done today in the context where these tuples are already used today.

Swift-evolution thread: **Still a Pitch**.

## Motivation

Today in Swift, system programmers interact with homogeneous tuples when working
with imported fixed size C-arrays and when defining a fixed buffer of a specific
type. As a quick example, consider the following C struct,

```c
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
tuple elements increase, using homogeneous tuples in this manner does not scale
from a usability and compile time perspective. We explore these difficulties
below:

### Problem 1: Basic Value and Collection operations on large homogeneous tuples require use of a source generator or unsafe code

To explore the implications of the current homogeneous tuple model, imagine that
we are defining a System API that wants to maintain a cache of 128 pointers to
improve performance. Since we need to have enough storage for 128 pointers, we
use a homogeneous tuple of `UnsafeMutablePointer` that point at objects of type
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

### Problem 2: Fixed size C array types imported as Homogeneous Tuples do not compose with idiomatic Swift or imported C functions that take fixed size arrays

The Clang Importer today imports C fixed size arrays into Swift as homogeneous
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

Currently, common idiomatic operations one performs on such a fixed size array
in C do not have an equivalent expression in idiomatic Swift. Instead the
programmar must resort to the use of a code generator or unsafe
code. Additionally, there are artificial limitations imposed by the Clang
Importer on importable code. We go through each of the issues below:

#### Problem 2a: Imported Fixed size C arrays can not be iterated over due to lacking Collection conformance

One of the most basic tasks when working with fixed size arrays in C is to
iterate over the contents of a fixed size array. When fixed size C arrays are
imported into Swift as a homogeneous tuple, one can not idiomatically iterate
over the C array's homogeneous tuple representation since homogeneous tuples do
not have a Collection conformance. Instead, one must use unsafe code by defining
a nominal type wrapper and use one of the techniques above from our
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

#### Problem 2b: Imported C APIs that assume Array To Pointer Decay of a Fixed Size Array can not be called without using unsafe code

In C, we would be able to write the following code to pass our data buffer into
`printFloatData` due to fixed size arrays decaying to pointers:

```c
void MyData_printFloatData(MyData_t *data) {
    // printFloatData expects a 'float *', but data->dataBuffer is of type
    // 'float [1024]'. This is valid C b/c 'float [1024]' decays to 'float *'
    // implicitly.
    printFloatData(data->dataBuffer);
}
```

Once we import `MyData_t` into Swift though, `data.dataBuffer` has type `(Float,
..., Float)` meaning that if we call `printFloatData` idiomatically by using the
`&` operator, the types no longer line up since `&dataBuffer` is an unsafe
pointer to a single tuple of N-`Float` instead of an unsafe pointer to a piece
of memory containing `N` float contiguously:

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

To make this example work, we must write instead the following verbose code to satisfy the type checker:

```swift
  extension MyData_t {
    func cPrint() {
      withUnsafeBytes(of: x?.dataBuffer) {
        printFloatData($0.baseAddress!.assumingMemoryBound(to: Float.self))
      }
    }
  }
```

#### Problem 2c: Scalability problems cause Clang Importer to not import arrays larger than 4096.

The ClangImporter due to limitations in the compiler around large homogeneous
tuples will not import a fixed size array if the array would be imported as a
tuple with more than 4096 elements (see
[ImportType.cpp](https://github.com/apple/swift/blob/e91b305b940362238c0b63b27fd3cccdbecadbaa/lib/ClangImporter/ImportType.cpp#L571)). At
a high level, the compiler seems to be running into the same problem of the
programmer: we have made the problem more difficult than it need to be by
forcing the expression of redundant information in the language.

### Problem 3: Large homogeneous tuples exascerbate linear type checking behavior from [SE-0283](0283-tuples-are-equatable-comparable-hashable.md)

[SE-0283](0283-tuples-are-equatable-comparable-hashable.md) changed tuples to
conform to Equatable, Hashable, and Comparable if all of the elements of such a
tuple also conform to those protocols. This increased language expressiveness at
the expense of requiring the type checker to perform linear work for such pieces
of code since the type checker must prove that all elements of the tuple
individually possess such conformances. This linear work is ignorable for small
tuples, but for large homogeneous tuples, the work becomes significantly more
expensive if one compares, hashes, or equates homogeneous tuples in a large code
base.

### Problem 4: Swift's Tuple ABI does not scale well at compile and runtime for large Tuples

Today Swift's ABI possesses a rule that all tuples are eagerly destructured at
all costs. This means that if one were to pass a 1024 element tuple as an
argument, the resulting function would have 1024 arguments resulting in poor
performance. This ABI is an artifact of Swift's early evolution where there was
an attempt to enable tuples to be passed as a function argument list. In such a
case, destructuring the tuple into arguments makes sense. Moving forward in
time, that functionality was eliminated from the language so now we are just
left with an ABI rule that is actively harmful to both compile time and runtime
performance. The result of these issues together is that people generally do not
use tuples beyond a certain size since such code does not scale well. As an
example, consider the following "scale file test":

```swift
struct Foo {
  var value: (
% for i in range(0, N*100):
  Int,
% end
  Int
  ) = (
% for i in range(0, N*100):
   0,
% end
   0)
}

@inline(never)
public func trampoline<T>(_ t: T) -> T { t }

func main() {
    let f = Foo()
    trampoline(f.value)
}

main()
```

Using this and the scale test utility in Swift's repo, one can see that as we
increase tuple argument size:

+ -Onone,-O total compilation wall time is quadratic [O(n^2)]
+ -Onone,-O type checker compilation wall time is quadratic [O(n^2)]
+ -Onone,-O code size is linear O(n).

On the author's computer (2019 Macbook Pro), when N = 4096, a near trunk
compiler takes ~25 seconds to compile a piece of code that should be instant.

## Proposed solution

### Syntax Change: Homogeneous Tuple Type Sugar

We propose a new sugar syntax for a "homogeneous tuple span" element. The
grammar of tuple elements in Swift would be extended as follows:

```swift
(5 * Int) -> (Int, Int, Int, Int, Int)
```

This sugar is expanded before type checking, so beyond the homogeneous bit that
we set in the tuple type itself, the rest of the compiler just sees a normal
tuple and thus will be minimally effected. We would restrict ones ability to
only use homogeneous tuple spans in tuples that are otherwise single element
lists.

### Standard Library Change: Add HomogeneousTuple protocol

We propose the addition of a new protocol called `HomogeneousTuple` that all
homogeneous tuples implicitly conform to:

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

Importantly notice that we are defining the Subsequence associated type to be
the default subsequence type (a slice on top of Self).

### Clang Importer Change: Import Fixed Size Arrays as Sugared Homogeneous Tuples

We propose changing the Clang Importer so that it sets the homogeneous bit on all
fixed size arrays that it imports as homogeneous tuples. This will allow for all
of the benefits from the work above to apply to these types when used in Swift.

### Language Change: Change `&` operator to convert `(N x T)` to `UnsafePointer<T>` instead of `UnsafePointer<(N x T)>` if `(N x T)` is an imported C type.

We propose allowing for `&` to convert `(N x T)` to `UnsafePointer<T>` instead
of `UnsafePointer<(N x T)>` if `(N x T)` was synthesize by the Clang
Importer. This will ensure that we can easily pass imported fixed size arrays to
corresponding imported C apis in nice idiomatic Swift:

```swift
   extension MyData_t {
     mutating func cPrint1() {
       // No type error here since '&dataBuffer' results in an
       // UnsafePointer<Float> since 'dataBuffer' was synthesized by the
       // ClangImporter and is being passed to a C function (printFloatData).
       printFloatData(&dataBuffer)
     }
   }
```

### ABI Break: Pass Tuples with 7 or more elements as a tuple aggregate instead of eagerly destructuring

This would cause us to pass tuples with 7 or more elements indirectly as an
entire aggregate rather than passing the tuple as individual destructured
values. This will eliminate the poor compile time performance of the compiler
when compiling such code and the runtime performance of such code. We discuss
the ABI break below in the ABI difference section with a detailed analysis upon
what ABI incompatibilities we can expect and what methodology/tooling we will
provide so that the ABI break can be done incrementally at the leasure of
the relevant ABI stable library's author.

## Detailed design

### Grammar Change: Add support for homogeneous tuple span

In order to prase homogeneous tuple spans, we modify the swift grammar as follows:
```
  type-tuple:
    '(' type-tuple-body? ')'
  type-tuple-body:
    type-tuple-element (',' type-tuple-element)* '...'?
  type-tuple-element:
    identifier? identifier ':' type
    type
    type-homogeneous-tuple-span
  integer_literal_gt_one:
    [2-9][0-9]*
  type-homogeneous-tuple-span:
    integer_literal_gt_one '*' type
  type-pure-homogeneous-tuple:
    '(' type-homogeneous-tuple-span ')'
```

When we parse a tuple that consists of a single homogeneous tuple span element,
we set in the tuple type a homogeneous tuple bit using spare bits already
available in the tuple type.

NOTE: We specifically ban homogeneous tuple spans that provide an integer literal
of 0 or 1. The reason for this is that given that Swift's generics do not
support variadics or type level integers, having homogeneous tuples of size 0, 1
are not useful. Instead we require our users to use an empty tuple or just an
element.

### Type Checker: Reduce number of Type variables for Homogeneous Tuples

The type checker today creates a type variable for all tuple elements when
solving constraints since any element of the tuple could be different. But if
the user provides us with the information ahead of time that they explicitly
have a homogeneous tuple, we can introduce a join constraint on all of the tuple
elements allowing the constraint solver to be more efficient via the use of its
merging heuristic:

```swift
// We know that all of these must be the same type, so we can provide the type checker with that constraint info.
let x: (3 * Int) = (1, 2, 3)
```

### Type Checker: Introduce homogeneous tuple bit and use it to eliminate linear type checking for comparing, equating, and hashing homogeneous tuples

Today the type checker has to perform a linear amount of work when type checking
homogeneous tuples. This is exascerbated by the comparable/equatable/hashable
conformances added in
[SE-0283](0283-tuples-are-equatable-comparable-hashable.md). We can eliminate
this for large homogeneous tuples by using a spare bit in Tuple type to stash if
the tuple was originally parsed or imported as a homogeneous tuple. Then when we
typecheck a hash, comparable, or equality operation we will be able to infer
that all elements of a tuple that has the homogeneous tuple bit set is the same,
allowing the type checker can just check that the first element of the tuple
type conform to the protocol, eliminating the linear work.

### Standard Library: Introduce HomogeneousTuple protocol and implicitly conform Homogeneous Tuples to it

Our approach to implementing the HomogeneousTuple protocol is guided by the
principles that:

+ The `HomogeneousTuple` protocol or any conformances to it should not be
  implemented using assembly in the runtime. Instead, we should declare it using
  an explicit protocol in the standard library.

+ Homogeneous tuples should conform to `HomogeneousTuple` implicitly and the
  conformance for HomogeneousTuples should be generated by IRGen when compiling
  the Swift core library. This will ensure that the implementation of the
  conformance will stay up to date with the rest of the compiler (vs hand
  written assembly) and will enable us to use IRGen to make the implementation
  platform independent.

+ Homogeneous tuples should have their conformance to the `HomogeneousTuple`
  protocol implemented as much as possible in the standard library using default
  a default protocol conformance implementation. We should implement only a
  minimal set of leaf functionality in the runtime by trampolining from the
  default implementation to a runtime c++ implementation using unsafe
  code. IRGen should only generate the necessary type metadata to ensure that
  the relevant builtin Conformance only defines type metadata required to
  conform to `MutableCollection` and `RangeReplaceableCollection` and otherwise
  delegate to the stdlib's default protocol conformance implementation.

The reason why we believe this is a superior approach is that the implementation
is more maintainable and platform independent. Also it becomes easy to declare
and implement additional helper methods in the standard library itself without
touching the runtime in many cases.

As an example implementation consider the code on the following branch:
`$INSERT_BRANCH_HERE`

NOTE: Since this is a protocol, we will allow for users to use the
protocol to constraint extensions and as a generic requirement, e.x.:

```swift
func myHomogeneousTupleFunc<T : HomogeneousTuple>(...) { ... }

extension X where Y : HomogeneousTuple { ... }
```

We do not expect any scalability issues to result due to the ability of the type
checker to use the homogeneous tuple bit to determine quickly if a tuple
conforms to the protocol.

### Clang Importer Changes

The Clang Importer will be modified so that we set the homogeneous tuple bit on
imported fixed size arrays. This will ensure that we print out the imported
tuples in homogeneous form and will allow us to lift the 4096 element limit since
we will no longer have type checker slow down issues.

### Improve Scalability of Swift's Calling Convention Tuple ABI for Tuple with 7 or more elements

We propose that we change the Tuple ABI of Swift's calling convention for all
new swift code where a Tuple has size > 6. This will fix scalability problems
in the compiler (as shown by the quadratic behavior above) and improve
performance for any code written with legacy code recompiled with the newer
compiler as well as any code written to use large homogeneous tuples. We
describe how we handle breaking the ABI incrementally and why we chose 6 in the
ABI stability section below.

## Source compatibility

This is purely additive from a source perspective. The only possible concern is
from the ClangImporter changing how it prints such headers. This is not actually
a concern since this is not used by any Swift programs for compilation as an
artifact. Rather the ClangImporter in this case is instead only intended to show
programmers what the ClangImporter imported.

## Effect on ABI stability

The main effect on ABI stability of this proposal is the tuple ABI break. We
first describe how we decided that the cut off point where the new ABI should
start is 7. Then we will analyze the ABI implications of the change. Finally we
will lay out a methodology based on attributes/tooling that will allow for
stabilized library authors to incrementally update their libraries to use the
new tuple ABI when they are able to guarantee that down stream clients will not
break.

### Why break ABI when a tuple has N >= 7 elements vs some other N

The author's came up with the number 7 by applying a modified version of
[swift-ast-script](https://github.com/gottesmm/swift/tree/ast-script-for-tuples)
to print out the size of all tuple arguments and results in swift interface
files passed in on the command line. The author then applied this tool to all
swift interface files in Xcode (the version used to compile swift today). The
gathered data is here: []. As one can see most stabilzied frameworks use only
1-2 element tuples. The two exceptions are the Swift standard library which has
a max tuple size of 6 and Foundation which has a max tuple size of 32.

The data gathered is available here: . As one can see, except for Foundation, 

A natural question is why are the authors suggesting that we do the ABI break
when a tuple has 7 elements instead of some other N.

This is because the authors developed a tool that enabled 


calling convention ABI break. The authors believe that since large tuples are
not used often that breaking the ABI for tuples with 7 or more elements can be
accomplished in an incremental way, moving the ecosystem forward and eliminating
a non-scalable part of the ABI that is now vestigal and unneeded. The way that the authors came up with the 6 element number is that 

As with any proposed ABI break we first present data about how many
stabilized binaries will break given that we chose 

The main effect on ABI stability comes down to whether or not we decide to break
ABI for large N and if we do decide to do so in what way do we mitigate that
change. We believe that we can avoid having a major ABI break problem by taking
the following steps:

+ All code compiled by a new-ABI compiler will be emitted with the new tuple ABI
  unless explicitly marked in source with a new attribute called
  `@_eagerlyDestructuredTupleABI`. This attribute will cause the compiler to
  emit binary code with the old ABI and can be used by frameworks to ensure that
  specific APIs that are in use and are not ready to be migrated can still use
  the old ABI and thus not break their ABI. We also will provide a flag that
  stabilized libraries can use to opt in to their entire library using the old
  ABI to ease migration. All of these together will ensure that library authors
  can have flexibility/control over when the ABI break will occur and allow for
  incremental migration.

+ new-ABI compilers will be taught to infer that all tuples imported from a
  stabilized swift interface file will use the destructured tuple ABI unless
  explicitly marked with a new attribute called
  `@_nonDestructuredTupleABI`. This will have the effect that all stabilized
  interface files produced by an old-ABI compiler will still be readable by a
  new-ABI compiler and will use the correct old-ABI. It also will ensure that
  old-ABI compilers will be unable to compile against stabilized libraries that
  are using the new ABI since the old-ABI compiler will not recognize the new
  `@_nonDestructureTupleABI` attribute and will error.

The compiler will when creating a swift interface file for new code will
automatically for the user place the `@_nonDestructuredTupleABI` attribute
unless the user marked the function with the `@_eagerlyDestructuredTupleABI` or
if the user passes in a specific option that says

The effect of this at compile time will be:

+ Old compilers compiling against a stabilized swift interface file produced by
  an old compiler will succeed in compilation.

+ Old compilers that compile against a stabilized interface file produced by a
  newer compiler will fail to compile if the new compiler marks any functions as
  using the `@_eagerlyDestructuredTupleABI` that has some APIs that use the new
  tuple ABI will fail to compile since the compiler will not recognize the
  attribute and will error.

+ New compilers that compile against a stabilized interface file produced by an
  older compiler will succeed and produce a binary that uses the old eagerly
  destructured ABI since no APIs will be marked with
  `@_nonDestructuredTupleABI`.

+ New compilers that compile against a stabilized interface file produced by a
  newer compiler will succeed and be able to take advantage of all of the APIs
  that are marked as `@_nonDestructuredTupleABI` by the emitting compiler.

The effect of this in terms of backwards deployment are as follows (noting that
we use old/new binary to refer to binaries compiled with either the old/new
tuple ABI):

+ If an old ABI executable links against a new ABI executable

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
   homogeneous tuples will either need to be modified (a source compatibility
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

Now lets take a look at how this plays with homogeneous tuples stored in structs:

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
