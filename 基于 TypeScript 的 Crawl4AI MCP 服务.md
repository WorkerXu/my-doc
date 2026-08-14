# 基于 TypeScript 的 Crawl4AI MCP 服务

## 1. 项目概述

- 项目：`omgwtfwow/mcp-crawl4ai-ts`
- 地址：https://github.com/omgwtfwow/mcp-crawl4ai-ts
- 语言：TypeScript
- 定位：把 Crawl4AI 的 HTTP 服务封装为 MCP Server，对外提供统一、强类型的抓取工具，包括 Markdown 提取、批量抓取、智能抓取、递归抓取、Sitemap 解析、链接提取、浏览器自动化、会话管理和 LLM 结构化抽取。
- 与本知识库需求的关系：它不是一个可直接承担 1000 站点长期增量同步的完整采集平台，但其“能力探测 + 统一工具契约 + HTTP/Browser 能力封装 + 递归发现 + 会话复用”的思路，很适合作为站点接入阶段的 Discovery Probe / 调试适配层。

## 2. 核心实现结构

项目代码主要分为 MCP Server、参数 Schema、Handler、Crawl4AI Service 四层。

1. MCP Server 负责注册工具、校验参数、把工具调用分发给 Handler。
2. Handler 负责把业务参数转换为 Crawl4AI REST API 请求，并统一格式化返回结果。
3. `crawl4ai-service.ts` 封装底层 HTTP 客户端、鉴权和服务访问。
4. `schemas` 与 `types.ts` 负责把每个抓取工具的输入参数显式化，避免调用方直接依赖 Crawl4AI 内部 Python API。

这种结构的价值不在 MCP 本身，而在于把“抓取引擎能力”变成稳定的契约层。对长期知识库平台来说，抓取引擎可以替换，但上层站点配置、任务状态、质量审计和 Web 管理不应跟着底层库变化。

## 3. `smart_crawl` 的技术原理

`smart_crawl` 先通过 URL 特征和 HEAD 请求识别内容类型，再决定后续处理方式。

主要逻辑：

- URL 包含 `sitemap` 或以 `.xml` 结尾时优先判断为 Sitemap；
- URL 包含 `rss` 或 `feed` 时判断为 RSS；
- HEAD 的 `Content-Type` 用于补充识别 text/xml/json；
- 随后统一调用 Crawl4AI `/crawl` 接口；
- 当识别为 Sitemap/RSS/XML 且允许 follow links 时，从返回内容中解析链接，再批量抓取一小批子 URL。

这个设计体现出一个重要原则：**站点入口不应该在录入时就要求人工精确分类，系统应先做能力探测，再把探测结果变成候选配置。**

但原项目实现更偏交互式工具，不适合直接作为生产级全量发现器：

- Sitemap/RSS 链接通过正则表达式提取，面对 namespace、Sitemap Index、CDATA、Atom、异常 XML 时不可靠；
- follow link 只取有限数量 URL，适合预览，不等于历史全量；
- 内容类型判断主要依赖 URL 和 HEAD，缺少 body sniffing、重定向后 URL、真实 XML root element 等证据；
- 没有持久化 Provider checkpoint、coverage、断点和断档状态。

因此在本方案中应保留“自动能力探测”思想，但实现为独立的 `Discovery Probe`，并使用正式 XML/Feed parser，把探测结果写入 PostgreSQL，之后由 durable job 执行真正的全量回灌。

## 4. `crawl_recursive` 的技术原理

项目的递归抓取使用一个典型的 BFS frontier：

- `visited` 集合防止重复；
- `toVisit` 队列保存 URL 和 depth；
- 通过 `max_depth` 与 `max_pages` 进行有界控制；
- 每页抓取完成后读取 Crawl4AI 返回的 internal links；
- 只接受与起始 URL hostname 相同的链接；
- include/exclude 正则用于过滤 URL；
- 单个页面失败不会中断整个递归过程。

该实现非常适合作为“接入时结构探测”与“小规模站点回归测试”，但不能直接扩展成 1000 站点生产 frontier，原因是：

1. frontier 和 visited 仅存在进程内存，进程重启即丢失；
2. 没有 URL lease、任务幂等、失败重试次数和下一次执行时间；
3. 没有 domain-level rate limit 和跨站公平调度；
4. hostname 相同并不等于 URL 合法，仍需 canonical、allow/deny、robots、路径策略；
5. 没有 canonical 去重、redirect identity、内容版本与 coverage 统计。

本方案应将 BFS 的“同域、深度、页面上限、过滤器”保留为 Discovery Provider 的能力，但 frontier 必须落在 PostgreSQL durable table 中，并通过 Redis Streams 仅做任务分发。

## 5. `batch_crawl` 的实现启示

`batch_crawl` 支持两种模式：

- 一组 URL 共用 crawler config；
- 每个 URL 使用独立 config。

第二种方式对异构博客非常重要。1000 个技术博客不可能只靠一套统一抓取参数，必须允许同一批任务中的不同站点使用各自的 Browser/Profile/Extraction Contract。

值得吸收的设计：

- “批次”只负责调度聚合，不强迫所有 URL 使用同一配置；
- 抓取结果可以携带处理时间、内存峰值等服务侧指标，便于做成本与容量分析。

在本方案中应进一步演进为：

- 每个 fetch task 固化 `profile_release_id`、`contract_release_id`、`action_plan_release_id`；
- worker 按 domain / transport / cost class 分组批处理；
- Browser 与 HTTP 使用不同资源池；
- 指标写入 crawl_attempt 和 OpenTelemetry，而不是只返回给调用者。

## 6. 会话管理与动态站点

项目提供 `manage_session`，允许创建、列出和清理 session，并在 `crawl` 中通过 `session_id` 复用浏览器会话。

其意义是：某些动态博客、文档站、SPA 的分页、加载更多、路由状态需要跨请求保持浏览器状态。

生产化时不能把 session 只存在 MCP 进程内存中，而应该：

- 使用 `browser_session_lease` 记录 session 与 worker 的绑定；
- session 设置 TTL、最大请求数、最大生命周期；
- Action Plan 明确是否允许会话复用；
- worker 崩溃后 session 自动失效并重建；
- cookie、localStorage、token 等敏感状态默认不持久化，确有需要时单独加密并受访问策略控制。

## 7. 参数契约化的价值

项目把浏览器类型、viewport、headers、cookies、proxy、JS、wait condition、滚动、iframe、cache、CSS selector、Markdown generator、LLM extraction 等能力显式暴露为工具参数。

对知识库平台的启发是：**不要把站点特例写成散落在 worker 代码里的 if/else。**

应将这些能力拆成三类版本化配置：

1. `Fetch Profile`：HTTP/Browser、UA、headers、timeout、cache、代理和资源预算；
2. `Browser Action Plan`：wait、scroll、click、pagination、JS 等页面动作；
3. `Extraction Contract`：CSS/DOM/JSON/正文抽取与 metadata 映射。

这样站点规则变更时可以灰度、回滚，并对历史 snapshot 重新抽取，而无需重新请求源站。

## 8. 不应直接照搬的实现

### 8.1 XML 与 Sitemap 不应使用正则解析

项目为了工具轻量化使用正则读取 `<loc>` 和 RSS link，这在生产系统中风险较高。应使用流式 XML parser，并正式支持：

- Sitemap Index；
- gzip Sitemap；
- namespace；
- Atom/RSS/JSON Feed；
- lastmod；
- 超大 Sitemap 分片；
- URL 去重和规范化。

### 8.2 `HEAD` 只能是探测信号之一

很多站点不支持 HEAD、HEAD 与 GET 行为不同、CDN Content-Type 错误。Discovery Probe 应综合：

- 最终 URL；
- Content-Type；
- body magic/root element；
- HTML `<link rel="alternate">`；
- robots.txt 中 Sitemap；
- 常见 `/sitemap.xml`、`/feed`、`/rss.xml`；
- 页面结构和链接密度。

### 8.3 单机内存 frontier 不可用于长期任务

全量历史和增量同步需要可恢复状态，因此 URL frontier、checkpoint、retry、lease、coverage 必须持久化。

### 8.4 MCP 不应成为生产调度真相源

MCP 适合作为调试、运维和智能体访问接口，但生产抓取的任务、状态和幂等必须由 Control Plane + PostgreSQL + durable queue 管理。MCP 调用最多创建 job 或查询结果，不能绕过业务状态机直接改变采集状态。

## 9. 对现有博客知识库技术方案的优化建议

本项目带来的最有价值新增能力是 **Discovery Probe / Capability Probe（站点能力探测）**。

建议在已有 Source Discovery 前增加一个明确的接入探测阶段：

1. 用户只需输入 root URL；
2. Probe 同时探测 robots Sitemap、常见 Sitemap、Feed、HTML alternate、归档页、公开 API 和动态列表；
3. 对候选入口执行有界 sample crawl；
4. 自动判断 `provider_type`、`capability`、是否需要 Browser、是否需要 session、文章 URL pattern 候选；
5. 自动生成 `site_source`、Fetch Profile、Extraction Contract、Action Plan 草稿；
6. Web 管理端展示探测证据和样本，让管理员确认后发布；
7. 发布后才创建 durable backfill job。

这一层能显著降低未来持续新增网站的接入成本，又不会把“自动探测”误当成“全量完成”。

## 10. 最终结论

`mcp-crawl4ai-ts` 的最佳借鉴点不是把 MCP 放进核心采集链路，而是把抓取能力抽象成稳定工具契约，并在入口处提供自动内容类型识别、批量配置、递归发现和浏览器会话能力。

针对 1000+ 技术博客知识库，应该吸收这些思想并生产化为：

- Discovery Probe；
- 版本化 Fetch Profile / Action Plan / Extraction Contract；
- 按 URL 固化配置的异构批处理；
- durable recursive frontier；
- Browser session lease；
- MCP 仅作为可选的只读查询与运维调试入口。

原项目的正则 XML 解析、有限 follow links、进程内 frontier 和 session 状态不应直接用于生产级历史全量与长期增量同步。