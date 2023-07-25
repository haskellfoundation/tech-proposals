.. sectnum::

**Module naming conventions for GHC base libraries**

Background and motivation
===========================
The accepted `Proposal #51: GHC base libraries <https://github.com/haskellfoundation/tech-proposals/blob/main/proposals/accepted/051-ghc-base-libraries.rst>`_
defines the following libraries:

* ``base``: the foundational library on which the rest of the ecosystem is based.  Is API is carefully curated by the `Core Libraries Committee <https://github.com/haskell/core-libraries-committee>`_, and is kept rather stable.

* ``ghc-experimental``: the home of experimental extensions to GHC, usually ones proposed by the
  `GHC Steering Committee <https://github.com/ghc-proposals/ghc-proposals/>`_.
  * Functions and types in here are usually candidates for later transfer into ``base``.  But not necessarily: if a collection of functions is not adopted widely enough, it may not be proposed for a move to `base`.
  * It is user-facing (user are encouraged to depend on it), but its API is less stable than ``base``.

* ``ghc-prim, ghc-internals`` (and perhaps others): define functions and data types used internally by GHC to support the API of ``base`` and ``ghc-experimental``.
  * These libraries come with no stability guarantees: they may change at short notice.

In addition we already have:

* ``ghc``: this library exposes GHC as a library, through the (currently ill-defined) GHC API.

All these libraries follow the Haskell Package Versioning Policy (PVP).

The question arises of *what module names should be used*. For example, suppose that all three exposed a module called ``Data.Tuple``.  In principle that would be fine -- GHC allows you
to use the package name in the ``import`` statement, to disambiguate.  But it's *extremely* confusing.  This proposal articulates a set of conventions to
help us design module names.

The proposal
============

This proposal is split into four for easier discussion.  Each sub-proposal builds on the
earlier ones -- they are increments, not alternatives.

The goals of this proposal are deliberately limited to establish naming conventions.  We propose no new mechanisms.

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

* Over time, and with the agreement and support of the Core Libraries Committee, we may remove some ``GHC.*`` modules
  from ``base``, especially ones that are barely used, or are manifestly "internal" (i.e. part of the implementation
  of other, more public functions.
  Of course there would be a significant deprecation cycle, to allow client libraries to adapt.

Proposal 3 only expresses a direction of travel.  We will have to see what the CLC's attitude is,
and what the Haskell community thinks.  Anything that disturbs the API of base needs to be considered
rather carefully.


Proposal 4
------------

* The public API of package ``ghc`` (GHC as a library) should have modules of form ``GhcAPI.*``.

All of the modules in package ``ghc`` currently start with ``GHC.*`` which correctly signals that they are part of GHC's internals.
As part of the GHC API redesign (a HF project in its own right, currently stalled) it would be very helpfult
to modules with stable APIs, and a new prefix, such as ``GhcAPI.*``.


Timescale
==========
The first release of GHC with `ghc-experimental` and `ghc-internals` will be GHC 9.10, which expect to
release in early 2024.  It would be good to establish naming conventions for modules well before this date.

Example lifecycle
===================

By way of example, consider the ``HasField`` class, which supports overloaded record fields.
It is currently defined in ``base:GHC.Records``, which is an odd module to have to import.
Moreover there is
more than one GHC proposal that suggest changes to its design (e.g. see `GHC Proposal 158 <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0158-record-set-field.rst>`_); it is not nearly as stable as most of ``base``

If ``ghc-experimental`` had existed we would have put it in ``ghc-experimental:Experimental.Records``.
That would have made it clear that the design of overloaded records still evolving.
Once the design becomes settled and stable, it could move to ``base``, perhaps in a module like ``Data.Records``.

Other similar examples include
* The tuple proposal of `GHC Proposal 475 <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0475-tuple-syntax.rst>`_
* The `DataToTag CLC proposal <https://github.com/haskell/core-libraries-committee/issues/104>`_ would have been easier to expose through ``ghc-experimental`` in the first instance.

Alternatives
==============
* We could dispute Proposal 1: one could imagine deliberately naming modules in ``ghc-experimental`` with the
  same module name as their eventual expected (by someone) home in ``base``.  The goal would be to reduce impact if and when
  the module moves from ``ghc-experimental`` to ``base``. For example, we might add ``Data.Tuple`` to ``ghc-experimental`` containing the new type constructors ``Tuple2``, ``Tuple3`` etc that are proposed in `GHC Proposal 475 <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0475-tuple-syntax.rst>`_.   However:

  * In the meantime there are two modules both called ``Data.Tuple``.  This is bad.  Which one does ``import Data.Tuple`` import?  (Look at the Cabal file, perhaps?)  How can I import both?  (Package-qualified imports perhaps.) So it will really only help in the case of a brand-new module, not already in ``base``.
  * It loses the explicit cue given by ``import Experimental.Data.Tuple``.

* We could use ``GHC.*`` for modules in ``ghc-experimental``, and maybe ``GHC.Internals.*`` for module in ``ghc-internals``.  But

  * There are two sorts of GHC-specific-ness to consider:
    * Modules that are part of GHC's implementations
    * Modules that support a GHC extension, blessed by the GHC Steering Committee

    It is worth distinguishing these: it's confusing if both start with ``GHC.``.

  * It would be a huge upheaval (with impact on users) to rename hundreds of modules in ``ghc-internals``.

* We could use ``GHC.Experimental.*`` for modules in ``ghc-experimental``.  But that seems a bit backwards: ``GHC.Tuple`` (in ``ghc-internals``) would look more stable (less experimental) than ``GHC.Experimental.Tuple`` in ``ghc-experimental``; but the reverse is the case.

* We could use a suffix ``*.Internals`` or ``*.Experimental`` instead of a prefix.  But
  * This sort of naming is conventionally used to distinguish modules *within* a package, not *between* packages.
  * It would still suffer from the cost of renaming hundreds of modules in ``ghc-internals``


