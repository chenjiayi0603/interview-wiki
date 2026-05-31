# gRPC Protocol Buffers

## 1. Protocol Buffers

```protobuf
syntax = "proto3";

package user;

option go_package = "user/pb";

service UserService {
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
    rpc CreateUser(stream CreateUserRequest) returns (CreateUserResponse);
    rpc StreamUsers(GetUserRequest) returns (stream User);
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
    int64 created_at = 4;
}

message GetUserRequest {
    int32 id = 1;
}

message GetUserResponse {
    User user = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
}

message ListUsersResponse {
    repeated User users = 1;
    int32 total = 2;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message CreateUserResponse {
    User user = 1;
}
```

## 2. gRPC拦截器

```go
// 服务端拦截器
funcUnary := grpc.UnaryInterceptor(func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    // 前置处理
    start := time.Now()
    
    // 调用实际处理函数
    resp, err := handler(ctx, req)
    
    // 后置处理
    log.Printf("method=%s duration=%v", info.FullMethod, time.Since(start))
    
    return resp, err
})

// 客户端拦截器
func main() {
    conn, err := grpc.Dial("localhost:50051",
        grpc.WithUnaryInterceptor(func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker) error {
            // 添加认证信息
            md := metadata.Pairs("token", "my-token")
            ctx = metadata.NewOutgoingContext(ctx, md)
            return invoker(ctx, method, req, reply, cc)
        }),
    )
}
```

## 3. gRPC负载均衡

```go
// 客户端负载均衡
// 等访问特定Endpoint后获取服务列表

// 简单轮询
conn, _ := grpc.Dial(
    "xds:///my-service",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"round_robin":{}}]}`),
)

// 区域感知
conn, _ := grpc.Dial(
    "xds:///my-service",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"cluster_manager":{}}]}`),
)
```

[src: raw/ingested/2技术/虚拟化/云原生与K8s-三、gRPC.md]