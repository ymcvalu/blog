---
title: go函数栈布局
date: 2019-04-29 23:09:12
tags:
	- go
---

# Go函数调用布局

### 函数具有局部变量（栈帧大小大于0）

![](/img/go-func-statck1.png)

从上图我们可以看到，函数调用时，参数和返回值是通过栈来传递的，通过栈来传递函数，能够很好的实现多返回值，而且当发生`goroutine`的调度时，只需要切换`SP/BP`等少量寄存器，而不需要对通用寄存器进行切换，而且，当我们使用**命名返回值**时，是直接在对应的栈上进行更新，因此我们可以在`defer`函数内更新返回值。

##### 测试代码

```go
func frameInfo(i int) (uintptr, uintptr, uintptr, uintptr,int,int)
func main() {
	fmt.Println(frameInfo(15))
}
```

```assembly
TEXT ·frameInfo(SB),$8-54 // 这里8表示函数局部栈帧8个字节，参数和返回值共54个字节
    MOVQ SP, AX // 取硬件寄存器SP的内容，作为第一个返回值
    MOVQ AX, ret0+8(FP)
    LEAQ i+0(SP), AX // 取pseudo_sp的值，作为第二个返回值
    MOVQ AX, ret2+16(FP)
    MOVQ BP, AX  // 取硬件寄存器BP的值，作为第三个返回值
    MOVQ AX, ret1+24(FP)
    LEAQ i+0(FP), AX // 取pseudo_fp的值，作为第四个返回值
    MOVQ AX, ret3+32(FP)
    MOVQ i+0(FP), AX // 通过伪寄存器fp获取参数i作为第五个返回值
    MOVQ AX, ret4+40(FP)
    MOVQ i+16(SP), AX // 通过伪寄存器sp获取参数i作为第六给返回值
    MOVQ AX, ret5+48(FP)
    RET

```

查看输出内容：

```sh
$ go run .
824634146480 824634146488 824634146488 824634146504 15 15
```

可以看到伪寄存器`SP`和栈底寄存器`BP`指向的是同一个地址，而伪寄存器`FP`比伪寄存器`SP`大`16`个字节，其中高`8`个字节保存函数的返回地址，而低`8`个字节保存函数调用者的`BP`；而硬件`SP`和伪寄存器`SP	`之间相差的字节数刚好是函数栈帧（这里的函数栈帧不包含保存调用者`BP`的`8`个字节）的大小，可以通过调整函数的局部栈帧大小观察其关系

```assembly
TEXT ·frameInfo(SB),$16-54 // 调整栈帧大小为16字节
```

查看输出内容：

```sh
$ go run .
824634187432 824634187448 824634187448 824634187464 15 15
```

可以看到，新的输出中，硬件`SP`和伪寄存器`SP`之间相差为`16`字节，正好为栈帧大小。



### 函数栈帧大小为0

当函数栈帧大小为`0`时，情况就有点不同了。

因为这个时候，当前函数没有分配栈帧，因此硬件寄存器`BP`不需要保存当前函数栈的栈底，也就不需要在栈上额外分配一个`8`字节的空间来保存函数调用者的`BP`寄存器内容，也就是说当前`BP`寄存器直接保存的就是函数调用者的`BP`寄存器信息。

这时候的栈结构：

![](/img/go-func-statck2.png)

##### 测试代码

```go
func getBp() (uintptr, uintptr)
func zeroFrame(i int) (uintptr, uintptr, uintptr, uintptr, int, int, uintptr)
func main() {
	fmt.Println(getBp())
	fmt.Println(zeroFrame(11))
}
```

```assembly
TEXT ·getBp(SB),$8-16
    MOVQ 0(BP), AX // 获取调用者函数的栈底地址
    MOVQ AX, ret0+0(FP)
    MOVQ ra+8(SP), AX // 获取当前函数的返回地址
    MOVQ AX, ret0+8(FP)
    RET

TEXT ·zeroFrame(SB),$0-64
    MOVQ SP, AX // 取硬件寄存器SP的内容，作为第一个返回值
    MOVQ AX, ret0+8(FP)
    LEAQ i+0(SP), AX // 取pseudo_sp的值，作为第二个返回值
    MOVQ AX, ret1+16(FP)
    MOVQ BP, AX  // 取硬件寄存器BP的值，作为第三个返回值
    MOVQ AX, ret2+24(FP)
    LEAQ i+0(FP), AX // 取pseudo_fp的值，作为第四个返回值
    MOVQ AX, ret3+32(FP)
    MOVQ i+0(FP), AX // 通过伪寄存器fp获取参数i作为第五个返回值
    MOVQ AX, ret4+40(FP)
    MOVQ i+8(SP), AX // 通过伪寄存器sp获取参数i作为第六给返回值
    MOVQ AX, ret5+48(FP)
    MOVQ addr+0(SP), AX // 获取当前函数的返回地址
    MOVQ AX, ret6+56(FP)
    RET

```

查看输出内容：

```sh
$ go run .
824634187656 4776270
824634187392 824634187392 824634187656 824634187400 11 11 4776438
```

函数`getBp`返回了调用者也即`main`函数的栈底信息，和当前函数的返回地址

然后调用函数`zeroFrame`，该函数栈帧大小为`0`，可以看到执行该函数时，`BP`寄存器依然是`824634187656`，而且硬件寄存器`SP`和伪寄存器`SP`指向同一个位置，并且和伪寄存器`FP`只相差`8`个字节，而这`8`个字节保存的是当前函数的返回地址，我们可以看到两个函数的返回地址是在同一个段中的。