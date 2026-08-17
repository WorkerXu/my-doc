# Crawl4AI 代码示例

## 1. 调研对象与证据边界

- 编号：30
- 名称：Crawl4AI 代码示例
- 原始地址：https://www.reddit.com/r/DataHoarder/comments/1iknxwj
- 状态：已调研

原 Reddit 页面当前无法稳定直接抓取，因此不伪造原帖全文。可核验材料包括：

1. 2025-02-24 的 Medium 文章《Custom web scraper for Amazon Bedrock KB with metadata》直接引用该 Reddit 帖，并复现了关键 Crawl4AI 调用片段；
2. Crawl4AI 当前官方文档（v0.9.x）对 `AsyncWebCrawler`、`CrawlerRunConfig`、`arun_many()`、`MemoryAdaptiveDispatcher`、Markdown/fit Markdown 的定义；
3. Crawl4AI 官方文档明确说明 `markdown_v2` 在 v0.5 后已移除，应使用 `result.markdown`。

因此本文把“可直接核验的代码事实”和“面向 1000 站长期知识库的工程推导”分开。核心结论不是照搬一个 demo，而是把 demo 暴露出的执行能力放到正确的平台边界中。

---

## 2. 可核验代码与直接含义

Medium 文章复现了如下关键代码形态：

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
        file_name = result.url.replace("https://www.", "") \
                              .replace(" ", "_") \
                              .replace(".", "_") \
                              .replace("/", "-") \
                              .replace(":", "")
```

这段代码至少确认了五点：

1. 使用 `AsyncWebCrawler` 的异步生命周期；
2. 单 URL 抓取入口是 `arun()`；
3. 通过 `result.success` 判断执行结果；
4. 同时消费完整 Markdown 与过滤后的 Fit Markdown；
5. demo 直接把抓取结果映射为知识库文件。

对单站 demo 这些做法足够，但对 1000+ 技术博客、百万级文章的长期系统，需要重新定义边界。

---

## 3. 关键版本问题：`markdown_v2` 已经不是当前 API

旧代码使用：

```python
result.markdown_v2.raw_markdown
result.markdown_v2.fit_markdown
```

当前官方文档使用：

```python
result.markdown.raw_markdown
result.markdown.markdown_with_citations
result.markdown.references_markdown
result.markdown.fit_markdown
result.markdown.fit_html
```

官方文档明确说明：`markdown_v2`、顶层 `fit_markdown`、顶层 `fit_html` 在 v0.5 后移除，应从 `result.markdown` 读取。

这类 API 演进对长期系统非常重要。若业务代码、数据库字段、导出逻辑直接依赖某一版 Crawl4AI 的 DTO，那么一次库升级可能迫使整条流水线改结构，甚至导致历史回放不可重现。

正确做法是把 Crawl4AI 放在 Adapter 边界：

```text
Crawl4AI / Playwright / HTTP client
              │
              ▼
       Fetch Adapter
              │
              ▼
     FetchAdapterResult
              │
              ▼
Snapshot / Extract / IR / Version / Export
```

稳定内部契约建议：

```text
FetchAdapterResult
- requested_url
- final_url
- redirect_chain[]
- success
- status_code
- response_headers
- content_type
- raw_body_ref/raw_html_ref nullable
- cleaned_html nullable
- raw_markdown nullable
- fit_markdown nullable
- markdown_with_citations nullable
- references_markdown nullable
- metadata nullable
- links[]
- media[]
- extracted_content nullable
- error_class nullable
- error_message nullable
- dispatch_metrics nullable
- runtime_release_id
- adapter_release_id
```

旧版 `markdown_v2.raw_markdown` 与新版 `markdown.raw_markdown` 都只映射为平台内部 `raw_markdown`。第三方字段名变化不进入 Snapshot、Document、Markdown Export 的核心数据模型。

---

## 4. `AsyncWebCrawler` 的正确生命周期

`AsyncWebCrawler` 的价值之一是复用浏览器和执行环境，而不是每个 URL 启动一次 Chromium。

错误的生产方式：

```text
for url in urls:
    launch browser
    crawl url
    close browser
```

这样会反复支付：浏览器冷启动、context 初始化、renderer 创建、脚本和字体加载等成本，也更容易产生进程抖动。

推荐：

```text
Browser Worker
  └─ Browser Pool
       ├─ long-lived browser/crawler A
       │    ├─ context/session 1
       │    └─ context/session 2
       └─ long-lived browser/crawler B
```

Browser 可以跨批次复用，但 Session 不能无条件共享。Cookie、LocalStorage、IndexedDB、登录态、代理身份必须按 tenant/source/profile/security scope 隔离。

回收条件应包括：最大页面数、最大存活时间、RSS soft/hard limit、renderer crash、context 清理失败、Runtime Release 切换和 worker drain。

---

## 5. HTTP First，Browser Last

示例使用 Crawl4AI 不意味着所有网站都应该走 Browser。

以下应优先 HTTP：

```text
Sitemap
RSS/Atom/JSON Feed
WordPress/Ghost/Substack API
JSON API
静态 HTML
Markdown/TXT
PDF/文件流
```

只有静态抓取无法得到合格正文时才升级 Browser，例如：

- SPA/CSR；
- Load More；
- Infinite Scroll；
- 必须执行 JS 才出现正文；
- 交互后才加载文章；
- HTTP 版本正文质量明显低于 Browser。

这样可以把昂贵的 Browser 预算集中在少量确实需要的 URL 上。

---

## 6. Sitemap/Seeder 是历史发现入口，不是完整性证明

技术博客经常提供 Sitemap，因此它非常适合全量历史回填：

- 枚举成本低；
- 可按 Sitemap Index/child sitemap 分片；
- `<lastmod>` 可作为增量候选；
- 不必先加载正文页面才能发现 URL。

但“读完 Sitemap”不等于“历史抓全”，因为可能存在：

- 只保留近期文章；
- 老文章迁移后未收录；
- 某些栏目不进 Sitemap；
- `<lastmod>` 被批量重写；
- CMS 插件规则变化；
- 已删除页面完全不可见。

因此历史 Coverage 应交叉多个 Provider：

```text
CMS/API
 -> Sitemap
 -> Feed
 -> Archive
 -> Category/Tag/Author
 -> Docs TOC
 -> Common Crawl
 -> 受控 Deep Crawl
```

Sitemap 处理应分 shard checkpoint：

```text
conditional fetch
 -> stream XML parse
 -> 100~1000 URL mini-batch
 -> url_observation commit
 -> task publish
 -> checkpoint advance
```

不能把几十万 URL 一次性物化到进程内存。

---

## 7. Discovery 与 Fetch 必须解耦

无论 URL 来自 Sitemap、Feed、页面链接还是 Common Crawl，都应先保存 Observation：

```text
url_observation
- source_id
- observed_url
- provider_type
- provider_run_id
- parent_url nullable
- anchor_text nullable
- depth nullable
- observed_at
- evidence_ref
```

再进入：

```text
Observation
 -> Normalize
 -> Scope Check
 -> Filter
 -> Classify
 -> Dedupe
 -> Priority/Budget
 -> Durable Fetch Task
```

这样才能实现：

- Worker 重启不丢 frontier；
- Filter 升级可回放；
- 能解释“为什么没抓这个 URL”；
- 多 Provider 对同一 URL 合并证据；
- Deep Crawl 有硬预算；
- Source 暂停后状态仍可恢复。

页面里的 `result.links` 同样必须回到这一链路，不能让 Fetch Worker 自己无限递归。

---

## 8. `arun_many()` 的真正价值：有界并发 + 流式完成

当前官方 `arun_many()` 支持多 URL 并发、URL-specific `CrawlerRunConfig`、Dispatcher 和 `stream=True`。

典型流式形态：

```python
async for result in await crawler.arun_many(urls, config=stream_config):
    await process_result(result)
```

这比“所有 URL 完成后一起处理”更适合知识库，因为每个 URL 可以独立：

- 保存 Fetch Attempt；
- 保存 Snapshot；
- 写 Object Storage；
- 产生 Extract Task；
- 新 links 形成 Observation；
- ACK 当前任务。

但不能把全站 50 万 URL 一次性交给 `arun_many()`。正确方式是 bounded lease window：

```text
lease 20~200 tasks
 -> acquire platform permits
 -> arun_many(stream=True)
 -> per-result commit
 -> refill below low watermark
```

这样把内存、失败范围和恢复粒度都限制在一个小窗口内。

---

## 9. Dispatcher 只是 Worker 本地控制，不是平台调度真相

当前官方提供：

```text
MemoryAdaptiveDispatcher
SemaphoreDispatcher
RateLimiter
```

这些很适合在单 Worker 内基于内存和局部速率进行自适应，但平台还需要第一层全局控制：

```text
Global
 -> route
 -> registrable-domain
 -> source
 -> host
```

原则：

```text
worker_local_concurrency <= platform_permit_ceiling
```

本地 Dispatcher 可以因为 RSS、429/503、browser crash、DB/Object Storage backlog 而收缩，但不能突破平台对域名和 Source 的上限。

Retry 也必须 durable。Crawl4AI 内部重试只算单次执行的局部行为，平台还应保存：`attempt`、`retry_budget`、`next_retry_at`、`last_error_class`、`last_status_code`。

---

## 10. `fit_markdown` 不能当归档真相

示例直接把 `fit_markdown` 写到下游知识库，这对 demo 很方便，但对历史知识库风险很高。

Fit Markdown 是有损过滤结果，可能误删：

- 短警告和版本提示；
- API 参数短说明；
- 代码附近的短文本；
- 脚注；
- 低文本密度表格；
- 某些结构性上下文。

因此必须分层：

```text
Snapshot / Raw HTML       网络事实
        │
        ▼
Canonical IR             结构事实
        │
        ├─ Canonical Markdown      完整归档
        └─ Filtered/Fit Markdown   检索/RAG Projection
```

Filter 升级时从旧 Snapshot/IR 离线重建，而不是重新访问源站。

---

## 11. 标题提取不能取 Markdown 第一行

示例使用：

```python
raw_markdown.split("\n")[0]
```

这很脆弱，第一行可能是：

- 空白；
- 导航文本；
- logo alt；
- cookie banner；
- 非文章标题的 H1；
- Markdown generator 生成的辅助内容。

生产系统应多源融合：

```text
CMS/API title
JSON-LD headline
OpenGraph title
HTML <title>
Article H1
Markdown heading fallback
```

同时保存 field confidence 与 source evidence。标题只是 Canonical IR 的一个字段，不应由某个 Markdown 生成器的排版细节决定。

---

## 12. URL 字符串替换不能当文件身份

示例通过 `replace()` 把 URL 转成文件名。这个方法在 demo 可用，但生产环境会遇到：

- 不同 URL 可能碰撞；
- query 被错误折叠；
- redirect/canonical 变化导致重复文件；
- 超长路径；
- 文件系统非法字符；
- 同一文章多个 alias 产生多份文件。

正确做法：

```text
Document Identity != URL != Export Path
```

内部使用稳定 `document_id/document_version_id`，导出路径只是 Projection：

```text
{source_slug}/{published_year}/{document_id}-{safe_slug}.md
```

Front Matter 保存 canonical URL、source、version、content hash 和 release，从文件可以反查平台事实。

---

## 13. 增量同步不能每天重跑全量脚本

全量历史回填完成后，增量应优先使用：

```text
Feed/API cursor + overlap
Sitemap Conditional GET
ETag / Last-Modified
high-yield surfaces
recent archive pages
periodic reconcile
```

文章页面使用 `If-None-Match`、`If-Modified-Since`。304 不重复保存 body，不重新抽取，不创建新 Document Version。

只有 Canonical IR 或关键元数据真正变化时才创建新 Version。Markdown/Chunk/Embedding/Graph/AI 策略升级只重建 Projection。

---

## 14. 对主技术方案的具体优化

本次调研确认原方案的大方向已经正确：Durable Frontier、bounded-window streaming、Browser Pool、两层并发、Snapshot First、Canonical/fit 分层、页面链接回流 Discovery Plane 都已经存在。

真正值得补充的是 **Crawl4AI 版本演进与 Adapter 稳定契约**。因此主方案新增/强化了以下内容：

1. 增加 `Adapter Is Boundary` 原则；
2. Release Registry 增加 `fetch_adapter_release`；
3. FETCH 幂等键纳入 adapter/runtime release；
4. 定义稳定 `FetchAdapterResult`；
5. Snapshot 保存 `runtime_release_id`、`adapter_release_id`；
6. 明确 `markdown_v2 -> markdown` 等 API 变化只在 Adapter 内转换；
7. Web 管理新增 Adapter/Runtime 视图、normalization error、old/new canary diff；
8. 明确 title 不取 Markdown 第一行；
9. 明确 URL 简单字符串替换不能作为文件身份；
10. 增加 Adapter fixture/regression/canary 验证流程。

这使平台在后续升级 Crawl4AI 时，不需要修改事实层和下游知识库格式。

---

## 15. 推荐 Fetch Worker 执行契约

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

执行：

```text
lease bounded window
 -> group by route/profile/domain
 -> acquire platform permits
 -> choose HTTP/Browser adapter
 -> Crawl4AI arun/arun_many
 -> normalize third-party DTO
 -> persist each URL independently
 -> emit Snapshot / Observation / Extract Task
 -> release permit
```

输出：

```text
FetchAttempt
FetchAdapterResult trace
Snapshot
New URL Observations
Next-stage Tasks
Metrics
```

Fetch Worker 不直接创建 Document Version，也不直接把 `fit_markdown` 当最终知识库文件。

---

## 16. 风险与边界

- 默认遵守 robots、站点条款、版权和抓取许可策略；
- 不把绕过认证、付费墙或反爬作为默认能力；
- Browser/Proxy/Session 不跨安全域共享敏感状态；
- 起始 URL、redirect、Browser navigation、页面新链接每跳重新执行 SSRF/DNS/IP/origin scope 校验；
- Crawl4AI 的 cache、dispatcher、进程内 frontier 都不是 durable truth；
- `PruningContentFilter`、BM25、LLM、Embedding 都不能成为唯一事实源；
- 第三方库升级必须固定版本、fixture、回归、canary 后再切换；
- 源站友好、可恢复、可审计、可回放优先于极限吞吐。

---

## 17. 结论

这篇 Crawl4AI 示例证明 Crawl4AI 非常适合承担“单 Worker 内高效页面执行”的职责：异步抓取、浏览器执行、Markdown、内容过滤、链接发现和局部并发都能直接利用。

但 1000 个技术博客的长期知识库不能把 Crawl4AI 本身当系统真相。正确边界是：

```text
平台负责：
Coverage / Durable Frontier / Task / Idempotency / Retry / Snapshot
Identity / Version / Release / Incremental Sync / Audit / Web 管理

Crawl4AI Adapter 负责：
Browser Execution / arun_many / Dispatcher / Markdown candidate
Structured candidate / Links / Runtime-local optimization
```

通过稳定 Adapter 契约隔离 `markdown_v2`、`markdown` 以及未来 API 演进，可以既利用 Crawl4AI 的执行效率，又不把百万级、长期运行的知识库可靠性绑定到某个第三方版本的对象模型。