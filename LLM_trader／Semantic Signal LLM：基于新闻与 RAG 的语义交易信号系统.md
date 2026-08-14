# LLM_trader／Semantic Signal LLM：基于新闻与 RAG 的语义交易信号系统

## 1. 调研对象

- 编号：57
- 项目：LLM_trader / Semantic Signal LLM
- 地址：https://github.com/qrak/LLM_trader
- 调研基线：`master` 分支，提交 `8a0e60b2bb2638dbbd4212ce617a34111e35211f`
- 重点源码：RSS 新闻采集、Crawl4AI 正文补全、异步并发、正文清洗与去重、缓存与内容升级、RAG 更新/检索、相关性评分、FastAPI Dashboard。

LLM_trader 本质上是交易系统，新闻/RAG 只是其上下文子系统，所以它不能直接承担“1000 个技术博客全量历史归档 + 增量同步 + Web 管理”。可迁移价值主要来自几个工程模式：低成本 Feed 优先、正文不足才升级抓取、Browser 复用、失败降级、缓存兜底、依赖注入、评分策略隔离，以及 Dashboard 控制面。与此同时，它的内存任务、批次超时语义、URL/标题去重、文件缓存和查询热路径刷新也清楚暴露了平台化后必须修正的问题。

## 2. 关键源码与调用链

重点阅读：

- `src/rag/news_ingestion/rss_primitives.py`
- `src/rag/news_ingestion/rss_provider.py`
- `src/rag/news_ingestion/crawl4ai_enricher.py`
- `src/rag/news_ingestion/schema_mapper.py`
- `src/rag/news_manager.py`
- `src/rag/news_repository.py`
- `src/rag/rag_engine.py`
- `src/rag/scoring_policy.py`
- `src/dashboard/server.py`

主要数据流：

```text
RSS Feed
  │
  ▼
rss_primitives.fetch_source()
  │ RSS 解析 / URL 归一化 / Feed 正文抽取
  ▼
RSSCrawl4AINewsProvider._fetch_raw_items()
  │ 多源并发
  ▼
URL 去重 → 标题二次去重 → 时间排序
  │
  ├─ body 已足够长 ───────────────────────────┐
  │                                           │
  └─ body 过短 → Crawl4AIEnricher             │
                   │                           │
                   ├─ Crawl4AI / Playwright   │
                   └─ aiohttp fallback        │
                                               ▼
                                  schema_mapper.to_article_schema()
                                               │
                                               ▼
                                         NewsManager
                                               │
                                         NewsRepository
                                               │
                                               ▼
                                           RagEngine
                                               │
                           Index / Context / ScoringPolicy
                                               │
                                               ▼
                                     Dashboard / LLM 消费
```

这个调用链的核心思想是“逐层提高成本”，但绝大多数状态都只存在于当前 Python 调用栈和本地文件中；博客知识库平台必须把每一层改造成持久、可恢复、可审计的阶段。

## 3. RSS：把结构化 Feed 当作低成本入口

`rss_primitives.py` 内置 Feed Source Registry，也允许配置覆盖 URL 或只启用部分来源。RSS 2.0 条目会读取：

- `title`
- `link`
- `guid`
- `description`
- `content:encoded`
- `dc:creator`
- `pubDate`
- `category`

正文候选优先级为：

```text
content:encoded > description > 空
```

并把来源写入 `body_source`。这一点很重要：Feed 全文、Feed 摘要、HTTP 页面、Browser DOM 都应当被视作不同 provenance 的 Content Candidate，不能直接混成一个最终字符串。

### 3.1 Feed HTML 抽取并不是一次正则替换

正文抽取采用逐级降级：

1. 优先 BeautifulSoup + lxml；
2. 删除 `script/style/noscript/svg/header/nav/footer/aside/form`；
3. 尝试 article-body/article-content/post-content/entry-content/main 等主体 selector；
4. 从标题、段落、列表、blockquote 组装正文；
5. BeautifulSoup 不可用时回退自定义 `HTMLParser`；
6. 最后才进行粗粒度 strip-html。

值得迁移的是“多个 extractor/策略逐级产生候选”，而不是把项目当前最终字符串作为平台模型。长期知识库应保存每个候选和质量结果，支持规则更新后的离线重选。

### 3.2 URL 归一化是必要但不充分的

项目会删除 fragment、常见 `utm_*`、`gclid`、`fbclid`、`mc_*` 等追踪参数并处理尾斜杠。这适合作为第一层语法归一化，但不能直接充当平台 canonical identity，因为还缺少站点级 AMP/print/mobile 规则、redirect、`rel=canonical`、镜像域名和内容证据。

### 3.3 Feed 不能证明历史完整

当前实现面向“最近新闻”，Feed 每次取有限条目。即使 Feed 抓取成功，也只能说明当前窗口的数据可见，不代表多年历史文章已被发现。因此博客平台必须明确分离：

```text
FeedItemObservation / Change Signal
≠
Coverage Exhaustion Evidence
```

Feed 可以显著降低增量成本，但全量历史仍要依赖 API、Sitemap、Archive、Category/Tag/Author、Docs Navigation 等 Provider。

## 4. 多源异步并发：小规模可用，平台级存在结果丢失语义

`RSSCrawl4AINewsProvider._fetch_raw_items()` 的核心形态是：

```python
await asyncio.wait_for(
    asyncio.gather(*(fetch_source(...) for source in sources)),
    timeout=stage_timeout,
)
```

每个 Feed 自己有 timeout，外层又有一个阶段总 timeout。单个 `fetch_source()` 会把错误包装为 `FetchResult`，这一点具备错误隔离意识。

问题在于：**外层总超时后函数直接返回空列表。** 即使此前已有多个 Feed 成功完成，其结果也会因为聚合阶段超时而整体丢失。

在 4 个新闻源中这属于可接受的 fail-soft；在 1000 个站点中会变成严重数据一致性问题。平台正确语义应是：

- 每个 Source/Provider Task 独立持久化；
- 完成一个立即落库，而不是等待大批次统一提交；
- Run deadline 到达只终止未完成任务；
- 已完成结果不可回滚为“空”；
- Run 允许 `PARTIAL_SUCCESS`；
- 后续只重试失败或过期任务。

因此不能让 `asyncio.gather()` 的调用边界决定业务事务边界。业务完成事实必须先于 Python 协程生命周期存在。

## 5. Crawl4AI Enrichment：按质量信号升级抓取成本

`Crawl4AIEnricher.enrich_items()` 只挑选：

```text
URL 通过基础安全检查
AND
当前 body_text 长度 < min_chars
```

默认 `min_chars` 约 400。也就是说 Feed 已经给出足够正文时，系统不会无条件启动浏览器。这是项目最值得迁移的工程思想之一。

平台化后应把单一字符阈值升级为 Quality Gate：

```text
Feed/API Content Candidate
        │
        ▼
Quality Gate
        ├─ Accept
        └─ Need more evidence
               │
               ▼
HTTP Snapshot + deterministic extractors
        │
        ▼
Quality Gate
        ├─ Accept
        └─ Escalate
               │
               ▼
Browser / Crawl4AI
        │
        ▼
Quality Gate / Site Recipe / Manual
```

质量至少综合正文长度、文本密度、标题相关度、导航/推荐/页脚比例、错误页 marker、代码/表格/图片完整度、发布时间/作者/canonical 合理性、Feed 与页面相似度、页面类型和历史 extractor 接受率。

## 6. Crawl4AI 的具体实现原理

### 6.1 Browser 实例复用

`_enrich_crawl4ai_batch()` 一次创建 `AsyncWebCrawler`，再批量处理 URL，而不是每篇文章启动一次 Chromium。这样减少 Playwright 进程启动和销毁成本。

项目限制 Browser worker 数量：

```text
worker_count = max(1, min(configured_concurrency, 6))
```

说明浏览器并发必须受严格资源约束。1000 站平台还应进一步限制：

- 全局 Browser capacity；
- 单 registrable domain 并发/QPS；
- Source 并发；
- 单 worker page/context 数；
- 内存和 CPU；
- browser-seconds budget；
- Browser 生命周期与最大任务数。

Browser 进程应定期 recycle，避免长期运行后的页面泄漏、cookie 污染和内存增长。

### 6.2 Crawl4AI 运行配置

项目使用的关键思路包括：

```text
BrowserConfig:
  headless = true

CrawlerRunConfig:
  page_timeout
  remove_overlay_elements = true
  remove_consent_popups = true
  semaphore_count = worker_count
  magic = true
  simulate_user = true
  override_navigator = true
  wait_until = load
  delay_before_return_html = 1s
```

Markdown 使用 `DefaultMarkdownGenerator` + `PruningContentFilter`，动态阈值约 0.48，并设置很低的最小词数用于新闻正文过滤。

这些值是项目经验参数，不应成为知识库全局常量。生产平台要将 Crawl4AI 版本、参数、等待策略和站点特殊 recipe 放入版本化 Route/Profile Release，通过 Adapter 隔离第三方 API 字段变化。

### 6.3 输出不是“crawler success 就接受”

`_extract_crawl4ai_body()` 会尝试：

1. `fit_markdown`；
2. `raw_markdown`；
3. `markdown_with_citations`；
4. `cleaned_html` / `html` 再抽取。

候选经过文本清洗和最小长度检查，还会拒绝常见错误正文，如 page not found、404、oops 等。

这体现出一个必须保留的边界：

```text
HTTP success
≠ Browser success
≠ Extraction success
≠ Quality Accepted
```

平台状态模型要把四者分开，否则 Web 运维无法判断失败发生在哪一层。

### 6.4 批次 deadline 与重复 fallback 风险

项目按 URL 数量和 worker 数估算 wave count，再推导 batch timeout。这种“波次估算”可用于软 deadline，但不能作为平台 correctness 机制。

如果 Crawl4AI 整批超时，当前实现可能把全部 target 再交给 aiohttp fallback。问题是浏览器批次里可能已经有部分 URL 请求成功，只是聚合结果未被消费，于是 fallback 会产生重复流量。

平台应把 fallback 显式建模成持久 `FetchRouteAttempt`：

```text
HTTP_STATIC
   ↓ quality fail
BROWSER_CRAWL4AI
   ↓ runtime fail
HTTP_ALT_EXTRACTOR
   ↓
SITE_RECIPE / MANUAL
```

每个 Attempt 独立记录请求次数、字节、Browser 秒数、Snapshot、错误和预算。只有明确未得到可用 Snapshot/候选的任务才进入下一 Route。

## 7. Windows 事件循环兼容：说明 Browser Runtime 不应泄漏到主控制面

项目专门检查 Windows 上当前是否是 `SelectorEventLoop`，因为 Playwright 需要启动子进程；必要时用 `asyncio.to_thread()` 创建独立线程，并在其中运行 `ProactorEventLoop`。

这是很实用的兼容代码，同时也说明 Browser runtime 的平台差异很容易污染业务事件循环。生产推荐：

- Browser Worker 独立进程/容器；
- Linux 作为标准生产运行环境；
- API/Scheduler 不直接 import 和管理 Playwright 生命周期；
- Crawl4AI/Playwright 只暴露统一 Adapter Contract。

这样第三方运行时约束不会扩散到控制面。

## 8. SSRF 与 Redirect 关联

### 8.1 当前 URL 安全检查不够完整

`_is_safe_external_url()` 能拒绝非 HTTP(S)、localhost 以及 host 字面量为 private/loopback/link-local IP 的 URL，体现了安全意识。

但 Web 管理端允许用户新增站点后，生产级 SSRF 防护必须进一步处理：

- DNS 解析后的 A/AAAA 地址；
- redirect 每一跳重新校验；
- DNS rebinding；
- IPv4-mapped IPv6；
- metadata service 地址；
- 内部 DNS 域名；
- 端口策略；
- Browser 子请求网络拦截。

因此 Feed、HTTP、Browser、Asset、Webhook 都应统一走一个 Egress Policy，不能每个模块各自写“小型 URL 安全函数”。

### 8.2 Redirect 后用 URL 反查请求存在脆弱性

Crawl4AI batch 先用原 URL 建映射，结果返回后再根据 `result.url`/redirected URL 找原 item。如果抓取库返回 final URL，而原始 URL 不相同，就可能匹配失败，随后又被错误送入 fallback。

平台必须使用：

```text
task_id / route_attempt_id / observation_id
```

作为关联主键；requested URL、redirect chain、final URL、canonical URL 都只是一次 Observation 的属性。这样站点迁移或 301 不会破坏任务身份。

## 9. Schema、正文清洗、稳定 ID 与身份模型

`schema_mapper.py` 使用：

```text
sha256(url)[:16]
```

生成确定性文章 ID。相比随机 UUID，这对下游稳定性有帮助，但“URL 稳定”仍不等于“逻辑文章身份稳定”。站点改 slug、换域名、迁移 CMS 后 URL 会变化。

平台应拆分：

```text
URL Candidate ID
Document ID
Document Version ID
Chunk ID
```

并支持多个 URL Alias 指向同一 Document。

项目还维护大量通用和 CoinDesk 专用尾部 marker、市场 ticker、菜单/导航裁剪正则。它说明纯通用 extractor 最终一定会遇到站点特殊规则，但生产系统不能把这些规则永久硬编码在 schema mapper 中，而应进入 `SiteProfileRelease`：有版本、可回滚、可统计命中率、可对旧 Snapshot 离线 replay。

### 9.1 Prompt-friendly 文本不能代替知识库 Markdown

Crawl4AI 清洗会删除图片、去掉 Markdown heading marker、把链接变纯文本，这对 LLM prompt 压缩是合理的，但会破坏长期知识资产的结构。

平台需要分离：

```text
Canonical IR            长期语义真相
Markdown Projection     稳定可读归档
Prompt Text Projection  面向 LLM 的压缩文本
```

Markdown 必须保留 heading、code fence、表格、图片、链接、blockquote、列表等必要结构。

## 10. NewsManager / NewsRepository：缓存降级与“正文升级”

`NewsRepository` 把文件持久化与 NewsManager 业务逻辑隔离，提供 recent load/save、按年龄过滤和 fallback cache。这个 Repository 边界值得迁移：上层不应直接操作存储细节。

但当前后端是本地文件，只适合单机新闻缓存，不能作为 1000 站平台的事实层。平台应替换为 PostgreSQL + 对象存储，并保持 Repository/Service Contract。

### 10.1 Provider 失败时使用旧缓存

`NewsManager.fetch_fresh_news()` 在 provider 返回空或异常时，会使用较旧缓存作为 fallback。这是 RAG 可用性的正确思路：外部数据暂时失败，不应该把系统现有有效知识清空。

知识库应把这个原则升级为：

- 抓取失败不会删除 Accepted Document Version；
- Search/Vector 继续使用最近成功 Projection；
- UI 标记 freshness / source lag；
- 只有明确删除、撤回或 tombstone 策略才移除内容。

即“**stale-but-valid 优于 transient-empty**”。

### 10.2 短正文后续升级

`update_news_database()` 对同 URL 不是一律跳过：如果旧正文短于阈值而新正文更长，会原地更新旧记录。这非常符合真实抓取场景：第一次只有 RSS summary，之后页面抓取拿到全文。

长期知识库不能原地覆盖，而应：

1. 创建新的 Extraction Candidate；
2. 重新经过 Quality；
3. 生成 Canonical IR hash；
4. 内容更完整时创建 `EXTRACTION_UPGRADE` Document Version；
5. 原 Snapshot/Candidate/Version 全部保留；
6. 只重建变化的 Markdown/Chunk/Embedding。

## 11. RagEngine：依赖注入、并发刷新与查询热路径问题

### 11.1 依赖注入和状态隔离

`RagEngine` 的 NewsManager、IndexManager、ContextBuilder、MarketDataManager 等通过构造函数注入，而不是在引擎内部硬创建。这使组件容易测试和替换，平台的 Fetcher/Extractor/Retriever/Vector Store 也应沿用相同思路。

### 11.2 更新锁和重试抑制

引擎使用 `asyncio.Lock` 防止同进程重复刷新，并记录 `_last_update_attempt`，设置最小重试间隔，避免主循环刷新失败后查询路径立刻再次触发外部更新。

这是防止“失败后热循环”的实用手段。不过 1000 站平台不能只靠进程内 Lock；应把刷新权、lease、idempotency key 和 retry/backoff 持久化，使多实例和进程重启下依然成立。

### 11.3 独立 I/O 并发且隔离异常

`refresh_market_data()` 使用 `asyncio.gather(..., return_exceptions=True)` 并行更新相互独立的数据源，并在新闻变化后重建索引。这比“一项失败导致全批失败”更合理，值得迁移为独立 Stage/Task。

平台进一步要求每个任务完成即持久提交，不能把 Python gather 结果本身当事务。

### 11.4 查询热路径可能触发更新，不适合 1000 站系统

`retrieve_context()` 发现数据较旧时，可能调用 `update_if_needed()`，虽然有 5 分钟重试抑制，但这仍意味着一次用户查询可能间接等待外部更新逻辑。

交易新闻规模小、时效强，这种设计可接受；博客知识库中，查询请求绝不能同步触发：

- 站点抓取；
- Browser；
- 大规模重建索引；
- 长时间第三方 API 调用。

正确模式：

```text
Query
  ↓
读取最近成功 Search/Vector Projection
  ↓
返回结果 + projection_age/source_lag

若 stale：
  └─ 仅创建/合并异步 refresh task
```

这保证检索 SLA 不被源站状态拖垮。

## 12. ScoringPolicy：把检索规则从编排层拆出来

`ArticleScoringPolicy` 是本项目对知识库 RAG 很有价值的设计：相关性算法被独立为 policy，而不是散在 ContextBuilder 中。

当前字段权重大致为：

```text
title      10
body        3
categories  5
tags        4
```

评分还组合：

- keyword frequency；
- category relevance；
- domain-specific coin relevance；
- important category；
- recency；
- body density；
- 多关键词共现；
- 标题/正文中的实体相关度。

`RagEngine` 还会把完整正文候选排在短 summary 前面，再由 ContextBuilder 在 token budget 下选文章。

### 12.1 可迁移的不是具体分数，而是“策略版本化”

交易新闻的 recency 设计会让 24 小时之外内容快速失去价值，这完全不适合技术博客：一篇 5 年前的数据库原理文章可能比昨天的短新闻更权威。

因此知识库应新增 `retrieval_policy_release`：

```text
id
version
candidate_sources
lexical_weight
vector_weight
field_weights
freshness_policy
source_priority_policy
quality_policy
full_body_boost
reranker_config
context_budget
created_at
```

检索链推荐：

```text
Metadata Filter
  ↓
BM25 + Vector Recall
  ↓
RRF / Weighted Merge
  ↓
Policy Scoring
  ↓
Optional Reranker
  ↓
Context Budget Selection
```

这样字段权重、新鲜度、质量、来源优先级都可重放、A/B 和回滚，而不是硬编码进业务代码。

### 12.2 Content Quality 与 Retrieval Quality 分离

正文长短可以作为检索排序信号，但不能替代 ingestion Quality。一个页面是否应被接受进知识库，与一次用户查询是否优先召回它，是两套不同决策：

- Ingestion Quality：判断内容是不是合法、完整、可用文章；
- Retrieval Policy：判断该 Accepted Version 对当前 query 是否最相关。

二者必须独立版本化，避免“某次搜索偏好”反向决定历史文章是否进入知识库。

### 12.3 稳定 Citation

LLM_trader 主要记录当前 article URL；长期知识库需要把引用固定到：

```text
document_id
document_version_id
chunk_id
source_url
```

否则原页面修改后，历史回答无法复现。

## 13. Dashboard：FastAPI 控制面的可迁移设计

`src/dashboard/server.py` 使用 FastAPI 组织 Dashboard，并具备：

- 多 Router 分层；
- `DashboardState`；
- 日志流与 Console Buffer；
- WebSocket Router；
- Admin Router；
- 可写配置封装；
- 可选认证初始化；
- CORS；
- CSP/HSTS/Referrer-Policy/Permissions-Policy 等安全响应头；
- API rate limit；
- ETag 与 Cache-Control 策略；
- 生命周期启动/关闭处理。

这说明项目不仅做“抓取函数”，还重视运营可见性和控制面，这与博客知识库的 Web 管理需求高度相关。

### 13.1 可直接借鉴的 Web 思路

博客平台 Web 应实时展示：

- Source 状态与 lag；
- Coverage Provider 进度；
- Run/Task stage；
- queue depth / worker health；
- route attempt；
- Browser 使用率；
- extractor/quality 结果；
- 实时日志和告警。

可通过 WebSocket/SSE 只推送状态变化，不让浏览器高频轮询整个任务表。

### 13.2 “可写全局配置”要升级成不可变 Release

LLM_trader 的 Dashboard 可以写应用配置，这对单应用很方便；但多站点抓取平台若直接在线修改全局 ini，会丢失“某个 Run 当时用了哪套规则”的可追溯性。

因此 Web 中所有影响抓取、抽取、质量、检索的配置都应创建 immutable Release，Run 固定引用 Release ID，支持 diff、回滚和审计。

## 14. 哪些设计可以迁移，哪些不能直接复制

### 可迁移

- Feed-first 的低成本结构化采集。
- Feed 正文足够时跳过 Browser 的渐进式 enrichment。
- Browser 实例复用和有限并发。
- Crawl4AI 失败后的多路降级思想。
- Repository/Service 分层。
- 外部数据失败时使用最近有效缓存。
- 短正文后来得到全文的 Content Upgrade 思想。
- 依赖注入。
- 独立 Scoring Policy。
- token budget 下的 context selection。
- FastAPI Dashboard、实时日志、WebSocket、安全响应头和 rate limit。

### 不能直接复制

- 仅靠最近 RSS 窗口作为数据发现。
- 内存 asyncio task 作为任务真相。
- stage timeout 后返回空集合。
- 整批 Browser 超时后重新抓所有 target。
- 字符数作为唯一正文质量标准。
- URL hash 直接当逻辑文章身份。
- normalized title 硬去重。
- hardcode 站点尾部清洗规则。
- 本地 JSON/文件缓存作为主存储。
- 查询路径同步触发外部刷新。
- 交易新闻 24 小时 recency 直接套到技术知识检索。

## 15. 对博客知识库方案的具体优化

### 15.1 Provider Contract

所有 Sitemap/Feed/API/Archive Provider 统一支持 capability probe、durable cursor、candidate + evidence、explicit exhaustion，不把某个库的返回对象渗透到业务层。

### 15.2 Progressive Enrichment

正式采用：

```text
Coverage/Metadata
→ Feed/API body
→ HTTP deterministic extraction
→ Browser/Crawl4AI
→ Site Recipe / Manual
```

每次升级都由 Quality Policy 驱动，而不是固定全站 Browser。

### 15.3 Partial-result-preserving Fanout

所有 Source/Provider 结果完成即提交；Run 可以 partial success；禁止批次总 timeout 抹掉已经成功的事实。

### 15.4 Durable Route Fallback

Route 失败与 fallback 都持久化为 `FetchRouteAttempt`，可从进程崩溃和重启中恢复，并避免不必要重复请求。

### 15.5 Immutable Content Upgrade

Feed summary → HTTP 正文 → Browser 正文的提升统一表示为 Candidate + Version，不原地覆盖历史。

### 15.6 Retrieval Policy Release

将字段权重、BM25/Vector 权重、freshness、source priority、quality/full-body boost、reranker、context budget 版本化。检索策略变化只重建/重评分投影，不修改 Accepted Content。

### 15.7 Retrieval Hot Path 解耦

查询只消费最近成功 Projection；源站/索引刷新交给 Scheduler。数据旧时返回 freshness，而不是在请求线程执行 Browser 或全量同步。

### 15.8 Stale-but-valid Cache

短暂抓取失败不清空搜索索引或知识版本。显式记录 source lag、projection age 和 error，保障知识库连续可用。

### 15.9 Web 实时运维与安全控制

Dashboard 增加 WebSocket/SSE 的 Run/Task/worker 状态，配合 RBAC、Audit、CSP、rate limit。所有可编辑规则都发布为 Release，而不是直接覆盖运行中的配置。

## 16. 最终结论

LLM_trader 的新闻/RAG 子系统不适合直接当作 1000 站技术博客爬虫平台，但它提供了几组非常有价值的工程原型：

1. Feed 中已有结构化正文时先使用低成本候选；
2. 只有正文不足才升级 Crawl4AI/Browser；
3. Browser 应复用而不是每 URL 启停；
4. 外部 Provider 失败时保留最近有效数据；
5. 短正文后续可升级成更完整内容；
6. RAG 评分策略应与编排分离；
7. Dashboard 应同时承担监控、控制和安全边界。

真正平台化时，需要把这些“进程内模式”提升为“持久化协议”：Coverage、Task/Lease、Route Attempt、Snapshot、Candidate、Document Version、Retrieval Policy Release 和 Projection freshness 全部成为可查询、可恢复、可回放的状态。这样才能同时满足全量历史抓取、长期增量同步、可扩展站点接入、Web 管理和稳定 RAG 的要求。
