# CPU 高但业务量低

## IO/网络（重试风暴、阻塞 IO）

- 使用异步 IO（io_uring/epoll ET）、批量收发、Nagle/Delay Ack 配置、调整 RTO/keepalive。
- 减少序列化格式开销（protobuf/flatbuffers/capnproto），压缩只在必要时开启；开启 TLS 会话复用。

**举例**：

- 同步 read/write 改 epoll+非阻塞，核心伪代码：
  ```cpp
  int fd = ...;
  int n = read(fd, buf, len); while(n > 0) { n = read(fd, buf, len); }
  ```
- epoll 注册 ET：https://man7.org/linux/man-pages/man7/epoll.7.html
- sysctl 调整 TCP 参数：
  ```
  sysctl -w net.ipv4.tcp_tw_reuse=1
  sysctl -w net.ipv4.tcp_rmem='4096 65536 16777216'
  ```
- protobuf 替换 JSON：
  ```cpp
  MyMsg.ParseFromArray(buf, len);
  ```
- TLS session 复用：
  ```
  openssl s_client -reconnect ...
  ```

**分析**：网络/IO 瓶颈常表现为 CPU 闲但 QPS 卡住、尾延迟受 RTT/重传影响；异步化与协议轻量化优先。

## 磁盘/存储

- 顺序 IO 优先，批量刷盘，合并小写入；使用直接 IO/异步 IO；SSD 上避免小随机写放大。
- KV/DB：合理 cache、批写、表分片、冷热分层；SQL 加索引/覆盖索引、避免全表扫。

**举例**：

- 批量写 WAL + 一次 fsync:
  ```cpp
  buffer.write(data);
  if (buffer.size() > BATCH_SIZE || timeup()) {
      ::write(fd, buffer.ptr(), buffer.size());
      fsync(fd);
      buffer.clear();
  }
  ```
- RocksDB/LSM 优化参数：
  ```
  rocksdb_options.set_max_background_flushes(4);
  ```
- MySQL 创建索引：
  ```
  CREATE INDEX idx_xxx ON users(uid);
  ```
- Redis 本地缓存：
  ```cpp
  LRUCache<std::string, User> cache;
  ```

- 监控磁盘 util:
  ```
  iostat -x 1
  ```

**分析**：磁盘瓶颈看 util、await、IOPS；DB 瓶颈看慢查询与锁；先顺序与批量，再考虑 O_DIRECT 与 SQL 优化。

## 磁盘 IO 策略

**要点**：**顺序优先**：尽量顺序写日志、顺序读大文件，避免随机小 IO。**批量与合并**：批量刷盘、合并小写入（如 log buffer 攒够再 write），减少 IO 次数。**直接 IO / 异步 IO**：绕过 page cache 时用 O_DIRECT，配合对齐与块大小；异步 IO（io_uring、libaio）避免阻塞线程。**SSD**：避免小随机写导致写放大，尽量顺序写或批量写。

**举例**：

- 批量 write + fsync:
  ```cpp
  static char buffer[4096]; static size_t buf_len = 0;
  memcpy(buffer + buf_len, data, datalen); buf_len += datalen;
  if (buf_len >= 4096) { write(fd, buffer, buf_len); fsync(fd); buf_len = 0; }
  ```
- LSM 顺序写（rocksdb cf_options）:
  ```
  rocksdb_options.compression = rocksdb::kSnappyCompression;
  rocksdb_options.max_write_buffer_number = 3;
  ```
- io_uring 批量提交示例
  ```cpp
  io_uring_prep_write(sqe, fd, buffers[i], bufsz, off);
  io_uring_submit(&ring);
  ```
- O_DIRECT:
  ```cpp
  int fd = open("file", O_WRONLY|O_DIRECT);
  ```

## KV/DB 与 SQL 优化

**要点**：**缓存**：热点数据放本地 cache 或 Redis，减少直连 DB。**批写**：插入/更新尽量 batch commit，减少网络与事务开销。**分片与分层**：表按 key 或时间分片；冷热分离，热数据 SSD、冷数据 HDD 或归档。**SQL**：为常用查询加索引、覆盖索引避免回表；避免全表扫描与深分页。

**举例**：

- 本地 LRU cache:
  ```cpp
  LRUCache<int, UserInfo> cache;
  auto val = cache.get(id); if (!val) { fetch_from_db(); cache.set(id, ...);}
  ```
- Redis 查询:
  ```
  redis-cli get user:123
  ```
- 批量插入 MySQL:
  ```
  INSERT INTO users (id, name) VALUES (1, 'a'), (2, 'b'), ...;
  ```
- 批量事务：
  ```cpp
  txn.start();
  for(...) txn.write(row); txn.commit();
  ```
- 索引优化:
  ```
  CREATE INDEX idx_col1 ON table1(col1);
  ```
- 覆盖索引 查询优化:
  ```
  SELECT col1, col2 FROM t1 WHERE col1=...;
  ```
  并为 col1, col2 添加联合索引。

[src: raw/ingested/2技术/性能优化/瓶颈-瓶颈症状与治理手段-CPU-高但业务量低.md]