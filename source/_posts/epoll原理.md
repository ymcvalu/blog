---
title: epoll实现探究
date: 2020-01-09 21:16:16
tags:
    - linux - epoll
---

### IO多路复用
在以前，传统的网络编程是多线程模型，一个线程单独处理一个请求。

然而，线程是很昂贵的资源：

1. 线程的创建和销毁成本很高，linux的线程实际上是特殊的进程；因此通常会使用线程池来减少线程创建和销毁的开销
2. 线程本身占用较大的内存，如果并发比较高，那么光是线程本身占用的内存就很大了
3. 线程上下文切换的成本也比较高，如果频繁切换，可能cpu用于处理线程切换的时间会大于业务执行时间
4. 容易导致系统负载升高

因此，当面对海量连接的时候，传统的多线程模式就无能为力了。

而且：
1. 处理请求的很大比重时间，线程都是在等待网络io，线程很多时候都是在等待io
2. 在推送服务场景下，有海量连接，但是大多数时候只有少数连接是活跃的

这时候，我们就想，能不能有一种机制，可不可以让多个连接复用一个线程？就像cpu的分时复用一样。

答案肯定是可以的，这就是IO多路复用。要能够实现IO多路复用，需要：
1. 非阻塞IO。传统的阻塞IO，当需要等待IO时，会阻塞线程，这还复用个屁？
2. 需要有一个poller，能够轮询连接的io事件。比如当需要往一个连接写入内容，这个连接当前缓冲区是满的，无法写入，我们希望当它的缓存区空闲的时候能够收到通知，从而完成写入

对于非阻塞io，我们可以在`accept`连接的时候指定`SOCK_NONBLOCK`标志，或者在一些低版本的linux内核上，通过`fcntl`系统调用来设置。

而poller，linux提供了[select](http://man7.org/linux/man-pages/man2/select.2.html)、[poll](http://man7.org/linux/man-pages/man2/poll.2.html)和[epoll](http://man7.org/linux/man-pages/man7/epoll.7.html)三种系统接口。本篇文章的主角是epoll。

首先来看select接口：
```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
```
> An fd_set is a fixed size buffer. The Linux kernel allows fd_set of arbitrary size, determining the length of the sets to be checked from the value of nfds. However, the glibc implementation make the fd_set a fixed-size type is fixed in size, with size defined as 1024.
>
>Three independent sets of file descriptors are watched.  The file descriptors listed in readfds will be watched to see if characters become available for reading (more precisely, to see if a read will not block; in particular, a file descriptor is also ready on end-of-file).  The file descriptors in writefds will be watched to see if space is available for write (though a large write may still block). The file descriptors in exceptfds will be watched for exceptional conditions.
> 
> The timeout argument specifies the interval that select() should block waiting for a file descriptor to become ready. If both fields of the timeval structure are zero, then select() returns immediately.  (This is useful for polling.)  If timeout is NULL (no timeout), select() can block indefinitely.

select具有如下几个缺陷：
1. 如果一个fd需要同时监听readab和writable事件，那么需要同时加入到readfds和writefds列表中
2. 每次调用select，都需要将要监听的fd列表从用户空间传到内核空间，内核每次都需要COPY FROM USER，而往往fd列表是固定的内容
3. glibc的实现中，限制了fd_set的大小固定为1024，因此当需要监听的fd列表超过了这个大小，就无能为力了

poll接口相对select接口有所改进：
```c
struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};

// fds is a pollfd array
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

首先，poll使用结构体`pollfd`来描述需要监听的文件，并且可以通过设置`pollfd#events`来描述感兴趣的事件，同时当返回时，可以通过`pollfd#revents`来获取就绪的事件类型。这样就不需要像select接口分别指定三种类型的文件描述符集合了，而且事件类型也更加丰富。而且poll接口对于fds数组的大小也没有限制。

但是，每次调用poll，依然需要传入感兴趣的fd列表。

也正是因为poll的不足，才有了后来epoll的出现。

接下来简单看一下epoll的接口：
```c
// 创建一个epoll实例
int epoll_create(int size);

// 往一个epoll实例中添加/移除/修改对fd的事件监听 
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

// 等待就绪事件
int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);
```

光从接口就可以看到epoll的强大。select和poll才只有一个接口，而epoll自己就足足有3个啊！

epoll本身也是一个file，我们需要先调用`epoll_create`创建它，才能使用，该方法会在内核空间创建一个`epoll`实例，然后返回对应的`fd`。
既然`epoll`是一个file，那么当不再使用的时候，需要被close掉，就像我们close一个打开的文件一样。


当我们通过epoll_create创建一个epoll实例，然后通过epoll_ctl来添加，移除或者修改对文件的就绪事件的监听。比如，当一个连接到达时，我们将其加入epoll，并监听其EPOLLIN/EPOLLOUT/EPOLLRDHUP事件。然后可以多次调用epoll_wait方法来轮询是否有就绪事件，而不用重新加入；当接收到EPOLLRDHUP事件时，移除对该文件的监听，并将其关闭。


epoll也可以被别的epoll轮询！！！

接下来我们基于linux kernel 5 的代码看一下epoll的实现。

### epoll实现

##### epoll的创建
我们首先来看一下epoll的创建过程。

在`fs/eventpoll.c`文件下，可以看到`epoll_create`系统调用的定义，
```c
SYSCALL_DEFINE1(epoll_create, int, size)
{
	if (size <= 0)
		return -EINVAL;

	return do_epoll_create(0);
}
```
`SYSCALL_DEFINEN`是一系列用于定义系统调用的宏，`N`表示系统调用的参数个数。因为`epoll_create`只有一个参数，因此使用的是`SYSCALL_DEFINE1`。

我们可以看到，如果传入的`size`小于或者等于`0`，则会返回`EINVAL`，表示参数错误。在早期的版本中，`epoll`是使用哈希表来管理注册的文件列表，需要传入一个`size`参数；但是现在的版本已经换做红黑树了，该参数已经没有具体的意义了，但是为了兼容性还是保留了下来，使用时传入一个任意大于0的数就好了。

接下来，我们看一下`do_epoll_create`的实现，该方法才是真正干活的主：
```c
static int do_epoll_create(int flags)
{
	int error, fd;
	struct eventpoll *ep = NULL; // eventpoll是epoll实例
	struct file *file; // 前面说过epoll实例本身也是一个file

   // flags只支持设置EPOLL_CLEXEC
	if (flags & ~EPOLL_CLOEXEC)
		return -EINVAL;
    
    // 分配一个epoll实例
	error = ep_alloc(&ep);
	if (error < 0)
		return error;
	
    // 分配一个未使用的文件描述符fd
	fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
	if (fd < 0) {
		error = fd;
		goto out_free_ep;
	}
    
    // 获取一个具有匿名inode的file，并设置file->f_op为eventpoll_fops
    // 同时设置file->private_date为ep
	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC));
	if (IS_ERR(file)) {
		error = PTR_ERR(file);
		goto out_free_fd;
	}
    // 管理epoll实例和file
	ep->file = file;
    // 设置fd对应的file
	fd_install(fd, file);
	return fd;

out_free_fd:
    // 创建失败，释放fd
	put_unused_fd(fd);
out_free_ep:
    // 创建失败，释放内存
	ep_free(ep);
	return error;
}
```
`do_epoll_create`接收一个`flags`参数，如果我们使用`epoll_create`来创建，默认是没有设置任何`flags`的。但是内核后面又添加了`epoll_create1`系统调用，使用该系统调用是可以设置`flags`的，具体可以查看`man`手册。

上面代码已经加了具体的注释了。在linux中正所谓一切皆文件，可以看到当我们创建epoll实例的时候，同时会创建一个`file`。其实`linux`的文件系统正是面向对象的思想，我们可以把`file`看成是一个统一的接口，具体的实现只需要把对应的方法实现注册到`file->f_op`中就好了，而且`file->private_data`保存了实际实现的指针。那些支持接口的语言，背后的实现无外乎也就是这样。

##### epoll实例
既然是创建epoll实例，那我们是不是应该看一下这个epoll实例到底是什么？
```c
struct eventpoll {
	// 资源保护锁
	struct mutex mtx;

	// 阻塞在epoll_wait的等待队列
	wait_queue_head_t wq; 

	// epoll实例本身实现了file->f_op->poll方法，对应的poll等待队列
    // 比如epoll实例本身也可以被添加到其他epoll实例中
	wait_queue_head_t poll_wait;

	// 就绪队列，保存已经就绪的事件
	struct list_head rdllist;

    // 用于保护rdllist和ovflist的锁
	rwlock_t lock;

	// 用于存储添加到epoll实例的fd的红黑树
	struct rb_root_cached rbr; 

	// 当正在转移就绪队列中的事件到用户空间时，这段时期就绪的事件会被暂时加入该队列，等待转移结束再添加到rdllist。
	struct epitem *ovflist; 

	struct wakeup_source *ws;

	// 创建epoll实例的用户
	struct user_struct *user;

    // epoll本身也是一个文件
	struct file *file;

	/* used to optimize loop detection check */
	int visited;
	struct list_head visited_list_link;

#ifdef CONFIG_NET_RX_BUSY_POLL
	/* used to track busy poll napi_id */
	unsigned int napi_id;
#endif
};
```

epoll实例使用红黑树来管理注册的需要监听的fd，关于红黑树的介绍可以参考[该篇](https://mcll.top/2019/12/06/%E7%BA%A2%E9%BB%91%E6%A0%91/)，这里就不介绍了。

每个被添加到epoll实例的fd对应一个`epitem`：
```c
struct epitem {
	union {
		// 红黑树节点，linux内核中红黑树被广泛使用，因此有必要实现成通用结构，通过rb_node连接到rb-tree
        // 当需要访问具体结构体时，通过简单的指针运算以及类型转换就可以了
		struct rb_node rbn;
		/* Used to free the struct epitem */
		struct rcu_head rcu;
	};

	// 用于添加到epoll实例的就绪队列的链表节点
	struct list_head rdllink; 

    // 在将就绪列表中的事件转移到用户空间期间，新的就绪事件会先加入到epoll的ovflist；为什么不复用rdllink字段呢？因为同一个fd，可能还在就绪队列中，但是又有了新的就绪事件了，这时候它既在就绪队列中，也在ovflist中
	struct epitem *next; 

	// 该epitem关联的fd和file信息，用于红黑树中的比较；我们知道，红黑树是二叉搜索树，在查找的时候，是需要比较大小的
	struct epoll_filefd ffd; 

	// Number of active wait queue attached to poll operations 
	int nwait;

	// List containing poll wait queues
	struct list_head pwqlist;

	// 该epitem注册的epoll实例
	struct eventpoll *ep; 

	// used to link this item to the "struct file" items list
	struct list_head fllink;

	// wakeup_source used when EPOLLWAKEUP is set 
	struct wakeup_source __rcu *ws;

	// 该结构体保存epitem对应的fd和感兴趣的事件列表
	struct epoll_event event; 
};
```

当一个`fd`被添加到epoll实例时，epoll实例会调用对应file的poll方法，poll方法有两个作用：

- 设置callback，当文件有新的就绪事件产生时，调用该callback
- 返回文件当前的就绪事件

而当我们从epoll实例移除一个fd时，需要从file中移除注册的callback。

注册callback，实际上就是添加到file的poll wait列表；而这个时候，也会把file的poll wait列表加入到epitem的pwdlist这个列表中。当需要移除注册的callback时，遍历epitem的pwdlist就好了。

接下来看一下 `epoll_filefd`以及在搜索红黑树时，是如果通过该结构体比较大小的：
```c
struct epoll_filefd {
	struct file *file; // 对应file
	int fd;            // 对应的fd
} __packed;            // _packed设置内存对齐为1

/* Compare RB tree keys */
static inline int ep_cmp_ffd(struct epoll_filefd *p1,
			     struct epoll_filefd *p2)
{
	return (p1->file > p2->file ? +1:
	        (p1->file < p2->file ? -1 : p1->fd - p2->fd));
}
```
可以看到，比较的逻辑是：

1. 首先比较file的地址
2. 如果属于同一个file，则比较fd；比如使用dup系统调用，就可以产生两个fd指向同一个file

##### 添加监听文件



##### 更新监听文件

##### 移除监听文件


##### 等待就绪事件



##### 惊群效应


##### Flags


##### Patterns