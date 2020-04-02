---
title: go隐藏的内存padding
date: 2020-04-01 18:27:10
tags:
    - go
---

今天面试，被问到一个题目：
```go
type S struct {
	A uint32
	B uint64
	C uint64
	D uint64
	E struct{}
}
```
上面的`struct S`，占用多大的内存。
首先，我们先明确`S`的是8字节对齐的，因此我给出的答案是32。

但是很明显，答案并不是32，否则就没有这篇文章了。

我们先来看下正确的答案：
```go
func main() {
	fmt.Println(unsafe.Offsetof(S{}.E))
	fmt.Println(unsafe.Sizeof(S{}.E))
	fmt.Println(unsafe.Sizeof(S{}))
}
```
输出：
```
32
0
40
```
可以看到，`S.E`的offset是32，并且size确实是0，但是S实例的size却是40。说明S.E后面还有一个隐藏的8byte的padding。

为什么需要这个padding呢？

带着疑问，我在github上面提了个[issue](https://github.com/golang/go/issues/38194)，很快就得到了社区大佬的回复：
> **Trailing zero-sized struct fields are padded because if they weren't, &C.E would point to an invalid memory location.**

并且还给了个相关的[issue](https://github.com/golang/go/issues/9401)地址：
> If a non-zero-size struct contains a final zero-size field f, the address &x.f may point beyond the allocation for the struct. This could cause a memory leak or a crash in the garbage collector (invalid pointer found). 

go的内存分配，首先是按照`sizeclass`划分span，然后每个span中的page又分成一个个小格子：
```
|-------------
|obj1|obj2|...
|-------------
```
比如上面的，如果一个结构体是以`struct{}`结尾，并且其末尾本身没有额外的padding，就像上面的`S`，假如obj1是一个S的实例，那么`&S.E`就会指向obj2。这会产生内存泄漏或者gc crash，因此编译器会在后面自动添加一个隐藏的padding，来规避这个问题。

但是，如果S本身末尾已经有额外的padding了，就不需要额外添加这个隐藏的padding了，比如
```go
type SS struct {
	A uint32
	B uint64
	C uint64
	D uint32
	E struct{}
}
```
上面的`SS`本身末尾已经有4byte的padding了，其size就是32byte。

而因为`S`是8byte对齐的，因此最后的padding也就只能是8byte了，因此最后总的内存占用就是40byte了。

那为什么8byte对齐，其padding就要是8byte呢？这个问题可以归纳为，如果一个struct是n byte对齐，那么其最终大小需要是n的倍数。

因为，假如我们现在有一个struct的数组，那么在内存上，这些struct是相连存放的：
```
|arr[0]|arr[1]|...
```
因为arr[0]需要内存对齐，arr[1]也需要内存对齐，这就要求n byte对齐的struct，其内存占用就要是n的倍数。




继续上面的S结构体，我们再来看一下另一个迷惑行为：
```go
func main() {
	s1 := GetS()
	s2 := GetS()
	ptr1 := uintptr(unsafe.Pointer(s1))
	ptr2 := uintptr(unsafe.Pointer(s2))
	fmt.Printf("%p %p\n", s1, s2)
	fmt.Println(ptr2 - ptr1)

}

func GetS() *S {
	return &S{}
}

type S struct {
	A uint32
	B uint64
	C uint64
	D uint64
	E struct{}
}
```
上面输出是：
```
0x454030 0x454060
48
```
我们上面已经分析了，`S`的内存占用是40byte，这里连续在heap上分配两次内存，在内存上应该是会连续的，输出也验证了我们的猜想，但是他们中间间隔却是48byte？

因为，go的内存分配，已经将内存按照sizeclass划分成不同的一个个小格子了，但是并没有40byte大小的小格子，只能用与他最接近的也就是48byte的了，我们可以在`runtime/sizeclasses.go`里面看到：
```go
var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}
```