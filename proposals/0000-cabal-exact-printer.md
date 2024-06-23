# Community Project Cabal Exact Printer


## Abstract

The Exact Printer project aims to develop a precise parsing and printing tool for .cabal files in Haskell, 
inspired by ghc-exactprint. 
This tool will enable byte-for-byte bidirectional parsing and printing.
Which will allow both cabal and other tools to modify cabal files without
mangling the format, structure or comments of users.

The overal goal would be to roundtrip 99% of all hackage packages.


## Background

Cabal is a build tool for haskell.
It directs GHC and deals with "packages".
Which are collections of haskell modules.
cabal allows you to depend on libraries,
publish libraries and manage build flags.

In essence if you want to do something non-trivial
with haskell you want to use cabal.

.cabal files are fundamental in Haskell programming, used to describe package structures. 
Current methods of parsing and printing these files lack the precision needed 
for certain tasks. 
Such as generating bounds for dependencies,
formatting,
or adding missing modules from the manifest.

The project is inspired by the capabilities of [ghc-exactprint](https://github.com/alanz/ghc-exactprint) and aims to bring similar 
[functionalities](https://gitlab.haskell.org/ghc/ghc/-/wikis/api-annotations#in-tree-exact-printing-annotations) to Cabal.
It defines exact printing as follows:
> Taking an abstract syntax tree (AST) and converting it into a string that looks like what the user originally wrote
> is called exact-printing. An exact-printed program includes the original spacing and all comments.

## Problem Statement

This [issue](https://github.com/haskell/cabal/issues/7544) is stracked on the cabal bug tracker.

Currently if you build a project with an extra module not listed in your cabal file,
ghc emits a warning:
```
<no location info>: error: [-Wmissing-home-modules, -Werror=missing-home-modules]
    These modules are needed for compilation but not listed in your .cabal file's other-modules: 
        X
```

You'd say, why doesn't cabal just add this module to the cabal file?
Well, it can't.
Cabal is currently only able to parse Cabal files, 
and print them back out in a mangled form.
There are other programs providing module detection, 
but nothing is integrated in cabal itself.

This problem has been solved by the community, several times outside of cabal.
For example:
<ul>
 <li> [hpack](https://github.com/sol/hpack),              </li>
 <li> [autopack](https://github.com/kowainik/autopack)    </li>
 <li> [cabal-fmt](https://github.com/phadej/cabal-fmt)    </li>
 <li> [gild](https://taylor.fausak.me/2024/02/17/gild/)   </li>
</ul>

Of course many of these projects do more then just module expension.
hpack provides a completly different cabal file layout for exampe,
cabal-fmt and gild are formatters for cabal file.
Only auto-pack just does this one feature.
However, since all these programs implement this functionality
in their own distinct way,
there is clearly demand for it.

There are more issues then just module expension however.
For example [cabal gen-bounds](https://github.com/haskell/cabal/issues/7304) could modify a cabal file in place,
with [cabal edit](https://github.com/haskell/cabal/issues/7337) we could add a dependency via cli, 
[cabal init](https://github.com/haskell/cabal/issues/6187) could be simplified.

The current implementation of writing, cabal format, has the following issues:

1. Deletes all comments
2. merge any `common` stanza into wherever it was imported. https://github.com/haskell/cabal/issues/5734
3. changes line ordering.
4. changes spacing (although perhaps to be expected from a formatter)

A similar problem occurs when HLS want's to do any modification to a cabal
file during development.
For example if a module was added or renamed, or if a (hidden) library is required.
Or perhaps some function used in a known library via hoogle for example.
HLS has no clue what to do,
because even if it links against the cabal library,
there is no function to modify a generic cabal representation and print a cabal file that keeps
it similar to the users'.

The goal is to make non invasive changes.
This tech proposal therefore aims to address all these issues.

## Prior Art and Related Efforts

Previous attempts to address this problem have been fragmented, and no comprehensive solution has been developed. This project builds upon the ideas discussed in various issues over the past six years (e.g., Haskell/cabal issues #3614, #6621, #6187, #4965). The project will synthesize these discussions into a cohesive solution.

I think prior art would be tools such as cabalfmt, hpack and autopack

Previous attempts were [abandoned](https://github.com/haskell/cabal/pull/7626).
Or they revolved around creating a seperate AST[^ast], which was against maintainer recommendation, 
and then [abandoned](https://github.com/haskell/cabal/pull/9385).

A related effort is to build combinators that allow modifyng the `Field` type directly.
This would depracate the GenericPackage structure and make an alternative structure
available.
A proof of concept was developed during zurich hack
https://discourse.haskell.org/t/pre-proposal-cabal-exact-print/9582/9?u=jappie

I suppose the idea is to completly replace `GenericPackageDescription` with
this `Field` type.
Which is a significant effort,
however this work can be amend the exact print effort, 
because better modification of cabal files would be appreciated.
Exact printing is mostly a module that takes some input type and then
does the formatting.

Furthermore the test suite created by the exact print effort this module
describes can also be used in the related `GenericPackageDescription` to `Field` effort.

## Technical Content

This proposal want's to add a function to cabal:

```
printExact :: GenericPackageDescription -> Text
```

Which will do exact printing.
This function has the following properties:

byte for byte roundtrip of all hacakgePackage:
```
  forall (hackagePackage :: ByteString) . (printExact <$> (parseGeneric hackagePackage)) == Right hackagePackage
```

where `hackagePackage` is a cabal package found on hackage.


to support exact printing a new field is added to `GenericPackageDescritpion`:

```haskell
data GenericPackageDescription {
 ...
 , exactPrintMeta :: ExactPrintMeta
 }
```

which in turn contains various meta data we need for exact printing:
```haskell
data ExactPrintMeta = ExactPrintMeta
  { exactPositions :: Map [NameSpace] ExactPosition
  , exactComments :: Map Position Text
  }
```
It's unclear what other fields are required right now.
For example build bounds require another map 
like `Map ([NameSpace], PackageVersionConstraint) Original`,
this kind of representation allows us to retrieve the original only if it hasn't changed.
However initial inspection of the parser showed it's difficult to retrieve `PackageVersionConstraint` and `[NameSpace]` together,
because they're deeply nested within field grammars.
so perhaps an intrusive design is more easy for that,
it's unclear to me right now.
However it is clear these problems can be solved, it just takes time and effort.

However peliminary testing shows that this approach works with multiple secrion cabal files.

Pertubation of the `GenericPackageDescription` must be possible for
+ module addition/removal
+ library addition/removal

The issue for addition is that you now have to invent exact positions.
for removal, if it involves a line, you've to fix up all following lines, 
(and it has to know something was removed).

The overal goal would be to roundtrip 99% of all hackage packages.

### Partials

+ modification to the `GenericPackageDescription`.
  covering every conceivable modification would be tough.

### Not included

+ Any warnings during parsing won't be included (low value add)
+ Any integration in tools such as `cabal format` or `cabal gen-bounds`.
  These tasks are relatively easy in comparison, however if we include this as an unused well tested library
  function, it'll be easier to release.

## Timeline

I expect that after this project is approved it'd take roughly 4 months in total to complete.
This would be about 2 'man' months, however there maybe random hickups.
Most work will be done by either me or one of my employees @Riuga.

Intermediate steps are the listed tests in the linked PR,
then after all those pass we'll move onto a full hackage run,
and sift out more .

We can easily track progress via the tests being finished.

I'd expect the project to be completed by April 2024.

I've a free week in decemeber for example, but then it'll be weekends and nights.

## Budget

I think this should be around 15'000 euro to complete,
considering the size of the overal work.

The money will be used to compensate for opportunity cost,
and allowing me, and hopefully others,
to justify taking on similar large projects in the future.

## Stakeholders

The primary benificiaries would be cabal users.
we can:

+ opens up the possibility to improve cabal user experience, via inserting modules or running gen-bounds.
+ makes it easier for cabal library to deal with cabal files, such as hpack, HLS and cabal-fmt
+ makes maintaining cabal itself easier, eg cabal init could be described via GenericPackageDesription

Furthermore I've heard that the HLS project will benefit greatly from this effort,
(TODO where?)

## Success

This proposal successful once the cabal exact print branch is merged into cabal proper with the provided tests passing.
Most of hackage should be exact printable, say 99%,
which means, an existing hackage cabal file is being parsed, and then printed again resulting into the same output as input.
