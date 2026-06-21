# 美团 / 百度 · 搜索 + 推荐 + LBS 方向

> 适用：美团 / 百度 / 滴滴
> 核心语言：C++ / Go

---

## 一、公司特色考点

| 公司 | 核心业务 | 面试侧重点 |
|------|---------|-----------|
| **美团** | 本地生活/外卖 | GeoHash、配送调度、UV 统计、系统设计 |
| **百度** | 搜索/AI/自动驾驶 | 倒排索引、搜索引擎、Trie 树、Embedding |
| **滴滴** | 出行 | 订单匹配、ETA 预测、GeoHash、实时路况 |

> **通用考点**统一在 `05-通用考点/`，此处不再重复

---

## 二、美团特色题

### Q1: GeoHash 怎么实现附近查询？

#### 原理

```
经纬度 → 二进制(交替合并) → base32 编码 → 字符串
前缀相同 → 距离越近
长度:  6 位 ≈ 1km,  5 位 ≈ 5km,  4 位 ≈ 20km
```

#### 完整实现

```go
// GeoHash 编码实现
package geo

import (
    "math"
    "strings"
)

var base32 = "0123456789bcdefghjkmnpqrstuvwxyz"

// Encode 经纬度 → GeoHash 字符串
func Encode(lat, lng float64, precision int) string {
    latRange := [2]float64{-90, 90}
    lngRange := [2]float64{-180, 180}
    var bits []byte

    // 交替二分逼近，每次取 1 bit
    for len(bits) < precision*5 {
        // 经度
        mid := (lngRange[0] + lngRange[1]) / 2
        if lng >= mid {
            bits = append(bits, '1')
            lngRange[0] = mid
        } else {
            bits = append(bits, '0')
            lngRange[1] = mid
        }
        // 纬度
        mid = (latRange[0] + latRange[1]) / 2
        if lat >= mid {
            bits = append(bits, '1')
            latRange[0] = mid
        } else {
            bits = append(bits, '0')
            latRange[1] = mid
        }
    }

    // 每 5 bit 转 base32
    var result strings.Builder
    for i := 0; i < len(bits); i += 5 {
        idx := 0
        for j := 0; j < 5; j++ {
            if i+j < len(bits) && bits[i+j] == '1' {
                idx |= 1 << (4 - j)
            }
        }
        result.WriteByte(base32[idx])
    }
    return result.String()[:precision]
}

// FindNearbyRiders 查找附近骑手
func FindNearbyRiders(db *sql.DB, lat, lng float64, radiusKm float64) ([]Rider, error) {
    // 精度选择: 根据半径确定 GeoHash 长度
    precision := 6  // 默认 ~1km
    switch {
    case radiusKm > 20:
        precision = 4
    case radiusKm > 5:
        precision = 5
    default:
        precision = 6
    }

    prefix := Encode(lat, lng, precision)

    // MySQL LIKE 查询
    rows, err := db.Query(
        "SELECT id, name, lat, lng, geohash, "+
            "ST_Distance_Sphere(point(lng, lat), point(?, ?)) AS dist "+
            "FROM riders WHERE geohash LIKE ? AND status = 'online' "+
            "HAVING dist < ? ORDER BY dist LIMIT 50",
        lng, lat, prefix+"%", radiusKm*1000,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var riders []Rider
    for rows.Next() {
        var r Rider
        rows.Scan(&r.ID, &r.Name, &r.Lat, &r.Lng, &r.GeoHash, &r.Distance)
        riders = append(riders, r)
    }
    return riders, nil
}
```

#### 追问: GeoHash 边界跳跃问题

```
问题: 两个很近的点在 GeoHash 网格边界，前缀不同

方案:
  1. 查周围 8 个网格的 GeoHash（当前 + 东 + 西 + 南 + 北 + 4 个角）
  2. Redis GEO 底层是 GeoHash + Sorted Set（ZRANGEBYSCORE）
  3. 美团实际: GeoHash 粗筛 → Redis GEO 精确距离排序

Redis GEO 实现:
  GEOADD riders:online 116.397 39.908 "rider_1001"
  GEORADIUS riders:online 116.397 39.908 5 km WITHDIST COUNT 50
```


#### 面试话术

```
GeoHash 的本质是把二维经纬度转成一维字符串。
查询附近骑手时，先用 GeoHash 前缀匹配粗筛（MySQL LIKE），
再用 Redis GEO 精确排序。注意边界跳跃问题：
两个点可能在网格边界，前缀不同但实际很近，
所以要查周围 8 个网格。美团实战中 GeoHash 定位到 1km 精度，
再配合 Redis 精确距离排序，性能从 2s 降到 200ms。
```
### Q2: 亿级 UV 怎么统计？

#### 多层级方案

```
实时看板 → HyperLogLog（12KB 存 2^64，误差 0.81%）
日报/周报 → Bitmap（1 亿用户 ≈ 12MB/天）
多维分析 → ClickHouse（用户分群、留存分析）
```

#### 代码实现

```go
// HyperLogLog + Bitmap 双层 UV 统计
type UVService struct {
    rdb *redis.Client
}

// PageView 用户访问（QPS 10 万+）
func (s *UVService) PageView(ctx context.Context, pageID string, userID int64) error {
    pipe := s.rdb.Pipeline()

    // 1. HyperLogLog: 实时 UV（12KB，误差 0.81%）
    pipe.PFAdd(ctx, "uv:hll:"+pageID+":"+today(), userID)

    // 2. Bitmap: 每日用户活跃（1 天 1 个 key）
    pipe.SetBit(ctx, "uv:bitmap:"+pageID+":"+today(), userID, 1)

    // 3. 同时发送到 Kafka 做离线分析
    // ...
    _, err := pipe.Exec(ctx)
    return err
}

// GetRealtimeUV 实时 UV（秒级更新）
func (s *UVService) GetRealtimeUV(ctx context.Context, pageID string) (int64, error) {
    result, err := s.rdb.PFCount(ctx, "uv:hll:"+pageID+":"+today()).Result()
    return result, err
}

// GetDailyUV 精确日活（Bitmap 精确统计）
func (s *UVService) GetDailyUV(ctx context.Context, pageID, date string) (int64, error) {
    // BITCOUNT 统计 1 的个数
    result, err := s.rdb.BitCount(ctx, "uv:bitmap:"+pageID+":"+date, nil).Result()
    return result, err
}

// GetWeeklyUV 周活合并
func (s *UVService) GetWeeklyUV(ctx context.Context, pageID string) (int64, error) {
    // 将 7 天的 Bitmap 做 OR 合并
    destKey := "uv:weekly:" + pageID
    keys := make([]string, 7)
    for i := 0; i < 7; i++ {
        keys[i] = "uv:bitmap:" + pageID + ":" + dateNDaysAgo(i)
    }
    s.rdb.BitOpAnd(ctx, destKey, keys...)
    return s.rdb.BitCount(ctx, destKey, nil).Result()
}

// 空间对比:
// 1 亿用户:
//   HyperLogLog: 12KB（误差 ~0.81%）
//   Bitmap: 1亿bit ≈ 12MB/天，1年 ≈ 4.3GB
// 取舍: 实时看板用 HLL，日报用 Bitmap，留存分析用 ClickHouse
```

#### 追问: HyperLogLog 原理？

```
原理: 
  - 每个元素 hash 后看二进制末尾连续 0 的个数（前导零计数）
  - 取多个桶（16384 个）的 harmonic mean
  - 12KB 能统计 2^64 个元素

误差来源: hash 冲突 + 小基数时稀疏编码
优化: Redis HLL 自动判断稀疏/密集存储
```


#### 面试话术

```
亿级 UV 统计没有银弹，要看场景选方案。
实时看板用 HyperLogLog，12KB 就能统计 2^64 个用户，
误差 0.81% 业务上可接受。日报需要精确数据用 Bitmap，
1 亿用户每天约 12MB。留存分析用 ClickHouse，
支持多维下钻。三层配合：HLL 看实时趋势，
Bitmap 出日报，ClickHouse 做分析。
```
### Q3: 外卖订单分配的 C10K → C1000K 怎么做？

```
C10K: epoll + 非阻塞 IO + 广播给附近骑手
C1000K: 
  ├── 连接池（非独享连接）
  ├── 网关合并（区域维度合并）
  └── Goroutine 每连接
```

---


#### 面试话术

```
从 C10K 到 C1000K 不是简单的参数调大。
核心变化：从每个连接一个线程/进程，
变成事件驱动（epoll）+ 非阻塞 IO。
C1000K 还需要连接池复用、网关合并减少内部调用、
Goroutine 池控制并发数。真正的瓶颈往往不在网络，
而在业务逻辑的处理速度和内存占用。
```

## 三、百度特色题

### Q4: 搜索引擎的倒排索引怎么构建？

```
构建:
  文档 → 分词 → 去停用词 → (词 → docId + 位置 + 词频)

查询:
  "北京 天气" → 查索引: 北京→[doc1,2,3] 天气→[doc1,4,5]
  → 合并 → doc1 得分最高 → TF-IDF/BM25 排序

压缩:
  ├── Varint 编码
  ├── 差值编码
  └── Roaring Bitmaps
```

#### 倒排索引实现（Go）

```go
package search

import (
    "sort"
    "strings"
    "unicode"
)

// 倒排索引
type InvertedIndex struct {
    // term -> posting list (docID, positions, tf)
    dict map[string]*PostingList
}

type PostingList struct {
    Docs []*Posting
}

type Posting struct {
    DocID     int
    Positions []int  // 词在文档中的位置
    TF        int    // 词频
    TFIDF     float64
}

// Build 构建倒排索引
func (idx *InvertedIndex) Build(docs []*Document) {
    idx.dict = make(map[string]*PostingList)

    for _, doc := range docs {
        // 1. 分词
        tokens := Tokenize(doc.Content)

        // 2. 统计词频 + 位置
        termPositions := make(map[string][]int)
        for pos, token := range tokens {
            token = Stem(strings.ToLower(token)) // 词干提取
            termPositions[token] = append(termPositions[token], pos)
        }

        // 3. 写入倒排索引
        for term, positions := range termPositions {
            if _, ok := idx.dict[term]; !ok {
                idx.dict[term] = &PostingList{}
            }
            posting := &Posting{
                DocID:     doc.ID,
                Positions: positions,
                TF:        len(positions),
            }
            idx.dict[term].Docs = append(idx.dict[term].Docs, posting)
        }
    }
}

// Search 查询（AND 合并）
func (idx *InvertedIndex) Search(query string) []*Posting {
    terms := Tokenize(query)
    if len(terms) == 0 {
        return nil
    }

    // 取第一个 term 的 posting list
    firstTerm := Stem(strings.ToLower(terms[0]))
    result := idx.dict[firstTerm]
    if result == nil {
        return nil
    }

    // 与其余 term 做 AND 合并（双指针）
    for _, term := range terms[1:] {
        term = Stem(strings.ToLower(term))
        list := idx.dict[term]
        if list == nil {
            return nil // 缺少任意 term 就无结果
        }
        result = intersect(result.Docs, list.Docs)
        if len(result) == 0 {
            return nil
        }
    }

    // 按 TF-IDF 排序
    sort.Slice(result, func(i, j int) bool {
        return result[i].TFIDF > result[j].TFIDF
    })
    return result
}

// 双指针求交集（两个有序 docID list）
func intersect(a, b []*Posting) []*Posting {
    var result []*Posting
    i, j := 0, 0
    for i < len(a) && j < len(b) {
        if a[i].DocID < b[j].DocID {
            i++
        } else if a[i].DocID > b[j].DocID {
            j++
        } else {
            result = append(result, a[i])
            i++
            j++
        }
    }
    return result
}
```


#### 面试话术

```
搜索引擎的核心是倒排索引的构建和查询。
构建时分词 → 去停用词 → 建立 term→doc 映射。
查询时多个 term 的 posting list 做 AND 合并（双指针），
按 TF-IDF/BM25 排序。压缩很重要：差值编码 + Roaring Bitmaps
能把索引体积缩小 5-10 倍。
```
### Q5: 搜索提示 (Auto Suggest) 怎么实现？

```
Trie 树: 前缀节点 → DFS 找 Top K 热度词

优化:
  ├── 双数组 Trie (Double Array Trie): MB→KB
  ├── Redis 缓存 Top 10 前缀
  └── 离线重建 + 实时增量
```


#### 面试话术

```
搜索提示的核心是 Trie 树前缀匹配。
原始 Trie 树内存太大，所以用 Double Array Trie
把内存从 MB 级降到 KB 级。实时性方面：
热前缀缓存到 Redis（Top 10），离线每 10 分钟重建一次索引，
保证用户输入每个字符都能在 50ms 内返回提示。
```
### Q6: Embedding 向量召回怎么做？

```
离线训练: Item2Vec → 128 维向量
在线召回: 
  ├── 用户 Embedding = 历史行为平均
  ├── Faiss / HNSW 向量检索
  └── 返回 Top 100 Item
```

---


#### 面试话术

```
向量召回是搜索推荐的进阶方案。
离线用 Item2Vec 或 Bert 训练 128 维向量，
在线用 Faiss HNSW 做 ANN 检索（召回 Top 100）。
HNSW 的核心是多层跳表结构，查询复杂度 O(log n)，
百万级向量检索只需几毫秒。
```

## 四、滴滴特色题

### Q7: 打车订单匹配怎么设计？

```
GeoHash 查附近 → 构建二分图(订单×司机)
  → 边权重(距离+等待+评分) → KM 算法最大权匹配
```


#### 面试话术

```
订单匹配的本质是二分图最大权匹配。
GeoHash 粗筛附近司机减少候选集，
再构建订单×司机的二分图，边权重综合考虑距离、等待时间、司机评分，
用 KM 算法（匈牙利算法）求最优匹配。
滴滴高峰期还要加入方向匹配（顺路程度），
避免司机接了单却要掉头跑远路。
```
### Q8: ETA 预测特征工程？

```
特征: 距离 + 时段 + 天气 + 路况 + 司机习惯
模型: GBDT(XGBoost) / LSTM(时序)
```

---


#### 面试话术

```
ETA 预测是典型的回归问题。
特征工程比模型更重要：距离、时段、天气、路况、司机历史速度。
模型选 GBDT（XGBoost/LightGBM）效果好、可解释性强。
时序数据多的话可以用 LSTM 捕捉周期性规律。
实际落地时注意特征实时性，路况数据最好 1 分钟内更新。
```

## 五、通用算法题补充（百度/美团高频）

### Q9: 最长公共子串/最长公共子序列（百度高频）

```go
// 最长公共子串（连续）
func LongestCommonSubstring(a, b string) string {
    m, n := len(a), len(b)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }
    maxLen, endIdx := 0, 0
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if a[i-1] == b[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
                if dp[i][j] > maxLen {
                    maxLen = dp[i][j]
                    endIdx = i
                }
            }
        }
    }
    return a[endIdx-maxLen : endIdx]
}

// 最长公共子序列（可不连续）
func LongestCommonSubsequence(a, b string) int {
    m, n := len(a), len(b)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if a[i-1] == b[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}
```


#### 面试话术

```
最长公共子串和子序列的区别是连续 vs 不连续。
子串用 dp[i][j] = dp[i-1][j-1] + 1，记录最大值。
子序列用 dp[i][j] = max(dp[i-1][j], dp[i][j-1])，
相等时 +1。两个都是 O(mn) 时间和空间，
优化空间可以只用两行 dp。
```
### Q10: Top K 高频元素（小顶堆）

```go
func TopKFrequent(nums []int, k int) []int {
    freq := make(map[int]int)
    for _, n := range nums {
        freq[n]++
    }

    h := &MinHeap{}
    heap.Init(h)
    for num, count := range freq {
        heap.Push(h, [2]int{num, count})
        if h.Len() > k {
            heap.Pop(h)
        }
    }

    result := make([]int, k)
    for i := k - 1; i >= 0; i-- {
        result[i] = heap.Pop(h).([2]int)[0]
    }
    return result
}

type MinHeap [][2]int

func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i][1] < h[j][1] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)        { *h = append(*h, x.([2]int)) }
func (h *MinHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}
```

---


#### 面试话术

```
Top K 高频元素的标准解法是小顶堆。
先统计频率，再维护大小为 K 的小顶堆，
堆顶是最小频率的元素，每个新元素比堆顶大就替换。
时间复杂度 O(n log k)，空间 O(n)。
如果数据量极大（内存放不下），可以用分治 + 多路归并。
```

## 六、面试故事框架（STAR 格式）

### 美团故事

```
S（情境）: 外卖配送系统，高峰期 100 万订单/天，需要实时分配骑手
T（任务）: 订单分配到骑手延迟 < 1s，配送效率最大化

A（行动）:
  ├── GeoHash 6 位编码（~1km 精度）索引骑手位置
  ├── Redis GEO 缓存在线骑手位置（每 3s 更新一次）
  ├── 订单分配: GeoHash 查附近骑手 → 距离排序 → 最优匹配
  └── HyperLogLog 实时看板（日活、订单量、配送时长）
  
R（结果）:
  ├── 订单分配延迟从 2s 降到 200ms
  ├── 骑手空驶率降 15%
  └── 实时看板 3s 刷新，支撑运营决策
```

### 百度故事

```
S（情境）: 搜索提示服务，用户输入前缀实时补全，QPS 10 万+
T（任务）: 响应 < 50ms，内存可控

A（行动）:
  ├── 离线构建 Double Array Trie（内存从 1GB 降到 200MB）
  ├── Redis 缓存 Top 10 前缀（冷热分离，热前缀内存命中）
  ├── 增量更新: 实时搜索日志 → 离线重建索引（每 10 分钟一次）
  └── Embedding 向量召回: Item2Vec → Faiss HNSW 索引（Top 100）

R（结果）:
  ├── P50 延迟 8ms，P99 延迟 30ms
  ├── 内存从 1GB 降到 200MB
  └── 提示准确率提升 12%
```

### 滴滴故事

```
S（情境）: 早高峰打车难，乘客等待时间长，司机空驶率高
T（任务）: 供需匹配效率最大化，等待时间最小化

A（行动）:
  ├── GeoHash 粗筛附近司机（5km 范围）
  ├── 构建二分图（订单 × 司机），边权重 = f(距离, 评分, 方向)
  ├── KM 算法求最大权匹配（Hungarian Algorithm）
  └── ETA 预测: GBDT 模型，特征 = 距离 + 时段 + 天气 + 路况

R（结果）: 匹配成功率从 75% 提升到 85%，平均等待时间降 20%
```

| 类型 | 链接 |
|------|------|
| 通用-算法 | `02-算法专项/01-必刷题清单.md` |
| 通用-网络 | `05-通用考点/03-计算机网络.md` |
| 通用-MySQL | `05-通用考点/05-MySQL数据库.md` |
| 通用-Redis | `05-通用考点/06-Redis缓存.md` |
