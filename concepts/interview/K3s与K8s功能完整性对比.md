# K3s 与 K8s 功能完整性对比

## 5.1 核心功能对比

| 功能模块 | K3s | K8s | 说明 |
|----------|-----|-----|------|
| **容器编排** | ✅ 完全支持 | ✅ 完全支持 | 核心功能一致 |
| **Pod 管理** | ✅ 完全支持 | ✅ 完全支持 | API 完全兼容 |
| **Service** | ✅ 完全支持 | ✅ 完全支持 | ClusterIP/NodePort/LoadBalancer |
| **Ingress** | ✅ Traefik（内置） | ✅ Nginx（可选） | K3s 内置 Traefik |
| **ConfigMap/Secret** | ✅ 完全支持 | ✅ 完全支持 | 配置管理 |
| **PV/PVC** | ✅ 完全支持 | ✅ 完全支持 | 存储管理 |
| **RBAC** | ✅ 完全支持 | ✅ 完全支持 | 权限控制 |
| **NetworkPolicy** | ✅ 完全支持 | ✅ 完全支持 | 网络策略 |
| **HPA/VPA** | ✅ 完全支持 | ✅ 完全支持 | 水平/垂直扩缩容 |
| **CRD** | ✅ 完全支持 | ✅ 完全支持 | 自定义资源 |
| **Webhook** | ✅ 完全支持 | ✅ 完全支持 | 准入控制 |
| **Metrics Server** | ✅ 内置 | ❌ 需安装 | K3s 开箱即用 |

## 5.2 移除/简化的功能

**K3s 移除的功能：**

| 移除的功能 | 原因 | 替代方案 |
|------------|------|----------|
| Cloud Provider 集成 | 减少复杂性 | 使用 in-tree 或 CCM |
| 某些 alpha API | 稳定性考虑 | 保持核心 API 稳定 |
| Dockershim | 容器运行时统一 | 使用 containerd |
| 某些 admission webhook | 减少资源占用 | 可按需启用 |

## 5.3 K3s 特有功能

```bash
# K3s 特有的功能标记
k3s server --help

# 可禁用的内置组件
--disable "traefik"           # 禁用 Traefik Ingress
--disable "servicelb"        # 禁用 Klipper LoadBalancer
--disable "local-storage"   # 禁用本地存储
--disable "metrics-server"  # 禁用 Metrics Server
--disable "coredns"         # 禁用 CoreDNS

# 启用特定功能
--enable "secrets-encryption"  # 静态加密 Secrets
--cluster-cidr "10.42.0.0/16"   # Pod CIDR
--service-cidr "10.43.0.0/16"   # Service CIDR
```

[src: raw/ingested/3项目/服务网格-即构/k3s与k8s技术对比分析-五、功能完整性对比.md]