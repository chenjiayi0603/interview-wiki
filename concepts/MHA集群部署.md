# MHA 集群部署

**MHA**（Master High Availability Manager and tools for MySQL）是 DeNA 公司开发的 MySQL 高可用解决方案，用于**自动化主从故障转移**，减少人工干预和业务中断时间。

| 特性 | 说明 |
|------|------|
| 核心功能 | 主库宕机时自动选举新主、完成 failover |
| 数据保护 | 尽量从旧主库拉取未同步的 binlog，减少数据丢失 |
| 应用透明 | 配合 VIP/DNS 切换，应用无需修改连接配置 |
| 开源 | Perl 编写，GitHub 可获取 |

**适用场景**：1 主多从架构，需要主库故障后自动切换且尽量少丢数据。

---

## MHA 架构与角色

| 角色 | 说明 | 部署位置 |
|------|------|----------|
| **MHA Manager** | 监控主从、执行 failover | 独立节点或与从库同机 |
| **MHA Node** | 执行本地命令（拉 binlog、切换等） | 所有 MySQL 节点 |
| **Master** | 主库，提供写服务 | 1 台 |
| **Slave** | 从库，提供读服务 | ≥2 台（即至少2台，1 个可作为候选 Master） |

**拓扑示意**：
```
[应用] --> VIP/DNS --> [Master]
                        |
                   [Slave1] [Slave2] ...
                        |
                 [MHA Manager] 监控
```
> **VIP 实现方式**：图中 VIP/DNS 为泛指。常见做法有 **Keepalived**（VIP 漂移到新主）、MHA 配置的 `master_ip_failover_script` 自定义脚本（绑 IP/arping）、或 DNS 切换；云上也可用弹性 IP/内网 VIP 等。应用只连 VIP 或域名，主切换后无感知。

**MHA 从 MySQL 5.0/5.1 开始支持，2011 年左右开始开源**。

- **版本要求**：MHA 并不依赖特定高版本 MySQL，MySQL 5.0 及以上即可（5.1、5.5、5.6、8.0 等都支持，支持 MySQL 社区版及 Percona/MariaDB）。
- **开源时间**：MHA 最初由日本 DeNA 公司工程师开发，2011 年左右项目正式开源，很快广泛应用于企业 MySQL 集群高可用场景。
- **兼容性说明**：MHA 已停止新功能维护，但其 failover 机制对绝大多数 MySQL 主从集群适用，支持 GTID、传统 binlog 两类复制。

> **简要总结**：  
> - **MySQL 5.0/5.1+ 起可用 MHA**  
> - **2011 年左右开源，老牌且实用的开源 MySQL高可用工具**

---

## 前置条件

1. **MySQL 主从已搭建**：binlog 开启，复制用户已创建  
2. **SSH 免密**：Manager 与各 Node 之间互相可免密登录  
3. **所有 Slave 均可成为 Master**：任一从库都能被提升为新主（需 binlog 完整）

**生产常见规模**：MHA 要求至少 **2 台从库**（1 台被选为新主后，其它从库需指向新主）。生产上多为 **1 主 2 从** 或 **1 主 3 从**：2 从可满足故障切换与一候选主；3 从在读多写少场景下兼顾读扩展与高可用。

**重要**：`binlog-do-db` / `replicate-ignore-db` 等过滤规则在主从之间必须一致，否则 MHA 检测复制会失败。

---

## 依赖安装

**所有节点**（Master、Slave、Manager）安装 Node 及 Perl 依赖：

```bash
# 安装 Perl 依赖
yum install -y perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch \
  perl-Parallel-ForkManager perl-Time-HiRes

# 安装 MHA Node
tar xf mha4mysql-node-0.58.tar.gz
cd mha4mysql-node-0.58
perl Makefile.PL
make && make install
```

**仅 Manager 节点**安装 Manager：

```bash
tar xf mha4mysql-manager-0.58.tar.gz
cd mha4mysql-manager-0.58
perl Makefile.PL
make && make install
```

---

## SSH 免密配置

Manager 需要 SSH 到所有 MySQL 节点执行命令，需配置免密：

```bash
# 在 Manager 上生成密钥
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# 复制到各节点（含 Manager 自身）
ssh-copy-id -i ~/.ssh/id_rsa.pub root@db01
ssh-copy-id -i ~/.ssh/id_rsa.pub root@db02
ssh-copy-id -i ~/.ssh/id_rsa.pub root@db03
```

验证：
```bash
masterha_check_ssh --conf=/etc/mha/app1.cnf
```

---

## MHA 配置文件

创建 `/etc/mha/app1.cnf`：

```ini
[server default]
# MHA 连接 MySQL 的用户
user=mha
password=xxx
# SSH 用户
ssh_user=root
# 工作目录
manager_workdir=/var/log/mha/app1
manager_log=/var/log/mha/app1/manager.log
# VIP 切换脚本（可选，用于应用透明切主）
# master_ip_failover_script=/usr/local/bin/master_ip_failover
# master_ip_online_change_script=/usr/local/bin/master_ip_online_change

[server1]
hostname=192.168.1.101
port=3306
candidate_master=1

[server2]
hostname=192.168.1.102
port=3306
candidate_master=1

[server3]
hostname=192.168.1.103
port=3306
candidate_master=1
```

`candidate_master=1` 表示该节点可被选为新主。

---

## MySQL 复制用户与监控用户

主库执行：

```sql
-- 复制用户
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl_pass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

-- MHA 监控与管理用户
CREATE USER 'mha'@'%' IDENTIFIED BY 'mha_pass';
GRANT ALL ON *.* TO 'mha'@'%';
FLUSH PRIVILEGES;
```

---

## 健康检查

```bash
# 检查 SSH
masterha_check_ssh --conf=/etc/mha/app1.cnf

# 检查复制
masterha_check_repl --conf=/etc/mha/app1.cnf

# 检查 MHA 状态（启动后）
masterha_check_status --conf=/etc/mha/app1.cnf
```

任一检查失败需解决后再启动 Manager。

---

## 启动 MHA Manager

```bash
# 前台启动（调试用）
masterha_manager --conf=/etc/mha/app1.cnf

# 后台启动（生产推荐）
nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_consensus \
  --ignore_last_failover < /dev/null >> /var/log/mha/app1/manager.log 2>&1 &
```

---

## Failover 流程简述

当主库宕机时，MHA 自动执行：

1. **检测主库不可达**：Manager 心跳检测失败  
2. **选举新主**：选择 binlog 最完整、优先级最高的 Slave  
3. **差异日志补全**：从宕机主库（若可访问）或其他从库拉取缺失 binlog  
4. **提升新主**：选中的 Slave 执行 `STOP SLAVE`，应用差异日志  
5. **其它从库指向新主**：`CHANGE MASTER TO` 指向新主  
6. **VIP 切换**（若配置）：调用脚本将 VIP 漂移到新主，应用无感知

#### 补全 binlog 从哪里补？选了新主还要补别的节点的 log？

在 MHA 主库故障切换过程中，`binlog` 的补全是核心环节，目的是**尽可能弥补主库宕机时尚未同步到其它节点的事务，减少数据丢失**。具体流程如下：

1. **binlog 补全的来源**  
   - 在主库宕机后，MHA 会检测所有从库（Slave）与主库的 `binlog` 位置，选出**数据最新、最完整**的从库作为候选新主（通常 binlog most updated slave）。
   - 此时，有可能仍有**部分 binlog 日志只保存在原主库或部分同步到某些从库**，不同从库可能 binlog 应用进度不一。
   - MHA 会尝试**从原主库（若还能短暂连接）拉取未同步的 binlog 日志**，补全到新主上。
   - 如果原主库网络不可达，只能以“数据最完整的那个从库”作为新主，可能会丢失极少未同步事务。

2. **新主 binlog 补全后，是否要补别的节点的 log？**  
   - **需要！** 其它从库（未被选为新主的 Slave）其 binlog 应用进度可能**落后于新主**。
   - MHA 会自动让这些“老 Slave”从新的主库获取并补齐缺失的 binlog，保持复制一致性。
   - 即：  
      - 先选出数据最新的从库作为新主  
      - 尽量让新主补全所有未同步 binlog  
      - 其它从库切换 `CHANGE MASTER TO` 指向新主，并补齐缺少的 binlog 日志

3. **简要流程总结**  
   - 主库宕机  
   - 检查各从库 binlog 位点，选新主  
   - 拉取原主未同步的 binlog（可连得上则补, 否则以新主现状为准）
   - binlog 的补全操作是由 MHA Manager 发起和主导的。具体来说，MHA Manager 发现主库心跳不可达后，会通过 SSH 登录原主库所在主机，直接访问其文件系统或磁盘，尝试拉取尚未同步到从库的 binlog 日志末尾部分，并将这些日志补齐到新主库（最优 Slave）。这一步并不依赖 MySQL 服务进程本身是否可用（即便 mysqld 挂了，只要主机和存储还能访问即可）。如果原主机彻底不可访问（如宕机或存储损坏），那么只能以复制最完整的从库数据为准，极少量最近的事务数据有可能丢失。换句话说，binlog 的补全整个流程均由 MHA Manager 自动化完成，大大降低了人工介入和数据丢失风险。
   - 其它从库重设复制，向新主“对账”并补齐缺少的 binlog（日志位点自动追平）
   - 新主通常拥有当前集群中最完整的数据，log 位点已经“领先”于其他从。其它从库需要重设复制源，向新主“查缺补漏”自动同步落后部分的 binlog（不是新主向从拉，而是从向新主追），这样所有从库数据最终与新主一致，不会丢失最近事务。

4. **MHA 优点**  
   - 最大化减少故障切换期间的数据丢失，兼顾一致性和可用性  
   - 整个过程自动完成，无需人工手动复制 binlog

> **小结：**  
> - binlog 补全以原主为最权威来源，原主拉不到只认数据最新的从库  
> - 新主补全后，老从库自动向新主同步补齐  
> - 这样保证选新主后，集群所有节点的数据都能一致，丢数据风险降到最低

---

## 常见问题与面试要点

| 问题 | 说明 |
|------|------|
| binlog 过滤不一致 | 主从 `binlog-do-db` 等必须一致，否则 `check_repl` 失败 |
| SSH 超时 | 确保所有节点互通，防火墙开放 22 端口 |
| 数据丢失 | 主库宕机前未同步的 binlog 可能丢失，建议配合半同步复制 |
| Manager 单点 | Manager 本身无 HA，可配合 keepalived 等做双机 |

**面试简答**：MHA 在主库宕机时自动选举最优从库为新主，通过补全 binlog 尽量少丢数据，配合 VIP 实现应用透明切换；部署需 SSH 免密、复制用户、binlog 过滤一致等前置条件。

---

*本文档适用于面试与实战中的 MySQL 性能分析与架构选型参考。*

[src: raw/ingested/2技术/mysql/MySQL高可用与集群架构-六、MHA-集群部署分析和详细部署过程.md]