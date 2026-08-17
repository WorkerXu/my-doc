# LLM应用实战-财经新闻自动聚合：实现细节与技术原理分析

- 编号：62
- 文章：LLM应用实战-财经新闻自动聚合
- 地址：https://www.cnblogs.com/mengrennwpu/p/18609896
- 来源：博客园
- 原文发布日期：2024-12-16
- 调研日期：2026-08-14
- 相关框架：Crawl4AI、AsyncWebCrawler、JsonCssExtractionStrategy

## 1. 文章定位

这篇文章实现的是一个“多财经站点近期新闻自动聚合器”。作者列出了财联社、凤凰财经、新浪财经、环球、早报、Fox、CNN、Reuters 等来源，并展示了财联社抓取实现：先从动态栏目页发现新闻链接，再进入详情页结构化抽取，最后使用 URL 的 MD5 作为历史去重键，将新数据增量保存到本地文件，并返回前一日新闻。

它不是面向 1000 个技术博客的全量历史归档系统，但恰好暴露了生产级博客知识库最关键的几个问题：

1. 同一网站往往不只有一个“站点入口”，而是多个栏目、频道、分页/API/Feed 入口；
2. 动态列表发现和文章详情抽取是两种完全不同的任务；
3. JavaScript 下拉/加载更多需要明确的停止条件，否则无法证明覆盖范围，也可能无限执行；
4. 本地文件 + URL MD5 能实现单机幂等，但无法支撑多 Worker、版本、补漏、回放和可观测；
5. “昨天的新闻”属于消费查询窗口，而不是采集状态或文章身份的一部分；
6. 每站点独立 CSS schema 很常见，因此 schema、Browser 动作和站点差异必须配置化、版本化，而不能散落在代码类中。

对博客知识库最有价值的不是照搬脚本，而是把这些局部实现提升为 `site -> site_source/discovery_endpoint -> discovery provider -> durable frontier -> fetch -> snapshot -> extract -> article version` 的可扩展模型。

## 2. 原文代码的数据流

财联社实现可抽象为：

```text
3 个栏目页
  depth?id=1000 头条
  depth?id=1003 A股
  depth?id=1007 环球
        |
        v
Crawl4AI Browser
  + JsonCssExtractionStrategy
  + js_commands 下拉/点击“加载更多”
        |
        v
提取详情页链接 -> set 去重 -> URL -> category
        |
        v
逐条 crawl_newsletter()
  -> 标题
  -> 时间
  -> 摘要
  -> 正文段落
  -> 阅读数
        |
        v
id = md5(url)
        |
        +-- history 已存在 -> 跳过
        |
        +-- 新文章 -> JSONL 追加
        |
        v
get_last_day_data() -> 输出前一日新闻
```

从职责上看，代码已经隐含了四层：

- **Source/Channel**：1000、1003、1007 三个栏目；
- **Discovery**：动态加载栏目页并提取详情链接；
- **Detail Fetch + Extraction**：详情页结构化抽取；
- **Incremental State**：本地 `history` 防重复，并按日期查询输出。

生产方案应该保留这四层的语义，但必须把状态和执行机制彻底拆开。

## 3. `JsonCssExtractionStrategy` 的技术原理

原文为列表页和详情页分别声明 JSON schema。核心思想是把“目标字段”描述为结构化配置，而不是在业务代码里逐个 `querySelector`。

以详情页为例，schema 声明：

- `baseSelector`：限定抽取根节点；
- `fields[].selector`：字段 CSS selector；
- `type=text`：抽文本；
- `type=list/nested_list`：抽多项或嵌套项；
- `type=attribute`：抽 `href` 等属性。

这一机制的本质是：

```text
Rendered DOM
    + Extraction Schema
    -> Deterministic Structured Candidate
```

它比直接让 LLM 从网页“理解并生成 JSON”更适合稳定站点，因为：

- 可重复；
- 成本低；
- 易测试；
- selector 漂移可监控；
- 能明确知道字段来自哪个 DOM 节点。

但生产系统不能把 schema 当成永久正确的单一真相。应把它设计为版本化 `extraction_contract_release`，并允许多个候选来源：

```text
JSON-LD / OpenGraph / Meta / Feed Metadata / CSS Schema / Readability / Trafilatura
                         |
                         v
                  Metadata Resolver
```

正文也可保留多个 extractor candidate，由质量门禁选择 canonical 结果。这样 selector 更新后可以对已有 DOM 快照离线重放，而无需重新请求源站。

## 4. 动态栏目发现：`js_commands` 的作用与风险

文章中最值得深入分析的是 `crawl_url_list()` 的动态加载逻辑。页面初始只显示一部分新闻，JS 代码不断：

1. 统计当前新闻项数量；
2. 滚动到页面底部；
3. 等待；
4. 查找“加载更多”按钮；
5. 点击；
6. 再等待；
7. 重新统计数量；
8. 直到达到 100 条或找不到按钮。

这说明对于现代站点，“URL 发现”不能只理解为下载 HTML 后解析 `<a>`。很多列表需要浏览器交互、无限滚动或客户端 API 才能暴露更多 URL。

### 4.1 原实现缺少严格终止条件

虽然代码设置 `targetItemCount = 100`，但仍存在一个重要边界：如果按钮一直存在、点击却不再增加条目，则 `currentItemCount` 不变，而 `loadMoreButton` 仍存在，循环可能持续运行。

生产系统的 Browser Action Plan 必须同时具备：

- `max_items`
- `max_iterations`
- `max_duration_ms`
- `max_scrolls`
- `max_clicks`
- `no_growth_rounds`
- `same_link_set_rounds`
- `button_missing` 停止
- `next_cursor_missing` 停止
- 全局页面超时

推荐停止条件：

```text
达到业务上限
OR 连续 N 轮 URL 集合没有增长
OR 找不到下一页/加载更多
OR 返回 cursor 重复
OR 达到最大迭代/时间/字节预算
```

只有这样动态列表才能成为可控的生产任务。

### 4.2 不应把任意 JS 字符串直接作为长期配置

原文直接传 `js_commands` 字符串，对个人脚本简单有效。但在 Web 管理系统中允许任意 JS 会带来：

- 配置不可静态校验；
- 很难评估资源消耗；
- 难以比较 release diff；
- 容易出现死循环；
- 安全边界扩大；
- 无法可靠统计“滚动了几次、点了几次、为何停止”。

更合理的是定义受限动作 DSL，例如：

```yaml
steps:
  - wait_for: "div.content-left"
  - repeat:
      max_iterations: 20
      until:
        no_new_links_rounds: 2
      actions:
        - scroll_to_bottom: true
        - wait_ms: 800
        - click_if_exists: ".list-more-button.more-button"
        - wait_ms: 800
extract_links:
  selector: "div.content-left a[href]"
budgets:
  max_duration_ms: 30000
  max_links: 500
```

底层可以翻译为 Playwright/Crawl4AI 动作。确实无法表达的特殊站点再进入 `Custom Adapter`，而不是把任意 JS 作为默认模式。

### 4.3 动态列表优先识别底层数据接口

浏览器交互只是手段，不是目标。许多“加载更多”最终请求 JSON/XHR/API。若接口是公开、稳定、合规可用的，应优先将其注册为 discovery provider，因为 API 通常：

- 更省 CPU/内存；
- 更容易分页；
- cursor 更适合作 checkpoint；
- 更容易判断终点；
- 更容易做条件请求和重试。

因此 Source Discovery 应同时分析 HTML、网络请求和平台特征，形成 `HTTP page / API / Feed / Sitemap / Browser action` 多种候选 provider。

## 5. 同一站点的多个栏目必须是一等实体

原文把财联社三个栏目写在 `menu_dict` 中：

```text
1000 -> 头条
1003 -> A股
1007 -> 环球
```

这说明 `site` 粒度太粗。生产系统至少还需要 `site_source`（也可叫 `discovery_endpoint`）：

- `site_id`
- `name/category`
- `source_url`
- `source_type`
- `locale/region`
- `provider_type`
- `capability`：exhaustive/windowed/best_effort
- `profile_release_id`
- `action_plan_release_id`
- `sync_policy_id`
- `enabled`

这样一个网站可以有：

- 主 Sitemap；
- 多个分类 Sitemap；
- RSS；
- “最新文章”页；
- 年份归档；
- 标签页；
- 动态栏目；
- 官方 API。

每个 endpoint 都可以有独立 checkpoint、水位、连续性状态和调度频率。

## 6. 原实现的分类覆盖会丢信息

`crawl_url_list()` 最后使用一个字典：

```text
results[url] = category
```

同一文章如果同时出现在“头条”和“A股”，后写入的 category 会覆盖前一个。对于知识库，发现来源本身就是 provenance，不能覆盖。

正确模型应是：

```text
url_record
  1 --- N discovery_edge
             |- source_endpoint_id=头条
             |- source_endpoint_id=A股
             |- discovered_at
             |- run_id
```

文章标签/分类也应允许多值，并区分：

- 来源站点声明的分类；
- 系统推断的主题；
- 人工标签。

这些语义不能压缩成一个 `type` 字符串。

## 7. URL MD5 历史去重：有效但不够

原文用：

```python
id = md5(url)
```

并在本地 `history` 中判断是否出现过。它解决了最基本的“同一 URL 不重复抓”问题，但生产环境有四个不足。

### 7.1 未规范化 URL

以下 URL 很可能是同一内容：

```text
https://example.com/post?id=1&utm_source=a
https://example.com/post?id=1&utm_source=b
https://example.com/post?id=1#comments
```

直接 hash 原 URL 会产生多个 identity。应先规范化：

- scheme/host 标准化；
- 去 fragment；
- tracking 参数规则化；
- trailing slash 规则；
- redirect chain；
- canonical evidence。

再对 `url_normalized` 使用 SHA-256/BLAKE3 等稳定 hash，并通过数据库唯一约束保证幂等。

### 7.2 URL 身份不等于文章身份

文章可能：

- 改 URL；
- 从旧域迁移；
- 有 AMP/打印版；
- 有 canonical；
- 被多个入口镜像。

因此还要有独立 `article_id`，由 canonical/redirect/content 证据解析 URL 与文章实体之间的关系。

### 7.3 URL 去重不等于内容去重

同一正文可能出现在多个 URL。需要：

- 规范化正文 `content_hash` 做强重复；
- MinHash/SimHash 只生成近似重复候选；
- 保留所有来源 URL 和发现证据。

### 7.4 MD5 不适合作为长期主身份算法

MD5 在非对抗脚本中可做快速 key，但已经不是合适的长期主身份 hash。知识库建议至少采用 SHA-256，并保存 hash algorithm/version，便于未来演进。

## 8. 本地 JSONL `history` 的扩展性边界

`AINewsCrawler.init()` 会读取整个文件并构造：

```text
{id -> article}
```

`FinanceNewsCrawler.save()` 再用 append 模式追加新数据。

这种实现对个人脚本的优点是零部署、简单、可观察；但当数据持续增长或多 Worker 并发时，会出现：

- 启动 O(N) 扫描；
- 内存随历史线性增长；
- 多进程写冲突；
- 无法做 task lease；
- 无法表达 attempt/retry；
- 无法表达 immutable snapshot；
- 无法做事务性的“发现 URL + 创建任务”；
- 无法可靠处理 canonical、版本和删除；
- 不能支撑 Web 管理查询。

生产方案应采用：

- PostgreSQL：业务状态真相源；
- Transactional Outbox：跨 DB/队列可靠投递；
- Redis Streams：短期任务分发；
- S3/MinIO：不可变大对象；
- Search/Vector：可重建派生层。

## 9. 每次 `crawl()` 都创建 Crawler 的资源成本

`AINewsCrawler.crawl()` 内部使用：

```python
async with AsyncWebCrawler(...) as crawler:
    result = await crawler.arun(...)
```

而 `CLSCrawler.crawl()` 对每个详情链接都逐个调用 `crawl_newsletter()`，后者再次进入 `super().crawl()`。

也就是说，从结构上看，Crawler 生命周期非常细，且详情页是串行抓取。即使具体 Crawl4AI 版本内部有缓存或浏览器复用，这种业务层生命周期也不适合 1000 站点规模。

生产系统应采用独立 Browser Worker Pool：

- Browser/process 长生命周期；
- page/context 按任务创建与回收；
- context 按安全边界隔离；
- worker 记录 RSS/page/context；
- 达到任务数、内存、存活时间阈值后 draining/recycle；
- Browser 任务有较低并发并受域限流；
- HTTP Worker 和 Browser Worker 分离扩容。

## 10. 详情抓取是串行的，吞吐量受单页延迟限制

原文：

```python
for link, category in link_2_category.items():
    ...
    news = await self.crawl_newsletter(link, category)
```

每次 `await` 完成后才处理下一条。假设每页 2 秒，100 条至少需要约 200 秒，还未计浏览器启动和失败时间。

生产系统不应简单改成 `asyncio.gather(数千 URL)`，而应把 URL 写入 durable frontier，由 Scheduler 控制：

- global concurrency；
- per-domain concurrency；
- request interval/token bucket；
- job priority；
- weighted fairness；
- Browser capacity；
- downstream backpressure。

这样 Full Backfill 可以高吞吐，但不会压垮单个站点或让一个大站点占满系统。

## 11. Cache 全绕过不适合长期增量同步

原文默认：

- `always_by_pass_cache=True`
- `bypass_cache=True`

这有利于开发阶段确保拿到最新页面，但长期增量会重复下载大量未变化内容。

生产增量应优先：

- Feed/Sitemap delta；
- ETag / `If-None-Match`；
- Last-Modified / `If-Modified-Since`；
- 304 只更新 `observed_at`；
- 200 后先比较 response/content hash；
- 只有内容实质变化才生成新 `article_version`。

Browser 页面无法使用标准 304 机制时，也应使用发现端点的 checkpoint、URL 新增判断和较低复查频率减少成本。

## 12. `get_last_day_data()` 的时间语义问题

原文：

```python
last_day = (date.today() - timedelta(days=1)).strftime('%Y-%m-%d')
return [v for v in datas.values() if last_day in v['date']]
```

这里把“业务输出昨天新闻”与采集数据混在一起，而且依赖字符串包含关系。风险包括：

- 来源时间格式变化；
- 时区不明确；
- 新闻发布时间和抓取时间混淆；
- 跨时区站点“昨天”不同；
- 只有日期、无 timezone provenance。

知识库应保存：

- `published_at`：来源声明发布时间，可空；
- `updated_at`：来源更新时间，可空；
- `first_seen_at`：系统首次发现时间；
- `captured_at`：抓取时间；
- `source_timezone / timezone_confidence`；
- 原始时间字符串和解析器 release。

“昨天的文章”应该是 Search/Export 层的查询：

```text
WHERE published_at >= window_start
  AND published_at < window_end
```

窗口由调用者时区确定，而不是写死在 crawler 内部。

## 13. 失败处理与可观测性不足

代码大量使用：

```python
except Exception:
    print(...)
```

并通过 `assert result.success` 判断抓取成功。

生产系统需要明确错误分类：

- DNS/connect/read timeout；
- 429；
- 5xx；
- 404/410；
- robots denied；
- browser crash；
- selector miss；
- challenge/login/paywall；
- parse/quality failure。

每一次真实尝试都要追加 `fetch_attempt`，记录：

- transport；
- worker/generation；
- status code；
- error class；
- bytes；
- retry decision；
- next_attempt_at；
- snapshot_id；
- Browser 升级原因。

不要用 `assert` 承担生产状态控制，因为优化模式可能关闭 assert，而且失败原因也不可结构化查询。

## 14. 详情抽取会丢失复杂 Markdown 结构

原文详情页主要提取：

```text
div.detail-content p -> text
```

再用换行拼接。这对于纯新闻段落够用，但技术博客知识库会丢失：

- 标题层级；
- 列表；
- fenced code block 和语言；
- 表格；
- 图片；
- 链接；
- blockquote；
- 数学公式；
- figure/caption。

生产系统应构造 Article IR/Markdown AST，而不是把正文压平为字符串。推荐过程：

```text
DOM Snapshot
  -> Content Region Candidate
  -> Article IR
     heading / paragraph / list / code / table / image / quote
  -> Normalize
  -> Markdown Serializer
  -> Markdown Parse Validation
```

这样抽取规则和 Markdown 序列化可以独立版本化。

## 15. 对 1000 技术博客方案的直接优化结论

### 15.1 新增 `site_source/discovery_endpoint`

同一站点可维护多个栏目、归档页、Sitemap、Feed、API。每个端点独立配置：

- provider 类型；
- capability；
- checkpoint；
- schedule；
- action plan；
- category/locale；
- enabled 状态。

### 15.2 新增版本化 `browser_action_plan_release`

动态发现不直接存任意 JS，而以受限 DSL 描述：

- wait；
- scroll；
- click；
- paginate；
- extract links；
- stop conditions；
- budgets。

每次执行保存 action trace 和停止原因。

### 15.3 Discovery 证据必须多值保存

同一 URL 来自多个栏目、Feed、Sitemap 时，全部保留 `discovery_edge`，不要用一个字典 category 覆盖。

### 15.4 URL 去重升级为规范化 + 数据库唯一约束

使用：

```text
url_raw
 -> URL Canonicalizer release
 -> url_normalized
 -> sha256(url_normalized)
 -> UNIQUE(site_id, url_hash)
```

再用 canonical/content 证据解析 `article_id`。

### 15.5 “近期窗口”与“历史全量”严格区分

文章展示的动态栏目最多加载 100 条，本质是 `windowed provider`，不能证明历史全量。历史回灌必须继续查：

- Sitemap；
- 年/月归档；
- 分类分页；
- 平台 API；
- 公开历史索引；
- Archive provider（可选）。

### 15.6 Provider 连续性必须持久化

对近期新闻流记录：

- last cursor/GUID；
- watermark；
- last_success_at；
- oldest_visible_at；
- retention_hint；
- overlap window；
- continuity state。

若停机时间超过可见窗口，自动 `gap_suspected -> reconciliation`，不能认为下一次抓到新数据就恢复完整。

### 15.7 Browser 是稀缺资源池，不是每 URL 新建

由独立 Browser Worker Pool 负责：

- 长生命周期 browser；
- 资源阈值；
- generation recycle；
- 域限流；
- 动作预算；
- 捕获 rendered DOM 以供离线 replay。

### 15.8 日期窗口只用于查询/发布，不用于 canonical 数据裁剪

知识库始终保留完整历史；“昨天/最近 24 小时/本周”都属于搜索、报告或发布层 view。

## 16. 推荐的生产数据流

```text
Site
  -> Site Source / Discovery Endpoint
      -> Sitemap / Feed / Archive / API / HTML / Browser Action Provider
          -> checkpoint + continuity
          -> Discovery Run
          -> URL Canonicalizer
          -> Durable Frontier (PostgreSQL UNIQUE)
          -> Admission / Robots / Egress
          -> Fetch Task
              -> HTTP first
              -> Browser if evidence requires
          -> Immutable Snapshot (S3/MinIO)
          -> Extraction Contract Release
          -> Metadata Candidates + Article IR
          -> Resolver + Quality Gate
          -> Article / Article Version
          -> Markdown Serializer
          -> Search / Vector / Publish async
```

这一结构既覆盖原文的“多栏目 + 动态加载 + 结构化抽取 + 增量去重”，又把它升级为可以支撑 1000 站点、数百万 URL、长期运行的 durable architecture。

## 17. Web 管理端需要为动态站点增加的功能

除了站点、任务、错误和 Markdown 预览，还应支持：

- 一个站点下维护多个 Discovery Endpoint；
- 显示每个 endpoint 的 provider type/capability；
- Browser Action Plan 可视化编辑与 dry-run；
- 动作执行 trace：滚动次数、点击次数、新增链接数、停止原因；
- endpoint checkpoint/watermark/连续性；
- 同一 URL 的多来源发现证据；
- 动态列表“连续无增长”告警；
- selector 命中率和 extraction drift；
- endpoint 级同步频率、预算和 Browser 升级率。

## 18. 最终判断

这篇文章证明了 Crawl4AI 在“动态栏目页 + CSS schema 结构化抽取”场景中的实用性，也展示了最小增量去重的实现方式。但其设计边界是单机、近期聚合和少量来源：100 条目标上限无法证明历史全量，本地 JSONL history 不能作为分布式 durable state，逐条 Browser 抓取吞吐低，MD5 原 URL 去重无法解决 canonical/版本/内容重复，日期字符串过滤也不适合作为知识库时间模型。

对博客知识库方案最重要的落地变化是：**将站点内部的栏目/归档/API/Feed 提升为 `site_source/discovery_endpoint` 一等实体；将动态下拉/加载更多提升为有界、版本化、可审计的 Browser Action Plan；将本地 URL-MD5 历史升级为规范化 URL + PostgreSQL 唯一约束 + 文章身份/版本模型；将“昨天新闻”降为查询视图。**

这样才能在新增站点和不断变化的网页模板下，持续扩展而不把系统退化成 1000 份互相独立、无法运维的爬虫脚本。