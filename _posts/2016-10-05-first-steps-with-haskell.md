---
title: First steps with Haskell
date: 2016-10-05T21:54:01.000Z
---

A few days ago I started working through the [*Write Yourself a Scheme in 48
Hours*][wikibook] wikibook. The book uses Haskell to build a working Scheme parser and REPL
from the ground up. As a means of learning Haskell, one should be aware that the
book doesn't describe language features well (read: at all), and the code is a
bit dated, to the point where `ghci` complains about use of deprecated
functionality (in this case, `Control.Monad.Error`). For my purposes --- I just
want to see the language before really diving in --- it gets the job done.

I originally had it in mind to create a pet programming language compiling to
LLVM. When talking to friends about that project though, I was referred to the
Haskell wikibook. I decided that it was probably worthwhile to see how a real
language is parsed and evaluated before spending too much time designing my own
language and parser. Haskell has long been on my to-do list, and my friend V's
enthusiasm convinced me it's worth taking the time to learn.

Well, I was mistaken in understanding the "48 hours" part of the tutorial as
meaning "two days". I guess it really means closer to a week's time.

On the bright side, I now have a functional REPL for a useful subset of Scheme.
Moreover, I have a better understanding of Haskell. There's still a lot I need
to learn, but I thought I'd share some insights.

### Making analogies to Rust

Rust is an excellent language, and makes quite a few functional programming
concepts available in the context of systems programming. As my language of
choice right now, I've found that it's been helpful to understand new Haskell
concepts by making an analogy to Rust.

There are some obvious ones:

* Haskell's `Maybe a` is Rust's `Option<T>`
* Haskell's `case ... of` is Rust's `match ...`
* Both languages support pattern matching and destructuring

There are plenty of Haskell concepts that I'm still learning.

As far as I understand them, typeclasses in Haskell are analogous to Rust's
`trait`s. For example, the following Haskell:

{% highlight haskell %}
class Eq a where
   (==) :: a -> a -> Bool

data Foo = String

eqFoo a b = a == b

instance Eq Foo where (==) = eqFoo
{% endhighlight %}

...translates roughly to Rust as:

{% highlight rust %}
trait Eq {
   fn eq(a, b) -> Bool;
}

struct Foo(String);

impl Eq for Foo {
   fn eq(&self, &other: Self) -> bool {
      self.0 == other.0
   }
}
{% endhighlight %}

I don't know how/if Haskell has a concept of [trait objects]. I also don't know
much about Haskell's support for generics, at least in the Rust sense (ie, `<T>`).

Haskell seems to be able to constrain types, but I don't grok the syntax yet.
In Rust, I can write the following:

{% highlight rust %}
fn myfunc<T: Display>(str: T) {
   println!("{}", str)
}

{% endhighlight %}

For Haskell, I think it's something like this:

{% highlight haskell %}
myfunc :: Show str => str -> String
myfunc str = show str
{% endhighlight %}

I might revisit the comparison between these languages as I get more
comfortable with each.

### Understanding Monads and `do` notation

Monads might be more subtle than I'm aware of, but for the moment I'm just
thinking of them as a container for value that does some additional background
work through the `>>=` and `>>` operators. I like to think of this as a more
functional Unix pipe --- the book even calls it a "monad pipeline" at one point.

Consider the following command pipeline: `cat /usr/share/dict/words | tee file`

I claim that `| tee` is a monad, where the "additional background work" is to
repeat `stdin` both into a file and into `stdout`. The key idea is that there's
a value coming into the monad, which is then free to do whatever it wants before
passing the value to the next action in the chain.

This "action, extra work, forward" cycle is hidden by the `do` notation, so it's
useful to know how to desugar the syntax. Consider the following Haskell code,
which implements the evaluation of an `if` statement in Scheme:

{% highlight haskell %}
eval env (List [Atom "if", pred, conseq, alt]) =
   do result <- eval env pred
      case result of
         Bool False -> eval env alt
         otherwise -> eval env conseq
{% endhighlight %}

We can desugar the `do` to:
{% highlight haskell %}
eval env (List [Atom "if", pred, conseq, alt]) =
   eval env pred >>= \result -> case result of ...
{% endhighlight %}

Essentially, `>>` can be pronounced as "and then ...", while `>>=` can be
pronounced as "and forward the result to ...".


[wikibook]: https://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours
[trait objects]: https://doc.rust-lang.org/book/trait-objects.html
