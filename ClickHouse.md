# ClickHouse：项目介绍与生产落地资料存档

> 存档时间：2026-08-24
>
> 主题：ClickHouse 生产落地、实时分析、日志/可观测性、Kafka/Flink、Materialized View、Projection、MergeTree、性能优化、LLM/Agent Observability
>
> 说明：只保留本轮已确认、可访问或有明确来源的结果。DeepSeek 返回但无法实时核验的案例不作为事实证据写入主清单。

---

## 一、项目介绍

### ClickHouse/ClickHouse

- 项目名称：ClickHouse
- GitHub：https://github.com/ClickHouse/ClickHouse
- 项目定位：开源列式实时分析数据库（OLAP）
- License：Apache 2.0
- 创建时间：2016-06-02
- 最近更新时间：2026-08-24
- Star：约 49.4k（本轮检索期间观察到 49,407～49,409）
- 项目关联性：★★★★★
- 是否有源码：是
- 是否有部署步骤：是，官方文档、Docker、Linux/K8s 等路径完整
- 是否有真实案例：是，大量大型企业生产案例

### 核心能力

ClickHouse 主要解决“海量事件数据上的低延迟聚合查询”问题，典型场景包括：

- 日志、指标、Trace、OpenTelemetry 遥测
- 用户行为、埋点、漏斗、留存、AB Test
- 广告、风控、IoT、网络流量
- 实时数仓、BI、Dashboard
- AI Gateway / LLM / Agent 的请求、Token、Cost、Latency、Tool Call、Trace 分析

核心技术点：

1. **列式存储**：分析查询只读取需要的列，减少 I/O。
2. **MergeTree 家族**：ClickHouse 最核心的存储引擎体系。
3. **ORDER BY / Primary Key**：决定数据物理排序和稀疏索引效果，是生产优化第一优先级之一。
4. **Partition**：用于数据生命周期、分区裁剪和运维，不宜过细。
5. **Materialized View**：写入时增量计算，可用于实时预聚合、解析和派生表。
6. **Projection**：为同一张表提供另一种物理排序/聚合结构，查询优化器可自动选用。
7. **TTL / 冷热分层**：常与本地 SSD、对象存储组合降低长期存储成本。
8. **向量化执行 + 并行扫描**：支撑高吞吐分析。
9. **Replication / Sharding / Keeper**：用于高可用和分布式部署。
10. **Kafka / Flink / OpenTelemetry 集成**：构成大量真实实时分析链路。

### 适合与不适合的边界

适合：

- 大量 append-only / event 型数据
- 海量 `GROUP BY`、聚合、时间窗口分析
- 日志、Trace、Metric、Clickstream
- 实时分析和长时间历史分析

通常不适合直接替代 PostgreSQL/MySQL 承担：

- 高频单行 UPDATE/DELETE
- 强事务订单/余额系统
- 复杂 OLTP 工作负载

常见生产组合：

```text
PostgreSQL / MySQL
→ 用户、订单、配置、权限等 OLTP 真相数据

Redis
→ Cache / Rate Limit

Kafka / OpenTelemetry Collector
→ 流量缓冲、削峰、可重放

ClickHouse
→ 日志、事件、Token、Cost、Latency、Trace、实时分析
```

---

# 二、全网生产落地文章

## 1. Shopify：用 ClickHouse 构建全球统一可观测性平台

- 标题：Shopify observability at global scale
- 链接：https://clickhouse.com/blog/shopify-observability-at-global-scale
- 日期：2026-08-20
- 作者/来源：ClickHouse 官方博客 / Shopify 工程案例
- 关联性：★★★★★
- 真实案例：是
- 可验证实现细节：是
- 源码：文章以架构与生产数据为主，非完整业务源码
- 热度：官方最新大型生产案例，网页未公开统一互动数

### 规模

- 稳态约 5000 万 events/s
- BFCM 峰值约 1 亿 events/s
- 约 110 GB/s 未压缩 telemetry
- 大量 logs / metrics / traces / exceptions / profiles 统一进入平台

### 实现/架构

- Kafka 作为缓冲层
- 对 ClickHouse 进行同步大批量写入
- 多 tenant / 多团队隔离
- 自研 Kubernetes Operator 管理 ClickHouse
- 使用 Materialized View 做元数据、关联和派生计算
- 高频查询的 Map/动态字段逐步升格为 typed column
- 冷热数据与对象存储结合

### 可复用场景

- 大型统一可观测性平台
- 多租户日志/Trace/Metric 平台
- 高 cardinality telemetry
- AI 平台遥测数据仓库

---

## 2. Mercado Libre：将分布式追踪扩展到 4 亿 spans/分钟

- 标题：Mercado Libre observability on ClickHouse Cloud
- 链接：https://clickhouse.com/blog/mercado-libre-observability-on-clickhouse-cloud
- 日期：2026-08-04
- 来源：ClickHouse 官方博客 / Mercado Libre
- 关联性：★★★★★
- 真实案例：是
- 可验证实现：是
- 热度：官方大型客户案例

### 规模

- 从约 700 万 spans/min 扩展到 4 亿+ spans/min
- Traces 达到万亿级行数/大规模压缩存储

### 实现重点

- OpenTelemetry 采集
- ClickHouse Cloud 存储/查询 traces
- 高 cardinality attribute 查询优化
- Materialized View / 派生表减少查询时转换开销
- 业务属性（payment/user ID 等）作为可直接过滤的分析维度

### 可复用场景

- Trace 平台
- 微服务链路追踪
- 业务 ID 与 Trace 联合查询

---

## 3. Netflix：每天 5PB 日志的 ClickHouse 架构

- 标题：Petabyte-scale logging with ClickHouse at Netflix
- 链接：https://clickhouse.com/blog/netflix-petabyte-scale-logging
- 日期：2025-10-23
- 来源：ClickHouse 官方博客 / Netflix
- 关联性：★★★★★
- 真实案例：是
- 可验证实现：是
- 热度：大型公开工程案例

### 规模

- 约 5 PB logs/day
- 平均约 10.6M events/s
- 峰值约 12.5M events/s
- 查询约 500–1000 QPS

### 架构

```text
Service / Sidecar
→ Buffer
→ S3 / Kinesis
→ ClickHouse Hot Layer
→ Iceberg / Cold Layer
```

### 优化点

- 日志指纹/模式提取
- 原生协议序列化优化
- Tag Map 分片/拆分减少扫描
- 某类查询从约 3s 降到 700ms
- 热数据快速查询 + 冷数据湖长期保存

### 可复用场景

- PB 级日志平台
- Elasticsearch/Splunk 替代架构研究
- 热/冷日志分层

---

## 4. Canva：每秒数百万 Trace/Log 的可观测性系统

- 标题：Canva: faster search, lower costs with ClickHouse
- 链接：https://clickhouse.com/blog/canva-faster-search-lower-costs
- 日期：2025-12-01
- 来源：ClickHouse 官方博客 / Canva
- 关联性：★★★★★
- 真实案例：是
- 可验证实现：是

### 规模

- 约 3M spans/s
- 约 3M logs/s

### 架构/实践

- 5 shards × 3 replicas
- Kubernetes + ArgoCD
- OpenTelemetry Collector
- 在 Collector 层先进行 batch，每批约 20 万 spans
- ARM Graviton 等硬件/实例 benchmark

### 效果

- P90 trace 搜索约 30s → 2.5s
- 约 14× 压缩
- 成本降低约 70%

### 可复用场景

- OpenTelemetry → ClickHouse
- Logs + Traces 统一平台
- Collector batching 设计

---

## 5. Clever：同样成本扩大可搜索日志量

- 标题：Clever observability at scale
- 链接：https://clickhouse.com/blog/clever-observability-at-scale
- 日期：2026-06-30
- 来源：ClickHouse 官方博客 / Clever
- 关联性：★★★★★
- 真实案例：是
- 可验证实现：是

### 规模

- 约 400 个 Fargate service/worker
- 约 200 Lambda
- 约 150 TB/月未压缩日志

### 实现

- 原 Datadog/Athena → ClickHouse Cloud
- 60 天 retention
- 跨 region `remoteSecure`
- Terraform/KMS
- 高频字段列化
- skip index
- 排序键优化

### 可复用场景

- 中大型 SaaS 自建日志平台
- CloudWatch/Datadog 成本优化
- 多 Region 查询

---

## 6. LY / LINE Yahoo：每秒 700 万行写入

- 标题：LINE Yahoo ClickHouse production case
- 链接：https://clickhouse.com/blog/line-yahoo
- 日期：2026-02-09
- 来源：ClickHouse 官方博客 / LY Corporation
- 关联性：★★★★★
- 真实案例：是

### 规模

- Kafka 平台约 2.6 PB/day I/O
- ClickHouse 约 7M rows/s
- 4.1T+ rows
- 24 台服务器

### 数据链路

```text
Kafka API Interceptor
→ Protobuf
→ Internal Kafka
→ Log Ingestor Batch
→ ClickHouse
→ Redash / SQL Diagnosis
```

### 可复用场景

- Kafka 基础设施监控
- 超高吞吐实时 ingestion
- 平台级日志诊断

---

## 7. Critical Manufacturing：制造业实时分析

- 链接：https://clickhouse.com/blog/criticial-manufacturing
- 日期：2026-03-10
- 来源：ClickHouse 官方客户案例
- 关联性：★★★★☆
- 真实案例：是

### 实现

- Kafka → MergeTree
- Materialized View 做实时派生和聚合
- ReplacingMergeTree 处理变化数据
- 反规范化宽表
- TTL 管理历史数据

### 效果

- 原 SQL Server 分钟级查询下降到毫秒/秒级范围

### 可复用场景

- 工业 IoT
- Shop-floor 事件
- MES 实时分析

---

## 8. Respan：用 ClickHouse 做 LLM Observability

- 标题：Respan AI LLM observability with ClickHouse
- 链接：https://clickhouse.com/blog/respan-ai-llm-observability
- 日期：2026-04-06
- 来源：ClickHouse 官方博客 / Respan
- 关联性：★★★★★（与 AI Gateway/Agent 场景直接相关）
- 真实案例：是
- 可验证实现：是

### 规模

- 约 5000 万 events/day
- 约 3000 万 requests/day
- 接近 10 亿 events/月

### 迁移背景

- PostgreSQL 在约 50–100 writes/s 后逐渐成为瓶颈
- LLM tracing 数据天然属于 append-heavy、高 cardinality、聚合查询型数据

### Schema / 实现

把高频维度单独做 typed column：

- latency
- cost
- model
- provider
- routing
- token usage
- status

Materialized View 做 Trace/Session 聚合。

对 prompt/output 进行合理截断，避免单行过宽。

### 可复用场景

- AI Gateway
- LLM Proxy
- Agent Trace
- Token/Cost/Latency 分析
- 模型路由分析

---

## 9. ClickHouse 如何用 ClickHouse 观测自己：100PB+

- 标题：Scaling observability beyond 100PB
- 链接：https://clickhouse.com/blog/scaling-observability-beyond-100pb-wide-events-replacing-otel
- 日期：2025-06-19
- 来源：ClickHouse 官方自用案例
- 关联性：★★★★★
- 真实案例：是

### 规模

- 100 PB+ 未压缩遥测
- 接近 500 万亿行
- 部分日志流约 2M lines/s

### 实现

- System-table exporter
- OpenTelemetry
- 高 cardinality wide events
- HyperDX 做查询与分析

### 可复用场景

- ClickStack/HyperDX 架构
- 数据库自身可观测性
- 超大规模 wide event 模型

---

## 10. 滴滴：ES → ClickHouse 的 PB 级日志系统

- 标题：Didi migrates from Elasticsearch to ClickHouse
- 链接：https://clickhouse.com/blog/didi-migrates-from-elasticsearch-to-clickHouse-for-a-new-generation-log-storage-system
- 日期：2024-04-19
- 来源：滴滴 / ClickHouse 官方博客
- 关联性：★★★★★
- 真实案例：是

### 规模

- PB 级日志/天
- 400+ 物理节点
- 峰值写入 >40 GB/s
- 约 1500 万查询/天

### 架构

- ES/HDFS 旧架构迁移 ClickHouse
- MergeTree 日志表
- 热数据本地/高性能存储
- 冷数据 HDFS 等低成本存储

### 效果

- 硬件成本约下降 30%+
- 查询速度约提升 4×
- 多数 P99 查询 <1s

### 可复用场景

- ELK 替换/降本
- 海量日志平台
- 冷热分层

---

## 11. PostHog：1PB/月日志的数据丢失事故复盘

- 标题：PostHog US logs data loss post-mortem
- 链接：https://github.com/PostHog/post-mortems/blob/main/2026-02-20-posthog-us-logs-data-loss.md
- 日期：2026-02-20
- 来源：PostHog 官方 Post-mortem
- 关联性：★★★★★
- 真实事故：是
- 源码/配置证据：有公开仓库与复盘

### 规模

- 约 500 MB/s
- 约 1 PB/月未压缩日志

### 根因/教训

- S3 + experimental Zero Copy Replication 发生灾难性删除
- Kafka 仅保留约 3 天，帮助恢复近期数据
- Replica 只能解决高可用，不是备份
- Shared storage/zero-copy 也不是独立 backup

### 必须复用的安全原则

1. S3 Versioning / Object Lock
2. 独立 backup
3. Kafka retention 作为 replay layer
4. 删除/Mutation/后台任务监控
5. 灾难恢复演练

---

## 12. PostHog：Materialized Column 加速 JSON 查询

- 链接：https://github.com/PostHog/posthog.com/blob/master/contents/blog/clickhouse-materialized-columns.md
- 日期：2021-10-26
- 来源：PostHog
- 关联性：★★★★★
- 真实案例：是

### 优化数据

- JSON 提取查询：约 3433 ms → 980 ms
- 读取数据：约 79.17 GiB → 14.36 GiB

### 实现

- 从 slow query log 找高频 JSON/Property
- 自动/手工物化为 typed/materialized columns
- 减少 query-time JSON extraction

---

## 13. PostHog ClickHouse 生产架构

- 链接：https://github.com/PostHog/posthog.com/blob/master/contents/docs/how-posthog-works/clickhouse.md
- 来源：PostHog 官方架构文档
- 关联性：★★★★★
- 源码：是

### 核心链路

```text
Kafka
→ Kafka Table
→ Materialized View
→ Replicated / Distributed MergeTree
→ Product Analytics Query
```

Kafka 同时承担 buffer/replay 角色。

---

## 14. OneUptime：ClickHouse 日志分析架构

- 链接：https://oneuptime.com/blog/post/2026-03-31-clickhouse-log-analytics-pipeline/view
- 日期：2026-03-31
- 来源：OneUptime
- 关联性：★★★★☆
- 可验证实现：是

### 实现

- Vector / HTTP Insert
- Materialized View + SummingMergeTree 做分钟级错误统计
- token bloom filter 做日志关键词加速
- TTL / 冷热盘
- ingest lag 监控

---

## 15. Picnic：实时业务分析 Materialized View

- 链接：https://clickhouse.com/blog/picnic-real-time-analytics
- 来源：ClickHouse 官方客户案例
- 关联性：★★★★☆
- 真实案例：是

### 实现

- Kafka events → Materialized View → 解析表/聚合表
- Refreshable MV 做复杂转换
- Grafana + dbt
- Kafka retention 支持 replay/backfill

---

## 16. Tesla Comet：PB / Quadrillion-scale Metrics

- 链接：https://clickhouse.com/blog/how-tesla-built-quadrillion-scale-observability-platform-on-clickhouse
- 来源：ClickHouse 官方博客 / Tesla
- 关联性：★★★★★
- 真实案例：是

### 实现方向

- OpenTelemetry Collector
- Kafka-compatible queue
- ClickHouse metrics store
- PromQL → ClickHouse 查询层

### 可复用场景

- Prometheus 长期存储
- 超高 cardinality Metrics
- 大型 AI/GPU 基础设施监控

---

## 17. LaunchDarkly / Highlight：ClickHouse 日志服务

- 链接：https://launchdarkly.com/blog/how-we-built-logging-service-with-clickhouse/
- 日期：2025-09-15
- 来源：LaunchDarkly / Highlight
- 关联性：★★★★☆
- 真实案例：是

### 可复用场景

- SaaS 日志服务
- ClickHouse schema / query / ingestion 设计

---

# 三、GitHub 开源项目

> GitHub 来源包含 gh-reader Elasticsearch `multi_match` 搜索以及 `site:github.com` 字面补搜。以下保留与 ClickHouse 实际生产、运维、采集、日志、AI/Agent 场景关联度较高的项目。

## 1. PostHog/posthog

- 链接：https://github.com/PostHog/posthog
- Star：38,787
- 最近更新：2026-08-24
- 技术栈：Python / Django / Kafka / ClickHouse / React 等
- 落地能力：产品分析、事件、日志、Session Replay、Feature Flag、AI observability
- 实践场景：用户行为事件、漏斗、留存、实时分析
- 关联性：★★★★★
- 源码：是
- 真实生产：是

## 2. langfuse/langfuse

- 链接：https://github.com/langfuse/langfuse
- Star：33,599
- 最近更新：2026-08-24
- 技术栈：TypeScript / PostgreSQL / ClickHouse / Redis / Object Storage / OpenTelemetry
- 落地能力：LLM Trace、Eval、Prompt、Dataset、Cost/Latency Metrics
- 实践场景：LLM Gateway、Agent、RAG Observability
- 关联性：★★★★★
- 源码：是
- 部署：Docker/K8s 等

## 3. SigNoz/signoz

- 链接：https://github.com/SigNoz/signoz
- Star：31,882
- 最近更新：2026-08-19
- 技术栈：OpenTelemetry / ClickHouse / Go / TypeScript
- 落地能力：Logs / Metrics / Traces / APM / Infra Monitoring
- 关联性：★★★★★
- 源码：是
- 真实案例：大量生产部署

## 4. hyperdxio/hyperdx

- 链接：https://github.com/hyperdxio/hyperdx
- Star：9,856
- 最近更新：2026-08-24
- 技术栈：ClickHouse + OpenTelemetry + Web UI
- 落地能力：Logs、Metrics、Traces、Errors、Session Replay
- 实践场景：ClickStack 核心 UI/分析层
- 关联性：★★★★★
- 源码：是
- 部署：是

## 5. OneUptime/oneuptime

- 链接：https://github.com/OneUptime/oneuptime
- Star：7,506
- 最近更新：2026-08-24
- 落地能力：完整 observability / incident / monitoring
- ClickHouse 关联：日志分析平台架构
- 关联性：★★★★☆

## 6. ClickHouse/clickhouse-go

- 链接：https://github.com/ClickHouse/clickhouse-go
- Star：3,326
- 最近更新：2026-08-11
- 技术栈：Go
- 落地能力：官方 Go ClickHouse Driver，高吞吐 batch insert/query
- 关联性：★★★★★

## 7. Altinity/clickhouse-operator

- 链接：https://github.com/Altinity/clickhouse-operator
- Star：2,556
- 最近更新：2026-08-20
- 技术栈：Kubernetes Operator
- 落地能力：创建、扩缩、升级、管理 ClickHouse Cluster
- 实践场景：K8s 生产 ClickHouse
- 关联性：★★★★★

## 8. clickvisual/clickvisual

- 链接：https://github.com/clickvisual/clickvisual
- Star：1,640
- 最近更新：2026-07-27
- 技术栈：ClickHouse / Log Analytics / Web UI
- 落地能力：轻量日志分析、数据可视化
- 关联性：★★★★★

## 9. Altinity/clickhouse-backup

- 链接：https://github.com/Altinity/clickhouse-backup
- Star：1,636
- 最近更新：2026-08-19
- 落地能力：ClickHouse backup/restore，支持对象存储、增量备份
- 实践场景：生产灾备
- 关联性：★★★★★

## 10. ClickHouse/clickhouse-java

- 链接：https://github.com/ClickHouse/clickhouse-java
- Star：1,614
- 最近更新：2026-08-23
- 技术栈：Java/JDBC
- 落地能力：Java 服务访问 ClickHouse
- 关联性：★★★★★

## 11. mymarilyn/clickhouse-driver

- 链接：https://github.com/mymarilyn/clickhouse-driver
- Star：1,301
- 最近更新：2026-07-17
- 技术栈：Python / Native Protocol
- 落地能力：Python 批量写入和查询
- 关联性：★★★★☆

## 12. ClickHouse/ClickBench

- 链接：https://github.com/ClickHouse/ClickBench
- Star：1,088
- 最近更新：2026-08-24
- 落地能力：OLAP benchmark
- 实践场景：表结构/数据库/查询性能对比
- 关联性：★★★★★

## 13. ClickHouse/mcp-clickhouse

- 链接：https://github.com/ClickHouse/mcp-clickhouse
- Star：857
- 最近更新：2026-08-21
- 技术栈：MCP / ClickHouse / AI Assistant
- 落地能力：Claude/Codex 等 Agent 通过 MCP 安全查询 ClickHouse
- 实践场景：AI Data Analyst / Data Agent
- 关联性：★★★★★

## 14. mr-karan/logchef

- 链接：https://github.com/mr-karan/logchef
- Star：853
- 最近更新：2026-08-19
- 落地能力：ClickHouse 单二进制日志查询与可视化
- 实践场景：轻量自托管日志平台
- 关联性：★★★★★

## 15. Altinity/clickhouse-grafana

- 链接：https://github.com/Altinity/clickhouse-grafana
- Star：774
- 最近更新：2026-08-10
- 落地能力：Grafana ClickHouse datasource
- 关联性：★★★★☆

## 16. caioricciuti/ch-ui

- 链接：https://github.com/caioricciuti/ch-ui
- Star：705
- 最近更新：2026-08-23
- 落地能力：ClickHouse Web UI、SQL 查询、实例指标查看
- 关联性：★★★★☆

## 17. PostHog/HouseWatch

- 链接：https://github.com/PostHog/HouseWatch
- Star：626
- 最近更新：2026-08-05
- 落地能力：ClickHouse Cluster 监控与管理
- 实践场景：PostHog 自身生产 ClickHouse 运维
- 关联性：★★★★★

## 18. apecloud/ape-dts

- 链接：https://github.com/apecloud/ape-dts
- Star：594
- 最近更新：2026-08-21
- 技术栈：Rust / CDC / Replication
- 落地能力：MySQL/PostgreSQL/Redis/Mongo/Kafka 与 ClickHouse 数据复制和迁移
- 关联性：★★★★★

## 19. ClickHouse/clickhouse-rs

- 链接：https://github.com/ClickHouse/clickhouse-rs
- Star：551
- 最近更新：2026-08-21
- 技术栈：Rust
- 落地能力：官方 Rust typed client
- 关联性：★★★★☆

## 20. ClickHouse/agent-skills

- 链接：https://github.com/ClickHouse/agent-skills
- Star：511
- 最近更新：2026-08-06
- 落地能力：ClickHouse 官方 Agent Skills
- 实践场景：Claude Code / Codex / AI Data Engineering
- 关联性：★★★★★

## 21. glassflow/clickhouse-etl

- 链接：https://github.com/glassflow/clickhouse-etl
- Star：493
- 最近更新：2026-06-29
- 落地能力：ClickHouse ingestion + transformation pipeline
- 实践场景：Kafka/stream ingestion、清洗、转换
- 关联性：★★★★★

## 22. ClickHouse/adsb.exposed

- 链接：https://github.com/ClickHouse/adsb.exposed
- Star：448
- 最近更新：2026-07-26
- 落地能力：ADS-B 实时航空数据分析与可视化
- 关联性：★★★★☆
- 源码：是

## 23. ClickHouse/ch-go

- 链接：https://github.com/ClickHouse/ch-go
- Star：426
- 最近更新：2026-08-10
- 技术栈：Go
- 落地能力：低层 Native Protocol client
- 关联性：★★★★☆

## 24. getsentry/snuba

- 链接：https://github.com/getsentry/snuba
- Star：409
- 最近更新：2026-08-22
- 技术栈：Python / ClickHouse / Kafka
- 落地能力：Sentry 事件查询与分析层
- 实践场景：Error/Event Analytics
- 关联性：★★★★★
- 真实生产：是

## 25. ClickHouse/clickhouse-operator

- 链接：https://github.com/ClickHouse/clickhouse-operator
- Star：281
- 最近更新：2026-08-20
- 落地能力：ClickHouse 官方 Kubernetes Operator
- 关联性：★★★★★

## 26. grafana/clickhouse-datasource

- 链接：https://github.com/grafana/clickhouse-datasource
- Star：219
- 最近更新：2026-08-20
- 落地能力：Grafana 官方 ClickHouse datasource
- 关联性：★★★★☆

## 27. ClickHouse/clickhouse-kafka-connect

- 链接：https://github.com/ClickHouse/clickhouse-kafka-connect
- Star：208
- 最近更新：2026-08-05
- 技术栈：Kafka Connect
- 落地能力：官方 Kafka → ClickHouse Connector
- 实践场景：Kafka 流式摄入
- 关联性：★★★★★

## 28. bryzgaloff/airflow-clickhouse-plugin

- 链接：https://github.com/bryzgaloff/airflow-clickhouse-plugin
- Star：184
- 最近更新：2026-07-27
- 技术栈：Airflow / Python / ClickHouse
- 落地能力：Airflow ETL 调度 ClickHouse
- 关联性：★★★★☆

## 29. clklog/clklog

- 链接：https://github.com/clklog/clklog
- Star：179
- 最近更新：2026-08-03
- 落地能力：私有化用户行为分析平台
- 功能：事件、漏斗、留存、路径、画像、分群
- 关联性：★★★★★

## 30. mintance/nginx-clickhouse

- 链接：https://github.com/mintance/nginx-clickhouse
- Star：159
- 最近更新：2026-07-25
- 落地能力：Nginx 日志解析并写入 ClickHouse
- 关联性：★★★★☆

## 31. ClickHouse/clickpy

- 链接：https://github.com/ClickHouse/clickpy
- Star：153
- 最近更新：2026-08-07
- 落地能力：PyPI Analytics powered by ClickHouse
- 关联性：★★★★☆

## 32. doneyli/claude-code-langfuse-template

- 链接：https://github.com/doneyli/claude-code-langfuse-template
- Star：104
- 最近更新：2026-07-29
- 落地能力：自托管 Langfuse 采集 Claude Code session observability
- 关联性：★★★★★（AI Coding Agent）

## 33. nano-rs/nano

- 链接：https://github.com/nano-rs/nano
- Star：69
- 最近更新：2026-08-07
- 技术栈：Rust / ClickHouse / PostgreSQL
- 落地能力：SIEM，ClickHouse 存日志，Postgres 存状态
- 关联性：★★★★☆

## 34. Wave-RF/WaveHouse

- 链接：https://github.com/Wave-RF/WaveHouse
- Star：31
- 最近更新：2026-08-21
- 技术栈：ClickHouse API Gateway
- 落地能力：Async ingestion、per-table batching、JWT/JWKS、query cache、SSE、dedup
- 关联性：★★★★★

## 35. joshleecreates/awesome-clickhouse-observability

- 链接：https://github.com/joshleecreates/awesome-clickhouse-observability
- Star：19
- 最近更新：2026-08-17
- 落地能力：整理 ClickHouse Observability 文章、工具、视频
- 关联性：★★★★☆

## 36. doneyli/langfuse-llm-certification-finance

- 链接：https://github.com/doneyli/langfuse-llm-certification-finance
- Star：15
- 最近更新：2026-07-23
- 落地能力：Langfuse + ClickHouse 的金融 LLM 模型认证/分析流水线
- 关联性：★★★★★

## 37. clklog/clklog-processing

- 链接：https://github.com/clklog/clklog-processing
- Star：10
- 最近更新：2026-08-22
- 落地能力：直接消费 Kafka，清洗后写 ClickHouse
- 关联性：★★★★★

## 38. tarique-iqbal/nyc-taxi

- 链接：https://github.com/tarique-iqbal/nyc-taxi
- Star：9
- 最近更新：2026-08-22
- 技术栈：Kafka / DDD validation / ClickHouse / Grafana
- 落地能力：Production-style streaming ETL
- 关联性：★★★★☆

## 39. doneyli/clickhouse-llm-observability

- 链接：https://github.com/doneyli/clickhouse-llm-observability
- Star：8
- 最近更新：2026-08-12
- 技术栈：LibreChat / Langfuse / ClickHouse
- 落地能力：LLM Observability Demo
- 关联性：★★★★★

## 40. chenghaonan777/realtime-mall

- 链接：https://github.com/chenghaonan777/realtime-mall
- Star：4
- 最近更新：2026-08-09
- 技术栈：Flink + Kafka + ClickHouse
- 落地能力：电商实时数仓
- 关联性：★★★★☆

## 41. PalenaAI/langfuse-operator

- 链接：https://github.com/PalenaAI/langfuse-operator
- Star：3
- 最近更新：2026-07-20
- 技术栈：Kubernetes Operator
- 落地能力：一套 CR 管 Langfuse Web / Worker / PostgreSQL / ClickHouse / Redis / Blob Storage
- 关联性：★★★★★

## 42. ClickStack / HyperDX 部署资料

- GitHub/文档：https://github.com/ClickHouse/clickhouse-docs/blob/main/docs/use-cases/observability/clickstack/ingesting-data/collector.md
- 获取通路：`site:github.com` Web 补搜
- 落地能力：ClickHouse + HyperDX + OpenTelemetry Collector 完整可观测性栈
- 关联性：★★★★★
- 部署/复现：是

---

# 四、知乎落地文章

## 1. ClickHouse 到底有多神？

- 标题：ClickHouse 到底有多神？
- URL：https://www.zhihu.com/question/505958148/answer/1987457848125440066
- 作者：花宝宝
- 时间：2026-03-13
- 热度：搜索接口未返回赞同/评论数
- 关联性：★★★★★
- 是否真实实践：作者称使用一年，有具体性能结果；属于个人生产经验，无法像企业官方案例一样外部审计
- 是否有 SQL：是

### 实践要点

1. `ORDER BY (event_time, user_id)` → `(user_id, event_time)`，按 user_id 查询提升约 10×+
2. 不要单行 INSERT
3. Kafka → Flink → ClickHouse
4. Flink 积攒约 10 秒、约 50 万条后批量写入
5. Materialized View + SummingMergeTree 预聚合

---

## 2. clickhouse相关

- URL：https://zhuanlan.zhihu.com/p/655046641
- 作者：Xman
- 时间：2026-08-13
- 关联性：★★★★☆
- 实现细节：有

### 重点

- Projection 有自己的物理排序结构
- 优化器自动选择 Projection
- Projection 会增加写入放大和磁盘占用
- 可通过 `system.projections` 监控
- 写入前排序/批量写入可减少 Merge 压力

---

## 3. ClickHouse 集成 Kafka 表引擎详细解析

- URL：https://zhuanlan.zhihu.com/p/677551513
- 作者：张飞的猪
- 时间：2024-07-27
- 关联性：★★★★☆
- 可复现：是

### 重点

Kafka Engine 的虚拟列：

- `_topic`
- `_key`
- `_offset`
- `_timestamp`
- `_timestamp_ms`
- `_partition`

适合作为 Kafka Engine + MV 入门与排错参考。

---

## 4. 告别 ELK：基于 ClickHouse 的日志方案

- URL：https://zhuanlan.zhihu.com/p/1117015950
- 作者：云观秋毫
- 时间：2024-10-15
- 关联性：★★★★★
- 可验证实现：有 SQL/schema/架构

### 架构思想

- 不让 ClickHouse Kafka Engine 直接承担不可控洪峰
- 使用 Vector 控制 ingestion/batch
- Vector → Null Engine → Materialized View → 真正 MergeTree 日志表
- 原始日志保留 `_raw_log_`
- `tokenbf_v1` skip index 用于近似全文检索
- TTL 控制 retention

### 可复用价值

对中小规模自托管日志平台非常实用，比“Kafka Engine 直接拉取一切”更容易控制内存和背压。

---

## 5. Salla：MySQL 到 ClickHouse CDC 流水线问题与优化

- URL：https://zhuanlan.zhihu.com/p/1921559771942745450
- 作者：Timeplus
- 时间：2025-06-26
- 关联性：★★★★★
- 真实客户场景：是，属于厂商案例文章，需要注意厂商立场

### 生产问题

- 数千关系表，每张表对应 Kafka Topic
- 每张表需要 Kafka consumer + Materialized View
- ClickHouse 与 Kafka 建立大量连接
- CPU/Memory 随表数量上升
- 单消息/过小写入影响吞吐
- ReplacingMergeTree + `FINAL` 在 JOIN 时性能差
- 业务查询可能 JOIN 5–30 张事实/维度表

### 可复用价值

明确展示了“ClickHouse 原生 Kafka Engine + 海量表”架构的扩展边界。

---

## 6. ClickHouse 在自助行为分析场景的实践应用

- URL：https://zhuanlan.zhihu.com/p/590257631
- 作者：转转技术团队
- 时间：2022-12-08
- 关联性：★★★★★
- 真实案例：是，企业技术团队

### 架构

```text
MySQL
→ Flink CDC
→ Kafka

App / Log
→ Flume
→ Kafka + HDFS

Kafka / 清洗
→ ClickHouse 宽表
→ 行为分析 / AB Test / 用户画像
```

### ClickHouse 实践

- MV + 聚合表 + 明细表
- AB Test 从 T+1 改成实时
- Flink 自定义 Sink，可配置 flush batch/time
- 内存不足时使用近似函数、external group by/sort 参数

---

## 7. 网络安全中的 ClickHouse 流式增量聚合

- URL：https://zhuanlan.zhihu.com/p/1921135399050404176
- 作者：ji ant
- 时间：2025-06-25
- 关联性：★★★★☆

### 数据流

```text
Kafka
→ Kafka Engine
→ raw MV
→ MergeTree raw table
→ 5min MV
→ SummingMergeTree
```

### 生命周期

- raw：7–14 天
- 5min aggregate：90 天或更久

### 场景

- 网络安全流量
- 实时监控大盘
- 长期趋势 + 原始下钻

---

## 8. ClickHouse 九层存储架构 / 故障式教程

- URL：https://zhuanlan.zhihu.com/p/2057372441618408024
- 作者：Lambert-zw
- 时间：2026-07-09
- 关联性：★★★☆☆
- 说明：有大量具体 SQL/架构，但叙事型教学属性明显，不能等同真实企业事故证据

覆盖：Projection、Materialized View、缓存、写入放大等。

---

## 9. ClickHouse 数据库故事技术教程

- URL：https://zhuanlan.zhihu.com/p/2062301213530629668
- 作者：Lambert-zw
- 时间：2026-07-19
- 关联性：★★★☆☆
- 说明：体系化教学，不按真实客户案例评级

覆盖：

- MergeTree Part
- WAL
- Compaction
- Skip Index
- Projection
- Keeper
- 灾备
- CBO/RBO
- Pipeline Execution

---

## 10. ClickHouse 集群 ZooKeeper 平滑搬迁

- URL：https://zhuanlan.zhihu.com/p/360844107
- 来源：知乎 Web 字面补搜
- 关联性：★★★★★
- 实战价值：高

### 重点

- ZooKeeper 配置平滑切换
- 动态 reload ZooKeeper config 的工程问题
- 涉及 ClickHouse patch/PR
- 排查 `optimize_trivial_count_query` 等行为导致 ZooKeeper 访问

这种真实运维迁移比普通三节点搭建教程更值得参考。

---

# 五、站点文章（CSDN / 掘金 / YouTube 聚合搜索）

> 本轮站点文章搜索执行了 3 页，约 190 条原始结果。第三页开始明显出现大量重复、泛教程和 SEO 内容，因此以下只保存有企业实践、部署复现、性能测试、迁移/故障等强信号的条目。

## 掘金企业/生产实践

### 1. ClickHouse 在酷家乐日志监控系统中的实践
- URL：https://juejin.cn/post/7251786922615111740
- 来源：掘金
- 关联性：★★★★★
- 类型：企业日志监控实践
- 真实案例：是/企业技术分享属性

### 2. ClickHouse 冷热分离存储在得物的实践
- URL：https://juejin.cn/post/7158651378808651807
- 来源：掘金
- 关联性：★★★★★
- 类型：冷热分层、成本优化

### 3. SkyWalking 使用 ClickHouse 存储实战
- URL：https://juejin.cn/post/7079748123148419079
- 关联性：★★★★★
- 类型：APM / Trace Storage

### 4. ClickHouse 在自助行为分析场景的实践应用
- URL：https://juejin.cn/post/7174673746957959227
- 关联性：★★★★★
- 类型：行为分析、AB Test、用户画像

### 5. 微信 ClickHouse 实时数仓最佳实践
- URL：https://juejin.cn/post/7034697994318381063
- 关联性：★★★★★
- 类型：实时数仓

### 6. ClickHouse 在酷家乐指标系统中的实践
- URL：https://juejin.cn/post/7309692103056179235
- 关联性：★★★★★
- 类型：指标系统

### 7. ClickHouse 投影实战
- URL：https://juejin.cn/post/7255512131658678332
- 关联性：★★★★☆
- 类型：Projection

### 8. ClickHouse 主键索引最佳实践
- URL：https://juejin.cn/post/7291836033719779362
- 关联性：★★★★★
- 类型：ORDER BY / 稀疏索引

### 9. ClickHouse 数据表迁移实战：remote 方式｜京东云
- URL：https://juejin.cn/post/7248242792283226168
- 关联性：★★★★★
- 类型：数据迁移/生产运维

### 10. Shopee ClickHouse 冷热数据分离架构与实践
- URL：https://juejin.cn/post/7018020316228091940
- 关联性：★★★★★
- 类型：电商/冷热存储

### 11. ClickHouse 实战：安装部署
- URL：https://juejin.cn/post/6933039239739703309
- 关联性：★★★☆☆
- 类型：部署复现

### 12. ClickHouse 实战：集群配置
- URL：https://juejin.cn/post/6938379148368805925
- 关联性：★★★★☆
- 类型：Cluster 部署

### 13. ClickHouse 与 Greenplum 集成实践
- URL：https://juejin.cn/post/7330839699807305738
- 关联性：★★★★☆
- 类型：异构 OLAP

### 14. ClickHouse + Amazon S3 三种存算分离方式
- URL：https://juejin.cn/post/7280007832714870796
- 关联性：★★★★★
- 类型：S3 / Object Storage

### 15. 记一次 ClickHouse 性能测试
- URL：https://juejin.cn/post/7131778389865660452
- 关联性：★★★★☆
- 类型：Benchmark

### 16. ClickHouse 与 Kafka 数据同步
- URL：https://juejin.cn/post/7072361777727537182
- 关联性：★★★★☆
- 类型：Kafka ingestion

### 17. 使用 Vector 将 Nginx 日志实时发送到 ClickHouse
- URL：https://juejin.cn/post/7281213804699500601
- 关联性：★★★★★
- 类型：轻量日志采集
- 可复现：强

### 18. ClickHouse 物化视图踩坑记录
- URL：https://juejin.cn/post/6903508511637340173
- 关联性：★★★★★
- 类型：Materialized View pitfall

### 19. ClickHouse + Grafana 可观测性解决方案：日志篇
- URL：https://juejin.cn/post/7400253623790157859
- 关联性：★★★★★
- 类型：Observability / 日志 Schema / Grafana

### 20. ClickHouse 中处理实时更新
- URL：https://juejin.cn/post/7039978768022110238
- 关联性：★★★★☆
- 类型：Update/Mutation

### 21. ClickHouse 中处理 Update/Delete/Upsert
- URL：https://juejin.cn/post/7282691710072897573
- 关联性：★★★★☆
- 类型：Mutation / ReplacingMergeTree

---

## CSDN 可复现/运维文章

### 1. ClickHouse Keeper：Server 无法连接 Keeper 的故障记录

- URL：https://blog.csdn.net/qq_51950769/article/details/148818001
- 作者：爱吃萝卜的猪
- 日期：2025-06-20
- 热度：约 1231 views / 29 likes
- 关联性：★★★★★
- 实战：是

### 问题

1 shard 2 replicas + Keeper，ClickHouse Server 无法连接 Keeper，DDLWorker 阻塞。

### 根因

最终定位到 `listen_host` 配置位置不正确。

---

### 2. 3 种企业级 ClickHouse Backup 方案

- URL：https://blog.csdn.net/gitblog_00108/article/details/154518663
- 作者：葛瀚纲Deirdre
- 日期：2025-11-06
- 热度：635 views / 10 likes
- 关联性：★★★★☆

覆盖：

- 单节点备份
- Kubernetes
- 高可用集群
- S3/Azure/GCS
- 增量备份
- 灾备演练

---

### 3. ClickHouse + Superset Docker OLAP 平台

- URL：https://blog.csdn.net/sat99/article/details/155621107
- 日期：2026-02-05
- 热度：708 views / 30 likes
- 关联性：★★★★☆

覆盖：

- Docker Compose
- Superset 初始化
- MergeTree 表设计
- Materialized View / Projection
- Query Cache
- RBAC / SSL

---

### 4. LangChain4j + ClickHouse 向量存储 RAG

- URL：https://blog.csdn.net/boling_cavalry/article/details/157033786
- 关联性：★★★★★（AI/RAG）
- 类型：ClickHouse Vector / RAG 实战
- 可复现：有代码路径

---

# 六、Bilibili

> Bilibili 搜索接口没有返回发布日期，因此不猜测时间；保留作者、播放、时长和可见互动。

## 1. VIVO ClickHouse 百亿级日志场景及优化分享

- URL：https://www.bilibili.com/video/BV11eoVBdE6f
- 作者：ClickHouseInc
- 时长：23:12
- 播放：109
- 赞：8
- 投币：5
- 收藏：10
- 关联性：★★★★★
- 场景：百亿级日志生产优化
- 真实案例：是

## 2. Kafka + Flink + ClickHouse 实时生产消费数据演示

- URL：https://www.bilibili.com/video/BV1no4y1W7W9
- 作者：菜籽oil
- 时长：2:10
- 播放：约 5,342
- 关联性：★★★★☆
- 场景：最小实时链路演示

## 3. 快速插入大量数据 ClickHouse 参数调优

- URL：https://www.bilibili.com/video/BV1xw411X7M6
- 作者：小工蚁创始人
- 时长：14:23
- 播放：约 1,280
- 关联性：★★★★☆
- 场景：Batch Insert / Write Tuning

## 4. ClickHouse 亿级数据 JOIN 调优案例

- URL：https://www.bilibili.com/video/BV14TDQYoELF
- 作者：程序员阿奇
- 时长：9:13
- 播放：约 977
- 关联性：★★★★☆
- 场景：亿级 JOIN

## 5. ClickHouse + Langfuse Agent 可观测

- URL：https://www.bilibili.com/video/BV1q3oVBrE8V
- 作者：ClickHouseInc
- 时长：37:51
- 播放：约 1,158
- 关联性：★★★★★
- 场景：AI Agent / LLM Observability

## 6. ClickHouse 处理亿级淘宝行为数据

- URL：https://www.bilibili.com/video/BV1E1421r7ht
- 作者：古道东风破88
- 时长：5:24
- 播放：约 17,063
- 关联性：★★★★☆
- 场景：用户行为分析

## 7. Apache DolphinScheduler + ClickHouse 入库最佳实践

- URL：https://www.bilibili.com/video/BV1nFy1Y2EhQ
- 作者：DolphinScheduler
- 时长：38:18
- 播放：约 531
- 关联性：★★★★☆
- 场景：调度、批量入库

## 8. ClickHouse 完整教程

- URL：https://www.bilibili.com/video/BV1xg411w7AP
- 作者：京东架构师诸葛
- 时长：294:33
- 播放：约 122,471
- 关联性：★★★☆☆
- 说明：热度高、体系完整，但落地证据弱于 VIVO 等生产案例

---

# 七、Grok 检索结果

Grok 本轮主要用于发现大型 ClickHouse 生产案例。关键结果均通过其它网页来源二次核验后才进入主清单。

## 已二次核验

- Shopify Observability：✅
- Mercado Libre Tracing：✅
- Netflix Logging：✅
- Canva Observability：✅
- Didi Logging：✅
- Respan LLM Observability：✅

## 找到线索但本轮未单独展开全部细节

- Anthropic 使用 ClickHouse 做 AI/模型基础设施 observability
- Replo 100B+ events 优化
- Buildkite Test Analytics
- D.E. Shaw 高 cardinality observability
- ByteDance / ByteHouse 生产演进

处理原则：Grok 的发现只作为“候选入口”；规模、日期、架构数字必须能在原文中确认后才视为已验证事实。

---

# 八、DeepSeek 检索结果

DeepSeek 本轮明确返回：当前调用环境无法实时联网验证。

因此：

- DeepSeek 返回的公司案例、URL、日期、吞吐数字只作为候选搜索词。
- 无其它网页/GitHub 交叉验证的内容不进入主证据清单。
- 本轮可直接作为证据的 DeepSeek 独立结果：0 条。

它提示的有价值主题包括：

- ClickHouse + Kafka/Flink
- ORDER BY / Primary Key 设计
- Partition 设计
- Materialized View / Projection
- S3 对象存储
- Mutation/Update/Delete
- LLM Observability
- Kubernetes/Operator
- Keeper 故障

---

# 九、生产落地共识

从 Shopify、Netflix、Canva、Clever、ClickHouse 自身、PostHog、Respan、滴滴等案例，可以归纳出以下可复用原则。

## 1. 写入：批量是第一原则

不要：

```text
1 event
→ 1 INSERT
→ 1 Part
```

推荐：

```text
Application / Agent / OTel
→ Collector / Kafka / Vector
→ Batch
→ ClickHouse
```

小批频繁写会导致大量 Parts、Merge 压力和查询抖动。

## 2. ORDER BY 是最重要的 Schema 决策之一

原则：

- 高频等值过滤维度放前面
- 时间范围通常放在后续位置
- 不要机械地把 timestamp 放第一位

示例：

```sql
ORDER BY (tenant_id, service_name, created_at)
```

通常比：

```sql
ORDER BY (created_at, tenant_id, service_name)
```

更适合多租户单租户检索。

## 3. 动态 JSON/Map 可以保留，但热字段要列化

常见迁移：

```text
properties JSON / Map
        ↓
Slow Query Log 找高频属性
        ↓
Typed / Materialized Column
```

高频字段列化可减少 JSON extraction 和扫描数据量。

## 4. Materialized View 用于稳定、重复、高频的聚合

常见：

```text
raw_events
→ Materialized View
→ minute_agg / daily_agg
```

适合：

- QPS/错误率
- Token/Cost
- PV/UV
- Trace summary
- 5min 网络流量

## 5. Projection 用于同一份数据的第二种访问路径

适合：

- 一张表有多个稳定查询模式
- 主 ORDER BY 无法覆盖第二类查询

成本：

- 写入放大
- 存储放大
- 维护复杂度

## 6. Kafka 不只是传输，是恢复层

生产案例反复证明：

```text
Kafka
= Buffer
+ Backpressure
+ Replay
+ Backfill
+ Failure Recovery
```

Retention 的长度直接决定事故后能重放多远。

## 7. Replica ≠ Backup

PostHog 的事故是最强证据：

```text
Replication → Availability
Backup      → Disaster Recovery
```

必须分别建设。

## 8. TTL + Hot/Warm/Cold

典型：

```text
最近 7–14 天 raw
→ SSD / 高频查询

90 天 aggregate
→ 低成本存储

超长期
→ S3 / Iceberg / HDFS / Object Storage
```

## 9. OpenTelemetry + ClickHouse 已成为成熟组合

典型链路：

```text
Application / AI Agent
→ OpenTelemetry SDK
→ OTel Collector
→ Kafka（可选）
→ ClickHouse
→ HyperDX / Grafana / Langfuse
```

## 10. AI Gateway / Agent 是天然的 ClickHouse 场景

建议事件 Schema 至少包含：

```text
request_id
trace_id
session_id
user_id
model
provider
input_tokens
output_tokens
cached_tokens
latency_ms
time_to_first_token_ms
cost
cache_hit
fallback_count
tool_name
tool_latency
status
error_code
created_at
```

### 推荐架构

```text
AI Gateway / Agent
        ↓
OpenTelemetry
        ↓
Collector / Kafka
        ↓
ClickHouse
        ↓
┌──────────────┬───────────────┐
│ HyperDX      │ Langfuse      │
│ Logs/Trace   │ LLM Trace/Eval│
└──────────────┴───────────────┘
```

其中：

- ClickHouse：存 request / trace / token / cost / latency / tool event
- HyperDX：工程日志、Trace、错误定位
- Langfuse：LLM Trace、Eval、Prompt、Dataset
- Grafana：基础设施和聚合指标
- MCP ClickHouse：允许 Codex/Claude 直接做受控数据分析

---

# 十、优先参考清单

如果只挑最有参考价值的一批：

## 日志 / Observability

1. Netflix 5PB/day Logging
2. Shopify 100M events/s Observability
3. Canva 3M spans/s + 3M logs/s
4. ClickHouse 自己 100PB+
5. 滴滴 ES → ClickHouse
6. PostHog 1PB/月数据丢失复盘
7. HyperDX / ClickStack
8. SigNoz

## AI / LLM / Agent

1. Respan LLM Observability
2. Langfuse
3. `doneyli/clickhouse-llm-observability`
4. `doneyli/claude-code-langfuse-template`
5. `ClickHouse/mcp-clickhouse`
6. Bilibili：ClickHouse + Langfuse Agent 可观测

## 数据摄入

1. ClickHouse Kafka Connector
2. GlassFlow ClickHouse ETL
3. Kafka + Flink + ClickHouse
4. Vector → ClickHouse
5. Ape DTS

## 运维 / 生产安全

1. Altinity ClickHouse Operator
2. ClickHouse 官方 Operator
3. ClickHouse Backup
4. PostHog HouseWatch
5. PostHog 数据丢失事故复盘
6. ZooKeeper/Keeper 平滑迁移和故障案例

---

## 存档结论

ClickHouse 最成熟的生产路径并不是“直接把所有请求打到一张 MergeTree 表”。大型系统普遍演进成：

```text
高吞吐事件
→ Kafka / OTel Collector / Vector 做缓冲和批量
→ ClickHouse Raw Table
→ typed hot columns
→ Materialized View / Projection
→ TTL / Object Storage
→ HyperDX / Grafana / BI / Langfuse
```

最重要的工程经验是：

- 批量写，不要产生大量小 Part；
- 优先把 `ORDER BY` 设计对；
- 热字段从 JSON/Map 中列化；
- 只为稳定热点查询做 MV/Projection；
- Kafka retention 是恢复能力的一部分；
- Replica 绝不能替代 Backup；
- 日志、Trace、LLM Token/Cost 数据是 ClickHouse 最匹配的工作负载之一。
