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
One of the oldest encodings is ASCII: it has 2⁷=128 code points,
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
all code points take the same size, you can still index characters in a constant time.
However, memory requirements are 4x comparing to ASCII, and in a world of ASCII and UCS-2
there was little appetite to embrace one more incompatible encoding.

Next option on the list is to encode some code points as 2 bytes and some others,
less lucky ones, as 4 bytes. This is UTF-16. This encoding allowed to retain a decent
backward compatibility with UCS-2, so, for instance, modern Java and JavaScript
stick to UTF-16 as default internal representation. The biggest downside is that you
no longer know beforehand, where *n*-th code point starts. One must *parse* all characters
from the very beginning to learn which ones are 2-byte long and which are 4-byte long.

But once we abandon requirement of constant indexing, even better option arises. Let's
encode first 128 characters as 1 byte, some others as 2 bytes, and the rest as 3 or 4 bytes.
This is UTF-8. The killer feature of this encoding is that it's fully backwards compatible
with ASCII. This meant that all existing ASCII documents were automatically valid UTF-8
documents as well, and that 50-years-old executables could often parse UTF-8 data without
knowing a bit about it. This property appeared so important that in a modern environment
the vast majority of data is stored and sent between processes in UTF-8 encoding.

To sum up:

* Both UTF-8 and UTF-16 support exactly the same range of characters.
* For ASCII data UTF-8 takes half the space.
* UTF-8 is vastly more popular for serialization and storage.

# Motivation

`text` is a standard Haskell library for Unicode strings. Internally it stores
Unicode code points in UTF-16, so any character takes either 2 or 4 bytes.
In a modern enviroment this is a suboptimal choice: usually data
is stored (e. g., on a disc or in DB) and tranferred between agents (e. g., via web)
in UTF-8 encoding. So `text` needs to convert (UTF-8 to UTF-16) all inputs and usually
ends up converting outputs as well (this time UTF-16 to UTF-8).

Even within Haskell ecosystem UTF-16 is rarely used
for interprocess communication or as a component of binary formats.
The very `instance Binary Text` serializes `Text` in UTF-8 encoding.

If we switch the internal representation of `Text` from UTF-16 to UTF-8,
all such conversions would be made redundant and we'll be able just check that
a `ByteString` is a valid UTF-8 (which is most often the case) and copy it into `Text`.
If in future `ByteString` switch to be
backed by unpinned memory, we'd be able to eliminate copying entirely.

`Text` is also often used in contexts, which involve mostly ASCII characters.
This often prompts developers to use `ByteString` instead of `Text` to save 2x space
in a (false) hope that their data would never happen to contain anything non-ASCII.
Backing `Text` by UTF-8 removes this source of inefficiency, reduces memory consumption
up to 2x and promote more convergence to use `Text` for all stringy things
(as opposed to binary data).

Modern computer science research is focused on developing faster algorithms
for UTF-8 data, e. g., an ultra-fast JSON decoder
[simdjson](https://github.com/simdjson/simdjson).
There is much less work on (and demand for) UTF-16 algorithms.
Switching `text` to UTF-8 will open us a way to accomodate and benefit
from future developments in rapid text processing.

The importance of UTF-16 to UTF-8 transition was recognised long ago, and at least
two attempts have been made:
[in 2011](https://github.com/jaspervdj/text/tree/utf8) and five years later
[in 2016](https://github.com/text-utf8/text-utf8). Unfortunately, they did not get
merged into main `text` package. Today, five more years later it seems suitable
to make another attempt.

# Goals

-   Ensure that the new design of `text` and its migration strategy accommodates the needs of community members and industry.

-   Provide an implementation, migration, and delivery plan for changing the default encoding of `text` from UTF-16 to UTF-8.

-   Ensure stakeholders (e.g. GHC, Cabal, Stack, boot libs) have ample time to migrate and address any bugs.

-   Performance satisfies targets listed below in "Performance impact" section.

-   Compatibility story satisfies targets listed below in "Compatibility issues" section.

# People

-   Performers:

    -   Leader: Andrew Lelechenko (Bodigrim)

    -   Support: Emily Pillmore (emilypi)

-   Reviewers:

    -   The text maintainers

        -   Xia Li-Yao (lysxia)
        -   Callan McGill (boarders)
        -   Travis Athougies (tathougies)
        -   Matt Parsons (parsonsmatt)
        -   Emily Pillmore (emilypi)
        -   Dan Cartwright (chessai)

-   Stakeholders:

    -   Edward Kmett: has been vocal about his use of `text-icu` and requires it not be broken.

    -   Ben Gamari: integration with GHC.

    -   The Cabal maintainers (fgaz, emilypi, mikolaj): integration with Cabal.

Progress will be reported on a weekly basis to the HF Technical Agenda
Track, with Emily as support for Andrew.

# Timeline

We expect to finish the bulk of implementation in 3 months,
by the beginning of September, so that GHC could bump its `text`
submodule before GHC 9.4 feature freeze. Following months leading
to GHC release will provide us time to integrate with
stakeholders and diagnose any potential issues with the migration.
The project ends by Christmas 2021.

**Prior art**

The oldest attempt to use UTF-8 in `text` is
[jaspervdj's GSoC](https://github.com/jaspervdj/text/tree/utf8) back in 2011.
The final results can be found
[here](https://jaspervdj.be/posts/2011-08-19-text-utf8-the-aftermath.html).
Significant parts of Jasper's work got merged into `text`, including
comprehensive benchmark suite and many minor improvements. However,
70-commits-long UTF-8 branch itself was left unmerged and now, after 10 years,
is almost 800 commits past `master` branch.

Next comes [`text-utf8`](https://github.com/text-utf8/text-utf8) package,
which forked from `text` in 2016. Its authors rebased parts of Jasper's work
in a bulk commit and continued from this point onwards,
accumulating \~100 commits atop of it.
The work came to a halt in 2018 and at the moment is \~200 commits behind `master` branch.

It would be extremely challenging to rebase this work on top of current `text`,
audit and verify decade-old changes, then fix remaining
issues and pass a review. As one can imagine, reviewers are not usually quite happy
to review 300 commits of vague provenance. Moreover, we discovered that benchmarks
regressed severely in `text-utf8` and that fusion is broken on several occasions.
It's unclear where exactly the problem lurks there. Finally, we'd like to explore different
approaches to tackle potential performance issues.

We decided that the safest bet is to reimplement UTF-8 transition from the scratch,
paying close attention to tests and benchmarks step by step. This way we'll be able
to gain enough confidence and understanding of the nature of changes, and provide
reviewers with a clean sequence of commits, facilitating timely merge.

Talking about developments in a wider ecosystem, one must mention
`text-short` package, which provides a data structure, similar in characteristics
to `ShortByteString`, but interpreted as a UTF-8 encoded data. It was argued that
this type is worth inclusion into main `text` package to mirror `ShortByteString`,
exposed from `bytestring`. While such acquisition is out of scope for this project,
it will be easier to do so when `text` package itself switches to UTF-8, opening
possibilities for even better String story in Haskell.

**Compatibility issues**

`text` is a very old package, deeply ingrained in Haskell ecosystem.
A change of internal representation is necessarily a breaking change.
Our strategy to tackle compatibility issues is guided by a desire
to finish this project in a time-bound fashion with realistic expectations
about available resources.

Current `text` HEAD supports GHCs back to GHC 8.0.
At the moment we do not foresee any blockers
to keep compatibility with GHC 8.0 after UTF-8 transition,
and we plan to stick to it even if it causes some overhead in CPP.
However, if we discover that supporting old GHCs causes a significant overhead
(e. g., a dedicated non-trivial code path, emulating missing primop
or working around a bug), we may decide to shrink the compatibility window.
Such decision would not to be taken lightly, but we believe that getting
things done for the bright future should not be hindered by old unsupported luggage.

One suggestion to improve compatibility story was to keep both UTF-16 and UTF-8
implementations in `text` and switch between them via Cabal flag. It seems,
however, that such strategy will put an undue, indefinitely long burden
on `text` maintainers, and brings little benefits to downstream packages, because
they cannot detect build flags of `text` (and thus cannot rely on its internals at all).

Instead we mark a new, UTF-8 release as `text-2.0`, and put a call for volunteers
to maintain a legacy UTF-16 package. Depending on a demand, this could be done either
as a continuation of `text-1.X` series, or as a separate `text-utf16` package. We'll
facilitate such community project and will work with Hackage Trustees and Stackage
Curators to ensure timely transition of ecosystem.

With regards to API compatibility, we intend to keep signatures of non-`Internal`
modules unchanged, except `Word16` replaced by `Word8` where appropriate.
Such promise unfortunately cannot be made for `Internal` modules,
due to their nature: even while we'll strive to keep as much untouched as possible,
the semantics of internal functions is due to change drastically. This kind of breakage
should not come as a big surprise, because `Internal` modules have a disclaimer about
unstable API.

There are two places where `text` leaks details of internal representation.
First of them is `Data.Text.Array`, which provides an access to an underlying bytearray.
Not only its API is to change from `Word16` to `Word8`, but also the semantics
of array switches from UTF-16 to UTF-8. This will cause breakage of several packages
such as `unicode-transforms` and `unicode-collation`. We intend to communicate with
respective maintainers as early as possible to help with transition.

Another one is `Data.Text.Foreign`, which is mostly used by `text-icu` library,
which binds to `libicu` for certain Unicode manipulations. `libicu` provides
helpers to convert C strings
[from UTF8 to UTF16](https://unicode-org.github.io/icu/userguide/strings/utf-8.html).
It is up to `text-icu` maintainers to modify their bindings. We intend
to reach to them as soon as we have an MVP.

Since fixing downstream compatibility issues is up to external counterparties,
most of which are unpaid volunteers, we cannot expect them to do it in a limited
time frame. We are devoted to having a smooth migration story and will provide
as much guidance as possible, but to keep our targets time-bound we cannot tie the success
of this project to actions of third parties. We will not block this project
because of unmigrated packages downstream.

To sum up, we plan to:

* Keep `text` compatible with GHCs back to 8.0, unless it puts an undue cost (more than 50 lines of code per major release).
* Keep signatures of non-`Internal` modules compatible modulo `Word16`/`Word8` change.
* Provide migration guidance to clients of `Data.Text.{Array,Foreign}`.
* Facilitate a community project to keep UTF16-based legacy fork alive, if there is such demand.

**Performance impact**

A common misunderstanding is that switching to UTF-8 makes everything twice smaller and
twice faster. That's not quite so.

While UTF-8 encoded English text is twice smaller than UTF-16,
this is not exactly true even for other Latin-based languages, which frequently
use vowels with diacritics. For non-Latin scripts (Russian, Hebrew, Greek)
the difference between UTF-8 and UTF-16 is almost negligible: one saves on
spaces and punctuation, but letters still take two bytes. On a bright side, programs rarely
transfer sheer walls of text, and for a typical markup language (JSON, HTML, XML),
even if payload is non-ASCII, savings from UTF-8 easily reach \~30%.

As a Haskell value, `Text` involves a significant constant overhead: there is
a constructor tag, then offset and length, plus bytearray header and length.
Altogether 5 machine word = 40 bytes. So for short texts, even if they are ASCII only,
difference in memory allocations is not very pronounced.

Further, traversing UTF-8 text is not necessarily faster than UTF-16. Both are
variable length encodings, so indexing a certain element requires parsing everything
in front. But in UTF-16 there are only two options: a code point
takes either 2 or 4 bytes, and the vast majority of frequently used characters are
2-byte long. So traversing UTF-16 keeps branch prediction happy. Now with UTF-8
we have all four options: a code point can take from 1 to 4 bytes, and most non-English
texts constantly alternate between 1-byte (e. g., spaces) and 2-byte characters.
Having more branches and, more importantly, bad branch prediction is a serious penalty.
This to a certain degree is mitigated by better cache locality.

Existing `text` benchmarks are arguably favoring UTF-16 encoding: most of them are huge
walls of Russian, Greek, Korean, etc. texts without any markup. So encoding them in UTF-8
does not save any space, but we have to pay extra for more elaborate encoding. Our goal
here is nevertheless to stay roughly on par with existing implementation.

Benchmarks for `decodeUtf8` / `encodeUtf8` should improve significantly by virtue
of avoiding conversion between UTF-8 and UTF-16 conversion.
Fast validation of UTF-8 is not a trivial task, but we intend to employ
[`simdjson::validate_utf8`](https://arxiv.org/pdf/2010.03090.pdf) for this task.

Another important aspect of `text` performance is fusion. We are finalising
an `inspection-testing`-based [test suite](https://github.com/haskell/text/pull/337) to check that
pipelines, which used to fuse before, are fusing after UTF-8 transition as well.
Fusion is incredibly fragile matter: for example, of 100 tests, which fuse in GHC 8.10.4,
40 do not fuse in GHC 9.0.1, 30 do not fuse in GHC 8.4.4, etc. In such environment we cannot
bet on retaining all fusion capabilities, but we aim to thoroughly investigate
and explain all regressions.

We expect that switching to UTF-8 will be beneficial for client of `text`, both
libraries and applications. They'll be able to save memory for storage,
save time on encoding/decoding inputs and outputs, use state-of-the-art text algorithms,
developed for UTF-8. Parsers often benefit from UTF-8 encoding, because if a grammar
does not have specific rules for non-ASCII characters (which is most often the case),
parser can operate on a `ByteArray` without bothering about multibyte encodings at all.

We will seek clients' feedback as early as possible, and will act on it, if it arrives
before the end of the project. However, since our clients are external actors, often
unpaid volunteers, we cannot expect them to provide feedback by the given date.
Thus to keep targets of this project time-bound, we cannot include a goal
of waiting for an approval of indefinite number of parties for indefinitely long.
Such goal or sentiment, in our opinion, made a significant contribution into failure
of two previous attempts.

To sum up:

* `decodeUtf8` and `encodeUtf8` become at least 2x faster.
* Geometric mean of existing benchmarks (which favor UTF-16) decreases.
* Fusion (as per our test suite) does not regress beyond at most several cases.

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

-   `text-2.0.0.0`, which will provide a UTF-8 encoding for Text as a default for all versions going forward.

-   Updates to the `text` Haddocks that reflect the UTF-8 changes.

-   Announcements and updates across all Haskell channels covering the following:
    -   Significant dates and milestones.

    -   Release candidates.

    -   Migration guides.

    -   Performance impact analysis.

    -   Delivery.

    -   Design documentation.

# Outcomes

-   Addresses a recognized need and want from both Industry and the Haskell Community

-   Better UX for Haskell's text story

-   Establishes the ability for Haskell Foundation to Get Things Done that were previously blocked for 10+ years in the community.

-   Better interoperability and web story

# Risks

-   HF must minimize the cost to migrate, or people will just get mad (and rightfully so).

-   `text-icu` will need a bespoke UTF-8 conversion function. In general, the Unicode story must be tracked and made sure it will not break.
    -   Recommendation: make this a high-priority deliverable when project planning

-   The old UTF-16 text package will need to be preserved, and will require a maintainer.

-   Performance expectations should be managed: UTF-8 text is \*not\* a panacea. It will not result in a 2x or even significant
    performance increase in many cases. In fact, performance may regress in some programs.
    -   Recommendation: we must set expectations early and often throughout the implementation process. We expect there to be
        improvements to performance \*on the margins\*.

    -   Note: We have made this argument, and the appetite for change did not shift, but it is still important to track.
