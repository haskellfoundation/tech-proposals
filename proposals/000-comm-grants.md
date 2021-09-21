# Community Grants

## Introduction

This proposal is for the Haskell Foundation to undertake a direct grant program,
funded by individual donations to the foundation, to distribute funding where it
can make a difference to the Haskell community.  The aim is to make the Haskell
Foundation a more appealing target for individuals wishing to donate for the
good of the entire Haskell community.  This indefinite ongoing program would
accept submissions from the community on a rolling basis, and award up to $2000
grants to individuals as a stipend to support their work on the proposed
project.

The primary risk of the proposal is that funding decisions would generate
discouragement or ill will from rejected submissions.  The risk can be partially
mitigated by the use of a random lottery system to select winners after an
initial selection of submissions that the Foundation believes are worth
accepting.

## Background

Numerous mechanisms currently exist for individuals to financially support parts
of the Haskell community, including projects that have individually set up
fundraising, individuals who have requested donations via Patreon, GitHub
sponsorships, etc., and purchase of services and subscriptions to content.  Each
of these choices supports a specific part of the Haskell community, and none of
them offer a default way to contribute to Haskell as a whole.

The Haskell Foundation now also accepts individual donations, and its
organizational mission is consistent with being precisely such an appealing
target for general-purpose donations to Haskell.  However, there is currently
not a clear picture for how funds make it from donations to the Haskell
Foundation to directly encouraging contributions to the Haskell community.  In
surveys of the community, individuals who have decided not to contribute to the
Haskell Foundation have given the reason that they don't see that their
contributions would support the Haskell open source community.

The HFTP process superficially appears to offer such a path.  However, there are
several barriers to its being used directly by individuals or small projects for
which funding could make a difference.  Note that "individuals and small
projects" describes a vast portion of crucial Haskell libraries and tools.

1. The HFTP process is not encouraging to individuals or small projects, and
   says things like "Typically, this includes many rounds of discussion with HF
   leaders, Haskell library maintainers, and the broader community, several
   iterations on the design of the proposal, and some effort at prototyping the
   proposed change."
2. The benefit of grants can depend on life circumstances, such as one's need
   for child care, willingness to leave an employer, etc.  Individuals may
   prefer not to include this information in a document open for public comment.

Thus, the HFTP process isn't a great fit for direct requests from individuals or
small projects for financial support.  Furthermore, the HFTP process gives an
explicit example of a "Summer of Code" style program as within scope for an HFTP
proposal.  This is such a proposal.
Several other communities have similar grant programs.  Some examples include:

* [Google Summer of Code](https://summerofcode.withgoogle.com/), which awards
  between $1500 and $3300 per student, scaled by cost of living, for a student
  to contribute to a project for a summer.
* [Clojurists Together](https://www.clojuriststogether.org/), which awards
  between $1500 and $9000 per grant for people to contribute to a Clojure
  project for an indefinite period of time.
    * See [notes from a previous discussion of duplicating the program for
      Haskell](https://www.reddit.com/r/haskell/comments/89hnul/haskellers_together/dwrdc0y/)
* [The Unitary Fund](https://unitary.fund/), which awards $4000 per grant for
  research-based contributions to quantum technology.
* The [Tweag Open Source Fellowship](https://boards.greenhouse.io/tweag/jobs/4638654002),
  which states that it is twice the amount (and twice the length) of Google
  Summer of Code.
* The [PSF Grants Program](https://www.python.org/psf/grants/) and [Ruby
  Central](https://www.rubycentral.org/grants) specifically fund events and
  conferences.
* The [Perl Foundation](https://www.perlfoundation.org/running-grants.html)
  has no set amounts, and has given grants from $500 through $20000 in the past.

## Motivation

As discussed above, the primary motivation for this proposal is to make the
Haskell Foundation an appealing target for individuals who wish to donate money
for the good of the entire Haskell community, without expressing any specific
opinion on where that money is best invested.  This requires a mechanism for
community decision-making about how to distribute donations among the many
worthy causes where they could do some good, beyond the day to day operations
of the foundation itself.

There are several secondary motivations, as well.

* We hope to increase open source contributions to Haskell by finding high
  efficiency investments where a contribution can make a big difference because
  someone is in a place where they are blocked from contributing.  This doesn't
  necessarily have to be a top priority for the community.  If a reasonable
  investment can help even a second-tier priority, it is still a good thing.
* We hope to build more community support and awareness of people contributing
  to Haskell.  Bringing the community together in support of a shared goal can
  clear the path of procedural obstacles.

## Goals

The goal of this proposal is to set up a grant program administered by the
Haskell Foundation.

Submissions to the grant program should come from individuals who feel they
could perform additional work on projects of benefit to the Haskell community if
they had extra funding.  Submissions may include:

* Open source tools and libraries.  This should focus on tools that are broadly
  useful for Haskell developers.  Simply being written in Haskell is not enough.
* Documentation and learning materials that are freely available to the
  community.
* Project leadership and support, including user support, project management,
  bug triage and coordination, code review, and technical writing.

Certain submissions are not appropriate, and those submitting them would be
referred to the HFTP process, instead.  This includes anything asking for
financial support above $2000.  (This is less than comparable programs in other
communities, but we feel it's important to limit the risk given the lower
standard of review involved versus the HFTP.)  It also includes any submission
that might be controversial in any way, because the more private nature of these
proposals isn't the right process for anything potentially controversial.
Submissions should also not ask for assistance from the Haskell Foundation
beyond the grant money, so if someone wants to recruit contributors, find
infrastructure, etc. that they don't intend to pay for themselves, this is not
the right process for that.  (However, a submission that explains that someone
has already lined up extra assistance is welcome.)  Grants must be awarded to a
single individual, and not shared among multiple contributors.  More complex
proposals like this are also better suited the the HFTP process.

These individuals would write a short summary of their intended work, limited to
two pages, and answering the following questions:

* What are you intending to do, and how will it benefit the Haskell community?
* Why are you the right person to do this work?  If possible, submissions are
  strongly encouraged to include a statement of support from an existing
  maintainer of or core contributor to the project involved.
* How much funding are you asking for, and how will it make your work possible?

As a prerequisite for receiving a grant, individuals will agree to write two
blog posts and share a link to discourse.haskell.org.  The first is a project
plan, which must be posted before funds are dispersed.  The second is a detailed
report on the results, which should be posted after described work has been
completed, regardless of its level of success.  Individuals who do not follow
through on the second blog post are ineligible for future grants.

A Community Grant Committee, consisting of five members, will be chosen by the
Haskell Foundation executive team.  An email list will be set up, readable only
by the executive team and Community Grant Committee, for receiving and
discussing submissions.  Once per month, the Haskell Foundation executive team
will a communicate to the committee how much funding is available for grants,
based on the program's budget, which is at the discretion of the Foundation.

Each proposal will be considered by members of the committee, using their
informed judgement to determine whether accepting the submission is in the best
interest of the Haskell community.  Among other factors, committee members
should consider:
1. Whether the proposal might be controversial.  It should be rejected it if
   there is any likelihood that it will.
2. How much difference the grant is likely to make on the probability for
   success.  A good proposal will justify the amount asked for, which could be
   used for paying expenses directly related to the project, removing obstacles
   to contribution by, for example, paying for child care, or covering housing
   and basic expenses if the alternative is to spend less time on project work.
3. The overall impact if the effort succeeds.  In determining the impact, the
   committee should keep in mind the long-term benefit of building expertise, if
   the submission is likely to lead to opportunities for continuing work.

Community Grant Committee members will use approval voting to determine which
submissions to accept.   A submission is accepted if it receives majority
support from the committee.  When a submission is not accepted, the committee
may choose, by majority vote again, to provide feedback to the submitter on how
their proposal could be improved.  Otherwise, the results of the vote are not
announced, to avoid embarrassment or hard feelings.

A random lottery system will then be used to award grants to accepted proposals,
continuing until either the approved submissions are exhausted, or a submission
is drawn which cannot be funded with the remaining budget.  The committee will
then recommend to the executive team to award the grants selected in the
lottery.  The executive team will have final authority to award grants.

## People

- The Haskell Foundation executive team and board will determine how many
  projects are chosen by setting a budget as funds are available.
- There will be a Community Grant Committee of five people responsible for
  evaluating whether to accept each submission.  This will be chosen by the
  Haskell Foundation executive team.
- Proposals will be written and submitted by various members of the community
  who feel their contributions would benefit from funding.

## Resources

The project can consume whatever resources are allocated to it, at the
discretion of the Haskell Foundation board or executive team.

## Timeline

- Choosing a Community Grant Committee may require apprroximately a month.
- Once there is a Community Grant Committee, the first round of grants can occur
  as soon as the budget is available.  The committee will choose new grants
  monthly after that.

## Lifecycle:

N/A

## Deliverables

N/A

## Outcomes

* We expect an increase in individual donations to the Haskell Foundation.
* We expect an increase in project work by people receiving grants.
* We anticipate the possibility of some negative sentiment as some grants are
  not awarded.  We seek to minimize this by choosing winners in a random
  lottery, so that failure to receive an award isn't seen as rejection by the
  community.

## Risks

The main risk worth considering is that funding decisions will generate
resentment or discouragement.  To mitigate this, submissions that meet the
initial bar will be accepted in a lottery process, where the selection is at
random.  Projects not selected will be automatically pre-approved for the next
selection, if the author confirms that they are still interested in continuing.

A second risk is that Haskell Foundation funds are not spent on the best
projects.  We mitigate this risk by limiting the amount of the grants to $2000.
Additionally, there's an initial assessment of all submissions to ensure they
meet a high bar where investment of Haskell Foundation funds is likely to
generate a good return.
