# Adaptive Web Crawling

来源：

- https://docs.crawl4ai.com/core/adaptive-crawling/
- https://docs.crawl4ai.com/advanced/adaptive-strategies/
- https://docs.crawl4ai.com/api/adaptive-crawler/
- https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/adaptive_crawler.py

> 本文基于当前 Crawl4AI v0.9.x 官方文档与 `main` 分支 `adaptive_crawler.py` 的实际行为分析。生产结论以 **pinned runtime release + Golden Behavior Test** 为准，不能只按文档描述推断实现。

## 1. 结论

Crawl4AI Adaptive Crawling 本质是一个 **query-driven information foraging（查询驱动的信息觅食）** 运行时：维护当前知识集合，对候选链接估计相关性、新颖性和语义缺口收益，并在“专题信息已经足够”、边际收益下降、达到页数/轮次预算或候选耗尽时提前停止。

它和本项目“1000+ 技术博客全历史回填 + 持续增量同步”的核心语义不同。官方文档也明确把 **Full Site Archiving** 和 **Real-time Monitoring** 列为不推荐场景。因此必须保持：

```text
Adaptive confidence != Historical Coverage
Adaptive stopped     != Backfill Complete
Adaptive pending     != Platform URL Inventory
Adaptive max_depth   != Platform graph depth
```

最适合的角色是：

```text
Focused Information-Gain Scheduler
+ Expensive Slow-path Optimizer
```

即先由 CMS/API、Sitemap、RSS/Atom、Archive、Common Crawl、Wayback、Safe Recon、Deep Crawl 等建立持久 URL Inventory，再让 Adaptive 思想对 Browser、Deep Crawl、Repair、专题知识包等昂贵候选决定“先抓谁”。低分、未选择、preview 失败的 URL 仍保存在 Persistent Frontier，不能被永久删除。

生产上 **不要直接把 `AdaptiveCrawler.digest()` 当平台调度器**。平台应拥有 Frontier、Admission、真实 depth、请求身份关联、重试、严格预算、访问策略和业务 checkpoint；Crawl4AI Adaptive 只作为评分 baseline、实验 runtime，或经过补丁与 Golden Test 的 bounded runtime。

---

## 2. CrawlState 与状态边界

`CrawlState` 当前维护的主要字段包括：

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

`save()` 会把 `crawled_urls`、运行时 Markdown、`pending_links`、词频、embedding、expanded queries 等序列化为 JSON；`load()` 再恢复。

这只是 **AdaptiveCrawler 的局部运行状态**，不能成为平台业务恢复真相，原因：

1. 没有平台级 Lease、Fencing Token、Transactional Outbox；
2. `pending_links/crawled_urls` 只是当前 runtime 视图；
3. 状态包含 Markdown、links、embedding，体积随任务增长；
4. runtime/source profile/traversal/model 变化后兼容性不受业务层保证；
5. Embedding Strategy 的部分关键状态根本没有进入 `CrawlState.save()`；
6. `digest()` 自己的轮次 `depth` 不是 `CrawlState` 字段，也不会持久化。

因此：

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
focused_scorer_release
analyzer_release
query_expansion_release
embedding_release
schema_version
state_object_key
state_sha256
captured_at
```

任一关键 release 或 query hash 不匹配时，拒绝原样 resume，从 PostgreSQL Frontier 重建 focused slice。

---

## 3. Statistical Strategy：实际算法

### 3.1 Coverage 是 query term coverage，不是 URL coverage

当前源码对 query 和 Markdown 做简单 tokenization，然后大致计算：

```text
doc_coverage = document_frequency / total_documents
freq_signal  = log(1 + term_frequency) / log(1 + max_term_frequency)
term_score   = doc_coverage * (1 + 0.5 * freq_signal)
coverage     = sqrt(avg(term_score))
```

最后 clamp 到 0~1。

它表示 query terms 在当前知识集合中的文本覆盖程度，不是站点历史 URL 覆盖率。

### 3.2 Consistency 实际是 Jaccard topical overlap

高级文档把 consistency 描述成页面之间是否“coherent/non-contradictory”，但当前源码实际对两两文档词集合计算 Jaccard：

```text
J(A,B) = |A ∩ B| / |A ∪ B|
consistency = pairwise Jaccard 平均值
```

所以更准确的语义是：

```text
topical_overlap / topical_coherence
```

不能解释成事实一致性、矛盾检测或来源可信度。

### 3.3 Saturation 是新词增长衰减

源码记录每个页面带来的新 unique terms 数量：

```text
new_terms_history = [page1_new_terms, page2_new_terms, ...]
saturation = 1 - recent_rate / initial_rate
```

局限：

- 首页异常会改变尺度；
- 模板、导航和代码噪声会制造“新词”；
- 新词数量不等于有价值信息；
- 多语言和技术术语高度依赖 tokenizer。

### 3.4 配置权重与当前行为有漂移

`AdaptiveConfig` 定义并校验：

```text
coverage_weight
consistency_weight
saturation_weight
```

但当前 `StatisticalStrategy.calculate_confidence()` 直接使用：

```text
confidence = 0.4 * coverage
           + 0.3 * consistency
           + 0.3 * saturation
```

也就是说“配置字段存在”不代表当前 release 真正读取该配置。

生产规则：任何可调参数只有通过 pinned-release 行为测试证明进入执行路径后，才允许出现在 Web Admin 的“有效配置”区域；否则标记为 runtime advisory/unsupported。

### 3.5 默认 tokenizer 不适合技术博客

当前逻辑接近：

```text
去标点 -> split whitespace -> 丢弃长度 <= 2 的 token
```

会系统性伤害：

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

中文无空格文本也无法获得稳定词粒度。

生产默认实现 `platform_statistical_v1`：

- Unicode/CJK-aware analyzer；
- 保留 C++、C#、.NET、Go、R、JS、AI、DB、S3、K8s 等技术 lexeme；
- title/heading/anchor/body 分字段；
- 使用平台 clean text / Canonical IR；
- 权重真实进入公式；
- component score 可解释；
- score 只影响 priority，不删除 URL。

---

## 4. Statistical 链接排序与文档/源码漂移

官方高级文档用：

```text
ExpectedGain = Relevance × Novelty × Authority
```

描述链接信息增益，但当前 StatisticalStrategy 实际执行加权和：

```text
score = relevance_weight * relevance
      + novelty_weight   * novelty
      + authority_weight * authority
```

默认权重约为 0.5 / 0.3 / 0.2。

### 4.1 Relevance

源码优先使用 `link.contextual_score`；否则退化为 query terms 与 link text/title/head metadata 的简单覆盖比例。因此函数注释里的“BM25 relevance”不能理解成该函数始终自己运行完整 BM25。

### 4.2 Novelty

当前估计近似：

```text
novelty = |link_terms - existing_terms| / |link_terms|
```

这是基于链接预览文本的预测，不等于正文真实新颖度。

### 4.3 Authority 当前实际是常数

当前 `rank_links()` 中：

```text
authority = 1.0
# authority = self._calculate_authority(link)
```

虽然源码存在 `_calculate_authority()`，但当前主路径没有真正用它区分候选。

结论：**第三方 runtime score 必须视为 advisory。** 平台保存 component score、input snapshot hash、runtime/scorer release 和行为测试结果，不能把文档公式直接升级为业务真相。

---

## 5. 停止条件与严格预算

Statistical Strategy 会因以下条件停止：

```text
confidence >= confidence_threshold
len(crawled_urls) >= max_pages
pending_links 为空
saturation >= saturation_threshold
```

主 `digest()` 还会因：

```text
ranked_links 为空
最高 score < min_gain_threshold
while 轮次达到 max_depth
```

停止。

这些只能映射到 Focused Stop Reason：

```text
FOCUSED_SUFFICIENT
FOCUSED_SATURATED
FOCUSED_MAX_PAGES
FOCUSED_MAX_ROUNDS
FOCUSED_LOW_GAIN
FOCUSED_NO_LINKS
FOCUSED_IRRELEVANT
FOCUSED_BUDGET_EXHAUSTED
FOCUSED_CANCELLED
```

绝不能映射成 `BACKFILL_COMPLETE`。

### 5.1 `max_pages` 不是严格请求上限

运行时是在发批次前检查 `len(crawled_urls) >= max_pages`，随后仍可能一次抓 `top_k_links` 个 URL。

```text
已抓 19
max_pages = 20
top_k_links = 3
=> 下一轮仍可能发 3 个请求
```

平台 Admission 必须硬控：

```text
remaining = page_budget - admitted_count
batch_size = min(top_k, remaining)
```

分别记录：

```text
admitted_count
attempted_count
successful_count
preview_request_count
browser_attempt_count
embedding_call_count
query_expansion_call_count
```

runtime `max_pages` 只是 hint。

---

## 6. `max_depth` 的真实语义：当前更像“扩展轮次”

当前 `digest()` 使用一个局部整数：

```text
depth = 0
while depth < config.max_depth:
    ...
    crawl top_k batch
    depth += 1
```

这个 `depth` 并不是每个 URL 在链接图上的 `parent_depth + 1`，而是 **整个 Adaptive Run 的批次轮次**。`pending_links` 又是跨页面累积的全局 list，所以当前 runtime 的 `max_depth` 不能直接解释成传统 BFS/DFS 的 graph depth。

更重要的是，`depth` 没有写入 `CrawlState.save()`。恢复时 `digest()` 会重新 `depth = 0`，因此 native resume 可以再次获得一整套 `max_depth` 轮次。

生产约束：

```text
platform_depth = persistent edge depth(parent_url_id -> url_id)
runtime_round  = adaptive batch round
```

两者必须分开存储。Traversal Policy 的真实 `max_depth` 由 PostgreSQL Frontier 在 Admission 前校验；runtime 的 `max_depth` 只作为每个 focused slice 的 `max_rounds` hint。

---

## 7. Embedding Strategy：Query Expansion 与语义覆盖

### 7.1 Query Expansion

Embedding Strategy 会调用 chat completion 生成 query variations，再随机拆成 training 与 held-out validation queries，最后为 training queries 生成 embedding。

因此它实际上依赖：

```text
query expansion chat model
+ embedding model
```

如果 query model 未显式配置，当前代码存在兼容回退，最终还可能落到默认 provider。生产必须显式指定 provider、model、credential、base_url、budget 和 release。

### 7.2 输出 Schema 契约存在脆弱点

当前 prompt 要求：

```text
Return as a JSON array of strings
```

但随后代码读取：

```text
variations['queries']
```

即 prompt 契约和消费代码并不一致。

生产改成：

```text
JSON Schema
+ validator
+ bounded repair
+ raw model response artifact
+ variation_set_hash
```

### 7.3 Train/Validation split 非确定性

源码对 variations 使用 `random.shuffle()`，未固定 seed，然后划分 80/20 train/validation。

为了 Replay，平台必须保存：

```text
actual generated variations
actual train_queries
actual validation_queries
split_seed（若有）
query_expansion_release
raw response hash
```

不能只保存 `n_query_variations`。

### 7.4 Gap-driven Link Selection

该策略最有价值的思想是：对 query semantic gaps 和候选 link embedding 估计“抓这个链接能否降低语义缺口”，再施加与已有 KB 的 overlap penalty。

适合用在：

```text
Browser / Deep Crawl / Repair / Topic Pack
高成本预算 -> 优先投给预计最能填 semantic gap 的候选
```

但只影响顺序，不能删除历史候选。

### 7.5 正文 embedding 只取前 5000 字符

当前 `update_state()` 对正文类似：

```text
content[:5000]
```

再生成 embedding。

长技术文章的关键结论/代码可能在后半段，因此生产 `platform_embedding_v1` 使用 Canonical IR：

```text
title + headings
+ representative chunks
+ optional mean/max/attention pooling
```

不把“正文前 5000 字符”当长期语义真相。

---

## 8. Embedding 当前生效的 confidence 逻辑也存在代码漂移

这是当前源码中非常重要的实现细节。

`AdaptiveConfig` 定义了多项 embedding confidence 参数，例如：

```text
embedding_coverage_radius
embedding_k_exp
embedding_nearest_weight
embedding_top_k_weight
```

源码里也保留了一大段“nearest + top-k + exponential decay”的旧/候选实现，但当前真正生效的 `calculate_confidence()` 更简单：

```text
Q = L2_normalize(query_embeddings)
D = L2_normalize(kb_embeddings)
best = max cosine similarity for each query
score = mean(best)
# 只有存在非标准 coverage_tau 属性时才切为 hit-rate
```

因此大量“看起来可调”的配置当前并不控制 active confidence path。

还有一个更强的行为漂移：当前 active `calculate_confidence()` 保存的是 `coverage_score`，而 `get_quality_confidence()` 读取 `state.metrics['learning_score']`。如果 `learning_score` 没有被其它路径写入，它会回退为 0；最终展示 confidence 的映射可能与刚刚算出的 active coverage score 不一致。

生产结论：

1. `crawl4ai_embedding_baseline` 只能作为实验行为；
2. Web Admin 不得把未验证的 runtime config 暴露成“生产生效参数”；
3. 每个 pinned release 建立 **config -> actual behavior** 测试矩阵；
4. 平台 `platform_embedding_v1` 自己定义明确、可测试的 score contract；
5. 评分输入、component score、最终 score、release 全部可回放。

---

## 9. Embedding Native Resume 不是确定性恢复

`CrawlState.save()` 虽然保存 query embeddings、KB embeddings、expanded queries 等，但以下状态当前存在于动态属性或 Strategy 实例中：

```text
state.confidence_history       # 动态添加，save() 未保存
strategy._validation_queries   # held-out queries，未保存
strategy._validation_passed    # 未保存
strategy caches                # 易失
runtime batch round depth      # 未保存
```

`validate_coverage()` 在没有 `_validation_queries` 时会退化为当前 confidence，而不是重新执行真正 held-out validation。恢复后的 validation 语义可能改变，甚至可能出现“没有原 held-out set 却按当前 confidence 继续判定”的情况。

另外 `digest(resume_from=...)` 会直接把 `state.query` 替换成调用时的新 query，但不会自动重建与新 query 匹配的 query embeddings。这可能形成：

```text
new query text + old query embeddings
```

因此：

```text
Native state file != deterministic embedding checkpoint
```

生产规则：

1. `query_hash` 必须完全匹配；
2. 保存完整 generated variation set；
3. 保存 train/validation split；
4. 保存 confidence history / validation state，或明确重建；
5. 保存 runtime round；
6. model/prompt/analyzer/runtime release 任一变化拒绝 resume；
7. 未补齐这些语义前，embedding native resume 默认关闭。

---

## 10. Link Preview 的网络与候选语义

当前 `_crawl_with_preview()` 近似配置：

```text
include_internal = true
include_external = false
query = 当前 query
concurrency = 5
timeout = link_preview_timeout
max_links = 50
score_links = true
```

随后会过滤没有 `head_data` 的 internal links。

### 10.1 Preview 会放大网络请求

正文抓取可能同时触发多个 metadata preview 请求。治理必须覆盖每个 preview：

```text
Approved Scope
 -> RobotsPolicy
 -> SSRF Validation
 -> per-domain Token Bucket
 -> bounded preview concurrency
 -> strict request budget
 -> Audit
```

如果不能证明 runtime 内部每个 preview 请求都经过平台 Access Policy，生产主链关闭 native preview，改用平台生成的受控 Preview Artifact。

### 10.2 `head_data` 与 `max_links=50` 会形成候选偏差

preview 失败、HEAD metadata 缺失，或超过 runtime preview 限制的链接可能不会进入 Adaptive pending list。

正确顺序必须是：

```text
Raw Link Observation
 -> Durable URL Inventory
 -> Scope/Robots/SSRF
 -> Preview/Score
 -> Focused priority
```

不能先让 Adaptive runtime 过滤，再只保存幸存 URL。

---

## 11. 批次结果关联存在 positional correlation 风险

`_crawl_batch()` 用 `asyncio.gather()` 发起一批请求，然后过滤异常/失败结果，只返回成功结果列表。

主 `digest()` 随后用：

```text
for result, (link, score) in zip(new_results, to_crawl):
```

关联结果和请求。

示例：

```text
to_crawl     = [A, B, C]
真实结果      = [A成功, B失败, C成功]
filtered      = [A结果, C结果]
zip 后        = A结果->A, C结果->B
```

可能导致 B 被错标 crawled、C 的结果和新链接被归因给 B。

生产修复：

```text
attempt_id + url_id + requested_url
```

必须随请求和结果一起传递，禁止任何过滤后的数组再和原数组按位置 zip。

未打补丁前，如果确需 native `digest()`，保守地 `top_k_links=1`，并以 Golden Test 覆盖“批次中间项失败”。

---

## 12. AdaptiveCrawler 实例不能跨 Run 并发复用

`AdaptiveCrawler` 把以下状态放在实例上：

```text
self.state
self.strategy
strategy caches
strategy._validation_passed
strategy._validation_queries
```

`digest()` 每次都会覆盖 `self.state`。同实例并发多个 `digest()` 会互相污染；串行跨 Run 复用也可能受 Strategy 私有状态残留影响。

生产规则：

```text
1 AdaptiveCrawler instance = 1 Focused Run lifecycle
```

Worker 可以复用 HTTP/Browser 资源池，但 Adaptive run state/strategy 必须隔离。

---

## 13. Persistent Frontier 与 runtime pending_links 的边界

`pending_links` 是进程内 list，没有数据库级唯一约束、Lease、Retry Budget、Fairness、真实 graph depth 或跨进程恢复语义。

平台继续使用：

```text
UNIQUE(discovery_run_id, url_id)
```

并保存：

```text
parent_url_id
graph_depth
priority
score
state
lease_owner
lease_until
fencing_token
attempt_count
reason_code
```

Adaptive 只能生成/更新 score 与 focused admission，不拥有 URL 生命周期。

---

## 14. 推荐生产集成

### 14.1 默认：Platform-owned Focused Scheduler

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
Strict Budget + Graph Depth Admission
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

每次评分保存：

```text
adaptive_run_id
url_id
candidate_snapshot_hash
preview_artifact_id
scorer_release
analyzer_release
query_set_hash
relevance
novelty
authority
semantic_gap_gain
expected_gain
rank
scored_at
```

这样才能做到 deterministic replay、A/B 和回归分析。

### 14.2 Crawl4AI AdaptiveCrawler 的角色

保留为：

- 行为 baseline；
- 小规模专题实验；
- 已打补丁的 bounded runtime；
- 算法参考实现；
- Offline Replay 对照。

不作为：

- Persistent Frontier；
- Historical Coverage Truth；
- 全局 Scheduler；
- Graph Depth Truth；
- Robots/SSRF Access Engine；
- 业务 Checkpoint 真相。

---

## 15. 数据模型建议

```text
adaptive_run
- id
- source_id
- discovery_run_id
- query_hash
- strategy
- runtime_mode
- budget_pages
- budget_preview_requests
- budget_embedding_calls
- budget_query_expansion_calls
- max_graph_depth
- max_runtime_rounds
- state
- stop_reason
- runtime_release_id
- adaptive_policy_release_id
- scorer_release_id
- analyzer_release_id
- query_expansion_release_id
- embedding_release_id
- admitted_count
- attempted_count
- successful_count
- preview_request_count
- embedding_call_count
- started_at
- finished_at

adaptive_candidate_score
- adaptive_run_id
- url_id
- candidate_snapshot_hash
- preview_artifact_id
- graph_depth
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
- runtime_round
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

字段命名坚持 `focused_* / topical_* / runtime_round / graph_depth`，避免与历史 `coverage_snapshot` 或传统 traversal depth 混淆。

---

## 16. Web Admin 与可观测性

Focused Crawl 页面展示：

```text
query/topic profile
strategy/runtime mode
focused confidence
topical coverage/consistency/saturation
validation confidence
candidate expected gain/rank
candidate snapshot hash
graph depth / runtime round
admitted/attempted/successful
page/preview/model budget remaining
stop reason
checkpoint compatibility
runtime/model/analyzer/scorer release
config effective/unsupported 状态
cost
```

固定显示：

```text
Focused Sufficiency != Historical Coverage
Runtime Round != Graph Depth
```

关键指标：

```text
adaptive_run_total{strategy,runtime_mode,status}
adaptive_stop_total{reason}
adaptive_pages_admitted_total{source}
adaptive_pages_attempted_total{source}
adaptive_pages_success_total{source}
adaptive_preview_requests_total{source,status}
adaptive_embedding_calls_total{provider,model}
adaptive_budget_overshoot_total{type}
adaptive_result_correlation_error_total
adaptive_checkpoint_reject_total{reason}
adaptive_depth_violation_total
adaptive_runtime_config_drift_total{field}
adaptive_replay_score_diff_total{scorer_release}
adaptive_cost_total{source,strategy}
adaptive_browser_saved_total{source}
```

---

## 17. Golden Behavior Test

### 17.1 Truth Boundary

```text
Adaptive 达 confidence threshold
=> Source Backfill 状态不得改变

Adaptive 因 saturation/max_pages/low_gain 停止
=> 未选 URL 仍在 Persistent Frontier

preview/head_data 失败
=> Raw Observation 不丢
```

### 17.2 Budget / Correlation

```text
19/20 pages + top_k=3
=> 平台最多再 admission 1 个
```

```text
[A成功, B失败, C成功]
=> A/C 必须和各自 url_id 正确关联
```

### 17.3 Depth / Resume

```text
runtime max_depth=3
=> 验证它是 batch round 还是 graph depth
```

```text
resume after 2 rounds
=> 平台 graph_depth 上限不因 runtime round 重置而被突破
```

### 17.4 Statistical

- 0.4/0.3/0.3 是否仍硬编码；
- authority 是否仍固定 1.0；
- docs ExpectedGain 与实际公式是否一致；
- Go/R/C/C++/C#/JS/AI/DB/S3/K8s 与中文 analyzer；
- config weight 改变是否真实改变 score。

### 17.5 Embedding

- prompt/schema 不一致输入是否被 validator 拦截；
- query expansion 429/timeout 有 bounded retry；
- actual train/validation set 可 replay；
- query hash 变化拒绝旧 checkpoint；
- held-out validation 未恢复时不得伪装 validated；
- active confidence 是否使用声明的 embedding 配置；
- `coverage_score` / `learning_score` / final display confidence 是否一致；
- 长文后半段主题能通过 chunk/pooling 被表示；
- irrelevant query 可快速停止；
- overlap penalty 只降 priority，不删候选。

### 17.6 Safety

- Preview 不访问未批准 host；
- DNS/redirect 每跳做 SSRF/scope 校验；
- Preview 进入全局 per-domain token bucket；
- external links 只保存 Observation/Candidate，未经批准不主动访问。

---

## 18. 对博客知识库技术方案的最终优化结论

Adaptive Crawling 给本项目带来的价值，不是“再加一个全站爬虫”，而是把昂贵抓取变成 **可解释、可预算的信息增益排序**。

最终架构保持：

```text
全历史正确性：
CMS/API + Sitemap + RSS + Archive
+ Common Crawl + Wayback
+ Safe Recon + Deep Crawl Gap
+ Persistent URL Inventory
+ Platform Graph Depth / Coverage Ledger

高成本专题效率：
Platform-owned Focused Scheduler
+ Statistical/Embedding scorer
+ Strict Budget Admission
+ Browser/Repair priority
+ Deterministic score replay
```

生产新增约束：

1. 平台不直接把 `AdaptiveCrawler.digest()` 当业务 Scheduler；
2. batch 结果必须按 `attempt_id/url_id/requested_url` 关联；
3. page/preview/browser/model/query-expansion budget 由平台硬控；
4. `max_depth` 视为 runtime round hint，平台 graph depth 单独持久化并硬控；
5. 一个 AdaptiveCrawler 实例只服务一个 Run；
6. embedding native checkpoint 在补齐 query/validation/round 语义前不宣称 deterministic resume；
7. query expansion 使用严格 Schema，并保存实际 variation/train/validation 集；
8. 技术博客默认使用平台 analyzer；
9. 长文 embedding 基于 Canonical IR chunk/pooling；
10. runtime 配置必须通过 `config -> actual behavior` Golden Test，未验证字段不作为生产控制面；
11. 所有 score 输入保存 snapshot/hash，支持 deterministic replay；
12. Adaptive 的 score、confidence、saturation 只决定 **when**，不决定历史 URL 最终 **whether**。

最终建议：**保留 Crawl4AI Adaptive Crawling 作为高价值算法参考与可选 bounded runtime，但生产主链采用 Platform-owned Focused Scheduler。** 这样可以获得 Browser/Deep Crawl/RAG 专题任务的成本收益，同时不牺牲 1000+ 博客全历史知识库最关键的 Coverage、真实深度、身份正确性、严格预算、恢复确定性和可解释性。