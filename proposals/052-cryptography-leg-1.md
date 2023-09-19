# Cryptography Community Project: Leg 1 Proposal

# Abstract

This community project proposal is for full-time funding for the development of a suite of Haskell cryptography libraries and tooling suitable a wide range of uses including data integrity, privacy, security, and networking. This is necessary to relieve the burden [of correctly implementing cryptography] from Haskell developers seeking to provide privacy and security to their users. We propose the development of a community-owned set of cryptography libraries of increasing capability, starting with bindings to a compatible open-source cryptography library. This is expected to require a considerable effort over time, and if accepted will be broken up into several legs, each of which will be accompanied by its own specific proposal and set of goals.

The first leg of this proposal is for the development of high-quality, production-ready bindings to the Botan C++ cryptography library, and from acceptance to release on Hackage is estimated to take 3 months of full-time development. We see several challenges but no significant technical risk to this proposal; known challenges involve strictness and developing additional bindings to C++; these items are of high-value and may take significant effort to resolve, but are known to be solvable. Considerable progress has already been made, and the proposed funding will help ensure its completion.

# Background

Cryptography is an essential part of modern computing and information security, but it requires care, nuance, and knowledge to properly implement, and is extremely easy to mess up. It covers a diverse range of functions, with individual algorithms often having their own particular quirks that must be acknowledged to properly and safely use. Failure to properly implement cryptography carries the risk of severely compromising the security of any system in which it is used. Manually implementing common cryptographic primitives such as hashes, ciphers, digital signatures, big-integer arithmetic is thus considered extremely unwise, both in terms of security and performance.

Cryptography is even harder for functional programming languages. Properly implementing techniques such as sanitizing memory, performing operations in constant time, are already difficult enough in languages such as C, and in a lazy functional language it is made even more difficult by such things as the potential for references to be kept alive within thunks. As a result, the space of functional cryptography is relatively under-explored and under-developed, having favored other languages where such things are easier to manage.

However, if we accept this downside as a part of a trade-off, and explore the ways that functional programming is beneficial to cryptography (monads, type systems, referential transparency), there is then also the potential for functional cryptography to be made *easier*. The functional machinery of Haskell is well-suited for adding important contextual control to cryptography, and can help prevent or eliminate many issues and errors in ways that languages such as C cannot. I believe that there is room for significant improvement, given a directed effort to develop the space.

# Problem Statement

Cryptography in Haskell lacks significant capability beyond basic primitives. This places a significant burden on developers to properly implement various security techniques, and exposes end users to significant risk in the event of a lapse in security. Some companies have built their own solution to this, but there is no community-driven, community-owned solution.

Cryptography in Haskell is also fragile and outdated, having seen considerable flux / churn over the years as various libraries have been developed and then deprecated or abandoned in favor of newer ones. This has placed the long-term stability of important libraries such as `tls` and `x509` at risk of falling behind as compilers are upgraded, old standards are updated, and new standards are implemented.

There are several reasons for this:

1) Cryptography involves a lot of stateful low-level bit twiddling and random IO, something Haskell is not known for being good at. Classes like `Bits` and `FiniteBits` classes are critical for cryptography, but are terribly awkward to implement (like `Num`). Also, `ByteString` does not implement them for nuanced reasons.

2) Cryptography has a high knowledge and technical skill floor. Cryptography is more than just the usual primitives of ciphers, public and private keys, hashes, and MACs, and the scope of knowledge can grow quickly as primitives are combined into more advanced algorithms.

3) The notion that functional programming is "bad at cryptography" (ie, underperformant and has insurmountable challenges) is a self-fulfilling prophecy, and will remain so unless we invest in enough effort to disprove it.

4) A fair lot of cryptographic projects in recent years has been specifically towards *cryptocurrencies*, rather than *cryptography in general*, and their results are at large fairly useless if they cannot be used outside of their ecosystems.

All of these reasons have contributed to the stagnant state of Haskell cryptography, and have make it an unfavorable environment for developers of cryptosystems and distributed systems. However, none of these are actual blockers, and each of these things can be corrected with effort. There is much that can be done to improve the Haskell cryptography ecosystem, ranging from better low-level support, to higher-level abstractions:

- Up-to-date algorithms
- Hierarchy of typeclasses
- Monadic interfaces
- Safety abstractions
- Type-level composition
- Advanced data structure (eg, merkle trees)
- Use of Generic for crypto-serialization

As a result, there is much to be gained from investing in foundational Haskell cryptography. Haskell is well-suited to providing many aspects of information security through use of monadic interfaces and the type system, if the lower-level challenges of strictness and performant primitives can be solved. The community needs a cryptography library that is easily maintained, with minimal dependencies / baggage, and with documentation, tutorials, and unit tests to ease development.

A first step would be developing bindings to a suitably-licensed open-source C or C++ library containing modern and up-to-date cryptography algorithms, including post-quantum cryptography schemes.

There are [several C/C++ library candidates](https://en.wikipedia.org/wiki/Comparison_of_cryptography_libraries):

- Botan
- libgcrypt
- Crypto++
- libsodium / NaCl
- NSS

Of these, Botan was selected for several reasons:

- It has a BSD license
- It has a C FFI
- It offers a broad variety of functionality and algorithms, including post-quantum cryptography
- It has been [audited](https://botan.randombit.net/releases/audit_1.11.18.pdf) in the past
- There are dead bindings targeting Botan 2 / GHC 8 that can be examined / revived
- Significant implementation work has already been performed in order to validate the feasibility of Botan

The others were not selected for several reasons:

- libgcrypt is C so it requires no FFI, but it has a GNU LGPL license, and provides less functionality than Botan
- Crypto++ has a Boost license (similar to BSD), but is C++ and has no C FFI, and provides less functionality than Botan
- libsodium / NaCL does not provide the necessary functionality and there are several Haskell libsodium bindings already
- NSS has a copyleft license

Bindings to Botan would solve the significant problem of dealing with common cryptography by providing much of the necessary 'cryptographic kitchen sink' via bindings to a suitably performant, suitably licensed, open-source library. The Haskell community would not be required to maintain the Botan cryptography library, only bindings to it, and this can be accomplished readily with some effort which has already begun. Furthermore, the Botan library supports several post-quantum cryptography algorithms, which is highly desirable.

It is important that the Haskell community has a strong cryptography library / presence that is not bent towards a particular agenda. It must be **free as in speech (licensing)** and **free as in beer (no mining)**, in order to allow the growth of advanced cryptographic techniques the way that Haskell has furthered the growth of functional programming techniques.

Furthermore, bindings to a dedicated cryptography library may yield a performance benefit, as one would expect a popular C++ crypto library such as Botan to be highly performant. Testing in GHCi with `:set +s` for a requested example of `bcrypt` yielded the following results:

```
-- Botan
ghci> bcryptGenerateIO "Fee fi fo fum!" r 16
"$2a$16$j3Obv.HNT8zQ9OXFXEJeQeq3nT5WmUecPYDF0tc3grNsX9N0uDSCm\NUL"
(3.73 secs, 996,152 bytes)

-- cryptonite
ghci> hashPassword 16 ("Fee fi fo fum!" :: ByteString) :: IO ByteString
"$2b$16$nmHHDe/wnfSl85dIFtqjMuAHxqAmL.JQ6knp8e6kf.X1V9/LCZxUy"
(8.84 secs, 2,138,266,840 bytes)
```

It was quick and dirty testing in GHCi so your mileage may vary, but the results indicate that bindings to Botan may be significantly faster and consumes less memory than `crypton/ite`, if other modules / functions are similarly performant. Further implentation, testing, and benchmarking will be necessary to confirm this.

# Prior Art and Related Efforts

We can group Haskell cryptography efforts into several categories, which are not necessarily distinct:

- Libraries for specific algorithms
- Cryptographic "kitchen sinks"
- Bindings to foreign libraries
- Efforts by [Thomas DuBuisson](https://hackage.haskell.org/user/ThomasDuBuisson)
- Efforts by [Vincent Hanquez](https://hackage.haskell.org/user/VincentHanquez)

The wiki lists [several libraries](https://wiki.haskell.org/Applications_and_libraries/Cryptography) and Hackage lists [many more](https://hackage.haskell.org/packages/#cat:Cryptography) according to the category.

> NOTE: Cryptocurrency libraries and libraries not on Hackage are excluded from consideration.

Specific libraries worth noting are:

- Predecessors to `cryptonite` - all deprecated
    - [cryptohash](https://hackage.haskell.org/package/cryptohash)
    - [cryptocipher](https://hackage.haskell.org/package/cryptocipher)
    - [crypto-pubkey](https://hackage.haskell.org/package/crypto-pubkey)
    - [crypto-random](https://hackage.haskell.org/package/crypto-random)
- Cryptographic "kitchen sinks"
    - [cryptonite](https://hackage.haskell.org/package/cryptonite)
    - [crypton](https://hackage.haskell.org/package/cryptonite) - a fork of cryptonite
- Bindings to libsodium
    - [NaCl](https://github.com/serokell/haskell-crypto)
    - [saltine](https://hackage.haskell.org/package/saltine)
    - [sel](https://hackage.haskell.org/package/sel-0.0.1.0/candidate)
- Bindings to botan
    - [Z-Botan](https://hackage.haskell.org/package/Z-Botan) - part of Z-Haskell, GHC8
- Honorable mentions
    - [Crypto](https://hackage.haskell.org/package/Crypto) - a very old sink
    - [crypto-api](https://hackage.haskell.org/package/crypto-api) - cryptographic classes
    - [cryptol](https://hackage.haskell.org/package/cryptol) - a DSL for developing new cryptographic primitives

These prior efforts can be considered to be somewhat partially successful in presenting Haskell developers with access to cryptographic primitives, but present a significant development challenge and a high refactoring cost if dependences become deprecated, which happens often. Of these libraries, a considerable amount are deprecated or have been dead for many years, with `cryptonite`, `crypton`, and `saltine` being clear front-runners in terms of current usage and transitive dependencies (and `sel` being under active development). Of these four, one is a fork of another (`crypton` and `cryptonite`), and the other two are bindings to `libsodium` which does not sufficiently capture our need for low-level cryptographic primitives in general.

None of these sufficiently express cryptography through the type-system and with type-classes. `crypton/ite` exposes a few highly- opinionated / specific classes for hashing, block and stream ciphers, but otherwise is focused on concrete implementations and functions. Both `crypton` and `cryptonite` rely on bindings to C implementations of unknown veracity; though the code is itself not suspect, the lack of provenance makes it ill-suited for recommendation in trusted environments, and maintenance of the C implementation poses burden of high technical skill. Libsodium bindings are more trustworthy, but the limited options severely hamper interop with other systems unless they too are using libsodium, making them ill-suited for recommendation for some use cases. `Z-Botan` has been apparently dead for several years, and even if revived has a has a sizeable dependency in the form of the `Z-Haskell` ecosystem.

Furthermore, none of these libraries contain implementations of post-quantum cryptography schemes; although existing quantum computers are of insufficient capability to break commonly used algorithms such as Ed25519, this will not remain true forever, and it would be best to have quantum-resistant implementations ready for adoption long before such attacked become practical.

Each of these efforts has one or more of these issues that preclude them from being considered a complete success:

- Insufficient abstraction
- Maintenance of handwritten C
- Bindings to an insufficiently general library
- Burdensome dependencies
- Dead project
- Lack of post-quantum algorithms

However, there is also a lot to be learned from the ways these prior efforts *were* successful. This proposal will assess these existing efforts in an attempt to consolidate their learnings and functionality.

## An example of a successful ecosystem

The health of the Haskell server ecosystem may be used as an example of a successful community-driven niche, and we can aspire to bring that same quality and health to the Haskell cryptography ecosystem. There are many pertinent parallels in terms of managing performance, complexity, and safety that we can learn from:

- Low-level packages like `wai` supporting multiple backends
- Backends range from simple like `scotty` to complex like `servant`
- Frontends like `blaze-html` and `lucid`
- Libraries can take advantage of type system to provide good abstractions
- Flexible enough abstractions to explore the problem space with multiple solutions
- Appropriate licensing

`servant` (and the `wai` ecosystem in general) is a success story and an example of higher-order programming making something safer and easier, something that we can aspire to do with cryptography.

## Technical Content

This proposal is for the contruction of a suite of cryptography libraries, ranging from low-level bindings to specific algorithms, to higher-level abstractions and typeclasses.

This leg of the proposal focuses on bindings to the `botan` cryptography, in the form of the following libraries:

### botan-bindings

The `botan-bindings` library will contain raw bindings and is an almost direct, 1-1 translation of the C API into Haskell FFI calls. As such, it exposes and operates over C FFI types, requires buffer and pointer and pointer management, and returns error codes. This library only exposes FFI calls and constants, and is suitable for building your own abstraction over Botan.

### botan-low

The `botan-low` library will contain low-level bindings which wrap the FFI calls into monadic IO actions. This library handles the translation between buffers and ByteStrings, and throws exceptions in the case of errors, but is otherwise a fairly faithful translation of the interface. Thus this library is suitable for use, but is not recommended for day-to-day programming due to statefulness and a lack of type safety.

### botan

The `botan` library will contain high-level bindings which hides the stateful IO and exposes appropriately idiomatic and pure interfaces. It will provide data types for algorithms, enumerations for constants, convenience methods for key and nonce generation, incremental processing of lazy bytestrings, and otherwise bring significant safety and ergonomics over `botan-low` and `botan-bindings`.

## Optional / Future Technical Content

More advanced concepts such as typeclasses may be delivered in a future leg focused on generalization, but some development occur earlier in order to provide a testable 'use case' for the stack of libraries. If the first leg is completed ahead of schedule, the remaining effort will go towards these libraries.

### crypto-schemes

The `crypto-schemes` library will contain a hierarchy of abstract type classes and data families for cryptography and scheme identifiers, allowing for a unified interface to multiple back-ends. It will also provide various utility functions including sized bit and byte vectors, block and padding functions.

### crypto-schemes-botan

The `crypto-schemes-botan` library will be an implementation of `crypto-schemes` using `botan`, implementing class instances for the specific algorithms that `botan` provides.

### botanium

The `botanium` library will contain user-friendly bindings to specific algorithms, designed for ease of use much like `libsodium`, but using `botan` or `crypto-schemes-botan` as a backend.

### botanite

The `botanite` library will contain an as-close-as-possible reimplementation of `crypton/ite` using `botan` or `crypto-schemes-botan` as a back-end, and may be kept as a separate library or merged into `crypton` at a future time.

## Goals and Non-goals

The following are the immediate goals of this first leg:

- To publish the `botan`, `botan-low`, and `botan-bindings` libraries
- To build proper documentation and tutorials covering the subject material
- To establish a community project presence focused on Haskell and cryptography

The following are overarching goals, and will not be complete by this first leg:

- To publish the `botanium` libsodium- and `botanium` crypton- compatible libraries
- To publish the `crypto-schemes` library and `crypto-schemes-botan` backend.
- To provide a suitable alternative interface / backend (as an option) for `crypton`, `tls`, `x509`
- To consolidate Haskell cryptography into a consistent, unified, user-friendly system
- To enable the safe use and implementation of advanced cryptographic data structures
- To provide cryptographic typeclasses for safe and consistent use

The following are not goals:

- Finishing every goal by the first leg of the project
- Matching interfaces 1:1 with existing libraries / interfaces
- Implementing every algorithm permutation possible

## Timeline

This timeline will focus on the first leg of the proposal, and the immediate goal of publishing the following libraries:

- botan-bindings
- botan-low
- botan

The full project has a large scope, and will require significant time to complete. More advanced concepts such as typeclasses may be delivered in a future leg focused on generalizing.

### Expected project leg duration

We estimate that it will take 3 months for this leg, to achieve our goal of developing and publishing these libraries. This estimate is based on the significant amount of work already completed, the length of time in which it was accomplished, and the remaining amount of work.

Several work items (strictness, memory safety, C++ shims for missing FFI functionality) are difficult to estimate, and so time has been allocated for them in the estimate, but their delivery is considered optional, and may roll over into a future leg as necessary.

### Intermediate deliverables

The following are intermediate deliverables intended to keep the community apprised of any progress that is made, and any challenges that arise, and to seek community feedback:

- Frequent (2-3x weekly) activity update on devlog
- Weekly major update that includes github push
- Monthly report that summarizes what specific items have been worked on, what items will be worked on, and what challenges have arisen, if any.

### Deliverables

The following are the intended deliverables of this leg of the proposal:

- `botan-bindings`
- `botan-low`
- `botan`

Each library should be beta- or / production-ready, and submitted to hackage as a candidate or accepted package. 

Each library should have:

- Documentation
- Tutorials and examples
- Tracked issues
- Unit tests

A project website will also be created.

The official repo will be owned by the Haskell Cryptography Group, and another member of the Haskell Cryptography Group will be selected to hold backup keys and permissions to the website, server, and official repositories as needed, in order to give project members access and to reduce any conflicts about ownership or maintenance.

## Budget

The first leg of this proposal presents a minimum budget of $7000 USD per month, for one full-time engineer, for a duration of 3 months, for a total of $21,000 USD. This budget is based on cost-of-living and industry experience, and will cover housing, food, bills, taxes, and other life necessities for one engineer, as well as any project necessities such website and server hosting. This budget is roughly equivalent to $40 / hr or $84k / yr at full-time of 40 hrs / wk. Industry rates for an engineer of the necessary skill are on average considerably higher, and so we consider this budget to be reasonable. The exact legal contract / arrangement is left to the Haskell Foundation.

The question of continued funding will be revisited for each future leg of this proposal, wherein the budget and direction of future work will be decided again for each leg.

## Stakeholders

Several parties stand to gain from the implementation of this proposal.

- Leon Dillinger (Myself)

As the intended engineer carrying out the work, I have already invested significant effort in this project in order to establish its viability, and will benefit from this proposal in the form of being the recipient of the funding.

- [Haskell Foundation](https://haskell.foundation/)

As the intended source of funding, the Haskell Foundation would be invested in this proposal financially, and its effective success or failure would reflect respectively upon the Haskell Foundation.

- [Haskell Cryptography Group ](https://github.com/haskell-cryptography)

As the intended owner of the project, this project will be new maintenance burden on the Haskell Cryptography Group.

- Users of existing cryptography libraries such as `crypton`, `cryptonite`, `libsodium / saltine`.

As the intended target audience of this project, this proposal aims to directly benefit developers by providing high-quality cryptography libraries at several appropriate levels of abstraction, but will also place the burden of migration to these new libraries on developers.

- Downstream users of Haskell software, developers with 

As the eventual user of the developed software, their safety and security will rest on the proper execution of this proposal. 

## Risks

There are a number of risks.

- Cryptography is hard and each algorithm may have individual nuances or uncommon requirements. 
- Botan FFI is incompletely documented and has gaps (stream ciphers, X509 stores, a few broken functions) which need to be fixed with C++ shims
- Strictness, memory safety & scrubbing, C++ shims will take an unknown amount of time
- A thorough / safe implementation will require consulting the C++ source of the FFI
- Polynomial combinations of algorithms makes exhaustive testing complicated.

These risks are amortized in that while they are tedious and potentially time-consuming, they may be considered extended goals of high value, and there is still substantial value that can be delivered without them, and the completion of risk items may continue in a future proposal leg.

- Single point of failure

This is a proposal for one full-time engineer; this constitutes a single point of failure. To combat this, the official github repo will be owned by the Haskell Cryptography Group, and another member of the Haskell Cryptography Group will be selected to hold backup keys and permissions to the website, server, and official repositories as needed. In the case of an emergency for which I am unreachable, this will allow for others to take over as necessary.

## Success

This leg of the project will be considered a success if it results in the submission of a candidate or accepted package to hackage, with production-ready bindings, documentation, tutorials, and unit tests as appropriate for the following botan-related libraries:

- botan-bindings
- botan-low
- botan

The following Botan library features are considered optional to the success of this leg, due to requirement of C++ / FFI shims or additional research which will take an unknown amount of work, but effort will be made to include them if possible within the timeframe. 

- Extended X509 support
- Stream ciphers
- Test vectors for algorithms, especially [CAVP / FIPS / NIST-approved](https://csrc.nist.gov/projects/cryptographic-algorithm-validation-program)

The following libraries are considered non-essential to the success of this leg, but may be worked on as they provide a valuable use case with which to test the lower libraries against, and may be subsumed into another at any time (eg, botanite could be merged into crypton).

- botanium
- botanite
- crypto-schemes
- crypto-schemes-botan

The next leg of this proposal is still undetermined, but tentatively will be focused on compatibility with `crypton` through `botanite`, and on the development of abstract cryptography classes in `crypto-schemes`.
