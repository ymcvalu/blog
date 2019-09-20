---
title: grpc server解析
date: 2019-07-22 11:26:28
tags:
	- grpc - go
---

`grpc`是由谷歌开源的一个高性能、通用的开源`rpc`框架，具体的使用可以参考[该文章](<http://mcll.top/2018/12/21/grpc%E4%B8%8A%E6%89%8B%E4%BD%BF%E7%94%A8/>)。本文主要看一下`go`版本的`grpc`的服务端实现。

### grpc server

我们先看一下启动一个grpc server时的代码：

```go
func main() {
	lis, err := net.Listen("tcp", ":6060")
	if err != nil {
		log.Fatal(err)
	}
	s := grpc.NewServer()
	// 注册服务，EchoServer实现了.proto中声明的服务接口
	proto.RegisterEchoSvcServer(s, &EchoServer{})
	// 启动服务
	if err := s.Serve(lis); err != nil {
		log.Fatal(err)
	}
}
```

可以看到，第6行创建了一个grpc server，第8行将具体的服务注册到server中，然后第10行开始启动服务。

其中，`RegisterEchoSvcServer`这个函数由插件`protoc-gen-go`通过`.proto`文件自动生成的，我们先来看一下其实现：

```go
// 服务描述，由插件通过`.proto`文件自动生成
var _EchoSvc_serviceDesc = grpc.ServiceDesc{
	ServiceName: "proto.EchoSvc", // 服务名
	HandlerType: (*EchoSvcServer)(nil), // 这里声明了该服务要实现的接口
    // 服务具有的方法列表
	Methods: []grpc.MethodDesc{
		{
			MethodName: "Echo", // rpc方法名
			Handler:    _EchoSvc_Echo_Handler, // rpc请求的handler
		},
	},
    // stream方法列表，该服务没有
	Streams:  []grpc.StreamDesc{},
	Metadata: "echo.proto",
}

func RegisterEchoSvcServer(s *grpc.Server, srv EchoSvcServer) {
    // 传入服务描述和具体的服务实现对象
	s.RegisterService(&_EchoSvc_serviceDesc, srv)
}
```

上面的_EchoSvc_serviceDesc是有插件根据我们的服务声明自动生成的，grpc中的rpc方法主要有两种类型。第一种就是常见的普通的rpc方法，第二种是[stream rpc方法](<https://grpc.io/docs/guides/concepts/>)

可以看到，实际上调用的是grpc server的服务注册方法：

```go
func (s *Server) RegisterService(sd *ServiceDesc, ss interface{}) {
	// ss需要实现的接口类型
    ht := reflect.TypeOf(sd.HandlerType).Elem()
    // ss实际的类型
	st := reflect.TypeOf(ss)
    // ss需要实现sd.HandlerType中指定的接口
	if !st.Implements(ht) {
		grpclog.Fatalf("grpc: Server.RegisterService found the handler of type %v that does not satisfy %v", st, ht)
	}
    // 注册接口到server
	s.register(sd, ss)
}

func (s *Server) register(sd *ServiceDesc, ss interface{}) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.printf("RegisterService(%q)", sd.ServiceName)
    // 如果server已经开始运行，不允许注册
	if s.serve {
		grpclog.Fatalf("grpc: Server.RegisterService after Server.Serve for %q", sd.ServiceName)
	}
    // 同一个服务名不允许重复注册
	if _, ok := s.m[sd.ServiceName]; ok {
		grpclog.Fatalf("grpc: Server.RegisterService found duplicate service registration for %q", sd.ServiceName)
	}
	srv := &service{
		server: ss, // 具体的服务实现对象
		md:     make(map[string]*MethodDesc), // 普通rpc方法描述
		sd:     make(map[string]*StreamDesc), // stream类型的rpc方法描述
		mdata:  sd.Metadata,
	}
    // 添加方法描述到srv的md
	for i := range sd.Methods {
		d := &sd.Methods[i]
		srv.md[d.MethodName] = d
	}
    // 添加方法描述到srv.sd
	for i := range sd.Streams {
		d := &sd.Streams[i]
		srv.sd[d.StreamName] = d
	}
    // 将servicer添加到server的services表中
	s.m[sd.ServiceName] = srv
}
```

可以看到，grpc的server中有一个service表，相当于http服务中的路由表。

接下来看一下，grpc server如果提供服务：

```go
func (s *Server) Serve(lis net.Listener) error {
	s.mu.Lock()
	s.printf("serving")
    // 开始运行
	s.serve = true
 	
    s.serveWG.Add(1)
	defer func() {
		s.serveWG.Done()
		select {
		// serve会阻塞直到退出服务
		case <-s.quit:
			<-s.done
		default:
		}
	}()

    // 把lis添加到lis表中，可以看到一个server可以同时监听多个端口提供服务
	ls := &listenSocket{Listener: lis}
	s.lis[ls] = true

	s.mu.Unlock()

	defer func() {
        // 服务退出时，从lis表中移除
		s.mu.Lock()
		if s.lis != nil && s.lis[ls] {
			ls.Close()
			delete(s.lis, ls)
		}
		s.mu.Unlock()
	}()

	for {
        // 接受客户端请求
		rawConn, err := lis.Accept()
		if err != nil {
			// 错误处理...
		}
		
		s.serveWG.Add(1)
		go func() {
            // 处理客户端连接
			s.handleRawConn(rawConn)
			s.serveWG.Done()
		}()
	}
}

func (s *Server) handleRawConn(rawConn net.Conn) {
    // 设置read和write的Deadline
	rawConn.SetDeadline(time.Now().Add(s.opts.connectionTimeout))
	// 可能开启了tls/ssl，需要证书认证，完成tls/ssl握手
    conn, authInfo, err := s.useTransportAuthenticator(rawConn)
	if err != nil {
		// ...
		return
	}

	s.mu.Lock()
	if s.conns == nil {
		s.mu.Unlock()
		conn.Close()
		return
	}
	s.mu.Unlock()

	// grpc是基于http2协议进行通信的，完成http2协议的握手
	st := s.newHTTP2Transport(conn, authInfo)
	if st == nil {
		return
	}
	// 前面设置Deadline是为了尽快完成握手操作
    // 因为客户端连接之后，并不是一直在发送请求，设置Deadline没有意义，因此这里取消deadline的设置
	rawConn.SetDeadline(time.Time{})
    // 保存客户端的http2连接
	if !s.addConn(st) {
		return
	}
    
	go func() {
        // 等待客户端rpc请求到来，并提供服务
        // 基于http2的多路复用，客户端可以使用一条连接同时发送多个请求
		s.serveStreams(st)
        // 移除客户端的http2连接
		s.removeConn(st)
	}()
}

func (s *Server) serveStreams(st transport.ServerTransport) {
	defer st.Close()
	var wg sync.WaitGroup

    // HandleStreams接收两个参数：handler和tracer
    // 该方法有两个参数：hanlder和tracer
    // 该方法循环读取客户端连接发送过来的帧：
    //    1. 如果是HEADER帧，说明有新的rpc请求到来，回调handler
    //    2. 如果是DATA帧，将数据分发到对应的stream
    //    3. ...
	st.HandleStreams(func(stream *transport.Stream) {
		wg.Add(1)
        // 回调中开启子协程，处理rpc请求
		go func() {
			defer wg.Done()
            // grpc基于http2，同一条连接会分成多个stream，每个rpc请求使用一个stream
            // 这样多个客户端请求可以复用同一条连接
            // 当有新的rpc请求到来，会进入该回调，然后调用server的handleStream处理rpc请求
			s.handleStream(st, stream, s.traceInfo(st, stream))
		}()
	}, func(ctx context.Context, method string) context.Context {
		if !EnableTracing {
			return ctx
		}
		tr := trace.New("grpc.Recv."+methodFamily(method), method)
		return trace.NewContext(ctx, tr)
	})
	wg.Wait()
}
```

接下来我们看一下server的handleStream方法，该方法处理rpc请求：

```go
func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo) {
    // rpc所请求的方法：服务名/方法名
	sm := stream.Method()
    // 去掉开头的`/`
	if sm != "" && sm[0] == '/' {
		sm = sm[1:]
	}
	pos := strings.LastIndex(sm, "/")
	if pos == -1 {
        // 应该满足：服务名/方法名
        // ...
		return
	}
    // 请求的服务
	service := sm[:pos]
    // 请求的方法
	method := sm[pos+1:]
    // 在server的服务表中查找对应的服务实现
    // server运行时不允许注册新的service，因此这里并发读，不需要加锁
	srv, ok := s.m[service]
    // 如果请求的服务不存在
	if !ok {
        // 如果server的配置中，指定了处理未知服务的方法，则交由其处理
		if unknownDesc := s.opts.unknownStreamDesc; unknownDesc != nil {
			s.processStreamingRPC(t, stream, nil, unknownDesc, trInfo)
			return
		}
        // ...
		return
	}
	// 先在普通的rpc方法表中查找
	if md, ok := srv.md[method]; ok {
        // 处理普通的rpc方法
		s.processUnaryRPC(t, stream, srv, md, trInfo)
		return
	}
    // 尝试在strem rpc找
	if sd, ok := srv.sd[method]; ok {
        // 处理stream rpc
		s.processStreamingRPC(t, stream, srv, sd, trInfo)
		return
	}
    
    // 请求未知方法
    
    // ...
	if unknownDesc := s.opts.unknownStreamDesc; unknownDesc != nil {
		s.processStreamingRPC(t, stream, nil, unknownDesc, trInfo)
		return
	}
 	
    // ...
}
```

限于篇幅，我们这里主要看一下`processUnaryRPC`方法，`processStreamingRPC`方法大同小异：

```go
func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, md *MethodDesc, trInfo *traceInfo) (err error) {
    
    // ...
    
	var comp, decomp encoding.Compressor
	var cp Compressor
	var dc Decompressor
	// ...	
    // 设置压缩选项
	if s.opts.cp != nil {
		cp = s.opts.cp
		stream.SetSendCompress(cp.Type())
	} else if rc := stream.RecvCompress(); rc != "" && rc != encoding.Identity {
		// Legacy compressor not specified; attempt to respond with same encoding.
		comp = encoding.GetCompressor(rc)
		if comp != nil {
			stream.SetSendCompress(rc)
		}
	}

	var payInfo *payloadInfo
	if sh != nil || binlog != nil {
		payInfo = &payloadInfo{}
	}
	// 接收并解压缩数据
	d, err := recvAndDecompress(&parser{r: stream}, stream, dc, s.opts.maxReceiveMessageSize, payInfo, decomp)
	if err != nil {
		// ...
		return err
	}

	// df方法用于从接收的数据包d中反序列化为v
	df := func(v interface{}) error {
        // 反序列化请求参数
		if err := s.getCodec(stream.ContentSubtype()).Unmarshal(d, v); err != nil {
			return status.Errorf(codes.Internal, "grpc: error unmarshalling request: %v", err)
		}
		// ...
		return nil
	}
    
	// 创建context，该方法和header的获取以及写入有关，下面分析
	ctx := NewContextWithServerTransportStream(stream.Context(), stream)
	// 执行handler，这个handler是通过.proto文件生成的，该方法内会去调用server的对应的方法，该方法返回对应的resp
    // 这里第三个参数是反序列化方法，第四个参数是创建server时指定的interceptor选项
	reply, appErr := md.Handler(srv.server, ctx, df, s.opts.unaryInt)
	// 如果返回的错误，这里的err可能是由我们的rpc方法返回的
	if appErr != nil {
		appStatus, ok := status.FromError(appErr)
		if !ok {
			// 如过没有实现 interface{GRPCStatus()*Status} 接口
			appErr = status.Error(codes.Unknown, appErr.Error())
			appStatus, _ = status.FromError(appErr)
		}
		// ...
		// 写入错误信息到stream中
		if e := t.WriteStatus(stream, appStatus); e != nil {
			grpclog.Warningf("grpc: Server.processUnaryRPC failed to write status: %v", e)
		}
		// ...
		return appErr
	}
 
	opts := &transport.Options{Last: true}
	// 序列化reply给客户端
	if err := s.sendResponse(t, stream, reply, cp, opts, comp); err != nil {
		// ...
		return err
	}

	err = t.WriteStatus(stream, status.New(codes.OK, ""))
    
	return err
}

```

上面的代码略有删减，主要是删掉一些和统计、trace以及日志相关的代码。主要的逻辑就是从stream读取请求参参数，反序列化后调用methodDesc中的handler方法，然后把返回的内容序列化后写入stream返回给客户端。

我们知道，grpc是基于http2协议的，因此也是存在`header`的，grpc和http一样，可以设置和获取请求的header。在服务端，主要有获取客户端传递过来的header以及传递header给客户端两个操作。

我们先看上面出现的`NewContextWithServerTransportStream`方法：

```go
// 该接口用于服务端设置传递给客户端的header
type ServerTransportStream interface {
	Method() string
	SetHeader(md metadata.MD) error
	SendHeader(md metadata.MD) error
	SetTrailer(md metadata.MD) error
}

func NewContextWithServerTransportStream(ctx context.Context, stream ServerTransportStream) context.Context {
    // 基于ctx创建新的context，并把stream保存到新的context中
    // 当调用grpc.SetHeader时，会执行stream.SetHeader方法
	return context.WithValue(ctx, streamKey{}, stream)
}
```

我们可以看到在`processUnaryRPC`方法中，对该方法的调用如下：

```go
func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, md *MethodDesc, trInfo *traceInfo) (err error) {
 	// ...
	ctx := NewContextWithServerTransportStream(stream.Context(), stream)
	reply, appErr := md.Handler(srv.server, ctx, df, s.opts.unaryInt)
 	// ...
}
```

可以看到，传入的是当前请求的`stream`的`context`，接下来看一下`stream`的`context`创建：

```go
func (t *http2Server) operateHeaders(frame *http2.MetaHeadersFrame, handle func(*Stream), traceCtx func(context.Context, string) context.Context) (fatal bool) {
	streamID := frame.Header().StreamID
	state := decodeState{serverSide: true}
    // 解析header帧，包括获取header中的各个字段
	if err := state.decodeHeader(frame); err != nil {
	 	// ...
		return false
	}

	buf := newRecvBuffer()
	s := &Stream{
		id:             streamID,
		st:             t,
		buf:            buf,
		fc:             &inFlow{limit: uint32(t.initialWindowSize)},
		recvCompress:   state.encoding,
		method:         state.method,
		contentSubtype: state.contentSubtype,
	}
 
    // ...
    
 	// 除了grpc预定义的几个header之外，其他header都保存到mdata中
	if len(state.mdata) > 0 {
        // 这里会将state.mdata保存到新的context中
		s.ctx = metadata.NewIncomingContext(s.ctx, state.mdata)
	}
    // ...
	handle(s)
	return false
}

func NewIncomingContext(ctx context.Context, md MD) context.Context {
	return context.WithValue(ctx, mdIncomingKey{}, md)
}
```

当收到一个`Header`帧，就表明有新的rpc请求到来，这时候就会解析header帧并创建stream，在创建stream的时候，会把用户自定义的header字段保存到stream.context中

在我们实际编码时，可以通过`metadata`包来读取客户端传递过来的`header`：

```go
func (EchoServer) Echo(ctx context.Context, req *proto.EchoReq) (resp *proto.EchoResp, err error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if ok {
		log.Printf("%s: %v", md.Get("key"))
	}
    
	return &proto.EchoResp{
		Msg: VERSION,
	}, err
}
```

而设置header返回给客户端可以如下：

```go
func (EchoServer) Echo(ctx context.Context, req *proto.EchoReq) (resp *proto.EchoResp, err error) {
	// 最终会设置stream的header
	grpc.SetHeader(ctx, metadata.Pairs("key1", "val1"))

	return &proto.EchoResp{
		Msg: VERSION,
	}, err
}
```

接下来看一下写回返回内容给客户端的逻辑：

```go
func (s *Server) sendResponse(t transport.ServerTransport, stream *transport.Stream, msg interface{}, cp Compressor, opts *transport.Options, comp encoding.Compressor) error {
    // 反序列化返回内容
	data, err := encode(s.getCodec(stream.ContentSubtype()), msg)
	if err != nil {
		grpclog.Errorln("grpc: server failed to encode response: ", err)
		return err
	}
    // 压缩
	compData, err := compress(data, cp, comp)
	if err != nil {
		grpclog.Errorln("grpc: server failed to compress response: ", err)
		return err
	}
    // 创建消息头部
	hdr, payload := msgHeader(data, compData)
	// TODO(dfawley): should we be checking len(data) instead?
	if len(payload) > s.opts.maxSendMessageSize {
		return status.Errorf(codes.ResourceExhausted, "grpc: trying to send message larger than max (%d vs. %d)", len(payload), s.opts.maxSendMessageSize)
	}
    // 写回内容
	err = t.Write(stream, hdr, payload, opts)
	if err == nil && s.opts.statsHandler != nil {
		s.opts.statsHandler.HandleRPC(stream.Context(), outPayload(false, msg, data, payload, time.Now()))
	}
	return err
}

func (t *http2Server) Write(s *Stream, hdr []byte, data []byte, opts *Options) error {
    // 如果header还没有发送，先发送header
	if !s.isHeaderSent() { // Headers haven't been written yet.
		if err := t.WriteHeader(s, nil); err != nil {
			return status.Errorf(codes.Internal, "transport: %v", err)
		}
	} else {
		// ...
	}

	emptyLen := http2MaxFrameLen - len(hdr)
	if emptyLen > len(data) {
		emptyLen = len(data)
	}
	hdr = append(hdr, data[:emptyLen]...)
	data = data[emptyLen:]
    // 数据帧
	df := &dataFrame{
		streamID: s.id,
		h:        hdr,
		d:        data,
		onEachWrite: func() {
			atomic.StoreUint32(&t.resetPingStrikes, 1)
		},
	}
	if err := s.wq.get(int32(len(hdr) + len(data))); err != nil {
		select {
		case <-t.ctx.Done():
			return ErrConnClosing
		default:
		}
		return ContextErr(s.ctx.Err())
	}
    // 把数据帧加入到发送队列
	return t.controlBuf.put(df)
}
```

最后，看一下methodDesc中的handler，这个是由插件自动生成的包装方法：

```go
func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, md *MethodDesc, trInfo *traceInfo) (err error) {
    // ...
    reply, appErr := md.Handler(srv.server, ctx, df, s.opts.unaryInt)
    // ...
}
```

```go
func _EchoSvc_Echo_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	// 
    in := new(EchoReq)
    // dec是传入的反序列化方法
	if err := dec(in); err != nil {
		return nil, err
	}
    
    // 如果没有指定interceptor
	if interceptor == nil {
        // 直接调用service对应的方法
		return srv.(EchoSvcServer).Echo(ctx, in)
	}
    
    // 服务信息
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/proto.EchoSvc/Echo",
	}
    
    // 回调handler
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(EchoSvcServer).Echo(ctx, req.(*EchoReq))
	}
    // 先执行interceptor，然后在执行handler
	return interceptor(ctx, in, info, handler)
}
```

可以看到，用户创建`server`时，如果设置了`interceptor`选项，那么在执行具体的服务方法前，会先执行用户设置的`interceptor`，声明如下：

```go
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

在`interceptor`中，可以做一些通用处理，比如日志记录，异常处理或者请求拦截等





