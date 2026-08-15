# 基于 Playwright 的通用 AI 爬虫 Crawl4AI：实现细节与技术原理分析

## 1. 调研对象

- 编号：18
- 原文名称：基於 playwright 的萬用AI爬蟲 Crawl4AI
- 中文索引名称：基于 Playwright 的通用 AI 爬虫 Crawl4AI
- 作者：Bowen Chiu
- 发布时间：2024-10-13
- 地址：https://medium.com/@bohachu/%E5%9F%BA%E6%96%BC-playwright-%E7%9A%84%E8%90%AC%E7%94%A8ai%E7%88%AC%E8%9F%B2-crawl4ai-fcf7a2c77b4e

本文分析目标不是复刻一段 Crawl4AI 示例，而是判断文章里的异步抓取、JavaScript 交互、代理和 CSS Schema 抽取能力如何进入“1000+ 技术博客全量历史抓取 + Markdown 知识库 + 增量同步 + Web 管理”的长期生产架构。

> 原文发表于 2024 年，示例使用早期 Crawl4AI API。实现思想仍然有价值，但生产落地必须以当前 Crawl4AI Runtime 为准，并通过版本化 Adapter 和 smoke test 隔离 API 漂移。

---

## 2. 原文核心能力

文章主要展示四类能力：

1. `AsyncWebCrawler` 异步抓取，通过事件循环提升 I/O 并发效率；
2. 页面加载后执行 JavaScript，例如点击 `Load More`，再结合 CSS Selector 获取动态内容；
3. 使用代理访问页面；
4. 使用 `JsonCssExtractionStrategy` 以声明式 CSS Schema 提取结构化 JSON。

文章实际体现了 Crawl4AI 与 Playwright 的分层差异：Playwright 更接近浏览器自动化原语，Crawl4AI 则在其上提供抓取配置、异步批处理、内容清洗、Markdown、结构化抽取、缓存、代理和页面交互等面向数据采集的抽象。

对知识库平台而言，最重要的结论是：**Playwright/Crawl4AI 都只是 Fetch/Extraction 执行层，而不是知识库事实层。** URL Coverage、任务状态、Snapshot、Document Version、增量游标、质量证据和审计必须由平台自身保存。

---

## 3. 技术原理

### 3.1 浏览器抓取是多阶段状态机

一次动态页面抓取可以抽象为：

```text
Scheduler / Rate Limit / Retry
        ↓
Browser Context / Page
        ↓
Navigation
        ↓
JS Interaction / Wait / Scroll / Click
        ↓
Rendered DOM
        ↓
Content Selection / Cleaning
        ↓
Structured Extraction / Markdown
        ↓
Crawl Result
```

Playwright主要负责 Browser Context、Page、Navigation、DOM 和 JavaScript 操作；Crawl4AI把常见抓取行为进一步封装成统一配置和统一结果。

对 1000 个站点，真正有价值的不是“少写几行 Playwright”，而是让站点差异从 Python 分支代码转成可版本化配置，从而让 Browser Worker 的执行模型统一、可审计、可回滚。

### 3.2 异步并发不等于无限并发

`AsyncWebCrawler` 的性能收益来自 I/O 等待时释放事件循环，但吞吐仍受以下资源约束：

```text
CPU
Memory
Browser Context / Page 数量
目标站点 QPS / Retry-After
网络带宽
页面脚本执行时间
```

当前 Crawl4AI 提供 `arun_many()` 和 Dispatcher。适合的生产模型是两层调度：

```text
平台全局 Scheduler
  durable task / lease / retry
  global/source/domain/route token bucket
        ↓
Browser Worker
  Crawl4AI MemoryAdaptiveDispatcher 或 SemaphoreDispatcher
  RateLimiter
        ↓
AsyncWebCrawler.arun_many(stream=True)
```

Crawl4AI Dispatcher 只负责单 Worker 的并发和资源保护，不能替代平台 durable scheduler。Worker 崩溃恢复、跨机器公平性、任务幂等和 checkpoint 必须仍由 PostgreSQL/任务系统管理。

### 3.3 动态页面交互应建模为可重放 Recipe

原文用 JavaScript 点击 `Load More`。当前 Crawl4AI 的交互能力已经扩展到：

- `js_code_before_wait`：在等待条件前执行动作；
- `wait_for`：等待 CSS/JavaScript 条件；
- `delay_before_return_html`：有限稳定等待；
- `js_code`：页面加载完成后执行脚本；
- `session_id`：短期复用页面状态；
- `js_only=True`：复用 session，仅执行后续 JS；
- `virtual_scroll_config`：虚拟滚动；
- iframe / Shadow DOM 处理能力。

执行过程可以抽象为：

```text
navigate
 -> pre-interaction
 -> wait condition
 -> bounded delay
 -> click/scroll/js
 -> DOM transform
 -> capture rendered DOM
 -> extract
```

生产中不能把这些动作散落为：

```python
if source == "site_a": ...
elif source == "site_b": ...
```

而应形成不可变 `interaction_recipe_release`，例如：

```yaml
version: 3
navigation:
  wait_until: domcontentloaded
  timeout_ms: 30000
steps:
  - action: wait_css
    selector: article
  - action: click_until_exhausted
    selector: button.load-more
    max_steps: 20
    stop_when: item_count_unchanged
  - action: virtual_scroll
    container: '#feed'
    max_steps: 30
capture:
  selector: main
budgets:
  max_wall_clock_ms: 45000
  max_dom_bytes: 8000000
```

Recipe 必须版本化、可 diff、可灰度、可回滚，并绑定 Runtime Artifact Release。

### 3.4 CSS Schema 是确定性的站点 Adapter

`JsonCssExtractionStrategy` 的核心不是 AI，而是声明式 DOM 映射：

```text
base selector
 -> repeated element
 -> field selector
 -> text / attribute / nested / list
 -> JSON object
```

它很适合：

1. Archive/Category/Author/Blog Index 上提取文章 URL、标题、日期；
2. 文章页提取 title、author、published_at、updated_at、canonical；
3. 重复结构化列表、表格、卡片。

确定性 Schema 比 LLM 提取更快、更便宜、更可复现，因此在百万级页面上应该优先使用。LLM 可以辅助生成 Schema、解释低置信度页面或离线分析结构漂移，但不应作为默认正文抽取器。

Schema 同样必须 Release 化：

```text
structured_schema_release
  id
  source_id/template_key
  schema_type
  schema_payload
  required_fields
  minimum_item_count
  fixture_refs
  validation_policy
  runtime_artifact_release_id
```

原因是 selector 漂移常表现为“HTTP 200，但结果为 0”，比显式 500 更难发现。系统需要记录匹配行数、必填字段缺失率、字段长度分布和历史基线偏差。

### 3.5 CSS Selector 只能是 Extraction Hint

站点 selector 可能因为改版失效，同站点历史文章也可能跨多个模板。只保存 selector 提取结果会使历史数据不可重放。

更可靠的链路是：

```text
Snapshot / Rendered DOM
 -> Site Selector Candidate
 -> JSON-LD Candidate
 -> Trafilatura Candidate
 -> Readability Candidate
 -> Crawl4AI Candidate
 -> Quality Comparison
 -> Canonical IR
```

因此 CSS/XPath 是候选抽取器，不是最终事实模型。

### 3.6 Markdown 是 Projection，不是真相源

当前 Crawl4AI 的 Markdown 输出通常可区分 `raw_markdown` 和经过内容过滤后的 `fit_markdown`。

知识库应该保存：

```text
Network Snapshot / Rendered DOM
 -> Canonical IR
 -> deterministic Markdown Projection
```

尤其是 BM25/LLM 等 query-dependent Content Filter 生成的 `fit_markdown`，其结果依赖查询或过滤策略，不是稳定文档表示，只能用于某次检索/LLM 上下文优化，不能覆盖 canonical Markdown。

### 3.7 API 演进说明 Runtime Release 必不可少

原文示例使用：

```python
bypass_cache=True
```

当前 Crawl4AI 已使用 `CacheMode`：

```python
CrawlerRunConfig(cache_mode=CacheMode.BYPASS)
```

代理配置也更适合通过 `BrowserConfig.proxy_config` / `ProxyConfig` 表达，而非依赖旧版参数。

这说明生产系统不能只记录“使用 Crawl4AI”，必须锁定：

```text
crawl4ai_version
playwright_version
browser_revision
runtime_image_digest
interaction_recipe_release_id
structured_schema_release_id
```

升级 Runtime 时先运行 fixture/contract/smoke tests，再发布新 Release。

### 3.8 Cache、Session、Proxy 都是执行态，不是业务真相

Crawl4AI local cache、Browser session 和代理池都可以提高执行效率，但不能成为增量同步或任务恢复的唯一状态。

业务事实应存 PostgreSQL/Object Storage：

```text
ETag
Last-Modified
provider cursor
last_fetch_at
last_changed_at
body_hash
snapshot_id
interaction evidence
```

Proxy 应作为 Fetch Route 参数管理并审计，例如：

```text
DIRECT
REGIONAL_PROXY
APPROVED_PROXY_POOL
```

它不能用于绕过登录、付费墙、验证码、WAF 或明确访问控制。

---

## 4. 对现有博客知识库方案的优化

### 4.1 新增 Interaction Recipe Release

现有方案已有 `browser_recipe` 概念，但应升级为明确的一等 Release：

```text
interaction_recipe_release
  id
  source_id/template_key
  version
  navigation_policy
  wait_policy
  interaction_steps
  session_policy
  virtual_scroll_policy
  capture_policy
  expected_dom_signals
  step_budget
  wall_clock_budget
  runtime_artifact_release_id
  created_at
```

### 4.2 新增 Structured Extraction Schema Release

将 CSS/XPath/Regex Schema 从普通配置升级为可测试、可发布、可回滚的 Release，并允许 Discovery Surface 与正文元数据共用。

### 4.3 保存 Browser Interaction Evidence

每次 Browser Fetch 除普通 HTTP 指标外，记录：

```text
interaction_recipe_release_id
structured_schema_release_id
steps_attempted
steps_succeeded
wait_time_ms
js_time_ms
click_count
scroll_count
session_reused
matched_item_count
rendered_dom_hash
browser_seconds
browser_memory_peak
outcome_code
```

这样 Web 管理台才能解释：

- 为什么某站 Browser 成本突然增加；
- 为什么某 Archive 从每天发现几十条变成 0；
- 是源站真的没更新，还是 selector/interaction 漂移。

### 4.4 动态发现面与正文抓取分离

如果某站只有 Archive 的 `Load More` 必须 Browser 才能枚举历史 URL，Browser 只用于该 Discovery Surface：

```text
Browser Archive / Load More
 -> extract article URLs
 -> normalize/deduplicate
 -> ordinary HTTP article fetch
 -> Browser fallback only when body quality fails
```

这样不会因为一个动态列表页，就把全站百万次正文抓取都升级成 Browser。

### 4.5 Worker 内批处理保持 per-item durable result

可以用 `arun_many(stream=True)` 提高同一 Browser Worker 吞吐，但必须按 URL 单条持久化：

```text
lease N tasks
 -> arun_many(stream=True)
 -> one result arrives
 -> persist FetchAttempt/Snapshot
 -> ack that task
```

一个 URL 失败不能导致整批有效结果回滚。

---

## 5. 当前 API 形态示例

下面只表达当前架构形态，生产参数必须由 Runtime Artifact Release 锁定并经过 CI 验证：

```python
from crawl4ai import (
    AsyncWebCrawler,
    BrowserConfig,
    CrawlerRunConfig,
    CacheMode,
    JsonCssExtractionStrategy,
)

browser = BrowserConfig(
    headless=True,
    proxy_config=None,
)

schema = {
    "name": "Blog Cards",
    "baseSelector": "article",
    "fields": [
        {"name": "title", "selector": "h2", "type": "text"},
        {
            "name": "url",
            "selector": "a[href]",
            "type": "attribute",
            "attribute": "href",
        },
    ],
}

run = CrawlerRunConfig(
    cache_mode=CacheMode.BYPASS,
    js_code_before_wait=(
        "document.querySelector('button.load-more')?.click()"
    ),
    wait_for="css:article",
    extraction_strategy=JsonCssExtractionStrategy(schema),
)

async with AsyncWebCrawler(config=browser) as crawler:
    result = await crawler.arun(
        "https://example.com/blog",
        config=run,
    )
```

生产系统不应把该逻辑硬编码到业务 API，而是由 `interaction_recipe_release + structured_schema_release` 生成受控 `BrowserConfig/CrawlerRunConfig`。

---

## 6. 风险与治理

### 6.1 Browser 成本

Chromium 的内存、CPU、启动和脚本执行成本远高于 HTTP。Browser 必须独立 Worker Pool、小并发、可回收，并持续计算 `browser_seconds_per_changed_document` 与 `browser_seconds_per_discovered_url`。

### 6.2 Selector/DOM 漂移

每个 Schema 应有 fixture、required fields、minimum item count 和历史 DOM/质量基线。零匹配、缺字段率突升或 DOM signature 大幅变化时触发 drift 告警，而不是静默保存空结果。

### 6.3 无界交互

`click until exhausted`、无限滚动和长期 wait 都可能造成死循环。每个 Recipe 必须定义最大步骤、最大 wall clock、最大 DOM、最大新增 item 数和连续无变化停止条件。

### 6.4 Session 不可承担恢复语义

`session_id` 适合短期页面交互，但 Worker 重启后可能丢失。需要恢复的游标/任务状态必须转换为 Provider cursor / Task checkpoint 保存到数据库。

### 6.5 版本漂移

Crawl4AI 与 Playwright 的 API、浏览器版本和 DOM 行为都会演进。所有 Runtime 升级必须经过固定 Golden URL、Recipe fixture 和 Contract Test。

---

## 7. 最终结论

本篇不改变现有方案的主架构，仍应坚持：Coverage First、HTTP First、Snapshot First、IR First、Durable Task、Source Sync 与 AI 解耦。

本次值得写入最终方案的新增点是：

1. 把动态交互正式建模为 `Interaction Recipe Release`；
2. 把 CSS/XPath/Regex 正式建模为 `Structured Schema Release`；
3. 对 Browser interaction 保存执行证据、成本和漂移指标；
4. Crawl4AI `arun_many()` 仅承担 Worker 内资源调度，不替代全局 durable scheduler；
5. 动态 Discovery Surface 可以 Browser 化，但文章正文仍 HTTP First；
6. Crawl4AI cache、session、proxy 都属于执行态，不属于知识库事实状态；
7. Runtime Artifact Release 必须锁定 Crawl4AI、Playwright、浏览器 revision，并对旧 API/新 API 变化做兼容隔离。

这些改动能把“Crawl4AI 动态抓网页”从单机脚本能力升级为 1000+ 网站长期可运营、可审计、可扩展、可重放和可回滚的平台能力。

## 8. 参考资料

- 原文：https://medium.com/@bohachu/%E5%9F%BA%E6%96%BC-playwright-%E7%9A%84%E8%90%AC%E7%94%A8ai%E7%88%AC%E8%9F%B2-crawl4ai-fcf7a2c77b4e
- Crawl4AI Multi-URL Crawling：https://docs.crawl4ai.com/advanced/multi-url-crawling/
- Crawl4AI Page Interaction：https://docs.crawl4ai.com/core/page-interaction/
- Crawl4AI LLM-Free Extraction：https://docs.crawl4ai.com/extraction/no-llm-strategies/
- Crawl4AI Cache Modes：https://docs.crawl4ai.com/core/cache-modes/
- Crawl4AI Markdown Generation：https://docs.crawl4ai.com/core/markdown-generation/
- Crawl4AI Proxy & Security：https://docs.crawl4ai.com/advanced/proxy-security/
