# IM单聊消息发送流程

> 本页涵盖雷漫分布式IM系统中单聊消息的完整发送流程，包括端到端加密和非加密两种模式，以及已读回执、未读数缓存等关键设计要点。

## 单聊管理

### 用户A发单聊到B-端到端加密-完整流程图

```
1. 用户A在客户端输入消息
         │
         ▼
2. 客户端检查A与B的会话(单聊)是否存在
         │
         ├─►有会话      
         │       │
         │       ▼
         │ 3.1 复用会话ID（会话ID即A和B的用户ID组合，格式如Min(A,B)_Max(A,B)）
         │
         └─►无会话
                 │
                 ▼
           3.2 新建会话，生成会话ID（用A和B的用户ID拼接：Min(A,B)_Max(A,B)）
         │
         ▼
4. 客户端查询本地与B的端到端密钥（keyX，sessionKey）
         │
         ├─►密钥有效
         │
         ▼
         │
         │
         └─►密钥无效/过期
                   │
                   ▼
               a. 协商/拉取B的CA公钥
               b. 端到端加密协商 (生成sessionKey，协商keyX)
               c. 缓存/落地新密钥，并上传/备份keyX到服务器（服务器需安全存储加密的keyX，应确保服务端无法直接明文获取）
         │
         ▼
5. 客户端对消息内容用keyX加密，签名 
         │
         ▼
6. 封装消息体（加密消息体、密钥版本号、消息ID、A与B等元数据、时间戳等）
         │
         ▼
7. 通过长连接/网关接口发送消息请求给服务器
         │
         ▼
8. 服务器端收到消息包
         │
         ├─►参数/身份合法性校验（token/session检查）
         ├─►写入MongoDB单聊消息表(msg_c2c)
         ├─►更新用户双方session活跃、消息序列号
         ├─►推送流程:
         │      ├─B在线: 立即下发到B的所有在线设备
         │      └─B离线: 写入离线消息/推送队列
         ├─►同步更新B的未读数缓存
         └─►可选: 更新消息推送状态表
         │
         ▼
9. B客户端收到新消息包（在线推送/离线拉取）
         │
         ▼
10. B客户端解析消息体，用本地当前密钥集尝试解密
         │
         ├─►解密成功：展示明文，记录最后已读消息ID
         └─►解密失败：自动发起密钥协商，再尝试接收
         │
         ▼
11. B客户端回发已读回执（携带消息ID）
         │
         ▼
12. 服务端更新A端会话：对方已读消息ID、单聊未读数等
         │
         ▼
13. A/B客户端本地持久化加密消息、相关会话信息
```

简要说明：
- 端到端密钥仅存于A/B本地，消息明文仅短暂存在于内存，服务端仅转发/存储加密内容。
- 未读数、消息推送、会话、密钥版本等均通过缓存和数据库协同维护，保证消息实时性与一致性。
- 多端登录、掉线重登、撤回等分支可按需扩展补充。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-五、关键设计要点.md]

### 用户A发单聊到B-不用端到端加密-完整流程图

步骤如下：

1. 用户A在客户端发起单聊消息  
   │
   ▼
2. 客户端获取目标用户B的信息（userId、设备等）  
   │
   ▼
3. 客户端本地生成消息内容（明文，或含图片/文件等），分配msgId/seq
   │
   ▼
4. 封装消息体（JSON/Proto格式等），包含必要的元数据（如时间戳、消息ID、会话ID等）  
   │
   ▼
5. 客户端携带token/session等鉴权信息，通过长连接或网关接口，将消息包发送至IM服务器  
   │
   ▼
6. 服务器端收到消息包，依次处理如下：  
     ├─ 不校验Token/Session（此步在登录阶段已完成，消息发送时忽略）  
     ├─ 检查用户A与B的好友关系及是否被拉黑（如被拉黑则拒绝投递）  
     ├─ 使用Redis为本会话生成/分配msgSeq消息序号  
     ├─ 消息内容写入MongoDB单聊消息表(msg_c2c)  
     ├─ 根据双方ID，查询B当前在线状态  
     ├─ 在线推送：若B在线，将消息推送到B所有在线设备  
     ├─ 离线处理：若B离线，将消息写入离线消息队列/推送队列  
     └─ （可选）记录本次消息的投递或推送状态，便于后续追踪与补偿

7. 用户B端（客户端）收到新的单聊消息：  
   │
   ▼
8. 客户端本地保存消息，加到对应单聊会话消息列表  
   │
   ▼
9. 展示消息内容给用户  
   │
   ▼
10. 用户B收到A发送的单聊消息后，会根据自己的阅读进度，将最新已读的消息ID（即A发给B的msgId/seq）作为回执发送给服务器。服务器收到后，更新A端“对方已读”状态，并据此重新计算A端的未读消息数（未读数指A发出且B尚未标记为已读的消息，即msgId/seq大于已读回执的消息）。
   │
   ▼
11. 客户端本地持久化已读状态、消息内容、相关会话信息

b的已读消息就是一条新的消息。已读回执本质上属于消息响应，即每次B客户端读到A发的消息并主动上报已读进度时，这个“已读消息”也作为一条消息记录（具有独立的msgId/seq），存储进消息表（如msg_c2c）中，与普通文本/图片/撤回等消息同级。和普通聊天消息一样，已读回执消息会包含用于同步和追溯的必要元数据（如指向的已读消息ID、对话ID、发起方、时间戳等）。

// 具体说明：
// - 单聊消息表(msg_c2c)中，已读回执消息和普通消息一样插入，只是type或content字段表明其为“已读回执”。
// - B客户端主动把“已读到的A→B某条消息ID/seq”作为一条新消息发给服务端，服务端分配新msgId/newSeq并存储为一条“已读回执消息”。
// - 服务器处理时可以区分普通消息和已读回执消息（如type=read_receipt），同步给A端时A侧用来展示“对方已读到哪一条消息”状态。
// - 未读数计算、历史消息查询等均统一视为消息流处理，已读回执占用自己的msgId/seq。

// 总结：已读回执是B客户端主动发出的特别类型的新消息（消息响应），持久化进消息表、同步和展示流程与普通消息一致，不做单独分区、更新，而是完整存为一条新消息。

单聊的已读回执消息的content字段通常存储的是“被已读的消息的msgId？
是的，**单聊的已读回执消息的content字段通常存储的是“被已读的消息的msgId或msgSeq”**（即A发送给B的哪一条消息被B读到了）。

- 结构举例：
  - type: "read_receipt" 或类似枚举
  - content: 被已读的消息msgId（或msgSeq，根据系统主键设计）

例如：
```json
{
  "type": "read_receipt",
  "content": 12345678,  // 表示B已读到A发的msgId=12345678那一条
  "from": "userB",
  "to": "userA",
  "sessionId": "A_B",
  "timestamp": 1690000000
}
```
这样设计便于服务器据此刷新已读进度和计算未读数。同时带上必要的元数据（时间戳、发送者、会话ID等）辅助定位和追溯。

**注意：**
- content是单一ID（某些系统用msgSeq，也有用msgId，视后端ID主键设计为准）。
- 有的方案会扩展为结构体，包含多端场景下“已读设备列表”等，但单聊主流程以“已读到哪条消息”为主。
- 实际方案可根据需求，支持多条ID（如批量已读）、扩展信息等，但核心思路不变。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-五、关键设计要点.md]

### 2. 写入未读数缓存

- **新消息发送成功后**，更新 `lastMsgSeq`，根据当前已读点计算 `unreadCount`。
- **收到已读回执后**，更新 `readSeq`，同步刷新 `unreadCount`。
- **撤回时**，如被撤回消息序号在 `readSeq` ~ `lastMsgSeq` 区间，需重算未读数（见下）。

未读消息（即每个用户在每个会话的未读消息数）**通常不直接存入MySQL表，而是只存于Redis等缓存系统**，不做持久化。原因如下：

1. **高频读写，性能需求高**：未读数每次新消息/已读回执都会频繁变动，需要高性能实时读写，MySQL更适合底层消息和会话存档，热点数据靠Redis持久化代价太高。
2. **源数据可根据消息表重算**：未读数属于可由消息表、已读回执表等数据快速“重建”出的衍生数据。如果Redis缓存丢失，系统可遍历消息表和已读点重新计算，不会造成一致性问题。
3. **缓存为主，最终一致**：大厂普遍做法是将未读数（如 lastMsgSeq、readSeq、unreadCount）**实时缓存**于Redis结构（如Hash），必要时靠定时任务或回查手动矫正缓存，实现“快路径”体验与“最终一致性”兼顾。
4. **极少跨端边界持久化**：如需全端查询一致性或用于批量统计，可以定期异步刷写MySQL一份冗余，但主流程都在Redis，MySQL主要承载消息本体与已读回执持久化。

**补充：部分系统会将“总未读数”每晚异步落MySQL，作为运营统计用，并非实时主流程依赖。**

**结论：**
- 未读数主存Redis，不直接落MySQL。
- 需要时可完全由MySQL中的消息表+已读点重建，保障无数据丢失。
- 只有核心消息/已读消息“点”入库，未读数实时热数据推荐纯缓存。

举例：未读数缓存可结构如下
```
redis.hset(f'unread:{userid}:{sessionid}', mapping={
    'readSeq': ...,
    'lastMsgSeq': ...,
    'unreadCount': ...
})
```
如需恢复或校验一致性，后台定期全量扫描消息表、回执表重算缓存值即可。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-五、关键设计要点.md]

#### 4. 推荐实现策略：

- “快路径”优先：撤回集合为空，且seq连续、状态正常时→直接`lastMsgSeq - readSeq`；
- “慢路径”容错：只要撤回集合非空、缓存恢复后首次访问、监控异常等→走`calc_unread`；
- 保证大部分用户体验为“快、实时”，少量异常“慢、最终一致”；
- 可通过版本号或“撤回变更标记”做快速分支判断。

**伪代码：**
```
def get_unread(userid, sessionid):
    readSeq = redis.hget(f'unread:{userid}:{sessionid}', 'readSeq')
    lastMsgSeq = redis.hget(f'unread:{userid}:{sessionid}', 'lastMsgSeq')
    if not is_revoke_exists_after(sessionid, readSeq) and seq_is_continuous(readSeq, lastMsgSeq):
        # 快速路径：readSeq之后无撤回，序号连续
        return lastMsgSeq - readSeq if lastMsgSeq >= readSeq else 0
    else:
        # 精确统计：有撤回或异常
        # 优化点：撤回消息集合可改存为有序集合(zset)，直接根据序号范围取出被撤回seq，提高效率
        # 使用zrangebyscore仅获取(readSeq, lastMsgSeq]区间的撤回消息
        revoked_seqs = set(
            int(x) for x in
            redis.zrangebyscore(f'revokeSeqZSet:{sessionid}', readSeq + 1, lastMsgSeq)
        )
        unread = 0
        for seq in range(readSeq + 1, lastMsgSeq + 1):
            if seq not in revoked_seqs:
                unread += 1
        return unread

def is_revoke_exists_after(sessionid, readSeq):
    # 判断 readSeq 之后是否存在撤回seq
    # 利用zset，zcount统计(readSeq, +inf]区间是否有撤回
    return redis.zcount(f'revokeSeqZSet:{sessionid}', readSeq + 1, '+inf') > 0

# 大厂一般如何做seq连续性判断？
def seq_is_continuous(readSeq, lastMsgSeq, sessionid):
    """
    大厂常用的方式也是仅依赖缓存判断seq连续性，不查询MySQL主库。
    典型做法是利用Redis的zset存储每一条消息的seq，score为消息序号，value为msgid或序列化内容。
    需要判断(readSeq, lastMsgSeq]区间内是否所有seq都有落盘/缓存，无缺口即可。

    通常实现如下：
    1. Redis ZSET：用zcount快速统计区间数量
        - zcount(key, readSeq+1, lastMsgSeq) == (lastMsgSeq - readSeq) 则判定连续
    2. 集群服务间可能依赖本地LRU、异步回查等，极端场景也会兜底异步容错
    3. 如果有异常（如zcount不等），则进一步降级全量查/diff补偿，但快路径走纯缓存
    """
    key = f"msgSeqZSet:{sessionid}"
    expected = lastMsgSeq - readSeq
    actual = redis.zcount(key, readSeq + 1, lastMsgSeq)
    return actual == expected

// get_unread 客户端查询时机如下：
// 1. 用户进入会话页面（聊天窗口）时，前端会主动拉取未读数，调用 get_unread；
// 2. 前端收到“有新消息”（如推送、长连接新消息回包等）时，通常会刷新本地消息列表。理论上客户端可自行根据消息合并逻辑增减未读数，但为确保与服务器端状态完全一致，仍建议重新调用 get_unread 进行精确查询；
// 3. 用户主动下拉刷新或切换会话时，有时也会触发未读数重新拉取；
// 4. 特定场景如撤回、已读回执、消息漫游/补偿等，有可能需要重新触发 get_unread 查询；
// 5. 定时自动刷新（极少数，但部分客户端可能设有心跳自动刷新未读数）；
 
// 总结：只要客户端需要显示最新未读数（角标、红点等），就会通过API请求后端，实际调用 get_unread。

```

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-五、关键设计要点.md]

### 6.1 登录流程
1. 客户端发送登录请求（deviceid、token、userid、loginseq）
2. 微服务验证token并创建/更新登录状态缓存
3. 生成sessionid和sessionkey
4. 返回登录结果给客户端

### 6.2 消息发送流程
1. 客户端发送消息到Access网关
2. Access验证sessionkey并加解密
3. 消息服务生成msgId和msgseq
4. 写入MongoDB持久化
5. 更新Redis缓存（会话状态、推送状态等）
6. 推送给在线用户或写入离线消息

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-五、关键设计要点.md]