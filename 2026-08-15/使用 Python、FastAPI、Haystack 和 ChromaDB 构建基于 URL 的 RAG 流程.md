# 使用 Python、FastAPI、Haystack 和 ChromaDB 构建基于 URL 的 RAG 流程

## 1. 调研对象

- 编号：7
- 索引名称：使用 Python、FastAPI、Haystack 和 ChromaDB 构建基于 URL 的 RAG 流程
- 原文标题：Building a RAG Pipeline with FastAPI, Haystack, and ChromaDB for URLs in Python
- 原文地址：https://medium.com/@pwaykos1/building-a-rag-pipeline-with-fastapi-haystack-and-chromadb-for-urls-in-python-631575f3888b
- 同内容页面：https://www.aihello.com/resources/blog/building-a-rag-pipeline-with-fastapi-haystack-and-chromadb-for-urls-in-python/
- 调研日期：2026-08-15
- 相关官方资料：
  - Haystack ChromaDocumentStore：https://docs.haystack.deepset.ai/docs/chromadocumentstore
  - Haystack ChromaEmbeddingRetriever：https://docs.haystack.deepset.ai/docs/chromaembeddingretriever
  - Haystack FileTypeRouter：https://docs.haystack.deepset.ai/docs/filetyperouter
  - Haystack DocumentSplitter：https://docs.haystack.deepset.ai/docs/documentsplitter
  - Haystack DocumentWriter：https://docs.haystack.deepset.ai/docs/documentwriter
  - Haystack OllamaDocumentEmbedder：https://docs.haystack.deepset.ai/docs/ollamadocumentembedder
  - Haystack OllamaTextEmbedder：https://docs.haystack.deepset.ai/docs/ollamatextembedder
  - Haystack OllamaGenerator：https://docs.haystack.deepset.ai/docs/ollamagenerator
  - Chroma Metadata Filtering：https://docs.trychroma.com/docs/querying-collections/metadata-filtering
  - Crawl4AI Markdown Generation：https://docs.crawl4ai.com/core/markdown-generation/

## 2. 结论

这篇文章最有价值的不是“FastAPI + Haystack + ChromaDB”这一组具体依赖，而是它把**单 URL 摄取、资源类型路由、索引流水线、向量检索和问答 API**串成了一个完整的可运行闭环。对博客知识库尤其有参考价值的是：用户可以提交任意 URL，系统自行判断网页还是文件，完成摄取后再对指定内容问答。

但文章是面向教程和单机 PoC 的实现，不能直接扩展到 1000 个技术博客的历史全量抓取与长期增量同步。主要生产问题包括：

1. URL 类型判断仍混合了 HEAD、URL 后缀和同步 `requests`，缺少 redirect 后最终响应、证据置信度、预算与 SSRF 边界。
2. FastAPI `async` endpoint 内调用同步 `requests`，会阻塞 event loop。
3. HTML 与文件分别进入 `url-index`、`file-index`，把“输入形态”错误地变成了长期检索边界。
4. 查询时仍以 URL 为外部业务标识，并可能再次识别 URL 类型；这会让已入库查询依赖源站当前状态。
5. “对某个 URL 提问”若只通过选 collection 实现，并不能保证只召回该 URL 的 chunk。
6. 固定 `2000 words + 500 overlap` 适合作为教程参数，不适合作为代码、表格、长技术文档和多语言语料的生产默认值。
7. Haystack Pipeline、DocumentStore、Retriever、Embedder、Generator 在函数内反复构造，会增加连接、模型与对象初始化开销。
8. 没有完整的 Document/Version/Chunk/Embedding 稳定身份、幂等写、重复策略、索引 generation 和回滚模型。
9. Chroma 本地持久化适合 PoC，但不能成为百万文档平台的事实层。
10. Celery + Redis 作为可选异步化示例是合理的教学方向，但若平台已有 durable task/outbox，再增加一套 Celery 任务真相会造成状态分裂。

因此当前博客知识库技术方案不应退回文章的架构，而应吸收文章的“单 URL RAG 闭环”，同时坚持 Snapshot First、Ingress-neutral Corpus、Stable Query Target、Hard Scope Filter、Compiled Query Runtime 和 One Durable Task System。

---

## 3. 文章的端到端实现链路

文章实现的逻辑可概括为：

```text
POST /ingest(url)
  -> identify_url(url)
      -> article / pdf / txt / md ...
  -> article: Crawl4AI 抓取
  -> file: requests 下载临时文件
  -> Haystack indexing pipeline
      -> converter
      -> cleaner
      -> splitter
      -> document embedder
      -> Chroma DocumentStore

POST /generate(question, url)
  -> 根据 URL 类型选择 url-index / file-index
  -> text embedder
  -> ChromaEmbeddingRetriever
  -> PromptBuilder
  -> OllamaGenerator
  -> answer
```

文章又给出 Celery + Redis 作为可选增强，把长时间 ingestion 从 FastAPI 请求中移到后台任务。

这个闭环在 PoC 中很清楚：**写路径负责把 URL 变成可检索向量，读路径负责把问题变成 query embedding，再检索上下文给 LLM。**

---

## 4. URL 类型判断：文章做对了什么，以及生产上还缺什么

文章通过 `identify_url(url)` 把 URL 分成网页文章与文件。其思路大致是：

1. 请求目标 URL；
2. 查看 Content-Type；
3. 查看 URL path 的扩展名；
4. 不确定时下载少量字节，用 `filetype` 猜测文件类型。

这个思路比只看 `.pdf`、`.txt` 后缀可靠，因为真实网络中经常存在：

```text
/download?id=123          -> application/pdf
/report.pdf               -> 302 -> HTML 登录页
/file                     -> Content-Disposition: attachment; filename=x.pdf
/api/article/1            -> application/json
```

但生产 Resource Probe 必须更严格。

### 4.1 HEAD 只是 Hint，不是真相

服务器可能：

- 不支持 HEAD；
- HEAD 与 GET 返回不同 Content-Type；
- CDN/WAF 对 HEAD 使用不同路径；
- HEAD 不返回完整 redirect 行为；
- 动态下载链接只有 GET 才能确认资源。

因此推荐：

```text
HEAD
 -> 若证据足够则结束
 -> 否则 small Range GET
 -> 再不确定则 bounded GET
```

每一步都记录状态码、最终 URL、Content-Type、Content-Disposition、prefix magic bytes、响应大小和冲突信息。

### 4.2 URL 后缀必须降级为弱证据

扩展名适合做 Route Hint，例如 `*.pdf` 可以让系统倾向 PDF Probe，但最终 parser 必须由最终响应证据决定。否则 `.pdf -> HTML` 会把 HTML 错送进 PDF parser，而无扩展名 PDF 又会漏掉。

### 4.3 Redirect 后最终响应最重要

文章教程没有把 Resolution 独立成一层。生产中应先建立：

```text
Observed URL
 -> Normalized Candidate
 -> Redirect / Wrapper / Meta Refresh
 -> Resolved Fetch Target
 -> Resource Probe
```

这样 URL alias、最终抓取地址、canonical identity 和 query scope 才能分开治理。

---

## 5. FastAPI：API 异步不等于内部调用自动非阻塞

文章使用 FastAPI `async def`，但部分下载逻辑调用同步 `requests.get()`。这在小规模测试里通常能工作，但在并发 API 中会阻塞 event loop：

```text
async endpoint
  -> requests.get()
  -> 当前 event-loop thread 等待网络 I/O
  -> 同进程其它协程被拖慢
```

生产建议：

- HTTP 使用 `httpx.AsyncClient` / `aiohttp`；
- Browser 使用独立 Browser Worker；
- PDF/OCR/Embedding 等长任务不在 API request 中同步完成；
- `POST /ingestions` 只创建 durable command，返回 `202 + ingestion_id`；
- 前端通过状态查询/SSE/WebSocket 观察阶段推进。

文章最后引入 Celery 的动机是对的：长任务不应阻塞 HTTP 请求。但具体实现应服从平台统一任务模型。

---

## 6. Crawl4AI 在这个流程中的正确定位

文章用 Crawl4AI 抓网页文章，再进入 RAG。这说明 Crawl4AI 很适合作为**网页到正文候选/Markdown 候选**的执行层。

Crawl4AI 官方文档明确区分 raw markdown 与经过内容过滤的 fit markdown。对博客知识库来说：

```text
Network Response / Rendered DOM
 -> Snapshot
 -> Extraction Candidate
    - Crawl4AI raw markdown
    - Crawl4AI cleaned HTML
    - Trafilatura
    - Readability
    - Site selector
 -> Quality Selection
 -> Canonical IR
 -> Canonical Markdown
```

不能把 `fit_markdown` 当成唯一事实，因为 pruning/BM25/LLM filter 都可能丢掉对未来查询有价值的内容。文章用 Crawl4AI 作为 RAG 前处理是合理的，但生产知识库还必须保留 Snapshot 与可重建 canonical 层。

---

## 7. Haystack Indexing Pipeline 的技术原理

文章的文件路径采用典型 Haystack indexing pipeline：

```text
FileTypeRouter
 -> PyPDFToDocument / TextFileToDocument / MarkdownToDocument
 -> DocumentJoiner
 -> DocumentCleaner
 -> DocumentSplitter
 -> OllamaDocumentEmbedder
 -> DocumentWriter(ChromaDocumentStore)
```

### 7.1 Router / Converter

`FileTypeRouter` 根据 MIME 将文件分发给不同 converter。这个分层是对的：输入类型不同，解析器天然不同。

但“解析路径不同”不等于“最终语料库不同”。正确抽象是：

```text
PDF  -> PDF parser -----┐
HTML -> HTML extractor -┤
MD   -> MD parser ------┼-> Canonical IR -> unified Chunk contract
TXT  -> decoder --------┘
```

### 7.2 Cleaner

Cleaner 用于统一空白、噪声和文本格式，但生产系统必须保证它是版本化 Projection，而不能直接覆盖原始事实。任何会改变正文语义的清洗都需要 release 和 fixture。

### 7.3 Splitter

文章使用：

```text
split_by = word
split_length = 2000
split_overlap = 500
```

技术上 overlap 的作用是降低语义在 chunk 边界被截断的概率，但代价也明显：

- 向量数量增加；
- embedding token 成本增加；
- 相邻 chunk 高重复；
- top-k 容易被同一段重复内容占满；
- 代码块/表格可能被从中间切断。

技术博客更适合 structure-aware chunk：heading/section 优先，段落、代码块、表格保持完整，过长 section 才 recursive fallback；长度用目标 embedding tokenizer 的 token 计算，而不是简单 word count。

---

## 8. Embedding：文档向量与查询向量必须处在同一语义空间

文章使用 `OllamaDocumentEmbedder(nomic-embed-text)` 写入文档向量，查询时使用 `OllamaTextEmbedder` 生成 query embedding，再交给 `ChromaEmbeddingRetriever`。

核心原理是：

```text
E_doc(chunk)  ∈ R^d
E_query(q)    ∈ R^d
similarity(E_query, E_doc)
```

文档和 query 必须使用兼容的 embedding 模型、维度、归一化和距离度量，否则向量距离没有可比意义。

因此生产系统必须显式版本化：

```text
embedding_release_id
model_digest
dimension
normalization
metric
max_tokens
```

向量写入前校验 dimension、NaN/Inf、normalization 和 target generation；embedding 升级时构建新 generation，而不是原地混写不同模型的向量。

---

## 9. Chroma：适合 PoC，但最重要的启发是 metadata filter

文章为文件和 URL 建两个 collection：

```text
file-index
url-index
```

这在教程里很直观，但会产生两个长期问题。

### 9.1 输入类型不应该成为业务 Corpus 边界

用户真正关心的是：

- 全库；
- 某个 Source；
- 某篇 Document；
- 某个 Version；
- 某个 Tag；
- 某种语言。

并不关心文档最初是 HTML 还是 PDF。因此生产索引应统一 payload：

```text
source_id
document_id
document_version_id
chunk_id
content_kind
ingress_origin
language
tags
```

HTML/PDF/MD/TXT 进入同一逻辑 corpus。

### 9.2 “选择 collection”不能替代“只查这篇文章”

如果 URL A 和 URL B 都写入 `url-index`，查询 A 时只选择 `url-index`，B 的 chunk 仍然可能被召回。

Chroma 本身支持 metadata filtering，因此即使在开发 Adapter 中，也必须执行类似：

```text
where = {"document_id": requested_document_id}
```

而不是只切 collection。

这条原则应提升为跨向量引擎契约：**任何 VectorEngineAdapter 如果不能在召回阶段执行 hard scope filter，就不能用于 DOCUMENT/URL scope 的生产查询。**

---

## 10. DocumentWriter 与幂等：教程里缺失的关键层

Haystack `DocumentWriter` 支持重复策略，例如 SKIP、OVERWRITE、FAIL。教程没有把这个问题提升为平台 identity 设计，因此长期增量时容易出现：

- 相同 URL 重复摄取；
- 内容未变化却重复 embedding；
- chunk 顺序改变导致全部 ID 变化；
- 新模型向量覆盖旧模型向量；
- retry 后重复插入。

生产平台不能依赖 DocumentStore 默认 duplicate policy，而要先计算稳定身份：

```text
document_version_id
chunk_id = hash(document_version_id + chunk_release + logical_block_range + text_hash)
embedding_id = hash(chunk_id + embedding_release_id)
```

然后用 deterministic ID + upsert。DocumentWriter/VectorStore 的 duplicate policy 只是执行细节，不是业务身份真相。

---

## 11. Query Pipeline：文章链路正确，但对象生命周期不适合高 QPS

文章的查询链路是：

```text
question
 -> OllamaTextEmbedder
 -> ChromaEmbeddingRetriever
 -> PromptBuilder
 -> OllamaGenerator
```

这是标准 RAG 执行图。

问题在于教程函数中会反复创建：

- Pipeline；
- ChromaDocumentStore；
- Retriever；
- Text Embedder；
- PromptBuilder；
- Generator。

在生产 Query Service 中，这些对象应按 Retrieval Release 长生命周期复用：

```text
service startup / release activation
 -> init store clients
 -> init model clients
 -> compile pipeline graph
 -> warmup
 -> READY

request
 -> inject query + hard scope filter + top_k
 -> execute cached pipeline
```

这样可以减少连接抖动、模型冷启动、对象构建和版本漂移。

---

## 12. URL 作为查询参数的隐藏问题：URL 是 locator，不是稳定 Query Target

文章 `/generate` 继续接收 URL，这是很自然的用户体验，但服务端不应该在每次问答时重新访问该 URL 判断类型。

否则会出现：

- 源站已下线，历史知识却无法查询；
- URL 现在从 PDF 改成 HTML，历史索引身份发生漂移；
- query path 拥有任意公网访问能力，扩大 SSRF 面；
- URL redirect/canonical 变化导致查不到旧文档；
- 同一个 document 有多个 alias 时行为不一致。

正确链路：

```text
request URL
 -> local normalization
 -> URL Alias / Canonical Mapping
 -> document_id
 -> active document_version_id
 -> hard metadata filter
 -> retrieval
```

若 alias 不存在，应返回 `DOCUMENT_NOT_FOUND`，而不是 Query API 暗中去抓 URL。

若产品希望“粘贴 URL 后立即问”，也应拆成显式两阶段：

```text
POST /ingestions -> 202 ingestion_id
GET /ingestions/{id} -> READY / PARTIAL_READY
POST /ask(scope=document_id)
```

Web 可以把两阶段组合成一个交互，但后端语义不能混在一起。

---

## 13. Prompt 不能承担检索隔离

文章 Prompt 大致要求“根据提供的 documents 回答，不知道就说不知道”。这个方向可以降低幻觉，但它只作用于**已经召回的 context**。

如果 Retriever 错误地把 B 文档召回给 A，Prompt 无法知道 B 不属于 A。

因此 Scope 必须发生在：

```text
BM25 retrieval filter
Vector retrieval filter
```

而不是：

```text
prompt: only answer from URL A
```

这也是为什么博客知识库必须把 `Retrieval Scope` 做成一等公民，并测 `ScopeLeakRate`。

---

## 14. Celery + Redis：为什么不能简单照搬

文章把 Celery 作为可选异步增强，解决“摄取时间长，HTTP 请求不能一直等待”的问题，这个动机完全正确。

但 1000 站平台还要处理：

- discovery；
- backfill；
- incremental；
- PDF；
- Browser；
- extraction；
- embedding；
- indexing；
- retry；
- cancellation；
- partial success；
- lease/fencing；
- Web stage timeline。

如果平台已有 PostgreSQL Task + Transactional Outbox + Redis Streams，再引入 Celery 作为另一套业务任务状态，会出现：

```text
DB 说 RUNNING
Celery 说 RETRY
Redis broker message 已丢/已 ack
Web 无法判断哪个是真相
```

所以文章的“异步化”应吸收为原则，不应照搬为第二任务系统。

---

## 15. 对 1000 博客知识库的可落地改造

把文章的 PoC 升级为生产闭环，推荐：

```text
Manual URL
 -> POST /ingestions
 -> Normalization
 -> Resolution
 -> Resource Probe
 -> HTTP / Browser / File Fetch
 -> immutable Snapshot
 -> extractor candidates
 -> Canonical IR
 -> Document Version
 -> Canonical Markdown
 -> structure-aware Chunk
 -> Embedding Release
 -> BM25 / Vector Generation
 -> READY / PARTIAL_READY
 -> URL alias -> Document scope
 -> Hybrid Retrieval
 -> RAG Answer
```

### 15.1 单 URL 摄取与 Source Crawl 共用事实层

`MANUAL_URL` 只表示 ingress origin，不应该有另一套 storage/indexing pipeline。这样用户手工提交的 URL、1000 站自动抓取的文章、PDF 文件都能共享版本、Chunk、Embedding 和 Search。

### 15.2 明确 `ready_for`

单一 `READY` 对用户体验太粗。建议 Web/API 可推导：

```text
ready_for.canonical = true/false
ready_for.markdown  = true/false
ready_for.fulltext  = true/false
ready_for.semantic  = true/false
ready_for.ask       = true/false
```

例如 Canonical Markdown 已生成但 embedding backlog 较长时，可以先开放 Markdown 与全文检索，而不是整个 ingestion 都显示“未完成”。

### 15.3 VectorEngineAdapter 必须声明能力

建议统一接口至少包括：

```text
upsert(records, generation_id)
query(query_vector, scope_filter, top_k)
delete_generation(generation_id)
health()
capabilities()
```

`capabilities()` 至少声明：

```text
hard_metadata_filter
filter_operators
max_filter_terms
hybrid_support
batch_upsert
atomic_alias_or_generation_switch
```

DOCUMENT/URL scope 的生产 Adapter 必须支持 hard metadata filter；Chroma PoC Adapter 也要用 metadata `where` 实现同样语义，从开发阶段就避免“换引擎后才发现 scope 契约不同”。

---

## 16. 对当前技术方案的具体优化判断

当前《博客知识库技术方案.md》已经覆盖并且优于文章的多数关键点：

- 已有统一 Document Ingress；
- 已有 Manual Ingestion API；
- 已有 Resource Probe；
- 已明确 URL 后缀只是弱证据；
- 已有 Snapshot / Canonical IR / Markdown Projection；
- 已拒绝 `url-index/file-index` 长期分裂；
- 已有 Stable Query Target；
- 已有 hard Retrieval Scope；
- 已有 Structure-aware Chunk；
- 已有 Embedding Release / Index Generation；
- 已有 Compiled Query Runtime；
- 已明确 Query API 不隐式抓远程 URL；
- 已明确不增加 Celery 作为第二套平台任务真相。

这次文章仍带来三个值得补强的点：

1. **把“Query 不隐式摄取”提升为核心原则**：用户提供 URL 可以是 UI 输入，但 read path 必须先解析成内部稳定身份；不存在时显式要求 ingestion。
2. **增加 VectorEngineAdapter 的 hard-scope capability contract**：不仅架构上说“要过滤”，还要求每个向量引擎 Adapter 声明并通过 scope conformance test。
3. **把单 URL RAG 的 readiness 变成面向用户的能力状态**：canonical、markdown、fulltext、semantic、ask 分别可用，避免 embedding/LLM backlog 阻塞其它功能。

---

## 17. 建议的验收用例

除现有测试外，针对这篇文章暴露的问题应增加：

```text
1. ingest HTML URL A + HTML URL B
   ask scope=A
   -> 任何阶段不得召回 B

2. ingest PDF A + HTML B
   ALL scope
   -> 可跨 content-kind 召回

3. query URL A 时源站已 404
   -> 仍通过本地 alias 查询历史 Active Version

4. query 未入库 URL
   -> DOCUMENT_NOT_FOUND
   -> 不发出公网 HEAD/GET

5. Chroma dev adapter DOCUMENT scope
   -> 必须使用 metadata where 过滤

6. Vector adapter 不支持 hard filter
   -> capability check 阻止其激活为 scoped retrieval backend

7. 相同 URL + 相同 Idempotency-Key 重复 ingest
   -> 不创建重复 Run/Document/Embedding

8. 内容不变但重新抓取
   -> 不重复创建 Document Version

9. Embedding PENDING
   -> canonical/markdown/fulltext readiness 可独立 READY

10. Query Service 连续请求
    -> Pipeline/DocumentStore/Model client 不按请求重建
```

---

## 18. 最终判断

文章非常适合用作“URL -> RAG”最小闭环参考，但它的最佳用途是帮助明确产品行为和组件责任，而不是直接作为 1000 站生产架构。

应保留的思想：

- FastAPI 暴露 ingestion / generation 能力；
- Crawl4AI 负责网页抓取；
- 不同文件类型经过不同 converter；
- Haystack 用 Pipeline 编排 indexing 和 query；
- 文档 embedding 与 query embedding 对齐；
- vector retrieval 后再生成答案；
- 长 ingestion 应异步执行。

必须升级的部分：

- Resource Probe 替代简单 URL 类型判断；
- async HTTP 替代 event-loop 中同步 requests；
- Snapshot/IR/Version 替代“一抓即写向量库”；
- ingress-neutral corpus 替代 `url-index/file-index`；
- stable document identity 替代 query 时重新识别 URL；
- hard metadata scope 替代 collection 粗隔离；
- structure-aware chunk 替代固定 2000/500；
- deterministic ID + release 替代默认 duplicate behavior；
- compiled query runtime 替代每请求创建 Pipeline；
- durable task/outbox 替代第二套 Celery 业务状态；
- production vector/search generation 替代单机 Chroma 作为唯一存储。

对当前博客知识库方案而言，这篇文章不是推翻现有设计，而是进一步强化“**显式摄取、稳定身份、统一语料、硬 Scope、可复用 RAG Runtime**”这一条从单 URL 用户体验到百万文档生产系统的连续架构路线。