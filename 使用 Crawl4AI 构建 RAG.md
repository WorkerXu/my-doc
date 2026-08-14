# 使用 Crawl4AI 构建 RAG

## 1. 调研对象

- 项目：RAG_with_crawl4AI
- 地址：https://github.com/ssime-git/RAG_with_crawl4AI
- 主要技术：Python、Crawl4AI 0.6.2、Playwright、ChromaDB 1.0.7、Sentence Transformers、FastAPI、Streamlit、LiteLLM
- 项目定位：把网站抓取、Markdown 清洗、分块、向量入库、检索和 LLM 问答串成一套可运行的 RAG 示例。

该项目适合作为“小规模网站转 RAG 知识库”的参考实现，但不能直接承担“1000 个技术博客全量历史抓取 + 长期增量同步”的生产主链路。其最大价值不是现成代码，而是清晰展示了抓取层与 RAG 服务解耦、批量抓取、结构化分块、向量检索和 Web 查询的最小闭环；同时也暴露了生产化时必须补齐的 identity、版本、增量、幂等、覆盖证明和索引生命周期问题。

## 2. 总体架构与数据流

项目 README 给出的核心拓扑为：

```text
Crawler(insert_docs.py)
        |
        v
RAG Service(FastAPI) <---- Streamlit Web App
        |
        v
     ChromaDB
        |
        +---- LiteLLM / Gemini 用于最终生成
```

实际数据流可以拆成六步：

```text
URL / Sitemap / .txt
  -> Crawl4AI 抓取
  -> Markdown
  -> 按标题层级切块
  -> FastAPI /insert-documents
  -> ChromaDB embedding + HNSW
  -> /retrieve 语义检索
  -> /rag-query 拼接上下文
  -> LLM 生成答案
```

这种分层有一个值得保留的思想：**抓取/建库和查询/生成不是同一个运行时职责**。Crawler 只负责生产可索引文档，RAG Service 负责索引和查询，Web App 只做交互。生产方案应继续强化这种边界，而不是让爬虫 Worker 直接维护向量库或调用 LLM。

## 3. 抓取实现细节

### 3.1 URL 类型路由

`insert_docs.py` 先判断输入：

```text
.txt              -> crawl_markdown_file
包含 sitemap       -> parse_sitemap + crawl_batch
普通 URL           -> crawl_recursive_internal_links
```

这是一个轻量 Source Router。它的优点是简单，适合命令行单次任务；缺点是能力判断依赖 URL 字符串，而不是响应头、站点能力探测或版本化 Provider 配置。

生产实现应把这种判断升级为：

```text
Site Probe
  -> Sitemap Provider
  -> Feed Provider
  -> API Provider
  -> Archive Provider
  -> HTML Link Provider
  -> Browser Listing Provider
```

每个 Provider 都有独立 checkpoint、覆盖语义和 Release，而不是由 `is_sitemap()` 这类函数决定整个站点的历史抓取策略。

### 3.2 Sitemap 处理

项目使用同步 `requests.get()` 获取 Sitemap，再用 `ElementTree` 查找任意命名空间下的 `<loc>`。

优点：

- 对普通 `urlset` 足够直接；
- `{*}loc` 可以避免硬编码 Sitemap XML namespace。

生产问题：

1. 没有请求 timeout；
2. 没有 retry/backoff；
3. 没有 ETag/Last-Modified；
4. 没有 gzip sitemap；
5. 没有递归处理 sitemap index；
6. 没有 `lastmod` checkpoint；
7. 同步 `requests` 会阻塞 asyncio 主流程；
8. Sitemap 获取结果没有 Snapshot 和 Coverage Evidence。

因此生产方案中 Sitemap 必须是独立 Discovery Provider，并保存 sitemap index 层级、lastmod、条目数、终止原因和原始 Artifact。

### 3.3 普通站点递归抓取

`crawl_recursive_internal_links()` 维护：

```text
visited
current_urls
next_level_urls
```

每一层调用：

```python
crawler.arun_many(urls=urls_to_crawl, ...)
```

成功页面读取 `result.links["internal"]`，再进入下一层。

其本质是基于 Crawl4AI links 的分层 BFS。`urldefrag()` 去掉 fragment，可减少同页锚点重复；`visited` 在单次进程内避免重复。

这个实现有两个值得吸收的点：

- Crawl4AI `result.links` 很适合做 Link Discovery；
- `arun_many` + `MemoryAdaptiveDispatcher` 适合 Worker 内部受控并发。

但它不能证明历史全量：

- `max_depth=3` 是预算，不是覆盖证据；
- `visited` 只在内存，进程退出即丢失；
- 没有 URL Admission Rule，可能进入搜索、标签、日历、登录等 crawler trap；
- 没有 durable frontier、lease、retry、dead-letter；
- 没有保存“从哪个页面的哪个 href 发现该 URL”的 Link Evidence；
- 一个大站点的所有 URL 仍可能在某层一次性进入 `arun_many`，缺少跨 Worker backpressure。

生产设计应明确：`result.links` 只是 Discovery Evidence，真正任务必须落 PostgreSQL durable frontier；Crawl4AI Dispatcher 只负责 Worker 内资源调度，不承担全局业务状态。

### 3.4 Crawl4AI 并发与浏览器资源控制

项目为递归抓取和批量抓取配置：

```text
MemoryAdaptiveDispatcher
memory_threshold_percent = 60
max_session_permit       = max_concurrent
```

这是正确的局部资源保护方式。MemoryAdaptiveDispatcher 根据内存压力限制浏览器会话并发，比无界 `asyncio.gather()` 更适合页面抓取。

但项目所有主要页面都通过 `AsyncWebCrawler` 浏览器路径抓取，且 `CacheMode.BYPASS`，意味着：

- 静态文章也支付 Browser 成本；
- 重跑不会利用 HTTP 条件请求；
- 不能区分 Discovery 页需要 JS、Article 页不需要 JS 的情况；
- 对 1000 站长期同步成本偏高。

生产方案应保持 HTTP-first：普通文章优先 httpx/aiohttp，只有出现 hydration、动态列表、必须执行 JS、Golden Fixture 显示 HTTP 结果不完整等证据时升级 Crawl4AI/Playwright。

## 4. Markdown 分块实现与原理

项目的 `smart_chunk_markdown()` 使用层级切分：

```text
#
 -> ##
    -> ###
       -> 如果仍超过阈值，按字符硬切
```

默认每块小于 1000 字符。随后 `extract_section_info()` 保存：

```text
headers
char_count
word_count
chunk_index
source URL
```

优点：

1. 比纯固定长度切块更能保留 Markdown 章节边界；
2. Metadata 中保留 header，检索结果具备一定上下文；
3. 实现简单，适合文档站 PoC。

生产缺陷：

### 4.1 字符数不是模型 token 数

中文、英文、代码、URL 的 token/字符比例差异很大。1000 字符并不能保证 embedding 或 LLM 上下文预算稳定。生产应以 tokenizer 预算为主，并允许 `target_tokens/max_tokens/overlap_tokens` 配置。

### 4.2 硬切可能破坏语义结构

当三级标题内仍过长时直接 `h3[i:i+max_len]`，可能从代码块、表格、列表、链接、句子中间切断。生产应基于 Canonical Document IR 的 block 边界切块：

```text
heading path
paragraph
code block
table
list
quote
```

只在单个 block 超限时再做次级切分。

### 4.3 缺少父标题上下文继承

一个 `###` chunk 自身未必重复 `#` 和 `##` 父标题。生产 Chunk IR 应保存 `heading_path=[H1,H2,H3]`，索引文本可以显式前缀标题路径，而最终 Markdown 不必重复正文。

### 4.4 缺少 overlap 与语义窗口

连续段落跨 chunk 时可能丢失指代关系。可配置小比例 overlap，但 overlap 不能直接复制成独立“事实”；应记录 `start_block/end_block`，使引用能回到原始 Document Version。

## 5. 向量入库实现与原理

`db/chroma_client.py` 使用：

```text
chromadb.PersistentClient
SentenceTransformerEmbeddingFunction(all-MiniLM-L6-v2)
collection metadata hnsw:space = cosine
collection.add(...)
```

Chroma Collection 同时承担：

- chunk 文本；
- metadata；
- embedding；
- HNSW 相似度索引。

查询使用 `query_texts=[query]`，由 collection 的 embedding function 对查询生成向量，再按 cosine/HNSW 返回最近邻。

这个设计适合单机 RAG 原型，但不能把 ChromaDB 当作知识库业务真相源。生产中向量索引必须是 **Accepted Document Version 的 Projection**：删掉后可以重建，索引失败不能导致源文档丢失。

## 6. 项目中最关键的生产风险：Chunk Identity

`insert_docs.py` 为所有 chunk 生成：

```python
ids.append(f"chunk-{chunk_idx}")
```

而 `chunk_idx` 每次进程启动都从 0 开始。

这会导致严重问题：

1. 第二次导入不同网站仍生成 `chunk-0/chunk-1/...`；
2. `collection.add` 遇到已有 ID 时可能失败或产生冲突；
3. 无法判断某个 chunk 属于哪个 document version；
4. 文章更新后无法精确替换旧 chunk；
5. chunker 参数变化时无法建立新旧索引 lineage；
6. 无法安全做增量同步和离线重建。

生产 Chunk ID 应是稳定业务派生值，例如：

```text
chunk_id = sha256(
  document_version_id
  + chunker_release_id
  + start_block
  + end_block
  + normalized_chunk_hash
)
```

同时保留：

```text
document_id
document_version_id
chunker_release_id
embedding_release_id
heading_path
start_block/end_block
chunk_hash
token_count
```

这样可以做到幂等 upsert、索引重放、版本对账、精准删除和引用回溯。

## 7. Embedding 与索引版本问题

项目通过环境变量/参数选择 embedding model，但 Collection 本身没有显式的业务级 `embedding_release`。

对于长期知识库，Embedding 模型变化是必然事件。模型变化可能带来：

- 向量维度变化；
- 相似度分布变化；
- 召回质量变化；
- 同一 Collection 新旧向量不兼容；
- 全量重建耗时较长。

生产设计需要：

```text
embedding_release
- model/provider
- model_version
- dimension
- normalize policy
- distance metric
- tokenizer/version
- runtime_digest
- status
```

再引入：

```text
search_index_release
- chunker_release_id
- embedding_release_id
- backend
- index/schema config
- status=BUILDING|SHADOW|ACTIVE|RETIRED
```

模型升级使用蓝绿索引：

```text
ACTIVE v1
  -> 后台构建 v2
  -> completeness/quality 验证
  -> SHADOW 查询对比
  -> 原子切 ACTIVE=v2
  -> v1 延迟回收
```

避免在生产 Collection 内原地混写新旧 embedding。

## 8. 入库 API 的幂等与增量问题

RAG Service 的 `/insert-documents` 最终调用 `collection.add()`。这条链路没有：

- idempotency key；
- document version；
- upsert/delete lifecycle；
- source snapshot lineage；
- partial batch checkpoint；
- index completion manifest。

对于 1000 站增量同步，建议改成 Projection Job：

```text
Accepted document_version
  -> outbox INDEX_DOCUMENT_VERSION
  -> Chunk Projection Worker
  -> chunk_projection rows
  -> Embedding Worker
  -> vector index upsert
  -> index_projection_manifest
```

如果文章产生新版本：

1. 新 Document Version 先 Accepted；
2. 新 chunk 全部生成并索引；
3. manifest 完成后切换 current indexed version；
4. 旧 chunk 标记 stale/retired；
5. 异步删除旧索引记录。

不能“先删旧 chunk 再写新 chunk”，否则中途失败会造成检索空窗。

## 9. Retrieval 与 RAG Service 分层

项目 FastAPI 暴露：

```text
/health
/retrieve
/generate
/rag-query
/insert-documents
```

`/retrieve` 从 Chroma 召回内容并格式化为上下文；`/rag-query` 在召回后直接调用 LLM。

这一层拆分方向正确，但生产中建议进一步分成：

```text
Search API
  -> lexical FTS
  -> vector ANN
  -> metadata filter
  -> fusion
  -> optional reranker
  -> citation bindings

Answer API
  -> Search API
  -> context budgeter
  -> prompt release
  -> LLM
  -> cited answer
```

原因：搜索质量和生成质量必须能够独立测试、独立扩容、独立降级。LLM 不可用时搜索仍应可用。

项目 `rag_service/main.py` 在服务启动时检查 `GOOGLE_API_KEY`，这使本来只依赖 Chroma 的 retrieval/health 也与生成侧凭据耦合。生产中 Retrieval Plane 不应依赖 LLM Secret；Generate/Answer 能力应单独 readiness。

## 10. 检索质量优化

项目目前基本是纯向量 Top-K。技术博客知识库包含大量：

- 精确类名/函数名；
- 错误码；
- CLI 参数；
- 版本号；
- 文件路径；
- API 名称。

这些内容纯 embedding 不一定召回最佳。因此生产默认采用 Hybrid Search：

```text
PostgreSQL FTS / OpenSearch BM25
          +
Vector ANN(pgvector/OpenSearch)
          -> RRF/weighted fusion
          -> metadata filters
          -> optional reranker
```

过滤字段至少有：

```text
site_id
language
document_type
published_at
tags
document_version_id
accepted_at
```

还应建立 Retrieval Evaluation Dataset，跟踪 Recall@K、MRR/NDCG、citation hit rate、answer groundedness，而不是只看“能否返回 5 条结果”。

## 11. Web 管理可借鉴点

项目提供 Streamlit Web App，证明 RAG 查询层可以独立于抓取运行。对生产 Web Control Plane，建议不仅提供聊天框，还增加：

- Document -> Chunk 映射查看；
- Chunk heading path、token 数、chunk hash；
- embedding/index release；
- 某文档是否已完成索引；
- 新旧索引 shadow 查询对比；
- query lexical/vector/fused/rerank 各阶段结果；
- 召回 chunk 回溯到 document_version、source snapshot 和原文 Markdown；
- 重新切块、重新 embedding、重建索引操作；
- 索引 lag、失败、dead-letter、覆盖率和成本。

## 12. 对现有博客知识库技术方案的具体优化结论

本项目不会改变现有方案“PostgreSQL + S3 为事实源、Crawl4AI 只是执行层”的核心方向，但应补充一个明确的 **RAG/Search Projection Plane**。

### 12.1 新增数据模型

```text
chunker_release
chunk_projection
embedding_release
search_index_release
index_projection_job
index_projection_manifest
retrieval_eval_dataset
retrieval_eval_run
```

### 12.2 新增稳定 Chunk Contract

```text
chunk_id
document_id
document_version_id
chunker_release_id
heading_path
start_block/end_block
chunk_text_ref
chunk_hash
token_count
metadata
```

Chunk 是 Document Version 的 Projection，不是知识真相源。

### 12.3 新增索引生命周期

```text
BUILDING -> SHADOW -> ACTIVE -> RETIRED
```

支持双写、shadow compare、原子切换和可回滚。

### 12.4 新增索引幂等语义

索引写入使用 stable chunk ID + upsert；`index_projection_manifest` 记录预期 chunk 数、成功数、失败数和 index release。只有 manifest 对账完成才认为该 document_version 已可检索。

### 12.5 新增 Retrieval Plane

```text
FTS + Vector -> Fusion -> Filter -> Optional Rerank -> Citation Binding
```

查询结果必须绑定：

```text
chunk_id
document_version_id
source_url
heading_path
source span/block range
```

保证最终 RAG 答案能回到可审计原文，而不是只返回无来源的自由文本。

## 13. 最终评价

RAG_with_crawl4AI 是一个结构清晰的教学型闭环：它把 Crawl4AI、Markdown 分块、ChromaDB、FastAPI 和 LLM 组合成可运行系统，并正确展示了 `arun_many + MemoryAdaptiveDispatcher`、标题层级切块、RAG Service 独立化等实用模式。

但它的关键状态仍然是进程内/单机式的：`visited`、`chunk_idx`、固定深度、`collection.add`、单 Collection、单 embedding 配置，没有历史覆盖证明、durable frontier、document version、稳定 chunk identity、增量 checkpoint、索引 Release 和蓝绿重建，因此不能原样扩展到 1000 站长期知识库。

对目标方案最有价值的升级是：**把“向量库”从一个最终存储改造成可重建 Projection，把“分块”变成有版本、有稳定 ID、有来源 span 的 Chunk Contract，把 embedding/index 当作可发布 Release，并对每个 Document Version 用 Manifest 对账索引完整性。** 这样抓取、Markdown、搜索、Embedding、RAG 可以独立演进，同时仍保持端到端 lineage、增量幂等和可回滚能力。
