---
title: go中的gc写屏障
date: 2020-04-14 10:15:36
tags:
    - go
    - gc
---
go的gc实现是三色标记+并发扫描，gc协程和用户的工作协程并发运行。
![三色标记](img/../../img/tricolor-marking.gif)

在gc开始阶段，会先执行`STW`，在`STW`中会开启写屏障。最开始go使用的是Dijkstra写屏障。

Dijkstra写屏障的伪代码逻辑为：
```
writePointer(slot, ptr):   
    shade(ptr)    // 新的对象标记为gray
    *slot = ptr   // 实际赋值
```
对象指针赋值时，比如`a.x=&b`，保守认为a已经被标记为黑色了，因此在赋值前，先将b标记为灰色的，确保对象b不会被丢失。

可以看到，只有当指针赋值的时候才需要写屏障，读取是不需要的。

**trade-off：**
> 如果目标指针是在栈上，比如p=&b，p是栈上的指针值，如果也开启写屏障，代价会非常昂贵，因此go选择了让栈保持灰色（permagrey），也就是gc阶段栈保持灰色的，并在标记终止阶段对栈执行rescan，这需要在stw时执行，会延长stw时间。


比如，我们现在来看这么一个例子：
```go
type Node struct {
	Val  int
	Next *Node
}

func main() {
	n1 := NewNode(10).Next
	_ = n1
}

//go:noinline
func NewNode(v int) *Node {
	return &Node{
		Val:  v,
		Next: &Node{Val: v},
	}
}
```
在上面的代码中，我们把一个堆中的指针赋值给栈上的指针变量。

运行命令：
```sh
$ go tool compile -N -S wb.go > wb.s
```
查看`wb.s`中的汇编代码：
```
    0x0024 00036 (wb.go:22)	MOVQ	$10, (SP)
	0x002c 00044 (wb.go:22)	CALL	"".NewNode(SB)
	0x0031 00049 (wb.go:22)	MOVQ	8(SP), AX
	0x0036 00054 (wb.go:22)	MOVQ	AX, ""..autotmp_1+24(SP)
	0x003d 00061 (wb.go:22)	MOVQ	8(AX), AX
	0x0041 00065 (wb.go:22)	MOVQ	AX, "".n1+16(SP)
```
可以看到生成的汇编中并没有使用写屏障。

我们就`Node`来分析一下，为什么只使用`Dijkstra`，需要`rescan`栈。
1. n1 := NewNode(10)
2. n2 := NewNode(11)
3. n1.Next=n2
4. 假如p是栈上的指针，执行：p := n1.Next，这时候不需要写屏障
5. 假设这时候n1还没有被扫描，执行：n1.Next=n3
6. 这时候扫描n1，因为n1和n2已经没有关联了，n2不会被扫描到，但是n2还被栈上的p引用
7. 如果没有stack rescan，那么n2就会被回收，导致栈上的指针引用了未分配内存

gc的stack rescan是在`STW`期间执行的，这无疑会延长`STW`的时间。

为了进一步提高gc性能，减少stw的时间，同时减少gc实现的复杂性，go1.8引入了混合写屏障，结合了`yuasa-style deletion write barrier`和`Dijkstra-style insertion write barrier`。

我们来看一下`yuasa-style write barrier`：
```
writePointer(slot, ptr): 
  shade(*slot) 
  *slot = ptr
```
写入时，先将原来的对象标灰，这是一种快照技术，确保原来的对象引用关系不会丢失。

混合写屏障混合这两种之后：
```
writePointer(slot, ptr): 
    shade(*slot)
    if current stack is grey: 
      shade(ptr)
    *slot = ptr
```
>  如果栈已经被扫描了，那么这个对象不需要标记了（要么在栈标记的时候被标记，要么从堆移到栈，由yuasa写屏障保证，因此栈是gray才shade *ptr），因此在混合写屏障中，当栈为灰色才需要标记ptr。

接下来，我们分析一下，混合写屏障为什么可以解决只使用Dijkstra写屏障的问题，从而消除stack rescan。
1. 假如对象a在堆中，还没有被扫描
2. p是栈上指针，执行 p:=a.b，不需要写屏障
3. 执行：a.b=&c
4. 这时候a被扫描

当采用混合写屏障的时候，当执行上面步骤3的时候，通过`yuasa-style`写屏障，会标记b指向的对象，确保对象b不会被丢失。

可以看到，`yuasa-style`写屏障，实际上是一种**快照技术**，保存原来对象引用图的边。采用`yuasa-style`有一点缺点，可能对象b真的已经不再被引用了，但是需要等到下一轮gc才会被回收，但是相比stack rescan来说这点开销微乎其微。

写屏障实现：编译器在执行指针赋值时，会判断是否开启了写屏障，如果开启了写屏障，调用写屏障方法来进行指针赋值，该方法由汇编实现，并且标记nosplit，执行过程不会被抢占。因为写屏障的开启和关闭是在`STW`中执行的，因此判断是否开启写屏障时，是可以直接读取全局变量。

新分配的对象需要是黑色的，因为可能被赋值给已经标黑的对象，而新分配的对象又没有引用其他对象，不需要标记灰色。


