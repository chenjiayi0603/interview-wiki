# gRPC 与 Gin 网关模式

在微服务中常见做法：对外仍用 HTTP/REST（如 Gin），对内用 gRPC。Gin 作为 API 网关转发到 gRPC 服务。

```
┌──────────────┐     HTTP/JSON      ┌─────────────┐     gRPC      ┌─────────────┐
│  浏览器/App   │ ─────────────────→ │  Gin 网关   │ ────────────→ │ gRPC 服务   │
└──────────────┘                    └─────────────┘               └─────────────┘
```

## Gin 网关完整可运行示例

启动前需先运行 gRPC 服务端在 `localhost:50051`。

### greet.proto 示例

```protobuf
syntax = "proto3";

package greet;

option go_package = "your-module/proto/greet;greet";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
  rpc ListGreetings (HelloRequest) returns (stream HelloReply);
  rpc MultiGreet (stream HelloRequest) returns (HelloReply);
  rpc Chat (stream HelloRequest) returns (stream HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

### Go 代码示例

```go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"your-module/proto/greet"
)

var greetClient greet.GreeterClient

func main() {
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("grpc dial: %v", err)
	}
	defer conn.Close()
	greetClient = greet.NewGreeterClient(conn)

	r := gin.Default()
	r.GET("/hello", helloHandler)
	log.Println("Gin gateway on :8080")
	if err := r.Run(":8080"); err != nil && err != http.ErrServerClosed {
		log.Fatal(err)
	}
}

func helloHandler(c *gin.Context) {
	name := c.Query("name")
	if name == "" {
		name = "World"
	}
	reply, err := greetClient.SayHello(c.Request.Context(), &greet.HelloRequest{Name: name})
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"message": reply.GetMessage()})
}
```

访问：`http://localhost:8080/hello?name=张三` 即可得到 JSON：`{"message":"Hello, 张三"}`。

也可使用 **grpc-gateway**：根据 `.proto` 生成 REST 映射，由同一套 proto 同时提供 gRPC 和 HTTP API。

[src: raw/ingested/2技术/go/第三方库-go-grpc分析与应用例子-七、与-Gin-的配合（网关模式）.md]