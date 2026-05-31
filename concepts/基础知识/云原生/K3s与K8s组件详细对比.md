# K3s 与 K8s 组件详细对比

> 本文档基于即构科技服务网格项目中的技术对比分析，详细对比 K3s 与 K8s 在核心组件上的差异。

## 6.1 API Server 对比

| 特性 | K3s | K8s |
|------|-----|-----|
| **API 兼容性** | 100% K8s API | 标准 K8s API |
| **版本** | 跟随 K8s 主版本 | 官方主版本 |
| **认证授权** | ✅ 完全支持 | ✅ 完全支持 |
| **准入控制** | ✅ 大部分支持 | ✅ 完全支持 |
| **API 聚合** | ✅ 支持 | ✅ 支持 |
| **存储后端** | SQLite / etcd | etcd |
| **etcd 模式** | 单节点嵌入式 / 外部 | 外部集群 |

## 6.2 Scheduler 对比

| 特性 | K3s | K8s |
|------|-----|-----|
| **调度策略** | ✅ 标准调度器 | ✅ 标准调度器 |
| **亲和性/反亲和性** | ✅ 支持 | ✅ 支持 |
| **污点和容忍** | ✅ 支持 | ✅ 支持 |
| **优先级调度** | ✅ 支持 | ✅ 支持 |
| **自定义调度器** | ✅ 支持 | ✅ 支持 |

## 6.3 Ingress 对比

### 6.3.1 Traefik vs Nginx Ingress

```
┌─────────────────────────────────────────────────────────────────┐
│                        Ingress 对比                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────┐    ┌─────────────────────────┐    │
│  │        K3s              │    │         K8s              │    │
│  │    Traefik Ingress      │    │   Nginx Ingress         │    │
│  │                         │    │                         │    │
│  │  ┌───────────────────┐  │    │  ┌───────────────────┐  │    │
│  │  │  Traefik Pod      │  │    │  │  Ingress Pod      │  │    │
│  │  │  ┌─────────────┐  │  │    │  │  ┌─────────────┐  │  │    │
│  │  │  │  Traefik    │  │  │    │  │  │  Nginx      │  │  │    │
│  │  │  │  Ingress    │  │  │    │  │  │  Controller │  │  │    │
│  │  │  └─────────────┘  │  │    │  │  └─────────────┘  │  │    │
│  │  └───────────────────┘  │    │  │  ┌─────────────┐  │  │    │
│  │                         │    │  │  │  ConfigMap  │  │  │    │
│  │  自动发现: ✅           │    │  │  │  (模板配置) │  │  │    │
│  │  CRD: ✅                │    │  │  └─────────────┘  │  │    │
│  │  Let's Encrypt: ✅      │    │  │                   │  │    │
│  │                         │    │  │  模板驱动: ✅     │  │    │
│  │  配置方式:              │    │  │  ConfigMap: ✅    │  │    │
│  │  - Ingress 注解         │    │  │  Annotations: ✅ │  │    │
│  │  - CRD (IngressRoute)   │    │  │                   │  │    │
│  └─────────────────────────┘    │  └───────────────────┘    │
│                                  │                             │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3.2 详细功能对比

| 特性 | Traefik (K3s) | Nginx Ingress |
|------|---------------|---------------|
| **配置来源** | 动态发现 / CRD | ConfigMap / Annotations |
| **CRD 支持** | ✅ IngressRoute | ❌ 标准 Ingress |
| **自动发现 Service** | ✅ 支持 | ❌ 需手动配置 |
| **Let's Encrypt** | ✅ 内置集成 | ❌ 需 cert-manager |
| **TCP/UDP 路由** | ✅ 原生支持 | ✅ 需要额外配置 |
| **WebSocket** | ✅ 支持 | ✅ 支持 |
| **gRPC** | ✅ 原生支持 | ⚠️ 需配置 |
| **速率限制** | ✅ 内置 | ⚠️ 需插件 |
| **Circuit Breaker** | ✅ 支持 | ❌ 需外部 |
| **流量镜像** | ✅ 支持 | ✅ 支持 |
| **性能** | 中等 | 高 |
| **学习曲线** | 较陡 | 较平缓 |
| **文档完善度** | 良好 | 非常完善 |

### 6.3.3 配置示例对比

**Traefik IngressRoute（K3s）：**
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
  namespace: default
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`app.example.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: api-service
          port: 8080
      middlewares:
        - name: rate-limit
        - name: auth
  tls:
    secretName: app-tls
```

**Nginx Ingress（K8s）：**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
```

## 6.4 存储组件对比

| 存储特性 | K3s | K8s |
|----------|-----|-----|
| **默认存储** | SQLite | 无内置 |
| **分布式存储** | ✅ 可选 etcd | ✅ etcd |
| **CSI 支持** | ✅ 完整支持 | ✅ 完整支持 |
| **Local PV** | ✅ 支持 | ✅ 支持 |
| **StorageClass** | ✅ 默认 local-path | ❌ 需配置 |
| **动态存储分配** | ✅ 内置 | ⚠️ 需安装组件 |

## 6.5 网络组件对比

| 网络特性 | K3s | K8s |
|----------|-----|-----|
| **CNI 插件** | 内置/可替换 | 需自行安装 |
| **默认 CNI** | k3s-cni (bridge) | 无 |
| **DNS** | 内置 CoreDNS | 需安装 |
| **NetworkPolicy** | ✅ 支持 | ✅ 支持 |
| **负载均衡** | Klipper (L2) | 需云 LB 或 MetalLB |

[src: raw/ingested/3项目/服务网格-即构/k3s与k8s技术对比分析-六、组件详细对比.md]