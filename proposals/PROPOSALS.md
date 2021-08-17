A HIP (*Haskell Foundation Initiative Proposal*) is a process for submitting a proposal for an initiative to be undertaken by the Haskell Foundation (HF). The main motivation of this document is to detail the primary mechanism through which the community may propose, discuss and implement initiatives to be carried out through the Haskell Foundation that will affect the Haskell ecosystem. For each HIP, all changes to the ecosystem go through a proposal process, which are is discussed by the HF Technical Track, the broader community, and any and all interested stakeholders. Only upon reaching a consensus are initiatives accepted to be merged and executed.

The aim of the Haskell Improvement Process is to apply the openness and collaboration that have shaped Haskell's documentation and implementation to a process of evolving the HF and the broader Haskell ecosystem. This document captures our guidelines, commitments and expectations regarding this process.

## Why Write a Proposal?

HIPs are key to making the HF and its initiatives better for the good of everyone. If you decide to invest the time and effort of putting a proposal forward and see it through, your efforts and time will shape and improve the Haskell ecosystem, which means that your proposal may impact the life of a myriad of developers all over the world, including those on your own team. For many, this aspect alone can be quite worthwhile.

However, it's important to note that seeing a proposal through to its conclusion is an involved task. On the one hand, it takes time to convince people that your suggestions are a worthwhile change for hundreds of thousands of developers to accept. Particularly given the sheer volume of developers that could be affected by a proposal, its acceptance is conservative and carefully thought through. Typically, this includes many rounds of discussion with HF leaders, Haskell library maintainers, and the broader community, several iterations on the design of the proposal, and some effort at prototyping the proposed change. Often, it takes weeks to months of discussion, re-design, and prototyping for a proposal to be accepted. It is therefore important to note that seeing a proposal through to its conclusion can be time-consuming and not all proposals may end up accepted, although they may teach us all something!

If you’re motivated enough to go through this involved but rewarding process, go on with writing and keep on reading.

## What's the process for submitting a HIP?

There are five major steps in the HIP process:

1. Initial informal discussion 
2. Submission
3. Formal Review
4. Acceptance
5. Formal presentation

### Initial Discussion

Before submitting a HIP, it is required that you perform necessary preparations:

- Discuss your idea on the [Haskell.org Discourse](https://discourse.haskell.org/). Currently, we suggest cross-posting on the Haskell Foundation Slack for higher volume commentary. Create a Discourse topic under the category "Haskell Foundation", that starts with "Pre-HIP” and briefly describe what you would like to change and why you think it’s a good idea.

- Proposing your ideas on the Discourse is not an optional step. For every change to the ecosystem, it is important to engage in due diligence with the community and relevant stakeholders. Use this step to promote your idea and gather early feedback on your Pre-HIP proposal. It may happen that experts and community members may have tried something similar in the past and may offer valuable advice.


### Submission

A HIP is a Markdown document written in conformance with the [process template](https://github.com/haskellfoundation/tech-proposals/blob/main/TEMPLATE.md). When such changes significantly alter an existing library or tool, the author is invited to provide a proof of concept. Delivering a basic implementation can speed up the process dramatically. If your changes are big or somewhat controversial, don’t let people hypothesize about them and show results upfront.

A HIP is submitted as a pull request against [the official Haskell tech proposal
repo](https://github.com/haskellfoundation/tech-proposals). Within a week of receiving the pull request, members of the HF Technical Track will acknowledge your submission, validate that it conforms to the proposal template guidelines (see: [TEMPLATE.md](TEMPLATE.md)) and provide feedback to improve the overall quality of the document (if necessary). When the document conforms to the template guidelines, it is ready for formal evaluation by the HF Technical Track members, as well as the broader community. 

### Formal Review (up to 5 iterations)

While the majority of general technical commentary and feedback will occur in the proposals repo, it's hard to tell when a particular proposal is finalized. This what we call "Formal Review".  

#### Formal Review 

Formal Review of a proposal is done in iterations. These iterations take place in the HF Tech Track meetings and are usually monthly. However, they can last longer, in which case the author has more time to implement all the required changes. 

The maximum number of iterations is five. At the fifth iteration, the HF Technical Track can only vote to Accept, Postpone, or Reject.

The HF Technical Track decides the duration of the next iteration depending upon the feedback and complexity of the HIP. Consequently, authors have more time to prepare all the changes. If they finish their revision before the scheduled iteration, an HF Technical Track member will reschedule it for the next available meeting.

During every iteration, the HF Technical Track reviews the changes (updated design document, progress with the implementation, etc) to the proposal. Based on the feedback, the HIP is either:

1. **Accepted**, in which case the HF Technical Track has accepted the proposal, and it will be merged by the HF Technical Track members.
2. **Rejected**, in which case the HIP is closed and no longer evaluated in the future.
3. **Postponed**, in which case the HF Technical Track sets aside the HIP under some conditions. When those conditions are met, the HIP can be resubmitted.
4. **Under revision**, in which case the author needs to continue the formal evaluation and address all the HF Technical Track feedback. Thus, the follow-up discussion is scheduled for the next iteration. 
5. **Dormant**,  in which case no changes have been made to a HIP in two iterations, it’s marked as dormant and both the PR and issue are closed. Dormant HIPs can be reopened by any person, be it the same or different authors, at which point it will start from the formal evaluation phase.

#### Involving Stakeholders

At this point where a propsal is being reviewed, if there are relevant stakeholders from industry or in the community (e.g. library authors and maintainers who are affected), they should be solicited for comment and brought into the discussion. Remember, proposals may sound good, but it's best to iron out as many details as possible prior to their acceptance. This means that any hesitations stakeholders have with the project should be factored into the proposal's details. 

#### Updating Proposal Statuses

If the author of a particular proposal wants to update the status of their proposal, they should either comment on the issue and tag an HF Technical Track member with a note regarding what status they would like to update the proposal to. It is good to be redundant and raise the question in our other fora as well: Slack and Discourse. However, Github should be the first choice. 

### Formal presentation 

After a proposal is merged, the author will present KPI's and deliverables that define the proposal to the HF Technical track. During the next available Technical Track meeting after acceptance, the author or designated lead (as defined by the proposal) should give a short 5-10min presentation consisting of the following: 

1. **High level summary:** the author will present the high-level abstract for what the proposal intends to accomplish. 
2. **Timeline:** In what timeline, and should outline possible hurdles and risks that affect the timeline. 
3. **Resources:** What resources are required in order to take the project from start to completion.
4.  **Contact:** Who is going to update the HF Technical Track members, and on what schedule.

Once the formal presentation is given, the HF Technical Track members will track the progress over time at our ongoing meetings. This also gives the track members time to introduce themselves and and we can all get to know each other. Currently, HF Technical Track meetings are held weekly on Tuesdays at 11:30am-12:30pm UTC-5 (EST). 

## Structure of the process

The HIP process involves the following parties:

1. The HIP Authors
2. Relevant stakeholders affected by the change
3. The HF Technical Track members

### The HIP Authors

Authors are responsible for building consensus within the community and documenting dissenting opinions before the HIP is officially discussed by the HIP HF Technical Track . Their goal is to convince the HF Technical Track that their proposal is useful and addresses pertinent problems in the Haskell ecosystem as well as interactions with already existing features. Authors can change over the life-cycle of the HIP.

### The HF Technical Track

The HF Technical Track is an experienced group of people with knowledge of the Haskell ecosystem, responsible for the strategic technical direction on behalf of the Haskell Foundation. Members are tasked with

  - communicating with the community
  - weighing in pros and cons of every proposal
  - accepting, postponing or rejecting the proposal.

HF Technical Track members should be either individuals responsible for a specific part of the Haskell ecosystem, or contributors and committers to parts of it. The members are selected by the HF CTO based on their expertise and implication in the community.

The current HF Technical Track members are:

- Emily Pillmore ([@emilypi](https://github.com/emilypi)), Haskell Foundation
- Richard Eisenberg ([@goldfirere](https://github.com/goldfirere)), Tweag
- Michael Snoyman ([@snoyberg](https://github.com/snoyberg)), FPComplete
- Andrew Lelechenko ([@Bodigrim](https://github.com/Bodigrim)), Barclays
- Davean Scies ([@davean](https://github.com/davean)), XKCD
- Edward Kmett ([@ekmett](https://github.com/ekmett)), MIRI
- Theophile Choutri ([@Kleidukos](https://github.com/Kleidukos)), Scrive
- Gil Mizrahi (@soupi)

### Voting

When a HIP is scheduled at an HF Technical Track meeting (i.e. it is in 'Under Review'), it can be held to a vote for one of the following outcomes (in case it makes a difference: the HIP will be marked according to the first on this list to have a majority):

- Accepted (needs 66% of the HF Technical Track to vote in favor, and the HIP must specify a proposal lead, e.g. an implementor or project director, from the HF Technical Track who will represent the work done on the project.)
- Dormant (needs a simple majority, an HF Technical Track member that voted in favor will close the issue and PR and mark it as Dormant)
- Postponed (needs a simple majority, an HF Technical Track member that voted to postpone will close the issue and PR, and write clear conditions for reopening)
- Revision needed (needs a simple majority. This can only be the outcome of the vote four times, for a total of five rounds. An HF Technical Track member will write up what revisions are needed.)

If none of these applies, the proposal is Rejected.

### Responsibilities of the members

- Review all HIPs 
- Play a role in the discussions, learn in advance about the topic if needed, and make up their mind in the voting process.

### Guests

Experts in some fields may be invited to specific meetings as guests when discussing related HIPs. Their input would be important to discuss the current state of the proposal, both its design and implementation.

## Proposal states

The state of a proposal changes over time depending on the phase of the process and the decisions taken by the HF Technical Track. A given proposal can be in one of several states:

0. **Discussion:** The proposal is being discussed informally on discourse. In this phase the proposal only exists informally, no pull request exists yet.
1. **Submitted:** A pull request has been opened against the proposals repo.
2. **Validated:** The submitted proposal has been validated for conformity to the [proposal template](proposals/TEMPLATE.md), and made all necessary changes to the proposal name and its location in the repo.
3. **Under review:** The proposal will be under review until the next available Tech Track meeting takes place.
4. **Under revision:** Authors are addressing the issues pinpointed throughout the discussion and feedback process from the community and/or the HF Technical Track.
5. **Dormant:** When a HIP has been under revision for more than two iterations (that is, no progress has been made since the last review), it’s considered dormant, in which case any related activity will be paralyzed and the HF Tech Track will not allocate more resources to it.
6. **Postponed:** The HIP has been postponed under some concrete conditions. When these are met, the HIP can be resubmitted.
7. **Rejected:** The HIP has been rejected with a clear and full explanation.
8. **Merged:** The HIP has been accepted and has been merged into the repo.

## How do I submit? ##

The process to submit is simple:

* Submit a Pre-HIP topic to the HF Discourse, soliciting informal feedback.
* If the feedback is favorable, fork the Haskell Foundation tech-proposals repository, [https://github.com/haskellfoundation/tech-proposals](https://github.com/haskellfoundation/tech-proposals).
* Create a new HIP file in the `hips/` directory. Use the [HIP template](https://github.com/haskellfoundation/tech-proposals/blob/main/TEMPLATE.md)
 * Make sure the new file follows the format: `YYYY-MM-dd-{title}.md`. Use the current date for `YYYY-MM-dd`.
 * Use the [Markdown Syntax](http://daringfireball.net/projects/markdown/syntax) to write your HIP.
* Commit your changes to your forked repository
* Create a new [pull request](https://github.com/haskellfoundation/tech-proposals/pull/new).
* Notify the Haskell HIP team on the Slack instance or on Discourse (or both!).
