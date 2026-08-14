# crawlwithagents：网页抓取与元数据分析工具

来源项目：https://github.com/varunsaagar/crawlwithagents

## 1. 调研结论

`crawlwithagents` 是一个非常小的 Crawl4AI + Ollama + PraisonAI Tools 示例项目，核心不是大规模爬虫，而是把单页元数据处理拆成三个明确职责：

```text
WebScraperTool
 -> DataCleanerTool
 -> MetadataAnalyzerTool
```

项目真正值得吸收的不是它的运行时架构，而是三个局部思想：

1. **Schema-constrained extraction**：用 Pydantic/JSON Schema 约束 LLM 的结构化抽取结果。
2. **显式 Normalization**：抽取结果不直接消费，而是进入独立清洗/规范化阶段。
3. **Quality/Audit 与 canonical 解耦**：分析指标是派生能力，不应该反向修改正式元数据。

它不能直接作为 1000 个技术博客长期同步的底座，因为没有 Durable Frontier、站点公平调度、增量 checkpoint、条件请求、版本化、幂等、可重放状态、Web 控制面和生产级安全边界。

本次对现有博客知识库方案的主要增量优化是：**把原来隐含在 metadata candidate 字段中的 normalized value 升级为显式、版本化、可重放的 Metadata Normalization 阶段；同时为 LLM fallback 增加输入投影类型和版本，避免模型升级或输入选择变化时无法解释结果。**

## 2. 仓库状态与源码结构

项目仓库创建于 2024 年 7 月，代码提交集中在 2024-07-07 至 2024-07-08；当前主分支最后一次提交仍是 2024-07-08 的 README 更新。仓库很小，主要文件如下：

- `crawler.py`：核心实现，定义三个 Tool，并在 `__main__` 中串行调用。
- `Agents.yaml`：声明 Web Scraper、Data Cleaner、Data Analyzer 三个角色和任务。
- `README.MD`：安装、运行和工具说明。
- `requirments.txt`：完整环境冻结，包含 Crawl4AI、CrewAI、PraisonAI、LangChain、向量库和多种云 SDK 等大量依赖。

`crawler.py` 实际执行链：

```text
URL
 -> WebCrawler.warmup()
 -> WebCrawler.run()
 -> LLMExtractionStrategy
 -> PageMetadata.model_json_schema()
 -> result.extracted_content
 -> unicode_escape 二次解码
 -> json.loads()
 -> result_json[0]
 -> DataCleanerTool
 -> MetadataAnalyzerTool
```

`Agents.yaml` 虽然声明 `framework: crewai` 和三个角色，但仓库展示的主执行代码并没有 durable multi-agent orchestration；实际仍是普通同步函数串行调用。因此应该把它理解为“职责拆分示例”，而不是 Agent 编排系统。

## 3. WebScraperTool：实现细节与技术原理

### 3.1 PageMetadata Schema

项目定义：

```text
title: str
description: str
keywords: list
```

然后调用 `PageMetadata.model_json_schema()`，把 JSON Schema 传给 Crawl4AI 的 `LLMExtractionStrategy`。

这比仅在 prompt 中要求“输出 JSON”更可靠，因为 Schema 明确约束字段名和结构。其本质是把 LLM 从自由文本生成器降级为“结构化候选抽取器”。但必须注意：**Schema 只约束输出形状，不保证字段语义来自页面事实，也不能消除 hallucination。**

例如 `keywords` 很容易被模型根据正文推断，而不是读取页面实际声明的 keywords/tag。生产系统因此必须区分：

```text
source-declared metadata
inferred metadata
LLM-generated candidate
```

不能把三者混成同一种 canonical 来源。

### 3.2 Ollama + qwen2:1.5b

源码将 provider 配为：

```text
ollama/qwen2:1.5b
```

优点：

- 本地执行，不依赖外部 API；
- 不必把抓取正文发送给第三方模型；
- 调用成本主要变成本地算力；
- 适合异常样本、离线诊断、规则候选生成。

缺点：

- 小模型结构化输出稳定性有限；
- 对日期、作者、keywords 等字段可能出现语义补全；
- 模型版本和 Ollama runtime 变化可能影响结果；
- 如果逐页执行，1000 站点、数百万 URL 下仍会形成巨大的推理吞吐和排队成本。

所以本地模型也不应该成为 canonical metadata 的必经链路。正确位置是 Provider Adapter，并受 budget、schema validation、shadow replay 和 candidate-only 约束。

### 3.3 当前 JSON 处理的隐含假设

源码：

```text
result = result.extracted_content ...
result_json = json.loads(result)
return result_json[0]
```

存在以下假设：

1. `extracted_content` 必然存在；
2. 一定是合法 JSON；
3. 根节点一定是数组；
4. 数组一定非空；
5. 第一项一定是正确且唯一结果；
6. 把 Schema 传给 LLM 后就不需要应用层二次验证。

生产实现应把状态拆开：

```text
llm_call_success
 -> raw_output_available
 -> json_parse_success
 -> schema_validation_success
 -> field_normalization_success
 -> candidate_quality_pass
 -> resolver_accepted/rejected
```

每一步必须有 reason code，不能只靠一个异常字符串。

## 4. DataCleanerTool：为什么应升级为独立 Metadata Normalizer

项目清洗逻辑只有：

- title/description `.strip()`；
- keyword `.strip().lower()`。

它虽然简单，却提示了一个重要边界：**Extraction 与 Normalization 是不同职责。**

现有博客知识库方案已经保存 `raw_value_json` 和 `normalized_value_json`，但如果不记录 Normalizer release，就无法回答：

- normalized 值是由哪一版规则产生的？
- 某次 title/author/tag 变化来自源网页变化，还是 normalizer 升级？
- 只升级日期解析/Unicode/空白规则时，如何不 refetch、不重新调用 LLM？
- 新 normalizer 是否导致大量字段发生非预期语义变化？

因此应把 Normalization 升级为显式阶段：

```text
raw metadata candidate
 -> Metadata Normalizer(versioned, deterministic)
 -> normalized candidate
 -> Metadata Resolver
 -> canonical metadata
```

### 4.1 Normalizer 的字段级规则

建议：

- **title / description / author**：HTML entity 解码、Unicode NFC、不可见字符处理、稳定空白折叠；默认不做会改变语义的全局大小写转换。
- **keyword / tag**：trim、Unicode NFC、可配置 casefold、去重；是否大小写折叠由平台/Profile 决定，而不是全局强制 `.lower()`。
- **URL**：交给统一 URL Canonicalizer，不能在 metadata cleaner 内自造第二套 URL 规则。
- **published_at / updated_at**：保留原字符串和 source timezone evidence，规范化为统一时间表示；模糊日期不得假装精确。
- **author identity**：显示名规范化与身份合并分开；“John Smith”和“J. Smith”不能仅靠字符串清洗自动合并。
- **language**：标准化语言代码，但必须保存检测来源和置信度。

### 4.2 Normalizer 必须满足的性质

1. **Deterministic**：相同输入 + 相同版本得到相同输出。
2. **Idempotent**：同一 release 下应尽量满足 `normalize(normalize(x)) == normalize(x)`。
3. **Non-destructive**：raw candidate 永久保留。
4. **Field-aware**：不同字段使用不同规则，不能用通用 `.lower()` 覆盖所有字符串。
5. **Replayable**：可基于历史 candidate 重跑，不 refetch、不重新调用 LLM。
6. **Observable**：统计每字段 mutation rate、fail rate、empty-after-normalize、语义风险变更。

这也是本次对总方案最重要的补强。

## 5. MetadataAnalyzerTool：从示例指标升级为治理面

原项目输出：

```text
title_length
description_length
keyword_count
top_keywords = keywords[:5]
```

这里的 `top_keywords` 实际只是列表前五项，并没有频次、TF-IDF、权重或模型排序，因此字段名比算法能力更强。生产系统必须让指标名称与算法语义严格一致。

可吸收的是“分析与 canonical 分离”这一思想。对 1000+ 站点，Metadata Quality Audit 应包含：

- required field coverage；
- source coverage；
- JSON-LD/OpenGraph/meta/selector 多来源一致率；
- candidate conflict rate；
- resolver fallback rate；
- title/description 长度分布和重复率；
- 模板化/boilerplate 比例；
- tag/keyword explosion；
- author/date 异常；
- language mismatch；
- metadata 与正文一致性；
- Normalizer mutation/failure rate；
- Profile/Contract/selector drift；
- LLM candidate validation/pass/fail/token/cost。

这些指标应驱动 Web 告警、shadow replay、规则工作台和人工审核，而不是直接自动修改 metadata。

## 6. Agents.yaml：职责建模值得保留，Agent 运行时不必照搬

`Agents.yaml` 将流程拆为：

```text
Web Scraper
Data Cleaner
Data Analyzer
```

生产系统可以保留相同职责边界，但实现为 deterministic pipeline stage：

```text
Fetch/Capture
 -> Metadata Candidate Extraction
 -> Metadata Normalization
 -> Metadata Resolution
 -> Metadata Quality Audit
 -> optional LLM Semantic Insight
```

不建议为了形式上的“多 Agent”把三个阶段都实现成 LLM Agent，因为会引入：

- prompt/version 漂移；
- 非确定性重试；
- Agent 对话状态难以 durable resume；
- 同一输入难以稳定 replay；
- 成本和延迟放大；
- canonical 真相链路难以审计。

LLM 应只出现在确实需要语义推断的窄边界。

## 7. 旧 Crawl4AI API 与当前实现迁移

项目依赖固定到 Crawl4AI git commit：

```text
3ff2a0d0e7fed891a61ae3c66037d0d6a04749c1
```

源码使用旧式：

```text
WebCrawler
LLMExtractionStrategy(provider=..., api_token=...)
bypass_cache=True
```

当前 Crawl4AI 官方文档采用：

```text
AsyncWebCrawler
BrowserConfig
CrawlerRunConfig
LLMConfig
LLMExtractionStrategy(llm_config=...)
```

官方 v0.5.0 已移除 `LLMExtractionStrategy` 中直接传 `provider`、`api_token`、`base_url`、`api_base` 等旧参数的方式，因此该仓库代码不能作为当前 API 的直接模板。

对于稳定结构页面，当前官方文档提供 `JsonCssExtractionStrategy` / `JsonXPathExtractionStrategy`，这类确定性策略应优先用于博客 metadata；LLM 更适合复杂、非结构化页面或 schema/selector 候选生成。

## 8. LLM fallback 还需要补一层“输入投影”契约

原项目直接让 Crawl4AI 把抓取内容交给 LLM。对生产知识库，仅记录 `input_hash` 仍不够，因为同一 snapshot 可以有多种输入视图：

- raw HTML；
- cleaned HTML；
- fit markdown；
- Article IR 的正文投影；
- 只包含 `<head>`、JSON-LD、OpenGraph 和邻近上下文的 metadata projection。

当前 Crawl4AI 的 LLM extraction 支持把策略放进 `CrawlerRunConfig`，并可选择不同输入格式。生产系统应该把“送给模型的输入是什么”版本化：

```text
input_artifact_kind
input_projection_version
input_hash
input_size
redaction_policy_version
```

推荐默认：

```text
metadata_head / deterministic metadata candidates / bounded article context
 -> projection + redaction
 -> schema-constrained LLM
```

而不是把完整 raw HTML 默认发送给远程模型。

这样做有四个收益：

1. 降 token/cost；
2. 减少 prompt injection 和无关噪声；
3. 模型结果可复现、可解释；
4. 输入投影变化可以单独 shadow/replay，而不 refetch。

## 9. `unicode_escape` 二次解码问题

源码执行：

```text
encode('utf-8').decode('unicode_escape')
```

这对已经是 Python Unicode 的字符串是危险的，可能破坏中文、emoji、反斜杠和技术代码文本。

正确链路：

```text
HTTP raw response bytes
 -> charset detection/HTML rules
 -> Unicode document
 -> JSON parser 处理 JSON escape
 -> application validation
```

不能在后处理阶段再对正常 Unicode 做一次 `unicode_escape`。

博客知识库方案因此应持续保留：

- HTTP raw bytes；
- decoder observation；
- rendered DOM 单独 artifact；
- 解析后的 Unicode 文本；
- serializer/version provenance。

## 10. Cache、增量同步和 Source Politeness

项目使用 `bypass_cache=True` 是示例级行为。大规模长期同步不应把 Crawl4AI cache 当业务状态，但也不能每次无条件重新拉完整正文。

生产路径：

```text
ETag / Last-Modified
 -> conditional GET
 -> 304: unchanged
 -> 200: compare raw body hash
 -> unchanged: update last_seen
 -> changed: create immutable snapshot
 -> offline extract/normalize/resolve
```

这比“应用 cache 命中”更适合作为长期增量同步语义，因为 HTTP validator 属于源站协议证据，而业务状态仍由 PostgreSQL + immutable snapshot 控制。

## 11. 依赖和运行时隔离

`requirments.txt` 对一个只有数个 Python 文件的小项目来说依赖非常大，包含多种 Agent、RAG、向量库、云 SDK 和 AI provider，更像环境 freeze 而不是最小 production dependency set。

博客知识库应按 Worker 类型拆运行时：

```text
HTTP/Discovery Worker
Browser Worker
Extract Worker
Metadata LLM Worker
Index/Embedding Worker
Publish Worker
```

收益：

- Browser/native 崩溃不影响 API；
- LLM/向量依赖不进入抓取镜像；
- 镜像更小、CVE 面更窄；
- 每类 Worker 可独立扩缩容；
- GPU/CPU/Browser 节点可分别部署。

## 12. 对总方案的数据模型改进

### 12.1 Metadata extraction attempt

应至少保存：

```text
metadata_extraction_attempts
- id
- site_id
- snapshot_id
- extraction_attempt_id nullable
- strategy_type(selector/jsonld/opengraph/meta/dom/llm_schema/archive_hint)
- schema_version
- rule_or_contract_version
- model_provider nullable
- model_name nullable
- prompt_version nullable
- input_artifact_kind
- input_projection_version
- input_hash
- raw_output_ref nullable
- validated_output_json
- status(pass/fail/review)
- reason_codes_json
- token_usage nullable
- estimated_cost nullable
- created_at
```

### 12.2 Metadata normalization run

新增：

```text
metadata_normalization_runs
- id
- site_id
- snapshot_id
- metadata_extraction_attempt_id
- normalizer_version
- config_hash
- status(pass/partial/fail)
- changed_field_count
- reason_codes_json
- created_at
```

`metadata_candidates` 增加：

```text
raw_value_json
normalized_value_json
normalized_value_hash
normalizer_version
normalization_status
normalization_reason_codes_json
```

### 12.3 Article version

`article_versions` 增加：

```text
metadata_normalizer_version
```

这样任意 canonical metadata 都能追溯：

```text
source -> extraction strategy -> raw candidate -> normalizer -> normalized candidate -> resolver -> final value
```

## 13. 对 Web 管理功能的改进

Metadata Quality Console 除了已有 coverage/conflict/drift，应增加：

- raw → normalized 字段级 diff；
- Normalizer version；
- normalization pass/partial/fail；
- mutation rate；
- empty-after-normalize；
- “疑似过度清洗”样本；
- 按字段统计 normalizer reason code；
- `re-run normalization without refetch/without LLM`；
- 新 Normalizer release 的 golden/shadow diff；
- LLM input artifact/projection/version；
- LLM token/cost 与输入大小关系。

这会让 Web 管理端能够明确区分三类故障：

```text
抓错了（Extraction）
清洗错了（Normalization）
选错了（Resolution）
```

这比只显示“最终 metadata 不对”更适合长期维护 1000 个站点。

## 14. 推荐最终 Metadata Pipeline

```text
Immutable Snapshot
 -> deterministic candidate extractors
    - approved site/platform rule
    - JSON-LD
    - OpenGraph/meta
    - DOM selector
    - generic extractor
    - archive hint
 -> optional schema-constrained LLM candidate
    - bounded/versioned input projection
    - JSON parse
    - application schema validation
 -> raw Metadata Candidates
 -> deterministic versioned Metadata Normalizer
 -> normalized Metadata Candidates
 -> field-aware Metadata Resolver
 -> canonical Metadata
 -> Article IR / Markdown
 -> async Metadata Quality Audit
 -> optional semantic insight
```

关键约束：

1. LLM 不能直接写 canonical metadata。
2. Normalizer 不能决定哪个 source 胜出。
3. Resolver 不应破坏 raw/normalized candidate。
4. Audit 不应反向修改 canonical。
5. Extraction、Normalization、Resolution、Audit 都要独立版本化。
6. 上述任一规则升级优先基于 immutable snapshot/candidate replay，不 refetch。

## 15. 与当前 Crawl4AI 的实现映射

稳定站点：

```text
AsyncWebCrawler
 + CrawlerRunConfig
 + JsonCssExtractionStrategy / JsonXPathExtractionStrategy
 -> deterministic candidate
```

异常/非结构化 metadata：

```text
bounded/versioned input projection
 + LLMConfig
 + LLMExtractionStrategy
 + Pydantic model_json_schema()
 -> JSON
 -> application schema validation
 -> candidate only
```

模型调用使用独立 quota，并记录 provider 返回的 token usage；若 provider 不返回完整 usage，应显式标记未知，不能伪造精确成本。

## 16. 不应从该项目吸收的部分

- 不采用单 URL 同步主循环作为平台调度器。
- 不为每次请求重新初始化 crawler/warmup。
- 不用 `bypass_cache=True` 替代增量协议。
- 不让 LLM 成为 title/date/author 等确定性 metadata 的默认抽取器。
- 不把 Schema 约束视为事实正确性保证。
- 不使用 `unicode_escape` 二次解码。
- 不把 `.lower()` 当通用 metadata 标准化规则。
- 不把前三个 Tool 强行实现为三个长期对话 Agent。
- 不把其完整 requirements freeze 直接复制进生产镜像。

## 17. 最终评价

`crawlwithagents` 对本需求的价值是**局部模式，而不是系统底座**。

最值得吸收的原始模式：

```text
Schema-constrained extraction
 -> explicit cleaning
 -> metadata analysis
```

生产化后应变成：

```text
确定性 metadata sources
 -> raw candidates
 -> optional schema-constrained LLM candidate
 -> explicit schema validation
 -> versioned deterministic normalization
 -> normalized candidates
 -> field-aware resolver
 -> canonical metadata
 -> deterministic quality audit
 -> optional semantic insight
```

这一改造既保留了项目“抽取、清洗、分析分层”的优点，又补齐 1000+ 站点生产系统必须具备的可追溯、可重放、可扩展、成本可控和安全边界。

## 参考

- 项目：https://github.com/varunsaagar/crawlwithagents
- 核心源码：https://github.com/varunsaagar/crawlwithagents/blob/main/crawler.py
- Agent 配置：https://github.com/varunsaagar/crawlwithagents/blob/main/Agents.yaml
- 依赖：https://github.com/varunsaagar/crawlwithagents/blob/main/requirments.txt
- Crawl4AI AsyncWebCrawler：https://docs.crawl4ai.com/api/async-webcrawler/
- Crawl4AI LLM extraction：https://docs.crawl4ai.com/extraction/llm-strategies/
- Crawl4AI non-LLM extraction：https://docs.crawl4ai.com/extraction/no-llm-strategies/
- Crawl4AI v0.5.0 breaking changes：https://docs.crawl4ai.com/blog/releases/0.5.0/
