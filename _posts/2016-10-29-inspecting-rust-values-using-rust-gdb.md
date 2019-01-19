---
title: Inspecting Rust values using `rust-gdb`
date: 2016-10-29T19:29:47.000Z
---

In Rust, `Rc<T>` will auto-deref into `T` because `Rc` implements the `Deref`
traits. This is useful behavior and idiomatic within Rust. Unfortunately, no
such auto-deref functionality exists when operating within a debugger. Instead,
one must manually traverse the `Rc` substructure to reach `T`. This post shows
how to do so, with some comments on the current state of debuggers.

### Background

As part of [this Rust issue][issue], I have been working with with values of
type `Rc<cmt>` . A `cmt` is [a structure][cmt] that describes the category,
mutability, and type of a node within the AST. Enclosed within the `Rc` type, it
looks like this in `rust-lldb` (indentation added):

{% highlight shell %}
(lldb) p cmt
(alloc::rc::Rc<rustc::middle::mem_categorization::cmt_>) $1 = Rc<rustc::middle::mem_categorization::cmt_> {
   ptr: Shared<alloc::rc::RcBox<rustc::middle::mem_categorization::cmt_>> {
      pointer: NonZero<*const alloc::rc::RcBox<rustc::middle::mem_categorization::cmt_>>(&0x10b40cab0),
      _marker: PhantomData<alloc::rc::RcBox<rustc::middle::mem_categorization::cmt_>>
   }
}
{% endhighlight %}

There are a few more layers here than I expected. `Rc` has a field `ptr` of type
`Shared`, which itself has a field `pointer` of type `NonZero`. This latter type
is a feature deriving from Rust's safety guarantees: `NonZero` is [a
tuple-struct][nonzero] that acts as a signal to LLVM that the contained pointer
will never be null. This can allow for non-null pointer optimizations.
Unfortunately, it's a roadblock when trying to get access to the interior data
from a debugger:

{% highlight shell %}
(lldb) p cmt.ptr.pointer.0
(double) $2 = 0
  Fix-it applied, fixed expression was:
    cmt.ptr.pointer;.0
{% endhighlight %}

This is because `lldb` doesn't understand Rust syntax, and instead tries parsing
the expression as though it were C++ code. That's too bad for us: C++ doesn't
have Rust's concept of tuple-structs, so we can't access the data inside
`NonZero(T)`. For reasons I don't understand, `lldb` won't let you cast away
the `NonZero` and isn't even able to locate the `RcBox` type in a cast:

{% highlight shell %}
(lldb) p (alloc::rc::RcBox<rustc::middle::mem_categorization::cmt_>*) cmt.ptr.pointer
error: no member named 'RcBox' in namespace 'alloc::rc'
error: expected '(' for function-style cast or type construction
error: expected expression
{% endhighlight %}

We need some way to access the data held by the tuple-struct, but `lldb` doesn't
even know what a tuple-struct is. I've not been able to get past this with
`rust-lldb`. What can we do?

### Speaking Rust

Fortunately, `gdb` recently [announced support][gdbrust] for Rust. Let's try to
access the data contained by the `NonZero` using `rust-gdb`:

{% highlight shell %}
(gdb) p cmt.ptr.pointer.0
$29 = (alloc::rc::RcBox<rustc::middle::mem_categorization::cmt_> *) 0x7fffe4122a50
{% endhighlight %}

Nice!

Note the type of the address: it's not `cmt` as we might have thought, but
rather `RcBox<cmt_>`. The `cmt_` type is an internal struct that actually owns
the data. The interface exposes this under an `Rc` using the name `cmt`.  Rust's
auto-deref logic makes the `Rc` part transparent to any consumers of the API.

So what does the `RcBox<cmt_>` look like then?

{% highlight shell %}
(gdb) p *cmt.ptr.pointer.0
$30 = RcBox<rustc::middle::mem_categorization::cmt_> = {
  strong = Cell<usize> = {
    value = UnsafeCell<usize> = {
      value = 2
    }
  },
  weak = Cell<usize> = {
    value = UnsafeCell<usize> = {
      value = 1
    }
  },
  value = cmt_ = {
    id = NodeId = {53},
    span = Span = {
      lo = BytePos = {717},
      hi = BytePos = {718},
      expn_id = ExpnId = {4294967295}
    },
    cat = Deref = {Rc<rustc::middle::mem_categorization::cmt_> = {
        ptr = Shared<alloc::rc::RcBox<rustc::middle::mem_categorization::cmt_>> = {
          pointer = NonZero<*const alloc::rc::RcBox<rustc::middle::mem_categorization::cmt_>> = {0x7fffe4122b20},
          _marker = PhantomData<alloc::rc::RcBox<rustc::middle::mem_categorization::cmt_>>
        }
      }, 1, BorrowedPtr = {rustc::ty::BorrowKind::ImmBorrow, 0x7fffe4097e18}},
    mutbl = rustc::middle::mem_categorization::MutabilityCategory::McImmutable,
    ty = 0x7fffe420fe50,
    note = rustc::middle::mem_categorization::Note::NoteNone
  }
}
{% endhighlight %}

Our stack-based `Rc<T>` structure references a corresponding heap-based
`RcBox<T>` structure; the `strong` and `weak` fields are there to break circular
references, and the `value` field holds the contained data. Finally we can see
the fields of our `cmt`! Notice that this particular `cmt` has category `Deref`,
which references another `cmt`; this is the value we deref'd from. So `cmt`s can
form a hierarchy, loosely mirroring the AST.

How about that `ty` field? Why is it printed as an address rather than pretty
printing its fields? The reason is that `rustc::ty::Ty` [is an alias][tyalias]
for an `&rustc::ty::TyS`. We can see the data with a cast, but we need to help
`gdb` a bit with the namespacing:

{% highlight shell %}
(gdb) p *(*cmt.ptr.pointer.0).value.ty as extern rustc::ty::TyS
$32 = TyS = {
  sty = TyAdt = {0x7fffe40323d0, 0x1},
  flags = Cell<rustc::ty::TypeFlags> = {
    value = UnsafeCell<rustc::ty::TypeFlags> = {
      value = TypeFlags = {
        bits = 983040
      }
    }
  },
  region_depth = 0
}
{% endhighlight %}

The `extern ...` keyword is required to inform `gdb` that the namespace should
be taken from root rather than the current crate.

### Final word

For me, the ability to explore code in a debugger is invaluable. Without the
requisite knowledge to make changes without re-compiling to see their effects,
I've found it difficult to make much progress using an edit-compile-test
workflow, especially when re-compiling `rustc` takes upwards of 15 minutes on my
machine. The debugger allows me to examine local variables and the call stack
such that I can have a better understanding of what values are available and how
the compiler operates, without wasting time waiting for compilation to finish
just to see the output of a new `println!` or debug macro.

Note that, short of using a proper debugger, the compiler does have quite a bit
of debug logging in place. I wasn't aware of how to actually see these messages
until recently, and they can be quite useful given the limited debugger support
Rust has. An example that enables `debug` level logs within a module under the
borrow checker:

{% highlight shell %}
$ RUST_LOG=rustc_borrowck::borrowck::gather_loans=debug ./x86_64-apple-darwin/stage1/bin/rustc ...
{% endhighlight %}

See the [`env_logger`][envlogger] documentation for full details.

Finally, there is one big gotcha for mac developers: `gdb` is [currently
broken][gdbmacos] on macOS Sierra. Since `lldb` doesn't speak Rust and `gdb`
crashes when trying to run a program, I've had to do my debugging on a Linux
virtual machine, which is far from ideal, but at least works.

[cmt]: https://github.com/rust-lang/rust/blob/421b595f25e5dbab9c967afd03929d013f346322/src/librustc/middle/mem_categorization.rs#L171,L195
[cmttype]: http://manishearth.github.io/rust-internals-docs/rustc/middle/mem_categorization/type.cmt.html
[envlogger]: http://rust-lang-nursery.github.io/log/env_logger/index.html
[gdbmacos]: https://github.com/Homebrew/homebrew-core/issues/5912
[gdbrust]: https://sourceware.org/gdb/current/onlinedocs/gdb/Rust.html#Rust
[issue]: https://github.com/rust-lang/rust/issues/28419
[nonzero]: https://doc.rust-lang.org/core/nonzero/struct.NonZero.html
[tyalias]: http://manishearth.github.io/rust-internals-docs/rustc/ty/type.Ty.html
