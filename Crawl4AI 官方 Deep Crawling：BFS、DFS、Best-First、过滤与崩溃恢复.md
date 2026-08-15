# Crawl4AI 官方 Deep Crawling：BFS、DFS、Best-First、过滤与崩溃恢复

## 1. 调研对象与定位

- 官方文档：https://docs.crawl4ai.com/core/deep-crawling/
- 官方仓库：https://github.com/unclecode/crawl4ai
- 重点源码：
  - `crawl4ai/deep_crawling/bfs_strategy.py`
  - `crawl4ai/deep_crawling/dfs_strategy.py`
  - `crawl4ai/deep_crawling/bff_strategy.py`
  - `crawl4ai/deep_crawling/filters.py`
- 重点能力：BFS、DFS、Best-First、FilterChain、URLScorer、`max_depth`、`max_pages`、`score_threshold`、stream、prefetch、crash recovery、cancel。

本调研只研究 Crawl4AI Deep Crawling 对“1000+ 技术博客全历史知识库”有价值的实现机制。结论是：**Crawl4AI 应作为站内深度 URL 发现与动态页面补洞执行器，而不是平台级 Coverage Truth、持久化 Frontier 或业务任务系统。**

---

## 2. 核心结论

Crawl4AI Deep Crawling 本质上是一个运行时 Frontier Traversal 状态机：

```text
起始 URL
  -> URL 规范化
  -> FilterChain
  -> URLScorer
  -> BFS / DFS / PriorityQueue
  -> arun_many()
  -> CrawlResult
  -> 提取 links
  -> 产生下一批 Frontier
```

它提供了很好的遍历算法、过滤、打分、流式返回、取消和恢复接口，但不能直接承担平台级“全历史完整性”的职责，原因包括：

1. `max_depth`、`max_pages` 是执行预算，不是完整性证明；
2. `score_threshold` 会永久跳过低分 URL，不适合作为全历史默认策略；
3. 内部 `visited` / `stack` / `queue` 是一次运行状态，不是跨任务业务真相；
4. 当前源码中部分 URL 会在真正进入下一批 Frontier 前就被标记为 seen/visited；
5. stream/batch 中失败或未返回的 URL 可能已经进入 visited；
6. `on_state_change` 是运行时 checkpoint callback，不与平台的 Artifact、Observation、Task 状态形成数据库事务；
7. 版本升级会改变过滤、去重、遍历和恢复行为，因此必须纳入 release pin、Golden Replay 和兼容性测试。

因此平台设计应坚持：

> **先保存 Link Observation，再做 Admission / Priority；平台数据库 Frontier 是真相，Crawl4AI Frontier 只是执行切片。**

---

## 3. URL 图与 Observation

站点可抽象为有向图：

```text
G = (V, E)
V = URL
E = 页面 A 中观察到指向页面 B 的链接
```

一个发现事件至少需要保留：

```text
source_id
from_url_id
to_url_id
raw_href
normalized_url
provider_run_id
traversal_policy_release_id
anchor_text
depth
observed_at
admission_decision
reason_code
```

关键点是 **Observation 与 Frontier Admission 必须分开**。页面里出现了一个链接，是事实；是否马上抓，是策略。

推荐状态：

```text
OBSERVED
  -> ADMITTED
  -> DEFERRED_BUDGET
  -> REJECTED_POLICY
  -> PRUNED_FOCUSED
```

对全历史 Backfill，`DEFERRED_BUDGET` 以后仍必须可重新进入 Frontier；不能因为本轮预算不足就永久消失。

---

## 4. BFS 实现原理

### 4.1 数据结构

当前 `BFSDeepCrawlStrategy` 的核心结构是：

```text
current_level: [(url, parent_url)]
next_level: [(url, parent_url)]
depths: {url: depth}
visited: set(url)
_pages_crawled
```

每轮取整个 `current_level`，通过：

```python
crawler.arun_many(urls=urls, config=batch_config)
```

并发抓取，成功结果再执行 `link_discovery()` 生成下一层。

这使 BFS 非常适合博客初次探测，因为博客的归档、分页、分类和年份入口通常位于浅层。

### 4.2 正确的平台化方式

不要把“一层 URL 数量”等同于真实并发量。一个大站的某层可能有几千个 URL，平台必须在 Crawl4AI 外层实施：

```text
Global Admission
Per-domain Token Bucket
Per-source Fair Slice
HTTP Worker Quota
Browser Context Quota
Memory Admission
Retry-After Backoff
```

也就是说保留 BFS 的层级语义，但执行时切成 bounded batch。

### 4.3 当前源码中值得特别注意的行为

`link_discovery()` 的顺序大致是：

```text
normalize
-> visited check
-> FilterChain
-> score
-> score_threshold
-> visited.add(url)
-> valid_links
-> 按 remaining_capacity 截断
-> next_level
```

这里有一个对知识库 Coverage 很关键的细节：**URL 会先加入 `visited`，然后才受 `max_pages` 剩余容量截断。**

因此若当前页面一次发现 100 个合法链接，但 `remaining_capacity=10`，其余 90 个链接可能已经进入运行时 visited，却没有进入下一层。这再次证明：

- Crawl4AI visited 不能代表“平台已覆盖”；
- 原始 Link Observation 必须在策略裁剪前由平台保存；
- 预算不足应记为 `DEFERRED_BUDGET`，下一轮可恢复。

### 4.4 `max_pages` 的真实计数语义

当前 BFS 只对 `result.success` 增加 `_pages_crawled`。因此 `max_pages=1000` 更接近“最多 1000 个成功结果”，而不是“最多尝试 1000 个 URL”。

平台指标必须分别记录：

```text
attempted_count
success_count
failed_count
observed_count
admitted_count
deferred_count
```

不能仅用 crawler 的 pages_crawled 判断成本和 Coverage。

---

## 5. DFS 实现原理

DFS 复用了 BFS 的过滤、打分、页数限制和取消机制，但 Frontier 改为：

```text
stack: [(url, parent_url, depth)]
```

每次 `pop()` 一个 URL；成功后发现新链接，并逆序压栈，使第一个发现的链接优先向深处继续。

当前实现还维护 `_dfs_seen`，用于在发现阶段去掉重复链接。恢复状态中会保存：

```text
visited
stack
depths
pages_crawled
dfs_seen
```

### 5.1 适用场景

DFS 适合路径结构明确、需要快速走到深层内容的站点，例如：

```text
/year/2020/month/01/post
/docs/version/module/page
/category/subcategory/post
```

但全历史默认使用 DFS 会产生明显路径偏置：在有限预算下，它可能长时间深挖某一个分支，而其它年份或分类尚未扫描。

因此建议：

- Backfill 初探默认 BFS；
- DFS 仅作为 Source Recipe 的明确选择；
- DFS 的低优先分支也必须保存在持久 Frontier 中。

### 5.2 DFS 的失败语义

当前实现会在真正抓取前把 URL 加入 `visited`；只有成功抓取才更新 checkpoint callback。若请求失败，单次 crawler 的 visited 与业务任务状态可能不一致。

平台必须独立维护：

```text
READY -> LEASED -> DONE
                -> RETRY
                -> DLQ
```

失败重试由平台 Task/Frontier 管理，不能依赖 DFS 内部 seen 集合。

---

## 6. Best-First 实现原理

### 6.1 PriorityQueue

当前实现使用 `asyncio.PriorityQueue`，队列项形如：

```text
(-score, depth, url, parent_url)
```

Python 小值优先，所以使用负分实现高分优先。

实现里设置 `BATCH_SIZE = 10`，每轮从优先队列取一批 URL 并发抓取。网络完成顺序可能不同，因此源码先收集：

```text
results_by_url[url] = result
```

随后再按照原始 batch 的优先队列顺序处理结果和发现新链接，避免网络抖动改变下一轮 Frontier 排序。

这个设计对可重放性非常重要。

### 6.2 平台必须增加稳定 tie-breaker

PriorityQueue 元组虽然已经隐含 depth/url 比较，但平台不应依赖第三方实现细节。持久化 Frontier 应明确使用稳定排序：

```text
priority DESC,
score DESC,
depth ASC,
first_enqueued_at ASC,
url_id ASC
```

同一个 release、同一批 Observation 在离线 replay 中应尽量得到相同调度顺序。

### 6.3 Scorer 的正确角色

URLScorer 应用于“何时抓”，不应用于“永远不抓”。例如：

```text
+5 article_path
+3 archive_path
+2 pagination
-5 tag_noise
-8 query_explosion
```

对全历史模式：

```text
score -> priority
```

对专题或预算型任务才允许：

```text
score < threshold -> prune
```

即区分：

```text
exploration_mode: threshold 只影响优先级
focused_mode: threshold 可以裁剪
```

---

## 7. FilterChain 的技术原理

`FilterChain` 将 URL Admission Policy 数据化。当前实现：

- filters 存成 tuple；
- 同步 filter 若直接返回 False，会立即短路；
- 异步 filter 收集后 `asyncio.gather()` 并发执行；
- FilterChain 与各 Filter 都记录通过/拒绝统计。

典型 Filter：

- `URLPatternFilter`：prefix、suffix、glob、regex；
- `DomainFilter`：允许/拒绝域；
- `ContentTypeFilter`：根据 URL extension 做快速预判；
- 内容相关 Filter：可按 head/内容相关性进一步判断。

### 7.1 Filter 是执行策略，不是事实销毁器

平台应把 Filter 的结果写成有原因码的决策：

```text
OUT_OF_SCOPE_DOMAIN
DENY_PATH
LOGIN_PATH
SEARCH_EXPLOSION
UNSUPPORTED_SCHEME
MEDIA_TYPE
FOCUSED_LOW_SCORE
```

而不是只留下“没有进入 queue”。这样 Web Admin 才能回答“为什么这个链接没有抓”。

### 7.2 URL extension 不是最终 Content-Type

`ContentTypeFilter` 适合快速排除 `.jpg/.zip/...`，但无扩展名 URL、伪扩展名、重定向都很常见。最终路由必须依据 HTTP Response `Content-Type` 和 payload 检测。

### 7.3 Source Root 必须由平台先校验

当前 BFS/Best-First 的 `can_process_url()` 对 depth 0 起始 URL绕过 FilterChain。也就是说，不能指望 FilterChain 替控制面防止错误 seed。

平台在创建 run 前必须验证：

```text
seed 属于 approved domain set
scheme 合法
Source 已启用
robots/access policy 已计算
traversal release 已发布
```

---

## 8. URL 规范化与 Identity

Crawl4AI 在 link discovery 中会先调用 `normalize_url_for_deep_crawl()` 再去重，这个顺序是正确的：

```text
raw href
-> resolve relative URL
-> normalize
-> dedupe
-> filter
-> score
-> enqueue
```

但平台仍需更完整的 Identity 层：

- fragment 清理；
- tracking parameter 清理；
- query 参数排序/白名单；
- host case / default port；
- HTTP redirect identity；
- `<link rel=canonical>`；
- AMP/print/mobile 等 alternate；
- 内容 hash 与近重复判定。

最终不能简单用 URL 字符串作为 Document ID。

---

## 9. Crash Recovery 的真实边界

### 9.1 Crawl4AI 提供的恢复能力

官方支持：

- `resume_state`：从之前状态继续；
- `on_state_change`：状态变化时外部持久化；
- `export_state()`：读取最近一次已捕获状态；
- `cancel()` / `should_cancel`：主动取消。

不同算法保存的状态包括：

```text
BFS:
visited, pending, depths, pages_crawled

DFS:
visited, stack, depths, pages_crawled, dfs_seen

Best-First:
visited, queue_items, depths, pages_crawled
```

### 9.2 为什么它不是 exactly-once checkpoint

Crawler checkpoint 与平台业务提交不是一个事务。典型故障窗口：

```text
页面抓取成功
-> crawler 更新 visited/pages_crawled
-> on_state_change 保存 crawler state
-> 尚未提交 Artifact / Observation / Task DONE
-> 进程崩溃
```

或者反过来：

```text
Artifact 已保存
-> 数据库事务未提交 checkpoint
-> 崩溃
-> 下次 resume 再抓一次
```

因此正确架构是：

```text
Crawl4AI resume_state = Worker 执行优化
PostgreSQL crawl_frontier = 业务真相
S3 Artifact = 不可变抓取事实
Transactional Outbox = 状态与消息一致性
```

平台接受 at-least-once 执行，通过稳定 ID、幂等键、payload hash、唯一约束和 fencing token 消除重复副作用。

### 9.3 checkpoint 必须可对账

每次 Worker 恢复时应做 reconciliation：

```text
crawler checkpoint pending
vs
DB frontier READY/LEASED
vs
artifact 已存在
```

数据库结果优先；第三方 crawler state 只能作为候选恢复输入。

---

## 10. Streaming 的语义与风险

stream 模式适合快速把 CrawlResult 推给下游，降低 TTFM 和单次内存占用，但不改变 Coverage Truth。

当前 BFS stream 会在一层开始时先执行：

```text
visited.update(urls)
```

也就是说即使某个 URL 后续请求失败或没有返回 result，它也可能已经在 crawler 的 visited 中。源码对“整个 batch 没有返回结果”也会避免无限循环，而不是构建平台级重试。

因此：

- stream 只控制结果交付方式；
- Retry、DLQ、幂等、Coverage 必须由平台实现；
- 不得用 stream 内部 visited 当“已经成功抓取”。

---

## 11. Prefetch：最适合千站 Backfill 的能力

Prefetch 的核心价值是把“发现 URL”和“生成知识库内容”拆开。

推荐两阶段：

```text
Phase A - Discovery Prefetch
HTTP/Browser -> HTML/links/minimal metadata
             -> Observation
             -> URL Inventory
             -> Frontier

Phase B - Full Fetch
Admitted article URL
-> Raw Artifact
-> Extraction Portfolio
-> Quality Gate
-> Canonical IR
-> Markdown
```

这样一个含大量分类页、标签页、分页页的网站，不必对每个导航页都执行 Markdown、媒体、全文抽取和增强，成本会明显下降。

但 Prefetch 仍应保留最小证据，例如 response metadata、parent URL、observed links 与 run/release 信息，否则后续无法解释 URL 来源。

---

## 12. Cancellation 与控制面

`cancel()` 使用内部 `asyncio.Event`；`should_cancel` 可接同步或异步 callback。当前实现通常在进入下一个 URL/批次前检查，因此取消不是“强杀当前网络请求”。

Web Admin 的取消语义应定义为：

```text
RUNNING
-> CANCELLING
-> 当前不可中断步骤完成
-> checkpoint / lease release
-> CANCELLED
```

必要时再由平台 Worker Supervisor 提供硬超时。硬杀后依靠数据库 lease 超时和 fencing token 接管。

---

## 13. 对当前版本的兼容性风险

直接阅读当前源码可以看到几个不应泄漏到平台业务语义的实现细节：

1. BFS 与 Best-First 构造函数当前使用 `FilterChain()` 作为默认参数；平台 Adapter 应始终显式为每个策略实例创建独立 FilterChain，避免依赖共享可变默认对象行为；
2. `can_process_url()` 当前要求 `netloc` 中包含 `.`，单标签 hostname 会被拒绝；公网博客通常不受影响，但测试/内网环境可能踩坑；
3. 起始 URL depth 0 绕过 FilterChain；Seed Scope 必须平台校验；
4. BFS/DFS 在容量裁剪前就可能将候选加入 seen/visited；业务 Coverage 必须先记录 Observation；
5. 成功页才计入 `_pages_crawled`，执行预算与请求预算不是同一概念；
6. checkpoint callback 主要围绕成功结果推进，失败重试语义不能外包给 crawler；
7. Best-First 当前批大小属于实现常量，平台调度器仍要有自己的资源预算和公平性。

这些都说明必须 **pin 版本 + Adapter 隔离 + 契约测试**，而不是把第三方 crawler 的内部状态直接暴露给业务层。

---

## 14. 建议增加的 Deep Crawl Adapter

平台不要让业务代码直接构造 Crawl4AI Strategy，增加一层：

```text
DeepCrawlAdapter
- validate_seed()
- build_filter_chain(release)
- build_scorer(release)
- build_strategy(release)
- map_result_to_observations()
- persist_observations_before_selection()
- reconcile_checkpoint()
- translate_stop_reason()
- emit_metrics()
```

输入只接受 immutable `TraversalPolicyRelease`，输出统一平台事件。

### 14.1 事件模型

```text
LinkObserved
UrlAdmitted
UrlDeferred
FetchSucceeded
FetchFailed
FrontierCheckpointed
RunCancelled
RunStopped
```

所有事件带：

```text
source_id
run_id
url_id
runtime_release
traversal_policy_release
trace_id
occurred_at
```

---

## 15. 必须增加的 Golden / Contract Tests

每次升级 Crawl4AI 前至少回放：

1. 自链接首页：不能导致无限或重复抓取；
2. 同一 URL 的 fragment/trailing slash/query 变体；
3. 单标签 host 的测试环境行为；
4. 每个 strategy 的 FilterChain 实例隔离；
5. `max_pages` 小于一次发现链接数时，未入选 URL 仍在平台 Observation/Deferred 中；
6. BFS batch 与 stream 的平台最终 Frontier 结果一致；
7. DFS crash 后恢复，不能丢失平台 READY URL；
8. Best-First 同分 URL 的稳定顺序；
9. `score_threshold` 在 exploration 模式不能永久删除 URL；
10. cancel 后 lease 能正确释放/超时接管；
11. 失败 URL 能由平台 Retry，不受 crawler visited 阻止；
12. `include_external=false` 下外链仍可作为 observation 记录，但不能自动扩大 Source Scope；
13. HTML 无扩展名、错误扩展名、重定向后的 Content-Type 路由；
14. old checkpoint 与 new runtime release 不兼容时拒绝盲目 resume，转平台级 reconciliation。

---

## 16. 对博客知识库总体方案的具体优化

本次调研建议在总体方案中明确加入以下内容：

### 16.1 Observation Before Selection

从 Deep Crawl 页面观察到的链接必须先持久化，再执行 Filter/Score/Budget 选择。防止 `max_pages`、threshold 或 crawler seen 集合造成不可解释缺口。

### 16.2 Frontier Decision 可审计

给每个 URL 保存：

```text
admission_state
reason_code
score
priority
policy_release
```

Web Admin 可直接显示“发现了但为什么没抓”。

### 16.3 Runtime Checkpoint 与 Business Checkpoint 分层

`resume_state` 只用于加速同一执行切片恢复；PostgreSQL Frontier + Artifact + Outbox 才是业务恢复依据。

### 16.4 Crawl4AI Compatibility Gate

运行时版本、Adapter 版本、Traversal Policy 全部冻结。升级先跑 Golden Replay，重点检测 visited、budget、filter、stream、resume、cancel 的行为变化。

### 16.5 指标补充

增加：

```text
observed_links_total
admitted_links_total
deferred_budget_total
rejected_policy_total
focused_pruned_total
crawler_pages_crawled
fetch_attempted_total
fetch_success_total
frontier_remaining
checkpoint_reconcile_total
```

这些指标比单一“爬了多少页”更能反映全历史任务是否可靠。

---

## 17. 最终架构判断

Crawl4AI Deep Crawling 很适合平台中的 **Deep Crawl Gap Provider / Browser Discovery Engine**，尤其适合：

- Sitemap/RSS/API 无法覆盖的站点；
- 归档和分页结构复杂的网站；
- JavaScript 才暴露链接的网站；
- 首轮浅层拓扑探测；
- 定期 Coverage Gap 扫描。

但它不应该直接成为：

- URL Inventory 真相；
- Coverage Complete 判断器；
- 持久任务队列；
- exactly-once checkpoint；
- Document Identity 系统；
- 全站唯一调度器。

最稳妥的组合是：

```text
Authoritative Providers
CMS/API + Sitemap + RSS + Archive
          |
          v
Persistent URL Inventory / Observation
          ^
          |
Crawl4AI Deep Crawl Prefetch
(BFS/DFS/Best-First Gap Provider)
          |
          v
Persistent Frontier + Fair Scheduler
          |
          v
HTTP First / Browser Last Fetch
          |
          v
Immutable Artifact -> Extraction -> Quality -> Canonical IR -> Markdown
```

其核心原则可归纳成一句话：

> **吸收 Crawl4AI 的遍历算法和执行能力，但把 Coverage、Observation、Frontier、Checkpoint 与版本治理提升为平台自己的持久化语义。**
