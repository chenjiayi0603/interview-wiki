# IM AWS MongoDB 分片集群

AWS 环境下 IM 的 MongoDB 分片集群，含 mongos、configs、两个分片及目录配置。

## 组件说明

- **mongos**：
  - 172.19.1.193:27017
  - 172.19.1.38:27017
- **configs（配置服务器）**：
  - 172.19.1.193:21000
  - 172.19.1.163:21000
  - 172.19.1.38:21000
- **Cluster 分片**：
  - **im-shard01**：172.19.1.193:22001、172.19.1.38:22002
  - **im-shard02**：172.19.1.38:22001、172.19.1.193:22002
- **目录配置**：
  - mongodir → /data/mongo
  - logdir → /data/mongo/node{{port}}
  - datadir → /data/mongo/data/node{{port}}

[src: raw/ingested/3项目/分布式IM-雷漫/架构成本-架构图说明.md]