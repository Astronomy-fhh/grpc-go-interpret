

# grpc-go-server-interpret

## 1. 快速开始

[获取源码及安装必须的内容和运行一个简单示例](https://grpc.io/docs/languages/go/quickstart/)

## 2. 服务端流程

### 启动一个服务的精简代码如下

```go
func main() {
  // 在本地网络中，开始收听这个地址
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
  // 见Explain:NewServer
	s := grpc.NewServer()
  // 见Explain:注册服务
	pb.RegisterGreeterServer(s, &server{})
  // 见Explain:开始服务
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```



## 3. Explain 

### NewServer

``` go
// 见Explain:ServerOption
func NewServer(opt ...ServerOption) *Server {
	opts := defaultServerOptions 
	for _, o := range opt {
		o.apply(&opts)
	}
  // 服务rpc请求的server,详细结构见Explain:Server
	s := &Server{
		lis:      make(map[net.Listener]bool),
		opts:     opts,
		conns:    make(map[transport.ServerTransport]bool),
		services: make(map[string]*serviceInfo),
		quit:     grpcsync.NewEvent(),
		done:     grpcsync.NewEvent(),
		czData:   new(channelzData),
	}
  // 见Explain:chainUnaryServerInterceptors
	chainUnaryServerInterceptors(s)
  // 除了方法结构的区别，与chainUnaryServerInterceptors实现原理一样，不再赘述
	chainStreamServerInterceptors(s)
  // 如文档所说，友好关闭连接时的发出信号
	s.cv = sync.NewCond(&s.mu)
  // Package trace implements tracing of requests and long-lived objects.
  // It exports HTTP interfaces on /debug/requests and /debug/events.
  // 见Learn:EnableTracing
	if EnableTracing {
		_, file, line, _ := runtime.Caller(1)
		s.events = trace.NewEventLog("grpc.Server", fmt.Sprintf("%s:%d", file, line))
	}

  // 见Explain:initServerWorkers
	if s.opts.numServerWorkers > 0 {
		s.initServerWorkers()
	}
  // Channelz是一个工具，可提供有关gRPC中不同级别的连接的全面运行时信息
  // 具体内容见Learn:channelz
	if channelz.IsOn() {
		s.channelzID = channelz.RegisterServer(&channelzServer{s}, "")
	}
	return s
}
```

### ServerOption

ServerOption及server的设置选项.

```go
var defaultServerOptions = serverOptions{
	maxReceiveMessageSize: defaultServerMaxReceiveMessageSize,
	maxSendMessageSize:    defaultServerMaxSendMessageSize,
	connectionTimeout:     120 * time.Second,
	writeBufferSize:       defaultWriteBufSize,
	readBufferSize:        defaultReadBufSize,
}
```

``` go
type ServerOption interface {
	apply(*serverOptions)
}

type funcServerOption struct {
	f func(*serverOptions)
}

func (fdo *funcServerOption) apply(do *serverOptions) {
	fdo.f(do)
}

func newFuncServerOption(f func(*serverOptions)) *funcServerOption {
	return &funcServerOption{
		f: f,
	}
}
```

```go
type serverOptions struct {
	creds                 credentials.TransportCredentials
	codec                 baseCodec
	cp                    Compressor
	dc                    Decompressor
	unaryInt              UnaryServerInterceptor
	streamInt             StreamServerInterceptor
	chainUnaryInts        []UnaryServerInterceptor
	chainStreamInts       []StreamServerInterceptor
	inTapHandle           tap.ServerInHandle
	statsHandler          stats.Handler
	maxConcurrentStreams  uint32
	maxReceiveMessageSize int
	maxSendMessageSize    int
	unknownStreamDesc     *StreamDesc
	keepaliveParams       keepalive.ServerParameters
	keepalivePolicy       keepalive.EnforcementPolicy
	initialWindowSize     int32
	initialConnWindowSize int32
	writeBufferSize       int
	readBufferSize        int
	connectionTimeout     time.Duration
	maxHeaderListSize     *uint32
	headerTableSize       *uint32
	numServerWorkers      uint32
}
```

```go
func WriteBufferSize(s int) ServerOption {
	return newFuncServerOption(func(o *serverOptions) {
		o.writeBufferSize = s
	})
}
```

对于NewServer方法传递的多个serverOption（即funcServerOption）,每一个都对应于serverOptions的某个设置项， 最终达到的效果是遍历这些serverOption,执行serverOption的apply方法，将单个的设置项集中到一个serverOptions结构上。

如上面的WriteBufferSize，还有例如ReadBufferSize，InitialWindowSize等方法都是现有的 ,都可获取独立的设置项（ServerOption）



### Server

```go
type Server struct {
	opts serverOptions

	mu       sync.Mutex // guards following
	lis      map[net.Listener]bool
	conns    map[transport.ServerTransport]bool
	serve    bool
	drain    bool
	cv       *sync.Cond              // signaled when connections close for GracefulStop
	services map[string]*serviceInfo // service name -> service info
	events   trace.EventLog

	quit               *grpcsync.Event
	done               *grpcsync.Event
	channelzRemoveOnce sync.Once
	serveWG            sync.WaitGroup // counts active Serve goroutines for GracefulStop

	channelzID int64 // channelz unique identification number
	czData     *channelzData

	serverWorkerChannels []chan *serverWorkerData
}
```

### chainUnaryServerInterceptors

```go
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

```go
func chainUnaryServerInterceptors(s *Server) {
	// Prepend opts.unaryInt to the chaining interceptors if it exists, since unaryInt will
	// be executed before any other chained interceptors.
	
	// s.opts.unaryInt为单个拦截器，s.opts.chainUnaryInts是多个拦截器的数组。
	// 拦截器为一个方法类型：UnaryServerInterceptor
	// unaryInt保证要比其他拦截器更早执行，需要将unaryInt所指拦截器放在数组头部
	interceptors := s.opts.chainUnaryInts
	if s.opts.unaryInt != nil {
		interceptors = append([]UnaryServerInterceptor{s.opts.unaryInt}, s.opts.chainUnaryInts...)
	}

  // 如果整合后的拦截器为空，则空
	var chainedInt UnaryServerInterceptor
	if len(interceptors) == 0 {
		chainedInt = nil
		// 如果整合后只有一个拦截器，则直接指向此拦截器
	} else if len(interceptors) == 1 {
		chainedInt = interceptors[0]
	} else {
	  //需要有多个拦截器，则递归整合成一个链式拦截器
    // 链式拦截器具体实现，见Learned:链式拦截器
		chainedInt = func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (interface{}, error) {
			return interceptors[0](ctx, req, info, getChainUnaryHandler(interceptors, 0, info, handler))
		}
	}

	s.opts.unaryInt = chainedInt
}
```

### initServerWorkers

```go
// 根据numServerWorkers初始化工作的协程数量
func (s *Server) initServerWorkers() {
	s.serverWorkerChannels = make([]chan *serverWorkerData, s.opts.numServerWorkers)
	for i := uint32(0); i < s.opts.numServerWorkers; i++ {
		s.serverWorkerChannels[i] = make(chan *serverWorkerData)
		go s.serverWorker(s.serverWorkerChannels[i])
	}
}


func (s *Server) serverWorker(ch chan *serverWorkerData) {
	// To make sure all server workers don't reset at the same time, choose a
	// random number of iterations before resetting.
  // const serverWorkerResetThreshold = 1 << 16
  // 对于每一个协程，处理指定数量的请求之后需要重置（结束），以免内存中存在大堆栈
	threshold := serverWorkerResetThreshold + grpcrand.Intn(serverWorkerResetThreshold)
	for completed := 0; completed < threshold; completed++ {
		data, ok := <-ch
		if !ok {
      // 没有待处理的任务，此协程return结束
			return
		}
    // 处理客户端的一次请求（stream）
		s.handleStream(data.st, data.stream, s.traceInfo(data.st, data.stream))
		data.wg.Done()
	}
  // 重新启动一个工作协程
	go s.serverWorker(ch)
}

type serverWorkerData struct {
	st     transport.ServerTransport
	wg     *sync.WaitGroup
	stream *transport.Stream
}
```

### 注册服务

为了清晰的理解注册服务所需的结构化内容，先看一下服务的标准代码生成。

```go
package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

指定一个proto文件，定义相关的服务service及message,通过protoc相关命令生成下面所列内容。

```go
// Code generated by protoc-gen-go-grpc. DO NOT EDIT.

package helloworld

import (
	context "context"
	grpc "google.golang.org/grpc"
	codes "google.golang.org/grpc/codes"
	status "google.golang.org/grpc/status"
)

// This is a compile-time assertion to ensure that this generated file
// is compatible with the grpc package it is being compiled against.
const _ = grpc.SupportPackageIsVersion7

// GreeterClient is the client API for Greeter service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://pkg.go.dev/google.golang.org/grpc/?tab=doc#ClientConn.NewStream.
type GreeterClient interface {
	// Sends a greeting
	SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error)
}

type greeterClient struct {
	cc grpc.ClientConnInterface
}

func NewGreeterClient(cc grpc.ClientConnInterface) GreeterClient {
	return &greeterClient{cc}
}

func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
	out := new(HelloReply)
	err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

// GreeterServer is the server API for Greeter service.
// All implementations must embed UnimplementedGreeterServer
// for forward compatibility
type GreeterServer interface {
	// Sends a greeting
	SayHello(context.Context, *HelloRequest) (*HelloReply, error)
	mustEmbedUnimplementedGreeterServer()
}

// UnimplementedGreeterServer must be embedded to have forward compatible implementations.
type UnimplementedGreeterServer struct {
}

func (UnimplementedGreeterServer) SayHello(context.Context, *HelloRequest) (*HelloReply, error) {
	return nil, status.Errorf(codes.Unimplemented, "method SayHello not implemented")
}
func (UnimplementedGreeterServer) mustEmbedUnimplementedGreeterServer() {}

// UnsafeGreeterServer may be embedded to opt out of forward compatibility for this service.
// Use of this interface is not recommended, as added methods to GreeterServer will
// result in compilation errors.
type UnsafeGreeterServer interface {
	mustEmbedUnimplementedGreeterServer()
}

func RegisterGreeterServer(s grpc.ServiceRegistrar, srv GreeterServer) {
	s.RegisterService(&Greeter_ServiceDesc, srv)
}

func _Greeter_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(HelloRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(GreeterServer).SayHello(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/helloworld.Greeter/SayHello",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(GreeterServer).SayHello(ctx, req.(*HelloRequest))
	}
	return interceptor(ctx, in, info, handler)
}

// Greeter_ServiceDesc is the grpc.ServiceDesc for Greeter service.
// It's only intended for direct use with grpc.RegisterService,
// and not to be introspected or modified (even as a copy)
// Greeter_ServiceDesc记录了服务的描述和映射信息
var Greeter_ServiceDesc = grpc.ServiceDesc{
	ServiceName: "helloworld.Greeter",
	HandlerType: (*GreeterServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "SayHello",
			Handler:    _Greeter_SayHello_Handler,
		},
	},
	Streams:  []grpc.StreamDesc{},
	Metadata: "examples/helloworld/helloworld/helloworld.proto",
}

```

注册过程

```go
func RegisterGreeterServer(s grpc.ServiceRegistrar, srv GreeterServer) {
   // s的类型为Server,调用下面的方法
   // Greeter_ServiceDesc记录了服务的描述和映射信息
   s.RegisterService(&Greeter_ServiceDesc, srv)
}

func (s *Server) RegisterService(sd *ServiceDesc, ss interface{}) {
	if ss != nil {
    // 传递的ss必须实现GreeterServer定义的接口
		ht := reflect.TypeOf(sd.HandlerType).Elem()
		st := reflect.TypeOf(ss)
		if !st.Implements(ht) {
			logger.Fatalf("grpc: Server.RegisterService found the handler of type %v that does not satisfy %v", st, ht)
		}
	}
	s.register(sd, ss)
}

func (s *Server) register(sd *ServiceDesc, ss interface{}) {
	s.mu.Lock()
	defer s.mu.Unlock()
  // 属于EnableTracing的内容
	s.printf("RegisterService(%q)", sd.ServiceName)
  // s.serve该标示表示已开始服务，注册服务的调用必须在开始服务之前
	if s.serve {
		logger.Fatalf("grpc: Server.RegisterService after Server.Serve for %q", sd.ServiceName)
	}
  // 该服务已注册
	if _, ok := s.services[sd.ServiceName]; ok {
		logger.Fatalf("grpc: Server.RegisterService found duplicate service registration for %q", sd.ServiceName)
	}
  // 以下内容将ServiceDesc（示例中的Greeter_ServiceDesc）相关内容进行整合
	info := &serviceInfo{
		serviceImpl: ss,
		methods:     make(map[string]*MethodDesc),
		streams:     make(map[string]*StreamDesc),
		mdata:       sd.Metadata,
	}
	for i := range sd.Methods {
		d := &sd.Methods[i]
		info.methods[d.MethodName] = d
	}
	for i := range sd.Streams {
		d := &sd.Streams[i]
		info.streams[d.StreamName] = d
	}
  // 以ServiceName为key,加到Server.services中
	s.services[sd.ServiceName] = info
}
```

### 开始服务

```go
func (s *Server) Serve(lis net.Listener) error {
   s.mu.Lock()
   // 属于EnableTracing的内容
   s.printf("serving")
   // 置开始服务的标示。
   s.serve = true
   // 如服务已被关闭，lis为null,再次开始服务会返回错误
   if s.lis == nil {
      // Serve called after Stop or GracefulStop.
      s.mu.Unlock()
      lis.Close()
      return ErrServerStopped
   }

   // 激活的服务计数，便于关闭时处理
   s.serveWG.Add(1)
   defer func() {
      s.serveWG.Done()
      // Stop或GracefulStop被调用时，会执行quit.Fire()，quit.HasFired()为true
      if s.quit.HasFired() {
         // Stop or GracefulStop called; block until done and return nil.
         // Stop和GracefulStop执行的最后会执行done.Fire()，之后下面的才会停止阻塞
         <-s.done.Done()
      }
   }()

   // lis列表中维护连接信息
   ls := &listenSocket{Listener: lis}
   s.lis[ls] = true

   // channelz相关
   if channelz.IsOn() {
      ls.channelzID = channelz.RegisterListenSocket(ls, s.channelzID, lis.Addr().String())
   }
   s.mu.Unlock()

   defer func() {
      s.mu.Lock()
      if s.lis != nil && s.lis[ls] {
         ls.Close()
         delete(s.lis, ls)
      }
      s.mu.Unlock()
   }()

   // 临时错误重试的时间差
   var tempDelay time.Duration // how long to sleep on accept failure

   for {
      // 等待客户端的一次连接
      rawConn, err := lis.Accept()
      if err != nil {
         // 对于临时性的错误，进行一定范围内的重试。
         if ne, ok := err.(interface {
            Temporary() bool
         }); ok && ne.Temporary() {
            if tempDelay == 0 {
               tempDelay = 5 * time.Millisecond
            } else {
               // 不断延长重试的时间差
               tempDelay *= 2
            }
            if max := 1 * time.Second; tempDelay > max {
               tempDelay = max
            }
            s.mu.Lock()
            s.printf("Accept error: %v; retrying in %v", err, tempDelay)
            s.mu.Unlock()
            timer := time.NewTimer(tempDelay)
            select {
            case <-timer.C:
            case <-s.quit.Done():
               timer.Stop()
               return nil
            }
            continue
         }
         s.mu.Lock()
         s.printf("done serving; Accept = %v", err)
         s.mu.Unlock()

         if s.quit.HasFired() {
            return nil
         }
         return err
      }
      tempDelay = 0
      // Start a new goroutine to deal with rawConn so we don't stall this Accept
      // loop goroutine.
      //
      // Make sure we account for the goroutine so GracefulStop doesn't nil out
      // s.conns before this conn can be added.
      s.serveWG.Add(1)
      go func() {
        // 见Explain:处理原始连接
         s.handleRawConn(rawConn)
         s.serveWG.Done()
      }()
   }
}
```

### 处理原始连接

```go
func (s *Server) handleRawConn(rawConn net.Conn) {
   // 退出被调用，不再处理新连接
   if s.quit.HasFired() {
      rawConn.Close()
      return
   }
   // 对于原始连接，设置一个读写的截止日志
   rawConn.SetDeadline(time.Now().Add(s.opts.connectionTimeout))
  // 传输的安全验证，见Explain:useTransportAuthenticator
   conn, authInfo, err := s.useTransportAuthenticator(rawConn)
   if err != nil {
      // ErrConnDispatched means that the connection was dispatched away from
      // gRPC; those connections should be left open.
      // 如果err不为空，代表身份验证失败，但是对于ErrConnDispatched（从grpc分配的连接），应该忽略。
      if err != credentials.ErrConnDispatched {
         s.mu.Lock()
         s.errorf("ServerHandshake(%q) failed: %v", rawConn.RemoteAddr(), err)
         s.mu.Unlock()
         channelz.Warningf(logger, s.channelzID, "grpc: Server.Serve failed to complete security handshake from %q: %v", rawConn.RemoteAddr(), err)
         rawConn.Close()
      }
      rawConn.SetDeadline(time.Time{})
      return
   }

   // Finish handshaking (HTTP2)
   // 见Explain:newHTTP2Transport
   st := s.newHTTP2Transport(conn, authInfo)
   if st == nil {
      return
   }

   rawConn.SetDeadline(time.Time{})
   if !s.addConn(st) {
      return
   }
   go func() {
      s.serveStreams(st)
      s.removeConn(st)
   }()
}
```

### useTransportAuthenticator

应用层的传输安全协议：Application Layer Transport Security (ALTS)

```go
func (s *Server) useTransportAuthenticator(rawConn net.Conn) (net.Conn, credentials.AuthInfo, error) {
   if s.opts.creds == nil {
      return rawConn, nil, nil
   }
   // 进行身份验证的握手
   return s.opts.creds.ServerHandshake(rawConn)
}

type TransportCredentials interface {
  // 客户端握手
	ClientHandshake(context.Context, string, net.Conn) (net.Conn, AuthInfo, error)
  // 服务端握手
	ServerHandshake(net.Conn) (net.Conn, AuthInfo, error)
  // 返回安全协议的相关信息
	Info() ProtocolInfo
  // 克隆
	Clone() TransportCredentials
  // 覆盖服务的主机名称
	OverrideServerName(string) error
}

// 了解更多见Explain:TransportCredentials
```

### TransportCredentials

待更新

### newHTTP2Transport

```go
// newHTTP2Transport,基于http2,创建一个传输服务
func (s *Server) newHTTP2Transport(c net.Conn, authInfo credentials.AuthInfo) transport.ServerTransport {
  // 传输服务的配置取自 主server 的配置
	config := &transport.ServerConfig{
		MaxStreams:            s.opts.maxConcurrentStreams,
		AuthInfo:              authInfo,
		InTapHandle:           s.opts.inTapHandle,
		StatsHandler:          s.opts.statsHandler,
		KeepaliveParams:       s.opts.keepaliveParams,
		KeepalivePolicy:       s.opts.keepalivePolicy,
		InitialWindowSize:     s.opts.initialWindowSize,
		InitialConnWindowSize: s.opts.initialConnWindowSize,
		WriteBufferSize:       s.opts.writeBufferSize,
		ReadBufferSize:        s.opts.readBufferSize,
		ChannelzParentID:      s.channelzID,
		MaxHeaderListSize:     s.opts.maxHeaderListSize,
		HeaderTableSize:       s.opts.headerTableSize,
	}
  // 见Explain:newHTTP2Server
	st, err := transport.NewServerTransport("http2", c, config)
	if err != nil {
		s.mu.Lock()
		s.errorf("NewServerTransport(%q) failed: %v", c.RemoteAddr(), err)
		s.mu.Unlock()
		c.Close()
		channelz.Warning(logger, s.channelzID, "grpc: Server.Serve failed to create ServerTransport: ", err)
		return nil
	}

	return st
}	
```

### newHTTP2Server

```go
func newHTTP2Server(conn net.Conn, config *ServerConfig) (_ ServerTransport, err error) {
	writeBufSize := config.WriteBufferSize
	readBufSize := config.ReadBufferSize
	maxHeaderListSize := defaultServerMaxHeaderListSize
	if config.MaxHeaderListSize != nil {
		maxHeaderListSize = *config.MaxHeaderListSize
	}
  // 见Explain:newFramer
	framer := newFramer(conn, writeBufSize, readBufSize, maxHeaderListSize)
	// Send initial settings as connection preface to client.
	isettings := []http2.Setting{{
		ID:  http2.SettingMaxFrameSize,
		Val: http2MaxFrameLen,
	}}
	// TODO(zhaoq): Have a better way to signal "no limit" because 0 is
	// permitted in the HTTP2 spec.
	maxStreams := config.MaxStreams
	if maxStreams == 0 {
		maxStreams = math.MaxUint32
	} else {
		isettings = append(isettings, http2.Setting{
			ID:  http2.SettingMaxConcurrentStreams,
			Val: maxStreams,
		})
	}
	dynamicWindow := true
	iwz := int32(initialWindowSize)
	if config.InitialWindowSize >= defaultWindowSize {
		iwz = config.InitialWindowSize
		dynamicWindow = false
	}
	icwz := int32(initialWindowSize)
	if config.InitialConnWindowSize >= defaultWindowSize {
		icwz = config.InitialConnWindowSize
		dynamicWindow = false
	}
	if iwz != defaultWindowSize {
		isettings = append(isettings, http2.Setting{
			ID:  http2.SettingInitialWindowSize,
			Val: uint32(iwz)})
	}
	if config.MaxHeaderListSize != nil {
		isettings = append(isettings, http2.Setting{
			ID:  http2.SettingMaxHeaderListSize,
			Val: *config.MaxHeaderListSize,
		})
	}
	if config.HeaderTableSize != nil {
		isettings = append(isettings, http2.Setting{
			ID:  http2.SettingHeaderTableSize,
			Val: *config.HeaderTableSize,
		})
	}
  // 根据配置，向客户端写一个setting帧
	if err := framer.fr.WriteSettings(isettings...); err != nil {
		return nil, connectionErrorf(false, err, "transport: %v", err)
	}
	// Adjust the connection flow control window if needed.
  // 这里也是向客户端写一个setting帧
	if delta := uint32(icwz - defaultWindowSize); delta > 0 {
		if err := framer.fr.WriteWindowUpdate(0, delta); err != nil {
			return nil, connectionErrorf(false, err, "transport: %v", err)
		}
	}
  // 设置keepalive参数
	kp := config.KeepaliveParams
	if kp.MaxConnectionIdle == 0 {
		kp.MaxConnectionIdle = defaultMaxConnectionIdle
	}
	if kp.MaxConnectionAge == 0 {
		kp.MaxConnectionAge = defaultMaxConnectionAge
	}
	// Add a jitter to MaxConnectionAge.
	kp.MaxConnectionAge += getJitter(kp.MaxConnectionAge)
	if kp.MaxConnectionAgeGrace == 0 {
		kp.MaxConnectionAgeGrace = defaultMaxConnectionAgeGrace
	}
	if kp.Time == 0 {
		kp.Time = defaultServerKeepaliveTime
	}
	if kp.Timeout == 0 {
		kp.Timeout = defaultServerKeepaliveTimeout
	}
	kep := config.KeepalivePolicy
	if kep.MinTime == 0 {
		kep.MinTime = defaultKeepalivePolicyMinTime
	}
	done := make(chan struct{})
	t := &http2Server{
		ctx:               context.Background(),
		done:              done,
		conn:              conn,
		remoteAddr:        conn.RemoteAddr(),
		localAddr:         conn.LocalAddr(),
		authInfo:          config.AuthInfo,
		framer:            framer,
		readerDone:        make(chan struct{}),
		writerDone:        make(chan struct{}),
		maxStreams:        maxStreams,
		inTapHandle:       config.InTapHandle,
		fc:                &trInFlow{limit: uint32(icwz)},
		state:             reachable,
		activeStreams:     make(map[uint32]*Stream),
		stats:             config.StatsHandler,
		kp:                kp,
		idle:              time.Now(),
		kep:               kep,
		initialWindowSize: iwz,
		czData:            new(channelzData),
		bufferPool:        newBufferPool(),
	}
  // 见Explain:controlBuffer
	t.controlBuf = newControlBuffer(t.done)
  // 如果自定义了窗口尺寸，则初始化一个bdp估算器
	if dynamicWindow {
		t.bdpEst = &bdpEstimator{
			bdp:               initialWindowSize,
			updateFlowControl: t.updateFlowControl,
		}
	}
  // t.stats的类型为stats.Handler
  // 见Explain:stats.Handler
	if t.stats != nil {
		t.ctx = t.stats.TagConn(t.ctx, &stats.ConnTagInfo{
			RemoteAddr: t.remoteAddr,
			LocalAddr:  t.localAddr,
		})
		connBegin := &stats.ConnBegin{}
		t.stats.HandleConn(t.ctx, connBegin)
	}
  // channelz相关
	if channelz.IsOn() {
		t.channelzID = channelz.RegisterNormalSocket(t, config.ChannelzParentID, fmt.Sprintf("%s -> %s", t.remoteAddr, t.localAddr))
	}

  // 连接计数器增1
	t.connectionID = atomic.AddUint64(&serverConnectionCounter, 1)

  // Writer flush
	t.framer.writer.Flush()

	defer func() {
		if err != nil {
			t.Close()
		}
	}()

  // 检查客户端序言
	preface := make([]byte, len(clientPreface))
	if _, err := io.ReadFull(t.conn, preface); err != nil {
		return nil, connectionErrorf(false, err, "transport: http2Server.HandleStreams failed to receive the preface from client: %v", err)
	}
	if !bytes.Equal(preface, clientPreface) {
		return nil, connectionErrorf(false, nil, "transport: http2Server.HandleStreams received bogus greeting from client: %q", preface)
	}

  // 从连接中读取一个帧（此帧为连接建立后客户端发送的setting帧）
	frame, err := t.framer.fr.ReadFrame()
	if err == io.EOF || err == io.ErrUnexpectedEOF {
		return nil, err
	}
	if err != nil {
		return nil, connectionErrorf(false, err, "transport: http2Server.HandleStreams failed to read initial settings frame: %v", err)
	}
  // 记录lastRead时间点
	atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())
  // 转换成setting帧
	sf, ok := frame.(*http2.SettingsFrame)
	if !ok {
		return nil, connectionErrorf(false, nil, "transport: http2Server.HandleStreams saw invalid preface type %T from client", frame)
	}
  // 服务器收到客户端的setting帧后，开始处理
	t.handleSettings(sf)

  // 见Explain:loopyWriter
	go func() {
		t.loopy = newLoopyWriter(serverSide, t.framer, t.controlBuf, t.bdpEst)
		t.loopy.ssGoAwayHandler = t.outgoingGoAwayHandler
    // 见Explain:loopyWriter.run
		if err := t.loopy.run(); err != nil {
			if logger.V(logLevel) {
				logger.Errorf("transport: loopyWriter.run returning. Err: %v", err)
			}
		}
		t.conn.Close()
		close(t.writerDone)
	}()
	go t.keepalive()
	return t, nil
}

```

### newFramer

```go
func newFramer(conn net.Conn, writeBufferSize, readBufferSize int, maxHeaderListSize uint32) *framer {
   if writeBufferSize < 0 {
      writeBufferSize = 0
   }
   var r io.Reader = conn
   if readBufferSize > 0 {
      // 指定大小的Reader
      r = bufio.NewReaderSize(r, readBufferSize)
   }
   //指定大小的Writer(bufWriter类型)
   w := newBufWriter(conn, writeBufferSize)
   // 此类型结构在下方，注意与http2.Framer区分
   f := &framer{
      writer: w,
      fr:     http2.NewFramer(w, r),
   }
   f.fr.SetMaxReadFrameSize(http2MaxFrameLen)
   // Opt-in to Frame reuse API on framer to reduce garbage.
   // Frames aren't safe to read from after a subsequent call to ReadFrame.
   f.fr.SetReuseFrames()
   f.fr.MaxHeaderListSize = maxHeaderListSize
   // ReadMetaHeaders如果被设置，则HEADERS帧和CONTINUA帧会一起返回
   // 具体需了解HTTP/2--HPACK算法
   f.fr.ReadMetaHeaders = hpack.NewDecoder(http2InitHeaderTableSize, nil)
   return f
}

type bufWriter struct {
	buf       []byte
	offset    int
	batchSize int
	conn      net.Conn
	err       error
	onFlush func()
}

type framer struct {
	writer *bufWriter
	fr     *http2.Framer
}
```

### controlBuffer

```go
func newControlBuffer(done <-chan struct{}) *controlBuffer {
	return &controlBuffer{
		ch:   make(chan struct{}, 1),
		list: &itemList{},
		done: done,
	}
}
```

```go
// controlBuffer is a way to pass information to loopy.
// Information is passed as specific struct types called control frames.
// A control frame not only represents data, messages or headers to be sent out
// but can also be used to instruct loopy to update its internal state.
// It shouldn't be confused with an HTTP2 frame, although some of the control frames
// like dataFrame and headerFrame do go out on wire as HTTP2 frames.

// 1.传递控制帧( data, messages or headers)给loopy
// 2.控制loopy去更新内部的状态
type controlBuffer struct {
   ch              chan struct{}
   done            <-chan struct{}
   mu              sync.Mutex
   consumerWaiting bool
   list            *itemList
   err             error
   transportResponseFrames int
   trfChan                 atomic.Value // *chan struct{}
}
```

### stats.Handler

```go
// 相关统计数据的接口
type Handler interface {
   // 将一些RPC信息附加到上下文中
   TagRPC(context.Context, *RPCTagInfo) context.Context
   // 处理RPC的统计信息
   HandleRPC(context.Context, RPCStats)
   // 将一些连接信息附加到上下文中
   TagConn(context.Context, *ConnTagInfo) context.Context
   // 处理连接的统计信息
   HandleConn(context.Context, ConnStats)
}
```

### handleSettings

```go
func (t *http2Server) handleSettings(f *http2.SettingsFrame) {
   // f.IsAck代表是空的setting帧
   if f.IsAck() {
      return
   }
   // 有些setting是需要执行函数的，有些setting需要传递给 t.controlBuf做进一步的处理
   var ss []http2.Setting
   var updateFuncs []func()
   f.ForeachSetting(func(s http2.Setting) error {
      switch s.ID {
      case http2.SettingMaxHeaderListSize:
         updateFuncs = append(updateFuncs, func() {
            t.maxSendHeaderListSize = new(uint32)
            *t.maxSendHeaderListSize = s.Val
         })
      default:
         ss = append(ss, s)
      }
      return nil
   })
   // 往下看
   t.controlBuf.executeAndPut(func(interface{}) bool {
      for _, f := range updateFuncs {
         f()
      }
      return true
   }, &incomingSettings{
      ss: ss,
   })
}
```

```go
func (f *SettingsFrame) ForeachSetting(fn func(Setting) error) error {
   // 检查帧的有效性
   f.checkValid()
   for i := 0; i < f.NumSettings(); i++ {
      if err := fn(f.Setting(i)); err != nil {
         return err
      }
   }
   return nil
}
```

```go
// f.p为帧的数据部分，每个setting项的长度是6字节
func (f *SettingsFrame) NumSettings() int { return len(f.p) / 6 }
```

```go
func (c *controlBuffer) executeAndPut(f func(it interface{}) bool, it cbItem) (bool, error) {
  // 需要唤醒get(消费者)的标示
   var wakeUp bool
   c.mu.Lock()
   if c.err != nil {
      c.mu.Unlock()
      return false, c.err
   }
   if f != nil {
      if !f(it) { // f wasn't successful
         c.mu.Unlock()
         return false, nil
      }
   }
   // 如果get 在等待消费
   if c.consumerWaiting {
      // 需要唤醒get
      wakeUp = true
      // 将消费等待置false
      c.consumerWaiting = false
   }
  // 将it加到c.list(消费池中)
   c.list.enqueue(it)
   // 如果是这个帧类型
   if it.isTransportResponseFrame() {
      // 该计数+1
      c.transportResponseFrames++
      // 如果已等于最大的量，则需要创建一个节流channel
      if c.transportResponseFrames == maxQueuedTransportResponseFrames {
         // We are adding the frame that puts us over the threshold; create
         // a throttling channel.
         ch := make(chan struct{})
         c.trfChan.Store(&ch)
      }
   }
   c.mu.Unlock()
   // 如果需要唤醒
   // 通过c.ch做唤醒动作
   if wakeUp {
      select {
      case c.ch <- struct{}{}:
      default:
      }
   }
   return true, nil
}
```

### loopyWriter

```go
type loopyWriter struct {
   side      side
   cbuf      *controlBuffer
   sendQuota uint32
   oiws      uint32 // outbound initial window size.
   estdStreams map[uint32]*outStream // Established streams.
   activeStreams *outStreamList
   framer        *framer
   hBuf          *bytes.Buffer  // The buffer for HPACK encoding.
   hEnc          *hpack.Encoder // HPACK encoder.
   bdpEst        *bdpEstimator
   draining      bool
   ssGoAwayHandler func(*goAway) (bool, error)
}
```

### loopyWriter.run

```go
func (l *loopyWriter) run() (err error) {
   // 关闭连接所产生的错误，不应该作为run内部产生的错误，将错误置空
   defer func() {
      if err == ErrConnClosing {
         if logger.V(logLevel) {
            logger.Infof("transport: loopyWriter.run returning. %v", err)
         }
         err = nil
      }
   }()
   for {
      // 见Explain:loopyWriter.get
      it, err := l.cbuf.get(true)
      if err != nil {
         return err
      }
     // 见Explain:loopyWriter.handle
      if err = l.handle(it); err != nil {
         return err
      }
     // 见Explain:processData
      if _, err = l.processData(); err != nil {
         return err
      }
      gosched := true
   hasdata:
      for {
         it, err := l.cbuf.get(false)
         if err != nil {
            return err
         }
         if it != nil {
            if err = l.handle(it); err != nil {
               return err
            }
            if _, err = l.processData(); err != nil {
               return err
            }
            continue hasdata
         }
         isEmpty, err := l.processData()
         if err != nil {
            return err
         }
         if !isEmpty {
            continue hasdata
         }
         if gosched {
            gosched = false
            if l.framer.writer.offset < minBatchSize {
               runtime.Gosched()
               continue hasdata
            }
         }
         l.framer.writer.Flush()
         break hasdata

      }
   }
}
```

### controlBuffer.get

```go
func (c *controlBuffer) get(block bool) (interface{}, error) {
   for {
      c.mu.Lock()
      if c.err != nil {
         c.mu.Unlock()
         return nil, c.err
      }
      // 如果有消费的元素（c.list）
      if !c.list.isEmpty() {
         // 出队一个元素
         h := c.list.dequeue().(cbItem)
         // 当是该帧类型，且数量下降到最大值，则销毁节流通道
         if h.isTransportResponseFrame() {
            if c.transportResponseFrames == maxQueuedTransportResponseFrames {
               ch := c.trfChan.Load().(*chan struct{})
               close(*ch)
               c.trfChan.Store((*chan struct{})(nil))
            }
            c.transportResponseFrames--
         }
         c.mu.Unlock()
         return h, nil
      }
      if !block {
         c.mu.Unlock()
         return nil, nil
      }
      // 置消费等待
      c.consumerWaiting = true
      c.mu.Unlock()
      // 通过c.ch等待被唤醒
      select {
      case <-c.ch:
      case <-c.done:
         c.finish()
         return nil, ErrConnClosing
      }
   }
}
```

### loopyWriter.handle

```go
func (l *loopyWriter) handle(i interface{}) error {
   switch i := i.(type) {
   case *incomingWindowUpdate:
      return l.incomingWindowUpdateHandler(i)
   case *outgoingWindowUpdate:
      return l.outgoingWindowUpdateHandler(i)
   case *incomingSettings:
      return l.incomingSettingsHandler(i)
   case *outgoingSettings:
      return l.outgoingSettingsHandler(i)
   case *headerFrame:
      return l.headerHandler(i)
   case *registerStream:
      return l.registerStreamHandler(i)
   case *cleanupStream:
      return l.cleanupStreamHandler(i)
   case *incomingGoAway:
      return l.incomingGoAwayHandler(i)
   case *dataFrame:
      return l.preprocessData(i)
   case *ping:
      return l.pingHandler(i)
   case *goAway:
      return l.goAwayHandler(i)
   case *outFlowControlSizeRequest:
      return l.outFlowControlSizeRequestHandler(i)
   default:
      return fmt.Errorf("transport: unknown control message type %T", i)
   }
}
```

以incomingSettings为例（loopy.run之前客户端传入的setting帧作为incomingSettings类型，放在了controlBuffer.lis(消费池)，被取出）



```go
// 对于incomingSettings类型，执行该方法
func (l *loopyWriter) incomingSettingsHandler(s *incomingSettings) error {
  // 见Explain:applySettings
   if err := l.applySettings(s.ss); err != nil {
      return err
   }
   return l.framer.fr.WriteSettingsAck()
}
```

### applySettings

```go
func (l *loopyWriter) applySettings(ss []http2.Setting) error {
   for _, s := range ss {
      switch s.ID {
      // 如果setting帧的帧类型为SettingInitialWindowSize
      case http2.SettingInitialWindowSize:
         o := l.oiws
         l.oiws = s.Val
         if o < l.oiws {
            // If the new limit is greater make all depleted streams active.
            // 如果新的窗口尺寸大于原来的，则将等待分配的流全置入激活流中
            for _, stream := range l.estdStreams {
               if stream.state == waitingOnStreamQuota {
                  stream.state = active
                  l.activeStreams.enqueue(stream)
               }
            }
         }
      // 该类型执行该方法  
      case http2.SettingHeaderTableSize:
         updateHeaderTblSize(l.hEnc, s.Val)
      }
   }
   return nil
}
```

### WriteSettingsAck

```go
// 对于客户端的setting帧，需要一个ack答复
func (f *Framer) WriteSettingsAck() error {
   f.startWrite(FrameSettings, FlagSettingsAck, 0)
   return f.endWrite()
}
```

### processData

```go
func (l *loopyWriter) processData() (bool, error) {
   if l.sendQuota == 0 {
      return true, nil
   }
   str := l.activeStreams.dequeue() // Remove the first stream.
   if str == nil {
      return true, nil
   }
   dataItem := str.itl.peek().(*dataFrame) // Peek at the first data item this stream.
   // A data item is represented by a dataFrame, since it later translates into
   // multiple HTTP2 data frames.
   // Every dataFrame has two buffers; h that keeps grpc-message header and d that is acutal data.
   // As an optimization to keep wire traffic low, data from d is copied to h to make as big as the
   // maximum possilbe HTTP2 frame size.

   if len(dataItem.h) == 0 && len(dataItem.d) == 0 { // Empty data frame
      // Client sends out empty data frame with endStream = true
      if err := l.framer.fr.WriteData(dataItem.streamID, dataItem.endStream, nil); err != nil {
         return false, err
      }
      str.itl.dequeue() // remove the empty data item from stream
      if str.itl.isEmpty() {
         str.state = empty
      } else if trailer, ok := str.itl.peek().(*headerFrame); ok { // the next item is trailers.
         if err := l.writeHeader(trailer.streamID, trailer.endStream, trailer.hf, trailer.onWrite); err != nil {
            return false, err
         }
         if err := l.cleanupStreamHandler(trailer.cleanup); err != nil {
            return false, nil
         }
      } else {
         l.activeStreams.enqueue(str)
      }
      return false, nil
   }
   var (
      buf []byte
   )
   // Figure out the maximum size we can send
   maxSize := http2MaxFrameLen
   if strQuota := int(l.oiws) - str.bytesOutStanding; strQuota <= 0 { // stream-level flow control.
      str.state = waitingOnStreamQuota
      return false, nil
   } else if maxSize > strQuota {
      maxSize = strQuota
   }
   if maxSize > int(l.sendQuota) { // connection-level flow control.
      maxSize = int(l.sendQuota)
   }
   // Compute how much of the header and data we can send within quota and max frame length
   hSize := min(maxSize, len(dataItem.h))
   dSize := min(maxSize-hSize, len(dataItem.d))
   if hSize != 0 {
      if dSize == 0 {
         buf = dataItem.h
      } else {
         // We can add some data to grpc message header to distribute bytes more equally across frames.
         // Copy on the stack to avoid generating garbage
         var localBuf [http2MaxFrameLen]byte
         copy(localBuf[:hSize], dataItem.h)
         copy(localBuf[hSize:], dataItem.d[:dSize])
         buf = localBuf[:hSize+dSize]
      }
   } else {
      buf = dataItem.d
   }

   size := hSize + dSize

   // Now that outgoing flow controls are checked we can replenish str's write quota
   str.wq.replenish(size)
   var endStream bool
   // If this is the last data message on this stream and all of it can be written in this iteration.
   if dataItem.endStream && len(dataItem.h)+len(dataItem.d) <= size {
      endStream = true
   }
   if dataItem.onEachWrite != nil {
      dataItem.onEachWrite()
   }
   if err := l.framer.fr.WriteData(dataItem.streamID, endStream, buf[:size]); err != nil {
      return false, err
   }
   str.bytesOutStanding += size
   l.sendQuota -= uint32(size)
   dataItem.h = dataItem.h[hSize:]
   dataItem.d = dataItem.d[dSize:]

   if len(dataItem.h) == 0 && len(dataItem.d) == 0 { // All the data from that message was written out.
      str.itl.dequeue()
   }
   if str.itl.isEmpty() {
      str.state = empty
   } else if trailer, ok := str.itl.peek().(*headerFrame); ok { // The next item is trailers.
      if err := l.writeHeader(trailer.streamID, trailer.endStream, trailer.hf, trailer.onWrite); err != nil {
         return false, err
      }
      if err := l.cleanupStreamHandler(trailer.cleanup); err != nil {
         return false, err
      }
   } else if int(l.oiws)-str.bytesOutStanding <= 0 { // Ran out of stream quota.
      str.state = waitingOnStreamQuota
   } else { // Otherwise add it back to the list of active streams.
      l.activeStreams.enqueue(str)
   }
   return false, nil
}
```

### HandleStreams

```go
func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
   defer close(t.readerDone)
   for {
      t.controlBuf.throttle()
      frame, err := t.framer.fr.ReadFrame()
      atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())
      if err != nil {
         if se, ok := err.(http2.StreamError); ok {
            if logger.V(logLevel) {
               logger.Warningf("transport: http2Server.HandleStreams encountered http2.StreamError: %v", se)
            }
            t.mu.Lock()
            s := t.activeStreams[se.StreamID]
            t.mu.Unlock()
            if s != nil {
               t.closeStream(s, true, se.Code, false)
            } else {
               t.controlBuf.put(&cleanupStream{
                  streamID: se.StreamID,
                  rst:      true,
                  rstCode:  se.Code,
                  onWrite:  func() {},
               })
            }
            continue
         }
         if err == io.EOF || err == io.ErrUnexpectedEOF {
            t.Close()
            return
         }
         if logger.V(logLevel) {
            logger.Warningf("transport: http2Server.HandleStreams failed to read frame: %v", err)
         }
         t.Close()
         return
      }
      switch frame := frame.(type) {
      case *http2.MetaHeadersFrame:
         if t.operateHeaders(frame, handle, traceCtx) {
            t.Close()
            break
         }
      case *http2.DataFrame:
         t.handleData(frame)
      case *http2.RSTStreamFrame:
         t.handleRSTStream(frame)
      case *http2.SettingsFrame:
         t.handleSettings(frame)
      case *http2.PingFrame:
         t.handlePing(frame)
      case *http2.WindowUpdateFrame:
         t.handleWindowUpdate(frame)
      case *http2.GoAwayFrame:
         // TODO: Handle GoAway from the client appropriately.
      default:
         if logger.V(logLevel) {
            logger.Errorf("transport: http2Server.HandleStreams found unhandled frame type %v.", frame)
         }
      }
   }
}
```



## 3. Learned

### 链式拦截器

根据chainUnaryServerInterceptors源码实现规则，进行了一个精简版实现。

``` go
package test

import (
	"fmt"
	"testing"
)

// 与源码名称一样，对参数结构做了精简便于理解
type UnaryHandler func()(interface{}, error)

type UnaryServerInterceptor func(handler UnaryHandler)(interface{}, error)


func getChainUnaryHandler(interceptors []UnaryServerInterceptor,curr int,finalHandle UnaryHandler)UnaryHandler  {
	if  curr == len(interceptors) - 1 {
		return finalHandle
	}
	return func()(interface{}, error){
		return interceptors[curr+1](getChainUnaryHandler(interceptors, curr+1, finalHandle))
	}
}

func TestRecursion(t *testing.T) {

  // 三个拦截器
	f1 := UnaryServerInterceptor(func(handler UnaryHandler) (interface{}, error){
		fmt.Println(1)
		_,_ = handler()
		return nil,nil
	})

	f2 := UnaryServerInterceptor(func(handler UnaryHandler) (interface{}, error) {
		fmt.Println(2)
		_,_ = handler()
		return nil,nil
	})

	f3 := UnaryServerInterceptor(func(handler UnaryHandler) (interface{}, error) {
		fmt.Println(3)
		_,_ = handler()
		return nil,nil
	})

  
	interceptors := []UnaryServerInterceptor{f1, f2, f3}

  // 生成一个链式拦截器
  // f1(f2(f3(main)))
	chainedInt := func(handler UnaryHandler)(interface{}, error) {
		return interceptors[0](getChainUnaryHandler(interceptors, 0, handler))
	}

  // 测试输出
	mainFunc := func() (interface{}, error) {
		fmt.Println("main...")
		return nil,nil
	}
	_,_ = chainedInt(mainFunc)
	// 输出 1 2 3 main...
}
```

### EnableTracing

官网地址：https://pkg.go.dev/golang.org/x/net/trace#pkg-functions
 如官网所说，实现请求和长对象的追踪，并提供http视图查看
如果开启此行为，在本方法执行之前进行设置，后续相关内容不再陈述。

待更新

### channelz

待更新

typora

