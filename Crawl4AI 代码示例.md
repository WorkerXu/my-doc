# Crawl4AI 代码示例

## 1. 调研对象

- 编号：30
- 名称：Crawl4AI 代码示例
- 来源标题：Crawl4Ai code example
- 原始地址：https://www.reddit.com/r/DataHoarder/comments/1iknxwj
- 索引地址：https://incorporated-jail-assigned-figure.trycloudflare.com/
- 索引摘要：CNN Sitemap → AsyncWebCrawler → PruningContentFilter → clean Markdown；包含并发、延迟、重试与链接继续入队。

当前环境无法稳定直接读取原 Reddit 正文，因此本文不伪造原帖代码细节。实现分析以索引中已归档的摘要为起点，并用 Crawl4AI 当前官方源码、官方 `arun_many()` 文档、Multi-URL Crawling 文档、URL Seeder 示例、Dispatcher 示例和 Browser Optimization 示例交叉验证其技术模式。

这篇实践最有价值的不是“用 Crawl4AI 抓 CNN”本身，而是它把一个生产采集系统最关键的几件事串在了一条链路里：**先从可枚举入口发现 URL，再进行有界并发抓取，按结果流式处理，做内容裁剪/Markdown 生成，同时把页面新发现的链接重新送回发现队列**。如果把这套做法直接放大到 1000 个站点，仍然会遇到状态丢失、内存膨胀、浏览器成本、重复抓取、无法证明历史覆盖率等问题；但其执行层模式非常适合成为平台 Fetch Worker 的内部实现。

---

## 2. 核心流程还原

可以把该实践抽象为：

```text
Sitemap / Seed URLs
        │
        ▼
Normalize / Filter / Dedupe
        │
        ▼
Bounded URL Frontier
        │
        ▼
AsyncWebCrawler.arun_many(stream=True)
        │
        ├─ Memory/Semaphore Dispatcher
        ├─ RateLimiter / Retry
        └─ Browser Instance Reuse
        │
        ▼
Result-by-result processing
        │
        ├─ raw response / snapshot
        ├─ extraction
        ├─ raw markdown / fit markdown
        ├─ quality check
        └─ discovered links
        │
        ▼
Durable persistence + links re-enter discovery
```

关键点是“**流式、逐 URL 提交、资源复用、有界队列**”，而不是一次性把所有 URL 装进数组后 `gather()` 到结束。

---

## 3. Sitemap 发现：为什么它适合作为历史抓取入口

### 3.1 Sitemap 是高价值 Provider，不是唯一真相

对于新闻、技术博客和文档站，Sitemap 往往比从首页递归爬链接更接近站点维护者提供的文章清单，尤其是 Sitemap Index + 分片 Sitemap 可以覆盖大量历史页面。

但 Sitemap 只能作为一个 Coverage Provider：

- 站点可能只保留近期 URL；
- 某些文章从未进入 Sitemap；
- 删除、迁移、canonical 合并可能造成清单与真实历史不一致；
- `<lastmod>` 可能缺失、粗粒度甚至不可信。

因此生产系统不能把“读完 Sitemap”直接等价为“抓完历史”。正确做法是保存 Provider evidence、cursor、分片状态和 exhaustion reason，再用 Feed、CMS API、Archive、Category/Tag、Common Crawl 或受控 Deep Crawl 做缺口核对。

### 3.2 Sitemap Index 必须分片执行

1000 个站点中，少数大站的 Sitemap 可能包含几十万乃至更多 URL。应把每个 child sitemap 当作独立工作单元：

```text
sitemap index
  ├─ child sitemap A -> cursor/checkpoint A
  ├─ child sitemap B -> cursor/checkpoint B
  └─ child sitemap C -> cursor/checkpoint C
```

每个分片独立：

1. Conditional GET；
2. 解压/解析；
3. URL Observation 按小批次提交；
4. 生成 Fetch Task；
5. 保存分片 hash、ETag、Last-Modified；
6. 提交 durable checkpoint。

这样某个 50 万 URL 的站点不会因为单进程崩溃而从头开始，也不会要求在内存中同时保留全部 URL。

### 3.3 “发现”和“抓取”必须解耦

官方 URL Seeder 示例展示了 `discover -> filter -> crawl` 的自然链路，但平台中不能把 Seeder 返回值当作唯一状态。每发现一个 URL，先落 `url_observation`，再进行 normalization/filter/classification，最终生成持久化 Task。

好处是：

- 过滤规则升级后可以重放 Observation；
- 可以解释一个 URL 为什么没抓；
- Source 暂停后 frontier 不丢；
- Worker 崩溃不影响 Coverage 事实；
- 新抓取器版本可以复用历史 URL 清单。

---

## 4. AsyncWebCrawler 生命周期与浏览器复用

Crawl4AI 官方 Browser Optimization 示例强调一个很重要的工程原则：**不要为每个 URL 创建新的浏览器实例**。浏览器冷启动、Chromium 进程、上下文初始化和页面资源会远比 HTML 解析昂贵。

### 4.1 推荐的 Worker 生命周期

```text
Browser Worker process
  └─ BrowserPool
       ├─ crawler/browser instance 1
       │    ├─ context/session A
       │    └─ context/session B
       └─ crawler/browser instance 2
```

Worker 启动时创建长期存活的 `AsyncWebCrawler`/浏览器实例，批次之间复用；达到以下条件之一再回收：

- 已处理页面数达到阈值；
- RSS/浏览器内存超过阈值；
- 页面崩溃率上升；
- Browser/Runtime Release 发生切换；
- worker drain；
- 超过最大存活时间。

### 4.2 Browser Process 可以复用，Session 不能无条件复用

“复用浏览器”与“共享登录态/Cookie”必须分开。

可以复用：

- Chromium 进程；
- browser binary；
- browser context pool 的基础设施；
- 静态资源缓存（按安全策略）；
- 同一 Source 且明确允许的会话。

不应跨 Source/租户随意复用：

- Cookie；
- LocalStorage；
- 登录态；
- service worker；
- 站点特定 token；
- 代理身份。

因此生产设计应为 `runtime_release + source/profile + proxy/security_scope` 建立池键，避免为了性能引入跨站状态污染。

### 4.3 HTTP First，Browser Last 仍然成立

文章示例选择 `AsyncWebCrawler` 并不意味着 1000 站全部应该用 Browser。对于 Sitemap、Feed、API、静态 HTML，HTTPX/aiohttp 更便宜、更快、更容易做 Conditional GET。Browser 只处理：

- HTML 首屏没有正文，正文由 JS 请求生成；
- Load More / Infinite Scroll；
- 客户端路由；
- 需要执行少量交互才能出现内容；
- HTTP 抽取质量低于阈值。

Crawl4AI 可以同时支持轻量抓取和浏览器抓取，但平台路由层必须先做资源探测和成本判断。

---

## 5. `arun_many()`：批量并发的真正价值是“流式完成”

官方 `arun_many()` 文档支持批量并发，并可配 `stream=True` 以异步生成器形式按完成顺序返回结果。对百万级 URL 来说，**streaming 比单纯并发更重要**。

### 5.1 非流式批处理的问题

如果一次把 20 万 URL 交给一个流程并等待全部结果：

- URL 列表本身占内存；
- 完成结果可能继续堆在内存；
- 最慢的一批拖住整个批次提交；
- 进程崩溃时难以判断哪些结果已真正持久化；
- 下游解析/对象存储来不及消费时，没有自然背压。

### 5.2 流式消费的正确姿势

平台仍应由 Durable Task 负责真相，Worker 只在局部窗口内批量执行：

```text
1. 从 durable task store lease N 个任务
2. 取得 Source/Domain permit
3. 将 N 个 URL 交给 arun_many(stream=True)
4. 每完成一个 result：
   a. 保存 Snapshot
   b. 记录 Fetch Attempt
   c. 成功则生成后续 Extract Task
   d. 失败则分类并决定 retry / terminal
   e. 持久化新发现 links 为 Observation
   f. ACK 当前 URL，而不是等待整个 batch
5. 当前窗口降到 low watermark 后再 lease 新任务
```

这里的 N 是“小窗口”，可以是几十或数百，而不是整个站点 URL 总量。

### 5.3 为什么逐 URL Commit 很重要

这本质上是一个 at-least-once 分布式任务系统。无法要求网络请求、对象存储和数据库更新组成一个跨系统原子事务，因此策略应是：

- URL Task 有稳定幂等键；
- Snapshot 以 hash/attempt identity 去重；
- 每个成功 URL 立即持久化；
- batch 失败不回滚已经成功的 URL；
- 重试重复执行时通过幂等键合并。

这与官方 `arun_many()` 每个 `CrawlResult` 独立成功/失败的模型天然吻合。

---

## 6. Dispatcher、并发和背压：两层限流，而不是只靠 Semaphore

官方 Multi-URL Crawling 文档提供 `MemoryAdaptiveDispatcher`、`SemaphoreDispatcher` 和 `RateLimiter`。生产系统应把它们定位为 **Worker 内部执行层限流器**，不能替代平台调度真相。

### 6.1 第一层：平台级 Permit

平台按层级预算：

```text
Global
 -> route
 -> registrable domain
 -> source
 -> host
```

例如同一公司可能配置多个 Source，但它们最终都访问 `engineering.example.com`，不能每个 Source 各自放满并发而把源站打爆。

平台 permit 决定“这批 URL 是否允许开始”。

### 6.2 第二层：Worker 本地 Adaptive Dispatcher

Worker 再根据自身资源动态收缩并发：

- RSS / memory percent；
- 当前 browser page 数；
- event loop lag；
- p95 fetch latency；
- 429/503 比例；
- 页面崩溃率；
- 对象存储上传 backlog；
- DB commit latency。

例如平台给一个 Browser Worker 最多 20 个 permit，但当前内存升高时 `MemoryAdaptiveDispatcher` 可以只并行 6 个。**本地调低可以，不能绕过平台向上调高。**

### 6.3 RateLimiter 的合理定位

官方示例中的 `base_delay`、`max_delay`、`max_retries` 很适合单 Worker，但平台还要做统一策略：

- 尊重 `Retry-After`；
- 429/503：退避并降低 host concurrency；
- timeout/connection reset：指数退避 + jitter；
- 401/403/404/410：按原因分类，通常不盲重试；
- robots/policy：直接策略终止；
- Browser crash：可切换实例后重试；
- extract quality failure：可触发 Browser fallback，但不能无限循环。

重试预算必须是 Task/Run 级 durable 字段，不能只存在于 Dispatcher 内存里。

---

## 7. PruningContentFilter：适合“检索投影”，不适合当归档真相

索引摘要提到 `PruningContentFilter`。官方示例中它会生成更精简的 `fit_markdown`。对 RAG 很方便，但对“全量历史文章知识库”有一个关键风险：**它是有损过滤**。

### 7.1 为什么不能直接把 fit_markdown 当最终知识库

可能被误删的内容包括：

- 很短但重要的警告；
- API 参数表；
- 代码旁的简短说明；
- 脚注；
- 版权/版本信息；
- 小节标题与上下文；
- 低文本密度但有语义的表格；
- 某些特殊 DOM 结构。

一旦只保存 fit Markdown，未来更换过滤阈值、RAG 策略或模型时无法恢复被删内容，只能重新访问源站；而源站可能已经改变或消失。

### 7.2 推荐三层内容模型

```text
Snapshot / Raw HTML          不可变网络事实
        │
        ▼
Canonical IR                可重放结构事实
        │
        ├─ Canonical Markdown      完整、确定性、默认归档
        └─ Filtered/Fit Markdown   有损、面向检索的 Projection
```

因此：

- `raw_markdown`/清洗 HTML 可做候选；
- `PruningContentFilter` 的输出存为 `FILTERED_MARKDOWN`；
- RAG 可优先索引过滤版；
- 导出知识库默认导出 canonical Markdown；
- Filter Release 升级时从旧 Snapshot/IR 离线重建，不重新抓源站。

这是本次调研对现有方案最重要的一条确认和强化。

---

## 8. 链接继续入队：必须回到 Discovery Plane，而不是递归抓取

实践脚本里“抓完页面后把新链接继续入队”很自然，但平台化时不能让 Fetch Worker 直接无限递归。

正确链路：

```text
Fetch Result links
 -> URL Observation
 -> Normalize
 -> Scope Check
 -> Filter
 -> Classification
 -> Dedupe
 -> Priority/Budget
 -> Durable Fetch Task
```

每个 link 都要保留：

- parent URL；
- anchor text；
- depth；
- source snapshot；
- observed_at；
- provider type=`PAGE_LINK`；
- filter decision 和 reason。

这样可以避免：

- calendar trap；
- tag/category 无限组合；
- query permutation；
- logout/login/cart 等业务 URL；
- 外链扩散；
- 同一 URL 由多个页面反复发现而重复抓取。

Deep Crawl 必须有 max depth、max URLs、duplicate ratio、wall clock、bytes 和 Browser seconds 预算。

---

## 9. 多 URL 配置匹配：把“站点专用代码”降为声明式 Profile

官方 `arun_many()` 支持针对 URL pattern 使用不同 `CrawlerRunConfig`。这正好对应平台的 Site Profile / Route Release。

一个站点内部可能同时存在：

```text
/blog/*          -> HTML article extractor
/docs/*          -> docs extractor
*.pdf            -> PDF parser
/api/posts/*     -> JSON extractor
/search/*        -> discovery only
/account/*       -> hard reject
```

生产系统应把这些做成版本化 matcher/recipe：

```text
fetch_route_rule
- matcher
- resource_kind
- fetch_route
- extraction_policy
- browser_recipe nullable
- priority
- release_id
```

规则第一匹配或显式 priority，且必须有默认 fallback。新增第 1001 个站点时优先生成/审核 Profile，而不是新建一份 Python crawler。

---

## 10. 从脚本到 1000 站平台还缺什么

单站脚本通常没有以下能力，而这些恰恰决定长期可维护性：

### 10.1 Coverage

脚本知道“队列空了”，但不知道“历史是否完整”。平台必须记录各 Provider 的 cursor、exhaustion reason、known gap 和 evidence。

### 10.2 Durable State

脚本的 `asyncio.Queue`、crawler frontier、进程内 `set()` 都不能当真相。平台必须把 Observation、Task、Attempt、Snapshot、Version 持久化。

### 10.3 Stable Identity

同一文章可能经 redirect、canonical、Feed GUID、CMS ID、多 URL Alias 被发现。必须把 URL Locator 与 Document Identity 分开。

### 10.4 Incremental Sync

后续不能每天重抓全历史，需要：

- Feed/API cursor；
- Sitemap Conditional GET；
- ETag/Last-Modified；
- overlap window；
- high-yield surface；
- 周/月 Reconcile。

### 10.5 Reprocess Without Refetch

抽取器、Markdown、Chunk、Embedding、Graph、LLM 升级后，应从 Snapshot/Version 重放，不重新打源站。

### 10.6 Web 管理

运营人员需要看到：

- Source、Profile、Coverage；
- 当前 frontier/queue lag；
- Browser Pool 活跃实例和内存；
- adaptive concurrency 当前值/目标值；
- 429/503/timeout；
- 单 URL pipeline trace；
- Raw / Canonical / Filtered Markdown diff；
- quarantine/retry；
- sitemap child shard checkpoint。

---

## 11. 建议的 Fetch Worker 执行契约

### 11.1 输入

```text
FetchTask
- task_id
- source_id
- normalized_url
- route_release_id
- profile_release_id
- fetch_epoch
- priority
- retry_budget
- deadline
- lease/fencing_token
```

### 11.2 执行

```text
lease tasks
 -> group by route/profile/domain
 -> acquire platform permits
 -> choose HTTP or Browser adapter
 -> process bounded window with streaming completion
 -> persist each URL independently
 -> release permit
```

Browser Adapter 内部可以使用 Crawl4AI：

```text
long-lived AsyncWebCrawler
 + MemoryAdaptiveDispatcher
 + RateLimiter
 + per-URL CrawlerRunConfig matcher
 + stream=True
```

### 11.3 输出

```text
FetchAttempt
Snapshot
New URL Observations
Next-stage Tasks
Metrics/Trace
```

Worker 不直接创建 Document Version；Version 应在 Snapshot 经过 extraction + identity 后生成。

---

## 12. 背压模型

背压不能只看“crawler 有没有空闲槽位”。建议同时观察：

```text
fetch queue lag
snapshot upload lag
extract queue lag
DB commit p95
object storage p95
worker RSS
browser RSS
429/503 rate
error rate
```

简单控制逻辑：

```text
if memory high or downstream lag high:
    shrink local concurrency
elif 429/503 high:
    shrink host/domain concurrency + backoff
elif all healthy and queue lag high:
    grow local concurrency up to platform permit ceiling
```

这相当于把 Crawl4AI 的 memory-adaptive 思路从“只看浏览器”扩展为“采集流水线闭环控制”。

---

## 13. Web 管理新增建议

本次调研建议在现有 Web 管理中新增两个视图。

### 13.1 Streaming / Frontier 视图

展示：

- discovery emitted URL/s；
- durable Observation commit/s；
- pending/leased/running task；
- current batch window；
- downstream extract lag；
- provider checkpoint；
- oldest task age；
- known gap。

它能快速定位“发现过快、抓取跟不上”或“抓取很快、DB/对象存储跟不上”。

### 13.2 Browser Pool 视图

展示：

- worker 数；
- browser instances；
- active contexts/pages；
- worker/browser RSS；
- current adaptive concurrency；
- browser restart/crash rate；
- average pages per browser lifetime；
- HTTP -> Browser fallback ratio；
- Browser seconds/Source；
- janitor recycle reason。

这样才能判断 Browser 是否被滥用，以及复用是否真的降低成本。

---

## 14. 本次对《博客知识库技术方案》的优化结论

现有方案的总体方向正确：已经具备 Coverage、Snapshot、Canonical IR、Durable Task、HTTP First、Projection、增量同步、Web 管理等生产级骨架。本次文章没有推翻架构，而是把执行层进一步具体化，建议并已纳入最终方案的重点包括：

1. **增加 Durable Streaming Frontier**：Provider/Seeder 不一次性物化全部 URL，Observation 小批提交并形成持久化 frontier；超大 Sitemap Index 按 child shard checkpoint。
2. **明确 Fetch Worker 为 bounded-window streaming**：`arun_many(stream=True)` 只处理局部 lease 窗口，每个 URL 完成即提交，实现 bounded memory 和 partial success。
3. **增加 Browser Pool 生命周期治理**：长期复用 browser/crawler，按 Source/Profile/安全域隔离 session，按页面数、RSS、寿命和 crash 自动 recycle。
4. **明确两层并发控制**：平台全局/domain/source permit 是上限；`MemoryAdaptiveDispatcher`/Semaphore/RateLimiter 只做 Worker 内第二层自适应执行控制。
5. **强化有损内容边界**：`PruningContentFilter/fit_markdown` 只能生成 `FILTERED_MARKDOWN` Projection，不能覆盖 Snapshot、Canonical IR 或 Canonical Markdown。
6. **链接回流 Discovery Plane**：页面内部 links 必须经过 Observation → Normalize → Filter → Classify → Dedupe → Budget，禁止 Fetch Worker 无界递归。
7. **Web 管理增加 Streaming/Frontier 与 Browser Pool 视图**，让 backpressure、adaptive concurrency、浏览器复用和队列水位可观测。

---

## 15. 风险与边界

- 遵守 robots、站点使用条款和抓取许可策略；不把反爬绕过作为默认能力。
- Browser/Proxy 不能跨租户共享敏感状态。
- 任何自动发现的外链都必须重新做 SSRF、DNS/IP、origin scope 校验。
- Pruning/LLM/Embedding 等有损或模型派生能力不能成为唯一事实源。
- 源站内容可能更新或消失，因此 Snapshot First 比“只保存 Markdown”更重要。
- 大规模并发必须以源站友好为前提；吞吐最大化不是唯一目标，稳定、可恢复和可证明更重要。

---

## 16. 结论

这篇实践证明了 Crawl4AI 在“批量 URL + 并发 + Markdown + 内容过滤 + 链接发现”执行层上的适配性。对 1000 个技术博客的方案而言，最合理的定位不是让 Crawl4AI 接管整个调度系统，而是把它作为 **可替换的 Fetch/Browser Execution Adapter**：平台负责 Coverage、持久化 frontier、幂等、版本、预算、增量与审计；Crawl4AI 负责单 Worker 内的高效页面执行、浏览器复用、流式多 URL 抓取和局部自适应并发。

这样既能利用 Crawl4AI 的工程能力，又不会把百万级长期知识库的可靠性绑定到 crawler 内存状态。