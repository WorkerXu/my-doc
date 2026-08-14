# 深度新闻摘要器（Deepnews-Summarizer）

## 1. 调研对象与结论

- 项目：https://github.com/NhanPhamThanh-IT/Deepnews-Summarizer
- 调研基线：`main`，commit `e5782b4212b59e63a5996546168016504a658130`
- 仓库状态：已归档，MIT License
- 核心代码：`local/utils/scrapper.py`、`local/utils/summarize.py`、`local/utils/text_preprocessing.py`、`local/config.json`
- Web 交互：`local/pages/01_Daily_News.py`、`local/pages/02_Custom_Fetch.py`
- 部署版：`deploy/backend/main.py`、`deploy/backend/scraper.py`、`deploy/frontend/*`
- 模型实验：`models/finetuning/bart.py` 等

Deepnews-Summarizer 是一个“小型新闻抓取 + 摘要 + Web 展示”项目。它不是面向 1000 个技术博客的历史全量抓取系统，也没有持久化 frontier、增量同步、断点续跑、覆盖率证明和多站点调度能力，但其代码非常清晰地展示了三类值得吸收的产品/工程思想：

1. **站点配置与抓取逻辑解耦**：列表页 URL、CSS Selector 放在配置中，而不是全部硬编码在 UI；
2. **列表发现与单文章抓取分离**：先发现文章链接，再对具体 URL 做内容抓取；
3. **“自定义 URL 即时抓取”是很有价值的 Web 运维能力**：它本质上可以演化为生产系统中的 Targeted Fetch / Probe / Extraction Preview；
4. **摘要应作为下游派生能力，而不是替代原文**：项目本身把抓取和摘要耦合在一起，这正好暴露出生产知识库必须拆层的原因。

结论：不采用该项目作为生产底座，但应把它的“可配置站点抓取 + 单 URL 诊断 + Web 即时预览”产品形态升级为博客知识库的 **Site Profile 工作台、Targeted Fetch、Extraction Preview 和 Derived AI Projection**；同时必须避免它在正文结构破坏、浏览器生命周期、同步/异步桥接、硬编码 CSS、无持久状态和摘要替代原文等方面的设计缺陷。

## 2. 实际架构与执行链路

项目存在两套实现。

### 2.1 Local 版本

Local 版本是 Streamlit 单进程应用：

```text
Streamlit
  -> config.json
  -> Crawl4AI AsyncWebCrawler
  -> Markdown
  -> 正则抽链接 / CSS Selector 选正文
  -> 文本压平
  -> BART 摘要
  -> Streamlit 展示
```

`local/config.json` 中配置 CNN 分类页和 `.container__link--type-article`。`01_Daily_News.py` 按 tab 读取这些配置，调用 `get_links_from_homepage()`；用户点击某条新闻的 More details 后，再调用 `get_content_from_direct_url()`。

`02_Custom_Fetch.py` 则直接提供 URL 输入框，点击后对单 URL 抓取与摘要。这一页面非常接近生产运维系统中的“URL 调试器”。

### 2.2 Deploy 版本

部署版把前后端拆开：

```text
Streamlit frontend
  -> FastAPI
     -> httpx + BeautifulSoup
     -> CNN selector
     -> OpenAI 摘要
```

FastAPI 暴露：

- `GET /scrape`：发现列表页文章；
- `GET /scrape-article`：抓单文章并返回摘要。

这种拆分说明作者已经意识到 UI 与抓取逻辑应该隔离，但部署版为了降低运行成本，反而从 Crawl4AI 退化成 CNN 专用的 `httpx + BeautifulSoup`，进一步说明“抓取引擎”不应该直接等价于“业务系统”。生产系统必须把 Fetch Engine 抽象成 Adapter。

## 3. Crawl4AI 使用方式分析

`local/utils/scrapper.py` 的列表页抓取：

1. 每次调用创建 `BrowserConfig(headless=True)`；
2. 创建 `CrawlerRunConfig(cache_mode=CacheMode.BYPASS, css_selector=...)`；
3. `async with AsyncWebCrawler(...)`；
4. 对单 URL 执行 `crawler.arun()`；
5. 从 Crawl4AI 返回 Markdown；
6. 用 Markdown 正则抽链接。

正文抓取类似，但 CSS Selector 被硬编码为 `.vossi-paragraph`，并排除链接、外图、社交链接，然后直接把 Markdown 压成一个段落再做摘要。

### 3.1 值得吸收的点

**CSS Selector 属于站点 Profile，而不是爬虫核心代码。**

列表页 selector 已经进入配置，这个方向是正确的。对于 1000 个技术博客，应扩展成版本化 Site Profile：

```yaml
site_profile:
  discovery:
    list_selectors: []
    article_link_rules: []
  extraction:
    title_selectors: []
    body_selectors: []
    remove_selectors: []
    metadata_rules: []
  routing:
    prefer_http: true
    browser_fallback: true
```

并且每条规则需要 release/version，不能直接覆盖生产配置。

### 3.2 不能照搬的点：每次新建 Browser

当前每次函数调用都新建 `AsyncWebCrawler`。对单机 demo 没问题，但对百万级 URL 会带来浏览器启动、Context 初始化、连接握手、JS runtime 初始化、缓存丢失等固定成本。

生产系统应建立长生命周期 Browser Runtime Pool：

```text
Browser Worker Process
  -> Browser Runtime
     -> Context Pool
        -> Page lease
```

Site/Profile 决定 Context 隔离级别；任务完成只释放 Page/Context lease，不销毁整个 runtime。这样才能把浏览器抓取成本控制在可接受范围。

### 3.3 `CacheMode.BYPASS` 不应成为全局默认

项目为了展示“最新结果”每次绕过缓存。生产知识库如果全量历史与增量都无条件 bypass，会浪费带宽和站点配额。

正确做法是把缓存语义拆成：

- HTTP validator：ETag / Last-Modified；
- Snapshot content hash；
- Signal Plane 的 sitemap/feed 更新时间；
- Worker 本地连接缓存；
- 浏览器资源缓存策略；
- 重处理时直接 replay Snapshot，不访问源站。

## 4. 同步/异步桥接的隐患

`get_links_from_homepage()` 和 `get_content_from_direct_url()` 是同步函数，内部定义 async helper，然后：

- 没运行 loop 时：`asyncio.run()`；
- 已有运行 loop 时：`asyncio.run_coroutine_threadsafe(..., loop).result()`。

这个写法看起来兼容同步 UI，但存在典型风险：如果调用线程就是该 event loop 所在线程，提交 coroutine 后立刻 `.result()` 会阻塞调用线程，而 loop 又需要该线程推进，可能造成死锁或不可预测行为。

生产系统不能在业务层到处做同步/异步桥接，而应明确层次：

- API 层只写持久任务；
- Worker 天然 async；
- UI 查询 Task/Run 状态；
- Targeted Fetch 可以通过短任务 + SSE/WebSocket/轮询返回进度；
- 禁止 Web 请求线程直接持有几十秒浏览器抓取。

这也是为什么“Web 管理抓取”不能实现为 UI 点击后直接调用 crawler。

## 5. URL 发现机制分析

项目通过 `css_selector` 将列表页裁剪为相关节点，再从生成 Markdown 中用正则解析 `[title](url)`。

优点是简单、跨 HTML 结构有一定容错；问题有四类。

### 5.1 Markdown 不是 Discovery 的最佳结构源

HTML -> Markdown 已经发生一次有损转换，再从 Markdown 反向解析链接，可能丢失：

- DOM 层级；
- rel/canonical 属性；
- data-* 属性；
- 相对 URL 的原始 base 语义；
- 页面上同 URL 多锚文本的位置证据。

生产 Discovery 应优先保存结构化 link evidence：

```text
source_page
anchor_text
href_raw
href_resolved
rel
css_path/dom_path
container_hint
position
observed_at
snapshot_id
```

Markdown 可以用于展示，但不应成为发现真相层。

### 5.2 set 去重丢失顺序与证据

`extract_links_from_markdown()` 使用 set 去重。它会丢失页面顺序，也无法保留同 URL 在多个区块出现的证据。

生产系统应把“URL Identity”和“Discovery Evidence”分开：URL 可以唯一，但 Evidence 可以多条。

### 5.3 CSS Selector 失败需要可观测

如果 selector 因站点改版不再命中，demo 只显示“No links found”。生产环境必须区分：

- 页面真的无文章；
- selector 0 命中；
- selector 命中但无合法 URL；
- 页面被 challenge；
- 页面结构显著漂移；
- 解析器异常。

因此 Site Profile 需要 Golden URL 和自动 smoke test。

## 6. 正文抽取与结构破坏

这是该项目对知识库方案最重要的反例。

`get_content_from_direct_url()` 抓取 Markdown 后调用：

```python
merge_in_one_paragraph(result.markdown)
```

而该函数本质是：

```python
re.sub(r'\s+', ' ', content).strip()
```

这会把所有换行与连续空白折叠成一个空格。对新闻摘要输入可能勉强可用，但对技术博客是不可接受的，因为它会破坏：

- Markdown 标题层级；
- 列表；
- 代码缩进；
- fenced code block 内格式；
- 表格；
- 引用；
- 命令行输出；
- YAML/JSON/config 示例；
- 段落边界。

技术博客知识库的核心原则必须是：**Canonical IR 先结构保真，任何 NLP/LLM 输入都从 Canonical IR 生成独立的 text projection，不能反过来污染 Canonical Document。**

建议内部表示：

```json
{
  "blocks": [
    {"type": "heading", "level": 2, "text": "..."},
    {"type": "paragraph", "text": "..."},
    {"type": "code", "language": "python", "text": "..."},
    {"type": "list", "items": []},
    {"type": "table", "rows": []}
  ]
}
```

Markdown 只是从 IR 确定性渲染出来的 Projection。

## 7. 摘要实现与知识库的边界

`local/utils/summarize.py` 在模块 import 时加载 fine-tuned BART pipeline 和 tokenizer；每篇文章：

1. tokenizer 对全文编码且不截断；
2. 按 1000 token 切块；
3. 每块 `max_length=100, min_length=30` 摘要；
4. 最后直接用空格拼接各块摘要。

这种做法的原理是典型 map-only chunk summarization，优点是实现简单；缺点是：

- chunk 边界与语义段落无关；
- 可能把代码和正文混在一个 chunk；
- 不做第二阶段 reduce，长文最终摘要可能仍很长；
- 跨 chunk 事实无法去重；
- 模型在进程 import 时加载，Web 扩容会重复占用大量内存/GPU；
- 摘要结果没有 model/prompt/version/input-hash lineage。

部署版则直接在抓取函数里调用 GPT 摘要，并把摘要当成 `/scrape-article` 的 content 返回。这进一步证明需要将**原文抓取成功**与**AI 摘要成功**拆成两个业务状态。

生产方案应采用：

```text
Accepted Document Version
  -> AI Task
     -> block-aware chunking
     -> map summary
     -> optional reduce
     -> validation
     -> Derived Artifact
```

Derived Artifact 至少记录：

- document_version_id；
- canonical_ir_hash；
- model/provider；
- prompt release；
- chunker release；
- generation params；
- input/output token；
- cost；
- latency；
- status；
- output snapshot。

摘要失败不能影响原文入库。

## 8. Fine-tuning 代码的启示

`models/finetuning/bart.py` 以 `facebook/bart-large` 为基础，训练数据字段为 `article/highlights`，输入截断到 1024 token，target 最长 512 token，使用 Hugging Face Trainer，随后用 ROUGE/BERTScore 做评估。

它对知识库平台的启示不是“必须自己训练 BART”，而是：模型能力应该作为版本化 AI Provider 接入，而不是写死在采集服务里。

如果以后确实需要私有摘要模型，可把：

- 模型权重版本；
- 训练数据集版本；
- 评估集；
- ROUGE/BERTScore/事实一致性指标；
- 推理镜像 digest；

纳入 AI Model Release。只有通过 golden corpus 才能切换生产默认模型。

## 9. 部署版的 HTTP 抓取问题

`deploy/backend/scraper.py` 使用 `httpx.AsyncClient()` 直接抓 CNN，然后 BeautifulSoup + CSS Selector。

它暴露了几个生产问题：

1. 每次调用都创建新的 AsyncClient，不能复用连接池；
2. 没有显式 timeout/profile；
3. 没有持久重试和 backoff；
4. 没有 robots/policy admission；
5. 没有 ETag/Last-Modified 条件请求；
6. 没有 Snapshot 保存；
7. CSS Selector 与 CNN 强耦合；
8. 抓取与 OpenAI 摘要在同一函数内；
9. 没有记录正文为空是 selector 漂移、页面异常还是正常空文档。

对于目标系统，应反过来把“简单 HTTP 抓取”作为第一层，而不是把它当成 demo 的简化版：

```text
HTTP Fetch Worker
  -> reusable AsyncClient
  -> per-host limiter
  -> validator/cache
  -> immutable response snapshot
  -> extraction worker
  -> browser escalation only when needed
```

也就是说，HTTP-first 是生产优化，但前提是有完整的状态、证据和降级/升级链路。

## 10. Web 管理功能值得吸收的部分

项目的 `Daily News` 与 `Custom Fetch` 虽然只是 Streamlit 页面，但产品形态非常适合升级为运维工具。

### 10.1 Site Profile Preview

管理员编辑：

- 列表页 URL；
- CSS/XPath Selector；
- 正文 selector；
- remove selector；
- canonical/author/date 规则；
- HTTP/Browser route。

点击“测试”后只创建一条短生命周期 Probe Run，并展示：

- 网络状态；
- raw snapshot；
- selector 命中数；
- discovered links；
- extraction candidate；
- Canonical IR；
- Markdown preview；
- quality score。

测试通过后才能发布新的 Site Profile Release。

### 10.2 Targeted Fetch

“输入任意 URL 抓取”应保留，但不能绕过生产规则。目标 URL 必须经过：

```text
SSRF validation
 -> Site resolution
 -> robots/policy admission
 -> route resolution
 -> persistent Task
 -> standard Fetch/Extract/Quality pipeline
```

Web 页面实时展示阶段进度，最终允许“仅诊断”或“接受入库”。

### 10.3 Extraction Diff

对于站点改版，可让管理员用同一 Snapshot 并排比较：

- 当前 Profile；
- 草稿 Profile；
- Readability/Trafilatura；
- Crawl4AI markdown；
- CSS/XPath strategy。

比较必须共享一个网络 Snapshot，否则无法判断差异来自规则还是源站内容变化。

## 11. 对现有博客知识库方案的具体优化

基于该项目，方案应明确加入以下能力。

### 11.1 Site Profile Studio

把“config.json + Streamlit”升级为生产级 Profile Studio：

- Draft/Release 模型；
- selector editor；
- Golden URL；
- Probe；
- side-by-side preview；
- shadow replay；
- one-click rollback；
- release audit。

### 11.2 Targeted Fetch 不是特殊旁路

Custom Fetch 很实用，但必须复用生产主链路。所有人工 URL 诊断都产生 `TARGETED_FETCH` Run，拥有同样的 Snapshot、Outcome、Quality、Lineage。

### 11.3 原文与 AI 派生完全隔离

不能像 Deepnews-Summarizer 那样“抓取后直接返回摘要”。知识库 Source of Truth 必须保存原始 Snapshot + Canonical IR + Markdown；摘要、标签、聚类、Newsletter 都只是 Derived Artifact。

### 11.4 文本投影专供模型使用

为了摘要可以把段落拼成纯文本，但这个转换必须是独立 `AI_INPUT_TEXT` Projection，并携带来源 block id。绝不能用 `\s+` 全局压平后覆盖 Markdown/IR。

### 11.5 Runtime Pool

HTTP Client、Browser、模型服务都必须做长生命周期池化：

- HTTP connection pool；
- Browser runtime/context pool；
- local model inference service；
- external LLM provider client pool。

避免每 URL 初始化重型资源。

## 12. 适配 1000 博客的推荐落地形态

```text
Source Registry
  -> Site Profile Release
  -> Discovery Provider
  -> URL Candidate + Evidence
  -> Admission
  -> Durable Frontier
  -> HTTP Fetch
       -> Browser fallback
  -> Immutable Snapshot
  -> Multi-strategy Extraction
  -> Canonical IR
  -> Quality Gate
  -> Document Identity/Version
  -> Markdown Projection
  -> Search/Embedding Projection
  -> AI Summary/Tag/Digest Projection
```

其中 Web 管理端提供：

```text
Site Profile Studio
Targeted Fetch
Extraction Preview
Snapshot Replay
Version Diff
Coverage Dashboard
Run/Task Control
Quality Review
AI Artifact Viewer
```

这套形态保留了 Deepnews-Summarizer 最有价值的“配置驱动 + 即时抓取 + Web 可视化”体验，同时彻底补齐生产系统需要的持久状态、可扩展调度、增量同步、原始证据、结构保真和可重放能力。

## 13. 最终判断

Deepnews-Summarizer 更像一个教学型端到端样例，而不是可直接扩容的爬虫平台。它最值得学习的是产品边界，而不是现有实现：

- 配置站点，而不是为每个站点复制代码；
- 列表发现和单文章抓取分离；
- Web 上允许管理员即时输入 URL 验证；
- 摘要能力与抓取结果形成可视化闭环。

生产知识库应把这些体验建立在 durable task、snapshot、versioned profile、canonical IR、quality gate 和 derived artifact 之上。尤其要明确：**技术博客的最终知识资产必须是结构保真的原文版本，摘要只是可丢弃、可重建、可换模型的派生视图。**
