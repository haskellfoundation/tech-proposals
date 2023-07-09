================
GHC's base libraries
================
----------
Combining stability with innovation
----------

:Date: June 2023
:Authors:
  John Ericson — secretary,
  Ben Gamari — GHC,
  Adam Gundry — GHC,
  Andrew Lelechenko — CLC,
  Julian Ospald — CLC,
  Simon Peyton Jones — GHC

.. sectnum::
.. contents::

Introduction
=========

This document describes a plan agreed between the GHC Team and the Core Libraries Committee, saying how we plan to work in partnership to reconcile the goals of innovation and stability for GHC and its ecosystem.

Formally, then, it is not so much a proposal as a record of an outcome.
We are, nevertheless, using the Haskell Foundation Technical Working Group proposals repo, so that we can have a permanent public record of how we plan to work, so that others can comment, and so that the document can be polished to add clarity where necessary.

Goals
=====

This proposal allows the GHC team and the Core Libraries Committee to work in a productive partnership.
What we want to achieve is this:

* The CLC has full autonomy over decisions affecting ``base`` (API, performance, semantics), including the indirect consequences on ``base`` of changes to its dependencies.

* The GHC team has maximum freedom (consistent with the CLC's curation of ``base``) to:

  * Innovate in the language design.
    GHC has hundreds of extensions, and people suggest more all the time, via the GHC Proposals process.

  * Move rapidly to fix bugs, improve performance, and refactor GHC's internals to pay down technical debt.

The proposal sets up
mechanisms (e.g. what packages exist),
responsibilities (e.g. who curates which package),
and processes (e.g. who should be consulted and when)
that support both CLC and the GHC team to follow their respective goals without tripping over each other.

The proposal is based on `HF Proposal 47 <https://github.com/haskellfoundation/tech-proposals/pull/47>`__, but is independent of it.
You can find more background in `CLC issue #146 <https://github.com/haskell/core-libraries-committee/issues/146>`__.

Things we all agree about
=========================

Here are some points that everyone agrees about:

1. ``base``, and other packages that come with GHC, should adhere rigorously to the PVP.

2. Any complicated package, certainly including ``base``, has implementation "internals" that it may want to expose to hard-core users, but not to regular clients.
   The right pattern for accommodating this is described in `Nikita's blog post <https://nikita-volkov.github.io/internal-convention-is-a-mistake/>`__: have two packages, an "internals" one exposing the internals, and a "stable" one that exposes the stable API. Both adhere to the PVP.

3. We should use this pattern for ``base``.

4. The Core Libraries Committee curates the API of ``base`` (here is `the charter <https://github.com/haskell/core-libraries-committee#base-package>`__), including:

   - Types, specifically including what instances are exposed

   - Semantics (including strictness)

   - Performance

   - Semantic changes to documentation (the charter says *"Documentation changes normally fall under GHC developers purview, except significant ones (e.g., adding or changing type class laws)."*)

   Beyond that, it has no interest in the implementation details (e.g. alpha renaming, moving things between modules, comments).

5. In curating the ``base`` API, it is immaterial where code lives.
   For example, if a change to ``ghc-prim`` changes the ``base`` API (as defined above) the GHC developers must consult the CLC.
   The fact that the change isn't physically part of the ``base`` package is immaterial.
   (Incidentally, ``base`` and ``ghc-prim`` are both part of the same GitHub repository, which also includes GHC's source code.)

Proposal
========

We propose to divide ``base`` into three packages:

- ``ghc-internals``: exposes aspects of GHC's internals that may be of interest to "hard-core" developers interested in maximum performance (see `Nikita's blog post <https://nikita-volkov.github.io/internal-convention-is-a-mistake/>`__).
  The API of ``ghc-internals`` is fully under the control of the GHC team, and of no direct interest to the CLC — only its effects on the API of base.

- ``base``: as now, whose API is curated by CLC.
  Depends on ``ghc-internals``, and hence on ``ghc-bignum`` and ``ghc-prim``.

- ``ghc-experimental``, initially empty, depends on ``base`` and on ``ghc-internals``.
  Functions and data types here are intended to have their ultimate home in base, but while they are settling down they are subject to much weaker stability guarantees.
  Generally, new functions and types introduced in GHC Proposals would start their life here.
  Example: new type families and type constructors for tuples, `GHC Proposal #475 <https://github.com/ghc-proposals/ghc-proposals/pull/475>`__.

  Another example: future APIs to access RTS statistics, which are fairly stable and user-exposed, but which are (by design) coupled closely to GHC's runtime and hence may change.

  As its name suggests, the API of ``ghc-experimental`` is curated by the GHC team, although the CLC is willing to offer (non-binding) opinions, if consulted.

All three packages conform rigorously to the PVP.
(But see Section 5.3)

Some observations about this structure:

- We should develop both social and technical mechanisms to discourage people from depending directly on ``ghc-internals``, because if such dependencies become frequent and ossified, it will lead to future pain when the API changes.
  The very name ``ghc-internals`` should serve as a very strong signal in its own right, but even so, saying "we told you not to rely on it" may be true but won't lessen that pain.
  The specific mechanisms do not form part of this proposal, but some possibilities are `discussed in a separate section <#discourage-brainstorm>`__.

  Note: "discourage" does not imply "ban".
  It must remain possible for hard-core developers to depend on `ghc-internals`.
  Our goal is only that naive developers should not do so by accident.

- In contrast, clients are *not* discouraged from depending on ``ghc-experimental``; although again its name should convey the idea that it might change at short notice.

  ``ghc-experimental`` allows the GHC Steering Committee to make initially-experimental language changes, which often involve new types and functions, without committing to permanently supporting the precise API, since it often takes a little while for these designs to settle down.

  The existence of ``ghc-experimental`` should substantially ameliorate the difficulty that many GHC Proposals have a library-function component, but it is unlikely to be a *stable* API (having just been invented) and is therefore in conflict with the CLC's goals.

  As they become stable, the CLC may want to consider adopting the new types and functions from ``ghc-experimental`` into ``base``.
  (But CLC would not expect to curate the API of ``ghc-experimental``.)

- Perhaps ``ghc-experimental`` should be in the purview of the GHC Proposals process.
  GHC devs should not just make up random APIs and pop them into ``ghc-experimental``; a scrutiny process would be valuable.

- Under this proposal, there is initially no change (whatsoever) to the API exposed by ``base``, or its performance characteristics.
  The impact on clients should therefore be zero.

  Over time, the GHC developers may make CLC proposals to remove types and functions that are currently in the ``base`` API, but are in truth part of GHC's implementation, and were originally exposed by historical accident.
  But these are *future* proposals.

  To make the transition suggested in these future proposals easier to manage, we have in progress a `"deprecated exports" <https://github.com/ghc-proposals/ghc-proposals/pull/595>`__ mechanism that will ease such transitions.
  For a transitional period, ``base`` can continue to export the function, but with a deprecation warning saying something like:

    This is going to disappear from base.
    You probably don't want to use it at all.
    But if you absolutely must, get it from ``ghc-internals``.

- To expose a new function from ``ghc-internals`` requires that any functions on which it depends are also in ``ghc-internals`` (not base).
  So we may need to move code from ``base`` to ``ghc-internals``, leaving a shim behind in base.
  In practice, that may mean that quite a lot of code will move into ``ghc-internals`` quite quickly.
  But that's fine: *it is just an implementation matter*: provided the modules, exports, and API of ``base`` are maintained, it is immaterial to clients (and hence to CLC) exactly *how* they are maintained.

- This proposal is fully compatible with, and actively supports, the `CLC charter <https://github.com/haskell/core-libraries-committee#base-package>`__:

    The primary responsibility of CLC is to manage API changes of ``base`` package.
    The ownership of ``base`` belongs to GHC developers, and they can maintain it freely without CLC involvement as long as changes are invisible to clients.
    Documentation changes normally fall under GHC developers purview, except significant ones (e.g., adding or changing type class laws).

- It also supports GHC innovation, by

  - allowing GHC freedom to change aspects of its implementation

  - allowing the GHC Steering Committee to add new functions and types in ``ghc-experimental``.

- The three "internal" packages: ``ghc-internals``, ``ghc-bignum``, and ``ghc-prim``, could arguably be a single package, but the GHC team has decided that encapsulating the relevant code in this way helps to keep dependencies and responsibilities clear.
  And it's purely an internal GHC matter; if the team wants to structure GHC's internals with three packages, or ten, that's up to them.

Continuous integration
======================

A major difficulty is **knowing when the API of 'base' (as defined in Section 2) has changed.** A change requires CLC approval; but how do we know what commits (to ``base``, to ``ghc-internals``, to ``ghc-prim``) make such a change?

In the past we have relied on best efforts; but with a bunch of volunteers, mistakes will be made.
And mistakes can lead to a loss of trust.

The solution is obvious: we need to automate.
We therefore propose the following, as part of CI:

1. Compile a good chunk of Hackage (around 500 packages) against base.
   We already do this, and it is a huge help in reassuring ourselves that a change does not lead to accidental breakage.

2. Test if any of the types (incl their kinds), functions (incl their types) and instances exposed by the ``base`` API are accidentally changed by a commit.
   This is definitely going to happen, soon: @bgamari already has a prototype.

3. Run the test suite of those packages that have a testsuite that

   (a) is usable (e.g. that doesn't take too long to run),

   (b) does not have dependencies that are outside the set mentioned in point (1), and

   (c) passes before the change to GHC/``base``.

   This checks semantics as well as types.

4. Running the performance test suite of some carefully chosen packages.
   This checks for performance regressions.
   Similar to (3), except that perf suites are less common and often more expensive to run.

5. Develop a new suite of performance tests, specifically for base.
   This is quite open-ended; it is not clear what would be desirable, or how much it would cost.

Some modules in ``ghc-internals`` will very directly affect exports of ``base`` (e.g via shim).
These modules could be identified, via the existing ``CODEOWNERS`` mechanism, to ping CLC on any commit to those modules.
This list could be selective, or include all of ``ghc-internals``, at CLC's preference.

Some of these are cheap to do; others are less so.
Fortunately the HF seems willing to help.

*But whatever we do here will be a step forward* from our current, unsatisfactory situation.
Moreover, they will help with CI for changes to GHC itself! (It is rather *more* likely that a commit to GHC's simplifier will cause a perf regression in some package, than a commit to ``ghc-internals``.)

Discussion
==========

Discourging the (direct) use of ``ghc-internals``
-------------------------------------------------

.. _discourage-brainstorm:

Here are some ideas to be explored later for how to discorage the use of ``ghc-internals``.

- The name ``ghc-internals`` is a pretty strong signal all by itself.

- Cabal description and README explains how it is intended used (and not used).

- Hoogle could (by default anyway) never show stuff from ``ghc-internals``.

- Do not upload Haddocks for ``ghc-internals`` to Hackage.
  (Ditto ``ghc-prim``.) Need to make sure that if someone wants to follow the Haddock source-code link to (say) Functor, they should still find it regardless of where it is actually defined.

- We could consider issuing a warning if you say ``-package ghc-internals`` (or ``ghc-bignum`` or ``ghc-prim``), one that was hard to silence.
  Since we can have module-level ``WARNING`` pragmas with custom categories, one way to realise this would be to pick a category and add such pragmas to every module in the relevant packages, though we might want to do something more systematic.
  The text of the warnings could encourage users to

  - switch to a function exposed by base, and/or
  - petition the CLC to expose this super-useful function from base.

- ``cabal check`` (a per-package check) could warn on packages that use ``ghc-internals``.

GHC Proposals process
---------------------

Some GHC proposals (a minority) directly affect the existing API of ``base``, and are not simply additions that can be exposed in ``ghc-experimental``.
It is unproductive for the GHC Steering Committee to have a long discussion, accept the proposals, and only *then* involve the CLC.

We propose that:

- A GHC Proposal should advertise, in a separate section:

  -  What changes, if any, it make to ``ghc-experimental``

  -  What changes, if any, it make to ``base``

- If there are any such changes, the author (and shepherd) should explicitly invite the CLC to participate in the discussion about the proposal.
  The CLC will devote some effort to participating and, in the case of changes to ``base``, will subsequently hold a non-binding vote.

- Approval of the proposal (by the GHC Steering Committee, with the non-binding vote of CLC) is not a guarantee that the final implementation will land;
  that depends on the implementation being well engineered etc (GHC team);
  and the implementor should make an explicit proposal to the CLC specifying the precise changes.

Abstraction leakage
-------------------

We may foresee a couple of ways in which changes in ``ghc-internals`` could become client visible:

- Occasionally, an error message may mention a fully qualified name for an out-of-scope identifier.
  For example (GHC test ``mod153``)::

    Ambiguous occurrence ‘id’
               It could refer to either ‘Prelude.id’,
                            imported from ‘Prelude’ at mod153.hs:2:8
                            (and originally defined in ‘GHC.Base’)
                         or ‘M.id’, defined at mod153.hs:2:21

  The "originally defined in" mentions a module; and if that module is in a package that is not imported, GHC will package-qualify the module name.
  And seeing ``ghc-internals:GHC.Base`` is perhaps less nice.
  This is not a new problem: we already package-qualify modules in ``ghc-prim``.
  One solution is to remove the "originally defined in.." parenthesis for types and functions that would require such package qualification.

- Another form of leakage could be: a new class in ``ghc-internals``, *not exposed in base*, that is given instances for existing data types.
  There is a risk that those instances might confusingly be visible to clients of ``base``.
  If so, the CLC should at least be consulted.

These issues concern error messages and documentation, neither of which are in the direct scope of CLC.
They are not new because we already have ``ghc-prim``.
They may not be show-stoppers, but we should be thoughtful about mitigating them.

Versions and backports
----------------------

We agree that the version number of ``ghc-internals`` may have a major bump between minor releases of GHC.
(Why? Because to fix the bug we change something in ``ghc-internals``.)

This makes an exception to a general rule: generally, a minor release of GHC (say 9.6.4) which only fixes bugs, never makes a major version bump to ``base``, or indeed any boot package.

We should discuss this (rather important) exception with the Stackage curators.

But this same issue could in principle affect ``base`` too.
Very occasionally a **bug-fix** might involve a change to the user-visible API.
Example: `role annotations on SNat <https://github.com/haskell/core-libraries-committee/issues/170>`__ (although there is a debate as to whether this specific change constitutes a "breaking change" under the PVP).

Under these circumstances we (together) will have to decide whether to

- Back port the fix, and not bump the major version of ``base`` (i.e. bend the PVP), or
- Bump the major version of base, but therefore be unable to fix the bug in the released GHC.

This is a decision for the CLC.
See PVP issue https://github.com/haskell/pvp/issues/10.

New classes
-----------

Suppose the author of a new library ``l`` defines a new class ``C``.
Good practice is for them to define an instance of ``C`` for all types in boot packages (packages needed to build GHC and Cabal).

Should ``ghc-experimental`` be considered a boot package in this sense?
After all, type ``S`` in ``ghc-experimental`` may change, which would break ``l``.
Agreed answer: no.
That is, we do not make it best-practice for library authors to give ``C`` instances for types exported only by ``ghc-experimental``.
(They can, of course, but it's fine not to.)

Other teams to consult
======================

There are other stakeholders in this space who we should consult, in addition to seeking GHC Steering Committee and CLC approval:

**Stackage curators**

- Is it OK to make a major bump in ``ghc-internals`` for a minor release of GHC?

**Haddock team**

- Hiding (in the documentation) instances that are not usable because the type or the class is not exposed.
  Not clear that this is worth a technical solution.

**Hackage team**

- Can/should we support hiding ``ghc-internals`` on Hackage?

**Security team** / **Stability working group**

- It might be easy for the new security-vulnerability mechanism to also flag packages that depend transitively on ``ghc-internals``.
  If they depend on it via ``base``, this is fine.
  But if they depend on it via another package, this could be a hazard migrating to a newer GHC the code authors were not aware of.

**HLint team**

- Can we add a check for imports from ``ghc-internals``?
