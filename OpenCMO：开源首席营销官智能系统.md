# OpenCMO：开源首席营销官智能系统

## 调研对象

- 项目：OpenCMO
- 调研入口：https://github.com/Lling0000/OpenCMO
- 调研基线：2026-08-14 读取 `main`，代码搜索结果对应提交 `2d2faf2e54365fd00b24810a0b12ffbfb0b23898`
- 与博客知识库需求最相关的模块：`src/opencmo/tools/crawl.py`、`src/opencmo/tools/browser_pool.py`、`src/opencmo/scheduler.py`、`src/opencmo/services/monitoring_service.py`、`src/opencmo/web/routers/monitors.py`、`src/opencmo/background/service.py`、`src/opencmo/background/storage.py`、`src/opencmo/storage/jobs.py`

## 1. 项目与本需求的关系

OpenCMO 本身不是“1000 个博客全历史归档器”。它的主目标是对一个项目 URL 做 SEO/GEO/SERP/社区信号扫描，并按日、周、月周期重复执行，再把结果展示在 Web 控制台中。因此它不能直接替代博客知识库的 Discovery、Durable Frontier、全量回灌和文章版本系统。

但它对本需求有一组非常有价值的工程实现：**持久化周期计划、可重建的运行时调度器、Web 手工触发与重复运行保护、统一后台任务状态机、heartbeat、孤儿任务恢复、事件时间线、浏览器并发闸门**。这些能力正好补足“增量同步 + Web 管理”从功能描述走向可运维实现时容易遗漏的部分。

## 2. URL 内容获取链路

`src/opencmo/tools/crawl.py` 中的 `fetch_url_content()` 不是直接固定使用 Crawl4AI，而是使用两层策略：

1. 先调用 Tavily Extract，以 Markdown 形式尝试获得正文；
2. 如果没有内容，再进入 Crawl4AI；
3. Crawl4AI 调用外层有 90 秒 `asyncio.wait_for()`；
4. Crawl4AI 返回后优先取 `result.markdown`；兼容字符串和 `MarkdownGenerationResult.raw_markdown`；
5. 如果 Markdown 为空，再从原始 HTML 中提取 title、description、OpenGraph、Twitter metadata 作为兜底；
6. 返回内容时同时返回来源标记，例如 `tavily`、`crawl4ai`、`html_meta`。

这里真正值得复用的不是 Tavily 本身，而是“**分层获取 + 明确 provenance + 有界超时 + 降级链**”这个原则。本知识库不应把第三方 Extract API 放在历史全量主链路，因为成本、可复现性和访问策略不可控；但可以把同样的设计应用为：

```text
HTTP 原文
 -> HTTP + 结构化抽取
 -> Browser/Crawl4AI 渲染
 -> 站点专用 API/Adapter
 -> 人工诊断
```

每次升级都必须记录原因和执行来源，不能让“最终拿到了 Markdown”掩盖实际抓取路径。

## 3. Browser 并发控制

`src/opencmo/tools/browser_pool.py` 使用 `asyncio.Semaphore` 给 Crawl4AI 浏览器工作加全局进程内并发上限，默认并发为 1，可通过 `OPENCMO_BROWSER_CONCURRENCY` 调整。Semaphore 按 event loop 保存，使用 `WeakKeyDictionary` 避免生命周期泄漏。

`crawl.py` 在真正创建 `AsyncWebCrawler` 前进入 `browser_slot()`，因此即使上层同时发起多个 URL，也不会无限制创建浏览器实例。

这个实现对小型单进程系统很有效，但对本知识库只能作为最后一道进程内保护：

- 它无法约束多个 Pod/Worker 的总 Browser 并发；
- 无法表达 per-site/per-domain QPS；
- 无法实现 1000 站之间的 weighted fairness；
- `crawl.py` 的 fallback 每次调用都新建 `AsyncWebCrawler`，长期大规模运行会产生不必要的启动成本。

因此本方案应采用三层控制：全局 Browser 配额 + 站点/域名 token bucket + Worker 内 semaphore；Browser Worker 长期维护 browser/context/page pool，而不是每 URL 新建浏览器。

## 4. 持久化 Schedule 与可重建 Scheduler

这是 OpenCMO 最值得吸收的部分。

`src/opencmo/storage/jobs.py` 将周期计划保存在 `scheduled_jobs` 中，字段包括：

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

`src/opencmo/services/monitoring_service.py` 的 create/update/remove 流程都先修改持久化记录，再调用 scheduler 同步运行时状态：

- `create_monitor()`：创建 project -> 插入 scheduled job -> `_sync_runtime_job()`；
- `update_monitor()`：更新数据库 -> `_sync_runtime_job()`；
- `remove_monitor()`：先删除数据库记录，再从运行时 scheduler 移除。

`src/opencmo/scheduler.py` 则把 APScheduler 当成**可丢弃、可重建的内存镜像**：

- `sync_job_record()` 根据 DB 记录创建、替换或删除 APScheduler job；
- 使用 `CronTrigger.from_crontab()` 将 cron 表达式转为触发器；
- `load_jobs_from_db()` 启动时先清空内存 job，再从 DB 重新加载全部 enabled job；
- `start_scheduler()` 只负责启动时钟。

这个结构的关键原理是：**schedule 配置属于业务状态，不能只存在 APScheduler/进程内存里**。调度器宕机后重新启动，应能完全由数据库恢复。

对博客知识库应进一步强化：

- `sync_schedule` 保存在 PostgreSQL；
- scheduler 只负责发现“哪个 schedule 到期”，不直接执行抓取；
- 到期时创建一个新的 `crawl_run` 并通过 Transactional Outbox 投递；
- scheduler 多副本采用 leader lease 或数据库原子抢占，避免重复出发；
- `next_due_at`、timezone、jitter、overlap_window、policy_release 都持久化；
- 任何 Web 修改 schedule 都是 DB transaction，运行时只是 reconcile 后的结果。

## 5. Web 创建监控与首次运行

`src/opencmo/web/routers/monitors.py` 展示了一条完整的 Web 控制链：

1. 用户提交 URL；
2. 后端规范化 scheme/host/port，并拒绝明显非法 URL；
3. 自动推导 brand/category 等上下文；
4. 创建 monitor/schedule；
5. 创建完成后自动触发第一次 scan；
6. 返回 `task_id`，前端可以查询运行状态。

对本知识库很适合改造成：

```text
录入站点
 -> URL/Egress 预检查
 -> Discovery Probe
 -> 生成站点配置草稿
 -> 用户审核/发布
 -> 创建 FULL_BACKFILL run
 -> 后续创建 INCREMENTAL schedule
```

即 Web 页面不应该只是 CRUD，而应该把“站点接入 -> 首次回灌 -> 后续同步”做成同一条状态可追踪的 onboarding workflow。

## 6. 重复运行保护与 Force 语义

OpenCMO 在 monitor run 接口中使用固定 dedupe key：

```text
scan:monitor:{monitor_id}
```

如果同一个 monitor 已经存在 active task：

- 默认返回 409，拒绝再次启动；
- `force=true` 时先把旧任务标记失败，再创建新任务。

这个思想很重要：Web 的“立即运行”不能简单地每点一次就插入一个新任务，否则管理员连续点击、网络重试、浏览器重发请求都会制造并行扫描。

不过本知识库不应简单“强制失败旧任务”。更安全的语义是：

- 默认 `Run Now`：若同 schedule/site/crawl_mode 已有 active run，返回现有 run；
- `Force New Run`：创建新 run，并写 `supersedes_run_id`；旧 run 进入 `cancel_requested`，由 Worker 协作停止；
- 强制运行和定时运行仍走同一套 enqueue/idempotency 路径；
- 对 FULL_BACKFILL 默认禁止并行两个 active run，避免 frontier 和 coverage 判定混乱。

建议 dedupe key 包含计划窗口：

```text
sync:{site_id}:{schedule_id}:{crawl_mode}:{window_start}
```

数据库对 active 状态建立部分唯一约束，避免只靠“先查再插”的竞态保护。

## 7. 统一后台任务模型

`src/opencmo/background/service.py` / `storage.py` 实现了一个通用 task runtime。任务至少包含：

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

生命周期大致为：

```text
queued -> claimed -> running -> completed
                         |-> failed
                         |-> cancel_requested -> cancelled
                         |-> queued (retry/recovery)
```

其价值在于把业务任务变成**可持久化状态机**，而不是 Python coroutine 的生命周期。

### 7.1 Claim 原子性

OpenCMO 使用 SQLite `BEGIN IMMEDIATE` 包住 `SELECT queued task + UPDATE claimed`，避免两个 Worker 同时拿到同一任务。这是 SQLite 下正确的思路，但 PostgreSQL 生产方案应替换为：

```sql
SELECT ...
FROM background_task
WHERE status = 'QUEUED' AND run_after <= now()
ORDER BY priority DESC, created_at
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

或使用单条 `UPDATE ... RETURNING`/lease CAS。若主分发采用 Redis Streams，则数据库 claim 仍应作为最终状态所有权确认，而不能把 Redis consumer ownership 当作业务真相。

### 7.2 Heartbeat 与孤儿任务恢复

OpenCMO 的 Worker 会写 heartbeat。服务层提供：

- `recover_stale_tasks()`：heartbeat 超时则重新排队或达到最大尝试后失败；
- `recover_orphaned_tasks()`：进程重启后连 heartbeat 为空的 claimed/running task 也能恢复；
- 重试时递增 attempt_count；
- 状态更新要求 worker_id/status 匹配，防止旧 Worker 在 lease 丢失后覆盖新 Worker 结果。

这直接对应本知识库的长期运行需求。对数百万 URL，任何 Worker crash 都必须只造成有限重放，不能产生永久“running”僵尸任务。

## 8. Task Event 时间线

OpenCMO 为每个后台任务写 `background_task_events`：queued、started、completed、failed、cancelled 等状态变化和阶段进度都可以追加事件。

这对 Web 管理比单一 `status` 字段更有价值。博客知识库应该为 `crawl_run/task/attempt` 建立 append-only event timeline，例如：

```text
RUN_CREATED
DISCOVERY_STARTED
SITEMAP_SLICE_COMPLETED
FRONTIER_GROWTH
FETCH_THROTTLED
BROWSER_ESCALATED
EXTRACTION_REPLAYED
QUALITY_REJECTED
RUN_CANCEL_REQUESTED
RUN_RECOVERED
RUN_COMPLETED
```

Web 可以直接用这些事件实现 SSE/WebSocket 进度，不需要从日志文本猜状态；同时它们也是审计和故障复盘材料。

## 9. OpenCMO 不适合直接复用的部分

### 9.1 没有完整历史发现模型

OpenCMO 的核心抓取是项目 URL 扫描，没有 Sitemap Index 持久化树、Feed/Archive 多 Provider checkpoint、durable URL frontier 和 coverage ledger，因此不能证明“历史文章已经抓全”。

### 9.2 Scheduler 中存在直接执行大扫描的能力

`run_scheduled_scan()` 是一个很长的 orchestrator，会依次或并行执行 SEO、SERP、GEO、社区、报告等工作。对中小系统可接受，但在博客知识库中 scheduler 不应直接做任何长网络或 Browser 工作。Scheduler 只产生 run/task；实际工作必须进入独立 Worker。

### 9.3 APScheduler 是进程内时钟

单进程运行没有问题，但多副本部署若每个实例都加载同一 cron，会重复触发。生产环境必须通过 leader lease、DB due-row claim 或专门的调度服务保证同一 schedule 只有一个 logical trigger。

### 9.4 Browser semaphore 只在进程内有效

无法实现跨 Pod 的总配额和 per-domain politeness。本知识库要在进程内 semaphore 之外增加全局配额与站点/域名限流。

### 9.5 SQLite 并发模型不能照搬

`BEGIN IMMEDIATE` 是 SQLite 的并发控制方式。1000 站长期运行的主状态库应使用 PostgreSQL，并用 `FOR UPDATE SKIP LOCKED`、lease/CAS、advisory lock 或唯一约束实现并发安全。

## 10. 对博客知识库技术方案的最终优化结论

本次调研后，技术方案应明确补入以下约束：

1. **周期计划与执行实例分离**：`sync_schedule` 是持久配置，`crawl_run` 是 append-only 执行实例；任何计划变更不修改历史 run。
2. **Scheduler 可重建**：内存 scheduler/leader 只是时钟和缓存，启动时从 PostgreSQL reconcile；数据库才是 schedule 真相源。
3. **Scheduler 不执行抓取**：到期只在同一事务创建 run + outbox，真实工作全部由 Worker 执行。
4. **重复运行有数据库级保护**：`dedupe_key + active partial unique index`，Web 重试和 cron 重复触发不能制造重复 active run。
5. **Force 有显式 supersede/cancel 语义**：不静默覆盖旧任务状态。
6. **后台任务必须有 heartbeat/lease/attempt/recovery**：claimed/running 任务在 Worker crash 后自动恢复，旧 Worker 失去 lease 后不能提交结果。
7. **任务事件 append-only**：作为 Web 进度、审计、恢复解释和故障分析的数据源。
8. **Web onboarding 自动串起 Probe、首次 FULL_BACKFILL 和后续 INCREMENTAL schedule**，而不是三个孤立页面。
9. **Browser 三层限流**：全局配额 + site/domain 配额 + 进程内 semaphore；Crawl4AI/Browser 生命周期长期复用。
10. **分层获取必须保留 provenance**：HTTP、Browser、第三方 Adapter、metadata fallback 的结果来源和升级原因都进入 snapshot/attempt 元数据。

这些优化不会改变现有“PostgreSQL + Outbox + Redis Streams + HTTP-first + Browser 按证据升级 + 不可变 Snapshot + Article IR + append-only article_version”的主架构，而是把长期增量同步与 Web 运维部分补成真正可恢复、可并发、可审计的生产级控制面。
