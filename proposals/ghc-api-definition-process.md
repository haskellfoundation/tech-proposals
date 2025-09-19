# A process to incrementally document a GHC API

## Abstract

This proposal defines a process to define and document a GHC API. The process
involves tooling authors, GHC developers, and the Haskell Foundation in
defining and validating pieces of a GHC API in incremental fashion.

## Background

The Haskell Foundation started the GHC API stability initiative last year. This
is a project that aims to identify and mitigate how the GHC compiler affects the
maintainance costs of Haskell tools which use the GHC compiler as a library.

During an [outreach phase], the most cited concern by tooling authors was the lack
of documentation that would effectively help them use and upgrade the
[ghc library].

[ghc library]: https://hackage.haskell.org/package/ghc
[outreach phase]: https://discourse.haskell.org/t/ghc-api-stability-update-3/11407

Documentation is important to use any library, and the compiler is documented
both in the code and [beyond][ghc commentary]. However, the general sentiment
is that documentation is still lacking. This could be due to documentation
being not easy to navigate and discover, for instance if there are relevant
cross references that are missing. And secondly, it could be due to
documentation being written for an audience with a shared context about GHC
internals, which does not always include the authors of Haskell tooling.

[ghc commentary]: https://gitlab.haskell.org/ghc/ghc-wiki-mirror/-/blob/master/commentary.md

When considering what to document better, GHC developers conjecture that not
all of the GHC implementation is currently used by users of the `ghc` library,
and so it would be necessary to identify which parts of the implementation need
to be documented for external use.

Another conjecture is that documenting various parts of the `ghc` library for
external consumption, will most likely require some amount of refactoring to
separate implementation details from the essentials of the features that the
library provides.

## Problem Statement

The problem this proposal aims to address is identifying the parts of the GHC
implementation that are used in other packages, and improving the documentation
of these parts so it is accessible to an audience not initially acquainted with
the GHC implementation.

If the project succeeds, good documentation will save tooling authors the cost
of discovering what the GHC implementation does by trial and error. In practice,
poor understanding of the GHC implementation translates in a long stream of
bugs to fix in downstream projects until each project finally gets the
understanding right. Additionally, the definition of a GHC API should reduce the
amount of changes necessary to Haskell tools during upgrades of the API.

A solution should make easy for GHC developers to know when they are about to
change parts of the GHC implementation that are used in other packages, and it
should offer to tool authors the documentation they need to make effective use
of the GHC implementation in their projects. This documentation must allow a
newcomer to answer at least which features are offered by the GHC
implementation, how they are used, and what is the meaning of the involved
types and functions.

## Prior Art and Related Efforts

Improving documentation has been considered [before][ghc modularity] as a
desirable target, but it was often encountered that the best documentation
required code changes ([hierarchical modules proposal], [hierarchical modules ticket], [compiler modularity ticket]).

Additionally, defining an API for GHC has been tried
[before][overview of the current GHC API] too. All definitions
to be exposed for external use were put in the [GHC module] of the `ghc`
library, but at the moment tools still depend on definitions not exposed
in this way.

[ghc modularity]: https://hsyl20.fr/files/papers/2022-ghc-modularity.pdf
[hierarchical modules proposal]: https://github.com/ghc-proposals/ghc-proposals/pull/57
[hierarchical modules ticket]: https://gitlab.haskell.org/ghc/ghc/-/issues/13009
[compiler modularity ticket]: https://gitlab.haskell.org/ghc/ghc/-/issues/17957
[overview of the current GHC API]: https://github.com/ghc-proposals/ghc-proposals/pull/57#issuecomment-312111938
[GHC module]: https://gitlab.haskell.org/ghc/ghc/-/commit/71ae8ec9651216330ac49e9eae60d195e65c7506

This proposal aims to combine the documentation effort with the API definition.
Where earlier documentation attempts found difficulties related to the structure
of the code, this proposal aims to reduce the scope to the needs of specific
tools in incremental fashion. Where the earlier API definition didn't achieve
the engagement necessary to keep up with the evolution of the tool ecosystem,
this proposal aims to coordinate the stakeholders to support specific tools as
well.

## Technical Content

The core of the proposal is a project template to define and document a subset
of the GHC implementation needed for one of several tools. This template can then
be applied to more tools later on.

In executing such a project, there needs to be at least the following roles:
* a project developer to write the documentation and analyse and implement
  any necessary changes;
* a GHC mentor to represent the GHC developers, who will support the project
  developer by providing feedback on the work and providing technical insight
  when needed; and
* a tool maintainer to validate and provide feedback on the produced
  documentation and API.

The following steps are necessary to perform the project.

1. The Haskell Foundation and GHC developers select some tool to serve first.
   Availability of the tool maintainers needs to be checked at this stage, and
   the roles of the project developer and the GHC mentor need to be assigned.
   Also the project scope needs to be defined at this time.

2. The project developer studies the functions and types that the tool uses
   from the `ghc` library, and engages with the tool maintainer where their
   purpose of using them is unclear.

3. The project developer makes a proposal for an API that suits the tool use
   case, if any refactorings are necessary. Both the GHC mentor and the
   tool maintainer need to agree on the proposal before proceeding with the
   implementation. Feedback from the maintainers of other tools with similar
   needs could be invited at this time.

4. If there is agreement on an API proposal, the project developer implements
   it and documents or provides links so people unacquainted with the GHC
   implementation can understand it.

5. The GHC mentor and the tool maintainer validate the accuracy and the
   completeness of the documentation.

6. If appropriate, the project developer might update GHC so it uses the new
   API as well.

After the project, GHC developers are responsible for maintaining the new API.
No guarantees of backward compatibility are required, but guidance needs to be
provided to clients of the API to accommodate for changes to it.

In principle, there are no constraints on which client of the `ghc` library
should be chosen first, but the community already shows agreement on
[splitting a parser library] that could benefit from the template structure,
and there are perhaps smaller projects like [print-api] on which to test the template,
which could allow for a smaller initial commitment.

[splitting a parser library]: https://github.com/haskellfoundation/tech-proposals/pull/56
[print-api]: https://github.com/Kleidukos/print-api


### Risks and Limitations

The project could have difficulties if the scope definition in the initial step
ends up invalidated by new insight in later stages. This could require revising
scope and budget midway through.

Tools might evolve after the project, requiring APIs to be modified or redesigned.
Ideally, the design of an API would allow to use it together with unsupported GHC
definitions, so users that miss a feature are not forced to choose between using
the API or only using GHC unsupported definitions.

## Timeline

There are no specific deadlines to this proposal.

## Budget

The cost of this project involves the engineering time needed for each of
the identified roles, and it will need to be negotiated in each case.

## Stakeholders

* GHC developers
* Tooling authors from the [outreach phase]
* Users of Haskell tools who need them to stay up to date

## Success

The project will be successful if the maintainer of the chosen Haskell tool has an
accurate understanding of what it will take to upgrade their projects to use a
newer version of the compiler by reading changelogs and the API documentation,
thus eliminating the trial and error costs.

The project will be successful too if accidental breakage of downstream tooling
is avoided thanks to the definition of a GHC API.
