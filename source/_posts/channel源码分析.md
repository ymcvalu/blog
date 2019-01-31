---
title: channel源码分析
date: 2019-01-31 08:54:15
tags:
	- go
	- channel
---

`channel`是`go`中的一种数据结构，可以用来实现并发控制、协程间通信、...

### 结构定义

`channel`结构在`runtime`中的定义如下：

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue，缓冲区是一个环形队列
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32 // 标记该channel是否已经关闭
	elemtype *_type // element type，该channel内数据元素的类型
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters，
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}

type waitq struct {
	first *sudog // sudog封装了阻塞的g
	last  *sudog
}
```

### 创建channel

当我们要创建channel时，需要使用`make`接口，对应的创建逻辑如下：

```go 
type chantype struct {
	typ  _type
	elem *_type
	dir  uintptr
}
// 可以看到这里返回的是*hchan，也说明我们代码中的channel实际上就是一个指针类型 
func makechan(t *chantype, size int) *hchan {
	elem := t.elem // channel的元素类型

	// compiler checks this but be safe.
	if elem.size >= 1<<16 { 
		throw("makechan: invalid channel element type")
	}
   	// 内存对齐校验
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}
	// buf size校验
	if size < 0 || uintptr(size) > maxSliceCap(elem.size) || uintptr(size)*elem.size > maxAlloc-hchanSize {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case size == 0 || elem.size == 0: // 缓冲区为0
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = unsafe.Pointer(c)
	case elem.kind&kindNoPointers != 0: // 不包含指针
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
        // 包含指针，hchan和buf要分开分配内存
		c = new(hchan)
		c.buf = mallocgc(uintptr(size)*elem.size, elem, true)
	}
	// 初始化状态
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
	}
	return c
}
```

### 写入channel

```go
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 往空channel写入，会导致阻塞
    if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(unsafe.Pointer(c), callerpc, funcPC(chansend))
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second c.recvq.first or c.qcount depending on kind of channel).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation.
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)
	// channel已经关闭，panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
	// 如果有阻塞在写的g，直接发送给这个g
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
	// buf channel并且有足够buf，写入buf
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
    // 写阻塞
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
    // 挂起
	goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
	// 这里是阻塞被唤醒之后的逻辑
	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
    // channel被关闭时，会设置gp.param=nil
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
        // 该channel已经关闭，panic
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```



### 从channel读取

```go 
// 单返回值版本
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

// 两返回值版本
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}
	// 从空channel读会导致阻塞
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not ready for receiving, we observe that the
	// channel is not closed. Each of these observations is a single word-sized read
	// (first c.sendq.first or c.qcount, and second c.closed).
	// Because a channel cannot be reopened, the later observation of the channel
	// being not closed implies that it was also not closed at the moment of the
	// first observation. We behave as if we observed the channel at that moment
	// and report that the receive cannot proceed.
	//
	// The order of operations is important here: reversing the operations can lead to
	// incorrect behavior when racing with a close.
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)
	// 如果channel已经关闭，并且缓冲区内元素数量为0，则返回空元素
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(unsafe.Pointer(c))
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false // 第二个返回值false表示没有读取到真正的内容
	}
	// 如果存在等待写的g，直接从这个g读取
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
	// 如果缓冲区存在元素，则读取缓冲区元素
    // channel即使已经被关闭，如果缓冲区还有元素，仍可以读取
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
    // 阻塞在写操作
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
    // 挂起
	goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

    // 这之后是唤醒之后的逻辑
	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	// 如果已经关闭就返回false
    return true, !closed
}
```



### 关闭channel

```go 

func closechan(c *hchan) {
    // 如果关闭nil channel，则panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
   	// 如果channel已经关闭，则panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}
	// 竞争检测
	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(unsafe.Pointer(c), callerpc, funcPC(closechan))
		racerelease(unsafe.Pointer(c))
	}
	// 设置关闭标志位
	c.closed = 1
	// glist用于收集释放的g
	var glist *g

	// release all readers
    // 关闭时，要释放所有阻塞在读操作的协程
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
        // 从已关闭的channel中读取的都是空内容，因此这里将元素内存置零
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
        // 从sudog中获取阻塞的g
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, unsafe.Pointer(c))
		}
        // 链表设置
		gp.schedlink.set(glist)
		glist = gp
	}

	// release all writers (they will panic)
    // 释放等待写的channel
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, unsafe.Pointer(c))
		}
		gp.schedlink.set(glist)
		glist = gp
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
    // 将上面释放的所有g设置为就绪状态，等待调度
	for glist != nil {
		gp := glist
		glist = glist.schedlink.ptr()
		gp.schedlink = 0
		goready(gp, 3)
	}
}

```

