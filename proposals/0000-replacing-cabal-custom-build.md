# RFC: Replacing the Cabal Custom build-type

Adam Gundry, Matthew Pickering, Sam Derbyshire, Rodrigo Mesquita, Duncan Coutts (Well-Typed LLP)

-   [Abstract](#abstract)
-   [Background](#background)
    -   [The current interface](#the-current-interface)
    -   [Why do packages use the `Custom` build-type?](#why-do-packages-use-the-custom-build-type)
-   [Problem Statement](#problem-statement)
    -   [How can we move away from the `Custom` build-type?](#how-can-we-move-away-from-the-custom-build-type)
    -   [Requirements](#requirements)
    -   [Non-requirements](#non-requirements)
-   [Prior art and related efforts](#prior-art-and-related-efforts)
    -   [Issues with `UserHooks`](#issues-with-userhooks)
    -   [`code-generators`](#code-generators)
-   [High-level design of `build-type: Hooks`](#high-level-design-of-build-type-hooks)
    -   [`Hooks` from the package author's perspective](#hooks-from-the-package-authors-perspective)
    -   [`Hooks` from the build tool author's perspective](#hooks-from-the-build-tool-authors-perspective)
    -   [Designing for future compatibility](#designing-for-future-compatibility)
-   [Detailed design of `SetupHooks`](#detailed-design-of-setuphooks)
    -   [Cabal configuration type hierarchy](#cabal-configuration-type-hierarchy)
    -   [Hook phases](#hook-phases)
    -   [Configure hooks](#configure-hooks)
    -   [Build hooks](#build-hooks)
    -   [Copy hooks](#copy-hooks)
    -   [Test and benchmark hooks](#test-and-benchmark-hooks)
    -   [Clean hooks](#clean-hooks)
    -   [Pre-processors](#pre-processors)
-   [Open Questions](#open-questions)
    -   [Library API and versioning](#library-api-and-versioning)
    -   [Recompilation avoidance](#recompilation-avoidance)
-   [Examples](#examples)
-   [Alternatives](#alternatives)
    -   [Effects available to hooks](#effects-available-to-hooks)
    -   [Inputs and outputs available to hooks](#inputs-and-outputs-available-to-hooks)
    -   [Fine-grained build rules](#fine-grained-build-rules)
-   [Stakeholders](#stakeholders)
-   [Success](#success)
    -   [Testing and migration](#testing-and-migration)
    -   [Future work](#future-work)

## Abstract

Every Cabal package can supply its own build system, in the form of a `Setup.hs` program which implements a common command line interface.
Unfortunately, this makes it difficult to implement features in Cabal which require the build tool to have complete control over the build system.
This turned out to be a major architectural design flaw because, in practice, all packages use Cabal as their build system -- making the per-package build system abstraction an artificial limitation.
We propose a way forward to lift this restriction, which will establish foundations for improvements in tooling based on Cabal, and make Cabal easier to maintain in the long term.

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
   build system rather than each package is in overall control of the build,
   so it is not possible to entirely override any of the main build phases.

 * It should support essentially all of the existing uses of the
   `Custom` build-type.

 * It should minimise the effort required from package maintainers.  While some
   work to migrate away from the `Custom` build-type is inherently necessary, we
   need the migration path to be as straightforward as possible.

 * It should permit existing build systems such as RPM/DEB distribution
   packagers, Nix packages, etc. to continue using the `Setup.hs` interface
   unchanged.

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


### Non-requirements

Although it is not possible or practical to address too many problems at once,
it is a goal to make the new API more evolvable than the old `UserHooks`
API. Thus we believe that it will be possible to adapt the design more easily to
future features and requirements.

For example, `Cabal` does not currently have proper support for
cross-compilation, because it does not make a clear distinction between the host
and target.  This means the `Custom` build-type currently leads to issues with
cross-compilation, and in the first instance, the new design may inherit the
same limitations.  This is a bigger cross-cutting issue that needs its own
analysis and design.


## Prior art and related efforts

The `Cabal` developers have long been aware of the limitations arising from
`build-type: Custom` (see for example [#2395 Proposal for a Cabal plugin API
(inversion of control)](https://github.com/haskell/cabal/issues/2395) and [#3065
Lessons learned from Custom](https://github.com/haskell/cabal/issues/3065)).
Over time there have been attempts to gradually move packages away from
`Custom`, in some cases by adding declarative features to `build-type: Simple`
instead, which is preferable where possible.

Since the remaining "long tail" of packages have varied needs, however, we believe it is
better to design a more general mechanism for augmenting the process rather than
many specific knobs, so that integrating the new mechanism into Cabal and other
build-systems is more straightforward and general.  We presume that any existing
usage of `Custom` that merely augments the build process is justified and valid,
and seek to provide an alternative build-type to which existing packages can be
directly migrated.
The introduction of the alternative build-type that captures existing `Custom` extensions does not preclude the addition of declarative features that subsume use cases for it.

### Issues with `UserHooks`

As noted, the `Cabal` library already provides a customisable build system in
the form of the `defaultMainWithHooks` function and the
[`UserHooks` datatype](https://hackage.haskell.org/package/Cabal-3.10.1.0/docs/Distribution-Simple-UserHooks.html).
Thus it would be possible to define a build-type based on the package author
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

We propose that `Cabal` is augmented with a new build-type, `Hooks`.
To implement a package with the `Hooks` build-type, the user needs to provide
a `SetupHooks.hs` file which specifies the hooks using a Haskell API.

The `Hooks` build-type represents a middle-ground of customisation which specifically
only permits *augmenting* the build process at specific points, while disallowing
the complete replacement of individual build phases.


### `Hooks` from the package author's perspective

When `build-type: Hooks` is specified in the `.cabal` file,
the package author must supply a Haskell file named `SetupHooks.hs` that defines
a value `setupHooks :: SetupHooks`, which is a record of user-specified
hooks. Thus this interface is fundamentally a Haskell library interface, not
a command line interface as with the classic `Setup.hs`.

A hook is a Haskell function, with a type such as `HookInputs -> IO HookOutputs`
where `HookInputs` and `HookOutputs` are types specific to the particular hook.
See [§ Effects available to hooks](#effects-available-to-hooks) for discussion
of the choice of the `IO` monad.

Each hook is optional, i.e. each field of the `SetupHooks` datatype has a type
of the form `Maybe (HookInputs -> IO HookOutputs)`.  This means that the build
tool can statically determine that there is no hook at that particular stage.
At each stage, it is possible to specify a component-level pre-stage and/or a
post-stage hook, uniformly covering the various points at which authors might
want to run custom actions or customize the build.

See [§ Library API and versioning](#library-api-and-versioning) regarding which
Haskell library defines the `SetupHooks` datatype and any types describing the
inputs and outputs of particular hooks.

A `.cabal` file using `build-type: Hooks` must include a `custom-setup` stanza
with a `setup-depends` field describing the build dependencies of
`SetupHooks.hs` (just as with the `Custom` build-type).


### `Hooks` from the build tool author's perspective

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
import Distribution.Simple (  defaultMainWithSetupHooks )
import SetupHooks ( setupHooks )

main = defaultMainWithSetupHooks setupHooks
```

The build tool can compile the shim `Setup.hs` and run the traditional CLI
commands such as `./Setup configure` and `./Setup build`.  This does not realise
the full benefits of the new build-type, but is easy to integrate into existing
tools, because it is consistent with how the other build-types already support
the `Setup.hs` interface.

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
separately however the build tool arranges to do so. This restriction prevents
the hooks directly sharing state with each other: all the inputs and outputs
must be serialised.


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


## Detailed design of `SetupHooks`

Having described `build-type: Hooks` in the previous section, the remaining part
of the design process is to work out the specific interfaces for the individual
hooks as expressed by the `SetupHooks` type.

We want to arrive at a design by two means:

* The consideration of the existing usage of `Setup.hs` scripts to guide
  what hooks should be able to do.
* A high-level understanding of what the build process of a package should
  look like, taking into account concerns such as parallelisability.

These two viewpoints can inform each other about the precise details for the design.

As part of the design process we have been developing a [prototype
implementation of this design](https://github.com/mpickering/cabal/tree/wip/setup-hooks)
in the `Cabal` library.

This section unavoidably relies on a deeper understanding of the `Cabal` build
system than the previous sections.


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


### Hook phases

The `Cabal` build process defines various phases, which correspond to the fields
of the `SetupHooks` data type:

```haskell
data SetupHooks = SetupHooks
  { configureHooks :: ConfigureHooks
  , buildHooks     :: BuildHooks
  , copyHooks      :: CopyHooks
  , cleanHooks     :: CleanHooks
  , testHooks      :: TestHooks
  , benchmarkHooks :: BenchmarkHooks
  , hookedPreProcessors :: [PPSuffixHandler]
  }
```

These phases are as follows:

 * The *configure phase* is when decisions are made about how to build the
   subsequent phases (e.g. which tools and options to use).  This
   may involve running arbitrary code to detect information about the host
   system.  See [§ Configure hooks](#configure-hooks).

 * The *build phase* is when the project is compiled (including for the REPL)
   and build artifacts are generated (including libraries, executables and other
   artifacts such as Haddock documentation).  See [§ Build hooks](#build-hooks).

 * The *copy phase* is when build artifacts are moved from the build directory
   to the final installed location.  See [§ Copy hooks](#copy-hooks).

 * The *test phase* and *benchmark phase* are when test or benchmark components
   for the package are executed.  See [§ Test and benchmark hooks](#test-and-benchmark-hooks).

 * The *clean phase* is when files created during the build are deleted.
   See [§ Clean hooks](#clean-hooks).

See [§ Pre-processors](#pre-processors) for discussion of `hookedPreProcessors`.

We augment these phases uniformly with pre-hooks that execute before the main
`Cabal` build action, and post-hooks that execute afterwards.  Unlike the old
`UserHooks` datatype, there is deliberately no way for the package to remove or
replace the normal build action.

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

Hooks are not able to add entirely new phases, because which phases are
available is determined by the overall design of the build system, not the
individual package.


### Configure hooks

We propose to add the following configure hooks:

```haskell
type PreConfPackageHook    = PreConfPackageInputs -> IO PreConfPackageOutputs
type PostConfPackageHook   = PostConfPackageInputs -> IO ()
type PreConfComponentHook  = PreConfComponentInputs -> IO PreConfComponentOutputs
type PostConfComponentHook = PostConfComponentInputs -> IO ()

data PreConfPackageInputs
  = PreConfPackageInputs
  { configFlags      :: ConfigFlags
  , localBuildConfig :: LocalBuildConfig
  , compiler         :: Compiler
  , platform         :: Platform
  }

data PreConfPackageOutputs
  = PreConfPackageOutputs
  { localBuildConfig :: LocalBuildConfig }

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

data PostConfComponentInputs
  = PostConfComponentInputs
  { localBuildInfo :: LocalBuildInfo
  , component :: Component
  }

data ConfigureHooks
  = ConfigureHooks
  { preConfPackageHook    :: Maybe PreConfPackageHook
  , postConfPackageHook   :: Maybe PostConfPackageHook
  , preConfComponentHook  :: Maybe PreConfComponentHook
  , postConfComponentHook :: Maybe PostConfComponentHook
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
  the global configuration.

After the global configuration has completed, individual components can be
configured independently, as follows:

- Run the `preConfComponentHook`. This is the only means to apply specific
  options to a `Component`.

- Use the modified `Component` to perform per-component configuration and create
  the `ComponentLocalBuildInfo`.

- Run the `postConfComponentHook`, which can inspect but not modify the
  per-component configuration.


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
This is why the "post" configuration hooks (and any hooks after that)
can only run an `IO` action; they can't return any modifications that would affect the `PackageDescription`.


#### `LocalBuildConfig`

There are parts of the `LocalBuildInfo` which must be decided at a global (per-package) level.
For instance, whether to build dynamic libraries.
On the other hand, there are also things we want to decide on a local (per-component) level,
such as specific GHC options with which to compile the component.

Moreover, there are parts of the `LocalBuildInfo` which hooks cannot modify.
For example, things such as package dependencies can't be modified because they are
determined externally by the overall build plan (e.g. from the dependency solver).
Thus the hooks interface should prevent the modification of these
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

An alternative design would be to define a custom diff datatype which statically
distinguishes which fields are allowed to be modified. This would be more
difficult to maintain, as it requires `Cabal` developers to keep the `Component`
and `ComponentDiff` types up-to-date.  However if we end up with a design in
which `Cabal`'s version of the `Component` type is necessarily separate from the
type in the hooks API, we may want to reconsider this.


### Build hooks

The build hooks are intended to perform additional steps before or after
the normal build phase for a component. These steps cannot change
the configuration of the package; they can only perform side-effects.
The intention is that hooks should separate any cheap configuration
from expensive building.

One crucial observation that factors into the design of build hooks is that
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

data BuildComponentInputs
  = BuildComponentInputs
  { buildingWhat   :: BuildingWhat
  , localBuildInfo :: LocalBuildInfo
  , targetInfo     :: TargetInfo
  }

type BuildComponentHook = BuildComponentInputs -> IO ()

data BuildHooks
  = BuildHooks
  { preBuildComponentHook  :: Maybe BuildComponentHook
  , postBuildComponentHook :: Maybe BuildComponentHook
  }
```

This design ensures that the build hooks are consistently run in all
build-like phases. This reflects the common pattern in custom Setup scripts
that one would update the `haddock` and `repl` hooks to mirror the `build` hooks
(which one can easily forget to do, and end up with an unusable `repl`, for example, see `singletons-base`
which fails to update the `replHook`).

The build tool executes the build phase as follows:

- Run the `preBuildComponentHook`, giving it access to the specific information
  about the component being built.  This cannot modify any configuration but may
  have side effects.

- Build the component.

- Run the `postBuildComponentHook`.

A typical use case for the `preBuildComponentHook` is generating source code for
modules.

The `postBuildComponentHook` can be used to do further work after building the
component. For example, in Agda, once the `agda` executable is built a
post-build hook could run it to compile the builtin libraries and generate
`.agdai` interface files for them.

There are deliberately no package-level pre-build or post-build hooks, only
component-level hooks. This avoids introducing unnecessary synchronisation
points when multiple packages/components are being built in parallel.


### Copy hooks

The `copy` hooks run before and after the copy phase, which moves build artifacts
from the build directory into the install directory.

```haskell
data CopyComponentInputs
  = CopyComponentInputs
  { localBuildInfo :: LocalBuildInfo
  , copyFlags      :: CopyFlags
  , targetInfo     :: TargetInfo
  }

type CopyComponentHook = CopyComponentInputs -> IO ()

data CopyHooks
  = CopyHooks
  { preCopyComponentHook  :: Maybe CopyComponentHook
  , postCopyComponentHook :: Maybe CopyComponentHook
  }
```

The copy hooks can be used to copy files per-component.
The main use case for copy hooks is when set of things you want to install is
not fixed and predetermined. For example, in the Agda
example, if `postBuildComponentHook` has generated `.agdai` interface files then
`preCopyComponentHook` can be used to copy over the generated files when
installing the compiler.

We could imagine a more declarative way of specifying this being introduced in
the future, in which case packages will be free to migrate to it gradually.
There is not necessarily a problem with having some overlap between hooks and
declarative features.

An alternative approach would be to regard as illegitimate any use cases which
treat `Cabal` as a packaging and distribution mechanism for executables, and on
that basis, cease to provide copy hooks.  We do not follow this approach because
it would significantly inconvenience maintainers of packages that rely on this
behaviour (e.g. Agda and Darcs), for a relatively small reduction in complexity
in `Cabal`.

It is important that these copy hooks are also run when installing, as this
fixes the inconsistency noted in [Cabal issue #709](https://github.com/haskell/cabal/issues/709).
There is no separate notion of an "install hook", because "copy" and "install"
are not distinct build phases.


### Test and benchmark hooks

Test and benchmark hooks are executed before or after the tests or benchmarks
are compiled and run.  The main use cases for pre-test or pre-benchmark hooks
are generating modules for use in tests or benchmarks; while technically this
could be done in an earlier phase, if the generation step is expensive it makes
sense to defer running it until the user actually requests tests or benchmarks.

Another possible use cases is doctests, where:

 * the package author wants `cabal test` to run doctests (so an external `cabal
   doctest` command is not enough), e.g. because they use a different build tool
   and need `./Setup test` to include doctests; and

 * the `code-generators` feature is not suitable (e.g. because the package
   includes multiple library components with incompatible GHC options).

There are deliberately no package-level test or benchmark hooks, only
component-level hooks. This is consistent with build hooks, and avoids
introducing unnecessary synchronisation points if multiple packages are being
tested in parallel.

```haskell
data TestComponentInputs
  = TestComponentInputs
  { additionalCommandLineArgs :: [String]
  , localBuildInfo :: LocalBuildInfo
  , testFlags :: TestFlags
  , testSuite :: TestSuite
  , componentLocalBuildInfo :: ComponentLocalBuildInfo
  }

data BenchmarkComponentInputs
  = BenchmarkComponentInputs
  { additionalCommandLineArgs :: [String]
  , localBuildInfo :: LocalBuildInfo
  , benchmarkFlags :: BenchmarkFlags
  , benchmark :: Benchmark
  , componentLocalBuildInfo :: ComponentLocalBuildInfo
  }

type TestComponentHook      = TestComponentInputs      -> IO ()
type BenchmarkComponentHook = BenchmarkComponentInputs -> IO ()

data TestHooks
  = TestHooks
  { preTestComponentHook  :: Maybe TestComponentHook
  , postTestComponentHook :: Maybe TestComponentHook
  }

data BenchmarkHooks
  = BenchmarkHooks
  { preBenchComponentHook  :: Maybe BenchmarkComponentHook
  , postBenchComponentHook :: Maybe BenchmarkComponentHook
  }
```


### Clean hooks

Clean hooks are useful if an earlier hook has generated some additional files
that need to be cleaned up when the user runs the build system's clean action.
For example, the
`Setup.hs` files [`lhs2tex-1.25`](https://hackage.haskell.org/package/lhs2tex-1.25/src/Setup.hs)
and [`binembed-0.1.0.2`](https://hackage.haskell.org/package/binembed-0.1.0.2/docs/src/Distribution-Simple-BinEmbed.html#binembedClean)
implement versions of this.

`cabal clean` does not currently execute `./Setup.hs clean`, which appears to be a bug
(see [#6112](https://github.com/haskell/cabal/issues/6112), [#7877](https://github.com/haskell/cabal/issues/7877)).
In the new design we can ensure that each hook is executed before and after the
corresponding build phase, and hence avoid such bugs.

```haskell
data CleanComponentInputs
  = CleanComponentInputs
  { packageDescription :: PackageDescription
  , component :: Component
  , cleanFlags :: CleanFlags
  } deriving (Generic, Show)

type CleanComponentHook = CleanComponentInputs -> IO ()

data CleanHooks
  = CleanHooks
  { cleanComponentHook :: Maybe CleanComponentHook
  }
```


### Pre-processors

`UserHooks` includes `hookedPreProcessors`, which makes it possible to associate
a file extension with a preprocessing operation that turns files with that
extension into Haskell source modules.  Cabal will then execute such pre-processors
on all matching files before building, and re-run them as needed when the input files change.
This is similar to Cabal's built-in support for `alex`, `happy`, etc..  For example,
[`binembed`](https://hackage.haskell.org/package/binembed-0.1.0.3/docs/src/Distribution-Simple-BinEmbed.html#withBinEmbed)
defines a preprocessor that turns `M.binembed` into `M.hs` by running the
`binembed` executable.

A key restriction of `hookedPreProcessors` is that it assumes one input file
will turn into one Haskell module (see [#6232 Extended preprocessors
support](https://github.com/haskell/cabal/issues/6232)).

Packages requiring pre-processors could define a pre-build hook that searches
for files with their custom extension and produces the corresponding Haskell
file.  However this would require duplicating logic present in `Cabal`, and lead
to worse recompilation behaviour.  Thus we propose to retain
`hookedPreProcessors` essentially unchanged in the `SetupHooks` design.



## Open Questions

### Library API and versioning

There are a few options for where the `SetupHooks` datatype and any types
describing the inputs and outputs of particular hooks will be defined:

 * They could all be defined in `Cabal` itself, just like for
   `UserHooks`. (Potentially, a `Cabal-hooks` library could re-export just the
   definitions that are relevant for programs using the hooks API.)

 * They could be defined in a new `Cabal-hooks` library, with `Cabal` depending
   on it and reusing the same types. This has the advantage of more clearly
   defining the subset of the `Cabal` library that is exposed via the hooks API,
   which is helpful for modularity. However it is likely to involve moving quite
   a lot of code to the new library.

 * Separate hook input/output data types could be defined in a `Cabal-hooks`
   library, then `Cabal` could contain code for translating values of those
   datatypes into its internal representations (and vice versa).  This would
   require significant duplication of datatypes, but might provide more options
   for evolving the `Cabal` and `Cabal-hooks` datatypes independently.

A related question is whether the hooks API should be versioned separately from
`Cabal` itself.  Version compatibility concerns may arise because the build tool
and `SetupHooks.hs` may use different versions of the `Cabal` library. The
impact depends on how the build tool is using `SetupHooks.hs`:

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

It is important to be clear what future compatibility guarantees (if any) are
offered by `Cabal` and/or build tools to packages using `Hooks`.  There is a
tension here, because we do not want package maintainers to be over-burdened
with continual changes to support newer `Cabal` versions, but neither do we want
`Cabal` maintainers to be over-burdened by the costs of providing backwards
compatibility.

Since `Hooks` allows the build system to be extended, arguably packages using
this feature necessarily depend on the version of the build system (and hence
the `Cabal` version). Thus it may be reasonable to expect package authors to
declare the range of `Cabal` versions for which their package works (in
`setup-depends`) and require it to be updated to support newer versions of build
tools.

While there are still questions to be resolved here about the compatibility
guarantees that can be offered by build tools, the `Hooks` build-type makes the
situation no worse than with `Custom` (because a shim `Setup.hs` can still
always be used to compile `SetupHooks.hs` using an older `Cabal`), but gives
significantly more options to the build tool.  Where serialisation of hook
inputs/outputs is used, the serialisation format can be controlled by the build
tool, and is not necessarily fixed by `Cabal`.  Indeed, we could imagine a build
tool being compiled against multiple `Cabal` versions.

Note that the version compatibility problem exists in `cabal-install` already:
even where communication happens via the `Setup.hs` command line interface,
there is already a need for `cabal-install` to adapt to the command-line flags
that are supported by the version of `Cabal` in use (see
e.g. [`filterConfigureFlags`](https://hackage.haskell.org/package/cabal-install-3.10.2.1/docs/src/Distribution.Client.Setup.html#filterConfigureFlags)).
Once this proposal removes the need for `cabal-install` to go through the
`Setup.hs` interface, there is a potential for significant reduction in
complexity here.


### Recompilation avoidance

We could imagine allowing hooks to return dependency information to support
Cabal's recompilation checker in deciding whether it needs to rerun the hooks or
not. This is particularly relevant when building documentation, as ideally one
doesn't want to have to run the hooks twice in the workflow `cabal build &&
cabal haddock`.

A natural way to do this that fits with Cabal's current design would be to have
a hook that returns the files consulted by each phase, then rerun the phase only
if those files change.  For example, alongside
```haskell
preConfPackageHook :: Maybe (PreConfPackageInputs -> IO PreConfPackageOutputs)
```
we could have
```haskell
confPackageRecompile :: Maybe (PreConfPackageInputs -> IO Dependencies)
```
where `Dependencies` is some representation of possible dependencies
(e.g. files, file path globs, environment variables, etc.) that may be consulted
by `preConfPackageHook` and should cause recompilation if they change.

`UserHooks` does not currently support this, and it is not clear that there is
much demand for it to do so.  Thus it may be sufficient to simply rerun hooks
pessimistically.


## Examples

These examples are based on the [survey of packages using custom `./Setup.hs`
scripts](https://github.com/well-typed/hooks-build-type/blob/main/survey.md) we
undertook previously, and experimental patches we have created while testing the design
(see [§ Testing and migration](#testing-and-migration)).

#### Generating modules

One of the main uses of `Setup.hs` scripts is to generate modules.

To achieve this with `SetupHooks`, in the per-component configure hook
`preConfComponentHook`, the package should declare that it will generate certain modules.

These modules can be generated either in this configure hook itself,
or in the (per-component) pre-build hook. It depends on what kinds of modules
are being generated and how they are being generated as to which method is suitable.
One reason to choose to generate in the pre-build phase might be that the entire
configuration information is available at that point.

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

The `hookedPrograms :: [Program]` field of `UserHooks` allows the custom
`Setup.hs` file to specify new programs to be detected in the configure phase.
In the `SetupHooks` design this use case is supported by having the
`LocalBuildConfig` returned by the package-level pre-configure hook contain a
field `withPrograms :: ProgramDb`, which can be extended by the hook.


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

In general the monoid is non-commutative, because for the
pre-configure hooks the output of the first hook will be fed as an input to the
second.


## Alternatives

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

Thus using a DSL would risk needing frequent updates to support additional use
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
From a backwards compatibility perspective it would be easier to start with a
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

Ultimately, we take the view that once the build system supports a hooks
mechanism at all (e.g. for cases like pre-configure, where it is clearly
needed), there is little cost to uniformly extending the mechanism to cover all
the major build phases.


### Fine-grained build rules

The hooks proposed here are still comparatively coarse-grained. While they allow
packages to execute custom code before or after the Cabal build, they do not
allow the expression of declarative build rules that describe dependencies at
the level of individual files.

For example, suppose Cabal did not have built-in support for `happy`, then a
package making use of it might like to write a rule like this:

```
lib:my-component:module:Foo.Bar : src:blah/foo/bar.y
    ${happy:exe:happy} -o ${output} ${input[0]}
```

The build tool would then be given such a description, and could combine it with
its own rules to create a fine-grained build dependency graph, which could lead
to advantages including better parallelism and more precise recompilation
detection.

This is an appealing goal; however, attaining it would require significant work:

 * There is a large design space of possible declarative systems for expressing
   build rules, and the trade-offs are complex, so it is not clear we would
   reach consensus about a design in the short term.

 * Migrating existing custom `Setup.hs` scripts would be significantly more
   difficult, because of the need to rewrite existing imperative logic.

 * The current implementation of the Cabal build system is not designed in a way
   that would be able to take advantage of such fine-grained rules, and
   refactoring it to do so would be a major undertaking.

Since packages using the custom build-type are relatively uncommon, we do not
believe this work is currently justified.
Thus we do not propose such a design at present.  However, the proposed design
is a step towards making fine-grained rules expressible in the future:

 * It establishes the principle that the build system, not the individual
   package, has control of the overall build process.

 * The hooks proposed here can be extended or adapted in the future.  Thus it
   would be possible to add hooks in the future that express declarative build
   rules in a suitable DSL, and packages could gradually opt in to the
   declarative approach.


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
example, we have argued that `Cabal` should provide copy hooks so that packages
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
