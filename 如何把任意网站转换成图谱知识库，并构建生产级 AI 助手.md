# 如何把任意网站转换成图谱知识库，并构建生产级 AI 助手

## 1. 调研对象

- 编号：25
- 原文：How to Turn Any Website into a Graph Knowledge Base With A Production-Ready Co-Pilot
- 原文地址：https://todatabeyond.substack.com/p/how-to-turn-any-website-into-a-graph
- 作者页面标注日期：2025-10-07
- 核心组件：Crawl4AI、BFSDeepCrawlStrategy、FilterChain、URLPatternFilter、ContentTypeFilter、LXMLWebScrapingStrategy、LLMExtractionStrategy、Pydantic Schema、R2R / GraphRAG

本文展示了一条很典型的“网站 -> 深度抓取 -> 结构化抽取 -> 图谱知识库 -> AI 助手”链路。它对博客知识库方案最有价值的地方，不是照搬 R2R，也不是把所有页面交给 LLM，而是提供了三个可以独立吸收的设计思想：

1. 把站内遍历策略、URL 过滤策略、正文/结构化抽取策略拆成可组合配置；
2. 把结构化 Schema 当成抽取契约，而不是在业务代码里临时解析 JSON；
3. 在全文和向量检索之外，把知识图谱作为可重建的派生检索投影，而不是替代原始文档事实层。

对“1000 个技术博客全量历史文章 + Markdown 清洗 + 后续扩站 + 增量同步 + Web 管理”的目标而言，文章中的演示代码只能作为单站点 PoC，不能直接作为生产爬虫架构；但其 Deep Crawl、Filter Chain、Schema Extraction、Graph Projection 思路值得加入现有方案。

---

## 2. 文章实现链路拆解

### 2.1 深度抓取

文章使用 Crawl4AI 的 `BFSDeepCrawlStrategy` 做站内广度优先遍历，并同时设置：

- 最大深度；
- 最大页面数；
- 是否允许外域；
- URL 过滤链；
- 内容类型过滤。

BFS 的工作原理是按链接层级扩展 frontier：先处理入口页，再处理入口页直接发现的链接，再处理下一层。它适合层级较规则、文章详情页距离入口不深的网站，因为可以较快覆盖“离入口近”的页面。

但 BFS 本身不等于“全量历史发现”。博客可能有：

- sitemap 中存在但当前导航已不可达的旧文章；
- RSS 只保留最近几十篇；
- Archive 按年份分页；
- category/tag/author 页形成大量重复入口；
- JavaScript Load More；
- 旧 URL 已迁移或只在历史索引里存在；
- Docs/Blog 混合站点中大量非文章页面。

因此，生产方案中 Deep Crawl 只能是 Discovery Provider 之一，主要用于未知结构探索和补洞，而不能用“BFS 队列耗尽”证明历史覆盖完整。

### 2.2 FilterChain

文章用 `FilterChain` 组合 URL pattern 和 content type 条件，核心收益是“先阻止无价值 URL 进入昂贵抓取/抽取阶段”。

这一点对 1000 站点非常重要。抓取成本不是只有网络请求，还包括：

- Browser 秒数；
- HTML 解析 CPU；
- LLM token；
- 对象存储；
- Embedding；
- 搜索索引；
- 图谱实体关系抽取。

如果在 URL discovery 阶段就能排除明显的登录、搜索、标签分页、资源文件、重复参数页，可以显著降低后续放大成本。

但 FilterChain 也有高风险：过滤规则一旦写错，会静默漏文章。因此生产实现应把过滤分成两类：

```text
HARD_FILTER
- 明确禁止/无意义资源
- robots/policy
- 非允许域
- 明确静态资产
- 已知登录/后台路径

SOFT_FILTER
- URL pattern 推测
- 路径模板推测
- 相关性评分
- category/tag/navigation 推测
```

SOFT_FILTER 不应删除 URL Observation，而应保存“为什么被过滤”的 evidence，并允许在 Coverage 页面查看、抽样和重新放行。

### 2.3 结构化 Schema 抽取

文章定义 Pydantic 模型，再把 JSON Schema 交给 `LLMExtractionStrategy`，要求模型按固定字段返回结构化对象。

技术原理可以概括为：

```text
HTML
 -> chunking
 -> prompt + JSON Schema
 -> LLM structured generation
 -> JSON parse
 -> schema validation
 -> structured records
```

相较“让模型自由输出 JSON”，Schema 约束的优势是：

- 字段类型明确；
- 可以统一校验 required field；
- 输出可以直接进入程序化 pipeline；
- Schema 可以版本化；
- 可以对历史 Snapshot 重放；
- 可以统计字段缺失率和漂移。

文章还配置了 chunk token 阈值和 overlap。其原理是避免超长 HTML 一次进入模型上下文，同时通过少量重叠减轻语义被切断的问题。

但对于百万级技术博客文章，不能把 LLM Extraction 作为默认正文抽取方式。主要原因：

1. token 成本随 HTML 体积线性放大；
2. 吞吐受模型 API 限制；
3. 输出存在非确定性；
4. Schema 变化会触发大规模重放；
5. 文章正文的 Markdown 清洗并不需要 LLM 才能完成。

更合理的生产顺序是：

```text
平台 API / JSON-LD / CSS / XPath / Regex
 -> Trafilatura / Readability / Crawl4AI deterministic extraction
 -> quality validation
 -> LLM structured extraction only for complex/unstructured fields
```

当前 Crawl4AI 官方文档也提供 JSON CSS、XPath、Regex 等无需 LLM 的结构化抽取策略；LLM 适合复杂、缺乏稳定 DOM 规则的场景。因此应把 LLM Extraction 定位为“高成本可选 Projection”，而不是 canonical Markdown 的必要依赖。

### 2.4 LXMLWebScrapingStrategy 与 Browser 的关系

文章虽然通过 Crawl4AI 执行抓取，但内容处理使用 LXML 路线。对生产系统的重要启示是：

- Crawl4AI 不应被理解成“所有页面都必须打开真实浏览器”；
- 深度发现、抓取路由、解析和 Markdown 生成应该拆开；
- 静态 HTML 优先走 HTTP/LXML；
- 只有 JS 必须执行、动态分页、交互后才能得到正文时才升级 Browser。

这与“HTTP First, Browser Last”原则一致。

### 2.5 Token 使用量与成本

文章演示了统计 LLM extraction token usage。生产系统不能只打印日志，而应形成可查询的成本事实：

```text
structured_extraction_usage
- source_id
- document_version_id
- extraction_release_id
- model_release_id
- input_tokens
- output_tokens
- cached_tokens
- latency_ms
- estimated_cost
- outcome
```

并支持按 Source、Schema、Model、日期、成功文档数聚合，最终形成：

```text
llm_extract_cost_per_accepted_document
llm_extract_tokens_per_document
llm_extract_schema_failure_rate
```

这样才能判断某个站点是否应该继续使用 LLM Extraction，还是投入一次性成本编写 CSS/XPath Schema 更划算。

---

## 3. Crawl4AI Deep Crawl 的生产化理解

Crawl4AI 当前官方文档把 Deep Crawl 设计为可插拔策略，包含 BFS、DFS、Best-First，并支持 URL/domain/content relevance 等过滤。

### 3.1 BFS

优点：

- 层级覆盖直观；
- 适合从博客入口向文章页扩散；
- 容易设置 max depth / max pages。

缺点：

- 在导航很多的网站上，低层级无关页会占用大量预算；
- 不知道哪条链接更值得优先；
- 无法证明站点历史完整。

### 3.2 DFS

优点：可以快速深入某条路径。

缺点：容易被无限分页、日历、参数路径拖入局部深井，不适合作为默认历史发现策略。

### 3.3 Best-First

对博客知识库更有潜力。可以根据 URL、anchor、路径模式、时间、已知 article template 等给候选打分，优先抓“更像文章”的链接。

生产中建议：

```text
authoritative providers first
 -> sitemap/feed/CMS/archive
 -> URL template clustering
 -> best-first gap discovery
 -> bounded BFS for unknown surfaces
```

Deep Crawl 的 frontier 不应只存在 Worker 内存。平台应把“被发现 URL”及时变成 durable Observation/Task；Worker 崩溃后可以恢复，而不是重新从首页开始。

---

## 4. Graph Knowledge Base 的技术原理

文章后半部分把抓取结果交给 R2R 构建知识图谱。R2R 官方项目目前把 Knowledge Graph、Hybrid Search、Document/Collection Management 和 Agentic RAG 作为核心能力之一。

对本项目而言，不建议让 R2R 或任意图数据库成为文档事实真相。更稳妥的边界是：

```text
Snapshot / Canonical IR / Document Version = Truth
Markdown / Chunk / Embedding / Graph = Projection
```

### 4.1 图谱实体

技术博客语料可抽取的实体包括：

- 人物/作者；
- 公司/组织；
- 开源项目；
- 编程语言；
- 框架/库；
- API/协议；
- 数据库；
- 模型；
- 产品；
- 论文；
- CVE/漏洞；
- 技术概念。

### 4.2 图谱关系

例如：

```text
PROJECT --USES--> LIBRARY
PROJECT --DEPENDS_ON--> DATABASE
ARTICLE --MENTIONS--> PROJECT
PERSON --AUTHORED--> ARTICLE
COMPANY --MAINTAINS--> PROJECT
TECHNOLOGY --ALTERNATIVE_TO--> TECHNOLOGY
VERSION --FIXES--> CVE
ARTICLE --CITES--> ARTICLE/PAPER
```

### 4.3 Provenance 是关键

每个实体和关系都必须能回到原文证据，否则图谱很快变成不可审计的“LLM 生成数据库”。

建议：

```text
graph_entity
- graph_generation_id
- entity_id
- canonical_name
- entity_type
- aliases
- confidence

graph_relation
- graph_generation_id
- relation_id
- source_entity_id
- predicate
- target_entity_id
- confidence

graph_evidence
- relation_or_entity_id
- document_version_id
- chunk_id
- block_range
- quote_hash
- extraction_release_id
- model_release_id
- prompt_release_id
```

`quote_hash` 用于稳定引用证据，不要求数据库重复存大段正文。

### 4.4 Entity Resolution

真正困难的不是“让 LLM 抽出实体”，而是同一实体的归并：

- `Postgres` / `PostgreSQL`；
- `Node` / `Node.js`；
- `OpenAI API` 与 `OpenAI`；
- GitHub repo rename；
- 公司名和产品名相同；
- 同名开源项目。

因此应把实体识别和实体归一拆开：

```text
Mention Extraction
 -> Candidate Entity
 -> deterministic normalization
 -> alias lookup
 -> identifier match (URL/GitHub/package/CVE/DOI)
 -> embedding/name similarity
 -> optional LLM adjudication
 -> canonical entity
```

优先使用稳定外部 ID，例如 GitHub repo URL、package identifier、DOI、CVE ID，而不是仅靠名称相似度。

---

## 5. GraphRAG 不应替代 Hybrid Search

图谱检索擅长：

- 多跳关系；
- 关联项目/人物/技术栈；
- 聚合关系；
- “谁依赖谁”“哪些项目共同使用某组件”这类结构问题。

BM25 擅长：

- 精确技术 token；
- 错误信息；
- API 名称；
- 版本号；
- 代码符号。

Vector Search 擅长：

- 语义近似；
- 同义表达；
- 自然语言问题。

因此推荐：

```text
Query
 ├─ BM25 recall
 ├─ Vector recall
 └─ Graph seed + bounded expansion
        ↓
     graph-linked chunks
        ↓
 hard scope / ACL filter
        ↓
 fusion
        ↓
 optional reranker
        ↓
 context assembly
```

Graph Retrieval 必须有预算：

```text
max_hops
max_seed_entities
max_nodes
max_edges
max_linked_chunks
timeout_ms
```

否则高连接实体会产生候选爆炸。

同时，Graph channel 不能绕过现有 Scope。用户只查某个 Source/Document 时，图扩展后进入上下文的 Chunk 仍必须满足相同 scope/ACL。

---

## 6. 对现有博客知识库方案的具体优化

### 6.1 增加 URL Filter Release

现有 Site Profile 中虽然有分类和规则，但应明确增加有序 URL Filter Chain：

```text
url_filter_release
- id
- version
- rules[]
- rule_type
- action            # HARD_REJECT / SOFT_REJECT / DEPRIORITIZE / ACCEPT
- reason_code
- fixture_refs
- created_at
```

所有 filter decision 写入 evidence，支持 Web 模拟和抽样。

### 6.2 增加 Structured Extraction Release

把“输出 Schema”和“执行策略”分离：

```text
structured_schema_release
- schema_json
- required_fields
- field_descriptions
- validation_policy

structured_extraction_release
- engine            # CSS/XPATH/REGEX/LLM
- input_format
- schema_release_id
- selector_or_instruction
- chunk_policy
- model_release_id nullable
- prompt_release_id nullable
- quality_policy
- runtime_release_id
```

这样同一 Schema 可以分别用 CSS、XPath、LLM 实现，并在 fixture 上比较成本、正确率和稳定性。

### 6.3 增加 Structured Record Projection

```text
STRUCTURED_RECORD
```

它和 canonical Markdown 并列，都是 Document Version 的派生物。结构化抽取失败不影响 Markdown READY。

### 6.4 增加 Graph Projection

新增：

```text
ENTITY_MENTION
GRAPH_ENTITY
GRAPH_RELATION
GRAPH_COMMUNITY nullable
GRAPH_SUMMARY nullable
```

Graph index/build 必须使用 Generation：

```text
BUILDING -> VALIDATING -> READY -> ACTIVE -> RETIRED
```

不能在生产图谱上边跑新模型边原地覆盖旧关系。

### 6.5 增加 Graph Release

```text
graph_projection_release
- entity_schema_release_id
- relation_schema_release_id
- extraction_release_id
- entity_resolution_release_id
- community_release_id nullable
- graph_storage_adapter_release_id
- runtime_artifact_release_id
```

### 6.6 图数据库选型

首期不需要为了“有图谱”立刻引入复杂集群。

可以分阶段：

1. 百站以内：PostgreSQL 保存实体/关系/evidence，验证价值；
2. 图查询成为核心能力后：增加 Neo4j/Memgraph/其他图引擎作为可重建 projection；
3. R2R 可作为实验性 Graph/RAG Adapter，但不能接管 Source、Coverage、Document Version、Task 等平台真相。

这样避免同时维护两套文档生命周期和权限模型。

---

## 7. 1000 个技术博客场景下的端到端实现

```text
Source Onboarding
 -> Probe
 -> Authoritative Discovery
 -> URL Filter Chain
 -> URL Observation
 -> Normalize / Resolve / Resource Probe
 -> HTTP-first Fetch
 -> Snapshot
 -> Deterministic Extraction
 -> Canonical IR
 -> Markdown
 -> Document Version
     ├─ Chunk -> BM25
     ├─ Chunk -> Embedding -> Vector
     ├─ Structured Extraction -> Structured Record
     └─ Entity/Relation Extraction -> Graph Projection

Query
 -> BM25 + Vector + optional Graph
 -> scope/ACL enforcement
 -> fusion/rerank
 -> context
 -> answer
```

其中 Source Sync 的成功条件到 Document Version/Canonical Markdown 即可。Graph、Embedding、LLM Structured Extraction 都允许异步积压。

这点极其重要：如果图谱模型服务故障，不能导致博客增量同步停止。

---

## 8. Web 管理功能应补充

### 8.1 URL Filter 调试

对任意候选 URL 展示：

- 命中了哪条规则；
- HARD/SOFT；
- decision；
- rule release；
- 是否曾被其他 Provider 发现；
- 抽样放行后的结果。

支持输入一组 URL 运行新旧 Filter Release diff，防止发布规则后大面积漏文章。

### 8.2 Structured Extraction 调试

展示：

- 输入 Snapshot；
- Schema；
- engine；
- 输出 JSON；
- validation error；
- token usage；
- latency；
- fixture pass rate；
- CSS/XPath 与 LLM 输出 diff。

### 8.3 Graph Explorer

展示：

- Entity；
- Relation；
- 证据 Document/Chunk；
- confidence；
- Graph Release；
- Entity Resolution 过程；
- relation diff；
- source scope。

管理员应能把误归并实体拆开、添加 alias 或标记关系为 quarantine，但这些操作必须形成版本化 correction，而不是直接无审计修改图数据库。

### 8.4 Graph Retrieval Trace

Search/Ask 调试台增加：

```text
GRAPH_SEED
GRAPH_EXPAND
GRAPH_LINKED_CHUNK
GRAPH_FUSION
```

并显示 hop、edge、来源实体、对应证据和最终进入上下文的 chunk。

---

## 9. 可靠性与成本边界

### 9.1 Deep Crawl 预算

每个 Source/Profile 配置：

```text
max_discovered_urls
max_fetch_urls
max_depth
max_pages_per_surface
max_duplicate_ratio
max_wall_clock
max_browser_seconds
```

触发预算只表示 `known_gap / budget_exceeded`，不能表示历史完成。

### 9.2 LLM Extraction 预算

```text
max_llm_extract_documents_per_run
max_input_tokens_per_document
max_cost_per_source_per_day
max_schema_retry
```

超预算时 Structured Projection 进入 backlog，不影响 canonical 文档。

### 9.3 Graph 预算

图谱构建应限制：

- 单文档实体数；
- 单文档关系数；
- 单实体 alias 数；
- 关系置信度阈值；
- entity resolution 候选数；
- 图查询最大 hop/node/edge。

---

## 10. 测试与发布门禁

### 10.1 URL Filter Recall Test

建立已知文章 URL fixture。新 Filter Release 必须保证已知文章 recall 不下降到阈值以下。

指标：

```text
article_url_recall
non_article_reject_rate
soft_filter_false_negative_rate
```

### 10.2 Structured Extraction Contract Test

每个 Schema 至少测试：

- required field；
- 类型；
- 空值；
- 多 item；
- 超长输入 chunk；
- DOM 变化；
- 模型无效 JSON；
- 模型超时；
- deterministic engine 与 LLM engine 差异。

### 10.3 Graph Provenance Test

任何 ACTIVE relation 必须至少有一个有效 evidence 指向 Document Version/Chunk。

### 10.4 Entity Resolution Test

固定容易混淆的实体集，评估 merge precision / split recall，防止版本升级后把同名项目错误合并。

### 10.5 Graph Scope Leak Test

DOCUMENT/SOURCE/ACL scope 下运行图谱多跳检索，最终上下文不得含 scope 外 Chunk。

### 10.6 Graph Release Replay

同一批 Document Version 使用新 Graph Release 离线重建，比较：

- entity count；
- relation count；
- unsupported relation rate；
- entity merge/split drift；
- query benchmark；
- token/cost。

验证通过后切换 Graph Generation。

---

## 11. 为什么不直接照搬文章方案

文章是一个很好的端到端 Demo，但与 1000 博客生产系统存在几个规模和语义差异。

### 11.1 Demo 的 `max_depth/max_pages` 不是 Coverage

文章为了演示只抓极少页面。生产系统需要 Provider evidence、cursor、exhaustion reason 和 known gap。

### 11.2 电商 Schema 与技术博客正文不同

书籍详情字段高度结构化，而博客正文的核心目标是完整保留 heading、段落、代码、表格、图片和链接。博客 canonical Markdown 应以确定性 IR 为核心；Schema extraction 是补充。

### 11.3 全量 LLM Extraction 成本不可接受

稳定模板应使用 CSS/XPath/Regex；LLM 只解决复杂字段或长尾站点。

### 11.4 Graph 不是唯一检索答案

技术文档包含大量精确 token、版本、错误字符串和代码符号，BM25 仍不可替代。Graph 应作为第三召回通道。

### 11.5 R2R 不应成为第二套平台真相

R2R 的 Hybrid Search、Knowledge Graph、Document Management 很适合做独立 RAG 系统，但本项目已经需要 Source、Coverage、Snapshot、Version、Incremental Sync、Web Ops、Release、Audit 等完整控制面。如果再让 R2R 持有独立 canonical 文档生命周期，会产生双写、删除语义、权限和版本漂移问题。

最合理的方式是：把 R2R/图数据库置于 Adapter/Projection 边界，必要时可替换。

---

## 12. 本次方案结论

本次调研对现有方案的优化应落在以下能力：

1. URL Filter Chain 正式版本化，并区分 HARD/SOFT decision；
2. Deep Crawl 保留 BFS/DFS/Best-First Adapter，但只用于探索和 gap filling；
3. Structured Schema 与 Structured Extraction Strategy 分离；
4. 新增 STRUCTURED_RECORD Projection，LLM extraction 不进入 canonical 关键路径；
5. 新增 Graph Projection、Graph Release、Graph Generation、Graph Provenance；
6. GraphRAG 作为 BM25 + Vector 之外的可选第三召回通道；
7. 图谱检索同样执行硬 Scope/ACL；
8. R2R 可做 Graph/RAG Adapter 或实验 backend，不成为业务 truth；
9. Web 增加 Filter、Schema、Graph、Token Cost、Graph Trace 管理；
10. 发布门禁增加 URL recall、Graph provenance、Entity Resolution、Graph ScopeLeak 和成本回归。

这些优化可以获得文章“深度抓取 + 结构化抽取 + GraphRAG”的价值，同时不破坏现有方案最重要的 Coverage、Snapshot、Version、Incremental、可重放和可运营边界。

---

## 13. 参考资料

- 调研文章：https://todatabeyond.substack.com/p/how-to-turn-any-website-into-a-graph
- Crawl4AI Deep Crawling：https://docs.crawl4ai.com/core/deep-crawling/
- Crawl4AI Strategies：https://docs.crawl4ai.com/api/strategies/
- Crawl4AI LLM Strategies：https://docs.crawl4ai.com/extraction/llm-strategies/
- Crawl4AI LLM-Free Extraction：https://docs.crawl4ai.com/extraction/no-llm-strategies/
- R2R：https://github.com/SciPhi-AI/R2R
