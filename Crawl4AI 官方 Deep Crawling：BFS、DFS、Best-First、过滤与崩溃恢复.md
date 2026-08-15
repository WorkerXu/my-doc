# Crawl4AI 官方 Deep Crawling：BFS、DFS、Best-First、过滤与崩溃恢复

## 1. 调研对象与结论

- 官方文档：https://docs.crawl4ai.com/core/deep-crawling/
- 官方仓库：https://github.com/unclecode/crawl4ai
- 重点源码：
  - `crawl4ai/deep_crawling/bfs_strategy.py`
  - `crawl4ai/deep_crawling/dfs_strategy.py`
  - `crawl4ai/deep_crawling/bff_strategy.py`
  - `crawl4ai/deep_crawling/filters.py`
- 重点能力：BFS、DFS、Best-First、FilterChain、URLScorer、`max_depth`、`max_pages`、`score_threshold`、stream、prefetch、crash recovery、cancel。

面向“1000+ 技术博客全历史抓取 + 持续增量同步 + Markdown 知识库”的结论是：

> **Crawl4AI Deep Crawling 非常适合作为动态页面、站内链接和 Coverage Gap 的发现执行器，但不应承担平台级 Coverage Truth、持久化 Frontier、任务幂等、版本真相或 exactly-once 恢复。**

平台应把它封装在 `DeepCrawlAdapter` 后面，只使用它的遍历、渲染、链接抽取、过滤、打分、流式结果、运行时恢复和取消能力；URL Observation、Frontier、Artifact、Document Version、Coverage 和任务状态必须由平台数据库维护。

---

## 2. Deep Crawling 的技术本质

站点可以抽象为有向图：

```text
G = (V, E)
V = URL
E = 页面 A 中观察到指向页面 B 的链接
```

Crawl4AI Deep Crawling 的核心循环可以抽象为：

```text
Seed
 -> Frontier
 -> Fetch / Render
 -> CrawlResult.links
 -> normalize_url_for_deep_crawl()
 -> FilterChain
 -> URLScorer
 -> BFS / DFS / PriorityQueue
 -> 下一批 Frontier
```

它内部维护的 `visited`、`current_level`、`stack`、`PriorityQueue`、`depths`、`pages_crawled` 都是“当前 traversal 的运行时状态”。这些状态能帮助一次运行继续执行，但没有平台所需的以下语义：

- URL 是从哪个 Provider、哪次 Run、哪个父页面发现的；
- 被过滤、被预算裁剪、被低分延后的 URL 是否仍需在以后抓取；
- Artifact 是否已持久化；
- Task 是否已通过 fencing token 提交；
- Document Version 是否已经生成；
- 当前状态对应哪个 Source Profile / Traversal Policy / Crawler Release；
- 多个 Worker 是否发生重复处理。

所以必须把“crawler traversal state”和“business state”分层。

---

## 3. BFSDeepCrawlStrategy 实现原理

### 3.1 数据结构

BFS 的核心数据结构是：

```text
current_level: [(url, parent_url)]
next_level: [(url, parent_url)]
depths: {url: depth}
visited: set(url)
_pages_crawled
```

批处理模式每一轮将当前层 URL 整体交给：

```python
crawler.arun_many(urls=urls, config=batch_config)
```

其中 `batch_config` 会把 `deep_crawl_strategy` 清空，避免递归套娃；当前 Strategy 自己掌握遍历顺序。成功结果再执行 `link_discovery()` 产生下一层。

这是一种“层级遍历 + 每层批量抓取”的实现。优点是浅层覆盖快，很适合博客首页、年份归档、分类、分页、文档导航等结构；缺点是某一层 URL 数量可能非常大，因此生产平台不能把“一个 BFS level”直接等价为“一个无限并发批次”。

### 3.2 Link Discovery 顺序

当前 BFS 的链接发现核心顺序大致为：

```text
读取 internal links
(+ include_external 时读取 external links)
 -> normalize
 -> visited check
 -> can_process_url / FilterChain
 -> URLScorer
 -> score_threshold
 -> visited.add(url)
 -> valid_links
 -> remaining_capacity / max_pages 截断
 -> next_level
```

这里对“全历史抓取”有一个非常重要的正确性风险：

**候选 URL 在 `max_pages` 剩余容量截断之前就可能进入 `visited`。**

例如当前页面发现 100 个合法文章链接，而本轮只剩 10 个页面容量：

```text
100 links
 -> 100 个加入 visited / valid_links
 -> remaining_capacity = 10
 -> 只保留前 10 个进入 next_level
```

其余 90 个已经不再是“未见过”，但也没有真正进入下一层。这说明 `visited` 不能作为平台的“已覆盖”或“已抓取”证明。

平台必须采用：

```text
Observation First
 -> Admission Decision
 -> Persistent Frontier
 -> Runtime Slice
```

所有观察到的链接先落 `url_observation`，预算不足只记为 `DEFERRED_BUDGET`，以后仍可进入 Frontier。

### 3.3 `max_pages` 的真实语义

当前实现只在 `result.success` 时增加 `_pages_crawled`。因此：

```text
max_pages != URL 尝试次数
max_pages ~= 成功页面计数上限
```

平台不能拿 `pages_crawled` 计算真实请求量、失败率或 Coverage。至少应独立记录：

```text
observed_count
admitted_count
attempted_count
success_count
failed_count
deferred_count
filtered_count
```

### 3.4 BFS Stream 的恢复边界

Streaming 模式会把当前层 URL 一次交给 `arun_many(..., stream=True)`。当前实现开始处理该层时会更新当前层的 `visited`，而 `on_state_change` 保存的 `pending` 主要来自已处理结果发现出来的 `next_level`。

因此在“当前层部分结果已经返回、进程突然终止”的窗口里，crawler state 并不等价于平台级完整的“所有未完成 URL 清单”。这不是说官方 crash recovery 没有价值，而是说明：

> **它适合恢复一个 traversal 执行上下文，不适合作为知识库系统唯一的 durable frontier。**

---

## 4. DFSDeepCrawlStrategy 实现原理

DFS 继承 BFS 的 URL 校验、过滤、打分、页数上限、取消等逻辑，但把 Frontier 改成栈：

```text
stack: [(url, parent_url, depth)]
```

每次：

```text
stack.pop()
 -> crawl one URL
 -> discover links
 -> reversed(new_links)
 -> push stack
```

源码额外维护 `_dfs_seen`，目的是在发现阶段去重，而不是把“已发现”与“已抓取”完全混在 `visited` 中。

恢复状态包含：

```text
visited
stack
depths
pages_crawled
dfs_seen
```

### 4.1 适用场景

DFS 适合路径层次明确、希望快速走到深层内容的 Source，例如：

```text
/year/2020/month/01/post
/docs/version/module/page
/archive/category/subcategory/post
```

但作为全历史 Backfill 默认策略会有路径偏置：预算有限时，可能一直深入某个分支，而其它年份、分类、归档入口还没有扫描。

因此建议：

- Backfill / Site Mapping 默认 BFS；
- DFS 只在 Source Recipe 明确声明时使用；
- 即使 DFS 当前不处理某分支，Observation 也必须写入持久 Frontier；
- 不能把 `dfs_seen` 当作平台“已经抓过”的事实。

### 4.2 DFS 与容量裁剪

DFS 的 `link_discovery()` 也会先把通过校验和阈值的 URL 加入 `_dfs_seen`，再按 `remaining_capacity` 截断 `valid_links`。所以同样存在：

```text
seen != admitted != fetched
```

这进一步证明平台必须把三种状态分开建模。

---

## 5. BestFirstCrawlingStrategy 实现原理

### 5.1 PriorityQueue

Best-First 使用 `asyncio.PriorityQueue`，队列项为：

```text
(-score, depth, url, parent_url)
```

Python 的 PriorityQueue 小值优先，所以把 score 取负数后，高分 URL 优先。

当前实现还有固定：

```text
BATCH_SIZE = 10
```

每轮从优先队列拿一批 URL 并发抓取。为了避免网络完成顺序破坏优先级语义，它先收集：

```text
results_by_url[result.url] = result
```

然后再按原 batch 的队列顺序处理结果、生成下一批链接。这一点很重要：如果“谁先请求完成谁先扩展”，Best-First 会受到网络抖动影响，难以重放；按原优先级顺序处理能降低这种非确定性。

### 5.2 Scorer 的正确角色

对知识库全历史任务，Scorer 应当决定：

```text
何时抓
```

而不是决定：

```text
永远不抓
```

例如：

```text
+5 /blog/ article path
+4 /archive/ /year/
+3 pagination
+2 title / anchor looks like article
-5 tag/search noise
-8 query explosion
```

全历史模式建议：

```text
score -> priority
```

专题、预算型、搜索型任务才允许：

```text
score < threshold -> prune
```

也就是在配置上显式区分：

```text
coverage_mode: threshold_policy = priority_only
focused_mode:  threshold_policy = prune
```

### 5.3 Best-First 也不能替代持久 Frontier

当前实现以小批次弹出 PriorityQueue，URL 在弹出后进入 `visited`，再并发请求。队列 shadow 只在开启 `on_state_change` 时维护，用于序列化恢复状态。

这类内存 PriorityQueue 很适合作为单 Worker 的执行结构，但没有：

- lease；
- fencing token；
- 跨 Worker 抢占；
- retry / DLQ；
- per-source fairness；
- 事务提交；
- 跨 release 兼容性。

平台仍应把 Best-First 的 score 映射到 PostgreSQL `crawl_frontier.priority/score`，由全局调度器取队。

---

## 6. FilterChain 的实现与边界

官方文档提供的主要过滤器包括：

- `URLPatternFilter`；
- `DomainFilter`；
- `ContentTypeFilter`；
- `ContentRelevanceFilter`；
- `SEOFilter`。

源码中的 `FilterChain` 将 filters 保存为 tuple。调用 `apply(url)` 时：

1. 同步 filter 立即执行；同步返回 False 会短路；
2. 异步 filter 会收集为 tasks；
3. 使用 `asyncio.gather()` 并发等待异步过滤结果；
4. 维护 total / passed / rejected 统计。

### 6.1 Filter 是策略，不是事实删除器

平台需要把 Filter 结果显式记录为原因码，例如：

```text
OUT_OF_SCOPE_DOMAIN
UNSUPPORTED_SCHEME
DENY_PATH
LOGIN_PATH
SEARCH_EXPLOSION
MEDIA_EXTENSION
FOCUSED_LOW_SCORE
ROBOTS_DENY
```

否则 Web 管理端无法回答：

> “这个 URL 为什么没抓？”

全历史模式中，只有明确的安全/范围/协议/访问策略规则才可以永久拒绝；低相关性、预算不足和低分更适合 `DEFERRED`。

### 6.2 ContentTypeFilter 只是 URL 扩展名快速判断

当前 `ContentTypeFilter` 会根据 URL 文件扩展名映射 MIME；无扩展名时通常放行。因此它适合在 Discovery 阶段快速减少明显的图片、压缩包、二进制资源，但不能替代 Fetch 后真实的 HTTP `Content-Type` 判断。

最终内容路由仍应依据：

```text
HTTP Content-Type
payload magic bytes
redirect final URL
业务 Source Recipe
```

### 6.3 起始 URL 绕过 FilterChain

BFS / Best-First 的 `can_process_url()` 对 depth 0 的 Seed 不执行普通 FilterChain。因此平台不能把 seed 安全边界交给第三方 Filter。

控制面创建 Run 前必须验证：

```text
scheme in http/https
seed host 属于 approved_domains
Source enabled
robots/access policy 已计算
Traversal Policy 已发布
credential / proxy policy 合法
```

### 6.4 默认 FilterChain 实例不要依赖

当前 Strategy 构造函数形态仍能看到类似：

```python
filter_chain: FilterChain = FilterChain()
```

`FilterChain.add_filter()` 又会修改该实例。因此平台 Adapter 应始终显式创建新的 FilterChain，不依赖第三方默认对象；这样也能屏蔽未来版本修复或默认行为改变。

---

## 7. URL 规范化与 Identity

Deep Crawl 在链接发现阶段先调用 `normalize_url_for_deep_crawl()`，再去重、过滤，这是正确顺序：

```text
raw href
 -> resolve relative URL
 -> normalize
 -> dedupe
 -> filter
 -> score
 -> enqueue
```

但平台级 Document Identity 必须比 crawler URL 去重更完整：

- 去 fragment；
- host 大小写和默认端口归一化；
- tracking query 参数清理；
- query 参数排序、白名单；
- redirect chain；
- `<link rel="canonical">`；
- AMP / print / mobile alternate；
- CMS post id / GUID；
- 内容 hash；
- SimHash / MinHash 近重复候选。

因此应区分：

```text
raw_url
normalized_url
canonical_url
document_identity_key
```

不能直接用 URL 字符串作为最终 Document ID。

---

## 8. Streaming 与 Non-Streaming

官方 Deep Crawling 支持：

```text
stream=False -> 完成后返回列表
stream=True  -> Async Iterator 逐步返回
```

对平台更推荐流式消费，因为：

- 结果可以更早写 Artifact；
- 可降低大 Run 的内存压力；
- 更方便实时展示运行进度；
- 更容易对后续 Extract / Quality 形成流水线。

但“stream”不是“无限吞吐”。平台仍必须有：

```text
bounded queue
per-domain concurrency
per-source fairness
browser semaphore
memory admission
backpressure
```

流式结果消费时也不能在收到一页后直接把所有 child URL 无限制 fan-out。

---

## 9. Prefetch：非常适合两阶段站点发现

官方 `CrawlerRunConfig(prefetch=True)` 使用快速路径：

- 抓 HTML；
- 快速抽取 links；
- 跳过 Markdown 生成；
- 跳过完整 content scraping；
- 跳过 media extraction；
- 跳过 LLM extraction。

官方把它定位为 site mapping、link validation、selective deep crawl、crawl planning 的能力。

这与知识库平台最适合的架构高度一致：

```text
Phase A: Discovery / Prefetch
  -> URL Observation
  -> Persistent Frontier

Phase B: Full Fetch / Extract
  -> Immutable Raw Artifact
  -> Candidate Extraction
  -> Quality Gate
  -> Canonical IR
  -> Markdown
```

但平台仍应坚持 **HTTP First, Browser Last**：如果 Sitemap、RSS、CMS API、普通 HTTP HTML 已经能发现 URL，就没有必要为了 prefetch 强制走浏览器。Crawl4AI prefetch 应主要服务于 JS 导航、动态链接和权威 Provider 漏洞。

---

## 10. Crash Recovery：能力与真实边界

### 10.1 官方能力

官方当前文档给三种核心接口：

- `resume_state`：从之前保存的状态恢复；
- `on_state_change`：状态变化时调用外部回调；
- `export_state()`：获取最近一次捕获的 state。

BFS state 主要包含：

```text
strategy_type
visited
pending
depths
pages_crawled
cancelled
```

DFS 还包含：

```text
stack
dfs_seen
```

Best-First 使用：

```text
queue_items
```

官方示例用 Redis 存 JSON state，这是合理的单 traversal 运行时恢复方式。

### 10.2 为什么不是业务 exactly-once

Crawler state 和平台业务提交不在同一事务中。典型故障窗口：

```text
Fetch 成功
 -> crawler 更新 visited/pages_crawled
 -> callback 保存 resume_state
 -> Raw Artifact 尚未完成持久化
 -> 进程崩溃
```

也可能反过来：

```text
Raw Artifact 已写 S3
 -> DB Task 尚未 DONE
 -> checkpoint 尚未更新
 -> 崩溃
 -> 恢复后再次抓取
```

所以平台要接受“抓取至少一次”，再通过幂等键和内容 hash 收敛，而不是试图把第三方 crawler checkpoint 宣称为 exactly-once。

正确边界：

```text
Crawl4AI resume_state = Runtime Hint / Optimization
PostgreSQL crawl_frontier = Business Recovery Truth
S3/MinIO Artifact = Fetch Fact
Transactional Outbox = DB State 与 Queue Message 一致性
```

### 10.3 State 写放大

`on_state_change` 会传递包含 `visited`、`pending/stack/queue`、`depths` 的 JSON。随着 Run 增大，单次 state 体积也会增长。如果每个 URL 都把完整 state 直接同步写入主数据库，会形成明显的序列化和 I/O 写放大。

推荐：

```text
小型 Run：直接 checkpoint JSON
大型 Run：
  - 业务 Frontier 逐行更新
  - crawler state 仅作为周期性 runtime snapshot
  - state 存 Object Storage / Redis
  - PostgreSQL 只存 pointer + hash + release metadata
```

### 10.4 Resume 兼容性必须验证

恢复前至少验证：

```text
source_id
seed_url
strategy_type
crawler_runtime_release
source_profile_release
traversal_policy_release
filter/scorer release
checkpoint schema_version
```

只要遍历语义版本发生改变，就不要盲目把旧 `resume_state` 喂给新版本 crawler；应从 PostgreSQL Frontier 重建执行切片。

---

## 11. Cancellation：必须映射成平台任务状态机

官方支持两种取消方式：

```text
strategy.cancel()
should_cancel callback
```

并提供 `strategy.cancelled`。官方语义是当前正在处理的工作可以完成，然后在后续边界停止。

源码实现显示取消检查粒度会因 Strategy 不同而不同：

- DFS 更接近“每个 URL 前检查”；
- Best-First 在 batch 前检查；
- BFS 在 level / batch 边界检查。

因此 Web 管理端不能把“用户点击停止”立即显示为“已停止”。推荐状态：

```text
RUNNING
 -> STOP_REQUESTED
 -> DRAINING
 -> STOPPED
```

控制逻辑：

1. DB 写 `STOP_REQUESTED`；
2. Scheduler 不再发新 lease；
3. `should_cancel` 查询轻量取消状态；
4. 同进程时额外调用 `strategy.cancel()`；
5. 当前 in-flight HTTP/Browser 请求允许完成或由上层 timeout 控制；
6. Worker 提交最后 Artifact / Observation；
7. 释放 lease；
8. Run 进入 `STOPPED`。

这样可以避免“按钮显示停止但后台仍在大量抓取”的运营错觉。

### 11.1 Callback 失败策略需要平台自己定义

当前源码中 `should_cancel` callback 异常会被日志记录并 fail-open 继续抓取；而 `on_state_change` 的 await 不应被假设为同样的 fail-open 语义。

因此 Adapter 应显式包裹 callback：

```text
cancel check 失败：按策略 fail-open 或 fail-closed，并记录 reason
checkpoint 失败：记录 CHECKPOINT_DEGRADED；业务 Frontier 仍为真相
```

不要把第三方 callback 异常策略传播成不可解释的平台行为。

---

## 12. 对 1000+ 博客平台的适配方案

### 12.1 DeepCrawlAdapter

业务层只依赖稳定接口：

```text
DeepCrawlAdapter.discover_slice(
    source,
    seed,
    traversal_release,
    runtime_checkpoint?,
    cancellation_token,
)
 -> async stream ObservationBatch
```

Adapter 负责：

```text
validate_seed
build_fresh_filter_chain
build_scorer
select bfs/dfs/best_first
pin crawler release
set bounded max_pages slice
map should_cancel
capture runtime checkpoint
normalize CrawlResult
emit raw link observations
translate third-party errors
```

业务层不直接 import Strategy 私有字段。

### 12.2 Observation First

推荐流水线：

```text
CrawlResult
 -> quick_extract_links / links
 -> write url_observation(raw_href, from, to, depth, evidence)
 -> normalize identity
 -> classify scope
 -> admission reason
 -> persistent crawl_frontier
```

第三方 Filter/Scorer 可作为“候选决策器”，但平台需要保留决策前的事实。

### 12.3 Bounded Slice

不要让一个大型站点一次 Deep Crawl 占满系统。用外层调度器切片：

```text
source A: 200 URL slice
source B: 100 URL slice
source C: 200 URL slice
incremental queue: 始终高于 bulk backfill
```

每次 slice 结束后把未处理候选留在 PostgreSQL Frontier，再由下一轮继续。这同时解决：

- 公平调度；
- 取消延迟；
- Worker 重启；
- 大 BFS level；
- Browser 成本失控；
- 单进程内存膨胀。

---

## 13. 平台数据模型建议

### 13.1 URL Observation

```text
url_observation
- id
- source_id
- provider_run_id
- from_url_id
- to_url_id
- raw_href
- anchor_text
- depth
- crawler_score
- admission_state
- reason_code
- traversal_policy_release_id
- observed_at
- evidence_object_key
```

### 13.2 Persistent Frontier

```text
crawl_frontier
- id
- source_id
- discovery_run_id
- url_id
- parent_url_id
- depth
- score
- priority
- strategy
- state          # READY/LEASED/DONE/RETRY/SKIPPED/FAILED
- skip_reason
- lease_owner
- lease_until
- fencing_token
- first_enqueued_at
- updated_at
```

唯一键建议：

```text
UNIQUE(discovery_run_id, url_id)
```

稳定取队：

```text
priority DESC,
score DESC,
depth ASC,
first_enqueued_at ASC,
url_id ASC
```

### 13.3 Runtime Checkpoint Envelope

不要只保存 crawler 原始 JSON：

```text
runtime_checkpoint
- checkpoint_id
- source_id
- discovery_run_id
- seed_url
- strategy_type
- crawler_release
- traversal_policy_release_id
- schema_version
- state_object_key
- state_sha256
- pages_crawled
- captured_at
- superseded_at
```

恢复时先校验 envelope，再读取 state。

---

## 14. Web 管理功能映射

Deep Crawl 相关页面至少展示：

### Source / Coverage

- Provider URL 数量；
- Deep Crawl Observation 数量；
- admitted / deferred / rejected 数量；
- 当前最大深度；
- 按 depth 的 URL 分布；
- Known Gap；
- 本轮停止原因。

### Run / Frontier

- Strategy；
- Traversal Policy Release；
- `max_pages` 仅显示为 runtime guard；
- Ready / Leased / Done / Retry / Failed；
- crawler pages_crawled 与平台 attempted/success 分开展示；
- checkpoint 时间与 release；
- STOP_REQUESTED / DRAINING / STOPPED。

### URL Explain

输入任意 URL 能看到：

```text
由哪个页面发现
哪个 Provider / Run
原始 href
规范化结果
Filter / Admission 决策
score / priority
是否进入 Frontier
抓取记录
Artifact
Document Version
```

这是 1000 站长期运营时非常关键的可解释性能力。

---

## 15. 测试与版本升级

Crawl4AI 是活跃项目，Deep Crawling 行为会随着版本变化。因此必须固定 release 并建立 Golden Replay。

### 15.1 Golden Site Fixtures

覆盖：

- 普通博客分页；
- 年份归档；
- 无限 query 参数；
- canonical / redirect；
- 动态 JS 导航；
- 同一 URL 多 parent；
- 大 BFS level；
- 低分但合法文章；
- 失败后 retry；
- 运行中 kill / resume；
- 用户主动 cancel。

### 15.2 必测不变量

```text
所有观察到的合法链接都有 Observation
预算裁剪不会让候选永久消失
低分只降优先级（coverage mode）
重复执行不会重复生成 Document Version
旧 checkpoint 不跨不兼容 release 恢复
cancel 后不再获取新 lease
Worker kill 后 Frontier 可继续
```

### 15.3 第三方实现回归点

Adapter 测试需特别覆盖：

- `max_pages` 计数边界；
- BFS level / stream 顺序；
- DFS `_dfs_seen`；
- Best-First queue 顺序；
- FilterChain 默认实例隔离；
- seed filter bypass；
- callback 异常；
- resume state schema；
- cancellation granularity；
- URL normalize 行为。

---

## 16. 对最终技术方案的具体优化结论

本次调研对博客知识库方案应落地以下增强：

1. 保留 **Deep Crawl 只是 Gap Provider** 的定位；
2. 正式采用 **Prefetch Discovery -> Full Fetch/Extract** 两阶段模式；
3. 强制 **Observation Before Selection**；
4. `score_threshold` 在全历史模式只转为优先级，不做永久裁剪；
5. `max_pages` 只作为 runtime guard，剩余 URL 进入 `DEFERRED_BUDGET`；
6. PostgreSQL `crawl_frontier` 为恢复真相，Crawler `resume_state` 只是执行优化；
7. 增加 Runtime Checkpoint Envelope 和 release 兼容检查；
8. 增加 `STOP_REQUESTED -> DRAINING -> STOPPED` 协作式取消状态机；
9. `should_cancel` / `cancel()` 统一封装到 Adapter；
10. 大站 Deep Crawl 采用 bounded slice，不把完整 BFS level 直接变成无界系统并发；
11. 所有 Filter、Scorer、Strategy、Crawler Runtime 均版本化并进行 Golden Replay；
12. Web Admin 增加 URL Explain、Frontier、Checkpoint、Cancellation、Coverage Gap 的可视化。

最终原则可以浓缩为：

> **Crawl4AI 负责“怎么遍历和怎么抓”，平台负责“为什么知道这个 URL、是否必须最终抓到、抓到的事实如何持久化、失败后如何恢复、以及什么时候算覆盖完成”。**
