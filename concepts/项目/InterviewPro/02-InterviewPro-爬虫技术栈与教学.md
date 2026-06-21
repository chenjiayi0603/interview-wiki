# InterviewPro 爬虫技术栈与教学

> 从零搭建一个生产级面经爬虫 —— 用到的技术、开源库、设计思路详解。
> 配套代码：`/home/tommychen/english-learner/crawler/`

---

## 一、技术总览

```
┌─────────────────────────────────────────────────────┐
│                   爬虫技术栈全景                       │
├───────────┬─────────────────────────────────────────┤
│  语言     │  Python 3.12                            │
│  HTTP     │  requests（连接池 + Session + 重试）      │
│  解析     │  BeautifulSoup4 + re（正则）             │
│  配置     │  PyYAML + 环境变量                       │
│  去重     │  hashlib（SHA256）+ json                 │
│  数据结构 │  dataclasses                            │
│  CLI      │  argparse                               │
│  日志     │  logging（文件 + stdout 双写）             │
│  容器     │  Docker (python:3.12-slim)              │
│  调度     │  cron（Linux 定时任务）                   │
│  后端     │  Go + Gin（提供 JWT + API 接口）          │
│  AI       │  DeepSeek / Qwen（LLM 解析面经出题）      │
└───────────┴─────────────────────────────────────────┘
```

---

## 二、核心依赖（requirements.txt）

```txt
requests>=2.31.0
beautifulsoup4>=4.12.0
pyyaml>=6.0
```

只有 **3 个依赖**，体现了"轻量爬虫"的设计哲学：不做重封装，用 Python 标准库 + 少量第三方包完成全部功能。

---

## 三、Python 核心技术详解

### 3.1 requests — HTTP 请求与反爬对抗

`requests` 是整个爬虫的基石，所有网络交互都通过它完成。

**为什么选 requests 而不是 aiohttp/httpx？**
- 同步阻塞在爬虫场景不是问题（延迟本来就是模拟人类阅读）
- API 简洁直观，调试方便
- Session 机制天然支持连接池和 Cookie 持久化
- 社区成熟，坑少

**核心用法：Session 对象**

```python
# base.py — BaseSpider 初始化
self.session = requests.Session()
self.session.headers.update({
    "User-Agent": "Mozilla/5.0 ...",
    "Accept": "text/html,...",
    "Accept-Language": "zh-CN,zh;q=0.9,...",
})
```

Session 的作用：
1. **连接复用**：默认 keep-alive，减少 TCP 握手开销
2. **Cookie 持久化**：牛客 Spider 通过 `session.cookies.set()` 注入 Cookie，后续请求自动携带
3. **统一请求头**：所有请求共享 UA、Accept 等头

**反爬关键：每次请求换 UA**

```python
# base.py
self.session.headers["User-Agent"] = random.choice(USER_AGENTS)
```

9 种真实 UA（Chrome/Firefox/Safari × Win/Mac/Linux），每个请求随机切换，避免指纹固定。

**掘金 API 的特殊处理**：JuejinSpider 没有复用 Session 的浏览器头，而是用独立 `_api_post()` 方法构造请求头，因为掘金 API 识别的是 Web 客户端指纹（`X-Juejin-Client: web`），不需要完整的浏览器头集合。

### 3.2 BeautifulSoup4 — HTML 解析的瑞士军刀

牛客网的搜索结果页和面经板块都是 HTML 格式，bs4 负责从 HTML 中提取链接和正文。

**搜索页解析**：

```python
# nowcoder.py
soup = BeautifulSoup(html, "html.parser")
for a in soup.find_all("a", href=re.compile(r"/discuss/\d+")):
    discuss_id = re.search(r"/discuss/(\d+)", a.get("href", "")).group(1)
    title = a.get_text(strip=True)
```

- `find_all("a", href=re.compile(...))`：用正则过滤链接，只取面经帖子的 URL
- `get_text(strip=True)`：提取纯净文本，自动去掉空白字符

**`__INITIAL_STATE__` 提取**：

牛客的正文由 JavaScript 异步渲染，不在 HTML 标签里，而是藏在 `<script>` 中的 JSON 里：

```python
# nowcoder.py
_CONTENT_FIELDS = ("content", "richContent", "contentText", "richText")
for field in self._CONTENT_FIELDS:
    for m in re.finditer(r'"%s"\s*:\s*%s' % (field, _JSON_STR), html):
        v = json.loads('"' + m.group(1) + '"')
```

这个技巧的核心：
1. 用正则从原始 HTML 中提取 JSON 字符串值
2. `json.loads` 解码转义字符（`\n`、`\uXXXX` 等）
3. 逐个字段尝试，选最长的一个作为正文

**为什么不在页面加载后直接取 HTML 标签内容？**
因为牛客的 HTML 骨架里没有正文，正文是通过 JS 异步请求后渲染的。直接请求页面只能拿到空的容器标签。`__INITIAL_STATE__` 是服务端渲染时嵌入的初始数据，是获取正文的最可靠方式。

**兜底策略**：如果 `__INITIAL_STATE__` 提取失败（站方改了字段名），退回到 CSS 选择器：

```python
for sel in (".nc-post-content", ".discuss-content", ".post-content", "article"):
    el = soup.select_one(sel)
    if el:
        return el.get_text(separator="\n", strip=True)
```

### 3.3 re（正则表达式）— 文本模式匹配

正则在本爬虫中有三个核心用途：

| 用途 | 正则 | 说明 |
|------|------|------|
| 提取帖子 ID | `r"/discuss/(\d+)"` | 从 URL 中捕获数字 ID |
| JSON 字段提取 | `r'"%s"\s*:\s*%s'` | 匹配 JSON key-value |
| 单词边界检测 | `r"(?<![a-z0-9])"` | 避免 "go" 误中 "django" |

**单词边界技巧**：

```python
# base.py
def _token_in_title(token: str, title_lower: str) -> bool:
    if token.isascii():
        # "go" 匹配 "go后端" 但不匹配 "django"
        return re.search(
            r"(?<![a-z0-9])" + re.escape(token) + r"(?![a-z0-9])",
            title_lower
        ) is not None
    return token in title_lower
```

`(?<![a-z0-9])` 是负向后顾断言，确保 token 前面不是字母或数字。`(?![a-z0-9])` 是负向前瞻断言，确保后面也不是。这样 "go" 可以匹配 "Go后端"（后面是中文，不算词内），但不会匹配 "django"（"go" 前面有 "jan"）。

### 3.4 hashlib — 去重核心

```python
# dedup.py
def content_hash(title: str, content: str) -> str:
    text = (title + content[:500]).encode("utf-8")
    return hashlib.sha256(text).hexdigest()
```

**设计决策**：
- **SHA256** 而不是 MD5：虽然 MD5 更快，但 SHA256 在工程上更"安全"（防止碰撞争议），且去重场景性能差异可忽略
- **只取 content 前 500 字**：正文过长时哈希计算无意义，前 500 字（约 200-300 中文词）已能唯一标识一篇文章
- **title + content[:500]** 拼接：标题+内容开头，既能区分不同文章，又能识别同一篇文章的微小变化

**持久化**：

```python
# state.json
{
  "seen_hashes": [
    "a8f2c1d4...",
    "b3e7f1a2..."
  ]
}
```

- 内存 `set` 做 O(1) 判重
- 磁盘 `state.json` 做持久化，容器重启不丢失
- Docker 部署时通过 `CRAWLER_STATE_DIR` 环境变量挂载到持久卷

### 3.5 dataclasses — 数据建模

```python
@dataclass
class RawArticle:
    source: str              # "nowcoder" | "juejin"
    source_id: str
    title: str
    content: str
    url: str
    author: str = ""
    publish_time: str = ""
    job_keyword: str = ""
    crawled_at: str = field(default_factory=lambda: datetime.now().isoformat())
```

Python 3.7+ 引入的 `dataclasses` 装饰器自动生成 `__init__`、`__repr__`、`__eq__` 等方法，相比普通 class 减少约 60% 的样板代码。

**为什么不用 dict 或 namedtuple？**
- **dict**：类型不明确，IDE 无自动补全，refactor 困难
- **namedtuple**：不可变，爬虫数据需要修改（如 `article.content = detail`）
- **dataclass**：可变 + 类型注解 + 默认值 + `field()` 工厂函数，完美匹配爬虫数据流动场景

### 3.6 argparse — CLI 参数解析

```python
parser = argparse.ArgumentParser(description="InterviewPro 面经爬虫")
parser.add_argument("--job", type=str, help="指定岗位名称")
parser.add_argument("--source", type=str, choices=["juejin", "nowcoder"])
parser.add_argument("--count", type=int, help="每关键词最多抓取篇数")
parser.add_argument("--dry-run", action="store_true", help="干跑模式")
parser.add_argument("--config", type=str, default="config.yaml")
```

支持从 cron 无参调用（全量运行），也支持手动指定参数进行调试。

**cron 模式检测**：

```python
is_cron = not sys.stdin.isatty()  # cron 没有 stdin TTY
if is_cron:
    jitter = random.randint(0, 1800)  # 0-30 分钟随机延迟
    time.sleep(jitter)
```

### 3.7 logging — 生产级日志

```python
logging.basicConfig(
    level=level,
    format="[%(asctime)s] %(levelname)-5s %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    handlers=[
        logging.FileHandler("crawler.log", encoding="utf-8"),
        logging.StreamHandler(sys.stdout),
    ],
)
```

**双写策略**：文件 + stdout，stdout 被 Docker 捕获输出到 `docker logs`，文件持久化到磁盘用于回溯。

**日志级别使用规范**：

| 级别 | 场景 | 示例 |
|------|------|------|
| INFO | 正常流程 | `开始: Go后端开发 @ nowcoder` |
| WARNING | 可恢复异常 | `429限流, 等待 6s 后重试 (2/3)` |
| ERROR | 失败但继续 | `牛客详情获取失败 [...]` |
| EXCEPTION | 未捕获异常 | `main.py` 的 `try/except` 兜底 |

---

## 四、反爬技术深度解析

### 4.1 UA 池设计

```python
USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/125.0.0.0 ...",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ... Safari/605.1.15 ...",
    "Mozilla/5.0 (X11; Linux x86_64) ... Firefox/126.0 ...",
    # ... 共 9 种
]
```

**覆盖维度**：
- 浏览器：Chrome × 4、Firefox × 3、Safari × 2
- 系统：Windows × 2、Mac × 3、Linux × 4

**更新频率**：每 10-15 个请求重建 Session（换一整套指纹，不只是 UA）

### 4.2 延迟分布（高斯尾部分布）

```python
def _human_delay(self):
    low, high = self.delay_range  # 默认 [3, 8]
    roll = random.random()
    if roll < 0.7:        # 70%：正常
        delay = random.uniform(low, high)
    elif roll < 0.9:      # 20%：较长
        delay = random.uniform(high * 1.5, high * 2.5)
    else:                 # 10%：超长
        delay = random.uniform(high * 3, high * 5)
    time.sleep(delay)
```

**为什么不是均匀分布？**
真实的阅读行为是"偶尔停顿"而不是"固定间隔"：
- 读者可能快速扫过几页（70% 正常延迟）
- 也可能停下来仔细读一段（20% 较长延迟）
- 还可能有起身倒水、看手机等打断（10% 超长延迟）

这种分布比均匀延迟更难被反爬系统的行为分析识别。

### 4.3 指数退避重试

```python
max_retries = 3
for attempt in range(max_retries):
    resp = self.session.request(method, url, timeout=15, **kwargs)
    if resp.status_code == 429:
        wait = (2 ** attempt) * random.uniform(3, 8)
        time.sleep(wait)
        continue
    if resp.status_code in (403, 503):
        if "captcha" in resp.text.lower()[:2000]:
            raise AntiCrawlerException("验证码拦截")
        wait = (2 ** attempt) * random.uniform(5, 10)
        time.sleep(wait)
        continue
```

退避公式：`2ⁿ × random(3-8)`，第 1 次等 3-8s，第 2 次等 6-16s，第 3 次等 12-32s。

**验证码检测**：检查响应体中是否包含 "captcha" 或 "verify" 关键词。如果检测到验证码，放弃重试（人机验证无法通过重试绕过），抛出 `AntiCrawlerException` 停止当日对该源的抓取。

### 4.4 Session 轮换

```python
self.request_count += 1
if self.request_count >= random.randint(10, 15):
    self._init_session()  # 重建 Session + 换 UA
```

每 10-15 个请求重建一次 Session，相当于换了一台新浏览器访问，重置了服务端的行为分析计数器。

---

## 五、内容质量工程

### 5.1 两阶段过滤架构

```
第一阶段（闸1）：标题关键词匹配
  ┌──────────────┐
  │  spider.search()  │  →  标题命中技术词 → 保留
  └──────────────┘      →  未命中 → 丢弃（不抓详情）
                                ↓
第二阶段（闸2）：LLM 相关性评分
  ┌────────────────┐
  │  smart-parse    │  →  relevance ≥ 3 → 入库
  │  (Go 后端 LLM)  │  →  relevance < 3 → 丢弃
  └────────────────┘
```

### 5.2 技术词映射表 `TECH_TOKEN_MAP`

```python
TECH_TOKEN_MAP = {
    "c++":    ["c++", "cpp"],
    "golang": ["golang", "go"],
    "前端":   ["前端", "frontend", "vue", "react", "javascript"],
    "后端":   ["后端", "服务端", "backend", "server"],
    "算法":   ["算法", "algorithm", "leetcode"],
    "测试":   ["测试", "测开", "qa"],
    "ai":     ["ai", "llm", "大模型", "机器学习"],
    # ...
}
```

**为什么需要这个映射？**
搜索关键词（如 "C++后端开发"）太具体，直接用标题包含判断会漏掉很多相关但标题不同的帖子（如 "面经｜被问到优化数据库索引，答不上来"）。通过技术词映射，只要标题命中 "c++" 或 "后端" 就算相关。

**为什么不直接用 "面经" 作为通用匹配？**
因为标题里含"面经"的帖子可能是"24届校招面经汇总_已OC"（分享帖，非面经）或"帮选offer｜面经分享"等广告帖。技术词映射能过滤掉这些噪音。

### 5.3 LLM 解析质量评估

smart-parse API 返回的每个题目包含 5 维质量评分：

```python
quality_dimensions = {
    "relevance_to_position": 4.5,    # 岗位相关度
    "clarity": 4.2,                   # 问题清晰度
    "difficulty_appropriate": 3.8,    # 难度合理性
    "completeness": 4.0,              # 完整性
    "answer_quality": 4.1,            # 参考答案质量
}
quality = mean(quality_dimensions)  # 最终质量分 0.0-1.0
```

入库时写 `quality_score` 字段，供出题引擎排序使用。

---

## 六、开源栈一览

| 开源项目 | 用途 | 版本 | 许可证 |
|---------|------|------|--------|
| **Python** | 编程语言 | 3.12 | PSF |
| **requests** | HTTP 客户端 | ≥2.31 | Apache-2.0 |
| **BeautifulSoup4** | HTML/XML 解析 | ≥4.12 | MIT |
| **PyYAML** | YAML 配置解析 | ≥6.0 | MIT |
| **Docker** | 容器化 | latest | Apache-2.0 |
| **cron** | 定时调度 | 系统自带 | BSD |
| **Go** | 后端语言（InterviewPro） | ≥1.22 | BSD |
| **Gin** | Go Web 框架 | v1.9+ | MIT |
| **GORM** | Go ORM | v1.25+ | MIT |
| **DeepSeek** | AI 模型（云端） | - | 商业 |
| **Qwen** | AI 模型（本地） | 4B/7B | Apache-2.0 |

### 依赖为什么这么少？

整个爬虫只有 3 个 pip 依赖，原因：

1. **requests 一站解决所有 HTTP**：Session、Cookie、超时、重试全部内置，不需要额外库
2. **BeautifulSoup4 是 HTML 解析的事实标准**：比 lxml 更 Pythonic，比 html.parser 功能更强
3. **PyYAML 是 Python 社区的标准配置方案**：比 JSON 可读性更好，支持注释
4. **不引入 Scrapy**：Scrapy 太重，对于只有 2 个数据源的场景，自定义框架更灵活
5. **不引入 Redis**：去重用 `state.json` 持久化，队列用内存列表，足够满足单机每日几百篇的量级
6. **不引入代理库**：目前两个数据源都不需要代理（牛客/掘金对国内 IP 无封禁）

---

## 七、设计模式在教学中的价值

| 模式 | 使用场景 | 代码位置 |
|------|---------|---------|
| **工厂方法** | 根据配置创建不同 Spider | `main.py` `create_spider()` |
| **模板方法** | 基类定义请求流程，子类实现搜索/解析 | `base.py` → `juejin.py`/`nowcoder.py` |
| **策略模式** | 不同数据源的不同搜索策略 | `search_via_api` vs `search_via_feed` |
| **外观模式** | `Importer` 封装 Go 后端 5 个 API 的调用细节 | `importer.py` |
| **异常隔离** | `AntiCrawlerException` 阻止该源继续抓取 | `base.py` |

### 7.1 工厂方法

```python
def create_spider(source_name: str, source_config: dict, env: dict):
    if source_name == "juejin":
        return JuejinSpider(source_config, cookie=env.get("JUEJIN_COOKIE", ""))
    elif source_name == "nowcoder":
        return NowcoderSpider(source_config, cookie=env.get("NOWCODER_COOKIE", ""))
    else:
        raise ValueError(f"未知的数据源: {source_name}")
```

新增数据源只需：① 创建 spider 子类 ② 在工厂加一条 elif ③ 在 config.yaml 加配置。完全符合开闭原则。

### 7.2 模板方法

`BaseSpider` 定义了请求流程的骨架：

```python
class BaseSpider(ABC):
    def _get(self, url, **kwargs):  # 统一反爬逻辑
        ...  # UA 切换、重试、Session 轮换

    def _human_delay(self):         # 统一延迟策略
        ...

    @abstractmethod
    def search(self, keyword):       # 子类实现
        ...

    @abstractmethod
    def get_article_detail(self):    # 子类实现
        ...
```

子类只需关注业务逻辑（搜索什么、解析什么），不用关心 HTTP 细节。

---

## 八、Go 后端 API 技术

爬虫本身是 Python，但它调用的后端是 **Go + Gin**。这部分技术栈在爬虫视角：

### 8.1 API 通信协议

```
Python 爬虫 ──HTTP JSON──→ Go 后端（:8080）
  POST /api/v1/admin/login              → JWT
  GET  /api/v1/admin/positions           → 岗位列表
  POST /api/v1/admin/questions/smart-parse  → LLM 出题
  POST /api/v1/admin/questions/batch-insert → 入库
```

### 8.2 认证机制（JWT 双 Token）

```
内网：一次性登录，直接返回 token
  POST /api/v1/admin/login → { "data": { "token": "eyJ..." } }

外网：两步验证（防止暴力破解）
  POST /api/v1/admin/login → { "need_verification": true, "dev_code": "123456" }
  POST /api/v1/admin/verify-code → { "data": { "token": "eyJ..." } }
```

JWT 写在 `Authorization: Bearer <token>` 头中，过期后爬虫会重新认证。

### 8.3 LLM 出题接口（smart-parse）

```
POST /api/v1/admin/questions/smart-parse
  Body: { "content": "面经全文...", "position_id": "...", "source_type": "text" }
  Response: { "questions": [{ "question": "...", "answer": "...", "relevance": 5, ... }] }
```

- 超时 90 秒（LLM 推理较慢）
- Go 后端调用 DeepSeek/Qwen 模型解析面经文本
- 返回结构化题目，包含难度、场景、标签、相关性评分
- 爬虫根据 `relevance ≥ 3` 过滤后批量入库

---

## 九、Docker 与 DevOps

### 9.1 容器化设计

```dockerfile
FROM python:3.12-slim   # 仅 120MB，比 full 版本小 80%
RUN apt-get install cron tzdata  # 只装最小依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt  # 构建时安装
COPY . .
RUN echo "0 3 * * * cd /app && python main.py >> /proc/1/fd/1 2>&1" | crontab -
HEALTHCHECK --interval=60s CMD test -f /data/state.json || exit 0
CMD ["cron", "-f"]
```

**关键设计决策**：
- `python:3.12-slim`：比 alpine 兼容性好（alpine 用 musl libc，有时 pip 安装 C 扩展会失败）
- `pip install --no-cache-dir`：减少镜像层大小
- `cron -f`：前台运行 cron，符合 Docker 的"一个容器一个进程"原则
- `>> /proc/1/fd/1`：cron 任务的标准输出重定向到容器日志
- `HEALTHCHECK`：用 `state.json` 的存在性作为健康指标

### 9.2 容时区设置

```dockerfile
ENV TZ=Asia/Shanghai
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

cron 使用系统时区，确保 "0 3 * * *" 是北京时间凌晨 3 点。

### 9.3 随机 Jitter 避免整点风暴

```python
if is_cron:
    jitter = random.randint(0, 1800)  # 0-30 分钟
    time.sleep(jitter)
```

如果多个爬虫同时刻运行（或与后端批处理任务撞车），整点触发会导致流量尖峰。随机延迟将请求分散到 3:00-3:30 之间。

---

## 十、学习要点总结

### 10.1 从本项目可以学到

1. **如何设计轻量爬虫**：不盲目上 Scrapy/分布式，从具体需求出发选择技术
2. **反爬对抗的工程实践**：UA 池、延迟分布、Session 轮换、指数退避
3. **内容质量控制**：两阶段过滤（关键词 → LLM 评分）保证入库质量
4. **双系统协作架构**：Python 做 I/O 密集型（网络请求）+ Go 做 CPU 密集型（LLM 推理）
5. **Docker 部署的最佳实践**：slim 镜像、cron 前台运行、健康检查、时区设置
6. **Python 标准库的高效运用**：dataclasses、argparse、logging、hashlib、re

### 10.2 扩展思考

| 问题 | 当前方案 | 如果要扩展 |
|------|---------|-----------|
| 数据源超过 10 个？ | 工厂方法 + 子类 | 改为插件架构，每个数据源独立文件 |
| 每日抓取量超过 1 万篇？ | 内存列表 + state.json | 引入 Redis 队列 + Celery 分布式 |
| 需要实时去重？ | SHA256 + 内存 set | 改用 Redis Set 做分布式去重 |
| 需要代理 IP？ | 无（当前无需） | 集成快代理/芝麻代理 API |
| 需要浏览器渲染？ | 无（JSON 和 HTML 够用） | 引入 Playwright/Selenium |
| 需要实时监控？ | 日志 + 健康检查 | 接入 Prometheus + Grafana |

---

> **参考实现**：`/home/tommychen/english-learner/crawler/`（SSH: tommychen@192.168.3.61）
> **Python 版本**：3.12 | **第三方依赖**：requests, beautifulsoup4, pyyaml
> **部署方式**：Docker (python:3.12-slim) + cron