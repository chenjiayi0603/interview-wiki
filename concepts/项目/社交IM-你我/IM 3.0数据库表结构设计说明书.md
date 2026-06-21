
| 深圳市你我金融信息服务股份有限公司 | 深圳市你我金融信息服务股份有限公司 | 文档编号 | NIIWOO-09-18 | NIIWOO-09-18 | NIIWOO-09-18 | NIIWOO-09-18 | NIIWOO-09-18 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 深圳市你我金融信息服务股份有限公司 | 深圳市你我金融信息服务股份有限公司 | 名    称 | IM3.0系统存储设计说明书 | IM3.0系统存储设计说明书 | IM3.0系统存储设计说明书 | IM3.0系统存储设计说明书 | IM3.0系统存储设计说明书 |
| 编写 | 签名：卜应敏  日期：2015-9-18 | 签名：卜应敏  日期：2015-9-18 | 密级 | 内部公开 | 版本 | V3.0 |
| 审核 | 签名：      日期： | 签名：      日期： | 批准 | 签名：   日期： | 签名：   日期： | 签名：   日期： |

修订记录：

| 版本编号 | 编写/修订内容 | 修订人 | 修订日期 |
| --- | --- | --- | --- |
| V3.0 | 文档发布 | 卜应敏 | 2015-09-18 |
| V3.0.1 | 评审修改 | 卜应敏 | 2015-09-21 |
| V3.0.2 | 新增表字段结构 | 卜应敏 | 2015-11-17 |
| V3.0.3 | 字段规范 | 卜应敏 | 2015-11-24 |
| V3.0.4 | 给表结构新增记录时间 | 卜应敏 | 2015-12-03 |
| V3.0.5 | 1.删除管理中的3张表结构，2.增强文档结构的可读性 | 卜应敏 | 2015-12-07 |
| V3.0.6 | 1.新增分表分库策略；2删除表结构图 | 卜应敏 | 2015-12-08 |
| V3.0.7 | 1.新增普通用户关键信息表，2.调整业务与群映射信息表 | 卜应敏 | 2015-12-25 |
| V3.0.8 | 修改3.3.2的表名（手机号与imid映射表） 新增3.3.3 用户昵称与imid映射表 | 卜应敏 | 2016-01-11 |

深圳市你我金融信息服务有限公司
版权所有  不得复制
**目 录**

## 文档目的

本文档从设计的角度对系统进行综合描述，通过总体视图来描述系统的各数据表的关联关系，通过表结构具体了解业务关系。同时，也为系统研发人员提供编码依据，为后续的升级和维护提供参考。

## 术语和缩写解释

列出本说明书中专门术语的定义、英文缩写词的原词组和意义、达成一致意见的专用词汇，同时要求继承全部的先前过程中定义过的词汇。

| 词汇名称 | 词汇含义 | 备注 |
| --- | --- | --- |

备注中注明该词汇的来源，或有其他更详细的解释的文档位置；以及对该词汇的其他叫法。

## 参考资料

列出编写本报告时参考的文件、资料、技术标准，所有参考的资料必须在配置库中进行存储，除《配置识别表》中要求的存储路径外，其余参考资料（比如除需求规格说明书外的其它国家标准、外部通讯协议等）统一放在“/工程/其它/参考资料”目录下。

| 参考资料 | 存放路径描述 |
| --- | --- |
| IM2.0-数据库-ER图 - 自动生成.docx | http://192.168.18.14/svn/niiwoo/im/doc/技术文档 |
| IM2.0-数据库表整理（19服务器为准）.docx | http://192.168.18.14/svn/niiwoo/im/doc/技术文档 |
| IM 3.0系统存储设计bit位说明.png | http://192.168.18.14/svn/niiwoo/im/doc/技术文档 |


# mysql数据库分表分库设计


## 设计依据

IM 3.0数据库分表分库设计是遵循在以下几点假设的基础上进行的：
第一点：设计的目标是注册用户为5亿；
第二点：活跃用户比例为10%，即5000万活跃用户；
第三点：存储系统要支撑2年以上；
第四点：每个活跃用户每天单聊平均为10条；

## 总体设计图

根据以上要求分析，需要先进行水平切分数据库操作，再将数据分表操作。则将存储系统设计为 图1数据库集群部署图 所示。

![图](assets/IM 3.0数据库表结构设计说明书_image1.png)

图1 数据库集群部署图
如上图所示，整个数据层有GroupA，GroupB，GroupC三个集群组成，这三个集群就是数据水平切分的结果，当然这三个集群也就组成了一个包含完整数据的DB。每一个Group包括1个Master（当然Master也可以是多个）和N个Slave，这些Master和Slave的数据是一致的。比如GroupA中的一个slave发生了宕机现象，那么还有两个slave是可以用的，这样的模型总是不会造成某部分数据不能访问的问题，除非整个Group里的机器全部宕掉，但是考虑到这样的事情发生的概率非常小。
而对于每个节点数据库来说，节点，数据库和数据库表的关系如下：

|  | 数据库名 | 数据库表名 |
| --- | --- | --- |
|  | db_userinfo_001 | 用户信息: tb_userinfo_0---- tb_userinfo_99 |
| node1 | db_chatmsg_001 | 群聊：tb_groupchat_0----tb_groupchat_99 单聊：tb_singlechat_0----tb_singlechat_99 |
| node1 | db_friends_001 | 好友关系：tb_friends_0----tb_friends_99 映射表：tb_time_friendstype_0---- tb_time_friendstype_99 |
| node1 | db_groupmanager_001 | 被邀请者：tb_beinviter_0---- tb_beinviter_99 申请入群：tb_groupapply_0---- tb_groupapply_99 申请入群审核：tb_groupapply_verify_0---- tb_groupapply_verify_99 业务群信息：tb_groupbizinfo_0---- tb_groupbizinfo_99 群信息：tb_groupinfo_0---- tb_groupinfo_99 邀请入群：tb_groupinvite_0---- tb_groupinvite_99 邀请入群审核：tb_groupinvite_verify_0---- tb_groupinvite_verify_99 群用户：tb_groupuser_0---- tb_groupuser_99 |
| node1 | db_moments_001 | 用户消息关联：tb_mention_users_0---- tb_mention_users_99 背景图：tb_moments_bgimg_0---- tb_moments_bgimg_99 背景图历史记录：tb_moments_bgimg_history_0---- tb_moments_bgimg_history_99 消息关系：tb_moments_msg_0---- tb_moments_msg_99 消息评论：tb_moments_comment_0---- tb_moments_comment_99 消息内容：tb_moments_content_0---- tb_moments_content_99 点赞：tb_moments_like_0---- tb_moments_like_99 |
| node1 | db_niiwooim_001 | 普通用户与业务关联表： tb_syncuser_0---- tb_syncuser_99 钱小二与业务关联表： tb_customeruser_0---- tb_customeruser_99 |
| node1 | db_socialmanagement_001 | 证据列表: tb_evidencetlist_0----tb_evidencetlist_99 业务功能表: tb_opertorfunction 举报详细表: tb_reportlist_0---- tb_reportlist_99 用户权限: tb_userrights_0---- tb_userrights_99 |
| node2 | … | … |
| node3 | … | … |


## 分表分库策略

1.当前存储系统按照以下策略进行分库：依照号段进行切分数据库，每500万个imid存放到一个库中，比如1到500万数据存放到node1中，500万到1000万数据存放到node2中，以此类推。按照号段进行切分数据库的好处：一是扩展性强；二是数据可迁移；三是部署方便，简单。
2. 当前存储系统按照以下策略进行分表：数据库中主要字段是imid或group_id，则数据库按照imid或group_id进行取模操作分表。例如imid%100 或者group_id%100 对数据库进行分表操作。

# mysql数据表总体设计


## IM用户信息 db_userinfo

用户信息，数据库名为db_userinfo，该库下是用户信息表，就涉及到同一类数据表，即用户信息表(tb_userinfo_xx)，官方马甲用户补充信息表(tb_vestinfo_xx) 和 用户拥有的群表(tb_user_has_group_info)。
该库中全是用户信息表和官方马甲用户补充信息表，关系表中主要字段为imid，则分表的原则为imid%100。

| 数据库名 | db_userinfo |
| --- | --- |
| 描述 | 用户详细信息 |

用户信息的分库原则如下：

| 数据库名 | 描述 | 分库策略 |
| --- | --- | --- |
| db_userinfo_000 | 用户信息 | imid从1到500万的用户 |
| db_userinfo_001 | 用户信息 | imid从500到1000万的用户 |

所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_userinfo | 用户信息 | imid%100 |
| 2 | tb_vestinfo | 官方马甲用户补充信息表 | imid%100 |
| 3 | tb_user_has_group_info_xx | 用户拥有的群详情 | imid%100 |

注：X表示数字

### 用户信息 tb_userinfo

- 用户信息表分表策略
 所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_userinfo_xx | 用户信息 | imid%100 |

注：xx表示数字
2. 表结构字段详细说明
对用户信息表的字段描述见如下表（表3-1-1）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | 用户UID ，非空字段，默认值为0 | 主键 |
| 2 | guid | VARCHAR（40) | 业务层编号 |  |
| 3 | nickname | VARCHAR(100) | 用户昵称，编码格式为UTF-8;默认值为NULL |  |
| 4 | avatar_url | VARCHAR(1024) | 用户头像，编码格式为UTF-8;默认值为NULL |  |
| 5 | introduction | VARCHAR(1024) | 用户简介 |  |
| 6 | gender | TINYINT(4) UNSIGNED | 性别,默认值0，0：未知，1：男，2：女 |  |
| 7 | password | VARCHAR(128) | 密码 |  |
| 8 | is_punished | TINYINT(4) UNSIGNED | 是否被惩罚，0 否，1 是；默认值为0 |  |
| 9 | home_image | VARCHAR(128) | 你我圈背景图路径 |  |
| 10 | location | VARCHAR(128) | 用户所在地，编码格式为UTF-8;默认值为NULL |  |
| 11 | last_login_time | BIGINT(24) UNSIGNED | 用户最后一次登录时间,默认值为0 |  |
| 12 | last_logout_time | BIGINT(24) UNSIGNED | 用户最后一次登出时间,默认值为0 |  |
| 13 | login_num | INT(11) UNSIGNED | 登录次数,默认值为0，暂时不用 |  |
| 14 | online_time | INT(11) UNSIGNED | 用户在线时长(秒级)，默认值为0 |  |
| 15 | be_black_num | INT(11) UNSIGNED | 被拉黑次数，默认值为0 |  |
| 16 | blog_num | INT(11) UNSIGNED | 你我圈发布说说次数，默认值为0 |  |
| 17 | be_report_num | INT(11) UNSIGNED | 被举报次数，默认值为0 |  |
| 18 | single_chat_send_num | INT(11) UNSIGNED | 单聊次数统计[主动发送的次数]，默认值为0 |  |
| 19 | group_chat_send_num | INT(11) UNSIGNED | 群聊次数统计[主动发送的次数]，默认值为0 |  |
| 20 | single_chat_recv_num | INT(11) UNSIGNED | 单聊接收次数统计[被动接受的次数]，默认值为0 |  |
| 21 | group_chat_recv_num | INT(11) UNSIGNEDt | 群聊接収次数统计[被动接受的次数]，默认值为0 |  |
| 22 | member_level | TINYINT(1) UNSIGNED | 会员等级，默认值为0 |  |
| 23 | thumb_up_num | INT(11) UNSIGNED | 点赞数，默认值为0 |  |
| 24 | be_thumb_up_num | INT(11) UNSIGNED | 被点赞数，默认值为0 |  |
| 25 | comment_num | INT(11) UNSIGNED | 评论次数，默认值为0 |  |
| 26 | be_comment_num | INT(11) UNSIGNED | 被评论次数，默认值为0 |  |
| 27 | group_num | INT(11) UNSIGNED | 用户拥有群个数，默认值为0 |  |
| 28 | project_num | INT(11) UNSIGNED | 业务群数量(参与过的项目数)，默认值为0 |  |
| 29 | friends_num | INT(11) UNSIGNED | 用户拥有好友个数，默认值为0 |  |
| 30 | fans_num | INT(11) UNSIGNED | 用户拥有粉丝数量，默认值为0 |  |
| 31 | attentioned_num | INT(11) UNSIGNED | 关注了的总人数，默认值为0 |  |
| 32 | contacts_num | INT(11) UNSIGNED | 通讯录好友总人数，默认值为0 |  |
| 33 | user_type | TINYINT(4) UNSIGNED | 用户类型， 0: 未知用户类型, 1：普通用户， 12：尽调小助手， 2：钱小二， 3：官方马甲用户， 4:官方用户, 5：微担保公司 6:公众号 7:客户总台 |  |
| 34 | user_identity | BIGINT(24) UNSIGNED | 用户在你我金融的身份 从低位到高位依次为 第一位：是否实名认证 第二位: 是否借款人 第三位: 是否钱小保 第四位：是否钱大保 第五位：是否加V钱大保 第六位：是否微担保公司 |  |
| 35 | company | VARCHAR(64) | 公司 |  |
| 36 | occupation | VARCHAR(64) | 职业 |  |
| 37 | mobile | VARCHAR(64) | 手机号码[加密] |  |
| 38 | industry | VARCHAR(64) | 行业 |  |
| 39 | create_time | datetime | 创建时间 |  |
| 40 | update_time | datetime | 最近更新时间 |  |
| 41 | login_history | TEXT | 用户登陆记录： json结构， 最多记录10台设备。新增设备替换掉最长时间没有登录的设置信息。 | {     "common_device": [         {             "clienttype": 1,             "deviceInfo": "?? = NX507JSDK = 19SystemVersion = 4.4.2",             "app_version": 360,             "clientip": 4269801472,             "logincount": 8,             "last_login_time": "2016-06-01 16:07:20",             "address_book_md5": "6a28e5323c1b9a380e1b8cb638363257"         },         {             "clienttype": 1,             "deviceInfo": "?? = VPhoneSDK = 19SystemVersion = 4.4.2",             "app_version": 350,             "clientip": 4269801472,             "logincount": 4,             "last_login_time": "2016-05-31 14:50:53",             "address_book_md5": "6a28e5323c1b9a380e1b8cb638363257"         },         {             "clienttype": 2,             "deviceInfo": "6a7e993574b84e42a992dad318a6dd92",             "app_version": 3,             "clientip": 196,             "logincount": 3,             "last_login_time": "2016-05-31 15:21:40",             "address_book_md5": "6a28e5323c1b9a380e1b8cb638363257"         }     ],     "uncomfirm_device": {         "clienttype": 3,         "deviceInfo": "abce12-edfdf2ff3-3fa",         "app_version": 3.3,         "clientip": "192.168.18.5",         "login count": 101,         "last_login_time": "2016-03-17 10:44:50"     } } |


### 官方马甲用户补充信息 tb_vestinfo

1.马甲用户补充信息分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_vestinfo | 马甲用户补充信息 | 不分表 |

2. 表结构字段详细说明
对马甲用户\钱小二客户补充信息表的字段描述见如下表（表3-1-2）

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | IM用户ID,非空字段，默认值为0 | 主键 |
| 2 | nickname | VARCHAR(100) | 用户名称 |  |
| 3 | guid | VARCHAR(40) | 马甲ID |  |
| 4 | avatar_url | VARCHAR(128) | 用户头像 |  |
| 5 | description | VARCHAR(512) | 用户简介 |  |
| 6 | province | VARCHAR(32) | 省份(直辖市、自治区、特别行政区) |  |
| 7 | city | VARCHAR(32) | 市（自治州) |  |
| 8 | area | VARCHAR(32) | 区（县） |  |
| 9 | verify_info | VARCHAR(512) | 认证信息 |  |
| 10 | hotline | VARCHAR(64) | 客服电话[加密] |  |
| 11 | del_flag | TINYINT(4) UNSIGNED | 删除标识（0未删除，1已删除) |  |
| 12 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 用户拥有的群 tb_user_has_group_info

1.以用户的角度描述用户在群中的信息。
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_user_has_group_info_xx | 用户拥有的群 | imid%100 |

注：xx表示数字
     2.表结构字段详细说明
对用户拥有群表的字段描述见如下表（表3-4-1）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | 用户Id；非空字段，默认值为0 | 主键 |
| 2 | group_id | INT(11) UNSIGNED | 群ID, 非空字段；默认值为0 | 主键 |
| 3 | group_name | VARCHAR(100) | 群名称，非空字段，编码格式为UTF-8，默认值为empty string |  |
| 4 | group_avatar_url | VARCHAR(500) | 群头像，编码格式为UTF-8，默认值为empty string |  |
| 5 | group_type | TINYINT(4) UNSIGNED | 群类别 1，业务群(业务发起)，2普通群(app发起)，3讨论组; 非空字段，默认值为0 |  |
| 6 | group_member_num | INT(11) UNSIGNED | 群成员数量,默认值为0 |  |
| 7 | in_project_role | TINYINT(4) UNSIGNED | 用户在群里面的角色( = 0; //普通人, 非业务群会使用这个角色                                  = 1; //净调人                              = 2; //借款人                              = 3; //投资人                              = 4; //担保人                                           = 13;//尽调投资     = 43;//担保投资 ) |  |
| 8 | in_group_role | TINYINT(4) UNSIGNED | 群角色。0：未知；1群主;2：管理员 非空字段；3:普通成员 。默认为0 |  |
| 9 | block_type | TINYINT(4) UNSIGNED | 是否屏蔽群消息  1屏蔽， 2不屏蔽；非空字段，默认值为0 |  |
| 10 | record_time | DATETIME | 该条记录的生成时间;默认为'1970-01-01 08:00:00' |  |


## IM关系属性 db_relationship

IM用户关系，数据库名为db_relationship,该库下主要是好友关系表，即好友关系(tb_friends_xx) , 天秤通讯录好友控制表(tb_adressbook_friends_xx) 。
该库中全是存储好友关系的，关系不复杂，则分表的原则为imid%100。在该库中最多有100张好友关系表。

| 数据库名 | db_relationship |
| --- | --- |
| 描述 | IM用户关系 |

IM用户关系的分库原则如下：

| 数据库名 | 描述 | 分库策略 |
| --- | --- | --- |
| db_relationship_000 | IM用户关系信息详情 | imid从1到500万的好友关系 |
| db_relationship_001 | IM用户关系信息详情 | imid从500到1000万的好友关系 |

所包含数据库表，如下

| 序号 | 表名 | 表信息描述 |
| --- | --- | --- |
| 1 | tb_friends_xx | 好友信息详情 |
| 2 | tb_contacts_friends_xx | 天秤通讯录好友控制表 |

注：xx表示数字

### 好友关系 tb_friends

- 好友关系表。
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_friends_xx | 单聊信息详情 | imid%100 |

注：xx表示数字
2.表结构字段详细说明
对好友关系表的字段描述见如下表（表3-2-1）：

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | 用户ID，非空字段，默认为0 | 主键 |
| 2 | guid | VARCHAR(40) | 业务侧编号 |  |
| 3 | obj_imid | INT(11) UNSIGNED | 好友ID，非空字段，默认为0 | 主键 |
| 4 | obj_guid | VARCHAR(40) | 业务侧编号 |  |
| 5 | obj_nickname | VARCHAR(100) | 好友的昵称 |  |
| 6 | obj_avatar_url | VARCHAR(500) | 好友头像，编码格式为UTF-8，默认值为empty string |  |
| 7 | remark | VARCHAR(100) | UID用户对好友的备注，默认值为NULL，字符变为UTF-8 |  |
| 8 | relation | INT(11) UNSIGNED | 用户关系。默认值为0，按位存储，从低位（右）往高位（左）分别为, 0x00000000 无关系 0x00000001 关注 0x00000002 被关注（粉丝） 0x00000004 主动黑名单 0x00000008 被动黑名单 0x00000010 设置了消息免打扰 0x00000020 悄悄借屏蔽 0x00000040 通讯录好友 0x00000080 曾经通讯录好友 |  |
| 9 | relation_from | TINYINT(4) UNSIGNED | 关系来源，默认值为0 0:未知 1:通过搜索 2:通过发现附近的人 3:通讯录自动关注   4:尽调人和借款人  5:红包 6:项目 7:通过附近人的群 8:通过好友的群 |  |
| 10 | relation_time | VARCHAR(512) | 关系操作时间，默认值为NULL，json格式，: [{"r":1,"t":"2015-12-12 15:58:07"},{"r":2,"t":"2015-11-12 16:47:09"}, 其中"r"为关系操作类型，参考relation字段； |  |
| 11 | create_time | DATETIME | 初次建立好友关系的时间，非空字段；默认为'1970-01-01 08:00:00' |  |
| 12 | update_time | DATETIME | 最后更新好友关系的时间，非空字段；默认为'1970-01-01 08:00:00' |  |


### 天秤通讯录好友控制表 tb_contacts_friends

- 好友控制表。
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_contacts_friends_xx | 天秤通讯录好友控制表 | imid%100 |

- 注：xx表示数字
2.表结构字段详细说明
对好友控制表的字段描述见如下表（表3-3-2）：

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | 用户id，非空字段，默认值为0 | 是 |
| 2 | guid | VARCHAR(40) | 业务编号 |  |
| 3 | obj_imid | INT(11) UNSIGNED | 对方id，非空字段，默认值为0 | 是 |
| 4 | obj_guid | VARCHAR(40) | 业务编号 |  |
| 5 | type | TINYINT(4) UNSIGNED | 控制类型，默认值为0， 1：可以，2：不可以 |  |
| 6 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


## IM基础信息 db_im_foundation

   IM基础信息，数据库名为db_im_foundation，该库下是IM系统的基础信息库，主要是涉及四类数据表，即用户与业务关联表(tb_sync_user)，用户手机号与imid映射信息表(tb_phoneno_imid) ,用户昵称与imid映射信息表(tb_nickname_imid)和 业务与群映射信息表(tb_project_group)。该库不需要分段，只存放在指定的一个数据库分片中。
   该库中全是IM基础信息的关系表，关系表中主要字段为用户guid，用户手机号码[加密]及业务工程编号，该字段为varchar类型，需要查询该数据库时，要做hash，即hash(guid)。

| 数据库名 | db_im_foundation |
| --- | --- |
| 描述 | IM系统的基础信息库 |

你我圈的分库原则如下：

| 数据库名 | 描述 | 分库策略 |
| --- | --- | --- |
| db_im_foundation_000 | IM基础信息 | 所有对应数据全部存在一个库中，不分段，只分表 |

所包含数据库表，如下

| 序号 | 表名 | 表信息描述 |
| --- | --- | --- |
| 1 | tb_sync_user_xx | 用户与业务关联表 |
| 2 | tb_phoneno_imid_xx | 用户手机号与imid映射信息表 |
| 3 | tb_nickname_imid_xx | 用户昵称与imid映射信息表 |
| 4 | tb_project_group_xx | 项目与群映射信息表 |


### 普通用户与业务关联表 tb_sync_user

- 普通用户与业务关联表。
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_sync_user_xx | 用户与业务关联表 | hash(guid)%100 |

- 注：xx表示数字
2.表结构字段详细说明
对普通用户与业务关联表的字段描述见如下表（表3-3-1）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | guid | VARCHAR(40) | 业务侧编号，非空字段，主键 | 主键 |
| 2 | imid | INT(11) UNSIGNED | im用户 ID，非空字段，默认值为0 |  |
| 3 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 用户手机号与imid映射信息表 tb_phoneno_imid

- 1.用户手机号与imid映射信息表。
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_phoneno_imid_xx | 用户手机号与imid映射信息 | hash(sphoneno)%100 |

- 注：xx表示数字
2.表结构字段详细说明
对普通用户关键信息表的字段描述见如下表（表3-3-2）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | sphoneno | VARCHAR(40) | 用户手机号码，非空字段，主键 | 主键 |
| 2 | imid | INT(11) UNSIGNED | im用户 ID，非空字段，默认值为0 |  |
| 3 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 用户昵称与imid映射信息表tb_nickname_imid

- 1.用户昵称与imid映射信息表。
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_nickname_imid_xx | 昵称与imid映射表 | hash(nickname)%100 |

- 注：xx表示数字
2.表结构字段详细说明
对用户昵称与imid映射信息表的字段描述见如下表（表3-3-3）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | nickname | VARCHAR(100) | 用户昵称，非空字段，主键 | 主键 |
| 2 | imid | INT(11) UNSIGNED | im用户 ID，非空字段,默认值为0 |  |
| 3 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 业务与群映射信息表 tb_project_group

- 业务与群映射信息表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_project_group_xx | 业务与群关联映射信息 | hash(project_id)%100 |

注：xx表示数字
2.表结构字段详细说明
对业务与群映射信息表的字段描述见如下表（表3-7-2）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | project_id | VARCHAR(40) | 群业务标识ID，非空字段，编码格式为UTF-8， | 主键 |
| 2 | group_id | INT(11) UNSIGNED | 群Id,非空字段；默认值为0 |  |
| 3 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


## IM群消息存储 db_groupmsg

消息存储，数据库名为db_groupmsg,该库下主要分为群聊消息和单聊消息，即群聊(tb_groupchat_xx) 和 群通知消息(tb_group_notify_xx) , 

| 数据库名 | db_groupmsg |
| --- | --- |
| 描述 | IM群消息存储详情 |

消息存储的分库原则如下：

| 数据库名 | 描述 | 分库策略 |
| --- | --- | --- |
| db_groupmsg_000 | 消息存储详情 | group_id从1到500万的聊天消息 |
| db_groupmsg_001 | 消息存储详情 | group_id从1到500万的聊天消息 |

所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 |
| --- | --- | --- |
| 1 | tb_group_chat_xx | 群聊信息详情 |
| 2 | tb_group_notify_xx | 群通知消息 |

注：xx表示数字

### 群聊信息 tb_group_chat

- 群聊信息分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_group_chat_xx | 群聊信息详情 | group_id%100 |

注：X表示数字
2.表结构字段详细说明
对群聊消息的字段描述见如下表（表3-5-1）：

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | msgid | BIGINT(24) UNSIGNED | 微秒级时间消息戳，非空字段，默认值为0 |  |
| 2 | send_imid | INT(11) UNSIGNED | 发送者id，默认值为0 |  |
| 3 | guid | VARCHAR(40) | 业务侧编号 |  |
| 4 | group_id | INT(11) UNSIGNED | 群id；非空字段，默认值为0 |  |
| 5 | msg_type | INT(11) UNSIGNED | 聊天消息类型，有15种类型，见common.proto文件；默认值为0 |  |
| 6 | attribute_body | BLOB | 属性的JSON体，根据msgtype进行判断。存放经、纬度，图片大小，语音时长，消息URL,消息体，红包信息等；编码格式为UTF-8 |  |
| 7 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 群通知消息 tb_group_notify

- 群通知信息表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_group_notify_xx | 群通知消息详情 | group_id%100 |

- 注：X表示数字
2.表结构字段详细说明
对通知类群聊消息的字段描述见如下表（表3-6-2）：

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | msgid | BIGINT(24) UNSIGNED | 微秒级时间消息戳，非空字段，默认值为0 |  |
| 2 | group_id | INT(11) UNSIGNED | 群id，默认值为0 |  |
| 3 | notify_type | INT(11) UNSIGNED | 通知消息类型，有15种类型，见common.proto文件；默认值为0 |  |
| 4 | attribute_body | BLOB | 属性的JSON体，根据msgtype进行判断。存放经、纬度，图片大小，语音时长，消息URL,消息体，红包信息等；编码格式为UTF-8 |  |
| 5 | cmd_identify | INT(11) UNSIGNED | 命令字，区分消息类型场景；默认为0 |  |
| 6 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


## IM个人消息存储 db_singlemsg

通知消息，数据库名为db_singlemsg,该库下主要分为个人消息，即个人通知消息(tb_personal_notify_xx) ,群成员接受情况(tb_groupmsg_recv_xx) 和 单聊(tb_singlechat_xx)。

| 数据库名 | db_singlemsg |
| --- | --- |
| 描述 | IM个人消息消息存储详情 |

消息存储的分库原则如下：

| 数据库名 | 描述 | 分库策略 |
| --- | --- | --- |
| db_singlemsg_000 | IM个人消息消息存储详情 | imid从1到500万的通知消息 |
| db_singlemsg_001 | IM个人消息消息存储详情 | imid从1到500万的通知消息 |

所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 |
| --- | --- | --- |
| 1 | tb_groupmsg_recv_xx | 群成员接受情况 |
| 2 | tb_singlechat_xx | 单聊信息详情 |
| 3 | tb_personal_notify_xx | 个人通知消息 |


### 群成员接收情况表 tb_groupmsg_recv

1.群成员接受情况表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_groupmsg_recv_xx | 群成员接收情况 | imid%100 |

注：X表示数字
2.表结构字段详细说明
对群成员接收情况表的字段描述见如下表（表3-5-2）：

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | recv_imid | INT(11) UNSIGNED | 接受者id，默认值为0 | 主键 |
| 2 | recv_guid | VARCHAR(40) | 业务侧编号 |  |
| 3 | group_id | INT(11) UNSIGNED | 群Id，非空字段，默认值为0 | 主键 |
| 4 | msgid | BIGINT(24) UNSIGNED | 微秒级时间消息戳，非空字段，默认值为0 |  |
| 5 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 单聊信息 tb_single_chat

- 单聊信息表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_single_chat_xx | 单聊信息详情 | imid%100 |

注：X表示数字
     2.表结构字段详细说明
对单聊消息的字段描述见如下表（表3-5-3）：

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | msgid | BIGINT(24) UNSIGNED | 微秒级时间消息戳，非空字段,默认值为0 |  |
| 2 | imid | INT(11) UNSIGNED | 用户Id，非空字段，默认值为0 |  |
| 3 | send_imid | INT(11) UNSIGNED | 发送者id，默认值为0 |  |
| 4 | send_guid | VARCHAR(40) | 业务侧编号 |  |
| 5 | recv_imid | INT(11) UNSIGNED | 接受者id，默认值为0 |  |
| 6 | recv_guid | VARCHAR(40) | 业务侧编号 |  |
| 7 | msg_type | INT(11) UNSIGNED | 聊天消息类型，有15种类型，见common.proto文件；默认值为0 |  |
| 8 | attribute_body | BLOB | 属性的JSON体，根据msgtype进行判断。存放经、纬度，图片大小，语音时长，消息URL,消息体，红包信息等；编码格式为UTF-8 |  |
| 9 | send_flag | TINYINT(4) UNSIGNED | 该条消息是否成功推送给客户端，默认为0，0：发送失败，1：发送成功 |  |
| 10 | record_time | DATETIME | 该条记录的生成时间;默认值为 '1970-01-01 08:00:00' |  |


### 个人通知消息 tb_personal_notify

- 个人通知消息表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_personal_notify_xx | 个人通知消息详情 | imid%100 |

注：X表示数字
2.表结构字段详细说明
对通知类单聊消息的字段描述见如下表（表3-6-1）：

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | msgid | BIGINT(24) UNSIGNED | 微秒级时间消息戳，非空字段，默认值为0 |  |
| 2 | imid | INT(11) UNSIGNED | 用户Id，非空字段，默认值为0 |  |
| 3 | send_imid | INT(11) UNSIGNED | 发送者id，默认值为0 |  |
| 4 | send_guid | VARCHAR(40) | 业务侧编号 |  |
| 5 | recv_imid | INT(11) UNSIGNED | 接受者id，默认值为0 |  |
| 6 | recv_guid | VARCHAR(40) | 业务侧编号 |  |
| 7 | msg_type | INT(11) UNSIGNED | 聊天消息类型，有15种类型，见common.proto文件；默认值为0 |  |
| 8 | attribute_body | BLOB | 属性的JSON体，根据msgtype进行判断。存放经、纬度，图片大小，语音时长，消息URL,消息体，红包信息等；编码格式为UTF-8 |  |
| 9 | send_flag | TINYINT(4) UNSIGNED | 该条消息是否成功推送给客户端，默认为0，0：发送失败，1：发送成功 |  |
| 10 | cmd_identify | INT(11) UNSIGNED | 命令字，区分消息类型场景 |  |
| 11 | record_time | DATETIME | 该条记录的生成时间;默认值为 '1970-01-01 08:00:00' |  |
| 12 | deadline | DATETIME | 截止时间，这个时间之后当前通知将失效并不再推送 |  |


## IM群管理 db_group_manager

IM群管理，数据库名为db_group_manager，该库下是群相关的信息表，主要是涉及4类数据表，即申请入群审核(tb_group_apply_verify),群信息(tb_groupinfo), 邀请入群审核(tb_group_invite_verify) 和 群用户(tb_group_member)。
该库中全是群管理的关系表，关系表中主要字段为群号group_id，则分表的原则为group_id%100。

| 数据库名 | db_group_manager |
| --- | --- |
| 描述 | 群管理 |

IM群管理的分库原则如下：

| 数据库名 | 描述 | 分库策略 |
| --- | --- | --- |
| db_group_manager_000 | 群管理 | group_id从1到500万的群 |
| db_group_manager_001 | 群管理 | group_id从500到1000万的群 |

所包含数据库表，如下：

| 序号 | 表名 | 表信息描述 |
| --- | --- | --- |
| 1 | tb_groupinfo | 群信息 |
| 2 | tb_group_member | 群用户 |
| 3 | tb_group_apply_verify | 申请入群审核 |
| 4 | tb_group_invite_verify | 邀请入群审核 |


### 群信息 tb_groupinfo

- 群信息表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_groupinfo_xx | 群信息 | group_id%100 |

注：xx表示数字
2.表结构字段详细说明
对群信息表的字段描述见如下表（表3-7-1）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | group_id | INT(11) UNSIGNED | 群Id,非空字段，主键， | 主键 |
| 2 | owner_imid | INT(11) UNSIGNED | 创建人ID, 非空字段， |  |
| 3 | owner_guid | VARCHAR(40) | 业务层ID，非空字段 |  |
| 4 | group_name | VARCHAR(100) | 群名称，非空字段，编码格式为UTF-8，默认值为empty string |  |
| 5 | group_theme | VARCHAR(255) | 群主题，编码格式为UTF-8，默认值为empty string |  |
| 6 | group_avatar | VARCHAR(500) | 群头像，编码格式为UTF-8，默认值为empty string |  |
| 7 | group_intro | VARCHAR(250) | 群简介，编码格式为UTF-8，默认值为empty string，默认不用 |  |
| 8 | group_type | TINYINT(4) UNSIGNED | 群类别 1，业务群(业务发起)，2普通群(app发起)，3讨论组; 非空字段，默认值为0 |  |
| 9 | is_valid | TINYINT(4) UNSIGNED | 群是否有效 1：有效，2 ：失效；非空字段,默认值为0 |  |
| 10 | group_scale | INT(11) UNSIGNED | 群规模；非空字段，默认为0 |  |
| 11 | group_num | INT(11) UNSIGNED | 群成员数量,默认值为0 |  |
| 12 | is_public | TINYINT(4) UNSIGNED | 群公开：1允许游客访问  2不允许游客访问；非空字段,默认值为0 |  |
| 13 | join_verify_type | TINYINT(4) UNSIGNED | 加群验证 1 允许任何人 2需身份认证 3 不允许任何人；非空字段，默认为0 |  |
| 14 | group_loc | VARCHAR(255) | 群所在地区，编码格式为UTF-8，默认值为empty string |  |
| 15 | group_liveness | INT(11) UNSIGNED | 群活跃度 |  |
| 16 | create_time | DATETIME | 群创建时间；非空字段，默认值为非空字段, '1900-00-00 00:00:00' |  |
| 17 | update_time | DATETIME | 群更新时间；非空字段，默认值为非空字段, '1900-00-00 00:00:00' |  |
| 18 | project_id | VARCHAR(40) | 群业务标识ID,编码格式为UTF-8 | 不是项目群就填空 |


### 群成员 tb_group_member

- 群成员表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_group_member_xx | 群用户 | group_id%100 |

注：xx表示数字
2.表结构字段详细说明
对群成员表的字段描述见如下表（表3-7-3）：

| 序号 | 字段 | 类型 | 描述 | 备注 |
| --- | --- | --- | --- | --- |
| 1 | group_id | INT(11) UNSIGNED | 群Id；非空字段，默认值为0 | 主键 |
| 2 | imid | INT(11) UNSIGNED | 用户ID, 非空字段；默认值为0 | 主键 |
| 3 | guid | VARCHAR(40) | 业务侧面的业务编号 |  |
| 4 | card | VARCHAR(100) | 群名片 | 用户在群中的昵称 |
| 5 | nickname | VARCHAR(100) | 用户在群中别名，默认使用用户昵称，非空字段，编码格式为UTF-8；默认值为empty string |  |
| 6 | avatar | VARCHAR(255) | 用户头像，编码格式为UTF-8，默认值为empty string |  |
| 7 | join_time | DATETIME | 非空字段，用户加入时间，默认值为’1900-00-00 00:00:00’ |  |
| 8 | group_role | TINYINT(4) UNSIGNED | 群角色 3 普通成员 2管理员 1群主; 非空字段，默认为0 |  |
| 9 | block_type | TINYINT(4) UNSIGNED | 是否屏蔽群消息  1屏蔽， 2不屏蔽；非空字段，默认值为0 |  |
| 10 | biz_type | TINYINT(4) UNSIGNED | 用户在群里面的角色( = 0; //普通人, 非业务群会使用这个角色                                  = 1; //净调人                              = 2; //借款人                              = 3; //投资人                              = 4; //担保人                                           = 13;//尽调投资     = 43;//担保投资 ) |  |
| 11 | liveness | INT(11) UNSIGNED | 活跃群用户 |  |
| 12 | member_status | TINYINT(4) UNSIGNED | 成员状态： 3 已删除（退群等） |  |
| 13 | update_time | datetime | 最后更新时间 |  |


### 申请入群审核 tb_group_apply_verify

- 申请入群审核表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_group_apply_verify_xx | 申请入群审核 | group_id%100 |

注：xx表示数字
2.表结构字段详细说明
对申请入群审核表的字段描述见如下表（表3-7-4）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | group_id | INT(11) UNSIGNED | 群Id，非空字段；默认值为0 |  |
| 2 | msgid | BIGINT(24) UNSIGNED | 每次申请入群的批次标识，默认值为0 |  |
| 3 | apply_imid | INT(11) UNSIGNED | 申请人ID,非空字段；默认值为0 |  |
| 4 | apply_guid | VARCHAR(40) | 业务侧编号 |  |
| 5 | nickname | VARCHAR(100) | 申请人昵称，编码格式为UTF-8；默认值为NULL |  |
| 6 | apply_avatar | VARCHAR(255) | 申请人头像，编码格式为UTF-8；默认值为empty string |  |
| 7 | apply_time | DATETIME | 申请时间，非空字段；默认值为'1900-00-00 00:00:00' |  |
| 8 | apply_content | VARCHAR(255) | 申请备注，编码格式为UTF-8；默认值为NULL |  |
| 9 | verify_time | DATETIME | 非空字段，审核时间，默认值为'1900-00-00 00:00:00' |  |
| 10 | verify_status | TINYINT(4) UNSIGNED | 非空字段，审核状态 1 待审核 2 通过 3拒绝 4忽略;默认值为0 |  |
| 11 | manager_id | INT(11) UNSIGNED | 群主ID（如果群不需要验证就填写群主，否则填写审批的管理员ID），非空字段；默认值为0 |  |
| 12 | reply_content | VARCHAR(255) | 申请回复备注，非空字段，编码格式为UTF-8；默认值为NULL |  |
| 15 | manager_notify_flag | TINYINT(4) UNSIGNED | 管理员通知标识：0初始  1已通知， 2未通知;非空字段，默认值为0 |  |
| 16 | manager_notify_time | BIGINT(24) UNSIGNED | 通知管理员时间 |  |
| 17 | apply_notify_flag | TINYINT(4) UNSIGNED | 申请者通知标识：0 初始，1已通知，2未通知;默认值为0 |  |
| 18 | apply_notify_time | BIGINT(24) UNSIGNED | 申请者通知时间 |  |


### 邀请入群审核 tb_group_invite_verify

- 邀请入群审核表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_group_invite_verify_xx | 邀请入群审核 | group_id%100 |

注：xx表示数字
2.表结构字段详细说明
对邀请入群审核表的字段描述见如下表（表3-7-5）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | group_id | INT(11) UNSIGNED | 群Id；默认值为0 |  |
| 2 | msgid | BIGINT(24) UNSIGNED | 每次申请入群的批次标识，默认值为0 |  |
| 3 | be_invite_imid | INT(11) UNSIGNED | 被邀请人ID，默认值为0 |  |
| 4 | be_invite_guid | VARCHAR(40) | 业务侧业务编号 |  |
| 5 | be_invite_name | VARCHAR(100) | 被邀请人昵称，编码格式为UTF-8，默认值为NULL |  |
| 6 | be_invite_avatar | VARCHAR(255) | 被邀请者头像，编码格式为UTF-8，默认值为empty string |  |
| 7 | invite_imid | INT(11) UNSIGNED | 邀请人ID，默认值为0 |  |
| 8 | invite_guid | VARCHAR(40) | 业务侧的编号 |  |
| 9 | invite_name | VARCHAR(100) | 被邀请人昵称，编码格式为UTF-8，默认值为NULL |  |
| 10 | invite_avatar | VARCHAR(255) | 被邀请者头像，编码格式为UTF-8，默认值为empty string |  |
| 11 | invite_time | DATETIME | 邀请时间，非空字段，默认值为'0000-00-00 00:00:00' |  |
| 12 | invite_content | VARCHAR(255) | 邀请备注，编码格式为UTF-8；默认值为empty string |  |
| 13 | verify_time | DATETIME | 验证审核时间，非空字段，默认值为'0000-00-00 00:00:00' |  |
| 14 | verify_status | TINYINT(4) UNSIGNED | 审核状态1 待审核 2 通过 3拒绝 4忽略；非空字段，默认值为0 |  |
| 15 | manager_id | INT(11) UNSIGNED | 群主ID（如果群不需要验证就填写群主，否则填写审批的管理员ID）；非空字段，默认值为0 |  |
| 16 | reply_content | VARCHAR(255) | 邀请回复备注，非空字段，编码格式为UTF-8；默认值为empty string |  |
| 17 | manager_notify_flag | TINYINT(4) UNSIGNED | 管理员通知标识：0初始  1已通知， 2未通知；非空字段，默认值为0 |  |
| 18 | manager_notify_time | BIGINT(24) UNSIGNED | 通知管理员时间；非空字段，默认值为0 |  |


## 群信息搜索db_search_groupinfo

群信息搜索，数据库名为db_search_groupinfo，该库下是搜索群类信息表，主要是涉及1类数据表，即群搜索(tb_search_groupinfo)。
该库中只有一张表结构，不进行分表操作。没有主键，只有一个索引。

| 数据库名 | db_search_groupinfo |
| --- | --- |
| 描述 | 群搜索 |

群搜索的分库原则如下：

| 数据库名 | 描述 | 分库策略 |
| --- | --- | --- |
| db_search_groupinfo | 群搜索 | IM系统所有群信息 |

所包含数据库表，如下：

| 序号 | 表名 | 表信息描述 | 分表规则 |
| --- | --- | --- | --- |
| 1 | tb_search_groupinfo | 群搜索 | 不分表 |


### 群信息搜索 tb_search_groupinfo

- 1. 群信息搜索表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_search_groupinfo | 用于搜索群消息详情 | 暂时不用分表，采用一个大表存储群信息，后续用检索server替换 |

2.表结构字段详细说明
对群信息搜索的字段描述见如下表（表3-6-3）：

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | group_name | VARCHAR(100) | 群名称，非空字段，编码格式为UTF-8 | index |
| 2 | group_id | INT(11) UNSIGNED | 群id；非空字段,默认值为0 |  |
| 3 | group_avatar | VARCHAR(500) | 群头像，编码格式为UTF-8，默认值为empty string |  |
| 4 | group_intro | VARCHAR(250) | 群简介，编码格式为UTF-8，默认值为empty string，默认不用 |  |
| 5 | group_loc | VARCHAR(255) | 群所在地区，编码格式为UTF-8，默认值为empty string |  |
| 6 | record_time | DATETIME | 该条记录的生成时间;默认值为NULL |  |

现在暂用 group_name在数据库里查找进行搜索。

## 朋友圈 db_community

朋友圈，数据库名为db_community，该库下是朋友圈相关的信息表。

| 数据库名 | db_community |
| --- | --- |
| 描述 | 朋友圈 |

朋友圈的分库原则如下：

| 数据库名 | 描述 | 分库策略 |
| --- | --- | --- |
| db_community_000 | 朋友圈 | imid从1到500万的用户 |
| db_community_001 | 朋友圈 | imid从500万到1000万的用户 |

所包含数据库表，如下：

| 序号 | 表名 | 表信息描述 | 分表规则 |
| --- | --- | --- | --- |
| 1 | tb_blog | 说说表 | imid%100 |


### 说说表 tb_blog

- 分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_blog_xx | 用户说说、评论、回复、点赞 | blog_imid%100 |

注：xx表示数字
2.表结构字段详细说明
对说说表的字段描述见如下表（表3-8-1）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | blog_imid | INT(11) UNSIGNED | 发表说说人ID，非空字段，默认值为0 | 分表依据，index |
| 2 | msgid | BIGINT(24) UNSIGNED | 消息唯一标识ID；非空字段，默认值为0 |  |
| 3 | pmsgid | BIGINT(24) UNSIGNED | 父消息ID（说说本身父消息ID为0，评论的父消息ID为说说ID，回复的父消息ID为回复对象消息ID） |  |
| 4 | blog_msgid | BIGINT(24) UNSIGNED | 说说消息ID（如果是说说，msgid与blog_msgid一致） | index |
| 5 | imid | INT(11) UNSIGNED | 发表人（说说、评论人、回复人、点赞）ID; 非空字段，默认值为0 |  |
| 6 | guid | VARCHAR(40) | imid对应的业务侧的编号 |  |
| 7 | nickname | VARCHAR(100) | 发表人（说说、评论人、回复人、点赞）用户昵称 |  |
| 8 | avatar | VARCHAR(255) | 发表人（说说、评论人、回复人、点赞）用户头像，编码格式为UTF-8，默认值为empty string |  |
| 9 | pimid | INT(11) UNSIGNED | 父消息发表人imid（说说发表人、评论人、回复人）ID; 非空字段，默认值为0 |  |
| 10 | pnickname | VARCHAR(100) | 父消息发表人用户昵称 |  |
| 11 | msg_type | TINYINT(4) UNSIGNED | 内容类型 ，非空字段，默认值为0 0 点赞 1 文字  2 图片  3 自定义模板分享  4 语音  5 视频 |  |
| 12 | msg_content | TEXT | 消息内容，json结构 |  |
| 13 | msg_status | TINYINT(4) UNSIGNED | 消息状态： 0对所有粉丝可见，可评论回复  1不可见(仅自己可见)  2 仅好友可见 3 已删除 4 不可评论或回复 5 仅好友可评论或回复 |  |
| 14 | target_user | TEXT | 目标用户，json结构，包括@用户列表，允许查看用户列表，禁止查看用户列表。 |  |
| 15 | create_time | DATETIME | 点赞创建时间；非空字段，默认值为'1900-01-01 08:00:00' |  |
| 16 | update_time | DATETIME | 点赞更新时间；非空字段，默认值为'1900-01-01 08:00:00' |  |


## 公众号平台 db_official


### 公众号附加信息表 tb_official

1.公众号附加信息表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_xx | 公众号附加信息表 | offical_imid%100 |

注：xx表示数字
2. 表结构字段详细说明
公众号附加信息表的字段描述见如下表（表3-1-2）

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | official_imid | INT(11) UNSIGNED | 公众号IM用户ID，非空字段，默认值为0 | 主键 |
| 2 | description | VARCHAR(512) | 公众号简介 |  |
| 3 | accout_type | TINYINT(4) UNSIGNED | 公众号类型： 1:公众号-系统订阅号 用户注册后自动订阅， 2:公众号-用户订阅号 用户自己手动关注 3:公众号-服务号 4:公众号-企业号 默认值为0 |  |
| 4 | default_userset | INT(11) UNSIGNED | 默认分组： 普通公众号创建时会自己生成一个默认分组。 系统公众号填0 |  |
| 5 | userset_binding_list | VARCHAR(1024) | 发布对象权限分组id： 10000:所有人 (用户第一次登录IM系统自动加入) 10001:钱小宝、 10002:钱大宝、 10003:借口身份用户、 10004:尽调身份用户 10005:担保身份用户 (由管理员创建，并拉入分组) 2xxxx::关注订阅用户(订阅创建时自己生成 ) 最多支持100个 格式：json {	"userset_id":[10001,10002] } |  |
| 6 | pub_cycle_frequency | TINYINT(4) UNSIGNED | 发布的频率控制周期： 1:每天调用n次 2:每月调用n次 3:每周调用n次 |  |
| 7 | pub_limit_num | TINYINT(4) UNSIGNED | 发布的频率控制次数： 结合pub_frequency_cycle ，N次 |  |
| 8 | current_pub_num | TINYINT(4) UNSIGNED | 当前周期已经发布的次数，默认值为0 |  |
| 9 | last_pub_time | DATETIME | 最后一次的发布时间;默认值为'1970-01-01 08:00:00' |  |
| 10 | user_max_receive | TINYINT(4) UNSIGNED | 用户每天最可多接收条数，默认值为0 |  |
| 11 | del_flag | TINYINT(4) UNSIGNED | 删除标识（0未删除，1已删除)默认值为0 |  |
| 12 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 系统公众号列表 tb_official_system_account

1.系统公众号分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_system_account | 所有的系统公众号都放在这张表时 | 只有一张表 |

 2. 表结构字段详细说明
系统公众号表的字段描述见如下表（表3-1-2）

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | official_imid | INT(11) UNSIGNED | 系统公众号ID |  |
| 2 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 公众号用户分组信息表 tb_official_userset

1.公众号用户分组信息分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_userset_xx | 记录每个分组的详细信息 | userset_id%100 |

注：xx表示数字
2. 表结构字段详细说明
公众号用户分组信息表的字段描述见如下表（表3-1-2）

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | userset_id | INT(11) UNSIGNED | 公众号分组ID,非空字段，默认值为0 | 主键 |
| 2 | official_imid | INT(11) UNSIGNED | 公众号ID。 平台所有用户、 管理员创建的分组填0 |  |
| 3 | userset_type | TINYINT(4) UNSIGNED | 公众号分组类型： 1、平台所有用户 由管理员创建 3、用户订阅 |  |
| 4 | userset_name | VARCHAR(128) | 公众号分组名称 |  |
| 5 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 公众号用户分组成员表 tb_official_userset_member

1.公众号用户分组成员表分表策略
所包含数据库表及分表策略，如下
按userset_id分库

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_userset_member_xx | 记录了每个分组所有成员信息 | imid%100 |

注：xx表示数字
2. 表结构字段详细说明
公众号用户分组成员表的字段描述见如下表（表3-1-2）

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | userset_id | INT(11) UNSIGNED | 公众号分组ID,非空字段，默认值为0 | MUL(userset_id,record_time) |
| 2 | imid | INT(11) UNSIGNED | 分组用户IM ID,非空字段，默认值为0 | index |
| 3 | guid | VARCHAR(40) | 业务则ID，非空字段，默认值为NULL |  |
| 4 | remark | VARCHAR(128) | 用户备注： 公众号管理员可以对用户进行备注 |  |
| 5 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' | MUL(userset_id,record_time) |


### 公众号用户列表 tb_official_subscriber

1.公众号用户列表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_subscriber_xx | 记录了订阅了此公众的用户列表 | official_imid%100 |

注：xx表示数字
2. 表结构字段详细说明
公众号用户列表的字段描述见如下表 

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | official_imid | INT(11) UNSIGNED | 公众号用户IM ID,非空字段，默认值为0 | index |
| 2 | imid | INT(11) UNSIGNED | 用户IM ID,非空字段，默认值为0 |  |
| 3 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 用户订阅的公众号列表 tb_official_user_subscribe

1.用户订阅的公众号列表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_user_subscribe_xx | 记录了用户订阅了哪些公众号 | imid%100 |

注：xx表示数字
2. 表结构字段详细说明
用户订阅的公众号列表的字段描述见如下表（表3-1-2）

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | 用户IM ID，非空字段，默认值为0 | index |
| 2 | official_imid | INT(11) UNSIGNED | 公众号IM ID，非空字段，默认值为0 |  |
| 3 | current_num | INT(11) UNSIGNED | 当前周期内已经接收消息条数，默认值为0 |  |
| 4 | first_receive_time | DATETIME | 当前周期内第一条消息接收时间；默认值为'1970-01-01 08:00:00' |  |
| 5 | last_receive_time | DATETIME | 当前周期内最后一条消息接收时间；默认值为'1970-01-01 08:00:00' |  |
| 6 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 公众号消息详情存储表 tb_official_msg

- 公众号消息详情存储分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_msg_xx | 记录了公众号发过的消息详细记录 | send_imid%100 |

注：X表示数字
2.表结构字段详细说明
对公众号消息详情的字段描述见如下表（表3-5-1）：

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | msgid | BIGINT(24) UNSIGNED | 微秒级时间消息戳，非空字段，默认值为0 | 主键 |
| 2 | send_imid | INT(11) UNSIGNED | 发送者id，默认值为0 |  |
| 3 | guid | VARCHAR(40) | 业务侧编号，默认为NULL |  |
| 4 | userset_id | INT(11) UNSIGNED | 分组id；默认值为0 |  |
| 5 | attribute_body | BLOB | 属性的JSON体，根据msgtype进行判断。存放经、纬度，图片大小，语音时长，消息URL,消息体，红包信息等；编码格式为UTF-8 |  |
| 6 | msg_expires | DATETIME | 消息失效时间： 当前时间+消息失效小时 |  |
| 7 | msg_accept_time | VARCHAR(1024) | 接收时间的,防止休息时间打扰用户: 例： Json: [     {         "start":{             "hour":12,             "min":0         },         "end":{             "hour":14,             "min":0         }     },     {         "start":{             "hour":17,             "min":30         },         "end":{             "hour":19,             "min":0         }     } ] |  |
| 8 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 用户公众号分组消息接收状态表 tb_official_user_last_recv

1.用户公众号分组消息接收状态表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_user_last_recv_xx | 记录了用户每个分组最后一条接收到的消息 | imid%100 |

注：xx表示数字
2. 表结构字段详细说明
用户公众号分组消息接收状态表的字段描述见如下表 

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | 用户IM ID,默认值为0 | 主键，index |
| 2 | userset_id | INT(11) UNSIGNED | 分组id，默认值为0 | 主键 |
| 3 | msgid | BIGINT(24) UNSIGNED | 微秒级时间消息戳，非空字段，默认为0 |  |
| 4 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 公众号分组发出消息记录表 tb_official_userset_msg

1.公众号分组发出消息记录表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_userset_msg_xx | 记录了每一个分组发过的所有消息 | userset_id%100 |

注：xx表示数字
2. 表结构字段详细说明
公众号分组发出消息记录表的字段描述见如下表 

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | userset_id | INT(11) UNSIGNED | 分组id,默认值为0 | index |
| 2 | send_imid | INT(11) UNSIGNED | 发送消息的公众号id，默认值为0 |  |
| 3 | msgid | BIGINT(24) UNSIGNED | 微秒级时间消息戳，非空字段，默认值为0 |  |
| 4 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


### 公众号发指定用户的消息收听列表 tb_official_specified_user_msg

1.公众号发指定用户的消息收听列表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_official_specified_user_msg_xx | 给特定用户发消息时，记录了用户未接收的消息列表。用户接收完后删除 | imid%100 |

注：xx表示数字
2. 表结构字段详细说明
公众号发指定用户的消息收听列表的字段描述见如下表 

| 序号 | 字段 | 类型 | 描述 | 是否为主键 |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | 用户IM ID | index |
| 2 | send_imid | INT(11) UNSIGNED | 发送者id，默认值为0 |  |
| 3 | msgid | BIGINT(24) UNSIGNED | 微秒级时间消息戳，非空字段 |  |
| 4 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


## 用户管理 db_user_management

用户管理，数据库名为db_user_management，该库下是用户管理信息表。
该库分表以imid%100分表。

| 数据库名 | db_user_management |
| --- | --- |
| 描述 | 用户管理 |

分库原则如下：

| 数据库名 | 描述 | 分库策略 |
| --- | --- | --- |
| db_user_management_000 | 用户管理 | 不分库 |


### 用户惩罚信息 tb_punish

- 用户权限分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_punish_xx | 用户惩罚信息 | imid%10 |

注：xx表示数字
2. 表结构字段详细说明
对用户权限表的字段描述见如下表（表3-9-4）：

| 序号 | 字段 | 类型 | 描述 | Key |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | 用户ID，默认为0 | 主键，index |
| 2 | guid | VARCHAR(40) | 业务侧的编号 |  |
| 3 | permissions | TINYINT(4) UNSIGNED | 被禁止的权限 0 禁言 1 你我圈 2 建群 3 关注 4 登录 | 主键 |
| 4 | start_time | DATETIME | 用户被禁止行为的开始时间,默认时间为'1970-01-01 08:00:00' |  |
| 5 | end_time | DATETIME | 用户被禁止行为的结束时间,默认为'1970-01-01 08:00:00' |  |
| 6 | description | VARCHAR(250) | 描述，编码格式为UTF-8，默认为NULL |  |
| 7 | manager_id | INT(11) UNSIGNED | 后台管理员ID,非空字段 |  |
| 8 | state | TINYINT(4) UNSIGNED | 状态: 0表示无效, 1表示有效；默认值为1 |  |
| 9 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |
| 10 | forbidden_type | INT(4) UNSIGNED | 禁止类型:默认值为0 |  |


### 举报详细表 tb_report_list

1.举报详细表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_report_list | 举报详细表 | 不分表 |

注：xx表示数字
2. 表结构字段详细说明
对举报详细表的字段描述见如下表（表3-9-3）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | msgid | BIGINT(24) UNSIGNED | 举报事件消息ID | 主键 |
| 2 | imid | INT(11) UNSIGNED | 举报记录所有者imid |  |
| 3 | informant_imid | INT(11) UNSIGNED | 举报人imid |  |
| 4 | informant_guid | VARCHAR(40) | 举报人guid |  |
| 5 | be_reported_imid | INT(11) UNSIGNED | 被举报者imid |  |
| 6 | be_reported_guid | VARCHAR(40) | 被举报者guid |  |
| 7 | content | TEXT | 举报内容消息json格式,编码格式为UTF-8；默认值为NULL { } |  |
| 8 | state | TINYINT(4) UNSIGNED | 举报事件状态，1被举报初始化，2审核限制功能，3取消举报，4过期；默认值为0 |  |
| 9 | report_type | TINYINT(4) UNSIGNED | 举报类型,默认值为0， |  |
| 10 | remark | VARCHAR(1024) | 用户举报说明,编码格式为UTF-8，默认值为NULL |  |
| 11 | manager_id | INT(11) UNSIGNED | 管理员Id，默认为0 |  |
| 12 | manager_desc | VARCHAR(250) | 管理员描述，编码格式为UTF-8，默认为NULL（） |  |
| 13 | report_time | DATETIME | 举报时间，默认值为'1970-01-01 08:00:00' |  |
| 14 | verify_time | DATETIME | 核实时间，默认值为'1970-01-01 08:00:00' |  |
| 15 | picture_evidence | VARCHAR(2048) | 图片证据，最多10张图片 |  |
| 16 | be_reported_group_id | INT(11) UNSIGNED | 举报的群id |  |
| 17 | be_reported_group_name | VARCHAR(250) | 举报的群名称 |  |
| 18 | report_source | TINYINT(4) UNSIGNED | 举报来源(1:个人单聊、2:个人群聊、3:你我圈,4:普通群、5、项目群) |  |


## 系统全局表 db_im_oss


### 敏感词表 tb_sensitive_words

1.敏感词表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_sensitive_words | 敏感词表 | 不分表 |

注：xx表示数字
2. 表结构字段详细说明
对举报详细表的字段描述见如下表：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | sensitive_type | TINYINT(4) UNSIGNED | 敏感词库类型（1：通用、2、公司） |  |
| 2 | secondary_type | TINYINT(4) UNSIGNED | 二级类型 |  |
| 3 | word | VARCHAR(40) | 敏感词 |  |


### 举报提示语表 tb_report_prompt

1.举报提示语表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_report_prompt | 举报提示语表 | 不分表 |

注：xx表示数字
2. 表结构字段详细说明
对举报详细表的字段描述见如下表（表3-9-3）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | id | BIGINT(24) UNSIGNED | 举报提示语ID | 主键 是由type与forbid_type移位组成的 |
| 2 | type | INT(11) UNSIGNED | 行为禁止项（0 禁言1 你我圈2 建群3 关注4 登录） |  |
| 3 | forbidden_type | INT(4) UNSIGNED | 禁止类型 |  |
| 4 | content | VARCHAR(40) | 提示语内容 |  |
| 5 | update_time | DATETIME | 举报时间，默认值为'1970-01-01 08:00:00' |  |


### 客户端热修复补丁更新表 tb_app_release_patch

1.举报提示语表分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_app_release_patch | 客户端热修复补丁更新表 | 不分表 |

注：xx表示数字
2. 表结构字段详细说明
对客户端热修复补丁更新表的字段描述见如下表（表3-9-3）：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | patch_id | BIGINT(24) UNSIGNED | 补丁包id由服务器生成 | "patch_id":1455517781097798, |
| 2 | client_type | INT(11) UNSIGNED | 终端类型(1:安卓,2:IOS)4 登录） | 终端类型(1:安卓,2:IOS) |
| 3 | device_info | VARCHAR(512) | 设备详细信息(可不填，填了可以精确匹配具体哪个设备) | "deviceInfo": //设备详细信息(可不填，填了可以精确匹配具体哪个设备)      {           uuid="f7943b242b1b4363ad9b3f5f731eef95",           verson="9.3.4",           model="ME279CH"      }， |
| 4 | patch_version | INT(4) UNSIGNED | 补丁适合的版本 | "patch_version":381;//补丁适合的版本 |
| 5 | patch_url | VARCHAR(1024) | 补丁路径 | "patch_url":["http://192.168.18.22:8088/img/dk.jpeg","http://192.168.18.22:8088/img/dk.jpeg"], //补丁路径 |
| 6 | patch_module | INT(4) UNSIGNED | 适合的模块(由app确定) | "patch_module":1, |
| 7 | patch_specific_user | TEXT | 补丁包指定应用的用户(如果些项填了，patch_user_identity不管) | [13510245457,18925213564], |
| 8 | patch_user_identity | BIGINT(24) UNSIGNED | 补丁包应用对像(第一位：是否实名认证 第二位: 是否借款人 第三位: 是否钱小保 第四位：是否钱大保 第五位：是否加V钱大保 第六位：是否微担保公司) | "patch_user_identity":1 |
| 9 | patch_begin | DATETIME | 补丁生效时间 | "patch_begins":"2016-08-01 08:00:00 |
| 10 | patch_expires | DATETIME | 补丁失效时间 | "patch_expires":"2016-09-01 08:00:0 |
| 11 | update_time | DATETIME | 更新时间，默认值为'1970-01-01 08:00:00' |  |


## 用户通讯录表 db_im_contacts


### 用户通讯录发送失败数据表 tb_user_contacts_failed

1.tb_user_contacts_failed分表策略
所包含数据库表及分表策略，如下

| 序号 | 表名 | 表信息描述 | 分表策略 |
| --- | --- | --- | --- |
| 1 | tb_user_contacts_failed | 敏感词表 | 不分表 |

注：xx表示数字
2. 表结构字段详细说明
对举报详细表的字段描述见如下表：

| 序号 | 字段 | 类型 | 描述 | 是否主键 |
| --- | --- | --- | --- | --- |
| 1 | imid | INT(11) UNSIGNED | 用户UID ，非空字段，默认值为0 | 主键 |
| 2 | guid | VARCHAR（40) | 业务层编号 |  |
| 3 | json_body | BLOB | 属性的JSON体，根据msgtype进行判断。存放经、纬度，图片大小，语音时长，消息URL,消息体，红包信息等；编码格式为UTF-8 |  |
| 4 | record_time | DATETIME | 该条记录的生成时间;默认值为'1970-01-01 08:00:00' |  |


# 遗留问题

（本节描述留待下一阶段解决的遗留问题（如果有的话）。）

| 编号 | 问题描述 | 解决建议 |
| --- | --- | --- |


# 其它
