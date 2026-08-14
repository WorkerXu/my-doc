# 如何结合 rs-trafilatura 与 Crawl4AI 使用

- 编号：5
- 原文：https://dev.to/murroughfoley/how-to-use-rs-trafilatura-with-crawl4ai-3nfd
- Python 绑定：https://github.com/Murrough-Foley/rs-trafilatura-python
- Rust 核心：https://github.com/Murrough-Foley/rs-trafilatura
- 调研目标：评估 rs-trafilatura 与 Crawl4AI 的集成方式，以及它对“约 1000 个技术博客全量历史采集、Markdown 清洗、增量同步、持续扩站和 Web 管理”方案的价值。

## 1. 结论

rs-trafilatura 最有价值的地方不是“替换 Crawl4AI”，而是提供一个可插拔、页面类型感知、速度较快的正文抽取器。它适合放在平台的 **Extract Worker** 中，通过统一 `ExtractorAdapter` 接口消费已经保存的 HTTP raw HTML 或 Browser rendered DOM，并返回结构化候选结果，再由平台自己的 Quality Gate 决定是否接受、切换备用抽取器、升级 Browser 或进入人工审核。

不建议把 `RsTrafilaturaStrategy` 直接当成整个采集系统的业务核心，也不建议在生产环境硬编码“`extraction_quality < 0.80` 就调用 LLM”。当前源码显示，常规 `extract()` 返回的 `extraction_quality` 在 Rust 核心里是启发式质量分，而 Python 绑定另外暴露了一个需要调用方自行准备 27 个特征的 `predict_quality()` ML API。两者不是同一条执行路径。因此，第三方原始分数应作为一个信号保存，真正的上线门禁必须由平台基于 golden snapshot 校准，并按站点、页面类型、抽取器版本分别统计。

对于本项目，推荐新增：

1. 多抽取器 Adapter 层；
2. `extraction_attempts` 可追溯数据模型；
3. 平台自有 Quality Gate；
4. probing 阶段 shadow extraction 对比；
5. “抽取失败”和“页面未渲染完整”分流，避免所有低质量页面都升级 Browser；
6. 独立、受控的 CPU/线程并发，避免 `asyncio.to_thread()` 把网络异步优势变成线程池拥塞；
7. 抽取器、规则、质量门禁、Serializer 全部版本化，使已保存快照可以离线重抽。

## 2. Crawl4AI Adapter 的实际集成方式

`rs-trafilatura-python` 中的 `python/rs_trafilatura/crawl4ai.py` 实现了 `RsTrafilaturaStrategy`：

- Crawl4AI 可用时继承 `crawl4ai.extraction_strategy.ExtractionStrategy`，从而通过 `CrawlerRunConfig` 的类型检查；
- 初始化时使用 `input_format="html"`，意味着需要完整 HTML，而不是先把网页变成 Markdown 再抽取；
- `extract()` 直接调用 Rust/PyO3 暴露的 `_extract()`；
- `run()` 只取 `sections[0]`，代码注释明确说明它需要完整文档，而不是分块处理；
- `arun()` 使用 `asyncio.to_thread(self.run, ...)` 把同步抽取放入线程执行。

这几个细节对平台设计有直接影响。

### 2.1 Fetch 与 Extract 必须解耦

Crawl4AI 可以负责 Browser capture，但平台不应该要求“每次抽取都重新抓网页”。正确链路是：

```text
HTTP / Browser Capture
 -> 保存不可变 snapshot
 -> Extract Job
 -> RsTrafilaturaAdapter / 其他 Adapter
 -> Quality Gate
 -> Article IR
 -> Markdown Serializer
```

这样更新抽取器、selector 或 Markdown serializer 时，可以直接 replay 已保存的 raw/rendered snapshot，不重新访问 1000 个源站。

### 2.2 不要把 Crawl4AI 的 section/chunk 直接喂给 rs-trafilatura

rs-trafilatura 的正文识别依赖完整 DOM、metadata、JSON-LD、导航密度、页面类型信号以及结构关系。如果先切块，会丢失决定“主正文在哪里”的上下文，导致分类和正文选择退化。因此 Adapter 输入应是完整 raw HTML 或 rendered DOM，chunking 应发生在最终文章内容进入 RAG/索引阶段，而不是正文抽取之前。

### 2.3 `asyncio.to_thread()` 不等于无限可扩展

网页抓取主要是 I/O，而正文抽取包含 HTML 解析、特征提取、分类、清洗、Markdown 转换等 CPU 工作。若一个 Fetch Worker 同时对几千个结果直接 `to_thread()`，默认线程池会成为隐性队列，导致：

- CPU 饱和；
- 线程上下文切换增加；
- 内存中同时保留大量 HTML；
- Fetch 与 Extract 相互拖累；
- 无法独立水平扩容。

因此生产方案应保留独立 Extract Worker，并设置显式 `extract_concurrency` / worker pool；网络 domain limiter 约束源站访问，CPU limiter 约束本地抽取，两者是不同资源维度。

## 3. rs-trafilatura 核心算法

对 Rust 核心 `src/extract.rs` 的代码检查显示，其正文抽取不是单一 CSS selector，而是一条多阶段流水线。

### 3.1 先提取 metadata，再清理 DOM

算法首先解析完整 HTML，并在清理前提取 metadata，来源包括 JSON-LD、meta/OG、DOM fallback 等。这样可避免 `<script type="application/ld+json">` 等结构化信息在清理阶段被移除后无法恢复。

平台也应采用同样原则：metadata candidate 与正文抽取并行产生，并记录 provenance，而不是只保留最终一个字符串值。

### 3.2 三阶段页面类型分类

当前核心代码大致采用：

```text
URL heuristic
 -> HTML signal refinement
 -> ML classifier
 -> page-type extraction profile
```

页面类型覆盖 article、forum、product、collection/category、listing、documentation、service 等。URL heuristic 与 ML 一致时提升置信度；存在分歧时，最终会更多依赖 ML 结果。

价值在于：不同页面不能用同一个“正文”定义。例如论坛评论本身就是正文；listing 可能需要收集重复卡片；documentation 要去掉导航/sidebar；service 页面往往有多个独立 section；产品页可使用 JSON-LD Product 描述兜底。

对博客知识库而言，页面类型分类不应直接决定是否入库，而应作为 extraction profile 与质量判断的一个特征。特别是 archive/listing/tag 页面应进入 Discovery，不应误当 article version。

### 3.3 页面类型专用 profile

源码包含若干针对页面类型的差异处理，例如：

- forum：允许 comments 作为正文；
- service：正文过短时尝试合并多个非重叠 section；
- listing：检测 3 个以上重复 sibling/card 并聚合；
- collection：补充 category/SEO 描述；
- product：DOM 结果很差时尝试 JSON-LD Product.description；
- Discourse：可读取预加载内容；
- article：可比较 JSON-LD `articleBody` 与 DOM 正文。

这说明“通用抽取器 + 页面类型 profile”比“每个站写一个爬虫”更适合持续增加网站。但平台仍需允许 site-level selector 覆盖，因为通用模型无法保证所有主题、语言和历史模板都稳定。

### 3.4 主抽取失败后的候选比较

代码会在正文太短、缺少段落、表格比例异常、单词过少或看起来像导航时触发 fallback。fallback 并不是无条件替换，而会比较候选是否真的更好，例如避免把结果缩短超过一半，并偏向接受显著更完整的候选。

这个思想值得上升为平台级设计：**不要让任意一个抽取器的输出直接成为最终正文，先生成 candidate，再由统一 gate 决策。**

### 3.5 结构化数据兜底

如果 JSON-LD articleBody / Discourse 内容明显优于 DOM 结果，核心会选择结构化正文。例如 DOM 很短、结构化结果明显更长，或 DOM 很像 cookie/导航模板时，会倾向结构化内容。

平台可保留此行为，但需要记录 provenance：最终正文是 DOM、JSON-LD、rendered DOM 还是其它来源。否则未来 diff 时会出现“正文巨大变化但网页实际上没改”的误判。

## 4. Markdown 生成原理与平台边界

rs-trafilatura 的 Rust 核心可在 `output_markdown=True` 时使用 `quick_html2md` 生成 Markdown，并提供 GFM 相关能力，包括链接、图片、表格等。

但在本项目中，不建议让第三方抽取器直接拥有“最终 Markdown 格式”的唯一控制权。更稳妥的是：

```text
Extractor
 -> Content Candidate / Article IR
 -> 平台统一 URL/asset resolution
 -> normalization
 -> 版本化 Markdown Serializer
```

原因：

1. 需要统一 canonical、`<base href>`、相对链接、图片下载后的本地引用；
2. 需要稳定 Front Matter key 顺序；
3. 需要统一 code fence、表格、换行、Unicode 和 heading 规则；
4. 第三方 Markdown 转换库升级可能产生大量无意义版本 churn；
5. 同一篇文章使用不同 extractor 时，应该尽量由同一个 serializer 输出，便于 diff。

因此 `content_markdown` 可以保存为调试 candidate，但正式版本最好基于平台 Article IR/规范化 HTML 统一序列化。

## 5. 质量分数：最重要的源码差异

原文和项目 README 都强调 `extraction_quality`，并给出“低于 0.80 可路由到 LLM fallback”的示例。进一步检查当前源码后，需要区分两个概念。

### 5.1 常规 `extract()` 返回的质量分

Rust `src/extract.rs` 的常规执行路径调用 `compute_extraction_quality_heuristic(...)`，然后把结果写入 `ExtractResult.extraction_quality`。Python 绑定的 `ExtractResult` 直接暴露这个字段。

所以在当前被检查的版本里，常规 `rs_trafilatura.extract()` / `RsTrafilaturaStrategy` 返回的 `extraction_quality` 是核心算法计算的启发式分数。

### 5.2 独立的 ML `predict_quality()`

Python 绑定同时暴露：

```text
predict_quality(features: 27维特征) -> 0..1
```

它调用 `web_page_classifier::predict_quality`，需要调用方自己提供 27 个 post-extraction features。这个 API 并没有在刚才检查的常规 `extract()` 路径里自动调用。

### 5.3 对生产方案的影响

因此不能简单认为：

```text
result.extraction_quality == 已校准的 ML 预测 F1
```

也不能把 `0.80` 当成 1000 个博客的统一固定阈值。更可靠的办法是保留两层分数：

```text
raw_extractor_quality     # 第三方抽取器自己的分数
platform_quality_score    # 平台版本化 Quality Gate 的最终分数
```

平台分数按站点、page_type、模板和 extractor_version 校准。第三方分数发生语义或算法变化时，也不会直接改变生产准入标准。

## 6. 平台级 ExtractorAdapter 设计

建议接口：

```python
class ExtractorAdapter(Protocol):
    async def extract(
        self,
        snapshot: CaptureArtifact,
        rule: ExtractRule,
    ) -> ExtractionCandidate: ...
```

`ExtractionCandidate` 至少包含：

```text
extractor_name
extractor_version
config_hash
snapshot_id
input_kind(http_raw/rendered_dom/archive)
page_type
classification_confidence
raw_quality_score
main_text
normalized_html_or_blocks
metadata_candidates
links
assets
warnings
latency_ms
```

首期可以支持：

1. `SiteSelectorExtractor`：站点已有稳定 selector 时优先；
2. `RsTrafilaturaExtractor`：通用、page-type-aware 的快速路径；
3. `Crawl4AIGenericExtractor`：作为替代通用算法；
4. 其它 Readability/Trafilatura 类实现以后按 Adapter 增加。

第三方组件只返回候选，不更新 `articles.current_version_id`。

## 7. Quality Gate 设计

平台级 Gate 不应只有一个“正文字符数”。建议组合以下信号：

- 正文字数/字符数与段落数量；
- heading/list/code/table 保留情况；
- 导航、cookie、登录、版权、推荐区等 boilerplate 比例；
- 链接密度；
- 与站点 template fingerprint 的相似度；
- 与同站其它正文的异常重复度；
- title 与正文一致性；
- 页面类型与正文结构是否匹配；
- selector 命中情况；
- extractor warnings；
- raw HTML/rendered DOM 中正文信号是否存在；
- 第三方 `raw_quality_score`，仅作为一个 feature；
- golden snapshot 上该 extractor/version 的历史表现。

Gate 输出：

```text
pass
retry_alternate_extractor
need_browser
manual_review
reject_non_article
```

并记录 `reason_codes`。

## 8. 低质量页面的正确路由

原方案中“质量失败 -> Browser fallback”需要细化。Browser 只能解决“源 HTML 不完整/正文依赖 JS”，解决不了“抽取算法选错正文”。统一规则应是：

```text
HTTP raw HTML
 -> Extractor A
 -> Quality Gate
    -> pass: 发布候选
    -> DOM 完整但抽取差: Extractor B / selector profile
    -> CSR shell / DOM 缺失: Browser capture -> 重新抽取
    -> 仍失败: manual review
    -> 可选 LLM 辅助诊断/规则建议
```

这样可显著降低 Browser 成本，避免把 CPU 抽取问题误判成渲染问题。

LLM 不应自动重写正文。若未来使用 LLM，优先用于：

- 判断候选哪个更像正文；
- 生成 selector 草案；
- 解释失败原因；
- 人工 review 助手。

最终正文仍来自可追溯 snapshot + deterministic extractor/serializer。

## 9. probing 阶段的 Shadow Extraction

新增站点时可在 20~200 个 golden URL/snapshot 上同时运行两个通用 extractor：

```text
rs-trafilatura
vs
Crawl4AI generic
vs
site selector（若有）
```

比较：

- 正文覆盖；
- boilerplate；
- 代码块/表格/链接保真；
- page_type；
- metadata；
- 处理耗时；
- Browser 需求率；
- 人工抽样正确率。

最终发布一个 `extraction_profile_version`。生产正常流量只跑主 extractor，质量失败时再跑备用，避免每篇文章永远双跑造成 CPU 翻倍。

抽取器升级也先 shadow replay golden snapshots，再发布新版本，之后通过 snapshot re-extract 批量迁移，不直接在线替换。

## 10. 数据模型建议

新增：

```text
extraction_attempts
- id
- site_id
- snapshot_id
- extractor_name
- extractor_version
- config_hash
- input_kind(http_raw/rendered_dom/archive)
- page_type
- classification_confidence
- raw_quality_score
- platform_quality_score
- quality_gate_version
- status(pass/fail/fallback/review/rejected_non_article)
- reason_codes_json
- object_key_candidate
- warnings_json
- metrics_json
- created_at
```

`article_versions` 增加：

```text
selected_extraction_attempt_id
extraction_profile_version
quality_gate_version
```

这样能回答：某一篇 Markdown 为什么在某天发生变化，是网页变了、Browser 变了、抽取器升级了、selector 改了，还是 serializer 改了。

## 11. Web 管理端建议

增加“Extraction”视图：

- 每次 attempt 的 extractor/version；
- HTTP raw / rendered DOM 输入类型；
- page_type 与 classification confidence；
- raw quality 与 platform quality；
- gate reason；
- 主/备用 candidate diff；
- extractor latency；
- 是否触发 Browser；
- golden snapshot 批测结果；
- extractor profile 发布/回滚。

这样站点模板改版时，可以快速判断是 Discovery、Fetch、Render 还是 Extract 的问题。

## 12. 可观测性建议

按 `site + page_type + extractor_version` 分组统计：

- extract pass rate；
- quality score 分布；
- alternate extractor fallback rate；
- Browser escalation rate 及原因；
- candidate disagreement rate；
- empty/short/boilerplate rate；
- CPU time / latency；
- extractor version 上线前后质量 drift；
- golden regression failure。

特别要对 extractor 版本升级建立 canary/shadow，而不是只看 Python 包安装成功。

## 13. 风险

### 13.1 外部 benchmark 不能直接等价为生产效果

项目 README 给出了 WCXB 的 F1 与速度结果，这些数据可用于初筛，但属于项目自身公布的 benchmark。1000 个目标博客包含不同语言、历史主题、老旧 HTML、代码高密度页面和非常规模板，必须以自己的 golden corpus 为准。

### 13.2 新组件版本漂移

rs-trafilatura-python/Rust 核心属于独立第三方组件。应固定精确版本或 commit、生成 SBOM、保留 rollback，并把 `extractor_version` 写入 attempt/version provenance。

### 13.3 Native extension 部署

PyO3/Rust extension 带来 wheel/平台兼容、镜像构建、CPU 指令集和供应链维护成本。部署前应验证目标 Linux/CPU 架构，必要时在 CI 构建并缓存受控 wheel。

### 13.4 Page type 误分类

分类错误可能让 listing/forum/product profile 选择错误。平台应保存 `page_type` 与置信度，并允许 site rule 强制覆盖；文章准入不能完全依赖第三方分类器。

### 13.5 Markdown 版本 churn

直接使用第三方 Markdown 输出作为最终格式，升级 converter 时可能大量产生无意义 diff。因此推荐统一平台 serializer。

## 14. 最终落地建议

将 rs-trafilatura 作为 **可选通用 Extractor Adapter** 加入现有方案，而不是替换 Crawl4AI 或自研平台控制面：

```text
Discovery
 -> Frontier
 -> HTTP Fetch
 -> immutable raw snapshot
 -> Extract Worker
    -> SiteSelector if configured
    -> RsTrafilatura generic
    -> Quality Gate
       -> alternate extractor
       -> Browser only when DOM missing/CSR
       -> manual review
 -> Article IR
 -> canonical/link/asset normalization
 -> deterministic Markdown Serializer
 -> article version
```

这个设计保留了 rs-trafilatura 的速度、页面类型感知和结构化 fallback 优点，同时把第三方质量分、版本变化、线程并发和 Markdown 格式漂移隔离在 Adapter 边界内，符合 1000 个博客长期运行、持续扩站和可回放重抽的要求。
