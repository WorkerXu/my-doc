# Finviz 新闻爬虫：实现细节与技术原理分析

- 编号：52
- 项目：finviz-crawler
- 地址：https://github.com/Ezio0/finviz-crawler
- 调研基线：main 分支，最新可见提交 `f34e1832c06c176c1c8398d3d2a1653a311fd308`（v3.5）
- 状态：调研中

## 1. 项目定位

`finviz-crawler` 是一个面向金融新闻的持续采集守护进程。它不是通用网站归档器，而是把一个固定入口 Finviz、若干 RSS 源、第三方文章正文抓取、SQLite 存储、Markdown 文件落盘、主题分类、近似去重、过期清理和本地全文检索组合在一个单机 Python 进程中。

它对“1000 个技术博客全量历史 + 长期增量同步”的价值不在于直接复用代码，而在于展示了几个生产上很实际的问题：不同来源需要不同抓取通道、Browser 会泄漏资源、持续任务需要自愈和优雅退出、增量数据需要去重、全文搜索需要独立索引、失败需要重试和清理。与此同时，它也暴露了单机轮询架构、标题去重、固定 TTL、粗粒度 Browser 看门狗等方案在大规模知识库场景下的边界。

## 2. 实际架构与数据流

项目 README 给出的主流程是：

1. 周期性抓取 Finviz 新闻标题；
2. 拉取 Bloomberg、CNBC、MarketWatch、Yahoo Finance 等 RSS；
3. 将新条目写入 SQLite；
4. 对等待正文的文章使用 Crawl4AI/Playwright 抓正文；
5. 做主题分类、近似去重、失败重试和过期清理；
6. 将正文保存为 Markdown 文件；
7. 周期性重建 SQLite FTS5 索引；
8. Browser 每若干轮重启，并通过 `psutil` 监控 Chromium 子进程数量和 RSS 内存。

v3.5 又增加了一个轻量抓取通道：对 Finviz 页面优先使用 `curl_cffi`，失败后再回退 Crawl4AI；第三方正文仍由 Crawl4AI 负责。依赖只有 `crawl4ai`、`curl_cffi`、`feedparser`、`psutil`，说明项目刻意保持单机部署和低运维成本。

这个设计本质上是“来源发现 + 轻量抓取 + 重型 Browser 抓取 + 本地状态库 + 本地正文文件 + 本地检索”的一体化实现。

## 3. 关键实现细节

### 3.1 多通道抓取：轻量 HTTP 与 Browser 分层

代码对 Finviz 主新闻页和 ticker 页面优先走 `curl_cffi`，并设置浏览器指纹模拟；如果轻量通道失败，再回退 Crawl4AI。对第三方文章则直接调用 Crawl4AI，通过 Playwright 渲染并获取 Markdown 或 HTML。

这一点的核心原理是“不要把 Browser 当默认传输层”。Browser 的 CPU、内存、文件描述符、共享内存和进程数成本远高于普通 HTTP；大量页面其实通过 HTTP、RSS、Sitemap 就足够。一个可扩展采集系统应该把抓取器设计成策略链：

`Feed/Sitemap -> HTTP -> JS Render -> Manual/Blocked`

每次升级都必须有可观测原因，例如 `empty_body`、`js_required`、`challenge_page`、`selector_miss`，而不是所有失败都盲目升级 Browser。

需要强调的是，项目 v3.5 的 `curl_cffi` 主要用于绕过 Cloudflare challenge。博客知识库方案不应把“绕过站点明确访问控制”作为生产能力。可借鉴的是“多 transport adapter + 失败回退”的架构，不应直接照搬反爬绕过策略。所有 transport 都应统一经过 robots、站点政策、访问频率和 Egress Policy 判断。

### 3.2 RSS 与网页正文的混合采集

项目内置多组 RSS。RSS 条目进入数据库时，如果带 summary，会直接保存 Markdown，并把条目标记为 `done`。这样可以在正文抓不到时保留一个可查询的最小内容版本。

这个思路对博客知识库有价值：Feed 是低成本、高信号的增量发现源，应该优先使用。但不能把 RSS summary 与完整正文视为同一种内容。生产方案中应显式区分：

- `content_completeness=full`
- `content_completeness=summary_only`
- `content_completeness=metadata_only`

并且保留 `discovered_via=feed`、`fetched_via=http/browser`。否则 summary 被当作完整文章后，会污染全文检索、向量索引和后续 RAG。

### 3.3 SQLite WAL + 状态表

`init_db()` 使用 SQLite WAL，并维护 `articles` 表。主要字段包含：标题 hash、标题、URL、域名、来源、发布时间、抓取时间、正文路径、状态、重试次数、ticker、topic、最后重试时间等。状态主要是 `pending / done / failed`。

WAL 对单机守护进程很合理：读写并行性比默认 rollback journal 更好，部署简单。但 1000 个站点、数百万 URL、多 worker 下不应把 SQLite 作为全局 durable frontier。更适合的模式是：

- PostgreSQL 保存业务真相、URL frontier、lease、版本、规则、任务；
- Redis Streams 或其它消息系统做短期分发；
- S3/MinIO 保存不可变抓取快照和 Markdown；
- SQLite 只保留为本地开发、单站调试或离线工具缓存。

### 3.4 标题 Hash 与 Jaccard 近似去重

项目先对小写归一化标题计算 SHA-256 截断 hash，做精确去重；再对最近 48 小时文章标题进行单词集合 Jaccard 相似度比较，阈值为 0.75。另有定期 `purge_duplicates()`，对最近 72 小时数据做两两比较并删除后出现的近似项。

这是典型的“两阶段去重”：廉价精确键先过滤，再做近似相似度判断。原理正确，但实现不能直接扩展到博客知识库：

1. 标题不是文章身份。同一标题可能是不同文章，标题变化也不代表文章变成新实体。
2. 英文单词集合对中文、日文和混合技术标题支持差。
3. 0.75 的固定阈值很容易误杀同一事件的不同报道。
4. 72 小时窗口内两两比较是 O(n²)，规模扩大后成本迅速上升。
5. 代码直接删除重复行，会丢失“多个 URL 指向同一内容”的来源关系和证据。

博客知识库应改成分层身份模型：

- URL 层：规范化 URL + redirect/canonical 关系；
- 内容层：正文规范化后 `content_hash`；
- 近似层：SimHash/MinHash/LSH 生成候选集，再做相似度确认；
- 语义层：只做“重复组”或“相似文章”辅助，不直接删除 canonical 版本。

重复项应保存 `duplicate_group_id`、`duplicate_of` 和发现来源，而不是物理删除。

### 3.5 Browser 资源看门狗与代际重启

项目最值得借鉴的部分之一是 Browser 自愈。代码把 Chromium 配置为轻量模式，并设置每 5 个采集周期重建 Browser；每轮开始前用 `psutil` 统计 Playwright Chromium 后代进程数量与总 RSS。如果 renderer 数超过阈值或总 RSS 超过 1GB，就主动终止进程；Browser session 结束后再做一次 orphan cleanup。

原理是“长生命周期 Browser 必须按资源压力做代际回收”。Browser 的内存泄漏、页面未关闭、第三方脚本异常和渲染器残留在长期任务中很常见，不能只依赖 Python 对象退出。

但当前实现过于粗粒度：

- `--single-process` 会降低 Chromium 的进程隔离，不适合作为大规模生产默认值；
- `--no-sandbox` 不应在生产多租户环境默认开启；
- 直接 kill 整组 Chromium 子进程会让正在运行的任务无差别失败；
- 以固定“5 个周期”重启不如按任务数、RSS、heap、page 数和 event-loop lag 决策；
- 多 worker 场景中应该由每个 Browser Worker 管理自己的进程树或容器，而不是全局扫描共享主机进程。

博客知识库应采用独立 Browser Worker Pool：每个 worker 绑定自己的 browser/context，设置 `max_pages`、`max_tasks_per_generation`、`max_rss`、`max_uptime`，达到任一阈值后停止接新任务、等待当前任务结束并优雅轮换；必要时由容器/cgroup 兜底终止。

### 3.6 域间交错与固定 Domain Delay

项目会先按 domain 分组待抓文章，再将不同 domain 的列表交错，随后串行处理，并在每篇后 sleep 3 秒。这体现了“域内克制、域间公平”的原则。

问题是它只有一个串行消费者，吞吐量受固定 3 秒 delay 限制，而且没有按站点差异调节速率。1000 站点方案应保留原则但替换实现：

- 全局 worker 并发；
- 每域 token bucket；
- 每站点可配置 `max_concurrency`、`min_interval`；
- 429/503/Retry-After 动态降速；
- Scheduler 按 domain 做 weighted fair queue；
- HTTP 与 Browser 共享同一域配额，防止两个通道叠加把请求量翻倍。

### 3.7 重试状态机与一个明显实现缺口

项目定义了 `RETRY_INTERVALS = [0, 1h, 4h, 12h]`，也保存 `last_retry_at`。`mark_retry()` 会增加重试次数，并在达到上限后把状态改为 `failed`。

但 `get_pending()` 虽然计算了 retry delay，实际分支中只有 `pass`，最终仍然把该条目加入结果。因此所谓“指数退避”目前没有真正阻止条目立即再次被本轮或下轮选中，`last_retry_at` 也没有被用于 SQL 过滤。

这是对大规模系统很重要的教训：重试策略不能只存在于日志或变量里，必须成为 durable state machine 的调度条件。生产方案应存 `next_attempt_at`，取任务时直接用数据库条件：

`status='retry_wait' AND next_attempt_at <= now()`

并按错误类别决定重试策略：网络超时、DNS、429、5xx、robots deny、404/410、解析失败、Browser crash、质量门禁失败不能使用同一退避规则。

### 3.8 FTS5 搜索

项目建立 `articles_fts` FTS5 虚拟表，并每 10 分钟执行一次 `rebuild`。对单机几千到几万篇文章很方便，CLI 查询也无需额外服务。

但 `rebuild` 本质上是全量重建，文章数达到数百万时不适合高频执行。博客知识库应把搜索视为可重建派生层：

- 小规模可直接使用 PostgreSQL `tsvector`；
- 数百万文章、复杂分词或高 QPS 时使用 OpenSearch/Elasticsearch；
- Index Worker 订阅已发布 `article_version`，逐版本 upsert；
- 重建新 generation 后切 alias，不在原索引上破坏性重建。

### 3.9 Markdown 文件存储

项目用标题生成文件名，按 ticker/market 子目录落盘，并写入 URL、Source、Published、Crawled 等元信息。

这种方式人类可读、调试方便，但不适合作为 canonical 存储：标题可能变化、不同标题规范化后可能碰撞、文件名与文章身份绑定过紧。代码虽然检测首行并在冲突时追加 hash，但仍然缺少明确版本模型。

生产方案应让对象 key 与业务 ID/版本绑定，例如：

`articles/{site_id}/{article_id}/{version_id}/article.md`

对外导出时再生成友好路径。Markdown Front Matter 只是展示契约，不应替代数据库中的结构化 metadata/provenance。

### 3.10 主题分类

项目把 topic classifier 作为可选模块，先用标题/摘要分类，正文抓到后再用正文前 1000 字符重新分类。这是“先廉价后精化”的典型 enrichment 流程。

博客知识库应把主题、摘要、embedding、实体等能力统一放入异步 Enrichment Plane。它们只消费已发布 canonical `article_version`，失败不能阻止 Markdown 发布，也不能反向覆盖 canonical 正文。

### 3.11 保留策略与自动过期

项目默认删除 7 天以前的文章和文件，并清理长期失败记录，这符合“滚动金融新闻窗口”的用途。但这与“全量历史博客知识库”目标完全相反。

知识库的 canonical 内容必须长期保存，删除应改为：

- 源站 404/410：记录 tombstone 和最后可用版本；
- 用户主动删除：软删除 + 审计；
- 原始大对象：按合规和成本策略做生命周期分层，例如热存储转归档存储，而不是删除文章版本；
- 失败任务：可以压缩/归档日志，但不能因为旧而抹掉失败证据。

### 3.12 PID Lock、信号处理和优雅退出

项目通过 PID 文件避免本机重复启动，并监听 SIGTERM/SIGINT 设置 `shutdown_event`，各主要循环都检查该事件。这说明长期守护进程必须支持幂等启动和可中断 sleep。

单机 PID lock 不适用于多节点。生产方案应该使用数据库 lease、Kubernetes lease 或任务表的 compare-and-set 来防止同一 job 被重复执行。取消也要成为 durable 状态，不应只靠当前进程内 `asyncio.Event`。

## 4. 数据质量与正确性风险

### 4.1 `publish_at` 与 `observed_at` 混淆

Finviz headline 和 RSS 插入时大量使用当前时间作为 `publish_at`。这会把“系统观察时间”误当作“文章来源发布时间”，影响时间排序、增量窗口、历史回溯和版本判断。

生产模型必须至少拆分：`published_at`、`updated_at`、`first_seen_at`、`last_seen_at`、`fetched_at`、`captured_at`。确定不了真实发布时间时宁可为空，也不要用抓取时间伪装。

### 4.2 原始标题被转成小写

插入时会把标题存成 lower-case。对检索去重方便，但丢失了原始展示形式。正确方式是同时保存 `title_raw` 和 `title_normalized`，canonical 展示使用来源原文，规范化字段只用于匹配。

### 4.3 RSS summary 被直接标记 done

summary 可能只有几句话，却和抓到全文的记录共用 `done`。状态只表达“流程完成”，没有表达“内容完整度”。知识库必须把流程状态与内容质量状态拆开。

### 4.4 静态 PAYWALL/RSS 域名集合

域策略硬编码在代码里，不利于 1000 站点运营。应移入站点/平台 Profile，在 Web 管理端可见、可审计、可灰度发布，并带版本号。

### 4.5 正则直接解析页面结构

Finviz headline 解析使用正则匹配 HTML。对固定页面的快速脚本可接受，但模板微调即可失效。通用方案应优先 DOM parser/selector/结构化数据，正则只用于文本字段后处理。

### 4.6 Browser 与非 Browser 的 robots 策略不统一

第三方 Crawl4AI 正文抓取启用了 `check_robots_txt=True`，但轻量 `curl_cffi` 通道并没有同样的统一前置策略。生产系统应把 robots/egress/站点政策放在 transport 之前，而不是让每种 fetcher 自己决定。

### 4.7 README clone 地址与当前仓库 owner 不一致

README 示例使用 `eziosun/finviz-crawler.git`，当前项目地址为 `Ezio0/finviz-crawler`。这说明生产调研和依赖管理不能只依赖 README 文本，应记录 canonical repository、commit SHA、license、抓取时间，并在来源变化时告警。

## 5. 对博客知识库方案的可复用设计

本项目可以沉淀出以下可复用能力：

1. **Transport Strategy Chain**：Feed/HTTP/Browser 分层，不默认 Browser。
2. **Browser Generation Recycling**：按任务数、内存、寿命主动轮换 Browser，而不是等待 OOM。
3. **Worker Resource Guard**：监控 RSS、子进程、page/context 数、event-loop lag，资源压力时只收缩并发。
4. **Source Completeness**：RSS summary、metadata-only、full article 显式区分。
5. **Two-stage Dedup**：精确键 + 近似候选，但升级为 URL/content hash/MinHash，而非标题 Jaccard 直接删除。
6. **Incremental Search Update**：本地 FTS5 的思路保留，但实现改为可重建、版本化的 Search Generation。
7. **Durable Retry Scheduling**：把 `next_attempt_at` 作为数据库状态，而不是进程内 sleep。
8. **Graceful Shutdown**：任务要支持 cancel、drain、lease 过期和重新领取。
9. **Per-source Policy**：paywall、JS-heavy、RSS-capable、rate limit、Browser required 等属性进入 Profile，不硬编码。
10. **Operational Self-healing**：抓取 worker 必须能在 Browser 泄漏、孤儿进程、连续错误和内存压力下自愈。

## 6. 不应直接复用的设计

1. 单进程 SQLite 作为全局任务状态中心。
2. 固定 5 分钟轮询所有来源。
3. 标题 hash 作为文章唯一身份。
4. 固定 0.75 标题 Jaccard 作为自动删除依据。
5. 每 10 分钟全量 FTS rebuild。
6. 7 天 TTL 删除 canonical 历史文章。
7. `--no-sandbox` 与 `--single-process` 作为生产 Browser 默认配置。
8. 进程级 PID file 作为分布式互斥。
9. 通过 TLS 指纹模拟绕过明确反爬/访问控制。
10. RSS summary 与完整正文共用同一“done”语义。
11. 用当前时间代替真实发布时间。
12. 用正则作为长期 HTML 模板解析主方式。

## 7. 对最终技术方案应加入的具体优化

### 7.1 新增 Fetch Strategy / Transport Adapter 层

每个 URL 根据 Profile、上次结果、内容类型和站点政策选择 `FEED_ONLY / HTTP / BROWSER / BLOCKED / MANUAL`，并记录升级原因。HTTP 与 Browser 共享域级限流。

### 7.2 新增 Browser Worker 资源代际模型

Browser Worker 保存 generation 元数据：启动时间、已处理任务数、当前 page 数、RSS、连续失败数。超过任一阈值后进入 draining，完成在途任务后重建 Browser。容器/cgroup 做最终硬限制。

### 7.3 新增内容完整度与可用性字段

`article_version` 增加 `content_completeness`、`content_quality_state`、`source_kind`、`fetch_transport`，避免 summary-only 被误当全文。

### 7.4 新增相似文章组而不是“重复即删除”

URL 规范化和 content hash 负责强去重；MinHash/SimHash 只创建 `duplicate_group` 候选，经规则确认后选 canonical，仍保留所有来源证据。

### 7.5 新增真正可执行的退避调度

任务表保存 `attempt_count`、`last_error_class`、`next_attempt_at`、`lease_until`。Scheduler 只派发到期任务，并对 429、5xx、DNS、Browser crash、404、robots deny 使用不同策略。

### 7.6 搜索从全量 rebuild 改为 generation + incremental upsert

每个已发布文章版本产生索引事件；索引故障不会触发重新抓源站。需要重建时创建新 generation，完成后原子切 alias。

### 7.7 Web 管理端加入资源与完整度视图

除站点进度外，应直接展示 Browser pool RSS、page 数、generation age、HTTP/Browser 升级率、summary-only 比例、重复组命中率、域级 429/403、下一次同步时间等。

## 8. 结论

`finviz-crawler` 对大规模博客知识库最有价值的不是 Finviz 站点逻辑，而是它把持续运行中的真实工程问题暴露得非常直接：轻重抓取通道分层、Browser 资源泄漏、自愈重启、近似重复、全文索引、失败重试、优雅退出。

最终方案应吸收“多 transport、Browser 代际回收、资源看门狗、增量索引、近似重复候选、优雅退出”的思想，同时放弃其单机轮询、标题身份、固定 TTL、全量 FTS rebuild 和反爬绕过做法。对于 1000 个博客网站，正确的演进方向是把这些能力拆成可配置、可版本化、可观测、可横向扩展的控制面与 worker 平面，并让 PostgreSQL/S3 中的 canonical 状态始终独立于具体抓取工具和搜索后端。

## 9. 参考源码

- 项目主页：https://github.com/Ezio0/finviz-crawler
- README：https://github.com/Ezio0/finviz-crawler/blob/main/README.md
- 主采集脚本：https://github.com/Ezio0/finviz-crawler/blob/main/scripts/finviz_crawler.py
- 查询工具：https://github.com/Ezio0/finviz-crawler/blob/main/scripts/finviz_query.py
- 主题分类器：https://github.com/Ezio0/finviz-crawler/blob/main/scripts/topic_classifier.py
- 依赖：https://github.com/Ezio0/finviz-crawler/blob/main/requirements.txt
- v3.5 提交：https://github.com/Ezio0/finviz-crawler/commit/f34e1832c06c176c1c8398d3d2a1653a311fd308
