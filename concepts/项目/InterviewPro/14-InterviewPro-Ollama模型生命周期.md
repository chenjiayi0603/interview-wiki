# Ollama 模型生命周期

## 架构概览

```
┌──────────────────────────────────────────────────┐
│  K8s Pod: ollama (Deployment, 1 replica)          │
│  runtimeClassName: nvidia                         │
│                                                   │
│  ┌─────────────┐      ┌──────────────────┐        │
│  │ Ollama 服务  │◄─────│  Backend (Go)     │        │
│  │ :11434      │      │                  │        │
│  │             │      │ AI_PROVIDER=     │        │
│  │ GPU: RTX    │      │  deepseek        │        │
│  │ 4070 12GB   │      │  qwen_local      │        │
│  └──────┬──────┘      └──────────────────┘        │
│         │                                         │
│  ┌──────▼──────┐                                   │
│  │ ollama-pvc  │  /root/.ollama/models/           │
│  │   50Gi       │  ├── bge-m3/         (1.2GB)     │
│  │              │  └── qwen3-scoring/  (7.8GB)     │
│  └─────────────┘                                   │
└──────────────────────────────────────────────────┘
```

GPU 显存 12GB，两个模型不能同时常驻（7.8 + 1.2 = 9GB，勉强可以但会竞争）。

## 模型清单

| 模型 | GGUF 大小 | GPU VRAM | 系统 RAM | 用途 | 触发路径 | 当前状态 |
|------|----------|----------|----------|------|----------|----------|
| `bge-m3` | 1.2GB | ~1.2GB | ~0.5GB | 向量化 (embedding) | 面试题检索 / 简历解析触发 | 延迟加载 |
| `qwen3-scoring` | 7.8GB | ~7.8GB | ~1GB | 本地评分 | 仅 `AI_PROVIDER=qwen_local` 时触发 | 已卸载 |

> **内存分工**：GGUF 文件存磁盘 (ollama-pvc)，推理时加载到 GPU VRAM（模型权重主体），系统 RAM 只做缓冲缓存和 Ollama 进程开销。RTX 4070 12GB 显存，两个模型同时加载约 9GB，够用。

### GPU 显存拆解

qwen3-scoring 加载后占 7.8GB，拆开看：

```
qwen3-scoring 显存占用 (7.8GB)
  ├── 模型权重 (4B × Q4_K_M 4-bit)   ~2.4GB   ← 只存一份，并发共享
  ├── 上下文预分配缓冲区               ~5GB     ← llama-server 为 262k 上下文预占
  └── 运行时 (激活值/缓冲区)           ~0.5GB
```

**预分配缓冲区不等于真实使用**。token 少时大部分是空的。

KV cache 实际大小（单路）：

```
qwen3-scoring: 4B 参数，hidden=2560，36 层，GQA 8 KV heads

单 token 的 K: 8 heads × 80 dim × 2B (fp16) = 1280B
单 token 的 V: 同上                            = 1280B
36 层 × 2560B × 1 token ≈ 90 KB/token

理论最大 (262k token): ~23 GB (fp16) / ~12 GB (Q8_0)
面试场景 (4000 token):   ~350 MB
面试场景 (1000 token):   ~90 MB
```

**结论**：KV cache 本身不大（100-350MB），7.8GB 里的大头是预分配缓冲区。并发时：

```
并发 1:  权重 2.4GB + 缓冲 5GB + KV ~350MB    = ~7.8GB  ✓
并发 4:  权重 2.4GB + 缓冲 5GB + KV ~350MB×4  = ~8.8GB  ✓ 12GB 够
```

面试场景上下文短，4 路并发在 12GB 显存内完全够用。预分配缓冲区的 5GB 已经在容器启动时占好了，不会因为并发增加而膨胀。

### 并发瓶颈

**三层限制，实际并发 = min(三层)**

```
请求到达
  │
  ▼
┌─────────────────────────────┐
│ ① Go 信号量                  │  QWEN_MAX_PARALLEL=4
│    sem chan, 4 槽            │  满了阻塞等
│    当前：4                    │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ ② Ollama --parallel          │  OLLAMA_NUM_PARALLEL
│    llama-server 并发槽        │  默认 1 ← 当前瓶颈！
│    未设                       │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ ③ GPU 显存                   │  4.6GB 可用 ÷ 350MB/路
│    硬上限 ~13 路              │  超了 OOM
│    当前余量够，不是瓶颈        │
└─────────────────────────────┘
```

当前实际并发 = min(4, 1, 13) = **1 路**。要提升，把 `OLLAMA_NUM_PARALLEL=4` 设到 ollama-deployment.yaml。

#### 显存硬上限（物理决定，超了 OOM）

固定开销：权重 2.4GB + 预分配缓冲 5GB = 7.4GB。剩余 4.6GB 给多路 KV cache。

| 上下文长度 | 每路 KV cache | 显存硬上限 (4.6GB) |
|-----------|-------------|-------------------|
| 1000 token | ~90MB | ~50 路 |
| 2000 token | ~180MB | ~25 路 |
| 4000 token | ~350MB | ~13 路 |

#### GPU 算力上限（算出来的）

RTX 4070：504 GB/s 显存带宽，29 TFLOPS FP16 算力。

**两阶段**：输入 prefill（带宽 bound），输出 decode（batch 小时带宽 bound，batch 大时算力 bound）。

```
输入 prefill: 一次性读完所有 input token
  ≈ 50000 tok/s  不受 batch 影响，高度并行

输出 decode: 逐 token 生成，每 token 读一次全部权重
  1 路: 504 GB/s ÷ 2.4 GB = 210 tok/s   ← 带宽受限
  8 路: 29T ÷ (4B×2) ≈ 3600 tok/s 总   ← 算力受限 (compute bound)
```

**continuous batching**：权重只读一次，batch 塞几路同时产出。batch 小时每路分得多但总吞吐低，batch 大到算力用满后总吞吐不变，单路变慢：

```
batch=1:  总吞吐 210 tok/s，每路 210 tok/s
batch=4:  总吞吐 840 tok/s，每路 210 tok/s (还在带宽 bound)
batch=8:  总吞吐 3600 tok/s，每路 450 tok/s (算力用满)
batch=13: 总吞吐 3600 tok/s，每路 277 tok/s (算力上限，单路开始降)
```

**面试请求端到端时间**：

```
单请求: 输入 prefill + 输出 decode

出题 (300入+30出):
  batch=1:  300÷50000 + 30÷210  = 6ms + 143ms = 149ms
  batch=8:  300÷50000 + 30÷450  = 6ms + 67ms  = 73ms

五维评分 (400入+300出):
  batch=1:  400÷50000 + 300÷210 = 8ms + 1429ms = 1437ms ← 太慢
  batch=8:  400÷50000 + 300÷450 = 8ms + 667ms  = 675ms

会话反馈 (2000入+500出):
  batch=1:  2000÷50000 + 500÷210 = 40ms + 2381ms = 2421ms
  batch=8:  2000÷50000 + 500÷450 = 40ms + 1111ms = 1151ms
```

**总 QPS**（batch=8，算力用满）：

```
输入能力: ~50000 tok/s，输出能力: ~3600 tok/s
QPS = min(输入能力 ÷ 输入 token, 输出能力 ÷ 输出 token)

出题:      min(50000÷300, 3600÷30)   = min(166, 120) = 120 req/s  瓶颈: 输出
五维评分:  min(50000÷400, 3600÷300)  = min(125, 12)  = 12 req/s   瓶颈: 输出
会话反馈:  min(50000÷2000, 3600÷500) = min(25, 7)    = 7 req/s    瓶颈: 输出
```

| 请求类型 | 输入/输出 token | 输入耗时 | 输出耗时 | 总耗时 | QPS |
|----------|---------------|---------|---------|--------|-----|
| 出题 | 300+30 | 6ms | 67ms | 73ms | 120 |
| 五维评分 | 400+300 | 8ms | 667ms | 675ms | 12 |
| 会话反馈 | 2000+500 | 40ms | 1111ms | 1151ms | 7 |

三种请求全是输出瓶颈，输入占比可以忽略。

QPS = 总输出 3600 ÷ 输出 token，跟并发几路无关。并发只影响单路延迟。

- **显存**：硬上限，超了 OOM，RTX 4070 12GB
- **GPU 算力**：决定总输出 3600 tok/s 和 QPS，batch 小则带宽 bound 更慢
- **CPU**：不参与推理，不是瓶颈

## 加载机制

### 代码调用链

```
AI_PROVIDER=qwen_local
      │
      ▼
factory.go:92  GetModelProviderFromConfig()
      │  case "qwen_local"
      ▼
qwen_model.go:35  NewQwenLocalModel(
      apiURL      = "http://localhost:11434/v1/chat/completions"
      modelID     = "qwen3-scoring"
      temperature = 0.1
      maxTokens   = 3000
      maxParallel = 4
)
      │
      ▼  用户开始面试，后端调用评分
qwen_model.go:168  POST http://localhost:11434/v1/chat/completions
      body: {
        "model": "qwen3-scoring",
        "messages": [...],
        "temperature": 0.1,
        "max_tokens": 3000
      }
      │
      ▼  Ollama 收到请求
┌─────────────────────────────────────────────┐
│ 1. 检查 model="qwen3-scoring" 是否在显存    │
│    ├── 在 → 直接推理，返回结果               │
│    └── 不在 ↓                               │
│ 2. 从 /root/.ollama/models/ 读取 GGUF       │
│    （路径由 ollama-pvc 提供持久化）           │
│ 3. 加载模型权重到 GPU VRAM                   │
│ 4. 推理 → 返回结果                           │
│ 5. OLLAMA_KEEP_ALIVE=-1 → 常驻显存           │
└─────────────────────────────────────────────┘
```

### bge-m3 的加载路径（独立于 AI_PROVIDER）

```
embedding.go → POST http://localhost:11434/api/embeddings
body: {"model": "bge-m3", "input": "..."}
       │
       ▼  Ollama 同上流程
从 ollama-pvc 读 GGUF → 加载到 GPU → 返回 1024 维向量
```

bge-m3 不管 `AI_PROVIDER` 设什么都会用，只要向量检索功能触发（面试出题、简历解析等）。

### OLLAMA_KEEP_ALIVE 控制行为

| 值 | 行为 |
|----|------|
| `-1` | 加载后永不卸载（当前配置） |
| `0` | 请求完成后立即卸载 |
| `5m` | 闲置 5 分钟后自动卸载 |
| `1h` | 闲置 1 小时后自动卸载 |

当前 `OLLAMA_KEEP_ALIVE=-1` 的含义：
- **已加载的模型** → 永远留在显存
- **未加载的模型** → 不会自动加载，等首次请求触发

## 卸载机制

模型不会自动卸载（因为 `-1`）。只有以下方式：

### 方式一：CLI 卸载
```bash
# 在 Ollama 容器内
kubectl exec -n interviewpro deploy/ollama -- ollama stop qwen3-scoring
```

### 方式二：API 卸载
```bash
# 发一个 keep_alive=0 的请求，完成后立即卸载
curl -X POST http://localhost:11434/api/generate \
  -d '{"model":"qwen3-scoring","keep_alive":0,"prompt":"unload"}'
```

### 方式三：容器重启
```bash
kubectl rollout restart deploy/ollama -n interviewpro
# 重启后显存清空，两个模型都回到磁盘
```

## AI_PROVIDER 切换指南

### deepseek → qwen_local（启用本地评分）

```bash
# 1. 修改 .env
#    AI_PROVIDER=qwen_local
#    （AI_API_KEY 和 DEEPSEEK_* 可保留，不影响）

# 2. 预热模型（可选，避免首次请求卡顿）
curl -X POST http://localhost:11434/api/generate \
  -d '{"model":"qwen3-scoring","keep_alive":-1,"prompt":"warmup"}'

# 3. 重启 backend 使配置生效（如果是 K8s 部署）
kubectl rollout restart deploy/backend -n interviewpro
```

预热后 GPU 显存占用约 7.8GB（qwen3-scoring）+ 1.2GB（如果 bge-m3 也已加载）≈ 9GB。

### qwen_local → deepseek（切回云端）

```bash
# 1. 修改 .env
#    AI_PROVIDER=deepseek

# 2. 重启 backend
kubectl rollout restart deploy/backend -n interviewpro

# 3. 卸载本地模型释放显存（可选）
kubectl exec -n interviewpro deploy/ollama -- ollama stop qwen3-scoring
```

## 冷启动延迟

| 模型 | GGUF 大小 | 首次加载耗时 | 说明 |
|------|-----------|-------------|------|
| `bge-m3` | 1.2GB | 5-15 秒 | 首次 embedding 请求触发，之后常驻 |
| `qwen3-scoring` | 7.8GB | 30-60 秒 | 首次评分请求触发，之后常驻 |

延迟取决于磁盘 I/O（ollama-pvc 所在磁盘速度）和 GPU 带宽。

## 日常管理命令

```bash
# 查看当前加载的模型
kubectl exec -n interviewpro deploy/ollama -- ollama list
# 输出示例：
# NAME            ID              SIZE      PROCESSOR    UNTIL
# bge-m3:latest   xxx             1.2 GB    100% GPU     4 minutes from now

# 查看已下载的模型（含未加载的）
kubectl exec -n interviewpro deploy/ollama -- ollama list
# （同上命令，已下载但未加载的也会列出）

# 查看 GPU 显存使用
kubectl exec -n interviewpro deploy/ollama -- nvidia-smi

# 手动加载模型到 GPU
kubectl exec -n interviewpro deploy/ollama -- ollama run qwen3-scoring "warmup"

# 手动卸载模型
kubectl exec -n interviewpro deploy/ollama -- ollama stop qwen3-scoring

# 查看 Ollama 日志
kubectl logs -n interviewpro deploy/ollama
```

## 配置参考

### .env 相关变量

```bash
# 选择 AI 后端
AI_PROVIDER=deepseek            # 云端 DeepSeek API（当前）
# AI_PROVIDER=qwen_local        # 本地 Ollama qwen3-scoring

# Ollama 服务地址（仅 qwen_local 时使用）
QWEN_LOCAL_URL=http://localhost:11434/v1/chat/completions
QWEN_LOCAL_MODEL=qwen3-scoring
QWEN_TEMPERATURE=0.1
QWEN_MAX_TOKENS=3000
QWEN_MAX_PARALLEL=4
```

### K8s 相关配置（ollama-deployment.yaml）

```yaml
env:
- name: OLLAMA_KEEP_ALIVE
  value: "-1"           # 永不卸载
resources:
  limits:
    nvidia.com/gpu: 1   # 占用 1 个 GPU
    memory: 6Gi         # 内存上限
```

## 为什么用 bge-m3（对比 Elasticsearch）

### bge-m3 的核心优势

bge-m3 由 BAAI（北京智源）训练，**原生支持中英双语**，100+ 语言，1024 维向量，max 8192 token。MIT 协议，免费商用。正好匹配本项目场景：中国用户练英语面试，query 经常是中英混合（"tell me about 团队合作"）。

### MTEB 基准评分

| 模型 | MTEB 总分 | 部署方式 | 费用 |
|------|----------|----------|------|
| OpenAI text-embedding-3-large | 64.6 | API 云端 | $0.13/百万 token |
| **bge-m3** | **63.0** | 本地 Ollama | 免费 |
| Nomic-embed-text v1.5 | 59.4 | 本地 | 免费 |
| all-MiniLM-L6-v2 | 56.3 | 本地 | 免费 |

bge-m3 跟 OpenAI 最强商用 embedding 模型只差 1.6 分，完全免费。在开源模型中属于第一梯队。C-MTEB（中文基准）上 multilingual retrieval 子项长期排前列，中英混合 query 不偏科。

### 本项目实测性能

| 场景 | 延迟 | 说明 |
|------|------|------|
| 冷启动加载 | 5-15 秒 | GGUF 1.2GB 从 PVC → GPU，仅首次 |
| 常驻后单条 embedding | 50-200ms | GPU 推理，1024 维输出 |
| 串行批量 (10 条) | 0.5-2 秒 | EmbedBatch 逐条调 `/api/embeddings`，无并行 |
| GPU 显存占用 | ~1.2GB | RTX 4070 12GB，占比 10% |
| 系统 RAM 额外占用 | ~0.5GB | 进程开销 + 缓存 |

### 并发分析

**现状**：EmbeddingService (`embedding.go`) 没有并发控制——无信号量、无连接数限制、无速率限制。Go HTTP 服务器为每个请求起一个 goroutine，多个用户同时面试时，Embed 调用会并发打到 Ollama。

**Ollama GPU 推理**：CUDA 一次只能处理一个请求，其余排队。排队延迟 = 队列长度 × 单条推理时间。

| 并发请求数 | P50 延迟 | P99 延迟 | 说明 |
|-----------|---------|---------|------|
| 1 | 50-200ms | 200ms | 无排队，单路推理 |
| 3 | 100-400ms | 600ms | 队首 ~50ms，队尾 ~400ms |
| 5 | 150-600ms | 1 秒 | 队尾等前面 4 个完成 |
| 10 | 300ms-2 秒 | 2 秒+ | 线性退化，无背压 |

**瓶颈对比**：

| 组件 | 并发控制 | 并发度 |
|------|---------|--------|
| `QwenLocalModel` (LLM) | 信号量 `sem chan struct{}`，默认 4 槽 | 有上限，匹配 `llama-server --parallel` |
| `EmbeddingService` (embedding) | **无** | 无上限，全靠 Ollama 内部排队 |

### ES 并发对比

| | bge-m3 + Qdrant | Elasticsearch |
|---|---|---|
| **并发模型** | GPU 单流，请求排队；Go 侧无视信号量 | Java 线程池 (search thread pool)，默认 50-100 线程 |
| **推理并发** | 1 个/次（CUDA 串行），队列堆积时 P99 线性恶化 | CPU 多线程并行推理，并发度 = CPU 核数 |
| **检索并发** | Qdrant gRPC，服务端内部排队 | ES shard 级并行，单个查询可跨多个 Lucene 段并行 |
| **单条延迟** | 50-200ms (GPU) | 取决于模型，CPU 推理通常 100-500ms |
| **10 并发 P99** | ~2 秒（GPU 排队） | ~500ms-1 秒（线程池分摊） |
| **100 并发** | 不可用（无客户端限流，队列雪崩） | 稳定（内置队列 + 背压），P99 ~2 秒 |
| **扩容方式** | 加 GPU 或换多卡（如 RTX 4090 24GB） | 加节点水平扩展，自动分片均衡 |
| **运维成本** | 并发优化需改代码（加信号量） | ES 自带并发治理，开箱即用 |

ES 并发远高于此方案——线程池 + shard 并行 + 背压，10 年生产打磨。

**并发量级**：ES ~100-200 并发搜索，bge-m3 ~5-10 并发，差 ~20 倍。

### 当前检索架构

```
用户 query
    │
    ├──→ Qdrant 向量检索 (bge-m3 1024 维, limit 30, score ≥ 0.6)
    │         │        延迟 10-50ms (gRPC)
    │
    ├──→ PostgreSQL 关键词检索 (LIKE/ILIKE, limit 40)
    │         │        延迟 5-20ms (B-tree + GIN 索引)
    │
    └─────────┴──→ RRF 融合 (k=60) → 排序返回
                          延迟 <5ms (纯内存计算)
```

总检索延迟（不含 embedding）：**20-70ms**。三组件各司其职，没有冗余。

### bge-m3 + Qdrant vs Elasticsearch

| | bge-m3 + Qdrant | Elasticsearch (multilingual-e5) |
|---|---|---|
| **英文检索 (MTEB)** | 63.0 | 取决于模型，multilingual-e5 ~60 |
| **中文检索 (C-MTEB)** | 前列，中英混合 query 不偏科 | 中文弱于英文，混合 query 易偏向 |
| **向量推理** | 50-200ms (GPU) | 取决于模型部署，CPU 推理更慢 |
| **检索延迟** | Qdrant gRPC 10-50ms | ES REST/JSON 20-100ms |
| **总吞吐** | ~50-100 QPS（单 GPU） | ~20-50 QPS（CPU 推理） |
| **内存占用** | Qdrant ~200MB + bge-m3 ~1.7GB | ES JVM 2-4GB + embedding 模型 |
| **磁盘 IO** | 向量索引 + PG 关键词索引分离 | 统一索引，Lucene 段合并有 IO 尖峰 |
| **混合检索** | 应用层 RRF 融合，可控透明 | ES 内置 KNN + BM25，黑盒调参 |
| **运维** | 3 个轻量进程，K3s 单机无压力 | 1 个重量级 Java 进程，需调堆和 GC |

### 结论

ES 能做混合检索，但本项目已经有 Qdrant（向量）+ PostgreSQL（关键词）+ RRF（融合），加 ES 等于引入一个重型重复组件。bge-m3 在效果上跟 OpenAI 商用模型接近，性能上 GPU 推理 50-200ms 足够交互场景，运维成本远低于 ES。对本项目这个中英混合场景，bge-m3 是最优选择。

## 常见问题

**Q: 改 AI_PROVIDER 后要重启 Ollama 吗？**
不用。Ollama 是无状态的，改 backend 的 `.env` 后只需重启 backend。

**Q: 两个模型能同时加载吗？**
可以。bge-m3 (1.2GB) + qwen3-scoring (7.8GB) ≈ 9GB，RTX 4070 有 12GB 显存，够用。

**Q: 为什么 bge-m3 不管 AI_PROVIDER 设什么都会加载？**
bge-m3 是向量化模型，用于面试题检索和简历解析，跟 LLM 评分走不同路径。它的触发由 `embedding.go` 控制，不经过 AI factory。

**Q: 模型从磁盘加载到 GPU 的过程是什么？**
Ollama 读取 `/root/.ollama/models/` 下的 GGUF 格式文件（量化后的模型权重），通过 CUDA 将权重拷贝到 GPU 显存。GGUF 是 llama.cpp 的文件格式，已经做了 INT8/INT4 量化，加载比原始 safetensors 快很多。
