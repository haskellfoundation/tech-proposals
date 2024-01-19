# Botan Cryptography Community Project Extension

> NOTE: This document references an earlier proposal that is ongoing and close to meeting its goals. Language in this document may reflect a future perspective that assumes those goals have been met.

# Abstract

The Botan Cryptography Community Project is reaching the end of its funding on Jan 31, 2024. This proposal is for the extension of funding to and addition of new goals for the [Botan Cryptography Community Project](https://github.com/haskellfoundation/tech-proposals/pull/57), which is a previously funded proposal that is ongoing, and nearing completion of its initial goals. These new goals are for the continued refinement of the produced libraries, completion of the X509 C++ work, and many other small features and minutia. 

# Background

The [Botan Cryptography Community Project](https://github.com/apotheca/botan) is a project that was previously funded by the Haskell Foundation for the purpose of developing a suite of Haskell cryptography libraries and tooling suitable a wide range of uses including data integrity, privacy, security, and networking.

This project has maintained a [devlog](https://discourse.haskell.org/t/botan-bindings-devlog/6855) and produced several monthly reports [(1)](https://discourse.haskell.org/t/botan-cryptography-monthly-status-report-0/8280) [(2)](https://discourse.haskell.org/t/botan-cryptography-monthly-status-report-1/8497)

The project has produced 3 libraries: `botan-bindings`, `botan-low`, and `botan`.

This project also has made a substantial amount of progress towards one of its stretch goals of extending the Botan C FFI to include full X509 support. We were initially optimistic that this stretch goal might be completed by the end of initial funding, but the C++ work required more overhead than expected, and efforts towards it were paused to avoid endangering any required deliverables.

# Technical Content

Despite (being on pace for) fulfilling its goals, there is still more work to do, as the scope of the original proposal was deliberately restricted for the purposes of focus and achievability. Now that the original goals have been met, some of the previously out-of-scope goals may now be reconsidered.

## Continued improvements to `botan`

- `botan-bindings` and `botan-low` are effectively frozen, but small changes may still occur
- `botan` is likely to see significant churn as we continue to improve and respond to feedback

As `botan` is a very "fresh" library, it is expected to see some amount of churn in its API as it is developed and refined further. The initial goals of functionality are nominally complete, but there is still a great amount of minutia to be performed that is concerned with improving the safety and user experience. Expected improvements include limited lifespan objects with guaranteed cleanup, auto-zeroing of sensitive memory, and implementing instances from libraries such as `random`.

## Botan C++ development

- `Botan C++` is being actively developed, so project will need continued maintenance to achieve parity
- Several valuable features in `Botan C++` are not exposed by the C FFI, and require C / C++ development in order to expose them

One of the previous stretch goals was to develop the C bindings for X509 certificate architecture, and to [contribute that code back](https://github.com/randombit/botan/issues/3627) to `Botan C++`. Substantial progress was made, but further work is required in order to complete this. 

Another feature missing from the Botan C FFI is Stream Ciphers, and like the X509 bindings, it will require effort to extend the Botan C++ library in order to expose it to Haskell.

A third feature missing from the Botan C FFI is TLS, as Botan provides a full TLS server implementation, which will also require effort to expose to the FFI.

## Development of cryptographic abstractions

There are many 'class-y' improvements that could be made, abstractions that `botan` could implement instances for better ergonomics and better management of keys, nonces, and other cryptographic minutia. While many of these abstractions are not strictly necessary for `botan`, they do hold value, and investigations into the development of an abstract hierarchy and how it might be applied to botan and other cryptographic libraries is ongoing.

# Goals

The following are the immediate goals / deliverables of this proposal:

    - To continue to refine the `botan` libraries
    - To complete the X509 C / C++ FFI and Haskell bindings
    - To begin development of the Stream Cipher C / C++ FFI and Haskell bindings
    
The following are intermediate deliverables:

    - Frequent activity update on devlog
    - Weekly major update that includes github push
    - Monthly report that summarizes what specific items have been or will be worked on, and what challenges have arisen, if any.

The following are stretch goals / optional deliverables:
    
    - To complete development of the Stream Cipher C / C++ FFI and Haskell bindings
    - To develop the full TLS server C / C++ FFI and Haskell bindings
    - To investigate integrating `botan` into libraries such `x509`, `tls`
    - To develop abstract classes / hierarchy for cryptography and cryptosystems
    - To develop tutorials for integrating with other libraries such as `servant`
    
# Timeline and Budget

We propose a timeline of 3 months for these new goals, with a budget of $7000 USD per month for one full-time engineer, the same as the original proposal.
    
# Stakeholders

The primary stakeholders of this proposal are:

- Leon Dillinger (Myself)
- Haskell Foundation
- Haskell Cryptography Group

# Success

This proposal will be considered a success if the immediate and intermediate deliverables are met, and at least one of the stretch goals has seen progress.
