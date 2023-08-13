===========================================
Split out AST and Parser libraries from GHC
===========================================

:Date: August 2023
:Authors:
  John Ericson,
  ???

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

|haskell-src-exts|
------------------

An older attempt is the venerable "Haskell-Source with Extensions (HSE)" package, also known as |haskell-src-exts|_ .
This is actually part of a larger project called the `"Haskell Suite" <https://github.com/haskell-suite>`_, the purpose of which was "to implement the whole Haskell compiler as a set of libraries".
However, the whole compiler doesn't appear to exist, and the project as a whole ceased development .

`HLint <https://hackage.haskell.org/package/hlint>`_, the Haskell linter project developed continuously since 2008 relied heavily on |haskell-src-exts| which lasted longer than the rest of suite, but had great trouble keeping up with GHC.
In Februrary 2019, |ghc-lib-parser| `was released <http://neilmitchell.blogspot.com/2019/02/announcing-ghc-lib.html>`_ and in May 2019 an `announcement <https://mail.haskell.org/pipermail/haskell-cafe/2019-May/131166.html>`_ that there would be no further |haskell-src-exts| releases followed and the advice given for anyone wishing to parse Haskell programs to "use the GHC API, specifically you can use |ghc-lib-parser|".

The lessons are clear:

- |haskell-src-exts| succeeded in being modular and self-contained but failed trying to keep up with GHC;

- given the size of the community, competing head-on with GHC or trying to keep up with it is very difficult:

  - being used by GHC, so keeping up happens automatically, is the clearest way to avoid this problem.

|ghc-lib-parser|
------------------

In June 2019, `HLint began transitition <http://neilmitchell.blogspot.com/2019/06/hlints-path-to-ghc-parser.html>`_ to |ghc-lib-parser| and in May 2020, the release of HLint-3.0 which "uses the GHC parser" `was announced <http://neilmitchell.blogspot.com/2020/05/hlint-30.html>`_.
Today, most users of |haskell-src-exts| have largely migrated from |haskell-src-exts| to |ghc-lib-parser| [#exampleghclibparserusers]_.

A |ghc-lib-parser|_ package contains GHC compiler sources packaged as a library [#ghcinception]_.
This ensures it is up to date with the latest behavior but this extraction process is complex, requires constant patching to keep pace with GHC evolution, and results in a far larger library than is desired:
see the *hundreds* of modules included inside it, many of which have nothing to do with the Haskell surface language.
All this `"bycatch" <https://en.wikipedia.org/wiki/Bycatch>`_ in the extraction process results in a library that daunting to use, and which has a hard time presenting a stable interface.

Whereas |ghc-lib-parser| succeeds in keeping up with GHC because it *is* GHC, it fails in being self-contained because modularity cannot be `"fixed in post" [production] <https://tvtropes.org/pmwiki/pmwiki.php/Main/FixItInPost>`_.
Code that is intended to be separate from any one consumer must be developed with those boundaries enforced during development.

.. [#ghcinception] The extraction process was enabled by insights gained from the `"GHCinception" <https://mgsloan.com/posts/ghcinception/>`_ or "GHC in GHCi" initative.

.. [#exampleghclibparserusers] Today for example, notable users include `ormolu <https://hackage.haskell.org/package/ormolu>`_, `ghcide <https://hackage.haskell.org/package/ghcide>`_, `hls-hlint-plugin <https://hackage.haskell.org/package/hls-hlint-plugin>`_, `hindent <https://hackage.haskell.org/package/hindent>`_ & `stylish-haskell <https://hackage.haskell.org/package/stylish-haskell>`_.

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

**Time Estimate:** 1--2 Weeks

The first step is just separating data definitions.
We don't need to worry about code entangling, just data entangling.

The timeline for this is pretty short because there exists an easy last-resort way to decouple anything:
just add another TTG type family.

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
