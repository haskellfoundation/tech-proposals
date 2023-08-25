.. sectnum::

**Module naming conventions for GHC base libraries**

Background and motivation
===========================
The accepted `Proposal #51: GHC base libraries <https://github.com/haskellfoundation/tech-proposals/blob/main/proposals/accepted/051-ghc-base-libraries.rst>`_
defines the following libraries:

* ``base``:

  * The foundational library on which the rest of the Haskell ecosystem is based.
  * Its API is carefully curated by the `Core Libraries Committee <https://github.com/haskell/core-libraries-committee>`_, and is kept rather stable.
  * Depends on ``ghc-internal`` (and ``ghc-prim`` etc), but *not* on ``ghc-experimental``.
  * Major version bumps are at the Core Libraries Committee's discretion.

* ``ghc-experimental``:

  * The home of experimental extensions to GHC, usually ones proposed by the
    `GHC Steering Committee <https://github.com/ghc-proposals/ghc-proposals/>`_.

  * Functions and types in here are usually candidates for later transfer into ``base``.  But not necessarily: if a collection of functions is not adopted widely enough, it may not be proposed for a move to `base`.  Or it could move to another library entirely.

  * It is user-facing (user are encouraged to depend on it), but its API is less stable than ``base``.

  * Depends on ``base``.

  * Likely to have a major version bump with each GHC release.

* ``ghc-prim, ghc-internal`` (and perhaps others; it's an internal GHC implementation decision):

  * Define functions and data types used internally by GHC to support the API of ``base`` and ``ghc-experimental``.

  * These libraries come with no stability guarantees: they may change at short notice.
  * Certain to have a major version bump with every GHC release.

In addition we already have:

* ``ghc``: this library exposes GHC as a library, through the (currently ill-defined) GHC API.

All these libraries follow the Haskell Package Versioning Policy (PVP).  The reader is encouraged
to consult `Proposal #51: GHC base libraries <https://github.com/haskellfoundation/tech-proposals/blob/main/proposals/accepted/051-ghc-base-libraries.rst>`_ for more background and rationale for this library structure.

The question arises of *what module names should be used*. For example, suppose that all three exposed a module called ``Data.Tuple``.  In principle that would be fine -- GHC allows you
to use the package name in the ``import`` statement, to disambiguate.  But it's *extremely* confusing.  This proposal articulates a set of conventions to
help us design module names.

The proposal
============

This proposal is split into four sub-proposals for easier discussion.  Each sub-proposal builds on the
earlier ones -- they are increments, not alternatives.

The goals of this proposal are deliberately limited to establish naming conventions.  We do not propose
any changes to ``ghc`` or to ``cabal``.

Proposal 1
-----------

* Modules in ``base``, ``ghc-experimental``, ``ghc-prim``, ``ghc-internal`` etc should all have distinct names.

That principle leads immediately to the question: what should those names be?  Hence proposal 2.

Proposal 2
-----------

* Modules in GHC's internal libraries (``ghc-prim``, ``ghc-internal`` etc) should be of form ``GHC.Internal*``.
* Modules in ``ghc-experimental`` should be of form ``*.Experimental``.
* Modules in ``base`` should not have either of these forms.

So example we might have

* ``GHC.Internal.Bits`` in ``ghc-internal``,
* ``Data.Bits.Experimental`` in ``ghc-experimental``
* ``Data.Bits``, and currently also ``GHC.Bits``, in ``base``

Why ``GHC.Internal.*`` for modules in ``ghc-internal``?  Would ``GHC.*`` not be enough? Here's why:

* ``base`` already has ``GHC.Bits``, and Proposal 1 stops us re-using the same module name in ``ghc-internal``.
  If we were starting from a blank sheet of paper we might have no ``GHC.*`` modules in ``base``, but there
  curently 138 such modules and it seems unlikely that we will ever remove all, or even most, of them from
  ``base``.

* The prefix ``GHC.Internal`` serves as an additional clue to the importing module that this API is not stable.

* Since ``ghc-internal`` is brand new, we can name its modules however we like.  However, ``ghc-prim`` exists
  already and we may have to live with modules like ``GHC.CString`` in ``ghc-prim`` for a while.  Perhaps
  we make the switch slowly over time, by introducing ``GHC.Internal.CString`` and deprecating ``GHC.CString``.

Note that among the GHC implementation packages (``ghc-prim``, ``ghc-internal``, ``ghc-bignum`` etc) there
is no expectation that the module name signals which package the module is in. It's just an internal
implementation matter.

Using a prefix for ``ghc-internal`` and a suffix for ``ghc-experimental`` may seem inconsistent,
but it was a clear consensus from the discussion about the proposal:

* ``Data.Tuple.Experimental``, for example, is an companion/extension of ``Data.Tuple``; some exports may move from one to the other. Many developers sort their imports alphabetically. Making this a suffix means all ``Data.Tuple``-related imports are next to each other.  For example, one might prefer this::

    import Control.Applicative
    import Control.Applicative.Experimental
    import Control.Arrow
    import Data.Tuple
    import Foreign.C
    import Foreign.C.Experimental

  to this::

    import Control.Applicative
    import Control.Arrow
    import Experimental.Control.Applicative
    import Experimental.Foreign.C
    import Data.Tuple
    import Foreign.C

  This pattern, of a module in ``ghc-experimental`` that is closely related to one in ``base`` seems likely to be common.

* On the other hand, GHC-internal modules are often unrelated to the naming
  scheme of ``base``.  Here a prefix feels more appropriate.  Moreover using a
  prefix aligns with current practice: the ``GHC.*`` convention is extensively
  used in the GHC-internal modules currently in ``base``, and ``ghc-prim``, as
  well as the modules that implement GHC itself.

Proposal 3
-----------

The current ``base`` API exposes many modules starting with ``GHC.*``, so the proposed conventions could only
apply to *new* modules.

* Over time, and only with the agreement and support of the Core Libraries Committee, we may remove some ``GHC.*`` modules
  from ``base``, especially ones that are barely used, or are manifestly "internal" (i.e. part of the implementation
  of other, more public functions).
  Of course there would be a significant deprecation cycle, to allow client libraries to adapt.

Proposal 3 only expresses a direction of travel.  We will have to see what the CLC's attitude is,
and what the Haskell community thinks.  Anything that disturbs the API of base needs to be considered
rather carefully.


Proposal 4
------------

All of the modules in package ``ghc`` currently start with ``GHC.*`` which
(currently correctly) signals that they are part of GHC's internals.

As part of the GHC API redesign (a HF project in its own right, currently stalled) it would be very helpful
to identify a (multi-module) stable API for package ``ghc``. In that way, users of package ``ghc``
could know whether
they are using a curated, relatively-stable API function, or reaching deep into GHC's guts and using
a random fuction whose name or type, or very existence, might change without warning. Hence:

* The public API of package ``ghc`` (GHC as a library) should have modules whose names clearly distinguish them
  from internal modules.

For example, the public API could have modules of form ``GhcAPI.*``, or ``GHC.API.*``, or ``Language.Haskell.GHC.*`` or something else. The specifics are a matter for the future GHC API working group.



Timescale
==========
The first release of GHC with ``ghc-experimental`` and ``ghc-internal`` will be GHC 9.10, which expect to
release in early 2024.  It would be good to establish naming conventions for modules well before this date.

Example lifecycle
===================

By way of example, consider the ``HasField`` class, which supports overloaded record fields.
It is currently defined in ``base:GHC.Records``, which is an odd module to have to import.
Moreover there is
more than one GHC proposal that suggest changes to its design (e.g. see `GHC Proposal 158 <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0158-record-set-field.rst>`_); it is not nearly as stable as most of ``base``

If ``ghc-experimental`` had existed we would have put it in ``ghc-experimental:Data.Records.Experimental``.
That would have made it clear that the design of overloaded records still evolving.
Once the design becomes settled and stable, it could move to ``base``, perhaps in a module like ``Data.Records``.

Other similar examples include

* The tuple proposal of `GHC Proposal 475 <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0475-tuple-syntax.rst>`_
* `GHC Proposal 330 (Decorate exceptions with backtrace information) <https://github.com/bgamari/ghc-proposals/blob/stacktraces/proposals/0000-exception-backtraces.rst>`_ proposes significant new additions to the API of exceptions.

Alternatives
==============
* We could dispute Proposal 1: one could imagine deliberately naming modules in ``ghc-experimental`` with the
  same module name as their eventual expected (by someone) home in ``base``.  The goal would be to reduce impact if and when
  the module moves from ``ghc-experimental`` to ``base``. For example, we might add ``Data.Tuple`` to ``ghc-experimental`` containing the new type constructors ``Tuple2``, ``Tuple3`` etc that are proposed in `GHC Proposal 475 <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0475-tuple-syntax.rst>`_.   However:

  * In the meantime there are two modules both called ``Data.Tuple``.  This is bad.  Which one does ``import Data.Tuple`` import?  (Look at the Cabal file, perhaps?)  How can I import both?  (Package-qualified imports perhaps.) So it will really only help in the case of a brand-new module, not already in ``base``.
  * It loses the explicit cue, in the source code, given by ``import Experimental.Data.Tuple``.

