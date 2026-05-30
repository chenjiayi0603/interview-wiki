# FastDFS 逻辑架构图

基于 FastDFS 的分布式文件存储逻辑架构，含网关、应用层、Tracker、Storage 及 MySQL/Redis。

## 架构层次

1. **网关 (Gateway)**：统一入口，将请求转发到应用服务器。
2. **应用服务器**：应用服务器 1、2、…、N，处理业务并访问 Tracker、MySQL、Redis。
3. **Tracker 层 (FastDFS + Nginx)**：
   - 多台 Tracker，管理文件元数据与存储节点；
   - 与 MySQL 集群、Redis 集群有连接（配置/元数据）；
   - 应用通过 Tracker 访问 FastDFS。
4. **数据层**：
   - **MySQL 集群**：关系型数据持久化；
   - **Redis 集群**：缓存与高并发访问。
5. **FastDFS Storage 层**：
   - group1、group2、…、groupN；
   - 每台 Storage：FastDFS Storage + Nginx + FastDHT + BDB；
   - Tracker 与各 group 内 Storage 通信，统一管理存储。
6. **监控**：独立监控组件，对系统进行监控与告警。

[src: raw/ingested/3项目/分布式IM-雷漫/架构成本-架构图说明.md]