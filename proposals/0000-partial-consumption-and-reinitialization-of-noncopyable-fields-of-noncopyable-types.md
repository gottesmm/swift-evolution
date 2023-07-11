# Partial Consumption and Reinitialization of Noncopyable fields of Noncopyable Types

* Proposal: [SE-????](0000-partial-consumption-and-reinitialization-of-noncopyable-fields-of-noncopyable-types.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Pitch with implementation on main behind a flag**
* Upcoming Feature Flag: `MoveOnlyPartialConsume`

## Introduction

Currently noncopyable fields of Noncopyable types are currently not allowed at
all under SE-390. This reduces expressivity around destructuring noncopyable
values including self in mutating functions. We would like to loosen the
language rules here to allow for this increase in expressivity.

## Motivation

Currently given a var like construct (e.x.: var, inout), Swift does not allow
for a stored field of the type to be partially consumed:

```swift
struct E : ~Copyable {}

struct S : ~Copyable {
    var first: E
    var second: Klass
}

var s = S()
let _ = s.e // Error! Cannot partially consume s
```

Since this applies to inouts, this also applies to stored fields of self in
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
fields of self later in the function. 

## Proposed solution

Given this reduction in expressivity it is natural to ask... can we improve this
situation by allowing for partial consumption of self:

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

We can do this and in fact, the Swift compiler already has support for this,
albeit turned off! The reason that it is turned off is that we realized that
there is design space here that we wanted to explore and that when the
noncopyable proposal went through evolution we labeled partial consumption
explicitly as an extension of the proposal. 

## Detailed design

We consider below a few different axes in the design space:

1. On types with deinits
2. When Library Evolution is enabled
3. When Library Evolution is disabled
4. In either case, whether or not we should allow for partial consumption only
   in methods.

### Partial Consumption on types with Deinits

We ban partial consumption of noncopyable types with deinits since when a field
is partially consumed, we are allowing for the type to be destroyed in
parts. For example:

```swift
struct E : ~Copyable
struct S : ~Copyable {
   var first: E
   var second: E
   deinit {}
}

var s = S()
let _ = s.first // s.first is destroyed here
doSomething()
// s.second is destroyed here
```

Since s here has been partially consumed, we never destroy it all together
implying that we never would call its own deinit implying that we must not allow
it. Note that even though we do not allow this, we still allow for authors in
consuming methods to use the discard operator to turn off the deinit of the
value and then partially deconstruct the value.

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

<!--
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

Describe the impact on ABI compatibility.  As a general rule, the ABI
of existing code must not change between tools releases or language
modes.  This rule does not apply as often as source compatibility, but
it is much stricter, and the Language Workgroup generally cannot allow
exceptions.

The ABI encompasses all aspects of how code is generated for the
language, how that code interacts with other code that has been
compiled separately, and how that code interacts with the Swift
runtime library.  Most ABI changes center around interactions with
specific declarations.  Proposals that do not affect how code is
generated to interact with an external declaration usually do not
have ABI impact.

For proposals that affect general code generation rules, consider
the impact on code that's already been compiled.  Does the proposal
affect declarations that haven't explicitly adopted it, and if so,
does it change ABI details such as symbol names or conventions
around their use?  Will existing code change its dynamic behavior
when running against a new version of the language runtime or
standard library?  Conversely, will code compiled in the new way
continue to run on old versions of the language runtime or standard
library?

For proposals that affect the standard library, consider the impact
on any existing declarations.  As above, does the proposal change symbol
names, conventions, or dynamic behavior?  Will newly-compiled code work
on old library versions, and will new library versions work with
previously-compiled code?

This section will often end up very short.  A proposal that just
adds a new standard library feature, for example, will usually
say either "This proposal is purely an extension of the ABI of the
standard library and does not change any existing features" or
"This proposal is purely an extension of the standard library which
can be implemented without any ABI support" (whichever applies).
Nonetheless, it is important to demonstrate that you've considered
the ABI implications.

If the design of the feature was significantly constrained by
the need to maintain ABI compatibility, this section is a reasonable
place to discuss that.

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
-->
