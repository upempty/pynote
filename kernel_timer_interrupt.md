
- Periodic(not fixed time) scheduler
```

dynamic tick: red black tree stores the sw timers, which will trigger the hw timer changed as requested. if no task, then no timer interrupt.

The function scheduler_tick() is called on every timer interrupt with the frequency, set at
compile time, stored in a preprocessor constant named HZ. Many architectures or even machine
types of the same architecture have varying tick rates, so, when writing kernel code, never assume
that HZ has any given value (in current version of the kernel, if there is no tasks to be run, the tick is
disabled; this does not really have any impact on anything, but power saving).
```
