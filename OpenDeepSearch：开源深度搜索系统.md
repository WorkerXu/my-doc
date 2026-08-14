# OpenDeepSearch：开源深度搜索系统——实现细节与技术原理分析

- 编号：34
- 项目地址：https://github.com/sentient-agi/OpenDeepSearch
- 调研对象：OpenDeepSearch 当前公开代码结构与实现方式
- 适用判断：适合作为“搜索缺口发现 + 候选网页深抓 + Chunk 重排 + Agent 上下文构建”的参考实现，不适合直接承担 1000 个站点的生产级全量历史抓取与持续增量同步。

## 1. 项目定位与核心流程

OpenDeepSearch 的核心不是通用分布式爬虫，而是面向 Agent 的深度搜索工具。其主链路可以抽象为：

```text
用户 Query
  -> Search Provider（Serper / SearXNG）
  -> SERP Sources
  -> SourceProcessor
  -> 可选网页抓取（Crawl4AI）
  -> 文本切块（Chunker）
  -> 语义重排（Jina / Infinity）
  -> Context Builder
  -> LiteLLM / Agent
```

这个架构最有价值的地方是把“搜索结果发现”“源页面获取”“语义排序”“上下文生成”拆成不同职责，而不是把搜索、爬取和问答混在一个函数里。对于博客知识库系统，这种分层适合作为 Search Gap Diagnostic Plane，但不能替代 Source Registry、持久 Frontier、Coverage Evidence、版本存储和增量同步主链路。

## 2. 代码模块与职责

### 2.1 `ods_tool.py`

`OpenDeepSearchTool` 对外包装为 Agent Tool，内部构造 `OpenDeepSearchAgent`。当前工具调用路径会以 `pro_mode=True` 请求深度搜索，并限制较少的来源数量。这种写法适合交互式 Agent：用户提一个问题，只需要少量高价值来源；但对于历史博客发现，Top-N 只能视为预算，不能视为完整性边界。

生产系统应把 `max_sources`、是否深抓、最大网页数、最大字节数、搜索页数和时间预算全部作为版本化 Search Query Plan/Run 配置保存，禁止硬编码为 Agent 交互参数。

### 2.2 `ods_agent.py`

`OpenDeepSearchAgent` 负责组装 Search API、SourceProcessor 和 LLM：

```text
get_sources(query)
 -> process_sources(...)
 -> build_context(...)
 -> completion(...)
```

这说明搜索和网页抓取之间天然存在一个“候选源处理层”。博客知识库系统可以借鉴该层，但需要把它从进程内调用升级为持久化状态机：

```text
SearchQueryPlan
 -> SearchResponseSnapshot
 -> SearchResultEvidence
 -> SearchCandidateAdmission
 -> Optional Deep Fetch Task
 -> SearchRerankEvidence
```

任何一步都需要可重试、可追踪、可审计，而不是一次 Python 调用结束后只留下最终 Context。

## 3. Search Provider 抽象的价值与缺陷

OpenDeepSearch 在 `serp_search` 中定义统一 Search API，并提供 Serper 与 SearXNG 实现。这个抽象方向是正确的：业务层不应该直接依赖某个搜索后端的请求参数和返回结构。

当前实现会把不同 Provider 结果归一为较简洁的通用字段，例如标题、链接、摘要、日期。这对 Agent 消费很方便，但对知识库工程存在一个重要风险：**为了统一接口而丢失 Provider 原生 provenance**。

例如 SearXNG 原始结果可能包含 engine、score、category、排名和不同引擎聚合信息。如果 Normalizer 只保留 title/link/snippet/date，后续就无法回答：

- 这个 URL 是哪个搜索引擎发现的；
- 原始 score 与当前重排 score 分别是多少；
- 多个引擎是否同时命中；
- 哪次 Query、哪一页返回；
- 原始响应是否被客户端截断；
- 为什么它最终成为 Gap Candidate。

因此生产方案必须采用双层模型：

```text
Raw Provider Response（不可变，完整保存）
  -> Canonical Search Result Envelope（统一字段）
     + provider_fields_json（保留 Provider 专有字段）
     + raw_response_snapshot_id（可回溯原始响应）
```

统一抽象不能以牺牲证据为代价。

## 4. Default / Pro 两级模式的技术启发

OpenDeepSearch 的 SourceProcessor 体现了一个很实用的成本控制思想：默认模式尽量直接使用搜索结果；Pro 模式才抓取候选网页、切块并做语义重排。

这可以映射为博客知识库系统的两级 Search Gap：

```text
SEARCH_GAP_FAST
  Search Provider
  -> Raw Response
  -> Normalize / Scope / Dedup
  -> Gap Candidate

SEARCH_GAP_DEEP
  Gap Candidate Top-N（仅预算范围）
  -> 标准 Fetch Route
  -> Snapshot
  -> Chunk
  -> Reranker
  -> SearchRerankEvidence
  -> 人工/规则 Admission
```

FAST 用于低成本发现“可能遗漏的 URL”；DEEP 用于人工诊断、迁移站点分析、旧域名排查和候选优先级排序。

关键原则：

1. DEEP 模式不是历史枚举器，不能把 Top-N 当成 Coverage complete。
2. Reranker 只决定“优先看什么”，不能决定 FULL_BACKFILL 中合法文章是否永久丢弃。
3. 深抓结果必须重新经过 Scope、Admission、Fetch、Quality、Identity，而不是从 Search 结果直接发布到知识库。

## 5. Crawl4AI 抓取实现细节

`context_scraping/crawl4ai_scraper.py` 支持多个策略，包括 Markdown/HTML/fit-markdown、CSS、XPath、无额外抽取和 cosine 等策略，并可结合 `PruningContentFilter`、`DefaultMarkdownGenerator` 与 Crawl4AI 的浏览器能力。

这体现了“一个站点不应该只有一种正文提取方式”的思路，但当前运行方式更偏 Demo/Agent：

- 不同策略可能顺序执行；
- 抓取函数内部会创建 `AsyncWebCrawler` 生命周期；
- 多 URL 使用 `asyncio.gather` 聚合任务；
- 默认可使用 `CacheMode.BYPASS`；
- 没有持久任务、lease、跨进程限流和全局 backpressure。

对于 1000 站生产系统，直接复用这种运行模型会产生几个问题。

### 5.1 Browser/Crawler 生命周期过短

如果每次 URL 或每种策略都 `async with AsyncWebCrawler(...)`，会反复创建/销毁浏览器资源、连接和上下文，带来高启动成本、内存抖动、文件描述符压力和吞吐下降。

生产模型应改为：

```text
Worker Process
  -> Long-lived Browser Runtime
  -> Context Pool（按站点/租户隔离）
  -> Page/Tab Lease
  -> URL Fetch
```

Crawl4AI 只作为 Engine Adapter，生命周期由 Worker Pool 管理。

### 5.2 不应“每个 Extraction Strategy 再抓一次网页”

多策略抽取应尽可能消费同一份不可变 Snapshot：

```text
一次网络访问
 -> HTTP_BYTES / HTML_SOURCE / HTML_RENDERED Snapshot
 -> CSS Extractor
 -> XPath Extractor
 -> Readability/Trafilatura
 -> Crawl4AI Markdown Candidate
 -> 质量比较
```

只有当证据表明 representation 不足，例如 HTTP 页面为空、必须 JS 渲染，才升级 Browser 获得新的 Rendered Snapshot。不能为了比较 `markdown_llm/html_llm/css/xpath` 等策略反复请求源站。

这既节省网络与浏览器成本，也保证不同 Extractor 的比较基于同一版本页面，避免页面在两次请求之间变化导致错误结论。

### 5.3 `asyncio.gather` 不是生产调度器

`asyncio.gather(*tasks)` 适合小规模进程内并发，但不能承担百万 URL 的生产 Frontier。生产系统需要：

- PostgreSQL 持久任务；
- Transactional Outbox；
- Redis Streams/Kafka 运输；
- per-site token bucket；
- Worker lease/heartbeat；
- bounded concurrency；
- retry/dead-letter；
- queue lag/backpressure；
- run/stage deadline。

本地 semaphore 只能保护 Worker 资源，不能代替持久调度。

## 6. Chunk + Rerank 的原理与正确落点

OpenDeepSearch 在深度模式中会先抓网页内容，再用 Chunker 切块，随后调用 Jina 或 Infinity 语义模型进行重排，取最相关片段构造上下文。

这对问答检索很有效，因为模型不必处理整篇长文；但用于知识库抓取时必须区分两个概念：

- **Canonical ingestion**：保存完整、结构保真的文章内容；
- **Query-time relevance**：针对当前 Query 选择最相关 Chunk。

Reranker 不能反向决定 Canonical Document 保存哪些段落。否则不同 Query 会得到不同“正文”，知识库失去确定性。

正确数据模型应增加：

```text
reranker_release
search_source_fetch
search_rerank_evidence
```

`search_rerank_evidence` 至少保存：

```text
query_id
search_result_evidence_id
source_snapshot_id
chunk_id / source_block_ids
reranker_release_id
raw_score
normalized_score
rank_before
rank_after
created_at
```

这样既能复现“为什么这个结果被排到前面”，也不会污染 Canonical IR。

## 7. 同步 HTTP 客户端与连接复用问题

当前 Search Provider 代码使用 `requests` 风格同步请求，这在低频 Agent 查询里足够简单，但在 1000 个站点的 Search Gap/Reconcile 中会限制吞吐并占用线程。

生产 Search Adapter 应使用长生命周期异步客户端，例如 `httpx.AsyncClient`/aiohttp，并按 Provider Release 管理：

```text
connection_limit
keepalive_limit
connect_timeout
read_timeout
write_timeout
pool_timeout
provider_concurrency_limit
rate_limit
circuit_breaker
retry_policy
```

不能每个 query 新建一个完整 client，也不能把 Search 请求与 Source Fetch 共用没有隔离的连接池和限流策略。

## 8. CacheMode.BYPASS 的边界

深度搜索为了获取实时内容使用 bypass cache 有其合理性，但博客知识库的长期采集不能把“永远绕过缓存”作为默认策略。

知识库更适合分层：

1. Provider 元数据先判断是否可能变化；
2. HTTP conditional request 使用 ETag/Last-Modified；
3. body hash 判断网络内容是否变化；
4. Canonical IR hash 判断知识内容是否变化；
5. 只有确实变化才产生新 Document Version 和昂贵下游 Projection。

这比每次强制重新抓取更适合长期增量同步。

## 9. Agent Tool 参数不应成为生产真相

OpenDeepSearch Tool 为交互体验会采用固定 Pro 模式、小 `max_sources` 等参数。这些参数属于“交互工具默认值”，不应直接进入生产 Crawling Policy。

生产系统必须把真实有效配置固化到 Run：

```text
declared_config
resolved_config
effective_config_hash
consumed_keys
unknown_keys
unused_keys
defaulted_keys
runtime_attestation
```

这样才能避免“UI 写了一个值，但底层组件没有消费”“注释说是 BFS，运行时其实是 DFS”等配置漂移。

## 10. 对博客知识库方案的具体改进

基于 OpenDeepSearch 的实现，方案应增加或强化以下能力：

1. **Search Gap FAST/DEEP 两级诊断**：普通补漏只保存 SERP Evidence；深度模式只对有价值候选做网页抓取和语义重排。
2. **Search Provider 原始证据优先**：先保存完整响应，再做 Canonical Envelope；Provider 专有字段不能因统一接口丢失。
3. **Reranker Release**：Jina、Infinity 或其他 reranker 作为可替换、可版本化组件，保存模型和参数版本。
4. **Search Rerank Evidence**：Query、Result、Fetch Snapshot、Chunk、Score、Rank 全链路可回溯。
5. **一次 Fetch、多策略离线抽取**：同一页面的 CSS/XPath/Readability/Crawl4AI 等尽量共享 Snapshot，避免策略级重复访问源站。
6. **Browser Runtime/Context Pool**：禁止生产 Worker 为每个 URL 创建完整 Crawl4AI/Playwright Runtime。
7. **有界并发**：用持久队列和 per-site/provider budget 代替无边界 `asyncio.gather`。
8. **Rerank 不参与 Canonical 取舍**：语义相关性只影响 Search/Agent Projection，不删除或截断知识正文。
9. **Search Gap 不能提升完整性真相**：搜索只提供 Gap Evidence，历史完整性仍来自官方 API、Sitemap、Feed、Archive 等 Provider Coverage。
10. **Web 端增加 Deep Search Trace**：能查看 Query → Search Result → Deep Fetch → Chunk → Rerank → Admission 的完整证据链。

## 11. 最终结论

OpenDeepSearch 最值得借鉴的是“可替换 Search Provider + SourceProcessor + 选择性深抓 + Chunk/Rerank + Context”的分层思想，以及低成本默认模式与昂贵 Pro 模式的分级策略。

不能直接照搬的是其偏交互式 Agent 的运行模型：同步 Search 请求、进程内 `asyncio.gather`、较短生命周期的 Crawl4AI 实例、固定 Top-N、强制深度模式以及面向 Agent 的精简结果结构。这些实现用于低频问答没有问题，但在 1000 站、百万文档、长期增量运行的知识库中，必须升级为持久状态、异步连接池、资源池、有界调度、不可变 Snapshot、完整 provenance、版本化 Reranker 和可审计 Evidence。

因此，本项目应把 OpenDeepSearch 作为 **Search Gap / Deep Search Diagnostic 的参考架构**，而不是主抓取引擎；主链路继续坚持 Source Registry + Coverage Evidence + durable frontier + HTTP-first + Browser fallback + Canonical IR + append-only Document Version。