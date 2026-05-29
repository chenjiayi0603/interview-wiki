# K3s与K8s学习曲线与运维复杂度对比

## 学习曲线对比

```
┌─────────────────────────────────────────────────────────────────┐
│                       学习曲线对比                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  难度                                                           │
│   ▲                                                            │
│   │         ╭──────────────╮                                    │
│   │         │     K8s     │                                    │
│   │         │  陡峭的学习  │                                    │
│   │         │    曲线      │                                    │
│   │         ╰──────┬───────╯                                    │
│   │          ╭─────┴──────╮                                     │
│   │          │   K3s     │                                    │
│   │          │  较平缓    │                                    │
│   │          ╰─────┬──────╯                                     │
│   └──────────────────────────────────► 时间                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 学习难度分解

| 维度 | K3s | K8s | 说明 |
|------|-----|-----|------|
| **安装部署** | ⭐ 简单 | ⭐⭐⭐ 复杂 | K3s 一键安装 |
| **概念理解** | ⭐ 相同 | ⭐ 相同 | 核心概念一致 |
| **配置管理** | ⭐⭐ 中等 | ⭐⭐⭐ 复杂 | K3s 简化配置 |
| **故障排查** | ⭐⭐ 中等 | ⭐⭐⭐ 复杂 | 组件少更易排查 |
| **运维操作** | ⭐ 简单 | ⭐⭐⭐ 复杂 | 升级/备份更复杂 |
| **调试技能** | ⭐⭐ 中等 | ⭐⭐⭐ 复杂 | 需要更多工具 |

## 安装复杂度对比

**K3s 安装：**
```bash
# 单节点安装 - 一行命令
curl -sfL https://get.k3s.io | sh -

# 查看状态
kubectl get nodes
kubectl get pods -A

# 卸载
/usr/local/bin/k3s-uninstall.sh
```

**K8s 安装：**
```bash
# 方式 1: kubeadm（推荐学习）
# 1. 安装容器运行时
# 2. 安装 kubeadm, kubelet, kubectl
# 3. 初始化集群
kubeadm init --pod-network-cidr=10.244.0.0/16

# 4. 配置 kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# 5. 安装网络插件
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 方式 2: k3s + Rancher
# 方式 3: 云托管服务（EKS/GKE/AKS）
# 方式 4: Kubespray（ Ansible 自动化）
```

## 运维复杂度对比

| 运维任务 | K3s | K8s | K3s 优势说明 |
|----------|-----|-----|--------------|
| **升级集群** | ⭐ 自动/手动 | ⭐⭐⭐ 复杂 | K3s 自动升级 |
| **备份恢复** | ⭐ 简单 | ⭐⭐ 复杂 | SQLite 备份简单 |
| **证书管理** | ⭐ 自动 | ⭐⭐ 需关注 | K3s 自动轮换 |
| **日志收集** | ⭐⭐ 中等 | ⭐⭐⭐ 复杂 | 组件少日志少 |
| **监控配置** | ⭐ 内置 | ⭐ 需安装 | K3s 内置 Metrics |
| **故障恢复** | ⭐⭐ 中等 | ⭐⭐⭐ 复杂 | 组件故障隔离 |

## 升级对比

**K3s 升级：**
```bash
# 自动升级
curl -sfL https://get.k3s.io | K3S_CHANNEL=stable sh -

# 手动升级到特定版本
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.28.4+k3s1 sh -

# 查看可用版本
k3s --version
```

**K8s 升级：**
```bash
# kubeadm 升级流程（1.27 -> 1.28）
# 1. 升级控制平面
kubectl drain control-plane-node --ignore-daemonsets
kubeadm upgrade plan v1.28.0
kubeadm upgrade apply v1.28.0

# 2. 升级 kubelet
apt-get upgrade -y kubelet=1.28.0-*

# 3. 重启 kubelet
systemctl restart kubelet

# 4. 解锁节点
kubectl uncordon control-plane-node

# 5. 升级工作节点
kubectl drain worker-node --ignore-daemonsets
# 在工作节点执行相同的升级步骤
```

[src: raw/ingested/3项目/服务网格-即构/k3s与k8s技术对比分析-八、学习曲线与运维复杂度.md]