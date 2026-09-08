| 推荐理由 | 链接地址 |
| --- | --- |
| DeepSeek Harness 官方仓库，适合作为分享入口，覆盖插件化架构、Web UI、工具与运行模式。 | https://github.com/deepseek-ai/deepseek-harness |
| 官方架构总览直接给出工具、后台任务、Webhook、沙箱等扩展点，便于设计爬虫管理平台的模块边界。 | https://deepseek-harness.github.io/deepseek-harness/reference/ |
| 官方插件入门教程，适合参考如何把“爬虫引擎、任务控制、结果查询”封装成 Harness 插件。 | https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/ |
| Web Access 子系统说明搜索与抓取 provider 的抽象方式，可借鉴为多爬虫后端统一适配层。 | https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/web |
| Background Jobs 子系统提供任务注册、状态、停止与输出读取模型，和爬虫任务中心的核心需求高度一致。 | https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/jobs |
| Storage 子系统展示 JSON、SQLite 与领域存储的可插拔设计，可用于任务配置、爬取结果与运行元数据持久化。 | https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/storage |
| HTTP Server 子系统支持插件注册路由，适合扩展爬虫任务 API、回调接口和管理端后端能力。 | https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/web-server |
| Web Client Slots 是 Harness 的 React UI 扩展机制，可直接参考实现任务列表、运行详情、日志和监控面板。 | https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/slots |
| Workflow 子系统适合研究复杂爬取流程的编排，例如发现 URL、分片抓取、结构化抽取与后处理的多阶段任务。 | https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/workflow |
| Schedule 文档明确当前内置调度是 session-local 且不支持 Cron，可作为平台另建持久化调度器时的重要边界依据。 | https://deepseek-harness.github.io/deepseek-harness/guide/schedule |
| Crawl4AI 自带异步爬取、LLM 友好 Markdown、Docker API、任务接口与监控面板，适合作为 Harness 的爬虫执行后端。 | https://github.com/unclecode/crawl4ai |
| Firecrawl 提供 Search、Scrape、Crawl、Map、Interact 及异步任务 API，适合参考统一爬取服务与 Agent 接口设计。 | https://github.com/firecrawl/firecrawl |
| Crawlee 提供持久队列、自动扩缩容、代理轮换、重试、Playwright/Puppeteer 和存储，适合任务执行层设计。 | https://github.com/apify/crawlee |
| Gerapy 是基于 Scrapy/Scrapyd 的分布式爬虫管理框架，可重点参考项目管理、部署、调度与 Web 管理界面。 | https://github.com/Gerapy/Gerapy |
| ScrapydWeb 聚焦集群管理、日志分析、定时任务、监控告警和移动端 UI，适合借鉴管理平台的产品功能清单。 | https://github.com/my8100/scrapydweb |
| Browserless 提供容器化浏览器池、并发与排队能力，适合解决动态网页爬取时浏览器资源管理和隔离问题。 | https://github.com/browserless/browserless |
| DeepSeek Harness 的 MCP 管理控制台，带服务 CRUD、健康诊断和工具试调用，可直接参考把多个爬虫 MCP 服务接入统一管理面板。 | https://github.com/PerryLink/dsh-mcp-panel |
| Crawlab 是与语言和框架解耦的分布式爬虫管理平台，适合参考蜘蛛、任务、节点和运维管理的完整产品模型。 | https://github.com/crawlab-team/crawlab |
| Crawlab MCP 将蜘蛛 CRUD、任务运行/取消/重启、日志与文件管理暴露为 MCP 工具和资源，适合用 DeepSeek Harness MCP 客户端桥接爬虫平台能力。 | https://github.com/crawlab-team/crawlab-mcp |
| Crawlab Lite 是 Crawlab 的轻量版本，适合参考单机或小规模场景下如何裁剪爬虫管理平台的部署与功能边界。 | https://github.com/crawlab-team/crawlab-lite |
| DeepSeek Harness 管理插件套件覆盖 MCP、Skills、项目文档、规则与记忆、SSH，可借鉴把爬虫平台后台能力拆成独立 Harness 管理插件。 | https://github.com/tanleikingsley913/dsh-management-suite |
| 多平台公开数据采集 MCP 服务，覆盖 B 站、小红书、抖音、快手、微博、贴吧、知乎等，可作为 Harness 调用垂直爬虫服务的实现样例。 | https://github.com/mcp-service/media-crawler-mcp-service |
| SEO 爬虫 MCP 同时提供实时 Web Dashboard 和 Excel 报告，适合参考爬虫管理平台的运行可视化、结果查看与报表导出设计。 | https://github.com/hna2810/seo-crawler-mcp |
| 知乎实战把 DeepSeek Harness 用于数据采集，给出代理 IP、PTC 批处理“拉代理→并发采→落库→出报告”、轨迹审计与配置外置建议，最贴近爬虫管理平台落地。 | https://zhuanlan.zhihu.com/p/2076352308636664606 |
| 该回答展示 Harness 自动抓取 GitHub API 数据、生成构建脚本并用 GitHub Actions 每 6 小时刷新，适合参考采集任务自动化、验收与持续运行链路。 | https://www.zhihu.com/question/2071348486667237276/answer/2073468001341419539 |
| Litefuse 接入文章说明如何订阅 session/event、重建 Trace Tree、统计模型/工具耗时并关联 Subagent，适合实现爬虫任务链路追踪与可观测面板。 | https://zhuanlan.zhihu.com/p/2073095845998749671 |
| 规模化踩坑文章聚焦耗时、成本、失败定位，以及 Session 事件流、工具调用和跨机器汇聚，适合设计分布式爬虫平台的日志、指标与故障诊断。 | https://zhuanlan.zhihu.com/p/2075530029027550539 |
| Harness 教程介绍 Python SDK 通过 JSON-RPC stdio 驱动运行时、隔离 session 与自定义 cordis 配置，适合把 Harness 嵌入爬虫平台后端服务。 | https://zhuanlan.zhihu.com/p/2077694603986198816 |
| 插件教程整理 dsh-automation 等插件，其中定时任务可独立创建 Agent/会话并记录工作区、权限、结果和错误，可借鉴爬虫定时调度与运行审计。 | https://zhuanlan.zhihu.com/p/2079905240350986775 |
| 1Panel 部署教程给出 HTTPS/认证、网络来源限制、凭据保护、挂载范围与升级备份建议，适合补齐爬虫管理平台的部署安全和运维边界。 | https://zhuanlan.zhihu.com/p/2076067085524923336 |
| 该 DSH 插件把“查询→搜索页/sitemap→URL 评分→抓取→正文提取”串成统一工具，并加入域名白名单和双层缓存，最适合参考爬虫插件的数据通路、安全边界与缓存设计。 | https://juejin.cn/post/7673531977241460736 |
| 插件盘点同时覆盖 dsh-web-ui 的任务看板/统计面板与 dsh-computer-use 的浏览器自动化，可用于补齐爬虫管理平台的监控界面和动态网页操作能力。 | https://juejin.cn/post/7675273747710001206 |
| 容器化实测集中暴露版本锁定、非交互式调用和沙箱边界问题，适合爬虫 Worker 镜像化、无人值守执行以及运行隔离设计。 | https://juejin.cn/post/7674094098298961960 |
| 进阶玩法展示通过 MCP 给 Harness 接入工具以及多 Agent 协作，适合把爬虫任务 API 暴露为 MCP 工具，并让多个 Agent 分工执行采集、解析和验收。 | https://juejin.cn/post/7678161312636567598 |
| 从安装到写出第一个可用工具插件的真机教程提供最小工程闭环，可直接参考把爬虫启动、停止、状态和结果查询封装成 DSH 插件。 | https://juejin.cn/post/7680183953518690338 |
| 文章从 Cordis 插件、服务、依赖注入和事件机制搭出可执行 bash/fetch/文件搜索的 Mini Harness，适合理解如何把爬虫执行器、队列、存储和事件拆成可组合服务。 | https://juejin.cn/post/7673978658779955251 |
| 讲解 Cordis、插件机制与本地搭建流程，适合梳理爬虫执行器、任务服务、存储和管理端如何拆成可组合的 Harness 能力，并验证本地运行链路。 | https://www.youtube.com/watch?v=KaQWNDe2EaU |
| 插件开发实操可直接参考把爬虫启动、停止、状态、结果查询等能力封装成 Harness 插件，并为后续接入 MCP 爬虫服务建立扩展入口。 | https://www.youtube.com/watch?v=iieUVcoPjlI |
| Cordis 插件完整讲解有助于理解插件依赖、组合和扩展方式，适合设计可替换的爬虫执行层、队列、存储与监控模块。 | https://www.youtube.com/watch?v=APhQxIKKq0g |
| 以 SEO 六步自动化为例展示流程化任务思路，适合借鉴“发现目标→采集→分析→输出”的多阶段爬虫工作流编排和任务模板化。 | https://www.youtube.com/watch?v=HDN76QA9pjU |
| 从工程视角解析 DeepSeek Harness，适合补充理解其运行时、执行链路与扩展边界，为爬虫管理平台的任务运行架构取舍提供参考。 | https://www.youtube.com/watch?v=yjHdWGWgAfk |
| Agentic AI 速成课覆盖 Harness 驱动不同模型的工作方式，适合参考把模型层与爬虫工具层解耦，让不同模型承担调度、解析或验收角色。 | https://www.youtube.com/watch?v=legYz3Hk2rQ |
| dsh-free-search 为 Harness 增加统一搜索 provider，带多引擎自动回退、结果缓存、web_fetch 与 B站/GitHub/V2EX 等平台搜索，可直接参考爬虫平台的“多采集源适配 + 降级 + 缓存”设计。 | https://github.com/DDDMUC/dsh-free-search |
| dsh-Basics-Panel 在 DSH 设置页可视化管理 MCP、Skills 和规则，支持状态展示、开关、搜索过滤与模块化 feature 注册表，适合作为爬虫源、执行器和策略配置的管理面板原型。 | https://github.com/yxsj245/dsh-Basics-Panel |
| DSH-Remote 支持手机端查看会话/工作区、持续后台任务、中断任务、权限审批、环境诊断和断线自愈，可借鉴爬虫平台的远程任务控制与告警运维体验。 | https://github.com/201222-L/dsh-mobile-remote |
| 实测用一个 Skill 让 DeepSeek Harness 直接操控真实浏览器，适合动态网页采集、登录态页面抓取和需要点击交互的爬虫执行场景。 | https://www.bilibili.com/video/BV1164d6qEtA |
| MCP 教程覆盖 DSH 接入现成/自定义 MCP 服务、配置迁移和可视化管理，适合把各类爬虫服务统一包装成 MCP 后端并由 Harness 调用。 | https://www.bilibili.com/video/BV1xLtS6WEVG |
| Playwright 浏览器插件通过可访问性快照和稳定 ref 驱动真实浏览器，并提供结构化抽取、截图、标签页与会话隔离，适合动态网页采集 Worker。 | https://github.com/ChenyuHeee/dsh-browser-playwright |
| dsh-polling 把 Cron 轮询做成独立会话，支持自然语言创建、Web 管理、错过补跑与避免并发堆积，适合作为周期爬取任务中心。 | https://github.com/cnyac/dsh-polling |
| 系统级 Cron 调度插件通过 crontab 进程外启动 headless DSH，Web UI 无需常驻，适合无人值守爬虫调度与运行历史管理。 | https://github.com/Mappedinfo/dsh-cron-scheduler |
| 定时任务插件提供每日/每周/间隔调度、失败重试、超时、通知和无人值守权限，适合参考爬虫任务 SLA、告警与补偿机制。 | https://github.com/Jeff1573/dsh-plugin-scheduled-tasks |
| LoongSuite 将 DSH 的 session、step、LLM 与 tool 生命周期转换为 OpenTelemetry Trace/Metric，可用于爬虫任务链路、耗时、失败与资源消耗监控。 | https://github.com/loongsuite/dsh-plugin |
| 浏览器桥接插件直接连接现有 Chrome/Firefox 标签页并保留登录态、Session 与 Cookie，适合需要账号态和交互态的网页采集。 | https://github.com/Lum1104/dsh-browser |
| 轻量 Playwright 无头浏览器插件提供渲染后页面文本、截图、点击与输入工具，并按 Agent 隔离浏览器会话，适合作为动态网页爬虫 Worker 的最小执行实现。 | https://github.com/xu1132/dsh-plugin-browser |
| 原生 Chromium 插件通过 Puppeteer/CDP、增量 DOM、稳定元素引用和 Session 隔离提供 15 个浏览器工具，适合高交互网页采集并控制上下文开销。 | https://github.com/dengpeihua/dsh-browser |
| 本地优先任务看板把 Kanban、列表、Gantt、工作流、仪表盘与任务级 AI 对话嵌入 DSH，并以 SQLite 持久化，适合参考爬虫任务中心与运行记录界面。 | https://github.com/ttmouse/dsh-taskboard |
| Symphony 兼容的 DSH 任务编排与运行看板支持并发限制、失败退避、持久工作区、Runtime/Projects/Configuration 视图，可直接借鉴爬虫调度中心与 Worker 运维面板。 | https://github.com/Uddoo/dsh-dashboard |
| SSH 远程执行插件把 subprocess、文件系统、PTY 与 LSP 无缝切到远端主机，支持 ProxyJump 与 SFTP，适合把不同服务器组织成分布式爬虫 Worker 节点。 | https://github.com/UynajGI/dsh-ssh |
| Reef 把共享 Playwright 多标签/多 profile 浏览器、Cookie/表单能力、MCP Server 和原生状态面板打成一个 DSH 插件包，适合参考“采集 Worker + MCP 接口 + 管理控制台”一体化设计。 | https://github.com/huey1in/reef |
| 该桥接插件把官方 Playwright MCP 的导航、交互、网络、Cookie 与 Storage 等 41 个工具动态接入 DSH，并处理重连、超时和取消，适合实现标准化浏览器爬虫执行器适配层。 | https://github.com/toothemooon/dsh-playwright-mcp |
| 原生 Taskboard 以 SQLite 持久化项目、任务、关系、附件、工作流和自动化，并区分 Agent 提交与人工验收，适合参考爬虫任务中心的状态机、审核和持久化模型。 | https://github.com/shengsheng90/DSH-taskboard |
| dsh-task-dag 直接从 DSH 的 Session、Agent Teams 和 Workflow 投影构建实时拓扑与 blockedBy 依赖图，可借鉴多阶段爬取 DAG、子任务依赖和运行链路可视化。 | https://github.com/LeemanCheung/dsh-task-dag |
| TaskSwarm 按依赖 DAG 划分 wave、并行 lane 执行，带持久状态、崩溃恢复、独立审核和 Web Dashboard，适合参考大规模分片爬取的并发编排、恢复与质量门禁。 | https://github.com/february2015/dsh-taskswarm |
| Tensorlake 沙箱把 DSH 的文件、子进程、Bash、终端和 LSP 放进短生命周期 microVM，适合把高风险或不可信爬虫 Worker 与宿主机隔离，并统一控制资源与生命周期。 | https://github.com/tensorlakeai/dsh-tensorlake-sandbox |
| dsh-task-status 在对话区原生展示后台任务数量、状态、耗时和实时输出 tail，可直接参考爬虫任务运行进度、日志尾随和故障排查体验。 | https://github.com/vlln/dsh-task-status |
| dsh-plugin-chrome 为每个 Session 提供独立可视 Chrome、稳定 a11y 元素引用、16 个控制工具、实时画面和人工接管，适合动态页面采集调试及需要人机协同的爬虫 Worker。 | https://github.com/jiaererw/dsh-plugin-chrome |
| 文章明确 Web UI、headless、ACP 与 Python SDK 四种交付形态，可用于规划“管理端 + 无人值守爬虫 Worker + 平台嵌入接口”的运行边界。 | https://zhuanlan.zhihu.com/p/2072102005342999515 |
| 架构教程把 Harness 解释为无特权插件树，并强调可逆副作用与热替换，适合设计可动态装卸且能正确回收资源的爬虫执行器、存储和监控插件。 | https://zhuanlan.zhihu.com/p/2077502279024906783 |
| Cordis 源码解析覆盖 Context、Fiber、Service、Effect、Event 与 Loader，可用于建立爬虫 Worker 的插件生命周期、资源归属、事件通信和动态配置模型。 | https://zhuanlan.zhihu.com/p/2078170252194664835 |
| 插件清单中的 dsh-web-search-pro 支持多搜索引擎、知乎/小红书/B站等平台定向采集与本地缓存，适合参考多来源爬虫适配和缓存层。 | https://zhuanlan.zhihu.com/p/2077774076198794834 |
| 系统调研梳理 GuardService 的工具执行前检查、超时与重试，以及 Bundle/Profile/Patch 分层配置，适合爬虫平台的任务防护和环境配置管理。 | https://zhuanlan.zhihu.com/p/2078079607874516597 |
| 内置 Tool 分类整理了文件、Shell、持久终端与 run_code 等执行能力，可作为爬虫 Worker 能力清单、权限矩阵和任务执行接口设计的参考。 | https://zhuanlan.zhihu.com/p/2074569535211020354 |
| 深度解读指出执行环境本身也可插件化切换到本地 Docker、远程 SSH 或 K8s Pod，适合分布式爬虫 Worker 的部署与弹性运行架构。 | https://zhuanlan.zhihu.com/p/2072691741732365313 |
| 长篇教程覆盖 Web UI、工作区、工具调用树、权限模式与会话归档，可用于设计爬虫平台的管理端操作流、任务观测与权限控制。 | https://juejin.cn/post/7673390412729614390 |
| 插件合集同时包含 dsh-browser、dsh-agent-teams、dsh-web-ui 等，适合参考“浏览器采集 + 多 Agent 分工 + Web 管理面板”的组合架构。 | https://juejin.cn/post/7676098169974292530 |
| 命令大全梳理 profile、headless 与插件按 profile 管理等运行方式，适合把不同爬虫 Worker/环境拆成可配置运行 profile 并支持无人值守执行。 | https://juejin.cn/post/7675947280080240691 |
| 一周真实需求复盘聚焦环境配置、任务拆解、插件选型与预期管理，可用于制定爬虫任务模板、插件选择策略和运行边界。 | https://juejin.cn/post/7677441124442570761 |
| 实测将 Harness 定位为可组装的 Agent 运行时底座而非成品助手，适合用作爬虫管理平台插件化内核，将采集、调度、存储、UI 与 Agent 分层组合。 | https://juejin.cn/post/7673810995882672128 |
| 实测覆盖 WebUI 远程控制、执行轨迹、插件系统和任务分支，可参考爬虫管理端的远程运维、任务追踪与分支执行体验。 | https://www.youtube.com/watch?v=Aqn7EP8shJw |
| 展示自定义工作台、定时任务、快捷指令以及 Skill 与 MCP 管理，可参考把爬虫调度、执行器配置和管理 UI 集成到 Harness。 | https://www.youtube.com/watch?v=1BXLQj8C0Ps |
| 从零跑通 Web UI 和首个 Agent 任务，适合验证爬虫平台“任务创建→执行→查看结果”的最小管理闭环与新用户操作路径。 | https://www.youtube.com/watch?v=mpelxra1aL4 |
| dsh-workmate 同时提供长任务结束/失败 Webhook 通知、web_capture 网页正文抓取入库和最近抓取记录查询，可直接参考爬虫任务告警、采集入库与结果回看闭环。 | https://github.com/halosb/dsh-workmate |
| DeepSeek Flow 将 WORKFLOW.md/STEP.md 映射成可视化流程图，支持节点连线、条件门、会话隔离、后台 AI 作业与持久草稿，适合设计多阶段爬取流程编排和管理界面。 | https://github.com/kanghelyu/dsh-deepseek-flow |
| DSH WebUI 插件市场支持插件搜索、跨 Profile 安装/同步、FIFO 安装更新卸载队列、超时/重试/日志和来源白名单，可借鉴爬虫执行器/适配器的生命周期与插件管理。 | https://github.com/Sanqi-normal/dsh-webui-market-plugin |
| Firecrawl 官方 DSH 插件直接把 Harness 内置 `web_search`/`web_fetch` 接到 Firecrawl，并提供 JS 渲染、反爬处理与 PDF 解析，适合作为托管爬取执行层。 | https://github.com/firecrawl/dsh-firecrawl |
| dsh-web-search 把 SearXNG、Tavily、Brave、DuckDuckGo 做自动回退并提供 URL 正文提取，可参考爬虫平台的多源搜索路由、故障降级与抓取 provider 配置。 | https://github.com/haibinwang9/dsh-web-search |
| 安全优先的浏览器插件对每次导航和重定向重复校验域名并默认阻断私网访问，适合动态爬虫 Worker 的 SSRF 防护、域名白名单与下载隔离。 | https://github.com/coderdailyone/dsh-plugin-browser-use |
| Session Supervisor 可按静默超时、截止时间和连续异常 turn 形成持久 incident，并支持确认与恢复审计，可用于长任务爬虫的卡死检测和运行健康监督。 | https://github.com/acosmi/dsh-plugin/tree/main/plugins/dsh-session-supervisor |
| dsh-webhook 把签名 HTTP 事件转成 Agent 任务，并提供去重、重放、回执、冷启动和回调重试，可参考事件驱动爬取与外部触发任务入口。 | https://github.com/omdsh-dev/dsh-webhook |
| DSH Studio 把项目、会话、终端、浏览器和插件放在同一 Desktop/Web 工作台，适合参考爬虫项目工作区、运行操作台与插件市场的一体化管理体验。 | https://github.com/euanguo/dsh-studio |
| DeepSeek Harness Studio 提供零代码桌面端、插件发现/推荐/安装管理与视觉增强，适合参考面向非开发用户的爬虫管理平台安装、扩展器管理和桌面交付。 | https://github.com/fufankeji/deepseek-harness-studio |
| Tencent BrowserSkill 提供 DSH 原生浏览器插件与实时 Web UI 覆盖层，可复用真实登录态浏览器，适合账号态网站采集和需要人工观察或接管的爬取任务。 | https://github.com/Tencent/BrowserSkill |
| SSRF 防护型 WebFetch Provider 会校验公共地址、固定 DNS 解析结果并限制重定向、响应大小和并发，适合爬虫平台统一 URL 安全边界与出网控制。 | https://github.com/MostlyHarmlessxyz/dsh-safe-web-fetch |
| 本地插件同时提供 Node fetch 与 Playwright/Chrome 渲染通道，并以子进程隔离执行，适合按页面类型切换轻量抓取和动态浏览器 Worker。 | https://github.com/junhongchashui/dsh-plugin-web-access |
| 把 SearXNG 搜索与 Crawl4AI 抓取直接注册为 Harness 原生 `ctx.web` Provider，适合自托管“发现 URL→正文抓取”的统一采集层。 | https://github.com/cyijun/surfing-plugin |
| fastCRW Provider 保持原生 `web_search`/`web_fetch` 接口，可自托管并按页面自动升级浏览器抓取，适合做可替换爬虫后端与反爬降级层。 | https://github.com/us/dsh-crw |
| 统一导出 DSH 插件 Trace/Log/Metric 到 OTLP 并桥接 Session telemetry，适合给爬虫 Worker、Provider 和任务插件建立统一可观测体系。 | https://github.com/fly3366/dsh-o11y-plugin |
| 系统代理插件支持按规则路由代理、保护私网目标并安全处理代理凭据，可借鉴爬虫平台按域名/Provider 配置代理和出网策略。 | https://github.com/khiqwq/dsh-system-proxy |
| AgentCrawl 提供 SQLite 持久任务、检查点、取消/重试、失败记录、MCP/API 和本地 Dashboard，适合作为 Harness 下游爬虫服务或任务模型参考。 | https://github.com/JorG18/agentcrawl |
| 浏览器集群控制面支持多 Provider 负载均衡、并发上限、排队、故障切换、持久 Profile、回放、REST/MCP 与 Dashboard，适合管理动态爬虫浏览器池。 | https://github.com/browser-gateway/browser-gateway |
| Scraper MCP 提供批量并发、Playwright 渲染、缓存、重试、实时 Dashboard 和运行时配置，可直接作为 Harness MCP 爬取执行服务。 | https://github.com/cotdp/scraper-mcp |
| 直接面向 DeepSeek Harness 的爬虫插件，适合参考如何把资料/文献采集能力作为 DSH 外置爬虫组件接入平台。 | https://github.com/Leesky10124/dsh-sky-crawler |
| 增强型 `web_fetch` Provider 支持 CIDR/域名白名单、DNS 与重定向复核、响应上限和 Web 端即时配置，适合做爬虫平台可管控的抓取出网层。 | https://github.com/Yurzi/dsh-web-fetch-enhanced |
| 本地 Readability 抓取 Provider 带 SSRF 防护、正文净化、响应限制和可取消 FIFO 排队，适合轻量网页 Worker 与并发队列设计。 | https://github.com/Apoze/dsh-web-fetch-local |
| 为 `web_fetch`、`web_search` 与部分命令执行增加出网 allowlist、audit/enforce 和 OCSF 记录，适合爬虫平台的网络策略与审计层。 | https://github.com/CharlotteN7/dsh-netguard |
| 将 `web_fetch` 串成本地抓取→本地 Ollama 整理链路，适合隐私敏感场景下的本地内容处理与结果标准化。 | https://github.com/wanghj040530/dsh-local-web |
| 无 API Key 的 Bing 搜索 Provider，可补充爬虫平台“发现 URL”阶段的低成本搜索源，并提供 Bundle 化安装方式。 | https://github.com/SUJIElearning/dsh-search-free-nokey |
| RedFoxHub × DSH 一次接入 113 个数据技能与 40 个 MCP 工具，覆盖多社媒搜索、评论、热榜和定时账号订阅，适合参考多源采集适配器、任务模板与周期抓取设计。 | https://zhuanlan.zhihu.com/p/2073360129123149137 |
| 文章把 DSH headless 作为隔离子进程嵌入 Electron 产品，并通过 MCP 复用工具、用环境变量注入凭据，适合爬虫管理平台把 Agent 运行时作为后台 Worker 并与管理 UI 隔离。 | https://zhuanlan.zhihu.com/p/2078388330799051093 |
| 文章解析工具的 pre-execute→guard→execute→post-execute→result 流水线、Code Mode 与 MCP 接入，适合为爬虫工具建立统一权限、域名/出网策略和执行门禁。 | https://zhuanlan.zhihu.com/p/2079236658084459433 |
| 这份新插件清单覆盖插件市场、界面增强与多源搜索等能力，适合筛选爬虫平台的采集源适配器、管理端扩展和插件安装机制。 | https://juejin.cn/post/7679542577553473590 |
| 从插件架构、MCP、Hooks 和子代理委派对比 Harness 与 Claude Code，可用于确定爬虫服务的 MCP 接入层、任务委派和扩展边界。 | https://juejin.cn/post/7676239036387000335 |
| 文章从 Cordis 服务、依赖注入、事件分发和可逆副作用解释“一切皆插件”，适合设计爬虫执行器生命周期、任务事件与通知/监控插件。 | https://juejin.cn/post/7673436957741039631 |
| 聚焦 DeepSeek Harness UI 插件原理，可参考把爬虫任务列表、运行状态、日志与监控组件做成 Harness 原生管理端扩展。 | https://www.youtube.com/watch?v=D9W4BhG9HDk |
| 展示 DeepSeek Harness 插件生态与插件市场工具箱，可参考爬虫执行器、采集源和监控模块的发现、安装与统一管理入口。 | https://www.youtube.com/watch?v=QCxq__dLv5E |
| 展示用 Harness 插件快速配置 MCP 和 Skill，可参考集中管理爬虫 MCP 服务、采集 Skills 与运行配置的控制面设计。 | https://www.youtube.com/watch?v=CgYZ5EzX00U |
| 介绍 dsh-browser + argo 的浏览器与网页插件组合，可补充理解在 DSH 内把真实浏览器访问与网页能力组合为动态采集执行层。 | https://www.bilibili.com/video/BV1bC8A6vEno |
| 展示 dsh-raw-html 与 VCP 渲染组合，可参考在爬虫平台保留原始 HTML 并提供渲染/预览能力，方便解析调试与结果核验。 | https://www.bilibili.com/video/BV1WJbW6UE9K |
| 介绍版本自动更新与插件仓库管理两个开源插件，适合参考爬虫执行器/采集插件的版本检测、仓库维护和生命周期管理。 | https://www.bilibili.com/video/BV1Sp8m6oExn |
| 展示零侵入可视化工作台插件与多窗口应用能力，可借鉴爬虫管理平台把任务、结果与工具操作做成 Harness 内的可视工作台。 | https://www.bilibili.com/video/BV1kw8b6bEeL |
| 官方 `dsh-web-fetch-http` 明确公网地址校验、DNS 固定、同源重定向、响应字节/字符上限与无凭据抓取策略，适合直接作为爬虫平台默认安全抓取通道和出网基线。 | https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/web/web-fetch-http/README.md |
| 官方 `dsh-tool-web` 把 `web_search`/`web_fetch` 的工具 schema、超时、结果上限、HTML→Markdown 与 Web UI 展示统一封装，适合设计爬虫平台面向 Agent 的稳定工具契约。 | https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/web/tool-web/README.md |
| 官方 Web capability seam 架构记录说明搜索与抓取 Provider 注册、选择、错误语义和安全边界，适合把不同爬虫/搜索后端做成可热插拔 Provider。 | https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.md |
| 官方大工具输出 spill 设计以 `web_fetch` 为示例，把超大抓取结果自动落文件并保留引用，可用于长网页、批量抓取结果的上下文限流与结果留存。 | https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.md |
| dsh-web-fetch-playwright 将 Harness 原生 `web_fetch` 接到真实 Playwright/CDP 浏览器，带 Readability 净化、并发控制、CDP 登录态和动态配置，适合动态页面爬虫 Worker。 | https://github.com/chendefine/dsh-web-fetch-playwright |
| dsh-web-search-provider 同时支持 OpenAI Responses 与 Anthropic Messages 的原生搜索，并扩展 `open_page`/`find_in_page` 浏览动作和运行时探测，适合构建“发现→打开→定位内容”的采集链路。 | https://github.com/hiyms/dsh-web-search-provider |
| dsh-web-search-pro 集成多搜索引擎、20 类平台搜索、SQLite+LRU 缓存、Playwright 渲染、抓取历史和按站提取规则，功能形态很接近可嵌入 Harness 的轻量爬虫管理层。 | https://github.com/anweat/dsh-web-search-pro |
| @240xu/dsh-websearch 将 11 个搜索后端并发 fan-out、URL 去重并支持部分后端故障继续工作，可参考爬虫平台的多来源发现层、容错路由和统一凭据配置。 | https://github.com/240xu/dsh-websearch |
| dsh-read-url 自动识别 GBK/GB2312/UTF-8/Big5 并抽取正文，提供默认长度上限、缓存和 offset 续读，适合低成本正文采集、中文老站抓取及控制 Agent 上下文消耗。 | https://github.com/2672243194/dsh-read-url |
| deepspider 是基于 DSH、Patchright/CDP 和独立语义运行时的 AI 原生智能爬虫与 JavaScript 逆向平台，可参考把浏览器证据、反爬分析和可验证采集逻辑统一纳入爬虫 Worker。 | https://github.com/ma-pony/deepspider |
| dsh-browser4 面向智能抽取与大规模 Web 自动化提供 DSH 原生浏览器引擎，适合补充动态页面爬取、浏览器执行器标准化与规模化采集能力。 | https://github.com/platonai/dsh-browser4 |
| dsh-xhs-collector 是基于 CDP Chrome 与住宅代理的小红书批量采集案例，提供真实批采实践，可参考垂直站点采集任务模板、代理使用与批量结果管理。 | https://github.com/nataliwhite20534-droid/dsh-xhs-collector |
| AnySearch 为 DSH 提供 Web Search Provider 与高级搜索工具，适合作为爬虫平台“发现 URL”层的新搜索后端，并参考 Provider 化接入方式。 | https://github.com/anysearch-team/anysearch-dsh |
| dsh-fetch-third-party 把页面抓取委托给用户配置的第三方服务，强调无直接 URL 访问、托管凭据、会话预算与本地抓取服务栈，适合参考安全抓取网关和成本控制。 | https://github.com/tallahandsome-ux/dsh-fetch-third-party |
| 该 Playwright 插件直接为 DeepSeek Harness 提供浏览器自动化，可作为另一种动态网页 Worker 实现，便于比较不同浏览器插件的接口、部署和维护方式。 | https://github.com/Clizo1209/dsh-playwright-browser |
| deepseek-harness-multi-user 在 Harness 基础上加入认证授权、Kafka、MySQL、Redis、Elasticsearch、CDC 与 Web 管理 UI，适合参考多人爬虫平台的租户隔离、数据管道和管理控制面。 | https://github.com/Foreverlearners-cpu/deepseek-harness-multi-user |
| 跨会话监控中枢维护会话台账、派单/回执状态机、失败自动重试与 Web 控制台，可直接参考爬虫任务中心的跨 Worker 调度、状态回收、重试和主动通知设计。 | https://github.com/wenki2005/dsh-monitor-hub |
| DSH 定时任务插件支持 Cron/一次性触发、命令或 Webhook、状态查询、超时和重入保护，可用于周期爬取调度、回调与运行状态管理。 | https://github.com/yangyongzhen/dsh-scheduler |
| 持久 Playwright 浏览器 Profile 与 DSH Web 交互面板支持稳定快照引用、多标签、人工接管和登录态复用，适合动态/账号态爬虫 Worker 及管理端人机协同。 | https://github.com/syncended/deepseek-harness-browser-use |