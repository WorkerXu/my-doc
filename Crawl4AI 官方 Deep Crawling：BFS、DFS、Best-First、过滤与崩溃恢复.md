# Crawl4AI 官方 Deep Crawling：BFS、DFS、Best-First、过滤与崩溃恢复

## 1. 调研对象

- 官方文档：https://docs.crawl4ai.com/core/deep-crawling/
- 官方仓库：https://github.com/unclecode/crawl4ai
- 重点源码：
  - `crawl4ai/deep_crawling/bfs_strategy.py`
  - `crawl4ai/deep_crawling/dfs_strategy.py`
  - `crawl4ai/deep_crawling/bff_strategy.py`
  - `crawl4ai/deep_crawling/filters.py`
- 重点版本能力：Deep Crawl、FilterChain、URLScorer、stream、max_pages、score_threshold、crash recovery、cancel、prefetch。

这篇调研不把 Crawl4AI 当成完整的“1000 个技术博客知识库平台”，而是分析它在整个平台里的一个关键位置：**站内深度 URL 发现与动态页面补洞执行器**。

---

## 2. 结论先行

Crawl4AI Deep Crawling 最值得吸收的不是“递归抓网页”本身，而是它把深度遍历拆成了一个明确的 **Frontier Traversal 状态机**：

1. 维护待访问 Frontier；
2. URL 规范化与去重；
3. FilterChain 决定 URL 是否进入 Frontier；
4. URLScorer 决定 Frontier 的优先级或阈值；
5. `max_depth`、`max_pages` 控制遍历边界；
6. `parent_url`、`depth` 保存发现关系；
7. batch / stream 决定结果返回方式；
8. `on_state_change` + `resume_state` 把 Frontier 状态外置，支持崩溃恢复；
9. `cancel()` / `should_cancel` 允许控制面主动终止长任务。

对于博客知识库，最重要的架构结论是：

> **Deep Crawl 应作为 Coverage Gap Provider，而不是 Coverage Truth。**

即 Sitemap、RSS、CMS API、Archive 等仍然优先，因为它们更容易证明历史覆盖；Deep Crawl 负责发现导航链、分页、标签页、年份归档、JavaScript 生成链接和其它权威入口遗漏的 URL。

同时，Crawl4AI 自带的 `visited` / `stack` / `queue` 只能作为一次执行的临时状态，不能直接替代平台级 URL Inventory、任务队列、幂等键和持久化 Crawl Frontier。1000 站规模必须把它的算法语义提升到数据库层和调度层。

---

## 3. Deep Crawling 的核心数据结构

### 3.1 URL 图

站点可以抽象为有向图：

```text
G = (V, E)
V = URL
E = 页面 A 中发现了指向页面 B 的链接
```

每个 URL 至少应携带：

```text
url
normalized_url
source_id
depth
parent_url
discovered_at
provider
score
state
```

Crawl4AI 在单次执行中通过 `visited`、`depths`、`current_level`、`stack` 或 `PriorityQueue` 管理这些信息，并把 `depth` 和 `parent_url` 写入 `CrawlResult.metadata`。

对知识库平台，应把这种关系持久化为：

```text
url_observation
- source_id
- from_url_id
- to_url_id
- provider_type = deep_crawl
- depth
- anchor_text
- observed_at
- run_id
- recipe_release
```

这样才能回答“某个 URL 是从哪里发现的”“为什么被抓”“为什么没有继续抓”。

### 3.2 URL 规范化

BFS、DFS、Best-First 在链接发现阶段都会先调用 `normalize_url_for_deep_crawl()`，然后再做 visited 去重。

这是非常关键的顺序：

```text
raw href
 -> resolve relative URL
 -> normalize
 -> dedupe
 -> filter
 -> score
 -> enqueue
```

如果先去重、后规范化，会把以下形式误认为不同页面：

```text
/post/1
/post/1#comments
https://example.com/post/1
https://example.com/post/1/
```

平台级实现还需要补充 query canonicalization、tracking parameter 清理、host case、default port、canonical tag 与 redirect identity，不能只依赖单次 crawler 的 normalize。

---

## 4. BFS：广度优先遍历原理与实现

### 4.1 算法语义

BFS 的 Frontier 是按“层”组织的：

```text
Depth 0: 首页
Depth 1: 首页直接链接
Depth 2: Depth 1 页面发现的链接
...
```

官方实现使用：

```text
current_level: List[(url, parent_url)]
next_level: List[(url, parent_url)]
depths: Dict[url, depth]
visited: Set[url]
```

每轮把 `current_level` 里的 URL 一次性交给 `crawler.arun_many()`，然后对成功结果做链接发现，生成下一层。

### 4.2 为什么 BFS 很适合博客历史发现

技术博客常见结构是：

```text
首页
 -> 分页 / page/2
 -> category
 -> archive/2025
 -> 文章详情
```

BFS 能优先扫描浅层导航和聚合页，因此在有限预算下通常比 DFS 更早覆盖更多栏目。

对 Backfill 首轮探测，可以使用类似策略：

```text
max_depth = 2~4
include_external = false
max_pages = 探测预算
```

先用浅层 BFS 建立站点链接拓扑，再识别：

- 分页模式；
- 年份归档；
- 分类/标签；
- 文章 URL pattern；
- 站内 sitemap 或 feed 入口。

随后转为确定性 Provider 扫描，而不是长期依赖无边界 BFS。

### 4.3 批处理并发

BFS 源码会把整层 URL 交给 `arun_many()`，这天然适合并发抓取。

但是平台不能把“这一层有多少 URL”直接等价为“同时开多少 Browser”。

必须在外层增加：

```text
Global concurrency
Per-domain concurrency
Browser-context quota
Memory-adaptive admission
Rate-limit / Retry-After
```

否则一个大站点某一层出现几千个 URL 时，可能瞬间把 Browser、内存和下游抽取服务打满。

因此平台应该保留 BFS 的“层级语义”，但把实际执行拆成受限 batch：

```text
BFS Frontier level
 -> Scheduler slice
 -> bounded arun_many batch
 -> persist results
 -> next scheduler slice
```

---

## 5. DFS：深度优先遍历原理与适用边界

### 5.1 源码结构

官方 `DFSDeepCrawlStrategy` 复用了 BFS 的过滤、打分、页数限制和取消机制，但把 Frontier 改成：

```text
stack: List[(url, parent_url, depth)]
```

每次 `pop()` 一个 URL，成功抓取后把新链接逆序压栈，使第一个发现的链接优先继续向下探索。

DFS 还维护独立的 `_dfs_seen` 集合，在“发现阶段”就去掉重复链接，避免重复把同一 URL 压入栈。

### 5.2 适用场景

DFS 适合路径形态明显、需要尽快走到深层详情页的站点，例如：

```text
/year/2020/month/01/post
/docs/product/version/module/page
/category/subcategory/post
```

但对于博客全历史覆盖，纯 DFS 风险更高：

- 容易在某一深链路花大量预算；
- 其它年份、栏目可能长时间没有访问；
- 如果 `max_pages` 较小，Coverage 会出现明显偏置。

因此平台里 DFS 更适合作为 **Recipe 级策略**，而不是默认策略。

---

## 6. Best-First：优先级 Frontier

### 6.1 算法结构

官方 `BestFirstCrawlingStrategy` 使用 `asyncio.PriorityQueue`。

队列项类似：

```text
(-score, depth, url, parent_url)
```

因为 Python PriorityQueue 默认小值优先，所以使用负分数实现“高分优先”。

执行过程：

```text
PriorityQueue
 -> 取最多 BATCH_SIZE 个高优先级 URL
 -> arun_many 并发抓
 -> 按原 priority 顺序处理结果
 -> 发现新链接
 -> scorer 重新计算分数
 -> 放回 PriorityQueue
```

源码特意把网络返回顺序和 Frontier 顺序解耦：即使 `arun_many()` 的请求完成时序不同，仍然按原优先级 batch 顺序处理，避免网络抖动改变遍历结果。

这个细节对可重放性很重要。

### 6.2 URLScorer 的作用

官方文档提供 KeywordRelevanceScorer 等 Scorer。

Best-First 的价值不是“只抓高分页面”，而是：

> 在固定抓取预算内，优先获得更可能是目标内容的页面。

对于博客平台，可定义确定性 scorer：

```text
score =
  + article_path_pattern
  + date_path_pattern
  + archive_path_pattern
  + pagination_pattern
  + canonical_same_domain
  - tag_noise
  - login/account/cart
  - search/query explosion
  - calendar infinite path
```

注意：**Scorer 只能改变处理顺序，不能替代 Coverage。**

如果目标是“全量历史”，低分 URL 不能永久丢弃，只能进入低优先级 Frontier 或 Gap Queue。

### 6.3 `score_threshold` 的风险

BFS / DFS / Best-First 都可配置阈值。

对内容检索任务，阈值能明显省成本；但对全历史知识库，阈值如果直接用于丢弃 URL，会造成不可解释的历史缺口。

建议平台实现两种模式：

```text
exploration_mode:
  threshold 只影响优先级，不永久排除

focused_mode:
  threshold 可直接 prune
```

Backfill 默认 exploration，专题采集或临时任务才允许 focused。

---

## 7. FilterChain：把“哪些 URL 可进入 Frontier”配置化

### 7.1 FilterChain 的实现

官方 `FilterChain` 持有不可变 filter tuple，并对每个 URL 应用全部 Filter。

同步 Filter 一旦返回 False，会立即短路；异步 Filter 会被收集并通过 `asyncio.gather()` 并发执行。

这说明 FilterChain 本质上是：

```text
URL Admission Policy
```

而不是正文抽取逻辑。

### 7.2 URLPatternFilter

实现里针对不同 pattern 做了优化：

- suffix；
- prefix；
- domain；
- path/glob；
- regex。

并使用 `lru_cache` 缓存 URL 匹配结果。

这适合把站点规则做成数据：

```yaml
include:
  - /blog/*
  - /posts/*
exclude:
  - /login/*
  - /search/*
  - /tag/*?page=*
  - /author/*
```

平台应把这些规则放入 `source_profile_release` / `interaction_recipe_release`，而不是硬编码到 Worker。

### 7.3 DomainFilter

全量博客抓取默认必须限制在 Source 的 canonical domain / approved domain set 内。

外链只能产生 `external_link_observation`，不能自动扩大 Collection Scope。

否则某个博客友情链接会把 crawler 扩散到整个互联网。

### 7.4 ContentTypeFilter

官方 Filter 可以按 URL extension 做快速类型排除。

知识库需要进一步区分：

```text
HTML -> HTML pipeline
PDF -> PDF pipeline
JSON/XML -> provider/parser pipeline
image/video/archive -> metadata only / skip
```

注意 URL extension 只是预判，最终仍要以 Response `Content-Type` 和实际 payload 为准。

---

## 8. `max_depth` 与 `max_pages`：预算不是完整性证明

`max_depth` 控制图的深度，`max_pages` 控制成功抓取页数。

它们适合保护执行预算，但不能作为“抓完了”的判断。

平台必须区分：

```text
run termination != coverage completion
```

一个 Crawl Run 因为以下原因结束：

```text
FRONTIER_EMPTY
MAX_DEPTH
MAX_PAGES
BUDGET
RATE_LIMIT
CANCELLED
ERROR
TIMEOUT
```

只有 `FRONTIER_EMPTY` 且 Provider Coverage Evidence 满足要求时，才可能接近“该 Discovery Strategy 扫描完成”。

因此 Web 管理后台需要展示：

- frontier remaining；
- discovered / fetched / passed；
- pruned by reason；
- known gaps；
- budget termination；
- depth histogram。

---

## 9. Crash Recovery：最值得平台化的能力之一

### 9.1 官方状态结构

BFS 的恢复状态包括：

```text
strategy_type
visited
pending
 depths
pages_crawled
cancelled
```

DFS 包括：

```text
visited
stack
depths
pages_crawled
dfs_seen
```

Best-First 包括：

```text
visited
queue_items(score, depth, url, parent_url)
depths
pages_crawled
```

`on_state_change` 在每个 URL 成功处理后输出当前状态，`resume_state` 可以在下一次构建 Strategy 时恢复。

### 9.2 平台不能直接把整个 JSON 当最终状态

对于 1000 个站和百万 URL，直接每页把整个 `visited` 集合序列化到 Redis/DB，会出现状态越来越大的问题。

因此应吸收“可恢复 Frontier”的语义，但改成增量持久化：

```text
crawl_frontier
- run_id
- url_id
- priority
- depth
- parent_url_id
- state: READY/LEASED/DONE/SKIPPED
- lease_until
- fencing_token

crawl_checkpoint
- run_id
- provider_cursor
- frontier_generation
- pages_crawled
- checkpoint_at
```

`visited` 不保存为一个巨型 JSON，而由 `url_identity + run_url_state` 唯一约束表示。

### 9.3 Exactly-once 不现实，应该追求幂等

Worker 可能在“页面已保存、状态未提交”时崩溃，所以恢复后同一个 URL 可能再次执行。

正确做法是：

```text
at-least-once execution
+ idempotent artifact write
+ unique task key
+ content hash
+ transactional state transition
```

而不是试图依赖 crawler 内存做到 exactly-once。

---

## 10. Cancellation：控制面必须能终止长任务

官方支持：

- `strategy.cancel()`；
- `should_cancel` callback；
- 每个 URL / batch 前检查取消。

平台应映射为：

```text
run.desired_state = CANCELLED
worker heartbeat -> should_cancel()
```

并保证取消不是直接 kill 进程，而是：

1. 当前 URL 尽量完成；
2. 保存 Artifact；
3. 持久化 Frontier；
4. 释放 lease；
5. Run 进入 CANCELLED；
6. 后续可以 Resume。

这样 Web 管理功能里的“暂停/停止 Backfill”才是真正可靠的。

---

## 11. Prefetch：Discovery 与 Full Fetch 解耦

Crawl4AI 新版本提供 `prefetch=True`，跳过 Markdown、Extraction、Media 等昂贵处理，主要获得 HTML 与链接。

这与知识库架构里的 `Coverage First` 非常一致：

```text
Phase 1: Discovery / Prefetch
 -> 快速发现 URL
 -> 建 URL Inventory

Phase 2: Full Fetch
 -> 按优先级抓正文
 -> 保存 Artifact
 -> Extract / Quality / Markdown
```

对于 1000 个技术博客历史回填，不能边发现一个 URL 就立刻执行完整正文管线，否则：

- 无法先判断站点规模；
- 资源调度失控；
- 大站会长时间占满 Browser；
- Coverage 无法解释。

所以技术方案应该显式增加 **Prefetch Discovery Mode**。

---

## 12. 对现有博客知识库方案的优化建议

### 12.1 增加持久化 Crawl Frontier

现有 `URL Inventory + Queue` 还需要明确 Frontier 模型：

```text
crawl_frontier
- source_id
- discovery_run_id
- url_id
- parent_url_id
- depth
- score
- strategy
- state
- reason
- lease
```

支持 BFS/DFS/Best-First 语义，但实际由平台 Scheduler 分批执行。

### 12.2 增加 Traversal Policy Release

Source Profile 里新增：

```yaml
traversal:
  mode: bfs | dfs | best_first
  max_depth: 3
  max_pages_per_run: 10000
  include_external: false
  prefetch: true
  filters: ...
  scorer: ...
  threshold_policy: priority_only
```

所有变更版本化，以便重放时解释为什么某条 URL 当时进入或没有进入 Frontier。

### 12.3 增加 Discovery Stop Reason

每个 Deep Crawl Run 必须记录明确终止原因，并在 Web Admin 显示。

如果因为 budget/max_pages 停止，系统应自动创建 `coverage_gap`，而不是把 Run 标为“完成”。

### 12.4 增加 Crawl Graph Evidence

保存：

```text
from_url -> to_url
anchor
first_seen
last_seen
depth
provider
```

用于：

- 解释 URL 来源；
- 找孤立文章；
- 识别分页链；
- 对比 Sitemap 与站内链接覆盖差异；
- 发现 URL 迁移或导航变化。

### 12.5 Best-First 只做调度，不做永久裁剪

全历史 Backfill 下：

```text
score -> priority
not -> permanent exclusion
```

除非规则明确属于 `deny`，否则低分 URL 应保留并进入低优先级队列。

### 12.6 Deep Crawl 执行器不能成为业务真相

Crawl4AI 的 resume JSON、crawler cache、visited set、browser state 全部是执行状态。

真正的业务状态仍应在 PostgreSQL + Object Storage。

---

## 13. 推荐落地流程

```text
Source Onboarding
    |
    v
Authoritative Discovery
CMS API / Sitemap / RSS / Archive
    |
    v
URL Inventory
    |
    +---- Coverage Gap ----+
                           |
                           v
                  Deep Crawl Prefetch
                   BFS / Best-First
                           |
               Filter + Normalize + Score
                           |
                           v
                  Persistent Frontier
                           |
                           v
                    URL Observation
                           |
                           v
                Full Fetch Scheduler
                   HTTP First
                   Browser Last
                           |
                           v
                 Immutable Artifact
                           |
                           v
            Extract -> Quality -> IR -> MD
```

推荐默认策略：

```text
Backfill:
  Sitemap/API/RSS 优先
  Deep Crawl = BFS prefetch
  Best-First = 调度优化
  DFS = 特定 Recipe

Incremental:
  Conditional Feed/API/Sitemap
  首页/归档浅层 prefetch
  定期低频 gap crawl
```

---

## 14. Web 管理功能应增加的页面

### Source / Discovery 页面

显示：

- 当前 traversal policy；
- provider coverage；
- frontier size；
- depth 分布；
- queued / leased / done / skipped；
- filter reject reason；
- scorer 分布；
- max_pages / budget stop；
- known gap。

### Run 页面

支持：

```text
Pause
Resume
Cancel
Requeue failed
Replay with new policy
Compare policy versions
```

### URL 详情

展示：

```text
URL Identity
Canonical URL
Discovered by
Parent URL
Depth
Score
Filter decisions
Fetch attempts
Artifact versions
Document versions
```

这样“Web 管理”才不仅是任务列表，而能真正解释覆盖与抓取决策。

---

## 15. 风险与限制

### 15.1 链接图不等于完整历史

孤立文章、旧文章、被导航移除的文章，Deep Crawl 无法凭空发现。

所以必须与 Sitemap、API、Feed、Archive、External Search Gap 组合。

### 15.2 动态无限空间

常见风险：

- calendar 无限翻页；
- 搜索参数组合；
- tag/filter 组合；
- session URL；
- tracking query；
- locale permutations。

必须依赖 URL normalization、query policy、FilterChain 和 run budget。

### 15.3 `visited` 作用域

单次运行 visited 不能当长期“不再抓”。

增量同步需要按：

```text
URL identity
last_success_at
etag
last_modified
content_hash
next_check_at
```

决定是否复抓。

### 15.4 评分漂移

如果 scorer 逻辑改变，Frontier 顺序也会改变。

因此 scorer 配置与代码必须进入 Release，并记录 `score` 与 `score_release`。

---

## 16. 最终判断

Crawl4AI Deep Crawling 对本知识库方案有明确增益，尤其是四点：

1. **把深度抓取抽象成 Frontier Traversal，而不是递归函数**；
2. **Filter + Score + Budget 让 URL Admission 可治理**；
3. **Crash Recovery / Cancel 证明长抓取任务必须有可恢复状态**；
4. **Prefetch 证明大规模采集应该先发现 URL，再执行完整内容处理**。

因此最终技术方案应新增：

- Persistent Crawl Frontier；
- Traversal Policy Release；
- Prefetch Discovery Mode；
- Crawl Graph Evidence；
- Stop Reason / Coverage Gap；
- Best-First Priority-only Policy；
- 控制面的 Pause/Resume/Cancel；
- Frontier 可观测指标。

这些改动能直接提升 1000+ 技术博客全历史回填的可扩展性、可恢复性、覆盖可解释性和 Web 运营能力。
