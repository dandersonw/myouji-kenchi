# Problem Statement

Often when scraping data from the Internet we will find unstructured names,
that is to say names without structured data about 
which part is a given name, a family name, etc.
A structured representation that covered 
the whole variety of naming conventions in the world
would be challengingly complex.
However,
for the purposes of handling Japanese names, 
we can assume that names consist of one family name and one given name.
The Japanese language convention is to order names `family name` `given name`,
and we assume that Japanese names written in Japanese follow this convention.
An issue arises however when we find a romanized Japanese name,
that is to say a Japanese name written in Latin characters.
The Western convention is of course to write names `given name` `family name`,
and Japanese people follow this convention when romanizing their names - sometimes.
Depending on the individual,
and on the context of where we found the name,
a romanized Japanese name could really be in either order.
In order to make the best use of scraped data,
we want to determine which of two romanized Japanese names 
is the given name and which is the family name.

While more than one hundred thousand 
Japanese family names have been confirmed to exist [1],
only twenty thousand family names
cover more than ninety nine percent of Japanese people [2, 3].
Furthermore,
there is a lot of data available about the frequency of individual family names.
The situation with personal names is less tractable,
in that there are more personal names
and less extensive data about them.
Because of this data difference it was decided to treat the problem as
picking which of two names is more likely to be a family name.

## Romanization

Data available about Japanese names naturally stores names in two forms,
the text of a name used in normal writing,
and the phonetic characters used to write how a name is read (furigana).
Furigana alone is not enough to determine the romanization of a name [4],
but for our purposes it is sufficient to convert romanizations to furigana.
An issue arises however,
in that there is more than one official romanization scheme in active use today,
and in practice names are often romanized omitting
the marks and accents prescribed by the official schemes.
We could not find an existing library that could handle converting
potentially ambiguous romanized strings 
to Japanese phonetic characters.
For this project we decided to target the 
(Modified) Hepburn, Kunrei-shiki, and Nihon-shiki romanization schemes,
as well as those schemes with some common variations (e.g. omitting macrons).

(Some diagrams about romanization? Are the details important?)


# Finite State Transducers

The finite state transducer (FST) is a venerable tool 
that has found a lot of use in natural language processing 
for tasks from string normalization to part of speech tagging.
Essentially, they are finite state machines
that can produce output when they transition between states.
On a theoretical level they translate between two "languages" of symbols,
where any one input can result in:
* zero outputs, if the transducer does not accept an input
* one output, if the transducer is deterministic for an input
* many outputs, if the transducer is non-deterministic for an input

Translating between two languages of symbols is a perfect match to transliteration,
and the ability to produce multiple outputs
is a principled way to handle ambiguity that might be present.
We make use of the [OpenFst](http://www.openfst.org/twiki/bin/view/FST/WebHome) 
library for a performant and expressive base implementation.

An FST that performs the following translations:
* `cat` -> `dog`
* `cats` -> `dogs`

![cat-dog](img/cat-dog.svg)

`ε` is a special symbol, epsilon, 
which means that the arc can be taken by the transducer 
without reading input (if there is an epsilon on the input side)
and/or writing output (if there is an epsilon on the output side).

An FST that performs the following translation:
* `dog` -> {`wolf`, `wolves`}

![dog-wolves](img/dog-wolves.svg)

One good quality of FSTs is that they naturally compose together.
The below FST is the composition of the two above 
and it performs the translation:
* `cat` -> {`wolf`, `wolves`}

![cat-wolves](img/cat-wolves.svg)

A final useful quality of FSTs is that 
they can have weights on their arcs or final states.
The following FST translates `meow` to:
* `woof`, with weight 1/2
* `woof woof`, with weight 1/4
* `woof woof woof`, with weight 1/8
* etc.

![meow-woof](img/meow-woof.svg)


# Our Solution

To order two names we combine some heuristics
with a finite state transducer responsible for 
evaluating the likelihood strings are romanized Japanese family names.
That transducer is composed of two modules which are composed together,
a transliterator responsible for 
translating between Latin characters and furigana,
and an acceptor [5] responsible for 
injecting into the process lexical knowledge,
i.e. empirical knowledge about the use of language.


## Transliterator

The general strategy of the transliterator
is to utilize FSTs potential for nondeterminism
to generate all katakana strings theoretically possible.
As mentioned above, 
one common deviation from the Hepburn standard of romanization 
is the omission of apostrophes.
Ideally, 
we could be sure that the furigana `シンイチ` would be romanized to `Shin'ichi`
and `シニチ` would be romanized to `Shinichi`,
but in practice there is ambiguity.
The following piece of a transliterator does the translations:
* `Shinichi` -> {`シンイチ`, `シニチ`}
* `Shin'ichi` -> `シンイチ`

![shinichi](img/shinichi.svg)

For now we do not distinguish
the likelihood of different translations between characters,
but it would be possible to bake in empirical data about romanization
such that transliterations should be returned with likelihood scores.
That is to say,
if we found that in names seventy five percent of the time `ni`
should be transliterated to `ニ` and twenty five percent of the time to `ンイ`
we could write a transliterator to do a mapping:
* `Shinichi` -> {(`シンイチ`, .25), (`シニチ`, .75)}
* `Shin'ichi` -> (`シンイチ`, 1)


## Lexical Data

Just using the transliterator alone is not enough,
we need to incorporate lexical knowledge 
in order to distinguish what merely looks like a Japanese family name
from what actually could be one.
We combine two sources of data: 
relatively fine grained frequency information 
for about ten thousand of the most popular family names,
and a list of a few tens of thousand 
attested (i.e. known to exist at least once) family names.
We store the whole set of recognized names in an acceptor,
taking advantage of terminal state weights to store frequency information.
By composing this second module onto the transliterator,
we create an integrated system that takes in strings of Latin characters 
and returns potential Japanese strings with weights.

The below acceptor represents the lexical knowledge that the names
サトウ, サトミ, and サトハラ exist,
and with frequencies 1000, 10, and 1 respectively.

![alt](img/satoumi.svg)

The below image is the transliterator composed with the above acceptor:

![alt](./img/transliterate-satoumi.svg)

## Heuristics

In order to reduce the potential for ambiguity in romanized Japanese names,
some writers take the approach of writing the family name in all capital letters.
Where one of the two names we are trying to order is in all capitals
we assume that that name is the family name.
Furthermore,
in our data,
we find that in names where one name is an initial it is very likely that
the initialized name is the given name.
In truth,
these two heuristics resolve a lot of ambiguity all on their own.


# Open Source Package

We decided to open source under a MIT license 
the myouji-kenchi package we developed.
You can find the project code 
[on Github](https://github.com/scouty-inc/myouji-kenchi)
and releases available for installation 
[on PyPI](https://pypi.org/project/myouji-kenchi/).


\[1\] https://myoji-yurai.net/ The definition of distinct for family names
varies based on the source of data and the context of the discussion. For the
purpose of the one hundred thousand number, a name is counted as distinct from
another if either the preferred writing or the pronunciation are at all
different.

\[2\] http://www.orsj.or.jp/\~archive/pdf/bul/Vol.23_06_357.pdf

\[3\] For comparison,
although the United States is perhaps unusually diverse,
in the 2010 census more than two million surnames 
fail to cover ninety nine percent of Americans.

\[4\] It also depends on the kanji, or Chinese characters, that make up a name

\[5\] An acceptor is a special case of an FST that either produces no output or exactly the input.
