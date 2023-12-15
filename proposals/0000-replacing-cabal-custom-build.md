# RFC: Replacing the Cabal Custom build-type

Adam Gundry, Matthew Pickering, Sam Derbyshire, Rodrigo Mesquita, Duncan Coutts (Well-Typed LLP)

- [Abstract](#abstract)
- [Background](#background)
  * [The current interface](#the-current-interface)
  * [Why do packages use the `Custom` build-type?](#why-do-packages-use-the--custom--build-type-)
- [Problem Statement](#problem-statement)
  * [How can we move away from the `Custom` build-type?](#how-can-we-move-away-from-the--custom--build-type-)
  * [Requirements](#requirements)
    + [Integration with existing build systems](#integration-with-existing-build-systems)
  * [Non-requirements](#non-requirements)
- [Prior art and related efforts](#prior-art-and-related-efforts)
  * [Issues with `UserHooks`](#issues-with--userhooks-)
  * [`code-generators`](#-code-generators-)
- [High-level design of `build-type: Hooks`](#high-level-design-of--build-type--hooks-)
  * [`Hooks` from the package author's perspective](#-hooks--from-the-package-author-s-perspective)
  * [`Hooks` from the build tool's perspective](#-hooks--from-the-build-tool-s-perspective)
  * [Designing for future compatibility](#designing-for-future-compatibility)
  * [Library API and versioning](#library-api-and-versioning)
- [Detailed design of `SetupHooks`](#detailed-design-of--setuphooks-)
  * [Phases](#phases)
  * [Cabal configuration type hierarchy](#cabal-configuration-type-hierarchy)
  * [Configuring and building](#configuring-and-building)
  * [Configure hooks](#configure-hooks)
    + [Phase separation](#phase-separation)
    + [`LocalBuildConfig`](#-localbuildconfig-)
    + [`ComponentDiff`](#-componentdiff-)
  * [Build hooks](#build-hooks)
    + [Post-build hooks](#post-build-hooks)
  * [Install hooks](#install-hooks)
- [Pre-build hooks](#pre-build-hooks)
  * [Motivation: fine-grained rules](#motivation--fine-grained-rules)
  * [Motivation: a simplistic first design](#motivation--a-simplistic-first-design)
  * [Proposed design of rules](#proposed-design-of-rules)
  * [Dependency structure](#dependency-structure)
  * [Rule demand](#rule-demand)
  * [Identifiers](#identifiers)
  * [Inputs to pre-build rules](#inputs-to-pre-build-rules)
  * [Hooked preprocessors](#hooked-preprocessors)
- [Examples](#examples)
    + [Generating modules](#generating-modules)
    + [`./configure` style checks](#--configure--style-checks)
    + [Doctests](#doctests)
    + [Hooked programs](#hooked-programs)
    + [Hooked preprocessors](#hooked-preprocessors-1)
    + [executable-hash](#executable-hash)
    + [Composing `SetupHooks`](#composing--setuphooks-)
- [Alternatives](#alternatives)
  * [Decoupling `Cabal-hooks`](#decoupling--cabal-hooks-)
  * [Effects available to hooks](#effects-available-to-hooks)
  * [Inputs and outputs available to hooks](#inputs-and-outputs-available-to-hooks)
  * [Identifiers for fine-grained rules](#identifiers-for-fine-grained-rules)
  * [No searching in fine-grained rules](#no-searching-in-fine-grained-rules)
  * [Making other hooks fine-grained](#making-other-hooks-fine-grained)
- [Stakeholders](#stakeholders)
- [Success](#success)
  * [Testing and migration](#testing-and-migration)
  * [Future work](#future-work)

## Abstract

Every Cabal package can supply its own build system, in the form of a `Setup.hs`
program which implements a common command line interface.
Unfortunately, this makes it difficult to implement features in Cabal which
require the build tool to have complete control over the build system.
This turned out to be a major architectural design flaw because, in practice,
all packages use Cabal as their build system -- making the per-package build
system abstraction an artificial limitation.
We propose a way forward to lift this restriction, which will establish
foundations for improvements in tooling based on Cabal, and make Cabal
easier to maintain in the long term.

The key obstacle to changing this design is the existence of the `Custom`
build-type, through which a package may supply an arbitrary `Setup.hs` file
implementing its build system.  While the majority of packages do not use this
feature, a significant minority do, and thus we need to provide a migration path
that allows them to adapt to the new architecture. The primary technical content
of this proposal is a design for a new build-type that addresses the fundamental
design issues with the `Custom` build-type, while still allowing packages to
augment the build system according to their needs.

This work is being carried out by Well-Typed LLP thanks to investment from the
Sovereign Tech Fund. (For more information, read our [blog post announcing the
project](https://www.well-typed.com/blog/2023/10/sovereign-tech-fund-invests-in-cabal/).)
It has previously been discussed at [Cabal issue #9292](https://github.com/haskell/cabal/issues/9292),
and is now being [discussed at this pull request](https://github.com/haskellfoundation/tech-proposals/pull/60).

## Background

A fundamental assumption of the existing Cabal architecture is that each package
supplies its own build system (provided by `Setup.hs`), with Cabal specifying the interface to that build
system, for instance, the behaviour of `./Setup.hs build` or `./Setup.hs install`.

Modern projects consist of many packages. However, an aggregation of
per-package build systems makes it difficult or impossible to robustly implement
cross-cutting features (that is, build system features that apply to multiple
packages at once).  This includes features in high demand such as:

* fine-grained intra-package and inter-package build parallelism;
* rich multi-package IDE support;
* multi-package REPL.

The fundamental assumption has turned out to be false: all modern Haskell
packages implement the build system interface using a standard implementation
provided by Cabal itself, albeit in some cases with minor customisations. Cabal
already provides a build system based on declarative configuration, and the
majority of packages use this `Simple` build-type. A minority use the `Custom`
build-type, which allows wholesale replacement of the build system, but in
practice is mostly used to make minor customisations of the standard
implementation. It is the existence of this `Custom` build-type which is holding
us back, but its flexibility is mostly unused.

Thus the solution is to invert the design: instead of each package supplying its
own build system, there should be a single build system that supports many
packages.

By way of example, consider an IDE backend like HLS. An IDE wants to *be* the
build system (and compiler) itself, not for any final artefacts but for the
interactive analysis of all the source code involved. It wants to prepare (i.e.
configure) all of the packages in the project in advance of
building any of them, and wants to find all the source files and compiler
configuration needed to compile the packages. This is incompatible with a set
of opaque per-package build systems, each of which is allowed to assume all
dependencies already exist, and which can only be commanded to build artefacts.

The overall goal is the deprecation and eventual *removal of support for the
`Custom` build-type*. It is the removal of the `Custom` build-type that will enable
simplifications and easier maintenance in Cabal, and enable easier
implementation of new features.


### The current interface

Today, each package can provide a `Setup.hs` script defining how it should
be built.
The Cabal specification dictates that a `Setup.hs` script must obey a [specific
interface](https://cabal.readthedocs.io/en/stable/setup-commands.html#setup-commands).
One can build an individual package by first compiling `Setup.hs`
and then invoking:

```
./Setup configure
./Setup build
```

Packages indicate how much customisation of the build process they require by
declaring which
[`build-type`](https://cabal.readthedocs.io/en/3.10/cabal-package.html#pkg-field-build-type)
they use.  Currently there are four options:

| Type        | Description                                                                              |
|-------------|------------------------------------------------------------------------------------------|
| `Simple`    | Use the standard Cabal build system.  Most packages use this option. |
| `Configure` | Run a `./configure` script to discover system configuration, then build using the standard build system. |
| `Make`      | Invoke `make` to build the package using its own build system (obsolete). |
| `Custom`    | Compile and run a custom `Setup.hs` script defining the package's build system. |

The most flexible option used in practice is `build-type: Custom`.  In this
case, the package can define an arbitrary `Setup.hs` script which implements the
command-line interface defined by the Cabal specification. That is, the program
must support being executed as `./Setup configure`, `./Setup build`, and so on,
but it can implement whatever logic it wants.

In practice, almost all custom `Setup.hs` scripts depend on the `Cabal` library,
which allows for customisation of its build system by calling
`defaultMainWithHooks`, providing a value of the
[`UserHooks` datatype](https://hackage.haskell.org/package/Cabal-3.10.1.0/docs/Distribution-Simple-UserHooks.html).
However, nothing about the `Setup.hs` interface guarantees this, so build tools
must pessimistically assume that `Setup.hs` may do anything.

This means that where a package in the build plan uses `Custom`, build tools
such as `cabal-install` must fall back on legacy code paths to compile it. This
causes various problems and requires hacky workarounds.  In particular, the
`Setup.hs` interface works only with whole packages, and cannot easily be
changed, even though modern `cabal-install` versions support building individual
components independently (e.g. compiling a library and one selected test-suite
without compiling other test-suites in the same package).


### Why do packages use the `Custom` build-type?

We have been surveying existing Haskell packages that currently use the `Custom`
build-type, identifying where their requirements can be fulfilled with the
`Simple` build-type using existing declarative features, where new declarative
features would be useful, or where extending the build system is necessarily
required.  The [survey report](https://github.com/well-typed/hooks-build-type/blob/main/survey.md)
describes packages we investigated in more detail.

There are various reasons why packages currently use the `Custom` build-type, such as:

 * detecting system configuration;

 * generating source code for modules during a build (e.g. to make information
   about the build environment available to the final executable);

 * adding build system features which are not natively supported by Cabal, such
   as doctests (though it is debatable whether this is a good approach to
   integrating doctests);

 * working around bugs in build tools; or

 * executing additional steps when a package is installed (e.g. the Agda
   compiler is a normal Haskell package that needs to compile some Agda
   libraries when it is installed).

Over time, Cabal has gradually incorporated more features to allow some of these
use cases to be subsumed, typically by adding more declarative information to
the `.cabal` file format.  For example,
[`pkgconfig-depends`](https://cabal.readthedocs.io/en/3.10/cabal-package.html#pkg-field-pkgconfig-depends)
reduces the need for packages to have custom configuration logic. Similarly,
`build-type: Configure` can be used to implement configuration logic in a more
targeted way than `build-type: Custom`.

However, this necessarily captures only the more common use cases. There is a
"long tail" of packages using the `Custom` build-type for their own very specific
needs.


## Problem Statement

This proposal aims to solve the problem that Cabal's architecture currently
prevents build tools from having complete control of the build system, and hence
limits their development and makes maintenance harder.


### How can we move away from the `Custom` build-type?

We are engaging with package maintainers to remove the requirement for the
`Custom` build-type where better alternatives exist.  For example, some packages
use the `Custom` build-type only to support doctests, and in these cases it is
often relatively easy to switch to the simple build-type and use a different
mechanism to run doctests.

However, it is not feasible to simply migrate all existing uses of the `Custom`
build-type to use the simple build-type.  Some packages still need to customise
the behaviour of the build system, such as running code during a configuration
step to gather information about the host system, where this dependency cannot
be expressed declaratively.  We believe that **it should be possible to
augment the Cabal build process in unanticipated but controlled ways, to cover
parts of the build process that were not imagined or supported by the Cabal
maintainers**.

Thus, for selected packages, replacing the `Custom` build-type will require a new
mechanism, which will permit controlled extensions
to the build system. This mechanism should be designed on the basis that the
build tool, not each individual package, is in control of the build system.
Moreover, the architecture needs to be flexible to accommodate future changes,
as new build system requirements are discovered.


### Requirements

The key requirements for the new mechanism are as follows:

 * It should provide an alternative to the `Custom` build-type for packages that
   need to augment the Cabal build process, based on the principle that the
   build system rather than each package is in overall control of the build.

 * It should support essentially all of the existing uses of the
   `Custom` build-type.

 * It should minimise the effort required from package maintainers.  While some
   work to migrate away from the `Custom` build-type is inherently necessary, we
   need the migration path to be straightforward.

 * It should integrate with existing build systems; see
   [§ Integration with existing build systems](#integration-with-existing-build-systems).

 * It should make limited assumptions about how the build tool will operate, and
   in particular should not *require* use of the `Setup.hs` interface.

 * It should learn lessons from previous efforts to migrate away from the `Custom`
   build-type.

 * It should be open to future evolution of the design, as new requirements
   become clear, rather than getting stuck in a local optimum where there is
   again a difficult migration problem.

A crucial goal of this work is making the `Cabal` library easier to maintain and
develop over the long term, but it should also have benefits for downstream
tools.  In particular, Haskell build tools such as `cabal-install` and `stack`
will have more flexibility to implement their build systems, making it easier to
add features such as fine-grained cross-package build graphs (leading to more
parallelism for faster builds), more accurate file change detection, etc..

#### Integration with existing build systems

The `Hooks` `build-type` should integrate with the following three different
ways of building `Cabal` packages:

  * Building Haskell packages with `cabal-install`. In this case, we want to
    expose enough information to enable features such as per-module build graphs,
    coordination of parallelism through `-jsem`, and multi-repl.

  * Building Haskell packages inside other build systems using the `Setup.hs`
    interface. This is important as we don't want to break the workflow of
    RPM/DEB distribution packagers, Nix packages, etc.

  * The Shake-like build system of the Haskell Language Server. Here, we want
    HLS to be able to re-run actions on demand as part of an interactive
    developer environment.

### Non-requirements

Although it is not possible or practical to address too many problems at once,
it is a goal to make the new API more evolvable than the old `UserHooks`
API. Thus we believe that it will be possible to adapt the design more easily to
future features and requirements.

For example, `Cabal` does not currently have proper support for
cross-compilation, because it does not make a clear distinction between the build
and host.  This means the `Custom` build-type currently leads to issues with
cross-compilation, and in the first instance, the new design may inherit the
same limitations.  This is a bigger cross-cutting issue that needs its own
analysis and design.


## Prior art and related efforts

The `Cabal` developers have long been aware of the limitations arising from
`build-type: Custom` (see for example [#2395 Proposal for a Cabal plugin API
(inversion of control)](https://github.com/haskell/cabal/issues/2395) and [#3065
Lessons learned from Custom](https://github.com/haskell/cabal/issues/3065)).
Over time, there have been attempts to gradually move packages away from
`Custom`, in some cases by adding declarative features to `build-type: Simple`
instead, which is preferable where possible.

Since the remaining "long tail" of packages have varied needs, however, we believe it is
better to design a more general mechanism for augmenting the process rather than
many specific knobs, so that integrating the new mechanism into Cabal and other
build-systems is more straightforward and general.  We presume that any existing
usage of `Custom` that merely augments the build process is justified and valid,
and seek to provide an alternative build-type to which existing packages can be
directly migrated.
The introduction of the alternative build-type that captures existing `Custom`
extensions does not preclude the addition of declarative features that subsume
use cases for it.

### Issues with `UserHooks`

As noted, the `Cabal` library already provides a customisable build system in
the form of the `defaultMainWithHooks` function and the
[`UserHooks` datatype](https://hackage.haskell.org/package/Cabal-3.10.1.0/docs/Distribution-Simple-UserHooks.html).
Thus, it would be possible to define a build-type based on the package author
providing a value of the existing `UserHooks` datatype directly
(rather than providing a `Setup.hs` file which just happens to call `defaultMainWithHooks` on such a value).

However, redesigning the hooks interface gives us the opportunity to learn
lessons from the original `UserHooks` design, and take into account the
intervening years of Cabal development and the resulting changes in
architectural details (such as multiple components support).

The existing `UserHooks` mechanism used for the `Custom` build-type presents a
few problems (see [#3600 Hook redesign](https://github.com/haskell/cabal/issues/3600) and other
[tickets labeled `Hooks`](https://github.com/haskell/cabal/issues?q=is%3Aissue+is%3Aopen+Hooks+label%3A%22Cabal%3A+hooks%22)):

* It is too expressive, as it allows users to override entire build phases. This flexibility
  turns the building of a package into a black box, which pessimises certain parts
  of `cabal-install`.
* It is opaque, as all the customisation is encapsulated inside the `Setup` executable.
  This means that build tools such as `cabal-install` or `HLS` have no way to
  inspect which customisations have been made (this means for example that `HLS`
  is not aware of any `hookedPreProcessors` declared by the user).
* It often isn't possible to perform the customisation you need using only pre/post hooks, as they aren't expressive enough.
  This leads to users instead overriding the main phase, manually taking some pre/post steps
  and propagating the information to/from the main build phase.
* Configuration customisation is not propagated: any modifications to the project configuration have
  to be reapplied in each hook. This is quite unintuitive, as you would expect to only have
  to apply an update to the results of configuration once, instead of needing to re-apply it
  in every build phase.
* The interface is not component aware. The hooks were designed before Cabal had support for multiple components,
  so it's quite awkard to provide a hook which affects just one component. At best, it's possible
  to handle the main library and executables, but there is no concept of named sublibraries or
  of other component types such as testsuites or benchmarks in the `UserHooks` design.
* The API is hard to change in a backwards compatible way, and so it has become ossified.

As will be made clear in later sections, our proposed `SetupHooks` type takes
these issues into account and provides a carefully thought out, cohesive
interface to augmenting the Cabal build process.


### `code-generators`

The
[`code-generators`](https://cabal.readthedocs.io/en/3.10/cabal-package.html#pkg-field-test-suite-code-generators)
feature is intended for defining test suites with source code created via test
autodiscovery. It allows an executable to be invoked that is passed some options
describing the build configuration, and is expected to generate some modules in
response.

This provides a `Cabal`-native alternative for some uses of the custom
build-type for test autodiscovery.  Where such an alternative exists and is
suitable for the needs of a particular package, it is fine to make use of
it. Indeed, where it is possible to add declarative features to describe the
build configuration, those are likely to be preferable to executing arbitrary
custom code during the build.

However, the declarative features supported by `Cabal` are necessarily limited,
and will not always meet the needs of package authors. For example,
`code-generators` cannot be used to generate modules in components other than
test suites, and it has access to only one set of GHC options, but in general
each component may have different options (see [Cabal issue
#9238](https://github.com/haskell/cabal/issues/9238)).

Thus we do not think it is feasible to completely migrate away from the custom build-type
merely by adding features like `code-generators`, but not a more general,
uniform mechanism.


## High-level design of `build-type: Hooks`

We propose to augment `Cabal` with a new build-type, `Hooks`.  
To implement a package with the `Hooks` build-type, the user needs to provide
a `SetupHooks.hs` file which specifies the hooks using a Haskell API.

The `Hooks` build-type represents a middle-ground of customisation which specifically
only permits *augmenting* the build process at specific points, while disallowing
the complete replacement of individual build phases.


### `Hooks` from the package author's perspective

When `build-type: Hooks` is specified in the `.cabal` file,
the package author must supply a Haskell file named `SetupHooks.hs` that defines
a value `setupHooks :: SetupHooks`, which is a record of user-specified
hooks. This means that this interface is fundamentally a Haskell library interface,
not a command line interface (unlike `build-type: Custom`, which simply specifies
a replacement for the the `Setup.hs` CLI with no other means of interaction).

A hook is a Haskell function, with a type such as `HookInputs -> IO HookOutputs`
where `HookInputs` and `HookOutputs` are types specific to the particular hook.
See [§ Effects available to hooks](#effects-available-to-hooks) for discussion
of the choice of the `IO` monad.

Each hook is optional, e.g. each field of the `ConfigureHooks` datatype has a type
of the form `Maybe Hook`.  This means that, when using the library interface
to hooks, the build tool can statically determine that there is no hook at that
particular stage, which might enable certain optimisations.

See [§ Library API and versioning](#library-api-and-versioning) regarding which
Haskell library defines the `SetupHooks` datatype and any types describing the
inputs and outputs of particular hooks.

A `.cabal` file using `build-type: Hooks` must include a `custom-setup` stanza
with a `setup-depends` field describing the build dependencies of
`SetupHooks.hs` (just as with the `Custom` build-type).


### `Hooks` from the build tool's perspective

Since the API is expressed using a Haskell library, rather than a CLI, build
tools such as `cabal-install` have a choice of implementation techniques for how
they execute the hooks.

In order to maintain backwards compatibility with build systems which solely use
the `./Setup.hs` interface (such as `nixpkgs` and Linux distribution packagers),
`Cabal` will provide a function `defaultMainWithSetupHooks :: SetupHooks -> IO ()`
which will ensure that the hooks are invoked in the correct place during the
normal build pipeline.  Then the source distribution of a package using the
`Hooks` build-type will contain an automatically-generated shim `Setup.hs` file
of the following form:

```haskell
import Distribution.Simple ( defaultMainWithSetupHooks )
import SetupHooks ( setupHooks )

main = defaultMainWithSetupHooks setupHooks
```

The build tool can compile the shim `Setup.hs` and run the traditional CLI
commands such as `./Setup configure` and `./Setup build`.  This does not realise
the full benefits of the new build-type, but it means that the change is completely
transparent to existing tools, as they can continue to use the `Setup` interface
without any modifications.

With `Hooks`, however, build tools will be able to use alternative
implementation techniques for executing the hooks, rather than being forced to
go through the `Setup.hs` interface:

* The build tool could define a different IPC interface that can invoke
  individual hooks selectively. It could then compile a "hooks executable" that
  exposes `SetupHooks.hs` via this interface. This will allow finer-grained
  build plans, as is described in [§ Future work](#future-work).

* The details of such IPC interfaces are under the control
  of the build tool, not dictated by the specification (e.g. the hooks
  executable could provide a command-line interface for each hook where hook
  inputs/outputs are serialised, or it could be started as a child process and
  communicate over a pipe).

* We could even imagine the build tool dynamically loading `SetupHooks.hs` into an
  existing process so that it can invoke hooks directly with minimal overhead.

Crucially, hooks are independent, in the sense that each can be invoked
separately however the build tool arranges to do so.

### Designing for future compatibility

The design strategy for the hooks API should encourage clients to use it in ways
that are unlikely to break when it is evolved in the future.  This includes:

 * offering high-level APIs and reusable utilities for common operations where
   possible, rather than requiring clients to depend on the details of
   particular types;

 * using field names rather than positional constructors;

 * using a distinct record type for each hook's inputs and outputs, so that
   fields can be added to the record in the future;

 * hiding the underlying data constructors and providing smart constructors
   instead.

### Library API and versioning

It is important to be clear what future compatibility guarantees are offered by
`Cabal` and/or build tools to packages using `Hooks`.  There is a tension here,
because we do not want package maintainers to be over-burdened with continual
changes to support newer versions of the hooks API, but neither do we want
`Cabal` maintainers to be over-burdened by the costs of providing backwards
compatibility.

We propose to introduce a new library, `Cabal-hooks`. A package using the
`Hooks` build-type must declare a dependency on the
`Cabal-hooks` library in the `setup-depends` field of their package.
The range of `Cabal-hooks` library versions declared in `setup-depends`
indicates the versions of the hooks API that the package supports.

The requirement for such a dependency codifies existing practice. Indeed,
while, in theory, a package using `build-type: Custom` can implement its `Setup`
script without depending on `Cabal`, we saw that this flexibility was unused in
practice, as `Setup` scripts end up being defined in terms of `UserHooks`.
This usage pattern incurs a corresponding dependency on the `Cabal` library in
`setup-depends`, in much the same way as we propose here for `Cabal-hooks`.  
An additional benefit of the separate `Cabal-hooks` library is that it makes it
possible to evolve the Hooks API without requiring a version bump of the
`Cabal` library.

In practice, we expect the initial versions of `Cabal-hooks` to mostly
re-export `Cabal` datatypes, as it is these types (such as `LocalBuildInfo`)
that get passed back-and-forth between the build system and the hooks in our
current design (see e.g. [§ Configure hooks](#configure-hooks)).  
This design choice does introduce some coupling between the versions of
`Cabal-hooks` and `Cabal` (but see [§ Decoupling `Cabal-hooks`](#decoupling--Cabal--hooks-)).
At any rate, this design makes the situation no worse than with `Custom`
(because a shim `Setup.hs` can still always be used to compile `SetupHooks.hs`
using an older version of `Cabal-hooks`), but it gives more options to the build tool,
e.g. where serialisation of hook inputs/outputs is used, the serialisation
format can be controlled by the build tool, and is not necessarily fixed by `Cabal-hooks`.
Indeed, we could imagine a build tool being compiled against multiple `Cabal-hooks`
versions.  
The version compatibility problem exists in
`cabal-install` already: even where communication happens via the `Setup.hs`
command line interface, there is already a need for `cabal-install` to adapt to
the command-line flags that are supported by the version of `Cabal` in use (see
e.g. [`filterConfigureFlags`](https://hackage.haskell.org/package/cabal-install-3.10.2.1/docs/src/Distribution.Client.Setup.html#filterConfigureFlags)).
Once this proposal removes the need for `cabal-install` to go through the
`Setup.hs` interface, there is a potential for a significant reduction in
complexity here.

## Detailed design of `SetupHooks`

Having described `build-type: Hooks` in the previous section, the remaining part
of the design process is to work out the specific interfaces for the individual
hooks as expressed by the `SetupHooks` type.

We want to arrive at a design by the following means:

* The consideration of the existing usage of `Setup.hs` scripts to guide
  what hooks should be able to do.
* The needs of the rest of the Haskell ecosystem, in particular the Haskell
  Language Server.
* A high-level understanding of what the build process of a package should
  look like, taking into account concerns such as parallelisability.

These viewpoints can inform each other about the precise details for the design.

As part of the design process we have been developing a [prototype
implementation of this design](https://github.com/mpickering/cabal/tree/wip/setup-hooks)
in the `Cabal` library.

This section unavoidably relies on a deeper understanding of the `Cabal` build
system than the previous sections.

### Phases

The `Cabal` build process defines various phases that package authors should
be allowed to customise in some way:

 * The *configure phase* is when decisions are made about how to perform the
   subsequent phases (e.g. which tools and options to use).  This
   may involve running arbitrary code to detect information about the host
   system.

 * The *build phase* is when the project is compiled (including for the REPL)
   and build artifacts are generated (including libraries, executables and other
   artifacts such as Haddock documentation).

 * The *install phase* is when build artifacts are moved from the build directory
   to the final installed location or installation image (e.g. for subsequent packaging).

We propose to extend these three phases, with the following high-level structure
for `SetupHooks`:

```haskell
data SetupHooks = SetupHooks
  { configureHooks :: ConfigureHooks
  , buildHooks     :: BuildHooks
  , installHooks   :: InstallHooks
  }
```

See [§ Configure hooks](#configure-hooks), [§ Build hooks](#build-hooks)
and [§ Install hooks](#install-hooks).

Unlike the old `UserHooks` datatype, there is deliberately no way for the
package to remove or replace existing phases wholesale (such as replacing the
`buildHook`), and it is not possible to change the behaviour of operations
such as tests, benchmarks and cleanup.  Nor is it possible to add entirely
new phases, because which phases are available is
determined by the overall design of the build system, not the individual package.

### Cabal configuration type hierarchy

Cabal has a number of datatypes which are used to store the result of
configuration. We will briefly describe them here before getting into
the precise design of the hooks.

* `LocalBuildInfo`: the whole result of the configure phase.
* `GenericPackageDescription`: the parsed version of a `.cabal` file.
* `PackageDescription`: a resolved `GenericPackageDescription`, flattened relative to a flag assignment
  (see [`Distribution.Types.PackageDescription`](https://hackage.haskell.org/package/Cabal-syntax-3.10.2.0/docs/Distribution-Types-PackageDescription.html)), may contain several `Component`s.
* `Component`: a sum type which captures the possible different component types such as `Library`, `Executable`, etc..
* `BuildInfo`: the shared part of a component which describes the options which will
                be used to build it.
* `ComponentLocalBuildInfo`: additional information which `Cabal` knows about a component which is not present in `Component`.
   This is usually pieced together from the parent `LocalBuildInfo` and the individual `BuildInfo` of the `Component`.
* `ConfigFlags`/`BuildFlags`/`HaddockFlags`: flags to `./Setup configure`, `./Setup build`, `./Setup haddock`, etc..
* `TargetInfo`: all the information necessary to build a specific target (combination of a `Component` and `ComponentLocalBuildInfo`).

### Configuring and building

All decisions about *how to build* a project should be made in the configuration
phase.  Hooks during the build phase should not (re)calculate options, but
should only be used for actually *building* the package.
Therefore, it is the hooks to the configuration phases which have the ability to
augment the build environment with additional settings. The hooks for other
phases will receive this configuration in their inputs, and must honour it.
See [§ Phase separation](#phase-separation).

Configuration happens at two levels:

 * global configuration covers the entire package,
 * local configuration covers a single component.

Once the global package configuration is done, all hooks should work on a
per-component level. This avoids introducing additional synchronisation points
in a build that would limit the amount of available parallelism.

### Configure hooks

We propose to add the following configure hooks:

```haskell
type PreConfPackageHook    = PreConfPackageInputs -> IO PreConfPackageOutputs
type PostConfPackageHook   = PostConfPackageInputs -> IO ()
type PreConfComponentHook  = PreConfComponentInputs -> IO PreConfComponentOutputs

data PreConfPackageInputs
  = PreConfPackageInputs
  { configFlags      :: ConfigFlags
  , localBuildConfig :: LocalBuildConfig
  , compiler         :: Compiler
  , platform         :: Platform
  , programDb        :: ProgramDb
  }

data PreConfPackageOutputs
  = PreConfPackageOutputs
  { localBuildConfig :: LocalBuildConfig
  , configuredProgs :: ConfiguredProgs }

data PostConfPackageInputs
  = PostConfPackageInputs
  { localBuildConfig  :: LocalBuildConfig
  , packageBuildDescr :: PackageBuildDescr
  }

data PreConfComponentInputs
  = PreConfComponentInputs
  { localBuildConfig  :: LocalBuildConfig
  , packageBuildDescr :: PackageBuildDescr
  , component         :: Component
  }

data PreConfComponentOutputs
  = PreConfComponentOutputs
  { componentDiff :: ComponentDiff }

data ConfigureHooks
  = ConfigureHooks
  { preConfPackageHook    :: Maybe PreConfPackageHook
  , postConfPackageHook   :: Maybe PostConfPackageHook
  , preConfComponentHook  :: Maybe PreConfComponentHook
  }
```

From the build tool's perspective, the global configuration phase goes as follows:

- Firstly, decide on the initial global configuration for a package,
  producing a `LocalBuildConfig`.

- Run the `preConfPackageHook`, which has the opportunity to modify the
  initially decided global configuration (producing a new `LocalBuildConfig`). After
  this point, the `LocalBuildConfig` can no longer be modified.

- Use the `LocalBuildConfig` in order to perform the global package
  configuration. This produces a `PackageBuildDescr` containing the information
  Cabal determines after performing package-wide configuration of a package,
  before doing any per-component configuration.

- Run the `postConfPackageHook`, which can inspect but not modify the result of
  the global configuration. This can be used to propagate custom package-wide
  logic to the subsequent per-component configure hook (and is used for
  example to re-implement the `Configure` `build-type`).

After the global configuration has completed, individual components can be
configured independently, as follows:

- Run the `preConfComponentHook`. This is the only means to apply specific
  options to a `Component`.

- Use the modified `Component` to perform per-component configuration and create
  the `ComponentLocalBuildInfo`.

#### Phase separation

Only configure hooks can make changes to the `PackageDescription`.  Once
configuration is finished, the package description should be set in stone, and
subsequent hooks such as build hooks are not able to modify it.
This differs from `UserHooks`, where modifications to the project configuration
in the configure phase are not propagated to the other phases, and instead
subsequent hooks must re-apply changes, in the form of `HookedBuildInfo`.
This old design lead to several subtle bugs and maintenance headaches which
this new design will allow us to get rid of, once we remove support for
the `Custom` build-type.

The configuration hooks thus follow a simple philosophy:

* All modifications to global package options must use `preConfPackageHook`.
* All modifications to component configuration options must use `preConfComponentHook`.

If a hook modifies the options in these phases, then the configuration is propagated
into all subsequent phases, and the design of the interface ensures that
this is the only point where hooks can modify the options.

It is only the pre-configuration hooks which allow modification of the options.
This is because the configuration process computes some more complicated data
structures from these initial inputs. If hooks were allowed to modify the results
of configuration then it would be error-prone to ensure that they suitably updated
both the options in question as well as the generated configuration.
For example, both `PackageDescription` and `ComponentLocalBuildInfo` contain
a list of exposed modules for the library.
This is why the "post" configuration hook (and any hooks subsequent to the
configure phase) can only run an `IO` action; they can't return any modifications
that would affect the `PackageDescription`.

#### `LocalBuildConfig`

There are parts of the `LocalBuildInfo` which must be decided at a global
(per-package) level; for instance, whether to build dynamic libraries.
On the other hand, there are also things we want to decide on a local
(per-component) level, such as specific GHC options with which to compile the
component.

Moreover, there are parts of the `LocalBuildInfo` which hooks cannot modify.
For example, things such as package dependencies can't be modified because they are
determined externally by the overall build plan (e.g. from the dependency solver).
Thus, the hooks interface should prevent the modification of these
parts of `LocalBuildInfo`.

We propose to achieve this by defining a new type `LocalBuildConfig` which
contains only the parts of the existing `LocalBuildInfo` datatype that can be
modified by `preConfPackageHook`.

#### `ComponentDiff`

The `ComponentDiff` records the modifications that should be applied to each component.

For each component, `preConfComponentHook` is run, returning a `ComponentDiff`.
This `ComponentDiff` is applied to its corresponding `Component`
by monoidally combining together the fields.

```haskell
newtype ComponentDiff = ComponentDiff { componentDiff :: Component }

emptyComponentDiff :: ComponentName -> ComponentDiff
```

The diff is represented by a `Component`; not all fields of a `Component` are
allowed to be modified, and when the diff is applied it is dynamically checked
that the hook has not modified any fields which it shouldn't.

Some alternative designs:

  1. Specify a Haskell function `Component -> Component` which can modify
     the component at will.
  2. Define a custom `ComponentDiff` datatype which contains only the fields
     of a `Component` which we allow hooks to modify.

The benefit of (2) is that it trims down the amount of internal details exposed
from `Cabal`,making it less likely that an internal change in Cabal would end up
breaking the `Hooks` defined by package authors. However, one would need to
ensure this interface is general enough in order to avoid locking out Hooks
authors, e.g. if `Cabal` adds a new field to `Component` without updating the
corresponding `ComponentDiff` type in order to make it modifiable by hook authors.  
If we end up with a design in which `Cabal`'s version of the `Component` type
is necessarily separate from the type in the hooks API, we may want to reconsider
this alternative.

### Build hooks

The design of the pre-build hooks has generated significant discussion during review.
There are several trade-offs. The initial proposal was essentially a port of the
build hooks in the old `UserHooks`, but updated to the per-component world.
This included monolithic pre and post hooks for each component, plus the existing
"hooked pre-processors" abstraction. This had the advantage that it would be easy
for package authors to port their `Setup.hs` scripts to the new design, and it was
a relatively minimal change in the Cabal codebase.  
Many Cabal contributors share a long term goal to move the Cabal design towards
one based on a build graph with fine-grained dependencies. From this perspective,
the critique was that the initial proposal was too conservative a change, and that
we should take this opportunity of making a significant API change to establish a
new API that would not hold back the move towards finer-grained dependencies.
Another critique was that the original `UserHooks` design was somewhat ad-hoc,
since it used both monolithic hooks and hooked pre-processors to provide
finer-grained dependencies for a modest subset of use cases.  
On the other hand, there is a very large design space for finer-grained
dependencies, and so picking a point in the design space is not simple.
Another disadvantage is that it will of course be more work for package authors
to port their existing `Setup.hs` scripts, which currently use monolithic hooks.

The proposed design for pre-build hooks tries to balance these trade-offs.
Instead of an ad-hoc combination of monolithic hooks and hooked pre-processors,
we use a single general system of rules, but we take a relatively conservative
approach to the expressive power of the rules. In particular, the style of the
rules is relatively low level. For example it does not include "rule patterns"
such as generating a `*.hs` from a `*.y`. Instead, each rule specifies the
individual files involved as inputs and outputs. It should nevertheless be
possible to build higher level patterns on top, using Haskell's usual powers of
abstraction to generate the lower level rules. Crucially, the design allows the
rules to be used across an IPC interface, which is necessary for build tools
like `cabal-install` or HLS to be able to interrogate and invoke them.  

The full details of the design of pre-build hooks are provided in
[§ Pre-build hooks](#pre-build-hooks).

On top of pre-build hooks, we also propose a limited notion of post-build hooks,
which accomodates package authors which need to perform an `IO` action in order
to modify executables after they are built, as described in
[§ Post-build hooks](#post-build-hooks).

To summarise, we propose two different kinds of build hooks:

```haskell
-- | Build-time hooks.
data BuildHooks
  = BuildHooks
  { preBuildComponentRules :: Maybe PreBuildComponentRules
  , postBuildComponentHook :: Maybe PostBuildComponentHook }
```

Build hooks cannot change the configuration of the package.  
There are deliberately no package-level build hooks, only component-level hooks.
This avoids introducing unnecessary synchronisation points when multiple
packages/components are being built in parallel.

#### Post-build hooks

Separately from the pre-build rules, we also propose to introduce post-build
hooks. These cover a simple use case: namely to perform an IO action after an
executable has been built.

This functionality gives package authors a way to modify an executable after
it has been built, which is useful if one wants to:

  - inject data into an executable after it has been built
    (see for example [§ executable-hash](#executable-hash)),
  - strip an executable with an external tool,
  - perform code signing on an executable, e.g. using `xattr`.

Post-build hooks are run after the normal build phase completes. This means that
a tool such as HLS would never run them, as in a sense HLS never finishes
building. Note however that, were HLS to support running test-suites, it would
run the post-build hooks for the testsuite right after building it, before
running it.

We propose the following simple API for post-build hooks:

```haskell
data PostBuildComponentInputs
  = PostBuildComponentInputs
  { buildFlags :: BuildFlags
  , localBuildInfo :: LocalBuildInfo
  , targetInfo :: TargetInfo
  } deriving (Generic, Show)

type PostBuildComponentHook = PostBuildComponentInputs -> IO ()
```

Note that this is a single monolithic step that would simply be re-run
any time the `build` action is re-run.

### Install hooks

The `install` hooks run allows package authors to run an extra `IO` action
when copying/installing a package:

```haskell
data InstallComponentInputs
  = InstallComponentInputs
  { localBuildInfo :: LocalBuildInfo
  , copyFlags      :: CopyFlags
  , targetInfo     :: TargetInfo
  }

type InstallComponentHook = InstallComponentInputs -> IO ()

data InstallHooks
  = InstallHooks
  { installComponentHook :: Maybe InstallComponentHook
  }
```

The install hooks can be used to install files per-component.
The main use case for install hooks is when set of things you want to install is
not fixed and predetermined. One example is Agda, which wants to run the built
`Agda` executable on the associated standard library `.agda` modules in order
to generate `.agdai` interface files for them. These should then be installed
alongside the `Agda` executable.
This allows users to obtain a functional Agda compiler by using the single
invocation `cabal install Agda`. This also means that we can use
`build-tool-depends: Agda` in other projects.

We could imagine a more declarative way of specifying this being introduced in
the future, in which case packages will be free to migrate to it gradually.
There is not necessarily a problem with having some overlap between hooks and
declarative features.

An alternative approach would be to regard as illegitimate any use cases which
treat `Cabal` as a packaging and distribution mechanism for executables, and on
that basis, cease to provide install hooks.  We do not follow this approach because
it would significantly inconvenience maintainers of packages that rely on this
behaviour (e.g. Agda and Darcs), for a relatively small reduction in complexity
in `Cabal`.

It is important that these install hooks are consistently run both when copying
and when installing, as this fixes the inconsistency noted in
[Cabal issue #709](https://github.com/haskell/cabal/issues/709).
There is no separate notion of an "copy hook", because "copy" and "install"
are not distinct build phases.

## Pre-build hooks

The pre-build hooks consist of a collection of fine-grained build rules.
These are run before building a particular component of a package.

### Motivation: fine-grained rules

Suppose that Cabal did not have built-in support for `happy`, then a
package making use of it might like to write a rule like this:

```
lib:my-component:module:Foo.Bar : src:blah/foo/bar.y
    ${happy:exe:happy} ${input[0]} -o ${output[0]}
```

The key components of such a rule description are:

  - The input of the rule (in this case, the source file `blah/foo/bar.y`).
  - The output of the rule (the Haskell module `Foo.Bar`, inside a particular
    component of the package).
  - The action to run (in this case running the executable `happy`).
    Note that it is the build system that decides where inputs/outputs are
    located (in this case, the rule refers to them using `${input}` and
    `${output}`).

In particular, such a design fits the needs of the Haskell Language Server,
which needs to be able to:

  1. Query the package for all its pre-build rules.
  2. Find out all the rules that have become stale and need to be re-run.
  3. Execute individual rules.

However, the textual description of rules presented above suffers from some
limitations that would make migrating existing packages with `Custom` build-type
difficult. In particular, one often wants to allow rules to depend on each other
in a more dynamic manner.  
For example, consider how one might want to query an external executable in
order to determine the dependency structure; say by running
[`ghc -M`](https://downloads.haskell.org/ghc/latest/docs/users_guide/separate_compilation.html#makefile-dependencies)
on a root Haskell module in order to compute a build graph.

### Motivation: a simplistic first design

To explain the design we have arrived at for fine-grained pre-build rules, let
us first consider what a first draft design, which accommodates both the design
of HLS as well as the existing `Custom` setup scripts, might look like:

```haskell
type TentativeRules env = env -> IO [TentativeRule]
data TentativeRule = TentativeRule
  { dependencies :: [Dependency]
  , results :: [Result]
  , action :: [ResolvedLocation]
               -- ^ locations of __dependencies__ (determined by the build system)
           -> [ResolvedLocation]
              -- ^ locations for __results__ (chosen by the build system)
           -> IO ()
  }
```

That is, rules are specified by a function that takes in an environment
(which in practice consists of information known to `Cabal` after configuring,
e.g. `LocalBuildInfo`, `ComponentLocalBuildInfo`).
This function returns an `IO` action that computes a list of rules. Each rule
declares its inputs and outputs, and from this information arises the dependency
structure between rules.  
To run a rule, the build system must determine specific locations for all
these inputs and outputs, and pass them to the `action` in order to
execute the rule. For example, the build system would search for `blah/foo/bar.y`
in the source directories of the project, and pass an absolute path to the
file it found to the `action`.

### Proposed design of rules

There are several shortcomings with the above simplistic design:

  1. it does not fit well with the proposed IPC interface for hooks:
     namely, the build tool should be able to query the separate hooks
     executable to obtain all the hooks that a package with `Hooks` `build-type`
     provides. Doing so with the design proposed above would require a mechanism for
     serialising and deserialising the `IO` actions that execute the rules, which in
     practice would mean providing a DSL for `IO` actions that can be serialised,

  2. it lacks information that would allow us to determine when the rules need
     to be recomputed:

       a. if the rules were computed by invoking `ghc -M` (or similar), we would
          need to recompute them if the user adds a new file that would
          have been found by that call to `ghc -M`.

       b. if the `env` environment changed, we might or might not need to recompute
          the rules. We should have a mechanism for rules to declare what part
          of the environment they depend on, so that we don't need to pessimistically
          rerun the computation anew each time.

We propose to fix (1) by adding an extra layer of indirection: instead of a
`Rule` directly storing an `IO` action, it stores a reference to an action.
We can then separately query the external hooks executable with this reference
in order to run the action.

We fix (2) by adding `monitoredValue` and `monitoredDirs` fields to `Rule`.

We thus propose:

```haskell
newtype Rules env =
  Rules { runRules :: env -> ActionsM ( IO [Rule] ) }

data Rule = Rule
  { dependencies :: ![ Dependency ]
     -- ^ Dependencies of this rule.
     --
     -- When the build system executes the action associated to this rule,
     -- it will resolve these dependencies and pass them as an argument
     -- to the action, in the form of a @['ResolvedDependency']@.
  , results :: ![ Result ]
     -- ^ Results of this rule.
  , actionId :: !ActionId
     -- ^ To run this rule, which t'Action' should we execute?
     --
     -- The t'Action' will receive exactly as many 'ResolvedLocation'
     -- arguments as there are 'Dependency' values stored in the
     -- 'dependencies' field.

  , monitoredValue :: !( Maybe ByteString )
     -- ^ A monitored value.
     --
     -- The rule is considered out-of-date when the environment passed to
     -- compute this rule changes and this value also changes.
     --
     -- A value of @Nothing@ means: always consider the rule to be out-of-date.
     --
     -- A value of @Just _@ means: consider the rule to be out-of-date if,
     -- after re-computing rules from the environment, the stored value
     -- has changed.
  , monitoredDirs :: ![ Location ]
     -- ^ Monitored directories; if the contents of these directories change,
     -- the rule is considered out-of-date.
  }

newtype Action =
  Action
    { action
      :: [ ResolvedLocation ]
           -- ^ locations at which the __dependencies__ of this action
           -- were found
      -> [ ResolvedLocation ]
           -- ^ locations in which the ___results__ of this action
           -- should be put
      -> IO () }
```

The specific monadic type of rules, namely

```haskell
env -> ActionsM ( IO [Rule] )
```

is meant to address concerns surrounding generation of `ActionId`s, as is
explained in [§ Identifiers for fine-grained rules](#identifiers-for-fine-grained-rules).
In practice, we can think of the rules as being specified by a Haskell
function with the following type:

```haskell
env -> ( Map ActionId Action, IO [Rule] )
```

### Dependency structure

Rule dependencies and results are declared using the following datatypes:

```haskell
data Dependency
  -- | Declare a dependency on a file from the current project that should
  -- be found by looking at project search paths.
  --
  -- This file might exist already, or it might be the output of another rule.
  = ProjectSearchDirFile
      !Location
        -- ^ where to go looking for the file
      !FilePath
        -- ^ path of the file, relative to a Cabal search directory
type Result = ( Location, FilePath )
data Location
  -- | A source file:
  --
  -- - for a rule dependency, we will go looking for it in
  --   the source directories and in autogen modules directories;
  -- - for a rule output, the file should be put in an autogen module directory.
  = SrcFile
  -- | A build-file, that belongs to some build directory.
  | BuildFile
  -- | A temporary file.
  | TmpFile
```

To illustrate, a rule could declare a dependency on `Parser.y` (using the
`dependencies` field). The build-system will look through appropriate
search paths to resolve this dependency, and pass the location in which the file
was found on disk as one of the elements of the list passed as the first argument
to the `action` stored in the `Action` that the rule refers to through its
`ActionId`. (Although see [§ No searching in fine-grained rules](#no-searching-in-fine-grained-rules)
for an alternative in which searching is performed when computing the rules
themselves.)

Note that we do not allow a rule to directly depend on another rule, as this
can easily introduce bugs. For example, suppose that `Rule {ruleId = 1}`
outputs `A.y` and `Rule {ruleId = 2}` depends on `A.y` in order to produce
`A.hs`. It is much more direct and robust for rule 2 to directly declare its
dependency on `A.y`, rather than on (the output of) rule 1, as the latter is
prone to breakage if one refactors the code and changes which rule generates
`A.y`.

### Rule demand

The general flow is that we can find, by traversing the `[Rule]`
returned from querying all pre-build rules, what all the dependencies of
rules are. Whenever any of these changes, we must then:

  1. Re-query the pre-build rules to obtain all up-to-date rules. This step
     is necessary because the dependency structure might have changed.
  2. Find out all the rules that are now stale and need to be re-run.
  3. Re-run all demanded stale rules by calling out to the separate hooks
     executable, passing the `ActionId` and additional action arguments to
     that executable in order to run the `Action` associated to each
     demanded stale rule.

A rule is considered **demanded** if:

  - it generates a Haskell module that is declared to be an autogenerated
    module of the component we are building, or
  - another rule that is itself demanded depends on the output of the rule.

The rules as a whole are considered **out-of-date** precisely when any of the
following conditions apply:

<dl>
  <dt>O1</dt>
  <dd>a file dependency of a rule has changed in some way,</dd>

  <dt>O2</dt>
  <dd>the environment passed to the computation of rules has changed,</dd>

  <dt>O3</dt>
  <dd>there has been a relevant change in a file or directory monitored
      by a rule.</dd>
</dl>

If the rules are out-of-date, we re-run the computation that computes
all rules. Once this is done, we compute which rules are stale.

A rule is considered **stale** if, after re-running the computation of all
of the rules, any of following conditions apply:

<dl>
  <dt>S1</dt>
  <dd>a dependency of the rule has been modified/created/deleted,
      or a (transitive) rule dependency of the rule is itself stale;</dd>

  <dt>S2</dt>
  <dd>the monitor value is stale, i.e. either:
     <ol>
      <li>the <code>monitoredValue</code> of the rule changed, or</li>
      <li>the rule declares <code>monitoredValue = Nothing</code>.</li>
     </ol>
  </dd>
</dl>

A stale rule becomes no longer stale once we run its associated action; the
build system is responsible for re-running the actions associated with
each demanded stale rule, in dependency order.

Justification:

  - (O1)/(S1) are clear. If we change a file that a rule depends on,
    the rule needs to be re-run.  
    Because the dependency structure might also change, we need to recompute
    all the rules first, before then re-running the ones that have been staled.
  - (O2) is also clear: if we change the environment, we need to re-compute
    the rules (as the rules are given by specifying a function from an
    environment).  
    Note that this covers the event of the package configuration changing
    (e.g. after `cabal configure` has been re-run).
  - (S2) is an optimisation that ensures we don't pessimistically re-run a rule
    every time we change the environment, but only when the monitored value changes.
  - (O3) covers the use case in which we invoke an external tool (such as
    `ghc -M`) which performs a search in order to compute a dependency graph.
    We want to ensure that, if the user adds a new module, we re-run this
    dependency computation.  
    We don't want to rely on the user necessarily re-configuring the package,
    especially as the package description might not necessarily have changed
    (even though, in common cases, one expects that adding a new source file
    would correspond to a new module declared in the `.cabal` file).

### Identifiers

Note that [§ Rule demand](#rule-demand) requires a notion of persistence across
invocations, as is necessary to compute staleness of individual rules. We can't
simply use the index of the rule in the returned `[Rule]`, as this might vary
if new rules are added. Instead, we propose that a rule be uniquely identified
by the set of outputs that it produces. This gives the necessary way to match
up rules output by two different executions of the `IO` action that computes
the set of pre-build rules.

The question of `ActionId` is somewhat different. With the above design, actions
are uniquely identified by `ActionId`s, but requiring users to manually generate
these `ActionId`s is unergonomic and potentially error-prone.  
In particular, one might want to combine the `Rules` declared by two different
libraries, as described in [§ Composing `SetupHooks`](#composing--setuphooks-).  
To avoid having to know that e.g. another package uses `ActionId 17`
(so as to avoid clashing with it), the implementation is free to provide a
monadic API that handles creation of fresh `ActionId`s in order to
ensure that these identifiers are correct by construction.  

We thus propose the following API for hook authors:

```haskell
newtype ActionId -- in practice a newtype around 'Int', but this is crucially
                 -- not exposed in the API in order to prevent users from
                 -- manually constructing these values

newtype ActionsM a -- in practice, something like 'State (Map ActionId Action) a',
                   -- but this is an implementation detail
  deriving (Functor, Applicative, Monad)

-- | Register an action. Returns a unique identifier for that action.
registerAction :: Action -> ActionsM ActionId

newtype Rules env =
  Rules { runRules :: env -> ActionsM ( IO [Rule] ) }
```

This frees the package author from the responsibility of handling identifiers
manually.

Note that having `ActionId`s vary across invocations of the external hooks
executable is not problematic: these depend purely on the input environment,
and every time this input environment changes we recompute the rules themselves.
So there is no risk that a `Rule` refers to an "outdated" `ActionId`, as long
as one makes sure that every time the environment changes we recompute the set
of rules before running any actions.

### Inputs to pre-build rules

One important observation that factors into the design of build hooks is that
many different Cabal phases can be thought of as "building something".
Indeed, preparing an interactive session or generating documentation
share many commonalities with a normal build.
For example, if one is generating modules in a pre-build phase, one
should also generate these same modules before running `GHCi` or `haddock`.
The fact that `UserHooks` has entirely separate hooks for conceptually
similar phases leads to bugs (e.g. [#9401](https://github.com/haskell/cabal/issues/9401)).

For this reason, we propose abstracting over these different build-like phases
in the build hooks:

```haskell
data BuildingWhat
  = BuildNormal   BuildFlags
  | BuildRepl     ReplFlags
  | BuildHaddock  HaddockFlags
  | BuildHscolour HscolourFlags

data PreBuildComponentInputs
  = PreBuildComponentInputs
  { buildingWhat :: BuildingWhat
  , localBuildInfo :: LocalBuildInfo
  , targetInfo :: TargetInfo
  } deriving (Generic, Show)

type PreBuildComponentRules = Rules PreBuildComponentInputs
```

This design ensures that the build hooks are consistently run in all
build-like phases. This reflects the common pattern in custom Setup scripts
that one would update the `haddock` and `repl` hooks to mirror the `build` hooks
(which one can easily forget to do, and end up with an unusable `repl`, for example, see `singletons-base`
which fails to update the `replHook`).

Note also the interaction with the `monitoredValue` field of `Rule`: when running
`cabal build && cabal haddock`, we might (or might not) want to re-run the build
hooks. The way hooks authors can choose which behaviour they want is to output
a `monitoredValue` that changes (or does not change, respectively) when the
`BuildingWhat` parameter changes, say from `BuildNormal _` to `BuildHaddock _`.

### Hooked preprocessors

`UserHooks` includes `hookedPreProcessors`, which makes it possible to associate
a file extension with a preprocessing operation that turns files with that
extension into Haskell source modules. Cabal will then execute such pre-processors
on all matching files before building, and re-run them as needed when the input files change.
This is similar to Cabal's built-in support for `alex`, `happy`, etc..

The framework of fine-grained pre-build rules subsumes this feature.
This presents some significant advantages:

  - It lifts a key restriction of `hookedPreProcessors`,
    namely that it assumes one input file will turn into one Haskell module
    (see [#6232 Extended preprocessors support](https://github.com/haskell/cabal/issues/6232)).
  - It ensures that external tools such as HLS have visibility into the custom
    preprocessors used by the package. In particular, HLS will be able to
    re-run hooked preprocessors on demand when the relevant source files change.
  - It naturally takes into account the correct dependency structure, obviating
    the need for hacky workarounds such as the `ppOrdering` field of `PreProcessor`
    which allowed one to re-order the modules in order to take into account
    dependency information. This workaround doesn't compose well, and it
    unnecessarily serialises the build graph.

## Examples

These examples are based on the [survey of packages using custom `./Setup.hs`
scripts](https://github.com/well-typed/hooks-build-type/blob/main/survey.md) we
undertook previously, and experimental patches we have created while testing the design
(see [§ Testing and migration](#testing-and-migration)).

#### Generating modules

One of the main uses of `Setup.hs` scripts is to generate modules.

To achieve this with `SetupHooks`:

  - The per-component configure hook, `preConfComponentHook`, should declare
    that it will generate these modules, by adding onto the `autogenModules`
    field of that component.
  - Fine-grained pre-build rules are provided, which specify how to generate
    the source code for these modules. This can for example take the form of
    an individual preprocessor rule for each module which can be re-run on demand,
    or a single monolithic rule that generates all the sources ex-nihilo and
    which never needs to be re-run.

#### `./configure` style checks

Checks for things to do with the system configuration can either be performed in
the global pre-configure hook or in a per-component pre-configure hook. The
results of these checks are then propagated into subsequent phases.

In addition, the `Configure` build-type can be implemented in terms of
`SetupHooks` by running the `./configure` script once per package in the global
configuration step, and then applying the result in the per-component configure
hook. That is:

- The `./configure` script is executed once in a `preConfPackageHook` and produces
  the `<pkg>.buildinfo` file which contains the modified `BuildInfo` for main library
  and executable component.
- In the `preConfComponentHook` the `<pkg>.buildinfo` field is read from disk and
  the configuration is applied to each component.

This also allows defining a variant of the `Configure` build-type, using
`SetupHooks`, which is component-aware (e.g. so individual components could have
their own `./configure` scripts used only if the specific component is built).

#### Doctests

Arguably, running doctests is a cross-cutting build system feature (rather than
a feature that should be defined independently by each package's build system).
Thanks to the new external command system for `cabal-install`, it is possible to
add a [`cabal doctest` external command using a simple script to invoke
`doctest`](https://github.com/mpickering/cabal-doctest-demo).  This avoids the
need for the package to use the `Custom` or `Hooks` build-type at all, which is
generally preferable in most cases.

In some cases
(e.g. [`pretty-simple`](https://github.com/cdepillabout/pretty-simple/pull/129)),
package authors may wish to integrate the doctests for their package into `cabal
test`, in particular so that doctests can easily be executed by build tools
other than `cabal-install`.  This will require the use of either
`code-generators` or the `Hooks` build-type.

#### Hooked programs

The `hookedPrograms :: [Program]` field of `UserHooks` allows a custom
`Setup.hs` file to specify new programs to be detected in the configure phase.

In the `SetupHooks` design, this use case is supported by returning from the
package-level pre-configure hook a collection of configured programs. Recall:

```haskell
type PreConfPackageHook = PreConfPackageInputs -> IO PreConfPackageOutputs

data PreConfPackageInputs
  = PreConfPackageInputs
  { ..
  , programDb :: ProgramDb
  }

data PreConfPackageOutputs
  = PreConfPackageOutputs
  { ..
  , configuredProgs :: ConfiguredProgs }
```

The configured programs returned by the package-wide pre-configure hook will
then be used to extend `Cabal`'s `ProgramDb`, which will then get stored in
`Cabal`'s `LocalBuildInfo` datatype and passed to subsequent hooks.  

Note that we require hook authors configure the programs themselves (using
functions provided by the hooks API). This is justified by the fact that
arbitrary unconfigured programs cannot be straightforwardly serialised:

```haskell
type UnconfiguredProgram = (Program, Maybe FilePath, [ProgArg])
data Program = Program
  { programName :: String
  , programFindLocation
      :: Verbosity
      -> ProgramSearchPath
      -> IO (Maybe (FilePath, [FilePath]))
  , programFindVersion :: Verbosity -> FilePath -> IO (Maybe Version)
  , ...
  }
```

#### Hooked preprocessors

[`binembed`](https://hackage.haskell.org/package/binembed-0.1.0.3/docs/src/Distribution-Simple-BinEmbed.html#withBinEmbed)
defines a preprocessor that turns `M.binembed` into `M.hs` by running the
`binembed` executable. This can be implemented as a fine-grained pre-build rule;
as explained previously, these subsume the previous notion of `hookedPreProcessors`.

#### executable-hash

The [`executable-hash`](https://hackage.haskell.org/package/executable-hash-0.2.0.4)
package supplies a function `injectExecutableHash :: FilePath -> IO ()`
that can be run once an executable has been built, in order to inject
information into that executable.

Package authors wanting to make use of this functionality require some form of
post-build hook, which is why we provide a `postBuildComponentHook` to cover
this use-case and allow such packages to migrate away from `build-type: Custom`.

#### Composing `SetupHooks`

A pattern observed in some `Setup.hs` code is that general-purpose `Cabal` build
system extensions are implemented as functions of type `UserHooks -> UserHooks`,
to allow for composition.

For example, the [custom `Setup.hs` for
`termonad`](https://hackage.haskell.org/package/termonad-4.5.0.0/src/Setup.hs)
composes two such functions, one to add CPP options based on detecting system
configuration, and the other to run doctests:

```haskell
main = do
  cppOpts <- getTermonadCPPOpts
  defaultMainWithHooks . addPkgConfigTermonadUserHook cppOpts $
    addDoctestsUserHook "doctests" simpleUserHooks
```

To make composition of independent hooks more direct, we provide a `Monoid
SetupHooks` instance, so this example
[can become](https://gitlab.haskell.org/mpickering/hooks-setup-testing/-/blob/0575b7dd85d2151ec6ce4936c5d1800de869e356/patches/termonad-4.5.0.0.patch):

```haskell
setupHooks :: SetupHooks
setupHooks = doctestsSetupHooks "doctests" <> termonadSetupHooks
```

In general the monoid is non-commutative; for example, for the
pre-configure hooks, the output of the first hook will be fed as an input to the
second.


## Alternatives

### Decoupling `Cabal-hooks`

Instead of `Cabal-hooks` re-exporting datatypes from `Cabal`, one could imagine
defining datatypes in `Cabal-hooks` instead; then `Cabal-hooks` would not depend
on `Cabal`.  
This would completely encapsulate the Hooks API, which would no longer be tied
to a particular `Cabal` version.  
Here are two conceivable ways in which this change might then impact `Cabal`:

  1. `Cabal` itself depends on `Cabal-hooks`. This is attractive from a
     modularity perspective, as it clearly defines the subset of the `Cabal`
     library that is exposed via the hooks API.
     However, it is not clear in advance how datatypes such as `LocalBuildInfo`
     should be split up without hampering the needs of hooks authors, and would
     likely involve moving quite a lot of code from `Cabal` to `Cabal-hooks`.

  2. `Cabal` does not depend on `Cabal-hooks`. Instead, `Cabal-hooks` would
     define entirely separate hook input/output data types. Some other library
     would then contain code for translating values of those datatypes into
     the internal representation in `Cabal` (and vice versa).
     This would require significant duplication of datatypes (and an associated
     maintenance burden for keeping them in sync), but it might provide more
     options for evolving the `Cabal` and `Cabal-hooks` datatypes independently.

The benefit of decoupling the version of the hooks API from the version of the
`Cabal` library depends on how the build tool is using `SetupHooks.hs`:

 - If the build tool is compiling a shim `Setup.hs`, it can in principle pick a
   compatible version of `Cabal` even if the build tool itself uses a different
   version, provided various constraints are satisfied (both those from the
   `setup-depends` of the package itself, and those arising from the build tool,
   e.g. [`cabal-install`](https://github.com/haskell/cabal/blob/c97092f5af484dfd5c1a465e0754114571264447/cabal-install/src/Distribution/Client/ProjectPlanning.hs#L1419-L1454)).

 - If the build tool is serialising data structures from its version of
   `Cabal[-hooks]` to the version against which the "hooks executable" including
   `SetupHooks.hs` is built, then the serialisation formats need to be
   compatible.  We could imagine using a serialisation format that is versioned
   and has some support for migrations (cf. `safecopy`) but this would lead to
   additional complexity.

 - If the build tool is dynamically loading `SetupHooks.hs`, ABI compatibility
   means both will need to use exactly the same `Cabal[-hooks]` library.

### Effects available to hooks

The proposal allows hooks to execute arbitrary IO effects. An alternative would
be to limit the available effects using some kind of DSL, and thereby allow the
possibility that the build tool could analyse the hooks or vary the
implementation of the permitted effects.

However, introducing a DSL would be a significant design contention point,
because it is far from clear how best to design it or what effects are required.
In general, the space of possible effects needed to augment the build system is
unbounded (e.g. imagine a package that needs to generate code by parsing some
binary file format, with a parser implemented using the FFI).

Thus, using a DSL would risk needing frequent updates to support additional use
cases for custom setup scripts that were not previously considered. Moreover, it
would increase the migration burden by requiring `Setup.hs` files to be
significantly rewritten to use the DSL instead of IO.

Thus we believe the best approach is for hooks to have access to arbitrary IO,
but to encourage package authors to use declarative features instead of hooks
where suitable features exist.

### Inputs and outputs available to hooks

There is a tension in how to design the hook inputs and outputs, with two
possible approaches:

 * Maximal: Start by making hooks "as powerful as possible", i.e. allowing the
   input and output of any given hook to be as large as possible. This allows
   users to use as much of the information from `Cabal` as possible, and to
   influence as much as the build process as possible.

 * Minimal: Start by making hooks "as weak as possible", i.e. design the input
   and output types to include the minimal information needed to address the use
   cases in the sample.

The maximal approach requires the build tool more closely match `Cabal`'s
(current) design, which may reduce flexibility to change `Cabal` in the future.
From a backwards compatibility perspective, it would be easier to start with a
minimal approach and later add new fields, rather than removing existing fields.

However, the maximal approach should reduce the likelihood of situations where
existing custom `Setup.hs` files cannot be migrated to hooks because they
require additional information.  Given that our goal is to remove the need for
`Custom` entirely (from the last few percent of packages), and those are
inevitably edge cases where the use cases are not always easy to predict, it is
important that we handle as many as possible (rather than covering common use
cases only).

Even though the minimal approach can be incrementally expanded, there would
still be a significant time lag between adding fields and them being supported
by widely used versions of `cabal-install` so they can be adopted by packages
that need them.  Thus the minimal approach risks further extending the migration
window during which `Custom` will need to be supported.

### Identifiers for fine-grained rules

Instead of using a monadic API to guarantee correctness of `ActionId`s, we could
imagine letting users manually create their own `ActionId`s, perhaps with
an existential type such as:

```haskell
data ActionId where
  ActionId :: (Typeable a, Ord a, Read a, Show a) => a -> ActionId
```

This design means that each hook author would be able to use a type that is
specific to them to identify `Action`s, ensuring the `ActionId`s can't clash
with those defined elsewhere.

This would allow getting rid of the monadic API for creation of fresh `ActionId`,
but would require additional thought about how we would deserialise such
`ActionId`s as required by IPC with the external hooks executable.  
This design would arguably be less robust to internal refactorings, as we end
up in a situation in which `ActionId`s are aware of their own names.

### No searching in fine-grained rules

As noted in [Pre-build hooks](#pre-build-hooks), rules are described at a
low-level with explicit inputs and outputs. For example, this framework does
not include "rule patterns" (generating `*.hs` from `*.y`).  
However, filepaths are not specified entirely explicitly: for example,
inputs are declared relative to a search path, and the build system will
perform the search and then pass the resolved locations of these dependencies.

An alternative design would be to perform all file searching logic inside
the pre-build hooks, in the computation of the rules. The `Cabal` library would
provide a convenient interface to do file searching that could be used by
hook authors. This would simplify the design by making it more uniform, and
it would eliminate all searching from the semantics of the rules. That is, the
rules framework would not itself contain any notion of searching; any necessary
searching would be implemented on top. In particular, rules would no longer need
separate datatypes for unresolved and resolved dependencies.  
This is the approach taken by the `ninja` build system, which is designed around
a low-level syntax of rules being generated by a higher-level framework.  

One possible drawback of such a design is that it risks baking into the hooks
API assumptions that `Cabal` and `cabal-install` currently make about the
directory structure of dependencies and outputs: the library interface for
searching would be provided by `Cabal` for use by hook authors, but one would
want to account for the needs of other build systems which might use a
completely way of organising files inputs and outputs.

### Making other hooks fine-grained

Note that we do not currently propose to use the same design of fine-grained
rules for other hooks, e.g. the hooks into the configure phase or the install
hooks.  
The downside of this choice is that we do not track fine-grained dependency
information that would let us know when to re-run these hooks.
However, it is not clear that there is much demand for it to do so. Thus it may
be sufficient to simply re-run these hooks pessimistically every time.

## Stakeholders

The ideas underlying this proposal have been under discussion at [Cabal issue
#9292](https://github.com/haskell/cabal/issues/9292).  We are grateful to all
those who have contributed to the discussion and pointed out areas of the design
needing improvement, and we welcome further feedback.

This proposal aims to provide more flexibility to the implementors of Haskell
build tools such as `cabal-install` and `stack`, as well as
non-language-specific build systems that may build Haskell code (such as Nix).
Thus we would appreciate input on the proposal from developers of build systems.

We would also appreciate input from authors of packages with custom `Setup.hs`
scripts which might be impacted by this proposal, to clarify their needs.  For
example, we have argued that `Cabal` should provide install hooks so that packages
currently using them as a distribution mechanism are able to continue doing so,
but we could reconsider this the impact on packages was considered acceptable.

GHC itself uses custom `Setup.hs` files for
[`ghc-boot`](https://gitlab.haskell.org/ghc/ghc/-/blob/master/libraries/ghc-boot/Setup.hs)
and
[`ghc-prim`](https://gitlab.haskell.org/ghc/ghc/-/blob/master/libraries/ghc-prim/Setup.hs),
so we should ensure that GHC's use cases are accounted for in the new design.


## Success

In the short term, we have funding to propose a design and implementation of the
new `Hooks` build-type in the `Cabal` library. Once this work is done, its
inclusion in Cabal will require a cycle of reviews and amendments, and general
consensus among Cabal developers for accepting it.

This proposal will provide an alternative and a migration path for existing uses
of the `Custom` build-type that cannot be migrated to the `Simple` build-type.
This migration is intended to take place over a long timescale to allow the
gradual migration of packages which use `Custom`.  While we intend that `Cabal`
will ultimately be able to remove support for `Custom` entirely (and hence the
legacy code paths that support it), this requires a well-established
alternative.

The new `Hooks` build-type can be implemented in the `Cabal` library in a way
that is backwards compatible for build tools using the `Setup.hs` interface
(e.g. distribution packages) or the `Cabal` build system (e.g. `cabal-install`,
`stack`).  Thus while build tools will need to use a new enough version of the
`Cabal` library, they should then support packages using `Hooks` without further
modification.


### Testing and migration

We have created the [`hooks-setup-testing`](https://gitlab.haskell.org/mpickering/hooks-setup-testing/)
repository which contains [patches to migrate various packages to use `build-type: Hooks` instead of
`build-type: Custom`](https://gitlab.haskell.org/mpickering/hooks-setup-testing/-/tree/master/patches).

This repository follows the same design as `head.hackage`, as it provides an overlay package repository,
but instead of enabling building against GHC HEAD it instead provides our patched version of `cabal-install`
and builds packages using our patched `Cabal` libraries.

This repository fulfills several goals:

* Design: by collecting all the patches in one place, we can be sure that the design of the hooks is
    robust enough to handle the common use-cases.
* Testing: CI in the repository ensures that our patches continue to build against
    migrated packages, as the design of the `Hooks` interface evolves.
* Overlay: users can use the repository as an overlay like `head.hackage`, allowing them to
    experiment with the new `Hooks` design locally. The overlay will also allow us to test new features of `cabal-install` which rely on
    having migrated packages away from `build-type: Custom`.
* Migration: the patches can later be upstreamed to help library authors migrate their own packages
    away from `Custom`.


### Future work

In the future, it will be desirable for Haskell build tools like `cabal-install`
to avoid using the `./Setup` CLI interface at all.

* There is no need to perform the full configuration phase for each package, because `cabal-install` has already
  decided about the global and local configuration.
* This will allow finer grained build plans, because we don't have to rely
  on `./Setup.hs build` in order to build a package. Instead,
  `cabal-install` could build each module one at a time.
* It will allow `cabal-install` and other tools to use the `Cabal` library
  interface directly to manage the build process, which is a richer and more
  flexible interface than what can be provided by a CLI.
* `cabal-install` will be able to use per-component builds for all packages,
  where currently it must fall back to per-package builds for packages
  using `build-type: Custom`. This will reduce the number of different code
  paths and simplify maintenance.

We do not currently have guaranteed funding to implement improvements for
`cabal-install`, but intend to seek this in the future.
