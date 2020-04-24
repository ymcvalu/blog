---
title: go性能分析
date: 2019-05-12 21:44:08
tags:
	- go
	- 性能分析
---

CPU问题：

使用`pprof`定位`cpu`瓶颈：
- 代码逻辑问题，优化算法
- gc问题，使用pprof定位**频繁对象分配**的代码，使用`sync.Pool`优化

内存问题：

使用`pprof`定位**大量内存分配**的代码
- 是否内存泄露
- 是否`slice`一直扩容
- ...


1. 影响`gc`的因素是分配的对象数量，而不是内存分配的大小，因此`[]Bean`更优于`[]*Bean`
2. 对于频繁分配的对象，应该使用对象池进行复用
3. 对于`slice`和`map`，尽量进行预分配，防止重复扩容
4. 注意`goroutine`的泄露问题
5. 避免创建大量`goroutine`，`g`结束后并不会被释放，而是加入空闲队列中等待复用
6. 注意全局的指针引用


[TODO]

### 参考文档

- [high-performance-go-workshop](<https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html>)