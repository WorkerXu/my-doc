# searxNcrawl：SearXNG 搜索与 Crawl4AI 抓取集成项目

项目地址：https://github.com/DasDigitaleMomentum/searxNcrawl

## 1. 项目定位

searxNcrawl 是一个把 SearXNG 元搜索、Crawl4AI/Playwright 网页抓取、Markdown 清洗以及 FastMCP/CLI 接口组合在一起的轻量工具。它适合“搜索到 URL -> 抓正文 -> 返回 Markdown/JSON”的交互式工作流，也能进行带 `max_depth/max_pages` 限制的站点深度抓取。

对“1000 个技术博客全量历史文章 + 后续增量同步 + Web 管理”的目标而言，它不应直接作为完整平台使用，但非常适合拆成两类可复用能力：

1. `SearXNG Search Gap Adapter`：站点官方 Provider 覆盖异常时进行旧 URL、迁移 URL 和站外索引缺口诊断。
2. `Crawl4AI Engine Adapter`：作为 Browser/渲染抓取 Worker 的一种实现，并借鉴其 result normalization、timeout、内容过滤和认证处理方式。

项目最大的价值不在“直接拿来跑 1000 个站”，而在于暴露出一批生产化时必须显式处理的第三方 crawler/search 运行时问题。

## 2. 代码结构与调用链

主要模块：

- `crawler/__init__.py`：单页、批量、站点抓取 API。
- `crawler/site.py`：Crawl4AI DFS deep crawl、域名过滤、结果归一化。
- `crawler/config.py`：CrawlerRunConfig、MarkdownGenerator、PruningContentFilter 和选择器策略。
- `crawler/builder.py`：把 Crawl4AI `CrawlResult` 转换为内部 `CrawledDocument`。
- `crawler/markdown_dedup.py`：Markdown 段落级 exact dedup。
- `crawler/mcp_server.py`：FastMCP 的 `crawl`、`crawl_site`、`search` 工具。
- `crawler/auth.py` / `session_capture.py`：Playwright storage state 与登录会话捕获。
- `tests/`：timeout、auth、dedup、CORS、MCP、config 等测试。

典型抓取链路：

```text
MCP/CLI/Python API
 -> crawl_page_async / crawl_pages_async / crawl_site_async
 -> build_markdown_run_config
 -> AsyncWebCrawler.arun
 -> CrawlResult / CrawlResultContainer / list / async generator
 -> build_document_from_result
 -> MarkdownGenerationResult
 -> exact dedup
 -> CrawledDocument
 -> Markdown/JSON 输出
```

搜索链路：

```text
MCP search
 -> 读取 SEARXNG_URL / Basic Auth
 -> httpx.AsyncClient
 -> GET /search?format=json
 -> 网络错误重试
 -> 截断 max_results
 -> 可选 SEARCH_RESULT_FIELDS 字段过滤
 -> JSON 返回
```

## 3. Crawl4AI 抓取实现细节

### 3.1 单页抓取

`crawl_page_async` 每次调用都创建一个 `AsyncWebCrawler` 上下文，然后对 `crawler.arun` 使用 `asyncio.wait_for` 包裹外层 deadline。其优点是简单且能避免 Crawl4AI 内部某个请求无限挂住；缺点是批量生产时创建/销毁 crawler/browser runtime 的成本高。

认证场景通过 `BrowserConfig(storage_state=...)` 把 Playwright storage state 注入 Browser。这个方式适合人工登录后复用 Cookie/LocalStorage，但生产系统必须把 storage state 视为 Secret Artifact 管理，不能随普通配置存储或输出日志。

### 3.2 批量抓取

`crawl_pages_async` 使用 `asyncio.Semaphore(concurrency)` 控制并发，并用 `asyncio.gather` 保持输入顺序。单 URL 失败被转换为 `CrawledDocument(status="failed")`，不会让整个 batch 抛错。

但它内部仍然调用 `crawl_page_async`，也就是说每个 URL 都会独立进入 `async with AsyncWebCrawler()`。在交互式工具中可接受，在百万级页面的知识库系统中应改为：

```text
Worker Process
 -> 长生命周期 Browser/Crawler Runtime
 -> Context Pool
 -> Page Pool
 -> per-site semaphore / token bucket
```

进程内 semaphore 仅保护本 Worker 资源；真正的任务状态、lease、retry、dead-letter 必须持久化。

### 3.3 Site Crawl

`site.py` 实际构造 `DFSDeepCrawlStrategy(max_depth, max_pages, filter_chain)`，并通过 `DomainFilter` 限制 host。`include_subdomains` 时使用 `tldextract` 求 registrable domain。

这里有两个重要生产经验：

1. `max_depth/max_pages` 只是预算，绝不能据此宣称历史文章已经完整抓完。
2. registrable domain 只适合作为 scope 构造的辅助信息，`blog.example.com`、`docs.example.com`、`www.example.com` 是否同属正文边界仍应由 Site Profile 明确配置。

更值得注意的是，源码注释写着“stream must be False for BFS deep crawl”，但实际创建的是 DFS strategy；同时注释说明某 Crawl4AI deep crawl 场景在 `stream=True` 下会得到 0 条结果，因此代码强制 `config.stream=False`。

这类“注释、预期 strategy、实际 strategy、库版本行为”不一致，说明生产平台不能只存声明式配置，而必须记录运行时事实：

```text
engine_version
runtime_strategy_class
stream_mode
deep_crawl_limits
return_shape
known_quirk_ids
```

并通过 Capability Test 验证。

## 4. CrawlResult 归一化原理

Crawl4AI 不同版本/模式可能返回：

- 单个 `CrawlResult`
- `CrawlResultContainer`
- list
- async generator
- list 中再嵌套 container

项目在 `__init__.py` 的 `_extract_first_result` 和 `site.py` 的 `_iterate_results` 中显式兼容这些 shape。

这是一条很重要的架构边界：业务层不应该直接依赖第三方库的返回类型。更合理的生产实现应先统一成内部 Contract：

```text
CrawlPageOutcome {
  request_url
  final_url
  status
  status_code
  headers
  html_source
  html_rendered
  cleaned_html
  markdown_candidate
  links
  metadata
  error_class
  error_message
  engine_release_id
  runtime_config_digest
}
```

然后 Extraction、Quality、Identity 等下游只消费该内部类型。若第三方库升级导致 shape 变化，应返回 `ENGINE_RESULT_SHAPE_ERROR`，不能静默变成空正文。

## 5. Markdown 生成与内容过滤

### 5.1 PruningContentFilter

`build_markdown_generator` 使用：

```text
PruningContentFilter
threshold=0.45
threshold_type=dynamic
min_word_threshold=1
```

然后通过 `DefaultMarkdownGenerator` 生成 Markdown，关闭 citations、忽略图片、跳过 internal links，并把 `body_width` 设为 0。

这个做法适合面向 LLM 的“正文优先”输出，但不适合作为唯一知识真相，因为：

- 图片被忽略会丢失资产关系；
- internal link 被跳过可能损失知识图谱边；
- PruningContentFilter 是启发式过滤；
- Markdown 是第三方引擎的 projection，不是结构化 Canonical IR。

因此知识库方案应保存原始 HTML/rendered DOM 与 Extraction Candidate，再生成自己的 Canonical IR 和确定性 Markdown。

### 5.2 全局抓取 recipe 的风险

`build_markdown_run_config` 默认设置了大量主内容选择器和排除选择器，并执行：

```javascript
window.location.reload();
setTimeout(() => window.scrollTo(0, document.body.scrollHeight), 500);
```

同时等待：

```text
document.querySelector('main')
&& main.innerText.trim().length > 50
```

这种配置能提高部分文档站的命中率，但作为 1000 站点统一默认存在明显问题：

1. 页面没有 `<main>` 时可能白等。
2. 强制 reload 增加一次额外请求，扩大带宽与源站压力。
3. 滚到底可能触发无限加载、广告或无关请求。
4. 宽泛排除如 `[class*='nav']` 可能误删正文中包含 `nav` 字样的 class。
5. SPA/静态站/CMS 的正确等待条件不同。

生产系统应把 `wait_until / reload / scroll / wait_for / target_elements / excluded_selector / js_code` 下沉为 Site Profile/Browser Recipe，并以 Golden Corpus 回归。

## 6. Builder 与 Markdown 选择逻辑

`build_document_from_result` 的核心顺序是：

1. 先读取 result metadata、request/final URL、headers。
2. `result.success=False` 时构造失败文档。
3. 成功时优先选 `fit_markdown`。
4. `fit_markdown` 为空则用 citations markdown。
5. 再退化到 raw markdown。
6. 如果 Crawl4AI 没生成 Markdown，则从 `html` 或 `cleaned_html` 重新调用 generator。
7. 做 exact dedup。
8. 解析 references。
9. 返回 `CrawledDocument`。

优点是对 Crawl4AI 返回不完整有 fallback，缺点是 `fit_markdown` 经过过滤，若直接作为最终正文可能发生不可逆内容损失。生产方案应同时保存：

```text
raw engine candidate
fit candidate
cleaned candidate
chosen extraction candidate
quality decision
```

而不是覆盖成唯一 Markdown。

## 7. Exact Markdown 去重

`markdown_dedup.py` 的算法：

1. 统一 CRLF/CR 为 LF。
2. 按一个或多个空行分块。
3. 如果块内包含 Markdown heading，再按 heading 起点继续拆分。
4. 每块右侧去空白后计算 SHA-256。
5. 同一 fingerprint 只保留第一次出现。
6. 输出 `dedup_sections_total / removed / chars_removed / applied`。

Builder 还设置 guardrail：总块数至少 4、删除至少 2 个且删除率 >= 45% 时打 `high-removal-rate` warning。

这个实现非常适合说明“去重证据”和“知识真相”的差别。Guardrail 只发告警，但重复段已经从最终 `markdown` 中删除；如果重复内容本来是多个独立代码示例、免责声明中的关键差异、API 参数说明等，就可能不可恢复。

更稳妥的知识库实现：

```text
Canonical IR 原始 blocks 保留
 -> 计算 block hash
 -> Repetition Evidence
 -> 可选 dedup projection
 -> removed block ids + reason + rate
 -> 高删除率时 projection 失败或进入 review
```

去重版 Markdown 可以给 RAG/LLM 节省 token，但不能覆盖 Canonical IR。

## 8. SearXNG 搜索实现

### 8.1 请求模型

`mcp_server.py` 从环境读取：

```text
SEARXNG_URL
SEARXNG_USERNAME
SEARXNG_PASSWORD
SEARCH_RESULT_FIELDS
```

`_get_searxng_client` 创建 `httpx.AsyncClient(base_url=..., BasicAuth, timeout=30)`，`search` 调 `/search` 并固定 `format=json`。

支持：

- query
- language
- time_range
- categories
- engines
- safesearch
- pageno
- max_results
- max_retries

`max_results` 被限制在 1..50。

### 8.2 重试

当前只对 `httpx.RequestError` 做指数退避，初始 0.5 秒；HTTPStatusError 进入统一错误处理，401 有专门提示。

生产 Search Gap Provider 还应补：

- 429/503 + `Retry-After`
- 5xx 可重试分类
- jitter
- per-query deadline
- provider circuit breaker
- query budget
- 原始 response snapshot

### 8.3 字段过滤的证据风险

`SEARCH_RESULT_FIELDS` 可以只保留 title/url/content/publishedDate 等字段，MCP 返回会更紧凑。

对 Agent UX 很有用，但对知识库缺口审计不应直接这么做，因为可能丢失：

- engine
- score
- category
- rank 相关 provenance

正确做法是先完整保存 SearXNG 原始 JSON，再为 UI/MCP 生成精简 Projection。

## 9. 为什么 SearXNG 适合 Gap Discovery 而不适合 Coverage Truth

搜索引擎结果有天然不完备性：

- 索引覆盖不可证明；
- 排名和召回受引擎策略影响；
- 历史旧页面可能被去索引；
- 同一 URL 可能多引擎重复；
- 搜索结果页数、max result 都是预算；
- 结果会随时间变化。

因此 SearXNG 最合理的位置是：

```text
官方 Provider Coverage
 -> 发现日期/数量/迁移缺口
 -> SEARCH_GAP Query Plan
 -> SearXNG 多引擎搜索
 -> Search Result Evidence
 -> URL normalize + scope gate
 -> 与官方 Candidate 集合做差
 -> Targeted Fetch / Reconcile
```

Search Gap Evidence 能证明“外部索引发现了一个站内 Provider 未发现的 URL”，不能证明“没有搜索结果就没有遗漏”。

## 10. MCP/HTTP 接口的可借鉴点

项目用 FastMCP 同时暴露 STDIO 和 HTTP，并明确区分：

- HTTP bind host
- allowed Host headers
- Browser Origin/CORS allowlist

这对 Web 管理端或 Agent 接口很重要。`0.0.0.0` 只是监听地址，不等于允许所有 Host/Origin。生产系统应保留 FastMCP 默认安全策略；外网暴露时配置明确 Host allowlist 与 CORS origin，`*` 只用于本地开发。

但 MCP 工具的 request/response 生命周期不适合承担长时间全站同步。正确定位是：

- `targeted_fetch`
- `diagnose_site`
- `search_gap`
- `get_run_status`
- `reprocess_snapshot`

真正的 FULL_BACKFILL/INCREMENTAL 由 API 创建持久 Run/Task 后异步执行。

## 11. 输出格式的生产问题

MCP Markdown 输出会给每篇文档加：

```text
# URL
_Crawled: 当前时间_
```

多篇结果再用 `---` 拼接。这个输出适合聊天或临时文件，但不能作为 Canonical Markdown，因为同一内容每次抓取时间不同，hash 会变化，也不便单篇版本管理。

`remove_links` 通过正则去掉 Markdown 链接和裸 URL，也只能作为面向 LLM 的额外 Export Projection；若用作知识真相，会破坏代码示例、配置、参考链接和 provenance。

## 12. 当前项目不具备但知识库必须补齐的能力

searxNcrawl 本身没有完整实现：

- PostgreSQL durable state；
- source/provider registry；
- 全量历史 Coverage Evidence；
- sitemap/feed/API checkpoint；
- 增量同步；
- ETag/Last-Modified/body hash；
- append-only document version；
- durable queue/lease/heartbeat/dead-letter；
- object store snapshot；
- Web 管理后台；
- per-site 长期 rate limit/fair scheduling；
- Current Projection/Search/Embedding lineage；
- 资产归档；
- drift/shadow replay；
- Search Gap 与官方 Provider 差集审计。

因此不能把它直接横向扩容成“1000 站知识库”，而应把其 crawler/search 能力包装进更大的 Control/Data Plane。

## 13. 对现有技术方案的具体优化

本次调研后，技术方案应增加或强化以下设计。

### 13.1 新增独立 Search Gap Plane

把 SearXNG 从泛化“搜索能力”提升为正式 `Search Provider Release + Search Gap Worker + Search Evidence`，保留原始响应和查询计划。

### 13.2 Search Evidence 保留 provenance

不能为了 MCP token 省字段就永久丢掉 engine/score/category/rank。精简字段只属于 UI/API Projection。

### 13.3 Query Plan 可重放

每次缺口搜索记录 query、language、time_range、categories、engines、page 范围、触发原因、provider release，便于以后解释“这个 URL 为什么被发现”。

### 13.4 Browser Runtime 复用

明确禁止生产批量链路每 URL 创建完整 `AsyncWebCrawler`。Browser Worker 使用长生命周期 runtime/context pool，per-site semaphore/token bucket 控制资源。

### 13.5 Browser Recipe 站点化

把项目默认的 reload/scroll/wait_for(main) 视为“一个可用 preset”，而不是全局默认；进入 Site Profile Release 和 Golden Test。

### 13.6 Quirk Registry + Runtime Attestation

把 DFS/BFS 注释不一致、stream=False workaround、第三方返回 shape 变化等记录为 Engine Quirk；运行时保存实际 strategy class、stream mode 和版本。

### 13.7 Repetition Evidence 替代破坏性 dedup

吸收 exact SHA-256 block fingerprint 的简单有效部分，但先保存 Evidence，再生成可选去重 Projection；不在 Canonical IR 前删除。

### 13.8 Canonical Markdown 确定性

禁止把 `_Crawled: now_`、批量拼接头部或 remove-links 结果作为 Canonical Markdown。

## 14. 推荐集成方式

不 fork searxNcrawl 作为主平台，而是抽象两个 Adapter：

```text
SearxngSearchAdapter
  search(query_plan) -> SearchResultEnvelope

Crawl4AIEngineAdapter
  fetch(fetch_spec) -> CrawlPageOutcome
```

`SearchResultEnvelope` 必须含 provider release、query plan、raw response snapshot key 和 result provenance。

`CrawlPageOutcome` 必须把 Crawl4AI 的多种返回 shape 归一化，并把 runtime strategy、stream、timeout、browser recipe digest 记录到 attestation。

这样以后更换 SearXNG、Crawl4AI 或 Playwright 都不会改变业务状态模型。

## 15. 结论

searxNcrawl 适合作为“搜索 + 浏览器抓取 + MCP”能力参考，不适合作为 1000 站长期知识库的控制面和状态层。它对方案最有价值的启示是：

1. SearXNG 应成为可审计的 Search Gap Provider，而不是 Coverage 真相。
2. 第三方 crawler 返回 shape 和运行 quirks 必须由 Engine Adapter 隔离。
3. timeout 必须包住整个 URL await，而不只依赖底层 page timeout。
4. Browser runtime 要在生产 Worker 中复用。
5. crawler 默认 selector/JS/wait recipe 必须站点化、版本化和回归测试。
6. exact dedup 可以作为 Repetition Evidence 算法，但不能直接破坏 Canonical IR。
7. MCP/CLI 输出适合调试与 Agent，不适合作为持久同步调度或 Canonical Markdown。
8. Search、Crawler 的“声明配置”都需要 Runtime Attestation 证明真实生效。
