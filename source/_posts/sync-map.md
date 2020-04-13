---
title: sync.map
date: 2020-04-13 23:07:53
tags:
    - go    
    - sync
---
go的`map`并不是并发安全的，因此在需要并发访问时，需要额外加锁，或者使用`sync.Map`。
> 应该尽量使用原生的map+锁，因为有编译时的类型安全检查

因为go语言本身的特性，因此大家都比较关心并发安全，最近面试有被问到`sync.Map`的实现，讲道理，之前是有看过源码的，但是时间比较久了，记忆比较模糊了，因此只能卑微的说不了解。。。

今天就来简单过一下`sync.Map`的实现。

### 定义
```go
type Map struct {
	// 互斥锁，用于保护dirty
    mu Mutex
	// readonly，这里的readonly是指map的key不会更新，但是val是可以更新的
	read atomic.Value // readOnly
	// 更新会写到dirty
	dirty map[interface{}]*entry
	// 记录read的时候miss的次数，达到阈值则将dirty提升为read
	misses int
}

type readOnly struct {
    m       map[interface{}]*entry
    // 如果dirty中有不存在于read中的key，则设置为true
	amended bool // true if the dirty map contains some key not in m.
}

// marks entries which have been deleted from the dirty map.
// 当read中的val是expunged，表示已经从dirty中删除了
var expunged = unsafe.Pointer(new(interface{}))

type entry struct {
	// unsafe.Pointer可以使用原子操作
	p unsafe.Pointer // *interface{}
}
```
首先，`sync.Map`内部是有两个map的，一个是`readonly`的read，和一个使用`mutex`保护的`dirty`。这里的`readonly`并不是指这个`map`只是只读的，而是指这个`map`不会添加`key`，或者删除`key`，但是`key`对应的`val`是可以修改的。并且`sync.Map`大量用到了`sync/atomic`包中的原子操作。

### Store实现
```go
func (m *Map) Store(key, value interface{}) {
	read, _ := m.read.Load().(readOnly)
    // 首先尝试直接cas修改，如果该key存在于dirty中，则可以成功设置
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}

	m.mu.Lock()
    // 双重锁检查
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
            // key存在于read中，但是不在dirty中，加入dirty
			m.dirty[key] = e
		}
        // 直接atomic set
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok {
        // 存在dirty中，但是不在readonly中
		e.storeLocked(&value)
	} else {
        // key还不存在
		if !read.amended {
			// lazy init dirty
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
        // 添加到dirty中
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}

func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

// 初始化dirty，并把read中未删除的key添加到dirty中
func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read, _ := m.read.Load().(readOnly)
	m.dirty = make(map[interface{}]*entry, len(read.m))
	for k, e := range read.m {
		if !e.tryExpungeLocked() {
            // 如果key未删除，将其加入dirty
			m.dirty[k] = e
		}
	}
}

// 如果key的值为nil，则认为已经删除了，将其设置为expunged
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}

// cas操作
func (e *entry) tryStore(i *interface{}) bool {
	for {
        p := atomic.LoadPointer(&e.p)
        // dirty中不存在该key，需要竞争锁添加到dirty中
		if p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
			return true
		}
	}
}
```
`Store`的逻辑，首先是，如果`read`里面已经有这个`key`存在了，会尝试直接修改它的`val`。否则需要竞争锁。

### Load实现
```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	// 首先尝试从read中读取
    read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
    // 如果read中没有，并且dirty存在额外的key
	if !ok && read.amended {
        // 尝试从dirty中读取
		m.mu.Lock()
		// 双重锁检查
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
            // 从dirty中读取
            e, ok = m.dirty[key]
     		// 更新miss
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

func (m *Map) missLocked() {
	m.misses++
	if m.misses < len(m.dirty) {
		return
	}
    // 将dirty提升为readOnly
	m.read.Store(readOnly{m: m.dirty})
	m.dirty = nil
	m.misses = 0
}

func (e *entry) load() (value interface{}, ok bool) {
	p := atomic.LoadPointer(&e.p)
    // 如果标记删除了
	if p == nil || p == expunged {
		return nil, false
	}
	return *(*interface{})(p), true
}
```
首先，尝试直接读取`read`，如果读取失败，并且`read.amended == true`，则需要加锁，尝试到`dirty`中读取。

### Delete实现
```go
// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
    // 首先从read中查找要删除的entry
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
        // 双重锁检查
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
            // 直接从dirty中删除
			delete(m.dirty, key)
		}
		m.mu.Unlock()
	}
	if ok {
		e.delete()
	}
}

func (e *entry) delete() (hadValue bool) {
    // cas标记删除
	for {
		p := atomic.LoadPointer(&e.p)
		if p == nil || p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return true
		}
	}
}
```

### 总结
现在再来看源码注释：
> The Map type is optimized for two common use cases: (1) when the entry for a given key is only ever written once but read many times, as in caches that only grow, or (2) when multiple goroutines read, write, and overwrite entries for disjoint sets of keys. In these two cases, use of a Map may significantly reduce lock contention compared to a Go map paired with a separate Mutex or RWMutex.

简单来说，就是`sync.Map`的适用场景主要有：
1. 一个给定的`key`读多写少的场景，根据上面的源码，如果写少，读的时候可以直接命中`read`中的数据并返回
2. 当多个goroutine读、写或者修改不相交的key集合时。根据源码，`Map`只有在插入不存在的key或者不存在于dirty的key时才需要拿锁。而读取和修改、删除都可以直接在`read`中通过原子操作完成，如果多个协程操作不相交的key集合，竞争情况会比较少

而且，**根据`sync.Map`的定义，是不允许进行值拷贝的，应该使用指针。**