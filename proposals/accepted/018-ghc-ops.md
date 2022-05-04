## Introduction


This proposal is for hiring a full-time DevOps engineer for GHC and related Haskell projects and tooling. This is necessary to relieve the drain on time of core GHC developers with specialized knowledge on DevOps work. From acceptance to hiring, this proposal should hopefully take two to four months. We see no significant technical risk to this proposal, but one modest concern to be aware of is that transition of DevOps could cause temporary disruption to stability of GHC developer experience.


## Background


Currently there is a significant amount of time being spent collectively by GHC Operations Contributors keeping systems working. Estimates given in the recent past suggest between Ben Gamari and Moritz Angermann they were spending 20 hours per week on systems.


Gitlab takes a moderate amount of attention and issues with it affect a large pool of contributors. Gitlab has had (unplanned) outages on 19 March, and 18 October of 2021. Both were caused by being in a reactive instead of proactive mode since this is no one's job. Full time attention to CI systems can help move into that proactive mode, not only dealing with individual problems as they arise, but stabilizing and making the core machinery more reliable in general -- i.e. dealing with anticipated problems before they occur. (For example, improving garbage-collection of build products and system monitoring).


In addition the CI systems are highly varied and numerous. Among them are around 30 runners, across at least 4 kernels - with more user spaces on top. Further we’re missing some CI targets due to a lack of available time and energy from the GHC Operations Contributors to set up and maintain them. GHC CI can not use most hosted CI reasonably because it is already tested on a wider variety of platforms than most provide, and some of its CI jobs far exceed resource limits due to its size.


GHC Operations Contributors include but are not limited to:
* Ben Gamari who is the primary maintainer of our GitLab nix configuration, and both the gitlab and associated gitlab-storage physical boxes. Additionally they run marge-bot and are a primary maintainer of the CI scripts, and several CI boxes.
* Moritz Angermann handles much of Mac CI, and some CI management in general.
* davean does some physical Mac cluster management, and supports some of the GitLab servers on a primarily emergency basis.


Operational needs extend beyond these core issues. Further basic developer experience tools like the GHC dashboard proposal are stalled for lack of time.


## Motivation


Hiring a dedicated person to maintain and improve the infrastructure, CI, and related tooling would alleviate the burden on some of GHC's core contributors. This would free these core contributors to make more optimal use of their time instead of spending it on devops - a position which we are more prepared to hire for and requires less onboarding. Further being preemptive in avoiding issues would improve the flow for every contributor. While the freed-up time from contributors might be expended in many productive directions, it is beyond the scope of this proposal to speculate in any detail.


Supporting GHC’s process is directly in line with the community glue and adoption directives of the Haskell Foundation -- it connects a wider audience of Haskellers than any other. Additionally it is a challenge for volunteers to reliably sustain this necessary and vital infrastructure. And further, this project should be as far upstream as possible, as GHC development infrastructure is the commons of the Haskell community.


With a full-time steward, the infrastructure could provide benefits beyond just GHC. Already, CI machinery is shared by cabal and ghcup, and a paid maintainer could further scale this out. In turn, this enhances the returns on other related Haskell Foundation-funded projects.


Responsibilities of a maintainer should include:
* Maintain gitlab & CI (including diagnosing surprising CI failures).
* Improve GHC CI execution (including marge).
* Expand CI coverage to the base ecosystem.
* Support related Haskell project CI.


Further responsibilities could include:
* Support additional Haskell server infrastructure (help to the admin team).
* Support OS packag(ing/ers) of Haskell (i.e. distros, such as debian, alpine, etc.).
* CI for additional distros and architectures.
* Assist in fixing hadrian.
* Documenting the infrastructure
* GHC performance dashboard


It should be noted that our CI infra today is not fixed in stone, and along with maintaining and incrementally evolving it, a qualified CI hire could well look into alternatives for particular components (such as marge), especially as the landscape evolves over time.


As the CI system genuinely evolves into something stable and requiring less ongoing work, the position will also evolve into covering other aspects of GHC and community ops work (some contemplated under “further responsibilities” above).


## Goals


Hire a dedicated devops person to support Haskell Foundation's interest in GHC and Haskell infrastructure.


## People


This section should detail the following people:


-   **Performers:**
Andrew Boardman: Personnel management, Leading search
Ben Gamari: Technical management


-   **Reviewers:** 
Andrew Boardman
Ben Gamari
Moritz Angermann


Most affected people:
* Ben Gamari
* Moritz Angermann
* The Haskell Infrastructure team
* All GHC developers, as well as cabal and ghcup
* Hopefully developers on other major Haskell projects




## Resources


The HF would provide the people management side via Andew Boardman in the CEO role, while the technical direction side of management would come from the GHC Operations Contributors - primarily Ben Gamari. 


In addition the Haskell Foundation would provide a budget currently estimated to be up to US$125k/year. This estimated budget is based on discussions with people familiar with such roles (either through direct experience or as managers) internationally. Given the current job market and the HF’s current funding, a hire around this price point should bring a reasonable amount of capacity to the table.


Finally HF, via the CEO role, would lead the candidate search in coordination with GHC Operations Community and other stakeholders.


## Timeline


Job listing writing: 1 week.
Candidate search period: up to 2 months.
Hiring process: up to 2 months, hopefully less.


## Lifecycle:


After the initial hiring process, this is expected to be an ongoing position. Past and current experience has demonstrated that CI work cannot be scoped to a short-term contract, but rather requires some degree of continuous attention and evolution. Platforms, build systems, OS concerns and architectures evolve over time, and the scope of release testing has a lot of headway to grow more comprehensive in many directions, including library coverage and tooling integration.


## Deliverables


Deliver one full-time developer operative within approximately 2-4 months. The exact legal arrangement is left to the Haskell Foundation Officers.


## Outcomes


We expect more time available from core ghc contributors currently bogged down in operations work, as well as more stability to the CI infrastructure, and ultimately ongoing improvements in features and coverage.


## Risks


* Hiring is hard. Especially since we’d prefer a wide mix of skills such as Nix, operations, and ideally Haskell. The bar is raised for hiring for a security sensitive position. Requires a budget sufficient to hire someone who suits the qualifications.
* In the short term, time spent in terms of mentoring and bringing a new hire up to speed will be required from the GHC Ops team. 
* We may require a more senior and with a more varied skill set than is feasible
   * We need someone who shows up and figures out what needs to be done independently.


* Hire competency will affect the stability of services.


* Hiring a new person is a significant financial burden to take on for the HF.


* Documentation and knowledge must be maintained so this person does not become a critical failure point.