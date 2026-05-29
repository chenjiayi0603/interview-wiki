# IM AWS MySQL 高可用架构

基于 **MySQL Group Replication (MGR)** 的高可用集群，通过 VIP 对外提供统一入口。

## 组件说明

- **VIP**：172.19.1.240:3306
  - 仅挂载在主节点所在机器，主切换时 VIP 随之漂移。
- **MGR 三节点**：
  - 172.19.1.163:3306
  - 172.19.1.38:3306
  - 172.19.1.193:3306
- **目录配置**：
  - basedir → /data/mysql
  - logdir → /data/mysql/db3306
  - datadir → /data/mysql/data/db3306

[src: raw/ingested/3项目/分布式IM-雷漫/架构成本-架构图说明.md]