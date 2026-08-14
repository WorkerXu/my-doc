# 使用 Crawl4AI 抓取搜狐文章教程：实现细节与技术原理分析

## 1. 调研对象

- 编号：2
- 原文：<https://juejin.cn/post/7528337041614618676>
- Crawl4AI 官方文档：<https://docs.crawl4ai.com/>
- 重点能力：链接发现、CSS Schema 结构化抽取、`arun_many` 批量抓取、Dispatcher 并发控制、图片资产下载、失败回退。

## 2. 对博客知识库方案最有价值的结论

这篇文章的价值不在于“搜狐”本身，而在于给出了一个很清楚的两阶段抓取模型：

```text
列表/入口页
  -> 发现文章 URL
  -> 过滤出真正文章 URL
  -> 批量抓取文章详情页
  -> 按站点 Schema 提取标题/作者/时间/正文/图片
  -> 清洗正文
  -> 下载资产
  -> 保存结构化结果
```

这与面向 1000 个技术博客的生产方案高度吻合，但要把教程级实现升级为平台级实现，需要额外解决：持久化 Frontier、历史覆盖证明、Schema 版本化、动态路由、图片资产去重与安全、增量同步、失败可恢复、漂移检测和 Web 管理。

最值得吸收的三个原则：

1. **Discovery 与 Extraction 分离。** 文章发现可以优先依赖 URL 形态和 Crawl4AI `result.links`，正文提取则使用站点 Schema；两者生命周期和稳定性不同，不能耦合成一套选择器。
2. **稳定 URL 规则优先于列表页 DOM 规则。** 搜狐示例只判断链接是否以 `https://www.sohu.com/a/` 开头，列表页布局变化时仍可能继续发现文章；这比依赖卡片 DOM selector 更抗页面改版。
3. **站点级 Schema 是高性能路径，但必须版本化并可回退。** `JsonCssExtractionStrategy` 对重复模板非常高效且无需 LLM，但 CSS selector 天生与 DOM 模板绑定，因此必须有 Release、Golden Sample、质量门禁和 fallback。

## 3. 文章实现拆解

### 3.1 链接发现：使用 `CrawlResult.links`

示例通过 `AsyncWebCrawler.arun()` 抓取入口页，然后读取：

```python
result.links.get("internal", [])
```

再将内部链接转换为业务 `Link`，并通过 URL 前缀判断文章页：

```python
link.href.startswith("https://www.sohu.com/a/")
```

Crawl4AI 官方文档确认 `CrawlResult.links` 通常区分 `internal` 与 `external`，每项包含 `href`、`text`、`title` 等信息。其技术意义是：浏览器/HTML 解析层负责“把页面上的链接结构化出来”，业务层只负责决定哪些 URL 可以进入 Frontier。

#### 原理

页面发现本质上不是内容提取问题，而是图遍历问题：

```text
当前页面 -> 出边 links[] -> URL normalize -> scope/filter -> Frontier
```

如果把“文章卡片 CSS selector”同时当作 URL 发现条件，一旦列表页换版，历史抓取会直接断流；而 URL path pattern 往往比视觉 DOM 稳定。

#### 对生产系统的要求

URL 规则不能硬编码在 Worker 中，应成为版本化配置：

```text
url_admission_rule_release
- site_id
- include_patterns[]
- exclude_patterns[]
- canonicalization_rules[]
- article_url_classifier
- version
- status
```

搜狐可以配置为：

```text
include: https://www.sohu.com/a/*
```

而 WordPress、Ghost、Substack、Docusaurus 等站点可分别配置 path pattern 或平台 Adapter。

### 3.2 正文提取：`JsonCssExtractionStrategy`

文章定义了一个站点 Schema，核心结构为：

```text
baseSelector: #article-container
fields:
  author          -> .user-info h4        -> text
  title           -> ... h1               -> text
  published_date  -> #news-time            -> text
  content         -> article               -> html
  original_link   -> [data-role=...]       -> text
  imgs            -> .article img          -> list(src, alt)
```

`JsonCssExtractionStrategy` 的核心是将页面 DOM 映射成固定 JSON Schema。官方文档也将它定位为适合重复页面结构的无 LLM 结构化抽取方式。

#### 原理

CSS Schema 相当于一个站点模板的“编译规则”：

```text
Rendered/Cleaned DOM
  + Extraction Schema Release
  -> deterministic JSON candidate
```

优点：

- 快；
- 成本低；
- 输出字段稳定；
- 可测试；
- 可对同一 Snapshot 离线重放。

缺点：

- DOM 改版会导致 selector 静默失效；
- 同站点可能同时存在多个文章模板；
- A/B、移动版、旧版文章、视频页可能结构不同；
- selector 命中不代表语义正确，例如导航区也可能出现 `h1`。

因此生产方案不能把“成功返回 JSON”视为业务成功。

### 3.3 `content` 提取为 HTML，而不是直接 Markdown

示例把正文设置为：

```text
type = html
```

随后执行 `clean_html()`。

这个选择是合理的：如果一开始直接把 Markdown 当成最终事实，后续很难重新修复链接、图片、代码块、表格、脚注和站内相对 URL。更稳妥的模型是：

```text
Snapshot HTML
 -> Extraction Candidate HTML
 -> Canonical Document IR
 -> Markdown Projection
```

Markdown 是发布格式，不应是唯一的中间真相。

### 3.4 多 URL 抓取：`arun_many` + Dispatcher

文章使用：

```python
async for result in await crawler.arun_many(
    urls=urls,
    config=run_config,
    dispatcher=dispatcher,
):
    ...
```

Crawl4AI 官方文档说明 `arun_many()` 支持：

- 多 URL 并发；
- Dispatcher 控制并发和资源；
- streaming 模式按完成顺序返回结果；
- `MemoryAdaptiveDispatcher` 可根据内存压力自适应；
- `SemaphoreDispatcher` 可固定并发并结合 RateLimiter；
- 多个 `CrawlerRunConfig` 可按 `url_matcher` 为不同 URL 路由不同配置，按顺序 first-match-wins。

#### 原理

这是“执行器级并发”，而不是“业务队列可靠性”。

`arun_many` 适合 Worker 已经领取一小批 durable tasks 后做局部批处理：

```text
PostgreSQL Frontier
  -> lease 20~100 个任务
  -> Worker
  -> arun_many(stream=True)
  -> 每个结果独立落 Snapshot/Outcome
  -> ACK 每个 durable task
```

不能把百万 URL 一次性传给 `arun_many()`，也不能让 Crawl4AI 内存中的任务列表成为唯一任务状态。Worker 崩溃时，仍必须靠数据库 lease 恢复。

### 3.5 结果解析与业务字段处理

文章示例从 `result.extracted_content` JSON 数组中读取第一个元素，然后：

```text
content          -> clean_html
imgs             -> Img.new_imgs
published_date   -> parse_datetime
author           -> author，否则 original_link fallback
images            -> download_imgs
```

这里暴露出一个重要设计点：**Extractor 输出只是 Candidate，之后仍需要字段标准化与质量验证。**

例如：

- `published_date` 解析失败不能用抓取时间替代；
- `author` 为空时用 `original_link` 文本兜底并不一定具有作者语义，必须记录 provenance/confidence；
- `content` 非空也可能是登录提示或模板噪声；
- `extracted_content[0]` 假定了单个 article container，若页面意外匹配多个节点，需要显式判定。

### 3.6 图片下载

教程通过 `httpx.AsyncClient` 下载每张图片，并按 hostname + URL path 保存本地文件；遇到 `//cdn...` 这样的 protocol-relative URL 时补充 scheme。

这一实现适合作为小规模示例，但不适合百万级知识库直接照搬。

主要问题：

1. 一个 article 内图片循环是串行的；
2. 本地路径由远端 URL path 派生，存在超长路径、非法字符、query 冲突等问题；
3. 相同图片被多个文档引用时会重复下载；
4. 没有 content hash、MIME、尺寸、状态码、最大文件大小验证；
5. 没有独立 retry/timeout/限流；
6. 没有 SSRF/redirect/IP 复核；
7. 图片失败可能不应该让正文抓取整体失败；
8. 没有保存原始 URL 与最终对象之间的 lineage。

生产方案应该改为独立 Asset Pipeline：

```text
Document IR image reference
  -> asset_task
  -> URL safety/admission
  -> bounded HTTP fetch
  -> MIME/size/hash validation
  -> S3/MinIO content-addressed object
  -> asset record
  -> Markdown renderer rewrite URL
```

推荐对象键：

```text
assets/sha256/ab/cd/<sha256>.<ext>
```

同一 hash 只保存一次，文档通过引用表关联。

## 4. 教程方案的边界与生产风险

### 4.1 只靠入口页 `result.links` 无法证明历史抓全

某个列表页能看到的链接通常只覆盖最新一页，或者依赖分页/无限滚动。对于“1000 个博客全量历史”需求，必须组合：

```text
Sitemap
RSS/Atom/JSON Feed
CMS/API
Archive Pagination
Category/Tag/Author
HTML links
Browser Dynamic Listing
```

并为每个 Provider 保存覆盖终止证据。否则“抓不到更多链接”不等于“历史完整”。

### 4.2 URL Pattern 稳定，但不能成为唯一分类器

`https://www.sohu.com/a/*` 这种规则非常有价值，但可能同时包含视频、图集、专题或异常页面。需要：

```text
URL classifier -> 页面预检 -> content-type/page-type -> article fitness
```

URL Pattern 应进入 Admission/优先级，而不是无条件 Accepted。

### 4.3 单一 CSS Schema 容易静默失效

应给每个站点维护有序 Schema Set：

```text
site_extraction_profile_release
  schema A: current article
  schema B: legacy article
  schema C: alternate template
  generic candidate: Trafilatura/Crawl4AI Markdown
```

每个 Candidate 都计算质量分，并由 Quality Gate 选择 Accepted Candidate。

### 4.4 `result.success` 只是爬虫执行状态

真正的业务成功至少需要：

```text
HTTP/Browser success
AND Snapshot 已持久化
AND Extractor 产生 Candidate
AND title/body 通过 fitness
AND Document IR 成功构造
AND version 写入成功
```

因此必须有 Stage Outcome Contract 和 Finalizer 对账，不能只判断 `result.success`。

### 4.5 作者兜底不能混淆 provenance

教程里的作者 fallback 体现了“工程上需要兜底”，但生产知识库必须知道这个值来自哪里。建议：

```text
metadata_evidence:
  field=author
  value=...
  source_type=CSS_SELECTOR|JSON_LD|OPEN_GRAPH|FEED|API|FALLBACK|MANUAL
  source_artifact_id=...
  extractor_release_id=...
  confidence=...
```

### 4.6 并发必须同时受系统资源与域名礼貌策略约束

Crawl4AI Dispatcher 解决单 Worker 的并发调度，但平台还应增加：

- 全局 Browser slot；
- per-domain concurrency；
- per-domain RPS/token bucket；
- 429/503 动态退避；
- weighted fair scheduling；
- Browser/HTTP/PDF/OCR 分池；
- backpressure。

## 5. 对现有博客知识库技术方案的优化建议

现有方案已经具备 Discovery、Durable Frontier、HTTP-first、Browser Escalation、Canonical IR、Quality、Drift、Asset 字段和 Web Control Plane 等正确基础。结合本篇调研，建议进一步明确以下能力。

### 5.1 新增 `site_extraction_profile_release`

Source Adapter 不应同时承担正文模板细节。新增独立 Release：

```text
site_extraction_profile_release
- id
- site_id
- version
- page_type
- url_matcher
- required_dom_signals[]
- extraction_candidates[]
- field_mapping
- fallback_order
- quality_policy_release_id
- golden_fixture_ids[]
- status
```

意义：同一个站点可以同时存在多种页面模板，并可以独立升级 Schema 而不改 Discovery Adapter。

### 5.2 Discovery Rules 与 Extraction Rules 解耦

新增明确规则：

```text
Discovery URL Rule 负责“这个 URL 是否值得进入 Frontier”
Extraction Profile 负责“这个页面怎么抽”
Quality Policy 负责“抽出来的结果能否接受”
```

这正是搜狐示例中“URL 前缀发现 + CSS Schema 正文”的可推广形式。

### 5.3 增加 URL-specific Crawl4AI Config Routing

利用 Crawl4AI 当前 `CrawlerRunConfig.url_matcher`/多 config first-match-wins 能力，在单 Browser Worker 内按 URL 类型路由：

```text
*.pdf             -> PDF route
*/article/*       -> article profile
*/docs/*          -> docs profile
动态站点           -> JS/wait_for profile
fallback          -> generic profile
```

但路由配置必须来自版本化 Profile，而不是 Worker 硬编码。

### 5.4 增加独立 Asset Pipeline

核心模型：

```text
asset
asset_source
asset_reference
asset_task
```

建议字段：

```text
asset:
  sha256
  mime_type
  size
  width/height
  object_key

asset_source:
  original_url
  final_url
  fetched_at
  response_headers
  snapshot/artifact reference

asset_reference:
  document_version_id
  asset_id
  role=image|attachment|cover
  original_url
  alt
  ordinal
```

图片下载失败属于可单独重试的子任务，正文成功不应被回滚。

### 5.5 增加 Schema Drift 的强信号

除正文长度和 selector 命中率外，增加：

```text
required_field_hit_rate
candidate_count distribution
baseSelector match count
field source distribution
image count distribution
published_at parse success rate
profile fallback rate
```

如果主 Schema 的 fallback rate 突然上升，应自动触发 drift event。

### 5.6 Worker 内批处理使用 Streaming，而不是大列表等待

对于从 Frontier lease 出来的一小批 Browser 任务：

```text
CrawlerRunConfig(stream=True)
+ arun_many(...)
+ bounded dispatcher
```

结果完成一个就立即写 Snapshot/Outcome，降低峰值内存并缩短 lease 占用时间。

## 6. 建议的最终执行链路

```text
Site Probe
  -> Discovery Provider Set
  -> URL candidates
  -> Normalize + URL Admission Rule Release
  -> Durable Frontier
  -> Fetch Router
       -> HTTP
       -> Browser/Crawl4AI
       -> PDF
  -> Immutable Snapshot
  -> Page Type / Extraction Profile Match
  -> Candidate Set
       -> CSS Schema Candidate
       -> Generic Trafilatura Candidate
       -> Crawl4AI Candidate
       -> API/JSON-LD Candidate
  -> Metadata Evidence
  -> Content Fitness + Candidate Ranking
  -> Canonical Document IR
  -> append-only Document Version
  -> Asset Tasks
  -> Markdown Projection
  -> Search/Embedding/AI downstream
```

## 7. 验收测试建议

针对站点级 Schema 至少建立这些 Fixture：

1. 当前文章模板；
2. 老文章模板；
3. 无作者；
4. 无发布时间；
5. 多图片；
6. protocol-relative 图片；
7. 视频/图集误入 URL pattern；
8. 正文很短但合法；
9. selector 命中但内容为登录/WAF；
10. DOM 小改版导致主 Schema 失败、fallback 成功；
11. `baseSelector` 匹配 0 个；
12. `baseSelector` 意外匹配多个。

验收不能只断言代码无异常，而要断言最终 Outcome、Candidate、IR、Metadata Provenance 和 Asset Reference 是否符合预期。

## 8. 结论

这篇教程证明了一条非常适合大规模博客抓取平台的工程路径：**用链接图完成发现，用 URL 规则做低成本分类，用站点 Schema 做确定性高性能抽取，用 `arun_many`/Dispatcher 提高单 Worker 吞吐。**

但要支撑 1000 个站点和长期增量同步，必须把教程中的函数级逻辑上升为平台级的版本化能力：Discovery Rule、Extraction Profile、Quality Policy、Durable Frontier、Immutable Snapshot、Asset Pipeline、Drift Sentinel 和可恢复 Outcome。

因此本次对总方案最重要的新增不是“再加一个搜狐 Adapter”，而是明确 **Discovery Rule 与 Extraction Profile 解耦**，并把 **站点 Schema、多模板 fallback、Crawl4AI URL-specific config routing、图片资产流水线** 变成一等公民。