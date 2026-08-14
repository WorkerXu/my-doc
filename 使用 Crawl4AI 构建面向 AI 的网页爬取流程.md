# 使用 Crawl4AI 构建面向 AI 的网页爬取流程：实现细节与技术原理

来源：https://medium.com/towards-data-engineering/ai-ready-web-crawling-using-crawl4ai-c4abc3701257

核对资料：
- https://docs.crawl4ai.com/core/deep-crawling/
- https://docs.crawl4ai.com/core/adaptive-crawling/
- https://docs.crawl4ai.com/core/fit-markdown/
- https://docs.crawl4ai.com/extraction/no-llm-strategies/
- https://docs.crawl4ai.com/advanced/multi-url-crawling/
- https://docs.crawl4ai.com/api/parameters/

## 1. 结论先行

这篇文章真正有价值的不是某一段 Crawl4AI 调用代码，而是把网页知识采集拆成一条可组合流水线：

```text
Seed Discovery
  -> URL Admission / Filter
  -> URL Scoring / Frontier Priority
  -> Fetch / Render
  -> Markdown / Structured Extraction
  -> Quality Gate
  -> Knowledge Storage / RAG
```

对“1000+ 技术博客、历史全量、长期增量、Web 管理”的目标，Crawl4AI 很适合做 **单个 worker 内的抓取与抽取执行内核**，但不应成为平台级调度、任务状态、去重、版本、checkpoint、审计和站点公平性的真相源。

最重要的工程判断有三个：

1. **Best-First 是优先级策略，不是全量完备性策略。** 首次历史全量不能因为分数低、达到 `max_pages` 或 AdaptiveCrawler 认为“信息已经足够”就丢掉合法历史文章。
2. **Adaptive Crawling 适合专题扩展、补漏和增量刷新，不适合定义首次全量完成。** 它的停止依据是 coverage / consistency / saturation 等“信息充分度”，不是“站点所有文章 URL 已枚举完”。
3. **Crawl4AI 的 FilterChain、Scorer、Markdown 和 Extraction Strategy 应被提升为平台可版本化策略。** 规则变化后要能离线 replay 原始快照，而不是重新访问整个互联网。

## 2. 文章中的 Crawl4AI 执行模型

### 2.1 BrowserConfig 与 CrawlerRunConfig

Crawl4AI 将浏览器生命周期与单次抓取行为拆开：

- `BrowserConfig`：浏览器类型、headless、proxy、浏览器级资源与上下文；
- `CrawlerRunConfig`：缓存、等待条件、超时、robots、deep crawl、Markdown generator、extraction strategy、URL matcher 等运行策略；
- `AsyncWebCrawler`：异步执行器；
- `CrawlResult`：最终 URL、状态、HTML、cleaned HTML、Markdown、提取结果、链接和 metadata 等。

这种拆分适合映射到平台中的两个配置层：

```text
FetchProfileRelease
  - http/browser
  - timeout
  - proxy policy
  - user-agent
  - resource blocking
  - robots policy

ExtractionPolicyRelease
  - markdown generator
  - content filter
  - css/xpath/json schema
  - fallback chain
```

生产环境不要直接把文章中的示例参数硬编码到 worker。Crawl4AI API 仍在演进，应该固定镜像版本和依赖 lock；例如当前 v0.9.x 文档中 `wait_until` 用于 `domcontentloaded/networkidle` 导航完成条件，而 `wait_for` 用于 CSS/JS 条件等待。所有配置都应经过 Adapter 翻译，避免升级 Crawl4AI 时影响业务数据模型。

### 2.2 为什么需要浏览器，但不能默认浏览器

文章以浏览器抓取为主，适合动态网页演示，但 1000 个博客的历史回灌不能把 Chromium 当默认传输层。

浏览器请求的成本主要来自：

- Chromium 进程与页面上下文的内存；
- JS 执行、字体、图片、第三方资源；
- 页面 ready 的等待时间；
- 崩溃、僵尸进程、session 泄漏；
- 反爬和指纹策略的复杂度。

因此平台应执行 HTTP-first：

```text
HTTP GET/Conditional GET
  -> 能获得完整正文：直接抽取
  -> 正文为空 / SPA shell / load-more / JS pagination：升级 Browser
```

Browser 升级必须有证据并记录 `browser_reason`，例如 `EMPTY_ARTICLE_BODY`、`JS_PAGINATION`、`DYNAMIC_RENDER_REQUIRED`，这样 Web 端才能统计 Browser fallback rate 并优化成本。

## 3. Seed Discovery：先把“URL 从哪里来”独立出来

文章先从 Sitemap 或主题入口得到 seed，再进入 deep crawl。这个思想对大规模系统非常重要：**发现 URL 与抓正文不是同一个阶段。**

技术博客常见的 Provider：

1. Sitemap / Sitemap Index；
2. RSS / Atom / JSON Feed；
3. WordPress / Ghost / Dev.to 等公开 API；
4. 年/月归档；
5. 分类、标签、作者分页；
6. 普通 HTML 同域链接；
7. JS 动态列表；
8. 外部公共索引补漏。

每个 Provider 应独立保存 capability 与 checkpoint：

```text
exhaustive   理论上可以完整枚举，例如可靠 Sitemap Index
windowed     只提供最近窗口，例如多数 RSS
best_effort  普通链接递归，只能尽力覆盖
```

这解决一个经常被忽略的问题：`frontier == empty` 并不等于“历史全量完成”。如果归档分页因为 selector 漂移根本没发现下一页，队列也会为空，但覆盖其实是不完整的。

## 4. FilterChain：把无效 URL 挡在网络请求之前

文章使用 `DomainFilter`、`URLPatternFilter` 等组合过滤。其本质是 **Admission Control**：越早拒绝无价值 URL，越少浪费网络、Browser、解析和存储资源。

平台化后建议固定如下顺序：

```text
normalize
 -> SSRF / Egress
 -> robots / site policy
 -> domain/subdomain allowlist
 -> path allow/deny
 -> query normalization
 -> file/content-type policy
 -> crawler-trap detection
 -> canonical/redirect identity hints
 -> provider budget/depth
 -> priority scorer
```

### 4.1 URL 规范化

至少处理：

- host 小写、IDN 规范化；
- 去 fragment；
- 去默认端口；
- tracking 参数（utm、fbclid 等）移除；
- query 参数排序；
- 已知 session 参数移除；
- trailing slash 策略；
- percent-encoding 规范化。

规范化规则必须可版本化。URL 唯一键建议为：

```text
(site_id, sha256(normalized_url))
```

Bloom Filter 可以做前置快速判断，但不能作为唯一去重真相，因为 Bloom 的 false positive 可能让一个从未抓过的历史 URL 被错误跳过。最终准确性必须由 PostgreSQL unique constraint 保证。

### 4.2 crawler trap

典型陷阱：

- 日历无限前后翻；
- 搜索页组合参数；
- tag/filter 排列组合；
- `?page=` 无上限；
- session/token 参数；
- 同内容多种排序 URL。

需要记录 trap reason，并按站点可配置最大 query cardinality、最大 page number、路径深度和重复模板比率。

## 5. Best-First：从“深度遍历”升级成 durable frontier 优先级

网站可视为有向图：页面是节点，链接是边。

- BFS：按深度扩散，覆盖稳定，但高价值详情页未必最早出现；
- DFS：快速深入单条分支，容易被深层路径占用预算；
- Best-First：用启发式分数维护优先队列，优先访问更可能产生价值的 URL。

Crawl4AI 当前官方文档支持 `BestFirstCrawlingStrategy`、`FilterChain` 和 URL scorer。文章里的思想可以推广成平台级分数：

```text
priority_score =
    source_priority
  + article_path_score
  + archive_expansion_score
  + recency_score
  + page_type_score
  + optional_semantic_score
  - trap_penalty
  - duplicate_penalty
```

但是必须区分两种模式。

### 5.1 FULL_BACKFILL

目标是“尽可能枚举并抓完合法历史文章”。

- score 只决定先后顺序；
- 低分 URL 不能仅因低分被永久丢弃；
- `max_pages` 只能作为单个 execution slice 的预算，不是站点最终停止条件；
- worker 每完成一批就把新 URL 持久化回 PostgreSQL，后续继续调度；
- 最终完成依赖 Provider checkpoint、frontier 收敛、失败队列处置和覆盖审计。

### 5.2 INCREMENTAL / TOPIC_EXPANSION

目标是有限预算下优先抓最可能变化或最相关的内容。

- 可以使用 score threshold；
- 可以限制 max pages；
- 可以结合语义相关性；
- 可以使用 AdaptiveCrawler 的“信息增益递减”作为停止条件。

这一区分是把文章方案用于生产环境时最关键的修正。

## 6. AdaptiveCrawler 的技术原理与正确边界

当前 Crawl4AI AdaptiveCrawler 通过三层信号判断是否继续：

- Coverage：已采集内容对查询词/主题的覆盖程度；
- Consistency：不同页面所得信息是否形成稳定、一致的覆盖；
- Saturation：继续抓新页面是否仍带来明显增量；
- 综合形成 confidence，并结合 `confidence_threshold`、`min_gain_threshold`、`max_pages` 决定停止。

这本质上是一个“信息搜集充分性”问题，而不是“集合枚举完备性”问题。

因此在博客知识库里推荐三种用途：

1. **专题知识扩展**：围绕某个技术主题，从站内高相关页面开始扩散；
2. **低频站点补漏**：历史全量结束后，以 query/embedding 找可能漏掉的高价值区域；
3. **增量预算优化**：站点很大但近期资源有限时，优先刷新最可能贡献新信息的区域。

不推荐：用 AdaptiveCrawler 达到 0.8/0.9 confidence 就宣布“本站历史文章全量完成”。

## 7. Markdown 清洗：fit_markdown 不是最终真相，需要质量门禁

文章使用 `DefaultMarkdownGenerator` + `PruningContentFilter` 得到更适合 LLM 的 Markdown。当前官方文档说明，Pruning 主要依据文本密度、链接密度、标签重要性和结构上下文做启发式裁剪；开启 filter 后可以获得 `raw_markdown` 与 `fit_markdown`。

对技术博客，这种方法很有价值，但存在一个特别的风险：技术文章正文里常有短代码行、表格、引用、公式、API 列表，过强的 pruning 可能把“看起来稀疏但很重要”的内容删除。

生产方案不应只保存 `fit_markdown`，而应保存：

```text
raw response/html
rendered dom（如用 Browser）
raw_markdown
fit_markdown
structured metadata candidate
final Article IR
final markdown
```

然后用 Quality Gate 选择最终候选。

### 7.1 Quality Gate

建议至少计算：

- 标题存在且与 `<title>/og:title/h1` 一致；
- 正文字符数和段落数；
- 导航/链接密度；
- 代码块数量与 DOM 中 `pre/code` 的保留率；
- 表格保留率；
- 图片引用是否被规范化；
- 发布时间证据；
- boilerplate 比率；
- 与历史版本重复度；
- Markdown 是否存在明显截断；
- canonical 与 final URL 是否冲突。

低于阈值时不要静默入库，而应触发下一层 extractor 或进入人工 review。

## 8. CSS/JSON Schema 抽取：适合稳定模板，必须版本化

文章使用 `JsonCssExtractionStrategy` 从稳定 DOM 中提取标题、日期、正文等字段。对 1000 站规模，这是非常高性价比的路径：确定性、便宜、容易测试。

建议每站维护 `ExtractionContractRelease`：

```text
site_id
release_id
match_rule
base_selector
field selectors
content selector
exclude selectors
published_at rules
canonical rules
markdown rules
canary sample urls
created_at
```

发布新规则前：

1. 从对象存储抽取 20~100 个历史 snapshot；
2. 对新旧 release 离线 replay；
3. 比较标题、正文长度、代码块、发布时间、hash；
4. Web 端展示 diff；
5. canary 通过后再切换默认 release。

这样 selector 漂移时无需重新访问网站，大幅减少网络和反爬压力。

## 9. LLM Extraction：只做昂贵升级路径

文章也展示了 LLM extraction。它适合：

- DOM 极不稳定；
- 需要语义字段而非纯正文；
- 通用抽取器和站点 Contract 都失败；
- 少量长尾页面需要人工规则之前的临时兜底。

不适合把数百万篇文章默认送入 LLM，因为会带来：

- token 成本；
- 延迟与吞吐下降；
- 模型升级导致结果漂移；
- 非确定性；
- 重放成本；
- 外部数据合规风险。

因此 LLM 输出只能作为 candidate，必须保存：模型、provider、prompt version、schema version、输入 snapshot hash、输出、token、成本、错误。每站设置月度预算和熔断阈值。

## 10. arun_many / Dispatcher：worker 内并发，不是全局调度器

当前 Crawl4AI `arun_many()` 支持多 URL 并发，默认 dispatcher 会结合内存压力控制并发。这非常适合单个 worker 内进行批处理。

但它解决的是“本进程如何更安全地并发请求”，不解决：

- 1000 个站点之间的公平性；
- 单站 token bucket；
- durable retry；
- crash recovery；
- exactly-once-ish 状态迁移；
- 跨 worker lease；
- checkpoint；
- 全局 backpressure。

所以应采用两层调度：

```text
平台 Scheduler / PostgreSQL frontier / Redis Streams
    -> 分配一小批同类任务给 worker
        -> worker 内 arun_many + MemoryAdaptiveDispatcher
            -> 每条结果独立回写 attempt/snapshot
```

批处理只是传输和执行优化，任务状态不能被 batch 隐藏。

## 11. robots、限流与礼貌抓取

文章强调 robots 与访问伦理。当前 Crawl4AI 文档中的 `CrawlerRunConfig` 已有 `check_robots_txt`，但平台仍要实现统一策略，因为第三方 Adapter、普通 HTTP worker、Browser worker 都必须遵守同一套规则。

推荐：

- 每站 token bucket；
- 每 host 最大并发；
- 429/503 读取 `Retry-After`；
- 指数退避 + jitter；
- 连续错误熔断；
- robots policy 记录版本与读取时间；
- Terms / 站点黑名单可在控制面禁用；
- 任何 Adapter 都不能绕过 Egress/SSRF 检查。

`check_robots_txt=True` 可以作为 Crawl4AI Adapter 的一层保护，但不能代替平台 policy。

## 12. 对总体技术方案的具体优化

### 12.1 新增 Crawl Mode

任务明确区分：

```text
FULL_BACKFILL
INCREMENTAL
RECONCILE
TOPIC_EXPANSION
REPROCESS
```

不同模式使用不同停止条件。特别是 FULL_BACKFILL 中，Best-First 只排序不淘汰，AdaptiveCrawler 不作为 completeness 判据。

### 12.2 新增 Provider Coverage Ledger

每个 `discovery_run` 记录：

- provider capability；
- pages/entries scanned；
- discovered URL；
- admitted/rejected URL；
- rejected reason；
- checkpoint before/after；
- pagination continuity；
- expected count（若 Sitemap/API 可提供）；
- frontier newly created；
- unresolved error。

Web 端按站展示覆盖证据，而不是只显示“任务成功”。

### 12.3 新增 Extractor Drift Detection

按站监控：

- extraction success rate；
- body length p50/p95；
- title missing rate；
- code-block retention；
- Browser fallback rate；
- `fit_markdown/raw_markdown` 长度比；
- LLM escalation rate；
- duplicate ratio。

若某指标在 release 后突变，自动暂停该 Contract 的全站发布并触发 Probe。

### 12.4 原始快照与离线 Replay

任何网络抓取成功都先不可变保存 snapshot，再执行抽取。Extractor、Markdown Filter、Metadata Resolver 升级时优先对 snapshot replay；只有 snapshot 太旧或页面变化检测要求时才重新访问站点。

### 12.5 Web 管理端增加“策略试跑”

管理员应能：

1. 输入一个 URL；
2. 选择历史 snapshot 或实时抓取；
3. 选择 Filter/Scorer/Extraction release；
4. 查看 raw HTML、cleaned HTML、raw Markdown、fit Markdown、最终 Markdown；
5. 查看 rejected reason / priority score 解释；
6. 对新旧规则做 diff；
7. canary 后发布。

## 13. 推荐的 Crawl4AI Adapter 边界

Adapter 输入只接收平台已经批准的任务：

```text
Crawl4AITask {
  task_id
  url
  crawl_mode
  browser_profile_release
  run_config_release
  extraction_release
  budget
}
```

Adapter 输出统一转换成平台事件：

```text
FetchResult {
  task_id
  success
  final_url
  status_code
  response_headers
  raw_snapshot_ref
  rendered_dom_ref
  raw_markdown_ref
  fit_markdown_ref
  extracted_content_ref
  discovered_links[]
  engine_metrics
  error_class
}
```

平台业务层不依赖 Crawl4AI 的内部数据库或内存 frontier。未来更换 Playwright、自研 HTTP crawler 或其他引擎，只需实现相同 Adapter 契约。

## 14. 最终判断

这篇文章应吸收到总体方案中的能力包括：

- Seed 与抓取解耦；
- FilterChain 作为抓取前 Admission；
- Best-First/Scorer 作为 frontier 优先级；
- `DefaultMarkdownGenerator` + content filter 作为 Markdown 候选；
- CSS/JSON Schema 作为稳定模板的高性价比抽取器；
- LLM Extraction 作为低质量升级路径；
- AdaptiveCrawler 作为专题/增量的预算优化器；
- `arun_many()`/dispatcher 作为 worker 内并发优化；
- Crawl4AI 作为 Adapter 执行内核而非平台状态中心。

同时必须做两项生产化修正：

1. **首次历史全量的完备性由 Provider coverage + durable frontier + checkpoint + reconcile 定义，而不是 Best-First 的预算或 Adaptive confidence。**
2. **任何 Crawl4AI 内部能力都不能绕过平台统一的去重、SSRF、robots、限流、版本、质量和审计。**

按这个边界使用 Crawl4AI，可以得到它在深爬、浏览器、Markdown、结构化抽取和局部并发上的开发效率，同时避免把 1000+ 站点长期知识采集系统绑定到单个爬虫库的内部状态。