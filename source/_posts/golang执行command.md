---
title: golang执行command
date: 2018-12-27 17:51:05
tags:
	- go
---

### 在golang中使用cmd

在日常开发中，我们有时候需要在程序中调用系统的其他指令来完成任务，比如通过调用`mysqldump`来执行数据库备份。

`golang`提供了`Cmd`，可以很方便的帮助我们来完成这些内容。

> Package exec runs external commands. It wraps os.StartProcess to make it
> easier to remap stdin and stdout, connect I/O with pipes, and do other
> adjustments.
>
> Unlike the "system" library call from C and other languages, the
> os/exec package intentionally does not invoke the system shell and
> does not expand any glob patterns or handle other expansions,
> pipelines, or redirections typically done by shells. The package
> behaves more like C's "exec" family of functions. To expand glob
> patterns, either call the shell directly, taking care to escape any
> dangerous input, or use the path/filepath package's Glob function.
> To expand environment variables, use package os's ExpandEnv.

### demo

```go
func Backup(p string){
    cmd := exec.Cmd{}
	cmd.Path = "/usr/bin/mysqldump"
    cmd.Args = []string{"-uuname", "-ppasswd", `db_name`}
	fd, err := os.OpenFile(p, os.O_CREATE|os.O_TRUNC|os.O_RDWR, 0666)
	if err != nil {
		panic(err)
	}
	defer fd.Close()
	cmd.Stderr = os.Stderr // 重定向错误输出，可以在控制台中看到子进程的错误信息，方便排查
	cmd.Stdout = fd // 重定向cmd的输出，保存到目标文件中
	err = cmd.Run() // Run实际上就是Start和Wait的组合，会等待子进程结束才返回，如果需要异步直接使用Start
	if err != nil {
		panic(err)
	}
}

func Restore(p string) {
	cmd := exec.Cmd{}
	cmd.Path = "/usr/bin/mysql"
    cmd.Args = []string{"-uuname", "-ppasswd", "-Ddb_name"}
	fd, err := os.Open(p)
	if err != nil {
		panic(err)
	}
	defer fd.Close()
	cmd.Stdin = fd // 重定向标准输出为打开文件
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		panic(err)
	}
}
```

如同上面`demo`所示，我们可以通过重定向子进程的`stdin`，`stdout`，`stderr`等

此外，`Cmd`也提供了`Pipe`接口，看一下实现：

```go
func (c *Cmd) StdinPipe() (io.WriteCloser, error) {
	if c.Stdin != nil {
		return nil, errors.New("exec: Stdin already set")
	}
	if c.Process != nil {
		return nil, errors.New("exec: StdinPipe after process started")
	}
	pr, pw, err := os.Pipe() // 创建一条管道
	if err != nil {
		return nil, err
	}
	c.Stdin = pr // 重定向标准输出为读取端
	c.closeAfterStart = append(c.closeAfterStart, pr)
	wc := &closeOnce{File: pw} // 包装管道的写出端
	c.closeAfterWait = append(c.closeAfterWait, wc)
	return wc, nil // 返回
}
```

管道可以用于两个进程之间单向传输数据，一端用于写入，一端用于读取。



### Cmd不是Shell

**当我们在控制台执行命令的时候，实际上我们输入的命令会先通过`shell`进行预处理，然后才会被实际的程序执行**

比如，当我们在控制台执行：

```sh
$ rm -rf *
```

`shell`会先将`*`替换成所有匹配的文件列表，然后再把`-rf`和待删除的文件列表传给`rm`命令执行

而如果通过`Cmd`进行调用，并不会执行这些预处理。

比如：

```go
func main(){
    cmd := exec.Cmd{}
	cmd.Path = "/bin/rm"
    cmd.Args = []string{"-r", "-f", "*"}
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		panic(err)
	}
}
```

`*`会直接作为参数传递给`rm`，而`rm`本身并不会执行模糊匹配，而是把`*`当做普通的文件名对待，如果当前目录没有存在文件名为`*`的文件，则会报错：`No such file or directory`

解决的方法一：

```go
func main(){
    cmd := exec.Cmd{}
	cmd.Path = "/bin/bash"
    cmd.Args = []string{"-c", "rm -rf *"} // 使用 bash -c "rm -rf *"
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		panic(err)
	}
}
```

解决方法二：

```go
func main(){
    fs, _ := filepath.Glob("*") // 获取匹配`*`的文件列表
    cmd := exec.Cmd{}
	cmd.Path = "/bin/rm"
    cmd.Args = []string{"-r", "-f"}
    cmd.Args = append(cmd.Args, fs...)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		panic(err)
	}
}
```

### 子进程结束问题
当我们使用commad启动一个子进程后，如果这个子进程又启动了一个子进程：
```
当前进程 --> 子进程 --> 孙子进程
```
如果我们直接kill这个子进程，这个孙子进程并不会被跟着`kiil`掉，而是会被1号进程(`init`)进程继承，成为1号进程的子进程。

如果需要将孙子进程一起kill掉，可以在创建子进程时给他分配一个独立的pgid，然后kill整个进程组。
