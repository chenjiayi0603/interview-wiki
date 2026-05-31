# IM群消息协议定义

> 本文档整理自雷漫分布式IM系统的群消息协议定义，涵盖群消息发送/接收、消息状态、输入状态、订阅、群通知、群成员状态拉取及官方广播等协议细节。

## 3.4 发送群消息 APP->TS （cmd=4501）

协议报文同单聊。

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| fromId | Int64 | 8 | 发送ID |
| fromNickname | string | | 发送者昵称 |
| fromHeadImg | string | 4 | 发送者头像url 可选 |
| dstId | Int64 | 8 | 接收ID |
| dstType | Int32 | 4 | 1:用户id 2：群id |
| sendTime | Int64 | 8 | 客户端发送时间微秒 |
| isTransmit | Int32 | 4 | 0:否 1：是转发消息 |
| msgType | int32 | 4 | 参考消息定义 |
| msg | string | 3000 | 消息体内容 |
| clientMsgId | Int64 | 8 | |
| createSession | Int32 | | 检查和创建会话 |
| isTrySend | Int32 | 1 | 0:否 1：重发 |
| notifyUserId | Int64 (Array) | | @群成员， 非空则代表@某人 |
| notifyMsgId | Int64 | 8 | 针对群某单条消息回复，会产生@某成员，回复的那个成员id填在notifyUserId |

**ACK TS->APP （cmd=4502）**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| code | Int32 | 4 | 参考返回码 |
| codeMsg | string | 128 | 返回描述 |
| serverMsgId | uint64 | 8 | 消息id |
| serverTime | Uint64 | 8 | 服务器时间戳(单位:) |
| clientMsgId | uint64 | 8 | 客户端产生的消息id |
| nodeStartTime | Int32 | | 节点启动时间（这里用的是群内存对象创建时间） |
| nodeId | string | | 节点id（这里用的是群id） |
| msgseq | Int32 | | 对该目的id（这里用的是群id）的消息流水，必须递增且连续 |

## 3.6 接收群消息 TS->APP（cmd=4503）/ 系统通知的群聊消息（cmd=4527）/ 系统通知的群通知（cmd=4531）

基本同单聊。

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| fromId | Int64 | 8 | 发送ID |
| fromNickname | string | | 发送者昵称 |
| fromHeadImg | string | 4 | 发送者头像url 可选 |
| dstId | Int64 | 8 | 接收ID |
| dstType | Int32 | 4 | 1:用户id  2 群id |
| msgId | Int64 | 8 | 消息id |
| sendTime | Int64 | 8 | 服务器时间戳微秒 |
| isTransmit | Int32 | 4 | 0:否 1：是转发消息 |
| msgType | int32 | 4 | 参考消息定义 |
| msg | string | 3000 | 消息体内容 |
| clientMsgId | Int64 | 8 | |
| createSession | Int32 | | 检查和创建会话 |
| nodeStartTime | Int32 | | 节点启动时间（这里用的是群内存对象创建时间） |
| nodeId | string | | 节点id（这里用的是群id） |
| msgseq | Int32 | | 对该目的id（这里用的是群id）的消息流水，必须递增且连续 |
| isTrySend | Int32 | 1 | 0:否 1：重发 |

**ACK APP->TS （cmd=4504）/ 系统通知的群聊消息响应（cmd=4528）/ 系统通知的群通知响应（cmd=4532）**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| code | Int32 | 4 | 参考返回码 |
| codeMsg | string | 128 | 返回描述 |
| msgId | Int64 | 8 | 消息id（服务端生成的） |
| groupId | Int64 | 8 | |
| nodeStartTime | Int32 | | 节点启动时间（这里用的是群内存对象创建时间） |
| nodeId | string | | 节点id（这里用的是群id） |
| msgseq | Int32 | | 对该目的id（这里用的是群id）的消息流水，必须递增且连续 |

## 3.10 （状态服务）发送输入状态(无需保存不计数) APP->TS (cmd=4513)

**请求 APP->TS(cmd=4513)**

| 字段名 | 数据类型 | 默认必选 | 长度 | 备注 |
|--------|----------|----------|------|------|
| fromId | Int64 | | 8 | 发送ID |
| dstType | | | | 类别1：用户id2：群id3：系统 |
| dstId | Int64 | | 8 | 接收ID |
| state | Int32 | | 4 | 1：输入完成2：输入中3：录音中 4.视频 5.图片 其他扩展 |
| msg | string | 可选 | | 拓展字段。Json格式 {"nickName":"昵称","sendTime":客户端消息发送时间（单位微妙）} |

**返回ACK (cmd=4514)**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| code | Int32 | 4 | 参考返回码 |
| codeMsg | string | 128 | 返回描述 |

## 3.11 接收输入状态TS->app（cmd=4515）

| 字段名 | 数据类型 | 默认必选 | 长度 | 备注 |
|--------|----------|----------|------|------|
| fromId | Int64 | | 8 | 发送ID |
| dstType | | | | 类别1：用户id 2 群id |
| dstId | Int64 | | 8 | 接收ID 为用户id |
| state | Int32 | | 4 | 1：输入完成2：输入中3：录音中4.视频 其他扩展 |
| msg | string | 可选 | | 拓展字段。Json格式 {"nickName":"昵称","sendTime":客户端消息发送时间（单位微妙）} |

**App->TS: 返回ACK (cmd=4516)**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| code | Int32 | 4 | 参考返回码 |
| codeMsg | string | 128 | 返回描述 |

## 3.14 发送群消息状态APP->TS （cmd=4517）

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| fromId | Int64 | 8 | 发送ID |
| dstId | Int64 | 8 | 接收ID |
| msgState | Int32 | 4 | 3 :已读 4：撤回  5：删除自己6:清空会话内容7：上报语音已播状态 8：群"搜索场景"上报已读 |
| opmsgid | Int64 array | 8 | 消息id（服务端生成的），撤回/删除可以填多个 |
| sendTime | Int64 | 8 | 客户端发出状态消息的时间 |
| msgUnReadedNums | Int32 | 4 | 用于群会话在线同步未读数 |
| lastMsgId | Int64 | 8 | 群会话上报未读数时，顺便把当前群会话离线获取到的最后一条消息ID上传 |
| msgNotifyReadedNums | int32 | 4 | 用于群会话在线同步@未读数，群会话拉消息存在部分的场景，只用于多端在线同步群会话未读数 |

**ACK TS->APP （cmd=4518）**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| code | Int32 | 4 | 参考返回码 |
| codeMsg | string | 128 | 返回描述 |
| msgId | Int64 | 8 | 服务器生成的消息ID |
| nodeStartTime | Int32 | | 服务端产生 |
| nodeId | string | | 服务端产生 |
| msgSeq | Int32 | | 服务端产生 |
| serverTime | Int64 | | 服务端开始处理这条消息的时间 |
| lastMsgId | int64 | 8 | 群会话上报未读数时，顺便把当前群会话离线获取到的最后一条消息ID上传,原样带回 |
| extend | string | | 扩展字段封装到json里，此处extend = {"unReadMsgNum":0, "unReadAtMsgNum":0} |

## 3.15 接收群消息状态TS->APP （cmd=4519）

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| fromId | Int64 | 8 | 发送ID |
| dstId | Int64 | 8 | 接收ID |
| msgState | Int32 | 4 | 2：已送达 3 :已读 4：撤回 5：删除自己6:清空会话内容 7：上报语音已播状态 8：群"搜索场景"上报已读 |
| msgId | Int64 | 8 | 消息id（服务端生成的） |
| opmsgid | Int64(array) | | 撤回/删除可以填多个 |
| nodeStartTime | Int32 | | 服务端产生 |
| nodeId | string | | 服务端产生 |
| msgSeq | Int32 | | 服务端产生 |
| sendTime | Int64 | | 服务端开始处理这条消息的时间 |
| msgUnReadedNums | Int32 | 4 | 用于群会话在线同步未读数，群会话拉消息存在部分的场景，只用于多端在线同步群会话未读数 |
| lasMsgId | Int64 | 8 | 群会话上报未读数时，顺便把当前群会话离线获取到的最后一条消息ID上传，只用于多端在线同步群会话未读数 |
| msgNotifyReadedNums | int32 | 4 | 用于群会话在线同步@未读数，群会话拉消息存在部分的场景，只用于多端在线同步群会话未读数 |

**在线未读数：客户端要根据情况自己计算**
1. 聊天/通知消息到来，在线接到会话未读数加1；
2. 已读状态消息不计数，但接到已读状态消息也要处理未读数，fromid==self, 大于本地自己已读位置，要更新会话的未读数；
3. 在线收到撤回消息，未读数减1（前提要判断此消息是否已读过）；
4. "删除自己"状态消息一开始就没算到未读数里边，所以"删除自己"状态消息不引起未读数变化。

**ack APP->TS（cmd=4520）**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| msgId | int64 | | 消息id |

## 3.13 订阅获取类消息APP->TS (cmd=4521)

消息平台收到后，根据订阅。

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| fromId | Int64 | 8 | 发送ID，上报者 |
| dstId | Int64 | | 接受ID 群id |
| subCmd | Int32 | | 子命令：100 订阅群人数，和状态 |
| msg | string | | Json 具体命令请求信息补充内容，具体根据subCmd指定。{"overtime":100 //有效时间，单位秒（超时续约需要重新订阅）} |

**ACK TS->APP （cmd=4522）**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| code | Int32 | 4 | 参考返回码 |
| codeMsg | string | 128 | 返回描述 |
| dstId | Int64 | | 接受ID 群id |
| msg | string | | 具体被订阅数据内容json，示例：{"total_num":50,"online_num":50} |

订阅成功后，会返回订阅的数据，并使用户处于订阅状态。用户离线（登出、或者断开连接），会消除订阅状态。处于订阅状态时，被订阅的数据有改动时，会推送订阅数据到该用户（cmd=4523）。

## 3.14 订阅数据推送TS->app (cmd=4523)

消息平台收到后，根据订阅，下发推送信息：如状态。

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| dstId | Int64 | | 接受ID 群id |
| subCmd | Int32 | | 子命令：100 订阅群人数 |
| msg | string | | 具体被订阅数据内容json，示例：{"total_num":50,"online_num":50} |

**ACK TS->APP （cmd=4524）**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| code | Int32 | 4 | 参考返回码 |
| codeMsg | string | 128 | 返回描述 |

## 3.15 群通知 TS->APP （cmd=4525）（暂时不用）

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| fromType | Int64 | | 发送者类型：1用户id 2 群 3.服务 |
| fromId | Int64 | 8 | 发送ID |
| fromNickname | string | | 发送者昵称 |
| fromHeadImg | string | | 头像url 可选 |
| dstId | Int64 | 8 | 接收ID |
| sendTime | Int64 | 8 | 服务器时间 微秒 |
| msgId | Int64 | 8 | 消息ID（服务器生成） |
| noticeType | Int32 | 4 | 参考通知类型定义 |
| msg | string | 2000 | 消息体内容json格式 |
| nodeStartTime | Int32 | | 目标启动时间（这里用的是群内存对象创建时间） |
| nodeId | string | | 目标id（这里用的是群id） |
| msgseq | Int32 | | 对该目的id（这里用的是群id）的消息流水，必须递增且连续 |

**ACK APP->TS （cmd=4526）（暂时不用）**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| msgId | Int64 | 8 | 消息ID（服务器生成） |

## 3.16 拉取群成员状态TS->app (cmd=4529)

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| fromId | Int64 | | 请求者id |
| userIds | array | 8 | 获取这些用户的状态（有指定用户列表的则获取用户列表的，否则获取群的所有的成员的） |
| groupId | Int64 | | 群id |

**ACK TS->APP(cmd=4530)**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| code | Int32 | 4 | 参考返回码 |
| codeMsg | string | 128 | 返回描述 |
| dstId | Int64 | 8 | 接收者ID |
| groupId | Int64 | | 群id |
| msg | String | | 拉取群成员状态，示例：json数组[{uid：12333,"state":1,"logintime":1545986177}] |

## 4 官方广播

### 4.1 官方广播通知客户端消息TS->APP （cmd=5141）

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| fromId | Int64 | 8 | 发送ID |
| fromNickname | string | | 发送者昵称 |
| fromHeadImg | string | 4 | 发送者头像url 可选 |
| dstId | Int64 | 8 | 接收ID |
| dstType | Int32 | 4 | 1:dstId是用户id 2：dstId是群id 3：系统 |
| msgId | Int64 | 8 | 消息id |
| sendTime | Int64 | 8 | 客户端发送时间 |
| isTransmit | Int32 | 4 | 0:否 1：是转发消息 |
| msgType | int32 | 4 | 参考消息定义 |
| msg | string | 3000 | 消息体内容 |
| clientMsgId | Int64 | 8 | |
| createSession | Int32 | | 检查和创建会话 |
| nodeStartTime | Int32 | 4 | 节点启动时间 |
| nodeId | string | | 会话id [string("Min(fromId,dstId)_Max(fromid,dstId)")] |
| msgseq | Int32 | 4 | 对该目的用户的消息流水，必须递增且连续 |
| isTrySend | Int32 | 1 | 0:否 1：重发 |
| keyVersion | int64 | 8 | |

**ACK TS->APP （cmd=5142）**

| 字段名 | 数据类型 | 长度 | 备注 |
|--------|----------|------|------|
| code | Int32 | 4 | 参考返回码 |
| codeMsg | string | 128 | 返回描述 |
| msgId | uint64 | 8 | 消息id（服务器生成） |
| serverTime | int64 | 8 | 服务器时间戳(单位: 微秒) |
| clientMsgId | uint64 | 8 | 消息id |
| nodeStartTime | int32 | 4 | 服务端返回 |
| nodeId | string | | 服务端返回 |
| msgseq | int32 | 4 | 服务端返回，消息如果能顺利转发，需要在ACK里带回服务端生成的序列号 |
| keyUpdated | int32 | 4 | |

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-3-群.md]