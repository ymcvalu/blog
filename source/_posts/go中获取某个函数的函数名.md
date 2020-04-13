---
title: go中获取某个函数的函数名
date: 2020-04-13 16:33:01
tags:
    - go
---
在`go` 中，有时候出于某些特殊目的，我们想要获取某个函数的函数名。

首先，如果我们知道了函数的`PC`值，那么就可以通过`runtime.FuncForPC`来获取函数名：
```go
fnName := runtime.FuncForPC(pc).Name()
```

所谓`PC`值就是函数的地址。现在的关键是，我们如何获取函数的`PC`值呢？

我们首先要知道，我们代码中的用于保存一个函数的变量，实际是什么：
```go
var fn1 = func(){}
var fn2 = foo
func foo(){}
```

我们可以在`runtime/runtime2`中找到：
```go
type funcval struct {
	fn uintptr
	// variable-size, fn-specific data here
}
```
我们代码中的`fn1`和`fn2`，实际上是`*funcval`。而`funcval`是一个"动态"的类型。它的第一个字段固定是函数的`PC`地址，而后面可变数量的字段，用于保存闭包捕获的外部变量，或者方法的`reciver`等。

现在，我们就可以很容易的通过一个函数值拿到它的`PC`地址了，比如获取`fn1`的`PC`地址：
```go
pc := **((**uintptr)(unsafe.Pointer(&fn1)))
```
将其封装成一个函数：
```go
func funcPC(fn interface{}) uintptr {
	ptr := (*[2]uintptr)(unsafe.Pointer(&fn))[1]
	return **((**uintptr)(unsafe.Pointer(&ptr)))
}

pc1 := funcPC(fn1)
pc2 := funcPC(fn2)
```
结合`runtime.FuncForPC`，我们就得到获取指定函数名的方法了：
```go
func funcName(fn interface{})string{
    pc := funcPC(fn)
    return runtime.FuncForPC(pc).Name()
}
```

实际上，`reflect`包已经提供了获取函数的`PC`地址了：
```go
func TestFnPC(t *testing.T) {
	fn := TestFnPC
    pc1 := **((**uintptr)(unsafe.Pointer(&fn)))
	pc2 := reflect.ValueOf(fn).Pointer()
	if pc1!=pc2{
		t.Error("failed to get pc by reflect")
	}
}
```
当`reflect.Value`表示的是一个函数值时，其`Pointer`返回的就是该函数的`PC`地址。