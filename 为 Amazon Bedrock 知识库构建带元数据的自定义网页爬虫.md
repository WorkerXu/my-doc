# 为 Amazon Bedrock 知识库构建带元数据的自定义网页爬虫

## 1. 调研对象与结论

- 编号：43
- 文章：Custom web scraper for Amazon Bedrock KB with metadata
- 中文名：为 Amazon Bedrock 知识库构建带元数据的自定义网页爬虫
- 地址：https://medium.com/@gargasm/custom-web-scraper-for-amazon-bedrock-kb-with-metadata-b3c7942e0ff2
- 作者：Magdalena Gargas
- 发布日期：2025-02-24
- 调研日期：2026-08-14
- 相关组件：Crawl4AI、Amazon S3、Amazon Bedrock Knowledge Bases

这篇文章实现的是一条非常直接的 RAG 数据准备链路：

```text
URL 列表
 -> Crawl4AI AsyncWebCrawler
 -> 每个 URL 提取 title + fit_markdown
 -> 正文文件
 -> 同名 .metadata.json 旁车文件
 -> 上传 S3
 -> Amazon Bedrock Knowledge Base 摄取
 -> 检索结果携带原始 URL metadata
```

文章的代码量不大，但对本项目有一个很值得吸收的设计点：**canonical Markdown 发布完成之后，应当存在独立的“知识库发布 / Knowledge Sink”层，把正文与可过滤、可追溯的 metadata 一起投递到外部知识库，而不是让抓取器直接绑定某一种 RAG 后端。**

对当前约 1000 个技术博客的方案而言，文章本身不能直接作为生产架构，因为它没有解决大规模 URL Discovery、全量历史证明、增量 checkpoint、幂等、版本、并发公平、限流、SSRF、删除同步、失败补偿、规则漂移等问题。但它暴露出当前主方案还可以进一步加强的一块：**把“Markdown 导出”升级成版本化的 Publish / Knowledge Sink Plane，并把 provenance metadata 当作正式数据契约，而不是导出时临时拼接的附属 JSON。**

本次建议保持现有 `PostgreSQL + Redis Streams + S3/MinIO + immutable snapshot + article_version + Search/Index Plane` 主链路不变，在 `article_version` 发布之后增加：

```text
article_version
 -> Publish Event
 -> Export Bundle Builder
 -> Target Adapter
    -> local Markdown
    -> generic S3
    -> Amazon Bedrock S3 data source
    -> 其它 RAG/搜索后端
```

外部知识库失败不得回滚 canonical Markdown，也不得触发源站重新抓取。

## 2. 文章实现拆解

### 2.1 Crawl4AI 抓取

文章使用 `AsyncWebCrawler`，对 URL 逐个执行：

```python
async with AsyncWebCrawler() as crawler:
    result = await crawler.arun(url=url, config=run_config)
```

成功后提取：

```text
url      = result.url
title    = result.markdown_v2.raw_markdown 第一行
content  = result.markdown_v2.fit_markdown
```

随后把 URL 中的协议、点、斜杠等字符替换成文件名，分别写正文文件和 metadata 文件。

这种实现适合教程或几十个固定 URL 的一次性导入，但对于长期知识库存在几个根本限制：

1. URL 列表是输入，不解决 URL 如何完整发现。
2. 文件名来自字符串替换，不能作为稳定 identity。
3. 抓取结果直接进入最终知识库，没有 immutable raw snapshot。
4. 没有 `ETag`、`Last-Modified`、304 或 raw hash 增量判断。
5. `fit_markdown` 是面向检索的裁剪结果，不应直接被默认视为“原文 canonical 归档”。
6. 标题取 Markdown 第一行属于弱启发式，无法处理页面模板、无 H1、错误 H1 或内容过滤结果变化。
7. 没有抓取版本、抽取器版本、规则版本和 serializer 版本，因此后续无法解释差异。

因此本方案仍应坚持：

```text
Discovery
 -> Frontier
 -> Fetch raw bytes / Browser DOM
 -> immutable snapshot
 -> Extract Candidate
 -> Quality Gate
 -> Article IR
 -> deterministic Markdown
 -> article_version
```

Crawl4AI 是 Fetch/Browser/Extractor Adapter，不是业务状态真相源。

### 2.2 `fit_markdown` 的角色

文章直接把 `fit_markdown` 存入知识库，核心目标是减少导航、模板和低价值内容，让向量检索更聚焦。

这个思路适合“派生检索视图”，但对于本项目的长期知识资产，应该把数据分成三层：

```text
Layer 1: Raw Evidence
HTTP 原始 response bytes / Browser rendered DOM

Layer 2: Canonical Content
Article IR -> deterministic Markdown

Layer 3: Retrieval/Export View
chunk / fitted text / embedding input / target-specific document
```

原因是：检索算法变化很快，而 canonical 正文需要长期稳定。如果把某个时间点某个版本的 `fit_markdown` 当作唯一正文，以后更换过滤算法时无法可靠重建。

所以 Crawl4AI 的 fitted Markdown 可以作为：

- 一个 extractor candidate；
- RAG target 的派生 document view；
- Quality Gate 的辅助候选；

但不能替代 raw snapshot 和平台确定性 Markdown。

## 3. Metadata 旁车文件的技术原理

文章最关键的部分是为每个正文文件同时生成：

```text
<document>.txt
<document>.txt.metadata.json
```

文章中的最小格式类似：

```json
{
  "metadataAttributes": {
    "url": "https://example.com/article"
  }
}
```

Amazon Bedrock 的 S3 数据源支持这种每文档 metadata sidecar。当前 AWS 文档还支持更丰富的 typed metadata，metadata 可用于检索过滤；metadata 文件需要与对应文档建立确定的命名/位置关系，并受大小限制。S3 数据源本身支持新增、更新和删除内容的增量同步。

这个机制的本质不是“多写一个 JSON 文件”，而是：

> **把正文内容与检索/治理所需的文档级属性解耦，但又通过稳定 identity 绑定成一个发布单元。**

这对 1000 站点知识库非常重要，因为检索层至少需要保留：

```text
article_id
article_version_id
site_id
source_url
canonical_url
title
author
published_at
updated_at
tags
content_sha256
source_snapshot_id
rule/profile/contract/serializer version
```

这些 metadata 不应该只存在于 Markdown Front Matter，也不应该只存在于向量数据库。它们应来自 `article_version.metadata_json + provenance_json` 的正式 schema，然后由 target adapter 转换成目标系统格式。

## 4. 为什么不能直接照搬“正文文件 + metadata 文件”

### 4.1 两文件一致性窗口

若业务代码先上传正文，再上传 metadata：

```text
PUT document.md 成功
PUT document.md.metadata.json 失败
```

外部知识库可能看到不完整发布单元。

反过来也一样。

因此生产级发布不能把“两个 PUT 成功”当作事务。建议采用 generation/staging：

```text
exports/{target_id}/{generation_id}/staging/...
 -> 上传全部正文和 metadata
 -> 生成 manifest
 -> 校验 count/hash
 -> 标记 generation ready
 -> 触发 target sync
 -> sync 成功后 active
```

如果目标后端不能原子切换 generation，则平台至少要把“目标端状态”与 canonical 状态分离，并保存每个 document 的 publish state。

### 4.2 URL 派生文件名不稳定

文章把 URL 字符替换后当文件名，存在：

- 不同 URL 归一化后冲突；
- query 参数导致超长文件名；
- canonical URL 变化导致路径变化；
- Windows/对象存储命名边界；
- tracking 参数污染；
- Unicode/percent encoding 不一致；
- redirect 后同文章产生多个对象。

应改为业务 identity：

```text
knowledge/{site_id}/{article_id}/{article_version_id}/document.md
knowledge/{site_id}/{article_id}/{article_version_id}/document.md.metadata.json
```

人类可读 slug 只能作为展示属性，不能参与唯一性。

### 4.3 metadata schema 演进

如果 sidecar 是临时字典，后续添加字段会出现：

- 不同年代文档字段形态不同；
- 下游 filter 类型不一致；
- 同字段有时 string、有时 number/list；
- metadata 大小超过目标后端限制；
- 重建时不知道使用哪个 schema。

因此需要：

```text
metadata_schema_version
export_adapter_version
export_profile_version
```

核心 metadata 使用平台内部统一 schema；Bedrock、OpenSearch、Qdrant 等只做 adapter mapping。

## 5. 建议新增 Publish / Knowledge Sink Plane

### 5.1 设计原则

1. 只消费已发布 `article_version`。
2. canonical 内容与外部知识库生命周期分离。
3. target adapter 只负责格式和 API，不拥有业务真相。
4. 导出失败可从 PG/S3 replay，不 refetch 源站。
5. 每个 target 有独立 generation、checkpoint、重试和限流。
6. metadata/provenance 是一等公民。
7. 删除、合并、重命名必须同步成显式 target mutation，不能只追加。
8. 对象 key 基于 ID/version，不基于原始 URL 字符串。
9. secret 只存 secret reference，不进入 metadata、日志或 manifest。
10. 外部后端 ingest 成功不等价于检索可见，必须分别记录 upload/sync/index/verify 状态。

### 5.2 推荐数据模型

```text
knowledge_targets
- id
- name
- kind(local_md/s3_generic/bedrock_s3/custom)
- status
- config_json_sanitized
- secret_ref
- metadata_schema_version
- adapter_version
- active_generation_id
- created_at / updated_at

publish_generations
- id
- target_id
- status(building/ready/syncing/active/failed/retired)
- based_on_article_watermark
- manifest_object_key
- document_count
- total_bytes
- config_hash
- adapter_version
- created_at / activated_at

publish_documents
- id
- target_id
- generation_id
- article_id
- article_version_id
- content_object_key
- metadata_object_key
- content_sha256
- metadata_sha256
- status(pending/staged/synced/verified/failed/deleted)
- target_document_id nullable
- last_error_code
- updated_at

publish_tombstones
- target_id
- article_id
- previous_article_version_id
- reason(deleted/merged/rejected/replaced)
- status
- created_at
```

关键幂等约束：

```text
UNIQUE(target_id, generation_id, article_version_id)
```

如果目标是滚动增量而不是全 generation，则增加：

```text
UNIQUE(target_id, article_id, article_version_id, adapter_version)
```

### 5.3 Publish Bundle

平台内部先生成目标无关的 bundle：

```json
{
  "schema_version": 1,
  "article_id": "...",
  "article_version_id": "...",
  "site_id": "...",
  "source_url": "...",
  "canonical_url": "...",
  "title": "...",
  "author": "...",
  "published_at": "...",
  "updated_at": "...",
  "tags": [],
  "content_sha256": "...",
  "source_snapshot_id": "...",
  "provenance": {
    "profile_release": "...",
    "contract_release": "...",
    "extractor_release": "...",
    "serializer_version": "..."
  }
}
```

随后：

```text
BedrockS3Adapter
 -> document.md
 -> document.md.metadata.json

LocalMarkdownAdapter
 -> document.md with Front Matter
 -> manifest.jsonl

GenericSearchAdapter
 -> content + filter fields
```

这样 Bedrock 的字段限制不会污染 canonical 数据模型。

## 6. Bedrock S3 Adapter 的实现建议

### 6.1 Sidecar 生成

对 Bedrock S3 目标，建议由 adapter 生成：

```text
.../document.md
.../document.md.metadata.json
```

可过滤 metadata 优先放稳定、低基数或明确业务含义字段，例如：

```text
site_id
article_id
article_version_id
canonical_url
domain
author
published_at
tags
language
content_sha256
```

`source_snapshot_id`、extractor release 等运维 provenance 可以保留在平台内部；是否导入 Bedrock 要考虑 metadata 大小和检索价值，避免把大量内部字段塞入 embedding 或过滤索引。

### 6.2 `includeForEmbedding`

当前 Bedrock metadata schema 可以控制 metadata 是否参与 embedding。生产上不应全部设置为 true。

建议：

- `title`、少量高价值主题 tag 可按检索评估决定是否参与 embedding；
- `article_id`、hash、内部版本号通常只用于过滤/追溯，不参与 embedding；
- URL 是否参与 embedding 需要实验，很多技术 URL 含路径关键词，但也可能制造噪声；
- 任何字段变化都通过 target config version + shadow evaluation 管理。

### 6.3 增量同步

Bedrock S3 数据源支持对新增、修改、删除进行增量同步，但平台仍不能把 S3/Bedrock 当 checkpoint 真相源。

正确关系是：

```text
PG article current version
 -> publish_documents desired state
 -> S3 object desired state
 -> Bedrock sync job
 -> target observed state
```

平台的调度依据是 `article_version` 是否变化，而不是“上次脚本是否执行过”。

### 6.4 删除与文章合并

当文章：

- 多次 404/410 后确认删除；
- canonical merge 到另一 article；
- 被人工 reject；
- 站点退出知识库；

Publish Plane 必须生成 delete/tombstone 操作。只做新增/覆盖会让外部 RAG 长期保留幽灵文档。

## 7. 增量发布算法

目标状态以 `articles.current_version_id` 为准。

```text
for each target:
  read published watermark / article change feed
  for each changed article:
    if current_version exists and accepted:
       compare target last article_version_id
       same -> noop
       different -> build bundle -> stage -> sync
    else:
       emit tombstone/delete
```

推荐由 outbox 产生：

```text
article_version_published
article_deleted
article_merged
article_visibility_changed
```

Publish Worker 订阅这些事件并 UPSERT desired state。为了防止消息丢失或长期失败，还要有低频 reconciliation：

```text
PG current article set
 vs
publish target observed set
 -> missing / stale / orphan diff
```

这样即使 Redis Streams 消息曾经失败，也能最终收敛。

## 8. 对文章代码的逐点改进

### 8.1 `AsyncWebCrawler` 生命周期

文章在循环逻辑附近创建 crawler context。生产上应：

- Browser process 池复用；
- page/context 按站点或任务隔离；
- domain limiter 跨 HTTP/Browser 共用；
- Worker 有 max pages/process、RSS、age recycle；
- 不允许每个 URL 冷启动 Chromium。

### 8.2 标题提取

不能使用 `raw_markdown.split("\n")[0]` 作为唯一标题来源。

应构建 metadata candidates：

```text
site selector
> JSON-LD headline/name
> OpenGraph og:title
> <title>
> article H1
> generic extractor
```

最终值携带 source/confidence/provenance。

### 8.3 内容选择

不能只保存 fitted Markdown。

建议：

```text
raw response bytes      -> S3 immutable snapshot
rendered DOM (如需要)   -> S3 immutable snapshot
extractor candidates    -> S3/PG metadata
canonical Markdown      -> article_version
Bedrock retrieval view  -> Publish Plane 派生
```

### 8.4 文件写入

文章直接写本地文件。生产上应使用：

- 临时 object key / staging prefix；
- content hash；
- manifest；
- idempotent UPSERT；
- 对象创建完成后再更新 PG 状态；
- 大对象 streaming；
- server-side encryption；
- checksum/size verify。

### 8.5 URL 和 SSRF

文章默认接受传入 URL。生产系统中所有 URL 来源，包括人工输入、Sitemap、Feed、canonical、redirect、图片和外部索引，都必须经过：

```text
scheme check
host normalization
DNS resolution
private/reserved IP deny
approved host/scope
robots
redirect hop recheck
request/byte/time budget
```

## 9. Web 管理端需要增加的发布能力

建议在现有 Web 管理中增加 `Knowledge Targets / Publish` 页面：

### Target 管理

- 新增/禁用 target；
- target kind；
- bucket/prefix/region 等非敏感配置；
- secret reference；
- metadata mapping；
- includeForEmbedding 配置；
- adapter/config version；
- 测试连接。

### Generation / Sync

- building/ready/syncing/active/failed；
- document count / bytes；
- pending/stale/failed/orphan；
- 最近同步时间；
- target index freshness；
- re-publish without refetch；
- generation rebuild；
- target reconciliation。

### 单文档诊断

从文章页可以看到：

```text
article_version
 -> canonical Markdown
 -> publish bundle
 -> target content object
 -> target metadata object
 -> target sync/index status
```

并可查看 metadata mapping、hash 和最近错误。

## 10. 可观测性

新增指标：

```text
publish_events_total
publish_documents_staged_total
publish_documents_failed_total
publish_target_lag_seconds
publish_generation_age_seconds
publish_reconciliation_missing_total
publish_reconciliation_orphan_total
publish_bytes_total
publish_metadata_size_bytes
publish_adapter_errors_total
target_sync_duration_seconds
target_sync_failures_total
```

SLO 建议：

- canonical article 发布不受 target 故障影响；
- 正常增量文章在目标知识库的 freshness lag 有明确 p95/p99；
- 每个 target 可独立重建；
- 任意 target 文档可追溯到唯一 `article_version`；
- target 孤儿文档能通过 reconciliation 发现并处理。

## 11. 安全与权限

Publish Plane 是新的出站边界，需要单独治理：

1. AWS/外部后端凭证通过 secret manager/secret ref 获取，不进入数据库明文配置、日志、Markdown 或 metadata。
2. Web 管理页默认掩码敏感配置。
3. S3 bucket/prefix 使用最小权限 IAM。
4. target adapter 只能访问配置允许的 bucket/index/collection。
5. metadata 中禁止 Authorization、Cookie、内部 token、抓取代理密钥。
6. 对象存储启用服务端加密和生命周期策略。
7. 发布到第三方 SaaS 前支持 site/tenant policy 和数据分类门禁。
8. 外部知识库返回的文本仍是不可信数据，不可因此扩大 MCP/Agent 工具权限。

## 12. 对现有技术方案的具体优化

本次建议加入以下最终能力：

### 优化 1：新增 Knowledge Sink / Publish Plane

现有 Knowledge Access Plane 负责平台自身检索；新增 Publish Plane 负责向外部 RAG/知识库投递。二者都消费已发布 `article_version`，互不作为 canonical 真相源。

### 优化 2：把 metadata/provenance 变成正式导出契约

Front Matter 继续保留给人类阅读，但机器集成使用版本化 `Publish Bundle`。Target adapter 只映射字段。

### 优化 3：支持正文 + metadata sidecar 的成对发布

Bedrock S3 Adapter 生成同 identity 的 content/metadata pair，并使用 staging + manifest + target state 避免“正文成功、metadata 失败”被误认为发布成功。

### 优化 4：引入 target generation 与 reconciliation

任何 downstream index 都可重建、可 shadow、可切换；增量事件之外增加周期性 desired-vs-observed 对账。

### 优化 5：外部知识库失败不 refetch

只基于已发布 Markdown/S3 artifact 重试 publish/index，不再次访问 1000 个源站。

## 13. 最终推荐链路

```text
                        ┌──────────────────────────────┐
                        │ Source Discovery Providers   │
                        └──────────────┬───────────────┘
                                       v
Frontier -> HTTP/Browser -> Immutable Snapshot -> Extractors
                                       |
                                       v
                                Quality Gate
                                       |
                                       v
                         Article IR + Metadata Resolver
                                       |
                                       v
                          Deterministic Markdown
                                       |
                                       v
                              article_version
                          /                    \
                         v                      v
              Knowledge Access Plane      Publish Plane
              chunks/search/vector        export bundle
                    |                     /     |       \
                    v                    v      v        v
              Search API/MCP          Local   S3   Bedrock KB
```

这使“抓取 1000 个博客”和“把知识投递到哪一种 RAG 产品”彻底解耦。未来即使替换 Bedrock、向量数据库或自建检索引擎，也不需要修改 Source Discovery、Fetch、Extract、Article Version 主链路。

## 14. 验收标准

加入该能力后应满足：

- 任意已发布 `article_version` 能生成确定性的 Publish Bundle；
- 同一 `article_version + target config + adapter version` 重放得到相同 content/metadata hash；
- Bedrock S3 target 能生成正文 + `.metadata.json` sidecar；
- metadata 至少包含 source/canonical URL 与文章/版本 identity；
- target metadata schema/version 可追溯；
- 目标端故障不影响 canonical Markdown 发布；
- target retry/rebuild 不访问源站；
- 文章更新只发布新版本，不全量重传所有文章；
- 文章删除/合并能形成明确 tombstone/delete；
- target reconciliation 能发现 missing/stale/orphan；
- Web 能查看 target lag、generation、文档映射、失败原因并触发 re-publish；
- credentials 不进入 metadata、manifest、错误日志或前端正文预览。

## 15. 参考资料

- 原文章：https://medium.com/@gargasm/custom-web-scraper-for-amazon-bedrock-kb-with-metadata-b3c7942e0ff2
- Crawl4AI：https://github.com/unclecode/crawl4ai
- Amazon Bedrock Knowledge Bases S3 data source：https://docs.aws.amazon.com/bedrock/latest/userguide/s3-data-source-connector.html
- Amazon Bedrock metadata：https://docs.aws.amazon.com/bedrock/latest/userguide/kb-metadata.html
