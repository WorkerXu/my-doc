# 我为自己自动化了一个家庭实验室／自托管主题 Newsletter，后来决定开放给所有人使用

## 1. 调研对象与结论

- 编号：39
- 原帖名称：I automated a homelab/self-hosting newsletter for myself, but then I thought I'd make it available for everyone
- 调研表名称：我为自己自动化了一个家庭实验室 / 自托管主题 Newsletter，后来决定开放给所有人使用
- 原帖：https://www.reddit.com/r/homelab/comments/1kc3wr1
- 项目/Newsletter：I Am the Cloud
- 已公开发行样例：https://iamthecloud.substack.com/p/from-sudos-funeral-to-saudis-billions
- 形态：Python + Docker + Crawl4AI + 多角色 AI Writer + AI Editor + 人工审核 + Substack 发布
- 代码仓库：本次检索未发现作者公开的对应源码仓库，因此以下实现分析基于作者公开描述、其 Newsletter 实际成品和可观察的运行行为进行架构还原。

这个项目最值得研究的地方并不是“Crawl4AI 把网页转 Markdown”本身，而是它把内容生产拆成了一个很轻量的“虚拟编辑部”：不同 AI 角色先从多个 homelab/self-hosting 信息源获取材料并形成候选内容，再交给一个 AI editor（作者称为 **Son of Anton**）编排成稿，最后由人做审核和修改，再发布到 Substack。

从 1000 个技术博客长期知识库的目标来看，这个项目**不能直接作为采集主架构**：它没有证明全历史 Coverage，没有持久 URL Inventory，没有不可变 Raw Artifact、Document Version、增量变更检测、幂等任务、跨站公平调度、质量闸门等生产能力；而且“Crawler 直接产 Markdown -> AI Writer/Editor”会把采集事实与内容编排混在一起。

但它对现有博客知识库技术方案有四个很有价值的补强点：

1. **把多 AI Writer 正式建模成可选的 Editorial Perspective Workers，而不是让 Agent 自己随意联网抓取。**它们只能消费已经通过知识库 Quality Gate 的 Evidence Package。
2. **LLM Provider 必须抽象化。**作者计划从云模型迁到 LocalAI/本地运行，这说明 Writer/Editor 不应该绑定具体厂商 API；本地、云端、OpenAI-compatible endpoint 应统一为可版本化 Provider。
3. **发布适配器应该“Draft First + Capability Negotiation”。**作者现在仍人工修改稿件并借助 Substack 快速发布，生产系统不能假设所有外部 Newsletter 平台都有稳定写 API，更不能默认用脆弱 Browser 自动化直接发布。
4. **增加单机 Docker Compose Pilot 拓扑。**这个项目证明 Python + Docker 很适合快速验证，但 Pilot 和 1000 站生产环境必须共享同一状态机、数据模型、任务协议和对象存储契约，才能做到“先小后大而不重写”。

因此本次调研的最终结论是：**保留现有 Collection Truth 架构不变，把该项目的“虚拟编辑部、本地模型、Draft-first 发布、单机 Docker Pilot”吸收到可选 Editorial/Deployment 平面；明确拒绝把 Crawl4AI Markdown、Agent 自主抓取或全自动发布升级为采集事实链路。**

---

## 2. 项目公开描述所反映的实际链路

作者描述的目标是给自己做一个 homelab/self-hosting 自动 Newsletter，随后决定公开给其他人订阅。整体可还原为：

```text
多个技术站点 / Feed / Reddit / 社区内容
        |
        v
Crawl4AI 周期抓取
        |
        v
Markdown / 页面内容
        |
        +-------------------+
        |                   |
        v                   v
 AI Writer A           AI Writer B ...
  不同角色/人格          不同关注方向
        \                   /
         \                 /
          +------ fan-in --+
                   |
                   v
          AI Editor: Son of Anton
                   |
                   v
                Draft
                   |
                   v
              人工修改/审核
                   |
                   v
               Substack
```

作者说明整个程序是 Python 编写、在本地 Docker 中运行，每周抓取相关内容。Crawl4AI 被用来把目标网页变成适合后续 LLM 消费的 Markdown。多个“writers”提交内容给 AI editor，editor 生成一期草稿。作者当时仍会花时间人工修改，并希望未来逐步自动化发布。

作者还明确考虑将模型迁到本地，例如通过 LocalAI 一类兼容服务运行，从而减少外部依赖，并进一步考虑自托管 Newsletter，而现阶段选择 Substack 的主要价值是快速上线、降低发布和订阅管理成本。

公开的 `I Am the Cloud` 成品也能验证这种编辑链路。实际 Newsletter 不是简单“把若干链接拼起来”，而是包含稳定栏目、栏目级开场语、单条内容摘要、编辑点评以及人工挑选内容。例如可以看到类似：

```text
Boot Message
Homelab Homies
Release Radar
Paranoia Corner
Community Champions
Rack Candy & Wallet Sins
```

并且存在 `Son of Anton's thoughts`、`Boss pick!` 这类明显的“AI 编辑人格 + 人工强制精选”痕迹。这说明它本质上是：

```text
事实素材集合
 -> 多角色观点/摘要候选
 -> 栏目化编辑
 -> 人工 PIN/修改
 -> 发布
```

而不是单纯的网页抓取器。

---

## 3. Crawl4AI 在这里解决的是什么问题

### 3.1 它主要解决“给 LLM 一个干净输入”

对于 Newsletter 场景，作者关心的是每周把若干网页转换成较干净、Token 友好的文本，再给 Writer/Editor 使用。Crawl4AI 在这种场景中非常合适，因为它能把动态网页渲染、DOM 清洗和 Markdown 输出封装起来。

它解决的核心问题是：

```text
Web Page
 -> Browser / HTML processing
 -> remove obvious page chrome
 -> readable Markdown-ish content
 -> LLM context
```

这与“长期知识库 Canonical Markdown”完全不是同一个可靠性级别。

### 3.2 为什么不能把 Crawl4AI Markdown 直接当知识库真相

Newsletter 的容错与知识库不同。某一期漏掉一个代码块、表格或某段正文，可能只影响摘要质量；但知识库要求：

- 能重放；
- 能解释某个版本从哪里来；
- 能在 Extractor 升级后离线重建；
- 能判断正文是否截断；
- 能区分页面更新与导航噪声变化；
- 能保留代码、表格、图片、引用的结构；
- 能对同一 Document 做版本管理。

所以生产方案必须保持：

```text
Raw Artifact
 -> 多 Extraction Candidate
 -> 独立 Quality Gate
 -> Canonical IR
 -> Markdown Projection
```

而不是：

```text
Crawl4AI Markdown -> 直接保存为最终知识库
```

该项目证明 Crawl4AI 很适合“LLM 输入候选”，并不证明其原始 Markdown 应成为 Canonical Truth。

---

## 4. “虚拟编辑部”的技术原理

### 4.1 Fan-out / Fan-in

多个 AI Writer 的本质不是“多个模型更聪明”，而是把同一个候选素材集合做并行变换：

```text
Evidence Set
   |----> Perspective A ----|
   |----> Perspective B ----|--> Editor/Composer
   |----> Perspective C ----|
```

这是一种典型的 fan-out / fan-in 工作流。

优势：

- 可以让不同 Writer 关注不同主题；
- 可以让一个 Writer 偏技术细节、另一个偏社区、另一个偏安全；
- Editor 能从多个候选中做选择，而不是让一个 Prompt 同时承担“选题、事实判断、摘要、风格、排序、组稿”全部职责；
- 单个 Writer 失败时可降级，不必让整期失败。

风险：

- 如果每个 Writer 都自行联网，会重复抓取和放大成本；
- 每个 Agent 看到的网页时间点不同，会出现证据不一致；
- Agent 可能自己扩展来源集合，使引用无法审计；
- 同一 URL 被多个 Agent 独立抓取，无法共享缓存、限速和 Artifact；
- Persona 容易被误用为“事实权限”。

因此生产系统应把虚拟编辑部改造成**证据受控的多角色生成层**。

### 4.2 Evidence Package

每一期开始时，由确定性的 Candidate Selector 从已经 PASS 的 `document_version` 中冻结一份 Evidence Package：

```json
{
  "issue_id": "...",
  "snapshot_at": "...",
  "documents": [
    {
      "document_version_id": "...",
      "title": "...",
      "canonical_url": "...",
      "published_at": "...",
      "source_id": "...",
      "markdown_object_key": "...",
      "evidence_hash": "..."
    }
  ]
}
```

Writer 只允许读取这个 Package，不允许默认再访问外部网络。

这样能保证：

- 所有 Writer 基于同一事实快照；
- 引用可以回溯；
- 多 Agent 之间不会重复抓源站；
- 能重放同一期；
- Prompt/Model 升级后能做离线对比；
- Collection Truth 不会被编辑风格污染。

### 4.3 Editorial Candidate

每个 Writer 输出结构化对象，而不是一大段自由文本：

```json
{
  "agent_release": "homelab_security_writer_v3",
  "items": [
    {
      "document_version_id": "...",
      "section": "Paranoia Corner",
      "headline": "...",
      "summary": "...",
      "why_it_matters": "...",
      "evidence_refs": ["..."],
      "confidence": 0.91
    }
  ]
}
```

Editor 再负责：

- 去重；
- 栏目排序；
- 长度预算；
- 风格统一；
- 冲突检测；
- 人工 PIN 的强制插入；
- 最终结构组装。

### 4.4 Persona 只影响表达，不拥有事实权限

`Son of Anton` 这种人格非常适合提升 Newsletter 可读性，但生产系统必须规定：

```text
Persona CAN:
- 语气
- 幽默程度
- 栏目风格
- 标题候选
- 解释角度

Persona CANNOT:
- 修改 canonical URL
- 修改作者/发布时间
- 创造不存在的来源
- 将 REJECT 文档变成可用文档
- 修改 Document Identity
- 绕过 Citation Gate
```

人格是展示层策略，不是事实源。

---

## 5. “Boss pick” 对生产方案的启发

公开 Newsletter 中能看到人工精选的 `Boss pick!`。这是一个很关键的设计信号：即使 AI 能做选题，作者仍希望保留“这一条必须进本期”的编辑权。

现有方案中的 `Manual PIN` 正好对应这个需求：

```text
editorial_directive
- issue_id
- document_version_id
- action = PIN | EXCLUDE | PRIORITIZE
- reason
- created_by
```

需要坚持的原则是：

- PIN 只影响指定 Issue；
- 不修改 Collection Coverage；
- 不改变正文 Quality；
- 不改变 Document Identity；
- 不反向影响未来增量同步；
- 全部可审计、可撤销。

这比“把人工喜欢的文章提高全局爬取优先级，并永久保留更多”更安全，因为**编辑偏好与采集完整性是两个不同维度**。

---

## 6. 本地 Docker 的价值与边界

### 6.1 为什么 Python + Docker 很适合 Pilot

这个项目可以在本地 Docker 中运行，说明对于少量站点的内容聚合，单机部署有几个明显优势：

- 环境可重复；
- Chromium/Crawl4AI 依赖容易固定；
- 数据可以留在本地；
- 调试简单；
- 成本低；
- 适合快速调整 Prompt、栏目和 Writer Persona。

因此博客知识库不应该要求开发阶段一开始就上 Kubernetes。

### 6.2 但不能让 Pilot 形成第二套架构

错误方式：

```text
Pilot:
cron + SQLite + local files + ad-hoc scripts

Production:
重新写 PostgreSQL + queue + S3 + worker
```

这样从 20 个站扩到 1000 个站时几乎必然重写。

推荐方式：

```text
Pilot Docker Compose
- API/Scheduler/Worker 可合并部署
- PostgreSQL
- Redis
- MinIO
- Browser Worker optional

Production
- 同一 PostgreSQL schema
- 同一 Redis Streams contract
- 同一 Object Storage key contract
- 同一 Run/Task state machine
- 仅拆分并水平扩容 Worker
```

**部署拓扑可以变，业务语义不能变。**

---

## 7. LocalAI / 本地模型带来的架构要求

作者考虑把 AI 部分迁到 LocalAI，这个方向对知识库方案很有价值，因为 1000 站长期运行后，Summary、Topic、Entity、Curation、Writer/Editor 的调用量可能很大。

如果代码直接写：

```python
client = SomeCloudVendor(...)
```

后续迁本地会把业务逻辑和模型供应商绑死。

应增加统一的 LLM Provider Gateway：

```text
Generation Job
 -> llm_policy
 -> provider resolver
    |-- cloud provider
    |-- OpenAI-compatible local endpoint
    |-- LocalAI
    |-- Ollama/vLLM compatible adapter
 -> structured response validator
```

建议记录：

```text
llm_provider_release
- provider_type
- endpoint_profile
- model_id
- model_revision
- tokenizer/profile
- max_context
- structured_output_capability
- tool_capability
- privacy_class
- cost_profile
- timeout_policy
```

路由可以根据：

- 数据隐私；
- Token 成本；
- 延迟；
- 任务类型；
- 所需上下文；
- 结构化输出能力；
- 质量评测结果；

选择模型。

关键原则：

1. Collection 主链不能依赖 LLM；
2. 本地模型不可用不能阻塞 Markdown 发布；
3. Writer/Editor 的 Provider、Model、Prompt 必须全部可回放；
4. 不允许本地模型因为“更私密”就绕过 Citation/Quality Gate；
5. 模型 Provider 切换应通过 Golden/Offline Eval，而不是直接替换。

---

## 8. Substack 发布链路的生产化改造

### 8.1 为什么作者先用 Substack 是合理的

对个人项目来说，Substack 解决了：

- 订阅；
- 邮件投递；
- Web 归档；
- 基础模板；
- 发布后台；
- 用户管理。

这能让作者把时间放在内容管线而不是邮件基础设施。

### 8.2 不能把外部平台假设成稳定 API

生产系统需要把 Publication Adapter 定义为能力协商，而不是统一假设：

```text
publish(post)
```

更合理的契约：

```text
PublicationCapabilities
- create_draft
- update_draft
- publish
- unpublish/archive
- upload_media
- schedule
- webhook
- idempotency_key
```

每个 Adapter 在运行时声明能力。

### 8.3 Draft First

默认策略：

```text
APPROVED_CONTENT
 -> create external draft
 -> store external_draft_id
 -> human preview
 -> explicit publish command
```

而不是：

```text
AI generated -> direct publish
```

如果外部平台没有稳定、授权明确的写 API：

```text
Markdown/HTML Export
 -> human handoff
```

优先于用 Playwright 模拟后台点击。Browser 自动发布属于高脆弱、高风险路径，必须单独 Policy 开启，并做 Locator Release、Trace 和幂等防重。

---

## 9. 与 1000 站全历史知识库目标的差距

### 9.1 没有 Coverage Truth

“每周把相关站点再抓一遍”并不能回答：

- 这个站 2012 年有多少文章？
- Sitemap 和 Archive 是否一致？
- 哪个时间段存在缺口？
- 分页是否真的到尾？
- RSS 只保留最近 20 条时，旧文章从哪里发现？
- 某个站是否换过域名？

所以必须维持：

```text
Provider Run
 -> URL Observation
 -> URL Inventory
 -> Coverage Evidence
 -> Known Gap
```

### 9.2 没有增量版本语义

每周抓取如果没有 `ETag/Last-Modified/content hash/document version`，会出现：

- 重复做相同工作；
- 无法区分正文更新和页面模板变化；
- 无法知道当前 Markdown 对应哪个网页版本；
- 无法重放某天的知识库状态。

### 9.3 没有不可变 Artifact

如果只留下 Crawl4AI Markdown：

- Extractor 升级无法离线重跑；
- 无法复核页面当时是什么样；
- 无法定位正文为何缺失；
- Browser/HTTP 差异无法比较；
- 质量争议只能重新访问已经变化的网站。

### 9.4 没有独立 Quality Gate

AI Writer 能顺畅摘要不等于正文抽取正确。WAF 页面、登录页、soft-404、截断文章都可能“很像自然语言”，所以必须先做确定性质量检查，再允许进入知识库与 Editorial Plane。

### 9.5 没有分布式公平调度

1000 个站中会有：

- 几十篇的小博客；
- 几万篇的大站；
- Browser 重站；
- 经常 429 的站；
- RSS 高频更新站；
- 很久不更新的静态站。

如果只是单个 asyncio batch / cron，很容易让大站 Backfill 占满资源，导致 Incremental 延迟失控。因此需要 Queue Class、域名 Token Bucket、source fairness、global admission、lease/fencing 和 backpressure。

---

## 10. 推荐的生产级“虚拟编辑部”实现

将该项目的思想放入现有知识库后，推荐链路：

```text
               Collection Truth Plane

CMS/API/Sitemap/RSS/Archive/Search Gap
                |
                v
           URL Inventory
                |
                v
      HTTP/Crawl4AI/Playwright
                |
                v
          Immutable Artifact
                |
                v
   Extraction -> Quality -> Canonical IR
                |
                v
          Document Version PASS
                |
                v
             Markdown

               Editorial Plane

PASS Document Versions
        |
        v
Issue Snapshot + Candidate Selector
        |
        v
Evidence Package
   |       |       |
   v       v       v
Writer A Writer B Writer C
   \       |       /
    \      |      /
     Editorial Candidates
             |
             v
      AI Editor/Composer
             |
             v
 Citation + Structure Gate
             |
             v
       Human Review
             |
             v
       Draft Publication
             |
             v
      Explicit Publish
```

这个改造保留了作者项目最有用的“虚拟编辑部体验”，但把事实、身份、引用、抓取、模型和发布边界全部固定下来。

---

## 11. 并发、预算与失败处理

多 Writer 不能无界 fan-out。每一期应有明确预算：

```text
max_editorial_agents
max_total_input_tokens
max_total_output_tokens
max_generation_cost
per_agent_timeout
minimum_successful_candidates
```

执行状态：

```text
PENDING
 -> RUNNING
 -> SUCCEEDED | FAILED | TIMED_OUT
```

Editor 只在满足最小候选数后开始。如果某个 Writer 失败：

- 不自动无限重试；
- 允许使用其他候选继续；
- 在 Web UI 标记降级；
- 记录失败原因；
- 预算耗尽不得静默切换成更贵模型。

如果 Citation Gate 失败，稿件必须保持 `REVIEW_REQUIRED`，不能因为其它段落正常就自动发布。

---

## 12. Web 管理功能建议

针对“虚拟编辑部”，在已有 Source/Run/Document 页面之外增加 Curation 页面：

### Issue 页面

展示：

- 本期 Evidence Snapshot；
- Candidate Selector 条件；
- 被选/被排除的 Document Version；
- Manual PIN / EXCLUDE / PRIORITIZE；
- 每个 Writer 的输入、输出、耗时、Token、成本；
- Editor 输出；
- Citation Gate；
- 人工修改 diff；
- 外部 Draft/Publication 状态。

### Agent Release 页面

展示：

- Persona；
- Prompt Version；
- Model；
- LLM Provider；
- 输出 Schema；
- Golden Eval；
- 最近失败率；
- 平均成本。

### Publication 页面

展示：

- Adapter；
- Capabilities；
- external_draft_id；
- Draft/Approved/Published 状态；
- publish attempt；
- retry；
- external URL；
- 发布快照。

---

## 13. 应纳入博客知识库技术方案的优化

### 13.1 新增 LLM Provider 抽象

新增：

```text
llm_provider_release
llm_route_decision
```

Writer、Editor、Summary、Entity 等生成任务不直接依赖某一个云模型 SDK。

### 13.2 增强 Editorial Plane

新增：

```text
evidence_package
editorial_agent_release
editorial_candidate
```

并把多 Writer 定义为受控的 Perspective Worker。

### 13.3 Publication Adapter 改为能力协商

默认 `DRAFT_FIRST`，外部平台不支持稳定写 API 时退化为 Markdown/HTML Export，而不是默认 Browser 自动操作。

### 13.4 增加 Docker Compose Pilot

Pilot 与生产共享：

- PostgreSQL schema；
- Redis Streams contract；
- S3/MinIO object key；
- Run/Task state machine；
- Source/Profile/Release；
- Artifact/Version/Quality 模型。

只合并进程，不改变语义。

---

## 14. 明确不采纳的设计

### 14.1 不采纳“所有 Writer 独立抓网页”

原因：重复请求、来源快照不一致、限速失控、证据不可重放。

### 14.2 不采纳“Crawl4AI Markdown 直接 Canonical”

原因：缺少 Raw Artifact、Extractor Portfolio、Quality Gate 和可重建 IR。

### 14.3 不采纳“Persona 决定事实”

Persona 只能决定表达方式。

### 14.4 不采纳“AI Editor 输出后直接发布”

默认必须经过 Citation Gate 和 Human Review；自动发布只能作为显式策略单独开启。

### 14.5 不采纳“Pilot 用 SQLite/目录，生产再重写”

规模变化应只改变部署拓扑，不改变核心状态语义。

---

## 15. 最终架构判断

该项目最像一个成功的“内容产品原型”，而不是长期知识库爬虫。它证明了以下产品思路很有价值：

```text
Crawl/Knowledge Base
 -> 多角色 AI 观点候选
 -> AI Editor
 -> 人工精选/修改
 -> Newsletter Draft
 -> Publication
```

真正适合 1000+ 技术博客知识库的实现，应把它重新约束成：

```text
Collection Truth 与 Editorial Plane 严格分离
Evidence Package 冻结事实快照
多 Writer 只消费 Evidence，不自主扩大来源
LLM Provider 可本地/云端替换并版本化
人工 PIN 只影响某一期内容编排
Publication 默认 Draft First
Docker Pilot 与生产使用同一业务协议
```

这样既保留了原项目低成本、可玩性强、内容有“人格”的优点，又不会牺牲全历史 Coverage、增量同步、Markdown 可重建、可追溯性和 1000 站规模下的可靠性。