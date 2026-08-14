# OpenDeepSearch：开源深度搜索系统——实现细节与技术原理分析

- 编号：34
- 项目地址：https://github.com/sentient-agi/OpenDeepSearch
- 调研基线：`main`，HEAD `ec7aa06dc5ead71821a3d92ea56e54a8a9d16ece`
- 调研对象：OpenDeepSearch 当前公开代码、README 与运行链路
- 适用判断：适合作为“Search Gap 缺口发现 + 候选网页深抓 + Chunk 重排 + Agent 上下文构建”的参考实现；不适合直接承担约 1000 个站点的生产级全量历史抓取、增量同步、覆盖率证明与长期知识版本管理。

## 1. 项目定位

OpenDeepSearch 的目标是给 AI Agent 提供深度网页搜索，而不是做通用分布式爬虫。核心链路可抽象为：

```text
User Query
  -> Search Provider（Serper / SearXNG）
  -> SERP Results
  -> SourceProcessor
  -> 可选网页抓取（Crawl4AI）
  -> Chunk
  -> Semantic Rerank（Jina / Infinity）
  -> Context Builder
  -> LiteLLM / Agent
```

它最值得借鉴的是职责分层：搜索负责候选发现，SourceProcessor 决定是否进一步抓取，Chunk/Rerank 负责 Query-time 相关性，Context Builder 再把结果投影为 Agent 可消费的文本。这个结构非常适合博客知识库的 Search Gap Diagnostic Plane，但不能替代 Source Registry、Provider Coverage、durable frontier、不可变 Snapshot、Canonical IR、Document Version 和增量 checkpoint。

## 2. 代码结构与真实调用链

### 2.1 `ods_tool.py`：Agent Tool 外壳

`OpenDeepSearchTool.forward()` 固定执行：

```python
self.search_tool.ask_sync(query, max_sources=2, pro_mode=True)
```

这意味着默认 Tool 入口实际上选择了：

- 深度模式；
- 最多处理少量来源；
- 最终返回 LLM 生成的字符串答案。

这对低延迟 Agent 交互合理，但 `max_sources=2` 明显只是成本预算，不能被解释为历史文章完整性边界。生产知识库不能把 Agent Tool 的默认参数当作 Crawling Policy 真相。

### 2.2 `ods_agent.py`：搜索、抓取、上下文、LLM 编排

`OpenDeepSearchAgent.search_and_build_context()` 的链路是：

```text
self.serp_search.get_sources(query)
  -> self.source_processor.process_sources(..., max_sources, query, pro_mode)
  -> build_context(...)
```

`ask()` 再把构建出来的 Context 发送给 LiteLLM。

这里有一个容易被忽略的语义：`max_sources` 主要控制后续 SourceProcessor 处理多少候选，并不等价于 Search Provider 实际请求多少结果。生产系统必须同时保存“请求意图”和“运行时实际请求参数”，不能只记录 UI 上的 `max_sources`。

推荐生产模型：

```text
SearchQueryPlan
 -> SearchExecutionManifest
 -> SearchResponseSnapshot
 -> SearchResultEvidence
 -> Candidate Admission
 -> Optional Deep Fetch Task
 -> SearchRerankEvidence
 -> Agent/UI Projection
```

其中每一步都是持久化、可重试、可审计状态，而不是一次 Python 调用结束后只保留最终 Context。

## 3. Search Provider 抽象：方向正确，但当前会损失 provenance

`serp_search/serp_search.py` 定义统一 `SearchAPI`，并提供 Serper 与 SearXNG 两个实现。这个抽象方向是正确的：业务层不应耦合某一家搜索后端。

### 3.1 Serper 实现

Serper 通过同步 `requests.post()` 调用，参数包括：

```text
q
num = clamp(1..10)
gl
```

返回后，`organic` 只保留：

```text
title
link
snippet
date
```

其他 Provider 原始字段在当前运行链路中不会继续保留。

### 3.2 SearXNG 实现

SearXNG 使用同步 `requests.get()`，当前参数中硬编码或默认包含：

```text
pageno=1
categories=general
language=all
safesearch=0
engines=google,bing,duckduckgo
max_results=clamp(1..20)
```

随后客户端再对 `data.results[:num_results]` 截断，并转换为 Serper 类似结构：

```text
title
link
snippet
date
```

这会丢失 SearXNG 原始结果中非常重要的字段，例如 engine/engines、score、category、position、provider-specific metadata 等。

另外当前实现固定 `pageno=1`，没有生产级分页状态机，因此不能把“返回若干 Search Result”解释为枚举完成。

### 3.3 正确生产模型

统一接口不能以丢失证据为代价。应采用双层结构：

```text
Raw Provider Response（完整、不可变）
  -> Canonical Search Result Envelope
     + provider_fields_json
     + raw_response_snapshot_id
```

并额外保存 SearchExecutionManifest：

```text
requested_query
requested_num_results
requested_page_range
requested_engines
requested_language
requested_categories

effective_query
effective_num_results
effective_page
effective_engines
effective_language
effective_categories
provider_release_id
request_started_at
request_finished_at
provider_request_id
```

这样才能区分“用户/策略想请求多少”和“客户端 clamp、默认值、Provider 限制后实际请求了多少”。

## 4. Provider count 语义必须拆开

OpenDeepSearch 当前面向 Agent，最终更关心 Top-N 内容，而不是覆盖率统计。博客知识库则必须严格拆开：

```text
provider_reported_total
provider_page_item_count
client_returned_count
normalized_unique_count
in_scope_count
new_gap_candidate_count
truncated_by_client
```

特别不能把 `len(results[:num_results])` 当作“搜索引擎总结果数”。

Search Top-N、搜索页数、`max_sources` 都只是成本预算，不是 Coverage Evidence。

## 5. Default / Pro 两级模式的工程启发

README 和 SourceProcessor 都体现了两级策略：

- Default：尽量直接使用 SERP 信息，低成本、低延迟；
- Pro：进一步抓候选网页、切块、语义重排，再构造更丰富上下文。

这非常适合映射为博客知识库的两种诊断模式：

```text
SEARCH_GAP_FAST
  Search Provider
  -> Raw Search Snapshot
  -> Normalize
  -> Scope
  -> Dedup
  -> Gap Candidate

SEARCH_GAP_DEEP
  FAST Candidate Top-N（预算）
  -> Admission Precheck
  -> 标准 Fetch Route
  -> Fetch Snapshot
  -> Chunk
  -> Reranker
  -> SearchRerankEvidence
  -> 人工/规则 Admission
```

FAST 用于周期补漏，DEEP 用于迁移域名、旧 URL、疑似缺口和人工诊断。

关键边界：

1. DEEP 不是历史枚举器；
2. Rerank 只能决定优先级，不能决定 FULL_BACKFILL 合法文章是否永久丢弃；
3. Deep Fetch 必须重新经过 Scope、SSRF、robots、限流、Fetch、Quality、Identity；
4. Search Evidence 永远不能单独把 Coverage 提升为 complete。

## 6. `SourceProcessor`：好的分层，但错误路径存在类型契约风险

`context_building/process_sources_pro.py` 的 `SourceProcessor` 把搜索结果处理拆出来，这是很值得借鉴的职责边界。

当前关键行为：

- 默认 `strategies=["no_extraction"]`；
- `pro_mode=False` 时通常直接使用搜索结果，若命中 Wikipedia 才对少量来源进一步处理；
- `pro_mode=True` 时抓取候选页面，再 Chunk + Rerank；
- `_fetch_html_contents()` 调用 Crawl4AI WebScraper；
- `_process_html_content()` 把内容切块后取重排 Top-K。

但当前代码存在一个生产级系统必须规避的问题：函数注解与实际返回形态并不稳定。

例如：

- 参数标注看起来像 `List[dict]`，实际代码依赖 `sources.data`；
- 某些路径返回 `sources.data`；
- 异常路径可能直接返回 `sources` 对象；
- 下游 `build_context()` 则期待字典并调用 `.get()`。

这类“异常被 catch 后返回另一种类型”的写法，在 Demo 中可能只是得到空结果，在持久任务系统中会造成不可审计的 contract drift。

生产设计必须统一结构化 Outcome：

```text
status=SUCCESS|PARTIAL|FAILED
payload_ref / data
error_class
error_message
retryable
provider_request_id
raw_snapshot_id
runtime_release_id
```

禁止 `dict/list/wrapper/object` 在不同错误路径之间互相替代，也不能用空字符串掩盖契约错误。

## 7. Crawl4AI WebScraper：多策略能力强，但当前会重复网络抓取

`context_scraping/crawl4ai_scraper.py` 支持：

```text
markdown_llm
html_llm
fit_markdown_llm
css
xpath
no_extraction
cosine
```

并使用 `PruningContentFilter`、`DefaultMarkdownGenerator` 与 Crawl4AI 浏览器能力。

### 7.1 每个 Extraction Strategy 可能重新创建 Crawler

`scrape()` 会依次遍历策略，每个策略调用 `extract()`；而 `extract()` 内部执行：

```python
async with AsyncWebCrawler(...) as crawler:
    result = await crawler.arun(...)
```

因此如果启用多个 strategy，就可能为了比较不同抽取方式而重复创建 crawler 生命周期、重复访问同一网页。

生产系统应改为：

```text
一次 Fetch
 -> 不可变 Snapshot
 -> CSS Extractor
 -> XPath Extractor
 -> JSON-LD Extractor
 -> Readability/Trafilatura
 -> Crawl4AI Candidate
 -> Quality Selector
```

只有证据表明 HTTP representation 不足时，才升级 Browser 获得新的 `HTML_RENDERED` Snapshot。网络访问与内容解释必须解耦。

### 7.2 Browser/Crawler 生命周期过短

每 URL/每 strategy 都启动和关闭完整 crawler，会带来：

- 浏览器启动成本；
- 内存抖动；
- 文件描述符与连接压力；
- Context/cookie 隔离难统一；
- 大规模吞吐下降。

生产模型应为：

```text
Browser Worker Process
 -> Long-lived Browser/Crawler Runtime
 -> Context Pool（按站点/租户隔离）
 -> Page/Tab Lease
 -> Fetch
```

Crawl4AI 是 Engine Adapter，不负责生产 Worker 生命周期。

### 7.3 `asyncio.gather` 不是生产 Frontier

`scrape_many()` 会把输入 URL 转成协程后 `asyncio.gather(*tasks)`。

这对少量 URL 很方便，但对于数百万 URL 不能承担：

- durable scheduling；
- 跨进程公平调度；
- lease/heartbeat；
- retry/dead-letter；
- per-site rate limit；
- backpressure；
- stage/run deadline。

生产系统应使用 PostgreSQL durable task + Transactional Outbox + Redis Streams/Kafka；`asyncio` semaphore 只保护单个 Worker 的局部资源。

## 8. `FastWebScraper` 的适用边界

`fast_scraper.py` 会使用 Crawl4AI 获取 HTML，再调用 vLLM/ReaderLM 做内容提取。它同样会在 `scrape()` 中创建短生命周期 `AsyncWebCrawler`，`scrape_many()` 则逐 URL 顺序执行。

它说明 LLM extraction 可以作为 Candidate Extractor，但不能成为 Canonical ingestion 唯一真相，因为：

- 模型版本和 prompt 变化会改变输出；
- LLM 可能丢块、重写或概括；
- 长上下文模型有显著 GPU/成本压力；
- 输出 parser 失败时可能退化为原始文本。

因此 LLM extraction 应是版本化 Projection/Candidate，并保留原始 Snapshot 与结构化 Canonical IR 作为可重放基础。

## 9. Chunker：适合 Query-time，不适合决定 Canonical 正文

`ranking_models/chunker.py` 使用 LangChain `RecursiveCharacterTextSplitter`，当前默认大致为：

```text
chunk_size=150
chunk_overlap=50
separators=["\n\n", "\n"]
length_function=len
```

也就是说它按字符长度与文本分隔符切块，而不是按知识库 Canonical Block 或 tokenizer token 切块。

这对 Query-time Rerank 简单有效，但生产系统必须：

- `chunker_release` 版本化；
- Chunk 保存 `source_block_ids`；
- Chunker 改版只触发 REINDEX，不重新抓源站；
- Canonical IR 不依赖当前 Query 的 chunk 结果；
- 技术博客需特别处理代码块、表格、标题层级和超长代码段。

## 10. Rerank 原理：当前是 embedding 点积 + 候选集内归一化

`base_reranker.py` 的核心流程：

```text
query_embeddings
 document_embeddings
 -> dot product
 -> normalize（默认 softmax）
 -> top-k
```

### 10.1 Softmax score 是候选集相对分数

默认 `softmax` 是对当前候选文档集合归一化。因此同一个 Chunk：

- 在 10 个候选中得到 0.42；
- 在 100 个候选中可能得到完全不同的归一化分数。

这种 score 不能跨不同 Query Run、不同 candidate set 直接比较。

生产 `search_rerank_evidence` 除了保存 `raw_score` 和 `normalized_score`，还必须保存：

```text
similarity_metric
normalization_method
normalization_scope
candidate_set_hash
candidate_set_size
top_k_requested
top_k_returned
```

只有 `reranker_release + candidate_set_hash + normalization_method` 一致时，normalized score 才具有可比语义。

### 10.2 当前 `get_reranked_documents()` 会丢弃分数

`get_reranked_documents()` 最终只把 Top-K document 文本拼接为字符串，原始 score 不会进入 Context。

Agent 交互里这很方便；生产 Evidence 链则必须保留：

```text
query
chunk
raw score
normalized score
rank before
rank after
model/release
```

然后另外生成面向 Agent 的紧凑 Context Projection。

## 11. Infinity Reranker 的 query/document role 风险

`infinity_rerank.py` 的 `_get_embeddings()` 支持 `embedding_type`，默认值为 `"query"`，并会给 query 加 instruction prefix。

但基类 `BaseSemanticSearcher.calculate_scores()` 对 query 和 document 都直接调用 `_get_embeddings(texts)`，没有显式传入不同 role。

这意味着当前 Infinity 子类在默认调用路径下，document 也可能走默认 query role/instruction prefix。即使底层模型在某些情况下仍能工作，这种“query/document 编码角色没有显式进入接口”的设计会使得运行语义不透明，也可能造成检索质量偏差。

生产 Reranker Adapter 必须把编码角色变成一等配置：

```text
query_encoding_mode
document_encoding_mode
query_instruction
document_instruction
instruction_digest
embedding_dimensions
model_digest
similarity_metric
normalization_method
```

并在 Capability Test 中用固定向量样本验证 query/document role，而不能只记录模型名称。

## 12. Jina Reranker 的特点

`jina_reranker.py` 当前实际调用的是 Jina Embeddings API，任务参数为 `text-matching`，再由基类做 dot-product 排序。

因此“Reranker”在这里本质上更接近“embedding-based semantic reranking”，而非独立 cross-encoder rerank endpoint。

方案中应避免把所有 Reranker 统一假设为同一种 score 语义。Reranker Release 至少应区分：

```text
EMBEDDING_DOT_PRODUCT
COSINE
CROSS_ENCODER
PROVIDER_NATIVE_RERANK
```

不同 score 类型不能无条件混用阈值。

## 13. Search 与 Rerank 均使用同步 HTTP，生产批量会阻塞

Serper、SearXNG、Infinity、Jina 当前都采用 `requests` 风格同步网络调用。

低频 Agent 请求简单可靠，但 1000 站周期 Reconcile/Search Gap 时会：

- 占用线程；
- 降低连接复用效率；
- 难统一 cancellation；
- 难做 provider-specific backpressure。

生产 Adapter 应复用长生命周期 `httpx.AsyncClient`/aiohttp client，并按 Provider Release 管理：

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

Search、Source Fetch、Rerank 建议使用独立连接池和独立预算，避免一个 Provider 拖垮主抓取链路。

## 14. CacheMode.BYPASS 的边界

Crawl4AI 路径默认可使用 `CacheMode.BYPASS`。对于实时 Deep Search 合理，因为用户关心最新页面。

长期知识库不应每次都强制重新抓取。更合适的是：

```text
Feed/Sitemap/API change signal
 -> ETag / Last-Modified conditional request
 -> body hash
 -> Extraction
 -> Canonical IR hash
 -> unchanged: freshness observation
 -> changed: append document_version
```

这样可显著减少长期网络成本、Browser 成本与下游 Embedding/AI 重算。

## 15. Context Builder 只是 Agent Projection，不是知识真相

`build_context.py` 会把标题、日期、链接、snippet 和可选的重排后网页片段拼成文本上下文。

这个结果适合 LLM，但会天然丢失：

- Provider 原始字段；
- score 细节；
- candidate set；
- Chunk/block identity；
- Snapshot provenance；
- Extraction/Quality 决策链。

因此必须把 Context Builder 放在最末端 Projection 层，绝不能让 Context 文本反向成为 Canonical Markdown 或 Document Version。

## 16. 异常处理：不能“吞掉错误后继续像成功一样返回”

项目多个位置倾向于 catch 异常后返回空内容、原始对象或简化错误字符串。这对交互工具可提升容错体验，但对于长期知识库会把以下情况混在一起：

```text
真实空页面
抓取失败
schema mismatch
rerank provider 失败
超时
类型契约错误
过滤后空内容
```

生产 Outcome 必须显式区分，并持久化错误类别。特别需要：

```text
SEARCH_BAD_RESPONSE
SEARCH_STAGE_CONTRACT_ERROR
FETCH_EMPTY_EXPECTED
FETCH_EMPTY_UNEXPECTED
EXTRACTION_SCHEMA_MISMATCH
RERANK_BAD_RESPONSE
RERANK_ROLE_ENCODING_MISMATCH
DEADLINE_EXCEEDED
```

不能用 `""`、`[]` 或原始 wrapper 作为所有错误的统一替代品。

## 17. Evaluation 与生产正确性测试必须分开

OpenDeepSearch 仓库有面向 SimpleQA、FRAMES 等问答效果的 `evals`，但当前 `tests` 目录基本没有覆盖生产爬虫工程所需的完整测试体系。

QA Benchmark 衡量的是“最终回答是否好”，不能证明：

- Search 原始证据是否完整保存；
- 分页是否正确；
- 429/Retry-After 是否处理；
- Snapshot 是否重复抓；
- query/document embedding role 是否正确；
- score 是否可重放；
- Coverage 是否被错误提升；
- Worker 崩溃后任务是否恢复。

博客知识库必须另外建立 Contract/Golden/Capability/Replay/Failure Injection 测试。

## 18. 对博客知识库技术方案的具体优化

基于以上代码级分析，最终方案应至少明确以下能力：

1. **Search Gap FAST/DEEP 两级诊断**：Search 是缺口传感器，不是历史枚举真相。
2. **Raw Provider Response 优先**：先保存完整响应，再生成 Canonical Envelope。
3. **SearchExecutionManifest**：同时记录 requested/effective query 参数，避免 `max_sources`、Provider `num`、客户端截断混为一谈。
4. **Search Provider provenance**：保留 engine、score、category、position 和 Provider 专有字段。
5. **Typed Search/SourceProcessor Outcome**：成功、部分成功、失败都固定 schema，禁止异常路径改变返回类型。
6. **一次 Fetch、多策略离线抽取**：CSS/XPath/Readability/Crawl4AI/LLM 尽量共享 Snapshot。
7. **Browser Runtime/Context Pool**：生产 Worker 不按 URL/strategy 重建完整 crawler。
8. **Durable Frontier + bounded concurrency**：`asyncio.gather` 仅作为 Worker 局部实现。
9. **Reranker Release**：记录模型、端点、模型 digest、score 类型、query/document role、instruction、dimensions、normalization。
10. **Candidate-set-aware score provenance**：保存 candidate set hash/size；Softmax score 不跨不同集合比较。
11. **Role-aware embedding Contract Test**：分别验证 query/document 编码路径，避免 Infinity 一类接口默认值造成语义漂移。
12. **Search Rerank Evidence**：Query → Result → Snapshot → Chunk → raw score → normalized score → rank 全链路可回溯。
13. **Rerank 不参与 Canonical 取舍**：只影响搜索/Agent 优先级，不删除知识正文。
14. **Async Provider Client Pool**：Search/Rerank 使用长生命周期异步客户端、限流、熔断和 deadline。
15. **Context 只是 Projection**：Agent 输出绝不能覆盖 Raw Evidence/Canonical IR/Markdown。
16. **QA Eval 与 ingestion correctness 分离**：生产发布必须通过 Contract、Golden、Capability、Replay 和故障测试。
17. **Web Deep Search Trace**：展示 requested/effective Search 参数、原始 Provider 数据、Deep Fetch、Chunk、score、candidate set、Admission。

## 19. 最终结论

OpenDeepSearch 最有价值的不是“直接拿来抓 1000 个博客”，而是它展示了一个清晰的 Agent 深搜分层：

```text
Search Provider
 -> Candidate Source Processing
 -> Selective Deep Fetch
 -> Chunk
 -> Semantic Rerank
 -> Context Projection
```

可以直接借鉴的思想包括：Provider 可替换、Default/Pro 成本分级、选择性深抓、Chunk/Rerank、Agent Tool 集成。

不能直接照搬的生产实现包括：同步 Search/Rerank HTTP、固定第一页与 Top-N、精简 Provider 字段、短生命周期 Crawl4AI、strategy 级重复 Fetch、无界 `asyncio.gather`、错误路径返回形态不稳定、Rerank 分数缺少 candidate-set 语义、query/document 编码角色不显式、缺乏面向 ingestion durability/coverage 的系统测试。

因此，OpenDeepSearch 应被定位为 **Search Gap / Deep Search Diagnostic Plane 的参考架构**。博客知识库主链路仍应坚持：

```text
Source Registry
 + Provider Coverage Evidence
 + Durable Frontier
 + HTTP-first / Browser fallback
 + Immutable Snapshot
 + Structure-preserving Canonical IR
 + Append-only Document Version
 + Incremental Checkpoint
 + Web Control Plane
```

Search/Rerank 只增强“发现缺口、排序候选、辅助 Agent”，绝不能成为历史完整性真相或 Canonical 内容裁剪器。