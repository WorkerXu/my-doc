# Browser Use：面向 AI Agent 的浏览器自动化框架

## 1. 调研对象

- 项目：Browser Use
- 地址：https://github.com/browser-use/browser-use
- 来源关系：本项目由调研索引中的“为 AG2 添加网页浏览能力”文章直接引出；该文章把 Browser Use 与 Crawl4AI 作为两类网页能力进行集成演示。当前调研表中的原 71 项均已完成，因此选取这一尚未单独调研、且与复杂网页抓取强相关的项目继续深入分析。
- 调研目的：判断 Browser Use 是否适合 1000+ 技术博客全历史抓取与增量同步，以及它能否补强现有方案中的 Browser Slow Lane、复杂交互站点诊断和 Recipe 自动生成能力。

## 2. 项目定位

Browser Use 的核心不是“高吞吐网页爬虫”，而是“让 LLM Agent 驱动真实浏览器完成多步骤网页任务”。典型任务由自然语言目标开始，Agent 反复观察浏览器状态、调用模型生成下一步结构化动作、执行动作，再基于新状态继续决策，直到完成、失败或达到预算上限。

这与 Crawl4AI、Trafilatura、普通 Playwright 抓取的定位不同：

- Crawl4AI / HTTP 抓取更适合批量、确定性、可重放的网页获取与内容抽取；
- Browser Use 更擅长“页面必须先操作才能到达目标内容”的任务，例如点击“加载更多”、展开归档、切换分页、进入多层导航、处理动态组件、根据页面状态选择下一步动作；
- Browser Use 的核心路径包含 LLM 推理，因此天然带来成本、延迟、非确定性和错误动作风险，不适合作为 1000 个博客的默认抓取主链。

因此，本项目最有价值的落点不是替换现有抓取层，而是增加一个 **Agentic Browser Repair / Recipe Studio（智能浏览器修复与 Recipe 工作台）**，专门处理确定性抓取失败的少数复杂站点，并把成功探索过程“编译”为可验证、可版本化、可重复执行的确定性 Recipe。

## 3. 当前实现架构与工作原理

### 3.1 Agent 闭环：Observe → Decide → Act → Observe

Browser Use 当前 `Agent.step()` 的实现清晰分成几个阶段：

1. 获取浏览器状态并准备上下文；
2. 调用 LLM 生成下一步结构化动作；
3. 执行动作；
4. 后处理、记录结果并进入下一步；
5. 对异常、暂停、停止、失败重试统一处理。

源码中 `_prepare_context()` 会通过 `BrowserSession.get_browser_state_summary()` 获取页面状态，并把截图、页面动作、前一步结果、计划、敏感数据约束等交给 MessageManager 形成下一轮模型输入。`_get_next_action()` 调用模型并受到 `llm_timeout` 约束；`_execute_actions()` 再执行模型输出的动作列表。

这种闭环的意义是：它不要求开发者事先把一个复杂网站的所有状态机写死。只要 Agent 能从当前页面状态判断下一步，就可以完成“先找入口，再点击，再滚动，再判断有没有下一页”这类开放式流程。

但对知识库采集而言，这个优势也正是风险来源：同一个页面在不同模型、提示词、DOM 微调、广告插入甚至截图差异下，可能产生不同动作。因此 Agent 行为只能作为“探索与修复证据”，不能直接成为 Coverage、Freshness、Document Version 或 canonical Markdown 的业务真相。

### 3.2 BrowserSession：事件驱动控制层 + CDP 执行层

`BrowserSession` 的源码明确采用两层结构：

- 上层：面向 Agent / Tool 的事件驱动接口；
- 下层：直接通过 CDP（Chrome DevTools Protocol）控制浏览器目标、页面和会话。

它使用 EventBus 分发导航、标签页、错误、下载等事件，并维护浏览器 Target 与 CDP Session。这个设计的优点是把“Agent 想做什么”与“浏览器底层如何执行”分离，便于插入 Watchdog、安全检查、日志和恢复逻辑。

对我们的平台而言，这个思想值得复用：Agentic Repair 不应直接获得任意 Playwright/CDP 权限，而应通过受控 Browser Gateway 发出高层动作，让平台在执行前检查 Source Scope、Host allowlist、预算、下载权限、Cookie/Secret 策略和副作用风险。

### 3.3 DOM 感知：DOM + Accessibility Tree + 页面快照

Browser Use 的 DOM 服务并非只把 `document.body.innerText` 交给模型。当前实现会结合：

- DOM 节点；
- Accessibility Tree（AX Tree）；
- 页面布局/快照信息；
- 可点击元素检测；
- viewport、元素可见性、iframe、shadow root 等信息。

例如 DOM 服务会识别 iframe 内隐藏的可交互元素，估算需要滚动多少页才能看到它们，并限制输出数量以防上下文膨胀。它还使用浏览器布局指标处理 CSS viewport 与设备像素的差异。

这说明 Browser Use 的关键技术并不是“让 LLM 看 HTML”，而是先把浏览器当前状态压缩成更适合 Agent 操作的 **交互语义视图**。对于复杂博客归档页，这种语义视图可以帮助 Agent 发现“下一页”“更多文章”“展开年份”等动态入口。

### 3.4 Tool Registry：结构化动作而不是自由文本命令

Browser Use 的 Tool Registry 使用 Pydantic 动态生成/校验动作参数模型。动作函数被标准化后，模型必须按结构化 schema 提供参数，而不是输出一段随意的脚本。

当前实现还会按页面 URL 过滤可用动作，使模型只看到当前页面允许的工具。Agent 支持自定义 Tool，也能使用 Pydantic `output_model_schema` 约束最终输出。

这对我们的 Recipe Studio 很重要：应当进一步收紧动作空间，只开放对抓取有意义且副作用可控的 DSL，例如：

```text
NAVIGATE(url)
WAIT_SELECTOR(selector, timeout)
CLICK(selector | stable_text_anchor)
SCROLL(max_viewports)
EXPAND(selector)
NEXT_PAGE(selector)
EXTRACT_LINKS(scope_selector)
ASSERT_URL(pattern)
ASSERT_SELECTOR(selector)
STOP(reason)
```

默认禁止任意 JS、表单提交、文件上传、购买/发布/删除等高副作用动作。只有特殊 Source 经审批后才能增加能力。

### 3.5 上下文控制与循环治理

Agent 当前具有多种避免无限执行的机制，包括：

- `max_actions_per_step`；
- `max_failures`；
- `llm_timeout` / `step_timeout`；
- planning 与 replan；
- loop detection；
- message compaction；
- 最大历史项；
- 外部 stop/pause callback。

这些能力说明 Agent Loop 本身已经考虑长任务的上下文和失败控制，但平台级使用仍需增加更硬的预算边界：最大步骤数、最大浏览器分钟数、最大模型 token、最大总费用、最大跨页面数，以及每个 Source 每日 Agentic Repair 配额。

### 3.6 安全边界：域名约束必须是执行层规则

Browser Use 当前提供 `allowed_domains`、`prohibited_domains` 与 IP blocking。`SecurityWatchdog` 会在导航开始前检查目标 URL，并在导航完成后再次检查，以捕获重定向到禁止域名的情况；新标签页也会被检查。

这是一项值得保留的模式：**不能只靠 prompt 告诉 Agent“不要离开本站”**，必须在浏览器执行层强制限制。

不过平台实现应比通用库更严格：

- allowlist 必须直接来自已审批 Source Scope；
- 默认阻止 IP、localhost、内网地址和 cloud metadata 地址；
- 默认禁用下载、上传、剪贴板、通知、摄像头、麦克风；
- Cookie / Storage State 必须按 Source 与 fetch variant 隔离；
- secret 只通过引用注入，不写入 Agent trace；
- 对 `data:`、`blob:`、跨源 iframe、popup 等也要建立明确策略，而不是沿用通用运行库的宽松默认值；
- 所有重定向、tab 创建与跨 host 尝试都进入审计日志。

## 4. 为什么不能把 Browser Use 当作默认抓取器

### 4.1 吞吐与成本不匹配

1000+ 博客的历史回填可能产生百万到千万 URL。默认对每个 URL 执行“构造上下文 → 调模型 → 执行动作 → 再调模型”的闭环，会把本来可用单次 HTTP 请求完成的任务放大成多个 Browser + LLM round trip。

因此 Browser Use 必须只消费极少数“确定性路径失败”的任务，目标是降低人工写站点脚本的成本，而不是提高主链 URL/s。

### 4.2 非确定性不适合作为数据版本真相

即使 Agent 两次都“成功拿到正文”，它可能走不同路径、点击不同控件、进入不同 locale/实验分组，甚至因页面改变而选择另一条分支。对增量同步而言，这会让“这次正文为什么变化”难以归因。

所以：

- Agent 产物首先是 `diagnostic artifact`；
- 真正的 Raw Artifact 仍必须由受控 Fetch Attempt 产生并绑定 `fetch_variant_key`；
- Document Version 仍由 body hash + canonicalization/quality 规则推进；
- Agent 不允许直接更新 `last_verified_at`、Coverage Complete 或 Version 状态。

### 4.3 Agent 成功不代表历史覆盖完整

Agent 找到 200 篇文章并不证明站点只有 200 篇。它可能漏掉隐藏年份、tag archive、无入口孤儿页，或者在“加载更多”看似停止时其实遇到临时错误。

因此历史覆盖仍必须由 CMS/API、Sitemap、Feed、Archive、Common Crawl、Wayback 等 Provider 形成可解释 Inventory，并通过 Completion Attestation 证明各 Provider 的结束语义。Agentic Browser 只能增加 Observation 或提供“可能存在新入口”的提示。

## 5. 推荐集成：Agentic Browser Repair / Recipe Studio

### 5.1 触发条件

只有以下情况进入 Agentic Repair：

1. HTTP Fast Lane 失败，且普通 Browser Recipe 仍无法得到合格内容；
2. 页面需要多步骤交互才能发现归档文章 URL；
3. 某 Source 长期出现相同类型的 REPAIR/REVIEW，人工希望自动生成 Recipe 候选；
4. Operator 在 Web Admin 主动发起诊断；
5. 新 Source onboarding 时自动探索少量样本页。

禁止对全站所有 URL 默认开启。

### 5.2 Agentic Repair Run 数据模型

建议增加：

```text
agentic_repair_run
  run_id
  source_id
  url_id
  fetch_variant_key
  trigger_reason
  browser_use_release
  model_release
  browser_profile_release
  agentic_repair_policy_release
  allowed_domains
  max_steps
  max_actions
  time_budget_ms
  token_budget
  cost_budget
  started_at / finished_at
  outcome
  stop_reason

agentic_action_trace
  run_id
  step_no
  browser_state_hash
  dom_snapshot_artifact_id
  screenshot_artifact_id
  current_url
  action_name
  action_params_redacted
  target_evidence
  result
  destination_url
  duration_ms
  model_tokens
  estimated_cost
```

Trace 必须写不可变对象存储；secret、Cookie 值和敏感 header 只能保存脱敏引用。

### 5.3 Recipe Candidate，而不是直接保存 Agent Prompt

Agent 成功后，不应把“以后再让同一个 Agent 重走一次”当成生产方案，而应把探索结果转换成稳定、有限的 Recipe DSL。

建议 Recipe Candidate 示例：

```yaml
recipe_candidate:
  source_id: example
  purpose: archive_discovery
  entry: /blog/archive
  steps:
    - wait_selector: "main"
    - repeat:
        max_iterations: 40
        until:
          selector_absent: "button.load-more"
        actions:
          - click: "button.load-more"
          - wait_network_idle: 1500ms
    - extract_links:
        selector: "main article a[href]"
        scope: same_source
  assertions:
    - min_links: 50
    - no_cross_host_navigation: true
```

Recipe DSL 只允许平台支持的有限动作，默认不接受 Agent 生成任意 JavaScript。Selector 还应记录候选置信度与替代锚点，避免仅依赖易变的 CSS class。

## 6. 核心新增机制：Deterministic Recipe Promotion Gate

这是本次调研对现有方案最重要的补强。

如果只引入 Browser Use，却没有“从 Agent 探索到确定性生产 Recipe”的晋升门禁，就会把 LLM 非确定性带入主抓取链。正确流程应为：

```text
REPAIR / Operator Diagnose
        |
        v
Agentic Browser Sandbox
        |
        +--> immutable action trace / screenshot / DOM evidence
        |
        v
Recipe Candidate
        |
        v
Static Policy Validation
        |
        v
Deterministic Replay on fixtures + sampled live pages
        |
        +-- fail --> Candidate Rejected / needs human edit
        |
        v
Shadow Run
        |
        v
Source Canary
        |
        v
Approved Recipe Release
        |
        v
普通 Browser Slow Lane 使用确定性 Recipe 执行
```

### 6.1 静态策略校验

至少验证：

- 只允许已批准 host/path；
- 不包含任意 JS / shell / filesystem side effect；
- 没有上传、表单提交、删除、支付、发布动作；
- loop 必须有最大次数；
- wait 必须有 timeout；
- scroll 必须有上限；
- selector 必须有稳定性规则；
- 下载、popup、iframe 必须显式授权。

### 6.2 确定性 Replay

在固定 fixture 或一组真实样本页上，由**不调用 LLM**的 Recipe Executor 重放：

- 所有步骤是否可执行；
- 目标元素是否稳定命中；
- 发现的文章链接数量/分布是否合理；
- 是否发生超范围导航；
- 是否出现不可控副作用；
- 对相同 fixture 多次执行是否得到相同结果；
- 生成的 Raw Artifact / Candidate 是否通过现有质量门禁。

如果某步骤只能依赖“再次询问 LLM 下一步是什么”，则它不能自动成为 Production Recipe，只能保留在 Agentic Repair 层。

### 6.3 Shadow + Canary

Replay 通过后仍不能直接全量使用，应：

1. 与当前 Recipe 并行 shadow；
2. 对比 URL discovery、正文质量、错误率、浏览器时间、成本；
3. 先在 1% 或少量 Source URL 上 canary；
4. 无明显回归后发布新的 `recipe_release`。

整个晋升过程必须进入审计日志，可随时回滚到上一 release。

## 7. 对现有技术方案的具体优化点

### 7.1 新增原则

增加：

> Agentic Repair Is Proposal, Not Truth：LLM 浏览器 Agent 只能提出诊断结果、URL Observation 或 Recipe Candidate；未经确定性 Replay / Shadow / Canary 门禁，不得成为生产 Recipe，更不能推进 Coverage、Freshness、Document Version 或 canonical Markdown。

### 7.2 Browser Slow Lane 分成两层

原方案只有 Browser Slow Lane，可以进一步拆成：

```text
Browser Slow Lane
  |- Deterministic Browser Recipe Executor   # 生产默认
  `- Agentic Browser Repair Sandbox          # 少量疑难诊断/候选生成
```

这能同时保留复杂网页交互能力和主链可重放性。

### 7.3 增加 Recipe Studio Web 管理能力

管理后台新增页面：

- 对某个 Source/URL 发起 Agentic Repair；
- 实时查看 screenshot、DOM/AX 摘要、当前 URL、动作 trace；
- 查看 token/时间/费用预算；
- 自动生成 Recipe Candidate；
- 人工编辑 selector 与动作参数；
- 一键运行 deterministic replay；
- 查看 N 个样本的成功/失败矩阵；
- 发起 shadow / canary；
- 审批、拒绝、发布、回滚 Recipe Release。

这比让工程师维护 1000 份 Playwright 脚本更符合“新增站点低成本”的目标。

### 7.4 增加版本化对象

建议纳入 Release/Schema 管理：

```text
agentic_repair_policy_release
browser_interaction_dsl_release
recipe_candidate_schema_release
recipe_promotion_policy_release
```

模型版本也必须记录，但模型输出本身不属于确定性 Release 的执行依赖。

## 8. 故障模式与治理

### 8.1 Selector 漂移

Agent 可能点击临时 class 或第 N 个按钮。Recipe Candidate 应优先组合：语义角色、稳定属性、可见文本锚点、DOM 路径和 fallback selector，并在 promotion 时做多页面命中率测试。

### 8.2 无限滚动/加载更多不终止

所有 repeat / scroll 必须有 `max_iterations`、总时间预算、连续无新 URL 次数阈值。停止原因必须 typed，例如 `NO_NEW_LINKS`、`MAX_ITERATIONS`、`TIME_BUDGET_EXHAUSTED`。

### 8.3 重定向/广告导致越界

浏览器执行层必须在导航前和导航后都检查 approved scope，新 tab 同样检查；越界即停止该动作并留下 Evidence。

### 8.4 LLM 误操作

Agentic Sandbox 默认使用只读动作集合。涉及表单、上传、下载、Cookie/Auth、高风险 JS 的能力按 Capability Firewall 单独审批，不允许模型自行扩大权限。

### 8.5 页面 A/B、locale、登录态导致结果不一致

Agentic Run 与后续 Deterministic Replay 都必须绑定 `fetch_variant_key`、browser profile、locale、timezone、UA class、Cookie/Auth class、Proxy/region 等上下文。否则“Agent 成功、生产 Recipe 失败”很难诊断。

### 8.6 成本失控

Agentic Repair 必须有 Source 级与全局预算：

- 最大步骤；
- 最大 LLM token；
- 最大浏览器 wall time；
- 最大页面数；
- 最大单次费用；
- 每小时/每日 Repair Run 配额。

预算耗尽必须返回显式 stop reason，不能把“预算耗尽但已经找到一些链接”误报为完成。

## 9. 最终结论

Browser Use **不应替代** 当前方案中的 HTTP Fast Lane、Crawl4AI Runtime、Persistent Frontier、Provider Completion Attestation、Raw Artifact 和 Canonical IR。把它作为默认抓取器会降低确定性、吞吐、可审计性和历史覆盖可信度。

它真正值得吸收的能力是：

1. 基于 DOM/AX/截图等浏览器状态进行多步骤自适应交互；
2. 结构化 Tool Registry 与有限动作空间；
3. 浏览器事件层与 CDP 执行层分离；
4. 页面级动作过滤、循环检测、上下文压缩和预算控制；
5. 在少量复杂站点上自动探索“怎样才能拿到内容/链接”。

因此推荐把 Browser Use 作为 **可选的 Agentic Browser Repair / Recipe Studio Runtime**，其输出只是一份候选。平台通过 **Deterministic Recipe Promotion Gate** 将成功探索转化为不依赖 LLM 的确定性 Recipe，经过 fixture replay、live sample replay、shadow、canary 后才发布到 Browser Slow Lane。

这样既利用 Agent 处理长尾站点的适应性，又维持知识库主链“Coverage 可证明、抓取可重放、版本可解释、增量同步可信、成本可控”的核心架构边界。
