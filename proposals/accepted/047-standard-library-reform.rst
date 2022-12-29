Standard library reform
=======================

.. sectnum::
.. contents::

Abstract
--------

Issues with the standard library are holding back the Haskell ecosystem.
The problems and solutions are multifaceted, and so the Haskell Foundation in its "umbrella organization" capacity is uniquely suited to coordinate the fixing of them.

The problems are briefly as follows:

#. Major version bumps every compiler release is a nuisance.

#. No clear boundary between GHC private/unstable library support code and public/stable standard library interfaces.
   `CLC Issue #105`_.

#. No clear portability guarantees with new supported platforms like the web browser and Web Assembly System Interface (WASI).

#. Breaking changes (when we do want them) have to be coupled with GHC versions rather than staggered, which is painful.

#. Popular and uncontroversial machinery like ``Text`` is not available from the standard library.

By reshuffling our interfaces and implementations alike, we should be able to solve all these problems.

.. _`CLC Issue #105`: https://github.com/haskell/core-libraries-committee/issues/105

Background
----------

The author deems these problems major and highly visible;
if this is true then Haskellers of all skill levels should at least have a cursory familiarity with them.

The details of the solution may require more advanced knowledge, but the *use* of the solution should not.
Indeed, the new standard library interfaces we come up with should be *easier* to use than today's.

Problem Statement
-----------------

Each of items from the abstract is described in detail.

**Problem 1**: Major version bumps every compiler release
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Currently, every major release of of GHC is accompanied with a major version of ``base``, and also other libraries like ``template-haskell``.
This causes numerous issues:

First and foremost, these major version bumps create a ton of busywork to upgrade to a new version of GHC as library version requirements must be relaxed.

Secondly they undermine our other processes by creating perverse incentives.

Library authors find it convenient to make too-loose requirements on ``base`` on the assumption that whatever base breakage happens next "probably" won't effect them.
But fast-and-loose version bounds undermine the version solver, which can no longer be trusted to choose good plans in that scenario.
We want version solving to be sound and complete, and the only way for that to be the case is if breaking changes are infrequent enough that people do not feel the urge to do this.

These major version bumps also make it harder to think about compatibility and ease of upgrading with GHC in general.
This and other long-shrugged-off paper cuts during the upgrade process result in a big picture where where some of us are numb to breakage, and others are irate about it.
We should do the little things well so the remaining thornier issues around GHC upgrading (syntax changes, type system changes, etc.) can be approached from a "decluttered" starting point.

Solution criteria
^^^^^^^^^^^^^^^^^

Users should usually be able to upgrade to the next GHC version without adjusting any library version requirements.

**Problem 2**: No clear boundary between private/unstable and public/stable interfaces in the standard library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The long discussion thread in `CLC Issue #105`_ demonstrates this exceedingly well.

On a simpler level, the lack of a firm boundary confuses users, who don't know which parts of ``base`` they ought to use, and GHC developers, who don't know what parts they are free to change.

On a more meta level, I think everyone in the thread was surprised on how hard it was to even discuss these issues.
Not only is there no firm boundary, but there wasn't even a collectively-shared mental model on what exactly the issue is, and how to discuss it or its solutions!
This is a "tower of Babel" moment where the inability to communicate makes it hard to work together.

Solution criteria
^^^^^^^^^^^^^^^^^

We should use standard off-the-shelf definitions and techniques to enforce this boundary.
The standard library should not expose private, implementation-detail modules.
The entirety of the standard library's public interface should be considered just that, its public interface.
Private modules that we do wish to expose to code that *knowingly* is using unstable interfaces should be exposed from a separate library.
The standard library should use regular PVP versioning.

In solving the immediate problem this way, we also solve the meta problem.
Using off-the-shelf definitions gives us a shared language reinforced by practice in the rest of the Haskell ecosystem. [#ubiquitous-language]_

**Problem 3**: No clear portability guarantees with new targets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The new compilation backends that come with GHC 9.6 correspond, strictly speaking, to new supported CPUs/Arches, like "x86" vs "Aarch64" vs "RISC-V", etc.
WASM and JS are, with enough squinting, just other ways of expressing computation: ways which should by and large not leak to the user. [#cpu-leaks]_

What is more interesting from a library design perspective is over what *software* will the code be run.
This would be analogous to the "Operating Systems" part of the platform description, like "Linux" vs "Windows" vs "macOS" etc.

JavaScript generated by GHC can be run in two places:

- The web browser
- Node.js and similar projects

WASM can also be run in two places:

- The web browser
- Wasmtime and similar projects

Node.js exposes as much of the underlying functionality of the OS as it can, and so a standard library with it in mind doesn't need to be that different from a standard library with the underlying OS in mind.
The other two, however are a radical departure:

- The web browser is nothing at all like Unix.

- WASI, the Web Assembly System Interface, is like a "functional unix" removing ambient authority and forcing side effects to be mediated via file descriptors.
  The upcoming `WASI Component Model <https://github.com/WebAssembly/component-model>`_ also plans on creating replacements for some "stringly typed" Unix functionality with "richly typed" interfaces.
  Both these things are an *excellent* fit for Haskell.

The existing implementations in GHC duck-tape over ``base`` and friends the best they can to get something working.
That is to say, we have some CPP::

  $ git grep js_HOST_ARCH libraries/ | wc-l
  52

  $ git grep wasm32_HOST_ARCH libraries/ | wc -l
  2

This made perfect sense for GHCJS, and perfect sense for just getting things going more broadly.
But they are poor long-term choices for a mature, first-class backend.

A first issue is that since this is all based on the host *arch* and not *OS*, we have no distinguishing between the browser and non-browser runtimes.
One just has to hope that the intended deployment environment as the functionality they wish to use.

A second issue is that it is very easy to, when developing (say with GHCi or HLS) on one platform, accidentally depend on things that not available on the other platforms ones wishes to support.
Yes, CI which builds for all of the platforms can and should catch this, but it is always sub-optimal to only catch basic issues then.

The much lower CPP count for Web Assembly reflects that fact that the reference `WASI libc`_ itself tries to emulate POSIX the best it can.
But this just means the same infelicities are there, just less directly observable.
For example, it incorporates the techniques of `libpreopen`_ to simulate ambient authority such as opening arbitrary files by absolute path.
But best-effort techniques like this only if one is lucky; they are a great way for adapting *existing* applications but a *poor* way for writing new greenfield ones.

.. _`WASI libc`: https://github.com/WebAssembly/wasi-libc
.. _`libpreopen`: https://github.com/musec/libpreopen

Solution criteria
^^^^^^^^^^^^^^^^^

Projects should be able to depend on libraries that just expose functionality that is known to work on the platform(s) they run on.
The plural, "platforms" is key.
Projects that wish to support some subset of Unix, Windows, Web, and WASI must be able to depend on libraries that only offer the *intersection* of what works on each of those, i.e. what works on all of them.
We will thus need more than one standard library.

Platform-specific functionality should be exposed in ways that make sense in Haskell, not C.
Traditional libc idioms and "lowest common denominator" practice should be skipped when it does not make sense in a Haskell context.
It should be possible to use WASM and WASI without any "libc".

**Problem 4**: Breaking changes have to coupled, not staggered, with GHC versions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Wishful thinking would have it that we can just *stop* doing breaking changes, forever.
But requirements change, and mistakes are made.
Issues will arise in the standard library and we will wish to fix them, because whatever the cost is to existing programs (which we can still attempt to mitigate) is outweighed by the benefit to future programs.

However, if the standard library version is tied to GHC version, we have no choice but to do the breaking change coupled with a compiler version.
Gabriella Gonzalez laid out the case in `Release early and often <https://www.haskellforall.com/2019/05/release-early-and-often.html>`_ on why coupling changes, especially breaking changes, together is bad, and I will cite that rather than restate the argument.
For those reasons we shouldn't do that here with the standard library and GHC.

Solution criteria
^^^^^^^^^^^^^^^^^

Changes in the standard library in the compiler should always be staggered.
It should be possible to upgrade the compiler with only a minor version change or less in the standard library.
It should likewise be possible to upgrade a major version change in the standard library without breaking a compiler.

**Problem 5**: Popular and uncontroversial machinery like ``Text`` not available from the standard library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There has been much grumbling over the years that popular items like ``Text`` are not in the standard library.
Items like these are expected to be in languages' standard libraries and elsewhere indeed are found there.

Now, it is one thing for a standard library to be minimal, and say not offer any string type or operations on that.
That would not be so bad.
What is worse is that ``base`` does offer ``String``, and furthermore operations on ``String``.
The problem is thus not so much that it is inconvenient to grab the ``Text``-based functionality from elsewhere, as it is that ``base`` has a foot-gun in offering alternatives that should be *avoided*.
Standard libraries which *mislead* the user as to what they ought to do are worse than standard libraries which stay mum altogether.

Solution criteria
^^^^^^^^^^^^^^^^^

Firstly, do not offer bad alternatives in the standard library that users should not use.
Secondarily, do offer good alternatives, like ``Text`` and associated functionality, if they are suitable for inclusion.

.. [#ubiquitous-language]
  Compare the "Ubiquitous Language" concept from Eric Evan's *Domain-driven design* also cited in the GHC modularity paper.

.. [#cpu-leaks]
  The choice of CPU/Arch does leak through when wants to do certain special operations, like atomics that depend on the intricacies of memory models, or data-paralleld "SIMD" instrucitons.
  But these concerns are fairly niche and we can mostly not think about them for the purposes of standard library design.

Prior Art and Related Efforts
-----------------------------

There has been much discussion of these topics before, but to my knowledge this is the first time they have been consolidated together.

A few miscellaneous things:

Rust's ``core`` vs ``std``
~~~~~~~~~~~~~~~~~~~~~~~~~~

Rust also has multiple standard libraries, of which the most notable are ``core`` vs ``std``.
This split solves the portability problem:
Only maximally portable concepts, ones that work everywhere Rust does including embedded/freestanding contexts, can go in ``core``.
The rest must go in ``std``.

However, this doesn't go far enough to address the standard library --- language implementation coupling problem.
Both libraries still live in the compiler repo and are still released in tandem with the compiler.
``core`` also contains numerous definitions that, while perfectly portable, have nothing to do with interfacing the compiler internals.
(Think e.g. the equivalents of things like ``Functor`` and ``Monoid`` for us, perfectly portable across compilation targets, but also implementation-agnostic.)

Rust's ``cap-std``
~~~~~~~~~~~~~~~~~~

`cap-std <https://github.com/bytecodealliance/cap-std>`_ is a Rust library exploring what ergonomic IO interfaces for WASI system calls in a high level language should look like.
On one hand, it is great, and we should borrow from it heavily.
On the other hand, we should surpass it in not needing to be something on top of the "regular" standard library which ordinarily exposes more Unixy things than is appropriate.

Rust's ``std`` and WASI
~~~~~~~~~~~~~~~~~~~~~~~

While the best experience comes from using ``cap-std`` as described above, Rust's ``std`` still makes sure to avoid indirecting through ``wasi-libc`` wherever possible.
`This PR <https://github.com/rust-lang/rust/pull/63676>`_ made that change, using the ``wasi`` library (Rust bindings to WASI system calls) directly.
This is what we should emulate in order to provide a top-tier programming environment for greenfield WebAssembly applications in Haskell.

Prior attempts at splitting ``base``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There have been attempts to split ``base`` before, but they attempted to get everything done at once, setting a dangerously high bar for success.
This approach here, by contrast, first and foremost seeks to avoid those difficulties and find a sustainable, suitably low-risk approach.
It is much more concerned with how we safely approach these issues than what the exact outcome looks like.

Technical Content
-----------------

Here is a plan to solve these issues.

**Step 1A**: Task the CLC with defining new standard libraries
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Based on the conversation in `CLC Issue #105`_, ``base`` is exposing too much stuff, yet trying to limit what is exposed would be a big breaking change.

The solution is to reach for another layer of indirection.
The CLC should be tasked with devising new standard library interfaces, which would initially be implemented by reexporting modules from ``base``.

The new library interfaces should be carefully designed in and of themselves to tackle many, but not all, of the issues above:

- They should be designed *not* to break every release.
  Even though the underlying ``base`` from which modules are reexported would continue to have its regular problematic major version bumps, the portion reexported should have very infrequent breaking changes.

  This fixes **Problem 1**.

- These libraries should be emphasized in all documentation, and users should be encouraged to use them and not ``base`` in new end-application code.
  ``base``, in contrast, would be kept exposed as a mere legacy interface.
  As code migrates over to use the new standard libraries, ``base`` should become less important.
  GHC devs can therefore feel increasingly confident modifying parts of ``base`` which are *not* reexported in these new libraries.

  This partially fixes **Problem 2**.

- The new standard library should not be a single library but multiple.
  IO-free interfaces that are portable everywhere should be one library.
  Interfaces involving IO should be split into libraries where they run.

  For example, Unix and Windows are mostly a superset of WASI, so WASI-compatible file-descriptor-oriented code should work everywhere.

  Exactly how many separate libraries is justified is left to the CLC to decide.

  This fixes **Problem 3**.

- Because these are new libraries "on top" of ``base``, they can also reexport items from libraries, like ``text``.
  The CLC should consider such reexports.

  This fixes **Problem 5**.

New Goal: Rationalize dependencies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Step 1A** addresses most problems, but leaves behind **Problem 2** somewhat, and **Problem 4** completely.
But moreover, **Step 1A** doesn't exactly make for a maintainable solution.
As the famous David Wheeler quote states:
"All problems in computer science can be solved by another level of indirection, *except for the problem of too many layers of indirection*."
Reexporting modules from a less stable library (``base``) in more stable libraries is very error-prone.

The generalization of these concerns is *rationalizing* dependencies, or rationalizing the division of labor between libraries.
Once the purposes of libraries and the division of labor between them make more sense, it will be easier to maintain these libraries.
It should be in fact easier than it was before to maintain them.

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

   - It is the library that GHCi loads by default.

   - GHC's compilation is directly aware of it in the form of various "wired-in" identifiers.

   - Some modules of it are automatically trusted with Safe Haskell.

   In the new multi-library world, different libraries will inherit these special features, and we cannot be sure what the ramification will be until we try.

   It is best to "practice" this by splitting ``base`` as soon as possible.
   That will reduce the risk of everything else by both exploring "known unknowns" and scouting ahead for "unknown unknowns".

#. Ultimately, in the name of rationalizing dependencies and the library division of labor, ``base`` will never make sense in anything like its current form.
   We should therefore demote it to being a mere reexporter of other libraries that do make sense.

**Step 1B**: MVP Split ``base`` by making it all reexports
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The first steps of `GHC issue #20647`_ track what needs to be done here.
The key first step is finishing `GHC MR !7898`_.
This is crude: a ``ghc-base`` that ``base`` merely reexports in full is just as ugly as the original ``base``, but this is the quickest route to de-risking the entire project as described in item 2 of the previous section.

.. _`GHC issue #20647`: https://gitlab.haskell.org/ghc/ghc/-/issues/20647
.. _`GHC MR !7898`: https://gitlab.haskell.org/ghc/ghc/-/merge_requests/7898

**Step 2A**: Rationalize dependencies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

At this point we have the following:

- ``ghc-base``
- ``base`` which reexports all of ``ghc-base``
- A number of new libraries which reexport parts of ``base`` and possibly other libraries like ``text``.

The goal is to shuffle code around so that we have something which makes more sense.
That would look something like this:

- 1 or more libraries in the GHC repo that are deeply tied to GHC's implementation details.
  These libraries might depend on libraries in the next group.
- 1 or more libraries outside the GHC repo that are agnostic to GHC's implementation details.
  These libraries might depend on libraries in the previous group.
- ``base``, lives in the GHC repo, and merely reexports functionality from the first two groups.
- ``text``, lives outside the GHC repo, and should *not* depend on ``base``, but instead libraries from the first two groups.
- The new standard libraries, living outside the GHC repo, merely reexporting functionality from the first two groups and possibly ``text``.

It will take a while to untangle everything to get to this new maintainable end state.
The good news is that we can get there very incrementally.
The initial crude split will validate that shuffling definitions between libraries and modules works at all.
After that, continuing to shuffle items incrementally reduces risk.

The `GHC Wiki page on "Split Base" <https://gitlab.haskell.org/ghc/ghc/-/wikis/split-base>`_, especially Joachim Breitner's `prior attempt <https://github.com/nomeata/packages-base/blob/base-split/README.md>`_ offers good ideas backed by experience on where the natural cleavage points within ``base`` lie.

At the conclusion of this, **Problem 2** and **Problem 4** will be solved in their entirety, which means all problems are solved in their entirety.

**Step 2B**: Practice release management (Optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We won't know for sure if **Problem 4** is solved until a GHC release happens.
But waiting for that could take a while, and is thus a risky behavior because we to know whether our efforts are on the right track or doomed to fail as soon as possible.

Therefore, as soon as we have *some* splitting and reexporting in progress, it is good to test out our work against a *past* GHC release.
In particular, we can perform the same splits on that release, and see if the GHC-agnostic portions are swappable to allow for staggered breaking changes as intended.

This step is optional.
If the work appears to be going well or is quicker/cheaper than expected, maybe it is not worth the effort.
On the other hand, if we could do a minor release of the old GHC using the split, so the backported work isn't purely for de-risking but actually delivers some benefits to users, that provides more reason to do this.

Bonus **Step 3A**: Also split ``template-haskell``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``template-haskell`` also suffers from the same versioning problem as ``base``.
For issues unrelated to avoiding version churn busywork, in `GHC issue #21738`_ it was already proposed to split up the library.
`GHC proposal #529`_ likewise proposing adding language features such that the breakage-prone portion of ``template-haskell`` is way less likely to be needed.
If we implement that language feature, then it makes sense to additionally split of ``template-haskell`` for stability's sake, solving the equivalent of **Problem 1** for that library.

.. _`GHC issue #21738`: https://gitlab.haskell.org/ghc/ghc/-/issues/21738
.. _`GHC proposal #529`: https://github.com/ghc-proposals/ghc-proposals/pull/529

Bonus **Step 3B**: Rethinking Windows
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Right now, ``base`` relies on MinGW's and Windows's `libc` compat layer to approximate traditional Unix functionality.
The ``unix`` and ``Win32`` layers than expose additional platform-specific functionality.

Quite arguably, this is the wrong way of going about IO.

- It would be nice to make MinGW optional and support Windows more directly/natively.
  This is what Rust does.
  LLVM has made doing so (e.g. without relying on proprietary tools exclusively) much easier in recent years.
  As Ben Gamari and others can attest, the state of Windows support in GNU tools is not good.

- It would be nice to not limit ourselves to a lowest-common-denominator of ``libc``-esque functionality as our starting point.
  Windows and Linux have added all sorts of more modern functionality in recent years that often is (a) similar, and (b) represents better ways to do existing operations, e.g. avoiding around restrictions on character sets, file path length, etc.

From this perspective we should invert the dependencies:
``unix`` and ``Win32`` should be below, binding Unix and Windows APIs *as they are*,
and then *above* that is a compatibility layer creating portable interfaces with the latest best practice *without* the burden of libc tradition.

This sort of reshuffle is a continuation of the project of rationalizing dependencies and a natural extension of **Step 2A**.

Timeline
--------

The project is designed to proceed in parallel to minimize risk, in addition to being incremental.
Steps 1a and 1b are independent, and steps 2a and 2b are likewise independent.

In past discussion, consensus around a plan from **Step 1A** was emphasized as a blocker --- if we didn't know what sort of standard libraries we wanted to end up with, we shouldn't proceed.
In the author's opinion this is misguided.
The actual stumbling point is not disagreements about where we want to end up, but maintaining progress on something which is not incredibly hard, but has many steps and ushers in most of the benefit over the long term.
(For example, many users of GHC are behind the latest version, these reforms only benefit them going forward after they have caught up to the last unaffected release.)

As such, the most crucial step is considered to be **Step 1B**.
After that, we know the basic concept for sure works.
And indeed it is possible to start steps 2a and 2b before there is a complete **Step 1A** plan.

It may well additionally make sense to preliminarily accept *just* **Step 1B**, and then go back and refine this proposal's Timeline and Budget sections with the information we've learned from **Step 1B**.

Budget
------

**Step 1A** costs
~~~~~~~~~~~~~~~~~

It is unknown whether the CLC will need HF help to do the large amount of planning work for **Step 1A**.

The HF should reach out to the `Bytecode Alliance <https://bytecodealliance.org/>`, which is the HF equivalent for WASM and WASI, for financial and technical assistance ensuring the relevant new standard libraries can work well with WASI.

**Step 1B** costs
~~~~~~~~~~~~~~~~~

Finishing `GHC MR !7898`_ is conservatively estimated to take 1 person-month of work from an experienced GHC dev.
The HF should finance this work if there are no volunteers to ensure it is done as fast as possible, as everything else is far too uncertain until this trial round of splitting and reexports has been completed end to end.

**Step 2A** costs
~~~~~~~~~~~~~~~~~

**Step 2A** should be priced out per incremental item, with the hope that specific steps will entice volunteers which care about the functionality behind reshuffled in that step.
HF may need to play a coordination role but hopefully doesn't need to pay for the work being done directly.
This should serve as a way to recruit more standard library maintainers going forward, as the fine-grained boundaries between the underlying libraries naturally lend themselves to a division of labor.

**Step 2B** costs
~~~~~~~~~~~~~~~~~

This steps is optional.
But since it involves redoing the work already done on GHC master on a prior GHC, we can use our collective experience with backporting to estimate what the ratio of effort to that for the original work would be.
1/2 time is a rough estimate at a cautious upper bound.

Stakeholders
------------

The Core Libraries Committee
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Step 1A** constitutes a large chunk of new responsibility for the CLC.
This project depends on on them being interested and willing in taking on that work.

GHC developers
~~~~~~~~~~~~~~

`GHC MR !7898`_ from **Step 1A** has uncovered some bugs that will need fixing.,
**Step 2A** will eventually result in churn among which submodules GHC contains, which will be frustrating until that stabilizes.
**Step 2B**, if it were to be released not just done on a fork as a trial, will result in more release management work and possible fallout of reshuffling the implementation of ``base`` behind the scenes.

Due to **Problem 4**, the interest and cooperation of the developers of our new backends is especially solicited.

Success
-------

The project will be considered a success when all the enumerated problems are solved per their "solution criteria" (no moving the goalposts later without anyone noticing), and the standard library implementation is easier to maintain than before.

Appendix: What are standard libraries for?
------------------------------------------

*If parts of this proposal seems hard to understand or surprising, background information in the form of the author's critical view on the very concept of a standard library me prove illuminating.*

A contradictory dual mandate
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Standard libraries typically have a dual mandate which is hard to reconcile:

#. On one hand, they are supposed to be the *bottommost* library, abstracting over the unstable or non-portable details of the language's implementation.

#. On the other hand, they are supposed to be *feature-rich* and provide a bunch of convenient and widely agreed upon stuff that represents the language community's consensus on what functionality ought to always be available, and how certain common problems should be approached.
   To use the common phrase for this idea, they exist to make the language "batteries included".

The tension lies between *bottommost* from (1) and *feature-rich* from (2).
The only way to do both is to become truly massive and just span that gap.
And this is what most languages do.
But frequently results in a giant monolith which is hard to maintain and hard to change --- a source of endless frustration.
And indeed that is the experience of most language's over time: languages die young or live long enough to regret many of the decisions in their standard library.

Let's take a step bit.
The benefits of (2) are mainly for `"programming in the small" <https://en.wikipedia.org/wiki/Programming_in_the_large_and_programming_in_the_small>` and end applications.
For libraries, and especially the ecosystem of libraries as a whole, a primary objective is to be resilient in the face of change: in other words to have the lease disruption per breakage and controversy as possible.
To that end a few simple rules can help:

 - Libraries should do one thing, and do that one things well
 - Libraries should only depend on what they need.

These rules serve libraries well...until we reach the standard library.
The standard library of the above sort, trying to do (1) and (2), does *many* things, and not necessarily any of them well.
Downstream libraries furthermore will inevitably only use a small part of the standard library, and so both rules are provided.

Large libraries bad technically
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

From the perspective of this "little library programming in the large", standard-libraries are an anti-pattern.
We should follow a consistent practice, and have little modular libraries "all the way down", to the guts of primops, the runtime, or whatever other spooky dragons there be.
By following the two simple rules completely, the needs of such libraries are served quite while.
Mistakes can be remedied with the occasional breaking change, the breaking change impacts as few downstream libraries as possible, and it is easy to maintain the old and new versions of libraries (two major version series) in parallel, to allow for graceful migration periods.
From the perspective of *existing, large-scale* users of Haskell, who consume the existing library ecosystem voraciously, this would be a great improvement.

Large standard libraries good socially
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

But that doesn't mean we should leave "programming in the small" in the lurch!
This is still important, and quite arguably a weak-spot of Haskell already.
New users first experience of a language, unless it is on the job, is usually programming in the small, so it is an essential marketing opportunity to get right.
And this indirectly benefits programming in the large, too.
For example, companies programming in the large do want a steady influx of new Haskellers that can (eventually) fill out their hiring pool.

Furthermore, standard libraries still serve a *social* function that benefits programming in the small and large alike.
Little libraries all the way down represents apex of pluralism, of people being able to explore their own vision of what programming in the language ought to look like.
But there can be too much experimentation, and not enough cross-pollination of ideas.
The standard library reflects a chance to get together, hash out our differences, and maximize what we all agree on.
Again, we see indirect benefits of programming in the large.
For example, companies not only want a hiring pool of Haskellers on paper, but a pool of programmers who have some idea what the norms and idioms used in their codebases are.
Shared norms and idioms promote a single community rather than family of communities, and make it easier to switch between jobs and projects one works on without feeling like one is starting over completely.

Synthesis
~~~~~~~~~

So if we want to have little libraries for technical reasons, but large feature-rich standard libraries for social reasons, what do we do?
Both!
The original definitions of just about everything be incubated in little libraries, and continue to live in little libraries.
Standard libraries should have very little of their own definitions, but just focus on reexports, their role is not to *invent*, but to *curate*.
Plans today in the works like *moving* ``Profunctor`` to ``base`` should instead become having the new standard libraries merely *depend* on the ``profunctors`` library and reexport items.

In the `words of Shriram Krishnamurthi <https://twitter.com/ShriramKMurthi/status/1597942676560965634>`_, the slogan should not be "batteries included", but "batteries included — but not inserted".
When one just starts up GHCi without arguments, or runs ``cabal new``, one will get the nice feature-rich standard library loaded / as a ``build-depend`` by default,
but tweak a few flags and the cabal stanza, and its easy to remove those sledgehammer deps and just depend on exactly what one needs.

This is not normative!
~~~~~~~~~~~~~~~~~~~~~~

Hopefully the above appendix makes the vision of the proposal author more clear, but it should be equally stressed that this appendix is not normative.
Nowhere is the CLC being told exactly what the new standard libraries should look like.
Nowhere is it also specified how the implementation should be cut up behind the scenes.
But, if this proposal is to succeed, it seems like reaching a consensus position similar to the above compromise between two extremes is likely to be necessary.
