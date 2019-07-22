---
title: mmap使用
date: 2019-07-22 08:59:39
tags:
	- mmap - go
---

`mmap`可以进程的虚拟内存中创建内存映射，常用来将磁盘文件映射到内存，读写文件转变成读写对应的内存

使用`mmap`读写文件，通常比普通的`read`和`write`系统调用要更快：

- 直接操作内存，避免了系统调用的执行，从而避免用户态/内核态之间的切换开销
- 直接读写内存，不需要经过内核缓冲区，减少数据拷贝

用户访映射的内存时，如果磁盘数据还没有加载到内存中，会触发缺页异常，然后操作系统会以`page`为单位将磁盘文件加载到内存中；如果内存数据被修改了，操作系统会将脏页回写到对应的文件中（脏页回写并不是实时的）。缺页异常的处理开销会比较大，尤其是当物理内存不足时，还会涉及到内存页的淘汰，如果被淘汰的内存页是脏页的话，还需要将其同步到磁盘中。在某些情况下，使用`mmap`映射内存，可能会频繁的触发缺页异常，从而导致性能下降，甚至不如直接使用`read/write`系统调用，比如物理内存很小，而映射的文件很大时。

而且使用`mmap`并不会更改映射文件的大小，当需要更改文件大小时，需要重新映射。



`mmap`除了支持将文件映射到内存，还支持匿名映射，匿名映射不需要底层文件。匿名映射的内存也可以在多个进程间共享，因此可以用来实现进程间的内存共享通信。`go`运行时的内存管理，便是使用匿名映射向操作系统申请了一块大内存，然后基于`tcmalloc`进行内存管理。



### 在go中使用mmap

`mmap`相关的系统调用有：

- mmap：创建内存映射
- munmap：取消内存映射
- msync：同步内存到磁盘文件；内存的修改并不是实时写回磁盘的，当对实时性要求很高，比如数据库的写操作，需要在写内存后手动刷新到磁盘中

接下来通过一个实例看一下如果在go中使用这几个接口：

```go
func checkErr(err error) {
	if err != nil {
		log.Fatal(err)
	}
}

func main() {
    // 获取int类型的大小
	const INT_SIZE = unsafe.Sizeof(int(0))
    // 这里a.txt是我们要映射的底层文件
	fd, err := os.OpenFile("a.txt", os.O_RDWR|os.O_CREATE, os.ModePerm)
	checkErr(err)

	info, _ := fd.Stat()
    // mmap不会更改底层文件的大小，我们要确保访问的映射地址不会超过文件大小，否则会panic
    // 这里设置一下底层文件大小
	if info.Size() != int64(INT_SIZE) {
		fd.Truncate(int64(INT_SIZE))
	}

    // 使用syscall的mmap接口，创建内存映射
    // mmap接口相比posix接口，少了一个addr参数，如果有需要可以使用syscall.Syscall6接口
    // MAP_SHARED指定映射的类型，该模式下对映射空间的更新对其他进程的映射可见，并且会写回底层文件
    // 映射内存会通过[]byte的形式返回
	buf, err := syscall.Mmap(int(fd.Fd()), 0, int(INT_SIZE), syscall.PROT_WRITE|syscall.PROT_READ, syscall.MAP_SHARED)
	// mmap返回之后，底层文件的设备描述符可以立即close掉
    fd.Close() // After the mmap() call has returned, the file descriptor can be closed immediately
	checkErr(err)
	
    // 这里，直接将映射的内存强制类型转换成一个int指针p
	p := (*int)(unsafe.Pointer(&buf[0]))
	// 对指针p的读取就是读取文件内容
	log.Printf("the value saved on file is %d", *p)
    // 对指针p的更新就是更新文件内容
	*p = rand.New(rand.NewSource(time.Now().UnixNano())).Intn(1000)
	// 脏页写回并不是及时的，使用msync系统调用强制将更新写回磁盘文件
	_, _, errno := syscall.Syscall(syscall.SYS_MSYNC, uintptr(unsafe.Pointer(p)), uintptr(INT_SIZE), syscall.MS_SYNC)
	if errno != 0 {
		log.Fatal(syscall.Errno(errno))
	}
	// 使用munmap系统调用需求内存映射
	_, _, errno = syscall.Syscall(syscall.SYS_MUNMAP, uintptr(unsafe.Pointer(p)), uintptr(INT_SIZE), 0)
	if errno != 0 {
		log.Fatal(syscall.Errno(errno))
	}

}
```

可以看到，我们可以直接将`mmap`创建的内存，强制转换成具体的类型值，然后像普通的变量一样操作，连序列化/反序列化都省了。