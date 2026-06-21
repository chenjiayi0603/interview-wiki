
# 第一阶段设计文档

目标：完成核心聊天功能，不加密；两个客户端（安卓，ios）
服务端完成核心架构模块和功能，做一定量性能压测。

# 数据库表/缓存设计


## Cache key 值约定

1) 第一段放置项目名或缩写 如 im
  2) 第二段把表名转换为key前缀 如, user:
  3) 第三段放置用于区分区key的字段,对应mysql中的主键的列名,如userid
  4) 第四段放置主键值

# Mysql数据库设计


## 用户表:tb_user

索引：
idx_userid: user_id 唯一索引
idx_username: username 唯一索引
idx_mobile: mobile 唯一索引
分库表字段：user_id 
缓存：先更新数据库，再缓存设置失效；
所属库描述：im_user
分库格式：im_user_x

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键ID |
| username | varchar(32) | 用户名 |
| password | varchar(32) | 密码（MD5） |
| gender | tinyint(4) | 性别：1=男，2=女，3=其他 |
| mobile_area_code | varchar(11) | 手机区号 |
| mobile | varchar(11) | 手机号 |
| area_code | varchar(11) | 地区号 |
| nickname | varchar(30) | 昵称（15字符） |
| autograph | varchar(64) | 签名30汉字 |
| head_logo_url | varchar(255) | 头像图片地址 |
| head_logo_time | int(11) | 头像图片更新时间 |
| username_is_change | tinyint(4) | 用户名是否修改过：0未修改，1修改过 |
| search_by_mobile | tinyint(1) | 是否可以通过手机号搜索： 1=是，0=否 |
| search_by_username | tinyint(1) | 是否可以通过雷迅号搜索： 1=是，0=否 |
| state | tinyint(4) | 状态 1.正常 0.无效 |
| create_time | bigint(13) | 创建时间 |
| update_time | bigint(13) | 更新时间 |


## 用户功能设置表:tb_user_settings

索引：idx_userid: user_id 唯一索引
分库表字段：user_id 
所属库描述：im_user
分库格式：im_user_x

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | id |
| user_id | bigint(20) | 用户id |
| device_id | varchar(64) | 设备id |
| setting_key | tinyint (3) | 键 |
| setting_value | tinyint (3) | 值 |
| create_time | bigint(13) | 创建时间 |
| update_time | bigint(13) | 更新时间 |


## 用户意见反馈表:tb_feedback

分库表字段：user_id 
所属库描述：im_customer
分库格式：im_customer_x

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | int(11) | 主键id |
| user_id | bigint(20) | 用户id |
| content | varchar(255) | 反馈内容 |
| images | json | 图片链接组合（json格式存储） |
| create_time | int(11) | 创建时间 |


## 用户投诉表:tb_complaint

分库表字段：user_id 
所属库描述：im_customer
分库格式：im_customer_x

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | int(11) | 主键id |
| user_id | bigint(20) | 用户id |
| target_id | bigint(20) | 投诉目标用户id |
| type | tinyint(3) | 1=发布色情骚扰信息，2=发布暴力恐怖信息，3=发布政治敏感信息，4=发布其他违法信息，5=发布仿冒品信息 |
| msg | varchar(255) | 留言备注 |
| create_time | int(11) | 创建时间 |


## 版本更新表:tb_version


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | int(11) | 主键id |
| ver_number | varchar(32) | 版本号 |
| ver_minimum | varchar(32) | 最低兼容版本 |
| ver_info | varchar(255) | 版本相关信息 |
| ver_size | double(10, 2) | 版本大小(单位：MB) |
| update_type | tinyint(1) | 更新类型：1=普通更新，2=热更新 |
| update_url | varchar(255) | 更新链接 |
| platform_type | tinyint(1) | 平台类型：1=IOS，2=Android |
| create_time | int(11) | 创建时间 |


## 登录认证表:tb_auth_token

索引：idx_userid_type: user_id,terminal_type 唯一索引，联合索引
分库表字段：user_id  表暂时不用
缓存：
缓存key -> im:tb_auth_token:user_id:100001:terminal_type:0:device_id:xxxxxx:auth_token  value

| 业务 | 数据库表 | 字段名 | 主键值 | 列名 |
| --- | --- | --- | --- | --- |
| im | tb_auth_token | user_id | 100001 |  |

表说明：

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | int(11) | 主键id |
| app_type | tinyint (4) | 设备类型：1安卓，2ios ，3 mac 4 pc |
| auth_token | varchar(128) | 认证token |
| create_time | int(11) | 创建时间 |
| status | tinyint(4) | 状态 1 正常 2 失效 |
| update_time | int(11) | 更新时间 |
| user_id | bigint(20) | 用户id |
| terminal_type | tinyint(4) | 终端类型：0移动，1桌面 |
| device_id | varchar(255) | 设备id |


## 设备登录记录表：tb_device_log

索引：idx_userid_deviceid: user_id, device_id 唯一索引
分库表字段：user_id 数据插入异步入库
缓存：

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | int(11) | 主键id |
| create_time | Int(11) | 创建时间 |
| status | tinyint(4) | 状态 0 正常 -1 失效 |
| update_time | Int(11) | 更新时间 |
| device_type | Varchar(64) | 设备类型 |
| device_name | Varchar(64) | 设备名字 |
| last_login_time | Int(11) | 最后登录时间 |
| device_id | Varchar(40) | 设备ID |
| user_id | bigint(20) | 用户id |


## 全局系统默认会话表：tb_default_conversation

索引：idx_dstid: dst_id 索引 
分库表字段：
缓存：本地缓存，定时（5分钟）异步更新。

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 自增ID |
| dsttype | int | 会话类别：1 单人会话，2群，3系统功能 |
| dstid | varchar(128) |  |
| nickname | varchar(255) | 昵称 |
| intro | varchar(255) | 介绍 |
| create_time | int |  |
| update_time | int | 更新时间 |
| logo_img | varchar(128) | 头像 url |
| logo_img_time | int | 更新时戳 |


## 全局群组限制人数表：tb_group_conf

分库表字段：

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 自增ID |
| group_number_limit | int | 群组人数限制 |
| group_type | tinyint(3) | 群类型 |
| state | varchar(255) | 状态 |
| create_time | varchar(255) | 创建时间 |
| update_time | int | 更新时间 |


## 聊天会话表：tb_conversation

唯一键：user_id, dst_id
分库id：user_id

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键 |
| user_id | bigint(20) | 用户id(索引) |
| dst_type | tinyint(3) | 会话类别：1 =单聊，2=群聊，3=系统 |
| dst_id | bigint(20) | 会话对象id |
| title | varchar(64) | 会话名（标题） |
| logo_img | varchar(128) | 会话Logo |
| remark | varchar(64) | 会话备注 |
| draft | varchar(1000) | 草稿 |
| ext | varchar(128) | 扩展 |
| top | tinyint(1) | 是否置顶：1=是，0=否 |
| notify | tinyint(1) | 是否开启通知：1=是，0=否 |
| status | tinyint(3) | 状态：1=正常，0=删除 |
| max_msg_id | bigint(20) | 最大消息id |
| min_msg_id | bigint(20) | 起始消息id |
| draft_time |  | 草稿更新时间 |
| top_time | bigint(13) | 置顶时间 |
| create_time | bigint(13) | 创建时间 |
| update_time | bigint(13) | 更新时间 |


## 全局默认权限表:tb_rights_profile

索引：idx_userid_type: user_id,terminal_type 唯一索引
分库表字段：
缓存：本地加载缓存，定时刷新（5分钟）

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | int(11) | 主键id |
| rights_key | int(11) | 默认设置key对应项目类型 当rights_type=1 1. 没有打开--聊天通知消息 2. 没有打开--zf消息 3. 没有打开-语音/视频通话通知 4. 没有打开-显示文字 5. 没有打开-夜间模式 6. 打开-警示声 7. 打开-警示振动 8. 好友-朋友验证 9. 名片--好友可见 10. 名片--陌生人可见 11. 陌生人聊天 12. 黑名单 13. 自动下载-图片 14. 自动下载-视频 15. 自动下载-语音 16. 自动下载-文件 17. 媒体文件-自动保存媒体文件 18. 朋友圈查看范围 19. 朋友圈更新提醒 20. 会话广告通知 21. 发现广告通知 22. 朋友圈广告通知 rights_type=2  6.静音 7.聊天页面显示群昵称 8.置顶 rights_type=3 2：黑名单 3：静音 4：收藏 5：备注 6：置顶 7：阅读状态 8：在线状态 9：名片 |
| rights_type | int(11) | 默认设置类型1.用户2.群组3.好友 |
| status | tinyint(4) | 状态 |
| value | int(11) | 权限设置具体值，除自动下载 0 关闭 1 开启 自动下载： 0  never 1  wifi 2  wifi &cellular 朋友圈查看范围： 0  last 5 days 1  last 6 month 2  all |
| create_time | timestamp | 创建时间 |
| update_time | timestamp | 更新时间 |


## 用户设置表:tb_user_authority

索引：idx_userid: user_id 索引
分库表字段：user_id
缓存：先更新数据库，再缓存设置失效；

| 字段名 | 类型 | 长度 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| user_id | bigint(20) | 用户id |
| authority_type | smallint(5) | 设置类型 |
| authority_value | tinyint(3) | 设置的值 |
| create_time | int(11) | 创建时间 |
| update_time | int(11) | 更新时间 |


## 系统推送令牌表:tb_push_token

索引：idx_userid: user_id 唯一索引
分库表字段：user_id
缓存：先更新数据库，再缓存设置失效
缓存key -> im:tb_push_token:user_id:100001:token  value
设备token：只能保持一份；更新或者增加token时要把原来的token对应的用户清除，同时清除老userid对应的token记录。

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| create_time | int(11) | 创建时间 |
| token | varchar(128) | 设备token |
| type | tinyint(4) | 设备类型 1.ios 2.android |
| user_id | bigint(20) | 用户id |
| environment | varchar(255) | ios 使用，证书类型 |


## 二级密码：tb_passwd2


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| user_id | bigint(20) | 时间 |
| passwd2 | varchar(32) | 二级密码 |
| tips | varchar(32) | 提示 |
| email | varchar(64) | 恢复邮箱 |
| state | tinyint(4) | 0无效，1有效 |


## 二维码表：tb_user_qrcode

索引：idx_userid: user_id 唯一索引
分库表字段：user_id
缓存：先更新数据库，再缓存设置失效；

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| user_id | uInt64 | 8 | PK |
| qrcode | Varchar | 255 | 二维码 |
| state | Int8 | 1 | 默认 0正常 |
| create_time | Int32 |  | 创建时间 |
| update_time | Int32 |  | 最后更新时间 |


## 名片表:tb_user_contact:名片

索引：idx_userid: user_id 索引
分库表字段：user_id
缓存：先更新数据库，再缓存设置失效；

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| address | varchar(255) | 地址 |
| email | varchar(255) | 邮箱 |
| first_name | varchar(255) |  |
| last_name | varchar(255) |  |
| mobile | varchar(16) | 手机号 |
| organization | varchar(255) | 组织 |
| qr_code | varchar(255) | 二维码 |
| title | varchar(255) | 标题 |
| update_time | int(11) | 更新时间 |
| create_time | int(11) | 创建时间 |
| user_id | bigint(20) | 用户id |


## 好友信息表:tb_friend_info

索引：
idx_userid_friendid: user_id,friend_id 唯一索引 
idx_updatetime:update_time
idx_addupdatetime:add_update_time
分库表字段：user_id 
所属库描述：im_friend
分库格式：im_friend_x

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| user_id | bigint(20) | 用户id |
| friend_id | bigint(20) | 好友id |
| friend_nickname | varchar(32) | 好友昵称（冗余字段） |
| friend_head_logo_url | varchar(255) | 好友头像（冗余字段） |
| memo | varchar(255) | 备注名 |
| origin | smallint(5) | 来源：201=通过手机号添加，202=通过雷讯号添加，203=通过群聊添加，204=通过二维码添加，205=通过名片添加 |
| star | tinyint(1) | 星标朋友：1=是，0=否 |
| close | tinyint(1) | 不让他看：1=是，0=否 |
| shield | tinyint(1) | 不看他：1=是，0=否 |
| black | tinyint(1) | 黑名单：1=是，0=否 |
| type | tinyint(1) | 类型：1=申请方，2=接受方 |
| msg | varchar(255) | 申请信息 |
| status | tinyint(1) | 状态：0=初始值，1=等待验证，2=已添加，3=已拒绝，4=已过期，5=已删除 |
| create_time | bigint(13) | 创建时间 |
| update_time | bigint(13) | 更新时间 |
| add_update_time | bigint(13) | 好友申请记录更新时间 |


## 表:tb_user_relationship_set:用户关系设置


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| create_time | int(11) | 创建时间 |
| relationship_id | bigint(20) | 关系的用户id |
| relationship_key | int(11) | 关系key |
| status | tinyint(4) | 状态 |
| update_time | int(11) | 更新时间 |
| user_id | bigint(20) | 用户id |
| value | varchar(255) | 值 |


## 通讯录表:tb_tel_book:

索引：idx_userid_deviceid: user_id 
分库表字段：user_id 
所属库描述：im_contact
分库格式：im_contact_x
该表数据量很大，特别注意。

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| user_id | bigint(20) | 用户id |
| device_id | varchar(64) | 设备id |
| contact_name | varchar(255) | 联系人名称 |
| mobile_area_code | varchar(11) | 手机区号 |
| mobile | varchar(11) | 手机号 |
| create_time | int(11) | 创建时间 |
| update_time | int(11) | 更新时间 |


## 文件操作记录表:tb_file_log

索引：idx_userid: user_id 
分库表字段：user_id 
所属库描述：im_file
分库格式：im_file_x
该表数据量很大，特别注意。

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| user_id | bigint(20) | 用户id |
| operation_type | varchar(64) | 设备id |
| file_type | varchar(255) | 文件类型:1=文件，2=图片，3=语音，4=视频 |
| file_path | varchar(11) | 文件路径 |
| create_time | int(11) | 创建时间 |


## 群组文件记录表:tb_file_group

索引：idx_groupid: group_id 
分库表字段：group_id 
所属库描述：im_file
分库格式：im_file_x
该表数据量很大，特别注意。

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| user_id | bigint(20) | 用户id |
| group_id | varchar(64) | 群组id |
| file_type | varchar(255) | 文件类型:1=文件，2=图片，3=语音，4=视频 |
| duration | varchar(11) | 文件时长 |
| upload_time | bigint(13) | 上传时间 |
| thumbnail_path | varchar(255) | 预览图 |
| file_path | varchar(255) | 文件路径 |
| client_msg_id | bigint(20) | 客户端消息id |
| member_nickname | varchar(255) | 群成员昵称 |
| state | tinyint(3) | 状态：1=正常，0=异常 |
| create_time | bigint(20) | 创建时间 |
| update_time | bigint(20) | 更新时间 |


## 增加好友历史表:tb_history_friend

索引：user_id 索引
分库分表：user_Id
缓存：

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| create_time | int(11) | 创建时间 |
| friend_id | bigint(20) | 好友id |
| state | tinyint(4) | 状态 |
| update_time | int(11) | 更新时间 |
| user_id | bigint(20) | 用户id |


## 表:tb_official_user:官方账号


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| introduction | varchar(255) | 介绍 |
| account_body | varchar(255) | 账户体 |
| create_time | int(11) | 创建时间 |
| image | int(11) | 图片logo |
| nickname | varchar(255) | 昵称 |
| state | tinyint(4) | 状态 |
| type | int(11) | 类型 1.Team 2.钱包 |
| update_time | int(11) | 更新时间 |
| about | varchar(255) | 关于 |
| website | varchar(255) | 网站 |


## 群组:tb_group_info:群组

索引：group_id 索引唯一
分库分表：group_id
缓存：

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| group_name | varchar(255) | 群名 |
| logo_url | varchar(255) | 群logo图片地址 |
| notice | varchar(255) | 公告 |
| group_auth | tinyint(3) | 加群是否认证：1=是，0=否 |
| state | tinyint(3) | 状态 |
| create_time | bigint(13) | 创建时间 |
| update_time | bigint(13) | 更新时间 |


## 表:tb_group_member_set:群组成员设置


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| create_time | int(11) | 创建时间 |
| group_id | bigint(20) | 群组id |
| state | tinyint(4) | 状态 |
| type | smallint(6) | 设置类型 |
| update_time | int(11) | 更新时间 |
| user_id | bigint(20) | 用户id |
| value | int(6) | 设置的值 |


## 表:tb_group_member_info:群组成员

组合索引 ：`group_id`, `user_id`

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 主键id |
| group_id | bigint(20) | 群组id（索引） |
| user_id | bigint(20) | 用户id |
| member_role | tinyint(3) | 群成员类型：1=群主，2=管理员，3=普通成员 |
| member_nickname | varchar(255) | 群成员昵称 |
| member_head_logo | varchar(255) | 群成员头像 |
| invite_type | varchar(255) | 邀请方式-内容 |
| invite_user_id | bigint(20) | 邀请的人 |
| state | tinyint(3) | 状态 |
| create_time | bigint(13) | 创建时间 |
| update_time | bigint(13) | 更新时间 |

**7.公共使用的表 **

## 表:tb_language:语言


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | int(11) | 主键id |
| language_name | varchar(255) | 语言昵称 |
| create_time | int(11) | 创建时间 |
| status | tinyint(4) | 状态 |
| update_time | int(11) | 更新时间 |


## 表:tb_language_tips:语言文案


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | int(11) | 主键id |
| language_id | int(11) | 语言id1.中文2英文 |
| code | int(11) | 文案code |
| create_time | int(11) | 创建时间 |
| status | tinyint(4) | 状态 |
| tips | varchar(255) | 文案提示语 |
| update_time | int(11) | 更新时间 |

**9.****IM消息服务****单独使用 **

## 表:tb_no_read_sum:未读消息数量


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) unsigned | 自增id |
| user_id | bigint(20) unsigned | 用户id |
| user_type | int(11) | 用户类型1--IOS 2--ANDROID  3 mac 4--pc |
| no_read_sum | int(11) | 未读消息条数 |
| update_time | int(11) | 更新时间 |
| create_time | int(11) | 创建时间 |


## 表:tb_add_friend_info:添加好友

分库字段：to_id

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint(20) | 自增ID |
| create_time | int(11) | 创建时间 |
| end_time | int(11) | 结束时间 |
| from_id | bigint(20) | 发送ID |
| msg | varchar(255) | 内容 |
| msg_id | bigint(20) | 消息ID |
| status | smallint(6) | 状态 0--初始值 1--已通过 2--已拒绝3--过期 |
| to_id | bigint(20) | 接受ID |
| update_time | int(11) | 更新时间 |


## 表：tb_ca_info ca公钥信息表

索引：
idx_userid_version: user_id,version 唯一索引
idx_userid:user_id
分库表字段：user_id 
所属库描述：im_ca
分库格式：im_ca_x

| 字段名 | 类型 | 说明 |
| --- | --- | --- |
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


# Mongo表设计

群消息
单聊消息

## 1、公用信息

数据库 immsg
表名：msg_c2c
公用字段信息

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| fromId | uInt64 |  | 发送用户id |
| toId | uInt64 |  | 接受用户id |
| type | int |  | 参考离线消息类型定义 |
| msgId | Int64 |  |  |
| serverTime | Uint8 |  |  |
| status | Int8 |  | 默认0 |
| msgJson | String |  |  |


| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| userId | uInt64 |  | 等于fromId/toId， 用户的Id索引 |
| fromId | uInt64 |  | 发送用户id |
| dstId | uInt64 |  | 接受用户id |
| msgId | Int64 |  | 服务器生成的唯一整数，一条消息两份，用同一个Id |
| cmdType | Int32 |  | 消息命令字 |
| msgtype | Int32 |  | 参考客户端消具体消息内容类型 |
| content | string | 3000 | 具体的消息内容字符串结构 |
| msgPb | bytes |  | 整个pb序列化 |
| nodeStartTime | Int32 |  | 节点启动时间（单位秒） |
| nodeId | string |  | 节点值 |
| msgseq | Int32 |  | 消息序列号 |
| clientmsgid | Int64 |  | 客户端的消息Id，目前作用只在于消息重发时检查重发消息是否入库 |
| mobileStatus | Int32 |  | 手机端拉取消息与否 |
| pcStatus | Int32 |  | PC端拉取消息与否 |
| reserve | Int32 |  | 保留 |
| updateTime | Int64 |  | 消息文档更新时间 (单位微秒) |
| createTime | Int64 |  | 消息文档插入时间 (单位微秒) |


## 2、具体消息定义


### 2.1点对点消息 2份，A->B  msg_c2c

作用：点对点消息

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| userId | uInt64 |  | 等于fromId/toId， 索引（分片键） |
| fromId | uInt64 |  | 发送用户id |
| dstId | uInt64 |  | 接受用户id |
| dstType | int |  | 目的类型 1 用户，2群 ，3系统 |
| msgId | Int64 |  |  |
| sendTime | Uint64 |  | 消息产生时间（1970—微秒） 服务端收到的时间 |
| status | Int8 |  | 默认0，保留 |


| fromNickName | Varchar |  |  |
| --- | --- | --- | --- |
| fromHeadImg | Varchar |  | 头像时戳 |
| cmdType | int |  | 参考离线消息类型定义 |
| isTransmit | int |  | 0:默认 0x00000001:回复类型信息 0x00000002:转发类型信息，按位表示消息的类型 |
| msgType | Int32 |  | 消息类型 |
| content | String |  | msgType对应的具体的内容字符串 |
| msgPb | char |  | 整个pb序列化 |
| msgState | Int8 | 1 | 状态：0默认 1xxx  2送达3已读4撤回5删除自己6:清空会话内容（其中入库的状态消息分别是3、4、5）（删除好友也相当清空会话） |
| updateTime |  |  | 更新时间戳 |
| createTime |  |  | 创建入库时间 |
| mobileStatus | Int8 |  | 移动端状态 默认-1 ，0收到 |
| pcStatus | Int8 |  | 桌面端，同上 |
| reserve | Int8 |  | 保留 |
| nodeStartTime | Int32 |  | 节点启动时间 |
| nodeId | string |  | 节点id |
| msgseq | Int32 |  | 对该目的用户的消息流水，必须递增且连续 |
| version | int64 | 8 | 端到端密钥版本号,（也代表当前消息的版本）加密消息发送都应该带上，客户端根据消息版本号采用对应版本密钥解密 |
| platformType | Int32 | 4 | 消息发起者的所用设备平台类型客户登录客户端类型 0:未定义1:安卓 2:IOS 3:MAC 5:Windows; 聊天消息才有，状态消息，通知不没有本字段 |


### 2.2点对点消息 2份，端对端加密密钥消息表 keys_c2c

作用：点对点消息

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| userId | Int64 |  | 用户id，索引（分片键） |
| dstId | uInt64 |  | 接受用户id |
| dstType | int |  | 目的类型 1 用户，2群 ，3系统 |


| msgType | Int32 |  | 消息类型（通知子类型） |
| --- | --- | --- | --- |
| content | String |  | 密钥消息 |
| msgPb | char |  | 整个pb序列化 |
| msgState | Int8 | 1 | 状态：0默认 1删除 |
| createTime |  |  | 创建入库时间 |
| keyVersion | Int64 |  | 密钥版本号（雪花Id） |
| reserve | Int8 |  | 保留 |


### 2.3通知 msg_notify

作用：保存通知消息

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| fromId | Int64 |  | 发送用户id |
| dstId | Int64 |  | 接受用户id， 索引（分片键） |
| msgId | Int64 |  | 服务端产生消息id唯一 |
| serverTime | Int64 |  | 消息产生时间（1970—微秒） 服务端收到的时间 |
| status | Int32 |  | 默认0，保留 |
| fromNickName | string |  | 发送者名称 |
| fromHeadImg | string |  | 发送者logo |
| msgType | Int32 |  | 参考离线消息类型定义 |
| msg | string |  | json消息体 |
| nodeStartTime | Int32 |  | 节点启动时间 |
| nodeId | string |  | 节点id |
| msgseq | Int32 |  | 对该目的用户的消息流水，必须递增且连续 |


### 2.4群通知 groupmsg_notify--保留，暂未使用

作用：保存群通知消息

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| fromId | Int64 |  | 发送用户id（填用户id） |
| dstId | Int64 |  | 接受id（填群id） |
| msgId | Int64 |  | 服务端产生消息id唯一 |
| serverTime | Int64 |  | 消息产生时间（1970—微秒） 服务端收到的时间 |
| status | Int32 |  | 默认0，保留 |
| fromNickName | string |  | 发送者名称 |
| fromHeadImg | string |  | 发送者logo |
| msgType | Int32 |  | 参考离线消息类型定义 |
| msg | string |  | json消息体 |
| nodeStartTime | Int32 |  | 目标启动时间（服务器填群对象在群服务的时间） |
| nodeId | string |  | 目标id（服务器填群id）,string类型 |
| msgseq | Int32 |  | 对该目的id的消息流水，必须递增且连续 |


### 2.5群消息 msg_c2g

作用：群消息

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| fromId | Int64 |  | 发送用户id |
| groupId | Int64 |  | 时间+机器id+seq，索引（分片键） |
| cmdType | int |  | 参考离线消息类型定义 |
| msgId | Int64 |  | 服务端产生消息id唯一 |
| status | Int8 |  | 默认0 |
| fromNickName | Varchar |  |  |
| fromHeadImg | Int32 |  |  |
| isTransmit | Int32 | 4 | 0:默认 0x00000001:回复类型信息 0x00000002:转发类型信息，按位表示消息的类型 |
| msgType | Int32 |  | 消息子类型 |
| content | string |  | msgType对应的具体的内容字符串 |
| msgPb | char |  | Pb二进制消息 |
| clientMsgId |  |  | 客户端产生消息ID |
| msgState | Int8 | 1 | 状态：0默认 3已读4撤回5删除自己 (状态消息上来可能会对原始消息的增加状态字段，但只有撤回（全局删除）才会触发这个操作，删除自己的状态只会在状态消息那条记录里) |
| updateTime | Int64 |  | 更新时间戳（1970—微秒） |
| createTime | Int64 |  | 创建入库时间（1970—微秒） |
| nodeStartTime | Int32 |  | 目标启动时间（这里用的是群内存对象创建时间） |
| nodeId | String |  | 目标id（这里用的是群id）,string类型 |
| msgseq | Int32 |  | 对该目的id（这里用的是群id）的消息流水，必须递增且连续 |
| isTrySend | Int32 | 4 | 0:否 1：重发 |
| notifyType | Int32 | 4 | 标识消息是否为@消息，0：否 1：是 |
| notifyUserId | array | 4 | Json数组，@的用户列表 |
| platformType | Int32 | 4 | 消息发起者的所用设备平台类型客户登录客户端类型 0:未定义1:安卓 2:IOS 3:MAC 5:Windows; 聊天消息才有，状态消息，通知不没有本字段 |
| opmsgId | Int64/Arrary |  | 对于引用类型消息,cmdType=4501, opmsgId为单个值；对于状态消息，cmdType=4519, opmsgId结构为数组 |

***客户端离线上来拿到的******”******未读数******”******：从个人已读位置开始计算***
1、【count(cmdType 为原始聊天/通知消息, fromid != self) – msgState==4的消息数】
2、“删除自己”的状态消息直接记录为一条状态消息，不参与未读数计算

### 2.6群成员消息明细 group_msg_details--目前不需要

作用：群成员消息状态

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| fromId | Int64 | 8 | 发送者id |
| dstId | Int64 | 8 | 接受者ID |
| groupId | Int64 | 8 | 群ID |
| msgId | Int64 | 8 | 消息ID 唯一，单调递增 |
| msgState | Int32 | 4 | 状态 ，默认0，3--已读 4--已撤 5--删除自己 |
| selfstate | int32 | 4 | 0未读， 1已读， 2 已删 |
| updateTime | Int32 | 4 | 更新时间 |
| createTime | Int32 | 4 | 创建时间 |
| serverTime |  |  | 消息生成时间 |


### 2.7会话状态 session_status

作用：保存会话的状态，，持久化msgseq，群的还包括最后一条消息ID

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| dataId | Int64 | 8 | 用户ID或者群ID， 索引（分片键） |
| type | Int32 |  | 类型  1:用户    2：群 |
| targetId | string |  | 会话id |
| lastMsgId | Int64 | 8 | 会话的最后消息ID |
| updateTime | Int64 | 8 | 会话最大消息ID更新时间 （微秒） |
| maxMsgseq | Int64 |  | 会话预分配的消息序列号段最大值 |
| lastNodeid | String |  | 配当前消息序列号段的结点id（用于每次判断结点是否变化） |
| nodeStarttime | Int32 |  | last_nodeid启用的时间，结点不重启不变化 |

备注：目前只存群会话最后一条消息的ID

### 2.8语音播放消息记录played_voice_msg


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| _id | Long | 主键 |
| userId | Long | 用户id， 索引（分片键） |
| dstType | Integer | 会话类型 |
| dstId | Long | 会话对象id |
| msgId | Long | 语音消息id |
| createTime | Long | 创建时间 |


### 2.9 设备最后登录信息表device_last_login_info


| 字段名 | 类型 | 说明 |
| --- | --- | --- |
| _id |  | 主键（mongo自生成） |
| userId | int64 | 用户id，索引（分片键） |
| devId | String | 设备id |
| loginInfo | Json | JSON打包数据     { "loginTime":1596187168(s)} |
| updateTime | int64 | 更新时间，如果只需要时间可以取这个字段 |


## 会话消息序号 session_msgseq

作用：保存会话的消息序号下一个分配起点

| 字段名 | 类型 | 长度 | 备注 |
| --- | --- | --- | --- |
| sessionId | string | 8 | 会话id，索引（分片键） |
| msgseqStart | Int64 |  | 消息序号起点 |
| updateTime | Int64 |  | 更新时间（微妙） |


## 消息广播official_broadcast

作用：消息

| 字段名 | 类型 | 备注 |
| --- | --- | --- |
| fromType | Int32 | 发送者类型：1用户id 2 群 3.系统 |
| fromId | Int64 | 发送 id（索引，分片键） |
| fromNickName | string | 发送者名称 |
| fromHeadImg | string | 发送者logo |
| msgId | Int64 | 服务端产生消息id唯一（这可以在后续看官方账号消息的量再补上索引） |
| cmdType | int | 参考离线消息类型定义 |
| msgType | Int32 | 消息类型 |
| language | string | 客户端语言版本，如en-US、zh-CN、default，匹配不上的，使用default的 |
| msgPb | char | 整个pb序列化 |
| sendTime | Uint64 | 消息产生时间（1970—微秒） 服务端收到的时间 |
| status | Int32 | 默认0，1 无效 |
| dstRangeType | Int32 | 接收用户范围 0 全网用户 （目前只有本类型） 1 安卓 2ios 3 pc |
| updateTime | Int64 | 更新时间戳 |
| createTime | Int64 | 创建入库时间 |
| nodeStartTime | Int32 | 节点启动时间 |
| nodeId | string | 节点id |
| msgseq | Int32 | 对该目的用户的消息流水，必须递增且连续 |

备注：看官方账号后续是否会扩展，有则官方账号fromId分片，否则用msgid

## 设备启动次数记录表user_startup_count

备注：微服务加上分片/索引
//"deviceId" : "00000000-1abc-c233-ffff-ffff86937bff",
//"deviceType" : 1,
//"phoneModel" : "generic_x86_64",
//"systemVersion" : "27",
//"createTime" : NumberLong(1592206170587),
//"updateTime" : NumberLong(1592206170587),
//"_class" : "com.rayman.imms.user.entity.StartupCount"
作用：设备启动次数上报

| 字段名 | 类型 | 备注 |
| --- | --- | --- |
| deviceId | string | 设备Id，索引 |
| deviceType |  |  |
| phoneModel |  |  |
| systemVersion |  |  |
| createTime |  |  |
| updateTime |  |  |
| _class |  |  |


# 消息类型定义 cmdtype


| 发送命令字 | 转发命令字 | 备注 |
| --- | --- | --- |
| 4001 | 4003 | 点对点消息 |
| 4005 | 4007 | 点对点消息状态 |
| 4517 | 4519 | 群消息状态 |
| … | 4037（个人相关）  4527（群相关） | 通知消息 |
| 4501 | 4503 | 群消息 |

发送命令字：客户端发送命令
转发命令字：转发出去，同时也是存库时的cmdType