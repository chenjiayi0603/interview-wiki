# MySQL 高可用与集群

> MHA 高可用、分库分表、读写分离、QPS 估算、微信级集群设计、冷热分离。

---

## 一、分库分表

### 1.1 什么时候需要分库分表？

1. **数据量大**：单表超过 3000~5000 万行，性能明显下降
2. **并发高**：单库 QPS 达到瓶颈，经调优已无增长空间
3. **存储空间**：单库存储不足

### 1.2 分表策略

| 策略 | 方法 | 优点 | 缺点 |
|------|------|------|------|
| **范围分表** | 按时间/ID 范围 | 扩展简单，查询明确 | 可能数据倾斜 |
| **哈希分表** | 按分片键取模 | 数据均匀 | 范围查询困难 |
| **一致性哈希** | 环状分布 | 扩容影响小 | 实现复杂 |

```sql
-- 分片路由规则
shard_id = user_id % SHARD_COUNT  -- 32/64/128
```

### 1.3 分库分表带来的问题

| 问题 | 解决方案 |
|------|---------|
| 跨库/跨表查询 | 避免在应用层聚合，或用中间件 |
| 分布式事务 | Seata、消息队列最终一致性 |
| 全局主键 | 雪花算法（Snowflake）、UUID |
| 数据迁移 | 双写方案、数据同步工具 |

### 1.4 设计原则

- **不要过早分库分表**：先索引优化、读写分离
- **选择高区分度的分片键**：user_id、order_id 等
- **预留扩容空间**：分片数取 2 的幂（32/64/128）
- **跨分片查询尽量规避**：多数查询带分片键

---

## 二、读写分离

### 2.1 架构

```
应用层
  │
  ├── 写路由 ──► 主库（Master）
  │                  │
  └── 读路由 ──► 从库1（Slave）
                ──► 从库2（Slave）
                ──► 从库3（Slave）
```

### 2.2 读写分离实现

```cpp
// 应用层自路由示例（C++）
DBConnection* get_connection(OpType op, int shard_id) {
    if (op == WRITE)
        return conn_pool_[shard_id].get_master();
    else
        return conn_pool_[shard_id].get_slave();  // 轮询/随机
}
```

### 2.3 主从延迟问题

- **原因**：从库 SQL 线程单线程重放（可用并行复制缓解）
- **监控**：`Seconds_Behind_Master`
- **解决**：并行复制、半同步、读写分离时加延迟判断

---

## 三、MHA 高可用

### 3.1 MHA 概述

**MHA**（Master High Availability Manager）是 DeNA 开发的 MySQL 高可用方案，用于自动化主从故障转移。

| 特性 | 说明 |
|------|------|
| 核心功能 | 主库宕机时自动选举新主、完成 failover |
| 数据保护 | 尽量从旧主库拉取未同步的 binlog，减少数据丢失 |
| 应用透明 | 配合 VIP/DNS 切换，应用无感知 |

### 3.2 架构角色

| 角色 | 说明 |
|------|------|
| **MHA Manager** | 监控主从、执行 failover（独立节点或与从库同机） |
| **MHA Node** | 所有 MySQL 节点，执行本地命令 |
| **Master** | 主库，提供写服务 |
| **Slave** | 从库，至少 2 台（1 台候选新主） |

### 3.3 Failover 流程

```
1. Manager 检测主库心跳不可达
2. 选举 binlog 最完整的 Slave 为新主
3. 从宕机主库（若可访问）拉取缺失 binlog
4. 提升新主（STOP SLAVE，应用差异日志）
5. 其他从库 CHANGE MASTER TO 指向新主
6. VIP 漂移到新主（可选）
```

### 3.4 快速部署

```bash
# 所有节点安装 MHA Node
yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch \
  perl-Parallel-ForkManager perl-Time-HiRes
# 安装 MHA Node + Manager...

# Manager 节点配置
# /etc/mha/app1.cnf
[server default]
user=mha
password=xxx
ssh_user=root
manager_workdir=/var/log/mha/app1

[server1]
hostname=192.168.1.101
candidate_master=1
[server2]
hostname=192.168.1.102
candidate_master=1
[server3]
hostname=192.168.1.103
candidate_master=1

# 启动
nohup masterha_manager --conf=/etc/mha/app1.cnf &
```

---

## 四、QPS 估算与容量规划

### 4.1 单机能力基准

| 级别 | 说明 |
|:----:|------|
| **5,000 QPS** | 经验安全线：4C8G~8C16G + NVMe SSD，常规业务混合查询 |
| **2 万 QPS** | 极限优化：点查为主、高缓存命中、硬件顶配、极致参数调优 |
| **10 万+ QPS** | 集群分片：分片数 × 单分片能力 |

### 4.2 架构与 QPS 对应关系

| 架构 | 写上限 | 读上限 | 适用场景 |
|------|:------:|:------:|---------|
| 单机 | ~5000 TPS | ~5000 QPS | 中小型应用 |
| 1 主 1 从 | ~5000 TPS | ~10000 QPS | 小规模读写分离 |
| 1 主 3 从 | ~5000 TPS | ~20000 QPS | 中等规模 |
| 32 分片 × 1 主 2 从 | ~16 万 TPS | ~48 万 QPS | 高并发大规模 |

**核心制约因素：**
- 写瓶颈：主库 redo log 单线程顺序写盘（IOPS 极限）
- 读瓶颈：可通过加从库线性扩展
- 分库分表：写能力也变成可扩展（多个主库各自写入）

### 4.3 硬件清单参考

**中等并发（~2 万 QPS）：1 主 3 从**

| 设备 | 配置 | 数量 |
|------|------|:----:|
| MySQL 主库 | 8C16G、NVMe SSD 200GB+ | 1 |
| MySQL 从库 | 8C16G、NVMe SSD 200GB+ | 3 |
| Redis（可选） | 4C8G | 2 |
| **年成本** | 云服务器 | **~1.2 万/年** |

**高并发（数十万 QPS）：32 分片 × 1 主 2 从**

| 设备 | 配置 | 数量 |
|------|------|:----:|
| MySQL 主库 | 8C16G、NVMe SSD 300GB+ | 32 |
| MySQL 从库 | 8C16G、NVMe SSD 300GB+ | 64 |
| Redis 集群 | 8C16G | 6 |
| **年成本** | 云服务器 | **~30 万/年** |

### 4.4 常见性能排查

1. **慢 SQL**：开启慢查询 → EXPLAIN 分析 → 索引/SQL 改写
2. **连接数过多**：检查连接池、慢查询堆积、僵尸连接
3. **主从延迟**：大事务、binlog 量大、从库单线程回放 → 并行复制
4. **锁等待**：`SHOW ENGINE INNODB STATUS` → 优化事务粒度与索引

---

## 五、微信级集群设计

### 5.1 架构特点

```
┌─────────────────────────────────────────────┐
│        C++ 业务服务集群（多实例）               │
│  StorageAccess（存储访问层）                   │
│  ├── ShardRouter（分片路由）user_id % N       │
│  ├── ReadWriteRouter（读写分离）               │
│  └── ConnPool[]（每分片独立主/从池）           │
└──────────────┬──────────────────────────────┘
               │ 直连（无中间件）
         ┌─────┴─────┐
         │  Redis     │ 拦截 80%+ 读请求
         │  MySQL     │ 分片 0..N: 1主2从
         └───────────┘
```

**核心原则：** 自路由、直连、C++ 友好、可扩展

### 5.2 分片策略（以 IM 为例）

| 表类型 | 分片键 | 策略 |
|--------|--------|------|
| 用户表 | user_id | user_id % 32 |
| 消息/会话 | conversation_id | conversation_id % 64 |
| 群组 | group_id | group_id % 32 |

### 5.3 冷热分离

| 层级 | 介质 | 用途 |
|------|------|------|
| 热数据 | NVMe SSD | 近期消息、活跃用户 |
| 冷数据 | HDD / 对象存储 | 历史消息归档 |

### 5.4 MySQL 能力边界

| 规模 | 推荐方案 |
|------|---------|
| 日活 < 100 万、QPS < 10 万 | MySQL 集群 |
| 日活 100 万~1000 万、QPS 数十万 | MySQL + Redis + 消息队列 |
| 日活 1000 万~亿、QPS 百万+ | TiDB / 分布式 KV |
| 微信级（10 亿日活） | 自研 PaxosStore |

> 微信核心消息存储**未使用 MySQL**，而是自研 **PaxosStore**（多主、非租约 Paxos Log、LSM 树、6 个 9 可用性）。
