# uptime

## using uptime

Users of Linux systems can use the BSD uptime utility, which also displays the system load averages for the past 1, 5 and 15 minute intervals:

```shell
$ uptime
  18:17:07 up 68 days,  3:57,  6 users,  load average: 0.16, 0.07, 0.06
```

## using /proc/uptime

Shows how long the system has been on since it was last restarted:

The first number is the total number of seconds the system has been up. The second number is how much of that time the machine has spent idle, in seconds.[14] On multi core systems (and some Linux versions) the second number is the sum of the idle time accumulated by each CPU.

```shell
$ cat /proc/uptime
41341320.04 322383316.50
```
