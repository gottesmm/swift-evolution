# Partial Consumption and Reinitialization of Noncopyable fields of Noncopyable Types

* Proposal: [SE-????](0000-partial-consumption-and-reinitialization-of-noncopyable-fields-of-noncopyable-types.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Pitch with implementation on main behind a flag**
* Upcoming Feature Flag: `MoveOnlyPartialConsume`

## Introduction

In SE-390, all noncopyable types are defined as being either [fully initialized or fully destroyed outside of initializers](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md#finer-grained-destructuring-in-consuming-methods-and-deinit).
This reduces language expressivity by preventing a field of a noncopyable
binding from being consumed without fully consuming the entire binding. This is
especially noticable in the case of self in mutating methods. We would like to
loosen the language rules to allow for partial consumpition and initialization
in these cases.

## Motivation

Currently given a var like construct (e.x.: var, inout), Swift does not allow
for a stored field of the type to be partially consumed or initialized:

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

That being said, one can still of course pass the field inout to take the field
out by using the consume operator:

```swift
extension S {
    mutating func doSomething() {
        let _ = (consume self).e
        self = S()
    }
}
```

while this works, it is a significiant reduction in expressivity since one has
to consume /all/ of self causing one to be unable to access the rest of the
fields of self later in the function. E.x.:

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

The main semantic change to the language is that the consumption and
reinitialization rules of noncopyable types will become "field sensitive". This
means that instead of only allowing for a type to be consumed or reinitialized
entirely, the language allows for this to be done on a field by field basis,
e.x.:

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

Bindings that are of a type with a trivial deinits like `x` above, can be
deconstructed and its remaining fields will be cleaned up at the end of `x`'s
maximized lifetime scope, e.x.:

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

If one of the above conditions is not true, the compiler will emit an error
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

The reason for these considerations is that:

1. Swift requires a value to be completely live at the point in which a deinit
   is applied [(*)](#footnote-1). This implies if we were to allow for such
   values to be partially initialized, we would necessarily have to destroy the
   initialized fields of the type and not call the deinit.

2. Deinits are used to clean up resources that are uniquely owned (consider a
   file descriptor) and thus in such situations a key part of the API contract
   that an author is providing to the user. If an assignment operation is all
   that was required to turn off such a deinit, it would create an easy way to
   break a type's API contract in a manner that would be difficult to audit or
   to track down in a large project.

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

Describe the impact of this proposal on source compatibility.  As a
general rule, all else being equal, Swift code that worked in previous
releases of the tools should work in new releases.  That means both that
it should continue to build and that it should continue to behave
dynamically the same as it did before.  Changes that cannot satisfy
this must be opt-in, generally by requiring a new language mode.

This is not an absolute guarantee, and the Language Workgroup will
consider intentional compatibility breaks if their negative impact
can be shown to be small and the current behavior is causing
substantial problems in practice.

For proposals that affect parsing, consider whether existing valid
code might parse differently under the proposal.  Does the proposal
reserve new keywords that can no longer be used as identifiers?

For proposals that affect type checking, consider whether existing valid
code might type-check differently under the proposal.  Does it add new
conversions that might make more overload candidates viable?  Does it
change how names are looked up in existing code?  Does it make
type-checking more expensive in ways that might run into implementation
limits more often?

For proposals that affect the standard library, consider the impact on
existing clients.  If clients provide a similar API, will type-checking
find the right one?  If the feature overloads an existing API, is it
problematic that existing users of that API might start resolving to
the new API?

## ABI compatibility

### Partial Consumption when Library Evolution is enabled

The clear invariant that partial consumption of noncopyable types relies upon is
that all stored fields of the noncopyable type must be accessible in the module
where the partial consumption occurs. Naturally this means that in library
evolution our ability to partially consume types is significantly
limited. Specifically:

Frozen types regardless of access control level can always be partially
consumed. This includes even frozen types with private fields since even
though the private field is not available to be used it is still exposed at
the ABI level.

Public and usableFromInline types can never be partially consumed outside of
the resilience domain where the type is defined. Since resilience domains are
today limited to the current module, this means that one could not partially
consume outside of the current module.

Internal types that are not usableFromInline, private, and fileprivate
noncopyable types can always have their stored properties partially consumed.

### Partial Consumption when Library Evolution is disabled

When we compile without library evolution, from an ABI perspective we have
everything that we need to always partially consume even public types since when
library evolution is disabled all types have a frozen ABI. But we have
additional Source Compatibility constraints to consider: we have always allowed
for authors to convert fields from being stored to computed and back. This
creates source stability issues since:

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

These source compatibility concerns imply that even in non-ABI stable libraries
we do not want to allow for public noncopyable types to be partially consumed by
default. This suggests that we may want to add the ability for a library author
to explicitly notate such public types that they are giving up these source
compatibility properties. A natural way to do this is to endow @frozen with
this meaning when applied to public noncopyable types in libraries without
library evolution enabled. As an additional benefit, by extending the meaning of
frozen in this manner, we prevent an additional difference in between Swift when
compiled with/without library evolution enabled: in both language modes, public
noncopyable types can only be partially consumed outside of their current module
if they have @frozen attached.

### Partial Consumption when Library Evolution is enabled

The clear invariant that partial consumption of noncopyable types relies upon is
that all stored fields of the noncopyable type must be accessible in the module
where the partial consumption occurs. Naturally this means that in library
evolution our ability to partially consume types is significantly
limited. Specifically:

Frozen types regardless of access control level can always be partially
consumed. This includes even frozen types with private fields since even
though the private field is not available to be used it is still exposed at
the ABI level.

Public and usableFromInline types can never be partially consumed outside of
the resilience domain where the type is defined. Since resilience domains are
today limited to the current module, this means that one could not partially
consume outside of the current module.

Internal types that are not usableFromInline, private, and fileprivate
noncopyable types can always have their stored properties partially consumed.

### Partial Consumption when Library Evolution is disabled

When we compile without library evolution, from an ABI perspective we have
everything that we need to always partially consume even public types since when
library evolution is disabled all types have a frozen ABI. But we have
additional Source Compatibility constraints to consider: we have always allowed
for authors to convert fields from being stored to computed and back. This
creates source stability issues since:

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

These source compatibility concerns imply that even in non-ABI stable libraries
we do not want to allow for public noncopyable types to be partially consumed by
default. This suggests that we may want to add the ability for a library author
to explicitly notate such public types that they are giving up these source
compatibility properties. A natural way to do this is to endow @frozen with
this meaning when applied to public noncopyable types in libraries without
library evolution enabled. As an additional benefit, by extending the meaning of
frozen in this manner, we prevent an additional difference in between Swift when
compiled with/without library evolution enabled: in both language modes, public
noncopyable types can only be partially consumed outside of their current module
if they have @frozen attached.

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

## Future directions

Describe any interesting proposals that could build on this proposal
in the future.  This is especially important when these future
directions inform the design of the proposal, for example by making
sure an attribute encodes enough information to be used for other
purposes.

The rest of the proposal should generally not talk about future
directions except by referring to this section.  It is important
not to confuse reviewers about what is covered by this specific
proposal.  If there's a larger vision that needs to be explained
in order to understand this proposal, consider starting a discussion
thread on the forums to capture your broader thoughts.

Avoid making affirmative statements in this section, such as "we
will" or even "we should".  Describe the proposals neutrally as
possibilities to be considered in the future.

Consider whether any of these future directions should really just
be part of the current proposal.  It's important to make focused,
self-contained proposals that can be incrementally implemented and
reviewed, but it's also good when proposals feel "complete" rather
than leaving significant gaps in their design.  For example, when
[SE-0193](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md)
introduced the `@inlinable` attribute, it also included the
`@usableFromInline` attribute so that declarations used in inlinable
functions didn't have to be `public`.  This was a relatively small
addition to the proposal which avoided creating a serious usability
problem for many adopters of `@inlinable`.

## Alternatives considered

The reason why we take this approach is that:

Rather than introduce such an issue to the language, we instead use the discard 
Luckily for us, we have a solution to this problem: the discard operator! The
discard operator provides a manner for us to turn off the deinit of a type in a
way that is explicit and gives appropriate emphasis to the reader of the code
that the deinit is being disabled and a key type contract is being broken.

Thus we 

```swift
struct StructWithDeinit : ~Copyable {
    var e1 = E1()
    var e2 = E2()

    deinit { ... }
}

do {
    var s = StructWithDeinit()
    let _ = s.e1
    // We 
}
```

<a name="footnote-1">(*)</a>: This is contrast to languages like C where it is
safe to pass an uninitialized pointer to a function as long as one does not
access any memory through the pointer.

### Partial Consumption outside of Methods

The final axis to consider is whether or not we should be even more restrictive
and only allow for partial consumption of noncopyable types inside methods. The
argument in favor of this approach is that the author of a type has the greatest
understanding of the invariants of the type and the impact of a value being
consumed and thus self being invalid. The argument against this is that the move
checker will prevent any such misuses, e.x.: if one were to call any method on
the partially consumed noncopyable type, we would get an error. So even if a
user of a type made such a mistake, it would never actually result in a valid
program. So we would be giving up expressivity without any real gain.

Describe alternative approaches to addressing the same problem.
This is an important part of most proposal documents.  Reviewers
are often familiar with other approaches prior to review and may
have reasons to prefer them.  This section is your first opportunity
to try to convince them that your approach is the right one, and
even if you don't fully succeed, you can help set the terms of the
conversation and make the review a much more productive exchange
of ideas.

You should be fair about other proposals, but you do not have to
be neutral; after all, you are specifically proposing something
else.  Describe any advantages these alternatives might have, but
also be sure to explain the disadvantages that led you to prefer
the approach in this proposal.

You should update this section during the pitch phase to discuss
any particularly interesting alternatives raised by the community.
You do not need to list every idea raised during the pitch, just
the ones you think raise points that are worth discussing.  Of course,
if you decide the alternative is more compelling than what's in
the current proposal, you should change the main proposal; be sure
to then discuss your previous proposal in this section and explain
why the new idea is better.

## Acknowledgments

Thanks to Kavon, JoeG, and many others.

