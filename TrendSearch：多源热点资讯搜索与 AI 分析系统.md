# TrendSearch：多源热点资讯搜索与 AI 分析系统

项目地址：https://github.com/LIhong42/TrendSearch

调研结论：**可借鉴其“统一数据源接口、聚合型上游数据源、分层 AI 分析、Embedding 多样性排序、Web 反馈和 Channel Adapter”设计，但不能直接作为 1000 个技术博客全量历史知识库抓取底座。** TrendSearch 面向的是“近期热点聚合与需求挖掘”，核心数据窗口、去重语义、存储保留、抓取完整性和任务可靠性都与长期博客知识库不同。最值得吸收的能力应放在 Source Adapter 开发体验、Aggregator Discovery Provider、Derived AI Pipeline、Feedback Signal Projection 和 Web 运营层；其 URL 存在即跳过、30 天自动删除、SQLite、进程内队列、反射式注册和直接删除 Markdown 链接等做法不应进入权威抓取链路。

## 1. 项目定位与总体结构

TrendSearch 是一个“多源热点抓取 -> 正文补全 -> LLM 单条分析 -> 分组分析 -> 全局分析 -> 评分/排序 -> 报告 -> Web 展示/飞书推送”的应用。

README 给出的技术栈为：

- React 18 + TypeScript + Ant Design；
- FastAPI；
- LangChain/OpenAI 兼容模型；
- SQLite；
- Crawl4AI + aiohttp；
- 飞书开放平台。

后端代码大致拆分为：

```text
backend/app/
  data_sources/      数据源适配
  pipeline/          分析流水线
  agent_framework/   Agent、Prompt、模型配置、工具/中间件注册
  storage/           SQLite 存储
  router/            FastAPI API
  channels/          飞书与消息总线
  embedding/         Embedding
  analyze_script.py  端到端编排入口
```

这个拆分说明项目已经有“Source -> Pipeline -> Storage -> API/Channel”的边界意识，但它仍属于单机应用级结构，而不是具有 durable frontier、不可变 Snapshot、版本模型、Coverage Ledger、分布式任务和恢复语义的抓取平台。

## 2. 数据源抽象：策略模式的优点与边界

### 2.1 `BaseDataSource` + `DataItem`

`backend/app/data_sources/base.py` 定义：

```python
class DataItem(BaseModel):
    title: Optional[str]
    url: Optional[str]
    content: Optional[str]
    source: Optional[str]
    summary: Optional[str]

class BaseDataSource(ABC):
    @abstractmethod
    async def fetch(self, **kwargs) -> list[DataItem]:
        ...
```

这实际上是一个轻量 Source Adapter SDK：不同来源只需要实现 `fetch()` 并统一返回 `DataItem`。知乎、B 站、GitHub、HackerNews、财联社等来源都在这个协议上工作。

对博客知识库的启发是：**适配器必须有一个小而稳定的接口**。但生产协议不应只返回标题、URL、正文、摘要，而应至少返回 `DiscoveryRecord`，携带：

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

TrendSearch 的接口把“发现 URL”和“抓取正文”放在同一个 Source 类中，适合小项目，但长期知识库应该继续坚持：

```text
Source Adapter / Discovery Provider
            ↓
       DiscoveryRecord
            ↓
      URL Admission
            ↓
       Content Fetch
```

这样 Adapter 不会绕过统一的 robots、Scope、SSRF、限流、Snapshot、重试和质量链路。

### 2.2 反射式自动注册

`backend/app/data_sources/__init__.py` 使用：

```python
return {cls.__name__: cls() for cls in BaseDataSource.__subclasses__()}
```

优点是开发体验很好：新增一个子类后几乎零配置即可被系统发现。

缺点是 import 顺序、运行时模块装载和代码版本会隐式决定“当前有哪些 Adapter”，无法可靠回答：

- 某次历史 Run 到底用了哪个 Adapter 版本；
- 配置变更发生在什么时候；
- 哪个 Adapter Release 已经通过 Golden/Canary；
- 如何回滚；
- 同名实现和依赖镜像发生变化后如何复现。

因此推荐保留两层：

```text
开发态：ABC / Protocol + entrypoint/反射自动发现，提高写 Adapter 的效率
生产态：显式 Manifest + source_adapter_release + adapter_binding_release，作为唯一业务真相
```

反射可以成为“生成待发布 Manifest”的工具，但不能成为运行中 Registry 的事实来源。

## 3. 聚合型上游 API：值得抽象成 Aggregator Discovery Provider

TrendSearch 多个来源并不是直接从目标平台枚举热点，而是通过 `fetch_data_with_news_now()` 请求环境变量 `NEWS_API_URL` 指向的聚合 API：

```text
NEWS_API_URL?id=<source>&latest
```

然后各个 Source 类把聚合 API 的 `items` 映射为 `DataItem`，再按 URL 抓正文。

这是一种很实用的“两级发现”结构：

```text
Aggregator API
   ↓ 只给候选 URL/标题/热度等
Source-specific mapping
   ↓
Origin Content Fetch
```

对博客知识库可以新增 `AGGREGATOR_API` Provider，支持：

- 第三方目录/聚合站；
- GitHub/社区维护的博客列表；
- NewsNow 类热门源；
- 搜索 API；
- 企业内部 URL 清单 API。

但必须明确：**Aggregator 只能提供发现证据，不能证明历史覆盖，也不能成为正文事实来源。**

具体约束：

1. Aggregator 返回 URL 后仍需 URL Admission；
2. 正文仍由 origin fetch 获取并保存 Snapshot；
3. Aggregator 的 item ID、时间、排名、payload 作为 `discovery_evidence`；
4. 聚合源掉数据不能删除已有文档；
5. 不能因为 Aggregator 只给最近 N 条就把 FULL_BACKFILL 判为完整；
6. Aggregator Provider 必须有独立 checkpoint、健康状态和覆盖说明。

## 4. Crawl4AI 正文抓取实现与不适用点

`BaseDataSource.fetch_with_urls()` 的主流程是：

```text
创建 BrowserConfig/CrawlerRunConfig
  -> 启动 AsyncWebCrawler
  -> 逐条遍历 DataItem
  -> 先 url_exists(url)
  -> 不存在则 crawler.arun(... magic=True ...)
  -> result.markdown
  -> clear_markdown_url()
  -> 随机 sleep 1~4 秒
```

### 4.1 可取之处

- Crawl4AI 生命周期覆盖一批 URL，而不是每个 URL 重建进程；
- 对单 URL 异常做局部捕获，不直接终止全部；
- 有随机 sleep，说明作者意识到不要无限制打源站；
- 抓取结果直接进入 Markdown，适合快速验证原型。

### 4.2 不适合作为知识库主链路的地方

#### URL 已存在就跳过

代码先执行：

```python
if self.data_store.url_exists(item.url):
    continue
```

这把“identity 已知”和“内容仍然新鲜”混为一谈。博客文章会修订，文档站更会持续更新；URL 一旦入库便永久跳过会让增量同步失效。

正确语义应是：

```text
URL 已知
  -> 更新 last_seen/discovery evidence
  -> 根据 freshness policy 判断是否 revalidate
  -> ETag/Last-Modified/lastmod/body hash/IR hash 判定变化
  -> 有实质变化才创建新 document_version
```

#### 直接删除 Markdown URL

`clear_markdown_url()` 将 `[text](https://...)` 替换为纯 `text`。对新闻摘要可能可接受，但知识库会因此永久丢失引用、外链、文档内部导航和 Link Graph 证据。

主链路必须保留真实链接。若 AI 分析需要“去 URL 的纯文本”，应额外产生一个 Projection：

```text
Canonical Document IR
  ├─ FINAL_MARKDOWN（保留链接）
  └─ LLM_PLAIN_TEXT（可去 URL）
```

#### 所有正文默认走 Browser/Crawl4AI

TrendSearch 面向少量热点条目，直接 Browser 的成本尚可；1000 个站点的全历史抓取必须 HTTP-first，再根据 `CSR_SHELL/EMPTY_BODY/LOAD_MORE` 等证据升级 Browser。

## 5. 并发模型：LLM 有 Semaphore，但抓源仍偏串行

`SingleNewsDemandMiningPipeline` 的 `run_for_all_sources()` 分成两个阶段。

第一阶段逐来源：

```python
for source_name in self.data_source.keys():
    news_items = await self.data_source[source_name].fetch(...)
```

所有数据源抓取是串行的。

第二阶段将新闻分析任务一次性建立：

```python
tasks = [self._process_single_news(news) for news in all_fetched_news]
results = await asyncio.gather(*tasks, return_exceptions=True)
```

而 `_process_single_news()` 内使用：

```python
async with self.semaphore:
    await agent.ainvoke(...)
```

这说明项目正确地区分了“任务数量”和“LLM 并发额度”，但仍有两个生产问题：

1. `asyncio.gather()` 一次性创建全量 task，只适用于数量可控的数据集；
2. Semaphore 是进程内额度，无法表达跨 Worker/跨站点公平性和 durable checkpoint。

博客知识库继续采用：

```text
PostgreSQL durable task/frontier
 -> claim 小批 20~100
 -> Redis Streams 运输
 -> worker-local Semaphore
 -> site/domain token bucket
 -> lease/heartbeat/retry/dead-letter
```

也就是说，TrendSearch 的 Semaphore 思想只适合保留为“worker-local 最后一层并发闸门”。

## 6. SQLite 存储、URL Hash 与批量写入

`storage.py` 有几个值得注意的实现细节。

### 6.1 SQLite WAL + Thread Local Connection

它对每个线程复用 SQLite connection，并设置：

```sql
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;
```

这是单机 SQLite 中比较合理的并发优化，但不适合作为百万文档、多 Worker 的中心状态库。

博客知识库仍应使用 PostgreSQL。

### 6.2 URL SHA256 + 唯一约束

项目同时保存 `url` 和 `url_hash=SHA256(url)`，并对 URL/hash 建唯一索引，用 hash 快速判断 URL 是否存在。

这可以吸收为性能优化，但必须建立在“规范化 URL identity”之后：

```text
raw_url
 -> normalized_url(versioned normalization rules)
 -> normalized_url_hash
```

Hash 只是索引加速，不应替代真实 normalized URL，也不能承担 canonical/document identity。

### 6.3 `INSERT OR IGNORE` 与 `executemany`

项目通过 `INSERT OR IGNORE` 避免先 SELECT 再 INSERT，并使用 `executemany()` 做批量写入，减少 Python/SQLite 往返。

生产 PostgreSQL 中可对应为：

```sql
INSERT ... VALUES (...), (...)
ON CONFLICT (...) DO UPDATE/NOTHING
RETURNING ...
```

但“冲突后做什么”必须按领域语义区分：

- URL Observation 冲突：更新 `last_seen_at` 和 discovery evidence；
- document identity 冲突：增加 evidence，不覆盖版本；
- task idempotency 冲突：返回已有任务；
- accepted version：append-only，不能 `DO UPDATE` 覆盖旧事实。

### 6.4 30 天自动清理与长期知识库冲突

`DataStore.__init__()` 会调用：

```python
self.clean_old_data(days=30)
```

`clean_old_data()` 直接删除 30 天以前的 news/demands。

对于热点分析这很自然，但对历史知识库是明确反模式。权威 Snapshot、accepted document/version、quality lineage 默认应长期保留；GC 必须基于 retention class 和引用图，而不是创建时间。

## 7. AI Agent 框架：配置驱动、Prompt 外置与分层分析

### 7.1 Agent 配置驱动

`AgentConfigManager` 从 `agents_config.yaml` 读取：

- model provider；
- model name；
- temperature；
- max_tokens；
- JSON response mode；
- tools；
- middleware。

模型密钥通过环境变量解析，并支持 OpenAI-compatible provider。`AgentBase` 再组合：

```text
AgentConfigManager
PromptManager
ToolRegistry
MiddlewareRegistry
LangChain create_agent()
```

这个分层对 Derived AI 平面有价值：模型、Prompt、Schema、Tool、Middleware 不应硬编码在业务代码里。

博客知识库应进一步把这些配置做成不可变 release：

```text
model_provider_release
prompt_release
schema_release
toolset_release
runtime_release
```

每个 `derived_job` 记录完整 release identity 和 `input_hash`，从而支持离线重放和结果比较。

### 7.2 多层级分析流水线

TrendSearch 的编排顺序是：

```text
阶段 1：单条新闻分析 / 需求挖掘
阶段 2：部分新闻分组分析
阶段 3：全量新闻综合分析
阶段 4：需求评分 + 新闻筛选
阶段 5：新闻/需求报告
阶段 6：飞书投递
```

这比“一次把所有文档塞进超长 Prompt”更合理，核心原理是分层压缩上下文：

```text
Document-level facts
  -> Group-level synthesis
  -> Window/corpus-level synthesis
  -> Ranking / Report
```

这非常适合博客知识库的下游 AI：

- 单文档摘要/主题/实体；
- 同主题或同时间窗聚类；
- 周/月技术趋势；
- 站点级 digest；
- 跨站点综合报告。

需要强调：这些都必须从 accepted document version 出发，不能反过来决定“源正文是否抓取成功”。

## 8. Embedding 多样性排序：Farthest-First / Max-Min 的价值

`BasePipeline.merge_lists_with_min_max_sim()` 实现了一个有价值的多样性排序算法。

大意是：

1. 对已有 `list1` 和候选 `list2` 的 Embedding 做 L2 归一化；
2. 对每个候选，计算它与当前已选集合的“最大相似度”；
3. 每轮选择这个“最大相似度最小”的候选；
4. 新选入一个元素后，只增量更新所有候选与它的相似度；
5. 重复直到排序完成。

其目标不是找“最像”的文档，而是优先补充当前集合最缺失的语义方向，本质接近 farthest-first traversal / max-min diversity。

对博客知识库的下游报告非常有用：如果仅按热度或向量中心性选 100 篇，结果容易被高度重复的话题占满。可以把“相关性”和“覆盖度”分成两层：

```text
Candidate Filter: 时间窗/站点/主题/质量/用户偏好
  -> Relevance Score
  -> Diversity Selector(MMR 或 max-min)
  -> Bounded Context Set
  -> Group/Global LLM Analysis
```

必须保存 Selection Manifest：

```text
selection_run_id
candidate_set_hash
selected_document_version_ids
ranking_release_id
diversity_algorithm/params
feedback_profile_release_id
created_at
```

这样报告能够解释“为什么这批文章被选中”。

## 9. Web API 与用户反馈：从“点赞”升级为 Feedback Signal Projection

TrendSearch 的 FastAPI 提供：

- 新闻列表/详情；
- 需求列表/详情；
- 新闻报告/需求报告；
- stats；
- 新闻/需求点赞。

前端把“浏览 + 搜索/筛选 + 报告中心 + 点赞”做成闭环。README 还明确说明点赞会影响后续需求挖掘和个性化推荐。

这个思路值得加入知识库，但不能把 `is_liked` 直接混在源文档事实里。推荐改成 append-only 事件：

```text
curation_event
  operation = LIKE/FAVORITE/PIN/TAG/REJECT/QUALITY_FEEDBACK
```

再异步投影为：

```text
preference_signal_projection
  actor/team
  topic/site/tag/entity
  positive_weight/negative_weight
  decay_policy
  source_event_watermark
  release_id
```

Derived AI 的 selection/ranking 可以引用某个 `feedback_profile_release_id`，但：

- 不允许据此跳过历史抓取；
- 不允许修改 accepted version；
- 不允许覆盖原始 metadata；
- 不允许把个人兴趣当成“质量真相”。

这样既保留 TrendSearch 的个性化价值，又保持知识库事实层纯净。

## 10. 消息总线与飞书：概念正确，可靠性不足

`channels/bus/queue.py` 使用两个 `asyncio.Queue`：

```text
inbound
outbound
```

用于解耦 Channel 与核心逻辑。这个抽象方向正确，说明“消息通道”和“分析逻辑”应该解耦。

但进程内 `asyncio.Queue` 在进程退出后全部丢失；没有 ack、lease、retry、dedup 和 dead-letter，不能承担生产投递事实。

`analyze_script.py` 在报告生成后直接调用飞书发送。如果飞书失败，代码只返回失败结果；也没有持久化 Delivery 状态。

博客知识库现有的 Notification/Delivery Plane 更适合：

```text
Domain Event
 -> notification_delivery(PENDING)
 -> Transactional Outbox
 -> Notification Worker
 -> Feishu/Slack/Email/Webhook Adapter
 -> SENT/RETRY/DEAD
```

TrendSearch 的 Channel Adapter 可以作为协议层参考，但 transport 必须 durable。

## 11. Web 管理能力的差距

TrendSearch 的 Web 是内容消费/分析 Web，不是抓取控制面。

目前 API 主要处理近期数据展示，甚至有“12 小时无数据再查 24 小时”的逻辑，列表中部分接口会直接把当前窗口全部数据装入 Python 后过滤/排序。

对于百万级博客知识库，需要的是：

```text
站点接入 / Probe
Adapter Registry
Provider Coverage
Run / Task / Dead-letter
URL 诊断
HTTP/Browser Route
Snapshot / Candidate / IR / Quality
Drift
Version Diff
Retention / GC
搜索
Curation / Feedback
Derived AI / Report
Notification Delivery
```

查询必须使用数据库/OpenSearch 服务端过滤和 keyset/cursor pagination，不能复制 TrendSearch 的“小窗口全量加载”方式。

## 12. 建议吸收到博客知识库方案中的能力

### 12.1 Source Adapter 开发体验层

新增明确约定：

```text
Python ABC/Protocol + typed DiscoveryRecord
本地可用 entrypoint/反射自动发现
生成 Adapter Manifest 草稿
Contract/Golden/Checkpoint Test
Canary
发布 source_adapter_release
生产只从 Registry 加载已发布 Release
```

这样兼顾 TrendSearch 的扩展效率和生产可追溯性。

### 12.2 Aggregator Discovery Provider

增加 `AGGREGATOR_API` 能力类型，专门接入“聚合 API/目录/搜索服务给 URL”的来源，严格限制为 discovery evidence。

### 12.3 Hierarchical Derived AI Pipeline

把 Derived AI 从简单的“每文档一个 Job”扩展为：

```text
L0 Deterministic Features / Embedding
L1 Document Analysis
L2 Cluster/Batch Analysis
L3 Window/Corpus Synthesis
L4 Ranking / Personalized Digest / Report
```

每层输入都引用具体 accepted version 和上游 artifact，失败独立重试。

### 12.4 Diversity Selector

在报告/摘要/人工 Review Queue 中加入 MMR/max-min/farthest-first 选择器，避免高重复内容占满上下文预算。

### 12.5 Feedback Signal Projection

从 append-only `curation_event` 生成可版本化偏好画像，供 Derived AI 排序/报告使用；画像只影响“看什么、怎么排”，不影响“抓不抓、事实是什么”。

### 12.6 批量 UPSERT

吸收 URL hash + 批量写的性能思路，在 PostgreSQL 中使用批量 `INSERT ... ON CONFLICT`，但每个领域对象定义不同冲突语义。

## 13. 明确不应吸收的实现

以下做法不应进入长期知识库主链路：

1. **URL 已存在即永久不抓**：破坏 freshness 和增量更新；
2. **30 天自动删除权威数据**：破坏历史知识库；
3. **抽取后直接删除 Markdown 链接**：破坏 Link Graph 和可追溯性；
4. **SQLite 作为中心业务真相**：无法支撑多 Worker 与百万级长期状态；
5. **反射式 `__subclasses__()` 作为生产 Registry**：无法版本化和回放；
6. **进程内 `asyncio.Queue` 作为可靠消息系统**：崩溃即丢状态；
7. **一次性 `asyncio.gather()` 全量任务**：规模增大后产生内存和恢复问题；
8. **所有正文优先 Browser**：成本高且不必要；
9. **聚合 API 的近期列表当历史覆盖**：无法证明 FULL_BACKFILL；
10. **报告投递与主任务同步耦合**：外部 Channel 故障会扩大故障域；
11. **Web 小时间窗全量加载再 Python 过滤**：无法扩展到百万级；
12. **LLM 输出直接承担源数据质量判定**：AI 幻觉不能成为抓取正确性的证明。

## 14. 对现有博客知识库技术方案的具体优化

本次建议在现有方案上补充以下设计，而不是替换其核心架构：

1. `Source Adapter SDK` 增加“开发态自动发现、生产态 Manifest/Release”的双层机制；
2. Discovery Provider 增加 `AGGREGATOR_API` 类型；
3. `curation_event` 增加 Signal Projector，生成版本化 `preference_signal_projection`；
4. Derived AI 增加 Hierarchical Analysis Run；
5. Derived AI Selection 增加 MMR/max-min 多样性策略和 `selection_manifest`；
6. 报告中心保存输入集合 hash、选中文档 version、Prompt/Model release 和偏好 profile release；
7. 数据模型新增 `analysis_run`、`derived_selection_manifest`、`preference_signal_projection`；
8. 测试增加偏好投影可重放、Selection 可解释、分层 AI 上游 lineage 完整性；
9. 验收要求用户 Like/Favorite 只影响派生排序，不改变源抓取覆盖或 accepted truth。

## 15. 推荐的派生 AI 数据流

```text
Accepted Document Version
  -> Deterministic Feature Job
      -> Embedding / Topic / Entity
  -> Document Analysis Job
      -> Summary / Key Points / Trend Signals

Time Window / Query / Collection
  -> Candidate Set
  -> Relevance Rank
  -> Feedback Profile Adjustment
  -> Diversity Selector(MMR / Max-Min)
  -> Selection Manifest
  -> Cluster/Batch Analysis
  -> Corpus Synthesis
  -> Report Artifact
  -> Notification Delivery
```

这个结构把 TrendSearch 最有价值的“单条 -> 分组 -> 全局 -> 排序 -> 报告”思想转化为可追溯、可重放、可控制成本的下游能力。

## 16. 最终评价

TrendSearch 对本项目最大的价值并不在“如何抓网页”，而在于它展示了三个很实用的应用层模式：

1. **同一协议接多来源，降低扩展成本；**
2. **分层 AI 分析和 Embedding 多样性排序，降低上下文冗余；**
3. **Web 点赞/偏好 -> 后续排序/分析，形成反馈闭环。**

但其抓取、状态和存储语义是典型的“近期热点应用”语义，不具备长期知识库所需的历史完整性、Freshness、不可变版本、可靠任务、质量门禁和可恢复性。

因此最终落地应坚持现有的 PostgreSQL + S3 + durable frontier + 多 Provider + HTTP-first + Snapshot/IR/Version + Quality/Drift + Retention/GC 主架构，同时把 TrendSearch 的 Adapter 开发体验、Aggregator Provider、Hierarchical Derived AI、Diversity Selection、Feedback Signal Projection 和 Report Center 作为增强能力并入。