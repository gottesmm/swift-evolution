# Partial Consumption and Reinitialization of Noncopyable fields of Noncopyable Types

* Proposal: [SE-????](0000-partial-consumption-and-reinitialization-of-noncopyable-fields-of-noncopyable-types.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Pitch with implementation on main behind a flag**
* Upcoming Feature Flag: `MoveOnlyPartialConsume`

## Introduction

SE-390 defines all noncopyable types as being either [fully initialized or fully destroyed (outside of initializers)](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md#finer-grained-destructuring-in-consuming-methods-and-deinit).
This reduces language expressivity by preventing a noncopyable field of a noncopyable
binding from being consumed without fully consuming the entire binding. A
particularly annoying case where this restriction is noticeable is self in
mutating methods. We would like to loosen the language rules to allow for
partial consumption and initialization in these cases.

## Motivation

Given a mutable binding (for example: var, inout), Swift does not allow for a stored
noncopyable field of the binding to be partially consumed or initialized:

```swift
struct Socket : ~Copyable {
    // Initialize an unused socket.
    init() { ... }
    deinit() { ... }
    mutating func read() -> UnsafeMutableRawBufferPointer { ... }
    consuming func close() { ... }
}
class Model { ... }

struct MicroServiceRequest : ~Copyable {
    var readSocket: Socket
    var writeSocket: Socket
    var model: Model
}

var request = MicroServiceRequest()
let _ = s.readSocket // Error! Cannot partially consume s
```

Since these rules apply to inout arguments and mutating self is passed inout,
this also applies to stored fields of self in mutating methods:

```swift
extension MicroServiceRequest {
    /// Read the remaining data and close our read socket.
    mutating func readRemainingData() -> UnsafeRawBufferPointer {
        let data = self.readSocket.read()
        self.readSocket.close() // Error! Cannot partially consume self!
        return data
    }
}
```

One can still take advantage of `self` being passed inout to mutating methods to
retrieve the field by using the `consume` operator on self and reinitializing
`self` before the end of the function:

```swift
extension MicroServiceRequest {
    /// Read the remaining data and close our read socket.
    mutating func readRemainingData() -> UnsafeRawBufferPointer {
        let data = self.readSocket.read()
        (consume self).readSocket.close()
        self = MicroServiceRequest()
        return data
    }
}
```

This successfully compiles without error, but in the process we also are forced
to close the write socket since one has to consume /all/ of self to consume the
read socket.

Another approach would be to create a helper function that takes in the Socket
inout, closes the socket, and reinitializes the memory with an empty Socket:

```swift
extension MicroServiceRequest {
    private static func closeSocket(_ x: inout Socket) {
      x.close()
      x = Socket()
    }

    /// Read the remaining data and close our read socket.
    mutating func readRemainingData() -> UnsafeRawBufferPointer {
        let data = self.readSocket.read()
        consumeSocket(&self.readSocket)
        self = MicroServiceRequest()
        return data
    }
}
```

this again works, but again shows reduced expressivity since we had to create a
separate helper function just to close the socket.

## Proposed solution

Given this reduction in expressivity it is natural to ask... can we improve this
situation by allowing for self to be partially initialized:

```swift
extension MicroServiceRequest {
    mutating func readRemainingData() -> UnsafeRawBufferPointer {
        let data = self.readSocket.read()
        self.readSocket.close()
        self.readSocket = Socket()
        return data
    }
}
```

We propose relaxing these restrictions to allow for code like the above to be
written.

## Detailed design

Swift's consumption and re-initialization rules for noncopyable types will be
changed to be "field sensitive". This means that the language will now allow for
a noncopyable binding to have its noncopyable fields be invalidated on a field
by field basis:

```swift
var request: MicroServiceRequest
let _ = request.readSocket
writeToSocket(request.writeSocket) // This is ok!
```

The specific behavior of these field sensitive invalidation rules vary depending
on whether or not the type of the binding has a deinit. We go through each of
the semantics in the next section below.

For copyable fields, the current behavior of copying the underlying field
without invalidation will remain the unchanged:

```swift
var x = NonCopyableStructWithCopyableField()
let _ = x.copyableField
useK(x.copyableField) // This is ok since we copied x.copyableField above.
```

Given a copyable `borrowing` or`consuming` binding, since the underlying type is
copyable, we know its fields must also be copyable implying that we will just
copy them without invalidating any part of the underlying binding:

```swift
func f(_ x: borrowing CopyableType) {
  let _ = x.copyableField // No invalidation. Copy copyableField
}
func g(_ x: consuming CopyableType) {
  let _ = x.copyableField // No invalidation. Copy copyableField
}
```

### NonCopyable Values without Deinits

A binding without a deinit like `x` above, can be deconstructed and its
remaining fields will be cleaned up at the end of `x`'s maximized lifetime
scope:

```swift
struct S : ~Copyable {
    var noncopyableField1: E
    var noncopyableField2: E
}
var x = S()
if boolTest {
    let _ = x.noncopyableField1 // x.noncopyableField1 is consumed here.
    doSomething()
    // x.noncopyableField2 is destroyed here.
} else {
    doSomething()
    // x is destroyed here.
}
```

Such values can also be fully consumed and then reinitialized in pieces with
only the reinitialized fields being destroyed at the end of the variable's
maximized lifetime scope:

```swift
var x = S()
let _ = consume x
if boolTest {
    x.noncopyableField1 = E()
    doSomething()
    // x.noncopyableField1 is destroyed here. x.noncopyableField2 is left uninitialized.
} else {
    doSomething()
    // x is uninitialized so no destruction occurs.
}
```

This also applies to inout parameters and self in mutating methods,

```swift
extension S : ~Copyable {
    mutating func doSomething() {
        let _ = consume self // Both noncopyableField1 and noncopyableField2 are destroyed.
        self.noncopyableField1 = E()
        self = S() // We only destroy noncopyableField1.
    }
}
```

### NonCopyable Values with Deinits

NonCopyable values with a deinit can only be partially consumed or reinitialized
if:

1. The value is completely reinitialized before end of scope.
2. The `discard` operator is explicitly used to disable the value's deinit.

If there exists a path through the program where neither of the above conditions
are true, the compiler will emit an error:

```swift
struct StructWithDeinit : ~Copyable {
    var noncopyableField1 = E()
    var noncopyableField2 = E()

    deinit { ... }
}

do {
    var s = StructWithDeinit()
    let _ = s.noncopyableField1
} // Error! s has a deinit and is not fully initialized at end of its lifetime.

do {
    var s = StructWithDeinit()
    let _ = s.noncopyableField1
    s.noncopyableField1 = E() // Ok! We reinitialize noncopyableField1 before the end of scope.
}

struct StructWithDeinit2 {
    var noncopyableField1 = E()
    var noncopyableField2 = E()

    deinit { ... }

    consuming func consumeValue() {
        let _ = self.noncopyableField1
    } // Error! self has a deinit but is not fully initialized before end of lifetime

    consuming func consumeValue2() {
        let _ = self.noncopyableField1
        discard self // Ok! We discard self so the deinit will not run.
    }
}
```

The reasons for this behavior is that:

1. Swift requires a value to be completely live at the point in which a deinit
   is applied. This implies if we were to allow for such values to be partially
   initialized at the end of its lifetime, we could not call the deinit. This
   would result in us being forced to clean up the partially initialized value
   in pieces since that is the only thing that we /could/ do.

2. Deinits are used to clean up resources that are uniquely owned (consider a
   file descriptor) and thus in such situations a key part of the API contract
   that an author is providing to the user. Requring only an assignment
   operation to turn off such a deinit would result in code bases where such
   type contracts are simple to break and hard to track down or audit if done in
   error. In contrast, requiring an explicit discard along paths where such
   behavior is desired provides an explicit opt in that avoids such pitfalls.

By requiring the value to be completely initialized (allowing the deinit to be
called) or requiring an explicit discard to be used (making it easy to tell
where deinits are being disabled), we create a programming model where the user
can partially consume/reinit types with deinits in a safe manner with the
compiler's guidance.

### Source stability guarantees and `@frozen`

For copyable types, Swift provides source stability guarantees that allow for a
library author to convert a stored property on a public type to a computed
property and vis-a-versa. This guarantee does not apply naturally to noncopyable
types. We go through each case below:

When we convert a stored property to a computed property, we will be
replacing a partial liveness use of just one of the value's stored fields to
a use of the entire value since a computed property takes self as a fully
live value:

```swift
// Library
public struct E : ~Copyable {}
public struct S : ~Copyable {
    var first: E
    var second: E
}

// Executable
let _ = s.first // Invalidates s.first
let _ = s.second // Invalidates s.second

->

// Library
struct S : ~Copyable {
    var first: E
    var second: E { E() }
}

// Executable
let _ = s.first // Invalidates s.first.
let _ = s.second // Uses all of s when calling the getter s.second. Use after free!
```

In words, the conversion from the stored property to the computed property
causes what was a partial use of `s.second` into a use of all of `s` causing a
use after free violation.

When we convert a computed property to a stored property, we introduce a new
partial invalidation potentially causing later code to stop compiling:

```swift
// Library
struct E : ~Copyable {}
struct S : ~Copyable {
   var first: E { E () }
   func doSomething() { }
}

// Executable
let _ = s.first // We call the s.first the getter.
s.doSomething() // Call s.doSomething()

->

// Library
public struct S : ~Copyable {
   var first: E
   func doSomething() { }
}

// Executable
let _ = s.first // Invalidate s.first
s.doSomething() // Error! s is not completely initialized.
```

In words, by converting the computed property `s.first`, we change a
non-invalidating use of `s` to a use that invalidates `s.first` causing later
uses that require `s` to be entirely alive to no longer be legal.

Given these source compatibility issues, we require library authors in all
compilation modes to explicitly opt public noncopyable types into partial
initialization semantics by attaching the `@frozen` attribute to such types. By
doing this, we are changing the current semantics of `@frozen` when library
evolution is disabled from having no semantic meaning to instead restricting a
type from being changed in the following manners without a major semver
increment:

1. A type's fields being re-ordered.
2. A stored property being converted to a computed property or vis-a-versa.
3. Inserting a new stored property in between two stored properties.

These are the same semantic restrictions that a `@frozen` type has when library
evolution is enabled except that we allow for them to be changed after a semver
increment. By requiring the semver increment, we signal to users of the library
that API stability has been broken by the library.

In order to enforce this when library evolution is disabled, Swift's API checker
will be taught that when a public type marked with `@frozen` is changed in the
above way, one must perform a semver major bump. This type of API checking is
already supported in Xcode and also in the Swift package manager via the command
`swift package diagnose-api-breaking-changes`.

## Source compatibility

This proposal will not have any source compatibility impacts on code that is
already written. All noncopyable code written today do not allow for values to
be partially live implying that we are strictly increasing the set of valid
Swift programs. Due to the new semantics of `@frozen`, libraries written with
library evolution disabled will need to ensure that when they change `@frozen`
types they bump their major semver number.

## ABI compatibility

The invariant that partial consumption of noncopyable types relies upon is that
all stored fields of the noncopyable type must be accessible in the module where
the partial consumption occurs. Naturally this means that when library evolution
is enabled our ability to partially consume types is significantly
limited. Specifically:

1. `@frozen` types regardless of access control level can always be partially
consumed. This includes even frozen types with private fields since the field is
exposed at the ABI level even if it cannot be used directly in source.

2. `public` and `@usableFromInline` types can never be partially consumed
outside of the resilience domain where the type is defined. Since resilience
domains are today limited to the current module, this means that one could not
partially consume outside of the current module.

3. noncopyable types with access control that is `private`, `fileprivate`, or
`internal` without `@usableFromInline` can always have their stored properties
partially consumed.

When library evolution is disabled, we do not have any additional ABI concerns
since ABI stability is not guaranteed implying we can rely on library users
recompiling their code when changes are made to the library.

## Implications on adoption

A library adopter of these features needs to be aware that if one marks a public
type as frozen, one now will have opted into a truly frozen type from a semantic
versioning perspective as talked about in the section above.

## Future directions

### Discard in Mutating Methods

Another common pattern we expect users to want to be able to implement is to be
able to return a struct's internal state via a mutating function without
triggering the deinit. An example of such a case would be a FileDescriptor where
one wishes to return the internal file descriptor state and reinitialize the
FileDescriptor struct with a new default initialization. One cannot implement
such a thing using a mutating method without an additional artificial consuming
function since SE-390 restricts discard to consuming functions:

```swift
struct FileDescriptors : ~Copyable {
    var fd1: Int
    var fd2: Int

    deinit {}

    private consuming func getFD1() -> Int {
        let result = self.fd1
        discard self
        return result
    }
}
```

Since we are relying upon `discard` as one of our ways to allow for partial
consumption, we also loosen the requirements around discard by allowing for
`discard` to be applied to self in mutating methods. Since a mutating method
involves self being passed inout naturally we also allow for `discard`ed
variables in these contexts to be reinitialized after being discarded.

```swift
extension StructWithDeinit2 {
    mutating func test() -> E {
        let result = self.noncopyableField1
        discard self
        self = StructWithDeinit2() // No Deinit Runs
        return result
    }
}
```

### Allow for Partial Invalidation of Fields using `consume`

The `consume` operator currently is not allowed to be applied to fields of
types regardless of copyability:

```swift
var x = CopyableType()
let _ = consume x.k // Error! 'consume' can only be applied to a local binding ('let', 'var', or parameter)
```

We could loosen the restrictions on `consume` by applying the rules around
partial consumption from this proposal. This would apply to noncopyable types
with copyable fields preventing a copy of the copyable field:

```swift
var x = NonCopyableType()
let _ = x.copyableField // copyableField is copied.
let _ = consume x.copyableField // We invalidated copyableField
```

and would also apply to copyable fields of copyable types:

```swift
var x = CopyableTypeWithDeinit()
let _ = consume x.copyableField // We invalidate copyableField
// Need to reinitialize x.copyableField before we call the deinit at end of scope.
```

## Alternatives considered

### Partial invalidation always disables Deinit

We could make it so that any partial consumption or initialization would disable
the deinit. This would work against Swift's goals of being a safe easy to use
language since we would be introducing a very easy way to break a library
invariant that would be hard to audit in comparison to deinit:

```swift
ADD EXAMPLE HERE
```

### Partial Consumption outside of Methods

We could be even more restrictive and only allow for partial consumption of
noncopyable types inside methods. The argument in favor of this approach is that
the author of a type has the greatest understanding of the invariants of the
type and the impact of a value being consumed and thus self being invalid. The
argument against this is that the compiler will prevent any such misuses via
noncopyable diagnostics. For example if one were to call any method on the
partially consumed noncopyable type, we would get an error. So even if a user of
a type made such a mistake, it would never actually result in a valid
program. So we would be giving up expressivity without any real gain.

### Forcing Full Initialization of Values after Partial Consumption

We could force a binding to be completely initialized using an init after
partial consumption:

```swift
var s = S()
let _ = consume s
s.noncopyableField1 = E() // Error! Can only reinitialize s by invoking s's initializer
s = S() // Ok! We are reinitializing s with a value by calling its init
```

The reason why this was proposed was that often times an init creates specific
invariants and expectations in the type. By allowing for a type to be partially
reinitialized after full consumption, we could allow for those invariants to be
broken. After some discussion it was realized that this is actually programmer
error due to an encapsulation issue that would also occur given a copyable
type. Consider a copyable type with an init that enforces invariants. If the
copyable type exposes the fields that maintain that invariant to outside users,
the copyable type's invariants could also be broken. If the user wants to
maintain these invariants, it needs to hide the internal stored property and use
a computed property to maintain these invariants. There was agreement that the
init issue was not a real issue since if the author exposed a stored property to
code outside the type, then it was actually the author's programming error since
to maintain the invariant, they should not have exposed the stored field.

## Acknowledgments

Thanks to Kavon, JoeG, and many others.
