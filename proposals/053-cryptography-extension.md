# Botan Cryptography Community Project Extension

> NOTE: This document references an earlier proposal that is ongoing and close to meeting its goals. Language in this document may reflect a future perspective that assumes those goals have been met.

# Abstract

The Botan Cryptography Community Project has reached the end of its funding on Jan 31, 2024. This proposal is for the extension of funding to and the addition of new goals for the [Botan Cryptography Community Project](https://github.com/haskellfoundation/tech-proposals/pull/57), which is a previously funded proposal.

These new goals are for the continued refinement of the produced libraries, the continuation of C++ work, the development of cryptographic typeclasses and data families, improvements to the installation & CI pipeline, increased safety measures such as memory scrubbing and conditional compilation for optional modules, data types and instances for libraries such as `random` and `mtl`, and continued support of existing code.

# Background / Retrospective

The [Botan Cryptography Community Project](https://github.com/apotheca/botan) is a project that was previously funded by the Haskell Foundation for the purpose of developing a suite of Haskell cryptography libraries and tooling suitable a wide range of uses including data integrity, privacy, security, and networking.

This project has maintained a [devlog](https://discourse.haskell.org/t/botan-bindings-devlog/6855) and produced several monthly reports [(1)](https://discourse.haskell.org/t/botan-cryptography-monthly-status-report-0/8280) [(2)](https://discourse.haskell.org/t/botan-cryptography-monthly-status-report-1/8497) [(3)](https://discourse.haskell.org/t/botan-cryptography-3rd-monthly-status-report/8754)

The project has produced 2 libraries which have been released to hackage: `botan-bindings`[(1)](https://hackage.haskell.org/package/botan-bindings-0.0.1.0), `botan-low` [(2)](https://hackage.haskell.org/package/botan-low-0.0.1.0).

A third library `botan` [(3)\*](https://hackage.haskell.org/package/botan-0.0.1.0) is entering package candidate status soon, which will complete the set of milestones laid out in the initial proposal.

> \* Package candidate link to `botan` is not yet live at time of writing.

This project was a moderate-to-high success, with room for improvement, mostly in terms of scoping and communication. The low-level `botan-bindings` and `botan-low` libraries were completed satisfactorily, with the high-level `botan` experiencing some minor delays due to a few factors.

Significant progress was made towards one of the stretch goals (extending the Botan C FFI to include full X509 support) and while we were initially optimistic that this stretch goal might be completed, this effort came at a opportunity cost towards required deliverables, and was ultimatly paused to avoid endangering them. 

In addition to this, the algorithm ADT approach (a core component of the high-level `botan` interface) turned out to be insufficient in practice. This was really only evident after having a first implementation, and some time was needed to come up with a better interface with much stronger typing, especially after feedback stressing the desire for per-algorithm type-safety.

Solving this required bringing back cryptographic typeclasses, which were originally and explicitly removed from the proposal due to being out-of-scope. This caused a noticable increase in scope, as it was necessary to bring them back in due to the insufficiency of the ADTs.

This has ultimately ended up delaying the `botan` library by a few weeks after feedback indicated a strong desire to see more effort being spent on polishing, documentation, and release of `botan-bindings` and `botan-low` before continuing work on `botan`.

These issues could have been handled by better prioritization and scope management, and a more clear communication of any effect on the project timeline. As such, this proposal is slightly less aggressive in scope, to allow for a little more flexibility to account for shifting needs and feedback.

# Technical Content

There is still more work to do, as the scope of the original proposal was deliberately restricted for the purposes of focus and achievability. Now that the original goals have been met, some of the previously out-of-scope goals may now be reconsidered.

## Continued improvements to `botan`

While `botan-bindings` and `botan-low` are effectively frozen, small changes may still occur. However, `botan` is likely to see significant churn as we continue to improve and respond to feedback.

As `botan` is a very "fresh" library, it is expected to see some amount of churn in its API as it is developed and refined further. The initial goals of functionality are nominally complete, but there is still a great amount of ways to improve the safety and user experience.

Expected improvements include:

- Managed / limited lifespan objects with bracket / guaranteed cleanup
- Auto-zeroing of sensitive memory
- Implementing data types and instances for libraries such as `random` and `mtl`
- Conditional compilation suport for optional modules
- Improvements to the `Make / Remake` factory methods used to generate bindings

## Botan C++ development

`Botan C++` is being actively developed, so this project will need continued maintenance in order to achieve continued parity. Furthermore, several valuable features in `Botan C++` are not exposed by the C FFI, and require C / C++ development in order to expose them:

- Stream ciphers
- Complete X509 infrastructure implementation
- Complete TLS server implementation

One of the previous stretch goals was to develop the C bindings for X509 certificate architecture, and to [contribute that code back](https://github.com/randombit/botan/issues/3627) to `Botan C++`. Substantial progress was made, but further work is required in order to complete this. 

Another feature missing from the Botan C FFI is Stream Ciphers, and like the X509 bindings, it will require effort to extend the Botan C++ library in order to expose it to Haskell.

A third feature missing from the Botan C FFI is TLS, as Botan provides a full TLS server implementation, which will also require effort to expose to the FFI.

## Development of cryptographic typeclasses and data families

Due to the insufficiency of the ADT approach towards managing algorithms, cryptographic typeclasses and data families have been brought back into scope as a deliverable after being declared out-of-scope in the original proposal.

While a first attempt has already been made to typeclassify the various cryptographic primitives, this interface will require extensive refinement and feedback over the coming months, and is likely to experience significant change before settling down.

It is expected that these typeclasses will result in improved per-algorithm type safety, and better ergonomics through more consistent handling of cryptographic primitives such as keys, nonces, and ciphertexts.

## Improvements to installation / CI / Unit tests

One of the largest warts in using these libraries is the installation of the Botan C++ library. Package distributions are available for some users and operating systems via `pkg-config`, but other users currently require manual installation.

This is a one-time thing, but it still is a significant pain point in what should otherwise be a much smoother process. As such, research into packaging or bundling Botan C++ is ongoing, with multiple potential solutions already having been identified.

This is a high-priority goal, alongside improvements to CI and unit testing, and will be a major focus.

# Goals

The following are the immediate goals / deliverables of this proposal:

    - To continue to refine the `botan` libraries
    - To continue improvements to the Botan C FFI and Haskell bindings
    - To develop an abstract cryptographic typeclass / data family hierarchy exhibiting per-algorithm safety
    - To improve Botan installation / CI ease of use

The following are intermediate deliverables:

    - Frequent activity update on devlog
    - Weekly major update that includes github push
    - Monthly report that summarizes what specific items have been or will be worked on, and what challenges have arisen, if any.

The following are stretch goals / optional deliverables to be looked into only after required deliverables are met:
    
    - Development of the Stream Cipher C / C++ FFI and Haskell bindings
    - Completion of the extended X509 C / C++ FFI and Haskell bindings
    - Beginning development the full TLS server C / C++ FFI and Haskell bindings
    - To investigate integrating `botan` into libraries such `x509`, `tls`
    - To develop tutorials for integrating with other libraries such as `servant`
    
# Timeline and Budget

We propose a timeline of 3 months for these new goals, with a budget of $7000 USD per month for one full-time engineer, the same as the original proposal.
    
# Stakeholders

The primary stakeholders of this proposal are:

- Leon Dillinger (Myself)
- Haskell Foundation
- Haskell Cryptography Group

# Success

This proposal will be considered a success if the immediate and intermediate deliverables are met, new versions of the libraries are released as necessary, and at least one of the stretch goals has seen progress.
