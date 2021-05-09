# Improved Language Support for Homogenous Tuples

* Proposal: [SE-NNNN](NNNN-improved-homogenous-tuple-language-support.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

Define a homogenous tuple of type `T` as a tuple that only contains elements of
type `T`, e.x.: `(T, T, ..., T)`. These types of tuples are used by system
programmers in Swift to express a fixed size buffer of type `T` with a
guaranteed layout. Not much language support has been provided to make working
with these tuples easy/expressive and certain capabilities are impossible to
implement without resorting to reinterpret casting/language accidents. This
proposal is an attempt to add that missing language support so that it is easier
to: declare such tuples, initialize such tuples, treat these tuples as a
collection, and work with these tuples when importing from C programs.

NOTE: This proposal is specifically not attempting to implement a fixed size
"Swifty" array for all Swift programmers. Instead, we are attempting to extend
Swift in a minimal, composable way that helps system/stdlib programmers get
their job done today.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

Today in Swift, system programmers use homogenous tuples to represent a fixed
buffer of bytes of a certain type. As an example of this, consider the following
example from the swift standard library,

```swift
@frozen @usableFromInline
internal struct _SmallString {
  @usableFromInline
  internal var _storage: (UInt64, UInt64)
}

// Taken from:
// https://github.com/apple/swift/blob/833a453c8ad6e9982e849229d6f91532717cd354/stdlib/public/core/SmallString.swift#L33
```

By declaring `_storage` as a frozen homogenous tuple of type `UInt64`, we are
able to guarantee that `_SmallString` when laid out in memory is exactly 128
bits and can be treated as layout compatible with other 128 bit values. We can
define initializers based off of the tuple representation as well as access the
bottom/lower half of the bits using tuple syntax, e.x.:

```swift
extension _SmallString {
  @inlinable @inline(__always)
  internal init(rawUnchecked bits: RawBitPattern) {
    self._storage = bits
  }

  @inlinable
  internal var leadingRawBits: UInt64 {
    @inline(__always) get { return _storage.0 }
    @inline(__always) set { _storage.0 = newValue }
  }

  @inlinable
  internal var trailingRawBits: UInt64 {
    @inline(__always) get { return _storage.1 }
    @inline(__always) set { _storage.1 = newValue }
  }
}
```

Sadly, this type of model doesn't scale well as one can see due to the method in
which one must define accessors to access the top/bottom storage without needing
to index into the tuple using tuple indices. Lets naively extend this model to
represent a SmallArray of 128 nullable pointers used to model a cache of values
in a runtime data structure:

```swift
@frozen @usableFromInline
struct _SmallPointerArray128<T> {
  @usableFromInline
  internal var _storage: (UnsafeMutablePointer<T>?, UnsafeMutablePointer<T>?, UnsafeMutablePointer<T>?,
                          /* 122 more pointers */
                          UnsafeMutablePointer<T>?, UnsafeMutablePointer<T>?, UnsafeMutablePointer<T>?)
}
```

Clearly, even though `_storage` represents at a binary level 128 pointers, we
can not initialize it in a nice way without a source generator,

```swift
struct _SmallPointerArray128<T> {
  @usableFromInline
  internal init() {
    _storage = (nil, nil, ..., /* nil 125 times */, nil)
  }
  @usableFromInline
  internal init(sentinelValue: UnsafeMutablePointer<T>) {
    _storage = (sentinelValue, sentinelValue, ..., /* sentinelValue 125 times */, sentinelValue)
  }
}
```

or have a brief subscript implementation:

```swift
/* MG: NOTE to self: show codegen here */
extension SmallPointerArray128<T> {
  subscript(index: Int) -> T {
    switch index {
    case 0:
      return _storage.0
    case 1:
      return _storage.1
    /* ... 125 more cases ... */
    case 127:
      return _storage.127
    default:
      fatalError("Out of bounds!")
    }
  }
}
```

and can only iterate over the storage using indices via subscript rather than as
a true first class for loop,

```swift
for index in 0..<128 {
  let t = smallArray[index]
  ...
}
```

In sum, the model doesn't scale well since we can not succintly describe a tuple
of N elements without writing out the entire tuple and we can not use a tuple
like a collection without writing a huge function that iterates through the
array.

These issues also show up when importing code from C since fixed size arrays are
imported as tuples from C. As an example, consider the following C code:

```c
float globalDataBuffer[1024];
struct MyStruct {
    int dataBuffer[128];
};
```

These will be imported into Swift as:

```swift
var globalDataBuffer: (Float, Float, ..., /* 1021 Floats */, Float)
struct MyStruct {
  var dataBuffer: (Int, Int, ..., /* 125 Ints */, Int)
}
```

beyond being hard to read (going off the page), since such a tuple does not have
a collection conformance, one can not iterate over it in a naive way and since
one cannot get the size of a tuple from the underlying type, one would need to
hard code the size and write accessors like we did above to be able to iterate
over the tuple.

## Proposed solution

In order to make the life easier for System Programmers, we attack this lack of
expressivity in the following ways:

1. We propose a new sugar syntax for a "homogenous tuple span"
   element. This would be extending the grammar of tuple elements in Swift and
   would let one write a tuple element that expands out to a homogenous list of
   elements of the same type as follows:
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
   Any tuple that consists of a single homogenous tuple span is then defined as
   a _pure homogenous tuple_.
   
   As an additional bonus a homogenous tuple span element can be mix/matched
   with other elements to create more complex layout compatible data structures,
   e.x.:
   ```swift
     (Float, 5 x Int, String, 2 x AnyObject)
     // --> expands to
     (Float, Int, Int, Int, Int, Int, String, AnyObject, AnyObject)
   ```
   This is not integral to the proposal and if necessary can be sliced off and
   we can allow only for tuples to only have a single homogenous tuple element.

2. We propose adding a series of builtin inits for homogenous tuples to make it
   easier to initialize homogenous tuples. Specifically:

   * `init(repeating: repeatedValue: Element)`: This will allow for users to easily initialize a
     large tuple all with the same value:
     ```swift
     init(repeating repeatedValue: Element)

     let x = (128 x Int)(repeating: 0)
     ```
     NOTE: Since the actual number of elements in the homogenous tuple is fixed,
     we do not need to pass in the count.

   * `init(initializingWith initializer: (UnsafeMutableBufferPointer<Element>) throws -> Void) rethrows`:
     This method allows for one to initialize all elements of a tuple with
     pre-known values avoiding the need to first zero initialize such a tuple:

     ```swift
     // Fill tuple with integral data.
     let x = (1024 x Int) { x: UnsafeMutableBufferPointer<Int> in
       for i in 0..<1024 { x[i] = i }
     }
     // Or more succintly:
     let x = (1024 x Int) { for i in 0..<1024 { $0[i] = i } }
     // memmove data from a data buffer into a tuple.
     let x = (1024 x Int) { x: UnsafeMutableBufferPointer<Int> in
       // TODO: Fix syntax
       memcpy(other, x, x.size)
     }
     ```

3. We propose adding a builtin collection conformance for pure homogenous tuples
   building upon the work done in `SE-TUPLE_COMPARABLE_HASHABLE`. This will then
   allow for full expressivity of the tuple:

   * Iteration.
   * Access To As Raw Bytes
   * Count
   * Algorithms

   Show how one can not use these things without the collection conformance and
   show how they improve the world and makes it cleaner, safer, more efficient t


This will thenall Describe your solution to the problem. Provide examples and
describe how they work. Show how your solution is better than current
workarounds: is it cleaner, safer, or more efficient?

## Detailed design

The main changes to the Swift language itself come in a few areas:

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
     possessing this property. This ensures that when we are performing type
     checking, we can just type check the single homogenous tuple type element
     (which we can get from the first element of the tuple) rather than
     performing a linear chain of type checking operations.

* Homogenous Tuple Types and the ObjC Importer

* Homogenous Tuple Types and extra methods

* Homogenous Tuple Types and collection conformance.

Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.

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

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

## Acknowledgments

If significant changes or improvements suggested by members of the 
community were incorporated into the proposal as it developed, take a
moment here to thank them for their contributions. Swift evolution is a 
collaborative process, and everyone's input should receive recognition!
