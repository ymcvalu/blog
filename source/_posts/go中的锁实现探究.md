---
title: mutex解析
date: 2019-04-14 14:05:30
tags:
	- go
---

`go`的基础包`sync`提供了两种锁的实现，分别是`Mutex`和`RWMutex`。其中`Mutex`是互斥锁，一次只允许一个协程获取锁，`RWMutex`是读写锁，允许同时有多个协程获取读锁，但是只能有一个协程获取读锁，并且读写互斥。

### Mutex

好习惯，看源码先看注释：

```
// Mutex fairness.
//
// Mutex can be in 2 modes of operations: normal and starvation.
// In normal mode waiters are queued in FIFO order, but a woken up waiter
// does not own the mutex and competes with new arriving goroutines over
// the ownership. New arriving goroutines have an advantage -- they are
// already running on CPU and there can be lots of them, so a woken up
// waiter has good chances of losing. In such case it is queued at front
// of the wait queue. If a waiter fails to acquire the mutex for more than 1ms,
// it switches mutex to the starvation mode.
//
// In starvation mode ownership of the mutex is directly handed off from
// the unlocking goroutine to the waiter at the front of the queue.
// New arriving goroutines don't try to acquire the mutex even if it appears
// to be unlocked, and don't try to spin. Instead they queue themselves at
// the tail of the wait queue.
//
// If a waiter receives ownership of the mutex and sees that either
// (1) it is the last waiter in the queue, or (2) it waited for less than 1 ms,
// it switches mutex back to normal operation mode.
//
// Normal mode has considerably better performance as a goroutine can acquire
// a mutex several times in a row even if there are blocked waiters.
// Starvation mode is important to prevent pathological cases of tail latency.
```

##### 结构声明

首先我们来看一下`Mutex`的声明

```go
type Mutex struct {
	state int32 
	sema  uint32  
}
```

可以看到，`Mutex`只包含两个字段，其中`state`用于记录锁的状态，第一位表示锁是否被占用，第二位用于通知`unlock`方法不要唤醒一个`waiter`参与锁的抢夺，第三位表示是当前锁否处于饥饿模式，从第四位到第32位则用于记录当前阻塞在等待锁的协程数量；而`sema`是用于在多个协程之间进行同步的信号量，这个后面再说。

##### 抢占锁

我们首先来看一下`Lock`方法实现：

```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// 首先先尝试获取unlock的锁
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        // 竞争检查，忽略。。。
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
        // 表示cas操作成功，获取锁，直接返回
		return
	}
	
    // 记录开始等待的时间
	var waitStartTime int64
	// 是否处于饥饿模式
    starving := false
    // 是否设置了state的mutexWoken状态位
	awoke := false
    // 记录自旋次数
	iter := 0
    // 获取当前的状态
	old := m.state
	for {
		// 如果当前锁处于Locked状态，并且允许自旋，则进入自旋状态
        // 允许自旋的条件：不处于饥饿模式，自旋次数小于4，running on a multicore machine and GOMAXPROCS>1 and there is at least one other running P and local runq is empty
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// 更新state的第二位，表示当前有运行中的协程在抢占锁
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
            // 执行自旋，所谓自旋，就是一种busy wait，当前协程不阻塞，等待一会儿重新尝试获取锁，这里的doSpin是在runtime包中实现的
			runtime_doSpin()
			iter++ // 添加自旋次数
			old = m.state
			continue 
		}
        
        // 参与锁的抢夺
        
		new := old
		// 如果不处于饥饿模式，设置状态位尝试获取锁
        if old&mutexStarving == 0 {
			new |= mutexLocked
		}
        
        // 如果已经锁住或者处于饥饿模式，需要进入等待队列，waiter数量加1
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
        
		// 如果需要进入饥饿状态，则设置饥饿标志位
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
        
        // state的mutexWoken状态位
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
        // cas更新锁状态
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 更新成功，并且原来锁为unlock并且不属于饥饿模式，直接返回
            // 这种情况是锁处于正常模式，新的协程与从等待中唤醒的协程竞争锁，并且竞争成功
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
            
			// 是否是从等待状态中唤醒的
			queueLifo := waitStartTime != 0
            // 设置开始等待时间
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
            
            // 当前协程抢占锁失败，阻塞等待，这里需要传入信号量sema和queueLifo
            // 信号量sema用来在多个协程之间同步
            // queueLifo如果为true，表示当前协程与新的协程竞争锁失败，加入队首，否则加入队尾
			runtime_SemacquireMutex(&m.sema, queueLifo)
            
            // 执行到这里表明协程被从等待队列中唤醒了
            // 如果等待时间大于1ms则进入饥饿模式
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
            // 当前锁处于饥饿模式，表明没有其他协程会与当前协程竞争锁
			if old&mutexStarving != 0 {
				// 检查状态位
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
                // 更新状态位，饥饿模式，只有当前协程能够抢占锁
                // 阻塞等待锁的协程数量需要减1
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
                // 如果不需要进入饥饿模式，或者当前等待队列为空，则清空饥饿模式
				if !starving || old>>mutexWaiterShift == 1 {
					delta -= mutexStarving
				}
                // 这里使用原子Add而不是CAS操作，因为可能在这个时刻有新的协程因为等待锁而阻塞，这时候如果使用CAS会失败
				atomic.AddInt32(&m.state, delta)
				break // 返回
			}
            // 当前锁处于正常模式，因此需要和新加入的协程竞争锁
			awoke = true
			iter = 0
		} else { // cas失败，重试
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```

接下来我们看一下`runtime_SemacquireMutex`这个方法，该方法的实现在`runtime`包中：

```go
// 这里使用`go:linkname`告诉链接器将该方法与sync包的runtime_SemacquireMutex方法链接
//go:linkname sync_runtime_SemacquireMutex sync.runtime_SemacquireMutex
func sync_runtime_SemacquireMutex(addr *uint32, lifo bool) {
	semacquire1(addr, lifo, semaBlockProfile|semaMutexProfile)
}

func semacquire1(addr *uint32, lifo bool, profile semaProfileFlags) {
    // 获取当前g
	gp := getg()
    // 不允许在系统栈执行Lock方法
	if gp != gp.m.curg {
		throw("semacquire not on the G stack")
	}

	// 尝试捕获信号量，成功则直接返回
	if cansemacquire(addr) {
		return
	}
	
    // sudog用来表示一个阻塞的g，同一个锁的等待协程会构成一条sudog链表，多条链表会组织成一颗平衡树
	s := acquireSudog()
   // 通过sema的地址获取对应的root，一个root中有多个sema的等待队列
	root := semroot(addr)
	t0 := int64(0)
	s.releasetime = 0
	s.acquiretime = 0
	s.ticket = 0
	if profile&semaBlockProfile != 0 && blockprofilerate > 0 {
		t0 = cputicks()
		s.releasetime = -1
	}
	if profile&semaMutexProfile != 0 && mutexprofilerate > 0 {
		if t0 == 0 {
			t0 = cputicks()
		}
		s.acquiretime = t0
	}
	for {
		lock(&root.lock)
		// Add ourselves to nwait to disable "easy case" in semrelease.
		atomic.Xadd(&root.nwait, 1)
		// 再次尝试捕获信号量
		if cansemacquire(addr) {
			atomic.Xadd(&root.nwait, -1)
			unlock(&root.lock)
			break
		}
		// 加入等待队列，一个root内有多个等待队列，这些等待队列通过平衡树来组织，等待队列通过addr来标识，每个等待队列就是一个sudog列表
		root.queue(addr, s, lifo)
        // 阻塞当前g
		goparkunlock(&root.lock, waitReasonSemacquire, traceEvGoBlockSync, 4)
		// g被唤醒，如果ticket不为0或者再次尝试捕获信号量成功
        if s.ticket != 0 || cansemacquire(addr) {
			break // 捕获成功
		}
	}
	if s.releasetime > 0 {
		blockevent(s.releasetime-t0, 3)
	}
    // 释放sudog
	releaseSudog(s)
}
```

##### 释放锁

现在来看一下`Unlock`方法实现：

```go
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
    // 清空锁状态
	new := atomic.AddInt32(&m.state, -mutexLocked)
    // 检查状态位
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
    // 如果不处于饥饿模式
	if new&mutexStarving == 0 {
		old := new
		for {
			// 如果没有等待锁的协程，或者设置了mutexWoken标志位，直接返回
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
            
			// 唤醒一个等待的协程，参与锁竞争
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false)
				return
			}
			old = m.state
		}
	} else {
		// 当前处于饥饿模式，从等待队列唤醒协程
		runtime_Semrelease(&m.sema, true)
	}
}
```

接下来看一下`runtime_Semrelease`方法：

```go
//go:linkname sync_runtime_Semrelease sync.runtime_Semrelease
func sync_runtime_Semrelease(addr *uint32, handoff bool) {
	semrelease1(addr, handoff)
}

func semrelease1(addr *uint32, handoff bool) {
	root := semroot(addr)
	atomic.Xadd(addr, 1) // addr值加1

	// Easy case: no waiters?
	// This check must happen after the xadd, to avoid a missed wakeup
	// (see loop in semacquire).
	if atomic.Load(&root.nwait) == 0 {
		return
	}

	// Harder case: search for a waiter and wake it.
	lock(&root.lock)
	if atomic.Load(&root.nwait) == 0 {
		// The count is already consumed by another goroutine,
		// so no need to wake up another goroutine.
		unlock(&root.lock)
		return
	}
    // 从队首获取等待的sudog
	s, t0 := root.dequeue(addr)
	if s != nil {
		atomic.Xadd(&root.nwait, -1)
	}
	unlock(&root.lock)
	if s != nil { // May be slow, so unlock first
		acquiretime := s.acquiretime
		if acquiretime != 0 {
			mutexevent(t0-acquiretime, 3)
		}
		if s.ticket != 0 {
			throw("corrupted semaphore ticket")
		}
        // 如果handoff为true，则尝试捕获信号量
		if handoff && cansemacquire(addr) {
            // 成功，则更新ticket为1
			s.ticket = 1
		}
        // 唤醒g
		readyWithTime(s, 5)
	}
}
```



> 新版本的`Mutex`实现，将`Lock`和`Unlock`方法中的只保留了`fastpath`，而`slowpath`部分移到新的子过程中，用于内联优化。



### RWMutex

现在来看一下`RWMutex`，同样都，先看注释：

```
// A RWMutex is a reader/writer mutual exclusion lock.
// The lock can be held by an arbitrary number of readers or a single writer.
// The zero value for a RWMutex is an unlocked mutex.
//
// A RWMutex must not be copied after first use.
//
// If a goroutine holds a RWMutex for reading and another goroutine might
// call Lock, no goroutine should expect to be able to acquire a read lock
// until the initial read lock is released. In particular, this prohibits
// recursive read locking. This is to ensure that the lock eventually becomes
// available; a blocked Lock call excludes new readers from acquiring the
// lock.
```

##### 结构声明

```go
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers
	readerWait  int32  // number of departing readers
}
```



##### 抢占读锁

```go
func (rw *RWMutex) RLock() {
   if race.Enabled {
      _ = rw.w.state
      race.Disable()
   }
    // readCount加1，如果小于0，表明当前有写锁正在等待，则阻塞等待
   if atomic.AddInt32(&rw.readerCount, 1) < 0 {
      // A writer is pending, wait for it.
      runtime_SemacquireMutex(&rw.readerSem, false)
   }
   if race.Enabled {
      race.Enable()
      race.Acquire(unsafe.Pointer(&rw.readerSem))
   }
}
```

##### 释放读锁

```go
func (rw *RWMutex) RUnlock() {
   if race.Enabled {
      _ = rw.w.state
      race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
      race.Disable()
   }
    // readerCount减1，小于0说明有等待写
   if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
      if r+1 == 0 || r+1 == -rwmutexMaxReaders {
         race.Enable()
         throw("sync: RUnlock of unlocked RWMutex")
      }
      // A writer is pending.
       // readerWait减1，如果为0表示读锁释放完，唤醒等待读
      if atomic.AddInt32(&rw.readerWait, -1) == 0 {
         // The last reader unblocks the writer.
         runtime_Semrelease(&rw.writerSem, false)
      }
   }
   if race.Enabled {
      race.Enable()
   }
}
```

##### 抢占写锁

```go
func (rw *RWMutex) Lock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	// First, resolve competition with other writers.
	rw.w.Lock()
	// Announce to readers there is a pending writer.
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	// Wait for active readers.
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
		race.Acquire(unsafe.Pointer(&rw.writerSem))
	}
}
```

##### 释放写锁

 ```go
func (rw *RWMutex) Unlock() {
	if race.Enabled {
		_ = rw.w.state
		race.Release(unsafe.Pointer(&rw.readerSem))
		race.Disable()
	}

	// Announce to readers there is no active writer.
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}
	// Unblock blocked readers, if any.
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false)
	}
	// Allow other writers to proceed.
	rw.w.Unlock()
	if race.Enabled {
		race.Enable()
	}
}
 ```





