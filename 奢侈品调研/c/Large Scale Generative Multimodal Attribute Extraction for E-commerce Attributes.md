# Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes

> 分析人：c  
> 目标需求：跨源二奢/腕表商品实体匹配（雷小安 × 腕表之家 × 奢当家）  
> Notion Spec：`https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31`  
> 核心约束：100 万–1000 万级、持续增量、字段稀疏、reference 可能在标题/详情/图片中；“同一个商品”定义为**同一 reference number / 型号**；**precision 极端优先，绝不能误匹配，允许漏匹配**。

## 1. 选题与去重

本次从 `奢侈品文章调研.md` 选择了以下尚未由 c 分析的论文：

- 论文：**Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes**
- 作者：Anant Khandelwal, Happy Mittal, Shreyas Kulkarni, Deepak Gupta
- ACL 2023 Industry Track
- 论文主页：https://aclanthology.org/2023.acl-industry.29/
- PDF：https://aclanthology.org/2023.acl-industry.29.pdf
- DOI：https://doi.org/10.18653/v1/2023.acl-industry.29

执行前已检查 `奢侈品调研/c/`。此前 c 已分析 Ameli、AnyMatch、Confidence Classifiers、DeepBlocker、Efficient Model Repository、End-to-end multi-modal product matching、Selective Entity Matching、GraLMatch、How to Fix a Broken Confidence Estimator、Multi-Value-Product RAG、PAM、Progressive Fine-Tuning、Tailoring entity resolution、TransClean、Using LLMs for Extraction and Normalization、parts-distributor-sku-classifier、pyJedAI 等条目；目录中不存在本论文，因此本次不重复。

选择这篇论文的原因是：当前 Spec 的“匹配”本质上可以拆成两个问题：

1. 从三个来源的稀疏文本、结构化字段、图片中，**高精度恢复 canonical reference**；
2. 在 reference 已可靠恢复后，用严格等值规则完成跨源实体归并。

相比直接训练一个 pairwise matcher，第一步更决定最终 precision。MXT 是已经做过大规模生产部署的多模态属性抽取架构，论文报告覆盖 6 个英语市场、超过 10K/12K 个 `(product-type, attribute)` 组合，并抽取超过 150M 个属性值，因此它非常适合参考“如何把一个多模态属性抽取模型扩展到生产规模”。

但是本需求不能原样照搬 MXT。MXT 的目标是最大化高精度区间的属性覆盖率，而 reference 是**身份标识符**，任何一个字符的生成幻觉都可能造成错误合并。因此最重要的改造是：

> **借 MXT 的多模态证据融合与多任务共享能力，但把“开放式生成 reference”改造成“候选抽取 + canonical registry 受限解析 + 冲突否决 + 可拒识决策”。模型只能解释或打分证据，不能凭空创造可用于自动合并的 reference。**

---

## 2. 论文解决的问题与生产指标

论文把电商属性抽取定义为一个多模态 Q&A/生成任务：

```text
Question = attribute name
Context  = product type + product text + product image
Answer   = attribute value
```

例如：

```text
attribute = neck style
product_type = dress
text = ...
image = ...
=> answer = Ruched Neck
```

作者认为生产级属性抽取系统应具备四个能力：

1. **Scalability**：一个模型处理大量 product-type/attribute，而不是一属性一模型；
2. **Multi-modality**：同时利用文本和图片；
3. **Zero-shot inference**：训练没见过的属性值也能输出；
4. **Value-absent inference**：值不在文本里、只可从图片推断时也能输出。

论文的核心生产指标不是单纯 F1，而是 `Recall@90P`：在 precision 固定到 90% 的情况下比较召回率。这一点对当前 Spec 很有启发，但腕表 reference 的目标 precision 应远高于 90%，更适合使用 `Recall@99.9P`、`Recall@99.99P` 或在黄金集上直接限制 false positive 的统计上界。

论文报告：

- 自建 30 product-type 数据集中包含 38 个唯一属性；
- 训练集约 569K 商品，验证集约 84K，测试集约 73K；
- E-commerce5PT 基准上包含 22 个 product-attribute；
- 相比 NER-MQMRC / CMA-CLIP，MXT 在相同 90% precision 下显著提升 recall；
- 一个模型可以跨多个 PT-attribute 共享；
- 已在 6 个英语市场部署，覆盖 >10K（论文摘要/主页描述为约 12K）PT-attributes；
- 已抽取 >150M 属性值。

对于本需求最值得注意的不是具体提升百分点，而是：**作者已经遇到多任务阈值、脏标签、值归一化、模型刷新、数万属性对监控等真正的生产问题，并给出了可扩展路径。**

---

## 3. MXT 技术实现与架构拆解

## 3.1 总体数据流

MXT 可以概括为三段：

```text
                         ┌──────────────────────────────┐
attribute name ─────────►│                              │
product type ───────────►│ T5 text embedding           │
product text ───────────►│                              │
                         └──────────────┬───────────────┘
                                        │
product image ─► ResNet-152 global ─────┼─► MAG
                                        │   image-aware text embedding
                                        ▼
                                  T5 Encoder
                                        │
                                        ▼
                              encoded text sequence
                                        │
product image ─► Xception local/attribute-aware visual features
                                        │
                                        ▼
                              cross-attention fusion
                                        │
                                        ▼
                                  T5 Decoder
                                        │
                                        ▼
                              generated attribute value
```

它不是简单把“图片 embedding 拼到文本 embedding 后面”，而是做了两次不同目的的视觉融合：

1. **MAG：让每个文本 token 的语义被整张商品图修正**；
2. **Xception + cross-attention：让模型围绕具体 attribute 去关注图片中最相关的局部视觉模式**。

这种“双层视觉融合”值得参考，因为腕表图片里不同区域的证据价值差异极大：

- 表背刻字：reference 强证据；
- 保卡/证书：reference 强证据；
- 吊牌：reference 中强证据；
- 表盘：品牌/系列/外观辅助证据；
- 整表外观：只能做兼容性检查，不能证明同 reference。

因此落地时我们同样应分成“全局商品语义”和“局部证据区域”两层，而不是把所有图片平均池化成一个向量。

## 3.2 输入序列：attribute name + product type + product text

MXT 将下面三部分编码成一个文本序列：

```text
[attribute-name] [product-type] [textual-description]
```

论文把 attribute extraction 写成 Q&A：attribute 是问题，product type + 描述是上下文。

对腕表系统，可以改成更结构化的 prompt/schema：

```text
TASK=reference_extraction
BRAND=Rolex
CATEGORY=watch
SOURCE=雷小安
TITLE=劳力士潜航者 ... 126610LN ...
DESCRIPTION=...
STRUCTURED_REFERENCE=...
```

这里 product-type 不应只有“watch”，而应注入更多有助于约束候选空间的上下文：

- `brand_id`
- 可选的 `collection_id`
- `source_id`
- `record_type`：整表 / 表带 / 配件 / 盒证 / 零件
- `locale/language`

但要注意：这些上下文字段只是帮助模型判断“哪个字符串是 reference”，**不能改变 reference 的字符内容**。

## 3.3 MAG：把视觉信息注入文本 token

MXT 使用 Multimodal Adaptation Gate（MAG）。对第 `i` 个文本 token embedding `T_i` 和 ResNet-152 提取的全局图像向量 `V_R`，简化后的核心过程是：

```text
g_i = ReLU(W_g [T_i ; V_R] + b_g)
H_i = g_i · (W_H V_R) + b_H
T'_i = T_i + α H_i
F_i  = Dropout(LayerNorm(T'_i))
```

其中 `g_i` 是门控向量。不同 token 会选择不同视觉信息，因此图片不是统一地加到整句话上，而是“按 token 条件化地修正文本表示”。`α` 还通过范数比值限制视觉位移幅度，避免视觉向量完全压过文本语义。

这对本需求的启示是：视觉证据要有**门控和上限**。尤其是 reference matching：

```text
文本/OCR reference 决定身份
        │
        ├─ 图片支持：提高证据可信度
        ├─ 图片不确定：保持原判
        └─ 图片明确冲突：降级到人工复核
```

不能变成：

```text
文本 reference 不一致 + 图片很像 => 自动判同款
```

因此落地版不需要在 embedding 层完全照搬 MAG，但要保留“视觉只能调节证据置信度、不能越权覆盖硬标识符”的设计原则。

## 3.4 ResNet-152 分支：全局商品图语义

论文使用预训练 ResNet-152，为每张商品图产生一个全局 visual embedding，再送入 MAG。

对腕表场景，建议把这个分支从“通用图片向量”改为更有生产价值的 `image_role_classifier + global compatibility encoder`：

```text
image -> image_role_classifier
      -> dial / caseback / warranty_card / hang_tag / certificate / box / accessory / other

image -> global encoder
      -> brand/collection/record_type compatibility score
```

这样全局视觉分支主要解决：

- 这张图是不是表背，是否值得跑高成本 OCR；
- 是否明显不是目标品牌；
- 商品到底是整表还是表带/盒证/配件；
- 图片和文本的品牌/大类是否冲突。

不要让它直接输出“这就是 126610LN”。

## 3.5 Xception 分支：attribute-aware 局部视觉特征

论文第二个视觉通路使用 Xception。Xception 的 depthwise separable convolution 可以学习不同局部视觉特征，再通过 cross-attention 与 T5 encoder 的文本表示融合。

作者的直观例子是：当问 `sleeve type` 时，模型应该关注衣服袖子，而不是整张图。

腕表可以把这个思想直接替换成“reference evidence region attention”：

```text
TASK=reference
    ├─ caseback engraving ROI
    ├─ warranty card model/reference ROI
    ├─ hang tag reference ROI
    ├─ certificate reference ROI
    └─ dial/bracelet visual region（仅辅助）
```

2026 年落地无需坚持 Xception；更适合使用：

- 文档/场景文字 detector；
- OCR encoder；
- ViT/SigLIP/CLIP 类视觉 encoder；
- ROI crop encoder；
- cross-attention 或 late-fusion scorer。

真正需要继承的是“**目标属性决定关注哪个图片区域**”，不是模型名称。

## 3.6 Cross-attention：文本问题决定看哪个视觉区域

论文把 Xception 产生的视觉特征与 T5 encoder 输出做 multi-head cross attention，得到 attribute-aware fused representation。

可以理解为：

```text
Query  = “我要找哪个 attribute” + 商品文本语义
Key/Value = 图片局部区域
```

对 reference，我们可以把 Query 进一步显式化：

```text
Query = brand + collection + candidate references + OCR/text spans
```

例如候选集是：

```text
126610LN
126610LV
116610LN
116610LV
```

模型只回答：

```json
{
  "supported_candidates": ["126610LN"],
  "contradicted_candidates": ["126610LV"],
  "evidence_regions": ["caseback_crop_03", "card_crop_01"],
  "abstain": false
}
```

而不是生成一个任意新字符串。

## 3.7 T5 Decoder：为什么论文里合理、在 reference 场景却必须受限

MXT 最后用 T5 decoder 自回归生成属性值：

```text
P(y_j | y_<j, x, I)
```

训练目标是标准 token-level negative log likelihood。

这对 `color=red`、`neck style=Ruched Neck` 这类开放属性非常自然，也使模型具备 zero-shot/value-absent 能力。

但 reference number 的风险完全不同：

- `126610LN` 和 `126610LV` 只差两个字符，但代表不同 reference；
- `116500LN` 与 `126500LN` 一个数字错位就可能造成错误归并；
- BPE/SentencePiece 会把字母数字串切成多个 subtoken；
- beam/greedy 生成都可能产生训练中不存在、甚至品牌从未发行过的“合法外观字符串”。

所以本项目**禁止直接把生成文本当自动合并依据**。

建议的 decoder 改造顺序：

1. `span copy`：优先复制标题/详情/OCR 中已出现的完整字符 span；
2. `candidate classification`：在 canonical registry 检索出的有限候选中分类；
3. `constrained decoding`：如果必须生成，只允许输出候选 trie 中存在的 token 路径；
4. 生成模型产生的新字符串仅标记为 `unresolved_candidate`，永不自动合并。

---

## 4. 论文训练方法：Distant Supervision 的价值与风险

## 4.1 为什么作者用 distant supervision

大规模电商属性抽取最大的问题是人工标注成本。MXT 直接利用 catalog 中已有属性作为训练标签，构造 distant supervision 数据，从而避免为海量 PT-attribute 单独人工标注。

这非常适合当前三源腕表系统，因为我们已经有天然弱标签：

- 平台结构化 `reference/model` 字段；
- 标题中高置信 exact reference；
- 多字段一致的 reference；
- 同来源历史人工修正；
- 品牌官方/可信型号表。

可以构造：

```text
HIGH_CONF_WEAK_POSITIVE
- structured_reference == exact canonical reference
- title contains same reference span
- brand agrees
- record_type == whole_watch
```

作为 reference extractor 的弱监督正样本。

## 4.2 作者遇到的脏标签问题

论文部署章节明确提到 catalog 中的属性值存在大量不规范和 junk value，作者使用启发式合并相似值、并裁剪长尾值。

这对 reference 特别危险：

普通属性可以把：

```text
navy blue
navy-blue
Navy
```

近似归并；

reference 却不能随便把：

```text
126610LN
126610LV
```

看成“相似值”。

因此当前项目的 reference normalization 必须改成**registry-backed normalization**：任何字符变换只有在 canonical reference registry 中能唯一落到一个真实 reference 时才允许。

安全/相对安全操作：

```text
Unicode NFKC
全角 -> 半角
大小写统一
首尾空白清理
明显排版分隔符规范化（必须 registry 验证）
```

危险操作：

```text
任意 edit distance <= 1 自动纠错
O <-> 0 直接替换
I/l/1 直接替换
去掉所有后缀
只取数字前缀
模糊相似度超过阈值自动合并
```

这些只能用于**候选召回**，不能用于最终自动匹配。

---

## 5. 论文的生产经验如何映射到本 Spec

## 5.1 多任务共享：从 `(product-type, attribute)` 改成 `(brand/source, extraction-task)`

MXT 证明可以用一个共享模型处理大量 PT-attribute，避免维护成千上万个单独模型。

腕表系统可以设计一个共享 `ReferenceEvidenceModel`，任务由 prompt/schema 控制：

```text
brand = Rolex / Omega / Cartier / ...
source = 雷小安 / 腕表之家 / 奢当家
record_type = whole_watch / accessory / document
attribute = reference
```

共享底座学习：

- 什么样的字母数字串像 reference；
- reference 常出现在什么上下文；
- OCR 噪声模式；
- 标题中“适配 XXX 型号”“附件 for XXX”等反例语境；
- 品牌、系列、型号之间的共性关系。

品牌特有规则则放在 registry 和 rule layer，不写死在模型参数中。

## 5.2 多模态不是“图片投票”，而是“证据补全 + 冲突检测”

论文强调 value-absent inference，即文本没有值时从图片推断。

对于腕表 reference，建议分级：

```text
A 级证据：结构化 reference 字段 + registry exact hit
A 级证据：标题/详情中 exact canonical reference span
A 级证据：保卡/证书/吊牌 OCR exact canonical reference
B 级证据：表背 OCR exact/唯一可解析 canonical reference
C 级证据：品牌、系列、材质、表径等属性兼容
D 级证据：纯视觉外观相似
```

自动归并原则：

```text
必须至少存在 A 级 identity evidence；
C/D 只能支持或否决，不能单独建立 match；
任何两个 A 级证据互相冲突 => reject/人工复核；
```

## 5.3 阈值不应全局统一

MXT 生产中需要为大量 PT-attribute 找到达到目标 precision 的阈值。作者指出逐个属性人工标注几百条不可行，因此用 catalog 规则做自动评估辅助。

当前系统也不能只设一个：

```text
score >= 0.97 => match
```

阈值至少要按以下维度分桶：

```text
brand
source
extraction_source (structured/title/ocr)
record_type
reference_pattern_family
```

例如 Rolex reference 的字符结构、Omega reference 的点分数字、Cartier reference 的 CR 编号/型号表现不同；雷小安和腕表之家的标题模板也不同。

但更重要的是：最终自动合并不应该由“模型分数阈值”独立决定，而应该是**硬规则 + 置信度 + 冲突状态**联合决定。

---

## 6. 面向当前 Spec 的直接落地架构

建议把系统拆成 8 个服务/逻辑层：

```text
┌─────────────────────────────────────────────────────────────┐
│ 1. Source Ingestion                                        │
│ 雷小安 / 腕表之家 / 奢当家，full + incremental              │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Canonical Preprocess                                    │
│ text normalization / brand resolver / record type classifier│
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Reference Observation Extractor                         │
│ structured field / title span / description span / OCR      │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Canonical Reference Registry                            │
│ brand + reference + aliases + safe normalization rules      │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. MXT-inspired Evidence Scorer                            │
│ text + image-role + ROI/OCR + candidate references          │
│ output: support / conflict / abstain                         │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. Precision-First Decision Engine                         │
│ hard exact rule + evidence policy + conflict veto           │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. Entity Index / Cluster                                  │
│ key = brand_id + canonical_reference                        │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. Audit + Human Review + Feedback                         │
│ hard negatives / corrections / threshold calibration       │
└─────────────────────────────────────────────────────────────┘
```

下面逐层说明。

---

## 7. Layer 1：Source Ingestion

统一三源输入为稳定 schema：

```json
{
  "source": "leixiaoan",
  "source_item_id": "123456",
  "crawl_version": "2026-08-18T05:00:00Z",
  "title": "...",
  "description": "...",
  "structured": {
    "brand": "...",
    "model": "...",
    "reference": "..."
  },
  "images": [
    {"url": "...", "sha256": "..."}
  ],
  "raw_payload_uri": "..."
}
```

主键应使用：

```text
(source, source_item_id, crawl_version)
```

不能用 title/image 作为商品主键。

持续增量时，只对发生变化的字段重跑相应 extractor：

```text
text changed  -> text reference extractor
images changed -> image role + OCR
brand changed -> brand resolver + all reference resolution
```

---

## 8. Layer 2：Brand Resolver + Record Type Classifier

在 reference 解析前先确定：

```text
canonical_brand_id
record_type
```

`record_type` 至少分：

```text
whole_watch
watch_head
strap
bracelet
box
warranty_card
certificate
accessory
part
unknown
```

这是降低误匹配的关键。标题中可能写：

```text
“适配 Rolex 126610LN 的橡胶表带”
```

如果只抽到 `126610LN` 就会把表带错误归入腕表实体。

因此：

```text
record_type != whole_watch/watch_head
=> 即使存在 reference，也不能进入“腕表商品同款自动归并”路径
```

这与之前 `parts-distributor-sku-classifier` 里“先区分 manufacturer identifier 与 distributor/SKU 角色”是同一原则，只是这里进一步把“被提及的 reference 是否属于当前售卖主体”也纳入判定。

---

## 9. Layer 3：Reference Observation Extractor

不要直接输出一个 `reference`，而是输出多条可审计 observation。

### 9.1 结构化字段

```json
{
  "value_raw": "126610LN",
  "source_type": "structured_reference",
  "field": "reference",
  "confidence": 1.0,
  "span": null
}
```

### 9.2 标题/详情 span extractor

建议 regex + token classifier 双路召回：

```text
Regex/brand pattern -> 高 recall 候选 span
Token classifier    -> 判断 span 的语义角色
```

输出：

```json
{
  "value_raw": "126610LN",
  "source_type": "title",
  "char_start": 21,
  "char_end": 29,
  "context": "劳力士潜航者 126610LN 黑盘...",
  "role_probs": {
    "current_product_reference": 0.997,
    "compatible_reference": 0.001,
    "platform_sku": 0.001,
    "other": 0.001
  }
}
```

### 9.3 图片 OCR

图片先分类，再决定 OCR 策略：

```text
caseback        -> 多方向/弧形文字 OCR
warranty_card   -> document OCR
hang_tag        -> dense text OCR
certificate     -> document OCR
box             -> lower priority OCR
dial            -> logo/text OCR
```

OCR 必须保存 top-k hypothesis：

```json
{
  "image_id": "img-8",
  "image_role": "warranty_card",
  "bbox": [0.23, 0.41, 0.71, 0.53],
  "raw_text": "12661OLN",
  "alternatives": [
    {"text": "126610LN", "p": 0.47},
    {"text": "12661OLN", "p": 0.41},
    {"text": "1266101N", "p": 0.07}
  ]
}
```

注意：top-1 OCR 不是事实，registry resolver 才负责把 hypothesis 映射到候选 reference。

---

## 10. Layer 4：Canonical Reference Registry

这是整个系统最重要的数据资产之一。

建议表结构：

```sql
CREATE TABLE canonical_reference (
    reference_id        BIGINT PRIMARY KEY,
    brand_id            BIGINT NOT NULL,
    canonical_reference VARCHAR(128) NOT NULL,
    collection_id       BIGINT NULL,
    record_type         VARCHAR(32) NOT NULL DEFAULT 'whole_watch',
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    provenance          JSONB NOT NULL,
    UNIQUE (brand_id, canonical_reference)
);

CREATE TABLE reference_alias (
    brand_id            BIGINT NOT NULL,
    alias                VARCHAR(128) NOT NULL,
    canonical_reference VARCHAR(128) NOT NULL,
    alias_type           VARCHAR(32) NOT NULL,
    transform_rule_id    VARCHAR(64) NULL,
    verified             BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (brand_id, alias)
);
```

registry 来源建议按可信度分级：

```text
L0：品牌官方/官方目录/官方页面
L1：高可信专业数据库 + 多源一致
L2：人工审核确认
L3：自动发现候选，未审核
```

只有 L0/L1/L2 可进入自动归并。

### 10.1 Resolve API

```text
resolve_reference(
  brand_id,
  raw_string,
  context,
  source_type
)
```

返回：

```json
{
  "status": "unique_exact|unique_safe_alias|ambiguous|unknown",
  "canonical_reference": "126610LN",
  "normalization_steps": ["upper", "remove_format_space"],
  "registry_level": "L0",
  "candidate_count": 1
}
```

只有：

```text
unique_exact
unique_safe_alias
```

才可能进入自动匹配路径。

---

## 11. Layer 5：MXT-inspired Reference Evidence Scorer

这里才是本论文最直接的落地点。

与原 MXT 的区别：

```text
原论文：text + image -> 自由生成 attribute value
本方案：text + image + candidate references -> 对候选做支持/冲突/拒识
```

### 11.1 输入

```json
{
  "brand": "Rolex",
  "record_type": "whole_watch",
  "text": "...",
  "observations": [...],
  "candidate_references": [
    "126610LN",
    "126610LV",
    "116610LN"
  ],
  "image_features": [...],
  "ocr_regions": [...]
}
```

### 11.2 输出

```json
{
  "candidates": [
    {
      "reference": "126610LN",
      "support_score": 0.9994,
      "conflict_score": 0.0002,
      "evidence": [
        "title_span:21-29",
        "warranty_card:img-8:roi-3"
      ]
    }
  ],
  "global_conflict": false,
  "abstain": false
}
```

### 11.3 推荐模型结构

不建议直接复现 `ResNet-152 + Xception + T5-base`，而是复现其思想：

```text
Text Encoder
  ├─ title/description/context
  └─ observation spans

Image Branch A
  └─ global image-role / compatibility encoder

Image Branch B
  └─ ROI/OCR region encoder

Candidate Encoder
  └─ canonical reference + brand + series metadata

Cross Attention / Late Fusion
  └─ candidate-wise evidence aggregation

Heads
  ├─ support(candidate)
  ├─ conflict(candidate)
  ├─ role(current product / compatible / SKU / other)
  └─ abstain
```

候选 reference 应是显式输入，这样模型不会开放式生成。

---

## 12. Layer 6：Precision-First Decision Engine

建议最终规则明确写成代码，而不是“交给模型自己学”。

### 12.1 自动接受规则

一个商品记录可以得到 `resolved_reference`，必须满足：

```text
1. brand_id 已唯一确定；
2. record_type 属于允许归并的商品主体；
3. 至少一个强 observation 唯一解析到 canonical reference；
4. 不存在另一个强 observation 指向不同 canonical reference；
5. reference registry 等级达到自动使用标准；
6. evidence scorer 无明确冲突；
7. 当前 brand/source bucket 通过 precision calibration；
```

伪代码：

```python
def resolve_product(item):
    if item.brand.status != "unique":
        return ABSTAIN("brand_not_unique")

    if item.record_type not in {"whole_watch", "watch_head"}:
        return REJECT("not_watch_subject")

    strong = collect_strong_reference_observations(item)
    resolved = [registry.resolve(item.brand_id, x) for x in strong]
    resolved = [x for x in resolved if x.status in {"unique_exact", "unique_safe_alias"}]

    refs = set(x.canonical_reference for x in resolved)

    if len(refs) == 0:
        return ABSTAIN("no_strong_reference")

    if len(refs) > 1:
        return ABSTAIN("strong_reference_conflict")

    ref = only(refs)

    evidence = scorer.verify(item, candidate=ref)
    if evidence.global_conflict:
        return ABSTAIN("multimodal_conflict")

    if not calibrated_bucket_allows_auto_accept(item, evidence):
        return ABSTAIN("risk_threshold")

    return ACCEPT(ref)
```

### 12.2 明确禁止的自动接受路径

以下情况必须禁止：

```text
只有 visual similarity
只有 fuzzy title similarity
只有 edit-distance reference
只有 LLM 自由生成 reference
只有低置信 OCR
reference 来自“适配/兼容/替换/表带/配件”上下文
多个强证据 reference 冲突
brand 不确定
```

---

## 13. Layer 7：跨源匹配其实应退化为 deterministic join

一旦每条商品记录获得：

```text
brand_id
canonical_reference
resolution_status=accepted
```

跨源匹配不需要复杂模型：

```text
entity_key = hash(brand_id, canonical_reference)
```

三源数据：

```sql
SELECT *
FROM normalized_products
WHERE brand_id = ?
  AND canonical_reference = ?
  AND resolution_status = 'accepted';
```

即可归入同一实体簇。

这正符合 Spec 对“同一个商品 = 同一 reference”的业务定义。

注意：如果业务未来把“同一个物理二手单品”定义为同一序列号/同一块表，则 reference 只能定义同款，不再足够。但在当前 Spec 中 reference 就是实体定义，因此应该让最终匹配尽可能确定化。

---

## 14. Layer 8：人工复核与反馈闭环

几百对黄金标签应该优先标“最危险”的边界，而不是随机抽样。

### 14.1 Hard Negative 设计

至少覆盖：

```text
同品牌同系列不同 reference
reference 只有一个字符不同
同一 reference 前缀/后缀不同
表带/配件标题提到腕表 reference
店铺 SKU 形似 reference
OCR 0/O、1/I/l、5/S、8/B 混淆
标题 reference 与保卡 reference 冲突
标题 reference 与结构化字段冲突
图片与文本品牌冲突
跨来源同 reference 但文本风格差异极大
```

### 14.2 人工界面必须展示证据，而不是只展示 score

复核页面应显示：

```text
左：原始商品文本/结构化字段
中：抽取出的 reference observations + span
右：图片 ROI / OCR 结果
下：canonical registry 命中路径 + normalization steps
```

人工只需要回答：

```text
1. 当前商品 reference 是什么？
2. 提取的字符串角色是什么？
3. 是否存在证据冲突？
```

复核结果回流：

- extractor 训练数据；
- role classifier；
- OCR confusion rules；
- registry alias；
- threshold calibration。

---

## 15. 训练方案：如何借用 MXT 的 distant supervision，但不把噪声放大

### 15.1 自动正样本

只用高精度规则构造：

```text
structured ref exact hit registry
AND title exact same canonical ref
AND brand exact
AND record_type valid
```

或者：

```text
structured ref exact hit registry
AND warranty-card OCR exact same canonical ref
```

### 15.2 自动负样本

优先构造“难负例”：

```text
same brand + same collection + different reference
```

例如：

```text
126610LN vs 126610LV
```

比随机跨品牌负样本更重要。

### 15.3 三种训练目标

建议多任务：

```text
L = λ1 * L_reference_role
  + λ2 * L_candidate_support
  + λ3 * L_conflict
```

其中：

- `reference_role`：当前商品 reference / compatible model / SKU / other；
- `candidate_support`：某 canonical reference 是否有证据支持；
- `conflict`：多模态/多字段是否存在冲突。

不要把“最终 pair 是否同商品”作为唯一训练目标，否则模型很难学到可解释错误原因。

---

## 16. 推理时的候选生成：把开放世界变成受限检索

候选集可按以下顺序生成：

```text
1. exact span -> registry exact
2. safe normalized span -> registry alias
3. OCR top-k -> registry lookup
4. brand + collection -> reference dictionary prefix/index
5. fuzzy lookup -> 只用于召回候选，不用于自动接受
```

候选集建议保持很小：

```text
K <= 20
```

然后再让 evidence scorer 做 candidate-wise 判断。

这样原 MXT 的“zero-shot 生成优势”在本场景被改写为：

> **即使模型训练没见过某 reference，只要 registry 中存在、并且文本/OCR 出现了对应字符串，系统仍可 zero-shot 地解析它。**

这比让语言模型学会“生成未见型号”更安全。

---

## 17. 图片策略：只把钱花在真正有身份信息的图片上

100 万–1000 万级商品如果每张图都跑重模型，成本会很高。

建议两阶段：

### Stage A：廉价筛选

```text
thumbnail encoder / classifier
=> image_role
```

只对以下图片进入 Stage B：

```text
caseback
warranty_card
hang_tag
certificate
high-text-density image
```

### Stage B：高质量 OCR + ROI verification

```text
text detector
-> crop rectification
-> orientation normalization
-> OCR top-k
-> registry lookup
```

表背可增加：

```text
circular text detection
polar unwrap
multi-angle OCR
```

保卡/证书可增加 layout parsing。

这比“所有图片都算视觉相似度”更符合业务价值。

---

## 18. Incremental Architecture：适配持续更新

建议事件化：

```text
source crawl
  -> ProductUpserted
  -> preprocess
  -> brand resolved
  -> text observations extracted
  -> image tasks queued
  -> reference resolved
  -> entity membership updated
```

重要字段均带版本：

```text
extractor_version
ocr_version
registry_version
scorer_version
policy_version
```

结果表：

```sql
CREATE TABLE product_reference_resolution (
    source_item_key      VARCHAR(256) NOT NULL,
    product_version      VARCHAR(64)  NOT NULL,
    brand_id             BIGINT,
    canonical_reference  VARCHAR(128),
    status               VARCHAR(32) NOT NULL,
    reason_code          VARCHAR(64) NOT NULL,
    confidence_bucket    VARCHAR(32),
    evidence_json        JSONB NOT NULL,
    extractor_version    VARCHAR(64) NOT NULL,
    registry_version     VARCHAR(64) NOT NULL,
    policy_version       VARCHAR(64) NOT NULL,
    created_at           TIMESTAMP NOT NULL,
    PRIMARY KEY (source_item_key, product_version)
);
```

任何 registry 更新都可以有选择地重算受影响商品，而不是全量重跑。

---

## 19. Precision Calibration：不能只看 F1

当前业务的核心指标建议：

```text
1. Auto-accept precision
2. False positive count
3. Statistical upper bound of FP rate
4. Coverage / auto-accept rate
5. Abstain rate
6. Conflict detection rate
7. Per brand/source/reference-pattern slices
```

黄金集规模只有几百对时，不要声称“测试 0 个误匹配所以 precision=100%”。

应该报告：

```text
observed FP = 0
sample size = N
one-sided upper confidence bound = ...
```

并把自动接受阈值设置在满足风险上界后再上线。

如果样本不足，宁愿让更多记录进入 abstain。

---

## 20. 可直接实现的 API 设计

### 20.1 Extract observations

```http
POST /v1/reference/observations
```

```json
{
  "source": "watchhome",
  "item_id": "xxx",
  "brand_hint": "Rolex",
  "title": "...",
  "description": "...",
  "structured": {...},
  "images": [...]
}
```

返回：

```json
{
  "brand": {"id": 10, "status": "unique"},
  "record_type": "whole_watch",
  "observations": [...]
}
```

### 20.2 Resolve canonical reference

```http
POST /v1/reference/resolve
```

返回：

```json
{
  "status": "accepted",
  "canonical_reference": "126610LN",
  "reason_code": "TITLE_EXACT_PLUS_CARD_OCR_EXACT",
  "evidence": [...],
  "conflicts": []
}
```

### 20.3 Match/cluster

```http
GET /v1/entities/by-reference?brand_id=10&reference=126610LN
```

返回三源同 reference 的商品记录。

---

## 21. MVP：不需要先训练复杂多模态大模型

最稳妥的第一版应按收益递增上线。

### Phase 1：纯规则 + Registry

实现：

```text
brand normalization
reference regex/span extraction
canonical registry
safe normalization
exact join
conflict detection
```

这一步就能吃掉大量高精度样本。

### Phase 2：OCR 身份证据

只处理：

```text
warranty card
hang tag
caseback
certificate
```

加入 OCR top-k + registry resolver。

### Phase 3：Reference Role Classifier

训练模型判断：

```text
current product reference
compatible reference
platform SKU
other
```

优先解决配件/适配语境误抽取。

### Phase 4：MXT-inspired Multimodal Evidence Scorer

把文本、OCR ROI、图片角色、候选 reference 一起输入，做支持/冲突/拒识。

### Phase 5：统计风险校准 + 持续学习

用黄金集和人工反馈更新 per-bucket policy。

这样比一开始训练一个“端到端图文 pair matcher”更容易达到“绝不能误匹配”。

---

## 22. 与原 MXT 的对应关系

| MXT 原设计 | 当前腕表系统改造 |
|---|---|
| `(product-type, attribute)` prompt | `(brand, source, record_type, task=reference)` schema |
| T5 生成任意 attribute value | span copy / candidate classification / constrained decoding |
| ResNet-152 全局视觉 embedding | image role + brand/record compatibility |
| MAG 图像修正文本表示 | 视觉对文本证据做门控支持/冲突 |
| Xception attribute-aware image features | reference-evidence ROI/OCR region encoder |
| multi-head cross attention | candidate-reference 与 evidence region cross attention |
| distant supervision from catalog | 高置信结构化 reference + 多证据一致弱标签 |
| heuristic value normalization | registry-backed safe normalization |
| Recall@90P | Recall@99.9P/99.99P + FP risk upper bound |
| generative zero-shot | registry-based zero-shot resolution |
| 单模型覆盖大量 PT-attribute | 单 evidence model 跨品牌/来源共享 |

---

## 23. 对论文的批判性结论

### 23.1 最值得借鉴

1. **多任务共享**：一个模型覆盖大量任务组合，适合品牌/来源持续扩展；
2. **双层视觉融合**：全局语义 + attribute-aware 局部区域，而不是一个图片向量解决所有问题；
3. **distant supervision**：适合百万级持续增量，人工标签只留给 hard cases；
4. **高 precision 区间评估**：生产目标不是只看平均 F1；
5. **部署经验**：值归一化、阈值、模型维护才是大规模系统真正难点。

### 23.2 不能直接照搬

1. **开放式生成**不适合作为 reference 最终值；
2. 90% precision 对本项目远远不够；
3. 通用图片属性可从外观推断，reference 不能靠“看起来像”推断；
4. catalog 的启发式相似值合并对型号可能产生灾难性 false positive；
5. 原论文没有把“identifier role”作为核心任务，而腕表标题中兼容型号、SKU、配件 reference 很常见。

---

## 24. 最终推荐方案

综合 MXT 与当前 Spec，推荐实际落地采用：

```text
Canonical Reference Registry
        +
Multi-source Reference Observation Extraction
        +
OCR on identity-bearing image regions
        +
MXT-inspired multimodal evidence verification
        +
Hard conflict veto
        +
Selective/abstain decision policy
        +
Deterministic (brand_id, canonical_reference) join
```

关键原则只有三条：

### 原则 1：模型负责“找到和解释证据”，registry 负责“定义合法 reference”

模型不能自由创造用于自动归并的 reference。

### 原则 2：图片负责“补证和否决”，不负责越过 reference 规则判同款

纯视觉相似度永不单独触发自动匹配。

### 原则 3：最终 entity matching 尽量退化成确定规则

只要每条记录的 canonical reference 恢复可靠，跨源匹配就是：

```text
brand_id 相同 AND canonical_reference 相同
```

而不是另一个黑盒模型。

从工程收益看，这篇论文最有价值的不是“换一个更强的 multimodal model”，而是提供了一套经过大规模电商验证的**多模态属性恢复框架**。把它的生成式输出改造成受限候选验证后，可以直接成为当前 Spec 的 reference extraction/evidence layer，并与此前 c 已分析的 DeepBlocker、TransClean、Selective Classification、Confidence Classifier 等方案组合：

```text
MXT-style evidence extraction
-> reference exact entity key
-> DeepBlocker 只服务于未解析/人工候选召回
-> selective classifier 控制自动接受覆盖率
-> TransClean/GraLMatch 做跨源簇一致性审计
```

这样既保留大模型/多模态对脏数据和缺失字段的处理能力，又把最终自动合并控制在可审计、可证明、低风险的硬约束之内，最符合“precision 优先到极致，允许漏匹配”的业务要求。
