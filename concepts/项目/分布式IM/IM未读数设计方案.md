# IM 未读数设计方案

> 本文档归纳 IM 系统中未读数（红点）的设计方案，基于项目文档与讨论整理。

## 一、核心设计

### 1.1 未读计算公式

**单聊**：
```
未读数 = count(msgId > lastReadMsgId AND fromId != self AND msgState != 4)
```

**群聊**：
```
未读数 = count(
  cmdType IN [4501, 4505] AND
  fromId != self AND
  msgId > lastReadMsgId AND
  msgState != 4
)
```

说明：
- `msgState != 4`：排除撤回消息
- `fromId != self`：排除自己发的
- `cmdType`：只计聊天/通知类消息

### 1.2 已读点来源：用户活跃会话数据 (sessionactive)

> 详见：[[IM数据库与缓存设计|IM详细设计--数据库缓存总结]] 第 3.4.2 节

**Redis 结构**：
- **Key 格式**：`1:10:im:sessionactive:userid:${userid}`
- **用途**：存储用户活跃会话的消息 ID 状态
- **数据结构**：Hash
- **字段**：
  - `${targetid}:read_msgid`：个人已读消息 ID（已读点）
  - `${targetid}:friend_read_msgid`：对方已读自己的位置（单聊）
  - `${targetid}:min_msgid`：起始消息 ID
  - `${targetid}:max_msgid`：会话最大消息 ID
- **说明**：群聊 targetid 为群 ID，单聊 targetid 为目标用户 ID

`lastReadMsgId` = Redis Hash 中的 `${targetid}:read_msgid`。  
会话列表和已读点均可从 **sessionactive** 获取（HKEYS 解析 targetid），无需再查 MongoDB 获取会话列表。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

## 二、为什么必须用 msgId 作为已读点

| 维度 | msgId | seq |
|------|-------|-----|
| 唯一性 | 全局唯一 | 仅会话内唯一 |
| 循环/溢出 | 64 位，一般不循环 | Int32 会溢出循环 |
| 分配依赖 | 雪花/分布式 ID | 依赖 Redis 预分配 |
| Redis 故障 | 不影响已读点 | seq 可能重叠、未读错乱 |
| 撤回处理 | count(msgState!=4) 自然排除 | 需额外 revoke_seq，异步写入易失败 |
| 结论 | ✅ 适合作为已读点 | ❌ 不适合 |

**结论**：已读点必须使用 msgId，未读统计必须基于 MongoDB count。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

## 三、Redis INCR 方案的问题

| 问题 | 说明 |
|------|------|
| 写扩散 | 大群 1 条消息 → N 个成员各写 1 次，N 次 Redis 写 |
| 撤回 | 需 read_msgid 判断是否 DECR，逻辑复杂 |
| revoke_seq | 异步写入易失败，与消息表易不一致 |

**结论**：不采用 Redis INCR 作为未读主方案。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

## 四、整体架构

### 4.1 推荐方案

| 场景 | 做法 |
|------|------|
| 在线 | 推送 + 客户端本地维护 |
| 离线 | 不维护 Redis 未读计数 |
| 登录 | 必须查 MongoDB count，得到当前未读数 |

### 4.2 数据流

```
1. 会话列表：从 Redis sessionactive 获取（targetid + read_msgid）
2. 未读统计：MongoDB count(msgId > lastReadMsgId AND ...)
3. 总未读：各会话未读之和，无需单独查总
```

### 4.3 MySQL tb_no_read_sum 的取舍

| 结论 | 说明 |
|------|------|
| 离线时写的快照 | 会过期（离线期间别人发消息），不可作为登录时唯一数据源 |
| 登录查 MongoDB 后写 MySQL | 下次登录会过期，意义不大 |
| 在线期间定时写 MySQL | 快照很快过期，必要性低 |
| 总体 | 在「登录查 MongoDB + 在线推送 + 客户端本地」方案下，tb_no_read_sum 可省略 |

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

## 五、拉与推

| 模式 | 场景 |
|------|------|
| 拉 | 登录、刷新、兜底 |
| 推 | 实时消息、未读变化 |
| 组合 | 拉做初始化与兜底，推做实时更新 |

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

## 六、MongoDB 查询优化

### 6.1 会话列表来源

- **Redis sessionactive**：HKEYS 解析 targetid，同时拿到 read_msgid，足够获取所有有效会话及已读点。

### 6.2 每个会话未读

- 每个会话 lastReadMsgId 不同，需按会话分别 count。
- 若需要会话列表红点，则需要每个会话的未读数。

### 6.3 总未读

- 总未读 = sum(各会话未读)，无需单独查总。

### 6.4 按需查询优化

| 策略 | 做法 |
|------|------|
| 首屏 | 只查当前展示的 K 个会话（如前 20） |
| 总未读 | 1 次 aggregation 覆盖所有会话 |
| 滚动 | 新展示的会话再按需查 |

### 6.5 合并查询与分片

- 使用 `$match` + `$or`（包含分片键 groupId）可在 MongoDB 分片集群上执行。
- 单次 aggregation 可同时返回多会话未读及总未读。

### 6.6 点对点消息表 (msg_c2c) 与未读数计算

> 详见：[[IM数据库与缓存设计|IM详细设计--数据库缓存总结]] 第 4.1 节

#### 表结构
- **分片键**：userId
- **索引**：userId, msgId（建议复合索引 userId+msgId）

#### 主要字段（未读相关）
| 字段名 | 类型 | 说明 |
|--------|------|------|
| userId | uInt64 | 用户ID（等于 fromId/toId，分片键） |
| fromId | uInt64 | 发送用户ID |
| dstId | uInt64 | 接收用户ID |
| msgId | Int64 | 消息ID |
| cmdType | int | 离线消息类型 |
| msgState | Int8 | 消息状态（0:默认 2:送达 3:已读 4:撤回 5:删除自己 6:清空会话） |

#### 未读数计算规则
1. 统计大于 lastReadMsgId 的消息（cmdType 为原始聊天/通知消息，fromId ≠ self），得到总数 A
2. "删除自己"（msgState=5）状态消息不纳入未读——客户端收到时自行扣减
3. 未读数 = A - 已撤回消息数 B（可选，服务端可不推送或客户端过滤）

#### MongoDB 单聊未读查询示例
```javascript
const unreadCount = await db.msg_c2c.countDocuments({
  userId: selfId,                     // 当前用户ID（分片键）
  msgId: { $gt: lastReadMsgId },      // 大于已读点
  cmdType: { $in: [4501, 4506] },     // 4501:聊天 4506:通知
  fromId: { $ne: selfId },            // 排除自己发的
  msgState: { $ne: 4 }                // 可选，排除撤回（cmdType 过滤已足够）
});
```

说明：4501 为原始聊天消息，4506 为通知消息，均属未读统计范围。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

### 6.7 群消息表 (msg_c2g) 与 MongoDB 查询示例

> 详见：[[IM数据库与缓存设计|IM详细设计--数据库缓存总结]] 第 4.2 节

#### 表结构
- **分片键**：groupId
- **索引**：groupId, msgId（建议复合索引 groupId+msgId）

#### 主要字段（未读相关）
| 字段名 | 类型 | 说明 |
|--------|------|------|
| groupId | Int64 | 群ID（分片键） |
| msgId | Int64 | 消息ID（唯一） |
| fromId | Int64 | 发送用户ID |
| cmdType | int | 离线消息类型 |
| msgState | Int8 | 消息状态（0:默认 3:已读 4:撤回 5:删除自己） |

#### MongoDB 群聊未读查询示例
```javascript
const unreadCount = await db.msg_c2g.countDocuments({
  cmdType: { $in: [4501, 4505] },    // 只查聊天/通知消息
  fromId: { $ne: selfId },           // 排除自己发的
  groupId: groupId,                  // 群ID
  msgId: { $gt: lastReadMsgId },     // 大于已读点
  msgState: { $ne: 4 }               // 排除撤回状态
});
```

#### countDocuments 复杂度

有 **(groupId, msgId)** 联合索引时：

| 阶段 | 操作 | 复杂度 |
|------|------|--------|
| 索引查找 | 在 B+ 树中一次性查找复合键 (groupId, lastReadMsgId+1) | O(log M) |
| 范围扫描 | 遍历 msgId > lastReadMsgId 的索引项并过滤 | O(K) |

**总复杂度**：O(log M) + O(K)

- **M**：分片内文档总数（索引键数）
- **K**：未读区间条数（msgId > lastReadMsgId 的文档数）

复合索引一次性定位，无需先找 groupId 再找 msgId；单次 count 通常 <5ms。

**建议索引**：`db.msg_c2g.createIndex({ groupId: 1, msgId: 1 })`

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

## 七、读扩散（大群优化）

| 维度 | 写扩散 | 读扩散 |
|------|--------|--------|
| 1 条消息 | N 次 Redis 写 | 1 次（仅更新群 last_msgid） |
| 未读计算 | 直接读缓存 | last_msgid - lastReadMsgId 或 count |
| 适用 | 小群、单聊 | 大群 |

有撤回时，读扩散仍依赖 MongoDB count，与 msgId 方案一致。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

## 八、登录时必须查 MongoDB

在「不用 Redis INCR」「MySQL 快照会过期」的前提下：

| 数据源 | 登录时是否可用 |
|--------|----------------|
| MySQL | ❌ 已过期 |
| Redis INCR | ❌ 有写扩散和撤回问题 |
| MongoDB count | ✅ 唯一可靠来源 |

**结论**：登录时必须查 MongoDB 才能得到准确未读数。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

## 九、流程简图

```
┌─────────────────────────────────────────────────────────────────┐
│ 登录                                                             │
│   1. Redis HGETALL sessionactive → 会话列表 + read_msgid         │
│   2. MongoDB count（按会话或 aggregation）→ 各会话未读 + 总未读   │
│   3. 返回客户端                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 在线                                                             │
│   1. 新消息 → 推送                                               │
│   2. 客户端本地 +1                                                │
│   3. 已读 → 推送 → 客户端本地更新                                 │
└─────────────────────────────────────────────────────────────────┘
```

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]

## 十、要点速查

| 项目 | 结论 |
|------|------|
| 已读点 | msgId，来自 Redis sessionactive |
| 未读统计 | MongoDB count，带 msgState!=4 等条件 |
| seq 作为已读点 | 不可行（溢出、撤回、故障重叠） |
| Redis INCR | 不采用（写扩散、撤回复杂） |
| MySQL tb_no_read_sum | 可省略（快照易过期） |
| 登录 | 必须查 MongoDB |
| 总未读 | sum(各会话未读)，不必单独查 |
| 查询优化 | 按需查展示会话，总未读可 1 次 aggregation |

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM未读数设计方案总结.md]