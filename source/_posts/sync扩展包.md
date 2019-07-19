---
title: sync扩展包
date: 2019-07-19 11:38:23
tags:
	- go
---

[glang.org/x/sync](<https://github.com/golang/sync>)包提供了一些方便使用的用于并发操作的扩展



### errgroup

`errgroup`用于执行一个整体任务的一组子任务

```go
// Group代表一组子任务
type Group struct {
	cancel func()

	wg sync.WaitGroup

	errOnce sync.Once // 用于初始化err
	err     error
}

// 当一个子任务返回error或者所有子任务成功运行结束，cancel会被执行
func WithContext(ctx context.Context) (*Group, context.Context) {
    ctx, cancel := context.WithCancel(ctx)
	return &Group{cancel: cancel}, ctx
}

// 等带所有子任务都运行完成
func (g *Group) Wait() error {
	g.wg.Wait() 
	if g.cancel != nil {
		g.cancel() // cancel
	}
	return g.err
}

// Go开始运行一个子任务
func (g *Group) Go(f func() error) {
	g.wg.Add(1)

	go func() {
		defer g.wg.Done()
		// 如果返回了error
		if err := f(); err != nil {
            // errOnce确保只执行一次
			g.errOnce.Do(func() {
				g.err = err
                // 返回错误，则执行cancel
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
	}()
}
```

可以看到`Group`的源码很简单，接下来看一下官方给的demo，看一下使用方法：

```go
package main

import (
    "context"
    "crypto/md5"
    "fmt"
    "io/ioutil"
    "log"
    "os"
    "path/filepath"

    "golang.org/x/sync/errgroup"
)

func main() {
    // 计算指定路径下所有文件的md5
    m, err := MD5All(context.Background(), ".")
    if err != nil {
        log.Fatal(err)
    }
	
    for k, sum := range m {
        fmt.Printf("%s:\t%x\n", k, sum)
    }
}

// 保存文件md5
type result struct {
    path string // 文件路径 
    sum  [md5.Size]byte // md5信息
}

func MD5All(ctx context.Context, root string) (map[string][md5.Size]byte, error) {
    // 创建Group
    g, ctx := errgroup.WithContext(ctx)
    paths := make(chan string)

    // 第一个子任务遍历目录树，并把文件路径通过paths传给其他计算md5的子任务
    g.Go(func() error {
        defer close(paths)
        // 遍历目录树
        return filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
            if err != nil {
                return err
            }
            if !info.Mode().IsRegular() {
                return nil
            }
            select {
            case paths <- path:
            // 如果有子任务返回了error，则结束当前子任务
            case <-ctx.Done():
                return ctx.Err()
            }
            return nil
        })
    })

    c := make(chan result)
    const numDigesters = 20
    // 运行多个计算md5的子任务
    for i := 0; i < numDigesters; i++ {
        g.Go(func() error {
            // 从paths中读取文件路径，paths关闭则退出循环
            for path := range paths {
                data, err := ioutil.ReadFile(path)
                if err != nil {
                    return err
                }
                select {
                // 写入计算结果
                case c <- result{path, md5.Sum(data)}:
                // 如果某个子任务返回error，结束当前子任务    
                case <-ctx.Done():
                    return ctx.Err()
                }
            }
            return nil
        })
    }
    go func() {
       	// 等待所有任务结束，关闭c
        g.Wait()
        close(c)
    }()

    m := make(map[string][md5.Size]byte)
    // 读取计算结果
    for r := range c {
        m[r.path] = r.sum
    }
    
    // 检查是否存在error
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return m, nil
}
```



### semaphore

`semaphore`实现了一个加权信号量

```go
// 一个正在等待分配权重的请求
type waiter struct {
	n     int64 // 请求分配的权重
	ready chan<- struct{} // 用于通知已经分配成功
}

// 创建一个加权信号量，n表示最大可获取的权重
func NewWeighted(n int64) *Weighted {
	w := &Weighted{size: n}
	return w
}

type Weighted struct {
	size    int64 // 最大权重
	cur     int64 // 当前已经分配的权重
	mu      sync.Mutex
	waiters list.List // 等待队列
}
```

看一下信号量的acquire逻辑：

```go
// Acquire acquires the semaphore with a weight of n, blocking until resources
// are available or ctx is done. On success, returns nil. On failure, returns
// ctx.Err() and leaves the semaphore unchanged.
//
// If ctx is already done, Acquire may still succeed without blocking.
func (s *Weighted) Acquire(ctx context.Context, n int64) error {
	// 加锁保护
    s.mu.Lock()
    // 如何剩余权重足够并且等待队列为空，直接分配
	if s.size-s.cur >= n && s.waiters.Len() == 0 {
		s.cur += n // 更新当前分配的权重
		s.mu.Unlock()
		return nil
	}
	
    // 如果请求权重超过最大限制，阻塞直到context取消，直接返回，不要加入到waiters防止阻塞其他协程
	if n > s.size {
		s.mu.Unlock()
		<-ctx.Done()
		return ctx.Err()
	}
	
    // 加入到等待队列的末尾
	ready := make(chan struct{})
	w := waiter{n: n, ready: ready}
	elem := s.waiters.PushBack(w)
	s.mu.Unlock()
	
    // 等待上下文取消，或者信号量分配成功
	select {
	case <-ctx.Done():
		err := ctx.Err()
		s.mu.Lock()
		select {
		case <-ready:
            // 取消的时候分配成功了，这时候忽略掉上下文取消操作
			err = nil
		default:
            // 上下文取消，从等待队列中移除
			s.waiters.Remove(elem)
		}
		s.mu.Unlock()
		return err
	// 成功分配
	case <-ready:
		return nil
	}
}
```

接着看一下信号量释放逻辑：

```go
// Release releases the semaphore with a weight of n.
func (s *Weighted) Release(n int64) {
	s.mu.Lock()
    // 更新当前分配的权重
	s.cur -= n
	if s.cur < 0 {
		s.mu.Unlock()
		panic("semaphore: released more than held")
	}
    
    // 唤醒等待队列
	for {
		next := s.waiters.Front()
		if next == nil {
			break // No more waiters blocked.
		}
		
		w := next.Value.(waiter)
		if s.size-s.cur < w.n {
			break
		}

		s.cur += w.n
		s.waiters.Remove(next)
		close(w.ready)
	}
	s.mu.Unlock()
}
```



### singleflight

`singleflight`提供了防止函数同一时刻重复执行的功能

```go
// call表示一个函数调用
type call struct {
   wg sync.WaitGroup

   // 函数返回值
   val interface{}
   // 返回的错误
   err error

   // forgotten indicates whether Forget was called with this call's key
   // while the call was still in flight.
   forgotten bool

   // 表示该函数有多少次重复调用
   dups  int
   // 异步返回执行结果
   chans []chan<- Result
}

// Result holds the results of Do, so they can be passed
// on a channel.
type Result struct {
   Val    interface{}
   Err    error
   Shared bool
}

// Group represents a class of work and forms a namespace in
// which units of work can be executed with duplicate suppression.
type Group struct {
   mu sync.Mutex       // protects m
   m  map[string]*call // lazily initialized
}


```

```go
// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
// The return value shared indicates whether v was given to multiple callers.
// 具有相同key的函数，同一时刻多次调用只会执行一次
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
	g.mu.Lock()
    // lazy init
	if g.m == nil {
		g.m = make(map[string]*call)
	}
    
    // 如果已经存在
	if c, ok := g.m[key]; ok {
		c.dups++
		g.mu.Unlock()
		c.wg.Wait() // 等待执行结束
		return c.val, c.err, true // 执行返回调用结果
	}
    // 创建一个新的call，加入到g.m中
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

    // 同步执行函数调用
	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}
```



```go
// DoChan is like Do but returns a channel that will receive the
// results when they are ready.
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
   ch := make(chan Result, 1)
   g.mu.Lock()
   if g.m == nil {
      g.m = make(map[string]*call)
   }
   if c, ok := g.m[key]; ok {
      c.dups++
      c.chans = append(c.chans, ch)
      g.mu.Unlock()
      return ch
   }
   c := &call{chans: []chan<- Result{ch}}
   c.wg.Add(1)
   g.m[key] = c
   g.mu.Unlock()
   // 异步执行函数调用
   go g.doCall(c, key, fn)

   return ch
}
```

```go
// doCall handles the single call for a key.
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	// 调用函数
    c.val, c.err = fn()
    // 通知函数调用结束
	c.wg.Done()
    
    // 从g.m中移除
	g.mu.Lock()
	if !c.forgotten {
		delete(g.m, key)
	}
    // 如果存在异步调用，通知执行结果
	for _, ch := range c.chans {
		ch <- Result{c.val, c.err, c.dups > 0}
	}
	g.mu.Unlock()
}
```

```go
// Forget tells the singleflight to forget about a key.  Future calls
// to Do for this key will call the function rather than waiting for
// an earlier call to complete.
// 从g.m中移除指定key的函数调用
func (g *Group) Forget(key string) {
   g.mu.Lock()
   if c, ok := g.m[key]; ok {
      c.forgotten = true
   }
   delete(g.m, key)
   g.mu.Unlock()
}
```



### syncmap

`syncmap`提供了一个并发安全的`map`实现，已经加入到了标准库中

```go
// Map is a concurrent map with amortized-constant-time loads, stores, and deletes.
// It is safe for multiple goroutines to call a Map's methods concurrently.
//
// The zero Map is valid and empty.
//
// A Map must not be copied after first use.
type Map struct {
   mu sync.Mutex
   // 查询时会先从read中查询，如果没有才到dirty中查询
   read atomic.Value // readOnly
   
   dirty map[interface{}]*entry
   // 记录到dirty中查询的次数，当达到一定阈值，会使用dirty作为新的read
   misses int
}

// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
	m       map[interface{}]*entry // entry保存value
    // 是否dirty中包含m中不存在的key
	amended bool // true if the dirty map contains some key not in m.
}

// An entry is a slot in the map corresponding to a particular key.
// readOnly虽然是只读的，但是entry可以通过cas更新p字段
type entry struct {
	p unsafe.Pointer // *interface{}
}
```

```go
// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	// 先尝试直接从read中查找，readOnly是只读的，因此并发访问安全
    read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
    // 如果查询不到并且dirty中包含read中不存在的key，则到dirty中查找
	if !ok && read.amended {
        // 需要加锁
		m.mu.Lock()
		// 首先先再次从read中查找一遍，防止加锁过程中，其他协程触发了read的更新
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
        // 如果read中没有，并且dirty包含read中没有的key，从dirty中查找
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// 更新misses字段，并且如果达到阈值，则更新read为dirty
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}
```

```go
// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
    // 首先判断是否read包含要更新的key
	read, _ := m.read.Load().(readOnly)
    // 更新对应的entry，tryStore使用cas操作，保证并发安全
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}
    
	// 如果read中没有，则保存到dirty中
	m.mu.Lock()
    // 首先再次检查一下read
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
        // read中的val已经被删除了，同时保存到dirty中
		if e.unexpungeLocked() {
			// The entry was previously expunged, which implies that there is a
			// non-nil dirty map and this entry is not in it.
			m.dirty[key] = e
		}
		e.storeLocked(&value)
        // if m.dirty == nil, then ok == false
	} else if e, ok := m.dirty[key]; ok {
		e.storeLocked(&value)
	} else {
        // dirty中没有包含read中没有的key，但是read中可能包含dirty中没有的key
        // 这时候的dirty应该还没有初始化
		if !read.amended {
            // 初始化dirty，并将read中没有被标记为删除的kv拷贝到dirty中
 			m.dirtyLocked()
            // 更新read，应该readOnly是只读的，这里重新创建一个readOnly
			m.read.Store(readOnly{m: read.m, amended: true})
		}
        // 把新的kv保存到dirty中
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}
```

```go
// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
	// 先尝试直接从read中查找
    read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
    // read中不存在，并且可能在dirty中
	if !ok && read.amended {
		m.mu.Lock()
        // 再次检查read
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
            // 直接从dirty中删除
			delete(m.dirty, key)
		}
		m.mu.Unlock()
	}
    // read中存在，直接标记为已经删除
	if ok {
		e.delete()
	}
}
```

```go
func (m *Map) Range(f func(key, value interface{}) bool) {
	read, _ := m.read.Load().(readOnly)
	// dirty中包含read中不存在的kv
    if read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		if read.amended {
			// 替换read为dirty
            read = readOnly{m: m.dirty}
			m.read.Store(read)
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}
    
	// 遍历read
	for k, e := range read.m {
		v, ok := e.load()
        // 如果已经标记为删除，跳过
		if !ok {
			continue
		}
		if !f(k, v) {
			break
		}
	}
}
```



