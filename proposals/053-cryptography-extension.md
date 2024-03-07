# Botan Cryptography Community Project #2

> NOTE: This document references an [earlier proposal][first proposal]. This new proposal continues with the same long term goals and it is suggested that the reader be familiar with it. A retrospective is also included - see: Appendix: Retrospective.

# Abstract

The [Botan Cryptography Community Project][botan repo], a [previously funded proposal][first proposal], successfully concluded having met its immediate goals and deliverables. This new proposal continues with the same long term goals, and a new set of immediate goals and deliverables in furtherance of them. These new goals are for the continued refinement of the produced libraries, the improvement of installation and CI pipeline support for Windows and Linux, the development of a drop-in interface replacement for `crypton`, and continued development of `botan` to include features such as: secure memory erasure, data types and instances for `random` and `mtl`, and the continued development of cryptographic typeclasses and gold-standard (per-algorithm type-safe) modules.

# Background

Botan is an comprehensive, open-source, BSD-licenced, C++ cryptography library with a stable C API. It offers a broad variety of functionality and algorithms, including **post-quantum cryptography**, is developed and maintained by an active community, and has been [audited][botan audit] in the past.

By binding to Botan, we solved a significant problem of providing much of the necessary 'cryptographic kitchen sink' via a suitably performant, suitably licensed, open-source library. Furthermore, we did this without imposing a large maintenance burden on the Haskell community, as we are only required to maintain the bindings, and not the Botan cryptography library itself.

That was the first milestone and a big step towards building a stable foundation for the Haskell cryptography ecosystem, but there's still work to do. With this milestone, the project enters a new phase in the software development lifecycle - maintenance and development. Now that we have something working, with an initial release and users, we need to listen to feedback and keep it working while we continue to further development of the long-term goals.

That is what this proposal is about - listening to feedback, improving the user experience, and seeing where the pain points are.  Now that the original goals have been met, we'd also like to reconsider some previously out-of-scope goals as optional or stretch goals. Here's what we've heard, and what we're planning for the next three months.

# Technical Content

## Improved installation and CI support

One of the biggest pieces of feedback that we've received is the need for improved support for the installation of the `botan3` C++ library. This was a recurring item, and we've made it one of our priorities. Installation of the Botan C++ library is one of the largest warts in using these libraries - package distributions are available for some users and operating systems via `pkg-config`, but other users currently require manual installation. This is a one-time process, but it is still a significant pain point in what should otherwise be a much smoother process.

We'd like to spend a good chunk of time improving the installation process, with a specific focus on Windows[^1] and Linux support, working on the CI + unit tests for improved reliability and reporting overall.

[^1]: Definitely more highly requested than anticipated

This is a high-priority goal, alongside improvements to CI and unit testing, and will be a major focus. We're already looking into using `build-type: Configure` for bundling Botan C++ as a Haskell package for easy installation on all operating systems - we'd like for usage to be as easy as adding `botan` to your dependencies.

## Development of a drop-in interface replacement for `crypton`

This was mentioned several times in feedback. `crypton` is a dependency in many important libraries in the Haskell ecosystem, and we would like to build an interface that is as near a drop-in replacement for `crypton` as possible.

These libraries sit at the root of a lot of production haskell code, and their list of transitive dependents is quite large. Anything involving networking, APIs, and the internet is likely to depend on at least one of them:

- `amazonka-core`
- `hpack`
- `scotty`
- `servant`
- `stack`
- `tls`
- `x509`
- `warp`
- `websockets`

They are all directly or indirectly dependent on `crypton`, a fork of `cryptonite` containing significant quantities of unaudited C that must be maintained by the Haskell community after the original author became unavailable. This is a problem, and we're trying to change that with bindings to `botan`. Having a solid, reliable, and well-maintained cryptography libraries is a huge benefit to the Haskell ecosystem in general, and is essential for most commercial and industrial use cases.

There will be some differences, as `botan` doesn't necessarily support everything[^2] in the same way as `crypton` does, but we'll give it our best effort to make migration as simple as possible, through a drop-in replacement, migration guide, or other suitable adapter.

[^2]: There are a few things that `crypton` supports that `botan` doesn't, but also vice versa - `botan` supports things like modern post-quantum algorithms and `crypton` doesn't.

## Continued development of the cryptographic typeclasses and gold-standard modules

The development of [cryptographic typeclasses][cryptographic typeclasses] and gold-standard per-algorithm modules is ongoing, in an effort to improve per-algorithm type safety and ergonomics through more consistent handling of cryptographic primitives such as keys, nonces, and ciphertexts.

> This is also a necessity for the `crypton` drop-in replacement.

Due to the inadequacy of the ADT approach towards managing algorithms, cryptographic typeclasses and data families have been brought back into scope as a deliverable after being declared out-of-scope in the original proposal. While a first attempt has already been made to typeclassify the various cryptographic primitives, this interface will require extensive refinement and feedback over the coming months, and is likely to experience significant change before settling down.

It is expected that these typeclasses will result in improved per-algorithm type safety, and better ergonomics through more consistent handling of cryptographic primitives such as keys, nonces, and ciphertexts.

## Continued improvements of `botan`

While `botan-bindings` and `botan-low` are effectively frozen, small changes may still occur. However, `botan` will be seeing significant development as we continue to improve and respond to feedback, as there are still a great number of ways to improve the safety and user experience:

### Improved error handling

We'd like to reduce the amount of noise, and to use `HasCallstack` appropriately throughout the library.

### Secure memory erasure and limited lifespan objects with guaranteed cleanup

Ensuring that the instructions to securely erase memory are not compiled out is a historically complex subject. Botan has appropriate functionality for secure memory erasure, but it must be called manually. We'd like to take advantage of the secure memory erasure support by providing functions for creating temporary objects that are automatically zeroed when they go out of scope. This is important, we don't want sensitive data such as keys hanging around in memory after we are done using them.

### Integration with other libraries

We'd also like to investigate integrating with other libraries. In particular, we'd like to integrate with `mtl` and `random` in order to provide a `MonadRandomIO / NonDeterministic` monad and take advantage of the `Uniform` class to make sampling / generation of random data easy. This monad already exists in a primitive state in `botan`, but significant refinement is expected.

### Improvements to the binding generator functions

Finally, we'd also like to continue improving, unifying, and standardizing the `botan-low` factory methods used to generate bindings bindings, for improved low-level consistency overall. The techniques for binding to functions were refined over time, and have not been evenly applied, and there is a significant amount of duplicated code or one-off implementations. It would improve maintenance significantly if all of the buffer handling et al were in one place.

## Development of a high-level libsodium-like interface

We'd like to expose a high-level libsodium-like interface of selected best-in-class algorithms in order to make usage dead simple. We don't want you managing primitives yourselves - we want you calling a simple function purely or in an appropriate monad / transformer. This is bit of a stretch goal, however, in favor of focusing our efforts on primary goals such as replacing `crypton`.

# Timeline and Budget

We propose a timeline of 3 months for these new goals, with a budget of $7000 USD per month for one full-time engineer. Please see the [first proposal][first proposal] for more detail.

# Goals and Deliverables

## Primary deliverables

The following are the primary deliverables of this proposal, in order of priority:

- Completion and publishing[^3] version `0.0.1` of `botan`, and continued updates to the library suite as necessary
- Improved installation and CI support for more operating systems, with a focus on Windows and Ubuntu
- Development of one or more suitable ways of assisting users in migrating from `crypton` to `botan`, be it drop-in interface replacement, fork,and/or migration guide

[^3]: `botan` is a package candidate, and is currently undergoing pre-release polishing and refinemenet. Publishing and release of `0.0.1` is expected to occur with a few weeks at time of writing.

## Secondary deliverables

The following are secondary deliverables, to be considered as optional stretch goals to be completed if time permits after completion of primary deliverables

- Continued development of the cryptographic typeclasses and gold-standard algorithm modules
    - There is no set completion date on this, but progress should be made if possible
- Completion of the objectives in 'Continued improvements to `botan`'
- Development of a high-level libsodium-like interface

## Intermediate deliverables

The following are intermediate deliverables intended to keep the community apprised of any progress that is made, and any challenges that arise, and to seek community feedback:

- Frequent (2-3x weekly) activity update on devlog
- Weekly major update that includes github push
- Monthly report that summarizes what specific items have been worked on, what items will be worked on, and what challenges have arisen, if any.

# Stakeholders

The primary stakeholders of this proposal are:

- Leon Dillinger (Myself)
- Haskell Foundation
- Haskell Cryptography Group
- The Haskell Community
- Current and previous sponsors

 Please see the [first proposal][first proposal] for more detail.

 # Success

This proposal will be considered a success if the immediate and intermediate deliverables are met, and new versions of the libraries are released as necessary. The secondary stretch goals are not to be considered necessary for success, but should be worked on if time permits after completion of the primary goals.

# Future work

There are many things that are out-of-scope or are of lower priority than the the project goals already mentioned. The following are long-term goals that are likely to be revisited in a future proposal:

- Implementation of advanced algorithms such as Merkle Trees, Distributed JSON, [Signal's Double Ratchet][double ratchet], and [Apple's PQ3][apple pq3]
- Improving APIs with higher-order functions
- To investigate integration / migrating specific libraries to `botan`, such as `x509`, `tls`, `servant`, as an alternative to `crypton`
- Splitting off cryptographic classes as a separate package to be backend agnostic
- Building an application framework that takes care of cryptography & security

# Appendix: Retrospective

The [Botan Cryptography Community Project][botan repo] is a project that was [previously funded][first proposal] by the Haskell Foundation with the immediate goals of developing bindings to the [Botan C++ cryptography library][botan cpp], and the long term goals of developing a stable, open-source cryptography ecosystem for the Haskell community, with uses including data integrity, privacy, security, and networking.

This project successfully maintained a [devlog][devlog], produced several monthly reports [(1)][monthly report 1], [(2)][monthly report 2], [(3)][monthly report 3], 2 libraries which have been released to hackage: `botan-bindings`[(1)][botan-bindings] and `botan-low` [(2)][botan-low], and a third library `botan` [(3)][botan candidate] has entered package candidate status, and completed the set of milestones and deliverables laid out in the initial proposal.

This project was a moderate-to-high success, with room for improvement, mostly in terms of scoping and communication. The low-level `botan-bindings` and `botan-low` libraries were completed satisfactorily, with the high-level `botan` experiencing some minor delays due to a few factors.

Significant progress was made towards one of the stretch goals (extending the Botan C FFI to include full X509 support) and while we were initially optimistic that this stretch goal might be completed, this effort came at a opportunity cost towards required deliverables, and was ultimatly paused to avoid endangering them. 

In addition to this, the algorithm ADT approach (a core component of the high-level `botan` interface) turned out to be insufficient in practice. This was really only evident after having a first implementation, and some time was needed to come up with a better interface with much stronger typing, especially after feedback stressing the desire for per-algorithm type-safety.

Solving this required bringing back cryptographic typeclasses, which were originally and explicitly removed from the proposal due to being out-of-scope. This caused a noticable increase in scope, as it was necessary to bring them back in due to the insufficiency of the ADTs.

This ultimately ended up delaying work on the `botan` library after feedback indicated a strong desire to see more effort being spent on polishing, documentation, and release of `botan-bindings` and `botan-low` before continuing work on `botan`.

These issues and confounding factors could have been handled by better prioritization and scope management, and a more clear communication to the community of any effect on the project timeline. 

However, there are mitigating factors as well: significant progress was made towards a high-value stretch goal (Extended X509 suppport), and the deliverables were met despite the size and scope of the project being significantly larger than originally estimated.

As such, this proposal is slightly less aggressive in scope, to allow for a little more flexibility to account for shifting needs and feedback.

# Appendix: Links

- [First proposal][first proposal]
- [Botan repository][botan repo]
- [Botan C++][botan cpp]
- [Devlog][devlog]
- [Monthly report 1][monthly report 1]
- [Monthly report 2][monthly report 2]
- [Monthly report 3][monthly report 3]
- [botan-bindings package][botan-bindings]
- [botan-low package][botan-low]
- [botan package candidate][botan candidate]

<!-- Link references -->

[first proposal]: https://github.com/haskellfoundation/tech-proposals/pull/57 "First proposal"

[botan repo]: https://github.com/haskell-cryptography/botan "Botan repository"
[botan cpp]: https://botan.randombit.net/ "Botan C++"
[botan audit]: https://botan.randombit.net/releases/audit_1.11.18.pdf "Botan audit"

[devlog]: https://discourse.haskell.org/t/botan-bindings-devlog/6855 "Devlog"

[monthly report 1]: https://discourse.haskell.org/t/botan-cryptography-monthly-status-report-0/8280 "Monthly report 1"
[monthly report 2]: https://discourse.haskell.org/t/botan-cryptography-monthly-status-report-1/8497 "Monthly report 2"
[monthly report 3]: https://discourse.haskell.org/t/botan-cryptography-3rd-monthly-status-report/8754 "Monthly report 3"

[botan-bindings]: https://hackage.haskell.org/package/botan-bindings-0.0.1.0 "botan-bindings"
[botan-low]: https://hackage.haskell.org/package/botan-low-0.0.1.0 "botan-low"
[botan candidate]: https://hackage.haskell.org/package/botan-0.0.1.0 "botan candidate"

[cryptographic typeclasses]: https://discourse.haskell.org/t/botan-bindings-devlog/6855/ "Cryptographic typeclasses"
[double ratchet]: https://signal.org/docs/specifications/doubleratchet/ "Double-ratchet"
[apple pq3]: https://security.apple.com/blog/imessage-pq3/ "Apple PQ3"
