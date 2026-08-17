# 使用 Crawl4AI 构建面向 AI 的网页爬取流程：实现细节与技术原理

- 编号：4
- 原文标题：AI ready web crawling using Crawl4AI
- 原文作者：Varunrajeevan
- 原文：https://medium.com/towards-data-engineering/ai-ready-web-crawling-using-crawl4ai-c4abc3701257
- 原文发布时间：2025-11-24
- 本次核对时间：2026-08-14
- Crawl4AI 当前官方文档：https://docs.crawl4ai.com/
- 深度抓取：https://docs.crawl4ai.com/core/deep-crawling/
- 无 LLM 抽取：https://docs.crawl4ai.com/extraction/no-llm-strategies/
- LLM 抽取：https://docs.crawl4ai.com/extraction/llm-strategies/
- CrawlResult：https://docs.crawl4ai.com/api/crawl-result/
- Browser/Crawler 配置：https://docs.crawl4ai.com/core/browser-crawler-config/
- 调研目标：判断文章中的 Crawl4AI 深爬、URL 过滤、结构化抽取和 LLM 抽取机制，哪些适合进入“约 1000 个技术博客历史全量 + 长期增量 + Web 管理”的生产方案，哪些只能作为演示代码。

## 1. 结论

原文真正展示的是一条面向 AI 数据准备的局部执行流水线：

```text
入口页
 -> CrawlResult.links 暴露候选链接
 -> Deep Crawl
 -> URL FilterChain
 -> URL Scorer / Best-First
 -> 页面抓取
 -> CSS/XPath/Regex/LLM Extraction
 -> JSON
 -> 后续向量库 / RAG
```

它对当前博客知识库方案有直接价值，但不能原样放大到 1000 个站点。最值得吸收的不是“用 Crawl4AI 把整个系统做完”，而是下面六个机制：

1. **发现、过滤、排序、抓取、抽取是不同决策。** 不能用一个 `arun()` 调用把业务状态、抓取状态和知识库状态混在一起。
2. **Best-First 解决先抓谁，不解决是否抓全。** `KeywordRelevanceScorer` 是优先级启发式，不是历史全量完成判据。
3. **FilterChain 适合做 URL 准入，但硬过滤必须按 Crawl Mode 使用。** 对首次全量，语义/SEO/关键词规则不能因为“不相关”就永久丢 URL；对专题抓取才允许强裁剪。
4. **CSS/XPath/Regex 应成为默认的确定性结构化抽取层。** 特别是 Regex 可用于日期、URL、标识符等字段；LLM 只做规则生成或特殊页面升级。
5. **LLMExtractionStrategy 的 chunking 不是“免费扩展上下文”。** overlap 会增加 token 成本，chunk 合并会制造重复和结构冲突，必须作为版本化策略并做 schema validation、去重和成本治理。
6. **Crawl4AI 应是 Adapter 内核，不是平台状态真相源。** `BrowserConfig`、`CrawlerRunConfig`、`CrawlResult` 都需要被平台自己的稳定契约包住，并固定 Crawl4AI 版本。

此外，本次重新核对原文后要纠正一个归因问题：**原文重点是 Deep Crawl、URL Filtering、CSS/XPath/Regex/LLM Extraction，并没有以 AdaptiveCrawler、PruningContentFilter 或 Sitemap Provider 为核心展开。** 这些能力可以继续存在于总体技术方案中，但应视为其他调研和官方文档带来的能力，不能归因于本篇文章。

## 2. 原文的起点：为什么单页抓取不够

文章用 GeeksforGeeks 的面试准备入口页举例。第一次只对入口页执行普通 `arun()` 时，结果中的 Markdown 主要是题目链接和摘要，真正需要的问答正文存在子页面中，因此必须继续访问链接。

简单调用由四个对象组成：

```text
BrowserConfig
CrawlerRunConfig
AsyncWebCrawler
CrawlResult
```

其中：

- `BrowserConfig` 管浏览器环境，例如 headless、header、cookie、浏览器级设置；
- `CrawlerRunConfig` 管本次运行，例如链接排除、iframe、overlay、deep crawl、extraction strategy、cache 等；
- `AsyncWebCrawler` 是异步执行器；
- `CrawlResult` 是结果容器，包含 HTML、cleaned HTML、Markdown、links、media、metadata、extracted_content、状态和错误等。

这套分层对平台设计很重要：**浏览器环境配置与单次任务配置本来就是两个生命周期。** 生产平台不能把它们合并成一个不可审计的 JSON blob，更不能让 worker 自己临时拼参数。

建议在平台中映射为：

```text
FetchProfileRelease
  -> HTTP/Browser、UA、proxy、timeout、resource policy、session policy

CrawlerExecutionRelease
  -> Crawl4AI 版本、CrawlerRunConfig 投影、deep crawl 参数、stream、cache 行为

ExtractionPipelineRelease
  -> CSS/XPath/Regex/通用正文抽取/LLM 候选链
```

Crawl4AI 升级时只调整 Adapter 的投影和兼容层，业务数据模型不随第三方库参数名变化。

## 3. Deep Crawl 的技术原理

网站可以抽象成有向图：页面是节点，超链接是边。Deep Crawl 本质上是在维护一个 frontier，并反复执行“取一个候选 URL -> 抓页面 -> 发现新 URL -> 放回 frontier”。不同策略只改变 frontier 的取出顺序。

### 3.1 BFS

BFS 使用 FIFO queue：

```text
depth 0 全部处理
 -> depth 1 全部处理
 -> depth 2 ...
```

优点：

- 层级覆盖容易理解；
- 对归档页、分类页这种浅层扩散型站点比较稳定；
- 方便限制最大深度。

缺点：

- 一个入口页有大量导航链接时，会先消耗大量低价值 URL；
- 高价值文章详情页不一定尽早执行；
- 对超大站点若把一层候选全部放内存，frontier 会膨胀。

### 3.2 DFS

DFS 使用 stack 或递归式 frontier：

```text
选一条边一直向下
 -> 到底后回溯
```

优点是实现简单、局部路径推进快，但很容易被日历、分页、目录树等深路径拖住。原文因此认为当前案例不需要 DFS。

### 3.3 Best-First

Best-First 使用带优先级的 frontier，通常可以理解为 priority queue：

```text
score(url) 越高
 -> 越早出队
```

原文使用 `KeywordRelevanceScorer`，关键词是 `sort/find/most/nth`，再配合 `BestFirstCrawlingStrategy`。这种 scorer 的价值是**把有限预算优先花在更可能是目标页的 URL 上**。

如果 frontier 用 heap 维护，单次插入/取出通常是 `O(log n)`；真正昂贵的仍然是网络、浏览器和抽取。评分本身只是调度先验。

关键生产结论：

```text
Best-First = ordering
Best-First != completeness
```

对 `FULL_BACKFILL`，score 只能决定顺序。只要 URL 已通过合法准入，就不能因为分数低永久丢弃；`max_pages` 也只能是单次 worker slice 的预算。对 `TOPIC_EXPANSION`，才允许使用 score threshold 或固定总页数作为真正停止条件。

## 4. FilterChain：硬准入与软相关性必须分开

原文列出并演示了 Crawl4AI 的 URL Filter：

- `DomainFilter`；
- `URLPatternFilter`；
- `ContentTypeFilter`；
- `ContentRelevanceFilter`；
- `SEOFilter`。

案例中只需要 GeeksforGeeks 的 DSA 题目页，因此组合：

```text
DomainFilter(geeksforgeeks.org)
AND URLPatternFilter(*dsa*)
```

这在单任务场景非常合理，但不能照搬为通用博客历史全量规则。

### 4.1 平台应把过滤器分成三类

**安全硬过滤**：任何 Crawl Mode 都可启用，例如：

- SSRF/Egress；
- 明确的域名 allowlist；
- robots / site policy；
- 明确不可抓的二进制类型；
- tracking/session 参数清理；
- 已确认的 crawler trap。

**站点结构硬过滤**：只有在 Probe + canary 证明稳定后才用于 `FULL_BACKFILL`，例如某 WordPress 站的 `/wp-admin/`、搜索页、登录页。

**相关性软过滤**：关键词、SEO、BM25/语义相关性默认只做 score，不应在首次历史全量中成为永久 reject 条件。

否则“标题/路径里没有某关键词”的旧文章会被永远漏掉，且系统还可能错误地把“队列空了”当作全量完成。

### 4.2 URLPatternFilter 的维护风险

原文的 `*dsa*` 很适合演示，因为目标站 URL 结构稳定。但博客迁移、slug 规则变化、旧文章路径不同、语言切换都会破坏 pattern。

因此生产系统要记录：

```text
filter_release_id
rule_id
decision = ADMIT / REJECT / SCORE_ONLY
reason
matched_value
source_provider
```

Web 管理端必须可以解释“为什么一个 URL 没被抓”，而不是只显示 0/1。

## 5. 原文的抽取路线：确定性优先，LLM 最后

原文把 Data Extraction 明确分为两类：

1. LLM-free；
2. LLM-based。

这个方向与 1000 个博客的成本约束高度一致。

## 6. CSS / XPath Schema Extraction

原文用电商测试页演示 `JsonCssExtractionStrategy` 和 `JsonXPathExtractionStrategy`。两者本质上都是：

```text
选择重复记录的 base node
 -> 对每个 node 执行字段 selector
 -> 输出结构化 JSON
```

### 6.1 适合什么

- 模板稳定的博客正文容器；
- 标题、作者、日期、标签；
- 归档页文章链接；
- 平台型站点（WordPress/Ghost/静态生成器）可复用模板。

### 6.2 为什么适合大规模

它没有逐页模型调用，结果可重现、容易测试、速度快。规则变化后只要保留 raw snapshot，就能离线 replay。

### 6.3 必须避免 selector 直接成为“最终事实”

选择器漂移是必然事件，因此结果应先成为 candidate：

```text
candidate.value
candidate.source = CSS/XPath
candidate.selector_release
candidate.snapshot_id
candidate.evidence
```

再由 Metadata Resolver / Quality Gate 决定最终 title/date/content。

## 7. RegexExtractionStrategy：原方案最值得补的一层

原文专门介绍了 `RegexExtractionStrategy`，包括内置 URL、Currency 等模式、custom regex，以及“让 LLM 一次性生成 regex，然后复用”的思路。当前官方文档仍支持这一机制。

对技术博客知识库，Regex 不适合识别整篇正文，但非常适合做廉价字段提取和证据补充：

- ISO 日期；
- 版本号；
- issue/PR 编号；
- 邮箱/URL；
- 站点自定义 ID；
- 某些固定 front matter 文本；
- 页面中的 canonical-like 标识。

推荐候选顺序增加：

```text
JSON-LD / meta
 -> CSS/XPath
 -> Regex field rules
 -> Trafilatura / Readability / Crawl4AI Markdown
 -> LLM
```

### 7.1 LLM 生成 Regex 的正确生产用法

不能每页请求 LLM 生成 pattern。正确流程是：

```text
Probe 抽样 10~50 个页面
 -> LLM 只生成一次候选 regex
 -> 静态安全检查 / catastrophic backtracking 检查
 -> golden sample 验证
 -> 人工或 canary 发布
 -> 保存 RegexRuleRelease
 -> 后续百万页面纯 regex 执行
```

需要记录：

```text
regex_rule_release
pattern
input_format
generated_by_model
prompt_release
sample_snapshot_ids
validation_metrics
created_at
```

这样把 LLM 从“每次执行成本”变成“规则生产成本”。

### 7.2 原文代码里的一个实际问题

原文示例先把 `generate_pattern()` 的结果赋给 `description_pattern`，随后又放进 `regex_pattern`，但构造 `RegexExtractionStrategy` 时使用的是另一个未定义变量名 `llm_pattern`。这是示例代码的变量名不一致，直接复制会失败。

这恰好说明生产平台必须有：

- 配置 schema 校验；
- Adapter contract test；
- canary URL；
- 发布前真实执行；

不能把教程代码当成已验证的生产配置。

## 8. LLMExtractionStrategy：chunking 的真实成本与合并问题

原文用 Pydantic schema 定义结构化结果，并配置：

```text
schema
instruction
chunk_token_threshold
overlap_rate
apply_chunking
input_format
extra_args
```

其原理是把页面输入切成多个 chunk，每个 chunk 独立调用模型，然后聚合结果。

假设页面输入 token 数为 `T`，chunk 上限为 `C`，重叠率为 `r`，近似步长为：

```text
step ~= C * (1 - r)
```

chunk 数大致随 `T / step` 增长。`r` 越大，上下文连续性越好，但重复输入 token 和费用也越高。

### 8.1 overlap 不是越大越好

对问答、表格或代码块，chunk 边界可能切断语义；适当 overlap 有价值。但它会带来：

- 同一实体在相邻 chunk 被抽两次；
- 同一 Question/Answer 出现重复对象；
- chunk A/B 对同一字段给出冲突值；
- token 使用量上升；
- merge 阶段需要稳定 identity key。

因此平台 LLM candidate 必须有 chunk-aware merge：

```text
chunk_id
chunk_span
candidate_identity_key
merge_rule_release
schema_validation_result
```

结构化对象优先按稳定字段或内容 hash 去重，不能简单 `list.extend()`。

### 8.2 schema 只约束输出形状，不保证事实正确

Pydantic/JSON Schema 可以让“输出字段结构”更稳定，但不能证明内容与网页一致。最终仍要：

- parse validation；
- 必填字段检查；
- 与 DOM/metadata 证据交叉验证；
- provenance；
- 低置信度降级/人工审核。

### 8.3 温度与可重放

原文 `LLMConfig` 示例中设置过较高 temperature，同时 `extra_args` 又使用 0.0。生产环境应避免这种双层配置冲突，Adapter 必须计算最终 effective config，并把它固化到 release。

对结构化抽取推荐确定性参数优先，例如 temperature 0 或接近 0；任何模型、prompt、schema、chunk 参数变化都应产生新 release。

### 8.4 `show_usage()` 只是观测入口，不应停留在日志

官方文档说明 `LLMExtractionStrategy` 会累计 chunk/call usage，并可用 `show_usage()` 输出。生产系统不能只打印日志，应把使用量结构化写入：

```text
llm_call_count
input_tokens
output_tokens
total_tokens
estimated_cost
provider_latency
model
prompt_release
```

然后按站点、任务、月份做预算和熔断。

## 9. 原文最终示例的完整执行链

最终例子把：

```text
QuestionAnswerSchema
+ LLMExtractionStrategy
+ DomainFilter
+ URLPatternFilter
+ KeywordRelevanceScorer
+ BestFirstCrawlingStrategy
+ CrawlerRunConfig
```

组合起来，对入口页做 deep crawl，再把每个成功结果的 `extracted_content` 解析成 JSON，最后写入 `kb_result.json`。

这个示例非常适合证明“Crawl4AI 能把 URL 筛选和结构化抽取连起来”，但生产系统还缺很多关键边界。

## 10. 不能原样照搬的生产问题

### 10.1 Browser-first 成本过高

教程直接使用 `BrowserConfig(headless=True)`，但 1000 个博客不能默认让 Chromium 承担静态 HTML 抓取。平台应 HTTP-first，只有出现 SPA shell、JS pagination、load more、infinite scroll 等证据才升级 Browser。

### 10.2 `max_pages` 是演示预算，不是历史全量上限

原文把页数限制在很小的开发值。生产 `FULL_BACKFILL` 如果直接把该值当作站点完成条件，会确定性漏数据。

正确含义：

```text
execution slice budget
```

worker 达到预算后把 frontier/checkpoint 持久化，Scheduler 后续继续。

### 10.3 文章把 discovery、fetch、LLM extraction 耦合在一次 crawl 中

这会导致网络延迟、Browser 资源和 LLM 延迟互相拖累。生产应拆为：

```text
Discovery Worker
 -> durable frontier
Fetch Worker
 -> immutable snapshot
Extract Worker
 -> deterministic candidates
LLM Extract Worker（少量升级）
```

网络抓取完成后应先保存 snapshot；即使 LLM 服务故障，也不需要重新请求网站。

### 10.4 `CacheMode.BYPASS` 不能替代增量同步

文章最后示例使用 `CacheMode.BYPASS`，这只是在 Crawl4AI 本地缓存层选择是否读缓存，不是业务级 freshness 协议。

增量同步应优先使用：

```text
Sitemap lastmod
Feed GUID / updated_at
API cursor
ETag / If-None-Match
Last-Modified / If-Modified-Since
normalized content hash
```

Crawl4AI cache 只是 Adapter 内优化，不能决定文章是否产生新版本。

### 10.5 结果不能只攒在进程内列表

教程把结果累积到 `all_qa_data` 后一次性写 JSON。大规模 deep crawl 应优先流式消费：

```text
每产生一个 CrawlResult
 -> 立即归一化
 -> 保存 snapshot/attempt
 -> links 进入 durable frontier
 -> ack 当前 task
```

当前 Crawl4AI 官方 Deep Crawling / multi-URL 文档支持 streaming；在 worker slice 内使用 streaming 可降低峰值内存，但 durable 状态仍由平台数据库负责。

### 10.6 URL scorer 只看词面时容易产生偏差

原文关键词是为 DSA 题目人工挑选的。博客路径可能是日期、UUID、slug、数字 ID，关键词 scorer 只能作为一种先验，不应成为通用判定器。

平台 scorer 应组合：

```text
source_priority
path_pattern_score
page_type_score
archive_expansion_score
recency_score
optional_semantic_score
trap_penalty
duplicate_penalty
aging
```

其中 `aging` 防止历史低分 URL 永久饿死。

## 11. CrawlResult 应被归一化成稳定平台契约

当前官方 `CrawlResult` 提供的核心数据包括：

- 原始 HTML；
- cleaned HTML；
- MarkdownGenerationResult（raw/fit 等）；
- extracted_content；
- links；
- media；
- metadata；
- status / headers / error；
- dispatcher 相关运行信息（特定场景）。

这些字段在 Crawl4AI 版本演进中曾发生过层级或命名变化，例如 fit Markdown 不应假定永远是顶层字段。因此平台 Adapter 输出应固定为：

```text
FetchResult {
  task_id
  attempt_id
  crawl_mode
  engine = crawl4ai
  engine_version
  final_url
  status_code
  response_headers
  raw_html_ref
  cleaned_html_ref
  rendered_dom_ref
  raw_markdown_ref
  fit_markdown_ref
  extracted_content_ref
  links_ref
  media_ref
  engine_metrics
  error_class
}
```

业务层绝不直接依赖 `result.xxx` 的第三方对象结构。

## 12. 对当前总体方案应新增/强化的能力

### 12.1 增加 RegexRuleRelease

把 Regex 作为确定性字段抽取器，并支持“一次性 LLM 生成 -> 验证 -> 版本化复用”。

### 12.2 增加 LLMChunkPolicyRelease

至少固化：

```text
input_format
chunk_token_threshold
overlap_rate
apply_chunking
merge_strategy
identity_keys
schema_release
model_release
prompt_release
effective_generation_args
```

### 12.3 Filter 必须标注决策语义

每个 Filter Rule 除了表达式，还需要：

```text
HARD_REJECT
SOFT_SCORE
OBSERVE_ONLY
allowed_crawl_modes
```

这样同一条“关键词不相关”规则可以在 `TOPIC_EXPANSION` 里硬裁剪，在 `FULL_BACKFILL` 里仅降低优先级。

### 12.4 Crawl4AI worker 使用流式结果摄取

Deep crawl / `arun_many()` 一旦启用 stream，Adapter 每拿到一个结果就落 durable 状态，避免一次 crawl 数百/数千页后才统一提交。

### 12.5 把 Crawl4AI 配置做成“平台配置 -> Adapter 投影”

不要直接在数据库存第三方 Python 对象。平台存稳定配置：

```text
BrowserProfileRelease
FetchProfileRelease
URLFilterRelease
URLScorerRelease
ExtractionPipelineRelease
LLMChunkPolicyRelease
```

Adapter 根据固定 engine version 生成 `BrowserConfig/CrawlerRunConfig`，并做 capability validation。

### 12.6 LLM 规则生成与 LLM 逐页抽取分账

两者目的不同：

```text
Rule Authoring LLM：低频、一次生成 CSS/XPath/Regex 候选规则
Per-page LLM Extraction：高成本、只用于难页面
```

Web 管理端、预算和指标必须分开统计。

## 13. 与总体架构的最终映射

推荐生产链路：

```text
Site Probe
 -> Provider Discovery
 -> URL Normalize / Admission
 -> Durable Best-First Frontier
 -> HTTP Fetch
 -> [必要时] Browser Fetch / Crawl4AI Adapter
 -> Immutable Snapshot
 -> JSON-LD / CSS / XPath / Regex candidates
 -> Trafilatura / Readability / Crawl4AI Markdown candidates
 -> Quality Gate
 -> [低质量或语义任务] LLM Extraction + Chunk Merge
 -> Article IR
 -> Markdown Serializer
 -> Article Version
 -> Search / Vector / External KB
```

Crawl4AI 的定位：

```text
很好用的执行引擎
!=
整个知识库平台
```

它最适合提供：Browser、Deep Crawl、FilterChain、Scorer、Markdown、CSS/XPath/Regex/LLM extraction、worker 内并发和 streaming；平台继续负责 durable frontier、跨站公平、checkpoint、幂等、版本、质量、成本、安全、审计和 Web 管理。

## 14. 对现有方案的优化结论

当前《博客知识库技术方案.md》的总体方向已经正确，不需要推翻。根据本篇原文，应重点补四处：

1. **抽取链补 Regex，并把 LLM 辅助生成 Regex/CSS/XPath 规则定义为 Probe/Rule Authoring 能力。**
2. **LLM 抽取补 chunk policy、chunk merge、schema validation 和结构化 token/cost 记录。**
3. **Crawl4AI Adapter 补 BrowserConfig/CrawlerRunConfig 的版本化投影，以及 CrawlResult 的稳定归一化。**
4. **Deep Crawl worker 明确采用 bounded slice + stream ingestion；`max_pages`、Filter 和 Scorer 在不同 Crawl Mode 下采用不同语义。**

原文中的 GFG Demo 适合验证能力，不适合直接定义生产架构。把其强项收进 Adapter，把状态、完备性和质量留在平台层，才符合 1000+ 技术博客长期运行的要求。