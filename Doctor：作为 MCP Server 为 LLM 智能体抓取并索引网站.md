# Doctor：作为 MCP Server 为 LLM 智能体抓取并索引网站

## 1. 调研对象与结论

- 项目：`sisig-ai/doctor`
- 地址：https://github.com/sisig-ai/doctor
- 调研日期：2026-08-14
- 本次阅读基线：GitHub `main`，tree SHA `4860bdf425613180762c0b8e40f54734ecb2f66b`
- 许可证：MIT

Doctor 是一个把“网站抓取 → Markdown/文本 → 分块 → Embedding → 向量/全文检索 → FastAPI → MCP”串成完整闭环的轻量知识库项目。它的最大价值不在于生产级爬虫调度，而在于两个设计点：

1. **把抓取时的父子关系保存下来，并提供可浏览的站点 Map/树形导航。**这对大规模博客知识库的“发现覆盖诊断、孤儿 URL 检测、回溯某个 URL 是怎样被发现的”非常有价值。
2. **把已抓内容继续做成独立的知识访问面：混合检索 + 只读 MCP。**这说明抓取系统的最终产物不应只是一批 md 文件，还应有一个可重建、可替换、与 canonical 内容解耦的查询索引。

但 Doctor 本身不适合直接作为“1000 个技术博客、全量历史、长期增量”的核心架构：它使用 DuckDB 作为主数据与向量存储、Redis RQ 做单任务队列，抓取结果整体回收后再顺序处理页面，没有 durable frontier、Provider checkpoint、conditional fetch、版本模型、transactional outbox、域级限流、安全出网和大规模多租户调度。它更适合作为**功能原型和产品体验参考**，而不是生产核心。

因此对现有《博客知识库技术方案》的结论是：

- **保留现有 PostgreSQL + Redis Streams + S3/MinIO + immutable snapshot + Article Version 主链路，不引入 DuckDB/RQ 替代。**
- **新增“站点拓扑 / Discovery Graph”能力**，把 Doctor 的 hierarchy/map 思路生产化，但不把 Web 强行压成单父树。
- **新增“Knowledge Access Plane”能力**：以已发布 `article_version` 为唯一索引输入，异步构建 FTS/向量索引，对外提供只读 Search API / 可选 MCP；索引可丢弃重建，不能反向成为内容真相源。

## 2. Doctor 总体架构

README 给出的链路是：

```text
Crawl4AI deep crawl
 -> hierarchy tracking
 -> Markdown/text extraction
 -> LangChain chunking
 -> LiteLLM/OpenAI embedding
 -> DuckDB pages + document_embeddings
 -> DuckDB FTS + VSS/HNSW
 -> FastAPI search/map API
 -> FastApiMCP
```

部署上用 Docker Compose 启动：

- `redis`：RQ broker；
- `crawl_worker`：消费抓取任务；
- `web_service`：FastAPI + MCP；
- `./data`：挂载 DuckDB 数据文件；
- `./src`：开发态源码挂载。

这是一种适合个人/小团队、本机单实例的“全栈一体化”方案，部署简单、功能闭环快，但其数据与并发模型不适合直接扩展为上千站点长期运行平台。

## 3. 抓取实现细节

### 3.1 Crawl4AI 深度抓取

核心实现位于 `src/lib/crawler.py`。

项目使用：

- `AsyncWebCrawler`
- `CrawlerRunConfig`
- `BFSDeepCrawlStrategy`
- `PruningContentFilter`
- `DefaultMarkdownGenerator`

关键配置逻辑：

```text
BFSDeepCrawlStrategy
- max_depth: 默认 2
- max_pages: 默认 100
- include_external: false
```

Markdown 侧：

```text
PruningContentFilter(threshold=0.6, threshold_type="fixed")
DefaultMarkdownGenerator
- ignore_links: strip_urls
- body_width: 0
- ignore_images: true
- single_line_break: true
```

并在抓取配置里排除：

```text
nav / footer / aside / header
remove_overlay_elements = true
```

### 3.2 原理

BFS 的优点是：从入口页按链接层级扩展，天然能得到 `depth`，适合生成站点导航结构；`include_external=False` 可以控制在站内。

但 BFS 深度不是“历史全量”的充分条件。技术博客常见的问题包括：

- 历史文章只在 Sitemap 中出现，不在当前页面链接图中；
- archive 分页深度很大；
- tag/author/calendar 页面产生大量非文章分支；
- 同一文章通过多个入口被发现；
- JS 列表、virtual scroll、load-more 不适合静态 BFS；
- 站点迁移后旧路径可能完全不在当前链接图中。

所以 Doctor 的 BFS 应作为现有方案中的 `DeepCrawlProvider` 补充，而不能替代 Sitemap、Feed、Ordered Archive、Common Crawl、Wayback 等 Provider。

### 3.3 内存模型问题

`crawl_url()` 调用 `crawler.arun()` 后得到整个 `crawl_results` 列表，然后才返回并继续处理。这种方式在 `max_pages=100` 时很简单，但在几十万 URL 的站点历史回灌中会产生明显问题：

- URL frontier 不 durable；
- 中途失败只能重来；
- 大列表驻留内存；
- 无法对 Fetch、Extract、Publish 独立 backpressure；
- 单任务难以公平地与其他站点共享资源。

因此生产方案必须继续使用现有“Provider 流式产 URL → Frontier UPSERT → Scheduler/Lease → 独立 Worker”的设计。

## 4. hierarchy tracking 的实现与价值

### 4.1 Doctor 如何记录层级

`src/lib/crawler_enhanced.py` 对 Crawl4AI result 再包装为 `CrawlResultWithHierarchy`，读取：

- `metadata.parent_url`
- `metadata.depth`

并补充：

- `root_url`
- `relative_path`
- `title`

标题优先从 `<title>` 正则读取，失败后从 Markdown 第一条 H1，再失败则取前几行文本，最后退化到 URL path。

随后它建立 `url_to_result`，按 parent URL 拼接 `relative_path`。

### 4.2 数据库存储

`pages` 表增加：

```text
parent_page_id
root_page_id
depth
path
title
```

`processor_enhanced.py` 会把抓取结果按 `depth` 排序，确保父页面先写入，再用 `url_to_page_id` 找到 parent/root id。

### 4.3 Web Map

Doctor 暴露：

```text
GET /map
GET /map/site/{root_page_id}
GET /map/page/{page_id}
GET /map/page/{page_id}/raw
```

`MapService` 会：

- 查询 root pages；
- 按 domain 合并多个 root；
- 对旧数据构造 synthetic domain root；
- 根据 `parent_page_id` 构建树；
- 提供 parent/siblings/children/root 导航。

这套体验对于知识库运维很实用：管理员不只看到“有 10000 个 URL”，还可以从站点结构视角检查哪些区域被发现、哪些分支异常、某页位于什么层级。

### 4.4 为什么不能直接复制 Doctor 的单父树模型

Web 本质上是图，不是树：

- 一个文章会同时出现在首页、分类、标签、作者、年度归档、Sitemap；
- canonical/redirect 是另一类关系；
- Sitemap index 与 child sitemap 也存在层级；
- 同 URL 可能由多个 Discovery Provider 发现；
- 链接图可能有环。

Doctor 在一次 BFS 中保留一个 `parent_url`，能用于“导航树”，但会丢失其他发现证据。

此外，它的 `relative_path` 会把**标题**拼进路径。标题变化、重复标题、特殊字符都会让这个路径不稳定，因此不能作为文章 identity 或导出物理路径。

生产化方案应该保存 **Discovery Graph/Edge**，树仅是 Web 展示时从图中选择的一种投影。

## 5. 页面持久化与增量能力分析

### 5.1 `store_page()` 的行为

`src/lib/database/operations.py` 中 `store_page()` 默认每次生成新的 UUID：

```text
page_id = page_id or uuid.uuid4()
```

然后直接 `INSERT INTO pages`。

`pages` 表只有 `id` 主键，没有 URL 唯一约束，也没有 canonical article identity、content hash、snapshot id、version id。

因此同一 URL 多次抓取会形成多个 page 记录，而不是：

```text
URL identity
 -> conditional fetch
 -> unchanged
or
 -> new immutable snapshot
 -> new article_version
```

这对于一次性索引可以接受，但不满足长期增量同步。

### 5.2 缺少 HTTP 增量 checkpoint

Doctor 的核心抓取路径没有展示以下生产能力：

- ETag / `If-None-Match`
- Last-Modified / `If-Modified-Since`
- 304 short circuit
- raw response hash
- content hash / Markdown hash 分层比较
- 删除确认状态机
- Provider checkpoint
- reconciliation

所以不能用它的 page 表来承担当前方案中的 `frontier_urls + fetch_snapshots + articles + article_versions`。

## 6. 任务与队列实现

### 6.1 Redis RQ

`src/crawl_worker/tasks.py` 的 `create_job()` 先在 DuckDB 插入 job，再通过 RQ：

```text
Queue("worker").enqueue(perform_crawl, ...)
```

worker 把 job 改为 `running`，然后执行整条 crawl pipeline，最后标记 completed/failed。

### 6.2 双写窗口

这里存在典型的 DB + Queue 非原子双写：

```text
DuckDB commit job
 -> enqueue Redis
```

如果 DB commit 成功但 enqueue 失败，会留下永远 pending 的 job；反向场景也可能形成状态错位。

Doctor 代码里为了 DuckDB 持久化还显式 `CHECKPOINT` 并读回 job 验证，这可以提高单机可见性，但不是跨系统一致性方案。

现有方案里的 **PostgreSQL transactional outbox** 比 Doctor 更适合生产：

```text
PG transaction: job + job_item + outbox
 -> commit
 -> relay publish Redis Streams
```

因此这里不应改用 RQ。

## 7. 抓取、存储、Embedding 的耦合

`processor_enhanced.py` 对每个页面做：

```text
extract page text
 -> store_page
 -> chunk
 -> generate_embedding
 -> index_vector
```

页面是按 depth 排序后**逐页**处理；页内 chunk 的 embedding 以最多 5 个并发分批执行。

### 优点

- 实现简单；
- 处理流程容易理解；
- embedding 并发有明确上限；
- parent 在 child 前写入，层级关系简单。

### 问题

1. 抓取结果已经整体在内存中，不能边发现边处理。
2. 页面正文写入后，如果 embedding 中途失败，page 和 chunks 会进入部分状态；没有显式的 derived-index state machine。
3. canonical 内容发布与外部 Embedding API 成败耦合太紧。
4. 页面间处理串行，吞吐受限。
5. 每个 page 新建 `VectorIndexer()` / 数据库连接，长期大规模运行成本不理想。

当前生产方案应维持：

```text
article_version published
 -> outbox/index event
 -> independent indexing worker
 -> chunk/index state
```

Embedding、FTS、MCP 全部是派生能力，失败不能阻塞 Markdown 正式发布。

## 8. Chunking 与 Embedding

### 8.1 Chunking

Doctor 使用 LangChain `RecursiveCharacterTextSplitter`，参数来自 `CHUNK_SIZE` 和 `CHUNK_OVERLAP`，长度函数是 Python `len`。

优点是通用、稳定、实现成本低；缺点是它主要按字符递归分割，并不理解 Markdown AST、标题层级、代码块、表格、引用块。

对于技术博客知识库，更适合：

```text
Markdown AST
 -> section path
 -> block-aware grouping
 -> token budget split
 -> code/table atomicity guard
 -> overlap only at semantic boundary
```

chunk metadata 至少保留：

```text
article_id
article_version_id
site_id
section_path
heading_path
chunk_index
text_hash
source_url
published_at
```

这样检索命中后可以稳定回到具体版本与章节。

### 8.2 Embedding

Doctor 使用 LiteLLM `aembedding()`，区分 `doc` 与 `query` model 配置，默认 timeout 30 秒。

这个抽象值得保留：生产系统也应把 embedding provider/model 作为可替换 Adapter，并把：

```text
provider
model
model_revision/dimension
input_hash
chunker_version
```

写入索引版本元数据，方便重建和回滚。

## 9. DuckDB FTS + Vector Search

### 9.1 存储结构

Doctor 在 DuckDB 中维护：

- `pages`
- `document_embeddings`
- `jobs`

向量表使用固定维度 `FLOAT[VECTOR_SIZE]`，并尝试通过 VSS/HNSW 建向量索引；全文检索使用 DuckDB FTS `match_bm25`。

### 9.2 Hybrid Search

`document_service.py` 并行运行：

```text
vector search
BM25/FTS search
```

然后合并 page 级结果。

默认思路是向量权重约 0.7，BM25 权重约 0.3，并把 BM25 raw score 除以经验常数 10 再裁剪到 0~1。

### 9.3 可借鉴与需要改进的地方

**值得借鉴：**

- lexical + semantic 双路召回；
- 两路检索并行；
- tags filter；
- 可选择只返回命中 chunk 或返回全文。

**不建议直接复制：**

- 不同检索器的 raw score 分布不可稳定比较，固定除以 10 是数据集相关经验值；
- 固定加权容易随 embedding model、语言和语料分布变化失真。

生产方案建议优先使用：

- Reciprocal Rank Fusion（RRF）；或
- 分路标准化 + 可离线评估的 learned/rule rank；
- 对中文技术博客增加 title/heading/code symbol/URL path 等 field boost。

DuckDB 可以作为单机分析/离线导出工具，但不应替代主系统的 PostgreSQL 状态层。检索规模较小时可从 `PostgreSQL FTS + pgvector` 起步；规模/检索 QPS 上升后再把派生索引迁往 OpenSearch/Elasticsearch、Qdrant、Milvus 等独立服务。

## 10. FastAPI 与 MCP 设计

`src/web_service/main.py` 使用 FastAPI，并通过 `fastapi_mcp.FastApiMCP` 直接把 API 暴露为 MCP。

值得注意的是它明确排除了：

```text
fetch_url
job_progress
delete_docs
```

也就是说 MCP 主要暴露：

- `list_tags`
- `search_docs`
- `get_doc_page`
- `list_doc_pages`

这是一个非常好的边界：**给 LLM 的 MCP 默认只读，不把抓取、删除、管理操作直接暴露给模型。**

对于我们的博客知识库可以采用相同原则，但还需要增加：

- 身份认证和 token scope；
- site/tag/time 范围授权；
- raw snapshot、rendered DOM、管理 API 永不进入普通 MCP；
- 搜索结果返回 `article_id + article_version_id + canonical_url + provenance`；
- 限制单次全文读取大小和调用速率；
- 对多租户场景强制 tenant/site filter 下推。

## 11. Doctor 对 1000 站点场景的主要不足

### 11.1 没有 durable Frontier

BFS frontier 在 Crawl4AI 单次执行中存在，没有平台级 lease、priority、next_fetch_at、重试与跨 job 去重。

### 11.2 没有多 Discovery Provider 与 checkpoint

没有把 Feed、Sitemap、Archive、Common Crawl、Wayback 分成独立 Provider，也没有 source coverage/end condition/checkpoint。

### 11.3 页面 identity 不稳定

同 URL 重新抓取默认生成新 UUID；没有 URL hash/canonical/article alias/version 体系。

### 11.4 DuckDB 不适合作为平台级多 Worker 真相源

Doctor 在 `DatabaseOperations` 实例内部用 `asyncio.Lock` 控制写入，但这个 lock 只在**单 Python 对象/进程内**生效，并不能解决多个 worker/container 对同一个 DuckDB 文件的跨进程写协调问题。

### 11.5 DB + RQ 非原子

缺少 outbox，任务状态可能和 Redis 队列漂移。

### 11.6 主链路与 Embedding 耦合

Embedding 外部 API 失败会影响 page processing；生产系统应把 enrichment/indexing 与 canonical publish 隔离。

### 11.7 缺少抓取安全治理

核心路径没有体现生产级：

- robots policy；
- SSRF/Egress Policy；
- redirect hop 校验；
- DNS rebind 防护；
- domain token bucket；
- response byte budget；
- Browser 子资源限制。

### 11.8 不支持 immutable raw snapshot/replay

Doctor 主要存 `raw_text`，没有保存 HTTP raw bytes + rendered DOM + extractor release provenance，因此规则升级后无法完整离线 re-extract。

### 11.9 单父 hierarchy 会丢证据

同 URL 的多个发现入口无法完整表达；生产系统应保存 graph edge。

### 11.10 FTS/向量索引缺少版本生命周期

没有显式 `chunker_version / embedding_model_version / index_release / active generation / rebuild generation`，难以在线无损重建。

## 12. 对现有技术方案的直接优化

### 12.1 新增 Discovery Graph / Site Topology

保留 `frontier_urls` 作为 URL 节点，新增关系表：

```text
discovery_edges
- id
- site_id
- from_frontier_url_id nullable
- to_frontier_url_id
- relation(link/sitemap_child/feed_item/archive_entry/redirect/canonical)
- source_id
- snapshot_id nullable
- provider
- depth_hint nullable
- anchor_text_hash nullable
- first_seen_at / last_seen_at
- evidence_json
```

原则：

1. 同一个 URL 可以有多个 parent/source。
2. tree 是 graph 的 UI 投影，不是 identity。
3. 仅长期保存站内 accepted URL 之间、Sitemap 层级、redirect/canonical 等高价值边；普通正文超大量链接可以采样或只计数，防止边爆炸。
4. 每个 snapshot 的 edge 提取有 `max_edges_per_page` 硬预算。
5. `relative_path`/导出路径只由稳定 identity + slug 生成，不使用 parent title 拼接。

Web 管理端增加：

- Site Topology / Map；
- depth 分布；
- orphan URL；
- source-only URL；
- 多入口 URL；
- 页面 discovered-via 路径；
- Sitemap 树和 HTML 链接图切换；
- 某一归档分支突然断裂的 drift 对比。

这个能力直接提升“全量历史是否真的覆盖”的可解释性。

### 12.2 新增 Knowledge Access Plane

在 canonical 内容面之外增加派生查询面：

```text
article_version published
 -> index_outbox
 -> chunk worker
 -> lexical index
 -> embedding worker
 -> vector index
 -> generation ready
 -> search/MCP
```

数据模型建议：

```text
search_index_generations
- id
- kind(fts/vector/hybrid)
- backend
- chunker_version
- embedding_provider/model/dimension
- status(building/active/retired/failed)
- created_at / activated_at

search_chunks
- id
- generation_id
- site_id
- article_id
- article_version_id
- section_path
- chunk_index
- text_hash
- object_key_or_text_ref
- metadata_json
- created_at

article_index_states
- article_version_id
- generation_id
- lexical_status
- vector_status
- last_error
- indexed_at
```

关键约束：

1. 只索引**已发布 article_version**。
2. 新版本发布后异步创建新 chunks；active generation 切换后再清理旧版本索引。
3. 索引可以从 S3/PG 重建，因此不是业务真相源。
4. Embedding 失败不影响 Markdown 发布。
5. chunker/embedding model 变化创建新的 generation，支持 shadow evaluation 和原子切换。
6. 搜索默认 hybrid，优先 RRF，而不是直接相加不可比 raw score。
7. MCP 默认只读，明确排除 crawl/delete/admin。

### 12.3 Web 管理增加知识库视角

在现有抓取运维页面之外增加：

- article/version 搜索；
- 站点 Map；
- chunk preview 与 heading path；
- lexical/vector 命中对比；
- index generation 状态；
- embedding backlog/失败；
- article version 与 index version 对齐检查；
- MCP/Search API 调用审计。

## 13. 推荐落地顺序

### 第一步：先做 Discovery Graph

它依赖现有 Frontier、Discovery Source、Snapshot，数据天然已经在主链路里，而且能立即帮助判断全量历史覆盖与调试 URL 发现问题。

### 第二步：做 Article-Version 驱动的 FTS

先用 PostgreSQL FTS 或独立全文索引实现，不立即引入向量依赖。先验证检索字段、中文分词、title/heading boost 和 Web 搜索体验。

### 第三步：加异步 Embedding + Hybrid Search

对 `article_version` 做 AST-aware chunking，索引 generation 版本化；使用 RRF 合并 lexical/vector。

### 第四步：暴露只读 MCP

MCP 只开放 search/list/get article 等读取能力；不开放抓取、删除、规则发布、raw snapshot 等管理操作。

## 14. 最终判断

Doctor 值得吸收的是**产品与派生能力设计**，不是核心运行时架构：

- `hierarchy/map` → 升级为生产级 Discovery Graph / Site Topology；
- `DuckDB FTS + vector` → 升级为可重建、版本化的 Knowledge Access Plane；
- `FastApiMCP` → 保留“只读工具暴露、管理操作排除”的边界；
- `Crawl4AI BFS` → 继续作为 DeepCrawlProvider，而不是历史全量入口；
- `DuckDB + RQ + page UUID per crawl` → 不进入 1000 站点生产主链路。

现有技术方案在抓取、安全、增量、可追溯、调度方面明显比 Doctor 完整；本次真正应该补的是**站点拓扑可视化与知识查询出口**，从“能稳定采集 md”进一步提升为“能解释覆盖、能检索、能安全提供给 LLM 使用”的知识库平台。
