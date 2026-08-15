# Crawl4AI 简单实用：36Kr AI 资讯动态栏目抓取

## 1. 调研对象

- 编号：45
- 名称：Crawl4AI 简单实用：36Kr AI 资讯动态栏目抓取
- 地址：https://adg.csdn.net/69706db7437a6b40336a3679.html
- 原文发布时间：2025-04-18
- 原文使用版本：Crawl4AI 0.5.0.post8
- 调研目标：分析 JavaScript 动态列表、滚动加载和 Crawl4AI 交互机制，判断其对“1000+ 技术博客历史全量抓取 + 增量同步 + Web 管理”平台的价值，并给出可生产化的设计。

## 2. 原文实现拆解

原文目标是抓取 36Kr AI 资讯动态栏目 `https://36kr.com/information/AI/`。列表内容依赖浏览器 JavaScript 渲染，普通 HTTP 请求只能看到不完整的初始页面，因此作者使用 `AsyncWebCrawler` 启动浏览器抓取。

原文核心逻辑可以概括为：

```text
AsyncWebCrawler
  -> 打开 36Kr AI 栏目
  -> js_code: window.scrollTo(0, document.body.scrollHeight)
  -> wait_for: document.querySelector('.information-flow-list')
  -> 得到 CrawlResult
  -> model_dump_json() 写入 news.json
  -> 遍历 media.images
  -> 根据图片 URL 前缀筛选
  -> 用 image.alt 作为新闻标题输出
```

原文代码证明了三件事：

1. 浏览器执行 JavaScript 后可以得到静态 HTTP 看不到的动态列表内容；
2. `js_code` / `wait_for` 可以控制页面交互与等待；
3. CrawlResult 会同时携带链接、媒体、Markdown 等多种结果，适合做快速原型。

但这只是“动态页面可以抓”的 PoC，不足以直接承担知识库的历史 Coverage、增量同步和数据身份管理。

## 3. 动态列表的技术原理

### 3.1 为什么初始 HTML 会漏文章

技术博客、资讯站和 Newsletter 聚合页常见以下实现：

- 首屏 HTML 只是壳，列表由 XHR/fetch 返回后插入 DOM；
- 首屏只返回前 N 条，滚动到底再请求下一批；
- 点击 Load More 后追加数据；
- 客户端路由或 hydration 后才生成真实链接；
- 虚拟列表只保留当前可视区域附近的 DOM 节点，旧节点会被回收；
- 页面将 cursor/page 参数藏在接口响应或客户端状态中。

因此，“正文抓取”之前必须先解决“文章 URL 是否被完整发现”的问题。对知识库而言，动态列表首先是 **Discovery / Coverage 问题**，其次才是 Browser Fetch 问题。

### 3.2 浏览器渲染为什么能看到更多内容

浏览器会执行页面 JavaScript、触发 IntersectionObserver、scroll/click 事件、发起 XHR/fetch，并等待 DOM 更新。Crawl4AI 底层通过浏览器自动化能力把这些动作包装成 crawler 配置，因此可以抓取动态列表。

生产系统不应该把这些工具参数直接暴露成业务模型。业务层应表达“语义动作”，例如：

```text
WAIT_READY
HARVEST
SCROLL_PAGE
WAIT_ITEM_GROWTH
HARVEST
CLICK_MORE
WAIT_ITEM_GROWTH
HARVEST
```

再由 Adapter/Recipe Compiler 翻译为具体 Crawl4AI 或 Playwright 调用。

## 4. 一个重要的版本兼容风险：0.5.x 到 0.9.x 的 JS 执行顺序变化

原文使用 Crawl4AI `0.5.0.post8`，其示例把 `js_code` 和 `wait_for` 放在同一次运行中，并按“执行滚动，再等待列表”的思路理解。

当前 Crawl4AI 0.9.x 文档中的页面执行管线已经明确区分：

```text
navigation
 -> js_code_before_wait
 -> wait_for
 -> delay
 -> js_code
 -> capture content
```

Crawl4AI 0.8.5 的发布说明也明确记录过这次管线顺序调整：`js_code` 被移动到 `wait_for` 之后；如果业务语义是“先触发交互，再等待结果”，应使用 `js_code_before_wait`。

这意味着：**不能把旧版本 Recipe 的参数名原样复制到新版本运行时。** 否则最危险的结果不是直接报错，而是等待条件提前满足、交互发生太晚、最终只抓到首批内容，却仍返回“成功”。对于历史全量抓取，这属于静默 Coverage 损失。

因此平台必须引入 Runtime Capability Contract 与 Recipe Compiler，而不仅仅是“锁版本”。

## 5. Runtime Capability Contract / Recipe Compiler

### 5.1 业务 Recipe 与工具配置彻底解耦

平台定义稳定的 Recipe IR：

```yaml
recipe:
  kind: dynamic_listing
  ready:
    type: css
    value: ".information-flow-list"
  interaction:
    mode: append_scroll
    action: SCROLL_PAGE
    wait_after_action:
      type: ITEM_GROWTH
      selector: ".information-flow-list article"
      timeout_ms: 5000
  harvest:
    item_selector: ".information-flow-list article"
    link_selector: "a[href]"
  convergence:
    no_growth_rounds: 3
    max_rounds: 40
    max_runtime_seconds: 120
```

Recipe 不出现 `js_code_before_wait`、`js_only`、`VirtualScrollConfig` 等具体库参数。

### 5.2 Runtime Capability Contract

每个 crawler/runtime release 发布一份能力描述，例如：

```json
{
  "crawler": "crawl4ai",
  "crawler_version": "0.9.x",
  "supports_js_before_wait": true,
  "supports_js_only_session": true,
  "supports_scan_full_page": true,
  "supports_virtual_scroll": true,
  "js_execution_order": [
    "navigate",
    "js_before_wait",
    "wait_for",
    "delay",
    "js_after_wait",
    "capture"
  ]
}
```

能力信息与 `runtime_release_id` 一起版本化。Recipe Compiler 根据语义动作和 capability 生成可执行计划。

### 5.3 当前 Crawl4AI 的典型编译映射

```text
“先滚动，再等列表增长”
 -> js_code_before_wait + wait_for

“同一页面连续点击多次”
 -> session_id + js_only=True + 多轮调用

“页面不断 append 内容”
 -> scan_full_page 或受控逐轮 scroll + harvest

“虚拟列表替换旧节点”
 -> VirtualScrollConfig 或逐轮 harvest + 业务侧 union

“普通 next 链接分页”
 -> 提取 next href，重新进入 URL pipeline
```

旧 runtime 若只有旧语义，则由旧版本 Adapter 做兼容映射；**不能让新 runtime 猜测旧配置是什么意思。**

### 5.4 升级门禁

Crawler、Playwright、Chromium 或 Recipe Compiler 任何一个升级时：

```text
生成 Runtime Capability
 -> 编译全部 ACTIVE Recipe
 -> 静态兼容检查
 -> Golden Corpus 回放
 -> 对比 URL Coverage / Quality / Cost
 -> Canary
 -> 审批 Release
```

若无法保持语义，直接阻止发布：

```text
RECIPE_RUNTIME_INCOMPATIBLE
UNSUPPORTED_INTERACTION_MODE
WAIT_SEMANTICS_CHANGED
VIRTUAL_SCROLL_CAPABILITY_MISSING
```

Render Artifact / Coverage Evidence 记录：

```text
recipe_release_id
recipe_compiler_version
runtime_release_id
runtime_capabilities_hash
compiled_plan_hash
```

这样某次历史抓取即使几年后回放，也能知道当时到底执行了什么语义。

## 6. 原文方案的优点

### 6.1 极小成本验证 Browser 必要性

对新站点 Probe 来说，原文模式非常适合快速判断：

- 是否必须执行 JavaScript；
- 是否需要滚动或点击；
- 哪个 selector 可作为 ready signal；
- 动态动作后是否出现文章链接；
- Browser 与纯 HTTP 的 URL 产出差异。

### 6.2 天然可以抽象为配置化 Recipe

原文的：

```text
URL + JS action + wait condition
```

本质上就是一个最小交互 Recipe。将其声明式化以后，第 1001 个站点通常只需要配置和验证，而不是新增专用 Python 爬虫类。

## 7. 原文不能直接生产化的地方

### 7.1 一次 `scrollTo(bottom)` 不能证明历史抓全

Infinite Scroll 通常按批次加载。一次滚到底可能只触发一批，也可能因为请求尚未完成就结束。

历史回填必须循环执行：

```text
harvest -> normalize -> dedupe -> action -> wait growth -> harvest
```

直到出现可解释的收敛条件，而不是“脚本执行完了”。

### 7.2 `wait_for` 命中不等于 Coverage 完成

`.information-flow-list` 出现只说明容器存在，无法证明：

- 下一批已经加载；
- cursor 已耗尽；
- 所有分页已经遍历；
- 已到最老历史边界；
- 虚拟列表此前出现过的 URL 已被保存。

Coverage 结束必须由 URL 增长和 Provider 边界来判断。

### 7.3 图片 `alt` 不能作为文章身份

原文最终遍历 `media.images`，根据图片 URL 过滤，再打印 `alt`。这不适合知识库：

- 一篇文章可以无图或多图；
- alt 可能为空或与标题不同；
- 图片 CDN 会变；
- 广告、头像、推荐图可能混入；
- 没有真实文章 URL，无法进入正文抓取和增量去重。

生产 Discovery 的主键必须来自真实 `<a href>` 经 URL normalize 后得到的稳定 URL Identity。

当前 Crawl4AI 可直接返回 `result.links["internal"]`；若列表 DOM 稳定，也可使用 `JsonCssExtractionStrategy` 结构化提取 `href/title/time`。

### 7.4 整个 CrawlResult 直接作为业务数据会形成工具耦合

`model_dump_json()` 很适合调试，但不应作为业务领域模型。否则：

- 工具升级会改变下游 schema；
- 大量无关字段重复保存；
- 无法区分 Render Fact 与业务 Observation；
- 数据血缘依赖第三方对象结构。

正确边界：

```text
Crawler Result
 -> Adapter
 -> DiscoveryItem[]
 -> URL Normalize / Scope / Classify / Dedupe
 -> URL Observation
```

原始 CrawlResult/DOM 可作为 Render Artifact 保存到对象存储，用于回放和审计。

### 7.5 缺少业务级 checkpoint

Browser session、当前 DOM、crawler frontier 都是易失执行状态。Worker 崩溃后不能依赖它们恢复。

平台应持久化：

```text
provider cursor
incremental watermark
最近 Observation
最近已知 URL window
round / convergence evidence
recipe release
runtime release
```

任务重启时从 PostgreSQL 事实恢复。

### 7.6 缺少成本和全局公平控制

1000 个站点全部使用 Browser 滚动会非常昂贵。动态列表应排在低成本权威 Provider 后：

```text
CMS/API
 > Sitemap
 > RSS/Atom
 > 静态 Archive/Pagination
 > Prefetch Mapping
 > Dynamic Listing Browser
 > bounded Deep Crawl
```

只有前面的 Provider 无法证明 Coverage 时才升级。

## 8. 生产级 Dynamic Listing Discovery

### 8.1 独立 Provider

增加：

```text
DYNAMIC_LISTING
```

它负责“从动态归档/栏目页发现文章 URL”，与“打开一篇文章获取正文”的 Browser Content Fetch 分离。

输入：

```text
SourceProfile
DynamicListingRecipe
RunBudget
RuntimeRelease
```

输出：

```text
DiscoveryItem[]
CoverageEvidence
RenderTrace
CostEvent[]
```

### 8.2 Recipe 模型

```yaml
name: dynamic_blog_list_v4
start_urls:
  - https://example.com/blog
ready_selector: ".information-flow-list"
list_item_selector: ".information-flow-list article"
link_selector: "a[href]"
interaction_mode: APPEND_SCROLL
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

至少支持：

```text
STATIC
PAGINATION
CLICK_MORE
APPEND_SCROLL
VIRTUAL_SCROLL
```

### 8.3 URL 身份必须来自链接

最小 DiscoveryItem：

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

提取后统一执行：

```text
resolve relative URL
 -> normalize
 -> scope/SSRF check
 -> article path classify
 -> dedupe
 -> persist URL Observation
```

### 8.4 收敛算法

```python
seen = set()
no_growth = 0

for round_no in range(recipe.max_rounds):
    items = await harvest_current_dom()
    urls = normalize_scope_filter(items)
    new_urls = urls - seen
    seen |= new_urls

    persist_observations(new_urls, round_no)

    if incremental_overlap_reached(new_urls):
        stop("OVERLAP_BOUNDARY")
        break

    if not new_urls:
        no_growth += 1
    else:
        no_growth = 0

    if no_growth >= recipe.no_growth_rounds:
        stop("NO_GROWTH")
        break

    await perform_next_semantic_action()
    await wait_for_item_growth_or_timeout()
else:
    stop("MAX_ROUNDS")
```

关键原则：**判断是否继续的核心信号是“规范化、通过 Scope/分类后的文章 URL 是否增长”，不是 DOM 高度。** DOM 高度只能作为辅助信号。

### 8.5 虚拟列表必须逐轮 harvest

Virtual Scroll 会回收旧节点。如果只在最终 DOM 中提取一次，早期出现过的文章可能消失。

两种安全策略：

- 使用 Crawl4AI Virtual Scroll 能力，让 Adapter 获取合并后的结果，并仍在业务层去重；
- 每轮动作后即时 harvest，将 URL Observation 持久化，最终 DOM 是否还保留旧节点不重要。

### 8.6 Coverage Evidence

动态列表至少记录：

```text
provider_type = DYNAMIC_LISTING
start_url
recipe_release_id
recipe_compiler_version
runtime_release_id
runtime_capabilities_hash
compiled_plan_hash
interaction_mode
rounds
observed_items
unique_article_urls
new_urls_by_round[]
duplicate_ratio
selector_hit_rate
first_published_hint
last_published_hint
exhausted
exhaustion_reason
max_rounds
runtime_ms
render_artifact_id
```

`NO_GROWTH`、`CURSOR_EXHAUSTED`、`OVERLAP_BOUNDARY` 可以是正常结束；`MAX_ROUNDS`、`TIME_BUDGET`、`MAX_ITEMS` 表示存在潜在 Known Gap，不能伪装成“历史已抓全”。

## 9. 增量同步

动态列表通常按时间倒序，因此增量不必每次滚完整历史：

```text
打开列表顶部
 -> harvest 新 URL
 -> 继续滚动
 -> 进入 overlap 时间窗口
 -> 连续 K 个已知 URL
 -> 连续 N 轮无 unseen URL
 -> 停止
```

建议保留 12~48 小时 overlap；站点没有可靠时间时，使用“连续已知 URL + 无新 URL 轮数”的组合边界。

还应周期性做低频 Verify/Deep Recheck，处理：

- 置顶旧文；
- 编辑后重新上浮；
- 延迟发布；
- URL 迁移；
- 列表排序策略变化。

Watermark 是性能优化，不是正确性的唯一依据。

## 10. 1000+ 站点的资源与调度模型

Dynamic Listing Browser 与正文 Browser 拆成独立 Worker Pool：

```text
Browser Discovery Worker
- 长页面
- 多轮交互
- 低并发
- 强 max_rounds / max_runtime / max_items

Browser Content Worker
- 单篇文章
- 生命周期短
- 正文完整度优先
```

全局控制由平台负责：

- per-domain token bucket；
- per-domain semaphore；
- Source fairness；
- Backfill / Incremental 分优先级；
- Browser 全局并发上限；
- 429/503/Retry-After 自适应降速；
- 每 Source 每日 Browser 秒数/流量预算；
- lease、fencing token、retry、DLQ。

Crawl4AI 自带 Dispatcher/RateLimiter 可以保护单 Worker，但不能替代跨 Pod 的全局 Admission 和公平调度。

## 11. Drift Detection

动态列表最危险的故障是“HTTP 200 + 空列表/少列表”，因此必须监控：

```text
ready selector 命中率
每轮 item 数
每轮新增唯一文章 URL 数
article candidate / all links 比例
重复率
Browser fallback 比例
平均 interaction round
与历史基线相比的 URL yield
```

若 selector 命中率或 URL yield 突然下降：

```text
Drift Alert
 -> Source DEGRADED
 -> 保护性降速
 -> 保存失败 Render Artifact
 -> 最近样本 / Golden Corpus 回放
 -> 生成 Recipe 候选
 -> 对比 Coverage / Cost
 -> 审批新 Release
```

Runtime 升级造成的群体性下降应优先判定为平台回归，而不是 1000 个站点同时改版。

## 12. 安全设计

### 12.1 Web 不允许任意 JavaScript

Web 管理员只编辑声明式动作：

```text
WAIT_SELECTOR
CLICK_SELECTOR
SCROLL_PAGE
SCROLL_CONTAINER
WAIT_ITEM_GROWTH
EXTRACT_LINKS
```

Recipe Compiler 生成受控脚本。禁止外部 API 直接透传任意 `js_code`、Playwright evaluate、代理地址或系统参数。

### 12.2 每次跳转重新验证 Scope/SSRF

以下对象均重新校验：

- start URL；
- redirect target；
- 提取出的 href；
- Browser navigation；
- iframe；
- Asset URL。

检查 scheme、allowed_hosts、私网/metadata 地址、重定向次数、URL 长度和协议白名单。

## 13. Web 管理功能

动态列表管理页应提供：

- Recipe 编辑与版本历史；
- 语义动作可视化，不显示/接受任意 JS；
- 编译后的 Execution Plan 预览；
- Runtime Capability 与不兼容提示；
- Probe 时 HTTP vs Browser URL 产出对比；
- 每轮新增 URL 曲线；
- selector hit rate；
- exhaustion reason / Known Gap；
- Render Artifact 预览；
- 候选链接审核；
- Release 发布、Canary、回滚。

这样管理员可以回答“为什么认为这个栏目已经抓到边界”和“升级 crawler 后哪些 Recipe 语义发生变化”。

## 14. 对总技术方案的优化结论

本次调研不改变已有的 `Authoritative First / HTTP First / Browser Last` 主方向，重点补强两层能力。

第一层是 Dynamic Listing Discovery：

1. 动态栏目/Infinite Scroll/Load More/Virtual Scroll 是独立 Discovery Provider；
2. 从真实 anchor href 或结构化 extraction 获取文章身份；
3. 以规范化文章 URL 的新增量作为收敛核心信号；
4. 保存独立 Coverage Evidence 与可解释 exhaustion reason；
5. 增量使用 overlap + 已知 URL 收敛；
6. Browser Discovery 与 Browser Content 分池；
7. selector/URL yield 纳入 Drift Detector；
8. Web 提供 URL 增长曲线、Recipe 审核和收敛证据。

第二层是本次进一步识别出的 **Runtime Semantic Compatibility**：

1. Recipe 表达业务语义，不绑定 Crawl4AI 参数；
2. Runtime Release 发布 Capability Contract；
3. Recipe Compiler 把 `trigger -> wait -> harvest` 编译为具体版本的执行顺序；
4. 当前 0.9.x 中“先触发再等待”应映射为 `js_code_before_wait + wait_for`；
5. Runtime/Compiler 升级必须跑 ACTIVE Recipe 静态兼容检查和 Golden Corpus；
6. 不兼容时 Fail Closed，不允许静默降级为少抓；
7. Coverage Evidence 记录 compiler/runtime/capability/compiled plan 哈希，可审计可回放。

这层设计对 1000+ 站点尤其重要：站点数量越多，Crawler 库一次行为语义变化造成的影响越可能成为“全局静默漏抓”，因此必须把工具版本兼容从人工经验升级为平台级契约。

## 15. 参考资料

- 调研文章：https://adg.csdn.net/69706db7437a6b40336a3679.html
- Crawl4AI Page Interaction：https://docs.crawl4ai.com/core/page-interaction/
- Crawl4AI Crawler Run Config API：https://docs.crawl4ai.com/api/parameters/
- Crawl4AI Session Management：https://docs.crawl4ai.com/advanced/session-management/
- Crawl4AI Virtual Scroll：https://docs.crawl4ai.com/advanced/virtual-scroll/
- Crawl4AI Crawler Result：https://docs.crawl4ai.com/core/crawler-result/
- Crawl4AI No-LLM Extraction：https://docs.crawl4ai.com/extraction/no-llm-strategies/
- Crawl4AI v0.8.5 Release Notes：https://docs.crawl4ai.com/blog/releases/0.8.5/

## 16. 最终判断

原文代码适合做单站点 PoC，但不能直接复制成大规模生产爬虫。其真正价值是暴露了动态列表作为历史 URL 入口的重要性。

生产化的正确方向是：

```text
Dynamic Listing Provider
 + Declarative Recipe IR
 + Runtime Capability Contract
 + Recipe Compiler
 + URL Observation
 + Convergence Evidence
 + Incremental Checkpoint
 + Global Admission / Budget
 + Drift Detection
 + Golden Corpus / Canary
```

这样平台既能处理今天的 Crawl4AI 0.9.x，也能在未来替换 Crawl4AI、Playwright 或浏览器版本时维持业务语义和 Coverage 可验证性，而不是把知识库正确性绑定在某个 crawler 参数的偶然行为上。