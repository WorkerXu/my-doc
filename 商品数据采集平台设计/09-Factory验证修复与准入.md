# Factory 验证、修复与自动准入

> 设计版本：v2.0 · 来源：《商品数据采集平台实施计划》 · 状态：待实施

[← 返回设计文档索引](./README.md)

---

## 5. 生成、验证、修复、自动准入发布

### 5.1 Factory 状态机

```text
REQUIREMENT_READY
  -> ANALYZING
  -> EVIDENCE_READY
  -> GENERATING
  -> BUILT
  -> CONTRACT_TESTING
  -> COMPONENT_TESTING
  -> INTEGRATION_TESTING
  -> INDEPENDENCE_TESTING
  -> ADMISSION_PENDING
       -> PUBLISHED -> ACTIVE
       -> MANUAL_REVIEW -> PUBLISHED / BLOCKED

任一步骤失败：FAILED -> REPAIRING -> BUILT -> 重新走必要测试
证据不足、环境不可移植、预算超限：BLOCKED（保留证据与候选包）
```

每个 Factory Job 保存 `requirement_version`、`apk_sha256`、`connector_hash`、SDK/依赖哈希、验证报告、失败证据引用、生成轮次、预算与审计事件。Pi 单轮默认最多 60 分钟、100 次工具调用，可暂停与恢复；修复必须设最大轮次/时间/费用策略（具体阈值按管理配置）。

### 5.2 独立 Validation Pipeline

| 阶段 | 被验证内容 | 失败结果 |
|---|---|---|
| Contract/Build | Manifest/Schema、SDK 版本、允许域名、凭据引用、依赖锁及源码安全策略 | 退回 Agent 修复；禁止加载为生产包 |
| Scheduler tests | 分类展开、DAG 合法性、动态分页、终止条件、任务幂等、限流 | 输出重现输入和实际任务图 |
| Executor tests | 响应解析、字段映射、认证续期、重试分类、错误处理 | 输出脱敏网络/解析证据 |
| SourceValidator tests | 正确数据应通过、变异和故障数据必须被拒绝 | 标记测试器失效，不能准入 |
| Integration tests | 真实受控试采、列表→详情、分类范围、身份键、数据库提交与重启恢复 | 输出失败商品/页/范围 |
| Independence tests | 禁止连接本地客户端、ADB、APK 及分析工作目录；在独立 Linux 沙箱运行 | 标记 BLOCKED |
| Admission | 所有强制项、构建哈希不变、发布模式、授权与覆盖证据、容量上限 | 手工审核或 BLOCKED |

**完整性声明**：如果来源提供可信商品总数，则报告 `expected/listed/unique/detail_success/detail_failed`；若来源无可信总数，则报告已发现入口数、游标终止证据、未知覆盖范围，不能以“所有测试 PASS”替代“已采齐全部商品”的证明。正式全量采集完成后，再对实际数据做批量对账。

**修复规则**：Agent 只修改候选包，不能修改 PlatformValidator；代码、配置、依赖或构建镜像变化必须生成新 `connector_hash`，重做受影响测试并在准入前执行完整强制回归；所有结论绑定同一最终产物哈希。

### 5.3 预授权自动发布

`release_policy.mode`：

- `MANUAL_APPROVAL`：默认；平台完成所有测试后展示包、证据、差异和报告，管理员确认特定哈希才发布。
- `PREAUTHORIZED_AUTO`：管理员事先定义允许来源、域名、凭据范围、采集类型、并发/资源上限、通过条件及适用版本范围。仅当固定 AdmissionController 判定通过才自动将不可变包注册、激活并启动周期调度。

以下条件禁止直接自动发布：新增未授权域名、签名依赖真机、凭据不可续期、严重覆盖不确定或关键字段未验证、修改平台固定安全契约、依赖存在阻断性安全问题、未通过干净服务端独立运行测试。默认进入 `MANUAL_REVIEW/BLOCKED`，不可由 AI 将错误降级为警告绕过。

生产运行中的任务固定 `connector_hash`；新版本通过后对**后续任务**切流，可灰度验证和回滚。旧版本及历史运行证据可审计，未完成任务保持原版本或者遵循平台显式的迁移策略。
