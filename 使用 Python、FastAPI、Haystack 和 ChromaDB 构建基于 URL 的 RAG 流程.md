# 使用 Python、FastAPI、Haystack 和 ChromaDB 构建基于 URL 的 RAG 流程：实现细节与技术原理分析

## 1. 调研对象

- 编号：7
- 索引名称：使用 Python、FastAPI、Haystack 和 ChromaDB 构建基于 URL 的 RAG 流程
- 原文标题：Building a RAG Pipeline with FastAPI, Haystack, and ChromaDB for URLs in Python
- 原文地址：https://medium.com/@pwaykos1/building-a-rag-pipeline-with-fastapi-haystack-and-chromadb-for-urls-in-python-631575f3888b
- 可访问镜像：https://www.aihello.com/resources/blog/building-a-rag-pipeline-with-fastapi-haystack-and-chromadb-for-urls-in-python/
- 调研日期：2026-08-15

本文的价值不在于它给出了一个可以直接支撑 1000 个站点的生产架构，而在于它把“任意 URL 输入”拆成了一个很清晰的产品路径：**URL 类型识别 → 抓取/文件解析 → 文档入库 → Embedding → 检索 → LLM 生成 → 异步任务化**。对博客知识库方案最有启发的部分是“按资源类型路由”和“把手工 URL 导入、RAG 查询做成 API 能力”。

---

## 2. 原方案的端到端流程

文章实现的逻辑可以抽象为：

```text
Client
  │
  ├─ POST /ingest(url)
  │      │
  │      ▼
  │   URL 类型判断
  │      ├─ HTML/article ──> Crawl4AI AsyncWebCrawler
  │      │                         │
  │      │                         ▼
  │      │                  LLMExtractionStrategy
  │      │                         │
  │      └─ PDF/TXT/MD ──> 文件解析器
  │                                │
  │                                ▼
  │                       Haystack Document
  │                                │
  │                                ▼
  │                         Embedding Pipeline
  │                                │
  │                                ▼
  │                         ChromaDocumentStore
  │
  └─ POST /generate(url, question)
         │
         ▼
      查询 Embedding
         │
         ▼
      Retriever Top-K
         │
         ▼
      Prompt + Context
         │
         ▼
      Ollama Chat Model
         │
         ▼
      Answer
```

文章还给出一个可选的异步版本：把 `/ingest` 的长任务交给 Celery Worker，Redis 作为 broker，从而避免 API 请求线程一直等待抓取、解析和索引完成。

这套设计本质上已经形成两个平面：

1. **Ingestion Plane**：负责把 URL 转成可检索文档；
2. **Query Plane**：负责把问题转成向量、召回文档并生成答案。

这个拆分方向是正确的，但文章实现仍然是教程级单机 RAG，缺少大型知识库所需的事实层、版本、增量同步、幂等、覆盖率和运营能力。

---

## 3. URL 类型识别：真正值得吸收的第一层路由

文章首先把 URL 分成两类：

```text
1. Web page / article
2. File: PDF / TXT / MD / ...
```

其实现思路是利用 HTTP 请求结果、MIME type 和文件扩展名判断资源类型。这个思想非常重要，因为“URL”不是“HTML 页面”的同义词。技术博客经常直接链接：

- PDF 白皮书；
- Markdown 原文；
- `.txt` / `llms.txt`；
- JSON API；
- RSS / Atom；
- 附件、代码压缩包；
- 没有扩展名但响应 `application/pdf` 的下载地址；
- 有 `.html` 外观但经过重定向后实际返回 PDF 的地址。

### 3.1 教程实现的问题

若只根据 URL suffix 判断会出现大量误分类，例如：

```text
/download?id=123        -> 实际是 PDF
/report.pdf?token=...   -> suffix 解析可能错误
/article/123            -> HTML
/api/post/123           -> JSON
/file.pdf               -> 302 -> HTML 登录页
```

只发 `HEAD` 也不可靠：有些服务器不支持 HEAD，或者 HEAD 与 GET 返回的 `Content-Type` 不一致。

### 3.2 生产级改造：Resource Probe

博客知识库应增加一个轻量的 **Resource Probe** 阶段，位置在 URL Resolution 之后、正式 Fetch Route 之前：

```text
Observed URL
 -> Normalize
 -> Redirect / Wrapper Resolution
 -> Resource Probe
 -> Content-kind Classification
 -> Fetch Route
```

建议识别：

```text
HTML_DOCUMENT
PDF_DOCUMENT
TEXT_DOCUMENT
MARKDOWN_DOCUMENT
JSON_API
XML_FEED
SITEMAP
IMAGE_ASSET
ARCHIVE_FILE
BINARY_UNSUPPORTED
UNKNOWN
```

判断证据优先级：

```text
1. Redirect 后最终响应
2. Content-Type
3. Magic bytes / 文件签名
4. Content-Disposition filename
5. 站点 Profile 规则
6. URL extension（弱证据）
```

Probe 结果应持久化，不能只是 Worker 内临时变量：

```text
resource_probe_attempt
  id
  url_id
  final_url
  http_status
  content_type_header
  detected_mime
  content_kind
  confidence
  content_length
  etag
  last_modified
  accept_ranges
  content_disposition
  prefix_hash
  route_hint
  evidence_json
  created_at
```

这样不仅可以给 PDF/HTML 选择不同 extractor，也能在增量同步时用 ETag、Last-Modified、长度变化等低成本信号判断是否需要完整抓取。

---

## 4. Crawl4AI 抓网页：文章思路正确，但 API 与生产用法需要升级

文章使用 `AsyncWebCrawler` 抓取网页，并通过 `LLMExtractionStrategy` 把网页转换为更适合 RAG 的文本。文章示例中的核心思想是：

```text
AsyncWebCrawler
 + LLMExtractionStrategy
 + local Ollama
 + instruction
 -> extracted content
```

文章使用本地 Ollama 的意义是避免把网页正文发送给外部模型，也方便本地实验。

### 4.1 当前 Crawl4AI API 已发生变化

文章所描述的旧式 `LLMExtractionStrategy(provider=..., base_url=..., api_token=...)` 在当前 Crawl4AI 中已经不是推荐接口。官方在后续版本引入了统一 `LLMConfig`，并移除了旧参数。当前模式应类似：

```python
llm_strategy = LLMExtractionStrategy(
    llm_config=LLMConfig(
        provider="ollama/model-name",
        base_url="http://ollama:11434"
    ),
    schema=..., 
    instruction="..."
)

config = CrawlerRunConfig(
    extraction_strategy=llm_strategy
)

result = await crawler.arun(url=url, config=config)
```

这说明生产系统不能把第三方库当前参数直接写死在 Site Profile 中，而要通过 `runtime_artifact_release_id`、`extractor_release_id` 绑定具体 Crawl4AI/Playwright 版本。

### 4.2 为什么不能把 LLM Extraction 当 canonical 正文

教程把 LLM extraction 放在网页摄取主路径上，对单篇问答体验很好，但不适合百万篇历史文章：

- LLM 输出有随机性；
- Prompt/模型升级会改变结果；
- 长文会产生 chunk token 成本；
- 模型可能遗漏代码、表格或细节；
- LLM 可能改写原文而不是忠实抽取；
- 大规模回填速度受模型吞吐限制；
- 模型不可用时会阻塞 Source Sync。

因此生产方案应坚持：

```text
Snapshot
 -> Deterministic Extraction Candidate
 -> Canonical IR
 -> Canonical Markdown

Canonical IR / Markdown
 -> optional LLM Extraction Projection
 -> Summary / Entity / QA View
```

LLMExtractionStrategy 可以作为：

1. 结构未知站点的低置信度 fallback；
2. Schema 生成辅助；
3. 非 canonical 的摘要/实体/知识图谱 Projection；
4. 少量复杂页面的人工审核候选。

不能让它决定 Document Identity 或正文版本。

---

## 5. 文件类 URL：应从“特殊分支”升级为统一 Document Ingress

文章支持 PDF、TXT 等文件，并对 URL 与文件分开处理。这个思路适合扩展成统一的 `Document Ingress`：

```text
Ingress
  origin = SOURCE_CRAWL | MANUAL_URL | FILE_UPLOAD | ARCHIVE_RECOVERY
  locator
  detected_content_kind
  source_id(optional)
  requested_scope
```

再根据 `content_kind` 路由：

```text
HTML_DOCUMENT     -> HTML extractor stack
PDF_DOCUMENT      -> PDF extractor stack
TEXT_DOCUMENT     -> text decoder
MARKDOWN_DOCUMENT -> markdown parser
JSON_API          -> JSON/schema extractor
XML_FEED          -> feed parser
```

### 5.1 PDF 处理原则

博客知识库中的 PDF 不能只做 `pypdf.extract_text()` 后直接 Embedding。至少需要保存：

```text
page_no
text blocks
heading candidate
code/table signal
image refs
page bbox（若解析器支持）
source file hash
parser release
OCR flag
```

文本型 PDF 优先确定性解析；扫描型 PDF 才进入 OCR fallback。OCR 结果是 Extraction Candidate，不覆盖原始 PDF Snapshot。

### 5.2 Markdown/TXT

对 `.md` 不应该经过 HTML Readability，而应保留：

- fenced code；
- heading tree；
- tables；
- links；
- frontmatter；
- 原始换行语义。

这与技术博客知识库的 structure-aware chunking 直接相关。

---

## 6. Haystack + Chroma：教程的 RAG 机制

文章把经过处理的文档交给 Haystack，并使用 ChromaDB 保存 Embedding。RAG 需要两个模型：

```text
Embedding model -> 把文档和 query 映射到向量空间
Chat model      -> 利用召回的 context 生成答案
```

文章示例使用：

- Chat：`llama3.2`；
- Embedding：`nomic-embed-text`；
- Vector Store：ChromaDB；
- Orchestration：Haystack。

正确的查询链路是：

```text
question
 -> query embedder
 -> ChromaEmbeddingRetriever / vector query
 -> top_k documents
 -> prompt builder
 -> chat generator
 -> answer
```

Haystack 的 Chroma retriever 支持 metadata filters，这一点对博客知识库尤其重要，因为“向某个 URL 提问”不应该在整个知识库中无约束召回。

### 6.1 应吸收：Retrieval Scope

文章 `/generate` 同时接收 URL 和 question，实际上隐含了一个重要概念：**查询作用域**。

生产系统应显式建模：

```text
retrieval_scope
  ALL
  SOURCE(source_id)
  DOCUMENT(document_id)
  DOCUMENT_VERSION(document_version_id)
  URL(resolved_url)
  TAG(tag)
```

查询时把 scope 转成 metadata filter：

```text
WHERE source_id = ...
WHERE document_id = ...
WHERE version_id = ...
```

Web 管理台可以直接提供：

- 向单篇文章提问；
- 向一个博客站点提问；
- 向全部知识库提问；
- 在某个版本时间点做检索回放。

### 6.2 为什么不建议把 Chroma 直接作为生产 Truth

Chroma 很适合教程、PoC 和中小规模独立 RAG，但对于本项目的事实层，应保持：

```text
PostgreSQL + Object Storage = Truth
Vector DB                  = Projection
```

原因不是 Chroma “不能用”，而是：

- Embedding 可以重算；
- Vector index 参数会升级；
- Chunk release 会变化；
- 模型维度会变化；
- 需要双 Generation 构建和切换；
- 删除/更新必须与 Document Version 精确关联；
- 生产需要 BM25 + Vector + RRF，而不是仅向量召回。

因此 Chroma 可以保留为开发/单机 Adapter：

```text
VectorEngineAdapter
  ├─ ChromaAdapter      # dev / local
  ├─ PgVectorAdapter    # initial production option
  ├─ OpenSearchKNN
  ├─ QdrantAdapter
  └─ MilvusAdapter
```

但 `document_version_id / chunk_id / embedding_release_id` 的稳定身份必须由平台事实层维护。

---

## 7. Embedding 的核心问题不是“调用模型”，而是稳定身份和版本

文章的教程流程通常是“抓到文档 → 切块 → embedding → 写入 Chroma”。对于增量知识库，必须进一步回答：

1. 同一篇文章没变化，为什么又生成了一批向量？
2. 文章只增加一段，能否复用旧 Chunk？
3. 模型从 768 维换到 1024 维时旧向量怎么办？
4. 删除文章后旧向量如何准确 tombstone？
5. 查询是否误召回旧版本 Chunk？

建议稳定键：

```text
chunk_id = hash(
  document_version_id
  + chunk_release_id
  + logical_block_range
  + normalized_text_hash
)

embedding_id = hash(
  chunk_id
  + embedding_release_id
)
```

Vector Store 写入使用 deterministic ID + upsert。Chroma 官方同样支持 `upsert`，但业务主键不能由 Chroma 自己生成随机 UUID，否则重放和去重很难保证一致。

---

## 8. `/ingest` API：应增加为博客知识库的 Web 管理能力

这是本次调研对现有方案最明确的功能优化。

现有系统主要围绕 Source 的自动发现、全量回填、增量同步运行，但运营过程中一定会出现：

- 临时补一篇遗漏文章；
- 用户粘贴一个 URL 要立即入库；
- 新站正式建 Source 之前先验证单 URL；
- 一个站点的某个 PDF 需要补采；
- 对失败 URL 手工重试；
- 外部系统通过 API 推送待收录 URL。

因此建议增加统一 Manual Ingestion API：

```http
POST /api/v1/ingestions
Idempotency-Key: <client-generated-key>

{
  "url": "https://example.com/post/1",
  "source_id": "optional",
  "mode": "AUTO",
  "priority": "NORMAL",
  "requested_projections": ["CLEAN_MARKDOWN", "FULLTEXT", "EMBEDDING"]
}
```

返回：

```http
HTTP/1.1 202 Accepted

{
  "ingestion_id": "...",
  "run_id": "...",
  "state": "QUEUED"
}
```

状态接口：

```http
GET /api/v1/ingestions/{id}
```

返回阶段状态：

```text
QUEUED
PROBING
FETCHING
SNAPSHOT_READY
EXTRACTING
CANONICAL_READY
MARKDOWN_READY
INDEXING
READY
PARTIAL_READY
FAILED
```

### 8.1 为什么必须返回 202，而不是等待完成

抓取网页、启动 Browser、下载 PDF、生成 Embedding 都可能是长任务。FastAPI 官方也明确建议，重计算/多进程后台任务应由外部队列/Worker 承担。

本项目已经采用 PostgreSQL + Outbox + Redis Streams + Worker，因此**不需要为了吸收文章经验而改成 Celery**。正确做法是吸收“API 与后台任务解耦”的思想，并保留现有 durable scheduler。

---

## 9. Celery + Redis：文章方案的价值和不足

文章可选章节把 ingest 改为 Celery 异步任务：

```text
FastAPI
 -> enqueue task
 -> Redis broker
 -> Celery Worker
 -> crawl / parse / embed
```

它解决了三个问题：

1. HTTP 请求不需要等待；
2. Worker 可以独立扩容；
3. 长任务失败后可以重试。

但对于我们的系统，直接采用 Celery 仍缺少关键能力：

- 业务状态与消息发布的事务一致；
- per-URL lease/fencing；
- Provider cursor；
- Run DAG；
- 多阶段 partial success；
- `Retry-After` 持久化；
- 消费者崩溃后的业务级重放证据；
- Source/domain 的全局限流；
- Projection 与 Source Sync 优先级隔离。

因此结论是：

> 不引入 Celery 作为新的第二套任务系统，继续使用 PostgreSQL durable state + Transactional Outbox + Redis Streams；在 API 设计上采用 Celery 教程体现的“异步接受、任务状态查询、Worker 独立扩容”模型。

---

## 10. Ingestion 状态与 Projection Readiness

文章把“入库成功”看成一个动作，但生产系统应区分“事实已经抓到”和“所有检索 Projection 已经准备好”。

建议为每个 `document_version` 维护 readiness：

```text
canonical_state = READY | FAILED
markdown_state  = READY | PENDING | FAILED
fulltext_state  = READY | PENDING | FAILED
chunk_state     = READY | PENDING | FAILED
embedding_state = READY | PENDING | FAILED
vector_state    = READY | PENDING | FAILED
summary_state   = READY | PENDING | FAILED
```

于是 API 可以出现合理的：

```text
PARTIAL_READY
```

例如文章已经抓取并生成 Markdown，但 Embedding 服务积压，此时 Source Sync 仍然成功，全文检索可以先用，向量检索稍后补齐。

这与现有“Source Sync 与 AI 解耦”原则完全一致，但应在 Web 和 API 上显式暴露。

---

## 11. `/generate` API：不要直接和抓取耦合

教程的 `/generate(url, question)` 会再次判断 URL 类型，然后走对应 query 逻辑。生产系统应避免 Query API 隐式触发昂贵的同步抓取。

建议拆成：

```http
POST /api/v1/ingestions     # 摄取
GET  /api/v1/ingestions/{id}
POST /api/v1/search         # 检索
POST /api/v1/ask            # RAG 问答
```

`/ask`：

```json
{
  "question": "这篇文章如何处理异步任务？",
  "scope": {
    "type": "DOCUMENT",
    "document_id": "..."
  },
  "retrieval_release": "active",
  "max_context_tokens": 12000
}
```

如果目标文档尚未入库，返回明确状态：

```text
DOCUMENT_NOT_READY
```

客户端可先调用 ingestion，而不是 Query API 在内部偷偷抓取网络资源。这能避免 SSRF、超时、重复抓取和不可审计的副作用。

---

## 12. 安全：教程中最需要补齐的部分

“任意 URL 输入”是 SSRF 的典型入口。生产系统必须在 Manual Ingestion API 前增加安全边界：

```text
parse URL
 -> scheme allowlist(http/https)
 -> DNS resolve
 -> block loopback/private/link-local/metadata IP
 -> connect
 -> each redirect re-check
 -> Browser navigation re-check
```

必须阻止：

```text
http://127.0.0.1
http://localhost
http://169.254.169.254
http://10.0.0.0/8
file://...
gopher://...
```

同时：

- 限制响应大小；
- 限制 PDF/附件大小；
- 限制 redirect 次数；
- 限制 Browser wall-clock；
- 限制解压比，防止压缩炸弹；
- 不自动绕过登录、WAF、验证码；
- API 需要 RBAC、审计、请求配额。

手工 URL 导入不能绕开 Source Policy。

---

## 13. 增量同步：文章没有解决，但必须补在同一模型中

文章每次 `/ingest` 都更像“重新处理 URL”。博客知识库则需要低成本判变：

```text
Resource Probe
 -> ETag / Last-Modified / Content-Length
 -> Conditional GET
 -> 304: stop
 -> 200: compare body hash
 -> unchanged: stop
 -> changed: Snapshot -> Extract -> New Version
```

对于文件 URL 同样适用：

```text
PDF ETag / Last-Modified / length
 -> conditional request
 -> file hash
 -> parser replay only when needed
```

这意味着 Resource Probe 并不是单纯的文件类型判断器，而可以成为增量同步前置的 metadata probe。

---

## 14. 对 Web 管理台的直接优化

建议增加“手工摄取 / 单 URL 调试”页面：

### 14.1 输入区

- URL；
- 可选 Source；
- AUTO / HTTP_ONLY / BROWSER_ALLOWED；
- 是否生成 Embedding；
- 是否强制刷新；
- Idempotency key 自动生成。

### 14.2 执行时间线

```text
Normalize
Resolution
Resource Probe
Route Match
Fetch
Snapshot
Extraction
Quality
Identity
Version
Markdown
Chunk
Embedding
Index
```

每一步展示：

- 状态；
- 耗时；
- Release；
- 输入/输出 hash；
- 错误；
- 重试次数；
- evidence。

### 14.3 结果区

- detected MIME/content-kind；
- 原始 Snapshot；
- Canonical Markdown；
- Extraction Candidate 对比；
- Chunk；
- scoped search；
- “对本篇提问”。

这样文章中的 FastAPI `/ingest` + `/generate` 被升级成真正可运营的知识库调试工具。

---

## 15. Route Match 与 MIME Probe 的关系

现有方案已经有 per-URL `CrawlerRunConfig` / Route Match Release，例如：

```text
PDF URL     -> PDF config
Article URL -> article config
Dynamic Hub -> interaction config
```

仅依赖 URL matcher 有一个隐患：URL pattern 是先验，MIME Probe 是运行时证据。

建议最终路由采用两级：

```text
Route Hint（URL/Profile 静态规则）
       +
Resource Probe（响应证据）
       ↓
Resolved Route
```

冲突示例：

```text
URL matcher: *.pdf
Probe: text/html
```

可能原因是：

- 登录页；
- WAF；
- 失效下载；
- HTML 错误页。

此时不能按 PDF parser 强行处理，应记录：

```text
route_hint = PDF
observed_content_kind = HTML_DOCUMENT
route_conflict = true
```

并进入质量/错误策略。

---

## 16. 测试与 Contract Test 增补

根据本次调研，现有 Runtime Contract Test 应新增：

```text
无扩展名 PDF URL -> 正确识别 PDF
.pdf URL -> 302 -> HTML -> 不进入 PDF parser
HEAD 不支持 -> Range GET fallback
Content-Type 错误 -> magic bytes 能纠正
超大文件 -> size budget 拦截
HTML / PDF / TXT / MD 相同 ingest API 路由正确
重复 Idempotency-Key -> 不重复创建 Run
重复 URL 未变化 -> 不重复建 Document Version
Embedding 延迟 -> ingestion 返回 PARTIAL_READY
按 document_id scope 查询 -> 不召回其他文档
旧 version vector -> ACTIVE 查询不召回
LLM extraction 失败 -> canonical extraction 不受影响
manual URL redirect -> 每跳 SSRF 校验
```

---

## 17. 对当前博客知识库方案的修改结论

本次调研不建议替换现有技术栈，而建议新增/强化以下能力：

### 高优先级

1. **Resource Probe / Content-kind Classification**：在 Resolution 与 Fetch Route 之间增加运行时 MIME/文件类型探测；
2. **Manual Ingestion API**：Web/API 支持输入任意允许的 URL，异步返回 `202 + ingestion_id`；
3. **Ingestion Status / Projection Readiness**：把 canonical、Markdown、全文、Embedding、Vector 的就绪状态分开；
4. **Retrieval Scope**：支持按单文档、站点、版本过滤检索，支撑“对这篇文章提问”；
5. **统一多类型 Document Ingress**：HTML/PDF/TXT/MD/JSON/Feed 共享同一事实和版本模型；
6. **Route Hint + Probe Evidence**：URL matcher 不再是 PDF/HTML 类型判断的唯一依据。

### 中优先级

7. Chroma 作为开发 Adapter，而不是生产 Truth；
8. LLMExtractionStrategy 仅作为版本化、可重建 Projection/fallback，不进入 canonical 主路径；
9. 手工摄取 Web 调试台展示全链路证据；
10. 对 MIME、redirect、文件大小、scope filter 增加 Contract Test。

### 不采纳

- 不新增 Celery 作为第二套任务系统；
- 不用 Chroma 替代 PostgreSQL/Object Storage 事实层；
- 不把教程中的同步 `/generate` 隐式抓取行为带入生产；
- 不使用 LLM 抽取结果作为 canonical Markdown；
- 不按 URL 扩展名直接决定资源类型。

---

## 18. 方案新增的数据实体建议

```text
Resource Probe Attempt
Manual Ingestion
Ingestion Stage State
Retrieval Scope
Content-kind Classification Evidence
```

### 18.1 Manual Ingestion

```text
id
requester
url
source_id
mode
idempotency_key
requested_projection_types[]
run_id
state
canonical_document_id
error_code
created_at
updated_at
```

### 18.2 Ingestion Stage State

```text
ingestion_id
stage
state
attempt_count
release_id
input_hash
output_hash
started_at
finished_at
error_code
```

这些实体让 Web/API 能明确回答“这一个 URL 到底处理到哪一步”，而不是只有一个模糊的成功/失败。

---

## 19. 技术原理总结

这篇文章最值得提炼的不是具体库组合，而是四个架构原理：

### 19.1 URL 是资源定位符，不是 HTML 类型

必须先识别资源类型，再选择解析器。这能把网页、PDF、Markdown、API 等统一纳入知识库。

### 19.2 Ingestion 与 Query 是两个生命周期

抓取/解析/建索引属于写路径，问答属于读路径。二者解耦之后才有异步、扩容、重试、缓存和版本治理空间。

### 19.3 Vector Store 是检索投影，不是原始事实

向量可以根据 Chunk/Model 重新构建，因此必须把原始 Snapshot、Canonical IR 和 Document Version 保存在独立事实层。

### 19.4 长任务必须任务化

HTTP API 只接受命令并返回 durable task id；真正的抓取、解析、Embedding 在 Worker 中执行。Celery 是一种实现，但本项目已有更适合全链路状态治理的 PostgreSQL + Outbox + Redis Streams 架构。

---

## 20. 参考资料

1. 原文：Building a RAG Pipeline with FastAPI, Haystack, and ChromaDB for URLs in Python  
   https://medium.com/@pwaykos1/building-a-rag-pipeline-with-fastapi-haystack-and-chromadb-for-urls-in-python-631575f3888b
2. 可访问镜像：  
   https://www.aihello.com/resources/blog/building-a-rag-pipeline-with-fastapi-haystack-and-chromadb-for-urls-in-python/
3. Crawl4AI LLM Strategies：  
   https://docs.crawl4ai.com/extraction/llm-strategies/
4. Crawl4AI v0.5.0 Release Notes（`LLMConfig` 迁移说明）：  
   https://docs.crawl4ai.com/blog/releases/0.5.0/
5. Crawl4AI Multi-Config / URL Matcher：  
   https://docs.crawl4ai.com/blog/releases/0.7.3/
6. FastAPI Background Tasks：  
   https://fastapi.tiangolo.com/tutorial/background-tasks/
7. Haystack ChromaEmbeddingRetriever：  
   https://docs.haystack.deepset.ai/docs/chromaembeddingretriever
8. Chroma Update / Upsert：  
   https://docs.trychroma.com/docs/collections/update-data

---

## 21. 代码级二次审计：集合分裂、Scope 泄漏与 Query Runtime

进一步核对文章完整代码后，可以看到教程实现里有几个比“用什么向量库”更关键的生产问题，这些问题直接影响多文档知识库的正确性和吞吐。

### 21.1 `identify_url()` 的真实判定链

教程不是单纯按后缀判断，而是：

```text
HEAD allow_redirects
 -> Content-Type == text/html ? article
 -> MIME -> extension
 -> final URL path -> MIME/extension
 -> GET(stream=True) 读取前 2048 bytes
 -> filetype.guess()
```

这比纯 suffix 已经更可靠，但仍有生产缺口：HEAD 可能被禁用、Content-Type 可能错误、GET fallback 必须受响应大小/超时/SSRF 约束，而且 Query 路径不应该再次执行这个网络探测。正确演进仍然是将其提升为持久化、版本化的 Resource Probe。

### 21.2 文件管线和 URL 管线实际上被拆成两个物理语料库

教程的文件摄取使用：

```text
FileTypeRouter
 -> PyPDFToDocument / TextFileToDocument / MarkdownToDocument
 -> DocumentJoiner
 -> DocumentCleaner
 -> DocumentSplitter(split_by="word", split_length=2000, split_overlap=500)
 -> OllamaDocumentEmbedder
 -> Chroma collection: file-index
```

URL 摄取则是：

```text
Crawl4AI / LLM extraction
 -> concatenate_content()
 -> JSONConverter(extra_meta_fields=["tag", "url"])
 -> OllamaDocumentEmbedder
 -> Chroma collection: url-index
```

这意味着“输入是文件还是网页”被编码进了物理 Collection。对于教程演示很直观，但不适合长期知识库。摄取来源和资源类型应该是 metadata，而不是检索拓扑。HTML、PDF、Markdown、TXT 在进入 Canonical IR / Chunk 后应进入同一个逻辑 Corpus/Index Generation；只有 Embedding 维度/Release、租户或安全边界、容量分片等真正影响索引兼容性的因素才应该决定物理分区。

否则会出现：

- 跨 HTML + PDF 的一次搜索必须查两个 Collection 再人工融合；
- 新增 JSON/Feed/Office 文档会继续产生更多 Collection；
- Collection 名称逐渐承担业务路由真相；
- Resource Kind 变更或 URL 重定向到另一种类型时迁移困难；
- Hybrid Search / Generation 切换需要维护多套并行索引。

### 21.3 `/generate(url, question)` 存在真实的 Scope 泄漏风险

教程的 generate 路径会重新调用 URL 类型识别，然后按类型选择 `get_url_result()` 或 `get_file_result()`。但两个 query 函数中的 Chroma retriever 都只选择 Collection，并没有按传入 URL 做 metadata filter。

因此如果已经摄取：

```text
URL A -> url-index
URL B -> url-index
```

用户调用：

```text
/generate(url=A, question=...)
```

实际语义只是“去 `url-index` 搜”，并不是“只在 A 中搜”。Retriever 完全可能返回 B 的 Chunk。文件集合也同样存在这一问题。

这说明生产系统不能把“URL 参数存在”误认为“检索已限定到该 URL”。必须执行稳定身份解析：

```text
requested URL
 -> normalize only（无远程网络访问）
 -> URL Alias / Canonical Mapping
 -> document_id
 -> active document_version_id（或显式指定版本）
 -> metadata filter
 -> BM25 / Vector retrieval
```

如果 URL alias 无法解析到已入库 Document，返回 `DOCUMENT_NOT_FOUND`，而不是重新访问远程 URL 来判断它属于 `file-index` 还是 `url-index`。

### 21.4 Query Path 不应依赖“此刻的远程资源类型”

教程在生成答案时再次执行 `identify_url(url)`，意味着一个纯读取请求仍可能发送 HEAD/GET。这样会导致：

- 源站暂时不可用时，已经入库的数据也无法查询；
- URL 从 PDF 改成 HTML 后查询被路由到另一 Collection；
- 重定向变化导致历史文档的查询语义漂移；
- Query API 获得不必要的 SSRF/网络权限；
- 查询延迟受外部站点网络影响。

生产系统必须坚持 Query Plane 的网络隔离：`search/ask` 只访问内部 Truth/Projection；远程 URL 只在 Ingestion/Sync Plane 中访问。

### 21.5 Query Pipeline 不应每个请求重新构建

教程的 `get_file_result()` 和 `get_url_result()` 每次调用都会重新创建：

```text
Haystack Pipeline
OllamaTextEmbedder
ChromaDocumentStore
ChromaEmbeddingRetriever
PromptBuilder
OllamaGenerator
```

功能上没问题，但服务化后会带来重复对象初始化、连接/客户端抖动、配置漂移和额外延迟。生产 Query Service 应把 Pipeline 视为 `retrieval_release_id` 对应的已编译运行图：

```text
service startup / release activation
 -> build or load pipeline graph
 -> initialize long-lived model/store clients
 -> warm up
 -> cache by retrieval_release_id

request
 -> resolve scope
 -> inject query / filters / top_k / token budget
 -> execute cached pipeline
```

Haystack/LlamaIndex 的职责是编排，不负责保存唯一业务状态。Release 切换时加载新 Pipeline 实例，验证后原子切 active pointer，旧实例在 in-flight 请求结束后回收。

### 21.6 教程的 Chunk 策略不能直接迁移到技术博客

文件管线固定使用 `split_length=2000`、`split_overlap=500`，即 25% word overlap；URL 管线则依赖 Crawl4AI/LLM extraction 产生的 item 粒度，代码中没有统一的后置 splitter。

生产风险包括：

- 2000 words 可能超过部分 Embedding 模型有效窗口；
- 500 words overlap 会显著放大存储和 Embedding 成本；
- fenced code、表格和标题层级可能被切断；
- URL 与 PDF 的 Chunk 粒度完全不一致；
- LLM extraction item 太大时可能整块被模型截断。

因此必须在 Canonical IR 后执行统一、版本化的 Structure-aware Chunk Release；资源解析器只负责产生结构块，不直接决定最终检索 Chunk。

### 21.7 对主方案的新增约束

在已有 Resource Probe、Manual Ingestion、Retrieval Scope 基础上，主方案还应明确增加以下约束：

1. **Ingress-neutral Corpus**：`file-index` / `url-index` 这种按输入形态拆 Collection 的方式仅用于教程，不进入生产；
2. **Stable Query Target**：URL scope 先解析成 `document_id/version_id`，Query 不访问远程 URL；
3. **Hard Scope Filter**：scope 必须在 BM25/Vector 召回阶段执行，不能只靠选择 Collection 或写入 Prompt；
4. **Compiled Query Runtime**：Haystack Query Pipeline 与 Store/Model Client 按 Retrieval Release 复用；
5. **Unified Chunk Release**：HTML/PDF/MD/TXT 都在 Canonical IR 后使用统一 Chunk 契约；
6. **Cross-resource Contract Test**：同时摄取多篇 HTML 与 PDF，验证 `DOCUMENT` scope 零泄漏，并验证 `ALL/SOURCE` 可以跨资源类型召回。

这几条将教程中的“能跑通单 URL RAG”进一步升级为“多来源、多文档、可持续运行且不会串文档的知识库检索服务”。