# Go SDK 设计

> 面向 K8s + Service Mesh 环境的微服务基础 SDK，统一应用生命周期、通信链路、观测日志和配置管理。

---

## 一、设计背景

ZG SDK 面向公司内部微服务体系，运行在 **Kubernetes + Service Mesh（Sidecar）** 环境，解决以下重复建设问题：

| 问题 | SDK 方案 |
|------|---------|
| 启动依赖顺序混乱（应用已监听但依赖未就绪） | 统一 Initializer 编排 |
| 健康检查与 K8s 探针对齐 | 内置 HTTP /healthy 端点 |
| gRPC/HTTP 元数据透传（trace id、租户、灰度标签） | 统一拦截器链 |
| 指标/日志/链路追踪分散实现 | 统一 Middleware |
| 本地/测试/生产配置不一致 | 统一配置加载（zego-micro.yaml） |

---

## 二、模块架构

```
app（应用生命周期）
  ├── WithInitializers   → 顺序初始化依赖（DB、缓存、RPC 客户端）
  ├── WithHealthChecks   → 健康检查（K8s liveness/readiness）
  └── WithShutdownHooks  → 优雅退出（SIGINT/SIGTERM）

transport（通信链路）
  ├── gRPC 客户端/服务端  → 服务发现、负载均衡、元数据注入
  └── HTTP 客户端/服务端  → 同上，兼容 REST 协议

log（观测日志）
  ├── 结构化日志
  ├── 链路追踪集成
  └── 指标采集

config（配置管理）
  ├── zego-micro.yaml    → 中心化配置
  └── 本地文件 + 环境变量覆盖
```

---

## 三、app — 应用生命周期

### 3.1 核心结构

```go
type app struct {
    startTimeout  time.Duration
    initializers  []func()
    healthChecks  []func() error
    shutdownHooks []func(os.Signal)
}
```

### 3.2 启动流程

```go
func (a *app) Run() {
    // 1. 带超时的顺序初始化
    go func() { a.initialize(); close(done) }()
    select {
    case <-timeout.Done(): panic("start timeout")
    case <-done:          // 初始化完成
    }
    // 2. 启动健康检查 HTTP 服务
    go a.runHealthCheckServer()
    // 3. 等待退出信号
    a.waitAndHandleShutdown()
}
```

**关键设计**：
- 初始化超时默认 **7s**（与 K8s 首次探针延迟对齐）
- 初始化失败直接 panic → K8s 自动重启 Pod
- 健康检查端口默认 **9999**，可通过配置覆盖

### 3.3 健康检查

```go
func (a *app) runHealthCheckServer() error {
    return http.ListenAndServe(":9999", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        for _, h := range a.healthChecks {
            if err := h(); err != nil {
                w.WriteHeader(http.StatusServiceUnavailable)
                return
            }
        }
        w.WriteHeader(http.StatusOK)
    }))
}
```

**场景**：下游依赖（MySQL、Redis、关键 RPC）异常时主动返回 503，让 Service/Istio 及时摘流。

### 3.4 优雅退出

```go
func (a *app) waitAndHandleShutdown() {
    signal.Notify(termChan, syscall.SIGINT, syscall.SIGTERM)
    s := <-termChan
    for _, h := range a.shutdownHooks {
        h(s)  // 串行执行：摘流 → flush 日志 → 关连接
    }
}
```

**与 K8s 配合**：
```
Pod 滚动更新
  → K8s 发 SIGTERM
  → SDK 执行 shutdown hooks（deregister、flush、close）
  → 进程退出
  → K8s 启动新 Pod
```

---

## 四、transport — 通信链路

### 4.1 设计要点

| 能力 | 说明 |
|------|------|
| 服务发现 | 通过 Kubernetes Service DNS 解析 + Sidecar 转发 |
| 负载均衡 | 由 Envoy 完成（HTTP2 连接池 + 主动健康检查） |
| 元数据透传 | gRPC metadata / HTTP header 透传 trace id、租户、灰度标签 |
| 统一超时/重试 | SDK 侧配置默认值（超时 3s，重试 1 次） |
| 协议 | 优先 gRPC（性能 + 双向流 + 强类型），降级 HTTP |

### 4.2 拦截器链

```go
// gRPC 客户端拦截器
grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor(
    // trace 注入
    // 灰度标签透传
    // 指标采集
))
```

---

## 五、log & config — 日志与配置

### 5.1 结构化日志

```go
log.WithField("service", "user-service").
    WithField("trace_id", traceID).
    Info("request handled")
```

### 5.2 配置加载

```
优先级：zego-micro.yaml → 环境变量 → 命令行参数
```

---

## 六、SDK 与原生 Go 的关系

| 场景 | 使用 SDK | 使用标准库 |
|------|---------|-----------|
| 微服务间调用（网格内东西向流量） | ✓ | ✗ |
| 访问公网/集群外服务 | ✗ | ✓ (标准库) |
| 本地开发测试 | ✗ | ✓ |

SDK 不替代标准库，只在 **网格内流量路径** 上做增强（元数据注入、trace、指标）。
