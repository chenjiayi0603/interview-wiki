# Login设计

IM  Login设计
修订记录：

# 登录缓存设计

消息平台与微服务共享的缓存cash，微服务平台创建key并skey的有效期。消息平台在客户端登录时刷新其中的域值并加入必要Field,但不会删除其中域值。
Key:  1:2:im:status:userid:100122
Hash:
{
“logintime:deviceid”  		1578987836 //秒，每次tcp重建都需要刷新
“clienttype:deviceid”  		0/1/2/3 //用于推送平台的识别,配合newmsgtip判断离线是否推送到某个平台（android, ios, Huawei…）
“status:deviceid”  		0/1/2  //online/busy/leave，主要是在线离线状态维护，消息路由转发也要判断，影响这个状态的场景相对较多，后续列举
“loginseq:deviceid” 		100000,  //防重登攻击，在分布式场景下，这个seq可以阻止同一个设备 “seq <= loginSeq”的登录请求
“ssidfreshtime:deviceid”  	15344838// 秒，依据心跳频率刷新，微服务其实查询最近登录时间可以取这个时间比logintime更合理些
“sessionid:deviceid”  		123222222//由Login生成要带回给Access，代表客户端在Access的链路唯一标识，防止分布式的消息包路由错误
“sessionkey:deviceid”  	“abbcdefghijk”, //这个是对称加密key, 32字节的字符串（缓存共享可以使得加解密分散给各逻辑服务，避免Access集中加解密耗时，目前也要带回给Access，由Access统一加解密）
“accnodeidentify:deviceid”   “ip:port:workIndex” //用户登录的网关定位标识
“newmsgtip:devieid”		   0/1 //消息开关提醒，来源于微服务的数据库
}
注：其中deviceid代表的是具体的设备值，如Field
“ssidfreshtime:7A5E955A-95BB-4C8E-9338-9176A40A57ED”
ssidfreshtime:deviceid由客户端心跳触发刷新，目的是为了转发消息的时候，消息结点可以判断客户端会话的有效性（CurrentTime() - Value(“ssidfreshtime_deviceid”) > N mins)
编程对应redis缓存的数据结构定义
typedef struct{
int32	loginTime;
int32 	clientType;
int32 	status;
int64    loginSeq;
int32	ssidFreshTime;
int64	sessionId;
string	sessionKey;
string 	accNodeIdentify;
int32	newmsgtip;
} UserDeviceInfo;

# 登录流程处理


## 登录流程

![图](assets/Login设计_image2.png)
<a>

## 登录逻辑描述

1~2、client建立到Access的tcp连接。
3~5、client把登录请求发上来，Access会把请求消息转成内部协议包，（如果加解密放在Access，要先对称解密再转成内部pb协议，送出Access前要对称加密）顺着Access-》Router-》Login的网路，把请求送到Login进行登录逻辑处理
6、Login登录逻辑解密Rsa，Aes数据，取到登录鉴权需要的数据，加解密的流程见下图<c>
7~8、Login用解密出来的token, userid, deviceid到缓存获取对应设备token
9~10、Login比较token合法性， 成功则更新设备对应的缓存结构（与微服务共享结构1.2）。 无论验证登录成功与否，都要返回给客户端。
11~13、登录鉴权成功，则生成对称秘钥sessionkey, 用ecdh算法加密放到loginRsp返回给client
登录返回经过Access，Access需要做一些相应的逻辑处理，比如建立<sessionId, socket>与连接的对应关系, sessionid标识唯一连接，也代表了唯一设备，并且在连接对象中存储sessionkey
Access把login结果放回给客户端

## 登录错误码返回

ERR_PARASE_PROTOBUF  		= 20000,    ///< 解析Protobuf出错;
ERR_KEY_FIELD_VALUE  			= 20031,    	///< redis数据结构指定的key_field所对应的值缺失或值为空;
ERR_GENERATE_ECHDKEY_ERROR = 20224,    ///< 生成最终对称加密key错误;
ERR_RSADECRYPT_ERROR 		 = 20226,    ///< rsa解密错误;
ERR_AESDECRYPT_ERROR 		 = 20227,    ///< aes解密错误;
ERR_CHECK_PRE_LOGIN_TOKEN 	 = 21007,   ///< 校验预登录token失败;
ERR_LOGIN_REPEATED   		     = 21030 ,	//用户重复登录(客户端难免会出现logseq不总是增长)

# 加密通道建立流程


## 加密通道建立

![图](assets/Login设计_image2.png)
<b>

## 加密通道建立过程描述

a、加密通道建立和登录逻辑一同处理
b、 Login产生最终的对称加密密钥SessionKey, 返回给client, 返回到Access时，Access要存储Sessionkey，可能用于后续加解密
注：通道加密可以在逻辑服务里边处理，也可以在Access处理。对称加密的sessionKey要存到相应的连接对象的中，为Access接入层对称加解密准备。从客户端发过来的业务消息统一由Access  Decode对称解密后再走内部服务间PB协议传输。Access最终给客户端回消息在Encode的时候要用sessionkey对称加密再传输

# 加解密流程图

![图](assets/Login设计_image1.png)
<c>

# 登录缓存的status状态的维护

a、客户端登录后，在线状态status的维护是后续消息路由转发的关键
b、影响stauts的场景比较多，列举如下
Login(不触发踢人，status不变化)；Logout（客户端退出账号，微服务要通知Im消息后台把客户端的长连接断掉，并跟新缓存状态。当然也可由客户端主动close给后台一个关闭信号）；Kitout(来自于管理列表的踢设备下线，微服务要给长连接后台通知，从而更新缓存）；客户端app推到后台/或关闭程序而发出close（后台收到close信号也要更新缓存staus);客户端异常，Access应该还要具备自动检测无效链路，从而清理，顺道跟新对应链路会话的staus; Access异常宕机，后台没有close信号，缓存的状态更新只能配合缓存里的“ssidfreshtime”，在某次转发逻辑触发的时候，发现这个时间超过了可接受的时间，那么会话sessionid也就认为失效了

| 版本编号 | 编写/修订内容 | 修订人 | 修订日期 |
| --- | --- | --- | --- |
| 2020-01 | V1.0 | 编写 | Sammy/Frank |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |
