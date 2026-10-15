# Connector 构建与产物规范

> 设计版本：v2.0 · 来源：《商品数据采集平台实施计划》 · 状态：待实施

[← 返回设计文档索引](./README.md)

---

## 3.3 生成三组件和完整产物

工厂将 **SDK 文档、Schema、参考 Connector、RequirementSpec、DiscoveryEvidence、测试规范** 一并提供给 Agent。Agent 生成一个完整包；允许多轮修复，但每次修复产生新候选哈希。

```text
connectors/leixiaoan/1.0.0/
├── manifest.yaml              # sdk_version、hash、capabilities、权限和发布参数
├── scheduler/
│   ├── workflow.yaml          # 定时、DAG、展开/分页、结束条件
│   └── planner.py             # 可选：复杂动态调度插件
├── executor/
│   ├── categories.py
│   ├── product_list.py
│   ├── product_detail.py
│   └── auth_signer.py         # 可选：必要的合法服务端认证适配器
├── validator/
│   ├── source_rules.yaml
│   └── test_source.py
├── mapping.yaml               # 来源字段到原始记录标准 Envelope 的映射
├── requirements.lock          # 固定 SDK 与运行依赖
├── tests/fixtures/            # 脱敏样本、异常样本和预期结果
└── evidence/index.json        # 可追溯到每个步骤和字段的接口证据
```

目录只是协议示例；无复杂调度或签名时优先生成声明式配置，避免无意义的可执行代码。`manifest.yaml` 中的能力声明不等于已获平台认证，真实能力标记由 AdmissionController 根据验证结果填入 Registry。
