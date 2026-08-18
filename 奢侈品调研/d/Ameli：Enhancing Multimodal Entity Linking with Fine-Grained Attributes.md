# Ameli: Enhancing Multimodal Entity Linking with Fine-Grained Attributes

> 分析者：d  
> 分析日期：2026-08-18  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选择 **Ameli: Enhancing Multimodal Entity Linking with Fine-Grained Attributes**（Yao et al., EACL 2024）进行深入分析。

- 论文页面：<https://aclanthology.org/2024.eacl-long.172/>
- arXiv：<https://arxiv.org/abs/2305.14725>
- 论文列出的代码仓库：<https://github.com/VT-NLP/Ameli>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

这篇工作的核心价值不是“多模态模型本身”，而是它把电商商品实体链接明确拆成了两级：

```text
Candidate Retrieval
    -> 从大实体库中召回少量候选

Entity Disambiguation
    -> 用文本、图片和结构化属性在候选中消歧
```

而且论文实验证明：在“外观和描述非常相似、只差少数细粒度属性”的商品里，**结构化属性是最重要的消歧信号之一**。这与当前腕表需求高度同构：同系列腕表的标题和图片可以极其相似，但业务定义已经明确规定：

> **同一个商品 = 同一个 reference number / 型号。**

因此，对当前需求最有价值的迁移不是照搬 AMELI 的 `SBERT + CLIP + DeBERTa + weighted score -> argmax`，而是把它改造成一个 **Reference-Centric Attribute-aware Entity Resolution** 系统：

```text
原始三源商品
  -> 品牌规范化
  -> reference 候选抽取（字段 / 标题 / 描述 / OCR）
  -> 编号角色识别
  -> 品牌级 canonicalization
  -> 候选 reference 检索
  -> 文本 / OCR / 图片 / 元数据多证据验证
  -> 强冲突否决
  -> VERIFIED / CONFLICT / ABSTAIN
  -> (canonical_brand, canonical_reference) 精确实体键
  -> 跨源聚合
```

最终的自动 MATCH 规则应该极其简单：

```text
same canonical_brand
AND same VERIFIED canonical_reference
AND no trusted conflicting_reference
=> MATCH

else
=> ABSTAIN / CONFLICT
```

**不建议**让一个多模态相似度模型直接输出 `MATCH`。AMELI 自己的结果也说明这一点：它的完整模型在论文实验中的 end-to-end F1 仍明显不适合作为“绝不能误匹配”的直接裁决器。它真正适合当前系统的位置是：

1. 候选 reference / 候选实体召回；
2. 从文本和图片补结构化属性证据；
3. 对已召回的 reference 候选做冲突检测和辅助验证；
4. 在无法得到 reference 硬证据时选择 **拒识**，而不是强制选一个。

因此，本轮给出的直接落地方案是：

> **以 `(brand_id, canonical_reference)` 为唯一跨源实体键，以 AMELI 式“多模态候选召回 + 属性感知消歧”作为 reference 解析辅助层，再加 precision-first 的硬门控与拒识机制。**

---

## 1. 为什么选择 AMELI

`奢侈品文章调研.md` 对这篇工作的推荐点非常贴合当前 Spec：它不是把商品匹配做成一次性的 pairwise 分类，而是把大规模商品实体链接拆成 Candidate Retrieval 和 Entity Disambiguation，并显式利用 Model Number 等细粒度属性过滤候选。

当前三源腕表数据具有以下典型问题：

1. 数据量达到 100 万–1000 万级，无法做全笛卡尔商品对比较；
2. reference 可能有独立字段，也可能只埋在标题、详情或图片里；
3. 各来源字段稀疏且结构不同；
4. 同品牌同系列的相邻 reference，文本和图片高度相似；
5. 图片可以补充证据，但不能替代 reference；
6. 允许漏匹配，但不能接受误匹配；
7. 需要持续增量，而不是一次性离线比赛。

AMELI 的任务恰好也是：

> 给定一个带文本和图片的商品 mention，从一个包含文本、图片、结构化属性的大型商品 KB 中找到准确实体。

论文特别强调：很多候选产品只有少量属性不同，必须做 fine-grained attribute reasoning。这一点与腕表的同系列不同 reference 非常相似。

最重要的是，AMELI 的实验揭示了一个对当前架构很有价值的事实：

> **属性信息对消歧的贡献，比单纯增加图片/文本相似度更直接。**

对于当前业务，这个“属性”应该进一步收缩成：

```text
最高优先级：reference number
第二优先级：brand / collection / model family / material / size / movement 等
辅助证据：image / OCR / description semantic similarity
```

换句话说，当前不是普通的“商品相似匹配”，而是一个**reference 解析问题**。

---

## 2. AMELI 原始问题与数据模型

## 2.1 Entity / Mention 的定义

AMELI 把目标商品库中的每个商品当作 entity，每个 entity 包含：

```text
Entity
  ├── product title
  ├── description
  ├── image(s)
  └── structured attributes
```

输入 mention 来自用户 review：

```text
Mention
  ├── review text
  ├── image(s)
  └── product mention span
```

系统需要将 mention 链接到唯一 entity。

EACL 最终版本页面给出的规模是：

- 16,735 个 mentions；
- 30,472 张 mention images；
- 34,690 个 KB entities；
- 177,873 张 entity images；
- 798,216 个 attributes。

arXiv 较早版本中的样本统计略有差异，但方法主体一致。这个版本差异不影响本次对架构的分析。

## 2.2 为什么它比普通商品相似度更接近腕表问题

AMELI 的一个关键设置是：

> 候选商品之间可能极其相似，只在一个或几个属性上有细微差异。

例如同一型号族产品可能只差：

- 存储；
- 内存；
- 颜色；
- 屏幕尺寸；
- 某个规格后缀。

腕表同理：

```text
同系列
  + 相似表盘
  + 相似表壳
  + 相似标题
  + 相近年份
```

仍可能因为一个 reference 后缀、材质、尺寸或机芯配置而属于不同实体。

所以这类任务的正确思路不是：

```text
similarity(item_a, item_b) > threshold
```

而是：

```text
retrieve_plausible_candidates(item)
verify_fine_grained_identity_evidence(item, candidate)
```

当前 Spec 又进一步把业务身份定义缩小成 reference，因此可以做得比 AMELI 更保守、更确定。

---

## 3. AMELI 原始技术架构

## 3.1 总体两阶段架构

论文使用标准 Entity Linking 两阶段结构：

```text
                         ┌─────────────────────┐
Mention text/image ----> │ Candidate Retrieval │ ---- Top-K entities
                         └─────────────────────┘
                                   |
                                   v
                         ┌──────────────────────┐
                         │ Entity Disambiguation│
                         │ Text + Image + Attr  │
                         └──────────────────────┘
                                   |
                                   v
                              target entity
```

这个拆法对 1000 万级商品尤其重要，因为真正昂贵的 cross-encoder / NLI / 多模态打分只能运行在很小的候选集合上。

---

## 3.2 预处理：Prior Probability

在正式召回前，论文先根据产品 title 和 category 的 noun chunks 统计 mention/entity、mention/category 的 prior probability。

直觉上类似：

```text
P(entity | mention noun chunk)
P(category | mention noun chunk)
```

如果某些 mention 词组历史上几乎不可能对应某个实体/类别，就可以降低其候选优先级甚至过滤掉。

### 对腕表需求的迁移

当前不能照搬 noun chunk prior，但可以做更强、更业务化的 prior：

```text
P(reference | canonical_brand, token_pattern)
P(reference_family | brand, collection)
P(role=reference | token_shape, left_context, right_context, source)
P(platform_sku | token, source)
```

特别是 `source` 很重要：

```text
腕表之家某字段 -> 可能高概率是官方型号
奢当家某字段 -> 可能是内部货号
标题中“货号/编号”后字符串 -> 可能是店铺 SKU
图片表背 OCR -> 可能是 serial，也可能是 reference
```

因此当前系统需要的不是简单先验，而是 **provenance-aware evidence prior**。

---

## 3.3 预处理：Attribute Value Extraction

AMELI 先从 mention 中抽属性值，再用这些属性值过滤候选。

论文使用两条路径：

### 路径 A：图片 OCR

用 OCR 从 review 图片识别文本，如果 OCR 文本能与 KB 中已知 attribute value 匹配，则把它当作 mention attribute。

这是非常适合腕表的部分，因为 reference 可能出现在：

- 表背；
- 保卡；
- 吊牌；
- 证书；
- 盒签；
- 商品实拍中的标签。

但对当前系统必须增加一层“编号角色分类”，因为图片里的数字可能是：

```text
reference
serial number
机芯编号
生产编号
表壳编号
保卡编号
店铺标签
价格
尺寸
年份
```

因此 OCR 输出绝不能直接 `exact match -> MATCH`。

### 路径 B：文本属性抽取

论文早期版本尝试让 GPT-2 按属性 key 从 review text 中生成 attribute value，并且只保留能与 KB attribute values 对齐的结果。

关键思想不是 GPT-2，而是：

> **自由生成的属性值不能直接相信；必须被约束到已知实体属性集合。**

这与当前 reference 场景非常重要。

建议当前系统把“reference 抽取”做成 constrained extraction：

```text
自由文本 / OCR
  -> candidate strings
  -> brand-specific reference registry lookup
  -> canonical form candidates
  -> verify context
```

而不是：

```text
LLM(text) -> reference -> 直接入库
```

---

## 3.4 Candidate Retrieval：文本 + Cross Encoder + 图片

AMELI 的候选召回同时使用三种信号。

### 3.4.1 SBERT 双塔召回

把 mention text 和 entity description 各自编码成向量：

```text
h_m = SBERT(mention_text)
h_e = SBERT(entity_description)
score_text_dense = cosine(h_m, h_e)
```

优点：

- KB entity embedding 可以预计算；
- 可以 ANN 检索；
- 适合大规模召回。

缺点：

- 对仅差少量 reference 字符的同系列产品区分力不够；
- dense similarity 容易把“外观/语义相似”误当成“同实体”。

### 3.4.2 BERT Cross Encoder

对候选文本对联合编码：

```text
score_cross = CrossEncoder(mention_text, entity_description)
```

Cross Encoder 比双塔更精细，但成本高，因此只能在召回后的局部候选上运行。

### 3.4.3 CLIP 图片召回

对 mention image 和 entity image 做 CLIP embedding，再按 cosine similarity 召回。

```text
score_image = cosine(CLIP(img_m), CLIP(img_e))
```

### 3.4.4 融合

论文对文本余弦分、Cross Encoder 分、视觉分做加权，取 Top-K，再结合 prior filtering。

可以抽象成：

```text
score_retrieval =
    w1 * dense_text
  + w2 * cross_text
  + w3 * image_similarity
  + prior_adjustment
```

这本质上是一个“召回问题”，而不是“最终业务判定问题”。

### 论文结果带来的重要警告

arXiv 版本中，最佳配置之一在 Candidate Retrieval 上 Top-10 Recall 约 61.69%，Top-100 也不是 100%。这说明：

> **只要把系统设计成“必须从 Top-K 中选一个”，召回错误就必然向下游传播。**

论文也明确指出 end-to-end 性能受 Candidate Retrieval error propagation 影响。

当前业务不允许这种错误传播，因此必须允许：

```text
candidate set 不可信 / gold reference 不在候选中
=> ABSTAIN
```

而不是强行找最像的。

---

## 3.5 Entity Disambiguation：属性感知 NLI

这是 AMELI 最值得迁移的部分。

### 原理

如果 mention text 描述的是某个实体，那么它应该：

- entail 正确实体的属性；
- contradict 错误实体的冲突属性。

例如：

```text
mention: “颜色偏浅粉”
候选 A attribute: color = pink
候选 B attribute: color = black
```

可以将每个候选属性转成 hypothesis，然后做 NLI。

论文使用 DeBERTa 编码 mention 与候选 entity attribute / description，多个属性表示再拼接，经 MLP 得到文本消歧分数。

抽象为：

```text
for entity in candidates:
    evidence = []
    for attribute in entity.attributes:
        h = DeBERTa(review_text, attribute_statement)
        evidence.append(h)

    score_text(entity) = MLP(concat(evidence))
```

然后用交叉熵训练候选实体分类。

### 对腕表的迁移

当前最重要的 hypothesis 不应该是通用属性，而应该是：

```text
H_ref:    “当前售卖腕表的 reference 是 X”
H_brand:  “当前商品品牌是 B”
H_series: “当前商品属于系列 S”
H_size:   “当前商品尺寸是 D”
H_mat:    “当前商品材质是 M”
```

其中 `H_ref` 权重应远高于其他属性。

更关键的是，我们不能让 NLI 直接负责“判断 reference 是否相同”，而应该只输出证据：

```text
ENTAIL
CONTRADICT
UNKNOWN
```

最后由规则决策层决定 VERIFIED / CONFLICT / ABSTAIN。

---

## 3.6 Image Disambiguation：CLIP + Adapter + Contrastive Loss

AMELI 把 CLIP 图像表示经过一个轻量 adapter：

```text
H' = H + FFN(H)
```

本质是残差形式的 task adapter，把通用 CLIP 表示映射到当前商品链接任务空间。

训练时用 cosine similarity + contrastive loss，并使用 in-batch negatives。

这个设计值得保留，因为当前三源腕表图像分布与通用 CLIP 训练数据不同：

- 商家白底图；
- 柜台图；
- 表盘近照；
- 表背；
- 保卡；
- 二奢环境图；
- 水印；
- 多图顺序差异。

可以通过 adapter 或轻量 LoRA 让视觉模型适应“腕表相关相似度”。

但当前业务必须明确：

> **图像只能作为候选召回、证据增强和冲突提示，不能覆盖 reference 硬冲突。**

例如两个不同 reference 的同系列钢款可能几乎完全一样；视觉模型再高分也不能把它们自动合并。

---

## 3.7 最终融合

AMELI 推理时把：

- 文本 NLI 分数；
- 图片 cosine similarity；

做加权，选择最高分 entity，并进一步用已抽取属性过滤不匹配候选。

抽象为：

```text
score(entity) = alpha * score_text(entity)
              + beta  * score_image(entity)

prediction = argmax(score)
```

这里就是当前需求**不能照搬**的地方。

当前要求 precision-first，因此不能使用：

```text
argmax(candidates)
```

而应使用：

```text
if hard_reference_verified:
    VERIFIED
elif hard_conflict:
    CONFLICT
elif high_confidence_but_not_provable:
    ABSTAIN
else:
    ABSTAIN
```

也就是把 closed-set classification 改成 selective prediction。

---

## 4. AMELI 实验对当前需求的启示

## 4.1 属性是最强的直接信号之一

论文的消歧实验中：

```text
AMELINK_w/o_Attribute: 35.70 F1（Disambiguation）
AMELINK full:          52.35 F1（Disambiguation）
```

完整模型 end-to-end F1 为 33.47%。

这说明结构化 attribute 对区分近似商品贡献非常大。

对当前系统，应当做更极端的业务化处理：

```text
reference = identity attribute
其他 attribute = supporting / contradiction evidence
```

## 4.2 多模态适合召回，不适合直接定义“同商品”

论文中 text + image 的召回优于单模态，说明图片确实能补文本缺失。

但是当前“同一个商品”的定义已经明确等价于 reference，因此视觉相似不能改变 identity rule。

正确关系应是：

```text
视觉高相似
    -> “值得检查是否同 reference”

而不是

视觉高相似
    -> “就是同商品”
```

## 4.3 必须防止 retrieval error propagation

论文 end-to-end 性能显著低于“gold 在候选集内”的 disambiguation 性能，说明两阶段系统最大的工程风险是：

```text
召回错
  -> 下游再强也救不回来
```

当前方案可以通过一个更简单的业务事实降低这个风险：

> 如果我们能够解析出 VERIFIED reference，就不需要再“猜实体”。

因此候选召回层只服务于：

- reference registry lookup；
- 模糊拼写恢复；
- OCR 纠错候选；
- 人工复核候选。

它不应该决定最终 MATCH。

---

# 5. 面向当前 Spec 的改造版架构

下面给出可以直接落地的生产架构。

## 5.1 总体结构

```text
┌─────────────────────────────────────────────────────────────┐
│ Data Sources                                                │
│ 雷小安 / 腕表之家 / 奢当家 / future sources                │
└──────────────────────────────┬──────────────────────────────┘
                               v
┌─────────────────────────────────────────────────────────────┐
│ 1. Raw Ingestion                                             │
│ item_raw + source + raw fields + images + crawl_version      │
└──────────────────────────────┬──────────────────────────────┘
                               v
┌─────────────────────────────────────────────────────────────┐
│ 2. Normalization                                             │
│ brand normalize / Unicode / field mapping / text cleanup     │
└──────────────────────────────┬──────────────────────────────┘
                               v
┌─────────────────────────────────────────────────────────────┐
│ 3. Reference Evidence Extraction                             │
│ explicit field / title / description / OCR / image metadata │
│ -> reference candidates + provenance                         │
└──────────────────────────────┬──────────────────────────────┘
                               v
┌─────────────────────────────────────────────────────────────┐
│ 4. Reference Role Classifier                                 │
│ PRODUCT_REF / SERIAL / SKU / ACCESSORY_TARGET / OTHER       │
└──────────────────────────────┬──────────────────────────────┘
                               v
┌─────────────────────────────────────────────────────────────┐
│ 5. Brand-specific Canonicalization + Registry Retrieval      │
│ exact / normalized / constrained fuzzy candidates            │
└──────────────────────────────┬──────────────────────────────┘
                               v
┌─────────────────────────────────────────────────────────────┐
│ 6. Attribute-aware Evidence Verifier                         │
│ text NLI / OCR consistency / image support / metadata        │
│ + hard contradiction detection                               │
└──────────────────────────────┬──────────────────────────────┘
                               v
┌─────────────────────────────────────────────────────────────┐
│ 7. Precision-first Decision                                  │
│ VERIFIED / CONFLICT / ABSTAIN                                │
└──────────────────────────────┬──────────────────────────────┘
                               v
┌─────────────────────────────────────────────────────────────┐
│ 8. Exact Entity Key                                          │
│ (canonical_brand_id, canonical_reference)                    │
│ -> cross-source entity group                                 │
└─────────────────────────────────────────────────────────────┘
```

这保留了 AMELI 的“Retrieval + Disambiguation”，但把最终输出从“选择一个 entity”变成“验证 reference”。

---

# 6. 数据模型设计

## 6.1 `item_raw`

保存不可变原始抓取记录：

```sql
CREATE TABLE item_raw (
  item_id            BIGINT PRIMARY KEY,
  source             VARCHAR(32) NOT NULL,
  source_item_id     VARCHAR(128) NOT NULL,
  crawl_time         TIMESTAMP NOT NULL,
  raw_title          TEXT,
  raw_description    TEXT,
  raw_brand          TEXT,
  raw_reference      TEXT,
  raw_json           JSONB,
  record_hash        CHAR(64),
  UNIQUE(source, source_item_id, record_hash)
);
```

不要覆盖旧数据，便于以后重放新版本解析器。

## 6.2 `reference_evidence`

这是整个系统最关键的表。

```sql
CREATE TABLE reference_evidence (
  evidence_id        BIGSERIAL PRIMARY KEY,
  item_id            BIGINT NOT NULL,
  raw_candidate      TEXT NOT NULL,
  normalized_candidate TEXT,
  canonical_brand_id BIGINT,
  evidence_source    VARCHAR(32) NOT NULL,
  source_field       TEXT,
  role               VARCHAR(32),
  role_score         DOUBLE PRECISION,
  extractor_version  TEXT NOT NULL,
  span_start         INT,
  span_end           INT,
  image_id           TEXT,
  ocr_bbox           JSONB,
  context_text       TEXT,
  created_at         TIMESTAMP DEFAULT NOW()
);
```

`evidence_source` 建议枚举：

```text
EXPLICIT_REFERENCE_FIELD
TITLE_REGEX
DESCRIPTION_REGEX
OCR_BACKCASE
OCR_CARD
OCR_TAG
LLM_EXTRACTED
CATALOG_LOOKUP
HUMAN_LABEL
```

## 6.3 `reference_registry`

全局 canonical reference 注册表：

```sql
CREATE TABLE reference_registry (
  reference_id       BIGSERIAL PRIMARY KEY,
  canonical_brand_id BIGINT NOT NULL,
  canonical_reference TEXT NOT NULL,
  reference_family   TEXT,
  aliases            JSONB,
  pattern_version    TEXT,
  status              VARCHAR(16) DEFAULT 'ACTIVE',
  UNIQUE(canonical_brand_id, canonical_reference)
);
```

registry 来源可以是：

1. 腕表之家结构化型号库；
2. 品牌官网/公开 catalog；
3. 三源中高置信 reference 聚合；
4. 人工审核新增。

## 6.4 `item_resolution`

```sql
CREATE TABLE item_resolution (
  item_id              BIGINT PRIMARY KEY,
  canonical_brand_id   BIGINT,
  reference_id         BIGINT,
  canonical_reference  TEXT,
  status               VARCHAR(16) NOT NULL,
  decision_reason      TEXT,
  evidence_ids         JSONB,
  conflicting_refs     JSONB,
  verifier_version     TEXT,
  resolved_at          TIMESTAMP
);
```

状态只建议三个：

```text
VERIFIED
CONFLICT
ABSTAIN
```

不要设置 `LIKELY_MATCH` 并自动入组，否则迟早会被下游误当成确定事实。

---

# 7. Reference Candidate Extraction

## 7.1 证据优先级

建议按来源可靠度分级，而不是所有字符串一视同仁。

### Tier 0：人工 / 官方 catalog

```text
人工确认 reference
品牌官方 catalog reference
可信结构化 reference registry
```

### Tier 1：来源独立结构化字段

例如平台明确命名为：

```text
型号
reference
ref.
model no.
```

但也必须检查这个字段在该平台是否真的表示 manufacturer reference，而不是 source SKU。

### Tier 2：标题 / 描述明确上下文

例如：

```text
“型号: XXXXX”
“Ref. XXXXX”
“reference XXXXX”
```

### Tier 3：OCR

例如表背、保卡、吊牌。

OCR 强度进一步取决于图像类型：

```text
保卡 + 明确 ref 标签 > 表背散乱字符 > 商品环境图 OCR
```

### Tier 4：LLM / Dense Retrieval 推断

只能用于生成 candidate，不能直接 VERIFIED。

---

## 7.2 编号角色分类是必需的

每个 candidate 都必须分类为：

```text
PRODUCT_REFERENCE
SERIAL_NUMBER
PLATFORM_SKU
SHOP_SKU
CALIBER
CASE_NUMBER
ACCESSORY_TARGET_REFERENCE
YEAR_OR_SIZE
UNKNOWN_CODE
```

这是 precision-first 的关键。

例如：

```text
“适用于 XXX 型号的原装表带”
```

即使 `XXX` 是真实 reference，它的 role 应该是：

```text
ACCESSORY_TARGET_REFERENCE
```

而不是当前商品的 `PRODUCT_REFERENCE`。

### 特征

可以组合：

```text
candidate 形状
左右 20~50 token 上下文
brand
source
field name
是否命中 known reference registry
是否命中 source SKU pattern
是否与标题中的商品类别“腕表/表带/盒证”一致
```

模型可以是轻量 DeBERTa / MiniLM classifier，甚至初期规则 + Logistic Regression 都可以。

由于 precision 优先，role classifier 也应允许 UNKNOWN。

---

# 8. Canonicalization：必须品牌级、保守、可逆

reference 规范化最危险的错误是“过度清洗”。

错误做法：

```text
remove all punctuation
remove all suffixes
只保留数字
```

因为不同 suffix 可能对应不同实体。

建议只做确定无损的统一：

```text
Unicode NFKC
英文大小写统一
全角/半角统一
明显 OCR 空格统一
品牌级已验证 separator alias
品牌级 alias table
```

例如：

```text
raw -> normalized -> canonical
```

必须保留转换轨迹：

```json
{
  "raw": "AB 12-34",
  "steps": [
    "unicode_nfkc",
    "uppercase",
    "brand_rule_v3"
  ],
  "canonical": "AB12-34"
}
```

任何会丢失信息的规则都不能作为全局规则。

---

# 9. 候选 Reference Retrieval：把 AMELI 的召回改造成受约束召回

## 9.1 第一层：品牌 Blocking

所有 reference 检索必须先在 canonical brand 内进行：

```text
brand_id = B
-> search reference_registry where brand_id = B
```

不同品牌即使编号字符串相同也不能直接互相匹配。

## 9.2 第二层：Exact / Alias

```text
exact canonical match
known alias match
```

这是最高优先级。

## 9.3 第三层：字符级候选

用于 OCR / 抓取噪声：

```text
char trigram
Damerau-Levenshtein
separator-aware distance
confusable char mapping
```

OCR 常见混淆可以生成候选：

```text
O <-> 0
I <-> 1
S <-> 5
B <-> 8
```

但这些只用于召回，不可自动 canonicalize。

## 9.4 第四层：语义 / 图片候选

只有 reference 完全没抽到或 OCR 过差时，才使用：

```text
SBERT title/description ANN
CLIP image ANN
```

得到可能的 reference family 或近邻商品，再回到 reference verifier。

这一层等价于 AMELI Candidate Retrieval，但权力被严格限制为：

> **proposal only**。

---

# 10. Attribute-aware Reference Verifier

这是 AMELI 的 NLI 消歧思想在当前系统里的核心改造。

对每个候选 reference `r`，构造结构化证据：

```text
candidate reference: r
brand: b
known family: f
known attributes: a1...an
```

然后计算以下证据。

## 10.1 Reference direct evidence

```text
E_explicit_field
E_title_context
E_description_context
E_ocr_card
E_ocr_back
E_registry_alias
```

## 10.2 NLI / Cross Encoder evidence

将候选 reference 转成 hypothesis：

```text
“当前售卖的腕表 reference 是 r。”
```

分别对：

```text
title
description
OCR context
```

做：

```text
ENTAIL / CONTRADICT / UNKNOWN
```

注意：模型输出不能直接决策，只是 evidence。

## 10.3 Supporting attributes

如果 registry 有 candidate reference 的其他属性：

```text
collection
size
material
dial color
movement
```

可以构造：

```text
item evidence vs reference profile
```

这里可以借鉴 AMELI 的 attribute-aware NLI。

## 10.4 Image evidence

视觉证据建议拆成两个用途。

### A. 商品图相似度

```text
CLIP / DINOv2 embedding
```

仅用于 support。

### B. 图中 reference/OCR 证据

如果保卡或标签里直接出现 candidate reference，其价值远大于整体 image similarity。

因此视觉层应先做 image type classifier：

```text
WATCH_FRONT
WATCH_BACK
CARD
TAG
BOX
OTHER
```

再决定是否 OCR、是否赋高权重。

---

# 11. Hard Conflict Gate：比总分更重要

当前业务不能采用“正证据多就覆盖冲突”的设计。

必须设置 hard conflicts。

例如：

```text
可信结构化字段 = REF_A
保卡 OCR 高置信 = REF_B
REF_A != REF_B
```

即使图片与 `REF_A` 的 CLIP 分数很高，也必须：

```text
status = CONFLICT
```

建议硬冲突包括：

```text
两个 Tier0/Tier1 reference 不一致
Tier1 与高置信 CARD OCR 不一致
不同可信字段解析成不同 canonical reference
候选 reference 与明确品牌不一致
商品类型是配件但 reference 被识别为主腕表 reference
```

这是从“ranking architecture”切换到“safety architecture”的关键。

---

# 12. 决策状态机

建议完全避免连续相似度直接决定 MATCH，使用状态机。

```text
function resolve(item):
    brand = resolve_brand(item)
    if brand.status != VERIFIED:
        return ABSTAIN

    evidences = extract_reference_evidence(item, brand)
    candidates = retrieve_reference_candidates(evidences, brand)

    if candidates is empty:
        return ABSTAIN

    verdicts = verify(candidates, evidences)

    if exists_hard_conflict(verdicts):
        return CONFLICT

    verified = [v for v in verdicts if v.pass_hard_gate]

    if len(verified) != 1:
        return ABSTAIN

    return VERIFIED(verified[0].reference)
```

### `pass_hard_gate` 示例

初始版本可以非常保守：

```text
Case A:
  trusted explicit reference field
  + role=PRODUCT_REFERENCE
  + exact/alias registry hit
  + no conflict

Case B:
  title explicit ref context
  + independent OCR card evidence
  + both resolve to same canonical ref
  + no conflict

Case C:
  two independent textual/OCR evidence sources
  + exact same canonical ref
  + known registry ref
  + no conflict
```

其他全部 ABSTAIN。

这会牺牲 recall，但完全符合 Spec。

---

# 13. 跨源实体聚合：不再做 pairwise 模型

一旦 item 得到 VERIFIED reference：

```text
entity_key = (canonical_brand_id, canonical_reference)
```

就可以直接 group：

```sql
SELECT canonical_brand_id,
       canonical_reference,
       array_agg(item_id)
FROM item_resolution
WHERE status = 'VERIFIED'
GROUP BY canonical_brand_id, canonical_reference;
```

这样可以避免：

```text
雷小安 × 腕表之家
腕表之家 × 奢当家
雷小安 × 奢当家
```

分别训练三个 pairwise matcher。

实体身份由 reference key 决定，来源只是 evidence provenance。

这也是当前需求相比普通 Entity Matching 更简单、也更可审计的地方。

---

# 14. 100 万–1000 万级数据的工程实现

## 14.1 不要做 N² pairing

如果 1000 万记录做两两比较，规模不可接受。

当前架构中绝大部分记录只需：

```text
O(1) / O(log N) reference registry lookup
```

只有 ABSTAIN 项才进入 ANN / Cross Encoder / NLI。

## 14.2 推荐服务拆分

```text
reference-normalizer
reference-extractor
reference-registry
reference-retriever
reference-verifier
resolution-writer
review-console
```

初期不需要全部微服务化，可以一个 Python service + job runner，只保留模块边界。

## 14.3 存储建议

第一阶段足够：

```text
PostgreSQL
  -> raw metadata / evidence / resolution / registry

Object Storage
  -> images

OpenSearch / Elasticsearch
  -> title/reference lexical candidate retrieval

FAISS / HNSWlib
  -> 只给 unresolved items 做 dense retrieval
```

不建议一开始上复杂图数据库，因为身份键已经非常明确。

## 14.4 增量处理

每个 item 根据内容 hash 做幂等：

```text
(source, source_item_id, record_hash, pipeline_version)
```

如果原始记录未变化且 pipeline version 未变化：

```text
skip
```

如果：

- reference registry 更新；
- extractor version 更新；
- brand rule 更新；

只重跑受影响分区。

## 14.5 版本化是必须的

所有自动结论记录：

```text
extractor_version
canonicalizer_version
registry_version
verifier_version
```

否则一旦规则改错，无法精准回滚错误实体组。

---

# 15. 图片处理如何借鉴 AMELI，但不越权

AMELI 证明 image + text 在 retrieval 上有互补价值。

当前建议采用三级图片策略。

## Level 1：OCR 优先

对以下图片优先 OCR：

```text
保卡
吊牌
盒签
表背
证书
```

OCR token 经过 reference candidate parser。

## Level 2：视觉 family retrieval

对表盘/正面图做 embedding，只用于：

```text
候选系列 / reference family 召回
```

## Level 3：冲突检查

例如某候选 reference profile 明确属于某种可见配置，而图片明显是另一类，可以产生：

```text
VISUAL_CONFLICT
```

但视觉冲突最好先进入人工复核，不直接覆盖强 reference 字段，除非后续验证证明视觉冲突的 precision 极高。

---

# 16. 人工黄金标签如何使用

Spec 允许人工标注几百对。

不要只随机标 `same / different` pair。

应该优先标最有价值的 hard cases。

## 16.1 标签结构

推荐标：

```json
{
  "item_id": 123,
  "brand": "...",
  "reference": "...",
  "candidate_span": "...",
  "candidate_role": "PRODUCT_REFERENCE",
  "evidence_source": "TITLE",
  "decision": "VERIFIED"
}
```

同时保留 pair 标签：

```text
same_reference
not_same_reference
unknown
```

但 item -> canonical reference 的标签更可复用。

## 16.2 Hard negative 采样

重点抽：

```text
同品牌
同系列
reference 只差 1~2 字符
图片极像
标题极像
配件包含主表 reference
OCR 容易混淆字符
平台 SKU 很像 reference
```

这比随机 negative 更能测 precision。

## 16.3 标注量与“绝对 precision”

几百条黄金标签足够：

- 调阈值；
- 找规则漏洞；
- 做第一版 role classifier；
- 验证 hard gate。

但不够从统计上证明极低误匹配率。

经验上，如果自动接受样本中观察到 0 个错误，一侧 95% 置信的错误率上界粗略可用 `3/n` 估计：

```text
n = 300  -> 上界约 1%
n = 3000 -> 上界约 0.1%
```

所以“几百对”应该用于模型和规则开发，生产上线后还需要持续抽检自动接受样本，逐渐累积数千级 verified acceptance 审计集。

---

# 17. Precision-first 校准

AMELI 主要优化 F1，而当前业务目标不同。

当前建议看四个指标：

```text
Auto-Accept Precision
Auto-Accept Coverage
Conflict Rate
Abstain Rate
```

首要指标：

```text
Precision(VERIFIED reference)
```

而不是整体 F1。

可以分层设置 acceptance policy：

```text
Policy P0:
  仅 Tier0/Tier1 exact ref

Policy P1:
  P0 + 两路独立 reference 证据

Policy P2:
  P1 + OCR/card strong evidence

Policy Experimental:
  NLI / multimodal inferred reference
```

生产默认只放 P0/P1，P2 在验证后逐步开启。

---

# 18. 为什么不能直接照搬 AMELI

## 18.1 AMELI 是 closed-set argmax

它假设从候选集中选一个最可能 entity。

当前需求必须允许：

```text
NONE OF THE ABOVE
UNKNOWN
CONFLICT
```

## 18.2 AMELI 的优化目标不是超高 precision

论文报告的是 Recall@K 和 F1，目标不是“0 false positive”。

当前应重新设计 loss / threshold / evaluation。

## 18.3 图片相似度在腕表中风险更大

同系列相邻 reference 外观可能非常相似。

所以 image score 永远不能成为 identity 充分条件。

## 18.4 reference 是更强的业务定义

AMELI 要推断完整商品 entity；当前 Spec 已明确 identity key 是 reference。

因此没必要让模型学习一个模糊的“same product”概念。

直接解析 reference 更稳、更可解释。

---

# 19. 推荐的三阶段落地计划

## Phase 1：无模型/轻模型 MVP

目标：最快建立高 precision baseline。

### 做：

1. canonical brand dictionary；
2. 三源字段语义梳理；
3. reference regex candidate extraction；
4. known-reference registry；
5. source-specific SKU / serial 排除规则；
6. brand-specific conservative canonicalization；
7. exact `(brand, reference)` grouping；
8. VERIFIED / CONFLICT / ABSTAIN；
9. 人工复核页。

### 不做：

```text
通用 pairwise matching model
图片直接判同商品
LLM 直接生成 reference 后自动入组
```

这一步大概率已经能覆盖一批结构化较好的商品，同时 precision 最容易控制。

---

## Phase 2：AMELI 式候选召回

用于 Phase 1 的 ABSTAIN。

增加：

```text
OpenSearch char/token retrieval
reference char ANN / edit distance
SBERT title retrieval
CLIP image retrieval
```

只生成 candidate references。

例如：

```json
{
  "item_id": 123,
  "candidates": [
    {"ref":"R1", "reason":"title_char_match"},
    {"ref":"R2", "reason":"ocr_confusable"},
    {"ref":"R3", "reason":"image_neighbor"}
  ]
}
```

仍不自动 MATCH。

---

## Phase 3：Attribute-aware Verifier

增加：

```text
Reference Role Classifier
DeBERTa NLI verifier
OCR image type aware evidence
CLIP/DINO task adapter
hard-negative training
confidence calibration
```

模型只决定：

```text
support / contradict / unknown
```

最终状态仍由 hard policy engine 决定。

---

# 20. 一个可以直接编码的 Verifier 设计

## 20.1 Feature schema

```python
ReferenceCandidateFeatures = {
    "brand_exact": bool,
    "registry_exact": bool,
    "registry_alias": bool,
    "explicit_field_match": bool,
    "title_explicit_match": bool,
    "description_match": bool,
    "ocr_card_match": bool,
    "ocr_back_match": bool,
    "role_product_ref_prob": float,
    "nli_entail": float,
    "nli_contradict": float,
    "image_similarity": float,
    "title_similarity": float,
    "family_consistency": bool,
    "attribute_conflict_count": int,
    "trusted_ref_conflict": bool,
}
```

## 20.2 第一版规则

```python
def accept_reference(f):
    if f["trusted_ref_conflict"]:
        return False

    if not f["brand_exact"]:
        return False

    if f["role_product_ref_prob"] < 0.995:
        return False

    strong = 0
    strong += int(f["explicit_field_match"] and f["registry_exact"])
    strong += int(f["title_explicit_match"] and f["registry_exact"])
    strong += int(f["ocr_card_match"] and f["registry_exact"])

    if strong >= 2:
        return True

    if (
        f["explicit_field_match"]
        and f["registry_exact"]
        and f["nli_contradict"] < 0.01
        and f["attribute_conflict_count"] == 0
    ):
        return True

    return False
```

注意：

```text
image_similarity
```

在第一版 accept 规则中甚至可以完全不作为正向充分条件，只用于候选和 review UI。

这非常符合“宁可漏，不可错”。

---

# 21. 候选召回与 verifier 的接口

建议把 AMELI 的两阶段边界保留下来：

```json
POST /reference-candidates
{
  "item_id": 123
}

=>
{
  "candidates": [
    {
      "reference_id": 99,
      "canonical_reference": "...",
      "retrieval_scores": {
        "lexical": 0.98,
        "dense_text": 0.77,
        "image": 0.91
      }
    }
  ]
}
```

然后：

```json
POST /reference-verify
{
  "item_id": 123,
  "reference_id": 99
}

=>
{
  "status": "VERIFIED",
  "evidence": [...],
  "conflicts": [],
  "policy_version": "ref-policy-v1"
}
```

这样 retrieval 模型升级不会改变 identity policy。

---

# 22. 人工复核 UI 应该展示什么

对 ABSTAIN / CONFLICT，只展示“最有助于判断 reference”的证据，不要展示一个难解释的总分。

建议 UI：

```text
左侧：原始商品
  - 标题
  - 来源字段
  - 图片
  - OCR 框

中间：抽取证据
  - candidate strings
  - candidate role
  - provenance
  - canonicalization trace

右侧：reference candidates
  - canonical reference
  - known aliases
  - reference profile
  - 冲突属性
  - image neighbors
```

人工操作：

```text
确认 reference
否决 candidate
标记 SKU
标记 serial
标记 accessory target
新增 alias
```

这些操作可以直接回流训练 Role Classifier 和 Verifier。

---

# 23. 与持续增量数据的结合

每次新抓取记录到来：

```text
1. source field mapping
2. brand resolution
3. reference evidence extraction
4. registry lookup
5. verifier
6. resolution write
7. exact key group update
```

如果 registry 新增一个 reference：

```text
只重跑此前同品牌、且存在相似 candidate 的 ABSTAIN items
```

如果 canonicalization 规则升级：

```text
只重跑受该 brand_rule_version 影响的记录
```

如果视觉模型升级：

```text
只需要更新 unresolved candidate retrieval
```

这比重新跑全量 pairwise matcher 更适合长期维护。

---

# 24. 风险清单

## 风险 1：平台字段名字叫“型号”但其实不是 reference

解决：

```text
source_field_semantics registry
+ 抽样验证
+ per-source trust tier
```

## 风险 2：同一 reference 多种格式

解决：

```text
brand-specific alias
+ lossless normalization
```

## 风险 3：不同 reference 被过度 canonicalize 成一个

解决：

```text
禁止全局 aggressive cleaning
+ canonicalization trace
+ collision audit
```

## 风险 4：配件标题包含主表 reference

解决：

```text
Reference Role Classifier
+ product_type gate
```

## 风险 5：OCR 把 serial 当 reference

解决：

```text
image type
+ context
+ registry constrained lookup
+ independent evidence requirement
```

## 风险 6：视觉模型把同系列不同 reference 判得很像

解决：

```text
visual = retrieval/support only
reference conflict = hard veto
```

## 风险 7：新 reference 不在 registry

解决：

```text
ABSTAIN
+ review queue
+ human-approved registry expansion
```

绝不能为了 coverage 强行映射到最相似旧 reference。

---

# 25. 评测集设计

建议构造以下 slice：

```text
S1: 独立 reference 字段完整
S2: reference 仅在标题
S3: reference 仅在描述
S4: reference 仅在 OCR
S5: 同系列相邻 reference
S6: reference 只差一个字符
S7: 配件包含主表 reference
S8: 平台 SKU 类似 reference
S9: serial 类似 reference
S10: 新 reference / registry OOD
S11: 图片高度相似但 reference 不同
S12: 三源字段相互冲突
```

核心指标：

```text
Verified Precision
Conflict Detection Recall
Abstain Coverage
Reference Extraction Precision
Role Classification Precision(PRODUCT_REFERENCE)
```

其中最重要的是：

```text
Verified Precision
```

---

# 26. 推荐的生产决策原则

最终系统建议遵守以下不变量。

## Invariant 1

```text
没有 VERIFIED reference，不自动跨源合并。
```

## Invariant 2

```text
视觉相似不能覆盖 reference 冲突。
```

## Invariant 3

```text
LLM / NLI / Dense model 只能提供 evidence，不拥有最终 identity 权限。
```

## Invariant 4

```text
所有 reference canonicalization 必须可追溯。
```

## Invariant 5

```text
任何两个强 reference 证据冲突 => CONFLICT，不投票。
```

## Invariant 6

```text
未知 reference => ABSTAIN，而不是 nearest-known-reference。
```

---

# 27. AMELI 方案与当前落地方案对照

| AMELI | 当前腕表系统 |
|---|---|
| mention | source item |
| entity KB | reference registry / product reference profile |
| product attribute | reference + support attributes |
| SBERT retrieval | brand-blocked lexical/dense retrieval |
| CLIP retrieval | image candidate/support retrieval |
| prior probability | source/brand/reference role priors |
| OCR attribute extraction | OCR reference evidence extraction |
| DeBERTa NLI | reference hypothesis verifier |
| top-K candidate | candidate reference list |
| argmax entity | VERIFIED / CONFLICT / ABSTAIN |
| end-to-end F1 | verified precision + coverage |
| forced linking | precision-first selective linking |

---

# 28. 最小可落地方案

如果只给一套可以马上开始开发的方案，我建议：

```text
A. PostgreSQL 建 4 张核心表
   item_raw
   reference_evidence
   reference_registry
   item_resolution

B. 建 brand normalizer

C. 针对三个来源写 reference candidate extractor
   explicit field
   title
   description

D. 给 candidate 做 role classification
   PRODUCT_REFERENCE / SKU / SERIAL / ACCESSORY_TARGET / UNKNOWN

E. 建 brand-specific canonicalization

F. 只对 registry exact/alias hit 做自动 VERIFIED

G. 出现强冲突直接 CONFLICT

H. 其他全部 ABSTAIN

I. 对 ABSTAIN 再加 AMELI 式 Candidate Retrieval
   lexical -> SBERT -> CLIP

J. 对候选用 attribute-aware verifier
   NLI + OCR + support attributes

K. 最终仍只由 hard policy 决定是否 VERIFIED
```

第一版甚至可以先不训练任何大型模型。

只要把：

```text
reference evidence
role
canonicalization
registry
hard conflict
abstain
```

做扎实，就已经比一个端到端模糊 matcher 更接近业务目标。

---

# 29. 最终建议

AMELI 最值得当前项目吸收的三个设计是：

### 1. Retrieval 与 Disambiguation 分离

百万到千万级数据必须先缩小候选，再做昂贵判断。

### 2. 属性优先于整体相似度

细粒度商品必须比较 identity-relevant attributes。当前业务中，reference 本身就是最强 identity attribute。

### 3. 图片用于补证据，而不是替代结构化身份

OCR 和视觉相似可以极大帮助 reference 缺失场景，但不能越过 reference 硬规则。

而 AMELI 最需要被改造的一点是：

> **从“候选中选最像的一个”改为“只验证能证明的 reference，其余拒识”。**

因此建议当前系统的核心命名甚至不要叫 `Product Matcher`，而叫：

```text
Reference Resolution Service
```

对每条商品只回答：

```json
{
  "brand": "...",
  "reference": "...",
  "status": "VERIFIED | CONFLICT | ABSTAIN",
  "evidence": [...],
  "version": "..."
}
```

跨源匹配则降维成最简单、最可靠的一步：

```text
(canonical_brand, VERIFIED canonical_reference) exact equality
```

这套结构最符合当前 Spec 的业务定义，也比直接复刻 AMELI 的 argmax 多模态 entity linker 更容易做到：

- 高 precision；
- 可解释；
- 可人工复核；
- 可增量；
- 可回滚；
- 可逐步提升 coverage。

---

## 参考资料

1. Barry Yao, Sijia Wang, Yu Chen, Qifan Wang, Minqian Liu, Zhiyang Xu, Licheng Yu, Lifu Huang. **Ameli: Enhancing Multimodal Entity Linking with Fine-Grained Attributes.** EACL 2024.  
   <https://aclanthology.org/2024.eacl-long.172/>

2. arXiv preprint:  
   <https://arxiv.org/abs/2305.14725>

3. 论文列出的项目代码：  
   <https://github.com/VT-NLP/Ameli>

4. 当前项目需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
   <https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
