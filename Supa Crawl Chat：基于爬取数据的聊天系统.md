# Supa Crawl Chat：基于爬取数据的聊天系统

## 1. 调研对象

- 文章：Supa Crawl Chat
- 文章地址：https://bigsk1.com/posts/supa-crawl-chat/
- 项目地址：https://github.com/bigsk1/supa-crawl-chat
- 调研目标：评估其 Crawl4AI + PostgreSQL/Supabase + pgvector + FastAPI + Web UI 的实现方式，对“约 1000 个技术博客全量历史抓取、增量同步、Markdown 知识库、Web 管理、搜索/RAG”的技术方案有什么可吸收和需要规避的设计。

## 2. 项目定位与总体结构

Supa Crawl Chat 不是单纯爬虫，而是一套把“抓取 → 清洗 → 分块 → Embedding → PostgreSQL/pgvector → 搜索/聊天 → Web 管理”串起来的端到端应用。其主要模块可概括为：

```text
URL / Sitemap
  -> Crawl4AI API
  -> crawler.py 结果归一化与内容清洗
  -> 语义/Token 分块
  -> Embedding
  -> PostgreSQL/Supabase + pgvector
  -> FastAPI
  -> React/Vite Web UI
  -> Search / Chat

同时：
  -> crawl_jobs 记录抓取任务状态
  -> Streamlit Supabase Explorer 做数据库运营分析
  -> Docker Compose 提供 app-only / app+Crawl4AI / full-stack 部署
```

这套结构的价值主要不在“用 Crawl4AI 抓页面”本身，而在于它已经把抓取结果、父页面、Chunk、向量、站点、任务状态和 Web 操作连接成一个可以实际使用的产品闭环。

## 3. 抓取结果处理机制

### 3.1 对 Crawl4AI 不同返回形态做兼容

`crawler.py` 的 `process_crawl_results()` 没有假设 Crawl4AI 永远只返回一种字段，而是按优先级寻找正文：

```text
markdown.raw_markdown
 -> markdown.fit_markdown
 -> markdown string
 -> extracted_content
 -> cleaned_html
 -> html
```

同时兼容：

- `results[]` 形式；
- `pages{url: page}` 形式；
- 单个 `result` 形式。

这是很实用的“边界适配层”思想：第三方爬虫框架的返回结构会随版本变化或调用方式不同而变化，业务层不应该把某个 SDK 的单一返回字段当作永久契约。

但生产级知识库不能只靠“有字段就依次 fallback”。每个 Candidate 应保存来源、版本、质量分、字段 provenance，并经过结构化校验后再决定是否 Accepted，否则 fallback 可能把导航 HTML、挑战页或低质量内容当正文。

### 3.2 内容清洗先于入库和 Embedding

项目通过 `clean_crawled_content()` 清洗 Crawl4AI 返回内容，并在清洗结果为空时直接跳过索引；质量标记会被放进 metadata。

技术上这是正确顺序：

```text
Raw Crawl Result
 -> Content Hygiene
 -> Canonical Content Candidate
 -> Chunk
 -> Embedding
```

如果先 Embedding 再清洗，导航、Cookie Banner、页脚、重复菜单都会污染向量空间，并提高 Token 成本。

对于长期知识库，还应再加一层 Canonical Document IR，使 Markdown、Chunk、Embedding 都从 IR 投影生成，而不是让“某一次 Crawl4AI Markdown 输出”成为唯一事实源。

## 4. Sitemap 与 URL 发现

项目除了标准 Sitemap，还考虑了一个实际问题：用户可能把 `llms.txt`、Markdown URL 列表等“非 XML 文本”当作 sitemap 输入。因此 `_extract_urls_from_text_or_markdown()` 会从 Markdown 链接和裸 HTTP(S) URL 中提取链接。

这个能力值得吸收为 Discovery Provider 的一种：

```text
SITEMAP_XML
SITEMAP_INDEX
RSS_ATOM_JSONFEED
HTML_ARCHIVE
DOC_NAV
TEXT_URL_LIST / LLMS_TXT
API
```

但必须把“发现 URL”和“抓正文”分开。文本中出现 URL 只是 Discovery Evidence，后面仍要经过：

```text
normalize
 -> scope
 -> robots
 -> SSRF/redirect policy
 -> URL admission
 -> durable frontier
 -> origin fetch
```

不能把文本里提取到的 URL 直接当作可信文章对象。

## 5. 语义分块实现与原理

### 5.1 当前实现

`crawler.py` 的 `chunk_content()` 先统计整个正文 Token 数。如果未超过阈值，就保留为一个父页面；超过阈值后优先按照 Markdown 标题边界拆分，若没有标题则按段落拆分。

基本流程是：

```text
Markdown
 -> 按 # ~ ###### 标题形成 section
 -> 无有效 section 时按空行形成 paragraph sections
 -> 对每个 section 统计 token
 -> 组合到 max_tokens
 -> 单 section 仍超限时继续 token 级切分
 -> 相邻 chunk 保留 overlap
```

项目文档还明确说明 overlap 会优先寻找段落、句子、子句、单词等自然边界。

### 5.2 为什么这比固定字符切分好

技术博客的语义单位常常是：

- 一个章节；
- 一段解释 + 紧随其后的代码；
- 一张表；
- 一组 CLI 命令；
- 一个错误及解决步骤。

固定每 N 个字符切分会把这些结构截断，导致向量检索召回半段代码、没有标题的解释或缺上下文的参数说明。

因此生产方案应保留“结构优先、Token 限制兜底”的思想，但从 Canonical IR block 做分块，而不是重新解析已经扁平化的 Markdown。IR 可以准确知道 heading、paragraph、code、table、list 的边界，避免正则识别 Markdown 标题带来的误差。

### 5.3 当前 Chunk Identity 的不足

Supa Crawl Chat 会把 chunk URL 变成：

```text
<page-url>#chunk-0
<page-url>#chunk-1
...
```

并以 `chunk_index` 识别 Chunk。这个做法对单机 UI 很直观，但不适合长期版本化系统：上游内容稍有变化就可能改变所有后续 chunk index，从而让 `chunk-7` 指向完全不同的内容。

更稳妥的方式是：

```text
chunk_id = hash(
  document_version_id
  + chunker_release_id
  + block_range
  + chunk_hash
)
```

URL fragment 只作为展示信息，不承担全局 Chunk Identity。

## 6. PostgreSQL / Supabase 数据模型

项目核心表包括：

```text
crawl_sites
crawl_pages
crawl_jobs
chat_conversations
user_preferences
```

其中 `crawl_pages` 同时存父页面与 Chunk：

```text
site_id
url
title
content
summary
embedding
metadata
content_hash
is_chunk
chunk_index
parent_id
created_at
updated_at
```

pgvector 直接存储在同一张表，项目建立向量索引并提供相似度查询函数。

这种设计的优点是简单：一个页面及其 Chunk 可以直接在一张表里展示和搜索；Web UI 也容易实现“查看父页面 / 查看 Chunk”。

但面向百万级长期知识库，父文档、版本、Chunk Projection 和 Search Index 应拆开：

```text
document
document_version
chunk_projection
index_projection
```

否则父页面、Chunk、历史版本、不同 Chunker/Embedding 模型全混在同一表后，唯一性、版本迁移、索引蓝绿切换和 GC 都会变复杂。

## 7. 重复抓取与“更新已有页面”的实现

### 7.1 Supa Crawl Chat 的实现

`db_client.py` 的 `add_pages()` 会把输入拆成父页面和 Chunk。

父页面处理逻辑：

```text
SELECT id
FROM crawl_pages
WHERE url = ? AND is_chunk = false
```

如果已存在，则直接：

```text
UPDATE title/content/summary/embedding/metadata/content_hash
```

否则 INSERT。

对于 Chunk，同样按 URL + chunk_index 判断更新或新增。若 `replace_chunks=True`，会先删除父页面旧 Chunk，再插入当前抓取产生的新 Chunk。

因此项目的“增量”本质上是：

```text
同 URL 再抓
 -> 覆盖当前父页面
 -> 可删除并重建当前 Chunk
 -> updated_at 变化
```

### 7.2 这个模式值得借鉴的地方

它提供了非常好的“当前视图”体验：业务查询只看一行最新页面，不需要每次从版本表里找最新版本；Web 管理和 Search 都很简单。

### 7.3 不应直接复制的地方

对于知识库，原地 UPDATE 会丢失历史：

- 无法看某文章三个月前是什么内容；
- 抽取规则错误覆盖后难以恢复；
- 无法比较模板改版前后；
- 无法证明某次 RAG 答案使用的是哪个内容版本；
- Chunk 被 delete + recreate 后，搜索 lineage 断裂。

因此推荐吸收 Supa 的“当前视图简单”优点，但实现为 **append-only 版本 + Current Projection**：

```text
document_version            append-only 历史真相
        |
        +--> document_current_projection   每 document 一行当前 Accepted Version
        +--> chunk_projection              按 version + chunker_release 生成
        +--> search_current_projection     当前可检索版本指针
```

`document_current_projection` 可以像 Supa 的 `crawl_pages` 一样高效服务 Web 列表和详情，但它只是可重建读模型，不覆盖历史。

## 8. Current Projection 的推荐事务语义

这是本次调研对现有方案最值得补充的一点。

当新抓取内容通过质量门后：

```text
BEGIN
  1. 计算 canonical IR hash
  2. 若当前 document_version 已有相同 content_hash：只记录 freshness observation
  3. 若变化：INSERT 新 document_version
  4. INSERT quality/lineage/evidence
  5. UPSERT document_current_projection(document_id -> new_version_id)
  6. INSERT outbox events：publish/chunk/index/ai
COMMIT
```

注意 Search Current Pointer 不应立刻跟着第 5 步切换。新的 Document Version 要先完成 Chunk + Embedding + Index Manifest：

```text
new version accepted
 -> build chunks/index
 -> manifest complete
 -> atomic switch search_current_projection
 -> old index projection async GC
```

这样同时得到：

- Supa 式“当前页面一查就有”的管理体验；
- append-only 历史；
- 搜索无空窗；
- 可重放和可审计。

## 9. FastAPI Web 管理实现

项目的 API Router 按功能拆为：

```text
crawl.py
sites.py
pages.py
search.py
chat.py
```

Web UI 能完成：

- 新建抓取；
- 管理站点；
- 查看站点页面和 Chunk；
- 搜索；
- 查看原始内容 / Markdown；
- 对单页刷新；
- 查看抓取状态。

这种“站点 → 页面 → Chunk → 搜索 → 单页诊断”的信息架构非常适合知识库管理系统，可直接作为 Web IA 的参考。

项目还提供 Streamlit Supabase Explorer，可看统计、执行查询、画图和导出 CSV。对内部工具很高效，但生产环境不建议给普通管理用户任意 SQL。更稳妥的实现是：

- 预定义只读运营查询；
- Saved Query 模板；
- Read-only DB role；
- 查询超时与行数限制；
- 导出审计；
- 需要任意 SQL 的高级运维入口与普通业务 Web 权限隔离。

## 10. 抓取任务状态与 Durable Job 的差距

项目增加了 `crawl_jobs`：

```text
id
site_id
url
status
options
crawl4ai_task_id
pages_found
pages_crawled
chunks_created
error
started_at
finished_at
updated_at
```

这是从“请求触发爬虫”向“任务化”迈出的重要一步，至少让 Web 可以查询任务状态。

但 `api/routers/crawl.py` 仍使用 FastAPI `BackgroundTasks` 执行长期抓取。也就是说：

```text
HTTP request
 -> 创建/更新 crawl_jobs
 -> API 进程 BackgroundTasks 执行爬虫
```

如果 API 进程重启、Pod 被驱逐或部署滚动更新，数据库里虽然有 job 记录，但真正执行上下文并不是 durable worker lease，任务不能天然从另一个 worker 接管。

因此现有博客知识库方案中“DB 真相 + Outbox + Redis Streams + lease/heartbeat + worker”应保留，不应退回 FastAPI BackgroundTasks。Supa 的 `crawl_jobs` 表可以借鉴为用户可见的 Run/Job Read Model，但不作为实际调度器。

## 11. 安全设计

项目目前已经加入多项值得保留的安全措施：

- URL 必须为 HTTP/HTTPS；
- 抓取 URL 经过 SSRF 校验；
- 外链跟随默认关闭；
- redirect、custom proxy、文件下载等高风险选项需要显式环境开关；
- Web/API 可配置鉴权；
- 关键管理操作写审计日志。

这说明 Web 爬虫管理系统不能把“URL 输入框”理解为普通业务字段，而应视为网络访问能力入口。

在 1000 站生产环境还要加强：DNS rebinding、每次 redirect 重新做目标地址校验、私网/metadata endpoint 封禁、端口 allowlist、Asset URL 同策略、租户级 egress policy。

## 12. 搜索与 Embedding

项目使用 PostgreSQL + pgvector，父页面和 Chunk 都可拥有 embedding，并提供向量相似度搜索，同时还有文本搜索和 Hybrid Search 思路。

对技术博客而言 Hybrid Search 是必要的，因为很多查询包含：

```text
函数名
类名
错误码
CLI 参数
版本号
文件路径
配置项
```

这些 token 使用纯向量检索未必稳定，必须保留 lexical/FTS/BM25 路径，再用 RRF/加权融合。

本项目“Chunk 保留 parent reference”的做法应保留，并进一步扩展为：

```text
chunk
 -> document_version
 -> document
 -> source snapshot
 -> source URL
 -> heading/block range
```

这样 RAG 引用能回到真实原文版本。

## 13. AI 内容增强的边界

Supa Crawl Chat 会为页面生成标题、摘要、站点描述，并支持聊天长期记忆。这对产品体验有帮助，但对于知识库同步主链路，LLM 派生内容不能成为抓取成功的前置条件。

正确边界是：

```text
Accepted Canonical Document Version
 -> Async Summary / Tags / Entity / Embedding / Digest / Chat Index
```

OpenAI 或其他模型故障时：

- 抓取仍能继续；
- Markdown 仍能发布；
- FTS 仍能工作；
- 后续可补跑 AI Projection。

因此现有方案“Derived AI 与抓取解耦”是比该项目更适合长期运行的设计，应保持。

## 14. Docker 部署模式的启发

项目提供三类部署：

```text
轻量：App only
标准：App + Crawl4AI
完整：App + Crawl4AI + Supabase
```

这个分层非常适合开发与交付：

- 本地开发者可以一条 compose 启动全栈；
- 已有 PostgreSQL/Crawl4AI 的环境只部署 App；
- CI 可以启动最小依赖跑集成测试。

生产环境不必照搬单个 Compose，而应把这个思想变成 Deployment Profile：

```text
DEV_FULLSTACK
DEV_EXTERNAL_DB
STAGING
PROD_K8S
```

每个 Profile 固定组件版本、资源限制、Secret Binding 和网络策略，避免文档里的环境变量组合在不同环境中产生隐式差异。

## 15. 对现有博客知识库方案的具体优化结论

### 15.1 应新增：Current Document Projection

现有方案已经坚持 `document_version append-only`，但应再明确一层可重建的 Current Read Model，以获得 Supa Crawl Chat 的简单查询体验：

```text
document_current_projection
- document_id PK
- current_version_id
- site_id
- canonical_url
- title
- published_at
- updated_at
- content_hash
- quality_state
- markdown_artifact_id
- accepted_at
- projection_updated_at
```

用途：

- Web 站点/文章列表；
- 当前文章详情；
- 增量同步快速比对；
- 运营统计；
- Markdown 发布定位。

它不是历史真相，删除后可从 `document + document_version` 重建。

### 15.2 应新增：Search Current Projection

不要让“当前 Accepted 文档版本”和“当前可检索版本”共用一个指针：

```text
search_current_projection
- document_id
- search_index_release_id
- document_version_id
- index_projection_manifest_id
- switched_at
```

只有新版本完整索引成功才切换，可避免文档刚更新时 Search 短暂无结果。

### 15.3 Web 管理增加“当前视图 + 历史视图”双层模型

站点文章页默认看 current projection，详情页可以切到：

```text
Current
History
Version Diff
Raw Snapshot
Candidate
Canonical IR
Markdown
Chunks
Search Trace
```

这样日常运营不被复杂 lineage 淹没，排障时又能下钻。

### 15.4 增加只读运营 Explorer

吸收 Streamlit Explorer 的便利性，但生产版采用受控查询：

- 预定义站点覆盖、freshness、失败率、Chunk/Index 完成度等查询；
- Saved Query；
- CSV 导出；
- Read-only role；
- 超时、最大行数、审计；
- 任意 SQL 仅限高级管理员。

### 15.5 增加 Deployment Profiles

将 app-only / integrated crawler / full-stack 思想固化为版本化 Deployment Profile，用于本地、测试、生产的一致部署。

## 16. 不应照搬的设计

1. **不能用 URL 原地 UPDATE 代替文档版本。** 当前行易用，但历史和审计必须 append-only。
2. **不能用 `#chunk-N` 作为稳定 Chunk Identity。** Chunk Index 会因内容变化漂移。
3. **不能用 FastAPI BackgroundTasks 承担长期抓取。** Job 状态持久化不等于执行持久化。
4. **不能把父页面和所有版本 Chunk 永久混在同一表。** 小项目方便，大规模版本化后维护成本高。
5. **不能让 LLM 标题/摘要生成阻塞抓取成功。** AI 是异步 Projection。
6. **不能把“抓完 sitemap 的 max_urls 条”当历史完整。** 必须有 Coverage Evidence 和 terminal reason。
7. **生产 Web 不应默认开放任意 SQL。** Explorer 必须权限和资源隔离。

## 17. 最终评价

Supa Crawl Chat 最值得借鉴的是“把爬虫做成产品”的思路，而不是某个具体库：它用 PostgreSQL/Supabase 维护站点和页面，用父页面/Chunk 关系支撑搜索，用 FastAPI + Web UI 让用户可以管理站点、查看页面、刷新单页、搜索和诊断，再用 Docker 把整套环境快速交付。

对于 1000 个技术博客知识库，它的简单 URL 更新模型不足以承担多年版本历史，但恰好提示了现有方案一个重要补充：**在 append-only 真相层上增加可重建的 Current Projection / Search Current Projection**。这样可以同时拥有生产级审计能力和日常 Web 管理所需的简单、快速“当前状态”查询。

因此本次调研结论是：保留现有 Durable Frontier、Immutable Snapshot、Canonical IR、append-only Version、Outbox、版本化 Search/RAG 的主架构；新增 Current Read Model、Search Current Pointer、受控运营 Explorer 和 Deployment Profile，并继续明确禁止 BackgroundTasks 承担 durable crawl。