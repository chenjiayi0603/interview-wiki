# IM 项目 MongoDB 分片集群图

IM 场景下的 MongoDB 分片集群，2 个 mongos、1 个 configsvr 副本集、2 个分片。

## 组件说明

- **客户端**：通过 mongos 访问集群。
- **mongos**：
  - 10.3.0.77:27017
  - 10.3.0.78:27017
- **configsvr**：
  - PRIMARY 10.3.0.79:27017
  - SECONDARY 10.3.0.80:27017、10.3.0.81:27017
- **分片**：
  - **shard01**：PRIMARY 10.3.0.71:27017、SECONDARY 10.3.0.72:27017、ARBITER 10.3.0.73:27017
  - **shard02**：PRIMARY 10.3.0.74:27017、SECONDARY 10.3.0.75:27017、ARBITER 10.3.0.76:27017

[src: raw/ingested/3项目/分布式IM-雷漫/架构成本-架构图说明.md]