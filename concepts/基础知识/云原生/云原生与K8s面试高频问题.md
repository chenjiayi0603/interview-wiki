# 云原生与K8s面试高频问题

## Q1: K8s和Docker的区别？
**参考答案**：
- Docker：容器运行时，单机容器管理
- K8s：容器编排平台，跨主机集群管理
- Docker是K8s的底层实现之一（CRI）

## Q2: Pod和Container的区别？
**参考答案**：
- Pod是K8s最小调度单元
- Pod可以包含多个容器（sidecar模式）
- Pod内容器共享网络、存储、UTS命名空间
- 容器是实际运行的进程

## Q3: Service类型有哪些？
**参考答案**：
- **ClusterIP**：集群内部访问
- **NodePort**：通过节点端口访问
- **LoadBalancer**：云厂商负载均衡
- **ExternalName**：CNAME映射
- **Headless**：无头服务，自主选择Pod

## Q4: K8s如何实现服务发现？
**参考答案**：
- DNS：CoreDNS解析Service名
- 环境变量：kubelet注入SERVICE_HOST
- DNS更推荐

## Q5: Deployment更新策略？
**参考答案**：
- **RollingUpdate**：滚动更新（默认）
  - maxSurge：最多超出副本数
  - maxUnavailable：最多不可用副本数
- **Recreate**：重建，更新前删除所有Pod

## Q6: Istio Sidecar注入原理？
**参考答案**：
- 自动注入：通过MutatingAdmissionWebhook
- 手动注入：istioctl kube-inject
- 注入后：init容器设置iptables规则
- 流量拦截：所有流量经过Sidecar

## Q7: Envoy如何做熔断？
**参考答案**：
```yaml
circuit_breakers:
  thresholds:
  - max_connections: 100
    max_pending_requests: 100
    max_requests: 100
    max_retries: 10
```
- 连接数熔断
- 请求数熔断
- 熔断后快速失败

## Q8: gRPC vs HTTP/REST？
**参考答案**：
| 特性 | gRPC | HTTP/REST |
|------|------|----------|
| 协议 | HTTP/2 | HTTP/1.1/2 |
| 序列化 | Protobuf | JSON |
| 代码生成 | 自动 | OpenAPI |
| 流式 | 双向流 | 单向 |
| 性能 | 高 | 中 |

## Q9: 如何排查K8s Pod问题？
**参考答案**：
1. `kubectl describe pod` 查看事件
2. `kubectl logs` 查看日志
3. `kubectl exec` 进入容器
4. `kubectl get events` 查看集群事件
5. `kubectl top` 查看资源使用

## Q10: HPA工作原理？
**参考答案**：
- HorizontalPodAutoscaler
- 基于CPU/内存或自定义指标
- 周期性检查当前指标
- 计算需要副本数：ceil(current * desired / current)
- 更新Deployment副本数

[src: raw/ingested/2技术/虚拟化/云原生与K8s-七、面试高频问题.md]