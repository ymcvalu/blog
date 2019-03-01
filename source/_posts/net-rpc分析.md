---
title: net/rpc分析
date: 2019-02-28 16:50:08
tags:
	- go
---

### rpc

`golang`本身提供了`net/rpc`标准库，用于提供`rpc`服务。

`rpc`通过将网络传输和数据序列化/反序列化屏蔽在接口背后，提供一种简洁的调用接口，已达到调用远程服务方法在执行本地方法一样。

### server 

##### service

`service`代表每个注册的服务

```go
type methodType struct {
	sync.Mutex 				  // protects counters
	method     reflect.Method // 方法信息
	ArgType    reflect.Type   // 第一个参数类型
	ReplyType  reflect.Type   // 第二个参数类型，该参数用来返回结果
	numCalls   uint           // 统计调用次数
}

type service struct {
	name   string                 // 服务名
	rcvr   reflect.Value          // 服务对象的值
	typ    reflect.Type           // 服务对象的类型
	method map[string]*methodType // 该服务对外提供的方法
}
```

##### server

`server`代表一个`rpc server`

```go
type Server struct {
	serviceMap sync.Map     // 注册的服务：map[string]*service
	reqLock    sync.Mutex   // protects freeReq
	freeReq    *Request     // 缓存Request列表，避免每次请求都要重新创建一个
	respLock   sync.Mutex   // protects freeResp
	freeResp   *Response
}
```

##### Request & Response

```go 
// Request is a header written before every RPC call. It is used internally
// but documented here as an aid to debugging, such as when analyzing
// network traffic.
type Request struct {
	ServiceMethod string   // format: "Service.Method"
	Seq           uint64   // 请求的Seq，客户端会对请求进行编号，用于区分不同的请求
	next          *Request // for free list in Server
}

// Response is a header written before every RPC return. It is used internally
// but documented here as an aid to debugging, such as when analyzing
// network traffic.
type Response struct {
	ServiceMethod string    // echoes that of the Request
	Seq           uint64    // echoes that of the request
	Error         string    // error, if any.
	next          *Response // for free list in Server
}
```

##### Register

```go
func (server *Server) Register(rcvr interface{}) error {
	// 使用反射名作为服务名称
    return server.register(rcvr, "", false)
}

func (server *Server) RegisterName(name string, rcvr interface{}) error {
    // 自定义服务名称
   return server.register(rcvr, name, true)
}

func (server *Server) register(rcvr interface{}, name string, useName bool) error {
   s := new(service) 
   s.typ = reflect.TypeOf(rcvr)   // 设置类型
   s.rcvr = reflect.ValueOf(rcvr) // 设置值
   // 默认取类型名
   sname := reflect.Indirect(s.rcvr).Type().Name()
   if useName { // 如果需要使用自定义名称
      sname = name
   }
   if sname == "" {
      s := "rpc.Register: no service name for type " + s.typ.String()
      log.Print(s)
      return errors.New(s)
   }
    // 如果该service不是导出类型，保错
   if !isExported(sname) && !useName {
      s := "rpc.Register: type " + sname + " is not exported"
      log.Print(s)
      return errors.New(s)
   }
   s.name = sname

   // 存找该services用于提供对外服务的方法
   s.method = suitableMethods(s.typ, true)
	// 方法数必须大于0
   if len(s.method) == 0 {
      str := ""

      // To help the user, see if a pointer receiver would work.
      method := suitableMethods(reflect.PtrTo(s.typ), false)
      if len(method) != 0 {
         str = "rpc.Register: type " + sname + " has no exported methods of suitable type (hint: pass a pointer to value of that type)"
      } else {
         str = "rpc.Register: type " + sname + " has no exported methods of suitable type"
      }
      log.Print(str)
      return errors.New(str)
   }
	// 不允许同一个服务名称重复注册
   if _, dup := server.serviceMap.LoadOrStore(sname, s); dup {
      return errors.New("rpc: service already defined: " + sname)
   }
   return nil
}
```

`suitableMethods`用来查找`service`中需要暴露的方法，实现就是遍历该`service`的所有方法，并返回其中符合条件的方法

```go 
func suitableMethods(typ reflect.Type, reportErr bool) map[string]*methodType {
	methods := make(map[string]*methodType)
    // 遍历方法
	for m := 0; m < typ.NumMethod(); m++ {
		method := typ.Method(m)
		mtype := method.Type
		mname := method.Name
		// Method must be exported.
        // 如果method是导出的（方法名首字母大写），PkgPath为空
		if method.PkgPath != "" { 
			continue
		}
		// Method needs three ins: receiver, *args, *reply.
        // 参数个数必须为3，其中第一个参数为service对象
		if mtype.NumIn() != 3 {
			if reportErr {
				log.Printf("rpc.Register: method %q has %d input parameters; needs exactly three\n", mname, mtype.NumIn())
			}
			continue
		}
		// First arg need not be a pointer.
		// 第二个参数必须是内置类型或者自定义的导出类型，不需要是指针类型
        argType := mtype.In(1)
		if !isExportedOrBuiltinType(argType) {
			if reportErr {
				log.Printf("rpc.Register: argument type of method %q is not exported: %q\n", mname, argType)
			}
			continue
		}
		// Second arg must be a pointer.
		replyType := mtype.In(2)
        // 第三个参数必须是指针类型，该参数用来向客户端返回请求结果
		if replyType.Kind() != reflect.Ptr {
			if reportErr {
				log.Printf("rpc.Register: reply type of method %q is not a pointer: %q\n", mname, replyType)
			}
			continue
		}
		// Reply type must be exported.
        // 该参数也必须是内置类型或者导出类型
		if !isExportedOrBuiltinType(replyType) {
			if reportErr {
				log.Printf("rpc.Register: reply type of method %q is not exported: %q\n", mname, replyType)
			}
			continue
		}
		// Method needs one out.
        // 方法必须有且只有一个error类型的返回值
		if mtype.NumOut() != 1 {
			if reportErr {
				log.Printf("rpc.Register: method %q has %d output parameters; needs exactly one\n", mname, mtype.NumOut())
			}
			continue
		}
		// The return type of the method must be error.
		if returnType := mtype.Out(0); returnType != typeOfError {
			if reportErr {
				log.Printf("rpc.Register: return type of method %q is %q, must be error\n", mname, returnType)
			}
			continue
		}
        // 符合条件，添加
		methods[mname] = &methodType{method: method, ArgType: argType, ReplyType: replyType}
	}
	return methods
}
```

##### 启动服务

```go 
func (server *Server) Accept(lis net.Listener) {
   for {
      conn, err := lis.Accept()
      if err != nil {
         log.Print("rpc.Serve: accept:", err.Error())
         return
      }
      // 每个客户启用一个goroutine进行处理
      go server.ServeConn(conn)
   }
}
```

##### 处理请求

```go
func (server *Server) ServeConn(conn io.ReadWriteCloser) {
	buf := bufio.NewWriter(conn)
    // 默认使用gob编解码
	srv := &gobServerCodec{
		rwc:    conn,
		dec:    gob.NewDecoder(conn),
		enc:    gob.NewEncoder(buf),
		encBuf: buf,
	}
	server.ServeCodec(srv)
}

func (server *Server) ServeCodec(codec ServerCodec) {
	sending := new(sync.Mutex) // 写response内容时需要加锁
	wg := new(sync.WaitGroup)
	for {
        // 从连接中读取请求
		service, mtype, req, argv, replyv, keepReading, err := server.readRequest(codec)
		if err != nil {
			if debugLog && err != io.EOF {
				log.Println("rpc:", err)
			}
			if !keepReading {
				break
			}
			// send a response if we actually managed to read a header.
			if req != nil {
				server.sendResponse(sending, req, invalidRequest, codec, err.Error())
				server.freeRequest(req)
			}
			continue
		}
		wg.Add(1)
        // 每个rpc都使用一个goroutine进行处理
		go service.call(server, sending, wg, mtype, req, argv, replyv, codec)
	}
	// 优雅关闭
	wg.Wait()
	codec.Close()
}
```

```go
func (s *service) call(server *Server, sending *sync.Mutex, wg *sync.WaitGroup, mtype *methodType, req *Request, argv, replyv reflect.Value, codec ServerCodec) {
	if wg != nil {
		defer wg.Done()
	}
	mtype.Lock()
	mtype.numCalls++ // 统计调用次数
	mtype.Unlock()
	function := mtype.method.Func
	// 调用具体的请求方法
	returnValues := function.Call([]reflect.Value{s.rcvr, argv, replyv})
	// The return value for the method is an error.
	errInter := returnValues[0].Interface()
	errmsg := ""
    // 如果请求方法返回error
	if errInter != nil {
		errmsg = errInter.(error).Error()
	}
    // 写入响应结果
	server.sendResponse(sending, req, replyv.Interface(), codec, errmsg)
	// 释放req
    server.freeRequest(req)
}
```

### client

##### client

```go
type Client struct {
	codec ClientCodec // codec

	reqMutex sync.Mutex // protects following
	request  Request

	mutex    sync.Mutex // protects following
	seq      uint64 // 下一次请求的seq
	pending  map[uint64]*Call // 正在执行的请求
	closing  bool // user has called Close
	shutdown bool // server has told us to stop
}
```

##### Call

```go
type Call struct {
   ServiceMethod string      // 调用的远程方法
   Args          interface{} // 方法第一个参数
   Reply         interface{} // 第二个参数，用于接收返回值
   Error         error       // 保存请求的错误信息
   Done          chan *Call  // 用于通知请求结束
}
```



##### NewClient

```go
func NewClient(conn io.ReadWriteCloser) *Client {
	encBuf := bufio.NewWriter(conn)
	client := &gobClientCodec{conn, gob.NewDecoder(conn), gob.NewEncoder(encBuf), encBuf}
	return NewClientWithCodec(client)
}

func NewClientWithCodec(codec ClientCodec) *Client {
	client := &Client{
		codec:   codec,
		pending: make(map[uint64]*Call),
	}
	go client.input() // input用来处理server的响应
	return client
}

```



##### Call

使用方法`Call`和方法`Go`调用远程方法，其中`Call`会同步等待请求结束，`Go`是异步执行

```go 
func (client *Client) Call(serviceMethod string, args interface{}, reply interface{}) error {
    // Call方法内部也是调用Go，然后等待调用完成后返回
	call := <-client.Go(serviceMethod, args, reply, make(chan *Call, 1)).Done
	return call.Error
}

// Go方法返回一个channel用来通知调用结束
func (client *Client) Go(serviceMethod string, args interface{}, reply interface{}, done chan *Call) *Call {
	call := new(Call)
	call.ServiceMethod = serviceMethod
	call.Args = args
	call.Reply = reply
    
	if done == nil {
		done = make(chan *Call, 10) // buffered.
	} else {
		// If caller passes done != nil, it must arrange that
		// done has enough buffer for the number of simultaneous
		// RPCs that will be using that channel. If the channel
		// is totally unbuffered, it's best not to run at all.
        if cap(done) == 0 {
			log.Panic("rpc: done channel is unbuffered")
		}
	}
	call.Done = done
	client.send(call)
	return call
}

// send执行实际的请求发送
func (client *Client) send(call *Call) {
	client.reqMutex.Lock()
	defer client.reqMutex.Unlock()

	// Register this call.
	client.mutex.Lock()
	if client.shutdown || client.closing {
		client.mutex.Unlock()
		call.Error = ErrShutdown
		call.done()
		return
	}
	seq := client.seq // 获取此次请求seq
	client.seq++ // 计算下一次请求seq
	client.pending[seq] = call // 加入pending列表中
	client.mutex.Unlock() 

	// Encode and send the request.
	client.request.Seq = seq
	client.request.ServiceMethod = call.ServiceMethod
    // 发送请求
	err := client.codec.WriteRequest(&client.request, call.Args)
	if err != nil {
		client.mutex.Lock()
		call = client.pending[seq]
		delete(client.pending, seq)
		client.mutex.Unlock()
		if call != nil {
			call.Error = err
			call.done()
		}
	}
}
```

分析上面的`send`方法，当请求发送出去之后就返回了，那么如何处理请求的响应呢？我们可以看到每次新的请求都会加入到`client.pending`中，那么对应的应该有一个幕后的协程来处理，当收到`server`的响应时，根据`seq`获取对应的`call`，然后通过`call.Done`通知请求完成，

对应的方法为`input`：

```go
func (client *Client) input() {
	var err error
	var response Response
	for err == nil {
		response = Response{}
        // 读取server的返回结果
		err = client.codec.ReadResponseHeader(&response)
		if err != nil {
			break
		}
        // 获取这次响应对应的请求的seq
		seq := response.Seq
		client.mutex.Lock()
        // 获取对应的请求
		call := client.pending[seq]
		delete(client.pending, seq)
		client.mutex.Unlock()
		// 处理响应结果
		switch {
		case call == nil:
			// We've got no pending call. That usually means that
			// WriteRequest partially failed, and call was already
			// removed; response is a server telling us about an
			// error reading request body. We should still attempt
			// to read error body, but there's no one to give it to.
			err = client.codec.ReadResponseBody(nil)
			if err != nil {
				err = errors.New("reading error body: " + err.Error())
			}
		case response.Error != "":
			// We've got an error response. Give this to the request;
			// any subsequent requests will get the ReadResponseBody
			// error if there is one.
			call.Error = ServerError(response.Error)
			err = client.codec.ReadResponseBody(nil)
			if err != nil {
				err = errors.New("reading error body: " + err.Error())
			}
			call.done()
		default:
			err = client.codec.ReadResponseBody(call.Reply)
			if err != nil {
				call.Error = errors.New("reading body " + err.Error())
			}
			call.done()
		}
	}
    // 发生错误退出之后，停止所有等待的请求
	// Terminate pending calls.
	client.reqMutex.Lock()
	client.mutex.Lock()
	client.shutdown = true
	closing := client.closing
	if err == io.EOF {
		if closing {
			err = ErrShutdown
		} else {
			err = io.ErrUnexpectedEOF
		}
	}
    // 停止所有等待的请求
	for _, call := range client.pending {
		call.Error = err
		call.done()
	}
	client.mutex.Unlock()
	client.reqMutex.Unlock()
	if debugLog && err != io.EOF && !closing {
		log.Println("rpc: client protocol error:", err)
	}
}

// 通知调用结束
func (call *Call) done() {
	select {
	case call.Done <- call:
		// ok
	default:
		// We don't want to block here. It is the caller's responsibility to make
		// sure the channel has enough buffer space. See comment in Go().
		if debugLog {
			log.Println("rpc: discarding Call reply due to insufficient Done chan capacity")
		}
	}
}
```

