# 面向 RAG 的电商网站数据抓取：实现细节与技术原理分析

## 1. 调研对象

- 编号：24
- 名称：面向 RAG 的电商网站数据抓取
- 来源标题：eCommerce scraping for RAG
- 原始地址：https://www.reddit.com/r/webscraping/comments/1jcl77v
- 来源类型：Reddit 实践讨论
- 调研登记状态：已调研

该讨论描述了一条典型的“网页抓取直接接 RAG”流水线：先读取 robots.txt 并寻找 Sitemap，找不到时尝试 `/sitemap.xml`、`/sitemap_index.xml`、`/sitemap/sitemap.xml`、`/wp-sitemap.xml`、`/wp-sitemap-posts-post-1.xml` 等常见位置，再失败则从首页跟随同域链接；随后依据 URL 中的 `/product/`、`/faq`、`/blog` 等路径特征进行内容分类并选择不同 Crawl4AI 配置；抓取侧启动 Chromium `AsyncWebCrawler` 并以 asyncio 并发处理 URL；抽取侧调用 LLM；最后设置元数据、生成 embedding 并写入数据库。

帖子给出的一次运行日志中，Fetch 约 39.41 秒、LLM Extract 约 27.95 秒、总耗时约 67.46 秒。这个现象对“1000 个技术博客、全历史文章、长期增量同步”的目标很有代表性：**主要问题不是有没有使用 asyncio，而是 Browser 与 LLM 同时被放进逐 URL 的默认关键路径，高延迟与高成本会随 URL 数量近似线性放大。**

本次调研最值得吸收的设计不是电商字段本身，而是把采集链明确拆成：**URL Discovery → Page Type Classification → Execution Routing → Fetch → Extraction → Quality → Canonical → Projection**，并进一步对 Browser/LLM 等慢路径建立隔离、收益评估与熔断闭环。

---

## 2. 原方案逐步分析

### 2.1 Sitemap 优先是正确方向，但“首页递归”不能证明历史抓全

Sitemap 通常比盲目深爬更接近站点公开维护的 URL 集合，优先读取 robots.txt 中的 Sitemap 声明，再探测常见 Sitemap 地址是合理的低成本路径。

但当 Sitemap 不存在时直接从首页递归，会把“导航可达性”误当成“历史完整性”：

1. 旧文章可能已从首页、分类页、分页页脱链，但仍存在于 CMS API、历史 Sitemap、Archive 或外部归档；
2. JavaScript 列表、Load More、虚拟滚动可能使普通 link-following 永远看不到后续 URL；
3. crawler visited set 只能说明“这次策略没有继续发现链接”，不能说明“历史 URL 已穷尽”；
4. 即使某个第三方归档已经扫描完，也只证明该归档集合已扫描，不等于源站历史完整。

因此知识库平台应把 Discovery 建模为多个独立 Provider：

```text
CMS/API
 -> Sitemap / Sitemap Index
 -> RSS/Atom
 -> Archive / Category / Pagination
 -> Dynamic Listing
 -> Bounded Deep Crawl
 -> Optional Common Crawl / Archive Gap Filling
```

每个 Provider 都输出 `url_observation` 与 `coverage_evidence`，至少保存 cursor/range、发现数、新增数、stop reason、是否 exhausted、Known Gap。Coverage 是业务事实，不能由 Crawl4AI/Scrapy 的内部 frontier 或 cache 代替。

Crawl4AI 的 AsyncUrlSeeder、Sitemap/Common Crawl seeding 可以作为 Discovery Adapter，但它们只是发现能力，平台仍需要自己的 URL Identity、Observation 与 Coverage Evidence。

---

### 2.2 URL 字符串分类不是低级方案，散落在代码里的 if/else 才是问题

原帖根据 `/product/`、`/faq`、`/blog` 等 URL path 选择配置，并考虑是否应该改成 LLM 分类。对于大规模长期采集，正确方向不是“把 if/else 全换成 LLM”，而是把规则升级成**版本化、可解释、可回放的 Page Type Classifier 与 Route Policy**。

页面类型是高频、相对低熵的判断，大量信号都可以低成本获得：

- Discovery Provider Hint：RSS item、CMS post API 本身就是高置信文章信号；
- URL path：`/blog/`、`/posts/`、`/docs/`、`/tag/`、`/author/`、`/category/`；
- MIME/Header：PDF、JSON、图片、附件可直接分流；
- JSON-LD：`Article`、`BlogPosting`、`TechArticle`、`Product`、`FAQPage`；
- OpenGraph/Microdata：`og:type=article` 等；
- DOM：`article/main/time`、H1、正文段落密度、链接密度；
- Source/Profile 中已验证的人工规则。

推荐分类层级：

```text
Provider Hint
 -> URL Rule
 -> MIME / Header
 -> Structured Metadata
 -> Cheap DOM Heuristic
 -> Manual Override
 -> LLM Exception（低比例、可关闭）
```

LLM 更适合 Source Probe 阶段读取少量代表样本，帮助生成候选 URL pattern、selector、repair rule；候选规则必须经过 Golden Corpus、Replay、Canary 后才能进入 ACTIVE Release。

逐 URL 使用 LLM 分类会增加网络 RTT、模型排队、token 成本、输出随机性、Schema 校验和重试；模型升级还会改变历史分类结果。对于 `/tag/`、`/author/` 之类确定性信号，LLM 几乎没有信息增益。

---

## 3. Page Type Classification 应成为持久事实

不要只把 `page_type` 放在 Worker 临时变量里。建议持久化：

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

职责边界：

- `url_identity`：它是谁；
- `url_observation`：哪个 Provider 在什么位置发现它；
- `url_classification`：为什么判断它是什么；
- `route_decision`：当前 Source Release 决定如何执行；
- Crawler matcher：某个运行时如何实现当前 route。

这样可以在 Classifier/Route Policy 升级后对历史 URL 做 Replay，而不是无审计地覆盖旧判断。

---

## 4. Execution Routing：先决定是否值得进入高成本路径

推荐的业务语义路由：

```text
ARTICLE -> HTTP First -> Extraction Portfolio -> Canonical
DOC     -> HTTP First -> Document Extraction -> Canonical
INDEX   -> Discovery Only
TAG     -> Discovery Only
AUTHOR  -> Discovery Only
ASSET   -> Asset Pipeline
API     -> Structured Fetch
UNKNOWN -> Cheap HTTP Probe -> Reclassify / Quarantine
```

Source Profile 可表达为：

```yaml
routing:
  page_types:
    article:
      url_patterns: [/blog/**, /posts/**]
      route: http_article
    doc:
      url_patterns: [/docs/**]
      route: http_doc
    index:
      url_patterns: [/tag/**, /category/**, /author/**]
      route: discovery_only
    asset:
      mime: [application/pdf]
      route: asset_pipeline
  unknown:
    route: cheap_probe
    llm_exception: false
```

Route Policy 是平台业务配置；Crawl4AI 的 `url_matcher`、多 `CrawlerRunConfig` 等只是 Adapter 编译目标，不能成为唯一业务事实。

---

## 5. Crawl4AI 的正确定位：Browser/Fetch Adapter，不是平台状态机

### 5.1 两阶段抓取

对 Index/Tag/Author/Archive 等页面，目标往往只是发现链接，不应该同时生成 Markdown、做正文抽取、跑媒体处理，更不应该逐页调用 LLM。

推荐：

```text
阶段 1：Discovery / Prefetch
  HTML/links/必要 metadata

阶段 2：Selected Fetch
  仅 ARTICLE/DOC/ASSET/API 等合法目标进入对应执行路径
```

如果锁定的 Crawl4AI Release 支持 prefetch、`arun_many`、`url_matcher`，可用于 Adapter 级批处理优化；平台仍保存独立的 Observation、Classification 和 Route Decision。

### 5.2 Async 不等于无限并发

原帖已经使用 asyncio，但单 URL 仍需要约 67 秒，说明主要耗时来自 Browser navigation 与 LLM inference，而不是 Python 调度。

生产平台需要受控并发：

```text
Global Admission
 + per-domain token bucket
 + per-domain semaphore
 + Browser slot semaphore
 + Source budget
 + memory-aware dispatcher
 + queue backpressure
```

HTTP Worker 与 Browser Worker 必须独立分池。一个动态站把 Chromium slot 占满时，不能阻塞其他静态博客的增量同步。

---

## 6. 39 秒 Fetch 应拆成阶段指标，而不是只调 asyncio 参数

Browser/HTTP Trace 至少拆成：

```text
DNS
TCP/TLS
TTFB
HTTP body download
Browser acquire
page.goto
DOM ready
wait_for / readiness
interaction recipe
render serialization
```

对于技术博客正文，默认策略应是：

1. httpx/aiohttp 先取 Raw HTML；
2. 运行 Structured Metadata + Trafilatura/确定性 Extractor；
3. Quality PASS 直接完成；
4. 只有 HTML 是 JS shell、正文缺失，或已验证 Browser 能恢复正文时才升级 Browser；
5. Browser 可按 Source Release 阻止非必要图片、字体、视频和跟踪资源，但必须经 Golden Sample 验证，不能误阻正文依赖的 XHR/fetch/script；
6. HTTP Artifact 与 Browser Render Artifact 分开保存，才能测量 Browser 的真实正文增益。

对 1000+ 技术博客，“HTTP 解决大多数正文，Browser 只解决确有增益的少数页面”比“全部 Chromium 并发”重要得多。

### 6.1 Slow Path Isolation 与 Fallback Gain

原帖最值得进一步补强的地方，是不能只知道 Browser/LLM 很慢，还要持续证明“慢路径是否值得”。建议对每次 HTTP -> Browser、Rule -> LLM Exception 的升级记录收益事实：

```text
fallback_evaluation
- fallback_evaluation_id
- url_id
- source_release_id
- fallback_kind: BROWSER | LLM_EXCEPTION
- trigger_reason
- base_quality_score
- fallback_quality_score
- quality_delta
- latency_ms
- estimated_cost
- recovered_to_pass bool
- created_at
```

核心派生指标：

```text
fallback_recovery_rate
  = fallback 后从非 PASS 恢复为 PASS 的数量 / fallback_attempts

fallback_quality_gain
  = quality(fallback) - quality(base)

cost_per_recovered_document
  = fallback 总成本 / recovered_to_pass 数量
```

慢路径治理原则：

1. Browser 与 LLM Exception 使用独立 Queue、并发配额和 Source Budget；
2. 高成本 fallback 超时或队列积压时，只影响该慢路径，不阻塞 Discovery、HTTP 主链与其他 Source；
3. 连续一段窗口内 Browser fallback 几乎没有正文增益时，打开 source-scoped circuit breaker，新的低质量页面进入 `QUARANTINE/DEFERRED`，而不是继续无限烧 Browser；
4. 熔断不能把低质量 HTTP Candidate 静默升级为 PASS；没有达到 Quality Gate 就不能进入 Canonical；
5. 观察结果可以生成新的 Candidate Route Policy，但不能直接修改 ACTIVE Release，必须经过 Replay/Canary/Approval；
6. 对真正长期依赖 Browser 的站点，Probe/Canary 可以证明其恢复率足够高，再给该 Source 更高 Browser Budget。

这形成一个闭环：**Rule/HTTP First → 有证据才升级 → 量化恢复收益 → 低收益熔断 → 生成候选策略 → Replay/Canary 发布**。

---

## 7. LLM Extraction 的性能问题与正确边界

原帖 LLM Extract 约 28 秒，这不是 asyncio 可以消除的成本。LLM Extraction 需要把 HTML/Markdown 发送给模型；长页面还可能 chunk、多请求、合并、校验与重试。

Canonical 正文主链应优先：

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

LLM 适合：

1. Source Probe：少量样本分析；
2. Rule Generation：生成候选 selector/route/repair rule；
3. Ambiguous Classification：规则确实无法判断的低比例异常；
4. Quarantine Triage：辅助解释失败原因；
5. Projection：摘要、实体、关系、主题等可重建派生结果。

LLM 不应：

- 每篇文章先问一次“这是 blog 还是 product”；
- 每篇文章重写 Canonical Markdown；
- 模型输出替代不可变 Raw Artifact；
- 用模型主观判断 Coverage 是否完成；
- 模型失败时阻塞全站增量同步。

---

## 8. Embedding 必须与抓取事务解耦

原帖最后直接生成 embedding 并写数据库。Demo 可以这样做，但长期知识库应改成：

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

Embedding 是 Document Version 的 Projection，而不是 Fetch 成功条件。这样可以保证：

- embedding 服务故障不阻塞抓取；
- 相同 `semantic_hash` 不重复向量化；
- 更换 chunker/embedding 模型可从 Canonical Version 重建；
- 默认只索引 PASS/WARN 的当前有效版本；
- 删除、迁移、版本切换通过版本与 Projection 状态同步，而不是让 crawler 直接操作向量库。

---

## 9. 面向 1000+ 技术博客的推荐主链

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
      INDEX/TAG/AUTHOR -> Discovery Only
      ASSET -> Asset Pipeline
      API -> Structured Fetch
      UNKNOWN -> Cheap Probe / Quarantine
 -> Immutable HTTP Artifact
 -> Extraction Portfolio
 -> Quality PASS ?
      YES -> Canonical IR
      NO + JS shell/recovery evidence -> Slow-path Browser
 -> Fallback Gain Evaluation
 -> Candidate Agreement
 -> Document Version
 -> Markdown Renderer
 -> Async Projection: BM25 / Vector / Export / Optional Graph
```

关键边界有两个：

1. **Page Type Classification 位于 Browser、正文 Extraction、LLM 和 Embedding 之前**，让高成本能力只作用于合法目标；
2. **Browser/LLM fallback 不是无条件兜底，而是受预算、恢复率、质量增益与熔断治理的 Slow Path**。

---

## 10. 对博客知识库技术方案的具体优化项

本次调研建议保留现有 Coverage / Artifact / Extraction Portfolio / Canonical IR 主体，并补强：

1. `Cheap Routing Before Expensive Execution`；
2. 持久化 `url_classification` 与 `route_decision`；
3. Source Probe 增加 page type、非文章样本和 routing rule 探测；
4. Source Profile 增加 `routing.page_types` 与 UNKNOWN fallback；
5. Normalize/Scope/Dedup 后、Fetch 前执行 Classification + Routing；
6. Index/Tag/Author 默认只用于 Discovery；
7. Crawl4AI `arun_many/url_matcher/prefetch` 只作为 Adapter 编译优化；
8. LLM 默认只用于 Probe、规则生成、低比例异常和 Projection；
9. 幂等键增加 Classifier/Route Policy 版本；
10. Web 管理增加 Routing/Classification、UNKNOWN、Browser avoided、LLM Exception 视图；
11. Metrics 增加 classification、route、browser avoided、LLM exception 与 stage latency；
12. 成本治理把 Cheap Classification/HTTP First 放在 Browser/LLM 前；
13. Release/Golden/Canary 验证页面分类、Route 与 Browser fallback；
14. 新增 **Slow Path Isolation + Fallback Gain**：记录 Browser/LLM fallback 的 recovery rate、quality delta、p95 latency、cost per recovered document，并提供 Source 级配额与 circuit breaker；低收益只能进入 Quarantine/候选策略重训，不能静默降低 Quality Gate。

---

## 11. 需要避免的反模式

### 11.1 每站一套 Python if/else

URL pattern 可以用，但应进入 Profile/Route Policy/Recipe，不能散落在 Worker 代码中。

### 11.2 每个 URL 调一次 LLM 做分类

把确定性问题升级成网络、成本、稳定性和模型版本问题。

### 11.3 所有 URL 默认 Browser

技术博客的大多数正文可由 HTTP 获取；Browser 应是有证据的 fallback。

### 11.4 Sitemap 不存在就无限深爬

必须 bounded deep crawl、checkpoint、max pages、duplicate signature、no-new-url stop，并把未穷尽显式记录为 Known Gap。

### 11.5 抓完立即 embedding，失败就整条任务失败

Projection 必须与 Canonical ingestion 解耦。

### 11.6 只看 `success=True`

Fetch success、Extraction success、Quality PASS、Coverage complete 是不同概念。

### 11.7 Browser fallback 长期无收益仍持续重试

Browser 成功打开页面不等于恢复了正文。必须比较 HTTP 与 Browser Candidate 的质量增益；低恢复率、高延迟、高成本的 fallback 应触发 Source 级熔断与重新 Probe，而不是不断占用 Browser Pool。

---

## 12. 参考资料

- 原始讨论：https://www.reddit.com/r/webscraping/comments/1jcl77v
- Crawl4AI 官方仓库：https://github.com/unclecode/crawl4ai
- Crawl4AI 官方文档：https://docs.crawl4ai.com/

---

## 13. 结论

这篇讨论暴露的问题并不是简单的“Crawl4AI 太慢”，而是**Sitemap/Discovery、页面分类、Browser、LLM、Embedding 被近似串成统一的逐 URL 路径，并且慢路径缺少持续收益证明**。

面向 1000+ 技术博客，应先建立可审计 URL Inventory 与 Coverage Evidence，再以可解释 Page Type Classification + Route Policy 选择最便宜且足够可靠的路径：规则处理大多数分类，HTTP 处理大多数正文，Browser 只处理确有恢复价值的动态页面，LLM 只处理少量异常和离线规则生成，Embedding/Graph 等全部异步投影。Browser/LLM fallback 还必须持续记录 recovery rate、quality gain、latency 与 cost，在低收益时熔断并通过 Candidate Release + Replay/Canary 修正策略。这样才能同时满足全历史、增量同步、可扩展接入、Web 管理、质量、成本与长期可维护性。