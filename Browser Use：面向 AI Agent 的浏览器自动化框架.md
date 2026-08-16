# Browser Use：面向 AI Agent 的浏览器自动化框架

## 1. 调研对象

- 项目：Browser Use
- 地址：https://github.com/browser-use/browser-use
- 源码基线：`main`，调研时 HEAD `f3298c559aabb327a61cf6a9caef5ea3462f45de`
- 项目版本：`0.13.7`
- Python：`>=3.11,<4.0`
- 目标场景：1000+ 技术博客全历史回填、持续增量同步、Markdown 知识库、Web 管理、新增站点低成本扩展。

结论：Browser Use 不适合替代 HTTP/Crawl4AI/确定性 Playwright 的批量抓取主链；它最适合作为 **Agentic Browser Repair / Recipe Studio**，用于少量复杂站点的交互探索、诊断和 Recipe Candidate 生成。其 Agent 产物必须经过平台的 Selector/Recipe Compiler 和确定性 Promotion Gate 后才可成为生产 Recipe。

本次进一步确认一个容易被忽略的实现事实：**Browser Use 当前内置 Action Loop Detection 是 soft detection，只向 LLM 注入提示，不会真正阻止重复动作。** 因此生产平台必须在 Agent Loop 外部增加独立、版本化、不可被 prompt 修改的 **Agentic Progress Guard**，对循环、状态停滞、无新增 Observation 和预算耗尽进行硬终止。

---

## 2. 项目定位与适用边界

Browser Use 的核心是让 LLM Agent 驱动真实 Chromium 浏览器完成多步骤网页任务。它把浏览器状态压缩为模型可理解的交互上下文，模型输出结构化动作，Runtime 执行动作，再重新观察状态进入下一轮。

典型优势：

- 点击“加载更多”；
- 展开年份/归档区；
- 动态分页；
- 滚动后才出现控件；
- iframe/shadow DOM 场景；
- 根据当前页面状态动态选择下一步。

但大规模知识库抓取的主链要求高吞吐、低成本、确定性、可重放、可审计。Browser Use 每一步都可能包含页面状态采集、LLM round trip、浏览器动作和下一轮观察，因此：

1. **吞吐低**：无法与普通 HTTP GET 相比；
2. **成本高**：Browser CPU/内存 + LLM token/视觉输入；
3. **非确定性**：模型、prompt、DOM、截图、A/B 都可能改变动作路径；
4. **安全面大**：通用 Tool 能力远高于只读爬虫；
5. **Agent 完成不等于 Coverage 完整**：找到一批 URL 不能证明全站已枚举。

正确分层：

```text
HTTP Fast Lane
      |
      v
Deterministic Browser Recipe
      |
      | hard cases only
      v
Agentic Browser Repair Sandbox
      |
      +--> Evidence / Observation proposal
      +--> Selector Candidate
      `--> Recipe Candidate
               |
               v
      Deterministic Promotion Gate
               |
               v
      Production Browser Recipe
```

---

## 3. Agent 执行模型：Observe → Decide → Act → Observe

### 3.1 Agent 主循环

`browser_use/agent/service.py` 中的 Agent 由 BrowserSession、MessageManager、LLM、Tools、AgentState、AgentHistory 协同驱动。一轮 step 可抽象为：

```text
Browser State
   -> build model context
   -> LLM structured AgentOutput(actions[])
   -> execute actions
   -> collect results/errors/history/cost
   -> next state
```

与生产风险直接相关的参数包括：

- `max_actions_per_step`：默认 5；
- `max_failures`：默认 5；
- `step_timeout`：默认 180 秒；
- `llm_timeout`；
- `max_history_items`；
- `use_vision`；
- `output_model_schema`；
- `fallback_llm`；
- `calculate_cost`；
- `on_step_start/on_step_end` hooks；
- `enable_planning`；
- `planning_replan_on_stall`；
- `planning_exploration_limit`；
- `loop_detection_window`；
- `loop_detection_enabled`；
- `message_compaction`。

`output_model_schema` 只能约束输出结构，不能把模型推理路径变成确定性流程。对平台最合理的用途是约束 `RecipeCandidate`/诊断结果 schema。

### 3.2 一步多动作与 stale DOM

Browser Use 支持一个模型响应返回多个 action。`multi_act()` 会在导航/切 tab 等终止型动作后停止同批动作，并在每个动作后检查 URL 和 focus target 是否变化。

这能减少“导航后继续点击旧页面 index”的风险，但无法覆盖：

```text
URL 未变
focus target 未变
React/Vue 原地重绘 DOM
selector map 已重新编号
```

例如点击“加载更多”后列表原地重建，后续 action 仍拿旧 index，就可能点击错误元素。

因此：

- 探索模式可允许 multi-action；
- **候选录制/编译默认 `max_actions_per_step=1`**；
- 任意改变 DOM/URL/focus target 的动作后强制重新采样 BrowserState；
- Production Recipe 每个 mutating action 都是明确状态边界；
- Agent step 不是平台事务或幂等边界。

### 3.3 ActionLoopDetector：软检测，而不是硬终止

本次重点核对了 `browser_use/agent/views.py` 中的 `ActionLoopDetector` 以及 `browser_use/agent/service.py` 的注入逻辑。

源码对该类的定义明确说明：

> 它是 soft detection system，只为 LLM 生成上下文提示，不阻止 action；Agent 仍然可以继续重复。

其实现包含两类信号。

#### 3.3.1 动作重复检测

每个动作先被归一化，再计算稳定 hash。Detector 保存最近 `window_size` 个 action hash，默认窗口为 20，并计算某个动作在窗口内的最大重复次数。

当前 nudge 分级大致为：

```text
>= 5  次类似动作 -> 第一级提醒
>= 8  次类似动作 -> 更强提醒
>= 12 次类似动作 -> 高强度提醒
```

`wait`、`done`、`go_back` 等动作会被排除或特殊处理，降低显然的误报。

#### 3.3.2 页面停滞检测

Detector 还记录页面 fingerprint。当前实现输入至少包括：

```text
current URL
DOM 的 LLM representation
selector_map 的元素数量
```

若连续页面 fingerprint 不变，会增加 `consecutive_stagnant_pages`；达到一定次数后向模型注入“页面没有变化”的提示。

#### 3.3.3 注入发生在上下文层

`_inject_loop_detection_nudge()` 取得 detector 的 nudge message，然后调用 MessageManager 向下一轮模型上下文追加消息；**它没有在 Browser Gateway、Task、Attempt 或 action dispatcher 层执行 hard stop。**

同样，planning/replan 也主要通过提示模型完成：

- 连续失败达到 `planning_replan_on_stall` 时，注入重新规划建议；
- 探索步骤达到 `planning_exploration_limit` 且没有计划时，注入创建 plan/done 的建议。

所以这些机制属于“提高 Agent 自我纠错概率”，不是生产平台的资源与正确性门禁。

### 3.4 为什么知识库平台必须有独立 Progress Guard

如果只依赖 Runtime 的 soft nudge：

- 模型可能忽略提醒继续点击；
- 页面可以通过 prompt injection 诱导 Agent 继续循环；
- 无限滚动场景可能每次 DOM 有微小变化，从而绕过简单 stagnant fingerprint；
- 找不到新 URL 但页面状态仍在变化时，Runtime 可能继续消耗 token/browser minute；
- “预算还没用完”与“任务有进展”是两回事。

因此平台需要独立 `Agentic Progress Guard`，运行在 LLM 决策之外。它的 hard-stop 决定不能被 prompt、model output 或页面内容覆盖。

建议每一步计算：

```text
progress_fingerprint = hash(
  normalized_url,
  focus_target_id_hash,
  dom_semantic_hash,
  discovered_url_set_hash,
  candidate_or_observation_hash,
  normalized_action_signature
)
```

同时维护：

```text
rolling action repetition
consecutive stagnant state
state fingerprint revisit count
steps without new URL/Observation
steps without useful artifact/candidate change
page/tab count
wall time
token/cost
```

平台 hard stop 示例：

```text
AGENT_ACTION_LOOP
AGENT_STATE_STAGNANT
NO_NEW_OBSERVATION_PROGRESS
REPEATED_STATE_CYCLE
PLANNING_STALL
MAX_STEPS
WALL_TIME_EXHAUSTED
TOKEN_BUDGET_EXHAUSTED
COST_BUDGET_EXHAUSTED
```

Browser Use 内置 detector/replan 仍可保留为 advisory signal，并写 Evidence，帮助 Operator 理解 Agent 在 hard stop 前已经收到过哪些提示。

---

## 4. BrowserSession：EventBus + Watchdog + CDP

### 4.1 事件驱动控制层

BrowserSession 不是简单 Playwright wrapper，而是围绕 EventBus 建立浏览器事件层。不同 Watchdog 负责浏览器生命周期、DOM、安全、下载、弹窗、截图、录制、权限、Storage State、崩溃等。

抽象：

```text
Agent / Tool intent
       |
       v
Browser Event
       |
       +--> Security Watchdog
       +--> DOM Watchdog
       +--> Download Watchdog
       +--> Crash / Session Watchdog
       +--> Recording / Screenshot ...
       |
       v
CDP / Actor execution
```

对平台最值得吸收的思想：**LLM 不拿任意 Playwright/CDP 对象，只调用平台批准的高层动作。** Browser Gateway 在执行前重新验证 Capability、Scope、预算、Progress Guard 和副作用。

### 4.2 Watchdog 恢复不等于业务恢复

Browser/CDP reconnect 解决连接层恢复，不等于 durable task recovery：

- 连接恢复不代表 LLM 会走同样路径；
- AgentHistory 不能替代 Task/Attempt/Lease/Fencing Token；
- in-memory session/event/history 不能成为业务 checkpoint；
- Agentic crash 应结束当前 Attempt，保存已晋升 Evidence，再按策略创建新的 Run；
- 确定性 Recipe 才适合按平台 step checkpoint 重放。

### 4.3 Actor/CDP 能力面

Browser Use Actor/CDP 层可实现导航、元素定位、click/fill/hover/focus/select/drag、JavaScript evaluate、截图、键鼠、Prompt-based element lookup、结构化 content extract。

能力上限很高，因此必须实施四层边界：

```text
Tool Registry
Browser Gateway
Container/Network Policy
Source Policy
```

---

## 5. DOM 表达与元素身份

### 5.1 DOM + Accessibility + 可见性

Browser Use 不只是把 HTML 全量塞给 LLM，而是结合 DOM、Accessibility Tree、可交互元素、布局/可见性、iframe/shadow DOM、截图等生成更适合行动的状态表示。

这使 Agent 能识别“下一页”“加载更多”“展开归档”等交互入口。

### 5.2 highlight index 是临时标识

Agent 经常使用 `click(index=...)`。index 来源于当前 DOM selector map，只属于当前页面快照：

- 插广告会改变 index；
- DOM 重绘会重建 selector map；
- locale/A-B/cookie/login 状态会改变元素集合；
- viewport/iframe/shadow DOM 也会改变观察结果。

强约束：

> Browser Use highlight index、CDP node id、backend node id 只能用于单次 Agent Run 的动作和证据关联，禁止直接进入 Production Recipe。

### 5.3 History Replay 的启发式重定位

当前 Agent history replay 会尝试重新找到历史交互元素，常见匹配层级包括：

```text
EXACT
STABLE
XPATH
AX_NAME
ATTRIBUTE
```

它适合调试与尽力复现，也适合收集 selector 候选，但不是长期 selector contract：hash/XPath/AX/name 都可能漂移、碰撞或出现歧义。

### 5.4 Selector Compiler

平台应把 Agent evidence 编译成：

```text
SelectorBundle
  semantic_role
  accessible_name
  stable_attributes
  data_attributes
  id/name candidates
  stable_text_anchor
  css_candidates[]
  xpath_candidates[]
  ancestor_constraints[]
  frame_scope
  expected_cardinality
  fallback_order[]
  selector_fingerprint
  fixture_match_rate
  live_sample_match_rate
  ambiguity_rate
  mutation_survival_rate
```

优先稳定业务属性、role + accessible name、稳定 `data-*`、稳定 id/name、语义文本锚点；随机 class、nth-child、纯位置只能最后 fallback。

---

## 6. Tool Registry 与能力最小化

Browser Use `Tools` 使用 Pydantic schema 暴露结构化动作，也允许注册自定义动作。通用 Tool 能力包含导航、click/input、upload、scroll、send_keys、evaluate JS、tab、extract、screenshot、dropdown、文件读写等。

对知识库 Agentic Sandbox 应只开放：

```text
NAVIGATE_APPROVED
WAIT_SELECTOR
CLICK_READONLY
SCROLL_BOUNDED
EXPAND
NEXT_PAGE
SWITCH_APPROVED_TAB
EXTRACT_LINKS
ASSERT_URL
ASSERT_SELECTOR
CAPTURE_DOM
CAPTURE_SCREENSHOT
STOP
```

默认拒绝：

- 任意 JS/evaluate；
- 任意 CDP；
- 文件读写；
- upload/download；
- 表单提交；
- 发布/删除/支付；
- 未审批跨 host；
- 页面临时要求增加工具/权限。

---

## 7. Browser 安全默认值不能直接作为平台安全边界

Browser Use 已提供 allowed/prohibited domains、导航前后检查、新 tab 检查、IP blocking 等能力，这些应保留，但平台不能依赖第三方默认值。

Agentic Sandbox 必须显式编译：

```yaml
allowed_domains: <Source Scope>
block_ip_addresses: true
accept_downloads: false
auto_download_pdfs: false
permissions: []
persistent_profile: false
allow_data_urls: false
allow_blob_urls: false
allow_file_scheme: false
allow_arbitrary_cdp: false
allow_arbitrary_js: false
allow_upload: false
allow_clipboard: false
```

网络层再提供独立 SSRF 防线：

- egress allowlist；
- DNS 后阻止 loopback/private/link-local/metadata；
- redirect 每跳重验；
- 防 DNS rebinding；
- Proxy 只允许平台托管 endpoint；
- remote CDP endpoint 不暴露给模型。

---

## 8. Prompt Injection：网页是数据，不是 Policy

网页正文、DOM 属性、Accessibility name、隐藏文本、截图文字都视为不可信输入。

平台规则：

1. System task / Tool Policy / Source Scope / Secret Policy 与网页内容隔离；
2. 模型即使输出越权动作，Gateway 也必须拒绝；
3. public blog 默认无 secret；
4. authenticated Source 仅使用 Source-scoped secret reference；
5. Cookie/Authorization/Storage State 不进入普通 trace/model memory；
6. 恶意网页 Fixture 验证不能扩大工具权限或外传数据；
7. Progress Guard 与 Security Policy 都在 LLM 外部执行，页面不能修改阈值或关闭保护。

---

## 9. Storage State 与 fetch variant

公开博客默认使用临时无认证 Profile。需要认证时按 Source 隔离、加密、TTL 管理 Storage State。

`fetch_variant_key` 至少考虑：auth/cookie class、locale、timezone、UA、proxy/region、browser profile、session/interaction class。

Agentic Run、Replay、Shadow、Canary 必须在兼容 variant 下比较，否则“Agent 成功、生产 Recipe 失败”难以解释。

---

## 10. Agentic Evidence 与可观测性

### 10.1 Run

```text
agentic_repair_run
  run_id / source_id / url_id / fetch_variant_key
  trigger_reason
  browser_use_release / browser_build_release
  model_release / prompt_release
  tool_policy_release / observation_schema_release
  security_policy_release / progress_guard_release
  max_steps / max_actions / wall_time / token / cost
  started_at / finished_at
  outcome / stop_reason
```

### 10.2 Step Evidence

```text
agentic_step_evidence
  run_id / step_no
  pre_state_hash / post_state_hash
  current_url / focus_target_id_hash
  progress_fingerprint
  new_observation_count
  loop_repetition_count
  stagnant_state_count
  state_revisit_count
  internal_loop_nudge_level
  progress_guard_decision
  dom_snapshot_artifact_id
  accessibility_snapshot_artifact_id
  screenshot_artifact_id
  network_trace_artifact_id
  actions_redacted[]
  policy_decisions[]
  interacted_element_evidence[]
  action_results[]
  destination_urls[]
  history_match_level
  token_usage / estimated_cost / duration_ms
```

DOM/AX/screenshot/HAR/trace/video 必须先经敏感字段、大小和身份校验后再晋升平台对象存储。

---

## 11. 从 Agent 探索到 Production Recipe

### 11.1 触发条件

只在以下情况进入 Agentic Repair：

- HTTP 与确定性 Browser Recipe 都失败；
- 归档 URL 需要多步骤交互才能暴露；
- Source 长期出现同类 REPAIR/REVIEW；
- onboarding 探索少量样本；
- Operator 主动诊断。

禁止对全站 URL 默认开启。

### 11.2 Recipe Candidate

```yaml
recipe_candidate:
  source_id: example
  purpose: archive_discovery
  entry: /blog/archive
  fetch_variant_class: public-default
  steps:
    - wait_selector:
        selector_ref: main_content_v1
        timeout_ms: 10000
    - repeat:
        max_iterations: 40
        until:
          no_new_urls_rounds: 3
        actions:
          - click:
              selector_ref: load_more_v2
          - wait_dom_change:
              timeout_ms: 5000
    - extract_links:
        selector_ref: article_links_v3
        scope: same_source
```

Agent action 先 Normalization，再把 ephemeral index 替换为 SelectorBundle，移除高风险动作，补全 wait/assert/loop bound，形成 Recipe Candidate + provenance。

### 11.3 Promotion Gate

```text
Recipe Candidate
 -> Schema / Static Policy Validation
 -> Security Validation
 -> Selector Ambiguity Validation
 -> Fixture Replay (no LLM)
 -> Repeated Replay
 -> Sampled Live Replay
 -> Shadow
 -> Source Canary
 -> Approval
 -> recipe_release
```

如果某一步仍需要“再问一次 LLM 下一步怎么做”，就不能自动晋升生产。

---

## 12. 任务恢复语义

必须区分：

```text
Browser Use rerun_history  = 尽力复现历史交互
Production Recipe replay   = 确定性 DSL 重放
Platform Task recovery     = Task/Attempt/Lease/Fencing/Artifact 恢复
```

- Agentic crash：保存已晋升 Evidence，Attempt 失败，按策略新建 Run；
- deterministic Recipe crash：可按 step checkpoint 重放；
- CDP reconnect：只记 runtime recovery，不推进 Coverage/Freshness/Version；
- history match level 只做诊断，不自动提高 Recipe 信任等级。

---

## 13. 成本与容量治理

资源池：

```text
fetch-http            高吞吐
fetch-browser         确定性 Browser Slow Lane
agentic-repair        低优先级、低并发、独立预算
```

Agentic Repair 限制：steps、actions、wall time、token、cost、page/tab、artifact bytes、Source daily quota、global slots、无新 Observation 阈值、state revisit 阈值。

Browser Use 内部 nudge 即使出现，也不能延长平台硬预算。

---

## 14. Web Recipe Studio

后台需要展示：

- URL、截图、DOM/AX、每步 action/result；
- 临时 index 对应元素证据；
- history replay match level；
- Browser Use 内部 loop/replan advisory nudge；
- 平台 Progress Guard fingerprint；
- action repetition / stagnant / state revisit / no-new-observation 计数；
- hard-stop decision/reason；
- Tool policy allow/deny；
- cross-scope / prompt injection / egress deny；
- token/browser minute/wall-time/cost；
- SelectorBundle 多样本稳定性；
- replay/shadow/canary；
- 审批、发布、回滚。

---

## 15. Release 模型

影响 Agent 行为的对象必须版本化：

```text
browser_use_release
browser_build_release
agent_model_release
agent_prompt_release
agent_tool_policy_release
agent_observation_schema_release
agent_trace_schema_release
agent_security_policy_release
agentic_repair_policy_release
agentic_progress_guard_release
browser_interaction_dsl_release
selector_strategy_release
recipe_candidate_schema_release
recipe_compiler_release
recipe_promotion_policy_release
agent_replay_policy_release
```

Browser Use/Chromium/model/prompt/tool/progress guard 任一变化都可能改变探索行为，必须进入 Contract Test；已发布 deterministic Recipe 不因 Agent Runtime 升级自动改变。

---

## 16. Metrics 与告警

新增：

```text
agentic_repair_run_total
agentic_repair_success_total
agentic_repair_budget_exhausted_total
agentic_repair_browser_minutes
agentic_repair_llm_tokens
agentic_repair_estimated_cost
agentic_internal_loop_nudge_total
agentic_progress_guard_stop_total{reason}
agentic_progress_fingerprint_revisit_total
agentic_stagnant_step_total
agentic_policy_denied_total
agentic_cross_scope_block_total
agentic_prompt_injection_block_total
agentic_browser_reconnect_total
agentic_history_match_total{level}
recipe_candidate_total
recipe_candidate_uncompilable_total
selector_ambiguity_total
selector_validation_pass_ratio
recipe_replay_pass_ratio
recipe_shadow_regression_total
recipe_canary_failure_total
recipe_promotion_total
recipe_rollback_total
```

重点告警：

- Progress Guard hard stop 突增；
- 同一 Source `NO_NEW_OBSERVATION_PROGRESS` 持续出现；
- Browser Use 内部 nudge 大量出现但 hard guard 仍长时间未结束；
- policy deny/cross-scope/injection 增长；
- Browser Use 升级后 selector/history match 退化；
- Agentic cost 激增；
- Candidate 多但 promotion pass ratio 低；
- canary URL/正文质量回归；
- trace/HAR/screenshot secret 泄露。

---

## 17. Browser Use Runtime Contract Test

每次 Browser Use/Chromium/model/prompt/tool/observation/selector/progress guard 升级至少验证：

### 17.1 Agent/Action Contract

- recording `max_actions_per_step=1`；
- multi-action 在导航/URL/target/DOM 变化后重新采样；
- output schema 失败显式失败；
- step/action/page/tab/wall-time/token/cost 全部可硬中止；
- **Runtime `ActionLoopDetector` 的 soft nudge 不被当作 hard stop；**
- 重复相同动作 fixture 被平台 `AGENT_ACTION_LOOP` 强制结束；
- 页面 fingerprint 长期不变 fixture 被 `AGENT_STATE_STAGNANT` 结束；
- 页面不断微变但没有新 URL/Observation fixture 被 `NO_NEW_OBSERVATION_PROGRESS` 结束；
- state A→B→A→B 循环被 `REPEATED_STATE_CYCLE` 结束；
- hard-stop 后模型不能通过下一条 action 继续执行；
- Evidence 包含 fingerprint/counter/decision。

### 17.2 Element/Replay Contract

- highlight index 改变时 history replay 只做诊断；
- EXACT/STABLE/XPATH/AX_NAME/ATTRIBUTE 正确记录；
- Production Recipe 不含 highlight index/node id；
- 同名/重复 selector 不被错误当唯一定位；
- nth-child/随机 class 无稳定 fallback 时 Promotion 失败；
- 无法编译临时引用返回 `UNCOMPILABLE_ELEMENT_REFERENCE`。

### 17.3 Security Contract

- approved domain 正常；
- 直接/redirect/new tab 跨域阻断；
- IP/localhost/private/link-local/metadata 阻断；
- data/blob/file 按平台策略阻断；
- upload/download/file/evaluate/CDP/clipboard 默认不可用；
- Storage State 不跨 Source/variant；
- 恶意网页不能扩大 Tool/Scope/Secret；
- egress policy 在 Runtime 层失效时仍兜底。

### 17.4 Recovery / Artifact Contract

- CDP reconnect 不错误推进 Task/Freshness/Coverage；
- crash 保存已晋升 Evidence；
- Agent rerun 不标 deterministic resume；
- Agent extract/final text 不绕过 Raw Artifact pipeline。

---

## 18. 对博客知识库方案的优化结论

本次源码级调研确认并固化以下不变量：

1. **Browser Use 只进入 Agentic Repair Plane，不进入默认批量抓取主链；**
2. **Agentic Repair Is Proposal, Not Truth；**
3. **highlight index/node id 是临时元素身份，必须编译为 SelectorBundle；**
4. **history replay 是启发式诊断，不是 Production Replay/Task Recovery；**
5. **任何会改变页面状态的动作后重新采样；**
6. **网页内容是不可信 prompt 输入，Capability 在执行层强制；**
7. **第三方 Browser 默认值不是平台安全边界；**
8. **Agent extract/final text 不是 canonical 文章正文；**
9. **所有 Agent Contract 都版本化并过 Contract Test；**
10. **新增：Soft Agent Heuristics Are Not Hard Limits。Browser Use 的 ActionLoopDetector、planning/replan 都只能作为 advisory signal，平台必须用 Agentic Progress Guard 在 LLM 外部硬终止循环、停滞、无进展与预算耗尽。**

---

## 19. 最终结论

Browser Use 的价值不是“替代爬虫”，而是把工程师手工分析长尾站点的过程转化为可审计的 Agentic 探索，再把成功探索编译成可确定性重放、可测试、可灰度、可回滚的 Production Recipe。

推荐分层：

```text
HTTP Fast Lane
    |
    v
Deterministic Browser Slow Lane
    |
    | hard cases only
    v
Agentic Browser Repair Sandbox
    |
    +--> Browser Use advisory loop/planning signals
    +--> Platform Agentic Progress Guard (hard stop)
    +--> immutable evidence
    |
    v
Selector Compiler + Recipe Compiler
    |
    v
Static/Security/Fixture/Live Replay
    |
    v
Shadow -> Canary -> Approval
    |
    v
Production Recipe Release
```

对于 1000+ 技术博客、百万到千万 URL 的知识库，主链应坚持：**Coverage 可证明、Raw Artifact 不可变、Freshness 有强证据、Production Recipe 可确定性重放、Markdown 可从 Artifact 重建、Agent 永远不直接拥有业务真相，且 Agent 的循环/停滞由平台硬保护而不是等待模型自我纠错。**
