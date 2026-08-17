# 我构建了一组自动写博客的 AI 智能体，因此不用再自己写了

## 1. 调研对象与结论

- 原文：I Built a Crew of AI Agents That Write Blogs So I Don’t Have To (Here’s How You Can Too)
- 原文地址：https://dev.to/kuldeep_paul/i-built-a-crew-of-ai-agents-that-write-blogs-so-i-dont-have-to-heres-how-you-can-too-2i84
- 示例代码：https://github.com/dskuldeep/blog-writing-agent
- 主要技术：CrewAI、Tavily、Crawl4AI、Gemini、Maxim AI、Python asyncio

这个项目的目标是自动完成“研究资料 -> 设计文章结构 -> 写作 -> 批评 -> 修订 -> 导出”的内容生产流程。它是一个典型的多 Agent 工作流 Demo，不是面向 1000+ 第三方技术博客、百万级文档、长期全量回填和增量同步设计的生产采集平台。

因此，项目不能原样套入博客知识库数据面，但有四类思想值得吸收：

1. **阶段化职责分离**：Research、Outline、Writer、Critique、Revision、Export 各自承担不同职责；
2. **Tool Adapter**：搜索、抓取、文件保存等能力通过 Tool 接口暴露给 Agent；
3. **Critique -> Revision**：候选结果不是直接发布，而是经过独立批评和修订；
4. **Observability**：模型调用、Tool 调用和 Workflow Event 可观测。

进一步检查 Notebook 实现后可以看到，它同时暴露了生产化时必须解决的问题：每 URL 新建 Crawl4AI crawler、URL 列表直接 `asyncio.gather`、Notebook 依赖 `nest_asyncio`、Tool 错误混入普通字符串/字典、Agent `memory=True` 形成隐式状态、自由文本阶段交接、FileSaver 可写任意路径、Tool 自行创建独立 Trace、Notebook 冷启动依赖不完整等。

对于博客知识库，最合适的吸收方式是：

```text
高吞吐数据面：确定性 Pipeline
低频控制面：确定性 Workflow Engine + 有界 Agent/Crew Activity
```

---

## 2. 原项目执行拓扑

Notebook 定义的核心 Agent/Task 可以概括为：

```text
Topic
 -> Research Agent
 -> Outline Agent
 -> Writer Agent
 -> Critique Agent
 -> Revision Agent
 -> Markdown / HTML Export Agent
```

Research Agent 使用 Tavily Search 查资料，再调用 Crawl4AI 抓取搜索结果页面；Outline Agent 基于研究材料生成文章结构；Writer 生成 Markdown；Critique 检查技术准确性、完整性和可读性；Revision 根据批评修改；最终生成 Markdown/HTML 并使用 FileSaverTool 写文件。

CrewAI 负责 Agent/Task 编排。示例中同一个 Gemini LLM 同时被作为 chat、manager、planning、function-calling LLM 使用。Maxim AI 被用于记录部分 LLM/Tool Trace。

这个结构的真正价值不在“必须有 6～8 个 Agent”，而在于：**任务被拆成可单独观察和评价的阶段**。

知识库平台可映射为：

```text
Discovery
 -> Fetch
 -> Extract Candidate
 -> Quality Critique
 -> Bounded Repair
 -> Canonicalize
 -> Version
 -> Render / Export
```

但数据面 Critique/Revision 必须是确定性的质量规则和有限修复，不能让 LLM 修改第三方原文后冒充源内容。

---

## 3. Tavily Search Tool：Adapter 与结构化输入

Notebook 自定义 `TavilySearchTool`，使用 Pydantic 定义输入，包括 query、max_results、search_depth，再调用 Tavily API，将供应商结果压成 title/content/url。

### 3.1 技术原理

这是典型的 Adapter Pattern：

```text
Agent
 -> Tool Schema
 -> Provider Adapter
 -> External API
 -> Normalized Result
```

知识库平台也应该把 CMS API、Sitemap、RSS、Crawl4AI、Playwright、搜索、Web Archive 等都适配成平台自己的稳定协议，而不是让业务逻辑理解每个供应商原始 JSON。

推荐统一结果：

```json
{
  "status": "OK | RETRYABLE_ERROR | PERMANENT_ERROR | DEGRADED",
  "evidence_refs": ["..."],
  "output_ref": "object://...",
  "metrics": {"latency_ms": 123},
  "error": {"code": null, "retryable": false},
  "trace_id": "..."
}
```

### 3.2 Demo 中的问题

示例 Tavily Tool 异常时直接返回类似 `An error occurred...` 的字符串。这会导致 Agent 可能把错误消息当作研究内容继续推理。

另外，工具输入虽然允许 `max_results`，但结果处理逻辑实际又取前 5 条，说明 Tool Contract 与实现语义可能发生漂移。生产平台必须对 Tool Schema 与行为做契约测试，而不只是“有 Pydantic 类型”。

### 3.3 在知识库里的定位

针对已经明确的 1000 个目标站点，通用搜索不应该成为历史 URL Discovery 主路径。推荐：

```text
CMS/API
 -> Sitemap/Sitemap Index
 -> RSS/Atom
 -> Archive/Category/Tag/Pagination
 -> Bounded Deep Crawl
 -> Search/Common Crawl/Web Archive 作为补洞
```

搜索适合 Source Probe、Coverage Gap、域名迁移诊断和控制面公开资料研究。

---

## 4. Crawl4AI Tool：异步 I/O 与 Browser 生命周期

Notebook 的 `WebCrawlerTool` 对每个 URL 执行：

```text
async with AsyncWebCrawler(...) as crawler:
    await crawler.arun(...)
```

外层又为全部 URL 构造任务并 `asyncio.gather`。

### 4.1 为什么 asyncio 有效

网页抓取主要耗时在 DNS、TCP/TLS、服务端响应、下载和 Browser 网络请求，是 I/O-bound。事件循环能在等待 I/O 时运行其他任务，减少一请求一线程的开销。

### 4.2 每 URL 一个 crawler 的生产风险

当 `crawl_url()` 自己创建 AsyncWebCrawler，而 batch 又并发执行多个 `crawl_url()` 时，会同时创建多个 Browser/Crawler 生命周期。规模放大后会造成：

- Chromium 初始化开销；
- 内存与文件描述符快速增长；
- DNS/连接池/cache/context 无法充分复用；
- Browser Slot 难以精确计量；
- 单 Worker 峰值资源不可控。

生产应采用：

```text
Browser Worker Process
 -> long-lived Browser/Crawler
 -> bounded Context Pool
 -> per-context Page Lifecycle
 -> recycle by request count / memory / time
```

### 4.3 无界 gather 的问题

对成千上万 URL 一次性创建协程会带来：

- 大量任务对象驻留内存；
- 无自然 backpressure；
- 一个大 Source 抢占全部并发；
- checkpoint、lease、精细 Retry、取消困难；
- batch 尾延迟放大。

正确方式：

```text
Scheduler
 -> Persistent Queue
 -> bounded prefetch
 -> Worker semaphore
 -> per-domain token bucket
 -> Task Result / Outbox
```

Backfill、Incremental、Browser、Repair 必须独立 Queue Class，支持公平调度。

---

## 5. `nest_asyncio`：Notebook 兼容技巧不是服务端架构

Notebook 使用 `nest_asyncio.apply()`，是为了在 Jupyter 已有事件循环中再次运行异步逻辑。

它适合交互式实验，不适合作为生产 Worker 的运行模型。服务端应该让 Worker 自己拥有事件循环，并保持同步/异步边界清晰：

```text
Queue Consumer
 -> async fetch loop
 -> timeout/cancel
 -> lease renewal
 -> commit result
```

这样 timeout、cancel、资源生命周期、故障恢复才可预测。

Crawl4AI 在知识库平台中应该是 Browser/Fetch Adapter，而不是由同步 Agent Tool 控制 Browser Worker 生命周期。

---

## 6. Notebook 冷启动可复现性

仓库主体只有一个已经执行过的 Notebook。公开代码中可看到 `Markdown.markdown(result.markdown)`，但导入区未看到对应 `Markdown` 导入；执行日志还出现过模型/Pydantic 转换相关的 `No module named 'google.genai'` 错误后流程继续运行。

这说明“Notebook 当前 Kernel 能跑”不等于可发布。

生产要求：

1. 锁定 Python/依赖版本；
2. Runtime Image 使用 digest；
3. Chromium/Browser 版本明确；
4. Fresh Container 执行集成测试；
5. Golden Corpus Replay 从全新 Worker 启动；
6. Crawler、Extractor、Renderer、Prompt、Model、Tool Schema、Workflow Contract 都进入 Release；
7. 禁止依赖 Notebook Kernel、上一次执行残留变量或隐式 import。

长期知识库需要半年后仍能解释和重建“为什么当时生成了这个 Markdown”。

---

## 7. Tool Failure 必须是控制流状态，而不是内容

Demo 中 Tavily 异常返回普通字符串，Crawler 异常返回普通字典里的 error 文本。Agent 很可能继续把这些信息塞入下一阶段上下文。

生产平台至少区分：

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

每个错误携带 retryable、retry_after、provider/source、trace_id、diagnostic_ref。

Workflow Engine 根据状态 Retry、DLQ、Quarantine 或终止，而不是让 LLM 从自然语言里猜失败语义。

---

## 8. Markdown 与图片：派生格式不能成为事实层

Notebook 直接使用 Crawl4AI Markdown，并将 Markdown 再转成 HTML 后用 BeautifulSoup 寻找图片。

### 8.1 Markdown 不是原始事实

Crawler、抽取算法或过滤规则升级，都可能让同一个 HTML 产生不同 Markdown。如果 Markdown 直接作为事实，就无法判断变化是源站变化还是抽取器变化。

知识库应采用：

```text
Immutable Raw Artifact
 -> Extraction Candidate
 -> Quality Gate
 -> Canonical IR
 -> Document Version
 -> Deterministic Markdown Renderer
```

Crawl4AI Markdown 只作为 Candidate/诊断产物。

### 8.2 图片不要 Markdown -> HTML 往返

图片/链接/媒体应来自原始 DOM、Crawler media metadata、HTML attributes、Structured Metadata：

- relative -> absolute；
- canonical asset URL；
- 保存 alt/title；
- 可选对象存储归档；
- sha256 去重；
- 记录来源 Artifact。

这样不会因派生格式往返丢失信息。

---

## 9. Critique -> Revision：项目最值得迁移的思想

原项目 Writer 输出后不是直接保存，而是先 Critique，再 Revision。

知识库应映射成：

```text
Extract Candidate
 -> Deterministic Quality Gate
 -> PASS/WARN
    or
 -> FAIL_RECOVERABLE
 -> Bounded Repair
 -> Quality Gate
```

Repair 顺序可为：

```text
Generic Extractor
 -> Deterministic Recipe
 -> DOM Repair + Generic
 -> Browser Refetch（有历史收益证据时）
 -> QUARANTINE
```

### 9.1 Quality Gate

至少检查 title/author/date/canonical、正文长度、paragraph/heading、link density、boilerplate、代码/表格/列表、错误页/WAF/CAPTCHA、Candidate Agreement、与历史版本的异常截断和 Page Type 冲突。

### 9.2 Revision 必须有限

Repair 有 max_steps、max_wall_time、browser_budget、max_cost 和 Source circuit breaker。修复不能变成“让 LLM 一直改到看起来正确”。第三方正文不能由模型改写后标成 Canonical Source Content。

---

## 10. Agent Memory：不能作为业务状态

示例中多个 Agent 使用 `memory=True`。这对于写作体验有帮助，但生产控制面若把关键事实放入 Agent Memory 会产生：

- 无法审计；
- 无法确定性 Replay；
- Context Window 截断可能静默丢事实；
- 多 Worker/多实例状态不一致；
- 模型升级后难以比较输入是否相同。

正确方式：

```text
Step A output
 -> JSON/Pydantic Schema Validate
 -> Object Storage
 -> content hash
 -> evidence refs
 -> Step B consumes refs
```

Agent Memory 只作为临时推理上下文。

---

## 11. Evidence Store 与 Context Window

Research Task 倾向把大量 raw content 直接交给下一 Agent。如果抓取多个完整页面直接串进 Context，会造成 Token 成本、重复材料、早期材料被截断、来源不可追踪和 Prompt Injection 面扩大。

生产控制面使用：

```text
Raw Evidence Artifact
 -> deterministic metadata / bounded excerpt
 -> evidence_id
 -> Agent context
```

Agent 的每条建议带 `evidence_refs`。最终 Validator/Replay 直接读取 Evidence Artifact，而不是依赖模型“记得什么”。

---

## 12. Maxim Trace：思想正确，但需要平台级 Trace Tree

项目用 Maxim 记录 LLM/Tool/Event，这是值得保留的思路。但 Notebook 中 Tool 自己创建 Trace 的方式不够：如果每个 Tool 都自行生成独立 trace id，就难以还原“某次 Source Probe -> 某个 Agent Step -> 某次模型调用 -> 某个 Tool Call”完整父子关系。

知识库应统一 OpenTelemetry 或等价模型：

```text
workflow_run(root trace)
 -> workflow_step
    -> agent_run
       -> model_call
       -> tool_call
       -> evidence_ref
 -> candidate_release
```

跨队列/Worker 传播 trace context；Tool 只能创建 child span，不能自行成为新的业务 Root Trace。若第三方 SDK 无法继承 parent，至少保存 root/parent correlation id，在平台重建完整树。

### 12.1 Trace 不存无限原文

Trace 只保存 object key、hash、size、schema/version、必要摘要。Cookie/Auth Header、Secret、大 Prompt/Response、未经策略允许的 PII 不应无界写入日志。

Observability 不能成为第二套未经治理的正文存储。

---

## 13. FileSaverTool：任意路径能力不适合 Agent

示例 FileSaverTool 接收 `save_folder` 和 `file_name`，由 Agent 写本地路径。这在个人 Notebook 中方便，在服务端属于过宽能力：

- path traversal；
- 覆盖非目标文件；
- 临时容器不可持久；
- 多 Worker 冲突；
- 无稳定版本、hash、权限与审计。

生产只能暴露语义化能力：

```text
SaveCandidateRelease(source_id, object_ref)
SaveDiagnostic(workflow_run_id, object_ref)
```

底层固定对象存储命名空间，并由 API 校验权限。Canonical Markdown 必须由 Renderer/Projection Worker 写入，而不是由 LLM File Tool 写入。

---

## 14. 进一步优化：Tool 必须按副作用分类

仅有 Tool allowlist 仍不够。不同 Tool 的重试和安全语义不同，应分为：

```text
READ_ONLY
CANDIDATE_WRITE
EXTERNAL_SIDE_EFFECT
```

### READ_ONLY

如 ProbeSitemap、FetchSample、InspectDOM。可在预算和 Rate Limit 下安全重试。

### CANDIDATE_WRITE

如 SaveCandidateRelease、SaveDiagnostic。必须限制目标命名空间，并使用：

```text
idempotency_key + content_hash + fencing/version check
```

避免 Agent Retry 导致重复写。

### EXTERNAL_SIDE_EFFECT

如未来的外部发布、修改第三方系统等。默认不允许 Agent 自动调用；若业务需要，必须 Human Approval、独立权限、明确幂等/补偿语义。

这比简单的“Agent 没有 Shell”更完整，因为很多风险并不来自 Shell，而来自合法 API 的高副作用调用。

---

## 15. Crew 不应成为生产流程的外层状态机

原 Demo 使用一个 Crew 顺序执行 Research、Outline、Writing、Critique、Revision、Export。对一次性的写作任务是合理的，但知识库控制面需要持久状态、分支、Retry、Resume、审批和幂等。

CrewAI 当前产品模型也把 Crews 定位为自治协作，把 Flows 定位为更结构化、事件驱动、有状态、可恢复的编排，并支持在 Flow 中嵌入 Crew。对本项目而言，即使未来不使用 CrewAI Flow，也应吸收这个边界：

```text
Deterministic Workflow Engine / State Machine
 -> Run bounded Agent/Crew Activity
 -> validate structured output
 -> persist result
 -> decide next state
```

而不是：

```text
Crew owns everything
 -> Agent memory owns context
 -> tools mutate environment
 -> final answer means success
```

参考：
- https://docs.crewai.com/core-concepts/Agents
- https://docs.crewai.com/

关键原则是 **确定性外层编排 + 局部智能 Pocket**。

---

## 16. 显式 Workflow Contract：解决自由文本阶段交接

Demo 的阶段很大程度依赖“上一个 Task 输出文本供下一个 Task 阅读”。这对写作很自然，但生产系统必须让每个 Step 有明确机器契约。

示例：

```yaml
step: profile_planner
consumes: ProbeEvidenceSet/v3
produces: CandidateProfile/v4
required_evidence_roles:
  - robots
  - discovery
  - article_sample
  - non_article_sample
allowed_tools: []
retry_policy: agent-semantic-v2
on_validation_failure: REVISE_OR_REVIEW
```

工作流执行：

```text
Input Snapshot
 -> Step
 -> Structured Output
 -> Schema Validate
 -> persist object + hash
 -> next Step
```

自由文本只作为 explain/rationale 字段，不承担唯一协议作用。

这样能解决：

- Prompt 改动造成字段遗漏；
- 上游错误文本渗透到下一步；
- 不同模型输出格式漂移；
- Replay 无法确认输入是否一致；
- 无法静态验证 Candidate Release。

---

## 17. Input Snapshot：保证 Agent 决策可重放

即使 Prompt/Model/Tool 都版本化，如果 Replay 时 Agent 看到的 Source Profile、Evidence 列表、策略默认值已经改变，也无法真正复现。

因此每个 workflow_run 应冻结：

```text
input_snapshot
- active_source_release_id
- agent_release_id
- workflow_contract_release
- evidence_refs + hashes
- policy refs
- site-family release
- runtime/tool releases
- created_at
```

Agent Run 只消费这个 Snapshot 的引用。

Replay 分两种：

1. **Exact Input Replay**：用相同 Input Snapshot + 相同 Agent Release 比较非确定性差异；
2. **Upgrade Evaluation**：保持 Input Snapshot 不变，只替换新 Agent/Prompt/Model Release，评估质量差异。

这比只记录 Agent Memory 或最终 Prompt 更可靠。

---

## 18. 多模型路由与 Release

Demo 把 chat/manager/planning/function calling 都绑定同一 Gemini，简单但难以治理。

生产控制面可按职责：

- 小模型：标签、归纳、样本初筛；
- 强模型：复杂 Probe/Quarantine 根因；
- Planner 与 Critic 使用不同 model release 减少同源偏差；
- Tool Calling 可使用更稳定的专门模型。

所有变化都进入：

```text
agent_release
- workflow_version
- workflow_contract_release
- prompt_release
- tool_schema_release
- planner_model_release
- critic_model_release
- function_call_model_release
- max_steps
- max_tool_calls
- max_tokens
- max_cost
- max_wall_time
```

任何 Agent 输出只产生 Candidate，不直接改 ACTIVE Source Release。

---

## 19. 推荐控制面工作流

把文章的 Research -> Outline -> Write -> Critique -> Revise 思路映射为：

```text
Deterministic Workflow Engine
 -> Freeze Input Snapshot
 -> Evidence Collector
 -> Profile Planner
 -> Rule/Recipe Generator
 -> Critic
 -> Reviser
 -> Candidate Release Assembler
 -> Static Validation
 -> Golden Replay
 -> Live Sample
 -> CANARY
 -> Metrics Compare
 -> Human Approval
 -> ACTIVE
```

### 19.1 Evidence Collector

仅调用受限 READ_ONLY Tool：ProbeRobots、ProbeSitemap、ProbeFeed、FetchSample、InspectDOM、InspectStructuredMetadata、SearchGapEvidence、ReplaySample。

### 19.2 Profile Planner

提出 Discovery Provider、Page Type pattern、Route、Extractor Portfolio、Quality Policy、Repair Sequence。

### 19.3 Critic

检查过拟合、Index/Tag/Author 遗漏、Browser 滥用、selector 脆弱、Coverage Evidence、robots/egress、安全和 Canonical 污染风险。

### 19.4 Reviser

只修改 Candidate Config，不修改 ACTIVE。

### 19.5 Candidate Release

必须经过 Deterministic Validator、Golden Replay、Canary 和 Approval。

---

## 20. Agent 预算和有限状态

即使 Agent 位于低频控制面，也必须设置：

- max_steps；
- max_tool_calls；
- max_wall_time；
- max_tokens；
- max_cost；
- per-provider RPM；
- Tool allowlist；
- side-effect policy；
- source-scoped egress；
- circuit breaker。

预算耗尽进入 `DEGRADED_NEEDS_REVIEW`，不能无限“反思 -> 搜索 -> 重试”。Agent 故障不能阻塞 HTTP/Incremental 数据面。

---

## 21. 从 Demo 到 1000 站平台的映射

| Demo 能力 | 知识库平台定位 | 生产约束 |
|---|---|---|
| Tavily Search | Gap Search / Probe Adapter | 不做主 Discovery |
| Crawl4AI Tool | Browser Fetch / Dynamic Discovery Adapter | 长生命周期池、受限并发 |
| Agent Memory | Temporary reasoning context | 不做业务真相 |
| Research Agent | Evidence Collector | 低频、只读 Tool |
| Outline Agent | Profile Planner | Structured Candidate |
| Writer Agent | Rule/Recipe Generator | 不改写正文 |
| Critique Agent | Agent Critic + Deterministic Validator | Critic 不能最终批准 |
| Revision Agent | Candidate Config Reviser | 有限次数 |
| Export/FileSaver | Candidate Writer / Renderer | 固定命名空间、幂等 |
| Crew Process | Bounded Agent Activity | 外层 Workflow Engine 持久状态 |
| Maxim Trace | OpenTelemetry Provenance | 单 Root Trace、跨 Worker 传播 |
| Crawl4AI Markdown | Extraction Candidate | 不做 Canonical |
| Blog Markdown | Canonical IR -> Renderer | 确定性、可重建 |

---

## 22. 对主技术方案的最终优化项

本次调研应最终落实为：

1. Browser Worker 复用 Browser/Crawler/Context，禁止每 URL 新建昂贵运行时；
2. 所有并发 bounded，禁止对全量 URL 一次性 gather；
3. Notebook `nest_asyncio` 不进入生产 Worker；
4. Tool 使用 Typed Result/Typed Failure；
5. Agent 只传版本化 Artifact/Evidence Ref，Memory 不是真相；
6. Evidence 大内容外置，Agent Context 有界；
7. Critique/Revision 映射成 Quality Gate 与有限 Repair，不让 LLM 改写第三方正文；
8. Canonical 主链坚持 Raw Artifact -> Candidate -> Quality -> IR -> Renderer；
9. Agent/Prompt/Model/Tool Schema/Workflow Contract 全部 Release 化；
10. 禁止 Agent 任意文件路径、Shell、任意 JS/Code Execution；
11. Tool 增加 READ_ONLY/CANDIDATE_WRITE/EXTERNAL_SIDE_EFFECT 分类和幂等规则；
12. Crew/Agent 作为有界 Activity，外层由持久化 Workflow Engine/State Machine 编排；
13. 每个 Step 有 consumes/produces Schema、Required Evidence、Failure/Retry Policy；
14. 每个 Workflow Run 冻结 Input Snapshot，支持 Exact Replay 和 Upgrade Evaluation；
15. Trace 使用单 Root Trace 贯通 Step/Model/Tool，禁止孤立 Tool Trace；
16. Runtime 必须通过 Fresh Container + Golden Corpus 冷启动测试；
17. Candidate Release 必须经过 Static Validation -> Golden Replay -> Canary -> Human Approval。

---

## 23. 最终判断

这篇文章最重要的启发不是“用 CrewAI 搭几个 Agent”，而是 **把复杂任务拆成职责清晰、可批评、可修订、可观测的阶段**。

但生产知识库需要再多走一步：不能让自治 Agent 同时掌握流程状态、工具副作用和最终发布权。

最终推荐边界：

```text
数据面：
Authoritative Discovery
 -> Classification / Route
 -> HTTP First
 -> Governed Browser Fallback
 -> Immutable Artifact
 -> Extraction Portfolio
 -> Deterministic Quality
 -> Bounded Repair
 -> Canonical IR
 -> Version
 -> Markdown

控制面：
Persistent Deterministic Workflow
 -> Immutable Input Snapshot
 -> Evidence Collection
 -> Bounded Agent Planning
 -> Agent Critique / Revision
 -> Structured Candidate
 -> Deterministic Validation
 -> Golden Replay
 -> Canary
 -> Human Approval
 -> Active Release
```

这样既继承多 Agent 的阶段化、Critique/Revision、Tool Adapter 和 Observability 优点，又把 Browser 生命周期、无界并发、Notebook 状态、自由文本错误、隐式 Memory、任意文件写、孤立 Trace、非幂等副作用和模型不确定性限制在可治理边界内，适合 1000+ 技术博客全历史抓取、长期增量同步和持续扩站。