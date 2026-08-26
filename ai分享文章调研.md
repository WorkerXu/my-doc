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
