# LLM_trader／Semantic Signal LLM：基于新闻与 RAG 的语义交易信号系统

## 1. 调研对象与结论

- 编号：57
- 项目：LLM_trader / Semantic Signal LLM
- 地址：https://github.com/qrak/LLM_trader
- 调研基线：`master` 分支，提交 `8a0e60b2bb2638dbbd4212ce617a34111e35211f`

`LLM_trader` 本质是交易系统，新闻/RAG 只是其上下文子系统，不能直接承担“1000 个技术博客全量历史归档 + 增量同步 + Markdown 知识库 + Web 管理”。但它提供了几组非常有价值的工程原型：

1. **Feed-first + 按需正文补全**：RSS 已有正文时直接使用，正文不足才调用 Crawl4AI。
2. **多级降级与 stale-but-valid**：实时源失败时继续使用缓存，不把已有有效知识清空。
3. **显式相关性策略**：title/body/category/tag 使用不同权重，再叠加正文密度、关键词共现、实体相关性和 freshness。
4. **严格 Token budget**：上下文总预算和单文档预算都有约束。
5. **SHA-256 增量向量索引**：代码/Markdown 索引按文件 hash 跳过未变化输入。
6. **Embedding 兼容性探测**：能够检测向量维度不匹配并触发重建。

同时，它也暴露出平台化后必须修正的问题：批次总超时会丢弃已成功结果、整批 Browser 超时会重复 fallback、标题硬去重有误删风险、URL hash 被当文章 ID、SSRF 校验只覆盖部分场景、ContextBuilder 遇到超预算候选直接 `break`、向量索引采用“先删旧 chunk 再 upsert”存在失败窗口。

因此该项目适合作为“**增量采集、RAG 排序和投影增量化的参考实现**”，而不是历史知识库平台骨架。

## 2. 与知识库相关的调用链

重点源码：

- `src/rag/news_ingestion/rss_primitives.py`
- `src/rag/news_ingestion/rss_provider.py`
- `src/rag/news_ingestion/crawl4ai_enricher.py`
- `src/rag/news_ingestion/schema_mapper.py`
- `src/rag/news_manager.py`
- `src/rag/scoring_policy.py`
- `src/rag/context_builder.py`
- `src/rag/code_vector_index.py`
- `src/dashboard/server.py`
- `tests/test_rss_provider_contract.py`
- `tests/test_crawl4ai_enricher.py`

主要数据流：

```text
RSS Sources
   │
   ▼
RSSCrawl4AINewsProvider.fetch_news()
   │
   ├─ 并发 fetch_source()
   │      └─ RSS XML → normalized raw item
   │
   ├─ URL 去重
   ├─ 规范化标题二次去重
   ├─ 时间排序
   │
   ├─ body_text < min_chars ?
   │       ├─ yes → Crawl4AIEnricher
   │       │          ├─ AsyncWebCrawler + PruningContentFilter
   │       │          └─ 失败 → aiohttp HTML extractor
   │       └─ no  → 保留 Feed body
   │
   └─ schema_mapper.to_article_schema()
          │
          ▼
NewsManager
   ├─ 本地缓存 / fallback
   ├─ URL-first merge
   └─ body 更完整时替换短正文
          │
          ▼
ContextBuilder + ArticleScoringPolicy
   ├─ 字段/实体/分类/时间/质量评分
   ├─ full-body 优先
   └─ Token budget 装配
          │
          ▼
LLM Prompt
```

项目另有独立向量索引链：

```text
Python / Markdown files
   ↓
SHA-256 file hash
   ↓ unchanged → skip
AST / Markdown heading parsing
   ↓
semantic chunks
   ↓
SentenceTransformer batch embedding
   ↓
ChromaDB upsert
   ↓
semantic query
```

两条链分别提供“内容采集增量化”和“索引增量化”的参考。

## 3. RSS：结构化 Feed 既是发现源，也是正文候选源

`rss_primitives.py` 会读取 RSS 的 `title`、`link`、`guid`、`description`、`content:encoded`、`dc:creator`、`pubDate`、`category`。正文优先级是：

```text
content:encoded > description > empty
```

并记录 `body_source`。这点应直接迁移：Feed/Atom/JSON Feed 不应只提供 URL，还应产生低成本 `ContentCandidate`。

### 3.1 HTML 抽取是多级降级

项目优先 BeautifulSoup+lxml，删除 `script/style/noscript/svg/header/nav/footer/aside/form`，再尝试 article-body/article-content/post-content/entry-content/main 等 selector；失败后回退自定义 `HTMLParser`，最后才粗粒度 strip-html。

生产知识库应把这些输出作为不同 Extraction Candidate，并保存 extractor release、quality breakdown 和 provenance，而不是只保留最终字符串。

### 3.2 URL normalization 只能作为 URL Candidate 规则

项目会去 fragment、`utm_*`、`gclid`、`fbclid`、`mc_*` 等 tracking 参数并处理尾斜杠。这是有效的语法规范化，但不能直接决定 Document Identity。博客平台还要处理 redirect、`rel=canonical`、AMP/print、站点迁移、镜像域名和内容相似证据。

### 3.3 Feed 不能证明历史完整

项目面向最近新闻，Feed 每次只取有限条目。对多年历史博客必须明确：

```text
FeedItemObservation / Change Signal
≠ Coverage Exhaustion Evidence
```

Feed 适合增量同步；历史 Coverage 仍要依赖 API、Sitemap、Archive、Category/Tag/Author、Docs Navigation 等 Provider。

## 4. 多源异步并发：外层总超时是关键反例

`RSSCrawl4AINewsProvider._fetch_raw_items()` 使用：

```python
await asyncio.wait_for(
    asyncio.gather(*(fetch_source(...) for source in sources)),
    timeout=stage_timeout,
)
```

每个 Feed 自己能返回 `FetchResult` 隔离单源错误，但外层总超时后函数直接返回空列表。测试 `test_fetch_stage_timeout_returns_empty` 还把这一行为固定为契约。

如果 100 个站点已有 90 个完成、10 个未完成，平台不能丢掉前 90 个。正确语义是：

- 每个 Source/Provider Task 独立持久化；
- 完成一个立即提交一个；
- Run deadline 只影响未完成任务；
- Run 可以 `PARTIAL_SUCCESS`；
- 后续只重试失败任务。

`asyncio.gather()` 的生命周期不能成为业务事务边界。

## 5. Crawl4AI Enrichment：按质量升级成本

`Crawl4AIEnricher.enrich_items()` 只处理 URL 安全检查通过且 `body_text` 小于 `min_chars` 的条目，默认正文阈值约 400 字符。已有足够 Feed 正文时不启动 Browser。

应抽象为：

```text
Feed/API metadata
→ Feed/API body
→ HTTP page + deterministic extractor
→ Browser/Crawl4AI
→ Site Recipe / Manual
```

生产环境不应只看字符数，而要由版本化 Quality Policy 综合正文长度、文本密度、标题相关度、导航比例、错误页 marker、代码/表格/图片完整度、发布时间/作者/canonical、页面类型、Feed 与页面相似度等因素决定是否升级。

### 5.1 Browser 实例复用

项目一个 `AsyncWebCrawler` 批量处理 URL，通过 `semaphore_count` 控制并发，并把 worker 数限制在最多 6，避免每篇文章启动/销毁 Chromium。

平台应进一步把 Browser 拆成独立 Worker Pool：限制 page/context 数、内存、CPU、Browser seconds、单 domain 并发，并按 `max_tasks_per_browser`/生命周期 recycle。

### 5.2 多候选提取

Crawl4AI 输出按 `fit_markdown`、`raw_markdown`、`markdown_with_citations`、`cleaned_html/html` 尝试，并拒绝 404/page-not-found 等错误正文。说明必须区分：

```text
HTTP success
≠ Browser success
≠ Extraction success
≠ Quality Accepted
```

### 5.3 Browser 失败回退 aiohttp

项目在 Crawl4AI 未安装、Browser session 失败、batch timeout、单条失败、正文不足时回退 aiohttp。可迁移的是“能力降级”思想；生产平台必须把每次 route 持久化为 `FetchRouteAttempt`，而不是只在内存 `try/except` 中决定下一步。

### 5.4 整批 timeout 会重复 fallback

`arun_many()` 被外层 `wait_for` 包裹。整批超时后，targets 可能全部重新走 aiohttp，即使部分 Browser 请求已经成功。这会产生重复流量和成本，也丢失部分成功事实。

生产设计应逐 URL 落 `RouteAttempt/Snapshot`，只对明确未完成或质量失败的 URL fallback。

### 5.5 Windows Event Loop 兼容的启示

项目为 Windows SelectorEventLoop/Playwright subprocess 特别建立线程 + ProactorEventLoop。对平台更好的边界是：生产 Browser Worker 统一 Linux，Crawl4AI/Playwright runtime 通过 Adapter 隔离，不让平台差异污染 API/Scheduler 进程。

## 6. Prompt Text 与知识库 Markdown 必须分开

`Crawl4AIEnricher._clean_markdown_text()` 会删除图片、把 Markdown 链接退化为纯文本、去掉 heading marker，并压缩空白。这适合 LLM Prompt，却不适合作为长期知识库 Markdown。

必须保持：

```text
Canonical IR
├─ Markdown Projection      # 保留 heading/link/image/code/table/list
└─ Prompt Text Projection   # 面向 LLM 降噪
```

站点专用 tail marker 和 selector 也不能硬编码，应进入不可变 `SiteProfileRelease`，支持 Web 预览、diff、回滚和旧 Snapshot replay。

## 7. 去重和身份：可借鉴 URL-first，但不能硬合并标题

### 7.1 URL-first 与 Content Upgrade

NewsManager 合并时按 URL 去重；若旧正文很短而新正文更长，会用更完整正文更新。这一思想适合改造成：

```text
同 URL 新观察
→ ContentCandidate
→ Quality
→ Canonical IR compare
→ DocumentVersion(EXTRACTION_UPGRADE)
```

历史知识库不能原地覆盖旧正文。

### 7.2 normalized title 硬去重不可采用

项目把标题小写、去标点后作为二次去重键并保留正文更长的一条。短时新闻流可用，但长期博客中的 `Weekly Update`、`Release Notes`、`Changelog` 等标题会多年重复。

标题相同只能生成 `DuplicateHint`，必须结合 canonical、redirect、发布时间、内容相似度等 Evidence 决策。

### 7.3 URL 哈希不等于 Document ID

`make_article_id()` 使用 URL SHA-256 前 16 位。新闻缓存可以接受；博客站点迁移、slug 修改后不能继续把 URL 当逻辑文章身份。必须区分 URL Candidate、Document、Document Version、Chunk。

## 8. SSRF 与 Redirect 关联

项目能拒绝非 HTTP(S)、localhost、显式 private/loopback/link-local IP，但完整平台仍需要：DNS A/AAAA 解析后校验、redirect 每跳重验、DNS rebinding 防护、metadata ranges、端口/allowed origins、Browser 子资源 interception。

所有 Feed/HTTP/Browser/Asset 请求应统一走 Egress Policy。

另外，Crawl4AI batch 先按原 URL 建映射，返回结果后又使用 resolved/final URL 查 item；redirect 后可能匹配失败。平台必须始终以 `task_id/route_attempt_id/observation_id` 关联结果，final URL 只能作为 redirect/canonical Evidence。

## 9. Scoring Policy：可迁移的是策略隔离，不是交易新闻参数

`ArticleScoringPolicy` 把相关性计算从 ContextBuilder 编排中拆出来。默认字段权重：

```text
title      10
body        3
categories  5
tags        4
```

关键词频次使用：

```text
weight * (1 + log(1 + count))
```

比线性词频更能抑制重复关键词堆积。

总分还组合 category、entity/coin、important category，并乘正文密度、多关键词共现、实体相关性和 freshness modifier。通用原则是：标题命中可高于正文、正文完整度可作为排序信号、多 query term 共现提高置信度、domain/entity feature 可进入 query-specific scoring。

### 9.1 Freshness 不能照搬

`calculate_recency_factor()` 在 24 小时内线性从 1 降到 0，最终 freshness 乘数为：

```text
0.3 + 0.7 * recency
```

因此 24 小时后新鲜度加成消失但仍保留 0.3 底分。交易新闻合理，技术知识不合理。技术博客应按 Query Intent/Document Type/Source 配置不同 freshness policy，例如“最新 API 变化”偏新，而“算法原理”几乎不衰减。

### 9.2 Retrieval Policy 必须版本化

字段权重、BM25/Vector 权重、freshness、source priority、quality/full-body boost、reranker、context budget 都应进入 `RetrievalPolicyRelease`，支持 replay、A/B 和回滚。

## 10. ContextBuilder 与 Token Budget

ContextBuilder 先在较宽候选池按 relevance 排序，再把正文达到阈值的 full-body item 提前，最后在：

```text
article_max_tokens
max_context_tokens
```

约束下生成 prompt。过长正文会逐步裁剪并尽量回退到句子/段落边界。

可迁移原则：

- relevance 与 content quality 都参与 context assembly；
- total token budget 和 per-document budget 必须同时存在；
- 有结构化 Chunk 时直接以 chunk token count 装配，不应重新从全文字符串裁剪。

### 10.1 Oversize 候选的 `break` 是缺陷

当前 `build_context()` 遇到某候选加入后超出总预算时直接 `break`，会导致后面更短且有价值的候选永远不能进入上下文。

生产算法应：

```text
oversize candidate
→ 尝试缩小到允许 chunk/段落预算
→ 仍不合适则 skip
→ continue 后续候选
```

同时加入 source/document diversity 和每来源/每文档上限。

## 11. Stale-but-valid 缓存与查询热路径

NewsManager 在 Provider 返回空或异常时使用本地缓存 fallback。这一原则应升级为：

```text
源站暂时失败
≠ 文档删除
≠ 搜索索引失效
```

Accepted Version、Markdown、Search/Vector Projection 继续可用，只暴露 `source_lag`、`projection_age`、`stale`。

交易系统可在查询附近触发数据更新；1000 站博客知识库查询热路径不能同步等待抓站、Browser、全量 index rebuild。Query 只消费最近成功 Projection；若 stale，只创建/合并后台 refresh task。

## 12. CodebaseVectorIndexer：本次最重要的新增启发

`src/rag/code_vector_index.py` 使用 ChromaDB + SentenceTransformer，对 Python 做 AST 结构切块，对 Markdown 按 heading section 切块，并使用 SHA-256 文件 hash 做 delta indexing。

### 12.1 SHA-256 Delta Indexing

流程：

```text
current_file_hash == indexed_file_hash
    → skip
否则
    → parse chunks
    → batch embedding
    → upsert
```

这个原则应推广为知识库统一的 `Projection Input Fingerprint`：

```text
projection_input_hash = sha256(
    canonical_ir_hash
    + chunk_policy_release_id
    + embedding_release_id
    + projection_schema_release_id
)
```

输入和策略都没变，就不重建 Markdown/Chunk/Embedding。这样可显著减少百万文档级的无意义重处理。

### 12.2 结构感知 Chunk

Python 按 class/function/method/docstring，Markdown 按 heading section。核心不是 AST 本身，而是“按原生语义结构切分”。博客应按 heading path → block group → token ceiling，代码 fence、表格、列表尽量保持原子性。

### 12.3 Batch Embedding

项目优先批量 embedding（例如 batch size 32），失败再逐条。生产平台应保留 batch 能力，但 batch policy 要进入 `EmbeddingRelease/RuntimePolicy`，根据 provider 限流、token 数、CPU/GPU 自适应。

### 12.4 Embedding 维度不匹配检测

Indexer 会用当前模型生成测试向量查询旧 collection；检测到 dimension/incompatible 错误时重建 collection。

值得采用的是“模型指纹/维度变化必须显式检测”，但生产不能直接删除线上 collection。应使用 **Index Generation**：

```text
Active G1
  ↓ embedding release changes
Build Shadow G2
  ↓ count/checksum/sample query verify
Atomic alias switch G1 → G2
  ↓ rollback window
Retire G1
```

模型、revision、dimension、normalize、distance metric 都必须进入 `EmbeddingRelease`。

### 12.5 先删旧 Chunk 再 Upsert 有失败窗口

当前变化文件会先删除旧 chunks，再 parse/embed/upsert。如果 embedding 或 upsert 失败，旧索引已被删掉，产生空洞。

生产应使用稳定 chunk key + 成功后 tombstone，或直接构建 staging/shadow generation 后原子切换。

## 13. Dashboard 的可迁移设计

项目 FastAPI Dashboard 使用 Router 分层、WebSocket、Admin Router、日志流、可选认证、CORS、CSP/HSTS/Referrer-Policy/Permissions-Policy、rate limit 和生命周期管理。

博客平台可以借鉴“控制面 + 实时状态”的体验，但所有会影响抓取/抽取/质量/检索结果的配置不能直接热改全局 ini，而应发布不可变 Release。Web 至少展示 Source lag、Coverage、Run/Task、RouteAttempt、Browser 使用率、Extractor/Quality、Projection Generation、Retrieval Explain 和实时 worker/queue 状态。

## 14. 对博客知识库方案的具体优化

### 14.1 Projection Input Fingerprint

每类投影保存：

```text
source_document_version_id
projection_release_id
input_hash
state
built_at
```

只有事实输入或处理策略真正变化才重建。

### 14.2 Embedding Release

至少记录：

```text
provider
model
model_revision
vector_dimension
normalize
metric
batch_policy
created_at
```

### 14.3 Index Generation

至少记录：

```text
id
index_type
embedding_release_id
state
expected_count
actual_count
build_started_at
verified_at
activated_at
retired_at
```

Embedding 模型/维度/metric 改变时创建新 generation，验证后原子切换，不破坏当前线上查询。

### 14.4 Context Assembly Policy Release

明确版本化：

```text
total_token_budget
per_document_token_budget
max_documents
max_chunks_per_document
max_chunks_per_source
full_body_quality_threshold
diversity_policy
oversize_policy
citation_policy
```

oversize 必须 trim/skip + continue，不能阻塞后续候选。

### 14.5 Retrieval Explain

可选记录：

```text
lexical_score
vector_score
field_score
quality_modifier
freshness_modifier
source_modifier
rerank_score
final_score
retrieval_policy_release_id
```

Web 才能回答“为什么这篇排在前面”。

### 14.6 增量投影指标

新增：

```text
projection_hash_hit_ratio
chunks_reused_total
embedding_reused_total
embedding_rebuild_total
index_generation_build_seconds
index_generation_verify_fail_total
context_candidate_skipped_oversize_total
context_source_diversity
```

## 15. 可采用与不可直接采用

| 项目机制 | 结论 | 知识库落地方式 |
|---|---|---|
| RSS `content:encoded` 正文候选 | 采用 | Feed Observation + ContentCandidate |
| 正文不足才 Crawl4AI | 采用并增强 | Quality Gate 驱动 route escalation |
| Browser 批量复用 | 采用 | 独立 Browser Worker Pool |
| aiohttp fallback | 采用思想 | 持久化 RouteAttempt |
| source-specific tail marker | 采用 | 版本化 SiteProfileRelease |
| prompt-friendly Markdown 清洗 | 仅用于 LLM | 与 Markdown Projection 分离 |
| URL-first dedup | 部分采用 | URL Candidate 层使用 |
| normalized title 硬去重 | 不采用 | 只生成 DuplicateHint |
| URL hash 作为 article id | 不采用 | Document/Version/URL ID 分离 |
| 字段权重 + log 词频 | 采用思想 | RetrievalPolicyRelease |
| 24h 新闻 freshness | 不直接采用 | Query Intent/Document Type 策略化 |
| full-body 优先 | 采用 | Retrieval/Context feature |
| 严格 Token budget | 采用 | 基于 chunk token count |
| oversize 后 `break` | 不采用 | trim/skip + continue |
| SHA-256 增量向量化 | 强烈采用思想 | Projection Input Fingerprint |
| embedding 维度检测 | 采用 | EmbeddingRelease fingerprint |
| 维度不匹配直接删 collection | 不采用 | Shadow Index Generation + alias switch |
| 先删旧 chunks 后 upsert | 不采用 | generation/staging 或成功后 tombstone |
| 批次 `wait_for(gather)` 超时返空 | 不采用 | durable partial-result fanout |
| 本地文件 cache | 仅单机参考 | PG + Object Storage Truth Store |

## 16. 最终结论

LLM_trader 证明以下组合在真实应用中有效：

```text
低成本结构化源
+ 按需正文 enrichment
+ 多级 fallback
+ 显式 relevance scoring
+ Token budget
+ 向量语义检索
+ 缓存降级
```

但它的目标是“少量新闻源、短时间窗口、单应用交易决策”，而博客知识库目标是“1000+ 站点、多年历史、可证明 Coverage、长期增量、版本追溯、Web 运维、可重放处理”。因此应吸收其数据面技巧，再由平台级 Coverage Evidence、Task/Lease、Snapshot、Version、Policy Release、Projection Fingerprint 和 Index Generation 解决规模化可靠性问题。

本次最重要的方案优化是：**把项目的 SHA-256 delta indexing 上升为知识库统一的 Projection Input Fingerprint，并把其向量维度恢复逻辑升级为可验证、可回滚的 Index Generation；同时把 ContextBuilder 中隐式的全文优先与 Token 装配规则正式版本化。**
