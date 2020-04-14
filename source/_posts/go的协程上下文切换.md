---
title: go的协程上下文切换
date: 2020-04-14 13:44:03
tags:
    - go
---
都说`go`的协程是轻量级的，上下文切换只需要切换少数寄存器即可。今天我们来看一下，协程上下文切换时需要做哪些操作。

> 本文基于go1.14.2，不涉及非协作式抢占相关的上下文切换，这块的具体实现还不太了解，但是应该不太一样。

首先，我们需要知道几种比较常见的触发协程上下文切换的场景：
- [协作式抢占](https://mcll.top/2019/05/25/%E5%8D%8F%E7%A8%8B%E6%8A%A2%E5%8D%A0/)的时候，检查抢占标志位，从而触发上下文切换
- 锁阻塞/channel阻塞/io阻塞等等，这类最后都会调用`runtime.gopark`将当前协程挂起，然后调度新的协程

协程的上下文切换，涉及到挂起协程的上下文保存，和调度协程的上下文恢复。

### 上下文
首先，`g`的上下文是保存在哪里的呢？答案是`g.sched`：
```go
type g struct {
    ...
	sched        gobuf
    ...
}

type gobuf struct {
	// The offsets of sp, pc, and g are known to (hard-coded in) libmach.
	//
	// ctxt is unusual with respect to GC: it may be a
	// heap-allocated funcval, so GC needs to track it, but it
	// needs to be set and cleared from assembly, where it's
	// difficult to have write barriers. However, ctxt is really a
	// saved, live register, and we only ever exchange it between
	// the real register and the gobuf. Hence, we treat it as a
	// root during stack scanning, which means assembly that saves
	// and restores it doesn't need write barriers. It's still
	// typed as a pointer so that any other writes from Go get
	// write barriers.
	sp   uintptr            // 栈顶指针
	pc   uintptr            // PC地址
	g    guintptr           // 关联的g
	ctxt unsafe.Pointer     // func的上下文，与闭包实现有关
	ret  sys.Uintreg
	lr   uintptr
	bp   uintptr            // 栈底指针
}
```

可以看到，需要保存的有：
- 与栈相关的`SP`和`BP`寄存器
- `PC`寄存器
- 用于保存函数闭包的上下文信息，也就是`DX`寄存器

而对于其他通用寄存器，因为`go`的函数调用规约，参数和返回值是通过栈进行传递的，并且总是在函数调用的时候触发协程切换，并不需要保存

### 上下文保存
上下文保存的逻辑，可以分为两种情况，下面分别来看一下。
##### 由协作式抢占触发
这种情况，是在`runtime·morestack`方法中保存上下文，代码在`runtime/asm_amd64.s`：
```
TEXT runtime·morestack(SB),NOSPLIT,$0-0
	// Cannot grow scheduler stack (m->g0).
	get_tls(CX)          ; m.tls
	MOVQ	g(CX), BX    ; 保存当前g到BX
	MOVQ	g_m(BX), BX  ; 保存当前m到BX
	MOVQ	m_g0(BX), SI ; 保存g0到SI
	CMPQ	g(CX), SI    ; g0上不允许触发morestack
	JNE	3(PC)
	CALL	runtime·badmorestackg0(SB)
	CALL	runtime·abort(SB)

	// Cannot grow signal stack (m->gsignal).
	MOVQ	m_gsignal(BX), SI
	CMPQ	g(CX), SI
	JNE	3(PC)
	CALL	runtime·badmorestackgsignal(SB)
	CALL	runtime·abort(SB)

	// Called from f.
	// Set m->morebuf to f's caller.
	NOP	SP	// tell vet SP changed - stop checking offsets
	MOVQ	8(SP), AX	// f's caller's PC
	MOVQ	AX, (m_morebuf+gobuf_pc)(BX)
	LEAQ	16(SP), AX	// f's caller's SP
	MOVQ	AX, (m_morebuf+gobuf_sp)(BX)
	get_tls(CX)
	MOVQ	g(CX), SI
	MOVQ	SI, (m_morebuf+gobuf_g)(BX)

	// Set g->sched to context in f.
    // 下面是保存g的上下文到g.sched中
	MOVQ	0(SP), AX                    ; morestack是nosplit的，返回地址就是要保存的PC
	MOVQ	AX, (g_sched+gobuf_pc)(SI)   ; 保存PC
	MOVQ	SI, (g_sched+gobuf_g)(SI)    ; 保存当前g
	LEAQ	8(SP), AX                    ; SP寄存器，这里扣掉PC占用的8字节
	MOVQ	AX, (g_sched+gobuf_sp)(SI)   ; 保存SP寄存器
	MOVQ	BP, (g_sched+gobuf_bp)(SI)   ; 保存BP寄存器
	MOVQ	DX, (g_sched+gobuf_ctxt)(SI) ; DX保存函数的闭包上下文

	// Call newstack on m->g0's stack.
	MOVQ	m_g0(BX), BX
	MOVQ	BX, g(CX)
	MOVQ	(g_sched+gobuf_sp)(BX), SP
	CALL	runtime·newstack(SB)
	CALL	runtime·abort(SB)	// crash if newstack returns
	RET
```

##### 由阻塞操作触发
`go`中的锁/channel/io操作等的阻塞，最终是由`runtime.gopark`实现的：
```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	if reason != waitReasonSleep {
		checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
	}
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	// can't do anything that might move the G between Ms here.
	mcall(park_m)
}
```
在`gopark`中，会通过`mcall`切换到`g0`栈上执行`park_m`，而`mcall`方法本身就会保存上下文：
```
TEXT runtime·mcall(SB), NOSPLIT, $0-8
	MOVQ	fn+0(FP), DI        ; mcall的参数是一个函数

	get_tls(CX)                 ; m.tls
	MOVQ	g(CX), AX	        ; 保存当前g到AX
	MOVQ	0(SP), BX	        ; 保存PC到BX
	MOVQ	BX, (g_sched+gobuf_pc)(AX)  ; 设置g.sched.pc
	LEAQ	fn+0(FP), BX	            ; SP寄存器
	MOVQ	BX, (g_sched+gobuf_sp)(AX)  ; 设置g.sched.sp
	MOVQ	AX, (g_sched+gobuf_g)(AX)   ; 设置g.sched.g
	MOVQ	BP, (g_sched+gobuf_bp)(AX)  ; BP寄存器

	// switch to m->g0 & its stack, call fn
	MOVQ	g(CX), BX
	MOVQ	g_m(BX), BX     ; 获取当前m
	MOVQ	m_g0(BX), SI    ; m.g0
	CMPQ	SI, AX	        ; 禁止在g0调用mcall
	JNE	3(PC)
	MOVQ	$runtime·badmcall(SB), AX
	JMP	AX
	MOVQ	SI, g(CX)	    ; 设置当前g为g0
	MOVQ	(g_sched+gobuf_sp)(SI), SP	; 恢复g0的SP
	PUSHQ	AX
	MOVQ	DI, DX
	MOVQ	0(DI), DI
	CALL	DI  ; 调用传入mcall的方法
	POPQ	AX
	MOVQ	$runtime·badmcall2(SB), AX
	JMP	AX
	RET
```
可以看到，`mcall`中就会保存当前`g`的上下文，然后切换到`g0`栈执行。

### 上下文恢复
当`runtime`通过调度逻辑确定下一个要运行的`g`之后，通过`runtime.execute`方法来开始运行这个`g`：
```go
func execute(gp *g, inheritTime bool) {
	_g_ := getg()

	_g_.m.curg = gp
	gp.m = _g_.m
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		_g_.m.p.ptr().schedtick++
	}

	// Check whether the profiler needs to be turned on or off.
	hz := sched.profilehz
	if _g_.m.profilehz != hz {
		setThreadCPUProfiler(hz)
	}

	if trace.enabled {
		// GoSysExit has to happen when we have a P, but before GoStart.
		// So we emit it here.
		if gp.syscallsp != 0 && gp.sysblocktraced {
			traceGoSysExit(gp.sysexitticks)
		}
		traceGoStart()
	}

	gogo(&gp.sched)
}
```
可以看到，`execute`在绑定了`g`和`m`，更新`g`的状态之后，通过`gogo`方法开始运行`g`：
```
TEXT runtime·gogo(SB), NOSPLIT, $16-8
	MOVQ	buf+0(FP), BX		; g.sched
	MOVQ	gobuf_g(BX), DX
	MOVQ	0(DX), CX		    ; make sure g != nil
	get_tls(CX)
	MOVQ	DX, g(CX)
	MOVQ	gobuf_sp(BX), SP	; restore SP
	MOVQ	gobuf_ret(BX), AX   
	MOVQ	gobuf_ctxt(BX), DX  ; restore DX
	MOVQ	gobuf_bp(BX), BP    ; restore BP
	MOVQ	$0, gobuf_sp(BX)	; clear to help garbage collector
	MOVQ	$0, gobuf_ret(BX)
	MOVQ	$0, gobuf_ctxt(BX)
	MOVQ	$0, gobuf_bp(BX)
	MOVQ	gobuf_pc(BX), BX    ; PC
	JMP	BX                      ; JMP PC
```
`gogo`的实现很简单，就是恢复上下文，然后通过`JMP`指令跳转到指定的`PC`开始执行。