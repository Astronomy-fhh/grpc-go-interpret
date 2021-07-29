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
  // 见Explain:RegisterGreeterServer
	pb.RegisterGreeterServer(s, &server{})
  // 见Explain:server
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
  // 官网地址：https://pkg.go.dev/golang.org/x/net/trace#pkg-functions
  // 如官网所说，实现请求和长对象的追踪，并提供http视图查看
  // 如果开启此行为，在本方法执行之前进行设置，后续相关内容不再陈述。
	if EnableTracing {
		_, file, line, _ := runtime.Caller(1)
		s.events = trace.NewEventLog("grpc.Server", fmt.Sprintf("%s:%d", file, line))
	}

	if s.opts.numServerWorkers > 0 {
		s.initServerWorkers()
	}

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
	threshold := serverWorkerResetThreshold + grpcrand.Intn(serverWorkerResetThreshold)
	for completed := 0; completed < threshold; completed++ {
		data, ok := <-ch
		if !ok {
			return
		}
		s.handleStream(data.st, data.stream, s.traceInfo(data.st, data.stream))
		data.wg.Done()
	}
	go s.serverWorker(ch)
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

