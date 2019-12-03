---
title: 用好切片
date: 2019-12-03 22:06:00
tags:
	- go
---
`slice`是`go`中比较简单的一个数据结构了（当然，`string`更加简单）。

先看其定义：
```go
type slice struct {
	array unsafe.Pointer  // 对应底层数组
	len   int             // 切片当前长度，也就是具有的元素个数
	cap   int             // 底层数组的实际长度
}
```

可以看到切片实际上有3个字段，第一个是其底层数组，第二个是当前具有的元素个数，我们可以直接使用下标`[0,len)`来访问他们，而`cap`字段是底层数组的长度。
我们可以使用`make`来创建一个切片：
```go
s1 := make([]User,0,10)
```
表示创建一个长度是0，容量是10的User切片，然后我们可以使用内置函数`append`来追加元素：
```go
 s1 = append(s1, User{})
```
使用`append`，如果切片的底层数组如果还有足够的容量，则可以直接扩展`len`字段，否则会发生切片的扩容，具体可以参考[切片扩容](https://mcll.top/2019/02/26/slice%E6%89%A9%E5%AE%B9/)
因为`slice`本质上是一个结构体，是一个值类型，而`append`需要更改内部状态，因此会创建一个新的切片返回。
如果发生了扩容，那么新返回的切片的底层数组和原来的底层数组是两个不同的数组，否则会共享同一个底层数组。

如果我们需要append可以返回一个有独立底层数组的切片，则可以：
```go
	s1 := make([]User, 10, 20)
	s2 :=append(s1[::len(s1)], User{}) // s1[::len(s1)]会创建一个cap等于len的切片，这样append就会发生扩容
```


在平常的开发中，我们经常使用到切片的一个场景是数据库分页查询，创建一个`slice`，然后把返回的结果集装到这个`slice`中。

那么在这个场景下我们应该怎么用好`slice`呢？

首先，因为我们是分页查询，结果集的大小是已知的，因此我们可以**预分配切片大小**。

然后，是使用`[]User`好还是`[]*User`好呢？

因为`go`的内存管理会将`mspan`按照`sizeClass`进行划分，从`8byte`到`32kb`，并且会在`P`中缓存每个级别的`mspan`（每个P中都有一个mcache，mcache中针对每种sizeClass缓存两个mspan，一个只用于分配不需要gc扫描的对象，这样整个mspan中的page都不需要扫描）。

因此，如果对象的大小不超过`32kb`（实际上我们很少会用到那么大的对象），那么内存分配的效率与分配的大小并没有多大关系。而且本身频繁的内存分配也是一种性能损耗。

因此，**推荐的是使用`[]User`类型**。这样只会分配一次内存。而如果使用`[]*User`类型，需要分配n+1次，底层数组1次加上n次切片元素。

然而，事与愿违的是，我们一般会使用第三方`orm`框架，这些框架中，会循环的通过反射去`new`对象，然后`append`进去。