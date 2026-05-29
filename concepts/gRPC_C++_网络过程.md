# gRPC C++ 网络过程

## C++ gRPC 网络过程

// 图画详解 C++ grpc 网络过程
// ---------------------------------------------
// C++ gRPC 网络处理全流程图&核心原理梳理（简明版）
//
// 服务启动流程（以同步API为例，类似于“阻塞”网络模型）
//
//  ┌───────────────┐         ┌───────────────────────────┐
//  │ main()入口   │         │ 1. 注册服务对象到ServerBuilder │
//  │              │───────▶│ 2. 配置Server监听端口         │
//  │              │         │ 3. builder.BuildAndStart()   │
//  └───────────────┘         │  ┌───────────────────────┐   │
//                            │  │   ◀   监听socket绑定  │   │
//                            │  └───────────────────────┘   │
//                            │  4. 阻塞等客户端接入：       │
//                            │     server->Wait()           │
//                            └───────────────────────────────┘
//
//       		　　┌─────────────┐                             
//               │ 客户端发起连接 │
//               └─────┬───────┘
//                     │ TCP三次握手
//                     ▼
//               ┌─────────────┐
//               │ 新连接到服务 │
//               └─────┬───────┘
//                     │ 服务收到Socket accept
//                     ▼
//  ┌────────────────────────────────────────────────────┐
//  │  Server内部流程（同步模式）：                       │
//  │  1. 每个rpc调用都由线程池分配工作线程              │
//  │  2. 工作线程block在request的WaitFor*上             │
//  │  3. 收到请求数据后反序列化为protobuf对象           │
//  │  4. 调用你注册的回调函数(Do业务逻辑/响应)         │
//  │  5. 业务回调结束后数据被序列化发送给客户端         │
//  │  6. 线程回到等待状态(或每次spawn新线程)            │
//  └────────────────────────────────────────────────────┘
//
// 【核心网络细节】
// - 底层以 epoll/kqueue 实现多路复用。
// - 新连接和每路RPC请求有对应的CompletionQueue事件。
// - 数据收发默认用protobuf编码，内容自动处理。
// - SSL/TSL等支持在Builder阶段配置。
// - async接口则由开发者显式轮询CompletionQueue获取事件(更灵活高效!)。

// -------
//
// 图示举例（同步服务端伪代码）:
//
// ┌───────────────────────────┐
// │ class MyServiceImpl : ... │
// └────┬──────────────┬──────┘
//      │              │
//   override      override
//   Foo()         Bar()
//      ↓              ↓
// ┌──────────────┐   ┌──────────────┐
// │收到rpc请求→│  │收到rpc请求→ │
// │ 业务逻辑处理 │   │业务逻辑处理│
// │返回响应      │   │返回响应   │
// └──────────────┘   └──────────────┘
//            │              │
//           返回序列化后的网络数据给客户端
//
// ─────常见网络处理路径─────
// 1. 客户端 new stub
// 2. stub.Foo(request) → 自动构造底层Call
// 3. Codec序列化编码 → 网络层写出
// 4. 服务端 epoll/kqueue监听 → 收到消息
// 5. server反序列化 → 调用impl::Foo()
// 6. 得到response再编码→写回
// 7. 客户端收到response，反序列化
//
// --- async模式: 
// 1. 多个CompletionQueue异步事件
// 2. 由主循环显式去取请求，业务线程池处理
// 3. 支持高并发和自定义调度
//
// ---------------------------------------------
//
// 【参考伪代码（同步服务端）】
// 
// int main() {
//   MyServiceImpl service;
//   ServerBuilder builder;
//   builder.AddListeningPort("0.0.0.0:50051", grpc::InsecureServerCredentials());
//   builder.RegisterService(&service);
//   std::unique_ptr<Server> server = builder.BuildAndStart();
//   server->Wait();
// }
//
// // 业务继承生成service，重写每个rpc方法(自动回调)
// class MyServiceImpl final : public MyService::Service {
//   Status Foo(ServerContext* ctx, const Request* req, Response* rsp) override {
//      ...业务处理...
//      return Status::OK;
//   }
// };
//
// 【解析】
// - ServerBuilder配置监听端口、注册service，幕后完成了socket监听、protobuf注册、线程池分发等。
// - “同步API”会阻塞主线程，适合简单RPC。
// - “异步API”由开发者自行管理高并发：CompletionQueue+tag轮询，效率更高，适用于重并发网络服务和高吞吐负载。
// 
// ---------------------------------------------
//
// 小结：
// gRPC C++ 网络流程 = 
//  (1) 监听端口（epoll多路复用）
//  (2) 客户端请求自动编码网络传输
//  (3) 反序列化protobuf，回调你注册的服务
//  (4) 对应线程池并发调度
//  (5) 自动编码响应，网络返回
//  (6) 支持安全(SSL)和负载均衡扩展
//
// （异步API还能实现更灵活高并发处理模型，主流生产大规模服务建议使用）
//
// 参考官方网络流程图：[gRPC C++ Architecture - 官网文档](https://grpc.io/docs/what-is-grpc/core-concepts/)

[src: raw/ingested/2技术/网络协议/网络库-grpc.md]