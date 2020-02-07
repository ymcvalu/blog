---
title: iptables入门
date: 2019-05-06 23:38:41
tags:
	- linux
	- iptables
---

# iptables

Netfilter是Linux 2.4.x引入的一个子系统，它作为一个通用的、抽象的框架，提供一整套的hook函数的管理机制，使得诸如数据包过滤、网络地址转换(NAT)和基于协议类型的连接跟踪成为了可能。

netfilter的架构就是在整个网络流程的若干位置放置了一些检测点（HOOK），用户可以在每个检测点上注册一些处理函数进行处理。

而iptables其实是一个命令行工具，位于用户空间，我们可以用这个工具来对netfilter进行配置。

### Doc

- [iptables tutorial](<https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html>)
- `iptables -h`
- `man iptables`

### Tables & Chains

表由链组成，链是一些按顺序排列的规则的列表。比如，默认的 `filter` 表包含 `INPUT`， `OUTPUT` 和 `FORWARD` 3条内建的链，这3条链作用于数据包过滤过程中的不同时间点。

各个表和包含的链关系如下：

| table      | chain                                                         | desc                                                        |
| ---------- | ------------------------------------------------------------- | ----------------------------------------------------------- |
| `raw`      | `PREROUTING`、`OUTPUT`                                        | 关闭nat表上使用的连接追踪机制；内核模块：`iptable_raw`      |
| `filter`   | `INPUT`、`OUTPUT`、`FORWARD`                                  | 负责过滤功能，防火墙；内核模块：`iptable_filter`            |
| `nat`      | `PREROUTING`、 `POSTROUTING`、 `OUTPUT`、 `INPUT(部分支持)`   | 网络地址转换；内核模块：`iptable_nat`                       |
| `mangle`   | `PREROUTING`、 `INPUT`、 `FORWARD`、 `OUTPUT `、`POSTROUTING` | 拆解、修改、重封装报文；内核模块：`iptable_mangle`          |
| `security` |                                                               | 用于强制[访问控制网络](http://lwn.net/Articles/267140/)规则 |

默认情况下，任何链中都没有规则。可以向链中添加自己想用的规则。链的默认规则通常设置为 `ACCEPT`，如果想确保任何包都不能通过规则集，那么可以重置为 `DROP`。默认的规则总是在一条链的最后生效，所以在默认规则生效前数据包需要通过所有存在的规则。用户可以加入自己定义的链，从而使规则集更方便管理，自定义链需要被内置的链引用才能生效。每个链下面可以设置一组规则，执行链时就是执行这组规则。

### Traversing Chains

![图1](/img/iptables1.jpg)

上图描述了网络数据包在netfilter中的处理过程。

比如看上面的`PREROUTING`，可以看到数据包需要先后经过多个表的`PREROUTING`链

我们还可以看到在经过`mangle`表的`PREROUTING`之前会先由`connection tracking`，也就是会跟踪连接，比如执行透明代理的时候，请求的目标端口被修改了，这时候应用层的代理服务需要知道原始请求的端口，因为由连接跟踪模块的存在，这时候可以通过系统调用`getsockopt`来获取。

而数据包首先会经过`raw`表的`PREROUTING`链，才会交由连接跟踪模块处理，它的优先级最高，用户可以通过`RAW`表添加`NOTRACK`目标来禁用连接跟踪：
```sh
$ sudo iptables -t raw -A PREROUTING -p tcp -j NOTRACK
```
当使用`NOTRACK`之后，后续的连接跟踪模块以及其他表的`PREROUTING`链将会被跳过。


数据包通过路径上的每一条链时，链中的每一条规则按顺序匹配；无论何时匹配了一条规则，相应的` target` 动作将会执行。

除了内置的规则链之外，用户还可以定义自己的规则链。内置的链有默认的策略，但是用户自定义的链没有默认的策略。在`jump` 到的自定义链中，若每一条规则都不能提供完全匹配，那么数据包像下图描述的一样返回到调用链，也可以在中途使用`RETURN`返回到调用链；在任何时候，若 `DROP` 的规则实现完全匹配，那么被匹配的数据包会被丢弃，不会进行进一步处理。如果一个数据包在链中被 `ACCEPT`，那么这个包就会被`ACCEPT`，不会再遍历后面的规则。

然而，要注意的是，数据包还会以正常的方式继续遍历其他表中的其他链。

![图2](/img/iptable_subtraverse.jpg)


接下来看一下数据包在内核的 netfilter 的流动：
![](/img/Netfilter-packet-flow.svg)

### Command

```sh
$ iptables -t 表名 <-A/I/D/R> 规则链名 [规则号] <-i/o 网卡名> -p 协议名 <-s 源IP/源子网> --sport 源端口 <-d 目标IP/目标子网> --dport 目标端口 -j 动作
```

###### 规则管理命令

- `-A` or `--append` ：将规则加到`chain`末尾

  ```sh
  $ iptables -t filter -A INPUT -i lo -j DROP #在INPUT链末尾添加规则，拒绝掉来自lo网卡的包
  ```

- `-I` or `--insert` ：在指定位置添加规则，原来位置的规则后移

  ```sh
  $ iptables -t filter -I INPUT 1 -i lo -j DROP #在INPUT链头部添加规则，插入位置从1开始计算
  ```

- `-R` or `--replace` ：替换指定位置规则

  ```sh
  $ iptables -t filter -R INPUT 1 -i lo -j ACCEPT #修改INPUT链头部规则
  ```

- `-D` or `--delete`：删除指定位置规则

  ```sh
  $ iptables -t filter -D INPUT 2 #删除INPUT链第二条规则
  ```
- `-L`：查看规则链
  ```sh
  $ iptables -t nat -L PREROUTING # 查看nat表的PREROUTING链
  ```
###### 链管理命令

- `-P` or `--policy`：改变指定链的默认策略，只有内置的链才有默认策略，自定义链没有默认策略

  ```sh
  $ iptables -P INPUT ACCEPT
  ```

- `-F` or `--flush` ：清空规则链的所有规则，如果省略规则链，则清空表上所有链的规则

- `-N` or `--new`：创建自定义链

- `-X` or `--delete-chain`：删除指定的链，这个链必须没有被其它任何规则引用，而且这条上必须没有任何规则。如果没有指定链名，则会删除该表中所有非内置的链。

- `-E` or `--rename-chain`：用指定的新名字去重命名指定的链。这并不会对链内部照成任何影响。

  ```sh
  $ iptables -E oldName newName
  ```

- `-Z` or `--zero`：把指定链，或者表中的所有链上的所有计数器清零，计数器是规则命中计数。

- `-L` or `--list`：查看指定链或者指定表上的所有规则

###### 规则参数

- `-t` or `--table`：指定操作的表，**如果不指定此选项，默认操作的是 `filter` 表**

- `-p` or `--protocol`：指定协议

- `-i` or `--in-interface`：network interface name，匹配流量流入的网络接口，只对`PREROUTING`、`INPUT`或者`FORWARD`生效；这里的网络接口不一定是网卡，比如`docker0`等虚拟网桥也可以；前缀`!`表示非，比如`! -i 127.0.0.1`表示非本机发送过来的数据包。

- `-o` or `--out-interface`：network interface name，匹配流量输出的网络接口，只对`OUTPUT`、`FORWARD`或`POSTROUTING`生效

- `-s` or `--source`：源地址，`ip`地址或者`CIDR`表示指定范围地址

- `--sport `：匹配来源端口

- `-d` or `--destination`：目标地址，`ip`地址或者`CIDR`表示指定范围地址

- `--dport`：匹配目标端口

- `-j` or `--jump`：规则目标，即满足规则时应该执行什么样的动作。目标可以是内置目标，也可以是用户自定义的链，内置的目标有：

  - `ACCEPT`：接收数据包，如果当前规则匹配成功则结束当前链及父链（如果当前是自定义子链）
  - `DROP`：丢弃数据包，不做任何响应。
  - `REJECT`：拒绝当前包，会返回拒绝数据包。
  - `REDIRECT`：重定向、映射、透明代理。
  - `SNAT`：源地址转换。
  - `DNAT`：目标地址转换。
  - `MASQUERADE`：`IP`伪装（`NAT`），用于`ADSL`。
  - `LOG`：日志记录，继续匹配下一个规则，不会结束当前链。

- `-m` or `--match`：使用扩展包匹配模块，可以使用`man iptables-extensions`命令查看扩展模块

  > iptables can use extended packet matching modules. **These are loaded in two ways: implicitly, when -p or --protocol is specified, or with the -m or --match options, followed by the matching module name**; after these, various extra command line options become available, depending on the specific module. You can specify multiple extended match modules in one line, and you can use the -h or --help options after the module has been specified to receive help specific to that module.

  - `statistic`：基于一些统计条件匹配

    ```sh
    $ iptables -A INPUT -m statistic --mode random --probability 0.5 -s 127.0.0.1 -p icmp -j DROP # 来自本机的ping包，有50%的几率被丢弃
    ```

  - `comment`：允许添加注释（最多256给字符）

    ```sh
    $ iptables -A INPUT -m comment --comment "a comment demo" -j ACCEPT
    ```



### DNAT & SNAT

##### SNAT

`SNAT`: Source Network Address Translation，修改网络包源ip地址。

比如内网机器只有私有ip，无法正常访问外网，可以在网关进行SNAT，将ip包的源地址替换为网关的公网ip，等请求返回的时候，网关再把返回的ip包的目标地址还原为原来的内网ip，然后由网关转发给具体的机器。

`SNAT`是多对一的映射，比如多个内网机器同时映射同一个网关的公网ip，不同内网机器可能使用同一个源端口，系统是通过源IP，源端口，目标ip和目标端口和协议等5元组来区分不同的连接的，因此执行`SNAT`时，除了修改源ip，还需要重新分配源端口号。

系统需要通过`SNAT`表来保存原来的ip/端口与转换后的ip/端口之间的映射关系，以便能够在数据流入流出时进行跟踪。

在容器网络中，当容器内部主动向外部发起网络请求时，需要使用`SNAT`将容器ip替换成主机的ip。

##### DNAT

`DNAT`用于将内网机器的端口映射到外网。当网关接收到数据包时，通过DNAT将目标ip和端口替换成内网机器的ip和端口，然后进行转发。

在容器网络中，容器的端口映射就是使用`DNAT`实现的。通过将容器的端口映射到主机端口上，当由数据包发送到该主机端口时，`netfilter`会将其替换成容器的ip和端口。


### 使用 REDIRECT 实现透明代理
在`service mesh`中，使用`sidecar`模式，每个应用容器都会挂载一个`proxy`容器，所有进出容器的流量都需要先经过`proxy`。

在`isito`中，`proxy`容器内运行的实际上是一个`envoy`进程，在`pod`启动时，会先执行`init-container`，设置`pod`中的`iptables`规则，将所有的进出流量通过`REDIRECT`重定向到`envoy`监听的`15001`端口。

而这就有个问题，`REDIRECT`会修改原数据包的头部字段，而在`envoy`中执行代理转发的时候，我们需要知道原来请求的目标地址。

举个例子，比如我们的应用容器监听了`80`端口和`5555`端口，现在两个端口的请求都转发到`15001`了，那么`envoy`如何区分请求的是哪个端口呢？

这就要用到前面说过的连接跟踪了。`netfilter`会跟踪连接信息，包括原始目标地址，而我们可以通过`getsockopt`系统调用来获取原始的目标地址。

下面是一个`demo`：
```go
package main

import (
	"encoding/binary"
	"log"
	"net"
	"syscall"
	"unsafe"
)

func main() {
	l, err := net.Listen("tcp", ":5555")
	if err != nil {
		panic(err)
	}
	defer l.Close()
	for {
		conn, err := l.Accept()
		if err != nil {
			log.Println(err)
			continue
		}
		go func() {
			defer conn.Close()
			sysc, _ := conn.(*net.TCPConn).SyscallConn()
			sysc.Control(func(fd uintptr) {
				// 实际上填充的内容是linux中定义的c结构体：
 				//struct sockaddr_in {
				// 	sa_family_t    sin_family; // addr family, 2 byte
				//	in_port_t      sin_port;   // port in network byte order, 2 byte
				//  struct in_addr sin_addr;   // addr in network byte order
				//};
				id_addr := [16]byte{}
				_len := uint32(len(id_addr))

				errno := getsockopt(int(fd), syscall.SOL_IP, SO_ORIGINAL_DST, unsafe.Pointer(&id_addr), &_len)
				if errno != 0 {
					log.Println("failed to get original addr")
					return
				}

				ip := net.IP(id_addr[4:8])
				// 返回的是大端序
				port := int(binary.BigEndian.Uint16(id_addr[2:4]))
				log.Printf("目标地址是：%s:%d", ip.String(), port)
			})

		}()
	}
}

const SO_ORIGINAL_DST = 80

func getsockopt(s int, level int, name int, val unsafe.Pointer, vallen *uint32) syscall.Errno {
	_, _, e1 := syscall.Syscall6(syscall.SYS_GETSOCKOPT, uintptr(s), uintptr(level), uintptr(name), uintptr(val), uintptr(unsafe.Pointer(vallen)), 0)
	return e1
}
```

在上面的例子中，我们打开一个socket，监听系统的5555端口，然后设置系统的`iptables`：
```sh
$ sudo iptables -t nat -I PREROUTING 1 -p tcp --dport 5555 -j ACCEPT
$ sudo iptables -t nat -I PREROUTING 2 -p tcp --j REDIRECT --to-port 5555
```
然后运行上面的demo，并使用`telnet`测试连接：
```sh
$ telnet 192.168.50.10 5555
$ telnet 192.168.50.10 5556
$ telnet 192.168.50.10 5557
```

可以看到服务输出：
```sh
$ go run main.go                                        
2020/02/07 08:18:26 目标地址是：192.168.50.10:5555
2020/02/07 08:18:28 目标地址是：192.168.50.10:5556
2020/02/07 08:18:30 目标地址是：192.168.50.10:5557
```

我们成功拿到了请求的原始目标地址

### 参考
- https://github.com/owenliang/go-orig-dst/blob/master/main.go
- https://www.servicemesher.com/istio-handbook/concepts-and-principle/sidecar-injection-deep-dive.html
