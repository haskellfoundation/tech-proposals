# RFC - Specify GHC's JSON Diagnostic Dump

## Abstract

GHC is currently undergoing a long term project to [move to a more structured error representation](https://gitlab.haskell.org/ghc/ghc/-/issues/18516). 
In harmony with this motion, there is a [desire for a JSON dump of GHC's diagnostic messages](https://gitlab.haskell.org/ghc/ghc/-/issues/19278).
Such an implementation would facilitate easier consumption for downstream consumers of GHC's diagnostic messages (e.g. IDE developers). 
To best enforce a standardized JSON output, a [JSON schema](https://json-schema.org/) can be used.

The purpose of this RFC is to open up the discussion to the broader Haskell community about:

- the desire for such a feature
- the requested fields present in such a JSON output
- opinions on the draft JSON schema (or the usage of JSON schemas at all)
- any other general feedback 

## Background

In the prior implementation of GHC, errors were passed around as `SDoc`s (for Structured Documents).
These `SDoc`s are simply a text encoding of the errors with some formatting features, such as nice alignment.
By transforming the errors into `SDoc`s at the moment of creation, a lot of type-level information is lost,
causing a headache for GHC developers and GHC API users.
To improve on this problem, the [journey to represent errors as structured values has begun](https://gitlab.haskell.org/ghc/ghc/-/wikis/Errors-as-(structured)-values ). 
In summary, this means to pass errors around as a specific type which contains the relevant information separated into different fields,
rather than combining it all into one single `SDoc`. 

Another change that has been made is the introduction of [error codes](https://gitlab.haskell.org/ghc/ghc/-/issues/21684) to uniquely identify each possible error output from GHC. 

These features together serve to make error consumption a breeze for GHC API users, but what about those that would like to consume errors without utilizing GHC's API?

## Problem Statement

The fundamental issue is that the structured errors are only presented to users who leverage GHC as a library in their application. If instead, you desire to consume error messages without tapping into the internals of GHC, your best bet is to parse the plain text errors printed to stdout by GHC. This poses two important problems. 

The first is that this is an _unnecessary_ pain. The errors are now passed as structured values internally, but the users are given a fairly unstructured value to parse themselves, leading to the requirement of always having to write a parser whenever consumption is desired.

The second problem is that this plain text output is subject to change. Messages are often changed to make the content clearer, but each change results bubbles into a new requirement for the hand-written plain-text parser of error messages. This problem is mitigated a bit by the above-mentioned new inclusion of error codes, but even these error codes must be parsed from a non-standard structured output. 

The requirements to solve this problem are to enable the consumption of GHC's diagnostic message in a structured representation that closely matches the internal representation without requiring users to utilize the internals of GHC. The most reasonable solution is to use a compiler flag to dump the structured messages into a versioned JSON output, which is described by a JSON Schema, to standardize the expected output. The hope is that the community can reach a general consensus on the requirements imposed by such a JSON schema. While it will not be too costly to change the JSON schema as new requirements come up (simply increment the schema version), the ideal situation is to present here an opportunity to provide input on the current schema and receive constructive criticism. The conversation is also open on whether or not a JSON schema is really required.

## Prior Art and Related Efforts

There is a currently existing GHC flag, `-ddump-json`, which is under-specified and thus cannot be depended on. Here is an example (prettified) output of the current flag:
```json
{
  "span": null,
  "doc": "[1 of 1] Compiling ShouldFail       ( testsuite/tests/typecheck/should_fail/T2414.hs, testsuite/tests/typecheck/should_fail/T2414.o )",
  "messageClass": "MCOutput"
}
{
  "span": {
    "file": "testsuite/tests/typecheck/should_fail/T2414.hs",
    "startLine": 9,
    "startCol": 13,
    "endLine": 9,
    "endCol": 17
  },
  "doc": "• Couldn't match type ‘b0’ with ‘(Bool, b0)’\n  Expected: b0 -> Maybe (Bool, b0)\n    Actual: b0 -> Maybe b0\n• In the first argument of ‘unfoldr’, namely ‘Just’\n  In the expression: unfoldr Just\n  In an equation for ‘f’: f = unfoldr Just",
  "messageClass": "MCDiagnostic SevError ErrorWithoutFlag Just GHC-27958"
}
```

Just on the surface, the first object is of a different kind than the second, they are not wrapped in array, and the second object lacks crucial information like error codes. And what is messageClass? These fields can and should be standardized to most authentically express the diagnostics in a way that makes them understandable and consumable.

The extra `d` in `-ddump-json` indicates that this is a developer flag, but our aim is to make this a first class feature.

The aim is to create a JSON interface for GHC diagnostics, standardized by a JSON schema. Then, the `-ddump-flag` flag will be replaced by the first-class `-dump-flag` flag, and subsequently, whatever implementation changes are necessary to make the new flag conform to the new specification will be performed.

The past attempt did not succeed simply due to lack of effort and resources devoted to it, as well as a lack of structured error representations. Now that structured errors are pervasive within GHC, such a JSON dump is both feasible and practical.

## Technical Content

The most important component of the technical content is the JSON schema itself. The JSON schema is not reproduced here, as it is quite long. Instead, it can be found in the directory of the same name, with title `schema.json`. To demonstrate the appropriateness of the proposed JSON schema, the internal error representation is produced below. Comments removed for brevity.
```haskell
-- compiler/GHC/Types/Error.hs
data MsgEnvelope e = MsgEnvelope
   { errMsgSpan        :: SrcSpan
   , errMsgContext     :: NamePprCtx
   , errMsgDiagnostic  :: e
   , errMsgSeverity    :: Severity
   } deriving (Functor, Foldable, Traversable) 
-- compiler/GHC/Types/Error.hs
class (HasDefaultDiagnosticOpts (DiagnosticOpts a)) => Diagnostic a where
  type DiagnosticOpts a
  diagnosticMessage :: DiagnosticOpts a -> a -> DecoratedSDoc
  diagnosticReason  :: a -> DiagnosticReason

  -- | Extract any hints a user might use to repair their
  -- code to avoid this diagnostic.
  diagnosticHints   :: a -> [GhcHint]
  diagnosticCode    :: a -> Maybe DiagnosticCode
```
All of the above is explained in the [wiki](https://gitlab.haskell.org/ghc/ghc/-/wikis/Errors-as-(structured)-values).

The output of the JSON dump would consist of a list of errors, as well as a top level version. Each error contains the following fields:
 - `"span"` corresponding to the above `SrcSpan`
   - This contains sub-fields, specified in the schema, which contains line numbers, etc.
 - `"severity"` corresponding to the above `Severity`
 - `"code"` corresponding to the change in this [proposal](https://github.com/haskellfoundation/tech-proposals/blob/main/proposals/accepted/024-error-messages.md) and the above `DiagnosticCode` in typeclass `Diagnostic`
 - `"hints"` corresponding to the above `[GhcHint]`
   - This is perhaps one that remains a bit of a question mark. To what degree should this be specified? There are quite a few possible constructors, and so it may be overly tedious to specify every single one as a possible output in the JSON schema. On the other hand, it may be insufficient to allow arbitrary JSON to be put forward. Maybe a reasonable compromise is to give back the name of the constructor followed by a list of the constructors arguments. 
 - `"message"` corresponding to `DecoratedSDoc` of `diagnosticMessage`
 - `"reason"` corresponding to `diagnosticReason`
 
 The top level version is crucial for indicating to downstream consumers which JSON schema they must comply with in the case that the schema is updated in the future (which is quite likely).
 
For demonstrative purposes, here is an example valid instance of the schema.
```json
{
  "version": "1.0.0",
  "diagnostics": [
    {
      "span": {
        "file": "typecheck/should_fail/T2414.hs",
        "startLine": 9,
        "startCol": 13,
        "endLine": 9,
        "endCol": 17
      },
      "severity": "Error",
      "code": 27958,
      "message": "    • Couldn't match type ‘b0’ with ‘(Bool, b0)’ \n Expected: b0 -> Maybe (Bool, b0) \nActual: b0 -> Maybe b0 \n• In the first argument of ‘unfoldr’, namely ‘Just’ \nIn the expression: unfoldr Just \nIn an equation for ‘f’: f = unfoldr Just"
    }
  ]
}
```

The schema itself will be brought into version control of the GHC repo and tests will be added to ensure that GHC-dump complies with the JSON schema. This will ensure that changes to the diagnostics will result in appropriate changes to the schema. In addition, a wiki page will be created to fill in the gaps for anyone needing to understand the project. Otherwise, I will be available to perform maintenance. 

One of the major benefits of utilizing a JSON schema is that the expected JSON payload can be well-defined for consumers of the messages. This has massive benefits, as consumers can presume the structure of the output without having to analyze the contents for the presence or absence of particular bits of data. However, one drawback may be the over specification of the output. There may be some opportunities in which flexibility is a benefit for the output. However, this schema can be adapted as further feedback rolls in (with incremented version), making the good net outweigh the bad.

In addition to adding a `-dump-json` flag, it may also prove useful to provide a `-dump-json-schema` flag which simply produces the relevant JSON schema for that particular version of GHC. This I leave open for discussion. Provided that the schema is in an easy to find location, it may be overkill.

The schema evolution process is currently undetermined, though I imagine that due to the infrequency with which the schema will need to be changed, it can be handled on a case-by-base basis. Though keeping a running list of all relevant stakeholders that may need to be informed could be a good idea. 

## Stakeholders

Stakeholders consist of anyone that desires to consume a JSON output of GHC diagnostics without leveraging the GHC API. Some possible stakeholders are listed explicitly [here](https://gitlab.haskell.org/ghc/ghc/-/issues/19278#note_503994). These are:
 - Chris Smith of https://code.world
 - Joseph Sumabat

Joseph Sumabat replied to my emails and his input has been incorporated into the included schema. I believe the Haskell community would benefit as a whole from improved tooling, and this effort will help in this regard by making an easy to consume representation of GHC's diagnostics.

## Success

I expect the implementation to take no more than 2 or so months. The project will be considered a success when an appropriate JSON dump flag is shipped with GHC which serves the purpose of providing a JSON structured representation of error messages which complies with a JSON schema in version control.
