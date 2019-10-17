---
title: golang中的defer实现
date: 2018-04-15 23:50:04
tags:
	- go
	- defer
---



`defer`是go独有的关键字，可以说是go的一大特色。

被`defer`修饰的函数调用，会在函数返回时被执行，因此常常被用于执行锁或者资源释放等。

在每次获得资源时，都紧接`defer`语句对其进行释放，可以防止在后续的操作中忘记释放资源。

在享受其便捷之后，你有没有想过defer机制是如何实现的呢？

首先编写简单的main函数

```go
func main() {
	defer func() {
		fmt.Println("exit")
	}()
}
```

使用`go tool compile -N -S main.go > main.s`命令编译查看输出的汇编代码

```assembly
"".main STEXT size=96 args=0x0 locals=0x18
	TEXT	"".main(SB), $24-0
	...
	MOVL	$0, (SP)	;deferproc第一个参数0
	LEAQ	"".main.func1·f(SB), AX ;匿名函数被编译成main.func1，保存函数地址到AX
	MOVQ	AX, 8(SP) ;deferproc第二个参数为匿名函数地址
	PCDATA	$0, $0
	CALL	runtime.deferproc(SB) ;调用defer函数
	...
	CALL	runtime.deferreturn(SB) ;返回之前执行deferreturn函数
	MOVQ	16(SP), BP
	ADDQ	$24, SP
	RET
	...
```

根据输出的汇编代码，可以看到defer语句被替换成了调用`runtime.deferproc`方法，查看具体的实现，而在函数返回时执行`runtime.deferreturn`方法

首先分析`runtime.deferproc`方法

```go
// Create a new deferred function fn with siz bytes of arguments.
// The compiler turns a defer statement into a call to this.
//go:nosplit
//siz表示fn函数的参数总大小
func deferproc(siz int32, fn *funcval) { // arguments of fn follow fn
    //deferproc不允许在系统栈执行
	if getg().m.curg != getg() {
		// go code on the system stack can't defer
		throw("defer on system stack")
	}

	// the arguments of fn are in a perilous state. The stack map
	// for deferproc does not describe them. So we can't let garbage
	// collection or stack copying trigger until we've copied them out
	// to somewhere safe. The memmove below does that.
	// Until the copy completes, we can only call nosplit routines.
	sp := getcallersp(unsafe.Pointer(&siz))
    //fn的参数紧跟在fn之后,因此通过简单的指针运算可以获取fn的参数起始地址
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
    // 获取当前函数的调用者的PC
	callerpc := getcallerpc()
	//获取一个_defer
	d := newdefer(siz)
	if d._panic != nil {
		throw("deferproc: d.panic != nil after newdefer")
	}
	d.fn = fn
	d.pc = callerpc
    // 保存当前的SP
	d.sp = sp
	switch siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
	default:
        //deferArgs:分配_defer时,连同参数存储空间一起分配,参数紧跟_defer之后存储,该函数进行指针运算,返回参数的起始地址：
        //拷贝参数,因此在执行defer语句语义之前,需要先准备好接收者和参数
		memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
	}

	// deferproc returns 0 normally.
	// a deferred func that stops a panic
	// makes the deferproc return 1.
	// the code the compiler generates always
	// checks the return value and jumps to the
	// end of the function if deferproc returns != 0.
	return0()
	// No code can go here - the C return register has
	// been set and must not be clobbered.
}
```

具体的逻辑已经很清楚了，这里要说明的是：`runtime.deferproc`接受两个参数，需要延时执行的函数fn的地址以及fn的参数总大小，而fn的参数需要紧跟着分配在`&fn`后面。

在函数中我们看到了`_defer`这个类型，该类型是实现`defer`机制的关键，其声明如下：

```go
// A _defer holds an entry on the list of deferred calls.
// If you add a field here, add code to clear it in freedefer.
type _defer struct {
	siz     int32	//参数size
	started bool	//是否执行过
	sp      uintptr // sp at time of defer
	pc      uintptr
	fn      *funcval //需要延时执行的函数地址
	_panic  *_panic // panic that is running defer
	link    *_defer //每个goroutine中的_defer以链表组织
}
```

在`runtime.newdefer`方法中，会获取一个_defer结构，**并将其加入当前goroutine的` _defer`队列头部**。

接着看一下`runtime.deferreturn`方法实现

```go
// Run a deferred function if there is one.
// The compiler inserts a call to this at the end of any
// function which calls defer.
// If there is a deferred function, this will call runtime·jmpdefer,
// which will jump to the deferred function such that it appears
// to have been called by the caller of deferreturn at the point
// just before deferreturn was called. The effect is that deferreturn
// is called again and again until there are no more deferred functions.
// Cannot split the stack because we reuse the caller's frame to
// call the deferred function.

// The single argument isn't actually used - it just has its address
// taken so it can be matched against pending defers.
//go:nosplit
func deferreturn(arg0 uintptr) { //这边的arg0只是为了获取当前的sp
	gp := getg()
	d := gp._defer	//获取_defer链表头部
    //如果没有_defer,则返回,详见上面注释
	if d == nil {
		return
	}
    // 当前goroutine的所有的_defer通过链表连接
    // 这里通过比较SP，确保只执行当前函数的_defer
	sp := getcallersp(unsafe.Pointer(&arg0))
	if d.sp != sp {
		return
	}

	// Moving arguments around.
	//
	// Everything called after this point must be recursively
	// nosplit because the garbage collector won't know the form
	// of the arguments until the jmpdefer can flip the PC over to
	// fn.
    //拷贝参数到sp中
	switch d.siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	d.fn = nil
	gp._defer = d.link //从链表中移除
	freedefer(d) //释放当前_defer
    //call runtime·jmpdefer,
    // which will jump to the deferred function such that it appears
    // to have been called by the caller of deferreturn at the point
    // just before deferreturn was called. The effect is that deferreturn
    // is called again and again until there are no more deferred fns.
    //执行fn,并修改pc为 `CALL	runtime.deferreturn(SB)`,下一条指令再次进入该函数,如果gp.defer为nil或者sp不一致,则返回,否则继续执行defer
    //每次添加defer时,总是添加到head,处理时则是从head开始处理,因此defer的处理顺序是FILO
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```

至此，defer语句的运行机制分析完成了，主要理了大概的执行流程，其中还有一些细节由于篇幅有限并没有细说，可以自行分析。

go中还有一个比较独特的地方，如果程序发生异常，会保证先执行所有defer声明的延时函数，然后才退出程序；而我们可以在延时函数中获取到当前整个堆栈的信息，比如说：

```
函数A执行defer语句，调用函数B
函数B函数B发生panic
执行函数A的延时函数，这时候是可以获取到函数B的栈帧数据的
```

按照上面的执行流程，在执行函数A的延时函数时，实际上这时候函数B的栈帧还没有弹出，神奇吧？这是因为执行panic时，就会去遍历当前goroutine的`_defer`链表，并依次执行这些延时函数，而不是返回函数A之后再执行函数A的延时函数。

实际的执行流程是这样的：

```
函数A执行defer语句，调用函数B
函数B函数B发生panic
在panic内部，遍历_defer链表，并依次执行延时函数
如果有延时函数执行了recover，则在延时函数返回后，直接跳转到_defer.pc，而不会执行后续的延时函数
```

```go
// 内置函数panic的实现
func gopanic(e interface{}) {
	gp := getg()  // 当前panic的g
	
    // 在系统栈panic
    if gp.m.curg != gp {
		print("panic: ")
		printany(e)
		print("\n")
		throw("panic on system stack") // throw是不可恢复的，直接终止进程
	}
    
    // 在内存分配过程中panic
	if gp.m.mallocing != 0 {
		print("panic: ")
		printany(e)
		print("\n")
		throw("panic during malloc")
	}
    
	if gp.m.preemptoff != "" {
		print("panic: ")
		printany(e)
		print("\n")
		print("preempt off reason: ")
		print(gp.m.preemptoff)
		print("\n")
		throw("panic during preemptoff")
	}
    
	if gp.m.locks != 0 {
		print("panic: ")
		printany(e)
		print("\n")
		throw("panic holding locks")
	}

	var p _panic
	p.arg = e
	p.link = gp._panic
    // 在defer中可以通过recover获取到该_panic
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))
    // 统计
	atomic.Xadd(&runningPanicDefers, 1)

    // 依次执行当前goroutine的_defer
	for {
		d := gp._defer
		if d == nil {
			break
		}

		// If defer was started by earlier panic or Goexit (and, since we're back here, that triggered a new panic),
		// take defer off list. The earlier panic or Goexit will not continue running.
        // defer已经开始执行了，执行defer的时候又触发了panic
		if d.started {
            // 如果存在早期的panic
			if d._panic != nil {
                // 终止原来的panic
				d._panic.aborted = true
			}
			d._panic = nil
			d.fn = nil
			gp._defer = d.link
			freedefer(d)
            // 继续下一个defer
			continue
		}

		// Mark defer as started, but keep on list, so that traceback
		// can find and update the defer's argument frame if stack growth
		// or a garbage collection happens before reflectcall starts executing d.fn.
		// 标记开始执行
        d.started = true

		// Record the panic that is running the defer.
		// If there is a new panic during the deferred call, that panic
		// will find d in the list and will mark d._panic (this panic) aborted.
		// 设置defer
        d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

		p.argp = unsafe.Pointer(getargp(0))
        // 调用defer延时的函数
		reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
		p.argp = nil

		// reflectcall did not panic. Remove d.
		if gp._defer != d {
			throw("bad defer entry in panic")
		}
		d._panic = nil
		d.fn = nil
		gp._defer = d.link

		// trigger shrinkage to test stack copy. See stack_test.go:TestStackPanic
		//GC()

		pc := d.pc
		sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
		freedefer(d)
        
        // 如果在defer中recover了
		if p.recovered {
			atomic.Xadd(&runningPanicDefers, -1)

			gp._panic = p.link
			// Aborted panics are marked but remain on the g.panic list.
			// Remove them from the list.
            // 移除已经aborted的panic
			for gp._panic != nil && gp._panic.aborted {
				gp._panic = gp._panic.link
			}
			if gp._panic == nil { // must be done with signal
				gp.sig = 0
			}
			// Pass information about recovering frame to recovery.
			gp.sigcode0 = uintptr(sp)
			gp.sigcode1 = pc
			// 调用recovery，恢复执行
            mcall(recovery)
			throw("recovery failed") // mcall should not return
		}
	}

	// ran out of deferred calls - old-school panic now
	// Because it is unsafe to call arbitrary user code after freezing
	// the world, we call preprintpanics to invoke all necessary Error
	// and String methods to prepare the panic strings before startpanic.
	preprintpanics(gp._panic)

	fatalpanic(gp._panic) // should not return
	*(*int)(nil) = 0      // not reached
}
```



最后，`defer`函数虽然方便，但是需要有额外的运行开销，在使用时需要进行取舍，尤其是具有多个参数的时候，会发生多次内存拷贝：

```
runtime.deferproc执行之前：移动到栈中
runtime.deferproc执行过程中，拷贝_defer之后
runtime.deferreturn执行时，移动到栈中
```

update：go1.13对defer进行了优化，如果`_defer`没有发生逃逸，则将其分配在栈上，可以提高30%的性能。





