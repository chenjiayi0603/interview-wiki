# IM端到端加密与长连接认证

## 长连接认证和链路加密方案

### 登录认证流程

#### 客户端请求数据
- 包头：
- 包体：使用Protobuf序列化，包含rsaData和aesData
  - rsaData：包含userid、ecdhpubkey、aeskey（RSA加密）
  - aesData：包含token、clientTime、devid、loginseq等（AES加密）

#### 步骤一：RSA解密获取客户端公钥和数据
1. 服务器使用RSA私钥解密rsadata，得到AesKey
2. 使用AesKey解密aesdata，获取客户端数据

```cpp
if(!m_pAppLoginData->m_loginReq.ParseFromArray(m_oReqMsgBody.body().c_str(), m_oReqMsgBody.body().size()))
{
    return ERR_PARASE_PROTOBUF;
}
std::string strRsaData;
if(!Rsa2048Decrypt(m_pAppLoginData->m_loginReq.rsadata(), strRsaData,m_pLoginSession->GetRsaPrivateKey()))
{
    return ERR_RSADECRYPT_ERROR;
}
if(!m_pAppLoginData->m_pbRsaData.ParseFromString(strRsaData))
{
    return ERR_PARASE_PROTOBUF;
}
std::string strAesData;
if (!Aes256Decrypt(m_pAppLoginData->m_loginReq.aesdata(), strAesData,m_pAppLoginData->m_pbRsaData.aeskey()))
{
    return ERR_AESDECRYPT_ERROR;
}
if(!m_pAppLoginData->m_pbAesData.ParseFromString(strAesData))
{
    return ERR_PARASE_PROTOBUF;
}
```

#### 步骤二：返回加密的随机AES密钥和服务器公钥
1. 服务器启动时生成ECDH公私钥对
2. 使用客户端ECDH公钥和服务器ECDH私钥计算共享密钥
3. 用共享密钥加密服务器随机生成的AES会话密钥
4. 将加密后的会话密钥和服务器ECDH公钥返回客户端

```cpp
generate_key_pair_public_private_ecdh_string(m_ecdhServerPubkey, m_ecdhServerPrivateKey);
uint8_t *signalKey = calculate_ecdh_share_key(m_pAppLoginData->m_pbRsaData.ecdhpubkey().c_str(),m_pLoginSession->GetEcdhServerPrivateKey().c_str());

if(!Aes256Encrypt(m_pAppLoginData->m_aesSessionKey, strEncryptSessionkey, std::string((char*)signalKey, ECDHKEY_LEN)))

m_pAppLoginData->m_loginRsp.set_ecdhserverpubkey(m_pLoginSession->GetEcdhServerPubkey());
m_pAppLoginData->m_loginRsp.set_sessionkey(strEncryptSessionkey);

if(!Aes256Encrypt(m_pAppLoginData->m_loginRsp.SerializeAsString(), strEncryptLoginRsp, m_pAppLoginData->m_pbRsaData.aeskey()))
```

#### 后续加密密钥
- aeskey1解密AES数据
- Aeskey2 = ECDH(client_pri_key, server_pub_key)
- Sessionkey = AES_decode(aeskey2, sessionkey)
- 后续AES加密密钥：sessionKey
- Auth_key = sessionKey
- 通讯消息加密方法：AES256 CBC模式 + HMAC-SHA256校验

## 端对端加密单聊

### CA中心服务维护公钥

CA服务保存的用户信息：
- 用户唯一标识
- 公钥（RSA2048或ECDH）及有效期、类型、更新版本

#### 用户新建公钥上传
1. 客户端随机生成一对ECDH公钥
2. 注册时提交公钥，服务端保存到IM CA服务中心
3. 登录时提交；如果是第一次在一台设备登录，满足重置公私钥条件，则服务端保存最新公钥映射

#### 更新步骤
1. 客户端产生新的密钥对，计算公钥、有效期等数据的SHA哈希值，用客户端原有私钥签名
2. 客户端将新公钥、有效期、签名等上传到CA服务
3. CA服务使用SHA计算公钥、有效期等数据得出哈希值，使用该用户的旧公钥做签名验证
4. CA服务确认通过后，更新新密钥相关信息
5. 客户端开始使用新密钥，老密钥对保留一段时间后删除

### 单聊端对端方案

密钥产生/更新：ECDH，各个用户拉取对方用户的公钥，对方用户的公钥和自己用户的私钥生成的密钥来作为对称密钥；每个用户每个版本提交一个公钥；每个版本一周过期。

#### 密钥协商流程
1. 用户A从CA服务中心根据用户B的标识获取B的密钥相关信息（可本地缓存）
2. 用户A判断用户B的公钥是否有效
3. 用户A计算产生一个真随机数作为对称加密密钥
4. 用户A使用用户B的公钥和临时密钥对产生加密密钥，对消息密钥加密，且对密钥相关信息产生HmacSHA256值
5. 用户A将加密结果和签名结果发送服务器
   - 如果服务器发现该会话有密钥且没有过期（暂定7天过期），且A、B的公钥没有过期，则返回该会话现有的密钥相关信息给A，A收到后解密且用该密钥作为自己的密钥；服务器不再进行转发
   - 如果服务器发现该会话目前密钥过期则对A进行身份验证后，服务器进行消息拆分，保存入库，同时转发给B用户各设备和A用户各设备
6. 用户B从CA服务中心获取用户A的密钥相关信息
7. 用户B判断用户A的公钥有效期是否有效
8. 用户B接收消息后取出加密结果和HmacSHA256信息，使用自己本地的私钥和A的公钥信息根据计算算法进行解密获得密钥相关信息，并进行SHA验证校验
9. 用户A获得后同样的计算方法获取密钥和进行校验

#### 收发消息
1. A或B收发消息时，根据产生的端对端密钥和相关算法进行解密或加密
2. 两端可以将该对称密钥存储到本地数据库中保存，使用该对称密钥解密传输数据
3. 两端任意一端可以根据服务端返回的应答更新对称密钥，只需要按照密钥协商传输规则产生新的密钥提交服务器

### 数据库表结构

#### MongoDB：端对端加密密钥消息表 keys_c2c
| 字段名 | 类型 | 备注 |
|--------|------|------|
| userId | Int64 | 用户id，索引（分片键） |
| dstId | uInt64 | 接受用户id |
| dstType | int | 目的类型 1 用户，2群 ，3系统 |
| msgType | Int32 | 消息类型（通知子类型） |
| content | String | 密钥消息 |
| msgPb | char | 整个pb序列化 |
| msgState | Int8 | 状态：0默认 1删除 |
| createTime | | 创建入库时间 |
| keyVersion | Int64 | 密钥版本号（雪花Id） |
| reserve | Int8 | 保留 |

#### MySQL：tb_ca_info CA公钥信息表
| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint(20) | 主键ID |
| user_id | bigint(20) | 用户id |
| identity_key_pub | varchar(255) | hex身份公钥 |
| identity_key_pri | varchar(255) | hex身份公私钥（aes加密保存） |
| signed_key_pub | varchar(255) | hex签名后公钥 |
| signed_key_pri | varchar(255) | hex签名后私钥（aes加密保存） |
| version | int(11) | 用户公私更新的版本 |
| ext | varchar(255) | 扩展 |
| state | tinyint(3) | 状态 1.正常 0.无效 |
| create_time | int(11) | 创建时间 1970—秒 |
| update_time | int(11) | 更新时间1970—秒 |

索引：
- idx_userid_version: user_id,version 唯一索引
- idx_userid: user_id

分库表字段：user_id
所属库描述：im_ca
分库格式：im_ca_x

## 消息结构

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| fromId | Int64 | 8 | 发送ID |
| fromNickname | string | | 发送者昵称 |
| fromHeadImg | string | 4 | 发送者头像url 可选 |
| dstId | Int64 | 8 | 接收ID |
| dstType | Int32 | 4 | 1:dstId是用户id 2：dstId是群id |
| msgId | Int64 | 8 | 消息id |
| sendTime | Int64 | 8 | 客户端发送时间 |
| isTransmit | Int32 | 4 | 0:否 1：是转发消息 |
| msgType | int32 | 4 | 参考消息定义 |
| msg | string | 3000 | 消息体内容 |
| clientMsgId | Int64 | 8 | |
| createSession | Int32 | | 检查和创建会话 |
| nodeStartTime | Int32 | 4 | 节点启动时间 |
| nodeId | string | | 会话id [string(“Min(fromId,dstId)_Max(fromid,dstId)”)] |
| msgseq | Int32 | 4 | 对该目的用户的消息流水，必须递增且连续 |
| isTrySend | Int32 | 1 | 0:否 1：重发 |
| keyVersion | int64 | 8 | 端到端密钥版本号 |

## MongoDB表结构msg_c2c

| 字段名 | 类型 | 长度 | 备注 |
|--------|------|------|------|
| userId | uInt64 | | 等于fromId/toId，索引（分片键） |
| fromId | uInt64 | | 发送用户id |
| dstId | uInt64 | | 接受用户id |
| dstType | int | | 目的类型 1 用户，2群 ，3系统 |
| msgId | Int64 | | |
| sendTime | Uint64 | | 消息产生时间（1970—微秒）务端收到的时间 |
| status | Int8 | | 默认0，保留 |
| fromNickName | Varchar | | |
| fromHeadImg | Varchar | | 头像时戳 |
| cmdType | int | | 参考离线消息类型定义 |
| isTransmit | int | | 0:默认 0x00000001:回复类型信息 0x00000002:转发类型信息 |
| msgType | Int32 | | 消息类型 |
| content | String | | msgType对应的具体的内容字符串 |
| msgPb | char | | 整个pb序列化 |
| msgState | Int8 | 1 | 状态：0默认 1xxx 2送达3已读4撤回5删除自己6:清空会话内容 |
| updateTime | | | 更新时间戳 |
| createTime | | | 创建入库时间 |
| mobileStatus | Int8 | | 移动端状态 默认-1 ，0收到 |
| pcStatus | Int8 | | 桌面端，同上 |
| reserve | Int8 | | 保留 |
| nodeStartTime | Int32 | | 节点启动时间 |
| nodeId | string | | 节点id |
| msgseq | Int32 | | 对该目的用户的消息流水，必须递增且连续 |
| version | int64 | 8 | 端到端密钥版本号 |
| platformType | Int32 | 4 | 消息发起者的所用设备平台类型 |

## 微服务获取密钥端对端流程

### 通信步骤
1. 客户端使用通信密钥加密请求正文，加密方式选择AES对称加密算法
2. 客户端将加密的数据使用POST形式调用接口，加密内容作为请求体提交，请求参数包括：加密密文、用户ID、设备ID、语言类型
3. 服务端接收参数后：
   a) 验证参数是否完整
   b) 判断是否URL放行
   c) 判断是否需要进行加密
   d) 根据用户ID加设备编号查询本地LRUCache缓存
   e) 如果查询到相应键值，则使用密钥尝试解锁，如果解锁成功，则刷新缓存
   f) 如果查询为空或解锁失败，则根据用户ID+设备编号查询Redis，得到用户登录状态中的Hash值为“sessionkey”的通信密钥
   g) 如果Redis缓存查询结果为空，则返回密钥不存在的异常
   h) 如果使用Redis中的通信密钥进行数据解密仍然失败，则返回非法访问
   i) 如果解密成功，则将密钥保存到缓存中
   j) 判断token是否登录过期
4. 业务处理完毕后，服务端使用通信密钥将返回结果中的数据进行加密并返回

[src: raw/ingested/3项目/分布式IM-雷漫/登录-leiman_加密.md]