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

.. _`CLC Issue #015`: https://github.com/haskell/core-libraries-committee/issues/105

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

First and foremost, it creates a ton of busywork to upgrade to a new version of GHC as library version requirements must be relaxed.

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
Gabriella Gonzalez laid out the case in `Release early and often <https://www.haskellforall.com/2019/05/release-early-and-often.html>`__ on why coupling changes, especially breaking changes, together is bad, and I will cite that rather than restate the argument.
For those reasons we shouldn't do that here with the standard library and GHC.

Solution criteria
^^^^^^^^^^^^^^^^^

Changes in the standard library in the compiler should always be staggered.
It should be possible to upgrade the compiler with only a minor version change or less in the standard library, and possible to upgrade a major version change in the standard library without breaking a compiler.

Problem 5: Popular and uncontroversial machinery like ``Text`` not available from the standard library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There has been much grumbling over the years that popular items like ``Text`` which are normally expected to be in standard libraries are not.

It is one thing for a standard library to be minimal, and say not offer any string type or operations on that.
What is worse is that ``base`` does offer ``String``, and furthermore operations on ``String``.
The problem is thus not so much that it is inconvenient to grab the ``Text``-based functionality from elsewhere, as it is that ``base`` is has a foot-gun in offering alternatives that should be *avoided*.

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

  However, this doesn't dress the standard library --- language implementation coupling problem as both libraries still live in the compiler repo and are still released in tandem with the compiler.

- `cap-std <https://github.com/bytecodealliance/cap-std>`__ is a Rust library exploring what ergonomic IO interfaces forWASI system in a high level language should look like.
  On one hand, it is great, and we should borrow from it heavily.
  On the other hand, we should surpass in not needing to be something on top of the "regular" standard library which ordinarily exposes more Unixy things than is appropriate.

There have been prior attempts to split ``base`` before, but they attempted to get everything done at once at thus failed.
This approach here, by contrast, first and foremost seeks to the difficulties and find a sustainable, suitably low risk approach.
It is much more concerned with how we safely approach these issues than what the exact outcome looks like.

Technical Content
-----------------

Here is a plan to solve these issues.

Step 1a: Task the CLC with defining new standard libraries
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Based on the conversation in `CLC Issue #015`__, ``base`` is exposing too much stuff, yet trying to limit what is exposed would be a big breaking change.

The solution is to reach for another layer of indirection.
The CLC should be tasked with devising new standard library interfaces, which would initially be implemented by reexporting modules from ``base``.

The new library interfaces should be carefully designed in and of themselves to tackle many, but not all, of the issues above:

- They should be designed *not* to break every release.
  Even though the underlying ``base`` from which modules are exported would continue to  have its regular problematic major version bumps, the portion reexport should have very infrequent breaking changes.

  This fixes **Problem 1**.

- These libraries should be emphasized in all documentation and users should be encouraged to used them not ``base`` in new end-application code.
  ``base``, in contrast would be kept around in mere legacy mode.
  As code migrates over to use the new standard libraries, ``base`` should become less important.
  GHC devs can therefore feel increasingly confident modifying parts of ``base`` which are *not* reexported in these new libraries.

  This partially fixes **Problem 2**.

- The new standard library should not be a single library but multiple.
  IO-free interfaces that are portable everywhere should be one library.
  Interfaces involving IO should be split into libraries where they run.
  
  For example, Unix and Windows are mostly a superset of WASI, so WASI-compatible file-descriptor-oriented code should work everywhere.

  Exactly how many separate libraries is justified is left to the CLC.

  This fixes **Problem 3**.

- Because these are new libraries "on top" of ``base``, they can also reexport items from libraries, like ``text``.
  The CLC should consider such reexports.

  This fixes **Problem 5**.

New Goal: Rationalize dependencies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Step 1a addresses most problems, but leaves behind **Problem 2** somewhat, and **Problem 4** completely.
But moreover than that, Step 1a doesn't exactly make for a maintainable solution.
As the famous David Wheeler quote states:
"All problems in computer science can be solved by another level of indirection, *except for the problem of too many layers of indirection*."
Reexporting a modules from a less stable library (``base``) in more stable libraries is very error-prone.

The generalization of these concerns is *rationalizing* dependencies, or rationalizing the division of labor between libraries.

New Goal: Split Base
~~~~~~~~~~~~~~~~~~~~

We should still split ``base``.
This might sound surprising --- wasn't the point of making new libraries that we didn't need to worry about ``base`` so much?
But it follows from the expanded "rationalize dependencies" goal.

#. It will take a while for code to be migrated off ``base``, and until that process is complete ``base`` cannot serve as a "holding pen" for GHC's private implementation details.
   Thus, until that process is complete, we would not have a solution to **Problem 2**.
   Rather than waiting for ``base`` to stop being used, we can split it, and then GHC devs have (at least one) *proper* place for their unstable stuff, making a far more robust **Problem 2** solution while the migration away from ``base`` is still underway.

#. Solving **Problem 4** requires that some of the code in ``base`` to day *not* be coupled with GHC and some of the code in ``base`` conversely *must* be coupled with GHC.
   Thus solving **Problem 4** requires splitting ``base`` eventually anyways.

#. ``base`` is treated specially in a few ways.
   For example:

   - it is the library that GHCi loads by default.

   - GHC's compilation is directly aware of it in the form of various "wired-in" identifiers.

   - Some modules of it are automatically trusted with Safe Haskell.

   With the new multi-library world, different libraries will inherit these special features, and we cannot be sure what the ramifications are until we try.

   It is best to "practice" this by splitting ``base`` as soon as possible.
   That will reduce the risk of everything else by exploring for "unknown unknowns" and "unknown unknowns" alike.

#. Ultimately, in the name of rationalizing dependencies and the library division of labor, ``base`` will never make sense in anything like its current form.
   We should therefore demote it to being a mere reexporter of other libraries that do make sense.

Step 1b: MVP Split ``base`` by making it all reexports
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The first steps of `GHC issue #20647 <https://gitlab.haskell.org/ghc/ghc/-/issues/20647>`__ track what needs to be done here.
The key first step is finishing `GHC PR !7898`__.
This is crude: a ``ghc-base`` that ``base`` merely reexports in full is just as ugly as the original ``base``, but this is the quickest route to de-risking the entire project as describe in item 2 of the previous section.

.. _GHC PR !7898: https://gitlab.haskell.org/ghc/ghc/-/merge_requests/7898

Step 2a: Rationalize dependencies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

At this point we have the following:

- ``ghc-base``
- ``base`` which reexports ``ghc-base``
- A number of new libraries which reexport parts of ``base`` and possibly other libraries like ``text``.

The goal is to shuffle code around so that we have something which makes more sense.
That would look something like this:

- 1 or more libraries in the GHC repo that are deeply tied to GHC's implementation details.
  These libraries might depend on libraries in the next group.
- 1 or more libraries outside the GHC that are repo agnostic to GHC's implemenation details.
  These libraries might depend on libraries in the previous group.
- ``base``, lives in the GHC repo, and merely reexports functionality from the first two groups.
- ``text``, if used by the new stand library, should *not* depend on ``base``.
- The new standard libres, living outside the GHC repo, merely rexporting functionality from the first two groups and possibly ``text``.

It will take a while to untangle everything to get to this new maintainable end state.
The good news is that we can get there very incrementally.
The initial crude split will validate that shuffling definitions between libraries and modules works at all.
After that, continuing to shuffle items reduces risk.

The `GHC Wiki page on "Split Base" <https://gitlab.haskell.org/ghc/ghc/-/wikis/split-base>`__, especially Joachim Breitner's `prior attempt <https://github.com/nomeata/packages-base/blob/base-split/README.md>`__ offers good ideas backed by experience on where the natural cleavage points within ``base`` lie.

At the conclusion of this, **Problem 2** and **Problem 4** will be solved in their entirety, which means all problems are solved in their entirety.

Step 2b: Practice release management (Optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We won't know for sure if **Problem 4** is solved until a GHC release happens.
But waiting for that could take a while, and is thus a risky behavior because we to know whether our efforts are on the right track or doomed to fail as soon as possible.

Therefore, as soon as we have *some* splitting and reexporting in progress, it is good to test out our work against a *past* GHC release.
In particular, we can perform the same splits on that that release, and see if the GHC-agnostic portions are swappable to allow for staggered breaking changes as intended.

This step is optional.
If the work appears to be going well or is quicker/cheaper than expected, maybe it is not worth the effort.
On the other hand, if we could do a minor release of the old GHC using the split, so the backported work isn't purely for de-risking but actually delivers some benefits to users, that provides more reason to do this.

Step 3: Bonus: Also split ``template-haskell``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``template-haskell``

Timeline
--------

The project is designed to proceed in parallel to minimize risk, in addition to being incremental.
Steps 1a and 1b are independent, and steps 2a and 2b are likewise independent.

In past discussion, consensus around a plan from step 1a was emphasized as a blocker --- if we didn't know what sort of standard libraries we wanted to end up with, we shouldn't proceed.
In the author's opinion this is misguided.
The actual stumbling point is not disagreements about where we want to end up, but maintaining progress on something which is not incredibly hard, but has many steps and ushers in most of the benefit over the long term.
(For example, many users of GHC are behind the latest version, these reforms only benefit them going forward after they have caught up to the last unaffected release.)

As such, the most crucial step is considered to be step 1b.
After that, we know the basic concept for sure works.
And indeed it is possible to start steps 2a and 2b before there is a complain step 1a plan.

Budget
------

Finishing `GHC PR !7898`__ is conservatively estimated to take 1 person-month of work from an experienced GHC's dev.
The HF should finance this work if there is no volunteers to ensure it is done as fast as possible, as everything else is far too uncertain until this trial round of splitting and reexports has been completed end to end.

It is unknown whether the CLC will need HF help to do the large amount of planning work for step 1a.

Step 2a should be priced out per incremental item, with the hope that specific steps will entice volunteers which care about the functionality behind reshuffled in that step.
HF may need to pay a coordination roll but hopefully doesn't need to pay for the work being done directly.
This should serve as a way to recruit more standard library maintainers going forward, as the fine-grained boundaries between the underlying libraries naturally lend themselves to a division of labor.

Stakeholders
------------

The Core Libraries Committee. Step 1a constitutes a large chunk of new responsibility for the CLC.

GHC developers: `GHC PR !7898`__ from step 1a has uncovered some bugs that will need fixing.
Step 2a will eventually result in churn among which submodules GHC contains, which will be frustrating until that stabilizes.
Step 2b, if it were to be released not just done on a fork as a trial, will result in more release management work and possible fallout of reshuffling the implementation of ``base`` behind the scenes.

Success
-------

The project will be considered a success when all the enumerated problems are solved per their "solution criteria" (no moving the goalposts later without anyone noticing), and the standard library implementation is easier to maintain than before.
