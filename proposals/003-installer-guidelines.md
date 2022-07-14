## Introduction

The Haskell Foundation would like see a central Haskell download page in which the Haskell curious
could easily download and run a installer to provide a standard Haskell environment that supports
the principal build tools (`cabal-install` and `stack`) on the main platforms (Linux, macOS and
Windows). All Haskell introductory material (text books, blog posts, etc.) should be able to point
to the Haskell Foundation download page and work on the assumption that it will yield their
widely-supported Haskell development environment.

To this end this proposal covers two things:

  * To publish a set of minimal requirements for a Haskell installer that we are prepared to
    recommend along with a roadmap of the minimal requirements that we are looking to add in the
    future. (The initial requirements will not require precompiled libraries or editor/IDE
    environments but we would like to move in that direction, for example.)

  * To provide a prototype Windows installer for cabal-install-based Haskell development
    environments based on the stack codebase. This is the coverage we are missing at the moment and
    it is easy to do. We anticipate it becoming quickly obsoleted as the more stablished installers
    cover this space, but by providing this we immediately guarantee its coverage.

  * To provide a simplified Haskell download page that prominently displays installers that
    meet our guidelines.

On the basis of early discussions with the principal stakeholders we believe that it is possible to
quickly agree an initial set of requirements and build a Windows installer.


## Background and Motivation

The exact configuration of Haskell development environments has recently been a contentious issue
leading to regrettable fragmentation and confusion. It should be a priority of the Haskell
Foundation to lead the way in restoring harmony for the health of the Haskell community and its
ecosystem.  The Haskell Foundation should also be well placed to do this with its broad
representation at board level and many sponsors and friends in the community.

## Goals

The goals are:

  * A uniform expectation of what constitutes a minimal Haskell development environment across the
    community.

  * A Haskell download page for getting said Haskell development environment for Linux, Windows or
    macOS.

## People

  * Chris Dornan to coordinate, draft the original requirements and edit them according to feedback.

  * Julian Ospald to cover the requirements from a `ghcup` perspective.

  * Michael Snoyman to cover the requirements from a `stack` perspective and deliver a prototype
    Windows installer based on the Stack code base.

  * Emily Pillmore to help with everything, but especially adding the download page.

  * Andrew Boardman to oversee everything, ensuring that things are being done right and in the
    correct order.

## Resources

No special resources should be needed to execute this proposal. Everything should be achievable with
volunteer labour.

## Timeline and Lifecycle

The initial requirements, Windows installer and Haskell download page should be completed
2021-08-01. The proposal is conservative and does not anticipate breaking anything.

## Deliverables

  * The `ghcup` and `stack` projects committed to delivering installers that meet our guidelines by
    (say) 2021-08-01.

  * A [published set of guidelines](003-installer/installer-guidelines.md) for Haskell installers.

  * (With haskell.org) a simplified haskell.org download page with links to the primary installers
    that deliver Haskell environments that conform to our guidelines.

  * A prototype Windows Haskell installer that supports cabal-install without the need for
    third-party installation frameworks.

## Outcomes

  * Substantial movement within the community towards a unified expectation of a minimal Haskell
    development environment.

  * A simplified primary Haskell download page that highlights installers meeting our guidelines.

  * Full coverage of minimal Haskell development environments for the three main operating systems.

## Risks

The most important risk is of increasing community disharmony around installation environments.

That said the early indicators are highly encouraging, with the `ghcup` and `stack` teams converging
a common set of guidelines and committed to delivering such an environment.

The best way of capitalising on this early progress is probably to lock in the wins and accept
conservative installers with room for improvement (with say imperfect stack/cabal-install/ghc
integrations) and trust to subsequent iteration.
