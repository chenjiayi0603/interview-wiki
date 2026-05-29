# 分布式IM消息系统-雷漫

See also: [[C++项目深挖问题]]

## 项目概述

**一句话介绍**：自研C++分布式IM系统，支撑社交APP的即时通讯。

**项目信息**：
- 公司：深圳雷漫网络科技有限公司
- 时间：2019.11 - 2021.4
- 产品：社交APP即时通讯模块
- 技术栈：C++ + Libev + Nacos + RocketMQ

[src: raw/ingested/3项目/分布式IM-雷漫/面试故事-雷漫IM.md]

## 技术架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              客户端层                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │  iOS     │  │  Android │  │  Web     │  │  小程序   │               │
│  │  (OC)    │  │  (Java)  │  │  (JS)    │  │  (原生)   │               │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                              TCP/WebSocket
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           连接接入层                                     │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                    Connection Server                            │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │    │
│  │  │  Libev事件循环 │  │  协议解析     │  │  连接管理     │         │    │
│  │  │  (单线程epoll)│  │  (自定义二进制)│  │  (心跳/超时)  │         │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │    │
│  │                                                                 │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │    │
│  │  │  加解密       │  │  压缩解压     │  │  限流熔断     │         │    │
│  │  │  (AES/RSA)   │  │  (Snappy)    │  │  (令牌桶)     │         │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  部署：8台服务器，每台支持10万+连接，总计100万+连接                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                              内部RPC (gRPC)
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                             服务层                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │  消息服务    │  │  用户服务    │  │  群组服务    │  │  推送服务    │   │
│  │             │  │             │  │             │  │             │   │
│  │  单聊/群聊   │  │  在线状态    │  │  群管理      │  │  离线推送    │   │
│  │  消息存储    │  │  好友关系    │  │  群成员      │  │  APNs/FCM   │   │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │
│  │  Nacos      │  │  Cat        │  │  Sentinel   │                    │
│  │  服务发现    │  │  调用链监控  │  │  流量控制    │                    │
│  └─────────────┘  └─────────────┘  └─────────────┘                    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                             存储层                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │  MySQL      │  │  Redis      │  │  RocketMQ   │  │  MongoDB    │   │
│  │  分库分表    │  │  Codis集群   │  │  消息队列    │  │  历史消息    │   │
│  │  (消息持久化) │  │  (在线状态)  │  │  削峰填谷    │  │  (冷数据)    │   │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

[src: raw/ingested/3项目/分布式IM-雷漫/面试故事-雷漫IM.md]

## 连接接入层详解

### Libev事件循环

```cpp
class ConnectionServer {
public:
    static const int MAX_EVENTS = 10240;
    static const int MAX_CONNECTIONS = 150000;
    static const int HEARTBEAT_TIMEOUT = 90;  // 90秒
    
    void Start(int port) {
        // 创建监听socket
        listen_fd_ = socket(AF_INET, SOCK_STREAM, 0);
        SetNonBlocking(listen_fd_);
        
        // 绑定端口
        struct sockaddr_in addr;
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port);
        addr.sin_addr.s_addr = INADDR_ANY;
        bind(listen_fd_, (struct sockaddr*)&addr, sizeof(addr));
        listen(listen_fd_, SOMAXCONN);
        
        // 创建Libev事件循环
        loop_ = ev_loop_new(EVFLAG_AUTO);
        
        // 注册监听事件
        ev_io_init(&listen_watcher_, OnAccept, listen_fd_, EV_READ);
        ev_io_start(loop_, &listen_watcher_);
        
        // 注册定时器（心跳检查）
        ev_timer_init(&heartbeat_timer_, OnHeartbeatCheck, 10.0, 10.0);
        ev_timer_start(loop_, &heartbeat_timer_);
        
        // 启动事件循环
        ev_run(loop_, 0);
    }
    
private:
    // 新连接处理
    static void OnAccept(EV_P_ ev_io* w, int revents) {
        auto* server = (ConnectionServer*)w->data;
        
        while (true) {
            struct sockaddr_in client_addr;
            socklen_t addr_len = sizeof(client_addr);
            
            int client_fd = accept(server->listen_fd_, 
                                   (struct sockaddr*)&client_addr, 
                                   &addr_len);
            if (client_fd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    break;  // 没有更多连接
                }
                continue;
            }
            
            // 连接数检查
            if (server->connection_count_ >= MAX_CONNECTIONS) {
                close(client_fd);
                continue;
            }
            
            // 设置非阻塞
            SetNonBlocking(client_fd);
            
            // 创建连接对象
            Connection* conn = server->AllocConnection();
            conn->fd = client_fd;
            conn->last_active = time(nullptr);
            conn->user_id = 0;  // 未登录
            
            // 注册读事件
            ev_io_init(&conn->read_watcher, OnRead, client_fd, EV_READ);
            conn->read_watcher.data = conn;
            ev_io_start(server->loop_, &conn->read_watcher);
            
            server->connection_count_++;
        }
    }
    
    // 数据读取
    static void OnRead(EV_P_ ev_io* w, int revents) {
        Connection* conn = (Connection*)w->data;
        
        while (true) {
            int n = read(conn->fd, conn->read_buf + conn->read_len, 
                         sizeof(conn->read_buf) - conn->read_len);
            
            if (n < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    break;  // 数据读完
                }
                // 错误，关闭连接
                CloseConnection(conn);
                return;
            }
            
            if (n == 0) {
                // 对端关闭
                CloseConnection(conn);
                return;
            }
            
            conn->read_len += n;
            conn->last_active = time(nullptr);
            
            // 尝试解析消息
            while (conn->read_len >= 4) {
                uint32_t msg_len = ntohl(*(uint32_t*)conn->read_buf);
                if (conn->read_len < 4 + msg_len) {
                    break;  // 数据不完整
                }
                
                // 处理消息
                ProcessMessage(conn, conn->read_buf + 4, msg_len);
                
                // 移动剩余数据
                memmove(conn->read_buf, conn->read_buf + 4 + msg_len, 
                        conn->read_len - 4 - msg_len);
                conn->read_len -= 4 + msg_len;
            }
        }
    }
    
    // 心跳检查
    static void OnHeartbeatCheck(EV_P_ ev_timer* w, int revents) {
        auto* server = (ConnectionServer*)w->data;
        time_t now = time(nullptr);
        
        for (auto& conn : server->connections_) {
            if (conn && conn->fd > 0) {
                if (now - conn->last_active > HEARTBEAT_TIMEOUT) {
                    // 超时，关闭连接
                    CloseConnection(conn);
                }
            }
        }
    }
    
    int listen_fd_;
    struct ev_loop* loop_;
    ev_io listen_watcher_;
    ev_timer heartbeat_timer_;
    std::vector<Connection*> connections_;
    int connection_count_ = 0;
};
```

[src: raw/ingested/3项目/分布式IM-雷漫/面试故事-雷漫IM.md]

## 消息可靠性保证

### 消息发送流程

```
发送方                    服务端                      接收方
   │                        │                          │
   │  1. 发送消息请求         │                          │
   │  (msgid, content, to)   │                          │
   │───────────────────────►│                          │
   │                        │                          │
   │                 ┌──────┴──────┐                   │
   │                 │ 2. 写本地消息表 │                  │
   │                 │    (pending)  │                  │
   │                 └──────┬──────┘                   │
   │                        │                          │
   │                 ┌──────┴──────┐                   │
   │                 │ 3. 写消息队列  │                  │
   │                 │   (RocketMQ) │                  │
   │                 └──────┬──────┘                   │
   │                        │                          │
   │  4. 返回ACK (msgid)     │                          │
   │◄───────────────────────│                          │
   │                        │                          │
   │                        │  5. 推送消息              │
   │                        │  (查询接收方连接)          │
   │                        │─────────────────────────►│
   │                        │                          │
   │                        │  6. 返回ACK (msgid)       │
   │                        │◄─────────────────────────│
   │                        │                          │
   │                 ┌──────┴──────┐                   │
   │                 │ 7. 更新消息状态 │                 │
   │                 │    (delivered)│                 │
   │                 └─────────────┘                   │
```

[src: raw/ingested/3项目/分布式IM-雷漫/面试故事-雷漫IM.md]

### 消息表设计

```sql
-- 消息表（按发送方ID分表）
CREATE TABLE message_{shard_id} (
    msg_id BIGINT PRIMARY KEY,        -- 消息ID（雪花算法生成）
    from_uid BIGINT NOT NULL,         -- 发送方ID
    to_uid BIGINT NOT NULL,           -- 接收方ID（或群ID）
    msg_type TINYINT NOT NULL,        -- 消息类型(1单聊 2群聊)
    content VARBINARY(4096),          -- 消息内容（加密压缩）
    status TINYINT DEFAULT 0,         -- 状态(0待发送 1已发送 2已送达 3已读)
    create_time DATETIME NOT NULL,
    update_time DATETIME,
    INDEX idx_to_uid (to_uid, create_time),
    INDEX idx_status (status, create_time)
) ENGINE=InnoDB;

-- 消息ID发号器
CREATE TABLE msg_id_generator (
    stub CHAR(1) PRIMARY KEY
) ENGINE=MyISAM;

-- 获取下一个消息ID
REPLACE INTO msg_id_generator (stub) VALUES ('1');
SELECT LAST_INSERT_ID();
```

### 分片策略

- 用户ID % 32 分到32个消息库
- 每个库64张消息表
- 按月份自动创建新表

[src: raw/ingested/3项目/分布式IM-雷漫/面试故事-雷漫IM.md]

## 性能测试数据

### 连接能力测试

| 指标 | 数值 | 测试条件 |
|------|------|---------|
| 单机连接数 | 1,000,000 | 8核16GB，空闲心跳 |
| 每连接内存 | 8KB | 含读缓冲1KB |
| 连接建立速度 | 5000/秒 | 新连接建立速率 |
| 心跳处理能力 | 10万/秒 | 心跳包处理 |
| CPU占用（空闲） | 5-10% | 100万连接心跳处理 |

**内存计算**：
```
每连接开销：
- Connection对象：2KB
- 读缓冲：1KB (tcp_rmem min=1KB)
- 写缓冲：1KB (tcp_wmem min=1KB)
- ev_io结构：1KB
- 内核元数据：~3KB (socket + conntrack)
合计：~8KB

100万连接 = 8KB × 1,000,000 = 8GB
实际占用约8GB（调优后）
```

### 消息吞吐测试

| 场景 | 吞吐量 | P99延迟 | 说明 |
|------|--------|--------|------|
| 单聊消息 | 50,000条/秒 | 30ms | 点对点推送 |
| 群聊消息（100人群） | 5,000条/秒 | 50ms | 广播开销大 |
| 群聊消息（1000人群） | 500条/秒 | 200ms | 需要优化 |
| 离线消息拉取 | 20,000条/秒 | 20ms | 批量拉取 |

**群聊瓶颈分析**：
```
1000人群聊，发一条消息：
- 写消息表：1次DB操作
- 推送给999人：999次推送
- 每推送需要：查连接位置 + 发送

优化方案：
1. 本地聚合：同一连接服务器的用户批量推送
2. 异步推送：写消息队列，异步消费推送
3. 推送降级：大群消息只推在线，离线靠拉取
```

### 消息延迟分解

| 阶段 | 延迟 | 占比 |
|------|------|------|
| 网络传输（客户端→服务端） | 2ms | 10% |
| 协议解析+解密 | 1ms | 5% |
| 消息路由 | 2ms | 10% |
| 写数据库 | 5ms | 25% |
| 写消息队列 | 2ms | 10% |
| 推送路由 | 3ms | 15% |
| 网络推送 | 5ms | 25% |
| **合计** | **20ms** | **100%** |

**优化方向**：
- 数据库：从主库写改成异步写从库，降5ms
- 消息队列：批量提交，降1ms
- 推送路由：本地缓存连接映射，降1ms

### 可用性数据

| 指标 | 数值 | 说明 |
|------|------|------|
| 可用性 | 99.99% | 年停机<53分钟 |
| 故障恢复时间 | < 30s | 单节点故障 |
| 数据零丢失 | 是 | WAL保证 |
| 消息重复率 | < 0.01% | 客户端去重 |

**99.99%可用性计算**：
```
全年时间：365 × 24 × 60 = 525,600分钟
允许停机：525,600 × 0.01% = 52.56分钟

实际停机事件：
- 计划内维护：2次 × 10分钟 = 20分钟
- 故障恢复：4次 × 5分钟 = 20分钟
- 合计：40分钟 < 52.56分钟
```

[src: raw/ingested/3项目/分布式IM-雷漫/面试故事-雷漫IM.md]

## 压测实录与瓶颈分析

### 群聊压测结论

群聊发送 (cmd = 4501) 消息QPS大约为8000，有小概率失败（小于1%）。绝大部分是Group节点在等待mongo插入群聊步骤时超时失败。

**压测环境（8000 QPS）**：
1. 压测客户端 1个 虚拟机 8进程
2. ACC 2个 虚拟机
3. GROUP 3个 虚拟机
4. MONGOAGENT 2个 独立虚拟机，mongo集群(8实例、4分片、2代理) 8个 独立虚拟机
5. REDISAGENT 2个 独立虚拟机，codis集群(4实例、2代理) 6个 独立虚拟机

**压测环境（10000 QPS）**：
1. 压测客户端 1个 虚拟机 11进程
2. ACC 3个 虚拟机，router 3个 虚拟机
3. GROUP 5个 虚拟机
4. MONGOAGENT 4个 虚拟机，mongo集群(8实例、4分片、3代理、1 config) 11个 独立虚拟机
5. REDISAGENT 3个 虚拟机，codis集群(6实例、2代理) 6个 独立虚拟机

合计：不重用时约 36 台，考虑重用时约 29～36 台。

### 瓶颈分析

- **Mongo集群**：高并发请求时，时延较多。客户端QPS越多，时延明显上升；在超负荷情况下，随时间往后，时延也上升。
- **增加节点无效**：Mongo集群在增加一个mongod、一个mongos后，依然存在时延明显。增加逻辑节点未能增加群聊消息QPS。
- **混合场景压力**：在登录和群聊同时发送混合场景，会产生突增的codis访问压力（偶尔瞬间codis访问几十万QPS）而产生redisagent访问codis失败。

### 优化方案

1. mongo集群需要考虑数据量大时的拓展方案，后期需要mongo集群支持拆分（如群消息表和状态表分集群）
2. mongo集群优化写入参数
3. mongo机器硬件配置升级（升级磁盘为磁盘阵列，升级内存到32G，目前mongod内存使用达到90%）。每个mongo节点使用独占虚拟机（mongo config和mongod有的混用虚拟机）
4. mongoagent读写分离
5. 状态数据持久化尽量使用redis来实现持久化，减少访问状态数据时访问mongo。（session_status 访问最后消息id、session_msgseq生成seq）

### Mongo群库压测

10进程client，mongo集合msg_c2g_testing分片2片，数据量6亿+。分片键1个groupId，索引3个（id、groupId、groupId和msgId）。Mognostat 77（5000）78（5000）总共约QPS 10000。

慢日志：插入偶尔超过7秒。

**资源使用**：
- Mongos 77/78：CPU 200%，内存较低，IO一般较低
- Mongod 114-117：CPU偶尔较高400-500%，内存55-60%，IO偶尔较高（100%），使用sdc盘

**读写混合QPS估算**：在读走SECONDARY、业务读请求不过重的前提下，读写混合总QPS约15000～25000（写约10000 + 读约5000～15000）。若mongos或某分片PRIMARY/SECONDARY先打满，需加mongos或扩容分片后再测。

### 网关单机百万连接

性能测试包括：单场景压测（登录、群消息、网关、mongodb写QPS；网关单机百万连接），瓶颈分析。

### Wasm自适应熔断

protection是指请求数小于等于protection则不会触发熔断，默认5；k是熔断敏感系数，在1.1到2之间，默认1.5，越小越容易熔断。

[src: raw/ingested/3项目/分布式IM-雷漫/面试故事-雷漫IM.md]

## 面试故事与问答

### 故事一：单机百万连接

**面试官问**：单机百万连接是怎么做到的？

**回答模板**：

> "IM系统的核心挑战是长连接管理。我们用C++ + Libev实现：
> 
> **技术要点**：
> 1. **epoll多路复用**：单线程管理大量连接
> 2. **非阻塞I/O**：避免连接阻塞影响其他连接
> 3. **连接分层**：按用户ID哈希分到不同连接池
> 4. **心跳保活**：客户端每30秒发心跳，超时断开
> 
> **优化手段**：
> - 调大文件描述符限制（ulimit -n 1000000）
> - 内核参数调优（TCP缓冲区、连接队列）
> - 内存池复用，减少分配开销
> 
> **测试数据**：
> - 单机支持100万连接
> - 内存占用约16GB（每个连接约13KB）
> - CPU占用约30%"

### 故事二：消息可靠性保证

**面试官问**：怎么保证消息不丢不重？

**回答模板**：

> "IM系统的核心是消息可靠，我们做了多层保障：
> 
> **发送流程**：
> 1. 发送方先写本地消息表
> 2. 发送到服务端，服务端写消息库
> 3. 服务端返回ACK，发送方删除本地消息
> 4. 如果超时未收到ACK，重试发送
> 
> **接收流程**：
> 1. 服务端推送消息，携带消息ID
> 2. 接收方返回ACK
> 3. 如果未收到ACK，服务端重推
> 4. 接收方去重：用消息ID判断是否已收到
> 
> **离线消息**：
> - 用户离线时，消息存入离线表
> - 上线后，拉取离线消息
> - 拉取完成后删除
> 
> **效果**：
> - 消息不丢：WAL + 重试机制
> - 消息不重：客户端去重
> - 消息延迟 < 50ms"

### 常问问题

**Q: 为什么用C++而不是Go？**

> 2019年做的项目，当时团队技术栈是C++。另外C++在内存控制和性能上有优势，单机百万连接用Go也能做，但内存占用会更高。

**Q: 消息顺序怎么保证？**

> 单聊：按发送方ID哈希到同一队列，保证顺序。群聊：按群ID哈希，同一群的消息走同一队列。

**Q: 可用性99.99%怎么算的？**

> 一年停机时间约52分钟。我们通过多机房部署、故障自动切换实现。

[src: raw/ingested/3项目/分布式IM-雷漫/面试故事-雷漫IM.md]

## Related Pages
- [[C++项目深挖问题]]
- [[TCP协议]]
- [[C++网络编程]]
- [[性能优化]]
- [[Linux_IO]]
- [[服务网格平台-即构]]
- [[线程同步]]
