---
title: grpc上手使用
date: 2018-12-21 09:25:47
tags:
	- go
	- grpc
---

# grpc上手使用

### 安装

`golang`版本的`grpc`要求`go`版本要在`1.6`以上

##### install gRPC

使用`go get`命令安装`grpc`包

```sh
$ go get -u google.golang.org/grpc
```

> 由于某些不可逆原因，上面命令会报连接超时，可以到`github`上将项目`clone`到`$GOPATH/src/google.golang.org/`下
>
> ```sh
> $ cd $GOPATH/src/google.golang.org
> $ git clone git@github.com:grpc/grpc-go.git grpc
> ```

##### install Protocol Buffers  v3

`grpc`默认使用`protobuf`作为序列化工具。

1. 打开[Releases](https://github.com/protocolbuffers/protobuf/releases)页面，下载对应平台的`.zip`包`protoc-<version>-<platform>.zip`
2. 解压
3. 添加二进制文件路径导`PATH`环境变量

##### install protoc plugin

安装`golang`版本对应的`protobuf`生成工具

```sh
$ go get -u github.com/golang/protobuf/protoc-gen-go
$ export PATH=$PATH:$GOPATH/bin
```

### 运行demo

进入`example`目录

```sh
$ cd $GOPATH/src/google.golang.org/grpc/examples/helloworld
```

删除原来的`helloworld.pb.go`文件，并使用`protoc`生成自己生成一个

```sh
$ rm helloworld/helloworld.pb.go // 删除原来的helloworld.pb.go文件
$ protoc -I helloworld/ helloworld/helloworld.proto --go_out=plugins=grpc:helloworld // 根据 .proto 文件生成对应的.go文件
```

编写`grpc`接口时，在`.proto`文件定义接口通信数据格式和接口信息，然后通过`protoc`自动生成对应的`go`代码，大大方便了开发

- `-I PATH`：specify the directory in which to search for imports.  May be specified multiple times; directories will be searched in order.  If not given, the current working directory is used.
- `--go_out`：指定输出`go`代码
- `plugins=grpc`：`.proto`中的`service `是`grpc`扩展的功能，需要使用`grpc`插件进行解析才能生成对应的接口定义代码。

运行 `grpc server `和 `grpc client`

```sh
$ go run greeter_server/main.go // 启动grpc server
$ go run greeter_client/main.go // 启动grpc client
```



### 实践

使用`grpc`开发一个简单的求和服务。

##### 定义.proto文件

在项目下创建`proto/sum.proto`文件：

```protobuf
syntax = "proto3"; // 使用 proto3

// java生成选项
option java_multiple_files = true;
option java_package = "io.grpc.examples.helloworld";
option java_outer_classname = "HelloWorldProto";

package proto; // 生成的go所属的package

message SumResp {
    int64 sum = 1;
}

message SumReq {
    int64 a = 1;
    int64 b = 2;
}


service CalcSvc {
    // 每个rpc接口声明都必须有且一个参数和一个返回值
    rpc Sum(SumReq) returns (SumResp) {}
}
```

##### 根据接口描述文件生成源码

进入`proto`目录，执行

```sh
$ protoc sum.proto --go_out=plugins=grpc:.
```

可以看到，在本目录下生成`sum.pb.go`文件，且`package`为`proto`

##### 开发服务端接口

首先查看生成的`sum.pb.go`文件，可以看到根据`sum.proto`文件中的`CalcSvc`接口定义生成了对应的接口：

```go
// CalcSvcServer is the server API for CalcSvc service.
type CalcSvcServer interface {
	// 每个rpc接口声明都必须有且一个参数和一个返回值
	Sum(context.Context, *SumReq) (*SumResp, error)
}
```

开发服务端接口只要就是根据这些接口定义实现具体的业务逻辑

在项目下创建`service/main.go`：

```go
package main

import (
	"context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
	"grpc-demo/proto"
	"log"
	"net"
)

// 类型断言
var _ proto.CalcSvcServer = new(CalcSvc)

type CalcSvc struct{}

func (CalcSvc) Sum(ctx context.Context, req *proto.SumReq) (resp *proto.SumResp, err error) {
    // 建议使用GetA，不要直接使用req.A，可能存在req=nil的情况
	a := req.GetA() 
	b := req.GetB()
	log.Println("request coming ...")
	return &proto.SumResp{
		Sum: a + b,
	}, err
}

func main() {
	lis, err := net.Listen("tcp", ":8888")
	if err != nil {
		log.Fatal(err)
	}
    // 注册服务到gRPC
	s := grpc.NewServer()
	proto.RegisterCalcSvcServer(s, &CalcSvc{})
    // 启用Server Reflection，可以使用gRPC CLI去检查services
    // https://github.com/grpc/grpc-go/blob/master/Documentation/server-reflection-tutorial.md
	reflection.Register(s)
    // 启动服务
	if err := s.Serve(lis); err != nil {
		log.Fatal(err)
	}
}
```

##### 客户端访问

在项目下创建`client/main.go`：

```go 
package main

import (
	"context"
	"google.golang.org/grpc"
	"grpc-demo/proto"
	"log"
)

func main() {
    // 创建gRPC连接
    // WithInsecure option 指定不启用认证功能
	conn, err := grpc.Dial(":8888", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}
    // 创建gRPC client
	client := proto.NewCalcSvcClient(conn)
    // 请求gRPC server
	resp, err := client.Sum(context.Background(), &proto.SumReq{
		A: 5,
		B: 10,
	})
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("5 + 10 = %d", resp.GetSum())
}
```

##### 运行

```go 
$ go run service/main.go
$ go run client/main.go
```



### grpc连接复用

首先修改服务端代码，**添加 `1s` 的睡眠时间**，模拟复杂业务处理场景：

```go
func (CalcSvc) Sum(ctx context.Context, req *proto.SumReq) (resp *proto.SumResp, err error) {
	a := req.GetA()
	b := req.GetB()
	log.Println("request coming ...")
    // 添加 1s 睡眠，模拟接口执行业务逻辑
	time.Sleep(time.Second)
	return &proto.SumResp{
		Sum: a + b,
	}, err
}
```

##### http2多路复用

`grpc`底层使用`http2`协议进行通信，因此单条连接支持多路复用

修改客户端代码：

```go

func main() {
	conn ,err:=grpc.Dial(":8888", grpc.WithInsecure())
	if err!=nil {
		log.Fatal(err)
	}
	client :=proto.NewCalcSvcClient(conn)

	wg := sync.WaitGroup{}
	begin := time.Now()
	concurrentNum := 1000
	wg.Add(concurrentNum)
    
	for i := 0; i < concurrentNum; i++ {
		go func() {
			resp, err := client.Sum(context.Background(), &proto.SumReq{
				A: 5,
				B: 10,
			})
			if err != nil {
				log.Fatal(err)
			}
			log.Printf("5 + 10 = %d", resp.GetSum())
			wg.Done()
		}()
	}
	wg.Wait()
	log.Printf("用时：%v", time.Now().Sub(begin))
}
```

在上面代码中，服务端每次都睡眠`1s`，客户端使用单条连接进行通信，**1000个并发请求总共执行时间为`1.1s`左右**

如果是`2000`个请求，平均在`1.2s`左右，`10000`个请求是`2`s左右。

可见`grpc`本身单条连接可用提供的并发效果足以满足大部分业务场景。

**注意：**上面的`1000`个并发请求并不是单条连接可以同时发起`1000`个请求，而是其内部支持类似`pipeline`的机制。

##### 连接池

接下来不使用`http2`的多路复用，采用连接池的方式来创建请求

首先实现一个连接池：

```go
package main

import (
	"google.golang.org/grpc"
	"sync"
	"time"
)

// 连接池选项
type Options struct {
	Dial        Dialer
	MaxConn     int
	MaxIdle     int
	WaitTimeout time.Duration
}

// 创建连接
type Dialer func() (*grpc.ClientConn, error)

type Pool struct {
	dial    Dialer
	maxConn int // 最大打开连接数
	maxIdle int // 最大空闲连接数

	waitTimeout time.Duration // 等待连接超时时间
    // 等待连接时通过connCh来传输可用连接
	connCh      chan *grpc.ClientConn

	curConnNum int // 记录当前打开的连接数
    // 保存空闲连接
	freeConn   []*grpc.ClientConn
	sync.Mutex
}

// 创建连接池
func NewPool(opts Options) *Pool {
	if opts.MaxConn <= 0 {
		opts.MaxConn = 10
	}
	if opts.MaxIdle <= 0 {
		opts.MaxIdle = 5
	}
	if opts.MaxIdle > opts.MaxConn {
		opts.MaxIdle = opts.MaxIdle
	}

	return &Pool{
		dial:        opts.Dial,
		maxConn:     opts.MaxConn,
		maxIdle:     opts.MaxIdle,
		waitTimeout: opts.WaitTimeout,
		connCh:      make(chan *grpc.ClientConn),
		freeConn:    make([]*grpc.ClientConn, 0, opts.MaxIdle),
	}

}

// 获取连接
func (p *Pool) Get() (conn *grpc.ClientConn) {
	p.Lock()
	// 已经到达最大连接数
	if p.curConnNum >= p.maxConn {
        // 如果等待超时时间为0，直接返回
		if p.waitTimeout == 0 {
			p.Unlock()
			return
		}

		var tm <-chan time.Time
        // 如果等待超时时间小于0，表示无限等待
		if p.waitTimeout > 0 {
			tm = time.After(p.waitTimeout)
		}
		p.Unlock()
        // 等待可用连接或者超时
		select {
		case <-tm:
		case conn = <-p.connCh:
		}
		return
	}
	// 如果存在空闲连接
	if ln := len(p.freeConn); ln > 0 {
		conn = p.freeConn[0]
		p.freeConn[0] = p.freeConn[ln-1]
		p.freeConn = p.freeConn[:ln-1]
	} else { // 创建新的连接
		c, err := p.dial()
		if err != nil {
			conn = nil
		} else {
			p.curConnNum++
			conn = c
		}
	}
	p.Unlock()
	return
}

// 释放连接
func (p *Pool) Put(conn *grpc.ClientConn) error {
	if conn == nil {
		return nil
	}
    // 首先判断是否有其他协程在等待连接
	select {
	case p.connCh <- conn:
		return nil
	default:
	}
	p.Lock()
	defer p.Unlock()
    // 放回空闲连接
	if len(p.freeConn) < p.maxIdle {
		p.freeConn = append(p.freeConn, conn)
		return nil
	}
    // 再次判断是否有等待可用连接
	select {
	case p.connCh <- conn:
		return nil
	default:
        // 关闭连接
		p.curConnNum--
		return conn.Close()
	}
}

// 统计连接池状态
func (p *Pool) Stat() PoolStat {
	p.Lock()
	p.Unlock()
	return PoolStat{
		ConnNum:     p.curConnNum,
		IdleConnNum: len(p.freeConn),
	}
}

type PoolStat struct {
	ConnNum     int
	IdleConnNum int
}

```

接下来，使用该连接池进行测试：

```go
package main

import (
	"context"
	"google.golang.org/grpc"
	"grpc-demo/proto"
	"log"
	"sync"
	"time"
)

func main() {
	opts := Options{
		Dial: func() (*grpc.ClientConn, error) {
			return grpc.Dial(":8888", grpc.WithInsecure())
		},
		WaitTimeout: time.Second * 10,
		MaxConn:     100, // 设置最大连接数为100
		MaxIdle:     50,
	}
	pool := NewPool(opts)
	if pool == nil {
		panic("nil pool")
	}

	wg := sync.WaitGroup{}
	begin := time.Now()
	concurrentNum := 1000
	wg.Add(concurrentNum)
	for i := 0; i < concurrentNum; i++ {
		go func() {

			conn := pool.Get()
			if conn == nil {
				panic("nil conn")
			}
			defer pool.Put(conn)
			client := proto.NewCalcSvcClient(conn)

			resp, err := client.Sum(context.Background(), &proto.SumReq{
				A: 5,
				B: 10,
			})
			if err != nil {
				log.Fatal(err)
			}
			log.Printf("5 + 10 = %d", resp.GetSum())
			wg.Done()
		}()
	}
	wg.Wait()
	log.Printf("用时：%v", time.Now().Sub(begin))
	log.Println(pool.Stat())
}
```

在上面的代码中，每次请求时都从连接池中获取一个连接，请求完成后将其释放。

运行上面代码，**`1000`个并发请求总共需要花费`10.15s`左右**。



### 负载均衡

这里使用`dns`来进行负载均衡进行演示。

我实验机器上面的本机`IP`是`127.0.0.1`，虚拟机`IP`是`192.168.50.12`

首先，修改系统的`hosts`文件，添加：

```
192.168.50.12 www.grpc.com
127.0.0.1 www.grpc.com
```

然后，同时在本地和虚拟机中启动`grpc server`

最后，修改`grpc client`代码：

```go
conn, err := grpc.Dial("dns:///www.grpc.com:8888", grpc.WithInsecure(), grpc.WithBalancerName(roundrobin.Name))
if err != nil {
	log.Fatal(err)
}
client := proto.NewCalcSvcClient(conn)
```

在创建`grpc`连接的时候，使用`dns:///www.grpc.com:8888`，同时指定负载策略为`roundrobin`。

执行`grpc client`，可用看到**两边的`grpc server`都有打印出请求日志**。

`grpc`提供的负载均衡测试是在**请求级别上进行负载均衡**。

`grpc`会同时为每个`grpc server`创建一条连接；每次要发起一个请求的时候，都会根据负载策略选择一条连接来发起请求。