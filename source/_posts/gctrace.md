---
title: gctrace
date: 2019-08-23 10:56:51
tags:
	- go
---

在优化`go`程序时，内存优化是其中一项很重要的内容，减轻`gc`的压力，能够极大的优化我们的程序运行效率。

今天先来看一下两个与`gc`相关的环境变量：`gctrace`和`GOGC`

### gctrace

`gctrace`本身是[`GODEBUG`](<https://golang.org/pkg/runtime/#hdr-Environment_Variables>)这个环境变量中的一个选项，用来开启`gc`日志的。每次当完成一次`gc`扫描时，就会打印出本次`gc`的相关信息，我们可以用来监控程序的内存情况

```sh
$ GODEBUG=gctrace=1 GO_BIN # 开启gctrace
gc 1 @1.569s 0%: 0.28+2.2+4.6 ms clock, 1.1+0/0.96/4.6+18 ms cpu, 4->4->1 MB, 5 MB goal, 4 P
gc 2 @3.124s 0%: 0.026+1.4+0.058 ms clock, 0.10+1.3/0.033/1.5+0.23 ms cpu, 4->4->1 MB, 5 MB goal, 4 P
gc 3 @5.326s 0%: 0.58+0.53+0.12 ms clock, 2.3+0.36/0.44/0.30+0.48 ms cpu, 4->4->1 MB, 5 MB goal, 4 P
...
scvg1: inuse: 4, idle: 58, sys: 63, released: 0, consumed: 63 (MB)
...
```

每一行对应一次`gc`，具体的输出格式如下：

```
gc # @#s #%: #+#+# ms clock, #+#/#/#+# ms cpu, #->#-># MB, # MB goal, # P [(forced)]
```

- `gc #`：第几轮gc，从1开始递增

- `@#s`：程序总的运行时间，单位`s`
- `#%`：从程序开始到现在运行`gc`的时间占比
- `#+#+# ms clock`：对应 <font color=red>第一次`STW`，终止`SWEEP`、开启写屏障</font> +  <font color=red>并发`Mark`和`Scan`</font> + <font color=red>第二次`STW`，结束`Mark`</font> 这三个阶段`wall-clock`的耗时，单位为`ms`
- `#+#/#/#+# ms cpu`：对应 <font color=red>第一次`STW`</font> + <font color=red>并发标记：`Assist Time `</font>/<font color=red> 并发标记：`Background GC time`</font> /<font color=red> 并发标记：`Idle GC time`</font> + <font color=red>第二次`STW`结束`Mark`</font> 这几个阶段的`cpu`时间，单位`ms`
- `#->#-># MB `：分别对应`gc`开始时的堆大小、`gc`结束时的堆大小以及`live heap`（`gc mark`阶段标记为黑色的内存总量）的大小

- `# MB goal`：目标`heap size`
- `# P`：使用的`P`数量
- `(forced)`：如果调用`runtime.GC()`强制触发`gc`

除了输出`gc`的信息，当`runtime`向操作系统归还内存时，也会打印出信息，比如上面的：

```
scvg1: inuse: 4, idle: 58, sys: 63, released: 0, consumed: 63 (MB)
```

- `scvg#`：第几次归还，从1开始计数
- `inuse: #`：正在使用或部分被使用的`spans`的内存大小，单位`MB`
- `idle: #`：空闲等待归还给操作系统的`spans`的内存大小，单位`MB`；
- `sys: #`：从操作系统映射的内存大小，实际上是`gc`堆可访问的内存虚拟地址空间，单位`MB`
- `released: #`：本次归还给操作系统的内存大小，单位`MB`
- `consumed: #`：从操作系统分配的内存大小，等于`sys - released`



### GOGC

根据`runtime/mgc.go`中的注释：

>  Next GC is after we've allocated an extra amount of memory proportional to
>  the amount already in use. The proportion is controlled by GOGC environment variable
>  (100 by default). If GOGC=100 and we're using 4M, we'll GC again when we get to 8M
>  (this mark is tracked in next_gc variable). This keeps the GC cost in linear
>  proportion to the allocation cost. Adjusting GOGC just changes the linear constant
>  (and also the amount of extra memory used).

在`go1.5`之前，运行`gc mark`阶段会`stop the world`， 能够根据`next_gc`变量（也就是**goal heap size**，可以直接通过`GOGC`变量调整）精确地控制堆内存的增长：

![](/img/gc_stw.jpg)

但是`go1.5`之后，`gc mark`可以跟用户协程并发运行，因此在`gc`执行过程中仍然会有新的内存被分配，因此`gc`的触发点需要相对`next_gc`提前：

![](/img/gc_bg.jpg)

如上图所示，`Hm(n-1)`表示上一次`gc`结束后的堆大小,而`Hg`是`next_gc`，而我们在`Ht`触发`gc`，因为gc过程中可能会有新的内存分配，当`gc`结束时，当前的堆大小为`Ha`。`go`的`gc`实现，需要提供一种动态调整的机制，根据内存分配情况调整`Ht`的值，使得`Ha`能够与`Hg`尽量接近。



总体来说，我们可以通过设置`GOGC`的值来调整`gc`的触发阈值：

- **当小于零或者等于`off`时，将会关闭`gc`**
- 设置较大的值：减少`gc`触发，但是会增加内存占用
- 设置偏小的值：频繁触发`gc`

