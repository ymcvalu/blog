---
title: redis集群搭建
date: 2019-08-30 10:18:43
tags:
  - redis
---

### 环境准备

准备六台服务器（系统为`centos7`），并且安装make、gcc等工具

每个节点需要开放指定端口，为了方便，关闭防火墙，生产不要关闭

```sh
$ service iptables stop # centos 6.x
$ systemctl stop firewalld.service # centos 7.x
```

### 编译redis源码
下载`redis`源码（`redis-5.0.3.tar.gz`)，解压进入项目目录后，执行`make MALLOC=libc`进行编译，编译完成后，可以在`src/`目录下看到编译生成的二进制可执行文件。

### 启动redis实例

##### 编辑配置文件

一个节点可以有多个redis实例，这里每台只配置两个实例。

在每个节点上创建目录`~/redis-cluster/7000`和`~/redis-cluster/7001`两个目录，这里`7000`和`7001`表示实例监听端口，如果一个节点需要部署多个实例，就创建多个不同端口号的目录，然后在每个目录下创建`redis.conf`文件：

```ini
#端口7000，7001
port 7000

#默认ip为127.0.0.1，需要改为其他节点机器可访问的ip，否则创建集群时无法访问对应的端口，无法创建集群
bind 0.0.0.0

#redis后台运行
daemonize yes

#pidfile文件 7000和7001
pidfile /var/run/redis_7000.pid

#开启集群
cluster-enabled yes

#集群的配置，配置文件首次启动自动生成   
cluster-config-file nodes_7000.conf

#请求超时，默认15秒，可自行设置 
cluster-node-timeout 10100

#aof日志开启，有需要就开启，它会每次写操作都记录一条日志
appendonly yes

#默认是yes，只要有结点宕机导致16384个槽不能都可以访问到，整个集群就全部停止服务，所以一定要改为no
cluster-require-full-coverage no
```

##### 启动
在每个节点执行：

```sh
$ for ((i=0;i<2;i++)); do redis-5.0.3/src/redis-server redis-cluster/700$i/redis.conf; done
```

启动时，会为每个实例生成nodeid，并在当前目录生成`nodes_7000.conf`和`nodes_7001.conf`两个文件。

检查进程：

```sh
$ ps -ef | grep redis           //redis是否启动成功
$ netstat -tnlp | grep redis    //监听redis端口
```

### 创建集群
官方提供了`redis-trib.rb`来创建集群，就在`redis-5.0.3/src`目录下，该脚本只需要在集群中任意一个节点上执行一次就行。

##### 安装ruby

如果是`redis-4.x.x`版本，需要安装ruby。

```sh
$ yum -y install ruby ruby-devel rubygems rpm-build
```

在`centos 7`中，安装的`ruby`版本过低，使用下面方法安装：

```sh
$ yum install centos-release-scl-rh
$ yum install rh-ruby23 -y
$ scl  enable  rh-ruby23 bash
$ ruby -v
```

安装`redis`

```sh
$ gem install redis -v 3.3.5
```

##### 集群创建
###### 5.0以下
如果为`5.0`以下版本创建集群，则执行官方的ruby脚本，这个工具能自动检测服务器分配master和slave，`--replicas`指定每个`master`有几个`slave`

```sh
$ redis-4.0.12/src/redis-trib.rb create --replicas 3 10.0.1.50:7000 10.0.1.50:7001 10.0.1.19:7000 10.0.1.19:7001 10.0.1.133:7000 10.0.1.133:7001 10.0.1.169:7000 10.0.1.169:7001 10.0.1.210:7000 10.0.1.210:7001 10.0.1.127:7000 10.0.1.127:7001
```

###### 5.0之后
通过`redis-cli`客户端工具来创建集群，` --cluster-replicas 3`指定每个`master`有3个`slave`
```sh
$ redis-cli --cluster create --replicas 3 10.0.1.50:7000 10.0.1.50:7001 10.0.1.19:7000 10.0.1.19:7001 10.0.1.133:7000 10.0.1.133:7001 10.0.1.169:7000 10.0.1.169:7001 10.0.1.210:7000 10.0.1.210:7001 10.0.1.127:7000 10.0.1.127:7001 --cluster-replicas 3
```
输出：

```
>>> Performing hash slots allocation on 12 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.0.1.169:7000 to 10.0.1.50:7000
Adding replica 10.0.1.210:7000 to 10.0.1.50:7000
Adding replica 10.0.1.127:7000 to 10.0.1.50:7000
Adding replica 10.0.1.50:7001 to 10.0.1.19:7000
Adding replica 10.0.1.133:7001 to 10.0.1.19:7000
Adding replica 10.0.1.169:7001 to 10.0.1.19:7000
Adding replica 10.0.1.19:7001 to 10.0.1.133:7000
Adding replica 10.0.1.210:7001 to 10.0.1.133:7000
Adding replica 10.0.1.127:7001 to 10.0.1.133:7000
M: 0c58af35acebf7988ff8eade95dfc9873eefe89c 10.0.1.50:7000
   slots:[0-5460] (5461 slots) master
S: da377f156086211e31170674caabb348bdb544e3 10.0.1.50:7001
   replicates 1e8cc5ef1f15cea04e94f48610a66ebc8a775d8d
M: 1e8cc5ef1f15cea04e94f48610a66ebc8a775d8d 10.0.1.19:7000
   slots:[5461-10922] (5462 slots) master
S: 903b50ea7ed1524f491d10b6f0be6e16dbfb8d40 10.0.1.19:7001
   replicates d0e3847c6b89140566a180e6a536c053b64ab901
M: d0e3847c6b89140566a180e6a536c053b64ab901 10.0.1.133:7000
   slots:[10923-16383] (5461 slots) master
S: 0f8e1743c272e959cabea082002a2b3e4dc24dd0 10.0.1.133:7001
   replicates 1e8cc5ef1f15cea04e94f48610a66ebc8a775d8d
S: bf03f0fe45fc6d1850f3edfeec918193a61f0bba 10.0.1.169:7000
   replicates 0c58af35acebf7988ff8eade95dfc9873eefe89c
S: 047f288ddf0ec6a92042520aa4b812e531803aed 10.0.1.169:7001
   replicates 1e8cc5ef1f15cea04e94f48610a66ebc8a775d8d
S: 3c1a50c8e195f93d569abcb0bc77b2b8dfcef828 10.0.1.210:7000
   replicates 0c58af35acebf7988ff8eade95dfc9873eefe89c
S: 229ee5af18dab07d5b813acbc9f9eb17b9132dd7 10.0.1.210:7001
   replicates d0e3847c6b89140566a180e6a536c053b64ab901
S: 2b7ad5f50a04b5df6015a06b2be3b07f0384c557 10.0.1.127:7000
   replicates 0c58af35acebf7988ff8eade95dfc9873eefe89c
S: 20172d062bd9b2a7d9c833ab1e653e25bbb6be03 10.0.1.127:7001
   replicates d0e3847c6b89140566a180e6a536c053b64ab901
Can I set the above configuration? (type 'yes' to accept): 
```
可以看到分配了三个主节点，并且每个主节点分配了三个从节点，输入`yes`完成创建：
```
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
..........
>>> Performing Cluster Check (using node 10.0.1.50:7000)
M: 0c58af35acebf7988ff8eade95dfc9873eefe89c 10.0.1.50:7000
   slots:[0-5460] (5461 slots) master
   3 additional replica(s)
S: 0f8e1743c272e959cabea082002a2b3e4dc24dd0 10.0.1.133:7001
   slots: (0 slots) slave
   replicates 1e8cc5ef1f15cea04e94f48610a66ebc8a775d8d
S: 229ee5af18dab07d5b813acbc9f9eb17b9132dd7 10.0.1.210:7001
   slots: (0 slots) slave
   replicates d0e3847c6b89140566a180e6a536c053b64ab901
S: 3c1a50c8e195f93d569abcb0bc77b2b8dfcef828 10.0.1.210:7000
   slots: (0 slots) slave
   replicates 0c58af35acebf7988ff8eade95dfc9873eefe89c
S: 047f288ddf0ec6a92042520aa4b812e531803aed 10.0.1.169:7001
   slots: (0 slots) slave
   replicates 1e8cc5ef1f15cea04e94f48610a66ebc8a775d8d
S: da377f156086211e31170674caabb348bdb544e3 10.0.1.50:7001
   slots: (0 slots) slave
   replicates 1e8cc5ef1f15cea04e94f48610a66ebc8a775d8d
S: 2b7ad5f50a04b5df6015a06b2be3b07f0384c557 10.0.1.127:7000
   slots: (0 slots) slave
   replicates 0c58af35acebf7988ff8eade95dfc9873eefe89c
S: bf03f0fe45fc6d1850f3edfeec918193a61f0bba 10.0.1.169:7000
   slots: (0 slots) slave
   replicates 0c58af35acebf7988ff8eade95dfc9873eefe89c
S: 903b50ea7ed1524f491d10b6f0be6e16dbfb8d40 10.0.1.19:7001
   slots: (0 slots) slave
   replicates d0e3847c6b89140566a180e6a536c053b64ab901
S: 20172d062bd9b2a7d9c833ab1e653e25bbb6be03 10.0.1.127:7001
   slots: (0 slots) slave
   replicates d0e3847c6b89140566a180e6a536c053b64ab901
M: d0e3847c6b89140566a180e6a536c053b64ab901 10.0.1.133:7000
   slots:[10923-16383] (5461 slots) master
   3 additional replica(s)
M: 1e8cc5ef1f15cea04e94f48610a66ebc8a775d8d 10.0.1.19:7000
   slots:[5461-10922] (5462 slots) master
   3 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

### 测试集群

使用客户端工具连接`redis`集群：

```sh
$ redis-5.0.3/src/redis-cli -c -p 7000
```

上面的`-c`表示连接集群，当访问的`key`不在当前节点时，可以重定向到目标节点获取数据：

```
127.0.0.1:7001> get foo
-> Redirected to slot [12182] located at 10.0.1.210:7001
"haha"
```

### 查看集群状态
```sh
10.0.1.210:7001> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:12
cluster_size:3
cluster_current_epoch:13
cluster_my_epoch:13
cluster_stats_messages_ping_sent:4896
cluster_stats_messages_pong_sent:3255
cluster_stats_messages_meet_sent:2
cluster_stats_messages_auth-req_sent:11
cluster_stats_messages_sent:8164
cluster_stats_messages_ping_received:3234
cluster_stats_messages_pong_received:3382
cluster_stats_messages_meet_received:9
cluster_stats_messages_fail_received:2
cluster_stats_messages_auth-ack_received:2
cluster_stats_messages_received:6629
```

### redis集群的数据分片
> Redis Cluster does not use consistent hashing, but a different form of sharding where every key is conceptually part of what we call an **hash slot**.
>
> There are 16384 hash slots in Redis Cluster, and to compute what is the hash slot of a given key, we simply take the CRC16 of the key modulo 16384.
>
> Every node in a Redis Cluster is responsible for a subset of the hash slots, so for example you may have a cluster with 3 nodes, where:
>
> - Node A contains hash slots from 0 to 5500.
> - Node B contains hash slots from 5501 to 11000.
> - Node C contains hash slots from 11001 to 16383.


redis集群没有使用hash一致性来存储来对key进行分配，而是使用哈希槽的概念。

在redis集群中有16384个哈希槽，通过使用`CRC16(key)%16384`来确定key存储在哪个哈希槽中。
redis集群中每个节点可以包含多个哈希槽，比如：
- 节点A：0~5500
- 节点B：5501~11000
- 节点C：11001~16383

##### mult-keys 操作
`redis`集群对`mult-keys`命令的支持有限：

**Redis Cluster supports multiple key operations as long as all the keys involved into a single command execution (or whole transaction, or Lua script execution) all belong to the same hash slot**.The user can force multiple keys to be part of the same hash slot by using a concept called *hash tags*.

一个事务、lua脚本或者`mult-keys`命令只能使用位于同一个哈希槽中的`key`，也就是不支持一个`mult-keys`操作不支持跨哈希槽。可以使用`hash tags`来强制让需要在一个命令中执行的数据存储到同一个哈希槽中。

在key中包含`{TAG}`，则会使用`{}`中的`TAG`来计算该key所属的哈希槽，比如`{foo}k1`和`{foo}k2`将会被分配到同一个哈希槽，这样就可以在`lua`脚本中同时使用`{foo}k1`和`{foo}k2`。否则如果在一个`lua`脚本中同时访问不同哈希槽中的key将会报错。

### codis
和redis集群不同的是，`Codis`采用一层无状态的`proxy`层，将分布式逻辑写在`proxy`上，底层的存储引擎还是`Redis`本身。

`codis`通过`presharding`把数据在概念上分成`1024`个`slot`，然后在`proxy`中将不同`key`的请求转发到不同的机器上，通过`crc32(key)%1024`计算`key`对应的`slot`。

`Codis`支持的`MGET/MSET`无法保证原本单点时的原子语义。 因为`MSET`所参与的`key`可能分不在不同的机器上，如果需要保证原来的语义，也就是要么一起成功，要么一起失败，这样就是一个分布式事务的问题，对于`Redis`来说，并没有`WAL`或者回滚这么一说，所以即使是一个最简单的二阶段提交的策略都很难实现，而且即使实现了，性能也没有保证。所以在`Codis`中使用`MSET/MGET`其实和你本地开个多线程`SET/GET`效果一样，只不过是由`codis`帮忙实现了。

`codis`支持`lua`脚本，但是**仅仅是转发而已，它并不保证你脚本操作的数据是否在正确的节点上。**比如，脚本里涉及操作多个`key`，`Codis`能做的就是将这个脚本分配到参数列表中的第一个`key`的机器上执行。所以这种场景下，你需要自己保证你的脚本所用到的`key`分布在同一个机器上，这里可以采用`hashtag`的方式。