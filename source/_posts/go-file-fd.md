---
title: 在go中获取连接的文件描述符，你掉坑里了吗
date: 2020-01-07 20:13:41
tags:
    - go
---

在`go`中，我们有时候想要拿到一条连接对应的`fd`，我们可能会这样写：
```go
    ln, _ := net.Listen("tcp", ":8080")
    defer ln.Close()

    tcpln := ln.(*net.TCPListener)

    cf, _ := tcpln.File()
    defer cf.Close()
 
    fd := cf.FD() // 拿到对应的fd
```
可以看到，在上面的代码中，我们分别调用了两次`Close`。这是因为`File`方法，实际上执行的是`dup`系统调用。

![](/img/openfile.png)

在表示进程的`task_struct`中，有一个`files`字段：
```c
struct task_struct {
    ...

	/* Open file information: */
	struct files_struct		*files;
    ...
}
```
`files_struct`用于保存当前进程打开的文件信息。

当前进程打开的文件，会有一个对应的`file`结构体，而在`files_struct`有一个`files *`类型的数组，保存当前进程打开的所有文件的`file`结构体的地址。我们在用户空间中使用到的文件描述符`fd`，实际上是该数组中的下标。当需要访问某个文件时，通过`fd`在与数组起始地址进行指针运算，就可以得到对应的`file`结构体的地址了。


该数组的前三个默认是：
- `0`：标准输入
- `1`：标准输出
- `2`：标准错误输出

替换这三个的内容，就可以实现重定向了。

一个打开的文件可以被多个进程同时引用，用于实现共享文件；也可以通过`dup`系统调用来实现同一个进程同时引用同一个文件多次：
> The dup() system call creates a copy of the file descriptor oldfd, using the lowest-numbered unused file descriptor for the new descriptor. After a successful return, the old and new file descriptors may be used interchangeably. They refer to the same open file description and thus share file offset and file status flags.

当打开一个文件时，会对应有一个`file`结构体来表示，也就是上面说的`file description`，而`file descriptor`对应的就是我们所说的`fd`。

`file`中有一个字段用来表示引用次数，当调用`close`时，会先把引用次数减`1`，只有当不再被引用时才执行真正的`close`。

回到开头的例子中，因为`File`方法内部实际上是执行`dup`系统调用，因此当前进程现在会有两个`fd`指向同一个打开的连接，因此如果要真正关闭该连接，需要分别执行两次`close`方法。

接下来，我们看一下`File`方法的实现：
```go
func (l *TCPListener) File() (f *os.File, err error) {
	if !l.ok() { // 判断文件是否为空
		return nil, syscall.EINVAL
	}
	f, err = l.file() // 真正干活的方法
	if err != nil {
		return nil, &OpError{Op: "file", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
	}
	return
}
```
可以看到，实际上调用的是`TCPListener#file`方法，接下来看一下该方法：
```go
func (ln *TCPListener) file() (*os.File, error) {
	f, err := ln.fd.dup()  // 这里出现dup了吧
	if err != nil {
		return nil, err
	}
	return f, nil
}

func (fd *netFD) dup() (f *os.File, err error) {
	ns, call, err := fd.pfd.Dup() // 继续看一下
	if err != nil {
		if call != "" {
			err = os.NewSyscallError(call, err)
		}
		return nil, err
	}

	return os.NewFile(uintptr(ns), fd.name()), nil
}

func (fd *FD) Dup() (int, string, error) {
	if err := fd.incref(); err != nil {
		return -1, "", err
	}
	defer fd.decref()
	return DupCloseOnExec(fd.Sysfd) 
}
```
可以看到最终会调用`DupCloseOnExec`方法：
```go
// DupCloseOnExec dups fd and marks it close-on-exec.
func DupCloseOnExec(fd int) (int, string, error) {
	if atomic.LoadInt32(&tryDupCloexec) == 1 {
        // 通过fcntl系统调用执行dup，同时设置 close-on-exec flag
		r0, e1 := fcntl(fd, syscall.F_DUPFD_CLOEXEC, 0) 
		if e1 == nil {
			return r0, "", nil
		}
		switch e1.(syscall.Errno) {
		case syscall.EINVAL, syscall.ENOSYS:
			// 老版本的linux内核
			atomic.StoreInt32(&tryDupCloexec, 0)
		default:
			return -1, "fcntl", e1
		}
    }
    // 老版本内核的fcntl不支持F_DUPFD_CLOEXEC命令
	return dupCloseOnExecOld(fd)
}

// dupCloseOnExecUnixOld is the traditional way to dup an fd and
// set its O_CLOEXEC bit, using two system calls.
func dupCloseOnExecOld(fd int) (int, string, error) {
    // 这里要加fork锁！！！
	syscall.ForkLock.RLock()
	defer syscall.ForkLock.RUnlock()
	newfd, err := syscall.Dup(fd)
	if err != nil {
		return -1, "dup", err
	}
	syscall.CloseOnExec(newfd)
	return newfd, "", nil
}
```
首先，会尝试使用`fcntl`系统调用的`F_DUPFD_CLOEXEC`命令来实现。该方法具有原子性，也就是执行`dup`和设置`close-on-exec`标志位两个操作是原子的。而老版本的内核并不支持该方法，则需要先执行`dup`系统调用，然后通过`fcntl`系统调用设置`close-on-exec`标志位。

在`go`中，打开的文件默认都会设置`close-on-exec`标志位。因为，`fork`的子进程默认会继承父进程打开的文件列表。而设置了`close-on-exec`标志位的文件，在子进程执行`exec`族函数时会先`close`掉。这样就可以防止文件被子进程继承，而子进程又没有关闭，导致文件泄露。如果子进程确实需要继承父进程的文件，则需要手动指定，这时候会取消`close-on-exec`标志位。

回到前面，在低版本内核中，需要分为两步执行，那么可能在执行`dup`系统调用后，设置`close-on-exec`标志前，在另一个协程中执行了`fork`系统调用，这时候这个文件就不会在子进程执行`exec`时被关闭，从而导致泄露。因此在`dupCloseOnExecOld`这个方法中，需要加`syscall.ForkLock`锁。

接下来，我们看一下获取`fd`的`FD`方法：
```go
func (f *File) Fd() uintptr {
	if f == nil {
		return ^(uintptr(0))
	}

	if f.nonblock {
        // 如果是非阻塞模式，则设置成阻塞模式
		f.pfd.SetBlocking() 
	}

	return uintptr(f.pfd.Sysfd)
}
```

在`go`，网络连接默认是非阻塞模式的。

在非阻塞模式中，当`accept/write/read`没有新的请求可以接受/没有空闲缓冲区可写/缓冲区没有内容可读，会立即返回`EAGEIN`。这时候，`runtime`会将其加入到`epoll`中监听，然后将对应的协程挂起，直到等待的事件到来才将其唤醒。

而如果是阻塞模式，当`accept/write/read`没有新的请求可以接受/没有空闲缓冲区可写/缓冲区没有内容可读，会一直阻塞，不仅会阻塞当前协程，还会把系统线程阻塞掉。一不小心就会导致系统线程激增。

因此在对网络连接使用`FD`方法时，需要格外小心。可以使用下面方法替代：
```go
rawConn, err := tcpConn.SyscallConn()
if err == nil {
	rawConn.Control(func(fd uintptr) {
		// 这里就可以拿到 fd 了
	})
}
```

最后看一下下面完整demo：
```go
package main

import (
	"fmt"
	"log"
	"net"
	"syscall"
)

func main() {
	ln, err := net.Listen("tcp", ":8080")
	if err != nil {
		log.Fatal(err)
    }
    
	tcpln := ln.(*net.TCPListener)

	nf, err := tcpln.File() // 使用dup系统调用，内核中的file的引用加1，返回的是新的fd
	ln.Close()              // close掉原先的连接

	fd := nf.Fd()    // 注意这时候 listener 变成 Blocking
	fmt.Println(fd) // 会输出 4, 而不是 3

	cid, _, err := syscall.Accept(int(fd))
	if err != nil {
		log.Fatal(err)
	}

	syscall.Close(cid) // 关闭请求连接
	syscall.Close(int(fd)) // 关闭listener
}
```