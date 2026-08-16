# Crawl4AI Virtual Scroll：虚拟滚动与动态归档发现

## 1. 调研对象与结论

- 官方文档：https://docs.crawl4ai.com/advanced/virtual-scroll/
- 官方源码：https://github.com/unclecode/crawl4ai
- 本次核对源码提交：`7e801521428ee12509994d39151006f64055ebe3`
- 核心实现：`crawl4ai/async_configs.py` 的 `VirtualScrollConfig` 与 `crawl4ai/async_crawler_strategy.py` 的 `_handle_virtual_scroll()`。

结论：**Virtual Scroll 很适合解决“列表 DOM 会复用/替换，普通滚动最终只能看到窗口末端内容”的抓取问题，但它应当被当作动态归档发现的 Runtime 能力，而不能直接承担历史文章 Coverage 真相。** 对 1000+ 技术博客知识库，最合理的集成方式是增加平台级 `DynamicArchiveProvider`：由平台负责模式判定、持久化进度、文章身份去重、停止原因、完整性证据与增量高水位，Crawl4AI 只负责具体浏览器滚动与 DOM 捕获。

## 2. Virtual Scroll 解决的底层问题

动态列表常见三种行为：

1. **静态/分页列表**：翻页后进入新 URL，或页码参数变化；
2. **无限追加（append）**：滚动后新条目追加到 DOM，旧条目仍保留，DOM 持续增长；
3. **虚拟滚动（replace/windowed rendering）**：页面只保留可视窗口附近的少量节点，向下滚动时旧节点被回收，新节点复用/替换，DOM 大小大致稳定。

第三种模式对普通 HTML 抓取非常危险。若只在滚动结束后读取 `container.innerHTML`，早期文章已经从 DOM 消失，因此即使浏览器“滚到了底”，最终 HTML 也可能只包含最后一个窗口。

Virtual Scroll 的核心思想不是让 DOM 永远增长，而是**在窗口被替换之前保存旧窗口快照，并在结束后把多个窗口合并成一个合成 DOM**，使后续提取器能够看到滚动过程中的历史内容。

## 3. 官方配置模型

`VirtualScrollConfig` 暴露四个主要参数：

```python
VirtualScrollConfig(
    container_selector="#feed",
    scroll_count=20,
    scroll_by="container_height",
    wait_after_scroll=0.5,
)
```

语义：

- `container_selector`：滚动容器 CSS selector；
- `scroll_count`：最多滚动多少次；
- `scroll_by`：每次滚动距离，可为 `container_height`、`page_height` 或固定像素；
- `wait_after_scroll`：每次滚动后固定等待时间。

官方文档明确区分：

- `scan_full_page` 更适合“内容持续追加”的传统无限滚动；
- `virtual_scroll_config` 更适合“旧内容被替换”的虚拟列表。

因此对博客归档，不能把所有动态页面统一设置成 `scan_full_page=True`。需要先识别列表行为，再选择执行策略。

## 4. `_handle_virtual_scroll()` 的实现流程

当前源码中的执行顺序很重要：页面先完成导航和 `wait_for`，确保容器已经出现，然后才执行 `_handle_virtual_scroll()`；Virtual Scroll 结束后再进入 `before_retrieve_html` 和最终 HTML 获取。

核心算法如下。

### 4.1 找到滚动容器

```javascript
const container = document.querySelector(config.container_selector)
```

找不到容器时抛错。平台侧因此不能把 selector 当成永久真值，需要把它纳入版本化 Recipe/SelectorBundle，并监控匹配率。

### 4.2 保存前一窗口 HTML

实现维护：

```text
htmlChunks = []
previousHTML = container.innerHTML
scrollCount = 0
```

每次滚动后等待 `wait_after_scroll`，重新读取 `currentHTML`。

### 4.3 判定三种变化

源码用字符串关系区分：

```text
currentHTML == previousHTML
    -> 没变化

currentHTML.startsWith(previousHTML)
    -> 认为是追加，新内容已经留在当前 DOM

其他情况
    -> 认为发生替换，把 previousHTML 放进 htmlChunks
```

这是一种低成本启发式：对真正的虚拟列表，只要旧窗口被回收，`startsWith()` 通常会失败，于是旧窗口在消失前得到保存。

### 4.4 终止条件

循环首先受 `scroll_count` 硬上限约束；另外检查：

```text
container.scrollTop + container.clientHeight >= container.scrollHeight - 10
```

满足后认为到达当前滚动容器底部；若之前检测到替换，再保存最后一个窗口并结束。

这里必须区分两个概念：

- “浏览器当前认为滚动到底”；
- “Provider 已证明所有历史文章枚举完成”。

它们并不等价。无限源、异步继续扩展 `scrollHeight`、服务端游标、延迟加载失败、预算耗尽都可能让前者出现而后者不成立。

### 4.5 合并窗口

只要发生过 replace，源码会把每个 chunk 放入临时 `div`，遍历其**直接子元素**，使用下面的归一化文本作为去重键：

```javascript
element.innerText
  .toLowerCase()
  .replace(/[\s\W]/g, '')
```

未见过的元素保存 `outerHTML`，最后：

```javascript
container.innerHTML = uniqueElements.join('\n')
```

也就是说，最终返回的 HTML 已经是 Crawl4AI 在页面内构造的**合成表示**，而不是服务端一次响应或浏览器某个自然时刻的原始 DOM。

### 4.6 失败语义

`_handle_virtual_scroll()` 外层捕获异常并记录日志，随后继续正常抓取流程。这样做对通用 SDK 很友好：Virtual Scroll 失败不会让整页抓取完全不可用。

但对知识库 Coverage 来说必须反向处理：**Virtual Scroll 失败、容器不存在、滚动预算耗尽、合并异常都必须映射成平台 typed reason；不能因为外层 crawl 最终仍返回 HTML，就把动态归档 Provider Run 标记为完整。**

## 5. 实现上的优势

### 5.1 解决 DOM 回收导致的历史内容丢失

这是 Virtual Scroll 最大价值。它保存滚动过程中即将消失的窗口，并在结束后提供统一 HTML 给下游抽取器。

### 5.2 无需站点端 API 知识即可工作

只要能定位滚动容器，就能处理一批 React/Vue/虚拟列表组件，适合作为新站 onboarding 或未知动态归档的通用慢路径。

### 5.3 有明确预算

`scroll_count` 与 `wait_after_scroll` 使一次运行具有可计算的时间下界/上界，便于平台做浏览器并发与成本控制。

### 5.4 与普通抽取链兼容

合并完成后仍输出 HTML，因此 CSS/XPath、链接抽取、文章候选识别等逻辑无需全部改造成事件流。

## 6. 不能直接用于生产 Coverage 的原因

### 6.1 文本去重可能误合并不同文章

当前去重键来自可见文本，去掉空白和非单词字符后比较。以下内容可能冲突：

- 两篇不同文章卡片标题相同；
- 卡片可见文本相同但 `href` 不同；
- “Part 1 / Part-1”等经过归一化后碰撞；
- 只有隐藏属性、日期属性或链接 URL 不同的条目。

对于 URL Inventory，稳定键应优先使用：

```text
normalized_article_url
> data-id / canonical item id
> (title, published_at, author) composite
> DOM structural fingerprint
```

文本 hash 只能作为辅助去重证据。

### 6.2 `startsWith(previousHTML)` 对轻微 DOM 抖动敏感

广告、相对时间、埋点属性、随机 class、计数器等变化都可能让 `startsWith()` 失败，使“追加模式”被误判为“替换模式”。这通常不会导致完全抓不到数据，但会增加 chunk 数、重复和合成误差。

生产层应基于**文章条目身份集合**判断 append/replace，而不是完整 HTML 字符串前缀。

### 6.3 固定 sleep 不等于数据已稳定

`wait_after_scroll=0.5` 只是时间等待。网络慢、接口排队、骨架屏、Mutation 延迟时，0.5 秒后读取 HTML 可能仍是旧窗口。

平台 Recipe 应支持：

- `wait_for` selector/JS 条件；
- MutationObserver 稳定窗口；
- 目标 XHR/fetch 完成；
- network quiet；
- “新 article key 出现”作为 progress 条件；
- 固定等待只作为兜底。

### 6.4 `scroll_count` 是预算，不是完整性证明

运行恰好滚了 20 次，最多说明预算耗尽。平台必须明确记录：

```text
stop_reason = SCROLL_BUDGET_EXHAUSTED
completion_type = PARTIAL
```

不能把它写成 `ATTESTED_COMPLETE`。

### 6.5 “到达底部”可能只是当前客户端窗口的底部

某些站点在接近底部后才异步请求下一批；另一些站点的滚动容器高度固定，而数据游标并未耗尽。只用 scroll geometry 可能早停。

更强的完成证据包括：

- API `next_cursor=null` / `has_more=false`；
- 明确 end-of-list DOM 标志；
- 服务端 total 与已观察稳定 ID 数一致；
- 确定性的分页 Next 不再存在且上一页数量符合协议；
- 多个独立 Provider 交叉验证无缺口。

### 6.6 合并后的 DOM 是 Derived Artifact

Virtual Scroll 会主动替换 `container.innerHTML`。因此最终 HTML 不能伪装成原始网络证据。

平台应分别保存：

1. 网络原始响应 / 初始 DOM Evidence；
2. 每个 scroll step 的 Observation；
3. Crawl4AI 合成 HTML（Derived Artifact）；
4. 从 Observation 得到的 URL Inventory。

合成 HTML 可以用于提取，但不能覆盖 Raw Artifact。

### 6.7 配置需要平台二次限界

当前 `VirtualScrollConfig` 构造函数直接接受 `scroll_count`、`scroll_by`、`wait_after_scroll`。源码的 untrusted config 安全边界允许 `VirtualScrollConfig`，而常见 `max_scroll_steps` 上限并不天然等同于嵌套 `VirtualScrollConfig.scroll_count`。

因此 Web 管理端不能把任意值直接透传。Config Compiler 必须独立限制：

```text
container_selector length / complexity
1 <= scroll_count <= policy.max_dynamic_archive_scrolls
scroll_by in approved enum/range
0 <= wait_after_scroll <= policy.max_wait
wall_time / bytes / unique_items / network_requests hard budget
```

## 7. 面向 1000+ 博客的 DynamicArchiveProvider 设计

### 7.1 Provider 角色

新增：

```text
DynamicArchiveProvider
  |- STATIC_PAGINATION
  |- LOAD_MORE
  |- INFINITE_APPEND
  `- VIRTUAL_REPLACE
```

Provider 只做 URL discovery，不抓文章正文真值。正文 URL 进入统一 Persistent Frontier 后，由 HTTP/Browser Fetch Plane 处理。

### 7.2 Onboarding Probe

新 Source 首次接入时做小预算探测：

1. HTTP 获取 archive/category/tag 首页；
2. 抽取初始 article keys 与 next link；
3. 若静态分页成立，优先静态 Provider；
4. 否则浏览器执行 2~3 个受限滚动步骤；
5. 比较 `item_key_set`、DOM child count、scrollHeight、network requests；
6. 分类为 append / replace / load-more / unknown；
7. 生成 `dynamic_archive_recipe_candidate`，经 fixture/live sample 验证后发布。

原则：**能不用 Browser 就不用；能找到底层 JSON/API 游标就优先升级为 API Provider。** 浏览器滚动适合发现协议，不应长期掩盖可用的权威 API。

### 7.3 每一步必须保存的 Observation

```text
dynamic_archive_step(
  run_id,
  step_no,
  observed_at,
  scroll_top,
  client_height,
  scroll_height,
  item_count,
  unique_item_count,
  new_item_count,
  item_key_set_hash,
  first_item_key,
  last_item_key,
  dom_fingerprint,
  network_cursor_hint,
  end_marker_seen,
  wait_condition,
  screenshot_or_dom_artifact_id,
  stop_reason
)
```

文章 URL 仍通过标准 `url_observation` 落库，带 `provider_run_id + step_no` provenance。

### 7.4 稳定条目身份

推荐：

```text
item_key = hash(
  normalized_href,
  source_specific_item_id?,
  published_at_hint?,
  stable_card_attributes?
)
```

若没有 href，才降级到标题/日期/作者等组合。不要使用纯 `innerText` 作为主身份。

### 7.5 完成状态

建议状态：

```text
ATTESTED_COMPLETE
HEURISTIC_EXHAUSTED
PARTIAL
FAILED
BLOCKED
```

`ATTESTED_COMPLETE` 只在有协议级结束证据时使用。

动态滚动常见 stop reason：

```text
END_MARKER
CURSOR_EXHAUSTED
SERVER_TOTAL_REACHED
NO_NEXT_PAGE
STABLE_NO_NEW_ITEMS
SCROLL_GEOMETRY_END
SCROLL_BUDGET_EXHAUSTED
WALL_TIME_EXHAUSTED
NETWORK_BUDGET_EXHAUSTED
CONTAINER_NOT_FOUND
SELECTOR_AMBIGUOUS
WAIT_TIMEOUT
VIRTUAL_SCROLL_RUNTIME_ERROR
POLICY_DENIED
```

其中 `STABLE_NO_NEW_ITEMS`、`SCROLL_GEOMETRY_END` 默认只能形成 `HEURISTIC_EXHAUSTED`，不能单独证明历史全集。

### 7.6 断点恢复

浏览器 session 是 Runtime 临时状态，不能作为 durable checkpoint。

持久 checkpoint 保存：

```text
last_unique_item_key
oldest_published_at_hint
unique_item_count
step_no
last_item_key_set_hash
last_cursor_hint
recipe_release
fetch_variant_key
```

重启后新建浏览器会话，按 Recipe 快速重放到 checkpoint 附近；若发现底层 cursor/API，则直接转为 cursor checkpoint。恢复过程产生新 Attempt，不伪装成原浏览器事务继续执行。

## 8. 增量同步优化

动态归档不应每 6 小时从第一页一直滚到历史底部。

### 8.1 高水位

保存：

```text
newest_known_article_key
newest_known_published_at
archive_head_fingerprint
last_incremental_archive_run
```

增量运行从顶部开始，持续观察新条目；出现连续 `K` 个已知稳定 item key 且排序单调性满足 Source Policy 后可 early stop。

### 8.2 防止漏掉回填/置顶/乱序文章

- 定期 forced deep archive scan；
- tag/category/sitemap/CMS 多 Provider reconcile；
- 对置顶文章单独识别，不把“首条已知”当停止条件；
- 保存最近 N 个窗口的 known-key Bloom/Set；
- 对异常时间倒序、文章插入历史位置的站点关闭 aggressive early stop。

### 8.3 修改文章仍由 Revalidation Engine 负责

DynamicArchiveProvider 只发现 URL/迁移线索；已知文章正文是否变化仍由 ETag/Last-Modified、CMS revision、强制完整 GET 和 body/semantic hash 判断，避免把“列表卡片没变”误当成“正文没变”。

## 9. Web 管理功能

Source 页面增加 `Dynamic Archive` 卡片：

- 模式：static/load-more/append/virtual-replace/unknown；
- Recipe release 与 container selector；
- 当前 Run：step、unique items、new URLs、oldest date、scroll position；
- append/replace 判定证据；
- Completion Type / Stop Reason；
- budget：scroll、wall-time、browser minutes、network requests；
- 每一步 DOM/screenshot/network evidence；
- `Force Probe`、`Run Incremental`、`Run Deep Backfill`、`Pause`、`Resume`、`Reclassify`；
- 与 Sitemap/CMS/RSS/Common Crawl/Wayback 的 URL 集合差异。

Operator 可以调整声明式参数，但不能直接注入任意 JS/CDP。复杂交互进入受控 Recipe Studio 和审批流程。

## 10. 调度与成本控制

1000 个站点中只有少数需要 Browser Dynamic Archive，因此应单独资源池：

```text
archive-http-static       高吞吐
archive-browser-dynamic   低并发、按 host/source 限速
fetch-browser-article     与 archive browser 分池
agentic-repair            更低优先级、独立预算
```

Dynamic Archive 默认优先级低于热增量正文 Revalidation，高于离线历史缺口修复；大型历史回填不得占满 Browser Pool。

建议硬预算：

```text
max_scrolls_per_attempt
max_wall_time
max_network_requests
max_unique_items
max_dom_artifact_bytes
max_browser_minutes_per_source_day
```

预算耗尽必须显式 PARTIAL，可继续从 checkpoint 派生新 Task。

## 11. 可观测性与告警

新增指标：

```text
dynamic_archive_run_total{mode,result}
dynamic_archive_step_total
dynamic_archive_unique_item_total
dynamic_archive_new_url_total
dynamic_archive_no_progress_step_total
dynamic_archive_stop_total{reason}
dynamic_archive_completion_total{type}
dynamic_archive_mode_mismatch_total
dynamic_archive_selector_failure_total
dynamic_archive_text_dedupe_collision_total
dynamic_archive_browser_minutes
dynamic_archive_replay_steps_total
```

告警：

- 同一 Source 连续多次 `CONTAINER_NOT_FOUND`；
- unique item 增长骤降；
- 原来 append 突然变 replace/unknown；
- forced deep scan 持续发现增量扫描遗漏 URL；
- `SCROLL_BUDGET_EXHAUSTED` 比例升高；
- Browser minutes/新 URL 成本异常。

## 12. 测试矩阵

必须有本地 Fixture：

1. 静态分页；
2. append infinite scroll；
3. replace virtual scroll；
4. 同标题不同 href，验证不会被纯文本错误去重；
5. DOM 属性每次滚动随机变化；
6. 延迟超过固定 sleep 才加载；
7. 滚到底后异步继续扩容；
8. 容器 selector 失效；
9. 网络错误/429；
10. `scroll_count` 预算耗尽；
11. crash 后从 semantic checkpoint 恢复；
12. 增量顶部存在置顶、乱序、历史插入文章。

Contract Test 必须固定 Crawl4AI release，验证 append/replace 判断、最终合成 DOM、错误处理、Config Compiler 上限以及“Runtime success 不推进 Provider Completion”的平台不变量。

## 13. 对现有技术方案的具体优化

本次调研应使最终方案增加以下能力：

1. 在 Discovery Plane 增加 `DynamicArchiveProvider`，明确四类 archive 模式；
2. `Crawl4AI VirtualScrollConfig` 只作为 VIRTUAL_REPLACE 的执行器，不拥有 Coverage 真值；
3. 把每次滚动变成可持久化 Observation，而不是只保存最终合成 HTML；
4. URL 去重使用稳定 article identity，不继承 Virtual Scroll 的纯文本主键；
5. 引入动态归档 typed stop reason 与 `HEURISTIC_EXHAUSTED`；
6. 增量同步加入 archive high-water mark + 连续已知条目 early-stop + forced deep scan；
7. Browser session 不作为断点，保存 semantic checkpoint；
8. Web Admin 增加动态归档实时进度、证据、模式与成本；
9. Config Compiler 对 `scroll_count/wait/selector/network/wall-time` 单独限界；
10. Fixture/Contract Test 增加 append、replace、文本碰撞、延迟加载、预算耗尽与恢复场景。

## 14. 最终判断

Virtual Scroll 不是“再加一个滚动参数”这么简单。它揭示了大规模博客采集方案里一个容易被忽略的事实：**URL discovery 本身也可能需要有状态的浏览器执行，而且最终 DOM 可能不是历史过程的完整证据。**

因此生产设计应把“浏览器如何滚”与“平台如何证明看到了哪些文章、为什么停止、崩溃后如何继续”拆开：Crawl4AI 负责高效执行，`DynamicArchiveProvider + Persistent Observation/Checkpoint + Coverage Attestation` 负责业务真相。这样才能在 1000+ 站点扩展时既覆盖 React/虚拟列表等动态归档，又不牺牲全历史完整性的可解释性与增量同步的成本控制。
