# TrendSearch：多源热点资讯搜索与 AI 分析系统

项目地址：https://github.com/LIhong42/TrendSearch

## 1. 项目定位

TrendSearch 是一个“多数据源热点采集 → 网页正文抓取 → LLM 分析/需求挖掘 → 排序评分 → 报告生成 → Web 展示/飞书推送”的完整应用。它不是面向千站历史归档设计的通用爬虫平台，但代码中包含若干对博客知识库有直接参考价值的工程模式：统一数据源接口、结构化 API 负责低成本发现、Crawl4AI 负责网页正文补全、数据库唯一约束、分阶段 AI Pipeline、FastAPI + React Web、用户运营状态、以及 Channel/MessageBus 形式的消息投递抽象。

对“约 1000 个技术博客抓取全量历史文章，并长期增量同步为 Markdown 知识库”的目标而言，TrendSearch 更适合作为“小规模原型 + 反例集合”。它证明了 Adapter、Pipeline、Web、Channel 分层方向是有效的，同时也清楚暴露出从热点聚合扩展到长期知识库时的边界：URL 存在即永久跳过、Browser 串行抓取、随机 sleep 代替全局限流、进程内 gather、进程内消息队列、SQLite 单机状态、30 天自动清理、批量完成后才落库、Web 全量读入后过滤，以及通知发送缺少 durable delivery 语义。

## 2. 代码架构与主数据流

仓库主要结构：

```text
frontend/                         React + TypeScript + Ant Design
backend/app/data_sources/         数据源适配器
backend/app/pipeline/             分阶段处理流水线
backend/app/agent_framework/      LLM Agent、Prompt、工具
backend/app/storage/              SQLite 数据层
backend/app/router/               FastAPI API
backend/app/analyze_script.py     总流程编排
backend/app/channels/             消息总线、Channel、飞书实现
backend/app/embedding/            Embedding 抽象
```

核心数据流：

```text
DataSource.fetch()
  -> DataItem(title/url/source/...)
  -> fetch_with_urls()
  -> Crawl4AI 抓网页并生成 Markdown
  -> SingleNewsDemandMiningPipeline
  -> LLM 分析 + Embedding
  -> SQLite news/demands
  -> Report Pipeline
  -> FastAPI / React / Feishu
```

这套分层对应用级项目足够清晰，但知识库需要继续拆分为独立可靠性域：

```text
Source Adapter
  -> Discovery
  -> URL Admission
  -> Durable Frontier
  -> HTTP/Browser/PDF Fetch
  -> Immutable Snapshot
  -> Extraction Candidate
  -> Canonical Document IR
  -> Fitness / Drift / Quality
  -> Accepted Document Version
  -> Search / Markdown / LLM Analysis / Publish Projection
```

原因是这些阶段的重试、幂等、完成判定、版本和成本语义完全不同，不能由一次 Python 调用链隐式承担。

## 3. 数据源策略模式：最值得保留的思想

### 3.1 `BaseDataSource + DataItem`

`backend/app/data_sources/base.py` 定义了标准 `DataItem`：

```text
title
url
content
source
summary
```

同时定义抽象 `BaseDataSource.fetch()`。不同来源只需要实现各自获取逻辑，上层 Pipeline 不必理解 Hacker News、知乎、GitHub 等具体协议。

这是典型 Strategy/Adapter 模式，价值在于：

1. 来源接入点统一；
2. 上层消费统一结构；
3. 新来源对现有流程影响较小；
4. 可以把“平台差异”限制在一个边界内。

### 3.2 当前注册方式为什么不能直接扩到 1000 站

`backend/app/data_sources/__init__.py` 显式 import 各 Source，再通过：

```python
{cls.__name__: cls() for cls in BaseDataSource.__subclasses__()}
```

构造全部数据源实例。

小规模时简单直接，但生产知识库存在这些问题：

- 注册依赖代码 import，无法由管理端动态启停；
- Adapter 没有业务版本，无法复现“某次 Run 用了哪一版”；
- 没有 capability 描述，不知道是否支持 BACKFILL、INCREMENTAL、cursor、分页、认证、附件、文档导航；
- 没有配置 schema/result schema；
- 没有 Golden Sample、Canary、健康检查和回滚；
- 站点配置与 Worker 镜像发布耦合；
- 一个特殊来源的故障容易扩大到整个运行时。

因此知识库应升级成：

**Versioned Source Adapter SDK + Capability Registry + Adapter Binding Release**。

能力至少描述：

```text
DISCOVER
BACKFILL
INCREMENTAL
CURSOR_PAGINATION
TIME_RANGE_PAGINATION
AUTH
SITEMAP
FEED
API
ARCHIVE
DOC_NAVIGATION
BROWSER_DISCOVERY
ATTACHMENT_DISCOVERY
```

Adapter 输出的应是标准 `DiscoveryRecord/FetchHint`，而不是最终 Markdown：

```text
source_item_id
source_url
canonical_hint
published_at_hint
updated_at_hint
cursor/page_token
content_type_hint
fetch_route_hint
discovery_evidence
raw_metadata_ref
```

然后统一进入 Admission、Frontier、Content Fetch、Extraction 和 Quality。

## 4. “结构化发现 + 通用正文抓取”的技术原理

`backend/app/data_sources/hackernews.py` 使用 Algolia HN API 获取条目，再构造 `DataItem`，随后调用 `fetch_with_urls()` 抓正文。

这个设计揭示了一条很适合知识库的原则：

**URL 发现优先走廉价、结构化、可分页、带时间语义的数据源；正文获取再走统一 Fetch 层。**

映射到技术博客：

```text
Sitemap / Feed / API / Archive / Docs Navigation
                    ↓
               URL Discovery
                    ↓
              Durable Frontier
                    ↓
          HTTP-first Content Fetch
                    ↓
        有明确证据才升级 Browser
```

相比从首页递归点击所有链接，这更容易控制成本，也更容易证明历史覆盖。

TrendSearch 的不足是 `fetch_with_urls()` 仍放在 `BaseDataSource` 内，Discovery 与 Content Fetch 耦合。生产系统应彻底分离：Adapter 只表达来源特有发现语义；通用 Fetch 负责条件请求、快照、重试、限流、Content-Type 路由、Browser 升级、质量与审计。

## 5. Crawl4AI 使用方式及吞吐瓶颈

`BaseDataSource.fetch_with_urls()`：

1. 创建一个 `AsyncWebCrawler` context；
2. `for item in datalist` 顺序遍历；
3. 对每个 URL 调用 `crawler.arun()`；
4. `magic=True`，并设置 `delay_before_return_html=1.0`；
5. 把 `result.markdown` 写入 `item.content`；
6. 每项随机 sleep 1~4 秒。

优点是没有每个 URL 重建 Browser，且单项异常会被捕获。但吞吐模型本质仍是串行：

```text
for item:
    await crawler.arun(item.url)
    await sleep(...)
```

对于百万级 URL 历史回灌不可扩展。随机 sleep 只能约束一个 Python 进程，无法阻止多个 Worker 同时轰击同一域名，也无法处理 429/Retry-After、站点级公平性和 Browser 资源隔离。

生产设计应为：

- PostgreSQL durable frontier 小批量 claim；
- domain/site 级 token bucket + 最大并发；
- HTTP Worker 与 Browser Worker 分池；
- Browser 只在 Preflight 证据充分时升级；
- Browser runtime 长生命周期复用；
- 每个 URL 的 Snapshot/Attempt 立即提交，而不是整批结束再落库；
- 429/503 依据 Retry-After 做域级退避与熔断；
- worker-local Semaphore 只做局部资源保护，跨 Worker 配额由控制面负责。

## 6. `clear_markdown_url()`：对知识库是不可逆信息损失

TrendSearch 用正则把：

```text
[text](https://example.com)
```

转换为：

```text
text
```

热点摘要场景可以减少视觉噪声，但知识库不能这样处理，因为 URL 承载：

- 原始引用关系；
- 站内上下文与上一篇/下一篇；
- 图片、附件和源码地址；
- broken-link 检测；
- 文档图谱；
- provenance 与审计；
- 后续重新抓取的线索。

因此最终 Markdown 不应是唯一真相。应先保留 Canonical Document IR 和 `document_link_edge`，再从 IR 生成 Markdown。如果需要“无 URL 的 LLM 文本版”，它只能是额外 Projection，不能覆盖结构化真相。

## 7. URL 去重正确，但 `url_exists()` 不能代表 Freshness

### 7.1 值得保留：数据库唯一约束

`storage.py` 使用 SHA256(url) 得到 `url_hash`，`news.url` 与 `url_hash` 均有 UNIQUE，并使用 `INSERT OR IGNORE`。

这说明一个正确原则：数据库唯一约束是最终幂等保障；内存 Set、Bloom Filter 只能加速。

### 7.2 关键问题：已存在 URL 被永久跳过

抓取前执行：

```text
if data_store.url_exists(url):
    skip fetch
```

这把“identity 已知”错误等价为“内容永远新鲜”。技术博客同 URL 会发生正文修订、代码修复、标题/标签变化、canonical 改变、文档滚动升级和 redirect。

正确语义应该是：

```text
known URL
  -> next_check_at
  -> If-None-Match / If-Modified-Since
  -> 304: 只更新 freshness
  -> 200 + body/IR hash 未变: 更新 freshness
  -> 200 + IR 实质变化: 创建 document_version candidate
```

Discovery 再次看到已知 URL 也应 UPSERT `last_seen_at/discovery_evidence/freshness_hint`，而不是丢弃本次观察事实。

## 8. `asyncio.Semaphore + gather` 的边界

`SingleNewsDemandMiningPipeline` 用：

```python
self.semaphore = asyncio.Semaphore(max_concurrency)
```

限制 LLM 调用并发，这是合理的 worker-local 保护。

但 `run_for_all_sources()` 的流程是：

1. 对所有 Source 逐个 `await source.fetch()`，抓取阶段串行；
2. 全部新闻积累到内存；
3. 一次构造全部 `_process_single_news()` coroutine；
4. `asyncio.gather(..., return_exceptions=True)`；
5. 全部处理完后批量入库。

这个模式适合短任务，不适合长期爬虫：

- 进程崩溃后内存任务全部丢失；
- 无 lease/heartbeat/checkpoint；
- 无 task-level idempotency；
- 大批量对象长期驻留内存；
- 来源串行导致头阻塞；
- Semaphore 只约束 LLM，不约束 HTTP、Browser、PDF、OCR、数据库等资源；
- 无站点公平调度；
- 批量落库前发生故障，会出现大量重复工作。

所以生产知识库应采用“小批 claim + durable task + lease + bounded concurrency + 每项提交”。`asyncio` 仍可作为 Worker 内部执行模型，但不能成为业务队列和业务状态真相。

## 9. SQLite WAL 的合理性与生产边界

TrendSearch 为 SQLite 开启 WAL，使用 thread-local connection，并为 URL、时间、score 等建立索引。这对单机应用是务实选择。

但千站知识库需要：

- 多 Worker 并发；
- `SKIP LOCKED` 任务 claim；
- lease/heartbeat；
- Outbox；
- HA；
- 百万文档与千万 URL 元数据；
- 大量版本和审计记录；
- 复杂服务端过滤与聚合。

因此 PostgreSQL 更适合作为生产业务真相源。SQLite 可以保留为本地开发、测试或离线单机演示模式。

## 10. 30 天自动清理与历史知识库目标直接冲突

`DataStore.__init__()` 会调用：

```text
clean_old_data(days=30)
```

并删除超过 cutoff 的 news/demands。

对热点产品合理，对历史知识库却是反目标。知识库应按数据角色做 retention：

```text
AUTHORITATIVE
  accepted document/version
  lineage root snapshot
  quality/drift manifest
  run context / audit
  默认长期保留

RECONSTRUCTIBLE
  extraction candidate
  index payload
  可重建 projection
  确认可从权威对象重放后才回收

EPHEMERAL
  staging/temp/cache
  可 TTL
```

GC 应是 lineage-aware mark-and-sweep，并记录 tombstone/audit，不应简单按“超过 N 天”删除。

## 11. Web API：功能形态值得借鉴，查询方式不能照搬

TrendSearch 已有 FastAPI + React 管理/阅读界面，这说明抓取系统最终需要可视化运营面，而不是只靠 CLI。

但 `/api/news` 和 `/api/demands` 的实现会：

1. 先从 SQLite 获取 12h/24h 的全部记录；
2. 在 Python 内存里做 source/keyword/score 等过滤；
3. 再排序；
4. 注释明确“不再分页，返回全部数据”。

在几十/几百条热点数据上没问题，在百万文档上会导致 API 内存、延迟和网络传输失控。

生产知识库应改成：

- PostgreSQL/OpenSearch 服务端过滤；
- keyset/cursor pagination；
- 聚合统计走 materialized view/预聚合；
- 详情页按 document/version 精确读取；
- Dashboard 不做全表 Python filter；
- 浏览器刷新不影响后台 Run/Task 状态。

## 12. Channel + MessageBus：值得升级成持久化通知投递层

这是 TrendSearch 中现有博客知识库方案容易忽略的一部分。

### 12.1 实现结构

`backend/app/channels/bus/events.py` 定义：

```text
InboundMessage
  channel
  sender_id
  chat_id
  content
  timestamp
  media
  metadata
  session_key

OutboundMessage
  channel
  chat_id
  content
  reply_to
  media
  metadata
```

`backend/app/channels/bus/queue.py` 的 `MessageBus` 用两个 `asyncio.Queue`：

```text
inbound  : channel -> core
outbound : core -> channel
```

`ChannelManager` 负责：

- 初始化启用的 Channel；
- 启停 Channel；
- 后台 `_dispatch_outbound()` 消费出站消息；
- 按 `msg.channel` 路由到具体实现；
- 当前包含 Feishu；
- 提供 `send_notification()` 与报告广播方法。

这个结构的思想是正确的：核心逻辑不直接绑定某个消息平台，Channel 作为 Adapter，MessageBus 作为解耦边界。

### 12.2 为什么不能直接用于生产告警

当前 MessageBus 是进程内 `asyncio.Queue`：

- 进程重启，未消费消息丢失；
- 没有持久化 delivery 状态；
- 没有 idempotency key；
- 没有 retry schedule/dead-letter；
- 没有 dedup/silence/escalation；
- `send_notification()` 还能直接 `await ch.send()`，调用方可能被外部信道延迟拖慢；
- 广播是顺序发送，单信道故障只能日志记录。

### 12.3 对知识库方案的升级方式

知识库应把这个模式升级为独立 **Notification/Delivery Plane**：

```text
Domain Event / Alert Decision
      ↓
notification_delivery 写 PostgreSQL
      ↓
Transactional Outbox
      ↓
Notification Worker
      ↓
Channel Adapter
  Feishu / Slack / Email / Webhook
```

核心原则：

1. 抓取 Run 完成不依赖消息平台可用性；
2. 通知失败不能把已成功抓取的 Run 改成 FAILED；
3. delivery 自己有 `PENDING/SENDING/SENT/RETRY/DEAD`；
4. `(event_id, channel_binding_id)` 做幂等；
5. 支持 retry/backoff、dedup window、silence、severity、route rule；
6. 支持 Run PARTIAL、dead-letter、coverage gap、drift、freshness lag、access challenge、GC blocked 等结构化事件；
7. Web 可查看投递历史并手工重发。

这样既保留 TrendSearch Channel abstraction 的优点，又补上生产可靠性。

## 13. `is_liked`：从简单点赞演化为独立人工标注/运营反馈层

`news` 和 `demands` 表都包含 `is_liked`，存储层还有 `toggle_news_like()` / `toggle_demand_like()`。这说明 TrendSearch 已经把“机器生成内容”和“用户运营状态”放在同一产品中考虑。

对知识库而言，不能简单复制一个 boolean，而应把它升级成 append-only 的 **Curation/Feedback Plane**：

```text
curation_event
  event_id
  actor_id
  document_id/version_id
  action
  label/note/reason
  created_at
  supersedes_event_id
```

可支持：

- favorite / pin；
- 标签、集合、笔记；
- “正文缺失”“抽错区域”“metadata 错误”“需要重抓”等质量反馈；
- 人工 approve/reject/quarantine；
- 发起 TARGETED_FETCH 或 REPROCESS；
- 对站点/规则的人工备注。

关键边界：

- 人工反馈是独立事实，不能直接覆盖不可变 Snapshot；
- 不能因为“点了赞”就改变源文档 identity；
- 质量修正应触发新 candidate/version/override event，而不是原地改旧版本；
- 所有人工 Override 保存 actor、reason、时间和被覆盖的机器判定；
- 可将反馈作为规则优化/Golden Sample 的输入，但不能让线上 LLM 直接自学习后无版本发布。

这会显著提升千站长期运营能力，因为真实生产问题往往需要“人发现异常 → 定位 URL → 重处理/修正规则 → 可审计恢复”的闭环。

## 14. AI 分析应该是 Accepted Document 的下游 Projection，而不是抓取主链路

TrendSearch 的业务价值主要来自 LLM 需求挖掘、评分和报告，但它把这些阶段放在一次主分析流程里。

知识库不应让 LLM 分析阻塞源内容同步。更合理的是：

```text
Accepted Document Version
   ├─ Markdown Projection
   ├─ Search Index
   ├─ Embedding
   ├─ Summary/Tags
   ├─ Topic/Entity Extraction
   └─ Periodic Report
```

派生 AI Job 必须：

- 引用确定的 `document_version_id`；
- 记录 model/prompt/schema/release；
- 可独立 retry/reprocess；
- 失败不影响 accepted source document；
- 新模型上线可基于已有 Document IR 重算，不重新访问源站；
- 结果作为 Derived Artifact，不反向覆盖源正文；
- 成本与配额独立控制。

这样既能保留 TrendSearch “采集后进一步分析”的能力，又不会把 LLM 的非确定性和费用引入抓取可靠性核心。

## 15. 完成状态不能由协程结束或报告生成来证明

TrendSearch 的流程编排主要由 `analyze_script.py` 直接调用 Pipeline，异常多通过日志/try-except 处理。对于应用脚本足够，但知识库必须定义独立业务完成语义。

建议：

```text
Worker success
   ≠ Stage success
   ≠ Run success
```

Stage Finalizer 应对账：

- 计划实体数；
- admitted/frontier 数；
- fetch attempt 与 COMPLETE Snapshot；
- extraction candidate；
- quality result；
- retry/dead-letter；
- accepted version；
- publish/index manifest。

最后给出 `COMPLETED/PARTIAL/FAILED/INCONSISTENT`，避免“协程都结束了但部分产物没落盘”的假成功。

## 16. 可直接迁移与必须重构的清单

### 可以保留的思想

- Strategy/Adapter 数据源接口；
- 标准化数据对象；
- 结构化 API 优先用于发现；
- Crawl4AI 作为网页正文候选生成器；
- 数据库唯一约束；
- worker-local Semaphore；
- Pipeline 分阶段思想；
- FastAPI + React 管理面；
- Channel Adapter + 消息事件抽象；
- 用户运营/反馈状态；
- 下游 AI 分析与报告能力。

### 必须重构的实现

- import-time `__subclasses__()` → Versioned Capability Registry；
- DataSource 内直接抓正文 → Discovery/Fetch 分离；
- Browser 全量串行 → HTTP-first + Browser evidence escalation；
- random sleep → 全局 domain quota + Retry-After；
- `url_exists => skip forever` → identity/freshness 分离；
- Markdown 链接删除 → Document IR + Link Graph；
- 大 `gather()` → durable task/frontier；
- 整批结束才入库 → 每项/小批持久化；
- SQLite → PostgreSQL 生产状态真相；
- 30 天统一清理 → lineage-aware Retention/GC；
- API 全量加载后过滤 → 服务端查询 + keyset pagination；
- `asyncio.Queue` MessageBus → durable Notification Outbox；
- boolean `is_liked` → append-only Curation/Feedback Event；
- LLM 分析内嵌主链路 → Accepted Version 下游 Derived Job；
- try/except + 日志完成 → Artifact 对账 Finalizer。

## 17. 对最终博客知识库方案的结论

TrendSearch 不应作为千站知识库的直接基础代码，但很适合作为架构启发来源。最值得吸收的是四类思想：

1. **来源 Adapter 化**：不同来源统一协议，上层消费统一对象；生产中进一步升级为版本化 Capability Registry。
2. **结构化发现与网页正文分工**：Feed/API/Sitemap/Archive 负责低成本枚举，正文统一进入 HTTP-first Fetch/Browser 路由。
3. **产品化运营面**：Web 不只展示结果，还要承载站点、Run、URL、版本、质量、人工标注和重处理闭环。
4. **Channel 与下游分析解耦**：通知、报告和 LLM 分析都应成为 accepted 文档之后的独立可靠性域，不能绑死抓取主链路。

对于约 1000 个技术博客，真正决定长期可靠性的不是“一个 URL 能不能被 Crawl4AI 转成 Markdown”，而是能否证明历史抓全、已知 URL 持续校验 freshness、动态页不会静默抓空、模板变化不会静默抓错、链接与结构不会在清洗时丢失、任务与通知在进程崩溃后可恢复、人工反馈可审计地形成修复闭环、历史不会被错误 TTL 清理、以及每个最终 Markdown/派生 AI 结果都能追溯到确定的 Snapshot、Run Context、规则版本和 Document Version。