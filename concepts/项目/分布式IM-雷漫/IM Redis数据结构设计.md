# IM Redis数据结构设计

IM  Redis 数据结构设计说明书
修订记录
修订记录：
目 录
1. Redis数据结构分类设计	3
1.1. Redis数据类型	3
1.2. IM数据结构分类	3
1.3. IM通用Redis Value数据定义	3
2. IM属性型数据结构定义	4
3. 消息系统使用redis缓存用到的key汇总	4
3.1. 用户登录Token的key (string)	4
3.2. 用户Login 缓存(hash)	4
3.3. 好友缓存信息（hash）	4

# Redis数据结构分类设计

IM的Redis数据结构的key由Redis数据类型、IM数据结构分类、数据关键字标识三部分组成，格式“E_REDIS_TYPE:E_REDIS_IM_DATA:logic_key”，例如：“2:1:12345678”。

## Redis数据类型

Redis数据类型定义是为了达到通过key识别所存储数据结构的目的而设置。定义如下：

## IM数据结构分类

IM数据结构分类的设计实现通过key识别所存储的数据结构属于何种类型，同时也解决了需要使用同一logic_key来作为redis key数据结构的冲突问题。
表2.1  IM数据结构分类说明

## IM通用Redis Value数据定义

IM通用Redis Value数据是为了让redis的数据结构可以表达更复杂的数据应用场景而设计。message Recod是一个通用的value结构。

# IM属性型数据结构定义

IM属性型数据结构包括用户属性、群属性等属性型状态数据，每个属性实体只会有一条数据，属性数据只会变更，不会增加或删除。属性型数据一般不设置过期时间，如果设置也是较长的时间，比如几个月。

# 消息系统使用redis缓存用到的key汇总


## 用户登录Token的key (hash)

每个设备每个用户id一个token, key如下
1:1:im:token:deviceid:00000000
Hash:
{
${userid} 	值为token
}

## 用户Login 缓存(hash)

同一个用户的多个设备信息存在一个hashkey
1:2:im:status:userid:121415276982521036
消息平台与微服务共享的缓存cash，微服务平台创建key并设置key的有效期。消息平台在客户端登录时刷新其中的域值并加入必要Field,但不会删除其中域值。
Key:  1:2:im:status:userid:100122
Hash:
{
“logintime:${deviceid}”  		1578987836 //秒，每次tcp重建都需要刷新
“clienttype:${deviceid}”  		0/1/2/3 //用于推送平台的识别,配合newmsgtip判断离线是否推送到某个平台（android, ios, Huawei…）
“status:${deviceid}”  		0/1/ offline/online，主要是在线离线状态维护，消息路由转发也要判断，影响这个状态的场景相对较多，后续列举
“loginseq:${deviceid}” 		100000,  //防重登攻击，在分布式场景下，这个seq可以阻止同一个设备 “seq <= loginSeq”的登录请求
“ssidfreshtime:${deviceid}”  	15344838// 秒，依据心跳频率刷新，微服务其实查询最近登录时间可以取这个时间比logintime更合理些
“sessionid:${deviceid}”  		123222222//由Login生成要带回给Access，代表客户端在Access的链路唯一标识，防止分布式的消息包路由错误
“sessionkey:${deviceid}”  	“abbcdefghijk”, //这个是对称加密key, 32字节的字符串（缓存共享可以使得加解密分散给各逻辑服务，避免Access集中加解密耗时，目前也要带回给Access，由Access统一加解密）
“accnodeidentify:${deviceid}”   “ip:port:workIndex” //用户登录的网关定位标识
“clientip:${deviceid}”“192.168.1.1”
“platformtype:${deviceid}” 2// 1:Android  2:Ios  3:Mac  4:windows…
“expire:${deviceid}”// sessionkey过期时间，一段时间内的登录使用的是同一个sessionkey
“language:${deviceid}” //客户端语言版本，如en-US、zh-CN、default，匹配不上的，使用default的
}
编程对应redis缓存的数据结构定义
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

## 好友缓存信息（hash）

用户好友统一放到一个hash下，可以用hmset获取其中一位好友的所有字段信息
Key:
1:3:im:friendship:userid:A
好友关系缓存结构hash【1】:
{
frd_id:friendId  		//好友B的ID
status:friedId   		//A->B是否好友
black:friendId   		//A->B是否拉黑
frd_status:friendId    //B->A是否好友
frd_black:friendId     //B->A是否拉黑
}
注：当A,B两人的好友关系变化时，比如A操作了B
先判断A的好友B的信息（B的所有field字段信息）是否存在，存在就增量更新（更新某个field），不存在微服务可以考虑从DB拉数据全量更新或者直接不管

## 群成员redis结构

缓存结构
key:    1:4:im:groupmember:groupid:${groupid},
例如 1:4:im:groupmember:groupid:123434445
缓存结构hash【1】:
{
id:${memberid}    // int64 群成员的ID
}
注：当群成员增量添加的时候，判断key是否存在；存在就增量更新，key不存在时微服务可以考虑从DB拉数据全量更新或者直接不管

## 用户所在群redis结构

缓存结构
key:    2:5:im:belonggroup:userid:${userid},
例如 2:5:im:belonggroup:userid:10000000000000
缓存结构set【2】:
{
${groupid}   // int64 群ID
${groupid}
${groupid}
}
注：用户所在群增量更新时，判断key是否存在；存在就增量更新，key不存在时微服务可以考虑从DB拉数据全量更新或者直接不管

## 群消息推送状态

缓存结构
key:    6:6:im:groupmsgpushstatus:groupid:${groupid}:userid:${userid}:deviceid:${deviceid}
例如
6:6:im:groupmsgpushstatus:groupid:10000000000000:userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
缓存结构zset【6】:
微秒（16位数）作为score，例如1586259378205824
score（16位数,score越小的推送优先级越高）为
1586259378205824
{
Score：
${score} //微秒级时间戳，例如1586259378205824
Val：
${msgid}
//例如msgid消息id ，6635468343765696713
}
遍历推送消息
ZRANGEBYSCORE  6:6:im:groupmsgpushstatus:groupid:10000000000000:userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
0 +inf WITHSCORES
分页遍历
ZRANGEBYSCORE  6:6:im:groupmsgpushstatus:groupid:10000000000000:userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
0  +inf WITHSCORES LIMIT 0  10
推送acc失败（包括推送acc超时响应），则直接推送离线消息；
推送acc，acc响应失败，则直接推送离线消息；
推送acc，acc响应成功，则写入推送消息状态结构；
App响应成功，则移除推送消息状态结构；
用户离线，检查推送消息状态结构，有消息则离线推送。
过期时间：
推送消息状态结构有效期为1天。

## 需要消息推送状态的用户设备--暂时未用

当需要定时全量搜索时，查询本数据结构。一般使用SSCAN key cursor [MATCH pattern] [COUNT count]来分批遍历。
缓存结构
key:    2:7:im:msgpushstatususerdevice
2:7:im:msgpushstatususerdevice
缓存结构set【2】:
{
userid:${userid}:deviceid:${deviceid}
//例如userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
}

## 群订阅类型结构

缓存结构
key:    1:8:im:subscribe:groupid:${groupid}
例如 1:8:im:subscribe:groupid:1000000
缓存结构hash【1】:
{
userid:${userid}:subcmd:${subcmd}    //json结构，订阅类型补充说明（无则设置空）
}

## 会话状态数据redis结构

缓存结构
key:    1:9:im:sessionstatus: sessionid:${sessionid}
例如 1:9:im:sessionstatus: sessionid:${sessionid}
缓存结构hash【1】:
{
common_read_msgid //群会话公共已读位置消息ID
max_msgseq 	//会话预分配的消息序列号段最大值
last_nodeid 		//预分配当前消息序列号段的结点id（用于每次判断结点是否变化）
node_starttime   //last_nodeid启用的时间，结点不重启不变化
}
群聊会话的sessionid为群id，单聊会话id为两个用户的id组合，common_waaread_msgid只对群会话有意义，单聊没用到

## 用户活跃会话数据redis结构

缓存结构
key:    1:10:im:sessionactive:userid:${userid}
例如 1:10:im:sessionactive:userid:100000
缓存结构hash【1】:
{
${targetid}:read_msgid   // int64 会话个人已读点
${targetid}:friend_read_msgid  read_msgid //int64 对方已读自己的位置
${targetid}:min_msgid   //int64 成员（包括单聊和群聊的成员，单聊的只有两个成员）的自己的起始消息ID（包括消息本身和消息状态）
${targetid}:max_msgid   //int64 会话最大消息ID（获取消息结束点）
}
群聊的targetid为群id，单聊targetid  为目标用户id
注：已读点目前考虑是允许过期的，没有已读点数据，计算未读数可能有短暂的不准确； 但没有已读点时，当作0来计算，超出99的显示99+

## 单聊消息推送状态

缓存结构
key:    6:11:im:personalmsgpushstatus:userid:${userid}:deviceid:${deviceid}
例如
6:11:im:personalmsgpushstatus:userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
缓存结构zset【6】:
微秒（16位数）作为score，例如1586259378205824
score（16位数,score越小的推送优先级越高）为
1586259378205824
{
Score：
${score} //微秒级时间戳，例如1586259378205824
Val：
${msgid}
//例如msgid消息id ，6635468343765696713
}
遍历推送消息
ZRANGEBYSCORE  6:11:im:personalmsgpushstatus:userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
0 +inf WITHSCORES
分页遍历
ZRANGEBYSCORE  6:11:im:personalmsgpushstatus:userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
0  +inf WITHSCORES LIMIT 0  10
推送acc失败（包括推送acc超时响应），则直接推送离线消息；
推送acc，acc响应失败，则直接推送离线消息；
推送acc，acc响应成功，则写入推送消息状态结构；
App响应成功，则移除推送消息状态结构；
用户离线，检查推送消息状态结构，有消息则离线推送。
推送消息状态结构有效期为1天。

## 单聊会话已读消息时间redis结构

缓存结构
key:  6:12:im:sessionreadmsg:userid:${userid}
例如 6:12:im:sessionreadmsg:userid:100000
缓存结构sortset【6】:
{
Score：
${score} //微秒级时间戳，例如1586259378205824,保存的已读消息id的时间戳
Val：
${targetid}
//例如 目标用户id，或者群id
}
注：已读点目前考虑是允许过期的，没有已读点数据，计算未读数可能有短暂的不准确； 但没有已读点时，当作0来计算，超出99的显示99+

## 用户订阅好友类型结构

缓存结构
key:    1:13:im:subscribe:userid:${userid}
例如 1:13:im:subscribe:userid:1000000
缓存结构hash【1】:
{
subscriberid:${subscriberid}:subcmd:${subcmd}    //json结构，订阅类型补充说明（无则设置空）
}
${userid} 被订阅的用户id
${subscriberid} 订阅的用户id
${subcmd} 订阅的子指令
值为Json 字符串，具体命令请求信息补充内容，具体根据subCmd指定，如：
{
"overtime":100   //有效时间，单位秒（超时续约需要重新订阅）
"cutofftime":11111111 //截止时间，单位秒
}

## 个人通知消息推送状态

缓存结构
key:    6:14:im:personalnotifypushstatus:userid:${userid}:deviceid:${deviceid}
例如
6:14:im:personalnotifypushstatus:userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
缓存结构zset【6】:
微秒（16位数）作为score，例如1586259378205824
score（16位数,score越小的推送优先级越高）为
1586259378205824
{
Score：
${score} //微秒级时间戳，例如1586259378205824
Val：
${msgid}
//例如msgid消息id ，6635468343765696713
}
遍历推送消息
ZRANGEBYSCORE  6:11:im:personalnotifypushstatus:userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
0 +inf WITHSCORES
分页遍历
ZRANGEBYSCORE  6:11:im:personalnotifypushstatus:userid:10000000000000:deviceid:00000000-797a-f292-ffff-fffffae68467
0  +inf WITHSCORES LIMIT 0  10
推送acc失败（包括推送acc超时响应），则直接推送离线消息；
推送acc，acc响应失败，则直接推送离线消息；
推送acc，acc响应成功，则写入推送消息状态结构；
App响应成功，则移除推送消息状态结构；
用户离线，检查推送消息状态结构，有消息则离线推送。
过期时间：
推送消息状态结构有效期为1天。

## 会话用户消息序号

每个会话单独一个消息序号(string)
4:15:im:msgseq:sessionid:1214152769825210369

## 单聊会话对方已读消息时间redis结构

缓存结构
key:  6:16:im:sessionfriendreadmsg:userid:${userid}
例如 6:16:im:sessionfriendreadmsg:userid:100000
缓存结构sortset【6】:
{
Score：
${score} //微秒级时间戳，例如1586259378205824,保存的已读消息id的时间戳
Val：
${targetid}
//例如 目标用户id，或者群id
}

## 多语言文案缓存

缓存结构
key:    1:17:im:international:language:official:noticetype:${type}
例如 1:17:im:international:language:official:noticetype:1
缓存结构hash【1】:
{
“en-US”: “”
“zh-CN”: “”
“default”: “”
“zh-TW”:””
}

| 日期 | 修订版本 | 修改描述 | 作者 |
| --- | --- | --- | --- |
| 2019-10 | V1.0 | 编写 | Tommy |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |


| 版本编号 | 编写/修订内容 | 修订人 | 修订日期 |
| --- | --- | --- | --- |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |


| 枚举名 | 枚举值 | 数据定义 |
| --- | --- | --- |
| IM_DATA_USER_TOKEN | 1 | 用户Token信息 |
| IM_DATA_USER_LOGIN | 2 | 用户登录缓存 |
| IM_DATA_FRIEND_RELATION | 3 | 好友关系缓存 |
|  | 4 | 群成员redis结构 |
|  | 5 | 用户所在群redis结构 |
|  |  |  |
