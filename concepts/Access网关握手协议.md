# Access 网关握手协议

## 5.1 握手与登录的关系

**握手和登录是分开的两个步骤：**

- **握手**（如 `/hello/shake`）仅用于协议能力协商和通道初始化，不涉及正式的用户身份认证。客户端连接 Access 后，需先发送握手消息，建立基础传输能力（如是否支持压缩、心跳参数等）
- **登录** 则发生在握手之后，客户端还需主动发送登录消息（携带 token、deviceid 等），由 Access 网关联系 Login 服务进行正式鉴权和 session 建立。只有登录通过，才视为真正的"在线"

**流程：** WebSocket 通道建立 → 握手协商（/hello/shake）→ 业务登录（token）

## 5.2 握手协议流程

当 WebSocket 连接通过 Nginx 从 wss（加密）升级为 ws（未加密）后，Access 网关需要对每个新连接进行"握手协议"处理，核心流程包含：

1. **协议协商**：检查协议版本、协商压缩/心跳等扩展参数
2. **身份校验**：解析用户标识、检测 Token 有效性、校验参数
3. **会话初始化**：生成 sessionId/sessionKey，绑定到连接
4. **状态及限流**：记录客户端 IP 和节点，检查并发连接限流
5. **返回结果**：成功则返回会话参数，失败返回错误码

**作用：**

- 保证只有合法用户可连接
- 每条连接有独立密钥，数据安全
- 防滥用，利于状态与安全管理

**典型握手协议示意图：**

```
Client         Nginx         Access         Login
   |---WSS----->|---WS------->|
   |  WebSocket协议握手  |
   |--------------------->|
   |    握手成功         |
   |<---------------------|
   |（此时仅完成 WS 标准协议升级，无业务认证）|
   |
   |--业务登录消息（token, deviceid...）----->
   |--------------------->|
   |         认证请求       |---->|
   |                       |    | 校验+Token认证+安全协商（在Login服务完成）
   |                       |<---|
   |<-----业务认证响应------|
   |  （sessionKey, sessionId等业务数据）    |
```

## 5.3 /hello/shake 握手消息

> **注意**：这只是个握手消息，登录消息在后面

WebSocket 客户端连接后，需主动向 Access 网关发送 `/hello/shake` 握手消息，用于建立基础通道和能力协商，但不负责正式身份登录。

#### 请求字段（客户端 → Access 网关）

| 字段          | 类型      | 说明                            | 是否必填  |
| ------------- | --------- | ------------------------------- | --------- |
| protocolVer   | string    | 协议版本号                       | 是        |
| ext           | object    | 扩展字段（可选，如压缩、心跳等协商参数） | 否        |

**请求示例：**

```json
{
  "cmd": "/hello/shake",
  "protocolVer": "1.0.0",
  "ext": {
    "compression": "gzip"
  }
}
```

#### 应答（Access 网关 → 客户端）

WebSocket 握手响应很简单，仅表示握手成功：

**响应示例：**

```json
{
  "cmd": "/hello/shake/ack",
  "code": 0,
  "msg": "ok"
}
```

**说明：**

- 握手成功后，业务协议与 app 协议一致，在目录 `public\消息协议pb`
- `ModuleShake.so` 用来处理握手协议


#### websocket协议

##### WebSocket协议分析

WebSocket 是基于 RFC 6455 定义的全双工实时通信协议，服务端和客户端只需完成一次握手，即可在单一的 TCP 连接上进行双向数据通信，典型特性包括：

- **握手阶段**：基于 HTTP/1.1 协议完成初始握手，采用 `Upgrade: websocket` 头部。Nginx 负责 TLS 终止后，将流量转发给后端 Access 网关，后端直接处理升级后的明文 WebSocket 数据。
- **数据帧结构**：
  - 分为文本帧、二进制帧、Ping/Pong（心跳）、Close（关闭）。
  - WebSocket 帧结构：`FIN | RSV | opcode | mask | payload length | masking-key | payload data`

  // INSERT_YOUR_CODE
  // 字段详解如下：
  // - FIN（1 bit）：表示是否为消息的最后一帧，1 表示消息结束。
  // - RSV（3 bit）：保留字段，通常为 0，扩展用。
  // - opcode（4 bit）：操作码，定义该帧的类型（0x1 文本帧，0x2 二进制帧，0x8 关闭，0x9 ping，0xA pong 等）。
  // - mask（1 bit）：是否有掩码（客户端发服务端时必须为 1，服务端到客户端一般为 0）。
  // - payload length（7/7+16/7+64 bit）：负载长度，7 位不足时用扩展位存储真实长度。
  // - masking-key（32 bit/4 字节，可选）：掩码密钥。mask=1 时存在，用于还原数据。
  // - payload data：真实消息内容（已按 mask 规则解码后得到最终数据）。

  
  - 客户端发往服务端的数据一般采用掩码，服务端发往客户端的数据通常不加 mask。

-------
payload data 才是包含业务消息的内容？包括业务消息头、业务消息体？
// INSERT_YOUR_CODE
// 回答：是的，WebSocket 帧中的 payload data 字段才是实际承载业务消息内容的部分。
// payload data 内部结构完全由上层业务协议定义（与 WebSocket 协议无关）。
// 在 IM/消息系统中，payload data 一般包含：业务消息头（如自定义包头、消息长度、消息类型、鉴权信息等）
// 和业务消息体（即真正的消息内容）。例如：
// [payload data] = [业务自定义包头 + 业务消息体]
// - 包头：可能含有 version、enctype、cmdId、seq、bodylen 等自定义结构
// - 消息体：二进制 Protobuf 、JSON 文本等实际消息
// WebSocket 层仅保证可靠传输和帧结构拆包，包内内容完全透明地交给业务上层协定。
//
// 总结：
// - payload data 是 WebSocket 协议承载业务消息的载体
// - 业务消息头、业务消息体都封装在 payload data 内，由上层协定解析和还原

--------------
握手消息的 payload data 不是 http 协议？
// INSERT_YOUR_CODE
// 回答：不是，握手消息的 payload data 不是 http 协议。
// WebSocket 的“握手”过程本身是通过标准的 HTTP/1.1 请求+响应完成协议升级；但一旦握手（即 Upgrade: websocket 成功）完成后，WebSocket 传输的数据帧(payload data)就是纯自定义二进制或文本协议，完全和 HTTP 无关。
// 换言之：WebSocket 握手阶段用 HTTP，握手成功后，双方通过 WebSocket 帧结构持续传输消息，此时 payload data 部分完全由上层业务协议定义。
// 在 IM 场景，握手相关的 payload data （如 1001 登录/握手消息）就是自定义业务包体，不再是 HTTP 格式。
//
// 总结：
// - 握手前是 HTTP 协议，完成升级后 WebSocket payload data 就不再是 HTTP，而是自定义业务协议

-------------
握手前是http消息？
// INSERT_YOUR_CODE
// 回答：是的，WebSocket 握手前是标准的 HTTP 消息（HTTP/1.1 请求和响应）。
// WebSocket 连接建立以普通 HTTP 请求发起（包含特殊头部如 Upgrade: websocket、Connection: Upgrade 等），
// 服务端响应 101 Switching Protocols（协议切换），握手成功后 HTTP 通道升级为 WebSocket 通道。
// 握手阶段消息完全遵循 HTTP 格式，仅用于完成协议升级；升级完成后才会切入 WebSocket 的帧通信机制。
// 
// 总结：
// - 握手前：客户端发的是 HTTP 请求、服务端回的是 HTTP 响应（用于协商协议升级）。
// - 握手成功后：通信数据就是 WebSocket 的二进制帧格式，已经不是 HTTP 协议。

-------------------
握手前的http的格式是什么？
// INSERT_YOUR_CODE
// 回答：HTTP 的格式是基于文本的请求-响应协议，分为请求报文和响应报文两种结构。
// 1. HTTP 请求报文结构：
//    <请求行>（如：GET /path HTTP/1.1）
//    <头部字段1>: <值1>
//    <头部字段2>: <值2>
//    ...
//    空行
//    <可选的消息正文（body）>
//
//    示例：
//    GET /chat HTTP/1.1
//    Host: example.com
//    Upgrade: websocket
//    Connection: Upgrade
//    Sec-WebSocket-Key: xxx
//    （...其它头部）
//
//    <空行>
//    （无 body/或升级 WebSocket 不需要 body）
//
// 2. HTTP 响应报文结构：
//    <状态行>（如：HTTP/1.1 101 Switching Protocols）
//    <头部字段1>: <值1>
//    <头部字段2>: <值2>
//    ...
//    空行
//    <可选的消息正文（body）>
//
//    示例：
//    HTTP/1.1 101 Switching Protocols
//    Upgrade: websocket
//    Connection: Upgrade
//    Sec-WebSocket-Accept: yyy
//
//    <空行>
//    （无 body）
//
// 总结：HTTP 消息由起始行（请求行或状态行）、若干头部字段、空行和可选正文组成。内容为纯文本，每一行用 CRLF(\r\n) 结束。

分析下 http_parser_execute？是开源接口？
// INSERT_YOUR_CODE
// 回答：`http_parser_execute` 是著名 C 开源库 [nodejs/http-parser](https://github.com/nodejs/http-parser) 中对 HTTP 协议解析的核心接口，广泛用于 Nginx、Node.js 等网络框架。它不是标准库，而是第三方开源实现。该函数可高效解析 HTTP 请求或响应的报文，将字节流逐步喂入，按解析状态机调用回调，自动提取出请求行、头部、body 等关键信息。
// 
// 典型用途：
// - 高性能网络服务器（如 Node.js、libevent、Nginx 某些版本）用它在 TCP 层面剥离 HTTP 协议的各部分。
// - 设计上采用"增量解析"（streaming）：支持分段推送数据（如每收到一块数据都从当前位置继续解析），对高并发、慢速连接等更友好。
// 
// 简单用法示意：
//   http_parser parser;
//   http_parser_init(&parser, HTTP_REQUEST);
//   http_parser_execute(&parser, &settings, buf, buflen);
// 
// 其中 http_parser_execute 依据预先设置的 settings 回调，对解析到的 piece（如 on_url、on_header_field、on_body、on_message_complete）逐步触发业务逻辑。
// 
// 总结：`http_parser_execute` 不是标准库，而是 C 语言开源社区广泛采纳的高性能 HTTP 解析接口，来源于 nodejs/http-parser 项目，可直接集成到服务端网关等代码实现高效 HTTP 协议处理。


https://gitee.com/chenjiayi/thunder/blob/master/code/Util/src/util/http/http_parser.h
https://github.com/nodejs/http-parser/blob/main/http_parser.h


##### Access 网关中的 WebSocket 解码器

Access 网关作为服务端，需实现 WebSocket 协议解码（以内存高效、零拷贝为目标），处理步骤基本如下：

1. **接收缓冲**：从网络事件回调获得原始 TCP 字节流，放入缓冲区。
2. **WebSocket帧解析**：
    - 检查缓冲区中数据是否有完整帧：根据 WebSocket 协议，需要先读取 2 字节的头部（含FIN/opcode/mask/len），然后根据长度扩展字段决定总帧长，再判断 buffer 是否足够。
    - 若客户端有掩码（mask bit=1），需读取 4 字节 masking-key，解码 payload。
    - 支持分片帧、多帧聚合与粘包拆包处理。

3. **payload 还原与分发**：
    - 对于完整 payload，可进一步判断 opcode（如 0x1 文本帧，0x2 二进制帧）。
    - 业务上一般采用自定义的二进制协议或 JSON 文本，数据交由上层分包逻辑与业务处理（如路由到指定模块，如分派至 ModuleShake.so 处理握手）。

4. **异常容错**
    - 当解帧异常或者帧协议非法时，及时断开连接并记录异常日志。

#### 典型伪代码流程（WebSocket 解码）

```c++
while (buffer 可读数据 >= 2) {
    // 1. 解析头部
    byte1 = buffer[0]; byte2 = buffer[1];
    FIN = (byte1 & 0x80) != 0;    // 是否为最后一帧（WebSocket 支持消息分帧：若 FIN=1 表示这是消息的最后一帧。通常小型消息可只用一帧发送——即 FIN=1 即可，无需分帧。只有超大消息或需流式分发时才会拆成多帧，每帧 FIN=0，直到最后一帧 FIN=1 结束。分帧主要用于传输大数据或流式场景，普通业务消息可以/推荐只用一帧。）

    // 聊天消息能否要求客户端固定只发一帧？
    // 回答：绝大多数 IM 系统的聊天消息“都可以、也推荐让客户端固定只发一帧（即 FIN=1，完整业务消息一次性封装于单一 WebSocket 帧中）”，这样简化服务端拆包处理，提升效率。
    // WebSocket 协议支持分帧（FIN=0/1），理论可拆成多帧发送——但在常规聊天场景：
    // - IM消息体一般较小（通常小于 1KB/4KB/64KB），完全可以用单帧承载。
    // - 单帧模式便于服务端一次性解包、分发，降低复杂度和 bug 风险。
    // - 仅遇到超大文件/图片/音视频流等大业务数据时（如需边传边播）可采用多帧分发。
    // 实际工程中可在“协议层加约定”：普通聊天、通知、信令等业务消息**客户端必须单帧发送**（FIN=1），服务端仅处理单帧业务帧。只有在明确支持流式大数据（如断点续传、流媒体）的场景，才额外开放分帧。
    // 开发建议：
    // - 聊天类场景可要求“业务层每条消息固定一帧”，服务端收到 FIN=0 或多帧合并的消息时直接拒绝或关闭（提升健壮性）。
    // - 如需支持大消息/大文件可专用协议通道，或按块业务层序列传递。
    //
    // 总结：IM聊天消息完全可以、并推荐协议层规定客户端只发单帧（FIN=1），无需服务端额外处理拆帧粘帧，更高效安全。

    opcode = byte1 & 0x0F;
    MASK = (byte2 & 0x80) != 0;
    payload_len = byte2 & 0x7F;

    // 2. 长度扩展
    if (payload_len == 126) {
        读取后续 2 字节 --> 得到真实长度
    } else if (payload_len == 127) {
        读取后续 8 字节 --> 得到真实长度
    }

    // 3. 检查掩码
    if (MASK) {
        读取 4 字节 masking-key
    }

    // 4. 判断 buffer 是否有足够完整帧
    if (buffer.length < header + mask + payload_len) break;

    // 5. 解 mask
    if (MASK) {
        for (i = 0; i < payload_len; i++)
            payload[i] ^= masking_key[i % 4];
    }

    // 6. FIN位处理与多帧判断
    // 一般业务（如IM聊天）可要求客户端固定FIN=1（即单帧完整消息发送）
    // 若收到的帧 FIN=0，表示是分片消息，需要合并后才能处理payload
    if (!FIN) {
        // 非最后一帧，需缓存payload碎片，等待FIN=1帧拼接完成再交给业务层
        缓存当前payload片段，合并至待拼接区
        跳出本轮解包，等待更多数据
        // 若业务协议强制要求单帧（IM协议约定），可直接拒绝或断开
    } else {
        // FIN=1，已是完整业务消息（或最后一片），可处理
        // 若有缓存分片，需先拼接完整payload
        if (存在历史分片缓存) {
            拼接历史分片与本帧payload为完整消息
            清理分片缓存
        }
        // 没有历史分片，当前payload即完整消息体
    }

    // 7. 处理业务消息
    switch (opcode) {
        case 0x1: // 文本帧
        case 0x2: // 二进制帧
            上报到上层业务解包、分发（如 JSON, protobuf，自定义二进制协议等）
            break;
        case 0x8: // close
            连接关闭
            break;
        case 0x9: // ping
            回复 pong
            break;
        case 0xA: // pong
            // 心跳响应
            break;
        default:
            // 非法 opcode，关闭连接
            break;
    }
    // 8. 移除已处理数据
    buffer 移除已处理的数据段
}
```

##### Access实现要点

- **解码模块与自定义消息协议隔离**：解码器专注 WebSocket 帧处理，payload 转交模块做大厅握手、认证、业务消息路由（如`ModuleShake.so`）。
- **高性能处理**：采用非阻塞 I/O、缓冲池、内存复用，通过事件驱动及时处理所有 socket 读写（如 epoll/kqueue 模型）。
- **安全防御**：带长度上限和异常检测，防粘包/分包、协议畸形攻击，超过长度或非法帧立即断开。
- **支持大包与分片重组**：严格按 RFC 6455，允许大 payload 与分片重组，业务层可二次拆分。

##### 参考开源组件

可参考 C++ websocket 库如 [libwebsockets](https://libwebsockets.org/)、[boost::beast](https://www.boost.org/doc/libs/release/libs/beast/) 内核解码实现，适配自身业务层协议。

---

**小结**：

Access 网关的 WebSocket 解码器负责“协议层与业务层解耦”，高效完成握手、数据帧拆包和掩码还原，只将已还原的 payload 转给 IM 业务模块处理，实现高并发、低延迟、强健的网关解包能力。

[src: raw/ingested/3项目/分布式IM-雷漫/登录-websocket网关总结-五、Access-网关握手协议-五、Access-网关握手协议.md]