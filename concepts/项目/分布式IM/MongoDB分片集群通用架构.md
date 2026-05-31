# MongoDB 分片集群分布图（通用）

典型 MongoDB 分片集群架构，包含客户端、mongos、configsvr 与多个分片。

## 组件说明

- **客户端/应用服务器**：顶部多个 server，通过虚线连接 mongos。
- **mongos（查询路由）**：端口 20000，接收读写请求，并向 configsvr 拉取元数据。
- **configsvr（配置服务器副本集）**：
  - PRIMARY 21001、SECONDARY 21002、SECONDARY 21003
  - 存储分片信息、chunk 分布等元数据。
- **mongodb 数据存储（分片副本集）**：
  - **shard01**：PRIMARY 22001、SECONDARY 22002、ARBITER 22003
  - **shard02**：PRIMARY 22004、SECONDARY 22005、ARBITER 22006
  - **shard03**：PRIMARY 22007、SECONDARY 22008、ARBITER 22009

说明：22001 为该节点数据端口，可配置拒绝非 DBA 访问。

[src: raw/ingested/3项目/分布式IM-雷漫/架构成本-架构图说明.md]