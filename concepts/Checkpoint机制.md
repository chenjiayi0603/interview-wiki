# Checkpoint 机制

- **checkpoint（检查点）** 就是数据库/分布式系统定期把内存中的数据或日志一次性写到磁盘，生成恢复用的一致性快照，方便崩溃后快速恢复。

- checkpoint 的作用包括：
    1. **缩短宕机恢复时间**：做了checkpoint后，宕机时只需从最近的检查点后的日志进行恢复
    2. **保证内存数据最终落盘**，防止因崩溃导致大量数据丢失
    3. **控制WAL日志体积**：日志只有在被checkpoint后的内容才能删除或复用
    4. **为系统提供强一致性"快照"**：为下游的恢复和校验提供可依赖的基线

- 刷盘是动作，checkpoint是过程。checkpoint必然包含刷盘，但刷盘不等价于checkpoint。
- checkpoint频率越高，恢复速度越快，但对系统性能压力越大。

[src: raw/ingested/2技术/mysql/数据库基础概念与范式设计.md]

## Related Pages
- [[MySQL事务提交与两阶段提交]]
- [[MySQL架构与存储引擎]]
