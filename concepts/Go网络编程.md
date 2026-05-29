# Go 网络编程

> Go 在网络编程领域非常强大，标准库自带高效的 `net`、`net/http` 包，能轻松实现 TCP、UDP、HTTP 服务器。

See also: [[Go语言基础]], [[Goroutine调度]], [[Channel]], [[Go并发安全]], [[TCP协议]]

---

## 🧩 一、网络编程基础概念

| 概念 | 说明 |
|:--|:--|
| IP 地址 | 网络中唯一标识主机 |
| 端口号 | 区分同一主机上的多个服务 |
| 套接字（Socket） | IP + 端口 的组合，用于通信 |
| TCP | 面向连接、可靠传输（如：HTTP） |
| UDP | 无连接、快速传输（如：DNS、游戏） |

[src: raw/ingested/2技术/go/标准库-Go网络编程入门.md]

---

## 🧱 二、TCP 编程

### 🖥️ 1. TCP 服务端

```go
package main

import (
    "bufio"
    "fmt"
    "net"
)

func main() {
    listener, err := net.Listen("tcp", "127.0.0.1:8080")
    if err != nil {
        panic(err)
    }
    fmt.Println("Server listening on 127.0.0.1:8080")

    for {
        conn, err := listener.Accept()
        if err != nil {
            fmt.Println("Accept error:", err)
            continue
        }
        go handle(conn)
    }
}

func handle(conn net.Conn) {
    defer conn.Close()
    reader := bufio.NewReader(conn)

    for {
        msg, err := reader.ReadString('\n')
        if err != nil {
            fmt.Println("Client disconnected")
            return
        }
        fmt.Println("Received:", msg)
        conn.Write([]byte("Echo: " + msg))
    }
}
```

### 💻 2. TCP 客户端

```go
package main

import (
    "bufio"
    "fmt"
    "net"
    "os"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:8080")
    if err != nil {
        panic(err)
    }
    defer conn.Close()

    reader := bufio.NewReader(os.Stdin)
    for {
        fmt.Print("输入消息：")
        msg, _ := reader.ReadString('\n')
        conn.Write([]byte(msg))
        reply := make([]byte, 1024)
        n, _ := conn.Read(reply)
        fmt.Println("收到:", string(reply[:n]))
    }
}
```

[src: raw/ingested/2技术/go/标准库-Go网络编程入门.md]

---

## ⚡ 三、UDP 编程

### 🧭 UDP 服务端

```go
package main

import (
    "fmt"
    "net"
)

func main() {
    addr, _ := net.ResolveUDPAddr("udp", "127.0.0.1:9000")
    conn, _ := net.ListenUDP("udp", addr)
    defer conn.Close()
    fmt.Println("UDP server running...")

    var buf [1024]byte
    for {
        n, clientAddr, _ := conn.ReadFromUDP(buf[:])
        fmt.Printf("收到来自 %v 的消息: %s\n", clientAddr, string(buf[:n]))
        conn.WriteToUDP([]byte("OK"), clientAddr)
    }
}
```

### 💬 UDP 客户端

```go
package main

import (
    "fmt"
    "net"
)

func main() {
    serverAddr, _ := net.ResolveUDPAddr("udp", "127.0.0.1:9000")
    conn, _ := net.DialUDP("udp", nil, serverAddr)
    defer conn.Close()

    conn.Write([]byte("hello UDP server"))
    buf := make([]byte, 1024)
    n, _, _ := conn.ReadFromUDP(buf)
    fmt.Println("服务端回应:", string(buf[:n]))
}
```

[src: raw/ingested/2技术/go/标准库-Go网络编程入门.md]

---

## 🌍 四、HTTP 编程

### 🚀 HTTP 服务端

```go
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", handler)
    fmt.Println("HTTP server running on :8080")
    http.ListenAndServe(":8080", nil)
}
```

访问：[http://localhost:8080/world](http://localhost:8080/world)

---

### 🌐 HTTP 客户端

```go
package main

import (
    "fmt"
    "io"
    "net/http"
)

func main() {
    resp, _ := http.Get("https://httpbin.org/get")
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

[src: raw/ingested/2技术/go/标准库-Go网络编程入门.md]

---

## 🧵 五、并发网络服务器

```go
go handle(conn)
```

```go
messages := make(chan string)

go func() {
    messages <- "ping"
}()

msg := <-messages
fmt.Println(msg)
```

[src: raw/ingested/2技术/go/标准库-Go网络编程入门.md]

---

## 🧠 六、示例：并发聊天服务器

```go
package main

import (
    "bufio"
    "fmt"
    "net"
)

var clients = make(map[net.Conn]bool)
var messages = make(chan string)

func main() {
    ln, _ := net.Listen("tcp", ":8080")
    defer ln.Close()
    fmt.Println("Chat server started on :8080")

    go broadcast()

    for {
        conn, _ := ln.Accept()
        clients[conn] = true
        go handle(conn)
    }
}

func handle(conn net.Conn) {
    defer conn.Close()
    reader := bufio.NewReader(conn)
    for {
        msg, err := reader.ReadString('\n')
        if err != nil {
            delete(clients, conn)
            return
        }
        messages <- msg
    }
}

func broadcast() {
    for msg := range messages {
        for conn := range clients {
            conn.Write([]byte(msg))
        }
    }
}
```

[src: raw/ingested/2技术/go/标准库-Go网络编程入门.md]

---

## 🧰 七、常用库与技巧

| 库 | 作用 |
|:--|:--|
| `net` | TCP/UDP 通信 |
| `net/http` | Web 服务、HTTP 客户端 |
| `io` / `bufio` | 数据流处理 |
| `encoding/json` | JSON 序列化/反序列化 |
| `context` | 控制超时与取消 |
| `sync.WaitGroup` | 并发等待管理 |

[src: raw/ingested/2技术/go/标准库-Go网络编程入门.md]

---

## 🔐 八、扩展学习方向

- Web 框架：Gin / Fiber / Echo  
- RPC 框架：gRPC  
- WebSocket：`golang.org/x/net/websocket`  
- 高性能网络库：`fasthttp`、`evio`

[src: raw/ingested/2技术/go/标准库-Go网络编程入门.md]

---

## ✅ 总结

| 网络模型 | 特点 | 适用场景 |
|:--|:--|:--|
| TCP | 可靠传输、面向连接 | 聊天、文件传输 |
| UDP | 无连接、高性能 | 游戏、视频流 |
| HTTP | 基于 TCP 的应用层协议 | Web 服务、API |
| WebSocket | 全双工通信 | 实时聊天、推送 |

[src: raw/ingested/2技术/go/标准库-Go网络编程入门.md]

## Related Pages
- [[Go语言基础]]
- [[Goroutine调度]]
- [[Channel]]
- [[Go并发安全]]
- [[TCP协议]]
- [[Go框架与工具]]
