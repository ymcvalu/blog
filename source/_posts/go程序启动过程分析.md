---
title: go程序启动过程分析
date: 2018-01-07 20:05:24
tags:
	- go
---



事实上，编译好的可执⾏⽂件真正执⾏时并⾮我们所写的 main.main 函数，因为编译器

总是会插⼊⼀段引导代码，完成诸如命令⾏参数、运⾏时初始化等⼯作，然后才会进⼊⽤

户逻辑。 

程序的入口因平台而异：

```sh
rt0_android_arm.s rt0_dragonfly_amd64.s rt0_linux_amd64.s ...
rt0_darwin_386.s rt0_freebsd_386.s rt0_linux_arm.s ...
rt0_darwin_amd64.s rt0_freebsd_amd64.s rt0_linux_arm64.s ...
```

rt0_linux_amd64.s:

```assembly
TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
   LEAQ   8(SP), SI ; argv
   MOVQ   0(SP), DI ; argc
   MOVQ   $main(SB), AX		;move address of main to ax
   JMP    AX
   
   TEXT main(SB),NOSPLIT,$-8
   MOVQ   $runtime·rt0_go(SB), AX	;跳转到runtime.rt0.go执行
   JMP    AX
```

asm_amd64.s:

```assembly
TEXT runtime·rt0_go(SB),NOSPLIT,$0
	// copy arguments forward on an even stack
	MOVQ	DI, AX		// argc
	MOVQ	SI, BX		// argv
	SUBQ	$(4*8+7), SP		// 2args 2auto
	ANDQ	$~15, SP
	MOVQ	AX, 16(SP)
	MOVQ	BX, 24(SP)
   ..
ok:
	; set the per-goroutine and per-mach "registers"
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX	;将g0的地址保存到CX
	MOVQ	CX, g(BX)	;设置 g(BX)为g0
	LEAQ	runtime·m0(SB), AX	

	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)	;设置m.g0
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)	;设置g.m
    ...
	;调用初始化函数
	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB)		;
	CALL	runtime·osinit(SB)		;
	CALL	runtime·schedinit(SB)	;

	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	PUSHQ	$0			// arg size
	;创建一个新的goroutine并加入到等待队列，该goroutine执行runtime.mainPC所指向的函数
	CALL	runtime·newproc(SB)
	POPQ	AX
	POPQ	AX

	;该函数内部会调用调度程序，从而调度到刚刚创建的goroutine执行
	CALL	runtime·mstart(SB)

	MOVL	$0xf1, 0xf1  // crash
	RET

;声明全局的变量mainPC为runtime.main函数的地址，该变量为read only
DATA	runtime·mainPC+0(SB)/8,$runtime·main(SB)	
GLOBL	runtime·mainPC(SB),RODATA,$8
```



runtime1.go:

```go
func args(c int32, v **byte) {
	argc = c
	argv = v
	sysargs(c, v)
}
func sysargs(argc int32, argv **byte) {
}
```

os_windows.go:

```go
func osinit() {
    ...
	ncpu = getproccount()	//获取cpu核数
    ...
}
```



proc.go:
```go
    // The bootstrap sequence is:
    //
    //	call osinit
    //	call schedinit
    //	make & queue new G
    //	call runtime·mstart
    //
    // The new G calls runtime·main.
    func schedinit() {
    	// raceinit must be the first call to race detector.
    	// In particular, it must be done before mallocinit below calls racemapshadow.
    	_g_ := getg()	//获取的是g0
    	if raceenabled {
    		_g_.racectx, raceprocctx0 = raceinit()
    	}
    	//最大系统线程数量限制
    	sched.maxmcount = 10000
    
    	tracebackinit()
    	moduledataverify()
      	//栈、内存分配器和调度器的相关初始化
    	stackinit()
    	mallocinit()
    	mcommoninit(_g_.m)
      
    	alginit()       // maps must not be used before this call
    	modulesinit()   // provides activeModules
    	typelinksinit() // uses maps, activeModules
    	itabsinit()     // uses activeModules
    
    	msigsave(_g_.m)
    	initSigmask = _g_.m.sigmask
    
      	//处理命令行参数和环境变量
    	goargs()
    	goenvs()
      	
      	//处理 GODEBUG、GOTRACEBACK 调试相关的环境变量设置
    	parsedebugvars()
      
      	//垃圾回收器初始化
    	gcinit()
    
    	sched.lastpoll = uint64(nanotime())
      	//通过 CPU核心数和GOMAXPROCS环境变量确定P的数量，P用于调度g到m上
    	procs := ncpu
    	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
    		procs = n
    	}
    	if procs > _MaxGomaxprocs {
    		procs = _MaxGomaxprocs
    	}
    	if procresize(procs) != nil {
    		throw("unknown runnable goroutine during bootstrap")
    	}
    
    	if buildVersion == "" {
    		// Condition should never trigger. This code just serves
    		// to ensure runtime·buildVersion is kept in the resulting binary.
    		buildVersion = "unknown"
    	}
    }
```

```go
    // Called to start an M.
    //go:nosplit
    func mstart() {
    	....
    	mstart1()
    }
```
```go
    func mstart1() {
         ...
      	//调度goroutine
    	schedule()
    }

```
```go
// The main goroutine.
func main() {
	g := getg()	//当前获取的g是刚刚在rt0_go内创建的goroutine

	// Racectx of m0->g0 is used only as the parent of the main goroutine.
	// It must not be used for anything else.
	g.m.g0.racectx = 0

	// Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
	// Using decimal instead of binary GB and MB because
	// they look nicer in the stack overflow failure message.
  	//执行栈最大限制：1GB on 64-bit，250MB on 32-bit
	if sys.PtrSize == 8 {	//64-bit下指针长度是8个字节
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}

	// Allow newproc to start new Ms.
	mainStarted = true

  	//启动系统后台监控（定期垃圾回收以及并发任务的调度等）
	systemstack(func() {
		newm(sysmon, nil)
	})

	// Lock the main goroutine onto this, the main OS thread,
	// during initialization. Most programs won't care, but a few
	// do require certain calls to be made by the main thread.
	// Those can arrange for main.main to run in the main thread
	// by calling runtime.LockOSThread during initialization
	// to preserve the lock.
	lockOSThread()

	if g.m != &m0 {
		throw("runtime.main not on m0")
	}

  	//执行runtime包内的所有初始化函数 init
	runtime_init() // must be before defer
	if nanotime() == 0 {
		throw("nanotime returning zero")
	}

	// Defer unlock so that runtime.Goexit during init does the unlock too.
	needUnlock := true
	defer func() {
		if needUnlock {
			unlockOSThread()
		}
	}()

	// Record when the world started. Must be after runtime_init
	// because nanotime on some platforms depends on startNano.
	runtimeInitTime = nanotime()

  	//启动垃圾回收器的后台操作
	gcenable()

	main_init_done = make(chan bool)

  	//执行用户包（包括标准库）的初始化函数 init，程序所有的包的init函数都会在这个函数内被全部执行
	fn := main_init // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
	fn()
	close(main_init_done
	needUnlock = false
	unlockOSThread()

  	//执行用户逻辑入口 main.main 函数
	fn = main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
	fn()
   ...
  	//执行结束，程序正常退出
	exit(0)
}
```


### 总结

• 所有 init 函数都在同⼀个 goroutine 内执⾏

• 所有 init 函数结束后才会执⾏ main.main 函数 

### 参考

- 雨痕的 Go 1.5源码剖析