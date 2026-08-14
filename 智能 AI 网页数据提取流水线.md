# 智能 AI 网页数据提取流水线：实现细节与技术原理分析

调研对象：https://github.com/ShyamThangaraj/Intelligent-AI-Web-Extraction-Pipeline

代码快照：`main` 分支，调研时 HEAD 为 `a77e3adeb9325a3789ab7ffaf1a27cbb969c8e14`。

## 1. 项目定位

该项目是一个“陌生站点结构化数据抽取”原型，目标不是长期运行的千站博客知识库，而是通过少量站点探测、LLM 选择入口、LLM 生成/修复抽取 Schema、确定性 selector 执行、LLM 再校验结果的方式，降低针对不同网站手写规则的成本。

它最有价值的思想是把 AI 放在“生成声明式知识”的位置，而不是让 AI 直接成为最终内容解析器：

```text
网页/候选入口
 -> AI 选择或生成规则
 -> 版本化 Schema
 -> 确定性 CSS/XPath/JSON-LD/Meta 执行
 -> 结果校验
```

对于 1000 个技术博客的知识库，这种思路可以吸收，但必须从“在线自动修规则”升级为受控的 `Extraction Contract candidate -> shadow replay -> quality gate -> release`，并把 LLM 从热路径中移走。

## 2. 主流程和模块职责

项目有两条入口：

- `main.py`：SERP 辅助发现入口。
- `main_norm.py`：直接抓站点首页链接，再用 LLM 选择博客/职位列表页。

核心模块：

- `modules/link_extracter_serp.py`：搜索结果驱动的列表页入口发现。
- `modules/link_extractor_norm.py`：首页链接收集、启发式过滤、LLM 选择、候选页面验证。
- `modules/schema_generator.py`：按域名和类型持久化版本化抽取 Schema，支持 LLM 生成和修复规则。
- `modules/data_extractor.py`：列表页翻页、文章 URL 发现、HTML 抓取、JSON-LD/Meta/CSS/XPath 抽取。
- `modules/data_validator.py`：用 LLM 判断提取出的标题是否像真正的博客或职位标题。
- `modules/database.py`：SQLAlchemy + SQLite 保存列表页运行结果和抽取数据。

总体执行链路可概括为：

```text
Domain
 -> Listing Source Discovery
 -> Schema Ensure/Generate
 -> Listing Pagination
 -> Item URL Discovery
 -> Fetch HTML
 -> JSON-LD + Meta + Schema Rules
 -> Merge Record
 -> LLM Title Validation
 -> SQLite
```

这个分层本身是正确方向，但生产知识库应进一步把 Source Discovery、URL Discovery、Fetch、Capture、Extract、Quality Gate、Publish 拆成独立状态机和 worker。

## 3. 列表页和 Source Discovery 的实现

### 3.1 首页候选收集

`link_extractor_norm.py` 先抓取首页的所有 `<a href>`，把相对链接转换为绝对链接，同时保留外链。然后用大量 URL 正则过滤看起来像“单篇文章/单个职位”的 URL，目标是把候选集收缩到列表页。

随后还会主动补充常见路径，例如：

- `/careers`
- `/jobs`
- `/blog`
- `/news`
- `/insights`
- `/articles`

再把整个候选 URL 列表发给 LLM，让模型必须各选择一个职位列表页和博客列表页。

这里有一个值得借鉴的模式：**先由廉价确定性逻辑做高召回候选生成，再让更昂贵的语义模型做候选排序/分类**。但原实现要求模型“不能返回 NONE、必须选一个”，在生产系统里是不合理的；没有可靠候选时应该显式返回 `unknown`，而不是强迫选错。

另外，程序虽然在 prompt 中要求模型“只能从给定 URL 中选择、不能发明 URL”，但候选列表本身包含程序自动拼出的 `/blog`、`/careers` 等未必真实存在的路径。因此真正安全的生产设计应该区分：

```text
observed_candidate   # 页面、robots、Sitemap、Feed、redirect、API 实际观察到
heuristic_candidate  # 程序猜测的常见路径
verified_source      # 实际抓取并通过 source/page-type 验证
```

只有 `verified_source` 才可进入 URL Discovery。

### 3.2 候选页面验证

项目会继续抓取候选页，尝试判断页面是否包含多个同类条目，并在一定深度内寻找更具体的列表入口。这比“LLM 直接选完就信任”更可靠，体现了一个重要原则：**AI 只给候选，最终接受必须由可验证信号完成**。

对于知识库，可以把该原则落实为：Source candidate 必须经过 HTTP status、content-type、page type、文章链接密度、模板重复性、host/scope、robots、Egress 和抽样 URL 可提取性验证。

### 3.3 SERP 的正确定位

项目还允许用搜索引擎找到站点的博客/职位入口。这个能力对“首页没有直接暴露 archive/blog 入口”的站点有帮助，但不应把搜索结果当历史全量来源。

适合知识库的定位是低频 `SearchIndexSourceDiscoveryProvider`：

- 只在新站 probing、长期发现 0 篇、coverage 异常或人工诊断时运行；
- 目标是发现博客首页、archive、年份页、Feed/Sitemap 或迁移域；
- 搜索结果只是 discovery evidence；
- 结果 URL 必须重新经过 scope、approved host、Egress、robots 和 HTTP 验证；
- 不能用搜索结果数证明“历史全量完成”。

## 4. URL Discovery 和翻页

`data_extractor.py` 使用标准库 `HTMLParser` 收集 `<a>` 的 `href/text/rel/class/id`，再通过路径启发式判断文章 URL。博客常见模式包括：

- `/blog/...`
- `/blogs/...`
- `/article...`
- `/post/`、`/posts/`
- `/insight/`、`/news/`、`/story/`
- `YYYY/MM/...`
- `.html`

导航、菜单、footer、登录、注册、社交等链接按 class/id/text 关键词排除。

翻页按以下顺序寻找：

1. `rel=next`；
2. `next`、`older`、`load more` 等锚文本；
3. `page=` / `p=` 参数。

只有发现新 URL 或第一次尝试下一页时才继续。

这套逻辑适合 probing，但默认 `max_pages=2`，无法承担全量历史回灌。生产系统必须把终止原因持久化，例如：

```text
end_reached
known_streak
no_new_urls
repeated_cursor
scroll_limit
budget_exhausted
selector_drift
coverage_unknown
```

否则“循环停了”不能等价为“历史抓完了”。

## 5. Schema 生成、版本化和修复

### 5.1 Schema 文件模型

`schema_generator.py` 按域名和类型保存：

```text
schemas/{domain}/{domain}.{type}.vN.json
```

Schema 包含：

- domain
- type
- version
- created_at / updated_at
- listing_url
- required_fields
- extraction_rules

并实现最新版本查找、保存、版本递增、旧目录迁移等逻辑。

值得保留的不是本地 JSON 文件本身，而是“抽取规则是可版本化 artifact”。在千站系统里应升级为正式的 Extraction Contract 数据模型，记录 source snapshot、生成器版本、prompt/model（如使用 LLM）、contract hash、测试指标、批准人和 release 状态。

### 5.2 初始 LLM Schema 的真实能力

`generate_schema_with_llm()` 会让模型返回严格 JSON，并为字段生成 CSS/XPath selector。但是当前初始调用并没有把真实页面 HTML/DOM 传给模型，主要输入只有：domain、scrape type、listing URL、required fields 以及一些通用 selector 提示。

因此这一步更像“生成一个通用候选模板”，并不是真正意义上的站点 DOM 反向推导。模型很容易产生语法合理但页面根本不存在的 selector。

生产 Rule Compiler 应改为：

```text
representative immutable snapshots
 -> page-type / DOM fingerprint clustering
 -> sanitized DOM structural summary
 -> candidate contract generation
 -> deterministic execute on golden snapshots
 -> required-field coverage
 -> body coverage / boilerplate / structure metrics
 -> shadow diff
 -> canary
 -> active release
```

### 5.3 默认规则

项目内置了 blog 和 Greenhouse 的默认规则。Blog 规则依次尝试 `article h1`、`h1`、OpenGraph title；author/date/tags 走 CSS 或 meta；正文尝试 `article`、`.node__content`、`.content` 等。

这说明一个可扩展系统不应为每站写代码，而应维护：

```text
Generic Profile
 -> Platform Profile
 -> Site Override
 -> Extraction Contract
 -> Custom Adapter
```

平台级规则可以复用，站点只声明差异。

## 6. 抓取层实现

### 6.1 Crawl4AI 主要被当作 Fetcher

在已检查的主链路里，`data_extractor.py` 首先调用 `AsyncWebCrawler.arun(url=...)` 获取 HTML，然后再由本地 BeautifulSoup/lxml/JSON-LD 代码执行字段抽取。README 虽然描述了 Crawl4AI + JsonCss 的结构化抽取，但当前主链路中更明显的事实是：**Crawl4AI 主要承担页面获取，真正的字段融合仍由项目自己的规则引擎完成**。

如果 Crawl4AI 没拿到 HTML，则退回 `urllib.request.urlopen`。

这种 Adapter 思想可保留，但实现存在明显规模问题：

- 每个 URL 都创建一次 `AsyncWebCrawler` 上下文；
- 同步函数里每次 `asyncio.run()`；
- 浏览器/事件循环不能跨 URL 复用；
- `CrawlerSettings.max_concurrency` 在这条路径中没有形成真正并发控制；
- `respect_robots` 只是设置字段，没有形成完整 enforcement；
- `polite_sleep()` 使用阻塞 `time.sleep()`；
- urllib fallback 使用一次性 `resp.read()`，没有 streaming bytes/解压预算。

生产系统应使用 HTTP worker pool + Browser process pool，Browser 按原因升级，并统一通过分布式域名限流器和 EgressPolicy。

### 6.2 原始输入 provenance 问题

Crawl4AI 结果若没有 HTML，代码会把 Markdown 包装成 `<html><body><pre>...</pre></body></html>` 再交给后续 HTML 解析。

这会丢失输入语义：后续系统无法区分“原始服务器 HTML”“Browser rendered DOM”“第三方清洗后的 Markdown”。生产方案应把这些保存为不同 artifact，并在 `extraction_attempt.input_kind` 中明确记录，不能互相冒充。

### 6.3 本地 URL cache

项目把 URL 的 SHA-1 作为本地 HTML cache 文件名。它能减少重复请求，但没有 ETag、Last-Modified、snapshot provenance、capture profile、fetcher version 和跨 worker 一致性。

因此 cache 只能是可丢弃优化，不能成为业务真相。业务状态应在 PostgreSQL，不可变原始内容放 S3/MinIO。

## 7. 抽取策略和字段融合

项目并不是单一 selector 路径，而是多源候选融合：

1. JSON-LD：识别 `BlogPosting`、`Article`、`NewsArticle`；
2. Meta/OpenGraph；
3. Schema 中声明的 CSS/XPath/regex/meta 规则；
4. 最后按规则 > JSON-LD > meta 的顺序合并。

Blog JSON-LD 可读取 headline/name、author、datePublished、keywords、articleBody/image；Schema rules 则支持：

```text
css
xpath
meta
regex
many=true
attr=href/src/content
```

这非常适合演化为生产的 Metadata Candidate Resolver，但生产系统不应只靠固定覆盖顺序。每个候选值应该带：

```text
field
value
source_type
source_location
confidence
rule_version
snapshot_id
raw_or_normalized
```

再由 resolver 根据站点规则、来源可信度和历史稳定性选择最终字段。

## 8. 代码级正确性问题

这一部分对方案设计很重要，因为它说明“看起来存在的自修复/校验能力”在原型代码里并不等于真正可靠。

### 8.1 Schema repair 主链路实际不可达

`extract_single_item()` 会计算：

```python
missing = validate_required(core, scrape_type)
```

但随后直接返回：

```python
return core, None
```

即使有 required fields 缺失，也不会把 failure context 返回上层。

而 `ensure_extraction_for_type()` 的 repair 分支要求：

```text
rec 为空
且 fail 非空
且 LLM 可用
```

由于 `build_record()` 总会补 `source_url` 和 `schema_version`，`core` 通常本身就是真值；同时 `fail` 被固定返回 `None`。因此设计文档里的“抽取失败 -> LLM 修 Schema -> retry”在当前主路径上实际上基本不会触发。

此外，`data_extractor.py` 顶部导入列表中没有导入 repair 分支所调用的 `attempt_schema_repair_with_llm`、`bump_schema_version`、`save_schema`。即便未来让 failure path 可达，这一分支也需要先修正依赖导入。

这正好说明生产系统不能把“在线自修复”当隐式副作用；应该将 drift evidence、candidate generation、validation、release 全部显式建模和测试。

### 8.2 discovered 统计字段不一致

`discover_listing_items()` 返回的结构是：

```text
{items: [...], pages_visited: N}
```

但 `main_norm.py` 写 Listing 运行结果时读取了类似：

```text
item_discovery[...].get("discovered", 0)
```

因此数据库里的 `items_discovered` 可能长期为 0，即使实际发现了 URL。

生产系统应统一事件/数据契约，并用 schema validation/typed model 保证 worker 之间的字段名一致。

### 8.3 Schema 版本记录可能失真

`main_norm.py` 落 listing 运行记录时把 `schema_version` 写成固定 `v1`，而真实 Schema 已支持 `vN` 递增。这会让“运行时使用哪个规则版本”与数据库记录不一致。

生产系统必须让每个 extraction attempt 直接引用不可变 `extraction_contract_release_id` 和 artifact digest，而不是写一个易漂移的字符串。

## 9. LLM 校验的实现和风险

`data_validator.py` 会把最多 20 个标题交给模型，要求输出二进制字符串表示是否为有效职位/博客标题。

关键行为：

- 一批最多验证前 20 个；
- 超过 20 个的剩余标题直接补 `False`；
- API 异常时整批返回 `False`；
- `main_norm.py` 只把校验为真的数据写入数据库。

对小样本 demo 这是保守策略，但对全量知识库会造成系统性误拒绝：LLM/provider 故障会被错误解释为“内容无效”。

正确设计应是：

```text
valid
invalid
unknown
```

LLM 只提供 `quality_signal`；不可用时必须是 `unknown`，canonical pipeline 继续运行。真正的质量门禁应组合 page type、selector hit、正文长度、段落/代码/表格结构、重复率、boilerplate、JSON-LD 类型、template fingerprint 和历史 golden regression。

## 10. 数据库和“增量”的真实语义

项目用 SQLite + SQLAlchemy 保存：

- listing_table
- extracted_data

所谓增量主要通过 `get_existing_urls()` 读取已有 `source_url`，新一轮发现时把这些 URL 从待抓列表里过滤掉。

这只能实现“URL 去重”，不能实现内容增量同步。它无法发现：

- 原 URL 正文被修改；
- 标题/作者/发布日期被修正；
- canonical 变化；
- 原文章删除或迁移；
- 同 URL 页面模板改变导致旧内容抽取错误；
- 规则/Serializer 新版本需要离线重放。

真正的增量需要至少保存：

```text
ETag
Last-Modified
HTTP raw body hash
content hash
Markdown hash
last_source_seen_at
snapshot id
extractor/contract/serializer version
```

并结合 Feed/Sitemap/archive checkpoint、304、低频 reconciliation 和删除确认状态机。

数据库层还只有对 `source_url` 的普通索引，没有数据库唯一约束；`data_already_exists()` 是“先查再插”的应用层去重，在并发 worker 下存在 race condition。生产系统必须依赖数据库唯一约束 + UPSERT 保证幂等。

## 11. 安全问题

项目更适合作为本地原型，不能直接放到拥有内网权限的生产 Worker 中。

### 11.1 TLS 校验被全局关闭

`link_extractor_norm.py` 把 Requests Session 设置为：

```text
verify = False
```

并关闭 urllib3 SSL warning。生产系统绝不能这样做，否则 HTTPS 身份验证失效。

### 11.2 外链和 SSRF 边界不足

候选发现允许外部链接；Fetch/redirect 没有统一逐跳 Egress 校验。输入站点可能通过链接或 redirect 把 worker 引向 localhost、RFC1918、cloud metadata 或其他内部服务。

因此所有 URL 来源——首页链接、Sitemap、Feed、canonical、`<base href>`、redirect、媒体、搜索索引和 Browser 子资源——都必须经过统一 `EgressPolicy`。

### 11.3 LLM 输入需要安全裁剪

Schema repair 会把最多约 8000 字符 HTML 发送给模型。网页内容本身是不可信输入，可能携带 prompt injection。生产 Rule Compiler 应只发送经过清洗的 DOM 结构摘要/字段上下文，明确隔离网页指令和系统指令，并记录输入 hash、prompt version、model/provider。

## 12. 规模与运维能力评估

不适合直接支撑“1000 站全量历史 + 长期增量”的地方：

1. 默认列表翻页 `max_pages=2`。
2. 默认单类型抽取 `max_items=10`。
3. SQLite 单机状态库。
4. URL 本地文件 cache。
5. 每 URL 新建 Crawl4AI 上下文并 `asyncio.run()`。
6. 阻塞 sleep。
7. 没有 durable queue、lease、outbox、backpressure、fair scheduling。
8. 没有 Web 管理面。
9. 没有 Markdown AST/Serializer 和确定性 md 输出。
10. 没有不可变 raw snapshot / rendered DOM provenance。
11. 没有媒体下载、去重、附件治理。
12. 没有完整的删除/迁移状态机。
13. 增量只是 URL skip。
14. LLM 是入口发现和质量校验的强依赖。

因此它应被视为“规则生成与多源抽取模式的参考实现”，而不是现成的知识库平台。

## 13. 对博客知识库方案真正值得吸收的能力

### 13.1 Rule Compiler / Schema Synthesis

最值得吸收：让 LLM、模板、统计推断或人工编辑生成声明式 `Extraction Contract candidate`，但生产只执行已批准 release。

```text
snapshots
 -> cluster/page type
 -> candidate contract
 -> deterministic replay
 -> quality metrics
 -> shadow diff
 -> canary
 -> active
```

模板漂移时也走同一个 candidate/release 流程，不能由单页失败直接在线修改 active 规则。

### 13.2 多源 Metadata Candidate

保留 JSON-LD、Meta、CSS/XPath、通用正文抽取器等多候选来源，但为每个候选保留来源和置信度，交给统一 resolver。

### 13.3 Search Index 只辅助 Source Discovery

搜索引擎适合找“博客入口/归档入口”，不适合证明历史全量。应作为低频 Provider，并有费用预算、provider version 和独立 evidence。

### 13.4 AI 可失败，确定性热路径不可依赖 AI

新站 probing 或模板漂移时允许 AI 帮助生成候选；正常日常同步应只运行已发布规则。这样 AI 成本从“每篇文章一次”降到“每个模板变化偶发一次”。

## 14. 与当前完整技术方案的关系

当前 `博客知识库技术方案.md` 已经具备并正确吸收了本项目最有价值的能力：

- Schema Synthesis / Rule Compiler；
- Extraction Contract candidate/release；
- golden snapshot、shadow replay、canary、rollback；
- Search Index Source Discovery Provider；
- JSON-LD/Meta/selector 多候选 resolver；
- LLM 只作为 `quality_signal`，故障为 `unknown`；
- HTTP raw bytes 与 Browser rendered DOM 分离；
- PostgreSQL 真相源、对象存储 artifact、Redis Streams 协调；
- Egress/SSRF/robots/预算；
- ETag/Last-Modified/body hash/Markdown hash 增量；
- Browser process pool；
- deterministic Markdown Serializer。

因此不应再把项目中的单机 SQLite、本地 cache、逐 URL crawler、URL-only skip、强制 LLM 选入口、LLM 在线修 selector 或 TLS `verify=False` 引入方案。当前方案在架构层已经比原项目更适合长期千站运行。

## 15. 结论

该项目最值得保留的技术原理可以概括为：

> **AI 负责生成/辅助发现声明式知识，确定性引擎负责执行，版本化规则负责复用。**

真正落地到博客知识库时，需要把它从“脚本中的 LLM 自动修复”提升为控制面的 Rule Compiler，把搜索引擎降级为 Source Discovery 辅助，把所有抓取输入保存为可重放 artifact，并用严格的状态机、质量门禁、安全边界和增量版本模型来支撑 1000+ 网站长期运行。