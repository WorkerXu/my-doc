# PicoNewsAgent：新闻自动采集、生成与发布智能体

- 调研编号：53
- 项目地址：https://github.com/HosLak/PicoNewsAgent
- 调研目标：分析其 RSS 聚合、并发抓取、Crawl4AI 深抓、去重、媒体提取、AI 处理与发布流水线的实现细节，并判断哪些设计适合吸收到“1000 个技术博客全量历史 + 增量同步 + Markdown 知识库 + Web 管理”方案中。

## 1. 项目定位与总体结构

PicoNewsAgent 是一个面向 AI 新闻的自动化流水线。入口 `main.py` 顺序执行 5 个阶段：

1. `core/fetcher.py`：从 `Sources.md` 中发现 RSS/Atom Feed，抓取条目元数据。
2. `core/processor.py`：按时间窗口过滤，并做 URL 与标题近似去重。
3. `core/agents.py::select_news()`：调用 LLM 对候选新闻做价值筛选。
4. `core/agents.py::generate_posts()`：对筛选后的文章使用 Crawl4AI 深抓正文和图片，再调用 LLM 生成 Telegram 内容。
5. `core/publisher.py`：发布到 Telegram，并针对消息过长、HTML 解析失败、媒体失败做降级。

这个项目本质上不是“历史知识库爬虫”，而是“Feed 信号发现 -> 少量正文深抓 -> AI 派生内容 -> Delivery”的实时内容流水线。它最值得借鉴的不是直接复制代码，而是其“便宜的元数据发现层”和“昂贵的正文抓取层”分离思想；同时，它的持久化、身份、增量语义和成功判定都不足以直接支撑 1000 站长期知识库。

## 2. Source 与 Feed 发现实现

### 2.1 来源配置

`Sources.md` 直接保存 Feed URL，例如 OpenAI、Google、Hugging Face、AWS、Substack 等 Feed。`fetcher.extract_rss_links()` 读取整个 Markdown 文本，同时用两类正则提取 Markdown 链接和裸 URL，再根据 URL 是否包含 `.xml`、`/rss`、`/feed`、`/atom`、`feed=` 判断其是否像 Feed。

优点：

- 增加来源成本低，改一个文本文件即可；
- 不需要为每个来源写专门代码；
- 适合小规模运营工具快速维护。

局限：

- Source 只是字符串，没有 `site_id/provider_id`、启停状态、调度周期、抓取策略、认证、robots、健康状态、最后成功时间等结构化业务信息；
- 仅凭 URL 关键字识别 Feed，会漏掉没有典型路径的 Feed，也可能误判；
- 无 Provider checkpoint，无法表达“这个 Feed 上次同步到哪个 GUID/updated”；
- 文本文件不能安全承担多用户 Web 管理和并发变更。

因此生产方案应保留“Feed 是通用 Provider”的思路，但把来源从 Markdown 文本升级为数据库中的版本化 Source/Provider Registry。

### 2.2 Feed 网络抓取

`fetch_with_retries()` 使用 `requests.get()`，配置浏览器风格 User-Agent 和 RSS/XML Accept，超时由 `REQUEST_TIMEOUT` 控制。失败重试 `MAX_RETRIES` 次，退避时间为 `2 ** attempt` 秒。

`run()` 使用 `ThreadPoolExecutor(max_workers=MAX_WORKERS)` 并发抓取多个 Feed；默认 `MAX_WORKERS=12`。这说明 Feed 轮询属于轻量 I/O 任务，适合高并发，与 Browser 深抓应使用不同资源池。

对知识库方案的启发：

- Feed/Sitemap/API/普通 HTTP 元数据探测与 Browser Worker 分池；
- Provider 并发之外还需要 host/domain 级限流，避免 1000 站中某个域名占满全局 worker；
- 指数退避应增加 jitter，并结合 `Retry-After`、HTTP 状态码、站点级熔断和失败预算；
- Feed 请求应使用 ETag/If-None-Match、Last-Modified/If-Modified-Since，304 时不重新解析正文候选。

## 3. Feed 条目解析与元数据

`parse_feed()` 使用 `feedparser` 解析 XML，最多读取 `MAX_ITEMS_PER_FEED` 条，默认 50。每个条目输出：

- `source_url`
- `title`
- `link`
- `lead_image`
- `summary`
- `published_date`
- `fetched_at`

### 3.1 图片候选提取

`extract_thumbnail()` 依次尝试：

1. `media_content`
2. `media_thumbnail`
3. entry 的 image 类型 links
4. `image/thumb/thumbnail/wp_post_thumbnail`
5. summary/description/content 中 `<img src>`

并过滤 pixel、analytics、icon、logo、avatar 等明显非正文图。

这是一个很实用的“多来源媒体候选”模式。知识库方案不应只在正文 HTML 中找图片，而应把 Feed enclosure/media、OpenGraph、Twitter Card、JSON-LD、正文 `<img>` 都记录成 `asset_candidate`，并保存 provenance，之后由 Asset Pipeline 下载、去重、校验、排序。

### 3.2 时间语义的关键问题

`normalize_date()` 优先 `published_parsed`，其次 `updated_parsed`；如果都没有，则直接用 `datetime.now()`。

这对新闻展示尚可容忍，但对长期知识库不可接受：抓取时间不能伪装成源站发布时间。正确方案必须将：

- `source_published_at`
- `source_updated_at`
- `discovered_at`
- `fetched_at`
- `observed_at`

严格拆开。源站发布时间未知就保存 NULL，并记录时间字段来自 Feed、JSON-LD、页面 meta、正文推断还是人工 Override。

## 4. 清洗与去重实现

`processor.py` 有两个核心步骤。

### 4.1 时间窗口过滤

默认 `DAYS_BACK=1`，只保留最近一天条目。这个策略非常适合日报，却与“全量历史回灌”目标相反。

知识库方案中时间窗口只能用于 `INCREMENTAL` 的优先级或扫描范围，不能成为 `FULL_BACKFILL` 的历史截断条件。首次回灌必须通过 Sitemap、Archive、API cursor、分页导航等持续枚举，并用 Coverage Ledger 解释终止原因。

### 4.2 运行内去重

项目使用：

- `seen_urls` 做精确 URL 去重；
- 清洗标题后用 `difflib.SequenceMatcher`，阈值默认 0.85，做模糊标题去重。

README 也明确指出当前去重只有单次运行内记忆，没有跨运行持久状态。

生产知识库必须改成持久化、多层 Evidence：

1. normalized URL exact match；
2. redirect/canonical 证据；
3. Feed GUID / API source key；
4. Canonical IR hash 完全内容一致；
5. 标题/正文 SimHash、MinHash/LSH 或其他近似相似度作为“可能重复”证据。

尤其不能直接把“标题像”当成删除依据。技术博客里“Release 1.2”“Release 1.3”、系列文章、翻译/镜像都可能高度相似。近似重复应生成 `similarity_evidence` / `duplicate_cluster`，用于搜索去重、运营提示和镜像识别，但保留每个来源文档及其历史版本。

## 5. AI 筛选阶段的适用边界

`select_news()` 会把条目的标题、摘要、链接、发布时间、图片简化后批量交给 Gemini，并要求返回 JSON 形式的精选结果。

这个步骤对“编辑精选”合理，但绝不能放在知识库主同步链路的 Admission 之前：

- LLM 评分不稳定；
- 模型不可用会阻断内容；
- 相关性不是历史完整性的证明；
- 低价值文章今天可能无价值，未来知识检索可能正需要它。

因此正确关系应是：

```text
源站同步 -> Accepted Document Version -> Search/Markdown Ready
                                  -> 异步 AI Summary/Tag/Cluster/Digest
```

AI 只能影响下游派生内容、优先级和人工工作队列，不能静默裁掉 FULL_BACKFILL 的合法文章。

## 6. Crawl4AI 深抓实现

`crawl_single()` 创建 `BrowserConfig(headless=True)` 和 `CrawlerRunConfig`，关键配置包括：

- `wait_for="body"`
- `wait_until="networkidle"`
- `wait_for_images=True`
- `remove_overlay_elements=True`
- `delay_before_return_html=2.0`

失败按 `CRAWL_RETRIES` 重试，默认 3 次、间隔 4 秒。

### 6.1 并发控制

`crawl_all_parallel()` 使用 `asyncio.Semaphore(MAX_CONCURRENT_CRAWLS)`，默认同时最多 3 个深抓任务。这种“轻量 Feed 高并发 + Browser 低并发”的资源分层是正确方向。

但 `crawl_single()` 每个 URL 都执行一次：

```python
async with AsyncWebCrawler(config=browser_cfg) as crawler:
```

也就是抓一篇文章就创建/销毁一个 crawler 生命周期。小批量新闻可以工作，但数十万/百万文章会产生明显 Browser 初始化开销。

生产方案应使用长期 Browser Runtime/Context Pool：

- Worker 进程长期持有浏览器；
- 每站或每安全域使用隔离 Context；
- Page 生命周期短，Browser 生命周期长；
- 设置 Browser pool 总内存/CPU 上限；
- 使用 host 级 semaphore；
- Browser 只作为 HTTP 抓取与抽取质量失败后的升级路径，而不是默认路径。

### 6.2 正文处理问题

抓取成功后优先取 `result.markdown`，否则取 `result.html`；随后 `clean_content()` 用正则删 script/style/nav/footer/header、去标签、压空白，最后硬截断到 2500 字符。

这适合给 LLM 写短帖，却完全不适合知识库：

- 代码块、表格、标题层级和链接结构被抹掉；
- 长技术文章被硬截断；
- Markdown 经过“HTML 正则清洗”会损失语义；
- 无法稳定重放、diff 或重新切块。

生产方案必须保存不可变 Snapshot，并建立：

```text
Snapshot
 -> Extraction Candidate
 -> structured block IR
 -> Canonical Document IR
 -> deterministic Markdown Projection
```

Canonical IR 保留 heading、paragraph、code、table、quote、list、image、link 等结构块；Markdown 只是可重建投影。

### 6.3 图片处理

项目从 `result.metadata` 读取 `og:image/twitter:image`，再遍历 `result.media.images`，过滤 avatar/logo/icon/ad/banner，并保留少量图片。

可借鉴其“metadata + media”多通道候选，但生产方案还需要：

- 图片 URL 归一化和相对地址解析；
- 下载到对象存储；
- MIME/尺寸/大小/恶意文件检查；
- SHA-256 去重；
- 保留原 URL、最终 URL、referer、发现位置、alt、caption、宽高；
- Markdown Projection 按策略改写为本地资产 URL。

## 7. 状态管理与成功语义

项目各阶段通过 `data/*.json` 文件传递：raw -> clean -> selected -> telegram posts。`main.py` 顺序调用函数，没有数据库任务状态、lease、outbox、manifest 或 stage finalizer。

风险包括：

- 进程中途退出只能从文件状态人工判断；
- 多实例同时执行容易覆盖 JSON；
- 单个阶段内部错误常被打印后 return，主流程仍可继续；
- 最终“Pipeline Completed”并不等于每篇文章都成功；
- 无跨运行幂等键；
- 无法回答“这个站漏了多少、失败在哪一步、哪个结果由哪个规则版本产生”。

1000 站方案应坚持：PostgreSQL 持久化决定性状态，S3/MinIO 保存大对象，Redis Streams 只传输任务；每阶段输出结构化 Outcome，Finalizer 根据 Run Manifest 判断真正完成。

## 8. Publisher 的降级模式

`publisher.py` 根据图片数量选择 Telegram `sendMessage/sendPhoto/sendMediaGroup`。如果失败：

1. 消息过长时调用 Shortener Agent，再重试；
2. HTML 解析失败或媒体失败时降级为纯文本/无图消息。

这个设计体现了一个重要原则：下游 Delivery 可以有自己的 fallback，但不能回写或污染源文档真相。

知识库方案应将 Markdown、Search、Embedding、摘要、Digest、外部发布全部视为 Accepted Document Version 的独立 Projection / Derived Workflow。某个 Delivery 失败只产生独立 `delivery_attempt` 和 dead-letter，不应让源站同步回滚。

## 9. 对现有博客知识库方案的具体优化结论

本次调研后建议在现有方案中显式强化以下能力。

### 9.1 增加 Feed Change-Signal Plane

虽然现有方案已经支持 RSS/Atom/JSON Feed，但应把 Feed 从普通 Discovery Provider 再提升为“低成本变化信号层”。引入 `provider_item_observation`：

```text
provider_id
provider_item_key        # GUID 优先，否则规范化 link + source evidence
url_candidate_id
source_published_at
source_updated_at
summary_hash
media_candidates
observed_at
raw_item_snapshot_ref
```

并保存 Provider 的 ETag、Last-Modified、最近成功轮询、最近 GUID/更新时间高水位。

Feed 的职责是快速发现“新增/可能更新”，触发 Article Fetch；它不能单独证明历史完整性，因为 Feed 通常只保留有限窗口。

### 9.2 明确 Metadata Intake 与 Article Fetch 两级队列

采用：

```text
Feed/Sitemap/API metadata intake
 -> cheap candidate normalization
 -> identity/freshness decision
 -> only changed/new candidate enters article fetch
```

这样 1000 站每日频繁轮询时，大部分 unchanged 信号在低成本层就被短路，不进入 Browser/正文抽取。

### 9.3 增加 Similarity Evidence / Duplicate Cluster

在 URL/Document Identity 之外增加非破坏性的相似内容关系：

```text
content_similarity_evidence
story_or_duplicate_cluster
cluster_membership
```

使用规范化标题、正文 SimHash/MinHash 等生成候选；精确 Canonical IR hash 可以判定内容完全一致，但近似相似只用于聚类，不自动删除文档。

这对转载、镜像、同步到多个博客平台、Newsletter 重发特别有价值，并可改善搜索结果多样性。

### 9.4 增加媒体候选 provenance

把 Feed media、OpenGraph、Twitter Card、JSON-LD、正文 media 全部统一为 `asset_candidate`，记录发现方式和置信度，再由 Asset Pipeline 决定下载与主图选择。

### 9.5 Browser Pool 必须是长期资源池

现有方案已有 HTTP-first / Browser fallback 原则，本次进一步明确：禁止生产代码按 URL 创建完整 Browser Runtime；Worker 应长期复用 Browser，按任务创建 Page/Context，并做 host 级隔离与资源预算。

### 9.6 人工审核是状态机，不是“加一个页面”

PicoNewsAgent Roadmap 提到 Human-in-the-Loop Dashboard。知识库方案中应把它实现为持久状态：

```text
AUTO_ACCEPTED
NEEDS_REVIEW
HUMAN_ACCEPTED
HUMAN_REJECTED
OVERRIDDEN
```

审核动作生成 append-only audit/override evidence。Web 页面只是这个状态机的操作界面。

## 10. 不应直接照搬的设计

1. 用 Markdown 文件保存 Source Registry。
2. 仅凭 Feed URL 关键字判断 Provider 类型。
3. 发布时间缺失时用当前时间代替。
4. `DAYS_BACK` 作为主采集过滤条件。
5. 单次进程内 `set` 和 `SequenceMatcher` 承担持久去重。
6. LLM 选择结果决定哪些合法文章进入知识库。
7. 每 URL 新建一个 Crawl4AI Browser 生命周期。
8. 正文统一清洗成纯文本并截断 2500 字符。
9. JSON 文件串联生产阶段和保存业务状态。
10. 只靠函数返回/打印日志判断流水线完成。
11. 发布失败与源内容处理强耦合。

## 11. 最终评价

PicoNewsAgent 对“1000 个技术博客知识库”最大的价值是验证了一个合理的资源分层模型：Feed 负责便宜地发现候选，正文深抓只处理需要的 URL，Browser 并发需要严格受控，图片应从多个元数据通道联合发现，最终 Delivery 可以独立做降级。

但它同时展示了小型自动化脚本在扩大规模后会遇到的典型边界：文件状态、运行内去重、日期伪填、Browser 生命周期过细、正文截断、LLM gating、无 durable workflow。生产知识库方案应吸收其轻重分层与媒体候选思想，同时用版本化 Provider、持久 Frontier、不可变 Snapshot、Canonical IR、Feed Change-Signal Plane、Similarity Evidence、Outcome/Finalizer 和 Web 审核状态机补足长期运行能力。
