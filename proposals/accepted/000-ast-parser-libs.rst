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

``haskell-src-exts``
--------------------

An older attempt is the venerable `haskell-src-exts <https://hackage.haskell.org/package/haskell-src-exts>`_ library.
This is actually was part of a larger project called the `"Haskell Suite" <https://github.com/haskell-suite>`_, the purpose of which was "to implement the whole Haskell compiler as a set of libraries".
However, the whole compiler doesn't appear to exist, and the project as a whole ceased development .
``haskell-src-exts`` lasted longer, but had great trouble keeping up with GHC, and is now also unmaintained since 2020.

The lessons are clear:
``haskell-src-exts`` succeeded in being modular and self-contained, failed in trying to keep up with GHC.
Given the size of the community, competing head-on with GHC or trying to keep up with it is very difficult.
Being used by GHC, so keeping up happens automatically, is the clearest way to avoid this problem.

``ghc-lib-parser``
------------------

A newer attempt is `ghc-lib-parser <https://hackage.haskell.org/package/ghc-lib-parser>`_ library.
Prior users of ``haskell-src-exts`` `like HLint <https://github.com/ndmitchell/hlint/issues/645>`_ have largely migrated from ``haskell-src-exts`` to ``ghc-lib-parser``.
But ``ghc-lib-parser`` is *not* actually a separately developed library; rather it is one that is extracted from GHC's own source.
This ensures it is up to date with the latest behavior.
But this extraction process is complex, and results in a far larger library than is desired: see the *hundreds* of modules included inside it, many of which have nothing to do with the Haskell surface language.
All this `"bycatch" <https://en.wikipedia.org/wiki/Bycatch>`_ in the extraction process results in a library that daunting to use, and which has a harder time presenting any sort of stable interface.

The developers of ``ghc-lib-parser`` would not dispute the above criticisms, for this state of affairs was never intended to be a permanent solution, but rather just a stop gap.
Whereas ``ghc-lib-parser`` succeeds in keeping up with GHC because it *is* GHC, it fails in being self-contained because modularity cannot be `"fixed in post" [production] <https://tvtropes.org/pmwiki/pmwiki.php/Main/FixItInPost>`_.
Code that is intended to be separate from any one consumer must be developed with those boundaries enforced during development.

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

The project is split into two separate steps: separating the parser, and separating the AST.
Each step has a method, time estimate, and (most importantly) clear success criteria, including use by downstream projects to ensure value is delivered.
The intent is thus that they are self-contained, and can be individually funded.

Separate the Parser
-------------------

Split library
~~~~~~~~~~~~~

**Time Estimate:** ??

Proof of success: Use by Haddock
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Time Estimate:** ??

Separate the AST
----------------

Split library
~~~~~~~~~~~~~

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
