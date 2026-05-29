# IM用户Login缓存(hash)

## 3.2 用户Login 缓存(hash)

同一个用户的多个设备信息存在一个hashkey

**Key格式**: `1:2:im:status:userid:${userid}`

**数据结构**: Hash

**说明**: 消息平台与微服务共享的缓存，微服务平台创建key并设置key的有效期。消息平台在客户端登录时刷新其中的域值并加入必要Field，但不会删除其中域值。

**Hash字段**:

| Field | 类型 | 说明 |
|-------|------|------|
| `logintime:${deviceid}` | int32 | 登录时间（秒），每次TCP重建都需要刷新 |
| `clienttype:${deviceid}` | int32 | 客户端类型（0/1/2/3），用于推送平台识别，配合newmsgtip判断离线是否推送到某个平台（android, ios, Huawei…） |
| `status:${deviceid}` | int32 | 在线/离线状态（0/1），主要是在线离线状态维护，消息路由转发也要判断 |
| `loginseq:${deviceid}` | int64 | 登录序列号，防重登攻击，在分布式场景下，这个seq可以阻止同一个设备“seq <= loginSeq”的登录请求 |
| `ssidfreshtime:${deviceid}` | int32 | 心跳刷新时间（秒），微服务查询最近登录时间可以取这个时间比logintime更合理些 |
| `sessionid:${deviceid}` | int64 | 会话ID，由Login生成要带回给Access，代表客户端在Access的链路唯一标识，防止分布式的消息包路由错误 |
| `sessionkey:${deviceid}` | string | 对称加密key，32字节的字符串，缓存共享可以使得加解密分散给各逻辑服务，避免Access集中加解密耗时 |
| `accnodeidentify:${deviceid}` | string | 用户登录的网关定位标识，格式：`ip:port:workIndex` |
| `clientip:${deviceid}` | string | 客户端IP |
| `platformtype:${deviceid}` | int32 | 平台类型（1:Android 2:Ios 3:Mac 4:windows…） |
| `expire:${deviceid}` | - | sessionkey过期时间，一段时间内的登录使用的是同一个sessionkey |
| `language:${deviceid}` | string | 客户端语言版本，如en-US、zh-CN、default，匹配不上的使用default |

**编程对应Redis缓存的数据结构定义**:

```cpp
typedef struct{
    int32	loginTime = 0;
    int32 	clientType = 0;
    int32 	status = 0;
    int64	loginSeq = 0;
    int32	ssidFreshTime = 0;
    int64  	sessionId = 0;
    std::string	sessionKey;
    std::string accNodeIdentify;
} UserDeviceInfo;
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-leiman_im_redis数据结构-3.2.用户Login-缓存(hash)----.md]