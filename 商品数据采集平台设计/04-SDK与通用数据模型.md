# Connector SDK 与通用数据模型

> 设计版本：v2.0 · 来源：《商品数据采集平台实施计划》 · 状态：待实施

[← 返回设计文档索引](./README.md)

---

## 4.1 核心数据模型

平台统一定义并维护版本化 JSON Schema / Pydantic 模型：

| 类型 | 必需信息 | 设计约束 |
|---|---|---|
| `WorkflowSpec` | workflow_id、DAG、触发条件、任务类型、依赖、终止条件 | DAG 无非法环；运行任务绑定固定 connector_hash |
| `TaskSpec` | source_id、connector_hash、task_type、参数、parent_id、逻辑幂等键、优先级 | 输入不可变；不直接包含明文凭据 |
| `ExecutionContext` | 凭据句柄、允许域名、速率/超时预算、运行 ID、取消信号、只读配置 | 由 Runner 创建，不向插件暴露数据库/宿主权限 |
| `ExecutionResult` | records、discoveries、next_cursor、metrics、evidence_ref、error | 不包含“请立即发布/直接写库”指令 |
| `ValidationReport` | rule_id、severity、PASS/RETRY/BLOCK、失败证据、覆盖指标 | 不能改变被测执行结果 |
| `RunCheckpoint` | task_key、处理游标、最后提交批次、connector_hash、状态版本 | 仅由平台事务提交，插件不可自行确认成功 |
