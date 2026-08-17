# 使用 Crawl4AI 构建 RAG

## 1. 调研对象与结论

- 项目：RAG_with_crawl4AI
- 地址：https://github.com/ssime-git/RAG_with_crawl4AI
- 主要技术：Python、Crawl4AI、Playwright、ChromaDB、Sentence Transformers、FastAPI、Streamlit、LiteLLM
- 主要代码：`src/crawler/web_crawler.py`、`src/insert_docs.py`、`src/db/chroma_client.py`、`src/rag_service/main.py`、`src/rag_service/client.py`、`src/llm/client.py`、`src/streamlit_app.py`、`docker-compose.yml`

该项目是一套完整的 RAG 教学型闭环：输入 URL、Sitemap 或文本文件，经 Crawl4AI 抓取为 Markdown，按标题层级切块，通过 FastAPI 写入 ChromaDB，查询时做向量召回，再经 LiteLLM 调用 Gemini 生成答案，Streamlit 提供交互界面。

它适合作为“小规模站点 -> RAG”的参考实现，但不能直接承担“约 1000 个技术博客全量历史抓取 + 长期增量同步 + Web 管理”的生产主链路。它最值得吸收的是组件边界和局部执行方式；最需要避免的是进程内状态、Browser-first、临时 Chunk ID、无版本索引、检索与生成依赖耦合以及缺少覆盖和索引完成证据。

对现有博客知识库方案的新增价值主要有五点：

1. 将 Search/RAG 明确作为 Accepted Document Version 的可重建 Projection；
2. 引入稳定 Chunk Identity 和索引 Manifest；
3. 把 Search API、Context Builder、Answer API、LLM Gateway 再拆分，保证 LLM 故障不影响检索和抓取；
4. 引入 Retrieval Profile / Context Builder Release / Query Trace，使查询同样可版本化、可审计、可回放；
5. 增加针对分块算法、健康检查、Web 参数传递和模型网关的生产级验收项。

## 2. 总体架构与数据流

README 描述的核心拓扑为：

```text
Crawler(insert_docs.py)
        |
        v
RAG Service(FastAPI) <---- Streamlit Web App
        |
        v
     ChromaDB
        |
        +---- LiteLLM / Gemini
```

实际链路为：

```text
URL / Sitemap / .txt
  -> Crawl4AI
  -> Markdown
  -> smart_chunk_markdown
  -> /insert-documents
  -> ChromaDB embedding + HNSW
  -> /retrieve
  -> context string
  -> /rag-query
  -> LiteLLM
  -> Gemini
  -> Streamlit
```

这种拆分已经体现一个正确原则：抓取运行时不直接承担用户问答，Web UI 也不直接访问向量数据库。生产设计应进一步强化为：

```text
Accepted Document Version
  -> Chunk Projection
  -> Embedding / Search Index
  -> Search API
  -> Context Builder
  -> Answer API
  -> Model Gateway
```

其中源站同步和 RAG 服务是两个故障域，任何 LLM、向量索引或问答失败都不能回滚抓取业务事实。

## 3. 抓取路由与发现实现

### 3.1 URL 类型路由

`insert_docs.py` 用字符串判断输入类型：

```text
.txt              -> crawl_markdown_file
包含 sitemap       -> parse_sitemap + crawl_batch
普通 URL           -> crawl_recursive_internal_links
```

这是一个轻量 Source Router，适合 CLI PoC，但不适合长期站点管理。生产中应由 Site Probe 生成版本化 Provider：

```text
SITEMAP
FEED
API
ARCHIVE
HTML_LINKS
DOC_NAV
BROWSER_LISTING
```

Provider 应有独立 checkpoint、覆盖语义、请求路由和 Release，而不是通过 URL 后缀永久决定站点能力。

### 3.2 Sitemap 实现

`parse_sitemap()` 直接使用同步 `requests.get()`，再以 `ElementTree` 查询 `{*}loc`。

优点：

- XML namespace 处理简单；
- 普通 `urlset` 能快速得到 URL。

生产缺陷：

1. 无 timeout；
2. 无 retry/backoff；
3. 无 ETag/Last-Modified；
4. 无 gzip sitemap 处理；
5. 无 sitemap index 递归语义；
6. 无 `lastmod` checkpoint；
7. 同步 `requests` 会阻塞 asyncio 流程；
8. Sitemap 原文、层级、条目数和终止原因没有保存为 Coverage Evidence。

特别是 sitemap index：当前实现会把所有 `<loc>` 当成普通页面 URL 返回，如果 `<loc>` 指向子 sitemap，后续 `crawl_batch()` 会把 sitemap XML 当成网页抓取，而不是继续展开。因此生产 Provider 必须区分 `sitemapindex` 与 `urlset`，并递归展开、去重和记录父子关系。

### 3.3 普通 URL 的 BFS

`crawl_recursive_internal_links()` 使用：

```text
visited
current_urls
next_level_urls
```

每一层用 `arun_many()` 抓取，成功后读取 `result.links["internal"]` 进入下一层。`urldefrag()` 去掉 fragment，可以减少锚点重复。

值得保留：

- `result.links` 非常适合作为 Link Discovery Evidence；
- `arun_many` 适合作为 Worker 内微批执行器；
- `MemoryAdaptiveDispatcher` 适合做浏览器会话局部资源保护。

但这一实现不能证明历史完整：

- `max_depth` 是预算，不是覆盖证据；
- `visited` 只在内存；
- 无 durable frontier；
- 无 lease/heartbeat/retry/dead-letter；
- 无 URL Admission Rule；
- 无 crawler-trap 防护；
- 无“来源页面 + raw href + 规范化结果 + 拒绝原因”的 Link Evidence；
- 单层 URL 可能整体进入一次 `arun_many()`，缺少跨 Worker 的公平调度和 backpressure。

生产设计应将 Crawl4AI Dispatcher 明确限制在 Worker 内，而 PostgreSQL durable frontier 承担业务状态。

## 4. Crawl4AI 并发和浏览器资源控制

项目配置：

```text
MemoryAdaptiveDispatcher
memory_threshold_percent = 60
max_session_permit       = max_concurrent
```

这比无界 `asyncio.gather()` 更可靠：并发上限不仅按任务数，还受进程内存压力约束，适合作为 Playwright Worker 的局部保护。

但所有主要抓取都进入 `AsyncWebCrawler` Browser 路径，并设置 `CacheMode.BYPASS`。对于 1000 站长期同步存在明显成本问题：

- 静态文章也启动浏览器；
- 无条件请求复用；
- Discovery 页和 Article 页无法分别选 HTTP/Browser；
- 重跑无法利用 304；
- Browser 内存和共享内存成为主吞吐瓶颈。

生产应采用 HTTP-first Fetch Router：

```text
HTTP
  -> 正文充分：直接保存 Snapshot
  -> hydration/动态证据：升级 Browser
  -> PDF：PDF Worker
  -> challenge/login：显式策略结果
```

Browser Worker 长期复用 runtime，单次只 lease 小批 durable tasks，再以 `arun_many(stream=True)` 执行，并逐 URL 持久化结果。

## 5. Markdown 分块算法：优点、缺陷与隐藏边界问题

`smart_chunk_markdown()` 先按 H1，再按 H2，再按 H3，超过阈值后按字符切片。默认最大长度 1000 字符。

### 5.1 优点

1. 比纯固定长度切片更尊重 Markdown 章节；
2. `extract_section_info()` 保存 headers、字符数、单词数和来源 URL；
3. 实现简单，适合验证“标题结构可以提升检索上下文”这一思路。

### 5.2 字符预算不是 Token 预算

中文、英文、代码、URL 的 token/字符比例不同。生产 Chunker 应使用固定 tokenizer/version，并配置：

```text
target_tokens
max_tokens
overlap_tokens
heading_context_policy
code/table policy
```

### 5.3 字符硬切会破坏结构

三级标题块超限时直接 `h3[i:i+max_len]`，可能切断：

- 代码块；
- Markdown 表格；
- 列表；
- URL；
- 句子；
- 数学公式。

生产应基于 Canonical Document IR 的 block 边界切分，只在单 block 超限时做次级分割。

### 5.4 代码存在“无 H1 内容丢失”风险

`split_by_header()` 的实现先收集匹配标题的起始位置，再追加 `len(md)`：

```python
indices = [m.start() for m in re.finditer(header_pattern, md, re.MULTILINE)]
indices.append(len(md))
```

如果 Markdown 完全没有 H1，则 `indices` 只有一个元素，`range(len(indices)-1)` 为 0，函数返回空数组。最外层只遍历 H1 结果，因此整个合法 Markdown 会被静默切成 0 个 chunk。

这属于典型的“函数无异常但业务结果为空”。生产 Chunker 必须有显式 Outcome：

```text
VALID
EMPTY_INPUT
NO_STRUCTURAL_HEADING
CHUNKER_ERROR
```

并保证“有正文 -> 至少一个 chunk”。

### 5.5 第一处 H1 前的前导内容会丢失

当 Markdown 在第一个 `# ` 之前有摘要、Front Matter 后的导语或免责声明时，`indices` 从第一个 H1 开始，0 到第一个 H1 之间的正文不会形成 chunk。因此不能直接复用该算法。

生产验收应加入：

- 无 H1 文档；
- 首个 H1 前有正文；
- 只有 H2/H3；
- 单超长代码块；
- 超长表格；
- 中文无空格长段；
- 空 Markdown。

### 5.6 父标题上下文与引用范围

项目 metadata 只记录 chunk 内出现的 headers，不能可靠继承父标题。生产 Chunk Contract 应保存：

```text
heading_path[]
start_block
end_block
chunk_hash
token_count
```

引用应回到 Document Version 的 block range，而不是只回到一段复制文本。

## 6. ChromaDB、Embedding 与向量索引原理

`chroma_client.py` 使用：

```text
chromadb.PersistentClient
SentenceTransformerEmbeddingFunction(all-MiniLM-L6-v2)
collection metadata hnsw:space = cosine
collection.add(...)
```

查询通过 `query_texts=[query]`，由 Collection embedding function 生成查询向量，再由 HNSW 近似最近邻返回结果。

这个设计适合单机 RAG 原型，但 ChromaDB 不能成为长期知识库的业务真相源。生产中向量索引必须是 Accepted Document Version 的 Projection：索引可以删除和重建，源文档、版本、Snapshot 和 IR 仍由 PostgreSQL + S3 保存。

还要注意 `format_results_as_context()` 使用 `1 - distance` 展示 Relevance，这只在距离定义和范围满足预期时才有直观意义，不能把它当成跨索引、跨模型可比较的统一置信度。生产应保留原始 lexical/vector/rerank score，并由 Retrieval Profile 解释融合方式。

## 7. 最关键风险：Chunk Identity 和增量生命周期

`insert_docs.py` 生成：

```python
ids.append(f"chunk-{chunk_idx}")
```

而 `chunk_idx` 每次进程启动都从 0 开始。

后果：

1. 不同站点重复生成 `chunk-0`；
2. 重跑可能和已存在 ID 冲突；
3. 无法绑定 document/version；
4. 无法精确替换文章更新后的旧 chunk；
5. chunker 升级无法区分新旧 projection；
6. 无法安全做 REINDEX；
7. 无法从 RAG 引用回溯到原始版本。

生产 Chunk ID 应稳定派生，例如：

```text
chunk_id = sha256(
  document_version_id
  + chunker_release_id
  + start_block
  + end_block
  + chunk_hash
)
```

并保存：

```text
document_id
document_version_id
chunker_release_id
heading_path[]
start_block/end_block
chunk_hash
token_count
text_artifact_ref
```

Embedding 和索引也必须版本化：

```text
embedding_release
search_index_release
index_projection_job
index_projection_manifest
```

模型升级不应在同一 Collection 中静默混写，而应：

```text
ACTIVE v1
 -> BUILDING v2
 -> completeness/eval
 -> SHADOW
 -> ACTIVE v2
 -> RETIRED v1
```

## 8. 入库 API：为什么 `collection.add()` 不足以承担生产同步

`/insert-documents` 最终调用 `collection.add()`，没有：

- document version；
- idempotency key；
- index job；
- partial batch checkpoint；
- expected/actual chunk 对账；
- 新旧版本切换；
- stale chunk 回收；
- source snapshot lineage。

生产链路应改为：

```text
Accepted document_version
  -> outbox INDEX_DOCUMENT_VERSION
  -> Chunk Projection Worker
  -> chunk_projection
  -> Embedding Worker
  -> Index Upsert
  -> index_projection_manifest
  -> Finalizer
```

文章更新时：

1. 先 Accepted 新版本；
2. 新版本全部 chunk 完成；
3. 新版本索引 Manifest 对账；
4. 原子切 current indexed version；
5. 旧 chunk 标记 stale；
6. 后台回收旧索引。

这样即使中间失败也不会产生检索空窗。

## 9. RAG Service：正确的分层方向与依赖耦合问题

FastAPI 暴露：

```text
/health
/retrieve
/generate
/rag-query
/insert-documents
```

把 retrieve 与 generate 分开是正确方向，但实现仍存在重要耦合。

### 9.1 Retrieval 被 LLM Secret 绑架

`rag_service/main.py` 在模块加载时检查 `GOOGLE_API_KEY`，缺少时直接 `sys.exit(1)`。这意味着：即使只想做向量检索，服务也无法启动。

生产中应拆成：

```text
Search API
Answer API
Model Gateway
```

Search API 只依赖 Search Backend；Answer API 才依赖 Model Gateway 和 Secret。LLM 停机、额度耗尽、Key 轮换都不能影响 `/search`。

### 9.2 `/health` 不是真正 readiness

当前 `/health` 固定返回 healthy，没有实际检查：

- Chroma 是否可读写；
- embedding 模型是否可用；
- LiteLLM 是否可用；
- 当前 Collection/Index 是否 ready。

生产应分：

```text
/health/live
/health/ready/search
/health/ready/answer
/health/components
```

并明确 Search Ready 和 Answer Ready 是不同状态。

### 9.3 错误字符串可能被当作正常答案

`LLMClient.generate()` 遇到 HTTP/连接错误时返回形如 `Error: ...` 的字符串，而不是抛出结构化异常。`rag_query()` 会把这个字符串当作正常 `answer` 返回 200。

生产必须使用结构化 Outcome：

```text
SUCCEEDED
RETRYABLE_ERROR
RATE_LIMITED
AUTH_ERROR
MODEL_ERROR
TIMEOUT
```

HTTP 状态、业务状态和生成文本不能混在同一个字符串字段中。

### 9.4 README 与实际 API 存在接口漂移

README 示例出现 `/documents`、`/rag/query` 等路径，而代码实际使用 `/insert-documents`、`/rag-query`。这说明接口文档和实现没有契约生成或测试约束。

生产应以 OpenAPI schema 为唯一接口契约，并在 CI 中验证 Web Client / SDK 与 API schema，避免 Web 管理端静默调用旧路径。

## 10. Retrieval 不应直接输出“拼好的字符串”

当前 `/retrieve` 最终调用 `format_results_as_context()`，把多个命中拼成一个字符串：

```text
Document 1...
metadata...
Content...
Document 2...
```

这种方式方便 Demo，但生产会丢失机器可读边界。Search API 应返回结构化 `RetrievalHit[]`：

```text
chunk_id
document_version_id
source_url
heading_path
block_range
lexical_score
vector_score
fusion_score
rerank_score
text_ref
metadata
```

再由独立 Context Builder 完成：

- token budget；
- 相邻 chunk 扩展；
- 同文档去重；
- 每站点/每文档上限；
- heading path 注入；
- 过长代码压缩策略；
- citation label 绑定。

因此应新增：

```text
retrieval_profile_release
context_builder_release
query_trace
```

每次问答可以复盘“用哪个索引、哪套融合参数、选了哪些 chunk、为什么被截断”。

## 11. 技术博客检索应采用 Hybrid Search

项目目前主要是向量 Top-K。技术博客大量包含：

- 函数名；
- 类名；
- 错误码；
- CLI 参数；
- 版本号；
- 文件路径；
- API 名称。

这些精确字符串通常需要词法检索。因此生产默认应为：

```text
FTS/BM25
   +
Vector ANN
 -> RRF/weighted fusion
 -> metadata filters
 -> optional reranker
 -> citation binding
```

并建立评测集跟踪：

```text
Recall@K
MRR/NDCG
citation hit rate
answer groundedness
index completeness
latency
cost
```

## 12. Streamlit Web App 暴露出的控制面契约问题

项目 UI 提供 collection、n_results、temperature 等侧边栏控件，但实际调用链并没有完整使用这些值。

### 12.1 UI 配置没有真正进入请求

`main()` 中创建：

```text
collection_name
n_results
temperature
```

但 `stream_response()` 调 `generate_answer()` 时仍使用固定参数，`generate_answer()` 请求 `/rag-query` 时又固定 `n_results=5`，也没有传 `collection_name`。因此用户看到的控件与服务实际行为可能不一致。

生产 Web Control Plane 必须做到：

```text
UI form
 -> command/request payload
 -> persisted query_trace
 -> API execution
```

界面展示的 Release/参数必须和服务实际使用的一致。

### 12.2 “流式输出”实际是假流式

`stream_response()` 先等待完整答案，再按字符循环并 `sleep(0.01)` 模拟流式显示。它不降低首 token 延迟，也不支持服务端中断。

生产 Answer API 若需要 streaming，应使用真实 SSE/WebSocket/HTTP chunked 流，并把 retrieval trace 在生成前就固定保存。

### 12.3 Retrieved Context 展示链路不一致

`process_user_query()` 会保存 context 到 `session_state.contexts`，但主流程实际调用的是 `stream_response()`，后者没有保存 context。因此“View Retrieved Context”可能没有本次查询对应的 context。

这再次说明生产系统不能把 Query Trace 只存在 UI session 中，应由后端生成持久 `query_trace_id`，Web 通过该 ID 查看命中、上下文和引用。

## 13. Docker Compose 的可借鉴点与生产问题

Compose 把 `litellm`、`rag-service`、`rag-app`、`crawler` 分成独立服务，这是正确的故障域拆分方向。Crawler 还设置了 Browser 的内存、CPU 和 `shm_size`，说明 Playwright 需要独立资源预算。

但生产化还需修正：

1. `litellm` 使用 `main-latest`，不可复现，应锁定版本和镜像 digest；
2. crawler 实际只需调用 RAG Service 插入文档，却被配置成依赖 litellm healthy，造成不必要耦合；
3. rag-service 又因 `GOOGLE_API_KEY` 启动检查而把插入/检索和生成绑定；
4. 单机 Chroma named volume 不适合作为百万级长期索引的唯一服务形态；
5. 没有独立 Worker 队列、autoscaling、durable job、index finalizer；
6. 开发环境把源码目录 bind mount 到容器，生产镜像应不可变并记录 digest。

应保留“Browser/LLM/Search 独立 Pool”的思想，但把依赖方向改成：

```text
Crawler/Indexer ----> State DB / Object Store / Search Backend
Search API ---------> Search Backend
Answer API ---------> Search API + Model Gateway
Model Gateway ------> External LLM
```

抓取和索引不依赖 Answer API 是否健康。

## 14. 对博客知识库技术方案的具体优化

### 14.1 新增查询侧版本化模型

在现有 `chunker_release / embedding_release / search_index_release` 基础上新增：

```text
retrieval_profile_release
context_builder_release
query_trace
```

`retrieval_profile_release` 至少声明：

```text
search_index_release_id
lexical/vector top_k
fusion algorithm/weights
metadata filter policy
reranker/model release
score normalization
```

`context_builder_release` 至少声明：

```text
tokenizer/version
max_context_tokens
per_document_limit
adjacent_chunk_policy
dedupe_policy
heading_prefix_policy
code/table policy
citation_label_policy
```

`query_trace` 保存：

```text
query
retrieval_profile_release_id
context_builder_release_id
search_index_release_id
retrieved_hits
selected_hits
score breakdown
context token count
answer_profile/prompt/model release
latency/cost
result status
```

### 14.2 Search API 与 Answer API 分离

Search API 返回结构化命中，不直接拼 prompt；Answer API 消费 Query Trace/Context Builder 输出，经 Model Gateway 生成带引用答案。

这样可以：

- LLM 不可用时继续搜索；
- 独立评测 retrieval；
- 独立扩容 Search 与 Generation；
- Query 可重放；
- 模型切换不影响抓取和索引。

### 14.3 Chunker 加入业务不变量

必须满足：

```text
正文非空 => 至少产生 1 个 chunk
chunk 可回溯到 document_version + block range
chunk ID 在相同 release 下幂等稳定
```

无 H1、标题前导语、只有低级标题等情况进入 Golden Fixture。

### 14.4 健康检查按能力拆分

禁止用单个固定 200 `/health` 表示所有能力健康。至少区分：

```text
live
state-db ready
object-store ready
search ready
answer ready
model-gateway ready
```

### 14.5 Model Gateway 作为可替换下游

LiteLLM 的核心价值是提供 OpenAI-compatible 多模型网关。生产可保留这一模式，但需：

- 锁版本和镜像 digest；
- Secret Manager 注入；
- provider/model routing 版本化；
- retry/rate-limit/circuit breaker；
- token/cost tracing；
- Answer 失败不得影响 Search readiness。

## 15. 对现有方案不需要改变的部分

本项目没有推翻以下核心方向，反而进一步证明这些设计必要：

- PostgreSQL 是业务状态真相；
- S3/MinIO 保存不可变 Snapshot/Artifact；
- Crawl4AI 只是 Worker 内执行能力；
- `max_depth/max_pages` 不是历史完整性证明；
- Browser 只应按证据升级；
- Document Version append-only；
- Chunk/Embedding/Search 都是 Projection；
- Index 必须有 Release + Manifest；
- Web 长任务不能依赖进程内状态；
- Search 与 Generation 必须解耦。

## 16. 推荐验收用例补充

基于该项目暴露的问题，生产方案应新增以下测试：

### Chunker

- Markdown 无 H1 但有正文，必须至少产生一个 chunk；
- 第一个 H1 前有导语，导语不能丢；
- 只有 H2/H3 时不能为空；
- 超长代码块/表格不能静默破坏结构；
- 重复 REINDEX 在同 release 下产生相同 chunk ID。

### Search / Answer

- 无 LLM API Key 时 Search API 仍可启动并查询；
- Model Gateway 故障时 Search readiness 仍为 ready；
- Answer 返回错误必须是结构化错误，不允许 200 + `Error: ...` 文本伪装成功；
- Query Trace 可复现相同 retrieval profile 下的候选命中；
- 每条引用能回到 chunk、document_version、source URL 和 block range。

### Web

- UI 中 collection/index、top_k、filter、temperature 等值必须进入后端请求并持久化；
- Query Trace 展示内容与实际请求参数一致；
- 服务端 streaming 与假字符动画能够被测试区分。

### Deployment

- LiteLLM/LLM 全部宕机时抓取、Accepted Document、Chunk Projection、Search 不受影响；
- Search Backend 故障不丢失源文档，只形成可重试 Projection Job；
- 容器镜像均锁定版本/digest；
- Browser Worker OOM 后 durable task 可重新 lease。

## 17. 最终评价

RAG_with_crawl4AI 的价值在于展示了一个最小但完整的 RAG 闭环：Crawl4AI 负责网页转 Markdown，标题层级用于分块，Sentence Transformer 负责 embedding，Chroma/HNSW 负责向量召回，FastAPI 暴露检索与生成接口，LiteLLM 隔离模型供应商，Streamlit 完成交互。

但它仍是单机/教学式状态模型：BFS frontier 和 visited 在内存，抓取以 Browser 为主，Sitemap 无完整覆盖语义，Chunk ID 每次从 0 开始，`collection.add()` 没有版本生命周期，Embedding/索引没有业务 Release，Retrieval 直接拼字符串，Search 被 LLM Secret 耦合，健康检查不验证依赖，LLM 错误可能伪装成正常答案，UI 参数和实际请求存在脱节。

对 1000 个技术博客知识库，正确吸收方式不是直接放大该项目，而是保留它的组件边界，再生产化为：

**Durable Discovery/Fetch/IR 主链路 + Accepted Document Version + Stable Chunk Projection + Versioned Embedding/Search Index + Index Manifest + Hybrid Retrieval + Retrieval Profile + Context Builder + Query Trace + 独立 Search API + 独立 Answer API + 可替换 Model Gateway。**

这样抓取、Markdown、索引、检索、RAG 和模型可以分别升级、回滚和降级，同时保证每一步都有稳定 identity、版本、证据、幂等和可审计 lineage。