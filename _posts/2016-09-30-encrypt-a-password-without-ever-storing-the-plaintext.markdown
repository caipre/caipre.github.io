---
layout: post
title: "Encrypt a password without ever storing the plaintext"
date: "2016-09-30 21:53:59 -0400"
---

The advice in [this article][] is largely correct, but I didn't want to write my
password in plaintext anywhere in the process. It turns out this isn't too
difficult. Both `bash` and `zsh` have builtin commands to read from `stdin`, and
the `-s` flag will prevent echoing back the typed characters. For `zsh`, another
flag (`-e`) will forward the bytes to `stdout` --- curiously though, the flag is
ignored when `stdout` is connected to a pipe. Solution: use a subprocess.

Thus, for `zsh`:

{% highlight shell %}
$ (read -s -e) | gpg --encrypt -r <your id> > pw.gpg
{% endhighlight %}

`bash` doesn't have the `-e` flag, so wrap the whole pipeline in a subprocess to
prevent the variable holding the password from living past the scope of the
command:

{% highlight shell %}
$ (read -s pw && echo "set my_password = $pw" | gpg --encrypt -r <your id> > pw.gpg)
{% endhighlight %}

And there you have it: `gpg` encryption, with no plaintext ever written to disk.

[this article]: https://pthree.org/2012/01/07/encrypted-mutt-imap-smtp-passwords/
