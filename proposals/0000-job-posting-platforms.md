# Curated list of community job advertisement platforms

## Abstract

Job postings and hiring threads are an important part of any language community. The Haskell Foundation
should maintain a list of well-established free platforms where companies can advertise positions, while
also incentivizing a common moderation standard for those platforms by means of inclusion in said list.

## Background

Currently, there are a couple of well-known places where companies can advertise their jobs and positions for free, such as:

* Haskell Discourse: [https://discourse.haskell.org/](https://discourse.haskell.org/)
* Haskell reddit: [https://www.reddit.com/r/haskell/](https://www.reddit.com/r/haskell/)
* Haskell weekly: [https://haskellweekly.news/advertising.html](https://haskellweekly.news/advertising.html)

These places have seen a fair amount of growth in job postings, which can also be seen by the following
data analysis: https://github.com/nh2/haskell-jobs-statistics

The Haskell Foundation should maintain a curated list of such platforms for convenient visibility
for companies, while also having inclusion requirements/criteria that promote healthy civil conduct.

## Problem Statement

One of the main problems this proposal is trying to solve is the frequent toxic nature of hiring threads,
which often derail for various reasons:

* someone doesn't like the industry (e.g. blockchain) and starts arguing why no one should ever work in that industry
* someone doesn't like the company or CEO and goes on a political rant why those are bad people and no one should work in that company
* someone doesn't like the proposed job conditions and goes on a rant why capitalism is the root of all evil
* someone feels the need to express their political views and takes a job posting as an opportunity to do so

Just to give a few examples of such threads:

* [https://discourse.haskell.org/t/senior-platform-engineer-tlon-hosting-platform-remote/5132](https://discourse.haskell.org/t/senior-platform-engineer-tlon-hosting-platform-remote/5132)
* [https://discourse.haskell.org/t/anduril-industries-is-hiring-haskell-developers/7089](https://discourse.haskell.org/t/anduril-industries-is-hiring-haskell-developers/7089)
* [https://www.reddit.com/r/haskell/comments/uozcvx/haskellers_needed_at_superrare_labs/?utm_source=share&utm_medium=web2x&context=3](https://www.reddit.com/r/haskell/comments/uozcvx/haskellers_needed_at_superrare_labs/?utm_source=share&utm_medium=web2x&context=3)
* [https://www.reddit.com/r/haskell/comments/ssnw8i/hiring_remote/?utm_source=share&utm_medium=web2x&context=3](https://www.reddit.com/r/haskell/comments/ssnw8i/hiring_remote/?utm_source=share&utm_medium=web2x&context=3)

This behavior of the community and a few engaged individuals has caused damage to the reputation of those
platforms in industry: some companies actively avoid posting on platforms, where users can leave replies and comments.
It causes chilling effects even for less controversial places, because for every company there may be people out there
that dislike them, their industry or their work for whatever reason. And those companies now face potential public shaming
or reputation damage.

This is purely a moderation issue. So the question is: how can the HF define and incentivize good moderation principles
on such platforms, without imposing any of those.

## Prior Art and Related Efforts

I have raised my concerns about moderation issues on both reddit and discourse in the past. From the (previous) reddit moderation
team, I got a reply, but no action was ever taken. The discourse admin team (or whoever is on the 'discourse-admin' alias/ML)
never replied.

I have also recently contacted Discourse admins in private, which seemed to foster some understanding and awareness of the problematic
situation.

My attempts to improve the situation by voicing my concerns directly to the appropriate moderation teams have failed.
As such, I think we can hack our own social behavior through incentivizing a standard set of moderation practices.

## Technical Content

We propose here that the Haskell Foundation maintains a list of platforms, where companies can advertise jobs.
A platform may be a social media site, a dedicated site for jobs, a forum or a newsletter, etc.

The inclusion criteria is as follows:

* advertising is free of charge
* the platform has a clear written policy about:
  - which type of job postings are allowed (e.g. which languages: Haskell/FP... or whether a salary range is a requirement)
  - which types of job postings are not allowed (e.g. industry XY is disallowed)
* if the platform allows users to reply or leave comments, then it must:
  - allow the job poster to request that the thread be locked or replies be disallowed (if the platform permits)
  - have the following moderation rules:
    - postings of personal opinions on companies or industries in job postings are considered spam and will be removed
    - postings that don't show the intention of engaging with the recruiter or asking genuine questions about the job are considered offtopic
	- generally, job postings are not discussion threads

We note that users who have opinions on politics, companies or industries can voice these in dedicated forums or threads or write personal
blog posts about it and link to them in a new thread (such as Stephen Diehl's [remarks on blockchain and cryptocurrency](https://www.stephendiehl.com/posts/crypto.html)).
As such, free speech is maintained.

Any platform that wishes to adhere to these criteria, can be included in the curated list of job posting platforms.
This proposals gives no details on how this list is implemented or presented.

## Timeline

None.

## Budget

None.

## Stakeholders

* companies looking to advertise jobs
* social media platforms and other platforms, where jobs can be advertised
* the moderation teams of those platforms
* Haskellers looking for jobs

## Success

* publishing of a curated list of platforms, where companies can advertise jobs
* listed platforms adhere to the moderation standards outlined in this document

