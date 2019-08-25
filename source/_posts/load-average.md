---
title: 系统平均负载
date: 2019-08-25 15:40:00
tags:
	- linux - 性能优化
---

当我们发现系统变慢时，通常会执行`top`或者`uptime`命令，来了解系统的负载情况，比如：

```sh
$ uptime
 07:48:28 up 8 min,  1 user,  load average: 0.00, 0.08, 0.07
```

`uptime`命令输出的信息分别是：当前系统时间，系统运行时间，当前登陆用户数，以及**最近1分钟、最近5分钟和最近15分钟的系统评价负载情况**



### 什么是系统平均负载？

我们可以通过`man uptime`查看命令帮助手册，有这么一段话介绍什么是系统的平均负载：

> ​    System load averages is the average number of processes that are either in a runnable or uninterruptable state.  A process in a runnable state is either using the CPU or waiting to use  the  CPU.   A process  in  uninterruptable state is waiting for some I/O access, eg waiting for disk.  The averages are taken over the three time intervals.  Load averages are not normalized for the number of CPUs in a system, so a load average of 1 means a single CPU system is loaded all the time while on a 4 CPU system it means it was idle 75% of the time.

简单来说，平均负载就是指单位时间内，系统处于**可运行状态**和**不可中断状态**的平均进程数，也就是**平均活跃进程数**。这里的**可运行状态**包括**正在使用CPU（Running状态）**和**正在等待CPU（Runnable状态）**。而**不可中断状态**是指进程运行在内核态，正在等待`I/O`，**为了能够尽快完成`I/O`操作**，等待`I/O`的进程不会被中断。

通过`ps`命令查看进程时，可运行状态对应的是`R`，而不可中断状态对应的是`D`。

系统平均负载不会根据 CPU 数量进行标准化，也就是说，当系统平均负载是1的时候：

- 在单核系统上，说明 CPU 满跑
- 而在4核系统上，说明 CPU 有75%的空闲



### 平均负载为多少合适？

理想的情况下，是每个 CPU 都刚好运行着一个进程，这样每个 CPU 都能得到充分的利用，因此**理想情况下平均负载应该等于系统 CPU 的个数**。

当我们在评判系统负载情况的时候，应该先要知道系统有几个CPU：

```sh
$ grep 'model name' /proc/cpuinfo| wc -l
```

`/proc/cpuinfo`文件内会记录系统 CPU 的信息，我们可以直接通过统计该文件信息获取 CPU 个数。

通过与 CPU 个数对比，我们就可以知道当前系统的负载情况是否过载。

平均负载的值有三个，分别对应最近1分钟，最近5分钟和最近15分钟，我们可以根据这三个值来更全面的了解系统的负载情况：

- 如果这三个值基本相差不大，那就说明系统负载比较平稳
- 如果最近1分钟的负载远小于15分钟的负载，说明过去15分钟负载很大，而现在在慢慢减小
- 如果最近1分钟负载很大，而15分钟的很小，说明最近1分钟的负载在逐渐增加，这种增加可能是临时性的，也有可能会持续增加，需要持续观察

**在生产环境中，当平均负载高于CPU数量的70%时，就应该分析排查负载高的问题了**。一旦负载过高，就会导致进程响应变慢，进而影响服务正常功能。



### 平均负载与CPU使用率

CPU使用率是单位时间内对CPU繁忙情况的统计。根据平均负载的含义，我们知道这两者是两个不同的概念。

平均负载与CPU使用率两者之间也不一定是对应的：

- CPU密集型进程：会使用大量的CPU，这时候两者一般是一致的
- IO密集型进程：等待IO会导致平均负载升高，但是这时候进程是处于等待状态的，因此这时候的CPU使用率不一定高
- 大量进程等待CPU调度：这时候的平均负载会很高，而CPU使用率一般也会比较高

当我们发现系统的平均负载很高时，首先就需要分析是由于上面哪种原因导致的，这样才能有效的解决问题。



### 案例模拟分析

接下来我们来模拟一下平均负载过高的案例分析。

首先要先安装一下`stress-ng`和`sysstat`两个包。其中`stress-ng`是压力测试工具，用来模拟平均负载过高的场景，而`sysstat`包含了常用的性能工具，用来监控和分析系统的性能，包括：

- `mpstat`：多核cpu性能工具，用来实时查看每个cpu的性能指标，以及所有cpu的平均指标
- `pidstat`：进程性能分析工具，用来实时查看进程的cpu、内存、i/o以及上下文切换等性能指标



##### 场景一：CPU密集型进程

首先，在第一个终端运行：

```sh
$ stress-ng --cpu 2 --timeout 600
```

- `-- cpu 2`：使用2个worker去不断执行cpu密集的任务，这里使用2是因为我本地测试的虚拟机只有两个cpu
- `--timeout 600`：表示持续运行600s

接着在第二个终端运行：

```sh
$ watch -d uptime # -d 会高亮显示变化的区域
..., load average: 2.01, 1.44, 0.71
```

我们可以在这个终端看到，平均负载会逐渐升高，最后最近1分钟的平均负载会稳定在2左右

最后，在第三个终端运行：

```sh
$ mpstat -P ALL 5 20 # -P All 表示统计所有cpu信息，5表示间隔5s打印一次统计，20表示共打印20组统计
...
09:03:00 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:03:05 AM  all   99.80    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00    0.00
09:03:05 AM    0   99.80    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00    0.00
09:03:05 AM    1   99.80    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00    0.00

Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all   99.71    0.00    0.28    0.00    0.00    0.01    0.00    0.00    0.00    0.00
Average:       0   99.78    0.00    0.22    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:       1   99.64    0.00    0.34    0.00    0.00    0.02    0.00    0.00    0.00    0.00
```

我们可以看到，两个cpu的用户使用率都接近100%，说明是因为cpu密集进程导致的平均负载过高

接下来，我们可以使用`pidstat`命令来查询异常进程：

```sh
$ pidstat -u 5 2 # -u输出cpu使用率，间隔5s打印一次统计信息，共打印2组
Linux 4.4.0-159-generic (ubuntu-xenial) 	08/25/2019 	_x86_64_	(2 CPU)

09:11:59 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
09:12:04 AM     0      5336   99.20    0.00    0.00   99.20     1  stress
09:12:04 AM     0      5337   99.60    0.00    0.00   99.60     0  stress
09:12:04 AM  1000      5344    0.20    0.20    0.00    0.40     1  watch
09:12:04 AM  1000      5538    0.00    0.20    0.00    0.20     1  pidstat

09:12:04 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
09:12:09 AM     0      1112    0.00    0.20    0.00    0.20     1  iscsid
09:12:09 AM     0      1179    0.20    0.00    0.00    0.20     0  containerd
09:12:09 AM     0      5336   99.60    0.00    0.00   99.60     1  stress
09:12:09 AM     0      5337  100.00    0.20    0.00  100.20     0  stress
09:12:09 AM  1000      5344    0.20    0.00    0.00    0.20     1  watch

Average:      UID       PID    %usr %system  %guest    %CPU   CPU  Command
Average:        0      1112    0.00    0.10    0.00    0.10     -  iscsid
Average:        0      1179    0.10    0.00    0.00    0.10     -  containerd
Average:        0      5336   99.40    0.00    0.00   99.40     -  stress-ng-cpu
Average:        0      5337   99.80    0.10    0.00   99.90     -  stress-ng-cpu
Average:     1000      5344    0.20    0.10    0.00    0.30     -  watch
Average:     1000      5538    0.00    0.10    0.00    0.10     -  pidstat
```

从上面可以看到，大量占用cpu使用率的进程是`stress-ng`命令



##### 场景二：io密集型进程

首先在第一个终端执行命令：

```sh
$ stress-ng --hdd 2 --timeout 600
```

- `--hdd 2`：表示使用两个`worker`不断的读写临时文件
- `--timeout`：表示持续运行600s



接着，在第二个终端运行：

```sh
$ watch -d uptime
..., load average: 2.79, 1.80, 1.16
```

可以看到，平均负载不断上升，最终文档在2.7~2.8之间

最后，在第三个终端运行：

```sh
$ mpstat -P ALL 5 1
Linux 4.4.0-159-generic (ubuntu-xenial) 	08/25/2019 	_x86_64_	(2 CPU)

09:30:45 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:30:50 AM  all    0.11    0.00    5.09   41.86    0.00    0.11    0.00    0.00    0.00   52.82
09:30:50 AM    0    0.21    0.00    1.47   21.85    0.00    0.21    0.00    0.00    0.00   76.26
09:30:50 AM    1    0.23    0.00    9.09   63.87    0.00    0.23    0.00    0.00    0.00   26.57

Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all    0.11    0.00    5.09   41.86    0.00    0.11    0.00    0.00    0.00   52.82
Average:       0    0.21    0.00    1.47   21.85    0.00    0.21    0.00    0.00    0.00   76.26
Average:       1    0.23    0.00    9.09   63.87    0.00    0.23    0.00    0.00    0.00   26.57
```

> %iowait: the percentage of time that the CPU or CPUs were idle during which the system had an outstanding disk I/O request.
>
> %idle: the percentage of time that the CPU or CPUs were idle and the system did not have an outstanding disk I/O request.

从输出结果我们可以看到，尽管系统的平均负载很高，但是cpu使用率并不高，而iowait却很高，可以判断是由于io密集导致的系统负载过高。



接下来，我们使用`pidstat`来查找出导致过载的进程：

```sh
$ pidstat -d # -d输出io统计信息
Linux 4.4.0-159-generic (ubuntu-xenial) 	08/25/2019 	_x86_64_	(2 CPU)

09:36:31 AM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
...
09:36:31 AM     0      6564      0.01      0.00      0.00       0  stress-ng
09:36:31 AM     0      6565   4394.88   8064.26      0.00   38334  stress-ng-hdd
09:36:31 AM     0      6566   4486.84   6272.20      0.00   39617  stress-ng-hdd
...
```



##### 场景三：大量进程场景

首先在第一个终端运行：

```sh
$ stress-ng -c 8 --timeout 600 # -c 8 表示开启8个进程
```



然后在第二个终端运行：

```sh
$ watch -d uptime
..., load average: 8.03, 5.47, 1.72
```

可以观察到，经过一段时间之后，系统负载上升到8.0左右，远远高于正常系统负载，我们使用`pidstat`看一下进程情况：

```sh
$ pidstat -u 3 1
Linux 4.4.0-159-generic (ubuntu-xenial) 	08/25/2019 	_x86_64_	(2 CPU)

10:11:03 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
10:11:07 AM     0      7489   24.75    0.00    0.00   24.75     0  stress-ng-cpu
10:11:07 AM     0      7490   24.75    0.00    0.00   24.75     0  stress-ng-cpu
10:11:07 AM     0      7491   24.75    0.00    0.00   24.75     1  stress-ng-cpu
10:11:07 AM     0      7492   24.75    0.00    0.00   24.75     1  stress-ng-cpu
10:11:07 AM     0      7493   24.75    0.00    0.00   24.75     0  stress-ng-cpu
10:11:07 AM     0      7494   24.75    0.00    0.00   24.75     1  stress-ng-cpu
10:11:07 AM     0      7495   25.08    0.00    0.00   25.08     0  stress-ng-cpu
10:11:07 AM     0      7496   25.08    0.00    0.00   25.08     1  stress-ng-cpu

Average:      UID       PID    %usr %system  %guest    %CPU   CPU  Command
Average:        0      7489   24.75    0.00    0.00   24.75     -  stress-ng-cpu
Average:        0      7490   24.75    0.00    0.00   24.75     -  stress-ng-cpu
Average:        0      7491   24.75    0.00    0.00   24.75     -  stress-ng-cpu
Average:        0      7492   24.75    0.00    0.00   24.75     -  stress-ng-cpu
Average:        0      7493   24.75    0.00    0.00   24.75     -  stress-ng-cpu
Average:        0      7494   24.75    0.00    0.00   24.75     -  stress-ng-cpu
Average:        0      7495   25.08    0.00    0.00   25.08     -  stress-ng-cpu
Average:        0      7496   25.08    0.00    0.00   25.08     -  stress-ng-cpu
```

我们看到有8个进程在争夺系统的两个cpu，平均每个进程占用25%的cpu使用率。



### 总结

平均负载提供了一个快速查看系统整体性能的手段，反应了系统整体的负载情况，但是只看平均负载本身并不能直接发现系统瓶颈。

- 平均负载高有可能是cpu密集型进程导致的
- 平均负载高不一定代表cpu使用率高，有可能是I/O繁忙导致的
- 当发现平均负载过高时，可以使用`mpstat`和`pidstat`等工具辅助分析



### 参考

- [极客时间专栏：Linux性能优化实践](<https://time.geekbang.org/column/intro/140>)