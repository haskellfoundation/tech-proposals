# Midstream bindists

## Abstract

Bindists for the Haskell toolchain have been produced by upstream (the developers of each respective tool) for
a long time and many tools rely on these "official" bindists (e.g. GHCup and stack).

We propose here that bindists are additionally built and maintained in joint effort by the projects that
provide installation experiences, removing the hard dependency on upstream bindists entirely.

## Background

Historically, installers like GHCup and stack have used upstream bindists for mainly one reason: it's easy to do
so and doesn't require further efforts.

However, using upstream bindists directly is extremely rare in the Linux world of distribution. Most distributions
build, package, test and curate binary packages themselves, not only because they have custom formats, but for
reasons of control, trust and quality.

The relationship of GHCup and bindists has also been described in the blog post
[GHCup is not an installer](https://hasufell.github.io/posts/2023-11-14-ghcup-is-not-an-installer.html) recently.

## Problem Statement

From the perspective of a GHCup developer, there are several issues with relying on upstream bindists.

### Platform support

GHC and other tools have in the past dropped support for certain platforms either entirely or requested
the community to step up and do the work (e.g. on GHC CI).

E.g. [GHC ARMv7 support was dropped silently without any call for help](https://gitlab.haskell.org/ghc/ghc/-/issues/21177#note_470440). Similarly, FreeBSD support just ceased to exist when the GHC FreeBSD CI stopped working. Later the community asked for a [revival](https://gitlab.haskell.org/groups/ghc/-/epics/5), but nothing signifcant has happened so far.
GHCup still produces bindists from time to time for FreeBSD, but e.g. the [HLS release manager for 2.5.0.0 recently refused to add FreeBSD bindists](https://github.com/haskell/ghcup-metadata/pull/159).

Similarly, stack used to have issues with Aarch64 bindists:

* https://github.com/commercialhaskell/stack/issues/5709
* https://github.com/commercialhaskell/stack/issues/5854
* https://github.com/commercialhaskell/stack/issues/5540
* https://github.com/commercialhaskell/stack/issues/5610
* https://github.com/commercialhaskell/stack/issues/5619
* https://github.com/commercialhaskell/stack/issues/6141
* https://github.com/commercialhaskell/stack/issues/6142

Recently, cabal-install had issues with i386 binaries and alpine, delaying a GHCup metadata PR:

* https://github.com/haskell/ghcup-metadata/pull/127#issuecomment-1766020410

These issues are frequent and so far the GHCup developers used to single handedly fix all those missing bindists manually
and provide them here: https://downloads.haskell.org/~ghcup/unofficial-bindists/

This is unfunded and significant work.

### Bindist maintenance

Sometimes, bindists are broken, e.g. for GHC there are a couple of instances:

* 9.0.2 shipping without profiling info: https://gitlab.haskell.org/ghc/ghc/-/issues/21841
* DESTDIR variable ignored by `make install`: https://gitlab.haskell.org/ghc/ghc/-/issues/19646

Sometimes, bindists have been built for very old version of linux distros and won't run well on newer linux versions.
This is currently a problem since Debian has removed ncurses5:

* https://github.com/haskell/ghcup-hs/issues/902
* https://answers.launchpad.net/ubuntu/+source/ncurses/+question/707838
* https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1025964

For cabal, there has been the infamous hLock issue:

* https://github.com/haskell/cabal/issues/7313
* https://github.com/haskell/cabal/issues/7950

This shows that bindists, for current and historical versions, need continuous maintenance. However, upstream developers
so far have very rarely engaged in this type of maintenance work, pushing it down to GHCup. As an example, here are all the
manually patched and re-packaged bindists that fix the DESTDIR bug outlined above: https://downloads.haskell.org/ghcup/unofficial-bindists/ghc/curated/

This type of work also requires significant time.

### Quality gateway

There's something pleasing about upstream providing bindists: there's a perceived trust about it in the community,
e.g. the [haskell.org committee expressed concerns about GHCup changing bindists in the past](https://github.com/haskell-infra/www.haskell.org/issues/212#issuecomment-1272312911).

However, rarely do upstream developers have signifcant experience in redistribution, nor do they have the time to focus
on all the issues that come with it. Bindists are mostly provided "as-is" and support beyond what the release CI outputs is
left to midstream (GHCup, stack, ...).

Conceptually, it is a good idea to separate concerns: upstream provides the sources and has tested it. Distributions build the binaries,
validate that the program can be successfully built from source and make sure that the final artifacts pass the test suite.
This is good, because we want to know whether end users can build e.g. a functioning GHC from source. If only the release CI outputs
a GHC that passes the test suite, then something is fundamentally broken.

Distributors are often closer to the end-users and can provide additional support and efforts for the installation experience.

### Security backports

Cabal has recently found to be vulnerable to [HSEC-2023-0015](https://github.com/haskell/security-advisories/blob/main/advisories/hackage/cabal-install/HSEC-2023-0015.md).

This vulnerability [had not been communicated to the GHCup team](https://github.com/haskell/security-advisories/issues/129) prior to disclosure,
causing high distress for a backport, since at the time of disclosure, cabal-3.6.2.0
was still 'recommended' by GHCup, since [cabal-3.10.2.0 is still broken on windows](https://github.com/haskell/cabal/issues/9334).

As such, GHCup developers needed a quick and efficient way to:

* patch cabal
* build release binaries for cabal
* ship the binaries

This had been done as a downstream release `3.6.2.0-p1` roughly a week after the disclosure.

Meanwhile, cabal upstream still has not finished their backport due to issues
with hackage dependencies: https://github.com/haskell/cabal/pull/9457

This shows that a certain amount of independences from upstream CI and upstream workflow
is essential to fulfill swift security backports to potentially under-maintained
branches/versions.

### GHC nightlies

As a special case, I want to point out that GHC nightlies have been frequently broken beyond repair:

* https://gitlab.haskell.org/ghc/ghcup-metadata/-/issues/2
* https://gitlab.haskell.org/ghc/ghc/-/issues/24000

The breakage was left unattended and some bindists completely vanished from gitlab CI artifacts, because of misconfiguration.
Here's a graph of nightlies availability: https://grafana.gitlab.haskell.org/d/ab109e66-a8a1-4ae9-b976-40e2dfe281ab/availabilitie-of-ghc-nightlies-via-ghcup?orgId=2

## Prior Art, Related Efforts and alternative solutions

So far, GHCup developers have tried to close the gap, doing signifcant work on upstream CIs and building bindists manually where
necessary.

Bindists for cabal-install are now produced by GHCup's own CI: https://github.com/haskell/ghcup-metadata/blob/develop/.github/workflows/cabal-release.yaml

HLS will likely follow shortly.

As mentioned before, there have been attempts to improve the coordination and collaboration across the entire Haskell
toolchain, view the tooling end-user experience in a holistic way and make decisions based on that end-user experience: https://github.com/haskellfoundation/tech-proposals/issues/48

The proposers believe that the community structure at the moment does not allow such an approach and there needs to be
significant work to align goals, perception and priorities. Otherwise there will be too much friction.

The most important currency in the open source volunteer world is **energy**. It is not code or technical effort. As such we
believe that the amount of saved energy by being more independent of upstream release processes and decisions far outweighs
potential costs of technical redundancy/duplication.

A future proposal may very well attempt to create a unified user experience across the entire Haskell toolchain
through joint management and collaboration. But that is not in the scope of this proposal and we have no concrete
idea how to achieve that.

## Technical Content

We propose here to start with the smallest step possible, to build the entire Haskell toolchain autonomically.
The way this will be implemented is to start a central GitHub repository that builds bindists for releases of:

- GHC
- HLS
- cabal
- stack

For the following platforms:

- FreeBSD x86_64
- Linux i386
- Linux x86_64
- Linux armv7
- Linux aarch64
- Darwin x86_64
- Darwin aarch64
- Windows x86_64

For the following Linux x86_64 distros:

- Debian
- Ubuntu
- Mint
- Fedora
- CentOS
- RedHat
- Rocky Linux
- Void Linux
- Amazon Linux
- Alpine

Linux i386, armv7 and aarch64 will be confined to Debian or Ubuntu.

These private runners will be made available to the whole Haskell GitHub org and as such benefit
other projects there as well (like HLS, Cabal, bytestring, etc.).

## Future work

The following ideas and goals are outside of the scope of this proposal, but are essential
to understand the broader mission and roadmap that motivated this proposal.

### Enhancements to bindist quality and installation experience

Further goals are:

* enhance the quality of the bindists by
  - running the entire test suite for all of the tools
  - having a mechanism to report test failures back upstream
  - publishing test failures for end users to see
  - communicte test status of bindists clearly through e.g. GHCup
  - resolve GHC issues related to test bindists:
    * https://gitlab.haskell.org/ghc/ghc/-/issues/22726
    * https://gitlab.haskell.org/ghc/ghc/-/issues/22723
    * https://gitlab.haskell.org/ghc/ghc/-/issues/22727
* make fixing bindists easier
  - implement revisions in GHCup: https://github.com/haskell/ghcup-hs/issues/361
  - make it easy to update an older GHC branch and re-run the release pipeline
- Make building upstream release binaries easier
  * https://github.com/haskell/cabal/issues/9461
  * https://github.com/haskell/haskell-language-server/issues/3878

One main idea is that bindists should be primarily tested **on the users system**, because that is where they're going to run.
It is great to know that e.g. the test suite passes on GHC CI, but that may have little value in different environments.
Additionally, issues with tests can flow back to upstream developers and we may develop workflows and processes to streamline
this type of feedback. Early release candidates can assist with this workflow.

Another perception shift necessary is that upstream projects should consider that their build system
are end-user interfaces, making it easier for both distributors and end-user to build release binaries correctly themselves.

### Nightlies

We also want to make nightlies available for GHC and cabal and HLS. This will require
coming up with a permanent storage solution and very robust nightly pipelines.

## Timeline

* 6 months: proof of concept of a central GitHub CI building most of the toolchain
* 12 months: building GHC via github actions

## Budget

We request funding for private GitHub CI runners to power our midstream bindist release pipelines. Prices are in USD.
This is an example/estimate. The HF and the proposer will negotiate the exact terms in private.

- Linux/FreeBSD x86_64 runner on Hetzner (AX52)
  * monthly: $62.24
  * yearly: $746.88
- Linux aarch64 runner on Hetzner (RX170)
  * monthly: $181.73
  * yearly: $2,180.76
- Darwin runner on Hetzner (Mac Mini M1)
  * monthly: $56.59
  * yearly: $679.08

To host one private runner per all these platform, the yearly cost would be: **$3,606.72**

There will likely also be an initial setup cost (as is usual for Hetzner).

We may request more runners depending on the demand, so it may very well be 3 runners per platform, resulting in yearly cost of: $10,820.16

**As such, the budget estimated is between 4 to 10k USD per year.**

## Stakeholders

* GHC developers
* cabal developers
* stack developers
* HLS developers
* VSCode Haskell developers
* GHCup developers
* Haskell toolchain end users

## Success

* reliable, continuously maintained bindists, readily available

