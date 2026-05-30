# IM数据库与缓存设计

> 本页涵盖雷漫分布式IM系统的数据库与缓存设计，包括Redis缓存数据结构、MongoDB数据库设计、MySQL数据库设计。

## 2.1 Redis缓存数据结构

### 2.1.1 用户登录Token (String)

- **Key格式**: `4:1:im:token:userid:${userid}:deviceid:${deviceid}`
- **用途**: 存储每个设备每个用户的登录token
- **数据结构**: String

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.2 用户登录状态缓存（Hash）

- **Key格式**: `1:2:im:status:userid:${userid}`
- **数据结构**: Hash
- **重要字段**:
  - `logintime:${deviceid}`: 登录时间（秒）
  - `clienttype:${deviceid}`: 客户端类型（0/1/2/3）
  - `status:${deviceid}`: 在线/离线状态（0/1）
  - `loginseq:${deviceid}`: 登录序列号（防重登攻击）
  - `ssidfreshtime:${deviceid}`: 心跳刷新时间
  - `sessionid:${deviceid}`: 会话ID
  - `sessionkey:${deviceid}`: 对称加密密钥（32字节）
  - `accnodeidentify:${deviceid}`: 网关节点标识（格式：ip:port:workIndex）
  - `clientip:${deviceid}`: 客户端IP
  - `platformtype:${deviceid}`: 平台类型（1:Android 2:IOS 3:Mac 4:Windows）
  - `expire:${deviceid}`: sessionkey过期时间
  - `language:${deviceid}`: 客户端语言版本

**多设备支持分析**: 该结构完全支持用户多设备同时在线，各设备独立管理、互不干扰。每个设备有独立的状态记录，每台设备登录都会生成独立的sessionkey、clienttype、loginseq等，允许同一用户ID下多个不同deviceid并存且互不影响。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.3 好友关系缓存（Hash）

- **Key格式**: `1:3:im:friendship:userid:${userid}`
- **字段**:
  - `frd_id:${friendId}`: 好友ID
  - `status:${friendId}`: A→B是否好友
  - `black:${friendId}`: A→B是否拉黑
  - `frd_status:${friendId}`: B→A是否好友
  - `frd_black:${friendId}`: B→A是否拉黑
- **更新策略**: 增量更新，存在则更新字段，不存在则从DB全量加载

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.4 用户订阅好友类型结构 (Hash)

- **Key格式**: `1:13:im:subscribe:userid:${userid}`
- **用途**: 存储用户订阅好友的状态信息
- **数据结构**: Hash
- **字段**: `subscriberid:${subscriberid}:subcmd:${subcmd}` (JSON格式)
- **JSON内容**: 包含有效时间(overtime)和截止时间(cutofftime)

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.5 群成员缓存（Hash）

- **Key格式**: `1:4:im:groupmember:groupid:${groupid}`
- **字段**: `id:${memberid}`: 群成员ID
- **更新策略**: 增量更新

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.6 用户所在群缓存（Set）

- **Key格式**: `2:5:im:belonggroup:userid:${userid}`
- **值**: 群ID列表

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.7 群订阅类型结构 (Hash)

- **Key格式**: `1:8:im:subscribe:groupid:${groupid}`
- **用途**: 存储群订阅信息
- **数据结构**: Hash
- **字段**: `userid:${userid}:subcmd:${subcmd}` (JSON格式)

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.8 会话状态数据（Hash）

- **Key格式**: `1:9:im:sessionstatus:sessionid:${sessionid}`
- **字段**:
  - `common_read_msgid`: 群会话公共已读位置消息ID（仅群聊）
  - `max_msgseq`: 会话预分配的消息序列号段最大值
  - `last_nodeid`: 预分配当前消息序列号段的结点ID
  - `node_starttime`: last_nodeid启用的时间
- **说明**: 群聊sessionid为群ID，单聊sessionid为两个用户ID组合

**max_msgseq（会话预分配的消息序列号段最大值）存在以下原因和目的：**

1. **高并发写入下的唯一性和有序性保证**  
   - IM系统消息写入需保证每条消息的序列号唯一且递增（有序）。
   - 为防止多节点/多进程并发写入出现序列号冲突，采用“批量分配消息序列号段”的机制（如每次分配1000个序列号）。
   - 每次分配后max_msgseq记录当前“已分配”的最大序列号，新的写入必须在该段范围内取序列号。

2. **提升写入性能，降低锁竞争**  
   - 批量预分配后，节点本地可直接生成序列号，无需每条写都全局加锁。
   - 只有序号段快用完时（“抢号”），才需全局同步/竞争，极大减少冲突和延迟。

3. **支持水平扩展和主备切换**  
   - 分布式架构下可保障多个写节点序列号不重叠，且冗余同步更高效。
   - max_msgseq通常与last_nodeid等搭配，支持主备动态切换和故障恢复。

4. **便于追溯和故障排查**  
   - max_msgseq可追踪已分配的历史区间，辅助数据修复、幂等等技术手段。

**举例说明**：  
- 节点A分配到max_msgseq=20000，段为[19001,20000]，各消息write只要从这段依次分配即可。
- 节点B要分配新段，需要先比对并原子递增max_msgseq，获得新区间。

**小结**：  
max_msgseq的存在主要为“分布式高性能消息写入”服务，核心作用是保证消息序号的**唯一、有序、批量分配**，防止并发写入冲突，提升整体IM系统的写入效率和可用性。

**申请语句常用于IM系统中分布式、高并发场景下向分布式系统（如Redis/MySQL/号段分配服务）发起“资源分配”的请求。**

Redis原子自增
// Hash结构无法对单个字符串字段直接进行INCRBY操作。如果max_msgseq存放在Hash（如im:session:xxx）下，可以这样：
// HINCRBY im:session:xxx max_msgseq 1000
// 如果使用简单String结构，则：
// INCRBY im:session:xxx:max_msgseq 1000

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.9 用户活跃会话数据（Hash）

- **Key格式**: `1:10:im:sessionactive:userid:${userid}`
- **字段**:
  - `${targetid}:read_msgid`: 个人已读消息ID
  - `${targetid}:friend_read_msgid`: 对方已读自己的位置（单聊）
  - `${targetid}:min_msgid`: 起始消息ID
  - `${targetid}:max_msgid`: 会话最大消息ID
- **说明**: 群聊targetid为群ID，单聊targetid为目标用户ID

**会话状态数据和用户活跃会话数据用途不同，互补设计，各有侧重：**

- **会话状态数据（sessionstatus）**：存储整个会话（群或单聊）的公共状态，比如群聊的全体已读消息位置、消息序列号等，全员共用，仅一份。
- **用户活跃会话数据（sessionactive）**：记录每个用户在各会话中的个人状态，如已读消息ID、活跃状态、未读等，按“每人-每会话”维度保存，便于计算个人未读与同步会话列表。

**总结**：
- 会话状态 = 全局元数据，群体共用。
- 用户活跃会话 = 个人维度进度，个性化状态。
- 两者分离设计便于高效查询和扩展。

**用户活跃会话数据的写入时机主要包括以下几种场景：**

1. 用户进入会话、点击后 -> 上报当前已读消息ID，写入sessionactive；
2. 收到新消息时（自动已读/页面在前台）-> 上报最新已读位置；
3. 离开会话时同步最后已读消息ID；
4. 历史消息补拉、漫游时，如有“补拉即已读”也会触发写入；
5. 多端同步，任一设备已读后自动同步其他设备进度；
6. 服务端特殊场景（如撤回/删除消息）时触发补写。

小结：凡“已读/浏览消息”时，客户端或服务端都会实时写入sessionactive，保证每个用户在每个会话里的已读进度是最新的。

**用户活跃会话数据(sessionactive)不需要长期保存，通常在以下几种情况下可以“取消”或删除：**

1. **用户主动退出登录或切换账号**：用户登出IM系统时，服务端可主动清理其sessionactive数据，确保账号隔离。
2. **设备解绑、账号异常（封号、注销）时**：涉及账户安全、隐私合规要求时，同步清理相关会话活跃数据。
3. **会话被彻底删除/清空时**：如用户手动删除会话或会话解散（如退群），对应的sessionactive（userid+targetid）数据可以物理删除，节省冗余存储。
4. **数据过期自动淘汰**：sessionactive通常可设置适当的过期时间（如30天未访问自动失效），后台定期清理长时间未活跃的会话记录。
5. **运维/合规要求的数据擦除或定向清理**：如用户申请个人信息删除时，需同步清理所有活跃会话数据。

**总结**：取消/删除sessionactive的常见场景为退出登录、会话删除、设备解绑和长期不活跃等。这样保证了数据的实时性和效率，也有利于用户隐私和资源节约。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.10 会话消息序号 (String)

- **Key格式**: `4:15:im:msgseq:sessionid:${sessionid}`
- **用途**: 存储会话的下一个消息序号起点
- **数据结构**: String

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.11 消息推送状态缓存（ZSet）

**群消息推送状态**:
- **Key格式**: `6:6:im:groupmsgpushstatus:groupid:${groupid}:userid:${userid}:deviceid:${deviceid}`
- **数据结构**: ZSet
- **Score**: 微秒级时间戳（16位数）
- **Value**: 消息ID
- **过期时间**: 1天

**单聊消息推送状态**:
- **Key格式**: `6:11:im:personalmsgpushstatus:userid:${userid}:deviceid:${deviceid}`
- **Score**: 微秒级时间戳（16位数，通常为消息推送时间）
- **Value**: 消息ID（仅存消息ID，不直接存消息体，消息内容实际存储在MongoDB，便于减少Redis内存占用及高效索引，且分布式架构下所有PushServer能统一查找/拉取消息体）
- **数据结构**: ZSet
- **过期时间**: 1天

**个人通知消息推送状态**:
- **Key格式**: `6:14:im:personalnotifypushstatus:userid:${userid}:deviceid:${deviceid}`
- **数据结构**: ZSet（同群消息推送状态）
- **过期时间**: 1天

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

推送acc失败（包括推送acc超时响应），则直接推送离线消息；
推送acc，acc响应失败，则直接推送离线消息；
推送acc，acc响应成功，则写入推送消息状态结构；
App响应成功，则移除推送消息状态结构；
用户离线，检查推送消息状态结构，有消息则离线推送。
推送消息状态结构有效期为1天。

为什么单聊推送状态（只记录消息id，用于重发）要存Redis，而不是只放内存或让单个PushServer节点处理？

简化原因如下：

- 大厂IM系统的PushServer通常是分布式多实例部署，支持弹性扩缩、节点重启或漂移。如果推送队列和未ack消息只在内存，一旦实例故障，消息就丢失，可靠性无法保证。
- Redis作为中心化存储，能让所有PushServer实例随时接管推送队列和重发任务，节点宕机、重启、扩容都不会影响消息可靠投递和去重。
- 重发流程：所有PushServer节点定时扫描Redis中的推送状态ZSet，按照分区分责任（如hash(userid/deviceid)），处理各自负责的用户消息，成功后及时移除，避免重复发。故障转移、分布式锁等也可以协作保障只发一次。
- 这种设计确保消息不会因单节点故障丢失，可靠性和扩展性高，是分布式IM推送的基本要求。

结论：推送队列、重发状态必须落Redis，靠全局定时扫描机制保证消息可靠不丢，节点重启、漂移时能够继续重发和转离线推送。

**推送流程**:
```
┌──────────┐         ┌──────────┐         ┌──────────┐
│PushServer│         │Access网关 │         │  客户端   │
└────┬─────┘         └────┬─────┘         └────┬─────┘
     │                    │                    │
     │ 1. 产生推送消息     │                    │
     │                    │                    │
     │─────────── 推送请求 ────────────────────▶│
     │                    │                    │
     │                    │                    │
     │    ┌───────────────┴───────────────┐   │
     │    │                               │   │
     │    ▼                               ▼   │
     │ 推送失败/超时                   推送成功 │
     │    │                               │   │
     │    │                               │   │
     │    ▼                               ▼   │
     │ 直接推送离线消息              写入推送状态│
     │    │                    (Redis ZSet) │
     │    │                               │   │
     │    │                               │   │
     │    │                    ┌──────────┴───┐
     │    │                    │              │
     │    │                    ▼              │
     │    │               App响应成功         │
     │    │                    │              │
     │    │                    ▼              │
     │    │              移除推送状态结构      │
     │    │                    │              │
     │    │                    │              │
     │    └────────────────────┴──────────────┘
     │                    │
     │                    │ 用户离线时
     │                    │
     │                    ▼
     │              检查推送状态结构
     │                    │
     │                    ├─ 有消息 → 离线推送
     │                    └─ 无消息 → 跳过
```

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.12 已读状态缓存（SortedSet）

**单聊会话已读消息时间**:
- **Key格式**: `6:12:im:sessionreadmsg:userid:${userid}`
- **Score**: 微秒级时间戳（已读消息ID的时间戳）
- **Value**: 目标用户ID或群ID

**单聊会话对方已读消息时间**:
- **Key格式**: `6:16:im:sessionfriendreadmsg:userid:${userid}`
- **Score**: 微秒级时间戳（对方已读消息ID的时间戳）
- **Value**: 目标用户ID或群ID
- **数据结构**: SortedSet

**单聊会话已读消息时间的作用是高效、准确地记录每个用户针对不同会话（单聊/群聊）已读消息的最新进度，实现消息已读同步与未读数统计。**

具体用途包括：
1. **精准未读数计算**：通过已读消息时间/ID，服务端在查询消息列表时能快速计算某会话未读消息数量，避免重复下发已读的消息。
2. **多端同步已读状态**：用户在某端已读消息时，已读时间写入Redis，所有设备自动同步，消息红点、未读条数实时刷新。
3. **加速新消息推送流转**：推送消息时检查用户已读进度，避免推送已读消息，提升推送效率。
4. **容错与持久化**：即使客户端异常或断线，服务端依旧能依据Redis中保存的已读时间恢复已读进度。

**总结**：存储单聊会话的已读消息时间，可实现高并发、分布式环境下的多端已读同步、未读数精准统计、带来更好的用户体验和数据一致性保障。

**写入时机：**

单聊会话已读消息时间的写入时机主要包括以下几个场景：

1. **用户已读消息上报时**：用户在某端阅读了单聊消息，客户端会主动向服务端上报已读消息ID或时间戳，服务端立即写入该用户对应会话的已读状态缓存。
2. **多端同步时**：用户在任一设备上已读后，服务端接收到上报，需立刻同步所有设备，更新Redis中的已读时间，保障多端一致。
3. **客户端启动/会话初始化时**：如客户端启动IM、重新进入单聊会话界面，或消息漫游/补拉历史消息结束后，上报最新已读位置，服务端据此写入。
4. **服务端事件触发**：如系统收到撤回、删除、清空会话等指令后，为数据一致性，服务端会同步更新（调整）相关已读状态缓存。
5. **自动刷新机制**：部分实现可能周期性刷新用户活跃会话的已读消息时间，以防遗漏或长时间未同步的异常场景。

**总结**：只要用户发生消息已读行为或服务端需校正已读状态时，都会触发单聊会话已读消息时间的写入，确保未读数准确和多端已读进度同步。

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 2.1.13 多语言文案缓存 (Hash)

- **Key格式**: `1:17:im:international:language:official:noticetype:${type}`
- **用途**: 存储多语言通知文案
- **数据结构**: Hash
- **字段**: 
  - `en-US`: 英文文案
  - `zh-CN`: 简体中文文案
  - `zh-TW`: 繁体中文文案
  - `default`: 默认文案

[src: raw/ingested/3项目/分布式IM-雷漫/消息-IM详细设计--数据库缓存总结-三、Redis缓存数据结构设计.md]

### 4.8 用户设备在线状态管理

**用户状态字段**（Redis Hash: `1:2:im:status:userid:${userid}`）:
- `logintime:${deviceid}`: 登录时间（秒）
- `status:${deviceid}`: 在线状态 (0/1)
- `loginseq:${deviceid}`: 登录序列号
- `ssidfreshtime:${deviceid}`: 心跳刷新时间
- `sessionid:${deviceid}`: 会话ID
- `sessionkey:${deviceid}`: 加密密钥
- `accnodeidentify:${deviceid}`: 网关定位（格式：ip:port:workIndex）
- `clientip:${deviceid}`: 客户端IP
- `platformtype:${deviceid}`: 平台类型

**状态更新机制**:
- 登录时：更新所有相关字段
- 心跳时：刷新`ssidfreshtime`
- 登出时：更新`status`为0
- 超时时：自动将`status`设为0

## 2.2 MongoDB数据库设计

### 2.2.1 点对点消息表 (msg_c2c)

- **分片键**: userId
- **索引**: userId, msgId
- **主要字段**:
  - `userId`: 用户ID（等于fromId/toId，分片键）
  - `fromId`: 发送用户ID
  - `dstId`: 接收用户ID
  - `msgId`: 消息ID
  - `sendTime`: 消息产生时间（1970-微秒）
  - `msgType`: 消息类型
  - `content`: 消息内容字符串
  - `msgPb`: PB序列化数据
  - `msgState`: 消息状态（0:默认 1:xxx 2:送达 3:已读 4:撤回 5:删除自己 6:清空会话）
  - `msgseq`: 消息流水号（必须递增且连续）
  - `version`: 端到端密钥版本号
  - `platformType`: 平台类型

**数据特点**:
- 每条消息存储2份（A→B和B→A各一份）
- 支持消息状态追踪（送达、已读、撤回等）
- 支持端到端加密（version字段）

### 2.2.2 群消息表 (msg_c2g)

- **分片键**: groupId
- **索引**: groupId, msgId（建议复合索引：groupId+msgId）
- **主要字段**:
  - `fromId`: 发送用户ID
  - `groupId`: 群ID（分片键）
  - `msgId`: 消息ID（服务端产生，唯一）
  - `msgType`: 消息子类型
  - `content`: 消息内容字符串
  - `msgPb`: PB二进制消息
  - `msgState`: 消息状态（0:默认 3:已读 4:撤回 5:删除自己）
  - `msgseq`: 消息流水号（必须递增且连续）
  - `notifyType`: 是否为@消息（0:否 1:是）
  - `notifyUserId`: @的用户列表（JSON数组）
  - `opmsgId`: 引用消息ID

**未读数计算规则**:
```javascript
// 查询大于 lastReadMsgId 的未读数
const unreadCount = await db.msg_c2g.countDocuments({
  cmdType: { $in: [4501, 4505] },    // 只查聊天/通知消息
  fromId: { $ne: selfId },           // 排除自己发的
  groupId: groupId,
  msgId: { $gt: lastReadMsgId },
  msgState: { $ne: 4 }               // 排除撤回状态
});
```

未读消息数量表 (tb_no_read_sum)
- **用途**: 存储用户各平台的未读消息条数
- **字段**: user_id, user_type（1:IOS，2:ANDROID，3:MAC，4:PC）, no_read_sum

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint(20) unsigned | 自增id |
| user_id | bigint(20) unsigned | 用户id |
| user_type | int(11) | 用户类型1--IOS 2--ANDROID 3 mac 4--pc |
| no_read_sum | int(11) | 未读消息条数 |
| update_time | int(11) | 更新时间 |
| create_time | int(11) | 创建时间 |

// tb_no_read_sum（未读消息数量表）写入时机：
//
// 1. 用户离线（退出/切后台/IM下线）时：
//    - 获取当前用户所有有效会话的lastReadMsgId（已读点），对每个会话统计未读数。
//    - 将各会话未读数累加，合成totalUnread，写入tb_no_read_sum，记录最新未读数。
//    - 示例伪代码：
//      sessionList = 查询用户所有活跃会话ID列表（如所有群/groupId、单聊/targetId等）
//      totalUnread = 0
//      for sessionId in sessionList:
//          lastReadMsgId = 查该用户在会话的已读消息ID
//          count = db.countDocuments({ groupId/sessionId, msgId: { $gt: lastReadMsgId }, ... })
//          totalUnread += count
//      tb_no_read_sum.upsert({ user_id, user_type, no_read_sum: totalUnread, update_time: now })
//
// 2. 用户切换设备/跨平台同步或提醒未读：
//    - 可利用tb_no_read_sum快速查询各端（iOS/Android/PC等）的未读提醒数。
//    - 当需要展示全端未读数（如账号总未读、各端分端未读），直接从该表获取最新值。
//    - 某些场景下设备自动同步或轮询，也会触发写该表以保证未读提醒准确性。
//
// 3. 服务端定时任务或批量校正：
//    - 如需定期矫正全量未读数，可批量刷新该表，避免漏统计。
//
// 4. 特别说明：
//    - 在线期间通常不实时写tb_no_read_sum（减少DB压力），主要用于离线兜底、端切换提醒等场景。
//    - 新消息到来时只更新incr内存计数或redis，持久化同步通过上述时机批量/定点写入。

// 网关定时检测每个连接的心跳，如果超过超时时间未收到心跳包，则判定为异常离线，立即触发强制下线流程并进行未读数兜底：
// onHeartbeatTimeout(userId, userType):
//     // 心跳超时，说明客户端已异常断开，无法主动上报未读数。
//     // 此时由服务端主动统计当前所有会话的未读消息数。
//     allNoRead = realTimeCountUnread(userId)
//     // 将统计结果落库，确保下次登录时未读消息数准确。
//     persistNoReadSumToMySQL(userId, userType, allNoRead)
// 
// 这样即使客户端崩溃、被杀进程或断网，因心跳丢失被踢出时，也能实时写入tb_no_read_sum，无需等下次登录触发补算，提升了数据准确性和用户体验。

**复杂度分析**:
- 利用groupId+msgId复合索引，查询复杂度为O(logN) + O(K)
- N为群历史消息总数，K为未读区间实际条数
- 单次count操作通常小于5ms

### 2.2.3 会话状态表 (session_status)

- **分片键**: dataId
- **用途**: 保存会话状态，持久化msgseq，群会话还包括最后一条消息ID
- **主要字段**:
  - `dataId`: 用户ID或群ID（分片键）
  - `type`: 类型（1:用户 2:群）
  - `targetId`: 会话ID
  - `lastMsgId`: 会话的最后消息ID（目前只存群会话）
  - `maxMsgseq`: 会话预分配的消息序列号段最大值
  - `lastNodeid`: 预分配当前消息序列号段的结点ID
  - `nodeStarttime`: last_nodeid启用的时间

### 2.2.4 会话消息序号表 (session_msgseq)

- **分片键**: sessionId
- **用途**: 保存会话的消息序号下一个分配起点
- **主要字段**:
  - `sessionId`: 会话ID（分片键）
  - `msgseqStart`: 消息序号起点
  - `updateTime`: 更新时间（微秒）

### 2.3 mysql表
#### mysql表结构列表

**用户表 (tb_user)**:
- **分库字段**: user_id
- **索引**: user_id（唯一）、username（唯一）、mobile（唯一）
- **核心字段**: user_id, username, mobile, nickname, head_logo_url

**好友信息表 (tb_friend_info)**:
- **分库字段**: user_id
- **索引**: user_id, friend_id（联合唯一索引）
- **核心字段**: status, type, star, black, shield, close, origin

**群组表 (tb_group_info)**:
- **分库字段**: group_id
- **索引**: group_id（唯一索引）
- **核心字段**: group_name, logo_url, notice, group_auth, state

**群组成员表 (tb_group_member_info)**:
- **组合索引**: group_id, user_id
- **核心字段**: member_role, member_nickname, invite_type, invite_user_id

**未读消息数量表 (tb_no_read_sum)**:
- **用途**: 存储用户各平台的未读消息条数
- **字段**: user_id, user_type（1:IOS，2:ANDROID，3:MAC，4:PC）, no_read_sum
- **使用场景**: 用户离线时统计未读数写入MySQL，上线时直接查MySQL获取未读数

**CA公钥信息表 (tb_ca_info)**:
- **分库字段**: user_id
- **索引**: user_id, version（联合唯一索引）
- **用途**: 存储端到端加密的公钥信息
- **核心字段**: identity_key_pub/pri, signed_key_pub/pri, version

**加密对称密钥存储表 (tb_encrypted_symmetric_key)**:
- **分库字段**: user_id
- **索引**: session_id, user_id, version
- **用途**: 存储端到端加密通信中，各端设备间传递的加密后的对称密钥密文
- **核心字段**: session_id, sender_id, receiver_id, encrypted_key, key_version, key_type, expire_time, status

[src: raw/ingested/3项目/分布式IM-雷漫/数据-leiman_IM系统设计-二、数据库与缓存设计-二、数据库与缓存设计.md]