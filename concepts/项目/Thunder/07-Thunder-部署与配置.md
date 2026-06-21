# Thunder 部署与配置

> 编译构建、配置说明、部署架构、监控告警。

---

## 一、编译构建

### 1.1 依赖

| 依赖 | 版本 | 说明 |
|------|------|------|
| CMake | >= 3.20 | 构建系统 |
| GCC / Clang | >= 11 / >= 14 | C++20 编译器 |
| Linux Kernel | >= 5.1 | io_uring 支持 |
| fmtlib | >= 8.0 | 格式化 |
| spdlog | >= 1.10 | 日志 |
| gtest | >= 1.11 | 单元测试 |

### 1.2 构建步骤

```bash
# 克隆仓库
git clone https://github.com/tommychen/thunder.git
cd thunder

# 配置
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release \
         -DTHUNDER_WITH_IOURING=ON \
         -DTHUNDER_WITH_EXAMPLES=ON

# 编译
make -j$(nproc)

# 运行测试
make test
```

### 1.3 CMake 选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| THUNDER_WITH_IOURING | ON | 启用 io_uring 支持 |
| THUNDER_WITH_EXAMPLES | ON | 编译示例 |
| THUNDER_WITH_BENCHMARK | ON | 编译基准测试 |
| THUNDER_ENABLE_ASAN | OFF | 启用 AddressSanitizer |
| THUNDER_ENABLE_COVERAGE | OFF | 启用覆盖率检测 |

## 二、配置说明

### 2.1 配置文件格式 (YAML)

```yaml
# thunder.yaml
server:
  name: "thunder-node-1"
  address: "0.0.0.0:8000"
  workers: 8                  # Worker 线程数
  max_connections: 500000     # 最大连接数
  io_model: "io_uring"        # epoll / io_uring

rpc:
  request_timeout_ms: 3000    # RPC 请求超时
  max_payload_size: 16777216  # 最大消息体 (16MB)
  compression: "zstd"         # 压缩算法 none/lz4/zstd
  serialize: "binary"         # 序列化方式

raft:
  enabled: true
  peers:
    - "192.168.1.1:8000"
    - "192.168.1.2:8000"
    - "192.168.1.3:8000"
  heartbeat_interval_ms: 50
  election_timeout_ms: 150
  log_dir: "/var/log/thunder/raft"

log:
  level: "info"               # trace/debug/info/warn/error
  output: "console+file"
  file: "/var/log/thunder/thunder.log"
  max_size_mb: 100
  max_files: 10
```

## 三、部署架构

### 3.1 单机部署

```
┌──────────────────────────────────────┐
│             Thunder Node             │
│  ┌────────────────────────────────┐  │
│  │  Manager 父进程               │  │
│  │  ├── Config Loader            │  │
│  │  ├── Service Registry         │  │
│  │  └── Raft 集群管理            │  │
│  ├────────────────────────────────┤  │
│  │  Worker 0 (CPU 0)   epoll     │  │
│  │  Worker 1 (CPU 1)   epoll     │  │
│  │  ...                          │  │
│  │  Worker 7 (CPU 7)   epoll     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### 3.2 集群部署 (3 节点 Raft)

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Node A   │─────│ Node B   │─────│ Node C   │
│ 10.0.0.1 │     │ 10.0.0.2 │     │ 10.0.0.3 │
│ :8000    │     │ :8000    │     │ :8000    │
└──────────┘     └──────────┘     └──────────┘
       │              │                 │
       └──────────────┼─────────────────┘
                      │
              ┌───────┴───────┐
              │  Load Balancer│
              │   (HAProxy)   │
              └───────────────┘
                      │
              ┌───────┴───────┐
              │    Client     │
              └───────────────┘
```

### 3.3 多数据中心

```
数据中心 A (主)          数据中心 B (从)
┌──────────┐             ┌──────────┐
│ Raft     │──── WAN ───→│ Raft     │
│ Leader   │             │ Follower │
└──────────┘             └──────────┘
     │                        │
 客户端                    客户端
 (读写)                   (只读)
```

数据中心之间延迟容忍：< 50ms（Raft 需要低延迟网络）

## 四、系统调优

### 4.1 操作系统参数

```bash
# /etc/sysctl.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535

# 文件描述符上限
ulimit -n 1048576

# 内存
vm.max_map_count = 262144
kernel.mm_transparent_hugepage = never  # 禁用 THP
```

### 4.2 CPU 调优

```bash
# 关闭超线程（延迟敏感场景）
echo off > /sys/devices/system/cpu/smt/control

# 设置 CPU 节能策略为 performance
cpupower frequency-set -g performance

# 中断亲和性（绑定到指定 CPU）
echo 1 > /proc/irq/IRQ_NUMBER/smp_affinity
```

## 五、监控告警

### 5.1 Prometheus 指标

| 指标 | 类型 | 说明 |
|------|------|------|
| thunder_connections_total | Gauge | 当前连接数 |
| thunder_rpc_qps | Counter | RPC 请求速率 |
| thunder_rpc_latency_ms | Histogram | RPC 延迟分布 |
| thunder_raft_term | Gauge | Raft 当前任期 |
| thunder_raft_leader | Gauge | 当前 Leader 节点 ID |
| thunder_raft_log_count | Gauge | Raft 日志条目数 |
| thunder_coroutine_count | Gauge | 活跃协程数 |

### 5.2 告警规则

```yaml
# prometheus-alerts.yml
groups:
  - name: thunder
    rules:
      - alert: RaftNoLeader
        expr: thunder_raft_leader == 0
        for: 10s
        labels: { severity: critical }

      - alert: HighLatency
        expr: thunder_rpc_latency_ms{p99} > 100
        for: 1m
        labels: { severity: warning }

      - alert: ConnectionDrop
        expr: rate(thunder_connections_total[1m]) < -100
        for: 30s
        labels: { severity: warning }
```

### 5.3 日志格式

```json
{"time":"2025-01-15T10:30:00.123Z","level":"INFO","thread":3,
 "coroutine":1024,"msg":"RPC request handled",
 "service":"Calculator","method":"Add","latency_ms":0.15}
```
