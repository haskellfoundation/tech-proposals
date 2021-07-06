# Are we fast yet? Performance monitoring for the Haskell ecosystem



## Introduction

Rust's [arewefastyet](https://arewefastyet.rs/) service is a great example of where we should strive to be, providing users with a sampling of compile-times for a number of commonly used packages over the last several major compiler releases. This serves to "gamify" contributions to compiler performance and provides a means of highlighting the fruits of developers' efforts to users.

While this is a great model, we should note that Haskell's ecosystem poses a few challenges which complicate replicating this approach verbatim:

 * lack of stability: while in the case of Rust stability is largely guaranteed across compiler releases, Haskell routinely breaks. This makes it challenging
   to build the same code across a wide number of compiler releases.
 * release schedule: while Rust releases on a six-week cadence, GHC releases on a six-month cycle. This may make release-level reporting too coarse-grained to provide a strong incentive to potential contributors due to the long feedback loop.

Here we propose a set of infrastructure for collecting, recording, and monitoring the compile-time and runtime performance of common Haskell
libraries, building on similar infrastructure used by GHC. 

## Motivation 

Users of Haskell expect that their applications consistently deliver predictable runtime performance across GHC and core library versions. While the
ecosystem has world-class tools for testing, characterizing and understanding runtime performance (e.g. `Criterion`, `weigh`, `inspection-testing`, to name a few), it has no consolidated infrastructure for tracking performance across the ecosystem over time.

#### Current GHC Infrastructure

This is project proposes to build on top of GHC's current performance tracking infrastructure. The core of this infrastructure is a common schema for
representing benchmark results and database to record these. This database is fed with samples from a number of sources including:

 * a small set of runtime and compile-time performance testcases in GHC's correctness testsuite, run during the normal continuous integration pipeline
 * runtime and compile-time metrics from the `nofib` benchmark suite, a subset of which is run during the normal continuous integration pipeline
 * compile-time statistics collected from builds of the head.hackage patch set, compiled on a nightly basis

Further technical detail on the implementation of this infrastructure will be
given in a later section.

#### Gipeda

Prior to the introduction of the above-described infrastructure, GHC used the [Gipeda](http://hackage.haskell.org/package/gipeda) package to visualize changes in the metrics from the GHC testsuite collected by a dedicated runner. The `gipeda` package provided a website generator which would produce a static website capable of rendering metric comparisons for pairs of commits. While the interface worked quite well, it was rather limited in temporal range due to the large volume of data that would otherwise need to be generated statically. This limited its utility for long-term tracking of performance characteristics.

## Goals

 The goal of this project is to give GHC authors, core library maintainers, and, eventually, end users insight into their code's performance over time. We hope that this both provides greater understanding of Haskell (especially compile-time) performance characteristics to both GHC developers and users, as well as provides an incentive to spur more contributions in the area of compiler performance.

## People

- **Performers**: Ben Gamari, Matthew Pickering 
- **Reviewers**: Emily Pillmore 
- **Stakeholders**: Core libraries maintainers, users

## Resources

There are two resources necessary to make this proposal happen: 

1. The CI cost of machines testing the implementation of the proposal. 
2. The developer time to implement the infrastructure necessary to achieve the goals of this proposal
3. The on-going cost of maintaining this infrastructure

While it is expected that cost (2) will likely fall on the shoulders of GHC maintainers, we hope that (3) will eventually be spread across a larger group in due time.

## Timeline

TODO

## Implementation

As mentioned above, the current infrastructure is built upon a shared representation for benchmark results. Specifically, a benchmark sample is a tuple of:

* a git commit hash
* a string "test environment" characterizing the operating system and hardware on which the test was run
* a string metric name (e.g. `ghc-testsuite//T1234//bytes_allocated`), which captures, e.g., the name of the test that
  gave rise to the sample (e.g. a GHC test name like `T1234`) and the name of the collected metric (e.g. `bytes_allocated`)
* a floating point value. For consistency we generally prefer metrics where growth corresponds to regression.

GHC feeds this database with results produced by continuous integration via a set of GitLab hooks. To expose this database to world we host a read-only [Postgrest](https://postgrest.org/) instance.

#### Proposed Implementation

While the GHC infrastructure described above has been useful for occasional manual analysis, there is a great deal of room for improvement:

1. there is no automated change-point analysis for catching systematic regressions after the fact
2. no attempt is made to monitor real runtime, instructions, cycles, and other performance characteristics as we lack sufficient capacity of
   adequately-controlled runner hardware.
3. the system has no explicit support for hosting measurements from multiple independent, but co-evolving projects (e.g. `bytestring`,
   which may be affected by changes in itself as well as those in GHC).

To address these, we propose the following concrete technical work:

1. capture metrics from the existing `Cabal` test performed by GHC CI.
2. collect performance metrics from all GHC commits
3. provision a readily-available operating system/hardware configuration which can be used for reproducible performance
   measurement
4. extend the metric representation to allow for multiple project support, allowing ecosystem-central projects like `text` and
   `bytestring` to contribute measurements. It is not currently clear how expressive this representation should be.
5. extend `head.hackage` to allow running and reporting of runtime benchmarks for a subset of widely-used packages
   (e.g. `text`, `vector`, `aeson`).
6. implement a scheme (perhaps by enabling write-access in our Postgrest instance) to allow submission of results from outside
   of GHC's infrastructure.

We expect that most of this work can be done by sufficiently motivated volunteers. The HF's role should principally be organizational.

## Compatibility Issues

N/A

## Performance Impact

N/A

## Deliverables

There are two deliverables associated with this project: 

1. A library, executable, and documentation that will allow the user to stand up performance dashboard monitoring for their project. 
2. A set of pre-built dashboards for GHC and the core libraries 

## Outcomes

Haskell ecosystem performance will be more observable, leading to more developer eyes on library and compiler performance.