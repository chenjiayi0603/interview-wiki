# Thunder Raft 实现详解

> Raft 状态机、选举优化、日志复制、快照、集群变更。

---

## 一、Raft 状态机

### 1.1 状态转换

```
                        ┌───────────┐
                        │  Follower │
                        └─────┬─────┘
                              │ 选举超时
                              ▼
                        ┌───────────┐
            ┌──────────→│ Candidate │←──────────┐
            │           └─────┬─────┘           │
            │                 │ 多数票           │
            │                 ▼                 │
            │           ┌───────────┐           │
            │           │  Leader   │────────────┘
            │           └───────────┘  心跳超时
            │                 │
            └─────────────────┘
            发现更高任期
```

### 1.2 核心数据结构

```cpp
class RaftNode {
    // 持久化状态
    int current_term_;
    int voted_for_;
    vector<LogEntry> log_;

    // 易失状态
    State state_;             // Follower / Candidate / Leader
    int commit_index_;        // 已提交的最大日志索引
    int last_applied_;        // 已应用到状态机的最大索引

    // Leader 易失状态
    vector<int> next_index_;  // 每个 Follower 的下一条日志索引
    vector<int> match_index_; // 已复制给 Follower 的最大日志索引
};
```

## 二、选举优化

### 2.1 标准选举 vs Thunder 优化

```
标准 Raft 选举（200ms+）：
  T+0    Leader 故障
  T+150 Follower 随机超时 (150-300ms)
  T+151 发起选举 RequestVote RPC
  T+153 收到多数投票，成为 Leader
  T+200 发送心跳确认 Leader 身份
  合计: ~200ms

Thunder 优化（45ms）：
  T+0     Leader 故障
  T+40    Follower 主动探测 Leader 存活（心跳间隔 50ms）
  T+41    发现 Leader 失联，立即发起选举
  T+42    异步批量发送 RequestVote
  T+43    收到多数投票，成为 Leader
  T+45    心跳确认
  合计: ~45ms
```

### 2.2 优化措施

```cpp
class RaftOptimizer {
    // 1) 缩短心跳间隔 150ms -> 50ms
    static constexpr int kHeartbeatIntervalMs = 50;
    static constexpr int kElectionTimeoutMs = 150;  // 容错

    // 2) 预投票 (Pre-Vote) 防止网络分区时的无效选举
    bool CheckPreVote() {
        // 先确认自己能联系上多数节点，再正式发起选举
        int votes = 0;
        for (auto& peer : peers_) {
            if (peer->Ping(current_term_)) votes++;
        }
        return votes > peers_.size() / 2;
    }

    // 3) 异步投票收集
    void StartElection() {
        state_ = Candidate;
        current_term_++;
        auto start = now();
        // 异步发送，不阻塞
        for (auto& peer : peers_) {
            peer->AsyncRequestVote({
                .term = current_term_,
                .candidate_id = id_,
                .last_log_index = log_.LastIndex(),
                .last_log_term = log_.LastTerm(),
            });
        }
    }
};
```

## 三、日志复制

### 3.1 单条 vs 批量复制

```
单条复制（延迟 = RTT * N）：
  Leader  Follower
    ├──── Entry 1 ────→│
    │←─── Ack 1 ──────┤  RTT=2ms
    ├──── Entry 2 ────→│
    │←─── Ack 2 ──────┤  RTT=2ms
    ...               N 条 = 2N ms

批量复制（延迟 = RTT + 等待）：
  Leader  Follower
    │  攒够 100 条或 5ms    
    │     ← 等待中 →
    ├── Batch[100] ──→│
    │←── Ack 100 ─────┤  1 RTT = 2ms + 5ms 等待
    提升 ~100 倍
```

### 3.2 批量复制实现

```cpp
class RaftLogReplicator {
    struct BatchBuffer {
        vector<LogEntry> entries;
        Timer flush_timer;
        int target_node;
    };

    unordered_map<int, BatchBuffer> buffers_;

    void AppendEntry(LogEntry entry) {
        int node_id = entry.target_node;
        auto& buf = buffers_[node_id];
        buf.entries.push_back(entry);

        // 达到批量阈值，立即发送
        if (buf.entries.size() >= 100) {
            FlushBatch(node_id);
            return;
        }

        // 启动延迟定时器（最大等待 5ms）
        if (!buf.flush_timer.IsRunning()) {
            buf.flush_timer.Start(5ms, [this, node_id]() {
                FlushBatch(node_id);
            });
        }
    }

    void FlushBatch(int node_id) {
        auto& buf = buffers_[node_id];
        if (buf.entries.empty()) return;

        // 批量发送 AppendEntries RPC
        peers_[node_id]->AppendEntries(buf.entries);
        buf.entries.clear();
    }
};
```

## 四、快照 (Snapshot)

### 4.1 快照机制

```cpp
class RaftSnapshot {
    // 快照生成（每 10000 条日志或 1 小时）
    void TakeSnapshot() {
        auto snapshot = MakeSnapshot();
        snapshot.meta.last_included_index = last_applied_;
        snapshot.meta.last_included_term = GetTerm(last_applied_);

        // 写入快照文件
        SaveToFile(snapshot, "snapshot.dat");

        // 删除已包含在快照中的日志
        CompactLog(last_applied_);
    }

    // 快照安装（Follower 落后太多时）
    void InstallSnapshot(Snapshot snapshot) {
        LoadFromFile(snapshot, "snapshot.dat");
        // 重置状态机到快照状态
        state_machine_->Restore(snapshot.state);
        // 删除冲突的日志
        log_.Truncate(snapshot.meta.last_included_index);
    }
};
```

### 4.2 快照性能

| 指标 | 数值 |
|------|------|
| 快照生成速度 | 50MB/s |
| 快照恢复速度 | 100MB/s |
| 快照间隔 | 10000 条日志 / 1 小时 |
| 快照压缩 | zstd (压缩比 ~5:1) |

## 五、集群变更

### 5.1 成员变更

```cpp
class ClusterMembership {
    // 一次只添加/删除一个节点（单节点变更，Raft 论文推荐）
    void AddNode(NodeInfo node) {
        auto op = ConfigurationOp{Add, node};
        raft_node_->Apply(op);  // 作为日志条目提交

        // 等待节点同步完成
        while (GetProgress(node.id) < 1.0) {
            sleep(100ms);
        }
    }

    void RemoveNode(int node_id) {
        auto op = ConfigurationOp{Remove, node_id};
        raft_node_->Apply(op);
        // 从集群中移除节点
        peers_.erase(node_id);
    }
};
```

## 六、线性一致性读

### 6.1 ReadIndex 优化

```cpp
class LinearizableRead {
    // 优化方案：Leader 记录当前 commit_index，返回给 Follower
    // 不需要走 Raft 日志复制

    int Read(Operation op) {
        if (state_ != Leader) {
            // 转发给 Leader
            return RedirectToLeader(op);
        }

        // 1) 记录当前 commit_index
        int read_index = commit_index_;

        // 2) 确认自己仍是 Leader（心跳）
        if (!CheckLeaderShip()) {
            return RedirectToLeader(op);
        }

        // 3) 等待状态机 apply 到 read_index
        while (last_applied_ < read_index) {
            sleep(1ms);
        }

        // 4) 执行读取
        return state_machine_->ExecuteRead(op);
    }
};
```
