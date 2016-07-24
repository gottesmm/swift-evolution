# Move AutoreleasingUnsafeMutablePointer from StdlibCore to Objective C overlay

* Proposal: [SE-NNNN](XXXX-Move-autoreleasing-pointer-to-objc-overlay.md)
* Author: [gottesmm](https://github.com/gottesmm)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

AutoreleasingUnsafeMutablePointer is part of a cohort of Objective C specific code in the Swift Standard Library that is conditionally compiled depending on whether StdlibCore is compiled with the ObjC flag enabled. If designed today much of this code would be placed into the Objective C overlay. This document proposes incremental work towards the goal of moving such code to the Objective C overlay by moving the public API of AutoreleasingUnsafeMutablePointer to the Objective C overlay from StdlibCore. The change is being proposed now for Swift 3 since this change is an API break (albeit a small one).

Swift-evolution thread: [[Proposal] Move public AutoreleasingUnsafeMutablePointer API from StdlibCore -> Objective C Overlay](http://thread.gmane.org/gmane.comp.lang.swift.evolution/24631)

## Motivation

A core principle of StdlibCore development is that StdlibCore should be as platform and os independent as possible. This implies attempting to eliminating code that is conditionally compiled depending on the platform and os when appropriate and possible. The reason why eliminating such a code is a general "good" is:

1. Conditionally compiled code can be difficult to reason about since often times large sections of code are conditionally compiled and the actual \#if/\#endif markers are only on the boundary. This especially comes into play when searching code for specific text.

2. Working on StdlibCore for any platform P where the relevant code is conditionally compiled out if the target platform is not a platform Q becomes more difficult since:

   a. Significant chucks of the file are irrelevant to P causing a spaghetti code like effect.

   b. When working on P, it becomes very easy to by mistake break Q via conflicting names, methods.

3. Additional unnecessary work is put onto the parser since all conditionally compiled code in Swift must still be parsed. As a code base increases in size this can become a significant impact on compile time. As an example of this consider the effort put into the vectorization of comment stripping in Clang.

Currently a large part of such conditionally compiled code is

Eliminating conditionally compiled code moves One technique used to do this is to sink the conditionally compiled code into a conditionally compiled overlay and use
extensions to provide the functionality of the formerly conditionally compiled
code. Eliminating conditionally compiled code and centralizing such code in an
overlay:

1. Shrinks the source code size of StdlibCore making it easier to reason about.
2. Shrinks The API delta in between StdlibCore on different platforms.
3. Eliminates visual noise from StdlibCore when reasoning about platforms that
   do not support Objective C.
4. Moves the conditionally compiled code to a centralized overlay. Thus instead
   of having to search in StdlibCore for conditionally compiled code and reason
   if the code is enabled or not, one can just go straight to the overlay where
   all of the code is static.
5. Shrinks the API delta in between StdlibCore on different platforms

Today there is a large amount of conditionally compiled code in StdlibCore that
ideally should be in the Objective C overlay from a code organization
perspective. Cleaning up this code would eliminate spaghetti code from
StdlibCore and 

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

