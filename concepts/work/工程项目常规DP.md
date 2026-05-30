# 工程项目常规 DP

检索、对话机器人、路径规划、序列标注里反复出现的几块：**A***、**KMP**、**Viterbi**、**TF-IDF**、**Trie / 倒排 / 编辑距离**。下面只保留「能照着推」的表示；场景速记：寻路调度用 A*；敏感词与单模式匹配用 KMP（多模式往 AC 自动机扩）；HMM/分词解码用 Viterbi；文档权重用 TF-IDF；FAQ 命中用 Trie+倒排，.tie-break 用编辑距离。

[src: raw/ingested/工程项目常规DP.md]

---

## A* 路径搜索

（open 按 `f = g + h` 最小，`h` 可采纳则最优）

```
OPEN ← 优先队列, 按 f 升序
CLOSED ← ∅
g[start] ← 0; parent[start] ← nil
OPEN.insert(start, f = g[start] + h(start, goal))

while OPEN 非空:
    cur ← OPEN.pop_min()
    if cur == goal: 回溯 parent 得路径; return
    CLOSED.add(cur)
    for nbr in neighbors(cur):
        if nbr in CLOSED: continue
        tentative ← g[cur] + cost(cur, nbr)
        if nbr 未访问 或 tentative < g[nbr]:
            g[nbr] ← tentative
            parent[nbr] ← cur
            OPEN.insert_or_decrease(nbr, f = g[nbr] + h(nbr, goal))
return 无解
```

[src: raw/ingested/工程项目常规DP.md]

---

## KMP 单模式字符串匹配

（主串指针不回退；`π[j]` = `p[0..j]` 最长相等真前后缀长）

```
// 线性构造前缀函数 π，p 下标 0..m-1
π[0] ← 0
k ← 0
for j ← 1 to m-1:
    while k > 0 and p[k] ≠ p[j]:
        k ← π[k-1]          // 回退到次长可行前缀
    if p[k] == p[j]: k ← k+1
    π[j] ← k

// 匹配（可重叠多段：每次 j==m 后 j ← π[m-1]）
i ← 0; j ← 0
while i < len(s):
    if j < m and s[i] == p[j]: i++; j++
    else:
        if j > 0: j ← π[j-1]
        else: i++
    if j == m: 报告起点 i - m; j ← π[m-1]
```

[src: raw/ingested/工程项目常规DP.md]

---

## Viterbi 序列解码

（篱笆图：第 t 层节点只连向第 t+1 层；求最大概率路径，乘积取 log 则变加法）

```
// 状态 s_t ∈ 第 t 层，emit 观测 o_t，转移 P(s_{t+1}|s_t)，发射 P(o_t|s_t)
δ[t][s] ← 到时刻 t、落在状态 s 的路径最大 log 概率
ψ[t][s] ← 同上路径上一时刻状态

for s in 第0层: δ[0][s] ← log P_start(s) + log P(o_0|s)

for t ← 1 to T-1:
    for s in 第t层:
        δ[t][s] ← max_{s'} ( δ[t-1][s'] + log P(s|s') + log P(o_t|s) )
        ψ[t][s] ← argmax_{s'} (...)

s* ← argmax_s δ[T-1][s]
路径 ← 从 s* 沿 ψ 回溯到 t=0
```

[src: raw/ingested/工程项目常规DP.md]

---

## TF-IDF 文本权重计算

（词 t 在文档 d、语料 D；变体多，记一种标准形）

```
tf(t,d)   ← count(t,d) 或 log(1 + count(t,d))
df(t)     ← 含词 t 的文档数
idf(t)    ← log( N / df(t) )   // N = |D|，常 +1 平滑
w(t,d)    ← tf(t,d) * idf(t)
文档向量 d ← 各词 w(t,d)；query 同理，相似度常用 cos(d,q)
```

[src: raw/ingested/工程项目常规DP.md]

---

## Trie 前缀树匹配

（词典驻内存；`children[c]` 为下一节点）

```
Insert(word):
    node ← root
    for c in word:
        if node.children[c] 空: 新建
        node ← node.children[c]
    node.is_end ← true

StartsWithOrMaxMatch(text, i):  // 从 text[i] 起沿 Trie 走到底或走最长前缀
    node ← root; j ← i; last_end ← -1
    while j < len(text) and node.children[text[j]] 存在:
        node ← node.children[text[j]]; j++
        if node.is_end: last_end ← j
    return (last_end)  // 最大匹配：取 last_end 为一词末下标
```

[src: raw/ingested/工程项目常规DP.md]

---

## 倒排索引检索

（term → 文档 id 列表；FAQ 里「命中 term 数」粗排）

```
build:
    for each doc id, 对 doc 分词得 terms:
        for t in terms: inverted[t].add(doc_id)

query(q):
    terms ← tokenize(q)
    score[doc] ← 0
    for t in terms:
        for doc in inverted[t]: score[doc]++
    return top_k by score[doc]   // 可再乘 idf(t) 或接 TF-IDF
```

[src: raw/ingested/工程项目常规DP.md]

---

## Levenshtein 编辑距离

（二次排序、纠错；`cost` 插入/删/换可为 1）

```
// dp[i][j] = s[0..i-1] 与 t[0..j-1] 最小编辑次数
dp[0][j] ← j;  dp[i][0] ← i
for i ← 1 to len(s):
    for j ← 1 to len(t):
        cost ← (s[i-1]==t[j-1]) ? 0 : 1
        dp[i][j] ← min( dp[i-1][j]+1, dp[i][j-1]+1, dp[i-1][j-1]+cost )
return dp[len(s)][len(t)]
```

[src: raw/ingested/工程项目常规DP.md]

---

## 参考链接

[A*](https://www.cnblogs.com/zhoug2020/p/3468167.html) · [A* 通俗](https://blog.csdn.net/weixin_30474613/article/details/97250983) · [Viterbi](https://www.zhihu.com/question/20136144) · [Viterbi 分词](https://www.cnblogs.com/carlber/p/12152177.html) · [TF-IDF](https://www.cnblogs.com/cppb/p/5976266.html) · [TfidfVectorizer](https://zhuanlan.zhihu.com/p/59473719)

[src: raw/ingested/工程项目常规DP.md]

## 工程备忘

大地图 A* 要分层/导航网格；动态障碍用重规划。KMP 多模式 → AC 自动机。搜索线上粗排常见 BM25，TF-IDF 常作基线特征。大 Trie+倒排占内存：几十万词库进程级数 GB 不罕见（视实现）。分词难点：规范、歧义、OOV。

[src: raw/ingested/工程项目常规DP.md]