Linux networking

```

ps aux | grep ksoftirqd
root          15  0.0  0.0      0     0 ?        S    Apr12   0:00 [ksoftirqd/0]
root          26  0.0  0.0      0     0 ?        S    Apr12   0:00 [ksoftirqd/1]
root       19313  0.0  0.1   6428  2304 pts/1    S+   20:44   0:00 grep --color=                                                                             auto ksoftirqd

 cat /proc/softirqs
                    CPU0       CPU1
          HI:          0          0
       TIMER:    1372148    1755180
      NET_TX:          8          2
      NET_RX:      47104      49305
       BLOCK:          0      80401
    IRQ_POLL:          0          0
     TASKLET:          4        524
       SCHED:    2045015    2303142
     HRTIMER:          0          0
         RCU:    1926990    1905630

watch -n1 grep RX /proc/softirqs
watch -n1 grep TX /proc/softirqs

```
