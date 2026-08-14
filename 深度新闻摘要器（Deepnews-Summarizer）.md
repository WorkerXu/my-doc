# 深度新闻摘要器（Deepnews-Summarizer）

## 1. 调研对象与结论

- 项目：https://github.com/NhanPhamThanh-IT/Deepnews-Summarizer
- 调研基线：`main`，commit `e5782b4212b59e63a5996546168016504a658130`
- 仓库状态：已归档
- License：MIT
- 核心抓取代码：`local/utils/scrapper.py`、`local/utils/text_preprocessing.py`
- Local 摘要代码：`local/utils/summarize.py`
- Local 站点配置：`local/config.json`
- Local Web：`local/pages/01_Daily_News.py`、`local/pages/02_Custom_Fetch.py`
- Deploy Backend：`deploy/backend/main.py`、`deploy/backend/scraper.py`
- 模型训练：`models/preprocessing/preprocessing.py`、`models/finetuning/bart.py`、`models/finetuning/led.py`
- Local 依赖：`local/requirements.txt`

Deepnews-Summarizer 本质上是一个面向课程/演示场景的“新闻列表发现 → 单文章抓取 → 摘要 → Web 展示”应用。它没有 1000 个技术博客长期知识库所需的 durable frontier、Provider Coverage Evidence、全量历史回灌、增量 checkpoint、版本留存、不可变网络 Snapshot、离线重处理、跨站公平调度、持久任务状态和故障恢复，因此不适合直接作为生产底座。

它的价值在于同时提供了可复用思路和典型反例。可复用思路包括：配置驱动站点、列表发现与详情抓取分离、自定义 URL Targeted Fetch、前后端拆分、抓取引擎与摘要模型可替换。典型反例包括：每次抓取创建新的浏览器 Runtime、把 Markdown 全局压平成单段文本、固定 token 生切摘要、同步函数包裹 async、在 async 路径中执行阻塞式外部 AI 调用、把摘要以 `content` 字段冒充正文、站点 Selector 硬编码、无持久状态、训练数据不可重复和“模型声明长上下文但实际仍截断为 1024 token”。

对博客知识库方案最重要的结论是：**抓取、Canonical 内容、Markdown、检索 Projection、AI 输入和 AI 摘要必须是不同表示层；AI 永远只能作为 Accepted Document Version 的派生层，不能替代原文真相层。**

## 2. 项目整体执行架构

项目同时保留 Local 和 Deploy 两套实现，二者技术栈不同，但业务路径相似。

### 2.1 Local 版本

```text
Streamlit
  -> config.json
  -> Crawl4AI AsyncWebCrawler
  -> Markdown
  -> 列表页：Markdown 正则提取链接
  -> 文章页：CSS Selector 提正文
  -> 全局空白压平
  -> 本地 BART 摘要
  -> Streamlit 展示
```

Local 版本的抓取核心集中在 `local/utils/scrapper.py`。列表页和文章页都创建 `BrowserConfig`、`CrawlerRunConfig` 和新的 `AsyncWebCrawler` 上下文；列表页使用调用者传入的 CSS Selector，文章页则固定使用 `.vossi-paragraph`。

### 2.2 Deploy 版本

```text
Streamlit frontend
  -> FastAPI
     -> httpx
     -> BeautifulSoup
     -> CNN 固定 Selector
     -> OpenAI summary
     -> JSON
```

FastAPI 暴露：

- `GET /scrape`：抓 CNN 分类页并返回文章标题、URL；
- `GET /scrape-article`：抓单篇正文，立即调用 OpenAI 摘要，然后把摘要放在 `content` 字段返回。

Deploy 版不再依赖 Crawl4AI，而改用 `httpx + BeautifulSoup`。这说明项目自身已经证明“抓取引擎只是执行实现，不应成为业务数据模型”。生产方案应继续把 HTTP、Crawl4AI、Playwright、PDF/OCR 等放在统一 Engine Adapter 后面。

## 3. Site Config：方向正确，但粒度远远不够

`local/config.json` 把 CNN 分类页 URL、tab 名称和列表 Selector 放进配置：

```text
source/category URL
+ list selector
```

这比完全硬编码到页面代码更可维护，但生产系统需要升级成不可变的 Site Profile Release，而不是直接读一个可变 JSON 文件。

推荐 Profile 至少包含：

```yaml
site_profile:
  identity:
    allowed_hosts: []
    canonical_rules: []
  discovery:
    sitemap_rules: []
    feed_rules: []
    api_rules: []
    archive_rules: []
    list_urls: []
    list_selectors: []
    link_accept_rules: []
    link_reject_rules: []
  fetch:
    http_first: true
    browser_escalation_rules: []
    headers: {}
    timeout_policy: {}
  extraction:
    title_rules: []
    author_rules: []
    published_at_rules: []
    updated_at_rules: []
    body_rules: []
    remove_rules: []
  quality:
    expected_selectors: []
    min_text_chars: 300
    min_block_count: 3
```

生产 Profile 必须支持 Draft/Release、Schema 校验、Golden URL、Snapshot Replay、diff、审批、回滚和审计。配置只是行为定义，运行时还必须保存本次 Task 实际解析得到的 `resolved_profile_release_id` 和 Effective Config。

## 4. 浏览器生命周期：每 URL 创建 Runtime 不适合规模化

`local/utils/scrapper.py` 的两个抓取函数都会在每次调用时：

1. 创建新的 `BrowserConfig`；
2. 创建新的 `CrawlerRunConfig`；
3. 进入新的 `AsyncWebCrawler` 上下文；
4. 执行一次 `arun()`；
5. 退出上下文并释放 Runtime。

对交互式 demo 可接受，对数百万 URL 会不断重复支付 Chromium 进程启动、浏览器上下文初始化、JS Runtime 初始化、TLS/DNS/连接预热等固定成本。

生产实现应采用长期 Runtime：

```text
Browser Worker Process
  -> long-lived Browser Runtime
     -> Context Pool
        -> Page Lease
```

任务只租用 Page/Context，正常完成后归还；仅在崩溃、内存泄漏、站点隔离策略或 Release 切换时重建 Runtime。

HTTP 也一样。Deploy 版每个函数都创建新的 `httpx.AsyncClient()`，无法充分复用连接池。生产 HTTP Worker 应维护长期 `AsyncClient`，并配置 keep-alive、per-host connection limit、DNS/TLS 复用、超时、重试、限流和熔断。

## 5. CacheMode.BYPASS 不是“最新性”方案

Local 抓取把 `CacheMode.BYPASS` 作为默认逻辑。它只意味着尽量重新访问页面，不等于拥有正确的增量同步语义。

长期知识库需要把“是否需要抓正文”与“抓到什么正文”分开：

```text
Change Signal
  -> Feed updated/GUID
  -> Sitemap lastmod
  -> API cursor/version
  -> ETag / Last-Modified
  -> conditional GET
  -> body hash
  -> Canonical IR hash
  -> Freshness Observation
```

如果内容未变化，不应重复生成 Document Version、Chunk、Embedding 和摘要；只记录一次 Freshness Observation 即可。浏览器自己的静态资源缓存也不应和业务层 Snapshot/Freshness 混为一谈。

## 6. 同步/异步桥接存在阻塞与死锁风险

`get_links_from_homepage()` 和 `get_content_from_direct_url()` 对外是同步函数，内部定义 async helper，然后根据当前线程是否已有 event loop 选择：

- 没有 loop：`asyncio.run(...)`；
- 已有运行中的 loop：`asyncio.run_coroutine_threadsafe(..., loop).result()`。

如果调用线程本身就是该 loop 所在线程，`.result()` 会阻塞当前线程，而 coroutine 又依赖同一个 loop 推进，存在死锁风险。

Deploy 版还有另一层问题：`scrape_direct_cnn_article_content()` 是 async，但内部调用同步 `summarize()`；`summarize()` 又发起阻塞式 OpenAI 调用。也就是说，一个慢 AI 请求可以直接占住 Web 进程的 event loop。

生产系统应该遵循：

```text
Web/API Command
  -> PostgreSQL Run/Task
  -> Transactional Outbox
  -> Queue
  -> async Worker
  -> persisted Outcome
  -> SSE/WebSocket/轮询展示结果
```

网络、浏览器、OCR、Embedding、本地 GPU 推理和外部 AI 都不能在 Web 请求线程里长时间同步等待。

## 7. Discovery：DOM → Markdown → Regex 会丢失发现证据

列表页先由 Crawl4AI 把 DOM 转成 Markdown，再用 `extract_links_from_markdown()` 从 Markdown 正则提取链接。这条链实际是：

```text
DOM
 -> Markdown Projection
 -> Regex
 -> URL
```

问题是 Markdown 已经是降维后的表示，转换时会损失：

- 原始 `href` 与 base URL 解析上下文；
- DOM 位置和 CSS/XPath 位置；
- `rel`、data-* 等属性；
- 列表卡片所在栏目；
- 同一 URL 在多个区块出现的多条独立 Evidence；
- 原始页面顺序和父子关系。

`extract_links_from_markdown()` 还把 `(title, url)` 放进 set 去重。set 适合做临时唯一化，不适合承担业务顺序和发现证据。

生产 Discovery 应直接从 DOM/Feed/Sitemap/API 产生：

```text
URL Candidate
+ Discovery Evidence[]
```

Evidence 至少记录：

- `parent_url`；
- `href_raw`；
- `href_resolved`；
- anchor/title；
- DOM/CSS/XPath position；
- provider；
- provider_release；
- `snapshot_id`；
- `observed_at`；
- 分类/作者/分页上下文。

URL 可以去重，但 Evidence 不能被覆盖。

## 8. 固定 `.vossi-paragraph` 是典型站点漂移风险

Local 和 Deploy 的文章正文都依赖 CNN 的 `.vossi-paragraph`。这会把“某个时间点某个站点的 DOM 结构”误当成稳定协议。

对 1000 个技术博客，正文规则必须进入 Site Profile，并对 0 命中做语义诊断，而不是简单返回空字符串：

```text
EMPTY_EXPECTED
EMPTY_UNEXPECTED
SELECTOR_DRIFT
WRONG_PAGE_TYPE
CHALLENGE_PAGE
FETCH_INCOMPLETE
PARSER_ERROR
POLICY_BLOCKED
```

管理端需要持续展示 Selector hit rate、正文字符数分布、block 数分布、Golden URL smoke test、Profile release 前后 replay 差异。规则漂移应被当成可观测事件，而不是“偶尔抓不到”。

## 9. `merge_in_one_paragraph()` 会破坏技术内容结构

`local/utils/text_preprocessing.py` 里的 `merge_in_one_paragraph()` 使用全局空白折叠，把所有换行和连续空白压成一个空格。

对新闻纯文本摘要尚可接受，但对技术博客会破坏：

- heading 与章节边界；
- Markdown 列表层级；
- fenced code block；
- Python/YAML/Makefile 等缩进；
- Markdown table；
- shell/日志输出；
- JSON/config；
- blockquote；
- 段落语义。

生产系统必须先形成结构保真的 Canonical IR，例如：

```json
{
  "blocks": [
    {"id":"b1","type":"heading","level":2,"text":"..."},
    {"id":"b2","type":"paragraph","text":"..."},
    {"id":"b3","type":"code","language":"python","text":"..."},
    {"id":"b4","type":"table","rows":[]}
  ]
}
```

Markdown 是 Canonical IR 的确定性 Projection。AI 如果需要纯文本，应该生成独立 `AI_INPUT_TEXT` Projection；AI Projection 可以规范空白，但必须保留 `block_id -> text span` 映射，且绝不能反向覆盖 Canonical IR 或 Markdown。

## 10. Local BART 摘要的实际原理

`local/utils/summarize.py` 在模块 import 时加载 Hugging Face `pipeline("summarization")` 和 BART tokenizer。每篇文章处理步骤是：

1. 全文 tokenization，不截断；
2. 按固定 1000 token 切块；
3. 每块单独 decode 回文本；
4. 每块独立生成约 30~100 token 的摘要；
5. 把所有块摘要直接用空格拼接。

这实际上是一个 **map-only summarization**，没有真正的 reduce 阶段。

优点：

- 实现简单；
- 单块输入可控；
- `do_sample=False`，输出相对稳定；
- 模型在进程 import 时加载，避免每次请求重新加载权重。

缺点：

- 1000-token 边界不理解 heading、段落或句子；
- 没有 overlap，跨边界事实容易丢失；
- 没有 reduce，文章越长最终摘要越长；
- 不同块之间可能重复相同事实；
- 代码、日志、表格和自然语言被混在同一种输入里；
- 没有 chunk-level lineage，无法解释最终一句摘要来自哪些原文块；
- 没有输入 hash/输出 hash，失败重试不能复用已完成 map 结果。

生产摘要应改成：

```text
Accepted Document Version
  -> AI Input Projection
  -> block-aware Chunk Plan
  -> Map Artifact[]
  -> optional hierarchical Reduce Artifact[]
  -> factuality/coverage/format validation
  -> Final Derived Artifact
```

每个 chunk 保存：`chunk_id`、block range、token count、input hash、recipe release、model release、runtime release、输出、耗时和错误。这样单块失败可独立重试，Recipe 升级也可精确重建。

## 11. 摘要不能伪装成正文

Deploy 的 `scrape_direct_cnn_article_content()` 抓到正文后立即调用 OpenAI，FastAPI `/scrape-article` 最终返回：

```text
{"content": <summary>}
```

这里 `content` 实际是摘要，不是原文，也不是清洗正文。这会制造 Representation Ambiguity：调用者无法从字段名判断拿到的是网络响应、正文、Markdown 还是 AI 输出。

生产 API 必须明确区分：

```text
fetch_snapshot
rendered_dom_snapshot
extraction_candidate
canonical_document_version
markdown_projection
ai_input_projection
summary_artifact
```

任何 AI 成功都不能成为“抓取成功”的必要条件。抓取主链路先独立完成并接受 Document Version，再异步触发 AI。

## 12. Deploy HTTP 路径的生产缺口

`deploy/backend/scraper.py` 的 HTTP 抓取属于最小 demo：

- 每次新建 `AsyncClient`；
- 未形成统一 timeout policy；
- 无 retry/backoff；
- 无 per-host limiter；
- 无 robots/policy admission；
- 无 SSRF 检查；
- 无 ETag/Last-Modified；
- 无 Snapshot；
- 无响应体 hash；
- 无 content-type/page-type validator；
- 无 challenge/WAF 诊断；
- 无重定向安全检查；
- 无阶段级 Typed Outcome。

生产 HTTP-first 的正确含义不是“只用 httpx”，而是：

```text
HTTP Worker
  -> long-lived AsyncClient
  -> admission/policy
  -> per-host rate/concurrency limit
  -> conditional request
  -> redirect validation
  -> immutable Fetch Snapshot
  -> typed FetchOutcome
  -> extraction
  -> quality gate
  -> evidence-driven Browser escalation
```

Browser 只在 HTTP 证据表明动态渲染、正文缺失或页面挑战时升级，不能默认所有站点都启动浏览器。

## 13. Web 产品形态值得保留，但执行方式必须改造

项目的 Daily News 和 Custom Fetch 两种交互，和生产管理端的两个核心入口非常接近。

### 13.1 Daily News → Source / Provider 管理

可以演化成：

- Source 列表与健康状态；
- Provider Coverage；
- 最近同步时间；
- 新增/更新/未变化/失败计数；
- Drift 告警；
- 历史 Backfill 进度；
- 增量 checkpoint。

### 13.2 Custom Fetch → Targeted Fetch

任意 URL 输入必须走标准生产主链路：

```text
URL
 -> normalization
 -> SSRF/policy validation
 -> Source/Profile resolution
 -> persistent TARGETED_FETCH Task
 -> Fetch Snapshot
 -> Extract
 -> Canonical IR
 -> Quality
 -> Identity
 -> Preview/Accept
```

不能像 demo 一样在 Web 请求内直接启动浏览器和模型并阻塞等待。

### 13.3 同 Snapshot 策略比较

CSS/XPath、Readability/Trafilatura、Crawl4AI、LLM extraction 的比较必须消费同一 Snapshot。如果每种策略各自重新访问源站，就无法区分“抽取策略差异”和“网页在两次请求之间发生变化”。

## 14. 模型训练数据预处理存在可重复性和正确性问题

`models/preprocessing/preprocessing.py` 暴露了几个对生产 MLOps 很有启发的反例。

### 14.1 随机抽样没有固定 seed

代码先把 CNN/DailyMail 的 train/test/validation 合并，然后执行约 12% 随机采样，但没有固定 `random_state`。同一份代码重复执行会得到不同子集，后续训练结果自然无法精确复现。

生产 Dataset Release 必须记录：

- 原始数据集版本/hash；
- 过滤规则版本；
- random seed；
- split manifest；
- 最终样本 id 列表/hash；
- 预处理代码 commit；
- 依赖和镜像 digest。

### 14.2 `tokenized_highlights` 实际使用了 article 字段

预处理脚本构建 `tokenized_highlights` 时再次读取 `article`，而不是 `highlights`。因此后续的 highlights token count 实际变成 article token count，所谓“摘要长度过滤”并没有按目标摘要执行。

这种错误说明训练数据转换不能只依靠 notebook 目测，必须有 Dataset Contract Test，例如：

- source/target 字段非同列；
- 样本 id 一致；
- target 长度分布合理；
- source != target 比例；
- split 不重叠；
- schema 与 null 检查。

### 14.3 采样后索引与布尔过滤存在对齐风险

`df.sample()` 默认保留原索引，而后生成的 cosine DataFrame 使用新的 RangeIndex。再用后者的布尔 Series 过滤前者时，存在 pandas 索引对齐不一致的风险。生产预处理应显式 `reset_index(drop=True)` 或使用稳定 sample id join，禁止依赖隐式 DataFrame index 语义。

### 14.4 “全局词表 overlap”指标退化

脚本先把所有样本词汇 union 成 `global_vocab`，然后对每个样本计算 `len(x & global_vocab) / len(global_vocab)`。由于每个样本的 `x` 本来就是 global vocab 的子集，交集等于自身，本质更接近“样本词汇量占全局词汇量比例”，并不能证明 train/validation/test 的语义覆盖质量。

这类自定义数据质量指标必须先明确统计含义，并配测试样本验证公式行为。

## 15. BART/LED 训练揭示“声明上下文长度 ≠ 实际有效上下文”

项目同时有 BART 和 `allenai/led-base-16384` 训练脚本。LED 名称暗示 16K 长上下文，但实际 `preprocess_function()` 仍对输入设置 `max_length=1024` 并 `truncation=True`。

因此运行事实是：**这个训练路径实际只消费最多 1024 token，而不是 16384 token。**

这对博客知识库非常关键。不能只在数据库里记录 `model_name` 或厂商宣称的 context window，必须记录 Runtime Attestation：

```text
tokenizer release
input projection
configured max_length
actual token count
truncated token count
attention/global-attention policy
dtype
quantization
device
generation config
library versions
model weight hash
```

长上下文模型只有在数据预处理、attention 配置、显存策略和推理参数都真正启用时才算“有效长上下文”。

## 16. 训练与推理配置被混在脚本里，不利于 Release 管理

BART/LED 脚本把模型名、Kaggle 路径、训练参数、量化配置、评估和模型保存写在同一 notebook 风格文件中。生产上应该拆成：

```text
Dataset Release
Model Training Recipe Release
Model Artifact
Model Evaluation Artifact
Runtime Release
AI Recipe Release
```

其中：

- Dataset Release 决定训练数据；
- Training Recipe 决定训练超参数；
- Model Artifact 是权重/tokenizer/hash；
- Evaluation Artifact 保存离线指标；
- Runtime Release 决定如何加载和执行模型；
- AI Recipe 决定面向某类文档如何切块、提示、reduce 和校验。

这样才能区分“模型权重变了”“推理量化方式变了”“Prompt/Chunk 变了”还是“输入文档变了”。

## 17. ROUGE/BERTScore 不能单独证明生产摘要正确

训练脚本使用 ROUGE 和 BERTScore 对参考摘要做离线评估。这些指标适合模型实验，但对知识库生产摘要仍不足以证明：

- 事实没有幻觉；
- 版本号/命令/函数名没有改写错误；
- 代码事实没有丢失；
- 风险声明、限制条件没有漏掉；
- 长文不同章节覆盖均衡；
- 摘要中的每个断言可回溯原文。

生产 AI 质量应额外维护：

- citation/block lineage coverage；
- unsupported claim 检查；
- entity/version/code-token preservation；
- section coverage；
- truncation rate；
- duplicate fact rate；
- human review sample；
- Golden Corpus 回归。

摘要可失败、可重试、可降级，但不能影响原文 Document Version 的接受状态。

## 18. 模型领域不匹配：新闻摘要不能直接等价技术博客摘要

项目训练数据来自 CNN/DailyMail 新闻摘要，内容风格与技术博客明显不同。技术博客中常包含：

- API 名称；
- 版本号；
- 命令；
- 文件路径；
- 错误码；
- 代码块；
- 配置；
- benchmark；
- 注意事项和兼容性条件。

新闻摘要模型倾向于自然语言压缩，可能把这些高价值技术 token 当成噪声。生产系统不能把某个新闻摘要模型写死成默认知识库摘要器，应使用 AI Model Registry + Recipe Registry，并按文档类型做路由和评估。

## 19. 依赖供应链不可重复

`local/requirements.txt` 只对部分依赖固定版本，Crawl4AI、aiohttp、BeautifulSoup、lxml、nltk、numpy、pydantic 等大量包没有 pin，同时还有重复依赖条目。

这会导致：

- 同一 requirements 在不同日期得到不同依赖树；
- Crawl4AI/Playwright 行为变化；
- parser 输出漂移；
- 模型与 tokenizer 版本变化；
- 难以重放历史结果。

生产 Release 至少需要：

- lockfile；
- container image digest；
- Python/OS/Browser 版本；
- SBOM；
- 模型/tokenizer hash；
- Playwright browser revision；
- Crawl4AI/Trafilatura 版本；
- 构建 provenance。

## 20. 对 1000 技术博客知识库的能力映射

| 能力 | Deepnews-Summarizer | 生产知识库要求 |
|---|---|---|
| 多站点配置 | 基础 JSON URL/Selector | 版本化 Site Profile + Adapter |
| 全历史发现 | 分类页列表 | Sitemap/Feed/API/Archive/内链/Gap Discovery + Coverage Evidence |
| 增量同步 | 无 | checkpoint + conditional fetch + overlap + reconcile |
| 持久任务 | 无 | Run/Task durable state + lease/heartbeat |
| 跨站公平调度 | 无 | host quota + priority + fairness + backpressure |
| HTTP 抓取 | 基础 httpx | 长期连接池 + policy + Snapshot + typed outcome |
| Browser 抓取 | 每次创建 Crawl4AI runtime | Browser Runtime Pool + evidence escalation |
| 正文抽取 | 固定 CNN selector | 多 Extractor + Profile + Replay + Drift |
| 内容结构 | 压平文本 | Canonical IR + deterministic Markdown |
| 版本历史 | 无 | append-only Document Version |
| 资产归档 | 无 | Asset Pipeline + hash + object storage |
| 搜索 | 无 | OpenSearch + vector + hybrid/rerank |
| Web 管理 | Streamlit demo | Control Plane + Source/Profile/Run/Quality/Drift/Assets |
| Targeted Fetch | 有交互雏形 | 持久标准流水线 |
| AI 摘要 | BART/OpenAI 直连 | Derived Artifact + Recipe/Model/Runtime Release |
| AI lineage | 无 | chunk/map/reduce lineage |
| MLOps | notebook 脚本 | Dataset/Training/Model/Eval/Runtime Release |
| 可重复构建 | 部分依赖未锁 | lockfile/image digest/SBOM |

## 21. 应纳入博客知识库最终方案的设计

### 21.1 保留并升级的设计

1. 配置驱动站点 → Site Profile Release。
2. 列表发现与单文章抓取分离 → Discovery Route 与 Article Fetch Route。
3. Crawl4AI 与 httpx 都可执行抓取 → Crawler Engine Adapter。
4. Custom Fetch → 标准 `TARGETED_FETCH`。
5. 本地模型与远程 OpenAI 可切换 → AI Model/Runtime Registry。
6. Web 交互 → Control Plane + 异步任务状态。

### 21.2 必须禁止的实现方式

1. 不把 Markdown 先压平再作为 Canonical 内容。
2. 不从 Markdown 正则反向恢复 Discovery 证据。
3. 不在所有文章上硬编码单一 Selector。
4. 不为每个 URL 重启浏览器 Runtime。
5. 不在 async Web 路径执行阻塞式 AI 调用。
6. 不把摘要写入模糊的 `content` 字段。
7. 不让 AI 摘要成功与否决定抓取成功与否。
8. 不用模型名称推断有效上下文。
9. 不允许训练数据随机采样无 seed、无 manifest。
10. 不允许生产依赖只写浮动版本。

### 21.3 新增工程能力

1. `AI_INPUT_TEXT` 独立 Projection，保留 block span lineage。
2. block-aware Chunk Planner，支持 heading/paragraph/code/table 语义。
3. Map/Reduce AI Artifact 持久化，单块可重试与复用。
4. AI Recipe Release、AI Model Release、Dataset Release、Runtime Release 分离。
5. Runtime Attestation 保存真实 tokenizer/truncation/context/device/quantization/generation 参数。
6. 模型训练保存 dataset hash、sample manifest、seed、代码 commit、镜像 digest。
7. AI 质量同时检查 factuality、技术 token 保真、section coverage 和 lineage coverage。
8. Model Runtime Pool/外部 AI Client Pool 与抓取 Worker 隔离伸缩。
9. Targeted Fetch、批量同步、离线 Replay 共用同一 Artifact Contract。
10. 依赖 lockfile、镜像 digest、SBOM 和能力测试进入 Release Gate。

## 22. 最终评价

Deepnews-Summarizer 适合作为“抓取 + 摘要 + Web 展示”的教学参考，不适合作为 1000 站点博客知识库的运行底座。

它对最终方案最有价值的贡献不是某个具体库，而是帮助明确了几个架构边界：

- 抓取引擎可以替换，业务真相层不能跟着替换；
- Site Config 必须升级为有版本、有测试、有回滚的 Profile；
- 原始内容、Canonical IR、Markdown、AI 输入和摘要必须分层；
- 长文摘要需要 block-aware map/reduce 和完整 lineage；
- 本地模型、远程 LLM、训练数据和推理 Runtime 都必须被独立版本化；
- “模型理论能力”必须由真实 Runtime Attestation 证明；
- Web 端的自定义抓取只是命令入口，不应成为同步执行器；
- 生产可重复性既包括抓取规则，也包括依赖、模型、数据集和训练过程。

这些约束应作为长期知识库平台的默认工程规则，而不是某个站点的特殊补丁。