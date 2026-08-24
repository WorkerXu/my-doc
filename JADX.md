# JADX：Android 反编译、jadx-core 二次开发与 AI / MCP 自动化逆向落地资料

> 调研与存档时间：2026-08-23  
> 主项目：<https://github.com/skylot/jadx>  
> 说明：本文保存本次已确认的 JADX 项目介绍、实战落地文章、GitHub 二开项目及知乎、掘金、Bilibili、X、DeepSeek、Exa 等来源的检索结果。结果是当前各检索通路可访问、可返回范围内的最大覆盖，不声称穷尽整个互联网。纯概念介绍、营销软文、无关转载和同名噪声未按同等权重收录。

---

## 一、项目介绍

### 1. 基本信息

- 项目：`skylot/jadx`
- GitHub：<https://github.com/skylot/jadx>
- Stars：49,783（本次检索时）
- 创建时间：2013-03-18
- 最近更新时间：2026-07-25
- License：Apache-2.0
- 定位：Android DEX / APK 到 Java 源码的反编译器，同时提供 CLI、GUI、`jadx-core` Java 库、插件 API、脚本和程序分析能力。

JADX 可以把 Android 应用中的 Dalvik / DEX 字节码和资源转换为更适合阅读、检索和二次分析的形式：

```text
APK / AAB / DEX / AAR / JAR / CLASS / SMALI / XAPK / APKM
                         ↓
                       JADX
                         ↓
        Java-like Source / Manifest / Resources
                         ↓
         Search / Xref / Call Graph / Deobfuscation
```

### 2. 主要能力

- APK、DEX、AAR、AAB、ZIP、JAR、CLASS、SMALI 等输入格式反编译。
- 解码 `AndroidManifest.xml` 和 `resources.arsc` 资源。
- 自带反混淆 / rename 能力，并支持 Tiny、Tiny v2、Enigma、ProGuard、SRG、TSRG、JOBF 等映射格式。
- Kotlin Metadata / SourceDebugExtension 支持，可改善类名、参数名、字段名、Companion Object、Data Class、Getter 等还原效果。
- GUI 支持语法高亮、跳转定义、Find Usage、全文搜索、Smali Debugger。
- CLI 可输出 Java 或 JSON，可导出 Gradle 工程。
- 支持 `--call-graph dot|json` 生成调用图。
- 提供插件系统，可以注册自定义 Pass、Options、GUI 扩展、输入格式和解密 / 反混淆逻辑。
- `jadx-core` 可以作为 Java Library 嵌入其他后端服务，实现无 GUI 的批量 APK 分析。

### 3. 典型落地架构

JADX 现在最有价值的方向已经不只是“人工打开 APK 看源码”，而是作为自动化 Android 程序分析底座：

```text
APK
 │
 ▼
jadx-core Headless
 │
 ├── Manifest / Resources
 ├── Classes / Methods
 ├── Strings
 ├── Xrefs
 └── Call Graph
        │
        ▼
   Analysis Index
        │
 ┌──────┼─────────┐
 ▼      ▼         ▼
规则扫描  LLM Agent  JNI Detector
 │        │         │
 │        │         ▼
 │        │      Ghidra Headless
 │        │
 └────┬───┘
      ▼
Evidence / Findings DB
      │
      ├── API / Auth Flow
      ├── Privacy & Permission
      ├── WebView / Crypto
      ├── Obfuscation Rename
      └── Security Findings
              │
              ▼
          REPORT.md
```

### 4. 适合落地的产品方向

1. **APK 自动静态分析平台**：上传 APK 后自动提取 Manifest、权限、组件、URL、API、SDK、字符串、调用关系和资源。
2. **AI APK 安全审计**：JADX 提供结构化源码 / Xref / 调用图，LLM 负责高层语义和证据归纳。
3. **AI 逆向助手**：将 JADX GUI / jadx-core 通过 MCP 暴露给 Claude、Codex、Cursor 等 Agent。
4. **隐私合规扫描**：识别权限、设备信息采集、广告 / 埋点 SDK、WebView、明文存储、网络请求等。
5. **反混淆 / 字符串解密插件**：利用 JadxPlugin / Pass API 在反编译流水线中自动恢复可读语义。
6. **Java + Native 联合分析**：检测 JNI 边界，把 `.so` 继续送到 Ghidra / IDA Headless 分析。
7. **Call Graph / 数据流分析服务**：直接消费 JADX 调用图或基于 bytecode INVOKE 建自己的 Caller / Callee 索引。
8. **自动生成交付报告**：把分析状态和证据写入 SQLite / JSON / Markdown，形成可复查的 `REPORT.md`。

综合参考价值：**⭐⭐⭐⭐⭐**。

---

## 二、本次检索准备

### 1. 主题

`JADX`

### 2. 中文关键词 / 扩展词

- JADX
- JADX 实战
- JADX 落地
- JADX 二次开发
- JADX 插件
- jadx-core
- Android 反编译
- APK 静态分析
- Android 逆向实战
- JADX MCP
- JADX AI
- JADX 反混淆
- APK 自动审计
- APK 安全分析
- 调用图
- JNI / SO 联合分析
- 源码 / 部署 / 复现 / 架构 / 项目实践

### 3. 英文关键词 / 扩展词

- JADX production implementation
- jadx-core headless
- JADX plugin tutorial
- JadxPlugin
- JADX MCP
- JADX-AI-MCP
- jadx-mcp-server
- Android APK static analysis
- AI APK security audit
- Android reverse engineering case study
- deobfuscation plugin
- call graph
- headless APK analysis
- LLM Android reverse engineering

### 4. 来源覆盖情况

| 来源 | 本次命中 / 状态 | 说明 |
|---|---|---|
| 全网落地文章 | 约 10 个强相关实战 / 官方实现文档 | 可用，优先保留安装、源码、架构、案例证据 |
| GitHub 项目 | ES 原始候选量很大，人工过滤后约 25 个直接相关项目 | 已使用 gh-reader ES，并用通用 Web 的 `site:github.com` 字面搜索补充去重 |
| 知乎 | 两轮约 20 个原始命中，去重后约 17 个；下文保留 8 个高落地结果 | 站内搜索 + Web 补搜已执行 |
| 掘金 | 通用 Web 约 11 个原始命中，5 个较强落地结果 | 指定 Tavily 通路本轮不可用；未安装 / 改配置，使用 `site:juejin.cn` 通用搜索补充 |
| Bilibili | 两轮各返回数十条，去重与去泛营销后保留 6 条直接相关 | 可用；部分结果未返回发布时间，不猜日期 |
| X | 6 条有明确 JADX 实践证据的帖子 | 仅将 x.com 结果作为 X 证据 |
| DeepSeek | 0 条实时可验证结果 | 当前 DeepSeek 连接明确不支持实时 Web 搜索，因此不把记忆知识伪装成实时结果 |
| Exa | 首轮得到约 15 个高相关实现结果 | 后续限定域名补搜失败，但首轮结果有效 |

---

## 三、最值得优先看的落地参考

| 项目 / 内容 | 价值 | 原因 |
|---|---|---|
| `zinja-coder/jadx-ai-mcp` | ⭐⭐⭐⭐⭐ | 成熟的 JADX GUI Plugin + MCP 方案，暴露源码、Smali、资源、Xref、Rename、Debugger 等能力 |
| `1013503897/jadx-headless-mcp` | ⭐⭐⭐⭐⭐ | 直接用 `jadx-core`，完全不依赖 GUI，适合后端服务化 |
| `jygzyc/decx` | ⭐⭐⭐⭐⭐ | HTTP + MCP + CLI + Skills + SQLite BlackBoard + Report / PoC，最像完整产品架构 |
| `samudoria/revoid` | ⭐⭐⭐⭐⭐ | Headless JADX + 内存索引 + Caller/Callee + Ghidra Headless，面向 LLM 的接口设计很有参考价值 |
| `xjoker/delamain` | ⭐⭐⭐⭐⭐ | Docker、鉴权、文件上传、缓存、大 APK、调用图、数据流、Frida、安全扫描，产品化思路完整 |
| `cys7885/jexray` | ⭐⭐⭐⭐½ | 在 JADX 内把 Java/JNI 与 Ghidra Native 伪代码衔接起来 |
| `yj94/JADX-NO-MCP` | ⭐⭐⭐⭐½ | 不依赖 MCP，直接实现壳检测、Call Graph、JNI 报告和安全发现 |
| `Fausto-404/ai-mobile-reverse-skills` | ⭐⭐⭐⭐½ | 把 JADX、Burp/Yakit、IDA/Ghidra MCP 编排成完整移动安全分析流程 |
| 官方 Jadx Plugins Guide | ⭐⭐⭐⭐⭐ | 自己写 JADX 插件的第一手 API 文档 |
| 官方 Use jadx as a library | ⭐⭐⭐⭐⭐ | 做 Headless APK 自动分析后端最直接的官方入口 |

---

# 四、全网落地文章 / 官方实现文档

## 1. JADX-MCP 逆向 APK 实战教程

- 标题：JADX-MCP 逆向 APK 实战教程
- 链接：<https://developer.cloud.tencent.com/article/2701762>
- 日期：2026-07-02
- 来源：腾讯云开发者社区
- 可见热度：本次抓取约 420 浏览
- 关联性：**高**
- 可验证实现：**是**
- 源码：引用 JADX / MCP 开源项目
- 部署 / 复现：**有**
- 真实案例：**有**

摘要与落地内容：

- 使用 WorkBuddy + JADX + MCP 进行 APK 分析。
- 从 Manifest、入口组件开始建立应用结构。
- 使用关键词搜索代码、硬编码 Key、WebView、API。
- 使用 Xref 追踪引用关系。
- 分析资源文件并做重命名。
- 讨论 MCP 调用中的 Token 优化。

这篇的价值在于已经从“打开 JADX”升级为“Agent 调用 JADX 工具完成结构化分析”。

## 2. AI 赋能 Jadx 静态分析 APK 代码实战指南

- 标题：AI 赋能 Jadx 静态分析 APK 代码实战指南
- 链接：<https://cn-sec.com/archives/5346256.html>
- 日期：2026-07-23
- 来源：CN-SEC
- 可见热度：本次抓取约 37 浏览
- 关联性：**高**
- 可验证实现：**是**
- 部署 / 复现：**有**
- 真实案例：**有**

摘要：

- Trae / Claude Code 接 JADX MCP。
- 实际做类搜索、协议逻辑分析和接口提取。
- 将静态分析结论继续转成 Frida 脚本 / 动态验证思路。
- 体现“JADX 静态层 → Agent 语义层 → 动态验证”的闭环。

## 3. Jadx plugins guide

- 标题：Jadx plugins guide
- 链接：<https://github.com/skylot/jadx/wiki/Jadx-plugins-guide>
- 日期：Wiki 2026-07-12 仍有编辑
- 作者 / 来源：`skylot/jadx` 官方 Wiki
- 热度：官方文档，无统一浏览数据
- 关联性：**极高**
- 可验证实现：**是**
- 源码 / API：**有**
- 部署 / 复现：**有**

核心实现：

- 引入 `io.github.skylot:jadx-core`。
- 插件主类实现 `jadx.api.plugins.JadxPlugin`。
- 通过 `JadxPluginContext` 注册功能。
- `addPass`：把自定义逻辑插入反编译流水线。
- `registerOptions`：向 CLI / GUI 暴露插件参数。
- `getGuiContext`：扩展 JADX GUI。
- JAR 可以通过 `jadx plugins --install-jar` 安装。
- 支持通过 GitHub Release 发布插件。

适合直接作为自研：字符串解密、反混淆、敏感 API 检查、资源处理、GUI AI 按钮等扩展的起点。

## 4. Use jadx as a library

- 标题：Use jadx as a library
- 链接：<https://github.com/skylot/jadx/wiki/Use-jadx-as-a-library>
- 日期：Wiki 2024-12-23 有编辑记录
- 来源：`skylot/jadx` 官方 Wiki
- 关联性：**极高**
- 可验证实现：**是**
- 源码 / API：**有**
- 部署 / 复现：**有**

核心价值：

直接嵌入 `jadx-core`，不需要启动 `jadx-gui`。这一路线最适合构建：

```text
Upload APK
→ JadxDecompiler
→ Build Index
→ Search / Source / Xref / Call Graph
→ REST / MCP API
→ Agent / SAST / Report
```

## 5. Jadx scripts guide

- 标题：Jadx scripts guide
- 链接：<https://github.com/skylot/jadx/wiki/Jadx-scripts-guide>
- 日期：Wiki 2026-03-31 有编辑记录
- 来源：`skylot/jadx` 官方 Wiki
- 关联性：**高**
- 可验证实现：**是**
- 部署 / 复现：**有**

摘要：

- 使用 Kotlin Script 自动化 JADX。
- 可以访问反编译实例、修改参数、做 rename、监听 `afterLoad` 等阶段。
- 对“小型自动化 / 快速 PoC”比开发完整 Plugin 更轻。

## 6. JADX-AI-MCP 官方文档

- 标题：JADX-AI-MCP
- 链接：<https://jadx-ai-mcp.readthedocs.io/en/latest/>
- 日期：2026 年仍活跃
- 来源：JADX-AI-MCP 官方 ReadTheDocs
- 关联性：**极高**
- 可验证实现：**是**
- 部署 / 复现：**有**
- 源码：<https://github.com/zinja-coder/jadx-ai-mcp>

架构：

```text
LLM Client
   ↓ MCP
jadx-mcp-server (Python / FastMCP)
   ↓ HTTP
JADX-AI-MCP Plugin (Java)
   ↓
JADX GUI / Core
```

它把当前打开 APK 的源码、Manifest、资源、方法、字段、Smali、Xref、重命名和调试信息暴露给 LLM。

## 7. JADX-AI-MCP Installation

- 标题：JADX-AI-MCP Installation
- 链接：<https://jadx-ai-mcp.readthedocs.io/en/latest/installation/>
- 来源：官方文档
- 关联性：**极高**
- 可验证实现：**是**
- 部署 / 复现：**完整**

包含：

- Java / JADX / Python / uv 环境要求。
- JADX Plugin 安装。
- Python MCP Server 启动。
- Claude / Cursor 等 MCP Client 连接。
- 端口配置。
- 非 localhost 绑定时的无鉴权明文 HTTP 风险提示。

## 8. JADX-AI-MCP User Guide

- 标题：JADX-AI-MCP User Guide
- 链接：<https://jadx-ai-mcp.readthedocs.io/en/stable/user-guide/>
- 来源：官方文档
- 关联性：**极高**
- 可验证实现：**是**
- 真实工作流：**有**

包含：

- 从 Package Tree / Main Activity 开始做 APK Triage。
- 搜索 WebView、Crypto、Auth 等高风险模块。
- Manifest / Resource 分析。
- Xref 追踪。
- 反混淆与 Rename。
- 大型 APK 使用分页和选择性分析减少 Context 消耗。

## 9. JADX-AI-MCP Contributing

- 标题：Contributing to JADX-AI-MCP
- 链接：<https://jadx-ai-mcp.readthedocs.io/en/latest/contributing/>
- 来源：官方文档
- 关联性：**高**
- 可验证实现：**是**
- 二次开发：**有**

适合研究 Java Plugin 与 Python Server 两侧如何增加新 Tool、测试与构建。

## 10. JADX + MCP: I let the AI read the APK so I don’t have to

- 标题：JADX + MCP: I let the AI read the APK so I don’t have to
- 链接：<https://infosecwriteups.com/jadx-mcp-i-let-the-ai-read-the-apk-so-i-dont-have-to-548d1e8210e6>
- 日期：2026-04-08
- 作者：Xcheater
- 来源：InfoSec Write-ups
- 关联性：**极高**
- 可验证实现：**是**
- 部署 / 复现：**有**
- 真实案例：**有**

具体流程：

1. 在 JADX-GUI 安装 `jadx-ai-mcp` 插件。
2. 启动插件 HTTP Server。
3. 下载 / 启动 `jadx-mcp-server`。
4. 配置 Cursor 的 MCP JSON。
5. 先让 Agent 读取 Manifest 验证链路。
6. 实战使用 Manifest / Components → Attack Surface → Source-to-Sink → Finding。
7. 推荐按 Authentication / Obfuscation / Business Logic 等小范围任务分析，而不是一次扫全 APK。

---

# 五、GitHub 项目

> GitHub 项目由 gh-reader Elasticsearch 搜索，并使用通用 Web `site:github.com` 补充；下列是去重后与 JADX / jadx-core / AI APK 分析直接相关的主要项目。

## 1. skylot/jadx

- 链接：<https://github.com/skylot/jadx>
- Stars：49,783
- 最近更新：2026-07-25
- 主要技术栈：Java / Gradle / Swing / DEX 反编译器
- 落地能力：CLI、GUI、`jadx-core`、Plugin API、脚本、Call Graph、反混淆、资源解析
- 关联性：**核心项目**
- 源码：**是**
- 部署 / 构建：**是**，JDK 17+ 可 `./gradlew dist`
- 场景：所有后续 JADX 二开 / MCP / Headless 项目的底座

## 2. zinja-coder/jadx-ai-mcp

- 链接：<https://github.com/zinja-coder/jadx-ai-mcp>
- Stars：2,609
- 创建：2025-04-07
- 最近更新：2026-08-06
- 技术栈：Java JADX Plugin + MCP 集成
- 落地能力：当前类、选中文本、源码、方法、字段、Smali、Manifest、资源、Xref、Rename、Debugger 等
- 关联性：**极高**
- 源码：**是**
- 部署：**完整**
- 场景：Claude / Cursor 等直接操作 JADX GUI 完成 AI APK 静态分析

## 3. zinja-coder/jadx-mcp-server

- 链接：<https://github.com/zinja-coder/jadx-mcp-server>
- Stars：738
- 创建：2025-04-08
- 最近更新：2026-08-06
- 技术栈：Python / FastMCP / HTTP
- 落地能力：作为 LLM 与 JADX Plugin 的 MCP 中间层
- 关联性：**极高**
- 源码：**是**
- 部署：`uv run jadx_mcp_server.py` 等方式可复现
- 注意：非 localhost 绑定可能是未认证的明文 HTTP，应做网络隔离 / 防火墙 / SSH Tunnel

## 4. 1013503897/jadx-headless-mcp

- 链接：<https://github.com/1013503897/jadx-headless-mcp>
- Stars：4
- 创建：2026-05-12
- 最近更新：2026-08-18
- 技术栈：直接基于 `jadx-core` 的 Headless MCP
- 落地能力：无需 JADX GUI 的 APK 静态分析服务
- 关联性：**极高**
- 源码：**是**
- 场景：Server / Docker / 批处理 / Agent 后端

这是本轮对“自己做 APK 分析后端”最直接的参考之一。

## 5. Qtty/jadx-mcp-server

- 链接：<https://github.com/Qtty/jadx-mcp-server>
- Stars：28
- 创建：2025-08-13
- 最近更新：2026-08-20
- 技术栈：Pure Java / MCP / JADX
- 架构：MCP Layer (`JadxToolService`) → API Layer (`JadxApkAnalyzerAPI`) → Core Layer (`JadxAnalyzerCore`) → CLI Layer
- 落地能力：Load APK、Class / Method / Field、Exported Components、Manifest、Resource、Smali
- 关联性：**极高**
- 源码：**是**
- 部署 / 复现：**有**

适合参考如何完全用 Java 封装 jadx-core，不引入 Python Bridge。

## 6. xjoker/delamain

- 链接：<https://github.com/xjoker/delamain>
- Stars：2
- 创建：2026-07-21
- 最近更新：2026-07-24
- 技术栈：Java JADX Backend + Python Gateway + Docker
- 落地能力：
  - Headless JADX
  - Class / Method / Smali
  - Indexed Search
  - Xref
  - Caller / Callee
  - Call Graph Export
  - Data Flow
  - Manifest / Resource
  - Frida Hook / Trace 生成
  - Attack Surface / Security Scan
  - Rename / Mapping
  - Session / Annotation
  - Out-of-band APK Upload
- 关联性：**极高**
- 源码：**是**
- Docker：**有**
- 鉴权：MCP Gateway 支持 Bearer Token

产品化思路很强，特别值得参考大 APK 文件传输、缓存、鉴权和 Agent Context 优化。

## 7. samudoria/revoid

- 链接：<https://github.com/samudoria/revoid>
- Stars：本次 Exa 结果未返回可靠数值
- 技术栈：Kotlin / Gradle / MCP Kotlin SDK / `jadx-core` / Ghidra Headless
- 落地能力：
  - Headless APK / DEX 分析
  - In-memory Class / Method / String Index
  - 基于 bytecode INVOKE 构建 Caller / Callee
  - Call Graph
  - 分页接口，减少 LLM Context 膨胀
  - Native `.so` 自动进入 Ghidra Headless
  - Native Function / String / Import / Export 查询
- 关联性：**极高**
- 源码：**是**
- 真实工程证据：**有**

这是 Java 层 + Native 层统一给 Agent 提供程序分析数据库接口的优秀参考。

## 8. jygzyc/decx

- 链接：<https://github.com/jygzyc/decx>
- Stars：71
- 创建：2025-06-13
- 最近更新：2026-08-13
- 技术栈：JADX + HTTP API + MCP + CLI + Skills + SQLite
- 落地能力：
  - APK / JAR 处理
  - 全局代码搜索
  - Exported Components / Deeplinks
  - App / Framework Vulnerability Hunt
  - SQLite BlackBoard 保存事实、意图、事件、链接和调用链
  - `decx-report` 生成 Markdown / HTML 报告
  - `decx-poc` 把最终 Finding 转成可构建 PoC
- 关联性：**极高**
- 源码：**是**
- 部署 / 复现：**有**

这是本轮最接近“Agent-native APK 安全分析产品”的项目。

## 9. yj94/JADX-NO-MCP

- 链接：<https://github.com/yj94/JADX-NO-MCP>
- Stars：12
- 创建：2026-07-03
- 最近更新：2026-08-08
- 技术栈：JADX GUI / CLI 扩展
- 落地能力：早期壳检测、Java Call Graph、JNI-focused 报告、安全 Finding、双语 README
- 关联性：**高**
- 源码：**是**
- MCP：**否**

适合参考“不使用 MCP，直接增强静态分析器本体”的设计。

## 10. cys7885/jexray

- 链接：<https://github.com/cys7885/jexray>
- Stars：4
- 创建：2026-08-05
- 最近更新：2026-08-06
- 技术栈：JADX Plugin + Ghidra
- 落地能力：识别 JNI Native Methods，在 JADX 内查看 Ghidra 伪代码 / 反汇编
- 关联性：**高**
- 源码：**是**
- 真实实现：**有**

非常适合参考 Java → JNI → Native `.so` 的上下文衔接。

## 11. Arsylk/jadx-string-decrypt

- 链接：<https://github.com/Arsylk/jadx-string-decrypt>
- Stars：2
- 创建：2026-05-30
- 最近更新：2026-07-25
- 技术栈：JADX Plugin / Decompilation Pass
- 落地能力：编译期常量反混淆、可解析块密码字符串解密
- 关联性：**高**
- 源码：**是**

这是研究 Jadx Pass API、自动解密和反混淆最直接的样例之一。

## 12. nitram84/jadx-apkspy-plugin

- 链接：<https://github.com/nitram84/jadx-apkspy-plugin>
- Stars：9
- 创建：2024-10-10
- 最近更新：2026-08-19
- 技术栈：JADX Plugin
- 落地能力：编辑反编译 Java 源并重新编译
- 关联性：**高**
- 源码：**是**

## 13. jadx-decompiler/jadx-script-kotlin

- 链接：<https://github.com/jadx-decompiler/jadx-script-kotlin>
- Stars：5
- 最近更新：2026-07-01
- 技术栈：Kotlin Script / JADX
- 落地能力：JADX Scripting
- 关联性：**高**
- 源码：**是**

## 14. nexusjellyfishhum/jadx-plus-elite

- 链接：<https://github.com/nexusjellyfishhum/jadx-plus-elite>
- Stars：56
- 创建：2026-07-30
- 最近更新：2026-08-17
- 定位：JADX 增强版 / 二开
- 关联性：**中高**
- 源码：**是**

## 15. Fausto-404/ai-mobile-reverse-skills

- 链接：<https://github.com/Fausto-404/ai-mobile-reverse-skills>
- Stars：142
- 创建：2026-04-20
- 最近更新：2026-08-14
- 技术栈：Agent Skills + JADX MCP + Burp/Yakit MCP + IDA/Ghidra MCP
- 落地能力：六阶段总控 Skill，统一调度 APK 静态侦察、流量与代码对齐、SO/JNI 深度分析、加密 / 漏洞分析、验证设计和报告交付
- 关联性：**高**
- 源码：**是**

## 16. zhaoxuya520/reverse-skill

- 链接：<https://github.com/zhaoxuya520/reverse-skill>
- Stars：27,272
- 创建：2026-05-13
- 最近更新：2026-08-22
- 技术栈：Agent Skill Router / 工具链检测
- 落地能力：检测本机是否有 JADX、apktool、Frida、radare2、Ghidra 等，再按任务选择工具；支持 Claude Code、Kiro、Cursor、Cline 等客户端
- 关联性：**高（JADX 是核心工具之一）**
- 源码：**是**

## 17. maosasagawa/blackbox-re-agent

- 链接：<https://github.com/maosasagawa/blackbox-re-agent>
- Stars：35
- 创建：2026-06-09
- 最近更新：2026-08-12
- 技术栈：Agent + radare2 / Ghidra / angr / JADX
- 落地能力：会话式黑盒逆向 Agent，自动准备分析工具链
- 关联性：**中高**
- 源码：**是**

## 18. Paresh-Maheshwari/morphe-ai

- 链接：<https://github.com/Paresh-Maheshwari/morphe-ai>
- Stars：123
- 创建：2026-07-12
- 最近更新：2026-07-19
- 定位：AI-powered Android APK patching workspace
- 落地能力：多 Agent 分析、目标定位、Patch 编写与部署
- 关联性：**中高**
- 源码：**是**

## 19. roomkangali/droid-llm-hunter

- 链接：<https://github.com/roomkangali/droid-llm-hunter>
- Stars：194
- 创建：2026-01-01
- 最近更新：2026-08-08
- 定位：LLM 驱动 Android App 漏洞扫描
- 落地能力：将静态分析结果送给 LLM 判断漏洞
- 关联性：**中高**
- 源码：**是**

## 20. labcif/ADBExtractorAndAnalyzer

- 链接：<https://github.com/labcif/ADBExtractorAndAnalyzer>
- Stars：10
- 创建：2024-05-04
- 最近更新：2026-07-19
- 技术栈：ADB + ALEAPP + JADX + MobSF + RootAVD
- 落地能力：Android 取证、APK 提取和多工具联合分析
- 关联性：**中高**
- 源码：**是**

## 21. CrisörHacker/mcp-pentest

- 链接：<https://github.com/CrisorHacker/mcp-pentest>
- Stars：3
- 创建：2026-05-17
- 最近更新：2026-08-04
- 技术栈：MCP + ADB + Apktool + JADX + Frida + Objection + Burp
- 落地能力：把 Android Pentest 常用工具暴露给 Claude / Agent
- 关联性：**高**
- 源码：**是**

## 22. Facetomyself/reverse_ENV

- 链接：<https://github.com/Facetomyself/reverse_ENV>
- Stars：22
- 创建：2026-06-26
- 最近更新：2026-07-31
- 技术栈：IDA Pro / JADX / apktool / radare2 / Frida MCP
- 落地能力：逆向工程工作环境和 Skill 仓库
- 关联性：**高**
- 源码：**是**

## 23. LING71671/open-reverselab

- 链接：<https://github.com/LING71671/open-reverselab>
- Stars：1,023
- 创建：2026-06-17
- 最近更新：2026-08-10
- 定位：Agent-native reverse-engineering lab
- 落地能力：197 篇知识库、MCP 工具、CTF / APK / PE 自动化 Workflow
- 关联性：**中高**
- 源码：**是**

## 24. mobilehackinglab/jadx-mcp-plugin

- 链接：<https://github.com/mobilehackinglab/jadx-mcp-plugin>
- Stars：本次结果未返回可靠值
- 技术栈：Java JADX Plugin + Embedded HTTP + Python FastMCP Adapter
- 落地能力：List Classes、Search Class、Get Source、Method / Field、Method Code 等
- 关联性：**极高**
- 源码：**是**
- 部署：把 JAR 安装进 JADX，再启动 FastMCP Adapter

该项目结构相对简单，很适合用来学习“最小可用 JADX MCP”实现。

## 25. APKLab/APKLab

- 链接：<https://github.com/APKLab/APKLab>
- Stars：3,910
- 最近更新：2026-07-16
- 定位：VS Code Android Reverse Engineering Workbench
- 落地能力：把 Android 反编译 / 修改 / 分析整合进 VS Code，JADX 是反编译工具链的重要一环
- 关联性：**中高**
- 源码：**是**

---

# 六、知乎

> 知乎站内接口未稳定返回赞同 / 阅读等互动数据，因此不虚构热度；时间、作者和内容取当前搜索结果。

## 1. 使用 JADX-MCP 高效反混淆

- 链接：<https://zhuanlan.zhihu.com/p/2010779940338029424>
- 时间：2026-02-27
- 作者：看雪
- 热度：站内结果未提供可验证互动数
- 关联性：**极高**
- 可验证实现：**是**
- 部署 / 复现：**有**

主要内容：

- 给出 MCP Server 配置。
- 列出 `fetch_current_class`、`get_class_source`、`search_method_by_name`、`get_android_manifest`、`get_smali_of_class` 等工具。
- 给出“Scope Definition → Deep Analysis → Evidence-based Rename”的 Agent 流程。
- 针对高混淆 App 做逐类、逐方法证据化反混淆。

## 2. 手把手配置 JADX-AI-MCP：人人都能用 AI 给手机 App 动手术

- 链接：<https://zhuanlan.zhihu.com/p/2047727749872329414>
- 时间：2026-06-09
- 作者：wallech
- 关联性：**极高**
- 可验证实现：**是**
- 部署 / 复现：**完整**

主要内容：

- 下载 `jadx-mcp-server` 和 `jadx-ai-mcp` Plugin JAR。
- 安装 Python 依赖。
- 在 JADX Plugin Manager 安装插件。
- 配置 Trae MCP JSON。
- 继续衔接 apktool / apktool-mcp-server。

## 3. 电子数据取证的常用 MCP 工具搭建及应用

- 链接：<https://zhuanlan.zhihu.com/p/2066591307351565853>
- 时间：2026-07-31
- 作者：小谢取证
- 关联性：**高**
- 可验证实现：**是**
- 部署 / 复现：**有**
- 真实应用场景：**有**

内容：SSH MCP、Chrome DevTools MCP、JADX MCP、IDA MCP。JADX 部分包括 Python 3.11、jadx-gui 1.5.3、插件 / Server 下载、虚拟环境、MCP JSON，并演示 Agent 直接反编译打开中的 APK。

## 4. 我把 Cursor / Claude / Codex 变成了 APK 逆向工程师：从 jadx 迷路到一份可交付 REPORT

- 链接：<https://zhuanlan.zhihu.com/p/2064763781104801029>
- 时间：2026-07-26
- 作者：小杨技术铺
- 关联性：**极高**
- 可验证工作流：**是**
- 报告交付：**有**

提出 APK Reverse Harness：

```text
APK Context
→ Architecture / Protocol
→ Critical Path
→ Evidence
→ REPORT.md
```

重点解决 AI 在几千个类中漫游、过早下结论、无法续接分析和最终无法交付的问题。

## 5. Android Reverse Engineering Skill：开源 Android 逆向工程工具

- 链接：<https://zhuanlan.zhihu.com/p/2028747106865750625>
- 时间：2026-04-18
- 作者：梁川
- 关联性：**高**
- 开源源码：**有**
- 真实实现：**有**

介绍的能力：

- APK / XAPK / JAR / AAR。
- JADX / Fernflower / Vineflower 多引擎并行。
- 自动提取 Retrofit、OkHttp、URL、认证头、Token 模式。
- Activity / Fragment → ViewModel / Repository → Network 调用链追踪。
- 五阶段工作流：依赖检查 → 反编译 → 结构分析 → API 提取 → 调用流追踪。

## 6. APP 逆向全过程

- 链接：<https://zhuanlan.zhihu.com/p/709619632>
- 时间：2024-07-18
- 作者：搜索结果未稳定返回作者名
- 关联性：**高**
- 真实案例：**是**

环境：Pixel 2XL Android 8.1、Charles + Postern、JADX 1.4.5、IDA Pro 8.3、Frida。通过抓包发现 Header 参数后，在 JADX 全局搜索并定位具体 `GalaxyResponse` / Request 代码生成路径。

## 7. 我的第一次逆向实战

- 链接：<https://zhuanlan.zhihu.com/p/2064465551347524633>
- 时间：2026-08-01
- 作者：搜索结果未稳定返回作者名
- 关联性：**高**
- 真实案例：**是**

流程：APK 静态扫描 → 证书 / 包名异常 → APKiD 识别 Jiagu 加固与反模拟器逻辑 → JADX → IDA 分析 `.init_array` 和早期 Native 执行逻辑。

## 8. Bugku CTF 安卓逆向 LoopAndLoop

- 链接：<https://zhuanlan.zhihu.com/p/686444220>
- 时间：2024-03-11
- 作者：搜索结果未稳定返回作者名
- 关联性：**中高**
- 真实案例：**是**

使用 JADX 还原 Activity 和 JNI 声明，进一步理解 Native 校验路径；规模较小，但可用于入门复现。

---

# 七、掘金

> 本轮指定 Tavily 搜索通路不可用，未安装 / 登录 / 改配置；使用通用 Web 字面查询 `site:juejin.cn JADX ...` 补充。当前较深的掘金实战多为 2021–2022 年文章。

## 1. Android 逆向之某 APP 逆向实践

- 链接：<https://juejin.cn/post/6949050113595539469>
- 时间：2021-04-09
- 可见热度：约 15,042 阅读
- 关联性：**高**
- 真实案例：**是**
- 部署 / 复现：**有**

流程：

- 分析壳 / Fake Application。
- JADX 静态分析。
- Root / 模拟器检测。
- Frida-DEXDump 脱壳。
- 导出真实 DEX 后再次 JADX 分析。

## 2. 安卓逆向入门与实践（上）

- 链接：<https://juejin.cn/post/7136957007113748517>
- 时间：2022-08-28
- 可见热度：约 775 阅读
- 关联性：**中高**
- 真实案例：**是**

对实际 APK 使用 JADX 搜索 WifiManager、JavascriptInterface 等功能点并定位业务逻辑。

## 3. 安卓逆向入门与实践（下）

- 链接：<https://juejin.cn/post/7138069952057049118>
- 时间：2022-08-31
- 可见热度：约 419 阅读
- 关联性：**高**
- 真实案例：**是**

形成完整修改闭环：

```text
JADX 定位
→ apktool 反编译
→ 修改 Smali
→ 回编译
→ uber-apk-signer 签名
→ 安装验证
```

## 4. 安卓逆向工具笔记

- 链接：<https://juejin.cn/post/6961000800160055304>
- 时间：2021-05-11
- 可见热度：约 1,459 阅读
- 关联性：**中**
- 实现 / 操作：**有**

JADX、apktool 等 Android 逆向工具实际使用记录，落地深度低于前三篇。

## 5. Android 之用 jadx 进行反编译

- 链接：<https://juejin.cn/post/7090576171510792200>
- 时间：2022-04-25
- 可见热度：约 617 阅读
- 关联性：**中**
- 复现：**有**

偏基础反编译命令和 GUI 操作，但属于可复现教程。

---

# 八、Bilibili

> Bilibili 搜索本轮未稳定返回发布时间字段，因此不猜日期；热度采用搜索 / 详情工具返回的当前播放与互动。

## 1. 64.jadx-ai-mcp 环境搭建 AI 逆向分析 JAVA 代码

- 链接：<https://www.bilibili.com/video/BV1dLCQBLEJC>
- UP：和沐阳学逆向与Ai
- 时长：15:18
- 播放：6,651
- 点赞：116
- 投币：52
- 收藏：321
- 分享：26
- 关联性：**极高**
- 部署 / 复现：**有**

简介直接说明：自动 MCP Server + JADX Plugin，通过 MCP 与 Claude 等 LLM 分析 APK、发现漏洞和做逆向。收藏显著高于点赞，呈现明显“准备照着部署”的工具教程特征。

## 2. jadx AI 辅助插件（重命名，反编译，注释，调用分析）

- 链接：<https://www.bilibili.com/video/BV1UWXbYeEGm>
- UP：Wker666
- 时长：23:24
- 播放：5,349
- 关联性：**极高**
- 实现演示：**有**

展示 AI 在 JADX 中做 Rename、反编译解释、注释与调用分析。

## 3. APK 反编译实战！用 Jadx 分析 APP 源码

- 链接：<https://www.bilibili.com/video/BV1uiTfzEEKh>
- UP：网络安全收藏家
- 时长：96:56
- 播放：31,172
- 关联性：**高**
- 真实实战：**有**

偏传统 Android 静态逆向，长时实操。

## 4. 通过 jadx gui 反编译 APK 为 Android Studio 源码

- 链接：<https://www.bilibili.com/video/BV1SA411U7N3>
- UP：请叫我老龚a
- 时长：17:51
- 播放：3,461
- 关联性：**中高**
- 复现：**有**

重点是把 JADX 输出继续整理成 Android Studio 可分析的源码工程。

## 5. 逆向工程师必备的 AI 神器—JADX！AI 助力逆向分析

- 链接：<https://www.bilibili.com/video/BV14fnMzqEiA>
- UP：官方Python开发课程
- 时长：6:52
- 播放：1,323
- 关联性：**高**
- 实操：**有**

展示 AI + JADX 辅助逆向的使用方式，内容较短。

## 6. AI 自动逆向 Java + SO 层，APK 自动审查

- 链接：<https://www.bilibili.com/video/BV1LWXVBfE4j>
- UP：白鸽zv
- 时长：5:56
- 播放：4,751
- 关联性：**高**
- 实操：**有**

已经从 Java 反编译延伸到 Java + Native SO 联合分析，与 jexray / Ghidra MCP 路线一致。

---

# 九、X

> X 结果来自只限定 x.com 的检索，未把普通 Web 页面当作 X 证据。

## 1. Het Mehta：JADX-AI — AI-Powered Reverse Engineering via MCP + Claude Desktop

- 链接：<https://x.com/hetmehtaa/status/1910588746912670022>
- 作者：@hetmehtaa / Het Mehta
- 时间：2025-04-11
- 热度：263 Likes / 98 Reposts / 164 Bookmarks / 13,196 Views
- 关联性：**极高**
- 开源实现：**有**

JADX-AI-MCP 较早的公开发布帖，明确给出 MCP + Claude Desktop + JADX 的方向，并链接插件 / Server 源码。

## 2. Dan Kornas：JADX-AI-MCP 功能与部署介绍

- 链接：<https://x.com/DanKornas/status/2059722783467184408>
- 作者：@DanKornas
- 时间：2026-05-27
- 热度：15 Likes / 3 Reposts / 22 Bookmarks / 1,916 Views
- 关联性：**极高**
- 部署证据：**有**

描述 Plugin JAR + `uv` Python Server + Claude Desktop，并列出 Source / Smali / Manifest / Resource / Xref / Deobfuscation / SAST 等能力。

## 3. Pavan：jadx-gui + jadx-ai-mcp 实际连接 OpenCode

- 链接：<https://x.com/eh_pavan/status/2086680093657796816>
- 作者：@eh_pavan
- 时间：2026-08-10
- 热度：10 Likes / 4 Replies / 287 Views
- 关联性：**高**
- 用户实践证据：**有**

直接说明其 JADX GUI 已安装 `jadx-ai-mcp` 并接到 OpenCode 使用。

## 4. gia ly bui：Android 私有 API 提取工作流

- 链接：<https://x.com/gialybui/status/2091137315838263681>
- 作者：@gialybui
- 时间：2026-08-22
- 可见热度：低互动
- 关联性：**高**
- 实战流程：**有**

给出典型三段：

```text
jadx decompile + grep @GET/@POST
→ Frida bypass SSL pinning
→ Reverse signing algorithm and reproduce in Python
```

说明 JADX 在完整 API 逆向工作流中负责静态定位与第一阶段代码理解。

## 5. John Ebinyi Odey：不要只看 DEX / JADX，也要分析 JNI / SO

- 链接：<https://x.com/i_am_giannis/status/2091089598352961820>
- 作者：@i_am_giannis
- 时间：2026-08-22
- 热度：1 Like / 1 Reply / 22 Views
- 关联性：**中高**
- 实践证据：**有**

强调 Android Pentest 不能只分析 JADX 输出，应继续检查 Native `.so`，并使用 Ghidra / IDA。

## 6. panoryama：隔离环境中的 JADX 分析

- 链接：<https://x.com/panoryama/status/2091105151893577918>
- 时间：约 2026-08-22
- 可见热度：约 2 Likes
- 关联性：**中**
- 实践证据：**有但较弱**

描述下载 App 后用 JADX Decompile，并在 PC + Android Emulator 隔离环境中分析。

---

# 十、DeepSeek

本轮 DeepSeek 连接器明确返回：**当前工具无实时搜索能力**。

因此：

- 实时可验证命中：**0**
- 不把其模型记忆中的 MobSF、Quark-Engine、APKLeaks 等内容当作本次实时搜索结果。
- DeepSeek 返回的可用价值仅是检索方向建议，未计入已确认落地条目。

状态：**不可用于本轮实时来源核验**。

---

# 十一、Exa

## 1. samudoria/revoid

- 链接：<https://github.com/samudoria/revoid>
- 时间 / Stars：本次 Exa 未返回可靠数值
- 关联性：**极高**
- 源码：**是**
- 可验证实现：**是**

实现特点：Headless `jadx-core`、Class / Method / String In-memory Index、Caller / Callee、Call Graph、分页、Ghidra Headless Native 分析。特别适合研究 LLM-friendly 的程序分析 API。

## 2. xjoker/delamain

- 链接：<https://github.com/xjoker/delamain>
- ES 补核：2 Stars；2026-07-24 更新
- 关联性：**极高**
- 源码 / Docker：**有**

Exa 补充确认了它的 Java Backend + Python Gateway、Docker、Bearer Token、OOB 文件上传、Call Graph、Data Flow、Frida、Security Scan、Annotation 等详细能力。

## 3. xjoker/jadx-ai-mcp

- 链接：<https://github.com/xjoker/jadx-ai-mcp/>
- 时间：2026-01-12 起
- 关联性：**极高**
- 源码：**是**

实现特点：

- Smart Batch Optimization。
- 大响应 Transfer API，规避 MCP 响应限制。
- ClassCacheManager。
- All-in-One Docker。
- 多实例。
- MCP / Plugin Token 鉴权。
- 约 39 个 MCP Tools。
- 提供 Quickstart、Logic Tracing、Security Audit、Crypto Analysis、Frida Hooks、Refactoring 等 Skills。

适合参考“生产化 JADX MCP”所需的缓存、传输和鉴权设计。

## 4. jygzyc/decx

- 链接：<https://github.com/jygzyc/decx>
- Stars：71
- 最近更新：2026-08-13
- 关联性：**极高**
- 源码：**是**

Exa 返回了 DECX 的 CLI / Skill / BlackBoard 细节：`.decx-analysis/.../decx-analysis.db` 保存事实和链路，下游 Report / PoC Skills 消费这些状态。

## 5. zinja-coder/jadx-ai-mcp

- 链接：<https://github.com/zinja-coder/jadx-ai-mcp>
- Stars：2,609
- 关联性：**极高**
- 与 GitHub 组重复，但 Exa 补充了完整 Tool 列表和非 localhost 绑定的安全警告。

## 6. zinja-coder/jadx-mcp-server

- 链接：<https://github.com/zinja-coder/jadx-mcp-server>
- Stars：738
- 关联性：**极高**
- Exa 补充确认：支持 Source、Method / Field、Smali、Manifest、Resource、Rename、Debugger、Xref 等工具。

## 7. Qtty/jadx-mcp-server

- 链接：<https://github.com/Qtty/jadx-mcp-server>
- Stars：28
- 最近更新：2026-08-20
- 关联性：**极高**
- Exa 明确返回 Pure-Java MCP / API / Core / CLI 分层，以及 Component / Resource / Bytecode 分析工具。

## 8. mobilehackinglab/jadx-mcp-plugin

- 链接：<https://github.com/mobilehackinglab/jadx-mcp-plugin/>
- 关联性：**极高**
- 源码：**是**

实现：Java Plugin 通过 Embedded HTTP 暴露 Jadx API，Python FastMCP Adapter 把 Claude MCP 调用翻译成 HTTP POST；适合作为最小 MCP Bridge 学习样例。

## 9. JADX + MCP: I let the AI read the APK so I don’t have to

- 链接：<https://infosecwriteups.com/jadx-mcp-i-let-the-ai-read-the-apk-so-i-dont-have-to-548d1e8210e6>
- 时间：2026-04-08
- 作者：Xcheater
- 关联性：**极高**
- 部署 / 实战：**有**

Exa 返回了完整 Plugin 安装、MCP Server 启动、Cursor 配置以及 Manifest → Attack Surface → Source-to-Sink → Reporting 的实际使用步骤。

## 10. JADX-AI-MCP Examples

- 链接：<https://jadx-ai-mcp.readthedocs.io/en/latest/examples/>
- 来源：官方文档
- 关联性：**极高**
- 可复现工作流：**有**

官方示例包括：

- Comprehensive Security Audit
- Authentication Flow Analysis
- Systematic Deobfuscation
- Native Method Analysis
- Code Refactoring
- Malware Behavioral Analysis
- Permission / Privacy Analysis

这些例子已经可以直接改造成 Agent Skill / Prompt Template。

---

# 十二、结论与可直接复用的技术路线

本轮最重要的结论不是“JADX 是一个好用的 APK 反编译器”，而是：

> **JADX 已经非常适合充当 AI / Agent Android 程序分析系统的底层解析与代码导航引擎。**

当前有三种成熟二开路线：

### 路线 A：JADX GUI Plugin + MCP

代表：

- `zinja-coder/jadx-ai-mcp`
- `mobilehackinglab/jadx-mcp-plugin`

适合：保留人工 GUI 分析体验，同时让 AI 读取当前类、Xref、资源、Smali、Rename 和 Debugger。

```text
JADX-GUI
→ Plugin
→ HTTP / MCP Server
→ Claude / Codex / Cursor
```

### 路线 B：直接 jadx-core Headless

代表：

- `1013503897/jadx-headless-mcp`
- `samudoria/revoid`
- `Qtty/jadx-mcp-server`
- `xjoker/delamain`

适合：真正做后端产品、批处理、API 服务、CI、Docker 和多用户分析。

```text
APK Upload
→ jadx-core
→ Index / Search / Xref / Graph
→ REST / MCP
→ LLM / Rules
```

### 路线 C：完整 Agent-Native 分析平台

代表：

- `jygzyc/decx`
- `Fausto-404/ai-mobile-reverse-skills`
- `open-reverselab`

适合：不仅解析源码，还要维护分析状态、证据、阶段、调用链、Native 结果、报告和验证任务。

```text
APK
→ Static Recon
→ Attack Surface
→ Code / Xref / Graph
→ JNI / Ghidra
→ Evidence Blackboard
→ Findings
→ Validation / PoC
→ REPORT.md
```

如果后续要开发自己的系统，优先拆解顺序建议：

1. `zinja-coder/jadx-ai-mcp`：看完整工具面。
2. `1013503897/jadx-headless-mcp`：看如何摆脱 GUI。
3. `Qtty/jadx-mcp-server`：看 Pure Java 分层。
4. `samudoria/revoid`：看调用图、索引和 Ghidra Headless。
5. `xjoker/delamain`：看产品化传输、Docker、鉴权和缓存。
6. `jygzyc/decx`：看分析状态 / Evidence / Report / PoC 的 Agent 工作流。
7. `cys7885/jexray`：看 Java 与 JNI / SO 结合。
8. `Arsylk/jadx-string-decrypt`：看 Jadx Pass / 反混淆插件实现。
