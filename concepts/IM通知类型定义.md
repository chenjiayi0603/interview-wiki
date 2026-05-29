# IM通知类型定义

> 本文档定义雷漫分布式IM系统中的通知类型及其消息格式。

## 通知类型枚举

| 值 | 备注 |
|----|------|
| 100 | JAVA推送过来的消息，具体格式由JAVA决定 |
| 101 | 好友申请通知 msg :{"friendInfo": Object, "text":"xxxx"} |
| 102 | 接受好友申请通知 msg:{"friendInfo": Object, "text":"xxxx"} |
| 103 | 拒绝好友申请通知 msg:{"friendInfo": Object} |
| 104 | 用户好友信息变更 :(目前是双向删好友和修改好友设置) |
| 105 | 删除好友通知 msg：{"friendId": 123123423442423, "operatorId":123123423442443} |
| 111 | 会话修改推送 |
| 120 | 端对端秘钥推送 |
| 130 | 表情/表情包增加、修改推送 |
| 131 | 表情/表情包删除推送 |
| 140 | 消息广播推送文本 |
| 141 | 新设备登录通知 |
| 142 | 验证码通知 |
| 200 | 设备登出 |
| 201 | 用户更新 |

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

## 详细消息格式

### 104 用户好友信息变更

```json
{
    "friendInfo": {
        "id": 2222222,
        "userId": 22222222,
        "friendId": 25245674567,
        "friendNickname": 234523452345,
        "friendHeadLogoUrl": "1234123412",
        "memo": "xxx",
        "identityKeyPub": "xvvvvv",
        "signedKeyPub": "ccccc",
        "caVersion": "ggggg",
        "origin": 1,
        "star": 0,
        "close": 1,
        "shield": 9,
        "black": 8,
        "type": 1,
        "msg": "22443",
        "status": 4,
        "createTime": 234234,
        "updateTime": 23452345234,
        "addUpdateTime": 2342342,
        "blackUpdateTime": 2345234523
    }
}
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

### 111 会话修改推送

```json
{"conversation": {
    "id": 1247444552229015555,
    "userId": 1244946294919344129,
    "dstType": 2,
    "dstId": 1247444527642017794,
    "title": "闲聊扯淡群",
    "logoImg": null,
    "remark": null,
    "ext": null,
    "top": 0,
    "notify": 1,
    "status": 1,
    "maxMsgId": 0,
    "minMsgId": 0,
    "createTime": 1586248932307,
    "updateTime": 1586248932307
}}
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

### 120 端对端秘钥推送

```json
{"shareKey": {
    "userId": 1244946294919344129,
    "dstId": 1234746751569805314,
    "version": 3,
    "chainKey": "adsfsfsafafadsfasfadf",
    "macKey": "adfasffasdfas",
    "einitiatorKey": "asdfsdfsaff",
    "identityKey": "pub00001",
    "seq": 1585653301,
    "keyVersion": 1263756011766964226,
    "createTime": 1590137888053
}}
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

### 130 表情/表情包增加、修改推送

```json
{
   "emoticons":[{
       "emoticonsId":1244946294919344129,
       "title":"测试表情包",
       "description":"测试表情包描述",
       "url":"http://www.baidu.com",
       "expressionCount":200,
       "imageType":"JPG",
       "status":0,
       "orderIndex":1596453909589001
   }],
    "expressions":[{
       "id":1244946294919344129,
       "emoticonsId":1244946294919344129,
       "url":"http://www.baidu.com",
       "imageType":"JPG",
       "status":0,
       "orderIndex":1596453909589001
   }]
}
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

### 131 表情/表情包删除推送

```json
{
   "emoticons":[{
       "emoticonsId":1244946294919344129
   }],
    "expressions":[{
       "id":1244946294919344129
   }]
}
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

### 140 消息广播推送文本

```json
msg:{ "content":"xxxx","parseType":"1"}
```

parseType为内容类型，枚举值：1: 文本，2: 图片，3:音频，4:视频，5:文件

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

### 141 新设备登录通知

```json
msg:{ "text":"xxxx"}
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

### 142 验证码通知

```json
msg:{ "text":"xxxx"}
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

### 200 设备登出

```json
{"userId": Object, "opType": Object, "tips": Object, "devids": Object[]}
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

### 201 用户更新

```json
{"userInfo": Data}
```

Data里返回全部数据，如下：
```json
{
    "gender": Object,
    "usernameIsChange": Object,
    "addByQrcode": Object,
    "mobile": Object,
    "autograph": Object,
    "updateTime": Object,
    "headLogoUrl": Object,
    "mobileAreaCode": Object,
    "addNeedCheck": Object,
    "headLogoTime": Object,
    "areaCode": Object,
    "addByCard": Object,
    "searchByUsername": Object,
    "nickname": Object,
    "id": Object,
    "searchByMobile": Object,
    "username": Object,
    "addByGroup": Object
}
```

[src: raw/ingested/3项目/分布式IM-雷漫/数据-消息定义-五、通知类型定义.md]

## 相关页面

- [[IM消息协议定义]]
- [[IM数据库与缓存设计]]
- [[IM_MongoDB表设计]]
- [[分布式IM消息系统-雷漫]]