# crawlwithagents：网页抓取与元数据分析工具

来源项目：https://github.com/varunsaagar/crawlwithagents

## 1. 项目定位

`crawlwithagents` 是一个体量很小的 Crawl4AI + 本地 LLM 示例项目，目标不是做大规模爬虫平台，而是把单页网页处理拆成三个连续步骤：网页抓取与结构化元数据抽取、元数据清洗、元数据分析。项目 README 将其描述为 Web Scraper and Metadata Analyzer，核心字段是 title、description、keywords。

项目的价值主要不在“如何高并发抓取”，而在两个工程思想：

1. 用显式 Schema 约束 LLM 的结构化输出，而不是直接接收自由文本。
2. 将“抓取/抽取”“清洗”“分析”拆成职责不同的工具，而不是把所有逻辑塞进一个爬虫函数。

这两个思想可以用于博客知识库方案中的 Metadata Candidate、Metadata Quality 和 Web 管理，但不能直接照搬它的运行架构。

## 2. 源码结构与实际执行链

仓库主要只有四类文件：

- `crawler.py`：三个 Tool 的实现以及本地串行测试入口。
- `Agents.yaml`：声明 web_scraper、data_cleaner、data_analyzer 三个角色及各自任务。
- `README.MD`：安装和命令行说明。
- `requirments.txt`：冻结的运行环境依赖。

`crawler.py` 的实际调用链是：

```text
URL
 -> WebScraperTool
 -> Crawl4AI WebCrawler
 -> LLMExtractionStrategy
 -> PageMetadata JSON Schema
 -> JSON
 -> DataCleanerTool
 -> MetadataAnalyzerTool
 -> 分析结果
```

`Agents.yaml` 则从概念上把同一流程拆成 Web Scraper、Data Cleaner、Data Analyzer 三个 Agent/角色。不过当前 `crawler.py` 的 `__main__` 仍然是普通同步串行调用，并没有展示一个真正的 durable multi-agent orchestration。因此它的“多 Agent”更接近职责建模，而不是任务基础设施。

## 3. WebScraperTool 的实现原理

### 3.1 Pydantic Schema 约束

项目定义 `PageMetadata`：

- `title: str`
- `description: str`
- `keywords: list`

随后把 Pydantic 生成的 JSON Schema 传给 Crawl4AI 的 LLM extraction strategy。这一点很重要：模型不是只收到“请给我一个 JSON”，而是收到一个机器可描述的字段契约。

原理上，这可以把 LLM 从“自由文本生成器”降级为“候选结构化抽取器”。对博客知识库而言，类似机制可用于缺失字段、非常规模板或离线规则生成，但必须经过后续校验和 provenance 记录。

### 3.2 本地 Ollama 模型

项目把 provider 指向 Ollama 的 `qwen2:1.5b`，说明它希望在本地完成结构化抽取。其优势是：

- 无外部 API 依赖；
- 单次调用成本可控；
- 敏感抓取结果不必发送到第三方模型；
- 适合作为低优先级 fallback 或离线分析能力。

但 1.5B 级小模型不应该直接决定 canonical metadata。它更适合生成候选值或辅助分析结果，因为 title/description/keywords 中至少 keywords 很容易被模型“补全”成页面并没有明确给出的词。

### 3.3 当前实现的数据流

当前实现会抓取页面、执行 LLM extraction、读取 `result.extracted_content`，再执行 JSON 解析，最后取结果数组的第一个元素。

这里有几个隐含假设：

- 抽取结果一定是合法 JSON；
- 返回值一定是数组；
- 数组至少包含一个元素；
- 第一个元素就是唯一正确结果；
- Schema 约束足以替代应用层再校验。

生产系统不能依赖这些假设。必须把“模型输出”“JSON 语法正确”“Schema validation 通过”“字段质量合格”“候选是否允许参与最终值解析”拆成不同状态。

## 4. DataCleanerTool 的实现原理

项目的清洗逻辑非常简单：

- title/description 去首尾空白；
- keywords 去首尾空白并转小写。

这展示的是“抽取结果不要直接消费”的思路，但离生产要求还很远。博客知识库中的元数据清洗至少需要：

- Unicode normalization；
- HTML entity 解码；
- 空白和不可见字符归一化；
- URL/base/canonical resolution；
- 时间字段时区与格式规范化；
- author/tag/keyword 去重与顺序规则；
- language/script 检查；
- field-level provenance 保留原值与 normalized 值；
- 规则版本可追踪。

尤其不能把“清洗”写成破坏性覆盖。应保留 raw candidate 和 normalized candidate，最终值由 resolver 决定。

## 5. MetadataAnalyzerTool 的实现原理

项目的分析结果包括：

- title length；
- description length；
- keyword count；
- 前 5 个 keywords。

这部分的意义不是这些指标本身，而是表明“元数据质量/策略分析”可以与 canonical 抽取解耦。

对于 1000 个博客站点，这个思想很有价值。可以做成 Metadata Quality/Audit 能力，持续统计：

- title/description 缺失率；
- JSON-LD/OpenGraph/meta/DOM 多来源一致率；
- 同站标题模板重复率；
- description 与正文摘要相似度；
- keyword/tag 重复和异常膨胀；
- author/published_at 覆盖率；
- selector/contract 命中率；
- metadata field drift；
- 站点模板变更导致的批量异常。

这些指标可以直接驱动 Web 管理端告警、Profile/Contract shadow replay 和人工审核。

## 6. Agents.yaml 的工程含义

`Agents.yaml` 把任务拆成三类职责：

```text
抓取者：负责获取结构化元数据
清洗者：负责规范化
分析者：负责质量和趋势分析
```

对于博客知识库，不建议把这三步强行实现成三个 LLM Agent。正确的吸收方式是把“职责边界”映射到 deterministic pipeline stage：

```text
Fetch/Capture
 -> Metadata Candidate Extraction
 -> Metadata Normalization/Resolution
 -> Metadata Quality Audit
 -> optional LLM Insight
```

这样保留了解耦性，又不会引入 Agent 对话、提示词漂移、不可控重试和状态难以恢复的问题。

## 7. 项目中的关键问题

### 7.1 单 URL、同步、无 Durable State

项目每次运行只处理单 URL，没有 Frontier、lease、checkpoint、增量状态、重试状态、站点级限流、outbox，也没有任务恢复能力。因此不能直接扩展到 1000 站点长期运行。

### 7.2 每次工具调用都初始化并 warmup crawler

如果把这种模式放到海量 URL，会重复产生初始化成本。生产方案应长期复用 HTTP client / Browser process，并把站点会话隔离与进程复用分开设计。

### 7.3 `bypass_cache=True`

示例为了每次直接抓源站而绕过 cache，但生产增量同步应使用 ETag、Last-Modified、304 和 raw body hash。缓存不是业务真相源，但 conditional request 是减少源站压力的关键机制。

### 7.4 LLM 直接参与基础 metadata 抽取

对 title、OpenGraph description、JSON-LD author/date 这类高度结构化字段，LLM 通常不是首选。规则/CSS/XPath/JSON-LD 具有更低成本、更高确定性和更容易审计的优势。

当前 Crawl4AI 官方文档也强调：结构化页面优先使用 CSS/XPath 等非 LLM strategy；LLM extraction 更适合复杂、非结构化或需要语义理解的场景。

### 7.5 旧 Crawl4AI API

该仓库创建于 2024 年，依赖固定到一个当时的 Crawl4AI git commit。现在的 Crawl4AI API 已采用 `AsyncWebCrawler + CrawlerRunConfig`，LLM 配置也应通过 `LLMConfig` 传递。官方 v0.5.0 之后已移除 `LLMExtractionStrategy` 中直接传 `provider`、`api_token` 等旧参数的方式。

因此该项目应被视为架构思想参考，而不是可以直接复制的最新实现。

### 7.6 `unicode_escape` 二次解码风险

源码对已是 Python Unicode 字符串的 extracted content 再进行 `unicode_escape` 解码。对中文、emoji、反斜杠、代码内容等存在误解码或字符破坏风险。

正确做法是：

- 网络层保留原始 bytes；
- 按 HTTP/HTML charset 规则统一解码；
- JSON parser 处理 JSON escape；
- 不对正常 Unicode 文本做二次 `unicode_escape`。

### 7.7 JSON 解析后没有 Pydantic 二次校验

虽然模型拿到了 JSON Schema，但应用端最终只是 `json.loads()`。生产实现还应执行 Pydantic/JSON Schema validation，并把校验失败作为结构化 reason code。

### 7.8 Cleaner 和 Analyzer 过于示例化

`top_keywords` 实际只是取 keywords 的前五项，并不是按频率、权重或相关度计算的“top”。这说明分析结果必须有明确语义，不能让字段名看起来比算法本身更强。

### 7.9 依赖面过大

`requirments.txt` 包含大量与该小工具无直接关系的 AI/RAG/云组件依赖，更像完整环境 freeze，而不是最小依赖清单。生产服务应按 Worker 类型拆依赖和镜像，Browser、Extract、LLM Enrichment、Index 不应共享一个超大 Python 环境。

## 8. 对博客知识库技术方案的可吸收设计

### 8.1 新增 Structured Metadata Candidate Adapter

在现有 Metadata Candidate Resolver 中增加一种低信任度 Adapter：

```text
JSON-LD / OpenGraph / HTML meta / Site Selector
 -> deterministic metadata candidates
 -> 如果 required field 缺失、冲突或进入离线诊断
 -> Structured Metadata Candidate Adapter
 -> JSON Schema/Pydantic validation
 -> candidate only
 -> Metadata Resolver
```

要求：

- 默认不对每篇文章调用 LLM；
- 只在规则命中失败、字段冲突、离线 shadow replay 或人工触发时运行；
- 模型输出只能成为 candidate，不能越过 resolver 直接写 canonical metadata；
- 记录 model/provider、prompt version、schema version、input hash、output hash、token/cost、validation error；
- 对相同 immutable snapshot 可无 refetch 重放。

### 8.2 新增 Metadata Quality Audit

将项目的 metadata analyzer 思路升级成生产能力：

```text
article_version / metadata candidates
 -> deterministic metadata metrics
 -> field conflict/drift detection
 -> optional LLM semantic insight
 -> Web console / alert / rule workbench
```

确定性指标优先，LLM insight 只作为诊断附加信息。

建议至少有：

- required field coverage；
- field source coverage；
- multi-source agreement；
- title/description length distribution；
- title boilerplate/template ratio；
- repeated title/description ratio；
- keyword/tag explosion；
- author/date anomaly；
- language mismatch；
- metadata vs article body consistency；
- site/profile/contract drift score。

### 8.3 数据模型

建议新增或细化：

```text
metadata_extraction_attempts
- id
- site_id
- snapshot_id
- extraction_attempt_id nullable
- strategy_type(selector/jsonld/opengraph/meta/llm_schema/...)
- schema_version
- rule_or_contract_version
- model_provider nullable
- model_name nullable
- prompt_version nullable
- input_hash
- raw_output_ref
- validated_output_json
- status
- reason_codes_json
- token_usage nullable
- estimated_cost nullable
- created_at

metadata_analysis_results
- id
- article_version_id
- analyzer_version
- deterministic_metrics_json
- semantic_insight_ref nullable
- status
- reason_codes_json
- created_at
```

### 8.4 Web 管理增强

新增 Metadata Quality 页面：

- 每站字段覆盖率和来源分布；
- metadata candidate 冲突；
- 原始值/normalized 值/最终值/provenance 对比；
- title/description 长度分布；
- drift 时间线；
- LLM schema candidate 的 validation/pass/fail/cost；
- 从异常样本一键进入 Contract/Rule Workbench；
- 对 immutable snapshot 发起“只重跑 metadata，不 refetch”。

### 8.5 LLM 使用边界

从该项目可以吸收“Schema 约束 + 本地模型”，但必须加上生产边界：

1. deterministic first；
2. LLM fallback/offline only；
3. schema validation；
4. candidate only；
5. provenance/version/cost 全记录；
6. 失败不阻塞 canonical 正文发布；
7. 模型变化通过 shadow generation 验证，不在线热替换；
8. 本地 Ollama 只是 Provider Adapter，不进入业务状态模型。

## 9. 适配当前 Crawl4AI 的实现建议

如果实现 Structured Metadata Candidate Adapter，不应复用该仓库的旧 API。当前实现建议：

```text
AsyncWebCrawler
 + BrowserConfig（仅需要 Browser 时）
 + CrawlerRunConfig
 + LLMExtractionStrategy(llm_config=LLMConfig(...))
 + Pydantic model_json_schema()
```

但对于稳定平台，优先使用：

```text
JsonCssExtractionStrategy / JsonXPathExtractionStrategy
```

LLM 可以用于一次性生成 selector/schema 候选，经过 golden snapshot 验证后固化为确定性规则。这比“每次抓文章都调用 LLM”更适合数百万 URL 的规模。

## 10. 最终结论

`crawlwithagents` 不适合作为 1000 站点博客知识库的爬虫底座，它缺少 durable frontier、增量同步、站点公平性、限流、版本化、可重放、Web 管理和生产安全边界。

但它提供了一个值得吸收的局部模式：

```text
Schema-constrained extraction
 -> explicit cleaning
 -> metadata analysis
```

最终应把这个模式改造成：

```text
确定性 metadata source
 -> versioned candidate
 -> optional schema-constrained LLM fallback
 -> validation
 -> resolver
 -> canonical metadata
 -> deterministic metadata audit
 -> optional semantic insight
```

这样既利用了 Schema + LLM 对复杂页面的适应能力，又不让模型进入 canonical 真相链路，并且能够在 1000+ 站点规模下实现可追溯、可重放、可观测和成本可控的元数据治理。

## 参考

- 项目：https://github.com/varunsaagar/crawlwithagents
- Crawl4AI LLM extraction：https://docs.crawl4ai.com/extraction/llm-strategies/
- Crawl4AI non-LLM extraction：https://docs.crawl4ai.com/extraction/no-llm-strategies/
- Crawl4AI v0.5.0 breaking changes：https://docs.crawl4ai.com/blog/releases/0.5.0/
