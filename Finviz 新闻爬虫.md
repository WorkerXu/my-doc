# Finviz 新闻爬虫：实现细节与技术原理分析

- 编号：52
- 项目：finviz-crawler
- 地址：https://github.com/Ezio0/finviz-crawler
- 调研对象：main 分支 v3.5 代码
- 状态：调研中

## 1. 项目定位

`finviz-crawler` 是一个持续运行的金融新闻采集守护进程。它把 Finviz 新闻入口、多个 RSS 源、第三方文章正文抓取、SQLite 状态库、Markdown 文件落盘、主题分类、近似去重、全文检索、失败重试和 Browser 资源自愈组合在一个 Python 进程中。

它并不是可直接用于“1000 个技术博客全量历史归档”的通用框架，但非常适合作为运行时工程样本：它暴露了长期抓取系统中最容易被方案文档忽略的真实问题，包括轻量 HTTP 与 Browser 的成本差异、Browser 内存膨胀、不同来源的完整度差异、近似重复、失败重试、持续增量、全文索引和长期守护进程的自愈。

## 2. 实际数据流

主流程可以抽象为：

1. 周期抓取 Finviz 新闻页；
2. 拉取 Bloomberg、CNBC、MarketWatch、Yahoo Finance、Investing、SeekingAlpha、TradingView 等 RSS；
3. 新条目进入 SQLite `articles` 表；
4. 等待正文的文章进入 `pending`；
5. 通过 Crawl4AI/Playwright 抓第三方正文；
6. 正文转 Markdown 并写本地文件；
7. 更新 `done / pending / failed` 状态；
8. 周期清理旧文章和失败项；
9. 周期重建 FTS5；
10. 监控 Chromium 子进程和 RSS 内存，并周期重建 Browser。

v3.5 又增加 `curl_cffi` 作为轻量通道：对 Finviz 页面先走普通 HTTP/TLS 指纹模拟，失败再回退 Crawl4AI；第三方正文仍由 Crawl4AI 执行。

从架构角度看，这是一个“Source Discovery -> Lightweight Fetch -> Heavy Browser Fetch -> Local State -> Markdown -> Search”的单机实现。

## 3. 多抓取通道的技术原理

### 3.1 HTTP 优先、Browser 按需升级

代码对 Finviz 主新闻页和 ticker 页面优先使用 `curl_cffi`，只有失败后才回退 Crawl4AI。第三方正文才默认走 Browser。

这一点对大型博客知识库非常重要：Browser 不应该是默认传输层。Browser 会引入 Chromium 进程、renderer、JS 执行、字体、图片、共享内存、文件描述符和更高的 CPU/RSS 成本。1000 个站点如果全部 Browser-first，会很快把吞吐量和稳定性拖垮。

生产方案应该把抓取抽象成统一 Transport Strategy：

`Feed/Sitemap -> HTTP -> Browser -> Blocked/Manual`

每次升级 Browser 都必须记录原因，例如：`js_required`、`empty_body`、`challenge_page`、`selector_miss`、`content_too_short`，以便后续优化站点 Profile。

需要注意，项目使用 TLS 指纹模拟来尝试绕过 Cloudflare challenge。大型知识库系统不应把“绕过明确访问控制”设计成默认能力。正确的借鉴点是“多 transport adapter + fallback”，而不是反爬绕过本身。

### 3.2 RSS 是发现源，不等于完整正文

项目把 RSS 当作低成本增量来源，这是正确方向。但 `insert_rss_article()` 会把 RSS summary 直接写成 Markdown，并标记为 `done`。

这里暴露出一个关键建模问题：流程状态不能代表内容完整度。`done` 只能说明“当前流程结束”，不能说明正文完整。

知识库应该显式保存：

- `content_completeness = full | summary_only | metadata_only`
- `discovered_via = feed | sitemap | archive | link | api`
- `fetch_transport = http | browser | archive`
- `quality_state = accepted | degraded | rejected`

否则 RSS 摘要会进入全文索引和 RAG，被误认为完整文章。

## 4. SQLite 状态库与其边界

项目使用 SQLite WAL，并维护 `articles` 表。字段包含标题 hash、标题、URL、domain、source、发布时间、抓取时间、正文路径、状态、重试次数、ticker、topic、最后重试时间等。

WAL 对单机进程很合理：部署简单，读写并发比默认 journal 模式更好。但 1000 个站点、多 worker、数百万 URL 的 Durable Frontier 不适合继续使用单机 SQLite。

生产方案应改为：

- PostgreSQL：站点、URL frontier、任务、lease、版本、规则、文章身份、状态真相；
- Redis Streams：短期任务分发和 backpressure，不做业务真相；
- S3/MinIO：原始响应、rendered DOM、抽取结果、Markdown、附件；
- SQLite：仅保留为本地调试、单站诊断或 CLI 缓存。

## 5. 去重实现与问题

项目有两层标题去重：

1. 标题 lower-case 后计算 SHA-256 截断 hash；
2. 对最近 48 小时标题做词集合 Jaccard，相似度 >= 0.75 判重复。

另有 `purge_duplicates()`，对最近 72 小时文章两两比较并删除后出现的重复项。

它体现了正确的“两阶段去重”思路：先做廉价精确判断，再做近似判断。但实现不能直接扩展到博客知识库：

- 标题不是文章身份；
- 同标题可能是不同文章；
- 改标题不代表生成新文章；
- 英文词集合对中文/日文效果差；
- 两两比较是 O(n²)；
- 直接删除重复行会损失 provenance。

知识库应该采用四层身份模型：

1. URL 层：canonical URL、redirect、normalized URL；
2. 内容层：正文规范化后的 `content_hash`；
3. 近似层：SimHash/MinHash/LSH 建候选；
4. 语义层：只做相似文章分组，不直接物理删除。

重复记录应该保存 `duplicate_group_id`、`duplicate_of`、来源 URL 和发现证据。

## 6. Browser 资源自愈

项目会统计 Chromium 子进程数量和总 RSS，并周期性重建 Browser。README 说明 Browser 大约每 5 个采集周期重启，以缓解长期运行后的内存膨胀。

这个思路非常有价值：长生命周期 Browser 必须进行“代际回收”，不能假设对象关闭就一定释放所有 renderer、V8 heap、共享内存和子进程资源。

但当前实现仍然偏单机脚本：

- `--single-process` 降低进程隔离；
- `--no-sandbox` 不适合作为生产默认值；
- 直接杀 Chromium 子进程可能影响在途任务；
- 固定每 5 轮重启不如按资源压力决策；
- 多 worker 时不能全局扫描并杀共享主机 Browser。

生产方案应该给每个 Browser Worker 保存 generation 状态：

- `started_at`
- `tasks_processed`
- `active_pages`
- `rss_bytes`
- `failure_streak`
- `generation_id`

达到 `max_tasks / max_rss / max_uptime / max_pages` 任一阈值后进入 `draining`，停止接新任务，等待在途任务结束，再优雅重建 Browser。容器/cgroup 作为最后的硬资源边界。

## 7. 域级公平与限速

项目把待抓文章按 domain 分组，再交错处理，并在每个页面后固定 sleep 3 秒。这体现了一个正确原则：跨域可以并行，域内应该克制。

但固定 sleep 不适合大规模系统。1000 个站点应升级为：

- 全局 worker 并发；
- 每域 token bucket；
- `max_concurrency`、`min_interval` 站点可配置；
- 429/503/Retry-After 自动降速；
- weighted fair scheduling；
- HTTP 与 Browser 共享同一 domain quota。

否则 HTTP Worker 与 Browser Worker 会分别限流，实际对源站造成双倍压力。

## 8. 重试状态机中的关键缺陷

项目定义：

`RETRY_INTERVALS = [0, 1h, 4h, 12h]`

也保存了 `last_retry_at`。`mark_retry()` 会增加 `retry_count` 并记录最后失败时间。

但是 `get_pending()` 中虽然计算了 retry delay，判断分支最后只是 `pass`，条目仍然被加入待执行列表。也就是说，代码层面的“指数退避”并没有真正阻止任务立即再次被取出。

这是对生产系统非常重要的警示：重试策略必须成为 durable scheduling condition，而不是日志层策略。

任务表应保存：

- `attempt_count`
- `last_error_class`
- `last_attempt_at`
- `next_attempt_at`
- `lease_until`

Scheduler 只领取 `next_attempt_at <= now()` 的任务。

错误必须分类：

- DNS/连接超时：指数退避；
- 429：优先尊重 `Retry-After` 并降低域级速率；
- 5xx：退避重试；
- 404/410：进入删除确认/tombstone 流程；
- robots deny：永久 blocked；
- Browser crash：换 worker/generation；
- extractor failure：优先 re-extract，不重新访问源站；
- quality gate failure：进入规则诊断，不盲目重新抓取。

## 9. 时间语义错误

项目在插入 Finviz headline 和 RSS 时大量使用当前系统时间作为 `publish_at`。这会把“首次观察时间”混成“文章来源发布时间”。

对博客知识库，这会直接破坏：

- 历史排序；
- 增量游标；
- 文章更新检测；
- 版本时间线；
- 搜索时间过滤。

生产数据模型至少要分开：

- `published_at`
- `updated_at`
- `first_seen_at`
- `last_seen_at`
- `fetched_at`
- `captured_at`

无法确认真实发布时间时应该为空，并保存时间候选及 provenance，而不是用抓取时间代替。

## 10. 标题规范化不能覆盖原文

项目插入时会把标题 lower-case 后存入主字段。这对匹配方便，但损失原始展示值。

正确设计应同时保存：

- `title_raw`
- `title_normalized`

同理，作者、标签、时间、canonical URL 等字段都应该保留 raw candidate、normalized candidate 和最终 resolved 值。

## 11. HTML 正则解析的脆弱性

Finviz headline 解析主要依赖正则匹配 HTML。对单一页面脚本可以快速工作，但模板稍微调整就可能全部失效。

通用知识库应该优先：

1. JSON-LD / OpenGraph / meta；
2. DOM selector；
3. 平台 Profile；
4. 通用正文抽取器；
5. 文本正则只用于字段后处理。

所有 selector/contract 都必须版本化，并可在 Web 管理端查看命中率和模板漂移。

## 12. FTS5 与索引重建

项目使用 SQLite FTS5，并周期执行 `rebuild`。对小型本地知识库方便，但数百万文章下频繁全量 rebuild 成本不可接受。

生产方案应把索引作为可重建派生层：

- 新 `article_version` 发布后发 Index Event；
- Index Worker 对该版本 incremental upsert；
- 需要重建时创建新 Search Generation；
- 完成后原子切 alias；
- 索引失败不能触发重新抓源站。

## 13. Markdown 文件模型

项目按标题生成 Markdown 文件名，并按 ticker/market 分目录。这适合人工查看，但不适合作为 canonical object identity。

生产对象路径应绑定稳定 ID 和版本，例如：

`articles/{site_id}/{article_id}/{version_id}/article.md`

友好文件名只用于导出。Markdown Front Matter 是展示接口，不是数据库真相。

## 14. 自动过期与知识库目标冲突

Finviz 的默认场景是滚动金融新闻，所以会删除超过若干天的旧文章和失败记录。这对新闻窗口合理，但对“全量历史知识库”完全不适用。

博客知识库应：

- 404/410：保留 tombstone 和最后成功版本；
- 用户删除：软删除 + 审计；
- 原始大对象：生命周期转冷存储，而不是删除 canonical 版本；
- 失败记录：可归档日志，但不能抹掉失败证据。

## 15. PID Lock 与优雅退出

项目使用 PID file 防止重复启动，并通过 SIGINT/SIGTERM 设置 `shutdown_event`，让循环可中断退出。

生产多节点环境不能使用 PID file 作为分布式互斥，应使用数据库 lease/compare-and-set。取消操作也必须持久化，例如 `cancel_requested_at`，Worker 在阶段边界检查并 drain，而不是只依赖进程内 Event。

## 16. Topic Classification 的可复用原则

项目先用标题/摘要分类，抓到正文后再用正文重新分类。这体现“先廉价、后精化”的 enrichment 模式。

博客知识库应把分类、摘要、实体、embedding、代码摘要统一放入异步 Enrichment Plane。它们只消费已发布的 canonical `article_version`；失败不能阻塞正文发布，也不能覆盖 canonical 内容。

## 17. 对最终方案应吸收的能力

1. Transport Strategy Chain：Feed/HTTP/Browser 分层。
2. Browser Generation Recycling：按资源压力和任务数代际回收。
3. Worker Resource Guard：RSS、page 数、event-loop lag、自适应降并发。
4. Content Completeness：`full / summary_only / metadata_only`。
5. URL/content/near-duplicate 多层去重，不以标题直接删除。
6. Durable Retry：`next_attempt_at` 真正进入调度条件。
7. Search Generation：索引 incremental upsert + generation rebuild。
8. Graceful Drain：cancel、lease、任务重领。
9. Site/Profile Policy：paywall、JS-heavy、feed、限速、Browser requirement 配置化。
10. 时间语义拆分：发布时间与首次观察时间严格分离。
11. 原始字段与规范化字段分离。
12. RSS/Feed 只作为发现或降级内容来源，不冒充完整正文。

## 18. 不应直接复用的设计

1. 单进程 SQLite 作为全局任务中心；
2. 固定周期轮询所有来源；
3. 标题 hash 作为文章唯一身份；
4. 标题 Jaccard 达阈值即自动删除；
5. 高频 FTS 全量 rebuild；
6. 7 天 TTL 删除 canonical 历史内容；
7. `--single-process`、`--no-sandbox` 作为生产 Browser 默认；
8. PID file 作为分布式锁；
9. TLS 指纹模拟作为访问控制绕过方案；
10. RSS summary 与完整正文共用 `done` 语义；
11. 当前时间代替真实发布时间；
12. 正则作为长期页面结构解析主方案。

## 19. 对博客知识库最终方案的具体优化结论

最终方案应正式加入以下约束：

- Fetch Task 必须记录 transport、升级原因、domain quota；
- Browser Worker 必须有 generation/draining 模型；
- `article_version` 必须有 `content_completeness` 与 `quality_state`；
- Task 必须有 `next_attempt_at`，Scheduler 不得忽略退避；
- `published_at` 与 `first_seen_at` 必须分离；
- duplicate 只创建关系和组，不默认物理删除；
- Search/Vector/外部知识库都是可重建派生层；
- RSS summary 只作为降级版本，不能覆盖完整正文；
- 站点策略全部进入版本化 Profile，不硬编码到抓取器；
- Browser/HTTP/Feed 必须统一经过 robots、Egress Policy 和域级限流。

## 20. 结论

`finviz-crawler` 的价值不是其金融新闻逻辑，而是它非常集中地展示了持续抓取系统的运行时现实：轻重抓取通道、Browser 泄漏、自愈、近似重复、重试、索引、优雅退出、内容完整度和时间语义。

对于 1000 个技术博客，应吸收这些工程思想，但把它们从单机脚本升级为可配置、可版本化、可观测、可横向扩展的控制面和 Worker 平面，并让 PostgreSQL + S3/MinIO 中的 canonical 状态始终独立于具体抓取工具、Browser、搜索后端和外部知识库。

## 21. 参考源码

- 项目主页：https://github.com/Ezio0/finviz-crawler
- README：https://github.com/Ezio0/finviz-crawler/blob/main/README.md
- 主采集脚本：https://github.com/Ezio0/finviz-crawler/blob/main/scripts/finviz_crawler.py
- 查询工具：https://github.com/Ezio0/finviz-crawler/blob/main/scripts/finviz_query.py
- 主题分类器：https://github.com/Ezio0/finviz-crawler/blob/main/scripts/topic_classifier.py
