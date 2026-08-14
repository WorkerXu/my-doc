# OpenCMO：开源首席营销官智能系统

## 调研对象

- 项目：OpenCMO
- 地址：https://github.com/Lling0000/OpenCMO
- 调研基线：2026-08-14，`main` 最新提交 `2d2faf2e54365fd00b24810a0b12ffbfb0b23898`
- 重点代码：
  - `src/opencmo/tools/crawl.py`
  - `src/opencmo/tools/browser_pool.py`
  - `src/opencmo/scheduler.py`
  - `src/opencmo/services/monitoring_service.py`
  - `src/opencmo/web/routers/monitors.py`
  - `src/opencmo/web/routers/events.py`
  - `src/opencmo/background/service.py`
  - `src/opencmo/background/storage.py`
  - `src/opencmo/background/worker.py`
  - `src/opencmo/storage/jobs.py`
  - `docs/superpowers/specs/2026-04-02-unified-background-task-runtime-design.md`

## 1. 项目与博客知识库需求的关系

OpenCMO 不是“1000 个博客历史归档系统”。它的核心业务是对一个品牌/项目 URL 周期执行 SEO、GEO、SERP、社区、报告等扫描，并通过 Web 控制台查看结果。因此它缺少博客知识库最核心的完整历史发现能力，例如：

- Sitemap Index 持久化任务树；
- Feed/Archive/API 多 Provider checkpoint；
- Durable URL Frontier；
- Provider Coverage Ledger；
- 文章身份与版本模型；
- 百万级文章长期归档和 Markdown 知识库发布。

但 OpenCMO 的工程实现对“长期增量同步 + Web 管理”非常有价值，尤其是：

- 持久化周期计划；
- 可重建 Scheduler；
- Web 手工触发与重复运行保护；
- 通用后台任务状态机；
- Worker claim/heartbeat/recovery；
- append-only 任务事件；
- SSE 实时进度；
- Browser 并发闸门；
- 通用 runtime 与领域数据分离的设计思想。

因此最合理的复用方式不是直接采用 OpenCMO，而是吸收其控制面和后台任务运行时设计，再用博客知识库自己的 Discovery、Frontier、Snapshot、Article Version 模型替换其业务扫描逻辑。

## 2. URL 内容获取链路

`src/opencmo/tools/crawl.py` 的 `fetch_url_content()` 使用分层 fallback：

```text
Tavily Extract
  -> 无结果时 Crawl4AI
      -> Markdown 为空时 HTML metadata fallback
```

实现细节：

1. Tavily 以 `format="markdown"` 获取内容；
2. 若结果为空，进入 Crawl4AI；
3. Crawl4AI 外层使用 `asyncio.wait_for(..., timeout=90)`；
4. `result.markdown` 兼容字符串和 `MarkdownGenerationResult.raw_markdown`；
5. Markdown 仍为空时，从原始 HTML 中提取 title、description、OpenGraph、Twitter metadata；
6. 返回 `(content, source)`，明确标识 `tavily/crawl4ai/html_meta`。

这一实现最重要的不是 Tavily，而是两个原则：

### 2.1 分层获取

不同获取手段成本、稳定性和可复现性不同，应按廉价、可控、可回放到昂贵、复杂逐级升级。

博客知识库应转换为：

```text
HTTP 原文
 -> HTTP + 结构化抽取
 -> Browser/Crawl4AI 渲染
 -> 站点专用 API/Adapter
 -> 人工诊断
```

第三方 Extract API 不适合作为百万级历史回灌主链路，因为成本、配额、数据可复现性和访问策略都不受平台完全控制。

### 2.2 Provenance 必须保存

OpenCMO 会返回内容来源。博客知识库应进一步把来源写入 attempt/snapshot：

```text
fetch_engine
fallback_reason
browser_escalation_reason
runtime_release
adapter_version
source_provenance
```

不能只保存“最终拿到的 Markdown”，否则后续无法解释为什么同一页面两次抽取不同，也无法判断 Browser 成本来自哪些站点。

## 3. Browser 并发控制

`src/opencmo/tools/browser_pool.py` 使用：

```python
asyncio.Semaphore
```

限制同一进程、同一 event loop 中的 Crawl4AI 并发。默认并发为 1，通过 `OPENCMO_BROWSER_CONCURRENCY` 调整。Semaphore 使用 `WeakKeyDictionary` 按 event loop 保存，避免 event loop 生命周期变化后长期残留引用。

`crawl.py` 和 `scheduler.py` 都在创建 `AsyncWebCrawler` 前进入 `browser_slot()`。

### 优点

- 实现简单；
- 可以防止单进程瞬间启动大量 Browser；
- 对桌面/小型单机产品足够有效。

### 对 1000 站系统的局限

1. 只能限制单进程，多个 Pod 之间完全不知道彼此并发量；
2. 无法实现 per-site/per-domain QPS；
3. 无法表达 1000 站之间的 weighted fairness；
4. `fetch_url_content()` 的 fallback 每次调用都新建 `AsyncWebCrawler`，长期运行启动成本较高；
5. Browser 和 HTTP 使用相同业务入口时，容易让重资源任务挤占轻量 HTTP 工作。

因此生产博客知识库应采用：

```text
全局 Browser 配额
+ site/domain token bucket
+ Worker 内 semaphore
+ 长期复用 Browser/context/page pool
```

Browser Worker 应使用独立 runtime lane 和镜像，不让 Playwright/Crawl4AI 依赖影响 HTTP/Sitemap Worker。

## 4. 持久化 Schedule 与可重建 Scheduler

这是 OpenCMO 对本需求价值最大的部分之一。

`src/opencmo/storage/jobs.py` 将计划保存在 `scheduled_jobs`：

```text
id
project_id
job_type
locale
cron_expr
enabled
last_run_at
next_run_at
```

`src/opencmo/services/monitoring_service.py` 的操作流程是：

```text
create_monitor:
DB 创建 project/job -> sync runtime scheduler

update_monitor:
DB 更新 job -> sync runtime scheduler

remove_monitor:
DB 删除 job -> runtime scheduler 删除 job
```

`src/opencmo/scheduler.py` 把 APScheduler 当作可重建的运行时镜像，而不是唯一事实来源。其设计包括：

- 根据数据库记录创建/替换 runtime job；
- `CronTrigger.from_crontab()` 转换 cron；
- 启动时从数据库重新加载 enabled job；
- runtime job 丢失后可重新构造。

### 技术原理

Schedule 是“未来执行意图”，不应只存在内存时钟里。只要计划记录仍在数据库，Scheduler 重启后就应能恢复。

### 对博客知识库的进一步强化

OpenCMO 的 `run_scheduled_scan()` 仍会直接执行较大的业务扫描。对 1000 站平台应进一步拆分：

```text
Scheduler 到期
 -> DB 原子创建 crawl_run
 -> 写 Transactional Outbox
 -> Redis Streams
 -> Worker 执行 Discovery/Fetch/Extract
```

Scheduler 只做“触发”，不做网络抓取。

多 Scheduler 副本还需要：

- leader lease；或
- PostgreSQL `FOR UPDATE SKIP LOCKED` 对 due schedule 原子 claim。

否则每个副本都加载相同 cron 时可能重复触发。

## 5. Web 创建监控与首次运行

`src/opencmo/web/routers/monitors.py` 的创建流程很接近一个完整产品 onboarding：

```text
输入 URL
 -> URL 规范化
 -> 创建 project
 -> 创建 scheduled job
 -> 自动 enqueue 第一次 scan
 -> 返回 task_id
```

URL 处理会校验：

- scheme；
- hostname；
- port；
- 基本 DNS label 形式。

同时如果用户没有填写 brand/category，系统会自动推导默认值。

这说明 Web 管理系统不应只是几个独立 CRUD 页面，而应该把首次接入做成完整工作流。

博客知识库可改造为：

```text
录入博客根地址
 -> URL/Egress 预检查
 -> Discovery Probe
 -> Provider/Rule 草稿
 -> 样本抽取质量验证
 -> 管理员审核发布
 -> FULL_BACKFILL
 -> 自动创建 INCREMENTAL schedule
```

## 6. Web Run Now、重复任务与 Force 语义

OpenCMO 对每个 monitor 使用：

```text
scan:monitor:{monitor_id}
```

作为 dedupe key。

手工执行前会查询 active task：

- 已存在且 `force=false`：HTTP 409；
- `force=true`：先把旧任务标记 failed，再创建新任务。

### 可吸收点

Web 的“立即运行”必须有重复执行保护。否则用户双击、浏览器 retry、API Gateway retry 都可能创建两个扫描任务。

### 不应照搬的点

OpenCMO 的应用层去重流程是：

```text
find_active_task_by_dedupe_key()
 -> 没有
 -> insert_task()
```

如果多个请求同时进入，这个“先查再插”在多进程环境存在竞态。设计文档也明确选择了“服务层 dedupe，而不是 SQLite partial unique index”，这是对 SQLite 产品形态的折中，不适合作为 1000 站生产方案。

PostgreSQL 应使用数据库级约束，例如 active 状态 partial unique index，或用事务/锁把去重与创建做成一个原子动作。

建议逻辑键：

```text
sync:{site_id}:{schedule_id}:{crawl_mode}:{window_start}
```

### Force 语义改进

把旧任务直接标记 failed 会混淆“业务失败”和“被新任务替代”。博客知识库更适合：

```text
旧 run -> CANCEL_REQUESTED
新 run -> supersedes_run_id = 旧 run
```

既能停止旧任务，又保留完整审计关系。

## 7. 通用后台任务 Runtime

OpenCMO 的 `background/service.py`、`background/storage.py`、`background/worker.py` 已经形成相对完整的通用后台任务层。

任务字段包括：

```text
task_id
kind
project_id
status
payload
result
error
dedupe_key
priority
run_after
attempt_count
max_attempts
worker_id
claimed_at
heartbeat_at
started_at
completed_at
created_at
updated_at
```

状态大致为：

```text
queued -> claimed -> running -> completed
                         |-> failed
                         |-> cancel_requested -> cancelled
                         |-> queued(retry/recovery)
```

### 技术原理

关键变化是把“一个 Python coroutine 正在运行”升级为“一个持久化状态机正在运行”。

进程内 coroutine 可以随进程消失；持久化 task 则可以：

- 被重新 claim；
- 判断所有权；
- 记录进度；
- 检测 stale；
- 恢复重试；
- 提供 Web 状态。

这类模型非常适合百万 URL 的长期抓取。

## 8. Claim、Heartbeat 与恢复

### 8.1 Claim

OpenCMO 在 SQLite 下通过 `BEGIN IMMEDIATE` 包裹队列 claim，避免多个 Worker 同时获取同一任务。

对 PostgreSQL 应替换为：

```sql
SELECT ...
FROM background_task
WHERE status = 'QUEUED'
  AND run_after <= now()
ORDER BY priority DESC, created_at
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

或 `UPDATE ... RETURNING` / lease CAS。

即使前面有 Redis Streams，PostgreSQL claim 仍应是最终业务所有权确认。

### 8.2 Heartbeat

`BackgroundWorker` 默认每 5 秒 heartbeat；`recover_stale_tasks()` 根据 stale threshold 识别长时间没有 heartbeat 的 claimed/running task。

### 8.3 Orphan Recovery

启动 Worker 时会调用 `recover_orphaned_tasks()`。它不仅处理 heartbeat 过期任务，还处理：

```text
claimed/running + heartbeat IS NULL
```

这覆盖了“Worker claim 后尚未开始 heartbeat 就崩溃”的窗口。

### 8.4 Worker Ownership

任务完成、失败、heartbeat 更新等操作都会带 `worker_id` 条件。这样旧 Worker 失去所有权后，不容易覆盖新 Worker 的状态。

博客知识库应进一步增加 `lease_version/lease_until`，所有关键结果提交都以 CAS 方式检查当前 lease。

## 9. Worker 并发模型与潜在队头阻塞

`BackgroundWorker` 有两级本地 semaphore：

```text
max_concurrency
+ kind_concurrency
```

主循环先取得 global semaphore，然后 claim task；真正执行前再取得 kind semaphore。

这种设计对单机简单，但大规模系统存在一个潜在问题：

- Worker 已经 claim 某个任务；
- 该 kind 的 semaphore 已满；
- 任务处于 claimed/owned 状态，却只能等待本机 capacity。

如果大量重型 Browser/Report 任务先被 claim，可能造成 lease 长时间占用和局部队头阻塞。

博客知识库更适合：

- 按 `runtime_class` 分独立 queue/stream；或
- claim SQL 只选择当前 Worker 可执行类型；
- Dispatcher 在分发前考虑 runtime capacity。

即：不要 claim 自己暂时执行不了的任务。

## 10. Runtime 与领域数据分离

OpenCMO 的统一后台任务设计文档明确提出：

- `background_tasks` 是 runtime 状态真相；
- `scan_runs/reports/graph_expansions` 等领域表仍是业务结果真相；
- task result 只保存轻量摘要；
- executor 不应该知道通用 task 存储表细节。

这是非常重要的架构边界。

如果把文章正文、抓取快照、覆盖统计都塞进通用任务表，会导致：

- task 表不断膨胀；
- runtime 和业务 schema 强耦合；
- 未来拆 Worker 时很难迁移；
- 重试/清理任务时可能误伤业务结果。

博客知识库因此应坚持：

```text
background_task = 生命周期/所有权/重试/进度
fetch_snapshot   = 网络快照
article_version  = 文章业务版本
provider_coverage = 历史覆盖证据
```

Task 只通过稳定 ID 引用领域对象。

## 11. Task Event 时间线

OpenCMO 使用 `background_task_events` 保存：

```text
event_type
phase
status
summary
payload
created_at
```

queued、running、completed、failed、cancelled 以及业务 progress 都可以追加事件。

这比只保存一个 `status` 字段强很多，因为 Web 可以直接回答：

- 任务什么时候开始；
- 当前在哪个 phase；
- 为什么进入 retry；
- 是否发生恢复；
- 为什么终止。

博客知识库应扩展为 run/task/attempt 级 append-only timeline，例如：

```text
RUN_CREATED
DISCOVERY_STARTED
SITEMAP_SLICE_COMPLETED
FRONTIER_GROWTH
FETCH_THROTTLED
BROWSER_ESCALATED
SNAPSHOT_COMMITTED
EXTRACTION_REPLAYED
QUALITY_REJECTED
RETRY_SCHEDULED
TASK_RECOVERED
RUN_COMPLETED
```

## 12. SSE 实现细节与可扩展性问题

`src/opencmo/web/routers/events.py` 的 SSE 流程：

1. 建立连接后读取当前所有 task events；
2. 用进程内 `cursor` 按数组位置向后发送 progress；
3. 每 500ms 再次调用 `list_task_events(task_id)`；
4. 任务 terminal 后发送 `done` 并关闭。

优点：

- 连接建立后能 replay 已有进度；
- 服务重启后历史仍在数据库；
- 已完成任务也能立即返回完整历史 + done。

但对百万 URL 长任务，这种实现存在明显扩展性问题。

### 12.1 每次轮询重新读取完整历史

如果一个任务已经有 N 条 event，每 500ms 查询一次全部 N 条，随着事件增长会产生重复读取；长任务总成本可能接近 O(N²) 的扫描模式。

### 12.2 reconnect 从头 replay

cursor 只存在当前 SSE 请求内存里。连接断开后重新连接会从 0 开始，再读一遍全部历史。

### 12.3 没有标准 resume cursor

没有把数据库 event ID 映射到 SSE `id:` / `Last-Event-ID`。

### 对博客知识库的改进

事件表应使用单调 `event_id`，接口支持：

```text
GET /tasks/{task_id}/events?after_id=123
Last-Event-ID: 123
```

SQL：

```sql
SELECT ...
FROM task_event
WHERE task_id = $1
  AND event_id > $2
ORDER BY event_id
LIMIT $3;
```

服务端先 catch-up，再持续推送新事件；客户端按 event_id 去重。事件很多时可使用 PostgreSQL LISTEN/NOTIFY、Redis Pub/Sub 仅做 wake-up，但真正 event 数据仍从持久化表读取。

此外：

- 状态变化/恢复/失败等关键事件长期保留；
- 高频百分比或计数进度可以聚合/降采样；
- terminal event 必须明确发送并关闭连接。

## 13. 取消机制

`BackgroundWorker` 启动一个 cancel-watch coroutine，周期读取数据库状态。如果看到：

```text
cancel_requested
```

就对当前 asyncio task 执行 `task.cancel()`。

取消后：

- 如果状态是 cancel_requested，则标记 cancelled；
- 如果只是 Worker 自身 shutdown 导致取消，则任务可 requeue。

这个设计很好地区分了“用户取消”和“Worker 生命周期结束”。

### 对抓取系统的强化

仅调用 `asyncio.Task.cancel()` 不足以保证外部资源安全退出。博客知识库还需要：

- Browser page/context cleanup；
- HTTP streaming request abort；
- 在分页 slice 边界检查 cancel；
- Artifact Commit 事务前后检查 lease/cancel；
- 已上传但未引用 staging object 交给 Cleanup；
- cancellation 和 supersede 都写 append-only event。

因此取消应是“协作式安全终止”，而不是数据库状态改写。

## 14. Secret 持久化风险

`background/service.py` 的 `enqueue_task()` 会从当前请求上下文读取 BYOK keys，并把它们直接写进 task payload：

```text
payload["_byok_keys"] = keys
```

Worker 后续从 payload 恢复这些 key。

这对单机产品实现很方便，但对长期持久化任务系统是需要避免的安全模式，因为：

- task payload 落数据库；
- DB backup/replica 也会包含 secrets；
- 管理后台或 debug API 若展示 payload，容易泄漏；
- 后续 task event/error 若错误地复制 payload，也可能扩大泄漏面；
- key rotation 后，长期队列中的旧明文 key 仍可能存在。

博客知识库未来若支持需要凭据的网站、API 或对象发布目标，应保存：

```text
credential_profile_id
secret_ref
secret_version
required_scope
```

Worker 在真正执行 attempt 时从 Vault/Secret Manager/KMS 解析，并执行最小权限、日志脱敏和生命周期控制。

## 15. Scheduler 直接执行长业务的局限

OpenCMO 的 `run_scheduled_scan()` 是一个较长 orchestrator，会执行：

- Crawl4AI SEO 扫描；
- SERP；
- GEO；
- community；
- content frequency；
- insight；
- report 等。

每个子步骤通过 try/except 尽量相互隔离，这对单产品很实用，但对博客知识库不能照搬。

原因：

1. Scheduler 线程/进程会被长网络任务占用；
2. Browser、第三方 API、CPU 工作混在一个 orchestrator 中；
3. 多副本 Scheduler 更容易重复实际工作；
4. 很难对不同 runtime class 做独立扩容；
5. 一个超长函数中的错误恢复粒度较粗。

博客知识库应把一次 `crawl_run` 拆成持久化阶段 task：

```text
Discovery
 -> Fetch
 -> Browser Escalation
 -> Extract
 -> Quality
 -> Index
 -> Publish
```

每一阶段有独立幂等、lease 和 retry。

## 16. OpenCMO 不适合直接复用的能力

### 16.1 没有完整历史发现模型

OpenCMO 只围绕项目 URL 扫描，没有持久化 Sitemap 树、Archive 遍历、Provider checkpoint 和 Coverage Ledger，因此无法证明“历史已经抓全”。

### 16.2 SQLite 并发模型不能直接扩到百万 URL

`BEGIN IMMEDIATE` 适合 SQLite，但 PostgreSQL 需要 `SKIP LOCKED`、lease/CAS、partial unique index 等真正多 Worker 模型。

### 16.3 应用层 dedupe 存在并发窗口

多请求同时执行 `find -> insert` 时可能插入重复 active task，需要数据库级原子约束。

### 16.4 Browser semaphore 是进程内状态

只能作为局部保护，不能替代跨 Pod 配额和 per-domain politeness。

### 16.5 Scheduler 仍承担业务执行

博客知识库必须把 Scheduler 收缩为逻辑触发器。

### 16.6 SSE 完整历史重复扫描不适合长任务

需要 event_id cursor、增量查询和断线续传。

### 16.7 明文 BYOK keys 不应进入持久化 task payload

应替换为 Secret Reference。

## 17. 对博客知识库技术方案的优化结论

本次 OpenCMO 调研最终应吸收以下工程能力：

1. **Schedule 与 Run 分离**：`sync_schedule` 是未来配置，`crawl_run` 是 append-only 执行实例。
2. **Scheduler 可重建**：内存 scheduler 只是数据库计划的运行时镜像。
3. **Scheduler 不执行抓取**：只创建 run + outbox，所有长任务进入 Worker。
4. **数据库级重复运行保护**：应用层 dedupe 之外必须有 active partial unique index/原子事务。
5. **Force 使用 supersede/cancel**：不要把“被替代”伪装成普通失败。
6. **通用后台任务状态机**：queued/claimed/running/retry/cancel/completed/failed 全部持久化。
7. **Heartbeat + orphan recovery + lease ownership**：Worker crash 后可恢复，旧 Worker 不能覆盖新结果。
8. **按 runtime capacity 调度**：不要 claim 暂时执行不了的任务，避免局部队头阻塞。
9. **Runtime 与领域数据分离**：task 只保存生命周期和结果引用，业务结果仍在领域表。
10. **Append-only Task Events**：Web 进度、审计、恢复解释都来自结构化事件。
11. **SSE 使用 event_id 断线续传**：避免每次重连从头 replay，避免周期性完整历史扫描。
12. **高频事件聚合/降采样**：关键状态长期保留，纯进度不无限膨胀。
13. **Browser 三层限流并长期复用**：全局 + site/domain + 进程内。
14. **分层获取保存 provenance**：记录每次 fallback/升级原因和实际引擎。
15. **取消是协作式安全终止**：Worker、Browser、Artifact Commit 都需要 cancel safe point。
16. **任务 payload 禁止持久化 secrets**：只保存 Secret Reference，由 Worker 运行时解析。
17. **Web onboarding 串起 Probe、FULL_BACKFILL、INCREMENTAL schedule**，形成完整产品工作流。

这些优化不会改变博客知识库既有“PostgreSQL + Transactional Outbox + Redis Streams + Durable Frontier + HTTP-first + Browser 按证据升级 + 不可变 Snapshot + Article IR + append-only article_version”的主架构，而是把长期调度、任务恢复、实时管理、安全边界和多 Worker 扩展能力补成生产级实现。