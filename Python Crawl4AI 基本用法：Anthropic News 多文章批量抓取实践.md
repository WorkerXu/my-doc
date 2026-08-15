# Python Crawl4AI 基本用法：Anthropic News 多文章批量抓取实践

## 1. 调研对象

- 编号：58
- 原文：https://juejin.cn/post/7556863932343287844
- 原文标题：`[1339]python crawl4ai基本用法`
- 原文发布时间：2025-10-06
- 主题：Crawl4AI 基本抓取、Markdown 生成、Pruning/BM25 内容过滤，以及 `arun_many()` 多 URL 并发抓取
- 相关官方资料：
  - https://docs.crawl4ai.com/advanced/multi-url-crawling/
  - https://docs.crawl4ai.com/api/arun_many/
  - https://docs.crawl4ai.com/core/fit-markdown/
  - https://docs.crawl4ai.com/core/cache-modes/
  - https://docs.crawl4ai.com/api/parameters/
  - https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/content_filter_strategy.py

这篇文章属于教程型实践，展示了“多个文章 URL -> Crawl4AI 并发抓取 -> Markdown 过滤 -> 本地文件”的最短链路。对 1000 个技术博客知识库项目而言，值得吸收的不是直接把示例放大，而是把 Crawl4AI 放到正确的工程边界：`arun_many()` 是 Worker 内批执行器，Pruning/BM25 是有损内容投影，Crawl4AI cache 是执行优化而不是增量同步事实源。

同时必须纠正原文一个重要概念错误：原文把 BM25 描述成“向量检索，根据内容生成向量再检索相似内容”。Crawl4AI 当前 `BM25ContentFilter` 的实现并不是向量检索，而是基于 `rank_bm25.BM25Okapi` 的词法相关性评分；这直接影响中文处理、版本设计和检索架构判断。

## 2. 原文实现链路

原文主要包含：

1. 安装 Crawl4AI，并使用异步 crawler 抓取页面；
2. 使用 `DefaultMarkdownGenerator` 生成 Markdown；
3. 使用 `PruningContentFilter` 做无查询条件的启发式裁剪；
4. 使用 `BM25ContentFilter(user_query=...)` 做查询相关过滤；
5. 使用 `arun_many()` 对多个 Anthropic News URL 批量抓取；
6. 将结果直接写入本地 Markdown 文件；
7. 示例中使用 `CacheMode.DISABLED` 获取最新内容。

教程链路可抽象为：

```text
URL List
  -> BrowserConfig / CrawlerRunConfig
  -> AsyncWebCrawler.arun_many()
  -> CrawlResult per URL
  -> raw_markdown / fit_markdown
  -> local markdown file
```

生产知识库必须拆为：

```text
Durable Task
  -> Fetch Route
  -> Worker-local Batch Dispatcher
  -> Fetch Attempt
  -> Immutable Snapshot
  -> Extraction Candidate
  -> Canonical IR
  -> Document / Document Version
  -> Canonical Markdown Projection
  -> Filtered Markdown Projection
  -> Chunk / Fulltext / Embedding / Index
```

网络事实、正文事实和派生内容必须分层，否则过滤器升级、抽取器升级、源站变化和检索重建会互相污染。

## 3. `arun_many()` 的执行模型

### 3.1 它是 Worker 内并发入口，不是平台调度器

当前 Crawl4AI 官方多 URL 文档将 `arun_many()` 作为多 URL 批执行入口，允许传入：

- URL 列表；
- 单个或多个 `CrawlerRunConfig`；
- Dispatcher；
- `stream=True` 流式结果；
- URL matcher 驱动的 per-URL config。

Dispatcher 负责当前进程/Worker 内的资源约束，典型包括：

```text
MemoryAdaptiveDispatcher
SemaphoreDispatcher
RateLimiter
CrawlerMonitor
```

它无法提供平台级的 durable task、跨节点 lease、站点级持久限流、优先级、全局预算、失败恢复和审计。因此 1000 站点必须采用两层调度：

```text
平台 Scheduler
  global token bucket
  + registrable-domain token bucket
  + source token bucket
  + route budget
  + priority
  + durable lease / retry / checkpoint
        ↓
Worker
  arun_many()
  + MemoryAdaptiveDispatcher / SemaphoreDispatcher
  + RateLimiter
  + stream=True
```

平台 Scheduler 决定“谁、何时、以什么路由抓”；Crawl4AI Dispatcher 决定“当前 Worker 如何在内存和并发预算内执行”。

### 3.2 `stream=True` 应与逐 URL 提交绑定

正确执行模式：

```text
worker lease N tasks
  -> arun_many(urls, stream=True)
  -> URL A 返回
     -> persist FetchAttempt
     -> persist Snapshot
     -> commit
     -> ack A
  -> URL C 返回
     -> persist / ack C
  -> URL B 失败
     -> retry B only
```

不能在一个数据库事务中包住整个 batch。否则慢 URL、失败 URL 或 Worker 取消会让已成功结果失去 partial success 价值。

推荐任务状态和 fencing：

```text
PENDING -> LEASED -> RUNNING -> SUCCEEDED
                      |            
                      -> RETRY_WAIT
                      -> FAILED_TERMINAL
                      -> CANCELLED
```

持久化时必须验证 lease token / fencing token，避免旧 Worker 超时后继续写入覆盖新 Worker 结果。

### 3.3 批大小必须自适应

批次大小至少受以下因素共同约束：

- Worker 可用内存；
- Chromium page/context 数；
- 单页 DOM / HTML 大小；
- 页面响应时间；
- 单域名并发上限；
- Object Storage 写入速度；
- Extraction backlog；
- Browser CPU；
- 结果消费速度。

因此不能固定 `batch_size=1000`。应采用“平台 lease 小批次 + Worker 自适应 Dispatcher + 单结果流式提交”。

## 4. Dispatcher 与 RateLimiter 的边界

### 4.1 MemoryAdaptiveDispatcher

其核心价值是根据 Worker 内存压力动态控制并发，避免 Browser page 大量堆积导致 OOM。它适合动态页面和 DOM 较大的站点，但依然只知道当前 Worker 的内存，不知道整个集群或某域名在其他 Worker 上的并发。

### 4.2 SemaphoreDispatcher

适用于明确固定并发上限、页面成本较稳定的场景。它简单可预测，但不能像 MemoryAdaptiveDispatcher 一样根据内存变化自调。

### 4.3 Crawl4AI RateLimiter

官方多 URL 文档中的 RateLimiter 可按响应码进行退避，并处理类似 429、503 的速率限制响应；它适合作为 Worker 内第二道保护。

平台仍必须单独维护：

```text
global limiter
registrable-domain limiter
source limiter
route limiter
provider limiter
```

原因是 Worker-local limiter 不能跨节点共享状态，也不能作为 durable 限流真相。`Retry-After` 应被解析并反馈给平台下一次可运行时间，而不是仅在当前进程 sleep。

## 5. `arun_many()` 取消、泄漏与 Runtime 固定

Crawl4AI 运行时版本必须被视为行为的一部分，而不是普通依赖。当前官方项目 v0.9.x 的发布说明中包含过与 `MemoryAdaptiveDispatcher`、流式 crawl 提前关闭后 task/page 清理相关的修复。这说明：

1. `stream=True` 的消费者如果提前退出，必须有明确取消与清理语义；
2. Browser page/context 泄漏会直接放大为长期 Worker 内存泄漏；
3. 不能只测试“完整消费全部结果”的 happy path。

Runtime Artifact Release 至少固定：

```text
crawl4ai_version
playwright_version
browser_revision
python_version
system_dependency_manifest
runtime_image_digest
```

CI Contract Test 必须包含：

```text
正常 arun_many 完整消费
stream=True 提前 break
任务 timeout / cancellation
单 URL exception
Worker SIGTERM graceful drain
Browser page/context 数归零
lease 未提交任务重新可见
```

升级 Crawl4AI 前先用固定 Golden URL 和提前取消测试验证，再切换 Runtime Release。

## 6. per-URL `CrawlerRunConfig` 与匹配规则

Crawl4AI 支持为不同 URL 使用不同配置，并通过 `url_matcher` 匹配。这很适合映射平台的 Fetch Route：

```text
PDF URL      -> PDF config
Article URL  -> article config
Dynamic Hub  -> interaction config
JSON/API     -> structured config
Fallback     -> default config
```

但这里有一个平台级风险：多个 matcher 可能重叠，顺序变化会导致同一 URL 命中不同配置。配置不能只作为 Python list 隐藏在 Worker 中。

建议新增可审计的 Route Match Contract：

```text
fetch_route_release
  ordered_rules[]
    rule_id
    matcher
    expected_url_type
    crawler_run_config_ref
    priority
  default_rule
```

发布前执行 fixture：

```text
URL -> expected_rule_id -> actual_rule_id
```

要求：

- 所有已知 URL 模板有确定命中；
- 重叠 matcher 有显式优先级；
- default rule 永远位于兜底位置；
- 新 Profile/Runtime Release 不能让既有 fixture 静默改路由。

## 7. `raw_markdown`、`fit_markdown` 与事实分层

官方文档区分：

- `raw_markdown`：由清洗后 HTML 转换得到的较完整 Markdown；
- `fit_markdown`：启用 Content Filter 后产生的裁剪 Markdown；
- `fit_html`：过滤后 HTML。

生产系统应定义：

```text
Snapshot                 = 不可变网络事实
Canonical IR             = 正文事实
CLEAN_MARKDOWN            = IR 确定性输出
CRAWL4AI_RAW_MARKDOWN     = 抽取候选/对照产物
FILTERED_MARKDOWN         = 有损可重建投影
BM25_QUERY_VIEW           = query-dependent 临时/缓存投影
```

`fit_markdown` 不是更权威的正文。Pruning、BM25、LLM Filter 都不能覆盖 Snapshot、Canonical IR 或 canonical Markdown。

## 8. `PruningContentFilter` 的当前实现原理

### 8.1 不是 LLM，而是 DOM 启发式评分

当前官方源码中的 Pruning 评分配置包含：

```text
text_density     weight 0.4
link_density     weight 0.2
tag_weight       weight 0.2
class_id_weight  weight 0.1
text_length      weight 0.1
```

它还结合：

- `min_word_threshold`；
- fixed / dynamic threshold；
- DOM tag importance；
- class/id 的正负向信号；
- 链接文本占比；
- 文本长度。

可以抽象为：

```text
score(node) =
  w1 * text_density
  + w2 * (1 - link_density)
  + w3 * tag_importance
  + w4 * class_id_signal
  + w5 * text_length_signal
```

低于阈值的 DOM node 被裁剪。其优势是速度快、无 LLM 成本、无需 query；风险则来自“技术内容并不总是长文本高密度”。

### 8.2 `preserve_tags` / `preserve_classes` 是重要保护能力

当前官方实现支持：

```text
preserve_tags
preserve_classes
```

命中白名单的节点可以绕过普通 pruning score 被保留。对技术知识库非常有价值，例如：

```text
preserve_tags:
  pre
  code
  table
  th
  time

preserve_classes:
  api-parameters
  signature
  code-block
  changelog
  references
```

但不能全局把 `div`、`span` 等宽泛标签加入 preserve，否则会重新带回导航、广告和推荐内容。正确做法是“全局少量语义标签 + Site Profile 精确 class 白名单”。

### 8.3 Pruning Release 需要包含保护策略

建议：

```text
content_filter_release
  strategy = PRUNING
  threshold
  threshold_type
  min_word_threshold
  preserve_tags[]
  preserve_classes[]
  preservation_policy_version
```

这样过滤结果才可复现。

## 9. `BM25ContentFilter` 的真实实现

### 9.1 原文“BM25 是向量检索”是错误的

当前官方源码直接使用：

```text
rank_bm25.BM25Okapi
```

处理链路大致是：

```text
HTML
 -> BeautifulSoup
 -> 提取 body 文本块
 -> 获取 query
    user_query 优先
    否则 title/h1/meta/首个显著段落
 -> tokenization
 -> optional Snowball stemming
 -> clean_tokens
 -> BM25Okapi.get_scores(query)
 -> tag priority weight
 -> threshold filter
 -> 返回对应 HTML chunks
```

这是词法排序，不产生 embedding，也不进行 cosine similarity。

### 9.2 tag priority 会修改纯 BM25 分数

源码对部分标签施加权重，例如标题、`strong`、`code`、`pre`、`th` 等，因此最终筛选不是“纯粹的 BM25 文本分数”，而是：

```text
adjusted_score = bm25_score * tag_weight
```

这让技术结构更容易保留，但也意味着行为与 Crawl4AI Runtime 版本绑定，不能只记录 `bm25_threshold`。

### 9.3 中文效果差的根因是词法处理，不是 BM25 理论失效

当前实现默认：

```text
language = english
use_stemming = true
```

并以类似 `lower().split()` 的空白切分得到 token，再执行 Snowball stemming/cleaning。英文天然适配，而中文句子通常没有空格，容易形成超长 token，导致 query 与正文词项匹配非常差。

因此生产平台必须把以下参数纳入 Filter Release：

```text
use_stemming
stemmer_language
tokenization_mode
language_scope
```

更重要的是，不要让 Crawl4AI 的页面级 BM25 Filter 承担生产全文检索。生产中文检索应该使用 OpenSearch/专用引擎的中文 Analyzer、ICU/Jieba 类 tokenizer，并建立独立 `analyzer_release`。

## 10. Content Filter Release 的完整建议

```text
content_filter_release
  id
  version
  strategy                  # NONE / PRUNING / BM25 / LLM / CUSTOM
  threshold
  threshold_type
  min_word_threshold
  preserve_tags[]
  preserve_classes[]
  preservation_policy_version
  query_mode                # NONE / FIXED / RUNTIME_QUERY
  fixed_query
  use_stemming
  stemmer_language
  tokenization_mode
  language_scope
  input_projection_type
  output_projection_type
  runtime_artifact_release_id
  benchmark_result_id
  created_at
```

对于 `RUNTIME_QUERY`：

```text
filter_idempotency_key =
  input_projection_hash
  + content_filter_release_id
  + runtime_query_hash
```

Query-dependent 视图必须设置 TTL 或缓存策略，不能无限保存所有查询结果。

## 11. Filter 质量门禁

过滤成功不能以“文件写出来了”为标准。至少记录：

```text
content_retention_ratio
heading_retention_ratio
code_block_retention_ratio
table_retention_ratio
list_retention_ratio
link_retention_ratio
reference_section_retained
preserved_node_count
preserve_rule_hit_count
language_consistency
truncation_signal
```

例如：

```text
content_retention_ratio = filtered_text_length / canonical_text_length
```

该指标不是越高越好，而是与同 Source/Template 的历史基线比较。若正常区间长期为 0.65~0.80，新 Filter Release 突然降到 0.12，应 quarantine，而不是自动发布。

技术博客要单独保护：

- fenced code；
- API signature；
- 参数表；
- shell command；
- reference section；
- heading path；
- changelog；
- footnote/link target。

## 12. CacheMode 与增量同步的边界

原文示例使用 `CacheMode.DISABLED` 获取最新内容，适合作为调试手段，但不能成为长期增量同步策略。

生产增量链路应优先：

```text
Feed/API cursor
  -> Sitemap lastmod
  -> Conditional GET (ETag / Last-Modified)
  -> Metadata Probe
  -> Full HTTP fetch
  -> Browser fallback
```

304 时：

- 记录 Observation / Fetch Attempt；
- 不创建正文 Snapshot 副本；
- 不重复 Extraction；
- 不创建 Document Version；
- 不重复 Markdown/Filter/Chunk/Embedding。

Crawl4AI CacheMode 只决定当前执行如何使用 Crawl4AI cache，它不是源站新旧判断的系统真相，也不能替代 ETag、Last-Modified、provider cursor 和 reconcile。

## 13. HTTP First 与 Browser Worker

对于普通技术博客正文：

```text
HTTPX/aiohttp
 -> immutable HTML snapshot
 -> deterministic extractors
```

以下情况再升级 Browser：

- 正文依赖 JS 渲染；
- Archive/Load More 只能交互发现 URL；
- HTTP 抽取质量低于阈值；
- 明确需要版本化 Interaction Recipe。

Browser 应独立 Worker pool。`arun_many()`、MemoryAdaptiveDispatcher 等正适合这一层，而不是让 1000 个站点所有静态文章默认走 Chromium。

## 14. `prefetch` 的正确使用位置

当前 Crawl4AI 版本提供更轻量的预取/发现能力，可在只需要 URL、链接、轻量页面信号时避免完整 Markdown/抽取链路。平台可以将它作为 Discovery/PROBE 的执行优化：

```text
Hub / Archive
 -> lightweight prefetch
 -> collect links / metadata
 -> classify URL
 -> only article candidates enter Fetch
```

但 Prefetch 结果仍是发现证据，不是正文 Snapshot。只有进入 Fetch Route 并按平台规则持久化的响应才能成为文章事实。

## 15. 对主技术方案的落地优化

### 15.1 Content Filter Release 增加保护与词法参数

必须增加：

```text
preserve_tags[]
preserve_classes[]
preservation_policy_version
use_stemming
stemmer_language
tokenization_mode
```

避免 Pruning 误删技术结构，也让 BM25 Filter 的语言行为可复现。

### 15.2 Runtime Contract Test 增加流式提前取消

除了静态/动态抓取 smoke test，还应测试：

```text
arun_many stream early-close
cancelled task cleanup
browser page/context cleanup
dispatcher permit recovery
unacked lease recovery
```

避免升级 Crawl4AI 后出现长期 page/task 泄漏。

### 15.3 URL Config 增加 Match Contract

所有 `url_matcher` 规则必须按 Release 保存顺序/优先级，并通过 URL fixture 测试，防止重叠 matcher 静默改变路由。

### 15.4 Worker RateLimiter 不代替平台限流

Worker 内 RateLimiter 继续保留，但域名/source/global limiter 必须由平台 durable scheduler 控制，并把 `Retry-After` 转换成下一次调度时间。

### 15.5 流式结果逐 URL 事务提交

每个 URL 独立保存 FetchAttempt/Snapshot 并 ack，禁止整个 `arun_many()` batch 共用一个长事务。

### 15.6 Web 增加 Dispatcher/Filter 诊断

增加：

```text
batch_size
active_pages
peak_memory
stream_completed_count
stream_cancelled_count
stream_cleanup_latency
rate_limit_wait_ms
retry_after_ms
config_rule_id
config_match_mismatch
preserve_rule_hit_count
```

这样能区分“站点慢”“Worker 资源紧张”“配置路由错”“过滤误删”“取消清理异常”。

## 16. 推荐执行流水线

```text
[Discovery / Probe]
  CMS / Sitemap / Feed / Archive / CC
  + Crawl4AI lightweight prefetch when useful
        ↓
[Scheduler]
  durable task
  + global/domain/source/route budget
  + Retry-After-aware next_run_at
        ↓
[Fetch Route]
  HTTP first
        ├─ high quality -> Snapshot
        └─ dynamic/low quality -> Browser/Crawl4AI Worker
                               ↓
                    lease small task batch
                               ↓
                    arun_many(stream=True)
                    + local Dispatcher
                    + local RateLimiter
                               ↓
                      result per URL
                               ↓
                 persist + commit + ack
                               ↓
[Extraction]
  Selector / JSON-LD / Trafilatura / Readability / Crawl4AI candidate
                               ↓
                        Canonical IR
                               ↓
                     Document Version
                               ↓
[Projection]
  CLEAN_MARKDOWN
  CRAWL4AI_RAW_MARKDOWN
  FILTERED_MARKDOWN(PRUNING)
  BM25_QUERY_VIEW
  CHUNK / EMBEDDING / INDEX
```

## 17. 验证用例

### 17.1 Batch / Dispatcher

- 100 URL 中 90 成功、10 失败，90 个事实必须保留；
- `stream=True` 消费一半后主动取消，不应持续残留 page/context；
- Worker 崩溃后未 ack task 能重新 lease；
- 大页面导致内存压力时并发应下降而不是 OOM；
- 429/503 能产生退避并反馈平台下一次调度。

### 17.2 Route Match

- article/PDF/API/dynamic hub 各有 fixture；
- 重叠 matcher 的命中顺序固定；
- Profile Release 改动后 fixture diff 可见；
- 未命中规则时进入明确 fallback，而不是无声失败。

### 17.3 Pruning

- code/pre/table 在配置保护后保持；
- reference section 不因链接密度高而误删；
- 不同 template 使用不同 threshold baseline；
- preserve 规则不能把导航/广告重新引入。

### 17.4 BM25

- 英文 query 可正常匹配相关块；
- 中文 query 在默认 whitespace tokenization 下必须暴露低召回风险；
- query 变化不创建 Document Version；
- BM25 Filter 与生产 OpenSearch BM25 指标/Release 完全分离。

### 17.5 Incremental

- 304 不创建正文版本和派生物；
- CacheMode 切换不会改变平台文档身份；
- Feed/Sitemap/Conditional GET/Reconcile 才驱动源站同步事实。

## 18. 结论

这篇教程最有价值的部分是展示了 Crawl4AI 的批量抓取和内容过滤入口，但放入 1000 站点知识库后必须重新划分边界：

1. `arun_many()` 是 Worker 内资源感知批执行器，不是 durable scheduler；
2. `stream=True` 必须与逐 URL 提交、partial success、lease fencing 和取消清理结合；
3. Worker-local Dispatcher/RateLimiter 是执行保护，平台仍需全局/domain/source 持久调度与退避；
4. per-URL `CrawlerRunConfig` 适合路由不同 URL 类型，但 matcher 顺序必须版本化并做 Contract Test；
5. Pruning 是启发式有损裁剪，当前实现支持 `preserve_tags` / `preserve_classes`，非常适合保护技术代码、表格和参数结构；
6. 原文对 BM25 的“向量检索”描述不正确，Crawl4AI 当前实现是 `BM25Okapi` 词法评分并叠加 tag weight；
7. 中文效果问题主要来自默认英文 stemming 与 whitespace tokenization，不意味着 BM25 理论本身不适用于中文；
8. Content Filter 必须独立 Release，并记录 preservation、stemming、tokenization、语言和 Runtime；
9. `raw_markdown`/`fit_markdown` 都不是系统事实源，Canonical IR 和 Snapshot 才能支撑长期重放；
10. `CacheMode.DISABLED/BYPASS` 不能替代 Feed/API cursor、Sitemap、Conditional GET 和 Reconcile 的生产增量同步。

采用这些边界后，Crawl4AI 可以作为可升级、可审计、可回滚的抓取执行组件，而不会绑架知识库的事实层、增量语义和长期可维护性。