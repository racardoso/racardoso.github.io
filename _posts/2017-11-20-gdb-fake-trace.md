---
layout: post
title:  "Improvising trace on GDB"
date:   2017-11-20 18:57:07 -0200
categories: gdb trace tracepoints tutorial debug
---

I've recently write about GDB [tracepoints][tracepoints_gdb] here but. what if your platform doesn't support tracepoint facility?

```console
Target does not support this command.
```

So, either GDB is old or the target architecture isn't supported. In that case, you can still improvise by specify actions for the *breakpoint*. But, keep in maind that *tracepoints* are non obstrutive and more lightweigth. So, if you intend to debug a real-time application you should better use others trace tools. If isn't the case, what I'll show here can be helpful.

Consider the following C program:

```c

#include <stdio.h>

int main(void) {
  int i;
  int a = 0;

  for (i = 0; i < 128; i++)
    a = i*2;

  return 0;
}

```

Now we compile it with debug info and launch GDB. Suppose we want to log the values of *a*:

```console

(gdb) b foo.c:8
Breakpoint 1 at 0x4004ea: file foo.c, line 8.
(gdb) commands 1
Type commands for when breakpoint 1 is hit, one per line.
End with a line saying just "end".
>silent
>print a
>cont
>end
(gdb) r
Starting program: /a.out 
$1 = 0
$2 = 0
$3 = 2
$4 = 4
$5 = 6
$6 = 8
...
$124 = 244
$125 = 246
$126 = 248
$127 = 250
$128 = 252

```
The *silent* command tells gdb to not print current frame info and then we print a value and continue the execution. Notice that you can use *printf* instead *print* here to give you a more clean log. In this example I only print the value, but we can inspect the register state by replace *print* a by *info reg* for instance, which is analogue to *collect $regs* that we use in *trace*. 

You can read more about *commands* [here][gdb_manual]. Hope you find this useful.

[tracepoints_gdb]: https://racardoso.github.io/gdb/trace/tracepoints/tutorial/debug/2017/11/17/gdb-trace-command.html
[gdb_manual]: ftp://ftp.gnu.org/old-gnu/Manuals/gdb/html_node/gdb_34.html

