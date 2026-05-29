# SO_REUSEPORT

## 概述

`SO_REUSEPORT` 是 Linux 3.9 内核引入的 socket 选项，支持多个进程或线程绑定到同一个 IP 和端口，实现内核级的负载均衡。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## 特性

- 允许多个 socket bind/listen 在相同的 IP、相同的 TCP/UDP 端口
- 目的是同一个 IP、PORT 的请求在多个 listen socket 间负载均衡
- 安全上，监听相同 IP、PORT 的 socket 只能位于同一个用户下

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## 解决惊群问题

传统多进程/多线程 TCP 服务中，多个进程同时在一个 listen socket 上监听请求，当新连接到来时，要么全部唤醒（浪费资源），要么只唤醒一个（降低并发处理能力）。

`SO_REUSEPORT` 通过创建多个独立的 listen socket，每个进程监听不同的 listen socket，内核根据数据包的四元组和 listen socket 数量，使用固定的 hash 算法路由数据包，实现均衡唤醒。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## 存在的问题

1. **Listen Socket 数量变化时**：会造成握手数据包的前一个数据包路由到 A listen socket，后一个握手数据包路由到 B listen socket，导致客户端连接请求失败。
2. **短时间内负载不均衡**：各个 listen socket 间的负载可能不均衡。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## Go 中的使用

Go 可以通过 `golang.org/x/sys/unix` 库设置 `SO_REUSEPORT` 选项：

```go
import "golang.org/x/sys/unix"

unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
```

完整示例：

```go
package main

import (
    "context"
    "fmt"
    "net"
    "net/http"
    "os"
    "syscall"
    "golang.org/x/sys/unix"
)

var lc = net.ListenConfig{
    Control: func(network, address string, c syscall.RawConn) error {
        var opErr error
        if err := c.Control(func(fd uintptr) {
            opErr = unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
        }); err != nil {
            return err
        }
        return opErr
    },
}

func main() {
    pid := os.Getpid()
    l, err := lc.Listen(context.Background(), "tcp", "127.0.0.1:8000")
    if err != nil {
        panic(err)
    }
    server := &http.Server{}
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, "Client [%s] Received msg from Server PID: [%d] \n", r.RemoteAddr, pid)
    })
    fmt.Printf("Server with PID: [%d] is running \n", pid)
    _ = server.Serve(l)
}
```

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## 带来的好处

1. **提高服务器程序的吞吐性能**：充分利用多核 CPU 资源，避免单核处理瓶颈。
2. **内核级负载均衡**：不需要在多个实例前面添加一层服务代理。
3. **不停服更新**：可以启动新的服务实例来接受请求，再优雅地关闭旧服务实例。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## 相关页面

- [[TCP协议]]
- [[C++网络编程]]
- [[Go网络编程]]
