## Introduction

This proposal seeks technical coordination from the Haskell Foundation
for improving the interop story around Haskell tooling error messages.
While much of the work may be doable by volunteers, the HF would play
a role in harnessing and corralling the volunteers, as well as coordinating
common APIs between tools that are both easy to implement and easy to use.
The HF may also be instrumental in managing an error code namespace, shared
among all tooling central to Haskell.

## Background

Currently, there is no discipline around error messages. This lack of
structure manifests itself in a number of ways:

 - Some tools parse the error messages that other tools
   produce. This is fragile, wasteful, and hard to keep up-to-date. For
   example, the HLS looks to see if a GHC extension name appears in an error
   message, in order to allow the user to automatically enable it via a pragma.
   But since `KindSignatures` is a substring of `StandaloneKindSignatures`, any
   message mentioning the latter causes HLS to suggest both enabling `KindSignatures`
   and `StandaloneKindSignatures` -- even though only `StandaloneKindSignatures`
   would actually work. While there is a workaround here, we can see that
   better communication between GHC and HLS would avoid this class of problem.

 - Many Haskell error messages refer to advanced concepts. This is unavoidable,
   as Haskell has advanced features. However, telling a user that their rigid
   type variable does not unify with a type because there is a kind mismatch
   is utterly bewildering to Haskell learners. Applying structure to error messages
   would allow for the creation of an error-message index that could explain
   what the messages mean -- and how to fix the errors.

 - Given that tools are increasingly working with one another and invoking
   one another, it can be hard to know who exactly is producing an error
   message. In one recent example that happened to me, I was trying to get
   GHC to work with a GHC plugin, and I got a baffling error. It took me
   more than an hour, if I recall, to discover that the problem was with
   Haddock (I forget the details) and that I just needed to `--disable-documentation`.

There is already work in this area, within GHC. For the past few years, GHC
has slowly been converting its error messages to be encoded in data constructors,
not just as (fancy) strings. [This wiki page](https://gitlab.haskell.org/ghc/ghc/-/wikis/Errors-as-(structured)-values)
and [this blog post](https://well-typed.com/blog/2021/08/the-new-ghc-diagnostic-infrastructure/) describe
roughly the state of play. However, this work currently lacks a very important
ingredient: clients. That is, if GHC is exporting new, fancy datatypes encoding
its error messages, are these datatypes of use to, say, HLS? We've reached out
to potential clients for feedback, but the best response we've gotten is something
along the lines of "sure, looks good". That's encouraging, but I would want to
a little more coordination to make sure that the interface GHC is building is one
that can be easily consumed. The HF could help here by coordinating this
communication between projects.

Furthermore, if this is successful in increasing interop between (say) GHC
and HLS, then we can expand the idea to other tooling, as well.

The website describing error messages and error-generator identification
are both fresh in this proposal.

## Motivation

- It is better for tools to collaborate by passing structured data than
by sending strings back and forth. Structured error messages will thus
accelerate the development of powerful editor integrations and other
code analysis tools.

- Establishing a website describing error messages will make it a standard
reference in the Haskell community and flatten the learning curve to
new Haskellers.

While the two main goals of this proposal (conversion of all error messages
to use datatypes; assigning error codes / creating a website) could be
considered separately, I think they make sense together in this proposal.
The second goal depends on the first, and it seems likely that many of
the same potential volunteers will be interested in both. That said, it would
be fine for, e.g. the HFTT to accept only one part of this proposal without
the other, or simply not to commit resources until a mid-way review were conducted.

## Goals

1. When compiling a program, HLS queries GHC for error messages and receives
structured errors, not strings. HLS can then use the information in these structures
to offer repairs to the user or other options.

1. All GHC error messages include a code. These codes can be searched for on a website
that explains the error message, with examples of what causes it and how the error
might be fixed.

1. Stretch goal: Building on the success of the HLS/GHC integration around error
messages, other central tooling adopts a similar approach. This would, for example,
enable the possibility that HLS can report more informative configuration errors
to users, or even to repair some of the problems itself.

1. Stretch goal: The HF would establish a global namespace for Haskell-tool error
message codes, where each tool includes a code in each message. This would both
broaden the domain of the website index of error messages and also serve to identify
the producer of error messages.

## What the Haskell Foundation Would Do

This section is meant to be suggestive of the concrete activity that would support
this proposal. It is possible the HFTT or other HF people would have an alternative
approach, which is fine, too.

1. Devote the time of an HF employee (hereby called the Coordinator) to stay on top of
this project. I think it would be reasonable to timebox this work at 5 hours / week from
the Coordinator.

1. A key task of the Coordinator is to source volunteers to help with this initiative.
Accordingly, the Coordinator would be responsible for publicity around this plan, as well
as thinking creatively about ways to attract volunteers. For example, it might be a fun
idea to plan a virtual hackathon with potential volunteers or to reward contributions
with t-shirts. I would expect the Coordinator to think creatively about how to source
the volunteers. Volunteer management is a primary requirement of the Coordinator; it is
assumed that the Coordinator is managing volunteers in parallel with all other tasks here.

1. The Coordinator would start by getting an exact handle on the state of structured
error messages in GHC, by working with current contributors (e.g. Alfredo di Napoli, Sam
Derbyshire, Richard Eisenberg) and looking at the GHC source code. The Coordinator
would then identify an area within GHC that would be an appropriate next step to add
similar structured error messages and source volunteers to contribute to that area.

1. In parallel with the previous item, the Coordinator would work with representatives
from the HLS team to figure out how HLS might take advantage of the structured error messages
GHC already has. Even if HLS is not ready to merge yet, the Coordinator and HLS would
work out a way to build a proof-of-concept based on the structured errors GHC already
has. This would validate the current API and increase the confidence in building on it.

1. Having established that the API is usable, the Coordinator would systematically work
through remaining error messages in GHC, directing volunteers to convert them to the
structured format.

1. As capacity is available, the Coordinator would also organize (or encourage a volunteer
to organize) a website where error messages could be explained. This might be a wiki,
or a git repository, or something exportable to e.g. readthedocs.io. Figuring out a good
format would be the responsibility of the Coordinator, possibly by contacting stakeholders
with a survey or looking at other language communities.

1. The Coordinator would devise a scheme for assigning error code to messages. These might
be terse, inscrutable alphanumeric identifiers, or perhaps they would be human-readable.
The namespace would include the possibility of covering tools beyond just GHC, though
recursive hierarchy seems likely unnecessary. With the help of volunteers, the Coordinator
would add these error codes into the error-message API.

1. The Coordinator would continue to encourage volunteers to document error messages on
the error-message website, learning from early successes and failures.

1. If the project is going well and with community support, the Coordinator could look at
extending this idea to other tools. For example, perhaps Cabal or Stack could start to
deliver similar structured error messages -- with buy-in from those maintainers, of course.

## People

-   **Performers:** The Coordinator, someone who will have time dedicated to this project. This person
    would ideally be an HF employee or part of the portfolio of an in-kind donation of labor.

-   **Reviewers:** The GHC team would review changes to GHC, while the HLS team would review changes to HLS.
    The GHC and HLS teams would work together, coordinated by the Coordinator, to make an API that is useful
    to both. Community volunteers would review the text of the website describing error messages. The Coordinator
    would review the uptake of any website by examining analytics.

-   **Stakeholders:** This would affect anyone who uses GHC, as the error codes would appear there. Key stakeholders
    include the GHC and HLS maintainers, as well as educators, who would have access to Haskell learners who
    would benefit from the results of this work.

## Resources

- The Coordinator would need to devote 5 hours / week.
- There would be a set of volunteers who would do much of the labor. If the volunteer pool runs low, the Coordinator
can do some of the technical work, as well.
- The GHC and HLS teams would have to devote some of their time to help support this initiative.

## Timeline

The timeline is highly dependent on the availability of volunteers to do the work. It thus seems
more sensible to timebox this effort at 5 hours of Coordination / week than to set a deadline for
completion. It would be sensible to review progress after 3 months to decide whether this project
is producing benefits (or is likely to soon).

## Lifecycle:

I don't think this really applies here. There would be a warm-up period at the beginning where the goal
is to source volunteers, but afterwards, it's all about keeping people moving forwards.

## Deliverables

1. A release of GHC where all of its error messages are structured.

1. A release of HLS which consumes the structured error messages of GHC.

1. A website explaining at least 20 different errors produced by GHC. (More is better!)
I think we should set a modest goal of having 100 unique visitors to this website
over the course of a month.

1. A blog post (ideally written by the Coordinator) describing this process, as a way
of creating publicity for the HF.

## Outcomes

- With the structured interface to errors, tools such as HLS will be better equipped to
offer more power to users to manipulate and reason about code.

- The error-message cataloguing website will help Haskell learners (and, likely, some
old hands) understand error messages better.

## Risks

- One risk is that the API being built around error messages is not useful to consumers.
This risk is intended to be mitigated by an early consultation with HLS.

- It is possible that the structured error messages will provide no opportunity for
improvement over the status quo. This is a risk the HFTT should consider. It might also
be worthwhile to reach out to HLS now to see what they think.

- It is possible that no one will find their way to the error-index website, or that
the format chosen for the site will not resonate with users. The Coordinator would ideally
reach out to users to understand their needs better as the website is being designed
in order to mitigate this risk.

- It is possible that the extra structure will provide an obstacle to evolution within
GHC and slow development down there. I do not think this is likely, but it is conceivable.