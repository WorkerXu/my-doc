# 工作台、API、权限与安全

> 设计版本：v2.0 · 来源：《商品数据采集平台实施计划》 · 状态：待实施

[← 返回设计文档索引](./README.md)

---

## 8. 工作台、权限和 API

### 8.1 工作台

原有工作台覆盖来源与 APK、AI 分析、流程配置、调度运行、标准库、匹配规则、待审核、最终数据与历史版本、API Key 和审计。新增：

- Factory 阶段视图：需求版本、发现的 API 证据、生成的三组件文件及差异、构建哈希。
- Scheduler DAG 可视化：分类/品牌/列表/详情的 fan-out、分页终止和当前游标。
- Validator 仪表板：来源测试、平台固定测试、故障注入、独立运行、覆盖缺口。
- 发布策略面板：人工/预授权自动模式、授权域名、资源配额、灰度回滚和失败告警。
- 对话修复：Agent 可从持久化失败证据继续修改候选包，未发布草稿不影响生产。

单组织管理员、操作员、审核员角色保持不变；多个外部调用方分别配置 API Key、权限和限流。生成、验证、自动发布策略变更必须授权并审计。

### 8.2 接口建议

既有对外 OpenAPI 保持不变：

| 接口 | 用途 |
|---|---|
| `GET /v1/products` | 分类、品牌、属性查询最终商品 |
| `GET /v1/products/{id}` | 最新已发布商品版本 |
| `GET /v1/products/{id}/versions` | 历史版本及详情 |
| `GET /v1/listings` | 已发布来源在售商品 |
| `GET /v1/changes?cursor=…` | 按游标同步商品变化 |

内部管理 API 建议扩展：

- `POST /internal/factory/jobs`：上传 APK/需求并启动 Factory Job。
- `GET /internal/factory/jobs/{id}`：查询阶段、日志、证据及失败原因。
- `GET /internal/connectors/{id}/builds/{hash}`：构建包、SDK/依赖、差异和证据。
- `POST /internal/connectors/{id}/builds/{hash}/validate`：绑定指定哈希进行验证。
- `GET /internal/validations/{runId}`：查看强制/来源测试结果。
- `POST /internal/connectors/{id}/release`：经 AdmissionController 发起受控发布；人工或预授权自动发布使用同一准入逻辑。
- `POST /internal/connectors/{id}/rollback`：切换后续任务版本。
- `GET /internal/collection-runs/{id}`：查看任务、断点、采集量与数据质量。

### 8.3 安全边界

客户端禁用通用宿主命令执行；仅通过白名单分析工具访问任务目录与选定设备。Java 分析工具使用独立 JDK 17，不修改已有 Java 11 配置。Seep 固定至[既定已核验提交 ccddca5](https://github.com/angusdevgo/Seep-Reverse-Lab/tree/ccddca577fbaea6d4daa7ed4f2db0f688bad7f4a)并保留许可证声明；Pi RPC 经 OpenCode 调用云模型，支持流式对话、取消和会话恢复。管理员选择可用模型；原方案不设全局金额硬上限，但每个 Factory Job 必须具备可配置的修复轮次、工具调用与成本预算和人工终止能力。

服务端使用隔离容器运行 Agent 生成的候选代码和生产采集适配器，限制网络目的地、文件系统、系统调用和资源；凭据仅通过平台受控引用注入，不能进入仓库、Manifest 正文、测试样本或日志。签名能力必须合法可部署并能持续续期，不得以绕过权限或固化设备秘密满足验收。测试和生产都遵守来源授权、速率和适用使用条款。
