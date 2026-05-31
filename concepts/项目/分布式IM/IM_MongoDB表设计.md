# IM MongoDB表设计

> 本文档整理自雷漫分布式IM系统的MongoDB表设计，涵盖点对点消息、群消息、通知消息、会话状态、端到端加密密钥等表结构。

## 1. 公用信息

**数据库：** immsg

### 1.1 公用字段信息

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| fromId | uInt64 | | 发送用户id |
| toId | uInt64 | | 接受用户id |
| type | int | | 参考离线消息类型定义 |
| msgId | Int64 | | |
| serverTime | Uint8 | | |
| status | Int8 | | 默认0 |
| msgJson | String | | |

### 1.2 扩展字段信息

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| userId | uInt64 | | 等于fromId/toId， 用户的Id索引 |
| fromId | uInt64 | | 发送用户id |
| dstId | uInt64 | | 接受用户id |
| msgId | Int64 | | 服务器生成的唯一整数，一条消息两份，用同一个Id |
| cmdType | Int32 | | 消息命令字 |
| msgtype | Int32 | | 参考客户端消具体消息内容类型 |
| content | string | 3000 | 具体的消息内容字符串结构 |
| msgPb | bytes | | 整个pb序列化 |
| nodeStartTime | Int32 | | 节点启动时间（单位秒） |
| nodeId | string | | 节点值 |
| msgseq | Int32 | | 消息序列号 |
| clientmsgid | Int64 | | 客户端的消息Id，目前作用只在于消息重发时检查重发消息是否入库 |
| mobileStatus | Int32 | | 手机端拉取消息与否 |
| pcStatus | Int32 | | PC端拉取消息与否 |
| reserve | Int32 | | 保留 |
| updateTime | Int64 | | 消息文档更新时间 (单位微秒) |
| createTime | Int64 | | 消息文档插入时间 (单位微秒) |

## 2. 具体消息定义

### 2.1 点对点消息表:msg_c2c

**作用：** 点对点消息，2份，A->B

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| userId | uInt64 | | 等于fromId/toId， 索引（分片键） |
| fromId | uInt64 | | 发送用户id |
| dstId | uInt64 | | 接受用户id |
| dstType | int | | 目的类型 1 用户，2群 ，3系统 |
| msgId | Int64 | | |
| sendTime | Uint64 | | 消息产生时间（1970—微秒）服务端收到的时间 |
| status | Int8 | | 默认0，保留 |
| fromNickName | Varchar | | |
| fromHeadImg | Varchar | | 头像时戳 |
| cmdType | int | | 参考离线消息类型定义 |
| isTransmit | int | | 0:默认 0x00000001:回复类型信息 0x00000002:转发类型信息，按位表示消息的类型 |
| msgType | Int32 | | 消息类型 |
| content | String | | msgType对应的具体的内容字符串 |
| msgPb | char | | 整个pb序列化 |
| msgState | Int8 | 1 | 状态：0默认 1xxx  2送达3已读4撤回5删除自己6:清空会话内容（其中入库的状态消息分别是3、4、5）（删除好友也相当清空会话） |
| updateTime | | | 更新时间戳 |
| createTime | | | 创建入库时间 |
| mobileStatus | Int8 | | 移动端状态 默认-1 ，0收到 |
| pcStatus | Int8 | | 桌面端，同上 |
| reserve | Int8 | | 保留 |
| nodeStartTime | Int32 | | 节点启动时间 |
| nodeId | string | | 节点id |
| msgseq | Int32 | | 对该目的用户的消息流水，必须递增且连续 |
| version | int64 | 8 | 端到端密钥版本号,（也代表当前消息的版本）加密消息发送都应该带上，客户端根据消息版本号采用对应版本密钥解密 |
| platformType | Int32 | 4 | 消息发起者的所用设备平台类型客户登录客户端类型 0:未定义1:安卓 2:IOS 3:MAC 5:Windows; 聊天消息才有，状态消息，通知不没有本字段 |

### 2.2 端对端加密密钥消息表:keys_c2c

**作用：** 点对点消息，2份

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| userId | Int64 | | 用户id，索引（分片键） |
| dstId | uInt64 | | 接受用户id |
| dstType | int | | 目的类型 1 用户，2群 ，3系统 |
| msgType | Int32 | | 消息类型（通知子类型） |
| content | String | | 密钥消息 |
| msgPb | char | | 整个pb序列化 |
| msgState | Int8 | 1 | 状态：0默认 1删除 |
| createTime | | | 创建入库时间 |
| keyVersion | Int64 | | 密钥版本号（雪花Id） |
| reserve | Int8 | | 保留 |

### 2.3 通知消息表:msg_notify

**作用：** 保存通知消息

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| fromId | Int64 | | 发送用户id |
| dstId | Int64 | | 接受用户id， 索引（分片键） |
| msgId | Int64 | | 服务端产生消息id唯一 |
| serverTime | Int64 | | 消息产生时间（1970—微秒）服务端收到的时间 |
| status | Int32 | | 默认0，保留 |
| fromNickName | string | | 发送者名称 |
| fromHeadImg | string | | 发送者logo |
| msgType | Int32 | | 参考离线消息类型定义 |
| msg | string | | json消息体 |
| nodeStartTime | Int32 | | 节点启动时间 |
| nodeId | string | | 节点id |
| msgseq | Int32 | | 对该目的用户的消息流水，必须递增且连续 |

### 2.4 群通知消息表:groupmsg_notify

**作用：** 保存群通知消息（保留，暂未使用）

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| fromId | Int64 | | 发送用户id（填用户id） |
| dstId | Int64 | | 接受id（填群id） |
| msgId | Int64 | | 服务端产生消息id唯一 |
| serverTime | Int64 | | 消息产生时间（1970—微秒）服务端收到的时间 |
| status | Int32 | | 默认0，保留 |
| fromNickName | string | | 发送者名称 |
| fromHeadImg | string | | 发送者logo |
| msgType | Int32 | | 参考离线消息类型定义 |
| msg | string | | json消息体 |
| nodeStartTime | Int32 | | 目标启动时间（服务器填群对象在群服务的时间） |
| nodeId | string | | 目标id（服务器填群id）,string类型 |
| msgseq | Int32 | | 对该目的id的消息流水，必须递增且连续 |

### 2.5 群消息表:msg_c2g

**作用：** 群消息

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| fromId | Int64 | | 发送用户id |
| groupId | Int64 | | 时间+机器id+seq，索引（分片键） |
| cmdType | int | | 参考离线消息类型定义 |
| msgId | Int64 | | 服务端产生消息id唯一 |
| status | Int8 | | 默认0 |
| fromNickName | Varchar | | |
| fromHeadImg | Int32 | | |
| isTransmit | Int32 | 4 | 0:默认 0x00000001:回复类型信息 0x00000002:转发类型信息，按位表示消息的类型 |
| msgType | Int32 | | 消息子类型 |
| content | string | | msgType对应的具体的内容字符串 |
| msgPb | char | | Pb二进制消息 |
| clientMsgId | | | 客户端产生消息ID |
| msgState | Int8 | 1 | 状态：0默认 3已读4撤回5删除自己 (状态消息上来可能会对原始消息的增加状态字段，但只有撤回（全局删除）才会触发这个操作，删除自己的状态只会在状态消息那条记录里) |
| updateTime | Int64 | | 更新时间戳（1970—微秒） |
| createTime | Int64 | | 创建入库时间（1970—微秒） |
| nodeStartTime | Int32 | | 目标启动时间（这里用的是群内存对象创建时间） |
| nodeId | String | | 目标id（这里用的是群id）,string类型 |
| msgseq | Int32 | | 对该目的id（这里用的是群id）的消息流水，必须递增且连续 |
| isTrySend | Int32 | 4 | 0:否 1：重发 |
| notifyType | Int32 | 4 | 标识消息是否为@消息，0：否 1：是 |
| notifyUserId | array | 4 | Json数组，@的用户列表 |
| platformType | Int32 | 4 | 消息发起者的所用设备平台类型客户登录客户端类型 0:未定义1:安卓 2:IOS 3:MAC 5:Windows; 聊天消息才有，状态消息，通知不没有本字段 |
| opmsgId | Int64/Arrary | | 对于引用类型消息,cmdType=4501, opmsgId为单个值；对于状态消息，cmdType=4519, opmsgId结构为数组 |

**客户端离线上来拿到的"未读数"：从个人已读位置开始计算**
1. 【count(cmdType 为原始聊天/通知消息, fromid != self) – msgState==4的消息数】
2. "删除自己"的状态消息直接记录为一条状态消息，不参与未读数计算

### 2.6 群成员消息明细表:group_msg_details

**作用：** 群成员消息状态（目前不需要）

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| fromId | Int64 | 8 | 发送者id |
| dstId | Int64 | 8 | 接受者ID |
| groupId | Int64 | 8 | 群ID |
| msgId | Int64 | 8 | 消息ID 唯一，单调递增 |
| msgState | Int32 | 4 | 状态 ，默认0，3--已读 4--已撤 5--删除自己 |
| selfstate | int32 | 4 | 0未读， 1已读， 2 已删 |
| updateTime | Int32 | 4 | 更新时间 |
| createTime | Int32 | 4 | 创建时间 |
| serverTime | | | 消息生成时间 |

### 2.7 会话状态表:session_status

**作用：** 保存会话的状态，持久化msgseq，群的还包括最后一条消息ID

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| dataId | Int64 | 8 | 用户ID或者群ID， 索引（分片键） |
| type | Int32 | | 类型  1:用户    2：群 |
| targetId | string | | 会话id |
| lastMsgId | Int64 | 8 | 会话的最后消息ID |
| updateTime | Int64 | 8 | 会话最大消息ID更新时间 （微秒） |
| maxMsgseq | Int64 | | 会话预分配的消息序列号段最大值 |
| lastNodeid | String | | 配当前消息序列号段的结点id（用于每次判断结点是否变化） |
| nodeStarttime | Int32 | | last_nodeid启用的时间，结点不重启不变化 |

**备注：** 目前只存群会话最后一条消息的ID

### 2.8 语音播放消息记录表:played_voice_msg

| 字段名 | 类型 | 说明 |
|--------|------|------|
| _id | Long | 主键 |
| userId | Long | 用户id， 索引（分片键） |
| dstType | Integer | 会话类型 |
| dstId | Long | 会话对象id |
| msgId | Long | 语音消息id |
| createTime | Long | 创建时间 |

### 2.9 设备最后登录信息表:device_last_login_info

| 字段名 | 类型 | 说明 |
|--------|------|------|
| _id | | 主键（mongo自生成） |
| userId | int64 | 用户id，索引（分片键） |
| devId | String | 设备id |
| loginInfo | Json | JSON打包数据 { "loginTime":1596187168(s)} |
| updateTime | int64 | 更新时间，如果只需要时间可以取这个字段 |

### 2.10 会话消息序号表:session_msgseq

**作用：** 保存会话的消息序号下一个分配起点

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| sessionId | string | 8 | 会话id，索引（分片键） |
| msgseqStart | Int64 | | 消息序号起点 |
| updateTime | Int64 | | 更新时间（微妙） |

### 2.11 消息广播表:official_broadcast

**作用：** 消息广播

| 字段名 | 类型 | 备注 |
|--------|------|------|
| fromType | Int32 | 发送者类型：1用户id 2 群 3.系统 |
| fromId | Int64 | 发送 id（索引，分片键） |
| fromNickName | string | 发送者名称 |
| fromHeadImg | string | 发送者logo |
| msgId | Int64 | 服务端产生消息id唯一（这可以在后续看官方账号消息的量再补上索引） |
| cmdType | int | 参考离线消息类型定义 |
| msgType | Int32 | 消息类型 |
| language | string | 客户端语言版本，如en-US、zh-CN、default，匹配不上的，使用default的 |
| msgPb | char | 整个pb序列化 |
| sendTime | Uint64 | 消息产生时间（1970—微秒）服务端收到的时间 |
| status | Int32 | 默认0，1 无效 |
| dstRangeType | Int32 | 接收用户范围 0 全网用户 （目前只有本类型） 1 安卓 2ios 3 pc |
| updateTime | Int64 | 更新时间戳 |
| createTime | Int64 | 创建入库时间 |
| nodeStartTime | Int32 | 节点启动时间 |
| nodeId | string | 节点id |
| msgseq | Int32 | 对该目的用户的消息流水，必须递增且连续 |

**备注：** 看官方账号后续是否会扩展，有则官方账号fromId分片，否则用msgid

### 2.12 设备启动次数记录表:user_startup_count

**作用：** 设备启动次数上报
**备注：** 微服务加上分片/索引

| 字段名 | 类型 | 备注 |
|--------|------|------|
| deviceId | string | 设备Id，索引 |
| deviceType | | |
| phoneModel | | |
| systemVersion | | |
| createTime | | |
| updateTime | | |
| _class | | |

[src: raw/ingested/3项目/分布式IM-雷漫/数据-im数据库mysql-MongoDB表设计.md]