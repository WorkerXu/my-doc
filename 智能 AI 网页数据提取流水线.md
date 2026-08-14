# 智能 AI 网页数据提取流水线：实现细节与技术原理分析

调研对象：https://github.com/ShyamThangaraj/Intelligent-AI-Web-Extraction-Pipeline

## 1. 项目定位

该项目不是面向“1000 个博客长期全量同步”的完整生产系统，而是一个面向任意站点的自适应结构化抽取原型。它把网页采集拆成几个相对独立的环节：站点入口发现、列表页识别、页面抓取、Schema 生成、字段抽取、结果校验和数据库落库。核心价值不在于它现成的调度能力，而在于它展示了一种值得吸收的模式：**让 AI 参与生成或修复声明式抽取规则，然后让确定性的爬虫与选择器执行实际抽取**。

这与知识库系统“LLM 不应直接决定 canonical Markdown”的原则并不冲突。更合理的用法是让 LLM 充当 Rule/Schema Synthesizer，只产生候选规则；候选规则经过真实快照回放、质量门禁和人工或自动发布流程后，才进入生产运行。

## 2. 代码结构与职责划分

项目主要由以下模块组成：

- `main.py`：SERP 路径的总流程，先借助搜索结果寻找博客/职位列表页，再生成 Schema 并抽取。
- `main_norm.py`：不依赖 SERP 的直接站内发现流程。
- `modules/link_extracter_serp.py`：搜索引擎结果驱动的入口发现。
- `modules/link_extractor_norm.py`：站内直接抓取和候选入口识别。
- `modules/schema_generator.py`：Schema 的创建、读取、版本递增、LLM 生成以及失败后的 LLM 修复。
- `modules/data_extractor.py`：列表页翻页、URL 发现、HTML 抓取、JSON-LD/Meta/selector 等字段抽取。
- `modules/data_validator.py`：使用 LLM 对职位或博客标题做语义合法性检查。
- `modules/database.py`：SQLite 存储和运行记录。
- `schemas/<domain>/...vN.json`：按域名和内容类型保存版本化抽取 Schema。

这种模块划分说明项目已经意识到“发现”和“抽取规则”是两种不同能力。对大规模博客知识库来说，应继续把它们拆得更彻底：Source Discovery、URL Discovery、Fetch、Capture、Extract、Quality Gate、Publish 各自拥有独立状态和重试边界。

## 3. URL 发现实现原理

### 3.1 列表页中的文章 URL 识别

`data_extractor.py` 内部先通过 `HTMLParser` 收集 `<a>` 的 `href/text/rel/class/id`，再调用 `urljoin` 把相对地址变成绝对地址，并去除 fragment。

候选文章 URL 主要通过路径启发式判断，例如：

- `/blog/xxx`
- `/blogs/xxx`
- `/article...`
- `/post/`、`/posts/`
- `/insight/`、`/news/`、`/story/`
- `YYYY/MM/...` 形式的时间型永久链接
- `.html`

导航、菜单、footer、登录、注册、社交等链接则用 class/id/text 关键词过滤。

这是一种典型的“廉价先验 + 后续验证”策略：第一阶段只做高召回候选生成，不必在 URL 层精确判断是否一定是文章。生产系统应保留这一思想，但不应把 URL pattern 当最终事实；候选 URL 后续还要经过 page type classifier、content signal、canonical/identity 和 Quality Gate。

### 3.2 翻页发现

项目按三种优先级寻找下一页：

1. `<a rel="next">`；
2. 锚文本为 `next`、`older`、`load more` 等；
3. URL 中出现 `page=` 或 `p=`。

每次访问一页，抓取候选 URL，只有在发现新条目或第一次尝试下一页时才继续。默认 `max_pages` 很小，因此它更像验证原型，而不是全量历史发现器。

对知识库方案的启发是：**翻页终止条件必须被显式建模**。`end_reached`、`known_streak`、`repeated_cursor`、`no_new_urls`、`scroll_limit`、`budget_exhausted` 应是数据库中的可审计结果，而不是 while-loop 隐含结束。

### 3.3 搜索引擎作为 Source Discovery

项目提供 SERP 模式，用搜索引擎寻找难发现的博客/职位列表入口。它的合理定位应是“入口候选发现”，不是“历史文章全量来源”。

对 1000 站点方案可以增加一个可选 `SearchIndexSourceDiscoveryProvider`：

- 只在 probing、站点长期发现 0 篇、source coverage 异常或人工诊断时运行；
- 搜索目标是 `/blog`、`/archive`、`/posts`、年份归档、Sitemap、Feed、平台入口等 Source，而不是把搜索结果中的所有文章直接视作全量；
- 搜索得到的 host/URL 仍必须经过 approved host、Egress、robots 和范围审核；
- 搜索结果只能作为 discovery evidence，不能作为覆盖率终点证明。

这样既能吸收项目“搜索辅助发现”的优点，又避免 API 成本、结果不稳定和 coverage 不可证明的问题。

## 4. 抓取层实现及局限

### 4.1 Crawl4AI 优先、urllib 兜底

`data_extractor.py` 的抓取逻辑首先尝试 `AsyncWebCrawler.arun(url=...)`，然后从 `html/cleaned_html/content/raw_html` 中寻找 HTML；若这些字段都没有，则甚至会把 Markdown 包进 `<pre>` 作为后续输入。如果 Crawl4AI 失败，则退回 `urllib.request.urlopen`。

这里体现了“多实现 fallback”的思路，但原型写法不适合生产：

- 每个 URL 内部重新创建 `AsyncWebCrawler`，浏览器启动/销毁成本高；
- 同步函数内部反复 `asyncio.run()`，无法形成真正的端到端异步流水线；
- `urllib` fallback 直接 `resp.read()`，没有 streaming bytes 上限；
- 没有逐跳 redirect 的 Egress/SSRF 校验；
- `CrawlerSettings.respect_robots` 只是配置字段，在该段实现中没有形成完整 robots enforcement；
- `time.sleep()` 会阻塞执行线程；
- Crawl4AI 抓取结果与 urllib 原始 response bytes 的语义混在一起；
- 将 Markdown 再包装为 HTML 会破坏“输入类型 provenance”，不利于精确 replay。

因此，知识库系统应坚持当前方案中的更严格边界：HTTP 原始 bytes 是一种不可变 artifact，Browser rendered DOM 是另一种 artifact，二者不得互相冒充。Browser 应使用进程池和隔离 context，而不是逐 URL 启停。

### 4.2 URL cache

项目把 URL 做 SHA-1 后作为本地 HTML cache 文件名。这种缓存对单机原型简单有效，但没有：

- ETag/Last-Modified；
- TTL/失效规则；
- 原始 bytes hash 与 decoded HTML hash 的区分；
- 抓取器版本、规则版本、capture profile；
- 跨 worker 一致性；
- immutable snapshot / provenance。

因此生产知识库不能把类似 cache 作为业务状态；它只能作为可丢弃优化层。真正的状态应在 PostgreSQL，artifact 在对象存储。

## 5. Schema 生成与版本化

### 5.1 Schema 模型

项目为 blogs 定义了固定 required fields，例如：

- title
- author
- published_date
- tags
- content
- full_description
- source_url
- schema_version

Schema 按 `schemas/{domain}/{domain}.{type}.vN.json` 存储，包含：

- domain
- type
- version
- created_at / updated_at
- listing_url
- required_fields
- extraction_rules

并提供查找最新版本、保存、版本递增和旧目录迁移能力。

这个设计值得保留的核心不是文件目录，而是**Extraction Contract 可版本化**。对大型知识库，应该把它升级为正式数据模型并纳入 Profile/Rule release：

```text
extraction_contract_candidates
- id
- site_id / platform_profile_id
- page_type
- based_on_snapshot_ids
- generator_kind(manual/llm/template/inferred)
- generator_version
- prompt_version
- contract_json
- contract_hash
- status(draft/testing/approved/rejected/superseded)
- validation_metrics_json
- created_by / created_at

extraction_contract_releases
- id
- scope(site/platform)
- release_version
- contract_hash
- artifact_digest
- approved_by / approved_at
- rollback_of
```

运行期 Extract Worker 只读取 `approved` release，不允许现场调用 LLM 随意修改生产规则。

### 5.2 LLM 生成规则

`generate_schema_with_llm()` 让模型返回严格 JSON，并把每个字段映射到按顺序排列的 CSS/XPath selector。temperature 较低，目标是稳定输出。

它的关键思想是：LLM 输出的是“如何抽取”的声明式规则，而不是最终文章内容。这个方向比“每篇文章都让 LLM 看全文并改写为 Markdown”更适合 1000 站点：

- 推理成本从每篇文章下降为每个模板/站点偶发调用；
- selector 能在同模板大量页面复用；
- 规则可以回放、测试、diff、审批和回滚；
- canonical 内容仍由确定性解析器生成。

不过当前实现的初始 Schema prompt 主要基于 domain/type/listing URL 和一些通用 selector 建议，没有真正把多个代表性页面结构作为系统化训练样本，因此很容易生成“看起来合理但未命中”的 selector。

生产方案应改成：

1. probing 阶段保存 20~200 个代表性 snapshot；
2. 先用 DOM fingerprint/page type 聚类，避免一个站点多个模板混在一起；
3. 从每个模板抽取少量 sanitized DOM/结构摘要，而不是把整页无限送给 LLM；
4. LLM 仅生成 candidate contract；
5. 在全部 golden snapshots 上执行 selector；
6. 计算 required field coverage、正文长度、DOM 区域稳定性、模板污染、链接/代码/表格保留、canonical 一致性；
7. 未过阈值则自动让备用生成器修复或进入人工 review；
8. 通过后才发布 release。

这应称为 **Schema Synthesis / Rule Compiler**，属于控制面，不属于每篇文章的热路径。

## 6. Schema 自修复机制

`attempt_schema_repair_with_llm()` 会把当前 Schema、失败字段、页面 URL 以及裁剪后的 HTML sample 发送给模型，让模型只返回更新后的 `extraction_rules`。这构成了一个简单的闭环：

```text
extract -> required field missing -> collect failure context -> LLM repair -> new rules -> retry
```

技术上它类似 selector self-healing，但生产环境不能直接“失败即在线改规则”。否则单个异常页面可能污染整个站点的生产规则。

更稳妥的机制是：

```text
production extraction fail
 -> create drift evidence
 -> aggregate same template failures
 -> generate candidate contract repair
 -> shadow replay on historical golden snapshots
 -> compare old/new metrics
 -> canary a small URL set
 -> approve release
 -> replay affected snapshots
```

只有当多个同模板页面持续出现 selector zero-hit 或质量下降时，才值得发起修复。这能把 LLM 从“实时不可控依赖”变成“离线规则维护助手”。

## 7. 内容抽取策略

项目并不完全依赖 AI selector。`data_extractor.py` 还优先解析 JSON-LD，并识别 `BlogPosting`、`Article`、`NewsArticle`，从中读取 headline/name、author、datePublished、keywords 等字段。这是很重要的层次化策略：结构化元数据是低成本高置信度候选，而正文 selector、通用正文抽取器和启发式负责补足。

对知识库应明确 Metadata Candidate 的来源优先级，而不是“谁最后写入谁覆盖”：

```text
site/platform explicit rule
> valid JSON-LD / canonical metadata
> OpenGraph/meta
> article DOM selector
> generic extractor
> archive/listing hint
> LLM inferred candidate
```

每个值保留 source、location、confidence、rule_version、snapshot id，最终字段由 resolver 决定。

## 8. LLM 数据校验的实现与问题

`data_validator.py` 使用 LLM 判断提取出来的标题是否像真实 blog title/job title，并要求模型返回 YES/NO 或二进制字符串。这个方法适合作为辅助信号，但不能成为 canonical 发布的硬依赖。

实现中还有一个值得特别警惕的行为：批量校验只处理前 20 个标题，如果更多，剩余项会被补成 `False`；API 失败时同样全部返回 `False`。在原型里这是一种保守策略，在生产全量同步里会产生大量系统性误拒绝。

知识库应该把这类 LLM 校验降级为：

- `quality_signal`，而非直接删除；
- 失败时状态为 `unknown`，不能等价为 `invalid`；
- 允许采样检查，而不是每篇调用；
- 与确定性 page type、正文长度、template fingerprint、重复率、URL pattern、selector hit、JSON-LD 类型等组合打分；
- LLM provider/model/prompt version 和输入 hash 必须记录；
- LLM 不可用时 canonical pipeline 仍应正常运行。

## 9. 对现有博客知识库技术方案的可吸收优化

此次调研后，现有方案整体架构方向正确，无需把该项目的 SQLite、本地文件 cache、同步分页逻辑或逐页 LLM validation 直接引入。真正值得加入的是两项能力。

### 9.1 增加 Schema Synthesis / Rule Compiler

在 Control Plane 中增加“抽取规则候选生成器”：

- 输入：经过安全裁剪的 golden snapshots、DOM fingerprint、已有规则、失败 reason 和目标 Article IR 字段；
- 生成器：模板规则、统计 selector 推断、LLM candidate generator、人工编辑；
- 输出：结构化 Extraction Contract；
- 验证：历史 snapshots shadow replay + quality gate；
- 发布：testing -> canary -> active；
- 回滚：release 级回滚；
- 热路径：只执行已经批准的 contract，不调用 LLM 动态改规则。

这可以显著降低新增大量异构站点时的 selector 运维成本，同时保持确定性、可审计和低推理成本。

### 9.2 增加可选 Search Index Source Discovery Provider

在 probing/诊断阶段允许用通用搜索索引帮助发现站点博客入口、归档页、历史年份页、Feed/Sitemap 入口和平台托管域，但：

- 只作为 Source candidate；
- 默认低频运行；
- 有严格费用和请求预算；
- 不能用搜索结果数量声明“全量完成”；
- 所有候选 URL 都必须重新经过 scope、Egress、robots、approved host 和当前站点验证。

## 10. 不应照搬的部分

1. **SQLite 作为主状态库**：无法支撑多 worker durable job、lease、outbox 和长期高并发运维。
2. **本地 HTML cache 作为核心缓存/状态**：缺少一致性和 provenance。
3. **每 URL 创建 Crawl4AI 实例**：浏览器生命周期成本过高。
4. **同步 `sleep` 与 `asyncio.run` 混用**：无法形成高效异步 worker。
5. **全量 `resp.read()`**：缺少输入大小和解压预算。
6. **简单 host 字符串比较**：不能承担安全边界和真正的 registrable domain 判断。
7. **LLM 在线自动修 selector 并立即生效**：可能由异常页面导致规则污染。
8. **LLM 失败即判数据无效**：把“验证器不可用”和“内容无效”混为一谈。
9. **SERP 作为全量发现来源**：结果有排名/索引偏差，不可能证明完整性。
10. **固定极小翻页上限**：只能用于 probing，不适用于历史回灌。

## 11. 推荐落地方式

将此次调研结论落实到现有系统时，不增加新的“AI 抽取 Worker”作为必经路径，而是扩展控制面：

```text
Site probing
 -> representative snapshots
 -> DOM/page-type clustering
 -> existing Profile/selector match
 -> if coverage insufficient:
      Schema Synthesis candidate
      -> deterministic validation on golden snapshots
      -> quality metrics
      -> human/canary approval
      -> publish Extraction Contract release
 -> backfill uses approved deterministic rules
 -> failures aggregate as drift evidence
 -> candidate repair -> shadow replay -> new release
```

这样可以同时获得该项目“自动适配陌生网页结构”的优势，以及现有方案对 1000 站点长期运行所要求的可扩展性、增量同步、可审计、可重放、可回滚和 Web 管理能力。

## 12. 结论

该项目最有价值的技术原理是：**AI 负责发现和生成声明式抽取知识，确定性引擎负责执行，版本化 Schema 负责复用**。对于大规模博客知识库，应进一步把 Schema 生成从在线脚本提升为受控的 Rule Compiler 和 release 流程，并把 SERP 降级为低频 Source Discovery 辅助能力。

不建议照搬其单机 SQLite/cache、逐 URL Crawl4AI 生命周期、同步阻塞、在线 LLM 自修复和 LLM 校验硬依赖。经过上述改造后，AI 的使用频率从“每篇文章”降到“每个模板变化”，可显著降低成本，也更符合长期增量同步和可运维性要求。
