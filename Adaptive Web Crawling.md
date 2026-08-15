# Adaptive Web Crawling

来源：

- https://docs.crawl4ai.com/core/adaptive-crawling/
- https://docs.crawl4ai.com/advanced/adaptive-strategies/
- https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/adaptive_crawler.py

## 1. 结论

Crawl4AI 的 Adaptive Crawling 本质上不是“把网站抓全”的爬虫，而是一个 **query-driven information foraging（查询驱动的信息觅食）** 层：它持续估计当前知识集合对某个查询是否“已经足够”，对待抓链接计算预期信息增益，然后在置信度达到阈值、收益趋于饱和、达到页数上限或没有候选时提前停止。

这和博客知识库的核心目标“1000+ 站点全历史回填 + 持续增量同步”存在语义冲突。官方文档也明确指出 Adaptive Crawling 不适合 Full Site Archiving 和 Real-time Monitoring。因此它不能替代 Sitemap/RSS/CMS/Archive/Common Crawl/Wayback/Persistent Frontier，也不能作为 Coverage Truth 或 Backfill Complete 的判据。

但它非常适合作为一个 **Focused Gap / Expensive Slow-path Optimizer**：当全量 Inventory 已经建立后，对需要 Browser、动态渲染、深层导航、专题补洞或用户专题知识包的候选进行排序，用有限预算优先抓“最可能补信息缺口”的页面。这样可以在不丢 URL 的前提下降低 Browser 和 Deep Crawl 成本。

最终建议：在现有方案中新增 `adaptive_focused_gap` Provider/Job 类型，默认使用 statistical strategy，embedding strategy 作为可选增强。Adaptive 的分数只决定“先抓谁”，不能决定“永远不抓谁”。

---

## 2. 核心对象与状态模型

源码的核心状态是 `CrawlState`，主要包含：

```text
crawled_urls
knowledge_base
pending_links
query
metrics
term_frequencies
document_frequencies
documents_with_terms
total_documents
new_terms_history
crawl_order
kb_embeddings
query_embeddings
expanded_queries
semantic_gaps
embedding_model
```

它说明 AdaptiveCrawler 不是无状态的单页抽取器，而是维护一个不断增长的“当前知识集合 + 未抓候选 + 统计/向量状态”。

`CrawlState.save()` 会把：

- crawled URL；
- 知识库中的 Markdown；
- pending links；
- term frequency/document frequency；
- new term history；
- embedding 数组；
- expanded queries；
- semantic gaps；

序列化到 JSON；`load()` 可以恢复。

### 对本项目的意义

这个状态可以作为 **runtime resume hint**，但不能成为业务恢复真相：

1. 状态中包含运行时 Markdown、pending links 和 embedding，体积会随任务变大；
2. 它没有平台级 lease/fencing/outbox 语义；
3. crawler 版本、Source Profile、Traversal Policy、Embedding Model 变化后，旧状态可能不可兼容；
4. `pending_links` 和 `crawled_urls` 代表 AdaptiveCrawler 自己的局部视图，不等于平台完整 URL Inventory。

因此应继续沿用：

```text
PostgreSQL Persistent Frontier = 业务真相
S3 runtime checkpoint          = AdaptiveCrawler 恢复优化
```

Checkpoint 必须附：

```text
source_id
run_id
query_hash
strategy
crawler_release
source_profile_release
traversal_policy_release
embedding_release
schema_version
state_object_key
state_sha256
captured_at
```

任一 release 不兼容时，丢弃 runtime checkpoint，从 PostgreSQL Frontier 重建 focused slice。

---

## 3. Statistical Strategy 技术原理

StatisticalStrategy 不依赖 LLM 和 embedding，适合作为生产默认策略。

### 3.1 Coverage

源码先对 query 和文档做简单 tokenization，然后针对每个 query term 计算：

```text
doc_coverage = document_frequency / total_documents
freq_signal  = log(1 + term_frequency) / log(1 + max_term_frequency)
term_score   = doc_coverage * (1 + 0.5 * freq_signal)
coverage     = sqrt(avg(term_score))
```

最后 clamp 到 0~1。

这不是“网页覆盖率”，而是 **query term 在当前知识集合中的文本覆盖程度**。例如 query 是 `postgres redis scheduler`，大量页面覆盖这几个词会提高 coverage；但这不能证明某博客历史 URL 已全部发现。

### 3.2 Consistency

源码对 knowledge base 两两取词集合并计算 Jaccard：

```text
J(A,B) = |A ∩ B| / |A ∪ B|
consistency = pairwise Jaccard 的平均值
```

这实际上更接近“内容主题重叠/相似度”，并不是真正的事实一致性检测。两个页面使用相同术语但结论相反，仍可能得到较高分。

因此在本项目里只能把它解释为：

```text
topical_coherence
```

不能解释成“事实可靠性”。

### 3.3 Saturation

源码记录每个新页面引入的 unique term 数量：

```text
new_terms_history = [page1_new_terms, page2_new_terms, ...]
```

然后用：

```text
saturation = 1 - recent_rate / initial_rate
```

估计边际收益是否下降。

它能很好地表达“继续抓类似页面新增信息越来越少”，但也有明显局限：

- 以第一个页面为基准，首个页面异常会影响后续尺度；
- 新术语数量不等于有价值信息；
- 模板、代码、导航噪声会制造新词；
- 多语言站点、中文分词、代码符号都会影响 token 统计。

所以进入平台前应基于 Canonical/clean text 或轻量 cleaned text 计算，而不是直接信任未清洗 Markdown。

### 3.4 总置信度

当前源码：

```text
confidence = 0.4 * coverage
           + 0.3 * consistency
           + 0.3 * saturation
```

一个值得注意的实现细节是：`AdaptiveConfig` 虽然定义并校验 `coverage_weight / consistency_weight / saturation_weight`，但当前 `StatisticalStrategy.calculate_confidence()` 直接硬编码了 0.4/0.3/0.3，没有读取 config。

因此本项目不能假设调整 config 权重一定生效。Crawler release 升级时必须用 Golden Behavior Test 验证“配置字段 -> 实际运行行为”。

---

## 4. 链接排序原理

StatisticalStrategy 的意图是：

```text
score = relevance_weight * relevance
      + novelty_weight   * novelty
      + authority_weight * authority
```

默认权重为：

```text
relevance 0.5
novelty   0.3
authority 0.2
```

### 4.1 Relevance

`_calculate_relevance()` 优先使用 `link.contextual_score`。该值可以由 Crawl4AI 的 Link Preview / score_links 路径产生；如果没有 contextual score，则退化为 query terms 与 link preview terms 的覆盖比例。

所以“BM25 relevance”并不是所有情况下都直接在这个函数里计算；没有 contextual score 时只是简单 term overlap。

### 4.2 Novelty

源码从 link text/title/head metadata 提取词集合：

```text
novelty = |link_terms - existing_terms| / |link_terms|
```

它估计的是“这个链接的预览文本中有多少词当前知识库还没见过”。

这适合做候选优先级，但不能用于永久过滤：一个标题很普通的历史文章，其正文可能完全不同；反之一个标题包含很多新词，也可能是标签页或导航页。

### 4.3 Authority 的实现偏差

源码存在 `_calculate_authority()`，会根据 `/docs/`、`/api/`、`/guide/`、PDF 等 URL 特征加权，但当前 `rank_links()` 中实际上写的是：

```text
authority = 1.0
# authority = self._calculate_authority(link)
```

也就是说当前 release 下 authority 对所有候选基本是常数，只提供统一偏置，不能按 URL 结构真正区分权威度。

这再次说明：平台必须把第三方 runtime score 当 advisory，不应把文档描述直接等同于当前版本实际行为。

---

## 5. 停止条件

StatisticalStrategy 的 `should_stop()` 会在以下任一条件满足时停止：

```text
confidence >= confidence_threshold
pages >= max_pages
pending_links 为空
saturation >= saturation_threshold
```

主 `digest()` 循环还会在：

```text
ranked_links 为空
最高 link score < min_gain_threshold
达到 max_depth
```

时停止。

这对专题研究是合理的，但对“全历史归档”危险，因为：

- `max_pages` 是预算，不是完整性边界；
- `saturation` 只代表术语边际收益下降；
- `min_gain_threshold` 会让低分历史 URL 永远不进入这次 Adaptive run；
- `max_depth` 无法覆盖所有归档结构；
- `pending_links` 仅是 AdaptiveCrawler 当前观察到的链接集合。

因此平台必须把 Adaptive stopped reason 保存为：

```text
FOCUSED_SUFFICIENT
FOCUSED_SATURATED
FOCUSED_MAX_PAGES
FOCUSED_MAX_DEPTH
FOCUSED_LOW_GAIN
FOCUSED_NO_LINKS
FOCUSED_IRRELEVANT
```

这些原因都不能映射成 `BACKFILL_COMPLETE`。

---

## 6. Embedding Strategy 技术原理

Embedding strategy 用语义空间而不是纯关键词判断“知识是否覆盖 query”。

### 6.1 Query Expansion

`map_query_semantic_space()` 会先让 chat completion 模型生成 query variations，再把 query 分为训练集和 held-out validation 集。

这意味着 embedding strategy 实际包含两类模型调用：

```text
query expansion: chat completion
semantic vector: embedding model
```

如果未明确配置 query model，当前实现存在向默认 provider 回退的行为。因此生产环境不能把 embedding strategy 直接暴露为“无依赖”能力，必须显式配置模型、凭据、预算和 release。

### 6.2 Query Space Coverage

当前实现对 query embeddings 与 knowledge-base embeddings 做 L2 normalization，然后计算 cosine similarity：

```text
best_similarity(q) = max(q · d_i)
coverage_score      = mean(best_similarity)
```

如果配置 `coverage_tau`，则可以变成“best similarity 超过阈值的 query variations 占比”。

这比关键词策略更能覆盖同义词、概念变体和不同表述，但仍然是 **query semantic coverage**，不是 URL inventory coverage。

### 6.3 Gap Detection

对每个 query embedding 找到距离最近的知识文档：

```text
gap_distance(q) = min cosine_distance(q, d_i)
```

距离大的 query variation 被视为当前知识缺口。

候选链接也做 embedding，然后估计它能把这些 gap 拉近多少：

```text
improvement = old_gap_distance - candidate_distance
```

同时如果候选链接 embedding 与现有 KB 高度相似，会加 overlap penalty。

这就是 Adaptive Crawling 最值得借鉴的部分：**把昂贵抓取预算分配给预计最能填补当前知识缺口的候选。**

### 6.4 Convergence + Validation

Embedding strategy 记录 confidence history，计算平均 improvement。当平均提升低于相对阈值时，不会立刻停止，而是用 held-out validation queries 验证。

如果 validation score 达到阈值，则标记 `converged_validated` 并停止；如果 validation 很低，则继续抓取。

这比单纯“分数达到阈值就停”更稳健，可以减少训练 query 过拟合。

### 6.5 Irrelevant Query Fast Stop

当 confidence 低于 `embedding_min_confidence_threshold` 且已经抓过页面时，会停止并标记：

```text
below_minimum_relevance_threshold
is_irrelevant = true
```

这个机制非常适合 Web Admin 的专题抓取：用户输入一个与站点完全无关的专题时，可以快速停止，避免浪费 Browser/Embedding 成本。

---

## 7. Link Preview 的网络语义

`AdaptiveCrawler._crawl_with_preview()` 会构造 `LinkPreviewConfig`，当前源码包含类似：

```text
include_internal = true
include_external = false
query = 当前 query
concurrency = 5
timeout = link_preview_timeout
max_links = 50
score_links = true
```

并且抓取结果中还会过滤掉没有 `head_data` 的 internal links。

这带来两个生产级问题：

### 7.1 Preview 本身会扩大请求量

一次正文抓取可能伴随多个 link-preview metadata 请求。因此不能只对主 URL 做限流，必须把 Preview 请求也纳入：

```text
approved scope
robots policy
SSRF validation
per-domain token bucket
audit
request budget
```

### 7.2 `head_data` 过滤会造成候选偏差

没有成功得到 head metadata 的链接可能被 Adaptive runtime 丢掉。这在 Focused Job 可以接受，但绝不能让它影响平台全量 URL Inventory。

正确顺序仍应是：

```text
Raw Link Observation
 -> durable URL Inventory
 -> Adaptive preview/score
 -> priority/admission for focused slice
```

而不是：

```text
Adaptive runtime filter
 -> 只保存幸存 URL
```

---

## 8. 与 1000+ 博客知识库的正确集成方式

新增一个独立能力：

```text
adaptive_focused_gap
```

它不属于权威历史 Provider，而属于 Gap/Optimization Plane。

### 8.1 输入

```text
source_id
approved seed URL
query / topic_profile
candidate frontier subset
budget_pages
max_depth
strategy
runtime_release
embedding_release(optional)
```

候选 URL 必须已经经过：

```text
URL Observation
 -> Platform Normalization
 -> Approved Scope
 -> Robots Policy
 -> URL Validity/Admission policy
```

### 8.2 输出

平台保存：

```text
adaptive_run
adaptive_metric_snapshot
adaptive_candidate_score
adaptive_runtime_checkpoint
```

建议字段：

```text
adaptive_run
- id
- source_id
- query_hash
- strategy
- state
- budget_pages
- crawler_release
- traversal_policy_release
- embedding_release
- started_at
- finished_at
- stop_reason

adaptive_candidate_score
- adaptive_run_id
- url_id
- relevance
- novelty
- expected_gain
- rank
- scored_at

adaptive_metric_snapshot
- adaptive_run_id
- pages_crawled
- focused_confidence
- topical_coverage
- topical_consistency
- saturation
- validation_confidence
- avg_improvement
- captured_at
```

字段命名必须故意使用 `focused_*` 或 `topical_*`，避免和平台 `coverage_snapshot` 混淆。

### 8.3 调度语义

推荐：

```text
Inventory/Frontier Truth
 -> 找出 Browser/Deep/Repair 等昂贵候选
 -> Adaptive score
 -> 高 expected_gain 先调度
 -> 低分候选保留为 READY/DEFERRED，不删除
```

也就是：

```text
Adaptive score = when
not whether
```

### 8.4 适用场景

1. **Browser 慢路径预算**：同一站有大量 JS 页面时，优先抓最可能补知识缺口的候选；
2. **Deep Crawl Gap Scan**：权威 Provider 已跑完，但仍有大量深层候选时，优先探索高信息增益区域；
3. **专题知识包**：用户需要“某站关于 PostgreSQL 性能优化的全部高相关内容”时，生成 focused collection；
4. **RAG 冷启动**：在全历史 backfill 尚未完成时，先得到能回答某专题的高价值子集；
5. **修复队列优先级**：抽取失败/Browser fallback 候选中，先修复对热点查询最有价值的文档。

### 8.5 不适用场景

1. 全历史 Backfill 完成判定；
2. RSS/CMS/Sitemap 增量同步；
3. Tombstone/删除检测；
4. URL Identity；
5. robots/access decision；
6. Soft-404 Truth；
7. Canonical Markdown 生成；
8. Source Approved Scope 扩张。

---

## 9. 对现有技术方案的具体优化

### 9.1 Provider 增加

在 Provider 类型中增加：

```text
adaptive_focused_gap
```

其优先级低于权威 Provider，只从已经观察到的 URL/受控 Deep Crawl slice 中选候选。

### 9.2 Traversal Policy 增加 focused 模式

```yaml
adaptive_focused:
  enabled: false
  strategy: statistical
  max_pages_per_run: 30
  max_depth: 3
  top_k_links: 3
  min_gain_threshold: 0.1
  confidence_threshold: 0.7
  allow_embedding: false
  persist_runtime_state: true
```

生产默认 `allow_embedding=false`，需要专题语义能力时单独发布 release。

### 9.3 Web Admin 增加 Focused Crawl 页面

展示：

```text
query
strategy
focused confidence
coverage/consistency/saturation
validation score
expected gain 排名
pages crawled
budget remaining
stop reason
runtime checkpoint
```

UI 必须显示警告：

```text
Focused Sufficiency ≠ Historical Coverage
```

### 9.4 Metrics 增加

```text
adaptive_run_total{strategy,status}
adaptive_pages_crawled_total{source,strategy}
adaptive_stop_total{reason}
adaptive_focused_confidence{source}
adaptive_expected_gain{source}
adaptive_validation_confidence{source}
adaptive_preview_requests_total{source,status}
adaptive_embedding_calls_total{provider,model}
adaptive_cost_total{source,strategy}
```

### 9.5 Release 增加

```text
adaptive_policy_release
adaptive_runtime_release
query_expansion_release
embedding_release
```

特别是 Crawl4AI AdaptiveCrawler 当前源码存在“配置字段与实际行为可能不一致”的情况，升级必须做行为级 Golden Replay。

---

## 10. Golden Test / 验收测试

新增测试至少包括：

### 10.1 Truth Boundary

```text
Adaptive 达到 confidence threshold
=> 不得把 Source Backfill 标记 COMPLETE
```

```text
Adaptive 因 saturation 停止
=> 低分 URL 仍在 Persistent Frontier
```

```text
Adaptive 因 max_pages 停止
=> stop_reason=FOCUSED_MAX_PAGES，不丢 Observation
```

### 10.2 Runtime Behavior

验证当前 crawler release 下：

- Statistical confidence 是否仍硬编码 0.4/0.3/0.3；
- authority 是否仍为固定 1.0；
- Link Preview 是否产生额外 metadata 请求；
- 无 `head_data` 的链接是否被 runtime 过滤；
- `min_gain_threshold` 实际何时生效；
- `top_k_links` 是否真正限制每轮 fan-out；
- resume state 在 release 变化时是否拒绝复用。

### 10.3 Embedding

- query expansion 失败时不影响平台业务 Frontier；
- embedding provider 429/timeout 有 bounded retry；
- 模型切换后旧 checkpoint 不直接 resume；
- irrelevant query 能快速停止；
- convergence 但 validation 低时继续运行；
- candidate overlap 只降低 priority，不删除 URL。

### 10.4 Safety

- Link Preview 不能访问未 approved host；
- redirect 每跳做 SSRF/scope 校验；
- Preview 请求进入全局 per-domain token bucket；
- external links 即使被发现也只保存 Observation/Candidate，不自动主动抓取。

---

## 11. 最终建议

Adaptive Crawling 值得加入，但必须改变它在系统中的“身份”：

```text
不是：全历史爬虫 / Coverage Truth
而是：Focused Information-Gain Scheduler
```

平台仍以：

```text
CMS/API + Sitemap + RSS + Archive
+ Common Crawl + Wayback
+ Safe Recon + Deep Crawl Gap
+ Persistent URL Inventory
```

建立历史完整性。

Adaptive Crawling 只在已经有业务真相边界之后，对高成本候选做：

```text
query-aware relevance
+ novelty
+ semantic gap
+ saturation
+ validation
```

排序和预算控制。

这样可以同时得到两种能力：

1. **Archive Correctness**：不因“信息足够”而漏历史文章；
2. **Focused Efficiency**：在 Browser/Deep Crawl/RAG 专题任务中，用更少页面更快获得高价值知识。

这也是本次调研对最终技术方案最有价值的增量。
