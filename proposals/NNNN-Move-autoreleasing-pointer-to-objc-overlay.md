# Move AutoreleasingUnsafeMutablePointer from StdlibCore to Objective C overlay

* Proposal: [SE-NNNN](NNNN-Move-autoreleasing-pointer-to-objc-overlay.md)
* Author: [Michael Gottesman](https://github.com/gottesmm)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

AutoreleasingUnsafeMutablePointer is part of a cohort of Objective C specific code in the Swift Standard Library that is conditionally compiled depending on whether StdlibCore is compiled with the ObjC flag enabled. If designed today much of this code would be placed into the Objective C overlay. This document proposes incremental work towards the goal of moving such code to the Objective C overlay by moving the public API of AutoreleasingUnsafeMutablePointer to the Objective C overlay from StdlibCore. The change is being proposed now for Swift 3 since this change is an API break (albeit a small one).

Swift-evolution thread: [[Proposal] Move public AutoreleasingUnsafeMutablePointer API from StdlibCore -> Objective C Overlay](http://thread.gmane.org/gmane.comp.lang.swift.evolution/24631)

## Motivation

A core principle of StdlibCore development is that StdlibCore should be as platform and os independent as possible. This implies attempting to eliminating code that is conditionally compiled depending on the platform and os when appropriate and possible. The reason why eliminating such a code is a general "good" is:

1. Conditionally compiled code can be difficult to reason about since often times large sections of code are conditionally compiled and the actual \#if/\#endif markers are only on the boundary. This especially comes into play when searching code for specific text.

2. Given two distinct platforms P and Q, attempting to make changes to a StdlibCore file F for P in the presense of code conditionally compiled only on platform Q results in the following difficulties:

   a. Significant blocks of code in F are irrelevant to P causing a spaghetti code like effect.

   b. When working on P, it becomes very easy to by mistake break Q via conflicting names, methods.

3. API deltas are created in between StdlibCore on different platforms. Reducing the size of such deltas eases reasoning about the overall API of StdlibCore by eliminating special cases that must be remembered by contributors.

4. Since all conditionally compiled code in Swift is parsed, additional unnecessary work must be done by the Swift Frontend and the OS. While this may seem uninteresting, consider that significant speedups were seen in Clang's compile time by the vectorization of comment stripping. In this case we are not just ignoring blocks of code, we are actually parsing the code and then throwing it away. Especially as StdlibCore gets larger, this will become a more significant problem.

## Proposed solution

Describe your solution to the problem. Provide examples and describe
how they work. Show how your solution is better than current
workarounds: is it cleaner, safer, or more efficient?

## Detailed design

Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.

## Impact on existing code

Describe the impact that this change will have on existing code. Will some
Swift applications stop compiling due to this change? Will applications still
compile but produce different behavior than they used to? Is it
possible to migrate existing Swift code to use a new feature or API
automatically?

## Alternatives considered

Editors could help alleviate this issue through code folding, but why work
around such an issue if it is possible to just eliminate such code?

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

