## Abstract

This proposal seeks technical coordination from the Haskell Fondation
for improving integration between IDEs and testing libraries in Haskell.

This work will involve two major implementation areas: testing library
integration, and IDE integration. The author will aim to implement both
of these features, though help is certainly welcome :).

This undertaking requires buy-in and feedback from the maintainers of
Haskell's testing frameworks, since we would like to have one interface
that satisfies everyone's needs.

 ## Background

IDE integration for test suites is a feature that is expected in most
if not all major programming languages. First class support exists for
[JUnit with multiple IDEs](https://junit.org/junit5/docs/current/user-guide/#running-tests-ide),
[Python with VSC](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwiRyt_VwcD7AhUXIUQIHQe1CjUQFnoECBIQAQ&url=https%3A%2F%2Fcode.visualstudio.com%2Fdocs%2Fpython%2Ftesting&usg=AOvVaw0U1x8oEhgUTX1GeoPJqVqo),
[.NET with VSC](https://marketplace.visualstudio.com/items?itemName=formulahendry.dotnet-test-explorer)
and many others.

Often these IDEs will supply inline hints for "Run this test" and
"Re-run failed tests" as well as others. For users of mainstream
languages, such features are no longer features, but instead
expectations from a mature language.

Haskell has a mature and well-maintained testing ecosystem, including
the libraries: `tasty`, `HUnit`, and `hspec`.  While these libraries
themselves are mature, there is currently minimal
integration between Haskell's testing libraries and HLS, as well as
the primary IDEs used for Haskell (VSC;Emacs;Neovim).

Attempts at integration between testing frameworks have focused on
regex searches, with support for hspec:
 1. [neo-test](https://github.com/mrcjkb/neotest-haskell)
 2. [vim-test](https://github.com/vim-test/vim-test)

While these are valuable contributions to the ecosystem, their choice
of regex for the collection of the user's test tree, and support for only
specific frameworks are symptomatic of a lack of ecosystem coordination.

## Motivation

With the implementation of this proposal, the Haskell community will
have gained a valuable IDE tool which will improve the user experience
of Haskellers as well as make Haskellers more productive.

Support for IDE integration of testing libraries will provide the following
benefits:
 1. Ability for IDEs with Haskell support to provide inline hints for "Run
 Tests" style commands.
 2. Ability for IDEs with Haskell support and support for a test explorer
 view to display the user's test tree and run individual tests.
 3. A unified framework for testing libraries to provide integration with
 IDEs. When a new testing framework is developed, they will be able to
 provide the results in a Haskell datatype via serialization, which
 avoids framework-specific test tree discovery logic.
 4. Potential to share test result rendering logic between different testing
 frameworks, whereas currently each testing framework has their own test result
 display logic.

## Goals

1. Create a new library, `hs-test`, with common datatypes for testing framework
 test trees, and test results.
2. Generate test trees and test results that can be serialized into JSON from
Haskell's major testing frameworks, starting with `tasty`.
3. Integrate test tree output output with VSC's Haskell extension, via VSC's [Testing API](https://code.visualstudio.com/api/extension-guides/testing).
4. Stretch goal: support IDEs other than VSC.
5. Stretch goal: support test results caching, and features such as "Rerun only failed tests".

## Technical Proposal

The `hs-test` library will contain the following data declarations:

```haskell
-- | Datatype representing all the tests that exist in a test
-- suite that could be run.
--
-- `SrcLocs` are used to indicate where inline hints should be displayed
-- in an IDE.
data TestTree =
  TestGroup {name :: Text, subTrees :: [TestTree], location :: SrcLoc}
  | TestCase {name :: Text, location :: SrcLoc}

data RunTestsRequest =
  RunTestsRequest {path :: NonEmpty Text}

-- | A lazy list in order to support arbitrary execution
-- order and streaming between the test runner and IDE
data TestResults = TestResults [TestCaseResult]

data TestCaseResult = TestCaseResult {
    -- | Represents the path to get to this test case from the
    -- TestTree, where each element represents a node in the original
    -- TestTree.
    path :: NonEmpty Text,
    result :: TestResult,
    timeTaken :: Maybe Double
  }

data TestResult = Success | Skipped | Failure FailureReason

newtype FailureReason = FailureReason Text
```

It will also contain the logic to encode this datatype
to JSON, for consumption by IDEs.

------------

For testing frameworks that want to support IDE integration,
they should provide support for the following commands:

```bash
./test-suite test-tree --output="FILE_NAME"
./test-suite run-tests --input="FILE" --output="FILE"

# Sample usages:
> cabal test test-suite-name \
      \--test-options='test-tree --output="output.tmp.json"'
> cabal test test-suite-name \
      \--test-options='run-tests --input="input.tmp.json" --output="output.tmp.json"'
```

The `test-tree` command outputs a JSON-serialized `TestTree` to the given
`output` file.

The `run-tests` command reads a JSON-serialized `RunTestsRequest` from
`input`, and, after executing tests, writes the result to the `output` file.

`hs-test` will provide helper functions for these serialization tasks.

-------------

For IDEs, they will utilize these commands to:
 -  find the test tree
 -  execute a test run request, and display the results to the user.

One problem that could occur is that having the IDE run `cabal test` to
reindex the `TestTree` is too slow and incurs too large a latency for
the user. However, this problem is no worse than the current situation,
in which the user has to call `cabal test` anyway in order to run the tests.

Furthermore, IDEs often having access to stale artifacts for code that used
to compile but no longer compiles, therefore, the IDE could run the `test-tree`
command while still displaying a mostly-correct test tree and inline hints
in the user's IDE. Once the call to `test-tree` has completed, the new test
tree can be displayed to the user.

## Alternative Approaches

### A typeclass driven approach with HLS integration

Described in the following [discourse comment](https://discourse.haskell.org/t/hf-coordination-for-test-framework-ide-integration/5221/2?u=santiweight)

In summary, take each function signature and detect whether said function
is runnable as a test based on whether its type is a member of the
proposed typeclass `IsTest`. Testing frameworks will each provide their
own `IsTest` instances for their common testing types.

I have chosen the option in this proposal for the following reasons:
 1. Such an approach would run Haskell code in GHCi, which is slower, and
 would therefore scale poorly for production codebases.
 2. Such an approach requires integration with GHC's API, which the author
 is unfamiliar with and therefore cannot be confident in their ability to
 maintain the integration library.
 3. The author cannot see how to acquire the test tree that is common to
 IDE test integration, since it requires inspecting the typechecking artifacts
 of all files in the test suite.
 4. Sometimes testing frameworks will provide conflicting `IsTest` instances for the
 the same type, which is a problem when using multiple testing frameworks in the
 same codebase.

The typeclass proposal seems incredibly useful however for the typical `Run Program`
command now provided, or as an additional tool for users. It is certainly a good
idea for a future project.

### Regex-based test discovery

While regex discovery is a good and quick solution implementation wise, it does
not scale to many real-world usecases. For example, many codebases will have helpers
such as:

```haskell
testDecodeJsonFromFile fileName = testCase fileName $ readJsonFromFile fileName
```

Since the fileName value is not known until runtime, regex-based approaches miss
such cases, which is a non-starter for an ecosystem-wide tool.

The proposal put forth by the author will also have to handle this use-case, by
providing support in testing frameworks for users to define their own combinators
that declare their location on use site. But this problem is a purely technical
one with many solutions, such as the following pseudo-code:

```haskell
testDecodeJsonFromFile :: HasCallStack => TestTree
testDecodeJsonFromFile = Tasty.customTestCase callstack fileName $ readJsonFromFile fileName
```

## Future Work

This proposal is purposefully minimal in scope, for a number
of reasons:
 1. In order for testing frameworks to buy in, we will need a
 minimally-controversial interface.
 2. Support for more features should be done incrementally, since
 the design space here is rather large, and we will not know users'
 needs until after integration is alright out in the real world.

Here are some features that have been neglected in this proposal:
 1. Distinction between "error" and "failure" when a test fails.
 2. Support for multiple filters when running a test.
 3. Support for options when running a test, such as: `--accept` when
 running golden tests, `-jN` to run tests in parallel, and options
 related to property-based tests.
 4. Support for localized options when running a test.
