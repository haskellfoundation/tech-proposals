# Tooling for maintaining a GHC API

## Abstract

This proposal is to build tools to define and maintain a GHC API. Some
automation is necessary to monitor the needs of projects using GHC as a library,
and to make GHC developers aware when their changes affect these projects. With
this knowledge, the involved parts of GHC can be better defined and documented.

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

To the best of my knowledge, no project has tried before to improve the
documentation of the GHC implementation, though there have been efforts
to refactor the implementation itself to make it easier to maintain and
reuse. This author thinks that accessible documentation amplifies the
the benefits of any code changes.

## Technical Content

In order to identify which parts of the GHC implementation are used by other
packages, GHC developers should have an index of the names from the `ghc`
library that are used in a selected set of packages, called henceforth the
indexing set. This index can be used to define the curated subset of the GHC
implementation that will be exposed to tooling authors under some designated
module hierarchy. In this document, we will refer to this curated subset as
the GHC API.

The size of the initial GHC API can be tuned by growing the indexing set
progressively, starting with the projects that are considered most relevant to
the community, and relaxing it as more resources become available. 

The GHC API will indicate the features that need to be documented for external
use, and it will allow to flag the changes to GHC that affect it. GHC
developers would then have the opportunity to decide whether to make the changes
backward compatible or document the API changes for their users.

The following phases emerge from these considerations.

### Indexing Phase

This phase should produce a tool that can build the index of names from the
`ghc` library (and perhaps `ghc-lib-parser`) which are used in other packages.
It should be possible to configure which packages or units to include in the
indexed set.

In addition, a library should be provided that allows us to query the index.
The following queries should be possible to answer:

* The list of names from the `ghc` library that are used by other packages. Note
  that the index should provide enough information to allow importing the name
  (e.g. whether it is a pattern synonym; or if it is the name of a data
  constructor or a field, it should be accompanied by the name of the data type).
* The modules from other units that are using a given name
* The most commonly used names from the `ghc` library

This phase could be based on the compiler plugin and the analysis script in
[this repo][indexing repo], or it could be based on other indexing solutions.

[indexing repo]: https://github.com/tweag/ghc-api-usage-stats

### API generation phase

This phase should produce a tool that generates or regenerates modules in the
GHC API from the index. If a module does not exist yet, it should be
created from some configurable template. If the module already exists, the tool
should edit the export list and import declarations while trying to preserve
the contents in the rest of the module file. Other generators sometimes
implement special comments to designate lines that should not be modified by
the generator.

The tool should probably allow us to specify rules to indicate a few things:
* which names should be exposed in which modules
* which modules should be used to bring some names into scope
* to exclude some names from being exported despite appearing in the index.
  A file with a list of excluded names should be generated if using globbing
  or similar in the rules, so new excluded names are made visible when
  regenerating the API.

### Documentation review phase

In this phase, the code documentation of GHC needs to reviewed, and procedures
need to be documented to keep it up to date.

For the review part, a team of a newcomer and an experienced contributor should
systematically review the documentation of each module in the exposed subset.
Perhaps starting by the most commonly used definitions as indicated by the
index queries.

For the update procedures, it should be documented what the GHC API is, how to
update it, and when to update it. Newcomers should be invited to request
documentation improvements. Documentation improvements should be made fast and
easy to merge. Maybe most continuous integration (CI) jobs could be skipped for
documentation updates except for some linting.

Additionally, the immutability of the GHC API needs to be checked in GHC's CI.
Tooling to do this already exists for other parts of GHC, so this task should
be mostly about configuration work.

### Risks and Limitations

The project could fail if the size of the GHC API exceeds the availability of
the community to document it all. In such a case, the project should still be
helpful to identify the areas of the GHC implementation that still need
additional effort to better support their exposure.

Not all changes to the GHC API will be possible to detect automatically, in
particular, changes in behavior that don't modify types or the type signatures
of functions. Alternatively, the proposal could be extended to try to detect
changes to documentation of definitions that appear in the GHC API. But still
there will be shades of behavior that will likely not be caught in documentation
either.

## Timeline

There are no specific deadlines to this project.

## Budget

The cost of this project involves the engineering time needed to perform
the identified phases. The following is a rough guess from the proposer,
but it needs to be refined with whoever is appointed to execute the project.

```
Indexing phase             --- 40 hours
API generation phase       --- 80 hours
Documentation review phase --- depends on the chosen indexing set
```

The actual money required also needs to be negotiated with the appointed
developers.

## Stakeholders

* GHC developers
* Tooling authors from the [outreach phase]
* Users of Haskell tools who need them to stay up to date

## Success

The project will be successful if the users of the `ghc` library have an
accurate understanding of what it will take to upgrade their projects to use a
newer version of the compiler by reading changelogs and the API documentation,
thus eliminating the trial and error costs.

The project will be successful too if accidental breakage of downstream tooling
is avoided thanks to the definition of a GHC API whose modifications are
flagged by GHC's CI.
