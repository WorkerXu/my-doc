# 我为自己自动化了一个家庭实验室 / 自托管主题 Newsletter，后来决定开放给所有人使用

## 1. 调研对象

- 编号：39
- 原文标题：I automated a homelab/self-hosting newsletter for myself, but then I thought I'd make it available for everyone
- 原文：https://www.reddit.com/r/homelab/comments/1kc3wr1
- 同作者补充讨论：https://www.reddit.com/r/selfhosted/comments/1kie9g7
- 输出实例：https://iamthecloud.substack.com/
- 核心技术：Python、Docker、Crawl4AI、LLM 多角色协作、人工编辑、Substack

本文只分析公开信息能够确认的实现与基于这些事实可合理推导的工程设计。没有公开源码的部分会明确标为“工程推断”，避免把推断当成作者已实现事实。

## 2. 公开信息可以确认的实现

作者描述的流水线非常清晰：

1. 平时关注大量 homelab、自托管相关网站、Feed、Reddit 等来源；
2. 每周由本地 Python 程序统一抓取；
3. 抓取层使用 Crawl4AI，把网页转换成 Markdown；
4. 系统模拟一个“虚拟编辑部”：不同 AI writer 消化素材并向 AI editor 投稿；
5. editor 汇总后生成一期 Newsletter 草稿；
6. 作者仍会人工阅读、修改草稿，因为 AI 会产生质量问题；
7. 作者还会记录自己特别喜欢的文章，并提供“强制加入本期”的能力；
8. 当前发布到 Substack，未来设想包括 LocalAI、本地模型以及自托管 Newsletter；
9. 整套程序用 Python 编写，在本地 Docker 中运行。

这不是一个“爬虫 + 一次 LLM prompt”的脚本，而是一个具有采集、内容标准化、角色化生成、聚合编辑、人工审核和发布适配的分阶段内容流水线。

## 3. 可能的系统结构

结合作者描述，最合理的工程结构如下：

```text
Source Registry
  |-- Websites
  |-- RSS / feeds
  |-- Reddit / community pages
  |-- Manual Pins
        |
        v
Weekly Scheduler
        |
        v
Crawler / Crawl4AI
        |
        v
Markdown Material Pool
        |
        +-------------------------+
        |                         |
        v                         v
Writer A / Persona A       Writer B / Persona B ...
        |                         |
        +------------+------------+
                     v
               AI Editor
                     |
              Issue Draft
                     |
             Human Review
                     |
              Publish Adapter
                     |
                  Substack
```

这里最值得借鉴的是**层与层之间的职责边界**，而不是“给多个模型起不同名字”。如果 writer、editor、发布器之间没有结构化输入输出、版本和证据约束，多 Agent 只会把一次不可控生成变成多次不可控生成。

## 4. 抓取层实现与技术原理

### 4.1 Crawl4AI 在这里承担什么角色

Crawl4AI 官方提供 `AsyncWebCrawler`，能够抓取网页并将最终 HTML 转换为 Markdown；对于批量 URL 可以使用 `arun_many()` 配合 dispatcher 控制并发、内存和速率。官方文档同时区分 raw markdown 与经过内容过滤后的 fit markdown。

相关官方文档：

- https://docs.crawl4ai.com/core/quickstart/
- https://docs.crawl4ai.com/core/markdown-generation/
- https://docs.crawl4ai.com/api/arun_many/
- https://docs.crawl4ai.com/advanced/multi-url-crawling/

作者选择 Crawl4AI 的原因很直接：LLM 后续最容易消费结构清晰的 Markdown，而不是包含导航、脚本、样式和广告的原始 HTML。

### 4.2 “网页转 Markdown”并不等于“正文已正确抽取”

这是这类方案最容易被忽略的地方。

Crawl4AI 可以稳定完成 HTML -> Markdown，但真实知识库需要继续回答：

- 抓到的是文章正文还是首页/分类页？
- 导航、推荐阅读、评论区、Cookie Banner 是否混入？
- 代码块、表格、脚注、图片说明是否保留？
- JavaScript 懒加载内容是否抓全？
- 页面是否被 WAF 返回了“验证页面”，却仍然是 HTTP 200？
- 同一篇文章是否在多个 URL 重复出现？

所以若把该实践扩展成生产知识库，Crawl4AI 输出应该是 **Extraction Candidate**，而不是最终事实层。应先保留原始 Artifact，再经过正文抽取、质量检查、Canonical IR 和 Markdown Renderer。

### 4.3 批量抓取的正确调度方式

作者按周运行很适合 Newsletter，但扩展到 1000 个技术博客时不能简单写成：

```python
await asyncio.gather(*(crawl(url) for url in urls))
```

生产方案需要：

- 每域名并发上限；
- 全局 Browser Context 上限；
- 429/503 指数退避；
- Retry 与 DLQ；
- Backfill 和 Incremental 分队列；
- 大站点公平调度，避免一个来源占满 Worker；
- Checkpoint，避免任务失败后从头跑；
- robots/访问策略与来源级限速。

Crawl4AI 的 dispatcher 可以解决单进程中的一部分资源控制，但平台级公平性、任务持久化、幂等和恢复仍应该由外部调度层负责。

## 5. Writer 层的实现原理

作者把多个 AI 设计成具有不同 persona 的“记者”。从工程角度看，persona 实际上等于一组版本化的生成策略：

```text
writer_release
- writer_id
- persona_version
- system_prompt_version
- model
- temperature
- max_tokens
- allowed_topics
- output_schema_version
```

### 5.1 Writer 不应该直接接整站 Markdown

推荐先形成结构化素材包：

```json
{
  "document_version_id": "...",
  "title": "...",
  "source": "...",
  "canonical_url": "...",
  "published_at": "...",
  "markdown": "...",
  "evidence_refs": ["artifact://..."],
  "topic_tags": ["homelab", "self-hosted"]
}
```

writer 的输出同样应该结构化：

```json
{
  "document_version_id": "...",
  "headline": "...",
  "summary": "...",
  "why_it_matters": "...",
  "citations": ["..."],
  "confidence": 0.82
}
```

这样 editor 才能进行去重、排序、引用验证和重试，而不是解析一大段自由文本。

### 5.2 Persona 的正确用途

Persona 适合控制：

- 语气；
- 关注角度；
- 输出格式；
- 读者画像；
- 选题偏好。

Persona 不应该有权修改：

- 原文 URL；
- 作者；
- 发布时间；
- 抓取事实；
- Canonical Markdown；
- Document Identity。

也就是说，persona 属于 **Presentation / Curation Plane**，不能进入 Collection Truth。

## 6. AI Editor 层的技术原理

“编辑”这个 Agent 的价值不是把几段文字拼起来，而是做跨稿件决策：

1. 去重：多个 writer 是否在写同一件事；
2. 排序：什么值得放在前面；
3. 覆盖：本期是否过度集中在单一来源或主题；
4. 结构：建立栏目和叙事顺序；
5. 风格：把不同 writer 的语言统一；
6. 引用：确保每个事实能回溯原始材料；
7. 预算：控制篇幅和生成成本。

推荐把 editor 设计成显式状态机：

```text
COLLECT
 -> SCORE
 -> DEDUP
 -> SELECT
 -> OUTLINE
 -> COMPOSE
 -> VERIFY_CITATIONS
 -> HUMAN_REVIEW
 -> APPROVED
 -> PUBLISH
```

任何阶段失败都可以单独重跑，不应该重新抓整批网站。

## 7. “Boss 强制加入文章”是一个很重要的设计

作者提到自己会在一周中记下喜欢的文章，并能强制要求系统加入。这看似是小功能，实际上体现了正确的产品边界：**自动推荐和人工编辑权并存**。

建议将它建模为不可变的 `editorial_directive`：

```text
editorial_directive
- directive_id
- issue_id
- document_version_id
- action: PIN | EXCLUDE | PRIORITIZE
- reason
- created_by
- created_at
```

关键点：

- PIN 只影响某一期内容选择；
- 不修改 Source Coverage；
- 不修改文档质量评分；
- 不修改全局检索排序；
- 所有人工操作可审计、可撤销。

这对知识库 Web 管理端同样有价值：管理员可以把某篇文章“钉住”到专题、Digest 或人工审阅队列，而不会污染基础采集事实。

## 8. 人工审核为什么仍然必要

作者明确表示当前仍需花时间编辑，因为 AI 会犯错。这一点比“未来全自动发布”更值得重视。

对于生成式 Newsletter，至少应检查：

- 是否编造事实；
- 是否错误归因；
- 引用 URL 是否真实存在；
- 是否把两篇不同文章混成一篇；
- 是否遗漏反例和限定条件；
- 是否重复内容；
- 是否出现不合适的语气或敏感内容；
- 是否把抓取失败页面当成新闻素材。

因此默认策略应该是：

```text
AI Draft != Publishable Artifact
```

只有通过自动 Citation Gate + Quality Gate，再由人工批准，才能进入发布状态。后续即使提高自动化，也应保留基于风险等级的审批策略：低风险自动发布，高风险强制人工。

## 9. Docker 与本地模型的意义

作者在本地 Docker 运行 Python 系统，并考虑 LocalAI。这个选择带来三个工程价值：

### 9.1 可复现

固定：

- Python 版本；
- Crawl4AI 版本；
- Chromium / Playwright 版本；
- Prompt；
- 模型；
- Renderer。

否则同一个输入在几周后可能得到完全不同的输出，却无法解释原因。

### 9.2 模型可替换

建议定义统一 `ModelGateway`：

```text
ModelGateway
  |-- OpenAI-compatible cloud provider
  |-- LocalAI
  |-- Ollama / local runtime
  |-- other provider
```

业务层只依赖结构化生成接口，不依赖某个厂商 SDK。

### 9.3 隐私与成本边界

Canonical Markdown 可以只在本地保存；送往外部模型的内容可以经过策略控制。对于版权或隐私敏感来源，可以强制使用本地模型，甚至完全禁用生成。

## 10. 该实践目前存在的工程短板

从公开描述看，它是很好的个人自动化项目，但不能直接等价为 1000 站知识库方案。

### 10.1 缺少历史 Coverage 模型

“每周抓 everything”无法证明：

- 网站历史文章是否全部发现；
- Sitemap 是否分区；
- Archive 页是否翻到底；
- 是否存在已删除/迁移 URL；
- 哪段时间仍有 Known Gap。

Newsletter 只要求“这周够用”，知识库要求“历史完整性可解释”。

### 10.2 抓取、事实、生成混得太近

如果 Crawl4AI Markdown 直接成为 writer 的唯一输入，一次抽取错误会一路传播到 editor 和最终 Newsletter。生产系统必须保存 Raw Artifact 和 Canonical IR，生成层只能读取已通过质量门的 Document Version。

### 10.3 缺少显式版本与证据

Prompt、persona、模型升级都会改变输出。没有 release id 就无法重放和比较。

### 10.4 周批处理不等于增量同步

知识库需要针对 RSS/Sitemap/API 的短周期增量，以及 ETag / Last-Modified / Content Hash 等更新检测；周批处理更适合 Digest，而不是 Collection。

### 10.5 “多 Agent”可能放大错误和成本

每多一层生成都会增加：

- token 成本；
- 延迟；
- 不确定性；
- 风格漂移；
- hallucination 机会。

所以多 Agent 只应出现在确实需要角色分工的 Curation Plane，采集主链保持确定性。

## 11. 对博客知识库方案的直接优化建议

这篇实践最适合给现有方案增加一个**可选的 Editorial / Digest Plane**，而不是修改基础抓取主链。

### 11.1 新增边界

```text
Collection Truth
Raw Artifact -> Canonical IR -> Document Version -> Markdown
                                      |
                                      v
                              Curation / Digest Plane
                                      |
                        Candidate Selection / PIN
                                      |
                         Writer Agents (optional)
                                      |
                                AI Editor
                                      |
                             Citation Gate
                                      |
                             Human Review
                                      |
                          Publish / Export Adapter
```

### 11.2 新增领域对象

```text
curation_issue
curation_item
editorial_directive
generation_attempt
writer_release
editor_release
editorial_review
publication_target
publication_attempt
```

### 11.3 新增 Web 管理能力

- 创建一期 Digest / 专题；
- 从已通过 Quality Gate 的文档中选材；
- 手工 PIN / EXCLUDE / PRIORITIZE；
- 查看 writer/editor 每一步输入输出；
- 对照原文证据；
- 局部重生成某一段，而不是整期重跑；
- 人工 Approve / Reject；
- 发布到 Markdown、Webhook、邮件或第三方平台；
- 查看 Prompt / Model / Persona Release；
- 统计每一期生成成本和人工修改量。

### 11.4 必须保持的隔离规则

1. Digest 失败不能阻塞抓取；
2. Digest 的选题不能修改 Source Coverage；
3. LLM 不能创造 citation URL；
4. 生成文本不能覆盖 Canonical Markdown；
5. PIN 只影响指定 Curation Issue；
6. 所有生成结果可删除、可重建；
7. 所有引用必须指向真实 Document Version / Artifact 证据；
8. Human Review 默认开启，自动发布必须由策略显式放行。

## 12. 推荐的生产级落地方式

如果把作者的实践升级成知识库上的可选内容编排服务，建议按下面的最小流程实现：

```text
1. Scheduler 创建 curation_issue
2. Query Service 从 PASS Document Version 生成候选集
3. 应用 PIN / EXCLUDE 等 editorial_directive
4. Story Cluster 去重并做来源多样性控制
5. writer 读取结构化 Evidence Package 生成稿件块
6. editor 只在稿件块和 Evidence 上做编排
7. Citation Validator 校验 URL 与 Document Version
8. 自动质量评分
9. Human Review 查看 diff + 原始证据
10. Approve 后由 Publication Adapter 发布
11. 保存 publication_attempt 与最终快照
```

生成结果应是 Projection，不是业务真相。

## 13. 结论

这个项目最大的启发不是“用 Crawl4AI + 多个 AI 就能自动写 Newsletter”，而是一个非常实用的分层模式：

- 抓取工具负责把异构网页变成机器可消费材料；
- writer 负责局部理解和表达；
- editor 负责跨文档编排；
- 人工拥有最终编辑权；
- 手工 PIN 给人类保留强制干预能力；
- 发布平台只是最后一个 Adapter；
- Docker / 本地模型让运行环境和模型供应商可替换。

对于 1000+ 技术博客知识库，应该保留这个模式中“分层、人工审核、人工 PIN、模型可替换”的优点，但必须补上历史 Coverage、增量同步、Artifact、Canonical IR、质量闸门、稳定身份、版本、审计和可重放机制。
