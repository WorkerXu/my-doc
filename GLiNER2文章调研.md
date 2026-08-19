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
| 该中文商品名细粒度抽取项目直接定义 brand、model、series、category、size、material、color 等字段，并给出真实商品名逐字段样例；非常适合拿来校准 GLiNER2 中文 Schema 的字段边界，尤其区分品牌、系列、款式和商品类目。 | https://github.com/hetonghui02/Fine-Granularity-extraction-of-product-name-information |
| NER-Multilingual-Product 从 119 个供应方、67 万条英芬混合商品标题/描述中抽取 BRAND、SIZE、COLOR、VOLUME、WEIGHT 等字段，并把 NER 与正则组合；适合参考 GLiNER2 对品牌等语义字段与尺寸/单位等规则字段采用混合抽取和后处理。 | https://github.com/YuTian8328/NER-Multilingual-Product |
| 该 Flipkart 电子商品 NER 项目直接覆盖 Brand、Model、Processor、OS、RAM、Disk、Dimension 等字段，并从结构化商品特征自动构造 span 训练数据；适合参考 GLiNER2 品牌/型号/规格伪标注生成和传统监督基线。 | https://github.com/mahathi-p/Classification_of_Products_using_NER |
| AIWebscraper 从电子商品网页直接抽取 Brand、Model、Price、Availability、Condition、Category 等实体；适合参考 GLiNER2 与网页抓取层衔接后的字段 Schema，以及从商品详情页文本到结构化目录记录的端到端流程。 | https://github.com/TopreGroup/AIWebscraper |
| OFBiz-NER 以可部署插件形式识别 brand、color、garment type/style、garment number、size 等商品字段，并兼顾中英文分词；适合参考 GLiNER2 商品抽取服务的业务系统集成，以及品牌/货号/尺码等固定字段的对照基线。 | https://github.com/YYWorks/OFBiz-NER |
| Product_Info_Extractor 直接从原始商品名抽取 Brand、Core Name、Size，并把结果用于外部供应商清单与内部目录的商品匹配；适合验证 GLiNER2 的品牌/核心品名/规格抽取能否直接支撑同款匹配和目录对齐。 | https://github.com/Taha-azizi/Product_Info_Extractor |
| Home Depot Smart Home 数据集把商品标题与 brand、model_number、UPC、SKU、color、尺寸等结构化真值放在同一记录中，且品牌/型号覆盖完整；很适合直接构造 GLiNER2 的“标题/描述→品牌、型号、规格”回归集，并用标识符校验抽取结果。 | https://huggingface.co/datasets/crawlfeeds/HomeDepot-Smart-Home-Dataset |
| Amazon 2023 样例元数据同时提供 title 与 brand、manufacturer、item_model_number、color、material、size、dimensions 等大量目录字段；可批量生成 GLiNER2 的真实商品多字段评测样本，尤其适合测试字母数字型号和跨类目属性 Schema。 | https://huggingface.co/datasets/smartcat/Amazon_Sample_Metadata_2023 |
| Shein 商品样例直接包含 brand、model_number、size/all_available_sizes、other_attributes、类目树等字段；适合用服饰短标题验证 GLiNER2 对品牌、SKU/型号、尺码和长尾变体属性的 span 边界，并评估类目约束 Schema。 | https://github.com/luminati-io/Shein-dataset-samples |
| IKEA 样例同时给出 brand、model_name、model_number、product_series、color、size、materials_description、measurements 和 other_attributes；非常适合专项验证 GLiNER2 如何区分品牌、系列、型号与规格，并覆盖家居类多值属性。 | https://github.com/luminati-io/Ikea-dataset-sample |
| Product Query Benchmark 基于 Amazon ESCI 并补充 EUIPO/Wikidata 可验证品牌 ID、多语言品牌别名及查询-品牌标注；适合评测 GLiNER2 的品牌抽取、别名归一和跨语言品牌泛化，并把 span 结果进一步映射到稳定品牌实体。 | https://huggingface.co/datasets/thepian/product-query-benchmark |
| Home Depot Scraper 能实时返回商品标题/描述以及 brand、model number、UPC、SKU、类目和分组规格等 30+ 结构化字段；适合作为 GLiNER2 线上数据采集与自动造回归集的上游，对抽取结果可直接和页面结构化真值做差异检查。 | https://github.com/omkarcloud/homedepot-scraper |
| Walmart Scraper 的商品详情 JSON 明确包含 brand、model_number、product_category、specifications、variant options、ingredients 等字段；可用来持续采集真实目录样本，验证 GLiNER2 在标题/描述中抽出的品牌、型号与规格是否和商城结构化字段一致。 | https://github.com/omkarcloud/walmart-scraper |
| RuBERT 电商查询 NER 面向短、噪声较大的俄语商品查询，直接识别 TYPE、BRAND、VOLUME、PERCENT，并强调大小写、缩写和轻微拼写错误；适合作为 GLiNER2 多语言品牌/规格抽取在真实搜索词噪声下的对照基线。 | https://huggingface.co/Martsv07/rubert-ner-search-queries |
| MODA 在完整时尚商品检索流水线中直接使用 GLiNER2（fastino/gliner2-base-v1）做零样本 fashion attribute extraction，并保留 GLiNER v1/GLiNER2 消融结果；非常适合直接参考 GLiNER2 商品属性抽取在检索链路中的集成方式、字段增益评估和回归测试。 | https://github.com/hopit-ai/Moda |
| 该 B2B 商品查询项目直接抽取 brand、product type、power 等属性，并同时学习 canonical key 名称及属性优先级；适合对照 GLiNER2 的动态 Schema，把 manufacturer/brand 等同义字段统一、按业务优先级输出，并设计 key+value 精确评测。 | https://huggingface.co/Atkr07/gemma4-priority-attribute-extraction |
| phi4-3b-ec-magento 专门把商品文本或 Magento custom_attributes 按“目标属性→JSON”抽取，并明确用 None 表示字段不存在；适合与 GLiNER2 对照动态单字段/多字段抽取、缺失字段 abstain、严格 JSON 解析和 Magento/PIM 入库接口。 | https://huggingface.co/gabrielgts/phi4-3b-ec-magento |
| RexBERT-large 是电商领域专用 encoder，官方用途直接包含 brand、color、size、material 的 attribute extraction/slot filling，并针对长商品页和属性块做领域预训练；适合作为 GLiNER2 商品字段抽取的固定标签监督基线与领域编码器对照。 | https://huggingface.co/thebajajra/RexBERT-large |
| german-ecommerce-ner-xlmr 提供完整的商品标题属性 NER 工程流程，覆盖 BIO 标注、子词标签对齐、XLM-R 微调、span 解码和置信度过滤；适合用作 GLiNER2 多语言商品标题 brand/model/规格抽取的数据加工、监督基线和低置信度门禁参考。 | https://github.com/shreyar04/german-ecommerce-ner-xlmr |
| Walmart Global Tech 文章直接把商品属性分成闭集属性与开放属性，并明确以 brand、model number 为开放字段用序列标注抽取；很适合用来设计 GLiNER2 的“可抽 span 的品牌/型号 + 可分类的枚举属性”混合落地方式，并衔接商品匹配。 | https://medium.com/walmartglobaltech/product-matching-in-ecommerce-4f19b6aebaca |
| Databricks 的 2026 商品说明书抽取案例用声明式 Schema 从 PDF 中直接抽 manufacturer/brand、model_number、product_type、尺寸/电压等规格，并对缺失值返回 NULL；和 GLiNER2 的动态 Schema 字段提取高度同构，尤其适合参考说明书/规格文档的批量入库与质量评测。 | https://community.databricks.com/t5/technical-blog/intelligent-document-processing-for-data-extraction-transforming/ba-p/153847 |
| LFM2-350M-Extract 是面向 JSON/XML/YAML 的轻量 Schema 驱动结构化抽取模型，社区电商示例已使用 brand、model_number、specifications 等字段；适合与 GLiNER2 做本地轻量模型的抽取质量、CPU/边缘部署和严格结构化输出对照。 | https://huggingface.co/LiquidAI/LFM2-350M-Extract |
| qwen3.5-4b-ec-magento 专门面向 Magento 电商目录，把商品文本或 custom_attributes 按目标字段抽成 JSON，并显式处理字段不存在；可用于对照 GLiNER2 的动态字段抽取、缺失字段 abstain 和 PIM/Magento 入库接口设计。 | https://huggingface.co/gabrielgts/qwen3.5-4b-ec-magento |
| Ecommerce-AVE-Dataset 直接把商品标题/描述配成结构化目标 Schema，样例字段包含 brand、model_number、dimensions、material、features 等；很适合快速构造 GLiNER2 品牌/型号/规格的回归集，并检查多层嵌套字段与缺失值。 | https://huggingface.co/datasets/commotion/Ecommerce-AVE-Dataset |
| Lowe’s Product Scraper 将真实 PDP 统一产出 Brand、Model Number、SKU、Description、Specification 等字段，并包含 DOM/嵌入 JSON 抽取与归一化；可用其结构化结果做 GLiNER2 从页面文本回抽品牌/型号/规格的对照真值和网页采集上游。 | https://github.com/seth-risenow/lowes |
| GLiNER2 官方 API 教程直接用 `extract_json` 将 iPhone 15 Pro Max 文本解析为 name、price、storage、color，并支持置信度和字符位置；适合参考把品牌/型号/规格 Schema 封装成线上 API、做结果置信度门禁和可追溯 span。 | https://github.com/fastino-ai/GLiNER2/blob/main/tutorial/7-api.md |
| ANE 的 Qwen3.5 2B 商品抽取模型从德国二手/电商 Listing 直接输出 productName、brand、model、condition、category 和 attributes JSON，且面向低成本规模化推理；可作为 GLiNER2 品牌/型号/属性结构化抽取的轻量生成式对照基线。 | https://huggingface.co/Ekwav/ane-extraction-qwen3.5-2b |
| 该 Qwen3 4B Product Extractor 专门从商品目录文本生成结构化 JSON，并提供 CPU 优化的 GGUF 量化版本；适合与 GLiNER2 做本地部署、CPU 推理、结构化输出稳定性和资源成本的横向对照。 | https://huggingface.co/pragnesh002/Qwen3-4B-Product-Extractor-GGUF-Q4-K-M |
| DoorDash 的生产实践把商品名中的 Brand、Size、UOM 等字段做实体抽取，并构建“分类器低置信度→LLM 品牌抽取→知识图谱去重→回流训练”的品牌入库链路，还用 RAG+LLM 批量生成通用属性标注；非常适合参考 GLiNER2 的品牌发现、去重、弱标注和持续训练闭环。 | https://careersatdoordash.com/blog/building-doordashs-product-knowledge-graph-with-large-language-models/ |
| ai-fashion-assistant-v2 在 44,417 个商品上抽取 30 万级视觉属性，覆盖 pattern、fit、material、style 等 10 类，并将属性用于多模态检索；适合把 GLiNER2 的文本品牌/型号/属性作为主链路，再用视觉属性补充文本缺失字段并验证抽取结果对搜索的实际增益。 | https://github.com/haticebaydemir/ai-fashion-assistant-v2 |
| 该 2025 研究系统分析真实电商商品标题的句法、内容、词序和属性分布，并验证这些特征对商品任务建模的影响；适合用来设计 GLiNER2 的短标题 Schema、字段边界和顺序鲁棒性测试，尤其处理品牌、型号、规格被压缩拼接的场景。 | https://www.sciencedirect.com/science/article/pii/S0957417425013247 |
| 这项已部署的商品标题浅层语义解析工作会先把 offering title 切成具有业务语义的片段，并提供可跨电商品类复用的标注结构；适合放在 GLiNER2 前做标题分段/候选片段定位，减少品牌、型号、商品类型和修饰属性互相串位。 | https://doi.org/10.1145/2623330.2623343 |
| Constructor 的电商搜索意图方案明确从查询中识别 Brand、Color、Size/Dimensions 等属性，并建议组合规则模式与机器学习抽取器；适合参考 GLiNER2 与正则/单位词典混合，对品牌等语义字段和尺寸规格等规则字段分别优化。 | https://constructor.com/blog/extracting-ecommerce-search-intent-from-head-to-tail |
| BatchGPT 的商品名批量解析流程直接要求逐条找 brand、model、color、size，并显式保留缺失值；和 GLiNER2 的 Schema 驱动 JSON 抽取非常接近，可参考批处理字段契约、缺失字段表示和结果落表方式。 | https://aifficientools.com/blog/extract-key-attributes-product-names-batch/ |
| 该 2026 零售目录匹配案例先从宣传页文本抽出 brand、product name、pack size，再做单位归一、精确/候选匹配并拒绝字段不全的样本；适合把 GLiNER2 输出直接接到商品目录 ID 对齐，并用“品牌+品名+规格”做强校验。 | https://medium.com/@conyeneke1/from-flyer-text-to-catalogue-id-building-a-retail-product-matching-pipeline-34738294aff0 |
| Trend-Setters 在实际检索应用中用 LLaMA 从用户查询抽取 brand、color、size、category、gender 等商品属性，再进入 Qdrant 检索；可作为 GLiNER2 动态 Schema 抽取结果驱动商品搜索/推荐的轻量端到端实现参考。 | https://github.com/priyam-hub/Trend-Setters |
| Logic 的 Catalog Attribute Extraction 工作流从商品名称、描述和可选规格文档中直接抽取 brand、size、colour 等字段并结构化输出；适合对照 GLiNER2 的多来源文本输入、字段 Schema 和目录富化批处理形态。 | https://logic.inc/workflows/extract-brand-size-colour-from-product-text |
| MetaData-Extraction 是服饰商品的视觉元数据抽取项目，用 GPT-4 Vision 从商品图片补充品牌、颜色、风格等目录属性；适合作为 GLiNER2 文本品牌/型号/属性抽取的图片兜底，在标题缺字段时用视觉证据补充或交叉校验。 | https://github.com/sanket98a/MetaData-Extraction |
| TREC Product Search 2025 商品语料同时提供标题/描述与 brand、model、color、size、material 等结构化字段，适合直接把目录字段当弱标签构建 GLiNER2 品牌/型号/规格回归集，并专项测试型号与多属性 span 边界。 | https://huggingface.co/datasets/trec-product-search/product-recommendation-2025 |
| Channel3 Universal Product Graph 把跨商家商品统一成 brands、materials、structured_attributes、variants 等规范字段，并处理同款与变体归并；适合参考 GLiNER2 抽取后的品牌/属性标准化、变体对齐和目录实体消歧。 | https://huggingface.co/datasets/trychannel3/channel3-universal-product-graph-sample |
| Japan Fashion Items 含 446 万级日语时尚商品，提供品牌、商品名/描述、属性关键词及颜色/尺码变体；适合验证 GLiNER2 在日语商品短文本中的品牌、材质、颜色、尺码和货号等多字段抽取与跨语言泛化。 | https://huggingface.co/datasets/otdnnc/japan-fashion-items |
| ASOS Fashion Products 数据把商品名/描述与 brand、sku、color、sizes、多个 material 字段以及 closure/toe shape 等细粒度属性放在同一记录；非常适合按服饰类目动态生成 GLiNER2 Schema，并逐字段做品牌/货号/规格回归评测。 | https://huggingface.co/datasets/crawlfeeds/asos-fashion-products-dataset |
| Shopee Product Listings 提供真实 marketplace 标题/描述、brand/brandId 以及 models 变体/SKU JSON，且包含中文与东南亚电商噪声文本；适合测试 GLiNER2 的长尾品牌、字母数字型号、变体规格抽取以及结果到目录 ID 的映射。 | https://huggingface.co/datasets/rebrowser/shopee-dataset |
| PCPartPicker Parts 将商品名与 brand、category 和 category-specific specs JSON 对齐，标题中大量包含 Ryzen 9800X3D、MAG A750GL 等型号；很适合构建 GLiNER2 电子硬件品牌/型号/功率/容量等规格字段的高难度回归集。 | https://huggingface.co/datasets/Doshiba/pcpartpicker-parts-dataset |
| Tesco Grocery UK 数据把商品名与 brand、category、unit、price_per_unit 等目录字段对齐，并明确支持 NER 用途；适合用 GLiNER2 抽取品牌、包装规格、计量单位等快消字段，并验证数值+单位的 span 与标准化。 | https://huggingface.co/datasets/crawlfeeds/tesco-grocery-uk |
| DEFLATE 将商品属性抽取统一为“多模态候选生成 + 判别校验”，同时覆盖文本显式值和图片/语义中的隐式值；适合把 GLiNER2 作为高效显式 span 主抽取器，再为缺失/低置信度 brand、model、颜色、材质等字段接视觉/生成式兜底与验证。 | https://aclanthology.org/2023.findings-acl.831/ |
| ImPaKT 面向购物指南做开放 Schema 信息抽取，标注了商品属性、属性类型、属性摘要及复合/原子属性关系；适合在 GLiNER2 固定 brand/model/规格 Schema 之外发现新字段并归并同义/复合属性，形成“属性发现→审核→Schema 扩充”的冷启动数据。 | https://arxiv.org/abs/2212.10770 |
| KG-FLIP 直接利用商品属性 Schema 做电商多模态预训练，并已用于 Amazon Catalog 回填缺失属性；适合参考 GLiNER2 的类目字段 Schema 与图片表征结合，在文本缺失颜色、材质、款式等属性时做目录补全。 | https://aclanthology.org/2023.acl-industry.9/ |
| Tab-Cleaner 在百万级 Amazon 商品目录上同时做“属性是否适用”和“属性值是否可信”的弱监督校验；适合接在 GLiNER2 抽取后，对 brand/model/规格等字段先判断类目适用性再做值校验，减少无关字段误抽和脏值入库。 | https://aclanthology.org/2023.acl-industry.18/ |
| PV2TEA 专门把视觉信息补到已有文本属性抽取器中，并针对图文松耦合、背景噪声和文本弱标注偏差设计纠偏；适合保留 GLiNER2 文本主抽取链路，同时用商品图片补强颜色、形状、图案等字段并校验文本误导。 | https://aclanthology.org/2023.findings-acl.127/ |
| 该 2026 实践文章把商品数据抽取目标明确为可直接消费的结构化商品记录，字段覆盖 brand、model number、variants、specs、identifiers，并强调抽取后的标准化与下游搜索/推荐/商城接口；适合用来定义 GLiNER2 品牌、型号、规格输出的数据契约和后处理边界。 | https://www.getcatalog.ai/blog/product-data-extraction |
| 该完整落地指南把 LLM 目录富化拆成结构化属性抽取、taxonomy 映射/归一化、证据引用、幻觉控制和质量保障流水线；适合补充 GLiNER2 抽取后如何做字段约束、标准化、QA 与批量目录治理。 | https://productphilosophy.com/articles/llm-powered-catalog-enrichment |
| 该 2026 商品数据抽取案例直接覆盖供应商 PDF、Excel、OCR 与文本等异构输入，并把结果转换为可校验的结构化数据；很适合拿来设计 GLiNER2 的多来源文本接入、品牌/型号识别、规格归一化和脏数据测试集。 | https://trueparse.ai/blog/ai-powered-product-data-extraction-ecommerce |
| 该商品匹配研究把 brand、model id、product number、units 从非结构化商品标题中先行抽取，再用于跨商城商品去重/匹配；适合验证 GLiNER2 的品牌、型号、货号、单位字段能否直接提升同款识别，并参考字段抽取顺序与归一化策略。 | https://www.scitepress.org/PublishedPapers/2019/80694/pdf/index.html |
| EC-Guide 提供 7,429 条电商 NER 样本并覆盖商品属性相关任务，可直接补充 GLiNER2 的品牌/型号/属性微调与回归集，尤其适合验证同一动态 Schema 在多种电商文本任务上的迁移稳定性。 | https://github.com/fzp0424/EC-Guide-KDDUP-2024 |
| AdaSeq 的 RaNER 在电商 NER 上用“检索相似样本→联合编码/投票”增强领域实体识别；可迁移为 GLiNER2 的候选上下文层，为长尾品牌、型号和别名抽取提供相似商品证据。 | https://github.com/modelscope/AdaSeq/tree/master/examples/RaNER |
| Jellyfish-8B 明确在 AE-110K、OA-Mine 商品属性值抽取基准上给出结果，可作为 GLiNER2 的生成式/数据处理中型基线，帮助比较动态 Schema 抽取与 LLM AVE 在精度、成本和结构化输出上的差异。 | https://huggingface.co/NECOUDBFM/Jellyfish-8B |
| DataWeBot 的目录富化链路明确覆盖 brand、manufacturer、MPN/model number、技术规格、类目专属 Schema，并配套归一化、冲突消解和 QA；适合参考 GLiNER2 从抽取到 PIM/目录入库的工程字段契约与质量门禁。 | https://www.datawebot.com/solutions/product-catalog-enrichment |
| Shopee TH/SG 的 Product Taxonomy Extraction 项目把品牌/商品体系归一、LLM 多模态字段抽取和 43 个类目的数据管道放在同一工程中；适合参考 GLiNER2 的“类目路由→brand/model/属性 Schema→抽取→品牌/商品规范化→数仓入库”完整落地链路。 | https://github.com/iethes/product-taxonomy-extraction |
| Firecrawl 的 Amazon 商品案例用 Pydantic/JSON Schema 约束网页抽取并明确采集品牌等商品详情字段；适合放在 GLiNER2 前后作为网页正文获取与结构化 Schema 对照，尤其参考 PDP 批量采集、字段契约和可验证输出。 | https://github.com/firecrawl/firecrawl/blob/main/examples/blog-articles/amazon-price-tracking/notebook.md |
| 这篇电商 Schema 设计指南专门讨论跨 Amazon、Shopify 等不同站点抽统一商品字段，并以 brand、SKU 等作为商品页基础字段；适合指导 GLiNER2 按商品详情页设计 brand/model/规格 Schema、处理字段缺失与跨站差异。 | https://scrapewithruno.com/blog/schema-design-ecommerce/ |
| 该多模态电商数据富化研究用“规则手册 + 动态 Prompt”按商品类型确定应抽属性并输出指定结构，且规则式提示显著优于零样本；适合迁移到 GLiNER2 的类目专属 Schema、属性定义约束、图片补证和抽取一致性校验。 | https://www.bibliomed.org/?mno=243170 |
| OpenTag 原始工作把商品标题/描述中的开放词表属性值抽取建模为序列标注，并结合注意力与主动学习，在少量标注下发现训练中未见的新属性值；适合参考 GLiNER2 对长尾品牌、新型号、规格新值的开放抽取能力及低成本标注/难例采样策略。 | https://www.amazon.science/publications/opentag-open-attribute-extraction-from-product-profiles |
| 该研究从非结构化多语言商品网页中抽取细粒度、标准化商品属性，并验证模型可跨网店、跨语言迁移，同时支持商品 taxonomy 对齐；适合评估 GLiNER2 的品牌/型号/规格字段在跨站点、多语言目录中的泛化，以及抽取后字段标准化与类目映射。 | https://arxiv.org/abs/2302.12139 |
| 这个 Ruby/ONNX 的 GLiNER2 推理封装直接给出 iPhone 商品结构化抽取示例，可一次得到完整型号、storage、processor、price 等字段，并支持 INT8 ONNX；适合参考 GLiNER2 商品品牌/型号/规格服务的跨语言封装、本地低成本部署和固定 Schema 输出。 | https://github.com/elcuervo/gliner |
| PARSE 会自动优化 JSON Schema，并用反思式抽取结合静态与 LLM guardrail 降低字段歧义和抽取错误；适合迁移到 GLiNER2，持续改进 brand、model、规格字段描述，并对低置信结果做 Schema 级校验。 | https://aclanthology.org/2025.emnlp-industry.184/ |
| 该研究先对冗长商品 description/bullets 做句子级相关性排序，再交给 NER 类抽取链路；适合放在 GLiNER2 前筛除噪声，只保留最可能包含品牌、型号、规格的片段，降低长文本误抽和推理成本。 | https://arxiv.org/abs/1907.06330 |
| tech-product-ner 的标签直接覆盖 BRAND、COLOR、CATEGORY 与价格约束等电商实体；可作为 GLiNER2 商品查询侧品牌/属性 span 的小型回归集，并扩展 MODEL、规格字段做 few-shot 校准。 | https://huggingface.co/datasets/roundspecs/tech-product-ner |
| datasheet_extractor 用 PDF parser、LLM/VLM、Pydantic Schema 和 validator 把电子元件 datasheet 转成结构化规格；适合参考 GLiNER2 从说明书/规格书抽品牌、型号和参数字段时的文档解析、Schema 约束与结果校验。 | https://github.com/hsinjuitsai/datasheet_extractor |
| Alibaba 商品详情抓取项目直接产出 brand、SKU、MPN、商品 ID 等结构化字段；可将页面结构化结果作为弱标签或对照真值，批量验证 GLiNER2 从标题/描述回抽品牌、型号/货号和规格的准确率。 | https://github.com/AbsoluteAnchor/alibaba-single-product-details-scraper |
| CommerceTXT 为 AI Agent 定义机器可读电商商品规范，字段映射中直接包含 Brand 等核心商品信息；适合把 GLiNER2 抽取结果映射到稳定的商品交换 Schema，明确品牌、型号与属性的标准化输出边界。 | https://github.com/commercetxt/commercetxt/blob/main/spec/README.md |
| third-eye 是自托管商品页解析 API，可从页面直接得到 title、brand、price、sizes 等结构化数据，并融合 JSON-LD/OpenGraph/Shopify/DOM；适合给 GLiNER2 提供干净候选文本与页面结构化对照值，减少网页噪声并做字段回归。 | https://github.com/myselfshravan/third-eye |
| GLiNER2 官方 LoRA 教程展示了按领域训练轻量 Adapter 并按文档类型路由切换的方式；适合把商品 brand/model/规格 Schema 做成电商专用 Adapter，在保持基础模型通用能力的同时低成本迭代长尾字段。 | https://github.com/fastino-ai/GLiNER2/blob/main/tutorial/10-lora_adapters.md |
| mobile-phone-specs 覆盖 1 万+ 手机并直接提供 brandName、modelName、dimensions、display、processor、camera 等结构化字段；可用作电子类 GLiNER2 品牌/型号/规格抽取的对照真值，专项测试长型号和参数边界。 | https://github.com/rawford-ilderman/mobile-phone-specs |
| DigiKey Scraper 直接提供 manufacturer、manufacturer_part_number、series 以及 30+ 参数规格和 datasheet；适合构造电子元器件场景的 GLiNER2 品牌/MPN/规格回归集，并把说明书抽取结果与标准目录字段逐项校验。 | https://github.com/omkarcloud/digikey-scraper |
| Open Icecat API 支持用 Brand + ProductCode 或 GTIN 定位标准商品记录；适合接在 GLiNER2 品牌/型号/货号抽取后做目录反查、实体归一和错误拦截，也可据此自动生成带可信标识符的评测样本。 | https://github.com/Tjark-Kuehl/open-icecat |
| 该 2026 商品匹配项目在 Amazon/Walmart 商品上以 brand、size、color、pack count、model year 等 10 个关键属性做二阶段校验，并设置自动合并/人工复核/拒绝区间；适合验证 GLiNER2 抽取字段能否直接支撑同款归并，并参考低置信结果的质量门禁。 | https://github.com/Wijt/product-matching |
| Cốc Cốc 的实战方案从约 200 万条越南电商 Listing 构建商品属性 NER 数据与词典，字段直接覆盖品牌、商品编码、尺寸、颜色等，并结合规则、聚类和置信度筛选弱标注；适合为 GLiNER2 的品牌/型号/规格字段低成本构造训练数据与难例词典。 | https://careers.coccoc.com/blogs/attribution-extraction-of-e-commerce-product-listing-part-1 |
| 该开源商品入库流水线覆盖供应商 XML、Playwright 商品页抓取、技术规格/属性/尺寸/重量信息抽取、规范化以及 Shopify CSV 落库；适合参考把 GLiNER2 放进真实供应商目录处理链路，负责 brand/model/规格语义抽取并接后续标准化与增量更新。 | https://github.com/vaskyp/product-scraping-shopify-csv-automation-pipeline |
| product-query-ner 在 Amazon ESCI QueryNER 基础上加入欧洲品牌与多语言商品名数据，保留 brand、product name、UoM、color、material 等 17 类标签，并提供 span 后处理与量化部署版本；适合作为 GLiNER2 多语言商品查询品牌/属性抽取的新增对照基线。 | https://huggingface.co/thepian/product-query-ner |
| Flipkart 商品属性抽取项目用字符级表示 + BiLSTM + CRF 做序列标注，示例直接标注 B_BRAND、B_COLOR、B_SLEEVE、B_TYPE，且面向卖家提供的噪声商品文本；适合作为 GLiNER2 在短商品标题上的固定标签监督基线，并用于比较品牌与细粒度属性 span 边界。 | https://github.com/SumitVermakgp/NLP-Attribute-Extraction-Flipkart |
| multimodal-product-catalogue 把商品入库阶段的属性抽取做成独立 Agent，以严格 JSON/Pydantic 输出 colour、style、material、shape、extras，并将属性直接用于文本/图片联合检索；适合参考把 GLiNER2 作为文本属性主抽取器接入“抽取→结构化校验→向量索引→检索”的端到端目录链路。 | https://github.com/visy-ani/multimodal-product-catalogue |
| Ivory-Parts-Finder 从真实电脑零售商品名批量抽取 manufacturer 和 part_number/SKU，并写入机器可读 JSON，样例可识别 Samsung 与 MZ-V9P2T0BW 这类品牌和字母数字型号；非常适合用来验证 GLiNER2 的品牌/型号边界、批量处理与抽取结果入库格式。 | https://github.com/danielrosehill/Ivory-Parts-Finder |
| 这个中文商品信息抽取模型直接从商品名称输出“品牌、型号、主商品”JSON，字段与 GLiNER2 要落地的核心目标几乎一一对应；适合拿来做中文 SKU 标题的基线、Schema 设计参考，并重点对照型号误判和幻觉问题。 | https://huggingface.co/ykallan/SkuInfo-Qwen2.5-3B-R1 |
| Saleor 的实战方案先读取商品类型已有 Schema，只让模型补齐缺失属性，并用枚举白名单、证据片段和人工审核后再回写；非常适合迁移成 GLiNER2 的“类目 Schema 约束→字段抽取→证据校验→安全写库”生产链路。 | https://saleor.io/blog/saleor-app-ai-catalog-enrichment |
| 该 4 万商品案例直接处理“只有混乱商品名、没有品牌/型号/类型”的目录，用外部权威资料交叉验证后抽取并归一化 brand/model，还设置低置信度剔除；很适合参考 GLiNER2 后面的型号标准化、别名合并和质量闸门。 | https://granulargroup.com/case-study/using-ai-to-turn-40000-unstructured-products-into-a-navigable-seo-ready-catalog/ |
| AliExpress Product Extraction Engine 已把品牌、型号、规格、SKU、尺寸等 70+ 商品字段做成可插拔抽取器，并为每个字段保留置信度、来源和候选值；适合参考 GLiNER2 的“语义字段抽取→多来源合并→归一化→校验”工程链路，尤其可借鉴品牌/型号结果的证据追踪和质量门禁。 | https://github.com/sherifmohamed3378-ui/aliexpress-actor-products |
| Mouser Electronics Price Tracker 的标准输出直接包含 brand、model_number、sku/part number 和商品描述，适合用真实电子元器件目录构造 GLiNER2 的品牌/型号/货号回归集，重点验证字母数字混合型号、MPN 与 SKU 的边界及抽取后标准化。 | https://github.com/luminati-io/mouser-electronics-price-tracker |
| Amazon Selling Partner API 的官方 Catalog Items Schema 明确定义 brand、manufacturer、modelNumber、partNumber 等商品目录字段，并提供结构化 attributes、dimensions、identifiers 等数据；适合用作 GLiNER2 输出的 canonical Schema，统一品牌/制造商、型号/料号等相似字段边界并约束目录入库。 | https://github.com/amzn/selling-partner-api-models/blob/main/models/catalog-items-api-model/catalogItems_2022-04-01.json |
| AWS 官方 Product Catalog Enhancement Guidance 把原始商品数据拆成标题、描述与特征抽取链路，特征步骤直接覆盖材质、颜色、尺码、重量、尺寸、容量、兼容性等；适合参考 GLiNER2 的多字段 Schema、异步批处理和结构化入库流程。 | https://docs.aws.amazon.com/solutions/product-catalog-enhancement-with-generative-ai-on-aws/ |
| Google Cloud 的商品数据富化实践用生成式 AI 从现有目录数据补全和规范化商品信息，并围绕 PIM 持续维护属性；适合参考 GLiNER2 抽取后的目录富化、字段标准化和批量回写。 | https://cloud.google.com/blog/products/ai-machine-learning/generating-new-product-information-with-vertex |
| Google Cloud/Tamr 的电商案例把商品名称作为结构化文本抽取源，直接从现有目录中生成新属性而无需逐字段人工标注；适合参考 GLiNER2 从短商品标题批量抽品牌、型号及长尾属性并持续扩展 Schema。 | https://cloud.google.com/blog/products/data-analytics/how-tamr-data-products-leverage-generative-ai/ |
| GLiNER2 官方 Regex Validator 教程可对抽取 span 做格式约束并过滤假阳性；很适合给型号、SKU/MPN、容量、尺寸等规则性强的商品字段增加校验门禁，降低相似数字串误抽。 | https://github.com/fastino-ai/GLiNER2/blob/main/tutorial/5-validator.md |
| GLiNER2 官方训练教程覆盖实体描述、JSONL 数据、验证集和完整 NER 微调流程；适合把品牌、型号、规格难例沉淀成电商训练集，并用字段描述拉开相似 Schema 的边界。 | https://github.com/fastino-ai/GLiNER2/blob/main/tutorial/9-training.md |
| AWS Product Attribution and Personalization Guidance 用多模态 AI 从商品图片识别标准与上下文属性并映射到目录；适合在 GLiNER2 文本抽取缺少颜色、风格等字段时做视觉补证和类目属性补全。 | https://docs.aws.amazon.com/solutions/product-attribution-and-personalization-using-amazon-bedrock/ |
| AWS 商品描述生成 Guidance 会先用计算机视觉和 NLP 分析商品图片并抽取相关属性，再生成目录内容；适合把 GLiNER2 作为文本主抽取器，并用图片侧属性作为缺失字段兜底和一致性校验。 | https://docs.aws.amazon.com/solutions/generating-product-descriptions-with-amazon-bedrock/index.html |
| 该智能手机电商研究直接比较结构化规格表与非结构化商品标题的信息抽取，并围绕 Brand、Model、Color 等实体及其关系做 BERT 抽取；适合用来评估 GLiNER2 对品牌/型号/颜色 span 的边界，以及抽取后品牌-型号-属性组合的一致性。 | https://aclanthology.org/2021.stil-1.12/ |
| 该中文电商 NER 研究把实体类型写成 MRC 问题，从商品描述中识别品牌与商品实体；和 GLiNER2 用自然语言 Schema 定义字段的方式很接近，适合参考中文 brand/商品实体字段描述、边界定位与少样本迁移。 | https://drpress.org/ojs/index.php/fcis/article/view/12817 |
| 该研究专门从印尼电商商品标题联合抽取属性，结合词典、词表示与序列/联合建模处理 brand、商品名等字段；适合用作 GLiNER2 多语言短标题的传统基线，并设计品牌/品名与细粒度属性的联合 span 回归测试。 | https://www.scielo.org.mx/scielo.php?pid=S1405-55462018000401367&script=sci_arttext |
| 该 NER 研究在电商数据集中直接使用 Brand、Product、Model 三类实体，并结合远程监督与跨域数据降低人工标注依赖；适合为 GLiNER2 的中文长尾品牌、新型号构造弱标注训练数据，并验证跨商城/跨域迁移。 | https://pmc.ncbi.nlm.nih.gov/articles/PMC9168158/ |
| OneSearch 的开源代码在真实电商搜索链路中用 NER 维护 18 类结构化属性，直接包含 Brand、Model、Specifications、Color、Material，并把这些字段用于 query/item 语义增强；适合将 GLiNER2 作为动态 Schema 抽取器接入，验证品牌、型号和规格抽取对检索链路的实际增益。 | https://github.com/benchen4395/onesearch-family |
| OneRetrieval 的开源实现把电商 Brand、Model/GOOD_MODEL、Specification、Color、Material 等细粒度属性先抽取和归并，再编码进可检索的结构化表示；适合参考 GLiNER2 抽取后的字段分组、词典维护、增量更新以及“抽取→检索”衔接。 | https://github.com/xuxinzhang/oneretrieval |
| KuaiSearch 提供 1800 万级真实电商商品的 title+brand，以及 query/title/brand_name/attribute 相关性数据；可用现成目录字段构造 GLiNER2 品牌与属性弱标注/回归集，并直接评估抽取结果对召回、相关性和排序的业务价值。 | https://github.com/benchen4395/KuaiSearch |
| SchemaRAG 在真实电商数据上用检索动态裁剪大 Schema，最高提升 micro-F1 8.8%、延迟降低 47%、Token 成本降低 48%；适合在 GLiNER2 面对大量类目属性时先按商品文本召回 brand/model/规格候选字段，减少无关 Schema 干扰和推理成本。 | https://aclanthology.org/2026.acl-industry.78/ |
| 这个土耳其电商 NER 基准直接把 GLiNER2 与微调 Qwen、GLiNER、BERT 放在同一 200 条留出集上比较，标签明确包含 BRAND、MODEL、COLOR、SIZE_VARIANT、MATERIAL、SPECIFICATION；非常适合复用为 GLiNER2 品牌/型号/规格多语言回归基准。 | https://github.com/gururaser/magibu-uygulamali-yz-egitim/tree/main/les4/unsloth_ecommerce_ner_benchmark |
| fast_gliner 为 GLiNER2 提供 Rust + ONNX Runtime 的 Python 推理实现，支持 NER、结构化抽取和多任务 Schema，README 直接给出商品 name/price/features/category 抽取示例并报告约 4× CPU 加速；适合商品目录高吞吐批处理和本地服务部署。 | https://github.com/talmago/fast_gliner |
| Product Extraction Benchmark 提供真实商品页 HTML/WARC、ground truth、评测代码和开源基线，标注中包含 brand、GTIN、SKU，并专门分析 SKU/MPN 混淆；适合扩展成 GLiNER2 网页商品品牌/型号/货号抽取的固定输入回归集，避免线上页面变化影响对比。 | https://github.com/scrapinghub/product-extraction-benchmark |
| DySECT 是 2026 ACL 的动态自演化结构化抽取系统，会把抽取结果持续沉淀到可扩展知识库并适应新术语和罕见离群值；适合借鉴到 GLiNER2 长尾品牌、新型号、新规格的“抽取→知识库积累→候选增强/校验”持续演化闭环。 | https://aclanthology.org/2026.acl-demo.69/ |
| GLiNER2Swift 是 GLiNER2 的 Swift/MLX 原生实现，支持 NER、结构化抽取与 LoRA Adapter，并可在 Apple Silicon/iOS/macOS 本地运行；若商品扫描、离线识别或端侧目录工具需要提取 brand/model/规格，可参考其端侧部署与适配器复用方式。 | https://github.com/MacPaw/Gliner2Swift |
| 该研究针对京东中文商品标题中的大量专业属性词和非规范短文本，用标签语义与多粒度上下文提升实体边界识别；适合参考 GLiNER2 中文 brand/model/规格 Schema 的字段描述、相似字段区分和短标题回归测试。 | https://doi.org/10.1111/coin.12654 |
| 该 2025 电商实体抽取研究用 BERT-BiLSTM-CRF 处理复杂、非标准化商品描述，并基于真实任务自建数据验证高精度实体抽取；适合作为 GLiNER2 商品文本 span 抽取的监督基线及脏文本难例对照。 | https://doi.org/10.1007/s11227-025-07035-x |
| Rakuten 的研究直接比较 ChatGPT、Gemini 与 Llama-2 系模型在日本时尚商品属性抽取上的准确率、结构化输出、基础设施和成本；适合给 GLiNER2 做生产选型对照，重点评估字段一致性、批量成本与本地模型优势。 | https://doi.org/10.1145/3678610.3678619 |
| Rakuten 的互联网规模商品匹配方案在标题/描述上做类目专属 NER 属性抽取，并用知识图谱字段定义、Snorkel 弱监督和持续训练支撑商品匹配；很适合参考 GLiNER2 的“类目 Schema→弱标注→属性抽取→同款匹配”生产链路。 | https://doi.org/10.1145/3556089.3556149 |
| 该开源项目对应“电商商品标题 NER”硕士研究，围绕品牌及选定商品属性构建多语言数据与基准；适合拿来补充 GLiNER2 在英、波、德、西、法等多语言标题上的 brand/属性 span 评测和 few-shot 对照。 | https://github.com/grant-TraDA/named-entity-recognition-in-titles-of-e-commerce-products |
| 该商品属性抽取研究明确覆盖 brand、size、weight、dimension 等字段，同时尝试用商品描述预测品牌并用无监督方法关联属性名和值；适合和 GLiNER2 对照品牌抽取、属性值配对及低标注数据场景。 | https://doi.org/10.13140/RG.2.2.11045.47842 |
| 该工作从厂商官网端到端定位产品页面并自动抽取规格信息，目标是把分散网页规格直接整合进企业系统/网店；适合参考 GLiNER2 的“先定位商品正文/规格区→再按 brand、model、技术参数 Schema 抽取”的网页落地链路。 | https://www.scitepress.org/PublishedPapers/2010/28743/pdf/index.html |
| 该研究从多个电商站点的商品描述页无监督抽取热门商品属性，并利用评论侧信息跨越页面与用户表达的词汇差异；适合补充 GLiNER2 固定 Schema 之外的属性发现、长尾字段候选生成和新类目冷启动。 | https://doi.org/10.1145/2857054 |
| 该 VLDB 工作面向在线商品目录自动维护，从商家 offer 落地页抽取属性-值对并通过 Schema Reconciliation 消除不同来源字段噪声；适合参考 GLiNER2 抽取后的 supplier 字段归并、同义属性对齐和多来源目录入库。 | https://doi.org/10.14778/1988776.1988777 |
| 该产品集成研究用网页中已有结构化商品数据作为弱监督，训练模型从非结构化商品描述抽取 attribute-value pairs，再用于商品匹配和分类；适合借鉴用现有 PIM/商城字段自动构造 GLiNER2 的 brand/model/规格伪标注并验证抽取对同款匹配的业务增益。 | https://journals.sagepub.com/doi/10.3233/SW-180300 |
| Width.ai 的 2026 商品属性富化实践明确把 brand、model、voltage、color、weight 等作为结构化目标字段，并强调先做来源检索再按规则、示例和阈值生成/校验；适合让 GLiNER2 负责固定 Schema 的快速抽取，再用来源证据与阈值复核型号和规格，降低脏目录中的误抽与幻觉。 | https://www.width.ai/post/product-attribute-enrichment-2026 |
| Akeneo 2026 的 Web-based Attribute Enrichment 已在 PIM 中落地“缺失字段→联网查证→带来源建议→人工/工作流审核”，用例直接包含 brand、manufacturer、model 和技术规格；适合把 GLiNER2 作为现有标题/描述/供应商文档的一阶段本地抽取器，仅对缺失或低置信字段触发联网补证。 | https://help.akeneo.com/using-ai-in-the-pim/what-is-web-based-attribute-enrichment |
| 这篇 2026 商品目录实践专门围绕 model number 自动抽取与跨商城匹配，强调型号作为同款识别、去重、目录标准化和持续更新的核心标识；适合把 GLiNER2 的 model/model number/MPN 抽取结果接到规范化与商品匹配层，并据匹配冲突反向发现型号误抽。 | https://www.productdatascrape.com/automated-product-matching-by-model-number.php |
| 该 2026 目录富化选型文章建议把批量标题清洗、Schema 可靠的属性抽取、复杂类目推理和多模态兜底分层处理；很适合 GLiNER2 作为低成本本地 brand/model/属性主抽取器，仅将难例、隐式属性或图片证据缺失的商品升级到更强模型，控制大规模目录处理成本。 | https://aimodels.deepdigitalventures.com/blog/ai-models-for-ecommerce-catalog-enrichment-which-models-best-clean-up-product-titles-attributes-and-categories/ |
| RoundSpecs 的商品数据同时保留 title、description、specTableContent、brand、category 和商品聚类信息，可直接把“非结构化标题/描述/规格表→brand 与规格字段”做成 GLiNER2 回归集，并用 cluster_id 检查抽取字段对同款聚类/匹配的帮助。 | https://huggingface.co/datasets/roundspecs/product-dataset |
| 该 Alibaba 实时商品抓取项目会把商品完整规格、变体、MOQ 与供应商信息整理成结构化 JSON，变体中明确包含 model number；适合把复杂 B2B 商品页作为 GLiNER2 的型号/规格抽取输入，并用结构化结果做回归校验。 | https://github.com/omkarcloud/alibaba-scraper |
| WANDS 提供 Wayfair 商品标题、描述、类目以及大规模 product_features 键值属性（材质、颜色、尺寸、承重等）；适合把这些键值属性反向当作弱监督/评测真值，测试 GLiNER2 在家具长描述中的多属性抽取和字段归一化。 | https://huggingface.co/datasets/shuttie/wands |
| IKEA US CommerceTXT 将 3 万多商品统一为 Name、SKU、Brand、Category、Materials、Dimensions 等清晰字段；很适合构造“商品文本→GLiNER2 Schema 输出”的标准样本，验证品牌、SKU/型号类标识和材质/尺寸等属性的结构化抽取稳定性。 | https://huggingface.co/datasets/tsazan/ikea-us-commercetxt |
| ArgusFlow 是开源商品数据抽取套件，既能从商品 HTML 提取 brand 与技术规格，也能把脏商品标题转成结构化 JSON，并带商品匹配能力；适合参考 GLiNER2 的“页面/标题→品牌/型号/规格→目录匹配”端到端工程链路。 | https://github.com/getargusflow/argus |
| ScrapeGraphAI Extract 支持对 URL、HTML、Markdown 按 JSON Schema/Pydantic 做结构化抽取，官方直接给出商品 Schema 示例；适合参考 GLiNER2 前置网页正文获取、动态字段 Schema 契约以及抽取结果类型校验。 | https://docs.scrapegraphai.com/services/extract |
| product-harvest 可无模型地从商品页 JSON-LD、OpenGraph、meta 直接解析 brand、SKU、GTIN、类目等结构化字段；适合与 GLiNER2 组合成“已有结构化数据优先、缺失品牌/型号/规格再语义抽取”的低成本混合链路，并作为对照真值。 | https://pypi.org/project/extract-product/ |
| NuExtract3 是开源本地结构化抽取模型，输入文本/图片与 JSON 模板即可输出结构化 JSON，并支持多语言、多模态和 vLLM 部署；可把 brand、model、规格定义成同一模板，与 GLiNER2 做本地吞吐、结构约束和多模态兜底对照。 | https://github.com/numindai/nuextract |
| SKU Launch 的商品抽取流程覆盖标题、URL、PDF、图片和联网补证，并把 Brand、Voltage、Weight 等结果统一映射到自定义 Schema，附逐字段置信度、来源和人工复核；很适合借鉴 GLiNER2 的多来源输入、规范化与低置信度质量门禁。 | https://skulaunch.com/platform/product-data-extraction |
| Upsonic 的电商 Agent 示例会自动发现官网、导航到商品页，再将 product_name、product_brand、price、availability 等写入 Pydantic 模型；适合参考给 GLiNER2 增加“站点发现/页面路由→字段抽取→类型验证”的自动化采集外壳。 | https://docs.upsonic.ai/examples/business-sales/find-example-product |
| 该 ECIR 研究专门识别论坛噪声文本里的商品型号（用户常直接以 model number 指代商品），并用自训练 CRF 利用无标注目标域数据显著提升召回；适合为 GLiNER2 的 MODEL/MODEL_NUMBER 字段构造弱监督、拼写变体和长尾型号难例。 | https://doi.org/10.1007/978-3-319-16354-3_27 |
| Scrapfly 的 Product Extraction Schema 直接从非结构化商品页定义并抽取 brand、SKU、MPN、specifications、color、size、variants 等字段；适合参考 GLiNER2 的动态商品 Schema、网页抽取字段契约以及与结构化页面结果的交叉校验。 | https://scrapfly.io/docs/extraction-api/automatic-ai/models/product |
| Shopedia 可从普通商品文本自动拆出 Manufacturer、Family、Model Code、Model Suffix、Model Version、Model Generation、Model Year 与 MPN；适合直接校准 GLiNER2 中 brand/family/model/model_number 等相近字段的边界和归一化规则。 | https://shopedia.com/platform/product-categorization-enrichment-platform/product-attributes-specifications-24 |
| BlackFalconData 的 eBay Scraper 深度模式对真实 Listing 固定输出 brand、model、MPN、EAN 和完整 itemSpecifics；适合持续采集 GLiNER2 品牌/型号/属性回归样本，并用商城结构化字段做伪标签或抽取结果一致性校验。 | https://github.com/BlackFalconData-org/ebay-scraper |
| Basalam 的 product-catalog-generator 提供电商商品专用 LLM/VLM 微调代码、合成数据集和模型，直接从商品数据推断 product type 与 attributes；适合对照 GLiNER2 的类目路由、动态 Schema、多字段抽取和文本/图片联合富化方案。 | https://github.com/basalam/product-catalog-generator |
| 这篇电商目录案例从商品名/描述自动推断 brand、size、organic 等属性，并进一步做新品牌发现与实体归一；和 GLiNER2 的“字段抽取→品牌 taxonomy 扩充→标准化入库”链路高度吻合，适合参考质量校验与实体解析设计。 | https://www.rohan-paul.com/p/ml-case-study-interview-question-861 |
| Constructor 的生产级属性富化实践直接比较正则、NER 与 LLM 的文本方案，NER 示例覆盖品牌、商品名、尺码、颜色、材质等实体；适合参考 GLiNER2 作为通用文本主抽取器，并按字段搭配规则校验与 LLM 难例兜底。 | https://medium.com/constructor-engineering/attribute-enrichment-under-the-hood-acd10b8cf7a7 |
| Semantics3 的 Text Attribute Extraction 已把非结构化商品标题/描述之外的用户问答转成结构化属性，示例覆盖防水、年龄、表盘尺寸、数量、材质等；适合让 GLiNER2 将 Q&A/评论作为补充证据源，补齐缺失规格并做跨来源一致性校验。 | https://medium.com/datascience-semantics3/introducing-attribute-extraction-from-user-generated-content-315852e3b567 |
| Google 的 LANTERN 把网页属性抽取建模为 DOM 节点标注，并验证跨领域训练数据能提升新站点抽取；适合放在 GLiNER2 前面做商品页候选节点定位，把标题、描述、规格区域降噪后再按 brand/model/属性 Schema 做语义抽取。 | https://research.google/pubs/learning-transferable-node-representations-for-attribute-extraction-from-web-documents/ |
| Meesho 的生产实践针对卖家属性缺失、错误 taxonomy 和数千商品类目自动补全商品属性，并将模型结果直接接入上架流程；适合参考 GLiNER2 的“类目路由→属性 Schema→自动抽取→质量检查”规模化工程设计，并用视觉模型补文本不可见字段。 | https://medium.com/meesho-tech/we-automated-attribute-tagging-using-deep-learning-models-part-1-b5bc455d2305 |
| Shopify 2026 商品数据富化指南明确以供应商 SKU、model number、尺寸等原始数据为起点，再经过清洗、单位/类目标准化与技术属性补全形成可消费目录；适合定义 GLiNER2 抽取后的 brand/model/规格标准化、缺失字段治理和 PIM/商城入库边界。 | https://www.shopify.com/enterprise/blog/product-data-enrichment-ecommerce |
| CaptionQA 的电商子集包含真实商品页面/图片上的品牌、型号/型号编号、颜色、尺码、材质与规格问答，可直接构造 GLiNER2 + OCR/VLM 组合链路的多模态回归集，重点验证文本缺失时的品牌/型号补证与属性一致性。 | https://huggingface.co/datasets/Borise/CaptionQA |
| Kaufland 的大规模商品属性抽取系统把卖家已有结构化属性转成 QA/span 训练信号，从标题与描述中抽取目标字段，并用负样本学习“无答案”；非常适合迁移到 GLiNER2 的类目动态 Schema、缺失字段 abstain 与海量商品批处理评测。 | https://kaufland-ecommerce.com/blog/developing-a-large-scale-attribute-extractor-for-e-commerce/ |
| Zalando 的 Content Creation Copilot 已把 AI 属性抽取接入商品上新流程，并通过内部属性代码映射与人工确认控制质量；适合参考 GLiNER2 的“动态 Schema 抽取→平台字段映射→人工审核→目录写回”生产闭环。 | https://engineering.zalando.com/posts/2024/09/content-creation-copilot-ai-assited-product-onboarding.html |
| eBay 官方目录匹配规范明确把 Brand、MPN、Model、Color、Storage Capacity 等作为类目商品 aspects，并支持用 Brand+MPN 等标识匹配标准目录商品；适合把 GLiNER2 抽取结果直接用于目录实体匹配、字段校验和标准化入库。 | https://developer.ebay.com/api-docs/sell/static/inventory/matching-products.html |
| Aito 的开源电商 Demo 在 Product Filling 场景中利用已有商品名称、品牌等字段并行预测缺失类目属性，同时返回置信度和候选值；适合把 GLiNER2 的 brand/model 等确定性抽取作为上游，再做缺失属性补全与低置信度复核。 | https://github.com/AitoDotAI/aito-ecommerce-demo |
| PRAISE 会从商品评论与卖家描述中抽取并对比结构化属性信息，显式标记缺失、冲突和部分匹配并保留证据；适合把 GLiNER2 对标题/描述的 brand/model/规格抽取扩展到评论证据源，用于缺失属性补证和冲突字段复核。 | https://arxiv.org/abs/2506.17314 |
| Velou Commerce-1 是面向零售专门训练的商品模型，架构显式跟踪 color、material、fit、seasonality，并支持结构化属性生成与变体聚类；可作为 GLiNER2 在商品属性抽取、同款/变体归并和实时目录服务上的领域模型对照基线。 | https://www.velou.com/commerce1 |
| 该 Ministral 3B Magento 适配器用 5.6 万级电商指令训练，直接把商品文本或 Magento custom_attributes 按目标字段抽成 JSON，并提供本地 LoRA/GGUF 形态；适合与 GLiNER2 做 3B 级本地部署、结构化属性输出以及同一电商数据上的跨模型吞吐/精度对照。 | https://huggingface.co/gabrielgts/ministral3-3b-ec-magento |
| FewIE 是低资源 few-shot NER 评测框架，并内置 Zhang 等人的电商 NER 数据；适合评估 GLiNER2 在新增品牌、型号、规格字段只有极少标注时的样本效率，并作为固定标签 few-shot 基线。 | https://github.com/DFKI-NLP/fewie |
| 该项目先对电商商品名做类目路由，再在类目内从标题拆出 color、style、size、material、gender 等属性；适合参考 GLiNER2 的“类目→字段 Schema”裁剪思路，并作为短标题属性抽取的传统方法对照。 | https://github.com/JinghuiZhao/product-item-name-classification |
| Linkfox 的 Product Title Analysis Skill 会把商品标题按 brand、material、color 等字段做结构化拆解与关键词统计；适合对照 GLiNER2 的标题级字段 Schema、批处理输出以及字段聚合分析。 | https://github.com/linkfox-ai/linkfox-skills/blob/main/skills/linkfox-product-title-analyze/SKILL.md |
| AskPoly 将不同来源的商品引用统一解析到 canonical `brand|family|model` 键；适合接在 GLiNER2 的 brand/model 抽取之后做品牌-系列-型号归一、同款聚合和实体消歧。 | https://github.com/ask-poly/askpoly |
| Alibaba CIC 含 189 万级淘宝服饰交互记录，每条都带商品标题和由服饰专家人工标注的属性集合；适合构造 GLiNER2 中文商品属性回归集，验证真实标题噪声下的类目属性覆盖与抽取稳定性。 | https://github.com/ixuejiaozhao/Alibaba-Custermers-Interaction-Dataset |
| Catalog 的 2026 指南把商品数据富化拆成多源采集、字段抽取、类目/单位/变体归一，并强调抽取结果要按类目定义且可追溯来源；适合 GLiNER2 从供应商页面、PDF、Feed 中抽 brand/model/规格后，统一做证据留存与标准化再入库。 | https://www.getcatalog.ai/blog/product-data-enrichment-ai-commerce |
| Lasso 的 2026 商品数据富化指南从供应商 SKU、model number 等原始目录字段出发，强调优先补齐高价值结构化属性并统一数据格式；适合为 GLiNER2 设计 brand/model/SKU/规格字段优先级、清洗规则和分阶段目录治理策略。 | https://productlasso.com/en/blog/product-data-enrichment-2026 |
| GLiNER2 官方 Boundary proposer 基线显示候选组装重写后典型 CPU B=8 场景达到 8.36× 加速，并保留随机一致性、容量压力等回归检查；适合把商品 brand/model/长规格 span 抽取纳入生产吞吐压测和性能回归门禁。 | https://github.com/fastino-ai/GLiNER2/blob/main/docs/boundary_baseline.md |
| Amazon Search 的端到端系统从短商品查询中提取 brand、color、product type 等属性并用于属性推荐；适合评估 GLiNER2 在短查询品牌/属性 span 抽取后的下游用法，并参考属性缺失时的推荐补全链路。 | https://www.amazon.science/publications/query-attribute-recommendation-at-amazon-search |
| CMA-CLIP 在 Amazon 商品属性抽取任务上融合文本与图片并学习细粒度跨模态对齐，还能抑制无关模态；适合作为 GLiNER2 文本品牌/型号/规格抽取的图片补证和多模态鲁棒性参考。 | https://www.amazon.science/publications/cma-clip-cross-modality-attention-clip-for-text-image-classification |
| PGE 面向商品图谱中的自由文本属性值和噪声三元组做错误检测，联合文本与图结构验证字段可靠性；适合接在 GLiNER2 抽取后，对 brand/model/规格等入库结果做图谱级一致性校验和异常拦截。 | https://www.amazon.science/publications/pge-robust-product-graph-embedding-learning-for-error-detection |
| 该 Amazon 工作在商品变体检索中直接用 NLP 抽取 pack size，并据此校验同款不同包装的价格一致性；适合专项验证 GLiNER2 对包装数量、容量、单位等规格字段的抽取与变体归并业务价值。 | https://www.amazon.science/publications/study-on-price-consistency-regarding-pack-size-via-product-variant-retrieval-and-pack-size-extraction |
| Amazon 的文本属性校验方法会把 brand、product name、functionality、flavor 等短文本值与商品 profile 交叉验证，并专门面向少标注、多类目场景；适合放在 GLiNER2 抽取后做品牌/型号/属性值的二次真实性校验和低置信度拦截。 | https://www.amazon.science/publications/automatic-validation-of-textual-attribute-values-in-e-commerce-catalog-by-learning-with-limited-labeled-data |
| Amazon CIKM 2024 的商品搜索接口研究用 LLM、搜索日志和商品信息自动生成训练数据，再训练多任务 Schema 生成器把自然语言购物查询转换成结构化 API 条件；适合把 GLiNER2 的 brand/category/color/size 等动态字段抽取放到查询侧做对照，并参考低标注冷启动的数据生成方式。 | https://www.amazon.science/publications/building-natural-language-interface-for-product-search |
| Walmart 的线上对话电商 NER 联合识别 brand、product、size、quantity、unit 等商品实体，并展示 BERT 对未见商品和品牌/商品歧义的泛化；适合用作 GLiNER2 查询侧多字段抽取的生产基线，重点测试短查询中品牌与品名边界。 | https://medium.com/walmartglobaltech/joint-intent-classification-and-entity-recognition-for-conversational-commerce-35bf69195176 |
| Walmart Shopping Assistant 的生产 Search Refinement 模型用 BERT+BILOU 抽取 brand、size 等 facet 及其修饰关系；适合参考 GLiNER2 将属性值与操作意图拆分，构建“实体字段抽取 + 属性关系/约束”两阶段查询理解链路。 | https://medium.com/walmartglobaltech/understanding-conversational-search-refinement-queries-in-walmart-shopping-assistant-fc04e4f97532 |
| Walmart Retail Graph 从商品标题、描述和元数据抽取实体，再结合 product type 做实体链接，并对低置信结果加入治理/人工校验后写入商品知识图谱；适合把 GLiNER2 的品牌、型号、属性 span 接到实体标准化、类目上下文消歧和质量门禁。 | https://medium.com/walmartglobaltech/retail-graph-walmarts-product-knowledge-graph-6ef7357963bc |
| search-query-parser 会把原始电商搜索词直接生成结构化 JSON，字段包含 brand、category、color、size、price 等；可作为 GLiNER2 动态 Schema 在搜索词场景的生成式对照基线，比较严格 JSON、缺失字段和复合约束解析。 | https://huggingface.co/aagzamov/search-query-parser |
| amazon-product-api 的商品详情结果直接暴露 brand 与 model_number 等结构化字段；可用其采集真实 Amazon 商品标题/描述并把页面结构化字段当对照真值，持续构造 GLiNER2 品牌、型号和规格抽取回归样本。 | https://github.com/drawrowfly/amazon-product-api |
| Octaprice 每周发布跨电商站点商品数据，并明确标注可用于 product categorization 与 attribute extraction；适合持续补充 GLiNER2 的新鲜商品标题/描述语料，构造长尾品牌、型号和属性的回归集及数据漂移检查。 | https://github.com/octaprice/ecommerce-product-dataset |
| Google CSS FeedViz 的商品 Feed Schema 直接包含 brand、MPN、color 等字段及商品数据校验状态；适合把 GLiNER2 抽出的品牌、型号/料号和属性映射到标准 Feed 字段，并参考校验问题设计入库质量门禁。 | https://github.com/google-marketing-solutions/css-feedviz |
| Cromwell 的 B2B AI 搜索支持按 partial SKU、manufacturer part number、technical specification 等混合条件找商品，并与 Akeneo PIM 集成；适合把 GLiNER2 用在采购查询侧抽取品牌/MPN/规格，再映射到 PIM 字段做候选检索与实体匹配。 | https://elogic.co/projects/cromwell/ |
| Adobe LLM Optimizer 的 Catalog Agent 会读取每个 SKU 的技术属性、类目上下文、变体及现有名称/描述并执行目录富化；适合参考 GLiNER2 抽取后的“结构化商品字段→目录富化→跨渠道消费”生产闭环与字段完整度治理。 | https://experienceleague.adobe.com/en/docs/llm-optimizer/using/dashboards/opportunities/enrich-product-catalog |
| Lasso 给出按类目约束的商品标题模板，核心公式直接包含 Brand、Model、Key Spec、Connectivity、Color；适合用来定义 GLiNER2 的品牌/型号/规格字段边界，并批量生成字段顺序变化、缺失字段和压缩标题等回归测试样本。 | https://productlasso.com/en/blog/product-title-templates-by-category |
| Arovon 专门把供应商 PDF、目录和 datasheet 转成结构化商品记录，直接抽取 SKU、manufacturer part number、product family、attributes、dimensions、material 等字段；适合把 GLiNER2 接到 PDF 文本解析后做品牌/型号/MPN/规格提取，并保留来源上下文供人工校验。 | https://arovon.com/ai-product-data-extraction-from-pdfs |
| omkarcloud/amazon-scraper 从 Amazon PDP 返回 brand、item model number、technical details、product details、variants 等大量结构化字段；可把商城结构化结果作为 GLiNER2 从标题/描述抽取品牌、型号与规格的持续对照真值，并专项测试字母数字型号与变体字段。 | https://github.com/omkarcloud/amazon-scraper |
| Inference.net 的商品数据抽取指南把任意电商 HTML 转为 Pydantic/Zod 约束的 typed JSON，Schema 直接含 brand、SKU/GTIN、specs、variants，并强调字段描述、缺失值和验证；适合参考 GLiNER2 动态 Schema 定义、空值处理和网页批量抽取契约。 | https://inference.net/content/product-data-extraction/ |
| DataWeBot 的独立商品数据抽取方案把 brand、manufacturer、model number、SKU/ASIN/UPC/EAN、尺寸和技术规格统一结构化；适合用来定义 GLiNER2 输出字段契约，并设计型号/标识符与规格的后处理、标准化及 PIM 入库。 | https://www.datawebot.com/services/product-data-extraction |
| MyDataScraper 的电商抽取字段表把 Product Identity 明确拆成 Title、Brand、SKU、ASIN、Model Number、Category，并同时保留 Technical Specs 和 Size/Colour variants；适合校准 GLiNER2 的品牌/型号/平台标识边界与变体 Schema。 | https://mydatascraper.com/ecommerce-data-scraping-services/ |
| Meesho AI Data Challenge 公开按 Category + Attribute Keys/Values 组织的真实电商商品数据，并以属性级 micro/macro F1 评测；适合把 GLiNER2 动态 Schema 评测扩展到类目专属属性集合，也可用于文本主抽取与图片属性兜底的对照。 | https://www.meesho.io/ai/data-challenge |
| Akeneo 2026 Spring 的 Smart Attribute Discovery 会自动生成属性结构（label/code/type/scope/config）并给出值/选项建议、审核与 API 流程；适合借鉴 GLiNER2 在新类目下“候选字段发现→人工审核→动态 Schema→持续迭代”的治理机制。 | https://akeneo.helpjuice.com/2026/april-2026-serenity-updates |
| 该 2026 专利把规则、NER 与 GenAI 组成混合商品目录抽取链路，从多语言非结构化/半结构化商品数据提取品牌、规格、尺寸、材质等属性并映射为 attribute-value pairs；适合把 GLiNER2 作为其中 Schema/NER 主抽取器，参考其模型分工、验证和目录映射。 | https://patents.justia.com/patent/20260064648 |
| 该 2026 汽配项目把 Air Lift 与 Timbren 的原始商品标题解析为 year、make、model、product type 并输出结构化表；适合做 GLiNER2 在汽配短标题中型号/产品类型抽取的规则基线和回归样本来源。 | https://github.com/abdurrehman616/vehicle-parts-scraper |
| Asset Information Extraction System 以产品 model number 和资产分类为入口，联网检索产品资料后用 LLM 提取结构化数据；很适合接在 GLiNER2 的 model/model number 抽取之后，形成“型号识别→联网补证→属性富化”的完整链路。 | https://github.com/Bhanuraj23m0316iitb/Asset-Information-Extraction-System |
| Amharic-Ecommerce-NER 专门从阿姆哈拉语非结构化电商文本识别商品名、品牌、价格、数量、商家等实体；可作为 GLiNER2 多语言商品品牌/属性抽取的低资源语言对照基线，并检验动态 Schema 的跨语言泛化。 | https://github.com/Habtamu91/Amharic-Ecommerce-NER |
| Olostep × Merchkit 案例把多零售商 PDP/规格页统一按固定 Schema 抽成确定性 JSON，字段直接包含 title、brand、model、dimensions、materials、variants，并可批量处理数万 URL；很适合参考 GLiNER2 的“网页采集→品牌/型号/规格抽取→跨来源统一 Schema→批量目录富化”生产链路。 | https://www.olostep.com/blog/how-merchkit-automates-catalog-enrichment-with-olostep |
| Deep Mist AI 的 5 万 SKU 实战用双模型流水线做属性抽取、标准化与 taxonomy 补全：一模型负责图文语义推断，另一模型按目标 Schema 输出结构化字段，并把模型分歧送人工审核；适合给 GLiNER2 设计“主抽取→交叉验证→低置信/冲突复核→批量入库”的质量闭环。 | https://deepmist.ai/case-studies/catalog-intelligence |
| Extralt 将不同商城商品统一成 taxonomy、attributes、identifiers 与精确 variant 记录，并在有标识符时用 GTIN 或 brand+MPN 做跨卖家同款归一；适合把 GLiNER2 抽出的 brand/model/MPN/颜色/尺码接到 canonical product/variant 层，做跨站实体消歧与变体对齐。 | https://extralt.com/use-cases/product-data |
| GigaCommerce 的 2026 目录富化 Playbook 把“描述中埋藏的规格→独立结构化字段”、类目级必填属性 Schema 和兼容性关系作为核心治理项；适合用来定义 GLiNER2 各类目的 brand/model/规格必抽字段、缺失字段检测及抽取覆盖率指标。 | https://gigacommerce.co/insights/catalog-for-ai/catalog-enrichment-for-ai-playbook |
| AdYogi Catalog.ai 从网站、表格、Feed 和图片中识别类目相关属性并输出结构化字段，再映射到 Amazon/Flipkart/Myntra 等平台 taxonomy 与合法取值；适合参考 GLiNER2 的“多来源输入→动态类目 Schema 抽取→字段标准化→渠道/PIM 映射”落地方式。 | https://www.adyogi.com/catalog-ai |
| 该研究围绕时尚电商构建商品属性体系，覆盖品牌、颜色、材质、版型等字段，并使用 LLM 做零样本属性抽取；适合参考 GLiNER2 的类目级 Schema 设计、多字段抽取和长尾描述归一化。 | https://link.springer.com/article/10.1186/s40691-025-00435-w |
| 该零售数字孪生框架会从单商品图像中识别 brand、product name、variant、volume/size 等信息；适合作为 GLiNER2 文本抽取的多模态兜底，并用于验证包装图上品牌、型号/变体和规格字段的证据一致性。 | https://link.springer.com/article/10.1007/s44163-026-01221-3 |
| Unidata 的电商商品归组案例明确把“从商品描述提取 model name”作为关键步骤，并要排除材质、颜色、袖长等非型号属性；适合设计 GLiNER2 的 model 与普通属性边界规则，并把抽取结果直接接到同款归组/去重。 | https://unidata.pro/cases/product-grouping-for-e-commerce/ |
| 该商品匹配项目用 spaCy NER 从商品别名中抽取 BRAND、FORM、DOSAGE、QUANTITY 等关键属性，再结合向量检索映射到标准 SKU；适合参考 GLiNER2 的“字段抽取→属性加权→SKU 匹配”下游链路，并验证抽取字段对同款召回的业务价值。 | https://github.com/Sahori2003/product-matching |
| product_tagger 专门把电商商品标题拆成核心商品词、brand 和描述性修饰词，可作为 GLiNER2 的轻量标题解析基线，尤其适合对照品牌、商品类型与颜色/性别/材质等修饰属性的边界切分。 | https://pypi.org/project/product-tagger/ |
| Amazon-M2 的多语言商品记录同时提供 title、brand、colour、description 等结构化字段，覆盖英语、德语、日语及法语、意大利语、西班牙语；可将目录字段转成弱标签，构建 GLiNER2 多语言品牌/颜色抽取与跨语言泛化回归集。 | https://github.com/haitaomao/amazon-m2 |
| ChocoData 的 Amazon 商品抓取项目提供当前可复现的真实 PDP 结构化样本，输出 title、brand、product_details/technical_details 等字段并保留原始响应；适合持续构造 GLiNER2 品牌与规格抽取回归集，并用页面结构化字段定位品牌槽位和规格解析误差。 | https://github.com/ChocoData-com/amazon-product-scraper |
| ShopSavvy Python SDK 支持按 model number、条码、ASIN 或商品 URL 反查统一商品记录，并返回品牌等结构化字段且支持批量查询；适合把 GLiNER2 抽出的型号/品牌作为二阶段检索键，做跨零售商实体确认、型号纠错和目录归一。 | https://github.com/shopsavvy/sdk-python |
| Google Product Feed Enrichment Kit 会统一识别 GTIN/EAN/UPC、MPN/manufacturer_part_number 与 brand/manufacturer 等字段别名并检查标识符是否存在；适合接在 GLiNER2 后做品牌、MPN/型号字段归一和 Feed 入库前完整性校验。 | https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-identifier-exists.md |
| Product Research Agent 已实现“商品搜索→品牌/规格抽取→Pydantic 结构化验证”的 Agent 流程，并支持从购物搜索结果汇总证据；适合给 GLiNER2 的低置信品牌、型号和规格增加联网补证与结构化复核层。 | https://github.com/scarnyc/product-research-agent |
| Bright Data 的 Amazon Global Dataset Scraper 输出商品 title/description 的同时提供 brand、manufacturer、model_number、Model Name、颜色、容量等结构化字段；适合持续生成 GLiNER2 品牌/型号/规格回归样本，并用商城结构化值自动做字段级差异校验。 | https://github.com/luminati-io/amazon-products-global-dataset-scraper |
| Meta Business SDK 的 ProductItem 目录对象直接定义 brand、GTIN、manufacturer_part_number、material、color 等字段；适合把 GLiNER2 的品牌、MPN/型号和属性输出映射到实际电商目录 Schema，并据目录字段约束设计入库质量门禁。 | https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/productitem.py |
| X5 Tech 的零售搜索 NER 实战直接从短且噪声较大的商品查询中抽取 TYPE、BRAND、VOLUME、PERCENT，并围绕线上速度与准确率设计流水线；很适合作为 GLiNER2 品牌、商品类型、容量/百分比等字段在真实搜索词噪声下的生产基线与回归参考。 | https://habr.com/ru/companies/X5Tech/articles/941634/ |
| Naratix 的电商目录富化案例从原始商品 Feed/非结构化输入按类目抽取 size、color、material、weight、dimensions、technical specs 等字段，再做 canonical Schema 归一化、同义值映射和验证回流；适合参考 GLiNER2 的“类目 Schema→多属性抽取→标准化→质检/反馈”完整落地闭环。 | https://agixtech.com/case-studies/naratix/ |
| Agility AI 的生产案例用 NER 从多语言、格式混乱的商品/运输描述中抽取 Brand、Type、Processing、Grade、Form、Origin，并采用离线 ONNX INT8 部署与字段覆盖率评估；适合参考 GLiNER2 品牌/类型等字段的本地化部署、噪声文本鲁棒性和逐字段覆盖率门禁。 | https://agilitytech.ai/case-studies/hsn-classification |
| gliner-fahrzeugschein-onnx 直接基于 GLiNER2 训练并导出 ONNX，Schema 明确抽取 brand、model、VIN、功率、排量、重量、颜色等字段；虽然场景是车辆证件，但与商品“品牌+型号+结构化属性”抽取高度同构，适合参考 GLiNER2 字段定义、领域微调、ONNX 部署及字母数字型号识别。 | https://huggingface.co/morilinger/gliner-fahrzeugschein-onnx |
| Carl Fung 的奢侈品转售实战把商品照片直接抽成 brand、specific model、material、color、price 等结构化字段并批量写入 Google Sheets；适合参考 GLiNER2 文本主抽取配合视觉兜底后的“字段抽取→可编辑复核→表格/库存入库”业务链路。 | https://carlfung.dev/blog/building-ai-product-identifier |
| DataWeBot 的 NLP 商品管线直接从标题/描述抽取 brand、model、size、material 等字段，示例可把 Samsung…QN65Q80D 拆成品牌、型号、尺寸和显示规格，并配置信度及人工复核；适合参考 GLiNER2 的多语言字段 Schema、低置信度门禁和目录富化闭环。 | https://www.datawebot.com/solutions/nlp-product-categorization |
| Octoparse 的 Amazon Details Scraper 给出可直接落地的商品字段契约，统一输出 Brand、Manufacturer Part Number、Item Model Number、UPC、规格及 color/size 变体；适合用这些页面结构化字段作为 GLiNER2 品牌/型号/规格抽取的对照真值和跨站回归样本。 | https://www.octoparse.com/template/amazon-details-scraper |
| Hiflylabs 用 BigQuery + 多模态 AI 批量校验商品标题与图片，对 brand、size、model 缺失/冲突判 MISMATCH，并输出置信度和原因；适合接在 GLiNER2 后做品牌/型号/规格的图文一致性校验与低置信度人工复核。 | https://hiflylabs.com/blog/2025/9/23/validating-ecommerce-products-bigquery-multimodal-ai |
| SIGIR eCom 的商品标题研究把短标题生成直接建模为多类 NER，显式从商品标题抽 BRAND、FUNCTION、VARIATION、SIZE、COUNT 并用 BIO 标注训练；适合用作 GLiNER2 短标题多字段 span 抽取的传统基线和字段边界回归集设计参考。 | https://sigir-ecom.github.io/ecom2019/ecom19Papers/paper36.pdf |
| Knowledgator 的电商查询解析 Cookbook 直接用 GLiNER 按 product_name、brand、category、color 等电商标签做零样本实体抽取；适合直接参考 GLiNER/GLiNER2 在商品搜索词中的 Schema 设计、字段解析和查询结构化落地。 | https://docs.knowledgator.com/docs/cookbooks/enterprise-search-parsing/index.html |
| 该实践把 GLiNER 部署成 AWS SageMaker NER REST API，并以 Brand、Product 等标签做在线抽取；适合参考 GLiNER2 品牌/商品字段服务化、API 封装、云端推理和生产部署方式。 | https://dimitarmitkov.com/blog/ner-rest-api/ |
| wdc-pave-ave 将商品标题/描述与 gold_json 对齐，类目字段直接包含 Brand、Model Number、Manufacturer Stock Number 等；很适合构建 GLiNER2 品牌/型号/货号/规格的结构化回归集，并逐字段评测缺失值。 | https://huggingface.co/datasets/siavashsaki/wdc-pave-ave |
| 该 Amazon 服饰鞋履珠宝数据集同时提供商品文本与 brand、manufacturer、item_model_number、color、material、size 等目录字段；适合批量生成 GLiNER2 的真实商品品牌/型号/属性弱标签和跨类目回归样本。 | https://huggingface.co/datasets/smartcat/Amazon_Clothing_Shoes_and_Jewelry_2023 |
| ScrapeGraphAI 的商品抽取提示工程指南把标题解析目标明确为 brand + model + key spec，并强调结构化输出；适合用作 GLiNER2 Schema 字段定义、网页商品文本清洗和 LLM 对照基线的工程参考。 | https://scrapegraphai.com/blog/prompt-engineering-guide |
| Width.ai 的商品匹配实践把 brand、model number、size、condition、color 等抽取字段作为同款匹配信号，并结合标题信息抽取；适合验证 GLiNER2 抽出的品牌/型号/属性能否直接提升商品去重、归并和跨站匹配。 | https://www.width.ai/post/product-matching-in-ecommerce |
| 该项目用 AWS Textract OCR 配合 spaCy 自定义 NER，从商品图片文字中识别 product name、brand、price；适合在 GLiNER2 文本链路前增加包装/图片 OCR 兜底，再按 brand/model/规格 Schema 做统一抽取与校验。 | https://github.com/jeevan251203/Product_Info_Extraction_using_OCR |
| 该项目直接从商品描述抽取 attribute-value pairs，并专门讨论 noisy descriptions 与 seed data generation 难题；适合用作 GLiNER2 在脏商品描述、少标注/弱监督场景下的传统基线和困难样本构造参考。 | https://github.com/vamsilnm/Attribute-Value-Extraction |
| Ferret 是开源声明式数据抽取运行时，商品示例可从网页直接取得 brand/title/price，并支持按 Schema 对 HTML 做 AI 结构化抽取；适合放在 GLiNER2 前面做网页候选字段定位与统一输入，减少 DOM 噪声。 | https://ferretlang.org/ |
| Match Data Studio 的工程实践会把“Samsung 65-inch 4K Smart TV QN65Q80C”拆成 brand、size、resolution、category、model number，再结合确定性规则、Embedding 和 LLM 做匹配确认；适合把 GLiNER2 抽出的品牌/型号/规格直接接到商品去重、实体匹配和低置信复核。 | https://match-data.studio/blog/deterministic-vs-probabilistic-matching/ |
