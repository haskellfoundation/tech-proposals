# Community Project Cabal Exact Printer


## Abstract
The Exact Printer project aims to develop a precise parsing and printing tool for .cabal files in the cabal library.
This will enable byte-for-byte bidirectional parsing and printing.
Which will allow both cabal and other tools to modify cabal files without
mangling the format, structure or comments of users.

## Background
This is the formal application of my [blogpost](https://jappie.me/cabal-exact-printing.html).
Ironically I started on this proposal first,
but then I got cold feet and decided to write a "lower stake" informal blogpost instead.
Writing this proposal has been difficult. 
I've no idea why this is so hard for me.
I think it's partly because there is no going back after opening the proposal.
and I'm mostly just making these numbers and timelines up y'know.
I've no idea for it'll take 2 months, but it seems reasonable.
I've no idea if there is even budget for this, but asking 10k for a man month
seems reasonable as well. 
Getting this accepted and solved won't make me rich, 
but it could make me happy. 
It'll solve a pain point bringing my work life closer to epicurean atraxia.

Cabal is a build tool for haskell.
It directs GHC and deals with "packages".
Which are collections of haskell modules.
cabal allows you to depend on libraries,
publish libraries and manage build flags.
In essence if you want to do something non-trivial
with haskell you want to use cabal.

`.cabal` files are used to describe package structures. 
The current implementation for printing these files lacks the precision needed 
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
it can print them back out, but in a mangled form.

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
`cabal-fmt` and `gild` are formatters for cabal files.
Only auto-pack just does this one feature.
However, since all these programs implement this functionality
in their own distinct way,
there is clearly demand for it.

There are more issues then just module expansion however.
For example [cabal gen-bounds](https://github.com/haskell/cabal/issues/7304) could modify a cabal file in place,
with [cabal edit](https://github.com/haskell/cabal/issues/7337) we could add a dependency via cli, 
[cabal init](https://github.com/haskell/cabal/issues/6187) could be simplified.

The current implementation of printing in cabal via the `cabal format` command, has the following issues:

1. It Deletes all comments [^1]
2. It merges any `common` stanza into wherever it was imported. https://github.com/haskell/cabal/issues/5734
3. It changes line ordering.
4. It changes spacing (although perhaps to be expected from a formatter)

[^1]: Andreas helped me solve this on zurich hack.

A similar problem occurs when HLS want's to do any modification to a cabal
file during development.
For example if a module was added or renamed, or if a (hidden) library is required.
Or perhaps some function used in a known library via hoogle for example.
HLS has no clue what to do,
because even if it links against the cabal library,
there is no function to modify a generic cabal representation and print a cabal file that keeps
it similar to the users'.

The goal is to make non invasive changes to cabal files.
This tech proposal therefore aims to address all these issues.
Furthermore by bringing it directly into cabal we can enforce the round tripping
property.
This will ensure clients of the cabal library can print cabal files more
easily and have some stability guarantees.

## Prior Art and Related Efforts
TODO: describe why previous efforts didn't succeed. How is this proposal different?

This [issue](https://github.com/haskell/cabal/issues/7544) is tracked on the cabal bug tracker.
Essentially this proposal attempts to "solve" that issue.
As can be seen in the issue, there have been previous attempts,
previous attempts to address this problem have been fragmented, 
and no comprehensive solution has been finished. 
There is however a work in progress implementation of the [exact printer](https://github.com/haskell/cabal/pull/9436/).
The goal of this proposal is to buy time to finish that implementation.

The current work left to be done on this pull request is:
+ Add more comment preservation on more locations.
+ Deal with changes in generic package description.
   + We need to add tests about which changes we care about. 
     For example add a build field, add a field which causes comment overlap on x,y. 
     Delete a section, add a language flag, etc.
   + it the algorithm just relatively shifts everything if you add
     a build field for example.
     You know something got added because you can't find it in ExactPrintMeta.
+ Redo how common stanza's are handled (they're currently "merged" into sections directly, which is unrecoverable).
+ add support for comma printing,
+ add support for braces.
+ add support for conditional branches.

Previous attempts for making this directly into cabal were [abandoned](https://github.com/haskell/cabal/pull/7626).
I guess they got demotivated by the shear size of the effort,
or they revolved around creating a seperate AST[^ast], 
which was against maintainer recommendation (because it'd make the issue even bigger), 
and then [abandoned](https://github.com/haskell/cabal/pull/9385).

A related effort is to build combinators that allow modifyng the `Field` type directly.
This would depracate the `GenericPackage` structure and make an alternative structure
available.
A proof of concept was developed during zurich hack
https://discourse.haskell.org/t/pre-proposal-cabal-exact-print/9582/9?u=jappie

I suppose the idea is to completly replace `GenericPackageDescription` with
this `Field` type.
Which is a significant effort,
however this work can be added to the exact print effort, 
because better modification of cabal files would be appreciated.
Exact printing is mostly a module that takes some input type and then
does the formatting.

Unlike the `Field` effort, 
this proposal only focusess only on getting exact printing
to work with a minimal footprint.
We don't want to do any additional refactoring.
Furthermore the test suite created by the exact print effort this module
describes can also be used in the related `GenericPackageDescription` to `Field` effort.



## Technical Content
This proposal want's to add a function to the cabal library:

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

For comments some parser modifications were needed which were developed during zurich hack 2024.
This was a big uncertainty which now has been addressed.

It's unclear what other fields are required right now.
For example build bounds might require another map 
like `Map ([NameSpace], PackageVersionConstraint) Original`,
this kind of representation allows us to retrieve the original only if it hasn't changed.
However I'm confident all of this is do-able.

Peliminary testing shows that this approach works with multiple sections of cabal files,
and now also with some basic comments.
Comments likely have to be worked out further as we only made that one test pass,
but the 'tools', and more importantly, the understanding is in place.
There is a working prototype.

Change of the `GenericPackageDescription` must be possible for
+ module addition/removal
+ library addition/removal

The issue for addition is that you now have to invent exact positions.
for removal, if it involves a line, you've to fix up all following lines, 
(and it has to know something was removed).

The overal goal would be to roundtrip 99% of all hackage packages.

### Common stanzas
Currently if you run the pretty printer on a parsed generic package you will
lose all common stanzas.
The reason for this is that they're merged in the imported sections[^see-pr],
and then forgotten about.

[^see-pr]: I marked where this happened https://github.com/haskell/cabal/pull/9436/files#diff-39a353df50e7eed47b5958c6025b67b06fac735a8b5b994c1464d6fd84df745eR718

I think there are two somewhat viable approaches to dealing with common stanzas.

1. Keep the current "merging" approach. However track their original shape in 
   `ExactPrintMeta`. Then let the printer figure out based on that information
    what came from the commen stanza and what came in the original section.
    
2. Refactor `GenericPackageDescription` so that the parser no longer merges these
   sections and instead stores them as proper records.
   Then make the callsites smart enough to deal with common stanzas.
   
In this case I'd attempt approach 2 first.
It seems less fragile to me to just refactor this.
It also removes the concern of carrying duplicate information, and should
make it more obvious to the users of the cabal syntax library 
on how to manipulate common stanzas.

how this is done concretly, you go to the associated types and
add imports field:
```haskell
newtype CommonStanzaName = CommonStanzaName Text

data Library = Library
  { libName :: LibraryName
  , imports :: [CommonStanzaName]
  ...
  }
```

and to generic package description you add another map:
```haskell
data GenericPackageDescription = GenericPackageDescription
  { packageDescription :: PackageDescription
  , gpdCommonStanzas :: Map CommonStanzaName CommonStanza
  }
```

I'm not sure if this is fast enough, but
one easy and dirty way to make it all work with
backwards compatibility is to
create custom getters, (original setters are fine)

we expose a getter (which is the current record field name `condSubLibraries`):

```haskell
condSubLibraries :: GenericPackageDescription -> [( UnqualComponentName , CondTree ConfVar [Dependency] Library)]
```

which merges the common stanzas on every "get" call.
Then the actual record field will be renamed, 
and doesn't have this merging in behavior.

This way, new code will be able to access a library without common stanzas merged in,
as would be expected wthin the cabal file.
Old code, will have the same behavior as the current implementation.
And it will allow the exact printer to make a distinction between common stanzas and
whatever is in the other stanzas.

### Conditionals
TODO: see how these work - so I can think of a design, I've a suspicion but i just need to 
confirm my mental model maps to reality

### Partials

modification to the `GenericPackageDescription`.
covering every conceivable modification would be though.
However we should be able to do common operations such as adding libraries or modules.
What we want is a sort of default behavior, all subsequent mappings in the
exact printer position should be shifted if we add a line.

### Not included

+ Support for braces
+ Any warnings during parsing won't be included (low value add)
+ Any integration in tools such as `cabal format` or `cabal gen-bounds`.
  These tasks are relatively easy in comparison, however if we include this as an unused well tested library
  function, it'll be easier to release so other people can go out and test.

## Timeline
I expect that after this project is approved it'd take roughly 4 months in total to complete.
This would be about 2 'man' months, however there maybe random hickups.
Most work will be done by me or a subcontractor if I can find them.

Intermediate steps are the listed tests in the linked PR,
then after all those pass we'll move onto a full hackage run,
and sift out more .
We can track progress via the tests being finished.
I'd expect the project to be completed by 4 months after accepting this proposal.

If there are delays we'll communicate them promptly.

## Budget
I think this should be around 20'000 euro to complete,
considering the size of the overal work.
I think it's best to spread this out over subgoals, for example:

5k: finish test suite in the linked PR
5k: 60% of hackage roundtripped
10k: 99% of hackage roundtripped

The money will be used to compensate for opportunity cost,
and allowing me, and hopefully others,
to justify taking on similar large projects in the future.

This is not particularly prestigious work (you can't write a phd on this),
but as shown in the problem statement it bothers *a lot* of people.
So a monetary 'carrot' should help us drag this over the line.

## Stakeholders
The primary benificiaries would be cabal users.
we can:

+ opens up the possibility to improve cabal user experience, 
  via inserting modules or running gen-bounds.
+ makes it easier for cabal library to deal with cabal files, such as hpack, HLS and cabal-fmt
+ makes maintaining cabal itself easier, 
  eg cabal init could be described via `GenericPackageDesription`

Furthermore I've heard that the HLS project will benefit this effort,
it could add a dependencies plugin for example: https://github.com/haskell/haskell-language-server/issues/155

## Success
This proposal successful once the cabal exact print branch is merged into cabal proper with the provided tests passing.
Most of hackage should be exact printable, 
which means, an existing hackage cabal file is being parsed, and then printed again resulting into the same output as input.
