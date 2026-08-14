# 使用 Crawl4AI 构建面向 AI 的网页爬取流程

来源：https://medium.com/towards-data-engineering/ai-ready-web-crawling-using-crawl4ai-c4abc3701257

## 1. 文章核心思路

文章给出一条典型的“AI-ready crawling”链路：入口页发现链接 → 深度抓取 → URL 过滤/打分 → 内容抽取 → 结构化结果 → 向量库或 RAG。其重点不在单页抓取，而在于把“发现哪些页面”和“如何抽取页面内容”拆成两个可配置阶段。

Crawl4AI 中 `AsyncWebCrawler` 负责异步访问，`CrawlerRunConfig` 负责控制抓取行为，`CrawlResult` 提供 HTML、清洗 HTML、Markdown、链接和媒体等结果。文章进一步使用 BFS、DFS、Best-First 三类深度抓取策略，并通过 `DomainFilter`、`URLPatternFilter`、`KeywordRelevanceScorer` 把无关 URL 尽早排除。

## 2. 深度抓取的技术原理

站点可抽象成有向图，页面是节点，超链接是边。BFS 适合按层扩大覆盖面；DFS 适合沿一条路径深入，但容易被深层路径占用预算；Best-First 则通过启发式评分建立优先队列，优先访问更可能是目标内容的 URL。

对于 1000 个技术博客，不应直接把 Crawl4AI 内部 deep crawl 当成生产级全局 frontier。单次 deep crawl 更适合作为“某个 Provider 的发现执行器”。真正的生产 frontier、visited、priority、checkpoint、lease 应落 PostgreSQL，以保证进程崩溃后可恢复、可跨 worker 分片、可重放。

推荐把文章中的 Best-First 思路升级为平台级 `URLScore`：

`score = source_priority + path_pattern_score + content_type_score + recency_score + semantic_score - trap_penalty - duplicate_penalty`

其中 Sitemap/API/Feed 发现的文章页可以直接给予高分，归档页次之，未知站内链接较低；query 参数爆炸、日历页、搜索页等 crawler trap 给予高惩罚。

## 3. URL 过滤链

文章通过 `DomainFilter` 和 `URLPatternFilter` 在抓取前剔除无关链接，这一点对大规模系统非常关键，因为“减少无效请求”比“抓完再清洗”更节省网络、Browser 和抽取成本。

生产环境建议将过滤升级为可版本化的 Admission Pipeline：

1. URL 规范化：scheme/host、fragment、tracking query、默认端口、trailing slash；
2. SSRF/Egress 检查；
3. robots 与站点访问策略；
4. 域名/子域 allowlist；
5. URL path allow/deny；
6. Content-Type 预判；
7. canonical/redirect 去重；
8. crawler trap 检测；
9. Provider 级预算和最大深度；
10. URLScore 排序。

文章中的关键字 scorer 适合做软排序，不适合成为唯一准入规则，因为博客 URL 未必包含语义关键词。更稳妥的做法是把 URL pattern、页面模板证据和语义 scorer 分层组合。

## 4. 抽取策略

文章区分 LLM-free 与 LLM 两类抽取。

LLM-free 包括 CSS/XPath schema 和 Regex。CSS/XPath 对结构稳定站点成本低、速度快、可测试，适合 1000 站大规模运行。Regex 更适合字段级数据，例如日期、邮箱、价格等，不应承担正文 DOM 解析。

LLMExtractionStrategy 会把页面内容分块，结合 instruction 与 schema 发送给模型，再合并结果。优点是对非结构化页面适应性强，缺点是 API 成本、延迟、非确定性和重放成本都更高。

因此知识库方案应采用“确定性优先、LLM 例外升级”：

- 第一层：平台 API / JSON-LD / meta；
- 第二层：站点 Extraction Contract（CSS/XPath/JSON）；
- 第三层：Trafilatura/Readability/Crawl4AI Markdown 候选；
- 第四层：质量 Resolver；
- 第五层：仅对低质量或特殊结构页面启用 LLM 抽取。

LLM 输出只作为候选，不直接覆盖 canonical 正文；应保存模型、prompt、schema、输入 snapshot hash 和输出，保证可审计。

## 5. 对现有技术方案的优化

### 5.1 增加可版本化 URL Filter Chain

现有方案已经有 allow/deny 和 Admission，但应明确抽象 `url_filter_chain_release`，将 DomainFilter、URLPatternFilter、ContentTypeFilter、页面类型分类、crawler trap 检测和 scorer 统一版本化。每个 Discovery Provider 固化 filter release，避免规则变化导致同一历史任务语义漂移。

### 5.2 增加 URL Priority / Best-First Frontier

现有 durable frontier 应从简单 BFS/priority 扩展为可插拔评分。对历史回灌，优先处理 Sitemap/API/Feed 直接发现的详情页；对 HTML recursive provider，采用 Best-First，优先访问“高概率文章页或高价值归档页”。这样可以更早产出知识库内容，也能在预算不足时取得更高有效覆盖率。

### 5.3 将 deep crawl 限定为 Provider 局部执行器

Crawl4AI 的 BFS/DFS/BestFirst 可由 Adapter 暴露为能力，但不能替代平台全局 frontier。生产中的 URL 发现结果必须逐条回写 PostgreSQL，批次结束后由 Scheduler 再分发，而不是让某个 Browser/AsyncWebCrawler 实例无限递归。

### 5.4 确定性抽取优先、LLM 作为升级路径

对百万级文章，默认 CSS/XPath/DOM IR/Readability/Trafilatura；只有质量门禁失败、结构高度不规则或用户明确需要语义字段时才进入 LLM。建议给每个站点维护 `llm_escalation_policy` 与月度 token 预算。

### 5.5 增加抽取质量反馈闭环

抽取结果需记录 `extractor_candidate` 和质量分数。若某站点连续出现正文过短、导航比过高、标题不匹配，可自动触发 Probe 重新采样，生成新 Contract 草稿，再经 Web 审核/canary 发布。

## 6. 推荐落地接口

新增核心对象：

- `url_filter_chain_release`：过滤器列表、顺序、参数、版本；
- `url_scorer_release`：评分器及权重；
- `frontier.priority_score`：发现时计算，可重算但任务固化版本；
- `llm_escalation_policy`：触发条件、模型、预算、最大 token；
- `extraction_candidate`：多抽取器候选结果；
- `quality_evaluation`：正文长度、链接密度、标题匹配、结构完整度、重复度等；
- `discovery_run`：记录 Crawl4AI deep crawl 的策略、深度、max_pages、filter/scorer release 与产出 URL。

Adapter 可映射 Crawl4AI 的 `BFSDeepCrawlStrategy`、`DFSDeepCrawlStrategy`、`BestFirstCrawlingStrategy`、`FilterChain`、`DomainFilter`、`URLPatternFilter`、`KeywordRelevanceScorer`，但统一输出仍是平台标准的 discovered URL records。

## 7. 结论

这篇文章最值得吸收的不是某段 Crawl4AI 调用代码，而是三个设计原则：发现要有策略、URL 要在抓取前过滤、抽取要按成本分层。对于 1000+ 技术博客，应把这些能力提升为平台级、可版本化、可恢复的组件：Best-First 作为 durable frontier 的优先级思想，FilterChain 作为 Admission 的可配置规则链，LLM extraction 作为质量失败后的昂贵升级路径。这样既保留 Crawl4AI 的便利性，又不会把核心业务状态绑定在单个爬虫实例上。