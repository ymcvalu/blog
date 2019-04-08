---
title: go网络io模型分析
date: 2019-04-07 19:08:14
tags:
	- go
---

在过去，传统的网络编程模型是多线程模型，在主线程中开启一个网络监听，然后每次有一个客户端进行连接，就会单独开启一个线程来处理这个客户端请求。

然而，如果并发量比较大，服务端就会创建大量的线程，而且会有大量的线程阻塞在网络IO上，频繁的线程上下文切换会占用大量的cpu时间片，严重影响服务性能，而且大量的线程也需要占用大量的系统资源

这样就引出著名的`C10K`问题，如何在单台服务器上支持并发`10K`量级的连接

我们知道，虽然同一时间有大量的并发连接，但是同一时刻，只有少数的连接是可读/写的，我们完全可以只使用一个线程来服务提供服务，这也是目前解决`C10K`问题的主要思路，对应的解决方案叫做**IO多路复用**，现在主流的高性能网络服务器/框架都是基于该网络模型，比如`nginx`、`redis`或者`netty`网络库等。

说到这，就不得不提[`epoll`](<http://man7.org/linux/man-pages/man7/epoll.7.html>)，这是`linux`内核提供的用于实现**IO多路复用**的系统调用，其他操作系统上也有类似的接口，关于`epoll`具体内容网上有一大堆的[资料](<http://man7.org/linux/man-pages/man7/epoll.7.html>)，这里就不重复介绍了

**IO多路复用模型**，也可以称作是**事件驱动模型**，虽然能够有效解决`C10K`问题，但是相对传统的多线程模型也带来了一点复杂性。比如说，在多线程模型下，每个连接独占一个线程，而线程本身有自己的上下文；而如果是IO多路复用模型，需要在一个线程中处理多个连接，而每个需要有自己的上下文，需要开发者手动管理。比如服务端还没有接收到一个完整的协议报文时，我们需要把先前接收的部分内容保存到当前连接上下文中，等到下次其余内容到底时再一起处理。

今天，我们主要来看一下`go`中的网络模型。

在`go`中我们可以像传统的多线程模型那样为每个网络连接单独使用一个`goroutine`来提供服务，但是`goroutine`的资源占用相比系统级线程来说非常小，而且其切换在运行在用户态的，并且只需要交换很少的寄存器，因此`goroutine`的上下文切换代价也是极小的，更重要的是，其底层也是基于`epoll`（linux系统下）来实现事件通知的，因此只需要占用很少的系统级线程。

很明显可以看出，`go`中的网络IO模型是传统多线程模型和IO多路复用模型的结合，既有前者的易用性，又有后者的效率，因此使用`go`可以很容易地开发高性能服务器。

今天我们就来看一下，`go`中的网络IO模型是如何实现的。

### 一切从创建Listener开始

我们从创建`Listener`开始说起。

先看下面代码：

```go
ln,_ :=net.Listen("tcp",":80")
```

我们使用`Listen`来创建一个`Listener`，那么底层具体会发生什么呢？让我们一步一步来揭开

首先查看`net.Listen`方法

```go
func Listen(network, address string) (Listener, error) {
	var lc ListenConfig
	return lc.Listen(context.Background(), network, address)
}
```

可以看到实际上工作的是`ListenConfig.Listen`,我们继续往下看：

```go
func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
    ...
	var l Listener
	la := addrs.first(isIPv4)
	switch la := la.(type) {
	case *TCPAddr:
		l, err = sl.listenTCP(ctx, la)
	...
	return l, nil
}
```

因为我们创建的是`tcp`连接，这里我们只关注`sl.listenTCP`方法，继续往下

```go
func (sl *sysListener) listenTCP(ctx context.Context, laddr *TCPAddr) (*TCPListener, error) {
	fd, err := internetSocket(ctx, sl.network, laddr, nil, syscall.SOCK_STREAM, 0, "listen", sl.ListenConfig.Control)
	if err != nil {
		return nil, err
	}
	return &TCPListener{fd}, nil
}
```

我们看函数第一行，调用了`internetSocket`，很明显里面就是创建实际`socket`的逻辑了，继续往下走

```go
func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
	if (runtime.GOOS == "windows" || runtime.GOOS == "openbsd" || runtime.GOOS == "nacl") && mode == "dial" && raddr.isWildcard() {
		raddr = raddr.toLocal(net)
	}
	family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
	return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr, ctrlFn)
}
```

这里我们只看`linux`的情况，因此继续看`socket`方法：

```go
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
    // 这里是实际创建socket的代码
	s, err := sysSocket(family, sotype, proto)
	if err != nil {
		return nil, err
	}
    // 设置socket选项
	if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}
    // 根据socket创建netFD，netFD是net包对底层socket的封装
	if fd, err = newFD(s, family, sotype, net); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}

	if laddr != nil && raddr == nil {
		switch sotype {
        // 看上面的参数，我们传入的sotype是SOCK_STREAM，因此会走这个分支
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
			if err := fd.listenStream(laddr, listenerBacklog, ctrlFn); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		case syscall.SOCK_DGRAM:
			if err := fd.listenDatagram(laddr, ctrlFn); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		}
	}
	if err := fd.dial(ctx, laddr, raddr, ctrlFn); err != nil {
		fd.Close()
		return nil, err
	}
	return fd, nil
}
```

我们先来看`sysSocket`方法：

```go
func sysSocket(family, sotype, proto int) (int, error) {
    // 这里的socketFunc实际上是创建socket的系统调用
    // 	socketFunc func(int, int, int) (int, error)  = syscall.Socket
    // 注意这里传入的SOCK_NONBLOCK，表明我们创建的是非阻塞的socket
    // 这里的SOCK_CLOEXEC表明在执行fork系统调用时，当执行exec时需要关闭从父进程继承的文件设备
	s, err := socketFunc(family, sotype|syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC, proto)
	switch err {
	case nil:
		return s, nil
	default:
		return -1, os.NewSyscallError("socket", err)
        // 低版本内核不支持创建时指定SOCK_NONBLOCK或者SOCK_CLOEXEC
        // 这时候需要分两步，先创建socket，然后再设置flag
	case syscall.EPROTONOSUPPORT, syscall.EINVAL:
	}

    // 这里需要加锁，与fork操作互斥，防止在创建socket而没有设置`SOCK_CLOEXEC`时执行了fork和exec
	syscall.ForkLock.RLock()
    // 创建socket
	s, err = socketFunc(family, sotype, proto)
	if err == nil {
        // 设置SOCK_COLEXEC
		syscall.CloseOnExec(s)
	}
	syscall.ForkLock.RUnlock()
	if err != nil {
		return -1, os.NewSyscallError("socket", err)
	}
    // 设置非阻塞IO
	if err = syscall.SetNonblock(s, true); err != nil {
		poll.CloseFunc(s)
		return -1, os.NewSyscallError("setnonblock", err)
	}
	return s, nil
}
```

`sysSocket`主要通过系统调用创建了`socket`，**同时设置了`SOCK_NONBLOCK`标志位**，这点非常重要，这里要明确，我们在`go`中使用的网络连接一般都是非阻塞的。关于阻塞IO和非阻塞IO的区别网上有一大堆的资料，这里就不重复说明了。使用非阻塞IO的主要的原因是，**在go中，当使用阻塞系统调用时，当前goroutine对应的底层系统级线程就会被占用，无法与当前g解绑为其他g提供服务**，这样当需要执行其他`g`时就需要创建新的线程来执行

接着来看`netFd.listenStream`

```go
func (fd *netFD) listenStream(laddr sockaddr, backlog int, ctrlFn func(string, string, syscall.RawConn) error) error {
	...
    // 为socket绑定监听的ip和端口
	if err = syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
		return os.NewSyscallError("bind", err)
	}
    // listenFunc func(int, int) error = syscall.Listen
    // 这里的listenFunc实际上是系统调用Listen
    // 开始监听
	if err = listenFunc(fd.pfd.Sysfd, backlog); err != nil {
		return os.NewSyscallError("listen", err)
	}
    // 执行初始化操作
	if err = fd.init(); err != nil {
		return err
	}
	lsa, _ = syscall.Getsockname(fd.pfd.Sysfd)
	fd.setAddr(fd.addrFunc()(lsa), nil)
	return nil
}
```

这里就是常规的绑定监听地址和端口，然后开始监听，这里重要的是`netFD.init`函数，先来看`netFD`的结构：

```go
// Network file descriptor.
type netFD struct {
	pfd poll.FD

	// immutable until Close
	family      int
	sotype      int
	isConnected bool // handshake completed or use of association with peer
	net         string
	laddr       Addr
	raddr       Addr
}

// FD is a file descriptor. The net and os packages use this type as a
// field of a larger type representing a network connection or OS file.
type FD struct {
	// Lock sysfd and serialize access to Read and Write methods.
	fdmu fdMutex

	// System file descriptor. Immutable until Close.
	Sysfd int

	// I/O poller.
	pd pollDesc

	// Writev cache.
	iovecs *[]syscall.Iovec

	// Semaphore signaled when file is closed.
	csema uint32

	// Non-zero if this file has been set to blocking mode.
	isBlocking uint32

	// Whether this is a streaming descriptor, as opposed to a
	// packet-based descriptor like a UDP socket. Immutable.
	IsStream bool

	// Whether a zero byte read indicates EOF. This is false for a
	// message based socket connection.
	ZeroReadIsEOF bool

	// Whether this is a file rather than a network socket.
	isFile bool
}
```

接着看上面的`netFD.init`函数：

```go
func (fd *netFD) init() error {
    // 这里的pfd实际上就是poll.FD，用来表示一个网络连接或者打开的系统文件
	return fd.pfd.Init(fd.net, true)
}
```

我们来看一下`pollFD.Init`：

```go
func (fd *FD) Init(net string, pollable bool) error {
	// We don't actually care about the various network types.
	if net == "file" {
		fd.isFile = true
	}
	if !pollable {
		fd.isBlocking = 1
		return nil
	}
    // 这里又有个init，这里的pd是pollDesc类型
	err := fd.pd.init(fd)
	if err != nil {
		// If we could not initialize the runtime poller,
		// assume we are using blocking mode.
		fd.isBlocking = 1
	}
	return err
}
```

可以看到上面又有个`init`函数，我们先来看一下`fd.pd`对应的`pollDesc`类型：

```go
type pollDesc struct {
	runtimeCtx uintptr // 这个运行时上下文很重要
}
```

我们来看一下`init`函数：

```go
var serverInit sync.Once

func (pd *pollDesc) init(fd *FD) error {
	// 保证runtime_pollServerInit只会执行一次
    serverInit.Do(runtime_pollServerInit)
    // 执行runtime_pollOpen
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	if errno != 0 {
		if ctx != 0 {
			runtime_pollUnblock(ctx)
			runtime_pollClose(ctx)
		}
		return syscall.Errno(errno)
	}
    // 把返回值保存到runtimeCtx中
	pd.runtimeCtx = ctx
	return nil
}
```

上面这个函数才是关键所在，这里涉及到了`runtime_pollServerInit`和`runtime_pollOpen`两个函数，从命名可以很容易看出这两个函数是在`runtime`包中实现的，然后在链接器链接过来的

先来看一下`runtime_pollServerInit`实现：

```go
func poll_runtime_pollServerInit() {
	netpollinit()
	atomic.Store(&netpollInited, 1)
}

func netpollinit() {
    // 执行系统调用创建epoll
    // 先尝试使用create1系统调用
	epfd = epollcreate1(_EPOLL_CLOEXEC)
	if epfd >= 0 {
		return
	}
    // 这边的1024是历史原因，只要大于0就好了
    // 原先epoll底层使用hash表实现，需要传入一个size指定hash表的大小，后面基于rb-tree实现，因此这个参数没有实际意义了，大于0即可
	epfd = epollcreate(1024)
	if epfd >= 0 {
		closeonexec(epfd)
		return
	}
	println("runtime: epollcreate failed with", -epfd)
	throw("runtime: netpollinit failed")
}
```

很简单，就是创建了一个`epoll`

再来看一下`runtime_pollOpen`的实现：

```go
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
	// 分配一个pollDesc，这个pollDesc是runtime的pollDesc，和上面的pollDesc不是同一个东西，但是他们之间又有关联
    pd := pollcache.alloc()
	lock(&pd.lock)
	if pd.wg != 0 && pd.wg != pdReady {
		throw("runtime: blocked write on free polldesc")
	}
	if pd.rg != 0 && pd.rg != pdReady {
		throw("runtime: blocked read on free polldesc")
	}
	pd.fd = fd
	pd.closing = false
	pd.seq++
	pd.rg = 0
	pd.rd = 0
	pd.wg = 0
	pd.wd = 0
	unlock(&pd.lock)

	var errno int32
	errno = netpollopen(fd, pd)
    // 这里返回了pd的地址，也就是poll.pollDesc中的runtimeCtx实际上保存的就是runtime.pollDesc的地址
	return pd, int(errno)
}

func netpollopen(fd uintptr, pd *pollDesc) int32 {
	var ev epollevent
    // 设置需要通知的实际类型，这里设置了边缘触发模式，关于epoll的边缘触发和水平触发模式可以网上有一堆的资料
	ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
    // 可以看到，这里把pollDesc的地址存到了ev.Data中
	*(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
    // 执行epollctl系统调用，添加socket到epoll中
	return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

至此一个`net.Listener`就创建完成了，总结一下主要的逻辑：

1. 创建一个非阻塞`socket`，并执行`bind`和`listen`
2. 如果没有初始化过`runtime`包的`epoll`，则执行初始化，创建一个`epoll`
3. 以边缘触发模式将`socket`添加到`epoll`中
4. 返回封装后的`net.Listener`

### Accept又是如何执行的呢

接下来我们来看一下执行`Accept`时会发生什么

```go
func (l *TCPListener) Accept() (Conn, error) {
	if !l.ok() {
		return nil, syscall.EINVAL
	}
	c, err := l.accept()
	if err != nil {
		return nil, &OpError{Op: "accept", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
	}
	return c, nil
}

func (ln *TCPListener) accept() (*TCPConn, error) {
	fd, err := ln.fd.accept()
	if err != nil {
		return nil, err
	}
	return newTCPConn(fd), nil
}
```

我们上面创建的是一个`TcpListener`，因此自然是执行对应的`Accept`，可以看到是调用`netFD.Accept`：

```go
func (fd *netFD) accept() (netfd *netFD, err error) {
    // 执行poll.FD的Accept方法，获取新的客户端连接
	d, rsa, errcall, err := fd.pfd.Accept()
	if err != nil {
		if errcall != "" {
			err = wrapSyscallError(errcall, err)
		}
		return nil, err
	}
	// 封装netFD
	if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
		poll.CloseFunc(d)
		return nil, err
	}
    // 这里的netFD.init上面分析过了，就是将新的socket加入到epoll中
	if err = netfd.init(); err != nil {
		fd.Close()
		return nil, err
	}
	lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
	netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
	return netfd, nil
}
```

接下来看一下`poll.FD`的`Accept`方法：

```go
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
    // 尝试加锁
	if err := fd.readLock(); err != nil {
		return -1, nil, "", err
	}
	defer fd.readUnlock()
    
	if err := fd.pd.prepareRead(fd.isFile); err != nil {
		return -1, nil, "", err
	}
	for {
        /// 首先尝试直接获取客户端连接
		s, rsa, errcall, err := accept(fd.Sysfd)
		if err == nil { // 获取成功，直接返回
			return s, rsa, "", err
		}
		switch err {
            // 因为我们创建的socket是非阻塞的，当没有新的连接可以accept时会直接返回EAGAIN而不是阻塞
		case syscall.EAGAIN:
            // 如果是可轮询的，表明可以等到epoll事件通知
			if fd.pd.pollable() {
                // 
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}
		case syscall.ECONNABORTED:
			// This means that a socket on the listen
			// queue was closed before we Accept()ed it;
			// it's a silly error, so try again.
			continue
		}
		return -1, nil, errcall, err
	}
}

func accept(s int) (int, syscall.Sockaddr, string, error) {
    // var Accept4Func func(int, int) (int, syscall.Sockaddr, error) = syscall.Accept4
    // 首先使用系统调用accept4获取一个非阻塞的socket
	ns, sa, err := Accept4Func(s, syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC)
	switch err {
	case nil:
		return ns, sa, "", nil
	default: // errors other than the ones listed
		return -1, sa, "accept4", err
	case syscall.ENOSYS: // syscall missing
	case syscall.EINVAL: // some Linux use this instead of ENOSYS
	case syscall.EACCES: // some Linux use this instead of ENOSYS
	case syscall.EFAULT: // some Linux use this instead of ENOSYS
	}
	// 有些内核不支持accept4
	ns, sa, err = AcceptFunc(s)
	if err == nil {
		syscall.CloseOnExec(ns)
	}
	if err != nil {
		return -1, nil, "accept", err
	}
    // 设置非阻塞模式
	if err = syscall.SetNonblock(ns, true); err != nil {
		CloseFunc(ns)
		return -1, nil, "setnonblock", err
	}
	return ns, sa, "", nil
}
```

接着来看`pollDesc.waitRead`实现：

```go
func (pd *pollDesc) waitRead(isFile bool) error {
	return pd.wait('r', isFile)
}

func (pd *pollDesc) wait(mode int, isFile bool) error {
	if pd.runtimeCtx == 0 {
		return errors.New("waiting for unsupported file type")
	}
    // 又是一个runtime包的方法
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res, isFile)
}
```

接着看一下`runtime_pollWait`实现：

```go
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
	err := netpollcheckerr(pd, int32(mode))
	if err != 0 {
		return err
	}
	// As for now only Solaris uses level-triggered IO.
	if GOOS == "solaris" {
		netpollarm(pd, mode)
	}
    // 实际干活的是netpollblock
	for !netpollblock(pd, int32(mode), false) {
		err = netpollcheckerr(pd, int32(mode))
		if err != 0 {
			return err
		}
		// Can happen if timeout has fired and unblocked us,
		// but before we had a chance to run, timeout has been reset.
		// Pretend it has not happened and retry.
	}
	return 0
}

func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
    // 这里如果是'r'模式，则gpp是&pd.rg
    // 'w'模式则是'&pd.wg'
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	// cas操作，设置gpp为pdwait
	for {
		old := *gpp
		if old == pdReady {
			*gpp = 0
			return true
		}
		if old != 0 {
			throw("runtime: double wait")
		}
		if atomic.Casuintptr(gpp, 0, pdWait) {
			break
		}
	}

	// 这里直接执行gopark，将当前协程挂起 ^-^
	if waitio || netpollcheckerr(pd, mode) == 0 {
        // 这里netpollblockcommit会被调用，把当前g的引用保存到gpp中，也就是pollDesc的rg或者wg中
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}
	// be careful to not lose concurrent READY notification
	old := atomic.Xchguintptr(gpp, 0)
	if old > pdWait {
		throw("runtime: corrupted polldesc")
	}
	return old == pdReady
}
```

至此，`Accept`的流程也很清晰了：

1. 首先直接尝试通过`socket`执行`accept`来获取可能的客户端连接
2. 如果此时客户端没有连接，因为`socket`是非阻塞模式，会直接返回`EAGAIN`
3. 调用`runtime.poll_runtime_pollWait`将当前协程挂起，并且根据是等待读还是等待写将当前`g`的引用保存到`pollDesc`中的`rg`或者`wg`中
4. 当有新的客户端连接到来时，`epoll`会通知将当前阻塞的协程恢复，然后重新执行第一步

### 那么epoll的wait又是什么时候调用的呢

我们可以在协程的调度逻辑中看到这样一段代码段：

```go
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
        // 这里的netpoll的参数false表示不阻塞
		if gp := netpoll(false); gp != nil { 
            // 这里获取的可能是一个列表，将后面多余的g加入调度队列，这里调度一次只能调度一个
			injectglist(gp.schedlink.ptr())
            // 设置g为runnable
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}
```

我们来看一下`netpoll`的执行：

```go
func netpoll(block bool) *g {
	if epfd == -1 {
		return nil
	}
	waitms := int32(-1)
    // 调度逻辑中传入的是0
	if !block {
		waitms = 0
	}
	var events [128]epollevent
retry:
    // 执行epoll_wait系统调用
	n := epollwait(epfd, &events[0], int32(len(events)), waitms)
	if n < 0 {
		if n != -_EINTR {
			println("runtime: epollwait on fd", epfd, "failed with", -n)
			throw("runtime: netpoll failed")
		}
		goto retry
	}
    // 这里gp是一个链表
	var gp guintptr
	for i := int32(0); i < n; i++ {
		ev := &events[i]
		if ev.events == 0 {
			continue
		}
		var mode int32
		if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'r'
		}
		if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'w'
		}
		if mode != 0 {
            // 从ev.data取出pollDesc，还记得上面分析过，在加入epoll时会把对应的pollDesc保存到ev.Data中，而协程阻塞时会把g指针保存在pollDesc中的rg或者wg中
			pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
			// 这里执行netpollready，把对应阻塞的g加到gp链表头部
			netpollready(&gp, pd, mode)
		}
	}
	if block && gp == 0 {
		goto retry
	}
	return gp.ptr()
}

func netpollready(gpp *guintptr, pd *pollDesc, mode int32) {
	var rg, wg guintptr
	if mode == 'r' || mode == 'r'+'w' {
        // 这里调用了netpollunblock，获取对应的g
		rg.set(netpollunblock(pd, 'r', true))
	}
	if mode == 'w' || mode == 'r'+'w' {
		wg.set(netpollunblock(pd, 'w', true))
	}
    // 链表设置，将新的g添加到链表头部
	if rg != 0 {
		rg.ptr().schedlink = *gpp
		*gpp = rg
	}
	if wg != 0 {
		wg.ptr().schedlink = *gpp
		*gpp = wg
	}
}

func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
    // 如果是等待读则rg是阻塞的g的引用
    // 如果是等待写则wg是阻塞的g的引用
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	for {
		old := *gpp
		if old == pdReady {
			return nil
		}
		if old == 0 && !ioready {
			// Only set READY for ioready. runtime_pollWait
			// will check for timeout/cancel before waiting.
			return nil
		}
		var new uintptr
		if ioready {
			new = pdReady
		}
        // 状态为ready
		if atomic.Casuintptr(gpp, old, new) {
			if old == pdReady || old == pdWait {
				old = 0
			}
			return (*g)(unsafe.Pointer(old))
		}
	}
}
```

可以看到，在执行协程的调度时，会执行`epoll_wait`系统调用，获取已经准备好的`socket`，并唤醒对应的`goroutine`

除了在调度时会执行`epoll_wait`，在后台线程`sysmon`中也会定时执行`epoll_wait`：

```go
func sysmon() {
	...
	for {
		...
		if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
			atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
			gp := netpoll(false) // non-blocking - returns list of goroutines
			if gp != nil {
				incidlelocked(-1)
				injectglist(gp)
				incidlelocked(1)
			}
		}
		...
	}
}
```

### 大同小异的读写操作

那么接下来，我们来看一下`Read`操作，实际上`Read`最后会执行

```go
func (c *conn) Read(b []byte) (int, error) {
   if !c.ok() {
      return 0, syscall.EINVAL
   }
   n, err := c.fd.Read(b)
   if err != nil && err != io.EOF {
      err = &OpError{Op: "read", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
   }
   return n, err
}

func (fd *netFD) Read(p []byte) (n int, err error) {
	n, err = fd.pfd.Read(p)
	runtime.KeepAlive(fd)
	return n, wrapSyscallError("read", err)
}
```

最后到了`poll.FD`的`Read`方法：

```go
func (fd *FD) Read(p []byte) (int, error) {
    // 这里执行对应的加锁操作
	...
	for {
        // 首先尝试直接读，如果无可读内容，因为是非阻塞模式，会返回EAGAIN
		n, err := syscall.Read(fd.Sysfd, p)
		if err != nil {
			n = 0
			if err == syscall.EAGAIN && fd.pd.pollable() {
                // 这里的waitRead有没有似曾相识？这个方法在accept流程的时候已经分析过了，最后会将当前协程挂起
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}

			// On MacOS we can see EINTR here if the user
			// pressed ^Z.  See issue #22838.
			if runtime.GOOS == "darwin" && err == syscall.EINTR {
				continue
			}
		}
		err = fd.eofError(n, err)
		return n, err
	}
}
```

再来看一下写过程，最后会执行：

```go
func (fd *FD) Write(p []byte) (int, error) {
    // 这里执行对应的加锁操作
	...
    // 记录已经写入字节数
	var nn int
	for {
		max := len(p)
		if fd.IsStream && max-nn > maxRW {
			max = nn + maxRW
		}
		n, err := syscall.Write(fd.Sysfd, p[nn:max])
		if n > 0 {
			nn += n
		}
        // 写入方法与读方法的区别在于，读方法只要读取到内容就会返回
        // 而写入需要将传入的字节切片全部写入才返回
		if nn == len(p) {
			return nn, err
		}
        // 这里的waitWrite和上面的waitRead类似
		if err == syscall.EAGAIN && fd.pd.pollable() {
			if err = fd.pd.waitWrite(fd.isFile); err == nil {
				continue
			}
		}
		if err != nil {
			return nn, err
		}
		if n == 0 {
			return nn, io.ErrUnexpectedEOF
		}
	}
}

// 其实最后都是调用的pd.wait
func (pd *pollDesc) waitWrite(isFile bool) error {
	return pd.wait('w', isFile)
}

// 最终调用runtime_pollWait将当前协程挂起
func (pd *pollDesc) wait(mode int, isFile bool) error {
	if pd.runtimeCtx == 0 {
		return errors.New("waiting for unsupported file type")
	}
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res, isFile)
}
```

### 差点被遗忘的close

接着来看一下`Close`方法，实际执行的是：

```go
func (c *conn) Close() error {
	if !c.ok() {
		return syscall.EINVAL
	}
    // 这里执行netFD.Close
	err := c.fd.Close()
	if err != nil {
		err = &OpError{Op: "close", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
	}
	return err
}

func (fd *netFD) Close() error {
    // 清除finalizer
	runtime.SetFinalizer(fd, nil)
    // 调用poll.FD的Close方法
	return fd.pfd.Close()
}


func (fd *FD) Close() error {
	if !fd.fdmu.increfAndClose() {
		return errClosing(fd.isFile)
	}

	// 这里evict方法唤醒所有阻塞读写的g
	fd.pd.evict()
	// 减少引用，如果引用为0则关闭
	err := fd.decref()

	if fd.isBlocking == 0 {
		runtime_Semacquire(&fd.csema)
	}

	return err
}

func (pd *pollDesc) evict() {
	if pd.runtimeCtx == 0 {
		return
	}
	runtime_pollUnblock(pd.runtimeCtx)
}

func poll_runtime_pollUnblock(pd *pollDesc) {
	lock(&pd.lock)
	if pd.closing {
		throw("runtime: unblock on closing polldesc")
	}
	pd.closing = true
	pd.seq++
	var rg, wg *g
	atomicstorep(unsafe.Pointer(&rg), nil)
    // 获取阻塞的g
	rg = netpollunblock(pd, 'r', false)
	wg = netpollunblock(pd, 'w', false)
	if pd.rt.f != nil {
		deltimer(&pd.rt)
		pd.rt.f = nil
	}
	if pd.wt.f != nil {
		deltimer(&pd.wt)
		pd.wt.f = nil
	}
	unlock(&pd.lock)
	if rg != nil {
        // 调用goready唤醒g
		netpollgoready(rg, 3)
	}
	if wg != nil {
        // 唤醒g
		netpollgoready(wg, 3)
	}
}


func (fd *FD) decref() error {
	// 减少引用，如果引用为0，则返回true
    if fd.fdmu.decref() {
        // 关闭连接
		return fd.destroy()
	}
	return nil
}

func (fd *FD) destroy() error {
	// 调用runtime_pollClose方法
	fd.pd.close()
    // var CloseFunc func(int) error = syscall.Close
    // 这里的CloseFunc就是系统调用close
	err := CloseFunc(fd.Sysfd)
	fd.Sysfd = -1
	runtime_Semrelease(&fd.csema)
	return err
}

func (pd *pollDesc) close() {
	if pd.runtimeCtx == 0 {
		return
	}
	runtime_pollClose(pd.runtimeCtx)
	pd.runtimeCtx = 0
}

func poll_runtime_pollClose(pd *pollDesc) {
	if !pd.closing {
		throw("runtime: close polldesc w/o unblock")
	}
	if pd.wg != 0 && pd.wg != pdReady {
		throw("runtime: blocked write on closing polldesc")
	}
	if pd.rg != 0 && pd.rg != pdReady {
		throw("runtime: blocked read on closing polldesc")
	}
    // 从epoll中删除fd
	netpollclose(pd.fd)
    // 释放pollDesc
	pollcache.free(pd)
}

func netpollclose(fd uintptr) int32 {
	var ev epollevent
    // 系统调用epoll_ctl删除对应的fd
	return -epollctl(epfd, _EPOLL_CTL_DEL, int32(fd), &ev)
}
```

综上，关闭一个连接时：

1. 设置pollDesc相关flag为已关闭，唤醒该连接上阻塞的协程
2. 减少对应poll.FD的引用，如果引用为0，则只需真正的关闭
3. 执行关闭操作，先从epoll删除对应的fd，然后执行close系统调用关闭

### 最后

可以看到，`go`使用非阻塞IO来防止大量系统线程阻塞带来的上下文切换，取而代之的是让轻量级的协程阻塞在IO事件上，然后通过`epoll`来实现IO事件通知，唤醒阻塞的协程。