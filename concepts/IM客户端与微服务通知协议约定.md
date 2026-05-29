# 客户端与微服务通知协议约定

## 1. 发起群聊通知（cmd=4527）

- **msgType**: 401
- **msg**: `{"text": "", "data": Object}`
- **Object 字段**:

| field | value |
|-------|-------|
| id | 群id |
| groupName | 群名称 |
| logoUrl | 群logo |
| notice | 公告 |
| groupAuth | 加群是否认证：1=是，0=否 |
| state | 状态：1=正常，0=解散 |
| createTime | 创建时间 |
| updateTime | 更新时间 |

## 2. 修改群信息通知（cmd=4527）

- **msgType**: 301
- **msg**: `{"text": "", "data": Object}`
- **Object 字段**:

| field | value |
|-------|-------|
| id | 群id |
| groupName | 群名称 |
| logoUrl | 群logo |

## 3. 新成员进群通知（cmd=4527）

- **msgType**: 403
- **msg**: `{"text": "", "data": Object}`
- **Object 字段**:

| field | value |
|-------|-------|
| groupId | 群id |
| members[ ] | 新增成员列表 |

**members 字段**:

| field | value |
|-------|-------|
| groupId | 群id |
| userId | 进群成员userId |
| memberRole | 群成员角色：1=群主，2=管理员，3=普通成员 |
| memberNickname | 群成员昵称 |
| memberHeadLogo | 群成员头像 |
| inviteUserId | 邀请人userId |
| state | 状态：1=已加入，0=已退出 |
| createTime | 创建时间戳 |
| updateTime | 更新时间戳 |

## 4. 增减管理员通知（cmd=4531）

- **msgType**: 302
- **msg**: `{"data": Object}`
- **Object 字段**:

| field | value |
|-------|-------|
| groupId | 群id |
| addIds[ ] | 新增管理员userId列表 |
| removeIds[ ] | 移除管理员userId列表 |

## 5. 移除群成员通知（cmd=4527）

- **msgType**: 405
- **msg**: `{"text": "", "data": Object}`
- **Object 字段**:

| field | value |
|-------|-------|
| groupId | 群id |
| removeMembers[ ] | 移除群成员信息列表 |

**removeMembers 字段**:

| field | value |
|-------|-------|
| nickName | 被移除者的昵称 |
| removeId | 移除成员userId |

## 6. 群成员退出通知（cmd=4527）

- **msgType**: 406
- **msg**: `{"text": "", "data": Object}`
- **Object 字段**:

| field | value |
|-------|-------|
| groupId | 群id |
| quitId | 退出成员userId |
| nickName | 退群者昵称 |

## 7. 群解散通知（cmd=4527）

- **msgType**: 407
- **msg**: `{"text": "", "data": Object }`
- **Object 字段**:

| field | value |
|-------|-------|
| groupId | 群id |

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-客户端与微服务通知协议约定.md]