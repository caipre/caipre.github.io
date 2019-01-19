---
title: "A functional Scheme REPL in 500 lines of Haskell"
date: "2016-10-07 17:15:17 -0400"
---

I've just finished the [*Write Yourself a Scheme in 48 Hours*][wikibook]
wikibook, implementing a basic REPL and stdlib for Scheme using Haskell and
Parsec. You can browse the code [on my GitHub][hume]; it's only some 500 lines.

Having completed it, though, I wouldn't recommend the book to another beginner
as a means of learning Haskell. The lessons spend too much time dictating code
without providing any background for *why* it's being done that way. Despite
claiming to be written for people interested in learning Haskell, the book
mostly assumes familiarity with central concepts such as monads and typeclasses.
I also found that not having a working knowledge of Scheme made for a rather odd
experience, as the lessons never consider questions like, "What's a dotted
list?", or "How are `eqv?`, `eq?`, and `equal?` different?".

I think the book would benefit tremendously from an introductory chapter that
discusses the design of the interpreter at a high level, perhaps broadly
introducing some of the more core concepts of Haskell and specifying the basic
Scheme syntax that will be implemented. Without such guidance, the novice reader
is left to plod on through the book assumingly, running headlong through two
unfamiliar languages in a nontrivial problem domain. Much of the reader's time
will be spent scratching their head, squinting hard to glimpse some particle of
understanding from a seemingly arbitrary composition of partially-applied
higher-order functions pulled from an inscrutable, previously unknown library,
then lifted, mapped, and returned within one or sometimes two monadic contexts.

Lest this entry be too defamatory, I should assure you that the book really
isn't so bad and that I'm exaggerating the case for effect. In actual fact, I
did learn quite a few things along the way, and the final lesson (of
implementing a very small portion of Scheme's standard library) was rather fun.

Highlights of the book include going from `eval args` to REPL in just a few
lines, adding support for variables, functions, and lambdas, and the discussion
of `foldr` versus `foldl` when implementing `stdlib.scm`.

Where next to continue learnng Haskell? Christopher Allen, the author of the
forthcoming book [*Haskell Programming from First Principles*][haskellbook]
discusses and reviews at length many of the resources available in [Functional
Education][funcedu]. I've also thought about trying to modernize the code I
wrote for the Scheme interpreter, possibly updating the wikibook afterwards. At
the moment, though, I'm most interested in beginning an LLVM tutorial that
implements a toy language named Kaleidoscope. The tutorial is written using
OCaml, a language not too unlike Haskell, and translating their implementation
seems like a worthwhile exercise.


[wikibook]: https://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours
[hume]: https://github.com/caipre/hume
[haskellbook]: http://haskellbook.com/
[funcedu]: http://bitemyapp.com/posts/2014-12-31-functional-education.html
