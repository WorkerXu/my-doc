# 商品数据采集平台设计文档

> 版本：v2.0 · 更新日期：2026-10-15 · 状态：设计待实施
>
> 这组文档根据《商品数据采集平台实施计划 v2.0》逐模块拆分，保留原方案约定，并以模块边界为独立维护单位。

## 设计目标

**上传已获授权的 APK + 自然语言采集需求 → AI 生成 SourceScheduler、SourceExecutor、SourceValidator → 平台独立验证与准入 → Registry 固定不可变版本 → 平台自动调度采集 → 商品清洗、匹配与发布。**

- AI 按 SDK 生成**来源特有的三个组件**，不重新生成底层 Celery、数据库和 Worker。
- 平台预置 `PlatformScheduler / Runner / PlatformValidator / AdmissionController / Registry`。
- 默认人工确认；启用预授权策略时，在强制测试全部通过后可自动发布和激活。
- Connector 生命周期与商品后处理分离；首次全量建立基线，后续只采新增。
- 当前文档为**拟实施设计**，不表示已完成组件开发、真实 APK 分析或上线验证。

## 模块索引

| 文档 | 主要内容 |
|---|---|
| [项目目标与系统架构](./01-项目目标与系统架构.md) | 范围、部署约束、核心架构决策、平台/AI 职责划分 |
| [Factory 需求与 APK 探索](./02-Factory需求与APK探索.md) | 自然语言需求合同、APK 接口探索与证据治理 |
| [Connector 构建与产物规范](./03-Connector构建与产物规范.md) | Agent 产物目录、依赖锁、Manifest 和证据包 |
| [Connector SDK 与通用数据模型](./04-SDK与通用数据模型.md) | WorkflowSpec、TaskSpec、ExecutionResult 等协议 |
| [SourceScheduler 来源调度插件](./05-SourceScheduler调度插件.md) | AI 生成 DAG、分类展开、分页与任务边界 |
| [SourceExecutor 来源采集插件](./06-SourceExecutor采集插件.md) | 受控 HTTP、认证签名、记录和证据输出 |
| [SourceValidator 与平台强制校验](./07-SourceValidator验证插件.md) | 来源语义验证、平台真值约束、故障注入 |
| [Runner、动态 DAG 与幂等事务](./08-Runner执行事务与幂等.md) | 记录入库、检查点、Outbox 和至少一次执行 |
| [Factory 验证、修复与自动准入](./09-Factory验证修复与准入.md) | Factory 状态机、双层测试、人工与预授权发布 |
| [生产调度、监控与自修复](./10-生产调度与运维自修复.md) | 平台调度器、限流、告警、故障回流与新版本修复 |
| [商品数据建模、清洗匹配与发布](./11-数据清洗同款匹配与发布.md) | 原始商品、标准库、同款规则、人工审核与不可变版本 |
| [工作台、API、权限与安全](./12-工作台权限API与安全.md) | 桌面工作台、对外/内部接口、凭据安全与执行隔离 |
| [实施路线与验收标准](./13-实施路线与验收.md) | 五阶段实施路线、关键验收及容量目标 |
| [架构决策记录](./14-架构决策记录.md) | 本轮架构决策、变更及保留项 |

## 核心模块协作

```text
APK + Prompt
    │
    ▼
Factory：RequirementSpec → DiscoveryEvidence → Builder
    │
    ├─ SourceScheduler（AI 生成）
    ├─ SourceExecutor（AI 生成）
    ├─ SourceValidator（AI 生成）
    └─ Manifest / Lockfile / Fixtures / Evidence
    │
    ▼
PlatformValidator + 隔离 Runner → AdmissionController
    │                              │
    └── 失败证据 → Agent 修复       └── PASS → Registry
                                           │
                                           ▼
                                    PlatformScheduler
                                           │
                                           ▼
                                        Runner
                                           │
                                           ▼
                           原始商品 → 清洗匹配 → 审核发布
```

## 建议的阅读与实施顺序

先看 [系统架构](./01-项目目标与系统架构.md) 与 [SDK 契约](./04-SDK与通用数据模型.md)，然后分别实现 [来源调度器](./05-SourceScheduler调度插件.md)、[采集执行器](./06-SourceExecutor采集插件.md)、[验证器](./07-SourceValidator验证插件.md) 的平台接口。优先用固定模拟来源验证 [Runner 与幂等事务](./08-Runner执行事务与幂等.md) 和 [Factory 自动准入](./09-Factory验证修复与准入.md)，再接入 Pi + Seep 进行真实 APK 分析。

## 文档维护约定

- 新的来源逻辑或接口契约应写入相应模块，不要在总览 README 堆积设计细节。
- SDK、发布准入、运行时接口变更应同步更新相关模块并登记到 [架构决策记录](./14-架构决策记录.md)。
- 准入与运行状态必须基于平台真实测试结果，不使用 AI 自我声明的“已通过”代替验收证据。
- 任何模块进入实施阶段后，明确区分“已实现”“已验证”和“仍属设计”的内容。

## 来源说明

本目录保留了《商品数据采集平台实施计划 v2.0》的架构边界、系统规格、流程和验收要求；章节中的示意契约均为拟建方案，并非已经发布的软件 SDK。
