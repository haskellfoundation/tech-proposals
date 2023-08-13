===========================================
Split out AST and Parser libraries from GHC
===========================================

:Date: August 2023
:Authors:
  John Ericson,
  Shayne Fletcher

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

However, no library has so far done both, to meet both criteria.

Solution
--------

The purpose of this proposal is to make that library finally exist.
The Haskell Foundation will finance the completion of the existing "Trees that grow" project, decoupling GHC's AST and parser from the rest of the compiler so they can be moved to separate libraries.
Those libaries will be "normal" haskell libraries, without any weird dependencies or build process, and published on Hackage.
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
------------------

In Februrary 2019, |ghc-lib-parser| `was released <http://neilmitchell.blogspot.com/2019/02/announcing-ghc-lib.html>`_ and in May 2019 an `announcement <https://mail.haskell.org/pipermail/haskell-cafe/2019-May/131166.html>`_ that there would be no further |haskell-src-exts| releases followed and the advice given for anyone wishing to parse Haskell programs to "use the GHC API, specifically you can use |ghc-lib-parser|".

A |ghc-lib-parser|_ package contains GHC compiler sources packaged as a library [#ghcinception]_.
This ensures it is up to date with the latest behavior but this extraction process is complex, requires constant patching to keep pace with GHC evolution, and results in a far larger library than is desired:
see the *hundreds* of modules included inside it, many of which have nothing to do with the Haskell surface language.
All this `"bycatch" <https://en.wikipedia.org/wiki/Bycatch>`_ in the extraction process results in a library that daunting to use, and which has a hard time presenting a stable interface.

The lesson from |haskell-src-exts| are clear:

- it succeeded in being modular and self-contained but failed trying to keep up with GHC;

- given the size of the community, competing head-on with GHC or trying to keep up with it is very difficult:

  - being used by GHC, so keeping up happens automatically, is the clearest way to avoid this problem.

Whereas |ghc-lib-parser| succeeds in keeping up with GHC because it *is* GHC, it fails in being self-contained because modularity cannot be `"fixed in post" [production] <https://tvtropes.org/pmwiki/pmwiki.php/Main/FixItInPost>`_.
Code that is intended to be separate from any one consumer must be developed with those boundaries enforced during development.

.. [#ghcinception] The extraction process was enabled by insights gained from the `"GHCinception" <https://mgsloan.com/posts/ghcinception/>`_ or "GHC in GHCi" initative.

Downstream projects
-------------------

In addition to going over the major AST/parser libraries for Haskell, it is also useful to talk about the most notable projects that use them.
Ultimately, it is those projects we want to help out.

HLint_, the Haskell linter project developed continuously since 2008 is the most notable one, not just because its longstanding and wide use, but also because many of the developers that worked on the previous two libraries also worked on it --- use by HLint served as proof the libraries were fit-for-purpose.

In June 2019,
following the release of |ghc-lib-parser| and deprecation of |haskell-src-exts| earlier that year as described above,
`HLint began the transitition <http://neilmitchell.blogspot.com/2019/06/hlints-path-to-ghc-parser.html>`_ to |ghc-lib-parser|.
In May 2020, the release of HLint-3.0 which "uses the GHC parser" `was announced <http://neilmitchell.blogspot.com/2020/05/hlint-30.html>`_.

Today, most users of |haskell-src-exts| have largely migrated from |haskell-src-exts| to |ghc-lib-parser| [#exampleghclibparserusers]_.
But just because all these projects are using |ghc-lib-parser| doesn't mean everything is well.
**Insert quote about maintainence overhead.**
The cost of detail with changes to the AST is inevitable --- supporting new language features will inevitably cost developer time.
But all the other busywork of re-extracting the code, etc., is entirely avoidable, *not* inherent to the task at hand.

It is the opinion of the authors of this proposal that should an independent AST parser libraries be maintained upstream with GHC, the costs saved for downstream developers should _greatly_ exceed any costs incurred by GHC developers.
The goal is thus *not* to simply shift a burden from one group of community members to another, but create a positive-sum outcome where there is far less busywork and more flourishing tooling than before.

.. [#exampleghclibparserusers] Today for example, notable users include HLint_, `ormolu <https://hackage.haskell.org/package/ormolu>`_, `ghcide <https://hackage.haskell.org/package/ghcide>`_, `hls-hlint-plugin <https://hackage.haskell.org/package/hls-hlint-plugin>`_, `hindent <https://hackage.haskell.org/package/hindent>`_ & `stylish-haskell <https://hackage.haskell.org/package/stylish-haskell>`_.

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
I have separated out the modules defining the AST under `Language.Haskell.Syntax.*` we wish to split out, and we have tests to track progress reducing their deps, and the parser's deps.
But progress is unsteady and unpredictable.

The basic problem is that the benefits don't actually kick in until the deps are *all* gone, and the code is actually separated out.
Partial progress isn't really directly useful to anyone, and these counters just scoreboard by which we hope to get closer to the end goal.
It is thus hard to do this work with volunteers only, because it is emphatically *not* `"itch scratching" <https://en.wikipedia.org/wiki/The_Cathedral_and_the_Bazaar>`_ work where incremental progress leads immediate incremental benefits to the contributor.

The Haskell Foundation's support in getting this "over the finish line", at which point the community *will* benefit, and benefit greatly, is thus a crucial way we can surmount the coordination failure the lack of incremental payoff causes.

.. [#intra]
  It might sound like the goal is only different usages within GHC, but remember that ``template-haskell`` is a separate library used by users of Haskell not just developers of Haskell.
  A goal of at least some usage outside GHC was always there.

Roadmap
=======

*This section should describe the work that is being proposed to the community for comment, including both technical aspects (choices of system architecture, integration with existing tools and workflows) and community governance (how the developed project will be administered, maintained, and otherwise cared for in the future).
It should also describe the benefits, drawbacks, and risks that are associated with these decisions.
It can be a good idea to describe alternative approaches here as well, and why the proposer prefers the current approach.*

*Are there any deadlines that the HF needs to be aware of?*

*How much money is needed to accomplish the goal? How will it be used?*

The project is split into two separate steps: separating the AST, and separating the parser.
Each step has a method, time estimate, and (most importantly) clear success criteria, including use by downstream projects to ensure value is delivered.
The intent is thus that they are self-contained, and can be individually funded.


Separate the AST
----------------

Split library
~~~~~~~~~~~~~

**Time Estimate:** 1 -- 2 Weeks

The first step is just separating data definitions.
We don't need to worry about code entangling, just data entangling.

The timeline for this is pretty short because there exists an easy last-resort way to decouple anything:
just add another TTG type family.
This came up with some acrimony in `GHC Issue #21628 <https://gitlab.haskell.org/ghc/ghc/-/issues/21628>`_, discussing whether it was better to try to change GHC's ``FastString`` or abstract over it.
The purpose of this proposal isn't to relitigate that issue, but because this proposal *is* about resource allocation, something does need to be said on the broader tradeoffs at play

There is no disagreement that as-is, that data type is not suitable for a nice self-contained library. [#faststring-unsuitable]_
The disagreement is whether TTG should be blocked on reworking ``FastString`` somehow to be better for GHC and non-GHC alike, or whether we should just side-step the issue entirely.

I make no claims about what is better in the long term for GHC, but when reworking ``FastString`` and benchmarking the new algorthms might take **Days to Weeks**, we can side-step the issue with a new ``StringP`` type family "extension point" like the existing ``IdP`` one in **minutes**. [#extension-point]_

Out of a basic fiduciary towards the ``Haskell Foundation``, we thus declare that unless "Plan A" works out very quickly, "Plan B" of just introducing another extension point should be used.
We can also revisit the issue later, *after* we have our factored-out AST library.

.. [#faststring-unsuitable]
  Everyone agrees it is insuitable in its current state because things like:

  - Global state because of `string interning <https://en.wikipedia.org/wiki/String_interning>`, with a global variable baked into the RTS no less!

  - Memoizing features for other parts of the compiler unrelated to parsing, such as the `"Z-Encoding" <https://gitlab.haskell.org/ghc/ghc/-/blob/261c4acbfdaf5babfc57ab0cef211edb66153fb1/libraries/ghc-boot/GHC/Utils/Encoding.hs#L43>` GHC happens to use for object file symbol `name mangling <https://en.wikipedia.org/wiki/Name_mangling>`.

  Everyone *also* agrees that it is worth revising whether these algorithmic decision still make sense given modern hardware, see `GHC Issue #17259 <https://gitlab.haskell.org/ghc/ghc/-/issues/17259>`_.

.. [#extension-point]
  "Extension point" is Trees That Grow parlance for such a type family.
  The idea is that the AST library no longer refers to a data type like ``FastString`` directory, but instead refers to an abstract ``StringP p``.
  Then, GHC can define `StringP (GhcPass _) = FastString`` to use it client side, across all compilation passes.
  All term-level code continues to works exactly the same as before without modification.

Proof of success: Use by Haddock
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Time Estimate:** ??

https://gitlab.haskell.org/ghc/ghc/-/issues/21592#note_519447 Note how this use-case only needs the AST not parser.

Separate the Parser
-------------------

**Time Estimate:** ??

Proof of success: Use by HLint
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Time Estimate:** ??

Stakeholders
============

*Who stands to gain or lose from the implementation of this proposal? Proposals should identify stakeholders so that they can be contacted for input, and a final decision should not occur without having made a good-faith effort to solicit representative feedback from important stakeholder groups.*

- GHC Developers

- Haddock Developers

- HLint Developers

Future Work
===========

Factored out pretty print (exact print)

Depends on resolution of things like https://gitlab.haskell.org/ghc/ghc/-/issues/23447
