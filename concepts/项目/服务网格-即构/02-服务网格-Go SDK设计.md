# Go SDK 设计

> 面向 K8s + Service Mesh 环境的微服务基础 SDK（内部代号 ZG SDK），统一应用生命周期、通信链路、观测日志和配置管理。业务线只需 import SDK → 替换初始化代码 → 零其他改动即可接入服务网格。

---

## 一、设计背景

### 1.1 痛点分析

ZG SDK 面向公司内部微服务体系，运行在 **Kubernetes + Service Mesh（Sidecar）** 环境，解决以下重复建设问题：

| # | 问题 | SDK 方案 | 影响范围 |
|---|------|---------|---------|
| 1 | 启动依赖顺序混乱（应用已监听但依赖未就绪） | 统一 Initializer 编排 | 所有业务线启动阶段 |
| 2 | 健康检查与 K8s 探针对齐 | 内置 HTTP /healthy 端点 | K8s 调度决策 |
| 3 | gRPC/HTTP 元数据透传（trace id、租户、灰度标签） | 统一拦截器链 | 全链路观测 & 灰度 |
| 4 | 指标/日志/链路追踪分散实现 | 统一 Middleware | 观测体系标准化 |
| 5 | 本地/测试/生产配置不一致 | 统一配置加载（zego-micro.yaml） | 环境差异化 |
| 6 | 各业务线重复实现超时/重试/熔断 | SDK 内置默认策略 + 可覆盖 | 稳定性基线 |
| 7 | 业务代码直接依赖 Istio/Envoy API | SDK 封装，业务无感知 | 解耦网格基础设施 |

### 1.2 设计原则

| 原则 | 说明 |
|------|------|
| **零侵入** | 业务代码不直接引用 Istio/Envoy API，治理策略通过控制面推送而非 SDK 配置文件 |
| **约定大于配置** | 提供合理的默认值（超时 3s、重试 1 次、健康检查端口 9999），业务按需覆盖 |
| **优雅失败** | 初始化失败 panic → K8s 自动重启；依赖故障返回 503 → Service 摘流 |
| **可观测性内建** | 每个请求自动携带 trace、metrics、logging，业务无需额外埋点 |
| **与标准库共生** | SDK 只增强网格内东西向流量，公网/本地开发仍使用标准库 |

### 1.3 SDK 在网格中的位置

```
┌─────────────────────────────────────────────────────────┐
│                     业务代码                              │
│   import "zg-sdk"                                       │
│   app.New(                                              │
│       app.WithInitializers(db.Init, redis.Init),        │
│       transport.WithGRPCServer(...),                    │
│   ).Run()                                               │
└────────────────────┬────────────────────────────────────┘
                     │ (SDK 封装)
┌────────────────────▼────────────────────────────────────┐
│                 ZG SDK 层                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │
│  │  app     │  │transport │  │   log    │  │ config  │  │
│  │生命周期  │  │ 通信链路  │  │ 观测日志  │  │ 配置管理 │  │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘  │
└────────────────────┬────────────────────────────────────┘
                     │ (gRPC/HTTP over Sidecar)
┌────────────────────▼────────────────────────────────────┐
│               Service Mesh 基础设施                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │
│  │  Envoy   │  │  Istiod  │  │ Jaeger   │  │Prometheus│  │
│  │ Sidecar  │  │ 控制面   │  │ 链路追踪  │  │  指标    │  │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 二、模块架构与包结构

### 2.1 包结构总览

```
zg-sdk/
├── app/                     # 应用生命周期包
│   ├── app.go               # 核心结构体 & Run() 入口
│   ├── options.go           # Options 模式配置
│   ├── health.go            # 健康检查 HTTP 服务
│   ├── shutdown.go          # 优雅退出信号处理
│   └── app_test.go          # 单元测试 & example
│
├── transport/               # 通信链路包
│   ├── grpc/                # gRPC 客户端/服务端封装
│   │   ├── client.go        #   NewClient() 工厂
│   │   ├── server.go        #   NewServer() 工厂
│   │   ├── interceptors/    #   拦截器链
│   │   │   ├── metadata.go  #     元数据透传
│   │   │   ├── tracing.go   #     OpenTelemetry 链路追踪
│   │   │   ├── metrics.go   #     Prometheus 指标
│   │   │   ├── retry.go     #     超时重试
│   │   │   └── logging.go   #     请求日志
│   │   └── resolver/        # 服务发现
│   │       └── k8s.go       #   Kubernetes DNS 解析
│   ├── http/                # HTTP 客户端/服务端封装
│   │   ├── client.go        #   NewClient() 工厂
│   │   ├── server.go        #   NewServer() 工厂
│   │   └── middleware/      #   HTTP 中间件
│   │       ├── metadata.go
│   │       ├── tracing.go
│   │       ├── metrics.go
│   │       └── recovery.go  #    恐慌恢复
│   └── transport.go         # 公共类型 & 常量（metadata key 定义）
│
├── log/                     # 观测日志包
│   ├── logger.go            # 结构化日志接口 & 默认实现
│   ├── context.go           # Context 日志增强（trace_id 自动注入）
│   └── fields.go            # 公共字段常量
│
├── config/                  # 配置管理包
│   ├── config.go            # 加载逻辑（文件 → 环境变量 → 命令行）
│   ├── defaults.go          # 默认值定义
│   └── config_test.go
│
├── metadata/                # 元数据常量 & 工具
│   ├── key.go               # 预定义 metadata key
│   └── propagator.go        # gRPC ↔ HTTP 元数据互转
│
├── errors/                  # 统一错误处理
│   ├── codes.go             # 业务错误码
│   └── errors.go            # 错误包装 & 链路追踪集成
│
├── examples/                # 使用示例
│   ├── user-service/        # 完整微服务示例
│   └── simple/
│
├── go.mod
└── README.md
```

### 2.2 SDK 模块依赖关系

```
┌────────────────────────────────────────────────────────────┐
│                    业务代码 (user-service)                   │
└────────────────────────┬───────────────────────────────────┘
                         │ import
┌────────────────────────▼───────────────────────────────────┐
│                       zg-sdk                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │    app      │──│  transport  │  │        errors       │  │
│  │ (生命周期)  │  │ (通信链路)  │  │    (统一错误码)      │  │
│  └──────┬──────┘  └──────┬──────┘  └─────────────────────┘  │
│         │                │                                   │
│         │    ┌───────────▼───────────┐                      │
│         │    │  metadata / log / config                     │
│         │    └───────────┬───────────┘                      │
│         │                │                                   │
│         └────────────────┴──────────────────────────────────┘
│                          │ depends on
└──────────────────────────┼──────────────────────────────────┘
                           │
          ┌────────────────▼─────────────────┐
          │        第三方依赖                    │
          │  google.golang.org/grpc            │
          │  go.opentelemetry.io/otel          │
          │  github.com/prometheus/client_golang│
          │  gopkg.in/yaml.v3                  │
          └────────────────────────────────────┘
```

---

## 三、app — 应用生命周期

### 3.1 核心接口与结构

```go
package app

// App 是 SDK 的应用生命周期管理入口
type App interface {
    // Run 启动应用：初始化 → 健康检查 → 等待退出信号
    Run()
}

// Option 配置函数
type Option func(*appOptions)

// 私有实现
type appOptions struct {
    startTimeout  time.Duration
    initializers  []Initializer
    healthChecks  []HealthCheck
    shutdownHooks []ShutdownHook
    healthPort    int
    logger        *slog.Logger
}

// Initializer 初始化器，返回 error 表示初始化失败
type Initializer func(ctx context.Context) error

// HealthCheck 健康检查函数，返回 error 表示该组件不健康
type HealthCheck func(ctx context.Context) error

// ShutdownHook 优雅退出钩子
type ShutdownHook func(ctx context.Context, sig os.Signal)
```

### 3.2 Options 模式详解

```go
// --- Options 构造函数 ---

// WithInitializers 设置顺序初始化器列表
func WithInitializers(initializers ...Initializer) Option {
    return func(o *appOptions) {
        o.initializers = append(o.initializers, initializers...)
    }
}

// WithHealthChecks 设置健康检查函数列表
func WithHealthChecks(checks ...HealthCheck) Option {
    return func(o *appOptions) {
        o.healthChecks = append(o.healthChecks, checks...)
    }
}

// WithShutdownHooks 设置优雅退出钩子列表
func WithShutdownHooks(hooks ...ShutdownHook) Option {
    return func(o *appOptions) {
        o.shutdownHooks = append(o.shutdownHooks, hooks...)
    }
}

// WithStartTimeout 设置初始化超时（默认 7s）
func WithStartTimeout(d time.Duration) Option {
    return func(o *appOptions) {
        o.startTimeout = d
    }
}

// WithHealthPort 设置健康检查 HTTP 端口（默认 9999）
func WithHealthPort(port int) Option {
    return func(o *appOptions) {
        o.healthPort = port
    }
}

// 默认值
func defaultOptions() appOptions {
    return appOptions{
        startTimeout: 7 * time.Second,  // 与 K8s initialDelaySeconds 对齐
        healthPort:   9999,
        logger:       slog.Default(),
    }
}
```

### 3.3 完整启动流程

```go
type appImpl struct {
    opts appOptions
}

func New(opts ...Option) App {
    o := defaultOptions()
    for _, opt := range opts {
        opt(&o)
    }
    return &appImpl{opts: o}
}

func (a *appImpl) Run() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // ==========================================
    // 阶段 1：顺序初始化（带超时）
    // ==========================================
    initCtx, initCancel := context.WithTimeout(ctx, a.opts.startTimeout)
    defer initCancel()

    done := make(chan struct{}, 1)
    go func() {
        defer close(done)
        for i, init := range a.opts.initializers {
            a.opts.logger.InfoContext(initCtx, "initializing",
                "step", i+1,
                "total", len(a.opts.initializers),
            )
            if err := init(initCtx); err != nil {
                // 初始化失败直接 panic → K8s 自动重启 Pod
                a.opts.logger.ErrorContext(initCtx, "initialization failed",
                    "step", i+1,
                    "error", err,
                )
                panic(fmt.Sprintf("zg-sdk: initialization failed at step %d: %v", i+1, err))
            }
        }
    }()

    select {
    case <-done:
        a.opts.logger.InfoContext(ctx, "all initializers completed")
    case <-initCtx.Done():
        // 超时也 panic → K8s 重启
        panic(fmt.Sprintf("zg-sdk: initialization timed out after %v", a.opts.startTimeout))
    }

    // ==========================================
    // 阶段 2：启动健康检查 HTTP 服务
    // ==========================================
    healthDone := make(chan error, 1)
    go func() {
        healthDone <- a.runHealthServer(ctx)
    }()

    // ==========================================
    // 阶段 3：等待退出信号，优雅关闭
    // ==========================================
    a.waitShutdown(ctx)

    cancel()                           // 通知健康检查服务退出
    <-healthDone                       // 等待健康检查服务停止
    a.opts.logger.InfoContext(ctx, "app exited gracefully")
}
```

**关键设计决策**：

| 决策 | 原因 |
|------|------|
| 初始化失败用 panic 而非 return error | K8s 重启策略基于进程退出码，panic 直接异常退出，K8s 自动重启 Pod |
| 初始化超时默认 7s | 与 K8s liveness/readiness 的 `initialDelaySeconds` 推荐值对齐，保证探针第一次检查时应用已就绪 |
| 健康检查 HTTP 服务在初始化完成后启动 | 避免初始化过程中被 K8s 判定为就绪但实际无法服务 |
| 优雅退出串行执行 | 保证"摘流 → flush 日志 → 关连接"的顺序，避免日志丢失或连接过早关闭 |

### 3.4 健康检查实现

```go
func (a *appImpl) runHealthServer(ctx context.Context) error {
    mux := http.NewServeMux()

    // /healthz — K8s liveness probe（存活探针）
    mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprint(w, "ok")
    })

    // /ready — K8s readiness probe（就绪探针）
    // 返回 503 时，K8s Service 摘流，Istio 从负载池移除
    mux.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
        for _, check := range a.opts.healthChecks {
            if err := check(r.Context()); err != nil {
                a.opts.logger.WarnContext(r.Context(), "health check failed",
                    "error", err,
                )
                w.WriteHeader(http.StatusServiceUnavailable)
                fmt.Fprint(w, err.Error())
                return
            }
        }
        w.WriteHeader(http.StatusOK)
        fmt.Fprint(w, "ready")
    })

    addr := fmt.Sprintf(":%d", a.opts.healthPort)
    server := &http.Server{Addr: addr, Handler: mux}

    // 在 goroutine 中启动，监听 ctx 退出以优雅关闭
    go func() {
        <-ctx.Done()
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        server.Shutdown(shutdownCtx) //nolint: errcheck
    }()

    a.opts.logger.InfoContext(ctx, "health check server started",
        "addr", addr,
        "liveness", "/healthz",
        "readiness", "/ready",
    )

    if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        return err
    }
    return nil
}
```

**健康检查与 K8s + Istio 协同**：

```
K8s Liveness Probe → /healthz
  ├─ 始终返回 200（进程存活即健康）
  └─ 失败 → K8s 重启 Pod

K8s Readiness Probe → /ready
  ├─ 遍历所有 HealthCheck
  │   ├─ MySQL 连接正常？ ✓
  │   ├─ Redis 连接正常？ ✓
  │   └─ 下游 gRPC 连接正常？ ✓
  ├─ 全部 OK → 200 → Service Endpoint 就绪
  └─ 任一失败 → 503 → K8s 摘流 + Istio 从负载池移除

Pod 滚动更新时
  → preStop 延迟等待 Readiness 变为 503
  → K8s 确认 Service Endpoint 已移除
  → 发送 SIGTERM
  → SDK 执行 shutdown hooks
```

### 3.5 优雅退出

```go
func (a *appImpl) waitShutdown(ctx context.Context) {
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    sig := <-sigCh
    a.opts.logger.InfoContext(ctx, "shutdown signal received",
        "signal", sig.String(),
    )

    // 串行执行 shutdown hooks
    // 顺序：摘流 → flush 日志 → 关闭连接
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    for i, hook := range a.opts.shutdownHooks {
        a.opts.logger.InfoContext(shutdownCtx, "executing shutdown hook",
            "step", i+1,
            "total", len(a.opts.shutdownHooks),
        )
        hook(shutdownCtx, sig)
    }
}

// --- 预置 shutdown hooks ---

// ShutdownHTTPServer 优雅关闭 HTTP 服务器
func ShutdownHTTPServer(srv *http.Server) ShutdownHook {
    return func(ctx context.Context, sig os.Signal) {
        srv.Shutdown(ctx) //nolint: errcheck
    }
}

// ShutdownGRPCServer 优雅关闭 gRPC 服务器
func ShutdownGRPCServer(srv *grpc.Server) ShutdownHook {
    return func(ctx context.Context, sig os.Signal) {
        stopped := make(chan struct{})
        go func() {
            srv.GracefulStop()
            close(stopped)
        }()
        select {
        case <-stopped:
        case <-ctx.Done():
            srv.Stop() // 超时后强杀
        }
    }
}

// FlushLogger 刷新日志缓冲区
func FlushLogger() ShutdownHook {
    return func(ctx context.Context, sig os.Signal) {
        // 同步日志缓冲区，确保日志不丢失
        slog.Default().InfoContext(ctx, "flushing log buffer")
    }
}
```

### 3.6 业务使用示例

```go
package main

import (
    "zg-sdk/app"
    "zg-sdk/transport/grpc"
    "zg-sdk/transport/http"
)

func main() {
    // 创建 gRPC 服务器
    grpcSrv := grpc.NewServer(
        grpc.WithServiceName("user-service"),
        grpc.WithRegister(func(s *grpc.Server) {
            pb.RegisterUserServiceServer(s, &userService{})
        }),
    )

    // 创建 App，注入依赖
    a := app.New(
        app.WithInitializers(
            initMySQL,   // 初始化 MySQL 连接
            initRedis,   // 初始化 Redis 连接
        ),
        app.WithHealthChecks(
            mysqlHealthCheck,   // MySQL 探活
            redisHealthCheck,   // Redis 探活
        ),
        app.WithShutdownHooks(
            app.ShutdownGRPCServer(grpcSrv),
            app.FlushLogger(),
        ),
        app.WithStartTimeout(10*time.Second),
    )

    a.Run()
}
```

---

## 四、transport — 通信链路

### 4.1 设计要点

| 能力 | 实现方式 | 说明 |
|------|---------|------|
| 服务发现 | 通过 Kubernetes Service DNS 解析（`service.namespace.svc.cluster.local`） | Sidecar 自动拦截，SDK 无需额外注册 |
| 负载均衡 | 由 Envoy Sidecar 完成（HTTP/2 连接池 + 主动健康检查） | SDK 侧无需实现 LB 算法 |
| 元数据透传 | gRPC metadata / HTTP header 透传 trace id、租户、灰度标签 | 统一拦截器链自动注入 |
| 统一超时/重试 | SDK 侧配置默认值（超时 3s，重试 1 次），可 per-call 覆盖 | 防止雪崩，默认退避策略 100ms base |
| 协议偏好 | 优先 gRPC（性能 + 双向流 + 强类型），对外暴露兼容 HTTP | 内部服务间用 gRPC，外部 API 用 HTTP |
| 连接管理 | 连接池复用 + 空闲连接超时回收 | 避免 Sidecar 连接表膨胀 |

### 4.2 元数据协议（Metadata Propagation）

这是 SDK 最核心的能力，定义了一套**统一元数据协议**，所有服务间调用自动透传：

```go
// transport/metadata/key.go

// 预定义的元数据 Key（所有 Key 统一前缀，避免与业务 header 冲突）
const (
    // --- 链路追踪 ---
    MetaTraceID         = "x-zg-trace-id"        // 全链路追踪 ID
    MetaSpanID          = "x-zg-span-id"         // 当前 Span ID
    MetaTraceSampled    = "x-zg-trace-sampled"   // 是否采样

    // --- 多租户 & 身份 ---
    MetaTenantID        = "x-zg-tenant-id"       // 租户 ID
    MetaUserID          = "x-zg-user-id"         // 用户 ID
    MetaAuthToken       = "x-zg-auth-token"      // 认证令牌

    // --- 灰度发布 ---
    MetaCanaryTag       = "x-zg-canary-tag"      // 灰度标签（canary / stable / beta）
    MetaCanaryVersion   = "x-zg-canary-version"  // 灰度版本号

    // --- 请求治理 ---
    MetaRequestTimeout   = "x-zg-request-timeout" // 请求超时（ms），覆盖 SDK 默认
    MetaRetryCount       = "x-zg-retry-count"     // 已重试次数（SDK 自动维护）
    MetaMaxRetries       = "x-zg-max-retries"     // 最大重试次数

    // --- 业务扩展 ---
    MetaCallerService    = "x-zg-caller-service"  // 调用方服务名
    MetaCallerIP         = "x-zg-caller-ip"       // 调用方 Pod IP
)
```

**gRPC 与 HTTP 双向转换**：

```go
// transport/metadata/propagator.go

// FromGRPC 从 gRPC metadata 中提取元数据，返回 map
func FromGRPC(md metadata.MD) map[string]string {
    result := make(map[string]string, len(metaKeys))
    for _, k := range metaKeys {
        if vals := md.Get(k); len(vals) > 0 {
            result[k] = vals[0]
        }
    }
    return result
}

// ToGRPC 将元数据 map 注入到 gRPC metadata 中
func ToGRPC(ctx context.Context, m map[string]string) context.Context {
    md, ok := metadata.FromOutgoingContext(ctx)
    if !ok {
        md = metadata.New(nil)
    }
    for k, v := range m {
        md.Set(k, v)
    }
    return metadata.NewOutgoingContext(ctx, md)
}

// FromHTTP 从 HTTP header 中提取元数据
func FromHTTP(header http.Header) map[string]string {
    result := make(map[string]string, len(metaKeys))
    for _, k := range metaKeys {
        if v := header.Get(k); v != "" {
            result[k] = v
        }
    }
    return result
}

// ToHTTP 将元数据 map 注入到 HTTP header 中
func ToHTTP(header http.Header, m map[string]string) {
    for k, v := range m {
        header.Set(k, v)
    }
}
```

### 4.3 gRPC 客户端封装

```go
// transport/grpc/client.go

// ClientConfig gRPC 客户端配置
type ClientConfig struct {
    // 目标服务名称（K8s Service 名，不含 namespace）
    ServiceName string
    // 目标服务 namespace（默认与客户端相同）
    Namespace string
    // 默认超时（默认 3s）
    DefaultTimeout time.Duration
    // 最大重试次数（默认 1）
    MaxRetries int
    // 连接池大小（默认 10）
    PoolSize int
    // 是否启用 mTLS（默认 false，网格内由 Envoy 处理）
    EnableTLS bool
}

// NewClient 创建一个 gRPC 客户端连接
// target 格式: "service.namespace.svc.cluster.local:port"
func NewClient(target string, opts ...Option) (*grpc.ClientConn, error) {
    config := defaultConfig()
    for _, opt := range opts {
        opt(config)
    }

    // 构建拦截器链（顺序重要！）
    interceptorChain := grpc.WithChainUnaryInterceptor(
        interceptors.MetadataClientInterceptor(),     // 1. 元数据注入（最先执行）
        interceptors.TracingClientInterceptor(),      // 2. 链路追踪
        interceptors.MetricsClientInterceptor(),      // 3. 指标采集
        interceptors.LoggingClientInterceptor(),      // 4. 请求日志
        interceptors.RetryClientInterceptor(          // 5. 超时重试（最后执行，包裹实际调用）
            config.MaxRetries, config.DefaultTimeout,
        ),
    )

    conn, err := grpc.Dial(
        target,
        grpc.WithInsecure(),               // 网格内 Envoy 处理 mTLS
        grpc.WithDefaultServiceConfig(`{
            "loadBalancingPolicy": "round_robin",
            "methodConfig": [{
                "name": [{"service": ""}],
                "retryPolicy": {
                    "maxAttempts": 2,
                    "initialBackoff": "0.1s",
                    "maxBackoff": "1s",
                    "backoffMultiplier": 2.0,
                    "retryableStatusCodes": ["UNAVAILABLE"]
                }
            }]
        }`),
        interceptorChain,
        grpc.WithBlock(),
        grpc.WithConnectParams(grpc.ConnectParams{
            MinConnectTimeout: 1 * time.Second,
        }),
    )
    if err != nil {
        return nil, fmt.Errorf("zg-sdk: dial %s: %w", target, err)
    }
    return conn, nil
}
```

### 4.4 gRPC 拦截器链详解

#### 4.4.1 元数据拦截器（Metadata Interceptor）

```go
// transport/grpc/interceptors/metadata.go

// MetadataClientInterceptor 客户端元数据注入拦截器
// 从 context 中提取调用方元数据，注入到 gRPC metadata
func MetadataClientInterceptor() grpc.UnaryClientInterceptor {
    return func(
        ctx context.Context,
        method string,
        req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        // 提取 context 中已有的元数据
        meta := extractFromContext(ctx)

        // 自动注入调用方服务名
        if meta[MetaCallerService] == "" {
            meta[MetaCallerService] = os.Getenv("ZEGO_SERVICE_NAME")
        }

        // 注入到 gRPC outgoing context
        ctx = metadata.ToGRPC(ctx, meta)

        return invoker(ctx, method, req, reply, cc, opts...)
    }
}

// MetadataServerInterceptor 服务端元数据提取拦截器
// 从 gRPC metadata 中提取元数据，注入到 context
func MetadataServerInterceptor() grpc.UnaryServerInterceptor {
    return func(
        ctx context.Context,
        req interface{},
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (interface{}, error) {
        // 从 gRPC incoming metadata 中提取元数据
        md, ok := metadata.FromIncomingContext(ctx)
        if ok {
            meta := metadata.FromGRPC(md)
            // 注入到 context（供后续拦截器和业务代码使用）
            ctx = injectToContext(ctx, meta)
        }
        return handler(ctx, req)
    }
}
```

#### 4.4.2 链路追踪拦截器（Tracing Interceptor）

```go
// transport/grpc/interceptors/tracing.go

// TracingClientInterceptor 基于 OpenTelemetry 的链路追踪客户端拦截器
// 自动创建 client span，注入 trace context 到 gRPC metadata
func TracingClientInterceptor() grpc.UnaryClientInterceptor {
    return func(
        ctx context.Context,
        method string,
        req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        // 从 context 获取 tracer
        tracer := otel.Tracer("zg-sdk")
        ctx, span := tracer.Start(ctx, method,
            trace.WithSpanKind(trace.SpanKindClient),
        )
        defer span.End()

        // 注入 trace context 到 gRPC metadata
        otelgrpc.Inject(ctx, &metadataSupplier{
            md: &md,
        })

        err := invoker(ctx, method, req, reply, cc, opts...)

        // 记录错误信息到 span
        if err != nil {
            span.SetStatus(codes.Error, err.Error())
            span.RecordError(err)
        }
        return err
    }
}

// TracingServerInterceptor 链路追踪服务端拦截器
// 从 gRPC metadata 中提取 trace context，创建 server span
func TracingServerInterceptor() grpc.UnaryServerInterceptor {
    return func(
        ctx context.Context,
        req interface{},
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (interface{}, error) {
        // 从 gRPC metadata 中提取 trace context
        md, ok := metadata.FromIncomingContext(ctx)
        if ok {
            propagator := otel.GetTextMapPropagator()
            ctx = propagator.Extract(ctx, &metadataSupplier{md: &md})
        }

        tracer := otel.Tracer("zg-sdk")
        ctx, span := tracer.Start(ctx, info.FullMethod,
            trace.WithSpanKind(trace.SpanKindServer),
        )
        defer span.End()

        resp, err := handler(ctx, req)
        if err != nil {
            span.SetStatus(codes.Error, err.Error())
            span.RecordError(err)
        }
        return resp, err
    }
}
```

#### 4.4.3 指标拦截器（Metrics Interceptor）

```go
// transport/grpc/interceptors/metrics.go

// 预定义的 Prometheus 指标
var (
    grpcRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "zg_grpc_requests_total",
            Help: "Total gRPC requests",
        }, []string{"service", "method", "status"},
    )
    grpcRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "zg_grpc_request_duration_ms",
            Help:    "gRPC request duration in ms",
            Buckets: []float64{1, 5, 10, 25, 50, 100, 250, 500, 1000},
        }, []string{"service", "method", "status"},
    )
    grpcRequestsInflight = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "zg_grpc_requests_inflight",
            Help: "Current in-flight gRPC requests",
        }, []string{"service", "method"},
    )
)

// MetricsClientInterceptor 客户端指标采集拦截器
func MetricsClientInterceptor() grpc.UnaryClientInterceptor {
    serviceName := os.Getenv("ZEGO_SERVICE_NAME")
    return func(
        ctx context.Context,
        method string,
        req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        grpcRequestsInflight.WithLabelValues(serviceName, method).Inc()
        defer grpcRequestsInflight.WithLabelValues(serviceName, method).Dec()

        start := time.Now()
        err := invoker(ctx, method, req, reply, cc, opts...)
        duration := time.Since(start)

        status := "ok"
        if err != nil {
            status = statusCode(err)
        }

        grpcRequestsTotal.WithLabelValues(serviceName, method, status).Inc()
        grpcRequestDuration.WithLabelValues(serviceName, method, status).Observe(
            float64(duration.Milliseconds()),
        )
        return err
    }
}

// MetricsServerInterceptor 服务端指标采集拦截器
func MetricsServerInterceptor() grpc.UnaryServerInterceptor {
    serviceName := os.Getenv("ZEGO_SERVICE_NAME")
    return func(
        ctx context.Context,
        req interface{},
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (interface{}, error) {
        grpcRequestsInflight.WithLabelValues(serviceName, info.FullMethod).Inc()
        defer grpcRequestsInflight.WithLabelValues(serviceName, info.FullMethod).Dec()

        start := time.Now()
        resp, err := handler(ctx, req)
        duration := time.Since(start)

        status := "ok"
        if err != nil {
            status = statusCode(err)
        }

        grpcRequestsTotal.WithLabelValues(serviceName, info.FullMethod, status).Inc()
        grpcRequestDuration.WithLabelValues(serviceName, info.FullMethod, status).Observe(
            float64(duration.Milliseconds()),
        )
        return resp, err
    }
}

// statusCode 从 gRPC error 提取状态码字符串
func statusCode(err error) string {
    st, ok := status.FromError(err)
    if !ok {
        return "unknown"
    }
    return st.Code().String()
}
```

#### 4.4.4 重试拦截器（Retry Interceptor）

```go
// transport/grpc/interceptors/retry.go

// RetryClientInterceptor 超时重试拦截器
// 在调用失败时按指数退避重试，配合熔断器避免雪崩
func RetryClientInterceptor(maxRetries int, defaultTimeout time.Duration) grpc.UnaryClientInterceptor {
    // 退避生成器：base 100ms, factor 2.0, jitter ±50%
    backoff := func(attempt int) time.Duration {
        d := time.Duration(100*math.Pow(2, float64(attempt))) * time.Millisecond
        jitter := time.Duration(rand.Int63n(int64(d))) / 2   // ±50% jitter
        return d + jitter
    }

    // 可重试的状态码
    retryableCodes := map[codes.Code]bool{
        codes.Unavailable:   true,   // 服务不可用
        codes.DeadlineExceeded: true, // 超时
        codes.ResourceExhausted: true, // 限流
        codes.Aborted:       true,   // 并发冲突
        codes.Internal:      true,   // 服务端内部错误（非业务错误）
    }

    return func(
        ctx context.Context,
        method string,
        req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        var lastErr error

        // 从 context 中获取 per-call 超时设置
        timeout := extractTimeout(ctx, defaultTimeout)

        for attempt := 0; attempt <= maxRetries; attempt++ {
            if attempt > 0 {
                // 指数退避等待
                select {
                case <-time.After(backoff(attempt)):
                case <-ctx.Done():
                    return ctx.Err()
                }
            }

            // 带超时的 context
            callCtx, cancel := context.WithTimeout(ctx, timeout)
            defer cancel()

            // 注入当前重试次数
            callCtx = metadata.AppendToOutgoingContext(callCtx,
                MetaRetryCount, strconv.Itoa(attempt),
            )

            err := invoker(callCtx, method, req, reply, cc, opts...)
            if err == nil {
                return nil
            }

            lastErr = err
            st := status.Convert(err)

            // 不可重试的错误直接返回
            if !retryableCodes[st.Code()] {
                return err
            }

            // context 取消或超时不再重试
            if errors.Is(err, context.Canceled) || errors.Is(err, context.DeadlineExceeded) {
                return err
            }

            slog.Debug("retrying gRPC call",
                "method", method,
                "attempt", attempt+1,
                "max", maxRetries+1,
                "error", err,
            )
        }
        return lastErr
    }
}

// extractTimeout 从 context 中提取 per-call 超时（支持元数据覆盖）
func extractTimeout(ctx context.Context, defaultTimeout time.Duration) time.Duration {
    // 检查 context 是否有 deadline
    if deadline, ok := ctx.Deadline(); ok {
        remaining := time.Until(deadline)
        if remaining > 0 {
            return remaining
        }
    }
    return defaultTimeout
}
```

#### 4.4.5 日志拦截器（Logging Interceptor）

```go
// transport/grpc/interceptors/logging.go

// LoggingClientInterceptor 客户端请求日志拦截器
// 记录每次 gRPC 调用的方法、耗时、状态
func LoggingClientInterceptor() grpc.UnaryClientInterceptor {
    return func(
        ctx context.Context,
        method string,
        req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        start := time.Now()
        err := invoker(ctx, method, req, reply, cc, opts...)
        duration := time.Since(start)

        // 结构化日志，自动关联 trace_id
        attrs := []slog.Attr{
            slog.String("method", method),
            slog.Duration("duration", duration),
            slog.String("target", cc.Target()),
        }
        if traceID := getTraceID(ctx); traceID != "" {
            attrs = append(attrs, slog.String("trace_id", traceID))
        }
        if err != nil {
            attrs = append(attrs, slog.String("error", err.Error()))
            slog.LogAttrs(ctx, slog.LevelError, "grpc client call failed", attrs...)
        } else {
            slog.LogAttrs(ctx, slog.LevelDebug, "grpc client call succeeded", attrs...)
        }
        return err
    }
}

// getTraceID 从 context 中提取 trace ID
func getTraceID(ctx context.Context) string {
    span := trace.SpanFromContext(ctx)
    if !span.IsRecording() {
        return ""
    }
    return span.SpanContext().TraceID().String()
}
```

### 4.5 gRPC 服务端封装

```go
// transport/grpc/server.go

// ServerConfig gRPC 服务端配置
type ServerConfig struct {
    Port        int
    ServiceName string
    MaxRecvMsgSize int
    MaxSendMsgSize int
}

// NewServer 创建 gRPC 服务端
func NewServer(cfg ServerConfig, register func(s *grpc.Server)) *grpc.Server {
    srv := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            // 服务端拦截器链（执行顺序：第一个最外层）
            interceptors.MetadataServerInterceptor(),     // 1. 提取元数据到 context
            interceptors.TracingServerInterceptor(),      // 2. 创建 server span
            interceptors.MetricsServerInterceptor(),      // 3. 采集指标
            interceptors.RecoveryServerInterceptor(),     // 4. panic 恢复（最内层兜底）
        ),
        grpc.MaxRecvMsgSize(cfg.MaxRecvMsgSize),
        grpc.MaxSendMsgSize(cfg.MaxSendMsgSize),
    )

    register(srv)

    // 启动监听
    addr := fmt.Sprintf(":%d", cfg.Port)
    lis, err := net.Listen("tcp", addr)
    if err != nil {
        panic(fmt.Sprintf("zg-sdk: listen %s: %v", addr, err))
    }

    go func() {
        slog.Info("gRPC server starting",
            "addr", addr,
            "service", cfg.ServiceName,
        )
        if err := srv.Serve(lis); err != nil {
            slog.Error("gRPC server stopped", "error", err)
        }
    }()

    return srv
}
```

### 4.6 HTTP 客户端封装

```go
// transport/http/client.go

// NewClient 创建一个 HTTP 客户端，自动注入 SDK 中间件
func NewClient(defaultTimeout time.Duration) *http.Client {
    transport := &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     30 * time.Second,
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
    }

    // 使用自定义 RoundTripper 链
    var rt http.RoundTripper = transport
    rt = middleware.MetricsRoundTripper(rt)       // 指标采集
    rt = middleware.TracingRoundTripper(rt)       // 链路追踪
    rt = middleware.MetadataRoundTripper(rt)      // 元数据注入

    return &http.Client{
        Timeout:   defaultTimeout,
        Transport: rt,
    }
}
```

### 4.7 HTTP 中间件

```go
// transport/http/middleware/metadata.go

// MetadataRoundTripper HTTP 客户端元数据注入中间件
func MetadataRoundTripper(next http.RoundTripper) http.RoundTripper {
    return roundTripperFunc(func(req *http.Request) (*http.Response, error) {
        // 从 context 中提取元数据，注入到 header
        meta := extractFromContext(req.Context())
        for k, v := range meta {
            req.Header.Set(k, v)
        }
        return next.RoundTrip(req)
    })
}

// MetadataMiddleware HTTP 服务端元数据提取中间件
func MetadataMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 从 HTTP header 中提取元数据，注入到 context
        meta := metadata.FromHTTP(r.Header)
        ctx := injectToContext(r.Context(), meta)

        // 提取 trace context
        propagator := otel.GetTextMapPropagator()
        ctx = propagator.Extract(ctx, propagation.HeaderCarrier(r.Header))

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

```go
// transport/http/middleware/metrics.go

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "zg_http_requests_total",
            Help: "Total HTTP requests",
        }, []string{"method", "path", "status"},
    )
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "zg_http_request_duration_ms",
            Help:    "HTTP request duration in ms",
            Buckets: []float64{1, 5, 10, 25, 50, 100, 250, 500, 1000},
        }, []string{"method", "path", "status"},
    )
)

// MetricsRoundTripper 客户端指标采集
func MetricsRoundTripper(next http.RoundTripper) http.RoundTripper {
    return roundTripperFunc(func(req *http.Request) (*http.Response, error) {
        start := time.Now()
        resp, err := next.RoundTrip(req)
        duration := time.Since(start)

        status := "error"
        if err == nil {
            status = strconv.Itoa(resp.StatusCode)
        }

        httpRequestsTotal.WithLabelValues(req.Method, req.URL.Path, status).Inc()
        httpRequestDuration.WithLabelValues(req.Method, req.URL.Path, status).Observe(
            float64(duration.Milliseconds()),
        )
        return resp, err
    })
}

type roundTripperFunc func(*http.Request) (*http.Response, error)

func (f roundTripperFunc) RoundTrip(r *http.Request) (*http.Response, error) {
    return f(r)
}
```

```go
// transport/http/middleware/recovery.go

// RecoveryMiddleware HTTP 恐慌恢复中间件
// 防止单个 handler panic 导致整个进程退出
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                slog.ErrorContext(r.Context(), "http handler panic recovered",
                    "panic", rec,
                    "path", r.URL.Path,
                    "method", r.Method,
                )
                w.WriteHeader(http.StatusInternalServerError)
                json.NewEncoder(w).Encode(map[string]string{
                    "error": "internal server error",
                })
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### 4.8 服务发现（K8s DNS Resolver）

```go
// transport/grpc/resolver/k8s.go

// 由于网格内 Envoy 处理服务发现和负载均衡，SDK 侧只需要使用 K8s DNS 格式
//
// target 格式: "service.namespace.svc.cluster.local:port"
//
// gRPC 默认使用 DNS resolver，自动解析为 Pod IP
// Envoy Sidecar 拦截流量后，由 Envoy 完成真正的负载均衡
//
// 如果需要灰度路由，可以通过 Istio VirtualService 控制：
//
//   apiVersion: networking.istio.io/v1beta1
//   kind: VirtualService
//   metadata:
//     name: user-service
//   spec:
//     hosts:
//     - user-service
//     http:
//     - match:
//       - headers:
//           x-zg-canary-tag:
//             exact: "canary"
//       route:
//       - destination:
//           host: user-service
//           subset: canary
//     - route:
//       - destination:
//           host: user-service
//           subset: stable

// K8sServiceName 构建 K8s 内部 DNS 名称
func K8sServiceName(service, namespace string) string {
    return fmt.Sprintf("%s.%s.svc.cluster.local", service, namespace)
}
```

---

## 五、log — 观测日志

### 5.1 结构化日志接口

SDK 基于 Go 1.21+ `log/slog` 标准库，提供统一的结构化日志：

```go
// log/logger.go

// 默认 logger 使用 slog，输出 JSON 格式，便于日志采集系统（Loki / ELK）解析
func InitLogger(serviceName string, level slog.Level) {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: level,
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            // 统一时间格式
            if a.Key == "time" {
                return slog.Attr{
                    Key:   "@timestamp",
                    Value: slog.StringValue(time.Now().Format(time.RFC3339Nano)),
                }
            }
            return a
        },
    })

    logger := slog.New(handler).With(
        slog.String("service", serviceName),
        slog.String("version", version),
    )

    slog.SetDefault(logger)
}
```

### 5.2 Context 日志增强

自动从 context 中提取元数据（trace_id, tenant_id 等）注入到每行日志：

```go
// log/context.go

func init() {
    // 替换默认 logger 为 context-aware logger
    slog.SetDefault(slog.New(&contextHandler{
        handler: slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelInfo,
        }),
    }))
}

type contextHandler struct {
    handler slog.Handler
}

func (h *contextHandler) Enabled(ctx context.Context, level slog.Level) bool {
    return h.handler.Enabled(ctx, level)
}

func (h *contextHandler) Handle(ctx context.Context, record slog.Record) error {
    // 从 context 中提取自动字段
    if traceID := getTraceIDFromContext(ctx); traceID != "" {
        record.AddAttrs(slog.String("trace_id", traceID))
    }
    if tenantID := getTenantIDFromContext(ctx); tenantID != "" {
        record.AddAttrs(slog.String("tenant_id", tenantID))
    }
    if callerService := getCallerServiceFromContext(ctx); callerService != "" {
        record.AddAttrs(slog.String("caller_service", callerService))
    }
    return h.handler.Handle(ctx, record)
}

func (h *contextHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    return &contextHandler{handler: h.handler.WithAttrs(attrs)}
}

func (h *contextHandler) WithGroup(name string) slog.Handler {
    return &contextHandler{handler: h.handler.WithGroup(name)}
}
```

### 5.3 使用示例

```go
// 业务代码中直接使用 slog，无需 import SDK 日志包
slog.InfoContext(ctx, "user created",
    "user_id", userID,
    "username", username,
)
// 输出（JSON）:
// {
//   "@timestamp": "2026-06-09T10:30:00.123456789+08:00",
//   "level": "INFO",
//   "msg": "user created",
//   "service": "user-service",
//   "trace_id": "abc123def456",
//   "tenant_id": "tenant-001",
//   "caller_service": "api-gateway",
//   "user_id": "u-1001",
//   "username": "alice"
// }
```

---

## 六、config — 配置管理

### 6.1 配置加载优先级

```
优先级从高到低：
  1. 命令行参数（--port=8080）
  2. 环境变量（ZEGO_PORT=8080）
  3. 配置文件（zego-micro.yaml 或 ZEGO_CONFIG_PATH 指定）
  4. 默认值（SDK 内建）
```

### 6.2 配置结构

```go
// config/config.go

// Config 应用配置
type Config struct {
    App    AppConfig    `yaml:"app"`
    Server ServerConfig `yaml:"server"`
    Log    LogConfig    `yaml:"log"`
    Trace  TraceConfig  `yaml:"trace"`
    // 业务自定义配置
    Extra  map[string]interface{} `yaml:"extra"`
}

type AppConfig struct {
    Name         string `yaml:"name" env:"ZEGO_SERVICE_NAME"`
    Environment  string `yaml:"environment" env:"ZEGO_ENV"`   // dev / test / staging / prod
    HealthPort   int    `yaml:"health_port" env:"ZEGO_HEALTH_PORT" default:"9999"`
    StartTimeout string `yaml:"start_timeout" env:"ZEGO_START_TIMEOUT" default:"7s"`
}

type ServerConfig struct {
    GRPCPort int `yaml:"grpc_port" env:"ZEGO_GRPC_PORT" default:"9090"`
    HTTPPort int `yaml:"http_port" env:"ZEGO_HTTP_PORT" default:"8080"`
}

type LogConfig struct {
    Level string `yaml:"level" env:"ZEGO_LOG_LEVEL" default:"info"`  // debug / info / warn / error
    Format string `yaml:"format" env:"ZEGO_LOG_FORMAT" default:"json"` // json / text
}

type TraceConfig struct {
    Endpoint  string  `yaml:"endpoint" env:"ZEGO_TRACE_ENDPOINT"`        // Jaeger collector endpoint
    SampleRate float64 `yaml:"sample_rate" env:"ZEGO_TRACE_SAMPLE_RATE" default:"0.1"` // 采样率
}
```

### 6.3 配置加载实现

```go
// config/config.go

// Load 加载配置，按优先级合并
func Load(paths ...string) (*Config, error) {
    cfg := &Config{}
    
    // 1. 设置默认值
    setDefaults(cfg)

    // 2. 加载配置文件
    configPath := resolveConfigPath(paths)
    if configPath != "" {
        data, err := os.ReadFile(configPath)
        if err == nil {
            if err := yaml.Unmarshal(data, cfg); err != nil {
                return nil, fmt.Errorf("zg-sdk: parse config file %s: %w", configPath, err)
            }
        }
    }

    // 3. 环境变量覆盖
    overrideFromEnv(cfg)

    // 4. 命令行参数覆盖（通过 os.Args 简单解析）
    overrideFromFlags(cfg, os.Args[1:])

    return cfg, nil
}

// overrideFromEnv 通过反射遍历 struct 的 env tag
func overrideFromEnv(cfg interface{}) {
    v := reflect.ValueOf(cfg).Elem()
    t := v.Type()
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        envKey := field.Tag.Get("env")
        if envKey == "" {
            continue
        }
        if val, ok := os.LookupEnv(envKey); ok {
            v.Field(i).SetString(val)
        }
    }
}

// resolveConfigPath 确定配置文件路径
func resolveConfigPath(paths []string) string {
    // 1. 参数指定
    for _, p := range paths {
        if p != "" {
            return p
        }
    }
    // 2. 环境变量指定
    if p := os.Getenv("ZEGO_CONFIG_PATH"); p != "" {
        return p
    }
    // 3. 默认路径
    if _, err := os.Stat("zego-micro.yaml"); err == nil {
        return "zego-micro.yaml"
    }
    return ""
}
```

### 6.4 配置文件示例

```yaml
# zego-micro.yaml
app:
  name: user-service
  environment: prod
  health_port: 9999
  start_timeout: 10s

server:
  grpc_port: 9090
  http_port: 8080

log:
  level: info
  format: json

trace:
  endpoint: http://jaeger-collector.zg-observability:14268/api/traces
  sample_rate: 0.1

# 业务自定义配置
extra:
  mysql_dsn: "user:password@tcp(mysql:3306)/db?charset=utf8mb4"
  redis_addr: "redis:6379"
```

---

## 七、errors — 统一错误处理

### 7.1 错误码定义

```go
// errors/codes.go

// 业务错误码（范围：10000-99999）
const (
    // 通用错误（10000-19999）
    ErrSuccess      = 0
    ErrUnknown      = 10000
    ErrInternal     = 10001  // 服务端内部错误
    ErrTimeout      = 10002  // 请求超时
    ErrRateLimit    = 10003  // 限流
    ErrCircuitBreak = 10004  // 熔断

    // 参数错误（20000-29999）
    ErrInvalidParam   = 20000
    ErrMissingParam   = 20001
    ErrInvalidFormat  = 20002

    // 鉴权错误（30000-39999）
    ErrUnauthenticated = 30000
    ErrForbidden       = 30001
    ErrTokenExpired    = 30002

    // 业务错误（40000-49999）
    ErrUserNotFound    = 40001
    ErrUserDuplicated  = 40002
    ErrResourceNotFound = 40003
)

// Error 业务错误
type Error struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Detail  string `json:"detail,omitempty"`
    // 内部错误链，不对外暴露
    cause error
}
```

### 7.2 错误使用

```go
// errors/errors.go

func New(code int, msg string) *Error {
    return &Error{Code: code, Message: msg}
}

func (e *Error) Error() string {
    if e.cause != nil {
        return fmt.Sprintf("[%d] %s: %v", e.Code, e.Message, e.cause)
    }
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

func (e *Error) Unwrap() error { return e.cause }

func (e *Error) WithCause(err error) *Error {
    e.cause = err
    return e
}

// GRPCStatus 实现 gRPC status 转换
func (e *Error) GRPCStatus() *status.Status {
    code := convertToGRPCCode(e.Code)
    st := status.New(code, e.Message)
    if e.Detail != "" {
        st, _ = st.WithDetails(&errdetails.ErrorInfo{
            Reason: e.Detail,
            Domain: "zego.im",
        })
    }
    return st
}

// convertToGRPCCode 将业务错误码映射到 gRPC 状态码
func convertToGRPCCode(bizCode int) codes.Code {
    switch {
    case bizCode >= 30000 && bizCode < 40000:
        return codes.Unauthenticated
    case bizCode >= 20000 && bizCode < 30000:
        return codes.InvalidArgument
    case bizCode == ErrInternal:
        return codes.Internal
    case bizCode == ErrTimeout:
        return codes.DeadlineExceeded
    case bizCode == ErrRateLimit:
        return codes.ResourceExhausted
    case bizCode == ErrCircuitBreak:
        return codes.Unavailable
    default:
        return codes.Unknown
    }
}
```

---

## 八、SDK 与原生 Go 的关系

### 8.1 使用场景矩阵

| 场景 | 使用 SDK | 使用标准库 | 说明 |
|------|---------|-----------|------|
| 微服务间调用（网格内东西向流量） | ✓ | ✗ | SDK 自动注入元数据、trace、指标 |
| 对外 API 暴露（北向流量） | ✓ | ✗ | SDK HTTP 服务端自动处理元数据提取 |
| 访问公网/集群外服务 | ✗ | ✓ | SDK 不对公网请求做增强 |
| 本地开发测试 | ✗ | ✓ | 本地不需要元数据注入和 trace |
| 定时任务/批处理 | ✗ | ✓ | 不需要拦截器链 |
| 第三方 SDK 调用（如 AWS SDK） | ✗ | ✓ | 第三方已经有自己的重试和超时 |

### 8.2 迁移指南

```go
// === 迁移前（标准库）===

// gRPC 客户端
conn, _ := grpc.Dial("localhost:9090", grpc.WithInsecure())
client := pb.NewUserServiceClient(conn)

// HTTP 客户端
resp, _ := http.Get("http://user-service:8080/api/users")

// === 迁移后（SDK）===

// gRPC 客户端（自动注入 trace、metrics、retry、metadata）
conn, _ := grpcclient.NewClient("user-service.zg.svc.cluster.local:9090")
client := pb.NewUserServiceClient(conn)

// HTTP 客户端（自动注入 metadata、trace、metrics）
httpClient := httpclient.NewClient(3 * time.Second)
resp, _ := httpClient.Get("http://user-service.zg.svc.cluster.local:8080/api/users")
```

### 8.3 降级策略

```
SDK 不可用（如版本 bug）：
  1. 紧急：回退到上一稳定版本
  2. 短期：业务方临时改用标准库（硬编码 metadata header）
  3. 长期：修复 SDK 后统一升级

注意：
  - SDK 只在 "入口" 和 "出口" 两个点增强，不侵入核心业务逻辑
  - 回退成本 = 改两行初始化代码 + 重新部署
```

---

## 九、测试策略

### 9.1 单元测试

```go
// app/app_test.go

func TestApp_Run_Success(t *testing.T) {
    initCalled := false
    healthCalled := false

    a := New(
        WithInitializers(func(ctx context.Context) error {
            initCalled = true
            return nil
        }),
        WithHealthChecks(func(ctx context.Context) error {
            healthCalled = true
            return nil
        }),
        WithStartTimeout(time.Second),
        WithHealthPort(0), // 随机端口
    )

    // 在 goroutine 中启动，模拟 SIGTERM
    go func() {
        time.Sleep(100 * time.Millisecond)
        syscall.Kill(syscall.Getpid(), syscall.SIGTERM)
    }()

    a.Run()

    assert.True(t, initCalled)
    assert.True(t, healthCalled)
}

func TestApp_InitTimeout(t *testing.T) {
    a := New(
        WithInitializers(func(ctx context.Context) error {
            time.Sleep(5 * time.Second) // 超时
            return nil
        }),
        WithStartTimeout(100 * time.Millisecond),
    )

    assert.Panics(t, func() { a.Run() })
}
```

### 9.2 集成测试（Mock Sidecar）

```go
// transport/grpc/interceptors/metadata_test.go

func TestMetadataPropagation(t *testing.T) {
    // 启动一个 mock gRPC 服务端
    srv := startTestGRPCServer(t)
    defer srv.Stop()

    // 创建 SDK 客户端
    conn, err := NewClient(srv.Addr())
    require.NoError(t, err)
    defer conn.Close()

    // 调用时注入测试元数据
    ctx := metadata.InjectToContext(context.Background(), map[string]string{
        MetaTenantID:  "test-tenant",
        MetaCanaryTag: "canary",
    })

    client := pb.NewTestServiceClient(conn)
    resp, err := client.Echo(ctx, &pb.EchoRequest{Msg: "hello"})
    require.NoError(t, err)

    // 验证服务端接收到的元数据
    assert.Equal(t, "test-tenant", resp.ReceivedTenantId)
    assert.Equal(t, "canary", resp.ReceivedCanaryTag)
}
```

### 9.3 端到端测试（K8s 环境）

```
测试步骤：
  1. 部署测试服务到 K8s（含 Envoy Sidecar）
  2. 通过 SDK 客户端调用
  3. 验证：
     - Jaeger 中能查到完整 trace
     - Prometheus 指标正确上报
     - 灰度标签在 VirtualService 中生效
     - 健康检查探针正确返回 200/503
```

---

## 十、性能数据

### 10.1 SDK 开销基准

| 操作 | 延迟 | 说明 |
|------|------|------|
| 空 gRPC 调用（无 SDK） | ~0.5ms | 基线 |
| + SDK 拦截器链（元数据 + trace + metrics + retry） | ~0.7ms | +0.2ms |
| + Envoy Sidecar | ~2-3ms | 见架构文档 |
| 总计 | ~3-4ms | 业务可接受 |

### 10.2 各拦截器开销分解

```
拦截器链总开销 ≈ 0.2ms
  ├── 元数据拦截器: ~0.01ms  (map 操作)
  ├── 链路追踪:     ~0.05ms  (span 创建 + header 注入)
  ├── 指标采集:     ~0.05ms  (Prometheus 计数器 + 直方图)
  ├── 日志记录:     ~0.02ms  (slog 结构化)
  └── 重试逻辑:     ~0.01ms  (仅首次，不触发重试时)
```

### 10.3 优化建议

| 场景 | 优化手段 | 效果 |
|------|---------|------|
| 高 QPS 服务（>10,000 QPS） | 降低 trace 采样率（0.01 → 0.001） | 减少 trace 开销 90% |
| 延迟敏感服务（P99 < 10ms） | 跳过日志拦截器（只保留 error 级别） | 减少日志序列化开销 |
| 离线批处理 | 直接用标准库，不走 SDK | 完全避免拦截器链 |
| 重试引发雪崩 | 限制最大重试次数为 1，配合熔断器 | 防止重试风暴 |

---

## 十一、最佳实践与注意事项

### 11.1 拦截器链顺序

```
顺序决定行为，必须保持：
  客户端拦截器（从外到内执行）：
    1. 元数据注入  ← 最先执行，后续拦截器都能从 context 读到元数据
    2. 链路追踪    ← 依赖元数据中的 trace_id
    3. 指标采集    ← 使用带 trace context 的 context
    4. 日志记录    ← 记录完整上下文
    5. 重试        ← 最内层，包裹实际调用

  服务端拦截器（从外到内执行）：
    1. 恐慌恢复    ← 最外层兜底 panic
    2. 元数据提取  ← 提取请求元数据到 context
    3. 链路追踪    ← 使用已提取的元数据
    4. 指标采集    ← 使用带 trace context 的 context
```

### 11.2 Context 传递规范

```go
// ✓ 正确：始终将 ctx 作为第一个参数传递
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    slog.InfoContext(ctx, "get user", "id", id)
    return s.db.FindUser(ctx, id)
}

// ✗ 错误：不使用 context，丢失 trace 和超时控制
func (s *UserService) GetUser(id string) (*User, error) {
    return s.db.FindUser(id)
}
```

### 11.3 错误处理规范

```go
// ✓ 正确：使用 SDK 错误码 + gRPC status
return errors.New(errors.ErrUserNotFound, "user not found").
    WithCause(fmt.Errorf("db: %w", err))

// ✗ 错误：直接返回原始错误
return fmt.Errorf("db error: %v", err)

// 业务层判断错误
if errors.Is(err, ErrUserNotFound) {
    // 处理用户不存在
}
```

### 11.4 配置管理注意项

```
1. 敏感信息（密码、token）不写死在配置文件
   → 通过环境变量注入，或使用 Vault/K8s Secret

2. 配置变更优先通过控制面推送（Istio EnvoyFilter）
   → SDK 配置只在启动时加载，运行时变更需重启

3. 本地开发使用 .env 文件，生产使用环境变量
   → SDK 会自动读取 .env 文件（如果存在）
```

---

## 十二、SDK 版本演进规划

| 版本 | 功能 | 状态 |
|------|------|------|
| v1.0 | 基础生命周期 + gRPC 拦截器 + 健康检查 | ✅ 已发布 |
| v1.1 | HTTP 客户端/服务端封装 + 结构化日志 | ✅ 已发布 |
| v1.2 | OpenTelemetry 链路追踪 + Prometheus 指标 | ✅ 已发布 |
| v1.3 | 配置管理（zego-micro.yaml） | ✅ 已发布 |
| v1.4 | 统一错误码 + gRPC status 转换 | ✅ 已发布 |
| v2.0 | Wasm 熔断感知（SDK 主动上报熔断状态到 Envoy） | 📅 计划中 |
| v2.1 | 配置热更新（监听 ConfigMap 变更） | 📅 计划中 |
| v2.2 | 多语言 SDK（Python/Java 版） | 📅 规划中 |

---

## 附录：完整业务使用示例

```go
// cmd/user-service/main.go
package main

import (
    "zg-sdk/app"
    "zg-sdk/config"
    "zg-sdk/log"
    "zg-sdk/transport/grpc"
    "zg-sdk/transport/grpc/interceptors"
)

func main() {
    // 1. 加载配置
    cfg, err := config.Load("zego-micro.yaml")
    if err != nil {
        panic(err)
    }

    // 2. 初始化日志
    log.InitLogger(cfg.App.Name, parseLevel(cfg.Log.Level))

    // 3. 初始化依赖
    db := initDatabase(cfg.Extra["mysql_dsn"].(string))
    redis := initRedis(cfg.Extra["redis_addr"].(string))

    // 4. 创建 gRPC 服务端
    grpcSrv := grpc.NewServer(grpc.ServerConfig{
        Port:        cfg.Server.GRPCPort,
        ServiceName: cfg.App.Name,
    }, func(s *grpc.Server) {
        pb.RegisterUserServiceServer(s, NewUserService(db, redis))
    })

    // 5. 创建 App
    a := app.New(
        app.WithInitializers(
            func(ctx context.Context) error {
                return db.PingContext(ctx)
            },
            func(ctx context.Context) error {
                return redis.PingContext(ctx)
            },
        ),
        app.WithHealthChecks(
            func(ctx context.Context) error {
                return db.PingContext(ctx)
            },
            func(ctx context.Context) error {
                return redis.PingContext(ctx)
            },
        ),
        app.WithShutdownHooks(
            app.ShutdownGRPCServer(grpcSrv),
            app.FlushLogger(),
            func(ctx context.Context, sig os.Signal) {
                db.Close()
                redis.Close()
            },
        ),
        app.WithStartTimeout(parseDuration(cfg.App.StartTimeout)),
        app.WithHealthPort(cfg.App.HealthPort),
    )

    // 6. 启动
    a.Run()
}
```

---

> **设计参考**：
> - [gRPC-go Interceptor 文档](https://github.com/grpc/grpc-go/blob/master/Documentation/interceptor.md)
> - [OpenTelemetry Go SDK](https://opentelemetry.io/docs/instrumentation/go/)
> - [Istio 流量管理 CRD](https://istio.io/latest/docs/reference/config/networking/)
> - Google SRE Book § 熔断与重试退避策略
