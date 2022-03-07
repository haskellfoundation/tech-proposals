# Adapting to a new version of GHC, way faster

*Michael Peyton Jones, Ben Gamari, Simon Peyton Jones, Théophile “Hécate” Choutri*


## 1. Motivation

At the moment we have a problem with propagating support for a new version of GHC through the Haskell ecosystem. Suppose package `A` depends on `B` which depends on `C`. Then the process goes like this

1. GHC version X is released.
2. The maintainer of `C` must wake up, fix their package, and make a Hackage release. The maintainers of `A` and `B` can do nothing at this point.
3. The maintainer of `B` must wake up, fix their package, and make a Hackage release. The maintainer of `A` can still do nothing.
4. The maintainer of `A` can finally fix their package and make a Hackage release.

This process is terrible in lots of ways

- It is utterly serial. The maintainer of `B` cannot lift a finger until the maintainer of `C` has not only fixed package `C`, but also uploaded a new release to Hackage.
- Each step has multiple serial parts: often the maintainer will merge a patch (perhaps in response to prompting) but _not_ do a release, blocking further progress.
- If a maintainer is unavailable for any reason, the entire dependency tree of that package is blocked. In an ecosystem with hundreds of widely used packages, the chances of every single maintainter being available in a timely fashion are close to zero.
- It is not clear to even a willing and available maintainer _when_ they need to wake up and do some work. Often it is up to a motivated individual (say the maintainer of an application that uses `A`) to sequentially bug the maintainers of `C` and `B` and `A` in sequence so they know that they are able to do something.

**The result of this is that it usually takes months or even years for support for a GHC version to percolate through the ecosystem.** For example, GHC 9.0.1 was released in February 2021, but as of December 2021 haskell-language-server still does not support it fully, largely because of difficulty updating dependencies.

We could enormously speed up the process of updating the package ecosystem for a new version of GHC if

- We could parallelise the process.
- We could allow many people to contribute to the routine updates necessary to adapt a package to changes in its dependencies.
- We could allow package maintainers to do their work when it was convenient to them, without holding up the whole train

We propose to partially improve the situation by widespread usage of a Hackage overlay (see Section 2), GHC.X.hackage, populated with non-maintainer-contributed patches for the purpose of unblocking package maintainers in adapting their library to new GHC releases.


## 2. Background


### 2.1 Additional package repositories and Hackage overlays

Cabal supports _additional package repositories_ beyond Hackage. These can be configured in the Cabal config file, or in a cabal.project file.

Users can state which repositories they want to be _active_, and how they should be consulted. This allows additional repositories to override, or merely augment Hackage.

  
Overriding Hackage can be accomplished by listing the repositories in a Cabal config file, like so:
```
active-repositories: hackage.haskell.org, X
```
This will cause repository X to be consulted first for any given version of a package, but allow other versions to come from Hackage. For example, if `X` contains `A-1` and `A-2`, and Hackage contains `A-2` and `A-3`, then `A-1` and `A-2` will be taken from `X`, but `A-3` can be taken from Hackage. ([Here is the documentation](https://cabal.readthedocs.io/en/3.6/cabal-project.html) for active-repositories; oddly, the list is searched last-to-first.)

We refer to this style of additional repository as a **Hackage overlay**, since it provides a way to _override_ specific versions of packages which appear in Hackage, while using Hackage for everything else normally.

Additional package repositories behave well in the face of changes. Cabal will recompute the build plan whenever repositories are changed or updated; and since Cabal identifies packages by a hash which includes the hash of the source tarball, Cabal will never get confused between packages from different repositories, and it will cope gracefully even if package sources are _mutated_ in the repository.


### 2.2 head.hackage

[head.hackage](https://gitlab.haskell.org/ghc/head.hackage/) is an example of a Hackage overlay. It is built from the linked source repository which consists of a set of patches to released Hackage versions of packages. head.hackage contains patches that enable packages to build with GHC head (hence the name), and is used by the GHC developers to test the upcoming GHC release.


## 3. Proposal


### 3.1 GHC.X.hackage


#### 3.1.1 The GHC.X.hackage package repository

For each released major version _X_ of GHC, a Hackage overlay is provided called GHC.X.hackage. The intention of GHC._X_.hackage is to override versions of packages that require changes to build with GHC-_X_.

GHC._X_.hackage is a _non-authoritative_ set of patched packages, whose goal is primarily to unblock package maintainers. It should not be used in production, and any faults are not to be laid at the door of the package maintainers.

Since GHC._X_.hackage is opt-in, users who do not use GHC._X_.hackage will (obviously) not be able to use the packages which have been fixed in GHC.X.hackage, until fixed versions of those package and all their dependencies are actually released on Hackage.

Additionally, head.hackage becomes a pre-release version of GHC._X_.hackage, and as such will change to behave in the same way as GHC._X_.hackage.

When the release branch for GHC version _X_ is created, the current state of the head.hackage repository is transitioned into GHC._X_.hackage. A new head.hackage is created for the new GHC head.

GHC.X.hackage repositories continue to exist for old versions of GHC in perpetuity. They are not retired, although it is expected that they will be rendered redundant as proper releases are made in Hackage.


#### 3.1.2 Adding patches to GHC.X.hackage

The source of GHC.X.hackage is held in a Git repository, and accepts patches from community members _who need not be the package maintainer_. For example, the package maintainer of A might submit patches to fix B and C. This will use the same infrastructure as the existing head.hackage repository: here is an [example of a PR](https://gitlab.haskell.org/ghc/head.hackage/-/merge_requests/194) for head.hackage.

The inclusion criteria for a patch are:

1. The patched package should build with GHC X.
2. The patch should represent a patch-version level change according to the PVP, in particular it should not change the package’s API.
3. The patch should be plausibly acceptable by the upstream maintainer.

Criterion (3) is deliberately vague, but suggests that the tests should pass (although this will likely not be easy to verify given the diversity of packages), that the patch should not needlessly break compatibility with previous versions of GHC, etc. Of course, since in most cases the patch will _not_have been vetted by the maintainer, precise correctness cannot be guaranteed.

Package maintainers, or others, can update packages even _before_ GHC X is released, by uploading patches to head.hackage. After the release, those patches will be part of GHC.X.hackage. The same rules for patch inclusion apply to head.hackage before it becomes GHC.X.hackage.


#### 3.1.3 Removing patches from GHC.X.hackage

Patches should not _need_ to be removed from GHC.X.hackage. They will simply become obsolete as Hackage becomes more and more up-to-date.

However, in any case if a package maintainer requests the removal of a patch from GHC.X.hackage (for example, it might be outright wrong) then it should certainly be removed.


### 3.2 Versioning and releases

While GHC.X.hackage overrides particular versions of packages from Hackage, those overrides will quickly become obsolete when a new version of the package is released to Hackage.

Suppose that GHC.X.hackage contains a patched version of A-X. When A’s maintainer next releases A, they will release A-Y for Y>X. Users who use GHC.X.hackage will in most cases therefore find that cabal picks build plans with the new A-Y instead of the old (patched) A-X. However, if they have constraints that prevent this (say, because A-Y is a major version bump), then they may keep getting the GHC.X.hackage version.

It would be nice to argue that this case is unlikely: since by assumption we only allow patch-level changes into GHC.X.hackage, shouldn’t the next release of A be a patch-level release, and therefore unlikely to be forbidden by the user’s version constraints? But this requires an unrealistic picture of how maintainers work: it is more likely that the maintainer of A has some work-in-progress on A, to which they will _add_ the GHC compatibility patches, to be included in the new release. So in general those changes might reach Hackage in any kind of release!

However, if users are stuck using the GHC.X.hackage versions of packages for longer than strictly necessary that is not too bad: the patches should be inoffensive, and the whole mechanism is opt-in regardless.


### 3.3 Package maintainers

Anyone submitting a patch to GHC.X.hackage should also submit a corresponding PR to the source repository of the package. (The PR will usually be identical to the GHC.X.hackage patch.) The maintainer can, _in their own time_, merge the PR and make a release to Hackage.

The only change to the work facing maintainers is that they are hopefully more likely to receive a good-quality patch fixing their compatibility problems for them.


### 3.4 Community engagement

The community should be encouraged to engage with GHC.X.hackage both before release (when it is head.hackage) and after release. One could imagine hackathons or other community events, with lists of packages to update, and celebrations when they are all done.

GHC.X.hackage should be announced as part of the GHC X release announcement.

Whenever GHC.X.hackage is mentioned, it must be clearly stated that it is strictly to be used for testing and compatibility work, since it contains patches that have not been vetted by the package maintainers.


## 4. Rationale


### 4.1 Speed

GHC.X.hackage enables significantly faster and more parallel fixing of packages.

Consider the example from the Motivation section. With GHC.X.hackage, the maintainer of A can, on the day of the GHC X release, fix both B and C in GHC.X.hackage, and use GHC.X.hackage to test and release a fix for A.

However, this speedup _only_ materializes so long as the changes required in A’s dependencies are simple patch-level changes, since those are the only patches that GHC.X.hackage accepts. For everything else, we will still have to await a Hackage release. That may be problematic in practice, since many of the packages which will need major changes to support GHC X are down at the bottom of the dependency tree (e.g. because they make assumptions about GHC internals).

We will have to see in practice how much faster things get: perhaps GHC X will not be usable on day 1, but hopefully it will be usable significantly sooner than it would be otherwise.


### 4.2 Community involvement

Because GHC.X.hackage accepts patches from people who are not the package maintainers, it becomes possible for community members to get really involved. A single motivated individual can fix large swathes of the package ecosystem!

If we harness this well, we could potentially enable people to experiment with a mostly-working ecosystem via GHC.X.hackage in a few weeks, rather than months.


### 4.3 Maintainer awareness

GHC.X.hackage also partially resolves the problems of maintainers knowing when they are able to do compatibility work. The answer becomes: you can at least _try_ as soon as GHC X is released.

This still isn’t ideal: many packages will be blocked on key blockers that require major changes (and hence Hackage releases), but the current proposal cannot fix that.


## 5. Implementation

Since head.hackage already exists, along with all the infrastructure to support it, we believe that it should be relatively straightforward to support multiple GHC.X.hackages, perhaps even with the same source repository (using branches).

There are a couple of roles that need to be filled on an ongoing basis

- Someone needs to maintain GHC.X.hackage. That is: vet patches for inclusion, merge them, keep the CI working, etc.
- If we want to get significant community involvement, it would be useful for someone to manage that involvement, a “community release manager” of sorts. That is: help people make contributions, drum up excitement, update the community on progress, help upstreaming patches, etc.

There’s a lot of flexibility in filling these roles. They could be

- Occupied by the same person, or different ones
- Rotating per release, or few releases
- Filled by a keen community member, or a paid professional

Currently, the GHC.X.hackage maintainer role is effectively filled by the GHC developers who maintain head.hackage. However, if the scope of the role expands, this is likely to be unsustainable. Potentially this is somewhere where the HF could provide assistance.


## 6. Alternatives and interactions


### 6.1 GHC Maintainer Preview

The[GHC Maintainer Preview proposal](https://github.com/ghc-proposals/ghc-proposals/pull/417) suggested cutting an early pre-release of new GHC versions in order to help maintainers update their packages before the “real” release. A frequent comment in the proposal discussion was that the Maintainer Previews would _not_ actually be directly useful to most maintainers, because they would still be stuck waiting for their (many levels of) dependencies to update before they could do anything. It’s no good having a maintainer pre-release if maintainers can’t _use_ it to fix their packages!

GHC.X.hackage alleviates this problem. Package maintainers could fix their packages and their dependencies by submitting patches to head.hackage, which would become GHC.X.hackage in due course.


### 6.2 What’s wrong with head.hackage?

The package upgrade workflow described here is occasionally done today with head.hackage. For example,[here is an attempt](https://github.com/haskell/haskell-language-server/pull/2503) to use head.hackage to pre-emptively fix up haskell-language-server for GHC 9.2.

So maybe we don’t need to do anything except

- Make one head.hackage for each version GHC version _X_
- Encourage more maintainers to use head.hackage
- Encourage more people to submit patches to head.hackage

But that’s essentially what the GHC._X_.hackage proposal is, just with a little bit more structure, policy, and marketing.


### 6.3 Precognitive releases

The proposal as it stands does not address one of the major roadblocks to getting support out: updating packages that require major changes and hence Hackage releases.

It would be nice to do better than that. We could shoot for the following property:

**Desired Property**. An maintainer of package A can update and release to Hackage a new version of `A`, relying on GHC._X_.hackage for `A`'s dependencies. _This release of `A` should continue to work when the maintainers of those dependencies make their releases to Hackage in due course._

That is, we would like maintainers to be able to make _precognitive_ Hackage releases based on how things _will_ turn out in the future on Hackage.

A sketch of a solution could be:

1. Allow patches in GHC.X.hackage that make more than patch-level changes, and which must create _new_ versions in GHC.X.hackage.
2. Encourage maintainers to make proactive Hackage releases based on the changes _and versions_ in GHC.X.hackage.

For example, we might have a patch in GHC.X.hackage that fixes `B-2` by creating `B-2.1`. The maintainer of `A` could then release a fixed version of `A` to Hackage, depending on `B < 3.0`, assuming that when `B` is finally updated on Hackage, the compatibility fixes will appear in `B-2.1`.

From this example it is clear that this system relies heavily on _assumptions_ about how package maintainers release new versions of their packages. But this is quite problematic: as discussed in Section 3.2, package maintainers often have work-in-progress which they may want to release _with_ the compatibility patches. So it is quite possible that the next release of B may be B-3, not B-2.1. At that point the maintainer of A has to do _another_ release. That’s terrible - we’ve then made _more_ work for the maintainer of A than they had before GHC.X.hackage existed!

It’s unclear whether a system like this can be made to work. In particular, it is not good enough for the assumptions to be _mostly_ true: even if it _only sometimes_ creates more work for maintainers, this is likely to be enough of a risk to put them off using it.


### 6.4 Tested-With

It’s a bad user experience to compile a package with GHC X, and just get a compile error. The `Tested-With` field of a Cabal file helps: if present it specifies which versions of GHC this package has been tested with.

The goal should be that if `A` releases to Hackage a new version of package `A`, depending on (`B < 1.6`), but `B` has not yet released a new version to Hackage, then a naive user who is ignorant of GHC.X.hackage will get a message like

> Package B has not yet been updated to GHC X
>
> Consider using GHC.X.hackage (link)

Cabal could produce this message immediately from its solver, without compiling anything.

But perhaps `B` requires no updates at all to work with GHC X (this is a common case). Then this message would be over-conservative. Maybe Hackage could proactively set the `Tested-With` field, by building the package and running its test suite? Or maybe we need two fields: a manual one and an automatic one.


### 6.5 Versioning of patched packages

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

