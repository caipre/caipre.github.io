---
title: "Toward Making My First Rust Contribution"
date: "2016-10-12 16:53:21 -0400"
updated: "2016-10-13T10:18:32-04:00"
---

Rust is a modern systems programming language from Mozilla that focuses on
memory safety, high performance, and powerful language constructs (ADTs, type
inference, pattern matching, lambdas and closures, and lots more). For more
details on the language, refer to the [Rust homepage][rust]. This is the first
post in a series that will document my experience making contributions to
`rustc`, the mainline Rust compiler.

As I'm still in the early stages of what will be my first non-trivial
contribution, this entry has more questions than answers. Hopefully I'll be able
to write followup posts as I get more comfortable with the Rust codebase. For
now, I wanted to write something that might encourage others who are interested
in Rust development but are at the same level of unfamiliarity with the codebase
and unsure of how to start. So, then, what have I run into so far?

## I Have No Idea What I'm Doing!

This is scary! I can't think of another instance where I've been so intimidated
to get involved in a project. It's true that the Rust community is very helpful
and welcoming, but the language itself is (initially) unforgiving and I've not
worked on a compiler before, let alone one as complex as this. I was determined
to get involved though, so when [an interesting sounding issue][issue] showed up
in [This Week in Rust][twir], I jumped on it even though I had no idea where to
start. Since the issue was marked `E-mentor`, I had the assurance that if I got
too far stuck I could reach out for help from someone more knowledgable.

After browsing the code a bit, my list of unknowns became progressively more
distressing:

* How do I map from an AST node back to the code?
* After the recent error refactoring, are `struct_span_{err,warn}` still the
  correct way to prepare error messages (called "diagnostics")?
* How can I add a message to a diagnostic without introducing a whole new error?
* What does this [`tcx` field][tcx] represent, and where is it defined?
* Wait, `ctags` doesn't work? And `racer` can't figure it out either?
* Erhm, debuggers don't really work?
* Where are the docs for internal crates anyway?
* How do I even build `rustc`?!

## What I've Learned So Far

**How to build `rustc`**

Answering the last question first, there are currently two build systems for
`rustc`: the standard `make` based system, and the newer `rustbuild` system. My
understanding is that the former is still the standard method. The latter will
eventually replace it but work is still ongoing (as I write this though, I notice
that the [Rust CI on Travis][travisci] *does* `--enable-rustbuild`).

For my purposes (working on `rustc`, rather than anything in the standard
library), I don't need to build all the [stages of the compiler][rustcstages].
Hence, I build my `rustc` with the following options:

{% highlight shell %}
$ ./configure --enable-ccache --enable-debug --enable-clang --enable-fast-make
$ make x86_64-apple-darwin/stage1/bin/rustc
{% endhighlight %}

You should replace `x86_64-apple-darwin` with the appropriate target-triple for
your platform. Alternatively, you can `make rustc-stage1`. I'd love to know
whether these are equivalent!

*Edit:* The two targets are in fact distinct, and the latter is the correct
choice. Building just `rustc` leaves out the standard libraries, which are
required when running the various tests (either by `make check` or directly, eg,
`./x86_64-apple-darwin/stage1/bin/rustc ./src/test/compile-fail/...`).

**About `ctags` knowing nothing about `rustc`**

My problem was that I was using `make TAGS.vi`. This target (helpfully?)
excludes parsing the internal compiler crates, instead providing tags only for
the standard library. So to provide tags for my purposes, I instead run:

{% highlight shell %}
$ ctags --options=./src/etc/ctags.rust --recursive -o tags ./src/librustc*
{% endhighlight %}

Now that I have `ctags`, [`vim-racer`][racer], and [YouCompleteMe][ycm]
installed, I'm finding it much easier to jump around the codebase and write Rust
code naturally. Vim tip: `ctrl-o` will move to the previous point in the jump
stack, `ctrl-i` will move forward. Since `racer` doesn't operate on the tag
stack, the usual `ctrl-t` binding won't work.

**Using debuggers**

The most important piece for working with a debugger is ensuring that the
`--enable-debug` flag passed during the build step above. Unlike default builds
when using `cargo`, `rustc` will default to an optimized build without debugging
symbols.

When actually using the debugger, know that support is limited. The build
process provides scripts to add rudimentary language support to `lldb`, but you
shouldn't expect much more than printing local variables or setting breakpoints
at lines in the current file. GDB recently [announced support for Rust][gdb],
so hopefully this pain point is no longer an issue.

**Finding documentation and getting help**

Proper documentation for the internal crates does exist, once you know how to
find it. There are `README` files within many directories that are required
reading; these usually provide a detailed overview of the crate or module. A
good starting point that covers `rustc` as a whole is found [in the `librustc`
crate][readme].

If no `README` is present, check for a module doc comment in the crate's
`lib.rs`.  These can be a bit painful to read inline, which is where the
[internal documentation][rustcdocs] pages come into play. These are organized
just like the standard library docs, meaning they're incredibly useful to
understand how structures relate and what functionality a crate provides. Note
though that the search still queries the standard library's index, so you have
to cheat by using your favorite search engine with a filter, eg,
`mem_categorization site:http://manishearth.github.io/rust-internals-docs/rustc/`

If all else fails, hop on IRC. The `#rust` and `#rust-beginners` channels are
active and the community is welcoming, friendly, and helpful. Setting up an IRC
client was a process in itself (I eventually set myself up with [weechat]), but
more than worth it. I've now had multiple enlightening conversations with Niko
about the issue: initially we believed that the solution lay within the type
checker (see [conversation in issue thread][conversation]), but after talking on
IRC, Niko realized that a better solution is possible via changes in the borrow
checker.

That the Rust project core team offers such mentorship is incredible and speaks
highly of the community and of those individuals. Take advantage of it!

## Where Am I Now?

I've had my name on this issue for far too long. I assigned myself to it back in
August, but since then I've been distracted by several other projects that were
also interesting and maybe a bit less intimidating. A handful of these were Rust
projects, which at least helped me to get more comfortable with the language.
Now it's time to buckle down and figure out this issue.

[rust]: https://rust-lang.org
[issue]: https://github.com/rust-lang/rust/issues/28419
[twir]: https://this-week-in-rust.org/
[tcx]: https://github.com/rust-lang/rust/blob/9cb01365eed598811aef847a8ee414dab576f3c8/src/librustc_typeck/check/autoderef.rs#L198
[travisci]: https://travis-ci.org/rust-lang/rust/
[racer]: https://github.com/racer-rust/vim-racer
[ycm]: https://github.com/Valloric/YouCompleteMe
[rustcstages]: https://github.com/rust-lang/rust/blob/9cb01365eed598811aef847a8ee414dab576f3c8/Makefile.in#L149,L159
[gdb]: https://sourceware.org/gdb/current/onlinedocs/gdb/Rust.html#Rust
[rustcdocs]: http://manishearth.github.io/rust-internals-docs/rustc/index.html
[readme]: https://github.com/rust-lang/rust/tree/master/src/librustc#readme
[weechat]: https://weechat.org/
[conversation]: https://github.com/rust-lang/rust/issues/28419#issuecomment-242909711
