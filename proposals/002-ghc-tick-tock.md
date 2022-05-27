A tick-tock release schedule for GHC
====================================

*Ben Gamari*

As Haskell adoption has grown, the needs of GHC's user-base have grown more
diverse. While all of GHC's users expect predictable, consistently high-quality
releases, preferences on the frequency and longevity of these releases differ.
While frequent releases helps to deliver new language and compiler features
into users' hands quickly, high frequency comes at the expense of making work
for packagers and commercial users who want longer support windows. Moreover,
the benefits of frequent releases must be weighed against the considerable
fixed cost inherent in making a GHC release.

Background
-----------

Prior to 2017, GHC had no formal release schedule; the compiler was released
when its maintainers believed that the progress and stability of the compiler
warranted. As a result, the interval between major releases fluctutated
significantly between six and 18 months and many users understandably expressed
a desire to see more consistency in release timing. In response, in mid-2017
(shortly after the release of GHC 8.2.1), we
[proposed](https://www.haskell.org/ghc/blog/20170801-2017-release-schedule.html)
to move GHC to a time-based six-month release cadence. Since then, release
frequency has been higher but has not quite exhibited the degree of consistency
that we set out to achieve.

| Release  | Release date  | Days since previous major release |
| -------- | ------------- | --------------------------------- |
| 8.0      | 21 May 2016   | 421                               |
| 8.2      | 22 Jul. 2017  | 427                               |
| 8.4      | 8 Mar. 2018   | 229                               |
| 8.6      | 21 Sept. 2018 | 197                               |
| 8.8      | 25 Aug. 2019  | 338                               |
| 8.10     | 24 Mar. 2020  | 212                               |
| 9.0      | 4 Feb. 2021   | 317                               |
| 9.2      | 29 Sep. 2021  | 237                               |

However, as Haskell usage has grown so has the pool of resources which supports it. Thanks to the Haskell Foundation and support from commercial users, GHC now benefits from a team of three full-time engineers charged with, among other things, getting functional compilers into users' hands.

With these resources, we believe that the goal of consistent six-month releases is finally attainable.

The Problem
------------
While all users have expressed a desire to have predictable time-based releases, many commercial users, and library authors, have expressed concern that they cannot keep pace with a six-month major release cadence. In addition, GHC developers have been straining to keep pace with backports given that there have, until recently, been three active release series:

 * 8.10, which is still by far the most popular supported release
 * 9.0, which for a variety of reasons has seen less usage
 * 9.2, which has not yet been released for long enough to be widely used

Given that we are a small-but-growing community, it is important that we use
our human resources wisely and ensure that commercial users are not burdened by
GHC's faster release cycle. On the other hand, long release periods and long
backport windows had a considerable cost on GHC developers, as well as making
it harder for users to benefit from new GHC features.

Proposed change
---------------

To address the above problem, we propose a "bi-modal" six-month release cadence
with two alternating types of major releases:

* *Long-term support releases*, for which the GHC team commits to provide critical backports for at least 18 months
* *Intermediate releases*, which will continue to have critical backports only until the next release

The goal of this distinction is to offer users (especially in industrial
contexts) who need stability and predictable maintenance windows clear
guidance.

Note that the these two release types are *not* distinguished by the new
features or changes that they include: we expect that both release types may
contain breaking changes.   There are two reasons for this choice:

* Delaying merging a feature for six months is a major drag on GHC's developers
* If missing the intermediate release leads to a year-long delay, a consequence
  is iterative pressure to "just delay the release a little longer".

However, it is our hope that this distinction will more clearly set user
expectations and give GHC's maintainers a clearer picture of the expected
lifetime of release branches.  In practice, bimodal release structure reflects
what GHC has been doing for quite some time:

 * GHC 8.6 was essentially an LTS release, enjoying five minor releases over the course of nine months
 * GHC 8.8 was comparatively shorter, having only three minor releases over seven months
 * GHC 8.10 was quite long-lived and even had significant additions well after 8.10.1 (e.g. with the addition of AArch64/Darwin support)
 * GHC 9.0 was something of a runt release as focus remained on stabilizing GHC 8.10's AArch64/Darwin support
 * GHC 9.2 is still young but the GHC developers are currently planning on supporting it into 2022.

```

  ┌───────────────────────────────────────────────┐
  │ GHC 9.2 (LTS)                                 │
  └───────────────────────────────────────────────┘

                 ┌────────────────────┐
                 │ GHC 9.4            │
                 └────────────────────┘

                                ┌──────────────────────────────────────────────┐
                                │ GHC 9.6 (LTS)                                │
                                └──────────────────────────────────────────────┘

                                               ┌────────────────────┐
                                               │ GHC 9.8            │
                                               └────────────────────┘

                                                              ┌─────────────────────────────────────────────┐
                                                              │ GHC 9.10 (LTS)                              │
                                                              └─────────────────────────────────────────────┘


  │                             │                             │                             │
  │              │              │              │              │              │              │
  ├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┼──────────────┼───────────►
  │              │              │              │              │              │              │
  │                             │                             │                             │
  Y1                            Y2                            Y3                            Y4

```

### Alternative designs

In general the decision of one-LTS, one "normal" release is somewhat arbitrary;
one could envision similar schedules with less frequent LTS releases (and,
perhaps, commensurately longer support windows for such releases).  However,
supporting major releases of GHC for long periods of time begins to exact a
particularly high cost on GHC maintainers after around a year. We
suspect that the design presented above strikes the right balance between
longevity and maintenance cost, but we would appreciating hearing feedback from
the community.
