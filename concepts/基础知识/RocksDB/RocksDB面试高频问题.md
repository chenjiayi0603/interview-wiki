# RocksDB面试高频问题

## Q1: LSM树的优势和劣势？
**参考答案**：
- **优势**：写性能高，顺序写磁盘；空间紧凑；支持高并发
- **劣势**：读性能差（需多层级查找）；写放大（多次compaction）；空间放大

## Q2: RocksDB如何保证数据不丢失？
**参考答案**：
1. WAL预写日志
2. MemTable满时刷盘
3. 定期SSTable checkpoint
4. 分布式存储副本

## Q3: 什么是写放大？如何优化？
**参考答案**：
- 写放大：写入1KB实际磁盘写入10KB
- 原因：WAL + MemTable flush + Compaction多次重写
- 优化：
  - 调大MemTable减少flush频率
  - 选择合适compaction策略
  - 批量写入
  - RocksDB可压缩

## Q4: RocksDB的MemTable用什么实现？
**参考答案**：
- **SkipList（跳表）**：默认，O(log n)查找，有序
- **Hash SkipList**：按前缀分桶
- **Vector**：append-only，适合写多读少
- **PlainTable**：小数据量

## Q5: 布隆过滤器的误判率如何计算？
**参考答案**：
```
p = (1 - e^(-kn/m))^k
k = (m/n) * ln(2)
p ≈ (0.6185)^m/n  # m/n为每个key的位数
```
- m/n越大，误判率越低
- 常用配置：10位/key，约1%误判率

## Q6: 如何解决缓存穿透？
**参考答案**：
1. **布隆过滤器**：快速判断不存在
2. **缓存空值**：短TTL缓存null
3. **参数校验**：白名单
4. **限流熔断**：保护后端

## Q7: 大Key如何处理？
**参考答案**：
- **拆分**：string类型限制在10MB
- **压缩**：对value进行gzip
- **拆分结构**：hash按field拆分
- **异步删除**：避免阻塞

## Q8: Redis和RocksDB如何选型？
**参考答案**：
| 场景 | 选择Redis | 选择RocksDB/KeyDB |
|------|----------|-------------------|
| 超大数据量 | ✗ | ✓ |
| 低延迟<1ms | ✓ | ✗ |
| 数据结构丰富 | ✓ | ✗ |
| 内存充足 | ✓ | ✗ |
| 成本敏感 | ✗ | ✓ |
| 持久化需求 | 可选 | 必须 |

## Q9: RocksDB的Compaction触发条件？
**参考答案**：
- **L0 -> L1**：L0文件数达到阈值（默认4）
- **Ln -> Ln+1**：Ln层大小超过预期
- **手动触发**：DisableAutoCompactions=false

## Q10: KeyDB的多线程模型？
**参考答案**：
- **主线程**：接受连接
- **Worker线程**：处理命令
- **后台线程**：Compaction、Replication
- **IO线程**：网络I/O（多线程读取）

[src: raw/ingested/2技术/rocksdb/RocksDB与存储.md]

## 手写代码

### 1. 简化MemTable
```cpp
#include <map>
#include <mutex>

class MemTable {
private:
    std::map<std::string, std::string> data;
    std::mutex mtx;
    uint64_t sequence;
    
public:
    void put(const std::string& key, const std::string& value) {
        std::lock_guard<std::mutex> lock(mtx);
        data[key] = value;
    }
    
    std::string get(const std::string& key) {
        std::lock_guard<std::mutex> lock(mtx);
        auto it = data.find(key);
        if (it != data.end()) {
            return it->second;
        }
        return "";
    }
    
    bool del(const std::string& key) {
        std::lock_guard<std::mutex> lock(mtx);
        return data.erase(key) > 0;
    }
};
```

[src: raw/ingested/2技术/rocksdb/RocksDB与存储.md]

## Related Pages
- [[RocksDB与LSM树]]
- [[KeyDB与RocksDB]]
- [[布隆过滤器]]
- [[缓存策略]]
- [[KeyDB存算分离项目]]
- [[C++高频面试问题]]
- [[C++手写代码模板]]
