# A process to incrementally document GHC for tooling authors

## Abstract

This proposal defines a process to define and document GHC for tooling authors.
The process involves tooling authors, GHC developers, and the Haskell Foundation in
defining how GHC is to be used to satisfy the needs of language tooling in
incremental fashion.

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

The core of the proposal is a project template to write a cookbook to use the
GHC implementation for a particular task needed by some tool. This template can then be applied to
more tasks later on. Additionally, follow up projects might integrate the
produced documentation into the GHC development process so it is kept up to
date.

In executing such a project, there needs to be at least the following roles:
* a project developer to define the task with precision, and documentation how
  to accomplish it with the exiting implementation of GHC;
* a GHC mentor, who will support the project developer by providing pointers
  to existing resources and giving feedback on the accuracy and of the
  documentation; and
* a tool maintainer that needs to accomplish the chosen task, to validate and
  provide feedback on the produced documentation.

The following steps are necessary to perform the project.

1. The Haskell Foundation and GHC developers select some task to document.
   Availability of the tool maintainers that will use the documentation needs
   to be checked at this stage, and the roles of the project developer and
   the GHC mentor needs to be assigned.

2. The project developer studies the functions and types that the tool uses
   from the `ghc` library, and engages with the tool maintainer where the
   purpose of using them is unclear.

3. The project developer documents the how the tool is using the GHC API,
   which the GHC mentor should review and then propose alternatives if
   there are any. If there are corrections to how the task should be
   implemented, the tool maintainer needs to agree on the changes.
   Feedback from the maintainers of other tools with similar
   needs could be invited at this time.

4. If there is agreement on how the taks is to be done, the documentation is
   updated, and the GHC mentor and the tool maintainer validate the accuracy
   and the completeness of the documentation.

5. The project developer updates the tool to test the accuracy of the
   documentation.

After the project, GHC developers link the cookbook from some place where it
can be discovered by GHC users.
No commitments are made to maintain the cookbook up to date.

In principle, there are no constraints on which task and client of the `ghc` library
should be chosen first. Working on tools that depend on the GHC parser could inform
the proposal for [splitting a parser library],
and there are perhaps smaller projects like [print-api] where tasks could be identified
as well with modest effort.

[splitting a parser library]: https://github.com/haskellfoundation/tech-proposals/pull/56
[print-api]: https://github.com/Kleidukos/print-api


### Risks and Limitations

The project could become too long if the chosen task ends up being more complex than
anticipated. This could require either chosing a different task or splitting the initial
task some some of the smaller tasks can be documented instead.

The cookbook might become outdated as GHC evolves. To avoid this, follow up projects
could incorporate tests of the cookbook in the GHC development process to keep it up
to date.

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

In the short term, the project is successful if it informs follow up
documentation-related and refactoring projects.

In the long term, the project will be successful if the maintainer of the chosen
Haskell tool has an accurate understanding of what it will take to upgrade their
projects by reading the cookbook, thus eliminating the trial and error costs.

The project will be successful too if accidental breakage of downstream tooling
is avoided thanks to the definition of the cookbooks.
