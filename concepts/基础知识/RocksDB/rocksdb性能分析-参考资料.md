# RocksDB 性能分析参考资料

> 本文档整理了 RocksDB 性能分析相关的官方文档、博客文章、研究论文等参考资料。

[src: raw/ingested/2技术/rocksdb/rocksdb性能分析-参考资料.md]

## 官方 Wiki

1. [RocksDB Performance Benchmarks（官方 Wiki）](https://github.com/facebook/rocksdb/wiki/Performance-Benchmarks)
2. [RocksDB In-Memory Workload Performance Benchmarks](https://github.com/facebook/rocksdb/wiki/RocksDB-In-Memory-Workload-Performance-Benchmarks)
3. [WAL Performance（官方 Wiki）](https://github.com/facebook/rocksdb/wiki/WAL-Performance)
4. [Pipelined Write（官方 Wiki）](https://github.com/facebook/rocksdb/wiki/Pipelined-Write)
5. [Write Stalls（官方 Wiki）](https://github.com/facebook/rocksdb/wiki/Write-Stalls)
6. [RocksDB Architecture Guide（官方 Wiki）](https://github.com/facebook/rocksdb/wiki/Rocksdb-Architecture-Guide)

## RocksDB Blog

7. [Higher write throughput with unordered_write feature（RocksDB Blog）](https://rocksdb.org/blog/2019/08/15/unordered-write.html)
8. [FlushWAL; less fwrite, faster writes（RocksDB Blog）](https://rocksdb.org/blog/2017/08/25/flushwal.html)

## 第三方博客与文章

9. [How We Optimize RocksDB in TiKV — Write Batch Optimization](https://medium.com/@siddontang/how-we-optimize-rocksdb-in-tikv-write-batch-optimization-28751a4bdd8b)
10. [How We Optimize RocksDB in TiKV — The Battle Against the DB Mutex](https://medium.com/@siddontang/how-we-optimize-rocksdb-in-tikv-part-1-the-battle-against-the-db-mutex-5c4f12c02c01)
11. [Intel RocksDB Benchmarking with Xeon-based Systems](https://www.intel.com/content/www/us/en/developer/articles/guide/rocksdb-benchmarking-with-xeon-based-systems.html)
12. [Small Datum: RocksDB benchmarks（Mark Callaghan Blog）](https://smalldatum.blogspot.com/)

## 学术论文

13. [vLSM: Low tail latency and I/O amplification in LSM-based KV stores（arXiv 2024）](https://arxiv.org/abs/2407.15581)

## 其他

14. [B-Tree vs LSM-Tree（TiKV Deep Dive）](https://tikv.org/deep-dive/key-value-engine/b-tree-vs-lsm)
15. [GitHub Issue #11669: Throughput not increasing with increased parallelism](https://github.com/facebook/rocksdb/issues/11669)

## 相关页面

- [[RocksDB性能测试数据与硬件配置]]
- [[RocksDB文件体系]]
- [[KeyDB存算分离项目]]