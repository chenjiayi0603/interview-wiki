# Kubernetes基础

## 核心概念
- **Pod**：最小调度单元，包含一个或多个容器
- **Node**：工作节点，运行Pod
- **Cluster**：一组Node，组成计算集群
- **Namespace**：资源隔离
- **Deployment**：声明式更新Pod
- **Service**：服务发现和负载均衡
- **Ingress**：HTTP/HTTPS路由

## K8s架构
```
┌─────────────────────────────────────────────────┐
│                    Master节点                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │API Server│  │Scheduler│  │Controller│        │
│  │         │  │         │  │ Manager  │        │
│  └─────────┘  └─────────┘  └─────────┘        │
│                     │                          │
│               ┌─────┴─────┐                    │
│               │  etcd     │                    │
│               └───────────┘                    │
└─────────────────────────────────────────────────┘
          │                    │
┌─────────┴─────────┐  ┌─────────┴─────────┐
│   Worker Node     │  │   Worker Node     │
│  ┌─────────────┐  │  │  ┌─────────────┐  │
│  │   kubelet   │  │  │  │   kubelet   │  │
│  │  kube-proxy │  │  │  │  kube-proxy │  │
│  │ ┌─────────┐ │  │  │  │ ┌─────────┐ │  │
│  │ │  Pod1   │ │  │  │  │ │  Pod2   │ │  │
│  │ │ container│ │  │  │  │ │ container│ │  │
│  │ └─────────┘ │  │  │  │ └─────────┘ │  │
│  └─────────────┘  │  │  └─────────────┘  │
│  ┌─────────────┐  │  │  ┌─────────────┐  │
│  │   CRI      │  │  │  │   CRI      │  │
│  └─────────────┘  │  │  └─────────────┘  │
└──────────────────┘  └──────────────────┘
```

## Pod生命周期
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:v1
    ports:
    - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

[src: raw/ingested/2技术/虚拟化/云原生与K8s-一、Kubernetes基础.md]