# tldraw 项目与落地实践存档

存档时间：2026-08-23

> 仅保存本轮已经确认的项目介绍、落地文章、GitHub 实践项目和平台内容。优先保留具有实现细节、架构方案、部署/复现步骤、源码或真实案例证据的内容。纯概念介绍、泛工具盘点不作为核心落地结果。
>
> GitHub 检索说明：本轮已完成 gh-reader 项目索引检索，并通过全网/Exa 补到部分 GitHub 项目；但此前没有完整单独执行 `site:github.com tldraw` 及所有扩展字面查询，因此本文不声称穷尽 GitHub。

## 一、项目介绍

### tldraw/tldraw

- 项目名称：tldraw/tldraw
- 链接：https://github.com/tldraw/tldraw
- GitHub Star：49,903（本轮项目索引结果）
- 最近更新时间：2026-08-22
- 创建时间：2021-05-09
- 定位：用于 React 的无限画布 SDK，可作为白板、流程图、节点编辑器、AI Canvas、Workflow Builder、可视化协作工具等产品的底层画布基础设施。
- 主要技术栈：React、TypeScript、DOM/SVG 渲染、tldraw Editor API、响应式 Store、Shape/Tool/Binding 扩展体系、`@tldraw/sync`、WebSocket、Cloudflare Durable Objects（官方多人协作模板）。
- 核心能力：
  - 无限画布、平移、缩放、框选、吸附、撤销重做。
  - 压感手绘、几何图形、富文本、箭头、图片、视频和网页嵌入。
  - 自定义 Shape、Tool、Binding、UI、事件和副作用。
  - Runtime Editor API，可由程序或 Agent 直接驱动画布。
  - `@tldraw/sync` 多人实时协作，可自托管。
  - AI 集成能力，包括 Agent、Workflow、Chat、Branching Chat、Image Pipeline 等 Starter Kit。
- 官方快速开始：`npm i tldraw`，或 `npx create-tldraw@latest`。
- 已知真实使用方：Google、Shopify、BlackRock、Autodesk、ClickUp、Replit、Google Stitch、Luma、Runway、Padlet、Mobbin、Jam、Craft、Honeycomb、SchoolAI 等。
- 许可证注意：当前主 SDK 的生产环境使用需要 license key；Starter Kit 的许可应按具体仓库/版本再次核对。官方 README 本轮明确说明生产使用需要许可证。
- 与主题关联性：直接主体。
- 可验证实现：是。
- 源码：是。
- 部署/复现：是。
- 真实案例：是。

---

## 二、落地文章 / 官方工程实践

### 1. Multiplayer Starter Kit

- 标题：Multiplayer starter kit
- 链接：https://tldraw.dev/starter-kits/multiplayer
- 来源：tldraw 官方文档
- 日期：当前官方文档，本轮未稳定返回单独发布日期
- 摘要：官方生产级多人协作参考实现，用于构建自托管的 tldraw 实时协作应用。
- 实现方案 / 架构：
  - 每个协作房间由一个 Cloudflare Durable Object 承载，保证每个房间只有一个权威实例。
  - 客户端通过 WebSocket 连接房间；变更进入 Durable Object 后更新内存文档并广播给其他客户端。
  - `TLSocketRoom` 负责房间内同步逻辑。
  - `SQLiteSyncStorage` 使用 Durable Object 内置 SQLite 自动持久化房间状态。
  - 图片、视频等大文件存入 Cloudflare R2，而不是通过 WebSocket 同步。
  - 支持 WebSocket hibernation、会话恢复、实时光标和 Presence。
  - 自定义 Shape 需要客户端和服务端同时注册 Schema / Shape Utils，避免验证和版本迁移失败。
  - 提供 `worker/TldrawDurableObject.ts`、`client/pages/Room.tsx`、`client/multiplayerAssetStore.tsx`、`wrangler.toml` 等工程文件。
- 部署 / 复现：有，支持 Cloudflare Worker + Durable Objects + R2 部署。
- 热度 / 证据：官方 Starter Kit；官方说明属于 production-ready foundation，并与 tldraw.com 多人协作栈同源。
- 与 tldraw 关联性：直接官方实现，极高。
- 可验证实现：是。
- 源码：是。
- 部署步骤：是。
- 真实案例：是（与 tldraw.com 架构同源）。

### 2. tldraw sync

- 标题：tldraw sync
- 链接：https://tldraw.dev/docs/sync
- 来源：tldraw 官方文档
- 日期：当前官方文档
- 摘要：tldraw 的实时多人共享文档同步体系，官方在 tldraw.com 生产环境使用。
- 实现方案 / 架构：
  - 前端使用 `useSync` / `@tldraw/sync` 建立同步 Store。
  - 后端使用 `@tldraw/sync-core` 的 `TLSocketRoom` 管理每个文档房间。
  - `TLSocketRoom` 负责权威文档、WebSocket 多客户端通信和持久化钩子。
  - 支持 `InMemorySyncStorage` 与推荐的 `SQLiteSyncStorage`。
  - 自定义 Shape / Binding 必须在服务端 `createTLSchema` 和客户端同步注册。
  - 支持自定义记录类型、评论 / 反应对象、权限和 object-store lane。
  - 明确指出生产部署时客户端与服务端 tldraw 版本应协调升级。
- 部署 / 复现：有；官方推荐 Cloudflare 模板，也可接入自己的 Node / Bun / JavaScript WebSocket 后端。
- 与 tldraw 关联性：直接官方核心同步能力。
- 可验证实现：是。
- 源码：是。
- 部署步骤：是。
- 真实案例：是（tldraw.com）。

### 3. Agent Starter Kit

- 标题：Agent starter kit
- 链接：https://tldraw.dev/starter-kits/agent
- 来源：tldraw 官方文档
- 日期：当前官方文档
- 摘要：完整演示 AI Agent 如何理解并操纵 tldraw Canvas，不只是文本聊天，而是直接创建、修改、删除、排列画布元素。
- 实现方案 / 架构：
  - Agent 上下文同时包含当前画布截图和结构化 Shape 表示。
  - 支持创建、更新、删除 Shape，以及绘图、旋转、缩放、对齐、分布、堆叠和排序等高级动作。
  - Action System 为每种画布动作提供独立的 validation、execution 和 chat-panel presentation。
  - Streaming Action：模型响应生成过程中逐步更新画布，而不是等待整次回复结束。
  - Agent 使用 Managers 分离 chat history、模型选择、上下文 Shape、todo、mode 等职责。
  - Mode System 控制不同阶段能使用的上下文和动作，可实现 working / reviewing / planning 等不同角色。
  - 支持 OpenAI、Anthropic、Google 等模型提供方。
  - 支持 `agent.prompt()`、`agent.request()`、`agent.cancel()`、`agent.reset()` 等程序化调用。
  - 自定义 Shape 可扩展 FocusedShape schema 和转换函数，让 Agent 读写自定义业务属性。
- 部署 / 复现：有完整 Starter Kit 源码。
- 与 tldraw 关联性：直接官方实现，极高。
- 可验证实现：是。
- 源码：是。
- 部署 / 复现：是。

### 4. AI Integrations

- 标题：AI integrations
- 链接：https://tldraw.dev/docs/ai
- 来源：tldraw 官方文档
- 日期：当前官方文档
- 摘要：官方总结 tldraw 与 AI 的三种主要落地模式：Canvas as Output、Visual Workflows、AI Agents。
- 实现方案：
  - Canvas as Output：把 AI 生成的图片、图表、网站等内容呈现在无限画布上。
  - Visual Workflow：Shape 表示操作 / 数据源，Binding 表示连接，形成可执行数据流。
  - AI Agent：LLM 通过 Editor API 读取并修改画布。
  - Agent 结合截图和结构化 Shape 数据理解视觉和精确属性。
- 与 tldraw 关联性：直接官方架构说明。
- 可验证实现：是，关联 Agent / Workflow Starter Kit。

### 5. Mermaid → tldraw Shapes / Interactive Pipeline

- 标题：Customize Mermaid diagrams
- 链接：https://tldraw.dev/examples/custom-shape-mermaids
- 来源：tldraw 官方示例
- 日期：当前官方示例
- 摘要：把 Mermaid Flowchart 转换成原生 tldraw Shape，而不是只嵌一张 SVG；转换后的节点可以继续拖拽、编辑、连接并参与业务逻辑。
- 实现方案 / 架构：
  - 使用 `@tldraw/mermaid` 将 Mermaid 语义解析与布局映射到 Canvas。
  - 通过 `blueprintRender.mapNodeToRenderSpec` 将 Mermaid 顶点映射到自定义 Shape。
  - 导入后从 tldraw Arrow Binding 重新提取 DAG，而不是仅依赖 Mermaid 源文本。
  - Pipeline 执行使用 Kahn layer 计算步骤层次，并支持 AND-join 语义。
  - 示例模拟 CI/CD Pipeline 的节点运行、失败与重试。
- 与 tldraw 关联性：直接官方工程实现。
- 可验证实现：是。
- 源码：是。
- 复现步骤：是。

### 6. tldraw MCP App：让 Agent 直接画图

- 标题：tldraw MCP App: Letting your agents draw
- 链接：https://tldraw.dev/blog/tldraw-mcp-app
- 来源：tldraw 官方博客
- 日期：2026-03-03
- 摘要：将 tldraw Canvas 作为 MCP App 暴露给外部 Coding / AI Agent，使 Agent 可以创建、编辑、删除图形，并与用户在同一个 Canvas 上共同迭代。
- 实现方案 / 架构：
  - Canvas 状态通过 MCP / 工具接口提供给 Agent。
  - Agent 主要使用 create / edit / delete 等结构化动作操作画布。
  - 用户手工修改后，最新 Canvas State 再反馈给 Agent，形成共享视觉工作区。
  - 适合架构图、流程图、产品流程、代码仓库可视化等 Agent 工作流。
- 与 tldraw 关联性：直接官方落地。
- 可验证实现：是。
- 源码 / 工具调用证据：是。

### 7. Building with tldraw offline

- 标题：Building with tldraw offline
- 链接：https://tldraw.dev/blog/building-with-tldraw-offline
- 来源：tldraw 官方博客
- 日期：2026-07-22
- 摘要：使用 tldraw Offline / Local-first 方式将 Canvas 作为本地 Agent 和 JavaScript 应用运行环境，展示多种可落地 Canvas App。
- 实践案例：数据可视化、Presentation、Kanban、Palette Extractor、Task Tracker、WebLLM Local Model Playground 等。
- 实现价值：证明 tldraw 不只是白板，可作为本地 AI / 可视化应用壳层。
- 与 tldraw 关联性：直接官方实践。
- 可验证实现：是。
- 真实案例：是。

### 8. ClickUp Whiteboards

- 标题：How ClickUp rebuilt Whiteboards with tldraw
- 链接：https://tldraw.dev/blog/clickup
- 来源：tldraw 官方 Case Study
- 日期：2025-04-09
- 摘要：ClickUp 将原有 Whiteboards 重构到 tldraw SDK 上，用更可扩展的画布基础设施替换旧实现。
- 实现方案：
  - 复用 tldraw 的快捷键、Context Menu、Frame 等编辑器能力。
  - 使用 Custom Shape 将 ClickUp 自身 Task、Doc 等产品对象直接放到 Canvas。
  - 自定义 Floating Toolbar 和交互 UI，使 tldraw 视觉与 ClickUp 主产品一致。
  - 继续在 Canvas 上扩展 AI 生成功能，并探索将白板内容转成完整 ClickUp Project。
- 热度 / 真实规模：ClickUp 面向 1000 万+ 用户；案例说明 Whiteboards 2024-12 重发后使用度明显增长。
- 与 tldraw 关联性：直接商业产品生产案例。
- 可验证实现：真实案例证据明确。
- 源码：产品源码未公开。
- 生产落地：是。

### 9. Padlet Sandbox

- 标题：How Padlet shipped Sandbox with tldraw
- 链接：https://tldraw.dev/blog/padlet
- 来源：tldraw 官方 Case Study
- 日期：2025-04-09
- 摘要：Padlet 为教育场景打造 Sandbox 白板产品，选择 tldraw 作为 Canvas 基础。
- 实现方案：
  - Padlet 原产品是 Vue，而 tldraw 是 React，团队实现 Vue / React 桥接。
  - 使用 Custom Shape 复用 Padlet Boards 原有内容 Block。
  - 使用 Camera Control 将无限画布改造成固定视图 / 分页式课堂体验。
  - 增加 Tabs，把 Canvas 组织成类似 Slideshow 的课堂页。
  - 接入 tldraw sync 迁移多人协作后端。
  - 后续加入 Speech Bubble、Connector、PDF Export、内容安全审核 Safety Net。
- 热度 / 真实规模：10 周上线；每月 40 万+ 用户使用 Sandbox、每月创建 14 万+ Sandbox，超过 10% 的新 Padlet 为 Sandbox。
- 与 tldraw 关联性：直接商业产品生产案例。
- 可验证实现：是。
- 生产落地：是。

### 10. Mobbin 内部工具

- 标题：How Mobbin shipped two internal tools in three months with the tldraw SDK
- 链接：https://tldraw.dev/blog/mobbin
- 来源：tldraw 官方 Case Study
- 日期：2025-04-09
- 摘要：Mobbin 用 tldraw 替代难以适配其业务流程的传统 CMS，构建定制视觉内部工具。
- 实现方案 / 业务场景：
  - ML 团队使用 Canvas 标注 UI 元素、管理训练数据、做模型 Benchmark。
  - Content 团队使用另一套 Canvas 工具管理、标记和整理数千个设计参考素材。
  - 依赖 tldraw 的 pan / zoom / selection 和 Custom Shape API 快速实现业务操作。
- 热度 / 真实规模：3 名工程师，不到 3 个月交付两套完整内部工具。
- 与 tldraw 关联性：直接生产案例。
- 可验证真实案例：是。

### 11. Jam Screenshot Annotations

- 标题：How Jam refreshed screenshot annotations with tldraw
- 链接：https://tldraw.dev/blog/jam
- 来源：tldraw 官方 Case Study
- 日期：2025-04-09
- 摘要：Jam 用 tldraw 重做截图标注编辑器，避免自行重建绘图、撤销重做、复制粘贴、导出等复杂编辑器能力。
- 实现方案：
  - 复用 tldraw 内置绘制与基础 Shape。
  - 复用 Undo / Redo、Copy / Paste、Image Export。
  - 继续使用 Jam 自己的 Toolbar，通过 Editor API 驱动 tldraw。
  - 使用 Custom Shape 实现 Blur Element，用于遮盖截图中的敏感信息。
- 热度 / 真实规模：1 名开发者 24 小时完成首个原型；6 周后向全量用户发布。
- 与 tldraw 关联性：直接生产案例。
- 可验证真实案例：是。

### 12. Jam Video Annotations

- 标题：How We Built Video Annotations with tldraw
- 链接：https://jam.dev/blog/how-we-built-video-annotations-w-tldraw/
- 来源：Jam 工程团队
- 日期：2025-01-17
- 摘要：Jam 工程师从第三方一方视角讲解将 tldraw 扩展到视频 Annotation 的技术实现和架构选择。
- 实现价值：讨论第三方 Library 接入、React/SVG 架构选择、从截图标注扩展到视频标注的工程取舍。
- 与 tldraw 关联性：直接第三方实现。
- 可验证真实案例：是。

### 13. Remotion + tldraw 视频编辑器探索

- 标题：Remotion + Tldraw = Video editor without the hassle?
- 链接：https://bonviddy.substack.com/p/remotion-tldraw-video-editor-without
- 作者：Ziyad
- 日期：2024-06-12
- 摘要：尝试把 tldraw 作为成熟的可视化编辑 / 对象布局交互层，把 Remotion 用作视频生成 / 渲染层，从而避免从零开发视频编辑器。
- 实现方案：利用 tldraw 已有的图形对象定位、拖拽和编辑能力，构建基于 Block 的视频 Composition Editor。
- 与 tldraw 关联性：直接第三方架构探索。
- 可验证实现：有明确产品 / 技术方案，但属于探索型而非已确认大规模生产案例。

### 14. Docker + cpolar 异地访问实践

- 标题：详细介绍：异地也能一起画图？Tldraw+cpolar 实现跨空间协作
- 链接：https://www.cnblogs.com/clnchanpin/p/19395628
- 来源：博客园
- 日期：2025-12-25
- 摘要：通过第三方 Docker 镜像运行 tldraw，再用 cpolar 将本地端口映射到公网。
- 部署步骤：`docker run -d --name tldraw -p 7900:3000 wbsu2003/tldraw`，再配置 cpolar HTTP Tunnel 和固定二级域名。
- 注意：这是第三方镜像 / 公网暴露实践，不等同于官方 `@tldraw/sync` + Durable Objects 的生产多人协作后端。
- 与 tldraw 关联性：直接部署，但工程参考价值中等。
- 可验证部署步骤：是。

---

## 三、GitHub 项目

### tldraw/tldraw-sync-cloudflare

- 项目：https://github.com/tldraw/tldraw-sync-cloudflare
- Star：135
- 最近更新时间：2026-08-05
- 创建时间：2024-07-15
- 简介：官方生产级 tldraw sync Cloudflare 后端模板。
- 技术栈：TypeScript、`@tldraw/sync-core`、Cloudflare Workers、Durable Objects、WebSocket、SQLite、R2。
- 落地能力：
  - 每个房间一个 Durable Object。
  - `TLSocketRoom` 提供实时同步与冲突处理。
  - SQLite 自动持久化。
  - R2 保存图片 / 视频等 Assets。
  - Bookmark URL unfurling。
  - 支持 WebSocket hibernation 与恢复。
- 部署 / 复现：`yarn`、`yarn dev`，生产 `yarn build` + `yarn wrangler deploy`；需 Cloudflare 账号和 R2 Bucket。
- 官方说明：与 tldraw.com 大规模多人协作系统同类架构；仓库说明单房间约可处理 50 名同时协作者。
- 关联性：极高，官方直接项目。
- 源码：是。
- 部署步骤：是。
- 真实案例：是。

### tldraw/tldraw-offline

- 项目：https://github.com/tldraw/tldraw-offline
- Star：312
- 最近更新时间：2026-07-20
- 创建时间：2024-12-28
- 简介：官方本地文件 / 离线桌面 tldraw 应用。
- 落地能力：Local-first 白板、桌面文件工作流、适合本地 Agent 和隐私敏感场景。
- 关联性：极高。
- 源码：是。

### tldraw/obsidian-plugin

- 项目：https://github.com/tldraw/obsidian-plugin
- Star：442
- 最近更新时间：2026-08-12
- 创建时间：2023-07-18
- 简介：将 tldraw 集成进 Obsidian。
- 主要场景：知识管理、笔记内无限画布、可视化思考。
- 关联性：直接官方插件。
- 源码：是。

### tldraw/agent-template

- 项目：https://github.com/tldraw/agent-template
- Star：31
- 最近更新时间：2026-08-05
- 创建时间：2025-09-17
- 简介：让 AI Agent 理解并与 Canvas Drawing / Elements 交互。
- 主要能力：Canvas Context、Agent Actions、Shape 操作、Mode / Manager 架构。
- 场景：AI 绘图助手、架构图 Agent、视觉协作 Agent。
- 关联性：极高。
- 源码：是。
- 可复现：是。

### tldraw/workflow-template

- 项目：https://github.com/tldraw/workflow-template
- Star：16
- 最近更新时间：2026-08-05
- 创建时间：2025-09-17
- 简介：拖拽节点和连接线的可视化工具模板。
- 主要能力：Custom Node、Port、Binding、连线交互和 Workflow 基础。
- 场景：n8n / Dify / Langflow / ComfyUI 风格的自动化、AI Workflow、Visual Programming、No-code。
- 关联性：极高。
- 源码：是。

### tldraw/chat-template

- 项目：https://github.com/tldraw/chat-template
- Star：6
- 最近更新时间：2026-07-15
- 简介：把草图、图片和标注作为 LLM Chat 的视觉上下文。
- 场景：AI Chat + Canvas、截图批注问答、设计 / 产品讨论。
- 关联性：直接官方模板。
- 源码：是。

### tldraw/branching-chat-template

- 项目：https://github.com/tldraw/branching-chat-template
- Star：15
- 最近更新时间：2026-08-05
- 创建时间：2025-09-17
- 简介：在无限画布上构建分支式 AI 对话树。
- 实现能力：Message Node、自定义 Shape、Connection Binding、AI Streaming、根据节点关系回溯上下文。
- 场景：分支对话、Conversation Design、Storytelling、复杂 Multi-turn AI UI。
- 关联性：直接官方模板。
- 源码：是。

### mrslimslim/gpt-image-canvas

- 项目：https://github.com/mrslimslim/gpt-image-canvas
- Star：690
- 最近更新时间：2026-07-04
- 简介：Local professional AI canvas built with tldraw。
- 场景：AI 图片生成 / 编辑无限画布。
- 关联性：高，明确基于 tldraw。
- 源码：是。

### Agents365-ai/tldraw-skill

- 项目：https://github.com/Agents365-ai/tldraw-skill
- Star：87
- 最近更新时间：2026-07-16
- 简介：用自然语言生成 tldraw `.tldr` 图表，支持 6 类图表预设并提供视觉自检。
- 主要能力：架构图、流程图、序列图、ML / DL 图、ER 图、UML；PNG / SVG 导出；多 Agent；视觉自检和多轮修正。
- 场景：Codex / Claude Code 等 Coding Agent 自动生成专业技术图。
- 关联性：高，直接操作 tldraw 文档。
- 源码：是。

### jananadiw/codex-tldraw-mcp

- 项目：https://github.com/jananadiw/codex-tldraw-mcp
- Star：21
- 最近更新时间：2026-08-05
- 创建时间：2026-06-14
- 简介：本地 Codex MCP Server，扫描代码仓库并生成 tldraw 产品 / Workflow 图。
- 场景：代码仓架构可视化、产品工作流图、Codex + Canvas。
- 关联性：高。
- 源码：是。

### atpaawej/openboard

- 项目：https://github.com/atpaawej/openboard
- Star：2
- 最近更新时间：2026-08-17
- 创建时间：2026-08-14
- 简介：Local-first personal whiteboard workspace for developers and external AI agents。
- 技术栈：tldraw + MCP stdio / SSE + SQLite。
- 场景：本地开发者白板、AI Agent 共享可视化 Workspace。
- 关联性：高。
- 源码：是。

### devjaw/Rishah

- 项目：https://github.com/devjaw/Rishah
- Star：30
- 最近更新时间：2026-07-11
- 简介：Tauri + tldraw 的原生桌面白板应用。
- 场景：桌面离线白板、跨平台桌面 Canvas。
- 关联性：高。
- 源码：是。

### Gaigen/Mutty

- 项目：https://github.com/Gaigen/Mutty
- Star：2
- 最近更新时间：2026-08-07
- 创建时间：2026-02-17
- 简介：Self-hosted voice communication platform，集成实时房间、tldraw 协作白板、AI Bot、屏幕共享。
- 场景：远程会议、语音协作 + 白板 + AI。
- 关联性：高。
- 源码：是。

### powerpratik/notetaker-canvas

- 项目：https://github.com/powerpratik/notetaker-canvas
- Star：2
- 最近更新时间：2026-07-28
- 创建时间：2026-07-28
- 简介：iPad-first tldraw 无限画布 + AI Study Partner，AI 可查看白板、聊天并在白板上绘制。
- 场景：教育、学习助手、人机共画。
- 关联性：高。
- 源码：是。

### kitschpatrol/tldraw-cli

- 项目：https://github.com/kitschpatrol/tldraw-cli
- Star：40
- 最近更新时间：2026-07-23
- 创建时间：2024-01-14
- 简介：CLI / TypeScript Library，将 `.tldr` Sketch 导出为 PNG 或 SVG。
- 场景：自动化导出、构建流水线、文档生成。
- 关联性：直接 tldraw 文件处理。
- 源码：是。

### kitschpatrol/unplugin-tldraw

- 项目：https://github.com/kitschpatrol/unplugin-tldraw
- Star：3
- 最近更新时间：2026-07-07
- 简介：构建工具插件，可模块化导入本地 `.tldr` 文件并自动转换为 SVG / PNG。
- 场景：文档站、静态站、前端构建流程中嵌入 tldraw 图。
- 关联性：直接。
- 源码：是。

### do-once/tldraw-cli（全网 / Exa 补充）

- 项目：https://github.com/do-once/tldraw-cli
- Star：本轮未通过项目索引稳定返回，未记录未验证数值。
- 简介：命令行驱动 tldraw Canvas 的本地工具，允许 LLM / Code Agent 通过 CLI + HTTP JSON-RPC 创建图形、读取 Canvas 状态和跟踪变更。
- 架构：CLI / Host / Browser Runtime 三端分离，通过 JSON-RPC 通信。
- 关键命令：
  - `tldraw-cli canvas snapshot`
  - `tldraw-cli canvas diff --since <revision>`
  - `tldraw-cli command apply`
  - `undo` / `redo`
- Agent 能力：提供 Skill 安装，让 Claude Code / Agent 读取最新 Canvas Diff 并继续编辑。
- 典型人机协作：Agent 画架构图 → 人在浏览器拖动 / 修改 → Agent 读取 diff → Agent 基于最新状态继续迭代。
- 文档：包含 Architecture、Local Development、Extensibility 和详细 Implementation Spec。
- 关联性：极高。
- 源码：是。
- 可复现：是。

---

## 四、知乎

### 41.6K Star前端!!!这款Web端开源白板工具,无限画布+实时协作,太6了!

- 链接：https://zhuanlan.zhihu.com/p/1949591137829490690
- 作者：人生海海路
- 时间：2025-09-12
- 摘要：提供 React 基础接入、本地持久化和多人协作示例代码。
- 实现细节：
  - `<Tldraw persistenceKey="your-project-key" />` 实现浏览器本地持久化。
  - 安装 `@tldraw/sync`，通过 `useSyncDemo({ roomId })` 实现快速多人协作 Demo。
- 关联性：直接教程。
- 可验证代码：是。
- 生产部署：否，`useSyncDemo` 仅适合原型。

### 4.7 万 star 的画布引擎 tldraw：源码全公开，商用为何还收 6000 刀/年

- 链接：https://zhuanlan.zhihu.com/p/2047290647195940150
- 作者：智能时代蛮子
- 时间：2026-06-08
- 摘要：偏源码架构和许可分析，给出了较具体的源码阅读路径。
- 实现 / 源码阅读路径：
  - `packages/state`：自研 reactive signals。
  - `packages/store`：响应式 Record Store / index。
  - `packages/editor/src/lib/editor/Editor.ts`：编辑器命令中枢。
  - `ShapeUtil.ts`：Shape 插件模型。
  - `StateNode.ts`：Tool 状态机。
  - `packages/sync-core/src/lib/TLSyncClient.ts`：多人同步实现。
- 关联性：直接源码分析。
- 可验证实现：是，但具体判断需结合官方源码核验。

### 4.7万Star的tldraw，一个SDK搞定所有画布应用

- 链接：https://zhuanlan.zhihu.com/p/2045899970033849961
- 作者：知识摆渡人
- 时间：2026-06-04
- 摘要：围绕 Agent、Workflow、Chat、Image Pipeline、Branching Chat、Multiplayer 等 Starter Kit 介绍实际二开方向。
- 关联性：直接。
- 实现细节：中等，更多是官方 Starter Kit 的结构化解读。

### tldraw，会画画就能做网站？

- 链接：https://zhuanlan.zhihu.com/p/699862265
- 作者：什么玩yeah
- 时间：2024-05-26
- 摘要：介绍 tldraw Make Real 和 React 集成，用画布草图驱动 AI 生成应用 / 界面。
- 关联性：直接。
- 实现细节：中等。

### Docker部署Tldraw：免费手绘白板，团队协作神器！

- 链接：https://zhuanlan.zhihu.com/p/1921715361914544652
- 作者：二冰
- 时间：2025-07-01
- 摘要：通过第三方 Docker 镜像部署 tldraw。
- 示例：`wbsu2003/tldraw`，端口映射 `7900:3000`。
- 关联性：直接部署。
- 注意：第三方镜像，不等同官方 tldraw sync 生产后端。

---

## 五、掘金

> 本轮 Tavily 独立掘金连接器未暴露，因此使用 `site:juejin.cn` 限定搜索和 Exa / 全网补充；这里只保存已经确认的结果。

### 『NAS』在绿联部署一个白板工具-tldraw

- 链接：https://juejin.cn/post/7605492906905223214
- 时间：2026-02-12
- 摘要：在绿联 NAS 中通过 Docker 部署 tldraw，配置端口并直接浏览器访问。
- 实现 / 部署：使用 `ratneo/tldraw` Docker 镜像；示例 NAS 端口为 39445；支持导出 SVG / PNG / JSON。
- 关联性：直接部署。
- 可复现步骤：是。
- 注意：第三方 Docker 镜像。

### 基于草图或界面截图，快速生成可交互代码——tldraw、v0 与 GPTs 使用方式与体验（下）

- 链接：https://juejin.cn/post/7330442541439270912
- 时间：2024-02-01
- 可见热度：约 1,553 阅读（本轮检索结果）
- 摘要：对 tldraw Make Real 进行实际体验，并与 v0 / GPTs 等 AI UI 生成方式比较。
- 场景：草图 / 界面截图 → AI → 可交互代码。
- 关联性：直接实践。

### GitHub Daily · 第08期｜tldraw - 无限画布 React SDK 深度解析

- 链接：https://juejin.cn/post/7629565459172261942
- 时间：本轮搜索显示 2026 年结果，未稳定返回精确日期
- 摘要：偏 tldraw SDK 深度解析。
- 关联性：直接。
- 实现细节：中等，需结合正文进一步核验。

---

## 六、Bilibili

### 30K stars的开源白板工具tldraw安装及上手体验 | 开源效率工具

- 链接：https://www.bilibili.com/video/BV1R94y1N7KT
- UP：约会开源
- 时长：03:10
- 可见热度：15,385 播放、229 赞、58 币、717 收藏、65 分享（本轮读取）
- 摘要：tldraw 安装、上手和产品体验。
- 关联性：直接。
- 复现价值：中等。

### tldraw画布智能体实战：多AI协作者如何高效完成复杂任务？ | AI Engineer

- 链接：https://www.bilibili.com/video/BV1xpR4B9E31
- UP：量子菠萝_
- 时长：19:54
- 可见热度：203 播放、5 赞、4 收藏（本轮读取）
- 原始分享：AI Engineer，原视频发布时间说明为 2026-05-02。
- 摘要：tldraw 创始人 Steve Ruiz 分享 Fairydraw 多 Agent 画布实验。
- 实现 / 架构亮点：
  - Leader / Follower 多 Agent 分工。
  - Canvas 作为所有 Agent 与人类共同可见的共享状态空间。
  - Agent 可以观察其他 Agent 的动作并协调大型任务。
  - 讨论 Desktop App 暴露本地执行能力的边界和风险。
- 关联性：极高，直接讲 tldraw Agent 落地实验。
- 真实案例证据：是。

### tldraw-skill：用自然语言画专业架构图、流程图、序列图 | Claude Code Skill 详解

- 链接：https://www.bilibili.com/video/BV1jQ9sBAEYC
- UP：AI技术投降派
- 时长：06:44
- 可见热度：6,470 播放、129 赞、18 币、638 收藏、98 分享（本轮读取）
- 摘要：介绍 Agents365-ai/tldraw-skill 的安装、图表生成、视觉自检和多 Agent 能力。
- 实现能力：
  - 架构图、流程图、序列图、ML / DL 图、ER、UML 六类预设。
  - 视觉自检闭环，自动检查箭头、标签、布局并修复。
  - 最多多轮迭代审查。
  - 支持 Claude Code、OpenCode、OpenClaw、Hermes Agent、OpenAI Codex 等。
- 关联性：高，直接 tldraw 文档生成实践。
- 开源源码：https://github.com/Agents365-ai/tldraw-skill

### 其他已命中的 tldraw 相关视频

- 免费使用无限AI画布！tldraw Computer × Gemini 2.0 Flash & Image 3
  - UP：kate人不错
  - 可见播放：8,176
  - 方向：AI Workflow / 多模态 Canvas。
- 太真实了！tldraw 发布 makereal 功能
  - UP：沧海九粟
  - 可见播放：12,625
  - 方向：Sketch → App。
- 让 AI 一句话画出手绘白板风图表 | tldraw-skill
  - UP：探索未至之境
  - 可见播放：12,311
  - 方向：自然语言图表生成。
- tldraw Agent 自动画图
  - UP：i陆三金
  - 可见播放：1,061
  - 方向：Canvas Agent。
- TLDraw 一键生成应用程序
  - UP：鱼C-小甲鱼
  - 可见播放：13,629
  - 方向：Make Real。
- 让各种想象变成现实！tldraw 继续 makereal
  - UP：沧海九粟
  - 可见播放：3,034
  - 方向：AI UI / App 生成。
- Make It Real：画示意图让 AI 写代码做游戏
  - UP：谜之声
  - 可见播放：59,286
  - 方向：Sketch → Code / App。

---

## 七、Exa / 全网补充结果

以下结果由 Exa / 通用 Web 搜索补充，并与官方或 GitHub 结果交叉验证：

- Multiplayer Starter Kit：生产级自托管协作。
- Agent Starter Kit：完整 Canvas Agent 架构。
- Workflow Starter Kit：可执行 Node Graph。
- Branching Chat Starter Kit：AI Conversation Tree。
- AI Integrations：Agent / Workflow / Output 三种模式。
- tldraw sync：WebSocket / TLSocketRoom / SQLite / Schema / Migration。
- `tldraw-sync-cloudflare`：可直接部署的多人协作源码。
- Padlet Sandbox Case Study：40 万+ 月活真实产品。
- ClickUp Whiteboards：千万级用户产品中的 Canvas。
- Mobbin Internal Tools：ML Annotation + Content CMS。
- Jam Annotation：Screenshot / Video Annotation。
- Jam Video Annotation Engineering：第三方团队自身技术复盘。
- Remotion + tldraw：Video Editor 架构探索。
- Custom Shape Geometry：ShapeUtil 完整实现。
- Multiplayer Custom Shape：自定义数据结构进入同步 Schema。
- Mermaid Custom Shapes：Mermaid → DAG → Interactive Pipeline。
- `do-once/tldraw-cli`：Agent + CLI + JSON-RPC + Canvas。
- `tldraw-offline`：Local-first Agent Canvas。

---

## 八、当前最值得直接参考的落地路线

### 1. AI Workflow Builder

- 推荐基础：`tldraw/workflow-template`
- 可借鉴：Custom Node、Port、Binding、拖拽连线、依赖关系、执行引擎。
- 适用：Dify / n8n / Langflow / ComfyUI 风格产品。
- 参考价值：★★★★★

### 2. Canvas Agent

- 推荐基础：`tldraw/agent-template`
- 核心思路：截图 + 结构化 Shape 同时作为模型上下文，Typed Action 驱动画布。
- 适用：AI 架构图、设计助手、可视化 Agent Workspace。
- 参考价值：★★★★★

### 3. 多人实时协作

- 推荐基础：`tldraw/tldraw-sync-cloudflare`
- 核心架构：Durable Objects + WebSocket + TLSocketRoom + SQLite + R2。
- 适用：在线白板、协同 Workflow、多人 AI Canvas。
- 参考价值：★★★★★

### 4. MCP / Coding Agent 共画

- 推荐基础：tldraw MCP App、`jananadiw/codex-tldraw-mcp`、`do-once/tldraw-cli`。
- 核心思路：Agent 读取 Canvas State / Diff，再通过结构化动作继续修改；用户和 Agent 共用同一块视觉状态空间。
- 适用：Codex / Claude Code 代码架构图、产品工作流图、可视化代码理解。
- 参考价值：★★★★★

### 5. AI 图片无限画布

- 推荐基础：`mrslimslim/gpt-image-canvas`
- 核心思路：tldraw 作为无限 Canvas 和对象布局层，AI 生成 / 编辑作为节点或动作能力。
- 适用：Lovart / Canva AI / 图片工作流类产品。
- 参考价值：★★★★★

### 6. Mermaid / 结构化图生成后继续编辑

- 推荐基础：官方 Mermaid Custom Shape 示例、`Agents365-ai/tldraw-skill`。
- 核心价值：避免 AI 只生成不可编辑图片，把结果落成真正的 tldraw Shape / Binding。
- 适用：架构图、流程图、UML、CI/CD Pipeline、Agent 自动画图。
- 参考价值：★★★★★

### 7. Local-first / Offline Agent Canvas

- 推荐基础：`tldraw/tldraw-offline`、`atpaawej/openboard`。
- 适用：本地数据、隐私敏感、桌面 Agent Workspace、离线知识工作。
- 参考价值：★★★★★

---

## 九、结论

本轮确认后，tldraw 不应只按“白板 SDK”理解。它已经形成几条具有实际源码和产品证据的落地路线：

1. 无限画布基础设施。
2. 自托管多人实时协作。
3. AI Agent 直接读取和修改 Canvas。
4. 节点式 AI / 自动化 Workflow。
5. MCP / Coding Agent 与人共享同一块可编辑画布。
6. Mermaid / 结构化图转为原生可编辑 Shape。
7. AI 图片无限画布。
8. Local-first / Offline Agent Workspace。
9. ClickUp、Padlet、Mobbin、Jam 等真实商业产品和内部工具。

其中最值得继续深入源码的四条线是：**Workflow、Agent、MCP、Multiplayer Sync**。
