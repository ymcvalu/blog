---
title: go中的keepalive
date: 2020-04-03 20:43:07
tags:
    - go
---
在go中我们不需要去手动管理内存的分配和释放。然而有些时候，我们需要手动与与内存管理进行交互。

先来看下面一个出自[go doc](https://golang.org/pkg/runtime/#KeepAlive)的例子
```go
func main() {
	f, err := OpenFile("./keep_alive.go")
	if err != nil {
		log.Fatalf("failed to open file: %s", err.Error())
	}

	byts, err := ReadFile(f.FD)
	if err != nil {
		log.Fatalf("failed to read file: %s", err.Error())
	}

	log.Println("file content:\n", string(byts))
}

type File struct {
	FD   syscall.Handle
	once sync.Once
}

func OpenFile(path string) (*File, error) {
	fd, err := syscall.Open(path, syscall.O_RDONLY, 0)
	if err != nil {
		return nil, err
	}
	file := &File{FD: fd}
	runtime.SetFinalizer(file, func(f *File) {
		f.once.Do(func() {
			syscall.Close(f.FD) // 在gc时关闭文件
		})
	})
	return file, nil
}

func ReadFile(fd syscall.Handle) ([]byte, error) {
	runtime.GC()
	var buf [1024]byte
	n, err := syscall.Read(fd, buf[:])
	return buf[:n], err
}
```
在上面的例子中，我们：

1. 打开了一个文件，并且将文件句柄（windows平台上测试）保存到File中
2. 给File注册一个finalizer，当File被gc回收时，关闭对应的文件
3. 读取文件内容，在读取文件之前，先调用`runtime.GC`手动触发一次gc操作

那么函数的运行结果什么呢？答案是读取的时候会失败：
```sh
$ go run keep_alive.go
2020/04/03 21:15:42 failed to read file: The handle is invalid.
exit status 1
```
为什么呢？因为在`main`中，`f`变量在读取`FD`文件句柄之后就没有再使用到了，因此我们在`ReadFile`中手动触发gc的时候，`f`指向的内存分配就会被gc回收，这时候会调用事先设置的finalizer，把我们打开的文件close掉。这时候我们去读取一个已经关闭的file，自然会报错。

要修复该问题也很简单，runtime包提供了`KeepAlive`方法，用于保证一个变量在执行到该方法之前不会被gc回收掉，我们只需要修改main函数，添加对该方法的调用：
```go
func main() {
    ...

	byts, err := ReadFile(f.FD)
	if err != nil {
		log.Fatalf("failed to read file: %s", err.Error())
	}

    runtime.KeepAlive(f) // 确保f至少可以存活到这里
    
	log.Println("file content:\n", string(byts))
}
```

`KeepAlive`这个方法本身的实现很简单：
```go
//go:noinline
func KeepAlive(x interface{}) {
	if cgoAlwaysFalse {
		println(x)
	}
}
```
可以看到，该方法不会被`inline`。

编译器对该方法有特殊处理，我们可以通过`GOSSAFUNC`来查看编译器对该方法的优化过程：
```sh
$ GOSSAFUNC=main go build main
```
在最终生成的汇编中，该方法会被优化掉，应该是转变成对`PCDATA`的设置了。