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

---

## 七、K8s 安装工具

### 7.1 工具对比

| 工具 | 适用场景 | 集群规模 | 复杂度 |
|------|---------|---------|:------:|
| **kubeadm** | 生产/多节点集群 | 多节点 | ⭐⭐⭐ |
| **minikube** | 本地单节点开发 | 单节点 | ⭐ |
| **kind** (Kubernetes-in-Docker) | CI/CD 测试 | 单/多节点 | ⭐ |
| **k3s** | 边缘/IoT/ARM | 单/多节点 | ⭐⭐ |
| **kube-spray** | 生产裸机/云 | 多节点 | ⭐⭐⭐⭐ |
| **MicroK8s** | 本地开发/边缘 | 单节点 | ⭐ |
| **Rancher RKE2** | 生产安全增强 | 多节点 | ⭐⭐⭐ |
| **EKS/GKE/AKS** | 云托管 | 企业级 | 托管 |

### 7.2 kubeadm（生产标准）

```bash
# 初始化控制平面
kubeadm init --apiserver-advertise-address=192.168.1.100 \
             --pod-network-cidr=10.244.0.0/16 \
             --service-cidr=10.96.0.0/12

# 加入工作节点（控制平面输出的 token）
kubeadm join 192.168.1.100:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# 查看/刷新 token
kubeadm token list
kubeadm token create --print-join-command
```

**安装流程**：
1. 所有节点安装容器运行时（containerd）
2. 所有节点安装 kubelet / kubeadm / kubectl
3. 控制平面：`kubeadm init`
4. 安装 CNI（网络插件，详见下方对比）
5. 工作节点：`kubeadm join`

### 7.2a CNI 网络插件对比

| 插件 | 网络模型 | 性能 | 功能特性 | 适用场景 |
|------|---------|:----:|---------|---------|
| **Flannel** | Overlay（VXLAN） | ⭐⭐ | 仅提供扁平网络，无策略 | 小型集群、快速搭建、学习环境 |
| **Calico** | 纯三层（BGP） | ⭐⭐⭐⭐⭐ | 网络策略（NetworkPolicy）、BGP 路由、eBPF 加速 | 生产环境、多租户、安全敏感场景 |
| **Weave** | Overlay（fastdp） | ⭐⭐⭐ | 自动加密、DNS 发现、多网络拓扑 | 多主机通信、需要透明加解密 |
| **Cilium** | eBPF 内核态转发 | ⭐⭐⭐⭐⭐ | 网络策略、可观测性、Service Mesh sidecar 替代 | 云原生前沿、高性能、可观测性要求高 |
| **Antrea** | Overlay（Open vSwitch） | ⭐⭐⭐⭐ | NetworkPolicy、流量监控、多集群 | VMware 生态、K8s + 虚拟化混合场景 |

**选型建议**：
- **学习/测试**：Flannel（最简单）
- **生产**：Calico（BGP 直连，无封包损耗）或 Cilium（eBPF 高性能）
- **多租户安全隔离**：Calico 的 NetworkPolicy 最成熟
- **云原生可观测性**：Cilium 的 Hubble 提供 L3-L7 流量可视化

### 7.2b Cilium 实战（eBPF 网络 + 可观测性）

Cilium 是 K8s 的 CNI 插件，基于 Linux eBPF 实现**高性能网络、安全策略和可观测性**，不需要 iptables 转发。

```bash
# 1. 用 kubeadm 创建集群后，直接安装 Cilium
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true       # 完全替代 kube-proxy

# 2. 验证安装
cilium status
kubectl get pods -n kube-system -l k8s-app=cilium

# 3. 启动 Hubble（流量可视化）
helm upgrade cilium cilium/cilium --namespace kube-system \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# 4. 打开 Hubble UI 查看流量拓扑
cilium hubble ui

# 5. 网络策略示例：只允许特定 Pod 访问
kubectl apply -f - <<EOF
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
EOF
```

**核心优势**：
| 能力 | 传统方案（iptables） | Cilium（eBPF） |
|------|:---:|:---:|
| 网络转发性能 | 规则链线性匹配 | 内核态哈希查找 |
| kube-proxy | 必须部署 | 可完全替换（性能更高） |
| NetworkPolicy | L3-L4 | L3-L7（支持 HTTP/gRPC） |
| 可观测性 | 额外部署 Prometheus | Hubble 内置，按流可视化 |
| 安全 | 第三方工具 | 基于身份的安全策略（Identity） |

### 7.3 kind（CI/CD 首选）

```bash
# 安装
brew install kind

# 创建集群
kind create cluster --name test --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  extraMounts:
  - hostPath: /path/to/data
    containerPath: /data
EOF

# 加载本地镜像（无需 push 到 registry）
kind load docker-image myapp:v1 --name test

# 删除集群
kind delete cluster --name test
```

### 7.4 minikube（本地开发）

```bash
# 安装
brew install minikube

# 启动集群
minikube start --cpus=4 --memory=8192 --driver=docker

# 启用仪表盘
minikube dashboard

# 访问 Service
minikube service myapp

# SSH 进入节点
minikube ssh

# 停止/删除
minikube stop && minikube delete
```

### 7.5 k3s（边缘/轻量）

```bash
# 安装服务端（自带 containerd + 轻量 SQLite）
curl -sfL https://get.k3s.io | sh -

# 工作节点加入
curl -sfL https://get.k3s.io | K3S_URL=https://server:6443 K3S_TOKEN=<token> sh -

# 查看节点
kubectl get nodes

# 停止服务
systemctl stop k3s
```

**特点**：二进制仅 60MB，内存占用低，内置 Traefik Ingress 和 Local Path Provisioner。

---

## 八、K8s 客户端工具

### 8.1 kubectl（CLI 标准）

| 命令 | 说明 |
|------|------|
| `kubectl get po \| deploy \| svc \| no \| ev --all-namespaces` | 资源查询 |
| `kubectl describe po <name>` | 查看详情与事件 |
| `kubectl logs -f <pod> [-c <container>]` | 查看日志 |
| `kubectl exec -it <pod> -- sh` | 进入容器 |
| `kubectl port-forward <pod> 8080:80` | 端口转发调试 |
| `kubectl cp <pod>:<path> <local>` | 文件复制 |
| `kubectl top pod --sort-by=cpu` | 资源排序查看 |
| `kubectl api-resources` | 列出所有资源类型 |
| `kubectl explain pod.spec` | 查看字段文档 |
| `kubectl rollout status deploy <name>` | 滚动更新状态 |

### 8.2 client-go（Go 客户端库）

**用途**：在 Go 代码中操作 K8s 资源（Operator、Controller、CI/CD 工具）。

```go
import (
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/clientcmd"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// 1. 加载 kubeconfig
config, _ := clientcmd.BuildConfigFromFlags("", filepath.Join(homedir.HomeDir(), ".kube", "config"))
clientset, _ := kubernetes.NewForConfig(config)

// 2. 列举所有 Pod
pods, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
for _, pod := range pods.Items {
    fmt.Println(pod.Name)
}

// 3. 创建 Deployment
deploy := &appsv1.Deployment{...}
clientset.AppsV1().Deployments("default").Create(ctx, deploy, metav1.CreateOptions{})

// 4. Watch 资源变化
watcher, _ := clientset.CoreV1().Pods("").Watch(ctx, metav1.ListOptions{})
for event := range watcher.ResultChan() {
    pod := event.Object.(*corev1.Pod)
    fmt.Println(event.Type, pod.Name)
}
```

**典型应用**：
- **Operator**（Kubebuilder / Operator SDK）
- **自定义调度器**
- **CI/CD 工具**（ArgoCD 管道的自定义步骤）
- **资源清理/备份工具**

### 8.3 常用第三方客户端

| 工具 | 类型 | 核心功能 |
|------|------|---------|
| **K9s** | TUI | Vim 风格快捷键、资源实时监控、日志/终端内置 |
| **Lens (OpenLens)** | 桌面 GUI | 集群管理、指标面板、终端集成 |
| **Rancher** | Web UI | 多集群管理、RBAC、应用商店 |
| **Octant** | Web UI | 资源可视化、插件扩展 |
| **Stern** | 日志尾部 | 多 Pod 日志聚合，标签/正则过滤 |
| **kubectx/kubens** | CLI 增强 | 秒级切换集群/命名空间 |
| **kubectl-tree** | CLI 插件 | 树状查看父子资源关系 |
| **popeye** | 安全扫描 | 集群配置安全/最佳实践检查 |

### 8.4 集群内客户端

```go
// InCluster 模式（Pod 内访问 API Server）
config, _ := rest.InClusterConfig()
clientset, _ := kubernetes.NewForConfig(config)

// 实现 Leader Election（避免多副本冲突）
leaderElector.Run(ctx)
```

**场景**：
- **Operator/Pod 内**：用 InClusterConfig，无需 kubeconfig
- **本地开发/CI**：用 BuildConfigFromFlags 加载 kubeconfig
- **Web 管理后台**：用 Token 认证，鉴权后限制 RBAC

### 面试考点

**Q: kubeadm 和 minikube 分别适用于什么场景？**
- kubeadm：**生产**多节点集群的搭建标准，支持高可用（堆叠 etcd / 外部 etcd）
- minikube：**本地开发/学习**，单节点，支持多种驱动（Docker/HyperV/VirtualBox）

**Q: kind 如何实现低成本多节点测试？**
- 每个 K8s 节点是一个 Docker 容器，共享宿主机内核
- 适合 CI 流程（启动秒级，用完即删）
- `kind load docker-image` 直接注入本地镜像，无需镜像仓库

**Q: client-go 的两种认证模式？**
- **OutOfCluster**：读取 `~/.kube/config`（本地开发）
- **InCluster**：Pod 挂载的 ServiceAccount Token（集群内运行）

**Q: List-Watch 模式的原理？**
- **List**：全量拉取当前资源状态（首次/重连）
- **Watch**：监听后续变更事件，通过 HTTP 长连接推送
- 结合本地缓存实现**增量同步**，避免频繁全量拉取
