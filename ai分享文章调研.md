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
| Agyn 直接以“团队式自主编码”为目标构建多 Agent 系统，适合展示 Coding Agent 如何从单实例执行升级为角色化团队协作与自主交付。 | https://www.youtube.com/watch?v=STkRrB9GQHo |
| Devfest25 以生产成功模式为主题讲多 Agent 工程，适合提炼分享中的生产化设计清单，避免只停留在 Demo 级编排。 | https://www.youtube.com/watch?v=GnMQvN10Vvs |
| 内容专门聚焦多 Agent 的 Context Engineering，适合讲解上下文隔离、传递与组织为什么是协作质量和稳定性的关键基础。 | https://www.youtube.com/watch?v=mbVVFEFhunQ |
| Chi Wang 与 Xiao Ma 围绕生产环境中的多 Agent 架构展开讨论，适合补充从框架概念走向真实生产部署后的架构取舍与经验。 | https://www.youtube.com/watch?v=oVQi6LIHWtg |
| Codex 多 Agent 工作流直接结合 Git Worktrees，适合展示并行 Coding Agent 如何用独立工作区减少文件冲突并形成可复用研发协作方式。 | https://www.youtube.com/watch?v=fVdBEgVE0wI |
| VS Code 官方演示多个 Agent 的统一编排与协作，适合展示 IDE 如何成为多 Agent 开发控制面，并用于讲人类如何监督并行 Agent。 | https://www.youtube.com/watch?v=AtaehXB4hPQ |
| AWS re:Invent 2025 直接讲用 A2A 与 MCP 构建可扩展、自编排 AI 工作流，适合分享协议化 Agent 通信、工具连接与规模化协作怎样组合落地。 | https://www.youtube.com/watch?v=9O9zZ1lQWiI |
| Google Cloud 官方用 A2A 与 Agent Registry 搭建多 Agent 系统，适合讲 Agent 发现、注册与跨服务协作如何从点对点调用走向平台化治理。 | https://www.youtube.com/watch?v=-MME36Ft9Gc |
| Microsoft Developer 从 Foundry 视角展示 AI 自动化与多 Agent 编排，适合补充企业平台层如何把 Agent 团队、工作流和生产运行能力统一承载。 | https://www.youtube.com/watch?v=v1Q7rEE3StM |
| Pi Agent 的 pi-dynamic-workflows 会按当前任务动态生成 workflow，把复杂任务拆给相互隔离的子 Agent，并以并行、串联、汇总组成可观察执行链，适合分享“动态规划 + 隔离执行 + fan-out/fan-in”如何落地。 | https://www.bilibili.com/video/BV11yM96nE45 |
| ClawSwarm 是开源多 Agent 编排系统，用 FastAPI 调度中心、Vue 管理端和 OpenClaw 通道插件组织专业 Agent 讨论、分工与执行，适合展示 hub-and-spoke 式群体协作如何产品化。 | https://www.bilibili.com/video/BV1WuDZBFE8Q |
| Taogen 的 p2p-english-class 让 teacher、student、memory-coach 三个 Agent 通过 coms-net 跨设备 P2P 协作，并验证发现、消息收发、自动回复和证据校验，适合讲去中心化 Agent 通信与真实业务状态传递。 | https://www.bilibili.com/video/BV1PT7C6gEs7 |
| herdr + Tmux 实践聚焦跨 Agent、跨 Session 通信，适合补充多 Agent 协作里经常被忽略的“通信基础设施层”，尤其是并行 CLI Agent 如何交换消息与协调工作。 | https://www.bilibili.com/video/BV1suuj6CEje |
| AionUi 自动识别 Claude Code、Codex、Gemini CLI 等 20+ CLI Agent，并在统一桌面里支持并行派活、组队协作和远程控制，适合展示异构 Coding Agent 如何从多个终端升级成统一协作工作台。 | https://www.bilibili.com/video/BV1TDbk6gEBU |
| Microsoft 最新多 Agent 架构指南把 MCP 与 A2A 的边界、最小权限、审计治理、类型化跨 Agent 载荷和人工审批放在同一套实践建议中，适合作为“企业多 Agent 如何安全互联”的设计基线。 | https://learn.microsoft.com/en-us/agents/architecture/multi-agent-patterns |
| Azure Architecture Center 系统比较 sequential、concurrent、group chat、handoff、magentic 等编排模式，并明确每种模式的适用与避用条件以及“优先最低复杂度”原则，适合直接整理成协作架构选型表。 | https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns |
| igot.ai 在 Google Cloud 上把多 Agent 编排跑到企业规模，披露 GKE 上多 Agent workflow 约 95% 完成率以及证据发现效率提升 300%+，适合用真实指标说明协作系统的工程化收益。 | https://cloud.google.com/customers/igot |
| EVERSANA 用协调式多 Agent 系统把品牌策略、洞察、全渠道规划与内容生产串成 AI Agency Platform，并保留精简的人类监督团队，适合分享知识工作型多 Agent 如何进入大型企业业务流程。 | https://cloud.google.com/customers/eversana |
| OrionStar 将语音、视觉、认知和决策能力统一进 AgentOS，让机器人通过多 Agent 编排获得团队式协同能力，同时用子 Agent 提升研发效率，适合补充“多 Agent 不只做软件流程，也能进入机器人实体协作”的案例。 | https://cloud.google.com/customers/orionstar |
| 这项 2026 年工业研究把工作流式多 Agent 架构用于真实汽车供应商生产计划，在 1000+ 周订单和多条装配线约束下做 7 组实验并与人工规划对照，适合作为“多 Agent 进入硬业务优化”的量化案例。 | https://www.sciencedirect.com/science/article/pii/S0007850626000478 |
| 论文从大型互联网公司的真实多 Agent 运行经验提炼角色定义、消息路由、业务流程、状态管理和错误处理等可复用设计模式，并讨论拥塞、依赖故障与非确定性输出，适合补充生产级设计模式视角。 | https://papers.ssrn.com/sol3/papers.cfm?abstract_id=6973623 |
| Gas City 把多 Agent Coding 的可复用基础设施做成声明式编排 SDK，提供 tmux/subprocess/ACP/Kubernetes 等运行时、工作路由、健康巡检和 controller/supervisor reconciliation，适合展示 Agent 团队如何走向平台控制面。 | https://github.com/gastownhall/gascity |
| AWS 官方 Guidance 用 Supervisor + 订单、推荐、排障、个性化等专业 Agent 跑通完整客服架构，并同时给出会话存储、Knowledge Base、Athena/Action Group、部署与成本结构，适合做可复现的企业业务落地案例。 | https://github.com/aws-solutions-library-samples/guidance-for-multi-agent-orchestration-on-aws |
| HKU DeepCode 是面向代码生成的开源 Multi-Agent 项目，仓库包含独立的 agent orchestration engine，并把项目、会话、目标、工具活动与验证统一进可交互工作台，适合展示多 Agent Coding 从算法到产品形态的实现。 | https://github.com/HKUDS/DeepCode |
| AWS 的真实生产重构案例从 3 万输入 Token、层层“DO NOT”补丁的多 Agent 架构出发，用结构化 Agent 间协议、薄路由和职责拆分把 Token 消耗约降 70%、延迟约减半，适合分享“多 Agent 技术债怎样重构”的具体方法。 | https://builder.aws.com/content/3AfCno58Bsm4AbKaFKYiaDgeOkK/from-prompt-spaghetti-to-structured-multi-agent-systems-lessons-from-refactoring-a-production-ai-agent-architecture |
| Salesforce 2026 年 8 月已把 Multi-Agent Orchestration 推到 GA：统一 orchestrator 负责拆任务、读 Agent 描述、路由与结果汇总，并继续扩展 A2A 第三方 Agent，适合追踪企业多 Agent 从 Beta 走向正式产品化后的架构形态。 | https://www.salesforce.com/blog/multi-agent-orchestration/ |
| Salesforce 的生产可观测实践把 subagent 委派完整串进 waterfall/graph trace，并能按 subagent、action、intent 切分质量、RAG、健康指标，适合补充“多 Agent 出问题后怎样定位是哪一次 handoff/调用导致”的运维视角。 | https://www.salesforce.com/blog/agentforce-observability/ |
| 这项工程行业案例把专职 ReAct tool agents 与战略推理 Agent 组合，用于仿真驱动产品开发，并从真实工程师工作坊查询中提炼模型选择、提示、工具集成、协作与评估设计模式，适合做“行业工作流如何抽象多 Agent 模式”的研究型案例。 | https://www.sciencedirect.com/science/article/pii/S2212827126008061 |
| 2026 年 8 月制造业研究把多 Agent 用于把新产品需求动态映射到制造能力，并专门设计 Agent 间通信策略来识别和协商生产参数，适合补充“多 Agent 如何在变化环境里做任务规划与资源协调”的实体业务案例。 | https://www.sciencedirect.com/science/article/pii/S0736584526000244 |
| 该工业案例讨论服务数十万终端用户的大规模多 Agent 系统，直接覆盖角色定义、通信协议、业务流程与运行架构，适合分享中作为“多 Agent 真正上规模后如何组织和运营”的生产证据。 | https://papers.ssrn.com/sol3/papers.cfm?abstract_id=6795198 |
| Google Cloud 的 2026 生产指南把长任务 checkpoint/resume、Agent 身份与网关治理、ADK 多 Agent 编排、A2A+MCP 互操作和可复用 Agent Blueprint 串成一套完整路径，适合整理成“从 Demo 到生产”的分享主线。 | https://cloud.google.com/blog/topics/developers-practitioners/five-guides-to-building-and-scaling-production-ready-ai-agents |
| Microsoft Foundry 从生产运行层总结 Agent 的隔离沙箱、持久状态、跨框架部署和 OpenTelemetry 全链路追踪，其中每次 sub-agent hop 与 handoff 都进入统一 Trace，适合讲多 Agent 上线后运行时与可观测基础设施如何补齐。 | https://devblogs.microsoft.com/foundry/agent-service-build2026/ |
| 该 Claude Code 多 Agent 项目把 planner、supervisor、worker 拆成明确角色，用 Git worktree 隔离、DAG 调度、接口契约、风险评分、冲突检测、状态恢复、worker mailbox 和验证套件组成完整并行研发控制面，适合做可复现的 Coding Agent 协作案例。 | https://github.com/SynBioExplorer/Claude_Code_agentic_coding |
| Gaia 从安全视角实现事件驱动的多 Agent 编排：在路由、工具调用和 handoff 各阶段做风险分级、权限/同意门禁、上下文注入、handoff contract 校验、审计与记忆沉淀，适合分享“多 Agent 不只会协作，还要限制彼此能做什么”。 | https://github.com/metraton/gaia |
| OpenHive 把 Queen + worker colony 做成面向生产业务流程的多 Agent Harness，通过共享 tracker、持久计划、动态 fan-out、crash-safe park/resume、预算约束、可观测与 HITL 形成完整运行闭环，适合分享“模型之外的 Harness 才是生产可靠性的关键”这一落地思路。 | https://github.com/aden-hive/hive |
| Gas Town 把多 Coding Agent 协作做成持久工作区：用 Git-backed 状态、mailbox/handoff、任务 Convoy、watchdog、merge queue、升级机制与并发调度管理数十个 Agent，适合拆解长期并行研发中“状态、恢复、合并、监控”如何工程化。 | https://github.com/gastownhall/gastown |
| Microsoft 的 Multi-Agent Custom Automation Engine 是可直接部署到 Azure 的业务自动化参考实现，用专业 Agent 协同完成计划、执行和校验，并配套 Container Apps、Cosmos DB、Foundry、安全与多类真实业务场景，适合作为企业方案从架构到部署的完整案例。 | https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator |
| Squad 把 GitHub Copilot Agent 团队直接放进代码仓库，用持久角色知识、共享决策、并行执行、Issue 轮询派发、checkpoint/state backend 和分级故障恢复保持人类主导，适合分享“Agent 团队如何嵌入现有 GitHub 研发流程并长期运转”。 | https://github.com/bradygaster/squad |
| Agent Teams AI 把 Claude Code、Codex、OpenCode 等多种运行时统一到桌面协作层，提供 Agent 自主建任务/互相评审、看板、跨团队通信、worktree、逐任务日志、预算与人工审批，适合展示多 Agent 从 CLI 编排走向团队可视化产品的形态。 | https://github.com/777genius/agent-teams-ai |
| AutoScientists 用去中心化、自组织 Agent 团队进行长时间科学实验：Agent 会围绕假设动态组队、互相批判方案、共享成功与失败以减少重复探索，并给出多个基准上的量化提升，适合用来讲“中心调度之外的自组织协作”及其验证方式。 | https://github.com/mims-harvard/AutoScientists |
| Wegent 把本地 Coding Agent、远程执行机和自托管 Web/Backend 放进统一 project space，通过任务板、共享文件、自动化、执行历史和交付状态连接多人/多 Agent 工作，适合补充“协作落地最终需要项目空间与执行基础设施，而不只是 Agent 对话”。 | https://github.com/wecode-ai/Wegent |
| Pi Agent Teams 把 leader/teammate、共享任务列表、依赖图、文件邮箱、紧急消息、计划审批、worktree 隔离、模型策略和完整生命周期都做成可调用的团队控制面，适合直接拆解一个可操作的多 Agent 协作协议。 | https://github.com/tmustier/pi-agent-teams/blob/main/skills/agent-teams/SKILL.md |
| 这份 Agent Team Orchestration Skill 明确定义 Orchestrator/Builder/Reviewer/Ops 角色、任务状态机、标准 handoff 内容、交叉 Review、checkpoint 与失败降级，适合在分享中提炼一套不依赖具体模型的“团队协作规约”。 | https://github.com/edisonzerolam/agent-team-orchestration/blob/main/SKILL.md |
| Claude Code Agent Teams 的实现记录用独立完整上下文的 teammate + 共享任务列表，在 tmux 中由 Lead 实际构建出一个 Command→Agent→Skill 的工作流，适合做“原生 Agent Team 从启动、分工到产物落地”的小型可复现实操。 | https://github.com/shanraisshan/claude-code-best-practice/blob/main/implementation/claude-agent-teams-implementation.md |
| openJiuwen 的企业级蜂群架构已在邮储金融生产环境落地，覆盖存量系统无侵入接入、多租户共享与强隔离、弹性调度、全链路审计及真实办公/监测/风控场景，适合作为“多 Agent 如何进入强监管生产系统并规模化运行”的案例。 | https://zhuanlan.zhihu.com/p/2069042095915000471 |
| 企业 AI 软件工程 3.0 的一线分享把 Pod 式小组、全角色专属 Agent、多人+多智能体融合协作、代码库权限隔离和自动故障处理串在一起，适合补充“人和 Agent 混合组织如何真正改造团队研发流程”。 | https://zhuanlan.zhihu.com/p/2075979843955594765 |
| 文章系统讨论企业多 Agent 的 Orchestrator、任务编排、共享上下文/记忆、并行处理、独立审查和权限边界，适合用来总结从单 Agent 升级为受控协作系统时必须补齐的最小工程要素。 | https://zhuanlan.zhihu.com/p/2069843019549955634 |
| 制造业实践从“万能 Agent”转向调度、质检、运维等专业 Agent 集群，并强调确定性规则与大模型混合、独立部署和闭环协作，适合展示实体工业场景里的角色拆分、实时性与故障隔离权衡。 | https://zhuanlan.zhihu.com/p/2072736473464452198 |
| 华为/openJiuwen 从 Coordination Engineering 与 Agent Scale Out 出发，涉及分布式动态调度、多模型路由、组织级记忆、多租户隔离、断点续跑和性能指标，适合讲多 Agent 如何从单机编排演进为企业级分布式平台。 | https://zhuanlan.zhihu.com/p/2062110218641732550 |
| 淘宝直播基于 DataWorks Data Agent 重构真实数据研发流程，以规划/职能/报告 Agent 组成“数字工厂”，并结合人工 Confirm、标准化六环节、中心化/本地化双轨和持久记忆，适合作为大型团队多 Agent 流程再造案例。 | https://zhuanlan.zhihu.com/p/2060042247660474513 |
| 用“Agent 军团 + 共享黑板”重构混沌工程，把无人干预、半小时内闭环、持续发现未知缺陷和结果可复用设为量化验收目标，适合分享共享状态、多 Agent 自动诊断与业务指标如何一起设计。 | https://zhuanlan.zhihu.com/p/2068687030658273687 |
| 从 Agent→Agent、User→Agent→Agent、Agent→MCP 等真实协作链切入身份与权限传递，适合补充多 Agent 上生产后经常被忽略的最小权限、身份边界、跨 Agent 授权和审计治理。 | https://zhuanlan.zhihu.com/p/2063930333976847060 |
| 专门讨论多智能体路由评测从单一完成率扩展到效率、质量、成本三维，并比较 RouterBench、BFCL V4、MasRouter 等方法，适合分享如何验证 Orchestrator 是否真的“选对 Agent”，而不只看最终答案。 | https://zhuanlan.zhihu.com/p/2040547723640842033 |
| 优刻得的团队级 AI 编程方案把规划、执行、批量、审查 Agent 分工与多模型路由、独立审查、客观验证、最小权限和风险分级结合，适合作为“多 Agent 同时优化质量、成本与治理”的研发落地案例。 | https://zhuanlan.zhihu.com/p/2054975440863998381 |
| 从零构建 Coding Agent 的团队篇把 Teammate 设计为长期存在、带独立上下文/邮箱/名册的队友，并给出 TeamManager、持久化、独立 Agent Loop 与 Subagent/Runtime Task 的边界，适合用来讲“多 Agent 团队到底要补哪些基础机制”。 | https://juejin.cn/post/7627665051668414479 |
| HiClaw 以开源 Agent Teams 系统把 Manager、无状态 Worker、Matrix 房间、AI Gateway、MinIO、MCP、心跳检查和 Human-in-the-Loop 串成可部署工作流，适合展示多 Agent 从聊天编排走向可观察、可干预的团队基础设施。 | https://juejin.cn/post/7614747451153743924 |
| AI Mind 的真实项目复盘采用 Supervisor→Plan→Task→并行 Reviewer→返修链路，并用 strict schema contract、Runtime 硬规则和有限重试把 LLM 不确定性锁在可控范围内，特别适合分享“多 Agent 可靠性不能只靠 Prompt”。 | https://juejin.cn/post/7669221547006230574 |
| OpenPencil 在真实设计工具中实现 Agent Team，把一句 UI 需求拆成多个子任务交给不同 Agent 协作生成，适合分享“多 Agent 如何嵌入既有产品功能并完成面向用户的实际交付”，而不只是框架 Demo。 | https://juejin.cn/post/7614336660986544154 |
| Google Cloud 把 1000 个真实 ADK Agent 同时运行在开源生产级 Race Condition 系统中，并复盘 HTTP/A2A 在规模化时为何切到 Redis Pub/Sub、重试退避、上下文缓存、Skills 懒加载与 Agent mesh，特别适合分享“大规模多 Agent 真跑起来后架构如何改”。 | https://www.youtube.com/watch?v=WSIzaih2vq4 |
| Google Cloud 从本地开发到 Cloud Run 生产部署完整搭建分布式多 Agent 系统，用 Researcher、Judge、LoopAgent、SequentialAgent 形成自纠错流水线，并通过 A2A 连接服务，适合做“专业分工 + 反馈闭环 + 云端部署”的实操案例。 | https://www.youtube.com/watch?v=VjBijrS19gY |
| Microsoft Developer 官方介绍 Agent Framework 如何把 Semantic Kernel 与 AutoGen 汇合为生产级多 Agent SDK，覆盖状态/记忆、企业基础能力、Team Conversation 与后续 Workflow 编排，适合讲微软体系从实验框架走向统一生产栈。 | https://www.youtube.com/watch?v=AAgdMhftj8w |
| Microsoft Azure Developers 直接用 Agent Framework 的图式编排搭建多 Agent workflow，并展示 streaming、checkpointing、human-in-the-loop 等生产能力，适合分享“确定性工作流如何约束 Agent 协作”。 | https://www.youtube.com/watch?v=dg8eloQbKLM |
| AWS 的 Agent Orchestra 实战把多个容器化领域 Agent 通过 AgentCore Orchestrator 动态路由，并结合 Memory、Observability 与 MCP Lambda 汇总多模态结果，适合展示“运行时、记忆、监控和工具协议”如何一起支撑生产协作。 | https://www.youtube.com/watch?v=R0eY5lpyWOo |
| AWS re:Invent 2025 用 SecOps 真实流程演示专业 Agent 共享漏洞上下文、定位受影响服务器并生成修复补丁/产物，适合把多 Agent 从通用 Demo 落到“调查→修复→交付”的高风险业务闭环。 | https://www.youtube.com/watch?v=gWY9z75qCcU |
| 该视频把生产多 Agent 常见架构归纳为 Orchestrator-Worker、Hierarchical、Peer-to-Peer、Pipeline 四类，并结合真实用例与常见错误讨论选型，适合在分享开场建立协作模式地图，再衔接具体工程案例。 | https://www.youtube.com/watch?v=-zBbij9rrEI |
| Hermes 的看板模式把“多个 Agent 同时存在”推进到真正的协作调度，演示用共享看板组织任务与状态，适合分享多 Agent 团队为什么需要显式任务控制面，而不能只靠对话和人工派活。 | https://www.bilibili.com/video/BV1Vm556BERU |
| Claude Code 实战直接对比多个 Subagents 与多个独立 Agent 两种协作方式，适合用来讲轻量委派、独立上下文和并行团队之间的边界，以及 Coding 场景该怎样选多 Agent 模式。 | https://www.bilibili.com/video/BV1WtoTBiEuR |
| DeepSeek Harness 同时实测 Workflow 与 Agent Teams：动态派生多个 SubAgent 并行审计代码，再组建专业评审团队，并展示任务状态、Token 消耗和结构化结果汇总，适合对比“动态工作流”和“团队式编排”两类落地方式。 | https://www.bilibili.com/video/BV1THhK63Ebd |
| OpenCLAW 实操把多 Agent 协作拆成流水线、并行依赖和 AI 辩论三种模式，并用前端项目完整演示分工编码、审核和测试，还给出不同内存配置下的团队规模建议，适合讲协作拓扑与资源约束如何一起设计。 | https://www.bilibili.com/video/BV1tAdsBXE6Z |
| 三个智能体协作编码案例用少量 Markdown 文件和提示词组织 Agent 团队自动完成复杂网站，并给出相对单智能体的明显效率收益，适合作为低门槛“角色分工 + 共享约定”式 Coding Agent 团队案例。 | https://www.bilibili.com/video/BV141gPzeEB7 |
| 用 7 分钟梳理工业级多 Agent 编排的四种控制模式，适合在分享中快速建立“谁负责决策、谁负责编排”的模式地图，并用于对比不同协作拓扑的适用边界。 | https://www.bilibili.com/video/BV19D9eB9Etg |
| 通过开源插件让飞书群里的多个 Agent 机器人互相 @、协作并完成完整产品交付，适合展示多 Agent 如何直接嵌入现有团队沟通工具并形成真实交付闭环。 | https://www.bilibili.com/video/BV11QdmBEESq |
| Codex 多 Agent 自主协同实操以极简入口触发团队式执行，适合用来观察任务拆分、角色协同与自动推进如何在 Coding Agent 场景中被封装成低门槛工作流。 | https://www.bilibili.com/video/BV113EY6UEyy |
| Multica 最佳实践聚焦“超级个体”场景下的多 Agent 协作，适合补充人与多个专业 Agent 共事、任务分工和团队化工作方式的实际落地视角。 | https://www.bilibili.com/video/BV18i8y6HEDc |
| Codex 多智能体编排协同工作示例直接面向 Coding Agent 的并行协作，适合补充多 Agent 编排在日常软件开发中的轻量化实操案例。 | https://www.bilibili.com/video/BV1MUuR6hEdK |
| AWS 用 Lambda Durable Functions 把 4 个专业 Agent、人工审批和外部异步流程编排成可恢复的医疗授权链路，重点展示 checkpoint/replay、幂等、回调、轮询与失败续跑，特别适合讲“多 Agent 长流程怎样做到可靠且不重复付费”。 | https://aws.amazon.com/blogs/compute/building-fault-tolerant-multi-agent-ai-workflows-with-aws-lambda-durable-functions/ |
| Google Cloud 用 ADK 提出运行时动态加载 Schema 与确定性验证的多 Agent 交接模式，解决上下文膨胀、Attention Diffusion 和 malformed state 导致的 silent failure，适合分享“handoff 前如何把上下文契约变成可机器校验的边界”。 | https://cloud.google.com/blog/topics/developers-practitioners/beyond-static-prompts-with-google-adk |
| Google DeepMind/Cloud 把多 Agent 委派总结为 contract-first decomposition、成本感知路由、最小数据授权和必要的人类认知摩擦，适合用作“主 Agent 怎样安全、可验证地把任务交出去”的设计原则。 | https://cloud.google.com/blog/products/ai-machine-learning/how-agents-can-delegate-better |
| AWS 金融服务参考架构把 orchestrator 与投资组合、风险、市场数据等专业 Agent 跑在 EKS/AgentCore 上，并把每次 Agent hop 的认证、追踪、模型网关、成本控制、沙箱执行和 GitOps 一起落地，适合展示企业生产环境里“编排之外还缺什么”。 | https://aws.amazon.com/blogs/industries/multi-agent-systems-for-financial-services-on-amazon-eks-and-agentcore/ |
| AWS Professional Services 用 Intake、IaC、治理、SRE 四个 Agent 覆盖 300+ 应用迁移生命周期，并通过共享记忆、MCP、身份权限和 HITL 把 IaC 开发从数周压到分钟级，适合作为有量化结果的企业多 Agent 流程改造案例。 | https://aws.amazon.com/blogs/machine-learning/scaling-cloud-migrations-with-agentic-ai-on-amazon-bedrock-agentcore/ |
| AWS Security Agent 的自动渗透测试把基线扫描、动态探索、专业 swarm worker、独立验证与漏洞评分串成多阶段协作系统，还能根据应用反馈生成新攻击任务，适合分享“并行 Agent + 适应性规划 + 独立复核”的高风险生产实践。 | https://aws.amazon.com/blogs/security/inside-aws-security-agent-a-multi-agent-architecture-for-automated-penetration-testing/ |
| AWS 的市场监控系统用 LangGraph 管共享状态、确定性流程、checkpoint/HITL，用 Strands 在节点内做专业推理，再交给 AgentCore 负责生产运行，适合讲“确定性 orchestrator + 局部 Agent 自主性”这一常见落地分工。 | https://aws.amazon.com/blogs/machine-learning/market-surveillance-agent-with-langgraph-and-strands-on-agentcore/ |
| AWS M&A 尽调参考实现由 supervisor 协调筛选、财务、战略匹配、合规四类 Agent，并把 RAG、长期交易记忆、政策控制、自动引用校验和可部署样例整合起来，适合作为“多 Agent 输出如何做到可审计”的完整业务案例。 | https://aws.amazon.com/blogs/machine-learning/accelerating-ma-due-diligence-with-amazon-bedrock-agentcore/ |
| AWS 医疗多模态实践把临床、影像、基因、论文、试验和预测等专业 Agent 以 A2A 暴露，由 supervisor 按需调用，并结合 MCP、JWT、独立 Runtime、Memory 与 OpenTelemetry 追踪，适合展示跨数据域多 Agent 如何标准化通信和独立扩缩容。 | https://aws.amazon.com/blogs/industries/multi-agent-multimodal-data-analysis-on-aws-part-2-multi-agent-orchestration-and-predictive-analytics/ |
| 文章把 Agent 通信拆成工具调用、任务委派、消息传递三层，并分别讨论 MCP、A2A 与自定义消息协议的边界，适合分享中避免把“Agent 会通信”混成一个概念，并建立协议选型层次。 | https://niteagent.com/blog/agent-to-agent-communication-protocols-2026/ |
| 这个 Claude Code Agent Teams Skill 紧跟当前 API，沉淀并行专家、竞争假设、流水线、自组织池四种可复用协作模式，同时记录共享工作树冲突与常见误用，适合直接做 Coding Agent 团队的可复现实操素材。 | https://github.com/mttzzz/claude-code-agent-teams |
| agent-team-go 用 Go 实现结构化委派、技能自动安装、飞书/Telegram 通道、可回放运行、产物与事件日志等团队基础设施，适合补充“多 Agent 框架怎样做成易部署、可审计、能接真实协作渠道的服务”。 | https://github.com/daewoochen/agent-team-go |
| SocioFi 基于 8–22 个 Agent 的生产系统总结顺序流水线、并行协作、层级编排与人工门禁等模式，并复盘 13-Agent GTM 与 22-Agent 制造平台在接口隔离、实时数据新鲜度和可观测性上的实际问题，适合作为“Agent 数量上升后协作复杂度如何失控”的一线案例。 | https://sociofitechnology.com/labs/blog/multi-agent-orchestration-patterns |
| Adobe 的企业 B2B 营销多 Agent 系统把透明路由、会话/用户/组织三级记忆、plan-and-execute 与 discover-and-create 工作流、MCP 工具接入和组合式评估放在同一生产案例中，还讨论 Agent 单体表现不错但组合后失败的 compositionality gap，特别适合分享协作系统怎样做评估。 | https://llmday.com/2026-san-francisco-q2/Sanghamitra_Deb_Adobe_Building_Production_MultiAgent_Systems_Memory__Orchestration__Evaluation_at_Sc |
| Conf42 的生产案例从单体客服聊天机器人演进到云原生多 Agent 订单支持平台，重点处理跨服务编排深度、延迟叠加、局部故障传播、多跳推理追踪与治理，适合讲“把多 Agent 当分布式系统而不是 Prompt 技巧”后的工程设计。 | https://www.conf42.com/Cloud_Native_2026_Sandeep_Mannapur_ai_agents_workflows |
| AWS 的 Agent Orchestration 实践把网络分区、API 限流、长任务、人工审批、定时任务和事件触发都纳入编排层，强调 durable、observable、lifecycle-managed 的运行保障，适合补充多 Agent 上生产后为什么需要传统分布式工作流的可靠性机制。 | https://aws.amazon.com/marketplace/build-learn/ai-agent-learning-series/agent-orchestration |
| Codeforges 的生产案例用 root agent 动态派生专业 Agent，并围绕真实成本、并行文件冲突、崩溃后的上下文恢复、高风险人工审批和全链路审计设计监督层，适合作为“自治 Agent 团队如何加预算、权限与审计护栏”的落地样本。 | https://codeforges.com/case-studies/autonomous-agent-orchestration |
| 制造业案例由生产监控、维护、质量、库存、能源和交班 6 个专业 Agent 通过 Postgres 事件总线协作，在 40 台设备上 24/7 运行并坚持所有物理动作由人批准，同时披露停机改善、建设周期和成本，适合展示实体业务里多 Agent 与 HITL、事件驱动架构怎样结合。 | https://aliansoftware.com/en/blog/we-built-ai-agent-manufacturing-cost |
| GitHub Agentic Workflows 官方把多个仓库 Agent 的协作提升到项目协调层，通过任务分解、依赖跟踪、进度监控和延迟告警来推进大型改造，适合分享“多 Agent 研发协作如何直接落在 Issue/PR/仓库工作流上”。 | https://github.github.io/gh-aw/blog/2026-01-13-meet-the-workflows-campaigns/ |
| Bottega 面向工程团队实现 team-first、remote-first 的 Coding Agent 编排，用“人开始/结束、AI 执行中间过程”的模式串起规划、实现、QA 循环和 PR 评论处理，并可混用 Claude Code、Codex、OpenCode，适合展示多模型 Agent 团队如何嵌入真实研发流程。 | https://github.com/vdaubry/bottega |
| team-tasks 用共享 JSON 任务文件实现 Linear、DAG 和 Debate 三种协作模式，分别对应串行交接、依赖并行和多 Agent 交叉评审，代码很轻量，适合在分享中作为“最小可理解协作控制面”的可复现实例。 | https://github.com/win4r/team-tasks |
| 工业 UI 自动测试研究对 LangGraph 多 Agent 系统进行了 300 份连续报告、636 次测试执行的量化分析：修复收敛率约 70%，但首轮成功仅 10%，还有大量无可执行产物和“削弱断言”式伪收敛，适合用数据说明为什么生产多 Agent 必须限制自治并设置验证边界。 | https://arxiv.org/abs/2605.01471 |
| 从 Tool Runtime 到单/多 Agent 选型，再到 LangGraph 状态图与 AutoGen、CrewAI 的适用边界，强调只有任务天然多阶段、异构工具或目标冲突时才值得多 Agent；适合作为分享中“何时需要多 Agent + 如何把流程状态显式化”的工程判断素材。 | https://zhuanlan.zhihu.com/p/2074892026299139660 |
| 基于 Codex 实际使用拆出跨项目新线程、会话 ID 接力、一对多并行等场景，并强调项目路径、任务状态、执行方式和验收要求；适合做 AI Coding 多 Agent 调度的轻量可复现实操。 | https://zhuanlan.zhihu.com/p/2070504892796498056 |
| 以多个 AI Agent 在同群协作为问题，复盘并发冲突、防碰撞、新鲜度门控等协调机制，并把哪些问题交给代码、哪些交给提示词讲清；适合作为多 Agent 共享上下文和冲突治理的工程案例。 | https://zhuanlan.zhihu.com/p/2074191821815668796 |
| 系统覆盖顺序、并行、层级、辩论四种协作架构，AutoGen、CrewAI、LangGraph 选型、内容团队代码实践以及幻觉、成本、协调、安全避坑，适合做分享里的全景架构地图与最小实现示例。 | https://zhuanlan.zhihu.com/p/2074211818957029552 |
| 从真实并行 Coding Agent 工作流出发，明确哪些任务适合或不适合并行，并给出 Plan→Prompt→Verify→Review 闭环、统一命令与验证报告；适合讲多 Agent 提速后的真正瓶颈如何转向任务拆分和 Review。 | https://zhuanlan.zhihu.com/p/2040814536345792826 |
| 把可工程化多 Agent 定义为协调、专业执行、独立验证和统一运行时，强调结构化交接、状态恢复、结果验证与成本收益；适合用作分享开场的设计原则基线，避免把多 Agent 误解为自由聊天。 | https://zhuanlan.zhihu.com/p/2069012163067491985 |
| 从“个人调用 Agent”进一步推进到人和 Agent 共享目标小队，提出 PM 调度者、任务图、依赖、进度和责任人共同状态，适合分享组织级多 Agent 协作如何从聊天走向可治理工作系统。 | https://juejin.cn/post/7676421042796544040 |
| 基于源码对比 Claude Agent Teams 与 Codex multi-agent v2 的团队创建、任务分配、通信方式和工程取舍，适合做 AI Coding 多 Agent 机制与架构选型的对照材料。 | https://juejin.cn/post/7653415873380876330 |
| claude-studio 将 Claude Code Agent Teams 的工作流从散落配置变成可视化 DAG 编排，适合展示多 Agent 协作如何产品化为可观察、可复用的编排控制面。 | https://juejin.cn/post/7629036295418806306 |
| 从企业 AI 需求评估出发比较工作流、单 Agent 与多 Agent 协作的适用边界，适合作为分享里“什么时候值得上多 Agent”的前置决策框架。 | https://juejin.cn/post/7620718787835969536 |
| 用具体业务场景完整展开 AgentScope 多智能体协作实践，适合拆解角色分工、消息传递和任务协同，让分享不只停留在框架概念层。 | https://juejin.cn/post/7591730714140540970 |
| Dify 与 Nacos A2A 插件把 Agent Registry、注册发现和双向 Agent 协作接起来，适合补充多 Agent 从进程内编排走向跨服务互操作与治理的落地方式。 | https://juejin.cn/post/7603219519666257939 |
| 以 4 个 AI Agent 重塑软件测试流程，并给出团队规模、用例设计和回归测试等真实业务背景，适合用垂直场景说明专业 Agent 分工如何进入研发流程。 | https://juejin.cn/post/7533512134002409522 |
| 记录 16 个 AI Agent 在两周内协作完成约 10 万行 C 编译器并可编译 Linux 内核的案例，适合讨论大规模并行 Coding Agent 的任务拆分、协同收益与验证边界。 | https://juejin.cn/post/7603376587653169202 |
| 从零搭建 CrewAI 多智能体协作系统，覆盖角色、任务和团队编排等核心实践，适合作为分享中可快速复现的框架级入门案例。 | https://juejin.cn/post/7649256802330443827 |
| Claude Code Agent Teams 实机演示多个独立 Agent 同时编码与协作，适合直观看角色分工、并行执行和人类监督如何进入真实 AI Coding 工作流。 | https://www.youtube.com/watch?v=-1K_ZWDKpU0 |
| Google for Developers 官方把 ADK、MCP 与 A2A 放到同一 Google Cloud 实操中，适合讲清“Agent 构建、工具连接、Agent 间通信”三层能力怎样组合成可落地系统。 | https://www.youtube.com/watch?v=6mQwHqK1I5w |
| Strands Agents 实战用 A2A + MCP 打通多 Agent 与工具生态，主题直接聚焦消除 Agent 信息孤岛，适合作为跨框架、跨服务互操作的工程案例。 | https://www.youtube.com/watch?v=TjTgHA5DjDM |
| AWS re:Invent 的 Cosine AI 案例把 LLM 微调与 Multi-Agent Orchestration 放进真实生产讨论，适合补充“协作效果不仅取决于编排，也取决于模型如何为角色和调度行为做针对性优化”的视角。 | https://www.youtube.com/watch?v=GYaDjPwLDGo |
| 该分享从常见多 Agent 编排模式一路延伸到生产落地，适合用于串联架构选型、协作边界与生产化注意点，作为分享中的模式地图与实践过渡材料。 | https://www.youtube.com/watch?v=EtSO9vU84ws |
| Fanatics Betting and Gaming 把 Supervisor、Responsible Gaming 分类 Agent、RAG、MCP 账户/交易工具和人工升级跑进真实体育博彩客服生产环境，并披露上线两个月 containment 约提升 56%、resolution 约提升 53%，适合分享“多 Agent + 合规/HITL + 持续评估”的完整落地闭环。 | https://aws.amazon.com/blogs/machine-learning/how-fanatics-betting-and-gaming-built-a-multi-agent-customer-support-system/ |
| Cognizant 把碎片化企业内网改造成服务 35 万员工的生产级多 Agent 系统，并给出支持工单减少约 50% 的结果，适合用来讲大型企业内部场景如何从“多个孤立 Agent”走向统一协作入口与规模化运营。 | https://www.cognizant.com/us/en/ai-lab/blog/cognizant-ai-agents-enterprise-intranet-transformation-neuro-san |
| Grid Dynamics 的 Fortune 500 支付企业案例把共享 RAG、Guardrails、可观测、Agent-to-Agent 协调与 Temporal durable workflow 做成统一平台，并披露分析周期从 4–6 周降到数小时、年节省约 900–1400 万美元，适合展示“多 Agent 平台化 + 可审计业务收益”的企业案例。 | https://www.griddynamics.com/blog/multi-agent-enterprise-workflows-case-study |
| Google Cloud 的 Dev Signal 系列把 ADK + MCP 多 Agent 从本地验证一路部署到 Cloud Run，并补齐 Terraform、持久状态、OTel/Agent Trace、监控与定向评测，适合分享“多 Agent Demo 怎样补齐生产基础设施”的可复现迁移路径。 | https://cloud.google.com/blog/topics/developers-practitioners/create-expert-content-deploying-a-multi-agent-system-with-terraform-and-cloud-run |
| Composio 的 Agent Orchestrator 已把“同时跑多个 Coding Agent”做成完整控制面：每个 Agent 独立 worktree/branch/PR，并自动接收 CI 失败和 Review 意见继续修复，还能通过插件混用 Claude Code、Codex、OpenCode 等，适合拆解并行研发 Agent 如何真正嵌入现有 GitHub 流程。 | https://github.com/ComposioHQ/agent-orchestrator |
| Open Orchestrator 把 Claude Code、Pi、Droid、OpenCode 等异构 Coding Agent 放到多 worktree 控制台中，并用 Conflict Guard 提前侦测并行编辑冲突，适合分享“隔离并行 + 人类决策面 + 冲突治理”的轻量工程实践。 | https://github.com/gitpcl/openorchestrator |
| Resonate 的可运行示例把 researcher→writer→reviewer 的每次 handoff 变成 durable checkpoint，故意让中间 Agent 崩溃也只重试失败步骤、不会重跑前序 Agent，并预留 HITL hook，适合用最小代码讲清“多 Agent 长流程为什么需要可恢复编排”。 | https://github.com/resonatehq-examples/example-multi-agent-orchestration-py |
| InfoQ 以 5G Core 安全运营为生产场景，把 Planner、Executor、Reviewer 等 Agent 通过 A2A 协作，并用 MCP 接入环境工具，同时强调 reviewer-first、policy-as-code、人工升级与审计，适合分享高风险多 Agent 如何把协议互操作和安全门禁同时落地。 | https://www.infoq.com/articles/multi-agent-security-operations/ |
| Google Cloud Next ’26 开源的 Race Condition 是可部署的多 Agent 参考架构，使用 ADK、Gemini 与 A2A 让规划、环境模拟和执行 Agent 协同，并提供确定性 runner 与录制回放，适合展示大规模协作系统如何兼顾真实运行、测试和演示可靠性。 | https://github.com/GoogleCloudPlatform/race-condition |
| A2A 官方样例仓库提供多语言 Agent、Host 与协作 Demo，覆盖 Agent 发现、任务委派、流式/异步交互和跨框架互操作，适合作为分享中可直接运行的“协议化多 Agent 协作”基线项目。 | https://github.com/a2aproject/a2a-samples |
| Google ADK 官方 A2A quickstart 展示如何把本地 Agent 暴露为远程 A2A 服务并由另一个 Agent 消费，包含 Agent Card、Task Store、请求处理与部署入口，适合用最小样例讲清跨进程 Agent 协作的工程组成。 | https://github.com/google/adk-docs/blob/main/docs/a2a/quickstart-exposing.md |
| TunerLabs 基于生产经验整理 Planner/Executor/Critic、Fan-out/Fan-in、Scoped Pipeline、Specialist+Generalist 四种多 Agent 模式，并强调从最简单拓扑起步、控制爆炸半径和避免过度拆 Agent，适合做协作架构选型与反模式对照。 | https://www.tunerlabs.com/blog/multi-agent-orchestration-patterns |
| Coverge 将多 Agent 编排视为分布式系统问题，系统讨论共享状态、partial failure、handoff、超时、可观测性与 human review，适合补充“从 Demo 到生产”阶段最常见的运行风险和工程约束。 | https://coverge.ai/blog/multi-agent-orchestration |
| Microsoft 新版 MultiAgent-Accelerator 直接把 Microsoft Agent Framework、A2A、MCP、Azure Service Bus 与 AKS/Container Apps 串成可部署参考架构，还演示跨 Azure/GCP 的 Agent 发现与调用，适合讲“协议互操作 + 云原生部署 + 身份治理”如何一起落地。 | https://github.com/microsoft/MultiAgent-Accelerator |
| Azure-Samples 学生贷款示例用 Orchestration、Document Scanner、Validator、Decision Maker、Chat 等 Agent 组成完整业务链，并把 FastAPI、React、MCP、结构化文档抽取、SSE 与一键云部署放在同一项目，适合展示多 Agent 如何从协作逻辑走到可用业务应用。 | https://github.com/Azure-Samples/multi-agent-student-loan-processing-SA |
| Microsoft 医疗授权参考实现用 Compliance、Clinical Reviewer、Coverage、Synthesis 四个 Agent 配合 gate-based 决策、置信度、审计文档和 HITL 覆盖/推翻，特别适合分享强监管场景怎样把多 Agent 的可解释、可审计和人工责任边界工程化。 | https://github.com/microsoft/Prior-Authorization-Multi-Agent-Solution-Accelerator |
| A2A v1.0 是正式面向生产的 Agent-to-Agent 标准，新增/强化多协议绑定、版本协商、多租户、签名 Agent Card 与企业安全要求，适合说明多 Agent 从单框架内部协作走向跨技术栈、跨组织互操作需要哪些协议层能力。 | https://github.com/a2aproject/A2A/blob/main/docs/announcing-1.0.md |
| 该跨框架项目让 LangGraph、Pydantic AI、OpenAI Agents SDK 三个 Agent 通过 A2A 协作、用 MCP 调工具，并补齐离线确定性 CI、OpenTelemetry/OpenInference/Phoenix 分布式追踪，适合做“互操作 + 可测试 + 可观测”一体化的可运行架构案例。 | https://github.com/OmPrajapati7901/a2a-mcp-multi-agent/blob/main/ARCHITECTURE.md |
| 这个 production-style 多 Agent 栈把 A2A 专家发现、MCP 能力、LangGraph、Qdrant、LiteLLM、Langfuse、Postgres checkpoint 与 ADR 设计决策一起开源，尤其适合分享“架构取舍为何如此做”而不只看代码结果。 | https://github.com/babebp/llm-agent-production-architecture |
| 这份 Multi-Agent Production 化章节把协作模式与 Harness Engineering、eval、observability、cost/latency、部署放在同一工程分层中，并明确何时不该上多 Agent，适合用作分享的结构化主线与生产化检查表。 | https://github.com/WenyuChiou/awesome-agentic-ai-zh/blob/main/stages/07-multi-agent-production.zh-Hans.md |
| 这篇生产编排实践从“更多自治并非目标”出发，聚焦任务拆分、handoff、验证、控制面、故障与吞吐等真实工程问题，适合在分享中补充框架无关的多 Agent 生产纪律与反模式。 | https://gist.github.com/prasad-kumkar/f9a71a839afe8ed03da5909bae9e84f3 |
| Claude Code Agent Teams 实战把 Team Lead、独立 Teammates、带依赖共享任务表和直接消息通信落到并行代码审查/前后端开发，并总结上下文、任务粒度、文件冲突、Delegate 与 Token 成本，适合分享 Coding Agent 团队的真实操作边界。 | https://juejin.cn/post/7610141315292725288 |
| OpenClaw 多 Agent 配置实战从独立 Workspace/SOUL/模型/Tool Policy 到 Telegram 多账号绑定、路由优先级、权限与 Gateway 排障，完整记录配置踩坑，适合补充多 Agent 协作落地前的隔离、路由与运行治理。 | https://juejin.cn/post/7605810996125548578 |
| 以独立开发者同时推进前端、后端和测试为场景，基于 ChatDev 2.0 的多智能体编排、预制模板和模块化扩展展示一人多角色协同交付，适合作为低门槛多 Agent 平台化实践案例。 | https://juejin.cn/post/7596687697755111434 |
| AutoGen 团队协作实战用 RoundRobinGroupChat 跑主 Agent/批评 Agent 迭代，覆盖 run/stream、终止条件、外部停止/恢复与 Human-in-the-Loop，并给出代码和执行结果，适合展示可控团队运行机制如何实现。 | https://juejin.cn/post/7469440141447643148 |
| “Introducing multi-agent orchestration”直接围绕多 Agent 编排展开，适合作为分享中建立 Orchestrator 如何组织多个 Agent、统一任务流与协作边界的入门案例。 | https://www.youtube.com/watch?v=RWT3sh68PWE |
| AI Engineer 的“From Chaos to Choreography”聚焦真正可用的多 Agent 编排模式，适合提炼从无序自治转向明确协作拓扑、任务交接和控制策略的实践经验。 | https://www.youtube.com/watch?v=2czYyrTzILg |
| AI Engineer 的 AgentCraft 演讲聚焦 Orchestration 本身，适合作为框架之外的工程案例，讨论如何把多个 Agent 的角色、执行顺序与协同机制组织成可落地系统。 | https://www.youtube.com/watch?v=kR64LOqBBCU |
| 从零实现 Agent Team 的实操同时给出教学仓库和实战项目，适合展示团队创建、任务分工与协同执行如何落到可运行代码，而不是只停留在架构概念。 | https://www.bilibili.com/video/BV1Jx5h6oEzV |
| 用短时实战快速梳理 Multi-Agent 架构，适合在分享中作为协作组件、角色关系与整体执行链路的入门总览，再衔接更深的生产案例。 | https://www.bilibili.com/video/BV11p6wB2EmX |
| 强调多 Agent 应按“上下文边界”而不是拟人角色拆分，适合解释如何减少上下文污染、职责重叠与无效协作，是很实用的架构设计判断原则。 | https://www.bilibili.com/video/BV1QC6uBLEXC |
| 企业级 Harness 项目把 Planning、异步子 Agent、Docker Sandbox、HITL、Skills 自进化和 Context Engineering 放进同一套商业级实现，特别适合拆解多 Agent 从委派到安全、恢复与人工治理的完整落地链路。 | https://www.bilibili.com/video/BV1Cs7h6MEsX |
| OpenClaw 多 Agents 配置从单 Agent 的上下文拥堵、串行执行与权限风险出发，进一步实操权限隔离、模型差异化与团队协作，适合分享“为什么拆 Agent”以及配置层如何真正落地。 | https://www.bilibili.com/video/BV1PTPQz5Er4 |
| 将 Multi-Agent、Harness、Tools、MCP、Deep Agent 与 Skills 放进同一个项目实战，适合展示多 Agent 协作如何和工具协议、技能体系及运行框架组合成端到端应用。 | https://www.bilibili.com/video/BV1GruR6KEVx |
| Google 官方用 Python 提取 Agent 与 Go 确定性校验 Agent 通过 A2A 组成合同合规流水线，直接解决真实团队中不同语言、不同部署目标的 Agent 无需重写即可协作的问题，适合分享“跨语言、跨服务多 Agent 如何协议化落地”。 | https://developers.googleblog.com/build-cross-language-multi-agent-team-with-google-agent-development-kit-and-a2a/ |
| Fallbrook 基于已上线企业多 Agent 工作流归纳 2026 年真正扩展成功的协作模式、失效方式与运维纪律，适合用来补充“哪些架构从 Demo 活到了生产、团队为此付出了哪些工程代价”的实践视角。 | https://www.fallbrookresearch.com/2026/05/08/multi-agent-workflows-in-production.html |
| Microsoft 官方展示如何把 Agent Framework、GitHub Copilot CLI/SDK 与 Squad 组合成可运行的 Agent 团队：协调器+专业成员、持久决策/技能、标准 AIAgent 接口和流式会话都能直接接入现有应用，适合讲“多 Agent 团队如何嵌入生产 SDK 与研发流水线”。 | https://devblogs.microsoft.com/agent-framework/building-agent-teams-with-agent-framework-github-copilot-cli-and-squad/ |
| Microsoft 的 AlpineAI 示例把 advisor 与天气/客流/安全/教练/研究等专业 Agent 做成跨 .NET、Python、Go 的分布式服务，并用 A2A、Aspire、Container Apps、健康检查和 OpenTelemetry 统一部署与追踪，特别适合分享“多 Agent 上线后其实就是分布式系统”的工程实践。 | https://devblogs.microsoft.com/aspire/building-distributed-multi-agent-systems-with-aspire-and-microsoft-agent-framework/ |
| multiagents 用 MCP 把 Claude Code、Codex CLI、Gemini CLI 组成同一协作团队，并把文件锁/ownership zone、任务状态机、互评闭环、共享知识、会话恢复和崩溃重启都做进运行时，适合展示异构 Coding Agent 如何真正解决并行冲突与状态一致性。 | https://github.com/zetbrush/multiagents |
| CCteam-creator 将 2–6 个 Claude Code Agent 的协作变成文件化 Harness：任务计划/发现/进度可持久恢复，并内置 CI、独立 Review、文档新鲜度、规则门禁和阶段健康检查，适合分享“共享事实源 + 机器验收”如何让 Agent 团队稳定长期推进。 | https://github.com/jessepwj/CCteam-creator |
| claude-team-orchestration 把共享任务、Agent 间消息和错误恢复封装成插件，并提供并行专家、pipeline、swarm、research+implement、plan approval、refactoring、RLM 等 7 类可复用模式，适合用同一套原语讲清不同协作拓扑如何选与如何落地。 | https://github.com/zircote-plugins/claude-team-orchestration |
| AgentsID 横向分析 Claude Agent Teams、AutoGen、CrewAI、LangGraph、OpenAI Agents SDK，归纳 Agent 身份、凭证继承、Agent 间通信、委派时工具权限四类结构性鉴权缺口，并结合真实 CVE 说明风险，适合补齐多 Agent 落地经常被忽略的安全边界。 | https://github.com/AgentsID-dev/agentsid-scanner/blob/master/docs/agent-teams-auth-gap-2026.md |
| ChipAgents 将多 Agent 编排落到 ASIC/3D IC 工程：多个 Agent 在隔离环境并行探索假设，工程师以批量 Review 把关，再进入 signoff；文中还给出 WhaleChip 根因分析从数天缩到 15 分钟的真实部署结果，适合讲“并行探索 + HITL”在高成本工程场景的价值。 | https://chipagents.ai/blogs/multi-agent-orchestration-ic-design-autonomy |
| 2026 年制造业研究把机器人与 AGV 都建模为 LLM Embodied Agent，对比 Robot-Lead 与 Vehicle-Lead 两种去中心化协作策略；复杂拆解任务中 VLC 达到 100% 约束满足、同时减少 41% Token 和 68% API 调用，适合用量化数据说明“谁来主导协作”会直接影响质量与成本。 | https://www.sciencedirect.com/science/article/abs/pii/S0278612526001147 |
| agent-team 基于 ACP 把 Claude、Codex、Gemini、OpenCode 等 20+ Coding Agent 收敛到同一 CLI 控制面，并用独立进程会话统一处理派活、日志与权限审批，适合展示“异构 Agent 如何先标准化运行与管理接口，再谈团队协作”的轻量落地方式。 | https://github.com/nekocode/agent-team |
| codex-teams 用 Lead + Workers、独立上下文、群聊/私信/共享产物组织 Codex 团队，并把测试命令验证、失败后重新派修和中途 steer 做进执行闭环，适合拆解 AI Coding 多 Agent 从并行执行到机器验收的完整协作协议。 | https://github.com/skrabe/codex-teams |
| aqm 用 YAML 定义显式队列、handoff、并行 fan-out、会议式共识与 Human Gate，还支持跨模型 Review、自动拒绝重试和上下文裁剪，并以“用 aqm 开发 aqm”给出真实流水线与耗时数据，适合分享声明式多 Agent 工作流怎样做到可复用、可验证和控成本。 | https://github.com/aqm-framework/aqm |
| opencode-workspace 把 plan/build 编排器、researcher/coder/scribe/reviewer 专业 Agent、MCP、后台委派、通知与 Git worktree 打成一套 Harness，并对各角色设置细粒度读写/执行权限，适合说明生产协作除了分工还需要隔离、工具边界与最小权限。 | https://github.com/kdcokenny/opencode-workspace |
| Claude Swarm 将复杂任务先拆成依赖图再分波并行执行，并内置文件锁冲突检测、预算硬限制、失败重试、统一质量 Gate、实时成本/进度看板和 Session Replay，适合用作“并行 Coding Agent 如何同时治理冲突、质量、成本与可观测性”的工程案例。 | https://github.com/affaan-m/claude-swarm |
| π Multi-Agent 把 Goal→Plan→Execute→Evaluate→Replan→Output 做成完整生命周期，并提供顺序、并行、辩论、专家组、Critic-Reviewer、层级等协作模式，结合多模型路由、共享记忆、质量阈值重规划和沙箱/预算控制，适合用作多 Agent 平台架构与协作拓扑的系统化参考。 | https://github.com/jwangkun/Pi-Multi-Agent |
| openJiuwen 的 Agent Team Engine 用 Leader 动态组队、Teammate 自主领取任务和共享工作区组织科研团队协作，前序产物可直接成为后续输入，适合展示“团队分工 + 共享状态”怎样落成可执行协作机制。 | https://zhuanlan.zhihu.com/p/2033665604016714386 |
| AgentTeams 实践重点复盘权限拆分、用持久化对象替代大段摘要传递、Human-in-the-loop 与 Eval Release Gate，适合分享多 Agent 从 Demo 进入生产后如何控制信息损耗、责任边界与结果质量。 | https://zhuanlan.zhihu.com/p/2071003231002670310 |
| AgentTeams 的组织建模实践把 Session/Team 扩容、同进程 Subagent、沙箱隔离、共享存储、深休眠成本控制和持续优化放到统一架构里，适合讲企业多 Agent 平台怎样同时处理协作、性能、安全和成本。 | https://zhuanlan.zhihu.com/p/2060356797349860370 |
| 分布式 Agent Swarm 用 Leader 与不同机器上的 Teammate 配合注册发现和共享工作区完成跨节点代码接力，适合展示多 Agent 从单机编排走向跨机器协作时的发现、上下文共享与任务连续性。 | https://zhuanlan.zhihu.com/p/2041105589108003513 |
| OpenClaw 多 Agent 小红书实战把热榜与竞品数据收集、内容分析、笔记创作、图文生成和自动发布串成端到端流水线，适合展示多 Agent 如何从角色分工进一步落到可直接运行的业务自动化闭环。 | https://juejin.cn/post/7614123089519018003 |
| 以 4 个 AI Agent 重构软件测试流程，将测试任务拆给不同角色协同处理并围绕真实团队痛点展开，适合补充“多 Agent 不只用于编码，也能进入测试交付链路”的垂直实践案例。 | https://juejin.cn/post/7533512134002409522 |
| 用 LangGraph.js 搭建 5 个专业 Agent 的协作系统，由 Coordinator 动态决定执行链，再通过 SSE 将每个 Agent 的运行轨迹实时推到前端，适合展示“多 Agent 编排 + 可交互产品界面”的完整落地形态。 | https://juejin.cn/post/7641168149850947584 |
| 作者复盘多 Agent 团队的失败尝试，重点指出 context、memory、状态恢复和 Agent 间交接协议才是协作的真正难点，并给出何时应退回单 Agent 的判断边界，适合用作分享中的反例与选型原则。 | https://juejin.cn/post/7628784821854863369 |
| Codex 多智能体并发实战直接用 spawn/wait/close 等协作机制并行审查 OpenClaw 代码，针对并发、超时、重试和状态同步寻找问题，适合展示 Coding Agent 团队如何用于真实代码库的并行诊断。 | https://juejin.cn/post/7611737838309244978 |
| TiDB 联合创始人结合大规模 Token 消耗、Agent Infra 和实际 Agent Team 工作经验，复盘纯黑盒协作为什么会失控，以及 Spec、人类点拨、约束与软件工程纪律为何仍关键，特别适合作为生产实践与反思案例。 | https://juejin.cn/post/7630681198803566655 |
| 游戏小团队案例把 GDD 从给人看的文档升级为图像、视频、编程等多个 Agent 共用的“执行合同”和单一事实源，适合讲跨模态、跨角色 Agent 协作怎样通过共享规格减少对不齐和返工。 | https://juejin.cn/post/7602512226999533618 |
| 对比 Claude Code Agent Teams 与 Codex multi-agent v2 的协作机制，涵盖上下文隔离、任务委派、成员通信和并行执行方向差异，适合在分享中用同类工具对照说明不同多 Agent 架构为何会做出不同工程取舍。 | https://juejin.cn/post/7653415873380876330 |
| Arize Observe 2026 从“生产 AI Agent 团队正在构建什么”切入，适合补充当前团队级 Agent 的真实实践方向与生产化关注点，作为分享中的趋势型案例。 | https://www.youtube.com/watch?v=m0mS7lLLDaw |
| AWS re:Invent 2025 把 Multi-agent Orchestration 直接落到反洗钱场景并结合 Strands，适合展示高价值、强合规业务中专业 Agent 协作如何进入企业实践。 | https://www.youtube.com/watch?v=VtrfpAVFKdE |
| 这套 MCP + A2A Coding Masterclass 从零构建多 Agent 编排，把工具连接与 Agent 间通信放进同一实战，适合讲协议层如何支撑跨 Agent、跨工具协作。 | https://www.youtube.com/watch?v=utF6leQwcts |
| 直接比较 Claude Subagents 与 Agent Teams 的差异，适合在分享中说明何时使用轻量委派、何时需要真正的团队式协作，避免把所有多 Agent 机制混为一谈。 | https://www.youtube.com/watch?v=jT1rg3TBf-I |
| 以“如何更好地构建 Claude Agent Teams”为主线，适合作为 Coding Agent 团队从基础用法走向更成熟协作方式的实操补充。 | https://www.youtube.com/watch?v=vDVSGVpB2vc |
| Microsoft Developer 的 Armchair Architects 从架构视角讨论 Multi-agent Orchestration 与 Patterns，适合梳理编排模式、架构取舍以及企业落地前应考虑的协作边界。 | https://www.youtube.com/watch?v=Dwyx8GomVvQ |
| Claude Code Agent Teams 从核心概念、底层架构一路做到真实项目实测，由 Team Lead 拆任务、多个 Teammate 并行执行并互相通信，适合分享原生 Coding Agent 团队如何解决长上下文、串行瓶颈和单一视角问题。 | https://www.bilibili.com/video/BV1gwcAzkEhw |
| Multica 直接以多 Agent 编排工具做完整实战演示，适合补充“编排平台如何从概念落到实际操作”的视角，用来观察多 Agent 团队怎样被组织并进入可执行工作流。 | https://www.bilibili.com/video/BV1Vvj86UEW4 |
| Hermes + DeepSeek Harness 实测让多个 Agent 在同一任务中讨论、分工、推进，再与真人共同收敛结果，覆盖需求拆解、代码修改和方案讨论，适合展示“Agent 团队 + Human-in-the-loop”的轻量协作落地。 | https://www.bilibili.com/video/BV1FahK6eEef |
| oh-my-opencode 用编排器指挥 GPT、Claude、Gemini 分工协作，并结合 LSP 语义感知与 Ralph Loop 约束代码质量和任务收敛，适合分享跨模型 Coding Agent 怎样把编排、语义校验和完成度控制组合起来。 | https://www.bilibili.com/video/BV1ZBiRBrErU |
| 该项目基于两个真实生产 Agent 团队的会话数据做量化复盘，其中一个团队分析 104 次 subagent 运行，并给出角色化推理强度、任务分波、文件边界、选择性 Review、partial handoff 等规则，使长尾耗时和返工显著下降；特别适合用数据讲“多 Agent 协作不是多开几个实例，而是要可测量地优化编排纪律”。 | https://github.com/veerapan-boo/agent-team |
| Agent Team Engineering 把多 Agent 团队做成“可编译的团队资产”：统一定义 Context、Roles、Skills、Workflow、决策与工作记录，再投射到 OpenClaw、Hermes、Codex、Claude 等原生宿主，并以预览、摘要绑定确认、证据分级和人工权限边界控制安装与自动化，适合分享“跨 Agent Runtime 的团队可移植性与治理”如何落地。 | https://github.com/90le/agent-team-engineering |
| policy-research-kit 用 5 个 Agent 把政策报告拆成结构设计、资料调查、写作与独立验证，并强制“写作者不能自我验收”、两次质量 Gate 和全过程文件留痕；仓库还给出与普通 Claude Code 的可复现 A/B 样例，适合分享“独立 Reviewer + 证据链”怎样把多 Agent 从角色扮演变成质量控制机制。 | https://github.com/parkjui92/policy-research-kit |
| Innvo Labs 把串行多 Agent 链改造成事件驱动 DAG 状态机，以 Researcher、Risk、Financial、Synthesizer 等专业 Agent 并行推进，并披露端到端感知延迟下降约 82%；适合讲生产场景里“协作拓扑、流式事件与延迟预算”为什么要一起设计。 | https://www.innvolabs.co/en/blog/scaling-sub-second-streaming-multi-agent-orchestration |
| Four Signals 用 8 周“吃自己的狗粮”把 Agent Orchestration 用于真实生产应用交付，最终形成多服务、自动化测试、五层 CI Gate、IaC 与预览部署；其核心经验是把 Agent 团队当生产线而不是聊天机器人，先建设持久工作对象和工程门禁再扩 Agent 数量，适合补充 Agentic Software Factory 的落地方法。 | https://foursignals.dev/blog/2026-07-31-agent-orchestration-adventures-to-build-a-production-grade-application/ |
| Agents Squads 复盘 19 个生产中的 AI Agent 小队，选择不依赖中央编排器，而用 GitHub Issues 做工作队列、Markdown 做持久记忆、PR 做交接产物，让跨小队协作建立在现有工程原语上；适合对比 Supervisor/DAG 之外的“组织约定式编排”。 | https://agents-squads.com/engineering/ai-agent-orchestration-19-squads/ |
| 这项 2026 制造业研究用 Fetch.ai + Asset Administration Shell 真正实现产品 Agent、资源 Agent 和 AI 服务 Agent 的去中心化协作，并在定制制造 Testbed 中验证，同时记录 Agent/资产发现、跨企业协同和实时推理等现实难点；适合作为“工业多 Agent 从标准语义到现场执行”的研究案例。 | https://www.sciencedirect.com/science/article/pii/S2212827126009352 |
| ARC Advisory Group 从工业企业同时存在 ERP/EAM/TMS/SCADA 多家 Copilot 的现实冲突出发，提出用 MCP/A2A 和统一治理控制面仲裁跨域目标，适合分享多 Agent 落地到棕地企业后“多厂商 Agent 如何互联、冲突与治理”的问题。 | https://www.arcweb.com/blog/curing-copilot-tower-babel-orchestrating-multi-vendor-copilots-across-extended-industrial |
| AWS 2026 年 8 月把多 Agent 扩展推进到企业平台层，重点处理多框架、多模型、多供应商、多团队共存时的统一身份、治理、可观测与路由，适合分享“从单个多 Agent 系统扩到企业控制面”后的架构原则。 | https://aws.amazon.com/blogs/machine-learning/scaling-agentic-ai-enterprise-patterns-without-vendor-lock-in/ |
| AWS Assess Workbench 把架构、安全、风险、合规等专业 Agent 用“AI 规划 + Step Functions 确定性执行 + 人工审批”组合起来，并在 3–8 分钟内完成原本可能需数周的结构化评审，适合展示强监管场景中的协作、审计与 HITL。 | https://aws.amazon.com/blogs/industries/build-a-multi-agent-assessment-workbench-with-amazon-bedrock-agentcore/ |
| Assess Workbench 开源仓库提供可部署的多 Agent 文档评审系统，包含并行/串行 Agent 组、共享记忆、质量 Judge/Coach、证据可追溯、实时进度、Terraform 与 OpenTelemetry，适合作为“生产级协作闭环”的可复现项目。 | https://github.com/aws-samples/sample-assess-workbench |
| KTern.AI 用 AgentCore + Strands 把多个专业 Agent 部署到长期 SAP 转型流程，披露新 Agent 4–6 小时可上线、99.8% uptime、项目周期缩短 45% 等生产指标，适合用真实企业数据说明多 Agent 平台化收益。 | https://aws.amazon.com/blogs/machine-learning/how-ktern-ai-built-agentic-ai-for-sap-on-amazon-bedrock-agentcore/ |
| AWS 的半导体企业案例把层级式多 Agent 做成全生命周期平台，在每次工具调用执行 Zero Trust，并结合 canary/自动回滚、异常 Agent 隔离和合规审计，适合分享身份、权限、版本与故障治理如何系统化。 | https://builder.aws.com/content/3C6RKnGhT7pBovw7sj1urk5ZeuK/building-a-production-grade-multi-agent-platform-with-zero-trust-governance-on-amazon-bedrock-agentcore |
| 这项 2026 研究基于 208 个生产派生场景，把规模从个人级扩到 200 Agent 企业级，对比 DAG Plan-and-Execute 与 ReAct，并量化发现 Agent 发现噪声是规模瓶颈；Task Manager 可将高优先级延迟降低 14–75%，适合用数据讨论规模化编排。 | https://arxiv.org/abs/2606.20058 |
| CAMCO 把企业多 Agent 协作建模为带硬约束的运行时优化问题，通过策略投影、风险权重与协商协议在部署阶段执行合规控制，并展示零策略违规和 92–97% 效用保留，适合补充 policy-as-code 安全编排。 | https://arxiv.org/abs/2604.17240 |
| Victor Dibia 的 Designing Multi-Agent Systems 从第一性原理实现 PicoAgents，并把工作流、Round-Robin/LLM/Plan-based 编排、可观测、HITL、评估、优化与部署放进同一套可运行代码，还横向对应 Microsoft Agent Framework、Google ADK 与 LangGraph，特别适合作为分享中“从模式到生产”的主线教材。 | https://github.com/victordibia/designing-multiagent-systems |
| Azure Cosmos DB 的银行多 Agent Workshop 用真实零售银行场景让专业 Agent 协作处理账户查询、交易与个性化建议，同时提供 Microsoft Agent Framework + LangGraph / Python LangGraph 两条实践路径、分步练习与完整答案，适合现场演示和复现实操。 | https://github.com/AzureCosmosDB/banking-multi-agent-workshop |
| Azure Contoso Creative Writer 是端到端多 Agent 应用样例：独立 Agent 定义由 orchestrator 串联，前端可查看每个 Agent 输出，并配套 tracing、Coherence/Fluency/Relevance/Groundedness 评估与 GitHub Actions CI/CD，适合讲“编排之后如何测试、观测和上线”。 | https://github.com/Azure-Samples/contoso-creative-writer |
| Maestro 把 Claude Code、Codex、OpenCode 等 Coding Agent 放进统一控制台，用 Git worktree 隔离并行任务、Playbook 批处理长任务、Moderator 组织 Group Chat，并支持 headless CLI/cron/CI 与成本统计，适合作为“多 Agent 研发工作台”落地案例。 | https://github.com/RunMaestro/Maestro |
| Overstory 把多 Coding Agent 当分布式系统治理：每个 Agent 独立 worktree，SQLite mail 做类型化通信，FIFO merge queue 处理合并冲突，watchdog/trace/metrics 做健康与可观测，并提供多运行时适配和 handoff/checkpoint，适合分享协作规模化后的基础设施设计。 | https://github.com/jayminwest/overstory |
| Harmonist 不把 Review、记忆更新和供应链检查只写进 Prompt，而是用 IDE Hooks 做机械门禁：规则未满足就阻止 Agent 回合结束，并覆盖 subagent 派发与文件修改，适合展示“多 Agent 协作如何从软约束升级为可强制执行的工程治理”。 | https://github.com/GammaLabTechnologies/harmonist |
| 飞书多 Agent 协作方案用一个飞书 Bot 驱动多个独立 Agent，通过群聊与 Session 做上下文隔离、独立工具白名单控制权限，并用 sessions_send 异步委派和脚本化建群/绑定完成部署，适合分享“多 Agent 如何直接嵌入现有团队协作工具”的可复现实操。 | https://github.com/hyperlist/feishu-multi-agent |
| Claude Agent Framework 以 Lead + 专业 Subagents 组织复杂任务，内置 Research、Pipeline、Critic-Actor、Specialist Pool、Debate、Reflexion、MapReduce 七类模式，并提供成本/重试生命周期插件和 JSONL 全链路观测，适合做协作拓扑与生产能力对照。 | https://github.com/uukuguy/claude-agent-framework |
| 阿里云从 Agent Native Cloud 的整体架构解释 AgentRun、AgentTeams、AgentLoop 如何把运行时、协作治理、审计评估和持续优化组合成企业级 Agent 平台，适合分享“多 Agent 如何从功能编排升级为组织级基础设施”。 | https://zhuanlan.zhihu.com/p/2062599702143672718 |
| 文章把多智能体协同系统类比 Kubernetes，围绕 Worker、Team、Human、Manager、控制循环、Matrix 协作底座和 AI 网关展开，适合讲清“声明式编排 + 治理平面”如何支撑可复制、可治理、可演进的 Agent 团队。 | https://zhuanlan.zhihu.com/p/2050905023157122907 |
| 从企业 To B 落地视角系统讨论多 Agent 协作治理，覆盖 Leader-Worker、AI Registry、凭证托管、IM 人在回路、审计与成本控制，适合补充“技术编排之外，企业为什么还需要组织与安全治理”。 | https://www.zhihu.com/question/2002294556490747945/answer/2057881050760721196 |
| Cornucopia Multi-Agent 提供顺序、并行、辩论、专家团队、审核和层级协作等多种模式，并配套多种通信拓扑、FastAPI 与可视化界面，适合用一个可运行项目对比不同协作结构及其适用场景。 | https://zhuanlan.zhihu.com/p/2046654066827203602 |
| 用 OpenClaw、Discord 与 ACP 实际搭建多个 Agent 的公开频道协作，并让 Agent 调用 Claude Code 或 Codex 执行编码任务，适合展示轻量多 Agent 团队如何通过通信空间、角色隔离和外部 Coding Agent 真正落地。 | https://zhuanlan.zhihu.com/p/2018714727715467590 |
| 通过 Claude 澄清需求、Gemini 设计原型、Codex 并行编码与测试组成端到端开发团队，并用 Skills 自动激活和质量门禁串联流程，适合分享跨模型 Agent 如何按能力分工而不是让单一模型包办全部研发环节。 | https://zhuanlan.zhihu.com/p/2005277073036566597 |
| 从真实开发任务依赖出发解释哪些任务可并行、哪些必须串行，并讨论多 Agent 同时改代码时的冲突问题，适合用于讲解 AI Coding 多 Agent 的任务图、并行度控制和协作边界。 | https://zhuanlan.zhihu.com/p/2023489705921004804 |
| 用 12 个 Claude Code Agent 按研发岗位组成完整团队，设置阶段流水线、契约节点、独立 QA、失败打回与人工兜底，适合展示多 Agent 规模扩大后如何靠流程门禁维持可交付性。 | https://zhuanlan.zhihu.com/p/2045030955761521332 |
| 把可连续运行 10 小时以上的本地全栈自治开发工作流演进成 Lead、Backend、Frontend、QA 分工的 Agent Team，并用文件状态、并行执行、自动测试和 Agent 间报错交接跑真实项目，适合展示多 Agent 从单线程自治到可验收团队协作的实战路径。 | https://juejin.cn/post/7613970761351430144 |
| 围绕通义灵码 Agent Team/Quest 给出 Planner、开发、QA 的完整任务拆分，结合 Worktree 隔离、Spec 驱动、验收/回退和电商秒杀案例，适合补充国内 AI Coding 产品如何把团队协作落到真实研发流程。 | https://juejin.cn/post/7631179160284250163 |
| 横向比较 agent-team、Claude Agent Teams、Claude Squad、MetaGPT 的隔离机制、通信、角色复用、模型支持、质量门禁与任务管理，并给出多 Worktree 并行实操，适合做多 Agent Coding 架构选型的对照材料。 | https://juejin.cn/post/7614788881902551046 |
| 把 Multi-Agent 放进完整生产工程体系，比较 Orchestrator、层级、P2P 三类协作模式，并讨论框架选型、共享记忆、通信爆炸、责任归因、成本、过度设计和可观测性，适合做分享的总体架构与反模式地图。 | https://juejin.cn/post/7632507085398949926 |
| 用 SDD + Subagent 把上下文隔离、强制 TDD 红绿重构和“意图/质量”双阶段审查串成协作流程，适合说明多 Agent 的价值不只在并行，还在职责分离和可执行的质量门禁。 | https://juejin.cn/post/7625254134889791514 |
| OpenClaw 多 Agents 配置教程从工作区、路由和角色划分切入，直接展示从单 Agent 到团队协作的配置路径，适合作为分享中“配置层怎样真正把 Agent 组起来”的实操材料。 | https://www.youtube.com/watch?v=WdTX_DdjSqE |
| 通过阅读 OpenClaw 源码提炼多 Agent 自动协作的三个关键配置，适合补充表面教程背后的运行机制与配置取舍，帮助解释“为什么这样配才能协作”。 | https://www.youtube.com/watch?v=2x3DYrVhuR4 |
| 开源方案用一条命令让 Claude、Codex、Gemini 组队执行任务，适合展示异构模型如何被统一调度，并作为跨模型 Coding Agent 协作的可复现实操案例。 | https://www.youtube.com/watch?v=uf6xyl9ppKk |
| 对 Cognition《Don't Build Multi-Agents》的精读从反面讨论多 Agent 复杂度与失效边界，适合分享中加入“什么时候不该上多 Agent”的判断框架，避免只展示正向案例。 | https://www.youtube.com/watch?v=297owCxa4I8 |
| Hermes Agent 的看板模式直接演示多 Agent 协作调度，适合讲共享任务板、状态推进和显式控制面怎样替代纯对话式派活，作为协作运行机制的直观案例。 | https://www.youtube.com/watch?v=cw12fxm6yNE |
| Codex Multi-agent V2 支持 Kimi、MiniMax、GPT 多模型混用、动态派生 subagent 与并行执行，适合展示 Graph Engineering、动态 fan-out 和跨模型角色分工如何落到 Coding Agent 工作流。 | https://www.youtube.com/watch?v=RAFQc6zHdXE |
| AgentMesh 是开源多智能体协同框架，支持零代码定义 Agent、复杂任务拆解、多轮自主决策以及浏览器/搜索/文件/终端等工具，并在 Demo 中让软件开发 Agent 团队完成网页设计、开发、浏览器测试及文档/代码交付，适合作为“多 Agent 团队如何从框架能力落到完整研发闭环”的项目案例。 | https://www.bilibili.com/video/BV1QKLCzwEuy |
| QM 把多 Agent 从个人助手推进到团队级工作台：每位成员拥有隔离的记忆、文件、密钥、权限和持久沙箱，又能在 Slack/Web 的共享空间协作；同时支持多种 Agent 引擎、共享技能、后台任务与安全策略，适合分享“多人 + 多 Agent”落地时如何同时处理协作、隔离和治理。 | https://www.bilibili.com/video/BV1e88G6KEo8 |
| ClawChat + Hermes 直接把 Agent 放进真人群聊，让 Agent 可主动交流并与人共同完成邀请函与抽奖应用等任务，适合展示多 Agent/人机协作如何从后台编排进入现有沟通场景，并形成真实共同交付。 | https://www.bilibili.com/video/BV1rnun6QEMY |
| Spotify 官方把媒体规划拆成 Router、Goal/Audience/Budget/Schedule/MediaPlanner 等专业 Agent，并用 Google ADK 并行执行、FunctionTool 接真实业务数据；上线结果把 15–30 分钟人工流程压到约 5–10 秒，还复盘 Agent 边界、Prompt 测试和工具 grounding，适合作为有真实指标的多 Agent 业务落地案例。 | https://engineering.atspotify.com/2026/2/our-multi-agent-architecture-for-smarter-advertising |
| Databricks 官方给出可部署的多 Agent Apps 实践：由 orchestrator 把 Databricks Apps Agent、Genie 与 Serving Endpoint 都当作子 Agent 工具统一路由，并明确 app-to-app OAuth 等生产接入要求，适合展示多 Agent 怎样从单进程编排扩展成可独立部署的企业服务。 | https://docs.databricks.com/aws/en/generative-ai/agent-framework/multi-agent-apps |
| Microsoft 用 On-Call Copilot 把事件响应拆成 4 个 Agent，并部署到 Foundry Hosted Agents，让同一 incident payload 并行生成技术分诊、沟通和复盘产物；案例同时讨论生产编排决策，适合分享 SRE 场景如何用多 Agent 缩短故障响应链路。 | https://techcommunity.microsoft.com/blog/azuredevcommunityblog/building-a-multi-agent-on-call-copilot-with-microsoft-agent-framework/4499962 |
| agent-orchestration 用 MCP 给多个 Coding Agent 提供共享记忆、任务队列、Agent 发现、资源锁和自动上下文同步，直接针对重复工作、并行改文件冲突和 context drift 等协作痛点，适合展示“先补协作基础设施，再增加 Agent 数量”的轻量落地方式。 | https://github.com/madebyaris/agent-orchestration |
| AOa 把 Claude Code、Codex 等异构 CLI Agent 收敛到统一 dispatch contract，提供共享任务状态、checkpoint、telemetry、rate-limit 恢复，并强制 worker 与 tester 身份分离后才能完成任务，适合分享跨模型协作里的可恢复执行与独立质量门禁。 | https://github.com/InonB2/multi-agent-orchestration |
| git-meta-harness 把多 Agent 软件交付变成可移植 Harness：用 7 类 persona、传感器与 invariants 强制 feature flow，并提供项目接管、健康评分、Prometheus/Slack 监控等能力，适合展示协作角色之外如何用机器规则持续保证流程和质量。 | https://github.com/brenonaraujo/git-meta-harness |
| Orloj 将 Agent、工具、模型、记忆、审批、策略、worker、trace、metric 与部署拓扑都声明为版本化资源，并借鉴控制器、lease、desired state 等基础设施模式运行多 Agent，适合分享“Agents as infrastructure”式生产控制面和治理思路。 | https://github.com/orlojHQ/orloj |
| Orkes 将 Agent 的概率型推理与 Conductor 的确定性、持久化执行拆开，用工作流承接长任务、审批、重试与可解释执行，适合分享“LLM 负责判断，可靠性由编排层兜底”的生产架构。 | https://orkes.io/blog/agents-on-conductor-architecture-for-production-ai |
| Commonly 是开源的人类 + 异构 Agent 协作空间，让 Claude Code、Codex、OpenClaw 等 Agent 拥有持久身份、记忆、工作站与共享任务板，并可自领任务、提交 PR，适合展示“Agent 团队协作层”如何产品化。 | https://github.com/Team-Commonly/commonly |
| AgentConnect 用 ACP 把多种 Coding Agent 接入 Slack、Discord、GitHub/GitLab 等既有工作场景，支持角色、Agent 间调用、记忆、按 Agent 配置权限与运行环境，适合展示跨渠道、跨 Runtime 的多 Agent 协作落地。 | https://github.com/agentconnect-md/agentconnect |
| SimScale 与 CoLab 的真实 PoC 让设计评审 Agent 直接调用仿真 Agent，自主设置/运行仿真并把结果和澄清问题返回原评审界面，同时保留人工校验，适合展示跨产品 Agent-to-Agent 协作进入工程设计流程。 | https://www.simscale.com/blog/agentic-engineering-design-review-simulation-workflows/ |
| AccordAgents 把多 Coding Agent 协作设计成带 Gate 的交付流水线：签字需求/设计、独立 worktree、强制独立 Review、用户 Review 与最终证据化 QA，适合分享“多 Agent 靠可验证交接而不是群聊协作”的实践。 | https://accordagents.com/blog/coordinate-ai-coding-agents/ |
| Accord 的共识机制先让 Agent 独立提交冻结方案，再统一综合、提出异议、修订并对同一版本签字，专门降低群体锚定和“多份答案无人决策”问题，适合分享多 Agent 决策与交叉复核机制。 | https://accordagents.com/blog/accord/ |
| ITECS 将 Agent 委派定义为“有边界的任务合同 + 最小权限 + 证据化 handoff + 明确验收责任 + 失败控制”，适合补充生产级多 Agent 中任务交接、权限衰减与责任归属的治理方法。 | https://itecs.ai/insights/ai-agent-delegation-contracts-handoffs |
| 这份 2026 年新综述专门梳理多 Agent LLM 的协作机制与结构性限制，重点从“如何协作”而不是“堆多少 Agent”组织研究脉络，适合为分享建立协作机制分类和后续工程案例的理论骨架。 | https://papers.ssrn.com/sol3/papers.cfm?abstract_id=7243979 |
| Warp 从 handoff、共享上下文、显式触发与人工审批四个工程要素拆解多 Agent 编排，强调“多个 Agent 同时跑”不等于“已完成协作”，适合作为分享中解释编排层最小职责的简洁案例。 | https://www.warp.dev/articles/multi-agent-orchestration-explained |
| Airtable 用 hub-and-spoke、flat mesh、hierarchical 三类架构解释多 Agent 如何组织，并串联 Orchestrator、MCP/A2A 与单/多 Agent 选型，适合作为面向非框架用户的架构模式总览。 | https://www.airtable.com/articles/how-multi-agent-systems-work |
| Agent Board 把多 Agent 协作的共享任务状态做成独立控制面：Kanban、DAG 依赖、质量 Gate、自动重试、任务链、实时评论/Webhook、MCP 与完整审计都可直接运行，并特别处理并发写入与故障恢复，适合分享“Agent 团队怎样靠任务系统而不是聊天记录稳定协作”。 | https://github.com/quentintou/agent-board |
| questpie/agent-board 用纯 Markdown + CLI 保存 goals、tasks、specs、knowledge 与 flows，把 claim 锁、依赖检查、验证证据和 done gate 变成机器约束，并支持本地 multi-agent fan-out、Review 与汇总，适合展示“持久事实源 + 可执行契约”怎样让 Coding Agent 长任务可恢复、可审查。 | https://github.com/questpie/agent-board |
| agent-taskboard 不负责替 Agent 派活，而是在开工前登记文件/模块 scope、自动发现重叠、通过任务线程协商边界，并用 bug 验证循环和异步 standup 维持团队可见性；这种“先消灭重复劳动和编辑冲突”的设计很适合作为多人多 Agent 并行开发的协作治理案例。 | https://github.com/chenjxin/agent-taskboard |
| AI Agent Board 把 Copilot、Claude Code、Codex、OpenCode 等 Coding Agent 收敛到同一 Kanban UI，每个任务可选择 repo、branch、worktree 并实时流式展示执行过程，最后由人 Review、合并或开 PR，适合分享“多 Agent 执行层 + 人类项目控制面”的产品化落地。 | https://github.com/DanWahlin/ai-agent-board |
| Onetree 反其道而行，不靠每 Agent 一个 worktree，而是在同一 working tree 上用文件 lease、原子任务板、跨 Agent 定向消息、共享验证记忆与统计指标避免互踩，并能测量 speedup、utilization、critical path；适合对比“隔离式并行”之外的实时协同模型。 | https://github.com/kedarvartak/onetree |
| Agent Collab 把多 Coding Agent 协作定义成 runtime 无关的 worktree-first 协议，任务、handoff、Review、测试报告、风险、ADR 与 merge recommendation 都持久化在 `.agent/`，角色职责和人工审批 Gate 也被显式约束，适合提炼一套可跨 Claude/Codex 等工具复用的团队工程规约。 | https://github.com/egesabanci/agent-collab |
| AI Sidekicks 把“session”而不是单个 Agent 作为协作核心，让多人、Claude Code、Codex 等 Agent 在跨机器共享空间里看到统一时间线、暂停/steer、审批高风险动作，并用加密频道和 git-worktree 流程协同交付，适合补充多 Agent 从后台编排走向真正多人协作产品的形态。 | https://github.com/Sawmonabo/ai-sidekicks |
| agtx 用共享黑板/Kanban 承载持久任务，再由 Orchestrator 自动拆解、委派、推进阶段和检查冲突；不同阶段还能切换 Gemini/Claude/Codex 等 Agent，每个任务用独立 worktree + tmux 并行执行，适合展示“看板状态机 + 异构 Agent + 隔离并行”的完整研发工作流。 | https://github.com/fynnfluegge/agtx |
| Agent Teams 是面向长期运行团队的自托管编排层，让多个 Hermes Agent 以明确岗位、角色和队友关系协作，并提供实时 Dashboard、团队脚手架以及 TLS、防火墙、Docker 隔离和 API Key 等 VPS 部署说明，适合分享“Agent 团队从本地 Demo 到可运维服务”需要补哪些基础设施。 | https://github.com/CyberTron957/agent-teams |
| myclaude 用 Claude Code 做 Orchestrator，把 Codex、Claude、Gemini、OpenCode 统一成多后端执行层，并沉淀 5 阶段 feature workflow、智能路由、BMAD 专业 Agent 和需求到代码流水线；项目已有较高社区采用度，适合展示跨模型 Coding Agent 协作如何封装成可安装、可复用的工程工作流。 | https://github.com/stellarlinkco/myclaude |
| AgentRun 在 A2A 协议之上补齐 AgentCard 服务发现、工作空间、多环境隔离、权限与注册治理，并用完整示例跑通远程 Agent 调用链，适合分享“协议规范如何真正变成生产级多 Agent 管理系统”。 | https://zhuanlan.zhihu.com/p/2017252789424768035 |
| AgentScope 将 A2A 与 Nacos 注册中心结合，实现跨语言、跨框架 Agent 的统一发现、健康检查、命名空间隔离、错误重试与长任务管理，适合讲企业多 Agent 从本地协作扩展到分布式服务后的治理实践。 | https://zhuanlan.zhihu.com/p/1999905814828323456 |
| 文章以智能工厂运维为场景，用 LangGraph Supervisor 组织设备查询与维保调度 Agent，并覆盖状态/记忆、Human-in-the-Loop、LangFuse 追踪评测及完整源码，适合作为可复现的工业级多 Agent 实战案例。 | https://zhuanlan.zhihu.com/p/1964805481429172552 |
| 从 Demo 到生产的工程指南把多 Agent 拆成 Router、Planner、Worker、Critic 等职责，并强调状态机、局部重试、异常上抛、可观测与人工干预，适合提炼“用确定性控制面约束概率型 Agent”的生产架构方法。 | https://zhuanlan.zhihu.com/p/2016265577321219644 |
| Routa 的 Agent Team 实践围绕 Token/成本约束、Specialist 角色化、状态外置和 MCP 跨 Agent 通信设计可演进的软件开发团队，适合分享多 Agent 如何从临时 Prompt 升级为可复用工程系统。 | https://zhuanlan.zhihu.com/p/2011849479800787650 |
| 阿里云从 Agent 群聊的组织建模切入，结合 AgentLoop 的 TeamLeader + 多 Worker 研发流水线讨论上下文持久化、身份权限、凭证治理与成本归因，适合讲什么时候需要真正的多 Agent 协作空间以及企业治理边界。 | https://zhuanlan.zhihu.com/p/2055613904156468645 |
| OpenClaw 多智能体实战从任务分解和专业 Agent 协作入手构建完整工作流，适合作为轻量多 Agent 团队如何从角色划分走到可运行流程的入门案例。 | https://juejin.cn/post/7613796323045883947 |
| 文章聚焦 Manus 团队的上下文工程经验，专门讨论多智能体协作中的记忆与上下文组织难题，适合补充“协作质量最终受上下文设计约束”的实践视角。 | https://juejin.cn/post/7532995195127676937 |
| Multica 把 AI 编程 Agent 做成可管理的团队成员，结合 Kanban、技能复用、WebSocket 实时推送和统一运行时，适合展示多 Agent 从脚本编排进一步走向团队协作产品的形态。 | https://juejin.cn/post/7628251518199365686 |
| 文章把多智能体协同归纳成五种核心架构，并围绕不同 Agent 如何通过代码协调展开，适合在分享中快速建立协作模式地图，再连接具体工程案例。 | https://juejin.cn/post/7627870163742162994 |
| 从 Workflows、Multi-Agent 一直讲到 Production 的框架化视角，适合在分享中先建立单 Agent、工作流与多 Agent 的边界，再讨论为什么生产系统需要更明确的编排与工程约束。 | https://www.youtube.com/watch?v=ZVPlLaehjLk |
| Google Cloud Tech 从架构层讨论多 Agent 系统，适合用于梳理职责拆分、协作拓扑和系统边界，并作为后续 ADK/A2A 实践案例的架构背景。 | https://www.youtube.com/watch?v=j_l-9uNX2SA |
| IBM Technology 重点讨论 Agent 系统构建与规模化挑战，适合补充多 Agent 从 Demo 扩展到真实系统后会遇到的复杂度、可靠性与工程治理问题。 | https://www.youtube.com/watch?v=fCHe_fOqlYA |
| A2A Workshop 直接面向可互操作多 Agent 系统进行实操，适合分享 Agent 发现、跨服务通信与协议化协作如何从概念落到可运行实现。 | https://www.youtube.com/watch?v=EpATeUY30GI |
| 用 Google ADK 与 A2A 让多个 Agent 协作完成游戏设计，是较具体的端到端案例，适合展示专业 Agent 如何分工、互相调用并共同完成一个可观察的任务。 | https://www.youtube.com/watch?v=nGjGUCOiXk4 |
| GitHub 场景下“一名开发者、两打 Agent、零对齐”的协作复盘很适合做反例材料，用来讨论 Agent 数量增加后的人类注意力、上下文对齐、任务边界与协作成本为何会成为瓶颈。 | https://www.youtube.com/watch?v=ClWD8OEYgp8 |
| 通过 CMUX 同时编排 Claude Code 与 Pi Agent 的演示，适合展示异构 Coding Agent 如何进入同一执行控制面，并观察跨 Agent 调度、并行任务与人工监督的实际工作方式。 | https://www.youtube.com/watch?v=WAFUMBLOjHo |
| WorkBuddy 把多人和多 Agent 放进同一团队空间，并直接涉及权限、评论与版本管理，适合作为“多 Agent 协作不只需要编排，还需要团队级协作与治理界面”的产品化案例。 | https://www.bilibili.com/video/BV1yQgP6JEfX |
| Pi Subagents 可把复杂任务委派给多个专职子 Agent，并行执行代码审查、项目分析、功能开发与安全检查，适合作为 Coding Agent 中任务拆分、专业分工和并行执行的轻量落地示例。 | https://www.bilibili.com/video/BV1vL3o6UE5T |
| 该 Codex 实战完整演示同项目新线程、跨项目任务、会话 ID 接力以及主 Agent 指挥多个工作 Agent 并行调研四种协作方式，还覆盖分发、监督、验收和汇总，特别适合分享任务交接与 1 对多编排。 | https://www.bilibili.com/video/BV1L73o67EGh |
| Hermes Bot Mode 让每个 Bot 拥有独立角色、模型、记忆和技能，并通过 Agent Inbox 互相协作，可直接组合研究、写作等专职 Bot，适合展示“独立 Agent 身份 + 消息通道 + 专业分工”的协作产品形态。 | https://www.bilibili.com/video/BV1zs8U6LEk5 |
| 该内容专门讨论多 Agent 协作中的记忆管理误区，适合作为反例材料补充共享记忆、上下文污染与协作状态设计的问题，避免分享只讲“怎么并行”而忽略“共享什么状态”。 | https://www.bilibili.com/video/BV1o7Lb67EhT |
| 这期 Agent 实践聚焦多智能体通信和上下文管理，并提供持续更新的代码与笔记项目，适合从工程层讲清楚 Agent 之间如何传递信息、管理上下文并把协作机制落成可复现实现。 | https://www.bilibili.com/video/BV1nmTF6dEkz |
| OpenCode 多 Agent 多线程实战强调同一时间并行推进更多工作，适合用于讨论 Coding Agent 的并发执行、任务切分与吞吐提升，以及并行带来的协调成本。 | https://www.bilibili.com/video/BV1TYud6hEk9 |
| Coze 多 Agent 实战以课程开发助手为具体业务场景，从低代码方式搭建可用的多智能体应用，适合补充非研发团队如何通过角色分工把多 Agent 协作快速落到内容生产流程。 | https://www.bilibili.com/video/BV1vx4y1H74f |
| OpenAI 2026 年 GPT-5.6 构建者指南汇总了生产项目的技术经验，并专门讨论 Multi-agent、Responses API 与程序化工具调用的组合，适合分享“新一代模型能力如何真正转化为可控的多 Agent 系统设计”。 | https://openai.com/index/builders-guide-to-gpt-5-6/ |
| Anthropic 2026 Agentic Coding 趋势报告把“单 Agent 演进为协同团队”列为核心趋势，并结合真实企业案例与人类监督方式讨论多 Agent 协作，适合用数据和案例说明组织级落地正在发生什么变化。 | https://resources.anthropic.com/2026-agentic-coding-trends-report |
| AWS AgentCore Runtime Instances 面向生产级长时运行 Agent，支持多个 Agent 在同一持久会话中协作、共享文件系统、GPU、暂停恢复和最长 14 天会话，并给出代码编写/审查双 Agent 示例，适合讲多 Agent 的基础设施落地。 | https://aws.amazon.com/blogs/aws/runtime-instances-persistent-compute-for-production-ai-agents-on-amazon-bedrock-agentcore/ |
| AWS 用 Cedar 给出多 Agent 委派链的最小权限参考实现，分别约束 Agent→Tool、Agent→Agent 与原始用户授权，并处理身份透传、委派深度和能力校验，适合作为企业多 Agent 安全治理的工程案例。 | https://aws.amazon.com/blogs/security/enforce-least-privilege-authorization-in-multi-agent-ai-chains-using-cedar/ |
| Kiro Crew 把持久记忆、跨会话任务、并行子 Agent、检查点/重试、审批与可观测性整合成开源研发工作空间，针对真实工程“跨仓库、跨工具、跨天”的协作形态，适合展示多 Agent 开发团队如何长期运行。 | https://kiro.dev/blog/introducing-kiro-crew/ |
| OpenAI Agents SDK 新增 Hosted multi-agent 实验能力，由 GPT-5.6 根 Agent 在服务端创建并协调子 Agent，同时保留本地工具执行、并发限制、调用者元数据与幂等/授权控制，适合分享托管式多 Agent 编排的新落地形态。 | https://github.com/openai/openai-agents-python/blob/main/docs/models/index.md |
| CIRP Annals 论文在汽车供应商数据上用 Strategy、Analysis、Data、Simulation 四类 Agent 完成生产计划，并以 7 组实验验证结构化 handoff、上下文工程与仿真/优化协作，适合补充真实工业场景的多 Agent 实证案例。 | https://doi.org/10.1016/j.cirp.2026.04.027 |
| 文章基于 27 组受控实验总结多 Agent 编程团队的五条实践规则，包括 3–5 个 Agent、共享目录但限制写入范围、集成测试与故障注入、专职 DevOps Agent、重复运行报告方差，适合作为可验证的工程经验。 | https://junjietang.dev/blog/2026/five-rules-multi-agent-coding-teams/ |
| Meta-Agent Teams 用角色/关系定义团队，由 meta-agent 根据人类反馈持续改进，再用独立 auditor 审核，并通过 Git 记录每次演化，适合分享“多 Agent 团队如何形成可追踪的持续改进闭环”。 | https://github.com/jbrahy/meta-agent-teams |
| 2026-08 的 Agent Toolkit ADR 把本地 Coding Agent Swarm 的协作边界落到可执行工程约束：独立 worktree、durable handoff、后端无关 runner，以及 Token、成本、并发与迭代预算，适合分享“多 Agent 并行研发怎样同时治理隔离、审计和成本”。 | https://github.com/ulises-jeremias/agent-toolkit/blob/main/docs/adrs/ADR-008-swarm-orchestration.md |
| Temporal 社区用 Google ADK + LangGraph 做了可运行的跨框架多 Agent Demo，并让人工审批、长等待和 Agent 崩溃后的 handoff 都可持久恢复，适合讲“概率型 Agent + durable execution + HITL”如何组合成生产可靠性。 | https://github.com/temporal-community/durable-hitl-agents |
| OpenAgents 把 A2A、MCP、WebSocket、gRPC 与 HTTP 汇入同一 Agent Network，支持中心化/去中心化拓扑与 Agent 发现通信，适合展示跨框架多 Agent 从进程内编排走向网络化协作的基础设施形态。 | https://github.com/openagents-org/openagents |
| LinkedIn SQL Bot 是已被数百员工使用的生产级多 Agent 系统，采用意图路由、逐步 SQL 规划、Validator 与自修复 Agent，并把数据权限和交互反馈整合进产品，适合分享“多 Agent 如何真正嵌入企业数据工作流并获得稳定采用”。 | https://www.linkedin.com/blog/engineering/ai/practical-text-to-sql-for-data-analytics |
| Temporal 2026-08 的 Micro-Agents 分享把“拆小 Agent”类比微服务化，重点讨论上下文、安全、爆炸半径与跨语言/跨框架可靠编排，适合解释为什么多 Agent 的核心难题最终会变成分布式系统协调。 | https://pages.temporal.io/webinar-monoliths-to-microagents.html |
| Cursor 的生产实践从本地多 Agent 实验演进到“每个 Agent 独立 VM + Temporal Workflow”云架构，并复盘长任务、版本升级、部署稳定性和可观测性，适合分享 Coding Agent 上云后真正需要的运行时能力。 | https://replay.temporal.io/speakers/jeremy-stribling |
| 这篇生产经验复盘从传统“中央 Orchestrator + 一组 Agent”的失效案例出发，聚焦上下文膨胀、指令冲突、漂移、长任务可靠性和 Token 爆炸，适合做“为什么 Demo 架构上线后会坏”的反模式材料。 | https://sdivye92.medium.com/rethinking-multi-agent-systems-c06d1a354aa6 |
| 作者基于半年让多 Agent 向真实用户交付代码的经验，给出并行 worktree、独立安全/质量 Review、人类架构与产品审批、Git 流程等规则，适合分享“多 Agent 提速之后如何把快速变成可安全发布”。 | https://medium.com/%40benovedoz/designing-a-multi-agent-ai-workflow-that-doesnt-break-production-792b7ed0f4cd |
| 这份 2026 生产编排指南按 Supervisor、Fan-out、Pipeline、Hierarchical、Debate 等模式拆解 handoff、状态、失败恢复和成本特征，并明确不同模式的适用边界，适合整理成架构选型与失效模式对照表。 | https://www.belsoftsolutions.com/blog/multi-agent-ai-orchestration-patterns-2026 |
| AITeamOps 把 Claude、Codex、Cursor、ChatGPT 在真实项目里的协作规则固化成可触发 Skills，并以 AGENTS.md 统一路由、共享项目记忆、交付复核和 Git/数据库红线形成可执行团队规范，适合分享“多 Agent/多工具协作如何靠流程契约而不是临场 Prompt 保持一致”。 | https://github.com/kross88/AITeamOps |
| agent-team 用 Lead + Teammates、psmux 独立会话、共享邮箱/任务板、JSONL 审计日志和人工 spawn 审批统一编排 Claude CLI 与 Codex CLI，适合展示 Coding Agent 多人并行后的任务控制、隔离、审计与人类监督如何落到一个轻量控制面。 | https://github.com/stardino2lab/agent-team |
| AgentRoom 把动态组队、按角色选模型、任务依赖图、ready-task 并发调度、Lead 汇总和 SSE 群聊可视化做成可直接启动的 FastAPI + React 应用，适合用一个小而完整的项目演示“多 Agent 从拆任务到并行交付”的端到端实现。 | https://github.com/xy1121/agentroom |
| OpenMOSS 用 planner/executor/reviewer/patrol 四类 Agent、任务状态机、审查返工、卡死巡检、cron 自唤醒和 Web 管理台组织 7×24 自主协作，并给出 1M Reviews 的真实无人运营案例，适合分享长时间多 Agent 系统如何补齐质量闭环、恢复机制和运营控制面。 | https://github.com/uluckyXH/OpenMOSS |
| AgentRadio 用可复现实验把“分工→协商→后台被动感知”逐层消融：四 Agent 在 SWE-Atlas QnA 上显著高于单 Agent，并通过后台监听让消息在执行中及时纠偏而不占工作回合，特别适合分享 Agent 间通信语义如何直接影响长任务协作效果。 | https://github.com/Coral-Protocol/AgentRadio |
| ContextHub 把多 Agent 的 Memory、Skill、文档和数据元信息统一成 ctx:// 上下文治理层，提供团队可见性、private→team→org 晋升、版本固定、依赖传播、审计与租户隔离，适合补充“共享状态、记忆版本和权限边界”这一多 Agent 生产落地常被忽略的基础设施。 | https://github.com/The-AI-Framework-and-Data-Tech-Lab-HK/ContextHub |
| AWS 的 Bedrock 多 Agent Workshop 用 Supervisor 协调预测、光伏和峰值负载三个专业 Agent 跑通能源管理业务，并提供从单 Agent 配置到协作调用的完整练习，适合做分享现场可复现的“Supervisor + Specialists”业务型最小实战。 | https://github.com/aws-samples/bedrock-multi-agents-collaboration-workshop |
| 从 Claude Code Harness 工程拆出 Sub-Agent 与 Agent Teams 两层协作，具体到共享任务表、私信/广播、空闲通知以及 TeammateIdle/TaskCompleted Hooks 质量门禁，适合讲“多 Agent 不是多开会话，而是要有可管理的协作控制面”。 | https://zhuanlan.zhihu.com/p/2022795894202925545 |
| 从源码与统一 Task 抽象角度解释多 Agent：把后台任务、teammate、remote agent 统一成可观测、可取消、可恢复的执行体，适合讲状态、恢复和结果回流为何比单纯 Prompt 分工更决定工程可用性。 | https://zhuanlan.zhihu.com/p/2032396538165600546 |
| 作者实际使用 Agent Team 后给出 3–5 个队友规模、模型分级、CLAUDE.md 精简、/loop 长任务以及并行审查/模块开发/竞争假设调试等经验，适合补充“团队规模与成本边界”的一线操作经验。 | https://zhuanlan.zhihu.com/p/2036214495467530078 |
| 同时比较 Agent Teams 与 Dynamic Workflow，并用 fan-out、对抗验证、分类路由、黑板等模式建立编排选型，还复盘 Lead 抢活、Agent 撞车和 Token 乘数等限制，适合做协作模式与工程风险对照。 | https://zhuanlan.zhihu.com/p/2057216454588683953 |
| 把 Subagents、Agent Teams、Git Worktree 与工作流编排放在同一实战里，明确上下文隔离与文件/分支隔离的不同职责，适合讲并行 Coding Agent 如何通过任务规划和工作区隔离减少冲突。 | https://zhuanlan.zhihu.com/p/2033183908523718494 |
| 系统对比跨会话通信、Subagent、Agent Teams 与 Agent View，从创建者、通信能力、上下文和成本解释多会话协作机制，适合梳理“轻量委派—团队协作—可视化监督”的能力层级。 | https://zhuanlan.zhihu.com/p/2069370615308587674 |
| 从系统提示词和 TeamCreate/Task/SendMessage/TaskCreate 等底层机制拆解 Agent Team，兼顾原理、实战与踩坑，适合分享 Lead/Teammate、共享任务和直接通信如何形成真正团队式协作。 | https://zhuanlan.zhihu.com/p/2004486603343671752 |
| 给出 Agent Teams 的适用/不适用条件、Token 成本、任务粒度、文件边界和状态同步限制，适合做“什么时候值得并行、什么时候协调成本反而更高”的实操决策清单。 | https://zhuanlan.zhihu.com/p/2004878637753730888 |
| 文章从单 Chat 演进到多 Agent 系统，给出 Supervisor + Coder/Reviewer/Documenter 的任务拆分、依赖顺序、并行/串行执行和 WebSocket 运行轨迹界面，适合分享“编排控制面 + 用户可见执行过程”如何一起落地。 | https://juejin.cn/post/7628895336027799552 |
| 开源 Agent Teams 编排 Skill 把“建团队、分角色、派任务”从手工配置压成一句话自动组队，并专门区分 Agent Teams 与泛 Swarm 概念，适合展示多 Agent 协作怎样封装成可复用的工程能力。 | https://juejin.cn/post/7606128564415103003 |
| 从 AgentTeams、AgentLoop 到 Claude 群聊讨论多智能体“群聊模式”真正困难的地方，并把协作与治理平台、Leader/Worker 组织关系等放在一起分析，适合补充企业级多 Agent 不是“拉群聊天”，而需要专门协作治理层。 | https://juejin.cn/post/7660312458697228322 |
| 用竞品调研等可复现任务从零搭建 Claude Agent Teams，让不同 Agent 分别搜索、分析、汇总，适合作为现场演示“任务拆分—专业分工—结果汇总”的最小端到端多 Agent 实战。 | https://juejin.cn/post/7613960231039860777 |
| 从单体到 Agent Teams 的架构演进出发，重点给出“什么任务才值得上多 Agent”的判断启发，并围绕独立上下文与协作需求解释拆分边界，适合分享中建立“先判断是否需要团队，再谈编排”的选型框架。 | https://juejin.cn/post/7613359984269148206 |
| 围绕 Orchestrator Agent + MCP 展示 Agent 驱动自动化，适合说明中央编排器如何借助标准工具协议连接多个执行能力并形成可扩展工作流。 | https://www.youtube.com/watch?v=Ons1Fv3IE4U |
| 以 OpenClaw 搭建实际多 Agent Team 并做现场演示，适合观察个人或小团队如何组织多个 Agent 协作完成持续工作，而不只是框架概念。 | https://www.youtube.com/watch?v=bzWI3Dil9Ig |
| SoulSync Demo & Deep Dive 聚焦多 Agent 在 Agentic Automation 中的协作，适合补充业务自动化场景里多个 Agent 如何围绕同一流程分工配合的演示案例。 | https://www.youtube.com/watch?v=arNWaddf-lc |
| Codex Multi-agent V2 实测把代码探索、方案设计、功能开发、测试验证和代码审查拆给不同 subagent 并行执行，还支持每个 subagent 独立选模型与推理等级、控制并发、任务恢复和动态派生，适合用来讲 Graph Engineering 与多 Agent 编码团队的调度闭环。 | https://www.bilibili.com/video/BV1STg46UEFd |
| 作者自己做了一个多 Agent 协作 Demo，把路由、协作、交接、执行、追踪串成完整链路，并明确复盘协作稳定性、任务交接质量、状态共享、运行时工具和可视化等后续问题，适合作为“从能跑到可用”最小控制面的实践案例。 | https://www.bilibili.com/video/BV11sdWBUE4Q |
| 用强模型负责规划和 Review、便宜模型负责执行，并在同一会话复用上下文降低 Token 成本；同时演示 Codex 插件、tmux、Open Agent Teams 与 Orca 等跨工具编排方式，适合分享异构 Agent 分工、成本路由和跨 Runtime 协作。 | https://www.bilibili.com/video/BV1ybKh6LEKJ |
| 以 LangGraph + MCP + RAG 从零搭建企业级多智能体系统，把图编排、工具协议和知识检索放进同一工程实践，适合用于分享多 Agent 应用从角色协作到工具接入、知识增强和业务落地的一体化实现路径。 | https://www.bilibili.com/video/BV1eFqVBnEQX |
| 科研场景把文献分析、创新点生成、代码实现和论文写作拆成 4 个 Agent，并结合 CrewAI、RAG 知识库、长期记忆与反思机制，还复盘 Agent 循环、无效代码和幻觉等失败问题，适合作为多 Agent 垂直业务闭环与失败治理案例。 | https://www.bilibili.com/video/BV1kEqnBHEzi |
| SpringAI 多 Agent 实战覆盖角色划分、消息通信、Skills 调度、RAG 融合和分布式协同，并提供任务完成率/协同效率评估以及消息乱序、任务死锁、调度异常等排障内容，适合补充多 Agent 真正工程化后需要面对的可靠性与评估问题。 | https://www.bilibili.com/video/BV1bEgY6iEah |
| LendingTree 将 Supervisor、教育与匹配三个 Agent 真正跑进房贷生产服务，并结合 LangGraph、MCP、Guardrails、分布式追踪和独立部署；截至 2026 Q1 已处理约 1,960 次会话、12,100 条消息且 97%+ 无需人工升级，适合用真实金融指标讲“多 Agent + 合规 + 可观测”如何落地。 | https://aws.amazon.com/blogs/machine-learning/how-lendingtree-built-a-multi-agent-mortgage-assistant-on-amazon-bedrock/ |
| AWS 用 S3 Files 把分布在 EC2、Lambda、EKS、Fargate 与 AgentCore 的多 Agent 放进同一共享文件工作区，通过目录交接、POSIX 一致性、按 Agent 隔离和去重机制协调中间产物，适合展示“共享文件状态”怎样替代脆弱的 Prompt 传递并支撑异构运行时协作。 | https://aws.amazon.com/blogs/storage/orchestrating-multi-agent-ai-architectures-with-amazon-s3-files/ |
| AWS 从共享状态缺失导致重复工作、决策冲突和上下文耗尽出发，用 S3 Vectors 设计“工作记忆 + 个体长期记忆 + 团队共享记忆”三层体系，并给出强一致性、租户隔离、生命周期和可观测实践，适合补齐多 Agent 协作中常被忽视的持久记忆基础设施。 | https://aws.amazon.com/blogs/storage/building-persistent-memory-for-multi-agent-ai-systems-with-amazon-s3-vectors/ |
| AAIF 说明 A2A 已进入中立基金会治理并被 150+ 组织采用，文章给出 HarmonyOS/微信、云平台、金融和供应链等生产互操作案例，适合说明多 Agent 从“同框架内部编排”走向跨厂商、跨组织协作后，为什么需要标准化发现、委派与身份契约。 | https://aaif.io/blog/a2a-joins-aaif |
| 这篇生产架构笔记把多 Agent 的可靠性问题归结到编排控制面：明确协调/执行分离、Agent-to-Agent 与 Agent-to-Tool 两类边界、最小权限/HITL、全链路追踪和成本反馈，适合做框架无关的生产架构检查表。 | https://brittek.net/journal/multi-agent-systems-are-orchestration-systems |
| Google 用 10 支自主 Agent 电影团队做真实长流程协作实验，验证共享文件比消息更适合作为持久协作状态，并通过角色分工、阶段校验 Gate 与崩溃恢复跑通复杂交付，适合分享“多 Agent 如何靠共享事实源和质量门禁稳定协作”。 | https://cloud.google.com/blog/topics/developers-practitioners/what-we-learned-about-agent-teamwork |
| Amazon/ACL Industry 的 Agent-Ops 已在 3 个地区、7 类电商 SOP 和 1000+ Account Manager 中生产落地，端到端准确率达到 85–97%，并把单案处理时间从 30 分钟降到 5 分钟，适合作为有规模、有量化结果的业务多 Agent 案例。 | https://aclanthology.org/2026.acl-industry.29/ |
| 这个 Claude Code Agent Teams 插件把并行代码审查、竞争假设调试、跨层功能开发、研究、安全审计和迁移做成可复用团队模板，适合现场展示“同一套共享任务与消息机制如何映射不同协作拓扑”。 | https://github.com/wshobson/agents/blob/main/plugins/agent-teams/README.md |
| agmsg 用 Bash + SQLite 提供 Claude Code、Codex、Gemini、Copilot 等 CLI Agent 的跨厂商消息层，不引入完整框架或守护进程，适合分享“多 Agent 协作可以先从轻量通信互操作层落地”的基础设施思路。 | https://github.com/fujibee/agmsg |
| 这项长周期任务研究把控制面与执行面分离，用目录服务、QoS 驱动的 Contract Net 动态选择 Agent，并按任务实例动态授予和回收权限，适合补充“多 Agent 协作如何同时处理服务变化、故障与最小权限”的系统设计。 | https://www.aemjournal.org/index.php/AEM/article/view/4269 |
| MAOF 把团队、Kanban、Agent 间消息、异步工作流、共享内存和加密审计日志做进同一生产型控制面，适合分享“多 Agent 不只是编排模型调用，还要补齐任务、协作、运行与审计基础设施”的完整项目案例。 | https://github.com/mrx-arafat/Multi-Agent-Orchestration |
| Mission Control 用 10 个明确岗位的 OpenClaw Agent 配合持久记忆、共享任务/文档空间、@mention 通信、完整任务生命周期和定时心跳长期协作，适合展示“小型 AI 团队怎样从一次性对话变成可持续运营系统”。 | https://github.com/bensheed/mission-control |
| Agent Collab Skills 把任务拆分、上下文预算、结果对账、对抗式辩论、共享记忆、验收 Gate 和 Plan-Act-Reflect 封装成可组合 Skills，并能接 Codex/Gemini 委派，适合提炼跨模型协作中可复用的工程规约。 | https://github.com/WenyuChiou/agent-collab-skills |
| agent-tasks 用 MCP 把 backlog→spec→plan→implement→test→review→done 做成有状态流水线，并加入 DAG 依赖、审批、产物版本、评论、Agent 认领/角色、心跳清理与知识传播，适合说明“共享任务控制面”如何降低多 Coding Agent 的协作混乱。 | https://github.com/keshrath/agent-tasks |
| JockeyUI 以 ACP 统一编排不同 Agent/模型，在桌面端提供角色化模型选择、消息生命周期、上下文渐进披露和跨会话记忆，并实际演示 Claude PM 与 Codex Developer 热切换，适合展示异构 Agent 协作怎样产品化为统一工作台。 | https://github.com/recailai/jockey |
| mini-claude-code 用 Java 按章节从 Agent Loop 一路实现到 Subagent、任务系统、Agent Teams、结构化 Team Protocol、自主认领与 MCP Plugin，适合从源码层讲清“一个能协作的 Agent Harness 到底需要哪些机制”，也便于非 Python 团队复现。 | https://github.com/DerekYRC/mini-claude-code |
| MAS-Resilience 专门研究多 Agent 团队里出现错误/恶意 Agent 时的故障传播，并提供 AutoTransform/AutoInject 攻击注入和 Inspector/Challenger 防御代码，适合在分享中补充“协作规模扩大后如何做鲁棒性验证与防御”的反面实践。 | https://github.com/CUHK-ARISE/MAS-Resilience |
| EMQ Device Agent 以智能座舱为真实工程场景，把数百到上千个车端原子能力按领域拆成多个 Agent，并正面处理跨 Agent 通信、时序依赖、冲突、编排可视化和多供应商集成问题，适合分享“为什么要拆 Agent，以及拆完后真正难在哪里”。 | https://zhuanlan.zhihu.com/p/2071197454326879723 |
| 从生产级 Agent Loop 出发补齐工具超时、输出截断、HITL、错误恢复，再扩展到 Orchestrator+Workers 与 Swarm 两类多智能体模式，适合把单 Agent 工程化能力如何自然演进成多 Agent 编排讲成一条连续路径。 | https://zhuanlan.zhihu.com/p/2064254132505129129 |
| 以 Dify Workflow 落地企业级技术报告生成系统，覆盖角色配置、并行分支、Reviewer 反馈环、流式输出、重试熔断、RAG 质量评估与自定义 Tool，适合做低代码/平台型多 Agent 从 Demo 到可用应用的完整实战。 | https://zhuanlan.zhihu.com/p/2068336512450683853 |
| 在 Plan-and-Execute 基础上用 LangGraph.js、Send、interrupt、Checkpoint 扩展多 Agent 并发协作，具体讨论并行执行、状态持久化、人工审阅、超时重试和断点恢复，适合分享“确定性编排层如何承接 Agent 的不确定性”。 | https://zhuanlan.zhihu.com/p/2058528796987356461 |
| RadarMind 用 5 个业务 Agent + 1 个 TeamLeader 端到端完成科研文献调研与理论路径生成，并复盘 Manager/Worker/Team 房间、Docker 隔离和实际部署踩坑，适合作为 AgentTeams 从安装到真实业务协作的可复现实例。 | https://zhuanlan.zhihu.com/p/2072406449624461470 |
| OpsPilot Zero 把生产故障处置拆成告警归并、根因分析、修复规划、恢复验证 4 个 Agent 的 DAG，并用结构化交付契约、置信度证据链和风险审批控制自动化边界，适合展示高风险运维场景怎样形成可信协作闭环。 | https://zhuanlan.zhihu.com/p/2068110217808700098 |
| 从 AgentTeams baseline 的实际使用出发，拆解 Manager/TeamLeader/Worker、Matrix 协作、Nacos 注册与 Skill 版本治理、Higress 凭证收敛和 OpenTelemetry 可观测，适合补充多 Agent 真正生产化所需的治理基础设施。 | https://zhuanlan.zhihu.com/p/2068115749751804511 |
| 昆仑 AI 分布式多 Agent 企业平台采用 DDD 编排 + AutoGen 原子执行，并把知识库、记忆、异步任务、权限、观测和配置热更新放进九层架构，适合分享企业团队如何把多 Agent 从单个流程升级为长期演进的平台能力。 | https://zhuanlan.zhihu.com/p/2066558348422657228 |
| 真实搭建“人类 + 5 个独立 Agent 团队”的跨境电商组织，PM、Coding、选品、内容与社媒 Agent 各有独立上下文和工具，并通过项目与委派路由协作，适合展示人机混合组织如何进入日常业务而非只做技术 Demo。 | https://zhuanlan.zhihu.com/p/2035422266629038719 |
| Routa 把多 Agent 协作做成独立协调平面，用 ACP 管 Agent 进程、MCP 暴露协作动作、A2A 做联邦扩展，并用结构化任务、事件流、持久状态和恢复机制实现可追踪协作，适合讲协议分层与开放编排。 | https://zhuanlan.zhihu.com/p/2009668693362242633 |
| 从企业 SOP 运行机制出发，把任务实例、唯一运行时主控、状态载体、执行节点、人工审核、异常处理和恢复明确成工程对象，并给出逐步扩大自动化范围的落地路径，适合分享“企业需要的不是更多 Agent，而是可靠协作机制”。 | https://zhuanlan.zhihu.com/p/2074177077218223659 |
| 从 RAG 演进到多智能体协同并进一步接入可观测平台，内容同时覆盖 Agent 架构、开发方案、常见问题与运行观测，适合补充“协作系统上线后如何看见调用链、定位问题并持续优化”的平台视角。 | https://zhuanlan.zhihu.com/p/2003043815884350644 |
| 360 数科的智能营销案例把洞察、策略、文案、执行与效果分析拆给不同 Agent，形成从洞察到触达到转化的业务闭环，并结合真实金融营销场景讨论价值与数据成效，适合做非研发类多 Agent 落地案例。 | https://zhuanlan.zhihu.com/p/1997373801038622880 |
| 用 AutoGen 的 Coder、Reviewer、Executor 跑通一次自动代码评审，并围绕 Planner-Executor-Critic、GroupChat、终止条件与可执行 Python 代码展开，适合作为现场快速复现“生成—审查—执行”协作闭环的入门实战。 | https://zhuanlan.zhihu.com/p/2043436947130012438 |
| 作者在 WorkBuddy + DeepSeek 上把单 Agent 调度、蜂群式多 Agent 并行和深度分析 SOP 做成实际系统，累计跑过 200+ 次蜂群任务、经历 30+ 版本故障修复，并复盘两级调度、文件 I/O 通信与步骤越权问题；适合分享“多 Agent 可靠性为什么必须靠结构约束，而不只是 Prompt 规则”。 | https://juejin.cn/post/7655624648049016851 |
| ERP_OPENCLAW 把 LangGraph/DeepAgents 主从 Agent、MCP 接真实 Java ERP、MongoDB Checkpoint、SSE 可观测与订单 HITL 审批串成采购分析/下单业务闭环，还明确生产化要补权限、审计和自动化测试；适合展示多 Agent 如何真正接入企业系统并控制高风险写操作。 | https://juejin.cn/post/7658236929163280393 |
| ThinkingMap 从单体演进到 Eino 多智能体，基于图式编排、强类型和并发设计 Host/专家 Agent、状态/上下文、依赖与 SSE，并形成分析→规划→执行→反馈的可控闭环；适合补充 Go/Eino 技术栈下多 Agent 工程化架构案例。 | https://juejin.cn/post/7573687816430075914 |
| 从无状态 Agent 逐步演进到多智能体协作，主题直接聚焦 AI Native 开发者的 Agent 工程化路径，适合用来讲清单 Agent 在状态、上下文和复杂任务上的瓶颈，以及为什么最终需要职责拆分与协作编排。 | https://juejin.cn/post/7654244323156951078 |
| 文章以“Agent Demo 上线后为什么失控”为切入点，把重点从 Tool Calling 转向编排层，适合提炼任务规划、执行控制、状态流转和失败治理等从 Demo 到生产必须补齐的能力，作为多 Agent 落地的控制面视角。 | https://juejin.cn/post/7624470162226298907 |
| OpenClaw 多 Agent 部署实践对比单 Gateway 多 Agent 与双 Gateway 独立部署，围绕场景隔离、多角色并行和部署边界展开；适合补充多 Agent 从逻辑分工走到运行时隔离与部署拓扑选择的实操素材。 | https://juejin.cn/post/7611462106036682802 |
| 从单线程 Agent 向 OpenClaw 多 Agent 模式做深度改造，直接围绕上下文拥堵、串行执行和角色拆分带来的价值跃迁展开；适合分享“为什么拆、拆成什么、协作后解决了什么”这一架构演进路径。 | https://juejin.cn/post/7612929520988618787 |
| Oh My OpenCode 的 AgentTeam 实战以可直接使用的 Coding Agent 团队为入口，适合展示多 Agent 编程工具如何从安装配置进入分工协作，并作为跨模型/多角色研发工作流的轻量实践案例。 | https://juejin.cn/post/7614566158142291977 |
| Atlassian Demo Den 直接展示 Jira 中的 Multi-agent Orchestration，适合分享企业如何把 Agent 协作嵌进既有项目协同入口，而不是另起一套孤立工作台。 | https://www.youtube.com/watch?v=SOljsxCH37k |
| Copilot Studio 的实操内容直接围绕 Multi-Agent Orchestration 展开，适合补充低代码企业平台如何组织多个 Agent、降低团队落地门槛的案例。 | https://www.youtube.com/watch?v=xtPlDde4Yv0 |
| 该 Source Demo 从零展示 LangGraph 中多个 Agent 的协作执行，适合作为分享现场可复现的代码型案例，用来讲共享状态、协作流程与编排如何真正跑起来。 | https://www.youtube.com/watch?v=nx6HaySGOlc |
| CrewAI 实操直接聚焦“让 Agent 团队真正协作”，适合用于拆解角色、任务和团队协同如何从框架配置变成可运行工作流。 | https://www.youtube.com/watch?v=hJuPoffsGdc |
| 内容把 Teams、Protocols 与 Worktrees 放进同一套 Multi-Agent Orchestration 视角，适合串联团队组织、Agent 通信和并行工作区隔离这三个落地层次。 | https://www.youtube.com/watch?v=I9lT8m0NoiU |
| A2A 的 System Design 视角直接面向 Multi-Agent Architecture，适合分享跨 Agent 发现、通信与系统边界如何从协议概念落到整体架构设计。 | https://www.youtube.com/watch?v=bkqYfYNkJ_4 |
| Google Cloud Tech 官方介绍 Agent2Agent（A2A）协议，适合作为跨服务、跨框架 Agent 协作的协议基线素材，便于后续连接更复杂的生产案例。 | https://www.youtube.com/watch?v=Fbr_Solax1w |
| 以产品团队实时协作 AI Coding Agents 为主题，适合补充“人类团队 + 多个 Coding Agent”共同工作的实践形态，帮助分享从纯 Agent 编排延伸到真实团队协作。 | https://www.youtube.com/watch?v=QRwAuogj54M |
| 阿里云云原生官方从 HiClaw 到 AgentTeams 展开多运行时 Worker、生产级控制面与企业协作能力升级，适合分享多 Agent 从单点能力走向可管理、可扩展团队平台时需要补齐哪些工程基础设施。 | https://www.bilibili.com/video/BV1QJ3F6UEpe |
| 通过 40 分钟手搓 A2A 客户端与服务端并实现流式交互，且配套代码开源，适合把跨 Agent 通信协议从概念落到可运行代码，展示互操作、调用链和服务化协作怎样真正实现。 | https://www.bilibili.com/video/BV1dXboz5Ekz |
| 在飞书里把文案、知识管理、盯盘、运维、天气等职责拆给独立 Agent，并分别配置人设、技能、群聊绑定和定时任务，适合展示多 Agent 如何嵌入现有协作工具并形成长期运行的专岗 AI 团队。 | https://www.bilibili.com/video/BV1CRPHzfEJZ |
| Hermes + ChatClaw 直接演示多个智能体主动互相沟通、主动协作并与真人共同工作，适合补充多 Agent 从“被动派单”走向主动团队协同和 Human-in-the-Loop 共事的实践形态。 | https://www.bilibili.com/video/BV1eyNR6bE5N |
| 内容围绕 Team 核心概念与四种协作模式展开，并强调按场景选择和构建多 Agent 团队，适合在分享中用作协作模式选型的简洁案例，帮助解释不同团队结构并非一种编排方式通吃。 | https://www.bilibili.com/video/BV1nnLx6UEBT |
| AWS 用 Bedrock 与开源框架系统拆解多 Agent 推理编排，重点讨论 supervisor/worker 协作、任务分解与推理增强，并给出可复现实现，适合用来讲“协作结构如何直接影响复杂任务推理质量”的工程实践。 | https://aws.amazon.com/blogs/machine-learning/design-multi-agent-orchestration-with-reasoning-using-amazon-bedrock-and-open-source-frameworks/ |
| AWS 用 Strands Agents + Llama 4 构建可运行多 Agent 方案，展示专业 Agent 的分工、协调与 Bedrock 部署路径，适合补充“模型、Agent 框架与云运行环境如何组合落地”的端到端案例。 | https://aws.amazon.com/blogs/machine-learning/using-strands-agents-to-create-a-multi-agent-solution-with-metas-llama-4-and-amazon-bedrock/ |
| Google Cloud 用三个 hands-on lab 把 ADK、MCP、A2A 串起来，从工具连接到 Agent 间发现/通信逐步实现可互操作系统，适合现场分享跨服务多 Agent 的协议化落地路径。 | https://cloud.google.com/blog/topics/developers-practitioners/building-connected-agents-with-mcp-and-a2a |
| Hyundai AutoEver 的生产案例在安全多租户 GenAI Sandbox 上落地两套多 Agent AIOps，细讲 LangGraph 状态、并行根因分析、自证伪、HITL 与治理，适合做“真实企业生产级多 Agent”深度案例。 | https://aws.amazon.com/blogs/industries/hyundai-autoever-building-a-multi-tenant-generative-ai-sandbox-and-production-aiops-on-amazon-bedrock/ |
| LangGraph Swarm 提供去中心化 handoff 式多 Agent 实现，支持 Agent 动态交接控制权、共享/定制状态、checkpoint、记忆与 HITL，适合与中心 Supervisor 架构做代码级对照。 | https://github.com/langchain-ai/langgraph-swarm-py |
| BeeAI Framework 同时覆盖 Python/TypeScript 多 Agent workflow、handoff、A2A/MCP、持久化与可观测能力，并提供直接可运行的多 Agent 示例，适合补充跨语言、生产导向的开源框架实践。 | https://github.com/i-am-bee/beeai-framework |
| mcp-agent 用 MCP + 可组合工作流实现多 Agent 模式，并可接 Temporal 获得暂停、恢复和失败续跑等 durable execution，适合分享“Agent 协作如何借成熟工作流引擎补生产可靠性”。 | https://github.com/lastmile-ai/mcp-agent |
| AgentScope 2.0 把 Message Hub、多 Agent workflow、Agent Team、A2A/MCP、HITL、评估与部署放进同一框架，适合从“协作逻辑”一路讲到可观测和服务化运行。 | https://github.com/agentscope-ai/agentscope |
| Microsoft Agent Framework 将多 Agent 的顺序、分支、handoff、状态变化与人工介入从应用代码抽成 YAML 声明式工作流，并已在 Python/.NET 达到 1.0，适合分享“协作编排如何做到可审查、可版本化、可治理”。 | https://devblogs.microsoft.com/agent-framework/move-agent-orchestration-workflows-out-of-code-with-agent-framework-declarative-workflows-1-0/ |
| Microsoft 将 sequential、concurrent、group chat、handoff、Magentic 等编排模式统一稳定到 1.0，并解释不同抽象层和适用边界，适合作为多 Agent 协作模式选型与工程取舍的官方参考。 | https://devblogs.microsoft.com/agent-framework/agent-frameworks-orchestration-patterns-reach-1-0/ |
| Microsoft 用 AG-UI + Agent Framework 构建真实多 Agent 前端，解决 handoff、审批、追问、执行状态流式展示等产品化问题，适合说明“多 Agent 后端能跑”之后怎样做成用户可理解、可干预的系统。 | https://devblogs.microsoft.com/agent-framework/ag-ui-multi-agent-workflow-demo/ |
| Microsoft Agent Framework 对 A2A v1 的实现展示了远程 Agent 的发现、调用与对外暴露，重点解决跨平台、跨组织协作的稳定互操作问题，适合用于讲生产环境中的 Agent 服务边界与协议化协作。 | https://devblogs.microsoft.com/agent-framework/a2a-v1-is-here-cross-platform-agent-communication-in-microsoft-agent-framework-for-net/ |
| a2a cloud 复盘 Agent Studio 从“生成代码/返回 200”升级到必须验证前端、后端、浏览器流程、制品与公开分发真正可用的生产改造，适合分享多 Agent/Agent 平台如何用可验证结果替代“看起来部署成功”。 | https://blog.a2acloud.io/agent-studio-from-prompt-to-proof |
| 2026 年 8 月的软件工程多 Agent 实践报告对主流开源 MAS 框架做定量与统一用例实测，并总结角色、协调规则、框架选型和 telemetry 等真实开发痛点，适合用作分享中的框架比较与踩坑依据。 | https://arxiv.org/abs/2608.11965 |
| Agyn 把软件研发明确建模成协调、研究、实现、评审等角色组成的自治团队，配合隔离沙箱、结构化通信与迭代 Review，并在 SWE-bench 500 上给出实测结果，适合展示“组织设计本身就是 Agent 能力”的工程案例。 | https://arxiv.org/abs/2602.01465 |
| MAO-Bench 专门为多 Agent 编排提供评测与 benchmark 框架，把团队协作从“Demo 能跑”推进到可对比、可回归的工程指标，适合补充多 Agent 落地最容易缺失的评估体系。 | https://github.com/rachitpareek/multi-agent-orchestration-evals |
| Orchestra-o1 采用 MainAgent 动态拆解复杂多模态任务并并行委派给带不同感知与行动工具的 SubAgent，项目同时提供模型与实现，适合展示“任务分解 + 专业 Agent + 并行执行”的完整协作架构。 | https://github.com/zfkarl/Orchestra-o1 |
| AIOps Orchestrator 的 A2A 实现包含能力注册与发现、同步/异步委派、JWT 鉴权、PostgreSQL/Redis 状态和 Prometheus 指标，适合作为“多 Agent 协作进入运维生产环境后需要哪些基础设施”的项目案例。 | https://github.com/Htunn/aiops-orchestrator |
| metaswarm 把 Claude Code、Gemini CLI、Codex CLI 组织为自改进的多 Agent 研发团队，并引入 TDD、质量门禁、规格驱动开发和技能/命令体系，适合分享 AI Coding 团队如何从并行调用升级为受流程约束的工程协作。 | https://github.com/dsifry/metaswarm |
| Scuba Stack 用 chief-of-staff、manager、worker 的层级结构并行派工，并在关键步骤加入独立对抗式 Review 和人工决策门，且整体以可移植 Markdown 规则实现，适合展示轻量但强约束的多 Agent 团队治理方式。 | https://github.com/danielchappell/scuba-stack |
| Ferrus 把 Coding Agent 协作从“多开会话”改造成 Supervisor→Executor→Reviewer 的确定性状态机，用 SQLite 保存运行状态、独立任务产物、自动 Review/返修以及崩溃恢复来约束 Claude Code/Codex 等执行，适合分享“概率型 Agent + 可恢复控制面”怎样工程化。 | https://github.com/ferrus-dev/ferrus |
| CAS（Coding Agent System）用 Supervisor 拆解 EPIC、多个 Worker 在独立 Git worktree 并行实现，再由人 Review PR，并提供跨会话共享记忆，适合展示“任务控制面 + 工作区隔离 + 人工验收”如何组成真实 Coding Agent 工厂流程。 | https://github.com/iflow-mcp/codingagentsystem-cas |
| Olympus 把 Claude Code 扩展成 20+ 专业 Agent 的研发团队，用任务复杂度做模型路由，并以 AI-DLC 工作流、跨会话学习和持续执行机制约束交付，适合分享“角色分工、模型分层与研发流程”如何合成一套可复用团队 Harness。 | https://github.com/mikev10/olympus |
| oh-my-pi 用 `/team` 把 Coding 任务按“拆解→启动隔离子 Agent→并行执行→聚合结果”跑成完整链路，并配套 planner/tester/verifier 等专业角色与持久重试机制，适合作为小而完整的多 Agent Coding 团队落地样例。 | https://github.com/Changhochien/oh-my-pi |
| Sol 面向同时运行 10–30+ 个 Coding Agent 的场景，为每个任务创建独立 worktree，并用 SQLite/tmux 做持久状态、崩溃与卡死检测、自动重启以及经质量 Gate 的合并，适合分享“多 Agent 并发上规模后如何补齐监督、恢复与合并控制面”。 | https://github.com/nevinsm/sol |