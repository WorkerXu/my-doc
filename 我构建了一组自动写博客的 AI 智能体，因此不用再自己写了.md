# 我构建了一组自动写博客的 AI 智能体，因此不用再自己写了

## 1. 调研对象与结论

- 原文：I Built a Crew of AI Agents That Write Blogs So I Don’t Have To (Here’s How You Can Too)
- 地址：https://dev.to/kuldeep_paul/i-built-a-crew-of-ai-agents-that-write-blogs-so-i-dont-have-to-heres-how-you-can-too-2i84
- 代码仓库：https://github.com/dskuldeep/blog-writing-agent.git
- 主要技术：CrewAI、Tavily Search、Crawl4AI、Gemini、Maxim AI、Python asyncio

原项目的直接目标是“让一组 AI Agent 自动研究、写作、审查、修订并导出博客”。它不是面向百万级第三方文档采集设计的生产爬虫，因此不能原样套到“1000+ 技术博客全历史抓取 + 增量同步 + Markdown 知识库 + Web 管理”场景。

真正值得吸收的是以下工程思想：

1. **复杂任务分阶段**：Research、Outline、Writer、Critique、Revision、Export 职责分离；
2. **工具通过统一接口暴露给编排层**：搜索、抓取、保存都封装为 Tool；
3. **候选结果与质量审查分离**：先生成，再 Critique，再 Revision；
4. **全过程可观测**：对 LLM、Tool Call、Workflow Event 建立 Trace；
5. **多模型/多角色可以按职责路由**：编排层与执行层不必绑定单一模型。

但从代码细节看，Demo 也暴露出一些生产化风险：每 URL 创建 Crawl4AI crawler、一次性 `asyncio.gather`、notebook 中使用 `nest_asyncio`、工具失败以普通字符串返回、自由文本在 Agent 间传递、Agent `memory` 可能形成隐式状态、FileSaver 接受任意本地路径、Trace 可能直接记录较大工具输入输出、Notebook 依赖交互式运行环境。这些问题正好说明知识库平台需要 **确定性数据面 + 受治理控制面 Agent**，而不是把多 Agent 直接放进热路径。

---

## 2. 原项目的执行拓扑

文章定义了 6 个 Agent：

1. **Research Agent**：使用 Tavily Search 和 Crawl4AI 搜索并读取资料；
2. **Outline Agent**：根据研究材料规划文章结构；
3. **Writer Agent**：生成 Markdown 初稿；
4. **Critique Agent**：检查准确性、可读性和完整性；
5. **Revision Agent**：根据 Critique 结果修改；
6. **Export Agent**：通过 FileSaverTool 保存最终文件。

任务拓扑本质上是顺序流水线：

```text
Topic
 -> Research
 -> Outline
 -> Write
 -> Critique
 -> Revise
 -> Export
```

CrewAI 负责 Agent/Task 编排；Gemini 在示例中同时承担主要 LLM、manager/planning/function-calling 等角色；Tavily 负责互联网搜索；Crawl4AI 负责网页抓取；Maxim AI 负责 Trace/Event/Tool Call 观测。

这个结构最有价值的部分不是“必须有 6 个 Agent”，而是 **阶段之间存在清晰的职责边界**。对于知识库平台，可映射为：

```text
Discovery
 -> Fetch
 -> Extract Candidate
 -> Quality Critique
 -> Bounded Repair
 -> Canonicalize
 -> Version
 -> Render/Export
```

这使重试、回放、成本统计、质量诊断和发布控制都能按阶段进行。

---

## 3. Tavily Search Tool：Adapter 与结构化输入

Notebook 自定义 `TavilySearchTool`，通过 Pydantic 输入模型约束：

- `query`；
- `max_results`；
- `search_depth`。

工具内部调用 Tavily，再将供应商返回结果收敛为 title、content、url 等字段。

### 3.1 技术原理

这是典型的 **Adapter Pattern + Tool Contract**：

```text
Agent
 -> Tool Schema
 -> Provider Adapter
 -> External API
 -> Normalized Result
```

业务编排不应该直接依赖 Tavily 的原始 JSON。对知识库平台同样如此：CMS API、Sitemap、RSS、Crawl4AI、Playwright、搜索引擎、互联网归档都应该先适配成平台自己的稳定协议。

推荐统一返回：

```json
{
  "status": "OK | RETRYABLE_ERROR | PERMANENT_ERROR | DEGRADED",
  "evidence_refs": ["..."],
  "output_ref": "object://...",
  "metrics": {"latency_ms": 123, "bytes": 456},
  "error": {"code": null, "retryable": false}
}
```

而不是让 Worker 或 Agent 直接解释供应商自由格式。

### 3.2 对历史抓取的适用边界

对于已经明确的 1000 个目标站点，通用搜索引擎不应成为历史 URL Discovery 主路径。推荐顺序：

```text
CMS/API
 -> Sitemap/Sitemap Index
 -> RSS/Atom
 -> Archive/Category/Tag/Pagination
 -> Bounded Deep Crawl
 -> Search/Common Crawl/Web Archive 作为补洞
```

搜索更适合：

- 新 Source Probe；
- Coverage Gap 补洞；
- 站点迁移/域名变化诊断；
- 控制面 Agent 搜索公开文档帮助生成 Candidate Recipe。

---

## 4. Crawl4AI Tool：异步 I/O 的实现与问题

Notebook 的 `WebCrawlerTool` 使用 `AsyncWebCrawler`。`_run` 中先为 URL 列表构造协程，再通过 `asyncio.gather` 等待所有抓取结束；单 URL 返回 Markdown、图片、URL 和 title。

### 4.1 为什么 asyncio 有效

网页抓取主要等待：

- DNS；
- TCP/TLS；
- 服务端响应；
- 下载；
- Browser 网络请求。

这些属于 I/O-bound，异步事件循环可以在一个线程里调度大量等待中的任务，避免“一请求一线程”的成本。

### 4.2 Demo 代码的生产风险：每 URL 一个 crawler

代码在 `crawl_url` 内部执行：

```text
async with AsyncWebCrawler(...) as crawler:
    await crawler.arun(...)
```

而 `process_urls` 又会并发调用多个 `crawl_url`。因此一个 batch 中可能同时创建多个 crawler/browser 生命周期。对于几十个 URL 的 Demo 可接受，但在大规模系统中会带来：

- Chromium/Browser 初始化开销；
- 内存与文件描述符快速放大；
- Browser context/session 无法复用；
- Cookie、DNS、连接池收益丢失；
- 单 Worker 资源峰值不可控。

生产设计应改为：

```text
Browser Worker Process
 -> Worker-scoped crawler/browser pool
 -> N 个受控 Browser Context
 -> bounded task queue
 -> per-host/global semaphore
```

即 **复用昂贵运行时，限制并发上下文**，而不是每个 URL 自建完整 crawler。

### 4.3 不应对海量 URL 一次性 gather

一次性对数万/数十万 URL 构造 `asyncio.gather` 会造成：

- 大量协程对象和参数同时驻留内存；
- 没有自然 backpressure；
- 失败批次尾延迟严重；
- 无法实现跨 Source 公平；
- 不方便 checkpoint、lease、取消和精细重试。

生产系统应使用：

```text
Scheduler
 -> Queue
 -> bounded prefetch
 -> Worker semaphore
 -> per-domain token bucket
 -> result/outbox
```

并让 Backfill、Incremental、Browser、Repair 使用不同 Queue Class。

---

## 5. `nest_asyncio` 与 Notebook 事件循环

Notebook 为了在已经运行事件循环的交互环境中调用异步代码，使用了 `nest_asyncio.apply()`。

这属于 Notebook 兼容技巧，不适合作为服务端 Worker 的运行模型。生产服务应让异步 Worker 自己拥有事件循环，并保持同步/异步边界清晰：

```text
Queue Consumer
 -> async fetch loop
 -> timeout/cancel
 -> lease renewal
 -> result commit
```

原因：

- 嵌套事件循环行为更难推理；
- timeout/cancel 传播容易复杂化；
- Agent 的同步调用不应决定爬虫 Worker 生命周期；
- Browser 与 HTTP 资源模型不同，应隔离部署。

因此 Crawl4AI 在知识库方案中应定位为 **Browser/Fetch Adapter**，不能让 CrewAI Agent 成为大规模抓取运行时。

---

## 6. Notebook 的可复现性问题

仓库只有一个已执行过的 Notebook。代码中可见对 `Markdown.markdown(...)` 的调用，但在公开 Notebook 的导入区没有看到对应的 `Markdown` 导入。这类情况在交互式 Notebook 中很常见：当前 Kernel 可能保留了之前执行过、后来被删除的变量或导入。

这说明生产系统不能接受“在我的 Notebook 里能跑”作为发布标准。推荐：

1. 所有依赖锁版本；
2. Runtime Image 使用 digest；
3. CI 在空环境/冷启动中运行集成测试；
4. Golden Corpus Replay 必须从全新 Worker 启动；
5. Source Release 同时记录 crawler、browser、extractor、renderer、schema 和 model release；
6. 禁止依赖 Kernel 隐式状态。

这对长期知识库尤其重要，因为半年后需要能够复现“为什么当时生成了这个 Markdown”。

---

## 7. 工具错误不能作为普通文本返回

Tavily Tool 的示例在异常时返回类似“An error occurred...”的字符串；Crawler Tool 也将失败放进普通字典/文本字段。

对 Agent Demo 来说方便，但生产系统存在问题：LLM 可能把错误字符串当成正常研究材料继续推理。

推荐使用 **Typed Failure Contract**：

```text
OK
RETRYABLE_NETWORK_ERROR
RATE_LIMITED
ACCESS_DENIED
TIMEOUT
INVALID_INPUT
PERMANENT_NOT_FOUND
POLICY_BLOCKED
INTERNAL_ERROR
```

每个错误带：

- retryable；
- retry_after；
- source/provider；
- trace_id；
- diagnostic_ref；
- user_visible_message。

Workflow Engine 根据状态决定 Retry/DLQ/Quarantine，而不是让 LLM 从字符串猜失败语义。

---

## 8. Markdown 与图片处理：不能把派生格式当事实

原项目直接使用 Crawl4AI Markdown，并把 Markdown 再转成 HTML 后用 BeautifulSoup 找图片。

对于内容生成 Demo 可以接受，但对可追溯知识库不理想。

### 8.1 Markdown 不是原始事实

Crawler 版本、抽取策略、过滤规则变化，都可能让同一 HTML 生成不同 Markdown。如果直接把 Markdown 当事实，就无法判断变化到底来自源站还是 Renderer。

推荐主链：

```text
Immutable Raw Artifact
 -> Extraction Candidate
 -> Quality Gate
 -> Canonical IR
 -> Document Version
 -> Markdown Renderer
```

Crawl4AI Markdown 只作为 Candidate/诊断产物。

### 8.2 图片不应走 Markdown -> HTML 往返

图片、链接、媒体应直接来自：

- 原始 DOM；
- Crawl4AI media metadata；
- HTML attribute；
- Structured Metadata。

处理：

- relative -> absolute URL；
- canonical asset URL；
- 保存 alt/title；
- 可选下载对象存储；
- sha256 去重；
- 记录 source artifact。

这样避免派生格式往返造成信息损失。

---

## 9. Critique -> Revision：最值得迁移的核心思想

原项目不是 Writer 输出后直接保存，而是经过 Critique，再交给 Revision。这对应生产知识库里的：

```text
Extract Candidate
 -> Quality Gate
 -> PASS
    or
 -> FAIL_RECOVERABLE
 -> Bounded Repair
 -> Quality Gate
```

推荐状态机：

```text
FETCHED
 -> EXTRACTED
 -> QUALITY_CHECK
      -> PASS/WARN -> CANONICALIZE
      -> FAIL_RECOVERABLE -> REPAIR_PENDING
           -> ALTERNATE_EXTRACTOR
           -> DETERMINISTIC_RECIPE
           -> DOM_REPAIR
           -> BROWSER_REFETCH（只有历史收益证据时）
           -> QUALITY_CHECK
      -> FAIL_UNKNOWN -> QUARANTINE
      -> NON_CONTENT -> REJECT
```

### 9.1 Quality Gate 指标

至少包括：

- title/author/date/canonical；
- 最小正文长度；
- paragraph/heading 数；
- link density；
- boilerplate；
- code/table/list 保留；
- error/login/WAF/CAPTCHA 指纹；
- Candidate Agreement；
- 与历史版本的异常截断/结构突变；
- Page Type 与正文特征是否冲突。

### 9.2 Revision 必须有限

不能把原文的“Agent 继续修订直到满意”照搬到爬虫热路径。Repair 必须是有限状态机，并配置：

- `max_steps`；
- `max_wall_time`；
- `browser_budget`；
- `max_cost`；
- Source circuit breaker。

未恢复到质量阈值就进入 QUARANTINE，不得让 LLM 改写第三方正文后冒充源文。

---

## 10. Agent Memory：不应成为业务状态

示例部分 Agent 开启 `memory=True`。对于研究/写作 Demo，Memory 可以让 Agent 记住前文；但生产控制面若把关键事实留在 Agent 隐式记忆中，会产生：

- 无法审计；
- 无法确定性重放；
- 升级模型后结果难比较；
- Context Window 截断可能静默丢信息；
- 多实例之间状态难共享。

知识库平台应采用 **Typed Artifact Handoff**：

```text
Agent Step A
 -> typed output JSON / object storage
 -> schema validation
 -> Agent Step B consumes refs
```

Agent memory 只允许作为临时推理上下文，不是 Source/Profile/Release/Coverage/Artifact 的真相存储。

推荐控制面对象：

```text
agent_run
agent_step
agent_tool_call
agent_evidence
candidate_release
```

其中输入输出尽量保存对象引用和 schema，而不是把所有网页原文塞进上下文。

---

## 11. Context Window 与研究材料外置

Research Task 的 expected output 倾向于保留较多“raw content”。如果将多个完整网页直接串进 Agent 上下文，会出现：

- Token 成本快速增加；
- 早期材料被截断；
- 重复页面挤占上下文；
- 难以追踪具体结论来自哪个页面；
- Prompt Injection 面扩大。

生产控制面应采用 **Evidence Store + Reference Passing**：

```text
Fetch Sample
 -> immutable evidence artifact
 -> deterministic metadata/summary
 -> evidence_id
 -> Agent receives bounded excerpts + evidence refs
```

Agent 输出的每条建议带 `evidence_refs`。真正配置发布时，Validator/Replay 直接读取证据 Artifact，而不是依赖 Agent 对原文的记忆。

---

## 12. Maxim Trace：应升级为平台级 Provenance

原项目用 Maxim 记录 LLM/Tool/Event Trace。这个思想非常适合知识库平台，但应扩展成统一 OpenTelemetry Trace。

推荐层级：

```text
sync_run_id
 -> source_id
 -> provider_run
 -> url_observation
 -> classification
 -> route
 -> fetch
 -> artifact
 -> extraction_attempt
 -> quality_result
 -> repair
 -> document_version
 -> markdown_projection
```

控制面 Agent 额外记录：

```text
agent_run_id
 -> agent_step_id
 -> model_call
 -> tool_call
 -> evidence_ref
 -> candidate_release
```

每个 span 至少带：

- source_id/source_release_id；
- run_id/task_id/url_id；
- artifact_id；
- model_release；
- tool_schema_release；
- token/cost；
- latency；
- status/error code；
- trace_id。

### 12.1 Trace 不能无限记录原文

Demo 里 Tool Trace 很容易把 query、output 等直接作为参数写入日志。生产环境应：

- Secret redaction；
- Cookie/Auth header 禁止进入 Trace；
- 大响应只存 object key + hash + size；
- Prompt/Response 设置大小上限；
- 对长期日志配置采样与保留周期；
- PII/版权内容按来源策略处理。

Observability 不能反过来成为第二套未经治理的正文存储。

---

## 13. FileSaverTool 暴露任意路径：生产环境必须去能力化

示例 FileSaverTool 接收 `save_folder` 和 `file_name` 并写本地文件。这适合个人 Notebook，但如果由 Agent 决定路径，在服务端属于高风险能力：

- path traversal；
- 覆盖非目标文件；
- 写入临时容器不可持久化；
- 多 Worker 并发冲突；
- 无版本、无 hash、无审计。

知识库平台不应给 Agent 任意文件系统写权限。正确做法是能力受限的 Adapter：

```text
SaveCandidateRelease(source_id, object_ref)
SaveDiagnostic(run_id, object_ref)
```

底层固定写入对象存储命名空间，并由 API 层校验 source/run/tenant 权限。Canonical Markdown 的写入更应由确定性 Renderer/Projection Worker 完成，而不是由 LLM Tool 保存。

---

## 14. 多模型能力：应转化为 Model Routing 与 Release

文章提到 CrewAI 可以为 chat、manager、planning、function calling 使用不同模型，示例为了简单全部使用同一个 Gemini 模型。

生产控制面可以利用角色差异，但必须版本化：

```text
agent_release
- workflow_version
- prompt_release
- tool_schema_release
- planner_model_release
- critic_model_release
- function_call_model_release
- max_steps
- max_tool_calls
- max_tokens
- max_cost
```

建议：

- 低成本模型：简单归纳、标签、样本初筛；
- 强模型：少量复杂 Probe/Quarantine 根因分析；
- Critic 可以与 Planner 使用不同模型，减少同源偏差；
- 任何 Agent 输出只是 Candidate，不直接改 ACTIVE Source Release。

必须能比较模型/Prompt 升级前后的 Golden Replay 与 Candidate Acceptance Rate。

---

## 15. 控制面 Agent 应采用“研究—计划—批评—修订—发布候选”工作流

原项目的多 Agent 思路适合放在低频控制面，建议映射为：

```text
Evidence Collector
 -> Profile Planner
 -> Rule/Recipe Generator
 -> Critic
 -> Reviser
 -> Candidate Release Assembler
```

### 15.1 Evidence Collector

只调用受限工具：

- `ProbeRobots`；
- `ProbeSitemap`；
- `ProbeFeed`；
- `FetchSample`；
- `InspectDOM`；
- `InspectStructuredMetadata`；
- `SearchGapEvidence`。

输出证据 ID，不保存业务真相。

### 15.2 Profile Planner

根据证据提出：

- Discovery Provider；
- Page Type pattern；
- HTTP/Browser route；
- Extractor portfolio；
- Quality threshold；
- Repair sequence。

### 15.3 Critic

检查：

- 是否过拟合几个样本；
- 是否遗漏 Index/Tag/Author；
- 是否引入 Browser 过多；
- selector 是否脆弱；
- Coverage 是否有证据；
- 是否违反 robots/egress policy；
- 是否可能将非正文写入 Canonical。

### 15.4 Reviser

只修改 Candidate 配置，不修改生产 ACTIVE。

### 15.5 Candidate Release Assembler

生成版本化 Source Release，进入：

```text
Static Validation
 -> Golden Replay
 -> Live Sample
 -> CANARY
 -> Metrics Compare
 -> Human Approval
 -> ACTIVE
```

这保留了原项目 Critique/Revision 的优点，同时把 LLM 不确定性限制在可审计、可回滚的控制面。

---

## 16. Control-plane Agent 也必须有预算和有限状态

虽然 Agent 不在热路径，仍需要：

- `max_steps`；
- `max_tool_calls`；
- `max_wall_time`；
- `max_tokens`；
- `max_cost`；
- per-provider RPM；
- Tool allowlist；
- source-scoped egress；
- circuit breaker。

达到预算后状态应是 `DEGRADED_NEEDS_REVIEW`，不能让 Agent 无限“反思—搜索—重试”。

任务失败也不应阻塞 HTTP/Incremental 数据面。

---

## 17. 从 Demo 到 1000 站平台的完整映射

| 原项目 | 知识库平台 | 生产约束 |
|---|---|---|
| Tavily Search | Gap Search / Probe Adapter | 不是主 Discovery |
| Crawl4AI Tool | Browser Fetch / Dynamic Discovery Adapter | 独立池、受限并发、可替换 |
| Agent Memory | Typed Agent Artifact | 不作为业务真相 |
| Research Agent | Evidence Collector | 低频控制面 |
| Outline Agent | Profile Planner | 只生成 Candidate |
| Writer Agent | Rule/Recipe Generator | 不改写正文 |
| Critique Agent | Agent Critic + Deterministic Validators | Critic 不能最终批准 |
| Revision Agent | Candidate Config Reviser | 有限次数 |
| Export Agent | Candidate Release Assembler | 不允许任意文件写 |
| Maxim Trace | OpenTelemetry + Agent Trace | 大内容外置、Secret redaction |
| Crawl4AI Markdown | Extraction Candidate | 不作为 Canonical |
| Blog export | Canonical IR -> Renderer -> Markdown Projection | 确定性、可重建 |

---

## 18. 对现有技术方案的具体优化项

基于原文与 Notebook 的实现细节，主方案应补充以下能力：

1. **Browser Worker 复用 crawler/browser/context 池**，禁止每 URL 无界创建昂贵运行时；
2. **所有并发必须 bounded**，不对全量 URL 一次性 gather；
3. **Agent Tool 使用 Typed Result/Typed Error**，错误不能作为普通研究文本；
4. **Agent 只传递版本化 Artifact/Evidence 引用**，Memory 不作为业务状态；
5. **控制面采用 Evidence -> Plan -> Critique -> Revise -> Candidate Release**；
6. **Agent Workflow 加 max_steps/tool_calls/tokens/cost/wall-time 预算**；
7. **Agent/Prompt/Tool Schema/Model 全部 Release 化**；
8. **Trace 大内容外置，日志做 Secret/PII redaction 和大小限制**；
9. **禁止 Agent 任意文件系统写入、任意 JS/Shell/Code Execution**；
10. **Notebook/运行时必须通过冷启动可复现测试**；
11. **Canonical Markdown 继续坚持 Raw Artifact -> Candidate -> Quality -> IR -> Renderer**；
12. **Critique/Revision 只用于配置候选和有限 Repair，不能改写第三方源正文。**

---

## 19. 最终判断

这篇文章最重要的启发不是“用 CrewAI 搭 6 个 Agent”，而是：**将复杂流程拆成角色清晰、可观察、可审查、可修订的阶段**。

对于 1000+ 技术博客知识库，正确吸收方式是：

```text
高吞吐数据面：
Authoritative Discovery
 -> Classification/Route
 -> HTTP First
 -> Immutable Artifact
 -> Extraction Portfolio
 -> Deterministic Quality
 -> Bounded Repair
 -> Canonical IR
 -> Version
 -> Markdown

低频控制面：
Evidence Collection
 -> Agent Planning
 -> Agent Critique
 -> Candidate Revision
 -> Static Validation
 -> Golden Replay
 -> Canary
 -> Human Approval
 -> Active Source Release
```

这样既继承多 Agent 的阶段化、Critique/Revision 和 Observability 优点，又避免 Demo 中的 Browser 生命周期、无界并发、隐式 Memory、自由文本错误、任意文件写、Notebook 状态以及 LLM 不确定性在百万级数据面放大。