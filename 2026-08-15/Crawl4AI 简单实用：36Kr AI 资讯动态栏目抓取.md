# Crawl4AI 简单实用：36Kr AI 资讯动态栏目抓取

## 1. 调研对象

- 编号：45
- 名称：Crawl4AI 简单实用：36Kr AI 资讯动态栏目抓取
- 地址：https://adg.csdn.net/69706db7437a6b40336a3679.html
- 原文发布时间：2025-04-18
- 原文使用版本：Crawl4AI 0.5.0.post8
- 调研目标：分析 JavaScript 动态列表、滚动加载与 Crawl4AI 页面交互机制，判断其对“1000+ 技术博客历史全量抓取 + 增量同步 + Web 管理”平台的价值，并给出可生产化的设计。

## 2. 原文实现拆解

原文目标是抓取 36Kr AI 资讯动态栏目 `https://36kr.com/information/AI/`。列表内容依赖浏览器 JavaScript 渲染，作者使用 `AsyncWebCrawler` 打开栏目页，通过 JavaScript 滚动页面，并等待 `.information-flow-list` 出现，再把 CrawlResult 序列化为 JSON。后续示例遍历 `media.images`，按图片 URL 规则过滤，并把图片 `alt` 当作新闻标题输出。

可抽象为：

```text
AsyncWebCrawler
  -> 打开动态栏目
  -> 执行滚动 JavaScript
  -> 等待列表容器
  -> 获取 CrawlResult
  -> 序列化调试结果
  -> 遍历媒体图片
  -> 依据 image.alt 输出标题
```

原文作为 PoC 证明了三件事：

1. 浏览器执行 JavaScript 后，可以看到纯 HTTP 初始 HTML 中不存在或不完整的列表内容；
2. 页面动作与等待条件可以由 crawler 配置控制；
3. CrawlResult 可以同时携带 HTML、链接、媒体、Markdown 等结果，适合快速验证动态页面可抓性。

但它还不是历史全量知识库方案：一次滚动、一次等待和图片 alt 都无法证明文章 URL 已抓全，也无法承担增量、去重、版本、审计和故障恢复。

## 3. 动态栏目背后的技术原理

### 3.1 为什么初始 HTML 会漏文章

技术博客、资讯站和 Newsletter 聚合页常见以下实现：

- 首屏只返回 HTML 壳，列表由 XHR/fetch 请求后插入 DOM；
- 首屏只返回前 N 条，滚动到底通过 IntersectionObserver 或 scroll 事件加载下一批；
- 点击 Load More 后按 cursor/page 加载；
- SPA hydration 后才生成真实链接；
- 虚拟列表只保留视口附近节点，向下滚动时旧节点被回收；
- 下一页 cursor 藏在接口响应、页面状态或客户端路由中。

因此对知识库而言，动态列表首先是 **Discovery / Coverage 问题**，其次才是 Browser Fetch 问题。应先回答“有哪些文章”，再抓每篇正文。

### 3.2 Browser 为什么能发现更多 URL

浏览器会执行站点 JavaScript、触发 scroll/click/IntersectionObserver、发起 XHR/fetch，并把响应渲染进 DOM。Crawler 只是把浏览器导航、动作、等待和采集包装为可调用接口。

生产平台不能把 crawler 参数直接当业务配置。业务层应描述稳定语义：

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

再由 Recipe Compiler / Adapter 翻译成当前 Crawl4AI 或 Playwright 的具体参数。

## 4. 版本语义风险：旧示例不能按参数名直接迁移

原文使用 Crawl4AI 0.5.0.post8，并把 `js_code` 与 `wait_for` 放在同次运行中。当前 Crawl4AI 0.9.x 官方页面交互文档明确给出的执行顺序是：

```text
navigation
 -> js_code_before_wait
 -> wait_for
 -> delay_before_return_html
 -> js_code
 -> flatten_shadow_dom（若启用）
 -> page.content()
```

因此，如果业务语义是“先触发加载，再等待加载结果”，当前运行时应使用：

```text
js_code_before_wait + wait_for
```

而不是机械复制旧示例的 `js_code + wait_for`。

危险点在于：版本语义变化不一定报错。最坏情况是容器早已存在，`wait_for` 立即通过，真正的滚动动作随后才执行，最终只采到首批内容，但任务仍显示 success。对历史 Coverage 来说，这是比显式失败更严重的“静默少抓”。

结论：必须把 **Recipe 业务语义** 与 **Crawler Runtime 参数语义** 解耦。

## 5. Semantic Recipe + Runtime Capability Contract + Recipe Compiler

### 5.1 稳定的业务 Recipe IR

示例：

```yaml
recipe:
  kind: dynamic_listing
  ready:
    type: CSS
    selector: ".information-flow-list"
  harvest:
    item_selector: ".information-flow-list article"
    link_selector: "a[href]"
    extract:
      url: "a::attr(href)"
      title: "a::text"
      published_hint: "time::attr(datetime)"
  interaction:
    mode: APPEND_SCROLL
    action: SCROLL_PAGE
    wait_after_action:
      type: ITEM_GROWTH
      selector: ".information-flow-list article"
      timeout_ms: 5000
  convergence:
    no_growth_rounds: 3
    max_rounds: 40
    max_runtime_seconds: 120
    max_items: 5000
```

Recipe 中不出现 `js_code_before_wait`、`js_only`、`VirtualScrollConfig` 等库私有参数。

### 5.2 Runtime Capability Contract

每个 crawler/runtime release 保存机器可读能力：

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

Capability 与 `runtime_release_id` 一起版本化。

### 5.3 编译映射

```text
TRIGGER -> WAIT -> HARVEST
 -> js_code_before_wait + wait_for + capture

同一页面连续 CLICK_MORE
 -> session_id + js_only=True + 多轮受控动作

APPEND_SCROLL
 -> scan_full_page 或逐轮 scroll + harvest

VIRTUAL_SCROLL
 -> VirtualScrollConfig 或逐轮 harvest + 业务 URL union

PAGINATION
 -> 提取 next href，再作为普通 URL 进入 discovery pipeline
```

旧 Runtime 由对应版本 Adapter 保持旧语义；新 Runtime 不猜测旧配置的意图。

### 5.4 Runtime 升级门禁

Crawler、Playwright、Chromium 或 Recipe Compiler 升级必须执行：

```text
Build Runtime Capability
 -> Compile ALL ACTIVE Recipes
 -> Static Compatibility Check
 -> Golden Corpus Replay
 -> Compare Coverage / Quality / Cost
 -> Canary
 -> Approval
 -> Release
```

不兼容时 Fail Closed：

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

## 6. 原文方案可复用的价值

### 6.1 用于新 Source Probe

极小成本即可验证：

- 静态 HTTP 是否足够；
- Browser 是否确实增加文章 URL；
- 是否需要 scroll/click；
- 哪个 selector 可作为 ready signal；
- 动作后 URL yield 是否增长；
- 是否存在 virtual scroll。

### 6.2 天然适合配置化

原文的 `URL + action + wait condition` 本质上是最小交互 Recipe。将其声明式化后，第 1001 个站点通常只需新增 Source Profile/Recipe 与 Golden Corpus，而不是新增专用爬虫类。

## 7. 原文不能直接生产化的关键问题

### 7.1 一次 `scrollTo(bottom)` 不能证明历史抓全

Infinite Scroll 往往按批次加载。一次滚动只可能触发一批，甚至动作执行后请求尚未完成。

历史回填必须循环：

```text
harvest
 -> normalize / scope / classify / dedupe
 -> persist URL Observation
 -> perform action
 -> wait for observable growth
 -> harvest again
```

直到出现可解释的收敛证据。

### 7.2 `wait_for` 命中不等于 Coverage 完成

列表容器出现只说明“页面已经有一个容器”，不能证明：

- 下一批完成加载；
- cursor 已耗尽；
- 所有分页已遍历；
- 已到最老历史边界；
- 虚拟列表早期出现的节点已被保存。

### 7.3 图片 alt 不能作为文章身份

图片可能缺失、多图、广告、推荐位或 CDN 迁移；alt 也可能为空或重复。

生产 Discovery 的稳定身份必须来自真实文章 `<a href>`，经：

```text
resolve relative URL
 -> normalize
 -> scope/SSRF check
 -> article classifier
 -> dedupe
 -> URL Identity / URL Observation
```

标题、图片、发布时间只作为 hint/evidence。

### 7.4 CrawlResult 是工具返回对象，不是业务领域模型

调试时保存 `model_dump_json()` 合理，但不能把其 schema 直接传播到核心数据模型。

正确边界：

```text
Crawler Result
 -> Adapter
 -> DiscoveryItem[]
 -> URL pipeline
 -> URL Observation
```

原始 CrawlResult、DOM、MHTML、网络摘要可作为 Render Artifact 放对象存储，用于回放和审计。

### 7.5 Browser session 不能当业务 checkpoint

Worker 崩溃、Pod 重启或 Runtime 升级后 session 会丢失。业务 checkpoint 必须落 PostgreSQL：

```text
provider cursor
incremental watermark
最近 URL Observation
round / convergence evidence
recipe_release_id
runtime_release_id
```

### 7.6 Browser 不能成为默认发现方式

1000 个站点全部用 Browser 长滚动成本过高。Provider 默认顺序应为：

```text
CMS/API
 > Sitemap
 > RSS/Atom
 > Archive/Pagination
 > Prefetch Mapping
 > Dynamic Listing Browser
 > bounded Deep Crawl
```

Dynamic Listing 是权威/低成本 Provider 无法证明 Coverage 时的补洞手段。

## 8. 生产级 Dynamic Listing Discovery

### 8.1 独立 Provider

动态栏目独立为：

```text
DYNAMIC_LISTING
```

它只负责“发现文章 URL”，与“打开单篇文章抓正文”的 Browser Content Fetch 分离。

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
RenderArtifact / ExecutionTrace
RuntimeOutcome
CostEvent[]
```

### 8.2 支持的交互模式

```text
STATIC
PAGINATION
CLICK_MORE
APPEND_SCROLL
VIRTUAL_SCROLL
```

每种模式均有明确动作、等待、harvest 与 convergence 语义。

### 8.3 URL-first Harvest

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

平台以 `url_id`/normalized URL 做业务唯一性，不以 DOM 节点、标题或文本指纹做唯一性。

### 8.4 收敛算法

```python
seen = set()
no_growth = 0

for round_no in range(recipe.max_rounds):
    items = await harvest_current_state()
    urls = normalize_scope_filter_classify(items)
    new_urls = urls - seen
    seen |= new_urls

    persist_observations(new_urls, round_no)

    if incremental_overlap_reached(new_urls):
        stop("OVERLAP_BOUNDARY")
        break

    no_growth = no_growth + 1 if not new_urls else 0
    if no_growth >= recipe.no_growth_rounds:
        stop("NO_GROWTH")
        break

    outcome = await perform_next_semantic_action()
    verify_action_outcome(outcome)
    await wait_for_item_growth_or_timeout()
else:
    stop("MAX_ROUNDS")
```

继续与停止的核心信号是 **规范化、通过 Scope/文章分类后的唯一文章 URL 是否增长**。DOM 高度、HTML 长度和 crawler success 都只是辅助信号。

### 8.5 虚拟列表必须逐轮保存业务身份

Virtual Scroll 会回收旧 DOM。如果只在最终 DOM 做一次 extraction，早期节点可能消失。

安全策略：

1. 每轮滚动后立即提取链接并写 URL Observation；或
2. 使用 Runtime 的 virtual-scroll chunk 合并能力，但仍把合并结果重新走 URL normalize/dedupe，不能把库内部“去重后 HTML”当 Coverage 事实。

### 8.6 新增：Runtime Outcome Contract

**Capability 只回答“理论上支持什么”，Outcome 必须回答“这一次实际上发生了什么”。**

这是本次调研对既有总方案最重要的补充。

当前 Crawl4AI 0.9.x 官方 Virtual Scroll 文档说明：

- Virtual Scroll 会检测 `NO_CHANGE / APPEND / REPLACE` 三类滚动行为；
- 对 REPLACE 场景按滚动位置捕获 HTML chunk，再合并；
- 合并去重使用规范化文本指纹；
- 如果配置的 container 找不到，Crawler 会继续普通抓取，而整个 crawl 仍可能 `success=true`。

这意味着两类静默 Coverage 风险：

**风险 A：文本去重不是文章身份去重。** 两个不同 URL 若正文/卡片文本恰好相同或高度相似，底层合并可能折叠；反过来，同一 URL 文本变化也可能留下重复。平台不能让库的文本 fingerprint 成为文章 Coverage 的主键。

**风险 B：feature fallback 不等于语义成功。** `result.success=true` 可能只代表页面抓取成功，不代表指定虚拟滚动容器命中，也不代表预期的滚动动作实际完成。

因此 Adapter 必须输出结构化 `RuntimeOutcome`：

```text
runtime_outcome
- task_id
- recipe_release_id
- runtime_release_id
- compiled_plan_hash
- feature: VIRTUAL_SCROLL | APPEND_SCROLL | CLICK_MORE | ...
- feature_requested boolean
- feature_engaged boolean
- container_selector
- container_found boolean
- detected_mode: NO_CHANGE | APPEND | REPLACE | UNKNOWN
- actions_attempted
- actions_completed
- waits_satisfied
- harvest_rounds
- raw_candidates_by_round[]
- normalized_urls_by_round[]
- unique_url_union_count
- runtime_merge_item_count nullable
- stop_reason
- warnings[]
```

Coverage Gate 规则：

```text
feature_requested && !feature_engaged
 -> PROVIDER_DEGRADED
 -> exhausted = false
 -> Known Gap

VIRTUAL_SCROLL && !container_found
 -> VIRTUAL_SCROLL_CONTAINER_NOT_FOUND
 -> exhausted = false

expected_mode 与 detected_mode 不一致
 -> INTERACTION_MODE_MISMATCH
 -> 保存 Render Artifact，进入 Recipe Review

crawler success=true 但 unique URL 无增长
 -> 只能进入 convergence 判断
 -> 不能直接判定 Provider success/exhausted
```

RuntimeOutcome 应进入 Execution Trace、Coverage Evidence、Drift 指标和 Web 管理页。

### 8.7 Dynamic Coverage Evidence

至少保存：

```text
provider_type = DYNAMIC_LISTING
start_url
recipe_release_id
recipe_compiler_version
runtime_release_id
runtime_capabilities_hash
compiled_plan_hash
interaction_mode
runtime_feature_engaged
container_found
detected_mode
rounds
actions_attempted
actions_completed
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
runtime_outcome_id
```

正常边界可以是：

```text
CURSOR_EXHAUSTED
NO_GROWTH（且动作 Outcome 正常）
OVERLAP_BOUNDARY（增量）
```

以下只能形成 Known Gap，不能宣称“历史已抓全”：

```text
MAX_ROUNDS
TIME_BUDGET
MAX_ITEMS
CONTAINER_NOT_FOUND
INTERACTION_MODE_MISMATCH
ACTION_FAILED
RUNTIME_FEATURE_NOT_ENGAGED
```

## 9. 增量同步

动态列表一般按时间倒序，日常同步无需滚完整历史：

```text
打开列表顶部
 -> harvest unseen URLs
 -> 继续滚动/点击
 -> 进入 overlap 时间窗口
 -> 连续 K 个已知 URL
 -> 连续 N 轮无 unseen URL
 -> 停止
```

建议配置 12~48 小时 overlap；若发布时间不可靠，则组合“连续已知 URL + 连续无新增轮数”。Watermark 只做性能优化，不是正确性的唯一依据。

周期性低频 Verify/Deep Recheck 处理：

- 置顶旧文；
- 编辑后重新上浮；
- 延迟发布；
- URL 迁移；
- 排序策略变化；
- 某次动态动作静默失败造成的历史空洞。

## 10. 1000+ 站点资源与调度

Browser Discovery 与 Browser Content 必须分池：

```text
Browser Discovery Worker
- 长页面
- 多轮交互
- 低并发
- max_rounds/max_runtime/max_items 硬预算

Browser Content Worker
- 单文章
- 短生命周期
- 正文完整度优先
```

平台全局控制：

- per-domain token bucket；
- per-domain semaphore；
- Source weighted fairness；
- Incremental > Verify > Backfill；
- Browser 全局并发上限；
- 429/503/Retry-After 自适应降速；
- 每 Source 每日 Browser 秒数/网络预算；
- lease、fencing token、retry、DLQ。

Crawler 内置 Dispatcher/RateLimiter 只能保护单个执行器，不能替代跨 Pod 的全局 Admission 与公平调度。

## 11. Drift Detection

动态列表最危险的故障是“页面请求成功，但 URL yield 突然下降”。至少监控：

```text
ready selector hit rate
container_found rate
runtime_feature_engaged rate
detected_mode distribution
action completion rate
每轮 item 数
每轮新增唯一 URL 数
article candidate / all links 比例
重复率
平均 interaction round
Browser fallback 比例
与历史基线相比的 URL yield
```

异常流程：

```text
Drift Alert
 -> Source DEGRADED
 -> 保护性降速
 -> 保存 Render Artifact + RuntimeOutcome
 -> Golden Corpus / 最近样本回放
 -> 生成 Recipe 候选
 -> Compare Coverage / Cost
 -> Approval
 -> New Release
```

如果同一 Runtime Release 后大量不同站点同时出现 `feature_engaged=false` 或 mode 分布突变，应优先判定为平台/runtime 回归，而不是站点同时改版。

## 12. 安全设计

Web 管理只编辑声明式动作：

```text
WAIT_SELECTOR
CLICK_SELECTOR
SCROLL_PAGE
SCROLL_CONTAINER
WAIT_ITEM_GROWTH
EXTRACT_LINKS
```

禁止 Web/API 直接透传任意 JavaScript、Playwright evaluate、任意代理或系统参数。

以下每跳重新做 Scope/SSRF 校验：

- start URL；
- redirect target；
- href；
- Browser navigation；
- iframe；
- Asset URL。

## 13. Web 管理功能

Dynamic Listing 页面应展示：

- Recipe 编辑与版本；
- 语义动作可视化；
- 编译 Execution Plan；
- Runtime Capability 与兼容检查；
- RuntimeOutcome：feature 是否真正 engaged、container 是否命中、detected mode、动作完成率；
- HTTP vs Browser URL 产出对比；
- 每轮新增唯一 URL 曲线；
- selector/container hit rate；
- exhaustion reason / Known Gap；
- Render Artifact；
- 候选链接审核；
- Release、Canary、Rollback。

管理员必须能回答两个问题：

1. 为什么认为这个动态栏目已经到达历史边界？
2. 这次 Runtime 是否真的执行了 Recipe 所要求的交互语义？

## 14. 对总技术方案的优化结论

本次调研不改变既有 `Authoritative First / HTTP First / Browser Last / Coverage First` 主方向，确认并强化：

1. 动态栏目/Infinite Scroll/Load More/Virtual Scroll 是独立 Discovery Provider；
2. 文章身份来自真实 href + URL Identity；
3. 以业务 URL 新增量而不是 DOM 高度判断收敛；
4. Dynamic Coverage Evidence 保存可解释 exhaustion；
5. 增量使用 overlap + 已知 URL 收敛；
6. Browser Discovery 与 Browser Content 分池；
7. Recipe 用稳定业务语义表达，Runtime Capability + Compiler 适配工具版本；
8. Runtime/Compiler 升级经过 Golden Corpus + Canary，语义不兼容时 Fail Closed。

新增的关键能力是：

9. **Runtime Outcome Contract**：Capability 证明“支持”，Outcome 证明“本次实际执行”；
10. **业务 URL Identity 不依赖 crawler 内部 HTML/text 去重**：尤其 Virtual Scroll 的文本指纹合并只能作为执行优化；
11. **feature fallback 必须可见**：container 未命中、模式不匹配、动作没完成时不能因 crawler success 而误判 Coverage；
12. **Outcome 纳入 Coverage/Drift/Web/Release**：把“动态动作实际发生了什么”变成可审计事实。

这对 1000+ 站点很重要：Crawler 库的一个行为变化、一个 selector 漂移或一次 silent fallback，都可能同时造成大规模静默漏抓。只有把“能力契约 + 编译计划 + 实际执行结果 + URL 业务事实”四层分开，才能长期保证 Coverage。

## 15. 参考资料

- 调研文章：https://adg.csdn.net/69706db7437a6b40336a3679.html
- Crawl4AI Page Interaction：https://docs.crawl4ai.com/core/page-interaction/
- Crawl4AI Crawler Run Config API：https://docs.crawl4ai.com/api/parameters/
- Crawl4AI Session Management：https://docs.crawl4ai.com/advanced/session-management/
- Crawl4AI Virtual Scroll：https://docs.crawl4ai.com/advanced/virtual-scroll/
- Crawl4AI Crawler Result：https://docs.crawl4ai.com/core/crawler-result/
- Crawl4AI No-LLM Extraction：https://docs.crawl4ai.com/extraction/no-llm-strategies/

## 16. 最终判断

原文代码适合单站点 PoC，其真正价值不是“滚一下就能抓 36Kr”，而是揭示动态栏目是历史 URL 发现的重要入口。

生产化应形成：

```text
Dynamic Listing Provider
 + Declarative Recipe IR
 + Runtime Capability Contract
 + Recipe Compiler
 + Runtime Outcome Contract
 + URL Observation
 + Convergence / Coverage Evidence
 + Incremental Checkpoint
 + Global Admission / Budget
 + Drift Detection
 + Golden Corpus / Canary
```

其中 Runtime Capability 回答“这个版本能否做”，Compiled Plan 回答“准备怎么做”，Runtime Outcome 回答“这次实际做了什么”，URL Observation/Coverage Evidence 回答“业务上究竟发现了哪些文章、为什么认为已到边界”。这样知识库正确性不依赖某个 crawler 参数或合并算法的偶然行为。