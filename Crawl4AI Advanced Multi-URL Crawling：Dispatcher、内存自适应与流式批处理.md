# Crawl4AI Advanced Multi-URL Crawling：Dispatcher、内存自适应与流式批处理

## 1. 调研对象与版本边界

调研对象：

- 官方文档：<https://docs.crawl4ai.com/advanced/multi-url-crawling/>
- 主项目：<https://github.com/unclecode/crawl4ai>
- 核心实现：`crawl4ai/async_dispatcher.py`
- `arun_many()`：`crawl4ai/async_webcrawler.py`
- 结果模型：`crawl4ai/models.py`
- Crawler Monitor：`crawl4ai/components/crawler_monitor.py`

本次实现分析固定到 Crawl4AI 仓库提交：

```text
7e801521428ee12509994d39151006f64055ebe3
```

固定提交很重要。Dispatcher 属于运行时行为，文档中的字段名、示例默认值和“听起来应该做什么”都不能替代源码与 Contract Test。当前官方文档把 `MemoryAdaptiveDispatcher.max_session_permit` 的默认值写成 10，而当前源码构造函数默认值是 20；这种差异足以说明生产平台必须 pin Runtime Release。

---

## 2. `arun_many()` 真正解决的问题

单 URL 串行：

```python
for url in urls:
    await crawler.arun(url)
```

吞吐低；一次性无界 `asyncio.gather()` 又容易导致：

- Browser Page/Context 激增；
- Python Task/Result 对象激增；
- Chromium 与 Python RSS 上升；
- 源站瞬时请求峰值；
- 429/503；
- 结果长期驻留内存；
- 取消后残留任务难清理。

Crawl4AI 的 `arun_many()` + Dispatcher 位于中间：

```text
URL List
   |
   v
AsyncWebCrawler.arun_many()
   |
   v
Dispatcher
   |- local queue / tasks
   |- concurrency gate
   |- optional memory gate
   |- optional local rate limiter
   |- optional monitor
   |
   v
AsyncWebCrawler.arun(url)
   |
   v
CrawlerTaskResult / CrawlResult
```

它的核心职责是：**单 Runtime 进程内的局部执行控制**。

它不拥有：

- durable task；
- 跨实例 lease/fencing；
- 全局 Source 公平；
- 跨 Worker domain QPS；
- 历史 Coverage；
- Document Identity；
- Canonical Markdown。

因此对 1000+ 技术博客，正确结构是：

```text
Platform Persistent Frontier
  -> Fair Scheduler
  -> Global Admission
  -> bounded micro-batch
  -> Crawl4AI Dispatcher
  -> streaming materialization
  -> Platform Truth Store
```

而不是把百万 URL 直接塞给一个 `arun_many()`。

---

## 3. BaseDispatcher：局部执行抽象

`BaseDispatcher` 主要定义：

```text
select_config(url, configs)
crawl_url(...)
run_urls(...)
```

当前主要实现：

1. `MemoryAdaptiveDispatcher`
2. `SemaphoreDispatcher`

Dispatcher 的状态大量存在于 Python 进程内：

- `asyncio.PriorityQueue`；
- `asyncio.Semaphore`；
- Python Task；
- 普通 Dict；
- pressure mode；
- domain delay state；
- 当前进程内存观测。

Runtime 一旦重启，这些状态天然消失。所以它们必须被平台视为 **ephemeral execution state**。

---

## 4. `select_config()`：first-match 是真实路由契约

当 `configs` 是单个 `CrawlerRunConfig` 时直接使用；当是列表时，从前到后调用 `config.is_match(url)`，第一个命中就返回；全部不匹配则返回 `None`。

因此：

```text
1. *.example.com/*        -> generic
2. blog.example.com/*     -> blog-specific
```

如果 generic 在前，specific 可能永久被 shadow。

未命中不是源站 404，而是配置没有覆盖 URL。当前 `crawl_url()` 会返回失败结果，并携带类似：

```text
metadata.status = no_config_match
```

平台应归一化：

```text
NO_CONFIG_MATCH
```

并进入配置修复，而不是消耗源站 retry budget。

生产 Config Compiler 必须做：

- overlap 检测；
- shadow 检测；
- explicit order；
- explicit fallback；
- representative URL fixtures；
- release 前 match coverage test；
- Attempt 保存最终 matcher/config identity。

---

## 5. RateLimiter：它是本地 pacing state，不是 durable retry

构造参数：

```text
base_delay
max_delay
max_retries
rate_limit_codes
```

默认 rate-limit 状态码是 429、503。

### 5.1 DomainState

当前状态非常轻：

```python
@dataclass
class DomainState:
    last_request_time: float = 0
    current_delay: float = 0
    fail_count: int = 0
```

保存于进程内 Dict。

### 5.2 `wait_if_needed()`

核心逻辑近似：

```text
get url.netloc
 -> get/create DomainState
 -> now - last_request_time < current_delay ?
      sleep remaining
 -> if current_delay == 0:
      random(base_delay)
 -> last_request_time = now
```

它的作用是实例内请求前 pacing。

### 5.3 `update_delay()`

收到 429/503：

```text
fail_count += 1
if fail_count > max_retries:
    return False
current_delay = min(current_delay * 2 * jitter, max_delay)
```

正常状态码：

```text
current_delay = max(random(base_delay), current_delay * 0.75)
fail_count = 0
```

关键点：`update_delay()` 只更新状态并返回是否超过 fail threshold。当前 `crawl_url()` 没有围绕 `crawler.arun()` 建立“对同一个 URL 自动重发 N 次”的显式 retry loop。

所以当前版本中的 `max_retries` 不能直接等价为平台 retry 次数。

### 5.4 同域并发的隐藏 burst 风险

源码没有为一个 domain 建立跨 sleep 周期的 async lock、reservation 或 token lease。

例如多个同域任务先后进入：

```text
Task A 更新 last_request_time = T0
Task B 看到 T0 -> sleep D
Task C 也看到 T0 -> sleep 约 D
```

B/C 都基于近似同一个 `last_request_time` 睡眠。推导上它们可能在相近时间同时醒来，然后各自更新 `last_request_time` 并继续请求。

这意味着 Runtime RateLimiter 更接近：

```text
local delay hint + local backoff state
```

而不是严格保证“同域请求间隔至少 D 秒”的 token bucket/serialized reservation。

对 1000 个博客，生产必须另有跨 Worker 的：

```text
per_host_qps
per_host_concurrency
per_registrable_domain_concurrency
Retry-After
origin_backoff_until
```

推荐：

```text
Redis Lua Token Bucket / Semaphore
  + shared origin_backoff_state
  + Runtime RateLimiter as local safety net
```

---

## 6. MemoryAdaptiveDispatcher：三阈值形成滞回状态机

构造参数包括：

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

内部重要状态：

```text
task_queue = asyncio.PriorityQueue()
memory_pressure_mode
current_memory_percent
_high_memory_start_time
```

### 6.1 PRESSURE / RECOVERY 滞回

如果只用一个阈值：

```text
89.9 -> 放任务
90.1 -> 停
89.9 -> 放
90.1 -> 停
```

会产生抖动。

当前实现：

```text
memory >= pressure
  -> memory_pressure_mode = True

memory <= recovery
  -> memory_pressure_mode = False
```

一旦进入压力态，需要真正回落到 recovery threshold 才恢复。这是典型 hysteresis。

### 6.2 CRITICAL

进入 `crawl_url()` 后，如果内存已达到 critical threshold，会：

```text
requeue URL into local PriorityQueue
retry_count + 1
return placeholder CrawlerTaskResult
```

所以 local requeue 是 Runtime 内部资源保护事件，不意味着源站已经失败，甚至不一定已经真正发出请求。

平台必须区分：

```text
runtime_local_requeue_count
origin_attempt_count
platform_attempt_count
```

### 6.3 memory_wait_timeout

后台 memory monitor 若长期处于高压，可抛 `MemoryError`。

平台错误分类应为：

```text
RUNTIME_MEMORY_PRESSURE_TIMEOUT
```

然后：

- 降低 batch；
- 切换其他 Runtime；
- drain/replace 异常实例；
- durable retry 未完成 item。

不能把它记录成“文章抓取失败”。

---

## 7. PriorityQueue 公平性：只覆盖当前 batch

`MemoryAdaptiveDispatcher` 初始把每个 URL 放入本地 `PriorityQueue`：

```text
priority = 0
retry_count = 0
enqueue_time = now
```

`_get_priority_score()` 大致：

```text
wait_time > fairness_timeout
  -> priority = -wait_time
else
  -> priority = retry_count
```

等待很久的 item 会被提高优先级，避免当前 batch 内长期饥饿。

但这完全不理解平台语义：

```text
INCREMENTAL
NEW_SOURCE_BACKFILL
HISTORICAL_BACKFILL
REPAIR
RECONCILE
Tenant
Source Budget
```

另外 `_update_queue_priorities()` 会把本地 queue drain 到临时列表、重新计算优先级、排序、再放回。它适合有限 batch，不适合百万级 Persistent Frontier。

所以：

```text
Platform DRR/WFQ
 -> lease bounded micro-batch
 -> Runtime local PriorityQueue
```

---

## 8. `crawl_url()` 的执行链

MemoryAdaptive 路径近似：

```text
select config
 -> monitor IN_PROGRESS
 -> concurrent_sessions += 1
 -> RateLimiter.wait_if_needed
 -> critical memory ?
      yes -> local requeue + placeholder
      no  -> crawler.arun(url, config, session_id=task_id)
 -> process RSS delta
 -> RateLimiter.update_delay(status)
 -> monitor result
 -> concurrent_sessions -= 1
 -> CrawlerTaskResult
```

### 8.1 Runtime identity 不等于平台 identity

源码用 `session_id=task_id` 调用 crawler。平台仍应维护：

```text
platform_task_id
attempt_id
runtime_task_id
correlation_key
```

Runtime UUID 只是 lineage 的一部分。

### 8.2 单任务 memory_usage 只是诊断值

源码在任务前后读当前 Python 进程 RSS 做差。

并发情况下其他 Task/Browser 的分配与释放会叠加，所以它不是精确单任务内存归因，不能：

- 相加当 Runtime 总内存；
- 用于精确按 Source 计费；
- 作为唯一 autoscaling 指标。

生产容量观察更应使用：

```text
cgroup memory.current
memory.events / OOM
PSI
process RSS
browser process RSS
page/context count
queue age
slot utilization
```

---

## 9. Batch 模式的 cardinality 陷阱

MemoryAdaptive `run_urls()`：

1. 所有 URL 先进入本地 PriorityQueue；
2. 非 pressure mode 时按 `max_session_permit` 填 active task；
3. `asyncio.wait(... FIRST_COMPLETED)`；
4. 完成的 `CrawlerTaskResult` 直接 append 到 results。

critical memory local requeue 时，`crawl_url()` 会：

```text
requeue same URL
+ return placeholder result
```

而 batch 路径会 append 这个 placeholder；以后 URL 真正执行完成时还会再产生结果。

所以至少在当前实现契约下，不能假设：

```python
len(results) == len(urls)
```

也不能：

```python
for url, result in zip(urls, results):
    ...
```

正确方法是每个输入都有 correlation identity，并把原始结果归一化为：

```text
FINAL_SUCCESS
FINAL_FAILURE
LOCAL_REQUEUED
NO_CONFIG_MATCH
```

其中 `LOCAL_REQUEUED` 非 terminal。

---

## 10. Streaming 模式更适合知识库物化

`run_urls_stream()` 完成一个结果就 yield 一个，适合：

```text
URL 完成
 -> 立即晋升 Raw Artifact
 -> 立即更新 Attempt item
 -> 立即释放大结果引用
```

而不是等整个 batch 完成后一次性处理所有 `CrawlResult`。

对 local requeue，当前 stream 路径有额外过滤：错误信息包含 requeue 语义时，不推进 `completed_count`，也不 yield placeholder；真正 terminal 后才完成。

因此同一个 Dispatcher 的 batch 与 stream 的 result contract 并不完全一致。

平台应优先使用 streaming micro-batch，并且每个 Runtime Release 都单独测试 batch/stream。

---

## 11. Stream early-close：取消也是协议

当前 `run_urls_stream()` 的 `finally` 会：

```text
cancel unfinished active_tasks
await gather(..., return_exceptions=True)
drain queued-but-unstarted items
cancel + await memory monitor
stop monitor
```

这个语义很关键，因为 consumer 可能因为：

- Worker shutdown；
- 用户取消；
- Materializer 故障；
- lease 即将超时；

提前关闭 async generator。

假设 20 个 URL：

```text
8 已 yield + durable materialized
4 in-flight
8 queued locally
```

此时 cancel：

- 前 8 个保持成功，不应全部重抓；
- 中间 4 个被 cancel，若没有 terminal materialization，则回平台 reconcile；
- 后 8 个被 local queue drain，同样回 durable scheduler。

所以“stream 已关闭”不是“所有平台 Task 都 terminal”。

---

## 12. SemaphoreDispatcher：固定并发下仍有 O(N) Task 风险

当前构造函数：

```text
semaphore_count = 5
max_session_permit = 20
rate_limiter
monitor
```

### 12.1 真实并发门

`run_urls()` 创建：

```python
semaphore = asyncio.Semaphore(self.semaphore_count)
```

每个 URL 的 `crawl_url()` 在：

```python
async with semaphore:
    await self.crawler.arun(...)
```

中进入真正抓取。

所以当前执行路径实际控制抓取并发的是：

```text
semaphore_count
```

### 12.2 `max_session_permit` 当前没有参与 Semaphore run path

源码保存：

```python
self.max_session_permit = max_session_permit
```

但当前 `run_urls()` 没有拿它再做一层 gate。

这意味着生产控制面不能因为字段存在，就宣称：

> SemaphoreDispatcher 的有效并发上限是 max_session_permit。

在当前 pinned release，应以实测 `semaphore_count` 为有效抓取并发旋钮；升级后重新 Contract Test。

### 12.3 更重要：所有 URL 先 create_task

`run_urls()` 对每个 URL 都直接：

```python
task = asyncio.create_task(
    self.crawl_url(url, config, task_id, semaphore)
)
tasks.append(task)
```

最后：

```python
await asyncio.gather(*tasks, return_exceptions=True)
```

所以 Semaphore 只是把真正进入 `crawler.arun()` 的并发限制住，并没有把 Python Task 数量限制为 `semaphore_count`。

如果一次传入 100000 URL：

```text
100000 asyncio Task/Future
+ 100000 task metadata
+ gather bookkeeping
```

仍然会先创建。

这是非常重要的工程边界：

> bounded crawl concurrency != bounded scheduler object count

因此 SemaphoreDispatcher 也必须只接 bounded micro-batch。

生产指标应增加：

```text
runtime_scheduled_async_tasks
runtime_semaphore_waiters
```

否则可能看到“Browser 并发只有 5”，但事件循环已经积压几万 Task。

---

## 13. MemoryAdaptive 与 Semaphore 的选择

### MemoryAdaptive 更适合

- 页面资源成本差异大；
- JS-heavy；
- Chromium RSS 波动明显；
- 希望在单实例内基于内存自动暂停发新任务。

### Semaphore 更适合

- 页面成本相对稳定；
- Runtime 内存限额已经比较明确；
- 更关注简单可预测的固定并发；
- 输入始终是严格 bounded micro-batch。

无论选哪种，都不能替代：

```text
Persistent Frontier
Global Fair Scheduler
Global Domain Admission
Lease / Fencing
```

---

## 14. CrawlerMonitor：诊断观察，不是业务状态机

Monitor 能展示：

- task status；
- wait time；
- duration；
- memory 字段；
- queue statistics；
- aggregated / detailed view。

它适合 Runtime 运维，却不能直接计算：

- 历史文章数；
- Coverage；
- PASS Markdown 数；
- Source 是否同步完成。

正确边界：

```text
Runtime Monitor -> Prometheus / diagnostics
PostgreSQL Truth -> Web business state
```

---

## 15. 1000+ 博客系统中的双层调度

最终结构：

```text
PostgreSQL Persistent Frontier
     |
     v
Platform Fair Scheduler
     |- Incremental reserved capacity
     |- Source/Tenant fairness
     |- domain budget
     |- global browser budget
     |
     v
Lease + Fencing
     |
     v
Global Admission
     |
     v
bounded micro-batch
     |
     v
Crawl4AI Dispatcher
     |- local memory pressure or semaphore
     |- local RateLimiter
     |- local queue/task state
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

两层职责：

```text
Platform Scheduler
= durable + global + business fairness

Runtime Dispatcher
= ephemeral + local + resource safety
```

---

## 16. 建议的 Runtime Dispatch Policy

```text
runtime_dispatch_policy_release
- dispatcher_type
- max_session_permit
- semaphore_count
- pressure_threshold
- critical_threshold
- recovery_threshold
- check_interval
- memory_wait_timeout
- fairness_timeout
- runtime_batch_max_urls
- stream_mode
- rate_limiter_profile
- config_match_policy
- local_requeue_classification
- stream_close_policy
- result_contract_version
```

字段存在并不等于字段生效。Release 必须同时保存 Contract Probe 结果，例如：

```text
observed_effective_concurrency
observed_batch_cardinality_behavior
observed_stream_requeue_behavior
observed_rate_limiter_retry_behavior
observed_stream_cleanup_behavior
observed_async_task_precreation_behavior
```

---

## 17. 建议的 Adapter Outcome

平台不要直接使用 Runtime error string 作为 durable state。Adapter 在 pinned release 下解析后输出：

```text
FINAL_SUCCESS
FINAL_FAILURE
LOCAL_REQUEUED
NO_CONFIG_MATCH
CANCELLED_ON_STREAM_CLOSE
RUNTIME_MEMORY_PRESSURE_TIMEOUT
RUNTIME_SCHEMA_MISMATCH
RUNTIME_RESULT_MISSING
```

每个 item 只有看到 terminal outcome + required artifact materialized 才闭合平台 Task。

---

## 18. 建议的 Contract Test 矩阵

### 18.1 MemoryAdaptive

验证：

- pressure 后不继续填槽；
- recovery 前不恢复；
- critical 触发 local requeue；
- memory timeout 可分类；
- local requeue 不消耗 origin retry。

### 18.2 Batch cardinality

构造 critical requeue，验证：

- 原始 result 数量可与 input 数量不同；
- Adapter 不按位置关联；
- 最终仍能闭合 N 个 platform item。

### 18.3 Streaming

验证：

- requeue placeholder 不推进 terminal；
- completed item 只物化一次；
- slow consumer 有明确 backpressure；
- early close 后 active task/page/context 清零；
- subsequent reuse 无残留任务。

### 18.4 RateLimiter

模拟：

```text
200 -> 429 -> 429 -> 200
```

记录：

- current_delay；
- fail_count；
- 实际源站请求次数；
- 是否真的自动重发；
- 多个同域 sleeper 是否出现 clustered wake-up。

### 18.5 Config matcher

验证：

- first-match；
- overlapping matcher；
- shadow；
- no-match；
- explicit fallback；
- Attempt 保存最终 config identity。

### 18.6 Semaphore

必须验证：

- `semaphore_count=K` 时真正 `crawler.arun()` 并发是否为 K；
- 改 `max_session_permit` 是否对当前版本实际并发有影响；
- N 个输入会创建多少 asyncio Task；
- N 很大时事件循环、RSS、gather bookkeeping 的增长；
- cancellation 后等待 semaphore 的 Task 是否全部收敛。

### 18.7 Runtime restart

中途 kill Runtime：

- 本地 queue/domain state 丢失不导致永久丢 Task；
- lease/reconciler 恢复；
- shared origin backoff 不清零；
- 已 durable materialized item 不重复提交。

---

## 19. Web 管理端建议

Runtime 页面：

```text
Runtime Release
Dispatch Policy Release
Dispatcher Type
Active Slots / Effective Limit
Scheduled asyncio Tasks
Semaphore Waiters
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

Attempt 详情：

```text
platform task id
attempt id
runtime task id
matched config release
local requeue observations
origin attempt count
final runtime outcome
stream batch id
materialized at
raw artifact id
```

这样才能区分：

- 平台 scheduler 等待；
- Runtime local queue 等待；
- Semaphore waiter 积压；
- Runtime 内存压力；
- 源站 429/503；
- config no-match；
- stream cancel；
- Artifact materialization 失败。

---

## 20. 推荐指标

```text
runtime_active_slots
runtime_scheduled_async_tasks
runtime_semaphore_waiters
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
result_cardinality_mismatch_total
raw_artifact_materialization_latency
```

其中 Runtime 指标只做执行面观察和 Admission 输入，不直接修改 Coverage、Document Version 或 PASS Markdown。

---

## 21. 最终结论

Crawl4AI Advanced Multi-URL Crawling 的工程价值很明确：

- `MemoryAdaptiveDispatcher` 提供局部内存压力滞回；
- `PriorityQueue` 提供当前 batch 内的局部公平；
- `SemaphoreDispatcher` 提供简单的固定抓取并发；
- `RateLimiter` 提供实例内 pacing/backoff；
- Streaming 提供边完成边物化；
- Monitor 提供 Runtime 级诊断；
- stream close 能显式取消并清理本地任务。

但它并不是一个 1000 站点的持久调度系统。

最关键的生产边界是：

```text
Crawl4AI Dispatcher
= local execution control

Platform Scheduler
= durable global scheduling truth

Runtime RateLimiter
= local pacing/backoff safety net

Platform Domain Admission
= strict cross-worker politeness truth

LOCAL_REQUEUED
= runtime transient event

Platform Retry
= durable task state transition

Semaphore bounded concurrency
!= bounded asyncio task count

Stream close
= cancellation + reconciliation event

Raw/IR/Markdown
= platform durable data plane
```

尤其需要牢记三个容易被文档字段掩盖的实现事实：

1. MemoryAdaptive 的 local requeue 会改变结果 cardinality，不能按列表位置关联；
2. Semaphore 当前真正限制抓取并发的是 `semaphore_count`，而且会为整批 URL 预创建 asyncio Task，所以 mega-batch 仍然危险；
3. Runtime RateLimiter 没有提供跨 Worker、严格串行的 domain token reservation，因此只能是平台共享限流之后的本地第二道保护。

把这些语义写进 Runtime Release、Adapter、Contract Test 和 Web 诊断后，Crawl4AI 才能安全地成为 1000+ 技术博客知识库的 Browser 执行层，而不会被误用成平台级 Frontier、Scheduler 或 Coverage 真相。