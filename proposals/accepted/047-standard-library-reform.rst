Standard library reform
=======================

.. sectnum::
.. contents::

Abstract
--------

Issues with the standard library are holding back the Haskell ecosystem.
The problems and solutions are multifaceted, and so the Haskell Foundation in its "umbrella organization" capacity is uniquely suited to coordinate the fixing of them.

THere are many such issues, but the one have chosen to focus on first in this proposal is:

> No clear boundary between GHC private/unstable library support code and public/stable standard library interfaces.
  `CLC Issue #105`_.

By reshuffling our interfaces and implementations alike, we should be able to solve this problem.

.. _`CLC Issue #105`: https://github.com/haskell/core-libraries-committee/issues/105

Background
----------

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
The benefits of (2) are mainly for "`programming in the small <https://en.wikipedia.org/wiki/Programming_in_the_large_and_programming_in_the_small>`" and end applications.
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

Problem Statement
-----------------

Each of items from the abstract is described in detail.

**Problem 1**: No clear boundary between private/unstable and public/stable interfaces in the standard library
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

Prior Art and Related Efforts
-----------------------------

Prior attempts at splitting ``base``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For years, there has been much interest in splitting `base`.
The `GHC Wiki page on "Split Base" <https://gitlab.haskell.org/ghc/ghc/-/wikis/split-base>`_ offers good context for this.
Especially notable is Joachim Breitner's `prior attempt <https://github.com/nomeata/packages-base/blob/base-split/README.md>`_, which offers good ideas backed by experience on where the natural cleavage points within ``base`` lie.

A problem with prior attempts is that they attempted to get everything done at once, setting a dangerously high bar for success.
This approach in this proposal, by contrast, first and foremost seeks to avoid those difficulties and find a sustainable, suitably low-risk approach.
It is much more concerned with how we safely approach these issues than what the exact outcome looks like.

Solution Statement
------------------

This is where we want to end up.
The proposal will break down these goals into an incremental roadmap to be actionable, but it is good to understand the destination before trying to understand the journey that gets us there.

#. Interfaces: We should have new standard library interfaces with clearly stated purpose, stability, and portability.

#. Implementation: Both ``base`` and the new standard library interfaces alike should be maximally decoupled from GHC.

Based on the conversation in `CLC Issue #105`_, ``base`` is exposing too much stuff, yet trying to limit what is exposed would be a big breaking change.
The solution is to make new standard libraries which don't need to worry about compatibility with ``base``.
The underlying *definitions* should be the same, e.g. we don't want two ``Functor`` classes, but we are free limit what we export to clarify the purpose --- what is in scope and out of scope --- for these new libraries.

- These libraries should be emphasized in all documentation, and users should be encouraged to use them and not ``base`` in new code.
  ``base``, in contrast, would be kept exposed as a mere legacy interface.
  As code migrates over to use the new standard libraries, ``base`` should become less important.
  GHC devs can therefore feel increasingly confident modifying parts of ``base`` which are *not* reexported in these new libraries.

  This fixes **Problem 1**.

- They should be designed *not* to break every GHC release.
  Breaking changes will still occur, but they should be less frequent.

  This fixes **Problem 2**.

- The new standard library should not be a single library but multiple libraries.
  IO-free interfaces that are portable everywhere should be one library.
  Interfaces involving IO should be split into libraries where they run.

  For example:

  - "Internal" ``IO`` which is just safe mutation with in the program rather than communication with the outside world, is also very portable.
    Item's like ``IORef``, ``MVar``, ``STVar``, mutable arrays, and STM can be defined in a library that works everywhere.

  - Unix and Windows are mostly a superset of WASI, so WASI-compatible file-descriptor-oriented code should work everywhere.

  Less portable standard libraries can publicly (in their interface) depend on more portable ones.
  For example, ``IO`` defined in the "internal IO" library is then given external IO capabilities in the WASI library.

  This all fixes **Problem 3**.

- The new libraries should not artificially limit themselves to items defined in ``base`` today, they can also reexport items from libraries, like ``text``.

  This fixes **Problem 5**.

It's fine to imagine what nicer interfaces look like, but just as important is how the implementation is structured, even though this wont be so directly visible to users.
The goal here must be this 

- Maximally decouple definitions from GHC, and furthermore to do this for the new libraries and ``base`` alike.

  This fixes **Problem 4**.

Additional rationale for implementation splitting
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Suppose instead that we just reached for another layer of indirection, and made the new libraries depend on ``base``, reexporting certain items.
This is quite easy, and all the problems addressed above are still fixed.
However, **Problem 4** was not mentioned above, and indeed it is *not* fixed.
Let's investigate why.

``base``, in this alternative plan, still contains everything it does today, and still has its regular problematic major version bumps.
But a version bump for ``base`` doesn't mean *everything* inside it had a breaking change, just *something* did.
In the happy path, that something is an internal implementation detail that none of the new standard libraries can reexport, so they can keep their major the version the same even as they reexport items from a newer major version of ``base``.
It is a bit difficult and error-prone to ensure only non-reexported items changed with today's tooling, but we can manually audit, and this works.

However, in the sad path, the something that changed is reexported, and now we have a problem.
We can bump the major version in the new library and bump the major version of the dependency on ``base``, satisfying the PVP, but now we are making a breaking change in lockstep with ``base`` which is released in lockstep with GHC.
**Problem 4** is still here rearing its ugly head --- users still need to go through the painful and higher risk process of upgrading everything at once.

So long as everything is defined in ``base``, there isn't a good solution to this.
We can put multiple versions of ``base`` in the GHC repo, allowing a staggered problem, but this results in quadratic overhead.
Given a N-release policy, where each major version of the new standard libraries must work with at least the previous N GHCs to allow staggered upgrades, we'll have to put up to N copies of ``base`` in with GHC to support all N N-long intervals that contain that GHC version.
Diagrammed for 3 this is::

  GHC  A, B, C, D, E, ...
  Base
  1    +--+--X
  2       +--X--X
  3          X--X--X
  4             X--X--...
  5                X--...

See how GHCs C, D, and E end up needing to support 3 ``base`` versions each.
This means we end up with quadratic N^2 versions of ``base`` being supported at a time, assuming base and GHC each support N major versions at a time.
This is a non-starter: too much code to maintain on an ongoing basis given our limited resources.

However, suppose instead we have broken up the implementation.
GHC-specific code lives with GHC in the same repo, but GHC-agnostic code lives in separate repos.
If we need to make a change to that GHC-agnostic code (for example, adding a superclass constaint to ``Bifunctor``, `CLC Issue #91`_), we just do it.
Because that code is, by definition, GHC agnostic, it is trivial to support multiple GHC versions, expanding dependency version bounds on GHC-specific libraries as need be.
Either the GHC-agonistic code is itself one of these standard libraries, or it is reexported in them;
either way, it is thus easy to release two version of the standard library, one exporting the old version of the standard library and one exporting the new version, and each supporting overlapping ranges of GHC versions.

With the above, we still have to CI N^2 build plans in the worst case, but we no longer have N^2 copies of the code.
The division of the labor between the GHC-agnostic and GHC-specific libraries should be comprehensible, and thus this shouldn't feel like arbitrary configuration combinatorial explosion, each of which could go *wrong*, but easy to undersand and well-typed composition, all of which should go *right* by construction.

The bottom line is the work of humans in the loop should, by and large, *not* be N^2, and only the far-cheaper machines, double checking our work with CI, should be shouldering the N^2 burden.
This takes something which was too costly to be feasible and makes it affordable.
In fact it should be *less* work than today because by "liberating" GHC-agnosic code from the GHC repo, we are reducing the surface are of supporting multiple GHC versions.

.. _`CLC Issue #91`: https://github.com/haskell/core-libraries-committee/issues/105

Alternative Preludes
~~~~~~~~~~~~~~~~~~~~

Technical Roadmap
-----------------

The end goal is layed out above (with some details such as exactly which libraries we want).
But that doesn't tell us how to get there.

Below is a roadmap to reach our end goal with an emphasis on reducing risk.
The goal is that the foundation should provide an extra boost at key moments, but between them the work should be broken down into very small bite-size chunks that are easier for volunteers to tackle.

See below in budget: *only the first step is normative* in the sense of asking for resources.
The rest are just to illustrate a possible larger context and how the problems of the motivation will be addressed.

**Step 1**: Shim ``base`` with new ``ghc-base``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Everything in ``base`` will be moved to a new library ``ghc-base``, and ``base`` will just reexport its contents.

Before we get into deciding what definitions ought to live where, and moving them there, we need to make sure that it's possible to move around definitions at all.
Today, ``base`` is treated specially in a few ways.
For example:

- It is the library that GHCi loads by default.

- GHC's compilation is directly aware of it in the form of various "wired-in" identifiers.

- Some modules of it are automatically trusted with Safe Haskell.

In the new multi-library world, different libraries will inherit these special features, and we cannot be sure what the ramification will be until we try.

It is best to "practice" this by shimming ``base`` like this as soon as possible.
That will reduce the risk of everything else by both exploring "known unknowns" and scouting ahead for "unknown unknowns".

The first steps of `GHC issue #20647`_ track what needs to be done here.
The key first step is finishing `GHC MR !7898`_.
This is crude: a ``ghc-base`` that ``base`` merely reexports in full is just as ugly as the original ``base``, but this is the quickest route to de-risking the entire project as described.

.. _`GHC issue #20647`: https://gitlab.haskell.org/ghc/ghc/-/issues/20647
.. _`GHC MR !7898`: https://gitlab.haskell.org/ghc/ghc/-/merge_requests/7898

Timeline
--------

Only **Step 1**, the preliminary exploration step, is being formally proposed at this time.
The rest is just there to illustrate how we could build upon it up towards the full solution addressing all problems.

Once that is completely, not only will we have a better idea of what challenges remain, we (assuming success) should have a bunch of incremental and parallel work that is better suited for volunteer or otherwise small-scale efforts.

Based on how that proceeds, follow-up tech proposals could be submitted in the future.

Budget
------

**Step 1** costs
~~~~~~~~~~~~~~~~~

Finishing `GHC MR !7898`_ is conservatively estimated to take 1 person-month of work from an experienced GHC dev.
The HF should finance this work if there are no volunteers to ensure it is done as fast as possible, as everything else is far too uncertain until this trial round of splitting and reexports has been completed end to end.

Stakeholders
------------

The Core Libraries Committee
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The latter steps give the CLC new material from which to curate the new standard libraries.
We can do the work without being blocked on the CLC, but ultimately we will need their blessing for any new libraries to reach the "cultural" primacy of ``base``.

GHC developers
~~~~~~~~~~~~~~

`GHC MR !7898`_ from **Step 1** has uncovered some bugs that will need fixing.
The later steps will eventually result in churn among which submodules GHC contains, which will be frustrating until that stabilizes.

Due to **Problem 4**, the interest and cooperation of the developers of our new backends is especially solicited.

Success
-------

The project will be considered a success when all the enumerated problems are solved per their "solution criteria" (no moving the goalposts later without anyone noticing), and the standard library implementation is easier to maintain than before.

Future work
-----------

It may seem that this first problem and solution are rather far-removed from actual users needs.
This proposal was originally just one part of a far larger proposal that did "build up" from this work fixing a problem behind the scenes to a more visible "end-problem".
However, committing to a complete plan to address all of these in one go is not feasible, so the rest was moved to (currently draft)
`Proposal 49 <https://github.com/haskellfoundation/tech-proposals/pull/49>`

Still, for readers interested in understanding everything in context, it may be helpful to read both proposals.
Of course, as a separate proposal acceptance of this one does not imply acceptance of the next.
But understanding what we *may* like to do next may still put this one in better context.

Appendix: What are standard libraries for?
==========================================

*If parts of this proposal seems hard to understand or surprising, background information in the form of the author's critical view on the very concept of a standard library me prove illuminating.*

Synthesis
---------

So if we want to have little libraries for technical reasons, but large feature-rich standard libraries for social reasons, what do we do?
Both!
The original definitions of just about everything be incubated in little libraries, and continue to live in little libraries.
Standard libraries should have very little of their own definitions, but just focus on reexports, their role is not to *invent*, but to *curate*.
Plans today in the works like *moving* ``Profunctor`` to ``base`` should instead become having the new standard libraries merely *depend* on the ``profunctors`` library and reexport items.

In the `words of Shriram Krishnamurthi <https://twitter.com/ShriramKMurthi/status/1597942676560965634>`_, the slogan should not be "batteries included", but "batteries included — but not inserted".
When one just starts up GHCi without arguments, or runs ``cabal new``, one will get the nice feature-rich standard library loaded / as a ``build-depend`` by default,
but tweak a few flags and the cabal stanza, and its easy to remove those sledgehammer deps and just depend on exactly what one needs.

This is not normative!
----------------------

Hopefully the above appendix makes the vision of the proposal author more clear, but it should be equally stressed that this appendix is not normative.
Nowhere is the CLC being told exactly what the new standard libraries should look like.
Nowhere is it also specified how the implementation should be cut up behind the scenes.
But, if this proposal is to succeed, it seems like reaching a consensus position similar to the above compromise between two extremes is likely to be necessary.
