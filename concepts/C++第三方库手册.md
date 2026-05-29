# C++ 常用依赖库手册（后台服务器方向）

> 侧重**后台服务 / 分布式系统**日常开发中真实会用到的第三方依赖，按功能分类，给出常见选型建议。
> 
> 默认假设：使用 C++17 及以上，构建系统为 CMake，包管理以 vcpkg / Conan 为主。

See also: [[C++网络编程]], [[Boost.Asio]], [[C++多线程与并发]], [[C++项目深挖问题]]

---

## 1. HTTP / Web 框架（服务端）

### 1.1 Drogon（推荐）

- **定位**：现代 C++14/17 Web 框架，偏“全家桶”，可直接写完整后端服务。
- **功能**：
  - HTTP/HTTPS、WebSocket、RESTful API
  - 路由（基于控制器与注解）、过滤器、插件机制
  - 内置 ORM（支持 PostgreSQL / MySQL / SQLite 等）
  - 内置定时任务、静态文件服务
- **优点**：
  - 接口风格有点类似 Spring MVC，写业务比较顺手
  - 性能不错，异步 IO + 多线程模型
  - 文档和示例相对完整，上手成本可控
- **适用场景**：
  - 中小型到中大型 HTTP 微服务
  - 快速搭建管理后台 / 内部接口服务
- **仓库**：`https://github.com/drogonframework/drogon`

### 1.2 Crow

- **定位**：轻量级 C++ Web 框架，风格类似 Python Flask。
- **特点**：
  - 路由 + 中间件，支持 JSON、WebSocket
  - 单头文件版本可快速集成到现有项目
- **适用场景**：
  - 轻量后台、小型管理工具、监控面板的 HTTP 接口
- **仓库**：`https://github.com/CrowCpp/Crow`

### 1.3 Boost.Beast + Asio（更底层）

- **定位**：基于 Boost.Asio 的 HTTP / WebSocket 协议实现，更接近“库”而非“框架”。
- **特点**：
  - 自己掌控连接生命周期、线程模型、缓冲区管理等
  - 可按需集成到已有 Asio 架构中
- **适用场景**：
  - 对性能、延迟、资源使用有较高要求，需要**完全掌控网络层细节**的服务
- **仓库**：`https://github.com/boostorg/beast`

### 1.4 其它可选 HTTP 服务器

- **Restinio**：嵌入式 HTTP/WebSocket 服务器，偏 IoT / 小型守护进程。
- **CivetWeb / Mongoose**：C/C++ 实现的嵌入式 HTTP 服务器，适合嵌入到现有应用做简单 HTTP 控制接口。

### 1.5 独立 WebSocket 库

- **WebSocket++**
  - 纯 C++ 实现的 WebSocket 客户端/服务器库，基于 Asio，支持 TLS。
  - 提供较完整的 WebSocket 协议支持，适合聊天室、推送服务器等长连接场景。
  - 仓库：`https://github.com/zaphoyd/websocketpp`
- **uWebSockets**
  - 以性能著称的 WebSocket / HTTP 库，适合高并发实时推送服务。
  - C++ 实现，依赖较少，常见于交易撮合、行情推送、IM 等场景。
  - 仓库：`https://github.com/uNetworking/uWebSockets`

---

## 2. HTTP 客户端 / 网络请求

### 2.1 libcurl（事实标准）

- **定位**：支持多协议的 C 客户端库（HTTP/HTTPS、FTP、SMTP 等）。
- **特点**：
  - 功能极其完整，跨平台，成熟稳定
  - C 接口，C++ 项目中需要再封装一层更友好接口
- **适用场景**：
  - 复杂 HTTP 客户端需求：代理、认证、断点续传、自定义 TLS 等
- **网站**：`https://curl.se/libcurl`

### 2.2 cpr（基于 libcurl 的 C++ 封装）

- **定位**：给 C++ 程序员用的“requests 风格” HTTP 客户端。
- **特点**：
  - API 接口类似 Python `requests`，非常易用
  - 底层仍然是 libcurl，功能可靠
- **适用场景**：
  - 普通后台服务中发 HTTP 请求调用第三方 API、自家其它微服务
- **仓库**：`https://github.com/libcpr/cpr`

### 2.3 Boost.Asio / standalone Asio

- **定位**：通用异步 IO / 网络编程库。
- **特点**：
  - 支持 TCP/UDP、定时器、串口等
  - 做自定义协议、内网 RPC、网关代理非常合适
- **适用场景**：
  - 有自己自定义二进制协议，或者需要较复杂的连接管理、复用模型
- **网站**：`https://think-async.com/Asio`

### 2.4 HTTP 协议解析库（仅解析层）

- **定位**：只负责 HTTP 报文解析/生成，本身不做网络 IO，适合嵌入自研网络栈或高性能网关内核中。
- **典型选择**：
  - **llhttp**：Node.js 新一代 HTTP 解析器，基于状态机生成，性能很好，C 实现，易于嵌入。
    - 仓库：`https://github.com/nodejs/llhttp`
  - **http-parser**：Node.js 早期使用的 C 语言 HTTP 解析器，接口简单、成熟稳定。
    - 仓库：`https://github.com/nodejs/http-parser`
  - **picohttpparser**：极简、追求极致性能的 HTTP 解析器，代码量很小，适合做基准或内嵌到自定义服务器。
    - 仓库：`https://github.com/h2o/picohttpparser`
- **适用场景**：
  - 需要完全掌控连接管理 / 缓冲策略，只想复用成熟的 HTTP 解析逻辑
  - 自研网关 / 反向代理 / 负载均衡器内核
  - 在已有 Asio / epoll / IOCP 框架中“拼装”自己的 HTTP 服务器

### 2.5 libev（事件循环库）

- **定位**：跨平台高性能事件循环库，封装 `epoll` / `kqueue` 等内核多路复用接口。
- **特点**：
  - 关注 IO 事件、定时器、信号等“事件源”，本身不限定协议和业务逻辑
  - 体积小、依赖少，适合做自研网络框架的底座
- **适用场景**：
  - 自己实现 TCP/UDP 协议栈、网关、代理、长连接 IM/推送服务
  - 需要对事件循环模型和性能调优有完全掌控
- **仓库**：`http://software.schmorp.de/pkg/libev.html`

---

## 3. RPC / 微服务通信

### 3.1 gRPC（首选）

- **定位**：基于 HTTP/2 + protobuf 的高性能跨语言 RPC 框架。
- **特点**：
  - IDL 使用 `.proto`，自动生成 C++ 代码
  - 支持 unary、server-streaming、client-streaming、双向 streaming
  - 内置负载均衡、拦截器、认证等机制（结合 Envoy 等网关更强）
- **适用场景**：
  - 微服务架构内部的服务间通信
  - C++ 与 Java/Go/Python 等多语言混合部署
- **仓库**：`https://github.com/grpc/grpc`

### 3.2 Thrift

- **定位**：Apache 出品的通用 RPC + 序列化框架。
- **特点**：
  - 多协议、多传输（支持二进制、压缩、HTTP 等）
  - 多语言支持成熟，适合老系统改造/集成
- **适用场景**：
  - 维护/迭代老的 Thrift 生态项目，或需要极高兼容性的多语言通信
- **仓库**：`https://github.com/apache/thrift`

### 3.3 ZeroMQ / nanomsg / nng

- **定位**：消息传递中间件 / 通信基元，而不是“完整 RPC 框架”。
- **特点**：
  - 提供 pub/sub、req/rep、push/pull 等通信模式
  - 需要自己定义消息格式、序列化方案（通常搭配 protobuf / FlatBuffers / JSON）
- **适用场景**：
  - 内部高吞吐消息分发、实时推送、日志收集等

---

## 4. 序列化 / 数据编码

### 4.1 JSON

- **nlohmann/json**（推荐）
  - 单头文件、API 像 `std::map` / `std::vector`，可直接 `json j; j["k"] = v;`
  - 适合配置、调试接口、轻量 HTTP API 的数据编码
  - 仓库：`https://github.com/nlohmann/json`

- **RapidJSON**
  - 强调性能的 JSON 解析与生成库
  - 有 DOM 和 SAX 两种风格接口，适合性能敏感路径
  - 仓库：`https://github.com/Tencent/rapidjson`

- **cJSON**
  - 经典的 C 语言 JSON 解析库，API 简单直接，容易在 C/C++ 混合项目中集成
  - 功能相对轻量，适合嵌入式、工具程序或对依赖体积敏感的场景
  - 仓库：`https://github.com/DaveGamble/cJSON`

### 4.2 二进制协议

- **Protocol Buffers（protobuf）**
  - 事实标准的跨语言二进制协议，gRPC 默认使用
  - 提供 `.proto` 描述文件，生成 C++ 类型与序列化代码
  - 仓库：`https://github.com/protocolbuffers/protobuf`

- **FlatBuffers / Cap’n Proto**
  - 追求“零拷贝”或少拷贝反序列化，适合游戏、实时系统、嵌入式

---

## 5. 日志 / 格式化 / 监控

### 5.1 日志

- **spdlog（推荐）**
  - 接口简单，支持同步/异步、多 sink（文件、控制台、syslog 等）
  - 性能好，和 `fmt` 深度集成
  - 仓库：`https://github.com/gabime/spdlog`

- **glog**
  - Google 出品的大型项目常用日志库
  - 与 gflags、gtest 等生态搭配良好
  - 仓库：`https://github.com/google/glog`

> 建议：后台服务优先选择 `spdlog`，配合 `fmt` 实现格式化。

### 5.2 格式化（fmt）

- **fmt**
  - C++ 安全的字符串格式化库，C++20 `std::format` 的前身
  - 支持自定义类型格式化，和 `spdlog` 配合使用非常自然
  - 仓库：`https://github.com/fmtlib/fmt`

### 5.3 Metrics / Profiling（简单提及）

- Prometheus C++ client（如 `prometheus-cpp`）：暴露指标给 Prometheus 抓取。
- Google Benchmark：写微基准测试定位性能瓶颈。
- LTTng / perf / VTune：更偏系统级性能分析工具。

---

## 6. 数据库 / 缓存

### 6.1 数据库访问

- **soci**
  - 面向 C++ 的数据库访问层，支持 PostgreSQL / MySQL / SQLite 等
  - 接口偏“ORM 风格但不完全 ORM”，对 SQL 仍有较好掌控
  - 仓库：`https://github.com/SOCI/soci`

- **libpqxx**
  - PostgreSQL 官方 C++ 客户端
  - 对 PG 特性支持完整
  - 仓库：`https://github.com/jtv/libpqxx`

- **MySQL Connector/C++**
  - MySQL 官方 C++ 驱动
  - 文档：`https://dev.mysql.com/doc/dev/connector-cpp/en`

- **SQLite + 封装库（如 sqlite_modern_cpp）**
  - 适合单机工具、嵌入式或轻量服务

- **ORM 方向**
  - **ODB**：较成熟的 C++ ORM，通过代码生成实现持久化
  - **Drogon ORM**：如使用 Drogon，可直接用其内置 ORM 减少重复工作

### 6.2 缓存 / KV 存储

- **Redis 客户端库**：
  - `redis-plus-plus`（sw::redis）：现代 C++17 Redis 客户端，封装 hiredis
  - `hiredis`：C 语言 Redis 客户端，更底层
- **适用场景**：
  - 分布式缓存、会话存储、限流计数、排行榜等

- **RocksDB / LevelDB**
  - 嵌入式 KV 存储，适合需要本地持久化、高写入吞吐的场景
  - 常用在服务内部做本地缓存、队列、状态存储

---

## 7. 配置管理 / 命令行解析

### 7.1 配置文件解析

- **YAML-CPP**
  - 解析 YAML 配置文件，适合较复杂的配置结构
  - 仓库：`https://github.com/jbeder/yaml-cpp`

- **toml++ / cpptoml**
  - 支持 TOML 配置格式，适合需要更强类型化配置的项目

- **JSON 方案**
  - 小型项目直接用 `nlohmann/json` 做配置即可。

### 7.2 命令行参数解析

- **CLI11**
  - 现代 C++ 风格命令行解析库，结构清晰
  - 仓库：`https://github.com/CLIUtils/CLI11`

- **cxxopts**
  - 轻量命令行解析库，接口简单

- **Boost.Program_options**
  - Boost 中的老牌方案，功能齐全但依赖 Boost。

---

## 8. 并发 / 线程池 / 任务调度

> C++11 之后标准库自带 `std::thread` / `std::mutex` / `std::future` 等基础并发原语，但在**后台服务器**里往往需要更高级的抽象（线程池、任务队列、actor 等）。

### 8.1 Intel oneTBB

- **定位**：任务并行库，原 Intel TBB。
- **特点**：
  - 基于任务的调度模型，而不是直接管理线程
  - 提供并行算法、并行容器、流水线等高层抽象
- **适用场景**：
  - CPU 密集型计算、数据处理服务
- **仓库**：`https://github.com/oneapi-src/oneTBB`

### 8.2 线程池 / 协程

- 常见选择：
  - 基于 **Boost.Asio** 自己封装线程池 + IO 服务
  - 使用开源线程池实现（如 `BS::thread_pool` 等轻量库）
  - C++20 之后可以考虑基于协程的库（如 `cppcoro` 等）

> 实战中，很多 Web 框架 / RPC 框架内部已经自带线程池和调度器，业务层一般只需要合理配置线程数和队列长度。

---

## 9. 测试 / Mock / Benchmark

### 9.1 单元测试

- **GoogleTest（gtest）**
  - 事实标准的 C++ 单元测试框架
  - 支持断言、夹具、参数化测试等
  - 仓库：`https://github.com/google/googletest`

- **Catch2 / doctest**
  - 单头文件、集成简单，适合中小型项目或库开发

### 9.2 Mock

- **GoogleMock（gmock）**
  - 与 gtest 深度整合，适合写接口 Mock、依赖注入场景

### 9.3 Benchmark

- **Google Benchmark**
  - 写微基准测试，量化接口/算法的性能变化
  - 仓库：`https://github.com/google/benchmark`

---

## 10. 构建 / 包管理（依赖获取方式）

> 虽然不属于“库”，但和依赖管理高度相关。

- **CMake**
  - 事实标准的 C/C++ 构建系统元配置工具
  - 新项目建议优先使用 CMake，配合 `FetchContent` / 外部包管理器

- **vcpkg**
  - 微软出品的 C/C++ 包管理工具
  - 与 VS / VSCode 集成好，Windows 平台体验优秀
  - 仓库：`https://github.com/microsoft/vcpkg`

- **Conan**
  - 更通用、更适合大规模项目的包管理器
  - 支持自建私有仓库、复杂依赖图
  - 仓库：`https://github.com/conan-io/conan`

---

## 11. 后台 C++ 项目常见选型组合（示例）

> 实际工程里一般会选一套“组合拳”，下面是几种典型方案，便于快速落地：

- **组合 A：典型 HTTP 微服务**
  - Web 框架：Drogon
  - 配置：YAML-CPP + YAML 文件
  - 序列化：JSON（nlohmann/json）+ protobuf（内部 RPC 使用）
  - 日志：spdlog + fmt
  - DB：PostgreSQL（libpqxx 或 Drogon ORM）
  - 缓存：Redis（redis-plus-plus）
  - 包管理：vcpkg / Conan

- **组合 B：高性能内部 RPC 服务**
  - RPC：gRPC + protobuf
  - 网络层：框架内部（gRPC 自带）或自定义 Beast + Asio
  - 配置：TOML / YAML
  - 日志：spdlog
  - Metrics：prometheus-cpp
  - Profiling：Google Benchmark + 系统工具

- **组合 C：老系统或多语言杂糅环境**
  - RPC：Thrift
  - DB：MySQL / Oracle 等传统数据库驱动
  - 日志：glog（与现有 C++/Java 生态统一）

---

## 12. 使用建议小结

- **先选体系，再挑库**：
  先决定整套架构偏 HTTP + JSON / RPC + protobuf / 自定义协议，再在每一层选对应的库。

- **首选有活跃维护的项目**：
  在 GitHub 看 star 只是参考，更重要的是：最近提交时间、issue 响应速度、CI 状态。

- **统一依赖来源**：
  尽量通过 vcpkg / Conan 管理依赖，避免手工拷源码导致版本失控。

- **及早搭好“骨架工程”**：
  在项目早期，先把 Web 框架、日志、配置、测试框架这些基础件一次性搭好，后续扩展会轻松很多。

[src: raw/ingested/2技术/cpp/第三方库-C++常用依赖库手册.md]

## Related Pages
- [[C++网络编程]]
- [[Boost.Asio]]
- [[C++多线程与并发]]
- [[C++项目深挖问题]]
- [[KeyDB存算分离项目]]
- [[分布式IM消息系统-雷漫]]
