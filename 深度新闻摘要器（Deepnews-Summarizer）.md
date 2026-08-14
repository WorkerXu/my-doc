# 深度新闻摘要器（Deepnews-Summarizer）

## 1. 调研对象与结论

- 项目：https://github.com/NhanPhamThanh-IT/Deepnews-Summarizer
- 调研基线：`main`，commit `e5782b4212b59e63a5996546168016504a658130`
- 仓库状态：已归档，MIT License
- 核心抓取代码：`local/utils/scrapper.py`、`local/utils/text_preprocessing.py`
- 摘要代码：`local/utils/summarize.py`
- 站点配置：`local/config.json`
- Local Web：`local/pages/01_Daily_News.py`、`local/pages/02_Custom_Fetch.py`
- 部署版：`deploy/backend/main.py`、`deploy/backend/scraper.py`、`deploy/frontend/*`
- 模型与数据：`models/preprocessing/preprocessing.py`、`models/finetuning/bart.py`、`models/finetuning/led.py`
- 依赖：`local/requirements.txt`

Deepnews-Summarizer 是一个教学/课程型的“新闻列表发现 → 单文章抓取 → 摘要 → Web 展示”项目，不具备 1000 个博客长期知识库所需的 durable frontier、全量覆盖证明、增量同步、版本留存、离线重处理、跨站公平调度和持久运维能力，因此不适合作为生产底座。

它最有价值的地方是同时提供了正例和反例：正例是配置驱动站点、列表发现与文章抓取分离、自定义 URL 即时抓取、前后端拆分、可替换摘要模型；反例是每次新建重型 Runtime、把 Markdown 压平成单段、固定 token 生切摘要、同步/异步桥接、把摘要当作正文返回、硬编码 CNN Selector、无持久状态，以及模型训练数据处理缺乏可重复性与数据契约。

本次结论：现有博客知识库方案的 Source Registry、Site Profile Studio、Targeted Fetch、Immutable Snapshot、Canonical IR、Derived AI Artifact、Runtime Pool 方向正确；在此基础上还应进一步补齐 **AI Recipe Release、Model/Dataset/Runtime Release、模型有效上下文 Attestation、摘要块级 lineage、模型训练数据可重复性、生产摘要质量验证和依赖供应链锁定**。

## 2. 两套执行架构

项目包含 Local 与 Deploy 两套实现。

### 2.1 Local 版本

```text
Streamlit
  -> config.json
  -> Crawl4AI AsyncWebCrawler
  -> Markdown
  -> Markdown 正则抽链接 / CSS Selector 抽正文
  -> 文本压平
  -> 本地 BART 摘要
  -> Streamlit 展示
```

`local/config.json` 配置 CNN 分类页 URL 和 `.container__link--type-article`。`01_Daily_News.py` 读取 tabs，先调用 `get_links_from_homepage()`；用户点击 More details 后再调用 `get_content_from_direct_url()`。

`02_Custom_Fetch.py` 接收任意 URL，点击按钮后同步等待抓取与摘要完成。这一产品形态非常接近生产管理端中的 Targeted Fetch / Probe，但执行模型不能照搬。

### 2.2 Deploy 版本

```text
Streamlit frontend
  -> FastAPI
     -> httpx + BeautifulSoup
     -> CNN CSS Selector
     -> OpenAI summary
```

FastAPI 暴露：

- `GET /scrape`：列表页发现文章；
- `GET /scrape-article`：抓单文章并直接返回摘要。

部署版从 Crawl4AI 降级为 `httpx + BeautifulSoup`，说明抓取引擎只是执行方式，不应成为业务模型本身。生产系统应把 HTTP、Crawl4AI、Playwright、PDF/OCR 都放在统一 Engine Adapter 后面。

## 3. Site Config 与抓取逻辑解耦

项目把分类页 URL、tab 名称和列表 selector 放进 `local/config.json`，避免把每个来源完全写死在页面代码里。这个方向应升级为版本化 Site Profile：

```yaml
site_profile:
  discovery:
    list_urls: []
    list_selectors: []
    article_link_rules: []
  extraction:
    title_rules: []
    body_rules: []
    author_rules: []
    date_rules: []
    remove_rules: []
  routing:
    http_first: true
    browser_escalation_rules: []
  quality:
    expected_selectors: []
    min_text_chars: 300
```

与 demo 配置不同，生产 Profile 必须有 JSON Schema/Pydantic 校验、Draft/Release、Golden URL、Snapshot replay、diff、审批、回滚和审计；配置文件本身不能直接成为运行真相源。

## 4. Crawl4AI 生命周期与缓存语义

`local/utils/scrapper.py` 每次调用都会：

1. 创建 `BrowserConfig(headless=True)`；
2. 创建 `CrawlerRunConfig(...)`；
3. 进入新的 `AsyncWebCrawler` 上下文；
4. 单 URL 执行 `crawler.arun()`；
5. 退出上下文并销毁 Runtime。

对交互式 demo 足够，但百万 URL 场景会重复支付 Chromium 启动、Context 初始化、JS Runtime 初始化等固定成本。

生产实现应为：

```text
Browser Worker Process
  -> long-lived Browser Runtime
     -> Context Pool
        -> Page lease
```

Site/Profile 决定 Context 隔离级别；任务结束释放 lease，不销毁整个浏览器。

项目还把 `CacheMode.BYPASS` 作为常用设置。生产系统不能把“总是重新抓”当作最新性保证，而应拆成：

- Feed/Sitemap/API Change Signal；
- ETag / Last-Modified；
- conditional request；
- immutable Fetch Snapshot；
- body hash / Canonical IR hash；
- 浏览器静态资源缓存；
- Snapshot Replay。

## 5. 同步/异步桥接风险

`get_links_from_homepage()` 与 `get_content_from_direct_url()` 对外是同步函数，内部定义 async helper，然后：

- 无 event loop 时 `asyncio.run()`；
- 已有 event loop 时 `asyncio.run_coroutine_threadsafe(..., loop).result()`。

如果调用线程就是该 loop 所在线程，`.result()` 会阻塞线程，而 coroutine 又需要同一 loop 推进，存在死锁或不可预测行为。

生产系统应“async all the way”到 Worker 边界：

```text
Web/API Command
  -> PostgreSQL Run/Task
  -> Outbox/Queue
  -> async Worker
  -> persisted Outcome
  -> SSE/WebSocket/轮询显示状态
```

禁止在 Web 请求线程中直接等待几十秒浏览器、GPU 模型或外部 LLM；短任务也必须经过标准任务链路，只是优先级更高。

## 6. Discovery：从 Markdown 反向解析链接的问题

列表页抓取先由 Crawl4AI 生成 Markdown，再通过正则抽 `[title](url)`。这条链是：

```text
DOM -> Markdown -> Regex -> URL
```

问题在于 DOM 转 Markdown 已经丢掉部分结构，再反向抽链接会损失：

- `href_raw` 与 base URL 语义；
- DOM/CSS 位置；
- `rel`、data-*；
- 同一 URL 在不同区块出现的多重证据；
- list item/container 上下文。

`extract_links_from_markdown()` 还使用 set 去重，会丢失页面原始顺序，而且 set 的输出顺序不应承担业务语义。

生产 Discovery 应保存：

```text
URL Identity
+ 多条 Discovery Evidence
```

Evidence 至少含 parent URL、href raw/resolved、anchor、DOM/CSS position、rel、observed_at、snapshot_id、provider_release。同一 URL 可以唯一，但 Evidence 不应被覆盖。

## 7. 固定 Selector 与漂移

Local 正文抓取把 `.vossi-paragraph` 写死在 `get_content_from_direct_url()`，Deploy 版也把 CNN 列表与正文 Selector 写死。

这会把“某一时刻 CNN DOM”错误地升级为“所有 URL 的正文规则”。对 1000 个技术博客必须将 selector 放入 Site Profile，并对 0 命中进行语义诊断：

- 页面正常但无正文；
- selector drift；
- 页面类型不对；
- challenge/WAF；
- fetch incomplete；
- 规则错误。

因此 Web 管理端需要 selector hit trend、Golden URL smoke test 和 Snapshot shadow replay。

## 8. Canonical 内容不能被 NLP 预处理污染

`merge_in_one_paragraph()` 本质是：

```python
re.sub(r'\s+', ' ', content).strip()
```

然后该结果直接送给摘要模型。

新闻纯文本摘要还能勉强接受，但技术博客中这会破坏：

- heading；
- 段落边界；
- list 层级；
- fenced code block；
- Python/YAML 等缩进；
- Markdown table；
- shell 输出；
- JSON/config；
- quote。

生产知识库必须先形成结构保真的 Canonical IR：

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

Markdown 是 IR 的确定性 Projection。AI 需要纯文本时，生成独立 `AI_INPUT_TEXT`，并保留 `block_id -> text span` 映射；AI Projection 可以有自己的空白规范化，但绝不能反向覆盖 Canonical IR/Markdown。

## 9. BART 摘要链路：map-only 的局限

`local/utils/summarize.py` 在 import 时加载 `pipeline("summarization")` 和 tokenizer。每篇文章：

1. 全文 tokenize，不截断；
2. 每 1000 token 固定切块；
3. 每块 `max_length=100`、`min_length=30`、`do_sample=False`；
4. 每块独立摘要；
5. 直接用空格拼接所有摘要。

优点是确定性强、实现简单；但存在：

- 固定 token 边界可能截断句子/章节；
- 无 heading/block 语义；
- 无 overlap，跨边界事实可能丢失；
- 无 reduce，长文最后仍可能产生很长摘要；
- 各块可能重复事实；
- 代码与正文会混入同一摘要输入；
- 无块级 lineage，不能解释某句摘要来自哪些原文块。

生产摘要应采用：

```text
Accepted Document Version
  -> AI_INPUT_TEXT / block projection
  -> block-aware chunk planner
  -> map artifacts
  -> optional reduce artifact
  -> factual/coverage validation
  -> final Derived Artifact
```

每个 map chunk 保存稳定 `chunk_id`、block range、input hash、model recipe、输出；最终摘要可回溯到原文 block。

## 10. Deploy 版把 summary 当 content：表示层混淆

`deploy/backend/scraper.py` 的 `scrape_direct_cnn_article_content()` 在抓取正文后立即调用 OpenAI，`/scrape-article` 最终把摘要放在 `{ "content": ... }` 中返回。

这会造成严重的 Representation Ambiguity：上游调用者无法从字段名判断这是原文、清洗正文还是摘要。

生产 API 必须显式区分：

```text
raw_snapshot
extraction_candidate
canonical_document
markdown_projection
summary_artifact
```

禁止把 summary 伪装成 `content`，也禁止把 AI 成功当成文章抓取成功的必要条件。

## 11. HTTP 实现的生产缺口

部署版每个请求都新建 `httpx.AsyncClient()`，且缺少完整的 timeout、持久重试、per-host limiter、validator、Snapshot、robots/policy admission、状态语义和漂移诊断。

生产正确形态是：

```text
HTTP Worker
  -> long-lived AsyncClient
  -> per-host concurrency/rate limit
  -> conditional request
  -> redirect/policy validation
  -> immutable response snapshot
  -> Typed FetchOutcome
  -> Extraction Worker
  -> Browser escalation when evidence exists
```

HTTP-first 是生产优化，不是“简化版爬虫”；只有与持久状态、证据、Quality Gate、升级链结合才成立。

## 12. Web 产品形态可直接升级

项目的 Daily News / Custom Fetch 很适合演化成：

### 12.1 Site Profile Studio

管理员编辑 URL、CSS/XPath、metadata、remove rules、fetch route；点击测试后创建 Probe Run，并展示：

- HTTP/browser trace；
- Snapshot；
- selector hit count；
- discovered URLs；
- candidate fields；
- Canonical IR；
- Markdown；
- quality score；
- current release vs draft diff。

### 12.2 Targeted Fetch

任意 URL 输入不能绕过生产主链路：

```text
URL
 -> SSRF/policy validation
 -> Site/Profile resolution
 -> persistent TARGETED_FETCH Task
 -> Fetch Snapshot
 -> Extract
 -> Canonical IR
 -> Quality
 -> Preview/Accept
```

### 12.3 同 Snapshot 的策略比较

CSS/XPath、Readability/Trafilatura、Crawl4AI、LLM extraction 的比较必须消费同一 Fetch Snapshot，不能分别访问源站，否则无法区分“策略差异”和“页面内容变化”。

## 13. 模型训练数据处理存在可重复性与正确性问题

`models/preprocessing/preprocessing.py` 提供了 TF-IDF/cosine 分析、采样和分层拆分思路，但源码里有几处不能忽略的训练数据风险。

### 13.1 `tokenized_highlights` 实际取了 article

代码写的是：

```python
tokenized_df['tokenized_article'] = df['article'].apply(...)
tokenized_df['tokenized_highlights'] = df['article'].apply(...)
```

第二行仍读取 `article`，不是 `highlights`。后续 `highlights_token_count` 因而实际上也是 article token count，按 article/highlights 长度过滤的逻辑失去原意；词汇分层也没有真实观察摘要 target。

这说明模型流水线必须有数据契约测试，而不能只看训练成功：

- 必须断言 source/target 字段不同且 schema 正确；
- 采样前后做字段统计与随机样本审阅；
- preprocessing 单测加入 known fixture；
- 数据转换输出 manifest/hash。

### 13.2 随机采样缺少固定 seed

`df.sample(frac=0.12)` 没有 `random_state`，同一代码重跑会得到不同子集。后续再讨论模型指标时，如果不知道训练样本到底是哪一批，模型 release 不可重现。

生产 Model Release 必须绑定：

```text
dataset_release
source dataset hash
preprocess code commit
random seed
split manifest
training config
model artifact hash
container image digest
metrics
```

### 13.3 “overlap” 指标与命名并不等价

代码把每条样本 vocab 与 `global_vocab` 取交集，再除以全局词表大小。由于样本 vocab 本身就是全局词表子集，结果更接近“该样本词表占全局词表比例”，不是通常意义的 train/validation vocabulary overlap。再叠加 highlights 字段错误，不能把该结果当成可靠的数据分层保证。

## 14. BART 与 LED：声明能力不等于有效运行能力

项目同时有：

- `facebook/bart-large`；
- `allenai/led-base-16384`。

但 `models/finetuning/led.py` 的 preprocessing 仍把输入 `max_length` 设置为 1024。也就是说，模型名声明 16384 长上下文，并不代表训练/推理链真正使用了 16384。

另外 LED 脚本与 BART 基本共用模板，没有显式表达长文模型特有的 global attention 策略。这并不意味着代码一定无法训练，但它证明：**README/模型名称的 capability 不能替代 Runtime Attestation。**

生产 AI Model/Recipe Release 应记录：

- tokenizer release；
- declared context window；
- effective input max tokens；
- truncation policy；
- reserved output/prompt budget；
- model-specific attention/config；
- generation params；
- quantization；
- runtime hardware；
- batch/concurrency；
- warmup/capability test 结果。

长上下文模型升级前必须用 Golden Document 验证“有效输入真的没有被静默截到 1024”。

## 15. 量化实验与真正 Serving 路径是两回事

BART/LED fine-tuning 脚本在训练完成后的评估阶段使用 `BitsAndBytesConfig(load_in_4bit=True, ...)` 重新加载模型，但 Local UI 的 `summarize.py` 是直接从 Hugging Face model id 加载 `pipeline`。

因此“代码库里有 4bit”不等于“Web 摘要服务正在 4bit 运行”。生产系统要把实验代码、模型 Artifact 和 Serving Release 分开：

```text
Model Artifact Release
+ Inference Runtime Release
+ Quantization Release
+ AI Recipe Release
```

Web/Worker 只解析已经发布的 Runtime Release，运行时记录实际 device、dtype、quantization、model hash，避免把设计意图当运行事实。

## 16. 摘要质量：ROUGE/BERTScore 不足以证明生产事实正确

fine-tuning 脚本使用 ROUGE 与 BERTScore，这对带 reference 的离线 benchmark 有价值，但它们主要衡量与参考摘要的词面/语义相似度，不能单独证明生产摘要没有幻觉、没有遗漏关键限制条件、没有改写代码/版本号。

面向技术博客，应把两类评估分开。

离线 Model/Recipe Benchmark：

- ROUGE/BERTScore；
- factuality benchmark；
- 长文 coverage；
- 代码/命令/版本号保真；
- latency、GPU memory、cost。

在线 Derived Artifact Validation：

- 输出非空与长度边界；
- block coverage；
- 摘要句到 block/source span 的证据映射；
- 关键实体/版本号与原文一致性；
- 高风险断言检查；
- model timeout/cost guardrail。

摘要可以失败或进入 REVIEW_REQUIRED，但绝不能使 Accepted Document Version 回滚。

## 17. 依赖与供应链可重复性

`local/requirements.txt` 同时存在：

- 一部分精确 pin；
- 一部分完全不写版本；
- `fastapi/httpx/beautifulsoup4/uvicorn` 等重复条目；
- 文件还带 BOM。

这对 demo 影响有限，但长期抓取平台中同一源码在不同时间安装出不同依赖组合，会导致浏览器行为、HTML 解析、Markdown 输出和模型结果漂移。

生产 release 应要求：

- lockfile；
- container image digest；
- SBOM；
- crawler/browser/model/runtime 的依赖版本；
- Capability Test；
- 旧 release 可回滚；
- Snapshot replay 验证输出差异。

## 18. 对博客知识库方案的最终优化项

结合上述源码，建议最终方案明确落下以下设计：

1. Site Profile Studio：从 `config.json` 升级为 Draft/Release/Golden/Replay/Approval/Rollback。
2. Targeted Fetch：保留任意 URL 调试体验，但永远经过 Admission、Task、Snapshot、Quality、Audit。
3. HTTP/Browser/Model Runtime Pool：所有重型资源长生命周期复用。
4. Canonical IR 与 AI_INPUT_TEXT 完全分离，禁止 NLP 预处理污染知识库真相。
5. Derived AI 使用 block-aware map/reduce，并保存 chunk/block lineage。
6. AI Provider Adapter 同时支持远程 LLM 和本地 HF Model Service，不把模型 import 写进抓取 Worker。
7. 新增 `ai_recipe_release`、`ai_model_release`、`dataset_release`、`runtime_release`。
8. Model Release 绑定数据 manifest、seed、split、preprocess commit、镜像 digest 和 benchmark。
9. Runtime Attestation 记录有效 context、truncation、attention、quantization、device、generation config。
10. AI API 使用显式 Artifact 类型，禁止把摘要字段命名成原始 `content`。
11. 参考指标和生产 factual/coverage validation 分层。
12. 依赖通过 lockfile/image digest/SBOM 固化。
13. Web 中增加同一 Document Version 的 AI Recipe/Model 对比、质量、成本和 lineage 查看。

## 19. 最终判断

Deepnews-Summarizer 最适合作为“小型端到端产品原型与反例集合”阅读，而不是作为可扩容爬虫平台复用。它验证了配置驱动、列表发现/文章抓取分离、Custom Fetch、Web 可视化、可替换摘要模型这些产品方向；同时它也清楚暴露了从 demo 走向 1000 站生产系统时必须补上的持久调度、增量、证据链、结构保真、Runtime 池化、模型 lineage 和可重复模型工程。

对最终博客知识库而言，最重要的边界仍然是：**原始 Snapshot + Canonical IR + Document Version 才是知识真相；Markdown、搜索索引、Embedding、摘要、标签和 Digest 都是可重建 Projection/Derived Artifact。** 在 AI 层还必须进一步做到“模型声明能力不等于有效运行能力”，所有上下文长度、截断、量化、训练数据和评估结果都要绑定到可验证 Release。