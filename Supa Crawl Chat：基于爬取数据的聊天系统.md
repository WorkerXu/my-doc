# Supa Crawl Chat：基于爬取数据的聊天系统

## 1. 调研对象与结论摘要

- 文章：Supa Crawl Chat
- 文章地址：https://bigsk1.com/posts/supa-crawl-chat/
- 项目地址：https://github.com/bigsk1/supa-crawl-chat
- 文章发布时间：2025-04-06
- 本次分析对象：文章描述 + 项目 `main` 当前实现，重点阅读 `crawler.py`、`crawl_client.py`、`content_hygiene.py`、`embeddings.py`、`db_setup.py`、`db_client.py`、`api/routers/crawl.py`、`api/routers/search.py`、`api/routers/sites.py`、`frontend/package.json`。
- 调研目标：评估其 Crawl4AI + PostgreSQL/Supabase + pgvector + FastAPI + React/Vite Web UI 的实现方式，对“约 1000 个技术博客全量历史抓取、长期增量同步、Markdown 知识库、Web 管理、搜索/RAG”的生产方案有哪些可吸收设计，以及哪些实现只能作为小型系统参考。

核心判断：Supa Crawl Chat 的价值不只是“调用 Crawl4AI”，而是已经形成了一个可用产品闭环：

```text
URL / Sitemap
  -> Crawl4AI API
  -> 结果兼容与内容清洗
  -> 标题/摘要增强
  -> 结构优先 + Token 兜底分块
  -> Embedding
  -> PostgreSQL/Supabase + pgvector
  -> FastAPI
  -> React/Vite Web UI
  -> Search / Chat / 单页刷新 / 站点管理
```

它非常适合作为 Web 管理、Chunk 组织、搜索 API 和用户可见任务状态的参考，但其核心抓取执行、版本模型、发现完整性和增量语义还不足以直接支撑 1000 个站点的多年知识库。

本次进一步确认了几项对现有方案有价值的优化：

1. Chunked 文档默认不要再给“完整父正文”做同类向量索引，避免父文档和子 Chunk 互相抢占 Top-K；父文档保留 metadata/summary/current pointer 即可。
2. 检索要分成“Chunk 候选召回 → Document 分组 → 邻接 Chunk/父上下文扩展”，不能简单按 URL 去重后只保留一条。
3. Canonical IR hash 应放到 LLM 摘要、Chunk、Embedding 之前，内容无变化时直接短路昂贵下游。
4. Web 列表 API 默认只返回 preview 和 artifact pointer，不加载整篇 Markdown，避免大正文导致数据库和 API 放大。
5. PostgreSQL FTS + pgvector 可作为第一阶段检索底座，但技术博客必须额外支持代码符号/配置项的精确 token 通道和多语言 lexical 检索；不要固定假设 `english` FTS。
6. `crawl_jobs` 可以借鉴为用户可见 Run Read Model，但不能用 FastAPI `BackgroundTasks` 作为 durable execution。
7. 失败 Embedding 不能写“全 0 向量”伪装成功，应作为显式失败状态进入重试/降级流程。

## 2. 项目定位与模块边界

项目整体可以拆成以下层次：

```text
Crawl4AI Remote Service
        |
        v
crawl_client.py
        |
        v
crawler.py
  |      |        |
  |      |        +--> ContentEnhancer / OpenAI
  |      +-----------> content_hygiene.py
  +------------------> EmbeddingGenerator
        |
        v
db_client.py -> PostgreSQL/Supabase + pgvector
        |
        +--> FastAPI routers
        |      |- crawl
        |      |- sites/pages
        |      |- search
        |      `- chat
        |
        `--> React/Vite frontend
```

这种分层比“在一个脚本里抓完就写 Markdown”更接近真正可运营系统，但边界仍然偏应用级：Crawler 同时负责抓取、清洗、Chunk、AI 增强、Embedding 和入库，长期运行后会出现故障域耦合和重处理成本过高的问题。

生产方案应继续拆分为：

```text
Discovery -> Admission -> Fetch -> Extraction -> Canonical IR
          -> Quality -> Accepted Version
          -> Chunk / Search / Embedding / Summary / Publish
```

即源站同步真相与所有 AI/Search Projection 解耦。

## 3. Crawl4AI 客户端兼容层

### 3.1 同步与异步 API 兼容

`crawl_client.py` 对 Crawl4AI 不同版本做了兼容：

- `POST /crawl` 可能直接返回同步 `results`；
- 也可能返回 `task_id`，然后轮询 `/crawl/job/{task_id}`；
- 老版本还兼容 `/task/{task_id}`。

这说明第三方抓取引擎必须被包在 Adapter 后面，不能让上层业务直接依赖某个版本的 API 形态。

推荐生产抽象：

```text
FetchAdapter.execute(request) -> FetchOutcome

FetchOutcome:
- status
- final_url
- http_status
- mime_type
- raw_snapshot_ref
- rendered_snapshot_ref
- extracted_candidates[]
- link_candidates[]
- retry_hint
- engine_metadata
```

Adapter 可以是 HTTP、Crawl4AI Browser、Playwright、PDF/OCR；上层只消费稳定 Outcome Contract。

### 3.2 当前客户端的执行粒度

项目的 Sitemap 路径最终逐 URL 调用 `crawl_and_wait()`。这种实现简单，但会把“URL 枚举”和“抓取吞吐”绑成串行循环。

1000 个站点生产环境应把 URL 发现后立即写入 durable frontier，由 worker 池按 host 限速、优先级和公平调度并发处理；Sitemap parser 不应该等待每个页面抓完才能继续。

## 4. Crawl 结果归一化与 Candidate 机制

`crawler.py::process_crawl_results()` 会兼容 Crawl4AI 多种返回形态，并按如下优先级寻找内容：

```text
markdown.raw_markdown
 -> markdown.fit_markdown
 -> markdown string
 -> extracted_content
 -> cleaned_html
 -> html
```

同时兼容：

- `results[]`；
- `pages{url: page}`；
- 单个 `result`。

这是正确的边界适配思想，但不能把 fallback 顺序本身当成“正文质量判定”。例如 `cleaned_html` 或 `html` 非空，只能证明有内容，不能证明它是文章正文。

生产方案应保存所有有价值 Candidate，并记录：

```text
candidate_type
engine
engine_version
extraction_profile_release
source_artifact_id
content_hash
field_provenance
quality_features
quality_score
```

然后由 Post-Processor + Quality Policy 选择 Accepted Canonical IR。

## 5. Content Hygiene：先清洗再索引

`content_hygiene.py` 做了一个很实用的“检索污染防护层”：

- 清理控制字符；
- 删除巨大 `data:` base64；
- 检测巨大编码 token；
- 检测疑似编码垃圾的 fenced code block；
- 超长内容截断；
- 生成 `quality_flags` 和清理计数。

其价值不是“把网页清洗干净”，而是避免大块 base64、编码垃圾和异常附件正文占据 Embedding / FTS 的主要权重。

生产方案应吸收这种“清理必须产生 Evidence”的方式，但有两个改进：

1. 不要直接把被截断内容视为完整 Canonical Document。`content_truncated` 必须进入质量门，并保留原 Snapshot 以便离线重新抽取。
2. 清洗规则必须版本化。否则将来规则变化时无法判断内容差异来自源站还是清洗器。

推荐：

```text
raw_snapshot
 -> extraction_candidate
 -> post_processor_release
 -> canonical_ir
 -> quality_result
```

每个清理动作写入 `field_transform_trace` / `artifact_transform_trace`。

## 6. Sitemap、llms.txt 与历史发现

### 6.1 项目值得吸收的兼容能力

项目不仅接受 XML Sitemap，还会在 XML 解析失败时从普通文本 / Markdown 中提取链接，因此可以兼容 `llms.txt`、Markdown URL 索引等形式。

这可以直接转化为 Discovery Provider：

```text
SITEMAP_XML
SITEMAP_INDEX
RSS_ATOM_JSONFEED
TEXT_URL_LIST / LLMS_TXT
HTML_ARCHIVE
CATEGORY_TAG_AUTHOR
DOC_NAV
PUBLIC_API
INTERNAL_LINK
```

### 6.2 当前实现的完整性缺口

当前 Sitemap 代码在遇到 sitemap index 时会读取 `<sitemap><loc>`，但随后这些地址仍进入普通 URL 抓取循环；它没有建立“递归 Sitemap Provider + 子 sitemap checkpoint + coverage ledger”的完整模型。

对历史全量抓取，这个差异非常关键：

```text
发现 sitemap index
 -> 枚举所有 child sitemap
 -> 每个 child sitemap 建 checkpoint
 -> 记录 URL 数、lastmod 范围、解析错误、terminal reason
 -> URL admission
 -> frontier
 -> 覆盖核对
```

不能把 `max_urls` 截断后的“循环执行完”当成历史完整。

## 7. 安全边界

项目已经意识到“用户输入 URL = 网络访问能力”：

- 仅允许 HTTP/HTTPS；
- URL 进入安全校验；
- 外链跟随默认关闭；
- redirect、custom proxy、文件下载等能力需要显式环境开关；
- 管理操作有审计日志。

这个方向应保留，并在生产进一步固化为 Admission Policy：

```text
URL parse
 -> scheme allowlist
 -> host normalization
 -> DNS resolve
 -> private/reserved/metadata IP block
 -> port policy
 -> robots/scope
 -> redirect hop re-validation
 -> asset URL same-policy validation
```

必须防 DNS rebinding；redirect 每一跳都要重新检查目标；代理和自定义 header/JS 不能由普通租户任意注入。

## 8. 语义分块实现

### 8.1 当前实现路径

`chunk_content()` 首先计算正文 Token 数：

```text
内容未超阈值
 -> 作为父页面，不生成 Chunk

内容超阈值
 -> 优先按 Markdown heading 分 section
 -> 无 heading 时按段落
 -> section 组合至 max_tokens
 -> 单 section 超限再 token 切割
 -> 创建 overlap
```

Overlap 会尝试从段落、句号、逗号、空格寻找自然断点。默认参数还可以通过环境变量调整。

这比固定字符切分更符合技术博客结构，因为代码、标题、命令和解释更容易保留在同一语义单元。

### 8.2 正确的生产升级方式

不要长期用正则重新解析扁平 Markdown，而应基于 Canonical IR blocks：

```text
heading
paragraph
code
list
table
blockquote
image
callout
```

Chunker 根据 heading path、block 类型、Token 预算组合块；这样代码块、表格和标题关系不会在 Markdown 正则阶段丢失。

### 8.3 Chunk Identity 不能依赖 `#chunk-N`

项目会生成：

```text
<url>#chunk-0
<url>#chunk-1
```

正文前面插入一段文字后，所有后续 index 都可能漂移。因此：

```text
chunk_id = hash(
  document_version_id
  + chunker_release_id
  + block_start/block_end
  + chunk_content_hash
)
```

`#chunk-N` 只作为 UI 展示，不承担 identity。

## 9. 一个值得直接吸收的优化：Chunked 父文档不再做完整正文向量

当前 `enhance_pages()` 对有 Chunk 的 parent page 会把 `embedding` 置空，只给 Chunk 生成 embedding。

这是一个非常重要但容易被忽略的检索设计。

如果父正文和所有 Chunk 都做同一种 Embedding，则一次查询 Top-K 很容易出现：

```text
同一文章 parent
同一文章 chunk-2
同一文章 chunk-3
同一文章 chunk-5
```

大量候选被同一篇文章占满，降低跨文档召回。

生产方案推荐：

```text
Document Current Projection
- title
- summary
- metadata
- optional lightweight document embedding

Chunk Projection
- chunk text
- chunk embedding
- heading path
- parent version
```

默认向量主索引只检索 Chunk；如需要 document-level semantic routing，可另建“标题 + 摘要”的 document embedding，不要重复嵌入整篇父正文。

## 10. Embedding 实现与需要规避的问题

`embeddings.py` 使用 OpenAI Embedding，并用 tiktoken 估算 Token；超长输入会截断/只取第一段。

项目还提供 batch 方法，但主 `enhance_pages()` 实际上按 batch 切列表后，内部仍逐 page 同步调用 `generate_embedding()`，并没有充分利用批量 API 和高并发 worker。

更重要的是，`enhance_pages()` 在 embedding 失败时会写一个 1536 维全 0 向量作为 fallback。

这个做法在生产知识库中应明确禁止：

```text
Embedding 失败 != 有效零向量
```

零向量会污染索引语义、掩盖失败率，还可能在不同向量实现中导致异常距离行为。

正确状态应为：

```text
embedding_state = PENDING | READY | FAILED | BLOCKED
embedding = NULL when not READY
error_code / retry_at / model_release recorded
```

FTS 可以继续工作；Embedding worker 独立重试，不阻塞抓取主链路。

## 11. AI 标题/摘要的成本顺序

项目在抓取后先生成标题/摘要，再 Chunk 和 Embedding，最后写入 `content_hash`。

对于重复抓取，这意味着即使正文完全没变，也可能再次付出 LLM/Embedding 成本。

生产方案应该把 Canonical IR hash 提前：

```text
fetch/extract
 -> Canonical IR
 -> canonical_ir_hash
 -> 与 current version 比较

hash 相同：
 -> 只记录 freshness observation / provider evidence
 -> 不生成新 version
 -> 不跑 summary/chunk/embedding

hash 不同：
 -> 新 document_version
 -> 异步触发下游 projection
```

这是 1000 站长期增量同步里非常直接的成本优化。

## 12. PostgreSQL / Supabase 数据模型

项目核心模型把父页面和 Chunk 放在 `crawl_pages`：

```text
crawl_sites
crawl_pages
crawl_jobs
user_preferences
...
```

`crawl_pages` 包含：

```text
site_id
url
title
content
summary
embedding vector(1536)
metadata
content_hash
is_chunk
chunk_index
parent_id
created_at
updated_at
```

优点：结构简单，父页/Chunk 一张表即可查询，适合中小型产品快速落地。

但长期知识库必须拆分：

```text
document                 稳定文档实体
document_version         append-only 内容历史
document_current_projection
chunk_projection         version + chunker release 派生
index_projection         embedding/search release 派生
search_current_projection
```

否则一旦需要多版本、多 Chunker、多 Embedding model、蓝绿索引和历史回放，一张 `crawl_pages` 会迅速失控。

## 13. URL 原地更新与 Current Projection

`db_client.py::add_pages()` 的父页面逻辑本质是：

```text
WHERE url = ? AND is_chunk = false
 -> exists: UPDATE
 -> missing: INSERT
```

Chunk 也类似；`replace_chunks=True` 时删除旧 Chunk 后重建。

这种模式非常适合“当前运营视图”，但会丢失知识库需要的版本历史。

推荐吸收其查询体验，而不是吸收其真相模型：

```text
document_version            append-only truth
        |
        +--> document_current_projection
        +--> chunk_projection
        `--> search_current_projection
```

新版本 Accepted 时：

```text
BEGIN
  INSERT document_version
  INSERT quality/evidence/lineage
  UPSERT document_current_projection
  INSERT outbox event
COMMIT
```

搜索指针在新 Chunk/Embedding/Index Manifest 完整后再原子切换。

## 14. Search：pgvector + lexical 的启发

### 14.1 向量搜索

数据库使用 pgvector cosine distance，并建立向量索引。对第一阶段部署，这比一开始就引入独立向量数据库更简单：业务真相与向量 projection 同库，运维成本低。

但索引参数不能永久写死。例如 `ivfflat lists=100` 对数据规模、召回率和 build strategy 都敏感。生产应把索引类型和参数版本化，并通过离线 Retrieval Eval 决定 HNSW/IVFFlat/OpenSearch 等选择。

### 14.2 Text Search 当前局限

代码使用 PostgreSQL `english` text search。技术博客知识库可能包含中文、日文、混合语言、代码符号、版本号和配置键：

```text
asyncio.create_task
vector_cosine_ops
ERR_CONNECTION_RESET
--max-depth
text-embedding-3-small
nginx.conf
```

仅 `english` stemmer 不足以覆盖这些 token。

生产检索应至少有三类 lexical 信号：

```text
1. language-aware FTS/BM25
2. exact keyword / code symbol / path token
3. trigram / substring fallback
```

再与向量结果通过 RRF 或学习权重融合。

## 15. Search API 的去重策略需要升级

`api/routers/search.py` 会把 `#chunk-N` 去掉，按 base URL 只保留 score 最好的结果。这个设计对“搜索结果列表”很友好，可以避免一篇文章占满页面。

但 RAG 检索不能简单“一文档只留一个 Chunk”。一个复杂技术问题经常需要同一文档的两个相邻章节。

推荐两阶段：

```text
Stage A: chunk candidate retrieval
  -> vector + lexical
  -> 取较大 candidate pool

Stage B: document-aware grouping
  -> 每 document 限制 max_chunks_per_document
  -> 保留多个高价值 chunk
  -> 邻接 chunk expansion
  -> parent metadata/title expansion
  -> token-budget context assembly
```

Web 搜索可以用 `dedupe=document`；RAG Retrieval Profile 则使用 `grouped-multi-chunk`，两者不要共用一个简单布尔开关。

## 16. Web API 的 preview 设计值得吸收

`sites.py` 和 `search.py` 默认允许：

```text
include_content=false
preview_chars=N
content_length
content_truncated
```

也就是说列表接口无需每次返回完整 Markdown。

这对百万级知识库非常重要。生产数据层进一步应做到：

```text
document_current_projection
- title
- metadata
- content_preview
- markdown_artifact_id
- content_length
```

完整 Markdown 从对象存储或专门 detail API 获取，站点列表和搜索列表不扫描大正文列。

## 17. Web 管理信息架构

项目的 React/Vite UI + FastAPI 已具备典型管理路径：

```text
Crawl
Sites
Site Detail / Pages
Search
Chat
```

结合 API 能完成：

- 新建抓取；
- 查看站点；
- 查看父页面和 Chunk；
- 单页刷新；
- 搜索；
- 聊天；
- 查看抓取状态。

这对本项目的 Web IA 很有参考价值：默认界面应该面向“当前状态”，排障时再逐层下钻：

```text
Site
 -> Current Documents
 -> Document Detail
    -> Current
    -> History
    -> Version Diff
    -> Raw Snapshot
    -> Extraction Candidates
    -> Canonical IR
    -> Chunks / Index
    -> Query Trace
```

## 18. `crawl_jobs`：Read Model 正确，Execution 仍不 durable

`crawl_jobs` 记录：

```text
site_id
url
status
options
crawl4ai_task_id
pages_found
pages_crawled
chunks_created
error
started_at
finished_at
updated_at
```

它非常适合做用户可见 Job Read Model。

但 `api/routers/crawl.py` 使用 FastAPI `BackgroundTasks` 真正执行长期抓取，因此：

```text
API process death
 -> BackgroundTasks execution lost
 -> DB job row remains
 -> 没有 lease 接管
```

生产应保留现有方案：

```text
command/run transaction
 -> outbox
 -> Redis Streams / durable queue
 -> worker lease + heartbeat
 -> stage execution
 -> finalizer
 -> job/run projection
```

Web 看到的 `crawl_job` 可以是 projection，而不是调度事实源。

另外，Job 的 `pages_crawled/chunks_created` 必须基于本次 Run Manifest 计数，不能简单读取站点当前总页数，否则增量刷新老站点时会把历史库存当成本次任务产出。

## 19. Freshness 与 `crawled_at` 不能混淆

Search API 支持按 metadata 中 `crawled_at` 做 after filter，这是一个好用的 UI 功能，但 `crawled_at` 只是抓取时间。

它不能代替：

- `published_at`；
- `updated_at`；
- Sitemap `lastmod`；
- Feed GUID/updated；
- HTTP ETag/Last-Modified；
- Canonical IR hash freshness。

生产方案必须保持 Identity、Publication Time、Source Freshness、Fetch Observation 四种概念分离。

## 20. 数据库性能与连接模型

当前代码主要使用 `psycopg2.connect()` 按操作创建连接，站点列表还会逐站点查询 page count。小规模很直观，但 1000 站 Web 页面会出现连接建立开销和 N+1 查询。

生产方案建议：

```text
- PgBouncer / application pool
- aggregate site_operational_projection
- 单次 SQL 返回站点 + current_doc_count + last_run + failure_rate
- 大正文不进入列表查询
- 热表合理分区/索引
```

Web 查询应优先读 projection，而不是临时聚合大量历史事实表。

## 21. AI Chat / Profile 的可借鉴边界

项目支持 chat profile、site filter、相似度阈值和结果数量配置。这揭示一个有价值的抽象：不同知识使用场景应该绑定不同 Retrieval Profile，而不是把所有参数硬编码在聊天 UI。

生产设计：

```text
retrieval_profile_release
- source/site filters
- lexical/vector weights
- candidate_k
- max_chunks_per_document
- neighbor_expansion
- reranker
- token_budget
- freshness policy
```

Chat Profile / Agent Profile 只引用一个 Retrieval Profile + Prompt Release。

这样 Search、RAG、Agent、Digest 可以共享可评估、可版本化的检索层。

## 22. Docker 部署模式的启发

项目文章提供 app-only、App+Crawl4AI、full-stack 等部署方式，开发体验很好。

生产可把这种思想固化为 Deployment Profile：

```text
DEV_FULLSTACK
DEV_EXTERNAL_DB
STAGING
PROD_K8S
```

每个 Profile 固定：

- 组件版本；
- Secret Binding；
- 网络策略；
- 资源 limit/request；
- worker 并发；
- Browser pool；
- 数据保留策略。

不把大量 `.env` 组合当作隐式部署契约。

## 23. 对现有博客知识库方案的具体优化

### 23.1 明确 Parent/Chunk 双层索引策略

新增原则：

- Chunked 文档的完整父正文默认不进入主向量索引；
- 父文档只保留 title/summary/metadata，必要时建立轻量 document embedding；
- 主要向量召回来自 chunk_projection；
- lexical document index 可以保留整文命中能力；
- RAG 通过 parent/neighbor expansion 重建上下文。

### 23.2 新增“变更短路门”

Canonical IR 生成后立即计算 hash：

```text
same hash
 -> observation only
 -> skip new version
 -> skip chunk/embed/summary/index

changed hash
 -> append version
 -> downstream projections
```

### 23.3 Retrieval 改为 Document-aware Multi-Chunk

不使用简单 URL 单条去重作为 RAG 默认逻辑，新增：

```text
max_chunks_per_document
neighbor_window
same_heading_bonus
document_diversity_limit
```

### 23.4 Web List/Detail 分离

列表 projection 只存/返回 preview 和 artifact pointer；完整正文走 detail API / object storage。

### 23.5 明确多语言 + 代码检索

基础方案采用：

```text
PostgreSQL FTS/trigram + pgvector
```

但抽象保持 Search Projection 可替换；规模或需求增长后可切 OpenSearch。代码符号、错误码、CLI 参数、路径、配置项建立 exact token 字段。

### 23.6 Embedding Failure Contract

禁止零向量 fallback；Embedding 失败保持 NULL + FAILED/PENDING 状态，并异步重试。

### 23.7 Run 指标必须 Run-scoped

Web 展示的 discovered/fetched/accepted/chunked/indexed/failed 数量来自 Run Manifest，不从站点累计库存反推。

### 23.8 Stale Refresh 改为 Freshness Policy

“30 天没抓”只能作为兜底，不是源站更新判断。优先使用 Sitemap lastmod、Feed/API 时间、ETag、Last-Modified、hash 和 Reconcile。

## 24. 不应照搬的设计

1. 不能用 URL 原地 UPDATE 代替 append-only 文档版本。
2. 不能用 `#chunk-N` 作为稳定 Chunk Identity。
3. 不能用 FastAPI BackgroundTasks 承担长期抓取。
4. 不能把父页面、Chunk、所有历史版本和所有 Embedding release 永久混在同一表。
5. 不能让 LLM 标题/摘要成为源站同步成功的前置条件。
6. 不能把 `max_urls` 执行完当作历史完整。
7. 不能把全 0 向量当 Embedding 失败 fallback。
8. 不能固定使用 `english` FTS 覆盖多语言技术博客。
9. 不能把 `crawled_at` 当发布时间或可靠更新证据。
10. 不能为 RAG 简单按 URL 去重到只剩一个 Chunk。
11. 不能让 Web 列表默认加载全文 Markdown。
12. 不能用站点当前总库存作为某次 Run 的实际抓取计数。

## 25. 最终评价

Supa Crawl Chat 最值得借鉴的是“把爬虫做成产品”的工程思路：站点、页面、Chunk、抓取任务、搜索、聊天和 Web 管理形成完整闭环；Chunked parent 不做完整正文 embedding、搜索结果提供 preview、单页刷新和可见 Job 状态等细节都对真正的知识库产品有直接价值。

但它仍是以应用快速落地为中心的架构，而不是以多年数据真相和可恢复执行为中心的抓取平台。对约 1000 个技术博客的长期知识库，应继续保留 Durable Frontier、Immutable Snapshot、Canonical IR、append-only Version、Transactional Outbox、Worker Lease、Coverage Evidence、版本化 Search/RAG 等主架构。

本次调研新增到最终方案的重点不是“改成 Supabase + Crawl4AI”，而是四个更具体的工程规则：

1. **Parent/Chunk 检索分层，避免父正文与 Chunk 重复向量召回。**
2. **Canonical IR hash 前置，未变化内容短路所有昂贵 AI/Search 下游。**
3. **Retrieval 使用 Document-aware Multi-Chunk grouping + neighbor expansion，而非简单 URL 单条去重。**
4. **Web 默认读轻量 Current Projection，全文正文通过 artifact/detail 按需加载。**
