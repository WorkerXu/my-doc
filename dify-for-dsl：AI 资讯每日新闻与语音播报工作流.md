# dify-for-dsl：AI 资讯每日新闻与语音播报工作流

- 项目地址：https://github.com/wwwzhouhui/dify-for-dsl
- 重点文件：`dsl/crawl4ai/aibase_craw4fastapi.py`
- 工作流：`dsl/AI资讯每日新闻+语音播报工作流.yml`
- 依赖文件：`dsl/crawl4ai/requirements.txt`
- 调研目标：分析“列表发现 → Crawl4AI 抽取 → FastAPI → Dify → LLM → TTS → 音频交付”的实现细节、边界与生产化问题，并提炼适用于 1000 个技术博客长期知识库的方案优化。

## 1. 项目整体链路

项目把“网页采集”做成一个小型 FastAPI 服务，再让 Dify 负责摘要、文本拼装、TTS 和最终回复：

```text
用户选择新闻数量
  -> Dify Code 节点请求 http://127.0.0.1:8086/news/?limit=N
  -> FastAPI /news/
  -> requests.get("https://www.aibase.com/zh/news")
  -> BeautifulSoup 遍历 a[href]
  -> 过滤 /zh/news/ 详情链接
  -> 对每个 URL 创建 AsyncWebCrawler
  -> JsonCssExtractionStrategy 提取 title/publication_date/content
  -> 返回 news[] + newsdetail
  -> Dify LLM 汇总 news[]
  -> Template 拼接“LLM 摘要 + newsdetail 全文”
  -> TTS 工具生成音频
  -> Code 节点拼接 <audio> HTML
  -> Answer 返回文本和音频
```

它非常适合证明“爬虫服务可以作为 AI 工作流的一个能力节点”，但其运行模型是单站点、少量最新内容、一次请求内同步完成，不适合作为 1000 站点历史知识库的核心架构。

## 2. Discovery：当前页 DOM 链接扫描

`get_news_urls()` 直接请求 AIbase 新闻列表页，用 BeautifulSoup 遍历所有 `<a href>`，通过路径中是否包含 `/zh/news/` 判断详情链接。

### 可借鉴点

- Discovery 和正文 Extraction 已经有基本分层。
- URL 是从 DOM 链接产生，而不是先转 Markdown 再正则找链接。
- 对结构稳定、规模很小的单站点，规则简单且成本低。

### 生产问题

1. 没有 URL 去重，同一链接在推荐区、导航区出现多次时可能重复抓取。
2. 没有 normalize/canonical，无法处理 tracking 参数、尾斜线、协议和 canonical 迁移。
3. 只扫当前列表页，没有 Sitemap、Feed、API、Archive、分页、Category/Tag/Author 等历史枚举能力。
4. 没有 Discovery Evidence，无法回答 URL 来自哪个 Provider、哪个 referrer、何时发现。
5. 没有 Provider checkpoint 或 reconcile，每次调用都从当前页重新开始。
6. 没有“每日新增”业务水位，连续运行可能反复取到同样的前 N 条新闻。
7. 列表请求没有显式 timeout、retry、backoff、响应大小限制。

### 对知识库的结论

Discovery 必须抽象为版本化 Provider。Provider 只产出 URL Candidate + Evidence；完整性由 Coverage Evidence 证明，不能用“列表当前页扫完”或“前 N 条已经拿到”代替历史完整性。

## 3. CSS 结构化抽取与 Selector Drift

项目的 `JsonCssExtractionStrategy` Schema 核心类似：

```text
baseSelector: div.pb-32
fields:
  title -> h1
  publication_date -> div.flex.flex-col > div.flex.flex-wrap > span:nth-child(6)
  content -> div.post-content
```

这种方式对已知站点很有价值：便宜、快、确定性强，比每页用 LLM 做结构化抽取更适合作为主路径。

但 `span:nth-child(6)` 和样式类 `pb-32` 都高度依赖当前 DOM，站点稍微改版就可能错位。现有代码只检查 `result.success`，随后直接取 `extracted_data[0]['title']`，没有区分：

- 网络/浏览器成功；
- Schema 是否匹配；
- title/date/content 是否完整；
- 正文是否是挑战页、导航、空内容；
- 日期是否抽到了错误字段。

因此生产系统必须把 Transport Outcome、Extraction Outcome、Quality Outcome 分离。CSS/XPath Schema 应属于 `site_profile_release/extractor_release`，并由 Golden URL、selector miss 指标、Snapshot Replay、shadow test、Drift 检测和回滚保护。

## 4. Crawl4AI 生命周期与吞吐

`extract_ai_news_article()` 每处理一篇文章都会：

```text
async with AsyncWebCrawler(...) as crawler:
    await crawler.arun(...)
```

`fetch_news()` 又使用普通 `for` 循环逐篇 `await`。因此当前实现同时存在：

- 每篇重新创建/销毁 crawler 生命周期；
- 浏览器/连接池不能充分复用；
- N 篇文章串行执行；
- 某一篇慢请求拖住整批；
- 单次 HTTP 请求耗时近似随 N 线性增长；
- 无 durable task、lease、heartbeat、断点续跑。

生产系统应改成长期 Browser/HTTP Runtime Pool + 持久 Task。Worker 内可以有界并发，但进程内 semaphore 只能做本机资源保护，不能替代任务持久化与全局调度。

Browser Context、HTTP Session、Model Runtime 等长生命周期资源必须有 hard cap、idle eviction、in-flight protection、最大寿命/页面数和最终释放机制。

## 5. Async FastAPI 中使用阻塞 I/O

FastAPI 路由是 `async def`，但列表页调用使用同步 `requests.get()`。这会直接阻塞 Event Loop。Dify Code 节点内部同样使用阻塞 `requests.get()`。

即使把这些调用替换为 `httpx.AsyncClient`，一次 Web 请求中同步等待“列表抓取 → N 篇 Browser → LLM → TTS”仍然不适合作为生产接口，因为它把最慢、最不稳定的外部依赖全部串在用户请求生命周期里。

生产接口应采用 Job 语义：

```text
POST /v1/runs                -> 202 + run_id
POST /v1/derived-jobs        -> 202 + job_id
GET  /v1/jobs/{id}           -> 状态/进度/result_ref
GET  /v1/artifacts/{id}      -> Artifact 元数据与访问引用
```

Dify、n8n、Agent 和 Web 都只创建任务、查结果或订阅 webhook，不直接占用请求线程等待 Browser/LLM/TTS 完成。

## 6. `bypass_cache=True` 不等于增量同步

示例用 `bypass_cache=True` 表达“确保最新”。这只控制 Crawl4AI 自身缓存行为，并不能回答业务层的 freshness。

长期同步应先使用低成本变化信号：

```text
Feed GUID/updated
Sitemap lastmod
API cursor
ETag / Last-Modified
body hash
Canonical IR hash
```

变化或未知才抓正文。网络表示变化但 Canonical IR hash 不变时，只写 Freshness Observation，不创建新 Document Version，也不重新 Embedding、摘要或 TTS。

## 7. Dify 工作流的输入 Contract 漂移

FastAPI 返回：

```text
{
  "news": [...],
  "newsdetail": "今天新闻第1条内容：..."
}
```

Dify Code 节点又兼容 `news` 元素可能是 dict，也可能是 JSON 字符串。这说明跨服务 Contract 并不稳定。`newsdetail` 进一步把多篇文章全文拼成一个无结构大字符串。

长期系统不应该让 `content/newsdetail/text` 在多个阶段复用不同含义。建议固定表示：

```text
DOCUMENT_VERSION_REF
MARKDOWN_PROJECTION_REF
AI_INPUT_PROJECTION_REF
AI_DERIVED_ARTIFACT_REF
AUDIO_ARTIFACT_REF
DELIVERY_PAYLOAD_REF
```

大型正文跨服务传 Artifact ID/Object Key，而不是反复复制整块 JSON。

## 8. LLM 摘要：整批全文输入的问题

工作流直接把 `news[]` 整体交给一个 LLM，模型与参数写在 DSL 中，摘要 temperature 为 0.7。小规模演示可以工作，但扩大后会遇到：

- 多篇全文总 token 超限；
- 不可见截断；
- 超长单篇挤掉其他文章；
- 摘要波动较大，难以做稳定 diff/cache；
- 失败只能整批重跑；
- 单篇摘要无法复用；
- 输出没有稳定的 `document_version_id` 和 block citation；
- Prompt、模型、温度和输出 Schema 没有作为不可变 Recipe 单独治理。

生产方案应采用单篇 Map/Reduce + 多篇 Digest Reduce：

```text
Document Version x N
  -> AI Input Projection
  -> block-aware Chunk Plan
  -> 单篇 Map Summary
  -> 单篇 Reduce Summary
  -> Selection/Ranking
  -> Digest Reduce
  -> Digest Artifact
```

摘要类 Recipe 应保存 prompt、temperature、输出 JSON Schema、语言、模型 Release、token budget、截断策略。面向知识库的摘要建议使用稳定、低随机度配置，并让每个 Digest Item 带 `document_version_id`、title、canonical、published_at 和可追溯 block refs。

## 9. “每日新闻”需要 Selection Manifest，而不是 `limit=N`

该 DSL 实际由用户手动选择新闻数量，默认选项只有 `2`。这并不构成可重放的“每日”语义：今天什么时候执行、覆盖哪个时区、跨午夜怎么办、失败重跑是否会多选或少选，都没有定义。

长期系统应增加不可变 Selection Manifest/Digest Batch：

```text
selection_manifest
- id
- trigger_type              # SCHEDULED/EVENT/MANUAL/WEBHOOK
- timezone
- window_start/window_end
- as_of
- selection_policy_release_id
- query/filter/scope
- ordered_document_version_ids
- ranking/dedup_evidence
- manifest_hash
- created_at
```

先冻结“这次 Digest 到底选了哪些 Accepted Version”，再运行 LLM/TTS。失败重试始终复用同一 manifest，避免查询结果随时间变化导致同一个“日报”内容漂移。

## 10. TTS：不应把“摘要 + 多篇全文”整体转语音

DSL 先把 LLM 摘要和 `newsdetail` 全文重新拼接，再送给 TTS；工作流注释还指出两条新闻语音生成大约需要 5 分钟。这直接说明 TTS 已经是高成本、长耗时任务。

生产形态应是：

```text
Digest Artifact
  -> TTS Recipe
  -> TTS Worker
  -> Audio Object
  -> Audio Artifact
  -> Delivery
```

默认只对摘要/Digest 做 TTS，并显式限制最大字符/token、目标音频时长、模型/音色 Release、deadline、并发/GPU、成本和重试。TTS 失败不能反向影响 Source Sync、Document Version、Markdown、Search 或 Embedding。

## 11. Audio 输出 Contract 与渲染安全

DSL 的音频处理 Code 节点存在两个值得生产方案吸收的问题：

1. 代码调用 `json.loads(arg1)`，但节点代码本身没有 `import json`。Python 不会自动提供 `json` 名称，因此这属于明显的运行时依赖/代码完整性风险。
2. 代码把 TTS 返回的 `etag` 当作 URL，再直接拼成 `<audio><source src='...'>` HTML。这样既耦合了供应商返回字段，又把渲染逻辑和 Artifact Contract 混在一起。

生产系统应由 TTS Adapter 把供应商返回统一规范化为：

```text
audio_artifact
- id
- source_derived_artifact_id
- tts_model_release_id
- voice_release_id
- mime_type
- duration_ms
- bytes
- object_key
- content_hash
- created_at
```

访问 URL按需生成短期 signed URL。Web/Dify 客户端根据 typed metadata 渲染播放器，不从供应商字段直接拼原始 HTML。

## 12. 服务发现、认证和网络边界

Dify Code 节点写死 `http://127.0.0.1:8086/news/`。在容器/Kubernetes 中，`127.0.0.1` 指向当前容器/Pod，不代表 crawler 服务。

生产系统应使用 Integration Gateway/Service Discovery，配置 Endpoint、认证、Secret、connect/read/total timeout、response byte limit、retry budget、circuit breaker、trace id。

如果外部工作流允许传 URL，则必须走标准 `TARGETED_FETCH` Admission：scheme/host、DNS/egress、SSRF、robots、policy、响应大小、Snapshot、Quality、Identity、Audit 一个都不能绕过。

## 13. 依赖与可重复运行问题

`requirements.txt` 固定了 `uvicorn`、`fastapi`、`cos-python-sdk-v5` 和 `crawl4ai==0.4.247`，但源码直接 import 的 `requests`、`bs4` 并未作为直接依赖显式列出。项目还要求单独执行 Playwright Chromium 安装。

这类 Demo 常依赖“某个上游包恰好带来了 requests/bs4”或运行环境已有浏览器，生产环境不可接受。需要：

- 所有直接 import 都声明为直接依赖；
- 使用 lockfile；
- Browser/Playwright revision 固化在镜像；
- image digest + SBOM；
- Crawl4AI/Playwright/Browser/Parser 版本写入 Runtime Release；
- CI 做 import/contract/startup test，避免依赖传递关系变化后突然启动失败。

## 14. Dify DSL 本身也需要版本化

当前方案容易只版本化模型或 Adapter，却忽略“Dify DSL 图本身”也是行为定义。节点顺序、变量映射、Prompt、Plugin、TTS 节点、代码节点任何改动都会改变结果。

因此建议增加 `workflow_recipe_release`：

```text
workflow_recipe_release
- id/version
- engine_type              # DIFY/N8N/INTERNAL
- dsl_object_key
- dsl_hash
- graph_hash
- compatible_engine_version
- input_schema/output_schema
- referenced_recipe_release_ids
- plugin/tool_release_refs
- source_commit
- test_result_refs
- created_at
```

Delivery/Derived Job 必须记录实际使用的 Workflow Recipe Release，这样才能重放“当时为什么生成了这个 Digest/TTS”。

## 15. 由该项目对博客知识库方案产生的最终优化

该项目最值得保留的不是“抓 AIbase 的代码”，而是它揭示了知识库后续一定会被工作流、LLM、TTS 等消费。因此生产方案应明确增加并强化以下能力：

1. **Workflow / Derived Artifact / Delivery Plane**：外部工作流是消费者，不是抓取真相源。
2. **异步 Job API**：Dify/n8n/Web 只创建 Run/Derived Job，不同步执行长链路。
3. **Selection Manifest**：日报/Newsletter 在生成前冻结版本集合、时间窗和排序依据，可重跑、可审计。
4. **Workflow Recipe Release**：Dify/n8n DSL/Graph 本身进入不可变 Release Registry。
5. **Structured Digest Item**：摘要逐条绑定 Document Version 和来源，不用无结构大字符串。
6. **Typed Audio Artifact**：TTS 供应商字段不直接泄漏到 UI，音频用对象存储与标准 metadata 管理。
7. **依赖显式化与 Runtime Attestation**：直接依赖、Browser revision、模型/Plugin/DSL 都可重复。
8. **Scheduler 拥有“每日”语义**：时区、cutoff、window、补跑由平台控制，不靠用户手动 `limit=N`。
9. **TTS 默认消费 Digest**：不把摘要和多篇全文再次拼接后无界转语音。
10. **外部工作流不得绕过主链路**：Targeted URL 仍需 Admission、Snapshot、Quality、Identity 和 Audit。

## 16. 最终结论

`dify-for-dsl` 是一个清晰的最小工作流样例：Crawl4AI 做站点专用结构化抓取，FastAPI 暴露能力，Dify 编排 LLM 与 TTS。它适合教学和少量即时内容消费，但不具备历史枚举、增量状态、持久调度、质量门禁、版本真相和规模化容错。

对 1000 技术博客知识库，正确的调用关系必须反转：

```text
长期 Source Sync
  -> Accepted Document Version
  -> Markdown/Search/Embedding
  -> Selection Manifest
  -> Summary/Digest Derived Artifact
  -> TTS Audio Artifact
  -> Delivery Job
  -> Dify / n8n / Agent / Newsletter / Webhook
```

Dify/n8n 负责消费和编排；Source Sync 负责真实、完整、增量、可追溯的数据。只有把这两层解耦，并把 Selection、Workflow DSL、AI/TTS 输出都版本化，才能让后续新增站点、增加消费者、切换模型或重放历史结果时仍保持可解释和可扩展。