---
title: goland 中获取 goid
date: 2018-08-18 17:28:15
tags:
	- go
	- goid
---

### introduce

目前网上有很多获取goroutine id的方法，主要分为两种：

- 通过runtime.Stack方法获取栈的信息，而栈信息以`goroutine {goid}` 开头，再通过字符串处理就可以提取出goid
- go中通过g来表示goroutine，而在tls中保存了当前执行的g的地址。可以通过汇编获取到g的地址，然后加上goid在g中的偏移量就可以获取到goid的值了

第一种方法实现方便，只要通过简单的字符串处理就可以获取到goid，但是性能开销较大；

第二种方法，需要结合汇编来获取当前执行的g的地址，而且需要获取到goid在g中的偏移量；而不同的版本中g的结构都不一样，因此该方法需要为每个版本都提供一种实现

### code

下面将基于go1.10实现上面两种获取goid的方案。

##### 方法一：

```go
	stack := make([]byte, 20) //读取前二十个字节
	runtime.Stack(stack, false)
	goid,_ :=strconv.Atoi(strings.Split(string(stack)," ")[1])
```

在上面的实现中，读取栈的前20个字节，其内容为`goroutine 6 ...`，我们这里只需要关注goid在字符串数组第二的位置，然后通过简单的字符串切割和类型转换就可以获取到goid了

##### 方法二：

因为g的定义在runtime.runtime2.go中，我们需要将其拷贝出来

```go
type g struct {
	stack       stack
	stackguard0 uintptr
    _defer      uintptr
	...
	goid           int64
	...
}

type stack struct {
	lo uintptr
	hi uintptr
}

type gobuf struct {
	sp   uintptr
	pc   uintptr
	g    uintptr
	ctxt unsafe.Pointer
	ret  uint64
	lr   uintptr
	bp   uintptr
}
```

拷贝的时候，因为g中还引用了其他类型，也需要一起拷贝出来。这里有个小技巧，因为我们只是需要使用g来计算goid的偏移量，因此如果有的字段是指针类型的，那么可以将其换成`uintptr`类型。比如说`_defer`是`*_defer`类型，那么可以将其换成`uintptr`类型，这样我们就不需要在自己的代码中声明`_defer`结构了。

然后，声明全局变量`offset`

```go
var offset =unsafe.Offsetof((*g)(nil).goid)
```

我们通过`unsafe.Offsetof`方法来计算goid在g中的偏移。

接着，在go文件中声明Goid方法的stub

```go
func Goid()int64
```

并在goid.s中实现该函数

```assembly
TEXT ·Goid(SB),NOSPLIT,$0-8
    MOVQ ·offset(SB),AX	//获取到全局变量offset
    MOVQ (TLS),BX		//获取当前g的地址
    ADDQ BX,AX			//计算goid的地址
    MOVQ (AX),BX		//获取goid的值
    MOVQ BX,ret+0(FP)
    RET
	//最后的空行必须保留，否则编译报错
```

在上述实现中，我直接在汇编中计算goid在内存中的地址。还有一种实现是在汇编中获取g的地址，然后将其转换成*g类型并获取goid的值，这样就不需要计算offset的值了，但是在实际测试中，前者的执行速度是后者的两倍。



以上代码可以在[github](https://github.com/ymcvalu/goid)上查看



