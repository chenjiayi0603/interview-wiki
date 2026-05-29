# Go gRPC 拦截器与错误处理

## 6.1 服务端一元拦截器（认证示例）完整示例

```go
package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
	"your-module/proto/greet"
)

type server struct{ greet.UnimplementedGreeterServer }

func (s *server) SayHello(ctx context.Context, req *greet.HelloRequest) (*greet.HelloReply, error) {
	// 可从 context 取出拦截器注入的 user_id
	if uid, ok := ctx.Value("user_id").(string); ok {
		log.Printf("user_id=%s", uid)
	}
	return &greet.HelloReply{Message: "Hello, " + req.GetName()}, nil
}

func authInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Error(codes.Unauthenticated, "missing metadata")
	}
	vals := md.Get("authorization")
	if len(vals) == 0 {
		return nil, status.Error(codes.Unauthenticated, "missing authorization")
	}
	// 校验 token（此处仅示例，实际可验证 JWT 等），通过后放入 context
	ctx = context.WithValue(ctx, "user_id", "123")
	return handler(ctx, req)
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}
	s := grpc.NewServer(grpc.UnaryInterceptor(authInterceptor))
	greet.RegisterGreeterServer(s, &server{})
	log.Println("gRPC server with auth interceptor on :50051")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("serve: %v", err)
	}
}
```

客户端调用时需在 metadata 中带上 `authorization`（如 `metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer xxx")`），否则会收到 `Unauthenticated`。

## 6.2 错误与 status 包完整示例

gRPC 使用 `status` 和 `codes` 表示错误，便于跨语言一致处理。

**服务端：在 Handler 中返回业务错误**

```go
package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"your-module/proto/greet"
)

type server struct{ greet.UnimplementedGreeterServer }

func (s *server) SayHello(ctx context.Context, req *greet.HelloRequest) (*greet.HelloReply, error) {
	if req.GetName() == "" {
		return nil, status.Error(codes.InvalidArgument, "name is required")
	}
	// 示例：按 name 查“用户”，未找到则返回 NotFound
	if req.GetName() == "nobody" {
		return nil, status.Errorf(codes.NotFound, "user %s not found", req.GetName())
	}
	return &greet.HelloReply{Message: "Hello, " + req.GetName()}, nil
}

func main() {
	lis, _ := net.Listen("tcp", ":50051")
	s := grpc.NewServer()
	greet.RegisterGreeterServer(s, &server{})
	log.Println("gRPC server (with status errors) on :50051")
	_ = s.Serve(lis)
}
```

**客户端：解析 gRPC 错误**

```go
package main

import (
	"context"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/status"
	"your-module/proto/greet"
)

func main() {
	conn, _ := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	defer conn.Close()
	client := greet.NewGreeterClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// 正常调用
	reply, err := client.SayHello(ctx, &greet.HelloRequest{Name: "World"})
	if err != nil {
		handleGRPCError(err)
		return
	}
	log.Println("reply:", reply.GetMessage())

	// 触发 InvalidArgument（空 name）
	_, err = client.SayHello(ctx, &greet.HelloRequest{Name: ""})
	if err != nil {
		handleGRPCError(err)
	}

	// 触发 NotFound（name=nobody）
	_, err = client.SayHello(ctx, &greet.HelloRequest{Name: "nobody"})
	if err != nil {
		handleGRPCError(err)
	}
}

func handleGRPCError(err error) {
	st, ok := status.FromError(err)
	if !ok {
		log.Println("non-gRPC error:", err)
		return
	}
	switch st.Code() {
	case codes.InvalidArgument:
		log.Println("invalid argument:", st.Message())
	case codes.NotFound:
		log.Println("not found:", st.Message())
	default:
		log.Printf("code=%s message=%s", st.Code(), st.Message())
	}
}
```

[src: raw/ingested/2技术/go/第三方库-go-grpc分析与应用例子-六、拦截器与错误处理.md]