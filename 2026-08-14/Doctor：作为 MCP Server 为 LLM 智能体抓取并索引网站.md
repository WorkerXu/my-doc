# Doctor：作为 MCP Server 为 LLM 智能体抓取并索引网站

## 1. 调研对象与结论

- 项目：`sisig-ai/doctor`
- 地址：https://github.com/sisig-ai/doctor
- 调研日期：2026-08-14
- 阅读基线：GitHub `main`，tree SHA `4860bdf425613180762c0b8e40f54734ecb2f66b`
- 项目版本：`0.2.0`
- 许可证：MIT

Doctor 是一个把“网站抓取 → Markdown/文本 → 分块 → Embedding → 向量/全文检索 → FastAPI → MCP”串成闭环的轻量知识库项目。它最值得借鉴的不是 1000 站点生产调度，而是三类产品/架构思路：

1. **抓取结果除了正文，还保存站点层级信息并提供 Map 导航。**这使“为什么发现这个 URL、某个分支是否漏抓、页面处于什么结构位置”可以被 Web 管理端解释。
2. **抓取后的内容继续形成独立知识访问面。**Doctor 同时做全文检索、向量检索并通过 MCP 暴露给 LLM；这说明正式 Markdown 与检索索引应是两个生命周期。
3. **MCP 只读边界的方向是正确的。**Doctor 主动排除了抓取、任务进度、删除等操作，但其实现仍是“黑名单排除”，生产方案应进一步升级成“显式白名单暴露、默认拒绝”。

Doctor 本身不适合作为“约 1000 个技术博客、全量历史、长期增量”的生产核心：它以 DuckDB 为主数据/向量存储、Redis RQ 为任务队列，整次 Crawl4AI deep crawl 完成后再顺序处理页面，缺少 durable frontier、Provider checkpoint、conditional fetch、版本模型、transactional outbox、域级限流、分布式公平调度、安全出网、租户权限和长期重放能力。

因此对现有《博客知识库技术方案》的最终判断是：

- 保留 **PostgreSQL + Redis Streams + S3/MinIO + immutable snapshot + Article Version** 主链路，不使用 DuckDB/RQ 替换生产状态层。
- 将 Doctor 的 hierarchy/map 思路生产化为 **Discovery Graph / Site Topology**，但图是事实，树只是 Web 投影。
- 将 Doctor 的搜索/MCP 思路生产化为 **Knowledge Access Plane**，只消费已发布 `article_version`，异步构建 lexical/vector 索引。
- Crawl4AI BFS 继续作为 `DeepCrawlProvider`，不能作为历史全量唯一来源。
- MCP 必须显式 allowlist、只读、最小权限，并采用“搜索返回摘要/片段 → 按需读取具体文章/章节”的渐进式访问协议。
- 抓取内容对管理端浏览器和 LLM 都属于不可信输入：Web 预览防 stored XSS，MCP/Search 防 prompt injection 与越权工具诱导。

当前主方案已经吸收了 Site Topology、Knowledge Access Plane、AST-aware chunk、RRF、只读 MCP 等核心价值，因此本次补充重点是把 Doctor 实现里暴露出的**权限默认值、检索协议、安全渲染和索引幂等**进一步说明清楚。

## 2. 总体架构与依赖

README 描述的主链路是：

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

`pyproject.toml` 的关键依赖包括：

```text
crawl4ai==0.6.0
redis
rq
duckdb
langchain-text-splitters
litellm
openai
fastapi
fastapi-mcp==0.3.2
mcp==1.7.1
markdown
```

默认配置中：

```text
VECTOR_SIZE = 3072
DOC_EMBEDDING_MODEL = openai/text-embedding-3-large
QUERY_EMBEDDING_MODEL = openai/text-embedding-3-large
DEFAULT_MAX_PAGES = 100
CHUNK_SIZE = 1000
CHUNK_OVERLAP = 200
RETURN_FULL_DOCUMENT_TEXT = True
```

Docker Compose 只包含：

- `redis`；
- `crawl_worker`；
- `web_service`；
- 本地 `./data` 挂载 DuckDB；
- 本地 `./src` 开发态源码挂载。

这种部署适合个人、小团队或功能原型：组件少、启动快、抓取到 MCP 的反馈链路短。但单 DuckDB 文件 + 单机共享 volume 的模型天然更接近“本地应用”，而不是多机、多 Worker、数百万 URL 的长期数据平台。

## 3. Crawl4AI 深度抓取实现

核心文件：`src/lib/crawler.py`。

Doctor 使用：

```text
AsyncWebCrawler
CrawlerRunConfig
BFSDeepCrawlStrategy
PruningContentFilter
DefaultMarkdownGenerator
```

关键抓取参数：

```text
BFSDeepCrawlStrategy
- max_depth: 默认 2
- max_pages: 默认 100
- include_external: false
```

Markdown 生成器：

```text
PruningContentFilter(threshold=0.6, threshold_type="fixed")

DefaultMarkdownGenerator
- ignore_links: strip_urls
- body_width: 0
- ignore_images: true
- single_line_break: true
```

Crawler 配置还排除：

```text
nav
footer
aside
header
remove_overlay_elements = true
```

### 3.1 技术原理

BFS 从入口按链接层级扩散，天然产生 `depth`，适合短链路文档站或网站结构浏览。`include_external=False` 可以避免默认跨域扩散。`PruningContentFilter` 和排除导航标签能快速降低模板噪声，适合生成可搜索文本。

但 BFS 的“走完”不等价于博客历史全量：

- 历史文章可能只在 Sitemap 中，当前页面已无入链；
- archive 可能需要几十/几百页分页；
- tag、author、calendar、相关推荐会制造巨大旁路；
- 动态归档可能依赖 virtual scroll/load-more；
- 一个文章会同时从首页、分类、作者、标签、Sitemap 被发现；
- 迁站或改版后，旧路径可能完全从当前站内链接图消失；
- `max_depth/max_pages` 是预算上限，不是历史边界证明。

因此生产方案中 Crawl4AI BFS 应只作为一个 `DeepCrawlProvider`，与 Feed、Sitemap、Ordered Archive、普通 Archive、Common Crawl、Wayback 等 Provider 并列。

### 3.2 结果批量回收的内存模型

`crawl_url()` 调用 `crawler.arun()` 后，获得整个 `crawl_results` 列表，再交给后续处理。对于默认 100 页非常简单，但对大站回灌会出现：

- URL frontier 不 durable；
- 中断无法从细粒度 checkpoint 续跑；
- 整批结果驻留内存；
- Discovery/Fetch/Extract/Index 无法独立背压；
- 一个大任务占用 Worker 时间过长；
- 无法自然实现多站 weighted fairness。

生产方案应继续使用：

```text
Provider streaming discovery
 -> Frontier batch UPSERT
 -> Scheduler/lease
 -> Fetch Worker
 -> immutable snapshot
 -> Extract Worker
 -> publish article_version
```

## 4. hierarchy tracking 与 Site Map

### 4.1 `CrawlResultWithHierarchy`

`src/lib/crawler_enhanced.py` 把 Crawl4AI result 包装成 `CrawlResultWithHierarchy`，读取/生成：

```text
url
parent_url
root_url
depth
relative_path
title
```

其中：

- `parent_url` 和 `depth` 从 result metadata 读取；
- `root_url` 取本次抓取入口；
- 标题优先从 HTML `<title>` 获取；
- 再退化到 Markdown 第一条 H1；
- 再退化到正文前几行；
- 最后取 URL path。

标题解析使用正则读取 `<title>`，只替换少量 HTML entity。对于正式生产元数据，应该使用 DOM parser/metadata resolver，不把它当可靠 title extractor。

### 4.2 相对路径算法

Doctor 先建立 `url_to_result`，再按 parent 的 `relative_path` 加当前 `title` 拼接路径。

这个设计适合 UI breadcrumb，但不能成为文章 identity 或物理导出路径：

- 标题可修改；
- 标题可能重复；
- 标题含 `/`、特殊字符或超长字符；
- 同一 URL 可能有多个 parent；
- 抓取顺序改变可能改变第一次 parent。

生产方案应把 UI path 视为 projection，不参与 `article_key`、canonical 或导出路径。

### 4.3 数据库存储

Doctor 的 `pages` 表包含：

```text
id
url
domain
raw_text
crawl_date
tags
job_id
parent_page_id
root_page_id
depth
path
title
```

`processor_enhanced.py` 先按 `depth` 排序，保证父页先处理，再通过进程内 `url_to_page_id` 找 parent/root id。

### 4.4 Map Service

Map 相关接口：

```text
GET /map
GET /map/site/{root_page_id}
GET /map/page/{page_id}
GET /map/page/{page_id}/raw
```

`MapService` 支持：

- root page 列表；
- 同域多个 root 的 synthetic group；
- legacy page 按 domain 归组；
- parent/siblings/children/root 导航；
- breadcrumb；
- Markdown 页面浏览。

`build_page_tree()` 会读取一个 hierarchy 的全部 pages，构建 `page_map`，给每页挂 `children`，最终在内存中形成嵌套树。小站体验很好，但大站不应在一次 Web 请求中读取/构造几十万节点。

### 4.5 生产化：Discovery Graph，而不是永久树

Web 的真实关系是有环、多父图：

```text
link
sitemap_child
feed_item
archive_entry
redirect
canonical
alias
historical_source
```

同一个 URL 可以被多个 Provider、多条入边共同证明。因此推荐模型是：

```text
frontier_urls
frontier_url_sources
discovery_edges
```

树仅在查询时投影，例如：

- Discovery Tree；
- Sitemap Tree；
- Archive Tree；
- Link Graph；
- Identity Graph。

大站 Web UI 还应采用：

- cursor pagination；
- depth/node hard limit；
- lazy expand；
- 按 discovery run 生成可丢弃 projection cache/materialized view；
- 图生成失败不阻塞抓取主链路。

## 5. 页面持久化与增量能力

Doctor 的 `store_page()` 默认给每次写入生成新的 UUID，然后 `INSERT pages`。`pages` 没有 URL identity、canonical article key、snapshot/version 关系。

因此同 URL 重抓更接近“再写一条 page”，不是生产系统需要的：

```text
stable URL identity
 -> conditional fetch
 -> unchanged: seen/checkpoint update
or
 -> changed raw response
 -> immutable snapshot
 -> extraction_attempt
 -> article_version
```

核心抓取路径也没有完整展示：

- ETag / `If-None-Match`；
- Last-Modified / `If-Modified-Since`；
- 304 short circuit；
- raw hash/content hash/Markdown hash 分层比较；
- 删除确认状态机；
- Provider checkpoint；
- reconciliation；
- extractor/profile/version replay。

所以 Doctor 的 `pages` 不能替代生产方案中的：

```text
frontier_urls
fetch_snapshots
extraction_attempts
articles
article_versions
article_aliases
```

## 6. Redis RQ 任务模型与一致性

`src/crawl_worker/tasks.py` 中，`create_job()` 先向 DuckDB 写 job，再创建：

```text
Queue("worker").enqueue(perform_crawl, ..., job_timeout=600)
```

worker 把任务改为 running，调用完整 pipeline，结束后写 completed/failed。

### 6.1 优点

- 代码直观；
- Web 请求可以很快返回 job id；
- Redis/RQ 对小型异步任务足够；
- 失败状态有基础记录。

### 6.2 DB + Queue 双写窗口

流程是：

```text
DuckDB commit job
 -> Redis enqueue
```

两者不是一个事务。DB 已提交、Redis enqueue 失败，会留下 pending job；反向异常也可能状态错位。代码里显式 `CHECKPOINT` 并读回 job 可以提高 DuckDB 落盘确认，但不能解决跨系统原子性。

1000 站点方案继续采用：

```text
PG transaction:
 job/job_item + outbox
 -> commit
 -> Outbox Relay
 -> Redis Streams
```

再配 consumer group、pending reclaim、DB lease 与幂等状态转换。

## 7. Crawl、Page、Embedding 强耦合

`processor_enhanced.py` 对每页执行：

```text
extract_page_text
 -> store_page
 -> TextChunker.split_text
 -> generate_embedding
 -> VectorIndexer.index_vector
```

页面按 depth 顺序逐页处理；页内 embedding 最多 5 个并发分批执行。

这个实现易懂，但有几个生产问题：

1. 整次 crawl 已经先回收到内存；
2. 页面间主循环顺序处理；
3. 每页初始化 `TextChunker`、`VectorIndexer`，连接复用不理想；
4. 正文成功写入与 embedding 成功不处于清晰的独立状态机；
5. 外部 Embedding API 故障会拖慢整条 crawl pipeline；
6. 没有 generation 级重建/切换语义。

生产方案必须把 canonical plane 与 access plane 解耦：

```text
canonical:
fetch_snapshot
 -> extraction_attempt
 -> quality gate
 -> article_version published

access:
article_version published
 -> outbox index event
 -> chunk
 -> lexical index
 -> optional embedding
 -> vector index
 -> article_index_state
```

Embedding、FTS、Vector DB、MCP 不可用时，正式 Markdown 仍应发布成功。

## 8. Chunking 与 Embedding

### 8.1 Doctor 的 Chunking

`src/lib/chunker.py` 使用 `RecursiveCharacterTextSplitter`：

```text
chunk_size = 1000
chunk_overlap = 200
length_function = len
```

优点是通用、依赖成熟、实现简单。缺点是：

- `len` 是字符尺度，不是模型 token budget；
- 不理解 Markdown heading path；
- 代码块可能被截断；
- table/list/blockquote 可能被机械拆开；
- 20% overlap 在大规模语料里会显著放大 embedding 成本。

技术博客更合适：

```text
Markdown AST / Article IR
 -> heading section
 -> block-aware grouping
 -> token budget
 -> code/table atomicity guard
 -> semantic-boundary overlap
```

每个 chunk 至少保存：

```text
site_id
article_id
article_version_id
heading_path
section_path
chunk_index
text_hash
token_count
chunker_version
```

### 8.2 Embedding

Doctor 使用 LiteLLM/OpenAI，文档和 query 可以使用不同模型配置。这个 Adapter 思路值得保留，但生产系统还要记录：

```text
provider
model
model/revision
embedding_dimension
chunker_version
config_hash
input_hash
index_generation_id
```

并把 provider quota、batch size、retry、429/Retry-After、费用预算作为独立 Index Worker 控制参数。

### 8.3 索引幂等键

为了让 reindex/retry 不重复膨胀向量，推荐：

```text
chunk_id = hash(article_version_id + chunker_version + heading_path + chunk_index + text_hash)
vector_id = hash(index_generation_id + chunk_id)
```

写入采用 UPSERT 或先存在检查；`article_index_states` 明确区分 lexical/vector 的 pending/ready/failed/skipped。这样 worker 重试、消息重复投递和 generation 重建都不会生成不可控重复项。

## 9. DuckDB FTS + Vector Search

Doctor 使用 DuckDB 存：

```text
jobs
pages
document_embeddings
```

向量表：

```text
id
embedding FLOAT[VECTOR_SIZE]
text_chunk
page_id
url
domain
tags[]
job_id
```

`VectorIndexer` 加载 VSS，代码还定义 HNSW 索引 SQL。查询计算：

```text
array_cosine_distance(embedding, query_vector)
similarity = 1 - distance
```

DuckDB 的优势是：

- 单文件；
- SQL + FTS + vector 都能在一个进程里完成；
- 对 demo、本地知识库、离线分析非常方便。

但不适合作为当前 1000 站点方案的唯一在线状态中心：

- 多进程/多机写入协调能力不是其核心优势；
- 抓取状态与检索 workload 会互相干扰；
- 向量重建、在线查询和抓取写入耦合在同一个数据库文件；
- 高可用、租户隔离、横向扩展较弱。

因此生产起步可使用 PostgreSQL FTS + pgvector，规模/延迟/QPS 成为瓶颈时把**派生索引**迁往 OpenSearch/Qdrant/Milvus 等，PG 仍做业务真相源。

## 10. Hybrid Search 的实现与问题

`document_service.py` 并行发起：

```text
vector search
BM25/FTS search
```

向量分数先乘 `hybrid_weight`；BM25 使用固定经验除数 `10.0` 归一化，再乘 `(1-hybrid_weight)`。如果同一 page 两路都命中，最终不是相加，而是取两路加权分数的 `max`。

优点：

- lexical/vector 任一路失败，另一路仍能降级返回；
- 两路并行，延迟不会简单相加；
- 可兼顾精确词和语义召回。

问题：

1. BM25 raw score 不同数据规模/查询之间不可用固定 10 做稳定归一化；
2. cosine similarity 与 BM25 raw score 本来不是同分布；
3. “两路都命中取 max”并没有真正奖励交叉命中；
4. vector map 以 `page_id` 为 key，同一文章多个高质量 chunk 只保留一个；
5. 默认 `RETURN_FULL_DOCUMENT_TEXT=True` 可能把全文直接放入搜索响应，扩大网络、token 和 prompt-injection 暴露面。

生产方案默认使用 rank-based fusion，例如 RRF：

```text
score(d) = Σ 1 / (k + rank_i(d))
```

进一步可以采用：

```text
lexical topK + vector topK
 -> RRF
 -> metadata/time/site filter
 -> optional reranker
 -> per-article grouping/diversity
 -> snippets + stable ids
```

对于技术博客，应对 title、heading、code symbol、tag、domain/path 等字段加 boost，并保留原文 chunk，不把 LLM summary 当唯一索引文本。

## 11. FastAPI 与 MCP 暴露机制

`src/web_service/main.py` 使用：

```text
FastApiMCP(app, ...,
  exclude_operations=["fetch_url", "job_progress", "delete_docs"]
)
```

这体现了正确方向：不把抓取/删除直接暴露给 LLM。Doctor 的 MCP 主要提供：

```text
list_tags
search_docs
get_doc_page
list_doc_pages
```

### 11.1 黑名单排除的 fail-open 风险

Doctor 使用的是 `exclude_operations`。这意味着未来如果开发者新增：

```text
/admin/reindex
/admin/export_raw
/admin/publish_rule
/admin/secret_debug
```

而忘了追加 exclude，理论上新 operation 可能被 MCP 自动暴露。生产系统不能把“开发者记得排除”当安全边界。

推荐改为 **explicit allowlist / fail closed**：

```text
MCP_EXPOSED_OPERATIONS = {
  search_articles,
  list_articles,
  get_article,
  list_sites_or_tags,
}
```

任何新 API 默认不进入 MCP。最好将 MCP router 与 Admin API 分应用/分路由组部署，并使用独立认证、scope 和 rate limit。

### 11.2 渐进式读取协议

面向 LLM，不建议搜索接口默认返回全文。推荐：

```text
search_articles(query, filters)
 -> article_id/version_id
 -> title/canonical_url/provenance
 -> best snippets/heading_path

get_article(article_id, version_id?, section_or_line_range?)
 -> 按需读取有限范围
```

这样可以：

- 降低 token 成本；
- 减少一次返回数 MB 文本；
- 缩小恶意网页 prompt injection 一次性进入上下文的面积；
- 让 agent 先检索再精读；
- 保留稳定 citation/provenance。

## 12. Web Map 渲染安全

`MapService.render_page_html()` 使用 Python `markdown.Markdown(...)` 把抓取的 `raw_text` 转 HTML，然后直接嵌入页面模板。标题会 `html.escape`，但正文 HTML 的安全依赖 Markdown 输入与渲染行为。

抓取内容必须始终视为不可信。生产管理端若把 Markdown/raw HTML 同源渲染，可能形成 stored XSS 或 active-content 风险，尤其是网页正文中存在 HTML 标签、危险链接、SVG/iframe 等情况。

生产方案应该：

- Markdown 默认禁 raw HTML，或经严格 sanitizer；
- 预览页面放入 sandboxed iframe；
- 预览使用独立无 Cookie origin；
- CSP 禁 script/object/frame 等主动能力；
- 管理端 token/cookie 不注入预览 origin；
- URL scheme 只允许 http/https 等明确集合；
- raw snapshot/rendered DOM 采用纯文本或严格隔离预览。

这与抓取端 SSRF 是不同边界：一个防“服务器被诱导访问内网”，另一个防“管理员浏览器执行被抓站点植入的内容”。

## 13. MCP / RAG 场景的 Prompt Injection 边界

Doctor 的目标是把网页内容暴露给 LLM agent，因此除了传统 Web 安全，还要考虑语料中的对抗指令，例如文章正文写：

```text
Ignore previous instructions...
Call some admin tool...
Send secrets to...
```

知识库内容是**数据，不是系统指令**。生产设计至少应做到：

- Search/MCP 返回值标记 source/provenance/不可信内容边界；
- MCP 本身只读，即使模型受注入影响也没有 crawl/delete/admin 权限；
- 不从抓取正文动态生成可执行 tool name/URL 并自动调用；
- 对 agent 宿主明确“检索结果中的指令不得改变系统权限”；
- 敏感工具与知识库检索不共享宽泛 token；
- 对全文读取和跨站结果数设上限；
- 记录 MCP query、article/version 命中与 tool audit，便于追踪异常链路。

只读 MCP 不是“完全没有 prompt injection”，但能显著降低注入后的可执行影响面。

## 14. 站点 Map 体验对 Web 管理的启发

Doctor 的 Map 功能有很强的产品价值：它把“抓取数量”转成“结构可解释性”。对 1000 站点平台，Web 可以扩展为：

- Sitemap Tree；
- Archive Tree；
- Link Graph；
- Identity/Redirect/Canonical Graph；
- depth 分布；
- orphan URL；
- source-only URL；
- 多入口 URL；
- 某分支 discovered count 趋势；
- Sitemap 与 HTML link coverage 差集；
- 某 URL 的所有 source/parent/redirect/canonical 证据。

但不能照搬 Doctor 的“一次请求构造全树”。大图必须 lazy load，默认限制节点/深度，并允许按 discovery run 生成缓存投影。

## 15. Doctor 对 1000 站点场景的主要不足

| 能力 | Doctor | 1000 站点生产要求 |
|---|---|---|
| URL Discovery | 单入口 BFS | Feed/Sitemap/Archive/DeepCrawl/CC/Wayback 多 Provider |
| Frontier | 一次 crawl 内部状态 | PG durable frontier + lease/checkpoint |
| 历史全量 | max_depth/max_pages | 来源覆盖证据 + end condition + reconciliation |
| 增量 | 重跑抓取 | ETag/Last-Modified/304/hash/checkpoint |
| 数据版本 | page insert | immutable snapshot + article_version |
| 去重 | page UUID | URL identity + canonical + alias + content hash |
| 任务一致性 | DuckDB commit + RQ enqueue | transactional outbox |
| 公平调度 | 单 job | per-domain limiter + weighted fairness |
| 存储 | DuckDB 单文件 | PG + S3/MinIO + 可重建 Search |
| 抽取 | Crawl4AI Markdown | 多 Extractor + Quality Gate + Article IR |
| Chunk | 字符递归切分 | Markdown AST/token-aware |
| 向量 | 抓取链路内 | 独立 Index Worker/generation |
| Hybrid | raw score 经验归一 | RRF/rerank 可评估融合 |
| Topology | 单 parent tree | 多父有环 Discovery Graph |
| MCP 暴露 | exclude 黑名单 | explicit allowlist/fail closed |
| Web 内容安全 | 轻量 HTML 渲染 | sandbox/CSP/sanitizer/独立 origin |
| LLM 注入 | 未形成完整边界 | untrusted corpus + read-only tools + audit |
| 多租户 | 基本无 | tenant/site scope + auth + audit |
| 可观测 | job progress | SLO/OTel/质量/预算/drift/index freshness |

## 16. 对现有技术方案的具体吸收

### 16.1 Discovery Graph / Site Topology

Doctor 的：

```text
parent_page_id
root_page_id
depth
path
title
```

生产化为：

```text
frontier_urls
frontier_url_sources

discovery_edges
- site_id
- from_frontier_url_id
- to_frontier_url_id
- relation
- source_id
- snapshot_id
- provider
- depth_hint
- first_seen_at / last_seen_at
- evidence_json
```

边幂等键按 relation/source/provider 构造，普通 HTML link 设每页边预算。redirect/canonical 与 snapshot/version 证据绑定。

### 16.2 Knowledge Access Plane

索引只从已发布版本开始：

```text
article_version published
 -> index outbox
 -> AST-aware chunks
 -> lexical index
 -> optional embedding
 -> vector index
 -> article_index_state ready/partial
 -> generation shadow
 -> atomic active switch
 -> Search API / read-only MCP
```

索引可以全部删除再从 PG/S3 重建，不承担 canonical 内容真相。

### 16.3 Search Generation

建议模型：

```text
search_index_generations
- id
- kind
- backend
- chunker_version
- embedding_provider/model/dimension
- config_hash
- status(building/shadow/active/retired/failed)

search_chunks
- generation_id
- article_id
- article_version_id
- heading_path
- chunk_index
- text_hash
- token_count

article_index_states
- article_version_id
- generation_id
- lexical_status
- vector_status
- last_error_code
- indexed_at
```

新模型/维度/chunker/backend 参数变化必须新建 generation，shadow 验证后原子切换，不能在线原地覆盖半新半旧索引。

### 16.4 Search/MCP 契约

推荐只开放：

```text
search_articles(query, site_ids?, tags?, time_range?)
list_articles(site_id?, cursor?)
get_article(article_id, version_id?, section_or_line_range?)
```

显式禁止：

```text
crawl/fetch arbitrary URL
delete article/site
publish rule/profile/contract
raw snapshot/rendered DOM
secret/admin audit internals
```

并且接口注册必须使用正向 allowlist，不通过“排除若干 operation”实现权限边界。

## 17. 推荐落地顺序

### Phase A：保持抓取主链路不变

继续完善：

- Source Discovery；
- durable Frontier；
- conditional fetch；
- immutable snapshot；
- Extraction Contract；
- Article IR；
- deterministic Markdown；
- outbox/lease/limiter/security。

### Phase B：Site Topology

先记录多来源/多父 discovery evidence，再做 Web lazy projection，而不是先定义一棵永久树。

### Phase C：Lexical Search

只消费 `article_version`；先实现低成本全文搜索与稳定 `article_id/version_id/provenance` 返回协议。

### Phase D：AST-aware Chunk + Embedding

加入独立 Index Worker、chunk idempotency、provider quota/budget、generation lifecycle。

### Phase E：Hybrid/RRF + 评估

建设固定 query set，比较 lexical/vector/hybrid 的 recall、MRR/nDCG、延迟、成本；满足门禁后切 active generation。

### Phase F：只读 MCP

MCP 使用显式 allowlist，先开放 search/list/get；落实 tenant/site scope、rate limit、正文长度上限、审计和 prompt-injection 边界。

## 18. 最终判断

Doctor 值得吸收的是**产品闭环和派生能力设计**，不是其单机运行时：

- `Crawl4AI BFS` → 作为 `DeepCrawlProvider`，不能证明历史全量；
- `parent/root/depth + Map` → 升级为 Discovery Graph / Site Topology；
- `RecursiveCharacterTextSplitter` → 升级为 Markdown AST/token-aware chunk；
- `DuckDB FTS + vector` → 升级为可重建、版本化的 Knowledge Access Plane；
- 经验 BM25/vector 分数拼接 → 升级为 RRF/可评估 rerank；
- `FastApiMCP + exclude_operations` → 升级为 explicit allowlist / fail closed 的只读 MCP；
- `get_doc_page` 的按行读取思想 → 保留为渐进式 article/section 读取，搜索默认不返回全文；
- DuckDB + RQ + 每次抓取写新 page UUID → 不进入 1000 站点生产主链路；
- Web Map 的 Markdown HTML 预览 → 生产化时必须 sandbox/sanitize，避免抓取内容反向攻击管理端；
- MCP 面向网页语料 → 抓取内容明确视为 untrusted corpus，避免 prompt injection 扩大到管理工具。

现有技术方案在抓取完整性、增量、可追溯、任务一致性、安全与横向扩展上已经明显强于 Doctor。Doctor 带来的真正提升是让平台从“稳定抓取并生成 Markdown”进一步变成“**能解释覆盖、能按结构浏览、能高质量检索、能安全供 LLM 使用**”的长期知识基础设施。