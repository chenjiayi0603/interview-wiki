# AI工程师学习大纲 —— 从14年后端老兵到AI Agent工程师的实战路线图

> **适用对象**：14年C++/Go后端开发工程师，有分布式系统、华为云Redis/存储、IM系统经验  
> **目标岗位**：AI Agent开发工程师  
> **核心策略**：发挥后端工程优势，补齐AI算法短板，强化AI工程落地能力  
> **预估总时长**：12-16周（全职学习）/ 20-24周（兼职学习）

---

## 📊 学习优先级总览

| 优先级 | 板块 | 理由 | 建议时长 |
|--------|------|------|----------|
| 🔴 P0 | 三、应用侧（AI工程核心） | 面试核心考点，转岗最大差异点 | 5-6周 |
| 🔴 P0 | 六、面试专项 | 直接决定面试结果 | 持续进行 |
| 🟡 P1 | 二、模型侧（算法基础） | 必须懂原理但不需手推公式 | 3-4周 |
| 🟡 P1 | 四、工程侧（落地能力） | 后端优势区，快速转化为AI工程力 | 2-3周 |
| 🟢 P2 | 五、产品侧（业务能力） | 加分项，面试中体现产品思维 | 1-2周 |
| 🟢 P2 | 一、传统后端基础 | 已有基础，查漏补缺 | 1-2周 |

---

## 🔄 后端转AI的优势与短板分析

### ✅ 你的优势（可直接迁移）
- **系统设计能力**：分布式系统设计经验 → Agent系统架构设计
- **工程化能力**：微服务、K8s、可观测性 → AI工程化部署
- **性能优化**：pprof/trace经验 → LLM推理优化、缓存策略
- **Go语言**：goroutine/channel → 并发Agent调度、MCP Server开发
- **数据库**：Redis/存储经验 → 向量数据库、缓存策略
- **gRPC/Protobuf**：直接复用于MCP协议、模型服务化

### ❌ 需要补的短板
- **Python生态**：AI/ML库几乎全是Python，需要快速上手
- **数学基础**：线性代数、概率论、微积分（不需要研究级，但要懂直觉）
- **模型原理**：Transformer、注意力机制、微调原理
- **Prompt Engineering**：全新的"编程范式"，需要大量实践
- **AI评估体系**：如何衡量AI系统的效果，与传统软件测试完全不同

---

# 一、传统后端基础（不能丢）

> 💡 本板块你有深厚基础，重点是查漏补缺和跟进新特性。面试中后端问题仍是基础考察项。

## 1. Go语言进阶

### 1.1 泛型 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：类型参数、类型约束（comparable/any/自定义接口）、泛型函数/结构体/方法
- **Go 1.26新特性**：泛型约束自我引用（`type Adder[A Adder[A]] interface{ Add(A) A }`）
- **与你经验关联**：已有Go基础，泛型是1.18后的重点，面试必问
- **学习资源**：
  - [Go Generics官方教程](https://go.dev/doc/tutorial/generics)
  - [Go 1.26 Release Notes](https://go.dev/doc/go1.26)

### 1.2 并发模式（goroutine/channel/context） ⭐⭐⭐ | 实操级 | 建议时间：3天
- **知识点**：
  - goroutine调度模型（GMP模型）
  - channel：无缓冲/有缓冲、select多路复用、方向限定
  - context：取消传播、超时控制、值传递
  - sync包：Mutex/RWMutex/WaitGroup/Once/Pool/Map
  - 并发模式：fan-in/fan-out、pipeline、worker pool、errgroup
- **与你经验关联**：核心优势区，面试中用来展示Go功底
- **学习资源**：
  - 《Go语言高级编程》并发章节
  - [Go Concurrency Patterns (Rob Pike)](https://go.dev/talks/2012/concurrency.slide)

### 1.3 内存管理、GC调优 ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - Go内存分配器：tcmalloc思想、mspan/mcache/mcentral/mheap
  - GC演进：v1.0 STW → v1.5 三色标记 → v1.8 混合写屏障 → v1.26 Green Tea GC
  - **Go 1.26重点**：Green Tea GC默认启用，GC CPU开销降低10%-40%，AVX向量化加速扫描
  - **Go 2026路线图**：`runtime.free` 显式释放、Specialized malloc、SIMD原生支持
  - GC调优：`GOGC`、`GOMEMLIMIT`、对象池化
  - 逃逸分析：堆/栈分配规则、减少逃逸的技巧
- **与你经验关联**：后端性能优化的核心能力，AI场景中显存+内存双重管理更需要此能力
- **学习资源**：
  - [Go GC Guide](https://go.dev/doc/gc-guide)
  - [Go 1.26 Green Tea GC](https://go.dev/blog/go1.26)
  - [Go 2026路线图](https://blog.csdn.net/bigwhite20xx/article/details/155367574)

### 1.4 错误处理哲学 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：error接口、errors.Is/As/Unwrap、panic/recover、sentinel error、wrap error
- **Go 1.26新特性**：`errors.AsType` 泛型错误类型断言
- **与你经验关联**：已有基础，了解1.26新特性即可
- **学习资源**：[Go Blog: Error Handling](https://go.dev/blog/error-handling-and-go)

### 1.5 性能分析：pprof/trace/benchmark ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - pprof：CPU/Heap/Goroutine/Block/Mutex profiling
  - trace：执行追踪、调度延迟分析
  - benchmark：基准测试、benchstat对比
  - **Go 1.26新特性**：`goroutineleak` profile（实验性）
- **与你经验关联**：已有基础，是AI工程中排查性能瓶颈的关键工具
- **学习资源**：[Go Diagnostics](https://go.dev/doc/diagnostics)

### 1.6 Go 1.26新特性与2026路线图 ⭐⭐ | 面试级 | 建议时间：2天
- **Go 1.26关键特性**：
  - `new(expr)` 指针初始化语法：`new(42)` 替代 `x:=42; &x`
  - 泛型约束自我引用
  - Green Tea GC默认启用
  - cgo开销降低约30%
  - 切片backing store栈分配优化
  - `go fix` 命令重写：基于analysis框架 + modernizers + inline analyzer
  - 实验性：`simd/archsimd` SIMD包、`runtime/secret` 安全擦除、`goroutineleak` profile
  - 新标准库：`crypto/hpke`、`crypto/mlkem/mlkemtest`、`testing/cryptotest`
- **Go 2026路线图**：
  - SIMD high-level API（ARM64 SVE可伸缩向量）
  - `runtime.free` / `runtime.freegc` 显式内存释放
  - Specialized malloc（noscan对象专用分配器）
  - `sync.Sharded` 分片值（减少缓存行争用）
  - Scheduling affinity（调度亲和性）
  - Memory regions（Arena的继任者）
  - 泛型方法（generic method）
  - 联合类型（union type）
  - 无C工具链CGO
- **与你经验关联**：面试加分项，展示技术跟进能力
- **学习资源**：
  - [Go 1.26 Release Notes](https://go.dev/doc/go1.26)
  - [Go 1.26 is released](https://go.dev/blog/go1.26)
  - [Go 2026路线图分析](https://blog.csdn.net/bigwhite20xx/article/details/155367574)

---

## 2. Python基础（AI生态主力语言）

> ⚠️ 这是你的最大短板之一。AI/ML生态90%以上是Python，必须快速补上。

### 2.1 Python基础语法速通 ⭐⭐⭐ | 实操级 | 建议时间：3天
- **知识点**：数据类型、列表/字典/集合推导式、装饰器、生成器、上下文管理器、类型注解
- **与你经验关联**：有编程基础，语法速通即可，重点关注Python独特的惯用法
- **学习资源**：
  - [Python官方教程](https://docs.python.org/3/tutorial/)
  - 《Fluent Python》第2版

### 2.2 虚拟环境与包管理 ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：venv、conda、pip、pyproject.toml、uv（新兴快速包管理器）
- **与你经验关联**：类似Go Module，理解依赖管理思想即可
- **学习资源**：[uv官方文档](https://docs.astral.sh/uv/)

### 2.3 常用库：NumPy、Pandas ⭐⭐ | 实操级 | 建议时间：3天
- **知识点**：
  - NumPy：ndarray、广播机制、向量化运算、索引/切片
  - Pandas：DataFrame/Series、数据清洗、groupby、merge
- **与你经验关联**：AI数据处理的基础，不深入但要会用
- **学习资源**：
  - [NumPy官方教程](https://numpy.org/doc/stable/user/quickstart.html)
  - [Pandas官方教程](https://pandas.pydata.org/docs/getting_started/intro_tutorials/)

### 2.4 异步编程：asyncio ⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：async/await、事件循环、Future/Task、aiohttp/httpx
- **与你经验关联**：有Go并发基础，理解事件循环vs goroutine的区别
- **学习资源**：[asyncio官方文档](https://docs.python.org/3/library/asyncio.html)

### 2.5 与Go的互操作（gRPC） ⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：gRPC Go Server + Python Client、Protobuf定义、跨语言调用
- **与你经验关联**：已有gRPC经验，这是AI工程中"Go做网关+Python做AI"的核心架构
- **学习资源**：[gRPC官方教程](https://grpc.io/docs/languages/)

---

## 3. 数据库

### 3.1 MySQL ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - 索引原理：B+树、聚簇索引/二级索引、覆盖索引、索引下推
  - 事务隔离级别：读未提交/读已提交/可重复读/串行化、MVCC实现
  - 锁机制：行锁/间隙锁/临键锁/意向锁、死锁检测
  - 分库分表：ShardingSphere、分片键选择、跨分片查询
- **与你经验关联**：华为云存储经验，已有深厚基础，重点复习面试高频点
- **学习资源**：《高性能MySQL》、[MySQL官方文档](https://dev.mysql.com/doc/)

### 3.2 Redis ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - 数据结构：SDS/ziplist/quicklist/skiplist/intset/hashtable
  - 持久化：RDB/AOF/hybrid、rewrite机制
  - 集群：主从复制、哨兵、Cluster分片、一致性哈希
  - 缓存策略：缓存穿透/击穿/雪崩、LRU/LFU
  - 应用：分布式锁（RedLock）、限流、消息队列
- **与你经验关联**：🔥华为云Redis项目经验，这是你的王牌，面试中深挖
- **学习资源**：《Redis设计与实现》、[Redis官方文档](https://redis.io/docs/)

### 3.3 MongoDB ⭐ | 面试级 | 建议时间：1天
- **知识点**：文档模型、BSON、聚合管道、索引类型、分片集群
- **与你经验关联**：了解即可，AI场景中主要用于存储非结构化数据
- **学习资源**：[MongoDB官方教程](https://www.mongodb.com/docs/)

### 3.4 向量数据库 ⭐⭐⭐ | 实操级 | 建议时间：3天
- **知识点**：
  - Qdrant：Rust实现、gRPC/REST API、过滤查询、_payload索引
  - Milvus：Go+C++实现、云原生架构、水平扩展、多索引类型
  - Chroma：Python实现、轻量级、适合原型开发
  - FAISS：Facebook开源、纯向量检索库（非完整数据库）
  - **对比维度**：性能、可扩展性、过滤能力、生态、部署复杂度
- **与你经验关联**：🆕 全新领域，但数据库经验可迁移，重点理解"向量索引"与传统索引的区别
- **学习资源**：
  - [Qdrant官方文档](https://qdrant.tech/documentation/)
  - [Milvus官方文档](https://milvus.io/docs)
  - 实战项目：搭建一个RAG系统，分别用Qdrant和FAISS对比

---

## 4. 微服务

### 4.1 gRPC + Protobuf ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：四种通信模式、流式RPC、拦截器、错误处理、proto3语法
- **与你经验关联**：已有经验，在AI工程中gRPC是模型服务化的核心协议
- **学习资源**：[gRPC官方文档](https://grpc.io/docs/)

### 4.2 服务发现与注册 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：Consul/Etcd/Nacos、健康检查、服务元数据
- **与你经验关联**：已有基础

### 4.3 熔断/限流/降级 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：令牌桶/漏桶/滑动窗口、熔断器模式、降级策略
- **与你经验关联**：在AI工程中，LLM API调用也需要限流和降级

### 4.4 可观测性：日志/指标/链路追踪 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - 日志：结构化日志（zap/logrus）、日志聚合
  - 指标：Prometheus + Grafana、RED/USE方法
  - 链路追踪：OpenTelemetry、Jaeger、trace传播
- **与你经验关联**：已有基础，在AI工程中需要新增：LLM调用延迟、token消耗、缓存命中率等指标

### 4.5 API网关 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：Kong/APISIX/Traefik、路由/鉴权/限流/灰度

### 4.6 go-zero / go-kratos 框架 ⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：框架设计理念、核心组件、与标准库的取舍
- **与你经验关联**：了解设计思想即可，AI工程中更常用轻量方案

---

## 5. K8s/K3s

### 5.1 Pod/Service/Deployment ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：Pod生命周期、Service类型、Deployment滚动更新、标签选择器
- **与你经验关联**：已有基础，AI工程中GPU Pod管理是重点

### 5.2 ConfigMap/Secret ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：配置管理、敏感信息管理、环境变量注入

### 5.3 Ingress配置 ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：Ingress Controller、路径路由、TLS终止

### 5.4 HPA/VPA自动伸缩 ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - HPA：基于CPU/内存/自定义指标（如请求队列长度）自动伸缩
  - VPA：垂直自动伸缩
  - **AI场景**：基于GPU利用率、推理队列深度做HPA
- **与你经验关联**：AI模型服务需要根据负载动态扩缩容

### 5.5 Helm/Kustomize ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：Chart模板、values文件、Kustomize overlay

### 5.6 故障排查实战 ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：kubectl describe/logs/exec、事件查看、资源限制排查、网络策略调试

---

## 6. 系统设计

### 6.1 分布式系统基础 ⭐⭐⭐ | 原理级 | 建议时间：2天
- **知识点**：CAP/BASE、一致性模型（强一致/最终一致/因果一致）、分区容错、分布式ID
- **与你经验关联**：核心优势区，直接迁移到Agent系统设计

### 6.2 消息队列 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - Kafka：分区/消费者组/offset管理/Exactly-Once
  - RabbitMQ：Exchange/Queue/路由模式
  - **AI场景**：Agent任务队列、异步推理调度
- **学习资源**：《Kafka权威指南》

### 6.3 分布式缓存 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：缓存一致性、缓存策略、热点key处理、多级缓存
- **与你经验关联**：Redis项目经验直接复用

### 6.4 分布式事务 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：2PC/3PC、TCC、Saga、本地消息表、事务消息

---

# 二、模型侧（算法基础）

> 💡 本板块是你的主要短板，但不需要成为算法工程师。目标是"懂原理、能讲清楚、知道边界"。

## 1. 机器学习基础

### 1.1 监督/无监督/强化学习 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - 监督学习：分类/回归，训练/验证/测试集划分
  - 无监督学习：聚类/降维/异常检测
  - 强化学习：状态/动作/奖励/策略、RLHF的基础
- **与你经验关联**：🆕 全新领域，重点理解直觉，不需要推导公式
- **学习资源**：
  - 吴恩达机器学习课程（Coursera）
  - 《统计学习方法》（李航）

### 1.2 常用算法 ⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：线性回归、逻辑回归、决策树、随机森林、SVM、K-Means
- **与你经验关联**：了解原理和适用场景即可，面试中能解释为什么选择某算法
- **学习资源**：[Scikit-learn官方教程](https://scikit-learn.org/stable/tutorial/)

### 1.3 评估指标 ⭐⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - 分类：准确率/精确率/召回率/F1/ROC-AUC
  - 回归：MAE/MSE/RMSE/R²
  - **AI工程重点**：LLM评估指标（BLEU/ROUGE/Perplexity）、RAG评估（检索准确率/回答质量）
- **与你经验关联**：后端监控思维可以迁移——"如何衡量系统效果"
- **学习资源**：[ML Metrics Guide](https://scikit-learn.org/stable/modules/model_evaluation.html)

### 1.4 过拟合/欠拟合、正则化 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：偏差-方差权衡、L1/L2正则化、Dropout、Early Stopping、数据增强
- **与你经验关联**：类比后端的"过度设计"vs"设计不足"

### 1.5 特征工程基础 ⭐ | 面试级 | 建议时间：1天
- **知识点**：特征缩放、特征选择、特征交叉、类别编码
- **与你经验关联**：了解即可，LLM时代特征工程的重要性在下降

---

## 2. 深度学习基础

### 2.1 神经网络：前向传播、反向传播 ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - 神经元模型：加权和 + 激活函数
  - 前向传播：输入→隐藏层→输出
  - 反向传播：链式法则、梯度计算
  - 自动微分：计算图、PyTorch autograd
- **与你经验关联**：🆕 核心短板，但只需要理解直觉，不需要手推梯度
- **学习资源**：
  - 3Blue1Brown《神经网络》视频系列
  - [PyTorch官方教程](https://pytorch.org/tutorials/)

### 2.2 CNN、RNN、LSTM ⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - CNN：卷积/池化/感受野、用于图像理解
  - RNN：序列建模、梯度消失/爆炸
  - LSTM：门控机制（遗忘门/输入门/输出门）
- **与你经验关联**：了解架构思想和适用场景，面试中说明"为什么Transformer取代了RNN"

### 2.3 Transformer架构详解 ⭐⭐⭐ | 原理级 | 建议时间：4天
- **知识点**：
  - Self-Attention：Q/K/V矩阵、缩放点积注意力
  - Multi-Head Attention：多头并行、注意力拼接
  - Position Encoding：正弦位置编码、RoPE（旋转位置编码）
  - Feed-Forward Network：两层MLP + 激活函数
  - Layer Normalization：Pre-Norm vs Post-Norm
  - 残差连接（Residual Connection）
  - Encoder-Decoder vs Decoder-Only架构
- **与你经验关联**：🔥 AI面试必考，必须能画图讲解整个前向传播流程
- **学习资源**：
  - [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)
  - [Attention Is All You Need 原论文](https://arxiv.org/abs/1706.03762)
  - [Let's build GPT from scratch](https://www.youtube.com/watch?v=kCc8FmEb1nY)（Karpathy）

### 2.4 损失函数、优化器 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - 损失函数：交叉熵、MSE、KL散度
  - 优化器：SGD → Momentum → RMSprop → Adam → AdamW
  - AdamW vs Adam：解耦权重衰减
  - 学习率调度：Cosine Annealing、Warmup
  - 梯度裁剪（Gradient Clipping）
- **与你经验关联**：理解"为什么用AdamW"比会推导公式更重要
- **学习资源**：[AdamW原论文](https://arxiv.org/abs/1711.05101)

---

## 3. 大语言模型（LLM）

### 3.1 GPT系列演进 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - GPT-1/2/3：规模扩展的胜利
  - GPT-3.5/4：RLHF对齐、多模态
  - GPT-4o/o1/o3：推理增强、多模态融合
  - 开源模型：LLaMA系列、Qwen系列、DeepSeek系列、GLM系列
- **与你经验关联**：了解发展脉络，面试中能讲清楚"为什么大模型有效"
- **学习资源**：[State of AI Report](https://www.stateof.ai/)

### 3.2 BERT/GPT/LLaMA架构对比 ⭐⭐⭐ | 原理级 | 建议时间：2天
- **知识点**：
  - BERT：Encoder-Only、MLM+NSP预训练、双向注意力
  - GPT：Decoder-Only、自回归生成、因果注意力掩码
  - LLaMA：RoPE位置编码、SwiGLU激活、RMSNorm、GQA（分组查询注意力）
  - **关键区别**：预训练目标、注意力掩码、位置编码方式
- **与你经验关联**：面试高频题，必须能对比三种架构的优劣
- **学习资源**：[LLaMA论文](https://arxiv.org/abs/2302.13971)

### 3.3 MoE架构 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - MoE原理：稀疏激活、专家路由、Top-K门控
  - 代表模型：Mixtral 8x7B、DeepSeek-V3、GLM-4
  - 优势：参数量大但推理成本低
  - 挑战：负载均衡、专家坍缩、All-to-All通信
- **与你经验关联**：类比微服务的"路由分发"，理解"稀疏激活"概念
- **学习资源**：[Mixtral论文](https://arxiv.org/abs/2401.04088)

### 3.4 上下文长度扩展 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - RoPE（旋转位置编码）：旋转矩阵、频率基、位置插值
  - ALiBi：线性偏置注意力，无需位置编码
  - 上下文扩展方法：位置插值（PI）、NTK-aware插值、YaRN
  - 长上下文优化：Sliding Window Attention、Ring Attention
- **与你经验关联**：理解"如何让模型处理更长的输入"，直接影响RAG和Agent设计
- **学习资源**：[RoPE原论文](https://arxiv.org/abs/2104.09864)

### 3.5 推理优化 ⭐⭐⭐ | 原理级 | 建议时间：4天
- **知识点**：
  - KV Cache：原理、PagedAttention（vLLM）、前缀缓存
  - Speculative Decoding：小模型草拟+大模型验证、投机采样
  - Continuous Batching：迭代级调度、动态批处理
  - Flash Attention：IO-aware注意力、减少HBM访问
  - 量化推理：FP8/INT8/INT4对KV Cache的影响
- **与你经验关联**：🔥 后端性能优化经验的直接迁移——缓存、批处理、预取
- **学习资源**：
  - [vLLM论文（PagedAttention）](https://arxiv.org/abs/2309.06180)
  - [Flash Attention论文](https://arxiv.org/abs/2205.14135)
  - [vLLM V1架构](https://docs.vllm.ai/en/latest/usage/v1_guide.html)

### 3.6 Tokenization ⭐⭐ | 坦白级 | 建议时间：1天
- **知识点**：
  - BPE（Byte Pair Encoding）：合并频率最高的字节对
  - SentencePiece：语言无关的分词器
  - Tokenizer对多语言/代码的影响
  - 特殊Token：`[CLS]`/`[SEP]`/`<|im_start|>`/`<|im_end|>`
- **与你经验关联**：了解即可，但要知道"token数直接影响成本和延迟"

---

## 4. 微调技术

### 4.1 Full Fine-tuning ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：全参数微调、灾难性遗忘、数据需求量大、计算成本高
- **与你经验关联**：了解概念即可，工程中更常用参数高效微调

### 4.2 LoRA / QLoRA ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - LoRA：低秩适配、矩阵分解（W = W₀ + BA）、秩r的选择
  - QLoRA：4-bit NormalFloat量化 + LoRA、双重量化、分页优化器
  - 优势：只训练0.1%-1%参数、可插拔适配器
  - 实战：使用PEFT库微调开源模型
- **与你经验关联**：面试高频题，理解"为什么低秩就够了"
- **学习资源**：
  - [LoRA论文](https://arxiv.org/abs/2106.09685)
  - [QLoRA论文](https://arxiv.org/abs/2305.14314)
  - [PEFT库](https://github.com/huggingface/peft)

### 4.3 RLHF / DPO / PPO ⭐⭐⭐ | 面试级 | 建议时间：3天
- **知识点**：
  - RLHF：奖励模型训练 → PPO强化学习对齐
  - DPO：直接偏好优化，无需奖励模型，简化对齐流程
  - PPO：近端策略优化，信任区域约束
  - ORPO / KTO 等新方法
- **与你经验关联**：面试必考，理解"如何让模型输出符合人类偏好"
- **学习资源**：
  - [InstructGPT论文](https://arxiv.org/abs/2203.02155)
  - [DPO论文](https://arxiv.org/abs/2305.18290)

### 4.4 数据准备与清洗 ⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：数据格式（Alpaca/ShareGPT）、数据质量过滤、去重、多样性、SFT数据构造
- **与你经验关联**：数据工程经验可以迁移

### 4.5 评估基准与指标 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：MMLU、HumanEval、GSM8K、C-Eval、MT-Bench、Arena Elo
- **与你经验关联**：理解"如何科学评估模型能力"

---

## 5. 量化与部署

### 5.1 量化 ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - 基础量化：FP16 → INT8 → INT4、对称/非对称量化
  - GPTQ：基于Hessian的逐层量化、校准数据集
  - AWQ：激活感知权重量化、保护salient weights
  - GGUF：llama.cpp格式、CPU/GPU混合推理
  - FP8：H100原生支持、训练+推理
  - **精度-性能权衡**：量化后质量损失评估
- **与你经验关联**：类比后端的"性能-正确性权衡"，量化就是"用精度换速度"
- **学习资源**：
  - [GPTQ论文](https://arxiv.org/abs/2210.17323)
  - [AWQ论文](https://arxiv.org/abs/2306.00978)
  - [llama.cpp](https://github.com/ggerganov/llama.cpp)

### 5.2 推理引擎 ⭐⭐⭐ | 实操级 | 建议时间：4天
- **知识点**：
  - **vLLM**（🔥重点）：
    - V1架构：统一调度器、零CPU开销
    - PagedAttention / KV Cache管理
    - Continuous Batching、Prefix Caching
    - Speculative Decoding
    - FP8 KV Cache
    - 多模态支持
    - OpenAI兼容API
  - **TensorRT-LLM**：NVIDIA官方、极致优化、编译期优化
  - **Ollama**：本地部署首选、GGUF支持、简单易用
  - **SGlang**：结构化生成、RadixAttention
- **与你经验关联**：🔥 核心工程能力，后端服务化经验直接迁移
- **学习资源**：
  - [vLLM官方文档](https://docs.vllm.ai/)
  - [Ollama官方文档](https://ollama.ai/)

### 5.3 模型服务化 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - OpenAI兼容API：/v1/chat/completions、流式输出
  - 模型路由：多模型调度、A/B测试
  - 负载均衡：round-robin / least-connections / 队列深度
- **与你经验关联**：微服务经验直接迁移，重点是"把LLM当微服务来管理"

### 5.4 GPU选型与成本 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - A100/H100：训练+推理、80GB显存、NVLink
  - 4090/3090：消费级推理、性价比高
  - L40/L4：推理优化、性价比
  - 云GPU：按需/预留/竞价实例成本对比
- **与你经验关联**：了解即可，面试中体现成本意识

---

# 三、应用侧（AI工程核心）

> 🔴 这是面试的核心考点，也是你转岗的最大差异点。本板块需要投入最多时间。

## 1. RAG（检索增强生成）

### 1.1 基础流程 ⭐⭐⭐ | 原理级 | 建议时间：2天
- **知识点**：切分 → 向量化 → 索引 → 检索 → 重排序 → 生成
- **完整架构图**：
  ```
  文档 → 解析 → 切分 → Embedding → 向量库
                                            ↓
  用户Query → Embedding → 向量检索 → 重排序 → LLM生成 → 回答
  ```
- **与你经验关联**：类比搜索引擎架构，理解"召回+排序"的思路
- **学习资源**：[LangChain RAG教程](https://python.langchain.com/docs/tutorials/rag/)

### 1.2 文档切分策略 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - 固定长度切分：简单但可能切断语义
  - 语义切分：基于Embedding相似度的断点检测
  - AST切分：代码文档按语法结构切分
  - 递归字符切分：按分隔符层级递归
  - 滑窗切分：chunk之间有overlap，保持上下文连续性
  - 元数据保留：页码、章节标题、来源
- **与你经验关联**：理解"如何让检索片段包含完整语义"
- **面试关键**：能讲清楚不同切分策略的优劣和适用场景

### 1.3 Embedding模型 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - bge-m3：多语言、多功能（Dense+Sparse+ColBERT）
  - OpenAI text-embedding-3-large/small：性价比高
  - Cohere embed-v3：搜索优化
  - E5/BGE系列：开源选择
  - **选择标准**：维度、性能、多语言支持、成本
- **与你经验关联**：理解"向量化就是特征提取"

### 1.4 向量数据库 ⭐⭐⭐ | 实操级 | 建议时间：3天
- **知识点**：
  - Qdrant：Rust实现、过滤+向量联合查询、gRPC API
  - Milvus：云原生、水平扩展、多种索引
  - Chroma：轻量级、快速原型
  - FAISS：纯库、高性能、无过滤能力
  - **选型维度**：数据规模、过滤需求、部署复杂度、语言SDK
- **与你经验关联**：数据库经验直接迁移，重点理解"向量索引"的特殊性
- **学习资源**：分别搭建实战项目对比

### 1.5 检索策略 ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - BM25：经典关键词检索、TF-IDF改进、适合精确匹配
  - 向量检索：语义相似度、适合模糊匹配
  - 混合检索：BM25 + 向量检索互补
  - RRF（Reciprocal Rank Fusion）：多路检索结果融合算法
  - **面试关键**：能讲清楚为什么混合检索优于单一检索
- **与你经验关联**：类比搜索的"召回+精排"架构
- **学习资源**：[RRF论文](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)

### 1.6 重排序 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - Cross-Encoder：对(query, doc)对编码、精确但慢
  - Reranker模型：bge-reranker、Cohere Rerank
  - LLM Rerank：用LLM判断文档相关性
  - 粗排→精排：向量检索粗排（top-100）→ Cross-Encoder精排（top-10）
- **与你经验关联**：类比搜索的"召回+排序"二级架构

### 1.7 高级RAG ⭐⭐⭐ | 面试级 | 建议时间：3天
- **知识点**：
  - **Self-RAG**：模型自决定是否检索、是否需要重检
  - **CRAG（Corrective RAG）**：检索结果评估 + 网络搜索兜底
  - **Agentic RAG**：Agent自主决定检索策略、多步检索
  - **Graph RAG**：知识图谱增强检索
  - **Modular RAG**：模块化组件组合
- **与你经验关联**：🔥 面试高频，体现对RAG前沿的理解
- **学习资源**：
  - [Self-RAG论文](https://arxiv.org/abs/2310.11511)
  - [CRAG论文](https://arxiv.org/abs/2401.15884)

### 1.8 文档解析 ⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - PDF解析：PyPDF/pdfplumber/Unstructured、双栏识别、跨页表格
  - Markdown解析：结构化保留、代码块/表格
  - OCR：PaddleOCR/Tesseract、扫描件处理
  - 多模态解析：用VLM直接"阅读"文档图片
- **与你经验关联**：了解即可，但要能讲清楚"文档解析是RAG的上游瓶颈"

### 1.9 文档去重 ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - 精确去重：SHA256/MD5哈希
  - 近似去重：MinHash + LSH、SimHash
  - 语义去重：向量相似度阈值
- **与你经验关联**：数据工程经验迁移

### 1.10 缓存策略 ⭐⭐⭐ | 原理级 | 建议时间：2天
- **知识点**：
  - 向量缓存：相同Query复用Embedding结果
  - Prompt缓存：Anthropic/OpenAI的Prompt Caching
  - LLM响应缓存：相同Query+Context复用回答
  - 缓存一致性：文档更新时的缓存失效策略
  - 命中率优化：语义缓存（相似Query也命中）
- **与你经验关联**：🔥 Redis缓存经验直接迁移！这是你的核心竞争力
- **面试关键**：能讲清楚AI场景下缓存与传统缓存的异同

---

## 2. Agent系统

### 2.1 Agent核心架构 ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - **LLM（大脑）**：推理、决策、生成
  - **Tool（手）**：外部工具调用、API执行
  - **Memory（记忆）**：短期/中期/长期记忆管理
  - **Plan（规划）**：任务分解、步骤编排
  - **Agent Loop**：感知→推理→行动→观察 的循环
- **与你经验关联**：🔥 系统设计经验直接迁移——Agent就是"有自主决策能力的微服务"
- **学习资源**：[LangChain Agent文档](https://python.langchain.com/docs/concepts/agents/)

### 2.2 ReAct范式及缺陷 ⭐⭐⭐ | 原理级 | 建议时间：2天
- **知识点**：
  - ReAct：Reasoning + Acting 交替进行
  - 流程：Thought → Action → Observation → Thought → ...
  - **缺陷**：
    - 线性执行，无法并行
    - 长链路容易累积错误
    - 无全局规划，容易"迷路"
    - Token消耗大（每步都需完整上下文）
- **与你经验关联**：面试必考，能对比ReAct和其他范式的优劣
- **学习资源**：[ReAct论文](https://arxiv.org/abs/2210.03629)

### 2.3 Plan-and-Execute ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - 先规划完整计划，再逐步执行
  - Planner（大模型）+ Executor（工具执行）
  - 可动态调整计划
  - 优于ReAct：有全局视野、减少无效探索
- **与你经验关联**：类比工作流引擎——先定义DAG，再逐步执行

### 2.4 LATS / Reflexion ⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - **LATS（Language Agent Tree Search）**：蒙特卡洛树搜索+LLM、多路径探索
  - **Reflexion**：自我反思、从失败中学习、经验记忆
- **与你经验关联**：了解前沿范式，面试中展示深度

### 2.5 多Agent协作模式 ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - **Supervisor模式**：一个主Agent协调多个子Agent
  - **Router模式**：根据任务类型路由到专业Agent
  - **Pipeline模式**：Agent链式处理，流水线式
  - **Parallel模式**：多Agent并行处理不同子任务
  - **Swarm模式**：Agent之间可以交接控制权
  - **黑板系统**：共享状态空间、Agent协作读写
- **与你经验关联**：🔥 类比微服务架构模式——每个Agent就是一个微服务
- **面试关键**：能设计一个多Agent系统并解释为什么选择某种协作模式
- **学习资源**：
  - [LangGraph文档](https://langchain-ai.github.io/langgraph/)
  - [CrewAI](https://github.com/joaomdmoura/crewAI)

### 2.6 MCP协议 ⭐⭐⭐ | 实操级 | 建议时间：3天
- **知识点**：
  - **MCP（Model Context Protocol）**：Anthropic提出的开放协议
  - 核心概念：Host（LLM应用）→ Client（连接器）→ Server（服务提供者）
  - 协议基础：JSON-RPC 2.0、状态连接、能力协商
  - Server三大能力：
    - Resources：提供数据/上下文
    - Tools：提供可执行工具
    - Prompts：提供模板化工作流
  - Client能力：Sampling（服务端发起的LLM调用）
  - 传输方式：Stdio / HTTP+SSE / Streamable HTTP
  - **2025-11-25新规范**：Task-based Workflows、简化授权流、Extensions
  - **用Go开发MCP Server**：mark3labs/mcp-go库
- **与你经验关联**：🔥 gRPC/Protobuf经验直接迁移！MCP就是"AI世界的gRPC"
- **学习资源**：
  - [MCP官方规范](https://modelcontextprotocol.io/specification)
  - [MCP Go SDK](https://github.com/mark3labs/mcp-go)

### 2.7 Skill vs Tool ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - Tool：原子操作，单次调用完成（如搜索、读文件）
  - Skill：复合能力，包含Prompt模板+工具组合+工作流
  - 类比：Tool是函数，Skill是类/模块
- **与你经验关联**：理解层次化抽象的设计思想

### 2.8 CLI vs MCP ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - CLI：命令行工具，通过Bash执行
  - MCP：标准化协议，结构化输入/输出
  - MCP的优势：类型安全、能力发现、权限管理、跨平台

### 2.9 Agent Loop vs ReAct ⭐⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - Agent Loop：while循环 + 工具调用 + 条件退出
  - ReAct：特殊的Agent Loop，强制的 Thought→Action→Observation 格式
  - Agent Loop更灵活，ReAct更结构化
- **与你经验关联**：理解"底层都是循环，区别在于循环内的结构约束"

---

## 3. Prompt Engineering

### 3.1 基础技术 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - Zero-shot：不给示例直接提问
  - Few-shot：提供少量示例引导格式/风格
  - Chain-of-Thought（CoT）：让模型"一步一步思考"
  - **面试关键**：能根据场景选择合适的Prompt策略
- **学习资源**：[Prompt Engineering Guide](https://www.promptingguide.ai/)

### 3.2 进阶技术 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - Self-Consistency：多次CoT采样+多数投票
  - Tree-of-Thought（ToT）：树状搜索，多路径探索+回溯
  - Directive Prompting：明确的指令约束（格式/角色/边界）
  - Re-reading：让模型重新阅读问题
  - Step-back Prompting：先抽象后具体
- **与你经验关联**：理解"Prompt就是新的编程语言"

### 3.3 System Prompt设计原则 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - 角色定义：你是谁、你的能力边界
  - 输出格式约束：JSON/XML/Markdown
  - 安全边界：不能做什么、拒绝策略
  - 示例驱动：包含输入/输出示例
  - **Claude Code的System Prompt设计**：~2800 tokens系统提示 + ~9400 tokens工具定义
  - 分层设计：身份层 → 能力层 → 约束层 → 示例层
- **与你经验关联**：类比API设计——接口定义、输入/输出规范、错误处理

### 3.4 上下文工程（Context Engineering） ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - 上下文窗口管理：Token预算分配、优先级排序
  - 上下文压缩：摘要、关键信息提取
  - 动态上下文：根据查询动态检索相关信息
  - 多轮对话上下文：历史消息管理、滑窗策略
  - **Claude Code的4级压缩策略**：Snip → MC → CC → AC（从轻到重）
- **与你经验关联**：🔥 类比内存管理——有限的资源如何高效利用
- **面试关键**：这是2025年最火的概念，必须能讲清楚

### 3.5 Harness Engineering（约束环境设计） ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - 为LLM构建结构化的执行环境
  - 输出格式约束、工具选择约束、行为边界约束
  - Fail-closed原则：默认不允许，显式授权
- **与你经验关联**：类比安全沙箱设计

### 3.6 Prompt版本管理与测试 ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：Prompt即代码、版本控制、A/B测试、回归测试
- **与你经验关联**：代码管理经验直接迁移

---

## 4. 向量检索

### 4.1 向量化流程 ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：文本 → Tokenization → Embedding模型 → 向量 → 索引

### 4.2 相似度计算 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：
  - 余弦相似度：方向相似性、归一化后等价于点积
  - 欧氏距离：空间距离
  - 内积（Dot Product）：未归一化的余弦
  - **选择原则**：归一化向量用余弦/内积，否则用欧氏距离
- **与你经验关联**：基础数学，理解直觉即可

### 4.3 HNSW算法原理 ⭐⭐⭐ | 原理级 | 建议时间：2天
- **知识点**：
  - 多层图结构：底层包含所有向量，上层是稀疏采样
  - 搜索过程：从顶层入口贪心搜索，逐层向下精确
  - 构建过程：逐点插入、启发式选择邻居
  - 参数：M（邻居数）、ef_construction（构建时搜索宽度）、ef_search（查询时搜索宽度）
  - 复杂度：构建O(n·log(n))，查询O(log(n))
- **与你经验关联**：类比跳表（Skip List）——多层索引加速查找
- **学习资源**：[HNSW论文](https://arxiv.org/abs/1603.09320)

### 4.4 ANN vs KNN ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - KNN：精确搜索，暴力遍历，O(n)
  - ANN：近似搜索，牺牲精度换速度，O(log(n))
  - ANN算法：HNSW、IVF、PQ（乘积量化）、LSH
  - **召回率-延迟权衡**

### 4.5 Qdrant实战 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - 集合创建、向量上传、索引构建
  - 搜索API：向量搜索 + 过滤条件
  - Payload索引：支持结构化+向量联合查询
  - 优化：量化配置、分片、别名
- **学习资源**：[Qdrant官方教程](https://qdrant.tech/documentation/)

### 4.6 bge-m3三模式 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - Dense：稠密向量，语义相似度
  - Sparse：稀疏向量（类似BM25），关键词匹配
  - ColBERT：交互式向量，token级匹配
  - 三模式联合：混合检索的终极方案
- **与你经验关联**：理解"不同粒度的检索有不同优势"

### 4.7 粗排与精排 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：
  - 粗排：向量检索top-K（K大，速度快）
  - 精排：Cross-Encoder/Reranker重排top-K（K小，精度高）
  - 经典漏斗：top-1000 → top-100 → top-10

### 4.8 LLM融合排序 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：用LLM对检索结果进行相关性判断和重排序
- **优缺点**：精度最高但成本也最高，适合关键场景

---

## 5. 记忆机制

### 5.1 短期记忆 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：
  - 上下文窗口管理：当前对话的完整历史
  - Token预算分配：System Prompt + 历史 + 工具输出 + 用户输入
  - 滑动窗口：保留最近N轮对话
- **与你经验关联**：类比内存——快速但容量有限

### 5.2 中期记忆 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - 会话摘要：用LLM总结历史对话
  - 关键信息提取：实体、关系、待办事项
  - **Claude Code的实现**：4级压缩（Snip→MC→CC→AC）
- **与你经验关联**：类比缓存——容量中等，需要淘汰策略

### 5.3 长期记忆 ⭐⭐⭐ | 原理级 | 建议时间：2天
- **知识点**：
  - 知识图谱：结构化存储实体和关系
  - 向量存储：语义检索历史经验
  - 文档存储：完整的项目文档、用户偏好
  - **Claude Code的记忆系统**：4类型+语义召回
- **与你经验关联**：类比数据库——持久化、可检索、容量大

### 5.4 记忆压缩 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - 滑动窗口：FIFO淘汰
  - 摘要压缩：LLM自动总结
  - 重要性评分：保留重要信息，丢弃琐碎内容
  - 向量去重：语义相似的记忆合并

### 5.5 跨会话记忆维护 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：会话间记忆持久化、用户级记忆隔离、记忆迁移

### 5.6 记忆生命周期管理 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：创建→激活→衰减→归档→删除、记忆遗忘曲线

---

## 6. 具体产品分析

### 6.1 Claude Code ⭐⭐⭐ | 原理级 | 建议时间：3天
- **知识点**：
  - **核心架构**：Agent Loop + 66+工具 + 记忆系统 + 安全系统
  - **主循环设计**：
    - 两层循环：外层QueryEngine（会话管理）+ 内层queryLoop（单轮执行）
    - AsyncGenerator连接，支持背压和中断传播
    - 隐式状态机：State结构体追踪所有状态
  - **消息压缩**：4级管线（Snip→MC→CC→AC），从轻到重
  - **工具执行**：StreamingToolExecutor——只读并行、写操作串行
  - **安全系统**：AST级别命令分析 + 23项静态检查 + 权限竞速
  - **记忆恢复**：会话状态持久化，重启后可恢复
  - **Token Budget**：nudge机制 + 递减收益检测
  - **MCP集成**：7种传输协议 + OAuth认证
- **与你经验关联**：🔥 这是目前最成熟的AI Agent产品，必须深入研究
- **学习资源**：
  - [Decoding Claude Code](https://www.anthropic.com/engineering/claude-code-best-practices)
  - Claude Code源码分析系列博客

### 6.2 Cursor/Windsurf ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - AI编程助手架构：编辑器集成 + LLM + 代码上下文
  - Cursor：Tab补全 + Chat + Composer、@codebase上下文
  - Windsurf：Cascade系统、Flow状态、Agent+Copilot混合
  - 关键设计：代码索引、AST感知、diff应用
- **与你经验关联**：理解"AI如何辅助编程"，面试中可以讨论设计思路

### 6.3 腾讯元宝/阿里高德地图AI ⭐ | 面试级 | 建议时间：1天
- **知识点**：了解国内AI产品的架构特点和工程挑战
- **与你经验关联**：面试中展示对行业的了解

---

# 四、工程侧（落地能力）

> 💡 本板块是你的优势区——后端工程经验直接转化为AI工程落地能力。

## 1. GPU部署

### 1.1 GPU选型 ⭐⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  | GPU | 显存 | 适用场景 | 参考价格（云） |
  |-----|------|----------|----------------|
  | H100 | 80GB | 大模型训练+推理 | $3-4/h |
  | A100 | 80GB | 大模型推理 | $2-3/h |
  | L40S | 48GB | 推理优化 | $1.5-2/h |
  | 4090 | 24GB | 消费级推理/微调 | $0.5-1/h |
  | 3090 | 24GB | 学习/原型 | $0.3-0.5/h |
- **与你经验关联**：了解成本，面试中体现工程意识

### 1.2 CUDA基础 ⭐ | 坦白级 | 建议时间：2天
- **知识点**：CUDA编程模型、Kernel/Block/Thread、显存层次、Stream
- **与你经验关联**：C++经验可迁移，但优先级低，了解即可

### 1.3 显存管理 ⭐⭐⭐ | 原理级 | 建议时间：2天
- **知识点**：
  - 显存组成：模型权重 + KV Cache + 激活值 + 临时缓冲
  - 显存估算：7B模型FP16≈14GB权重 + KV Cache
  - PagedAttention：虚拟内存管理思想，减少显存碎片
  - Offloading：GPU→CPU显存卸载
- **与你经验关联**：🔥 内存管理经验直接迁移！PagedAttention就是"虚拟内存分页"

### 1.4 多卡并行 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - **Tensor Parallelism（TP）**：单层内切分，需高带宽（NVLink）
  - **Pipeline Parallelism（PP）**：层间切分，可容忍较低带宽
  - **Data Parallelism（DP）**：数据并行，完全独立
  - **混合并行**：DP + TP + PP 组合
  - ZeRO优化：优化器状态/梯度/参数分片
- **与你经验关联**：分布式系统经验迁移——分片策略选择
- **学习资源**：[Megatron-LM论文](https://arxiv.org/abs/1909.08053)

### 1.5 云GPU vs 本地GPU ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：成本对比、弹性扩展、数据安全、延迟差异
- **与你经验关联**：云服务经验迁移

---

## 2. 模型服务化

### 2.1 Ollama部署与管理 ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：模型下载/运行、Modelfile自定义、API端点、多模型管理
- **学习资源**：[Ollama官方文档](https://ollama.ai/)

### 2.2 vLLM高并发推理 ⭐⭐⭐ | 实操级 | 建议时间：3天
- **知识点**：
  - **V1架构**（🔥 重点）：
    - 统一调度器：prefill + decode统一token budget
    - FCFS / 优先级调度
    - 前缀缓存（Prefix Caching）
    - 分块预填充（Chunked Prefill）
    - FP8 KV Cache
    - Speculative Decoding
    - LoRA动态加载
  - **vLLM 0.11.0**：
    - V0引擎完全移除，V1统一架构
    - CUDA Graph：FULL_AND_PIECEWISE默认启用
    - FlashInfer RoPE内核重写，2x提速
    - Q/K apply_rope融合，attention成本降11%
    - DeepGEMM默认启用，吞吐+5.5%
  - 部署：Docker/K8s、多GPU、多节点
- **与你经验关联**：🔥 高并发服务经验直接迁移
- **学习资源**：[vLLM官方文档](https://docs.vllm.ai/)

### 2.3 OpenAI兼容API ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - `/v1/chat/completions`：请求/响应格式
  - Function Calling / Tool Use
  - JSON Mode / Structured Output
  - 兼容性：vLLM/Ollama均支持

### 2.4 流式输出 ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - SSE（Server-Sent Events）：单向流、文本协议、简单可靠
  - WebSocket：双向流、二进制协议、更灵活
  - **AI场景常用SSE**：简单、兼容好、LLM生成是单向流
- **与你经验关联**：已有流式处理经验

### 2.5 模型热加载/版本切换 ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - vLLM睡眠模式：零重载模型切换
  - 灰度发布：新模型小流量验证
  - A/B测试：多模型对比
- **与你经验关联**：微服务部署策略迁移

### 2.6 健康检查与自动恢复 ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：GPU健康检查、OOM恢复、推理超时处理、自动重启

---

## 3. 缓存策略

### 3.1 向量缓存 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：相同Query的Embedding结果缓存、向量一致性
- **与你经验关联**：🔥 Redis缓存经验直接迁移

### 3.2 Prompt缓存 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：
  - Anthropic Prompt Caching：标记可缓存的System Prompt/工具定义
  - OpenAI Prompt Caching：自动缓存长prompt前缀
  - 节省：最高90%的输入token成本
- **与你经验关联**：类比HTTP缓存——不变的部分缓存，变化的部分动态

### 3.3 LLM响应缓存 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：
  - 精确缓存：相同Query+Context → 相同回答
  - 语义缓存：相似Query → 相似回答（向量相似度判断）
  - 缓存粒度：完整回答 vs 检索片段 vs 单步推理
- **与你经验关联**：🔥 这是最能体现你后端优势的领域

### 3.4 缓存一致性 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：
  - 文档更新 → Embedding重算 → 缓存失效
  - 知识库版本管理
  - 增量更新 vs 全量更新
- **与你经验关联**：Redis缓存一致性问题的高级版本

### 3.5 命中率优化 ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - 语义缓存：Embedding相似度阈值调优
  - 缓存预热：高频Query预加载
  - 缓存淘汰：LRU/LFU/基于重要性
  - 缓存穿透防护：空结果缓存

### 3.6 分层缓存 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：
  - L1 内存：最近使用的Prompt/响应，微秒级
  - L2 Redis：向量缓存、会话缓存，毫秒级
  - L3 向量库：语义检索缓存，十毫秒级
- **与你经验关联**：🔥 多级缓存架构设计，后端经典问题
- **面试关键**：能设计一个AI系统的完整缓存架构

---

## 4. 监控与运维

### 4.1 LLM调用监控 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - 延迟：TTFT（首Token延迟）、TPS（每秒Token数）、端到端延迟
  - Token消耗：输入/输出Token计数、按模型统计
  - 错误率：API错误、超时、内容过滤触发率
  - 工具调用监控：调用频率、成功率、平均耗时
- **与你经验关联**：🔥 可观测性经验直接迁移
- **实现**：Prometheus + Grafana仪表盘

### 4.2 成本控制 ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - Token计费模型：输入$/1M token、输出$/1M token
  - 缓存命中率 → 成本节省
  - 模型路由：简单问题用小模型（便宜）、复杂问题用大模型
  - 每日/每月成本预算和告警

### 4.3 A/B测试 ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - Prompt版本A/B测试
  - 模型版本A/B测试
  - 检索策略A/B测试
  - 统计显著性检验
- **与你经验关联**：后端A/B测试经验迁移

### 4.4 效果回归测试 ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - Prompt/模型变更后的效果验证
  - 自动化评估Pipeline
  - 黄金数据集
  - 回归阈值设定
- **与你经验关联**：CI/CD + 自动化测试经验迁移

### 4.5 红蓝对抗测试 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - Prompt注入攻击防御
  - 越狱测试
  - 数据泄露测试
  - 工具滥用测试
- **与你经验关联**：安全测试经验迁移

---

# 五、产品侧（业务能力）

> 💡 本板块是加分项，面试中展示产品思维和业务理解力。

## 1. 场景拆解

### 1.1 AI应用需求分析 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - 什么场景适合AI？什么不适合？
  - LLM能力边界：擅长生成/推理/理解，不擅长精确计算/实时性
  - 需求拆解：用户故事 → 功能点 → 技术方案
- **面试技巧**：能将业务需求转化为AI技术方案

### 1.2 技术可行性评估 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - 模型能力评估：能否胜任？准确率是否达标？
  - 延迟评估：用户可接受的响应时间
  - 成本评估：Token消耗 → 每次调用成本 → 月度成本
  - 数据评估：是否有足够的训练/检索数据？

### 1.3 MVP设计 ⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：最小可行产品、核心功能优先、快速验证

### 1.4 用户旅程设计 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：用户与AI系统的交互流程设计、多轮对话体验优化

---

## 2. 效果评估

### 2.1 LLM评估框架 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - 自动评估：BLEU/ROUGE/BERTScore/LLM-as-Judge
  - 人工评估：相关性/准确性/完整性/安全性
  - RAGAS框架：Faithfulness/Answer Relevancy/Context Recall/Context Precision
  - **面试关键**：能设计一个AI系统的评估方案
- **学习资源**：[RAGAS文档](https://docs.ragas.io/)

### 2.2 RAG评估 ⭐⭐⭐ | 实操级 | 建议时间：2天
- **知识点**：
  - 检索评估：召回率/精确率/MRR/nDCG
  - 回答评估：忠实度（是否基于检索内容）、相关性（是否回答了问题）
  - 端到端评估：用户满意度/问题解决率
- **与你经验关联**：搜索引擎评估经验迁移

### 2.3 Agent评估 ⭐⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - 任务完成率：能否完成指定任务
  - 工具调用准确率：是否选择了正确的工具和参数
  - 效率：完成任务的平均步数、Token消耗
  - 鲁棒性：面对异常输入的表现

### 2.4 人工评估 vs 自动评估 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：各自的优缺点、如何结合、LLM-as-Judge的局限性

### 2.5 评估基准与数据集 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：MMLU、HumanEval、SWE-bench、RAGBenchmark、AgentBench

---

## 3. 成本控制

### 3.1 Token成本计算与优化 ⭐⭐⭐ | 实操级 | 建议时间：1天
- **知识点**：
  - 各模型定价对比（GPT-4o/Claude/Qwen/DeepSeek）
  - 输入vs输出Token成本差异
  - 优化：Prompt精简、缓存、批量请求

### 3.2 模型选型 ⭐⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：
  - 大模型 vs 小模型：能力/成本/延迟三角
  - 通用模型 vs 专有模型：领域适配
  - 开源 vs 闭源：数据安全/成本/能力

### 3.3 混合路由 ⭐⭐⭐ | 面试级 | 建议时间：2天
- **知识点**：
  - 路由策略：基于复杂度分类 → 分流到不同模型
  - 分类器：规则/小模型LLM判断/Embedding相似度
  - 成本优化：80%简单问题用小模型、20%复杂问题用大模型
  - **vLLM语义路由**：v0.1 Iris版本
- **与你经验关联**：🔥 服务路由经验直接迁移
- **面试关键**：能设计一个多模型路由方案并估算成本节省

### 3.4 缓存策略对成本的影响 ⭐⭐⭐ | 原理级 | 建议时间：1天
- **知识点**：缓存命中率每提升10%，成本降低X%；语义缓存 vs 精确缓存

### 3.5 ROI计算 ⭐⭐ | 面试级 | 建议时间：1天
- **知识点**：AI系统的投入产出比计算、人力节省 vs API成本

---

# 六、面试专项

> 🔴 面试专项应与学习同步进行，边学边练。

## 1. 项目深度准备

### 1.1 华为云Redis/存储项目 ⭐⭐⭐ | 建议时间：2天
- **STAR框架**：
  - **S**：华为云Redis服务的规模和挑战
  - **T**：你负责的模块和目标
  - **A**：你采取的技术方案和决策
  - **R**：量化的成果（QPS提升、延迟降低、成本节省）
- **技术深挖点**：
  - Redis集群的一致性哈希实现
  - 主从复制的异步/同步选择
  - 热key检测与本地缓存方案
  - 大key拆分策略
  - 与AI场景的关联：Redis作为向量缓存/会话存储
- **难点与优化**：至少准备3个"踩过的坑"和解决方案

### 1.2 Leiman IM项目 ⭐⭐⭐ | 建议时间：2天
- **STAR框架**：同上
- **技术深挖点**：
  - 消息投递的可靠性保证
  - 在线状态管理
  - 群聊消息扇出策略
  - 与AI场景的关联：IM + AI = 智能客服/Agent通信
- **难点与优化**：长连接管理、消息时序保证

### 1.3 InterviewPro项目 ⭐⭐⭐ | 建议时间：2天
- **STAR框架**：同上
- **技术深挖点**：
  - 面试系统的核心架构
  - AI面试官的实现（RAG + Agent？）
  - 评估算法设计
  - 与AI场景的关联：直接相关！展示AI工程能力

---

## 2. 高频面试题

### 2.1 系统设计 ⭐⭐⭐ | 建议时间：持续练习

#### 设计一个RAG系统
- **思路**：
  1. 需求澄清：文档规模、并发量、延迟要求、准确率目标
  2. 架构设计：文档处理Pipeline → 向量库 → 检索服务 → LLM服务
  3. 关键决策：切分策略、Embedding模型、向量库选型、检索策略
  4. 优化：缓存、重排序、混合检索
  5. 运维：监控、评估、成本控制
- **加分点**：结合你的后端经验讨论缓存策略、可观测性

#### 设计一个Agent系统
- **思路**：
  1. 需求澄清：Agent类型、可用工具、协作模式
  2. 核心架构：Agent Loop + Tool Registry + Memory + Planner
  3. 关键决策：单Agent vs 多Agent、ReAct vs Plan-Execute
  4. 安全：权限控制、输入过滤、输出校验
  5. 可靠性：错误恢复、超时处理、状态持久化
- **加分点**：结合Claude Code的架构讨论设计取舍

### 2.2 算法 ⭐⭐⭐ | 建议时间：持续练习
- **LRU Cache**：哈希表+双向链表，O(1) get/put
- **一致性哈希**：虚拟节点、数据迁移最小化
- **倒排索引**：文档检索的基础、TF-IDF/BM25
- **HNSW**：多层图、贪心搜索
- **Top-K问题**：小顶堆/快速选择

### 2.3 AI专项 ⭐⭐⭐ | 建议时间：持续练习
- **RAG流程**：画完整架构图、解释每个环节的取舍
- **Agent架构**：对比ReAct/Plan-Execute/多Agent
- **Prompt优化**：实际案例、效果对比
- **向量检索**：HNSW原理、ANN vs KNN
- **LLM推理优化**：KV Cache、Speculative Decoding、Continuous Batching
- **MCP协议**：与gRPC的对比、为什么AI世界需要新协议
- **缓存策略**：AI场景下多级缓存设计

---

## 3. 面试策略

### 3.1 技术硬 + 讲得好 ⭐⭐⭐ | 建议时间：持续练习
- **公式**：问题 → 分析 → 方案 → 取舍 → 成果
- **示例**：
  > 面试官：如何优化RAG系统的检索准确率？  
  > 你：首先分析准确率低的可能原因（切分不合理/Embedding模型不够好/检索策略单一），  
  > 然后提出方案（混合检索+重排序），  
  > 解释取舍（BM25精确但缺语义、向量检索语义但可能偏题、混合检索两者兼顾但计算成本更高），  
  > 最后给出预期效果（准确率从70%提升到90%，延迟增加50ms，通过缓存可以优化）

### 3.2 心态管理 ⭐⭐ | 建议时间：持续
- 转岗面试≠零基础：你有14年工程经验，AI知识可以快速补
- 承认不会的：不会就说"这个我还在学习"，然后展示你的学习思路
- 主动关联：每个AI问题都尝试关联你的后端经验

### 3.3 英语面试准备 ⭐⭐ | 建议时间：持续
- AI术语的英文表达
- 技术方案的英文描述
- 常见面试题的英文回答练习

---

# 📅 推荐学习路线（12周计划）

| 周次 | 重点板块 | 具体任务 | 产出 |
|------|----------|----------|------|
| 1 | Python基础 | 语法速通+NumPy/Pandas+虚拟环境 | 能用Python写基本数据处理脚本 |
| 2 | 模型侧基础 | ML基础+深度学习基础+Transformer | 能画Transformer架构图并讲解 |
| 3 | LLM核心 | GPT演进+架构对比+MoE+上下文扩展 | 能对比主流LLM架构的优劣 |
| 4 | 推理优化+微调 | KV Cache+Speculative Decoding+LoRA/DPO | 能讲清楚推理优化的核心思想 |
| 5 | RAG核心 | 基础流程+切分+Embedding+向量库 | 搭建一个基础RAG系统 |
| 6 | RAG进阶 | 检索策略+重排序+高级RAG+缓存 | 优化RAG系统，提升准确率 |
| 7 | Agent核心 | Agent架构+ReAct+Plan-Execute+MCP | 实现一个简单Agent |
| 8 | Agent进阶 | 多Agent+记忆+Prompt Engineering | 设计一个多Agent系统 |
| 9 | 工程侧 | 量化+部署+模型服务化+缓存+监控 | 用vLLM部署模型服务 |
| 10 | Go进阶+后端复习 | Go 1.26新特性+Redis/MySQL深挖 | 更新后端知识到最新 |
| 11 | 面试专项 | 项目准备+系统设计+高频题 | 完成3个项目的STAR准备 |
| 12 | 模拟面试+查漏补缺 | 全真模拟+短板强化 | 准备好面试 |

---

# 📚 核心学习资源汇总

## 在线课程
- [吴恩达机器学习](https://www.coursera.org/learn/machine-learning) - ML基础
- [吴恩达深度学习](https://www.coursera.org/specializations/deep-learning) - DL基础
- [Stanford CS224N](http://web.stanford.edu/class/cs224n/) - NLP/LLM
- [Andrej Karpathy: Let's build GPT from scratch](https://www.youtube.com/watch?v=kCc8FmEb1nY) - Transformer实操
- [Prompt Engineering Guide](https://www.promptingguide.ai/) - Prompt工程

## 必读论文
- [Attention Is All You Need](https://arxiv.org/abs/1706.03762) - Transformer
- [LoRA](https://arxiv.org/abs/2106.09685) - 参数高效微调
- [PagedAttention/vLLM](https://arxiv.org/abs/2309.06180) - 推理优化
- [ReAct](https://arxiv.org/abs/2210.03629) - Agent范式
- [Self-RAG](https://arxiv.org/abs/2310.11511) - 高级RAG
- [DPO](https://arxiv.org/abs/2305.18290) - 对齐方法
- [HNSW](https://arxiv.org/abs/1603.09320) - 向量索引

## 实战项目
1. **搭建RAG系统**：Qdrant + bge-m3 + OpenAI API，处理真实PDF文档
2. **实现Agent框架**：Go + MCP，支持ReAct/Plan-Execute
3. **部署LLM服务**：vLLM + K8s + 监控，支持高并发推理
4. **开发MCP Server**：用Go开发一个MCP Server，连接现有后端服务
5. **构建多级缓存**：Redis + 向量缓存 + Prompt缓存，优化AI系统成本

## 社区与资讯
- [Hugging Face](https://huggingface.co/) - 模型/数据集/教程
- [vLLM Blog](https://blog.vllm.ai/) - 推理优化前沿
- [LangChain Blog](https://blog.langchain.dev/) - AI工程实践
- [MCP官方](https://modelcontextprotocol.io/) - MCP协议
- [Go Blog](https://go.dev/blog/) - Go语言更新

---

> 💪 **最后的话**：14年后端经验不是包袱，而是你最大的武器。AI工程的核心挑战不是算法创新，而是**把模型能力转化为可靠的工程系统**——这正是你最擅长的。加油！
