# IM AWS Codis 架构

Codis 分布式缓存架构：代理层、控制台、Redis 集群及目录映射。

## 组件说明

- **codis-proxy**：
  - 172.19.1.38:19000
  - 172.19.1.193:19000
- **codis 控制台**：
  - codis-fe：172.19.1.38:8090
  - codis-dashboard：172.19.1.38:18080
- **Cluster**：
  - **group1**：172.19.1.38:6379、172.19.1.193:6380
  - **group2**：172.19.1.193:6379、172.19.1.38:6380
  - **sentinels**：172.19.1.38:26379、172.19.1.193:26379、172.19.1.163:26379
- **目录映射**：
  - codisdir → /data/codis
  - confdir → /data/codis/conf
  - logdir → /data/codis/logs
  - datadir → /data/codis/data

[src: raw/ingested/3项目/分布式IM-雷漫/架构成本-架构图说明.md]