# Crawl4AI Virtual Scroll：虚拟滚动与动态归档发现

## 1. 调研对象与结论

- 官方文档：https://docs.crawl4ai.com/advanced/virtual-scroll/
- 官方源码：https://github.com/unclecode/crawl4ai
- 核对源码提交：`7e801521428ee12509994d39151006f64055ebe3`
- 关键实现：`crawl4ai/async_configs.py` 的 `VirtualScrollConfig`、`crawl4ai/async_crawler_strategy.py` 的 `_handle_virtual_scroll()`、`tests/test_virtual_scroll.py`。

结论：**Crawl4AI Virtual Scroll 是解决窗口化列表 DOM 回收问题的有效 Runtime 能力，但不适合作为 1000+ 博客历史 Coverage 的权威执行器。** 生产方案应由平台实现一个确定性的 `VirtualArchiveExecutor`，在每次 DOM 被下一窗口覆盖前提取稳定文章身份并持久化 Observation；Crawl4AI 原生 Virtual Scroll 主要作为“合成 HTML 恢复/提取候选”能力使用。Coverage、增量高水位、停止原因、完整性证明、断点和文章身份必须由平台掌握。

尤其需要注意一个多语言风险：源码的最终去重键为 JavaScript `innerText.toLowerCase().replace(/[\s\W]/g, '')`。JavaScript 传统 `\w` 基本只覆盖 ASCII 字母、数字和下划线，因此纯中文、日文等文本会被大量删除，甚至得到空键。多个不同 CJK 文章卡片可能因此被错误合并。对中文/日文/混排技术博客，**不得把 Crawl4AI 合成后的去重结果作为 URL Inventory 真相。**

---

## 2. Virtual Scroll 解决的底层问题

动态归档通常有四类：

1. `STATIC_PAGINATION`：有页码或 Next URL；
2. `LOAD_MORE`：点击按钮加载下一批；
3. `INFINITE_APPEND`：新条目追加到 DOM，旧条目保留；
4. `VIRTUAL_REPLACE`：只保留可视窗口附近节点，滚动后旧节点被回收或复用。

第四类最危险。若只在滚动结束后读取最终 DOM，早期文章已经消失，即使浏览器确实经过过这些内容，最终 HTML 也无法证明曾经看到它们。

Virtual Scroll 的核心思路是：**在旧窗口被替换前保存窗口 HTML，结束后把多个窗口重组为一个合成容器。** 这样后续 CSS/XPath/Markdown 抽取器仍能看到历史窗口内容。

这解决的是“最终 DOM 不包含滚动历史”的执行问题，不等价于“历史全集已枚举完成”。

---

## 3. 官方配置模型

官方 `VirtualScrollConfig` 只有四个核心字段：

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
- `scroll_count`：最大滚动次数，属于执行预算；
- `scroll_by`：整数像素、`container_height` 或 `page_height`；
- `wait_after_scroll`：每次滚动后固定等待。

它没有以下生产级语义：

- article/item selector；
- stable item identity；
- 上滚/反向列表方向；
- cursor/has_more/total；
- DOM Mutation 稳定条件；
- XHR/fetch 完成条件；
- network quiet；
- 每步 Observation 回调；
- durable checkpoint；
- typed completion/stop reason。

因此它天然是浏览器执行参数，不是 Provider 协议模型。

---

## 4. 在 Crawl4AI 页面流水线中的执行顺序

源码中与生产 Recipe 最相关的顺序是：

```text
navigation
 -> scan_full_page（如启用）
 -> js_code_before_wait
 -> wait_for
 -> virtual_scroll_config
 -> before_retrieve_html hook / delay
 -> js_code
 -> 最终 HTML / screenshot / 后处理
```

这个顺序有两个重要含义。

### 4.1 需要激活列表的动作必须发生在 Virtual Scroll 之前

例如“先点击 tab，再出现归档容器”，应通过平台确定性 Recipe 或允许的 pre-scroll action 完成；不能依赖 Virtual Scroll 之后的 `js_code`。

### 4.2 Virtual Scroll 后的 DOM 可能已经是合成 DOM

发生 replace 时，Crawl4AI 最后会执行：

```javascript
container.innerHTML = uniqueElements.join('\n')
```

这会用重组后的 HTML 替换真实运行时节点，原节点的事件监听器、框架绑定、内部对象身份可能消失。若后续 `js_code` 还想点击这些节点或依赖 React/Vue 状态，可能出现行为与真实页面不同的问题。

因此生产设计应把：

```text
动态归档发现
```

和：

```text
后续交互/文章正文抓取
```

拆成不同 Attempt/Recipe，不在原生 Virtual Scroll 合成 DOM 上继续做有状态交互。

同理，Virtual Scroll 后才生成的最终截图可能展示的是**合成后的 Derived View**，不能冒充某个自然滚动步骤的原始屏幕证据。

---

## 5. `_handle_virtual_scroll()` 的实际算法

### 5.1 初始化

JavaScript 先查找：

```javascript
const container = document.querySelector(config.container_selector)
```

然后维护：

```text
htmlChunks = []
previousHTML = container.innerHTML
scrollCount = 0
```

若容器不存在则抛错。

### 5.2 每步滚动

滚动距离：

- 整数：指定像素；
- `page_height`：`window.innerHeight`；
- 其他默认：`container.offsetHeight`。

执行：

```javascript
container.scrollTop += scrollAmount
await sleep(wait_after_scroll)
currentHTML = container.innerHTML
```

这里没有检查“请求滚动了多少”和“实际 `scrollTop` 改变了多少”。若 selector 指向错误节点、节点不可滚动，预算仍可能被消耗。

### 5.3 三分支变化判断

源码使用完整 HTML 字符串关系：

```text
currentHTML == previousHTML
    -> no change

currentHTML.startsWith(previousHTML)
    -> append

otherwise
    -> replace，保存 previousHTML
```

优点是简单、低成本；缺点是对广告、随机属性、相对时间、埋点 class、计数器等 DOM 抖动敏感。一个属性变化就可能使 append 被误判为 replace。

生产平台应改用“稳定 article key 集合的新增/消失关系”判断列表行为，而不是完整 HTML 前缀。

### 5.4 几何终止条件

源码判断：

```javascript
container.scrollTop + container.clientHeight >= container.scrollHeight - 10
```

这最多说明**当前客户端滚动几何接近底部**。它不能证明：

- 服务端 `next_cursor` 已耗尽；
- 触底后不会异步扩展 `scrollHeight`；
- 当前请求未因慢网/429/脚本异常漏加载；
- 所有历史页都被覆盖。

因此几何触底只能映射为 `HEURISTIC_EXHAUSTED`，不能单独产生 `ATTESTED_COMPLETE`。

### 5.5 窗口合并与去重

发生 replace 后，每个 chunk 被放进临时 `div`，源码遍历的是：

```javascript
tempDiv.children
```

即**直接子元素**，再按以下键去重：

```javascript
element.innerText
  .toLowerCase()
  .replace(/[\s\W]/g, '')
```

最后保存 `outerHTML` 并重写容器。

这有四类风险：

1. **相同可见文本、不同 href**：不同文章可能被合并；
2. **CJK/Unicode 风险**：中文、日文等字符会被 `\W` 大量去掉，纯 CJK 文本可能归一化为空字符串；
3. **嵌套结构风险**：真实卡片若位于 wrapper 的下层，直接子元素可能是整组容器，而不是文章卡片；
4. **语义丢失**：隐藏的 `data-id`、href、日期属性等稳定身份没有参与主去重键。

生产 URL Inventory 的身份优先级应是：

```text
normalized_article_url
> source item id / data-id / API id
> canonical link target
> title + published_at + author composite
> stable structural fingerprint
> Unicode-aware normalized text（仅弱辅助）
```

绝不能把原生 normalized `innerText` 当主键。

### 5.6 失败语义

`_handle_virtual_scroll()` 外层会捕获异常并记录日志，然后普通 crawl 仍可能继续并返回 HTML。

这对通用 SDK 是合理的 graceful degradation；对 Coverage 却意味着：

```text
crawl success != virtual archive success
virtual archive success != provider complete
```

此外，Virtual Scroll 的内部返回值主要用于日志，并没有形成平台可直接消费的每步结构化 Observation。平台无法仅依赖最终 CrawlResult 重建每次滚动的稳定身份、cursor 和停止证据。

---

## 6. 原生实现适合什么，不适合什么

### 6.1 适合

- 新站 onboarding 的快速 Probe；
- 已知简单虚拟列表的合成 HTML 恢复；
- 为正文/链接抽取器生成一个 Derived Candidate；
- 作为平台自研滚动执行器的行为对照和回归基线。

### 6.2 不适合直接承担

- 全历史 Coverage Authority；
- CJK/多语言 URL Inventory 去重；
- protocol-level completion；
- 每步 durable checkpoint；
- reverse/upward virtual list；
- 复杂事件等待；
- 浏览器 crash 后事务式继续；
- 需要滚动后继续依赖真实框架节点交互的 Recipe。

---

## 7. 官方测试覆盖与缺口

官方 `tests/test_virtual_scroll.py` 构造了 1000 条虚拟列表数据，每屏 10 条，滚动时替换容器内容，配置约 120 次滚动，最后按 `data-index` 检查是否收集 0~999。

这个测试证明：**在“ASCII 文本唯一、结构规则、立即替换、向下滚动”的理想 fixture 中，原生算法能把窗口内容重新收集到最终 HTML。**

但它没有覆盖生产知识库最关键的场景：

- 纯中文/日文卡片导致 normalized text 空键；
- 相同标题不同 href；
- 卡片嵌套 wrapper；
- DOM 属性随机抖动；
- 加载延迟大于固定 sleep；
- 触底后才发下一批网络请求；
- selector 指到不可滚动节点；
- 实际 scroll delta 为 0；
- 反向/向上加载；
- Runtime Virtual Scroll 失败但外层 crawl 成功；
- 合成 `innerHTML` 后继续执行 JS/点击；
- final screenshot 是 synthetic DOM 而非 raw step。

平台 Contract Test 必须补齐这些场景。

---

## 8. 推荐的生产架构：平台拥有 VirtualArchiveExecutor

### 8.1 角色边界

```text
DynamicArchiveProvider
  |
  +-- STATIC_PAGINATION -> HTTP
  +-- LOAD_MORE         -> Deterministic Browser Recipe
  +-- INFINITE_APPEND   -> Platform Archive Executor
  `-- VIRTUAL_REPLACE   -> Platform VirtualArchiveExecutor
                              |
                              +-- 每步 item identity observation
                              +-- wait/progress/cursor evidence
                              +-- durable checkpoint
                              +-- typed stop/completion
                              `-- 可选调用 Crawl4AI 生成 Derived Merged HTML
```

核心原则：**先在真实步骤上采集文章身份，再允许 Runtime 合并 DOM。**

### 8.2 单步状态机

推荐每步执行：

```text
1. 验证 container/item selector 与 scope
2. 读取 scrollTop/clientHeight/scrollHeight
3. 从当前真实 DOM 提取 stable item keys + href + metadata
4. 持久化 PRE_SCROLL Observation
5. 执行一次受控 scroll/click
6. 等待 meaningful progress：
   - 新 item key
   - MutationObserver quiet window
   - 目标 XHR/fetch 完成
   - cursor 变化
   - network quiet
   固定 sleep 仅兜底
7. 读取实际 scroll delta、网络/cursor、当前 item keys
8. 持久化 POST_SCROLL Observation
9. 计算新增身份、重复、模式漂移、collision risk
10. 判断协议结束/启发式停止/预算耗尽
11. 写 durable checkpoint
12. 进入下一步
```

每个 mutating action 后重新采样，不能跨多个 DOM 变化复用旧元素身份。

### 8.3 方向支持

平台 Recipe 增加：

```text
scroll_direction = down | up
```

以支持“最新在底部、向上加载更旧内容”或聊天式归档。原生 `VirtualScrollConfig` 只适合作为 down-scroll derived fallback。

---

## 9. 数据与证据模型

每一步建议保存：

```text
dynamic_archive_step(
  run_id,
  attempt_id,
  step_no,
  observed_at,
  representation_kind,          -- RAW_STEP / DERIVED_MERGED / DERIVED_VIEW
  scroll_direction,
  scroll_requested_px,
  scroll_top_before,
  scroll_top_after,
  scroll_delta,
  client_height,
  scroll_height_before,
  scroll_height_after,
  container_child_count,
  item_count,
  unique_item_count,
  new_item_count,
  item_key_set_hash,
  item_identity_artifact_id,
  first_item_key,
  last_item_key,
  mutation_count,
  dom_quiet_ms,
  relevant_network_request_count,
  cursor_before,
  cursor_after,
  end_marker_seen,
  wait_condition,
  dom_artifact_id,
  screenshot_artifact_id,
  runtime_branch,
  runtime_chunks_count,
  runtime_unique_count,
  stop_reason
)
```

文章 URL 单独写标准：

```text
url_observation(
  source_id,
  provider_run_id,
  provider_step_id,
  observed_url,
  stable_item_key,
  provider_metadata,
  evidence_artifact_id
)
```

原生 Crawl4AI 合成结果必须标记 `DERIVED_MERGED`，最终 screenshot 若发生在合成之后标记 `DERIVED_VIEW`，不能覆盖 `RAW_STEP` 证据。

---

## 10. 完整性与停止原因

统一 completion：

```text
ATTESTED_COMPLETE
HEURISTIC_EXHAUSTED
PARTIAL
FAILED
BLOCKED
```

只有协议级证据可以给 `ATTESTED_COMPLETE`：

- API `has_more=false` / `next_cursor=null`；
- 服务端 total 与稳定 ID 数一致；
- 明确 end marker 且 Recipe 有稳定 Contract；
- 确定性分页协议明确结束。

新增/保留 typed stop reason：

```text
END_MARKER
CURSOR_EXHAUSTED
SERVER_TOTAL_REACHED
NO_NEXT_PAGE
STABLE_NO_NEW_ITEMS
NO_ITEM_IDENTITY_PROGRESS
SCROLL_GEOMETRY_END
SCROLL_STALLED
NON_SCROLLABLE_CONTAINER
SCROLL_BUDGET_EXHAUSTED
WALL_TIME_EXHAUSTED
NETWORK_BUDGET_EXHAUSTED
PRECONDITION_FAILED
CONTAINER_NOT_FOUND
SELECTOR_AMBIGUOUS
WAIT_TIMEOUT
UNSUPPORTED_SCROLL_DIRECTION
RUNTIME_MERGE_UNSAFE
VIRTUAL_SCROLL_RUNTIME_ERROR
POLICY_DENIED
```

`STABLE_NO_NEW_ITEMS`、`NO_ITEM_IDENTITY_PROGRESS`、`SCROLL_GEOMETRY_END` 默认只能形成 `HEURISTIC_EXHAUSTED`。

---

## 11. CJK / 多语言安全门

必须新增 Runtime Merge Guard：

1. 根据 `language_profile` 判断 CJK/多语言风险；
2. 对原生 normalized-text key 计算空键比例和碰撞率；
3. 若存在高风险，禁止把 Crawl4AI merged DOM 的去重计数用于 Coverage；
4. URL Inventory 只接受平台 stable item key Observation；
5. merged HTML 仍可进入抽取 Candidate，但 provenance 明确为 derived；
6. 质量门监控“merged candidate 中 href 数”和“platform unique item keys”差异。

建议指标：

```text
dynamic_archive_runtime_text_key_empty_ratio
dynamic_archive_runtime_text_key_collision_total
dynamic_archive_identity_collision_total
dynamic_archive_merged_href_loss_total
dynamic_archive_unicode_guard_trigger_total
```

---

## 12. 增量同步

动态归档不应每次从头滚到历史底部。

保存：

```text
newest_known_article_key
newest_known_published_at
archive_head_fingerprint
recent_known_item_keys
last_incremental_archive_run
last_forced_deep_scan
```

增量从 archive head 开始。当连续出现 `K` 个已知稳定 item key，且 Source 的排序单调性、置顶策略、乱序风险满足政策时可 early stop。

防漏：

- 置顶条目单独识别；
- 定期 forced deep scan；
- Sitemap/CMS/RSS/Archive 多 Provider reconcile；
- 若 deep scan 持续发现漏 URL，降低该 Source 的 archive trust，扩大窗口或关闭 aggressive early-stop；
- 对 reverse list 使用方向对应的高水位；
- 文章正文是否变化仍由 Revalidation Engine 判断，不能由归档卡片“没变化”推导正文没变化。

---

## 13. 断点恢复

浏览器 session 不是 durable state。持久 checkpoint 保存：

```text
last_unique_item_key
oldest/newest_published_at_hint
unique_item_count
step_no
last_item_key_set_hash
last_cursor_hint
scroll_direction
recipe_release
fetch_variant_key
```

恢复时：

- 若已识别底层 API/cursor，直接转成 cursor Provider；
- 否则新建浏览器 Attempt，确定性重放到 checkpoint 附近；
- 重放期间仍按 item identity 幂等去重；
- 不把新浏览器伪装成原浏览器事务继续。

---

## 14. Web 管理功能

Source 的 `Dynamic Archive` 页面应展示：

- mode、scroll direction、Recipe release；
- container/item selector；
- 当前 step、actual scroll delta、unique/new item 数；
- wait condition、mutation/network/cursor 进展；
- Completion Type / Stop Reason；
- `RAW_STEP`、`DERIVED_MERGED`、`DERIVED_VIEW` 证据标签；
- CJK/Unicode merge guard 与 collision 告警；
- browser minutes、新 URL 成本；
- 与 Sitemap/CMS/RSS/Common Crawl/Wayback 的 URL 集合差异；
- Force Probe、Incremental、Deep Backfill、Pause、Resume、Reclassify、Replay。

Web 管理端只编辑声明式参数；不得直接透传任意 JS/CDP/代理/secret。

---

## 15. 调度、成本与安全

资源池分离：

```text
archive-http-static
archive-browser-dynamic
fetch-browser-article
agentic-repair
```

Dynamic Archive 默认不得抢占热增量正文 Revalidation。

硬预算至少包括：

```text
max_scrolls_per_attempt
max_wait_per_step_s
max_wall_time
max_network_requests
max_unique_items
max_dom_artifact_bytes
max_browser_minutes_per_source_day
```

配置编译器必须 clamp：

- selector 长度/复杂度；
- scroll direction；
- scroll px/range；
- wait policy；
- wall/network/artifact/browser budget；
- 域名、路径、下载、跳转、popup/iframe 权限。

网页内容不能修改这些能力边界。

---

## 16. 可观测性与告警

核心指标：

```text
dynamic_archive_run_total{mode,result}
dynamic_archive_step_total
dynamic_archive_unique_item_total
dynamic_archive_new_url_total
dynamic_archive_actual_scroll_delta
dynamic_archive_no_progress_step_total
dynamic_archive_stop_total{reason}
dynamic_archive_completion_total{type}
dynamic_archive_mode_mismatch_total
dynamic_archive_selector_failure_total
dynamic_archive_browser_minutes
dynamic_archive_replay_steps_total
dynamic_archive_unicode_guard_trigger_total
dynamic_archive_runtime_text_key_empty_ratio
dynamic_archive_runtime_text_key_collision_total
```

告警：

- 连续 `CONTAINER_NOT_FOUND` / `NON_SCROLLABLE_CONTAINER`；
- actual scroll delta 长期为 0；
- unique item 增长突降；
- append/replace 模式漂移；
- CJK text-key 空键/碰撞率升高；
- forced deep scan 持续发现漏 URL；
- SCROLL_BUDGET_EXHAUSTED 比例升高；
- Browser minutes / 新 URL 成本异常。

---

## 17. Contract Test 矩阵

固定 Crawl4AI release 和 Browser build，至少覆盖：

1. 静态分页；
2. append infinite scroll；
3. replace virtual scroll；
4. 同标题不同 href；
5. 纯中文卡片、纯日文卡片、中英混排；
6. 多个 CJK 卡片经原生 normalized text 变空/碰撞；
7. article card 嵌套 wrapper；
8. DOM 属性随机变化导致 `startsWith()` 误判；
9. 延迟加载超过固定 sleep；
10. 滚到底后异步继续扩展；
11. selector 指向不可滚动节点；
12. actual scroll delta=0；
13. reverse/upward virtual list；
14. 429/网络失败；
15. scroll/wall/network budget 耗尽；
16. Virtual Scroll 异常但外层 crawl 返回 success；
17. 合成 `innerHTML` 后事件监听器丢失；
18. post-scroll `js_code` 不得误当真实 DOM 操作；
19. final screenshot provenance 为 derived；
20. crash 后 semantic checkpoint 重放；
21. 增量顶部存在置顶、乱序、历史插入文章。

平台不变量必须验证：

```text
Runtime success 不推进 Provider Completion
Runtime merged uniqueCount 不等于平台 URL Inventory uniqueCount
CJK merged candidate 不拥有 Coverage 权威
Derived Artifact 永不覆盖 Raw Step Evidence
```

---

## 18. 对最终技术方案的要求

最终方案应明确采用：

1. `DynamicArchiveProvider` 负责动态归档 Discovery；
2. **平台自有 `VirtualArchiveExecutor` 为 Coverage 主路径**，每步先采稳定文章身份；
3. Crawl4AI 原生 `VirtualScrollConfig` 降为 Derived HTML 恢复/候选能力，不拥有完整性真相；
4. 增加 Unicode/CJK Merge Guard，避免原生 `\W` 去重造成静默丢文；
5. 增加 actual scroll delta、item identity、mutation/network/cursor 的每步证据；
6. 支持 up/down scroll direction；
7. 用事件/身份进展等待替代固定 sleep 作为主判据；
8. 把 synthetic merged DOM 和 screenshot 标记为 Derived；
9. 增加 `SCROLL_STALLED`、`NON_SCROLLABLE_CONTAINER`、`NO_ITEM_IDENTITY_PROGRESS`、`RUNTIME_MERGE_UNSAFE` 等停止原因；
10. 增量同步继续采用 stable-key high-water + forced deep scan + 多 Provider reconcile；
11. Browser session 不作为断点，使用 semantic checkpoint；
12. Web Admin、指标和 Contract Test 显式暴露多语言碰撞、模式漂移、实际滚动进展和证据来源。

---

## 19. 最终判断

Virtual Scroll 不是普通的“滚到底再取 HTML”。它本质上把一个时间序列中的多个 DOM 窗口合成为单一派生表示。这个能力对内容恢复很有价值，但其默认文本去重、固定等待、几何终止、直接子元素合并和失败降级语义，都不足以支撑生产级历史 Coverage。

对 1000+ 技术博客，正确架构是：**平台在每个真实滚动步骤先捕获稳定 article identity 和证据，Crawl4AI 负责浏览器能力与可选合成 HTML；平台负责 Inventory、Completion、Checkpoint、Incremental、Unicode 安全、审计和恢复。** 这样既能覆盖 React/Vue/windowed list，又不会把 Runtime 启发式误当业务真相。