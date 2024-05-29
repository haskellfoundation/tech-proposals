# Community Project Cabal Exact Printer


## Abstract

The Exact Printer project aims to develop a precise parsing and printing tool for .cabal files in Haskell, 
inspired by ghc-exactprint. 
This tool will enable byte-for-byte bidirectional parsing and printing, 
enhancing the functionality and usability of the Cabal package manager. 
The project addresses the need for accurate and efficient management of package descriptions, 
a crucial aspect for Haskell developers.

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

_This section should describe the problem that the proposal intends to solve and how solving the problem will benefit the Haskell community.
It should also enumerate the requirements against which a solution should be evaluated._


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
However, since all these programs implement this functionality,
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

I think prior art would be tools such as 

cabalfmt, hpack and autopack

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

### Partials

+ modification to the `GenericPackageDescription`.
  covering every conceivable modification would be tough.

### Not included

+ Any warnings during parsing won't be included (low value add)
+ Any integration in tools such as `cabal format` or `cabal gen-bounds`.
  These tasks are relatively easy in comparison, however if we include this as an unused well tested library
  function, it'll be easier to release.

## Timeline

_When will the project be completed?
What are the intermediate steps and intermediate concrete deliverables for the community?_

I think the overall work is roughly 2 and a half week of fulltime work.
However this maybe executed over several months.
I'd expect the project to be completed by April 2024.

I've a free week in decemeber for example, but then it'll be weekends and nights.

## Budget

120 hours * 120 euro per hours is 14'400 euro.

The money will be used to compensate for opportunity cost,
and allowing me, and hopefully others,
to justify taking on similar large projects in the future.

## Stakeholders

_Who stands to gain or lose from the implementation of this proposal?
Proposals should identify stakeholders so that they can be contacted for input, and a final decision should not occur without having made a good-faith effort to solicit representative feedback from important stakeholder groups._

The primary benificiaries would be HLS and cabal users.
we can:

+ improve cabal usability, gen-bounds comes to mind.
+ improve HLS and cabal interaction.
+ 

An indirect loser maybe the stack build tool.
Which misses this direct investment, 

and cabal may drastically.

## Success

_Under what conditions will the project be considered a success?_
