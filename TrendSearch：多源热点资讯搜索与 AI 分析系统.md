# TrendSearch：多源热点资讯搜索与 AI 分析系统

项目地址：https://github.com/LIhong42/TrendSearch

## 1. 项目定位

TrendSearch 是一个“多数据源热点采集 → 网页正文抓取 → LLM 分析/需求挖掘 → 排序评分 → 报告生成 → Web 展示/飞书推送”的完整应用。它不是面向千站历史归档设计的通用爬虫平台，但代码中已经出现了几个非常值得复用的工程模式：统一数据源接口、廉价 API 负责源数据发现、Crawl4AI 负责网页正文补全、URL 唯一约束、流水线分阶段处理、Web 查询与运营界面。

对“1000 个技术博客全量历史抓取并长期增量同步”的目标而言，TrendSearch 更适合作为一个反例与原型样本：它证明了数据源适配器模式和 Web 管理是有效的，但也暴露了小规模聚合系统扩展到长期知识库时会遇到的核心问题，包括“URL 存在即永久跳过”、抓取任务非持久化、浏览器串行抓取、进程内 gather、SQLite 单机状态、30 天自动清理、完成状态与真实产物不对账、Web 全量读入后过滤等。

## 2. 代码架构与数据流

仓库大体分为：

```text
frontend/                         React + TypeScript + Ant Design
backend/app/data_sources/         数据源适配器
backend/app/pipeline/             分阶段处理流水线
backend/app/agent_framework/      LLM Agent 与 Prompt
backend/app/storage/              SQLite 数据层
backend/app/router/               FastAPI API
backend/app/analyze_script.py     总流程编排
backend/app/channels/             飞书等输出渠道
```

核心链路可概括为：

```text
DataSource.fetch()
  -> DataItem(title/url/source/...)
  -> fetch_with_urls()
  -> Crawl4AI 抓网页并生成 Markdown
  -> SingleNewsDemandMiningPipeline
  -> LLM 分析
  -> SQLite news/demands
  -> Report Pipeline
  -> FastAPI/React/飞书
```

这一分层说明项目作者已经把“来源接入”“内容抓取”“AI 分析”“存储”“展示/分发”做了初步解耦。不过对于历史知识库，Discovery、Content Fetch、Extraction、Version、Quality、Publish 还需要继续拆开，因为它们的可靠性语义完全不同。

## 3. 数据源策略模式：最值得复用的部分

### 3.1 BaseDataSource + DataItem

`backend/app/data_sources/base.py` 定义 `DataItem`，统一字段包括：

```text
title
url
content
source
summary
```

同时定义抽象基类 `BaseDataSource`，要求所有来源实现异步 `fetch()`。不同平台只负责自己的发现逻辑，例如 Hacker News 通过 Algolia API 拉取热门条目，然后统一交给基类的网页正文抓取函数。

这是典型 Strategy/Adapter 模式。它带来三个优点：

1. 上层 Pipeline 不需要知道 Hacker News、知乎、GitHub 等具体协议；
2. 新增来源时改动面比较小；
3. 标准化 `DataItem` 后，分析流水线可以复用。

### 3.2 当前注册机制的局限

`data_sources/__init__.py` 通过显式 import 各 Source 类，再用：

```python
{cls.__name__: cls() for cls in BaseDataSource.__subclasses__()}
```

得到全部数据源实例。

这在十几个来源时足够简单，但不适合 1000 个博客站点：

- 注册发生在代码 import 阶段，无法由管理端动态启停；
- 没有 adapter version，无法知道某次 Run 用的是哪一版实现；
- 没有 capability 描述，系统不知道适配器是否支持历史回溯、增量 cursor、分页、认证、附件、站点导航等；
- 没有配置 schema/result schema，错误配置只能运行时发现；
- 没有健康检查、Golden Sample、Canary、回滚；
- 某个适配器的代码发布与整个 Worker 镜像强绑定。

因此在知识库方案里应该保留“统一适配器接口”的思想，但升级为 **Versioned Source Adapter SDK + Capability Registry + Adapter Binding Release**。

建议标准能力声明：

```text
DISCOVER
BACKFILL
INCREMENTAL
FETCH_METADATA
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

适配器不应该直接把页面抓成最终 Markdown。更合理的输出是标准化 `DiscoveryRecord/FetchHint`：

```text
source_item_id
source_url
canonical_hint
published_at/updated_at
cursor/page_token
content_type_hint
fetch_route_hint
discovery_evidence
```

随后统一进入 Admission、Frontier、HTTP/Browser Router、Extraction 与 Quality 流程。这样站点特有逻辑不会侵入通用抓取内核。

## 4. “API 发现 + Crawl4AI 正文抓取”的技术原理

Hacker News 数据源通过 Algolia API 获取 front page 条目，只把标题和 URL 放入 `DataItem`，随后基类 `fetch_with_urls()` 使用 Crawl4AI 抓 URL 正文。

这实际上体现了一个重要原则：**发现 URL 时优先使用低成本、结构化、可分页的官方/半官方数据源；真正正文再交给统一内容抓取层。**

对于 1000 个博客，可以对应为：

```text
Sitemap / Feed / API / Archive / Navigation
           ↓
      URL Discovery
           ↓
   HTTP-first Content Fetch
           ↓
必要时 Browser/Crawl4AI 升级
```

这比“从首页开始递归点击所有链接”更可控，也更容易证明历史覆盖。

但 TrendSearch 当前 `fetch_with_urls()` 把通用正文抓取放在 DataSource 基类中，使 Discovery 与 Content Fetch 仍然耦合。知识库平台应将两者彻底拆开：适配器只负责发现和来源语义，Content Fetch 统一执行条件请求、响应存证、路由、重试、限流和质量判定。

## 5. Crawl4AI 使用方式与扩展瓶颈

`fetch_with_urls()` 在一个 `AsyncWebCrawler` context 中遍历 `DataItem`，调用 `crawler.arun()`，并在每项后随机 sleep 1~4 秒。

积极的一面：

- 没有为每个 URL 单独重建浏览器进程；
- Crawl4AI 输出 Markdown，降低了后续处理成本；
- 错误被单项捕获，不会直接终止整个来源。

但其吞吐模型本质仍然是串行：

```text
for item:
    await crawler.arun(item.url)
    await sleep(...)
```

在千站历史回灌时，这种模式无法满足规模要求。随机 sleep 也只解决单进程“别太快”，不能解决多个 Worker 同时访问同一域名的全局礼貌限流。

生产方案需要：

- durable frontier 按小批次 claim；
- domain/site 级 token bucket 和最大并发；
- HTTP Worker 与 Browser Worker 分池；
- Browser 只在 Preflight 有证据时升级；
- 长生命周期 browser runtime，但每个任务结果立即持久化；
- 跨 Worker 公平调度与 backpressure；
- 429/503 按 Retry-After 做域级退避，而不是简单随机 sleep。

## 6. Markdown 清洗：链接不能直接删除

`clear_markdown_url()` 使用正则把：

```text
[text](https://example.com)
```

清理成：

```text
text
```

对“新闻摘要”应用而言这样能减少噪声，但对知识库是不合适的，因为 URL 本身承载：

- 文档之间的真实引用关系；
- 站内导航和“上一篇/下一篇”；
- 外部参考资料；
- 图片/附件来源；
- 后续 broken-link 检测和 link graph；
- provenance 与审计证据。

因此最终方案不应把 Markdown 当唯一真相，更不能在抽取阶段不可逆地删除链接。应先保存 Canonical Document IR 和 `document_link_edge`，Markdown 只是可重建 Projection。若希望面向 LLM 生成“无 URL 文本版”，应作为额外 projection，而不是覆盖源结构。

## 7. 并发流水线：Semaphore 有价值，但 gather 不是 durable queue

`SingleNewsDemandMiningPipeline` 对 LLM 调用使用全局 `asyncio.Semaphore(max_concurrency)`，这是正确的资源保护意识。

但 `run_for_all_sources()` 的执行方式是：

1. `for source in sources`，依次 `await source.fetch()`，来源抓取阶段串行；
2. 把全部已获取新闻积累到内存；
3. 一次创建所有 `_process_single_news()` coroutine；
4. `asyncio.gather(..., return_exceptions=True)`；
5. 全部完成后批量入库。

这种方式适合一次几十/几百条数据的短任务，不适合百万 URL：

- 进程崩溃后 gather 中的任务状态全部丢失；
- 无 lease/heartbeat，无法可靠重领；
- 无 checkpoint，无法知道哪个阶段已真实完成；
- 大批量对象长期驻留内存；
- source fetch 串行造成头阻塞；
- Semaphore 只限制 AI 调用，并没有限制源站抓取、Browser、数据库、PDF/OCR 等不同资源池；
- 无站点公平性，一个超大站可能占满后续队列。

对知识库平台而言，`asyncio.Semaphore` 可以继续作为 **worker-local** 的最后一道保护，但系统级吞吐控制必须建立在 PostgreSQL durable task/frontier、Redis Streams transport、lease、domain quota、runtime class 和 backpressure 之上。

## 8. URL 去重与 SQLite：小系统中正确，但不能等价于增量同步

### 8.1 URL hash 与唯一约束

`storage.py` 使用 SHA256(url) 生成 `url_hash`，`news.url` 与 `url_hash` 都有 UNIQUE，并使用 `INSERT OR IGNORE` 去重。这一点值得保留：**数据库唯一约束才是最终幂等保障，内存 Set/Bloom Filter 只能做加速。**

### 8.2 `url_exists()` 的语义问题

基类抓取前先：

```text
if data_store.url_exists(url):
    skip fetch
```

这把“这个 URL 曾经存过”错误地等价成“这个 URL 永远不需要再抓”。

对于博客知识库，同一个 URL 可能发生：

- 正文修订；
- 标题/作者/标签更新；
- canonical 改变；
- 代码示例修复；
- 文档站同路径内容滚动升级；
- 站点迁移后 redirect。

因此 URL 唯一约束只应该解决 identity/dedup，不应该决定 freshness。正确模型是：

```text
url observation exists
  ≠ content is fresh

known URL
  -> next_check_at
  -> If-None-Match / If-Modified-Since
  -> 304: freshness 更新，不创建新版本
  -> 200 + body/IR 未变: freshness 更新
  -> 200 + IR 实质变化: 创建 document_version candidate
```

Discovery 重复发现同 URL 时也应该 UPSERT `last_seen_at/discovery_evidence`，而不是完全忽略。

### 8.3 SQLite 的边界

TrendSearch 为 SQLite 开启 WAL、索引和 thread-local connection，这是单机应用的合理优化。但千站知识库需要多 Worker、lease、`SKIP LOCKED`、高并发任务 claim、HA 和海量元数据，因此 PostgreSQL 更适合作为业务真相源。

SQLite 可以保留为本地开发/单机演示模式，但不能成为生产调度状态源。

## 9. 30 天自动清理：知识库场景必须反过来设计

`DataStore.__init__()` 会调用 `clean_old_data(days=30)`，删除超过 30 天的 news/demands。

对热点分析产品来说，这符合“只关心近期”的业务语义；对历史知识库来说则是不可接受的，因为系统目标本身就是长期保存历史文章。

因此主方案需要显式的 Retention/GC 模型，而不是用年龄直接删：

```text
AUTHORITATIVE
  accepted document/version
  lineage root snapshot
  quality/drift manifest
  publish manifest
  默认长期/永久保留

RECONSTRUCTIBLE
  raw/fit markdown candidate
  index payload
  可在确认能够从 Snapshot + immutable release 重放后按策略回收

EPHEMERAL
  staging object
  temp render
  transient log
  可 TTL
```

GC 应从 accepted version、法律/人工 hold、publish manifest、run lineage 等根对象做 mark-and-sweep。任何删除都形成 tombstone/audit event。对象存储生命周期只能基于 retention class，而不能用“创建超过 30 天”这种统一规则。

## 10. Pipeline 完成状态：异常被吞掉后不能仍然 completed

`analyze_script.py` 的多个阶段都使用：

```text
try:
    ...
except Exception:
    log error
    continue
```

最后仍会返回总体 `status = completed`。

这种“尽量产出报告”的产品选择对日报系统可以接受，但对知识库同步会造成严重的假完成：Discovery、Fetch、Extraction、Artifact Commit 中任意环节漏掉数据，都可能被 UI 显示成成功。

这正好验证主方案里 `Stage Finalizer` 的必要性。生产完成状态必须由独立 Finalizer 对账：

```text
expected entities
vs persisted facts
vs COMPLETE artifacts
vs retry/dead-letter
vs quality/drift manifests
```

最终只能输出 `COMPLETED / PARTIAL / FAILED / INCONSISTENT`，而不是因为协程返回就 completed。

## 11. Web 管理：交互形态可复用，查询实现需要升级

TrendSearch 提供 Dashboard、新闻列表、详情、需求列表、报告中心、点赞等页面，说明“抓取系统必须可视化运营”是正确方向。

FastAPI `/api/news`、`/api/demands` 也支持 source/keyword/min_score 等过滤。不过当前实现会先把最近 12/24 小时数据读出，然后在 Python 内存中筛选，并直接返回全部结果，不做分页。

百万级知识库不能沿用这种方式，应改为：

- PostgreSQL/OpenSearch 服务端过滤；
- keyset pagination（优先于大 offset）；
- 索引覆盖常用过滤维度；
- 文档正文详情按需加载；
- dashboard 聚合使用预聚合/物化统计；
- WebSocket/SSE 只承载运行进度事件，不把前端连接当任务生命线。

同时应增加 TrendSearch 没有的“Source Adapter Registry”页面，显示：

```text
adapter release
capabilities
binding sites
health/canary
last checkpoint
coverage gap
error rate
cost
rollback target
```

这样新增网站和适配器可以在 Web 中受控发布，而不是修改 import 后整体部署。

## 12. 可复用技术原则与需要舍弃的实现

### 值得复用

1. **Strategy/Adapter 数据源抽象**：来源差异必须被隔离。
2. **结构化 API 优先发现**：发现 URL 不需要先开 Browser。
3. **统一数据模型**：上层 Pipeline 不依赖来源细节。
4. **数据库唯一约束幂等**：URL hash/unique 比纯内存去重可靠。
5. **Semaphore 资源保护**：每种昂贵资源都应有并发预算。
6. **Web 管理与详情诊断**：抓取平台不是只有命令行 Worker。
7. **Pipeline 分阶段**：发现、分析、报告可以独立演进。

### 不应直接照搬

1. `url_exists => skip forever`；
2. DataSource 内直接完成通用网页抓取；
3. 所有来源依次串行抓取；
4. 一次把全部对象塞入 `asyncio.gather`；
5. worker-local random sleep 代替跨 Worker 域级配额；
6. Markdown 清洗时不可逆删除 URL；
7. SQLite 作为大规模多 Worker 状态真相；
8. 默认自动删除 30 天前历史数据；
9. 阶段失败后总体仍直接 completed；
10. Web API 全量读入内存再过滤、无分页。

## 13. 对博客知识库主方案的具体优化结论

基于 TrendSearch，应在主方案中加入/强化以下能力：

### 13.1 Versioned Source Adapter SDK + Capability Registry

把 TrendSearch 的 `BaseDataSource` 进一步产品化，明确适配器 release、能力声明、配置/输出 schema、健康检查、Golden/Canary/rollback，以及 site_source 到 adapter release 的版本化绑定。

### 13.2 Discovery Adapter 与 Content Fetch 强制分离

适配器只输出 DiscoveryRecord/FetchHint；通用 HTTP/Browser/PDF Fetch 必须由统一 Worker 执行，保证条件请求、安全、限流、Artifact、质量与重试语义一致。

### 13.3 去重与 freshness 分离

URL 唯一键只防重复 identity；已存在 URL 仍按 schedule/ETag/Last-Modified/body hash/IR hash 做增量刷新。

### 13.4 显式 Retention/GC

禁止“按年龄清空知识库”。对 authoritative/reconstructible/ephemeral 对象分类，使用 lineage-aware GC。

### 13.5 适配器级隔离、公平性和 backpressure

来源抓取不做单线程串行；也不一次性 gather 全量。用 durable task + 小批 claim + site/domain quota + runtime class 执行。

### 13.6 Web 加入 Adapter/Source Registry 与服务端分页

新增网站时可在管理端查看 adapter capability、binding、checkpoint、coverage、canary 和 rollback；百万级列表必须服务端查询与 keyset pagination。

### 13.7 Finalizer 成为唯一完成裁判

任何 Worker、Pipeline 或异常捕获都无权直接宣告业务完成；完成状态由真实数据库事实和 Artifact 对账得到。

## 14. 最终评价

TrendSearch 的价值不在于它可以直接扩成“1000 博客知识库”，而在于它把一个小规模多源聚合系统的关键工程选择真实地暴露出来：统一适配器、API 发现、Crawl4AI 正文抓取、URL 去重、Pipeline、Web 管理确实能够快速构建可用产品；但同一套实现放大到历史全量、长期增量和百万文档后，URL freshness、持久任务、分布式限流、版本化适配器、数据保留、完成对账和服务端查询会成为决定系统是否可靠的核心。

因此主方案应吸收它的 **Adapter 思想和运营 Web 形态**，同时明确拒绝其 **URL 永久跳过、非 durable gather、统一 30 天清理、进程内完成语义**。这样既保留了新增来源的开发效率，也能保证知识库长期可扩展、可恢复、可审计和不丢历史。