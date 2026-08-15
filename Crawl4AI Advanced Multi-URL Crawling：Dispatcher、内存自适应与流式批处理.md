# Crawl4AI Advanced Multi-URL Crawling：Dispatcher、内存自适应与流式批处理

## 1. 调研对象与版本边界

调研对象：

- 官方文档：<https://docs.crawl4ai.com/advanced/multi-url-crawling/>
- 主项目：<https://github.com/unclecode/crawl4ai>
- 核心实现：`crawl4ai/async_dispatcher.py`
- 结果模型：`crawl4ai/models.py`
- `arun_many()`：`crawl4ai/async_webcrawler.py`
- 流式回归测试：`tests/general/test_stream_dispatch.py`
- v0.9.2 发布说明：`docs/md_v2/blog/releases/v0.9.2.md`

本次源码分析固定到 Crawl4AI 仓库提交：

```text
7e801521428ee12509994d39151006f64055ebe3
```

这是重要的版本边界。Dispatcher 属于运行时行为，文档中的默认值、字段名称和“看起来像什么”都不能替代源码与 Contract Test。例如官方文档示例中的 `max_session_permit` 默认值与当前源码构造函数并不完全一致；生产平台必须 pin Runtime Release 后测试真实行为，而不是依赖文档示例推断。

---

## 2. 这篇文档真正解决什么问题

当 URL 数量从几个扩大到数百、数千时，最直观的实现是：

```python
for url in urls:
    await crawler.arun(url)
```

这样简单，但吞吐低。另一种极端是为所有 URL 一次性创建大量异步任务，又容易造成：

- Chromium Page/Context 激增；
- 进程 RSS 上升；
- 源站被瞬时打满；
- 429/503 增多；
- 任务结果同时堆积；
- 失败、取消和资源释放变得难以控制。

Crawl4AI 的 `arun_many()` + Dispatcher 位于两者之间：

```text
URL List
   |
   v
AsyncWebCrawler.arun_many()
   |
   v
Dispatcher
   |- local queue
   |- concurrency gate
   |- memory pressure gate
   |- optional per-domain delay/backoff
   |- optional monitor
   |
   v
AsyncWebCrawler.arun(url)
   |
   v
CrawlerTaskResult / CrawlResult
```

它的核心价值是 **单个 Runtime 进程内的资源调度与保护**。

对“1000 个技术博客全历史回填 + 长期增量同步”这个目标，最关键的结论不是“用 `arun_many()` 就能大规模抓取”，而是：

> Dispatcher 只能解决 Runtime 内部怎样安全地执行一批 URL；它不能解决整个平台应该抓哪些 URL、哪个 Source 先抓、任务是否持久、Worker 崩溃后如何恢复、跨 Pod 如何全局限流、历史 Coverage 是否完整等问题。

所以正确架构是：

```text
Platform Durable Scheduler
   -> bounded micro-batch
   -> Crawl4AI Runtime Dispatcher
   -> streaming materialization
   -> Platform Truth Store
```

而不是：

```text
百万 URL
   -> 一个 arun_many()
   -> 希望 Dispatcher 负责全部调度语义
```

---

## 3. Dispatcher 抽象：运行时局部调度器

源码中 `BaseDispatcher` 主要定义：

```text
select_config(url, configs)
crawl_url(...)
run_urls(...)
```

当前主要实现有：

1. `MemoryAdaptiveDispatcher`
2. `SemaphoreDispatcher`

两者都属于 **in-process runtime scheduler**。

### 3.1 为什么这一边界重要

Dispatcher 使用的是 Python 进程内对象：

- `asyncio.Queue` / `asyncio.PriorityQueue`
- 普通 Dict 保存 domain state
- 普通变量保存 pressure mode
- Python Task 保存 active crawl
- 当前进程的内存观测

因此 Runtime 重启时，这些状态天然消失。

它们不具备：

- durable task；
- 跨实例一致性；
- 跨 Worker fencing；
- 全局公平性；
- 全局 Source budget；
- 可审计的长期 retry history；
- Coverage truth。

所以 Dispatcher 必须被平台视为“执行器内部状态”，而不是数据库状态机的替代品。

---

## 4. RateLimiter：文档描述与源码语义必须分开

官方文档把 `RateLimiter` 描述为用于请求节奏控制，并列出：

```text
base_delay
max_delay
max_retries
rate_limit_codes
```

默认关注 429、503。

从使用者视角很容易把 `max_retries` 理解为：

```text
收到 429
 -> 自动 sleep
 -> 自动重新请求同一个 URL
 -> 最多 retry N 次
```

但当前源码的核心实现并不是一个完整的 durable retry loop。

### 4.1 DomainState

`models.py` 中的 `DomainState` 只有：

```python
@dataclass
class DomainState:
    last_request_time: float = 0
    current_delay: float = 0
    fail_count: int = 0
```

也就是说它是非常轻量的、进程内的域状态。

### 4.2 wait_if_needed()

逻辑近似：

```text
取 url.netloc
 -> 找 domain state
 -> 如果上次请求距离现在小于 current_delay
      sleep 剩余时间
 -> 如果还没有 delay
      从 base_delay 随机初始化
 -> 更新 last_request_time
```

这里的作用是 **在本地请求之前延迟**。

### 4.3 update_delay()

收到响应后：

```text
if status in [429, 503]:
    fail_count += 1
    if fail_count > max_retries:
        return False
    current_delay = min(
        current_delay * 2 * jitter,
        max_delay
    )
else:
    current_delay = max(
        random(base_delay),
        current_delay * 0.75
    )
    fail_count = 0
```

重要区别：`update_delay()` 自身只更新 domain delay/fail-count，并返回是否超过阈值。

当前 `crawl_url()` 在拿到结果后调用 `update_delay()`；如果返回 False，会设置错误信息。但这里没有形成一个“重新执行同一 `crawler.arun()`”的显式循环。

因此在本版本中，应把它理解为：

```text
local pacing + local backoff state + local fail threshold
```

而不能把字段名 `max_retries` 直接当成平台的任务重试次数。

### 4.4 对生产平台的启示

1000 个博客可能同时由多个 Worker/Pod 抓取。假设：

```text
Pod A domain delay = 8s
Pod B domain delay = 1s
Pod C 刚重启，domain state 为空
```

如果只依赖 Runtime RateLimiter，Pod B/C 仍可能继续向源站施压。

因此平台必须有跨 Worker 的：

```text
per_host_qps
per_registrable_domain_concurrency
Retry-After
origin_backoff_until
source concurrency
```

推荐：

```text
Redis Lua Token Bucket / Semaphore
 + PostgreSQL/Redis shared origin backoff
 + Runtime RateLimiter local safety net
```

Runtime RateLimiter 是第二道保险，不是全局礼貌性策略的唯一事实层。

---

## 5. MemoryAdaptiveDispatcher：三阈值形成内存滞回

当前源码构造函数包括：

```text
memory_threshold_percent
critical_threshold_percent
recovery_threshold_percent
check_interval
max_session_permit
fairness_timeout
memory_wait_timeout
rate_limiter
monitor
```

同时建立：

```text
result_queue = asyncio.Queue()
task_queue = asyncio.PriorityQueue()
memory_pressure_mode
current_memory_percent
_high_memory_start_time
```

### 5.1 为什么需要 pressure/recovery 两个阈值

如果只用一个 90% 阈值：

```text
89.9 -> 开任务
90.1 -> 停
89.9 -> 开
90.1 -> 停
```

系统会频繁抖动。

当前实现使用类似：

```text
>= pressure threshold
    -> PRESSURE

<= recovery threshold
    -> NORMAL
```

即 **hysteresis（滞回）**。

例如：

```text
pressure = 90%
recovery = 85%
```

一旦进入压力态，就需要真正回落到 85% 以下才恢复放任务，不是在 89.9% 时马上重新加压。

这个思想适合 Browser Runtime，因为 Chromium 的 renderer/page 回收并不一定瞬时释放 RSS。

### 5.2 critical threshold

当前源码还区分 `critical_threshold_percent`。

在 `crawl_url()` 开始实际抓取前，如果当前内存已经达到 critical threshold，会把 URL 重新放回本地 `PriorityQueue`，而不是继续创建高成本抓取。

因此存在：

```text
PRESSURE
  -> 停止继续从 queue 放出新任务

CRITICAL
  -> 某个已进入 crawl_url 的任务也可能本地 requeue
```

这是一个很重要的细节，因为它会影响 **结果 cardinality 和任务状态解释**。

### 5.3 memory_wait_timeout

如果内存压力持续过久，后台 memory monitor 可以抛出 `MemoryError`。

这个 timeout 是单 Runtime 的 fail-fast 保护：

```text
持续高内存
 -> 不无限卡死 batch
 -> 让上层看到 Runtime 失败
```

平台不能把它当普通文章失败。更合理的错误分类是：

```text
RUNTIME_MEMORY_PRESSURE_TIMEOUT
```

随后由平台 Scheduler：

- 降低 batch；
- 切其他 Runtime；
- 延迟重试；
- drain/replace 异常实例。

---

## 6. PriorityQueue 与公平性：它只保证当前 batch 内的局部公平

`MemoryAdaptiveDispatcher` 使用 `asyncio.PriorityQueue`。

初始 URL 进入队列时大致是：

```text
priority = 0
retry_count = 0
enqueue_time = now
```

源码通过 `_get_priority_score(wait_time, retry_count)` 调整：

```text
等待超过 fairness_timeout
 -> priority = -wait_time
 -> 等越久，数值越小，优先级越高

未超过 fairness_timeout
 -> priority = retry_count
```

目的很清楚：避免某些 URL 长期饥饿。

### 6.1 但它不是跨 Source 公平

如果平台把：

```text
Source A: 100000 URL
Source B: 100 URL
```

全部塞进同一个 Runtime 本地队列，那么 Dispatcher 看到的只是 URL，不理解平台业务中的：

```text
INCREMENTAL
NEW_SOURCE_BACKFILL
HISTORICAL_BACKFILL
REPAIR
RECONCILE
Tenant
Source Daily Budget
```

即使本地 URL 最终都有机会执行，也可能让大站占满 Runtime 很久。

因此全局公平仍应是：

```text
Platform WFQ / DRR
 -> tenant/source/domain 分层
 -> 每次只租约 bounded micro-batch
 -> Runtime local fairness
```

### 6.2 为什么不能把百万 Frontier 放进本地 PriorityQueue

`_update_queue_priorities()` 会周期性：

```text
把 queue 中元素取出
 -> 重新计算 wait priority
 -> 临时列表排序
 -> 再放回 queue
```

这是适合当前 batch 的局部操作，却不是百万级 persistent frontier 数据结构。

全局 Frontier 应保存在 PostgreSQL，并由 Scheduler 查询/租约；Runtime queue 只承接已经 Admission 的有限任务。

---

## 7. `select_config()`：first-match 是隐藏但重要的配置契约

`BaseDispatcher.select_config()` 的语义：

```text
如果 configs 是单个 CrawlerRunConfig
 -> 直接使用

如果 configs 是空
 -> None

如果 configs 是列表
 -> 从前往后找第一个 config.is_match(url)
 -> 找到就立刻返回

全部不匹配
 -> None
```

### 7.1 first-match 的风险

假设配置：

```text
1. *.example.com/*        -> generic
2. blog.example.com/*     -> blog-specific
```

如果 generic 在前，blog-specific 永远不会命中。

这就是典型 shadow rule。

1000 个网站如果允许运营人员随意排列规则，很容易出现：

- 配置重叠；
- 特定规则被通配规则遮蔽；
- 新发布配置改变旧 URL route；
- 没有任何 match 却被误认为源站失败。

### 7.2 no-match 的真实结果

当前 `crawl_url()` 对 `selected_config is None` 会返回失败的 `CrawlerTaskResult`，其中结果 metadata 有类似：

```text
status = no_config_match
```

这不是：

```text
HTTP 404
origin unavailable
article missing
```

而是 **平台/Runtime 配置没有覆盖这个 URL**。

生产中应归类：

```text
NO_CONFIG_MATCH
```

并进入：

```text
CONFIG_ERROR / ROUTE_REPAIR / INTENTIONAL_SKIP
```

而不是消耗源站 retry budget。

### 7.3 方案要求

Config Compiler 应增加：

- matcher overlap 检测；
- shadow 检测；
- explicit priority/order；
- explicit default；
- representative URL fixture；
- 发布前 match coverage test。

每个 Attempt 保存：

```text
crawler_run_config_release_id
matcher_rule_id
matcher_evidence
```

使 Web 管理端可以回答：

> 为什么这个 URL 使用了 JS-heavy profile，而另一个同站 URL 使用 HTTP profile？

---

## 8. `crawl_url()`：一次 Runtime 执行的真实流程

`MemoryAdaptiveDispatcher.crawl_url()` 的简化流程：

```text
start timing
 -> select config
 -> monitor = IN_PROGRESS
 -> concurrent_sessions += 1
 -> RateLimiter.wait_if_needed(url)
 -> critical memory?
      yes -> local requeue + placeholder result
      no  -> crawler.arun(url, config, session_id=task_id)
 -> measure process RSS delta
 -> RateLimiter.update_delay(status)
 -> monitor final status
 -> concurrent_sessions -= 1
 -> CrawlerTaskResult
```

这里有四个对生产系统非常关键的细节。

### 8.1 `session_id=task_id`

Dispatcher 把 task identity 传入 crawler session。

平台 Adapter 不应因此直接复用 Runtime UUID 当业务 Task ID；业务身份仍应该是自己的：

```text
platform_task_id
attempt_id
runtime_task_id
input_correlation_key
```

第三方 Runtime identity 只是 lineage 的一部分。

### 8.2 local requeue 会先产生 placeholder

达到 critical memory 时：

```text
URL 再次 put 回 task_queue
retry_count + 1
```

同时函数会返回一个 `CrawlerTaskResult`，错误文字包含 requeued 语义。

这意味着：

```text
一次 input
可能经历：
placeholder requeue result
+ 后续真正 final result
```

如果 Adapter 把第一个结果当终态，就会把仍在 Runtime 队列里的任务误判失败。

### 8.3 本地 retry_count 不是平台 retry_count

这里的 retry_count 是 Dispatcher 本地 requeue 计数的一部分。

平台的：

```text
attempt_no
retry_count
fencing_token
next_attempt_at
```

必须独立维护。

否则会出现：

```text
Runtime 因内存 requeue 3 次
 -> 平台以为源站失败 3 次
 -> 错误触发 DLQ
```

实际上源站甚至可能一次都还没有被真正请求。

### 8.4 memory_usage 不是精确单任务内存

源码抓取前后读取：

```text
当前 Python 进程 RSS
```

然后做差。

但是多个 URL 是并发的：

```text
Task A start RSS = 1.0GB
Task B 启动后增加 200MB
Task C 结束释放 150MB
Task A end RSS = 1.05GB
```

A 的真实内存贡献不等于 50MB。

所以 `CrawlerTaskResult.memory_usage/peak_memory` 在并发环境应视为 **诊断性近似值**，不能：

- 相加作为 Runtime 总内存；
- 用于精确按 Source 计费；
- 用作唯一 Autoscaling 指标。

容量真相更应依赖：

```text
cgroup memory.current
memory.events / OOM
PSI
process RSS
Browser process RSS
page/context count
queue age
slot utilization
```

---

## 9. `run_urls()`：Batch 模式的 cardinality 陷阱

Batch 模式会：

1. 为每个 URL 创建 task_id；
2. 全部放入本地 PriorityQueue；
3. 在非 pressure mode 时尽量填满 `max_session_permit`；
4. `asyncio.wait(... FIRST_COMPLETED)`；
5. 将完成的 `CrawlerTaskResult` append 到 `results`；
6. 继续更新 queued item 的优先级。

### 9.1 关键问题

对 critical-memory local requeue，`crawl_url()` 会：

```text
requeue URL
+ return placeholder CrawlerTaskResult
```

而 `run_urls()` 对完成 task 的逻辑是直接：

```text
results.append(result)
```

之后同一个 URL 还会从 queue 再出来执行，最终再产生结果。

所以至少在当前源码逻辑上，Batch consumer **不能建立以下假设**：

```python
assert len(results) == len(urls)
for input_url, result in zip(urls, results):
    ...
```

这种 positional correlation 在压力态可能出错。

### 9.2 平台正确做法

每个输入先建立：

```text
runtime_task_item
- runtime_task_id
- attempt_id
- input_url_identity_id
- expected_correlation_key
```

Adapter 归一化结果：

```text
FINAL_SUCCESS
FINAL_FAILURE
LOCAL_REQUEUED
NO_CONFIG_MATCH
```

其中：

```text
LOCAL_REQUEUED != terminal
```

Materializer 只有看到 terminal outcome 才闭合 item。

### 9.3 为什么 Browser Repair 建议小批量

如果一次 Runtime Job 只有 10～50 个 URL：

- cardinality reconciliation 简单；
- Runtime crash 影响面有限；
- 结果可边到边落盘；
- lease deadline 容易控制；
- domain mix 可由平台调节；
- 内存阈值更有意义。

如果一次塞 100000 URL：

- 本地队列成为第二套 Frontier；
- Runtime 重启时大量局部状态丢失；
- 结果 reconcile 成本上升；
- 大站绕过全局公平；
- drain/upgrade 很困难。

因此生产方案采用 bounded micro-batch。

---

## 10. Streaming 模式：对知识库更合适，但必须正确处理取消

官方文档的 streaming 模式让结果完成一个就 yield 一个。

对于知识库采集，这是非常有价值的：

```text
URL 1 完成
 -> 立即写 Raw Artifact
 -> 立即持久化 Attempt item
 -> 立即释放大结果引用

而不是：
等待整个 batch 完成
 -> 所有 CrawlResult 一起留到最后
```

### 10.1 stream 与 batch 对 local requeue 的行为不同

当前 `run_urls_stream()` 中，对完成结果有额外判断：

```text
如果 error_message 包含 requeued
 -> 不增加 completed_count
 -> 不 yield 给 consumer
```

所以 local requeue placeholder 被 Runtime 内部屏蔽，真正 terminal 后才推进完成数。

而 Batch `run_urls()` 没有同样的过滤路径。

这正说明：

> “同一个 Dispatcher”不代表 batch 与 stream 的结果契约完全一致。

平台 Adapter 必须测试两种模式，不能共用未经验证的 cardinality 假设。

---

## 11. v0.9.2 的 stream-close 修复：取消协议是生产级关键语义

Crawl4AI v0.9.2 发布说明记录了一个非常重要的资源泄漏修复。

修复前：

```text
consumer 提前关闭 / cancel async generator
 -> run_urls_stream() 退出
 -> 某些 per-URL crawl task 仍在后台运行
 -> Browser Page/Context 仍被占用
 -> 后续同一个 crawler 再执行 arun_many()
 -> 新旧任务重叠
 -> 页面泄漏 / TargetClosedError / 杂散任务
```

修复后，stream `finally` 明确：

```text
遍历 active_tasks
 -> cancel 未完成 task
 -> gather(... return_exceptions=True)
 -> 清空 queued-but-unstarted URL
 -> cancel + await memory monitor
 -> stop monitor
```

发布说明还明确提到回归测试覆盖：

- in-flight task 被取消；
- queue 被 drain；
- session count 回零。

### 11.1 对平台的核心启示

“stream 已关闭”只说明：

```text
Runtime 的这一批本地执行已经被终止并清理
```

它不说明：

```text
所有平台 Task 都已经 terminal
```

因此平台状态应该是：

```text
RUNNING_STREAM
 -> RUNTIME_STREAM_ABORTED
 -> Runtime cleanup verified
 -> inspect materialized terminal items
 -> unmaterialized items reconciliation
 -> durable requeue with fencing
```

### 11.2 已完成和未完成必须拆开

假设 batch 有 20 个 URL：

```text
8 个已经 yield 且 Raw Artifact 已持久化
4 个正在抓
8 个还在 Runtime local queue
```

consumer 此时取消。

正确结果：

```text
前 8 个
 -> 保持 terminal，不重复抓

中间 4 个
 -> 被 cancel；如果没有 terminal materialization，回平台 reconcile

后 8 个
 -> local queue drain；回平台 reconcile
```

不能把 20 个全部重试，否则会制造重复请求；也不能把 20 个全部标成功，否则会永久丢 12 个。

### 11.3 Contract Probe 应故意制造 early close

Runtime 上线前自动测试：

1. 提交至少多于并发槽位的 URL；
2. 只消费前几个 stream result；
3. 主动 `aclose()` / cancel consumer；
4. 验证 active task 清零；
5. 验证 local queue 清空；
6. 验证浏览器 Page/Context 没有残留；
7. 立即复用同一 crawler 再跑 batch；
8. 验证不会出现旧任务继续输出或 `TargetClosedError`；
9. 验证平台只重发没有 terminal materialization 的 item。

这是比“import 成功”和“抓 example.com 成功”更有价值的生产 Contract Test。

---

## 12. SemaphoreDispatcher：简单固定并发的适用位置

`SemaphoreDispatcher` 的思想更简单：

```text
固定 semaphore
 -> async with semaphore
 -> crawler.arun()
```

它同样可搭配 RateLimiter 和 Monitor。

### 12.1 什么时候更合适

如果 Runtime：

- 资源足够稳定；
- URL 成本相对一致；
- 已经有外部 Pod memory limit；
- 希望行为简单可预测；

固定 semaphore 反而更容易做容量规划。

例如 HTTP-like 或轻量 Browser profile：

```text
Pod memory = 8GB
经验每 active page = 250MB P95
预留系统/Browser base = 3GB
安全可用 = 5GB
并发粗估 <= 20
再乘安全系数
```

可以固定较低并发。

### 12.2 什么时候 MemoryAdaptive 更合适

页面资源成本差异很大时：

- 某些站很轻；
- 某些 React 页面很重；
- 某些页面大量图片/脚本；
- renderer 内存释放延迟；

MemoryAdaptive 的 pressure/recovery 滞回更有价值。

但无论使用哪一个，都仍然只是 Runtime local admission。

---

## 13. Monitor：运维观察，不是业务真相

CrawlerMonitor 能提供：

- task status；
- wait time；
- duration；
- memory 字段；
- queue 统计；
- aggregated/detailed display。

它适合调试和 Runtime 运营。

但知识库业务不应根据 Monitor 直接计算：

- 已抓历史文章数；
- Coverage；
- Document PASS 数；
- Source 是否同步完成。

原因是 Runtime Monitor 只看到当前执行窗口，而业务真相跨越：

```text
Discovery
Fetch
Artifact Persist
Extraction
Quality
Version
Projection
```

因此：

```text
Runtime Monitor -> Prometheus / diagnostics
PostgreSQL Truth -> Web 业务状态
```

---

## 14. 对 1000+ 技术博客方案的具体优化

本次调研后，方案增加 `runtime_dispatch_policy_release`。

### 14.1 Dispatcher 配置必须版本化

至少保存：

```text
dispatcher_type
max_session_permit
semaphore_count
pressure_threshold
critical_threshold
recovery_threshold
check_interval
memory_wait_timeout
fairness_timeout
runtime_batch_max_urls
stream policy
rate limiter parameters
config match policy
local requeue classification
stream close policy
```

原因：同一个 URL 在不同 Runtime Dispatch Policy 下，可能：

- 是否进入 Browser 的时机不同；
- 是否被本地 requeue 不同；
- batch 结果 cardinality 不同；
- shutdown 行为不同；
- 性能和失败率不同。

没有 Release 就无法解释历史运行差异。

### 14.2 双层调度

最终结构：

```text
PostgreSQL Persistent Frontier
    |
    v
Platform Fair Scheduler
    |- Incremental reserved capacity
    |- Source fairness
    |- tenant fairness
    |- domain budget
    |- global Browser budget
    |
    v
Lease + Fencing
    |
    v
bounded micro-batch
    |
    v
Crawl4AI Dispatcher
    |- local memory pressure
    |- local semaphore
    |- local RateLimiter
    |- local queue aging
    |
    v
stream results
    |
    v
Result Materializer
    |
    v
Immutable Raw Artifact
```

两层职责不重复：

```text
Platform Scheduler = durable/global/business fairness
Runtime Dispatcher  = ephemeral/local/resource safety
```

### 14.3 micro-batch 而不是 mega-batch

Runtime Gateway 根据：

```text
available browser slots
expected result bytes
memory headroom
per-domain budget
lease deadline
runtime_batch_max_urls
```

决定每批数量。

Browser Repair 可以默认：

```text
1 URL / Job
```

普通 Browser Backfill 可以：

```text
small bounded micro-batch
```

具体数字由压测而不是文档默认值决定。

### 14.4 结果必须按 identity 关联

禁止：

```python
for url, result in zip(urls, results):
    persist(url, result)
```

推荐：

```text
expected item
 -> correlation key
 -> runtime task id
 -> normalized runtime outcome
 -> materialization
 -> terminal close
```

尤其要正确吸收：

```text
LOCAL_REQUEUED
NO_CONFIG_MATCH
CANCELLED_ON_STREAM_CLOSE
```

### 14.5 Runtime local requeue 不消耗源站 retry budget

如果因为内存 critical 而 requeue：

```text
source request may not have happened
```

所以不能增加：

```text
origin_retry_count
```

平台至少区分：

```text
runtime_local_requeue_count
origin_attempt_count
platform_attempt_count
```

### 14.6 跨 Worker Rate Limit

加入共享：

```text
origin_backoff_state
- registrable_domain
- host
- reason
- status_code
- retry_after
- backoff_until
- observation_id
- updated_at
```

Runtime 本地 429/503 feedback 可以上报，平台把它传播给所有 Worker。

---

## 15. 建议的 Runtime Contract Test 矩阵

### 15.1 Memory pressure

验证：

- pressure threshold 后不继续填槽；
- recovery threshold 前不会抖动恢复；
- critical threshold 触发 local requeue；
- memory wait timeout 输出可分类异常；
- local requeue 不被 Adapter 当 terminal failure。

### 15.2 Batch cardinality

构造可触发 local requeue 的测试：

```text
N inputs
 -> collect all raw runtime results
```

验证 Adapter 不使用列表下标关联，并能得到最终：

```text
N terminal input items
```

即使 raw Runtime Result 条目数量不是 N。

### 15.3 Stream cardinality

验证：

- local requeue placeholder 不错误推进 terminal count；
- 每个真正 terminal item 只 materialize 一次；
- stream consumer 慢时有明确 backpressure 上限。

### 15.4 Stream early close

验证：

- cancel in-flight；
- await cleanup；
- drain local queue；
- active session/page 回零；
- subsequent crawler reuse 无旧任务；
- durable reconciliation 正确。

### 15.5 RateLimiter

模拟：

```text
200 -> 429 -> 429 -> 200
```

记录：

- current_delay 变化；
- fail_count 变化；
- 实际源站请求次数；
- Runtime 是否真正重发当前 URL；
- `max_retries` 的实际语义。

不能只测试“字段存在”。

### 15.6 Config matcher

准备：

- generic matcher；
- overlapping specific matcher；
- no-match URL；
- explicit default。

验证：

- first-match 顺序；
- shadow 行为；
- no-match typed outcome；
- Attempt 保存最终 config identity。

### 15.7 Runtime restart

中途 kill Runtime，验证：

- 本地 queue/state 丢失不会导致平台永久丢任务；
- lease 到期/reconciler 恢复；
- origin shared backoff 不因 Runtime 重启被清空；
- 已 durable materialized item 不重复提交。

---

## 16. Web 管理端应该怎样呈现 Dispatcher

管理端不应只显示一个模糊“抓取中”。

Runtime 页面建议展示：

```text
Runtime Release
Dispatch Policy Release
Dispatcher Type
Active Slots / Max Slots
Memory Pressure State
Process / cgroup RSS
Local Queue Depth
Local Requeue Count
No Config Match Count
Stream Abort Count
Stream Cleanup Failure
429/503 Origin Feedback
Micro-batch P50/P95 Size
Micro-batch P95 Duration
```

Attempt 详情展示：

```text
platform task id
attempt id
runtime task id
matched config release
local requeue observations
final runtime outcome
stream batch id
materialized at
raw artifact id
```

这样运营人员可以区分：

- 源站慢；
- Runtime 内存压力；
- 配置没匹配；
- Runtime queue 满；
- 平台 global admission 等待；
- stream 被取消；
- Artifact materialization 卡住。

否则所有问题都会被压成一个“爬虫失败”，难以运营 1000 个异构网站。

---

## 17. 推荐指标

新增执行面指标：

```text
runtime_dispatcher_pressure_seconds
runtime_dispatcher_critical_seconds
runtime_local_requeue_total
runtime_no_config_match_total
runtime_stream_abort_total
runtime_stream_cleanup_failure_total
runtime_result_placeholder_total
runtime_micro_batch_size
runtime_micro_batch_duration_seconds
origin_backoff_active_domains
```

同时保留平台级：

```text
queue_age_seconds
lease_recovery_total
origin_rate_limit_total
browser_slot_utilization
cgroup_memory_pressure
raw_artifact_materialization_latency
result_cardinality_mismatch_total
```

注意：Runtime 指标只能影响运维/Admission 决策，不能直接修改：

```text
Coverage
Document Version
PASS Markdown
Known Gap
```

---

## 18. 最终结论

Crawl4AI Advanced Multi-URL Crawling 的价值非常明确：它提供了比“无界 asyncio 并发”成熟得多的单 Runtime 执行控制，尤其是：

- `MemoryAdaptiveDispatcher` 的内存压力滞回；
- bounded concurrent sessions；
- PriorityQueue 与等待公平性；
- 本地域级 delay/backoff；
- Batch / Streaming 两种结果消费模式；
- CrawlerMonitor；
- Semaphore 固定并发模式；
- v0.9.2 后更完整的 stream cancellation cleanup。

但对 1000+ 博客知识库，最重要的是 **不要把它升级成它并不具备的平台语义**。

最终边界应保持：

```text
Crawl4AI Dispatcher
= Runtime local execution control

Platform Scheduler
= durable global scheduling truth

Runtime RateLimiter
= local protection

Platform Domain Admission
= cross-worker politeness truth

LOCAL_REQUEUED
= runtime transient event

Platform Retry
= durable task state transition

Stream close
= cancellation + reconciliation event

Canonical Raw/IR/Markdown
= platform durable data plane
```

因此，本次调研对现有博客知识库方案的主要优化不是增加另一套爬虫组件，而是把 **Dispatcher 的局部调度语义、结果 cardinality、stream 生命周期和平台 durable scheduler 之间的契约** 写清楚。这样既可以利用 Crawl4AI 的高效并发执行，又不会把 1000 站点长期同步的可靠性错误地绑定在某个进程内队列或某个版本的 Runtime 细节上。
