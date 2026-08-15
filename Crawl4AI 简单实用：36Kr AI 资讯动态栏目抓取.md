# Crawl4AI 简单实用：36Kr AI 资讯动态栏目抓取

## 1. 调研对象

- 编号：45
- 原文：Crawl4AI简单实用：36Kr AI 资讯动态栏目抓取
- 地址：https://adg.csdn.net/69706db7437a6b40336a3679.html
- 原文发布时间：2025-04-18
- 原文使用版本：Crawl4AI 0.5.0.post8
- 本次调研目的：评估“JavaScript 动态列表 + 滚动加载”模式如何用于 1000+ 技术博客的历史 URL 发现和后续增量同步，并将可复用能力纳入博客知识库平台。

## 2. 原文方案在做什么

原文针对 36Kr AI 资讯动态栏目 `https://36kr.com/information/AI/`。该列表页不是只依赖首屏静态 HTML，而是由浏览器执行 JavaScript 后生成/补充内容，因此普通 HTTP 请求容易拿不到完整列表。

原文的核心流程可以概括为：

```text
启动 AsyncWebCrawler
 -> 打开 36Kr AI 栏目页
 -> 执行一次 window.scrollTo(0, document.body.scrollHeight)
 -> 等待 .information-flow-list 出现
 -> 获取 CrawlResult
 -> 将整个 CrawlResult 序列化为 news.json
 -> 遍历 media.images
 -> 用图片 alt 作为新闻标题输出
```

关键调用思路是 `js_code + wait_for`：浏览器进入页面后执行滚动脚本，并等待列表容器存在，然后返回渲染后的抓取结果。

这个例子最有价值的地方不是“抓 36Kr”本身，而是验证了一类在技术博客、新闻站、Newsletter 聚合页中非常常见的发现方式：**文章入口 URL 只在 JavaScript 执行、滚动、点击 Load More 或虚拟列表更新后出现。**

## 3. 技术原理

### 3.1 为什么静态 HTTP 会漏 URL

动态列表站点通常存在以下几种实现：

1. 首屏 HTML 只有壳结构，列表由 XHR/fetch 请求后插入 DOM；
2. 首屏返回部分文章，滚动到底部再请求下一批；
3. 点击“加载更多”后追加文章；
4. 使用虚拟列表，只在 DOM 中保留当前可视区域附近的元素，旧元素会被替换掉；
5. 页面通过客户端路由或 hydration 后才生成真实链接。

如果 URL discovery 只解析服务器返回的初始 HTML，就可能把这类站点误判为“只有几篇文章”。因此动态列表不是正文抽取问题，而首先是 **Coverage / Discovery 问题**。

### 3.2 Browser 渲染的作用

Crawl4AI 的浏览器路径底层依赖 Chromium/Playwright 类浏览器能力。浏览器会执行页面 JavaScript，触发网络请求、DOM 更新、滚动事件等，因此可以看到普通 HTTP 抓取看不到的链接。

原文使用：

- `js_code`：在页面环境中执行 JavaScript；
- `wait_for`：在返回结果前等待目标条件；
- `AsyncWebCrawler`：异步驱动浏览器抓取。

在当前 Crawl4AI 文档中，更推荐把行为放进 `CrawlerRunConfig`，并区分：

- `js_code_before_wait`：先触发点击/滚动，再等待条件；
- `wait_for="css:..."`：等待 CSS 条件；
- `wait_for="js:..."`：等待 JavaScript 条件；
- `js_only=True + session_id`：同一个浏览器会话中做多步交互；
- `VirtualScrollConfig`：处理真正的虚拟滚动列表；
- `scan_full_page`：更适合元素持续追加、懒加载的长页面。

### 3.3 “列表出现”与“历史文章发现完成”是两回事

原文等待 `.information-flow-list` 出现，只能说明列表容器已经存在，并不能证明：

- 下一批已经加载；
- 已经滚动到真正底部；
- 所有分页已经遍历；
- 历史数据已经穷尽；
- 当前列表没有因为虚拟滚动而替换旧元素。

同理，一次 `window.scrollTo(...bottom)` 只是一条交互动作，不是 Coverage 完成条件。

对生产知识库而言，真正的结束条件必须是**可审计的发现收敛证据**，例如：

```text
连续 N 轮滚动/点击后：
  新增 normalized article URL = 0
且页面没有显式 next cursor / next page
且达到站点配置的边界或历史时间边界
```

同时要记录为什么停止：`NO_GROWTH`、`CURSOR_EXHAUSTED`、`MAX_ROUNDS`、`TIME_BUDGET`、`MAX_ITEMS`、`OVERLAP_BOUNDARY` 等。

## 4. 原文方案的优点

### 4.1 极小代码证明了动态页面可抓

对于新站点 Probe，非常适合先用短脚本验证：

- 是否必须 Browser；
- 页面是否需要滚动；
- 哪个列表 selector 可作为 ready signal；
- JavaScript 执行后是否出现文章链接。

### 4.2 提供了一个可配置 Recipe 的雏形

原文中的：

```text
URL + js_code + wait_for
```

本质上已经接近生产平台中的声明式 Recipe。平台不应该为每个站点写 Python 爬虫类，而应该把差异收敛为：

```yaml
listing_recipe:
  start_url: ...
  ready_selector: ...
  action: scroll | click_more | paginate | virtual_scroll
  item_selector: ...
  link_selector: ...
  max_rounds: ...
```

这样第 1001 个站点通常只需要配置和验收，而不是新增代码。

## 5. 原文方案不能直接用于生产的地方

### 5.1 一次滚动不足以做历史全量发现

动态 feed 往往按批次加载。一次到底可能只触发一批，也可能因为页面还没稳定就返回。历史有几千篇时，更需要循环滚动/分页、增量去重与收敛判断。

### 5.2 用 `media.images[].alt` 推断文章身份过于脆弱

原文最终遍历 `news_data["media"]["images"]`，再通过图片 URL 前缀过滤并打印 `alt`。这种做法存在几个问题：

- 图片不是文章主键；
- 一篇文章可能无图、多图或图片 alt 为空；
- 图片 CDN 域名可以随时改变；
- `alt` 是展示文本，不一定等于标题；
- 没有得到文章 URL，就无法进入正文抓取；
- 广告图、推荐图、头像都可能混入。

生产 discovery 应优先抽取**真实 `<a href>`、标题、发布时间提示和父列表页关系**。

当前 Crawl4AI 结果本身可以提供 `result.links["internal"]`，也可以通过 `JsonCssExtractionStrategy` 对重复列表项进行结构化提取，将 `href` 作为核心字段，而不是从图片反推文章。

### 5.3 序列化整个 CrawlResult 到 JSON 成本高且职责混乱

原文把整个 `result.model_dump_json()` 写入 `news.json`，用于后续找标题。对于大规模平台，这会造成：

- 大量与 URL discovery 无关的 DOM/Markdown/media 元数据重复存储；
- 下游必须理解 Crawl4AI 私有 Result schema；
- 工具升级容易造成数据契约变化；
- 无法清楚区分“网络/渲染事实”和“业务 URL Observation”。

更合理的边界是：

```text
Crawler Result
 -> Adapter
 -> 标准化 DiscoveryItem[]
 -> URL Normalize / Scope / Dedupe
 -> URL Observation
```

原始 Render Artifact 仍可保存到对象存储用于回放，但业务层不要把工具的 Result 对象当领域模型。

### 5.4 缺少增量 checkpoint

动态列表适合“新到旧”排列。增量同步时没有必要每次滚完整历史，可以从顶部开始，在遇到最近已知 URL/时间 overlap 后停止。

平台需要保存的是：

- 已知 URL 集合和最近 Observation；
- provider cursor（如果页面/API 有）；
- 最近一批内容时间窗口；
- listing recipe release；
- 收敛证据。

**不应把 Browser session 或 crawler frontier 作为业务 checkpoint。** Browser 进程丢失后，任务必须能从 PostgreSQL 的事实恢复。

### 5.5 缺少并发、限流和预算

1000 个站点如果都使用 Browser 滚动会非常昂贵。动态 Browser discovery 应放在 Provider fallback 序列的后部：

```text
CMS/API
> Sitemap
> Feed
> 静态 Archive/Pagination
> 轻量 Prefetch Mapping
> Dynamic Listing Browser
> bounded Deep Crawl
```

只有权威/静态发现不足时才升级到 Browser。

### 5.6 缺少漂移检测

`.information-flow-list`、列表项 selector、链接结构都有可能改版。若 selector 失效但 HTTP 仍是 200，最危险的结果不是显式报错，而是“成功抓到一个空列表”。

因此要监控：

- ready selector 命中率；
- 每轮 item 数量；
- 每轮新增唯一文章 URL 数；
- article-link / all-links 比例；
- 重复率；
- browser fallback 比例；
- 与历史基线相比的 URL 产出骤降。

## 6. 生产级 Dynamic Listing Discovery 设计

### 6.1 抽象为独立 Provider

建议在平台 Discovery Provider 中增加：

```text
DYNAMIC_LISTING
```

它不是“正文 Browser Fetch”的别名，而是专门负责从动态归档页、栏目页、Infinite Scroll 页面发现文章 URL。

输入：

```text
SourceProfile + DynamicListingRecipe + Run budget
```

输出：

```text
DiscoveryItem[] + CoverageEvidence + RenderTrace
```

### 6.2 Recipe 数据模型

示例：

```yaml
discovery:
  providers:
    - type: dynamic_listing
      priority: 87
      mode: gap_only
      start_urls:
        - https://example.com/blog
      recipe:
        ready_selector: ".information-flow-list"
        list_item_selector: ".information-flow-list article"
        link_selector: "a[href]"
        interaction_mode: append_scroll
        max_rounds: 40
        max_runtime_seconds: 120
        max_items: 5000
        no_growth_rounds: 3
        wait_after_action_ms: 800
        extract:
          url: "a::attr(href)"
          title: "a::text"
          published_hint: "time::attr(datetime)"
```

`interaction_mode` 至少支持：

```text
STATIC
PAGINATION
CLICK_MORE
APPEND_SCROLL
VIRTUAL_SCROLL
```

### 6.3 当前 Crawl4AI 的执行映射

建议 Adapter 映射如下：

```text
STATIC
 -> HTTP / Crawl4AI 普通页面解析

APPEND_SCROLL
 -> CrawlerRunConfig + scan_full_page 或受控 js_code 滚动

CLICK_MORE
 -> session_id + js_only + click + wait_for 新 item 条件

VIRTUAL_SCROLL
 -> VirtualScrollConfig(container_selector, scroll_count, scroll_by, wait_after_scroll)

PAGINATION
 -> 提取 next href，重新进入 URL pipeline
```

不要把 Crawl4AI 配置直接暴露给 Web 用户。Web 只编辑平台 Recipe；Adapter 负责翻译成当前工具版本的参数，以便未来更换 Crawl4AI/Playwright 时领域模型不变。

### 6.4 结构化提取而不是图片启发式

动态列表的最小业务结果应是：

```json
{
  "observed_url": "https://example.com/post/123",
  "title_hint": "...",
  "published_hint": "2026-08-01T10:00:00Z",
  "parent_url": "https://example.com/blog",
  "round": 7,
  "provider": "DYNAMIC_LISTING"
}
```

若页面存在稳定 DOM，可使用 `JsonCssExtractionStrategy` 直接抽取列表项；若 selector 不稳定，可以先使用 `result.links["internal"]`，再通过 Source include/exclude/path classifier 筛选文章候选。

### 6.5 收敛算法

伪代码：

```python
seen_this_run = set()
no_growth = 0

for round_no in range(recipe.max_rounds):
    items = await harvest_current_dom()

    normalized = normalize_scope_filter(items)
    new_urls = normalized - seen_this_run
    seen_this_run |= new_urls

    persist_url_observations(new_urls, round_no)

    if reached_incremental_overlap_boundary(new_urls):
        stop("OVERLAP_BOUNDARY")
        break

    if len(new_urls) == 0:
        no_growth += 1
    else:
        no_growth = 0

    if no_growth >= recipe.no_growth_rounds:
        stop("NO_GROWTH")
        break

    await perform_next_action()
    await wait_for_growth_or_timeout()
else:
    stop("MAX_ROUNDS")
```

关键点不是比较 DOM 高度，而是比较**规范化、通过 Scope 的文章候选 URL 是否继续增长**。DOM 高度可以作为辅助信号，但不能成为 Coverage 真相。

### 6.6 对虚拟列表的特殊处理

虚拟列表与普通无限滚动不同：旧 item 会从 DOM 消失。如果每轮只在最终 DOM 上做一次提取，会丢掉之前出现过的文章。

处理方式：

- 使用 Crawl4AI `VirtualScrollConfig` 时，让 Adapter 接收其合并后的结果；
- 或在每轮滚动后即时 harvest，写入 `seen_this_run`；
- 业务层仍以 URL Observation 为准，不依赖最终 DOM 是否还保留旧节点。

### 6.7 Coverage Evidence

建议为动态列表保存：

```text
provider_type = DYNAMIC_LISTING
start_url
recipe_release_id
interaction_mode
rounds
observed_items
unique_article_urls
new_urls
duplicate_ratio
first_published_hint
last_published_hint
selector_hit_rate
exhausted
exhaustion_reason
max_rounds
runtime_ms
render_artifact_id
```

这样 Web 管理台可以解释：“为什么认为这个栏目已抓到边界”以及“本次为什么只新增 3 篇”。

## 7. 增量同步设计

动态列表一般按发布时间倒序，因此可以做廉价增量：

```text
打开列表顶部
 -> 抓第一批链接
 -> normalize/dedupe
 -> 继续滚动
 -> 一旦跨过 overlap 时间窗口，且连续若干批无新 URL
 -> 停止
```

建议 Source 保存一个业务级 `incremental_watermark`，但 watermark 只能帮助缩小范围，不能单独承担正确性。应保留 12~48 小时 overlap，并通过 URL 去重吸收重复。

如果列表没有可靠时间，可以使用“连续 K 个已知 URL + 连续 N 轮无 unseen URL”的组合边界。

对于偶尔置顶旧文章、编辑后重新上浮的站点，要允许：

- overlap；
- 周期性低频深度复查；
- Sitemap/CMS/Archive 与动态列表 provider 交叉补漏。

## 8. 1000+ 站点规模下的资源模型

动态 Browser discovery 必须与正文 Browser fetch 拆分 Worker Pool，因为两者资源特征不同：

```text
Browser Discovery Worker
- 长页面
- 多轮交互
- 低并发
- 强 max-runtime / max-rounds

Browser Content Worker
- 单文章
- 生命周期短
- 更关注 DOM/正文完整度
```

调度仍由平台做 Global Domain Admission：

- per-domain token bucket；
- per-domain semaphore；
- Source fairness；
- Browser 总并发上限；
- Backfill 与 Incremental 分优先级；
- 超时/429/503 降速；
- 每 Source 的每日 Browser 秒数预算。

Crawl4AI 自带 Dispatcher/RateLimiter 可以作为单 Worker 内部保护，但不能替代跨 Pod、跨 Worker 的全局公平与限流。

## 9. 安全与可运维性

### 9.1 JS Recipe 必须声明式和白名单化

不要让 Web 管理员或外部 API 直接提交任意 JavaScript。平台可以提供有限动作：

```text
WAIT_SELECTOR
CLICK_SELECTOR
SCROLL_PAGE
SCROLL_CONTAINER
WAIT_ITEM_GROWTH
EXTRACT_LINKS
```

Adapter 再生成受控脚本。这样可以减少任意脚本执行、跨域导航和不可审计行为。

### 9.2 每次导航都重新过 Scope/SSRF

列表页提取出的 href、重定向目标、Browser 新导航、iframe 和资源 URL 都必须重新校验：

- allowed_hosts；
- scheme；
- 私网/metadata 地址；
- 重定向次数；
- URL 长度和协议白名单。

### 9.3 失败必须可解释

动态 discovery 错误码至少区分：

```text
READY_SELECTOR_TIMEOUT
NO_ARTICLE_LINK
ACTION_TIMEOUT
NO_GROWTH
MAX_ROUNDS_REACHED
VIRTUAL_SCROLL_UNSUPPORTED
SCOPE_REJECTED
RATE_LIMITED
BROWSER_CRASH
TEMPLATE_DRIFT
```

其中 `NO_GROWTH` 在达到预期边界时可以是正常 exhaustion reason，不一定是失败。

## 10. 对博客知识库技术方案的具体优化结论

本次调研不改变原方案“Authoritative First / HTTP First / Browser Last”的基本方向，而是补足一个此前不够显式的能力：**动态列表发现（Dynamic Listing Discovery）应成为独立的 Discovery Provider 和版本化 Recipe，而不是散落在正文 Browser 抓取中的几个 scroll/click 参数。**

应加入以下能力：

1. `DYNAMIC_LISTING` Provider，位于静态/权威 discovery 之后、bounded Deep Crawl 前后按成本策略启用；
2. `DynamicListingRecipe`，显式描述分页、Load More、append scroll、virtual scroll；
3. 对当前 Crawl4AI 的 `CrawlerRunConfig`、`js_only/session_id`、`scan_full_page`、`VirtualScrollConfig` 做 Adapter 映射；
4. 从真实 anchor href 或结构化 extraction 获取文章身份，禁止依赖图片 alt 作为主发现路径；
5. 以“新增 normalized article URL 收敛”为停止条件，辅以最大轮数、运行时和最大 item 预算；
6. 动态列表独立 Coverage Evidence 与 exhaustion reason；
7. 增量执行从列表顶部开始，以 overlap + 已知 URL 收敛快速停止；
8. listing selector、唯一 URL 产出、重复率进入 Drift Detector；
9. Browser Discovery 与 Browser Content 分 Worker Pool，并统一受 Global Domain Admission 和预算控制；
10. Web 管理台提供动态列表 Recipe 预览、每轮 URL 增长曲线、收敛原因与候选链接审核。

## 11. 参考资料

- 调研文章：https://adg.csdn.net/69706db7437a6b40336a3679.html
- Crawl4AI Page Interaction：https://docs.crawl4ai.com/core/page-interaction/
- Crawl4AI Virtual Scroll：https://docs.crawl4ai.com/advanced/virtual-scroll/
- Crawl4AI Crawler Result：https://docs.crawl4ai.com/core/crawler-result/
- Crawl4AI Extraction Strategies：https://docs.crawl4ai.com/extraction/no-llm-strategies/

## 12. 最终判断

该文章的代码本身是一个演示级最小样例，但它暴露了大规模博客知识库中一个非常关键的 Coverage 缺口：**动态栏目页可能是历史文章 URL 的唯一入口之一。**

正确的生产化方式不是把原文的一次滚动脚本直接复制到每个站点，而是把“动态列表如何产生 URL Observation、如何证明收敛、如何增量、如何限流、如何漂移检测”抽象成平台级 Provider + Recipe + Evidence。这样才能在 1000 个站点扩展到数千站点时仍保持可配置、可审计、可恢复和可控成本。
