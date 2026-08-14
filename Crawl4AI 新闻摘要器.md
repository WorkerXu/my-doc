# Crawl4AI 新闻摘要器

## 1. 调研对象与结论

- 编号：54
- 项目：`cf2018/crawl4ai_news_summarizer`
- 地址：https://github.com/cf2018/crawl4ai_news_summarizer
- 项目形态：Flask Web + Crawl4AI + Gemini
- 固定依赖：Crawl4AI 0.7.0、Playwright 1.53.0、Flask 3.1.1、google-genai 1.25.0
- 核心代码：`app/crawler.py`、`app/routes.py`、`app/genai.py`、`app/templates/index.html`

这个仓库是一个很小的交互式原型，目标是“输入站点和主题 Prompt，在有限页面预算中抓网页，再把 Markdown 交给 Gemini 生成回答”。它不是为 1000 个站点的长期全量归档、增量同步或知识库治理设计的生产系统。

本项目最值得吸收的不是代码本身，而是四类架构启示：

1. 相关性优先探索适合作为 `FOCUSED_DISCOVERY`，但不能承担历史 Coverage；
2. 会联网的 head/SEO 过滤器必须显式建模成 `METADATA_PROBE`；
3. Web 交互应该创建持久 Run/Job，并渐进返回结果，而不是把 Browser + LLM 绑在一个 HTTP 请求内；
4. **第三方抓取库的“score”“priority”“max_pages”等语义必须做版本级契约验证，不能仅凭类名或文档推断。** 项目固定的 Crawl4AI 0.7.0 在 Best-First 排序方向和候选截断上存在会改变业务含义的实现细节。

另外，本项目当前 Web 页面还有一个直接的控制绑定缺陷：正常点击 “Deep Crawl” 实际仍会走 Simple Scrape，因此必须把“前端控件 → API Command → Run Type → Effective Config”纳入强类型契约和审计。

## 2. 代码结构与两条执行链路

### 2.1 设计意图中的 Deep Crawl 链路

`app/crawler.py` 定义了：

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

抓取成功结果在进程内被收集为：

```python
{
    'url': result.url,
    'title': result.metadata.get('title', ''),
    'score': result.metadata.get('score', 0),
    'depth': result.metadata.get('depth', 0),
    'markdown': result.markdown
}
```

### 2.2 当前 Web UI 的实际链路：Deep Crawl 控件失效

`index.html` 中两个按钮分别提交：

```html
<button name="action" value="deep">Deep Crawl</button>
<button name="action" value="simple" onclick="document.getElementById('simple').value='1'">Simple Scrape</button>
<input type="hidden" name="simple" id="simple" value="1">
```

但 `routes.py` 根本不读取 `action`，而是读取：

```python
simple = request.form.get('simple', None)
...
if simple or depth == 1:
    # Simple scrape
else:
    results = run_deep_crawl(...)
```

因为隐藏字段 `simple` 默认始终是字符串 `"1"`，在 Python 条件判断中恒为真，所以**正常页面提交时不论点击 Deep Crawl 还是 Simple Scrape，都会进入 Simple Scrape 分支**。

因此，仓库 README/UI 所表达的交互与实际后端行为并不一致。`run_deep_crawl()` 只有在调用方构造一个不带 `simple` 或带空值的 POST 时才可能从这个路由进入。

这不是单纯的前端小 Bug，而是生产控制面的典型风险：如果业务模式依赖隐藏字段、字符串真值或未使用的 action 参数，UI 显示的操作和后端实际 Run Type 可能长期不一致。

生产系统应使用：

```text
UI action enum
 -> Pydantic Command Schema
 -> server-side authorization + validation
 -> requested_run_type/requested_config
 -> config resolver
 -> effective_run_type/effective_config
 -> durable Run
```

禁止用 `if request.form.get("flag")` 这种 truthy string 决定核心运行模式。

## 3. Crawl4AI 0.7.0 Best-First 的真实实现语义

项目 `requirements.txt` 固定 `Crawl4AI==0.7.0`，因此必须分析该版本，而不能拿后续版本的实现替代。

### 3.1 它使用 `asyncio.PriorityQueue` 小根堆

v0.7.0 的 `BestFirstCrawlingStrategy` 把队列项定义成：

```text
(score, depth, url, parent_url)
```

Python `asyncio.PriorityQueue` 按元组从小到大弹出，所以**更小的 score 会更早被处理**。

实现中发现新 URL 后直接：

```python
new_score = self.url_scorer.score(new_url) if self.url_scorer else 0
await queue.put((new_score, new_depth, new_url, new_parent))
```

没有对 score 取负，也没有额外 comparator。

### 3.2 项目的 KeywordRelevanceScorer 却是“越相关分越高”

v0.7.0 `KeywordRelevanceScorer` 的逻辑是：

```text
URL 中没有关键词 -> 0.0
命中全部关键词     -> 1.0
部分命中           -> matches / keyword_count
```

再乘项目配置的 `weight=0.8`。

也就是说项目的业务直觉是：

```text
0.8 = 更相关
0.0 = 不相关
```

但 v0.7.0 Best-First 的小根堆实际优先级是：

```text
0.0 先抓
0.8 后抓
```

因此**这个固定版本上的“Best-First + KeywordRelevanceScorer”实际上会优先处理低相关分 URL，而不是高相关分 URL**。这与项目“根据 Prompt 优先找相关新闻”的意图相反。

后续 Crawl4AI 源码已经出现把 score 取负后入队的实现，但对于本项目不能用后续行为替代固定依赖的事实。生产系统必须以实际 `runtime_release` 为准。

### 3.3 v0.7.0 还会在打分前按剩余页面预算截断链接

`link_discovery()` 在一个页面发现若干有效链接后先计算：

```python
remaining_capacity = self.max_pages - self._pages_crawled
```

然后：

```python
if len(valid_links) > remaining_capacity:
    valid_links = valid_links[:remaining_capacity]
```

之后这些保留下来的 URL 才进入 scorer 和 priority queue。

这意味着假设首页有 100 个内部链接、剩余预算 14：

1. 先按页面 DOM/解析顺序保留前 14 个；
2. 后 86 个候选直接消失；
3. scorer 根本没有机会比较全部 100 个 URL；
4. 即使第 100 个 URL 与 Prompt 最相关，也不会进入 frontier。

所以 v0.7.0 在小 `max_pages` 下的行为不只是“预算有限”，还存在**pre-score truncation 导致的 DOM 顺序偏置**。

生产 `FOCUSED_DISCOVERY` 应把两个步骤拆开：

```text
Fetched page
 -> enumerate all observable links
 -> normalize/dedupe
 -> persist Candidate + Evidence
 -> local/probe scoring
 -> normalized priority
 -> fetch budget admission
```

页面 fetch 预算可以限制“接下来抓多少页”，但不应在已经拿到一个页面的链接后，先按 DOM 顺序裁掉候选再打分。

### 3.4 生产系统必须定义自己的 Priority Contract

不能把第三方库的 `score` 原样当调度优先级。建议统一内部语义：

```text
raw_score
  = 第三方 scorer 原生输出

normalized_priority
  = 经过 priority_adapter_release 转换后的 [0,1]
  = 数值越大越应该先处理

scheduler_key
  = (-normalized_priority, next_due, depth, discovered_at, canonical_url)
```

这样可以把“第三方用大分优先还是小分优先”“是否需要取负”“是否归一化”等差异封装在 Adapter Release 中。

每个版本发布前必须有语义契约测试：给定高分 URL 与低分 URL，断言高 `normalized_priority` 的 URL 确实先被调度。

## 4. FilterChain 的实现与隐含成本

### 4.1 FilterChain 是 AND 语义

同步 Filter 只要一个返回 False 就提前拒绝；异步 Filter 会被收集后 `asyncio.gather()`，最终要求全部为 True。

因此 Domain、URL Pattern、Content Type、SEO 都通过才会进入 frontier。

### 4.2 `URLPatternFilter(["*", ...])` 实际恒真

项目配置：

```python
URLPatternFilter(patterns=["*", "*blog*", "*article*", "*news*"])
```

v0.7.0 会把 glob 转换成正则，并对多个 pattern 做“任一命中即通过”。`*` 可匹配任意 URL，因此后面的 `*blog*`、`*article*`、`*news*` 不再具有筛选效果。

生产系统应在 Profile Preflight 中检查：

- 恒真 pattern；
- 恒假 pattern；
- include/exclude 相互覆盖；
- 通配符吞掉后续规则；
- 正则灾难性回溯风险；
- 规则对 Golden Corpus 的实际召回/误杀。

未知站点默认把 path pattern 作为 soft Priority Hint，而不是 hard filter。

### 4.3 `KeywordRelevanceScorer` 只看 URL，不看正文

项目：

```python
KeywordRelevanceScorer(keywords=prompt.split(), weight=0.8)
```

v0.7.0 实现只检查关键词字符串是否出现在 URL 中。

因此：

```text
Prompt: distributed systems
URL: /posts/raft-consensus
```

正文即使完全相关，只要 slug 没有 `distributed`/`systems`，这个 scorer 仍可能给 0。

所以其业务名称应是 `url_keyword_priority_score`，不能显示成“内容语义相关度”。正文相关度必须消费已抓取 Snapshot/Canonical IR，或走独立 Search/Reranker，而不能伪装成 URL scorer。

### 4.4 `SEOFilter` 是联网 Filter，不是本地 Predicate

v0.7.0 `SEOFilter.apply()` 会调用 `HeadPeekr.peek_html(url)`，读取候选页面 head，再对以下字段打分：

- title 长度；
- title 中关键词；
- meta description；
- canonical；
- robots/noindex；
- schema.org；
- URL 结构。

项目阈值是 0.3。

所以把 SEOFilter 放在 Discovery FilterChain 内意味着：**每个候选 URL 在真正正文 fetch 前可能先发生一次额外网络请求。** 在百万级 URL 下，这会放大源站压力、连接数、超时和成本。

生产系统应把这类行为显式物化成：

```text
METADATA_PROBE Task
 -> URL Admission
 -> rate limit / robots / policy
 -> HEAD 或受限 GET/range
 -> metadata_probe_snapshot
 -> title/canonical/robots/schema/content-type
 -> Priority Hint / Route Hint / Profile Draft
```

并记录请求字节、缓存命中、ETag/Last-Modified、deadline、QPS、总预算和 trace。

### 4.5 SEO 分数不能决定历史文章是否存在

老博客、早期静态站、自建工程博客常见：

- 无 canonical；
- 无 schema.org；
- meta description 不规范；
- title 长度不符合 SEO 模板；
- URL 包含年份或 query。

这些都不等于正文无价值。因此 SEO/head 信号只能用于 priority、route、onboarding diagnosis，不能从 `FULL_BACKFILL`/`RECONCILE` 的 Coverage Candidate 集合中删除 URL。

### 4.6 ContentTypeFilter 主要是 URL 扩展名启发式

v0.7.0 `ContentTypeFilter` 主要根据 URL 扩展名映射 MIME；没有扩展名时通常放行。它适合提前排除明显图片/压缩包 URL，但不能替代真实响应 `Content-Type` 与 MIME sniff。

生产 Fetch 后仍必须执行：

- response header 校验；
- MIME sniff；
- 最大响应字节；
- 解压膨胀比；
- PDF/HTML/JSON 等 Representation Route。

## 5. `stream=True`：方向正确，但项目没有把收益传给用户

项目 Deep Crawl 使用：

```python
stream=True
```

理论上可让抓完的页面逐个产出，适合：

- 渐进显示结果；
- 先处理高价值页面；
- 边抓边写 Snapshot；
- 边抓边做 Quality/Identity；
- 及时取消。

但项目 `deep_crawl()` 仍把所有成功结果 append 到 Python 列表；Flask 路由等抓取完成后才调用 Gemini，最后一次性 `render_template()`。

因此当前实现：

- 没有 SSE/WebSocket；
- 用户看不到中间结果；
- 内存仍随结果增长；
- Web 请求仍承担整个长任务生命周期；
- 浏览器/模型任一阶段超时都会拖住请求 worker。

生产链路应改为：

```text
POST /runs
 -> durable Run ID
 -> worker stream page result
 -> Snapshot/Candidate/Quality/Version
 -> progress event
 -> Web SSE/WebSocket/polling
 -> 用户可查看/取消/重试
```

`stream=True` 是 Worker 内部的数据流能力，不应等同于“Web 已经流式”。

## 6. AI 汇总实现的扩展性问题

`genai.py` 直接执行：

```python
context = "\n\n".join(markdowns)
full_prompt = f"Context:\n{context}\n\nQuestion: {prompt}"
```

然后调用 `gemini-2.0-flash-lite`。

这种“多篇 Markdown 拼一个大字符串”有以下问题：

1. token 随页面数和正文长度线性增加；
2. 没有 Document/Version 边界；
3. 没有 chunk lineage；
4. 没有 source block refs；
5. 没有输入集合 hash；
6. 没有可解释截断；
7. 没有单篇摘要缓存/复用；
8. 某个模板页、重复页、错误页会污染整体上下文；
9. 模型重试可能重新支付全部输入成本；
10. 无法稳定重放“当时到底汇总了哪几篇版本”。

生产方案应保持：

```text
Accepted Document Version
 -> AI Input Projection
 -> block-aware Chunk Plan
 -> per-document Map Artifact
 -> Selection Manifest
 -> Reduce/Digest Artifact
```

并记录：

- ordered version ids；
- input set hash；
- recipe/model/runtime release；
- token budget；
- truncation evidence；
- source block refs；
- cost/latency/status。

## 7. Web、运行时与可靠性问题

### 7.1 长任务同步运行在 Flask 请求线程

`routes.py` 在 POST 中直接：

```text
asyncio.run(crawl)
 -> Browser/HTTP
 -> collect markdown
 -> Gemini
 -> render HTML
```

没有 durable queue、lease、heartbeat、retry、resume、dead-letter、cancel propagation。

生产系统不能把这种模式用于批量抓取。Web 只负责创建 Command/Run、查询状态、展示事件和发起显式控制动作。

### 7.2 每次调用创建新的 `AsyncWebCrawler`

`deep_crawl()` 使用：

```python
async with AsyncWebCrawler() as crawler:
```

作为 Demo 很方便，但 1000 站点长期运行时会重复承担 Browser/Context 初始化成本，并难以统一管理：

- 最大总并发；
- per-site 并发；
- context/page 复用；
- crash recycle；
- idle timeout；
- 最大寿命；
- 内存阈值；
- busy pin；
- 浏览器版本 attestation。

生产应使用 Browser Runtime Pool，并让 Crawl4AI 只作为 Fetch Adapter/Runtime Adapter 之一。

### 7.3 `result.success` 只是传输/工具成功，不是知识库内容成功

项目只在：

```python
if result.success:
```

时收集 Markdown。

但 HTTP/Browser 成功不等于正文有效，可能拿到：

- 登录页；
- WAF challenge；
- 空壳 SPA；
- 404 soft page；
- 分类列表；
- cookie banner；
- 导航模板；
- 极短/异常正文。

生产必须再经过 Extraction Candidate、Canonical IR 和 Quality Gate，区分 transport outcome 与 content outcome。

## 8. 安全问题

### 8.1 Gemini Token 被日志打印

`routes.py` 的调试输出包含：

```python
print(... gemini_token: {gemini_token})
```

这是明确的 Secret 泄漏风险。生产日志、trace、error report 必须默认 redaction。

### 8.2 Token 被放入 Flask session

代码：

```python
session['gemini_token'] = token_input
```

项目没有配置服务端 Session 扩展；Flask 默认是 `SecureCookieSessionInterface`，即 session 数据由签名 cookie 承载，重点是完整性保护而不是把 API Secret 当作服务器端机密存储。

生产设计不能把模型/API Key 放入浏览器 cookie 或普通 session。应只保存 Secret Manager reference，Worker 在执行时用最小权限短期解引用。

### 8.3 默认 Secret Key 与 Debug 模式不适合生产

`app.py`：

```python
app.config['SECRET_KEY'] = os.getenv('SECRET_KEY', 'supersecretkey')
...
app.run(debug=True)
```

默认固定 secret 与 debug server 都只适合本地开发。生产 Web 必须：

- 强制外部 Secret；
- 禁止默认密钥；
- 禁止 debug；
- 使用正式 ASGI/WSGI runtime；
- 有 RBAC/CSRF/CSP/secure cookie 策略；
- Secret 永不进入日志。

### 8.4 任意 URL 输入需要统一 Admission

表单只做 URL 格式校验，不等于 SSRF 防护。生产的 Web/Agent/Dify 任意 URL 抓取必须进入统一 Admission：

- 只允许 http/https；
- DNS/IP 校验；
- 拒绝 localhost/私网/link-local/metadata/reserved；
- redirect 每跳重新校验；
- Browser Worker 网络层 deny 内网；
- response bytes/deadline/content-type 限制。

## 9. 测试覆盖与“版本语义契约”

仓库只有 `tests/test_genai.py`，并且依赖真实 `GEMINI_API_TOKEN` 调外部 Gemini。没有覆盖：

- Deep/Simple 控件映射；
- crawler；
- Best-First 顺序；
- scorer/filter；
- max_pages 边界；
- timeout/cancel；
- SSRF/redirect；
- quality；
- secret redaction；
- Web contract。

尤其本项目暴露出两个普通单元测试很容易漏掉的**语义契约问题**：

### 9.1 Priority Direction Contract

固定一组：

```text
high-score URL = 0.8
low-score URL  = 0.0
```

断言高 `normalized_priority` 的 URL 在队列中先被执行。测试对象应是实际 pinned runtime，而不是 mock scorer。

### 9.2 Candidate Admission Contract

构造一个页面包含 100 个候选，最相关 URL 位于 DOM 最后，`fetch_budget=10`。

断言：

1. 100 个 observable link 都先进入 Candidate/Evidence；
2. scorer 能看到全部候选；
3. 预算只限制后续 fetch，不按 DOM 顺序预先裁掉未评分候选。

### 9.3 Web Command Contract

自动化点击/提交每个 UI 控件，断言：

```text
Deep Crawl -> FOCUSED_DISCOVERY/DEEP 预期 Run Type
Simple     -> TARGETED_FETCH/SINGLE 预期 Run Type
```

并比较 requested config 与 effective config，防止隐藏字段、字符串 truthiness 或前端默认值悄悄改变模式。

## 10. 对 1000 站点知识库的正确抽象

### 10.1 Coverage Candidate 与 Priority Hint 必须分离

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

### 10.2 `FOCUSED_DISCOVERY` 的推荐链路

```text
root URL + optional topic + budget
 -> durable FOCUSED_DISCOVERY Run
 -> Provider/DOM Candidate enumeration
 -> persist Candidate/Evidence
 -> LOCAL_PREDICATE or METADATA_PROBE
 -> raw score
 -> priority adapter release
 -> normalized priority
 -> deterministic frontier
 -> fetch
 -> Snapshot
 -> Extraction/Quality
 -> Accepted Version
 -> progressive Web events
```

### 10.3 预算不等于完整性

`max_pages`、`max_depth`、`top_k`、score threshold、deadline 都只是执行预算/交互停止条件，不能生成 `coverage_complete`。

### 10.4 “已经观察到链接”与“决定抓链接”必须分开

对已经成功获取的页面，其可观察内链应先形成 Evidence，再由策略决定是否在本次预算内抓正文。这样：

- 可以后续重排，不必重抓父页面；
- 不受第三方库 pre-score truncation 影响；
- 能解释哪些 URL 因预算未抓；
- 可为 FULL_BACKFILL/Reconcile 复用 link evidence；
- Priority Policy 升级可离线重放。

## 11. 对最终技术方案的增补

在已有方案的 Coverage、Metadata Probe、Focused Discovery、Snapshot、AI Artifact 等设计基础上，本次进一步补强以下内容：

1. **Priority Contract 标准化**：保存 `raw_score` 与 `normalized_priority`，统一“数值越大越先处理”的业务语义，由版本化 Adapter 映射第三方队列方向；
2. **Candidate-before-budget**：页面已发现的链接先落 Candidate/Evidence，再评分和预算准入，禁止按 `remaining_capacity`/DOM 顺序在评分前裁掉；
3. **第三方 Runtime 语义契约测试**：每个 pinned Crawl4AI/抓取策略版本必须验证高低分顺序、max_pages/max_depth 边界、stream/cancel、Filter 副作用；
4. **Web Command 强类型化**：UI action 使用 enum/schema，持久化 requested/effective run type 与 config，禁止 hidden boolean/string truthiness 决定核心模式；
5. **Web Control Contract Test**：前端按钮到持久 Run 的映射必须端到端测试；
6. **Secret 传递边界明确化**：模型/API Token 不进入客户端 session/cookie、日志或 trace，只传 Secret reference。

这些优化不改变已有主结论：完整历史仍由可枚举 Provider + Coverage Evidence 证明；相关性只决定“先抓谁”，不能决定“谁不存在”。

## 12. 不应照搬的设计

以下做法不进入生产主方案：

1. 在 Web 请求内同步完成 Browser/深度抓取/LLM；
2. 每次请求创建新的完整 Browser crawler；
3. 用 `max_pages`、`max_depth` 或队列空了证明历史完整；
4. 用 SEO/Prompt 相关度过滤 FULL_BACKFILL 的合法文章；
5. 把多篇 Markdown 直接拼成一个无结构 LLM Context；
6. 把模型 Token 放普通 session/cookie 或打印到日志；
7. 只以 `result.success` 判断知识库正文成功；
8. 不保存 Snapshot、Version、Release、score evidence 和 task lineage；
9. 把 `URLPatternFilter(["*", ...])` 当作有效文章路径过滤；
10. 把 URL 关键词命中叫作正文语义相关性；
11. 直接把第三方 `score` 当调度 priority，不验证方向和边界；
12. 在评分前因为 `remaining_capacity` 按 DOM 顺序截断已观察到的候选；
13. 用隐藏字符串布尔值决定 Deep/Simple 等核心运行模式。

## 13. 主要源码依据

- 项目仓库：https://github.com/cf2018/crawl4ai_news_summarizer
- `requirements.txt`：固定 Crawl4AI 0.7.0、Playwright 1.53.0、Flask 3.1.1、google-genai 1.25.0
- `app/crawler.py`：FilterChain、KeywordRelevanceScorer、BestFirst、stream=True
- `app/routes.py`：Simple/Deep 分支、同步执行 crawler/Gemini、日志打印 token
- `app/templates/index.html`：`action=deep/simple` 与隐藏 `simple=1` 控件绑定
- `app/genai.py`：多 Markdown 直接拼接成 Context 后调用 Gemini
- `app/app.py`：默认 `supersecretkey` 与 `debug=True`
- `tests/test_genai.py`：仅真实 Gemini API 测试
- Crawl4AI v0.7.0 `crawl4ai/deep_crawling/bff_strategy.py`：PriorityQueue、score 入队、remaining_capacity 截断
- Crawl4AI v0.7.0 `crawl4ai/deep_crawling/scorers.py`：KeywordRelevanceScorer 为“命中越多分越高”
- Crawl4AI v0.7.0 `crawl4ai/deep_crawling/filters.py`：FilterChain、URLPatternFilter、ContentTypeFilter、SEOFilter/HeadPeekr
- Flask `SecureCookieSessionInterface`：默认 cookie session 机制
