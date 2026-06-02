# MySQL 读写流程

> 读流程 8 步、写流程、Buffer Pool、Change Buffer、Doublewrite、参数调优。

---

## 一、读流程（SELECT 执行过程）

### 1.1 整体 8 步

```
步骤  阶段               谁负责           做什么
────  ────────────     ────────────    ──────────────────────────
 ①   连接建立           连接器           TCP 连接、身份验证、权限检查
 ②   查询缓存           查询缓存         检查是否命中（MySQL 8.0 已移除）
 ③   词法语法分析        分析器/Parser    词法分析 → Token；语法分析 → AST
 ④   执行计划生成        优化器           选择索引、确定 JOIN 顺序
 ⑤   调用引擎接口        执行器           调用 Handler API
 ⑥   Buffer Pool 查找   InnoDB 引擎      查内存中的 Buffer Pool
 ⑦   磁盘读取            InnoDB 引擎      Buffer Pool 未命中时从磁盘读取
 ⑧   返回结果            执行器→客户端    结果集返回
```

### 1.2 各阶段详解

**连接器：**
- 长连接：持续复用，减少连接开销
- 权限在连接时读取并缓存，修改权限不影响已有连接
- `wait_timeout` 控制空闲超时（默认 8 小时）

**分析器：**
```sql
SELECT name, age FROM users WHERE id = 100;
-- 词法分析：识别 SELECT/name/FROM/WHERE 等 Token
-- 语法分析：生成 AST，验证表和列是否存在
```

**优化器核心决策：**

| 决策项 | 说明 |
|--------|------|
| 索引选择 | 哪个索引成本最低 |
| JOIN 顺序 | 小表驱动大表 |
| JOIN 算法 | Nested Loop / Hash Join |
| 子查询优化 | 子查询转 JOIN、物化 |

**执行器：**
```sql
-- 主键查询
SELECT name, age FROM users WHERE id = 100;
-- 执行器调用 ha_index_read(id=100)
-- InnoDB 定位主键索引 B+Tree 叶子节点
-- 返回行记录，提取需要的列

-- 全表扫描
SELECT * FROM users WHERE age = 25;
-- 执行器调用 ha_rnd_init() → ha_rnd_next() 逐行读取
```

---

## 二、写入流程（DML 执行过程）

### 2.1 写入流程总览

```
                  事务开始
                     │
              ┌──────▼──────┐
              │  执行 DML    │
              └──────┬──────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌────▼────┐    ┌─────▼─────┐
│ 写 Undo  │    │ 修改     │    │ 写 Redo   │
│ Log      │    │ Buffer   │    │ Log Buffer│
│ (回滚段) │    │ Pool 数据│    │ (内存中)  │
└────┬────┘    └────┬────┘    └─────┬─────┘
     │               │               │
     └───────────────┼───────────────┘
                     │
              ┌──────▼──────┐
              │  COMMIT 提交  │
              └──────┬──────┘
                     │
        ╔════════════╧════════════╗
        ║  两阶段提交 (2PC)        ║
        ║  ④ Redo Prepare + 刷盘  ║
        ║  ⑤ 写 Binlog + 刷盘     ║
        ║  ⑥ Redo Commit         ║
        ╚════════════╤════════════╝
                     │
              ┌──────▼──────┐
              │ 返回客户端成功 │
              └──────┬──────┘
                     │
              ┌──────▼──────┐
              │ Page Cleaner │
              │ 后台刷脏页   │
              └─────────────┘
```

### 2.2 各步骤详解

**步骤①：写 Undo Log**
- 记录修改前的值，用于回滚和 MVCC
- Undo Log 的修改也会产生 Redo Log

**步骤②：修改 Buffer Pool**
- 在 Buffer Pool 中修改数据页，标记为「脏页」
- 脏页加入 Flush List，等待后台 Page Cleaner 刷盘

**步骤③：写 Redo Log Buffer**
- 内存中记录数据页的物理修改
- 事务提交时刷盘（受 `innodb_flush_log_at_trx_commit` 控制）

**步骤④-⑥：两阶段提交（详见 04-日志与复制）**
- Redo Prepare → 写 Binlog → Redo Commit

---

## 三、Buffer Pool 详解

### 3.1 内存结构

```
Buffer Pool Instance:
┌─────────────────────────────────────────────────────────────┐
│  LRU List（页链表）                                           │
│  ┌──────────────────────┐  ┌──────────────────────────┐    │
│  │   Young 区域（5/8）   │  │   Old 区域（3/8）         │    │
│  │   热数据               │  │   冷数据/新加载页         │    │
│  └──────────────────────┘  └──────────────────────────┘    │
│                                                             │
│  Free List（空闲页链表）                                      │
│  Flush List（脏页链表）                                       │
│  Adaptive Hash Index（自适应哈希索引）                         │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 改进 LRU 算法

InnoDB 使用 Midpoint Insertion 策略，防止全表扫描冲掉热数据：

```
新页加载 → 插入 Old 区域头部
再次访问 → 晋升到 Young 头部（需等待 innodb_old_blocks_time=1000ms）
淘汰     → 从 Old 尾部移除
```

**参数控制：**
```ini
innodb_old_blocks_pct = 37          # Old 区域占比
innodb_old_blocks_time = 1000       # 晋升等待时间（ms）
```

---

## 四、Change Buffer 写优化

### 4.1 原理

当更新的二级索引页不在 Buffer Pool 时，不立即读盘，而是将修改缓存起来。

```
不用 Change Buffer：
  读二级索引页（随机读）→ 修改 → 写 Redo Log → 写回（随机写）

使用 Change Buffer：
  写 Change Buffer（顺序写）→ 写 Redo Log
  → 随机读被延迟到真正需要时
```

### 4.2 适用条件

- ✅ 必须是**二级索引**（非聚簇索引）
- ✅ 索引页**不在 Buffer Pool** 中
- ❌ 不能是**唯一索引**（需要读盘验证唯一性）

**最佳场景：** 写多读少的二级索引（日志表、流水表）

```ini
innodb_change_buffering = all         # all/none/inserts/deletes
innodb_change_buffer_max_size = 25    # 占 Buffer Pool 的 25%
```

---

## 五、Doublewrite Buffer

### 5.1 解决的问题

InnoDB 数据页大小 16KB，OS 页大小 4KB。写 16KB 页时可能发生**部分写入**（只写完 4KB 就崩溃），导致数据页「断裂」。

### 5.2 流程

```
Buffer Pool 脏页 → Doublewrite Buffer（顺序写连续 128 页）
                 → 数据文件 .ibd（随机写）

崩溃恢复：
  检查 .ibd 中数据页 checksum
  损坏 → 从 Doublewrite Buffer 获取完整副本恢复
```

---

## 六、参数调优

### 6.1 读流程优化

```ini
[mysqld]
# Buffer Pool 大小（建议物理内存 50%~70%）
innodb_buffer_pool_size = 12G
innodb_buffer_pool_instances = 8      # 减少锁竞争
innodb_old_blocks_time = 1000         # 防止全表扫描冲掉热数据
innodb_read_ahead_threshold = 56      # 线性预读阈值
```

### 6.2 写流程优化

```ini
# Redo Log
innodb_log_file_size = 2G             # Redo Log 文件大小
innodb_log_files_in_group = 2
innodb_log_buffer_size = 64M          # 大事务需要足够 Buffer
innodb_flush_log_at_trx_commit = 1    # 安全优先
sync_binlog = 1

# 脏页刷盘
innodb_max_dirty_pages_pct = 75
innodb_io_capacity = 2000             # SSD 建议 2000+
innodb_io_capacity_max = 4000
innodb_flush_neighbors = 0            # SSD 关闭邻居刷盘

# 组提交
binlog_group_commit_sync_delay = 100
binlog_group_commit_sync_no_delay_count = 10
```

### 6.3 场景化配置

| 场景 | innodb_flush_log_at_trx_commit | sync_binlog | 特点 |
|------|:---:|:---:|------|
| 金融/交易 | 1 | 1 | 最安全，性能较低 |
| 日志/分析 | 2 | 1000 | 可容忍少量丢失，性能高 |
| 大事务/批量导入 | 1 | 1 | 加大 log_buffer 和 log_file_size |

### 6.4 面试高频问题

**Q: 事务提交后，数据是否立即写入磁盘？**

答案：不一定！
- ✅ Redo Log 和 Binlog 在提交时刷盘（受参数控制）
- ❌ 数据文件 (.ibd) 由 Page Cleaner **异步**刷盘
- 持久性由 Redo Log 保证，不是由数据文件保证

**Q: 为什么需要 Redo Log？直接刷脏页不行吗？**

1. **性能**：刷脏页是随机写（慢），Redo Log 是顺序写（快几十倍）
2. **崩溃恢复**：Redo Log 记录完整修改，可以精确重放
3. 这就是 **WAL（Write-Ahead Logging）** 的核心思想
