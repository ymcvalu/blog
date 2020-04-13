---
title: go中连接的Deadline
date: 2020-04-13 23:32:25
tags:
    - go
---
conn可以通过 Deadline 方法来设置`Deadline`。调用`SetDeadline`设置之后，不会自动取消，会一直存在，除非手动取消掉。当达到`SetDeadline`设置的时间到达之后，所有的IO操作都会失败。
```
// A deadline is an absolute time after which I/O operations
// fail with a timeout (see type Error) instead of
// blocking. The deadline applies to all future and pending
// I/O, not just the immediately following call to Read or
// Write. After a deadline has been exceeded, the connection
// can be refreshed by setting a deadline in the future.
//
// An idle timeout can be implemented by repeatedly extending
// the deadline after successful Read or Write calls.
//
// A zero value for t means I/O operations will not time out.
```
`SetDeadline`设置之后，对后面所有的IO都生效，而不是一次IO之后就取消或者重置。当Set的Deadline到达之后，可以重新设置来刷新连接。
如果要实现 idle timeout，可以在每次成功执行一次读或者写之后，设置一下`Deadline`。
如果要实现timeout操作，可以通过重复设置`Deadline`来实现。

`conn`可以单独为`Read`或者`Write`设置`Deadline`，或者同时为`Read`和`Write`设置`Deadline`：
```go
// SetDeadline implements the Conn SetDeadline method.
func (c *conn) SetDeadline(t time.Time) error {
	if !c.ok() {
		return syscall.EINVAL
	}
	if err := c.fd.SetDeadline(t); err != nil {
		return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
	}
	return nil
}

// SetReadDeadline implements the Conn SetReadDeadline method.
func (c *conn) SetReadDeadline(t time.Time) error {
	if !c.ok() {
		return syscall.EINVAL
	}
	if err := c.fd.SetReadDeadline(t); err != nil {
		return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
	}
	return nil
}

// SetWriteDeadline implements the Conn SetWriteDeadline method.
func (c *conn) SetWriteDeadline(t time.Time) error {
	if !c.ok() {
		return syscall.EINVAL
	}
	if err := c.fd.SetWriteDeadline(t); err != nil {
		return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
	}
	return nil
}
```
这些方法最终，都是调用：
```go
// mode: 'r', 'w', 'r'+'w'
func setDeadlineImpl(fd *FD, t time.Time, mode int) error {
	var d int64
	if !t.IsZero() {
		d = int64(time.Until(t))
		if d == 0 {
			d = -1 // don't confuse deadline right now with no deadline
		}
	}
    // 引用计数加1，防止被close掉
	if err := fd.incref(); err != nil {
		return err
	}
	defer fd.decref()
    // runtimeCtx是runtime包中的pollDesc
    // 如果fd不支持poll，无法设置Deadline
	if fd.pd.runtimeCtx == 0 {
		return ErrNoDeadline
	}
    // 调用runtime.pollSetDeadline方法
	runtime_pollSetDeadline(fd.pd.runtimeCtx, d, mode)
	return nil
}
```
在runtime包中对应的实现：
```go
//go:linkname poll_runtime_pollSetDeadline internal/poll.runtime_pollSetDeadline
func poll_runtime_pollSetDeadline(pd *pollDesc, d int64, mode int) {
	lock(&pd.lock)
    // 如果正在close
	if pd.closing {
		unlock(&pd.lock)
		return
	}
    // 原来的deadline
	rd0, wd0 := pd.rd, pd.wd
	combo0 := rd0 > 0 && rd0 == wd0
	if d > 0 {
		d += nanotime()
		if d <= 0 {
			// overflow
			d = 1<<63 - 1
		}
	}
    // 更新新的deadline
	if mode == 'r' || mode == 'r'+'w' {
		pd.rd = d
	}
	if mode == 'w' || mode == 'r'+'w' {
		pd.wd = d
	}
	combo := pd.rd > 0 && pd.rd == pd.wd
	rtf := netpollReadDeadline
	if combo {
		rtf = netpollDeadline
	}
	if pd.rt.f == nil {
		if pd.rd > 0 {
			pd.rt.f = rtf
			// Copy current seq into the timer arg.
			// Timer func will check the seq against current descriptor seq,
			// if they differ the descriptor was reused or timers were reset.
			pd.rt.arg = pd
			pd.rt.seq = pd.rseq
			resettimer(&pd.rt, pd.rd)
		}
	} else if pd.rd != rd0 || combo != combo0 {
		pd.rseq++ // invalidate current timers
		if pd.rd > 0 {
			modtimer(&pd.rt, pd.rd, 0, rtf, pd, pd.rseq)
		} else {
			deltimer(&pd.rt)
			pd.rt.f = nil
		}
	}
	if pd.wt.f == nil {
		if pd.wd > 0 && !combo {
			pd.wt.f = netpollWriteDeadline
			pd.wt.arg = pd
			pd.wt.seq = pd.wseq
			resettimer(&pd.wt, pd.wd)
		}
	} else if pd.wd != wd0 || combo != combo0 {
		pd.wseq++ // invalidate current timers
		if pd.wd > 0 && !combo {
			modtimer(&pd.wt, pd.wd, 0, netpollWriteDeadline, pd, pd.wseq)
		} else {
			deltimer(&pd.wt)
			pd.wt.f = nil
		}
	}
	// If we set the new deadline in the past, unblock currently pending IO if any.
	var rg, wg *g
    // 如果设置的deadline是过去的时间，立即唤醒阻塞的连接
	if pd.rd < 0 || pd.wd < 0 {
		atomic.StorepNoWB(noescape(unsafe.Pointer(&wg)), nil) // full memory barrier between stores to rd/wd and load of rg/wg in netpollunblock
		if pd.rd < 0 {
			rg = netpollunblock(pd, 'r', false)
		}
		if pd.wd < 0 {
			wg = netpollunblock(pd, 'w', false)
		}
	}
	unlock(&pd.lock)
	if rg != nil {
		netpollgoready(rg, 3)
	}
	if wg != nil {
		netpollgoready(wg, 3)
	}
}
```
会给连接设置定时器，定时器过期的时候，会ready对应的协程，并把pollDesc中的rd或者wd设置为-1；

g被唤醒的时候，检查rd或者wd字段，就知道已经deadline了。

go中的Deadline是在runtime层面实现的。

http.Server中有三个字段：
- ReadHeaderTimeout：读取完header的Deadline
- ReadTimeout：读取整个请求的Deadline
- WriteOuter：写响应的Deadline

健壮的服务，应该合理设置这三个值，默认是没有设置的！！！

### 参考
- https://colobu.com/2016/07/01/the-complete-guide-to-golang-net-http-timeouts/