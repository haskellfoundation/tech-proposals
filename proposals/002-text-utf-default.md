# Introduction

This proposal is for the migration of the `text` package from its
default UTF-16 encoding to UTF-8. The lack of UTF-8 as a default in the
`text` package is a pain point raised by the Haskell Community and
industry partners for many years. The Haskell Foundation is uniquely
positioned to effect a change like this, granted there is an appetite
for breakage, and that such a project is well-socialized and planned.
During the meetings of the Haskell Foundation Tech Agenda Track, we
identified the pros and cons of such a migration against historical
attempts, and agreed that it was within our power to do.

Further, we solicited both community and industry feedback to gauge the
appetite for breakage, and what stakeholders would be affected by the
changes. Andrew Lelechenko has offered to lead the implementation of
this project.

# What is Unicode?

Representing a text via bytes requires to choose a character enconding.
One of the most old encodings is ASCII: it has 2⁷=128 code points,
representing Latin letters, digits and punctuations. These code points
are trivially mapped to 7-bit sequences and stored byte by byte (with leading zero).

Unfortunately, almost no other alphabet except English can fit into ASCII:
even Western European languages need code points for ñ, ß, ø, etc.
Since on the majority of architectures a byte contains 8 bits, able to encode
up to 256 code points, various incompatible ASCII extensions proliferated profusely.
This includes ISO 8859-1, covering additional symbols for Latin scripts, several
encodings to represent Cyrillic letters, etc.

Not only it was error-prone to guess an encoding of a particular bytestring, but also
impossible, for example, to mix French and Russian letters in a single text. This
prompted work on a universal encoding, Unicode, which would be capable to represent
all letters one can think of. At early development stages it was thought that
“64K ought to be enough for anybody” and a uniform 16-bit encoding UCS-2 was proposed.
Some programming languages developed around that period (Java, JavaScript)
chose this encoding for internal representation of strings. It is
only twice longer than ASCII and, since all code points are
represented by 2 bytes, constant-time indexing is possible.

Soon enough, however, Unicode Consortium discovered more than 64K letters.
Unicode standard defines almost 17·2¹⁶ code points, of which \~143 000 are assigned
a character (as of Unicode 13.0). Now comes a tricky part: Unicode defines how to map
characters to code points (basically, integers), but how would you serialise lists of
integers to bytes? The simplest encoding is just allocate 32 bits (4 bytes) per code
point, and write them one by one. This is UTF-32. Its main benefit is that since
all code points take the same size, so you can still index characters in a constant time.
However, memory requirements are 4x comparing to ASCII, and in a world of ASCII and UCS-2
there was little appetite to embrace one more, completely new encoding.

Next option on the list is to encode some code points as 2 bytes and some others,
less lucky ones, as 4 bytes. This is UTF-16. This encoding allowed to retain a decent
backward compatibility with UCS-2, so, for instance, modern Java and JavaScript
stick to UTF-16 as default internal representation. The biggest downside is that you
no longer know beforehand, where *n*-th code point starts. One must *parse* all characters
from the very beginning to learn which ones are 2-byte long and which are 4-byte long.

But once we abandon requirement of constant indexing, even better option arises. Let's
encode first 128 characters as 1 byte, some others as 2 bytes, and the rest as 4 bytes.
This is UTF-8. The killer feature of this encoding is that it's fully backwards compatible
with ASCII. This meant that all existing ASCII documents were automatically valid UTF-8
documents as well, and that 50-years-old executables could often parse UTF08 data without
knowing a bit about it. This property appeared so important that in a modern environment
the vast majority of data is stored and sent between processes in UTF-8 encoding.

To sum up:

* Both UTF-8 and UTF-16 support exactly the same range of characters.
* For ASCII data UTF-8 takes twice less space.
* UTF-8 is vastly more popular for serialization and storage.

# Motivation

-   UTF-16 by default requires that all Text values pay a premium for serialization. Arguably, the performance impact of Text is flipped
    upside-down: most text is UTF-8, and Haskell devs pay an undue cost when working with the wrong default.

-   UTF-8 is the industry standard and by far the most common text encoding, with roughly 97% of web pages existing in UTF-8. The
    existing UTF-16 default imposes an additional hurdle to working with the vast majority of web content on earth.

-   Many systems in Haskell are UTF-8 by default (e.g. Haddock)

# Goals

-   Solicit feedback from community members and industry to gauge appetite for such a change.

-   Provide an implementation, migration, and delivery plan for changing the default encoding of `text` from UTF-16 to UTF-8.

-   Ensure stakeholders (e.g. GHC, Cabal, Stack, boot libs) have ample time to migrate and address any bugs.

-   Implementation should not significantly alter the performance characteristics of the base `text` library within some tolerance
    threshold.

# People

-   Performers:

    -   Leader: Andrew Lelechenko (bodigrim)

    -   Support: Emily Pillmore (emilypi)

-   Reviewers:

    -   The text maintainers

        -   Xia Li-Yao (lysxia)

        -   Emily Pillmore (emilypi)

        -   Dan Cartwright (chessai)

        -   Callan McGill (boarders)

-   Stakeholders:

    -   Edward Kmett: has been vocal about his use of `text-icu` and requires it not be broken.

-   Ben Gamari: integration with GHC

-   The Cabal maintainers (fgaz, emilypi, mikolaj): integration with Cabal

Progress will be reported on a weekly basis to the HF Technical Agenda
Track, with Emily as support for Andrew.

# Timeline

We expect that this project will take roughly 6 months to fully
complete: 3-4 months to complete the code implementation, performance
testing, and unit testing, another 1-2 months to integrate with
stakeholders and diagnose any potential issues with the migration.

**Preparation:**

Using the HVR's existing [`text-utf8`](https://github.com/text-utf8) as
a starting point, the following must be done before an implementation is
started:

-   Modernize the codebase and clear out the bitrot

-   Establish a baseline for performance and any related issues.

-   Update testing and performance benchmarks to make use of `inspection-testing` to ensure fusion is not broken in
    subsequent UTF-8 related changes.

An MVP should completely preserve standard user-facing API, and not
break fusion. Performance should not significantly diverge from the
existing UTF-16 text package. There will be an expected change to the
exposed Text internals, in which case, breakage should be assessed by
circulating a git commit reference to a release candidate as soon as
possible. This candidate should be shared publicly and loudly.

**Implementation:**

-   TBD: There is a straightforward implementation, but this one is left up to Andrew for comment.

**Stakeholders:**

-   Library authors will need to be made aware of changes and adjust accordingly. HF will provide a git reference to a complete MVP as
    soon as possible, and produce a migration guide.

-   In the case of GHC, Cabal, and other core infrastructure, we will work closely with these packages to help migrate and assess+fix
    breakages.

While we do not expect many authors to experience significant changes,
there will be some help that needs to be given in terms of bumping
Hackage bounds since this migration will be a major version bump. HF
will need to coordinate with the Hackage Trustees to help move along
packages that go out of date.

# Deliverables

-   text-2.0.0.0, which will provide a UTF-8 encoding for Text as a default for all versions going forward.

-   A `text-utf16` package, which is a preservation of the current UTF-16 encoded text, for backwards compatibility.

-   Updates to the Text Haddocks that reflect the UTF-8 changes

-   Announcements and updates across all Haskell channels covering the following:
    -   Significant dates and milestones

    -   Expected code impact

    -   Release candidates

    -   Delivery

-   Migration instructions and design documentation

# Outcomes

-   Addresses a recognized need and want from both Industry and the Haskell Community

-   Better UX for Haskell's text story

-   Establishes the ability for Haskell Foundation to Get Things Done that were previously blocked for 10+ years in the community.

-   Better interoperability and web story

# Risks

-   HF must minimize the cost to migrate, or people will just get mad (and rightfully so).

-   Text-icu will need a bespoke UTF-8 conversion function. In general, the Unicode story must be tracked and made sure it will not break.
    -   Recommendation: make this a high-priority deliverable when project planning

-   The old UTF-16 text package will need to be preserved, and will require a maintainer.

-   Performance expectations should be managed: UTF-8 text is \*not\* a panacea. It will not result in a 2x or even significant
    performance increase in many cases. In fact, performance may regress in some programs.
    -   Recommendation: we must set expectations early and often throughout the implementation process. We expect there to be
        improvements to performance \*on the margins\*.

    -   Note: We have made this argument, and the appetite for change did not shift, but it is still important to track.
