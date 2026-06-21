# Go 微服务架构学习资料

> 本文档整理了基于 Go 语言的微服务架构学习资源，涵盖框架选型、服务治理、数据库缓存、监控运维、实战项目等核心领域。

---

## 一、Go 微服务框架

### 1.1 gRPC（官方推荐）

| 属性 | 说明 |
|------|------|
| **官网** | https://grpc.io/ |
| **GitHub** | https://github.com/grpc/grpc-go |
| **定位** | Google 开源的 RPC 框架，Cloud Native 默认标准 |
| **协议** | Protocol Buffers (Protobuf)，HTTP/2 传输 |
| **特点** | 高性能、多语言支持、强类型接口定义 |

**核心优势：**
- Protobuf 序列化效率高，体积小
- HTTP/2 多路复用，单连接并发
- 官方支持几乎所有主流语言
- 与 Kubernetes、Istio 等云原生生态无缝集成

**适用场景：** 对性能要求高的内部服务通信、跨语言微服务架构

---

### 1.2 go-zero（好未来开源）

| 属性 | 说明 |
|------|------|
| **官网** | https://go-zero.dev/ |
| **GitHub** | https://github.com/zeromicro/go-zero |
| **Stars** | 27.4k+ |
| **定位** | 一站式微服务框架，强调工程化与高性能平衡 |
| **开源方** | 好未来（TAL） |

**核心能力：**

| 能力 | 说明 |
|------|------|
| goctl 工具链 | 自动生成 API/RPC/Model 代码，支持 6 种客户端 |
| 服务治理 | 内置熔断、限流、超时、熔断器 |
| 服务发现 | 支持 etcd/Consul/Nacos |
| 可观测性 | Prometheus 监控、Jaeger 链路追踪 |
| 缓存 | 自动缓存管理（主键/索引/空缓存防穿透） |
| 数据库 | 支持原生 ORM 和 GORM |

**性能数据：**
- QPS：83,000+（50 并发/百万请求压测）
- 延迟：< 1.2ms
- 内存占用：仅 28MB

**适用场景：** 高并发 Web 服务、电商/社交类应用、需要快速落地的中小型项目

---

### 1.3 go-kratos（哔哩哔哩开源）

| 属性 | 说明 |
|------|------|
| **官网** | https://go-kratos.dev/ |
| **GitHub** | https://github.com/go-kratos/kratos |
| **Stars** | 22.3k+ |
| **定位** | 企业级微服务框架，强调标准化与可观测性 |
| **开源方** | 哔哩哔哩 |

**核心能力：**

| 能力 | 说明 |
|------|------|
| Protobuf IDL | 强制接口定义，统一服务契约 |
| 双协议支持 | HTTP/gRPC 无缝切换 |
| 分层架构 | API/Service/Data 三层分离 |
| 可观测性 | 深度整合 OpenTelemetry |
| 生态扩展 | kratos-transport（MQ）、kratos-authn/authz（安全） |

**性能数据：**
- QPS：78,000+
- 延迟：1.5-3ms

**适用场景：** 中大型团队、需要标准化接口的分布式系统、B 站级百节点集群

---

### 1.4 go-micro

| 属性 | 说明 |
|------|------|
| **官网** | https://micro.dev/ |
| **GitHub** | https://github.com/asim/go-micro |
| **Stars** | 21.3k+ |
| **定位** | 插件化微服务框架，强调灵活性与扩展性 |

**核心特性：**
- 插件化架构：支持 Consul/Nacos/Etcd 等多种注册中心
- 多协议支持：HTTP/gRPC/MQTT
- 云原生友好：原生支持 Kubernetes
- 模块化设计：RPC、Pub/Sub 组件解耦

**适用场景：** 需要高度定制化、跨语言微服务互通、复杂协议适配

---

### 1.5 go-kit

| 属性 | 说明 |
|------|------|
| **官网** | https://gokit.io/ |
| **GitHub** | https://github.com/go-kit/kit |
| **Stars** | 26.1k+ |
| **定位** | 微服务工具集，强调组合式设计 |

**核心特性：**
- 轻量级工具集，按需组合
- 支持多种传输协议和中间件
- 适合高度定制化场景
- 学习曲线较陡

**适用场景：** 中大型项目、需要模块化设计的复杂系统

---

### 1.6 其他框架

| 框架 | GitHub | Stars | 特点 |
|------|--------|-------|------|
| Kitex | https://github.com/cloudwego/kitex | 6.6k | 字节跳动出品，高性能 RPC |
| rpcx | https://github.com/smallnest/rpcx | 7.9k | 轻量级 RPC，支持多种协议 |
| dubbo-go | https://github.com/apache/dubbo-go | 4.6k | Dubbo 协议的 Go 实现 |
| Jupiter | https://github.com/jupiter_rpc/jupiter | 4.3k | 斗鱼开源，配置驱动治理 |

---

### 1.7 框架选型对比

| 维度 | go-zero | Kratos | go-micro | go-kit | gRPC |
|------|---------|--------|----------|--------|------|
| **学习曲线** | 平缓 | 中等 | 较陡 | 较陡 | 较陡 |
| **集成度** | 高 | 高 | 中 | 低 | 需自行集成 |
| **性能** | 最高 | 高 | 中 | 中 | 最高 |
| **灵活性** | 中 | 中 | 高 | 高 | 高 |
| **中文文档** | 完善 | 完善 | 一般 | 一般 | 一般 |
| **社区活跃** | 活跃 | 活跃 | 一般 | 一般 | 活跃 |

**选型建议：**
- **快速落地**：推荐 go-zero，工具链完善，中文友好
- **企业标准化**：推荐 Kratos，适合中大型团队
- **高度定制**：推荐 go-micro 或自研 + gRPC
- **极致性能**：推荐 gRPC + Istio 服务网格

---

## 二、服务治理

### 2.1 服务发现

#### Consul

| 属性 | 说明 |
|------|------|
| **官网** | https://www.consul.io/ |
| **特点** | 功能丰富、开箱即用、多数据中心支持 |
| **CAP** | CP（一致性优先）/AP 可选 |
| **健康检查** | HTTP/TCP/Script/TTL |

**Go 客户端：**
```go
import consulapi "github.com/hashicorp/consul/api"

// 创建客户端
config := consulapi.DefaultConfig()
client, _ := consulapi.NewClient(config)

// 注册服务
registration := &consulapi.AgentServiceRegistration{
    ID:   "user-service-1",
    Name: "user-service",
    Port: 8080,
    Check: &consulapi.AgentServiceCheck{
        HTTP:     "http://localhost:8080/health",
        Interval: "10s",
    },
}
client.Agent().ServiceRegister(registration)

// 服务发现
services, _, _ := client.Health().Service("user-service", "", true, nil)
```

**适用场景：** 多语言混合架构、需要 Web UI 的团队、需要 DNS 接口的项目

---

#### Etcd

| 属性 | 说明 |
|------|------|
| **官网** | https://etcd.io/ |
| **特点** | 强一致性、Raft 算法、Kubernetes 默认后端 |
| **CAP** | CP（强一致） |
| **协议** | gRPC/Protobuf |

**Go 客户端：**
```go
import "go.etcd.io/etcd/client/v3"

// 创建客户端
client, _ := clientv3.New(clientv3.Config{
    Endpoints:   []string{"localhost:2379"},
    DialTimeout: 5 * time.Second,
})

// Lease 模式注册
resp, _ := client.Grant(ctx, 10)
client.Put(ctx, "service/user/1", "localhost:8080", clientv3.WithLease(resp.ID))

// Watch 监听变化
rch := client.Watch(ctx, "service/user/")
for wresp := range rch {
    // 处理服务变更
}
```

**适用场景：** Kubernetes 生态、对数据强一致性有要求的配置中心、分布式锁

---

#### Nacos（Go 客户端）

| 属性 | 说明 |
|------|------|
| **官网** | https://nacos.io/ |
| **特点** | Alibaba 开源、功能全面（注册+配置）、AP/CP 可切换 |
| **Go 客户端** | https://github.com/nacos-group/nacos-sdk-go |

**适用场景：** Spring Cloud 迁移项目、需要配置中心的企业

---

### 2.2 配置中心

| 方案 | 特点 | GitHub |
|------|------|--------|
| Viper | 非侵入式、兼容多种配置源 | https://github.com/spf13/viper |
| Consul KV | 注册+配置二合一 | 内置 |
| Etcd | 强一致配置 | 内置 |
| Nacos | 功能全面 | https://nacos.io |

**Viper 使用示例：**
```go
import "github.com/spf13/viper"

viper.SetConfigName("config")  // 配置文件名
viper.SetConfigType("yaml")    // 配置文件类型
viper.AddConfigPath("./config") // 搜索路径
viper.AutomaticEnv()           // 环境变量覆盖
viper.ReadInConfig()           // 读取配置
```

---

### 2.3 API 网关

| 网关 | 定位 | 技术栈 | 特点 |
|------|------|--------|------|
| **Kong** | 插件化 API 管理 | Lua + Nginx | 插件生态丰富（百+官方插件） |
| **APISIX** | 云原生高性能 | Lua + Nginx | 毫秒级热更新、多语言插件 |
| **Traefik** | 容器自动路由 | Go | 自动服务发现、K8s 原生 |
| **APISIX-Go** | Go 扩展 | Go | 可用 Go 编写插件 |

**Kong 配置示例：**
```yaml
# kong.yml
services:
  - name: user-service
    url: http://user-svc:8080
    routes:
      - name: user-route
        paths:
          - /api/user

  - name: order-service
    url: http://order-svc:8080
    routes:
      - name: order-route
        paths:
          - /api/order
```

**选型建议：**
- 高性能+插件丰富：Kong/APISIX
- 容器+K8s 自动路由：Traefik
- 国产化+高并发：APISIX

---

### 2.4 负载均衡与熔断

**负载均衡策略：**

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| 轮询 (Round Robin) | 均匀分配 | 节点性能一致 |
| 加权轮询 (WRR) | 按权重分配 | 节点性能差异大 |
| 最少连接 (Least Conn) | 分配给负载最低 | 长连接场景 |
| P2C (Power of Two Choices) | 随机选2取最优 | go-zero 默认，高并发 |

**熔断器状态：**

```
关闭 → (错误率超阈值) → 打开 → (半开探测) → 关闭
                                ↓
                             (仍失败) → 打开
```

**常用库：**
- `sony/gobreaker` - 轻量熔断器
- `go-zero` 内置熔断器
- ` hystrix-go` - Hystrix 的 Go 实现

---

## 三、数据库与缓存

### 3.1 GORM

| 属性 | 说明 |
|------|------|
| **官网** | https://gorm.io/ |
| **GitHub** | https://github.com/go-gorm/gorm |
| **特点** | 功能完善、生态丰富、中文友好 |

**核心特性：**
- 多数据库支持：MySQL/PostgreSQL/SQLite/TiDB
- 关联查询：Preload、Joins、嵌套查询
- 事务支持：声明式/手动模式
- 插件系统：分库分表、读写分离

**基础使用：**
```go
import (
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{})

// CRUD
db.Create(&user)
db.First(&user, 1)
db.Where("name = ?", "Alice").Find(&users)
db.Model(&user).Updates(map[string]interface{}{"age": 20})

// 事务
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&order)
    tx.Create(&payment)
    return nil
})
```

---

### 3.2 分库分表

| 方案 | GitHub | 特点 |
|------|--------|------|
| GORM Sharding | https://github.com/go-gorm/sharding | 无侵入、64 分片、Snowflake ID |
| ShardingSphere-Proxy | 代理层 | 无代码改造、但失去细粒度控制 |
| 手动路由 | 自研 | 灵活、适合规则固定场景 |

**GORM Sharding 示例：**
```go
db.Use(sharding.Register(sharding.Config{
    ShardingKey:         "user_id",
    NumberOfShards:      64,
    PrimaryKeyGenerator: sharding.PKSnowflake,
}, "orders", "audit_logs"))

// 自动路由到分片表
db.Create(&Order{UserID: 2}) // INSERT INTO orders_2
db.Where("user_id", 2).Find(&orders) // SELECT FROM orders_2
```

---

### 3.3 Redis 分布式缓存

| 客户端 | GitHub | 特点 |
|--------|--------|------|
| go-redis | https://github.com/redis/go-redis | 功能全面、文档完善 |
| redigo | https://github.com/gomodule/redigo | 轻量、连接池 |

**缓存三剑客（防穿透/击穿/雪崩）：**
```go
func (r *EasyRedis) GetCacheWithProtection(
    key string,
    nullCacheExpire, mutexExpire int,
    fallback func() (interface{}, error)) (interface{}, error) {
    // 1. 先查缓存
    val, err := r.Get(key)
    if err == nil { return val, nil }
    
    // 2. 空缓存保护（防穿透）
    if err == redis.Nil {
        // 获取分布式锁
        lockKey := key + "_lock"
        if r.AcquireLock(lockKey, mutexExpire) {
            defer r.ReleaseLock(lockKey)
            // 回源
            result, err := fallback()
            // 写入缓存（含空值防止穿透）
            r.SetEx(key, result, nullCacheExpire)
            return result, nil
        }
    }
    return nil, err
}
```

---

### 3.4 分布式事务 - DTM

| 属性 | 说明 |
|------|------|
| **官网** | https://dtm.pub/ |
| **GitHub** | https://github.com/dtm-labs/dtm |
| **语言支持** | Go/Java/Python/PHP/Node.js/C# |

**支持的事务模式：**

| 模式 | 原理 | 适用场景 |
|------|------|----------|
| TCC | Try-Confirm-Cancel 三阶段 | 强一致性、预留资源 |
| SAGA | 正向补偿回滚 | 长事务、跨服务 |
| XA | 两阶段提交 | 强一致、对性能要求不高 |
| 消息 | 本地消息表变体 | 最终一致、性能优先 |

**TCC 示例：**
```go
import "github.com/dtm-labs/dtmcli"

// 发起 TCC 事务
gid := dtmcli.MustGenGid(dtmServer)
tcc := &dtmcli.TccGlobalTransaction{
    GID:      gid,
    Namespace: "order",
}
tcc.AddAction("trans_svc", "/TransOut/Try", "/TransOut/Confirm", "/TransOut/Cancel", req)
err := tcc.Submit()
```

**适用场景：** 跨服务转账、订单创建、库存扣减

**TCC 简图说明：**

```
┌────────────┐     Try      ┌────────────┐
│  调用方A   │ ───────────► │  服务B Try │
│  (发起者)  │◄─────────── │  服务C Try │
└────────────┘              └────────────┘
      │                          │
      ▼                          ▼
[全部Try预留资源成功]
      │
      ▼
[发起Confirm/Cancel]
      │
    ┌─┴─────────────┐
    │确认(Confirm)  │ 成功->Confirm
    │或取消(Cancel) │ 失败->Cancel
    └───────────────┘
```

- Try：资源预留，不实际扣减
- Confirm：业务正式提交
- Cancel：回滚释放资源

---

### 3.5 消息队列

#### Kafka vs RabbitMQ 对比

| 维度 | Kafka | RabbitMQ |
|------|-------|----------|
| **吞吐量** | 10万~100万 TPS | 万级 TPS |
| **消息模型** | Topic + Partition（日志） | Exchange + Queue（路由） |
| **消费模式** | Pull（消费者拉取） | Push（Broker 推送） |
| **消息路由** | 简单（Topic+Key） | 强大（多种交换机） |
| **消息持久化** | 默认持久化 | 可配置 |
| **适用场景** | 日志、大数据、流处理 | 微服务解耦、复杂路由 |

**Kafka Go 客户端：**

| 库 | GitHub | 特点 |
|-----|--------|------|
| segmentio/kafka-go | https://github.com/segmentio/kafka-go | 轻量、无 CGO、推荐 |
| Shopify/sarama | https://github.com/Shopify/sarama | 成熟、功能全 |

**Kafka 生产者：**
```go
import "github.com/segmentio/kafka-go"

writer := &kafka.Writer{
    Addr:     kafka.TCP("localhost:9092"),
    Topic:    "user-events",
    Balancer: &kafka.LeastBytes{},
}

writer.WriteMessages(ctx, kafka.Message{
    Key:   []byte("user-123"),
    Value: []byte(`{"event":"login","time":"2024-01-01"}`),
})
```

**RabbitMQ 生产者：**
```go
import "github.com/rabbitmq/amqp091-go"

conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
ch, _ := conn.Channel()

ch.QueueDeclare("order_queue", true, false, false, false, nil)
ch.Publish("", "order_queue", false, false, amqp.Publishing{
    ContentType: "application/json",
    Body:        []byte(`{"order_id":"123"}`),
})
```

---

## 四、监控与运维

### 4.1 可观测性三大支柱

| 支柱 | 核心问题 | 工具 |
|------|----------|------|
| Metrics（指标） | 发生了什么？当前状态如何？ | Prometheus、Grafana |
| Logs（日志） | 为什么发生？具体事件？ | ELK、Loki |
| Traces（追踪） | 在哪个环节？调用链如何？ | Jaeger、Zipkin |

---

### 4.2 Prometheus

| 属性 | 说明 |
|------|------|
| **官网** | https://prometheus.io/ |
| **定位** | 云原生监控事实标准 |
| **模型** | Pull 模型、时序数据库 |
| **特点** | CNCF 毕业项目、K8s 原生支持 |

**Go 集成示例：**
```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var httpRequestsTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    },
    []string{"method", "path", "status"},
)

func init() {
    prometheus.MustRegister(httpRequestsTotal)
}

http.Handle("/metrics", promhttp.Handler())
http.HandleFunc("/api/user", func(w http.ResponseWriter, r *http.Request) {
    httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, "200").Inc()
})
```

---

### 4.3 Jaeger 链路追踪

| 属性 | 说明 |
|------|------|
| **官网** | https://www.jaegertracing.io/ |
| **协议** | OpenTelemetry（OTel）原生支持 |
| **特点** | 分布式追踪、Span 记录、依赖关系图 |

**Go 集成 OpenTelemetry：**
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/jaeger"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

func initTracer() {
    exp, _ := jaeger.New(jaeger.WithCollectorEndpoint(
        jaeger.WithEndpoint("http://localhost:14268/api/traces"),
    ))
    tp := trace.NewTracerProvider(trace.WithBatcher(exp))
    otel.SetTracerProvider(tp)
}

// 自动追踪 HTTP 请求
handler := otelhttp.NewHandler(
    http.HandlerFunc(handleRequest),
    "handle-request",
)
```

---

### 4.4 日志系统

#### Loki + Promtail + Grafana（轻量方案）

| 组件 | 作用 |
|------|------|
| Loki | 日志存储与查询引擎 |
| Promtail | 日志采集代理 |
| Grafana | 统一展示界面 |

**优势：**
- 存储成本仅为 ELK 的 1/5~1/10
- 与 Prometheus 标签体系一致
- Helm 一键部署

**LogQL 查询示例：**
```logql
{service="user-service"} |= "error"
{service="order-service"} | json | status_code >= 500
```

#### ELK（经典方案）

| 组件 | 作用 |
|------|------|
| Elasticsearch | 日志存储与全文检索 |
| Logstash | 日志采集与处理 |
| Kibana | 可视化界面 |
| Filebeat | 轻量日志采集 |

---

### 4.5 Grafana 可视化

| 能力 | 说明 |
|------|------|
| 监控面板 | Metrics/Logs/Traces 三位一体 |
| 告警规则 | 基于 PromQL，支持多渠道 |
| 数据源 | Prometheus/Loki/Jaeger 统一入口 |
| SLO 监控 | 错误预算、发布冻结 |

---

## 五、实战项目推荐

### 5.1 开源项目

| 项目 | GitHub | 特点 |
|------|--------|------|
| go-ecommerce | https://github.com/thejasmeetsingh/go-ecommerce | Gin + JWT + Redis + Prometheus |
| eda-ecommerce | https://github.com/Joe5451/eda-ecommerce | 事件驱动 + Saga + Kafka |
| go-mall | 哔哩哔哩极客教程配套 | go-zero + 电商完整链路 |

**EDA 电商系统架构：**
```
用户服务 → API 网关 → 订单服务 → Saga 编排器
                              ↓
                         支付服务/库存服务/通知服务
                              ↓
                           Kafka 事件流
```

---

### 5.2 学习路线

#### 四阶段学习规划

| 阶段 | 时长 | 目标 | 核心内容 |
|------|------|------|----------|
| L0 起步 | 2 周 | 生产级 HTTP 服务 | Gin + GORM + CRUD |
| L1 进阶 | 4 周 | 掌握并发与标准库 | goroutine/channel/并发安全 |
| L2 高阶 | 8 周 | 微服务与云原生 | gRPC + 服务治理 + K8s |
| L3 专家 | 持续 | 中间件与性能优化 | 源码阅读 + 贡献开源 |

#### 阶段一：基础（2 周）

**目标：** 能独立写出基本的 HTTP 服务

**学习内容：**
- Go 基础语法：变量、函数、结构体、接口
- 并发初探：goroutine + channel + select
- Web 框架：Gin 路由、中间件、参数绑定
- 数据库：GORM CRUD、关联查询、事务

**实战项目：**
- 任务管理系统（TODO API）
- 用户认证服务（JWT + OAuth2）

---

#### 阶段二：进阶（4 周）

**目标：** 掌握 Go 并发模型，能写高性能程序

**学习内容：**
- GMP 调度模型
- sync 包：Mutex/RWMutex/atomic
- Context：链路传值、取消、超时
- Runtime：
  - GC：垃圾回收机制，自动回收无用内存，避免内存泄漏。
  - pprof：性能分析工具，用于分析 CPU、内存等运行时性能瓶颈。
  - 内存逃逸：变量本应分配在栈上但因被引用而逃逸到堆上，可能导致 GC 负担增加。

**实战项目：**
- 并发爬虫（万级 QPS）
- 线程安全缓存（LRU + 过期淘汰）

---

#### 阶段三：高阶（8 周）

**目标：** 理解微服务架构，能构建分布式系统

**学习内容：**

| 模块 | 技术栈 |
|------|--------|
| 服务框架 | go-zero / Kratos / gRPC |
| 服务通信 | Protobuf + gRPC + HTTP 双协议 |
| 服务治理 | 注册发现 + 负载均衡 + 熔断限流 |
| 可观测性 | OpenTelemetry + Jaeger + Prometheus |
| 消息队列 | Kafka / RabbitMQ |
| 分布式事务 | DTM（TCC/SAGA）—— 代码示例（SAGA 模式）：下单成功 -> 扣库存成功 -> 扣余额成功，所有步骤要么全部完成，要么全部回滚 |
| 部署 | Docker + Kubernetes |

**实战项目：**
- 用户服务（gRPC + etcd）
- 订单服务（Saga 分布式事务）
- 网关服务（JWT + 限流）
- 压测验证（K6 + 5 万并发）

---

#### 阶段四：专家（持续）

**方向：**
- 运行时：GC 调优、PGO
- 中间件：自写 Raft KV、分布式锁
- 开源贡献：向 K8s/etcd 提 PR
- 架构演进：Service Mesh（Istio）

---

## 六、技术选型参考

### 6.1 初创团队（快速上线）

```
框架: go-zero
通信: HTTP + JSON（快速迭代）
注册: Consul（开箱即用）
缓存: Redis
监控: Prometheus + Grafana
部署: Docker Compose
```

### 6.2 中型团队（稳定可扩展）

```
框架: Kratos / go-zero
通信: gRPC + Protobuf
注册: etcd / Consul
配置: Nacos / Apollo
缓存: Redis Cluster
数据库: MySQL + GORM Sharding
消息: Kafka
监控: Loki + Prometheus + Jaeger
部署: Kubernetes + Helm
```

### 6.3 大型团队（企业级）

```
框架: 自研 + gRPC
通信: gRPC + Protobuf（跨语言）
注册: etcd / Consul（多集群）
配置: 配置中心（自研/Apollo）
缓存: Redis + 分布式锁
数据库: TiDB / CockroachDB（NewSQL）
消息: Kafka + RabbitMQ
分布式事务: DTM
监控: 全链路可观测性
部署: Kubernetes + Istio Service Mesh
```

---

## 七、参考资源

### 官方文档
- Go 语言：https://go.dev/
- gRPC：https://grpc.io/docs/
- Kubernetes：https://kubernetes.io/zh/
- Prometheus：https://prometheus.io/docs/

### 优质学习资源
| 资源 | 说明 |
|------|------|
| Go 夜读 | 社区技术分享 |
| Go 语言圣经 | 经典教材 |
| 《Go 实战》 | 进阶必读 |
| GoCN 社区 | 中文社区 |
| Gopher China | 开发者大会 |

### GitHub 优质项目
| 项目 | 说明 |
|------|------|
| go-zero | 微服务框架 |
| kratos | 企业级框架 |
| dtm | 分布式事务 |
| go-redis | Redis 客户端 |
| gin | Web 框架 |
| grafana | 监控可视化 |

---

*文档更新时间：2025年*
