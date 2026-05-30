# MySQL 集群与高可用

## 一、主从复制

```sql
-- 主库配置
server-id=1
log-bin=mysql-bin
binlog-do-db=test

-- 从库配置
server-id=2
relay-log=relay-bin
read-only=ON

-- 复制类型
-- 异步复制：主库不等待从库
-- 半同步复制：至少一个从库确认
-- 全同步复制：所有从库确认
```

### 复制原理

```
┌────────┐  binlog  ┌────────┐  relaylog  ┌────────┐
│ Master │ ──────> │ Slave IO │ ────────> │ Slave SQL │
│        │         │  Thread  │           │  Thread  │
└────────┘         └────────┘            └─────────┘
```

## 二、GTID 复制

```sql
-- Global Transaction ID
-- 自动找位点
-- 故障转移简单
gtid_mode=ON
enforce_gtid_consistency=ON
```

## 三、分库分表

- **垂直分库**：按业务拆分
- **水平分表**：按数据拆分（哈希/日期/范围）
- **中间件**：ShardingSphere、MyCat、Cobar

## 四、面试高频问题

### Q1: IM消息系统如何设计消息存储？

- 分表策略：按用户ID哈希或时间分表
- 冷热分离：近期消息SSD，历史消息HDD
- 索引设计：conversation_id + created_at
- 消息顺序：全局序列号或逻辑时钟
- 群聊消息：多副本写入

### Q2: 社交IM项目900万用户如何优化？

- 分库分表
- 读写分离
- 热点数据缓存
- 批量处理
- 异步写入

### Q3: 为什么选择MySQL而不是其他数据库？

- 事务支持
- 成熟稳定
- 生态丰富
- 成本可控
- 团队熟悉

[src: raw/ingested/2技术/mysql/MySQL知识.md]

## Related Pages
- [[MySQL主从复制]]
- [[MySQL事务提交与两阶段提交]]
- [[MySQL架构与存储引擎]]
- [[分布式IM消息系统-雷漫]]
