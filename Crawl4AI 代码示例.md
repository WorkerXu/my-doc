# Crawl4AI 代码示例

## 1. 调研对象与证据边界

- 编号：30
- 名称：Crawl4AI 代码示例
- 来源标题：Crawl4Ai code example
- 原始地址：https://www.reddit.com/r/DataHoarder/comments/1iknxwj
- 调研状态：仅读取为“调研中”，未修改状态文件

原 Reddit 页面在当前抓取环境中无法稳定直接读取，因此本文不伪造原帖全文或不可核验的代码。可以确认的材料有三类：

1. 一篇 2025-02-24 发布的 Medium 文章《Custom web scraper for Amazon Bedrock KB with metadata》直接链接该 Reddit 帖，并复现了其中关键 Crawl4AI 调用片段；
2. 当前 Crawl4AI 官方仓库源码与官方文档，核验提交为 `7e801521428ee12509994d39151006f64055ebe3`；
3. 已归档索引摘要描述了“Sitemap/Seed URL → AsyncWebCrawler → PruningContentFilter → Markdown，并带并发、延迟、重试和链接继续入队”的整体模式。

因此以下分析将“可直接核验的代码事实”和“根据摘要还原的系统模式”分开，不把后者冒充为原帖逐行代码。

这篇实践真正有价值的部分不是某个单站脚本，而是它暴露出生产采集平台的几个关键执行问题：**如何批量发现 URL、如何有界并发、如何复用浏览器、如何按结果流式提交、如何把有损 Markdown 与归档真相分离，以及如何把新发现链接重新送回可审计的发现链路。**

---

## 2. 可核验代码、API 演进与版本兼容

### 2.1 2025 年代码片段使用旧版 `markdown_v2`

Medium 文章引用该 Reddit 示例时，复现了如下调用形态：

```python
async with AsyncWebCrawler() as crawler:
    result = await crawler.arun(url=url, config=run_config)
    if result.success:
        data = {
            "url": result.url,
            "title": result.markdown_v2.raw_markdown.split("\n")[0]
                     if result.markdown_v2.raw_markdown else "No Title",
            "content": result.markdown_v2.fit_markdown,
        }
```

这至少确认了几个事实：

- 使用 `AsyncWebCrawler` 的异步生命周期；
- 单 URL 执行入口是 `arun()`；
- 成功与失败通过 `result.success` 分支；
- 结果同时消费完整 Markdown 与过滤后的 Fit Markdown；
- 脚本把抓取结果直接写成下游知识库文件。

### 2.2 当前官方 API 已统一到 `result.markdown`

当前 Crawl4AI 官方 `CrawlResult` 文档中，Markdown 入口是：

```python
result.markdown.raw_markdown
result.markdown.markdown_with_citations
result.markdown.references_markdown
result.markdown.fit_markdown
result.markdown.fit_html
```

而不是旧示例中的 `result.markdown_v2`。

官方源码的 HTML 处理链路也表明：

```text
raw HTML
 -> scraping_strategy.scrap()
 -> cleaned_html
 -> choose markdown content_source
 -> MarkdownGenerationStrategy.generate_markdown()
 -> MarkdownGenerationResult
```

其中 Markdown 的输入源可以是：

```text
raw_html
cleaned_html
fit_html
```

这个版本漂移非常重要。一个需要运行多年的 1000 站知识库平台，不能让业务层、数据库模型或导出逻辑直接依赖某一版 Crawl4AI 的 DTO 字段名。

### 2.3 正确做法：Crawler Adapter 输出内部稳定契约

Crawl4AI 只能是 Fetch/Browser Adapter。Adapter 内部吸收版本差异，对平台只暴露稳定结果：

```text
FetchAdapterResult
- requested_url
- final_url
- success
- status_code
- response_headers
- raw_body_ref / raw_html
- cleaned_html nullable
- raw_markdown nullable
- fit_markdown nullable
- links[]
- media[]
- extracted_content nullable
- error_class nullable
- error_message nullable
- dispatch_metrics nullable
- runtime_release_id
- adapter_release_id
```

旧版：

```text
result.markdown_v2.raw_markdown
```

新版：

```text
result.markdown.raw_markdown
```

都只在 Adapter 内转换成：

```text
FetchAdapterResult.raw_markdown
```

因此 Crawl4AI 升级时，只需要：

1. 升级 Runtime Artifact Release；
2. 更新 Adapter；
3. 跑固定 fixture/regression；
4. canary；
5. 再切换 active runtime。

不应该修改下游 Snapshot、Document、Markdown、Chunk、Export 的核心数据契约。

现有《博客知识库技术方案》已经包含 Crawl4AI Adapter、Runtime Artifact Release、Release Everything 和固定 fixture/canary 的治理原则，因此这次版本核验不需要再新增主架构组件，但实现时必须把“第三方返回对象归一化”作为 Adapter 的硬边界。

---

## 3. 核心流程还原

根据归档摘要和当前官方能力，可以把该实践抽象为：

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
        ├─ MemoryAdaptiveDispatcher / SemaphoreDispatcher
        ├─ RateLimiter / Retry
        └─ Browser Instance Reuse
        │
        ▼
Result-by-result processing
        │
        ├─ Snapshot
        ├─ raw / cleaned HTML
        ├─ raw Markdown
        ├─ fit Markdown
        ├─ quality check
        └─ discovered links
        │
        ▼
Durable persistence
        │
        └─ links re-enter Discovery Plane
```

关键不是“并发越高越好”，而是：

- URL 发现流式化；
- 执行窗口有界；
- 每个 URL 单独提交；
- 浏览器进程复用；
- Session 隔离；
- 下游慢时能背压；
- 任何新链接都重新经过平台规则；
- 有损内容不能覆盖事实层。

---

## 4. Sitemap/Seeder：高价值历史入口，但不是完整性证明

### 4.1 为什么 Sitemap 适合技术博客历史回填

技术博客、新闻站、文档站经常暴露：

```text
/sitemap.xml
/sitemap_index.xml
/post-sitemap.xml
/news-sitemap.xml
```

相比从首页递归爬，Sitemap 的优势是：

- 枚举效率高；
- URL 通常由站点维护者主动声明；
- Sitemap Index 可以按 child sitemap 分片；
- `<lastmod>` 可作为增量候选信号；
- 不必先加载正文页面才能发现文章。

### 4.2 Sitemap 不是历史真相

不能把“读完 Sitemap”等价为“历史抓全”。常见缺口：

- 只保留近期文章；
- 老文章已迁移但未保留；
- 某些栏目或静态页不进 Sitemap；
- `<lastmod>` 不准确；
- CMS 插件生成规则改变；
- 删除页面与历史页面不可见。

因此 Sitemap 必须只是 `Coverage Provider` 之一，与以下入口交叉：

```text
CMS/API
RSS/Atom/JSON Feed
Archive
Category/Tag/Author
Docs TOC
Common Crawl
受控 Deep Crawl
```

### 4.3 大 Sitemap 必须按 shard checkpoint

对于几十万 URL 的站点：

```text
sitemap index
  ├─ shard A -> cursor/checkpoint A
  ├─ shard B -> cursor/checkpoint B
  └─ shard C -> cursor/checkpoint C
```

每个 shard 独立：

1. Conditional GET；
2. 解压和流式 XML 解析；
3. 小批写 `url_observation`；
4. normalization/filter/classification；
5. 生成 durable Fetch Task；
6. 保存 ETag/Last-Modified/body hash；
7. 提交 checkpoint。

这样 worker 重启不会从整个站点头部重新开始，也不会把几十万 URL 全部放进进程内数组。

---

## 5. Discovery 和 Fetch 必须解耦

无论 URL 来自 Sitemap、Seeder 还是页面内链接，首先应该形成 Observation：

```text
url_observation
- source_id
- observed_url
- provider_type
- provider_ref
- parent_url nullable
- anchor_text nullable
- depth nullable
- observed_at
- evidence
```

然后再进入：

```text
Observation
 -> Normalize
 -> Scope Check
 -> Filter
 -> Classification
 -> Dedupe
 -> Priority/Budget
 -> Durable Task
```

这比“发现一个链接马上递归 `arun()`”可靠得多，因为：

- Worker 崩溃不丢 frontier；
- Filter 升级可重放；
- 每个 URL 为什么没抓可以解释；
- Source 暂停后任务仍存在；
- 多 Provider 发现同一 URL 可以合并证据；
- Deep Crawl 可以严格受预算控制。

---

## 6. `AsyncWebCrawler` 生命周期与 Browser Pool

### 6.1 不要每 URL 启一个浏览器

浏览器成本主要来自：

- Chromium 冷启动；
- browser/context 初始化；
- JS runtime；
- renderer process；
- 字体、图片、脚本和网络资源；
- 页面关闭不干净导致的内存残留。

因此推荐：

```text
Browser Worker process
  └─ Browser Pool
       ├─ long-lived crawler/browser 1
       │    ├─ context A
       │    └─ context B
       └─ long-lived crawler/browser 2
```

Crawler/Browser 跨批次复用，但达到以下条件应 recycle：

- 最大页面数；
- 最大存活时间；
- RSS soft/hard limit；
- renderer crash 比例升高；
- page/context 泄漏；
- Runtime Release 切换；
- worker drain。

### 6.2 Browser 可共享，Session 不可无条件共享

可复用：

- Browser process；
- browser binary；
- worker 内池基础设施。

不能跨租户/Source/安全域随意共享：

- Cookie；
- LocalStorage；
- IndexedDB；
- 登录 token；
- Service Worker；
- 代理身份；
- 认证上下文。

推荐池键：

```text
(runtime_release, profile/security_scope, proxy_scope)
```

### 6.3 HTTP First，Browser Last

示例使用 Crawl4AI 不代表全平台都该走浏览器。以下应优先 HTTP：

```text
Sitemap
Feed
CMS API
JSON API
静态 HTML
Markdown/TXT
文件下载
```

只有在静态 HTTP 无法得到合格正文时，才升级 Browser：

- JS 客户端渲染；
- Load More；
- Infinite Scroll；
- SPA route；
- 必须交互后正文才出现；
- HTTP 抽取质量不达标。

---

## 7. `arun_many()`：价值不只是并发，而是流式完成

当前官方 `arun_many()` 支持：

```text
List[str] / task list
single or per-URL CrawlerRunConfig
custom dispatcher
stream=True
CrawlResult per URL
```

流式模式下结果按完成逐个消费：

```python
async for result in await crawler.arun_many(urls, config=stream_config):
    process(result)
```

### 7.1 为什么不能把整个站点一次性交给 `arun_many()`

即使库支持批量，也不能把 50 万 URL 一次性变成一个 batch：

- 输入 URL 自身占内存；
- 返回结果会形成大对象；
- 最慢任务拖累整批；
- 下游对象存储/数据库消费不及时会堆积；
- 进程崩溃难以恢复精确进度；
- 无法公平调度多个 Source。

### 7.2 正确方式：bounded lease window

```text
1. lease N tasks
2. 获取 domain/source/global permit
3. 将本地窗口交给 arun_many(stream=True)
4. 每个 result 完成：
   - 保存 Fetch Attempt
   - 保存 Snapshot
   - 保存 links Observation
   - 产生 Extract Task
   - ACK 当前 Task
5. 窗口下降到 low watermark 后 refill
```

N 是几十到数百的执行窗口，而不是站点总 URL 数。

### 7.3 逐 URL Commit 与 partial success

网络、对象存储、数据库无法做一个廉价的跨系统大事务。因此使用：

- at-least-once delivery；
- stable idempotency key；
- snapshot/content hash；
- per-URL commit；
- retry item independently。

某一 URL 失败不能回滚已经成功的 99 个 URL。

---

## 8. Dispatcher、RateLimiter 与两层并发

当前官方文档提供：

```text
MemoryAdaptiveDispatcher
SemaphoreDispatcher
RateLimiter
```

它们非常适合做 **Worker-local execution control**，但不能成为平台调度真相。

### 8.1 第一层：平台 Permit

```text
Global
 -> route
 -> registrable domain
 -> source
 -> host
```

平台决定某 URL 是否有资格启动。

### 8.2 第二层：Worker Local Dispatcher

Worker 再依据自身资源收缩：

- memory/RSS；
- browser pages；
- event loop lag；
- p95 latency；
- 429/503；
- browser crash；
- snapshot upload backlog；
- DB commit latency。

原则是：

```text
local concurrency <= platform permit ceiling
```

本地可以因为资源压力降并发，绝不能绕过平台配额提高并发。

### 8.3 Retry 必须 durable

Crawl4AI 的 RateLimiter/重试只能算 Worker 内尝试。平台还必须持久化：

```text
attempt
retry_budget
next_retry_at
last_error_class
last_status_code
```

分类建议：

- 429/503：尊重 `Retry-After`，降低 host concurrency；
- timeout/reset：指数退避 + jitter；
- 401/403：策略判断，不盲重试；
- 404/410：记录 tombstone/缺失证据；
- robots/policy：策略终止；
- browser crash：换实例有限重试；
- extract failure：优先离线重放 Snapshot。

---

## 9. `PruningContentFilter` / `fit_markdown` 不能作为归档真相

原示例把 `fit_markdown` 直接写到下游知识库文件，这对 demo 很方便，但对长期历史知识库风险很高。

`fit_markdown` 是有损结果，可能删除：

- 短警告；
- API 参数说明；
- 代码旁的短文本；
- 脚注；
- 低文本密度表格；
- 版本提示；
- 某些导航层级但具有上下文意义的内容。

正确分层：

```text
Snapshot / Raw HTML          不可变网络事实
        │
        ▼
Canonical IR                结构事实
        │
        ├─ Canonical Markdown      完整默认归档
        └─ Filtered/Fit Markdown   面向检索的有损 Projection
```

因此：

- 保存原始响应/HTML；
- 保存可回放 Canonical IR；
- Canonical Markdown 不走 Pruning；
- `fit_markdown` 作为 `FILTERED_MARKDOWN` Projection；
- RAG 可以优先索引 fit 版本；
- 导出归档默认 canonical；
- Filter 升级从旧 Snapshot/IR 重建，不重新抓源站。

---

## 10. 页面链接继续入队：必须回 Discovery Plane

页面抓完后继续处理 `result.links` 是合理的，但不能让 Fetch Worker 自己做无界递归。

推荐：

```text
result.links
 -> URL Observation
 -> Normalize
 -> SSRF/Scope
 -> Filter
 -> Classification
 -> Dedupe
 -> Priority/Budget
 -> Durable Fetch Task
```

每条 link 保存：

- parent URL；
- anchor text；
- depth；
- source snapshot；
- observed_at；
- provider=`PAGE_LINK`；
- filter decision/reason。

Deep Crawl 必须设置硬预算：

```text
max_depth
max_urls
max_bytes
max_wall_clock
max_browser_seconds
max_duplicate_ratio
```

否则非常容易进入 calendar trap、tag/query permutation、登录/购物车 URL、无限分页或外链扩散。

---

## 11. 多 URL Config 匹配与 Site Profile

当前官方 `arun_many()` 支持针对不同 URL 配置不同 `CrawlerRunConfig`，并按 matcher 匹配。

这可以映射为平台声明式路由：

```text
/blog/*      -> HTML article extractor
/docs/*      -> docs extractor
*.pdf        -> PDF parser
/api/*       -> JSON/API route
/search/*    -> discovery only
/account/*   -> hard reject
```

平台不要把这些规则硬编码成一堆 `if site == ...`，而应放在版本化 Profile/Release：

```text
fetch_route_rule
- matcher
- priority
- resource_kind
- fetch_route
- extraction_policy
- browser_recipe nullable
- release_id
```

新增第 1001 个站点应优先：

```text
PROBE -> Profile Draft -> fixture -> 人工审核 -> Activate
```

而不是新增一份 Python crawler。

---

## 12. 增量同步：不能每天重跑全量脚本

全量历史回填完成后，增量应由多个高收益信号驱动：

```text
Feed/API cursor + overlap
Sitemap Conditional GET
ETag / Last-Modified
high-yield surfaces
recent archive pages
periodic reconcile
```

对文章 URL 再使用：

```text
If-None-Match
If-Modified-Since
```

304 不产生新 Snapshot body；只有 canonical 内容真正变化时才产生新的 Document Version。

对 `<lastmod>` 只能当候选信号，不能当唯一真相，因为部分站点会批量重写 lastmod。

---

## 13. 建议的 Fetch Worker 契约

### 输入

```text
FetchTask
- task_id
- source_id
- normalized_url
- profile_release_id
- route_release_id
- adapter_release_id
- runtime_release_id
- fetch_epoch
- priority
- retry_budget
- lease_until
- fencing_token
```

### 执行

```text
lease bounded window
 -> group by route/profile/domain
 -> acquire platform permits
 -> select HTTP/Browser Adapter
 -> Crawl4AI adapter normalizes library DTO
 -> stream result
 -> persist each URL independently
 -> emit new Observation/Extract Task
 -> release permit
```

### 输出

```text
FetchAttempt
Snapshot
New URL Observations
Next-stage Tasks
Metrics / Trace
```

Fetch Worker 不直接创建 Document Version。Version 应在 Snapshot 经过 Extraction、Quality、Identity 后生成。

---

## 14. 端到端 Backpressure

不能只看 crawler 内存，至少需要联动：

```text
fetch_queue_lag
oldest_fetch_task_age
snapshot_upload_lag
extract_queue_lag
DB_commit_p95
object_store_p95
worker_RSS
browser_RSS
browser_page_count
429_503_rate
fetch_error_rate
```

控制逻辑：

```text
if memory high or downstream lag high:
    shrink worker-local concurrency
elif 429/503 high:
    shrink host/domain permits + backoff
elif healthy and queue lag high:
    grow local concurrency up to platform permit ceiling
```

这把 Crawl4AI 的 memory-adaptive 思路提升为整条采集流水线的闭环控制。

---

## 15. Web 管理需要暴露的执行细节

### 15.1 Frontier/Streaming 视图

- discovery URL/s；
- Observation commit/s；
- pending/leased/running；
- current bounded window；
- oldest task age；
- provider checkpoint；
- extract lag；
- known gap。

### 15.2 Browser Pool 视图

- worker/browser 数；
- active context/page；
- RSS；
- current adaptive concurrency；
- browser recycle reason；
- browser crash rate；
- pages per browser lifetime；
- HTTP → Browser fallback ratio；
- Browser seconds/Source。

### 15.3 Adapter/Runtime 视图

本次版本核验建议额外观察：

- active Crawl4AI/runtime release；
- adapter release；
- result normalization error；
- unknown/missing field rate；
- fixture pass rate；
- old/new runtime canary diff；
- raw Markdown hash diff；
- fit Markdown hash diff。

这样 Crawl4AI 升级造成字段或输出语义变化时，可以在切流前发现。

---

## 16. 对《博客知识库技术方案》的复核结论

当前最终方案已经包含并正确吸收了这篇实践真正值得平台化的部分：

1. Durable Streaming Frontier；
2. bounded-window `arun_many(stream=True)`；
3. per-URL commit / partial success；
4. Browser Pool 与 session 隔离；
5. 平台 Permit + Worker-local Dispatcher 两层并发；
6. Snapshot First；
7. Full Canonical Before Fit Markdown；
8. 页面 links 回流 Discovery Plane；
9. Site Profile/Route Release；
10. Runtime Artifact Release、Adapter 边界与 fixture/canary。

本次额外核验出的“`markdown_v2` → `markdown` API 演进”并不要求再增加新的主架构组件，因为当前方案已经把 Crawl4AI 定位为可替换 Adapter，并对 Runtime 版本化。实现阶段只需要把 **第三方 DTO 归一化契约**落实在 Adapter 内，并增加相关兼容 fixture 与监控即可。

因此本次不重复改写《博客知识库技术方案.md》，避免在已经完整的最终方案中加入重复章节。

---

## 17. 风险与边界

- 遵守 robots、站点条款、版权和抓取许可策略；
- 不把反爬绕过作为平台默认能力；
- Browser/Proxy/Session 不跨租户共享敏感状态；
- 起始 URL、redirect、Browser navigation、页面新链接都重新执行 SSRF/DNS/IP/origin scope 校验；
- `PruningContentFilter`、LLM、Embedding 都不能成为唯一事实源；
- Crawl4AI 内存 frontier、Dispatcher 状态、session cache 都不能当 durable truth；
- 第三方库升级必须固定版本、回归、canary 后再切换；
- 吞吐最大化不是唯一目标，源站友好、可恢复、可审计、可回放优先。

---

## 18. 结论

这篇 Crawl4AI 示例证明了 Crawl4AI 很适合承担“单 Worker 内高效页面执行”的职责：异步抓取、浏览器复用、批量 URL、流式返回、内容过滤、Markdown、链接发现和局部并发控制都能直接复用。

但对 1000 个技术博客的长期知识库，Crawl4AI 不应该成为系统真相层。正确边界是：

```text
平台负责：
Coverage / Durable Frontier / Task / Idempotency / Retry / Snapshot
Identity / Version / Release / Incremental Sync / Audit / Web 管理

Crawl4AI Adapter 负责：
Browser Execution / arun_many / Dispatcher / Markdown candidate
Structured candidate / Links / Runtime-local optimization
```

同时，通过 Adapter 输出稳定内部契约，把 `markdown_v2`、`markdown` 以及未来 Crawl4AI API/行为变化隔离在执行层。这样既能利用 Crawl4AI 的工程能力，又不会把百万级、长期运行的知识库可靠性绑定到某个第三方版本的对象模型或进程内状态。