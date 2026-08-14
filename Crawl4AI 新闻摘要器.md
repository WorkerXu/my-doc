# Crawl4AI 新闻摘要器

## 1. 调研对象

- 编号：54
- 项目：`cf2018/crawl4ai_news_summarizer`
- 地址：https://github.com/cf2018/crawl4ai_news_summarizer
- 项目形态：Flask Web + Crawl4AI 深度抓取 + Gemini 分析
- 关键依赖：Crawl4AI 0.7.0、Playwright 1.53.0、Flask 3.1.1、google-genai 1.25.0
- 核心代码：`app/crawler.py`、`app/routes.py`、`app/genai.py`

本项目不是面向大规模长期同步设计的爬虫平台，而是一个“输入站点 + 主题 Prompt，优先抓取相关页面，再交给 Gemini 汇总”的轻量原型。它最有价值的部分不是直接复用其代码，而是验证了一个适合加入博客知识库平台的辅助能力：**相关性优先发现（Best-First）可以显著缩短站点接入、人工探索和专题抓取的首屏等待时间，但它必须与历史完整性枚举严格隔离，不能承担 FULL_BACKFILL 的 Coverage 判定。**

## 2. 实际执行链路

项目的主流程可以还原为：

```text
用户提交 URL + prompt + depth + Gemini Token
  -> Flask POST 请求
  -> run_deep_crawl()
  -> asyncio.run(deep_crawl())
  -> AsyncWebCrawler
  -> BestFirstCrawlingStrategy
     -> DomainFilter
     -> URLPatternFilter
     -> ContentTypeFilter
     -> SEOFilter
     -> KeywordRelevanceScorer
  -> stream=True 持续产生 CrawlResult
  -> 在进程内收集全部结果 markdown
  -> "\n\n" 拼成一个大 Context
  -> Gemini 2.0 Flash Lite
  -> 同一个 HTTP 请求返回页面
```

抓取配置核心为：

```python
BestFirstCrawlingStrategy(
    max_depth=max_depth,
    max_pages=max_pages,
    filter_chain=filter_chain,
    url_scorer=scorer,
)
```

并设置 `stream=True` 与 `page_timeout=30000`。默认 `max_depth=2`、`max_pages=15`。

这意味着它的目标不是“把站点所有文章枚举出来”，而是在有限页面预算内，尽快找出更像用户 Prompt 的 URL。

## 3. Best-First 的技术原理与适用边界

Crawl4AI 的 Best-First 深度抓取会维护待访问 URL frontier，并对发现到的 URL 计算 score，优先访问高分 URL。相比 BFS 按层扫描、DFS 沿分支深入，Best-First 更适合：

- 专题研究；
- 站点初始探测；
- 快速找到疑似文章页；
- 交互式 Web 搜索；
- 在固定预算内优先取得高价值样本。

但是 `max_pages` 与 `max_depth` 本质上是**执行预算**，不是完整性证据。对 1000 站点全量历史归档，如果把 Best-First 的“队列跑完/达到 15 页”当成历史抓取完成，会系统性漏掉：

- URL 不含主题关键词的旧文章；
- 低 SEO 分但正文有效的文章；
- 目录层级较深的历史页；
- 年月归档中的冷门文章；
- 没有规范 meta/canonical/schema 的自建博客；
- Prompt 词汇与 URL slug 不一致的页面。

因此在知识库方案中，Best-First 应只承担“优先级”，不能承担“是否存在”的判断。

## 4. 项目中几个容易被误解的实现细节

### 4.1 KeywordRelevanceScorer 实际只检查 URL 字符串

项目使用：

```python
KeywordRelevanceScorer(keywords=prompt.split(), weight=0.8)
```

Crawl4AI 0.7.0 的实现并不会读取正文、标题或 anchor context，而是把 URL 转小写后，统计关键词是否直接出现在 URL 字符串中，然后按命中词比例计算分数。

因此：

```text
Prompt = "distributed systems"
URL = /posts/raft-consensus
```

即便正文高度相关，只要 URL 中没有 `distributed` 或 `systems`，该 scorer 仍可能给 0 分。

这说明生产方案不能把该 score 命名成“内容相关度”。更准确的语义是 `url_keyword_priority_score`，只能作为调度 hint。

### 4.2 URLPatternFilter 中的 `*` 使后续模式失去筛选意义

项目配置：

```python
URLPatternFilter(patterns=["*", "*blog*", "*article*", "*news*"])
```

Crawl4AI 0.7.0 的 `URLPatternFilter` 对多个 pattern 采用“任一匹配即通过”。其中 `*` 能匹配所有 URL，因此该配置实际上不会把 URL 限制到 blog/article/news 路径。

正确做法应二选一：

1. 真正做硬过滤时移除 `*`，只保留经过站点验证的 pattern；
2. 更推荐在通用知识库中把路径 pattern 变成**加分项而非排除条件**，避免未知站点的文章 URL 不含 `blog/article/news` 时被漏掉。

### 4.3 SEOFilter 会为候选 URL 额外读取 `<head>`

Crawl4AI 0.7.0 的 `SEOFilter.apply()` 会调用 `HeadPeekr.peek_html(url)` 读取候选页面 head，再依据以下因素打分：

- title 长度；
- title 是否包含关键词；
- meta description 长度；
- canonical；
- robots/noindex；
- schema.org JSON-LD；
- URL 结构。

项目阈值是 `0.3`。

这带来两个重要结论：

**第一，SEOFilter 不是零成本的本地 URL 过滤。** 对 frontier 中每个候选 URL 都可能产生额外网络 I/O。1000 个站点、百万级 URL 下，如果直接放在 Discovery 热路径，会造成额外连接、源站压力和抓取成本。

**第二，SEO 好坏不等于知识库内容有效性。** 很多老技术博客、自建文档站、早期静态站 title/meta/schema 不规范，但正文非常重要。FULL_BACKFILL 不能因为 SEO score 低而丢弃页面。

因此生产设计应把这种 head probe 变成显式 `METADATA_PROBE`，受预算、缓存、限流、快照复用和 Release 管理，而不是隐藏在通用 FilterChain 中。

### 4.4 ContentTypeFilter 在这里主要是 URL 扩展名预过滤

Crawl4AI 0.7.0 的 `ContentTypeFilter` 首先通过 URL 扩展名推断类型；没有扩展名时通常放行。因此它适合快速排除明显的图片、压缩包等 URL，但不能替代真实 HTTP `Content-Type` 校验。

生产链路仍需在 fetch 结果上执行 MIME sniff、响应头校验与安全限制。

## 5. Streaming 的价值与项目实现上的浪费

项目设置了 `stream=True`，这是正确方向。Best-First 与 streaming 组合可以让高分页面先返回，适合交互式 UI。

但项目随后又在 Flask 请求线程内：

1. 把每个结果 append 到内存数组；
2. 等深度抓取结束；
3. 才把 Markdown 全部传给 Gemini；
4. 最终一次性渲染 HTML。

因此它并没有真正把 streaming 的收益传给用户，也没有降低最终内存峰值。

大规模平台应改为：

```text
Crawler stream
 -> 每页结果写 Snapshot / Candidate / Task Outcome
 -> 发布 progress event
 -> Web 通过 SSE/WebSocket 查询进度
 -> Quality/Identity 异步执行
 -> 达到可用条件即可先展示前 N 个 Accepted Version
```

这样既保留“高价值结果先到”的交互优势，又不把状态绑定在单个 Web 请求生命周期中。

## 6. AI 汇总链路的关键问题

项目 `genai.py` 的逻辑是：

```python
context = "\n\n".join(markdowns)
full_prompt = f"Context:\n{context}\n\nQuestion: {prompt}"
```

然后整体发送给 `gemini-2.0-flash-lite`。

这在 15 页 Demo 中尚可，但不能扩展到大知识库，原因包括：

- 输入长度随抓取页数线性增长；
- 没有 document/version 边界；
- 没有 chunk lineage；
- 没有 token 预算与截断证据；
- 没有单篇摘要复用；
- 任意一次重跑都可能重新消耗全部 token；
- 无法回答最终结论来自哪篇文章的哪一块；
- 如果某页重复或是模板页，会污染整次上下文。

知识库方案中必须继续采用：

```text
Accepted Version
 -> block-aware chunk
 -> per-document Summary Artifact
 -> Selection Manifest
 -> Reduce/Digest Artifact
```

并保存 source block refs，而不是把多个 Markdown 简单拼接。

## 7. Web 与运行时设计问题

### 7.1 长任务同步运行在 Flask 请求中

`routes.py` 在 POST 请求里直接调用 `asyncio.run()` 完成 Browser/深度抓取/Gemini。它没有 durable job、queue、lease、heartbeat、cancel、retry、resume。

后果是：

- Web worker timeout 会直接打断业务；
- 浏览器卡死会占用请求 worker；
- 用户刷新页面后无法恢复任务视图；
- 无法跨节点执行；
- 无法横向扩容独立 Browser Worker；
- 无法形成可审计 task lineage。

因此生产 Web 只能创建 Run/Job，并异步查询状态。

### 7.2 Browser 生命周期粒度过小

`deep_crawl()` 每次调用都 `async with AsyncWebCrawler()` 创建新的 crawler 生命周期。Demo 简单，但批量任务会重复付出 Browser 初始化成本。

生产环境需要 Browser Runtime Pool，并按：

- host/site 隔离；
- 最大并发；
- 最大 page/context 数；
- idle timeout；
- 最大寿命/内存；
- crash recycle；
- busy pin；

进行管理。

### 7.3 Secret 泄漏风险

`routes.py` 的 debug print 会把 `gemini_token` 打到标准输出，同时 token 被放进 Flask session。生产系统中这属于明确的安全反例。

正确做法是：

- 只存 Secret Manager reference；
- 日志和 trace 自动 redaction；
- API 不回显；
- 前端使用最小权限 credential binding；
- Worker 运行时短期解引用；
- Audit 只记 secret scope/id，不记明文。

### 7.4 测试覆盖不足

仓库只有 `tests/test_genai.py`，而且它依赖真实 `GEMINI_API_TOKEN` 调外部 API；没有 crawler、filter、route、timeout、failure、security、Web contract 的测试。

这说明项目更适合作为概念验证，不能直接作为生产基座。

## 8. 对博客知识库技术方案的可复用能力

本次调研最值得吸收的是“**Coverage Queue 与 Priority Queue 分离**”。

### 8.1 两种语义必须分开

```text
Coverage Candidate
- 回答：这个 URL 是否属于应被枚举和审计的历史集合？
- 依据：API/Sitemap/Feed/Archive/分页连续性/DOM evidence
- FULL_BACKFILL 中不能因低相关度被丢弃

Priority Hint
- 回答：在预算有限或交互式场景里先处理哪个 URL？
- 依据：URL pattern、keyword、head metadata、link position、人工主题
- 只改变队列优先级，不改变 Coverage
```

### 8.2 建议新增 `FOCUSED_DISCOVERY`

作为辅助 Run Type：

```text
FOCUSED_DISCOVERY
- 输入：source/site + query/prompt + budget
- 输出：ranked URL Candidate + score evidence
- 用途：站点接入探测、专题研究、人工排错、快速找到 Golden URL
- 禁止：把 max_pages/max_depth/score threshold 当成 coverage complete
```

它可复用 Crawl4AI Best-First，但必须持久化 scorer/filter release 与 score evidence。

### 8.3 Head Probe 独立成低成本 Metadata Plane

SEOFilter 暴露了一个值得保留的思路：在完整 Browser fetch 前，只读取页面 `<head>` 就能得到 title、canonical、robots、description、schema 等信号。

生产方案不应直接照搬 SEOFilter，而应抽象成：

```text
METADATA_PROBE
 -> conditional/head-range fetch
 -> metadata snapshot
 -> title/canonical/robots/schema/link preview
 -> priority hints + route hints
```

并要求：

- 与正文 Snapshot 分开标记 representation；
- 可缓存并复用；
- 每站点有 probe QPS/budget；
- 结果只能做 priority/route hint；
- noindex/robots 等合规信号进入独立 Gate；
- 对 FULL_BACKFILL 不使用 SEO 分数做内容存在性过滤。

### 8.4 Web 端增加渐进式探索视图

针对人工接入新站点，可提供：

```text
输入 root URL + 可选主题
 -> 创建 FOCUSED_DISCOVERY Run
 -> 页面实时显示 discovered/filtered/scored/fetched/accepted
 -> 按 score、depth、provider、pattern 查看候选
 -> 一键将 URL pattern/Golden URL 提议写入 Site Profile Draft
 -> 管理员验证后发布 Profile Release
```

这能把小项目里的“即时探索体验”保留下来，同时避免同步 Flask 长任务和不可审计状态。

## 9. 不应照搬的设计

以下设计明确不进入生产主方案：

1. 在 Web 请求中同步执行深度抓取和 LLM；
2. 每个请求新建完整 Browser crawler；
3. 用 `max_pages`/`max_depth` 证明抓取完整；
4. 用 SEO/Prompt 相关度过滤 FULL_BACKFILL 的合法文章；
5. 把多篇 Markdown 直接拼成单个 LLM Context；
6. 把用户模型 token 放 session 并打印日志；
7. 只把 `result.success` 当业务成功；
8. 不保存 Snapshot、Version、Release、score evidence、task lineage；
9. 把 `URLPatternFilter(["*", ...])` 当作有效文章路径过滤；
10. 把 URL 关键词命中误称为正文语义相关性。

## 10. 对最终方案的修改结论

现有博客知识库技术方案的主架构方向正确：Coverage 与 Capability 已分离、`max_pages/max_depth` 已明确不是完整性证明、Web 长任务已异步化、AI 已采用 Version/Artifact/Selection Manifest 模型。

本项目仍带来四项可落地优化：

1. **新增 Focused/Prioritized Discovery 辅助平面**，把 Best-First 明确定位为“调度优先级”，而不是 Coverage；
2. **新增 metadata/head probe 的成本与缓存边界**，避免类似 SEOFilter 的隐式每 URL 网络请求放大 1000 站点成本；
3. **Discovery Candidate 增加 score evidence 与 scorer/filter release lineage**，保证为什么某 URL 被提前抓取可以解释、重放和比较；
4. **Web 增加流式探索/站点接入视图**，把 `stream=True` 的价值真正转化成渐进反馈，并可将候选 pattern/Golden URL 形成 Site Profile Draft。

这些优化不改变核心事实层：**完整历史仍由可枚举 Provider + Coverage Evidence 证明；相关性只决定“先抓谁”，不能决定“谁不存在”。**

## 11. 主要源码依据

- 项目仓库：https://github.com/cf2018/crawl4ai_news_summarizer
- `app/crawler.py`：BestFirst + FilterChain + KeywordRelevanceScorer + streaming
- `app/routes.py`：Flask 同步请求内执行 crawler 与 Gemini，且打印 token
- `app/genai.py`：多 Markdown 直接拼接为单 Context
- `tests/test_genai.py`：仅覆盖真实 Gemini API 调用
- Crawl4AI 0.7.0 `filters.py`：FilterChain、URLPatternFilter、ContentTypeFilter、SEOFilter 实现
- Crawl4AI 0.7.0 `scorers.py`：KeywordRelevanceScorer 只基于 URL 字符串关键词命中
- Crawl4AI Deep Crawling 文档：https://docs.crawl4ai.com/core/deep-crawling/
- Crawl4AI v0.7.0 release：https://docs.crawl4ai.com/blog/releases/0.7.0/
