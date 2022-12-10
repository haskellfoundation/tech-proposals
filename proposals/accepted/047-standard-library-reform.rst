Standard library reform
=======================

.. sectnum::
.. contents::

Abstract
--------

Issues with the standard library are holding back the Haskell ecosystem.
The problems and solutions are multifaceted, and so the Haskell Foundation in its "umbrella organization" capacity is uniquely suited to coordinate the fixing of them.

The problems are briefly as follows:

#. Major version bumps every compiler release is an excusable nuisance.

#. No clear boundary between GHC private/unstable library support code and public/stable standard library interfaces.
   `CLC Issue #015`__.

#. No clear portability guarantees with new targets like the web browser and Web Assembly System Interface (WASI).

#. Breaking changes (when we do want) have to coupled with GHC versions and not staggered is also painful.

#. Popular and uncontroversial machinery like `Text` is not available from the standard library.

By reshuffling our interfaces and implementations a like, we should be able to solve all these problems.

.. __`CLC Issue #015`: https://github.com/haskell/core-libraries-committee/issues/105>

Background
----------

The author deems these problems as major and highly visible; if this is true Haskellers of all skill levels should at least have a cursory familiarity with them.

The details of the solution may require more advance knowledge, but the *use* of the solution should not.
Indeed, the new standard library interfaces we come up with should be *easier* to use than today's.

Problem Statement
-----------------

Each of items from the abstract is described in detail.

Problem 1: Major version bumps every compiler release
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Currently, every major release of of GHC is accompanied with a major version of ``base``, and also other libraries like ``template-haskell``.
This causes numerous issues:

First and foremost, it crates a ton of busywork to upgrade to a new version of GHC as library version requirements must be relaxed.

Secondly it undermines our other processes by creating perverse incentives.

Library authors find it convenient to make too-loose requirements on ``base`` on the assumption that whatever base breakage happens next "probably" won't effect them.
But fast-and-loose version bounds conversely the version solver cannot be trusted to choose good plans.
We want version solving to be sound and complete, and the only way for that to be the case is if breaking changes are rare.

It also makes it harder to think about compatibility and ease of upgrading with GHC in general.
This and other long-shrugged-off paper cuts over the upgrade process result in a big picture where where some of us are numb to breakage, and others are irate about it.
We should do the little things well so the remaining thornier issues around GHC upgrading (syntax changes, type system changes, etc.) can be approached from a cleaner starting point.

Solution criteria
^^^^^^^^^^^^^^^^^

Users should be able to upgrade to the next GHC without adjusting any library version requirements.

Problem 2: No clear boundary between private/unstable and public/stable interfaces
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The long discussion thread in `CLC Issue #015`__ really demonstrates this purposefully.

On a simpler level, the lack of a firm boundary confuses users, who don't know which parts of ``base`` they ought to use, and GHC developers, who don't know what parts of the code they are free to change.

On a more complex level, I think everyone in the thread was surprised on how hard it was to even discuss these issues.
Not only is there not a firm boundary, but there wasn't even a collectively-shared mental model on how to discuss the issue or its solutions!

Solution criteria
^^^^^^^^^^^^^^^^^

We should use standard off-the-shelf definitions and techniques to enforce this boundary.
The standard library should not expose private, implementation-detail modules full-stop.
The entirely of the standard library's public interface should be considered just that, its public interface.
Private modules that we do wish to expose to code that *knowingly* is using unstable interfaces should be exposed from a separate library/
The standard library should use regular PVP versioning. 

Problem 3: No clear portability guarantees with new targets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The new backends that come with GHC 9.6 are chiefly thought of as new CPUs/Arches.
WASM and JS are, with enough squinting, just ways of expressing computation that like "x86" vs "Aarch64" vs "RISC-V", etc., should by and large not leak to the user.
(Exceptions would be when dealing with memory model or special instruction intricacies.)

What is more interesting from a library design perspective is where the code will be run.
This would be analogous to the "OS" part of the platform description, like "Linux" vs "Windows" vs "macOS" etc.

JavaScript can be run in two places:

- The web browser
- Node.js and similar projects

WASM can also be run in two places:

- The web browser
- Wasmtime and similar projects

Node.js exposes as much of the underlying functionality of the OS as it can, and so a standard library with it in mind doesn't need to be that different from a standard library with the underlying OS in mind.
The other two, however are a radical departure:

- The web browser is nothing at all like Unix.

- WASI, the Web Assembly System Interface, is like a "functional unix" removing ambient authority and forcing side effects to be mediated via file descriptors.
  The upcoming `WASI Component Model <https://github.com/WebAssembly/component-model>`__ also plans on creating replacements for some "stringly typed" Unix functionality with "richly typed" interfaces.
  Both these things are an *excellent* for Haskell.

The existing implementations in GHC, to my knowledge, duck-tape over ``base`` and friends as much as possible just to get something working.
This made perfect sense for GHCJS, and perfect sense for just getting things going.
But it is a poor choice for a mature, first-class backend.
Haskell has a mantra that "If it compiles, it probably works", and stubbing out functionality with ``error`` and friends is a huge regression from that.

Solution criteria
^^^^^^^^^^^^^^^^^

Projects should be able to depend on libraries that just expose functionality that is known to work on the platform(s) they run on.
The plural, "platforms" is key.
Projects that wish to some set of Unix, Windows, Web, and WASI must be able to depend on libraries that only offer the *intersection* of what works on each of those, i.e. what works on all of them.
We will thus need more than one standard library.

Problem 4: Breaking changes have to coupled, not staggered, with GHC versions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Wishful thinking would have it that we can just *stop* doing breaking changes, forever.
But requirements change, and no one never makes mistakes.
Issues will arise in the standard library and we will wish to fix them, because whatever the cost is to existing programs (which we can still attempt to mitigate) is outweighed by the benefit to future programs.

However, if the standard library version is tied to GHC version, we have no choice but to do the breaking change coupled with a compiler version.
Gabriella Gonzalez laid out the case in `Release early and often <https://www.haskellforall.com/2019/05/release-early-and-often.html>` on why coupling changes, especially breaking changes, together is bad, and I will cite that rather than restate the argument.
For those reasons we shouldn't do that here with the standard library and GHC.

Solution criteria
^^^^^^^^^^^^^^^^^

Changes in the standard library in the compiler should always be staggered.
It should be possible to upgrade the compiler with only a minor version change or less in the standard library, and possible to upgrade a major version change in the standard library without breaking a compiler.

Problem 5: Popular and uncontroversial machinery like ``Text`` not available from the standard library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There has been much grumbling over the years that popular items like ``Text`` which are normally expected to be in standard libraries are not.

It is one thing for a standard library to be minimal, and say not offer any string type or operations on that.
What is worse is that ``base`` does offer ``String``, and futhermore operations on ``String``.
The problem is thus not so much that it is inconvenient to grab the ``Text``-based functionality from elsewhere, as it is that ``base`` is has a footgun in offering alternatives that should be *avoided*.

Solution criteria
^^^^^^^^^^^^^^^^^

Firstly, do not offer bad alternatives in the standard library that users should not use.
Secondarily, do offer good alternatives, like ``Text`` and associated functionality, if they are suitable for inclusion.

Prior Art and Related Efforts
-----------------------------

There has been much discussion of these topics before, but to my knowledge this is the first time they have been consolidated together.

A few misc things:

- Rust's ``core`` vs ``std`` split of the standard library aims to help the portability problem.
  Only maximally portable concepts can go in ``core``, the rest goes in ``std``.

  However, this doesn't dress the standard library --- language implementation coupling problem as both libraries still live in the compielr repo and are still released in tandem with the compiler.

- `cap-std <https://github.com/bytecodealliance/cap-std>` is a Rust library exploring what ergnomic IO interfaces forWASI system in a high level language should look like.
  On one hand, it is great, and we should borrow from it heavily.
  On the other hand, we should surpass in not needing to be something on top of the "regular" standard library which ordinarily exposes more Unixy things than is appropriate.

Technical Content
-----------------

Here is a plan to solve these issues.

Step 1: Task the CLC with defining new interfaces
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


_This section should describe the work that is being proposed to the community for comment, including both technical aspects (choices of system architecture, integration with existing tools and workflows) and community governance (how the developed project will be administered, maintained, and otherwise cared for in the future).
It should also describe the benefits, drawbacks, and risks that are associated with these decisions.
It can be a good idea to describe alternative approaches here as well, and why the proposer prefers the current approach._

Timeline
--------

_Are there any deadlines that the HF needs to be aware of?_

Budget
------

_How much money is needed to accomplish the goal?
How will it be used?_

Stakeholders
------------

_Who stands to gain or lose from the implementation of this proposal?
Proposals should identify stakeholders so that they can be contacted for input, and a final decision should not occur without having made a good-faith effort to solicit representative feedback from important stakeholder groups._

Success
-------

_Under what conditions will the project be considered a success?_

