| 推荐理由 | 链接地址 |
|---|---|
| Anthropic 公开了其多 Agent Research 系统从原型到生产的真实工程经验，覆盖架构、并行子 Agent、工具设计、提示工程、评估、可靠性与成本权衡，特别适合作为“多 Agent 如何真正落地”的案例。 | https://www.anthropic.com/engineering/multi-agent-research-system |
| Anthropic 2026 年对多 Agent 系统中的协作模式、规模化交互与潜在失效问题进行了系统总结，适合分享中补充“从实现走向大规模 Agent-Agent 协作后会遇到什么”的前沿视角。 | https://www.anthropic.com/research/multiagent-systems |
| OpenAI Agents SDK 官方把多 Agent 编排拆成 manager/agents-as-tools 与 handoff 两类核心模式，并解释何时由代码控制、何时交给 LLM 决策，适合用于讲解协作架构选型。 | https://openai.github.io/openai-agents-python/multi_agent/ |
| Google ADK 官方给出了 Coordinator/Dispatcher 等多 Agent workflow pattern 及实现方式，强调确定性工作流与 LLM 驱动委派的组合，适合提炼可复用的协作设计模式。 | https://adk.dev/workflows/patterns/ |
| Google ADK 的 collaborative agent teams 文档直接讨论 coordinator + subagents 的团队协作，并通过 Chat、Task、Single-turn 模式限制子 Agent 行为和返回机制，对“如何让多 Agent 协作更可控”很有实践价值。 | https://github.com/google/adk-docs/blob/main/docs/workflows/collaboration.md |
| Microsoft Agent Framework 面向生产级 Agent 与多 Agent workflow，提供 sequential、concurrent、handoff、group collaboration、checkpoint、可观测性、HITL 等能力，适合作为企业级落地框架案例。 | https://github.com/microsoft/agent-framework |
| LangGraph Supervisor 展示了中心 supervisor 调度专业 Agent、工具化 handoff、上下文工程、记忆与 human-in-the-loop 的实现方式，适合与去中心化或 handoff 架构对比。 | https://reference.langchain.com/python/langgraph-supervisor |
| AgentTeams 用 Kubernetes 式声明式 API、控制器协调循环、组织结构、通信策略和共享状态来管理 Agent 团队，适合分享“多 Agent 从应用编排走向平台化控制面”的落地思路。 | https://github.com/agentscope-ai/AgentTeams/blob/main/docs/design/k8s-native-orchestration.md |
| Microsoft 的从零到生产教程用独立服务之间的 Agent-to-Agent（A2A）协作扩展传统进程内 handoff，适合讲解多团队、跨进程乃至跨组织 Agent 协作的工程边界。 | https://github.com/microsoft/Building-AI-Agents-From-Zero-To-Production/blob/main/lesson-7-multi-agent-a2a/README.md |
| Open Multi-Agent 采用运行时 coordinator 动态生成任务 DAG，再由确定性调度器执行，并支持审批、回放和可观测数据，适合展示“LLM 负责规划、系统负责可靠执行”的生产化分工。 | https://github.com/open-multi-agent/open-multi-agent |
| Agency Swarm 基于 OpenAI Agents SDK，把多 Agent 协作映射为现实组织中的角色、职责与通信关系，适合分享如何用组织结构建模专业 Agent 团队并降低编排复杂度。 | https://github.com/VRSEN/agency-swarm |
| 得物从个人 AI Coding 升级到团队级 Agent 工作流，覆盖任务平台、上下文工程、Harness、闭环 Loop 和多 Agent 提议/开发/评审/归档流程，并给出自动化治理数据，适合作为“团队级 Agent 工程化落地”案例。 | https://zhuanlan.zhihu.com/p/2073718776202379668 |
| 信永中和基于 AgentTeams 建设企业级多智能体平台，包含统一调度、专业 Agent 分工、业务系统接入、安全与模型治理，并披露一周完成联调上线的过程，适合分享企业多 Agent 平台从方案到上线的路径。 | https://zhuanlan.zhihu.com/p/2066163955928658877 |
| 阿里云 AgentTeams 的 github-manager 已在真实开源项目中持续处理 PR、Issue、CI 与贡献者沟通，文章同时复盘生命周期、凭证、通信和可观测性等生产门槛，适合说明多 Agent 从脚本走向“数字同事”的关键工程问题。 | https://zhuanlan.zhihu.com/p/2060719942450918822 |
| Dream-SaaS 的多 Agent 编排实战给出 LLM 拆解、Registry 匹配、DAG 并行调度、多策略合并、Checkpoint、SSE、HITL 与输出护栏等实现细节，适合讲清楚从 Demo 到生产级编排引擎还缺哪些能力。 | https://zhuanlan.zhihu.com/p/2064039480852533667 |
| Hermes Kanban 用三个 Agent 跑通真实协作流程，重点记录任务依赖、工作区隔离、中断接续与失败恢复等容易踩坑的细节，适合作为小规模多 Agent 协作如何真正跑起来的实操案例。 | https://zhuanlan.zhihu.com/p/2037533360826930853 |
| 文章从 Agent、Skills 到 Teams 梳理架构演进与技术选型，并结合多智能体和 Agent Teams 的落地经验讨论能力边界，适合在分享中建立“什么时候该从单 Agent 升级为团队协作”的选型框架。 | https://zhuanlan.zhihu.com/p/2022259323544281560 |
| 新浪微博案例聚焦企业 AI 应用平台、知识复用和多 Agent 业务场景的真实落地，同时覆盖内部推广和规模化应用路径，适合补充大型互联网组织如何把多 Agent 能力嵌入现有业务体系。 | https://zhuanlan.zhihu.com/p/1991186674080837925 |
| AgentOps 实践把单智能体、多智能体和智能体协同放进统一运维平台，并强调 Human-in-the-Loop、审计、工具治理和可量化 ROI，适合分享高风险生产场景下多 Agent 如何做到可控、可审计、可推广。 | https://zhuanlan.zhihu.com/p/2015796905158923426 |
| 从单体 Agent 推进到分布式协作，系统讲主从与对等模式、同步与异步通信、共享状态、超时重试、状态机、补偿机制、并发与 Token 成本优化，并给出完整工作流代码，适合分享“协作架构如何工程化”。 | https://juejin.cn/post/7610444188525592616 |
| 用同一业务场景对比 AgentScope 与 LangGraph 的多智能体实现，重点展示 StateGraph、Checkpoint Memory、跨 Agent 状态传递以及 Network、Supervisor、分层等协作模式，适合做框架选型与状态管理案例。 | https://juejin.cn/post/7591697558377431082 |
| 以 4 个 Agent 组成真实 AI 写作团队，完整记录跨 Agent 通信权限、会话可见性等配置导致的协作失败与修复过程，适合用来讲多 Agent 从“能跑”到“稳定跑”的常见工程坑。 | https://juejin.cn/post/7607255496454881280 |
| 基于 Kubernetes DevOps 多智能体系统，围绕 Coordinator、Analyzer、Fixer、Slack Bot 的 MCP 与 A2A 协作进一步讨论测试、调试、上下文丢失和非确定性故障，特别适合补充生产环境可靠性议题。 | https://juejin.cn/post/7613004898187739188 |
| 从 BMAD 实战经验出发比较 AgentTeams 与 Subagents，回答何时需要多 Agent、两种组织方式如何选以及怎样设计团队边界，适合作为分享中的“不要为了多 Agent 而多 Agent”决策框架。 | https://juejin.cn/post/7605164560711073818 |
| 以 LangGraph 为核心讨论生产级多智能体编排，针对单 Agent 的上下文、职责与复杂流程瓶颈展开，适合用来串联职责拆分、编排控制和生产化设计的整体方法。 | https://juejin.cn/post/7618405987397976104 |
| 从多租户 AI Agent 平台视角落地 Supervisor、Peer-to-Peer、层次化三类多 Agent 拓扑，并把编排与网关、RAG、人工干预、可观测性、资源隔离结合，适合分享企业平台化与规模化落地。 | https://juejin.cn/post/7637772413945643034 |
| 开源本地 AI 任务编排引擎把 Codex、Claude 等 Agent 与原生 CLI 串成自动化研发闭环，适合作为“多模型、多 Agent 与现有工程工具如何真正接力”的项目型案例。 | https://juejin.cn/post/7617679439414804516 |
| Factory 基于真实生产数据拆解 orchestrator、worker、validator 三角色协作，重点讲 validation contract、结构化 handoff、对抗式验证以及“何时串行胜过并行”，非常适合作为多 Agent 从架构图走向长期稳定执行的生产案例。 | https://www.youtube.com/watch?v=ow1we5PzK-o |
| Databricks 的生产实践演讲聚焦如何确定性地编排多 Agent 网络并降低多次 AI 调用带来的误差累积，适合用于分享“多 Agent 越多不等于越可靠，关键在协作约束与编排”的工程经验。 | https://www.youtube.com/watch?v=bBnOiPqDsvg |
| Google Cloud Next 2026 用 planner、simulator、evaluator 等专职 Agent 演示生产级协作，并深入到实时评估、A2A、Pub/Sub、WebSocket、Protocol Buffers 与数据治理，适合讲清楚多 Agent 的性能、通信和治理如何一起落地。 | https://www.youtube.com/watch?v=ge5cLd8uics |
| freeCodeCamp 用多 Agent PR Reviewer 做完整系统设计，覆盖安全、质量、测试、文档等专业 Agent 的 fan-out/fan-in、RAG、置信度与引用、HITL，以及幂等、超时、死锁等故障设计，特别适合作为“生产可靠性优先”的实战案例。 | https://www.youtube.com/watch?v=iqRcGCah0Kw |
| A2A 实战从协议基础一路做到 ADK 主 Agent 协调 ADK、CrewAI、LangGraph 等异构远程 Agent，并包含代码与运行演示，适合分享跨框架、跨服务多 Agent 协作怎样通过标准协议真正互操作。 | https://www.youtube.com/watch?v=mFkw3p5qSuA |
| Microsoft Copilot Studio 教程直接搭建 Master Agent 与多个 Connected/Child Agent 的协作团队，并展示与 Azure AI Foundry、Fabric、Microsoft 365 SDK 等生态连接，适合补充低代码与企业业务场景的多 Agent 落地路径。 | https://www.youtube.com/watch?v=WKKdBC2zM3k |
| 通过 LangGraph 从单 Agent 局限、协作结构讲到分层多 Agent 的完整编码实践，最终实现 supervisor 协调 research team 与 writing team 的端到端流程，适合在分享中做“层级式协作模式”的可复现实操示例。 | https://www.youtube.com/watch?v=RXOvZIn-oSA |
| DeepSeek Harness 的开源 dsh-agent-teams 插件把队长/成员、任务依赖 DAG、无依赖并行、共享任务池、通信与状态同步、差异化模型配置和最终归档串成完整闭环，适合展示多 Agent 团队如何做成可复用工程组件。 | https://www.bilibili.com/video/BV1Y1bv68Eq9 |
| Codex 多 Agent 协同开发实战直接展示研发任务里的多 Agent 配合方式，适合分享 AI Coding 场景如何从单 Agent 执行升级为角色分工与协同开发。 | https://www.bilibili.com/video/BV16hTc6xEpF |
| CrewAI 示例从任务拆分、角色设计到 Agent 协同执行完整跑通，可作为“从 0 到可用”的最小多智能体系统实践，用来讲清楚框架层的协作落地路径。 | https://www.bilibili.com/video/BV13svpBMEYV |
| Spring AI Alibaba Graph 用 Java 生态原生图编排搭建企业级多 Agent 工作流，适合补充非 Python 团队如何把图式编排和多 Agent 架构落到既有 Java 技术栈。 | https://www.bilibili.com/video/BV14WgA6SEXD |
| OpenCode 多智能体实战把 Agent 协同与浏览器工具执行结合，并给出可复用配置，适合展示多 Agent 如何协作完成真实外部环境中的自动化任务。 | https://www.bilibili.com/video/BV1jyAWzHEL7 |
| AutoGen 教程把多智能体协作与 DeepSearch 研究型流程组合在一起，适合用于分享 AutoGen 在角色协作、检索与复杂任务编排上的端到端实践。 | https://www.bilibili.com/video/BV17RC7BQEFt |
| OpenClaw 多 agents 协同配置实操聚焦让多个 AI 员工直接通信协作，适合展示轻量化 Agent 团队的通信配置与日常工作协同落地。 | https://www.bilibili.com/video/BV1CiAjzCEMd |
| AWS 2026 年生产实践把 SageMaker 上的自托管模型与 Bedrock 模型放进同一 Strands 多 Agent 系统，并部署到 AgentCore，适合分享异构模型协作、成本/数据驻留选择和生产运行时如何一起落地。 | https://aws.amazon.com/blogs/machine-learning/building-agentic-workflows-with-sagemaker-ai-and-bedrock-agentcore/ |
| AWS 从中心 supervisor 的瓶颈出发讨论 Kiro 自组织多 Agent 集群的扩展模式，重点涉及任务分解、并发、故障边界与分布式协调，适合分享多 Agent 从“单进程编排”走向规模化集群时的架构挑战。 | https://aws.amazon.com/blogs/architecture/scaling-patterns-for-self-organizing-multi-agent-clusters-with-kiro/ |
| Thrad.ai 在真实业务中同时实现 Strands 的 Swarm 与 Graph 两种多 Agent 编排，并用同一工作负载比较延迟、成本和输出质量，适合用于分享“协作拓扑如何选”以及如何用数据而不是直觉做架构决策。 | https://aws.amazon.com/blogs/machine-learning/multi-agent-social-intelligence-with-strands-agents-and-amazon-bedrock/ |
| MetaGPT 把产品经理、架构师、项目经理、工程师等角色及 SOP 映射为多 Agent 软件公司，能从一句需求贯穿需求、设计、编码等产物，适合作为“组织角色 + 标准流程驱动协作”的经典开源实现。 | https://github.com/FoundationAgents/MetaGPT |
| ChatDev 2.0 已从虚拟软件公司演进成零代码多 Agent 编排平台，可用配置定义 Agent、工作流和任务；其 1.0 又保留完整研发协作链路，适合对比“固定角色流程”与“通用编排平台”两种落地形态。 | https://github.com/OpenBMB/ChatDev |
| CAMEL 面向大规模多 Agent 系统提供角色交互、动态通信、状态记忆、工具与真实用例，并强调从少量协作扩展到大规模 Agent 社会，适合补充多 Agent 可扩展性与协作机制研究/工程两端的实践。 | https://github.com/camel-ai/camel |
| PydanticAI 官方把多 Agent 应用拆成 Agent delegation、程序化 handoff、图式控制流和 Deep Agents 等复杂度层级，并配套可观测性示例，适合建立“先简单、再逐级增加编排复杂度”的工程选型框架。 | https://pydantic.dev/docs/ai/guides/multi-agent-applications/ |
| Agno Teams 明确区分 coordinate、route、broadcast 等协作模式，还支持嵌套团队、并行执行、调试和生产平台能力，适合分享如何把团队结构、协作策略和可运维性统一建模。 | https://docs.agno.com/teams/overview |
| LlamaIndex 官方用同一研究-写作-评审场景对比 AgentWorkflow handoff、Orchestrator-as-tools 与自定义 planner 三种多 Agent 模式，并给出选型建议和代码，适合直接做“协作模式对照实验”的分享素材。 | https://github.com/run-llama/llama_index/blob/main/docs/src/content/docs/framework/understanding/agent/multi_agent.md |
| Grab 在真实数据平台中部署由 Classifier、Data、Code Search、On-call、Summarizer 等 Agent 协作的支持系统，并详细复盘上下文裁剪、工具收敛、SQL 安全、HITL 与反馈闭环，适合作为“多 Agent 从原型到稳定生产”的一手案例。 | https://engineering.grab.com/from-firefighting-to-building |
| Uber 围绕数千个内部生产 Agent 的委派链设计身份、鉴权与可追责机制，覆盖 agent-to-agent handoff、执行上下文传播、审计和低延迟 token exchange，适合补充“多 Agent 协作如何做企业级安全治理”。 | https://www.uber.com/us/en/blog/solving-the-agent-identity-crisis/ |
| Google 从生产规模的单/多 Agent 系统出发，把上下文建模为可编译视图，并系统解释 Session、Memory、Artifact、压缩过滤及 handoff 最小上下文语义，适合讲解多 Agent 最容易失控的上下文工程问题。 | https://developers.googleblog.com/architecting-efficient-context-aware-multi-agent-framework-for-production/ |
| n8n 从实际生产工作流出发总结多 Agent 架构、可复用子工作流、记忆管理、失败域隔离、降级与版本化等原则，尤其适合分享“什么时候该引入多 Agent，以及怎样避免复杂度悬崖”。 | https://blog.n8n.io/production-ai-playbook-complex-agent-patterns/ |
| NVIDIA 的 MAIW Blueprint 是面向仓储业务的开源生产级多 Agent 参考系统，用 LangGraph + MCP 编排设备、运营、安全、预测与文档 Agent，并把 RAG、Guardrails、RBAC、Prometheus/Grafana 一起落地。 | https://developer.nvidia.com/blog/multi-agent-warehouse-ai-command-layer-enables-operational-excellence-and-supply-chain-intelligence/ |
| Taskade 基于 50 万+ Agent 生产部署总结多 Agent 协作经验，重点覆盖多类型记忆、模型分层选择、团队协作模式、上下文窗口和循环保护，适合作为大规模运行后的“踩坑与经验”材料。 | https://www.taskade.com/blog/multi-agent-production |
| DeepLearning.AI 将多 Agent 从原型走向生产拆成可靠执行、可观测行为和可扩展架构，并结合 guardrails、memory、Flows、评估与真实企业案例，适合作为分享中的系统化工程清单。 | https://www.deeplearning.ai/blog/engineering-multi-agent-systems-a-path-from-prototype-to-production/ |
| Salesforce Summer ’26 已把 Multi-Agent Orchestration 引入 Agentforce，并用 orchestrator + 专业 subagents、Agent Script、MCP、trace/eval 等能力串起构建与治理链路，适合补充大型 SaaS 平台如何产品化多 Agent 协作。 | https://developer.salesforce.com/blogs/2026/06/the-salesforce-developers-guide-to-the-summer-26-release |
| 基于一次 32 小时连续运行、约 95 个 Agent 与 7 万行生产代码总结出的编排纪律，系统沉淀任务契约、worktree 隔离、证据分级、验收门禁、跨模型复核和 18 类反模式，特别适合分享“多 Agent 真正长期运行后暴露什么问题、如何治理”。 | https://github.com/indiekitai/orchestration-playbook |
| AWS Labs 的 CLI Agent Orchestrator 让 supervisor 在隔离终端中并行/串行调度 Claude Code、Codex、Kiro、Cursor 等多种 Coding Agent，并提供 Web UI、MCP 控制面、工作流、权限限制和跨会话记忆，适合作为“异构 Agent 团队统一控制面”的落地案例。 | https://github.com/awslabs/cli-agent-orchestrator |
| AWS 示例用 Full Stack lead + Coding/DevOps/Review 专业 Agent 跑通 spec-driven 的 plan→build→review→fix 闭环，包含共享任务队列、动态并发池、单一评审结论、最多三轮复核和 Hooks 机器门禁，适合直接拆解研发型多 Agent 团队的协作协议。 | https://github.com/aws-samples/sample-claude-code-agent-team |
| Shannon 把 DAG、Research、Swarm 等多 Agent 策略与 Temporal 容错工作流、Token 预算、WASI 沙箱、OPA 审批、可观测性和回放调试整合在同一生产架构中，适合分享“编排、可靠性、安全与成本治理如何一体化”。 | https://github.com/Kocoro-lab/Shannon |
| oh-my-claudecode 将 Team 作为核心编排面，提供 team-plan→team-prd→team-exec→team-verify→team-fix 的阶段流水线，并同时支持 Claude 原生团队与 Codex/Gemini 等 tmux CLI workers，适合作为 AI Coding 多 Agent 协同从个人插件走向团队化工作流的案例。 | https://github.com/Yeachan-Heo/oh-my-claudecode |
| 基于互联网平台销售客户分析智能体的真实生产经验，拆解上下级、师生式、竞争式三种多智能体协同范式，并给出 CrewAI 落地步骤与避坑点，适合补充业务型多 Agent 从设计到上线的案例。 | https://zhuanlan.zhihu.com/p/2069505265544720400 |
| 用企业级元数据管理系统验证 Claude Code、Codex、Gemini 三 Agent 协作，通过独立会话、文档和接口契约完成 PM/架构、后端、前端分工，适合分享跨模型 Agent 团队如何减少上下文爆炸与角色混乱。 | https://zhuanlan.zhihu.com/p/1989786047844991637 |
| 作者以一人公司常驻 30 个 Agent、4 台机器的实际运行经验复盘无人值守项目，涉及 PM/开发/DevOps 分工、任务 owner、集中状态管理和 Agent 间“对抗”，适合展示多 Agent 团队长期运转后的组织经验与边界。 | https://zhuanlan.zhihu.com/p/2073400762441511561 |
| MateClaw 2.0 用 Lead/Member/Reviewer 常设团队、共享实时任务板、blockedBy 依赖、并行派发、人工审批和失败恢复来组织复杂交付，适合分享“共享事实源 + 状态机”式多 Agent 协作。 | https://zhuanlan.zhihu.com/p/2067367489571382672 |
| 数据研发场景采用 Orchestrator + Specialist 架构，系统讲 Identity/IO/工具约束、按需编排、Spec 文件交接、状态持久化、断点恢复和错误规则化，适合作为 Harness 如何把多 Agent 变成可控工程系统的一线实践。 | https://zhuanlan.zhihu.com/p/2061086733131945183 |
| 钉钉 AI 招聘的真实 Harness 实践给出“Agent 不宜过多”、RPA 事务边界、文件化断点状态、白名单/Linter/独立 Reviewer 三层护栏等生产经验，特别适合用来讲多 Agent 数量、可靠性和安全之间的权衡。 | https://zhuanlan.zhihu.com/p/2058188813529244032 |
| 从多 Agent 底层运行环境切入，讨论容器化 Sandbox 的会话隔离、恶意代码防护、瞬时大规模弹性、长任务状态持久化与成本控制，并结合 OpenKruise Agents，适合补充协作系统真正上线所需的基础设施层。 | https://zhuanlan.zhihu.com/p/2070142074624668518 |
| CodeAgents 从真实 AI Coding 实验总结多编程智能体的关键不是“角色多”，而是任务图、状态流转、上下文和工作区隔离、Patch/Review/Test 门禁与恢复机制，适合讲清楚并行开发的边界治理。 | https://zhuanlan.zhihu.com/p/2030309910798391120 |
| 以 13 个 Agent 从市场研究一路接棒到开发、QA、运营和数据复盘的真实工具站为案例，把每次交接明确成输入、输出、证据和停止条件，并设置 GO/NO-GO 门禁，适合分享大规模串联协作如何避免“交接失真”。 | https://zhuanlan.zhihu.com/p/2043410428491977017 |
| 8 个 Agent 改进 6900 行 UECLI 模块的实测记录同时给出并行条件、不同模型分工、Token/时间收益，以及连续失败后人工升级和同区域不并行的决策，适合用数据讲“何时并行、何时串行、何时人接管”。 | https://zhuanlan.zhihu.com/p/2022398691491815748 |
| 9 个 Agent 从 0 到 1 复刻 Claude Code 的项目复盘强调严格角色边界、正式变更流程、QA/PM 权限隔离、依赖链跟踪和多轮 Review，适合展示研发型多 Agent 团队规模扩大后如何靠流程而非模型能力维持秩序。 | https://zhuanlan.zhihu.com/p/2025864547731272052 |
| TeamAgentX 是直接面向多 Agent 协同的软件项目，作者以自己已基本成型的产品为案例，适合分享“协作能力如何从框架概念变成可使用产品”的项目落地路径。 | https://juejin.cn/post/7671865305111691302 |
| Hermes Kanban 把多 Agent 协作从群聊派活推进到任务化看板，强调任务不丢、依赖关系和流水线式推进，适合作为“共享任务状态比聊天记录更重要”的工程实践案例。 | https://juejin.cn/post/7641571163690139683 |
| 文章专门讨论多 Agent 扩张后的反效果，用“2 个比 1 个强、10 个反而更差”切入协作成本与规模边界，适合分享如何判断 Agent 数量、并行度和组织复杂度是否过度。 | https://juejin.cn/post/7637823184210870307 |
| Open Claude Cowork 用 Electron + React + ACP 做本地多 Agent 桌面应用，以任务为中心统一管理多个 Agent 的对话与协作，适合展示协议化协作和桌面产品化的结合。 | https://juejin.cn/post/7599581687357571122 |
| 从零构建多 Agent DAG 编排系统，直接聚焦任务依赖、执行顺序与协同调度，适合把“多 Agent 为什么需要确定性编排层”讲成可复现的工程实现。 | https://juejin.cn/post/7615229750573547529 |
| 用 Codex 多 Agent 在半天内完成全栈 AI 批改平台，并给出对应 GitHub 项目，适合分享 AI Coding 场景中角色分工、并行开发与真实交付结果。 | https://juejin.cn/post/7641159847205765171 |
| 把 Claude Code 的 Skills、无头模式、CI/CD 与多智能体编排放在同一套高级实践里，适合分享 Coding Agent 如何从交互式助手进入自动化研发流水线。 | https://juejin.cn/post/7624910626885386294 |
| 从 LangChain4j + Spring Boot 视角讲多智能体协调架构，适合补充 Java 团队如何把多 Agent 协作嵌入现有企业技术栈，而不是只依赖 Python 生态。 | https://juejin.cn/post/7634116818672828470 |
| RuFlo 以分布式“蜂群”方式编排大量专业 Agent，适合与 Supervisor/DAG 模式对比，讨论多 Agent 从小团队扩展到大规模并发协作时的组织方式。 | https://juejin.cn/post/7635275904798965814 |
| 临床文献研究场景用 LangGraph 的 AgentState、Reducer 和 Send API 管理跨 Agent 数据流与动态并行，适合用垂直业务案例讲状态合并和并行调度如何落地。 | https://juejin.cn/post/7618764794954989631 |
| myclaude 把 Claude、Gemini、Codex 组成分工明确的 AI 开发团队，是跨模型多 Agent 编排的直接开源案例，适合分享不同模型按职责协作而不是由单一 Agent 包办全部任务。 | https://juejin.cn/post/7610981820637347880 |
| 用 Codex 与 Claude Code 搭建“Agentic Software Factory”，把 GitHub Issue 从细化、隔离 worktree、实现、测试、Review 到自动开 PR 串成无人值守流水线，适合分享多 Agent/Agentic Coding 如何从人工委派升级为可重复的软件交付系统。 | https://www.youtube.com/watch?v=AbpyqAfxZ8c |
| Claude Code 多 Agent 编排实战同时启动多个专业 Agent，每个 Agent 拥有独立上下文、模型与任务，并结合 Tmux、E2B Sandbox 和全链路可观测性管理并行执行，适合讲清“并行团队 + 隔离环境 + 可观测”如何一起落地。 | https://www.youtube.com/watch?v=RpUTF_U4kiw |
| 独立开发者展示同时运行约 6 个 Coding Agent 的真实工作流，包含 Plan Mode、跨项目/同项目并行、终端会话管理和人工 Review，适合用来讨论多 Agent 提效时的人机分工、注意力调度与安全边界。 | https://www.youtube.com/watch?v=dDeoblrGRGM |
| VS Code 官方演示统一管理本地、后台、云端以及 Claude、Codex 等多种 Coding Agent，并支持并行子 Agent 与独立上下文，适合分享 IDE 如何演进成多 Agent 研发控制台。 | https://www.youtube.com/watch?v=BsAHunfVwNs |
| 以 PR Review 为目标从零搭建生产导向的多 Agent 系统，覆盖安全/质量/测试/文档专业 Agent 的 fan-out/fan-in、结构化输出、RAG、置信度与 HITL、超时重试、幂等和可观测性，适合做“可靠性优先”的完整工程案例。 | https://www.youtube.com/watch?v=RiN02OXjeeQ |
| Google Cloud ADK 的实操型多 Agent 编排分享围绕任务拆解、Agent 通信、工作流协同与治理展开，并强调可扩展的协作系统设计，适合作为 ADK 体系下从概念到实践的案例素材。 | https://www.youtube.com/watch?v=Y3IFeghhT9Y |
| WorkBuddy 把多人和多 Agent 放进同一团队空间，直接涉及权限、评论与版本管理，适合分享“多 Agent 落地不只是编排，还需要人/Agent 协作治理与协作资产管理”的产品化实践。 | https://www.bilibili.com/video/BV1yQgP6JEfX |
| TabTin 以开源、可一键部署的团队 Agent 工作台为切入点，适合展示多 Agent 能力如何从脚本或 CLI 走向团队共享入口和可部署产品。 | https://www.bilibili.com/video/BV1fk8i65EUt |
| 4TORM 围绕长期 Agent 设计独立对话、会议、工作室、工作流和自动任务，让同一批 Agent 能按任务重新组合，同时保留人类掌控方向，适合讲“长期 Agent 团队”如何从聊天窗口演进成协作系统。 | https://www.bilibili.com/video/BV1ae8A6rE7z |
| Nexent 是开源多 Agent SDK 与平台，可自主规划 Agent 和工具，并把模型配置、数据处理、知识库、互联网查询、多模态、MCP 扩展及 Docker 部署串到一起，适合分享平台化 Agent 编排如何真正落地。 | https://www.bilibili.com/video/BV1ZejSzcEEk |
| 用 6 个 Agent 组建一人公司的 AI 联合创始团队，并打通飞书、微信和 Obsidian 知识库，是一个很具体的“小团队业务运营 + 多 Agent + 外部系统集成”案例，适合展示协作从 Demo 进入日常工作流。 | https://www.bilibili.com/video/BV1spNn62EBb |
| 360 智能体工厂把多 Agent 以“蜂群/拉群组队”的方式产品化，并强调自然语言搭建，适合补充低代码平台如何降低多智能体团队构建和协作门槛。 | https://www.bilibili.com/video/BV1B8tJzrEah |
| 阿里 CoPaw 的多角色智能体工作流演示聚焦角色化 Agent 如何串成工作流，适合在分享里补充“角色分工 + 流程编排”这一类轻量多 Agent 落地方式。 | https://www.bilibili.com/video/BV1nRXfBNEYo |
| Microsoft ISE 基于大型零售客户的生产聊天系统，比较 Router、Group Chat、Coordinator 等模式并给出延迟与耦合权衡，适合分享“多 Agent 微服务化后如何选编排拓扑”以及为什么最终采用 Coordinator。 | https://devblogs.microsoft.com/ise/coordinator-patterns-multi-agent-systems/ |
| Microsoft ISE 在真实 A2A 项目中比较共享存储、消息内嵌和 Agent 自持状态三种上下文传递方式，并落地“协调器推送摘要上下文 + 无状态域 Agent”，适合讲跨服务 Agent 协作的上下文、安全和独立部署边界。 | https://devblogs.microsoft.com/ise/a2a-context-passing-multi-agent-systems/ |
| Microsoft ISE 结合大型电商场景总结几十到上百 Agent 扩展时的 Agent 选择、LLM 调用控制、延迟和可扩展性问题，适合补充多 Agent 从小规模编排走向规模化服务目录后的工程设计。 | https://devblogs.microsoft.com/ise/multi-agent-systems-at-scale/ |
| AG2 用尽调场景把 Seed Crawler、6 个并行专业 Agent、Validator、Synthesis 与 Q&A 串成完整流水线，并明确单 Agent 失败时如何隔离、如何做结构化交付，适合作为 fan-out/fan-in + 质量校验的可复现案例。 | https://www.ag2.ai/blog/due-diligence-with-tinyfish |
| Agent Squad 是成熟的开源多 Agent 编排框架，既支持基于意图的路由与上下文管理，也提供 SupervisorAgent 做并行专业 Agent 协调和分层团队，适合展示从“选一个 Agent”进化到“一个 Agent 团队协作”的框架实现。 | https://github.com/2FastLabs/agent-squad |
| Microsoft ArgusAgent 把 Manager、Planner、Engineer、Reviewer 四类权限角色固化成可长期运行的多 Agent Runtime，任务、证据和 checkpoint 可跨会话保留，并要求独立 Reviewer 验证，适合分享长任务协作如何做到可恢复、可审查。 | https://github.com/microsoft/ArgusAgent |
| Microsoft Build 2026 的 DEM312 让 Researcher、Content Creator、Podcaster 三个 Agent 通过 A2A 协作，并在 Azure Container Apps 中做隔离执行和 Application Insights 观测，适合作为“跨框架、跨模型、多服务”部署级演示。 | https://github.com/microsoft/Build26-DEM312-multi-agents-in-action-with-3-ai-agents-3-frameworks-tools-models |
| Azure 的可部署实现把 3 个 Agent 做成内容工厂，从研究到多格式内容再到播客，并同时覆盖 Container Apps、Sandbox、Agent 注册、评估与可观测性，适合分享从协作逻辑到云端运行环境的一整套落地链路。 | https://github.com/Azure-Samples/azure-container-apps-multi-agent-workflow |
| Microsoft 的 Multi-Agent Reference Architecture 基于客户生产方案沉淀 Orchestrator、Classifier、Agent Registry、Memory、A2A/MCP、Observability、Evaluation、Security 与 Governance，适合做分享中的企业级总体架构基线。 | https://github.com/microsoft/multi-agent-reference-architecture |
| Azure Agentic Advisor 用 Planner 按需并行触发 News、SEC、Stock 等专业 Agent，并结合图关系、RAG、Mem0 持久记忆和 Phoenix Trace，适合展示“只运行必要 Agent + 业务数据层 + 可观测”的垂直生产型参考实现。 | https://github.com/Azure-Samples/postgres-agentic-advisor |
| Azure AgenticShop 用 LlamaIndex Workflow 组织 Planning、Personalization、Inventory、Review、Evaluation、Presentation 等 Agent，采用事件驱动、持久记忆、超时降级和流程可视化，适合演示松耦合多 Agent 在电商场景的落地。 | https://github.com/Azure-Samples/postgres-agentic-shop |
| frisian-mcp 的生产案例直接披露 194 个 worker 记录、50 个项目、17,000 个知识块以及多角色并发工作流，所有任务分配、讨论、决策、审批和检索都经 MCP 暴露给 Agent，适合展示“面向 Agent 设计的协作基础设施”真实运行形态。 | https://frisian-mcp.com/tech/test-cases/pre-release/production-consumer-case-study |
| Oracle 给出正在真实运行的企业多 Agent 平台：Orchestrator 通过 A2A 调度 HCM、MCP 集成和任意框架 Agent，并用 Agent Card 自动发现、并行调用、Langfuse 全链路观测与回归评估，适合讲“跨框架 Agent 平台如何动态扩容且保持可运维”。 | https://blogs.oracle.com/ai-and-datascience/building-a-dynamic-multi-agent-enterprise-platform |
| IBM 用 BeeAI + watsonx Orchestrate 搭建可扩展、可观测的预测性维护多 Agent 工作流，把故障预测、维护决策和任务调度串成端到端工业流程，适合作为“多 Agent 直接进入设备运维业务闭环”的实操案例。 | https://developer.ibm.com/tutorials/automated-predictive-maintenance-beeai-watsonx-orchestrate/ |
| IBM 演示把 CrewAI 多 Agent 服务部署到 Code Engine，再作为外部 Agent 接入 watsonx Orchestrate，适合分享“不同框架独立部署、通过平台统一编排”的跨框架生产集成方式。 | https://developer.ibm.com/tutorials/integrate-crew-ai-agents-watsonx-orchestrate/ |
| NVIDIA 用 Signal、Code、Evaluation 三个专业 Agent 构成“提出信号→生成代码→回测评估→迭代改进”的闭环，并由 NeMo Agent Toolkit 保持 handoff 上下文，适合展示多 Agent 如何把业务研究变成可执行、可验证的迭代系统。 | https://developer.nvidia.com/blog/automating-and-optimizing-financial-signal-discovery-with-multi-agent-systems/ |
| NVIDIA AI-Q 2.0 直接给出生产级部署路径：意图路由器协调浅层/深层研究 Agent，子 Agent 共享文件系统并在沙箱执行 Skills，同时用 Terraform + Helm 部署到 OKE 并接入 Vault，适合讲协作逻辑怎样落到真实云基础设施。 | https://developer.nvidia.com/blog/deploy-a-production-ready-nvidia-ai-q-blueprint-on-oracle-cloud-infrastructure/ |
| CAID 研究把多 Agent 软件开发建立在中心任务分解、异步并行、隔离工作区和 branch/merge/test 验证上，并在 PaperBench、Commit0 上显著优于单 Agent，适合用实验数据说明“Git 协作原语为什么能让 Coding Agents 更可靠”。 | https://arxiv.org/abs/2603.21489 |
| 这份生产级 Agentic AI 工程指南覆盖任务分解、多 Agent 设计模式、MCP、确定性编排、容器化部署、Responsible AI 与完整案例，适合作为分享里从架构设计到上线运维的一份端到端工程清单。 | https://arxiv.org/abs/2512.08769 |
| CoffeeAGNTCY 是 AGNTCY 的端到端多 Agent 参考应用，组合 A2A、SLIM/NATS、LangGraph、MCP、身份与可观测性，并提供两 Agent 与更复杂群组通信两套可运行配置，适合展示跨框架 Agent 基础设施如何真正拼成系统。 | https://github.com/agntcy/coffeeAgntcy |
| Edict 用“规划→审核→派发→并行执行”的强制分权流程组织 12 个专业 Agent，并配套状态流转校验、实时看板、任务叫停/恢复、完整审计与健康监控，适合作为“多 Agent 协作如何靠制度化质量门禁和可观测性提高可控性”的工程案例。 | https://github.com/cft0808/edict |
| 这份 Claude Code Agent Teams 实战指南不仅给出 PR 并行评审、竞争假设调试、跨层功能开发等可复用团队模式，还系统记录 plan approval、Hooks 质量门禁、文件冲突、Token 成本和团队规模边界，特别适合分享“什么时候该用团队、怎么避免协作反噬”。 | https://github.com/cobusgreyling/claude-agent-teams |
| agent-teams-rs 用 Rust 抽象出 lead/teammate、任务 DAG、邮箱通信、跨 Claude/Codex/Gemini 后端、共识投票、HITL、Git notes checkpoint 与成本看板，适合展示跨模型 Agent 团队如何把协作、审计和路由做成通用基础设施。 | https://github.com/ZhangHanDong/agent-teams-rs |
| Codex Team Orchestrator 把控制面、专业 worker、共享消息总线、任务板、产物交换与仲裁放进统一 runtime，并提供自适应 fan-out、结构化可观测、回放和 benchmark 工具，适合拆解 Coding Agent 团队如何从“并行开几个会话”走向可验证编排。 | https://github.com/ajjucoder/codex-team-orchestrator |
| Society Protocol 通过 P2P 网络让不同机器、不同团队中的 Agent 自动发现、通信和共享知识，并结合密码学身份、MCP 接入与按复杂度动态组建临时团队，适合分享去中心化、跨边界 Agent 协作的协议层落地思路。 | https://github.com/societycomputer/society-protocol |
| LlamaIndex 的 Multi-agent Concierge 用 orchestrator 路由多个专业 Agent，以共享全局状态处理跨任务依赖，并对高风险工具调用加入人工确认，适合用一个完整业务流程说明“专业化分工 + 状态共享 + HITL”怎样组合成可控多 Agent 服务。 | https://github.com/run-llama/multi-agent-concierge |
| 物流全流程案例用 5 个专业 Agent 分别承接报价、订舱、报关、跟踪等环节，强调先把人工业务流程与接口契约画清再逐步 Agent 化，适合分享“多 Agent 不是堆角色，而是把组织协作翻译成可执行接口”的业务落地方法。 | https://zhuanlan.zhihu.com/p/2070144231914714259 |
| 用 AgentTeams 从部署到代码审查团队完整演示多智能体协作，把审查任务拆给不同角色并给出可复现配置步骤，适合用作“从框架安装到第一个真实团队场景”的入门实战案例。 | https://zhuanlan.zhihu.com/p/2067014859225556471 |
| 作者实际搭建 10 个 OpenClaw AI 助理，以总助理负责分发、多个专业 Agent 使用独立工作区/记忆/技能并通过飞书协同，适合展示面向日常运营的一人多 Agent 团队如何长期组织。 | https://zhuanlan.zhihu.com/p/2013748116243894716 |
| JiuwenClaw AgentTeam 采用 Leader 动态组队、Teammate 并行执行、共享工作区与文件锁，并加入关键决策审批、故障恢复和实时可观测，适合分享“协同工程”如何把组织分工和运行治理一起产品化。 | https://zhuanlan.zhihu.com/p/2031476922773938489 |
| 从 Managed Agents 原理延伸到开源 Multica 平台，把 Agent 模板、隔离运行环境、Session 生命周期和不可篡改 Events 统一管理，适合讲人类与 Claude Code/Codex/OpenCode 等异构 Agent 如何进入同一团队协作层。 | https://zhuanlan.zhihu.com/p/2046904024838976352 |
| 用 LangGraph 搭建一个 Supervisor + 航班/酒店/行程三个专家 Agent，明确工具隔离、动态路由和完成条件，适合作为层级式多 Agent 的最小可运行教学案例。 | https://zhuanlan.zhihu.com/p/2074148599408207180 |
| OpsPilot Zero 用 4 个 Agent、7 个 Skill 跑零人工运维闭环，并从任务拆解、上下文传递、验证证据、安全审计与可回滚等角度总结 9 条工程判断，适合分享生产级协作系统应该具备哪些“可信闭环”条件。 | https://zhuanlan.zhihu.com/p/2066174785340564426 |
| Solon AI Harness 以技术文章生产为例，把调研、规划、写作、校对拆成带依赖的子 Agent DAG，并展示 task/multitask 委派和独立心智/记忆，适合讲主 Agent + 子代理如何落成可控任务编排。 | https://zhuanlan.zhihu.com/p/2058632024878059824 |
| 文章反向提醒“先别急着做 Agent Teams”，从真实 P0 故障处理说明多 Agent 前必须先补齐状态、运行环境、权限与 Timeline 等 AgentOS 闭环能力，适合分享中用于解释为什么协作规模化前要先解决单 Agent 的可恢复与可审计。 | https://zhuanlan.zhihu.com/p/2045902247062769700 |
| 文章直接以 Multi-Agent 协作模式与工程实践为主题，适合用来补充角色分工、协作机制与工程化约束的整体视角，便于在分享中建立从“为什么拆 Agent”到“怎样组织协作”的框架。 | https://juejin.cn/post/7668220205333790729 |
| 用 NestJS + LangGraph 从零搭建多 Agent 编排平台，包含 Supervisor 自动路由、DAG 工作流、RAG 与多租户隔离，适合展示多 Agent 如何从流程 Demo 走向可复用平台。 | https://juejin.cn/post/7605780454579929139 |
| 从零构建 6 个 Agent 的自动化研发流水线，覆盖需求分析、架构、代码、测试到部署，并明确讨论状态流转、Agent 通信与 LLM 不稳定问题，适合拆解 AI Coding 团队的端到端协作。 | https://juejin.cn/post/7650719111396720682 |
| 在多 Agent 架构跑通后继续补齐 FastAPI、SSE 与 Vue3 交互层，把命令行系统做成用户可用产品，适合分享“协作逻辑之外，产品化落地还需要什么”。 | https://juejin.cn/post/7643757385782411290 |
| 用不足百行可运行代码实现研究员、批评家、总结员协作，覆盖共享状态、条件边、循环与收敛机制，特别适合作为分享现场可快速讲清核心概念的最小示例。 | https://juejin.cn/post/7639756570968653860 |
| 以 DevOps 团队为场景，用 Manager Agent 编排专业 Agent，并结合 MCP、A2A 与人工反馈控制，适合展示跨工具、跨 Agent 协作怎样形成可治理的业务闭环。 | https://juejin.cn/post/7612949440031653888 |
| 直接让一个 Agent 调度 Claude Code、Codex、Gemini 并行执行任务，并给出对应开源项目 mco，适合分享异构 Coding Agent 如何统一编排、并发协同。 | https://juejin.cn/post/7611095964389687342 |
| 从 Managed Agents 原理讲到 Multica 实践，重点把 AI Agent 放进真实人机团队中作为正式协作成员，适合讨论“Agent 团队”从纯自动编排走向人与 Agent 共事的工程形态。 | https://juejin.cn/post/7647847267789832192 |
| 用 OpenClaw + 飞书搭建 5 个可协作 AI 助理，强调独立工作空间与专业化分工，适合展示低门槛、多角色 Agent 团队如何嵌入日常协作工具。 | https://juejin.cn/post/7609481369001492499 |
| 对比 Multi-Agent Routing 与 Sub-Agents 两种机制，解释多角色系统如何路由、委派和协作，适合作为分享中的架构选型案例，避免把所有“多 Agent”都当成同一种实现。 | https://juejin.cn/post/7613970761352462336 |
| 用 LangGraph 演示拆分并行汇总、Supervisor 调度以及生成-评估-返工等多 Agent 协作方式，适合用同一框架对比不同协作拓扑和控制策略。 | https://juejin.cn/post/7602824293165170703 |