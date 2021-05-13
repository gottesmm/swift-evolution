# Improved Language Support for Homogenous Tuples

* Proposal: [SE-NNNN](NNNN-improved-homogenous-tuple-language-support.md)
  * Authors: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

Define a pure homogenous tuple of type `T` as a tuple that only contains
elements of type `T`, e.x.: `(T, T, ..., T)`. Such tuples are used by system
programmers in Swift to express a fixed size buffer of type `T` with a
guaranteed layout. Not much language support has been provided to make working
with tuple typed storage easy/expressive. In this proposal, we attack this hole
by proposing the following changes to the language:

1. The addition of sugar for declaring large homogenous tuples.
2. When parsing tuple types, stashing a bit in the tuple type that states if we
   parsed the tuple as a homogenous tuple.
3. Improve type checker performance for such tuples by taking advantage of said
   bit to eliminate the need to perform type checking operations on each element
   of the tuple instead of just the first element.
3. Adding new initializers for homogenous tuples:
   a. A repeating initializer that initializes all elements of the tuple to the same value.
   b. An unsafe uninitialized memory based initializer similar to Array's.
4. Adding `RandomAccessCollection` and `MutableCollection` conformances to enable usage
   as a collection and accessing as contiguous storage.
5. Changing the Clang Importer to import fixed size arrays as sugared homogenous
   tuples and remove the arbitrary limitation on the number of elements that an
   imported fixed size array can have (4096 elements).
6. Eliminating the need to use unsafe type pointer punning to pass imported C
   fixed size arrays to related imported C APIs.
7. Adding to the Standard Library a typed array view over such tuples that
   treats the tuple as a bag of bits.

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
128 bit values, which is totally awesome! That being said, using homogenous
tuples in this manner starts to create usage difficulties as the number of
elements in the tuple increases. We explore this below.

### Basic operations on large tuples forced to use a source generator or unsafe code

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

If we want to define a subscript for our type, then we are forced to again use
either a Swift source code generator or use unsafe code,

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
        buffer.load(fromByteOffset: 5*MemoryLayout<PointerType?>.stride, as: PointerType?)
    }
  }
}
```

In all of these cases, the lack of language support add unnecessary complexity
to the program for what should be simple primitive operations that should scale
naturally to larger types without needing us to use unsafe code. Even if we say
that the unsafe code example is ok, we would be relying on the optimizer to
eliminate overhead. Relying on the optimizer is ok in the large, but when system
programming praying is insufficient since the programmer really needs to be able
to guarantee that the relevant defined performance constraints are guaranteed at
compile time. This generally forces the programmer to look at the assembly to
guarantee that the optimizer optimized the unsafe code as the programmer
expected.

### Large fixed size arrays

These issues of expressivity just in terms of Swift itself also show up when
working with imported C code. Specifically:

1. Large homogenous tuples increase compile time specifically by harming type
   checker performance. In fact, the ClangImporter will not import a fixed size
   array if the array would be imported as a tuple with more than 4096 elements
   due to this issue (see
   [ImportType.cpp](https://github.com/apple/swift/blob/e91b305b940362238c0b63b27fd3cccdbecadbaa/lib/ClangImporter/ImportType.cpp#L571)). At
   a high level, the type checker is running into the same problem of the
   programmer: we have made the problem more difficult than it need to be by
   forcing the expression of redundant information in the language.

2. When one imports a fixed size array from C as a tuple, one must define ones
   own nominal type on top that uses the techniques above in order to

3. Imported fixed size arrays from C no longer compose with associated C apis
   when working with C code in Swift. As an example:
   ```c
   float globalDataBuffer[1024];
   void useMyDataBuffer(const float *buffer);
   ```
   This API is imported into Swift as:
   ```swift
   var globalDataBuffer: (Float, Float, ..., /* 1021 Floats */, Float)
   func useGlobalDataBuffer(buffer: UnsafePointer<Float>)
   ```
   The user may expect to be able to just pass `globalDataBuffer` to
   `useMyDataBuffer` like one could do in C, but if one attempts to do so in
   Swift, one will immediately hit the following type checker error:
   ```swift
   func myF() {
     // Error! Cannot convert value of type 'UnsafeMutablePointer<(Float, /* Float
     // 1022 times */, Float)>' to expected argument type
     // 'UnsafeMutablePointer<Float>'
     useGlobalDataBuffer(&globalDataBuffer)
   }
   ```
   To work around this, we must unsafely use memory binding APIs to safely type pun
   our pointer (taking advantage of layout compatibility):
   ```swift
     withUnsafeBytes(of: &globalDataBuffer) { (buff: UnsafeRawBufferPointer) in
       getValue(buff.baseAddress!.assumingMemoryBound(to: Float.self))
     }
   ```
   bloating what should be a simple/straight forward operation (passing the address
   of a C variable to a C function).

## Proposed solution

In order to make the life easier for System Programmers, we attack this lack of
expressivity in the following ways:

1. We propose a new sugar syntax for a "homogenous tuple span"
   element. The grammar of tuple elements in Swift would be extended as follows:
   ```swift
     (5 x Int)
   ```
   which would expand out (that is de-sugared) to the following type in the parser:
   ```swift
     (Int, Int, Int, Int, Int)
   ```
   This directly attacks and eliminates the declaration issue around declaring
   large tuples and will enable us to import C types with brevity allowing us to
   import the earlier mentioned C code as:
   ```swift
   var globalDataBuffer: (1024 x Float)
   struct MyStruct {
     var dataBuffer: (128 x Int)
   }
   ```
   Define any tuple that consists of a single homogenous tuple span as
   a _pure homogenous tuple_.

   As an additional bonus a homogenous tuple span element can be mix/matched
   with other elements to create more complex layout compatible data structures,
   e.x.:
   ```swift
     (Float, 5 x Int, String, 2 x AnyObject)
     // --> expands to
     (Float, Int, Int, Int, Int, Int, String, AnyObject, AnyObject)
   ```
   This capability is not integral to the proposal and if necessary can be sliced off and
   we can allow only for tuples to only have a single homogenous tuple element.

2. We propose that TupleType in the compiler have a bit upon it that states if
   the TupleType was originally parsed from a homogenous tuple. This will ensure
   that we can print out the tuple with the sugar and can also use it to improve
   type checker compile time by allowing us to know that we can use the first
   element of the homogenous tuple (noting that the empty tuple is not a
   homogenous tuple) for type checking purposes.

2. We propose adding a series of pre-defined builtin initializers for homogenous
   tuples to ease homogenous tuple initialization. Specifically:

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

  * `init(initializingWith initializer: (UnsafeMutableBufferPointer<Element>) throws -> Void) rethrows`:
     This method allows for one to initialize all elements of a tuple with
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

3. We propose implementing in SILGen a special form of argument to pointer
   conversion for homogenous tuples that causes the tuple to be converted to
   `Unsafe[Mutable]Pointer<(T, ..., T)>` to `UnsafeMutablePointer<T>`. This
   change is layout compatible with tuple layout.

4. We propose adding a builtin Collection conformance for pure homogenous tuples
   building upon the work done in `SE-TUPLE_COMPARABLE_HASHABLE`, allowing for
   users to:

   a. Iterate over the tuple.
   b. Get the count of the tuple in a programatic matter.
   c. Use standard algorithms like map, reduce, and filter.
   d. Apply the full set of Swift's algorithms to the type.

## Detailed design

The main changes to the Swift language itself occurs when parsing/printing
homogenous tuples:

* Homogenous Tuple Span Parsing. This is implemented by changing the tuple
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
  In terms of implementation, we recognize the count and the 'x' when parsing
  tuple elements. So, given a tuple '(N x T)' where 'N' is an integer literal
  and 'T' is a type, we perform normal parsing, but:

  1. In ``Parser::parseTypeTupleBody``, while parsing tuple elements, we
     maintain additional state: a boolean called `isHomogenousTuple` that tracks
     if our tuple is a pure homogenous tuple (initialized optimistically to
     true) and a variable called `expectedHomogenousTupleType` that if non-null
     is the type of the first homogenous tuple span that we saw while parsing
     this tuple. This is easy to do in the parser since the method
     Parser::parseTypeTupleBody parses an entire list of tuple elements at a
     time using a closure based API, so we can just introduce this state without
     any effort.

  2. While parsing elements, if we see a non-homogenous tuple span element, we
     parse normally, but set the bit stating that this tuple is /not/ a pure
     homogenous tuple.

  3. While parsing elements, if we see a homogenous tuple span element of form
     `N x Type`:

     a. We first check if `isHomogenousTuple` is still true. In such a case, we
        check if `expectedHomogenousTupleType` is non-null.

    (i). If `expectedHomogenousTupleType` is null, then we know that we have not
        seen /any/ homogenous tuple span elements yet and initialize
        `expectedHomogenousTupleType` to the base type of our
        homogenous-tuple-span. Then we append `N` elements of type `Type` to our
        tuple type.

    (ii). If `expectedHomogenousTupleType` is non-null, then we check if the
        just parsed homogenous tuple element span type matches. If not, we set
        `isHomogenousTuple` to false. In other case, we then append `N` elements
        of type `T` to our tuple type.

  4. Once we have finished parsing all tuple body elements, we then check the
     `isHomogenousTuple` bit. If said bit is set, then we mark our tuple type as
     possessing this property. This ensures that when we can print out our
     tuples nicely and when performing type checking only type check a single
     type element rather than a linear sequence of tuples.

* Homogenous Tuple Types and the ObjC Importer

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
