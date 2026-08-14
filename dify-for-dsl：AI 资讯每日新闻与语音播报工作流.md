# dify-for-dsl：AI 资讯每日新闻与语音播报工作流

- 项目地址：https://github.com/wwwzhouhui/dify-for-dsl
- 重点文件：`dsl/crawl4ai/aibase_craw4fastapi.py`
- 工作流：`dsl/AI资讯每日新闻+语音播报工作流.yml`
- Crawl4AI 依赖：`crawl4ai==0.4.247`
- 调研目标：分析其“列表发现 → Crawl4AI 结构化抽取 → FastAPI 暴露 → Dify 编排 → LLM 摘要 → TTS 交付”链路，并提炼适用于 1000 个技术博客长期同步平台的设计原则。

## 1. 项目实现链路

该项目把采集能力拆成一个很小的 FastAPI 服务，再由 Dify 工作流编排 AI 与语音能力，主要链路如下：

```text
用户选择新闻数量
  -> Dify Code 节点请求 http://127.0.0.1:8086/news/?limit=N
  -> FastAPI get_news()
  -> requests.get("https://www.aibase.com/zh/news")
  -> BeautifulSoup 扫描 a[href]
  -> 过滤 /zh/news/ 详情链接
  -> 逐 URL 创建 AsyncWebCrawler
  -> JsonCssExtractionStrategy 按 CSS Schema 抽 title/date/content
  -> 拼成 news[] + newsdetail
  -> Dify LLM 对 news[] 总结
  -> Template 合并“摘要 + 详细正文”
  -> TTS 工具生成音频
  -> Code 节点把音频地址包装成 HTML <audio>
  -> Dify 直接回复文本和音频
```

这个设计很适合演示“爬虫服务作为 AI 工作流工具”，因为职责边界直观：采集端只负责取内容，Dify 负责摘要和交付。但其实现规模是单站点、少量新闻、即时调用，不能直接扩展为 1000 站点历史知识库。

## 2. 列表发现实现与原理

`get_news_urls()` 使用同步 `requests.get()` 拉取 `https://www.aibase.com/zh/news`，随后用 BeautifulSoup 遍历所有 `<a href>`，通过路径包含 `/zh/news/` 判断详情页。

### 2.1 优点

- 逻辑简单，目标站点结构固定时成本很低。
- Discovery 与 Article Extraction 是两个阶段，至少没有直接把列表页文本当成文章正文。
- 基于 DOM 链接发现，比“先转 Markdown 再用正则找 URL”更可靠。

### 2.2 不足

1. **没有 URL 去重。**同一链接可能因导航、推荐区等重复出现。
2. **没有 URL normalize/canonical。**tracking 参数、相对路径、尾斜线、canonical 迁移都没有处理。
3. **只扫描当前列表页。**没有 Sitemap、RSS、分页、Archive、Category、API 等历史枚举能力，因此无法证明全量历史完整性。
4. **没有 Evidence。**无法回答某 URL 来自哪个 Provider、哪个列表页、何时发现、历史枚举是否结束。
5. **没有增量 checkpoint。**每次调用都会重新扫列表页面。
6. **同步 `requests` 没有显式 timeout。**在 FastAPI 请求链路中遇到慢站点会放大请求占用。

### 2.3 对知识库方案的启示

列表发现必须抽象为版本化 Discovery Provider，而不是写死在一个函数里。每个 Provider 输出 URL Candidate + Evidence；Sitemap、Feed、API、Archive、DOM Link 都是不同 Provider。历史完整性使用 Coverage Evidence 表示，不能用“列表扫完”“当前页没有新链接”作为完成证明。

## 3. CSS 结构化抽取实现与原理

项目使用 `JsonCssExtractionStrategy`，Schema 大致为：

```text
baseSelector: div.pb-32
fields:
  title -> h1
  publication_date -> div.flex.flex-col > div.flex.flex-wrap > span:nth-child(6)
  content -> div.post-content
```

Crawl4AI 负责浏览/获取页面，然后按 CSS Schema 输出结构化 JSON。

### 3.1 价值

- 对已知站点，CSS/XPath 结构化抽取通常比 LLM 抽取便宜、快、确定性高。
- `title / publication_date / content` 显式分字段，优于只生成一个模糊的 `content` 字符串。
- Schema 可以自然演进为站点配置。

### 3.2 主要风险

`span:nth-child(6)` 这类位置选择器非常脆弱；只要站点增加一个标签、作者项或 UI 元素，发布日期就会错位。`div.pb-32` 这类样式类名也可能随着前端重构变化。

项目当前只检查 `result.success`，随后直接访问 `extracted_data[0]['title']`。因此存在以下情况：

- HTTP/Browser 成功，但抽取为空；
- title 存在，content 是导航或模板；
- publication_date 抽错但整体仍被视为成功；
- selector miss 导致数组为空并抛异常；
- 页面返回挑战页/登录页但 crawler transport 成功。

### 3.3 对知识库方案的启示

需要把“网络成功”“抽取成功”“内容通过质量门禁”分成不同 Outcome。CSS Schema 应进入 `site_profile_release / extractor_release`，并配套：

- Golden URL；
- selector miss 指标；
- metadata provenance；
- JSON-LD/OpenGraph/Readability/Trafilatura fallback；
- Snapshot replay；
- 新规则 shadow 对比；
- Drift 检测与回滚。

这样站点改版时只发布新的配置/Extractor Release，不改核心爬虫代码，也不必重新联网回抓全部历史页面。

## 4. Crawl4AI 生命周期与并发问题

项目在 `extract_ai_news_article(url)` 内部使用：

```text
async with AsyncWebCrawler(...) as crawler:
    await crawler.arun(...)
```

而 `fetch_news()` 对 URL 使用普通 `for` 循环逐个 `await`。

这意味着每篇文章都创建/销毁一次 crawler 生命周期，并且文章之间串行处理。对于只取 2~5 条新闻尚可，但扩展到大量文章会出现：

- 浏览器/会话初始化开销被重复支付；
- 连接池无法充分复用；
- 单次请求耗时与文章数近似线性增长；
- Dify 上游 HTTP 请求长时间占用；
- 某一篇卡死会拖住整个调用；
- 无 durable task，进程重启后无法从中断位置恢复。

### 4.1 正确的规模化方式

生产系统应该维护长生命周期 Browser/HTTP Runtime Pool，并把抓取工作物化为持久 Task：

```text
Run
  -> URL Task 1 -> lease -> Worker
  -> URL Task 2 -> lease -> Worker
  -> ...
```

Worker 内部再使用有界 semaphore 控制本机并发。进程内并发只是资源保护，不能取代持久调度。

Browser Context/HTTP Session 必须有：

- 全局和 per-site hard cap；
- idle eviction；
- in-flight protection；
- 最大寿命/页面数；
- finally 回收；
- pool utilization/eviction/leak 指标。

## 5. `bypass_cache=True` 的语义问题

示例明确设置 `bypass_cache=True`，意图是“确保新闻是最新的”。对于实时 Demo 合理，但长期知识库不能把“绕过缓存”当 freshness 策略。

真正的增量同步应该先判断变化信号：

```text
Feed/Sitemap/API updated
  -> ETag / Last-Modified
  -> body hash
  -> Canonical IR hash
```

只有变化或未知时才抓正文。即使网络表示发生变化，只要 Canonical IR hash 不变，也不应创建新 Document Version，更不应重复 Embedding、摘要或 TTS。

因此 freshness 与 cache 是两个概念：缓存只是性能优化；Freshness Observation 是业务事实。

## 6. FastAPI 边界存在的典型问题

项目的 `/news/` 是同步语义接口：请求进入后，服务立即完成列表抓取、N 篇文章抓取和结构化抽取，再返回结果。

同时列表页使用同步 `requests.get()`，处在 async FastAPI 调用链中。这会带来 Event Loop 阻塞风险。更重要的是，无论底层改成异步 HTTP 还是 Browser，这类长 I/O 都不适合由 Web 请求线程直接等待。

### 6.1 生产接口应改为 Job 语义

推荐：

```text
POST /runs or /derived-jobs
  -> 202 Accepted + job_id

GET /jobs/{job_id}
  -> status/progress/result_ref

GET /artifacts/{artifact_id}
  -> 已完成的 Markdown/摘要/Digest/TTS 等结果
```

Dify、n8n、Agent、Web 管理端都应该调用控制面创建任务，随后轮询状态、订阅 webhook，或查询已完成的 Projection，而不是让它们直接驱动 crawler 在一次 HTTP 请求中抓站点。

这也是把“知识库同步平台”和“AI 消费工作流”解耦的关键。

## 7. Dify 工作流编排的实现细节

工作流包含几个关键节点：

1. Start：用户选择新闻数量。
2. Code：使用 `requests.get('http://127.0.0.1:8086/news/')` 调采集服务。
3. LLM：将 `news` 列表整体交给模型总结。
4. Template：把 LLM 摘要与 `newsdetail` 全文重新拼接。
5. TTS Tool：把拼接文本交给语音模型。
6. Code：解析 TTS 返回 JSON，拼出 `<audio>` HTML。
7. Answer：同时返回文字和音频。

这个工作流体现了一个值得保留的思想：**采集、AI 变换、交付是三个层次，AI/语音可以在工作流层组合，而不是塞进爬虫核心。**

但实现中也暴露出生产化问题。

## 8. LLM 输入与 Digest 生成问题

当前 Dify 节点把 `news[]` 直接送给一个 LLM。当文章数量和长度增加时，会出现：

- 超过上下文窗口；
- 发生不可见截断；
- 输入成本随原文总长度增长；
- 某篇超长文章挤占其他文章；
- 无法复用单篇摘要；
- 失败后只能整批重跑；
- 无法追溯摘要的某一部分来自哪个 Document Version。

### 8.1 适用于知识库的 Map/Reduce

应该改成：

```text
Accepted Document Version x N
  -> 每篇建立 AI Input Projection
  -> block-aware chunk
  -> 单篇 Map Summary
  -> 单篇 Reduce Summary
  -> Digest Selector/Ranker
  -> 多篇 Digest Reduce
  -> Final Digest Artifact
```

每个中间 Artifact 保存：

- 输入 document_version_id；
- block range；
- recipe release；
- model release；
- runtime attestation；
- input/output hash；
- token、截断、延迟、成本。

这样新增一篇文章只计算新增部分，历史单篇摘要可以复用。

## 9. TTS 设计问题

示例把“LLM 摘要 + 新闻详细正文”整体送 TTS，并在注释中指出 2 条新闻生成语音大约需要 5 分钟。这说明 TTS 已经是明显的高成本、长耗时派生任务。

对知识库平台，TTS 应作为 `Derived Artifact / Delivery`，不能属于 Source Sync 成功条件。

推荐：

```text
Digest Artifact
  -> TTS Recipe
  -> Audio Artifact
  -> Object Storage
  -> Delivery Projection
```

默认只对摘要/Digest 做 TTS，而不是把所有原文全文转语音。配置必须显式限制：

- 最大输入字符/token；
- 最大目标音频时长；
- 单任务 deadline；
- 模型/音色 release；
- 并发/GPU 配额；
- 失败重试；
- 成本预算。

TTS 失败只标记派生任务失败，不影响 Markdown、搜索索引或原文版本。

## 10. 工作流中的服务发现与安全问题

Dify Code 节点写死 `http://127.0.0.1:8086/news/`，这是单机部署假设。容器化后，`127.0.0.1` 指向 Dify 自己的容器而不一定是 crawler 服务。

生产系统应使用：

- 服务发现或配置化 Endpoint；
- mTLS/API Token；
- Secret Manager；
- connect/read/total timeout；
- response byte limit；
- retry budget；
- circuit breaker；
- request id / trace id。

此外，不能让 Dify 工作流把任意 URL 直接传给内部 crawler 绕过 SSRF Admission。所有 Targeted Fetch 必须进入标准 URL Admission、安全 DNS/egress、robots/compliance、Snapshot、Quality 和 Audit 主链路。

## 11. 返回 Contract 的问题

FastAPI 返回：

```text
{
  "news": [...],
  "newsdetail": "今天新闻第1条内容：..."
}
```

其中 `newsdetail` 是把多个原文拼成一个无结构字符串。随后 Dify 又需要对可能是 dict/string 的 `news` 做兼容解析，说明 Contract 边界已经开始漂移。

长期系统不能在不同 Stage 反复使用模糊 `content/newsdetail/text` 字段。推荐固定 representation：

```text
DOCUMENT_VERSION_REF
MARKDOWN_PROJECTION_REF
AI_INPUT_PROJECTION_REF
AI_DERIVED_ARTIFACT_REF
AUDIO_ARTIFACT_REF
DELIVERY_PAYLOAD_REF
```

大型正文不要在服务之间反复复制 JSON；传 object key/artifact id，由下游按权限读取。

## 12. 由该项目提炼出的架构优化

### 12.1 增加 Workflow / Delivery Integration Plane

现有知识库系统除了 Source Sync、Search、AI 外，还需要一个明确的消费集成层，面向 Dify、n8n、Agent、Webhook、Newsletter、TTS 等外部工作流。

其职责：

- 创建 Derived Job；
- 按时间/source/tag/query 选择 Accepted Version；
- 调用版本化 AI/TTS Recipe；
- 生成 Derived Artifact；
- 通过 webhook/API/feed/object link 交付；
- 维护重试、幂等、状态与审计。

外部工作流不拥有抓取真相，也不直接修改 Canonical Document。

### 12.2 增加 Delivery Job 数据模型

建议：

```text
delivery_job
- id
- trigger_type
- scope_json
- input_version_set_hash
- recipe_release_id
- destination_adapter_release_id
- idempotency_key
- status
- attempt
- next_retry_at
- output_artifact_id
- error_class
- created_at/finished_at
```

### 12.3 增加 Derived Artifact 的可复用性

```text
derived_artifact
- id
- type                    # SUMMARY/DIGEST/FAQ/TTS/NEWSLETTER/REPORT
- input_version_ids
- input_projection_hash
- parent_artifact_ids
- recipe_release_id
- model_release_id
- runtime_attestation_id
- object_key
- content_hash
- status
- token/cost/latency
- created_at
```

相同 `input version set + recipe + model/runtime` 可直接复用结果，避免 Dify 每次请求都重做摘要和语音。

### 12.4 增加 Recipe Studio / Delivery 页面

Web 管理端应允许：

- 配置 Digest 选文规则；
- 配置摘要/合并/TTS Recipe；
- 预览本次将使用哪些 Document Version；
- 查看 token、截断、成本与预计音频长度；
- 测试 webhook；
- 查看 Delivery Job 重试；
- 对比 Recipe Release；
- 手工触发，但仍只创建持久 Job。

## 13. 与 1000 技术博客场景的对应关系

这个项目最大的价值不是其单站点爬取代码本身，而是证明了一个重要产品形态：**抓取后的内容会被工作流、LLM、TTS、Agent 等二次消费。**

因此知识库架构不能只做到“抓下来并保存 Markdown”，还需要把稳定的 Document Version 作为可编排数据产品输出。但为了保持可扩展性，必须反转 Demo 的调用关系：

### Demo 方式

```text
Dify 请求
 -> 临时抓网页
 -> 临时抽取
 -> 临时总结
 -> 临时 TTS
 -> 返回
```

### 生产知识库方式

```text
长期 Source Sync
 -> Accepted Version
 -> 可重建 Markdown/Search/Embedding
 -> 可缓存 AI Summary/Digest/TTS Artifact
 -> Dify/n8n/Agent 只消费稳定 Artifact 或提交异步 Derived Job
```

这样才能同时满足：

- 1000 站点独立同步；
- 全量历史完整性；
- 增量变化检测；
- 不重复抓站；
- 不重复做昂贵 AI；
- 外部工作流失败不污染原始知识库；
- Web 管理可追踪；
- 后续新增 Dify/n8n/Newsletter/TTS 等消费者时无需改变抓取核心。

## 14. 最终结论

`dify-for-dsl` 的 Crawl4AI 示例非常适合作为“站点专用 Extractor + AI 工作流消费”的最小实现参考。可直接借鉴的是：

- CSS Schema 做确定性结构化抽取；
- Crawl4AI 与业务工作流解耦；
- LLM/TTS 作为内容的下游派生能力；
- FastAPI 作为能力边界。

不能照搬到生产规模的部分包括：

- 单列表页发现；
- 同步请求链路即时抓站；
- FastAPI async 路径内使用阻塞 `requests`；
- 每篇文章新建 crawler；
- 串行逐篇抓取；
- `bypass_cache=True` 代替增量策略；
- 写死 CSS/localhost/模型调用；
- 把整批全文直接塞给 LLM/TTS；
- 缺少 durable task、Snapshot、Version、Quality、Coverage、Audit 与幂等。

对博客知识库方案的实质优化，是新增一个正式的 **Workflow / Derived Artifact / Delivery Plane**：爬虫负责长期同步真相，AI 工作流负责基于 Accepted Version 生成可重建派生结果，Dify/n8n/TTS 等成为消费者与编排器，而不是临时触发爬站的控制中心。
