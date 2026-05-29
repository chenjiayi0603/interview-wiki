# K3s 与 K8s 多节点扩展能力

## 扩展性对比

```
┌─────────────────────────────────────────────────────────────────┐
│                       扩展能力对比                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  节点数量                                                        │
│                                                                 │
│  K3s:  0 ─────●─────────────────────────── 100+                 │
│        单节点                           推荐上限                │
│                                                                 │
│  K8s:  0 ──────────────●──────────────────────────────────●    │
│        最小规模                常规使用                    5000+│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 扩展能力详细数据

| 指标 | K3s | K8s | 说明 |
|------|-----|-----|------|
| **推荐最大节点数** | 100 | 5000+ | 官方推荐 |
| **支持最大 Pod 数/节点** | 100-250 | 100-250 | 取决于 CNI |
| **集群启动时间（10节点）** | 1-2 分钟 | 5-10 分钟 | - |
| **etcd 性能** | SQLite（单节点） | etcd 集群 | 高可用差异 |
| **网络扩展性** | 良好 | 优秀 | CNI 依赖 |

## K3s 高可用部署

```bash
# K3s 高可用模式（嵌入式 etcd）
# 节点 1 - 初始化集群
curl -sfL https://get.k3s.io | sh -s - server \
    --cluster-init \
    --token=<CLUSTER_TOKEN> \
    --tls-san=<LOAD_BALANCER_IP>

# 节点 2, 3 - 加入集群
curl -sfL https://get.k3s.io | sh -s - server \
    --server https://<CLUSTER_IP>:6443 \
    --token=<CLUSTER_TOKEN> \
    --tls-san=<LOAD_BALANCER_IP>
```

## K8s 高可用部署

```bash
# K8s 高可用部署（使用 kubeadm）
# 初始化第一个控制平面节点
kubeadm init --control-plane-endpoint "lb.k8s.local:6443" \
    --upload-certs \
    --pod-network-cidr=10.244.0.0/16

# 加入其他控制平面节点
kubeadm join control-plane <CONTROL_PLANE_ENDPOINT> \
    --control-plane --certificate-key <CERT_KEY> \
    --token <TOKEN>

# 加入工作节点
kubeadm join <CONTROL_PLANE_ENDPOINT>:6443 \
    --token <TOKEN> \
    --discovery-token-ca-cert-hash <HASH>
```

[src: raw/ingested/3项目/服务网格-即构/k3s与k8s技术对比分析-七、多节点扩展能力.md]