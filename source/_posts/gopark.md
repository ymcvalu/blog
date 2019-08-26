---
title: gopark
date: 2019-08-26 22:16:53
tags:
	- go - runtime
---

 在 go 的 runtime 包中，经常能够看到`gopark`方法，今天来看一下该函数的实现和作用。



### 实现

该方法的源码在`runtime/proc.go`：

```go
// Puts the current goroutine into a waiting state and calls unlockf.
// If unlockf returns false, the goroutine is resumed.
// unlockf must not access this G's stack, as it may be moved between
// the call to gopark and the call to unlockf.
// Reason explains why the goroutine has been parked.
// It is displayed in stack traces and heap dumps.
// Reasons should be unique and descriptive.
// Do not re-use reasons, add new ones.

// 根据上面的注释，该方法的作用是如果unlockf返回true，则将当前协程挂起，否则继续执行
// 
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	if reason != waitReasonSleep {
		checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
	}
	mp := acquirem() // 这里获取当前的m并加锁
	gp := mp.curg // gp即为当前运行的协程
	status := readgstatus(gp)
    // 检查gp的状态
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
    // 将 lock 和 unlockf 分别保存到 mp 的 waitlock 和 waitunlockf
    // unlockf 需要接收两个参数，一个是 gp，另一个就是参数 lock
	mp.waitlock = lock
	mp.waitunlockf = *(*unsafe.Pointer)(unsafe.Pointer(&unlockf))
    // 等待的原因
	gp.waitreason = reason
    // trace相关
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp) // 释放当前m
	// can't do anything that might move the G between Ms here.
    // 这里mcall是一个由汇编实现的函数，接收一个函数作为参数，然后切换到 m 的 g0 栈去执行这个函数
	mcall(park_m) // 切换到 m 的 g0 栈执行 park_m 方法
}
```



接下来，我们要先来看一下 `park_m` 这个方法的实现，该方法同样位于`runtime/proc.go`：

```go
// park continuation on g0.
// mcall 方法会切换到 m 的 g0 运行，然后把原先的 g 作为参数传递给该方法
func park_m(gp *g) {
	_g_ := getg() // 获取当前运行的 g，这时候返回的是 g0
	
    // 如果开启了trace，则发送trace事件
	if trace.enabled {
		traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
	}

    // cas更新gp的状态
	casgstatus(gp, _Grunning, _Gwaiting)
    // 这里将 gp 与 m 分离
	dropg()
    
	// 这里的 waitunlockf 就是上面 gopark 方法的参数 unlockf
    // 在 gopark 中分别将其两个参数 unlockf 和 lock 保存到 m 的 waitunlockf 和 waitlock字段中
	if _g_.m.waitunlockf != nil { // 如果 gopark 方法接收的 unlockf 不为空
		fn := *(*func(*g, unsafe.Pointer) bool)(unsafe.Pointer(&_g_.m.waitunlockf))
		// waitlock 就是 gopark 方法的另一个参数 lock
        ok := fn(gp, _g_.m.waitlock)
        // 及时清空 m 的两个这两个字段
		_g_.m.waitunlockf = nil 
		_g_.m.waitlock = nil
        // 如果 unlockf 返回了 false，那么不要挂起，继续执行
		if !ok {
			if trace.enabled {
				traceGoUnpark(gp, 2)
			}
            // 状态切换为 _Grunnable
			casgstatus(gp, _Gwaiting, _Grunnable)
            // 重新绑定 gp 到当前 m，并恢复执行，该方法不会返回
			execute(gp, true) // Schedule it back, never returns.
		}
	}
    // unlockf 为 nil 或者返回 true，则挂起 gp，并重新调度新的 g 到当前 m
	schedule() // 协程调度，该方法也不会返回
}
```

根据上面的 `park_m`的分析，我们知道 `mcall` 方法需要保存 `gp` 的上下文信息，并切换到 `g0` 栈，以 `gp` 作为参数调用 `park_m`。



在看 `mcall` 方法的实现之前，我们先来看一下 `g` 中的 几个字段，`g`的实现在`runtime/runtime2.go`：

```go
type g struct {
	stack       stack   // 当前g的栈信息
 	...
	m              *m      // 当前g绑定的m
	sched          gobuf   // 保存g的运行上下文信息，下次调度时从哪里开始执行
	... 
}

type stack struct {
	lo uintptr // 栈底
	hi uintptr // 栈顶
}

type gobuf struct {
	sp   uintptr 	 // 栈顶
	pc   uintptr	 // pc，恢复点
	g    guintptr    // 对应的g指针值
	ctxt unsafe.Pointer
	ret  sys.Uintreg
	lr   uintptr
	bp   uintptr     // 栈底
}
```



接下来我们来看 `mcall` 在 `amd64` 体系下的实现，源代码位于 `runtime/asm_amd64.s`：

```assembly
// func mcall(fn func(*g))
// Switch to m->g0's stack, call fn(g).
// Fn must never return. It should gogo(&g->sched)
// to keep running g.
// 
// 根据注释，mcall 切换到 g0 栈调用 fn(g), fn 不能返回
TEXT runtime·mcall(SB), NOSPLIT, $0-8 // 栈帧大小为0，这种情况0(SP)保存的为返回地址
	// 关于go的函数调用栈的布局，可以参考 http://mcll.top/2019/04/29/go函数栈布局
	
	MOVQ	fn+0(FP), DI // 将mcall的参数保存到DI寄存器，实际上是一个funcval指针
	// 关于funcval的介绍，可以参考http://mcll.top/2019/03/06/go中的猴子补丁

	get_tls(CX)  // 将TLS保存到CX寄存器，TLS是一个伪寄存器，get_tls是一个宏，一直没找到其定义😓
	MOVQ	g(CX), AX	// 保存当前g到AX寄存器中
	MOVQ	0(SP), BX	// 0(SP)保存返回地址，也就是mcall调用者的PC
	MOVQ	BX, (g_sched+gobuf_pc)(AX) // 保存到g.sched.pc字段中
	LEAQ	fn+0(FP), BX	// fn+0(FP)也就是调用者的SP
	MOVQ	BX, (g_sched+gobuf_sp)(AX) // 保存到g.sched.sp字段中
	MOVQ	AX, (g_sched+gobuf_g)(AX) // 保存当前g到g.sched.g中
	MOVQ	BP, (g_sched+gobuf_bp)(AX) // 因为栈帧为0，BP就是调用者的BP，保存到g.sched.bp

	// switch to m->g0 & its stack, call fn
	MOVQ	g(CX), BX  // 将原来的g保存到BX
	MOVQ	g_m(BX), BX  // 获取g.m，还是保存到BX
	MOVQ	m_g0(BX), SI // 保存g0到SI
	CMPQ	SI, AX	// AX寄存器上面已经设置为原来的g了，禁止在g0栈上调用mcall，这里要判断一下
	JNE	3(PC) // 如果原来的g不是g0，跳转到pc+3的位置执行，也就是从这里往下第三条指令
	MOVQ	$runtime·badmcall(SB), AX // 如果在g0上调用mcall，直接panic
	JMP	AX // 跳转到 badmcall 方法，最终会panic
	MOVQ	SI, g(CX)	// 将g0设置到TLS中
	MOVQ	(g_sched+gobuf_sp)(SI), SP	// 恢复g0的SP寄存器
	PUSHQ	AX	// AX保存原来的g，入栈
	MOVQ	DI, DX // 上面说过，DI保存了要调用函数的funcval值，将其保存的DX寄存器，DX寄存器与闭包实现有关
	MOVQ	0(DI), DI // 将要调用函数的入口地址保存到DI寄存器
	CALL	DI // 调用该函数，参数就是刚刚入栈的AX中的值，也就是原来的g，该函数禁止返回
	POPQ	AX // 出栈
	MOVQ	$runtime·badmcall2(SB), AX // 如果调用函数返回了，panic
	JMP	AX 
	RET
```



### 使用

`runtime`包中有很多用到`gopark`方法的地方，这里举例几个



##### 场景一：写空 channel

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 根据go语义，向一个nil的channel会导致阻塞
    if c == nil {
     	// 一般block都是true
		if !block {
			return false
		}
        
        // 根据上面的分析，unlockf为nil会挂起当前协程
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

    ...
}
```



##### 场景二：select

`selectgo`实现了`select`语句的功能

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	...
    // selparkcommit会释放所有case的锁，并阻塞等待当前g，等待有case被触发
    gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)
    ...
}
```

