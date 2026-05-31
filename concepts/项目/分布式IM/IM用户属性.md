# IM用户属性

## 概述

IM用户属性数据结构存储IM用户基础信息。常用数据项和数据库列名保持一致。

## 数据结构

- **数据类型**：hash
- **Key**：`1:1:imuid`，其中第一位1表示hash结构，第二位1表示IM用户属性

## Field 列表

| Field | 说明 |
|-------|------|
| imid | IM用户ID |
| guid | 业务用户ID |
| nickname | 用户昵称 |
| avatar_url | 用户头像 |
| introduction | 用户简介 |
| sex | 性别 |
| location | 用户所在地 |
| last_login_time | 最后一次上线时间 |
| last_logout_time | 最后一次下线时间 |
| login_count | 登录次数 |
| online_time | 总在线时长 |
| beblack_count | 被拉黑次数 |
| niiwoo_talk_count | 发表你我圈说说数量 |
| bereport_count | 被举报次数 |
| single_chat_count | 发送单聊次数（消息数） |
| group_chat_count | 发送群聊次数（消息数） |
| singlechat_recv_count | 接收单聊次数（消息数） |
| groupchat_recv_count | 接收群聊次数（消息数） |
| member_level | 会员等级 |
| support_count | 点赞数量 |
| besupport_count | 被点赞数量 |
| comment_count | 评论数量 |
| becomment_count | 被评论数量 |
| group_number | 加入群数量 |
| project_number | 业务群数量(参与过的项目数) |
| friends_number | 好友数量 |
| fans_number | 用户拥有粉丝数量 |
| attention_number | 关注了的总人数 |
| addressbookfriend_number | 通讯录好友总人数 |
| user_type | 用户类型 |
| user_identity | 用户在你我金融的身份 |
| company | 公司 |
| occupation | 职业 |
| mobile | 手机号码[加密] |
| industry | 行业 |
| home_image | 你我圈背景图路径 |
| is_punished | 是否被惩罚 |
| password | 密码 |
| login_history | 用户登陆记录 |

[src: raw/ingested/3项目/社交IM-你我/niwo_im_redis数据结构设计-3.1.IM用户属性.md]