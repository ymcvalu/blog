---
title: paxos
date: 2019-12-28 23:31:51
tags:
    - paxos - consensus algorithm
---


### Paxos
`Paxos`是分布式领域的宗师`Lamport`提出的基于消息传递、失败容忍的分布式一致性算法。

`Google Chubby`的作者`Mike Burrows`说过，**世界上只有一种一致性算法，那就是`Paxos`**，可见`Paxos`在分布式领域的重要性。

接下来我们就基于`<<Paxos Made Simple>>`这篇论文来学习一下`Paxos`算法。

在该论文中，介绍的实际上是`Base Paxos`算法：假设有一组进程可以提议值，那么通过`Base Paxos`，在这些被提议的值中只会有一个值被选中，这组进程就这个值达到一致。

很重要的一点，`Base Paxos`只能选中一个值。

`Base Paxos`需要保证：
- 如果没有值被提出，就不会有值被选中，不能无中生有
- 最终只能有一个值被选中，否则就脑裂了
- 任意一个进程不能学习到未被选中的值


##### 系统中的角色
在`Paxos`算法中，存在三种角色的`agents`：`proposers`，`acceptors`，`learners`。在具体实现中，一个进程可能同时充当所有角色。

- `proposers`：向`acceptors`提议某个值
- `acceptors`：对`proposers`提出的值进行投票
- `leaders`：学习最后被选中的值

我们假设：
- `agents`可以以任意速度运行，可能停止或者重启；因为`agents`可能会重启，因此需要能够持久化一些关键信息，并在重启之后能够恢复。
- `agents`可以通过发送消息来进行通信。消息可以任意长，可以重复或者丢失，但是不能被篡改。

也就是，这是一个异步的、非拜占庭模型。

##### 算法推导
为了能够在多个进程中选中一个值，最简单的方法是只有一个`acceptor agent`，只要选中其第一个接收到的提议的值。然而，这样就存在单点故障，无法实现失败容忍了。只要这个`agent`挂了，那么系统就无法工作了。

因此，需要同时存在多个`acceptor`。一个`acceptor`可以接受它接收到的提议值。当一个提议值被大多数`acceptor`接受时，那么这个提议值就被选中。

> A proposer sends a proposed value to a set of acceptors. An acceptor may accepte the proposed value. The value is chosen when a large enough set of acceptors have accepted it.

这里的大多数，也就是`majority`应该是多少呢？假设有N个acceptor，那么应该至少需要 `N/2+1`。这样在N个acceptor中，任意两个`majority`会有一个公共的`acceptor`。如果一个`acceptor`只能接受一个值，那么就可以确保不会有超过一个提议值被选中。

不考虑节点故障或者消息丢失的情况下，我们想要即使只有一个`proposer`发表提案，系统最终都能够选中一个值，因此要求满足条件 P1：

**P1：acceptors必须接受他接收到的第一个提案**

然而，同一时间可能会有多个`proposer`同时发起提议，每个`acceptor`可能接收到不同的值，按照条件 P1，每个`acceptor`都接受他们第一个收到的值，从而导致没有一个提议值被`majority`的`acceptors`接受，进而导致最终没有值被选中。


选票瓜分是客端存在的，而为了能够达到最终有一个提案被`majority`接受，因此一个`acceptor`应该能够接受超过一个提案，当然，这肯定是有条件的，不能随便接受。

首先规定，每次`propose`提议一个值的时候，都要**生成一个全局唯一且可比较**的编号。因此，一个`proposer`发表的提案，实际上包含了一个唯一编号和提议值。有了这个唯一编号之后，我们就可以有效的区分不同的提案了，我们可以通过`<number, value>`唯一标识一个提案。

> 一个生成编号的可行方案：
> 
> In a system with n replicas, assign each replica r a unique id i<sub>r</sub> between 0 and n-1. Replica r picks the smallest sequence number s larger than any it has seen, such that s mod n = i<sub>r</sub>.
>
>如果集群成员需要变更，则 `unique id` 可以使用素数替代。

**每次`proposer`在生成一个编号的时候，都要确保新的编号大于它当前所看见的最大编号**。我们可以通过比较提案的编号来判断提案的新旧，更高编号的提案被认为更新。

一个提议值被选中，只有当对应的提案被大多数`acceptor`所接受，也就是对应的提案被选中。

在`paxos`中，**允许多个提案被选中，但是必须保证这些提案都具有相同的提议值，也就是具有相同的`value`**。

这时候，为了保证所有被选中的提案具有相同的值，需要保证条件 P2 成立：

**P2：如果一个包含值 v 的提案被选中，那么所有被选中的具有更高编号的提案也包含值 v**

因为提案的编号是全局可比较的，该条件保证了关键的安全性：只有一个值被选中。


为了被选中，一个提案至少需要被一个`acceptor`接受，因此，可以通过满足条件 P2<sup>a</sup> 来满足条件 P2：

**P2<sup>a</sup>：如果一个具有值v的提案被选中，那么任意一个accptor接受的更高编号的提案要包含值v**

条件 P1 仍然需要维护，以确保能够有提案被选中。

可能当一个值被选中时，某个`acceptor`还没有接收过任何提案，比如在此期间该`acceptor`重启了，刚好错过了`proposer`的请求；这时候如果该`acceptor`收到了一个具有更高编号但是包含了一个不同值的提案，这时候根据条件 P1，`acceptor`需要接受这个值，从而违反了条件 P2<sup>a</sup>。为了同时满足 P1和 P2<sup>a</sup>，需要增强条件 P2<sup>a</sup> 为 P2<sup>b</sub>：

**P2<sup>b</sup>：如果一个包含值v的提案被选中，那么任意proposer提出的更高编号的提案需要包含值v**

因为提案被接受之前需要先被提出，因此满足 P2<sup>b</sup> 也就满足 P2<sup>a</sup>，也就满足了P2。

要发现如何满足 P2<sup>b</sup>，我们可以尝试证明在什么条件下该条件成立。

我们假设一个提案`<m, v>`已经被选中了，现在需要证明任何具有大于`m`的编号`n`的提案也具有值`v`。

我们通过在`n`上使用归纳法来简化证明。我们首先引入额外的归纳假设，每个编号在`[m, n-1]`的提案都具有值`v`。

**因为编号m的提案被选中，因此，必定存在集合C，包含acceptors的majority，集合C中的每个acceptor都接受该提案**。

结合该条件以及归纳假设，我们假设m被选中，则意味着：

> 集合`C`中的每个`acceptor`都曾经接受过一个编号在`[m,n-1]`的提案，并且每个被任意`acceptor`接受的编号在`[m,n-1]`的提案都有值`v`。
>
> Every acceptor in C has accepted a proposal with number in m ..(n − 1), and every proposal with number in m ..(n − 1) accepted by any acceptor has value v. 

`acceptors`中的任意`majority`组成的集合`S`，至少与集合`C`有一个公共的成员，我们可以通过维护条件 P2<sup>c</sup>，来推断出编号`n`的提案具有值`v`这个结论：

**P2<sup>c</sup>：对于一个具有任意编号`n`和值`v`的提案被提议，那么存在集合`S`是`acceptors`中的`majority`，要么满足： a)`S`中没有`acceptor`接受过提案编号小于`n`的提案，即当前还没有编号小于 `n`的提案被选中；或者满足： b)`v`是`S`中的所有`acceptors`接受的所有编号小于`n`的提案中编号最高的那个提案的值，这个提案可能现在已经被选中了，也有可能还没有被选中，但是将来可能被选中。**

由此，我们可以通过维护条件 P2<sup>c</sup>，来满足条件 P2<sup>b</sup>。

为了维护不变式 P2<sup>c</sup>，一个`proposer`在提议编号为`n`的提案之前，必须先了解整个系统中，当前编号小于`n`的提案中具有最高编号的，并且已经被选中或者**将被选中**的提案。了解已经被接受的提案很容易，但是要预测将会被接受的提案很困难。

替代试图去预测未来，`paxos`选择了在提议一个值的时候，使用两阶段提交的方式。

`proposer`在提议一个编号为`n`的提案之前，需要先向`acceptors`发起一个`prepare`请求：请求`acceptor`不要接受编号小于`n`的提案。具体如下：

1. `proposer`选择一个新的提案编号`n`，然后向`acceptor`集合发送`prepare`请求：

   a). 要求承诺不再接受编号小于`n`的的提案

   b). 如果之前曾经接受过编号小于`n`的提案，则返回这些提案（指编号小于`n`的提案）中编号最大的那个提案

2. 如果`proposer`接收到了`acceptors`中的`majority`的响应，则可以提议一个新的编号为`n`的提案，如果这些响应中有包含`acceptor`已经接受的提案，则新的提案的值为这些返回的提案中编号最大的那个提案的值，否则该`proposer`可以自己决定一个值。

`proposer`通过向`acceptor`集合发送请求来提议一个提案，该请求要求这些`acceptor`接受该提案，因此称作`accept`请求。发送`prepare`请求的目标`acceptor`集合不需要与发送`accept`请求的目标集合相同。

​可以看到，`acceptor`可以接受两种请求：`prepare`请求和`accept`请求。`acceptor`可以忽略任何一种请求（本身由于网络的不稳定，请求或者响应消息本身就可能丢失），并不会影响协议的安全性。

因为引入了`prepare`请求，P1 条件需要进行修订：

**P1<sup>a</sup>：一个acceptor在接受一个编号为n的提案时，要求其没有响应过编号大于n的prepare请求**

我们假设所有提案的编号是唯一的，那么现在我们就有一个完整的算法，来满足在多个实例之间确定一个值的算法了。 

这里可以对`prepare`请求进行优化，如果一个`acceptor`在收到一个编号为`n`的`prepare`请求时，已经响应过一个具有更高编号的`prepare`请求，根据承诺这个`acceptor`不会再接受编号为`n`的提案了，因此这个时候可以忽略掉该次`prepare`请求。`acceptor`还可以忽略掉它已经接受的提案的`prepare`请求。

通过这种优化，`acceptor`只需要记住它曾经接受的最高编号的提案以及它已经响应的最高编号的`prepare`请求的编号。为了维护条件 P2<sup>c</sup> ，因此 `acceptor` 必须将这些信息持久化保存，并且再重启之后能够恢复。请注意，`proposer`可以放弃任何一个提案并完全忘记它，只要它不尝试发布另一个具有相同编号的提案。 

最终的算法分为两阶段：

阶段一：

1. `proposer`选择一个编号`n`，然后向`aceptors`中的`majority`发送`prepare`请求
2. 当`acceptor`接收到该编号为`n`的`prepare`请求，如果编号`n`大于所有它已经响应过的`prepare`请求的编号，那么它响应这次`prepare`请求，承诺不会接受任何编号小于`n`的提案，并在返回中携带它已经接受过的编号最高的提案（如果有的话）。

阶段二：

1. 如果`proposer`的编号为`n`的`prepare`请求得到了`acceptors`中的`majority`的响应，那么它可以发送一个`accetpt`请求给`acceptors`的`majority`，该请求提议的提案编号为`n`，值为所有响应中编号最高的提案的值，如果所有响应都不包含提案，说明还没有提案被选中，并且由于`majority of acceptors`对编号为`n`的`prepare`请求的承诺，这时候不会有编号小于`n`的提案被选中，`proposer`可以自己确定一个要提议的值。
2. 如果一个acceptor接收到了一个编号为n的accept请求，如果它还没有响应过一个编号大于n的prepare请求，则接受该提案。

`proposer`可以多次提议，也可以在提议中途丢弃某个提案。如果一个提案已经过时（有更高的编号被提议），`acceptor`可以通知`proposer`丢弃该提案，然后重新提议一个编号更高的提案。

##### 活锁问题
如果有多个`proposer`同时在提议，第一个提交了编号为`n`的提案，第二个提交了编号`n+1`的提案，导致前一个`n`的提案被废弃，然后重新提交了`n+2`的提案，导致`n+1`的提案被废弃。。。这样一直下去，会导致活锁，系统最终无法选中一个值。

`FLP Impossibility`已经证明了:
> no completely asynchronous consensus protocol can tolerate even a single unannounced process death.

我们可以参考`raft`的选主过程，每次`proposer`在下一次提议新的提案时，先等待一个**随机**的时间间隔，这样就可以有效改善活锁问题了。

##### 学习选中的值
前面说过，在`paxos`协议中有三中角色的`agent`，这里还有`learner`没有提及。

`learner`需要学习系统中被选中的值。为了学习被选中的值，`learner`需要知道被大多数`acceptor`接受的提案。

一种直观的做法是，可以让`acceptor`每次接受一个提案时，就通知所有的`learner`。假如系统有`m`个`acceptor`，`n`个`learner`，那么正常情况下一个提案会有`m*n`个请求被发送出去。

另一种做法是，因为是非拜占庭模型，消息不会被篡改。可以让所有的`acceptor`只通知一个`learner`，然后当这个`learner`发现一个提案被选中时，就通知其他的`learner`，这只需要`m+n`个请求。然而这就存在单点故障了。

更通常的做法是，`acceptor`通知一部分`learner`，然后这部分`learner`再去通知其余的`learner`。



### Multi Paxos
`Base Paxos`只能用于在多个进程中就一个值达到一致性，而这肯定是不符合我们的现实需求的。

通常我们会对数据进行分片和多副本存储。

分片的主要目的是为了解决单机的性能瓶颈，从而实现水平扩展，正所谓性能不够机器来凑。

而多副本存储则是为了保证数据安全，防止单点故障。如果每个分片采用单副本存储，如果某台机器故障了，比如磁盘损坏了，那么上面的数据就丢失了。因为我们需要对每个分片进行多副本存储。

那么，如果保证同一个分片的多个副本之间的数据一致性呢？

在`<<Paxos Made Simple>>`中，作者提出了确定性状态机(deterministric state machine)。

对于一个确定性状态机，从一个确定的状态输入相同的命令（也叫做日志），会进入另一个确定的状态。
对于一组确定性状态机，只要执行相同序列的命令，最终都会处于一致的状态。

将每个存储服务实现为确定性状态机，只要保证他们执行相同序列的命令，就可以保证多个副本之间的一致性。

现在，为了保证副本之间的一致性，我们只需要保证这些副本执行一致的命令序列就可以了。

我们可以将每个命令看成是一个日志。对于对序列中的每个日志，可以通过一个`BasePaxos`实例来确定。也就是，序列中第`i`个日志，就由第`i`个`BasePaxos`实例来确定。

然而，每个`BasePaxos`实例，都需要发送`prepare`和`accept`请求。这里可以做一个优化，如果一个`proposer`提议了一个日志被选中，那么后续可以继续由该`proposer`提议，并且跳过`prepare`阶段，直接发送`accept`请求，并且保持提案中的编号`n`不变。而为了能够对这些日志进行区分，需要额外引入一个日志序列号。也就是日志和它的序列号组成提案的`value`。

这种优化是安全的，即使同时存在多个`proposer`发起提议，也不过是退化成`BasePaxos`。

也就是，当一个`proposer`提议成功，可以将他看成是`leader`，并且定期向其他`proposer`发送心跳，这些`proposer`在本地维护`leader`的租期定时器，在该期间不允许发起提议，并且接收到的客户端请求转发到当前`leader`。而即使同时存在多个`leader`，也不过是退化成`BasePaxos`，根据前面的推导，该算法是安全的。

与`raft`协议进行对比，`multiPaxos`中提案的编号和日志的序列号，不就对应`raft`中的`term`和日志的`index`吗？只不过`raft`是强`leader`，只允许`leader`同步日志，从而保证日志的`index`是连续的，这更便于日志的查询和复制。而`multiPaxos`的日志序列号则允许空洞存在。


根据`CAP`理论，一个分布式系统不能同时满足：
- C：强一致性，每次读请求都可以获取到最新的写入，通常指的是线性一致性
- A：可用性，每个请求都可以接收到非错误的返回（不需要保证总是可以读取到最新的写入）
- P：分区容忍性，即使节点之间通信的任意数量的消息丢失或者延时，系统都可以正常运行。

当设计分布式系统时，三者只能择其二。因为网络分区总是客观存在的，因此`P`是无法舍弃的，只能在`CP`和`AP`之间做选择。

根据`Paxos`和`raft`的原理，通过这些协议实现的分布式存储系统，通常都是`CP`系统。





### 参考
- [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem)
- [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [paxosmulti-paxos-algorithm](http://amberonrails.com/paxosmulti-paxos-algorithm/)