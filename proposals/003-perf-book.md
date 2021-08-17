# Haskell Performance Tuning Book

##  Introduction

Performance matters, and Haskell’s runtime and memory models are sufficiently different from traditional programming languages to require additional knowledge, which isn’t commonly covered by most learning resources, in order to effectively debug and tune the performance of Haskell software.

This proposal aims to bridge this gap by creating a learning resource in the form of a book describing how to write fast idiomatic Haskell, and how to debug performance issues and tune Haskell programs.

## Goals

We would like to create a **freely available online book** which:

- Is relatively **short**
- Can be **maintained over time**, to make sure this knowledge remains accurate and up-to-date
- Targets **hobbyists and professional** Haskell developers
- Focuses on **real world** usage and techniques

These goals lead to the following suggestions:

- Writing this book should be a **community-driven** effort
- This book should focus on writing **idiomatic Haskell** code that is fast
  rather than advanced state-of-the-art techniques
- The book should cover the **principles and mindset** of performant Haskell
  code as well as provide concrete examples of debugging and tuning the
  performance of a Haskell project using **case studies**
- The book should **cover a modern "best-practice" toolset** used to generate
  data for examining a program's characteristics, and **methods for analyzing**
  and understanding the characteristics of a program using that data

## Motivation

One motivation for this book is that it can help **drive the adoption** of Haskell
in professional settings by providing knowledge regarding what to do when debugging
and tuning performance is necessary, and help **improve the confidence** of
practitioners and would-be practitioners.

**Currently, there is no clear path** to becoming proficient in writing performant Haskell.
One must search the web for tutorials and blogs posts in the hopes of finding up to date
information, and keep an eye on the ever changing landscape of tools and techniques.

Creating a book would provide a clear and up-to-date path for learners to learn
about performant Haskell, and will help advanced users of Haskell to refresh their knowledge
on latest advances and best practices as well.

## Prior Art (if applicable)

- [Parallel and Concurrent Programming in Haskell / Simon Marlow (2013, O'Reilly)](https://simonmar.github.io/pages/pcph.html)
- [Haskell High Performance Programming / Samuli Thomasson (2016, Packt)](https://www.packtpub.com/product/haskell-high-performance-programming/9781786464217)


## Timeline

## Outcomes

1. A freely available online book, hosted on Github
2. A repository containing the content of the book in markdown format, infrastructure for updating it, and a contribution process and guidelines
3. Content covering the latest developments in tooling, techniques and best practices with regards to performance programming in Haskell

## Procedure

Some work on this book has already started, specifically, drafting a [ToC](https://soupi.github.io/gotta-go-fast/01-about/00-about.html), organizing book infrastructure and tooling using github actions and [mdbook](https://github.com/rust-lang/mdBook), and drafting a [contributing guide](https://github.com/soupi/gotta-go-fast/blob/rfc/CONTRIBUTING.md).

## Help from the Haskell Foundation

There are a few items we would like help from the Haskell Foundation with that would greatly improve the ability to deliver a high quality book:

- Help with **getting the word out** about the project and looking for contributors
- Finding an **expert advisor** (or more than one) who could direct the “spirit” of the book
- Finding a **editor** (or more than one) and potentially paying them to ensure the quality of the text
- Paying for **design resources** for graphics and overall look and feel in case those are needed
- Asking for **feedback and/or contribution** from specific people or companies to ensure the quality and validity of the information and solutions presented in the book

## Risks

- Submitted content must abide by our [Guidelines for Respectful Communication](https://haskell.foundation/guidelines-for-respectful-communication).
- Since the knowledge of how to write fast idiomatic Haskell isn’t very common, we risk not having many **contributors**.
- **Maintenance** of the content, code and tooling usage can be difficult to achieve.
- By making this project community-driven, contributions will likely **vary in quality and style**.
- Advanced knowledge on how to make Haskell blazingly fast will still remain folklore knowledge.

## Acknowledgements

There are many important resources on Haskell performance on the internet and we’d like to acknowledge them. A few of them can be viewed on the [reference list](https://soupi.github.io/gotta-go-fast/07-conclusion/00-next.html) for the book.
