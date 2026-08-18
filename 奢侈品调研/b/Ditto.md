# Ditto：把预训练语言模型实体匹配改造成 precision-first 腕表 Reference 的“语义复核与否决层”

- 分析人：b
- 调研文章：Deep Entity Matching with Pre-Trained Language Models（Ditto）
- 论文地址：https://arxiv.org/abs/2004.00584
- 项目地址：https://github.com/megagonlabs/ditto
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31

## 1. 为什么这次选择 Ditto

当前 Spec 的目标是把雷小安、腕表之家、奢当家三源 100 万～1000 万级商品记录做跨源实体匹配，并持续增量更新。最关键的业务定义和风险约束是：

- “同一个商品”严格定义为 **同一 reference number / 型号**；
- reference 可能是结构化字段，也可能埋在标题、描述、图片中；
- 字段高度稀疏、schema 不统一；
- **绝对不能误匹配，precision 优先到极致，允许漏匹配**；
- 可人工标注几百对黄金数据。

此前 `b` 已经分析了几层互补组件：

1. `parts-distributor-sku-classifier.md`：先判断“长得像型号的字符串”到底是不是品牌 reference；
2. `DeepBlocker.md`：把千万级笛卡尔积压缩成高召回候选集，但 Blocking 绝不授权自动匹配；
3. `TransClean.md`：在 pairwise 结果形成图后，用传递一致性审计 false positive，防止一条错边污染整个实体簇；
4. `Ameli.md`：利用细粒度属性和多模态信息做候选消歧。

Ditto 适合补上中间的一层：

> **对“看起来很像、但 reference 证据存在歧义”的商品记录做语义级复核，尤其学习同系列、相邻型号、字段错位、文本噪声这类 hard negative。**

但 Ditto 原始论文优化的是普通 entity matching 的 F1，并不是 zero-false-positive 系统。因此，本次方案不会让 Ditto 直接产生“自动合并”结论，而是把它改造成：

- reference 候选归属验证器；
- exact-reference 匹配后的冲突否决器；
- 疑难样本人工复核排序器；
- hard-negative 学习器。

最终自动合并仍由 **可解释、可审计的 reference 硬证据** 收口。

---

## 2. Ditto 原始架构：为什么它适合字段稀疏、schema 不统一的数据

Ditto 的核心思路非常简单：把实体匹配从“复杂的字段对齐网络”改写成 Transformer 的 sequence-pair classification。

原始 EM 流程是：

```text
Table A                 Table B
   │                        │
   └──────── Blocker ───────┘
              │
        Candidate Pairs
              │
          Serialize
              │
        Inject Domain Knowledge
              │
           Summarize
              │
           Matcher
              │
         Match / NoMatch
```

它的重要边界是：**Blocking 和 Matching 分离。**

- Blocker 追求高召回，负责减少候选对数量；
- Ditto 只负责对已经形成的候选 pair 做精细判别。

这与当前项目很吻合。千万级数据不应该把 Transformer 用在全表 pairwise 上，而应该只处理经过 reference / brand / DeepBlocker 等候选层压缩后的极少量疑难 pair。

### 2.1 结构化记录如何变成 Transformer 输入

Ditto 把一条记录序列化成：

```text
COL title VAL ...
COL manufacturer VAL ...
COL price VAL ...
```

一对记录再通过 `[SEP]` 拼成 sequence pair：

```text
[CLS]
COL title VAL left title
COL model VAL left model
[SEP]
COL title VAL right title
COL model VAL right model
[SEP]
```

论文强调这种序列化方式不要求两侧 schema 完全一致，也不要求先做严格 attribute alignment。

这点对三源数据很实用。例如：

```text
腕表之家：
  型号 = 126334
  系列 = 日志型
  品牌 = 劳力士

奢当家：
  title = 劳力士日志型41蓝盘126334全套
  model = NULL

雷小安：
  goods_title = Rolex Datejust 41 126334 blue dial
  sku = LX123456
```

三方字段名不同，甚至 reference 只在标题里出现，但可以先映射成“语义槽位”再统一序列化，不需要物理 schema 完全相同。

### 2.2 模型本体其实非常薄

当前 GitHub `ditto_light/ditto.py` 中，`DittoModel` 的主体只有两层概念：

```text
AutoModel(预训练 Transformer)
        │
        └─ 取第一个 token / CLS 表示
                │
            Linear(hidden_size, 2)
                │
          Match / NoMatch logits
```

也就是说，Ditto 没有设计复杂的“品牌比较子网络”“型号比较子网络”“价格比较子网络”；它主要依赖预训练 Transformer 在拼接后的上下文里自行学习哪些 token 应该对齐、哪些差异属于冲突。

这也是它适合作为当前项目第二阶段 verifier 的原因：

- 输入 schema 可以快速扩展；
- 新来源只需要做字段映射与序列化；
- 少量人工 hard case 可以直接 fine-tune；
- 可以把 `REFERENCE`、`BRAND`、`SERIES`、`PLATFORM_SKU` 等角色显式做成特殊 token，减少模型自己猜字段语义的负担。

---

## 3. Ditto 三个优化点，哪些应该迁移，哪些不能照搬

## 3.1 Domain Knowledge：这是最值得迁移的一部分

论文将 domain knowledge 分为两类：

1. Span Typing：告诉模型某一段是 Product ID、Brand、Year 等；
2. Span Normalization：把语义相同、形式不同的 span 统一成 canonical form。

原始项目的 `ProductDKInjector` 会：

- 用 spaCy NER 标记 PRODUCT、NUM 等 span；
- 对数字做格式归一化；
- 对“长度至少 7 且含数字”的 token 加 `ID` 标记。

这个实现对通用电商有启发，但**不能原样用于腕表 reference**。

原因很直接：

- `126334` 只有 6 位，原规则不会因为长度条件而标成 ID；
- `5711/1A-010`、`116500LN`、`IW371604` 等 reference 含字母、数字、斜杠、连字符，形态差异大；
- 平台 SKU 同样可能是长字母数字串；
- 配件描述中出现“适配 126334 / 126300”时，reference 并不属于当前售卖主体；
- 序列号、机芯号、库存号也可能满足“像 ID”的字符模式。

因此当前项目应把 Ditto 的通用 DKInjector 替换成 **WatchReferenceDKInjector**，至少输出以下角色：

```text
[BRAND]              劳力士
[SERIES]             日志型
[REF_CANDIDATE]      126334
[CANONICAL_REF]      126334
[PLATFORM_SKU]       LX123456
[STORE_SKU]          A238912
[SERIAL]             8Fxxxxxx
[ACCESSORY_REF]      126300
[SIZE]               41mm
[YEAR]               2022
```

核心不是“模型更聪明”，而是先通过规则、字典和轻量分类器告诉模型 **编号角色**，让自注意力聚焦正确对象。

### 3.1.1 reference normalization 必须可逆、保留 provenance

Ditto 原论文建议把等价 span 重写成一致形式。当前项目也应该 canonicalize，但必须保留：

```text
raw_reference
normalized_reference
canonical_reference
normalization_rule_version
source_field
source_span
extractor
extractor_version
```

例如：

```text
raw:        "116 500 LN"
normalized: "116500LN"
canonical:  "116500LN"
```

又例如：

```text
raw:        "5711-1A-010"
normalized: "57111A010"
canonical:  "5711/1A-010"
```

是否允许第二种“强规范化”必须由品牌规则决定，不能仅靠删除符号。否则两个本来不同的 reference 可能在 normalization 中发生碰撞。

建议规则：

> **canonicalization 只能做已知等价变换，不能做“为了提高召回而猜测”的模糊归一化。**

任何不能证明等价的变换只生成 candidate，不进入自动匹配 key。

---

## 3.2 Summarization：迁移“信息预算”思想，不照搬 TF-IDF

Ditto 论文处理长文本时使用 TF-IDF 保留重要 token，因为 Transformer 有最大长度限制。论文还发现，在 WDC 商品数据中，把 title、description、brand、specTable 全部塞进去，反而可能因为序列过长导致关键信息被截断；在其实验设置下，title-only 经常更强。

这对当前项目的启示不是“只用 title”，而是：

> **reference-first 系统必须给硬证据保留固定 token 预算，不能让营销文案挤掉 reference。**

建议输入顺序固定为：

```text
1. canonical reference / raw reference
2. brand
3. category / product_role
4. series / model family
5. title 中 reference 前后窗口
6. 结构化关键属性
7. OCR reference 证据
8. 其余标题
9. description 摘要
```

绝不能使用普通 `text[:max_len]` 截断。

可以实现一个 `ReferenceAwareSummarizer`：

```python
reserved = [
    reference_tokens,     # 永不截断
    brand_tokens,         # 永不截断
    product_role_tokens,  # 永不截断
]

context = rank_by_priority([
    title_ref_window,
    series,
    structured_attrs,
    ocr_ref_window,
    title_rest,
    description,
])

return fit_to_budget(reserved, context, max_tokens)
```

这样即使 description 有几千字，reference 证据也不会被截走。

---

## 3.3 Data Augmentation / MixDA：适合训练脏数据鲁棒性，但不能破坏 reference 语义

Ditto 支持多种 augmentation：

- 删除 span；
- token/span shuffle；
- 删除整列；
- 把一个字段值移动到另一个字段；
- 多算子随机组合。

`ditto_light/ditto.py` 里的 MixDA 进一步对“原样本表示”和“增强样本表示”做插值：

```text
enc = lambda * enc_original + (1-lambda) * enc_augmented
```

其中 `lambda` 来自 Beta 分布。

这些方法的价值是让模型适应：

- 字段缺失；
- 字段错位；
- 标题少词；
- 词序变化；
- 平台脏数据。

但当前业务不能直接随机删除或改写 reference，因为 label 的语义会改变。

例如正样本：

```text
左：Rolex 126334
右：劳力士 126334
label = match
```

如果增强器把右侧 `126334` 删掉，仍保留 match label，模型就会被教成：

> “只要标题/系列足够像，即使看不到 reference 也可以判 Match。”

这正好违反当前 precision-first 约束。

所以需要实现 **ReferenceSafeAugmenter**：

可以随机破坏：

- 营销词；
- price；
- seller；
- 无关 description；
- 字段顺序；
- 非关键属性。

禁止随机破坏：

- `CANONICAL_REF`；
- `REF_CANDIDATE`；
- reference 归属标签；
- brand 硬冲突；
- product_role；
- 用于确定“不属于当前商品”的 accessory context。

对 hard negative 反而应该刻意生成更难的训练对：

```text
126334  vs 126300
126334  vs 126333
116500LN vs 116500
5711/1A-010 vs 5712/1A-001
同系列同尺寸不同 reference
同 reference 字符串出现在“适配/表带/配件”上下文
品牌一致 + 标题高度相似 + reference 不同
平台 SKU 恰好与另一品牌 reference 形态相似
```

这些才是当前模型必须学会坚决拒绝的样本。

---

## 4. 论文结果为什么和当前腕表场景高度相关，但不能直接拿 F1 当上线标准

Ditto 使用的 WDC 商品语料包含 2600 万商品 offer，并单独有 watches 类目。

其黄金测试数据里，四个品类各有 1,100 个人工标注 pair；训练数据中的正例依据 GTIN/MPN 等 product ID 构造，负例则刻意选择 **文本相似度高但 product ID 不同** 的商品。

这与当前问题非常接近：

```text
“标题很像、同品牌、同系列，但 reference 不同”
```

正是最危险的 false positive 来源。

论文中 Ditto 在 WDC watches 上报告的 F1 为：

| 训练规模 | Watches F1 |
|---|---:|
| xLarge | 96.53 |
| Large | 95.69 |
| Medium | 91.12 |
| Small | 85.12 |

这说明预训练 LM 对商品 hard negative 确实有价值，尤其少样本下比旧模型更强。

但这里必须强调：

> **96.53 F1 对普通商品匹配很强，对“绝对不能误匹配”仍远远不够。**

F1 同时平衡 precision 和 recall；当前需求实际上愿意牺牲 recall，因此阈值选择、模型目标、上线 gate 都必须重写。

更关键的是，当前 GitHub `evaluate()` 默认会在验证集上按 0.05 步长搜索 **F1 最大** 的 threshold。

```python
for th in np.arange(0.0, 1.0, 0.05):
    ...
    new_f1 = metrics.f1_score(...)
```

这个 threshold 策略不能用于本项目。

---

## 5. 对当前 Spec 的正确改造：Ditto 不做“Match 授权”，只做“Reference 语义验证”

最安全的改造不是训练一个：

```text
(left_record, right_record) -> same_product ?
```

然后分数大于 0.99 就自动合并。

而是拆成两个更窄、更可解释的任务。

## 5.1 Task A：Reference Attribution Verifier

判断某个从记录中抽出来的 reference 是否真的属于“当前售卖商品主体”。

输入：

```text
Record + candidate_reference
```

输出：

```text
BELONGS_TO_ITEM
NOT_BELONGS_TO_ITEM
ABSTAIN
```

需要重点识别：

- 标题中的兼容型号；
- 表带/表盒/配件适配型号；
- 对比文案中提到的其它型号；
- 平台 SKU；
- 店铺货号；
- 序列号；
- OCR 在背景/保卡上识别出的非当前商品信息。

例如：

```text
title = "适配劳力士126334/126300的钢带 20mm"
candidate_ref = "126334"
```

字符串抽取器会找到 `126334`，但 attribution verifier 必须输出：

```text
NOT_BELONGS_TO_ITEM
reason = ACCESSORY_COMPATIBILITY_CONTEXT
```

只有 `BELONGS_TO_ITEM` 且通过高精度校准的 reference 才能进入 canonical reference key。

## 5.2 Task B：Reference Consistency / Conflict Verifier

即使两条记录已经解析出了相同 canonical reference，也可以再用一个语义模型做“只否决、不授权”的安全检查。

输入：

```text
left.canonical_ref == right.canonical_ref
```

然后模型检查：

- brand 是否强冲突；
- product_role 是否一个是腕表、一个是配件；
- 标题是否明确出现另一个更可信 reference；
- OCR 是否出现冲突 reference；
- 一个 reference 是否只是兼容/对比上下文；
- 型号系列/尺寸是否出现不可能组合。

输出：

```text
CONSISTENT
CONTRADICT
ABSTAIN
```

自动合并规则：

```text
只有：
canonical_ref 严格一致
AND brand 不冲突
AND reference role 合法
AND attribution 通过
AND verifier != CONTRADICT
AND graph audit 通过

才能自动合并。
```

注意：`CONSISTENT` 本身也不是授权理由；授权仍来自 reference exact evidence。

---

## 6. 推荐的端到端系统架构

把目前 `b` 已分析的几个项目组合起来，可以形成如下 production pipeline：

```text
                    ┌──────────────────────┐
                    │  三源原始商品数据     │
                    └──────────┬───────────┘
                               │
                               ▼
                  ┌─────────────────────────┐
                  │  0. Source Adapter      │
                  │  字段映射 / 文本清洗     │
                  └──────────┬──────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │ 1. Reference Candidate Extract│
              │ 字段 / title regex / OCR / NER│
              └───────────┬──────────────────┘
                          │
                          ▼
        ┌─────────────────────────────────────────┐
        │ 2. Number Role Gate                     │
        │ BRAND_REF / SKU / SERIAL / ACCESSORY... │
        │ 参考 parts-distributor-sku-classifier   │
        └────────────────┬────────────────────────┘
                         │
                         ▼
        ┌─────────────────────────────────────────┐
        │ 3. Reference Attribution Verifier       │
        │ Ditto-style cross-encoder + abstention  │
        └────────────────┬────────────────────────┘
                         │
                         ▼
        ┌─────────────────────────────────────────┐
        │ 4. Brand-aware Canonicalization         │
        │ only proven equivalence rules           │
        └────────────────┬────────────────────────┘
                         │
              ┌──────────┴───────────┐
              │                      │
              ▼                      ▼
  ┌─────────────────────┐   ┌───────────────────────┐
  │ Exact Ref Index      │   │ Missing/Ambiguous Ref │
  │ (brand, canonicalRef)│   │ DeepBlocker candidates│
  └──────────┬──────────┘   └───────────┬───────────┘
             │                          │
             │                          ▼
             │               人工/OCR/抽取补证，
             │               不允许纯相似度越权合并
             │
             ▼
  ┌─────────────────────────────┐
  │ 5. Conflict Verifier        │
  │ Ditto-style semantic veto   │
  └───────────┬─────────────────┘
              │
              ▼
  ┌─────────────────────────────┐
  │ 6. Deterministic Safety Gate│
  └───────────┬─────────────────┘
              │
              ▼
  ┌─────────────────────────────┐
  │ 7. TransClean Graph Audit   │
  └───────────┬─────────────────┘
              │
              ▼
     AUTO_MATCH / REVIEW / REJECT
```

### 6.1 最重要的架构原则

整个系统里只有一类证据可以“主动授权”自动匹配：

```text
同品牌语境下、来源合法、归属当前商品、经过安全 canonicalization 的 reference 严格一致
```

以下信号都只能用于候选、排序或否决：

- 文本 embedding 相似；
- 图片相似；
- 同系列；
- 同尺寸；
- 同价格区间；
- Ditto match probability；
- OCR 模糊字符串相似；
- DeepBlocker KNN 分数。

这样可以从架构层避免“模型某次置信度过高就错误合并”的事故。

---

## 7. 推荐的数据模型

建议先把“reference 证据”从商品主表拆出来，不要只在商品表上维护一个 `reference` 字符串。

### 7.1 `reference_evidence`

```sql
CREATE TABLE reference_evidence (
    evidence_id            BIGINT PRIMARY KEY,
    record_id              BIGINT NOT NULL,
    source                 VARCHAR(32) NOT NULL,

    raw_reference          VARCHAR(128),
    normalized_reference   VARCHAR(128),
    canonical_reference    VARCHAR(128),
    brand_id               BIGINT,

    evidence_type          VARCHAR(32),   -- FIELD/TITLE/OCR/MANUAL
    source_field           VARCHAR(64),
    source_span            TEXT,
    image_id               BIGINT,

    number_role            VARCHAR(32),   -- BRAND_REFERENCE/PLATFORM_SKU/...
    role_score             DOUBLE,

    attribution_decision   VARCHAR(16),   -- BELONGS/NOT_BELONGS/ABSTAIN
    attribution_score      DOUBLE,

    extractor_version      VARCHAR(64),
    normalizer_version     VARCHAR(64),
    verifier_version       VARCHAR(64),

    decision_status        VARCHAR(32),
    created_at             TIMESTAMP
);
```

### 7.2 `match_decision`

```sql
CREATE TABLE match_decision (
    left_record_id         BIGINT NOT NULL,
    right_record_id        BIGINT NOT NULL,

    brand_id               BIGINT,
    canonical_reference    VARCHAR(128),

    decision               VARCHAR(16),   -- AUTO_MATCH/REVIEW/REJECT
    reason_codes           JSON,

    conflict_score         DOUBLE,
    gate_version           VARCHAR(64),
    verifier_version       VARCHAR(64),
    graph_audit_version    VARCHAR(64),

    decided_at             TIMESTAMP,
    PRIMARY KEY (left_record_id, right_record_id)
);
```

`reason_codes` 应该记录机器可读原因，例如：

```json
[
  "CANONICAL_REF_EXACT",
  "BRAND_EXACT",
  "REFERENCE_BELONGS_TO_ITEM",
  "NO_SECONDARY_REF_CONFLICT",
  "NO_PRODUCT_ROLE_CONFLICT",
  "GRAPH_TRANSITIVE_CONSISTENT"
]
```

这样每一次自动合并都可以追溯“为什么”。

---

## 8. Ditto-style verifier 的建议输入格式

不要把所有字段原样塞进去。建议构造高度受控的 serializer：

```text
COL source VAL she_dang_jia
COL product_role VAL WATCH
COL brand VAL ROLEX
COL canonical_ref VAL 126334
COL raw_ref VAL 126334
COL ref_role VAL BRAND_REFERENCE
COL ref_evidence VAL TITLE_EXPLICIT
COL title_ref_context VAL 劳力士 日志型 41 蓝盘 126334 全套
COL series VAL DATEJUST
COL size VAL 41mm
COL ocr_ref VAL NULL
```

对于 pair consistency：

```text
<LEFT>
COL brand VAL ROLEX
COL canonical_ref VAL 126334
COL title_ref_context VAL ...
COL product_role VAL WATCH

[SEP]

<RIGHT>
COL brand VAL ROLEX
COL canonical_ref VAL 126334
COL title_ref_context VAL ...
COL product_role VAL WATCH
```

这里最重要的是：

- `canonical_ref` 始终在前；
- `raw_ref` 也保留，避免 canonicalization 掩盖异常；
- `ref_role` 显式进入模型；
- reference 周围上下文比整段 description 更优先；
- 每个来源使用统一语义字段名，而不是平台原始字段名。

---

## 9. 训练数据：几百对黄金标签应该花在哪里

Spec 允许人工标注几百对。对于 precision-first 系统，不应该平均随机抽样，而应该集中在**最危险边界**。

推荐标注池：

### 9.1 40%：同品牌同系列相邻 reference hard negative

```text
126334 vs 126300
126334 vs 126333
同系列不同盘面/材质/尺寸导致的不同 reference
```

这是最接近线上 false positive 的样本。

### 9.2 20%：reference 角色歧义

```text
平台 SKU vs 品牌 reference
序列号 vs reference
机芯号 vs reference
库存号 vs reference
```

### 9.3 20%：reference 出现在非主体上下文

```text
适配某型号的表带
盒证/附件说明
对比文案
“同款参考 126334”但售卖的是另一型号
```

### 9.4 10%：OCR 冲突

```text
标题一个 reference
保卡 OCR 另一个 reference
背景图片出现其它表款 reference
OCR 字符 0/O、1/I、8/B 混淆
```

### 9.5 10%：普通正例

正例不需要太多，因为系统本身已经有 exact reference 强规则。人工预算更应该买“系统不知道自己会错在哪里”。

### 9.6 数据切分必须按 reference family 做 group split

不能普通 random row split。

如果：

```text
126334-A / 126334-B 类似记录进入 train
126334-C 进入 valid
```

模型很可能只是记住编号形态，验证结果会虚高。

建议至少按以下键 group split：

```text
brand + reference family / series
```

并额外建立一个 `future/unseen-reference` holdout，模拟增量中新型号。

---

## 10. 阈值和拒识：不能再“找 F1 最优点”

原 Ditto 的阈值搜索目标是最大化 F1，这与本需求完全不同。

建议输出三段决策：

```text
score <= T_reject       -> REJECT / CONTRADICT
T_reject < score < T_ok -> ABSTAIN / REVIEW
score >= T_ok           -> CONSISTENT（注意：仍不等于授权 Match）
```

对 `Reference Attribution Verifier` 同理：

```text
高置信 NOT_BELONGS -> 丢弃该 reference candidate
中间区间           -> 人工复核
高置信 BELONGS     -> 允许进入 canonicalization
```

### 10.1 “几百对标签”无法统计证明 99.9% precision

这一点必须提前写进验收标准。

假设某个自动接受区在独立验证集上连续 300 个样本全部正确，即 300/300、0 false positive。即使如此，用简单的单侧 95% binomial 置信下界估算，其真实 precision 下界也只有约 **99.0%**，并不能证明 99.9%。

若希望在“0 observed error”的前提下把 95% 单侧下界推到约 99.9%，样本量需要接近 **3,000 个自动接受样本**。

因此：

> **几百个黄金标签足以训练/筛选模型，但不足以证明模型概率本身满足近零误匹配。**

“绝不能误匹配”必须主要依赖：

- reference exact key；
- 编号角色闸门；
- brand-aware canonicalization；
- deterministic conflict rules；
- abstention；
- 图级一致性审计；
- shadow / canary 验证。

模型只负责处理语义噪声，不能成为最终安全证明。

---

## 11. 建议的自动匹配 Gate

可以把最终规则写成显式策略：

```python
def decide_pair(left, right):
    # 1. 必须先有可用 reference
    if not left.ref.accepted or not right.ref.accepted:
        return REVIEW("MISSING_TRUSTED_REFERENCE")

    # 2. 编号必须确认属于商品主体
    if left.ref.attribution != "BELONGS" or right.ref.attribution != "BELONGS":
        return REVIEW("REFERENCE_ATTRIBUTION_UNCERTAIN")

    # 3. 品牌硬冲突直接拒绝
    if left.brand_id and right.brand_id and left.brand_id != right.brand_id:
        return REJECT("BRAND_CONFLICT")

    # 4. canonical reference 必须严格一致
    if left.ref.canonical != right.ref.canonical:
        return REJECT("REFERENCE_NOT_EQUAL")

    # 5. 商品角色必须兼容
    if left.product_role != "WATCH" or right.product_role != "WATCH":
        return REVIEW("PRODUCT_ROLE_CONFLICT")

    # 6. Ditto-style 模型只负责发现语义冲突
    verdict = conflict_verifier(left, right)
    if verdict == "CONTRADICT":
        return REVIEW("SEMANTIC_CONFLICT")
    if verdict == "ABSTAIN":
        return REVIEW("VERIFIER_ABSTAIN")

    # 7. 图级安全审计
    if not transitive_consistency_check(left, right):
        return REVIEW("GRAPH_CONFLICT")

    return AUTO_MATCH("CANONICAL_REFERENCE_EXACT_AND_AUDITED")
```

这里故意把很多异常送到 `REVIEW` 而不是自动 `REJECT`，因为业务允许漏匹配，且人工几百条样本本来就应该优先用于这些高价值边界。

---

## 12. 性能与部署：千万级数据下 Ditto 应该放在哪里

Ditto 论文自己的大规模案例也明确将 blocker 和 matcher 分开；它不是为了做 N×M 全量比较。

当前系统可采用两条 fast path：

### Fast Path A：可信 reference 直接索引

对已经拿到可信 canonical reference 的记录：

```text
key = hash(brand_id, canonical_reference)
```

写入倒排索引：

```text
reference_index[key] -> [record_id_1, record_id_2, ...]
```

新记录增量到来时复杂度近似 O(1) 查 key，而不是和全库比较。

### Fast Path B：疑难记录才进入模型

只有以下记录调用 Ditto-style verifier：

- 一个标题抽出多个 candidate reference；
- reference 只来自 OCR；
- 编号角色分类器置信度不够；
- exact reference 相同但有强冲突属性；
- DeepBlocker 召回了疑似同款，但当前记录缺少可信 reference；
- 图审计发现某个 component 内部不一致。

也就是说，Transformer QPS 应该跟“疑难比例”相关，而不是跟总记录量线性相关。

### 12.1 增量处理建议

```text
Kafka / CDC
   -> normalize record
   -> extract reference candidates
   -> role gate
   -> attribution verifier（必要时）
   -> canonicalize
   -> exact reference index lookup
   -> conflict verifier（必要时）
   -> graph audit
   -> persist decision + provenance
```

历史 1000 万数据先离线批处理；新增数据走同一套 versioned pipeline。

---

## 13. 一个可以直接落地的 MVP

不建议先完整复刻 Ditto 原仓库。原项目依赖较旧的 PyTorch / Transformers / Apex 组合，更适合作为算法参考。

MVP 可以只复刻其核心架构：

```text
Serializer
  + Domain Knowledge Tags
  + Transformer Cross-Encoder
  + Classification Head
  + Abstention Thresholds
```

### Sprint 1：只做 reference attribution

输入：

```json
{
  "record_id": 123,
  "brand": "ROLEX",
  "title": "劳力士日志型126334蓝盘41mm全套",
  "candidate_reference": "126334",
  "candidate_source": "TITLE"
}
```

输出：

```json
{
  "decision": "BELONGS",
  "score": 0.9987,
  "reason_codes": ["TITLE_DIRECT_REFERENCE"],
  "model_version": "ref-attr-v1"
}
```

### Sprint 2：加入 hard-negative conflict verifier

训练重点：

```text
same brand + same series + different reference
```

模型上线先只 shadow，不改变匹配结果，观察它能发现多少当前规则遗漏的冲突。

### Sprint 3：接入 deterministic safety gate

模型只允许：

```text
veto / abstain / review ranking
```

不允许仅凭分数新建自动 match edge。

### Sprint 4：接 TransClean 图审计

每次准备把一条边写入实体簇之前，检查是否引入：

- 一个 component 多个 canonical reference；
- brand 冲突；
- product_role 冲突；
- 高置信 verifier contradiction。

---

## 14. 监控指标：不要把 Overall F1 放在第一位

生产看板建议至少拆成：

### 最重要

```text
Auto-match Precision
Auto-match False Positive Count
Reference Collision Count
Cross-brand Auto-match Count（目标 0）
Multi-reference Component Count（目标 0）
Verifier Contradiction After Auto-match（目标 0）
```

### 系统效率

```text
Auto-match Coverage
Review Rate
Abstain Rate
Candidate Pairs / New Record
Verifier QPS / P95 Latency
DeepBlocker Recall on gold set
```

### 漂移

```text
Unknown Brand Rate
Unknown Reference Pattern Rate
Reference Role Distribution by Source
OCR-only Reference Rate
Model score distribution by source/brand
```

必须按：

```text
source × brand × category × reference family
```

分桶监控，不能只看全局平均。

---

## 15. 最终结论

Ditto 最值得当前项目借鉴的不是“用 BERT 替代规则做同款判断”，而是四个更底层的设计：

1. **Blocker / Matcher 分层**：大规模候选召回与精细语义判别解耦；
2. **结构化序列化**：schema 不统一的数据可以用带字段角色的 token 统一输入；
3. **Domain Knowledge 显式注入**：Product ID / Brand / Configurations 这类强信号应告诉模型，而不是让模型从几百条标签里自己猜；
4. **Hard-negative 学习**：文本极像、reference 不同的商品必须成为训练核心。

但原 Ditto 有三个地方不能照搬：

- 不能用 F1 最大化选择上线阈值；
- 不能让通用 `ID`/NER 规则代替品牌级 reference 解析；
- 不能让 Match probability 越权替代 reference exact evidence。

因此，对当前 Spec 最合适的落地方式是：

> **把 Ditto 改造成 Reference Attribution + Conflict Verifier，只做“证明编号属于当前商品、发现语义冲突、排序人工复核”，真正的自动匹配仍必须经过 `(brand_id, canonical_reference)` 严格等值、安全 canonicalization、deterministic gates 和 TransClean 图一致性审计。**

这样既利用了预训练语言模型对脏文本、字段缺失、同系列 hard negative 的理解能力，又不会把系统安全性押在不可解释的相似度概率上。

---

## 参考资料

- Yuliang Li, Jinfeng Li, Yoshihiko Suhara, AnHai Doan, Wang-Chiew Tan. **Deep Entity Matching with Pre-Trained Language Models**. PVLDB 2021. https://arxiv.org/abs/2004.00584
- Ditto 官方实现：https://github.com/megagonlabs/ditto
- 当前需求 Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31
