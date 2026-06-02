# 容器基础与 Docker

> Docker 架构、Dockerfile 最佳实践、容器网络、资源限制。

---

## 一、Docker 核心概念

| 概念 | 说明 |
|------|------|
| **镜像（Image）** | 只读模板，包含运行环境 |
| **容器（Container）** | 镜像的运行实例，可读写 |
| **仓库（Registry）** | 存储分发镜像（Docker Hub / ACR） |
| **Dockerfile** | 镜像构建脚本 |
| **Volume** | 持久化数据存储 |

### 架构

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Client  │  │  Client  │  │  Client  │
│  docker  │  │  docker  │  │  docker  │
└─────┬────┘  └─────┬────┘  └─────┬────┘
      └─────────────┼──────────────┘
                    ▼
           ┌────────────────┐
           │  Docker Daemon │
           │  (dockerd)     │
           └────────────────┘
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Container│ │ Container│ │ Container│
   │  (runc)  │ │  (runc)  │ │  (runc)  │
   └──────────┘ └──────────┘ └──────────┘
```

---

## 二、Dockerfile 最佳实践

### 2.1 多阶段构建（减小镜像体积）

```dockerfile
# Node.js 示例
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
RUN addgroup -g 1001 -S appuser && adduser -S appuser -u 1001
COPY --from=builder --chown=appuser:appuser /app/dist ./dist
COPY --from=builder --chown=appuser:appuser /app/node_modules ./node_modules
USER appuser
EXPOSE 8080
HEALTHCHECK CMD wget -q --spider http://localhost:8080/health || exit 1
CMD ["node", "dist/main.js"]
```

### 2.2 Go 应用（极致精简，scratch 镜像）

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server .

FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### 2.3 Java/Spring Boot

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -g 1001 -S appuser && adduser -S appuser -u 1001
COPY --from=builder --chown=appuser:appuser /app/build/libs/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

---

## 三、容器网络

### 3.1 网络模式

| 模式 | 说明 | 使用场景 |
|------|------|---------|
| `bridge` | 默认，隔离的虚拟网络 | 单机多容器通信 |
| `host` | 共享宿主机网络栈 | 性能敏感场景 |
| `none` | 无网络 | 安全隔离 |
| `container:name` | 共享另一容器的网络 | sidecar 模式 |

```bash
docker network create myapp-net
docker run --network=myapp-net --name api ...
```

### 3.2 K8s CNI 插件

集群层面，CNI 负责为 Pod 分配 IP 并互联：

| 插件 | 特点 |
|------|------|
| Flannel | 简单，VXLAN 覆盖网络 |
| Calico | 高性能，支持网络策略 |
| Cilium | eBPF 驱动，最现代 |

---

## 四、资源限制

```yaml
# Docker Compose
services:
  api:
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

```yaml
# K8s Pod
spec:
  containers:
  - resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

---

## 五、安全最佳实践

| 实践 | 说明 |
|-----|------|
| 不使用 root 用户 | Dockerfile 中用 `USER appuser` |
| 使用 Alpine/scratch 基础镜像 | 减少攻击面 |
| 敏感信息用 Secrets / 环境变量 | 不硬编码到镜像中 |
| 定期更新基础镜像 | `docker pull` 获取安全补丁 |
| 限制容器资源 | 防止单个容器耗尽主机资源 |
| 使用 `.dockerignore` | 避免将 `.git`、`node_modules` 打入镜像 |
| 镜像扫描 | 使用 `docker scout` 或 Trivy 扫描漏洞 |

---

## 六、面试高频追问

### Q1: Docker vs VM 区别？
| 对比 | 容器 | 虚拟机 |
|:----:|:----:|:------:|
| 隔离级别 | 进程级（namespace） | 硬件级（Hypervisor） |
| 内核 | 共享宿主机内核 | 各自独立内核 |
| 启动时间 | 毫秒级 | 秒级至分钟级 |
| 镜像大小 | MB 级（Alpine ~5MB） | GB 级 |
| 性能开销 | 极低（≈ 原生） | 较高（虚拟化层损耗） |

### Q2: 多阶段构建解决了什么问题？
- 单阶段：构建工具和运行环境混合，镜像动辄 GB 级
- 多阶段：**第一阶段**（构建环境）安装编译器/依赖，**第二阶段**（运行环境）只拷贝产物
- Go 应用可从 800MB 压缩到 5MB（scratch 镜像）

### Q3: Docker 的核心隔离技术？
- **Namespace**：PID（进程隔离）、Network（网络栈）、Mount（文件系统）、UTS（主机名）、IPC、User
- **Cgroups**：资源限制（CPU/内存/磁盘 IO）、优先级、统计
- **UnionFS**：分层镜像（OverlayFS），写时复制

### Q4: host 网络模式什么时候用？
- 性能敏感场景（减少网络桥接开销）
- 需要直接宿主机端口（如 DPDK 应用）
- **代价**：容器不再有独立网络栈，端口冲突风险
