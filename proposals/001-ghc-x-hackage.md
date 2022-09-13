# Adapting to a new version of GHC, way faster

_Michael Peyton Jones, Ben Gamari, Simon Peyton Jones, Théophile "Hécate" Choutri, David Thrane Christiansen_


This proposal describes a way to make it faster and easier to migrate Haskell code with long transitive dependency chains to a new GHC version using an unofficial community repository of compatibility patches.

## 1. Motivation

At the moment we have a problem with propagating support for a new version of GHC through the Haskell ecosystem. Suppose package `A` depends on `B` which depends on `C`. Then the process goes like this

1. GHC version X is released.
2. The maintainer of `C` must wake up, fix their package, and make a Hackage release. The maintainers of `A` and `B` can do nothing at this point.
3. The maintainer of `B` must wake up, fix their package, and make a Hackage release. The maintainer of `A` can still do nothing.
4. The maintainer of `A` can finally fix their package and make a Hackage release.

This process is slow for a number of reasons:

- It is utterly serial. The maintainer of `B` cannot lift a finger until the maintainer of `C` has not only fixed package `C`, but also uploaded a new release to Hackage.
- Each step has multiple serial parts: often the maintainer will merge a patch (perhaps in response to prompting) but _not_ do a release, blocking further progress.
- If a maintainer is unavailable for any reason, the entire dependency tree of that package is blocked. In an ecosystem with hundreds of widely used packages, the chances of every single maintainer being available in a timely fashion are close to zero.
- It is not clear to even a willing and available maintainer _when_ they need to wake up and do some work. Often it is up to a motivated individual (say the maintainer of an application that uses `A`) to sequentially bug the maintainers of `C` and `B` and `A` in sequence so they know that they are able to do something.

**The result of this is that it usually takes months or even years for support for a GHC version to percolate through the ecosystem.** For example, GHC 9.0.1 was released in February 2021, but as of December 2021 `haskell-language-server` still does not support it fully, largely because of difficulty updating dependencies.

We could enormously speed up the process of updating the package ecosystem for a new version of GHC if

- We could parallelise the process.
- We could allow many people to contribute to the routine updates necessary to adapt a package to changes in its dependencies.
- We could allow package maintainers to do their work when it was convenient to them, without holding up the whole train

We propose to improve the situation by widespread usage of a Hackage overlay (see Section 2), GHC._X_.hackage, populated with **non-maintainer-contributed patches** for the purpose of unblocking package maintainers in adapting their libraries to new GHC releases.

## 2. Background

### 2.1. Additional package repositories and Hackage overlays

Cabal [supports](https://cabal.readthedocs.io/en/latest/cabal-project.html#cfg-field-active-repositories) _additional package repositories_ beyond Hackage. These can be configured in the Cabal config file, or in a `cabal.project` file.

Users can state which repositories they want to be _active_, and how they should be consulted. This allows additional repositories to override, or merely augment Hackage.

  
Overriding Hackage can be accomplished by listing the repositories in a Cabal project file like so:
```
active-repositories: hackage.haskell.org, X
```
This will cause repository `X` to be consulted first for any given version of a package, but allow other versions to come from Hackage. For example, if `X` contains `A-1` and `A-2`, and Hackage contains `A-2` and `A-3`, then `A-1` and `A-2` will be taken from `X`, but `A-3` can be taken from Hackage.

We refer to this style of additional repository as a **Hackage overlay**, since it provides a way to _override_ specific versions of packages which appear in Hackage, while using Hackage for everything else.

Additional package repositories behave well in the face of changes. Cabal will recompute the build plan whenever new versions are added to any repository.


### 2.2. head.hackage

[head.hackage](https://gitlab.haskell.org/ghc/head.hackage/) is an example of a Hackage overlay. It is built from the linked source repository which consists of a set of patches to released Hackage versions of packages. head.hackage contains patches that enable packages to build with GHC `HEAD` (hence the name), and is used by the GHC developers to test upcoming GHC releases.

While `head.hackage` has been an invaluable tool for testing GHC, it has a few
characteristics that have made it harder for end-users to consume.
Specifically, its lack of versioning and mutable nature makes it hard to rely
on since changes in GHC, head.hackage, and Hackage itself can all break a
previously-building project.

### 2.3. Local package overrides

Both Cabal and Stack support overriding packages with external versions. In a `cabal.project` file, this is accomplished using the [`source-repository-package`](https://cabal.readthedocs.io/en/3.4/cabal-project.html#specifying-packages-from-remote-version-control-locations) stanza, and the [`extra-deps`](https://docs.haskellstack.org/en/stable/yaml_configuration/#extra-deps) field of a `stack.yaml` file supports Git and Mercurial references in addition to package versions from Hackage. A common practice in the Haskell community today is to either patch dependencies or find existing unreleased or even unmerged packages, and then add references to them to the build script. Many busy developers copy-paste snippets of package overrides in an attempt to get their package to build.

This practice can help un-stick the migration process described in the introduction. It empowers developers who are further down the dependency chain to adopt new compilers, and it gives library authors in the middle of the chain the chance to parallelize the work of adapting to a new library ecosystem, even though there is some risk that the provisionally-adopted patch may not be entirely compatible with the eventual Hackage release.

In essence, projects today are creating something very much like a private overlay for their own purposes. There are a few differences:
 - Developers must monitor their private overrides to check that new versions of dependencies that support new versions of GHC are not available, and manually migrate to them (e.g. by deleting the override or by specifying it as a Hackage version rather than as a Git reference).
 - Developers are in full control of their patches. This is a double-edged sword, as they have the opportunity to check each dependency, rather than relying on the good faith of overlay contributors, but they must also maintain the specific set of patches over time, including ensuring that their local overrides are compatible with dependencies drawn from Hackage.

Local package overrides are not formally coordinated, however. Each developer is left to their own device, and knowledge of how to work around missing updates is spread through unreliable informal channels. Much of the work that is done to identify patches and override versions is repeated by many developers, rather than being performed in a single central location from which everyone can benefit.



## 3. Proposal

We propose to extend `head.hackage` for wider use beyond GHC developers. In particular, our goal is to make `head.hackage` more useful for library authors and end-users testing GHC pre-releases.

### 3.1. GHC._X_.hackage

#### 3.1.1. The GHC._X_.hackage package repository

For each released major version _X_ of GHC, a Hackage overlay is provided called GHC._X_.hackage. The intention of GHC._X_.hackage is to override versions of packages that require changes to build with GHC-_X_. For instance, `GHC.9.6.hackage` would be an overlay that contains packages that have been updated for GHC 9.6.

GHC._X_.hackage is a _non-authoritative_ set of patched packages and its goal is primarily to unblock package maintainers. It should not be used in production, and any faults are not to be laid at the door of the package maintainers.

Since GHC._X_.hackage is opt-in, users who do not use GHC._X_.hackage will not be able to use the packages which have been fixed in GHC._X_.hackage, until fixed versions of those package and all their dependencies are actually released on Hackage; however, we hope that by allowing Haskell packages to be made compatible with new compiler releases more quickly, we can help speed this process.

Under this plan, today's `head.hackage` repository becomes a pre-release version of GHC._X_.hackage, and as such will change to behave in the same way as GHC._X_.hackage.

When the release branch for GHC version _X_ is created, the current state of the `head.hackage` repository is branched to create GHC._X_.hackage. This branch will accept merge requests that add overridden packages for GHC _X_ in Hackage packages.

GHC._X_.hackage repositories continue to exist for old versions of GHC in perpetuity. They are not retired, although it is expected that they will become irrelevant in time as proper releases are made in Hackage.


#### 3.1.2. Adding patches to GHC._X_.hackage

The source of GHC._X_.hackage is held in a Git repository, and accepts overridden versions from community members _who need not be the package maintainer_. For example, the maintainer of package `A` might submit overridden versions of `B` and `C`. 

The inclusion criteria for a patch are:

1. The patched package should build with GHC _X_.
2. The patch should represent a patch-version level change according to the PVP, in particular it should not change the package’s API.
3. The patch should be plausibly acceptable by the upstream maintainer.
4. The patch should have been submitted to the upstream maintainer in a PR or MR, and this submission should be documented in the submission to GHC._X_.Hackage.

Criterion (3) is deliberately vague, but suggests that the tests should pass (although this will likely not be easy to verify given the diversity of packages), that the patch should not needlessly break compatibility with previous versions of GHC, etc. Of course, since in most cases the patch will _not_have been vetted by the maintainer, precise correctness cannot be guaranteed.

Criterion (4) is intended to help minimize divergence from Hackage over time.

Package maintainers, or others, can update packages even _before_ GHC _X_ is released, by adding patches to `head.hackage`. After the release, those patches will be part of GHC._X_.hackage. The same rules for patch inclusion apply to `head.hackage` before it becomes GHC._X_.hackage.


#### 3.1.3. Removing patches from GHC._X_.hackage

Patches should not _need_ to be removed from GHC._X_.hackage. They will simply become obsolete as Hackage becomes more and more up-to-date.

However, in any case if a package maintainer requests the removal of a patch from GHC._X_.hackage (for example, it might be outright wrong) then it should certainly be removed.


### 3.2. Versioning and releases

While GHC._X_.hackage overrides particular versions of packages from Hackage, those overrides will quickly become obsolete when a new version of the package is released to Hackage.

Suppose that GHC._X_.hackage contains a patched version `X` of package `A`. When `A`’s maintainer next releases `A`, they will release `A-Y`, where `Y > X`. Users who use GHC._X_.hackage will in most cases therefore find that Cabal picks build plans with the new `A-Y` instead of the old (patched) `A-X`. However, if they have constraints that prevent this (say, because `A-Y` is a major version bump), then they may continue using the GHC._X_.hackage version.

It would be nice to argue that this case is unlikely: since by assumption we only allow patch-level changes into GHC._X_.hackage, shouldn’t the next release of `A` be a patch-level release, and therefore unlikely to be forbidden by the user’s version constraints? But this requires an unrealistic picture of how maintainers work: it is more likely that the maintainer of `A` has some work-in-progress on `A`, to which they will _add_ the GHC compatibility patches, to be included in the new release. So in general those changes might reach Hackage in any kind of release!

However, if users are stuck using the GHC._X_.hackage versions of packages for longer than strictly necessary that is not too bad: the patches should be inoffensive, and the whole mechanism is opt-in regardless.


### 3.3. Package maintainers

Anyone submitting a patch to GHC._X_.hackage should also submit a corresponding PR to the source repository of the package. (The PR will usually be identical to the GHC._X_.hackage patch.) The maintainer can, _in their own time_, merge the PR and make a release to Hackage.

The only change to the work facing maintainers is that they are hopefully more likely to receive a good-quality patch fixing their compatibility problems for them.


### 3.4. Community engagement

The community should be encouraged to engage with GHC._X_.hackage both before release (when it is `head.hackage`) and after release. One could imagine hackathons or other community events, with lists of packages to update, and celebrations when they are all done.

GHC._X_.hackage should be announced as part of the GHC _X_ release announcement.

Whenever GHC._X_.hackage is mentioned, it must be clearly stated that it is strictly to be used for testing and compatibility work, since it contains patches that have not been vetted by the package maintainers.

### 3.5. Example workflow

This section describes an example workflow using both today's tools and a hypothetical GHC._X_.hackage overlay.

#### 3.5.1. The Situation

Cast of characters:
 * _Alice_ is the maintainer of package `A`, which depends on packages `B` and `C`.
 * _Bob_ maintains package `B`, though not particularly actively.
 * _Carol_ actively maintains package `C` in her spare time.
 * _Dan_ is an active user of `C` and an early adopter of new compilers.
 * _Erin_ is the maintainer of an in-house proprietary application `E`, which depends on package `A`.
 * _Frank_ maintains the open-source application `F`, which also depends on `A`.
 
Package dependencies:
```
                  /---------\
                  | E (app) |
                  \----+----/
                       |                /---------\
                       v           /--->| B (lib) |
/---------\        /---------\     |    \---------/
| D (app) +------->| A (lib) +-----|
\---------/        \---------/     |    /---------\
                        ^          \--->| C (lib) |
                        |               \---------/
                   /----+----\
                   | F (app) |
                   \---------/
```

Alice, the maintainer of package `A`, discovers that dependencies `B` and `C` have not been updated for compatibility with a new release of GHC, version X. Alice notices that `B` and `C` are both incompatible with the new release of `base`, because her build tool fails to solve the constraints and gives her a helpful message.

On investigation, she discovers that `C` has an unmerged pull request that updates it for the new version of GHC, submitted by Dan. Carol, the maintainer of `C`, is simply too busy to review and merge the pull request. Bob, the maintainer of `B`, has not yet even noticed that it does not build, as `B` is not a particularly popular package among early adopters of compilers.

#### 3.5.2. Without GHC._X_.hackage

Alice begins by adding a local override to her `cabal.project` file for `C` that points at the code from Dan's pull request, and she checks out her clone of `B` into a local directory. She figures out how to update it for the new version of `C`, and submits a pull request to Bob. In the meantime, she is able to continue work on `A`, but she is not yet able to release the update on Hackage.

Dan and Erin, who work on in-house proprietary applications called `D` and `E` that uses `A`, would be well-served with GHC version _X_'s improved support for new hardware. But they cannot update `E` until `A`, `B`, and `C` work. Dan and Erin must watch the branches of the `A` repository in order to know that there is a version of `A` that can, with appropriate overrides, work with GHC version _X_, but in practice, this is difficult and time-consuming and may not happen. Because of this, Erin doesn't discover an obscure bug in GHC _X_ that affects `E`, and waiting for the fix delays her company's migration even more.

Frank, who maintains the open-source application `F`, is in the same boat. He can't release a version compatible with GHC _X_ unless he discovers appropriate local overrides. Because Erin and Frank don't know each other, they will duplicate work.

#### 3.5.3. With GHC._X_.hackage

Alice opens a pull request to `GHC.X.hackage` that adds three files to the repository: overrides for the latest Hackage releases of `A`, `B`, and `C` that allow them to build with GHC version X. She follows the contribution guidelines for her `GHC.X.hackage` PRs and includes links to the sources of the new code, allowing reviewers to see the context that it came from. The `GHC.X.hackage` maintainers review and merge the PRs.

Dan is attempting to update his in-house application `D`, which depends on `A`. There have been no new Hackage releases of `A`, `B`, or `C`, so his build fails. He reconfigures the build to point at `GHC.X.hackage`, and everything still works just fine. He does not need to explore the repositories for `A`, `B`, and `C`. Dan can now sleep at night, confident that he's unlikely to have to update anything other than version bounds in `D` when the ecosystem catches up.

Erin is attempting to update her in-house application `E`, which depends on `A`. There have been no new Hackage releases of `A`, `B`, or `C`, so her build fails. She reconfigures the build to point at `GHC.X.hackage`, and discovers that her application triggers a rare bug in GHC version X. She reports the bug, and GHC-X.2 is released shortly later with a fix. Erin is able to sleep at night, confident that her application is compatible with the new compiler without waiting for the ecosystem to catch up first.

Frank is attempting to update his open-source application `F`, which depends on `A`. There have been no new Hackage releases of `A`, `B`, or `C`, so his build fails. He reconfigures the build to point at `GHC.X.hackage`, and he discovers that his application is incompatible with a change in `base`. He updates the application on a branch, and is prepared to do a quick Hackage release as soon as `A`'s update is released.

Alice, by doing a small amount of additional work on top of what she was doing anyway, was able to unblock Dan, Erin, and Frank, who would have had to repeat her investigations if not for `GHC.X.hackage`. This did not require Bob or Carol to be available or responsive, and they did not get any extra obligations or work as maintainers as a result of Alice's contributions.

## 4. Rationale

### 4.1. Speed

GHC._X_.hackage enables significantly faster and more parallel fixing of packages.

Consider the example from the Motivation section. With GHC._X_.hackage, the maintainer of `A` can, on the day of the GHC _X_ release, fix both `B` and `C` in GHC._X_.hackage, and use GHC._X_.hackage to test and release a fix for `A`.

However, this speedup _only_ materializes so long as the changes required in `A`’s dependencies are simple patch-level changes, since those are the only patches that GHC._X_.hackage accepts. For everything else, we will still have to await a Hackage release. That may be problematic in practice, since many of the packages which will need major changes to support GHC _X_ are down at the bottom of the dependency tree (e.g. because they make assumptions about GHC internals).

We will have to see in practice how much faster things get: perhaps GHC _X_ will not be usable on day 1, but hopefully it will be usable significantly sooner than it would be otherwise.


### 4.2. Community involvement

Because GHC._X_.hackage accepts patches from people who are not the package maintainers, it becomes possible for community members to make significant impact on GHC adoption; a single motivated individual can fix large swathes of the package ecosystem!

If we harness this well, we could potentially enable people to experiment with a mostly-working ecosystem via GHC._X_.hackage in a few weeks, rather than months.


### 4.3. Maintainer awareness

GHC._X_.hackage also partially resolves the problems of maintainers knowing when they are able to do compatibility work. The answer becomes: you can at least _try_ as soon as GHC _X_ is released.

This still isn’t ideal: many packages will be blocked on key blockers that require major changes (and hence Hackage releases), but the current proposal cannot fix that.


## 5. Implementation

Since `head.hackage` already exists, along with all the infrastructure to support it, we believe that it should be relatively straightforward to support multiple GHC._X_.hackages, perhaps even with the same source repository (using branches).

There are a couple of roles that need to be filled on an ongoing basis

- Someone needs to maintain GHC._X_.hackage. That is: vet patches for inclusion, merge them, keep the CI working, etc.
- If we want to get significant community involvement, it would be useful for someone to manage that involvement, a "community release manager" of sorts. That is: help people make contributions, drum up excitement, update the community on progress, help upstreaming patches, etc.

There’s a lot of flexibility in filling these roles. They could be

- Occupied by the same person, or different ones
- Rotating per release, or few releases
- Filled by a keen community member, or a paid professional

Currently, the GHC._X_.hackage maintainer role is effectively filled by the GHC developers who maintain `head.hackage`. However, if the scope of the role expands, this is likely to be unsustainable. Should this be the case, we will submit a separate proposal for support from the Haskell Foundation.


### 5.1. Foliage

We intend to migrate to [`foliage`](https://github.com/andreabedini/foliage) to construct both `head.hackage` and GHC._X_.hackage. This presents a workflow that is much more familiar to developers than the current `head.hackage` workflow of patch files that are used to build a Hackage repository. With Foliage, the repository is constructed from a collection of files that point at tarballs and provide hashes. This process is very similar to adding local overrides to a Stack or Cabal file, but it allows multiple versions to be overridden at once.


## 6. Alternatives and interactions


### 6.1. GHC Maintainer Preview

The [GHC Maintainer Preview proposal](https://github.com/ghc-proposals/ghc-proposals/pull/417) suggested cutting an early pre-release of new GHC versions in order to help maintainers update their packages before the "real" release. A frequent comment in the proposal discussion was that the Maintainer Previews would _not_ actually be directly useful to most maintainers, because they would still be stuck waiting for their (many levels of) dependencies to update before they could do anything. It’s no good having a maintainer pre-release if maintainers can’t _use_ it to fix their packages!

GHC._X_.hackage alleviates this problem. Package maintainers could fix their packages and their dependencies by submitting patches to head.hackage, which would become GHC._X_.hackage in due course.


### 6.2. What’s wrong with head.hackage?

The package upgrade workflow described here is occasionally done today with `head.hackage`. For example, [here is an attempt](https://github.com/haskell/haskell-language-server/pull/2503) to use `head.hackage` to pre-emptively fix up `haskell-language-server` for GHC 9.2.

So maybe we don’t need to do anything except

- Make one `head.hackage` for each version GHC version _X_
- Encourage more maintainers to use `head.hackage`
- Encourage more people to submit patches to `head.hackage`
- Fix the versioning issues mentioned above which make `head.hackage` hard to reliably use

But that’s essentially what the GHC._X_.hackage proposal is, just with a little bit more structure, policy, and marketing.


### 6.3. Precognitive releases

The proposal as it stands does not address one of the major roadblocks to getting support out: updating packages that require major changes and hence Hackage releases.

It would be nice to do better than that. We could shoot for the following property:

**Desired Property**. An maintainer of package A can update and release to Hackage a new version of `A`, relying on GHC._X_.hackage for `A`'s dependencies. _This release of `A` should continue to work when the maintainers of those dependencies make their releases to Hackage in due course._

That is, we would like maintainers to be able to make _precognitive_ Hackage releases based on how things _will_ turn out in the future on Hackage.

A sketch of a solution could be:

1. Allow patches in GHC._X_.hackage that make more than patch-level changes, and which must create _new_ versions in GHC._X_.hackage.
2. Encourage maintainers to make proactive Hackage releases based on the changes _and versions_ in GHC._X_.hackage.

For example, we might have a patch in GHC._X_.hackage that fixes `B-2` by creating `B-2.1`. The maintainer of `A` could then release a fixed version of `A` to Hackage, depending on `B < 3.0`, assuming that when `B` is finally updated on Hackage, the compatibility fixes will appear in `B-2.1`.

From this example it is clear that this system relies heavily on _assumptions_ about how package maintainers release new versions of their packages. But this is quite problematic: as discussed in Section 3.2, package maintainers often have work-in-progress which they may want to release _with_ the compatibility patches. So it is quite possible that the next release of `B` may be `B-3`, not `B-2.1`. At that point the maintainer of `A` has to do _another_ release. That’s terrible - we’ve then made _more_ work for the maintainer of `A` than they had before GHC._X_.hackage existed!

It’s unclear whether a system like this can be made to work. In particular, it is not good enough for the assumptions to be _mostly_ true: even if it _only sometimes_ creates more work for maintainers, this is likely to be enough of a risk to put them off using it.


### 6.4. Tested-With

It’s a bad user experience to compile a package with GHC _X_, and just get a compile error. The `Tested-With` field of a Cabal file helps: if present it specifies which versions of GHC this package has been tested with.

The goal should be that if `A` releases to Hackage a new version of package `A`, depending on (`B < 1.6`), but `B` has not yet released a new version to Hackage, then a naive user who is ignorant of GHC._X_.hackage will get a message like

> Package B has not yet been updated to GHC X
>
> Consider using GHC._X_.hackage (link)

Cabal could produce this message immediately from its solver, without compiling anything.

But perhaps `B` requires no updates at all to work with GHC X (this is a common case). Then this message would be over-conservative. Maybe Hackage could proactively set the `Tested-With` field, by building the package and running its test suite? Or maybe we need two fields: a manual one and an automatic one.


### 6.5. Versioning of patched packages

The decision of whether to bump the version number of patched packages is a
tricky one. While GHC.*X*.hackage patches should avoid making breaking changes
whenever possible, there will no doubt be times where doing so will be
unavoidable, even if only in "internal" modules. While the PVP would mandate
that we mark such changes with a major version bump, doing so would
significantly hurt the usability of GHC.*X*.hackage for its intended purpose as
downstream users seeking to test their package against the patch-set would need
to bump their package's bounds.

In the case of non-breaking changes it is tempting for patches to bump the
minor version number to reflect the fact that the package is not identical to
what is found in Hackage. However, doing so brings its own issues. For
instance, the bumped version may clash with a later version bump made by the
upstream maintainer. For this reason, we believe that performing no version
bump at all will lead to the least confusion.

