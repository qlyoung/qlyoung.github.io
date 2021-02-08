---
layout: post
title:  "GNU Readline 8.1 - disable bracketed paste at runtime"
---

Recently GNU Readline, the venerated line editing library used in everything
from bash to
[vtysh](http://docs.frrouting.org/projects/dev-guide/en/latest/vtysh.html),
enabled bracketed paste mode by default. This is a significant behavior change,
and while it's [noted](https://lwn.net/Articles/839213/) that there is a
compile time option to change this default, generally you can't tell users to
recompile some system library to get a feature back.

The Readline docs themselves are in the typical overly-terse GNU style so this
is a quick note on how to restore the old behavior in your readline-using
program. There's a function in readline that takes a string and interprets it
as if it were present in the user's `.inputrc` file; this can be leveraged to
disable bracketed paste mode like so:

```c
char *disable_bracketed_paste = strdup("set enable-bracketed-paste off");
rl_parse_and_bind(disable_bracketed_paste);
free(disable_bracketed_paste);
```
