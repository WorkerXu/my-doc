# 如何使用 Python 抓取网站，为 AI 训练数据和 RAG 流程提供数据——实现细节与技术原理分析

- 调研编号：17
- 原文：https://dev.to/zyvop/how-to-scrape-websites-for-ai-training-data-rag-pipelines-with-python-3b9l
- 原文主题：Crawl4AI 抓取 → Markdown 清洗 → 结构化切块 → Embedding → ChromaDB → RAG → 定时刷新
- 与当前需求的关系：文章给出了从网页到 RAG 的端到端最小闭环，但其示例主要适合单机、小规模、教学场景。对于约 1000 个技术博客的全量历史回灌、长期增量同步和 Web 管理，需要把示例中的进程内状态、URL 级覆盖、全量重刷、随机 Chunk ID、向量库直写等做法升级为可持久化、可审计、可重放的生产架构。

## 1. 文章实现链路拆解

文章的主链路可以抽象为：

```text
URL
  -> Crawl4AI
  -> clean Markdown
  -> Markdown heading split
  -> recursive character split
  -> sentence-transformers embedding
  -> ChromaDB
  -> similarity retrieval
  -> LLM answer with source URL
  -> fixed interval refresh
```

这条链路最大的价值不是具体库，而是把“抓取”和“知识消费”分开：先把网页转成可复用的知识对象，再由 RAG 使用。这个原则适合当前知识库需求；但文章把中间层简化成了 Python list/dict 和 ChromaDB，因此缺少历史完整性、版本、增量、失败恢复和多站点治理。

## 2. Crawl4AI 抓取部分

### 2.1 单页抓取

文章使用 `AsyncWebCrawler` 抓取页面，通过低字数阈值、overlay 移除和 iframe 配置得到 Markdown。其技术原理是：浏览器或 HTTP 页面获取完成后，先进行 DOM 清洗，再把正文结构转换成 Markdown，减少导航、广告、Cookie 弹窗等噪声，从而降低后续 Embedding 和 LLM 的无效 token。

这里要注意版本兼容。文章示例读取：

```python
result.markdown_v2.raw_markdown
```

而当前 Crawl4AI 官方文档已经说明 `markdown_v2` 自 v0.5 起移除，当前应读取：

```python
result.markdown.raw_markdown
```

当前官方 API 也更强调通过 `CrawlerRunConfig` 传递 `word_count_threshold`、`remove_overlay_elements`、`process_iframes`、`cache_mode`、Markdown generator 等运行参数。因此生产系统不能把第三方示例代码直接固化成业务 Contract，应建立 Crawl4AI Adapter，把不同 Crawl4AI 版本的输入参数、返回字段、错误和 Markdown 输出统一成平台自己的 Typed Result。

### 2.2 站点遍历

文章的整站示例本质上是一个 BFS：

```text
queue = [start_url]
while queue:
    url = queue.pop(0)
    crawl(url)
    extract internal links
    append new links to queue
```

技术上能工作，但只适合演示：

1. `list.pop(0)` 是 O(n)，队列增长后会持续搬移元素；至少应使用 `deque`，生产环境则应使用持久 Frontier。
2. `visited` 只在进程内，Worker 崩溃后全部丢失。
3. `queue`、`visited`、`pages` 都没有 lease/heartbeat/checkpoint，无法多 Worker 协作。
4. `max_pages` 只是停止条件，不能证明历史文章已经完整枚举。
5. `urlparse(url).netloc == base_domain` 过于简单，不能正确处理 `www`、多级子域、端口、IDN、registrable domain、canonical redirect 等边界。
6. 没有 URL canonicalization：fragment、tracking query、尾斜杠、大小写、redirect alias 可能造成重复抓取。
7. 没有 robots、per-host QPS、退避、429/503、熔断和站点公平性。
8. 没有 Sitemap、Feed、Archive、API 等更强的历史枚举 Provider，因此普通内链 BFS 很容易漏掉孤儿文章和深层归档页。

Crawl4AI 当前官方 Deep Crawl 已提供 BFS/DFS/Best-First、streaming、crash recovery、prefetch、link preview 等能力。这些能力适合做执行层加速，但仍不应该取代平台自己的 Coverage Truth 和 Durable Frontier：Crawl4AI 的 `resume_state` 可以作为一个 Task 内的执行 checkpoint，平台最终的 candidate/evidence/visited/task state 仍应落在 PostgreSQL。

### 2.3 Prefetch 与 Link Preview 的正确定位

当前 Crawl4AI 官方提供 `prefetch=True`，可只获取 HTML 和链接，跳过 Markdown、正文抽取、媒体和 LLM extraction；还提供 `link_preview_config` 对候选链接做 `<head>` 级预览和评分。

对当前 1000 站点方案有两个直接价值：

- `prefetch` 可以作为普通内链发现的 Fast Discovery Route，先快速产生 URL Candidate，再单独派发 Article Fetch；
- `link_preview_config` 可以作为 `METADATA_PROBE` 的一个执行 Adapter，只提取 title/meta/head 信息用于 Priority Hint 或分类。

但二者都不能成为 Coverage 证明。特别是 Best-First 或 score threshold，只应该改变“先抓谁”，不能在 `FULL_BACKFILL` 中因为低分删除合法历史候选。

## 3. Markdown 与内容清洗

文章直接把 Crawl4AI Markdown 作为下游输入，这在 Demo 中合理，但生产知识库应把 Markdown 定位为 Projection，而不是唯一真相。

推荐链路：

```text
Network Snapshot / Rendered DOM
  -> Extraction Candidate
  -> Canonical IR
  -> Accepted Document Version
  -> Markdown Projection
  -> Chunk Projection
  -> Search / Embedding Projection
```

原因：

- Markdown 生成规则以后会变化；如果只保存 Markdown，重新改 cleaner 时只能重新抓源站。
- 文章中的 Pruning/BM25 本质上是内容解释策略，不应和网络访问强绑定；保存 Snapshot 后可以离线反复调参。
- 对技术博客，代码缩进、表格、列表层级、引用和公式都属于正文数据，不能用通用 whitespace normalize 粗暴压平。
- BM25 “fit markdown”适合某个查询下的阅读/检索投影，不适合替代完整正文真相，否则会永久丢失当时查询不相关但以后可能需要的内容。

## 4. Chunking 实现与生产化问题

文章使用两级切块：先按 Markdown Header 分节，再对长节使用 `RecursiveCharacterTextSplitter`，例如 800 字符、80 字符 overlap。原理是先利用标题层级保留语义边界，再限制单块大小，减少向量检索时上下文过大或过碎。

这个方向正确，但生产系统需要进一步升级。

### 4.1 Chunk 必须是可重建 Projection

每个 Chunk 至少保存：

```text
chunk_id
source_id
document_id
document_version_id
chunk_projection_release_id
ordinal
heading_path
block_start / block_end
char_start / char_end
token_start / token_end
content
content_hash
language
created_at
```

`heading_path` 不能只保存 h2；应保存完整层级，例如：

```text
Python asyncio > Task > Error Handling
```

这样检索结果脱离页面也能保留结构上下文。

### 4.2 不应使用随机 UUID 作为 Chunk 业务身份

文章给 ChromaDB 的 ID 使用 `uuid.uuid4()`。这会导致相同 Document Version 重跑后得到完全不同 ID，使：

- 幂等 upsert 困难；
- Chunk diff 困难；
- Embedding cache 复用困难；
- 外部引用无法稳定定位；
- 重放和删除旧 projection 需要额外扫描。

推荐使用确定性 ID，例如：

```text
chunk_id = hash(
  document_version_id,
  chunk_projection_release_id,
  ordinal,
  normalized_chunk_hash
)
```

或者把 `document_version_id + projection_release + ordinal` 作为逻辑主键，同时独立保存 `content_hash`。

### 4.3 Chunk 尺寸不能写死为全局常量

800 字符 / 80 overlap 是教学参数，不应直接作为全站默认事实。不同语言、代码密度和模型 tokenizer 差异很大。应使用版本化 Chunk Policy：

- Markdown heading 优先；
- 保持代码块、表格、列表的原子性或受控拆分；
- 以 token budget 为主、字符数为兜底；
- 支持中英文不同阈值；
- 保存 split reason；
- 通过离线 retrieval benchmark 决定 release。

## 5. Embedding 与 ChromaDB 部分

文章使用 `sentence-transformers` 批量生成 embedding，并直接 `collection.add()` 到 ChromaDB。原理是把 Chunk 映射到向量空间，用 cosine distance 近似语义相关性。

对生产系统需要做以下分层：

### 5.1 Vector DB 只能是 Projection

ChromaDB、pgvector、OpenSearch kNN、Milvus、Qdrant 等都不能承担正文真相。Vector index 可删除重建，正文真相必须来自 Accepted Document Version + Chunk Projection。

### 5.2 Embedding 需要显式 Model Release

至少记录：

```text
embedding_model_release_id
provider/model
model_revision
dimension
tokenizer_revision
normalization
pooling
distance_metric
batch_config
```

模型升级导致维度或语义空间变化时不能原地混写。应使用新 namespace/index，完成 backfill 后切流，旧 index 再回收。

### 5.3 Embedding cache 以内容 hash 为核心

同一个 Chunk 内容在不同页面或同一页面不同版本中可能完全相同，应支持：

```text
embedding_cache_key = hash(chunk_content_hash, embedding_model_release_id)
```

这样标题元数据变化但正文 chunk 未变时可以复用 embedding。

## 6. 文章刷新逻辑的关键缺陷

文章的刷新逻辑每隔固定小时：

1. 重新爬整个 source；
2. 用 `source_url` 查询 Chroma；
3. 如果已有旧 chunk，就整 URL 删除；
4. 重新 chunk；
5. 重新 embedding；
6. 重新写入。

这对 1000 站点长期运行成本很高，而且会造成数据一致性问题。

### 6.1 固定周期全量重抓浪费网络资源

应先走 Change-Signal Plane：

```text
Feed updated/GUID
Sitemap lastmod
API cursor
ETag
Last-Modified
HEAD/metadata probe
    ↓
可能变化？
    ↓ yes
抓正文
    ↓
body / Canonical IR hash
```

只有真正发生内容变化时才创建新 Version 和后续 Chunk/Embedding。

### 6.2 删除旧向量再重新添加不是原子更新

如果 delete 成功而 embed/add 中途失败，检索会短暂甚至长期丢失该文章。正确方式是：

```text
new Document Version accepted
  -> build new Chunk Projection
  -> build new Embedding Projection
  -> projection ready
  -> atomic current-pointer/index alias switch
  -> old projection tombstone / delayed cleanup
```

这样失败时旧版本仍可服务。

### 6.3 URL 不是 Version Identity

文章通过 `where={source_url: page.url}` 识别旧数据，只能回答“这个 URL 有没有数据”，不能回答：

- 当前是哪一个 Document Version；
- 内容什么时候变化；
- canonical URL 是否变过；
- 一个文章是否曾从旧 URL redirect 到新 URL；
- 同内容是否被多个 URL 镜像。

因此必须分离 URL Identity、Document Identity 和 Document Version。

## 7. 增量 Chunk / Embedding 的推荐算法

真正适合当前知识库的更新算法是：

```text
1. Change Signal 命中
2. Fetch + Snapshot
3. Extract -> Canonical IR
4. 计算 canonical_ir_hash
5. 若 hash 未变：
     只写 Freshness Observation，结束
6. 若 hash 变化：
     创建新 Document Version
7. 用指定 Chunk Projection Release 生成 chunks
8. 与上一 Accepted Version 的 chunk 按 content_hash / structural key 做 diff
9. unchanged chunk：复用 embedding
10. added/changed chunk：生成 embedding
11. removed chunk：写 tombstone
12. 新 Projection 全部 ready 后切 Current Pointer
13. 异步清理旧 Search/Vector projection
```

这个算法把网络增量、正文版本、Chunk 增量和 Embedding 增量分别建模，避免文章示例中的“每次刷新都全部删除重建”。

## 8. RAG Retrieval 与引用

文章的检索只保存 `source_url/page_title/section`，回答时使用 `[Source N]`。这比无引用好，但不足以支持长期知识库的可解释性。

建议 Retrieval Result 至少包含：

```text
source_id
document_id
document_version_id
canonical_url
published_at / updated_at
chunk_id
heading_path
chunk_content_hash
projection_release_id
embedding_model_release_id
retrieval_score
retrieval_method
```

最终答案的 citation 应指向确定的 Document Version/Chunk，而不是只指向会持续变化的 URL。这样一周后重放同一 Analysis Manifest，仍能定位当时使用的文本。

## 9. ScrapeGraphAI / LLM Extraction 的正确位置

文章展示了自然语言描述字段，让 LLM 自适应页面结构。这种方式的优点是 selector 维护成本低，但缺点是成本、延迟和非确定性更高。

对当前方案更合适的策略：

```text
平台/API/RSS/Sitemap Fast Path
  -> deterministic CSS/XPath/Trafilatura/Readability
  -> Crawl4AI generic extraction
  -> site-specific browser recipe
  -> LLM structured extraction（必要时）
```

LLM 不宜作为所有文章正文的默认 cleaner。它更适合：

- 结构化字段补全；
- 非标准页面的 fallback；
- selector drift 后的辅助诊断；
- 复杂 schema extraction。

LLM 结果仍需绑定 Snapshot、Prompt/Schema Release、Model Release 和质量检查。

## 10. 对文章中性能数字的判断

文中给出抓取速度、Markdown 压缩比例、Embedding 吞吐和 Chroma 查询延迟等示例数字，这些不能直接用于 1000 站点容量承诺。实际瓶颈受以下因素影响：

- 源站礼貌限流；
- 网络延迟和 429；
- Browser 比例；
- 页面 JS 复杂度；
- 站点历史文章规模；
- 图片/附件；
- Chunk 数量和语言；
- Embedding 模型大小；
- Vector index 参数；
- Reconcile 频率。

容量规划必须用真实站点分布做 benchmark，指标按 Source/Route/Stage 分桶，而不是使用文章中的单一吞吐值。

## 11. 对现有博客知识库技术方案的具体优化

本轮文章调研建议把以下能力正式写入最终方案：

1. **Crawl4AI Runtime Compatibility Adapter**：固定并记录 Crawl4AI release，业务层只依赖平台 Typed Contract；对 `markdown_v2 -> markdown`、`CrawlerRunConfig` 等版本变化做 contract test。
2. **Prefetch/Link Preview 作为 Discovery Fast Path / Metadata Probe Adapter**：快速发现和评分，但不替代 Coverage Provider。
3. **结构化 Chunk Projection**：保存 heading path、block/token span、content hash、projection release，并使用确定性 Chunk ID。
4. **Embedding Delta Pipeline**：按 chunk hash + model release 缓存，新增/变化才生成向量；删除使用 tombstone，避免 URL 级 delete-all/re-add。
5. **显式 `RECHUNK` / `REEMBED` 运行类型**：Chunk Policy 或 Embedding Model 升级时从 Accepted Version 离线重建，不重新抓源站。
6. **RAG Citation Contract**：检索和回答必须引用 `document_version_id + chunk_id + heading_path`，不能只保留 source URL。
7. **Projection Ready + Pointer Switch**：新 Search/Embedding projection 完成后再原子切换 current，失败时旧 projection 可继续服务。
8. **Web 管理补充 Chunk/Embedding 页面**：查看某 Document Version 的 chunk diff、embedding model、projection release、stale/rebuild 状态和引用 lineage。
9. **向量库降级为可重建 Projection**：PostgreSQL/S3 仍保存真相；Chroma/pgvector/OpenSearch/Qdrant 等只作为检索实现。
10. **刷新从 fixed full recrawl 改为 Change Signal + Reconcile**：Feed/Sitemap/API/HTTP 条件请求触发正文抓取，周期 Reconcile 负责兜底。

## 12. 最终判断

文章非常适合作为“网页 → Markdown → Chunk → Embedding → RAG”的最小实现参考，其中 Header-aware chunking、clean Markdown、Embedding batch、source citation、定时刷新等方向都是正确的。

但其代码不应直接扩展到 1000 站点：进程内 BFS、URL 级状态、随机 Chunk UUID、固定时间全量重抓、先 delete 后 add、向量库承担唯一状态、只用 URL 引用，都无法满足长期知识库的完整性、幂等、版本化、增量成本和可重放要求。

生产方案应保留文章的“RAG-ready 内容投影”思想，同时把整个链路升级为：**Coverage 可证明、Frontier 可恢复、Snapshot 可重放、Version append-only、Chunk 可确定重建、Embedding 可增量复用、Vector index 可删除重建、RAG 引用固定版本、Crawl4AI 通过版本化 Adapter 隔离。**
