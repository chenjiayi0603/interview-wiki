# Go gRPC 客户端

## 5.1 一元与流式调用

```go
package main

import (
	"context"
	"io"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"your-module/proto/greet"
)

func main() {
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	client := greet.NewGreeterClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// 一元 RPC
	reply, err := client.SayHello(ctx, &greet.HelloRequest{Name: "World"})
	if err != nil {
		log.Fatal(err)
	}
	log.Println("SayHello:", reply.GetMessage())

	// 服务端流
	stream, err := client.ListGreetings(ctx, &greet.HelloRequest{Name: "Stream"})
	if err != nil {
		log.Fatal(err)
	}
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatal(err)
		}
		log.Println("ListGreetings:", msg.GetMessage())
	}

	// 客户端流
	upload, err := client.MultiGreet(ctx)
	if err != nil {
		log.Fatal(err)
	}
	for _, name := range []string{"A", "B", "C"} {
		if err := upload.Send(&greet.HelloRequest{Name: name}); err != nil {
			log.Fatal(err)
		}
	}
	resp, err := upload.CloseAndRecv()
	if err != nil {
		log.Fatal(err)
	}
	log.Println("MultiGreet:", resp.GetMessage())

	// 双向流
	chat, err := client.Chat(ctx)
	if err != nil {
		log.Fatal(err)
	}
	go func() {
		for {
			msg, err := chat.Recv()
			if err == io.EOF {
				return
			}
			if err != nil {
				log.Println("Recv err:", err)
				return
			}
			log.Println("Chat recv:", msg.GetMessage())
		}
	}()
	if err := chat.Send(&greet.HelloRequest{Name: "Hi"}); err != nil {
		log.Fatal(err)
	}
	time.Sleep(time.Second)
	chat.CloseSend()
}
```

## 5.2 客户端选项（超时、拦截器、TLS）完整示例

```go
package main

import (
	"context"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/credentials/insecure"
	"your-module/proto/greet"
)

func main() {
	// 有 CA 证书时用 TLS；本地开发可用 insecure
	opts := []grpc.DialOption{
		grpc.WithUnaryInterceptor(func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
			log.Printf("calling %s", method)
			return invoker(ctx, method, req, reply, cc, opts...)
		}),
	}
	if creds, err := credentials.NewClientTLSFromFile("ca.crt", ""); err == nil {
		opts = append(opts, grpc.WithTransportCredentials(creds))
	} else {
		opts = append(opts, grpc.WithTransportCredentials(insecure.NewCredentials()))
	}

	conn, err := grpc.NewClient("localhost:50051", opts...)
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	client := greet.NewGreeterClient(conn)

	// 带超时的一元调用
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	reply, err := client.SayHello(ctx, &greet.HelloRequest{Name: "World"})
	if err != nil {
		log.Fatal(err)
	}
	log.Println("SayHello:", reply.GetMessage())
}
```

[src: raw/ingested/2技术/go/第三方库-go-grpc分析与应用例子-五、Go-gRPC-客户端.md]