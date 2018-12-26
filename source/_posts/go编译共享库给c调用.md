---
title: go编译共享库给c调用
date: 2018-12-19 13:26:16
tags:
	- go

---

### introduce

使用` golang`开发`httpServer`非常方便，如果我们需要在`c`程序中内嵌`httpServer`，可以考虑使用`go`来开发服务模块，然后编译成共享库供`c`调用

### code by go 

##### code

```go
package main

import (
	"net/http"
	"time"
	"log"
)

import "C" // 需要导入`C`才可以生成`.h`文件


// 使用`export`导出函数
//export ServerRun
func ServerRun(_addr *C.char) int {
    // 转换c字符串为golang字符串
    addr :=C.GoString(_addr)
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(resp http.ResponseWriter, req *http.Request) {
		resp.Write([]byte{'H', 'i', '!'})
	})
	if err := http.ListenAndServe(addr, mux); err != nil {
		log.Println(err.Error())
		return -1
	}
	return 0
}

// 内部函数也可以导出
//export wait
func wait() {
	time.Sleep(time.Hour * 1)
}

func main() {}
```

注意点：

- 需要引入`C`包，可以使用`C`包中的`GoString`将`c`的字符串转换为`go`的字符串

- 需要导出的函数，需要使用`//export funcName`标识

- 包内函数也可以导出

- `go`和`c`两者的字符串内存布局不同，如果`go`函数参数声明为`go`的字符串类型，在`c`中相当于一个结构体：

  ```c
  typedef struct { const char *p; ptrdiff_t n; } _GoString_;
  typedef _GoString_ GoString;
  ```

  当要在`c`中调用`go`函数时，需要手动构造字符串，而且还有内存安全的问题。

##### compile

- 动态共享库：运行时动态加载；如果运行时加载失败则报错

  ```sh
  $ go build -buildmode=c-shared -o libtest.so main.go
  ```

  编译完成之后将生成`libtest.so`和`libtest.h`文件

- 静态共享库：编译时静态链接到程序中；生成二进制文件较大

  ```sh
  $ go build -buildmode=c-archive -o test.a main.go
  ```

  编译完成之后将生成`test.h`和`test.a`文件



### use in c 

##### code 

在开始写代码之前，我们要先看一下生成的`test.h`里面的内容：

```c
typedef struct { const char *p; ptrdiff_t n; } _GoString_;
typedef _GoString_ GoString;

typedef long long GoInt64;
typedef GoInt64 GoInt;

extern GoInt ServerRun(char* p0);

extern void wait();
```

可以看到，`.h`文件中包含了外部函数`ServerRun`和`wait`的声明

```c
#include<stdio.h>
#include "test.h"

int main(void){
	// 执行 ServerRun
    if (ServerRun(":8080") != 0){
        printf("failed to start server!");
        return -1;
    }
    return 0;
}
```



##### compile 

- 静态共享库

  ```sh
  $ gcc -pthread -o test main.c test.a 
  ```

  使用静态链接时，需要指定`-pthread`选项 

  > Link with the POSIX threads library.  This option is supported on GNU/Linux targets, most other Unix derivatives, and also on x86 Cygwin and MinGW targets.  On some targets this option also sets flags for the preprocessor, so it should be used consistently for both compilation and linking.

  也可以动态加载`pthread`库

  ```sh
  $ gcc -lpthread -o test main.c test.a
  ```

- 动态共享库

  ```sh
  $ gcc main.c -ltest -L. -I. -o test
  ```

  - `-l`：声明使用到的动态共享库，比如`libtest.so`，则这里传入`test`
  - `-L`：在指定路径中查找共享库；也可以将`.so`文件拷贝到默认共享库目录下
  - `-I`：在指定路径中查找`.h`头部文件



编译之后生成`test`文件，执行`./test`，然后在访问`http://localhost:8080`可以看到返回了`Hi!`内容。

如果使用动态加载，运行前需要先将`libtest.so`文件拷贝到动态加载库默认的加载路径中，或者将当前路径加到`LD_LIBRARY_PATH `环境变量中。
