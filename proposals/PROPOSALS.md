# Haskell Foundation Technical Working Group

## Purpose

The purpose of the Technical Working Group (TWG) is to serve as a forum in which technical ideas that have the potential to benefit a broad selection of the Haskell community can be evaluated and discussed.
In particular, the TWG should provide a forum for all the following:

 * Requests For Comment (RFCs), in which a community member seeks feedback before embarking on a project, but is not requesting any further financial or administrative resources from the Haskell Foundation
 * Haskell Foundation project proposals, in which a community member requests that the Haskell Foundation organize and execute on a project idea
 * Community projects, in which a project with broad relevance to the Haskell community seeks additional resources that will allow them to achieve a goal

Collectively, the documents that serve as the basis for these discussions are called _Haskell Foundation Technical Proposals_ (HFTP).
Additionally, at its option, the TWG will host projects and discussions that don't directly fit into the above categories, but that seem to be useful for the community as a whole.

The TWG will address the following concerns:
 * Soliciting projects from the community, where relevant
 * Ensuring that relevant stakeholders have been contacted and that their input and needs have been considered
 * Guiding discussions in a kind and productive direction
 * Providing a final recommendation as to whether a project should continue
If the TWG does not reach a consensus, then the working group's decision will be made by majority vote.

Projects undertaken by the Haskell Foundation need not go through this process, and approval of a project by this working group does not guarantee that the Haskell Foundation will actually commit resources. Committee members are tasked with recommending whether a project is likely to be of benefit to the community, not for prioritizing the Foundation's limited resources. However, we expect that, given sufficient resources, recommended projects will be supported by the executive team.

The TWG is a successor to Haskell Foundation Tech Track (HFTT), and will initially consist of its members.

## Membership

The committee will consist of 8 community members and the Haskell Foundation's executive director (ED), or other executive team member.
Volunteers should be recruited in such a way as to achieve a diversity of technical knowledge and experiences, so as to best serve the entire Haskell community.
In other words, volunteers should be recruited who have knowledge of and experience with at least the following:
 * Using Haskell in a variety of environments (research, traditional business applications, free-time open-source work, large teams, small teams)
 * Using different build tools and dependency management techniques
 * Domain knowledge such as cryptography, security, performance, and databases
The ED will report on the current state of the committee's coverage of these areas as membership is updated.
Applications to join the committee will be decided on by the committee's members.
Inactive members may be removed by the ED after making a good-faith effort to contact and engage them.

The current members of the TWG are:
 * Davean Scies (@davean)
 * Gershom Bazerman (@gbaz)
 * John Ericson (@Ericson2314)
 * Hécate (@Kleidukos)
 * Evie Ciobanu (@eviefp)
 * David Thrane Christiansen (@david-christiansen)

## Decisions

The committee will make all decisions using a simple majority without secret ballots.
Discussions and vote tallies regarding potential committee members are to be kept strictly confidential, while other discussions and votes should be transparent to the public.
Decisions should be made with the participation of 2/3 of the current members of the committee, rounded up.
The committee may decide to hold regular meetings, or to collaborate asynchronously, or some mix of the two.
For instance, a valid decision can be made by having a non-quorum meeting discuss a matter, summarize the results for the other members, and then vote by email based on the summary.
Disputes will be resolved by the ED.

## Process

The TWG, as an advisory committee to the Haskell Foundation and to the community at large, exists primarily to aid in making good decisions. 

Proposals should explicitly state which **category** they are in (RFC, Community Project, HF Project), or describe why they do not fit into an existing category but should be considered by the committee anyway.
Additionally, all proposals should address the following concerns:
 * **Problem statement:** What problem is the proposed work intended to solve? What are the requirements against which a solution should be evaluated? How will solving this problem benefit the community?
 * **Prior art:** What other similar work has been done in the community, and how is the proposal related to it?
 * **Related efforts:** What other related activities are planned or ongoing? How is the proposal connected to them?
 * **Technical and organizational work**: What work is actually being proposed? What technology should be developed, and how will it be managed and cared for going forward?
 * **People:** Who will do the work?
 * **Success:** What does it mean for the project to have succeeded?
 * **Stakeholders:** Who will be affected, and how?
 * **Time:** How long will the proposal take to implement?

Projects seeking HF support or execution (i.e. Community Projects and HF Projects) should additionally address:
 * **Budget:** How much time and money will the project cost initially? What about ongoing maintenance?
 * **Additional partners:** Who else will participate in the project?

One way to address these concerns is to begin with one of the [proposal templates](templates/).


### Pre-Proposal
Prior to submitting a detailed proposal, please post to the Haskell Discourse instance with a summary of the proposal's contents and a subject line that begins with "Pre-HFTP:".
This helps address the prior art and related efforts concerns, and can save time needed to rewrite a proposal in light of previously-unknown opportunities for collaboration.
While submitters are strongly encouraged to discuss their proposal on Discourse before writing them, proposals will not be summarily rejected for not having done so.
It is sufficient to post a link to an existing draft of the proposal to solicit prior art and related work before revising and formally submitting the proposal.

### Proposal
Having gathered initial feedback, the next step is to write and submit a proposal.
Proposals should be submitted as pull requests to the Haskell Foundation `tech-proposals` repository, [https://github.com/haskellfoundation/tech-proposals](https://github.com/haskellfoundation/tech-proposals). Please notify the TWG on Github using the team label `@haskellfoundation/tech-proposals`.
The pull request should add a new HFTP file in the `proposals/` directory.

When writing a proposal, the [templates](templates/) provide a starting point for addressing the important concerns.
   * The proposal should be written in a single Markdown file.
   * The file should be named `0000-NAME.md`. Supporting materials, such as diagrams, example code, or spreadsheets, should be placed in a directory named `0000-NAME`, and all files should be in standard formats that are readable and editable using open-source software.
 
The committee will publicize the proposal in a variety of Haskell fora, as they deem appropriate for the contents of the proposal.
At the very least, all proposals will be announced on Discourse.

### Community Discussion and Revision

The next step is community discussion.
During community discussion, the committee has two roles: to constructively contribute technical insights, and to keep the discussion on track.
By "on track", we mean that discussion is relevant, respectful, and productive.

Discussing alternatives to the proposal or parts of it that may better achieve its goals is relevant, as is questioning whether the goals are in fact beneficial to the community. Promoting alternative goals or proposing ways to reach them is not. In other words, comments like "you should solve this other problem instead" are not encouraged, while comments like "why not use this other library to achieve the goals?" are.

Respectful discussion lives up to the [respectful communication guidelines](https://haskell.foundation/guidelines-for-respectful-communication/). Additionally, we must take special effort to listen to each others' experiences and take them seriously. Haskell is used in a variety of contexts with a variety of cultures, interests, and requirements, and it's important that we are able to understand these needs, even if we end up not being able to serve all of them. The needs of researchers, business application developers, and open-source contributors are all important, as are those of people in different industries, different countries, and with different backgrounds.

Productive discussion contributes new insights as it progresses, and doesn't go in circles. When a point has been made, there's usually no need to make it again, although it can be worth checking whether a particular need or interest is unique or shared.

From time to time, the committee will summarize the current state of knowledge in the discussion thread. When there is no longer productive discussion, the committee will proceed to the next step, recommendation.

### Recommendation

The final step of a proposal is for the committee to give a recommendation.
While the committee should take community discussion into account, they are expected to make recommendations based on their own knowledge, rather than simply reflecting the voices of the participants in the discussion.
Fundamentally, there are three potential recommendations: rejection, revision, and acceptance:

 * *Rejection* means that the committee does not believe that the proposal would, in the balance, bring value to the community, and that it should not be acted upon.
   Furthermore, rejection implies that the committee sees no route to making the proposal acceptable that is short of rewriting it from scratch.
   Among reasons for rejection may be that the problem itself is not considered significant, that the proposal has costs, harms, or overheads that outweigh the benefits, or that the proposal is simply unworkable or based on incorrect understanding and assumptions.
 * *Acceptance* means that the committee believes that the proposal should be acted upon as-written.
 * *Revision* means that the committee believes that the proposal as written should not be accepted, but that with some changes it could be accepted.
   This may be technical details, project governance issues, or clarity.

At most five revisions will be recommended.
After five revisions, the committee may only recommend acceptance or rejection.

Committee recommendations are decided by a simple majority vote.
The reasons for the recommendation should be written and added as a comment to the discussion thread.
The minority may, if they desire, write up their reasons for recommending otherwise, and decision makers may take this into account.
