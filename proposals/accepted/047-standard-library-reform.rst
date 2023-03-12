Standard library reform: Split base
===================================

.. sectnum::
.. contents::

Abstract
--------

Issues with the standard library are holding back the Haskell ecosystem.
The problems and solutions are multifaceted, and so the Haskell Foundation in its "umbrella organization" capacity is uniquely suited to coordinate the fixing of them.

THere are many such issues, but the one have chosen to focus on first in this proposal is:

..

  No clear boundary between GHC private/unstable library support code and public/stable standard library interfaces.

(This is also `CLC Issue #105`_.)

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
The benefits of (2) are mainly for `"programming in the small" <https://en.wikipedia.org/wiki/Programming_in_the_large_and_programming_in_the_small>`_ and end applications.
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

..

  No clear boundary between private/unstable and public/stable interfaces in the standard library.

The long discussion thread in `CLC Issue #105`_ demonstrates this exceedingly well.

On a simpler level, the lack of a firm boundary confuses users, who don't know which parts of ``base`` they ought to use, and GHC developers, who don't know what parts they are free to change.

On a more meta level, I think everyone in the thread was surprised on how hard it was to even discuss these issues.
Not only is there no firm boundary, but there wasn't even a collectively-shared mental model on what exactly the issue is, and how to discuss it or its solutions!
This is a "tower of Babel" moment where the inability to communicate makes it hard to work together.

Solution criteria
~~~~~~~~~~~~~~~~~

We should use standard off-the-shelf definitions and techniques to enforce this boundary.
The standard library should not expose private, implementation-detail modules.
The entirety of the standard library's public interface should be considered just that, its public interface.
Private modules that we do wish to expose to code that *knowingly* is using unstable interfaces should be exposed from a separate library.
The standard library should use regular PVP versioning.

In solving the immediate problem this way, we also solve the meta problem.
Using off-the-shelf definitions gives us a shared language reinforced by practice in the rest of the Haskell ecosystem.

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

Alternative Preludes
~~~~~~~~~~~~~~~~~~~~

Splitting base may feel superficially like various alternative prelude / alternative standard library projects.
Indeed, on a technical level, adding a new layer on top, and then shuffling definitions around, vs shuffling and then splitting, are two routes to the same destination.

However, on a social level, they are very different.
We have an unclear division of labor between GHC developers and the CLC which was want to resolve right away --- this proposal immediately addresses that, but a layering on top approach would mean setting up a new library with presumably new governance, and only later seeing ``base`` "decay" into a GHC-specific legacy library.

Similarly, by splitting first, and keeping (at least for now) ``base`` as the name of the library users are intended to so, we ensure that existing programs (and their maintainers) benefit from the clearer governance and division of labor right away.

It may turn out that making new standard libraries and relegating ``base`` to a user-facing but legacy status is still a good idea.
This proposal doesn't prevent that, and ``ghc-base`` (or whatever ``ghc-*`` libraries it itself may split into) are still good building blocks for ``base`` and any brave new standard libraries alike.

Technical Roadmap
-----------------

Shim ``base`` with new ``ghc-base``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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

Validate the split
~~~~~~~~~~~~~~~~~~

Having made the a crude split base, we want do *something* to validate that this has put us on track to solving our division of labor issues.
This could take a few forms:

Deprecate or even remove the reexport of a highly-GHC-specific internal definition
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Items in the ``GHC.*`` namespace, for example, were originally placed there because they were outside the modules specified in the Haskell Report, and specific to GHC.
Many of them, like ``GHC.Generics``, are both wide use and may not even be exposing implementation details.
Some of them, however, are very tied to implementation details are used far less directly.

It is that latter sort that a ``ghc-base`` vs ``base`` split eventually seeks to relocated permanently out of ``base``.

Based on the results of an impact analysis, we could remove them out of ``base`` right away, or deprecate the reexport to indicate such removal in the future is planned.

Move back out of ``ghc-base`` highly portable definitions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The flip side of relegating implementation specific code to the lower library is that implementation agnostic code should be able to live in the higher library, and someday even be built against multiple GHCs / ``ghc-base `` versions.

For example, since `CLC Issue #10`_, ``Data.Functor.Classes`` is only used in 3 other ``Data.Functor.*`` modules to define instances which are themselves unused by the rest of base.
The instances could instead be moved to ``Data.Functor.Classes`` itself, at which point nothing else would depend on that module, and it also depends on nothing implementation-specific (in principle).
Finally, that module could be moved back to ``base`` from ``ghc-base``

.. _`CLC Issue #10`: https://github.com/haskell/core-libraries-committee/issues/10

Timeline and Budget
-------------------

Shim ``base`` with new ``ghc-base``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Finishing `GHC MR !7898`_ is conservatively estimated to take 1 person-month of work from an experienced GHC dev.
The HF should finance this work if there are no volunteers to ensure it is done as fast as possible, as everything else is far too uncertain until this trial round of splitting and reexports has been completed end to end.

*This section could be fleshed out with more concrete roles and responsibilities, or that can be figured out post-acceptance.*

Validate the split
~~~~~~~~~~~~~~~~~~

This item does not need example Haskell Foundation funding.
Both examples are ultimately social exercises for GHC devs and CLC members in getting familiar with a budding new division of labor and responsibility.
The actual technical effort needed to shuffle a reexport or move some instances is miniscule, not more than one day's work.

Stakeholders
------------

The Core Libraries Committee
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The latter steps give the CLC new material from which to curate the new standard libraries.
We can do the work without being blocked on the CLC, but ultimately we will need their blessing for any new libraries to reach the "cultural" primacy of ``base``.

GHC developers
~~~~~~~~~~~~~~

`GHC MR !7898`_ has uncovered some bugs that will need fixing.
The later steps will eventually result in churn among which submodules GHC contains, which will be frustrating until that stabilizes.

Success
-------

The project will be ultimately considered a success when the problem is solved per their "solution criteria" (no moving the goalposts later without anyone noticing).
That means, for example, that everything that doesn't belong in ``base``'s public interface by virtue of being too implementation-specific or unstable is deprecated or removed altogether.

The actually proposed technical roadmap doesn't take us this far, however. The triaging and moving of most items is left as an exercise for the CLC and GHC devs to perform incrementally and cheaply, without direct HF involvement.

Because of this, we need a narrower definition of success which is just for the parts of the work HF is directly overseeing (and possibly funding):

 - There is a new library called ``ghc-base``
 - ``base`` depends on ``ghc-base``
 - ``base`` only contains definitions which are portable across GHC versions / ignorant of GHC internal interfaces.
   Others items exported by ``base`` must be reexports, presumably from ``ghc-base``.
 - ``base`` contains at least on portable definition (it doesn't reexport everything).
 - ``base`` has at least one interface deprecated or removed from before, intended to intend be accessed only from ``ghc-base``.

Future work
-----------

It may seem that this first problem and solution are rather far-removed from actual users needs.
This proposal was originally just one part of a far larger proposal that did "build up" from this work fixing a problem behind the scenes to a more visible "end-problem".
However, committing to a complete plan to address all of these in one go is not feasible, so the rest was moved to (currently draft)
`Proposal 49 <https://github.com/haskellfoundation/tech-proposals/pull/49>`

Still, for readers interested in understanding everything in context, it may be helpful to read both proposals.
Of course, as a separate proposal acceptance of this one does not imply acceptance of the next.
But understanding what we *may* like to do next may still put this one in better context.
