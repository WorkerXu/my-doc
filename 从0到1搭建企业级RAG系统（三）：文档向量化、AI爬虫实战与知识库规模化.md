# 从0到1搭建企业级RAG系统（三）：文档向量化、AI爬虫实战与知识库规模化

原文地址：https://gitcode.csdn.net/69e19ccf54b52172bc6a79f2.html

## 1. 文章解决的问题

这篇文章把一个小型 RAG 项目从“有向量数据库和 Embedding 能力”推进到“可以抓取外部文章、切块、批量向量化、混合检索和 Reranker 精排”的完整闭环。文章的工程链路是：

```text
网页 / 本地文件
  -> Crawl4AI + Playwright 抓取
  -> Markdown / 文本
  -> RecursiveCharacterTextSplitter
  -> BGE-small Embedding
  -> Milvus
  -> Vector Search + BM25
  -> RRF
  -> BGE-Reranker
  -> Top-K
```

其价值主要在于展示了各组件如何串起来，以及真实开发中依赖、浏览器、中文分词、向量类型和模型加载等常见故障。它本身的规模只有 14 篇文章、718 个 Chunk，因此其中很多参数和实现方式适合作为原型基线，不适合直接复制到“1000 个站点、百万级文章、长期增量同步”的生产知识库。

---

## 2. Milvus Collection 与向量索引原理

文章使用以下字段：

```text
id      VARCHAR 主键
text    VARCHAR 原文块
vector  FLOAT_VECTOR，384 维
source  VARCHAR 来源路径
```

并采用：

```text
IVF_FLAT + COSINE
nlist = 128
```

### 2.1 IVF_FLAT 的原理

IVF_FLAT 先使用聚类把向量空间划分为 `nlist` 个倒排桶。查询时，不再扫描全部向量，而是先找到最近的若干聚类中心，再只扫描 `nprobe` 个桶中的原始向量。

因此它有两个核心调参旋钮：

- `nlist`：建索引时聚类桶数量；
- `nprobe`：查询时探测桶数量。

`nlist` 越大，桶越细，构建成本提高；`nprobe` 越大，召回通常越高，但延迟也越大。Milvus 官方文档把 `nlist=128` 作为默认值之一，但明确要求根据数据规模与资源调优，而不是固定为通用生产参数。

### 2.2 COSINE 与归一化

文章使用 BGE-small 的归一化向量并采用 COSINE。这个选择是合理的，但生产系统必须把以下内容放入同一个 Embedding Release：

```text
model_name
model_digest
dimension
normalization
metric
max_tokens
batch_policy
```

模型、维度、是否归一化、距离度量不能分别“隐式配置”。否则升级模型后很容易出现“旧向量 + 新查询向量”或 metric 不一致的问题。

### 2.3 不应照搬的地方：drop collection

文章示例在 `create_collection()` 中发现 Collection 已存在就直接 `drop_collection()`。对于演示项目很方便，但在生产知识库中风险很高：

1. 全量重建期间线上检索不可用；
2. 任意失败都会留下半成品或空库；
3. 无法做新旧版本对比；
4. 无法快速回滚；
5. Embedding / Chunk / Schema 升级与抓取事实被耦合。

生产实现应使用 Generation：

```text
BUILDING -> VALIDATING -> READY -> ACTIVE -> RETIRED
```

新索引后台构建并通过回归测试后，再通过 Alias 或逻辑指针原子切换。旧 Generation 保留观察窗口后再回收。

### 2.4 对目标方案的结论

Milvus 可以作为独立向量引擎，但不应把 `IVF_FLAT + nlist=128` 固化。应新增 `vector_index_release`，记录：

```text
engine
index_type
metric
build_params
search_params
schema_version
embedding_release_id
benchmark_result_id
```

索引类型通过固定评测集比较 FLAT、IVF_FLAT、HNSW 等方案后选择。FLAT 可作为小规模精确召回基准；大规模数据再在 recall、p95/p99 延迟、内存和构建时间之间取舍。

参考：

- https://milvus.io/docs/ivf-flat.md
- https://milvus.io/docs/hnsw.md
- https://milvus.io/docs/index_selection.md

---

## 3. 文档分块实现与稳定身份

文章使用 `RecursiveCharacterTextSplitter`：

```text
chunk_size = 512
chunk_overlap = 50
separators = ["\n\n", "\n", "。", "！", "？", "；", " ", ""]
```

其基本思想是按从强到弱的自然边界递归切分：先段落，再换行，再句号/标点，最后才退化到字符级边界。这比固定窗口直接切割更能保留语义连续性。

### 3.1 文章方案的优点

- 中文加入句号、感叹号、问号等分隔符；
- 有 overlap，降低边界信息丢失；
- PDF 与文本类文件采用不同 Loader；
- 避免为了 Markdown 引入体积和系统依赖都较重的 `unstructured`，降低安装复杂度。

### 3.2 对技术博客知识库仍不够

技术文章并不是普通纯文本。代码块、Markdown 表格、标题层级、引用和列表都有结构语义。如果先把 Markdown 当纯文本再递归切字符，会出现：

- fenced code 被切断；
- 表格表头和数据行分离；
- 标题与正文丢失父子关系；
- 同一 section 的 Chunk 无法稳定追踪；
- chunk_index 因前文插入内容而整体漂移。

因此生产方案应坚持：

```text
Canonical IR
  -> heading-aware / block-aware chunk
  -> 超长 section 才使用 recursive fallback
```

默认规则：

1. Heading / Section 先做第一层边界；
2. 段落、列表、引用尽量整块保留；
3. fenced code 默认不拆；
4. 表格默认不拆；
5. 超长 section 再按 tokenizer-aware 长度切分；
6. overlap 只用于普通文本；
7. Chunk 保存 `heading_path`、`block_start`、`block_end`。

### 3.3 `hash(source + chunk_index)` 是关键隐患

文章用：

```text
md5(source + chunk_index)
```

生成 Chunk ID。这个 ID 在原型中可用，但不稳定。文章头部新增一个段落后，后续全部 chunk_index 都可能移动，系统会把大量“未变化的内容”当成新 Chunk，造成无意义的 REEMBED 和索引抖动。

生产方案应使用 release-aware 的内容身份：

```text
chunk_id = hash(
  document_version_id
  + chunk_release_id
  + logical_block_range
  + normalized_text_hash
)
```

这样可以区分“正文版本真的变化”和“切块策略升级”。

---

## 4. 批量灌入脚本的扩展性问题

文章的 `ingest.py` 一次：

```text
遍历整个目录
 -> 全部 load/split 到内存
 -> texts = 所有 Chunk 文本
 -> 一次 encode
 -> 一次 insert
 -> flush
```

718 条向量时足够简单有效，但扩展到百万级 Chunk 会遇到：

- Python 内存峰值过高；
- 单次 Embedding batch 超出 GPU/CPU 能力；
- 中途失败后无法知道已经完成到哪里；
- 整批失败导致成功数据也要重做；
- 不能并行 worker；
- 无法限制 AI 处理积压对源站同步的影响。

生产实现应改为流式批处理：

```text
查询 PENDING Chunk
 -> lease 一批
 -> tokenize/embed
 -> 结果持久化
 -> 幂等 upsert 到目标 Generation
 -> 标记成功/失败
 -> checkpoint
 -> 下一批
```

每批必须具备：

```text
lease
idempotency_key
retry_count
checkpoint
partial success
OOM 降 batch
backpressure
```

Embedding backlog 必须和抓取链路解耦。即使模型服务宕机，也不能阻止新文章进入 Snapshot、Canonical IR 和 Markdown。

---

## 5. Crawl4AI + Playwright 的实际边界

文章选择 Crawl4AI + Playwright，是因为现代站点存在 JavaScript 渲染，普通 `requests` 不能保证拿到最终正文。Crawl4AI 的优势是已经封装浏览器抓取、内容抽取和 Markdown 生成能力，适合快速建立 AI-ready 抓取流程。

Crawl4AI 官方文档也把浏览器配置与单次抓取配置分开：`BrowserConfig` 控制浏览器，`CrawlerRunConfig` 控制缓存、提取、超时和 JS 等运行策略。

### 5.1 文章的重要实践结论

文章抓取知乎、百度等强反爬平台失败，最终选择放弃，并建议优先 RSS、API 或反爬较弱的平台。这一点对 1000 站点方案非常重要：

**Browser 是兼容动态网页的工具，不是反访问控制工具。**

正确成本顺序应该是：

```text
CMS/API
 -> Sitemap/Feed
 -> HTTP Static
 -> HTTP 特定 Header / 平台 API
 -> Crawl4AI Browser
 -> Playwright Recipe
 -> 明确 blocker / 人工处理
```

验证码、登录、付费墙、WAF 或明确访问限制不能通过无限升级浏览器重试来解决。

### 5.2 Crawl4AI Markdown 不应直接作为唯一真相

Crawl4AI 可以直接输出 Markdown，但对于长期知识库，直接“抓网页 -> Markdown -> 丢 HTML”会失去可重放能力。正文抽取规则升级时只能重新访问源站，既增加成本，也可能因为文章变化而无法重现旧结果。

正确链路：

```text
Fetch Attempt
 -> immutable Snapshot
 -> 多 Extraction Candidate
 -> Quality
 -> Canonical IR
 -> deterministic Markdown projection
```

Crawl4AI Markdown 应只是一个 Extraction Candidate，与 Trafilatura、Readability、站点 selector、JSON-LD 等候选一起进入质量选择。

参考：

- https://docs.crawl4ai.com/core/browser-crawler-config/
- https://docs.crawl4ai.com/core/markdown-generation/

---

## 6. Playwright / lxml 依赖故障的正确工程化方式

文章遇到：

- Python 3.13 与 lxml wheel 兼容问题；
- Playwright 浏览器内核下载超时；
- 通过固定/降级 Playwright 和补系统依赖解决。

这些是实际工程问题，但“遇到问题就降级到某个版本”不能成为长期设计。Playwright Python 包、Chromium revision、系统动态库和基础镜像是一组耦合运行时。

应建立 `runtime_artifact_release`：

```text
runtime_image_digest
python_version
crawl4ai_version
playwright_version
browser_revision
extractor_versions
system_dependency_manifest
model_digests
config_hash
```

构建阶段提前下载浏览器内核并制作不可变镜像，CI 做最小抓取 smoke test。生产 Run 只引用已验证的镜像 digest，而不是启动时临时下载 Chromium。

这样遇到新版本不兼容时，可以回滚 Runtime Release，而不是修改全局环境。

---

## 7. BM25、中文分词与为什么不应使用进程内 pickle 索引作为生产方案

文章用 `rank_bm25` 在内存中构建 BM25，再把索引 pickle 到本地文件，并用 Jieba 解决中文无法按空格切词的问题。

这个方案适合 718 个 Chunk 的演示，但不适合百万级、持续增量语料：

- 新文档加入后需要协调重建或更新内存索引；
- 多实例服务之间 pickle 状态难以一致；
- 容器扩缩容时本地文件不是可靠共享状态；
- 不能方便做 analyzer version、alias 和 generation；
- 过滤、聚合、权限和多语言策略都难扩展。

生产知识库应把 BM25 放入 OpenSearch 等专用全文引擎，并把 Analyzer 版本化：

```text
analyzer_release
  language_policy
  tokenizer
  normalizer
  stemmer
  stopword_set
  code_token_policy
  symbol_policy
```

中文可以采用 ICU/Jieba 类 tokenizer；英文使用语言匹配 analyzer；代码 token 要保护 `snake_case`、`camelCase`、`C++`、`.NET`、`Node.js`、版本号等技术词。

本地 `rank_bm25` 仍可保留，用于单元测试、离线小样本对照和算法实验，不作为生产检索真相源。

---

## 8. Hybrid Search + RRF 的技术原理

文章将：

```text
Vector Search
 + BM25
 -> RRF
```

作为召回层，这是适合技术知识库的方向。

向量召回擅长语义近似，但对精确 API、类名、版本号、缩写的可靠性不够；BM25 对关键词和技术 token 更敏感，却不擅长同义表达。RRF 不直接比较两种分数，而根据各自排名融合：

```text
RRF(d) = Σ 1 / (k + rank_i(d))
```

这避免了把 COSINE 相似度和 BM25 score 强行归一到同一尺度。

OpenSearch 官方也已经支持 Hybrid Search 和 RRF 类 rank-based 融合，并提供参数实验能力。因此生产方案应将：

```text
rrf_k
candidate_top_n
bm25_weight/vector_weight（若使用加权变体）
retrieval generation
```

放入 `retrieval_release`，不在代码里硬编码。

参考：

- https://docs.opensearch.org/latest/query-dsl/compound/hybrid/
- https://docs.opensearch.org/latest/search-plugins/search-relevance/optimize-hybrid-search/

---

## 9. Reranker 的价值与服务化方式

文章通过 BGE-Reranker 对融合后的候选做 Cross Encoder 打分，并展示英文查询的 Top-1 被纠正。这说明 Reranker 可以在粗召回后重新判断“Query 与 Chunk 是否真正相关”。

正确架构是：

```text
BM25 + Vector 粗召回
 -> RRF Top-N
 -> Cross Encoder Reranker
 -> Top-K
```

不能让 Cross Encoder 扫描全库，因为它需要对每个 `(query, document)` 对执行模型前向推理，成本远高于向量检索。

生产部署需要独立 `rerank-service`：

- 模型常驻；
- dynamic batch；
- 最大 batch token；
- GPU 并发限制；
- OOM 自动缩 batch；
- 可选 CPU fallback；
- 超时后直接降级到 RRF；
- 与 Source Sync 完全隔离。

文章只用少量查询展示效果，这不足以证明生产收益。应建立固定 judgment set，至少覆盖：

```text
中文概念
英文概念
中英混合
API/类名
版本号
代码 token
长查询
同义词
多文档答案
```

用 Recall@K、MRR、nDCG@K、HitRate@K、Reranker gain、p95/p99 延迟和 GPU seconds/query 做发布门禁。

### 9.1 RRF、向量相似度与 Reranker 分数不能横向比较

文章明确观察到 RRF 分数可能只有约 `0.03`，而 Reranker 输出可以是数值更大的另一套 score。这个现象不是“精排把分数提高了”，而是因为不同阶段的分数语义完全不同：

```text
BM25 score          = 关键词统计相关度
COSINE similarity   = 向量空间相似度
RRF score           = 基于 rank 的融合分数
Reranker score      = Cross Encoder 对 query-passage 的相关性打分
```

生产系统不能把它们塞进同一个 `score` 字段后直接比较大小，也不能用“Reranker score > RRF score”作为效果提升证据。正确做法是保存阶段化 Trace：

```text
query_id
chunk_id
stage
rank
score_type
raw_score
normalized_score_optional
release_id
candidate_source
```

效果判断使用同一阶段内的 rank、Recall@K、MRR、nDCG 等指标；跨阶段重点观察 rank movement、命中变化和最终回答质量。Web 调试台也应明确标注 score type，避免运营人员误读。

### 9.2 模型本地化应升级为不可变 Model Artifact

文章为了离线运行预先下载 `BAAI/bge-reranker-base`，这一实践值得保留，但生产系统不应只约定“某个目录里有模型”。模型文件、tokenizer、配置和推理依赖应纳入 `reranker_release + runtime_artifact_release`：

```text
model_name
model_revision
model_digest
tokenizer_digest
max_length
precision
runtime_image_digest
local_artifact_ref
smoke_test_result
```

模型下载发生在镜像/制品构建阶段，生产实例启动时只加载已校验的本地 artifact；外部模型仓库不可用时不应导致线上 Worker 在运行中随机下载、换 revision 或阻塞请求。

---

## 10. 文章方案与 1000 站点知识库的主要差距

### 10.1 缺少历史 Coverage 证明

文章是“手工选 URL 定点抓取”，没有解决一个站点历史文章如何枚举完整。1000 站点必须把 CMS API、Sitemap、Feed、Archive、Category、Docs TOC、Common Crawl URL Index 等发现 Provider 建模，并保存 cursor、exhaustion reason 和 known gap。

### 10.2 缺少增量同步

文章的数据灌入是离线一次性脚本。生产系统需要 Feed/API cursor、Sitemap lastmod、Conditional GET、Surface Scan、Reconcile，以及 ETag/Last-Modified/正文 hash。

### 10.3 缺少不可变 Snapshot 与版本链

文章把抓取结果直接变成 Markdown，再入向量库。生产方案需要：

```text
Snapshot -> Extraction -> Canonical IR -> Document Version -> Projection
```

以支持离线重放、规则升级和审计。

### 10.4 缺少任务可靠性

文章没有 queue、lease、checkpoint、retry、幂等键和 partial success。百万级数据必须把每一步做成可恢复任务。

### 10.5 缺少 Web 运营控制面

目标需求明确要求 Web 管理，因此必须能查看 Source、Coverage、Run、失败任务、抓取证据、版本差异、Markdown、Chunk、索引 Generation、检索解释和成本。

---

## 11. 可吸收到最终方案的优化

这篇文章并没有推翻现有总体架构，反而验证了以下已有方向是正确的：

- Crawl4AI + Playwright 适合作为 Browser 抓取层；
- Browser 不能作为强反爬的默认解决方案；
- Markdown/文本必须切块后再向量化；
- 中文 BM25 需要语言感知 tokenizer；
- Vector + BM25 + RRF 是技术语料的稳健默认组合；
- Reranker 应放在小候选集后做精排；
- 依赖和浏览器版本必须可复现。

需要新增或强化的设计：

1. 增加 `vector_index_release`，索引类型、metric、build/search 参数全部版本化；
2. 明确索引参数必须通过 Recall/Latency 基准测试选择，禁止固定照搬 `nlist=128`；
3. Chunk 超长 fallback 可采用 RecursiveCharacterTextSplitter 思路，但主策略仍为 structure-aware；
4. 明确禁止 `source + chunk_index` 作为稳定 Chunk ID；
5. Embedding 改为 leased streaming batch，不允许全库一次性 encode；
6. 生产 BM25 使用 OpenSearch/专用索引，`rank_bm25 + pickle` 只保留为实验工具；
7. Crawl4AI Markdown 降级为 Extraction Candidate，Snapshot/IR 保持真相层；
8. Playwright/Chromium/lxml 兼容性归入 Runtime Artifact Release 与 CI smoke test；
9. Reranker/Hybrid 参数升级必须经过固定 judgment set 的发布门禁；
10. Web 增加 Vector Index Generation、Reranker 前后排名和 Recall/Latency 对比视图；
11. 新增 Vector Payload Contract，在写入向量引擎前校验维度、数值类型、NaN/Inf、归一化约束、Embedding Release 与目标 Generation 一致性；
12. Retrieval Trace 按阶段保存 `score_type + raw_score + rank`，禁止直接横向比较 BM25/COSINE/RRF/Reranker 分值；
13. Embedding/Reranker 模型必须以不可变本地 Model Artifact 发布，生产运行时不得临时联网下载或静默换 revision。

### 11.1 Vector Payload Contract：把文章里的向量类型故障前移成数据契约

文章遇到过 Milvus 写入时向量数值类型不符合客户端预期的问题，并通过显式转换解决。这类错误在百万级批处理里如果等到数据库写入才暴露，会导致整批 retry、错误难定位，甚至让同一批部分成功部分失败。

因此在 Vector Index Adapter 前加入统一校验层：

```text
VectorPayloadValidator
  -> embedding_release_id matches generation
  -> dimension == expected_dimension
  -> numeric values only
  -> no NaN / Inf
  -> canonical dtype / serialization for target engine
  -> normalization invariant if release requires it
  -> chunk_id / embedding_id present
  -> per-item validation result
```

失败按单条记录 `VECTOR_PAYLOAD_INVALID`，保留错误原因和样本，不让一条坏向量拖垮整个 batch。需要强调的是，“canonical dtype”是 Adapter 的接口契约，不应在业务层硬编码某个向量数据库客户端的偶然实现细节。

---

## 12. 最终判断

文章是一篇很好的“小规模 RAG 工程串联案例”，其最值得借鉴的不是具体参数，而是它暴露出的组件边界和真实故障：抓取器与浏览器依赖必须可复现，中文全文检索需要专门分词，向量检索不能替代关键词检索，Reranker 需要放在粗召回之后。

但对于 1000 个技术博客、全历史抓取、长期增量同步的目标，不能采用文章里的目录全量扫描、一次性 Embedding、进程内 BM25 pickle、`source + chunk_index` 身份和 drop/recreate Collection。最终方案应继续采用“事实层与投影层分离、Release/Generation、流式批处理、可恢复任务、Coverage Evidence、HTTP First / Browser Last”的平台化架构，并吸收文章中的索引调优、Recursive fallback、Runtime 兼容性、Vector Payload Contract、阶段化 Retrieval Trace、不可变 Model Artifact 和 Reranker 量化评测经验。