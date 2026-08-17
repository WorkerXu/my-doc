# PicoNewsAgent：新闻自动采集、生成与发布智能体

- 调研编号：53
- 项目地址：https://github.com/HosLak/PicoNewsAgent
- 项目定位：RSS/Atom 聚合 + 轻量预处理 + LLM 选稿 + Crawl4AI 深抓 + LLM 改写 + Telegram 发布
- 调研目标：深入分析其 Source 管理、Feed 轮询、并发模型、去重、Crawl4AI 深抓、媒体提取、LLM 输出契约、发布降级与状态语义，并判断哪些能力适合吸收到“约 1000 个技术博客全量历史 + 增量同步 + Markdown 知识库 + Web 管理”方案中。

## 1. 总体结论

PicoNewsAgent 不是历史知识库爬虫，而是一个典型的实时内容运营流水线：

```text
Sources.md
 -> Feed 并发抓取
 -> 最近时间窗过滤 + 运行内去重
 -> LLM 选出高价值条目
 -> 只对入选条目用 Crawl4AI 深抓全文/图片
 -> LLM 生成 Telegram 内容
 -> Telegram Delivery + 失败降级
```

它最值得借鉴的不是具体代码，而是四个架构信号：

1. **便宜的元数据发现和昂贵的正文抓取分层。** Feed 用普通 HTTP 并发抓，正文 Browser 深抓只处理少量候选。
2. **媒体候选来自多个入口。** Feed media、页面 OpenGraph、Crawl4AI media 可以共同形成图片候选。
3. **AI 生成与 Delivery 是正文抓取之后的派生链。** 即使最终业务是发布，也可以和源数据采集解耦。
4. **来源维护体验非常低摩擦。** 直接往 `Sources.md` 加 Feed URL 就能扩展来源。生产知识库不能把 Markdown 文件当真相源，但应该保留这种“批量粘贴/导入即可接入”的运营体验。

现有博客知识库方案已经覆盖 Feed Change-Signal Plane、两级队列、长期 Browser Pool、相似度证据、Asset provenance、人工审核状态机、AI/Delivery 独立失败等主要启发。本次进一步识别出的可落地新增点是：**增加 Source Intake / Bulk Import 层，让未来持续新增网站时可以从 URL、Feed、OPML、Markdown、CSV/JSON 批量导入，经规范化、去重、站点归属解析、Probe、预览和审核后再进入正式 Source Registry。**

## 2. 代码结构与实际执行语义

入口 `main.py` 顺序执行五个阶段：

```python
fetcher.run()
processor.run()
agents.select_news()
await agents.generate_posts()
publisher.run()
```

对应：

- `core/fetcher.py`：读取 Source、HTTP 拉 Feed、解析条目和首图；
- `core/processor.py`：日期过滤、URL 去重、标题模糊去重；
- `core/agents.py`：Choicer、Crawl4AI 深抓、Writer、Shortener；
- `core/publisher.py`：Telegram 发布和 fallback；
- `config.py`：模型、并发、超时、时间窗、文件路径等全局配置；
- `prompts/*.md`：把选稿、写作、缩短逻辑从 Python 代码中分离。

这种结构清楚但属于单进程批处理编排。任一阶段大量依赖 `data/*.json` 文件作为阶段间状态，缺少数据库事务、任务 lease、跨进程幂等、run manifest 和精确 stage outcome。因此“小规模每天跑一次”可用，但不能直接扩展到百万文档、多实例长期运行。

## 3. Source 管理：低摩擦体验值得保留，文本文件真相源不适合生产

### 3.1 项目实现

`Sources.md` 本质是一个 Feed URL 列表。当前仓库中包含 OpenAI、Google AI、Hugging Face、AWS、DeepMind、Substack、TechCrunch、arXiv 等 Feed。

`extract_rss_links()` 同时用两类正则寻找 Markdown 链接和裸 URL：

```python
md_links = re.findall(r'\]\((https?://[^\s)]+)\)', content)
bare_links = re.findall(r'(https?://[^\s\)]+)', content)
```

随后通过 URL 中是否包含 `.xml`、`/rss`、`/feed`、`/atom`、`feed=` 粗略判断是否为 Feed。

### 3.2 优点

- 增加新来源几乎没有学习成本；
- Source 配置和程序代码分离；
- 同一 Fetcher 可以处理大量异构来源；
- 很适合个人工具或小团队快速运营。

### 3.3 局限

1. 文本 URL 没有稳定 `site_id/provider_id`；
2. 无启停状态、负责人、调度、优先级、robots、速率限制、凭据绑定；
3. 无版本和审计，Web 多人编辑时无法保证并发一致性；
4. 仅按 URL 关键字识别 Feed，既可能漏识别，也可能误识别；
5. 一个 Feed URL 和它所属的站点实体没有显式关系；
6. 不能表达“同一站点有 Sitemap + Feed + Archive + API 多个 Provider”。

### 3.4 对 1000 站知识库的新增启发：Source Intake

生产方案不应该回退到 `Sources.md` 做真相源，但应该提供等价甚至更低摩擦的接入入口：

```text
粘贴 URL / Feed URL
上传 OPML
上传 Markdown / TXT / CSV / JSON
Web 批量录入
       |
       v
Source Intake Batch
 -> parse
 -> normalize
 -> classify root/feed/provider
 -> duplicate detection
 -> canonical site association
 -> lightweight probe
 -> dry-run preview
 -> operator approve
 -> Site / Provider / Profile Release
```

关键原则：**导入文件只是输入证据，不是 Source Registry。** 原始导入内容和行号需要保存 provenance，正式站点配置仍由 PostgreSQL 的版本化实体承担。

## 4. Feed 网络层：轻量高并发与 Browser 资源池分离

### 4.1 HTTP 重试

`fetch_with_retries()` 使用 `requests.get()`，设置浏览器风格 User-Agent、RSS/XML Accept、请求超时，并对异常做指数退避：

```python
wait_time = 2 ** attempt
```

默认配置：

```text
MAX_WORKERS = 12
MAX_RETRIES = 3
REQUEST_TIMEOUT = 10
MAX_ITEMS_PER_FEED = 50
```

### 4.2 并发模型

`run()` 用 `ThreadPoolExecutor` 并行获取 Feed：

```python
with ThreadPoolExecutor(max_workers=config.MAX_WORKERS) as executor:
    future_to_url = {
        executor.submit(fetch_and_process, url): url
        for url in rss_links
    }
```

这与 Browser 深抓的 `MAX_CONCURRENT_CRAWLS = 3` 形成明显资源分层：Feed 属于轻 I/O，Browser 属于高内存/高 CPU/高延迟任务。

### 4.3 生产化需要补齐

对于 1000 站不能只设置一个全局 worker 数，至少需要：

```text
global concurrency
 -> worker-pool quota
 -> host/domain quota
 -> site quota
 -> task priority
```

另外重试需加入：

- jitter；
- `Retry-After`；
- 按 HTTP 状态分类可重试/永久失败；
- host 熔断；
- 动态降并发；
- ETag / Last-Modified 条件请求。

PicoNewsAgent 每轮直接 GET Feed，没有持久的 ETag / Last-Modified，因此即使 Feed 没变化也会重新下载、解析。对 1000 站长期增量而言，应把条件请求放进独立 Change-Signal Plane，304 时只记录 observation。

## 5. Feed 条目解析、时间和媒体候选

### 5.1 条目字段

`parse_feed()` 通过 `feedparser` 解析条目，并输出：

```text
source_url
标题 title
文章 link
lead_image
summary
published_date
fetched_at
```

相对链接用 `urljoin(base_url, link)` 归一化。summary 从 `summary` / `description` / `content[0]` 中取值，再用正则去 HTML 标签并截到 500 字符。

### 5.2 `MAX_ITEMS_PER_FEED` 的语义

项目只处理每个 Feed 前 `MAX_ITEMS_PER_FEED` 条，默认 50。这适合“最近一天新闻”，但不能用于历史完整性。

对知识库：

- `INCREMENTAL` 可以只看有限最新窗口，并且建议有 overlap；
- `FULL_BACKFILL` 绝不能把 Feed 的有限窗口当历史边界；
- Sitemap、API cursor、Archive/year-month 分页等才是历史覆盖的核心 Provider；
- Feed 只是一种变化信号和补充发现证据。

### 5.3 时间语义是一个典型反例

`normalize_date()` 优先 `published_parsed`，其次 `updated_parsed`，都没有时返回当前时间。

这对实时展示可以勉强工作，但对知识库会产生严重数据污染：**抓取时间不等于发布时间。**

生产字段必须拆开：

```text
source_published_at
source_updated_at
discovered_at
observed_at
fetched_at
```

源站发布时间未知就保存 NULL。每个时间字段还应保存 provenance，例如来自 Feed、JSON-LD、meta、API 或人工 Override。

### 5.4 图片候选的多通道思想

`extract_thumbnail()` 依次尝试：

1. `media_content`；
2. `media_thumbnail`；
3. entry links 的 image 类型；
4. `image/thumb/thumbnail/wp_post_thumbnail`；
5. summary/description/content 中的 `<img src>`。

并过滤 `pixel/analytics/icon/logo/avatar` 等明显非正文图片。

这说明“文章主图/正文图”应是候选集合，而不是单一 CSS selector。知识库 Asset Pipeline 应统一接收：

```text
Feed enclosure/media
OpenGraph
Twitter Card
JSON-LD
正文 img/source/picture
Crawl4AI result.media
站点专用 extractor
```

每个 `asset_candidate` 保存来源、source path、confidence、原 URL、发现页面和时间，再由 Asset Pipeline 下载验证、hash 去重和排序。

## 6. Processor：运行内去重适合日报，不适合文档身份

### 6.1 时间窗口

`processor.run()` 计算：

```python
cutoff_date = datetime.now() - timedelta(days=config.DAYS_BACK)
```

默认 `DAYS_BACK = 1`，旧条目直接过滤。这是编辑运营策略，不是采集完整性策略。

知识库的正确边界是：

- FULL_BACKFILL 不允许使用 DAYS_BACK 截断合法历史；
- INCREMENTAL 可以把时间窗作为扫描优化，但必须结合 Provider checkpoint/overlap/Reconcile；
- AI 或用户兴趣只能影响优先级和下游选稿，不能改变历史真相。

### 6.2 URL 和模糊标题去重

项目先用 `seen_urls` 精确去 URL，再用 `difflib.SequenceMatcher` 比较清洗后的标题，默认阈值 0.85。

这只能作为运行内降噪。它不适合作为生产 Document Identity，因为：

- 同一文章可能有 canonical、AMP、redirect、slug rename；
- 不同版本文章标题可能高度相似；
- 系列文章、翻译、转载、镜像不应被静默删除；
- 跨运行没有持久记忆。

生产模型应拆成：

```text
URL Identity
Document Identity
Document Version
Similarity Evidence / Cluster
```

精确 Canonical IR hash 可以判断“内容完全相同”；标题相似、SimHash、MinHash/LSH 只形成非破坏性相似证据，用于搜索降拥挤和人工审核。

## 7. LLM Choicer：可以做派生选稿，不能做知识库 Admission

### 7.1 实现

`select_news()` 把每个条目简化为：

```json
{
  "title": "...",
  "summary": "...",
  "link": "...",
  "published_date": "...",
  "lead_image": "..."
}
```

再把整个 JSON 交给 Gemini。`choicer_agent.md` 要求模型按频道主题和自定义条件筛选高影响新闻、去语义重复，并输出纯 JSON 数组。

模型调用设置 `temperature=0.2` 和 `response_mime_type="application/json"`，随后仍通过 `json.loads()` 解析。

### 7.2 值得借鉴的点

- Prompt 与代码分离；
- 明确输出结构；
- 低温度降低格式漂移；
- AI 处理建立在已抓到的元数据之上；
- 可以把人工运营标准编码为版本化 Prompt。

### 7.3 生产化要点

现有知识库方案中的 AI Stage 应继续要求：

```text
prompt_release
model_release
input_version_ids
output_schema
cost_budget
```

还应执行结构化验证：

```text
LLM response
 -> parse
 -> schema validate
 -> semantic validate
 -> persist AI artifact
 -> downstream action
```

解析失败必须是显式 Stage Outcome，不能只打印异常后继续假装流水线完成。

### 7.4 不能前置到主知识同步

Choicer 的“只选高价值内容”非常适合 Telegram，但如果照搬到知识库，会导致不可恢复的历史缺失。

正确关系：

```text
Source Sync
 -> Accepted Canonical Document Version
 -> Markdown/Search Ready
 -> AI Summary / Tag / Cluster / Digest / Editorial Selection
```

AI 只决定“怎么用”和“是否发布”，不能决定合法历史文章是否存在于知识库。

## 8. Crawl4AI 深抓：资源模型和数据保真是最重要的两个问题

### 8.1 Browser 配置

`crawl_single()` 使用：

```python
BrowserConfig(headless=True, verbose=False, text_mode=False)
```

以及：

```python
CrawlerRunConfig(
    wait_for="body",
    wait_until="networkidle",
    wait_for_images=True,
    remove_overlay_elements=True,
    delay_before_return_html=2.0
)
```

这说明项目优先保证 JS 页面和懒加载图片完成后再读内容。

### 8.2 并发限制

`crawl_all_parallel()` 用 `asyncio.Semaphore(MAX_CONCURRENT_CRAWLS)` 限制深抓并发，默认 3：

```python
semaphore = asyncio.Semaphore(config.MAX_CONCURRENT_CRAWLS)
```

这种“轻 Feed 高并发、Browser 低并发”应保留。

### 8.3 关键性能问题：每 URL 创建 crawler 生命周期

`crawl_single()` 内部每篇文章执行：

```python
async with AsyncWebCrawler(config=browser_cfg) as crawler:
```

因此 URL 级任务承担 Browser/Crawler 初始化和销毁成本。几十篇新闻问题不大，但百万级历史回灌会非常浪费。

生产形态应是：

```text
Browser Worker Process
 -> long-lived Browser Runtime
 -> Context Pool
 -> short-lived Page / crawl task
```

Browser 生命周期长，Page 生命周期短；Context 按站点/安全边界隔离；设置进程最大页面数、内存、CPU、任务数和定期 recycle。

### 8.4 HTTP-first 仍是更优默认

PicoNewsAgent 所有被选文章都走 Crawl4AI Browser，原因是候选量已经很小。对于 1000 站全量知识库，不能这样做。

应先普通 HTTP：

```text
HTTP Snapshot
 -> extractor quality pass? -> accept candidate
 -> app shell / empty / dynamic evidence? -> Browser fallback
```

Browser 是证据驱动的升级路径，不是默认抓取引擎。

### 8.5 正文保真问题

成功后代码优先取 `result.markdown`，否则 `result.html`，然后 `clean_content()`：

- 正则删除 script/style/nav/footer/header；
- 删除所有 HTML 标签；
- 压缩空白；
- **硬截断到 2500 字符。**

这非常适合 LLM 写 150~300 字新闻帖，却不适合知识库：

- 代码块丢失；
- 表格丢失；
- 标题层级丢失；
- 超长技术文档被截断；
- 图片与正文块的相对位置丢失；
- 无法稳定 diff/replay/rechunk。

知识库必须：

```text
Immutable Snapshot
 -> Extraction Candidate
 -> Structured Block IR
 -> Canonical Document IR
 -> Deterministic Markdown Projection
```

Markdown 可以作为候选输入，但不是唯一真相。

## 9. 页面媒体提取：候选与主图选择需要分离

Crawl4AI 深抓后：

- 从 `result.metadata` 读取 `og:image`、`twitter:image`、`og:image:secure_url`；
- 从 `result.media['images']` 收集图片；
- 过滤 `avatar/logo/icon/ad/banner`；
- 最多向 Writer 提供 5 个候选。

Writer Prompt 再要求模型只保留架构图、数据图、基准图、UI 截图等有信息价值的图片。

对知识库应拆成两个完全不同的职责：

1. **采集层不丢候选。** 只要合法且在 scope 内，尽量保存候选和 provenance；
2. **派生层可以选图。** Digest、Newsletter、社媒发布可以使用 AI/规则挑主图。

不能让“适不适合发 Telegram”反过来决定原始文章资产是否被归档。

## 10. Writer Agent：Prompt 版本化、结构化输出和渠道约束

`telegram_writer.md` 明确约束：

- 目标语言；
- 内容语气；
- 布局；
- 来源链接；
- Hashtag；
- Telegram HTML 允许标签；
- 禁止 Markdown；
- 输出 JSON：`telegram_post` + `approved_images`。

这体现了“一个 Accepted Article 可以派生多个渠道 Projection”的设计。

知识库系统可以把这抽象为：

```text
AI Workflow Release
 -> Stage Release
 -> Prompt Release
 -> Model Release
 -> Output Schema
 -> Channel Policy
 -> Artifact
```

例如同一 `document_version_id` 可以分别产生：

- 中文摘要；
- 英文摘要；
- Weekly Digest；
- Telegram Post；
- Slack/Discord 通知；
- Newsletter；
- 专题报告。

这些全部是可删除、可重跑、可重建的派生结果，不能修改 Document Version。

## 11. Publisher：独立 Delivery fallback 的设计值得直接吸收

`publisher.py` 根据图片数量分别走 Telegram：

```text
0 图 -> sendMessage
1 图 -> sendPhoto
多图 -> sendMediaGroup
```

发送失败后有两层 fallback：

1. “message too long” -> 调用 Shortener Agent 缩短后重试；
2. HTML 解析或媒体获取失败 -> 丢图、去 HTML/纯文本发送。

### 11.1 正确的抽象

生产系统应把这种策略定义在 `delivery_policy_release`：

```text
preferred payload
 -> channel length validation
 -> media validation
 -> send
 -> retry
 -> shorten fallback
 -> drop-media fallback
 -> plain-text fallback
 -> dead-letter
```

每次尝试保存：

```text
delivery_id
channel
payload_artifact_id
attempt
policy_release_id
request_at
response_code
remote_message_id
error_class
fallback_reason
ack_at
```

### 11.2 重要边界

Delivery 的任何缩短、去图、HTML 清理，只能改渠道 payload，不能回写 Canonical IR 或 Markdown。

## 12. 状态管理与成功判定：PicoNewsAgent 最大的生产化缺口

当前阶段间主要靠 JSON 文件：

```text
raw -> clean -> selected -> telegram_posts
```

存在的问题：

- 无数据库真相源；
- 无多实例并发控制；
- 文件写入缺少事务；
- 无跨运行持久去重；
- 中途失败难以精确恢复；
- 阶段函数可能打印错误后直接 return；
- `main.py` 最后仍打印“Pipeline Execution Completed Successfully”；
- 无法回答某篇文章在哪个阶段失败、用了哪个规则版本、是否已经发布过。

知识库必须继续使用：

```text
PostgreSQL authoritative state
+ S3/MinIO immutable artifacts
+ Redis Streams transport only
+ Transactional Outbox
+ idempotency key
+ lease/heartbeat
+ Run Manifest
+ Stage Finalizer
+ append-only Audit
```

**协程结束不等于业务成功，主函数走到最后也不等于 Run 成功。**

## 13. 配置与 Prompt 管理

`config.py` 把并发、超时、模型、时间窗、语言、语气、Source 文件和输出路径集中配置；Prompt 独立在 `prompts/*.md`。

优点是开发简单，但生产上全局可变配置会破坏可重现性。正确做法：

- Site/Profile/Policy/Prompt/Model/Chunker/Embedding 都生成 immutable release；
- Run 绑定具体 release ID；
- Secret 只保存 Secret Manager 引用；
- Web 修改配置时创建新 release，而不是原地覆盖；
- 任何 AI Artifact 保存 Prompt/Model/Input Version/Schema 版本。

这样才能在半年后回答“这篇 Markdown/摘要/发布内容为什么长这样”。

## 14. 新增优化：Source Intake / Bulk Import 的实现建议

这是本次在现有主方案之上最值得新增的能力。

### 14.1 为什么需要单独 Intake 层

用户目标不是只维护最初 1000 站，后续还会不断增加网站。如果每新增一个站都要求手工填写完整 Profile、Provider、Schedule，会形成明显运营瓶颈。

PicoNewsAgent 的 `Sources.md` 证明：**低摩擦添加来源非常有价值。** 生产系统应该把这种体验升级成可靠的批量导入流水线，而不是丢掉。

### 14.2 支持的输入

```text
单个 root URL
多个 root URL
直接 Feed URL
OPML
Markdown/TXT 裸 URL 或链接
CSV
JSON
Web 表格粘贴
```

### 14.3 数据模型

建议增加：

```text
source_import_batch
source_import_item
```

`source_import_batch`：

```text
batch_id
input_type
original_artifact_ref
created_by
status
created_at
parsed_count
valid_count
duplicate_count
needs_review_count
created_site_count
```

`source_import_item`：

```text
item_id
batch_id
row_or_line
raw_value
normalized_input
input_kind              # ROOT_URL / FEED_URL / OPML_ENTRY ...
resolved_site_root
resolved_provider_url
existing_site_id
existing_provider_id
probe_result_id
status
reason
provenance
```

### 14.4 状态机

```text
PARSED
 -> INVALID
 -> DUPLICATE
 -> MATCHED_EXISTING_SITE
 -> NEEDS_PROBE
 -> NEEDS_REVIEW
 -> READY_TO_APPLY
 -> APPLIED
```

### 14.5 站点和 Feed 的归属解析

如果用户直接输入 Feed：

```text
https://example.com/blog/feed.xml
```

Intake 不应简单把它当一个独立“站点”。应先根据：

- Feed channel link；
- HTML alternate feed；
- URL host/path；
- redirect；
- 已有 Site Registry；

解析到逻辑 `site`，再创建 `RSS_ATOM_JSONFEED` Provider。

### 14.6 Dry-run 与审核

批量导入先只做预览：

```text
输入 1000 条
 -> 920 个唯一逻辑站点
 -> 760 个新站
 -> 120 个已有站补充 Feed Provider
 -> 25 个重复
 -> 15 个无效/不可访问
```

运营人员在 Web 中确认后再创建正式 Site/Profile/Provider。这样既保留 `Sources.md` 的便利性，又不会破坏正式配置的一致性。

### 14.7 幂等

重复上传同一 OPML/Markdown 不应重复建站。可使用：

```text
import:{batch}:{normalized_input_hash}
site candidate: normalized root identity
provider candidate: site_id + provider_type + normalized endpoint
```

所有 Apply 操作写 Audit。

## 15. 与现有博客知识库主方案逐项对照

| PicoNewsAgent 机制 | 是否直接可用 | 主方案处理 |
|---|---|---|
| `Sources.md` URL 列表 | 不直接用 | 升级为 Source Registry，并新增 Bulk Source Intake |
| Feed 普通 HTTP 并发 | 可借鉴 | 独立 Signal Worker，高并发低成本 |
| 每次全量 GET Feed | 不足 | ETag / Last-Modified / 304 |
| Feed 前 50 条 | 仅增量启发 | Feed 是 Change Signal，不作为历史完整性证明 |
| 缺时间时写当前时间 | 禁止 | 发布时间 NULL + provenance |
| URL + 标题模糊去重 | 只作证据 | URL Identity + Document Identity + Similarity Evidence |
| LLM Choicer 先筛选再深抓 | 不用于知识主链 | AI 只在 Accepted Version 之后做派生/选稿 |
| Browser 深抓 | 可作为 fallback | HTTP-first，证据驱动升级 Browser |
| 每 URL crawler lifecycle | 不适合大规模 | 长期 Browser Runtime / Context Pool |
| 正文截 2500 字符 | 禁止 | Snapshot -> Block IR -> Canonical IR，不截断 |
| 多通道图片候选 | 值得借鉴 | `asset_candidate` + provenance + 独立 Asset Pipeline |
| JSON 文件传阶段 | 不适合 | PostgreSQL + S3 + Queue + Outbox + Manifest |
| Prompt 文件分离 | 值得借鉴 | Prompt/Model/Schema 全部 Release 化 |
| Telegram Shortener/fallback | 值得借鉴 | Delivery Policy + 独立 retry/dead-letter |
| Human-in-the-loop Roadmap | 值得借鉴 | 持久 Review State Machine + Audit |

## 16. 推荐生产数据流

结合该项目经验后，面向博客知识库的最终关系应是：

```text
Source Intake
 -> Site / Provider Registry
 -> Probe / Profile Release

Change-Signal Plane
 -> Feed / Sitemap / API metadata observations
 -> identity + freshness decision
 -> NEW / POSSIBLY_CHANGED

Discovery / FULL_BACKFILL
 -> Sitemap / API / Archive / Nav / Internal Link
 -> Coverage Ledger
 -> Durable Frontier

Fetch
 -> HTTP first
 -> Browser only on evidence
 -> Immutable Snapshot

Extraction
 -> Candidates
 -> Canonical Block IR
 -> Quality / Review
 -> Accepted Document Version

Projection
 -> Markdown
 -> Asset
 -> Chunk / Search / Embedding

Derived Workflow
 -> Summary / Tag / Cluster
 -> Digest / Newsletter / Editorial Selection
 -> Delivery
 -> channel-specific fallback
```

这里最关键的因果关系是：

- Feed 能决定“要不要优先重新抓”，不能决定“历史是否完整”；
- AI 能决定“要不要精选/怎么写”，不能决定“文章是否进入知识库”；
- Delivery 能改发布 payload，不能改源文档真相；
- 相似度能决定“如何聚类/降低搜索拥挤”，不能自动删除来源文章；
- Markdown 是稳定 Projection，不是唯一真相；
- 新来源导入可以很简单，但正式配置必须结构化、可审核、可版本化。

## 17. 本次对主方案的实际修改点

现有方案已经吸收了 PicoNewsAgent 的绝大多数核心经验，因此不重复堆叠同义组件。本次只增加一个此前缺失且与“后续持续增加网站”直接相关的能力：

**Source Intake / Bulk Import。**

具体需要在主方案中补充：

1. Site Onboarding 下增加批量来源导入；
2. 核心数据模型增加 `source_import_batch/source_import_item`；
3. Web 站点管理增加 URL/Feed/OPML/Markdown/CSV/JSON 导入、dry-run、冲突预览和审核；
4. API 增加 Source Import 创建、预览和 Apply；
5. Phase 1 即实现 Bulk Source Intake，避免 1000 站接入依赖手工逐条配置；
6. Web 验收标准增加“重复导入幂等、已有站点自动匹配、直接 Feed 能正确归属站点、导入历史可审计”。

其余已经进入主方案的设计保持不变：Feed Change-Signal、HTTP-first、长期 Browser Pool、Canonical IR、非破坏性相似聚类、Asset provenance、AI/Delivery 独立下游、Review State Machine、Outbox/Manifest/Finalizer 等。

## 18. 最终判断

PicoNewsAgent 适合作为“实时 Feed -> 深抓 -> AI -> 发布”参考实现，不适合作为 1000 站历史知识库底座直接扩容。

其核心价值在于把资源昂贵程度天然分层：

```text
Feed metadata 很便宜
< HTTP article fetch
< Browser render
< LLM processing
< 外部发布与媒体处理
```

生产知识库应该进一步把每一级都做成独立、持久、幂等、可观测、可重放的 Stage，并且让便宜信号尽量在昂贵 Stage 之前短路 unchanged 工作。

同时，PicoNewsAgent 用一个简单 Source 列表做到“加 URL 就能扩展来源”，提醒我们：可靠性和运营体验不能二选一。最终系统既要用数据库、Release、Audit 保证真相，又要让新增几十、几百个博客来源保持批量、低摩擦和可预览，这也是本次对现有技术方案最具体的新补强。