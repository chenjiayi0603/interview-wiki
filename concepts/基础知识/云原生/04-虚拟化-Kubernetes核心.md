# Kubernetes 核心

> K8s 架构、Pod/Deployment/Service/Ingress、HPA、网络模型。

---

## 一、核心概念

| 概念 | 说明 |
|------|------|
| **Pod** | 最小调度单元，包含一个或多个容器 |
| **Node** | 工作节点，运行 Pod |
| **Cluster** | 一组 Node 组成的计算集群 |
| **Namespace** | 逻辑隔离 |
| **Deployment** | 声明式 Pod 更新 |
| **Service** | 服务发现与负载均衡 |
| **Ingress** | HTTP/HTTPS 路由 |

---

## 二、K8s 架构

```
                    Master
┌──────────────────────────────────────┐
│  API Server   Scheduler  Controller  │
│  Manager                              │
│          etcd                         │
└──────────────────────────────────────┘
         │                   │
┌────────▼────────┐ ┌───────▼────────┐
│   Worker Node   │ │   Worker Node  │
│  ┌────────────┐ │ │  ┌────────────┐│
│  │   kubelet  │ │ │  │   kubelet  ││
│  │ kube-proxy │ │ │  │ kube-proxy ││
│  │ ┌──────┐  │ │ │  │ ┌──────┐  ││
│  │ │ Pod1 │  │ │ │  │ │ Pod2 │  ││
│  │ │container│ │ │  │ │container││
│  │ └──────┘  │ │ │  │ └──────┘  ││
│  │   CRI     │ │ │  │   CRI     ││
│  └────────────┘ │ │  └────────────┘│
└──────────────────┘ └────────────────┘
```

### 组件职责

| 组件 | 职责 |
|------|------|
| **API Server** | 所有操作的入口，RESTful 接口 |
| **Scheduler** | 将 Pod 调度到合适的 Node |
| **Controller Manager** | 维护集群状态（Deployment/RS 等） |
| **etcd** | 集群状态存储（键值数据库） |
| **kubelet** | 管理节点上的 Pod 和容器 |
| **kube-proxy** | 网络代理，实现 Service |

---

## 三、Pod 生命周期

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

### 探针类型

| 探针 | 作用 | 失败处理 |
|------|------|---------|
| **livenessProbe** | 检查容器是否存活 | 重启容器 |
| **readinessProbe** | 检查容器是否就绪 | 切出 Service 负载 |
| **startupProbe** | 检查容器是否启动完成 | 延迟 liveness 检查 |

---

## 四、工作负载

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 最多超出副本数
      maxUnavailable: 0  # 更新期间不可用数
  selector:
    matchLabels:
      app: myapp
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2
```

**更新策略**：
- **RollingUpdate**（默认）：逐步替换 Pod，`maxSurge`/`maxUnavailable` 控制速率
- **Recreate**：先删所有旧 Pod，再建新 Pod

### HPA（自动扩缩容）

```bash
kubectl autoscale deployment myapp --cpu-percent=80 --min=3 --max=10
```

**原理**：基于 CPU/内存/自定义指标，计算目标副本数 `ceil(current * desired / current)`。

---

## 五、Service 与网络

### 5.1 Service 类型

| 类型 | 访问范围 | 说明 |
|------|---------|------|
| **ClusterIP** | 集群内部 | 默认类型 |
| **NodePort** | 集群外（节点端口） | 测试用 |
| **LoadBalancer** | 公网 | 云厂商负载均衡 |
| **ExternalName** | CNAME 映射 | 外部服务别名 |

### 5.2 kube-proxy 模式

| 模式 | 原理 | 性能 |
|------|------|:----:|
| iptables | 规则匹配 | 中 |
| IPVS | 内核级转发 | 高 |

### 5.3 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: myapp-api
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-web
            port:
              number: 80
```

### 5.4 网络模型

- 所有 Pod 可直接通信（扁平网络）
- Pod 与 Service 通过 kube-proxy 通信
- Service 通过 CoreDNS 发现

---

## 六、面试考点

### Q1: K8s 和 Docker 的区别？
**A**：Docker 是容器运行时，单机管理；K8s 是容器编排平台，跨主机集群管理。Docker 是 K8s 的底层实现之一（CRI）。

### Q2: Pod 和 Container 的区别？
**A**：Pod 是 K8s 最小调度单元，可包含多个容器（sidecar 模式），共享网络/存储/UTS 命名空间。

### Q3: Service 如何实现服务发现？
**A**：CoreDNS 解析 Service 名，或 kubelet 注入环境变量 `SERVICE_HOST`。推荐 DNS。

### Q4: 如何排查 Pod 问题？
1. `kubectl describe pod` 查看事件
2. `kubectl logs` 查看日志
3. `kubectl exec` 进入容器
4. `kubectl get events` 查看集群事件
5. `kubectl top` 查看资源使用

### Q5: Ingress vs Service 的区别？
- **Service**：集群内部负载均衡（ClusterIP），或通过 NodePort/LB 暴露到外部
- **Ingress**：七层（HTTP/HTTPS）路由，根据域名/路径转发到不同 Service
- 选型：四层用 Service(LB)，七层用 Ingress

### Q6: ConfigMap 和 Secret 区别？
- ConfigMap：明文存储配置（键值对、文件），Pod 通过环境变量或挂载使用
- Secret：Base64 编码存储敏感信息（密码、证书），支持加密（KMS）+ RBAC
- **安全建议**：使用 Sealed Secrets / External Secrets Operator 管理

### Q7: etcd 在 K8s 中的作用？
- 集群所有状态的存储（键值数据库）
- API Server 的无状态设计依赖 etcd 提供一致性和持久化
- **高可用**：至少 3 节点 etcd 集群，Raft 协议保证一致性

### Q8: 如何实现零停机滚动更新？
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # 更新时不少于当前副本数
      maxSurge: 1              # 允许额外 1 个 Pod 临时运行
```

### Q5: HPA 工作原理？
**A**：HorizontalPodAutoscaler 基于 CPU/内存/自定义指标，周期性检查当前指标，计算目标副本数 `ceil(current * desired / current)`，更新 Deployment 副本数。
