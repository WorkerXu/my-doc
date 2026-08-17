# 基于 Crawl4AI 的 MCP + RAG 系统

来源项目：<https://github.com/coleam00/mcp-crawl4ai-rag>

## 1. 项目定位

该项目把 Crawl4AI、FastMCP、Supabase/pgvector、OpenAI Embedding、可选 CrossEncoder rerank 与可选 Neo4j Knowledge Graph 组合成一个面向 AI Agent/AI Coding Assistant 的“抓取 + 入库 + RAG”服务。核心路径是：

```text
URL
 -> smart_crawl_url 判断 URL 类型
 -> Crawl4AI 抓取
 -> Markdown
 -> 分块
 -> Embedding
 -> Supabase/pgvector
 -> Vector/Hybrid Search
 -> 可选 CrossEncoder rerank
 -> MCP 暴露查询工具
```

它是一个非常适合验证“抓完即可搜”的原型，但它的状态模型、递归发现、增量同步、安全边界和写入原子性还不足以直接承担 1000 个站点、数百万 URL、长期运行的生产知识库。对我们的方案最有价值的是：**资源自适应并发、上下文增强 Embedding、代码示例专用检索、Rerank 分层**；最需要避免的是：**进程内 Frontier、整站结果聚合后一次入库、删除旧 chunk 后再插新 chunk、把增强后的 embedding 文本直接当正文保存**。

## 2. 核心实现结构

### 2.1 FastMCP 生命周期

`src/crawl4ai_mcp.py` 通过 `crawl4ai_lifespan()` 创建并复用：

- `AsyncWebCrawler`；
- Supabase Client；
- 可选 `CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")`；
- 可选 Neo4j validator / repository extractor。

这比每个请求重新启动浏览器要合理：Browser 生命周期与 MCP server 生命周期绑定，可以减少 Chromium 冷启动。但生产环境仍需要进一步拆成 Browser Worker pool，因为单 MCP 进程同时承担抓取、Embedding、数据库和查询，故障域和资源隔离都太大。

### 2.2 URL 类型路由

`smart_crawl_url()` 按 URL 类型选择策略：

```text
.txt
 -> crawl_markdown_file

sitemap
 -> parse_sitemap
 -> crawl_batch

普通网页
 -> crawl_recursive_internal_links
```

这种“先识别 source 类型，再使用不同抓取策略”的思路值得保留，但生产化应该从“字符串后缀判断”升级成独立 `DiscoveryProvider`：Feed、Sitemap、Ordered Archive、普通 Archive、Deep Crawl、llms.txt、Common Crawl/Wayback 等分别维护 checkpoint、预算和 end condition。

### 2.3 Sitemap 解析

项目的 `parse_sitemap()` 使用同步 `requests.get()` + `ElementTree.fromstring()`：

- 一次把整个 sitemap 下载进内存；
- 一次构造完整 XML tree；
- 没有流式解析；
- 没有 gzip/decompression ratio 限制；
- 没有嵌套 sitemap checkpoint；
- 没有请求超时、robots、域级 limiter 或 SSRF/redirect hop 校验；
- 该同步请求还运行在 async MCP 工具调用路径上，会阻塞事件循环。

所以它适合小型文档站，不适合作为大站历史 URL 发现主实现。我们的生产方案应继续使用 streaming HTTP + bounded decompressor + secure XMLPullParser/iterparse + child sitemap checkpoint。

## 3. Crawl4AI 并发机制

### 3.1 `MemoryAdaptiveDispatcher`

`crawl_batch()` 与 `crawl_recursive_internal_links()` 都使用：

```python
MemoryAdaptiveDispatcher(
    memory_threshold_percent=70.0,
    check_interval=1.0,
    max_session_permit=max_concurrent
)
```

该机制的价值是：`max_concurrent` 只是上限，真实并发可以随着进程内存压力动态收缩。对于 Browser/HTML 重页面，固定并发非常容易出现“平时正常、少数大页面导致 OOM”的情况，因此“资源压力决定局部并发”比单纯 semaphore 更稳。

### 3.2 应吸收到生产方案的方式

不能直接把 `MemoryAdaptiveDispatcher` 当全局调度器。生产环境需要两级甚至三级限制：

```text
全局 Scheduler / Worker 类型配额
 -> domain/site limiter
 -> worker 本地 Resource-Adaptive Dispatcher
```

其中：

- 全局 Scheduler 负责优先级、公平性和总资源预算；
- domain limiter 负责对源站克制；
- 本地 dispatcher 只负责当前 worker 的 RSS/CPU/browser slot 压力；
- 本地 dispatcher 只能“继续收缩”，不能突破全局/domain 上限。

建议 worker heartbeat 暴露：RSS、CPU、active browser pages、event-loop lag、open files、object-store latency，并将本地并发动态值写成可观测指标，而不是固定 `70%` 的魔法数。

## 4. 递归发现实现与局限

`crawl_recursive_internal_links()` 是分层 BFS：

```text
current_urls
 -> arun_many
 -> result.links["internal"]
 -> next_level_urls
 -> 下一层
```

它使用 `visited = set()` 去重，并通过 `urldefrag()` 去 fragment。这种实现简单清晰，但对我们的目标有几个根本问题：

1. `visited` 只在进程内，进程重启后全部丢失。
2. 没有 durable checkpoint，无法从数十万 URL 中间可靠恢复。
3. 所有 URL 身份与状态没有进入数据库 Frontier。
4. 没有记录“这个 URL 是从哪个页面、Sitemap、Archive 被发现”的 provenance graph。
5. 没有 URL admission pipeline，也没有 approved host/scope 的强约束。
6. 没有增量语义：下一次重新运行基本等于重新遍历。
7. 深度和当前集合仍以进程内 set 承载，不适合百万 URL。

因此生产实现应继续坚持：**Crawl4AI 只负责一批 URL 的网络/浏览器执行，不负责整个站点的业务 Frontier 真相**。发现出的 URL 必须流式 UPSERT PostgreSQL Frontier，并由 Redis/lease 驱动下一批。

## 5. Markdown 分块实现

`smart_chunk_markdown()` 默认按约 5000 字符分块，优先尝试：

1. 最近的 ``` 边界；
2. 段落空行；
3. 英文句号。

这比机械固定长度切割更好，但仍然是字符级启发式，不理解 Markdown AST。尤其 `rfind('```')` 只是寻找最后一个 fence 标记，没有维护 fence 的开闭状态，也无法理解嵌套/语言、表格、列表和 heading hierarchy。

生产知识库应保留项目的“不要随意切断代码”的目标，但实现改成：

```text
canonical Markdown AST / Article IR
 -> heading section
 -> paragraph/list/blockquote/code/table block
 -> 超长 section 再按 token budget 切
```

代码块、表格默认是原子 block；chunk identity 应由 `article_version_id + chunker_version + heading_path + chunk_index/text_hash` 决定，而不是 URL + 当前切块编号。

## 6. Supabase / pgvector 数据模型

项目创建：

```text
sources
crawled_pages
code_examples
```

`crawled_pages` 使用：

```sql
unique(url, chunk_number)
embedding vector(1536)
```

并用 pgvector cosine distance 查询。这是一个可用的 RAG MVP，但它把“抓取结果”“检索 chunk”“source summary”混在一个数据库层，缺少 canonical article/version、snapshot、extract attempt、index generation 等中间层。

### 6.1 最大问题：先删后插

`add_documents_to_supabase()` 会先：

```python
DELETE FROM crawled_pages WHERE url IN (...)
```

然后重新生成 Embedding，再批量 INSERT。

这有明显写入窗口：

```text
删除旧版本成功
 -> Embedding/API/DB 插入失败
 -> 当前 URL 在检索库暂时或永久消失
```

并且 chunk boundary 一变化，原有 chunk_number 全部漂移，不利于增量索引和审计。

生产方案不应该照搬。应使用：

- canonical `article_version` 不可变；
- 新 index generation/staging chunk 先完整构建；
- 校验成功后原子切 active generation / desired state；
- 老 generation 延迟清理；
- 单 article 更新也应以 version 为粒度 UPSERT 新派生记录，不先破坏旧可用状态。

## 7. Contextual Embedding

项目支持 `USE_CONTEXTUAL_EMBEDDINGS=true`。对每个 chunk，把“全文前 25000 字符 + 当前 chunk”交给 LLM，让它生成简短 context，然后：

```text
context
---
chunk
```

再生成 embedding。

### 7.1 原理

普通 chunk embedding 容易丢掉文档级语义。例如一个 chunk 里只有：

```text
它默认是 10 秒。
```

如果没有 heading/全文背景，向量无法知道“10 秒”指 timeout、cache TTL 还是 retry delay。给 chunk 加入文章标题、heading path 或简短 document context 后，向量召回会更稳定。

### 7.2 项目实现中的一个重要问题

项目的 `add_documents_to_supabase()` 实际把 `contextual_contents[j]` 写入 `content` 字段，也就是增强后的文本进入正文检索结果，而不是只作为 embedding 输入。这会污染原始内容语义。

此外，并发 contextualization 使用 `as_completed()` 直接 `append` 到列表，没有按原始 index 写回固定位置；只在“结果数量不等”时 fallback，并没有在“数量相等但顺序已乱”时重排。因此并发完成顺序可能与 URL/chunk metadata 顺序错位。这是一个潜在数据错配风险。

### 7.3 生产化改法

必须分离：

```text
canonical_chunk_text      # 用户真正看到的正文
embedding_input_text      # 只用于 embedding
contextual_summary        # 可选派生字段
```

建议 embedding input 默认：

```text
文章标题
站点/作者/时间（少量）
heading_path
可选 contextual summary
canonical chunk
```

记录：

```text
contextualizer_version
contextualizer_model
prompt_version
embedding_input_hash
fallback_reason
```

LLM contextualizer 失败时退化成 `title + heading_path + chunk`，绝不能阻塞 canonical Markdown 发布。

## 8. Code Example 专用检索

项目在 `USE_AGENTIC_RAG=true` 时抽取 fenced code block，生成周边语境摘要并写入独立 `code_examples` 表，再提供 `search_code_examples` MCP 工具。

这对技术博客知识库非常有价值，因为用户经常不是想“搜一段解释”，而是想：

- 找某 API 的具体调用例子；
- 找完整配置片段；
- 找错误处理/重试代码；
- 找框架初始化方式。

普通 prose chunk 与 code chunk 混在一个向量空间时，代码 token、变量名和自然语言语义会互相干扰。

### 8.1 生产化改法

从 canonical Markdown AST 直接生成 `code_units`，不要再用字符串寻找 ```：

```text
code_units
- id
- article_version_id
- heading_path
- language
- code_sha256
- canonical_code_ref
- context_before_ref
- context_after_ref
- summary_ref nullable
- metadata_json
```

再建立独立 lexical/vector generation。查询层允许：

```text
search_articles(...)
search_code_examples(...)
```

其中 code summary 是 enrichment，可失败；原始 code block 必须来自 canonical AST，不被 LLM 改写。

项目 README 写“代码块 >=300 字符”，而 `utils.py` 默认 `min_length=1000`，说明配置/文档也可能漂移。生产系统应把 threshold 放进版本化 retrieval profile，而不是散落在代码常量和文档中。

## 9. Hybrid Search 与 Rerank

项目的 Hybrid Search：

1. 向量检索取 `2 * match_count`；
2. `ILIKE '%query%'` 做关键词匹配；
3. 同时命中时把 similarity 乘 1.2；
4. 纯关键词结果给固定 similarity=0.5；
5. 可选 CrossEncoder rerank。

优点是实现简单，确实能覆盖函数名/错误信息等精确词。问题是：

- `ILIKE` 是整段 substring，不等于成熟 BM25/FTS；
- vector cosine 与人工 0.5、乘 1.2 没有可比较的统计意义；
- 结果融合对 query 词序、语言、分词都比较脆弱。

我们的方案应继续使用：

```text
Lexical(BM25/PG FTS)
 + Vector
 -> RRF
 -> 可选 top-N CrossEncoder rerank
```

CrossEncoder 的思想值得吸收，但要做成版本化 `reranker_generation`，限定候选数和 p95 latency，并能降级回 RRF。

## 10. 增量同步能力缺口

项目抓取配置普遍使用 `CacheMode.BYPASS`，没有完整利用：

- ETag；
- Last-Modified；
- If-None-Match；
- If-Modified-Since；
- 304；
- raw response hash；
- provider checkpoint；
- reconciliation；
- 删除状态机。

所以它更像“主动重建当前索引”，而不是长期增量同步系统。对于 1000 站点，如果每轮都重新递归发现 + 全量重抓 + 全量 embedding，网络成本、源站压力和 API 成本都会不可接受。

我们的方案继续保持：发现增量与页面增量分别 checkpoint，页面 304/same raw hash 直接停止，只有 canonical 输出真的改变才创建新 `article_version` 和索引事件。

## 11. 安全边界缺口

项目作为公开示例没有完整生产防线：

- 任意 URL 抓取工具；
- sitemap `requests.get()`；
- 未统一做 redirect hop SSRF 校验；
- 未实现 public-only Browser egress；
- 未展示 robots 与 domain politeness；
- SQL 示例给 `crawled_pages/sources/code_examples` 创建 public read policy；
- MCP 抓取能力与检索能力在同一 server 中。

生产系统应把“管理/抓取 MCP”与“只读知识 MCP”彻底分离，最终面向普通 AI Client 的 MCP 只开放 search/list/get，不允许任意 crawl、delete、raw snapshot 或 admin 操作。

## 12. 对现有技术方案的优化结论

本项目不会改变现有方案“PostgreSQL 真相源 + Redis durable queue + S3 immutable snapshot + article_version canonical + Search/Publish 派生层”的主架构，但可以增加四项有明确收益的能力。

### 12.1 增加 Worker 本地 Resource-Adaptive Dispatcher

借鉴 Crawl4AI `MemoryAdaptiveDispatcher`，在全局调度和域级 limiter 之下增加 worker 本地资源压力门禁：

```text
Scheduler quota
 -> domain limiter
 -> local adaptive dispatcher
 -> actual fetch/browser task
```

当 RSS、Browser process memory、event-loop lag 或对象存储写延迟升高时自动收缩，而不是等待 OOM 后重启。

### 12.2 增加 Contextual Embedding，但不污染 canonical chunk

用标题、heading path、文章级短 context 构造 embedding input，保留 canonical chunk 原文。contextualizer、prompt、model 全部版本化并可离线重放。

### 12.3 增加 Code Search 派生面

从 Article IR/Markdown AST 提取代码 block 形成 `code_units`，建立独立 lexical/vector index；MCP 可选提供 `search_code_examples`。这样技术博客知识库更适合 coding assistant 使用。

### 12.4 增加可选 CrossEncoder Rerank Generation

Hybrid RRF 之后对 top 20~100 候选做 rerank，作为可关闭、可回滚、可评估的派生阶段。任何 reranker 不可用时降级，不影响 canonical 内容与基础搜索。

## 13. 最终评价

这个项目非常适合作为“Crawl4AI + RAG”能力参考，不适合作为 1000 站点生产调度框架直接复用。

可以直接借鉴的不是它的数据库表或 `smart_crawl_url()` 外壳，而是以下设计思想：

```text
资源压力感知并发
上下文增强向量
代码示例独立检索
Hybrid 后可选 Rerank
```

必须由我们的平台补齐：

```text
Durable Frontier
Provider Checkpoint
HTTP conditional GET
Immutable Snapshot
Article Version
Generation-based Index
Transactional Outbox
Domain Fairness
Robots / Egress / SSRF
Web Control Plane
Reconciliation
```

因此对主技术方案的正确调整是“把这些 RAG/资源调度能力作为 article_version 之后或 worker 本地的可插拔模块”，而不是把 Supabase RAG server 反向变成抓取业务真相源。
