# Botan Cryptography Community Project Proposal

# Abstract

This community project proposal is for full-time funding for the development of a suite of Haskell cryptography libraries and tooling suitable a wide range of uses including data integrity, privacy, security, and networking. This is necessary to relieve the burden of correctly implementing cryptography from Haskell developers seeking to provide privacy and security to their users. We propose the development of a community-owned set of cryptography libraries of increasing capability, starting with bindings to a compatible open-source cryptography library. This is expected to require a considerable effort over time, and will be broken up into individual proposals, each of which will be accompanied by its own  specific set of goals.

This first proposal is for the development of high-quality, production-ready bindings to the Botan C++ cryptography library, and from acceptance to release on Hackage is estimated to take 3 months of full-time development. We see several challenges but no significant technical risk to this proposal; known challenges involve strictness and developing additional bindings to C++; these items are of high-value and may take significant effort to resolve, but are known to be solvable. Considerable progress has already been made, and the proposed funding will help ensure its completion.

# Background

Cryptography is an essential part of modern computing and information security, but it requires care, nuance, and knowledge to properly implement, and is extremely easy to mess up. It covers a diverse range of functions, with individual algorithms often having their own particular quirks that must be acknowledged to properly and safely use. Failure to properly implement cryptography carries the risk of severely compromising the security of any system in which it is used. Manually implementing common cryptographic primitives such as hashes, ciphers, digital signatures, big-integer arithmetic is thus considered extremely unwise, both in terms of security and performance.

Cryptography is even more difficult for functional programming languages. Properly implementing techniques such as sanitizing memory, performing operations in constant time, and zeroing and ejecting from caches / registers, are already a challenge in a strict imperative language such as C, and in a lazy functional language it is even more difficult. Lazy evaluation presents a challenge towards managing and clearing sensitive data from memory, as care must be taken in order to ensure that it is not captured as part of a closure or thunk, and to force finalizers / cleanup appropriately or else the data may stay in memory for longer than intended. As a result, the space of functional cryptography appears to be relatively under-explored and under-developed, having favored other languages where such things are easier to manage.

However, if we accept this downside as a part of a trade-off, and explore the ways that functional programming is beneficial to cryptography, there is then also the potential for functional cryptography to be made *easier*.

The collective properties of referential transparency, type systems, and monadic interfaces provide a safer and more lawful context in which to write functions than the free-for-all that many imperative / OOP languages can offer, and so the functional machinery of Haskell is well-suited for adding important measures of safety and control to cryptography, helping to prevent or eliminate many issues and errors in ways that languages such as C cannot. I believe that there is room for significant improvement, given a directed effort to develop the space.

# Problem Statement

The Haskell cryptography ecosystem lacks significant capability beyond basic primitives. This places a significant burden on developers to properly implement various security techniques, and it exposes end users to significant risk in the event of a lapse in security. There may be commercial solutions, but there is no community-driven, community-owned solution, and one is unlikely to be developed without deliberate community effort (see Appendix A - Commercial Incentive Misalignment).

The Haskell cryptography ecosystem is also brittle and outdated. It is the product of considerable flux / churn over the years as various libraries have been developed and then deprecated or abandoned in favor of newer ones. The resulting libraries have highly-concrete interfaces and are strongly influenced by their implementation, both violating the abstraction barrier and preserving the design across successive libraries - as if their backend were to be changed, it would likely require changing or breaking the interface in turn.

As a result, there are multiple cryptography libraries with similar interfaces, and it is not always clear which should be used preferentially. This promotes rot as downstream users unknownly rely on unmaintained libraries that slowly stop working.

Furthermore, a number of important libraries in the cryptography ecosystem (eg, `cryptonite`, `asn1-encoding`, and `memory`) have recently been archived by their author and will no longer be receiving updates. Their backend is written in C, which presents a long-term maintenance burden with a high knowledge and skill requirement. NaCl bindings such as `saltine` are a high-quality alternative, but solve only a limited set of problems.

The resulting brittleness has placed the long-term stability of important libraries such as `crypton`, `tls` and `x509` at risk of falling behind as compilers are upgraded, old standards are updated, and new standards are implemented. Additionally the instability creates an unattractive environment for developers, effectively driving them away to other ecosystems that can provide the necessary stability.

There are several additional reasons for this lack of suitable engineers:

1) Cryptography involves a lot of stateful low-level bitwise manipulation and non-determinism, something Haskell is not known for being good at. Classes like `Bits` and `FiniteBits` classes are critical for cryptography, but are terribly awkward to implement (like `Num`). Notably, `ByteString` does not implement them for nuanced reasons (see Appendix B - Lack of ByteString `Bits` instance).

2) Cryptography has a high knowledge and technical skill floor. Cryptography is more than just the usual primitives of ciphers, public and private keys, hashes, and MACs, and the scope of knowledge can grow quickly as primitives are combined into more advanced algorithms.

3) There is a persistent notion that functional languages are too slow for cryptography, despite it being based on historical reasons that have been mostly since addressed. Well-written modern Haskell approaches C in terms of speed, and has a much higher ergonomic-use and type-safety factor, but the public image has not caught up yet, and so the ecosystem has not received the investment and attention necessary to attract the attention of skilled engineers for developing either pure implementations or foreign bindings.

4) A fair number of cryptographic projects in recent years has been specifically towards *cryptocurrencies*, rather than *cryptography in general*, and their results are at large fairly useless if they cannot be used outside of their ecosystems.

All of these reasons have contributed to the stagnant state of Haskell cryptography, and have make it an unfavorable environment for application developers, especially developers of cryptosystems and distributed systems. However, none of these are actual blockers, and each of these things can be corrected with effort. There is much that can be done to improve the Haskell cryptography ecosystem, ranging from better low-level support, to higher-level abstractions:

- Up-to-date algorithms
- Idiomatic interfaces
- Type safety
- Memory safety abstractions
- Advanced data structures (eg, Merkle trees)
- Use of Generic for crypto-serialization

<!-- As a result, there is much to be gained from investing in foundational Haskell cryptography. Haskell is well-suited to providing many aspects of information security through use of type-safe interfaces and the type system, if the lower-level challenges of strictness and performant primitives can be solved. The community needs a cryptography library that is easily maintained, with minimal dependencies / baggage, and with documentation, tutorials, and unit tests to ease development. -->

As a result, there is much to be gained from investing in foundational Haskell cryptography. Haskell is well-suited to providing information security through use of type-safe interfaces, and the community needs a cryptography library that is easily maintained, with minimal dependencies / baggage, and with documentation, tutorials, and unit tests to ease development. 

There are several factors to consider:

1) Resistance to modern (eg, side channel) attacks requires very low-level control over the implementation and computer, while Haskell is high-level and more concerned with abstraction and hiding the implementation
2) Trustworthy crypto libraries are extremely expensive to develop, but bindings are much cheaper
3) Haskell's C FFI can connect well to C implementations, and allows us to build an idiomatic interface on top
4) The cryptography community is more heavily invested into C, and so there are more mature libraries and resources in their ecosystem

A first step would be developing bindings to a suitably-licensed open-source C or C++ library containing modern and up-to-date cryptography algorithms, including post-quantum cryptography schemes.

# Candidate Libraries

There are [several C/C++ library candidates](https://en.wikipedia.org/wiki/Comparison_of_cryptography_libraries):

- Botan
- libgcrypt
- Crypto++
- libsodium / NaCl
- NSS

There are several factors for consideration:

- Licensing:
    - Does the library have an appropriate license, such as an MIT or BSD license?
    - It must be free to use, but not so free as to impose a burden upon on its users.
    - Commercial users are likely to be unwilling to depend on copyleft code, and we want to provide value to them too.
- Community:
    - Does the library have an active development community?
    - Does it have a large userbase?
- Bindings:
    - Does the library have a Haskell FFI-capable C API?
    - How difficult will it be to bind to this library?
- Auditing:
    - Has the library been audited for security concerns?
- Coverage:
    - What algorithms / primitives does this library implement?

Of these, Botan was selected for several reasons:

- It has a BSD license
- It has a Haskell FFI-capable C API
- It offers a broad variety of functionality and algorithms, including post-quantum cryptography
- It is developed and maintained by an active community
- It has been [audited](https://botan.randombit.net/releases/audit_1.11.18.pdf) in the past
- There are unmaintained Haskell bindings targeting Botan 2 / GHC 8 that can be examined / revived
- Significant implementation work has already been performed in order to validate the feasibility of Botan

The others were not selected for several reasons:

- libgcrypt is C, and so the Haskell FFI can bind directly to it, but it has a GNU LGPL license, and provides less functionality than Botan
- Crypto++ is C++, and has a Boost license (similar to BSD), but it does not export a C API, and provides less functionality than Botan
- libsodium / NaCL does not provide the necessary functionality, and there are several Haskell libsodium bindings already (see Appendix C - NaCl / libsodium)
- NSS has a copyleft license

Bindings to Botan would solve the significant problem of dealing with common cryptography by providing much of the necessary 'cryptographic kitchen sink' via bindings to a suitably performant, suitably licensed, open-source library. The Haskell community would not be required to maintain the Botan cryptography library, only bindings to it, and this can be accomplished readily with some effort which has already begun. Furthermore, the Botan library supports several post-quantum cryptography algorithms, which is highly desirable.

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

Further testing of `Bcrypt` and `SHA-3` with benchmarking using `tasty-bench` shows a considerable increase in speed, although `Botan` memory usage cannot be accurately measured due to the FFI.

```
$ cabal bench --benchmark-options '+RTS -T' botan-low-bench
Benchmark botan-low-bench: RUNNING...
All
  Bcrypt work factor
    Crypton
      twelve:    OK (1.65s)
        550  ms ±  27 ms, 109 MB allocated,  30 KB copied,  34 MB peak memory
      fourteen:  OK (6.57s)
        2.198 s ±  29 ms, 492 MB allocated,  21 KB copied,  34 MB peak memory
      sixteen:   OK (26.38s)
        8.797 s ±  34 ms, 2.0 GB allocated,  32 KB copied,  34 MB peak memory
    Botan
      twelve:    OK (1.39s)
        232  ms ±  14 ms,   0 B  allocated,   0 B  copied,  34 MB peak memory
      fourteen:  OK (1.85s)
        925  ms ±  30 ms,   0 B  allocated,   0 B  copied,  34 MB peak memory
      sixteen:   OK (11.10s)
        3.700 s ±  40 ms,   0 B  allocated,   0 B  copied,  34 MB peak memory
  Hash
    Crypton
      password:  OK (1.17s)
        1.10 μs ± 106 ns, 3.0 KB allocated,   0 B  copied,  45 MB peak memory
      plaintext: OK (1.75s)
        1.66 μs ± 109 ns, 3.0 KB allocated,   0 B  copied,  45 MB peak memory
      longtext:  OK (2.31s)
        9.04 ms ± 485 μs,   0 B  allocated,   0 B  copied,  45 MB peak memory
    Botan
      password:  OK (2.30s)
        1.09 μs ±  61 ns, 1.1 KB allocated,  93 B  copied,  48 MB peak memory
      plaintext: OK (1.58s)
        1.49 μs ± 119 ns, 1.2 KB allocated,  93 B  copied,  49 MB peak memory
      longtext:  OK (1.42s)
        5.55 ms ± 423 μs, 1.0 MB allocated, 117 B  copied,  97 MB peak memory

All 12 tests passed (60.66s)
Benchmark botan-low-bench: FINISH
```

Results indicate that bindings to Botan may be significantly more performant than `crypton/ite`, if other modules / functions are similarly performant.

Note that the memory use increase in Hash.Botan.longtext is due to the rough state of initial bindings, as the bindings are not yet leak-free. Despite this, it is still approximately twice as fast.

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
        - `cryptohash-md5`, `-sha1`, `-sha256`, `-sha512` are reduced-footprint forks
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

These prior efforts can be considered to be somewhat partially successful in presenting Haskell developers with access to cryptographic primitives, but present a significant development challenge and a high refactoring cost if dependencies become deprecated, which happens often. Of these libraries, a considerable amount are deprecated or have been dead for many years, with `cryptonite`, `crypton`, and `saltine` being clear front-runners in terms of current usage and transitive dependencies (and `sel` being under active development). 

Of these four, one is a fork of another (`crypton` and `cryptonite`), and the other two are bindings to `libsodium` which does not sufficiently capture our need for low-level cryptographic primitives in general. `crypton/ite` exposes a few highly- opinionated / specific classes for hashing, block and stream ciphers, but otherwise is focused on concrete implementations and functions. Both `crypton` and `cryptonite` rely on bindings to C implementations that have not undergone the same vetting as commonly-used C libraries; though the code is itself not suspect, the lack of provenance makes it ill-suited for recommendation in trusted environments, and maintenance of the C implementation poses burden of high technical skill. `libsodium` bindings are more trustworthy, but the limited options severely hamper interop with other systems unless they too are using `libsodium`, making them ill-suited for recommendation for some use cases.

Following that, there is also `Z-Botan`, which has been apparently unmaintained for several years. Reviving it was considered, but would require a comprehensive and near-total rewrite in order to divest itself of `Z-Haskell` as a dependency. New bindings could instead draw significantly from `Z-Botan`, while presenting a much smaller maintenance and dependency surface.

Furthermore, none of these libraries contain implementations of post-quantum cryptography schemes; although existing quantum computers are of insufficient capability to break commonly used algorithms such as Ed25519, this will not remain true forever, and it would be best to have quantum-resistant implementations ready for adoption long before such attacked become practical.

Each of these efforts has one or more of these issues that preclude them from being considered a complete success:

- Insufficient abstraction
- Maintenance of handwritten C
- Bindings to an insufficiently general library
- Burdensome dependencies
- Unmaintained project
- Lack of post-quantum algorithms

However, there is also a lot to be learned from the ways these prior efforts *were* successful. This proposal will assess these existing efforts in an attempt to consolidate their learnings and functionality.

## An example of a successful ecosystem

The health of the Haskell server ecosystem may be used as an example of a successful community-driven niche, and we can aspire to bring that same quality and health to the Haskell cryptography ecosystem. There are many pertinent parallels in terms of managing performance, complexity, and safety that we can learn from:

- Low-level packages like `wai` supporting multiple backends
- Backends range from simple like `scotty` to complex like `servant`
- Frontends like `blaze-html` and `lucid`
- Libraries can take advantage of type system to provide good abstractions
- Flexible enough abstractions to explore the problem space with multiple solutions
- Appropriate licensing that is free and unrestrictive, such as an MIT or BSD license

`servant` (and the `wai` ecosystem in general) is a success story and an example of Haskell making something safer and easier, something that we can aspire to do with cryptography.

## Technical Content

This proposal is for the contruction of a set of bindings to the `botan` cryptography library, in the form of the following libraries:

### botan-bindings

The `botan-bindings` library will contain raw bindings and is an almost direct, 1-1 translation of the C API into Haskell FFI calls. As such, it exposes and operates over C FFI types, requires buffer and pointer and pointer management, and returns error codes. This library only exposes FFI calls and constants, and is suitable for building your own abstraction over Botan.

### botan-low

The `botan-low` library will contain low-level bindings which wrap the FFI calls into IO actions. This library will handle the translation between buffers and ByteStrings, and throw exceptions in the case of errors, but will otherwise be a fairly faithful translation of the Botan interface. This library is suitable for use, but is not recommended for day-to-day programming due to statefulness and use of strings for algorithm selection.

### botan

The `botan` library will contain high-level bindings which hide the stateful IO and expose appropriately idiomatic or pure interfaces. It will provide data types for algorithms, enumerations for constants, convenience methods for key and nonce generation, incremental processing of lazy bytestrings, and otherwise bring significant safety and ergonomics over `botan-low` and `botan-bindings`.

## Related Technical Content

More advanced concepts and data structures may be delivered in a future proposal focused on generalization, but some development may occur earlier in order to provide a testable downstream use case for the `botan` libraries.

## Goals and Non-goals

The following are the immediate goals of this proposal:

- To publish the `botan`, `botan-low`, and `botan-bindings` libraries
- To build proper documentation and tutorials covering the subject material
- To build unit tests and provide test vectors for provided algorithms

The following are goals, but may not be complete by this proposal:

- To provide a suitable alternative interface / backend (as an option) for `crypton`, `tls`, `x509`
- To consolidate Haskell cryptography into a consistent, unified, user-friendly system
- To enable the safe use and implementation of advanced cryptographic data structures

The following are not goals of this proposal:

- Implementing a hierarchy of high-level cryptographic abstractions
- Matching interfaces 1:1 with existing libraries / interfaces
- Implementing every algorithm permutation possible

## Timeline

This timeline will focus on the immediate goal of publishing the following libraries:

- botan-bindings
- botan-low
- botan

This proposal has a defined scope and set of deliverables, and will require substantial time to complete.

### Expected project duration

We estimate that it will take 3 months for this project, to achieve our goal of developing and publishing these libraries. This estimate is based on the significant amount of work already completed, the length of time in which it was accomplished, and the remaining amount of work.

Several work items (strictness, memory safety, C++ shims for missing FFI functionality) are difficult to estimate, and so while time has been allocated for them in the estimate, their delivery is considered optional within the proposed timeline in order to ensure completion of the immediate goals. If necessary, the project may be extended at the discretion of the Haskell Foundation in order to ensure their completion.

### Intermediate deliverables

The following are intermediate deliverables intended to keep the community apprised of any progress that is made, and any challenges that arise, and to seek community feedback:

- Frequent (2-3x weekly) activity update on devlog
- Weekly major update that includes github push
- Monthly report that summarizes what specific items have been worked on, what items will be worked on, and what challenges have arisen, if any.

### Deliverables

The following are the intended deliverables of this proposal:

- `botan-bindings`
- `botan-low`
- `botan`

Each library should be beta- or / production-ready, and submitted to hackage as a candidate or accepted package. 

Each library should have:

- Installation instructions for the Botan library
- Documentation
- Tutorials and examples
- Tracked issues
- Unit tests
- CI tests for Linux, MacOS, and Windows

The official repo will be owned by the [Haskell Foundation](https://haskell.foundation/), and maintained by the [Haskell Cryptography Group](https://github.com/haskell-cryptography), with myself as the current maintainer. Another member of the Haskell Cryptography Group, or a member of the Haskell Foundation, will be selected to hold backup keys and permissions to any website, server, and official repositories as needed, in order to give project members access and to reduce any conflicts about ownership or maintenance.

## Budget

This proposal presents a minimum budget of $7000 USD per month, for one full-time engineer, for a minimum duration of 3 months, for an estimated total cost of $21,000 USD. This budget is based on cost-of-living and industry experience, and will cover housing, food, bills, taxes, and other life necessities for one engineer, as well as any project necessities such website and server hosting. This budget is roughly equivalent to $40 / hr or $84k / yr at full-time of 40 hrs / wk. Industry rates for an engineer of the necessary skill are on average considerably higher, and so we consider this budget to be reasonable. The exact legal contract / arrangement is left to the Haskell Foundation, and this timeline and budget may be extended at their discretion under the same terms.

## Stakeholders

### Positive stakeholders

Several parties stand to gain from the implementation of this proposal.

- Leon Dillinger (Myself)

As the intended engineer and member of the Haskell Cryptography Group carrying out the work, I have already invested significant effort in this project in order to establish its viability, and will benefit from this proposal in the form of being the recipient of the funding.

- [Haskell Foundation](https://haskell.foundation/)

As the intended source of funding, the Haskell Foundation would be invested in seeing that its funding is spent effectively.

- [Haskell Cryptography Group ](https://github.com/haskell-cryptography)

As the intended owner of the project, this project will be new maintenance burden on the Haskell Cryptography Group.

- Users of existing cryptography libraries such as `crypton`, `cryptonite`, `libsodium / saltine`.

As the intended target audience of this project, this proposal aims to directly benefit developers by providing high-quality cryptography libraries with an appropriate level of abstraction, but will also place the burden of migration to these new libraries on developers.

- Downstream users of Haskell software

As the eventual user of the developed software, their safety and security will rest on the proper execution of this proposal.

- [Botan / Randombit](https://botan.randombit.net/)

As the developers of the `Botan` library, this project depends on their work, and some of this work may result in contributions back upstream to the `Botan` source code (such as any improvements / C++ shims).

### Negative stakeholders

- Supply-chain attacker

As an attacker, every open-source project is a target for silently inserting compromised code into the codebase through PRs with the goal of penetrating downstream codebases. By relying on bindings to a public, well-scrutinized library, this project reduces the available attack surface comparative to implementing it ourselves.

## Risks

There are a number of risks.

- There are many algorithm-specific nuances that are absent from library documentation - this makes securing each algorithm an individual albeit minor effort that requires often consulting external documentation
- Botan FFI is incompletely documented and has gaps (stream ciphers, X509 stores, a few broken functions) which need to be fixed with C++ shims
- Strictness, memory safety & scrubbing, C++ shims will take an unknown amount of time
- A thorough / safe implementation will require consulting the C++ source of the FFI
- Polynomial combinations of algorithms makes exhaustive testing complicated.

These risks are amortized in that while they are tedious and potentially time-consuming, they relate to extended goals of high value, and while there is still substantial value that can be delivered without them, the completion of risk-related items may be ensured by extending this proposal's timeline and funding if necessary.

- Single point of failure

This is a proposal for one full-time engineer; this constitutes a single point of failure. To combat this, the official github repo will be owned by the Haskell Cryptography Group, and another member of the Haskell Cryptography Group, or a member of the Haskell Foundation, will be selected to hold backup keys and permissions to any servers and official repositories as needed. In the case of an emergency for which I am unreachable, this will allow for others to take over as necessary.

## Success

This proposal will be considered a success if it results in the submission of a candidate or accepted package to hackage, with production-ready bindings, documentation, tutorials, and unit tests as appropriate for the following botan-related libraries:

- botan-bindings
- botan-low
- botan

The following Botan library features are goals, but are considered optional to the success of this proposal, due to requirement of C++ / FFI shims or additional research which will take an unknown amount of work.

- Extended X509 support
- Stream ciphers
- Test vectors for algorithms, especially [CAVP / FIPS / NIST-approved](https://csrc.nist.gov/projects/cryptographic-algorithm-validation-program)

Effort will be made to include them if possible within the proposed timeline, and the project may be extended at the discretion of the Haskell Foundation in order to ensure their completion.


# Appendix A - Commercial Incentive Misalignment

There is no open, community-driven, community-owned cryptography solution, and it is unlikely that one will be developed without community investment. Commercial development is unlikely to provide us with a solution, due to the issue of ownership of the resulting ecosystem, and the resulting incentive misalignment:

- Cryptography and security are preventative measures and a cost center for most companies
- Companies for which cryptography / security are a product have a strong incentive to marshal users into a walled garden in the name of both security and profit
- Developing an ecosystem is a significant up-front investment, and so investors may expect ownership of the ecosystem in order to recoup their cost
- There is an incentive to wait for someone else to develop an open ecosystem, instead of expending resources yourself to do it

As such, commercial or for-profit approaches are incentivized to either invest in a private ecosystem, or invest in an established open ecosystem, but are perversely incentivized against building a new open ecosystem.

# Appendix B - Lack of ByteString `Bits` instance

Essentially, `Bits` (and the fixed-width `FiniteBits`) is a class that combines two concepts:

    `Boolean / Heyting` algebra, for which there is no concept of any 'internal bits' or structure and you are just operating on some object as a whole
    Things that are contructed from / encoded into a set of 'internal bits' using some encoding, for which individual bits are indexable / addressible, and thus individually boolean-operable. There is also a link here to indexing / representable, but that is getting way out of scope.

Notably, the documentation itself further states that "The `Bits` class defines bitwise operations over integral types.". Technically this means that only things that are integer numbers should be `Bits` and `FiniteBits`, but Godel numbering rears its ugly head, and that just adds the question of whether `Bits`'s boolean operations are intended to apply to an object itself or its encoding, and as a result `Bits` and `FiniteBits` are restricted to objects for which the boolean operations are isomorphic to boolean operations over its encoding.

Given this understanding, `ByteString` does not have a canonical `Bits` instance because it has no canonical integer representation - a given bytestring could represent many such integers, whether it be big endian or little endian, it could have a sign bit, or be two's complement. Its bits might not be contiguous or even ordered. It could easily satisfy a hypothetical `Boolean` instance, but the issue of indexing individual bits and integral type requirement interferes with a full instance of `Bits`.

Solving this issue is out of scope of this proposal, and the tangible impact is low as it is easily solved with a simple newtype wrapper.

# Appendix C - NaCl / libsodium is not a cryptographic sink

NaCl / libsodium is not a general purpose cryptography library; it is a highly opinionated and minimalist cryptography suite designed for safe and ergonomic use at the expense of control. 

- It makes all of the decisions for you
- It only implements a handful of common primitives
- It only implements a single algorithm variant for each primitive
- Interop with non- NaCl systems is difficult to impossible if NaCl doesn't support the algorithm or primitive.

The idea behind NaCl is that the developer doesn't need to worry about the algorithms - they've chosen great defaults for most crypto use cases, and provided a high-level interface to help you implement it correctly.

In fact, NaCl only provides:

- SHA-2 (or sometimes Blake2b) hashing
- HMAC-SHA-512/256 message authentication
- AES-GCM cipher encryption
- Salsa20 stream encryption
- Ed25519 / X25519 public key signatures / agreement
- Poly1305-Wegman-Carter one-time authentication
- Salsa20-Poly1305 AEAD
- X25519-Salsa20-Poly1305 public key encryption

Granted, NaCl does a good job of choosing a commonly used set of algorithms, but it is unsuitable as a general purpose cryptography library because it does not support anything except for these few select algorithms and primitives. If you need SHA3 hashing, or CBC mode for decryption, or RSA public / private keys, or arbitrary named elliptic curves, then NaCl can't really help.
