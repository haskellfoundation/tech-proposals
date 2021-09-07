An HFTP (*Haskell Foundation Technical Proposal*) is a process for submitting a proposal for a technical initiative to that will be executed under the guidance of the Haskell Foundation (HF). This document describes how the community may propose, discuss, and implement technical initiatives to be carried out through the Haskell Foundation that will affect the Haskell ecosystem. Each HFTP goes through an iterative process in which it is discussed by the HF Technical Track (HFTT), the broader community, and any and all interested stakeholders. If a consensus is reached, the initiative is accepted and may then be executed by the Foundation (resources permitting). Only upon reaching a consensus are initiatives accepted and executed. Upon accepted, an HFTP and its associated discussion are merged into this repository, and serve as a historical document that details a specification and rationale for work detailed within the process.

The aim of the HFTP process is to apply the openness and collaboration that have shaped Haskell's documentation and implementation to a process of evolving the HF and the broader Haskell ecosystem. This document captures our guidelines, commitments and expectations regarding this process.

## Why Write a Proposal?

HFTPs are key to making the HF and its initiatives better for the good of everyone. If you decide to invest the time and effort of putting a proposal forward and see it through, your time and efforts will shape and improve the Haskell ecosystem, which means that your proposal may impact the life of a myriad of developers all over the world, including those on your own team. For many, this aspect alone can be quite worthwhile, however, the proposal process itself offers several key benefits to the broader community: 

1. **Concretizing ideas:** as ideas for proposals are fleshed out, so too are the resource requirements, timelines, stakeholders, and available approaches. In this sense, a proposal allows us to fully concretize a proposal idea in terms of the work needed to complete the task. 
2. **Implementation guidance:**  the myriad of subject matter experts and stakeholders affected by a proposal can comment in a centralized place to form a consensus and guide the implementation strategy for a particular task. 
3. **Resource requirements:** ideas are often discussed in the abstract, without the consideration for the cost of work required for their implementation. This process allows for discussion that reifies the resource requirements for proposals, and allows the community to more accurately weigh the cost and benefits of an implementation. 

It's important to note that seeing a proposal through to its conclusion is an involved task. On the one hand, it takes time to convince people that your suggestions are a worthwhile change for hundreds of thousands of developers to accept. Particularly given the sheer volume of developers that could be affected by a proposal, its acceptance is conservative and carefully thought through. Typically, this includes many rounds of discussion with HF leaders, Haskell library maintainers, and the broader community, several iterations on the design of the proposal, and some effort at prototyping the proposed change. Often, it takes weeks to months of discussion, re-design, and prototyping for a proposal to be accepted. It is therefore important to note that seeing a proposal through to its conclusion can be time-consuming and not all proposals may end up accepted, although they may teach us all something!

If you’re motivated enough to go through this involved but rewarding process, go on with writing and keep on reading.

## What kind of proposals are appropriate for an HFTP?

Many people in Haskell donate a considerable amount of their time to improving Haskell and its ecosystem for free. However, sometimes the financial or human cost of a project is too great for the individual, or creates an undue out-of-pocket expense in order to support those contributions. This is where the HFTP process comes in, and what separates this process from the core libraries process, or a GHC proposal: the HF is capable of contributing time, money, and resources to make a proposal happen. 

For example, suppose a contributor needs computational resources for CI, or to produce stable Hackage subsets. Or for another example, say a contributor wants to write educational materials covering a yet-unwritten-about portion of Haskell, but requires an editor. Yet another example would be funding Summer of Code-type work on core tooling for Haskell. These costs should not come out of pocket. 

## What's the process for submitting a HFTP?

There are four major steps in the HFTP process:

1. Initial informal discussion 
2. Submission
3. Formal Review
4. Acceptance

### Initial Discussion

Before submitting a HFTP, it is required that you perform necessary preparations:

- Discuss your idea on the [Haskell.org Discourse](https://discourse.haskell.org/). Currently, we suggest cross-posting on the Haskell Foundation Slack for higher volume commentary. Create a Discourse topic under the category "Haskell Foundation", that starts with "Pre-HFTP” and briefly describe what you would like to change and why you think it’s a good idea.

- Proposing your ideas on the Discourse is not an optional step. For every change to the ecosystem, it is important to engage in due diligence with the community and relevant stakeholders. Use this step to promote your idea and gather early feedback on your Pre-HFTP proposal. It may happen that experts and community members may have tried something similar in the past and may offer valuable advice.


### Submission

A HFTP is a Markdown document written in conformance with the [process template](https://github.com/haskellfoundation/tech-proposals/blob/main/TEMPLATE.md). When such changes significantly alter an existing library or tool, the author is invited to provide a proof of concept. Delivering a basic implementation can speed up the process dramatically. If your changes are big or somewhat controversial, don’t let people hypothesize about them and show results upfront. Additionally, it would be ideal if the author of the proposal reached out to any stakeholders (e.g. library authors, maintainers) affected by the proposal and bring them into the formal discussion at this point.

A HFTP is submitted as a pull request against [the official Haskell tech proposal
repo](https://github.com/haskellfoundation/tech-proposals). Within a week of receiving the pull request, members of the HFTT will acknowledge your submission, validate that it conforms to the proposal template guidelines (see: [TEMPLATE.md](TEMPLATE.md)) and provide feedback to improve the overall quality of the document (if necessary). When the document conforms to the template guidelines, it is ready for formal evaluation by the HFTT members, as well as the broader community.

### Formal Review (up to 5 iterations)

While the majority of general technical commentary and feedback will occur on the proposal pull request and its associated issue, it's hard to tell when a particular proposal is finalized. This what we call "Formal Review".  The entire community is strongly encouraged to comment on, and help improve, a proposal. Ultimately, however, the Foundation needs a mechanism to make a decision, to accept, reject, or push back a proposal. This process is called "Formal Review", and is carried out by the HFTT working group.

#### Formal Review 

Formal Review of a proposal is done in iterations. These iterations take place in the HFTT meetings and are usually monthly. However, they can last longer, in which case the author has more time to implement all the required changes. 

The maximum number of iterations is five. At the fifth iteration, the HFTT can only vote to Accept, Postpone, or Reject.

The HFTT decides the duration of the next iteration depending upon the feedback and complexity of the HFTP. Consequently, authors have more time to prepare all the changes. If they finish their revision before the scheduled iteration, an HFTT member will reschedule it for the next available meeting.

During every iteration, the HFTT reviews the changes (updated design document, progress with the implementation, etc) to the proposal. Based on the feedback, the HFTP is either:

1. **Accepted**, in which case the HFTT has accepted the proposal, and it will be merged by the HFTT members.
2. **Rejected**, in which case the HFTP is closed and no longer evaluated in the future.
3. **Postponed**, in which case the HFTT sets aside the HFTP under some conditions. When those conditions are met, the HFTP can be resubmitted.
4. **Under revision**, in which case the author needs to continue the formal evaluation and address all the HFTT feedback. Thus, the follow-up discussion is scheduled for the next iteration. 
5. **Dormant**,  in which case no changes have been made to a HFTP in two iterations, it’s marked as dormant and both the PR and issue are closed. Dormant HFTPs can be reopened by any person, be it the same or different authors, at which point it will start from the formal evaluation phase.

#### Involving Stakeholders

At this point where a proposal is being reviewed, if there are relevant stakeholders from industry or in the community (e.g. library authors and maintainers who are affected), they should be solicited by the proposal author for comment and brought into the discussion. Remember, proposals may sound good, but it's best to iron out as many details as possible prior to their acceptance. This means that any hesitations stakeholders have with the project should be factored into the proposal's details. 

#### Updating Proposal Statuses

If the author of a particular proposal wants to update the status of their proposal, they should comment on the issue and alert the HFTT using the Github team tag `haskellfoundation/tech-proposals` with a note regarding what status they would like to update the proposal to. It is good to be redundant and raise the question in our other fora as well: Slack and Discourse. However, Github is required.  

### Acceptance

Upon acceptance, an HFTP is merged into the repository by an HFTT member, and the HFTP is assigned a shepherd from the HFTT. A shepherd is chosen at random, and will serve as a liason for the project, guiding its progress. Shepherds are required to report dutifully and accurately on the progress at the bi-weekly HFTT standup meetings. Project leaders are welcome to arrange alternative reporting schema (e.g. as their schedules allow, or if time off is required) upon request, as long as reports are given on a consistent basis. 

## The Role of the HFTT

Authors are responsible for building consensus within the community and documenting dissenting opinions before the HFTP is officially discussed by the HFTT. Their goal is to convince the HFTT that their proposal is useful and addresses pertinent problems in the Haskell ecosystem as well as interactions with already existing features. Authors can change over the life-cycle of the HFTP. For a formal charter, please see [CHARTER.md](CHARTER.md).

### The HF Technical Track

The HF Technical Track is an experienced group of people with knowledge of the Haskell ecosystem, responsible for the strategic technical direction on behalf of the Haskell Foundation. Members are tasked with:

- communicating with the community
- weighing in pros and cons of every proposal
- accepting, postponing or rejecting the proposal.

HFTT members should be either individuals responsible for a specific part of the Haskell ecosystem, or contributors and committers to parts of it. The members are selected by the HF CTO based on their expertise and reputation in the community.

The current HFTT members are:

- Emily Pillmore ([@emilypi](https://github.com/emilypi)), Haskell Foundation
- Richard Eisenberg ([@goldfirere](https://github.com/goldfirere)), Tweag
- Michael Snoyman ([@snoyberg](https://github.com/snoyberg)), FPComplete
- Andrew Lelechenko ([@Bodigrim](https://github.com/Bodigrim)), Barclays
- Davean Scies ([@davean](https://github.com/davean)), XKCD
- Edward Kmett ([@ekmett](https://github.com/ekmett)), MIRI
- Theophile Choutri ([@Kleidukos](https://github.com/Kleidukos)), Scrive
- Gil Mizrahi (@soupi)

### Voting

When a HFTP is scheduled at an HFTT meeting (i.e. it is in 'Under Review'), it can be held to a vote for one of the following outcomes (in case it makes a difference: the HFTP will be marked according to the first on this list to have a majority):

- Accepted (needs 66% of the HFTT to vote in favor, and the HFTP must specify a proposal lead, e.g. an implementor or project director, from the HFTT who will represent the work done on the project.)
- Dormant (needs a simple majority, an HFTT member that voted in favor will close the issue and PR and mark it as Dormant)
- Postponed (needs a simple majority, an HFTT member that voted to postpone will close the issue and PR, and write clear conditions for reopening)
- Revision needed (needs a simple majority. This can only be the outcome of the vote four times, for a total of five rounds. An HFTT member will write up what revisions are needed.)

If none of these applies, the proposal is Rejected.

### Responsibilities of the members

- Play a role in the discussions, learn in advance about the topic if needed, and make up their mind in the voting process.
- communicating with the community regarding technical the Haskell Foundation Technical Proposal (HFTP) process goings-on
- weighing in pros and cons of every HFTP
- accepting, postponing or rejecting each HFTP
- If shepherding, accurately and dutifully report the status of HFTPs 

### Guests

Experts in some fields may be invited to specific meetings as guests when discussing related HFTPs. Their input would be important to discuss the current state of the proposal, both its design and implementation.

### Joining the HFTT

If you would like to join the HFTT or would like to know how members are elected, please refer to the [charter](CHARTER.md)

## Proposal states

The state of a proposal changes over time depending on the phase of the process and the decisions taken by the HFTT. A given proposal can be in one of several states:

0. **Discussion:** The proposal is being discussed informally on discourse. In this phase the proposal only exists informally, no pull request exists yet.
1. **Submitted:** A pull request has been opened against the proposals repo.
2. **Validated:** The submitted proposal has been validated for conformity to the [proposal template](TEMPLATE.md), and made all necessary changes to the proposal name and its location in the repo.
3. **Under review:** The proposal will be under review until the next available Tech Track meeting takes place.
4. **Under revision:** Authors are addressing the issues pinpointed throughout the discussion and feedback process from the community and/or the HFTT.
5. **Dormant:** When a HFTP has been under revision for more than two iterations (that is, no progress has been made since the last review), it’s considered dormant, in which case any related activity will be paralyzed and the HFTT will not allocate more resources to it.
6. **Postponed:** The HFTP has been postponed under some concrete conditions. When these are met, the HFTP can be resubmitted.
7. **Rejected:** The HFTP has been rejected with a clear and full explanation.
8. **Merged:** The HFTP has been accepted and has been merged into the repo.

When a proposal has movede Haskell Foundation. Members are tasked with from one state to another, it will be appropriately labeled using the Github label system. The labels will be as they are above.

## How do I submit? ##

The process to submit is simple:

* Submit a Pre-HFTP topic to the HF Discourse, soliciting informal feedback.
* If the feedback is favorable, fork the Haskell Foundation tech-proposals repository, [https://github.com/haskellfoundation/tech-proposals](https://github.com/haskellfoundation/tech-proposals).
* Create a new HFTP file in the `HFTPs/` directory. Use the [HFTP template](https://github.com/haskellfoundation/tech-proposals/blob/main/TEMPLATE.md)
 * Make sure the new file follows the format: `YYYY-MM-dd-{title}.md`. Use the current date for `YYYY-MM-dd`.
 * Use the [Markdown Syntax](http://daringfireball.net/projects/markdown/syntax) to write your HFTP.
* Commit your changes to your forked repository
* Create a new [pull request](https://github.com/haskellfoundation/tech-proposals/pull/new).
* Notify the HFTT on Github using the team label `@haskellfoundation/tech-proposals`. Optionally, the address the HFTT Slack instance or on Discourse (or both!).