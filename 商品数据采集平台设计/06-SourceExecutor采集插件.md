# SourceExecutor 来源采集插件

> 设计版本：v2.0 · 来源：《商品数据采集平台实施计划》 · 状态：待实施

[← 返回设计文档索引](./README.md)

---

## 4.3 SourceExecutor：Agent 生成的来源适配器

**输入**：一个不可变 `TaskSpec` 和平台提供的受限 `ExecutionContext`。  
**输出**：`ExecutionResult`，其中包含原始 `RecordEnvelope`、页游标、下一层发现信息和证据引用。

Executor 只能通过 SDK 请求获准的 HTTP(S) 域名，读取显式授权的凭据句柄；复杂签名逻辑被封装在单独适配器中并在服务端验证。其自身无正式数据库写入能力，不得执行任意 shell/访问宿主文件/扩展未经批准的网络目的地。
