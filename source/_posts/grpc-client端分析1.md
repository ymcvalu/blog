---
title: grpc client端分析1
date: 2019-07-29 14:40:26
tags:
	- grpc - go
---

### grpc客户端连接创建

`grpc`本身提供了服务发现和负载均衡的接口，当需要创建`grpc`连接时，就会使用到这些接口。

我们先来看一下创建`grpc`连接时的主要流程：

![](/img/grpc_client1.png)

![](/img/grpc_client2.png)



##### 服务发现：Resolver

相关接口声明在`resolver/resolver.go`中

```go
// scheme://authority/endpoint
type Target struct {
	Scheme    string
	Authority string
	Endpoint  string
}

// 向grpc注册服务发现实现时，实际上注册的是Builder
type Builder interface {
    // 创建Resolver，当resolver发现服务列表更新，需要通过ClientConn接口通知上层
	Build(target Target, cc ClientConn, opts BuildOption) (Resolver, error)
	Scheme() string
}

type Resolver interface {
    // 当有连接被出现异常时，会触发该方法，因为这时候可能是有服务实例挂了，需要立即实现一次服务发现
	ResolveNow(ResolveNowOption)
	Close()
}

//
type ClientConn interface {
	// 服务列表和服务配置更新回调接口
	UpdateState(State)
	// 服务列表更新通知接口
	NewAddress(addresses []Address)
 	// 服务配置更新通知接口
	NewServiceConfig(serviceConfig string)
}
```

其中`Builder`接口用来创建`Resolver`，我们可以提供自己的服务发现实现，然后将其注册到`grpc`中，其中通过`scheme`来标识，而`Resolver`接口则是提供服务发现功能。当`resover`发现服务列表发生变更时，会通过`ClientConn`回调接口通知上层。

当我们使用`Dial`或者`DialContext`接口创建grpc的客户端连接时，首先会解析参数`target`，然后创建对应的`resolver`：

```go
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
	cc := &ClientConn{
		...
	}
	
    ... 
    
	// resolverBuilder，用于解析target为目标服务列表
	// 如果没有指定resolverBuilder
	if cc.dopts.resolverBuilder == nil {
		// 解析target，根据target的scheme获取对应的resolver
		cc.parsedTarget = parseTarget(cc.target)
 		cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
		// 如果scheme没有注册对应的resolver
		if cc.dopts.resolverBuilder == nil {
            // 使用默认的resolver
			cc.parsedTarget = resolver.Target{
				Endpoint: target, // 这时候参数target就是endpoint，passthrough的实现就是直接返回endpoint，即不使用服务发现功能，参数Dial传进来的地址就是grpc server的地址
			}
            // 获取默认的resolver，也就是passthrough
			cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
		}
	} else {
        // 如果Dial的option中手动指定了需要使用的resolver，那么endpoint也是target
		cc.parsedTarget = resolver.Target{Endpoint: target}
	}
    
	... 
    
	// newCCResolverWrapper方法内调用builder的Build接口创建resolver
	rWrapper, err := newCCResolverWrapper(cc)
	if err != nil {
		return nil, fmt.Errorf("failed to build resolver: %v", err)
	}

	cc.mu.Lock()
	cc.resolverWrapper = rWrapper
	cc.mu.Unlock()
    
 	... 

	return cc, nil
}

// 有效的target：scheme://authority/endpoint
func parseTarget(target string) (ret resolver.Target) {
	var ok bool
	ret.Scheme, ret.Endpoint, ok = split2(target, "://")
	if !ok {
        // 如果没有scheme，则整个target作为endpoint
		return resolver.Target{Endpoint: target}
	}
    // 如果指定了sheme，那么必须有`/`，分割authorigy和endpoint
    // 当不需要指定authorigy，比如使用dnsResolver时:`dns:///www.demo.com`
	ret.Authority, ret.Endpoint, ok = split2(ret.Endpoint, "/")
	if !ok {
		return resolver.Target{Endpoint: target}
	}
	return ret
}

func newCCResolverWrapper(cc *ClientConn) (*ccResolverWrapper, error) {
	// 在DialContext方法中，已经初始化了resolverBuilder
    rb := cc.dopts.resolverBuilder
	if rb == nil {
		return nil, fmt.Errorf("could not get resolver for scheme: %q", cc.parsedTarget.Scheme)
	}

 	// ccResolverWrapper实现resolver.ClientConn接口，用于提供服务列表变更之后的通知回调接口
	ccr := &ccResolverWrapper{
		cc:     cc,
		addrCh: make(chan []resolver.Address, 1),
		scCh:   make(chan string, 1),
	}

	var err error
	// 创建resovler，resovler创建之后，需要立即执行服务发现逻辑，然后将发现的服务列表通过resolver.ClientConn回调接口通知上层
	ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, resolver.BuildOption{DisableServiceConfig: cc.dopts.disableServiceConfig})
	if err != nil {
		return nil, err
	}
	return ccr, nil
}
```



##### Balancer：负载均衡接口

相关接口声明在`balancer/balancer.go`文件中：

```go
// 声明了balancer需要用到的回调接口
type ClientConn interface {
  	// 根据地址创建网路连接
	NewSubConn([]resolver.Address, NewSubConnOptions) (SubConn, error)
    // 移除无效网络连接
	RemoveSubConn(SubConn)
    // 更新Picker，Picker用于在执行rpc调用时执行负载均衡策略，选举一条连接发送请求
	UpdateBalancerState(s connectivity.State, p Picker)
    // 立即触发服务发现
	ResolveNow(resolver.ResolveNowOption)
	Target() string
}

// 根据当前的连接列表，执行负载均衡策略选举一条连接发送rpc请求
type Picker interface {
	Pick(ctx context.Context, opts PickOptions) (conn SubConn, done func(DoneInfo), err error)
}

// Builder用于创建Balancer，注册的时候也是注册builder
type Builder interface {
	Build(cc ClientConn, opts BuildOptions) Balancer
	Name() string
}

type Balancer interface {
    // 当有连接状态变更时，回调
	HandleSubConnStateChange(sc SubConn, state connectivity.State)
    // 当resolver发现新的服务地址列表时调用（有可能地址列表并没有真的更新）
	HandleResolvedAddrs([]resolver.Address, error)
	Close()
}
```

当`Resolver`发现新的服务列表时，最终会调用`Balancer`的`HandleResolvedAddrs`方法进行通知；`Balancer`通过`ClientConn`的接口创建网络连接，然后根据当前的网络连接连接构造新的`Picker`，然后回调`ClientConn.UpdateBalancerState`更新`Picker`。当发送`grpc`请求时，会先执行`Picker`的接口，根据具体的负载均衡策略选举一条网络连接，然后发送`rpc`请求。

当`resolver`发现新的服务列表之后，同通过`NewAddress`回调通知：

```go
// ccResolverWrapper是resolver.ClientConn的实现
func (ccr *ccResolverWrapper) NewAddress(addrs []resolver.Address) {
	if ccr.isDone() {
		return
	}
	
	ccr.curState.Addresses = addrs
	ccr.cc.updateResolverState(ccr.curState)
}

// 更新ClientConn的地址和ServiceConfig
func (cc *ClientConn) updateResolverState(s resolver.State) error {
	cc.mu.Lock()
	defer cc.mu.Unlock()
	
    // ClientConn已经close
	if cc.conns == nil {
		return nil
	}

	...

	// 负载均衡器变更
	var balCfg serviceconfig.LoadBalancingConfig
	// 如果调用Dial时没有手动指定要使用的LoadBalancer
	if cc.dopts.balancerBuilder == nil {
		var newBalancerName string
		// 如果serviceConfig中指定了负载均衡器配置
		if cc.sc != nil && cc.sc.lbConfig != nil {
			newBalancerName = cc.sc.lbConfig.name
			balCfg = cc.sc.lbConfig.cfg
		} else {
			var isGRPCLB bool
            // 判断是否存在grpclb类型的地址
			for _, a := range s.Addresses {
				if a.Type == resolver.GRPCLB {
					isGRPCLB = true
					break
				}
			}
			// 存在grpclb类型的addr，使用grpclb负载均衡器
			if isGRPCLB {
				newBalancerName = grpclbName
                // 如果配置中指定了负载均衡器
			} else if cc.sc != nil && cc.sc.LB != nil { 
				newBalancerName = *cc.sc.LB
			} else {
                // 默认使用PickFirst负载均衡器，每次都使用第一条连接
				newBalancerName = PickFirstBalancerName 
			}
		}
		// 使用新的负载均衡器
		cc.switchBalancer(newBalancerName)
	} else if cc.balancerWrapper == nil { // options指定了balancerBuilder但是还没有初始化
		// 初始化balancer
		cc.curBalancerName = cc.dopts.balancerBuilder.Name()
		cc.balancerWrapper = newCCBalancerWrapper(cc, cc.dopts.balancerBuilder, cc.balancerBuildOpts)
	}

	// 通知Balancer服务列表变更了
cc.balancerWrapper.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
	return nil
}
```

接下来，看一下`Balancer`接收到新的服务列表之后的执行逻辑：

```go
// baseBalancer可以看成是Balancer接口实现的基类，当要实现自己的负载均衡策略时只需要在器基础上实现Picker接口
func (b *baseBalancer) UpdateClientConnState(s balancer.ClientConnState) {
    // 记录新的服务列表
    addrsSet := make(map[resolver.Address]struct{})
	// 为新的地址创建连接
	for _, a := range s.ResolverState.Addresses {
		addrsSet[a] = struct{}{}
        // 如果该地址之前不存在
		if _, ok := b.subConns[a]; !ok {
            // 创建连接
			sc, err := b.cc.NewSubConn([]resolver.Address{a}, balancer.NewSubConnOptions{HealthCheckEnabled: b.config.HealthCheck})
			if err != nil {
				grpclog.Warningf("base.baseBalancer: failed to create new SubConn: %v", err)
				continue
			}
            // 保存到subConns
			b.subConns[a] = sc
            // 设置初始状态
            // 连接状态有：
            //     - IDLE: 未连接
            //     - CONNECTING: 连接中
            //     - READY: 已经连接，可用
            //     - TRANSIENT_FAILURE: 连接异常
            //     - SHUTDOWN: 连接关闭
			b.scStates[sc] = connectivity.Idle
            // 开始连接，更新状态为CONNECTING，然后异步执行连接逻辑
			sc.Connect()
		}
	}
	// 移除无效addr
	for a, sc := range b.subConns {
		// 如果不在新的连接列表中，则需要移除
		if _, ok := addrsSet[a]; !ok {
            // 更新状态为SHUTDOWN，并关闭连接
			b.cc.RemoveSubConn(sc)
			delete(b.subConns, a)
		}
	}
}
```

接下来我们看一下`Connect`的逻辑：

```go
func (ac *addrConn) connect() error {
	ac.mu.Lock()
	// 如果连接已经被移除
	if ac.state == connectivity.Shutdown {
		ac.mu.Unlock()
		return errConnClosing
	}
    
	// 如果状态不是Idle，表示已经执行过connect方法，直接返回
	if ac.state != connectivity.Idle {
		ac.mu.Unlock()
		return nil
	}

	// 更新状态为Connecting，表示正在连接中
	ac.updateConnectivityState(connectivity.Connecting)
	ac.mu.Unlock()

	// 异步启动一个协程去执行网络连接
	go ac.resetTransport()
	return nil
}

func (ac *addrConn) resetTransport() {
	for i := 0; ; i++ {
		// 如果发生重试，说明有可能要连接的服务已经挂掉了，这时候服务列表应该发生变化了，触发一下立即执行服务发现
		if i > 0 {
            // 该方法异步执行，最终调用resolver.ResolveNow
			ac.cc.resolveNow(resolver.ResolveNowOption{})
		}

		ac.mu.Lock()
		// 检查连接是否已经移除
		if ac.state == connectivity.Shutdown {
			ac.mu.Unlock()
			return
		}
        
		// 要连接的addr
		addrs := ac.addrs
        // backoffIdx保存一次连接的重试次数
		backoffFor := ac.dopts.bs.Backoff(ac.backoffIdx)
 
        dialDuration := minConnectTimeout
		if ac.dopts.minConnectTimeout != nil {
			dialDuration = ac.dopts.minConnectTimeout()
		}

		if dialDuration < backoffFor {
			// Give dial more time as we keep failing to connect.
			dialDuration = backoffFor
		}
		 
		connectDeadline := time.Now().Add(dialDuration)
		// 更新状态为Connecting
		ac.updateConnectivityState(connectivity.Connecting)
		ac.transport = nil
		ac.mu.Unlock()

		// 尝试创建连接，只要一个addr成功立即返回
		newTr, addr, reconnect, err := ac.tryAllAddrs(addrs, connectDeadline)
		if err != nil { // 创建失败
			ac.mu.Lock()
			// 检查连接是否已经移除
			if ac.state == connectivity.Shutdown {
				ac.mu.Unlock()
				return
			}
			// 更新状态为TransientFailure，表示连接异常
			ac.updateConnectivityState(connectivity.TransientFailure)

			// Backoff.
			b := ac.resetBackoff
			ac.mu.Unlock()
			// sleep一下，然后重试
			timer := time.NewTimer(backoffFor)
			select {
			case <-timer.C:
				ac.mu.Lock()
				ac.backoffIdx++
				ac.mu.Unlock()
			case <-b:
				timer.Stop()
			case <-ac.ctx.Done():
				timer.Stop()
				return
			}
			continue
		}

		// 这里表示已经创建连接成功
		ac.mu.Lock()
		// 双重检查，是否已经Shutdown
		if ac.state == connectivity.Shutdown {
			newTr.Close()
			ac.mu.Unlock()
			return
		}
		// 当前连接的addr
		ac.curAddr = addr
		// transport
		ac.transport = newTr
		ac.backoffIdx = 0
 
        ...
        // 不需要
        if !healthcheckManagingState {
			ac.updateConnectivityState(connectivity.Ready)
		}
		ac.mu.Unlock()
        // 等待连接异常，触发重连
        <-reconnect.Done()
        ...
	}
}
```

接下来，看一下连接的状态变更的逻辑：

```go
func (ac *addrConn) updateConnectivityState(s connectivity.State) {
	if ac.state == s {
		return
	}

	updateMsg := fmt.Sprintf("Subchannel Connectivity change to %v", s)
	// 状态变更
	ac.state = s
 
	// 调用ClientConn的SubConn状态变更回调
	ac.cc.handleSubConnStateChange(ac.acbw, s)
}

func (cc *ClientConn) handleSubConnStateChange(sc balancer.SubConn, s connectivity.State) {
	cc.mu.Lock()
	if cc.conns == nil {
		cc.mu.Unlock()
		return
	}
 	
	cc.balancerWrapper.handleSubConnStateChange(sc, s)
	cc.mu.Unlock()
}

// ccBalancerWrapper实现了balancer.ClientConn接口
func (ccb *ccBalancerWrapper) handleSubConnStateChange(sc balancer.SubConn, s connectivity.State) {
	if sc == nil {
		return
	}
    // 将变更事件加入到队列中
	ccb.stateChangeQueue.put(&scStateUpdate{
		sc:    sc,
		state: s,
	})
}
```

上面将连接的状态变更事件加入到了一个队列中，那么必然有地方从队列中取出事件，通知到balancer：

```go
func (ccb *ccBalancerWrapper) watcher() {
	for {
		select {
        // 获取连接状态变更事件
		case t := <-ccb.stateChangeQueue.get():
            ...
			// V2Balancer是新版的Balancer接口
			if ub, ok := ccb.balancer.(balancer.V2Balancer); ok {
                // 通知连接状态更新
				ub.UpdateSubConnState(t.sc, balancer.SubConnState{ConnectivityState: t.state})
			} else {
                // 通知连接状态更新
				ccb.balancer.HandleSubConnStateChange(t.sc, t.state)
			}
		case s := <-ccb.ccUpdateCh:
			...
		case <-ccb.done:
		}
		... 
	}
}
```

```go
func (b *baseBalancer) UpdateSubConnState(sc balancer.SubConn, state balancer.SubConnState) {
   // 新的状态
   s := state.ConnectivityState
   // 旧的状态
   oldS, ok := b.scStates[sc]
   if !ok {
      return
   }
   // 设置新的状态
   b.scStates[sc] = s
   switch s {
   case connectivity.Idle:
      sc.Connect()        // Idle触发连接 
   case connectivity.Shutdown: 
      delete(b.scStates, sc) // 连接已经删除
   }

   // balancer原先的状态
   oldAggrState := b.state
   // 更新balancer的状态：
   //   - 如果存在Ready的subConn，则状态为ready
   //   - 否则如果存在connecting，则为connecting
   //   - 否则为TransientFailure
   b.state = b.csEvltr.RecordTransition(oldS, s)

   // 当下面情况发生时，需要重新创建Picker：
   //    - 连接由其他状态转变为Ready状态
   //    - 连接由Ready状态转变为其他状态
   //    - balancer转变为TransientFailure状态
   //    - balancer由TransientFailure转变为Non-TransientFailure状态
   if (s == connectivity.Ready) != (oldS == connectivity.Ready) ||
      (b.state == connectivity.TransientFailure) != (oldAggrState == connectivity.TransientFailure) {
      // 重新生成Picker
      b.regeneratePicker()
   }

   // 回调更新状态和picker
   b.cc.UpdateBalancerState(b.state, b.picker)
}
```



##### 客户端连接创建：DialContext

```go
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
	cc := &ClientConn{
		target:            target,                       // target
		csMgr:             &connectivityStateManager{},  // 连接状态管理器
		conns:             make(map[*addrConn]struct{}), // 连接
		dopts:             defaultDialOptions(),         // 默认的options
		blockingpicker:    newPickerWrapper(),           // balancer.Picker的wrapper
		czData:            new(channelzData),
		firstResolveEvent: grpcsync.NewEvent(),
	}

	cc.retryThrottler.Store((*retryThrottler)(nil))
	cc.ctx, cc.cancel = context.WithCancel(context.Background())

	// 应用options
	for _, opt := range opts {
		opt.apply(&cc.dopts)
	}

	// 如果存在多个Interceptors，则组装成一个调用链
	chainUnaryClientInterceptors(cc)
	chainStreamClientInterceptors(cc)

	defer func() {
		if err != nil {
			cc.Close()
		}
	}()

	...

	// tls连接加密证书相关检查
	if !cc.dopts.insecure {
		if cc.dopts.copts.TransportCredentials == nil && cc.dopts.copts.CredsBundle == nil {
			return nil, errNoTransportSecurity
		}
		if cc.dopts.copts.TransportCredentials != nil && cc.dopts.copts.CredsBundle != nil {
			return nil, errTransportCredsAndBundle
		}
	} else {
		if cc.dopts.copts.TransportCredentials != nil || cc.dopts.copts.CredsBundle != nil {
			return nil, errCredentialsConflict
		}
		for _, cd := range cc.dopts.copts.PerRPCCredentials {
			if cd.RequireTransportSecurity() {
				return nil, errTransportCredentialsMissing
			}
		}
	}

	// 如果提供了服务配置
	if cc.dopts.defaultServiceConfigRawJSON != nil {
		sc, err := parseServiceConfig(*cc.dopts.defaultServiceConfigRawJSON)
		if err != nil {
			return nil, fmt.Errorf("%s: %v", invalidDefaultServiceConfigErrPrefix, err)
		}
		// 设置默认服务配置
		cc.dopts.defaultServiceConfig = sc
	}

    // keepAlive配置
	cc.mkp = cc.dopts.copts.KeepaliveParams

	// 如果没有提供dial，则默认使用ProxyDialer，会根据系统环境变量的代理配置进行网络连接
	if cc.dopts.copts.Dialer == nil {
		cc.dopts.copts.Dialer = newProxyDialer(
			func(ctx context.Context, addr string) (net.Conn, error) {
				network, addr := parseDialTarget(addr)
				return (&net.Dialer{}).DialContext(ctx, network, addr)
			},
		)
	}

	// 用户代理添加grpcUA
	if cc.dopts.copts.UserAgent != "" {
		cc.dopts.copts.UserAgent += " " + grpcUA
	} else {
		cc.dopts.copts.UserAgent = grpcUA
	}

	// 如果options设置了timeout
	if cc.dopts.timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, cc.dopts.timeout)
		defer cancel()
	}
	defer func() {
		select {
		case <-ctx.Done():
			conn, err = nil, ctx.Err()
		default:
		}
	}()

	scSet := false
	// 如果提供了scChan，支持对serviceConfig进行热更
	if cc.dopts.scChan != nil {
		// Try to get an initial service config.
		select {
		// 尝试获取初始的serviceConfig
		case sc, ok := <-cc.dopts.scChan:
			if ok {
				cc.sc = &sc
				scSet = true // 成功获取初始的serviceConfig
			}
		default:
		}
	}
    
	// 提供retry时的退避算法
	if cc.dopts.bs == nil {
		cc.dopts.bs = backoff.Exponential{
			MaxDelay: DefaultBackoffConfig.MaxDelay,
		}
	}

	// resolverBuilder，用于解析target为目标服务列表
	// 如果没有指定resolverBuilder
	if cc.dopts.resolverBuilder == nil {
		// Only try to parse target when resolver builder is not already set.
		// 解析target，根据target的scheme获取对应的resolver
		cc.parsedTarget = parseTarget(cc.target)
		grpclog.Infof("parsed scheme: %q", cc.parsedTarget.Scheme)
		cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
		// 如果没有的话，使用默认的resolver
		if cc.dopts.resolverBuilder == nil {
			// If resolver builder is still nil, the parsed target's scheme is
			// not registered. Fallback to default resolver and set Endpoint to
			// the original target.
			grpclog.Infof("scheme %q not registered, fallback to default scheme", cc.parsedTarget.Scheme)
			cc.parsedTarget = resolver.Target{
				Scheme:   resolver.GetDefaultScheme(),
				Endpoint: target,
			}
			cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
		}
	} else {
		cc.parsedTarget = resolver.Target{Endpoint: target}
	}

	// 连接证书
	creds := cc.dopts.copts.TransportCredentials
	if creds != nil && creds.Info().ServerName != "" {
		cc.authority = creds.Info().ServerName
	} else if cc.dopts.insecure && cc.dopts.authority != "" {
		cc.authority = cc.dopts.authority
	} else {
		// Use endpoint from "scheme://authority/endpoint" as the default
		// authority for ClientConn.
		cc.authority = cc.parsedTarget.Endpoint
	}

	// 如果提供了scChan但是还没有获取到初始的serviceConfig，则阻塞等待serviceConfig
	if cc.dopts.scChan != nil && !scSet {
		// Blocking wait for the initial service config.
		select {
		case sc, ok := <-cc.dopts.scChan:
			if ok {
				cc.sc = &sc
			}
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
    
	// 启动子协程，监听scChan，进行serviceConfig的热更
	if cc.dopts.scChan != nil {
		go cc.scWatcher()
	}

	var credsClone credentials.TransportCredentials
	if creds := cc.dopts.copts.TransportCredentials; creds != nil {
		credsClone = creds.Clone()
	}
    
	// balancerBuild的options
	cc.balancerBuildOpts = balancer.BuildOptions{
		DialCreds:        credsClone,
		CredsBundle:      cc.dopts.copts.CredsBundle,
		Dialer:           cc.dopts.copts.Dialer,
		ChannelzParentID: cc.channelzID,
		Target:           cc.parsedTarget,
	}

	// Build the resolver.
	// 创建resovler，并包装成resolverWrapper
	rWrapper, err := newCCResolverWrapper(cc)
	if err != nil {
		return nil, fmt.Errorf("failed to build resolver: %v", err)
	}

	cc.mu.Lock()
	cc.resolverWrapper = rWrapper
	cc.mu.Unlock()
	// A blocking dial blocks until the clientConn is ready.
	// 默认Dial不会等待网络连接完成，如果指定了blcok，则会阻塞等待网络连接完成才返回
	if cc.dopts.block {
		for {
			s := cc.GetState()
			// 如果已经Ready
			if s == connectivity.Ready {
				break
			} else if cc.dopts.copts.FailOnNonTempDialError && s == connectivity.TransientFailure {
				if err = cc.blockingpicker.connectionError(); err != nil {
					terr, ok := err.(interface {
						Temporary() bool
					})
					if ok && !terr.Temporary() {
						return nil, err
					}
				}
			}
			// 等待状态变更
			if !cc.WaitForStateChange(ctx, s) {
				// ctx got timeout or canceled.
				return nil, ctx.Err()
			}
		}
	}

	return cc, nil
}
```



##### 总结

到此，对grpc客户端的连接创建流程应该有了一个大体的了解，并且我们能够很容易的根据`Resolver`接口提供自己的服务发现逻辑。