# SourceScheduler 来源调度插件

> 设计版本：v2.0 · 来源：《商品数据采集平台实施计划》 · 状态：待实施

[← 返回设计文档索引](./README.md)

---

## 4.2 SourceScheduler：Agent 生成的来源计划/展开器

**输入**：`RequirementSpec`、来源发现记录、已提交任务结果摘要、只读调度状态。  
**输出**：符合 Schema 的 `WorkflowSpec` / `TaskSpec[]`。  
**职责**：定义分类→品牌（若存在）→商品列表分页→详情→变体的依赖和动态 fan-out，选择来源分页类型，声明增量遍历/去重策略。

不允许 SourceScheduler：启动 Celery Worker、直接调 Redis、修改系统时间、生成未授权目标域名任务、直接提交断点。平台为它设定 CPU、运行时长、单轮发出的任务数量和 DAG 深度上限。
