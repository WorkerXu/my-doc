# dify-for-dsl：AI 资讯每日新闻与语音播报工作流

- 项目地址：https://github.com/wwwzhouhui/dify-for-dsl
- 调研版本：`c76ae793fdec79a1595b927954f870b54e5d7403`
- 重点工作流：`dsl/AI资讯每日新闻+语音播报工作流.yml`
- 抓取服务：`dsl/crawl4ai/aibase_craw4fastapi.py`
- 依赖文件：`dsl/crawl4ai/requirements.txt`
- 安装说明：`dsl/crawl4ai/readme`
- 调研目标：分析“列表发现 → Crawl4AI 结构化抽取 → FastAPI → Dify → LLM → TTS → 音频交付”的实现细节、技术原理、运行边界和生产化风险，并提炼适用于 1000 个技术博客长期知识库的架构优化。

## 1. 项目定位与整体链路

这个项目本质上不是一个通用爬虫平台，而是一个把网页抓取能力嵌入 Dify 工作流的最小示例。它把 AIbase 新闻采集做成 FastAPI 服务，再由 Dify 完成摘要、文本拼装、TTS 和最终回答。

完整链路如下：

```text
用户在 Dify 选择新闻数量
  -> Dify Code 节点调用 http://127.0.0.1:8086/news/?limit=N
  -> FastAPI /news/
  -> requests.get("https://www.aibase.com/zh/news")
  -> BeautifulSoup 遍历 a[href]
  -> 过滤 /zh/news/ 详情链接
  -> 每篇文章创建 AsyncWebCrawler
  -> JsonCssExtractionStrategy 提取 title/publication_date/content
  -> FastAPI 返回 news[] + newsdetail
  -> Dify LLM 对 news[] 做摘要
  -> Template 拼接“摘要 + 多篇新闻全文”
  -> TTS Tool 生成音频
  -> Code 节点解析供应商返回值并拼接 <audio> HTML
  -> Answer 节点返回文本和播放器
```

这条链路证明了三个事实：

1. Crawl4AI 很适合被包装为站点专用抓取 Adapter；
2. Dify 很适合编排 LLM、TTS 和交付流程；
3. “能在一个请求里串通”不等于“适合做长期、百万级、可增量、可审计的知识库”。

对于 1000 站点方案，应该保留能力分层，反转运行关系：抓取平台长期同步数据，Dify/n8n 只消费稳定 Version/Artifact，而不是临时发起全链路抓取。

## 2. Dify DSL 的执行图与数据流

工作流为 `advanced-chat`，DSL 版本字段为 `0.1.5`。开始节点定义变量 `item`，标签为“新闻数量”，类型为 select，当前选项只有 `2`。

主要节点关系：

```text
Start
  -> Code: 调用服务端 crawl4ai 获取每日 AI 资讯新闻
  -> LLM: 新闻总结助理
  -> Template: LLM 新闻总结信息 + 新闻详细信息合并
  -> Tool: 新闻详细内容转换语音播报
  -> Code: 文字转音频文件处理
  -> Answer
```

这个 Graph 本身已经承载大量业务语义：

- 从哪个服务获取内容；
- 请求参数如何映射；
- 哪个模型做摘要；
- Prompt 是什么；
- temperature 是多少；
- TTS 使用哪个模型、哪个 voice；
- 如何把 TTS 返回值转为播放器；
- 最终回答如何拼装。

因此 DSL/Graph 不能只当“部署配置文件”。它是实际行为定义，必须像代码一样版本化、测试、发布和回滚。

## 3. Discovery：当前列表页 DOM 扫描

`get_news_urls()` 使用同步 `requests.get()` 请求：

```text
https://www.aibase.com/zh/news
```

然后用 BeautifulSoup：

```python
news_items = soup.find_all('a', href=True)
```

对每个 href 判断：

```python
if '/zh/news/' in link and len(link.split('/')) > 3:
```

满足条件就拼接 `https://www.aibase.com`，形成详情 URL。

### 3.1 可借鉴点

- URL Discovery 和正文 Extraction 已经有基本分工；
- 从 DOM 的真实 href 发现 URL，而不是先把页面转 Markdown 再靠正则恢复链接；
- 对单站点、小规模、结构稳定的列表页，这种规则成本低、可解释、易调试。

### 3.2 无法承担历史完整性的原因

当前 Discovery 没有：

- Sitemap/Sitemap Index；
- RSS/Atom/JSON Feed；
- Archive/年月分页；
- Category/Tag/Author；
- 官方 API；
- 分页游标；
- Provider checkpoint；
- URL normalize/canonical；
- URL 去重；
- Discovery Evidence；
- Coverage Evidence；
- reconcile。

它只回答“当前这个列表页上看到了哪些详情链接”，不能回答“这个站点历史上应该有哪些文章”。

对于 1000 站点，必须把 Discovery 抽象为 Provider，Provider 只产生 URL Candidate + Evidence；Coverage 由 Provider 的 enumeration boundary、cursor exhausted、oldest/newest、分页连续性、expected/discovered、known gap 和 blocker 证明。

## 4. Crawl4AI 结构化抽取的实现原理

单篇文章使用 `JsonCssExtractionStrategy`，Schema 近似：

```text
baseSelector: div.pb-32
fields:
  title:
    selector: h1
  publication_date:
    selector: div.flex.flex-col > div.flex.flex-wrap > span:nth-child(6)
  content:
    selector: div.post-content
```

Crawl4AI 获取页面表示后，以 CSS Selector 把目标节点映射到 JSON 字段。

这种方式有很强的生产价值：

- 确定性高；
- 成本远低于逐页 LLM 抽取；
- 字段 provenance 清晰；
- 对已知 CMS/模板可获得很高精度；
- 可以作为站点 Profile 的优先 Extractor。

但它也非常依赖 DOM 稳定性。`span:nth-child(6)` 属于位置型 selector，`pb-32` 属于样式类；站点换主题、调整布局或插入一个 span，都可能导致字段漂移。

因此生产系统不能只有“crawler success”，必须至少拆分：

```text
Transport Outcome
Extraction Outcome
Quality Outcome
Identity/Publish Outcome
```

网络成功但 selector miss，应是 `SELECTOR_MISS/NEEDS_PROFILE_REVIEW`；抽到 challenge/login/nav，应是内容质量失败；空正文不能覆盖已有正常 Version。

CSS/XPath Schema 应进入 `site_profile_release/extractor_release`，并通过 Golden URL、Snapshot Replay、selector miss 指标、正文长度漂移、shadow test 和回滚治理。

## 5. Crawl4AI 生命周期与并发模型

`extract_ai_news_article()` 每篇文章都执行：

```python
async with AsyncWebCrawler(verbose=True) as crawler:
    result = await crawler.arun(...)
```

而 `fetch_news()` 又是普通串行循环：

```python
for index, url in enumerate(news_urls, start=1):
    news_data = await extract_ai_news_article(url)
```

这会带来：

- 每篇重新创建/销毁 crawler runtime；
- Browser/Context/连接池无法充分复用；
- N 篇串行；
- 单个慢页面拖慢整个请求；
- 初始化浏览器的固定成本被重复支付；
- 进程中断后没有持久断点；
- 一篇失败时没有独立 lease/retry/dead-letter。

对于长期平台，正确形态是：

```text
Durable Task
  -> HTTP Runtime Pool / Browser Runtime Pool
  -> bounded local concurrency
  -> per-host/global token bucket
  -> lease + heartbeat + retry
```

`asyncio.gather` 或 semaphore 只适合 Worker 内部资源保护，不能替代持久任务系统。

Browser Context、HTTP Session、Model Runtime Handle 都需要 hard cap、idle eviction、in-flight pin、最大寿命/页面数/内存阈值和 finally 释放。

## 6. Async FastAPI 中的同步阻塞 I/O

FastAPI `/news/` 是 `async def`，但 `get_news_urls()` 使用同步 `requests.get()`。同步网络调用会阻塞当前 Event Loop 线程。

Dify Code 节点请求 FastAPI 时同样使用同步 `requests.get()`。

即使把同步请求替换成 `httpx.AsyncClient`，仍然没有解决架构问题，因为一次用户请求仍同步等待：

```text
列表页
-> N 篇抓取
-> LLM
-> TTS
-> 输出
```

任何慢站、Browser 启动、模型排队、TTS 长耗时都会扩大用户请求时长和失败半径。

生产 API 应采用 Job 语义：

```text
POST /v1/runs              -> 202 + run_id
POST /v1/derived-jobs      -> 202 + job_id
GET  /v1/jobs/{id}         -> status/progress/result_ref
GET  /v1/artifacts/{id}    -> metadata + access reference
```

Dify/n8n/Web 只提交任务、查询结果或等待回调，不在一个 HTTP 请求里同步承载 Browser + LLM + TTS。

## 7. `bypass_cache=True` 不等于增量同步

示例的 Crawl4AI 调用使用 `bypass_cache=True`，其语义只是“不使用 Crawl4AI 的缓存”。它不能证明内容有更新，也不等于业务层 freshness。

长期增量同步应优先读取低成本 Change Signal：

```text
Feed GUID/updated
Sitemap lastmod
API update cursor
ETag / Last-Modified
conditional GET
body hash
Canonical IR hash
```

推荐决策：

```text
Signal unchanged
  -> 跳过正文请求

changed/unknown
  -> conditional fetch

Network representation unchanged
  -> Freshness Observation

Canonical IR unchanged
  -> Freshness Observation，不建新 Version

Canonical IR changed
  -> append-only 新 Version + 刷新 Projection
```

这样才能让 1000 个站点长期运行时把成本集中在真正发生变化的文档上。

## 8. FastAPI 返回 Contract 的漂移

FastAPI 返回：

```json
{
  "news": [],
  "newsdetail": "..."
}
```

Dify Code 节点又显式兼容 `news` 的元素既可能是 dict，也可能是 JSON string。这说明服务边界上的类型已经存在不稳定性。

`newsdetail` 更进一步把多篇正文拼成无结构字符串：

```text
今天新闻第1条内容：...；
今天新闻第2条内容：...；
```

这种模式在 Demo 中方便，在知识库系统中会丢失：

- document identity；
- version identity；
- canonical URL；
- published_at；
- block boundary；
- source provenance；
- 单篇失败/重试粒度；
- chunk lineage。

生产 Contract 应传稳定引用：

```text
DOCUMENT_VERSION_REF
MARKDOWN_PROJECTION_REF
AI_INPUT_PROJECTION_REF
SELECTION_MANIFEST_REF
DERIVED_ARTIFACT_REF
AUDIO_ARTIFACT_REF
DELIVERY_PAYLOAD_REF
```

大正文存对象存储，跨服务传 Artifact Ref/Object Key；小 metadata 才内联 JSON。

## 9. LLM 摘要：整批输入与可重复性

工作流的 LLM 节点：

- Provider：SiliconFlow；
- Model：`internlm/internlm2_5-7b-chat`；
- temperature：`0.7`；
- 输入：整个 `news[]`；
- 输出要求：摘要 + 数字编号文章要点。

小规模两篇可以运行，但扩大后会出现：

- 多篇正文 token 总量不可控；
- 某篇过长挤占其余文章上下文；
- 不可见截断；
- 整批失败必须整批重算；
- 单篇 Summary 无法缓存复用；
- 结果缺少 `document_version_id`；
- 输出无法回溯到 Canonical Block；
- temperature 较高会放大重放差异。

长期系统应使用：

```text
Document Version
  -> AI Input Projection
  -> block-aware Chunk Plan
  -> Map Summary Artifact x N
  -> Reduce Summary Artifact

多个单篇 Summary
  -> Selection Manifest
  -> Digest Reduce
  -> Digest Artifact
```

Prompt、temperature、语言、JSON Schema、token budget、截断策略、Model Release、Runtime Attestation 都进入 AI Recipe/Model Release。

单篇摘要和 Digest Item 至少保留：

```text
document_version_id
title
canonical_url
published_at
summary
key_points[]
source_block_refs[]
```

## 10. “每日新闻”不是 `limit=N`

DSL 的“新闻数量”目前只有 `2` 这个 select 选项。它只是控制当前列表前 N 条，并没有真正定义“每日”的时间语义。

缺失的问题包括：

- 哪个时区；
- window_start/window_end；
- cutoff；
- 跨午夜；
- 迟到文章；
- 同一天补跑；
- 失败重试时源列表已变化；
- 排序和去重依据；
- 实际选中了哪些不可变文档版本。

生产系统需要 Selection Manifest：

```text
selection_manifest
- id
- trigger_type
- timezone
- window_start/window_end
- as_of
- scope_json
- selection_policy_release_id
- ordered_document_version_ids
- ranking_evidence
- duplicate_suppression_evidence
- manifest_hash
```

必须先冻结输入，再做摘要/TTS。失败重试复用同一 Manifest；重新选文要显式建立新 Manifest。

## 11. TTS 链路与长任务特征

工作流把 LLM 摘要与 `newsdetail` 全文合并，然后送入 TTS Tool：

- Provider：`siliconflowmakeaudioapi`；
- Tool：`generate-audio_post`；
- Model：`FunAudioLLM/CosyVoice2-0.5B`；
- Voice：`FunAudioLLM/CosyVoice2-0.5B:david`。

工作流注释明确指出，两条新闻的语音生成大约需要 5 分钟。这说明 TTS 已经是典型高成本长任务。

生产形态应为：

```text
Digest Artifact
  -> TTS Recipe
  -> TTS Worker
  -> Audio Object
  -> Audio Artifact
  -> Delivery
```

默认只对 Summary/Digest 做 TTS，而不是重新把多篇完整正文拼进去。

TTS 需要独立控制：

- 最大输入字符/token；
- 目标音频最大时长；
- deadline；
- GPU/并发配额；
- Model/Voice Release；
- 重试；
- 成本预算；
- Artifact hash/lineage。

TTS 失败不能影响 Document Version、Markdown、Search 或 Embedding。

## 12. Audio Code 节点暴露的两个具体问题

音频处理 Code 节点近似：

```python
def main(arg1: str) -> str:
    data = json.loads(arg1)
    filename = data['filename']
    url = data['etag']
    markdown_result = f"<audio controls><source src='{url}' type='audio/mpeg'>{filename}</audio>"
    return {"result": markdown_result}
```

### 12.1 缺失 `import json`

代码使用 `json.loads`，但节点代码本身没有 `import json`。除非运行时偶然把 `json` 注入全局命名空间，否则这是直接的运行时缺陷。

这个问题说明 Workflow DSL 不能只靠人工导入后点一次运行验证。发布前必须对 Code 节点做：

- 语法检查；
- 直接 import 扫描；
- 最小启动测试；
- 输入输出 Contract Test；
- target engine 实际沙箱运行。

### 12.2 把供应商 `etag` 当 URL

代码把 TTS 结果中的 `etag` 直接当成 URL，再拼 HTML。这会造成：

- 业务 Contract 绑定供应商私有字段；
- 字段语义一变就运行失败；
- UI 渲染和存储 Contract 耦合；
- 原始 HTML/URL 进入客户端，增加渲染安全风险。

应该由 TTS Adapter 归一化为标准 Audio Artifact：

```text
audio_artifact
- id
- source_derived_artifact_id
- tts_model_release_id
- voice_release_id
- mime_type
- duration_ms
- bytes
- object_key
- content_hash
- runtime_attestation_id
```

访问时按需生成短期 signed URL；客户端根据 typed metadata 渲染播放器，而不是让 Workflow 拼接供应商原始 HTML。

## 13. 服务发现与硬编码 localhost

Dify Code 节点写死：

```text
http://127.0.0.1:8086/news/
```

在容器/Kubernetes 环境中，`127.0.0.1` 指向当前容器或当前 Pod 网络命名空间，并不表示 crawler 服务。

长期方案必须使用：

- Service Discovery/API Gateway；
- 配置化 Endpoint；
- mTLS/Token；
- Secret Manager；
- connect/read/total deadline；
- retry budget；
- circuit breaker；
- request/response byte limit；
- trace id。

“禁止 localhost/固定容器地址”还应进入 Workflow Preflight，而不是只靠编码规范。

## 14. 依赖与可重复运行

`requirements.txt` 当前声明：

```text
uvicorn==0.34.0
fastapi==0.115.6
cos-python-sdk-v5==1.9.33
crawl4ai==0.4.247
```

但 Python 源码直接 import：

```text
requests
bs4 / BeautifulSoup
```

这两个直接依赖没有在该 requirements 文件显式声明。

安装说明还要求：

```text
python -m playwright install --with-deps chromium
```

因此可重复运行实际上还依赖浏览器 revision、Playwright、操作系统库和 Crawl4AI 的传递依赖。

生产 Release 应记录：

- 所有直接依赖；
- lockfile；
- container image digest；
- SBOM；
- Python/OS；
- Crawl4AI/Playwright 版本；
- Chromium revision；
- parser/extractor 版本；
- source commit；
- import/startup/contract test 结果。

不能依赖“上游包刚好带了 requests/bs4”。

## 15. Workflow Recipe Release：DSL 本身必须版本化

工作流变化不只是 Prompt 变化。任何以下改动都会改变最终行为：

- 节点增删；
- edge 变化；
- variable selector 变化；
- Code 节点代码变化；
- Prompt；
- Model/Provider；
- Tool/Plugin；
- TTS voice；
- Template；
- Endpoint；
- Secret scope；
- Sandbox 网络权限。

推荐保存：

```text
workflow_recipe_release
- id/version
- engine_type
- dsl_object_key
- dsl_hash
- graph_hash
- compatible_engine_version
- input_schema/output_schema
- referenced_recipe_release_ids
- plugin_tool_release_refs
- model_provider_release_refs
- resolved_node_manifest_hash
- code_node_hashes
- dependency_contract_hash
- sandbox_policy_release_id
- preflight_report_ref
- source_commit
- test_result_refs
```

Derived/Delivery Job 必须记录实际使用的 Workflow Recipe Release。

## 16. 新增关键边界：Code/Plugin 节点本身是安全边界

这个项目中最容易被忽略、但对生产系统最重要的事实是：Dify Code 节点可以直接执行 Python，并且示例确实在节点中使用 `requests.get()` 访问一个 HTTP 服务。

如果长期方案只规定“外部 URL 要走 `TARGETED_FETCH`”，却让 Workflow Code/Plugin 节点拥有自由出网能力，那么约束仍然可以被绕过：

```text
Workflow Code
  -> requests.get(任意 URL)
  -> 绕过 Admission
  -> 绕过 SSRF 检查
  -> 绕过 robots/policy
  -> 绕过 Snapshot
  -> 绕过 Quality/Identity
  -> 绕过 Audit
```

同样，如果 Code/Plugin 节点直接拿到 PG/S3/Redis 凭据，就可能绕过 Canonical Version 和 append-only 规则修改真相层。

因此 Code/Plugin/第三方 Tool 应被视为“不可信扩展运行时”，不是普通业务代码。

## 17. Workflow Sandbox 的生产设计

推荐增加 `workflow_sandbox_policy_release`，至少控制：

```text
- allowed_egress_services
- denied_cidr/private_network
- dns_policy
- redirect_policy
- allowed_secret_scopes
- cpu_limit
- memory_limit
- pid_limit
- ephemeral_storage_limit
- max_output_bytes
- execution_deadline
- filesystem_policy
- syscalls/seccomp profile
- runtime image digest
```

默认策略：

1. `deny-all` egress；
2. 只允许 Integration Gateway、批准的 LLM/TTS Provider、对象代理和 Destination；
3. 任意源站抓取只能提交 `TARGETED_FETCH`；
4. 禁止访问 PG/Redis/S3 管理端、Kubernetes API、Docker socket、metadata、内部管理网；
5. rootfs 只读，临时盘限额；
6. Secret 以 Workflow/Node 最小 scope + 短期 token 提供；
7. CPU/Memory/PID/deadline/output bytes 都有限额；
8. DNS 和 redirect 也经过 egress policy，避免重绑定/跳转绕过；
9. Sandbox 拒绝必须成为显式 Outcome，而不是静默网络错误。

这比“代码评审禁止 requests”可靠，因为真正的安全边界必须在运行时强制执行。

## 18. Workflow DSL Preflight：发布前编译/校验门禁

结合这个 DSL 暴露的缺失 import、localhost、外部 Tool/Model 依赖，生产系统应在 Workflow Release 发布前做 Preflight。

### 18.1 结构检查

- DSL 能在目标引擎版本 parse/import；
- export/import round-trip 不丢关键字段；
- graph edge source/target 存在；
- variable selector 引用存在；
- Template 变量能解析；
- input/output schema 匹配。

### 18.2 Code 节点检查

- 语法；
- 未定义名称；
- 直接 import；
- requirements/运行镜像依赖；
- 最小函数调用；
- 输出类型；
- deadline/byte limit；
- 禁止 Endpoint/网络 API 模式。

这一步可以在发布前捕获类似“调用 `json.loads` 却没有 `import json`”的问题。

### 18.3 依赖解析

解析实际使用的：

- Model；
- Provider；
- Tool；
- Plugin；
- Voice；
- Workflow Adapter；
- Secret scope。

所有依赖要么固定到 Release，要么明确记录不可固定的外部版本和风险。缺失/禁用/版本不兼容时不能发布。

### 18.4 网络策略检查

阻断：

- localhost/127.0.0.1；
- 私网 CIDR；
- cloud metadata；
- 写死容器 IP；
- 未经批准的外部域名；
- Code/Plugin 直接抓源站。

### 18.5 Golden Execution

在和生产一致的 Sandbox Policy 下执行最小 Golden Input，验证：

- 每个节点能启动；
- 输出满足 Contract；
- 所有资源上限生效；
- 网络策略符合预期；
- Node lineage 完整。

Preflight 结果生成不可变报告，绑定 Workflow Recipe Release。

## 19. 节点级 Execution Evidence

只有 DSL hash 仍然不能解释“某次实际运行发生了什么”。模型、Tool、Plugin、Code 和外部 Provider 都可能在运行时产生差异。

建议记录：

```text
workflow_execution
- id
- workflow_recipe_release_id
- sandbox_policy_release_id
- trigger_type
- input_artifact_refs/input_hash
- status
- started_at/finished_at
- trace_id

workflow_node_execution
- workflow_execution_id
- node_id/node_type
- resolved_code_hash
- resolved_model_release_id
- resolved_plugin_tool_release_id
- input_artifact_refs/input_hash
- output_artifact_refs/output_hash
- secret_scope_ids
- network_policy_decision
- latency_ms/cost_units
- outcome/error_class
- trace_id
```

Secret 只记录 scope/id，不保存明文。

这样任意 Digest/TTS/Newsletter Artifact 都能回答：

- 用了哪个 Workflow Release；
- 哪个节点；
- 哪段 Code；
- 哪个模型/Plugin/Tool；
- 输入哪个 Document Version/Artifact；
- 输出 hash 是什么；
- 网络访问是否被允许；
- 哪个节点失败；
- 是否可以重放。

## 20. 抓取服务与 Workflow 的正确权限关系

Demo 的关系是：

```text
Dify Code
  -> 私有 FastAPI
  -> Crawl4AI
```

生产系统应该变成：

```text
Dify/n8n/Agent
  -> Integration Gateway
  -> 持久 Job API
  -> Source Sync / Targeted Fetch / Derived Job
  -> Accepted Version / Artifact
  -> Workflow 消费 stable ref
```

如果 Workflow 想临时抓某个 URL，也只能：

```text
POST /v1/targeted-fetch
  -> Admission
  -> SSRF/robots/policy
  -> Route Decision
  -> Snapshot
  -> Extraction
  -> Quality
  -> Identity
  -> Accepted Version
```

不能存在“为了工作流方便”而绕过主链路的第二套爬虫入口。

## 21. 对 1000 技术博客方案的直接优化清单

从该项目提炼出的可落地优化分为两类。

### 21.1 现有方案已经覆盖且应保留

1. Source Sync 与 Dify/n8n 解耦；
2. 外部工作流只消费 Accepted Version/Artifact；
3. 长任务统一使用异步 Job；
4. Selection Manifest 冻结日报输入；
5. 单篇 Summary + Digest Reduce，避免整批全文一次性输入 LLM；
6. TTS 默认消费 Digest；
7. Typed Audio Artifact 屏蔽供应商私有字段；
8. Workflow DSL/Graph 进入不可变 Release；
9. Runtime/依赖/Browser revision 可重复；
10. Endpoint 服务发现、Secret、超时、重试和 Circuit Breaker 配置化。

### 21.2 本次需要进一步补强

1. **Workflow Code/Plugin Sandbox**：把可执行节点视为不可信边界；
2. **默认拒绝出网**：任意源站访问强制走 Integration Gateway/Targeted Fetch；
3. **真相存储隔离**：Workflow 无 PG/S3/Redis 高权限直连；
4. **Secret 最小权限**：Node/Workflow scope + 短期 token；
5. **Workflow DSL Preflight**：在发布前发现缺失 import、localhost、变量引用、依赖/版本问题；
6. **Resolved Dependency Manifest**：固定模型、Tool、Plugin、Provider、Code hash；
7. **Workflow Node Execution Evidence**：实际执行节点、输入输出、网络决策、错误和成本可追溯；
8. **Sandbox/Preflight Web 管理**：在 Recipe Studio 查看报告、依赖、策略差异与节点 trace；
9. **Sandbox 错误分类**：`WORKFLOW_PREFLIGHT_FAILED`、`WORKFLOW_DEPENDENCY_UNAVAILABLE`、`WORKFLOW_SANDBOX_VIOLATION`、`WORKFLOW_EGRESS_DENIED`、`WORKFLOW_NODE_TIMEOUT`；
10. **发布门禁测试**：对 forbidden egress、DNS/redirect、Secret 访问、资源超限和引擎升级做自动测试。

## 22. 最终架构判断

这个项目很适合作为“Crawl4AI + Dify + LLM + TTS 可以联通”的教学样例，但它的设计目标是少量最新内容即时消费，不是长期知识库。

它不具备：

- 历史 Coverage；
- 多 Provider 枚举；
- 增量 checkpoint；
- durable frontier；
- Snapshot/Version 真相；
- 质量门禁；
- 断点续跑；
- 大规模公平调度；
- 可重放的 Selection；
- 节点级 Workflow 安全与执行证据。

对 1000 个技术博客，推荐最终关系：

```text
Source Registry
  -> Coverage Providers
  -> Durable Frontier
  -> Change Signal
  -> Admission / Security
  -> Route Decision
  -> Fetch / Browser
  -> Immutable Snapshot
  -> Extractor Candidates
  -> Canonical IR + Quality
  -> Identity + Append-only Version
  -> Markdown / Assets / Search / Embedding
  -> Selection Manifest
  -> Summary / Digest / TTS Artifact
  -> Workflow Preflight
  -> Sandboxed Workflow Execution
  -> Delivery Job
  -> Dify / n8n / Agent / Newsletter / Webhook
```

核心原则是：**Dify/n8n 负责受控编排，抓取平台负责真实、完整、增量、可追溯的数据；Workflow DSL 既是版本化业务逻辑，也是需要预检和沙箱的可执行供应链。只有同时治理 Source Sync 和 Workflow Runtime 两个边界，后续新增站点、模型、Plugin、TTS、Newsletter 或 Agent 时，系统才能保持可扩展、可审计、可重放且不被工作流便利性破坏数据真相。**