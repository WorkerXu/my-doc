# Adaptive Web Crawling

来源：

- https://docs.crawl4ai.com/core/adaptive-crawling/
- https://docs.crawl4ai.com/advanced/adaptive-strategies/
- https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/adaptive_crawler.py

## 1. 结论

Crawl4AI 的 Adaptive Crawling 本质上是一个 **query-driven information foraging（查询驱动的信息觅食）** 运行时：它维护当前知识集合，对候选链接估计相关性/新颖性/语义缺口收益，并在“专题信息已经足够”、边际收益下降、达到页数或深度预算、候选耗尽等条件下提前停止。

它与本项目“1000+ 技术博客全历史回填 + 持续增量同步”的核心语义不同。官方文档也明确把 **Full Site Archiving** 和 **Real-time Monitoring** 列为不推荐场景。因此：

```text
Adaptive confidence != Historical Coverage
Adaptive stopped     != Backfill Complete
Adaptive pending     != Platform URL Inventory
```

最合适的身份不是 Coverage Provider，而是：

```text
Focused Information-Gain Scheduler
+ Expensive Slow-path Optimizer
```

即：先由 CMS/API、Sitemap、RSS、Archive、Common Crawl、Wayback、Safe Recon、Deep Crawl 等建立持久 URL Inventory，再让 Adaptive 能力对 Browser、Deep Crawl、Repair、专题知识包等昂贵候选决定“先抓谁”。低分、未选择、preview 失败的 URL 仍留在 Persistent Frontier，不能被永久删除。

进一步阅读当前源码后，生产方案还应增加一个更强约束：**不要直接把 `AdaptiveCrawler.digest()` 当平台调度器。** 当前实现存在批次结果关联、严格预算、恢复状态完整性、并发复用等问题。生产默认应由平台拥有 Frontier、Admission、请求关联、重试和预算；Crawl4AI Adaptive 只作为评分/实验运行时，或使用经过补丁和 Golden Test 的固定 release。

---

## 2. 核心对象与状态模型

`CrawlState` 主要维护：

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

这说明 AdaptiveCrawler 是有状态运行时，而不是单页抽取器。

`CrawlState.save()` 会把 crawled URLs、运行时 Markdown、pending links、词频、embedding、expanded queries 等序列化为 JSON；`load()` 可以恢复这些字段。

但它不能作为平台业务恢复真相，原因包括：

1. 状态包含 Markdown、pending links、embedding，体积随任务增长；
2. 没有平台级 Lease、Fencing Token、Transactional Outbox；
3. `pending_links/crawled_urls` 只是 AdaptiveCrawler 局部视图；
4. runtime/source profile/traversal/model 变化后状态可能不兼容；
5. embedding 模式的关键运行状态并未完整持久化，见第 7 节。

因此保持：

```text
PostgreSQL Persistent Frontier = 业务真相
S3 Runtime Checkpoint          = 可丢弃的恢复优化
```

Checkpoint Envelope 至少包含：

```text
source_id
run_id
query_hash
strategy
crawler_release
source_profile_release
traversal_policy_release
adaptive_policy_release
query_expansion_release
embedding_release
schema_version
state_object_key
state_sha256
captured_at
```

任何关键 release/query hash 不匹配时，不能原样 resume，应从 PostgreSQL Frontier 重建 focused slice。

---

## 3. Statistical Strategy 技术原理

### 3.1 Coverage

当前源码先对 query 与 Markdown 做简单 tokenization，然后对每个 query term 计算：

```text
doc_coverage = document_frequency / total_documents
freq_signal  = log(1 + term_frequency) / log(1 + max_term_frequency)
term_score   = doc_coverage * (1 + 0.5 * freq_signal)
coverage     = sqrt(avg(term_score))
```

最终 clamp 到 0~1。

它表达的是 **query terms 在当前知识集合中的文本覆盖程度**，不是站点 URL 覆盖率。

### 3.2 Consistency

当前源码不是事实矛盾检测，而是对知识库文档两两计算词集合 Jaccard：

```text
J(A,B) = |A ∩ B| / |A ∪ B|
consistency = pairwise Jaccard 平均值
```

因此它应被解释为：

```text
topical_overlap / topical_coherence
```

不能解释为事实一致性或来源可信度。

高级文档对 consistency 的描述比当前实现更强，涉及“关键陈述/矛盾”的概念，但当前源码实际是 term overlap。生产设计必须以 **pinned source behavior + Golden Test** 为准，而不是只按文档语义命名。

### 3.3 Saturation

源码记录每个新页面带来的新 unique terms 数量：

```text
new_terms_history = [page1_new_terms, page2_new_terms, ...]
saturation = 1 - recent_rate / initial_rate
```

它表达边际新增词汇是否下降。

局限：

- 首个页面异常会改变尺度；
- 模板/导航/代码噪声会制造“新词”；
- 新词数量不等于有价值信息；
- 多语言和技术术语对 tokenizer 非常敏感。

### 3.4 总置信度存在配置/行为偏差

`AdaptiveConfig` 定义并校验：

```text
coverage_weight
consistency_weight
saturation_weight
```

但当前 `StatisticalStrategy.calculate_confidence()` 实际直接写死：

```text
confidence = 0.4 * coverage
           + 0.3 * consistency
           + 0.3 * saturation
```

所以调配置不代表行为会变化。每次 runtime 升级都必须验证配置字段是否真的进入执行路径。

### 3.5 默认 tokenizer 对技术博客并不理想

当前 tokenizer 大致是：

```text
去标点 -> split whitespace -> 丢弃长度 <= 2 的 token
```

对技术知识库会产生明显偏差，例如：

```text
Go
R
C
C++
C#
JS
AI
DB
S3
```

可能被丢弃或被标点清洗破坏；中文没有空格时也不会得到理想词粒度。

因此生产默认不应把内置 statistical score 直接作为最终专题排序。建议实现 `platform_statistical_v1`：

- Unicode/CJK-aware analyzer；
- 保留 C++、C#、.NET、Go、R、JS、AI、DB、S3、K8s 等技术 lexeme；
- 对 title/heading/anchor/body 分字段；
- 使用平台 clean text 或 Canonical IR；
- 权重配置真正参与计算；
- score 只影响 priority。

内置 StatisticalStrategy 保留为 baseline/对照组。

---

## 4. 链接排序技术原理与文档漂移

当前源码的 StatisticalStrategy 实际是加权和：

```text
score = relevance_weight * relevance
      + novelty_weight   * novelty
      + authority_weight * authority
```

默认：

```text
relevance 0.5
novelty   0.3
authority 0.2
```

而高级文档使用“Expected Gain”概念描述时更接近乘积式相关性 × 新颖性 × 权威度。两者不是同一公式，因此平台不能依赖文档公式复现当前 runtime 排名。

### 4.1 Relevance

`_calculate_relevance()` 优先使用 `link.contextual_score`；有 Link Preview/score_links 时它可能来自 runtime 的上下文评分。如果没有，则退化为 query terms 与链接预览文本的简单覆盖比例。

因此函数注释里的“BM25 relevance”也不能理解成该函数始终执行完整 BM25。

### 4.2 Novelty

当前实现大致为：

```text
novelty = |link_terms - existing_terms| / |link_terms|
```

它只用 link text/title/head metadata 预测新颖性。标题普通不代表正文不新颖，因此只能用来排序，不能永久过滤。

### 4.3 Authority 当前实际上是常数

源码中有 `_calculate_authority()`，但当前 `rank_links()` 实际执行：

```text
authority = 1.0
# authority = self._calculate_authority(link)
```

所以 authority 并没有按 URL 结构区分候选，只给所有候选相同偏置。

结论：**第三方 runtime score 必须被视为 advisory。** 平台存 component score、runtime release 和行为测试结果，不能把“文档写了某算法”直接升级为业务语义。

---

## 5. 停止条件与严格预算问题

StatisticalStrategy `should_stop()` 会在以下条件之一满足时停止：

```text
confidence >= confidence_threshold
len(crawled_urls) >= max_pages
pending_links 为空
saturation >= saturation_threshold
```

主 `digest()` 还会因：

```text
ranked_links 为空
ranked_links[0].score < min_gain_threshold
达到 max_depth
```

停止。

这些都只能映射为 Focused Stop Reason：

```text
FOCUSED_SUFFICIENT
FOCUSED_SATURATED
FOCUSED_MAX_PAGES
FOCUSED_MAX_DEPTH
FOCUSED_LOW_GAIN
FOCUSED_NO_LINKS
FOCUSED_IRRELEVANT
FOCUSED_CANCELLED
```

不能映射成 `BACKFILL_COMPLETE`。

### 5.1 `max_pages` 不是严格请求上限

当前循环在发批次前检查 `len(crawled_urls) >= max_pages`，然后仍可一次选择 `top_k_links` 个 URL。

例如：

```text
已抓 19
max_pages = 20
top_k_links = 3
```

下一批仍可能发 3 个请求，使实际尝试/成功页数超过 20。

因此生产预算必须由平台 Admission 单独控制：

```text
remaining = budget_pages - attempted_or_admitted
batch_size = min(top_k, remaining)
```

并分别记录：

```text
admitted_count
attempted_count
success_count
preview_request_count
embedding_call_count
```

`max_pages` 只能当 runtime hint，不能当硬配额。

---

## 6. Embedding Strategy 技术原理

### 6.1 Query Expansion

`map_query_semantic_space()` 先调用 chat completion 生成 query variations，再随机拆成 training 与 held-out validation queries，最后对 training queries 生成 embedding。

因此 embedding strategy 实际依赖两类模型能力：

```text
query expansion: chat model
semantic vector: embedding model
```

如果 query model 未显式配置，会走兼容回退，最终还可能落到硬编码默认 provider。生产必须显式配置 provider、model、credential、base_url、budget 和 release。

### 6.2 Query Expansion 存在输出 schema 脆弱点

当前 prompt 文本要求“返回 JSON 字符串数组”，但后续代码按：

```text
variations['queries']
```

读取对象字段。注释中的 mock 数据也是 `{"queries": [...]}`。

也就是说 prompt 契约与消费代码存在不一致风险。如果模型严格返回 JSON array，代码会失败。

生产不能依赖自由格式模型输出，应使用：

```text
明确 JSON Schema
+ validator
+ bounded repair
+ raw response artifact
+ variation_set_hash
```

### 6.3 Query Space Coverage

实现对 query embeddings 与 KB embeddings 做 L2 normalization/cosine distance，衡量每个 query variation 距离已有知识最近点有多远。

其本质仍是：

```text
query semantic coverage
```

不是 URL inventory coverage。

### 6.4 Gap-driven Link Selection

对 query semantic gaps，候选链接也生成 embedding。如果候选能把某个 gap 的距离拉近，就得到正向 improvement；如果候选与现有 KB 过于相似，则施加 overlap penalty。

这是该模块最值得复用的思想：

```text
昂贵抓取预算 -> 优先投给预计最能填 semantic gap 的候选
```

### 6.5 正文 embedding 只取前 5000 字符

当前 `EmbeddingStrategy.update_state()` 对新正文使用类似：

```text
content[:5000]
```

再做 embedding。

长技术文章中，核心主题、代码或结论可能位于后半段，因此这会产生系统性偏差。生产 `platform_embedding_v1` 应基于 Canonical IR 做 chunk/pooling：

```text
title + headings
+ representative chunks
+ optional max/mean pooling
```

不要用“正文前 5000 字符”作为长期语义真相。

### 6.6 Convergence + Validation

Embedding strategy 维护 confidence history；平均提升低于相对阈值时，使用 held-out queries 做 validation。validation 足够高才停止，否则继续抓。

这个设计比单一阈值更合理，但当前恢复实现并没有持久化完整 validation 状态，见第 7 节。

### 6.7 Irrelevant Query Fast Stop

当 confidence 低于 `embedding_min_confidence_threshold` 且已经抓过页面时，会设置：

```text
stopped_reason = below_minimum_relevance_threshold
is_irrelevant = true
```

适合 Web Admin 专题抓取快速拒绝明显不相关 query，节约 Browser/Embedding 成本。

---

## 7. Native Resume 在 Embedding 模式下并非完整语义恢复

这是生产集成中非常重要的一点。

`CrawlState.save()` 会保存 query embeddings、KB embeddings、expanded queries 等，但当前运行时还有一些状态存在于动态属性或 Strategy 实例上，例如：

```text
state.confidence_history       # 运行时动态添加，save() 未保存
strategy._validation_queries   # held-out queries，未保存到 CrawlState
strategy._validation_passed    # Strategy 实例字段，未保存
strategy caches                # 非业务状态
```

`validate_coverage()` 在没有 `_validation_queries` 时会退化为已有 confidence；因此 resumed embedding run 的 validation 语义可能和原运行不同。

另外 `digest(resume_from=...)` 会把 `state.query` 更新为调用时 query，却不会自动重建与新 query 匹配的 query embeddings。若没有平台级 `query_hash` 检查，就可能出现“新 query + 旧 query embeddings”的错误恢复。

因此：

```text
Native state file != deterministic embedding checkpoint
```

生产规则：

1. query_hash 必须匹配；
2. 保存完整 generated variation set；
3. 保存 train/validation split；
4. 保存 expansion raw artifact/hash；
5. 保存 confidence history 与 validation state，或明确重建；
6. model/prompt/analyzer/runtime release 任一变化就拒绝 resume；
7. 在未补齐这些语义前，embedding native resume 默认关闭。

### 7.1 Query split 还存在非确定性

源码对生成的 variations 使用 `random.shuffle()`，未看到固定 seed。因此同一 variation set 的 train/validation split也可能不同。

平台要实现可 Replay：

```text
保存实际 train_queries
保存实际 validation_queries
保存 split_seed（若使用）
```

不要只保存“n_query_variations=10”。

---

## 8. Link Preview 的网络语义

当前 `_crawl_with_preview()` 使用类似：

```text
include_internal = true
include_external = false
query = 当前 query
concurrency = 5
timeout = link_preview_timeout
max_links = 50
score_links = true
```

并会过滤掉没有 `head_data` 的 internal links。

### 8.1 Preview 会放大网络请求

一次正文抓取可能伴随多个 preview metadata 请求。因此网络治理必须覆盖 preview，不仅覆盖主 URL：

```text
Approved Scope
 -> RobotsPolicy
 -> SSRF Validation
 -> per-domain Token Bucket
 -> bounded preview concurrency
 -> request budget
 -> Audit
```

如果无法确保 Crawl4AI 内部 preview 的每个网络请求都经过平台 Access Policy，就不应在生产主链直接打开 native preview；改为平台先生成受控 preview artifact，再交给 scorer。

### 8.2 `head_data` 过滤会造成候选偏差

preview 失败、HEAD metadata 缺失或被策略限制的链接可能从 Adaptive runtime 候选中消失。

正确顺序必须是：

```text
Raw Link Observation
 -> Durable URL Inventory
 -> Scope/Robots/SSRF
 -> Preview/Score
 -> Focused priority
```

而不是：

```text
Adaptive filter
 -> 只保存幸存 URL
```

---

## 9. 当前 `digest()` 的批次结果关联风险

源码中的 `_crawl_batch()` 使用 `asyncio.gather()` 发起一批请求，然后把异常/失败结果过滤掉，只返回成功结果列表。

主 `digest()` 随后大致按：

```text
for result, (link, score) in zip(new_results, to_crawl):
    ...
```

关联结果和请求。

这会产生一个严重的 positional correlation 风险。例如：

```text
to_crawl    = [A, B, C]
实际结果     = [A成功, B失败, C成功]
_crawl_batch = [A结果, C结果]
zip 后       = A结果->A, C结果->B
```

此时 runtime 可能把 B 错标为 crawled，C 的新链接也可能被错误归因。

对 Persistent Frontier 来说这是不可接受的。

### 9.1 生产修复策略

优先级从高到低：

1. **平台不直接使用 native `digest()` 做业务调度**，由平台逐 URL/批次关联 `attempt_id + url_id`；
2. 若要使用 `digest()`，维护 vendor patch，使 batch outcome 保留一一对应位置或返回 `{requested_url, result, error}`；
3. 在补丁未验证前，生产将 `top_k_links=1` 作为保守规避；
4. Golden Test 必须包含“批次中间项失败”的关联测试。

平台最终写 Frontier/Artifact 时必须按稳定 request identity 关联，绝不能按“过滤后的数组位置”关联。

---

## 10. AdaptiveCrawler 实例不能跨 Run 并发复用

`AdaptiveCrawler` 把状态保存在实例字段：

```text
self.state
self.strategy
strategy caches
strategy._validation_passed
strategy._validation_queries
```

`digest()` 每次都会重置或覆盖 `self.state`；EmbeddingStrategy 还持有跨调用实例状态。

因此同一个 AdaptiveCrawler 实例如果并发执行多个 `digest()`，不同 Run 会互相覆盖状态。即使串行复用，也可能受到 `_validation_passed` 等字段残留影响。

生产规则：

```text
1 AdaptiveCrawler instance = 1 Focused Run lifecycle
```

不要做全局 singleton；Worker 可以复用底层 HTTP/Browser 资源池，但 Adaptive run state/strategy 必须隔离。

---

## 11. Persistent Frontier 与 Runtime pending_links 的边界

`pending_links` 是 list，runtime 没有平台数据库级唯一约束、Lease、Retry Budget、Fairness 或跨进程恢复语义。深层抓取时重复链接还可能增加评分开销。

平台继续使用：

```text
UNIQUE(discovery_run_id, url_id)
```

保证 Frontier 去重，并保存：

```text
parent_url_id
depth
priority
score
state
lease_owner
lease_until
fencing_token
attempt_count
reason_code
```

Adaptive 只生成/更新 score 与 focused admission，不拥有 URL 生命周期。

---

## 12. 推荐的生产集成架构

### 12.1 默认：Platform-owned Focused Scheduler

```text
Persistent URL Inventory
        |
        v
Candidate Slice Builder
        |
        +--> Platform Preview (scope/robots/SSRF/rate limited)
        |
        v
Focused Scorer
  - platform_statistical_v1
  - platform_embedding_v1(optional)
  - crawl4ai_baseline(optional)
        |
        v
Strict Budget Admission
        |
        v
Persistent Frontier priority update
        |
        v
HTTP / Browser / Repair Worker
        |
        v
Canonical IR / focused metric update
```

核心优势：

- 所有 URL 先 durable；
- 所有网络访问经过同一 Access Policy；
- 严格预算由平台控制；
- 请求结果按 `attempt_id/url_id` 关联；
- Retry/DLQ/Fair Scheduler 不依赖 runtime；
- Adaptive scorer 可替换、A/B、回放。

### 12.2 Crawl4AI AdaptiveCrawler 的角色

保留为：

- 行为 baseline；
- 小规模专题实验；
- 已打补丁的 bounded runtime；
- 算法参考实现；
- Offline Replay 对照。

不作为：

- Persistent Frontier；
- Coverage Truth；
- 全局 Scheduler；
- Robots/SSRF Access Engine；
- 业务 Checkpoint 真相。

---

## 13. 数据模型增量

建议：

```text
adaptive_run
- id
- source_id
- discovery_run_id
- query_hash
- strategy
- runtime_mode              # PLATFORM_SCORER/CRAWL4AI_NATIVE
- budget_pages
- budget_preview_requests
- state
- stop_reason
- runtime_release_id
- adaptive_policy_release_id
- analyzer_release_id
- query_expansion_release_id
- embedding_release_id
- admitted_count
- attempted_count
- successful_count
- preview_request_count
- started_at
- finished_at

adaptive_candidate_score
- adaptive_run_id
- url_id
- preview_artifact_id
- relevance
- novelty
- authority
- semantic_gap_gain
- expected_gain
- rank
- scorer_release_id
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

adaptive_query_set
- adaptive_run_id
- raw_expansion_object_key
- expansion_hash
- train_queries_object_key
- validation_queries_object_key
- split_seed
- captured_at
```

命名必须使用 `focused_* / topical_*`，避免和历史 `coverage_snapshot` 混淆。

---

## 14. Web Admin 能力

Focused Crawl 页面展示：

```text
query/topic profile
strategy/runtime mode
focused confidence
topical coverage/consistency/saturation
validation confidence
candidate expected gain/rank
admitted/attempted/successful
page/preview/model budget remaining
stop reason
checkpoint compatibility
runtime/model/analyzer release
cost
```

必须固定显示：

```text
Focused Sufficiency != Historical Coverage
```

还应支持：

- Compare：platform scorer vs Crawl4AI baseline；
- 查看某 URL 为什么被排到当前 rank；
- 查看 preview artifact 与 score components；
- 手工强制提升/降低优先级，但不删除 Observation；
- 查看 runtime checkpoint 为什么可/不可恢复。

---

## 15. Metrics

```text
adaptive_run_total{strategy,runtime_mode,status}
adaptive_stop_total{reason}
adaptive_candidate_total{decision}
adaptive_pages_admitted_total{source}
adaptive_pages_attempted_total{source}
adaptive_pages_success_total{source}
adaptive_preview_requests_total{source,status}
adaptive_embedding_calls_total{provider,model}
adaptive_focused_confidence{source}
adaptive_validation_confidence{source}
adaptive_budget_overshoot_total{type}
adaptive_result_correlation_error_total
adaptive_checkpoint_reject_total{reason}
adaptive_cost_total{source,strategy}
adaptive_browser_saved_total{source}
```

尤其要同时记录 admitted/attempted/success，避免把“成功抓了多少页”误当成“花了多少网络预算”。

---

## 16. Golden Test / 验收测试

### 16.1 Truth Boundary

```text
Adaptive 达 confidence threshold
=> Source Backfill 状态不得改变

Adaptive 因 saturation/max_pages/low_gain 停止
=> 未选 URL 仍在 Persistent Frontier

preview/head_data 失败
=> Raw Observation 不丢
```

### 16.2 Current Runtime Behavior

每个固定 Crawl4AI release 验证：

- Statistical confidence 是否仍硬编码 0.4/0.3/0.3；
- authority 是否仍固定 1.0；
- docs 描述的 expected gain 与源码真实公式是否一致；
- Link Preview 实际产生哪些请求；
- 无 `head_data` 是否被过滤；
- `min_gain_threshold` 在何处生效；
- `top_k_links` 是否限制每轮 fan-out；
- `max_pages` 是否会因批次 overshoot；
- batch 中间项失败是否导致 result/link 错配；
- 同一实例并发 digest 是否被禁止；
- runtime state 字段是否变化。

### 16.3 Statistical Analyzer

- Go/R/C/C++/C#/JS/AI/DB/S3/K8s 等技术 query 不被错误丢词；
- 中文 query 有稳定切词；
- template/navigation 不主导 saturation；
- config weights 确实改变 score；
- score 变化不删除 URL。

### 16.4 Embedding

- query expansion 返回 array/object 两种异常输入时由 schema validator 正确处理；
- query expansion/provider 429/timeout 有 bounded retry；
- raw variation set 和 train/validation split 可 replay；
- resumed run query hash 不一致必须拒绝；
- validation queries 未恢复时不得伪装成 validated；
- model/prompt release 变化拒绝旧 checkpoint；
- 长文后半段主题能通过 chunk/pooling 被表示；
- irrelevant query 可快速停止；
- convergence 但 held-out validation 低时继续；
- overlap penalty 只降 priority，不删除候选。

### 16.5 Budget / Correlation

```text
19/20 pages + top_k=3
=> 平台最多再 admission 1 个
```

```text
[A成功, B失败, C成功]
=> A/C 的 Artifact、Frontier、Observation 必须与各自 url_id 正确关联
```

### 16.6 Safety

- Preview 不访问未 approved host；
- DNS/redirect 每跳做 SSRF/scope 校验；
- Preview 请求进入全局 per-domain token bucket；
- external links 只保存 Observation/Candidate，未经批准不主动访问。

---

## 17. 对最终博客知识库方案的优化结论

本次调研对现有方案的主要增量不是“再加一个爬虫”，而是把 Adaptive Crawling 的边界收得更清楚：

```text
全历史正确性：
CMS/API + Sitemap + RSS + Archive
+ Common Crawl + Wayback
+ Safe Recon + Deep Crawl Gap
+ Persistent URL Inventory

高成本专题效率：
Platform-owned Focused Scheduler
+ Statistical/Embedding scorer
+ Strict Budget Admission
+ Browser/Repair priority
```

同时新增以下生产约束：

1. 平台不直接把 `AdaptiveCrawler.digest()` 当业务 Scheduler；
2. 修复/规避 batch 成功结果过滤后再 positional zip 的 URL 错配风险；
3. 平台硬控 page/preview/model budget，不能相信 `max_pages` 是严格上限；
4. 一个 AdaptiveCrawler 实例只服务一个 Run，不并发/跨 Run 复用；
5. embedding native checkpoint 在补齐 validation/query split 语义前不能作为确定性恢复；
6. query expansion 使用严格 schema，并保存实际 variation/train/validation 集；
7. 技术博客默认使用 platform analyzer，不直接依赖长度 >2 的简单 tokenizer；
8. 长文 embedding 基于 Canonical IR chunk/pooling，而不是只取正文前 5000 字符；
9. docs/source 行为漂移由 pinned release + Golden Behavior Test 管控；
10. Adaptive 的所有 score、confidence、saturation 都只决定 **when**，不决定 **whether**。

最终建议：**保留 Crawl4AI Adaptive Crawling 作为非常有价值的信息增益算法参考和可选 runtime，但生产主链采用 Platform-owned Focused Scheduler。** 这样既能获得 Browser/Deep Crawl/RAG 专题任务的成本收益，又不会牺牲 1000+ 博客全历史知识库最关键的 Coverage、可恢复性、可解释性和数据身份正确性。
