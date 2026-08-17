# 两年后重新构建我的 LLM 网页爬虫：到底发生了哪些变化

## 1. 调研对象与结论

- 文章：Rebuilding My LLM Web Scraper Two Years Later: What Actually Changed
- 作者：Nacho Corcuera Platas
- 文章地址：https://medium.com/@ignacio.cplatas/rebuilding-my-llm-web-scraper-two-years-later-what-actually-changed-8dd2f6d0645d
- 对应项目：https://github.com/NachoCP/AIScraping
- 调研时间：2026-08-14

文章和仓库展示了一个很有代表性的 2024 → 2026 演进：网页获取从 `requests + BeautifulSoup + 手工清洗 + 额外浏览器` 收敛到 Crawl4AI，结构化抽取从 `Prompt + PydanticOutputParser` 收敛到模型原生 Structured Output/JSON Schema，多模型通过 LiteLLM/Provider Adapter 统一。

这些变化非常适合吸收到“1000 个技术博客、全量历史、长期增量”的知识库平台，但只能吸收**工具层和契约层思想**，不能照搬项目的单请求运行模型。当前仓库本质仍是轻量单用户工具，存在每请求 Browser 生命周期、同步 FastAPI 调用链、嵌套线程桥、无 durable queue/checkpoint、无跨站公平调度、LLM 每页调用、缓存语义隐式、依赖/文档/容器配置漂移等问题。

对知识库方案最值得新增或强化的能力有六项：

1. Crawl4AI 作为可替换 Adapter，而不是平台状态真相源；
2. Browser/LLM Worker 改为 async-native 长生命周期资源池；
3. 原生 Structured Output + Provider Schema Compiler + Schema Release；
4. 显式缺失值语义，禁止用 `0/false/""` 冒充真实观测值；
5. 显式 Cache Policy 与 Runtime Bundle Release，避免第三方隐藏缓存和依赖漂移破坏可重复性；
6. Schema Compiler 必须具备 `$ref` 循环检测、展开预算和 Provider 能力验证，不能简单递归内联。

## 2. 文章中的 2024 与 2026 架构变化

### 2.1 2024 版

文章描述的旧版路径为：

```text
URL
 -> requests
 -> BeautifulSoup 清理 nav/footer/script
 -> PromptTemplate
 -> ChatOpenAI
 -> PydanticOutputParser
 -> Pydantic object
```

主要故障面：

- 页面抓取和正文清洗由业务代码自己维护；
- JavaScript 页面需额外叠 Selenium/Playwright；
- JSON 格式靠 Prompt 约束；
- Markdown 代码围栏、字段名漂移、额外文字都会导致解析失败；
- 缺少系统化重试和 Provider 抽象；
- 页面超过模型上下文窗口时直接失败。

### 2.2 2026 版

新路径被压缩为：

```text
URL
 -> Crawl4AI
 -> Markdown
 -> LiteLLM / native structured output
 -> JSON Schema constrained output
 -> Pydantic validate
 -> typed result
```

核心变化不是“LLM 更聪明”这么简单，而是**两个易碎接口被协议化**：

1. 网页 → 可消费文本，由 Crawl4AI 封装抓取、浏览器渲染和 Markdown 生成；
2. LLM → 业务对象，由 JSON Schema/Structured Output 将输出形状从概率性 Prompt 约定提升为模型/API 协议约束。

对大规模知识库而言，这会显著降低集成代码，但不会自动解决持久化、全量覆盖、队列、幂等、成本、质量和长期运维问题。

## 3. 仓库实际实现

## 3.1 `src/crawler.py`：Crawl4AI 的同步桥

核心逻辑：

```text
crawl(url)
 -> _run_async(_crawl_async(url))
 -> ThreadPoolExecutor(1)
 -> 子线程 asyncio.run(coro)
 -> AsyncWebCrawler.arun(...)
 -> result.markdown.raw_markdown
```

每次 `_crawl_async()` 都创建：

```text
BrowserConfig(headless=True)
CrawlerRunConfig(
  cache_mode=CacheMode.BYPASS,
  page_timeout=30000
)
AsyncWebCrawler(..., thread_safe=True)
```

成功后只向后续抽取暴露 `raw_markdown`。

### 技术原理

`AsyncWebCrawler` 基于异步运行时和 Playwright。同步调用方不能随意在已有事件循环里再执行 `asyncio.run()`，因此项目把 coroutine 发送到一个新线程，并在新线程创建独立 event loop。

```text
调用线程
 -> 创建工作线程
 -> 工作线程创建 event loop
 -> event loop 驱动 Crawl4AI/Playwright
 -> 阻塞等待
```

这是**兼容桥**，不是生产并发架构。

### 平台级问题

- 每次调用创建/销毁 ThreadPoolExecutor；
- 每次调用创建/销毁 event loop；
- 每次调用重新进入 Browser/Crawl4AI 生命周期；
- Browser 成为默认抓取路径；
- 无统一 Browser 并发/内存预算；
- 无站点级 token bucket；
- 无 durable lease/checkpoint；
- 请求取消和底层 Browser 取消关系不清晰。

`thread_safe=True` 解决的是对象在特定线程环境中的调用安全，不等价于跨站点公平调度、资源池、背压或任务持久化。

## 3.2 FastAPI 同步路由带来的“嵌套阻塞”

`api/main.py` 的 `/api/scrape` 使用同步 `def` 路由，并同步执行：

```text
crawl(url)
 -> build schema
 -> extract(...)
 -> return
```

FastAPI/Starlette 对同步路由通常会使用线程池执行；而 `crawl()` 内部又创建一个单独线程池并等待子线程中的 `asyncio.run()`。

因此在并发场景中实际可能形成：

```text
HTTP request
 -> FastAPI/AnyIO worker thread
 -> crawl()
 -> child ThreadPoolExecutor thread
 -> new asyncio loop
 -> Playwright/Crawl4AI
```

随后同一个请求还会同步等待 LLM 调用。

这会导致：

- 每个请求长期占用 API 执行资源；
- 路由线程阻塞等待子线程；
- Browser 和 LLM 延迟直接放大 API tail latency；
- 并发上限受线程池和 Browser 内存共同约束，却没有统一 admission；
- 客户端断开不等于任务可恢复；
- 无 durable progress。

生产平台应改成：

```text
POST /jobs
 -> PostgreSQL transaction
 -> outbox
 -> queue
 -> async-native Worker
 -> Web 查询 job/status 或 SSE/WebSocket
```

控制面不直接承担重型 Browser/LLM 生命周期。

## 3.3 `CacheMode.BYPASS`：第三方缓存不能成为业务缓存语义

项目每次 Crawl4AI 调用都使用：

```text
CacheMode.BYPASS
```

对单用户工具很简单，但知识库平台需要明确区分三层缓存：

1. HTTP 条件请求缓存语义：ETag/Last-Modified/304；
2. 平台不可变 Snapshot：raw bytes、headers、DOM、Markdown；
3. 第三方引擎内部缓存：Crawl4AI/Browser 的执行优化。

第三方缓存不应偷偷决定“是否重新访问”“返回哪个版本内容”。生产方案需要 `cache_policy_release`，并规定：

- freshness 由平台 Scheduler/Provider checkpoint 决定；
- 业务可回放依据平台 snapshot；
- 第三方引擎缓存若启用，只作为透明执行优化；
- 每个 attempt 记录 cache hit/miss/source/version；
- 不允许无法审计的隐藏缓存覆盖条件请求和版本判断。

## 3.4 `src/schema_builder.py`：动态 Pydantic Schema

项目使用 Pydantic v2 `create_model()`，把用户定义字段动态编译成模型。

支持：

```text
string
integer
float
boolean
list[string]
```

并创建顶层 envelope：

```text
ResultModel(items=list[ItemModel])
```

这是值得借鉴的设计：管理端可把字段定义编译成不可变 Schema Release，再供抽取任务引用。

### 默认值问题

项目默认值为：

```text
string       -> ""
integer      -> 0
float        -> 0.0
boolean      -> false
list[string] -> []
```

这会把“缺失”与“真实值”混在一起。例如 `views = 0` 无法判断究竟是页面明确写 0，还是模型没找到数据。

更关键的是，字段 Python 类型本身并不是 `Optional`，随后 Strict Schema Compiler 又会把所有 object properties 放进 `required`。因此源码注释中的“Optional-style defaults”并不能真正表达可缺失语义。

知识库平台应显式存：

```text
value
presence = OBSERVED | INFERRED | MISSING
confidence
provenance
```

Schema 中使用 nullable/optional 或领域专门的 evidence 结构，而不是把默认值伪装成事实。

## 3.5 `src/extractor.py`：原生 Structured Output

项目先：

```python
schema = result_model.model_json_schema()
```

再生成：

```text
response_format = {
  type: json_schema,
  json_schema: {
    name: extraction_result,
    strict: true,
    schema: ...
  }
}
```

最后：

```text
json.loads(response.choices[0].message.content)
 -> result_model.model_validate(...)
```

### 技术原理

支持 Structured Output 的模型/API 会在生成阶段约束输出，使 token 序列满足给定 Schema 的结构空间。这消除了大量传统问题：

- JSON 语法错误；
- 字段名随机变化；
- Markdown code fence；
- 多余解释文本；
- 顶层结构漂移。

但它只保证**结构契约**，不保证**事实正确性**。模型仍可能：

- 将更新时间当成发布时间；
- 从导航区域抽错标题；
- 在证据不足时猜值；
- 对语义模糊字段选错值。

所以 Structured Output 后仍必须做 provenance、字段证据、确定性候选交叉验证和 Quality Gate。

## 3.6 `_make_strict_schema()` 的实现风险

项目为了兼容严格 Schema 做了：

1. 提取 `$defs`；
2. 递归解析 `$ref`；
3. 给 object 加 `additionalProperties: false`；
4. 将所有 properties 加入 `required`；
5. 递归处理 array/anyOf/oneOf/allOf。

这是一个很好的原型，但不能直接成为生产级 Schema Compiler。

### 风险 1：递归 `$ref`

`_resolve_refs()` 没有 visited set/cycle detection。若将来 Schema 支持递归模型，自引用或互相引用可能导致无限递归。

### 风险 2：Schema 膨胀

同一 `$defs` 被多处引用时直接内联会复制整棵子树。复杂 Schema 可能产生指数式体积膨胀，最终超过 Provider Schema 大小限制。

### 风险 3：Provider 能力不同

不同 Provider 对 `$ref`、union、nullable、enum、数字约束、数组和顶层 schema 的支持子集并不完全一致，不能用一个“统一内联算法”假设所有模型等价。

### 生产方案

Provider Schema Compiler 至少要有：

```text
canonical schema
 -> normalize
 -> reference graph
 -> cycle detection
 -> provider capability validation
 -> preserve ref / bounded inline
 -> max depth
 -> max expanded nodes/bytes
 -> required/nullable transform
 -> additionalProperties policy
 -> provider-specific fixture test
 -> effective_provider_schema_hash
```

如果超出能力，必须在发布期失败或显式降级，而不是在百万级运行时逐页试错。

## 3.7 Provider Adapter：统一 API 不等于统一能力

项目用 LiteLLM 的 `provider/model` 字符串统一模型，例如：

```text
openai/gpt-4.1-mini
anthropic/claude-sonnet-4-20250514
```

方向是正确的：业务代码不应散落厂商 SDK。

但知识库平台不能只保存“字符串模型名”，还需要 Model Capability Manifest：

```text
provider
model_id
supports_strict_json_schema
supports_tool_calling
supports_streaming
max_context_tokens
max_output_tokens
schema_subset
rate_limit_class
retry_policy
price_input
price_output
region
```

并且 Adapter 输出必须归一化，不能让业务层依赖某个 SDK 固定的：

```text
response.choices[0].message.content
```

平台应统一成：

```text
StructuredExtractResult
- parsed
- raw_payload_ref
- provider_request_id
- finish_reason
- token_usage
- latency
- cost
- error_class
```

这样更换 LiteLLM、厂商 SDK 或 tool calling 路径时不会污染业务层。

## 3.8 重试：指数退避正确，但异常分类过粗

项目用 Tenacity：

```text
wait_random_exponential(min=1, max=30)
stop_after_attempt(3)
retry_if_exception_type(Exception)
```

指数退避和 jitter 是正确方向，但所有 `Exception` 都重试会重复消耗 token 和延迟。

可重试：

```text
connect/read timeout
408
429
provider 5xx
transient gateway error
```

通常不可原样重试：

```text
schema compile error
unsupported schema keyword
invalid credential
permanent context overflow
budget exceeded
deterministic validation/config error
```

平台需要统一 error taxonomy、Retry-After、最大尝试次数、circuit breaker 和 dead-letter/review。

## 3.9 依赖、README、Dockerfile 漂移：一个很现实的生产风险

当前仓库不同文件之间存在明显不一致：

- `src/extractor.py` 实际 import `litellm`，但根 `pyproject.toml` 的 dependencies 没有声明 `litellm`；
- README 仍描述 `openai.parse()/anthropic.parse()`、Streamlit UI；
- 当前源码已有 `api/` 和 `frontend/`；
- Dockerfile 仍执行 `streamlit run app.py`、暴露 8501；
- 根目录当前并没有与该 Docker 命令相匹配的 `app.py`，且 pyproject 也未声明 streamlit。

这说明“代码看起来能运行”与“某个 commit 可重复构建部署”是两回事。

对知识库平台的直接启示：配置 release 还不够，还需要 **Runtime Bundle Release**：

```text
runtime_bundle_release_id
container_image_digest
source_commit_sha
python_lock_hash
crawler_engine_version
playwright_version
browser_revision
adapter_version
schema_compiler_version
build_sbom_ref
created_at
```

并在发布前执行：

```text
clean build
 -> dependency resolve
 -> unit tests
 -> golden fixture extraction
 -> browser smoke test
 -> structured output canary
 -> image digest pin
```

这样才能真正回放“当时那个抓取器/模型适配器”的行为。

## 3.10 安全边界

项目请求体直接接收 `api_key`，CORS 为 `*`，适合 demo，不适合长期平台。

生产平台必须：

- API Key 存 Vault/KMS/Secret Manager；
- Web 只提交 `credential_id`；
- 日志/trace/异常统一 redaction；
- CORS 明确 origin；
- API 做认证、RBAC、审计；
- 所有抓取 URL 经过 SSRF/Egress Gate；
- redirect 每跳重新校验；
- Browser 也必须受同一网络出口策略约束。

## 4. 哪些设计可以直接吸收

### 4.1 Crawl4AI Adapter

吸收：JS 渲染、Markdown、动态页面能力、deep crawl、CSS/XPath/Regex 等执行能力。

不吸收：任务状态、全量完成判定、全局 frontier、跨站公平调度、文章版本真相。

### 4.2 Dynamic Schema → Schema Release

管理端定义字段后，编译成 canonical JSON Schema/Pydantic，并发布为不可变 release。

### 4.3 Native Structured Output

对确需 LLM 的语义抽取，优先：

```text
canonical schema
 -> provider schema compiler
 -> native Structured Output
 -> platform-normalized result
 -> Pydantic/domain validate
 -> Quality Gate
```

### 4.4 Provider Adapter

可用 LiteLLM，也可用厂商 SDK，但平台只依赖统一请求/结果契约和 Capability Manifest。

### 4.5 Exponential Backoff

保留指数退避 + jitter，但放入统一错误分类、预算和熔断框架。

## 5. 哪些设计不能照搬

```text
每请求 ThreadPoolExecutor
每请求 asyncio.run
每请求 Browser 生命周期
Browser 默认抓所有 URL
CacheMode.BYPASS 作为唯一缓存策略
同步 API 请求串行完成抓取 + LLM
每页都调用 LLM
所有 Exception 都重试
缺失值用 0/false/"" 代替
运行时临时修改 Schema 且不版本化
业务层依赖 SDK 原始 response 结构
README/Docker/依赖声明未通过一致性 CI
```

## 6. 对博客知识库技术方案的具体优化

### 6.1 async-native Worker

```text
FastAPI Control Plane
 -> durable job
 -> outbox
 -> queue
 -> Browser/HTTP/LLM Worker
```

Browser Worker 长期维护 event loop、Browser/Context/Page pool、semaphore、TTL 和 cleanup。

### 6.2 显式 Cache Policy

新增 `cache_policy_release`，区分 freshness、条件请求、snapshot replay、第三方 engine cache，并记录 cache provenance。

### 6.3 Runtime Bundle Release

新增 `runtime_bundle_release`，绑定 source commit、image digest、dependency lock、Playwright/browser revision、Crawl4AI/Adapter/Schema Compiler 版本。

### 6.4 Provider Schema Compiler 强化

新增：

- `$ref` graph；
- cycle detection；
- bounded expansion；
- max depth/bytes；
- Provider schema subset validation；
- fixture/canary；
- effective schema hash。

### 6.5 Missing Value Semantics

结构化字段不使用类型默认值伪装缺失；Article IR 统一保存 `presence/confidence/provenance`。

### 6.6 Provider Result Normalization

业务层不直接读取 `choices[0].message.content`，由 Adapter 统一生成 `StructuredExtractResult`。

### 6.7 构建与发布门禁

所有 crawler/adapter/runtime release 在投入生产前必须执行 clean image build、依赖完整性检查、golden snapshot replay、Browser smoke、Provider Structured Output canary。

## 7. 最终判断

文章的核心判断是成立的：到 2026 年，抓取和结构化输出的“基础管线”已经明显商品化，开发者不必再把大量时间花在 HTML 清洗和 Prompt JSON parser 上。

但从单用户工具升级到 1000+ 技术博客的长期知识库时，真正困难的部分已经上移到：

- 历史 URL 覆盖证明；
- durable frontier；
- 增量连续性；
- 幂等和版本；
- 资源公平调度；
- Browser/LLM 成本；
- Schema/Provider 差异；
- 质量和 provenance；
- runtime 可重复构建；
- Web 运维和安全。

因此最终方案应继续坚持“平台状态与第三方执行引擎解耦”，同时吸收 Crawl4AI、Structured Output、动态 Schema 和多 Provider 的开发效率。新增显式 Cache Policy、Runtime Bundle Release、健壮的 Provider Schema Compiler 和 Provider Result Normalization 后，方案的可回放性、可扩展性和长期运维能力会更完整。