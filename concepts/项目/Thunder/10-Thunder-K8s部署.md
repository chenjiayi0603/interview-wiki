# Thunder K8s 部署

> Kubernetes 容器化部署方案，基于 Docker Compose 和 Thunder 服务架构迁移。

---

## 一、架构总览

### 1.1 服务列表

| 服务 | 镜像 | 端口 | 说明 |
|------|------|------|------|
| **center** | thunder-node | 27000/26000 | 注册中心 + Raft 集群 |
| **logic** | thunder-node | 16068 | 业务逻辑层 |
| **interface** | thunder-node | 27008/27009 | API 网关层 |
| **hello** | thunder-node | 27006/27007 | HTTP 示例服务 |
| **redis** | redis:7-alpine | 6379 | 缓存 |
| **mysql** | mariadb:11.2 | 3306 | 持久化存储 |

---

## 二、Dockerfile

```dockerfile
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates bash net-tools procps iproute2 \
        libjemalloc2 libbrotli1 libsnappy1v5 \
    && rm -rf /var/lib/apt/lists/*
COPY deploy/bin   /thunder/deploy/bin
COPY deploy/lib   /thunder/deploy/lib
COPY deploy/3lib  /thunder/deploy/3lib
COPY deploy/plugins /thunder/deploy/plugins
WORKDIR /thunder
```

---

## 三、K8s 资源

### 3.1 Namespace

> **资源作用**：Namespace 是 K8s 的逻辑隔离单元，将一组资源（Pod、Service、ConfigMap 等）划入独立空间，避免多项目之间的名称冲突，并为 RBAC、NetworkPolicy、ResourceQuota 提供作用边界。
>
> **选用逻辑**：将所有 thunder 服务放在独立 namespace 下，与集群中监控、日志、中间件等其他项目完全隔离。不另建独立集群（成本过高），不裸跑无隔离（多项目共享集群时配置极易互相干扰）。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: thunder
```

### 3.2 ConfigMap

> **资源作用**：ConfigMap 将配置从容器镜像中剥离，以 Key-Value 或文件形式注入 Pod，实现「配置与镜像分离」—— 同一套镜像挂载不同的 ConfigMap 即可变成不同的服务。
>
> **选用逻辑**：Center、Hello、Logic、Interface 共用 `thunder-node` 镜像，通过不同的 ConfigMap 区分各自的 JSON 配置文件。相比把配置硬编码进 Dockerfile：
> - 修改配置无需重打镜像，CI/CD 更轻量
> - 同一镜像可在测试/预发/生产等多环境通过不同 ConfigMap 复用到
> - 配合 `subPath` 挂载为单文件，不影响容器内其他目录结构
> - 不选 Secret，因配置不含密码/证书等敏感数据，Secret 仅 base64 编码无实际安全增益，反而增加 `kubectl describe` 调试时的阅读成本

Center 配置（Raft 3 节点共用，通过 StatefulSet DNS 区分）：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: center-config
  namespace: thunder
data:
  Center.json: |
    {
      "node_type": "CENTER",
      "inner_host": "0.0.0.0",
      "inner_port": 27000,
      "center": "center-0.center-svc.thunder:27000,center-1.center-svc.thunder:27000,center-2.center-svc.thunder:27000",
      "server_name": "Center_robot",
      "process_num": 1,
      "cpu_affinity": false,
      "log_path": "/dev/stdout",
      "log_level": "INFO",
      "custom": { "need_leadership": true }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-config
  namespace: thunder
data:
  Hello.json: |
    {
      "node_type": "HELLO",
      "access_host": "0.0.0.0",
      "access_port": 27006,
      "inner_host": "0.0.0.0",
      "inner_port": 27007,
      "center": "center-0.center-svc.thunder:27000,center-1.center-svc.thunder:27000,center-2.center-svc.thunder:27000",
      "server_name": "Hello_robot",
      "process_num": 1,
      "worker_capacity": 1000000,
      "log_path": "/dev/stdout",
      "log_level": "INFO",
      "io_backend": "asio_uring"
    }
```

### 3.3 StatefulSet - Center (Raft 3 节点)

> **资源作用**：StatefulSet 是为有状态服务设计的 Workload 控制器，保证 Pod 有**稳定的网络标识（主机名）**、**有序的启停**和**持久的存储**。每个 Pod 的名称固定为 `<statefulset-name>-<ordinal>`，从 0 开始递增，重建后不变。
>
> **选用逻辑**：Center 节点运行 **Raft 共识算法**，要求每个节点有唯一、可预测、稳定的网络标识，否则集群组建会失败。StatefulSet 精确满足这一需求：
>
> | 对比项 | StatefulSet ✅ | Deployment ❌ |
> |--------|---------------|--------------|
> | Pod 名称 | 固定 `center-0`、`center-1`、`center-2` | 随机后缀，重建即变 |
> | Pod 重建后身份 | DNS 名称不变 | 名称/IP 均变 |
> | 启动顺序 | 从 0 到 N-1 顺序创建 | 并发创建，不可控 |
> | Raft 适用性 | 配置可写死 `center-0.center-svc...`，天然匹配 | 无法预知 peer 地址 |
>
> 配合后面的 Headless Service，每个 Pod 获得固定 DNS 入口，Raft 配置中的三个 center 地址才能写死且永不变化。
>
> **LivenessProbe 选用 TCP socket**（而非 HTTP/Exec），因为 Thunder 使用二进制协议无 HTTP endpoint；exec 额外开销且可能和业务争锁。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: center
  namespace: thunder
spec:
  serviceName: center-svc
  replicas: 3
  selector:
    matchLabels:
      app: thunder-center
  template:
    metadata:
      labels:
        app: thunder-center
    spec:
      containers:
        - name: center
          image: thunder-node:latest
          command: ["./bin/Center_robot", "--conf=conf/Center.json"]
          workingDir: /thunder/deploy/Center
          ports:
            - containerPort: 27000
              name: inner
          volumeMounts:
            - name: config
              mountPath: /thunder/deploy/Center/conf/Center.json
              subPath: Center.json
            - name: plugins
              mountPath: /thunder/deploy/Center/plugins
          resources:
            limits:
              cpu: "3"
              memory: 2Gi
            requests:
              cpu: "1"
              memory: 256Mi
          livenessProbe:
            tcpSocket:
              port: 27000
            initialDelaySeconds: 15
            periodSeconds: 10
      volumes:
        - name: config
          configMap:
            name: center-config
        - name: plugins
          emptyDir: {}
```

### 3.4 Headless Service

> **资源作用**：Service 是 K8s 的服务发现与负载均衡抽象。其中 **Headless Service**（`clusterIP: None`）不创建 VIP，而是为每个后端 Pod 直接创建独立的 DNS A 记录，让客户端能够**精确解析到特定 Pod 的 IP**。
>
> **选用逻辑**：同样是 Service 资源，选择 Headless 还是 ClusterIP，取决于客户端需要**「找特定 Pod」**还是**「找任意一个可用 Pod」**。
>
> 这里选 Headless，因为 **Raft 节点之间需要直连到特定节点**（如 leader 选举的 RequestVote RPC），不能经过 VIP 随机转发 —— 必须精确找到 `center-0`、`center-1`、`center-2` 各自的身份。Headless Service 让 `center-0.center-svc.thunder` 精确解析到 `center-0` 的 IP，每个 Pod 的身份在 DNS 层就固定下来。
>
> 此外，Headless Service 是 StatefulSet 的强制伴侣 —— 没有它，StatefulSet 的 Pod 不会有稳定 DNS 名称，配置里也就无法写死 `center-0.center-svc.thunder:27000` 这类地址。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: center-svc
  namespace: thunder
spec:
  clusterIP: None
  selector:
    app: thunder-center
  ports:
    - name: inner
      port: 27000
```

### 3.5 Deployment - Hello

> **资源作用**：Deployment 是 K8s 最常用的 Workload 控制器，管理无状态 Pod 的声明式更新、扩缩容和自愈。Pod 名称随机、IP 可变、彼此完全等价，请求可打到任意副本。
>
> **选用逻辑**：Hello / Logic / Interface 都是**无状态服务**，不需要稳定网络标识，不需要有序启停，Deployment 是最轻量、最贴合的选择。
>
> | 对比项 | Deployment ✅ | StatefulSet ❌ |
> |--------|--------------|---------------|
> | 适用场景 | 无状态、可互换的 API 服务 | 有状态、需要稳定身份 |
> | 扩缩容 | 任意顺序，快速伸缩 | 按序号递增/递减 |
> | 滚动更新 | 支持，可回滚 | 支持但更保守 |
> | Pod 名称 | 随机（无需关注） | 固定但用不上 |
>
> Deployment 提供了声明式滚动更新、原地回滚、副本管理等全套能力，且复杂度最低。**不选 StatefulSet** 是因为这些服务不需要稳定网络标识，引入有序启动/终止反而增加不必要的约束。
>
> **LivenessProbe** 同样选用 TCP socket（port 27006），轻量可靠；Hello 启动较快，`initialDelaySeconds` 设为 10s 即可。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: thunder
spec:
  replicas: 2
  selector:
    matchLabels:
      app: thunder-hello
  template:
    metadata:
      labels:
        app: thunder-hello
    spec:
      containers:
        - name: hello
          image: thunder-node:latest
          command: ["./bin/Hello_robot", "--conf=conf/Hello.json"]
          workingDir: /thunder/deploy/HelloHttp
          ports:
            - containerPort: 27006
              name: access
          volumeMounts:
            - name: config
              mountPath: /thunder/deploy/HelloHttp/conf/Hello.json
              subPath: Hello.json
            - name: plugins
              mountPath: /thunder/deploy/HelloHttp/plugins
          resources:
            limits:
              cpu: "1.5"
              memory: 1Gi
            requests:
              cpu: "0.5"
              memory: 128Mi
          livenessProbe:
            tcpSocket:
              port: 27006
            initialDelaySeconds: 10
            periodSeconds: 10
      volumes:
        - name: config
          configMap:
            name: hello-config
        - name: plugins
          emptyDir: {}
```

### 3.6 Service

> **资源作用**：Service 为后端 Pod 提供**稳定的虚拟 IP 和 DNS 名称**，并自动将流量负载均衡到所有匹配的 Pod，屏蔽后端 Pod IP 变化的影响。ClusterIP 是默认类型，仅在集群内部可达。
>
> **选用逻辑**：同样是 Service 资源，Hello 这里选 ClusterIP，因为客户端不需要找特定 Pod —— 2 个 Hello 副本完全等价，请求打到任意一个都能正确处理。ClusterIP 提供的 VIP + 自动负载均衡正是这个场景需要的。
>
> | 对比项 | ClusterIP ✅（选它） | Headless ❌（不选） |
> |--------|-------------------|-------------------|
> | 行为 | 一个 VIP 随机转发到任意 Pod | DNS 返回所有 Pod IP，客户端自行选择或按名称查单个 |
> | 负载均衡 | ✅ kube-proxy 自动分发 | ❌ 需要客户端自己实现 |
> | Pod 身份 | 不关心，Pod 完全等价 | 需要精确区分每个 Pod |
> | 适用场景 | 无状态 API 服务 | StatefulSet + 有状态协议（Raft） |
>
> **与 Center 的 Headless Service 形成对比**：
> - Center → **定向访问**某个 Pod → Headless
> - Hello → **随机访问**任意 Pod → ClusterIP
>
> 两种 Service 解决相反的问题，选用逻辑取决于**客户端是否需要感知 Pod 身份**。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  namespace: thunder
spec:
  selector:
    app: thunder-hello
  ports:
    - port: 27006
      targetPort: 27006
  type: ClusterIP
```

---

## 四、资源底层原理

### 4.1 Namespace — Linux 内核隔离

K8s 的 Namespace 本质是对 **Linux Namespace 与 cgroups** 的封装：

- **进程隔离**：每个 Pod 运行在独立的 Linux **PID Namespace** 中，Pod 内进程只能看到自己的进程树（PID=1），无法感知宿主机或其他 Pod 的进程
- **网络隔离**：每个 Pod 拥有独立的 **Network Namespace**，包括自己的 lo、eth0、路由表、iptables 规则。Pod 间通过 veth pair + CNI 插件（如 Calico/Flannel）通信
- **文件系统隔离**：通过 **Mount Namespace** 实现，Pod 的 rootfs 来自镜像层叠（OverlayFS），ConfigMap/Secret 通过 tmpfs 挂载进来
- **资源限制**：cgroups 控制 CPU、内存、磁盘 IO 的硬限制（`resources.limits`）和软保证（`resources.requests`）

> **kubelet 的角色**：kubelet 在创建 Pod 时调用容器运行时（containerd/CRI-O），运行时为每个容器逐一设置上述 Namespace，最后通过 `cri` 接口返回 Pod 状态。

### 4.2 ConfigMap — etcd → API Server → kubelet → tmpfs

ConfigMap 的数据流是一条完整的事件驱动链路：

```
kubectl apply -f → API Server → etcd（持久化）
                            ↓ Watch
kubelet 监听 Pod 变更 → 检查引用的 ConfigMap
                            ↓
通过 CRI 将 ConfigMap 以 tmpfs 挂载到容器内指定路径
                            ↓
Pod 内进程读取 /path/to/file（实际是 tmpfs 内存文件系统）
```

- **存储**：ConfigMap 数据存在 etcd 中（默认 1MB 上限），API Server 提供 REST 接口存取
- **挂载方式**：`subPath` 将 ConfigMap 的某个 Key 挂载为单个文件，不覆盖目录下其他内容
- **热更新**：不重启 Pod，已挂载的文件会在**分钟级**内自动更新（kubelet 定期 sync），但使用 `subPath` 时不会热更新 —— 这是设计上在 `subPath` 与自动刷新之间的取舍
- **为什么不直接写环境变量**：环境变量不会自动更新，且不适合 JSON/YAML 等结构化配置

### 4.3 StatefulSet — 控制器工作模式

StatefulSet 的核心是 **Pod 身份与序号绑定**，由 `StatefulSet Controller` 实现：

```
用户声明:
  replicas: 3, serviceName: "center-svc"

控制器创建:
  center-0, center-1, center-2  （按序创建，每个 ready 再建下一个）

内部机制:
  ① Pod 名称 = <statefulset-name>-<ordinal>
  ② 每个 Pod 的 hostname / subdomain 设为其名称
  ③ 关联 Headless Service 后 DNS 记录自动生成
  ④ Pod 重建时，序号不变，重新获得同名 Pod
```

- **控制器实现**：StatefulSet Controller 是 `kube-controller-manager` 的一个 goroutine，通过 **Informer 机制**（List + Watch）监听 Pod 和 StatefulSet 的变更事件，维护期望副本数
- **Pod 身份字符串**：Pod 的 `metadata.name` 固定，K8s 不会重用已删除 Pod 的名称（Deployment 的 Pod 名带随机后缀就是为了避免冲突）
- **有序启停**：控制器的 `sync` 方法维护一个拓扑顺序数组（ordinal 0..N-1），扩容按序创建，缩容逆序删除，确保 Raft 等一致性协议不被并发打乱
- **存储绑定**：`volumeClaimTemplates` 为每个 ordinal 创建独立的 PVC，Pod 重建后自动重挂同名 PVC（本例未启用，但 StatefulSet 原生支持）

### 4.4 Headless Service — DNS 解析链路

Headless Service 的底层实现涉及 **CoreDNS 与 Endpoints Controller** 的协作：

```
创建 Headless Service (clusterIP: None)
        ↓
Endpoints Controller 监听 Pod 变化
        ↓ 更新 Endpoints / EndpointSlice 对象
CoreDNS 读取 EndpointSlice
        ↓ 为每个 Ready Pod 生成 A/AAAA 记录
客户端查询 center-0.center-svc.thunder
        ↓ DNS 查询到达 CoreDNS Pod
直接返回 center-0 的 Pod IP（非 VIP）
```

- **DNS 记录格式**：`<pod-name>.<service-name>.<namespace>.svc.cluster.local`，对应本例的 `center-0.center-svc.thunder:27000`
- **与普通 Service 的区别**：
  | Service 类型 | DNS 查询结果 | 返回策略 |
  |-------------|-------------|---------|
  | ClusterIP | `hello-svc` → 一个 VIP | 随机返回一个 ClusterIP |
  | Headless | `center-0.center-svc` → 具体 Pod IP | 返回所有 Ready Pod 的 IP 列表（或按名称查单个） |
- **SRV 记录**：Headless Service 还支持 SRV 记录查询，包含端口信息，适合需要自动发现端口的场景
- **kube-proxy 不参与**：Headless Service 不在 kube-proxy 的 iptables/IPVS 规则中创建转发规则

### 4.5 Deployment — ReplicaSet + 滚动更新

Deployment 的实现依赖于 **ReplicaSet 作为中间层**：

```
Deployment (声明 replicas=2, strategy=RollingUpdate)
       ↓ 管理
ReplicaSet-v1 (replicas=2)  →   Pod-xxx, Pod-yyy
       ↓ 滚动更新（如更新镜像版本）
ReplicaSet-v1 (replicas=0)  →   （缩容到 0）
ReplicaSet-v2 (replicas=2)  →   Pod-aaa, Pod-bbb（新版本）
```

- **底层机制**：
  1. Deployment Controller 也是 `kube-controller-manager` 的 goroutine
  2. 每次 spec 变更（镜像、环境变量等），控制器**创建新的 ReplicaSet**，按策略调整新旧 ReplicaSet 的副本数
  3. `RollingUpdate` 策略：逐个替换旧 Pod（可配置 `maxSurge` / `maxUnavailable`），保证服务不中断
  4. **回滚**：旧 ReplicaSet 保留（`revisionHistoryLimit` 控制数量），回滚时直接将旧 ReplicaSet 的副本数从 0 恢复到期望值
- **Pod 随机名称后缀**：ReplicaSet 创建 Pod 时，名称格式为 `<replicaset-name>-<random-suffix>`（如 `hello-6f9d4c8b7b-abcde`），确保相同模板每次部署产生不同名称
- **就绪检测配合**：`readinessProbe` 通过后才加入 Service 端点，确保滚动更新期间流量不切到未就绪的新 Pod

### 4.6 Service (ClusterIP) — kube-proxy 与 iptables/IPVS

ClusterIP Service 的底层数据平面由 **kube-proxy** 维护，主要有三种实现模式：

| 模式 | 原理 | 性能 | 适用场景 |
|------|------|------|---------|
| **iptables** | 为每个 Service 的每个端口创建 iptables `DNAT` 规则链，通过 `statistic` 模块做随机负载均衡 | 规则量大时更新慢 | 默认模式，集群规模 < 5000 |
| **IPVS** | 利用 Linux 内核的 IP Virtual Server 模块，基于哈希表做负载均衡 | 高吞吐，规则更新快 | 大规模集群 > 5000 Service |
| **userspace** | 用户态代理，性能差，已弃用 | 低 | 仅历史遗留 |

数据流（以 iptables 模式为例）：

```
Pod A 请求 hello-svc:27006
        ↓ DNS 解析到 ClusterIP (10.96.x.x)
        ↓ 到达 Pod A 的 eth0
        ↓ Netfilter PREROUTING 链
        ↓ iptables KUBE-SERVICES 链匹配 ClusterIP:Port
        ↓ DNAT 随机替换为目标 Pod IP （随机选择 hello-xxx 或 hello-yyy）
        ↓ 路由到目标 Pod 的 eth0
        ↓ hello 进程在 27006 端口收到请求
```

- **ClusterIP 的本质**：一个虚拟 IP，**没有对应任何网络接口**。它只存在于 iptables/IPVS 规则中，数据包经过 `DNAT` 后才真正发送到 Pod
- **Endpoints Controller**：监听 Pod 变化，实时更新 Endpoints 对象（Pod IP:Port 列表），kube-proxy 据此更新 iptables/IPVS 规则
- **为什么不直接连 Pod IP**：Pod IP 在重建后变化，客户端需要 Service 做稳定抽象；此外 Service 还提供多副本负载均衡

---

## 五、Docker Compose 迁移指南

> 从 Docker Compose 迁移到 K8s 时，各层配置的对应关系对照表。

| 项目 | Docker Compose | K8s |
|------|---------------|-----|
| 网络 | host 模式 | ClusterIP + DNS |
| 通信地址 | 127.0.0.1 | *.svc.cluster.local |
| 进程模式 | daemon + tail -f | 前台 exec |
| 日志 | 文件 | /dev/stdout |
| cpu_affinity | true | false (K8s CPU Manager) |
| 配置 | 文件挂载 | ConfigMap |
| 顺序依赖 | depends_on | initContainers + probe |

K8s 中 center 地址改为 DNS 名称：

```
center-0.center-svc.thunder:27000
center-1.center-svc.thunder:27000
center-2.center-svc.thunder:27000
```

---

## 六、启动顺序

```bash
# 基础设施
kubectl apply -f thunder-namespace.yaml
kubectl apply -f thunder-config.yaml

# Center (Raft)
kubectl apply -f thunder-statefulset-center.yaml
kubectl rollout status -n thunder statefulset/center

# Logic
kubectl apply -f thunder-deployment-logic.yaml
kubectl rollout status -n thunder deployment/logic

# 接入层
kubectl apply -f thunder-deployment-interface.yaml
kubectl apply -f thunder-deployment-hello.yaml

# Service + Ingress
kubectl apply -f thunder-svc.yaml
kubectl apply -f thunder-ingress.yaml
```

---

## 七、验证

```bash
kubectl get all -n thunder
kubectl logs -n thunder -l app=thunder-center
# 测试 Hello 服务
kubectl run -n thunder --rm -it test --image=curlimages/curl -- curl http://hello-svc:27006/hello/hello
```
