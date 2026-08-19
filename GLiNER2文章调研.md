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
| 该研究把商品属性值提取改造成统一生成任务，让单模型从商品标题覆盖多类属性并适应开放世界值；适合对照 GLiNER2 单次多字段抽取在长尾属性、未见值和跨类目泛化上的表现。 | https://aclanthology.org/2021.ecnlp-1.2/ |
| AVEQA 把“属性名”作为问题，从商品上下文定位对应属性值，并显式处理无答案情况；适合参考 GLiNER2 对 brand、model、color 等动态字段的 Schema 描述、缺失字段判定和零样本泛化设计。 | https://research.google/pubs/learning-to-extract-attribute-value-from-product-via-question-answering-a-multi-task-approach/ |
| TXtract 面向约 4000 个商品类目的层级 taxonomy，用单模型提取类目特定属性值；适合设计 GLiNER2 的“类目约束→字段集合→抽取”机制，避免所有类目共用一个过大的 Schema。 | https://aclanthology.org/2020.acl-main.751/ |
| AdaTag 用属性 embedding 动态生成不同属性的解码器，在共享知识的同时保留字段特性；其思路很适合对照 GLiNER2 为 brand、model、规格等字段编写差异化描述并验证字段间迁移效果。 | https://aclanthology.org/2021.acl-long.362/ |
| Ask-and-Verify 先为指定商品属性召回多个候选 span，再做验证过滤，可扩展到数千属性；适合给 GLiNER2 增加候选校验层，重点降低型号、规格等相似字符串的漏抽和误抽。 | https://aclanthology.org/2022.emnlp-industry.9/ |
| 该方案把部分闭集商品属性改成极端多标签分类，并利用电商属性 taxonomy 做 label masking；适合与 GLiNER2 组合成“开放字段用抽取、稳定枚举字段用分类”的混合架构。 | https://aclanthology.org/2022.ecnlp-1.16/ |
| AVEN-GR 联合做商品查询中的 NER 与实体链接，把 brand、material、color 等抽取结果映射到商品图谱实体；适合补足 GLiNER2 只抽 span 后的品牌别名消歧、属性标准化与目录 ID 对齐。 | https://aclanthology.org/2023.acl-industry.14/ |
| GAVI 是面向品牌值识别的类目感知生成方案，强调商品类别对品牌判断的约束；适合用于 GLiNER2 品牌抽取的类目条件设计，尤其可帮助处理品牌词与普通词重名、跨类目歧义。 | https://aclanthology.org/2023.icnlsp-1.11/ |
| SANTA 专门解决电商文本属性值规范化，可把同义缩写和不同表述映射到统一 canonical value；适合接在 GLiNER2 抽取之后，对型号别名、系统版本、尺寸单位等字段做标准化。 | https://aclanthology.org/2021.ecnlp-1.12/ |
| 该论文让模型从商品文本一次生成一组属性-值对，并专门处理未见值、多值属性与 canonicalized values；适合对照 GLiNER2 结构化多字段输出及属性值归一化的一体化方案。 | https://aclanthology.org/2023.findings-acl.413/ |
| MixPAVE 聚焦新商品属性只有少量标注数据时的属性值抽取，通过混合 Prompt Tuning 提升 few-shot 能力；适合指导 GLiNER2 在新增类目、新增品牌/型号字段时的小样本评测和快速适配。 | https://aclanthology.org/2023.findings-acl.633/ |
| 该项目同时提供 BERT NER、商品图片 OCR 与类目专属词表，并用文本/图片结果交叉校验属性；适合拿来设计 GLiNER2 的“类目约束 Schema + OCR 补充 + brand/color/size 等字段一致性校验”落地链路。 | https://github.com/rishav771/Product-Attribute-Extraction |
| productner 给出了完整的“商品类目分类→类目特定属性 NER”流水线，并用 Amazon 商品数据训练品牌识别；可直接参考在调用 GLiNER2 前先路由商品类目，再加载对应 brand/model/规格字段集合，减少大 Schema 干扰。 | https://github.com/etano/productner |
| 该项目将印尼电商商品标题中的 16 类属性抽取建模为 BERT Token Classification，并包含数据预处理和训练代码；适合用作 GLiNER2 在多语言短标题、多属性 span 抽取上的基线与迁移评测。 | https://github.com/mhilmiasyrofi/product-attribute-extraction |
| 这个轻量电商 NER 模型直接抽取 PRODUCT、BRAND、PRICE、QUANTITY，示例里包含 Samsung Galaxy 型号、RAM 和存储容量；很适合做 GLiNER2 品牌/型号/容量等核心字段的快速对照基线和回归测试。 | https://huggingface.co/MuneebAbro/ecommerce-ner-model |
| AttriSage 用商品与属性关系图配合图神经网络做属性值抽取，重点提升复杂属性关联下的识别；适合在 GLiNER2 抽取后利用目录图关系校验品牌、型号与规格组合是否合理。 | https://aclanthology.org/2024.eacl-srw.8/ |
| CoMave 通过多尺度 Masking 和困难负样本对比学习区分高度相似的细粒度属性；适合针对 GLiNER2 容易混淆的型号、系列、容量、尺寸等近邻字段设计困难样本与专项评测。 | https://aclanthology.org/2023.findings-acl.373/ |
| 该研究联合预测商品属性类型并从文本中抽取属性值，同时引入商品图片信息；适合参考“先确定当前商品应有哪些字段，再抽具体值”的设计，并补足 GLiNER2 仅靠文本时的缺失属性。 | https://aclanthology.org/2020.emnlp-main.166/ |
| 这套 Walmart 生产方案结合 BERT、CRF、类目-属性关系纠错和 LLM 辅助标注，并进行了线上部署验证；适合借鉴 GLiNER2 的类目约束、合成/弱标注数据扩充及错误结果二次纠偏。 | https://arxiv.org/abs/2312.06684 |
| 该大规模多模态方案用单模型覆盖数千个“商品类型-属性”组合，支持零样本属性和值在文本缺失时的预测，并采用远程监督降低标注成本；适合参考 GLiNER2 大 Schema 分层、长尾属性扩展和缺失字段补充策略。 | https://arxiv.org/abs/2306.00379 |
| LLM-Ensemble 在 Walmart 商品属性值抽取中通过加权集成多个模型并已用于生产；可作为 GLiNER2 落地后的高价值字段复核层，把 GLiNER2 与其他抽取器/LLM 的结果做置信度融合，降低品牌、型号冲突。 | https://arxiv.org/abs/2403.00863 |
| 该研究系统比较生成式模型在 Amazon/MAVE 商品属性抽取上的显式值、隐式值与多语言能力；适合对照 GLiNER2 在“文本直接出现字段”和“需要语义推断字段”之间的能力边界，并设计少样本评测。 | https://aclanthology.org/2023.emnlp-industry.55/ |
| QueryNER 是 eBay 参与构建的电商查询分段模型，基于 Amazon ESCI 数据做 17 类 token classification；适合把 brand、product name、color、material、UoM 等作为 GLiNER2 字段抽取的查询侧基线和标签体系参考。 | https://huggingface.co/bltlab/queryner-bert-base-uncased |
| 这个繁体中文商品名称 NER 模型直接覆盖品牌、商品系列/名称、产品序号、颜色、材质、尺寸、重量、容量和功能规格等 16 类属性；非常适合拿来定义中文 GLiNER2 的品牌/型号/规格 Schema，并做逐字段回归对照。 | https://huggingface.co/clw8998/Product-Name-NER-model |
| Çarşı 提供土耳其语电商 NER 模型，直接抽取 PRODUCT、BRAND、PRICE、COLOR、SIZE、MATERIAL、GENDER，并给出 BERT NER 与零样本 LLM 的对照；适合验证 GLiNER2 在多语言商品短文本上的迁移和动态字段优势。 | https://huggingface.co/cihatyldz/carsi-bert-turkish-ecommerce-ner |
| AI-PAVE-Br 提供巴西葡萄牙语人工校验的商品分类与属性抽取 Golden Dataset，并按商品类型维护不同属性 Schema；适合评估 GLiNER2 的“类目→字段集合→抽取”方案以及跨语言、跨类目稳定性。 | https://github.com/ai-luizalabs/AI-PAVE-Br |
| 网易严选的线上电商 NER 会识别商品名、商品属性名和属性值，并对 BERT/BiLSTM-CRF 的准确率与在线推理时延做了实际对比；适合参考 GLiNER2 上线时的字段定义、下游知识图谱衔接和实时服务性能取舍。 | https://blog.tensorflow.org/2020/10/how-netease-yanxuan-uses-tensorflow-for-chatbots.html |
| EIVEN 专门处理文本或图片中隐含、易混淆的商品属性值，并强调降低多模态 LLM 推理成本；可作为 GLiNER2 文本 span 抽取之外的补充层，用来识别无法直接从标题字面截取的隐式规格与属性。 | https://aclanthology.org/2024.naacl-industry.40/ |
| MSIT 面向开放世界商品属性挖掘，通过多模态自纠错指令微调同时发现新的属性和值；适合在 GLiNER2 固定 Schema 覆盖不足时，用于“新属性发现→人工确认→Schema 扩充”的持续迭代流程。 | https://aclanthology.org/2025.acl-long.85/ |
| 该方法针对 QA 式商品属性抽取中的稀有和歧义字段，用训练集中的可能属性值扩展查询，并在 AliExpress 数据上显著提升宏平均 F1；适合给 GLiNER2 的 brand/model/规格字段描述补充候选值示例或词典上下文，强化长尾字段识别。 | https://aclanthology.org/2022.acl-short.25/ |
| Amacer 从少量高质量种子属性出发，既扩展已有属性类型的值，也自动发现新属性类型；适合用在 GLiNER2 固定 Schema 之外做“开放属性发现→人工审核→Schema 增量”的低标注成本流程。 | https://aclanthology.org/2023.acl-long.683/ |
| GAVEL 在 2000 个商品类目、1000 多种属性上做多语言生成式属性值抽取，并通过 LLM 扩充训练数据且进行了线上 A/B 验证；适合参考 GLiNER2 大规模类目/属性覆盖、东南亚多语言适配和合成数据增强。 | https://aclanthology.org/2025.knowledgenlp-1.6/ |
| ViOC-AG 只依赖商品图片做零样本属性值提取，并结合 OCR 与 LLM 对 OOD 属性值纠错；可作为 GLiNER2 文本品牌/型号/属性抽取的视觉补充，在标题缺失规格或包装上才出现型号时进行兜底。 | https://aclanthology.org/2025.naacl-industry.38/ |
| 该研究用零样本/少样本提示从商品标题和描述中直接抽取属性-值对，重点考察对未见属性和值的泛化；适合拿来与 GLiNER2 动态 Schema 的零样本抽取做基准对照，并参考缺失字段与 OOD 评测设计。 | https://arxiv.org/abs/2306.14921 |
| Smart-Shopper 的生产化 NER 流水线会先做品牌/商品词表归一化和模糊匹配，再抽取 brand、product、color、budget 等字段，示例能把拼写噪声下的 Samsung Galaxy A15 正确结构化；适合借鉴 GLiNER2 前后处理、品牌别名和型号纠错链路。 | https://github.com/AmineElAtrache/Smart-Shopper |
| NVIDIA Retail Catalog Enrichment 直接输出包含 brand、model_or_variant、颜色、材质及商品详情的富 JSON，并把结果映射到商品协议 Schema；适合参考 GLiNER2 的结构化字段设计、多模态补充、字段证据校验和商品目录入库接口。 | https://github.com/NVIDIA-AI-Blueprints/Retail-Catalog-Enrichment |
| Shopify 全球商品目录的多模态 LLM 方案会直接抽取并归一化 color、size、material、brand、model 等核心字段，并把属性绑定到类目 taxonomy；非常适合参考 GLiNER2 的“类目→Schema→多字段抽取→标准化”大规模生产链路。 | https://shopify.engineering/leveraging-multimodal-llms |
| Shopify Catalog 的商品聚类流程先从每条商品记录抽取严格 JSON 格式的 brand 和 model，再用 brand:model 组合归并同款并二次校验异常；适合把 GLiNER2 的品牌/型号结果接入商品去重、同款聚合与结果纠错。 | https://shopify.engineering/catalog-clustering |
| BEATS 从缺少属性体系的商品目录出发，用 LLM + 人工反馈迭代生成类目属性 taxonomy，再对单品做结构化属性标注，已覆盖 540 多万商品；适合参考 GLiNER2 的 Schema 冷启动、类目级字段维护和持续扩展机制。 | https://arxiv.org/abs/2606.04909 |
| 该研究同时评估商品属性“抽取+归一化”，覆盖名称扩展、泛化、单位转换和字符串规整等后处理；适合把 GLiNER2 的 span 抽取结果接入统一 canonical value 层，并作为端到端基准对照。 | https://arxiv.org/abs/2403.02130 |
| 该框架通过可控修改、负样本生成和属性删除来合成高质量商品数据，并在 MAVE 上验证可接近真实训练数据效果；适合给 GLiNER2 的品牌/型号/规格字段构造低资源训练集、困难负样本和缺失字段测试集。 | https://arxiv.org/abs/2601.04200 |
| 该工业方案自动迭代生成用于“商品类目-属性”质量检查的提示，在数万种字段组合和多语言场景中提升质检准确率；适合放在 GLiNER2 抽取之后做字段合法性、类目匹配和异常值复核层。 | https://aclanthology.org/2025.emnlp-industry.63/ |
| TACLR 把商品属性值识别改造成 taxonomy-aware 检索任务，能处理隐式值、OOD 值和标准化输出，并已在闲鱼每天处理数百万商品；适合作为 GLiNER2 抽取后的候选值检索、规范化和高吞吐校验层。 | https://github.com/SuYindu/TACLR |
| MICE 用多种图像描述模型为商品图片生成更可靠的文本，再做商品属性值抽取；适合在商品标题/描述缺失或不可信时，为 GLiNER2 提供图片转文本后的补充证据，增强颜色、款式、规格等字段召回。 | https://aclanthology.org/2025.acl-industry.80/ |
| PAE 面向电商时尚趋势 PDF，同时从文本和图片抽取属性并通过 BERT 表征与现有目录属性做匹配；适合参考 GLiNER2 在说明书、趋势报告等非标准商品文本中的属性抽取与目录映射流程。 | https://arxiv.org/abs/2405.17533 |
| FeedGen 以商品 feed 为输入，通过生成式 AI 补充和优化结构化商品属性，示例明确使用 Brand、Color、Size、Material 等字段；适合参考 GLiNER2 抽取结果进入 Merchant Feed 后的字段补全、命名规范和质量控制。 | https://github.com/google-marketing-solutions/feedgen |
| lightfeed/extractor 提供基于显式 Schema 的网页结构化抽取示例，电商场景直接定义 name、brand、price 等字段；适合参考把 GLiNER2 包装成可配置 Schema 的商品详情页抽取服务，并与网页采集链路衔接。 | https://github.com/lightfeed/extractor |
| esci-s 在 Amazon ESCI 商品数据上补充了更丰富的商品元数据，原始字段包含 product_title、product_description、product_brand、product_color 等；适合构造 GLiNER2 的品牌/颜色等字段回归集，并为后续型号、规格扩充提供真实商品语料。 | https://github.com/shuttie/esci-s |
| 该 EACL 工业论文专门研究电商场景 VLM 的大规模适配，并把 dynamic attribute extraction 纳入评测；适合用来设计 GLiNER2 在商品多图、噪声描述和动态字段场景下的多模态补充与评测体系。 | https://aclanthology.org/2026.eacl-industry.38/ |
| 该研究用 18 个商品属性把“字段是否适用/可见”和“字段值分类”拆开评估，并要求结构化 JSON 输出；适合借鉴 GLiNER2 对缺失/不适用属性的判定、避免硬抽不存在字段，以及逐字段诊断评测。 | https://arxiv.org/abs/2601.15711 |
| 该论文直接解决电商搜索中的品牌实体链接：先用 NER 识别品牌，再做别名/多语言表述匹配和实体消歧，并经过线上 A/B 验证；很适合接在 GLiNER2 品牌抽取后做品牌标准化、母子品牌和别名归一。 | https://arxiv.org/abs/2502.01555 |
| INSPIRE 会从商品标题和描述生成包含 brand、flavor 等显式/隐式结构化属性，并用 LLM 教师弱监督蒸馏到轻量模型；适合参考 GLiNER2 批量生成伪标注、低成本扩充品牌/规格字段训练数据及线上轻量化部署。 | https://arxiv.org/abs/2606.23889 |
| WebFormer 面向网页结构化信息抽取，电商示例目标字段就包含 product title、description、brand、price，并显式利用 DOM/布局信息；适合在 GLiNER2 之前先从复杂商品详情页定位候选文本，再做品牌、型号和属性细粒度抽取。 | https://arxiv.org/abs/2202.00217 |
| eCeLLM 的 ECInstruct 提供 11 万级真实电商指令数据，任务明确包含 Attribute Value Extraction 和 Product Matching，并覆盖未见商品/未见指令 OOD 测试；适合用于 GLiNER2 动态 Schema 的微调数据设计和跨商品泛化评测。 | https://github.com/ninglab/eCeLLM |
| EshopInstruct 含 63,972 条电商指令数据，其中 NER 类直接提供 4,000 条 Attribute Extraction 和 2,120 条 Attribute Value Extraction；适合补充 GLiNER2 商品属性抽取的训练/回归样本，并和商品匹配等下游任务联合评估。 | https://github.com/suyan-liang/EshopInstruct |
| ImplicitAVE 提供公开多模态隐式属性值数据和评测代码，可用来验证 GLiNER2 在商品文本未显式出现属性值时的能力边界，并筛选哪些字段需要图片或推理链路补充。 | https://github.com/HenryPengZou/ImplicitAVE |
| MADIAVE 用多代理辩论迭代校验隐式商品属性，适合给 GLiNER2 的低置信度、冲突品牌/型号/属性结果增加二次复核层，避免一次抽取直接入库。 | https://aclanthology.org/2026.findings-eacl.159/ |
| HyperPAVE 利用异构超图做零样本未见属性值识别，适合把 GLiNER2 与商品目录、关系数据结合，为长尾品牌、型号和规格提供候选增强，重点处理训练数据未覆盖的新值。 | https://arxiv.org/abs/2402.08802 |
| 该 PAVI 研究显示“先识别属性、再抽属性值”的两阶段零样本方案优于一步式；适合在商品类目 Schema 不完整时先发现候选字段，再交给 GLiNER2 做 brand/model/规格等 span 抽取。 | https://arxiv.org/abs/2409.12695 |
| 该多模态方案先判断当前商品适用的属性集合，再在缩小后的字段范围内抽值；直接对应 GLiNER2 按类目/商品动态裁剪 Schema 的做法，可降低无关字段误抽和大 Schema 干扰。 | https://arxiv.org/abs/2207.07278 |
| Catalog Attribute Normalizer 把多源商品标题、描述和已有属性统一到受控字段词表，并区分 extracted/canonicalized 来源；适合接在 GLiNER2 的 brand/model/size/color 等抽取后做值归一化、字段约束和 taxonomy 校验，避免脏值直接入库。 | https://github.com/ACJLabs/catalog-normalizer |
| JPAVE 同时预测商品包含哪些属性并生成或分类对应值，不依赖字符级 span 位置标注，且面向开放世界未见值与跨平台描述差异；适合参考 GLiNER2 的“字段存在性判断+值抽取”联合评测和弱标注数据利用。 | https://arxiv.org/abs/2311.04196 |
| KEAF 针对新商品、新属性只有少量样本的场景，利用属性描述和类目信息做多标签 few-shot 属性值抽取；适合用于 GLiNER2 新类目、新品牌/型号字段的低资源基线，并启发更有效的 Schema 字段描述。 | https://arxiv.org/abs/2308.08413 |
| AE-smnsMLC 只需要商品级属性值弱标注、不要求值在文本中的位置，并通过语义匹配与困难负标签采样区分相似属性值；适合为 GLiNER2 的型号、容量、尺寸等易混淆字段构造 hard negatives，降低人工 span 标注成本。 | https://arxiv.org/abs/2310.07137 |
| 该跨类目多任务属性抽取研究会自动学习不同商品类目间的属性相似度，并提升低资源字段效果；适合参考 GLiNER2 多类目 Schema 的共享/隔离策略，把相似品牌、材质、规格字段经验迁移到长尾类目。 | https://aclanthology.org/2021.ecnlp-1.10/ |
| SynthAVE 面向大规模电商属性值抽取，构建覆盖 229 个商品类型、792 个属性、4 种语言的人审基准，并用多 LLM 投票自动质检合成标签；适合为 GLiNER2 的品牌/型号/规格字段批量构造训练集、回归集和低成本质量校验闭环。 | https://arxiv.org/abs/2607.07469 |
| HPD 专门优化“同一商品一次抽多个属性值”的推理吞吐，利用属性值之间的条件独立性并行生成，最高可将 AVE 推理时间和成本降低 13.8 倍；适合为 GLiNER2 大 Schema 的 brand/model/规格批量抽取设计并发、缓存和压测基线。 | https://aclanthology.org/2026.findings-acl.1832/ |
| Instacart 的 PARSE 是已覆盖百万级商品的自助式多模态属性抽取平台，按属性配置名称/类型/定义/样例，结合文本、图片、置信度自验证和低置信度人工复核；其架构与 GLiNER2 动态 Schema 落地非常贴近，可直接参考版本化配置、质量门禁和成本分层。 | https://company.instacart.com/tech-innovation/scaling-catalog-attribute-extraction-with-multi-modal-llms |
| Amazon Catalog AI 的 CascadeAgent 在生产级目录中自动生成和维护 27,000+ 个“商品类型-属性”专属指令，并用失败样本持续改写属性定义、取值约束与 abstention 规则；非常适合迁移到 GLiNER2，自动优化 brand/model/规格等 Schema 描述并按字段独立迭代。 | https://sigir-ecom.github.io/eCom26Papers/paper_785.pdf |
| Amazon 的该方案面向千级商品类型和百余视觉属性，用 CLIP 集成自动生成合成标签，并通过按类置信度拒绝阈值把准确率控制在 90%+；适合给 GLiNER2 文本抽取补充颜色、材质、款式等视觉属性训练数据，并设计“低置信度不入库”质量门禁。 | https://sigir-ecom.github.io/eCom26Papers/paper_773.pdf |
| Amazon SIGIR 2026 的两阶段方案先从商品标题/描述抽取结构化属性，再按类目构建可复用属性图并用于检索排序；很适合参考 GLiNER2 的“类目 Schema→字段抽取→结构化目录/图谱→下游检索”生产链路。 | https://arxiv.org/abs/2604.27410 |
| 该电商研究直接从商品标题抽取关键属性并以深度序列标注模型达到高精度；适合把它作为 GLiNER2 在短标题 brand/model/规格等 span 抽取上的传统监督基线，用于衡量零样本/少样本方案收益。 | https://arxiv.org/abs/1803.11284 |
| eBay 的经典方案把商品属性抽取建模为 NER，并通过 bootstrapping 扩展训练数据；适合参考 GLiNER2 对长尾品牌、新型号、拼写变体等低资源实体的弱监督扩充与字段回归评测。 | https://aclanthology.org/D11-1144/ |
| 该 NLPCC 工作直接研究时尚商品属性值抽取并引入视觉 Prompt；适合补足 GLiNER2 纯文本抽取在颜色、材质、款式等视觉属性上的盲区，形成“文本 Schema 主抽取 + 图片证据补充/校验”的方案。 | https://doi.org/10.1007/978-981-95-3343-5_9 |
| 该 NLPCC 工作针对直播电商商品属性识别采用弱监督生成框架；适合参考从直播文本/口语描述构造低成本训练信号，再迁移到 GLiNER2 的品牌、型号、规格字段抽取，覆盖噪声大、标注稀缺的商品内容。 | https://doi.org/10.1007/978-981-95-3343-5_17 |
| 该研究系统比较文本填空与答案生成两类生成式商品属性值抽取框架，覆盖联合多属性抽取和未见值泛化；适合与 GLiNER2 的动态 Schema 多字段抽取做基准对照，判断 span 抽取与生成式补全各自边界。 | https://www.sciencedirect.com/science/article/pii/S0957417423033523 |
| 该项目/论文专门从复杂商品详情页中识别规格块，不局限于 table/list 标签，并进一步抽取属性-值对；适合作为 GLiNER2 的网页上游，把 DOM 中真正含型号、尺寸、材质等规格的文本块先筛出来再做字段抽取。 | https://arxiv.org/abs/2201.02896 |
| 该研究从商品规格表/列表中自动识别属性列和值列，并处理多列结构和重复模板，还做跨站 Schema 匹配；适合参考 GLiNER2 前置网页结构解析以及抽取后的字段名对齐与统一。 | https://doi.org/10.1145/3106426.3106449 |
| DEXTER 面向大规模 Web 商品规格发现与抽取，覆盖站点发现、商品页识别、规格区域定位和 wrapper 抽取；适合把 GLiNER2 嵌入批量商品采集链路，在结构化规则覆盖不足时负责品牌、型号和长尾属性语义抽取。 | https://doi.org/10.14778/2831360.2831372 |
| Web Data Commons 的商品语料与 Gold Standard 覆盖跨电商站点的商品数据抽取、属性值与 Schema 对齐，可作为 GLiNER2 从真实网页做品牌/型号/规格字段抽取时的外部评测与跨站泛化数据来源。 | https://webdatacommons.org/productcorpus/index.html |
| YODA 是面向 Google Feed 商品优化的生产 NER，直接覆盖 brand、color、size、energy label 等字段，并在大规模商品数据上训练与部署；很适合作为 GLiNER2 商品字段抽取的生产级精度/吞吐基线和字段体系参考。 | https://huggingface.co/lighthousefeed/yoda-ner |
| ecombert-ner-v1 是商品标题/查询专用 span NER，标签直接包含 BRAND、MODEL、MATERIAL、COLOR、MEASUREMENT、ATTRIBUTE、COMPATIBILITY 等 23 类；与 GLiNER2 的品牌、型号、规格 Schema 几乎一一对应，适合做重叠 span 与细粒度字段回归基线。 | https://huggingface.co/xinyacs/ecombert-ner-v1 |
| 该 B2B 电商 NER 项目抽取 PRODUCT、QUANTITY、SIZE、UNIT，并把结果接入品牌模糊匹配、变体识别、置信度和 SKU 映射；适合参考 GLiNER2 从“文本字段抽取”到“目录/SKU 标准化与业务入库”的完整后处理链路。 | https://huggingface.co/Purva17/b2b-ecomm-ner |
| Amazon 的 PAM 把商品文本、图片 OCR 文字和视觉对象统一进序列到序列模型，并用商品类目条件化属性值预测；适合给 GLiNER2 增加“文本主抽取 + 包装 OCR 补证 + 类目约束”链路，尤其补齐型号、容量、规格只出现在图片上的场景。 | https://www.amazon.science/publications/pam-understanding-product-images-in-cross-product-category-attribute-extraction |
| Rakuten-Ichiba 的多模态属性抽取方案从文本与图片联合补全 color、material 等缺失属性，并专门处理多模态融合中的 modality collapse，且已实际部署；适合借鉴 GLiNER2 文本抽取与视觉兜底的融合、训练和线上评估方式。 | https://arxiv.org/abs/2203.03441 |
| MetaBridge 专门校验电商目录中 brand、product name 等短文本属性是否可信，并面向少标注、多类目场景做跨类目泛化；适合放在 GLiNER2 抽取后做品牌、型号、规格字段的二次一致性校验和低置信度拦截。 | https://arxiv.org/abs/2006.08779 |
| 该 AI Agent 框架能从非结构化商品描述自动完成“属性本体创建/扩展→本体精炼→知识图谱填充”，不依赖预定义 Schema；适合补充 GLiNER2 固定字段之外的新属性发现，并形成可持续扩展的类目属性体系。 | https://arxiv.org/abs/2511.11017 |
| VARM 同时识别同款商品的变体关系，并从变体组中抽取“共同属性”和“变化属性”；适合把 GLiNER2 的 brand/model/color/size/容量等结果接到 SKU 变体归并与型号差异校验，减少把系列名、型号和变体规格混淆。 | https://arxiv.org/abs/2410.02779 |
| 该研究在 QA 式商品属性值抽取中显式融合“属性类型”特征，解决只给属性名时语义不足的问题；适合指导 GLiNER2 为 brand、model、capacity、material 等字段加入更清晰的类型/定义描述，提升相似字段和短标题场景的区分度。 | https://doi.org/10.1145/3578741.3578778 |
| Delivery Hero QC 的生产链路从供应商标题与图片一次抽取 Brand、Flavor、Volume 等 22 类字段，并结合置信度与人工复核；非常适合参考 GLiNER2 的类目 Schema、多字段结构化抽取和低置信度质量门禁。 | https://deliveryhero.jobs/blog/how-delivery-hero-uses-agentic-ai-for-building-a-product-knowledge-base/ |
| Amazon 的 WSDM 2026 方案把非结构化品牌知识库交给 LLM Agent，同时完成商品条目修复与匹配；适合接在 GLiNER2 品牌抽取后做品牌标准化、别名校验、品牌知识补全和错误纠正。 | https://www.amazon.science/publications/using-brand-knowledge-bases-and-llm-agents-to-enhance-e-commerce-retailers-catalog-quality |
| AttributeForge 用 43 个专用 LLM Agent 自动完成端到端商品 Schema 建模，并配套 Schema 质量评估与自动修复；适合为 GLiNER2 自动生成、审核和持续迭代不同类目的 brand/model/规格字段定义。 | https://www.amazon.science/publications/attributeforge-an-agentic-llm-framework-for-automated-product-schema-modeling |
| 该方案用语义检索与 LLM 做商品 Schema 匹配和重复属性检测，在 1399 个属性上达到较高匹配效果；适合在 GLiNER2 落地前合并 manufacturer/brand、model/model_number 等同义字段，避免 Schema 膨胀和重复抽取。 | https://www.amazon.science/publications/effective-product-schema-matching-and-duplicate-detection-with-large-language-models |
| Amazon CIKM 2025 的 Tool Use 方案让 LLM 调用商品 Schema 和目录信息来修复结构化商品数据，并显著提升完整度；适合给 GLiNER2 增加目录查询/字段约束工具，对品牌、型号和规格做抽取后验证与缺失补全。 | https://www.amazon.science/publications/using-large-language-models-to-improve-product-information-in-e-commerce-catalogs |
| Amazon 的目录质量实践会让 LLM 识别标准属性值、收集同义表达并检测错误值；很适合把 GLiNER2 输出接到 canonical value、别名字典和异常值检查层，减少品牌/型号/规格的脏值直接入库。 | https://www.amazon.science/blog/using-llms-to-improve-amazon-product-listings |
| QUEACO 把电商查询中的 NER 抽取和属性值归一化统一起来，并用大规模弱标注行为数据做 teacher-student 学习；适合参考 GLiNER2 的“span 抽取→canonical value”链路以及低成本长尾品牌/规格数据扩充。 | https://www.amazon.science/publications/queaco-borrowing-treasures-from-weakly-labeled-behavior-data-for-query-attribute-value-extraction |
| Home Depot 的 TripleLearn 生产 NER 直接识别 brand、product type 等电商实体，并通过多数据集迭代训练获得线上收益；适合用作 GLiNER2 品牌/商品类型抽取的生产级基线，并参考持续难例回流训练机制。 | https://ojs.aaai.org/index.php/AAAI/article/view/17773 |
| PAVE 通过聚合相似商品邻居来提升开放词表属性抽取召回，同时保持精度；适合把 GLiNER2 的低置信度或缺失 brand/model/规格交给相似商品候选层做补全与交叉校验。 | https://www.amazon.science/publications/pave-lazy-mdp-based-ensemble-to-improve-recall-of-product-attribute-extraction-models |
| KSelF 针对稀有和未见商品属性，利用大规模“商品档案-属性-值”预训练语料和查询扩展增强抽取，并提供 EC-AVE 基准；适合用于 GLiNER2 新类目、新字段和长尾型号的低资源训练与泛化评测。 | https://aclanthology.org/2023.findings-emnlp.542/ |
| AFMRL 把商品细粒度理解直接建模为属性生成，先由多模态模型从商品图片和文本抽取关键属性，再用这些属性增强同款/相似商品表征；适合验证 GLiNER2 抽出的品牌、型号、规格能否作为商品匹配与检索的结构化特征。 | https://aclanthology.org/2026.findings-acl.704/ |
| 该 ACL 2026 工业论文把商品重量这类隐式/缺失属性做成“类目内样例检索 + 多模态推断”，并已在百万级商品线上运行；适合补充 GLiNER2 对文本中未直接出现的重量、尺寸等数值属性的兜底思路和类目条件化评测。 | https://aclanthology.org/2026.acl-industry.41/ |
| 该研究在 3000 个真实网店食品页上用 Schema 约束抽取配料、营养表等商品字段，并比较直接抽取与“生成函数后批量解析”两种方案；适合参考 GLiNER2 从商品页批量抽品牌/型号/规格时的结构约束、成本与吞吐权衡。 | https://arxiv.org/abs/2506.21585 |
| IndustryBench-MIPU 提供 4481 个工业商品、4554 个属性名和多图结构化属性基准，输入就是“商品图片 + 单品属性 Schema”，示例字段直接包含品牌、型号；非常适合给 GLiNER2 的动态 Schema、品牌/型号/规格抽取做多图补证和回归评测。 | https://github.com/alibaba-multimodal-industrial-ai/IndustryBench-MIPU |
| Enthusiast 是开源电商 Agent 框架，内置 Product Catalog Enrichment 场景，可从非结构化 product sheets 抽取商品描述和属性，并带验证/评估组件；适合参考把 GLiNER2 接进商品资料批处理、字段校验和目录富化工作流。 | https://github.com/upsidelab/enthusiast |
| 该研究在真实电商数据上公平比较 BERT-NER 与 QA 式多属性抽取，发现 NER 在准确率上可与 QA 相当且推理更快；这对 GLiNER2 很关键，可作为“单次 span 抽取多个 brand/model/规格字段”相对逐属性问答方案的性能与架构依据。 | https://aclanthology.org/2023.emnlp-industry.16/ |
| ASTRA 专门解决多卖家/制造商商品 Schema 到平台统一 Schema 的映射，并支持仅少量样本快速接入新属性；适合接在 GLiNER2 抽取之后，把 supplier 的 manufacturer/model no./capacity 等异构字段统一到 canonical brand/model/规格键。 | https://aclanthology.org/2024.emnlp-industry.92/ |
| GSID 针对手工类目与属性体系覆盖不了长尾商品的问题，从非结构化商品元数据学习并生成结构化语义表示，且已在真实电商平台部署；适合评估 GLiNER2 固定 Schema 在长尾商品上的盲区，并指导后续动态字段发现与结构化表示设计。 | https://aclanthology.org/2025.emnlp-industry.78/ |
| AWS 的 AI-Powered Product Catalog 示例把商品图片进入 Product Attribution Lambda，自动抽取产品 features/attributes 并持久化到 DynamoDB；适合直接参考 GLiNER2 商品字段抽取服务在异步工作流、存储和商品入库链路中的工程位置。 | https://github.com/aws-samples/sample-ai-powered-product-catalog |
| Google Cloud AgentSmithy 内置 Catalog Enrichment Agent，目标就是处理原始供应商数据并补齐商品目录缺失信息；适合参考把 GLiNER2 作为确定性 brand/model/规格抽取器嵌入 Agent 工具链，再由 Agent 负责搜索、补全和写入目录。 | https://github.com/GoogleCloudPlatform/agentsmithy/blob/main/agent_bar_v2/subagents/retail/intelligent_inventory_manager/sub_agents/catalog_enrichment/agent.py |
| Tandemn 的零售批量推理案例从脏商品目录出发，明确讨论 catalog enrichment、attribute extraction 与大批量推理基础设施；适合参考 GLiNER2 在百万级商品离线批处理时的任务切分、吞吐、重试和成本治理。 | https://www.tandemn.com/blog/1 |
| GLiNER2 官方论文系统说明了 Schema 驱动的实体、分类与层级结构化抽取能力，并强调单一紧凑模型和 CPU 友好部署；适合把 brand、model、SKU、规格等定义成动态 Schema，作为商品字段抽取服务的核心方法依据。 | https://aclanthology.org/2025.emnlp-demos.10/ |
| revizor 是专门拆解商品标题的 NER 项目，直接输出 type、brand、model、vendor_code，并给出品牌与型号的逐类效果；和 GLiNER2 目标字段几乎一一对应，可作为品牌/型号/货号抽取的监督基线及自动构造训练数据的参考。 | https://github.com/bureaucratic-labs/revizor |
| Amazon Catalog Team 的生产实践会从商品提交中抽取 dimensions、materials、compatibility、technical specifications，并用多小模型共识、强模型监督和知识库回流持续降低错误；适合给 GLiNER2 增加低置信度复核、难例沉淀与持续优化闭环。 | https://aws.amazon.com/blogs/machine-learning/how-the-amazon-com-catalog-team-built-self-learning-generative-ai-at-scale-with-amazon-bedrock/ |
| 该论文直接从商品标题联合识别属性及其值，示例把 Seiko、SUJ708、Gold 分别对应 Brand、Model number、Band color；与 GLiNER2 一次抽取 brand/model/color 等字段非常贴合，适合构建联合字段回归集并校验型号 span 边界。 | https://arxiv.org/abs/2208.07130 |
| DiffXtract 不要求只抽预先给定的值，而是联合识别区分商品变体的属性类型及对应值；适合在 GLiNER2 固定 Schema 之外发现 finish、pack size 等变体字段，并验证“动态属性发现→Schema 增补→再抽取”的流程。 | https://www.amazon.science/publications/diffxtract-joint-discriminative-product-attribute-value-extraction |
| Home Depot 的商品变体识别方案会从非结构化标题提取商品家族信息，并利用相似型号进行变体归组；适合把 GLiNER2 的 brand/model/color/pack size 结果接到 SKU 变体聚类、型号一致性校验和误抽纠偏。 | https://arxiv.org/abs/2104.05504 |
| Amazon 的半监督视觉属性抽取使用未标注商品图片增强 ViT，在减少标注数据的同时提升属性提取覆盖；适合作为 GLiNER2 文本抽取的视觉兜底，补充标题里缺失的颜色、材质、款式等可见属性。 | https://www.amazon.science/publications/semi-supervised-learning-and-visual-transformers-for-product-attribute-extraction-from-e-commerce-images |
| Mercado Libre 在数亿级商品数据清洗中专门处理品牌拼写错误、属性不一致和跨语言值，并让 GenAI 返回标准化 JSON；非常适合接在 GLiNER2 后面做品牌别名、拼写纠错、属性值翻译与 canonicalization。 | https://medium.com/mercadolibre-tech/genai-meets-crisp-dm-advancing-data-science-for-e-commerce-a9d6d98a9142 |
| Mercado Libre 的商品迁移案例覆盖类目预测、类目专属必填属性、属性映射与批量 Listing；适合参考 GLiNER2 的“类目路由→动态 Schema→字段抽取→平台字段映射/校验”生产编排。 | https://medium.com/mercadolibre-tech/boosting-store-integration-mcp-and-agentic-ides-for-mercado-libre-listings-6bf616a914f2 |
| eBay 的电商 NER 研究直接面向品牌、颜色、材质、尺码等商品实体，并针对短标题与领域表达做表示学习；适合用作 GLiNER2 商品标题 span 抽取的传统基线和领域难例设计参考。 | https://aclanthology.org/W15-1522/ |
| Structuring E-Commerce Inventory 从非结构化商品描述中无监督发现并抽取属性名-值对，再做属性名同义归并；适合补充 GLiNER2 固定 Schema 之外的“属性发现→字段归并→Schema 扩充”流程。 | https://aclanthology.org/P12-1085/ |
| 该研究从用户评论中抽取商品属性，并利用 Wikipedia 构建跨领域模型；适合把 GLiNER2 的字段抽取从标题/详情页扩展到评论等长文本，并参考外部知识增强长尾属性识别。 | https://aclanthology.org/I11-1163/ |
| 这项经典工作直接从商品文本描述抽取属性-值对，同时处理显式与隐式属性，并用半监督学习降低标注成本；适合参考 GLiNER2 的低资源数据构造与隐式字段补充边界。 | https://doi.org/10.1145/1147234.1147241 |
| 该 EMNLP 工业方案用“网页爬取→商品页轻量分类→商品信息抽取”的模块化流水线显著减少无效页面与推理成本；适合放在 GLiNER2 前面做商品页筛选，只对高价值页面执行品牌、型号、属性抽取。 | https://aclanthology.org/2024.emnlp-industry.106/ |
| Walmart 的商品目录实践把属性抽取与独立质量检查拆成两步，并按字段精度阈值决定是否入库；很适合给 GLiNER2 增加抽取后 QC、低置信度拦截和人工校验闭环。 | https://tech.walmart.com/content/walmart-global-tech/en_us/blog/post/using-llms-to-manage-product-catalogs.html |
| 该阿尔及利亚阿拉伯语电商 NER 模型直接覆盖 BRAND、PRODUCT、COLOR、SIZE、QUANTITY、ATTRIBUTE 等实体；适合用作 GLiNER2 多语言/方言商品短文本的迁移基线，并验证动态 Schema 对品牌与属性字段的鲁棒性。 | https://huggingface.co/haninebou/algerian-ner-ultimate |
| browser-act 的 Amazon Product Detail Skill 可直接从商品页抽取 brand、model、颜色、重量、技术规格、变体属性等 100+ 字段；适合参考 GLiNER2 与网页采集层衔接后的字段 Schema、证据来源和结构化入库形式。 | https://github.com/browser-act/skills/blob/main/solutions/ecommerce/amazon-product-detail/SKILL.md |
| 该 Gemma 1B 商品信息抽取模型专门把噪声商品描述转成严格 JSON，直接输出 brand、product、keywords、quantity，并做单位归一化；适合作为 GLiNER2 轻量化商品字段抽取、结构化输出约束和归一化后处理的对照基线。 | https://huggingface.co/Dinesh-Kumar/gemma3-1b-finetuned-v3 |
| 这个中文电商 BERT Token Classification 模型面向 Chinese e-commerce NER，可直接作为 GLiNER2 中文商品标题/描述字段抽取的监督基线，并用于比较动态 Schema 与固定标签体系在品牌、型号和属性字段上的差异。 | https://huggingface.co/jinchenliuljc/ecom_ner_model |
| CatalogAgent 围绕商品目录结构化属性补全构建“生成器→校验器→Supervisor→记忆/提示优化”闭环，并直接处理 model name、model number、material 等字段的误判；很适合给 GLiNER2 品牌/型号/规格抽取增加低置信度复核、难例沉淀和 Schema 描述持续优化。 | https://arxiv.org/abs/2607.14396 |
| SAGE 面向十亿级商品目录，把不同语言、商品类型和目标属性统一做结构化属性值生成，并显式支持未见值、隐式值、不可适用与不可获得判定；适合对照 GLiNER2 在大规模动态 Schema 下的长尾品牌/型号、缺失字段 abstain 和多语言泛化。 | https://arxiv.org/abs/2309.05920 |
| Pinterest 的生产级网页信息抽取系统把 DOM 结构、视觉布局和文本压缩成统一网页表示，并以超高吞吐抽取电商结构化属性；适合放在 GLiNER2 前面做商品页候选字段定位，减少无关文本后再抽 brand/model/规格。 | https://arxiv.org/abs/2508.01096 |
| 该 2026 电商查询 NER 研究构建 17 类细粒度标签，覆盖 brand、color、size、fabric、feature、product_code 等，并系统比较 Transformer 与零样本 LLM；适合用来验证 GLiNER2 多语言短查询中的品牌/规格/货号抽取鲁棒性和噪声表现。 | https://www.sciencedirect.com/science/article/pii/S0950705126009391 |
| Catalog Phrase Grounding 将商品 title/brand 与图片中的商品区域、品牌 Logo 区域做对齐，并在生产品牌匹配任务上提升召回；适合把 GLiNER2 抽出的品牌字段与视觉 Logo 证据交叉校验，降低品牌误抽和包装文字干扰。 | https://arxiv.org/abs/2308.16354 |
| IPL 是闲鱼已上线的多模态商品 Listing 系统，围绕 category、brand、color、condition 等商品属性做领域微调与 RAG，并有真实生产采用数据；适合反向参考 GLiNER2 的“图片/文本→品牌与属性 Schema→结构化结果→商品发布”生产链路和多模态兜底。 | https://aclanthology.org/2024.emnlp-industry.52/ |
| 该研究把 PAVI 统一成属性-值生成任务，并在三个数据集系统比较多种生成策略，端到端方式兼顾效果和推理效率；适合与 GLiNER2 单次动态 Schema 多字段抽取做直接基准，比较 span 抽取与属性-值生成在精度、成本和长尾泛化上的差异。 | https://arxiv.org/abs/2407.01137 |
| QPAVE 把细粒度商品属性作为问题，从包含多个粗粒度信息的商品 Profile 中定位对应值，并引入类目感知的多任务学习；适合参考 GLiNER2 的“类目→字段 Schema→细粒度 value span”设计，尤其用于把复合规格拆成独立字段。 | https://doi.org/10.1007/978-3-031-68323-7_28 |
| CAVE 专门解决商品目录中错误属性值的纠正与补全，联合商品标题和现有属性表通过 QA 校正错误值，并能从标题补出新值；适合作为 GLiNER2 抽取后的品牌、型号、规格冲突校验和纠错层。 | https://doi.org/10.1145/3511808.3557161 |
| Home Depot 的该工作针对供应商入驻时手工录入属性值带来的噪声与不一致，研究可扩展的 Transformer 属性值验证与纠错；适合给 GLiNER2 结果增加“抽取→目录值验证→自动纠错/拦截”的质量门禁。 | https://isir-ecom2022.github.io/papers/isir-ecom-2022_paper_7.pdf |
| Beauty Beyond Words 在真实护肤商品目录中根据成分信息抽取垂直类目的专属属性，并强调可解释、鲁棒和可持续增加新属性；适合验证 GLiNER2 按类目维护专属 Schema，提取成分、功效、适用类型等长尾属性的能力。 | https://arxiv.org/abs/2409.13628 |
| Amazon 的 TARS 少样本方案直接从非结构化商品描述推断结构化属性值，仅用约 40 条/属性的标注即可获得接近大样本基线的效果，并可利用搜索查询生成合成训练数据；适合 GLiNER2 在新增 brand/model/规格字段时做低标注快速适配与跨字段迁移。 | https://www.amazon.science/publications/enhancement-and-analysis-of-tars-few-shot-learning-model-for-product-attribute-extraction-from-unstructured-texts |
| 该弱监督信息抽取方案用原型表示自动过滤噪声标注，并在电商属性值抽取上提升下游精度；适合用于清洗词典、规则或商品目录自动生成的 GLiNER2 品牌/型号/属性伪标注，降低脏标签对微调效果的影响。 | https://www.amazon.science/publications/prototype-representations-for-training-data-filtering-in-weakly-supervised-information-extraction |
| 该数据质量框架通过 QA 式属性值抽取识别商品描述缺失的关键信息，并产出类目级缺失字段报告；可参考在 GLiNER2 抽取之后增加“字段是否缺失/描述是否完整”的质量检查，避免必填品牌、型号或规格缺失时直接入库。 | https://aclanthology.org/2022.ecnlp-1.4/ |
| Amazon 的大规模商品 Schema 匹配方案面向海量制造商/供应商异构字段，用属性语义相似度和业务相关性自动对齐统一目录字段；适合衔接 GLiNER2，将 manufacturer、model no.、capacity 等来源字段归并到 canonical brand/model/规格 Schema，减少重复字段与人工映射。 | https://www.amazon.science/publications/attribute-similarity-and-relevance-based-product-schema-matching-for-targeted-catalog-enrichment |
| AutoKnow 是已覆盖 11K+ 商品类型的自动商品知识采集系统，包含商品属性识别、知识抽取、异常检测和同义词发现；适合参考 GLiNER2 的“类目/属性体系→字段抽取→别名归并→异常校验→商品知识库”端到端生产架构。 | https://www.amazon.science/publications/autoknow-self-driving-knowledge-collection-for-products-of-thousands-of-types |
| 该电商 NER 工作用正-未标注学习、种子词典和迭代扩展在低资源商品描述上训练实体识别；适合为 GLiNER2 的长尾品牌、新型号、货号等字段构造低成本训练数据，并利用未标注商品语料持续扩充实体覆盖。 | https://aclanthology.org/2020.ecnlp-1.1/ |
| LINE Shopping TW 在 2000 万级商品规模上用 LLM 做属性抽取，明确拆分 Brand、Model Number、Series Name，并给出型号严格排除项、品牌特定格式、后处理、批处理和评测策略；非常适合直接参考 GLiNER2 的型号边界定义、字段 Schema 与生产质量门禁。 | https://speakerdeck.com/lycorptech_jp/ai-frontiers-revealed-transforming-line-shopping-tw-with-llm-driven-product-attribute-extraction |
| Stanford CS229 项目基于 BestBuy 电商 NER 数据，从商品描述中抽取 Brand、ModelName、ScreenSize、Storage、RAM 等字段，并比较 SVM、GBT、CRF；适合做 GLiNER2 品牌/型号/规格 span 抽取的传统监督基线与逐字段误差对照。 | https://cs229.stanford.edu/proj2018/report/190.pdf |
| NER.ProductAttributeExtraction 项目直接面向商品标题属性值抽取，包含 SVM/GBT 多分类、CRF 序列标注、标题清洗以及 Doccano 标注转换；适合参考 GLiNER2 训练/评测数据清洗、span 标注转换和传统模型对照流程。 | https://github.com/green-leo/NER.ProductAttributeExtraction |
| Amazon ML Challenge 项目从商品图片中用 PaddleOCR、规则和线段检测抽取高度、宽度、重量、功率、电压、容量等实体值；适合在 GLiNER2 文本抽取之外增加图片 OCR 兜底，补齐包装或图片里才出现的规格字段。 | https://github.com/nirvan840/Product-Image-Attribute-and-Entity-Extraction |
| Lasso 的供应商商品标题治理指南把 brand/model extraction 与标准化模板、归一化规则结合，专门处理不同供应商命名不一致；适合用作 GLiNER2 前后的标题清洗、品牌/型号规范化和目录一致性设计参考。 | https://productlasso.com/en/blog/fix-inconsistent-product-titles |
| DataHunk 的生产商品数据抽取方案明确输出 title、brand、model number、variant、technical specs，并提供统一 Schema 与置信度评分；适合参考 GLiNER2 落地到 PIM/ERP 时的字段契约、置信度门禁和批量目录富化形态。 | https://datahunk.in/product-data-extraction |
| LATEX-Numeric 专门从商品描述中抽取尺寸、容量、功率等数值属性，利用远程监督、缺失标注鲁棒训练和自动单位别名生成实现可扩展抽取；适合给 GLiNER2 的数值规格字段补充单位词典、弱标注训练和归一化策略，尤其处理“数值+单位”边界。 | https://aclanthology.org/2021.naacl-industry.34/ |
| VideoAVE 提供公开的商品视频到属性值抽取数据集，覆盖 14 个领域和 172 个属性，并同时评测指定属性取值与开放属性-值对抽取；适合把 GLiNER2 的文本主抽取扩展到商品视频字幕、OCR 或帧描述证据，并评估视频中才出现的型号、规格和功能属性。 | https://arxiv.org/abs/2508.11801 |
| 该 2026 中文电商研究使用细粒度电商 NER 从商品标题识别 category、brand、origin 等实体并回灌领域词表与本体；适合参考 GLiNER2 中文品牌与类目字段的 Schema 设计、领域词典增强和标题预处理，提升中文长尾品牌及领域术语覆盖。 | https://www.nature.com/articles/s41598-026-38214-2 |
| D-Extract 从商品图片中的 OCR 文字抽取长宽高等尺寸属性，并结合商品类目统计先验提升识别效果；适合作为 GLiNER2 文本抽取的视觉兜底，补齐包装图或规格图中才出现的数值规格，并用于多证据一致性校验。 | https://openaccess.thecvf.com/content/WACV2023/html/Ghosh_D-Extract_Extracting_Dimensional_Attributes_From_Product_Images_WACV_2023_paper.html |
| 该 SIGIR 工业工作将电商查询中的 brand、color、product type 等区分为显式属性与隐式属性，并结合商品知识图谱和用户行为补足隐含值；适合明确 GLiNER2 可证据化抽取的边界，把隐式属性推断放到独立二阶段，避免推断值与原文抽取值混淆。 | https://www.amazon.science/publications/implicit-query-parsing-for-product-search |
| Amazon Berkeley Objects (ABO) 提供 14.7 万+真实商品的多语言标题/描述与结构化元数据，字段直接包含 brand、model_name、model_number、color、material、dimensions 等；可把目录元数据作为参考标签，批量评测 GLiNER2 从商品文本回抽品牌、型号和规格的准确率与跨语言泛化。 | https://amazon-berkeley-objects.s3.us-east-1.amazonaws.com/index.html |
| Best Buy E-Commerce NER Dataset 是人工标注的真实商品搜索查询，标签直接覆盖 Brand、ModelName、ScreenSize、RAM、Storage、Price 等；几乎可原样映射到 GLiNER2 动态 Schema，用于品牌/型号/核心规格 span 抽取的回归评测和少样本微调。 | https://www.kaggle.com/datasets/dataturks/best-buy-ecommerce-ner-dataset/data |
| Shopify The Catalogue 含 4.8 万+真实商品的 title、description、ground_truth_brand 和层级类目；可用于评测 GLiNER2 品牌抽取及“类目→字段 Schema”路由，尤其适合真实目录噪声和跨类目泛化测试。 | https://huggingface.co/datasets/Shopify/product-catalogue |
| Bright Data 的 Home Depot 样例数据把 product_name 与 model_number、manufacturer、SKU、dimensions 等结构化字段放在同一记录中；可直接生成“标题→品牌/制造商、型号、货号”对照样本，专项测试 GLiNER2 对字母数字混合型号和 SKU 边界的抽取稳定性。 | https://github.com/luminati-io/Home-Depot-dataset-sample |
| Shopify Product Taxonomy 是开源商品分类与属性标准，覆盖 25+ 垂直领域并维护 categories、attributes、values 及跨 taxonomy 映射；适合给 GLiNER2 按类目动态生成字段 Schema，并对抽取后的属性名和值做标准化与约束校验。 | https://github.com/Shopify/product-taxonomy |
| 该研究针对商品信息抽取提出“PLM 一次预测 + LLM 二次校验”的两阶段验证，并专门提升弱表达、低显著度字段的识别，同时支持本地部署；适合给 GLiNER2 的品牌、型号、规格等低置信度结果增加轻量复核层，减少漏抽和误抽。 | https://arxiv.org/abs/2607.26780 |
