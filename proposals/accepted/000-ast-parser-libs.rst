===========================================
Split out AST and Parser libraries from GHC
===========================================

:Date: August 2023
:Authors:
  John Ericson,
  Shayne Fletcher,
  Laurent P. René de Cotret

.. sectnum::
.. contents::

Abstract
========

Problem
-------

The community lacks AST and Parser libraries for Haskell that are both self-contained and up-to-date.
Experience has shown that there is only way one way to meet each criterion:

- Be used by GHC, so the library cannot fall behind new language development

- Be separate from GHC, so the library is forced to be self-contained

However, no library has so far done both, nor met meet both criteria.

Solution
--------

The purpose of this proposal is to make that library finally exist.
The Haskell Foundation will finance the completion of the existing "Trees that grow" project, decoupling GHC's AST and parser from the rest of the compiler so they can be moved to separate libraries.
Those libraries will be "normal" Haskell libraries, without any unusual dependencies or build processes, and published on Hackage.
Those libraries will be used by GHC, ensuring they are maintained.

Background, Prior Art, and Related Efforts
==========================================

Making such a library has long been a goal of the Haskell community.
This section highlights various past and ongoing efforts accordingly.

*This section is purely informative; readers familiar with this backstory can skip this section and move on to the proposal proper.*

.. |haskell-src-exts| replace:: ``haskell-src-exts``
.. _haskell-src-exts: https://hackage.haskell.org/package/haskell-src-exts

.. |ghc-lib-parser| replace:: ``ghc-lib-parser``
.. _ghc-lib-parser: https://hackage.haskell.org/package/ghc-lib-parser

.. _HLint: https://hackage.haskell.org/package/hlint

|haskell-src-exts|
------------------

An older attempt is the venerable "Haskell-Source with Extensions (HSE)" package, also known as |haskell-src-exts|_ .
This is actually part of a larger project called the `"Haskell Suite" <https://github.com/haskell-suite>`_, the purpose of which was "to implement the whole Haskell compiler as a set of libraries".
However, the whole compiler doesn't appear to exist, and the project as a whole ceased development.
``haskell-src-exts`` lasted longer, but had great trouble keeping up with GHC, and is now also unmaintained since 2020.

|ghc-lib-parser|
----------------

In Februrary 2019, |ghc-lib-parser| `was released <http://neilmitchell.blogspot.com/2019/02/announcing-ghc-lib.html>`_ and in May 2019 an `announcement <https://mail.haskell.org/pipermail/haskell-cafe/2019-May/131166.html>`_ that there would be no further |haskell-src-exts| releases followed and the advice given for anyone wishing to parse Haskell programs to "use the GHC API, specifically you can use |ghc-lib-parser|".

A |ghc-lib-parser|_ package contains GHC compiler sources packaged as a library. [#ghc-inception]_
This ensures it is up to date with the latest behavior but this extraction process is complex, requires constant patching to keep pace with GHC evolution, and results in a far larger library than is desired:
see the *hundreds* of modules included inside it, many of which have nothing to do with the Haskell surface language.
All this `"bycatch" <https://en.wikipedia.org/wiki/Bycatch>`_ in the extraction process results in a library that daunting to use, and which has a hard time presenting a stable interface.

The lesson from |haskell-src-exts| are clear:

- it succeeded in being modular and self-contained but failed trying to keep up with GHC;

- given the size of the community, competing head-on with GHC or trying to keep up with it is very difficult:

  - being used by GHC, so keeping up happens automatically, is the clearest way to avoid this problem.

Whereas |ghc-lib-parser| succeeds in keeping up with GHC because it *is* GHC, it fails in being self-contained because modularity cannot be `"fixed in post" [production] <https://tvtropes.org/pmwiki/pmwiki.php/Main/FixItInPost>`_.
Code that is intended to be separate from any one consumer must be developed with those boundaries enforced during development.

.. [#ghc-inception]
  The extraction process was enabled by insights gained from the `"GHCinception" <https://mgsloan.com/posts/ghcinception/>`_ or "GHC in GHCi" initative.

Downstream projects
-------------------

In addition to going over the major AST/parser libraries for Haskell, it is also useful to talk about the most notable projects that use them.
Ultimately, it is those projects we want to help out.

HLint_, the Haskell linter project developed continuously since 2008 is the most notable one, not just because its longstanding and wide use, but also because many of the developers that worked on the previous two libraries also worked on it — use by HLint served as proof the libraries were fit-for-purpose.

In June 2019,
following the release of |ghc-lib-parser| and deprecation of |haskell-src-exts| earlier that year as described above,
`HLint began the transitition <http://neilmitchell.blogspot.com/2019/06/hlints-path-to-ghc-parser.html>`_ to |ghc-lib-parser|.
In May 2020, the release of HLint-3.0 which "uses the GHC parser" `was announced <http://neilmitchell.blogspot.com/2020/05/hlint-30.html>`_.

Today, most users of |haskell-src-exts| have largely migrated from |haskell-src-exts| to |ghc-lib-parser|. [#example-ghc-lib-parser-users]_
But just because all these projects are using |ghc-lib-parser| doesn't mean everything is well.
Shayne Fetcher reports that keeping up with the latest GHC chagnes with the |ghc-lib-parser|/``ghc-lib``/``ghc-lib-parser-ex``/HLint stack generally costs him **an hour or two a week, and often more**.
The cost of detail with changes to the AST is inevitable — supporting new language features will inevitably cost developer time.
But all the other busywork of re-extracting the code, etc., is entirely avoidable, *not* inherent to the task at hand.

It is the opinion of the authors of this proposal that should an independent AST parser libraries be maintained upstream with GHC, the costs saved for downstream developers should *greatly* exceed any costs incurred by GHC developers.
The goal is thus *not* to simply shift a burden from one group of community members to another, but create a positive-sum outcome where there is far less busywork and more flourishing tooling than before.

.. [#example-ghc-lib-parser-users]
  Today for example, notable users include
  HLint_,
  `ormolu <https://hackage.haskell.org/package/ormolu>`_,
  `ghcide <https://hackage.haskell.org/package/ghcide>`_,
  `hls-hlint-plugin <https://hackage.haskell.org/package/hls-hlint-plugin>`_,
  `hindent <https://hackage.haskell.org/package/hindent>`_,
  and
  `stylish-haskell <https://hackage.haskell.org/package/stylish-haskell>`_.

Trees that grow
---------------

As we can see, each of these prior two attempts did one of the two things right, and correspondingly met one of our two criteria.
There is, however, a third project, that over the years has aimed to allow us to finally hit both criteria: "Trees that grow".
The name comes from `this paper <https://www.microsoft.com/en-us/research/uploads/prod/2016/11/trees-that-grow.pdf>`_.
There are also
`some GHC Wiki pages <https://gitlab.haskell.org/ghc/ghc/-/wikis/implementing-trees-that-grow>`_,
and a `GHC Issue Label <https://gitlab.haskell.org/ghc/ghc/-/issues/?label_name%5B%5D=TTG>`_ for it.

The goal of the Trees that Grow paper was to allow creating variants of Haskell AST to more faithfully capture the input and output of each compilation pass, and also the ``template-haskell`` library. [#intra]_
It presents these data types:

.. code-block:: haskell

  data Component = Compiler Pass | TemplateHaskell

  data Pass = Parser | Renamer | TypeChecker

The idea that they are "promoted" via ``DataKinds``, and then type families used in the AST will have instances for these promoted values.
This allows those consumers to "adjust" the AST for their purpose.

The Trees That Grow project is now 6 years old, and has met great success in avoiding partiality in the compiler, "making illegal states unrepresentable" as many Haskellers would put it.
But progress on `reducing AST & parser dependencies <https://gitlab.haskell.org/ghc/ghc/-/issues/19932>`_ has been less easily forthcoming.
We have separated out the modules defining the AST under ``Language.Haskell.Syntax.*`` we wish to split out, and we have tests to track progress reducing their deps, and the parser's deps.
But progress is unsteady and unpredictable.

The basic problem is that the benefits don't actually kick in until the deps are *all* gone, and the code is actually separated out.
Partial progress isn't really directly useful to anyone, and these counters just scoreboard by which we hope to get closer to the end goal.
It is thus hard to do this work with volunteers only, because it is emphatically *not* `"itch scratching" <https://en.wikipedia.org/wiki/The_Cathedral_and_the_Bazaar>`_ work where incremental progress leads immediate incremental benefits to the contributor.

The Haskell Foundation's support in getting this "over the finish line", at which point the community *will* benefit, and benefit greatly, is thus a crucial way we can surmount the coordination failure the lack of incremental payoff causes.

.. [#intra]
  It might sound like the goal is only different usages within GHC, but remember that ``template-haskell`` is a separate library used by users of Haskell not just developers of Haskell.
  A goal of at least some usage outside GHC was always there.

Reinstallable GHC Lib
---------------------

One of the problems ``ghc-lib-parser`` aims to solve is that ``ghc`` the library is current cumbersome to install as a regular Haskell library (as opposed to by switching toolchains entirely).
There is currently work in flight to solve that.
One that is done, projects like HLint_ *could* just depend on ``ghc`` directly, and still be easily buildable (with Cabal / with Stack / from Hackage) as today.

Just doing this isn't a good solution though, because ``ghc`` exposes a much a wider surface area than what these projects actually want.
For stability's sake, it is better that those libraries dependent on narrower parsing / AST libraries that only provide what they need.
And longer term, we hope the "tug of war" of between GHC and these projects as consumers of those libraries, versus just the others having to deal with whatever GHC does with just itself in mind, will result in a higher-quality, more flexible, and overall friendlier library.

In `this comment <https://gitlab.haskell.org/ghc/ghc/-/issues/14409#note_506489>`_, it is suggested that factoring out the AST and parser can be a good first step making a more modular in GHC in general.
This proposal wishes to *stay neutral* on the merits of such a future direction, but it would be remiss not to at least highlight it as one possible outcome.

Roadmap
=======

The project is split into two separate steps: separating the AST, and separating the parser.
Each step has a method, time estimate, and (most importantly) clear success criteria, including use by downstream projects to ensure value is delivered.
The intent is thus that they are self-contained, and can be individually funded.

Separate the AST
----------------

Split library
~~~~~~~~~~~~~

**Executor**: Haskell Foundation

**Time Estimate:** 1 – 2 Weeks

The first step is just separating data definitions.
We don't need to worry about code entangling, just data entangling.
We have already separated those data definitions into modules in the ``Language.Haskell.Syntax.*`` namespace.

Concretely, the work in this step is to:

#. Modify those modules to not import any other modules in ``ghc`` (``GHC.*`` modules).

#. Move those modules to a new separate AST library in the GHC repo.

#. Adjust ``build-depends`` across the repo so ``ghc`` and any other Haskell Package gets those modules from the new library instead, and CI passes.

The timeline for this is pretty short because there exists an easy last-resort way to decouple anything:
just add another TTG type family.
This came up with some acrimony in `GHC Issue #21628 <https://gitlab.haskell.org/ghc/ghc/-/issues/21628>`_, discussing whether it was better to try to change GHC's ``FastString`` or abstract over it.
The purpose of this proposal isn't to relitigate that issue, but because this proposal *is* about resource allocation, something does need to be said on the broader tradeoffs at play

There is no disagreement that as-is, that data type is not suitable for a nice self-contained library. [#faststring-unsuitable]_
The disagreement is whether TTG should be blocked on reworking ``FastString`` somehow to be better for GHC and non-GHC alike, or whether we should just side-step the issue entirely.

We make no claims about what is better in the long term for GHC, but when reworking ``FastString`` and benchmarking the new algorthms might take **Days to Weeks**, we can side-step the issue with a new ``StringP`` type family "extension point" like the existing ``IdP`` one in **minutes**. [#extension-point]_

Out of a basic desire to minimize costs where possible, we thus declare that unless "Plan A" works out almost as quickly, "Plan B" of just introducing another extension point should be used.
We can also revisit getting rid of any newly-added extension points later, *after* we have our factored-out AST library.

N.B. Third-party code (e.g. HLint_ will often also need ``Data`` instances for the AST.
We could consider making those polymorphic again as they used to be, and factoring them out accordingly.
Or, we can just let downstream projects define their own instances specialized do their own extension type (as GHC does with ``GhcPass``).
The latter is a good cheap "plan B" to delay dealing with those instances so they don't block this milestone.

.. [#faststring-unsuitable]
  Everyone agrees it is insuitable in its current state because things like:

  - Global state because of `string interning <https://en.wikipedia.org/wiki/String_interning>`_, with a global variable baked into the RTS no less!

  - Memoizing features for other parts of the compiler unrelated to parsing, such as the `"Z-Encoding" <https://gitlab.haskell.org/ghc/ghc/-/blob/261c4acbfdaf5babfc57ab0cef211edb66153fb1/libraries/ghc-boot/GHC/Utils/Encoding.hs#L43>`_ GHC happens to use for object file symbol `name mangling <https://en.wikipedia.org/wiki/Name_mangling>`.

  Everyone *also* agrees that it is worth revising whether these algorithmic decision still make sense given modern hardware, see `GHC Issue #17259 <https://gitlab.haskell.org/ghc/ghc/-/issues/17259>`_.

.. [#extension-point]
  "Extension point" is Trees That Grow parlance for such a type family.
  The idea is that the AST library no longer refers to a data type like ``FastString`` directory, but instead refers to an abstract ``StringP p``.
  Then, GHC can define ``StringP (GhcPass _) = FastString`` to use it client side, across all compilation passes.
  All term-level code continues to works exactly the same as before without modification.

Conditions
^^^^^^^^^^

It is important to make clear what must *not* happen as a side-effect of this, so that we are careful to avoid extra costs.

- The new AST library must live in the same repo, and not cause and extra Git submodule.
  Synchronizing changes across Git submodules is a drag on on GHC development today, and we must not make that problem worse.

- The new AST library be loadable in the same GHCi session as rest of GHC.
  Having to restart tools to switch between libraries is a major productivity drag in the Haskell ecosystem, and we wouldn't want to impose it on GHC.
  There is existing prior art of loading ``ghc`` and ``ghc-bin`` at the same time, and also recent developments in Cabal that that allow doing this in a less "hacky" manner.

- Build / CI times should not be impacted.
  Since Hadrian build Haskell modules individually, it doesn't much care about library boundaries.
  Redividing the same modules into different libraries thus should have negligible impact on build times.

- Some layer violations are not actually impediments to splitting out an AST library, and thus we should *not* prioritize fixing them with Haskell Foundation funds.

  For example the type family ``GhcNoTc`` doesn't belong in a GHC-agnostic parsing library, as indicated by its name, as it doesn't incur any imports into the rest of GHC from those modules, it doesn't actually impose a problem.
  We can simply include it in the AST library, and non-GHC clients can write instances for it like they do for the proper extension points.

Proof of success: Use by Haddock
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Executor**: Laurent P. René de Cotret (volunteer)

**Time Estimate:** 4 weeks of part-time work

It might seem odd that there is a real-world use case for an AST without a Parser, but we do in fact have one: a `Haskell Foundation Technical Proposal <https://github.com/haskellfoundation/tech-proposals/pull/44>`_ and associated `Summer of Haskell <https://summer.haskell.org/news/2023-05-14-summer-of-haskell-2023-project-selections.html#maximally-decoupling-haddock-and-ghc>`_ project reducing Haddock's depedencies on GHC.
The situation is nicely described by Laurent who is mentoring the project `here <https://gitlab.haskell.org/ghc/ghc/-/issues/21592#note_519447>`_, but we'll recap the basics:

Haddock as a whole is still using the complete ``ghc`` library, and parsing is continuing to happen that way.
Individual rendering backends, however, are being split out into separate packages, and those are only using the ``Language.Haskell.Syntax.*`` modules.

That is all being done by the Summer of Haskell project, and will be finished by Laurent if need be once the project is over.
What is to be done in this step is to make those backend packages just depend on the new AST library.
This should be straightforward since it is precisely those ``Language.Haskell.Syntax.*`` modules that will end up in the AST library.
All code should continue to work as before, since ``ghc`` will also use the new AST library, and thus the parsing initiated by the frontend and the backends should automatically agree on data structures.

Separate the Parser
-------------------

Split library
~~~~~~~~~~~~~

**Executor**: Haskell Foundation

**Time Estimate:** ??

This work is more uncertain, because the parser and post-processing steps necessary to get an actual AST may use utility functions currently entangled with the rest of the compiler.
It maybe be the case that we need to finish the far more certain first step (AST library) to get better clarity on what work remains for the parser, and thus price this step accurately.

.. note::

   We have a couple options on how to deal with the uncertainty here.
   For example:

   - We could either remove this and the HLint integration from the current proposal, saving it for a future proposal.
   - we could accept the whole proposal but make sure we edit this section once the previous two are completed with more information before stored.

Conditions
^^^^^^^^^^

- The same conditions on splitting a library without negatively impacting GHC development are imposed as in the separating the AST step.

Proof of success: Use by HLint
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Executor**: Shayne Fletcher (volunteer)

**Time Estimate:** 8 weeks of part-time work

We will continue the tradition discussed in the background section of using HLint to validate that parsers for Haskell are usable by real-world programs that are not GHC.

The migration from |haskell-src-exts| to |ghc-lib-parser| was quite difficult because those libraries are nothing alike.
In contrast, we expect the migration from |ghc-lib-parser| to the new AST and parser libraries to be quite simple and pleasant, because the two new libraries should be very similar to |ghc-lib-parser|, and where they differ they should be strictly easier to use than before.

Note that HLint does use a few other things behind the AST and Parser that currently make it into |ghc-lib-parser|, but which we might want in our new libraries.

#. It uses GHC's multi-purpose ``Outputable`` instead of some more dedicated exact-printing machinary

#. It uses ``parseDynamicFilePragma``, and thus GHC's infamous ``DynFlags`` to support pragmas like ``{-# LANGUAGE ... #-}`` and ``{-# OPTIONS_GHC ... #-}``.

For the first case, we might consider factoring ``Outputable`` into a separate library too.
Or we can prioritize a more dedicated exact-print solution to use instead of ``Outputable`` (see the future work section).

For the second case, we might have to do something temporary like e.g. continuing to use an auto-extracted library like |ghc-lib-parser|, but depending on our newly factored-output libraries, to get this functionality for HLint_.
But longer term, we refer to the discussion of ``OPTIONS_GHC`` in [modularizing-ghc]_.
The steps advocated there will avoid this problem entirely by restricting ``OPTIONS_GHC`` and giving it a more minimal data structure that is easily to factor out.

Shayne Fetcher volunteers to lead the HLint integration as a core HLint maintainer.

.. [modularizing-ghc]
   https://hsyl20.fr/home/files/papers/2022-ghc-modularity.pdf

Stakeholders
============

*Who stands to gain or lose from the implementation of this proposal? Proposals should identify stakeholders so that they can be contacted for input, and a final decision should not occur without having made a good-faith effort to solicit representative feedback from important stakeholder groups.*

GHC Developers
--------------

The proposal is asking that we change out code in GHC is organized, so it is crucial that we solicit feedback from the broader `GHC Team <https://gitlab.haskell.org/ghc/ghc-hq/-/tree/main#2-the-ghc-team>`_, and the narrow `GHC HQ group <https://gitlab.haskell.org/ghc/ghc-hq/-/tree/main#3-ghc-hq-group>`_ in particular.
It is John's understanding that the GHC developers are broadly supportive of the goal here in the abstract
(after all, SPJ was an author of the Trees That Grow paper),
and also of the approach of tackling the AST and Parser separately.
`GHC Issue #21592 <https://gitlab.haskell.org/ghc/ghc/-/issues/21592>`_ from @alt-romes
contains a very good summary of that initial consensus, including relevant quotes from key people from various previous discussion threads scattered about and potentially hard to find otherwise.

However, some of the specific details needed to get this done in a timely manner may be more controversial.
In particular, introducing more extension points to ensure rapid progress was very controversial before, and in return for putting up with such a thing as stop-gap, the GHC HQ might want something in return, like an additional phase of work to eliminate the new extension points afterwords.

Haddock Developers
------------------

The Haddock maintainers will likewise be maintaining the result of the Summer of Code project, along with the integration work done as part of this.
We should ensure that they are satisfied with the work being done here and it comports with their overall desires for the project.

HLint Developers
----------------

The HLint developers have been heavily involved with reusable AST and parser work every step of the way, and should continue to be involved with this too.
In addition, we've chosen HLint to be the integration step for the second half just like Haddock was in the first.
Thankfully, one of the HLint developers, Shayne Fletcher, is also a co-author of this proposal!

Future Work
===========

Pretty-printing
---------------

Just as it is nice to accompany the AST with logic to convert raw text syntax to it (the parser),
so it is nice to also accompany the AST with logic to do the opposite: render back to text (the pretty-printer).

There has been much work to allow this to be done in a faithful round trip, know as "exact-print" functionality.
However, the detail of how this works are still fast-evolving. [#exact-print-evolving]_

We therefore think it is best to leave factoring out the pretty-printer into a reusable library (either part of the parser library, or a new 3rd reusable library) as a future work.

That said, if ``Outputable`` becomes too much of a hassle for HLint_, as described above, we might prioritize this.

.. [#exact-print-evolving]
   See `GHC Issue #23447 <https://gitlab.haskell.org/ghc/ghc/-/issues/23447>`_ for example.
