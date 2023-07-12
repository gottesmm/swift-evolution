# Partial Consumption and Reinitialization of Noncopyable fields of Noncopyable Types

* Proposal: [SE-????](0000-partial-consumption-and-reinitialization-of-noncopyable-fields-of-noncopyable-types.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Pitch with implementation on main behind a flag**
* Upcoming Feature Flag: `MoveOnlyPartialConsume`

## Introduction

SE-390 defines all noncopyable types as being either [fully initialized or fully destroyed outside of initializers](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md#finer-grained-destructuring-in-consuming-methods-and-deinit).
This reduces language expressivity by preventing a field of a noncopyable
binding from being consumed without fully consuming the entire binding. A
particularly annoying case where this restriction is noticeable is self in
mutating methods. We would like to loosen the language rules to allow for
partial consumption and initialization in these cases.

## Motivation

Given a var like construct (e.x.: var, inout), Swift does not allow for a stored
field of the type to be partially consumed or initialized:

```swift
struct E : ~Copyable {}
struct S : ~Copyable {
    var first: E
    var second: Klass
}

var s = S()
let _ = s.e // Error! Cannot partially consume s
```

Since these rules apply to inouts, this also applies to stored fields of self in
mutating methods. E.x.:

```swift
extension S {
    mutating func doSomething() {
        let _ = self.e // Error! Cannot partially consume self
    }
}
```

One can still of course pass the field inout to take the field out by using the
consume operator:

```swift
extension S {
    mutating func doSomething() {
        let _ = (consume self).e
        self = S()
    }
}
```

This work but exhibits a significant reduction in expressivity since one has to
consume /all/ of self causing one to be unable to access the rest of the fields
of self later in the function. E.x.:

```swift
extension S {
    mutating func doSomething() {
        let _ = (consume self).e
        print(k) // Error! self already consumed!
        self = S()
    }
}
```

## Proposed solution

Given this reduction in expressivity it is natural to ask... can we improve this
situation by allowing for self to be partially initialized:

```swift
extension S {
    mutating func doSomething() {
        let _ = e
        print(k) // I can still print k!
        e = E() // Reinitialize e so self is fully initialized at end of
                // doSomething()
    }
}
```

We propose relaxing these restrictions to allow for code like the above to be
written.

## Detailed design

Swift's consumption and re-initialization rules for noncopyable types will be
changed to be "field sensitive". This means that instead of only allowing for a
type to be consumed or reinitialized entirely, the language allows for this to
be done on a field by field basis, e.x.:

```swift
struct E1 : ~Copyable {}
struct E2 : ~Copyable {}
struct StructWithTrivialDeinit : ~Copyable {
    var e1 = E1()
    var e2 = E2()
}

var x = StructWithTrivialDeinit()
let _ = x.e1
useE2(e2) // This is ok!
```

There is different behavior depending on whether or a binding has a trivial
deinit like `x` does or if it has a non-trivial deinit. We consider these cases
separately.

### NonCopyable Values with Trivial Deinits

A binding with a trivial deinit like `x` above, can be deconstructed and its
remaining fields will be cleaned up at the end of `x`'s maximized lifetime
scope, e.x.:

```swift
var x = StructWithTrivialDeinit()
if boolTest {
    let _ = x.e1 // x.e1 is consumed here.
    doSomething()
    // x.e2 is destroyed here.
} else {
    doSomething()
    // x is destroyed here.
}
```

Such values can also be fully consumed and then reinitialized in pieces with
only the reinitialized fields being destroyed at the end of the variable's
maximized lifetime scope:

```swift
var x = StructWithTrivialDeinit()
let _ = consume x
if boolTest {
    x.e1 = E1()
    doSomething()
    // x.e1 is destroyed here. x.e2 is left uninitialized.
} else {
    doSomething()
    // x is uninitialized so no destruction occurs.
}
```

This also applies to inout parameters and self in mutating methods,

```swift
extension StructWithTrivialDeinit : ~Copyable {
    mutating func doSomething() {
        let _ = consume self // Both e1 and e2 are destroyed.
        self.e1 = E1()
        self = S() // We only destroy e1.
    }
}
```

### NonCopyable Values with Non-Trivial Deinits and Discard

NonCopyable values with a non-trivial deinit can only be partially consumed or
reinitialized if:

1. The value is completely reinitialized before end of scope.
2. The `discard` operator is explicitly used to disable the value's deinit.

If both of the above conditions are not true, the compiler will emit an error
telling the user that the value must be either discarded or fully reinitialized
before the end of its lifetime, e.x.:

```swift
do {
    var s = StructWithDeinit()
    let _ = s.e1
} // Error! s has a deinit and is not fully initialized at end of its lifetime.

do {
    var s = StructWithDeinit()
    let _ = s.e1
    s.e1 = E1() // Ok! We reinitialize e1 before the end of scope.
}

struct StructWithDeinit2 {
    var e1 = E1()
    var e2 = E2()

    deinit { ... }

    consuming func consumeValue() {
        let _ = e1
    } // Error! self has a deinit but is not fully initialized before end of lifetime

    consuming func consumeValue2() {
        let _ = e1
        discard self // Ok! We discard self so the deinit will not run.
    }
}
```

The reasons for this behavior is that:

1. Swift requires a value to be completely live at the point in which a deinit
   is applied. This implies if we were to allow for such values to be partially
   initialized, we would necessarily have to destroy the initialized fields of
   the type and not call the deinit.

2. Deinits are used to clean up resources that are uniquely owned (consider a
   file descriptor) and thus in such situations a key part of the API contract
   that an author is providing to the user. If an assignment operation is all
   that was required to turn off such a deinit, it would create an easy way to
   break a type's API contract in a manner that would be difficult to audit or
   to track down in a large project.

By requiring the value to be completely initialized (allowing the deinit to be
called) or requiring an explicit discard to be used (making it easy to tell
where deinits are being disabled), we create a programming model where the user
can partially consume/reinit types with deinits in a safe manner with the
compiler's guidance.

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
        let result = fd1
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
    mutating func test() -> E1 {
        let result = e1
        discard self
        self = StructWithDeinit2() // No Deinit Runs
        return result
    }
}
```

## Source compatibility

This proposal will not have any source compatibility impacts on code that is
already written. All noncopyable code written today do not allow for values to
be partially live implying that we are strictly increasing the set of valid
Swift programs. That being said, this proposal does affect the ability for
library maintainers to break source in the future when converting a noncopyable
stored property to a noncopyable computed property and vis-a-versa.

When we convert a stored property to a computed property, we will be
replacing a partial liveness use of just one of the value's stored fields to
a use of the entire value since a computed property takes self as a fully
live value. E.x.:

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

When we convert a computed property to a stored property, we introduce a new
partial invalidation potentially causing later code to stop compiling. E.x.:

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

These source compatibility concerns suggest that library authors should be
forced to explicitly opt public noncopyable types into being able to be
partially initialized by external users of their library. We propose that we
repurpose the attribute `@frozen` for this purpose in all compilation modes
since `@frozen` already has these implications when library evolution is
enabled.

In order to ensure that we are not introducing a new dialect into the language,
we will change Swift's API checking capability to know that even when library
evolution is disabled, a type marked with `@frozen` is not allowed to change its
layout by reordering fields or inserting fields in between other fields. This
will ensure that library authors have a mechanism to know that an API
incompatibility has been introduced and a semver major version increment is
necessary to inform downstream users of the library. This is already able to be
done in Xcode and support will be added into the Swift package manager for
maintaining API stability json files.

## ABI compatibility

The clear invariant that partial consumption of noncopyable types relies upon is
that all stored fields of the noncopyable type must be accessible in the module
where the partial consumption occurs. Naturally this means that in library
evolution our ability to partially consume types is significantly
limited. Specifically:

1. Frozen types regardless of access control level can always be partially
consumed. This includes even frozen types with private fields since even though
the private field is not available to be used it is still exposed at the ABI
level.

2. Public and usableFromInline types can never be partially consumed outside of
the resilience domain where the type is defined. Since resilience domains are
today limited to the current module, this means that one could not partially
consume outside of the current module.

3. Internal types that are not usableFromInline, private, and fileprivate
noncopyable types can always have their stored properties partially consumed.

## Implications on adoption

The compatibility sections above are focused on the direct impact
of the proposal on existing code.  In this section, describe issues
that intentional adopters of the proposal should be aware of.

For proposals that add features to the language or standard library,
consider whether the features require ABI support.  Will adopters need
a new version of the library or language runtime?  Be conservative: if
you're hoping to support back-deployment, but you can't guarantee it
at the time of review, just say that the feature requires a new
version.

Consider also the impact on library adopters of those features.  Can
adopting this feature in a library break source or ABI compatibility
for users of the library?  If a library adopts the feature, can it
be *un*-adopted later without breaking source or ABI compatibility?
Will package authors be able to selectively adopt this feature depending
on the tools version available, or will it require bumping the minimum
tools version required by the package?

If there are no concerns to raise in this section, leave it in with
text like "This feature can be freely adopted and un-adopted in source
code with no deployment constraints and without affecting source or ABI
compatibility."

## Alternatives considered

### Partial Liveness always disables Deinit

We could make it so that any partial consumption or initialization would disable
the deinit. This would work against Swift's goals of being a safe easy to use
language since we would be introducing a very easy way to break a library
invariant that would be hard to audit in comparison to deinit.

### Partial Consumption outside of Methods

We could be even more restrictive and only allow for partial consumption of
noncopyable types inside methods. The argument in favor of this approach is that
the author of a type has the greatest understanding of the invariants of the
type and the impact of a value being consumed and thus self being invalid. The
argument against this is that the move checker will prevent any such misuses,
e.x.: if one were to call any method on the partially consumed noncopyable type,
we would get an error. So even if a user of a type made such a mistake, it would
never actually result in a valid program. So we would be giving up expressivity
without any real gain.

### Forcing Full Initialization of Values after Partial Consumption

We could force a binding to be completely initialized using an init after
partial consumption:

```swift
var s = S()
let _ = consume s
s.e1 = E1() // Error! Can only reinitialize s by invoking s's initializer
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

