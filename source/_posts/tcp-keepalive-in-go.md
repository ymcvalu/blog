---
title: tcp keepalive in go
date: 2019-07-20 16:04:38
tags:
	- go - tcp
---

### overview

建立`tcp`连接后，可以通过`FIN`包或者`RST`包通知对端关闭连接，其中`FIN`包是正常连接双方四次握手关闭连接过程中发送的，需要接收对方的`ack`，而`RST`包是通知对方立即关闭，也不需要等待对方的`ack`。

但是，如果客户端在连接过程中宕机了，服务端不会收到任何消息，这时候服务端认为连接还存在，但是客户端并不了解。这时候，需要等到服务端向客户端写入消息时，因为客户端并没有这条连接的信息，向服务端返回`rst`包，服务端才会关闭这条连接，并释放相应的资源。

`tcp`连接建立之后，并不是一直处于读写状态，当有一方由于某种原因意外断开，另一方需要等到下一次发送数据时才能关闭连接，而这中间会一直占用系统资源。

这就需要有一种心跳机制，能够定时检查对方连接是否存活，而[`tcp keepalive`](<https://tools.ietf.org/html/rfc1122#page-101>)就算实现该功能的机制。`keepalive`不是`tcp`标准的一部分，并且默认是禁用的，但是目前大多数`tcp`实现都支持。

服务端开启`tcp keepalive`之后，当连接空闲的时候，会定时发送空`body`的`packet`给客户端。根据`tcp`的规范，当客户端接收到一个包之后，需要回复`ACK`，即使之前已经回复过相同的`ACK`了，因此即使客户端没有实现`keepalive`功能也可以正常工作。

在大多数实现中，设置`keepalive`主要有三个参数：

- `tcp_keepalive_time`：间隔多久没有发送数据后，就发送一个心跳包

- `tcp_keepalive_intvl`：发送的心跳包如果没有收到`ack`，间隔多久后，重新发送

- `tcp_keepalive_probes`：最多发送多少个心跳包没有收到回复后，认为对方挂掉了

比如，`tcp_keepalive_time`设置为30s，`tcp_keepalive_intvl`设置为5s，`tcp_keepalive_probes`设置为3，那么当连接空闲30s没有发送数据，会发送第一个心跳包，如果接收到了`ack`，那么会等待空闲30s后再次发送心跳包；而如果没有收到`ack`，5s后会重试，发送第二个心跳包，如果再没有收到`ack`包，那么等待5s后会重试，发送第三个心跳包，如果还没有收到`ack`包，那么就任务对方连接已经挂掉了。

在linux中，可以查看这三个参数的默认值：

```sh
$ cat /proc/sys/net/ipv4/tcp_keepalive_time 
7200
$ cat /proc/sys/net/ipv4/tcp_keepalive_intvl 
75
$ cat /proc/sys/net/ipv4/tcp_keepalive_probes 
9 
```

我们可以通过编辑`/etc/sysctl.conf`，来修改这三个参数的默认值，并使用`sysctl -p`使其生效，程序不需要重启，内核直接生效。

`keepalive`还有一个作用是当使用NAT代理或者防火墙的时候，防止连接因为不活动而被断开。



### code

`go`中的`net.TCPConn`提供了`SetKeepAlive`和`SetKeepAlivePeriod`两个方法

```go
for {
	conn, err := ln.Accept()
	if err != nil {
		log.Printf("failed to accept new conn: %s", err.Error())
		continue
	}

	tcpConn := conn.(*net.TCPConn)
	tcpConn.SetKeepAlive(true) // 开启keepalive
	tcpConn.SetKeepAlivePeriod(time.Second * 30) // 设置tcp_keepalive_time
	go handleConn(conn)
}
```

如果需要设置`tcp_keepalive_intvl`和`tcp_keepalive_probes`两个参数，则需要`syscall`包中的方法：

```go
	fd, err := tcpConn.File()
	if err == nil {
		// 设置tcp_keepalive_probes
		err = syscall.SetsockoptInt(int(fd.Fd()), syscall.IPPROTO_TCP, syscall.TCP_KEEPCNT, 3)
		if err != nil {
			// handle error
		}
		// 设置tcp_keepalive_intvl
		err = syscall.SetsockoptInt(int(fd.Fd()), syscall.IPPROTO_TCP, syscall.TCP_KEEPINTVL, 5)
		if err != nil {
			// handle error
		}
	}
```

上面使用`File.Fd`方法，该方法在`os/file_unix.go`的实现如下：

```go
func (f *File) Fd() uintptr {
	if f == nil {
		return ^(uintptr(0))
	}

	// If we put the file descriptor into nonblocking mode,
	// then set it to blocking mode before we return it,
	// because historically we have always returned a descriptor
	// opened in blocking mode. The File will continue to work,
	// but any blocking operation will tie up a thread.
	if f.nonblock {
        // 设置成阻塞模式
		f.pfd.SetBlocking()
	}

	return uintptr(f.pfd.Sysfd)
}
```

在`go`中，网路连接默认是非阻塞模式，对网路连接的读写会通过[netpoll](<http://mcll.top/2019/04/07/go%E7%BD%91%E7%BB%9Cio%E6%A8%A1%E5%9E%8B%E5%88%86%E6%9E%90/>)来实现非阻塞读写，而当转换成阻塞模式之后，每次读写都会变成一次阻塞的系统调用，从而导致**大量的系统线程**被创建。

`go1.11`之后，添加了`syscall.RawConn`接口，我们可以通过这个接口来规避使用`File.Fd`：

```go
	rawConn, err := tcpConn.SyscallConn()
	if err == nil {
		rawConn.Control(func(fd uintptr) {
			// 设置tcp_keepalive_probes
			err = syscall.SetsockoptInt(int(fd), syscall.IPPROTO_TCP, syscall.TCP_KEEPCNT, 3)
			if err != nil {
				// handle error
			}
			// 设置tcp_keepalive_intvl
			err = syscall.SetsockoptInt(int(fd), syscall.IPPROTO_TCP, syscall.TCP_KEEPINTVL, 5)
			if err != nil {
				// handle error
			}
		})
	}
```



### 参考链接

- [聊聊 TCP 中的 KeepAlive 机制](<https://zhuanlan.zhihu.com/p/28894266>)
- [Notes on TCP keepalive in Go](<https://thenotexpert.com/golang-tcp-keepalive/?utm_campaign=The%20Go%20Gazette&utm_medium=email&utm_source=Revue%20newsletter#an-important-note-on-file-descriptors>)
- [TCP keepalive overview](<http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html>)