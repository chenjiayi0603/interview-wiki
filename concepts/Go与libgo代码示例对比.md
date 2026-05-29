# Go 与 libgo 代码示例对比

> 本文对比 Go 和 libgo（C++ 协程库）在典型并发场景下的代码实现，包括生产者-消费者模型和并发 HTTP 请求。

See also: [[C++协程库对比-Folly-Asio-Cobalt-libgo]], [[Goroutine调度]], [[Channel]]

## 一、生产者-消费者模型

### Go 实现

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    ch := make(chan int, 10)
    var wg sync.WaitGroup
    
    // 生产者
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < 100; i++ {
            ch <- i
        }
        close(ch)
    }()
    
    // 消费者
    wg.Add(1)
    go func() {
        defer wg.Done()
        for val := range ch {
            fmt.Printf("Received: %d\n", val)
        }
    }()
    
    wg.Wait()
}
```

### libgo 实现

```cpp
#include <libgo/libgo.h>
#include <stdio.h>

int main() {
    co_chan<int> ch(10);
    
    // 生产者
    go [=]() mutable {
        for (int i = 0; i < 100; i++) {
            ch << i;
        }
        ch.Close();
    };
    
    // 消费者
    go [=]() mutable {
        int val;
        while (ch >> val) {
            printf("Received: %d\n", val);
        }
    };
    
    // 启动调度器
    // 启动调度器并指定2个调度线程（即创建2个工作线程）。
    // 线程数可以根据硬件或负载需求灵活调整，如 co_sched.Start(4)。
    // 但 libgo 目前不支持运行时动态增减调度线程数量，线程数需在启动时确定。
    co_sched.Start(2);
    return 0;
}
```

## 二、并发 HTTP 请求

### Go 实现

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
)

func main() {
    urls := []string{
        "http://example.com",
        "http://example.org",
        "http://example.net",
    }
    
    var wg sync.WaitGroup
    
    for _, url := range urls {
        wg.Add(1)
        go func(url string) {
            defer wg.Done()
            resp, err := http.Get(url)
            if err != nil {
                fmt.Printf("Error: %v\n", err)
                return
            }
            defer resp.Body.Close()
            body, _ := io.ReadAll(resp.Body)
            fmt.Printf("%s: %d bytes\n", url, len(body))
        }(url)
    }
    
    wg.Wait()
}
```

### libgo 实现（需配合网络库）

```cpp
#include <libgo/libgo.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

void fetch(const char* host) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    
    // 这部分代码做了以下事情：
    // 1. 定义 sockaddr_in 结构体变量 addr，用于描述目标服务器的地址和端口。
    // 2. 设置地址族为 AF_INET（IPv4）。
    // 3. 设置端口为 80（HTTP 默认端口），使用 htons 做网络字节序转换。
    // 4. 使用 inet_pton 将主机 IP 字符串转为网络字节序二进制，并赋值给 addr.sin_addr。
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(80);
    inet_pton(AF_INET, host, &addr.sin_addr);
    
    // Hook 后的 connect 不阻塞线程
    connect(fd, (struct sockaddr*)&addr, sizeof(addr));
    
    const char* req = "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n";
    write(fd, req, strlen(req));
    
    char buf[4096];
    int n = read(fd, buf, sizeof(buf));  // Hook 后不阻塞
    printf("Received %d bytes from %s\n", n, host);
    
    close(fd);
}

int main() {
    go []{ fetch("93.184.216.34"); };
    go []{ fetch("93.184.216.34"); };
    go []{ fetch("93.184.216.34"); };
    
    co_sched.Start(4);
    return 0;
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-八、代码示例完整对比.md]