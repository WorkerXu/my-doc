# 基于 Crawl4AI 的 Substack 抓取器

## 1. 调研对象

- 项目：`fullatron/crawl4AI-substack-scraper`
- 地址：https://github.com/fullatron/crawl4AI-substack-scraper
- 调研固定版本：`bb01ccf7a2d4b8efd3454458b7211b1bf4fe4239`
- 主要代码：`main.py`
- 技术栈：FastAPI、Crawl4AI、Chromium、OpenAI-compatible LLM client

该项目是一个面向 Substack 的小型抓取 API。它把“平台特征识别、归档页发现、文章正文抽取、浏览器交互、批量抓取”集中在单个 FastAPI 服务中。对于 1000 个技术博客的知识库系统，它最有价值的不是直接复用整套服务，而是证明了一个重要工程方向：**常见博客平台应该抽象为可复用、可版本化的 Platform Profile，而不是每个站点都写独立爬虫，也不能完全依赖无差别的通用深爬。**

## 2. 项目实现拆解

### 2.1 页面类型识别

代码通过 URL path 做轻量分类：

- 路径包含 `/p/` -> `article`
- 根路径或 `/archive` -> `homepage`
- 其它 -> `unknown`

随后按页面类型选择 CSS：

- 文章：`.available-content`
- 归档/主页：`.portable-archive-list`
- 未知页面：不限定 selector，回退整页

这是一种“URL 规则 + DOM selector”的平台知识。它比对所有页面跑相同通用正文抽取更节省噪音处理成本，也说明平台模板可以同时承载 page type、URL pattern、selector 和 discovery recipe。

但生产系统不能只依赖 URL path。平台升级、自定义域名、国际化路径、A/B DOM 都可能导致误判，因此应组合：URL pattern、DOM 特征、JSON-LD/OG 元数据、selector 命中率和历史模板指纹，并保存分类置信度。

### 2.2 Browser Capture

项目为每次抓取配置 Chromium：

- `headless=True`
- 固定浏览器 User-Agent
- `enable_stealth=True`
- `magic=True`
- `scan_full_page=True`
- `js_code` 执行弹窗处理
- `wait_for="css:body"`
- `delay_before_return_html=5`
- `page_timeout=60000`
- `CacheMode.BYPASS`

这一实现适合演示“动态页面一定能够等到 DOM 后再抽取”，但不适合作为大规模系统默认路径。1000 个站点长期同步时应 HTTP 优先，只有 CSR、滚动加载、受控交互或 HTTP DOM 明显缺正文时才升级 Browser。

### 2.3 订阅弹窗处理

项目注入 JavaScript，尝试点击多个 close/dismiss selector，并移除 overlay/dialog 等元素，最后恢复页面滚动。

这个思路适合抽象成 **Browser Action Profile**：把站点/平台需要的等待、点击、滚动、关闭遮罩动作版本化，而不是散落在业务代码中。

但生产方案必须增加边界：

- 只允许关闭遮挡公开内容的订阅/营销弹窗；
- 不通过删除 `paywall`、登录墙、验证码等 DOM 来规避访问控制；
- 如果正文因权限不可获得，应记录 `access_restricted`，而不是把视觉层删除后视为抓取成功；
- Browser action 的每一步、执行时长、网络请求和目标 selector 都应受预算和审计约束。

### 2.4 归档页发现

`/scrape-all` 把 base URL 规范化为 `scheme://host`，然后访问：

```text
/archive?sort=new
```

归档页使用 `.portable-archive-list` selector，从 `result.links.internal` 中筛选包含 `/p/` 的链接，按出现顺序去重，并保留 archive link text 作为标题候选。

这里有三个值得复用的模式：

1. **平台专属归档入口**：已知平台可以直接尝试高价值入口，不必先盲目深爬。
2. **文章 URL pattern**：平台 profile 可以快速排除分类页、作者页、设置页等非文章 URL。
3. **列表页元数据候选**：归档 link text 可以作为 title 的低置信度候选，为正文页 metadata 缺失提供 fallback，同时保留 provenance。

### 2.5 `scan_full_page` 的真实语义

Crawl4AI 官方文档说明，`scan_full_page=True` 会尝试从页面顶部滚到底部，主要用于触发懒加载以及“内容持续追加”的传统滚动场景。官方同时区分 Virtual Scroll：如果页面滚动时旧 DOM 节点会被替换，单纯 `scan_full_page` 不能保证最终 DOM 含有全部历史条目，应使用 `VirtualScrollConfig` 把多次窗口内容合并。

因此生产系统不能把“开启 scan_full_page”当作“全量历史已发现”的证明。归档 Provider 必须显式声明滚动模式：

```text
none
pagination
full_page_append
virtual_scroll
load_more_button
custom
```

并配置 max scroll、max pages、no-new-url stop、重复 cursor stop、预算耗尽状态和 end-of-archive 证据。

### 2.6 批量抓取与 Browser 复用

项目发现 URL 后，再创建一个 `AsyncWebCrawler`，在同一个 crawler 生命周期里顺序抓取多篇文章，而不是每篇文章都重新启动 Chromium。

这说明 Browser 启动成本可以通过池化和站点亲和性摊薄。但生产实现应更严格：

- 复用 Browser 进程；
- 默认不跨站复用浏览器上下文、Cookie 和 localStorage；
- 同站点可配置是否复用 context；
- 设 `max_pages_per_process`、`max_process_age`、RSS 阈值和失败后回收；
- 同一域仍受分布式限流约束；
- Browser Worker 只执行队列 job，不在 Web 请求里跑整批历史抓取。

### 2.7 标题 fallback

项目标题来源顺序为：

1. HTML `<title>`；
2. Markdown 一级标题；
3. 归档列表 link text。

虽然代码用正则解析 `<title>` 不是生产级 DOM 解析方式，但“多来源候选 + fallback”的思想是正确的。主系统应把这些候选写为 metadata provenance，最后由规则/质量门禁选择，而不是静默覆盖。

### 2.8 LLM 摘要

项目可把 Markdown 传给 OpenAI-compatible API 摘要，并把输入截断到约 24000 字符。

这不能进入知识库 canonical ingestion 主链路：

- 长文截断会丢信息；
- LLM 输出不可替代原始文章；
- 同步 LLM client 在 async 路由中直接调用会占用事件循环线程；
- 模型、prompt、供应商变化会导致不可复现输出。

合理做法是独立 `Enrichment` 阶段：在文章版本发布后异步生成摘要/标签/embedding，并记录 `article_version_id + model + prompt_version + input_hash + output_hash`。原 Markdown 永远保持确定性和可回放。

## 3. 项目对 1000 站知识库最有价值的能力

### 3.1 Platform Profile 层

现有“通用规则 + Custom Adapter”之间还需要一个可复用的中间层：`Platform Profile`。

例如 Substack Profile 可包含：

```yaml
platform: substack
fingerprint:
  url_hosts: ["*.substack.com"]
  dom_any: [".portable-archive-list", ".available-content"]
page_types:
  article:
    url_regex: "/p/"
    content_selectors: [".available-content", ".body.markup"]
  archive:
    paths: ["/archive?sort=new"]
    list_selector: ".portable-archive-list"
discovery:
  article_url_regex: "/p/"
  ordered_newest_first: true
  scroll_mode: full_page_append
browser_actions:
  dismiss_public_subscription_modal: true
metadata:
  archive_link_text_as_title_candidate: true
```

站点绑定 Platform Profile 后仍允许 per-site override；Profile 升级也不能直接影响生产，必须经过 golden regression 和 canary。

### 3.2 Ordered Archive Provider

Substack 的归档按 newest-first 排序，可泛化为 `OrderedArchiveProvider`。

首次 backfill：持续翻页/滚动直到检测到归档终点或预算耗尽。

日常增量：从头扫描，遇到连续 N 个“已知且内容 identity 稳定”的文章后可以提前停止；低频 reconciliation 仍做更深扫描，防止旧文补录、排序变化和归档修复。

建议 checkpoint：

```text
source_id
profile_release_id
newest_article_key
newest_published_at
last_cursor
head_signature
known_streak_stop_threshold
last_full_reconciliation_at
end_condition
```

必须区分：`completed_end_reached`、`completed_known_streak`、`partial_budget_exhausted`、`partial_scroll_limit`、`failed_selector_drift`。

### 3.3 Browser Scroll Capability

平台 Profile 不能只有“要不要 Browser”，而应明确 Browser capability：

```text
render_required
wait_condition
scroll_mode
scroll_container
scroll_count/max_scrolls
scroll_delay
load_more_selector
stop_when_no_new_urls
post_load_actions
```

这样才能把 Crawl4AI 的 `scan_full_page`、Virtual Scroll、JS hook 等能力纳入可审计、可测试的抓取策略。

### 3.4 Browser Site Affinity

对于 Browser-heavy 的平台，可把同一站点的短批任务调度到同一 Browser Worker，以复用 Browser process，同时维持：

- site/domain rate limit；
- 每站隔离 context；
- 进程级 RSS 监控；
- batch size 上限；
- worker recycle；
- crash 后 item 级重试。

这比“每 URL 启动浏览器”大幅降低固定开销，又不会像“全平台共享一个有状态 context”那样造成 Cookie/缓存/隐私串站。

## 4. 不能直接照搬到生产的部分

### 4.1 Browser-only 成本过高

项目无 HTTP 快路径，连归档和文章都直接 Chromium。对 1000 站长期运行不经济。主系统应先 `httpx/aiohttp` 获取原始 response bytes，只有输入不完整才 Browser。

### 4.2 没有 durable job 与状态真相源

`/scrape-all` 在一次 HTTP 请求中完成发现、抓取、可选 LLM 摘要。服务进程崩溃、请求断开、重启都会丢任务状态，也没有断点续跑、重试、取消、优先级和 backpressure。

主系统必须继续使用 PostgreSQL job/job_item/outbox + Redis Streams，Web 请求只创建任务并立即返回 job_id。

### 4.3 “全量”没有可证明终点

接口 `limit` 最大 100；发现结果取决于单次归档页面滚动后的 links。没有分页 cursor、scroll end 证据、URL coverage、历史 source attribution，也没有 Common Crawl/Wayback 补充。因此它适合演示批抓，不等于生产级“全量历史”。

### 4.4 没有增量同步

`CacheMode.BYPASS` 每次都新抓，没有 ETag、Last-Modified、304、body hash、article version hash、archive checkpoint 或 known-streak fast stop。

主系统的增量真相应由自己的 checkpoint 和 immutable snapshot 管理，不能依赖 Crawl4AI 本地 cache。

### 4.5 缺少 robots、限流与 SSRF 边界

API 直接接收 URL 并启动 Browser，没有看到统一的 host allowlist、private IP 防护、逐 redirect 校验、robots、domain limiter 和请求/字节预算。对于 Web 管理系统允许用户录入站点的场景，这是生产阻断项。

### 4.6 DOM 操作存在访问控制边界风险

脚本包含 `[class*="paywall"]` 等移除逻辑。生产系统不得把这类 DOM 删除作为绕过付费墙、登录或访问控制的策略。权限不可得时应显式失败/跳过。

### 4.7 依赖未锁定

`requirements.txt` 只有包名，没有版本或 hash。抓取器、浏览器和抽取库升级可能引入 DOM 行为、Markdown 输出、Stealth 行为或资源使用变化。生产构建应使用 lockfile/固定版本、镜像 digest、SBOM、contract test 和 canary。

### 4.8 顺序抓取吞吐有限

文章循环逐个 `await crawler.arun()`，不会压垮目标站，但对大规模 backfill 吞吐低。主系统应让 Scheduler 跨域并行、域内受控并发，而不是在单个 API handler 中串行完成所有工作。

## 5. 主技术方案应落地的优化

1. 增加 `Platform Profile / Profile Release`，覆盖平台 fingerprint、page type、URL pattern、discovery recipe、selectors、Browser actions、scroll capability 和 identity hints。
2. 增加 `site_platform_bindings`，允许站点采用某个 Profile release 并做少量 override。
3. Discovery 增加 `OrderedArchiveProvider`，支持 newest-first 快速增量停止与低频完整 reconciliation。
4. Browser Capture 增加 `scroll_mode` capability，明确区分普通 full-page append、virtual scroll、pagination、load-more 和 custom interaction。
5. Browser Worker 增加进程池与 site affinity；复用 browser process，不默认复用跨站 context。
6. Browser action profile 版本化、审计化，禁止用于绕过登录/付费墙/验证码。
7. Golden Workbench 增加 Profile release 回归：selector 命中率、archive URL 数、page type 分类准确率、Browser action 成功率和 Markdown diff。
8. 增加 Profile drift 告警：同平台多个站点同时出现 selector zero-hit、archive discovered_count 断崖、Browser action timeout 时，优先判断平台级模板变更。
9. Metadata provenance 增加 `archive_link_text` 等列表页候选来源。
10. LLM 摘要/标签作为独立 Enrichment job，不阻塞采集，不影响 canonical Markdown。
11. 依赖和 Crawl4AI release 固定版本/digest，任何升级先 contract test + golden shadow + canary。
12. 继续坚持 HTTP raw-byte snapshot 为权威输入；Platform Profile 只决定路由和候选，不成为不可回放的状态中心。

## 6. 建议数据模型补充

```text
platform_profiles
- id
- platform_name
- release_version
- status(testing/active/retired)
- fingerprint_json
- page_type_rules_json
- discovery_recipes_json
- extract_defaults_json
- browser_actions_json
- browser_capabilities_json
- identity_hints_json
- source_ref
- artifact_digest
- created_at

site_platform_bindings
- site_id
- platform_profile_id
- confidence
- detected_evidence_json
- override_json
- status(candidate/approved/disabled)
- approved_by / approved_at

profile_regression_runs
- id
- platform_profile_id
- golden_set_version
- sites_tested
- selector_hit_rate
- page_type_accuracy
- discovered_url_delta
- markdown_diff_summary
- browser_action_success_rate
- status(pass/fail)
- created_at
```

`discovery_sources` 建议增加：

```text
profile_release_id
ordered_stable
scroll_mode
end_condition
```

`discovery_checkpoints.cursor_json` 对 ordered archive 至少能保存：

```json
{
  "newest_article_key": "...",
  "head_signature": "...",
  "last_cursor": "...",
  "known_streak": 0,
  "end_condition": "completed_known_streak"
}
```

## 7. Substack Profile 的生产级执行流程

```text
Site probing
 -> host / DOM fingerprint
 -> candidate: substack profile
 -> sample root + /archive + /p/... golden URLs
 -> validate selectors/page type/archive ordering
 -> approve profile binding

Backfill
 -> OrderedArchiveProvider(/archive?sort=new)
 -> choose scroll mode by profile release
 -> bounded scroll/pagination
 -> /p/ URL filter
 -> Frontier UPSERT + archive title metadata candidate
 -> HTTP Fetch first
 -> Browser only when content incomplete
 -> site selector / generic extractor
 -> Quality Gate
 -> Article IR
 -> deterministic Markdown

Incremental
 -> scan archive head
 -> emit unseen/newly changed URLs
 -> N consecutive known stable entries -> fast stop
 -> conditional HTTP GET
 -> 304/body hash unchanged -> stop
 -> changed -> new snapshot/version

Reconciliation
 -> deeper archive pass
 -> verify old URL visibility/migrations
 -> optional Common Crawl / Wayback evidence
```

## 8. 验收与回归测试

Platform Profile 上线前至少验证：

1. 自定义 `substack.com` 域名与 custom domain 都能识别，且不会误绑定无关站点。
2. `/p/` URL 规则不会把导航页当文章。
3. `.portable-archive-list` 失效时能触发 profile drift，而不是静默发现 0 篇。
4. append scroll 与 virtual scroll 使用不同 capability；无法证明终点时状态为 partial。
5. 归档 link text 只作为 metadata candidate，不能覆盖正文页高置信度 title。
6. Browser action 仅关闭公开订阅遮罩；遇到登录、付费或验证码返回受限状态。
7. 同一 browser process 可复用，但跨站 Cookie/localStorage 不泄漏。
8. Browser process 达到页数、年龄或 RSS 阈值后能安全回收并继续 job。
9. 首次 backfill 中途崩溃后可从 checkpoint 继续，不重复发布文章版本。
10. 日常增量在 archive newest-first 稳定时能 known-streak 快停；低频 reconciliation 能发现旧文补录。
11. Crawl4AI/Profile release 升级后必须在 golden set 做 selector、URL coverage、Markdown、资源消耗回归。
12. LLM enrichment 失败不会影响 Markdown 发布。

## 9. 参考源码与文档

- 项目 README：https://github.com/fullatron/crawl4AI-substack-scraper/blob/bb01ccf7a2d4b8efd3454458b7211b1bf4fe4239/README.md
- 项目主代码：https://github.com/fullatron/crawl4AI-substack-scraper/blob/bb01ccf7a2d4b8efd3454458b7211b1bf4fe4239/main.py
- Crawl4AI Lazy Loading：https://github.com/unclecode/crawl4ai/blob/7e801521428ee12509994d39151006f64055ebe3/docs/md_v2/advanced/lazy-loading.md
- Crawl4AI Virtual Scroll：https://github.com/unclecode/crawl4ai/blob/7e801521428ee12509994d39151006f64055ebe3/docs/md_v2/advanced/virtual-scroll.md
