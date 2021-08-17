A HIP (*Haskell Foundation Initiative Proposal*) is a process for submitting a proposal for an initiative to be undertaken by the Haskell Foundation (HF). The main motivation of this document is to detail the primary mechanism through which the community may propose, discuss and implement initiatives to be carried out through the Haskell Foundation that will affect the Haskell ecosystem. For each HIP, all changes to the ecosystem go through a proposal process, which are is discussed by the HF Technical Track, the broader community, and any and all interested stakeholders. Only upon reaching a consensus are initiatives accepted to be merged and executed.

The aim of the Haskell Improvement Process is to apply the openness and collaboration that have shaped Haskell's documentation and implementation to a process of evolving the HF and the broader Haskell ecosystem. This document captures our guidelines, commitments and expectations regarding this process.

## Why Write a Proposal?

HIPs are key to making the HF and its initiatives better for the good of everyone. If you decide to invest the time and effort of putting a proposal forward and see it through, your efforts and time will shape and improve the Haskell ecosystem, which means that your proposal may impact the life of a myriad of developers all over the world, including those on your own team. For many, this aspect alone can be quite worthwhile.

However, it's important to note that seeing a proposal through to its conclusion is an involved task. On the one hand, it takes time to convince people that your suggestions are a worthwhile change for hundreds of thousands of developers to accept. Particularly given the sheer volume of developers that could be affected by a proposal, its acceptance is conservative and carefully thought through. Typically, this includes many rounds of discussion with HF leaders, Haskell library maintainers, and the broader community, several iterations on the design of the proposal, and some effort at prototyping the proposed change. Often, it takes weeks to months of discussion, re-design, and prototyping for a proposal to be accepted. It is therefore important to note that seeing a proposal through to its conclusion can be time-consuming and not all proposals may end up accepted, although they may teach us all something!

If you’re motivated enough to go through this involved but rewarding process, go on with writing and keep on reading.

## What's the process for submitting a HIP?

There are four major steps in the HIP process:

1. Initial informal discussion (2 weeks)
2. Submission
3. Formal presentation (up to 1 month)
4. Formal evaluation (up to 2 months)

### Initial informal discussion (2 weeks)

Before submitting a HIP, it is required that you perform necessary preparations:

- Discuss your idea on the [Haskell.org Discourse](https://discourse.haskell.org/). Currently, we suggest cross-posting on the Haskell Foundation Slack for higher volume commentary. Create a Discourse topic under the category "Haskell Foundation", that starts with "Pre-HIP” and briefly describe what you would like to change and why you think it’s a good idea.

- Proposing your ideas on the Discourse is not an optional step. For every change to the ecosystem, it is important to engage in Due Diligence with the community and relevant stakeholders. Use this step to promote your idea and gather early feedback on your Pre-HIP proposal. It may happen that experts and community members may have tried something similar in the past and may offer valuable advice.


### Submission

A HIP is a Markdown document written in conformance with the [process template](https://github.com/haskellfoundation/tech-proposals/blob/main/TEMPLATE.md). When such changes significantly alter an existing library or tool, the author is invited to provide a proof of concept. Delivering a basic implementation can speed up the process dramatically. If your changes are big or somewhat controversial, don’t let people hypothesize about them and show results upfront.

A HIP is submitted as a pull request against [the official Haskell tech proposal
repo](https://github.com/haskellfoundation/tech-proposals). Within a week of receiving the pull request, members of the HF Technical Track will acknowledge your submission, validate that it conforms to the proposal template guidelines (see: [TEMPLATE.md](TEMPLATE.md)) and provide feedback to improve the overall quality of the document (if necessary). When the document conforms to the template guidelines, it is ready for formal evaluation by the HF Technical Track members, as well as the broader community. 

### Formal evaluation (up to 5 iterations)

While the majority of general technical commentary and feedback will occur in the proposals repo, it's hard to tell when a particular proposal is finalized. This what we call "Formal Evaluation". 

Formal Evaluation of a proposal is done in iterations. These iterations take place in the HF Tech Track meetings and are usually monthly. However, they can last longer, in which case the author has more time to implement all the required changes. 

The maximum number of iterations is five. At the fifth iteration, the HF Technical Track can only vote to Accept, Postpone, or Reject.

The HF Technical Track decides the duration of the next iteration depending upon the feedback and complexity of the HIP. Consequently, authors have more time to prepare all the changes. If they finish their revision before the scheduled iteration, an HF Technical Track member will reschedule it for the next available meeting.

During every iteration, the HF Technical Track reviews the changes (updated design document, progress with the implementation, etc) to the proposal. Based on the feedback, the HIP is either:

1. **Accepted**, in which case the HF Technical Track will propose a release date.
2. **Rejected**, in which case the HIP is closed and no longer evaluated in the future.
3. **Postponed**, in which case the HF Technical Track sets aside the HIP under some conditions. When those conditions are met, the HIP can be resubmitted.
4. **Under revision**, in which case the author needs to continue the formal evaluation and address all the HF Technical Track feedback. Thus, the follow-up discussion is scheduled for the next iteration.
5. **Dormant**,  in which case no changes have been made to a HIP in two iterations, it’s marked as dormant and both the PR and issue are closed. Dormant HIPs can be reopened by any person, be it the same or different authors, at which point it will start from the formal evaluation phase.

When the author and the HF Technical Track members agree on the final document, it is formally accepted: assigned a number, and scheduled for a final formal presentation.

### Formal presentation (up to 1 month)

During the next available Technical Track meeting, the author of the HIP presents the proposal to the audience. 

### Merging the proposal

If the HIP is accepted, the HF Technical Track will propose a release date, and the proposal will be merged directly by the HF Technical Track members.

## Structure of the process

The HIP process involves the following parties:

1. The HIP Authors
2. Relevant stakeholders affected by the change
3. The HF Technical Track Leads

### The HIP Authors

Authors are responsible for building consensus within the community and documenting dissenting opinions before the HIP is officially discussed by the HIP HF Technical Track . Their goal is to convince the HF Technical Track that their proposal is useful and addresses pertinent problems in the Haskell ecosystem as well as interactions with already existing features. Authors can change over the lifecycle of the HIP.

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

For a HIP to be accepted, it must fulfill two requirements:

- 66% of the HF Technical Track votes in favor of it
- The HIP specifies a proposal lead (e.g. an implementor, or project director) who will represent the work done on the project.

### Responsibilities of the members

- Review all HIPs 
- Play a role in the discussions, learn in advance about the topic if needed, and make up their mind in the voting process.

### Guests

Experts in some fields may be invited to specific meetings as guests when discussing related HIPs. Their input would be important to discuss the current state of the proposal, both its design and implementation.

## Proposal states

The state of a proposal changes over time depending on the phase of the process and the decisions taken by the HF Technical Track. A given proposal can be in one of several states:

0. **Discussion:** The proposal is being discussed informally on discourse. In this phase the proposal only exists informally, no pull request exists yet.

1. **Submitted:** A pull request has been opened against the proposals repo.

2. **Under review:** The proposal will be under review until the next available Tech Track meeting takes place.

3. **Under revision:** Authors are addressing the issues pinpointed throughout the discussion and feedback process from the community and/or the HF Technical Track.

6. **Dormant:** When a HIP has been under revision for more than two iterations (that is, no progress has been made since the last review), it’s considered dormant, in which case any related activity will be paralyzed and the HF Tech Track will not allocate more resources to it.

7. **Postponed:** The HIP has been postponed under some concrete conditions. When these are met, the HIP can be resubmitted.

8. **Rejected:** The HIP has been rejected with a clear and full explanation.

9. **Merged:** The HIP has been accepted and has been merged into the repo.

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
