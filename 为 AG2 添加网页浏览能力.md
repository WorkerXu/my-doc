# 为 AG2 添加网页浏览能力：实现细节、技术原理与博客知识库方案启示

## 1. 调研对象与版本边界

- 原文：<https://docs.ag2.ai/latest/docs/blog/2025/01/31/Websurfing-Tools/>
- AG2：<https://github.com/ag2ai/ag2>
- Browser Use：<https://github.com/browser-use/browser-use>
- Crawl4AI：<https://github.com/unclecode/crawl4ai>
- 原文发布日期：2025-01-31
- 为避免把后续文档更新误认为 2025-01-31 当时实现，本次额外检查了 AG2 在 2025-02-01 前的仓库快照 `f9c8bc4c42cad859ca301f9bd0f42b26738fc648`，并结合该快照依赖的 Browser Use 0.1.x 实现分析。

当前在线原文已被维护者持续更新，例如页面现在写有 Crawl4AI `>=0.5.0`、测试过 0.8.x，并展示较新的模型与日志；而 2025-02-01 前 AG2 `pyproject.toml` 实际约束是：

```text
crawl4ai>=0.4.247,<0.5
browser-use>=0.1.27,<0.2
```

因此，本文把“原文表达的设计思想”和“文章发布时附近的真实实现”分开讨论。

---

## 2. 原文解决的问题

AG2 不把网页浏览硬编码进 Agent，而是把网页能力封装成 Tool：

```text
AssistantAgent / LLM
  -> 看见工具 schema
  -> 选择工具并生成参数

UserProxyAgent / Executor
  -> 真正执行 Tool
  -> 把结果返回给 Agent
```

原文展示两种性质完全不同的能力：

1. **Browser Use**：让 Agent 观察网页、决定下一动作、点击、输入、导航、抽取，适合未知 UI 和多步骤交互；
2. **Crawl4AI**：针对已知 URL 做抓取、Markdown 或结构化抽取，适合数据采集。

这不是“两个同类爬虫”的比较，而是：

```text
确定性抓取能力 + Agentic 交互浏览能力
```

对 1000+ 博客知识库而言，这一区分非常关键。高吞吐主链应尽量确定性；Agentic Browser 只能作为少量异常和探索能力。

---

## 3. AG2 Tool 注册机制：Planner 与 Executor 分离

原文的核心注册方式：

```python
browser_use_tool.register_for_execution(user_proxy)
browser_use_tool.register_for_llm(assistant)

crawlai_tool.register_for_execution(user_proxy)
crawlai_tool.register_for_llm(assistant)
```

两个注册动作职责不同：

```text
register_for_llm
  -> 把名称、描述、参数 schema 暴露给模型

register_for_execution
  -> 把真实 callable 注册给执行 Agent
```

因此 Tool 是“计划者”和“执行器”之间的能力契约。

这个思想可以直接吸收到知识库平台，但生产环境必须再加一层 Policy Gate：

```text
Planner
  -> FetchPlan
  -> Capability Check
  -> URL Scope / SSRF / Egress Check
  -> Budget / Rate / Action Policy
  -> Executor
```

不能因为 AG2 能让模型选择 Tool，就让 LLM 自由决定百万 URL 的抓取方式。日常 Backfill/Incremental 必须由规则路由；LLM 只能用于 Probe、Research 或异常提案。

---

## 4. Crawl4AITool 的真实实现

### 4.1 入口结构

文章发布时附近的 AG2 `Crawl4AITool` 直接包装 Crawl4AI：

```python
from crawl4ai import AsyncWebCrawler, BrowserConfig, CacheMode, CrawlerRunConfig
from crawl4ai.extraction_strategy import LLMExtractionStrategy
```

Tool 构造函数接受：

```text
llm_config
extraction_model: Optional[Pydantic BaseModel]
llm_strategy_kwargs
```

然后根据是否提供 `llm_config` 选择两个 callable：

```text
无 LLM -> crawl4ai_without_llm(url)
有 LLM -> crawl4ai_with_llm(url, instruction, dependencies...)
```

这说明 AG2 的抽象层非常薄：它没有自己实现抓取引擎，只负责把 Agent Tool 参数转换成 Crawl4AI 的运行配置。

### 4.2 无 LLM 路径

核心逻辑等价于：

```python
async with AsyncWebCrawler(config=browser_cfg) as crawler:
    result = await crawler.arun(url=url, config=crawl_config)

if crawl_config is None:
    return result.markdown
```

关键性质：

- 一个工具调用创建一个 `AsyncWebCrawler` 上下文；
- 调用 `arun` 抓一个 URL；
- 没有显式运行配置时直接返回 `result.markdown`；
- Tool 自身不保存 Snapshot、响应头、重定向链、渲染证据和版本信息。

这非常适合 Agent 工具演示，但不适合作为大规模知识库的最终事实层。生产平台必须在它外层增加 Artifact、版本、幂等和持久化边界。

### 4.3 LLM 抽取路径

有 `llm_config` 时，AG2 强制创建 headless Browser，并构造 `CrawlerRunConfig`：

```text
AG2 llm_config
 -> provider/api key
 -> optional Pydantic schema
 -> LLMExtractionStrategy
 -> CrawlerRunConfig
 -> AsyncWebCrawler.arun
 -> extracted_content
```

Pydantic 模型会通过 `model_json_schema()` 变成 JSON Schema。默认抽取类型为：

```text
有 schema -> schema
无 schema -> block
```

并且 AG2 禁止调用方在 `llm_strategy_kwargs` 中覆盖：

```text
provider
api_token
schema
instruction
```

这是一个值得保留的设计：**上层 Tool 固定安全和契约字段，调用方只能修改受控参数**。

### 4.4 CacheMode.BYPASS 的含义

当时 AG2 LLM 路径明确设置：

```python
CrawlerRunConfig(
    extraction_strategy=llm_strategy,
    cache_mode=CacheMode.BYPASS,
)
```

这适合交互式 Agent 想拿“本次最新结果”的场景，但不应该原样用于知识库增量同步：

- 增量需要 ETag/Last-Modified/304；
- 需要源站友好的 Conditional Fetch；
- 需要平台自己持久化 Snapshot 和内容 hash；
- 需要控制重复 Browser/LLM 成本。

因此平台的缓存/版本真相不能委托给 Tool 内部 cache mode。

### 4.5 生命周期问题

AG2 示例每次 Crawl4AI Tool 调用都进入新的 `AsyncWebCrawler` 上下文。这种生命周期简单、泄漏风险小，却会增加高频 Browser 初始化/回收成本。

1000 站生产方案应改为：

```text
Worker 级 Browser Process Pool
 -> Source/Security-Domain 级 Context
 -> Task 级 Page
```

同时保留强制 recycle 阈值，避免长生命周期 Browser 内存膨胀。

---

## 5. BrowserUseTool 的真实实现

### 5.1 AG2 封装层

文章附近的 `BrowserUseTool` 接收：

```text
llm_config
browser（可注入）
agent_kwargs
browser_config
```

如果没有注入 Browser，它会：

```text
browser_config.headless 默认 true
 -> BrowserConfig
 -> Browser
```

同时默认设置：

```text
generate_gif = false
```

这避免 Agent 每次执行都生成 GIF 的额外 I/O。

每次 Tool 调用时：

```text
AG2 llm_config
 -> 转换为 LangChain ChatOpenAI / ChatAnthropic / ChatGoogleGenerativeAI
 -> new browser_use.Agent(task, llm, browser, ...)
 -> await agent.run()
 -> BrowserUseResult(extracted_content, final_result)
```

AG2 返回的不是完整 Browser 状态，而只是 Browser Use 历史中的抽取内容和最终结果。

### 5.2 Browser 生命周期与 Context 隔离

AG2 Tool 可以复用同一个 Browser 实例。Browser Use 0.1.27 的 Agent 在收到“注入的 Browser”时：

- 不重新创建 Browser 进程；
- 为该 Agent 创建自己的 `BrowserContext`；
- Agent 结束时关闭自己创建的 Context；
- 因 Browser 是注入的，不会把共享 Browser 关闭。

这正好体现了适合生产的一个基础模式：

```text
Browser Process 复用
Context 隔离
Task/Agent 生命周期短
```

但知识库平台还需要进一步保证 Cookie、LocalStorage、认证态、下载目录、代理身份、允许域不会跨 Source 泄漏。

---

## 6. Browser Use 的 Agent 控制循环

Browser Use 0.1.27 `Agent.step()` 的核心链路是：

```text
browser_context.get_state(use_vision)
 -> 把当前 BrowserState 加入消息
 -> LLM structured output 生成下一组 Action
 -> controller.multi_act(actions, browser_context)
 -> 保存 ActionResult
 -> 保存 BrowserStateHistory
 -> 下一步
```

模型输出中包含类似：

```text
evaluation_previous_goal
memory
next_goal
action[]
```

原文日志中的“Eval / Memory / Next goal / Action”正来自这种循环。

`Agent.run()` 默认最多执行 100 步，并有：

- 连续失败上限；
- pause/stop；
- 每步结构化动作；
- 可选 output validation；
- history；
- 结束时清理 Agent 自建 Browser Context/Browser。

这说明 Browser Use 的本质不是“更强的抓取器”，而是：

```text
网页状态观测
 + LLM 决策
 + 浏览器动作执行器
 + 历史/记忆
 + 终止条件
```

它适合未知交互，却天然比确定性 crawler 更慢、更贵、更难复现。

---

## 7. Browser Use 对博客知识库的正确位置

### 7.1 适合

- Probe 陌生站点；
- 历史入口只有复杂 UI；
- 动态日期筛选、Load More、未知交互；
- 固定 Recipe 连续失败后的诊断；
- 临时 Research；
- 自动探索后生成 Recipe 草稿。

### 7.2 不适合

- 百万文章正文主抓取；
- 每日常规增量；
- Sitemap/RSS/CMS API 能解决的发现；
- 需要确定性、低成本、大吞吐的任务；
- 未设置安全策略的开放互联网浏览。

推荐路线：

```text
L0 Authoritative: CMS / Sitemap / Feed
L1 HTTP: 静态正文 / API / 文件
L2 Deterministic Browser: JS hydration
L3 Versioned Recipe: 固定 click/scroll/load-more
L4 Interactive Agent Browser: 未知 UI / 异常探索
```

成功的 L4 交互应尽量“编译”为 L3 Recipe，而不是永久依赖 Agent。

---

## 8. Crawl4AI 三种抽取模式的架构启示

原文把 Crawl4AI Tool 分为：

```text
Basic scraping, no LLM
LLM processing
LLM + Pydantic Schema
```

对知识库平台应映射成：

```text
Canonical 主链:
HTTP/Browser Artifact
 -> 本地确定性 Extractor
 -> Canonical IR
 -> Markdown

可选结构化链:
Artifact
 -> Rule/DOM
 -> Schema validation
 -> 低质量时才调用 LLM
 -> Schema validation
 -> Observation / Metadata Candidate
```

不能把 LLM 抽取结果直接当 Canonical Content。

Pydantic/JSON Schema 的真正价值是让“结果契约”与“抽取实现”解耦：以后可以从 Crawl4AI LLM 换成 DOM parser、其他模型或人工规则，而不改变下游数据接口。

---

## 9. 需要吸收到博客知识库方案的具体优化

### 9.1 增加 Tool/Capability Adapter 边界

业务层不依赖具体框架 DTO：

```text
CrawlerAdapter
  probe()
  discover()
  fetch()
  render()
  interact()
  extract_structured()
```

运行时记录 Capability Set 与版本。Crawl4AI、Playwright、Browser Use 都只是某个 Adapter 实现。

### 9.2 明确 Fetch Planner / Executor

```text
FetchPlan
  route
  capability_set
  allowed_hosts
  allowed_actions
  max_steps
  max_time
  max_bytes
  max_cost
  recipe_release
  decision_evidence
```

Planner 默认规则化；Executor 是唯一能联网的组件。LLM 只生成候选计划，必须经过平台 Policy Gate。

### 9.3 Browser 生命周期分层

从 AG2 + Browser Use 的 Browser 注入机制抽象成：

```text
Browser process = Worker 级复用
Context         = Source / Security Domain 隔离
Page            = Task 级
Session         = 明确交互任务级
```

并增加：RSS/page count/task count/uptime 阈值 recycle。

### 9.4 Interaction Trace 一等公民

AG2 Tool 最终只返回文本，不足以支撑知识库审计。生产平台需要保存：

```text
step_no
url_before
observation artifact
model/action-policy release
action + redacted args
url_after
result
screenshot/rendered DOM
policy decision
```

最终 Artifact 再进入普通 Extract/IR/Markdown 链。

### 9.5 Structured Extraction 与事实层分离

```text
Artifact
 -> Versioned Schema / Instruction
 -> Extractor
 -> Validation
 -> Candidate
 -> URL Scope / Normalize / Dedupe
 -> Observation
```

LLM 不得绕过这些步骤。

### 9.6 不采用 Tool 内部 Cache 作为增量真相

平台统一负责：

- ETag / Last-Modified；
- 304；
- raw hash；
- rendered hash；
- canonical content hash；
- immutable Snapshot；
- Version。

Tool/crawler 自带 cache 只是执行优化，不是业务真相。

### 9.7 Prompt Injection 与越域防护

Browser Use 的下一动作由页面状态影响，网页内容天然属于不可信输入。因此：

```text
Web content
 -> Agent proposes action
 -> Platform Policy Gate
 -> Allowed Host / SSRF / Egress / Action / Budget check
 -> execute or block
```

网页文字不能修改允许域、权限、Secret、预算或 Tool 集合。

---

## 10. 与 1000+ 站规模的成本与吞吐关系

若把所有页面都交给 Agent Browser，成本大致随：

```text
页面数 × 平均交互步骤 × LLM 调用成本 × Browser 时间
```

增长；而 HTTP/确定性 Browser 的主要成本是网络、CPU、内存和渲染槽位。

因此资源池必须分开：

```text
HTTP Pool           大
Deterministic Browser Pool  中
Interaction Pool    小
LLM Extraction Pool 小且有预算
```

优先级建议：

```text
Incremental > Backfill > Verify > LLM fallback > Interactive Research
```

Interactive Worker 不能挤占日常增量 SLA。

---

## 11. 失败模式

### Crawl4AI Tool 类

- 每调用新 crawler 带来初始化成本；
- Browser/Playwright crash；
- LLM extraction 超时或 Schema 失败；
- `CacheMode.BYPASS` 放大重复抓取成本；
- Tool 返回文本但缺少平台级网络证据；
- 模型/运行时升级使抽取发生漂移。

### Browser Use 类

- LLM 选错元素；
- 页面变化导致动作失效；
- 无限/过长探索；
- 被网页内容诱导越域；
- 多步骤交互不可确定复现；
- 长任务占用 Browser slot；
- 模型、System Prompt、Action Registry、Browser 版本任一变化都可能改变动作序列。

平台应以超时、步数、成本、导航次数、域名、动作白名单、失败次数和 circuit breaker 进行硬限制。

---

## 12. 测试与版本治理

需要固定：

```text
adapter release
crawler runtime release
browser version
browser-use/Crawl4AI version
planner release
recipe release
action policy release
system instruction release
schema release
extractor release
```

回归集至少包含：

- 静态文章；
- JS hydration；
- Load More；
- iframe/Shadow DOM；
- 复杂列表页；
- 重定向；
- Prompt Injection 页面；
- 越域链接；
- 429/403；
- Agent 多步骤成功/失败；
- Schema 抽取错字段/漏字段。

Runtime 升级应先 Golden Corpus/Golden Interaction canary，再激活。

---

## 13. 最终结论

AG2 这篇文章最有价值的不是“多了两个 Tool”，而是暴露了两种网页能力之间的结构差异：

```text
Crawl4AI Tool
 = 已知 URL -> 抓取/渲染/抽取
 = 相对确定、适合数据流水线

Browser Use Tool
 = 任务 -> 观察 -> LLM 决策 -> 动作 -> 再观察
 = 高灵活、高成本、低确定性、适合异常探索
```

进一步查看源码后还能得到四个生产级结论：

1. **Planner 与 Executor 必须分权**，AG2 Tool schema 可以借鉴，但生产需要额外 Policy Gate；
2. **Browser 进程复用、Context 隔离、Task 短生命周期**是兼顾吞吐与隔离的合理方式；
3. **Crawl4AI/Browser Use 的内部结果不能成为平台事实层**，必须落到 Snapshot、Render Artifact、Interaction Trace、IR、Version；
4. **LLM/Agent 能力应该被编译成稳定配置**：结构化抽取固化成 Schema/规则，成功交互固化成 Recipe，日常运行尽可能不再依赖 LLM。

因此，博客知识库的主链应保持：

```text
Coverage-first Discovery
 -> HTTP First
 -> Deterministic Browser
 -> Versioned Recipe
 -> exceptional Agent Browser
 -> immutable Artifact
 -> deterministic Extract / IR / Markdown
 -> incremental Versioning
```

这既保留 Agent Browser 对复杂网页的适应力，也不会牺牲 1000+ 站长期抓取最关键的确定性、成本、吞吐、可恢复性、安全性和可审计性。
