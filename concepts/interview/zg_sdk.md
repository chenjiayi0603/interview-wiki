# Zego 服务网格 SDK (zg_sdk)

## 概述

本模块提供了访问服务网格内的服务以及提供服务的 API。此模块 API 与原生（Go 标准库、gRPC 等）差异不大，但底层与 **Sidecar / 服务发现 / 元数据透传** 强绑定，因此仅适用于 **网格内东西向流量**；访问公网或集群外服务请用标准库客户端。

**行业侧**：类似 **Istio + Envoy**、Linkerd 或自研 mesh 下的应用 SDK——通过本地代理与统一配置（`zego-micro.yaml`）完成服务名解析、mTLS、路由与观测字段注入。

开发者需要在项目根路径下 `conf` 目录自行维护 `zego-micro.yaml` 来声明服务相关信息，示例如下：
```yaml
service: service-example # 服务名
server: # 本地服务及监听端口，需要时声明
  http:
    port: 9000 # http默认端口
  grpc:
    port: 9001 # grpc默认端口
  healthy:
    port: 9999 # 健康检查默认端口，不建议修改
called-services:
  - ipsvr.rtc # 该服务调用的服务的服务条目（#服务名.#命名空间）
```

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

## zgrpc

对grpc框架的封装。特别地，当需要给server添加处理中间件时，使用`zgrpc.UnaryServerInterceptor`或`zgrpc.StreamServerInterceptor`来声明server的一元/流式请求中间件，如果使用原生的`grpc.UnaryInterceptor`和`grpc.StreamInterceptor`来声明中间件则会引发panic。

同样地，使用`zgrpc.UnaryClientInterceptor`或`zgrpc.StreamClientInterceptor`来声明客户端一元/流式请求中间件。

说明：用户自定义的中间件和框架内置中间件需要分配正确的顺序以初始化请求上下文，使用原生方法将造成顺序分配失效，从而造成相关API的使用的异常。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

### client

使用`zgrpc.NewDialer`创建一个链接对象，参数为在`conf/zego-micro.yaml`中声明好的服务条目，使用该链接对象即可初始化grpc客户端，初始化时可调用`WithDialOptions`声明初始化客户端参数。

**使用场景**：网格内通过“服务名.命名空间”直连下游 gRPC 服务，同时统一挂载超时、重试、追踪等客户端中间件。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

### server

调用`zgrpc.NewServerBuilder`创建一个server建造者对象，并且调用`WithServerOptions`来声明server初始化参数，调用`WithRegister`来声明该server的处理函数，最后调用`Build`来获得一个server，调用server的`RunInBackground`方法来非阻塞地启动该服务。

**使用场景**：对外暴露 gRPC 能力时，通过统一 builder 注入框架中间件（metrics/metadata/debug），避免业务侧重复拼装拦截器链。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

### gRPC 测试

**场景**：网格内 gRPC 端到端自测——校验 **元数据随 RPC 透传**、客户端 **UnaryInterceptor** 是否生效，以及可选的 **流量标签**（灰度 / 泳道与 Istio `VirtualService` 按 header 路由类似）。

```go
type testServer struct {
	grpc_testing.TestServiceServer
}

func (t *testServer) UnaryCall(ctx context.Context, request *grpc_testing.SimpleRequest) (*grpc_testing.SimpleResponse, error) {
	var m map[string]string
	err := json.Unmarshal(request.Payload.Body, &m)
	if err != nil {
		return nil, err
	}
	md := zmetadata.FromContext(ctx)
	if md == nil {
		return nil, errors.New("nil md")
	}
	for k := range m {
		v, b := md.GetDownstream(k)
		if !b {
			return nil, errors.New("md is not exist:" + k)
		}
		if v != m[k] {
			return nil, errors.New("md is not match:" + k)
		}
	}

	// test test traffic label
	want, ok := os.LookupEnv("TEST_TRAFFIC_LABEL")
	if !ok {
		return &grpc_testing.SimpleResponse{}, nil
	}
	get, b := md.GetDownstream("test-traffic-label")
	if !b {
		return nil, errors.New("there is not a test traffic label")
	}
	if get != want {
		return nil, errors.New("test traffic label is not match")
	}
	return &grpc_testing.SimpleResponse{}, nil
}

func TestGrpc(t *testing.T) {
	type tc struct {
		port int
	}
	for _, c := range []tc{
		{50001}, {50002},
	} {
		addr := ":" + strconv.Itoa(c.port)
		t.Run("test "+addr, func(t *testing.T) {
			sb := NewServerBuilder()
			sb.WithRegister(grpc_testing.RegisterTestServiceServer, &testServer{})
			s := sb.Build()
			s.address = addr
			defer s.Stop()

			s.RunInBackground()

			time.Sleep(time.Second)
			timeout, f := context.WithTimeout(context.Background(), time.Second)
			defer f()

			network.IgnoredIP = "127.0.0.1" + addr

			var set bool
			d := NewDialer("host.ns").WithContext(timeout).WithDialOptions(UnaryClientInterceptor(func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
				set = true
				return invoker(ctx, method, req, reply, cc, opts...)
			}), grpc.WithBlock(), grpc.WithInsecure())
			conn, err := d.Dial()
			if err != nil {
				t.Error(err)
				return
			}
			if conn.GetState() != connectivity.Ready {
				t.Error("connection failed")
				return
			}
			md := map[string]string{
				"k": "v",
			}
			out := zmetadata.FromContext(nil)
			for k, v := range md {
				err = out.SetUpstreamGlobal(k, v)
				if err != nil {
					t.Error(err)
					return
				}
				err = out.SetUpstream(k, v)
				if err != nil {
					t.Error(err)
					return
				}
			}
			if err != nil {
				panic(err)
			}
			client := grpc_testing.NewTestServiceClient(conn)
			in := &grpc_testing.SimpleRequest{}
			marshal, _ := json.Marshal(md)
			in.Payload = &grpc_testing.Payload{
				Body: marshal,
			}
			_, err = client.UnaryCall(zmetadata.OutgoingContext(out), in)
			if !set {
				t.Error("interceptor not working!")
			}
			if err != nil {
				t.Error(err)
			}
		})
	}
}
```

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

## zhttp

对标准库http模块的封装。

### client

调用NewClientBuilder来创建客户端建造者对象，参数为在`conf/zego-micro.yaml`中声明好的服务条目，之后调用WithFilters来声明过滤器，最后调用Build函数来创建http客户端。使用客户端的Do方法来发送请求（由于被调用的服务在在初始化客户端时就被指定了，所以在Do方法的参数中不需要指定调用域名等信息）。

http客户端默认初始化10个全局链接来与proxy通信，通信协议为http2。

**使用场景**：服务 B 在处理用户请求时需要调用服务 C 的 HTTP 接口，要求自动透传 trace/header，并复用到 Sidecar 的长连接能力。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

### server

调用`zhttp.NewServer`创建一个server对象，并且调用`WithFilters`方法来声明服务级别的过滤器，最后用`RunInBackground`方法来非阻塞地启动该服务。

**使用场景**：需要快速提供 HTTP 接口并复用统一过滤器（鉴权、日志、熔断、观测埋点）时，用 zhttp server 统一接入。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

### HTTP 测试

**场景**：Sidecar / 本地代理走 **HTTP/2** 多路复用时的集成测试——验证 **zmetadata**、**B3/x-b3-traceid** 类追踪头与下游再发起 HTTP 调用时的上下文延续。

**zmetadata** 主要用于在微服务调用链中传递自定义的元数据（metadata），比如服务之间需要共享的一些上下文信息。这样可以保证在服务调用过程中，必要的信息，例如用户身份、语言环境、业务标识等，可以随着 RPC 或 HTTP 调用一同传递到下游服务，以便做上下文感知或者链路分析。  
常见用法是在中间件或过滤器中，从请求上下文中读取、注入或修改这些元数据。

**B3/x-b3-traceid** 则属于分布式链路追踪的标准头部。B3 是 Zipkin 推广的一套链路追踪上下文标准，`x-b3-traceid` 指定的是当前请求链路的全局唯一 Trace ID。每个服务在转发请求时会保留和传递这些 trace/head/span 头部，从而保证在链路分析工具（如 Zipkin、Jaeger）中，可以准确地重建分布式系统中的一次完整调用轨迹，实现服务调用的链路追踪和问题定位。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

## zmetadata

尽管在响应请求时，这里的元数据处理是与请求上下文关联的，所以必须使用该模块API来对元数据进行读写操作。

### 获取或自行创建元数据集合

使用`zmetadata.FromContext`来获取当前请求上下文包含的元数据集合,当使用`nil`作为参数时代表创建一个不与请求上下文关联的元数据集合。

**使用场景**：入站请求处理函数里读取上游传来的 trace id、租户、语言、灰度标签，或在离线任务里手动创建一份 metadata 发起下游调用。

***注意：由于语言特性限制，如果需要在一个请求处理处理函数中发起一次请求，必须显式地使用包含当前请求上下文的context对象来构造这一次请求以传递元数据、链路追踪等信息***

即使不需要对请求上下文中的元数据进行读写时，也需要调用`zmetadata.FromContext`和`zmetadata.OutgoingContext`两个函数来显式传递上下文信息。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

### 读取

使用`GetDownstream`来读取元数据。

**使用场景**：在业务逻辑中读取用户身份、实验组标识、链路追踪字段，决定路由策略或记录审计日志。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

### 写入

使用`SetUpstream`来写入元数据条目。当构造好元数据后，使用`zmetadata.OutgoingContext`来将需要发送的请求和当前元数据关联起来并构造好一个新的请求上下文。

**使用场景**：服务内转调下游前补充业务上下文（如 room_id、tenant_id、request_source），确保跨服务链路具备完整上下文信息。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

## gRPC / HTTP 拦截器链

### 初始化拦截器

**场景**：框架在 **SO/DO（Server/Client gRPC）**、**HH/RR（HTTP Server/Client）** 上预置链式中间件——对应行业里常见的 *metrics、metadata、debug logging* 分层，顺序由框架保证，避免业务混用 `grpc.WithChainUnaryInterceptor` 时顺序错乱。

**使用场景**：团队内多服务并行开发时，把“可观测性和上下文透传”收敛到统一拦截器链，业务代码只关注 handler 本身。

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-说明-说明.md]

## 相关页面

- [[服务网格平台-即构]]
- [[Wasm自适应熔断]]
- [[Envoy Sidecar性能]]
- [[Go-Zero微服务框架]]
- [[gRPC概述]]
- [[gRPC架构与RPC类型]]