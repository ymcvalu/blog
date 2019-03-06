---
title: go中的猴子补丁
date: 2019-03-06 10:05:07
tags:
	- go
---

### 函数值

首先查看下面代码：

```go 
func a()int {return 1}

func main() {
	fmt.Printf("%p\n", a) // 0x48f950
	fn := a
	fmt.Printf("0x%x\n",*(*uintptr)(unsafe.Pointer(&fn))) // 0x4c8680
	fmt.Printf("0x%x\n", **(**uintptr)(unsafe.Pointer(&fn))) // 0x48f950
}
```

根据上面的输出我们可以发现，函数值`fn`并没有直接持有函数`a`的地址，这是因为**`Go`的函数值可以包含一些额外的上下文信息**，这是实现闭包和绑定实例方法的关键，我们可以在[源码](https://github.com/golang/go/blob/e9d9d0befc634f6e9f906b5ef7476fbd7ebd25e3/src/runtime/runtime2.go#L75-L78)中找到点函数值类型的线索：

```go
type funcval struct {
	fn uintptr
	// variable-size, fn-specific data here
}
```

我们代码中的函数变量，实际上应该`*funcval`类型

##### 闭包实现原理

接下来，我们探究一下`golang`中闭包的实现原理，我们首先写一个简单的闭包demo，然后从编译后的汇编代码来探究其实现

```go
func main() {
	f := fn() // 这里的f是一个函数值
	f()
}

func fn() func() {
	var a = 10
	return func() {
		fmt.Println(a) // 捕获局部变量a
	}
}
```

接下来将上面代码编译成汇编：

```sh
$ go tool compile -S -N main.go > asm.s
```

下面是生成的汇编，只保留主要的内容：

```assembly
"".main STEXT size=72 args=0x0 locals=0x18
	0x0000 00000 (demo.go:7)	TEXT	"".main(SB), $24-0
	0x0024 00036 (demo.go:8)	CALL	"".fn(SB) # 调用fn函数获取
	0x0029 00041 (demo.go:8)	MOVQ	(SP), DX # 保存返回的函数值指针到DX，这里的DX是关键
	0x002d 00045 (demo.go:8)	MOVQ	DX, "".f+8(SP) # 把DX的值赋给局部变量f 
	0x0032 00050 (demo.go:9)	MOVQ	(DX), AX # 从上面funcval结构可知，(DX)为实际函数地址，也就是下面的"".fn.func1
	0x0035 00053 (demo.go:9)	CALL	AX # 调用实际的函数
	0x0040 00064 (demo.go:10)	RET

"".fn STEXT size=136 args=0x8 locals=0x28
	0x0000 00000 (demo.go:12)	TEXT	"".fn(SB), $40-8
	0x0036 00054 (demo.go:14)	LEAQ	type.noalg.struct { F uintptr; "".a int }(SB), AX # 这里表示实际funcval的类型
	0x003d 00061 (demo.go:14)	MOVQ	AX, (SP)
	0x0041 00065 (demo.go:14)	CALL	runtime.newobject(SB) # new一个funcval
	0x0046 00070 (demo.go:14)	MOVQ	8(SP), AX # 返回值
	0x0050 00080 (demo.go:14)	LEAQ	"".fn.func1(SB), CX # 取实际函数地址
	0x0057 00087 (demo.go:14)	MOVQ	CX, (AX) # 保存实际地址
	0x0061 00097 (demo.go:14)	MOVQ	"".a+16(SP), CX # 保存变量a到funcval
	0x0066 00102 (demo.go:14)	MOVQ	CX, 8(AX) 
	0x006f 00111 (demo.go:14)	MOVQ	AX, "".~r0+48(SP) # 设置返回值
	0x007d 00125 (demo.go:14)	RET

"".fn.func1 STEXT size=258 args=0x0 locals=0x88
	0x0000 00000 (demo.go:14)	TEXT	"".fn.func1(SB), NEEDCTXT, $136-0
	0x0036 00054 (demo.go:14)	MOVQ	8(DX), AX	# [DX+8]实际上存储的就是闭包引用外部的变量a
	0x003a 00058 (demo.go:14)	MOVQ	AX, "".a+48(SP) # 将AX赋值给变量a
	0x003f 00063 (demo.go:15)	MOVQ	AX, ""..autotmp_2+56(SP)
	0x0056 00086 (demo.go:15)	LEAQ	type.int(SB), AX # fmt.Println函数实际接收的是[]interface{}，这里需要先将a转换成interface{}类型
	0x005d 00093 (demo.go:15)	MOVQ	AX, (SP)
	0x0061 00097 (demo.go:15)	MOVQ	""..autotmp_2+56(SP), AX
	0x0066 00102 (demo.go:15)	MOVQ	AX, 8(SP)
	0x006b 00107 (demo.go:15)	CALL	runtime.convT2E64(SB)
```

从上面我们可以看到，`go`的闭包是通过`funcval`携带额外的上下文信息来实现的。

当创建闭包函数时，将被闭包捕获的变量的地址保存到`funcval`，当调用闭包函数时，会将`funcval`的地址保存到`DX`寄存器，执行闭包函数时，可以通过`DX`寄存器来访问这些变量。



### 实现猴子补丁

现在，我们要在`go`中实现猴子补丁，所想要实现的效果是：

```go
func a() {
	fmt.Println("run a")
}
func b() {
	fmt.Println("run b")
}

func main() {
	a() // run a
	replace(a, b)
	a() // run b
}
```

我们要在`replace`方法中，将对函数`a`的调用替换成对函数`b`的调用。

具体实现：

```go
func replace(a, b func()) {
	replaceFunction(**(**uintptr)(unsafe.Pointer(&a)), *(*uintptr)(unsafe.Pointer(&b)))
}

// from is a pointer to the actual function
// to is a pointer to a go funcvalue
func replaceFunction(from, to uintptr) {
    // demo只支持64bit
	if unsafe.Sizeof(uintptr(1)) != 8 {
		panic("only support amd64")
	}
    // jmpToFunctionValue生成跳转到to代表的函数的机器码
	jumpData := jmpToFunctionValue(to)
    // 使用生成的机器码替换from函数
	copyToLocation(from, jumpData)
	return
}

// movabs rdx,to # to是一个*funcval，需要将其存储到DX寄存器，rdx是64bit的DX寄存器
// jmp QWORD PTR [rdx] # 跳转到to对应的实际函数的开始处执行
func jmpToFunctionValue(to uintptr) []byte {
	return []byte{
		0x48, 0xBA,
		byte(to),
		byte(to >> 8),
		byte(to >> 16),
		byte(to >> 24),
		byte(to >> 32),
		byte(to >> 40),
		byte(to >> 48),
		byte(to >> 56), // movabs rdx,to
		0xFF, 0x22,     // jmp QWORD PTR [rdx]
	}
}

// 内存替换，因为code所在的代码段默认是只读的，因此需要使用系统调用mprotect将其更改为可写的
func copyToLocation(location uintptr, data []byte) {
	f := rawMemoryAccess(location, len(data))
	mprotectCrossPage(location, len(data), syscall.PROT_READ|syscall.PROT_WRITE|syscall.PROT_EXEC)
	copy(f, data[:])
	mprotectCrossPage(location, len(data), syscall.PROT_READ|syscall.PROT_EXEC)
}

// 将指定内存地址转换成一个slice
func rawMemoryAccess(p uintptr, length int) []byte {
	return *(*[]byte)(unsafe.Pointer(&reflect.SliceHeader{
		Data: p,
		Len:  length,
		Cap:  length,
	}))
}

// 使用系统调用mprotect修改指定内存的访问权限
func mprotectCrossPage(addr uintptr, length int, prot int) {
	pageSize := syscall.Getpagesize()
	for p := pageStart(addr); p < addr+uintptr(length); p += uintptr(pageSize) {
		page := rawMemoryAccess(p, pageSize)
		err := syscall.Mprotect(page, prot)
		if err != nil {
			panic(err)
		}
	}
}

// 内存页对齐
func pageStart(ptr uintptr) uintptr {
	return ptr & ^(uintptr(syscall.Getpagesize() - 1))
}
```

当执行`replace(a,b)`时，会动态将函数`a`的指令替换成`movabs rdx,to; jmp QWORD PTR [rdx]`

之后调用函数`a`时，会执行`call`指令，这时候会把传递给函数`a`的参数保存到栈上，并且将返回地址保存到指定的寄存器`RA`中；

因为函数`a`被替换成上诉两条指令，因此会跳转到函数`b`执行，这时候函数`b`可以直接使用栈上的参数（这就要求两个函数要有相同的函数签名）；

当函数`b`执行完成时，会执行`ret`指令，这时候会把返回值保存到栈上，同时将`RA`中的返回地址弹出到`PC`寄存器中；

对于函数调用者来说，整个过程是透明的。



### refer

[monkey patching in Go](https://bou.ke/blog/monkey-patching-in-go/)

[bouk/monkey](https://github.com/bouk/monkey)