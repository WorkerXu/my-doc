# 面向 RAG 的电商网站数据抓取：实现细节与技术原理分析

## 1. 调研对象

- 编号：24
- 名称：面向 RAG 的电商网站数据抓取
- 来源标题：eCommerce scraping for RAG
- 原始地址：https://www.reddit.com/r/webscraping/comments/1jcl77v
- 来源类型：Reddit 实践讨论
- 调研登记状态：调研中

该讨论描述了一条很典型的“网页抓取直接接 RAG”流水线：先寻找 robots.txt 与 Sitemap，失败后从首页跟随站内链接；再根据 URL 中的 `/product/`、`/faq`、`/blog` 等片段做内容类型分类；随后启动 Crawl4AI `AsyncWebCrawler`，用 Chromium 并发抓取；抽取阶段调用 LLM；最后设置元数据、生成 embedding 并写入数据库。

帖子中的一次日志显示，某页面 Fetch 约 39 秒、LLM Extract 约 28 秒、总耗时约 67 秒。这个现象对“1000 个技术博客、全历史文章、长期增量同步”的场景非常有代表性：**真正的瓶颈不是 Python 是否使用 asyncio，而是把 Browser 和 LLM 都放到了每个 URL 的默认关键路径。**

因此，本次调研的核心价值不是把电商抓取逻辑原样迁入博客系统，而是提炼出一层此前方案中需要进一步显式化的能力：**Page Type Classification + Execution Routing（页面类型分类与执行路由）**。它决定一个 URL 应该进入 Discovery、HTTP Fetch、Browser Fetch、Article Extraction、Asset Pipeline 还是直接过滤，并且应优先由低成本、可解释、可版本化的规则完成，而不是逐 URL 调用 LLM。

---

## 2. 原方案逐步分析

## 2.1 robots.txt / Sitemap 优先是正确方向，但“找不到 Sitemap 就从首页递归”不等于历史抓全

帖子首先读取 robots.txt 中的 Sitemap 声明，并尝试 `/sitemap.xml`、`/sitemap_index.xml`、`/wp-sitemap.xml` 等常见地址。这一策略是合理的，因为 Sitemap 通常比盲目页面遍历更接近站点维护方公开的 URL 集合，成本也更低。

但其 fallback 是“从首页开始跟随同域链接”，这里有两个根本问题：

1. **导航可达性不等于历史完整性**：旧文章可能已经从首页、分类页和分页中脱链，但仍存在于 Sitemap、CMS API、归档页或 Common Crawl；
2. **crawler visited set 不等于 Coverage Evidence**：爬虫跑完只能证明“当前策略没有新链接”，不能证明“历史文章集合已经穷尽”。

对知识库平台，应把 Discovery 拆成多个独立 Provider：

```text
CMS/API
 -> Sitemap / Sitemap Index
 -> RSS/Atom
 -> Archive / Category / Pagination
 -> Dynamic Listing
 -> Bounded Deep Crawl
 -> 可选 Common Crawl 补洞
```

每个 Provider 都写入 `url_observation` 和 `coverage_evidence`，保存 cursor、时间范围、发现数、停止原因、是否 exhausted 和 Known Gap。这样历史回填才是可审计的。

Crawl4AI 当前提供 `AsyncUrlSeeder`，可从 Sitemap 与 Common Crawl 做 URL seeding，并支持 pattern、BM25 relevance、并发与缓存。它适合成为某个 Discovery Adapter，但平台仍应自己保存 URL Observation 和 Coverage Evidence，不能把 Seeder 内部缓存当业务真相。

---

## 2.2 URL 字符串分类并不“低级”，它恰恰应该成为第一层廉价路由

帖子用类似以下逻辑选择配置：

```python
if content_type == "product":
    return product_config
elif content_type == "blog":
    return blog_config
```

作者担心这种 if/else 不够智能，并询问是否应该用 LLM 分类。对于大规模采集平台，正确方向不是“把 if/else 换成 LLM”，而是把零散 if/else 升级成**版本化规则路由器**。

原因很简单：页面类型分类是一个高频、低熵任务。绝大多数站点都有明显且稳定的信号：

- URL path：`/blog/`、`/posts/`、`/docs/`、`/tag/`、`/author/`；
- Discovery 上下文：RSS entry 天然更像 Article，Sitemap 可按子 sitemap 命名提供类型 hint；
- MIME：`application/pdf`、`application/json`、图片和附件可直接分流；
- JSON-LD：`Article`、`BlogPosting`、`TechArticle`、`Product`、`FAQPage`；
- OpenGraph：`og:type=article`；
- DOM：`article`、`main`、`time[datetime]`、单 H1、正文段落密度；
- CMS API endpoint 本身已有实体类型。

因此推荐分层：

```text
Provider Hint
 -> URL Rule
 -> MIME / Header
 -> Structured Metadata
 -> Cheap DOM Heuristic
 -> 仍不确定时 LLM Exception
```

LLM 应是最后的低比例异常通道，而不是默认分类器。

### 为什么不应逐 URL 用 LLM 分类

1. 每个 URL 增加网络 RTT 与模型排队时间；
2. 成本与文章量线性增长；
3. 输出存在随机性，需要 schema 校验和重试；
4. 模型升级会改变分类结果，必须额外版本化；
5. 对 `/tag/`、`/author/`、`/blog/foo` 这类确定性问题，LLM 没有信息增益；
6. 分类错误会把整个后续执行策略带偏，例如把 Index 页面误送到正文抽取和 embedding。

LLM 更适合在 Source Probe 阶段读取少量代表样本，帮助生成候选 routing rule，再由 Golden Corpus/Canary 验证后发布。

---

## 3. 建议新增 Page Type Classification 事实层

不要只把 `page_type` 塞进某个 Worker 的临时变量。建议持久化分类事实：

```text
url_classification
- url_classification_id
- url_id
- source_release_id
- page_type:
    ARTICLE | DOC | INDEX | TAG | AUTHOR | HOME |
    ASSET | API | AUTH | ERROR | UNKNOWN
- method:
    PROVIDER_HINT | URL_RULE | MIME | STRUCTURED_METADATA |
    DOM_HEURISTIC | LLM_EXCEPTION | MANUAL
- confidence
- evidence JSONB
- classifier_release
- route_policy_version
- created_at
```

设计原则：

- `url_identity` 负责“它是谁”；
- `url_observation` 负责“谁发现了它”；
- `url_classification` 负责“为什么认为它是什么”；
- `route_policy` 负责“这种类型应该怎么执行”。

这样即使未来规则升级，也可以比较新旧分类器结果，而不是直接覆盖历史判断。

---

## 4. Execution Routing：先决定“值不值得做”，再决定“用什么做”

建议 Source Profile 增加 Routing 配置：

```yaml
routing:
  page_types:
    article:
      url_patterns: [/blog/**, /posts/**]
      fetch: http_first
      extract: article
    docs:
      url_patterns: [/docs/**]
      fetch: http_first
      extract: document
    index:
      url_patterns: [/tag/**, /category/**, /author/**]
      fetch: discovery_only
    asset:
      mime: [application/pdf]
      fetch: asset_pipeline
  unknown:
    fetch: cheap_http_probe
    llm_exception: false
```

路由结果不是 Crawl4AI 配置本身，而是平台语义：

```text
ARTICLE -> HTTP First -> Extraction Portfolio -> Canonical
DOC     -> HTTP First -> Document Extractor -> Canonical
INDEX   -> Discovery only，不进入正文知识库
ASSET   -> PDF/Attachment Pipeline
API     -> Structured Fetch
UNKNOWN -> Cheap HTTP Probe -> 再分类 / QUARANTINE
```

这能解决帖子里“不同 URL 要选择不同 config”的问题，同时避免把框架 API 写进业务模型。

---

## 5. Crawl4AI 的正确使用方式：Adapter 级优化，而不是平台状态机

## 5.1 `arun_many` + `url_matcher` 可以承接执行路由

Crawl4AI 官方发布说明已经支持给 `arun_many()` 提供多个 `CrawlerRunConfig`，每个配置使用 `url_matcher` 对不同 URL 应用不同抓取策略。其价值在于把同一批 URL 按页面类型映射到不同运行参数，减少业务代码里的 if/else forest。

平台可以把 `route_policy` 编译成 Crawl4AI Adapter 配置，例如：

```text
ARTICLE_HTTP_FALLBACK -> 普通文章配置
DOC_DYNAMIC           -> 文档站等待/滚动配置
API                    -> 不进入 Browser Adapter
PDF                    -> 不进入 Crawl4AI Browser
INDEX_PREFETCH         -> 只取 HTML + links
```

但要注意：`url_matcher` 是执行层 matcher，不应成为唯一分类事实。真正的 page_type、method、evidence、classifier_release 仍由平台持久化。

## 5.2 Prefetch 非常适合“两阶段抓取”

Crawl4AI 当前版本提供 prefetch 模式，可跳过 Markdown、Extraction 和媒体处理，仅返回 HTML/Links，用于 URL Discovery。官方说明明确将其定位为“两阶段抓取：先发现，再选择性处理”。

这正适合博客平台：

```text
阶段 1：Discovery / Prefetch
  只做 URL、links、必要 metadata

阶段 2：Selected Fetch
  只有 ARTICLE/DOC URL 才进入正文抓取与抽取
```

它能避免在分类页、tag 页、author 页上浪费 Markdown 生成、正文抽取和媒体处理成本。

## 5.3 Async 不等于“无限并发”

帖子已经使用 asyncio，但单 URL 仍然很慢，这是因为：

- Browser navigation 本身可能等待资源和 JS；
- 页面资源包含图片、字体、广告与第三方脚本；
- LLM Extraction 是额外远程调用；
- 并发太高后，瓶颈会转为内存、Chromium tab、FD、网络和目标站限流。

生产平台需要的是**受控并发**：

```text
Global Admission
 + per-domain token bucket
 + per-domain semaphore
 + Browser slot semaphore
 + Source budget
 + memory-aware dispatcher
 + queue backpressure
```

HTTP Worker 与 Browser Worker 必须分池。某个动态站把 Browser 池打满时，不能阻塞其余 900 个静态技术博客的 HTTP 增量同步。

---

## 6. 为什么帖子里的 39 秒 Fetch 需要拆解，而不是只调 asyncio 参数

`[FETCH] ... Time: 39.41s` 表明主要时间可能花在浏览器导航与页面加载，而不是 Python 调度。

应记录阶段级指标：

```text
DNS
TCP/TLS
TTFB
HTTP body download
Browser launch/acquire
page.goto
DOM ready
network idle / wait_for
interaction recipe
render serialization
```

只有知道慢在哪一段，优化才有方向。

对于博客正文，默认策略应是：

1. 先用 httpx/aiohttp 获取原始 HTML；
2. 运行廉价 Structured + Trafilatura；
3. 若质量 PASS，直接结束；
4. 只有 HTML 是 JS shell、正文缺失或 Golden Sample 已证明 Browser 能恢复时，才进入 Browser；
5. Browser 可按 Source Profile 阻止非必要图片、字体、视频等资源，但不能盲目屏蔽会影响正文渲染的数据请求；
6. Browser Artifact 与 HTTP Artifact 分别保存，以便比较 fallback 是否真的带来正文增益。

对 1000 个技术博客，这比“所有页面都 Chromium”重要得多。

---

## 7. LLM Extraction 的性能问题与正确边界

帖子日志中 LLM 调用约占 28 秒，这不是 Crawl4AI 异步机制能消除的开销。`LLMExtractionStrategy` 本质上要把页面内容发给模型，再等待模型完成结构化输出；如果内容过长还可能发生 chunking、多次模型请求和重试。

因此 Canonical 正文主链不应默认使用 LLM Extraction。

### 推荐顺序

```text
Structured Metadata
 -> Deterministic Recipe
 -> Trafilatura Generic
 -> DOM Repair + Generic
 -> Candidate Agreement
 -> Quality Gate
 -> 必要时 Browser
 -> 极少量 LLM Exception / 人工
```

### LLM 的适合位置

1. **Source Probe**：分析 5～20 个代表样本，建议 URL pattern、selector、噪声节点；
2. **Rule Generation**：生成候选 Extraction/Repair Recipe，但必须静态校验和 Replay；
3. **Ambiguous Classification**：只有规则无法区分且业务价值足够高时调用；
4. **Quarantine Triage**：解释失败页面可能是什么类型；
5. **RAG Projection**：摘要、实体关系、主题等派生信息，可异步独立计算。

### LLM 不应做的事情

- 每篇文章先问一次“这是 blog 还是 product”；
- 每篇文章都用模型重写成 Markdown；
- 用模型输出替代不可变 Raw Artifact；
- 用模型主观判断“历史是不是抓全了”；
- 模型失败时阻塞整个增量同步。

---

## 8. Embedding 不应与抓取事务绑死

帖子第 4 步是抓取后直接生成 embedding 并写数据库。对小型 Demo 没问题，但对长期知识库会造成不必要的耦合。

正确关系应该是：

```text
Fetch
 -> Artifact
 -> Extraction
 -> Canonical IR
 -> Quality PASS
 -> Document Version
 -> Transactional Outbox
 -> Markdown / BM25 / Vector / Graph Projection
```

Embedding 是 Document Version 的 Projection，不是抓取成功的必要条件。

这样有几个好处：

- embedding 服务故障不阻塞抓取；
- 相同 `semantic_hash` 不重复向量化；
- 更换 chunker/embedding 模型可以从 Canonical Version 重建；
- 只有 PASS 当前版本进入默认向量索引；
- 删除/版本切换可原子更新索引指针，而不是让 crawler 直接写向量库。

---

## 9. 面向 1000+ 技术博客的推荐主链

综合本次调研，推荐把现有主链细化为：

```text
Authoritative Discovery
 -> URL Observation / Coverage Evidence
 -> Normalize / Scope / Dedup
 -> Page Type Classification
      Provider Hint
      -> URL Rule
      -> MIME
      -> Structured Metadata
      -> DOM Heuristic
      -> LLM Exception (rare)
 -> Execution Route
      ARTICLE/DOC -> HTTP First
      INDEX       -> Discovery Only
      ASSET       -> Asset Pipeline
      API         -> Structured Fetch
      UNKNOWN     -> Cheap Probe / Quarantine
 -> Immutable HTTP Artifact
 -> Extraction Portfolio
 -> Quality PASS ?
      YES -> Canonical IR
      NO + JS shell evidence -> Browser Fallback
 -> Candidate Agreement
 -> Document Version
 -> Markdown Renderer
 -> Async Projection: BM25 / Vector / Export / Optional Graph
```

这里最关键的新边界是：**Page Type Classification 位于 Browser、LLM 和 Extraction 之前。**

它不是为了追求“AI 分类更智能”，而是为了让高成本能力只作用于真正需要的页面。

---

## 10. 对现有博客知识库技术方案的具体优化项

本次调研建议对现有方案做以下增量优化，不推翻已有 Coverage / Artifact / Extraction Portfolio / Canonical IR 设计：

1. 在核心原则中加入 `Cheap Routing Before Expensive Execution`；
2. 增加 `url_classification` 领域模型，保存分类方法、证据、分类器版本和路由策略版本；
3. Source Probe 增加 page type pattern、非文章样本和路由规则探测；
4. Source Profile 增加 `routing.page_types` 和 unknown fallback；
5. 在 URL 规范化之后、Fetch 之前加入 Page Type Classification + Execution Routing；
6. 对 Index/Tag/Author 页面默认只做 Discovery，不进入 Canonical Extraction；
7. Crawl4AI `arun_many/url_matcher/prefetch` 作为 Adapter 编译目标，不进入业务真相；
8. LLM 默认只用于 Probe、规则生成、低比例异常分类和 Projection，不进入逐 URL 主链；
9. 幂等键增加 `CLASSIFY:{url_id}:{classifier_release}:{route_policy_version}`；
10. Web 管理增加 Routing/Classification 视图，可查看 page type 分布、分类证据、Unknown/LLM fallback 和 route 命中；
11. Metrics 增加 classification、route、browser avoided、llm exception 等指标；
12. 成本治理顺序中把“Cheap Classification / Routing”放在 Browser 和 LLM 之前；
13. Phase 2 与验收标准加入页面类型路由、分类漂移和无逐 URL LLM 依赖的验证。

---

## 11. 需要特别避免的反模式

### 11.1 每个网站一套 Python if/else

URL pattern 本身没问题，问题在于规则散落在代码里。应该沉淀为 Profile/Recipe/Route Policy，并版本化发布。

### 11.2 每个 URL 调一次 LLM 做分类

这会把简单规则问题升级成网络、成本、稳定性和版本问题。

### 11.3 所有 URL 默认 Browser

技术博客大部分正文可直接 HTTP 获取。Browser 应是由证据触发的 fallback。

### 11.4 Sitemap 不存在就无限深爬

必须有 bounded deep crawl、checkpoint、max pages、duplicate signature、no-new-url stop，并将未穷尽标为 Known Gap。

### 11.5 抓完立即 embedding，失败就整条任务失败

Projection 必须与 Canonical ingestion 解耦，使用 outbox/queue 异步完成。

### 11.6 只看 `success=True`

Fetch success、Extraction success、Quality PASS、Coverage complete 是四个不同概念。

---

## 12. 参考资料

- 原始讨论：https://www.reddit.com/r/webscraping/comments/1jcl77v
- Crawl4AI 官方仓库：https://github.com/unclecode/crawl4ai
- Crawl4AI `arun_many` / URL matcher 发布说明：https://github.com/unclecode/crawl4ai/blob/main/docs/blog/release-v0.7.3.md
- Crawl4AI AsyncUrlSeeder / CHANGELOG：https://github.com/unclecode/crawl4ai/blob/main/CHANGELOG.md

## 13. 结论

这篇讨论暴露的问题并不是“Crawl4AI 太慢”，而是**执行策略没有在高成本步骤之前完成分层**：Sitemap、页面类型、HTTP/Browser、抽取器、LLM、Embedding 全部被串成了一条近似统一路径。

对 1000+ 技术博客知识库，应该先建立可解释的 URL Inventory 和 Page Type Classification，再由 Route Policy 选择最便宜且足够可靠的执行路径。确定性规则处理大多数 URL，HTTP 处理大多数正文，Browser 处理真正依赖 JS 的少数页面，LLM 处理少量无法规则化的异常和离线配置生成。这样才能同时满足全历史、增量同步、可扩展接入、Web 管理、成本治理和长期可维护性。