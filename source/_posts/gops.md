---
title: gops
date: 2019-08-29 15:49:02
tags:
  - go - 性能优化
---
[gops](https://github.com/google/gops): A tool to list and diagnose Go processes currently running on your system

`gops`能够列出当前系统中运行的`go`进程，并且能够帮助对指定`go`进程进行诊断，是`gopher`必备居家良品。

### 安装
```sh
$ go get -u github.com/google/gops
```

### 简单使用
###### 查看命令帮助
```sh
$ gops 
gops is a tool to list and diagnose Go processes.

Usage:
  gops <cmd> <pid|addr> ...
  gops <pid> # displays process info
  gops help  # displays this help message

Commands:
  stack      Prints the stack trace.
  gc         Runs the garbage collector and blocks until successful.
  setgc	     Sets the garbage collection target percentage.
  memstats   Prints the allocation and garbage collection stats.
  version    Prints the Go version used to build the program.
  stats      Prints runtime stats.
  trace      Runs the runtime tracer for 5 secs and launches "go tool trace".
  pprof-heap Reads the heap profile and launches "go tool pprof".
  pprof-cpu  Reads the CPU profile and launches "go tool pprof".

All commands require the agent running on the Go process.
"*" indicates the process is running the agent.
```

###### 查看当前系统的go进程
```sh
$ gops 
3348  3088  docker-containerd       go1.7.5  /usr/bin/docker-containerd
3088  1     dockerd                 go1.7.5  /usr/bin/dockerd
26504 3088  docker-proxy            go1.7.5  /usr/bin/docker-proxy
26510 3348  docker-containerd-shim  go1.7.5  /usr/bin/docker-containerd-shim
29311 18550 gops                    go1.11.2 /usr/local/bin/gops
29135 27413 gops-demo             * go1.11.2 /home/vagrant/go/src/just-for-fun/gops/gops-demo
```
上面各列的含义分别是：
```
pid ppid 二进制文件名 go编译版本 二进制文件路径
```
而在go编译版本前面的 `*` 表明当前进程包含了`agent`。

###### 查看某个进程的信息
```sh
$ gops 29573
parent PID:	27413
threads:	6
memory usage:	0.278%
cpu usage:	0.012%
username:	root
cmd+args:	./gops-demo
elapsed time:	02:49
local/remote:	:::5050 <-> :::0 (LISTEN)
local/remote:	:::8989 <-> :::0 (LISTEN)
```

### 进程诊断
为了能够使用诊断功能，我们需要在我们的程序代码中启动一个`agent`：
```go
// import "github.com/google/gops/agent"

func main(){
	err := agent.Listen(agent.Options{
		Addr:            ":5050", // agent监听地址
		ShutdownCleanup: false, // 如果true，会监听SIGINT信号，自动关闭agent并退出进程
	})
	if err != nil {
		log.Fatal(err)
	}
	defer agent.Close() // 手动关闭agent

  .....

}
```

在启动`agent`的时候，我们需要指定监听的端口。

启动了`agent`之后，我们在`gops`中就可以看到多了一个`*`：
```sh
$ gops
29135 27413 gops-demo   * go1.11.2  /home/...
```

开启了`agent`之后，我们就可以使用`gops`强大的诊断功能了
```sh
$ gops <cmd> <pid|agent-addr>
```  
> Commands:
> - stack：查看进程栈信息
> - gc：手动触发一次`gc`，并阻塞等到`gc`结束
> - setgc：相当于设置`GOGC`值
> - memstats：查看进程的内存统计信息
> - version：查看构建二进制程序的go版本
> - stats：查看runtimes的统计信息，主要是goroutine数量和thread数量
> - trace：运行5s的 runtime tracer，并启动一个`http server`用于查看trace信息
> - pprof-heap：读取heap的profile并启动`go tool pprof`
> - pprof-cpu：读取cpu之后30s的profile并启动`go tool pprof`

###### 查看内存统计
```sh
$ gops memstats 29573
alloc: 196.65KB (201368 bytes) # 当前分配的堆对象字节数，同下面的`heap-alloc`
total-alloc: 3.64MB (3819256 bytes) # 分配的堆对象字节数的累加统计
sys: 68.69MB (72022264 bytes) # 从操作系统分配的内存的字节数，是下面所有`Xsys`的累加
lookups: 0 # runtime执行的指针查找次数统计，主要用于runtime内部的debug
mallocs: 2225 # 堆对象分配次数统计
frees: 1507 # 堆对象释放次数统计
# 堆内存统计
heap-alloc: 196.65KB (201368 bytes) # 当前分配的堆对象的字节数
heap-sys: 63.56MB (66650112 bytes) # 从操作系统分配的堆内存的大小
heap-idle: 62.80MB (65847296 bytes) # 空闲的堆内存大小（bytes in idle spans）
heap-in-use: 784.00KB (802816 bytes) # 当前正在使用的堆内存大小（bytes in in-use spans）
heap-released: 0 bytes # 归还操作系统的物理内存大小
heap-objects: 718 # 当前分配的堆对象的数量
# 下面是栈内存统计
# Stacks are not considered part of the heap, but the runtime can reuse a span of heap memory for stack memory, and vice-versa.
stack-in-use: 448.00KB (458752 bytes) # bytes in stack spans  
stack-sys: 448.00KB (458752 bytes) # bytes of stack memory obtained from the OS
# 下面主要是 runtime 内部数据结构分配的内存使用统计
stack-mspan-inuse: 15.29KB (15656 bytes) 
stack-mspan-sys: 32.00KB (32768 bytes)
stack-mcache-inuse: 6.75KB (6912 bytes)
stack-mcache-sys: 16.00KB (16384 bytes)
other-sys: 1.00MB (1049066 bytes) # runtime中一些杂项数据结构的内存占用
gc-sys: 2.26MB (2371584 bytes) # 垃圾收集器元信息的内存占用
next-gc: when heap-alloc >= 4.00MB (4194304 bytes) # 下一次触发gc的时机
last-gc: 2019-08-29 16:42:26.502415917 +0800 CST # 上一次gc时间
gc-pause-total: 946.946µs # gc总的STW的时间
gc-pause: 134271 #  最后一次 gc 的STW时间，单位ns
num-gc: 9 # gc次数
enable-gc: true
debug-gc: fals # 该字段当前未使用(go1.12.5)
```
**注意：读取内存统计信息的时候会触发`STW`**：
```go
// runtime/mstas.go
func ReadMemStats(m *MemStats) {
	stopTheWorld("read mem stats")

	systemstack(func() {
		readmemstats_m(m)
	})

	startTheWorld()
}
```

###### 设置gc触发百分比
默认`gc`的触发百分比是`100%`

查看当前`gc`触发比例：
```sh
$ gops memstats 29573
...
next-gc: when heap-alloc >= 4.00MB (4194304 bytes) # 4MB是初始值
...
```
当前的触发时机是，`heap-alloc`达到4MB

更改`gc`触发比例：
```sh
$ gops setgc 29573 200
New GC percent set to 200. Previous value was 100.

$ gops memstats 29573
...
next-gc: when heap-alloc >= 8.00MB (8388608 bytes)
...
```
`next-gc`的计算公式：
```go
next_gc = heap_marked + heap_marked*uint64(gcpercent)/100
```
`heap_marked`表示在一次`gc`中，标记为存活的总对象大小，程序刚开始时初始化为`4MB`，而`gcpercent`就是我们设置的`gc`触发比例，默认为`100`。

**注意：比如`next_gc`为`8MB`，并不是真的等到总共分配了8MB的对象才触发GC**。`这个next_gc`实际上是`gc`后的`goal heap size`。现在的`gc`实现，对象标记与业务代码会并发执行，而在业务代码中还会申请分配新的对象。如果真的等到`next_gc`才触发`gc`，那么等到`gc`结束之后，当前的`heap size`可能会大于`next_gc`，因此实际上`gc`的触发会提前一点（有另一个字段`gc_trigger`来决定）。

### gops的实现
我们程序中的`agent`，会在指定端口监听`tcp`连接，通时默认在`~/.config/gops`目录下，以进程号为文件名创建一个记录监听端口的文件。
当我们通过`gops`执行命令时，首先会读取该文件，获取监听的端口，然后与`agent`建立连接，发送事先定义好的命令：
```go
const (
  // StackTrace represents a command to print stack trace.
  StackTrace = byte(0x1)

  // GC runs the garbage collector.
  GC = byte(0x2)

  // MemStats reports memory stats.
  MemStats = byte(0x3)

  // Version prints the Go version.
  Version = byte(0x4)

  // HeapProfile starts `go tool pprof` with the current memory profile.
  HeapProfile = byte(0x5)

  // CPUProfile starts `go tool pprof` with the current CPU profile
  CPUProfile = byte(0x6)

  // Stats returns Go runtime statistics such as number of goroutines, GOMAXPROCS, and NumCPU.
  Stats = byte(0x7)

  // Trace starts the Go execution tracer, waits 5 seconds and launches the trace tool.
  Trace = byte(0x8)

  // BinaryDump returns running binary file.
  BinaryDump = byte(0x9)

  // SetGCPercent sets the garbage collection target percentage.
  SetGCPercent = byte(0x10)
)
```
`agent`收到`gops`的请求后，调用`runtime`的接口，并将结果返回。
