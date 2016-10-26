---
title: "Lessons learned from implementing Raft in Rust"
date: "2016-10-26 10:47:59 -0400"
---

A little more than a week ago, while I was working on [my toy OS project][mezzo]
in one of the Recurse Center's the side rooms, a few other Recursers entered the
room to discuss a shared effort toward implementing the Raft consensus
algorithm. The discussion immediately caught my attention: I knew vaguely about
the Raft algorithm, and distributed systems are of particular interest to me. I
was quickly drawn into the group and excited by the new project.

In this first meeting, we (the "Raft Implementers club") reviewed the paper's
proof of safety in log replication. The others had read the paper prior to the
meeting and talked about various points of confusion. The others were planning
to write implementations alternatively in Go, Julia, Haskell, and Elixer. I
decided that I would have a go at implementing the algorithm as well, using
Rust. That night, on the train home (and extending to the morning commute), I
read the paper describing Raft and familiarized myself with the various rules.

### raft.rs

After two false-starts, [raft.rs][] now has stable cluster communication and
leader election. Once I had the program compiling, I added quite a bit of
structured logging so that I can have confidence in the program's behavior.
Remaining work on the core algorithm includes accepting client commands and
tracking peer log indices. I also need to replace the message serialization
method with Protobuf; currently I'm simply encoding/decoding via colon-delimited
plaintext strings. I'd also like to move to finer grained locking: presently
there is one lock protecting the server's whole state. It should be possible to
answer read-only requests concurrently with a read/write lock.

Besides the fun of implementing the algorithm, this was a great growth exercise
as a Rust programmer. It used to be the case that I would write a bit of code
and run `cargo check` to learn about any borrowck or lifetime issues. After a
few days on this project, though, I was able to recognize and reason out such
problems while structuring the code. My first and second attempts mired in
deciding how to structure the code to satisfy the algorithm: how to represent
messages, timers, state, and object relationships. In my third attempt, having
resolved these design choices and learned to reason out ownership, I wrote the
majority of the program without ever running it through the borrow checker.
That's an immensely satisfying position!

### Takeaways from Rust

Rust is a fantastic language. The standard library is a joy to use: builtin
modules, traits, types, and methods are cohesive and well organized. The
documentation and its rendering are excellent. The ecosystem is large enough now
to provide multiple solutions for common functionality like argument parsing,
logging, working with dates and time, and object (de)serialization.

Here are a few things I've learned so far by working on this project:

* Writing a macro isn't nearly as hard as I thought, and they're even more
  useful than I imagined. I used macros to reduce [boilerplate][] and as a
  convenience for [building relatively][heartbeat] structures. I was pleasantly
  surprised to find [a similar pattern][implfrom] in use in the standard
  library.

* A `'static` lifetime parameter doesn't mean what I thought: I thought it
  required that the value must exist for the entire program, from initialization
  to termination. In fact, it only specifies that the value *is allowed to* live
  that long ([reference][]). Admittedly, this is still a bit fuzzy to me,
  especially since the definition given in [the book][conststatic] seems to
  agree with my original understanding. It may be that my confusion arises
  because lifetimes can be considered from the perspective of the value (which
  has a particular lifetime) or from the perspective of the item acting on the
  value (and describes a maximum lifetime bound).

* Working with dynamically typed data structures is a bit of a pain. Raft has
  three message types, each of which consists of a request/response part. I
  wanted my functions to be generic across these, and getting that right took a
  bit of trial and error. In my first attempt, I wrote parametric functions over
  an empty trait: `fn foo<T: RaftMsg>(...) { ... }`. Unfortunately, it isn't
  possible to use `match` patterns on such a type due to monomophization. In
  fact, the more general problem of using `enum` variants as types is a
  long-standing feature request and is properly called [refinement
  types][refinementtype]. For now I'm using separate `enum`s for each of the
  message [type][xtype], [part][xpart], and [data][msgdata]. The latter `enum`
  wraps the `structs` themselves so that I can use pattern matching to
  destructure the message and access the data.

* By implementing a periodic [`Timer`][timer], I feel like I finally have an
  understanding of how condition variables work. Really, they don't appear to be
  anything more than a useful interface defined over a `Mutex`.

* Locking plays nice with Rust's borrowck guarantees. A handful of various types
  available in `std::sync` provide a `get_mut` method that ensures read/write
  and thread-safety *statically*. This means that using this method to acquire
  the `RwLock` my protecting my server state is effectively a no-op: the
  compiler's guarantees are just as strong as the lock's. That's awesome.

* Properly done, a type system is absolutely amazing. When I dabbled in Haskell
  a few weeks back, I was introduced to the functor, applicative, monad
  hierarchy at an abstract level. Rust has solidified these concepts for me
  through use of the `Option` and `Result` types, along with the various
  `Iterator` adaptors. These are powerful, expressive constructs that Rust uses
  pervasively.

The more I use Rust, the more I enjoy it. Implementing Raft has been a very
rewarding exercise, touching on issues of system design, protocol serialization,
network communication, locking, and concurrency. There's been some debate about
whether or not it actually describes a distributed system (versus replicating
just a single state), but in any case it's been a fun project.

[mezzo]: https://github.com/caipre/mezzo.git
[raft.rs]: https://github.com/caipre/raft.rs
[boilerplate]: https://github.com/caipre/raft.rs/blob/e7f9bd13210db33fcb50518155eae890da2cdb06/src/lib.rs#L930,L943
[heartbeat]: https://github.com/caipre/raft.rs/blob/e7f9bd13210db33fcb50518155eae890da2cdb06/src/lib.rs#L624
[implfrom]: https://github.com/rust-lang/rust/blob/a6b3b01b5f7f5a9d7d340dacf7dbf72be29e2c07/src/libcore/num/mod.rs#L2782
[reference]: https://botbot.me/mozilla/rust/2016-10-25/?msg=75472223&page=26
[conststatic]: https://doc.rust-lang.org/book/const-and-static.html
[refinementtype]: https://www.reddit.com/r/rust/comments/2rdoxx/enum_variants_as_types/
[xtype]: https://github.com/caipre/raft.rs/blob/e7f9bd13210db33fcb50518155eae890da2cdb06/src/lib.rs#L741,L745
[xpart]: https://github.com/caipre/raft.rs/blob/e7f9bd13210db33fcb50518155eae890da2cdb06/src/lib.rs#L748,L751
[msgdata]: https://github.com/caipre/raft.rs/blob/e7f9bd13210db33fcb50518155eae890da2cdb06/src/lib.rs#L753,L758
[timer]: https://github.com/caipre/raft.rs/blob/e7f9bd13210db33fcb50518155eae890da2cdb06/src/timer.rs
