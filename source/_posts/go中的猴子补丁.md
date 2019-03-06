---
title: go中的猴子补丁
date: 2019-03-06 10:05:07
tags:
	- go
---

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

根据上面的输出我们可以发现，函数值`fn`并没有直接持有函数`a`的地址，这是因为`Go`的函数值可以包含一些额外的信息，用来实现闭包和绑定实例方法，我们可以在[源码](https://github.com/golang/go/blob/e9d9d0befc634f6e9f906b5ef7476fbd7ebd25e3/src/runtime/runtime2.go#L75-L78)中找到点函数值类型的线索：

```go
type funcval struct {
	fn uintptr
	// variable-size, fn-specific data here
}
```

现在，我们要在`go`中实现猴子补丁，所要实现的效果是：

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

具体实现如下：

```go
package main
import (
	"fmt"
	"reflect"
	"syscall"
	"unsafe"
)

// Assembles a jump to a function value
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

func rawMemoryAccess(p uintptr, length int) []byte {
	return *(*[]byte)(unsafe.Pointer(&reflect.SliceHeader{
		Data: p,
		Len:  length,
		Cap:  length,
	}))
}

func pageStart(ptr uintptr) uintptr {
	return ptr & ^(uintptr(syscall.Getpagesize() - 1))
}

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

// this function is super unsafe
// aww yeah
// It copies a slice to a raw memory location, disabling all memory protection before doing so.
func copyToLocation(location uintptr, data []byte) {
	f := rawMemoryAccess(location, len(data))
	mprotectCrossPage(location, len(data), syscall.PROT_READ|syscall.PROT_WRITE|syscall.PROT_EXEC)
	copy(f, data[:])
	mprotectCrossPage(location, len(data), syscall.PROT_READ|syscall.PROT_EXEC)
}

// from is a pointer to the actual function
// to is a pointer to a go funcvalue
func replaceFunction(from, to uintptr) {
	if unsafe.Sizeof(uintptr(1)) != 8 {
		panic("only support amd64")
	}
	jumpData := jmpToFunctionValue(to)
	copyToLocation(from, jumpData)
	return
}

func replace(a, b func()) {
	replaceFunction(**(**uintptr)(unsafe.Pointer(&a)), *(*uintptr)(unsafe.Pointer(&b)))
}
```



