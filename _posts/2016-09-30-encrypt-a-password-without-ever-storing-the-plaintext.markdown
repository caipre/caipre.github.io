---
layout: post
title: "Encrypt a password without ever storing the plaintext"
date: "2016-09-30 21:53:59 -0400"
---

The advice in [this article][] is still largely correct, but I wanted to avoid
writing my password in plaintext at any point in the process. It turns out this
isn't too difficult to achieve. Both `bash` and `zsh` have builtin commands to
read from `stdin`, and the `-s` flag will prevent echoing back the typed
characters. For `zsh`, another flag (`-e`) will forward the bytes to `stdout`.
Curiously though, that flag is ignored when `stdout` is connected to a pipe.
Solution: use a subprocess. Thus, for `zsh`:

{% highlight shell %}
$ (read -s -e) | gpg --encrypt -r <your id> > pw.gpg
{% endhighlight %}

Since `bash` doesn't support the `-e` flag, we just wrap the whole pipeline in a
subprocess to prevent the variable from living past the scope of the command:

{% highlight shell %}
$ (read -s pw && echo "set my_password = $pw" | gpg --encrypt -r <your id> > pw.gpg)
{% endhighlight %}

And there you have it: a `gpg` password file, and no plaintext was ever written
to disk.

[this article]: https://pthree.org/2012/01/07/encrypted-mutt-imap-smtp-passwords/
