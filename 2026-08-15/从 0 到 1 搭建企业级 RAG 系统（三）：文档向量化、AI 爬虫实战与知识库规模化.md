# 从 0 到 1 搭建企业级 RAG 系统（三）：文档向量化、AI 爬虫实战与知识库规模化

- 编号：26
- 原文：https://gitcode.csdn.net/69e19ccf54b52172bc6a79f2.html
- 调研目标：评估文章中的抓取、分块、向量化、混合检索与 Reranker 方案，判断其对“1000+ 技术博客全量历史抓取 + 增量同步 + Markdown 知识库 + Web 管理”平台的可复用价值，并给出生产化改造建议。

## 1. 文章实现链路概览

文章实现的是一条较完整的“小规模 RAG 数据面”链路：

```text
技术文章抓取
  -> Crawl4AI + Playwright
  -> Markdown / 文本文件
  -> RecursiveCharacterTextSplitter 分块
  -> BGE-small Embedding
  -> Milvus 向量入库
  -> BM25 + Vector 混合召回
  -> RRF 融合
  -> BGE Reranker 精排
  -> Top-K 上下文
```

文章最终规模约为 14 篇长文、718 个 Chunk。这个规模不足以直接证明方案可支撑百万文档，但它覆盖了知识库链路中几个非常关键的工程点：中文分词、混合检索、Reranker、浏览器抓取、分块参数化、离线模型和工程依赖治理。

## 2. Milvus Schema 与向量索引原理

文章的 Collection 主要字段为：

```text
id       VARCHAR 主键
text     VARCHAR 原始 chunk 文本
vector   FLOAT_VECTOR，384 维
source   VARCHAR 来源文件路径
```

向量索引使用：

```text
index_type = IVF_FLAT
metric_type = COSINE
nlist = 128
```

### 2.1 IVF_FLAT 原理

IVF（Inverted File）先对向量空间进行聚类，把全部向量分到多个 centroid 对应的倒排桶中。查询时只搜索距离查询向量最近的若干桶，而不是扫描整个集合。

`nlist` 决定聚类中心数量：

- 太小：每个桶太大，查询成本上升；
- 太大：训练、内存和索引维护成本上升，小数据下还可能产生大量稀疏桶；
- 查询侧通常还需要 `nprobe` 控制实际搜索多少桶，`nprobe` 越大召回越高、成本越高。

文章在 718 个 Chunk 上配置 `nlist=128` 可以用于演示，但不能直接迁移到百万级生产环境。生产中应该根据数据规模、向量分布、查询 SLA 和 Recall@K 做基准测试，而不是固定参数。

### 2.2 COSINE 的适用条件

文章使用已归一化的 BGE 向量，因此余弦相似度合理。对于归一化向量，余弦相似度和内积排序往往具有等价关系，但不同向量引擎对归一化、距离定义和 score 方向处理不同，平台应把 `metric` 固化在 Embedding / Index Release 中，避免模型升级后无意改变检索语义。

### 2.3 对目标平台的生产化改造

文章示例在创建 Collection 时如果已存在就直接删除重建，这在生产知识库中不可接受。百万级数据下应该使用 generation 模型：

```text
index_generation_001 BUILDING
  -> 批量写入
  -> 回归集验证
  -> READY
  -> 原子切换 alias
  -> ACTIVE
旧 generation 保留回滚窗口后 RETIRED
```

禁止为模型升级、Analyzer 升级或 Chunk 策略升级直接 `drop collection`。这些变化都应走 `RECHUNK / REEMBED / REINDEX` 的离线重建流程。

## 3. 分块实现与原理

文章使用 `RecursiveCharacterTextSplitter`，典型配置：

```text
chunk_size = 512
chunk_overlap = 50
separators = ["\n\n", "\n", "。", "！", "？", "；", " ", ""]
```

这种方法的优点是实现简单，对中英文混合文本有基本适配，并通过 overlap 缓解语义被边界截断的问题。

### 3.1 字符分块的局限

对技术博客而言，仅按字符递归切分会破坏很多结构信息：

- 标题层级丢失；
- 代码块可能被一分为二；
- Markdown 表格可能被拆坏；
- 列表项可能脱离父标题；
- 图片、图注和解释段落可能分离；
- Chunk 重建后 index 变化导致主键不稳定。

目标平台应坚持 `Canonical IR -> structure-aware chunk`，而不是把 Clean Markdown 当成无结构纯文本处理。

### 3.2 推荐 Chunk 模型

Chunk 应保留：

```text
chunk_id
source_id
document_id
document_version_id
chunk_release_id
heading_path
block_start
block_end
text_hash
content
language
code_language_hint
metadata
```

切分策略优先级：

1. 先按 Heading / Section 划分语义区域；
2. 段落、列表、引用保持 block 完整；
3. 代码 fenced block 默认不拆；
4. 表格默认不拆；
5. 仅超长 section 再按 token/字符二次切分；
6. overlap 只发生在普通文本边界，不复制完整大代码块或大表格；
7. `heading_path` 作为检索上下文的一部分保存。

### 3.3 Chunk ID 不能使用“文件路径 + chunk_index”

文章示例使用：

```text
md5(source + chunk_index)
```

这种 ID 在文档头部新增一段内容后会发生级联漂移：旧第 3 块可能变成第 4 块，导致大量 Chunk 被误判为删除和新增。

推荐：

```text
chunk_id = hash(
  document_version_id
  + chunk_release_id
  + logical_block_range
  + normalized_text_hash
)
```

Embedding 则进一步使用：

```text
embedding_id = hash(chunk_id + embedding_release_id)
```

这样可以明确区分“正文版本变化”“重新分块”“仅模型变化”。

## 4. 数据灌入脚本的优点与不足

文章将：

```text
遍历目录 -> 分块 -> 一次性 encode -> Milvus insert -> flush
```

串联在一个脚本中，适合本地实验，但不适合大规模平台。

### 4.1 主要问题

1. 一次性收集所有文本并统一向量化，会导致内存峰值不可控。
2. 缺少持久化 checkpoint，进程中断后只能重来。
3. 缺少幂等 upsert 语义。
4. Chunk、Embedding、Vector Index 状态没有独立生命周期。
5. 没有 backpressure，Embedding 服务变慢时无法保护抓取主链路。
6. 没有 generation / alias，升级索引可能影响在线查询。

### 4.2 推荐批处理协议

```text
Projection Task
  -> query pending chunks by generation
  -> reserve batch
  -> batch tokenize / embed
  -> persist embedding object/hash
  -> idempotent upsert to target generation
  -> commit projection state
  -> next batch
```

每批都有 `idempotency_key`、lease 和 retry。Embedding Worker 与抓取 Worker 解耦，Embedding 堵塞不能阻塞 Source Sync。

## 5. Crawl4AI + Playwright 的适用边界

文章指出现代动态网页仅靠 `requests` 不一定能拿到完整正文，因此选择 Crawl4AI + Playwright，并记录了 lxml、Playwright 浏览器内核下载、系统依赖等实际部署问题。

这部分对目标平台很有价值，但不能得出“所有页面默认 Browser”的结论。

### 5.1 正确的路由应是 HTTP First

推荐：

```text
Feed / CMS API / Sitemap
  -> Static HTTP
  -> HTTP with alternate headers
  -> Platform API
  -> Crawl4AI Browser
  -> Site-specific Playwright Recipe
```

Browser 只在以下情况使用：

- 正文必须依赖 JS 渲染；
- 页面需要滚动/点击才能完成内容加载；
- HTTP 抽取结果持续质量不足；
- PROBE 阶段用于确认页面结构；
- 特定动态站点配置明确要求。

### 5.2 抓取结果不能直接等同于知识库真相

Crawl4AI 可直接输出 Markdown，非常适合快速构建 RAG，但生产知识库仍应保存：

```text
Network Snapshot
  -> Extraction Candidates
  -> Canonical IR
  -> Clean Markdown Projection
```

原因是抽取器和 Markdown 规则会升级。只有保留 Snapshot 和 IR，才能在不重新访问源站的情况下重放、比较和重建 Markdown。

## 6. 对强反爬平台的处理原则

文章实践中遇到部分强反爬平台，即使使用 Playwright 也难稳定抓取，因此选择放弃，并建议优先使用 RSS/API/反爬较弱平台。

这一经验应上升为平台策略：

- 不绕过验证码、登录、付费墙、WAF 或明确访问控制；
- Browser 不是“绕过反爬”的手段；
- 优先使用 RSS、公开 API、Sitemap、公开 Archive；
- 对不可抓站点记录 Coverage Gap 和 blocker，而不是无限重试；
- Web 管理端必须区分 `ROBOTS_BLOCKED / POLICY_BLOCKED / WAF_BLOCKED / AUTH_REQUIRED` 等结果。

## 7. BM25 + Vector + RRF 混合检索

文章最值得吸收的部分之一，是把向量召回和 BM25 召回结合起来。

向量检索擅长语义近似，但技术知识库包含大量精确 token：

```text
HNSW
IVF_FLAT
BGE-small-en-v1.5
Python 3.13
Playwright 1.48.0
foo_bar
package.module
```

这些 token 对 BM25 往往更友好。

文章通过：

```text
Vector Search
 + BM25
 -> RRF(k=60)
```

融合两路排序。

### 7.1 RRF 原理

Reciprocal Rank Fusion 不依赖不同检索器的原始 score 是否同量纲，而使用排名：

```text
RRF(d) = Σ 1 / (k + rank_i(d))
```

这非常适合 BM25 score 与向量相似度 score 不可直接比较的场景。

### 7.2 生产化要求

融合参数不能写死在业务代码里，应建立 `retrieval_release`：

```text
id
bm25_generation
vector_generation
fusion_type
rrf_k
bm25_weight
vector_weight
candidate_top_n
reranker_release_id
rerank_top_n
final_top_k
context_policy_release_id
created_at
```

每次检索记录命中的 Retrieval Release，才能进行 A/B、回放和回归。

## 8. 中文 BM25 分词

文章指出空格分词对中文无效，并用 Jieba 做语言判断后的中文分词。这是正确方向，但生产平台不能把“检测到中文字符就 Jieba，否则 split”作为最终策略。

推荐 Analyzer Release：

```text
English -> standard/language analyzer + stemmer
Chinese -> ICU / Jieba 类 tokenizer
Japanese/Korean -> 语言专用 analyzer
Mixed Code -> 技术 token 保护规则
```

技术 token 保护尤其重要，应尽量保留：

```text
ClassName
snake_case
camelCase
package.module
v1.2.3
C++
.NET
Node.js
PostgreSQL-16
```

Analyzer 变化必须触发新全文索引 generation，而不能无声覆盖旧索引。

## 9. Reranker 精排

文章采用 `BAAI/bge-reranker-base`，先召回 Top-20 Chunk，再精排为 Top-3，并展示了 Reranker 对错误 Top-1 的纠偏效果。

### 9.1 原理

Embedding 检索通常是双塔：Query 和 Passage 分别编码，在线只做向量近邻，因此速度快但交互建模能力有限。

Cross Encoder Reranker 会同时编码 query + passage，能利用更细粒度的 token 交互，精度高但成本明显更高。因此正确架构是：

```text
大规模粗召回
 -> 少量候选融合
 -> Reranker 精排
 -> Context Assembly
```

而不是让 Reranker 扫描全部 Chunk。

### 9.2 生产部署建议

文章示例在需要时把模型加载到 GPU、用完释放。单用户实验可行，但多请求服务频繁 load/unload 会增加尾延迟和显存碎片风险。

生产建议独立 `rerank-worker` / `rerank-service`：

- 常驻模型；
- 动态 batch；
- 最大 batch token；
- GPU 并发限制；
- idle 时可按较长 TTL 卸载；
- OOM 自动降 batch；
- CPU fallback 可选；
- Reranker 故障时降级到 RRF 结果，不影响检索服务整体可用性；
- 更不能影响抓取、版本和 Markdown 生成链路。

## 10. 本地模型与可复现环境

文章记录了 Hugging Face 模型本地化下载、Playwright 版本兼容、浏览器内核和系统依赖问题。这说明生产系统不能只记录 Python requirements。

建议每个 Worker Release 记录：

```text
runtime_image_digest
python_version
crawl4ai_version
playwright_version
browser_revision
extractor_versions
embedding_model_digest
reranker_model_digest
analyzer_release_id
config_hash
```

模型和浏览器依赖通过内部镜像/Artifact Registry 分发，减少运行时在线下载的不确定性。

## 11. 本文方案与 1000+ 博客知识库目标的差距

文章适合验证“RAG 数据到检索”的基本闭环，但距离目标平台还缺少：

1. 1000+ Source 配置和站点生命周期；
2. Sitemap/Feed/CMS/Archive/Common Crawl 多 Provider 历史发现；
3. Coverage 证明；
4. URL 规范化、跳转解析和 Canonical Identity；
5. 增量同步和 Conditional GET；
6. Discovery Surface 与结构漂移；
7. Snapshot / Canonical IR / Version；
8. 任务 lease、checkpoint、retry、partial success；
9. 对象存储和不可变原始事实；
10. Web 管理、审核、审计、成本与告警；
11. 索引 generation 与无损切换；
12. 多语言 Analyzer Release；
13. Chunk / Embedding / Retrieval Release；
14. 删除、下线、Tombstone 与历史版本；
15. 生产限流、背压和 Browser 成本治理。

因此不应直接把文章代码扩容到 1000 个站点，而应把其中的检索和工程经验纳入平台现有“Truth Store + Projection + Generation”架构。

## 12. 对技术方案的具体优化结论

本次调研应对主方案加入或强化以下内容：

### 12.1 Chunk Identity 与 Projection 生命周期

明确 Chunk 是 `Document Version` 的可重建 Projection，Chunk ID 不能依赖可漂移的顺序编号；Chunk、Embedding 分别使用稳定、可解释的 release-aware id。

### 12.2 Hybrid Retrieval Release

将 BM25、Vector、RRF、Reranker、Context Assembly 统一纳入 Retrieval Release，记录 Top-N、RRF 参数、模型版本和回归结果。

### 12.3 RRF 作为默认融合基线

由于 BM25 和 Vector score 非同量纲，优先以 RRF 作为简单、稳定、可解释的默认基线；后续可再实验 learned fusion，但必须通过离线评测和 A/B。

### 12.4 Reranker 独立服务化

Reranker 是检索增强，不是知识库事实链路依赖。独立部署、可降级、支持 batch 和 GPU 资源治理。

### 12.5 Index Generation 禁止破坏式重建

任何 Chunk、Analyzer、Embedding 或 schema 升级都构建新 generation，验证后 alias 原子切换；生产禁止 drop-and-recreate。

### 12.6 分块必须结构感知

吸收文章“chunk_size/overlap 参数化”的思想，但升级为 heading-aware、block-aware、code/table-preserving 的结构化切分。

### 12.7 Batch Embedding + Backpressure

Embedding 使用批任务、checkpoint、幂等 upsert 和独立队列；不得一次性加载全量文档，更不得阻塞 Source Sync。

### 12.8 Runtime Artifact Release

把 Playwright/browser revision、模型 digest、Python/系统镜像版本纳入可追踪 Release，解决文章中出现的安装和运行时兼容问题。

## 13. 最终判断

这篇文章对目标平台最有价值的不是“Crawl4AI 可以输出 Markdown”，而是展示了一条现实可运行的检索增强链路：

```text
结构化分块
 -> Embedding
 -> Vector + BM25
 -> RRF
 -> Reranker
```

同时它暴露了从实验系统走向生产系统时必须补齐的关键问题：稳定身份、幂等增量、不可变事实、索引 generation、多语言 Analyzer、GPU 服务化和 release 治理。

因此主方案应保留现有 Coverage / Snapshot / IR / Incremental / Web 管理主干，并把本次调研沉淀为“检索投影与索引生命周期”的增强，而不是改造成以 Milvus 或某个单一爬虫为中心的架构。
