# LaTeX-Numeric：把“远程监督 + 缺失标签 Masking”改造成 Reference-First 腕表型号抽取系统

> 分析人：c  
> 目标需求：跨源二奢/腕表商品实体匹配（雷小安 × 腕表之家 × 奢当家）  
> Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31  
> 核心约束：100 万–1000 万级持续增量；字段高度稀疏；reference 可能在结构化字段、标题或图片中；“同一个商品”定义为**同一 reference number / 型号**；**precision 极端优先，绝不能误匹配，允许漏匹配**。

## 0. 选题、去重与结论先行

本次从 `奢侈品文章调研.md` 选取：

- 论文：**LaTeX-Numeric: Language-agnostic Text attribute eXtraction for E-commerce Numeric Attributes**
- 作者：Kartik Mehta, Ioana Oprea, Nikhil Rasiwasia（Amazon）
- NAACL Industry 2021
- 论文：https://aclanthology.org/2021.naacl-industry.34/
- PDF：https://aclanthology.org/2021.naacl-industry.34.pdf

执行前重新检查 `奢侈品调研/c/`。当前 `c` 已有 22 篇分析，包含 DeepBlocker、Ameli、AnyMatch、TransClean、GraLMatch、pyJedAI、Confidence Classifier、Conformal Selective Prediction、RAG 属性抽取、多模态属性抽取等；目录中不存在本论文标题，因此满足“每次分析前排除已分析文章”的要求。

### 结论先行

这篇论文对当前 Spec **有很高的工程参考价值，但不能原样照搬**。

最值得迁移的不是它针对 numeric attribute 的“单位 alias”，而是两个架构思想：

1. **用已有结构化字段做 distant supervision，自动从海量商品文本构造训练数据；**
2. **结构化字段缺失时，不要把文本中潜在正确的 reference 错标成 `O`，而是对该任务直接 mask loss。**

把它改造成当前业务后，推荐的主链路不是“训练一个模型直接判断两条商品是否相同”，而是：

```text
三源原始 listing
    -> 品牌 / 商品角色归一化
    -> Reference Candidate Generator
    -> Reference Role Classifier / Extractor
       （远程监督训练，缺失字段时 mask）
    -> Collision-safe Canonicalizer
    -> (brand_id, canonical_reference) Exact Index
    -> Strict Evidence Gate
    -> AUTO_LINK / ABSTAIN / REJECT
```

最终匹配仍然应该是**reference exact equality 的证据系统**，模型只负责从稀疏、脏、非结构化数据里恢复 reference，并判断某个“像型号的字符串”到底是不是当前商品自己的官方 reference。

> **不要让语义相似度、图片相似度或 NER 置信度成为“同一商品”的最终定义。当前需求已经给出了更强的业务主键：reference number。**

---

## 1. 论文到底解决了什么问题

论文处理的是大规模电商“属性值抽取”：商品本来存在结构化属性，但卖家填写不完整；同一个值往往又出现在标题或描述中，因此可以从文本恢复结构化字段。

作者把属性抽取建模为 NER：

```text
产品文本 tokens
    -> character encoder
    -> word encoder
    -> BiLSTM
    -> CRF / token output
    -> BIO attribute tags
```

训练数据不靠人工逐 token 标，而是直接用商品已有的结构化 attribute value 去文本里匹配，自动产生 BIO 标签。这就是 distant supervision。

论文使用 CNN-BiLSTM-CRF 作为主要底座，也验证了 BERT；它并没有把模型结构本身当作最大创新，而是重点解决**远程监督标签质量**。

这点与当前三源腕表数据非常像：

```text
结构化 reference 有值的一部分商品
          ↓
在 title / description / OCR 中找同值或安全变体
          ↓
自动得到大量 reference span 标签
          ↓
训练抽取器
          ↓
给结构化 reference 缺失的商品补 reference 候选
```

如果现有三平台中只要有一部分记录带明确型号字段，就已经拥有了一个天然的弱监督“标注工厂”。

---

## 2. 论文最重要的发现：Missing-PA

### 2.1 朴素 distant supervision 会制造“伪负例”

假设一个商品文本里有：

```text
... 16 GB RAM ... weighs 1.2 kg ...
```

但结构化字段里 RAM 有值、Weight 缺失。

朴素做法会把 RAM 标出来，却把 `1.2 kg` 当作 `O`。实际上它是正确的 Weight，只是数据库字段缺失。

作者称之为 **Missing-PA（missing partial annotation）**。

这在当前 reference 场景里更危险：

```text
title = "劳力士潜航者 126610LN 黑水鬼 ..."
structured_reference = null
```

如果我们因为 `structured_reference = null` 就把标题中 `126610LN` 标成非 reference，那么训练集会系统性地教模型：

> “真正的 reference 在字段缺失时是负例。”

而当前业务恰恰要靠模型去补这些缺失记录，因此这个错误会直接打击最重要的长尾样本。

### 2.2 MAST：一个模型、所有属性共用一个标签空间

论文把常规多属性 NER 称为 MAST-NER：

```text
K 个属性
-> 每个属性 B / I 两个标签
-> 再加统一 O
-> 共 2K + 1 个标签
```

问题是，只要某个属性的结构化值缺失，就无法正确标注该属性在文本中的 token；它们很容易落成 `O`。

### 2.3 MAMT：共享 encoder，但每个属性单独一个 task/head

作者提出 MAMT-NER：

```text
                  ┌─ Task head: attribute_1
char/word encoder ├─ Task head: attribute_2
      + BiLSTM ───┼─ Task head: attribute_3
                  └─ Task head: attribute_K
```

共享：

- character embedding / encoder；
- word embedding / encoder；
- BiLSTM contextual layer。

每个属性拥有独立输出任务。

关键并不只是“多头”，而是**loss masking**：

```python
if sample.attribute_k is missing:
    loss_k = 0
else:
    loss_k = token_loss(head_k, weak_labels_k)

loss = mean(active_losses)
```

也就是说：

> 字段缺失 = “不知道标签”，而不是“标签为 O”。

这是一条非常适合本需求的数据工程原则。

论文在 numeric attributes 上报告 MAMT 相比 MAST 平均 F1 提升 9.2%；换 BERT 底座仍有提升，说明这个机制并不依赖 BiLSTM。

---

## 3. 另一个核心：自动 alias 生成

numeric attribute 有一个典型问题：数据库里可能是 canonical unit，但商品文本存在不同表面形式。

例如：

```text
display_size: 13 inch / 13 inches / 13 in
weight: kg / kilograms / lbs / pounds
```

如果 distant supervision 只匹配一个 canonical form，会漏掉很多正例。

论文构建了两类 alias：

### 3.1 `alias_dw`：由已知 attribute value 反推附近单位

给定结构化 attribute value，在文本中找这个数值，取其后出现的字母 token 作为单位候选。

类似：

```text
structured display_size = 13
text = "13 inches display"
-> candidate alias = inches
```

为减少碰撞，作者会丢弃一条文本中存在多个可疑匹配的情况。

这一点值得直接借鉴：

> **弱监督标签生成宁可少，不要在歧义样本上硬标。**

### 3.2 `alias_bp`：扫描所有“数字 + 单位”候选，再用 embedding 过滤

作者还扫描文本里所有 numeric mention 后的单位候选，再利用 canonical unit 与候选词的 embedding similarity 过滤。

这样可以发现需要换算的单位，例如：

```text
kg -> pounds / lbs
```

最后把两类 alias 合并。

### 3.3 Exclusive Alias

论文用一个小 dev set 检查某个单位是否高度专属于某属性；如果 precision 高于阈值，就允许仅凭“数值 + 该单位”扩大弱监督标注。

这是典型的：

```text
高精度规则先被人工验证
    -> 再用于自动扩标
    -> 扩标后训练模型
```

而不是让模型自由猜测新标签。

### 3.4 效果

论文表 4 的数值是**相对 canonical-aliasing baseline 的相对指标**，不是绝对 F1；auto-aliasing 平均达到 120.2，相当于作者报告平均 F1 改善 20.2%。

这说明弱监督阶段的“标签生成规则”本身可以比换模型带来更大的收益。

对当前项目，这个结论非常关键：

> **先把 reference 候选、角色、格式归一化和冲突规则做好，再谈大模型。**

---

## 4. 为什么不能把论文的 alias 机制原样用到腕表 reference

numeric attribute 与 reference number 有本质差别。

numeric alias 往往允许“语义等价单位”：

```text
kg ≈ pounds
inch ≈ cm
```

但 reference 是离散标识符：

```text
126610LN != 126610LV
116610LN != 126610LN
```

只差一个字符，也可能是完全不同 reference。

因此当前业务里的“alias”只能包含**被证明不改变标识语义的表面格式变体**，例如：

```text
126610LN
126610 LN
126610-LN
ref. 126610LN
型号：126610LN
```

而不应该包含：

```text
编辑距离 <= 1
共同前缀
同系列 embedding 相近
视觉相似
```

### 推荐把 canonicalization 拆成两层

#### Layer A：Surface Normalization

只做不太可能改变语义的操作：

```text
Unicode NFKC
英文字母统一大写
全角 -> 半角
去外层空白
标准化明显分隔符
去 `Ref.` / `型号` 等字段标签
```

输出：

```text
surface_reference
```

#### Layer B：Dictionary Resolution

只有当品牌知识库能够证明若干 surface form 唯一对应同一个 reference 时，才产生：

```text
canonical_reference
```

例如：

```text
ROLEX + "126610 LN"
ROLEX + "126610-LN"
-> 126610LN
```

但如果一个规则可能把两个真实 reference 折叠到同一个字符串，则**禁止自动 canonicalize**。

推荐保存完整 provenance：

```json
{
  "raw": "Ref. 126610-LN",
  "surface_normalized": "126610-LN",
  "canonical_reference": "126610LN",
  "normalizer_version": "rolex-v3",
  "rule_id": "remove-safe-separator",
  "collision_checked": true
}
```

这样以后修改归一化规则可以回溯，而不是不可逆地改写历史数据。

---

## 5. 把 MAMT 改造成“Reference Role Extraction”

当前真正难的不是找出“像编号的字符串”，而是判断这个编号是什么角色。

一条商品页里可能同时出现：

- 当前整表官方 reference；
- 平台货号；
- 店铺 SKU；
- 库存号；
- 配件适配的腕表 reference；
- 保卡/证书上的编号；
- 文章或描述里比较的其它型号；
- 序列号/机芯号。

因此建议不要只训练一个 `REFERENCE / O` 二分类 NER，而是把论文的多任务思想改为**编号角色多任务抽取**。

### 5.1 推荐任务头

```text
shared encoder
├─ own_reference_head
├─ compatible_reference_head
├─ platform_sku_head
├─ inventory_id_head
├─ serial_number_head
└─ brand_head / product_role_head（可独立模型）
```

每个 head 可以是 BIO token classification；如果先用 regex 生成候选 span，则也可以更简单地把任务改成 span role classification。

我更推荐线上第一版采用：

```text
Broad Candidate Generator
    -> Span Role Classifier
```

而不是开放式 NER。

原因是 reference 通常是短字母数字串，regex + 品牌词典很容易得到高召回候选；模型最有价值的地方是“判角色”，不是“生成字符串”。

### 5.2 Missing Label Masking 如何迁移

对一条 listing：

```python
for task in tasks:
    value = structured_field[task]

    if value is None:
        # 未知，而非负例
        task_mask[task] = 0
        continue

    spans = safe_exact_alias_match(text, value)

    if unique_and_unambiguous(spans):
        labels[task] = make_bio(spans)
        task_mask[task] = 1
    else:
        # 弱监督碰撞时宁可不训练
        task_mask[task] = 0
```

这与论文 MAMT 的精神一致，同时更符合 precision-first。

### 5.3 为什么这比“标题里抓正则”强

一个简单正则可能抓到：

```text
适配劳力士 116610LN / 126610LN 的表带
```

如果售卖的是表带，两个 reference 都不是“当前商品 reference”。

Role Classifier 应输出：

```text
116610LN -> compatible_reference
126610LN -> compatible_reference
product_role -> accessory
```

随后 Strict Gate 直接拒绝把它并入任何腕表实体。

---

## 6. 推荐直接落地的系统：RefLaTeX

可以把论文思路改造成一个内部组件，暂称 **RefLaTeX**。

## 6.1 总体架构

```text
                 ┌──────────────────────────────┐
雷小安 ---------->|                              |
腕表之家 -------->| Raw Listing / Change Log     |
奢当家 ---------->|                              |
                 └──────────────┬───────────────┘
                                |
                                v
                 ┌──────────────────────────────┐
                 | Normalize Brand/Product Role |
                 └──────────────┬───────────────┘
                                |
                                v
                 ┌──────────────────────────────┐
                 | Reference Candidate Generator |
                 | structured/title/desc/OCR     |
                 └──────────────┬───────────────┘
                                |
                                v
                 ┌──────────────────────────────┐
                 | Role Classifier / Extractor  |
                 | weak-supervised + task mask  |
                 └──────────────┬───────────────┘
                                |
                                v
                 ┌──────────────────────────────┐
                 | Collision-safe Canonicalizer |
                 └──────────────┬───────────────┘
                                |
                                v
                 ┌──────────────────────────────┐
                 | Reference Evidence Store     |
                 └──────────────┬───────────────┘
                                |
                                v
                 ┌──────────────────────────────┐
                 | Exact Index                  |
                 | (brand_id, canonical_ref)    |
                 └──────────────┬───────────────┘
                                |
                                v
                 ┌──────────────────────────────┐
                 | Strict Evidence Gate         |
                 └───────┬───────────┬──────────┘
                         |           |
                    AUTO_LINK     ABSTAIN/REVIEW
                         |
                         v
                 Canonical Product Entity
```

注意：**没有 pairwise semantic matcher 作为主链路。**

既然业务定义已经是“reference 相同”，最便宜、最可靠的最终匹配器就是：

```text
(brand_id, canonical_reference) exact equality
```

机器学习只负责恢复这个 key。

---

## 7. 数据模型：不要直接“merge row”，要保存证据边

为了满足“绝不能误匹配”和可回滚，建议不要物理合并原始 listing，而是维护 evidence + link。

### 7.1 `listing_raw`

```sql
listing_raw(
    source,
    source_listing_id,
    version,
    raw_title,
    raw_description,
    raw_structured_json,
    image_urls,
    fetched_at,
    PRIMARY KEY(source, source_listing_id, version)
)
```

### 7.2 `reference_evidence`

```sql
reference_evidence(
    evidence_id,
    source,
    source_listing_id,
    brand_id,
    raw_reference,
    surface_reference,
    canonical_reference,
    channel,          -- structured/title/description/ocr
    role,             -- own/compatible/platform_sku/serial/other
    extractor,        -- rule/model/ocr/manual
    confidence,
    normalizer_version,
    model_version,
    provenance_json,
    created_at
)
```

### 7.3 `reference_entity`

```sql
reference_entity(
    entity_id,
    brand_id,
    canonical_reference,
    status,
    created_at,
    UNIQUE(brand_id, canonical_reference)
)
```

### 7.4 `listing_reference_link`

```sql
listing_reference_link(
    source,
    source_listing_id,
    entity_id,
    decision,          -- auto/manual/rejected/abstain
    evidence_ids,
    rule_version,
    decided_at
)
```

这样出现错误时可以撤销 link，而不破坏原始数据。

---

## 8. Weak Label Factory：把现有结构化 reference 变成免费训练集

### 8.1 安全标注规则

只从“可信结构化 reference”出发。

```text
structured_reference
    -> safe surface variants
    -> title/description exact span search
    -> 若唯一匹配：生成 own_reference 正例
    -> 若 0 个匹配：不强行标负
    -> 若多个冲突匹配：整条 task mask
```

### 8.2 不要生成普通随机负例

真正危险的负例是：

```text
Rolex 126610LN vs 126610LV
Rolex 116610LN vs 126610LN
Omega 同系列邻近 reference
整表 vs 适配该 reference 的表带
平台 SKU vs 品牌 reference
标题同时出现“现款型号”和“对比型号”
```

因此 gold set 和 hard-negative 训练集都应该以这类样本为主。

### 8.3 结构化字段缺失时必须 mask

错误：

```text
structured_reference = NULL
=> 所有候选 token = O
```

正确：

```text
structured_reference = NULL
=> own_reference task unknown
=> mask own_reference loss
```

这就是本论文对当前需求最值得直接落地的贡献。

---

## 9. Candidate Generator：先规则召回，再让模型判角色

### 9.1 候选来源

按可靠度建议分为：

```text
1. 官方/平台结构化 reference 字段
2. 标题
3. 商品描述
4. OCR：表背 / 保卡 / 吊牌 / 标签
5. 其它图片理解结果
```

### 9.2 Broad Regex 不要直接决定匹配

候选生成可以宽松，例如允许：

```text
[A-Z0-9][A-Z0-9./_-]{3,30}
```

再结合品牌特定 pattern。

但 regex 命中只意味着：

> “这里有一个像编号的 span。”

它不能意味着：

> “这是官方 reference。”

### 9.3 品牌内词典约束

维护：

```text
brand_id -> known reference dictionary
```

候选如果能唯一命中品牌词典，可信度大幅上升。

对未见 reference 不应直接丢弃，可以进入 extractor + review，但第一阶段不自动合并。

---

## 10. Strict Evidence Gate：最终 precision 必须由规则收口

论文优化目标仍然是 F1，而当前项目的目标函数完全不同。

尤其论文在非 numeric attribute 的表 6 里，MAMT 的相对结果是：

```text
Precision: 93.6
Recall:    117.8
F1:        107.4
```

即多任务机制可以通过 recall 提升获得更高 F1，但 precision 本身可能下降。

这恰好说明：

> **MAMT 可以做 extractor，但绝不能直接成为 AUTO_LINK 决策器。**

推荐证据等级：

### E3：强证据

```text
可信结构化字段
+ brand 已规范化
+ canonical reference 唯一
+ role = own_reference
```

### E2：中强证据

```text
标题/描述抽取
+ 模型高置信
+ 品牌词典唯一命中
+ role = own_reference
+ 无冲突 reference
```

### E1：弱证据

```text
OCR 单独出现
或视觉模型推断
或未知 reference
或 role 不确定
```

### 初始 AUTO_LINK 策略

建议 V1 非常保守：

```text
AUTO_LINK only if:
    A.brand_id == B.brand_id
    AND A.canonical_ref == B.canonical_ref
    AND each listing has >= 1 E3 own_reference evidence
    AND no conflicting E2/E3 own_reference exists
    AND product_role is watch
```

V2 在积累审计数据后，可以逐步开放：

```text
E3 + E2
```

但 `E2 + E2`、OCR-only、视觉-only 一律先 ABSTAIN。

---

## 11. 冲突比相似更重要：建立 Negative Gate

在 precision-first 系统里，强冲突应该拥有否决权。

```text
if brand conflict:
    REJECT

if own_reference has two distinct high-confidence canonical refs:
    ABSTAIN

if product_role in {strap, box, certificate, accessory}:
    REJECT_AUTO_LINK_AS_WATCH

if candidate role == compatible_reference:
    never use for entity key

if canonicalization collision detected:
    ABSTAIN
```

这比单纯训练一个更大的 matcher 更可靠。

图片也应该优先作为冲突信号：

```text
文本说某 reference
但表盘/表背 OCR 出现另一个明确 reference
=> ABSTAIN / REVIEW
```

而不是“图片相似所以覆盖 reference 冲突”。

---

## 12. 100 万–1000 万规模如何做：根本不需要 N² matching

如果业务定义是“同 reference 即同商品”，经过抽取后问题可以直接变成键连接：

```text
key = (brand_id, canonical_reference)
```

### 批处理

```sql
SELECT brand_id, canonical_reference, array_agg(listing_id)
FROM trusted_listing_reference
GROUP BY brand_id, canonical_reference;
```

### 增量处理

```text
新 listing 到达
-> extract evidence
-> resolve canonical reference
-> lookup exact key
-> existing entity: attach evidence edge
-> new key: create reference_entity
```

复杂度从潜在的 pairwise：

```text
O(N²)
```

变成近似：

```text
抽取 O(N)
+ exact index lookup O(1) / O(log N)
```

对于 1000 万 listing，这个差别比换任何 matcher 都更重要。

ANN / embedding blocking 只需要服务于：

- reference 完全抽不出来；
- 人工复核时寻找可能候选；
- 构造 hard negatives；
- 数据质量诊断。

它不应该成为主匹配路径。

---

## 13. 增量更新与版本化

持续增量场景必须版本化以下组件：

```text
brand_normalizer_version
reference_normalizer_version
candidate_generator_version
role_model_version
ocr_model_version
decision_rule_version
```

每条 link 记录当时使用的版本。

模型升级时：

```text
不要直接覆盖历史 entity mapping
-> 重算 reference_evidence
-> 生成新 decision proposal
-> 仅当满足当前规则才升级 link
-> 有冲突则进入 review
```

这样可以避免模型迭代导致大规模不可逆误合并。

---

## 14. 几百对黄金标签应该怎么花

Spec 允许人工标几百对。不要平均随机抽样，应该集中买“最有信息量的 precision 风险”。

推荐 500 左右起步：

### 150：同品牌、同系列、邻近 reference hard negatives

例如：

```text
126610LN vs 126610LV
116610LN vs 126610LN
```

### 100：整表 vs 配件/表带/盒证

重点包含“标题里写着适配型号”的样本。

### 100：结构化 reference 缺失，但标题/描述存在真实 reference

用来验证 Missing-PA masking 是否提高召回而不伤 precision。

### 75：OCR reference

包含表背、保卡、吊牌、多编号同时出现的图片。

### 75：canonicalization 边界

空格、点、斜杠、连字符、大小写、品牌不同格式。

### 一个重要统计现实

“几百对没有误匹配”并不能统计意义上证明 99.9%+ precision。

粗略使用 zero-error 的 rule-of-three：

```text
n = 500，0 个错误
95% 上界仍约为 3/500 = 0.6% error
```

所以如果业务要求接近“绝不误匹配”，几百条 gold 更适合：

- 发现规则漏洞；
- 校准 extractor；
- 挖 hard negatives；
- 决定哪些证据组合可以开放；

而不是声称“模型 precision 已被证明足够高”。

真正的安全性应来自：

```text
严格业务定义
+ exact canonical key
+ evidence provenance
+ negative gate
+ abstain
+ 可回滚 link
```

---

## 15. 评测方式必须从 F1 改成 Precision-First

建议至少报告：

```text
Extraction Precision by channel
Extraction Recall by channel
Role Classification Precision
Canonicalization Collision Rate
AUTO_LINK Precision
AUTO_LINK False Positive Count
AUTO_LINK Coverage
ABSTAIN Rate
Conflict Detection Recall
```

### 测试集切分不能随机 listing 切分

否则同一 reference 的不同 listing 很可能同时进 train/test，结果虚高。

推荐：

```text
split by canonical reference
```

同时单独建立：

- unseen reference set；
- same-series hard negative set；
- accessory set；
- OCR-only set；
- missing-structured-field set；
- normalization-collision set。

### 上线门槛

不建议写成：

```text
F1 > 0.95
```

而建议：

```text
AUTO_LINK hard-negative test: 0 false positive
+ production shadow audit pass
+ no unresolved canonical collision
+ evidence rule fully explainable
```

并逐证据等级开放覆盖率。

---

## 16. 最小可落地版本（两周量级的工程切分，不依赖大模型）

这里给出一个可以直接实现的 V0/V1，不要求先训练复杂模型。

### V0：规则 + Exact Index

1. 建 `listing_raw`；
2. 做品牌 canonicalization；
3. 读取三个来源已有 structured reference；
4. 实现 collision-safe reference normalizer；
5. 建 `(brand_id, canonical_reference)` unique entity index；
6. 只把 E3 + E3 作为 AUTO_LINK；
7. 所有冲突进入 review。

这一步就能建立非常高 precision 的基线。

### V1：Weak-supervised Reference Role Classifier

1. 从 E3 结构化字段自动找 title/description span；
2. 字段缺失时 mask，不标 O；
3. 多匹配/歧义样本丢弃；
4. 用 broad regex 构造 span candidates；
5. 训练 `own_reference / compatible_reference / sku / other` classifier；
6. E2 先只 shadow，不自动合并；
7. 人工审计高置信 E2，积累新 gold。

### V2：OCR Evidence

1. 图片 OCR；
2. 对 OCR span 走同一 canonicalizer + role/evidence pipeline；
3. OCR 只做佐证或冲突否决；
4. 经过独立审计后再考虑 E3 + OCR 的自动策略。

---

## 17. 一个可直接实现的弱监督伪代码

```python
TASKS = [
    "own_reference",
    "platform_sku",
    "serial_number",
]


def build_weak_labels(listing):
    text = normalize_text(listing.title + " " + listing.description)
    result = {}

    for task in TASKS:
        structured_value = get_structured_value(listing, task)

        # 论文 MAMT 最关键的迁移：missing != negative
        if not structured_value:
            result[task] = {"mask": 0, "labels": None}
            continue

        variants = safe_surface_variants(
            brand_id=listing.brand_id,
            value=structured_value,
        )

        spans = exact_find_variants(text, variants)

        # precision-first：歧义时不要自动造标签
        if len(spans) != 1:
            result[task] = {"mask": 0, "labels": None}
            continue

        result[task] = {
            "mask": 1,
            "labels": make_bio_labels(text, spans[0]),
        }

    return result
```

训练：

```python
loss = 0
active = 0

for task, weak in weak_labels.items():
    if weak["mask"] == 0:
        continue

    logits = heads[task](shared_encoder(tokens))
    loss += token_loss(logits, weak["labels"])
    active += 1

loss = loss / max(active, 1)
```

如果改成 span classifier，结构可以更简单，但“mask missing task”原则不变。

---

## 18. Canonical Reference Resolver 伪代码

```python
def resolve_reference(brand_id, raw_ref):
    surface = nfkc(raw_ref).upper().strip()
    surface = remove_safe_prefix(surface)  # REF / 型号等
    surface = normalize_safe_separators(surface)

    candidates = brand_reference_dictionary.lookup_surface(
        brand_id=brand_id,
        surface=surface,
    )

    if len(candidates) == 1:
        return Resolved(
            canonical=candidates[0],
            collision=False,
            provenance="brand_dictionary_unique",
        )

    if len(candidates) > 1:
        return Ambiguous(reason="normalization_collision")

    # 未知 reference 不做模糊替换
    return Unknown(surface=surface)
```

绝对不要：

```python
nearest = min(all_refs, key=edit_distance)
return nearest
```

因为“最近的 reference”在腕表业务里往往正是最危险的误匹配。

---

## 19. AUTO_LINK 决策伪代码

```python
def decide(listing_a, listing_b):
    if listing_a.brand_id != listing_b.brand_id:
        return REJECT("brand_conflict")

    a = best_own_reference_evidence(listing_a)
    b = best_own_reference_evidence(listing_b)

    if has_reference_conflict(listing_a) or has_reference_conflict(listing_b):
        return ABSTAIN("reference_conflict")

    if listing_a.product_role != "watch" or listing_b.product_role != "watch":
        return ABSTAIN("non_watch_or_uncertain_role")

    if not a or not b:
        return ABSTAIN("reference_missing")

    if a.canonical_reference != b.canonical_reference:
        return REJECT("reference_mismatch")

    # V1 only strongest evidence pair auto-links
    if a.level == "E3" and b.level == "E3":
        return AUTO_LINK("exact_reference_e3_e3")

    return ABSTAIN("insufficient_evidence")
```

这比“match_score > 0.98”更适合当前需求，因为每个自动决策都有可解释证据。

---

## 20. 与 c 已分析方案的组合方式

本论文并不是替代此前方案，而是可以补齐“reference 从哪里来”这一层。

一个更完整的组合架构可以是：

```text
LaTeX-Numeric 思想
    -> 弱监督训练 Reference Extractor
    -> Missing-PA mask
    -> 解决结构化字段稀疏

parts-distributor-sku-classifier 思想
    -> 区分官方 reference / 平台 SKU / 其它编号角色

RAG / Attribute Extraction
    -> 在品牌/reference 词典内做受限候选解析

DeepBlocker / ANN
    -> 只给 reference 缺失疑难样本召回候选

Confidence / Conformal Selective Prediction
    -> 控制哪些模型结果允许进入自动区

TransClean / GraLMatch
    -> 对多源 link graph 做冲突审计

最终 Strict Reference Gate
    -> exact canonical reference 才能形成自动实体链接
```

其中 **LaTeX-Numeric 提供的是“低成本造训练数据 + 正确处理缺失标签”的数据层能力**。

---

## 21. 这篇论文对 Spec 最值得记住的 6 条原则

1. **缺失字段不是负例。**
   `structured_reference = NULL` 时，reference task 应 mask，而不是标成 `O`。

2. **弱监督标签质量往往比换大模型更重要。**
   论文 auto-aliasing 带来的收益非常明显；本项目对应的是 reference normalization / role-aware weak labeling。

3. **歧义样本宁可不标。**
   作者遇到多重匹配会避免碰撞；当前需求更应该如此。

4. **多任务适合共享上下文，但不能直接承担最终 precision。**
   论文非 numeric 实验里 MAMT 提高 F1 时 precision 有下降，正说明必须另有 Strict Gate。

5. **reference 的 alias 必须 collision-safe。**
   只能做有证据的格式等价，不能做模糊编辑距离等价。

6. **匹配主键一旦恢复，就应退化成 exact index。**
   当前业务不是“找最相似商品”，而是“恢复并验证 reference identity”。

---

## 22. 最终建议

如果现在就开始实现，我建议优先顺序是：

```text
P0  Brand canonicalization
P0  Structured reference trust policy
P0  Collision-safe reference normalizer
P0  Evidence store + exact entity index
P0  E3+E3 strict auto-link

P1  Weak Label Factory
P1  Missing-label task masking
P1  Reference role classifier
P1  hard-negative gold set

P2  OCR evidence
P2  shadow E3+E2 policy
P2  graph-level conflict audit
```

不建议优先做：

```text
大型端到端 pair matcher
图片相似直接合并
reference 编辑距离近似匹配
LLM 自由生成 canonical reference
全量 ANN 代替 exact reference index
```

### 一句话落地方案

> **先用 LaTeX-Numeric 的 distant supervision + MAMT masking，从已有结构化字段低成本训练“reference 角色抽取器”；再用 collision-safe canonicalization 把可信候选映射到品牌 reference 字典；最终只通过 `(brand_id, canonical_reference)` 的严格等值和强证据 Gate 自动建边，所有模型不确定项一律 abstain。**

这条方案既能利用百万级已有数据扩大 reference 覆盖，又不会把模型的高 recall 直接转化成误匹配风险，和当前 Spec 的“precision 极端优先、允许漏匹配”约束是对齐的。
