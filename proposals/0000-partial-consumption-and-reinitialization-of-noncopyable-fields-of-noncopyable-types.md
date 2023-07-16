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

Given a var like construct (for example: var, inout), Swift does not allow for a stored
field of the type to be partially consumed or initialized:

```swift
struct E : ~Copyable {}
struct S : ~Copyable {
    var e1: E
    var e2: E
}

var s = S()
let _ = s.e1 // Error! Cannot partially consume s
```

Since these rules apply to inouts, this also applies to stored fields of self in
mutating methods:

```swift
extension S {
    mutating func doSomething() {
        let _ = self.e1 // Error! Cannot partially consume self
    }
}
```

One can still pass the field inout to take the field out by using the consume
operator:

```swift
extension S {
    mutating func doSomething() {
        let _ = (consume self).e1
        self = S()
    }
}
```

This work but exhibits a significant reduction in expressivity since one has to
consume /all/ of self causing one to be unable to access the rest of the fields
of self later in the function:

```swift
extension S {
    mutating func doSomething() {
        let _ = (consume self).e1
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
        let _ = e1
        print(k) // I can still print k!
        e1 = E() // Reinitialize e so self is fully initialized at end of
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
be done on a field by field basis:

```swift
var x = S()
let _ = x.e1
useE2(x.e2) // This is ok!
```

There is different behavior depending on whether or a binding does not have a
deinit like `x : S` does or if it has a non-trivial deinit. We go through each
below:

### NonCopyable Values without Deinits

A binding without a deinit like `x` above, can be deconstructed and its
remaining fields will be cleaned up at the end of `x`'s maximized lifetime
scope:

```swift
var x = S()
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
var x = S()
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
extension S : ~Copyable {
    mutating func doSomething() {
        let _ = consume self // Both e1 and e2 are destroyed.
        self.e1 = E1()
        self = S() // We only destroy e1.
    }
}
```

### NonCopyable Values with Deinits

NonCopyable values with a non-trivial deinit can only be partially consumed or
reinitialized if:

1. The value is completely reinitialized before end of scope.
2. The `discard` operator is explicitly used to disable the value's deinit.

If there exists a path through the program where neither of the above conditions
are true, the compiler will emit an error explaining to the the user that the
value must be either discarded or fully reinitialized before the end of its
lifetime:

```swift
struct StructWithDeinit : ~Copyable {
    var e1 = E()
    var e2 = E()

    deinit { ... }
}

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

### Source stability guarantees and `@frozen`

For copyable types, Swift provides source stability guarantees that allow for a
library author to convert a stored property on a public type to a computed
property and vis-a-versa. This guarantee does not apply naturally to noncopyable
types due to the above invalidation rules.

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

By requiring the semver increment when a `@frozen` type is compiled in such a
way, we signal to users of the library that API stability has been broken by the
library.

In order to enforce this when library evolution is disabled, we will teach
Swift's API checker to know that when a public type marked with `@frozen` is
changed in the above way, one must perform a semver major bump. This type of API
checking is already supported in Xcode and also in the Swift package manager via
the command `swift package diagnose-api-breaking-changes`.

## Source compatibility

This proposal will not have any source compatibility impacts on code that is
already written. All noncopyable code written today do not allow for values to
be partially live implying that we are strictly increasing the set of valid
Swift programs. Due to the new semantics of `@frozen`, libraries written with
library evolution disabled will not need to ensure that when they change
`@frozen` types they bump their major semver number.

## ABI compatibility

The invariant that partial consumption of noncopyable types relies upon is that
all stored fields of the noncopyable type must be accessible in the module where
the partial consumption occurs. Naturally this means that in library evolution
our ability to partially consume types is significantly limited. Specifically:

1. `@frozen` types regardless of access control level can always be partially
consumed. This includes even frozen types with private fields since even though
the private field can not be used directly it is still exposed at the ABI level.

2. `public` and `@usableFromInline` types can never be partially consumed
outside of the resilience domain where the type is defined. Since resilience
domains are today limited to the current module, this means that one could not
partially consume outside of the current module.

3. `internal` types that are not `@usableFromInline`, `private`, and
`fileprivate` noncopyable types can always have their stored properties
partially consumed.

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
