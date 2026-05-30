# Go gRPC 服务端

## 4.1 实现服务接口

生成代码后，服务端需实现 `greet_grpc.pb.go` 中的接口（如 `GreeterServer`）。

```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net"
	"strings"

	"google.golang.org/grpc"
	"your-module/proto/greet"
)

type server struct {
	greet.UnimplementedGreeterServer
}

// 一元 RPC
func (s *server) SayHello(ctx context.Context, req *greet.HelloRequest) (*greet.HelloReply, error) {
	return &greet.HelloReply{Message: "Hello, " + req.GetName()}, nil
}

// 服务端流
func (s *server) ListGreetings(req *greet.HelloRequest, stream greet.Greeter_ListGreetingsServer) error {
	for i := 0; i < 3; i++ {
		if err := stream.Send(&greet.HelloReply{
			Message: fmt.Sprintf("Hello %s #%d", req.GetName(), i),
		}); err != nil {
			return err
		}
	}
	return nil
}

// 客户端流
func (s *server) MultiGreet(stream greet.Greeter_MultiGreetServer) error {
	var names []string
	for {
		req, err := stream.Recv()
		if err == io.EOF {
			return stream.SendAndClose(&greet.HelloReply{
				Message: "Greeted: " + strings.Join(names, ", "),
			})
		}
		if err != nil {
			return err
		}
		names = append(names, req.GetName())
	}
}

// 双向流（示例：回显）
func (s *server) Chat(stream greet.Greeter_ChatServer) error {
	for {
		req, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		if err := stream.Send(&greet.HelloReply{Message: "Echo: " + req.GetName()}); err != nil {
			return err
		}
	}
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}
	s := grpc.NewServer()
	greet.RegisterGreeterServer(s, &server{})
	log.Println("gRPC server listening on :50051")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("serve: %v", err)
	}
}
```

注意：
- 必须内嵌 `UnimplementedGreeterServer`，以便在未实现某 RPC 时返回“未实现”而不是 panic。
- 流式方法通过 `stream.Send` / `stream.Recv` / `stream.SendAndClose` 与客户端交互。

## 4.2 服务端选项（TLS、拦截器）完整示例

```go
package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"your-module/proto/greet"
)

type server struct{ greet.UnimplementedGreeterServer }

func (s *server) SayHello(ctx context.Context, req *greet.HelloRequest) (*greet.HelloReply, error) {
	return &greet.HelloReply{Message: "Hello, " + req.GetName()}, nil
}

func main() {
	// 使用 TLS（生产环境推荐；无证书时可先注释掉 Creds，用 NewServer()）
	creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
	if err != nil {
		log.Printf("no TLS certs, serving without TLS: %v", err)
		creds = nil
	}
	var opts []grpc.ServerOption
	if creds != nil {
		opts = append(opts, grpc.Creds(creds))
	}
	opts = append(opts, grpc.ChainUnaryInterceptor(
		func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
			log.Printf("method=%s", info.FullMethod)
			return handler(ctx, req)
		},
	))
	s := grpc.NewServer(opts...)
	greet.RegisterGreeterServer(s, &server{})

	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}
	log.Println("gRPC server (with optional TLS + logging interceptor) on :50051")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("serve: %v", err)
	}
}
```

[src: raw/ingested/2技术/go/第三方库-go-grpc分析与应用例子-四、Go-gRPC-服务端.md]