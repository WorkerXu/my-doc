# Python Crawl4AI 基本用法：Anthropic News 多文章批量抓取实践

## 1. 调研对象

- 编号：58
- 原文：https://juejin.cn/post/7556863932343287844
- 主题：Crawl4AI 基本抓取、Markdown 生成、Pruning/BM25 内容过滤，以及 `arun_many()` 多 URL 并发抓取
- 相关官方文档：
  - https://docs.crawl4ai.com/advanced/multi-url-crawling/
  - https://docs.crawl4ai.com/api/arun_many/
  - https://docs.crawl4ai.com/core/fit-markdown/
  - https://docs.crawl4ai.com/core/markdown-generation/
  - https://docs.crawl4ai.com/api/parameters/

这篇文章属于教程型实践，示例规模不大，但它把“多个真实文章 URL -> Crawl4AI 批量抓取 -> Markdown 内容过滤 -> 文件落盘”这一条最短链路展示得比较清楚。对 1000 个技术博客知识库项目而言，真正值得吸收的不是把示例代码直接放大，而是明确三个边界：批量抓取只是 Worker 内执行机制，Markdown 过滤是派生视图，增量同步不能靠每次禁用缓存并全量重抓。

## 2. 原文实现链路与技术含义

原文主要包含以下能力：

1. 使用 Crawl4AI 抓取网页并生成 Markdown；
2. 使用 `PruningContentFilter` 过滤低价值 DOM 内容；
3. 使用 `BM25ContentFilter` 按查询相关性保留内容；
4. 对多个 Anthropic News URL 使用 `arun_many()` 批量抓取；
5. 将过滤后的 Markdown 写入本地文件。

从工程角度可抽象为：

```text
URL List
  -> CrawlerRunConfig
  -> AsyncWebCrawler.arun_many()
  -> CrawlResult per URL
  -> raw_markdown / fit_markdown
  -> local markdown file
```

教程中的链路适合作为单机原型，但知识库平台必须进一步拆成：

```text
Durable Task
  -> Fetch Route
  -> Worker-local Batch Dispatcher
  -> Fetch Attempt
  -> Snapshot
  -> Extraction Candidate
  -> Canonical IR / Document Version
  -> Markdown Projection
  -> Filtered Markdown Projection
  -> Chunk / Index / Embedding
```

关键区别是：教程直接把抓取结果当最终文件；生产系统必须把网络事实、正文事实和派生 Markdown 分开保存。

## 3. `arun_many()` 的实现原理与正确边界

### 3.1 它解决的是单 Worker 内并发

Crawl4AI 官方文档当前将 `arun_many()` 定义为多 URL 批处理入口，并提供 Dispatcher、资源监控、限流和流式返回能力。它非常适合解决一个 Browser Worker 或 Crawl4AI Worker 内部的并发控制问题。

典型执行模型：

```text
worker lease N tasks
  -> arun_many(urls, stream=True)
  -> dispatcher 控制并发与内存
  -> URL A 完成 -> 立即持久化并 ack A
  -> URL C 完成 -> 立即持久化并 ack C
  -> URL B 失败 -> 单独 retry B
```

这里应该启用 partial success，而不是等待整个批次全部成功后再提交。

### 3.2 它不能替代平台调度器

1000 个站点意味着同时存在：

- 不同域名的 robots 与访问策略；
- 不同站点的 QPS、并发和失败率；
- HTTP 与 Browser 的成本差异；
- FULL_BACKFILL、INCREMENTAL、RECONCILE 等不同优先级；
- Worker 崩溃后的任务恢复；
- 跨节点重试、checkpoint、幂等与审计。

因此需要两层调度：

```text
平台 Scheduler
  global token bucket
  + registrable-domain token bucket
  + source token bucket
  + route budget
  + durable lease/retry/checkpoint
        ↓
Worker
  arun_many()
  + MemoryAdaptiveDispatcher / SemaphoreDispatcher
  + RateLimiter
  + stream=True
```

平台 Scheduler 决定“谁应该什么时候抓”；Crawl4AI Dispatcher 只决定“当前 Worker 如何安全并发执行”。

### 3.3 批量不是越大越好

批次大小必须同时考虑：

- Worker 可用内存；
- Browser context/page 数；
- 单页 DOM 大小；
- 页面平均响应时间；
- 单域名并发上限；
- 返回结果持久化速度；
- Object Storage 写入吞吐；
- 下游 Extraction backlog。

因此不能固定 `batch_size=1000`。建议采用“平台 lease 小批次 + Worker 自适应调度 + 单结果流式提交”。

## 4. `raw_markdown` 与 `fit_markdown` 的本质区别

官方文档明确区分：

- `raw_markdown`：由清洗后 HTML 转换得到的较完整 Markdown；
- `fit_markdown`：启用内容过滤器后得到的裁剪版 Markdown；
- `fit_html`：生成 `fit_markdown` 的过滤后 HTML。

这对知识库非常重要，因为 `fit_markdown` 不是“更正确的原文”，而是一个经过算法删减的派生结果。

生产系统建议定义：

```text
Snapshot                 = 不可变网络事实
Canonical IR             = 正文事实
CLEAN_MARKDOWN            = 从 IR 确定性生成
CRAWL4AI_RAW_MARKDOWN     = 抽取候选/对照产物
FILTERED_MARKDOWN         = 可重建检索派生物
BM25_QUERY_VIEW           = query-dependent 临时/缓存视图
```

任何 Filter 都不能覆盖 Snapshot、Canonical IR 或 canonical Markdown。

## 5. PruningContentFilter 的技术原理

Crawl4AI 的 Pruning 过滤是 query-independent 的启发式裁剪。官方文档描述的核心评分因素包括：

- 文本密度；
- 链接密度；
- DOM 标签重要度；
- 节点结构上下文；
- `min_word_threshold`；
- 固定或动态 threshold。

可以把它理解为给 DOM block 计算一个内容价值分数：

```text
score(block) =
  + text_density
  - link_density_penalty
  + tag_semantic_weight
  + structural_context_weight
```

然后根据 threshold 丢弃低分节点。

它的优点：

- 不需要 LLM；
- 不需要用户 query；
- 成本低，适合批量预处理；
- 对导航、广告、侧栏等低密度区域通常有效。

但它仍然是有损过滤，存在以下风险：

- 短代码片段可能被当作低字数块删除；
- API 参数表、列表或短定义可能被误删；
- 技术博客中链接密集的参考资料区域可能被低估；
- threshold 对不同模板不能一刀切；
- 中文“词数”与英文 whitespace word count 的行为不同。

因此 Pruning 只能作为 Projection 或 Extraction Candidate，不能直接成为唯一归档版本。

## 6. BM25ContentFilter 的技术原理与局限

BM25 本质是 query-to-document/block 的词项相关性排序。简化表达：

```text
score(D, Q) = Σ IDF(q) * tf(q, D) * (k1 + 1)
              ----------------------------------
              tf(q, D) + k1 * (1 - b + b * |D|/avgdl)
```

因此 BM25ContentFilter 天生是 query-dependent：换一个 query，同一网页保留下来的内容可能完全不同。

这使它非常适合：

- 临时围绕某主题提取重点；
- 给 LLM 减少输入 token；
- RAG 查询阶段生成局部候选视图。

但不适合：

- 作为文章 canonical Markdown；
- 用于内容版本 hash；
- 用于判断源站正文是否变化；
- 直接覆盖完整历史文章。

原文还提到 BM25 对中文效果不理想。工程上不能把这一点简单理解为“BM25 不适合中文”，真正问题通常来自 tokenizer/analyzer。中文没有天然空格分词，如果过滤器内部的词法处理与语言不匹配，召回与相关性就会明显下降。因此知识库平台需要单独管理 Analyzer/Language Policy，不能复用英文默认词法假设。

## 7. 内容过滤需要独立 Release

现有方案应增加 `content_filter_release`，不要把参数隐含在业务代码或临时 CrawlerRunConfig 中。

建议数据结构：

```text
content_filter_release
  id
  version
  strategy                  # NONE / PRUNING / BM25 / LLM / CUSTOM
  threshold
  threshold_type
  min_word_threshold
  query_mode                # NONE / FIXED / RUNTIME_QUERY
  fixed_query
  language_scope
  input_projection_type
  output_projection_type
  runtime_artifact_release_id
  benchmark_result_id
  created_at
```

这样可以回答：某个 filtered Markdown 到底由哪种算法、哪个 threshold、哪一版 Crawl4AI、哪种语言策略生成。

## 8. Markdown Filter 的质量门禁

仅检查“文件生成成功”不够。建议为过滤后 Markdown 增加质量特征：

```text
content_retention_ratio
heading_retention_ratio
code_block_retention_ratio
table_retention_ratio
list_retention_ratio
link_retention_ratio
reference_section_retained
language_consistency
truncation_signal
```

其中：

```text
content_retention_ratio = len(filtered_text) / len(canonical_text)
```

该比例不是越高越好，而是用于发现异常。例如某模板历史基线在 0.65~0.80，突然降到 0.12，就可能是 filter threshold、DOM 结构或正文 selector 漂移。

对于技术文章，代码、表格和 heading 的保真度应设置独立门禁，避免一个看似“很干净”的 Markdown 实际已经丢失技术细节。

## 9. CacheMode 与增量同步不能混为一谈

教程为了“拿最新内容”常使用 `CacheMode.DISABLED` 或 BYPASS。这在手工测试中合理，但不能作为 1000 站点长期同步策略。

生产增量同步应优先：

```text
Feed/API cursor
  -> Sitemap lastmod
  -> Conditional GET (ETag / Last-Modified)
  -> Metadata Probe
  -> Full HTTP fetch
  -> Browser fallback
```

如果服务端返回 304：

- 不新增正文 Snapshot；
- 不重复抽取；
- 不创建 Document Version；
- 不重新生成 Markdown/Chunk/Embedding；
- 只记录 observation/fetch attempt。

这样才能把“是否访问源站”“是否下载正文”“是否重建派生物”拆开，避免每天对数十万篇历史文章做无意义全量重抓。

## 10. HTTP First 与 Crawl4AI Browser 的关系

原文以 Crawl4AI 作为统一入口很方便，但平台级方案仍应坚持 HTTP First。

对于普通技术博客正文：

```text
HTTPX/aiohttp
  -> HTML snapshot
  -> deterministic extractors
```

只有以下情况再进入 Browser：

- 正文依赖 JavaScript 渲染；
- Archive/Load More 只能通过交互发现 URL；
- HTTP 正文质量低于阈值；
- 需要执行明确、受控的 Interaction Recipe。

Browser 应独立 Worker 池扩容。`arun_many()` 的资源自适应能力非常适合这一层，但不能把所有静态文章都 Browser 化。

## 11. 对 1000 站点方案的具体优化

基于本篇调研，主方案应新增或强化以下能力：

### 11.1 新增 Content Filter Release

Pruning、BM25、LLM Filter 都版本化，记录参数、语言范围、运行时和 benchmark。

### 11.2 明确 Markdown 三层

```text
Canonical Markdown
  = 从 Canonical IR 确定性生成

Crawl4AI Raw Markdown
  = 抽取候选 / 对照结果

Filtered/Fit Markdown
  = 检索或 LLM 输入优化 Projection
```

即使 Pruning 不依赖 query，也不能直接升级为 canonical。

### 11.3 Worker 使用流式批处理

Browser/Crawl4AI Worker 从平台 lease 一小批 Task，使用 `arun_many(stream=True)`，每个 URL 成功即写 Snapshot、提交 Task；失败 URL 单独 retry。

### 11.4 按 URL 类型匹配 CrawlerRunConfig

Crawl4AI 官方支持为不同 URL 匹配不同 Run Config。平台可以将这一能力映射到 Site Profile/Fetch Route：

- PDF -> PDF scraping strategy；
- article -> raw markdown + optional pruning projection；
- dynamic hub -> JS interaction recipe；
- JSON/API -> structured extraction。

但配置真相仍应保存在数据库的 Release 中，而不是 Worker 内硬编码。

### 11.5 过滤质量可视化

Web 管理增加：

- raw/canonical/fit 三栏对比；
- threshold 调整预览；
- retention ratio；
- code/table/heading 丢失提示；
- Source 模板的历史基线；
- Filter Release A/B；
- 一键回滚到上一版 Filter Release。

## 12. 推荐执行流水线

```text
[Discovery]
  provider 枚举 URL
      ↓
[Scheduler]
  durable task + domain/source budget
      ↓
[Fetch Route]
  HTTP first
      ├─ success/high quality -> Snapshot
      └─ dynamic/low quality -> Browser/Crawl4AI
                                ↓
                      arun_many(stream=True)
                                ↓
                           Snapshot
                                ↓
[Extraction]
  Selector / JSON-LD / Trafilatura / Readability / Crawl4AI raw
                                ↓
                      Candidate Quality
                                ↓
                         Canonical IR
                                ↓
                       Document Version
                                ↓
[Projection]
  CLEAN_MARKDOWN
  CRAWL4AI_RAW_MARKDOWN
  FILTERED_MARKDOWN(PRUNING)
  BM25_QUERY_VIEW
  CHUNK / EMBEDDING / INDEX
```

## 13. 结论

这篇教程最值得吸收的不是把 `arun_many()` 直接作为系统调度器，也不是把 `fit_markdown` 直接当知识库正文，而是把 Crawl4AI 的两类能力放到正确层级：

1. `arun_many()` 是 Worker 内资源感知、多 URL 并发执行器；
2. Pruning/BM25 是内容过滤 Projection，不能替代原始事实与 canonical 内容；
3. Pruning 虽然 query-independent，仍然有损，必须版本化并做质量门禁；
4. BM25 Filter 是 query-dependent，不允许参与 canonical 文档身份和版本判断；
5. 1000 站点增量同步应依赖 durable scheduler、条件请求和 reconcile，而不是持续禁用缓存全量重抓；
6. Web 管理需要支持 raw/canonical/fit 的可视化比较、Filter Release 调参和回滚。

这些调整能让 Crawl4AI 在方案中从“单机抓取工具”变成可被审计、可批处理、可灰度、可回放的执行组件，同时保持知识库事实层的长期稳定性。
