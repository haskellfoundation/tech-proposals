.. sectnum::

Module naming conventions for GHC base libraries
==================================================

Motivation
----------
The accepted `Proposal #51: GHC base libraries <https://github.com/haskellfoundation/tech-proposals/blob/main/proposals/accepted/051-ghc-base-libraries.rst>`_
defines the following libraries:

* ``base``: the foundational library on which the rest of the ecosystem is based.  Is API is carefully curated by the `Core Libraries Committee <https://github.com/haskell/core-libraries-committee>`_, and is kept rather stable.

* ``ghc-experimental``: the home of experimental extensions to GHC, usually ones proposed by the
   `GHC Steering Committee <https://github.com/ghc-proposals/ghc-proposals/>`_.
  Functions and types in here are usually candidates for later transfer into ``base``.  It is user-facing (user are encouraged to depend on it), but its API is less stable than ``base``.

* ``ghc-prim, ghc-internals`` (and perhaps others): define functions and data types used internally by GHC to support the API of ``base`` and ``ghc-experimental``.
  These libraries come with no stability guarantees: they may change at short notice.  (They do, however, follow the PVP.)

The question arises of what module names should be used. For example, suppose that all three exposed a module called ``Data.Tuple``.  In principle that would be fine -- GHC allows you
to use the package name in the ``import`` statement, to disambiguate.  But it's *extremely* confusing.  This proposal articulates a set of conventions to
help us design module names.

The proposal
============

Proposal 1
-----------

* Modules in ``base``, ``ghc-experimental``, ``ghc-prim``, ``ghc-internals`` etc should all have distinct names.

That principle leads immediately to the question: what should those names be?  Hence proposal 2.

Proposal 2
-----------

* Modules in GHC's internal libraries (``ghc-prim``, ``ghc-internals`` etc) should be of form ``GHC.*``.
* Modules in ``ghc-experimental`` should be of form ``Experimental.*``.
* Modules in ``base`` should not have either of these prefixes.

So example we might have
* ``GHC.Tuple`` in ``ghc-internals``,
* ``Experimental.Tuple`` or ``Experimental.Data.Tuple`` in ``ghc-experimental``
* ``Data.Tuple`` in ``base``

Proposal 3
-----------

The current ``base`` API exposes many modules starting with ``GHC.*``, so the proposed conventions could only
apply to *new* modules.

* Over time, and with the agreement and support of the Core Libraries Committee, we should seek to remove ``GHC.*`` modules
  from ``base``, either exposing their desired API through a stable, CLC-curated, module in ``base``; or removing it altogether.  Of course
  there would be a significant deprecation cycle, to allow client libraries to adapt.

Alternatives
==============
* We could use ``GHC.*`` for modules in ``ghc-experimental``, and maybe ``GHC.Internals.*`` for module in ``ghc-internals``.  But

  * There are two sorts of GHC-specific-ness to consider:
    * Modules that are part of GHC's implementations
    * Modules that support a GHC extension, blessed by the GHC Steering Committee

    It is worth distinguishing these: it's confusing if both start with ``GHC.``.

  * It would be a huge upheaval (with impact on users) to rename hundreds of modules in ``ghc-internals``.

* We could use a suffix ``*.Internals`` or ``*.Experimental`` instead of a prefix.  But
  * This sort of naming is conventionally used to distinguish modules *within* a package, not *between* packages.
  * It would still suffer from the cost of renaming hundreds of modules in ``ghc-internals``

Discussion
============
What should we do about the ``ghc`` package, which exposes GHC as a library, through the GHC API?
It wouldn't really make sense to call it ``Experimental.*``.  And yet, under the above proposals, ``GHC.*`` connotes
"internal, unstable" which should not be true of the GHC API (although it is today).

Perhaps, as part of the GHC API redesign (a HF project in its own right) we can define modules with
stable APIs, and a new prefix, such as ``GhcAPI.*``?


