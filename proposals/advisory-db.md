# An Advisory Repository for Haskell

## Introduction

Many programming language ecosystems today have a repository for security advisories that affect libraries, and tooling to help developers to discover advisories about libraries that they depend on. For instance, JavaScript's `npm` and Rust's `cargo-audit` will both issue warnings when there are known advisories for the current project's dependencies. Similarly, development hosts such as GitHub have tooling that will notify developers when advisories appear for their code, and will sometimes even automatically adjust dependency bounds to avoid versions for which there are advisories.

This kind of tooling is nice for developers, and it can also be essential for achieving and maintaining certifications such as ISO 27001, which are requirements for working in certain regulated industries. The lack of this tooling can be an obstacle for the use of Haskell in these contexts.

This proposal does not seek to present a complete solution. Instead, it presents a first step towards advisory automation: the establishment of a repository of advisories. GitHub has expressed interest in building support for such a database into Dependabot, their tool for notifying authors and adjusting bounds automatically, so we can gain real value very quickly, and then defer further development to the future. Future work will be explained in order to demonstrate that this proposal's outcome will be a useful foundation on which to build.

The overall strategy of this proposal is to avoid reinventing the wheel. We should, to the extent possible, do what other language communities do. In particular, I plan to copy the Rust approach.

## Background


### Security Advisories

Information security can only be understood with respect to a given _threat model_ that enumerates the capabilities of an adversary. For instance, countermeasures that only prevent an adversary from accessing a system over the Internet are not relevant if the adversary is assumed to have physical access to the system. Thus, security advisories are not simple binaries - an advisory should explain the context of a potential vulnerability, the resources required for an attacker to exploit it, and any known mitigations. Additionally, security advisories need to have sufficient structure that they can be identified by automated tooling.

One well-known database of security advisories is the [Common Vulnerabilities and Exposure](https://en.wikipedia.org/wiki/Common_Vulnerabilities_and_Exposures) database.

Programs written in Haskell are not immune to security problems, notwithstanding Haskell's expressive type system, automatic memory management, and control over side effects. Programs that stick to the safe subset of Haskell may still leak private data, be subject to denial-of-service attacks, or misuse cryptography. Historically, the Haskell community has had an _ad hoc_ approach to security advisories, with each package author disclosing issues as they see fit.


### Freeze Files and Build Files

A _build file_ describes constraints on the direct dependencies needed to build a project. Typically, a build file will Haskell build tools use the following formats for build files:
 * `.cabal` files are the native format used by `cabal-install`, and part of the configuration of a package that uses `stack`.
 * `stack.yaml` files point the `stack` tool at a Stackage resolver, which contains a collection of packages that are tested and maintained in lockstep. When using Stack, the `.cabal` file often omits specific information about version bounds, because a Stackage resolver contains only one version of each package. The `stack.yaml` file can additionally point at additional dependencies, such as from a Git repository.
 * `package.yaml` files are used by `hpack` to generate `.cabal` files, filling out a number of default values by inspecting the code.
 * `cabal.project` files configure the relationship between a collection of packages in a repository, and can do things like specifying alternate sources for dependencies, much like a `stack.yaml` file.

It's important that we can reasonably support the following workflows:
 1. Users write `package.yaml` files and a `stack.yaml` file, and generate the `.cabal` file.
 2. Users write `.cabal` files and a `stack.yaml`
 3. Users write `.cabal` files and a `cabal.project` file
 
A _freeze file_ describes specific choices for each version of all transitive dependencies of a project. While a build file gives a set of constraints, a freeze file provides a solution to the conjunction of the build file's constraints and the constraints imposed by the rest of the package ecosystem. Unlike build files, freeze files are platform- and compiler-version-specific, because packages deeper down the dependency chain are often low-level wrappers around operating-system-specific functionality.

 
### Dependabot

Dependabot is a tool from GitHub that monitors repositories for security advisories in their dependencies. When an advisory is found, Dependabot can notify the repository owner, and even optionally attempt to automatically create a pull request that excludes the vulnerable version from the bounds.

Depending on which language it is used with, [it may support either lock files or build files](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph#supported-package-ecosystems). In either case, it checks only the dependencies that are explicitly mentioned in the repository. This means that using freeze files will lead to an inspection of transitive dependencies, while build files will only lead to an inspection of direct dependencies. On the other hand, because freeze files represent just one element of a set of possible solutions, they may miss potential advisories in direct dependencies.

GitHub has informed us that they need the following to add support for Dependabot for Haskell:
 1. A source of advisories, ideally in the form of a Git repository with one file per advisory
 2. A definition of the advisory file format
 3. A definition of the [versioning scheme used in Haskell](https://pvp.haskell.org/)
 4. A description of how to retrieve versioning information from build and/or freeze files


## Motivation

This project lays the groundwork for a variety of tools to be built in the future that will both relieve open-source developers and maintainers from tedious work and enable certain large organizations to use Haskell.

Writing Haskell code should be fun. Auditing dependencies for ad-hoc security advisories is not very fun, and it is work that should be automated to the extent possible. At the same time, responsible open-source maintainers do not presently have a clear place to communicate issues that they discover with their code, which can be a source of stress.

From the perspective of larger organizations, this project lays the foundation for tooling that can allow Haskell to be used in contexts where it is rejected today. Organizations that require certifications such as ISO 27001 must document their IT security practices, and if they can't provide a process for auditing their code, then certification becomes more difficult. I have experienced this in a previous workplace, and I have heard from Haskell Foundation sponsors that this would be useful.


## Goals

The goals of this project are to do the following:
 1. Adapt the [RustSec advisory DB](https://github.com/RustSec/advisory-db) data format and working processes to Haskell, and establish a canonical advisory database for Hackage.
 2. Liaise with GitHub to ensure that the database fulfills the needs of Dependabot
 3. Establish a sustainable and trustworthy governance structure around the advisory database
 4. Gather as many historical advisories as possible

The project is thus completed when Dependabot is capable of delivering alerts (not necessarily pull requests) to Haskell projects, and there is a functioning organization managing the database.

Goals left for future work are:
 1. Develop a reverse-import tool to import GitHub's own security advisories into the canonical database using their GraphQL API
 2. Augment build tools such as `cabal` and `stack` with the ability to audit dependencies and build plans for known advisories
 3. Augment Hackage and Stackage with information about advisories

## People

The people involved in executing this proposal, if accepted, are:
 * David Thrane Christiansen
 * Gershom Bazerman
 * Davean Scies

## Concrete Details
### File Format

The file format for advisories is based on that of RustSec, with changes made only for compatibility with Haskell tooling and concepts. An advisory consists of a Markdown file, the first element of which must be a fenced code block written in the `toml` language. This block contains the advisory's structured metadata.

The TOML frontmatter must contain a table called `advisory` and a table called `versions`, and it may contain a table called `affected`. The `advisory` table contains the following fields, all of which are mandatory unless otherwise indicated:
 * `id`, a string, which is a unique identifier. This string should have the form `HSEC-YYYY-NNNN`, where `YYYY` is the year and `NNNN` is a sequential numbering beginning at `0001`.
 * `package`, a string, the name of the affected Hackage package
 * `date`, a TOML local date, which is the disclosure date.
 * `url`, an optional string, which is a link to a resource such as release notes or a blog post that describes the issue in detail
 * `categories`, an optional array of strings, each of which one of `...` TODO
 * `cvss`, an optional string, which is a CVSS 3.1 vector
 * `keywords`, an optional array of strings, which may be any string that the submitter finds relevant. By convention, they are written in lowercase.
 * `aliases`, an optional array of strings, each of which is another identifier such as a CVE
 * `related`, an optional array of strings, each of which is an identifier for a related advisory (such as for a wrapped C library)
The `affected` table, if present, contains the following fields, all of which are optional:
 * `arch`, an array of strings, each of which is the value of `System.Info.arch` on the affected systems. The advisory only applies to the specified architectures. If this key is absent, then the advisory applies to all architectures.
 * `os`, an array of strings, each of which is the value of `System.Info.os` on the affected systems. The advisory only applies to the specified operating systems. If this key is absent, then the advisory applies to all operating systems.
 * `declarations`, a table that maps fully-qualified names from the package to Cabal v2.4 version ranges. These ranges must all be contained in the affected versions (specified later), and they specify that the given name is the source of the advisory in that sub-range. This allows one advisory to mention a function or datatype that is renamed at some point during development.
The `versions` table contains a single mandatory key, `affected`, whose value is a string that contains a Cabal v2.4 version range.

Cabal v2.4 version ranges are specified using the following grammar: TODO

Tools that detect vulnerabilities will need to check whether advisory version ranges overlap with dependency version constraints. The algorithm for this is TODO.

### Recommendations Regarding Build/Freeze Files

TODO: Which formats do we recommend they look in to start with? `.cabal`, `cabal.freeze`? Stack users' constraints mostly come from their snapshot - how can we make this work for people with no constraints in `.cabal` and a `stack.yaml` file?

### Governance and Administration

The Haskell Foundation will be responsible for appointing the administrators of the database. Administrators are expected to be trusted community members who are willing to provide timely feedback in the repository. The Haskell Foundation should check from time to time that feedback is timely, and can serve as a final arbiter of disputes.

## Resources

This proposal requires the following:
 * A sufficiently robust implementation of TOML 1.0 in Haskell. This can be done either by coordinating with volunteers, in which case the primary cost is time, or by hiring a contractor to update an existing library if maintainers are interested. This also has a positive effect on the rest of the Haskell ecosystem, as TOML is widely used.
 * A parser that validates that advisories conform to the format. The Markdown processing side is solved, and developing the TOML side is only a couple hours of work.
 * Accessible format documentation for non-Haskell libraries that will consume it. This will take a couple of hours by the proposal authors.
 * The HF will need to spend time contacting maintainers of various packages on Hackage and asking them to add advisories for old versions of libraries or programs that are known to have issues.
 * Ongoing administration of the database by a group constituted by the HF executive team. Administering this group will likely take a few hours per month from the HF.

## Deliverables

The deliverables are:
 1. A Git repository that is prepared to accept advisories
 2. An initial committee who will triage incoming advisories
 3. Documentation and example implementation for the file format
 4. Recommendations regarding build and freeze files to be examined for versions

## Risks

There are primarily reputational risks associated with this project. Low-quality or false advisories risk damaging the reputation of package authors or maintainers, as well as that of the project and/or organization.

There are very few technical risks to the project itself, as the technology involved is simple and well-understood.
