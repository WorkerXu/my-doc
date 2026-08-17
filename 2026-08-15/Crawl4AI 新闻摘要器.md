# Crawl4AI 新闻摘要器

## 1. 调研对象与结论

- 编号：54
- 项目：`cf2018/crawl4ai_news_summarizer`
- 地址：https://github.com/cf2018/crawl4ai_news_summarizer
- 项目形态：Flask Web + Crawl4AI + Gemini
- 固定依赖：Crawl4AI 0.7.0、Playwright 1.53.0、Flask 3.1.1、google-genai 1.25.0
- 核心代码：`app/crawler.py`、`app/routes.py`、`app/genai.py`、`app/templates/index.html`

这个仓库是一个轻量交互原型：用户输入站点和主题 Prompt，在有限页面预算中抓网页，把 Crawl4AI 生成的 Markdown 交给 Gemini 生成回答。它适合验证“主题优先探索 + AI 汇总”的交互体验，但不具备 1000 个站点长期全量归档、增量同步、可恢复调度和知识库治理所需的持久状态与运行契约。

对博客知识库方案最有价值的结论有五类：

1. 相关性优先探索适合作为 `FOCUSED_DISCOVERY`，但不能承担历史 Coverage；
2. 会联网的 head/SEO Filter 必须显式建模成 `METADATA_PROBE`，不能伪装成本地过滤；
3. Web 只能创建持久 Run/Job，不能把 Browser + LLM 长任务绑定在单个 HTTP 请求；
4. 第三方 crawler 的 `score`、priority queue、`max_pages`、batch 等语义必须对实际 pinned runtime 做契约验证，不能根据名称推断；
5. **并发系统必须把“候选数量、真正派发的网络请求数量、成功数量、Accepted 数量”分开。Crawl4AI 0.7.0 的 Best-First 实现存在 batch 预算边界问题，说明成功后计数的 `max_pages` 不能直接当作严格源站请求预算。**

## 2. 项目实际执行链路

### 2.1 设计意图中的 Deep Crawl

`app/crawler.py` 定义：

```text
root URL + prompt + max_depth/max_pages
 -> FilterChain
    -> DomainFilter
    -> URLPatternFilter
    -> ContentTypeFilter
    -> SEOFilter
 -> KeywordRelevanceScorer
 -> BestFirstCrawlingStrategy
 -> AsyncWebCrawler.arun(stream=True)
 -> CrawlResult.markdown
```

核心配置：

```python
filter_chain = FilterChain([
    DomainFilter(allowed_domains=[url.split('/')[2]]),
    URLPatternFilter(patterns=["*", "*blog*", "*article*", "*news*"]),
    ContentTypeFilter(allowed_types=["text/html"]),
    SEOFilter(threshold=0.3, keywords=prompt.split())
])

scorer = KeywordRelevanceScorer(
    keywords=prompt.split(),
    weight=0.8
)

config = CrawlerRunConfig(
    deep_crawl_strategy=BestFirstCrawlingStrategy(
        max_depth=max_depth,
        max_pages=max_pages,
        filter_chain=filter_chain,
        url_scorer=scorer
    ),
    stream=True,
    page_timeout=30000
)
```

成功结果被收集为 URL、title、score、depth、markdown。默认 `max_depth=2`、`max_pages=15`。

### 2.2 当前 UI 的 Deep Crawl 实际不会进入 Deep 分支

模板中：

```html
<button name="action" value="deep">Deep Crawl</button>
<button name="action" value="simple"
        onclick="document.getElementById('simple').value='1'">
  Simple Scrape
</button>
<input type="hidden" name="simple" id="simple" value="1">
```

后端却不读取 `action`：

```python
simple = request.form.get('simple', None)
...
if simple or depth == 1:
    # Simple scrape
else:
    results = run_deep_crawl(...)
```

隐藏字段默认始终提交字符串 `"1"`，Python 中它为 truthy，因此正常页面提交时无论点击 Deep Crawl 还是 Simple Scrape，都会走 Simple Scrape。

这暴露的是生产控制面的通用问题，而不只是一个前端 Bug：**UI 显示的动作与后端真实 Run Type 不能依赖隐藏字段、字符串 truthiness、按钮文字或未被读取的参数。**

生产应使用：

```text
UI action enum
 -> Pydantic Command Schema
 -> requested_run_type/requested_config
 -> server-side validation/authorization
 -> config resolver
 -> effective_run_type/effective_config
 -> durable Run
```

并保存 requested/effective diff 供审计和 Web 展示。

## 3. Crawl4AI 0.7.0 Best-First 的真实实现

项目固定 `Crawl4AI==0.7.0`，所以必须分析该版本的代码，而不能拿后续版本行为替代。

### 3.1 PriorityQueue 是小根堆，raw score 越小越先出队

v0.7.0 `BestFirstCrawlingStrategy` 使用：

```text
asyncio.PriorityQueue
queue item = (score, depth, url, parent_url)
```

发现新 URL 后直接执行：

```python
new_score = self.url_scorer.score(new_url) if self.url_scorer else 0
await queue.put((new_score, new_depth, new_url, new_parent))
```

没有对 score 取负，也没有自定义 comparator。Python PriorityQueue 按元组从小到大弹出，所以 **更小的 score 更早被抓取**。

### 3.2 KeywordRelevanceScorer 却是越相关分越高

v0.7.0 `KeywordRelevanceScorer` 只检查关键词是否出现在 URL 字符串：

```text
无关键词命中 -> 0.0
全部命中     -> 1.0
部分命中     -> matches / keyword_count
```

再乘项目配置 `weight=0.8`。

因此项目意图是：

```text
0.8 = 更相关
0.0 = 不相关
```

但 Best-First 实际会：

```text
0.0 先抓
0.8 后抓
```

所以在项目固定的 v0.7.0 组合里，`BestFirstCrawlingStrategy + KeywordRelevanceScorer` 会优先处理低相关 raw score URL，与“优先找主题相关页面”的意图相反。

生产系统不能把第三方 `score` 原样作为业务 priority。统一契约应为：

```text
raw_score
 -> priority_adapter_release
 -> normalized_priority in [0,1]
    1 = 最优先
 -> scheduler_key = (-normalized_priority, ...)
```

每个 pinned runtime 发布前用实际队列执行契约测试，而不是 mock scorer。

### 3.3 `remaining_capacity` 在打分前截断候选，造成 DOM 顺序偏置

v0.7.0 `link_discovery()` 先计算：

```python
remaining_capacity = self.max_pages - self._pages_crawled
```

收集有效链接后执行：

```python
if len(valid_links) > remaining_capacity:
    valid_links = valid_links[:remaining_capacity]
```

之后这些 URL 才进入 scorer/priority queue。

假设首页观察到 100 个内部链接，剩余 `max_pages` 只有 14：

1. 先按链接返回/DOM 顺序保留前 14 个；
2. 后 86 个候选直接从 crawler 内部 frontier 消失；
3. scorer 没机会比较全部 100 个 URL；
4. 即便 DOM 最后的 URL 与 Prompt 最相关，也不会进入 priority queue。

所以这个版本的 Best-First 不是“先对全部可观察候选评分，再选最高分”，而是“先按剩余容量截断，再对截断后的子集排序”。

生产平台必须采用：

```text
Fetched parent Snapshot
 -> enumerate all observable links
 -> normalize/dedupe
 -> persist Candidate + Evidence
 -> local score / Metadata Probe
 -> normalized priority
 -> fetch budget admission
```

预算可以限制后续抓多少页，但不能抹掉“父页面当时观察到了哪些链接”这个事实。

### 3.4 `max_pages` 不是 Coverage

无论排序方向是否正确，`max_pages`、`max_depth` 都只是执行停止条件。1000 站点历史归档若把“抓到 15 页”“队列空了”当完整性证明，会系统性漏掉：

- URL 不含主题词的旧文章；
- 低 SEO 分但正文有效的文章；
- 年月 Archive 深层页；
- 冷门 Category/Author 页；
- metadata 不规范的早期静态博客；
- Prompt 与 slug 词汇不同的文章。

历史完整性必须依靠 Sitemap/API/Feed/Archive 等 Provider 的 enumeration boundary、cursor continuity、expected/discovered count、known gap 与 blocker。

### 3.5 v0.7.0 还存在“计算了剩余 batch_size，但派发仍按固定 BATCH_SIZE”的预算问题

这是对生产方案最重要的额外结论。

v0.7.0 `_arun_best_first()` 中先计算：

```python
remaining = self.max_pages - self._pages_crawled
batch_size = min(BATCH_SIZE, remaining)
```

看起来意图是剩多少页就最多取多少个。但真正从优先队列取任务时却使用：

```python
for _ in range(BATCH_SIZE):
    if queue.empty():
        break
    item = await queue.get()
    ...
    batch.append(item)
```

这里没有使用刚计算的 `batch_size`。随后：

```python
urls = [item[2] for item in batch]
stream_gen = await crawler.arun_many(urls=urls, config=batch_config)
```

成功结果返回后才：

```python
if result.success:
    self._pages_crawled += 1
    if self._pages_crawled >= self.max_pages:
        break
```

因此当：

```text
BATCH_SIZE = 10
remaining max_pages = 1
queue 中还有 >= 10 个 URL
```

策略仍可能把最多 10 个 URL 交给 `arun_many()`。后面即使只继续计数/产出到第 1 个成功结果，其他 URL 的网络工作可能已经被启动。

这里必须区分两件事：

```text
“最终只计数 N 个 success”
!=
“最多只 dispatch N 个源站请求”
```

对小型 Demo，这可能只是额外请求；对 1000 站点平台，它会影响：

- 礼貌限速；
- per-host QPS；
- 成本预算；
- Browser page 数；
- 源站压力；
- 取消语义；
- 审计中“究竟发了多少请求”的真实性。

生产系统因此不能把第三方 `max_pages` 当严格网络预算，必须拆成：

```text
candidate_budget
metadata_probe_budget
fetch_dispatch_budget
fetch_success_budget
accepted_budget
retry_budget
byte_budget
wall_clock_budget
cost_budget
```

其中 `fetch_dispatch_budget` 在 **网络请求进入可执行队列前** 原子 Reservation：

```text
Candidate selected
 -> route resolved
 -> reserve host/global token
 -> reserve fetch_dispatch_budget
 -> persist reservation/task intent
 -> dispatch network request
 -> settle actual bytes/time/outcome
```

Worker batch 大小必须满足：

```text
batch_size <= remaining_dispatch_budget
batch_size <= available_host_tokens
batch_size <= available_global_tokens
batch_size <= local_worker_capacity
```

验收不能只检查“最后成功了几页”，而要检查 `reserved/dispatched/started`。

## 4. FilterChain 的真实语义与隐含成本

### 4.1 FilterChain 是 AND 语义

同步 Filter 任一返回 False 即拒绝；异步 Filter 被收集后执行，并要求最终全部通过。因此 Domain、URL Pattern、Content Type、SEO 都会参与 Candidate 是否进入内部 frontier 的判断。

### 4.2 `URLPatternFilter(["*", ...])` 实际恒真

项目配置：

```python
URLPatternFilter(patterns=["*", "*blog*", "*article*", "*news*"])
```

v0.7.0 多 pattern 是“任一匹配即通过”。`*` 可匹配全部 URL，所以后面的 `*blog*`、`*article*`、`*news*` 对过滤没有实际约束作用。

生产 Profile Preflight 应检测：

- 恒真/恒假 pattern；
- include/exclude 互相覆盖；
- 通配符吞掉后续规则；
- 正则异常成本；
- Golden URLs 上的实际 recall/误杀。

未知站点默认把路径 pattern 当 soft Priority Hint，不直接 hard exclude。

### 4.3 KeywordRelevanceScorer 只看 URL，不看正文

它不会读取正文、标题或 anchor context，只做 URL 字符串关键词命中。

例如：

```text
Prompt: distributed systems
URL: /posts/raft-consensus
```

正文即使完全相关，只要 URL 没有 `distributed`/`systems`，raw score 仍可能为 0。

因此业务命名应是 `url_keyword_priority_score`，不能展示成“正文语义相关度”。真正正文相关度应消费已存在 Snapshot/Canonical IR 或 Search/Reranker 结果。

### 4.4 SEOFilter 是 NETWORK_PROBE，不是本地 Predicate

v0.7.0 `SEOFilter.apply()` 会调用 `HeadPeekr.peek_html(url)` 读取候选页面 head，再按以下因素打分：

- title 长度；
- title 关键词；
- meta description；
- canonical；
- robots/noindex；
- schema.org JSON-LD；
- URL 结构。

项目阈值为 `0.3`。

这意味着每个候选 URL 在正文抓取前可能额外发生一次网络访问。如果百万级候选直接放在 Discovery 热路径，会放大连接、QPS、字节、超时和源站压力。

生产应显式建模：

```text
METADATA_PROBE Task
 -> Admission / robots / policy
 -> Probe Budget Reservation
 -> HEAD 或受限 GET/range
 -> metadata_probe_snapshot
 -> title/canonical/robots/schema/content-type
 -> Priority Hint / Route Hint / Profile Draft
```

并保存 cache、ETag/Last-Modified、bytes、deadline、QPS、budget、trace。

### 4.5 SEO 分数不能决定历史文章是否存在

老博客、自建静态站可能没有 canonical/schema、meta description 不规范、标题长度不符合 SEO 模板、URL 带年份/query，但正文仍有长期知识价值。因此 SEO/head 信号只能用于 priority、route、onboarding diagnosis；不能从 `FULL_BACKFILL`/`RECONCILE` Coverage Candidate 中删除 URL。

### 4.6 ContentTypeFilter 主要是 URL 扩展名启发式

v0.7.0 的该 Filter 主要用 URL 扩展名映射 MIME；没有扩展名通常放行。它适合提前排除明显图片/压缩包，但不能代替真实响应 Content-Type、MIME sniff、最大响应字节、解压膨胀比和 Representation Route。

## 5. `stream=True`：Worker 流式不等于 Web 流式

项目 Deep Crawl 配置 `stream=True`，理论上可：

- 页面完成后逐个产出；
- 高优先级页面先处理；
- 边抓边写 Snapshot；
- 边抓边做 Quality/Identity；
- 更早响应取消。

但项目 `deep_crawl()` 又把成功结果全部 append 到 Python list；Flask 路由等抓取结束才调用 Gemini，最后一次性 `render_template()`。

因此目前：

- 没有 SSE/WebSocket；
- 用户看不到中间结果；
- 内存仍随结果数增长；
- Web 请求承担 Browser + Gemini 的完整生命周期；
- `stream=True` 的收益没有传递给 Web。

生产应为：

```text
POST /runs
 -> durable Run ID
 -> Worker stream page result
 -> Snapshot / Candidate / Quality / Version
 -> progress event
 -> Web SSE/WebSocket/polling
 -> cancel/retry/replay
```

## 6. AI 汇总实现的扩展性问题

`genai.py`：

```python
context = "\n\n".join(markdowns)
full_prompt = f"Context:\n{context}\n\nQuestion: {prompt}"
```

再整体发送给 `gemini-2.0-flash-lite`。

问题：

1. token 随页面数和正文长度线性增长；
2. 没有 Document/Version 边界；
3. 没有 chunk lineage；
4. 没有 source block refs；
5. 没有输入版本集合 hash；
6. 没有可解释 truncation；
7. 没有单篇摘要缓存/复用；
8. 模板页、重复页、错误页会污染整个 Context；
9. 重试可能重复支付全部输入 token；
10. 无法稳定回答“当时到底汇总了哪几个版本”。

生产应采用：

```text
Accepted Document Version
 -> AI Input Projection
 -> block-aware Chunk Plan
 -> per-document Map Artifact
 -> Selection/Input Manifest
 -> Reduce/Digest Artifact
```

记录 ordered version ids、input set hash、recipe/model/runtime release、token budget、truncation evidence、source block refs、cost/latency/status。

## 7. Web、运行时与可靠性问题

### 7.1 长任务同步运行在 Flask 请求中

`routes.py` 直接在 POST 中调用 crawler 与 Gemini，没有 durable queue、lease、heartbeat、retry、resume、dead-letter、cancel propagation。Web worker timeout、浏览器卡死、用户刷新或实例重启都会影响业务任务。

生产 Web 只能创建 Command/Run、查询状态和发起显式控制动作。

### 7.2 每次调用创建新的 AsyncWebCrawler

`deep_crawl()`：

```python
async with AsyncWebCrawler() as crawler:
```

Demo 简单，但批量平台会重复支付 Browser/Context 初始化成本，并难以统一控制最大总并发、per-site 并发、context/page 复用、crash recycle、idle timeout、最大寿命、内存阈值和浏览器版本 attestation。

生产使用 Browser Runtime Pool，让 Crawl4AI 只是 Fetch/Runtime Adapter 之一。

### 7.3 `result.success` 不是知识库内容成功

项目仅在 `result.success` 时收集 Markdown，但成功页面可能是登录页、WAF challenge、空壳 SPA、soft 404、列表页、cookie banner、导航模板或极短正文。

生产必须区分：

```text
transport outcome
content/extraction outcome
quality outcome
identity/version outcome
```

只有通过 Canonical IR + Quality Gate 的 Accepted Version 才发布到知识库。

### 7.4 第三方 crawler 的内部 frontier 不是持久调度真相

内部 PriorityQueue、visited set、batch counter 都属于进程内状态。Worker 崩溃后不能用这些数据证明 Coverage、Budget 或 Resume。平台需在 PostgreSQL 保存 Candidate/Evidence、Task、Budget Reservation、Route Decision 和 Snapshot。

## 8. 安全问题

### 8.1 Gemini Token 被日志打印

`routes.py` 的 debug print 包含 `gemini_token`，属于明确 Secret 泄漏风险。日志、trace、error report 必须默认 redaction。

### 8.2 Token 被写入 Flask session

代码：

```python
session['gemini_token'] = token_input
```

项目没有服务端 Session 扩展；Flask 默认 cookie session 主要提供签名完整性，不应作为 API Secret 的服务器端机密存储。

生产只保存 Secret Manager reference，Worker 运行时以最小权限短期解引用。

### 8.3 默认 Secret Key 与 Debug 模式仅适合开发

`app.py`：

```python
app.config['SECRET_KEY'] = os.getenv('SECRET_KEY', 'supersecretkey')
app.run(debug=True)
```

生产必须强制外部 Secret、禁止默认密钥和 debug，使用正式 WSGI/ASGI runtime，并配置 RBAC/CSRF/CSP/secure cookie 策略。

### 8.4 任意 URL 必须统一 Admission

表单的 URL validator 不是 SSRF 防护。Web/Agent/Dify 任意 URL 抓取需统一：仅 http/https、DNS/IP 校验、拒绝 localhost/私网/link-local/metadata/reserved、redirect 每跳重校验、Browser Worker 网络层 deny 内网、response bytes/deadline/content-type 限制。

## 9. 测试覆盖与运行时语义契约

仓库只有 `tests/test_genai.py`，依赖真实 Gemini Token 调外部 API，没有覆盖 crawler/filter/UI/priority/budget/security。

生产至少增加：

### 9.1 Priority Direction Contract

给定：

```text
high raw score = 0.8
low raw score  = 0.0
```

断言经过 Priority Adapter 后 high normalized priority 实际先被平台调度。测试实际 pinned runtime，而不是只测 scorer 函数。

### 9.2 Candidate Admission Contract

父页面 100 个候选，最高价值 URL 位于 DOM 最后，fetch budget=10。断言：100 个 observable links 都先进入 Candidate/Evidence；评分可看到全部候选；预算只限制后续 fetch。

### 9.3 Web Command Contract

自动化提交 Deep/Simple 等 UI 操作，断言 UI action → Command → requested/effective run type → durable Run 一致，覆盖隐藏字段与字符串 truthiness。

### 9.4 Dispatch Budget Contract

构造：

```text
remaining fetch_dispatch_budget = 1
priority queue >= 10 URLs
worker/crawler batch concurrency > 1
```

断言：最多只有 1 个新 URL 获得 Budget Reservation 并进入 `dispatched/started`。不能只断言“最后只返回/计数 1 个 success”。

### 9.5 Cancellation / Retry Budget Contract

取消后不得产生新 Reservation；in-flight 实际消耗仍 settle/audit。消息重复投递和 retry 不得绕过 idempotency key 重复扣预算或重复发请求。

### 9.6 Secret Boundary Contract

session/cookie、日志、trace、error response、Audit、Task payload 均不得出现模型/API Secret 明文。

## 10. 对 1000 站点知识库的正确抽象

### 10.1 Coverage 与 Priority 分离

```text
Coverage Candidate
- 回答：历史全集中应该审计哪些 URL？
- 依据：API/Sitemap/Feed/Archive/分页/Docs Navigation/DOM Evidence
- FULL_BACKFILL/RECONCILE 不因低相关度删除

Priority Hint
- 回答：预算有限时先处理谁？
- 依据：URL keyword/path/anchor/head metadata/人工主题/已有 IR 语义
- 只改变调度顺序
```

### 10.2 Candidate、Budget 与 Fetch 再分一层

```text
Observed Candidate
- 父页面/Provider 已经证明“看到了这个 URL”

Budget Admitted
- 本次 Run 允许它进入昂贵阶段

Reserved / Dispatched / Started
- 已取得网络副作用预算，并真正派发/开始请求

Transport Success
- 网络/浏览器层成功

Content Accepted
- Extraction + Quality 通过，可成为 Version
```

这些计数必须全部独立展示，不能把“未在本次 budget 中抓”显示成“没发现”，也不能把“只接受 N 篇”显示成“只请求了 N 次”。

### 10.3 FOCUSED_DISCOVERY 推荐链路

```text
root URL + optional topic + budgets
 -> durable FOCUSED_DISCOVERY Run
 -> Provider/DOM Candidate enumeration
 -> persist Candidate/Evidence
 -> LOCAL_PREDICATE or METADATA_PROBE
 -> raw score
 -> priority_adapter_release
 -> normalized priority
 -> Budget Admission / Reservation
 -> deterministic durable frontier
 -> fetch
 -> Snapshot
 -> Extraction/Quality
 -> Accepted Version
 -> progressive Web events
```

### 10.4 第三方库只承担局部执行

Crawl4AI 可作为 Browser/Deep Crawl/Markdown Candidate 能力，但平台不能把其内部 visited、PriorityQueue、max_pages、success counter 当业务真相。平台自己持久化 Coverage、Candidate、Priority、Budget、Task、Snapshot、Version。

## 11. 对最终技术方案的落地增补

本项目要求最终方案明确包含：

1. **Priority Contract**：保存 raw score 与 normalized priority，由版本化 Adapter 映射第三方 score 方向；
2. **Candidate-before-budget**：已观察链接先落 Candidate/Evidence，再评分和预算准入；
3. **Metadata Probe 显式化**：任何联网 Filter/Scorer 转为 Probe Task；
4. **Pinned Runtime Semantic Test**：验证 priority direction、candidate truncation、stream/cancel、batch/max_pages 边界；
5. **Typed Web Command**：保存 requested/effective run type/config，禁止字符串 truthiness 决定模式；
6. **Secret Boundary**：模型/API Token 只使用 Secret reference；
7. **Budget Contract**：candidate/probe/dispatch/success/accepted/bytes/time/cost 分开建模；
8. **Dispatch-before-side-effect**：真正联网前持久化 Budget Reservation，batch 大小受 remaining dispatch budget 约束；
9. **Budget Observability**：分别记录 reserved/dispatched/started/completed/success/accepted 和 overrun/wasted in-flight；
10. **Dispatch Budget Regression Test**：剩余预算为 1 时，不论第三方 batch 多大都只能派发 1 个新请求。

## 12. 不应照搬的设计

1. 在 Web 请求内同步完成 Browser/深度抓取/LLM；
2. 每次请求创建新的完整 Browser crawler；
3. 用 `max_pages`、`max_depth` 或队列空了证明历史完整；
4. 用 SEO/Prompt 相关度过滤 FULL_BACKFILL 的合法文章；
5. 把多篇 Markdown 直接拼成一个无结构 LLM Context；
6. 把模型 Token 放普通 session/cookie 或打印到日志；
7. 只以 `result.success` 判断知识库正文成功；
8. 不保存 Snapshot、Version、Release、score evidence、Budget Reservation 和 task lineage；
9. 把 `URLPatternFilter(["*", ...])` 当有效文章路径过滤；
10. 把 URL 关键词命中叫正文语义相关性；
11. 直接把第三方 raw score 当调度 priority，不验证方向；
12. 在评分前因为 `remaining_capacity` 按 DOM 顺序截断已观察到的候选；
13. 用隐藏字符串布尔值决定 Deep/Simple 等核心模式；
14. 用“success counter 达到 max_pages”证明源站请求没有超过 max_pages；
15. 把第三方 batch/prefetch 的真实请求数隐藏在一个最终结果数量指标后面。

## 13. 主要源码依据

- 项目仓库：`https://github.com/cf2018/crawl4ai_news_summarizer`
- `requirements.txt`：固定 Crawl4AI 0.7.0、Playwright 1.53.0、Flask 3.1.1、google-genai 1.25.0
- `app/crawler.py`：FilterChain、KeywordRelevanceScorer、BestFirst、`stream=True`
- `app/routes.py`：Simple/Deep 分支、同步执行 crawler/Gemini、日志打印 token
- `app/templates/index.html`：`action=deep/simple` 与隐藏 `simple=1` 控件绑定
- `app/genai.py`：多 Markdown 直接拼接成 Context 后调用 Gemini
- `app/app.py`：默认 `supersecretkey` 与 `debug=True`
- `tests/test_genai.py`：仅真实 Gemini API 测试
- Crawl4AI v0.7.0 `crawl4ai/deep_crawling/bff_strategy.py`：PriorityQueue、raw score 入队、`remaining_capacity` 评分前截断、`batch_size` 计算与固定 `BATCH_SIZE` 出队、`arun_many`、成功后 `_pages_crawled` 计数
- Crawl4AI v0.7.0 `crawl4ai/deep_crawling/scorers.py`：KeywordRelevanceScorer 为 URL 字符串关键词命中且“命中越多分越高”
- Crawl4AI v0.7.0 `crawl4ai/deep_crawling/filters.py`：FilterChain、URLPatternFilter、ContentTypeFilter、SEOFilter/HeadPeekr