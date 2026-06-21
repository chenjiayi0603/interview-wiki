# Redis 速查

---

## 一、数据结构使用场景

| 类型 | 场景 | 限制 |
|------|------|------|
| String | 缓存、计数器 | 512MB |
| Hash | 用户信息、对象 | 适合 field 少 |
| List | 消息队列(LPUSH+BRPOP) |  |
| Set | 标签、去重、抽奖 | |
| ZSet | 排行榜(skiplist+dict) | |
| Bitmap | 日活签到 | BITCOUNT |
| HyperLogLog | UV 统计 | 12KB 存 2^64，误差 0.81% |

---

## 二、过期策略

```
被动: 访问 key 时检查是否过期
主动: 每 100ms 随机抽 20 个 key
  过期 key 占 25%+ → 重复抽
```

---

## 三、淘汰策略 (maxmemory-policy)

```
缓存场景 → allkeys-lru
存储场景 → noeviction（配合监控）
```

---

## 四、持久化

```
RDB: 全量快照（fork + COW），恢复快，可能丢数据
AOF: 追加日志，可配置 everysec/always，恢复慢
混合: Redis 4.0+，RDB + AOF 增量
```

---

## 五、高可用

| 方案 | 组件 | 特点 |
|------|------|------|
| 主从复制 | 1 master + N slave | 读写分离，手动切 |
| 哨兵 | Sentinel + 主从 | 自动故障转移 |
| Cluster | 16384 槽分片 | 最多 1000 节点 |
| Codis | Proxy + 分片 | 支持动态扩缩 |

---

## 六、缓存问题

```
穿透: 布隆过滤器 / 空值缓存
击穿: 互斥锁 / 逻辑永不过期
雪崩: 过期时间随机化 / 多级缓存
```
