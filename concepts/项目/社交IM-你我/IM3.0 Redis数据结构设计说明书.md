
| 深圳市你我金融信息服务股份有限公司 | 深圳市你我金融信息服务股份有限公司 | 文档编号 | NIIWOO-07-04 | NIIWOO-07-04 | NIIWOO-07-04 | NIIWOO-07-04 | NIIWOO-07-04 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 深圳市你我金融信息服务股份有限公司 | 深圳市你我金融信息服务股份有限公司 | 名    称 | IM3.0协议设计说明书 | IM3.0协议设计说明书 | IM3.0协议设计说明书 | IM3.0协议设计说明书 | IM3.0协议设计说明书 |
| 编写 | 签名：廖宝华  日期：2015-11-11 | 签名：廖宝华  日期：2015-11-11 | 密级 | 内部公开 | 版本 | V1.0 |
| 审核 | 签名：      日期： | 签名：      日期： | 批准 | 签名：   日期： | 签名：   日期： | 签名：   日期： |

修订记录：

| 版本编号 | 编写/修订内容 | 修订人 | 修订日期 |
| --- | --- | --- | --- |
| V1.0 | 文档发布 | 廖宝华 | 2015-11-11 |

深圳市你我金融信息服务有限公司
版权所有  不得复制
**目 录**

## 文档目的

本文档对Redis数据结构进行详细描述，为系统研发人员提供编码依据，为后续的升级和维护提供参考。

## 术语和缩写解释


| 词汇名称 | 词汇含义 | 备注 |
| --- | --- | --- |


## 参考资料


| 《IM3.0数据库设计说明书》 | http://192.168.18.14/svn/niiwoo/im/trunk/IM3.0/doc |
| --- | --- |
| 《IM3.0系统架构设计说明书》 | http://192.168.18.14/svn/niiwoo/im/trunk/IM3.0/doc |


# Redis数据结构分类设计

IM的Redis数据结构的key由Redis数据类型、IM数据结构分类、数据关键字标识三部分组成，格式“E_REDIS_TYPE:E_REDIS_IM_DATA:logic_key”，例如：“2:1:12345678”。

## Redis数据类型

Redis数据类型定义是为了达到通过key识别所存储数据结构的目的而设置。定义如下：

## IM数据结构分类

IM数据结构分类的设计实现通过key识别所存储的数据结构属于何种类型，同时也解决了需要使用同一logic_key来作为redis key数据结构的冲突问题。

| 枚举名 | 枚举值 | 数据定义 |
| --- | --- | --- |
| IM_DATA_USER_INFO | 1 | 用户基础属性数据，对应数据库中db_userinfo.tb_userinfo_xx |
| IM_DATA_USER_RELATION | 2 | 用户好友关系 |
| IM_DATA_GROUP_INFO | 3 | 群信息 |
| IM_DATA_GROUP_MEMBER | 4 | 群成员信息 |
| IM_DATA_USER_ONLINE | 5 | 用户在线状态 |
| IM_DATA_CREATE_ACCOUNT | 6 | 创建IM账号 |
| IM_DATA_USER_MAPPING | 7 | 业务guid与imid映射 |
| IM_DATA_USER_OWN_GROUP | 8 | IM用户拥有的群 |
| IM_DATA_GROUP_APPLY | 9 | 用户群申请表 |
| IM_DATA_PERSONAL_NOTIFY | 10 | 用户个人通知 |
| IM_DATA_SERVICE_INFO | 11 | 马甲用户/钱小二补充信息 |
| IM_DATA_PROJECT_ID_INFO | 12 | 项目ID与群ID关系对应表 |
| IM_DATA_PERSONAL_UNIQUE_MSG | 13 | 用户个人唯一消息通知 |
| IM_DATA_ADDRBOOK_FRIEND_INFO | 14 | 天秤通讯录好友控制表 |
| IM_DATA_USER_RELATION_ATTETION_LIST | 15 | 用户好友关系"关注"列表 |
| IM_DATA_USER_RELATION_FANS_LIST | 16 | 用户好友关系"粉丝"列表 |
| IM_DATA_PHONENO_IMID_INFO | 17 | 用户手机号与imid映射信息表 |
| IM_DATA_NICKNAME_IMID_INFO | 18 | 用户昵称与imid映射信息表 |
| IM_DATA_OFFICIAL_INFO | 19 | 公众号附加信息表 |
| IM_DATA_OFFICIAL_USER_SUBSCRIBE | 20 | 用户订阅的公众号列表 |
| IM_DATA_OFFICIAL_USERSET_MEMBER | 21 | 公众号分组用户列表 |
| IM_DATA_OFFICIAL_SYSTEM_ACCOUNT | 22 | 系统公众号用户列表 |
| IM_DATA_SENSITIVE_WORDS | 23 | 敏感词信息 |
| IM_DATA_APP_PATCH | 24 | APP补丁包列表 |
| IM_LOG_SINGLE_MSG | 1001 | 用户单聊消息 |
| IM_LOG_MSG_FRIEND_SET | 1002 | 用户单聊好友列表 |
| IM_LOG_GROUP_MSG | 1003 | 群聊消息 |
| IM_LOG_GROUP_MSG_LIST | 1004 | 群聊消息列表 |
| IM_LOG_GROUP_MSG_RECV_STA | 1005 | 用户群聊消息接收状态 |
| IM_LOG_GROUP_MSG_LAST_ID | 1006 | 群最新消息ID |
| IM_LOG_PERSONAL_NOTIFY | 1007 | 个人通知详情 |
| IM_LOG_PERSONAL_NOTIFY_SET | 1008 | 个人通知列表 |
| IM_LOG_GROUP_NOTIFY | 1009 | 群通知详情 |
| IM_LOG_GROUP_NOTIFY_LIST | 1010 | 群通知列表 |
| IM_LOG_GROUP_NOTIFY_RECV | 1011 | 用户群通知接收状态 |
| IM_LOG_IM_TOKEN | 1012 | 预登录ImToken映射 |
| IM_LOG_WRITE_BLOG | 1013 | 你我圈说说 |
| IM_LOG_WRITE_BLOG_LIST | 1014 | 发表的你我圈说说列表 |
| IM_LOG_LISTEN_BLOG_SET | 1015 | 收听的你我圈说说列表 |
| IM_LOG_BLOG_PRAISE_SET | 1016 | 说说点赞用户列表 |
| IM_LOG_BLOG_COMMENT_NUM | 1017 | 说说评论 |
| IM_LOG_OFFICIAL_MSG | 1018 | 公众号消息结构 |
| IM_LOG_OFFICIAL_USERSET_MSG | 1019 | 公众号分组消息列表 |
| IM_LOG_OFFICIAL_USER_LAST_RECV | 1020 | 用户公众号分组消息接收状态 |
| IM_LOG_OFFICIAL_SPECIFIED_USER_MSG | 1021 | 公众号发指定用户的消息收听列表 |
| IM_LOG_SENDER_SINGLE_MSG | 1022 | 发送者聊消息结构(有失效时间) |

表2.1  IM数据结构分类说明

## IM通用Redis Value数据定义

IM通用Redis Value数据是为了让redis的数据结构可以表达更复杂的数据应用场景而设计。message Recod是一个通用的value结构。

# IM属性型数据结构定义

IM属性型数据结构包括用户属性、群属性等属性型状态数据，每个属性实体只会有一条数据，对每个实体而言属性数据只会变更，不会增加或删除。属性型数据一般不设置过期时间，如果设置也是设置较长的过期时间，比如几个月。

## IM用户属性

IM用户属性数据结构存储IM用户基础信息。常用数据项和数据库列名保持一致
数据类型： hash
Key:    **1:1:imuid**  其中第一位1表示hash结构，第二位1表示IM用户属性
Field:   
      **imid                        IM用户ID**
**  guid                        业务用户ID**
**  ****nickname****                   用户昵称**
**  avatar_url                  用户头像**
**  introduction                用户简介                                                                          **
**  sex                         性别                                                                              **
**  location                    用户所在地**
**  last_login_time             最后一次上线时间**
**  last_logout_time            最后一次下线时间**
**  login_count                 登录次数**
**  online_time                 总在线时长**
**  beblack_count               被拉黑次数**
**  niiwoo_talk_count           发表你我圈说说数量**
**  bereport_count              被举报次数**
**  single_chat_count           发送单聊次数（消息数）**
**  group_chat_count            发送群聊次数（消息数）**
**  singlechat_recv_count       接收单聊次数（消息数）**
**  groupchat_recv_count        接收群聊次数（消息数）**
**  member_level                会员等级**
**  support_count               点赞数量**
**  besupport_count             被点赞数量**
**  comment_count               评论数量**
**  becomment_count             被评论数量**
**  group_number                加入群数量**
**  project_number              业务群数量(参与过的项目数)                                                        **
**  friends_number              好友数量**
**  fans_number                 用户拥有粉丝数量                                                                  **
**  attention_number            关注了的总人数                                                                    **
**  addressbookfriend_number    通讯录好友总人数                                                                  **
**  user_type                   用户类型   **
**  user_identity               用户在你我金融的身份                                                              **
**  company                     公司                                                                              **
**  occupation                  职业                                                                              **
**  mobile                      手机号码[加密]                                                                    **
**  industry                    行业 **
**  ****home_image****                你我圈背景图路径    **
**      i****s_punished****                 ****是否被惩罚**
**  password                   密码**
**  login_history                用户登陆记录**

## IM用户好友关系

IM用户好友关系数据结构存储IM用户好友关系信息。
数据类型： hash
Key:    **1:2:imuid**  其中第一位1表示hash结构，第二位2表示IM用户好友关系
  `Field:   ${friend_imid}    Value： message Record`
  `${friend_imid}    Value： message Record`
  `${friend_imid}    Value： message Record`

## IM群信息

IM群信息存储群属性数据。
数据类型： hash
Key:    **1:3:group_id**  其中第一位1表示hash结构，第二位3表示IM群信息。
Field:   **group_id**    uint32_t， group_id，IM群ID
**owner_id**   uint32_t， 群主ID
**guid**   群主业务ID
**group_name**    群名称
**group_theme**   群主题
**group_avatar**   群头像
**group_intro**     群简介
**group_type**     群类型
**is_valid**  是否有效
**group_scale**  群规模
**group_count**  群成员数量
**is_public**   群是否公开
**joinverify_type**  加群验证
**group_loc**  群所在地区
**group_liveness** 群活跃度
**create_time**  创建时间
**update_time**  更新时间
project_id      群业务标识ID

## IM群成员信息

IM群信息存储群成员信息数据。群成员信息结构的hash field value比较特殊，field_name不表示field_value的含义，而是作为一个key表示成员，field_name为成员ID，field_value为该成员在群中的信息。
数据类型： hash
Key:    **1:4:group_id**  其中第一位1表示hash结构，第二位4表示IM群成员信息。
  `Field:   ${imid}    message Record`
  `${imid}    message Record`
  `${imid}    message Record`

## IM用户状态

IM用户状态存储用户在线信息，。
数据类型： hash
Key:    **1:5:imid**  其中第一位1表示hash结构，第二位5表示IM用户状态信息。
Field:   **imid**    用户IM ID 
**        token**  用户登录验证token 
**        app**    用户移动端接入的Access节点，如： 192.168.11.12:9988.2
**        web**    用户WEB端接入的HttpAccess节点，如： 192.168.10.16:8081.3 
**        ****deviceInfo****   **用户登录的设备号

## 预登录创建IMID

    预登录时，如果业务ID查找不到对应的IMID，则调用此结构创建一个新的IMID。
数据类型： string
    Key:    **4****:****6****:****GET_IMID**其中第一位1表示string结构，第二位6表示创建IM账号
value:  INCR 

## 用户账号映射

IM用户的业务账号和IM账号的映射关系，如，查找一个业务账号对应的IM账号。
数据类型：string
**key:****	****4:7:guid****	**	其中第一位4表示string结构，第二位7表示IM用户业务编号与IM编号映射
  `value: imid`

## IM用户拥有的群

IM用户拥有的群存储每个im对应的群列表。
数据类型： hash
Key:    **1:****8****:imid**  其中第一位1表示hash结构，第二位8表示IM用户拥有的群。
  `Field:   ${group_id}    message Record`
  `${group_id}    message Record`
  `${group_id}    message Record`

## 创建群ID

    创建群时生成一个新的群ID。
数据类型： string
    Key:    **4****:****9****:****GET_GROUPID**其中第一位1表示string结构，第二位9表示创建群ID
value:  INCR 

## 马甲用户\钱小二客户补充信息

马甲用户\钱小二补充信息。
数据类型：hash 
**key:****	****1****:****11****:****imid****	****	**其中第一位1表示hash结构，第二位11表示马甲用户或钱小二补充信息，imid为马甲用户或钱小二用户的ID号
value中包括: 
imid
Name
business_id,face,description,provice,city,area,verify_info,customer_phone,del_flag等字段

## IM 项目ID与群ID关系表

IM群信息存储群属性数据。
数据类型： hash
Key:    **1:****12****:****group_biz_id**  其中第一位1表示hash结构，第二位12表示项目ID与群ID对应关系表，第三位表示业务ID。
Field:   **group_biz_id**     string  业务ID
**group****_id**   uint32_t， 群ID
**record_time**   生成时间

## 用户唯一离线消息通知

IM用户唯一离线消息通知。如朋友圈红点推送。
数据类型： string
**Key:****	****1:13:****${****imid****}** 其中第一位1表示hash结构，第二位12表示IM用户唯一离线消息通知，第三位imid表示接收用户id，
Field：${cmd}消息类型
  `Value:	message Record （具体内容与消息类型有关）`

## IM 项目天秤通讯录好友控制表

通讯录好友关系数据。
数据类型： hash
    Key:    **1:****14****:imuid**  其中第一位1表示hash结构，第二位2表示IM用户好友关系
  `Field:   ${friend_imid}    Value： message Record`
  `${friend_imid}    Value： message Record`
  `${friend_imid}    Value： message Record`

## IM 用户好友关系用户“关注”有序列表

    数据类型： sorted set
    Key:    **6****:****15****:imuid**  其中第一位1表示hash结构，第二位2表示IM用户好友关系关注列表，第三位表示用户 id
score:   **time （时间的 uint64 数值）**
member: im_fuid 

## IM 用户好友关系用户“粉丝”有序列表

    数据类型： sorted set
    Key:    **6****:****16****:imuid**  其中第一位1表示hash结构，第二位2表示IM用户好友关系粉丝列表，第三位表示用户 id
score:   **time （时间的 uint64 数值）**

## 用户手机号与imid信息映射

IM用户的基础信息（电话号码，手机号码是加密的）和IM账号的映射关系
数据类型：string
**Key:****	****4:17:phoneno****	**其中第一位4表示string结构，第二位17表示用户手机号与imid映射，phonenum是编码后的string
Value:	imid

## 用户昵称与imid信息映射

IM用户的基础信息（昵称）和IM账号的映射关系
数据类型：string
**Key:****	****4:18:nickname****	**其中第一位4表示string结构，第二位18表示用户昵称与imid映射，nickname为string
Value:	imid

## 公众号附加信息表

公众号附加信息是存储公众号的相关信息。
   数据类型： hash
Key:    **1:****19****:imid**  其中第一位1表示hash结构，第二位0221表示公众号账户属性
Field:      
**  **offical_imid**                  业务用户ID**
**  d****escription****                 ****公众号****简介**
**  accout_type                ****公众号类型**
 ** defaule_userset             默认分组**
**  userset_binding_list        发布对象权限分组id**
**  pub_cycle_frequency       发布的频率控制周期**
**  pub_limit_num             发布的频率控制次数**
**  current_pub_count         当前周期已经发布的次数**
**  last_pub_time              最后一次的发布时间**
**  user_max_receive          用户每天最可多接收条数**
**  record_time                生成时间**

## 用户订阅的公众号列表

用户订阅的公众号列表 
数据类型： hash
Key:    **1:****20****:imid**  其中第一位1表示hash结构，第二位20表示用户订阅的公众号列表,第三位表示订阅者的imid
  `Field:   ${official_imid}    Value： message Record`
  `${official_imid}    Value： message Record`
  `${official_imid}    Value： message Record`
  `message Record:`
**current_count          当前周期内已经接收消息条数     **
**first_receive_time      当前周期内第一条消息接收时间    **
**last_receive_time      当前周期内最后一条消息接收时间   **
**record_time            该条记录的生成时间; **

## 公众号分组用户列表

公众号分组用户列表用于存储加入了公众号分组的用户。
数据类型： sorted set
Key:    **6****:****21****:${****userset_id****}****:****${****imid}%100**其中第一位6表示sort set结构，第二位21表示公众号分组消息列表，${userset_id}分组ID。
score    ${record_time}
member  ${imid}
**Value:   **
1447299877 **17454 **1447219837 **23145 **1447299877 **10254 **  

## 系统公众号列表

此结构用于单独存系统公众，每个用户取分组离线消息时除了要查自己订阅的普通公众号，同时还要去查系统公众号(系统公众号默认不需要订阅)。
 数据类型： set
Key:    **2****:****22****:****0**其中第一位2表示 set结构，第二位22表示系统公众号列表，第三位固定0。这个结构只有一个key
Value是official_imid
例：
Value:   **$**{official_imid}  **$**{official_imid} ** ****$**{official_imid} **$**{official_imid}

## 敏感词列表

敏感词数据结构存储敏感词信息。
  数据类型： set
  Key:    2:23:${sensitive_type}:${secondary_type}其中第一位2表示set结构，第二位23表示敏感词库,第三位${sensitive_type}表示敏感词类型,第四位${secondary_type}表示二级类型（默认0）；
   Value是word
   例：
   Value:   ${word}  ${word}  ${word} ${word}

## APP补丁列表

客户端热修复补丁更新表
数据类型： hash
Key:    **1:2****4:APP_PATCH****(只有一个key)**
其中第一位1表示hash结构，第二位24表示APP补丁包列表
  `Field:   ${patch_id}    Value： message Record`
  `${patch_id}    Value： message Record`
  `${patch_id}    Value： message Record`

# IM日志型数据结构定义

IM日志型数据结构包括聊天消息，用户信息变更，你我圈说说等。日志型数据只会增加或删除，一旦生成就不会再修改。日志型数据一般设置较短的过期时间，比如几天。

## 接收者单聊消息结构(不失效)

数据类型： hash
Key:    **1:****1001:${self_****imid****}:${the_other_imid}**  其中第一位1表示hash结构，第二位1001表示用户单聊消息，${self_imid}为请求消息记录的用户ID，${the_other_imid}为聊天消息记录另一方的uid。
  `Field:   ${msg_serial_t}  message Record`
  `${msg_serial_t}  message Record`
  `${msg_serial_t}   message Record`

## 单聊消息好友列表

单聊消息好友列表，将${uid1}与${imid}的离线消息推送完毕之后需要从set中删除${uid1}。
数据类型： set
Key:    **2:1****002:${****imid****}**  其中第一位2表示set结构，第二位1002表示用户单聊消息好友列表，${imid}为具体用户ID。
Values:   ${uid1}  ${uid2}  ${uid3} ……

## 群聊消息结构

数据类型： hash
Key:    **1****:1003:${group_id}:****YYYYMMDD**  其中第一位1表示hash结构，第二位1003表示群聊消息详情，${group_id}为群ID，**YYYYMMDD**为key创建的时间（以天为单位，每周日为一周开始，一周生成一个key）。
Field:	${msg_serial_t}
  `Value:   message Record[msg_serial_t, attribute_body]`
写时通过PB来将消息写入REIDS。查询时，将msg_serial_t传给存储代理，存储代理将MYSQL返回的数据集以DATA_SET方式写入REIDS。

## 群聊消息列表

群聊消息列表用于消息失败重发或离线消息推送。
数据类型： list
Key:    **5****:1004:${group_id}:YYYYMMDD**  其中第一位5表示list结构，第二位1004表示IM用户单聊消息列表，${group_id}为群ID，YYYYMMDD为该key的创建日期（以天为单位，每周日为一周开始，一周生成一个key）。
Value:   **1447299877216741  1447299877216742  1447299877216743  **消息ID。
数据类型： sorted set
Key:    **6****:1004:${group_id}:YYYYMMDD**   其中第一位6表示sorted set结构，第二位1010表示群聊消息列表，${group_id}为群ID。YYYYMMDD为该key的创建日期（以天为单位，每周日为一周开始，一周生成一个key）。
Member:  {msgid}
Value:   **1447299877216741  1447299877216742  1447299877216743  **消息ID。

## 群聊消息接收状态

用户群聊消息接收状态存储的是用户对每个群接收的最近一条消息。
数据类型： hash
Key:    **1****:1005:${****imid****}**  其中第一位1表示hash结构，第二位1005表示用户群聊消息接收状态，${imid}为具体用户ID。
Value:   **${group_id}  **msg_serial_t**  **
**         ****${group_id}  **msg_serial_t
  `${group_id}   msg_serial_t`

## 群最近消息ID

群最新消息ID存储的是每个群接收的最近一条消息的ID。该消息ID将在REDIS中永久保存。
数据类型： string
Key:    **4****:100****6****:${****group_id****}**  其中第一位4表示string结构，第二位1006表示群最近消息ID，${group_id}为具体群ID。
Value:   msg_serial_t**  **

## 个人通知详情

个人通知详情在redis中保留三个月，若从数据库中读取通知推送，无须回写redis，若确认用户已收到某通知，需将该通知从redis中删除。
数据类型： string
Key:    **4****:****100****7****:${****imid****}:${****msg_id****}**  其中第一位4表示string结构，第二位1007表示个人通知详情，${imid}为通知所有者用户ID，${msg_id}为通知ID。
  `Value:   message Record`

## 个人通知列表

个人未接收通知列表，通知确认送达后需从列表中删除。在Redis中永久保存，若某用户无此结构，说明无属于该用户的通知。
数据类型： set
Key:    **2:1****00****8****:${****imid****}**  其中第一位2表示set结构，第二位1008表示用户未接收通知列表，${imid}为具体用户ID。
Values:   ${msg_id1}  ${msg_id2}  ${msg_id3} ……

## 群通知详情

群通知存储的是群通知详情，在Redis中保存一个月。
数据类型： string
Key:    4**:100****9****:${group_id}:${****msg_id****}**  其中第一位4表示string结构，第二位1009表示群通知详情，${group_id}为群ID，${msg_id}为通知消息ID。
  `Value:   message Record`
写时通过PB来将消息写入REIDS。查询时，如果redis未命中，无须回写redis。

## 群通知列表

群通知列表用于通知失败重发或离线通知推送。群通知列表在redis中保存一个月。
数据类型： sorted set
Key:    **6****:10****10****:${group_id}**  其中第一位5表示Sorted set结构，第二位1010表示群通知列表，${group_id}为群ID。
Member:  {msgid}
Value:   **1447299877216741  1447299877216742  1447299877216743  **消息ID。

## 群通知接收状态

用户群通知接收状态存储的是用户对每个群接收的最近一条通知。接收状态在redis中保存三个月，redis中若无此结构，把redis中存在并且属于用户所在群的通知全部推送给该用户。
数据类型： hash
Key:    **1****:10****11****:${****imid****}**  其中第一位1表示hash结构，第二位1011表示用户群通知接收状态，${imid}为具体用户ID。
Value:   **${group_id}  **msg_serial_t**  **
**         ****${group_id}  **msg_serial_t
  `${group_id}   msg_serial_t`

## 用户账号登录态信息

IM用户的ImToken和IM账号的映射关系，当用户预登录时，会将该数据写入redis,并将ImToken返回给客户端，客户端调用相关接口时，会把ImToken发送给IM，IM从而可以去redis中查询对应的imid。
数据类型： string
**Key:****	****4:1012:imToken **其中第一位4表示string结构，第二位5表示IM用户状态信息，imToken是经加密后的字符串表示当前用户登录态
Value:	imid

## 你我圈说说结构

数据类型： hash
Key:    **1:****10****13****:${****msgid****}****:${index}**  其中第一位1表示hash结构，第二位1013表示你我圈说说，${msgid}为说说ID，${index}为说说消息块序号，0为第一块，1为第二块，依次类推，如果只有一块则只有0。每个说说数据块最多存储2000条（说说、评论、回复、点赞）
  `Field:   ${msg_serial_t}  message Record`
  `${msg_serial_t}  message Record`
  `${msg_serial_t}   message Record`

## 发表的你我圈说说列表

数据类型： sorted set
score为说说msgid。
Key:    6**:****10****14****:${****imid****}**  其中第一位6表示sorted set结构，第二位1014表示发表的你我圈说说列表，${imid}为用户ID。
Value:   1447299877216741  1447299877216742  1447299877216743**  **说说ID。

## 收听的你我圈说说列表

数据类型： sorted set
score为说说msgid。一个用户发表你我圈说说时，遍历他的粉丝将说说ID写入收听的你我圈说说列表。该列表只保留最近一年的收听说说，每次写入时多执行一条zremrangebyscore删除一年以前的说说ID。若要获取一年以前的说说或者收听说说列表不存在，则以遍历好友列表（注意不是关注列表，这样是为了减少拉取的数量缩短拉取时间，拉取是熟人社交，推送是微博社交）的方式拉取。
Key:    6**:****10****15****:${****imid****}**  其中第一位6表示sorted set结构，第二位1015表示收听的你我圈说说列表，${imid}为用户ID。
Value:   ${imid}:1447299877216741  ${imid}:1447299877216742  ${imid}:1447299877216743**  **value为发表说说用户imid和说说ID的组合。

## 说说点赞用户列表

数据类型： hash
每个用户对一个说说只能点赞一次，此结构用于检查用户是否已经点赞过。客户端拉取说说列表时，把是否已经点赞过某一条说说随说说一起下发到客户端，客户端用于展示成点赞按钮还是取消点赞按钮。客户端发起点赞或取消点赞请求时也需要查询此结构来判断请求的合法性。
Key:    1**:****10****16****:${****msgid****}**  其中第一位1表示hash结构，第二位1016表示说说点赞用户列表，${msgid}为说说ID。
Field:   ${imid}  ${msgid}
        ${imid}  ${msgid}
        ${imid}** ** ${msgid} 
        Field中的${imid}为点赞用户ID，${msgid}为点赞消息ID。

## 说说评论数量

数据类型： string
每个用户。
Key:    4**:****10****17****:${****msgid****}**  其中第一位1表示hash结构，第二位1017表示说说点赞用户列表，${msgid}为说说ID。
Value:   1258   评论/回复数量，新增评论时incr，删除评论时decr。

## 公众号消息结构

数据类型： string
Key:    **4****:10****18****:${msg****id****}**其中第一位1表示hash结构，第二位1018表示公众号分组消息详情，**${msg****id****}****为消息****ID**
  `Value:   message Record`

## 公众号分组消息列表

公众号分组消息列表用于消息失败重发或离线消息推送。
数据类型： sorted set
Key:    **6****:10****19****:${****userset_id****}**其中第一位6表示sorted set结构，第二位1019表示公众号分组消息列表，${userset_id}分组ID。
score :    ${msgid}	
Member:  {send_imid}:${msgid}
Value:   **1447299877216741  1447299877216742  1447299877216743  **消息ID。

## 用户公众号分组消息接收状态

用户公众号分组消息接收状态存储的是用户对每个分组接收的最近一条消息。
数据类型： hash
Key:    **1****:10****20****:${****imid****}**  其中第一位1表示hash结构，第二位1020表示用户群聊消息接收状态，${imid}为具体用户ID。
  `Field Value:   ${userset_id}   message Record`
  `${userset_id}    message Record`
  `${userset_id}    message Record`

## 公众号发指定用户的消息收听列表

如果发送消息不以分组的形式发送，而是指定了接收者列表，那么离线消息存到此结构中去。
用户接收成功后会删除已经接收过的消息msgid
数据类型： set
Key:    **2****:10****21****:${****im****_id****}**其中第一位2表示 set结构，第二位1021表示公众号指定接收消息的用户列表，${im_id}接收者IMID。
Member : {send_imid}:{msgid}
例：
17574:1447299877216743** ** 17890:1447299877216745  10186:1447299877416746**  **** **

## 发送者单聊消息结构(有失效时间)

数据类型： hash
Key:    **1:****10****22****:${self_****imid****}:${the_other_imid}**  其中第一位1表示hash结构，第二位1001表示用户单聊消息，${self_imid}为请求消息记录的用户ID，${the_other_imid}为聊天消息记录另一方的uid。
  `Field:   ${msg_serial_t}  message Record`
  `${msg_serial_t}  message Record`
  `${msg_serial_t}   message Record`