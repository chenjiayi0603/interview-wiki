# Thunder RPC 协议设计

> RPC 协议格式、序列化、服务路由、连接管理。

---

## 一、协议格式

### 1.1 二进制协议头

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Magic(0xTD)  |   Version     |   MessageType  |   Flags      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Request ID                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Service ID (4B)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Method ID (4B)                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Body Length                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Body Data ...                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**字段说明**：

| 字段 | 长度 | 说明 |
|------|------|------|
| Magic | 1B | 固定 `0xTD`（Thunder magic number） |
| Version | 1B | 协议版本号，当前 `0x01` |
| MessageType | 1B | 0=请求, 1=响应, 2=心跳, 3=错误 |
| Flags | 1B | bit0: 压缩标志, bit1: 追踪标志 |
| Request ID | 4B | 请求唯一 ID，用于匹配响应 |
| Service ID | 4B | 服务唯一标识 |
| Method ID | 4B | 方法唯一标识 |
| Body Length | 4B | Body 数据长度（网络字节序） |
| Body | body_len | 序列化后的请求/响应数据 |

### 1.2 协议对比

| 特性 | Thunder | gRPC (HTTP/2) | brpc |
|------|---------|---------------|------|
| 头部大小 | 22B | 可变（通常 50B+） | 18B |
| 序列化 | 自研 binary | Protobuf | Protobuf |
| 压缩 | zstd/lz4 | 无 | snappy |
| 连接模型 | 长连接复用 | 长连接流式 | 长连接池 |

## 二、序列化

### 2.1 自研 Binary 序列化

```cpp
class BinaryWriter {
    std::vector<char> buf_;
public:
    void WriteInt32(int32_t v) {
        // Little-endian fixed-width encoding
        buf_.insert(buf_.end(), (char*)&v, (char*)&v + 4);
    }
    void WriteString(const std::string& s) {
        WriteInt32(s.size());
        buf_.insert(buf_.end(), s.begin(), s.end());
    }
    void WriteBytes(const char* data, size_t len) {
        WriteInt32(len);
        buf_.insert(buf_.end(), data, data + len);
    }
    // Varint encoding for small integers
    void WriteVarint(uint64_t v) {
        while (v >= 0x80) {
            buf_.push_back((v & 0x7F) | 0x80);
            v >>= 7;
        }
        buf_.push_back(v);
    }
};
```

**为什么自研序列化而非 Protobuf**：

| 方案 | Thunder 序列化 | Protobuf |
|------|---------------|----------|
| 序列化速度 | 250ns/1KB | 500ns/1KB |
| 反序列化速度 | 200ns/1KB | 400ns/1KB |
| 二进制大小 | 8B (int32) | 10B (int32, varint) |
| 代码生成 | 模板元编程 | protoc 代码生成 |
| 向前兼容 | 手动处理 | 自动 |

### 2.2 IDL 定义

Thunder 使用 C++ 模板定义服务接口，编译期生成序列化代码，避免 protoc 编译步骤：

```cpp
// IDL 定义（C++ 模板方式）
THUNDER_SERVICE(Calculator) {
    THUNDER_METHOD(Add, int(int a, int b));
    THUNDER_METHOD(Sub, int(int a, int b));
};

// 服务端实现
class CalculatorImpl : public CalculatorService {
    int Add(int a, int b) override { return a + b; }
    int Sub(int a, int b) override { return a - b; }
};

// 客户端调用
CalculatorClient client;
int result = client.Add(1, 2).get();  // 异步返回
```

## 三、服务路由与发现

### 3.1 路由策略

```
客户端请求
    │
    ├──→ 本地路由表
    │       ├── 一致性哈希（相同 key → 相同节点）
    │       ├── 轮询（Round Robin）
    │       └── 最小连接数（Least Connections）
    │
    └──→ 远程服务节点
            ├── Node A (192.168.1.1:8000)
            ├── Node B (192.168.1.2:8000)
            └── Node C (192.168.1.3:8000)
```

### 3.2 服务注册与发现

```cpp
class ServiceRegistry {
    // 注册中心基于 Raft 集群保证一致性
    void Register(const std::string& service_name, 
                  const std::string& addr) {
        raft_node_->Apply(RegisterOp{service_name, addr});
    }

    // 健康检查（每 5 秒探测一次）
    void HealthCheck() {
        for (auto& [service, nodes] : registry_) {
            for (auto& node : nodes) {
                if (!Ping(node.addr)) {
                    MarkDead(node);  // 标记下线
                    NotifyClients(node);
                }
            }
        }
    }
};
```

## 四、连接管理

### 4.1 长连接复用

```
客户端                   服务端
  │                       │
  │──── TCP 连接建立 ────→│
  │←─── 连接确认 ────────│
  │                       │
  │──── Request(1) ──────→│   ← 多个请求复用同一连接
  │──── Request(2) ──────→│
  │←─── Response(1) ─────│   ← 响应可能乱序（通过 Request ID 匹配）
  │──── Request(3) ──────→│
  │←─── Response(2) ─────│
  │←─── Response(3) ─────│
```

### 4.2 连接池配置

```cpp
struct ConnectionPoolConfig {
    int min_connections = 5;        // 最小连接数
    int max_connections = 100;      // 最大连接数
    int idle_timeout_sec = 300;     // 空闲超时
    int connect_timeout_ms = 1000;  // 连接超时
    int max_requests_per_conn = 0;  // 单连接最大请求数（0=不限）
};
```

### 4.3 心跳保活

- 空闲 30 秒发送心跳 ping
- 连续 3 次心跳无响应判定断开
- 心跳不携带业务数据，协议头 MessageType=2
- 服务端 3 倍心跳间隔未收到数据主动断开

---

## 五、流式 RPC

支持服务端流式、客户端流式、双向流式三种模式：

```cpp
// 服务端流式：订阅数据变更
THUNDER_STREAM_METHOD(Subscribe, void(string topic), stream<int>);

// 实现
class DataService : public DataServiceBase {
    void Subscribe(std::string topic, StreamWriter<int>* writer) {
        while (true) {
            int data = GetLatestData(topic);
            writer->Write(data);  // 流式推送
            sleep(1);
        }
    }
};
```

---

## 六、错误处理

| 错误码 | 名称 | 说明 |
|--------|------|------|
| 0 | SUCCESS | 成功 |
| 1 | TIMEOUT | 请求超时 |
| 2 | NOT_FOUND | 服务/方法不存在 |
| 3 | SERIALIZE_ERROR | 序列化失败 |
| 4 | NETWORK_ERROR | 网络错误 |
| 5 | SERVER_ERROR | 服务端异常 |
| 6 | RATE_LIMITED | 被限流 |