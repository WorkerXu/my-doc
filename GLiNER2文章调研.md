| 推荐理由 | 链接地址 |
|---|---|
| GLiNER2 官方组合 Schema 教程直接给出电商商品 Listing 示例，把 brand、model、color、size 作为实体，同时抽取 price、features、category 等结构化字段，和“品牌/型号/属性”落地目标高度一致，可直接参考 Schema 设计。 | https://github.com/fastino-ai/GLiNER2/blob/main/tutorial/4-combined.md |
| 该论文面向真实电商搜索场景，用 Transformer NER 抽取大规模属性值，并配套两阶段归一化后进入在线检索与排序系统；适合借鉴 GLiNER2 后处理、标准化词典和线上服务链路。 | https://aclanthology.org/2024.ecnlp-1.13/ |
| GenToC 专门解决商品标题/搜索词中的属性-值识别以及标注不完整问题，并已在 IndiaMART 生产环境落地；适合参考弱标注扩充、困难样本补标以及 GLiNER2 微调数据构建。 | https://arxiv.org/abs/2405.10918 |
| OpenTag 将“属性名”作为查询来抽取商品标题中的属性值，可扩展到数千属性，和 GLiNER2 通过动态 Schema 指定 brand/model/材质/规格等字段的思路非常接近，适合参考大属性体系设计。 | https://aclanthology.org/P19-1514/ |
| OpenBrand 聚焦商品描述中的品牌值抽取和新品牌发现，并提供开放世界品牌抽取数据集；可用于验证 GLiNER2 对未见品牌、长尾品牌和品牌别名的泛化能力。 | https://aclanthology.org/2022.ecnlp-1.19/ |
| OA-Mine 是开放世界商品属性挖掘项目，能从商品标题中同时发现新的属性类型和属性值；适合补充 GLiNER2 固定 Schema 之外的“属性发现→Schema 扩充”流程。 | https://github.com/xinyangz/oamine |
| MAVE 提供 220 多万 Amazon 商品、约 300 万属性值标注和字符级 span，可直接用于构建/评估 brand、model、材质、尺寸等商品属性抽取训练集与基准集。 | https://github.com/google-research-datasets/mave |
| WDC-PAVE 同时覆盖商品属性值抽取与归一化，包含多站点真实商品及标准化后的属性值，适合评估 GLiNER2 抽取后对品牌、规格、容量、颜色等字段做统一标准化的效果。 | https://github.com/wbsg-uni-mannheim/wdc-pave |
| EcomGPT 的 EcomInstruct 中包含中文 MEPAVE 属性值识别/检测和商品标题属性匹配任务，适合拿来补充中文电商商品文本的评测样本，并验证 GLiNER2 中文品牌、型号和属性字段抽取。 | https://github.com/Alibaba-NLP/EcomGPT |
| ExtractGPT 直接用商品标题和显式 Schema 抽取 Brand、Color、Material 等字段并输出 JSON，和 GLiNER2 的动态 Schema/结构化抽取方式非常接近，适合对照提示词、字段定义、缺失值处理和评测方式。 | https://github.com/wbsg-uni-mannheim/ExtractGPT |
| 该项目专门比较零样本、少样本、Schema-Guided、字段定义增强等商品属性抽取策略，目标字段包含 brand、color、size、material；很适合用来设计 GLiNER2 商品字段抽取的基准实验和失败案例分析。 | https://github.com/a-dwivedi/llm-product-attribute-extraction |
| eBay 的 Text-to-Text 方案从商品 Listing 预测 brand、size、size type、color 等属性和值，并在生产系统上验证收益；适合参考 GLiNER2 在多属性体系下的字段选择、缺失属性补全和线上评估指标。 | https://aclanthology.org/2022.ecnlp-1.12/ |
| SMARTAVE 面向商品标题、描述、规格和图片等多模态信息做属性值抽取；可作为 GLiNER2 文本抽取后的扩展参考，尤其适合处理型号、颜色、尺寸等只在规格或图片文字中出现的字段。 | https://aclanthology.org/2022.findings-emnlp.20/ |
| 该工业论文将商品属性抽取建模为生成式问答，支持零样本属性值和文本中缺失的属性值，并已覆盖数千个“商品类型-属性”组合；适合借鉴 GLiNER2 大规模 Schema 管理、长尾属性和召回增强方案。 | https://aclanthology.org/2023.acl-industry.29/ |
| MVP-RAG 针对工业商品属性值识别中的 OOD 属性值和泛化问题，把商品标题/描述与属性值检索、生成结合；适合为 GLiNER2 增加品牌/型号候选库检索与抽取结果校验，提升长尾字段稳定性。 | https://aclanthology.org/2025.emnlp-industry.147/ |
| Walmart 场景论文直接研究商品标题中的属性抽取，并以 brand 为重点展示序列标注与归一化方案；适合参考 GLiNER2 对品牌、型号等短文本字段的 span 抽取、标准化和线上商品目录处理。 | https://arxiv.org/abs/1608.04670 |
| Trendyol 的电商属性抽取研究同时覆盖显式与隐式属性，并基于真实商品目录比较 Transformer 与 LLM；适合用来评估 GLiNER2 在明确品牌/型号字段之外，对材质、风格等隐式属性的可提取边界。 | https://www.mdpi.com/2079-9292/14/10/1930 |
| AutoPKG 会按需归纳商品类型和类型级属性键，并从文本/图片抽取属性值后做统一规范化；很适合参考 GLiNER2 的动态 Schema 自动扩充、字段归一化和商品知识图谱衔接。 | https://aclanthology.org/2026.findings-acl.766/ |
| 该生产级方案用参数高效微调持续生成结构化商品属性，并按“预测与目录字段的完整度差距”筛选高价值样本；适合设计 GLiNER2 的错误样本挖掘、持续微调和低成本线上迭代。 | https://aclanthology.org/2026.acl-industry.40/ |
| PatternRAG 根据商品类型、文本相似度和品牌关系检索目录中的相似商品来补全缺失属性；可作为 GLiNER2 抽取后的候选召回、品牌/型号纠错和缺失字段补全层。 | https://aclanthology.org/2025.emnlp-industry.18/ |
| 该研究针对商品类别和属性持续增长的问题，将商品类型与属性建模解耦并减少灾难性遗忘；适合参考 GLiNER2 在大量类目、不断新增字段场景下的 Schema 分层和持续学习策略。 | https://aclanthology.org/2024.findings-acl.510/ |
| EAVE 专门优化“同一商品需要抽取很多属性”时的推理效率，通过复用商品上下文表示减少重复计算；适合用作 GLiNER2 大 Schema、多字段批量抽取时的性能设计和压测对照。 | https://aclanthology.org/2024.findings-emnlp.80/ |
| SelfRefinement4ExtractGPT 直接对 Brand 等商品字段做 JSON 抽取，并利用错误样本自动改写属性定义与自纠错；非常适合迁移到 GLiNER2 的实体描述/Schema 定义优化和失败样本闭环。 | https://github.com/wbsg-uni-mannheim/SelfRefinement4ExtractGPT |
| AWS Smart Product Onboarding 是完整商品入库项目，先做商品分类，再依据类目专属 Schema 一次抽取多个属性，并支持类目树变化；适合参考 GLiNER2 的“类目→Schema→字段抽取”生产架构。 | https://github.com/aws-samples/aws-smart-product-onboarding |
| brand-ner 项目专门从 Amazon 商品标题中识别品牌，包含清洗后的数据、训练流程和 spaCy/Flair 基线；可作为 GLiNER2 品牌字段的独立测试集、规则对照和长短标题误差分析参考。 | https://github.com/annis-souames/brand-ner |
