---
layout: post
title:  "Tracepoints: A GDB feature that you must known"
date:   2017-11-17 16:27:07 -0200
categories: gdb trace tracepoints tutorial debug
---

Althought *breakpoints* are very useful to debug programs, sometimes you can only get some useful information by collecting data over long periods of time for later analysis. Another problem is that, sometimes, you have some code that's is very time-sensitive to be debugged interactively. If the program depends on real-time behaviours to be correct, interrupt the program execution might change the execution behaviour in a way that's not desirable.

Using GDB’s *trace* and *collect* commands, you can specify locations on the program called *tracepoints*. With the *collect* you can specify what you wan't do collect. Once GDB hits a *tracepoint* and later you can examine those values. Because GDB records these values without interaction, it can do it quickly and unobtrusively. 

However, tracepoint facility is currently available only for remote targets and your remote target must know how to collect trace data.

### Starting GDB trace

Let's use a small example in C:

```c
#include <stdio.h>

int foo (int a) {
  int i;
  for (i = 0; i < 128; i++)
    a += i;

  return a;
}

int main (void) {
  int b = foo(1);
  printf("%d\n", b);

  return 0;
}
```

Save the file as foo.c and compile with debug info:

```console

gcc -O0 -g foo.c

```

First you need to start *gdbserver* at localhost on another terminal:

```console

gdbserver :0 ./a.out

```

This will start gdbserve at localhost in a random port. Note that you can pass the port or specify the device e.g: *:10000* will initiate gdb server at port 1000. You can read more about it [here][gdb_server].

```console

$ gdbserver :0 ./func
Process ./foo created; pid = 30387
Listening on port 40141
Remote debugging from host 127.0.0.1

```

Now open another terminal and start gdb:

```console

gdb -ex 'target remote :40141'

```
This will start gdb and execute a single command (-ex) *target remote :40141* - replace *40141* by the port given when you start gdbserver - and connect to the remote gdbserver.

```console

Remote debugging using :34119
Reading /a.out from remote target...
warning: File transfers from remote targets can be slow. Use "set sysroot" to access files...
Reading /a.out from remote target...
Reading symbols target:/a.out...concluído.
Reading /lib64/ld-linux-x86-64.so.2 from remote target...
Reading /lib64/ld-linux-x86-64.so.2 from remote target...
Reading symbols target:/lib64/ld-linux-x86-64.so.2...Reading /lib64/ld-2.23.so from remote...
Reading /lib64/.debug/ld-2.23.so from remote target...
(no debugging symbols found)...concluído.
0x00007ffff7dd7c30 in ?? () from target:/lib64/ld-linux-x86-64.so.2
(gdb) 

```
### Tracepoints

Lets set a *tracepoint* on our program. Suppose we want to trace things that happens when we execute the sum *a += 1* at *foo* function:

```

(gdb) trace foo.c:6
Tracepoint 1 at 0x40052d: file foo.c, line 6.

```
Notice that are other ways to set a *tracepoint*. You can read more [here][setting_tracepoints].

Check the *tracepoint* that you set using *t info* command:

```console

(gdb) info tr
Num     Type           Disp Enb Address            What
1       tracepoint     keep y   0x0000000000400536 in foo at func.c:6
	not installed on target

``` 
Notice that info result says **not installed on target**. The tracepoint will installed once you run *tstart* but before, you need to configure what will be monitored.

### Actions and Collect

Once you set a *tracepoint* you need to define what you (actions) will do when gdb hits this *tracepoint*:

```console

(gdb) actions 1
Enter actions for tracepoint 1, one per line.
>

```
The gdb command *actions*  will allow you to enter a set a list of actions for the tracepoint. The argument is the number of the tracepoint[^1]. The actions themselves appear on the following lines. Type a line containing just *end* to terminate the actions.

Now we need to especify what we want to *collect*:

```console

(gdb) actions 1
Enter actions for tracepoint 1, one per line.
> collect a
> end
(gdb)

```

In this case I want to trace variable *a*. Any global, static, or local variables may be collected. In addition, special arguments *$regs, $args, $locs* are supported, which allows to collect the machine register state, arguments and local variables of traced function.

You can check the *tracepoint* is now installed:

```console

(gdb) info tr
Num     Type           Disp Enb Address            What
1       tracepoint     keep y   0x0000000000400536 in foo at foo.c:6
        collect a
	installed on target

```


Now before run the trace, on this particulary example, we must set a *breakpoint* on return of main function. Why? Unlike *breakpoints*, *tracepoints* do not suspends program execution. Once you program finish the child process will exit with status 0, gdbserver terminates and you will lost the trace:

```console

(gdb) c
Continuando.
Reading /lib/x86_64-linux-gnu/libc.so.6 from remote target...
Reading /lib/x86_64-linux-gnu/libc-2.23.so from remote target...
Reading /lib/x86_64-linux-gnu/.debug/libc-2.23.so from remote target...
[Inferior 1 (process 30594) exited normally]
(gdb) tfind
May not look at trace frames while trace is running.
(gdb) tstop
You can't do that when your target is `exec'

```

Unless the execution is interrupted by a signal (like a `SIGSEGV`) we must find a point on the program execution that we can put a breakpoint without interfere on the trace analysis. In this case we can put that on return from main:

```console

(gdb) b foo.c:15
Breakpoint 2 at 0x400574: file foo.c, line 15.

```

### Running trace and dump results

The gdb command *tstart* will start to running the trace (it's a silent command):

```console

(gdb) tstart

```

And then you can run (continue) the program:

```console

(gdb) c

```
It will run and stop at breakpoint:

```console

(gdb) c
Continue.
Reading /lib/x86_64-linux-gnu/libc.so.6 from remote target...
Reading /lib/x86_64-linux-gnu/libc-2.23.so from remote target...
Reading /lib/x86_64-linux-gnu/.debug/libc-2.23.so from remote target...

Breakpoint 2, main () at func.c:15
15	  return 0;

```
We can use *tstop* now to stop[^2] collect data:

```console

(gdb) tstop

```

Now lets review the results. Every time the program hits a *tracepoint*, a trace event generates a data record (trace frame) that will be stored in the trace buffer. Each trace frame is identifyied by a sequential number starting from zero and the address (PC) of the event. The basic commands for selecting a trace frame and extracting data from it are *tfind* and *tdump*.

The *tfind* command select a trace frame by id number. *tfind start* ( is a synonym for tfind 0 (lowest trace frame). If no argument is specified, selects the next frame. *tfind -* selects the previous frame, and *tfind none* selects no frame. Lets see what's on the lowest frame (frame 0):

```console

(gdb) tfind start
Found trace frame 0, tracepoint 1
#0  foo (a=1, a@entry=<error reading variable: PC not avaliable>) at func.c:6
6	    a += i;
(gdb) tdump
Data collected at tracepoint 2, trace frame 0:
a = 1

``` 
So we found the value of *a* when we start trace is 1. Nice. We can navigate for all the frames by using *tfind* and *tdump*. But, what if we want to print them all, instead go one by one. We have some [convenience variables][tracepoint_convenience] that can help us to do that:

```console

(gdb) while ($trace_frame != -1)
 >tdump
 >tfind
 >end
Data collected at tracepoint 2, trace frame 0:
a = 1
Found trace frame 1, tracepoint 2
Data collected at tracepoint 2, trace frame 1:
a = 1
Found trace frame 2, tracepoint 2
Data collected at tracepoint 2, trace frame 2:
a = 2
...
Found trace frame 34, tracepoint 2
Data collected at tracepoint 2, trace frame 34:
a = 562
Found trace frame 35, tracepoint 2
Data collected at tracepoint 2, trace frame 35:
a = 596
Found trace frame 36, tracepoint 2
---Type <return> to continue, or q <return> to quit---q

```

## Conclusion

This is just a "toy" example what you can do with GDB trace command. But is indeed a powefull feature that you must know. I recommend you to read the [GDB manual][gdb_manual] and this [article][tracing_sourceware] for more details.

* * *

### Notes:
 
[^1]: With no tracepoint number argument, *actions* refers to the last tracepoint set.
[^2]: Trace data collection may also be stopped automatically if any tracepoint's [passcount][gdb_passcounts] is reached or if the trace buffer becomes full (if not in [circular buffer mode][gdb_buffer_mode]).

[gdb_server]: ftp://ftp.gnu.org/old-gnu/Manuals/gdb/html_node/gdb_130.html
[setting_tracepoints]: https://sourceware.org/gdb/talks/esc-west-1999/commands.html#tracepoints
[tracepoint_convenience]: https://sourceware.org/gdb/talks/esc-west-1999/commands.html#convenience
[gdb_manual]: https://sourceware.org/gdb/current/onlinedocs/gdb/Tracepoints.html#Tracepoints
[tracing_sourceware]: https://sourceware.org/gdb/talks/esc-west-1999/commands.html
[gdb_passcounts]: https://sourceware.org/gdb/onlinedocs/gdb/Tracepoint-Passcounts.html
[gdb_buffer_mode]: https://www.sourceware.org/gdb/wiki/ProcessRecord/Tutorial#set_record_stop-at-limit
