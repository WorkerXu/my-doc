# 如何把任意网站转换成图谱知识库，并构建生产级 AI 助手

## 1. 调研对象

- 编号：25
- 原文：How to Turn Any Website into a Graph Knowledge Base With A Production-Ready Co-Pilot
- 调研地址：https://todatabeyond.substack.com/p/how-to-turn-any-website-into-a-graph
- 原始 Medium 版本：https://medium.com/gitconnected/how-to-turn-any-website-into-a-graph-knowledge-base-with-a-production-ready-co-pilot-28ce88e8988e
- 时间说明：Substack 页面标注 2025-10-07；原始 Medium/Level Up Coding 版本标注 2025-05-05，因此前者应视为后续转载/重发时间，而不是最初发布时间。
- 核心组件：Crawl4AI、`BFSDeepCrawlStrategy`、`FilterChain`、`URLPatternFilter`、`ContentTypeFilter`、`LXMLWebScrapingStrategy`、`LLMExtractionStrategy`、Pydantic/JSON Schema、R2R、Hybrid Search、Knowledge Graph、RAG。

文章展示了一条完整的 PoC 链路：

```text
Website
 -> Crawl4AI Deep Crawl
 -> URL Filter Chain
 -> HTML Scraping
 -> Schema-constrained LLM Extraction
 -> JSON Records
 -> R2R Document Ingestion
 -> Chunk / Embedding / Entity / Relation
 -> Hybrid RAG / Knowledge Graph
```

这条链路适合证明“网页可以被抓取、结构化并送入图谱/RAG 系统”，但不能直接等价于“1000 个技术博客的生产级全量历史知识库”。生产系统还必须解决历史覆盖证据、增量同步、稳定文档身份、幂等任务、跨站限流、Snapshot 重放、Markdown 质量、数据边界、图谱 provenance、检索作用域、Web 运维和成本治理。

本文最值得吸收的不是某个具体框架，而是四类可组合能力：

1. Discovery / Filter / Fetch / Extraction 策略解耦；
2. Schema 作为结构化抽取契约；
3. Graph 作为 Document Version 的可重建 Projection；
4. 把 GraphRAG 放在 BM25/Vector 之外作为可验证的额外召回通道，而不是默认宣称“建了图就一定提升检索”。

---

## 2. Crawl4AI 深度抓取实现与原理

文章使用 `AsyncWebCrawler` 配置浏览器环境，再在 `CrawlerRunConfig` 中挂载 `BFSDeepCrawlStrategy`。示例核心参数包括：

```text
max_depth = 2
include_external = false
max_pages = 2
filter_chain = ...
```

并使用 `LXMLWebScrapingStrategy` 处理页面内容。

### 2.1 BFS frontier

BFS 维护按深度分层的 frontier：

```text
Depth 0: seed
  -> extract links
Depth 1: links from seed
  -> extract links
Depth 2: links from depth 1
  -> ...
```

它的优势是覆盖行为直观，适合从首页、博客入口、目录页向较浅层文章扩散；缺点是导航、分类、作者、tag、分页、登录、搜索页会大量占据浅层预算。

对技术博客历史抓取而言，BFS 只能是一个 Discovery Provider，不能成为 Coverage 判定依据。原因包括：

- 旧文章可能只存在 sitemap 或 archive 中；
- RSS/Feed 常只有最近窗口；
- 文章可能从当前导航断链；
- category/tag/author 产生大量重复入口；
- JS Load More/virtual scroll 不一定暴露静态链接；
- 旧 URL 可能已迁移、跳转或只出现在历史索引；
- Docs 与 Blog 混合站点会产生大量非文章路径；
- BFS queue exhausted 只能证明“当前策略在当前预算和可见链接图上没有继续扩展”，不能证明“历史文章已全部发现”。

因此生产顺序应当是：

```text
CMS/API
 -> Sitemap
 -> Historical Feed
 -> Archive / Category / Author
 -> Docs TOC / Blog Index
 -> Common Crawl URL Index
 -> Best-First gap discovery
 -> bounded BFS/DFS
```

权威 Provider 负责覆盖证明，Deep Crawl 负责未知结构探索和补洞。

### 2.2 Durable frontier

PoC 可把 frontier 留在 crawler 内存；生产系统不能这样做。任何被发现 URL 应尽快写成 durable Observation：

```text
url_observation
- source_id
- observed_url
- normalized_url
- provider_type
- provider_run_id
- parent_url nullable
- anchor_text nullable
- depth nullable
- discovered_at
- filter_decision
- evidence_ref
```

后续抓取任务由平台 Scheduler 生成。Worker 崩溃时只丢失尚未 checkpoint 的短执行窗口，不需要从首页重新遍历。

---

## 3. FilterChain：降低放大成本，但必须可审计

文章组合 `URLPatternFilter` 与 `ContentTypeFilter`，其本质是在昂贵处理前减少候选集合。

对 1000 站点而言，一条错误 URL 如果进入完整链路，成本会被逐层放大：

```text
network
 -> browser seconds
 -> HTML parsing
 -> object storage
 -> extraction
 -> LLM token
 -> chunk
 -> embedding
 -> fulltext/vector index
 -> graph extraction
 -> graph storage
 -> downstream analysis
```

因此“早过滤”很重要，但“误过滤”比“多抓一些”更危险：它可能静默造成历史缺口。

生产实现应区分：

```text
HARD_REJECT
- 非允许域
- 明确后台/登录路径
- robots/policy 禁止
- 明确静态资产
- 明确危险协议/地址

SOFT_REJECT / DEPRIORITIZE
- 看起来像 tag/category/navigation
- 路径模板推断
- 相关性评分低
- 参数页/分页的概率判断
- article template 不确定
```

所有 decision 必须保留：

```text
url_filter_decision
- url_id
- filter_release_id
- rule_id
- action
- reason_code
- score nullable
- evidence_ref
- decided_at
```

SOFT_REJECT 不删除 Observation，只改变是否立即抓取/调度优先级。新 Filter Release 上线前，用已知文章 URL fixture 做 recall 回归，避免规则升级后大面积漏文章。

---

## 4. LXML、HTTP 与 Browser 的边界

文章虽然使用 Crawl4AI 运行抓取，但内容策略选择 `LXMLWebScrapingStrategy`。这个细节说明 Crawl4AI 并不意味着所有页面都必须依赖真实浏览器执行。

生产系统应拆成三层：

```text
Discovery Strategy
Fetch Route
Extraction Strategy
```

默认路线：

```text
HTTP_STATIC
 -> deterministic HTML parser
 -> quality check
```

只有出现以下条件才升级 Browser：

- 页面主体依赖 JS 渲染；
- 必须点击 Load More/分页按钮；
- 需要滚动后才能得到完整 DOM；
- 静态响应只返回壳；
- HTTP 抽取质量低于阈值且 Browser 有明显改善。

这使 Browser 成为受控 fallback，而不是默认成本。

---

## 5. Pydantic + LLMExtractionStrategy 的技术原理

文章定义 Pydantic 数据模型，并把生成的 JSON Schema 交给 Crawl4AI 的 `LLMExtractionStrategy`。示例还设置了 chunk token threshold、overlap、模型温度等参数。

核心链路：

```text
HTML
 -> normalize / chunk
 -> prompt + JSON Schema
 -> LLM structured generation
 -> JSON parse
 -> schema validation
 -> structured records
```

### 5.1 Schema 的价值

Schema 不只是“让 JSON 好看”，而是生产契约：

- required field 可验证；
- 类型可验证；
- 多条记录结构统一；
- Schema 可以独立版本化；
- 同一 Snapshot 可用新 Schema 离线重放；
- 可以统计缺失率、类型错误、漂移；
- deterministic extractor 与 LLM extractor 可以输出到同一契约后比较。

更合理的系统模型是：

```text
structured_schema_release
- schema_json
- required_fields
- field_descriptions
- validation_policy
- fixture_refs

structured_extraction_release
- engine: API / JSON_LD / CSS / XPATH / REGEX / LLM
- schema_release_id
- input_projection
- selector_or_instruction
- chunk_policy
- model_release_id nullable
- prompt_release_id nullable
- runtime_release_id
- quality_policy
```

### 5.2 LLM 不应进入 canonical 关键路径

技术博客正文清洗的目标是稳定保留标题层级、段落、列表、代码、表格、图片和链接，这完全可以优先用确定性抽取完成。

生产顺序：

```text
platform API / JSON-LD / CSS / XPath / Regex
 -> Trafilatura / Readability / Crawl4AI deterministic extraction
 -> Canonical IR / Markdown
 -> optional LLM Structured Extraction
```

LLM 更适合抽取复杂语义字段、关系、分类等长尾需求。模型故障、限流或预算耗尽时，canonical Markdown 仍必须正常 READY。

### 5.3 Token 成本必须成为事实数据

文章展示 token usage，但生产系统不能只输出日志，应落库：

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

这样可以按 Source/Schema/Model 比较“继续调用 LLM”与“编写稳定 CSS/XPath 规则”的长期成本。

---

## 6. 文章样例暴露的第一个生产问题：记录边界污染

文章把多个抓取到的商品对象写入一个 JSON 文件，然后把整个文件作为一个 R2R Document 摄取。样例 RAG chunk 中可以看到一个商品记录附近出现下一个商品的标题/描述内容。

这说明仅靠通用 token chunking，结构化数组中的相邻 item 可能进入同一个 chunk。对技术博客场景，类似问题会变成：

- 两篇独立文章被打进同一个逻辑文档；
- 多个 structured item 被同一个 embedding 表示；
- graph extraction 把相邻记录的实体/属性混在一起；
- provenance 无法准确指出事实属于哪个记录。

生产方案必须定义 **Hard Boundary**：

```text
DOCUMENT_VERSION boundary
STRUCTURED_RECORD boundary
CODE/TABLE optional atomic boundary
```

规则：

1. 一个 canonical 文章对应一个独立 `Document Version`；
2. Structured Extraction 返回多个 item 时，为每个 item 生成稳定 `structured_record_id`；
3. Chunk 不允许跨 `Document Version` 或 `Structured Record` 硬边界；
4. Graph Extraction 默认输入单位是一个 Document Version 或一个 Structured Record；
5. Embedding、Graph Evidence、Retrieval Trace 都保存 boundary ID；
6. 测试指标 `chunk_boundary_violation_rate` 必须为 0。

这比单纯调整 chunk overlap 更重要，因为 overlap 只能缓解语义切断，不能修复记录隔离问题。

---

## 7. R2R 摄取、Hybrid Search 与 Graph 的实际边界

文章把 JSON 文件上传/创建为 R2R 文档，R2R 再完成 chunk、embedding、实体/关系等派生处理。这个思路适合把 R2R 当成独立 RAG 平台。

但本项目已经需要维护：

- Source / Site Profile；
- Coverage Provider；
- URL Observation；
- Fetch Snapshot；
- Canonical IR；
- Document Identity / Version；
- 增量同步；
- Web Ops / Audit；
- Release / Generation；
- Scope / ACL。

若再让 R2R 同时成为 canonical 文档生命周期真相，会出现双写、删除语义、版本漂移和权限不一致。

正确边界：

```text
Platform Truth:
Snapshot / Document / Document Version / Canonical IR

Projection:
Markdown / Structured / Chunk / Embedding / Fulltext / Graph / R2R
```

### 7.1 R2R Adapter Contract

如使用 R2R，应定义显式映射：

```text
platform_document_version_id <-> r2r_document_id
structured_record_id nullable
source_id
scope_id / acl_tags
graph_generation_id
projection_release_id
content_hash
```

要求：

- 默认一条平台 Document Version 对应一个 R2R 文档，不把多篇文章拼成同一文档；
- R2R document ID 不能反向成为平台 Document Identity；
- 删除/重建 R2R projection 不影响平台 canonical；
- 只有能映射回平台 Document/Chunk Evidence 的图谱结果才允许进入 ACTIVE Graph；
- R2R 可替换为其他 Graph/RAG 后端，不改变 Source Sync 主链路。

---

## 8. 文章样例暴露的第二个问题：建了 Graph，不代表 Graph 参与了答案

文章的示例 RAG 响应中，最终回答来自 chunk search，而 `graph_search_results` 是空的。也就是说，该例确实已经生成实体和关系，但演示问答本身并没有展示 Graph channel 对答案产生增量贡献。

这是非常关键的生产判断：

```text
Graph Built != Graph Retrieval Worked
Graph Retrieval Worked != Graph Improved Answer
```

因此不能只用 `entity_count`、`relation_count` 作为 GraphRAG 成功指标。应把 Graph Retrieval 做成可观测、可门禁的召回通道。

### 8.1 Graph Retrieval Mode

建议：

```text
OFF
ON_DEMAND   # 默认
ALWAYS      # 仅实验/特定场景
```

`ON_DEMAND` 根据 query intent、实体识别、多跳需求、结构化关系词等决定是否启用图扩展。精确错误字符串、代码符号、版本号问题通常让 BM25/Vector 先处理；“哪些项目共同依赖 X”“A 与 B 通过什么技术关联”更适合 Graph。

### 8.2 Trace 必须记录“有没有贡献”

```text
graph_attempted
graph_seed_entities
graph_expanded_nodes
graph_expanded_edges
graph_search_result_count
graph_linked_chunk_count
graph_context_contributed
graph_latency_ms
graph_cost
```

当 `graph_search_result_count=0` 时，不应把该请求统计成“Graph 成功”。

### 8.3 增量价值评估

固定同一 judgment set，对比：

```text
Baseline: BM25 + Vector
Candidate: BM25 + Vector + Graph
```

指标：

```text
graph_activation_rate
graph_nonempty_result_rate
graph_context_contribution_rate
graph_incremental_recall_at_k
graph_incremental_ndcg
graph_multi_hop_answer_gain
graph_latency_delta
graph_cost_per_contributed_query
```

只有增量收益超过阈值，某个 Graph Retrieval Release 才能升级为 ACTIVE。

---

## 9. 文章样例暴露的第三个问题：标量值不应该伪装成全局实体

文章展示的知识图谱中，除了书名、作者、版本等合理实体，还出现类似 `Price`、`Inventory Count`、`UPC` 等泛化节点，并通过关系把商品连到这些节点。

对单一 Demo，这种图还能展示；进入大规模知识库后会出现明显语义问题：

- 所有商品都可能连到同一个“Price”实体；
- 数量、日期、版本、分数、配置值被错误提升为实体；
- 图产生超高连接“垃圾枢纽”；
- entity resolution 会误合并本应属于不同文档的值；
- 多跳检索产生无意义路径。

技术博客中同样存在大量标量：版本号、发布日期、CVSS 分数、吞吐量、延迟、端口、配置值、参数默认值等。

### 9.1 Entity / Relation / Statement 分层

生产图谱应至少区分：

```text
Entity
- 有稳定身份、可复用、值得跨文档归并的对象

Relation
- Entity -> Entity 的语义边

Statement / Fact
- Entity -> Entity 或 Entity -> Literal 的带类型事实
```

建议模型：

```text
graph_statement
- graph_generation_id
- statement_id
- subject_entity_id
- predicate
- object_entity_id nullable
- object_value_json nullable
- value_type nullable
- unit nullable
- valid_from nullable
- valid_to nullable
- confidence
```

约束：`object_entity_id` 与 `object_value_json` 二选一。

示例：

```text
PostgreSQL 16 --RELEASED_AT--> "2023-09-14"^^date
BenchmarkRun --THROUGHPUT--> 125000 requests/s
CVE-2025-xxxx --CVSS_SCORE--> 9.8
PackageX --LATEST_VERSION--> "3.2.1"
```

这样不会把 `9.8`、`2023-09-14`、`Price`、`Inventory Count` 变成全局实体节点。

---

## 10. Graph Ontology Release：约束“什么能成为节点”

仅有 LLM prompt 不足以稳定控制图结构。应把图谱本体/事实规则版本化：

```text
graph_ontology_release
- allowed_entity_types
- allowed_predicates
- literal_types
- predicate_domain_range
- cardinality_rules
- inverse/symmetric_rules
- entity_vs_attribute_policy
- temporal_policy
- identifier_policy
- fixture_refs
```

`entity_vs_attribute_policy` 用于明确：

- GitHub repo、Package、CVE、DOI、Person、Company 可成为 Entity；
- 日期、分数、数量、配置值默认是 Literal Statement；
- 版本是否建实体由查询价值决定；
- URL 可以作为稳定 identifier，不一定要单独建 URL Entity。

Graph Extraction 输出先通过 ontology validation，再进入 Entity Resolution/Generation。

---

## 11. Provenance：文章样例提醒必须设置“证据门禁”

文章展示的关系对象中可见有关系缺少直接 chunk 证据映射的情况。生产系统若把这种 LLM 关系直接放入 ACTIVE Graph，会逐渐形成无法审计的“生成式数据库”。

正确模型：

```text
graph_evidence
- graph_object_type: ENTITY / RELATION / STATEMENT
- graph_object_id
- document_version_id
- structured_record_id nullable
- chunk_id nullable
- block_range
- quote_hash
- extraction_release_id
- model_release_id nullable
- prompt_release_id nullable
```

规则：

1. ACTIVE Relation/Statement 必须至少有一条有效 evidence；
2. evidence 必须能回到当前可访问的 Document Version/Chunk；
3. 无 provenance 的候选进入 QUARANTINE，不进入在线 Graph Retrieval；
4. `graph_provenance_coverage` 对 ACTIVE 边/事实必须为 1.0；
5. 文档重版本、Chunk Release 变化时，evidence lineage 可重建，而不是直接失效成黑盒关系。

---

## 12. Entity Resolution 才是图谱长期质量核心

LLM 抽实体并不难，困难的是跨百万文档归一。

典型冲突：

- `Postgres` / `PostgreSQL`；
- `Node` / `Node.js`；
- 公司名与产品名相同；
- GitHub repo rename；
- 同名开源项目；
- 包名、组织名、品牌名共用 token。

建议流程：

```text
Mention Extraction
 -> type validation
 -> deterministic normalization
 -> stable identifier match
 -> alias lookup
 -> candidate generation
 -> name/embedding similarity
 -> optional LLM adjudication
 -> canonical entity
```

稳定 identifier 优先：

```text
GitHub repository URL
package ecosystem + package name
DOI
CVE ID
RFC ID
official domain
vendor/product identifier
```

Entity merge/split 必须版本化、可回放，并有 fixture 测试，不能直接在图数据库中人工改完就失去历史。

---

## 13. GraphRAG 与 BM25/Vector 的合理组合

三种通道解决的问题不同：

### BM25

强项：

- 类名、函数名、包名；
- 错误字符串；
- CVE/版本号；
- `snake_case`/`camelCase`；
- C++、.NET、Node.js 等精确 token。

### Vector

强项：

- 语义近似；
- 自然语言问题；
- 同义表达；
- 跨语言/长问题。

### Graph

强项：

- 多跳关系；
- 项目/人物/公司/技术依赖；
- 共同邻居；
- 实体聚合和结构化关系问题。

推荐：

```text
Query
 ├─ BM25 recall
 ├─ Vector recall
 └─ Graph gate
      -> entity seed
      -> bounded expansion
      -> graph-linked chunks

all channels
 -> hard SOURCE/DOCUMENT/VERSION/ACL scope
 -> controlled fusion / RRF
 -> optional reranker
 -> context assembly
```

Graph 有严格预算：

```text
max_seed_entities
max_hops
max_nodes
max_edges
max_linked_chunks
timeout_ms
```

Scope 必须在 Graph expansion 和 linked chunk 两个阶段都执行，不能先跨 tenant 扩图后再只过滤最终文本。

---

## 14. 对 1000 个技术博客的生产化端到端映射

```text
Source Onboarding
 -> Probe
 -> Provider Discovery
      CMS/API/Sitemap/Feed/Archive/TOC/CommonCrawl/DeepCrawl
 -> URL Observation
 -> Normalize
 -> Evidence-preserving Filter
 -> Resolve / Resource Probe
 -> HTTP-first Fetch / Browser fallback
 -> Immutable Snapshot
 -> Deterministic Extraction
 -> Canonical IR
 -> Document Identity / Version
 -> Canonical Markdown

Async Projections
 ├─ Structured Record
 ├─ Chunk -> BM25
 ├─ Chunk -> Embedding -> Vector
 ├─ Entity/Relation/Statement -> Graph
 └─ Summary/Topic/Other AI

Query
 -> BM25 + Vector + on-demand Graph
 -> Scope/ACL
 -> Fusion/Rerank
 -> Context
```

Source Sync 的成功边界到 `Document Version + Canonical Markdown READY` 即可。Embedding、Graph、LLM Structured Extraction、AI Analysis 可以独立 backlog。

这保证：图模型服务故障不会阻止新博客文章同步；R2R 故障不会破坏 canonical；Graph Release 升级不需要重新抓源站。

---

## 15. Web 管理应具备的相关能力

### 15.1 Deep Crawl / Filter Debug

展示：provider、parent URL、depth、anchor、命中规则、HARD/SOFT、decision、Release、预算、是否被其他 Provider 发现。

### 15.2 Structured Record Debug

展示：Snapshot、Schema、engine、record boundary、输出 JSON、validation error、token、cost、CSS/XPath/LLM diff、fixture pass rate。

### 15.3 Graph Explorer

展示：Entity、Relation、Statement、literal value、unit、valid time、Evidence、Graph Generation、Ontology Release、Entity Resolution 过程、merge/split、quarantine。

人工纠错必须生成 versioned correction，不直接无审计改 ACTIVE 图。

### 15.4 Retrieval Trace

展示：

```text
BM25_CANDIDATE
VECTOR_CANDIDATE
GRAPH_GATE
GRAPH_SEED
GRAPH_EXPAND
GRAPH_LINKED_CHUNK
SCOPE_FILTER
FUSION
RERANK
CONTEXT_ASSEMBLY
```

并明确：Graph 是否 attempted、是否有非空结果、是否真正贡献最终 Context。

---

## 16. 可靠性、预算和回放

### Deep Crawl

```text
max_discovered_urls
max_fetch_urls
max_depth
max_pages_per_surface
max_duplicate_ratio
max_wall_clock
max_browser_seconds
```

达到预算只能产生 `known_gap / budget_exceeded`，不能标记历史完整。

### LLM Structured Extraction

```text
max_documents_per_run
max_input_tokens_per_document
max_cost_per_source_per_day
max_schema_retry
```

超预算进入 projection backlog。

### Graph Extraction

```text
max_entities_per_document
max_relations_per_document
max_statements_per_document
max_resolution_candidates
min_relation_confidence
min_statement_confidence
```

### Replay

Extractor、Schema、Ontology、Graph、Embedding、Chunk、Retrieval Release 升级优先基于已有 Snapshot/Document Version 离线重放，不重新访问源站。

---

## 17. 测试与发布门禁

### URL Filter

```text
article_url_recall
non_article_reject_rate
soft_filter_false_negative_rate
```

### Record Boundary

- Chunk 不跨 Document Version；
- Chunk 不跨 Structured Record；
- Graph Evidence 的 record/document 映射唯一；
- `chunk_boundary_violation_rate = 0`。

### Structured Extraction

required/type/null/multi-item/DOM drift/invalid JSON/timeout/chunk 边界，并比较 deterministic 与 LLM engine。

### Graph Ontology

- 不允许未声明 entity type/predicate；
- Literal 不被错误升级成通用实体；
- predicate domain/range 正确；
- merge/split fixture 通过。

### Graph Provenance

- ACTIVE Entity/Relation/Statement 均有有效 evidence；
- `graph_provenance_coverage = 1.0`；
- 无证据候选只能 quarantine。

### Graph Retrieval

同一 judgment set 比较基线与 +Graph：

```text
Recall@K
nDCG
MRR
multi_hop_answer_hit_rate
graph_nonempty_result_rate
graph_context_contribution_rate
graph_incremental_recall_at_k
latency_delta
cost_delta
ScopeLeakRate
```

### 发布

```text
fixture
 -> offline replay
 -> benchmark
 -> shadow
 -> canary
 -> staged rollout
 -> ACTIVE
```

---

## 18. 结论

文章是一个清晰的“Deep Crawl + Schema Extraction + R2R Graph/RAG”端到端 Demo，对本项目有直接参考价值，但需要把 Demo 能力放进更严格的生产边界。

最终应吸收的能力是：

1. Crawl4AI BFS/DFS/Best-First 作为可替换 Deep Crawl Adapter，而不是 Coverage 真相；
2. URL Filter Chain 版本化并保留 HARD/SOFT decision evidence；
3. HTTP/LXML 优先，Browser 仅作为动态页面和质量 fallback；
4. Schema 与 Extraction Strategy 分离，LLM 不进入 canonical Markdown 必需链路；
5. Structured Record、Document Version、Chunk、Graph Extraction 均执行硬记录边界，禁止跨记录污染；
6. R2R 作为可重建 Projection/Adapter，不接管 Source、Coverage、Snapshot、Version、Task 真相；
7. 图模型使用 Entity + Relation + Statement，标量/时间/数量不伪装成全局实体；
8. 引入 Graph Ontology Release，约束实体类型、关系、literal、domain/range 和 entity-vs-attribute 策略；
9. ACTIVE Graph 强制 provenance，未绑定 Document/Chunk Evidence 的生成关系不得上线；
10. Graph Retrieval 默认按需启用，并用“非空结果、Context 贡献、增量 Recall/nDCG、多跳收益、延迟和成本”证明价值；
11. BM25 + Vector 仍是技术博客检索基线，Graph 是第三通道；
12. 所有 Graph/Schema/Chunk/Embedding/Retrieval 变化均通过 Release、Generation、Replay、Benchmark、Canary 管理。

通过这些约束，可以获得文章中知识图谱与 AI 助手的增强能力，同时保持 1000+ 技术博客场景最重要的完整历史覆盖、低成本增量同步、稳定 Markdown、可审计 provenance、可回放和长期可运营性。

---

## 19. 参考资料

- 调研入口：https://todatabeyond.substack.com/p/how-to-turn-any-website-into-a-graph
- 原始 Medium 版本：https://medium.com/gitconnected/how-to-turn-any-website-into-a-graph-knowledge-base-with-a-production-ready-co-pilot-28ce88e8988e
- Crawl4AI Deep Crawling：https://docs.crawl4ai.com/core/deep-crawling/
- Crawl4AI Strategies：https://docs.crawl4ai.com/api/strategies/
- Crawl4AI LLM Strategies：https://docs.crawl4ai.com/extraction/llm-strategies/
- Crawl4AI LLM-Free Extraction：https://docs.crawl4ai.com/extraction/no-llm-strategies/
- R2R：https://github.com/SciPhi-AI/R2R
- R2R Application：https://github.com/SciPhi-AI/R2R-Application
