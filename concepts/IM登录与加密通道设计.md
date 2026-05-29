# IM 登录与加密通道设计

## 1. 设计目标与范围
- 描述 IM 客户端从接入到登录鉴权、会话加密、状态维护的整体流程。
- 定义跨消息平台与微服务共享的登录缓存结构，便于路由转发与安全校验。

[src: raw/ingested/3项目/分布式IM-雷漫/登录-leiman---IM登录与加密通道设计.md]

## 2. 登录缓存模型（Redis Hash）
- Key：`1:2:im:status:userid:{uid}`
- Hash 字段（按设备维度存储，示例字段名均拼接 deviceId）：
  - `logintime:{deviceId}`：最近登录时间（秒），TCP 重建需刷新。
  - `clienttype:{deviceId}`：0/1/2/3，用于推送平台识别（Android/iOS/Huawei…）。
  - `status:{deviceId}`：0/1/2 = online/busy/leave，影响路由与推送。
  - `loginseq:{deviceId}`：防重登序列，分布式下拒绝 `seq <= loginSeq` 的登录。
  - `ssidfreshtime:{deviceId}`：最近心跳时间（秒），用于判断会话有效性。
  - `sessionid:{deviceId}`：Login 生成的链路标识，Access 侧唯一连接 ID。
  - `sessionkey:{deviceId}`：对称加密 Key（32 字节），供各逻辑服务解密。
  - `accnodeidentify:{deviceId}`：接入网关定位 `ip:port:workIndex`。
  - `newmsgtip:{deviceId}`：0/1，消息提醒开关（来源于微服务数据库）。
- 说明：`ssidfreshtime` 随客户端心跳刷新；路由转发时若 `now - ssidfreshtime > N 分钟`，会话视为失效。

### 2.1 数据结构示例
```
typedef struct{
    int32  loginTime;
    int32  clientType;
    int32  status;
    int64  loginSeq;
    int32  ssidFreshTime;
    int64  sessionId;
    string sessionKey;
    string accNodeIdentify;
    int32  newmsgtip;
} UserDeviceInfo;
```

[src: raw/ingested/3项目/分布式IM-雷漫/登录-leiman---IM登录与加密通道设计.md]

## 3. 登录流程
### 3.1 流程概览
```
// 
//      ┌────────┐      ┌────────────┐      ┌──────────┐            ┌────────────┐
//      │        │─────▶│            │─────▶│          │            │            │
//      │ 客户端 │      │  Access    │      │  Router  │───────────▶│   Login    │
//      │        │      │            │      │          │            │            │
//      └────────┘      └────────────┘      └──────────┘            └────────────┘
//             │             ▲
//             │             │                                          
//             ▼             │                          
//         ┌───────────────┐ │      ┌────────────────┐   ┌─────────────┐ 
//         │ 解密/校验Token│◀┘      │ 生成会话密钥    │──▶│ 加密下发Rsp │ 
//         └───────────────┘         └────────────────┘   └─────────────┘ 
//                  │                        │
//               ┌──┴────────────────────────┘
//               │
//           ┌─────保存sessionKey────────────┐
//           │                               │
//        ┌─────────────────────┐         ┌────────────┐
//        │ sessionKey通信加密/ │─────────▶│ 客户端通信 │
//        │ 消息分发/缓存/踢人  │          └────────────┘ 
//        └─────────────────────┘                
// 
// 
// 说明：
// - sessionKey表示安全通道密钥
// - 箭头表数据/密钥/登录信息流程
// - “消息分发/缓存/踢人”为后续链路安全维护与断链处理
```

---

1. Client 建立到 Access 的 TCP 连接。
2. Client 发送登录请求；Access 转内部协议（若在 Access 加解密，则先对称解密再转 PB）。
3. Access → Router → Login 路由登录请求。
4. Login 服务端详细解密流程如下：
   - 首先使用服务端私钥进行 RSA 解密，解析出客户端登录请求中的 AES 对称密钥、userId、ecdh 公钥等敏感字段（详细字段结构见《加密流程》）。
   - 随后，利用解密得到的 AES 密钥对剩余业务参数（如 token、clientTime、deviceId 等）进行 AES 解密，获得完整的登录信息。
   - 整个过程确保 token、userId、deviceId 等关键信息均在安全通道中传递，经端到端加密与分级解密后还原，保障鉴权安全。
   - 相关密钥与流程请参考下方“长连接加解密流程”章节。

5. Login 读取缓存，对比 token 合法性。
6. 成功则更新缓存结构（含 sessionId、sessionKey 等）；失败也要回客户端。
7. 成功时生成对称密钥 sessionKey（ECDH），封装到 loginRsp 返回给 Client，经 Access 存储 `<sessionId, socket, sessionKey>`。
8. Access 将结果回传客户端。

### 3.2 错误码
- `20000` ERR_PARASE_PROTOBUF：解析 Protobuf 出错
- `20031` ERR_KEY_FIELD_VALUE：缓存指定字段缺失/为空
- `20224` ERR_GENERATE_ECHDKEY_ERROR：生成对称密钥错误
- `20226` ERR_RSADECRYPT_ERROR：RSA 解密错误
- `20227` ERR_AESDECRYPT_ERROR：AES 解密错误
- `21007` ERR_CHECK_PRE_LOGIN_TOKEN：预登录 token 校验失败
- `21030` ERR_LOGIN_REPEATED：重复登录（loginSeq 未增长）

[src: raw/ingested/3项目/分布式IM-雷漫/登录-leiman---IM登录与加密通道设计.md]

## 4. 加密通道
### 4.1 建立方式
- 与登录流程同一时序内完成：Login 产生最终 sessionKey，返回给 Client，并通过 Access 存储。

### 4.2 使用策略
- 对称加解密可在 Access 集中处理，也可按业务分散到逻辑服务。
- Access 需在连接对象中保存 sessionKey；上行统一 Decode 解密后再走内部 PB，回包 Encode 时用 sessionKey 加密。

[src: raw/ingested/3项目/分布式IM-雷漫/登录-leiman---IM登录与加密通道设计.md]

## 5. 在线状态维护
- `status` 直接影响消息路由/推送，需准确维护。
- 主要场景：
  - Login：默认不触发踢人，status 不变。
  - Logout：客户端退出，微服务通知长连后台断开并更新缓存。
  - Kickout：管理端踢设备下线，通知长连后台并更新缓存。
  - Client 退后台/关闭：后台收到 close 信号后更新状态。
  - 客户端异常：Access 自动检测无效链路，清理并更新状态。
  - Access 宕机：无 close 信号时，依赖 `ssidfreshtime` 超时判断 session 失效。

[src: raw/ingested/3项目/分布式IM-雷漫/登录-leiman---IM登录与加密通道设计.md]

## 6. 图示占位
- 登录流程图：`<a>`（待补）
- 加密通道图：`<b>`（待补）
- 加解密流程图：`<c>`（待补）

[src: raw/ingested/3项目/分布式IM-雷漫/登录-leiman---IM登录与加密通道设计.md]