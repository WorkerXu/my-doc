# 两年后重新构建我的 LLM 网页爬虫：到底发生了哪些变化

## 1. 调研对象

- 文章：Rebuilding My LLM Web Scraper Two Years Later: What Actually Changed
- 作者：Nacho Corcuera Platas
- 发布时间：2026-04-06
- 文章地址：https://medium.com/@ignacio.cplatas/rebuilding-my-llm-web-scraper-two-years-later-what-actually-changed-8dd2f6d0645d
- 对应项目：https://github.com/NachoCP/AIScraping
- 本次查看时间：2026-08-14

这篇文章的价值不在于给出一个可直接用于“1000 个博客、全量历史、长期增量”的现成系统，而在于非常清晰地展示了 2024 到 2026 年 LLM 网页抽取基础设施发生的变化：网页抓取层可以被 Crawl4AI 这类工具高度封装，结构化输出可以由模型原生 JSON Schema 约束替代提示词解析器，多模型接入可以通过统一 Provider Adapter 降低耦合，开发者因此能把更多精力投入重试、错误处理、UI 和产品能力。

但项目作者也明确承认它更接近单用户工具而不是高并发服务。对于本知识库平台，最有价值的是“采用哪些原则”，以及“哪些单机实现不能原样搬到生产架构”。

## 2. 2024 版与 2026 版的核心变化

文章中的旧版流程大致为：

```text
requests
 -> BeautifulSoup 清洗
 -> PromptTemplate
 -> ChatOpenAI
 -> PydanticOutputParser
 -> Pydantic 对象
```

主要问题是：

1. 抓取与 HTML 清洗代码由业务自行维护；
2. JavaScript 页面需要另外叠加 Selenium/Playwright；
3. JSON 格式主要靠提示词约束；
4. LLM 返回 Markdown 代码围栏、字段改名、格式漂移时，解析器直接失败；
5. 没有完整重试、错误分类和 Provider 抽象；
6. 超上下文窗口时直接失败。

2026 版被压缩为：

```text
URL
 -> Crawl4AI
 -> clean Markdown
 -> LiteLLM / 原生 Structured Output
 -> JSON Schema
 -> Pydantic validate
 -> typed result
```

核心变化是两个：

- 网页到可供 LLM 消费的 Markdown，由 Crawl4AI 封装抓取、浏览器渲染和基础清洗；
- LLM 到业务对象，由原生 Structured Output/JSON Schema 约束替代“请严格返回 JSON”的提示词约定。

这两个变化都适合吸收到本知识库方案，但应该放在 Adapter 和升级路径中，而不是让 Crawl4AI 或 LLM 直接成为业务状态真相源。

## 3. 项目实现细节

### 3.1 Crawl4AI 包装层

项目的 `src/crawler.py` 把 Crawl4AI 封装为同步函数 `crawl(url)`。核心实现逻辑是：

```text
crawl(url)
 -> _run_async(_crawl_async(url))
 -> ThreadPoolExecutor(1)
 -> 新线程中 asyncio.run(coro)
 -> AsyncWebCrawler.arun(...)
 -> result.markdown.raw_markdown
```

`_crawl_async` 每次请求都会创建：

- `BrowserConfig(headless=True)`；
- `CrawlerRunConfig(cache_mode=CacheMode.BYPASS, page_timeout=30000)`；
- `AsyncWebCrawler(config=browser_config, thread_safe=True)`。

抓取完成后，如果 `result.success == false`，返回错误；成功时只把 `result.markdown.raw_markdown` 暴露给后续 LLM。

这个实现对单用户 UI 很直接，但对于 1000 站点平台存在明显问题：

1. 每个同步调用都创建线程池；
2. 每次在线程中创建和销毁新的 asyncio event loop；
3. 每次 crawl 都重新进入 `AsyncWebCrawler` 生命周期；
4. Browser 成为默认入口，而不是 HTTP-first 的升级路径；
5. `CacheMode.BYPASS` 使 Crawl4AI 内置缓存每次都绕过；
6. 没有 durable task、lease、checkpoint、按域限流和跨请求资源池。

`thread_safe=True` 解决的是特定调用环境中的安全性问题，不应被理解为平台级并发调度方案。

### 3.2 为什么 ThreadPoolExecutor + asyncio.run 能工作

问题根源是同步 Web/UI 调用链与 Crawl4AI 的异步 Playwright 运行时不一致。`asyncio.run()` 不能安全地嵌套到一个已经运行的 event loop 中，因此项目把 coroutine 丢到一个独立线程，在该线程里创建一个新的 loop。

其本质是：

```text
同步调用线程
 -> 创建工作线程
 -> 工作线程创建独立 event loop
 -> event loop 驱动 Crawl4AI / Playwright
 -> 阻塞等待结果
```

这是一种兼容桥，而不是并发架构。流量上升后会出现：

- 线程创建/销毁成本；
- event loop 创建/销毁成本；
- Browser 生命周期频繁抖动；
- 同步请求线程长期阻塞；
- 对 Browser 内存峰值缺乏统一控制；
- 难以做 backpressure；
- 请求断开后，底层 crawl 的取消与回收不容易统一。

因此本知识库平台应反过来设计：Browser/Crawl4AI Worker 自身就是 async-native 服务，每个进程长期维护一个 event loop，并在受控并发额度内复用浏览器/上下文；FastAPI 控制面只入队，不直接执行重型抓取。

### 3.3 动态 Pydantic Schema

项目 `src/schema_builder.py` 使用 Pydantic v2 的 `create_model()` 根据用户在 UI 中定义的字段动态创建模型。

支持类型：

```text
string
integer
float
boolean
list[string]
```

字段定义形如：

```text
field_name -> {
  type,
  description
}
```

每个字段被转换为对应 Python 类型，并附带默认值：

```text
string       -> ""
integer      -> 0
float        -> 0.0
boolean      -> false
list[string] -> []
```

随后再创建：

```text
ResultModel(items=list[ItemModel])
```

原因是多数 Structured Output API 更适合以 object 作为顶层 JSON Schema，而不是裸 `list[Model]`。

这个模式非常适合知识库平台的“Schema Release”设计：Web 管理端可以定义需要的语义字段，后台编译为 Pydantic/JSON Schema，再以不可变 release 发布。

但默认值设计不适合知识库最终数据语义。例如：

```text
浏览量 = 0
```

到底表示“页面明确写了 0”，还是“没抽到所以填了 0”，二者不可区分。生产设计应优先使用：

```text
value = null
presence = MISSING | OBSERVED | INFERRED
provenance = ...
```

必要时再在下游展示层映射默认值。

### 3.4 Structured Output 的 Schema 编译

项目 `src/extractor.py` 先执行：

```python
schema = result_model.model_json_schema()
```

然后通过 `_make_strict_schema()` 对 Pydantic JSON Schema 做兼容处理，主要包括：

1. 抽出 `$defs`；
2. 递归解析 `$ref`，把引用内联；
3. 对所有 object 增加 `additionalProperties: false`；
4. 把 object 的全部 properties 加入 `required`；
5. 递归处理 array、anyOf、oneOf、allOf。

之后把 schema 传给 LiteLLM：

```text
response_format = {
  type: json_schema,
  json_schema: {
    name: extraction_result,
    strict: true,
    schema: strict_schema
  }
}
```

这比“在 Prompt 中写格式说明 + 后处理 JSON”稳定得多。其技术原理是：支持原生 Structured Output 的模型会在解码阶段约束可生成 token，使输出落在 Schema 允许的结构空间中。因此“JSON 语法是否合法、字段名是否错拼”从概率性提示词问题变成协议约束问题。

需要注意：Structured Output 只约束结构，不保证字段事实正确。模型仍可能：

- 把发布时间理解成更新时间；
- 从导航文本中取错标题；
- 对模糊字段给出错误类型语义；
- 在正文证据不足时猜值。

因此最终数据仍需要 provenance、确定性候选交叉验证和 Quality Gate。

### 3.5 LiteLLM Provider 抽象

项目使用 `litellm.completion()`，调用方传：

```text
provider/model
```

例如：

```text
openai/gpt-4.1-mini
anthropic/claude-sonnet-4-20250514
```

Provider 与展示模型名由 `src/config.py` 维护，API 端通过 `_resolve_litellm_model()` 转换。

它体现了一个重要原则：业务逻辑不应该把 OpenAI/Anthropic 的 SDK 细节散落在抽取代码中。

但生产平台还需要在统一 Adapter 之上增加 Capability Manifest，而不能只做字符串映射。至少要记录：

```text
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
provider_region
```

原因是“统一 API”不等于“所有模型能力完全一致”。相同 JSON Schema 在不同 Provider 上可能需要不同转换或降级策略。

### 3.6 重试实现

`extract()` 使用 Tenacity：

```text
wait_random_exponential(min=1, max=30)
stop_after_attempt(MAX_RETRIES)
retry_if_exception_type(Exception)
```

最大尝试次数为 3。

优点：

- 有随机指数退避；
- 避免瞬时错误立即放大；
- 调用代码保持简洁。

问题是它对所有 `Exception` 都重试。生产中必须把错误分为：

```text
可重试：
- connect/read timeout
- 408
- 429
- provider 5xx
- transient gateway error

通常不可原样重试：
- schema compile error
- unsupported schema keyword
- invalid API key
- context permanently too large
- deterministic validation failure
- budget exceeded
```

否则固定错误会重复烧 token 和延迟。

### 3.7 FastAPI 调用链

`api/main.py` 提供同步的：

```text
POST /api/scrape
```

调用顺序为：

```text
校验 fields/url
 -> resolve provider/model
 -> crawl(url)
 -> 动态 build Pydantic model
 -> extract(markdown)
 -> 返回 items
```

整个抓取和 LLM 抽取在一个 HTTP 请求生命周期中串行完成。

这适合交互式 demo，但不适合长任务平台。对本系统应改为：

```text
POST /jobs
 -> 持久化任务
 -> outbox
 -> worker async 执行
 -> Web 通过 job/status/stream 查询进度
```

用户关闭浏览器不应导致任务生命周期丢失。

### 3.8 安全边界

项目 API 请求体直接接收 `api_key`，CORS 使用 `allow_origins=["*"]`，更偏 demo 产品。

知识库平台不能照搬。需要：

- Provider Key 存 Vault/KMS/Secret Manager；
- Web 请求只引用 credential_id；
- 日志与 trace 对 secret 做 redaction；
- CORS 使用明确 origin；
- API 做身份认证、RBAC 和审计；
- 抓取 URL 必须经过 SSRF/Egress Gate。

## 4. 与文章描述存在的仓库状态差异

文章写的是 React frontend + FastAPI backend；当前仓库确实存在 `api/` 和 `frontend/` 目录，但根 README 的 Quick Start 和 Architecture 仍主要描述 Streamlit，并写有：

```text
Streamlit UI -> Crawl4AI -> LLM SDK -> JSON
```

同时 README 对 Structured Output 的说明也与当前 `src/extractor.py` 的 LiteLLM 实现存在一定演进差异。

这说明项目处于快速迭代状态，文档与实际代码可能不同步。对知识库平台的启示是：任何 Adapter、Schema、Prompt、Model、Crawler 配置都必须绑定 release/version/hash；不能只靠“当前 README 如何描述”重现历史结果。

## 5. 对 1000 博客知识库最值得吸收的原则

### 5.1 使用原生 Structured Output，而不是 Prompt Parser

对于必须由 LLM 完成的语义字段抽取：

```text
Pydantic Model
 -> JSON Schema
 -> Provider Schema Compiler
 -> native Structured Output
 -> Pydantic validate
 -> Quality Gate
```

不再把“请仅返回合法 JSON”作为核心可靠性机制。

### 5.2 LLM Provider 统一抽象，但能力必须显式化

可以使用 LiteLLM，也可以自建 Provider Adapter；无论实现方式如何，业务侧都只依赖统一接口：

```text
StructuredExtractRequest
StructuredExtractResult
```

同时保存真实 provider/model/schema/prompt/token/cost/latency。

### 5.3 动态 Schema 应变成版本化 Schema Release

Web 管理端可以让管理员定义：

```text
field name
type
description
required/nullable
identity key
validation rule
```

发布后生成不可变 `schema_release`，后续任务固定引用 release id，禁止线上静默修改。

### 5.4 Browser Worker 必须 async-native

禁止在平台级主路径采用：

```text
每请求 ThreadPoolExecutor
 + 每请求 asyncio.run
 + 每请求新 Browser 生命周期
```

正确方式：

```text
队列
 -> Browser Worker Process
 -> 一个长期 event loop
 -> bounded concurrency semaphore
 -> Browser/Context pool
 -> Crawl4AI Adapter
```

Browser Worker 的资源并发由内存和 CPU 限额决定，不由 API 请求数决定。

### 5.5 LLM 不能成为默认正文抽取器

文章项目的目标是“任意网页 + 用户自定义 schema -> JSON”，因此每个页面经过 LLM 是合理产品选择；本项目目标是数百万文章长期知识库，成本模型不同。

正文默认路径应继续是：

```text
JSON-LD/meta/CSS/XPath
 -> Trafilatura/Readability/Crawl4AI Markdown candidates
 -> Quality Gate
```

只有确定性结果低质量或确实需要语义字段时才进入 LLM。

### 5.6 重试必须按错误分类

指数退避是正确方向，但必须：

```text
error taxonomy
 -> retryable?
 -> provider retry-after
 -> jittered exponential backoff
 -> max attempts
 -> circuit breaker
 -> dead letter/review
```

禁止对所有异常无差别调用模型三次。

## 6. 本项目应新增或强化的设计

### 6.1 Async Runtime Topology

增加明确的运行时边界：

- FastAPI 只做控制面和轻量查询；
- HTTP Worker 使用长期 asyncio loop + connection pool；
- Browser/Crawl4AI Worker 使用长期 asyncio loop + Browser pool；
- CPU-heavy normalization 放独立进程池；
- LLM Worker 独立异步客户端和 provider concurrency limiter；
- 禁止在请求处理中创建新的 event loop 作为常规生产路径。

### 6.2 LLM Provider Adapter 与 Capability Manifest

新增：

```text
llm_provider_release
model_release
schema_release
structured_output_adapter_release
```

每个 model release 记录支持能力和限制，Provider Adapter 负责把平台 JSON Schema 编译到具体厂商支持的 schema 子集。

### 6.3 Strict Schema Compiler

Schema Compiler 做：

1. Pydantic/平台 Schema -> canonical JSON Schema；
2. 展开或保留 refs；
3. 根据 Provider 规则处理 `additionalProperties`、required、nullable、enum、union；
4. capability validation；
5. 计算 `canonical_schema_hash` 与 `effective_provider_schema_hash`；
6. 发布前跑 fixture/canary。

不能在每页请求时临时修改 Schema 而不保存版本。

### 6.4 Missing Value Semantics

避免用 `0/false/""` 掩盖缺失：

```text
value
presence
confidence
provenance
```

对于 Structured Output，可以让字段 nullable，或增加字段级 evidence/presence 信息；最终 Article IR 再按领域规则落值。

### 6.5 LLM Retry Policy

每次调用记录：

```text
attempt_id
provider_request_id
error_class
retry_after
input_tokens
output_tokens
estimated_cost
latency
```

只对瞬时错误重试；Schema 不兼容先走 Provider Schema Compiler 降级或人工修复，不能盲目重试。

## 7. 结论

这篇文章和项目证明了两个成熟方向：

1. Crawl4AI 可以显著降低“把网页变成干净 Markdown”的工程成本；
2. 原生 Structured Output 已经足以替代传统 LLM JSON Prompt Parser，Pydantic/JSON Schema 可以成为稳定的类型契约。

但项目中的 `ThreadPoolExecutor + asyncio.run`、每请求 Browser 生命周期、每页 LLM、同步 HTTP 请求串行执行、全异常重试和宽松 secret/CORS 设计，都只适用于轻量单用户工具，不能直接用于 1000 站点生产平台。

因此知识库方案应吸收其“抓取引擎封装、原生 Structured Output、动态 Schema、多 Provider、指数退避”思想，同时坚持平台自身的 durable frontier、HTTP-first、异步 Worker、不可变 snapshot、版本化 Schema/Model/Adapter、Quality Gate、成本治理和统一安全出口。这样既获得 2026 工具链带来的开发效率，又不会把 demo 级运行模型带入大规模长期任务系统。
