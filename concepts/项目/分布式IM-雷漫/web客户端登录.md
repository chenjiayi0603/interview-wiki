
# Web客户端登录流程图


![图](assets/web客户端登录_image1.emf)

Login节点会校验web登录来自Access（web）节点（检查来源的acc的节点名称）

# 协议头


## 长连接协议框架：

包头为浅黄色部分

| 字段名 | 描述 | 长度字节 | 备注 |
| --- | --- | --- | --- |
| len | 包总长度 | 4 | 固定包头+可变包体 |
| cmd | 消息类型 | 4 | Web登录消息指令 (cmd =1011) Web登录响应指令 (cmd=1012) |
| seq | 序列号id | 4(uint32) | 请求方填写，返回确认需带回SEQ (web客户端登录包请求填1，后续包填登录响应的startSeq开始递增的序号，如startSeq为10000，则从10001开始并且递增) |
| version | 协议版本号 | 1 | 协议版本号：当前固定为1 |
| reserve | 保留字段 | 1 | 黄色 01 aes 10 rsa  绿色 01 gzip压缩 （Web客户端不需要使用aes和rsa加密，目前也未使用gzip压缩 ，则填0。） |
| status | 状态 | 2 | 请求默认200，返回具体定义(web客户端登录包请求填0) |
| 协议包体 |  |  | Pb方式，具体根据指令文档确定。（MsgBody的大小长度） |


# 协议包体

  `message MsgBody`
{
  `optional bytes body         = 1;		///< 消息体主体 （客户端填写）（单独加密字段）`
  `optional bytes targetId     = 2;		///< 消息路由ID  （客户端填写）（单独加密字段）（单聊消息为接收者uid，个人信息修改为uid，群聊消息为groupid，群管理为groupid）`
}
  `消息体主体为具体逻辑请求内容。`
  `消息路由id，为后台路由选择使用，填写规则如：单聊消息为接收者uid，个人信息修改为uid，群聊消息为groupid，群管理为groupid`

## 消息体主体

Web登录消息指令1011，除了不需要加解密外，具体字段含义跟其他app的一致。
除了登录协议外，其他协议跟其他app一致，具体参考文档 客户端---消息服务通讯协议1.doc
消息体格式：
  `//Web登录消息指令 （cmd =1011）`
  `message WebLoginReq`
{
  `int64 userId = 1; // 用户id,8个字节`
  `string token = 2; // 用户token`
  `int32 clientTime = 3; // 时间戳，秒，4个字节`
  `string devId = 4; // 设备id`
  `int64 loginSeq = 5; // 登录序列号，每次自增，8个字节（web端设置为0）`
  `string other=6;    // 其余数据待定`
}
  `//Web登录响应指令 (cmd=1012)`
  `message WebLoginRsp{`
  `int32 code=1;//应答业务码`
  `string codeMsg=2;//错误描述`
  `string sessionID=3;//会话id`
  `int32  startSeq=4;//客户端后续消息的起始seq（后续消息头使用的msgseq的起始数）`
  `int32 serverTime = 5; // 时间戳，秒，4个字节`
  `string other=6;    // 其他业务数据`
  `int64 loginSeq=7; //redis记录的上一次loginseq,只用在登录返回错误码21030`
}
/*
  `*其中WebLoginReq中的other 定义为json：`
*{
  `*	"language":"zh-CN",  //客户端语言版本，如en-US、zh-CN、default，匹配不上的，使用default的`
  `"clientVersion":"1.0.3", 	//客户端版本`
  `"platformType":1,//android:1 ,ios:2,mac:3,win:4,web:5,manager:6,h5:7`
  `"userAgent":""//web浏览器等信息（有则填写，无则不填写）`
*}
*/

# 登录响应错误码

  `错误码：响应头部的status字段和响应消息体WebLoginRsp的code字段。`
  `需要先判断头部的status字段，再判断响应消息体WebLoginRsp的code字段。`
Status和code的错误码的值是一致的。
除了加解密外，其他错误码跟其他app一致。

## 登录错误码的值定义

ERR_PARASE_PROTOBUF  		= 20000,    ///< 解析Protobuf出错;
ERR_KEY_FIELD_VALUE  			= 20031,    	///< redis数据结构指定的key_field所对应的值缺失或值为空;
ERR_GENERATE_ECHDKEY_ERROR = 20224,    ///< 生成最终对称加密key错误;
ERR_RSADECRYPT_ERROR 		 = 20226,    ///< rsa解密错误;
ERR_AESDECRYPT_ERROR 		 = 20227,    ///< aes解密错误;
ERR_LOGIN_REPEATED             = 21030 ,    //用户重复登录，这只会发生客户已经成功登录，但依然重复发上来登录包。这种情况是不应该发生的,就算出现，客户端知道这个错误码也应该直接close tcp,重新连接
ERR_CHECK_PRE_LOGIN_TOKEN 	 = 21007,   ///< 校验预登录token失败;
ERR_LOGINSEQ_ERROR	         = 21031 ,	//用户loginseq错误(客户端难免会出现loginSeq不总是增长, 遇到这个错误码，客户端应该拿服务器返回的更新本地loginseq)
注：登录错误码21007，客户端要重新请求微服务登录接口；登录错误码21031，客户端需要把带回去的loginseq更新本地，重建TCP 登录
客户端登录遇到以上错误码返回, Tcp都会被服务器断开，并且返回的LoginRsp是不加密的。
     这样客户端可以统一处理登录返回：当头部status = 0 ， 认为是成功，要解密LoginRsp才能反序列化pb; 当头部status != 0, 认为是失败， LoginRsp没有被加密，这时客户端直接可以反序列化pb,并根据LoginRsp里的code值来处理接下来的逻辑。Status和code的值是一致的。Status==0，code也等0

# 地址

IM web消息平台 联调开发环境，接入服务提供域名接入，地址为：wss://dev-longlink-web-im.raymannet.com:443/hello/shake    