# Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes

## 0. 本次选择与排重

本次分析对象：**Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes**（Khandelwal et al., ACL 2023 Industry Track）。

- 论文主页：https://aclanthology.org/2023.acl-industry.29/
- PDF：https://aclanthology.org/2023.acl-industry.29.pdf
- 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》
- Spec 核心：100 万–1000 万级商品、持续增量；“同一个商品”严格定义为**同一 reference number / 型号**；字段稀疏；图片可用；**precision 极端优先，宁可漏匹配也不能误匹配**；可人工标注几百对黄金样本。

分析前已排除 `奢侈品调研/a/` 中已有结果：

1. `Ameli.md`
2. `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
3. `DeepBlocker.md`
4. `Tailoring entity resolution for matching product offers.md`
5. `parts-distributor-sku-classifier.md`
6. `pyJedAI.md`

因此本篇没有重复此前 `a` 已提交的分析对象。

---

## 1. 为什么这篇论文对当前 Spec 有直接价值

当前需求表面上是“跨来源商品匹配”，但如果把“同一个商品”严格定义为“同一 reference number”，真正的核心并不是训练一个通用 pairwise matcher，而是：

> **尽可能高精度地从每条商品记录的稀疏文本、结构化字段、图片/OCR 中恢复出可信的 canonical reference，然后用 reference 做严格等值连接。**

这篇论文恰好不是把商品匹配本身当作黑盒分类，而是把**商品属性抽取**做成一个大规模、多模态、可扩展的生成式系统。它解决了几个与 Spec 高度同构的问题：

- 商品属性可能缺失或填错；
- 属性值可能埋在文本里，也可能只能从图片推断；
- 一个模型需要覆盖大量 `(product type, attribute)` 组合，而不是每个属性训练一个模型；
- 生产上不能只看平均 F1，而需要在固定高 precision 下看 recall；
- 训练数据可以来自 catalog 的 distant supervision，而不是完全依赖人工标注；
- 线上持续扩展到大量属性组合，需要模型可维护、可刷新、可监控。

论文最终把模型部署到 6 个英语市场、覆盖超过 10K 个 PT-attribute 组合，并抽取了超过 150M 个属性值。这个量级说明其架构重点不仅是离线 benchmark，而是工程化可扩展性。

但对当前 Spec 不能原样照搬。原因是：**reference number 是身份标识符，不是普通语义属性。生成错一个字符就可能导致错误归并。** 因此本报告的核心结论是：

> **借 MXT 的“多模态属性问答 + 图像条件化 + 多属性单模型”框架做 reference 候选抽取，但把自由生成改造成“受约束候选选择 / 复制 + ABSTAIN”，最终匹配权仍交给 canonical reference 的硬规则。**

---

## 2. 论文方法深拆：MXT 的技术实现

论文把属性抽取写成 Question Answering / answer generation：

- Question：属性名，例如 `color`；
- Text Context：`product type + product textual description`；
- Visual Context：商品图片；
- Answer：目标属性值。

其模型名为 **MXT**，由三段组成：

1. Image-aware Text Encoder；
2. Attribute-aware Text-Image Fusion；
3. T5 Text Decoder。

### 2.1 输入表达：把属性名显式放进 prompt

设商品类型集合为 `P`，属性集合为 `A`。模型 `MXT(P,A)` 可以同时训练多个 product type 和多个 attribute，而不是每个属性一套模型。

输入文本由三部分组成：

```text
attribute-name + product-type + product textual description
```

例如：

```text
question: sleeve type
product type: dress
context: Tahari ASL Women's Sleeveless Ruched Neck Dress ...
```

这一步对我们非常重要，因为 reference 也可以被定义成专门的属性问题：

```text
question: manufacturer reference number
source: 腕表之家
product type: watch
brand: Rolex
context: 劳力士宇宙计型迪通拿系列 m126500ln-0001 ...
ocr: ROLEX OYSTER ... 126500LN ...
```

通过把 `attribute-name`、`product type`、`source`、`brand` 显式放入 prompt，同一模型可以按来源/品牌/品类学习不同的表达习惯。

### 2.2 第一层融合：ResNet-152 + MAG，生成 image-aware text embedding

论文首先使用预训练 **ResNet-152** 把商品图片编码成视觉 embedding `V_R`。

T5 对输入文本得到 token embedding `T_emb^i`。由于原始 T5 只能直接处理文本 embedding，论文没有简单把 image token 拼到文本后，而是使用 **Multimodal Adaptation Gate (MAG)** 对每个文本 token 做视觉条件化位移。

核心逻辑可以概括为：

```text
g_i = ReLU(W_g [T_emb^i ; V_R] + b_g)
H_i = g_i · (W_H V_R) + b_H
T_hat_emb^i = T_emb^i + α H_i
F_MAG^i = Dropout(LayerNorm(T_hat_emb^i))
```

其中 `g_i` 是门控向量，控制视觉信息对当前 token 的影响；`α` 会受文本向量和位移向量范数约束，避免视觉信号无限放大。

这个设计的本质不是“图像和文本简单拼接”，而是：

> **让同一个文本 token 的内部语义表示受当前商品图片影响。**

对腕表 reference 场景，价值主要体现在：

- 标题里出现多个类似编号时，图片上的表盘/表背/OCR 可以帮助判断哪个编号属于当前实物；
- “黑水鬼”“绿水鬼”“熊猫迪”等昵称本身不是 reference，但图片能为系列/外观属性提供辅助；
- 同系列相邻 reference 的文本可能高度相似，视觉可以作为冲突检测信号。

但必须强调：**视觉只能帮助定位证据，不能单独决定 reference 身份。**

### 2.3 第二层融合：Xception + Cross Attention，做 attribute-aware visual selection

论文不是只用一份全局图像 embedding，而是再引入 **Xception** 网络，使用 depthwise separable convolution 学习更细粒度、与属性相关的视觉区域特征。

之后用 multi-head cross attention 让已经过 T5 编码的文本表示 `T_enc` 去注意视觉特征 `V_X`，生成 attribute-aware fused embedding `F_A`。

论文的解释是：不同属性应关注图像不同区域。例如：

- `sleeve type` 关注袖子区域；
- `neck style` 关注领口区域。

迁移到腕表：

- `reference number` 应优先关注表背刻字、保卡、吊牌、标签、包装贴纸；
- `brand` 关注表盘 logo；
- `case material` 关注表壳；
- `dial color` 关注表盘；
- `bracelet/reference` 可能关注表扣或端链刻字。

也就是说，不应该把“整张图视觉相似度”直接当成同款证据，而应该做**属性条件化的区域注意力**。

### 2.4 解码：T5 自回归生成属性值

最终 `F_A` 交给 T5 decoder，自回归生成属性值：

```text
P(y_j | y_<j, x, I)
```

训练目标是标准 token-level negative log likelihood。

论文使用生成式结构的两个主要理由是：

1. **zero-shot attribute value**：属性值没有在训练类别集合里出现，也可以生成；
2. **value-absent inference**：文本中没写，但图片能推断出来，也可以生成。

这对普通颜色、款式、领型等属性很有优势，但对 reference number 是一把双刃剑：

- 优点：可处理训练没见过的新 reference；
- 致命风险：生成模型可能把 `126500LN` 生成成 `126500`、`126500LN-0001`、甚至一个“看起来合理”的未出现编号。

因此在当前 Spec 中，**不能允许 free generation 的输出直接成为实体主键**。

### 2.5 训练、推理与工程规模

论文给出的关键训练配置：

- 文本模型：`t5-base`；
- 图像全局特征：预训练 `ResNet-152`；
- 图像属性感知：Xception；
- 训练 20 epochs；
- batch size = 4；
- 8 × NVIDIA V100 16GB；
- Adam，learning rate `5e-5`；
- warmup ratio `0.1`；
- 选择 validation loss 最好的 checkpoint；
- inference 使用 greedy decoding。

数据方面，30PT 数据集包含：

- train 约 569K products；
- validation 约 84K；
- test 约 73K；
- 30 个 product types；
- 38 个 unique attributes。

生产部署覆盖：

- 6 个英语市场；
- >10K PT-attribute；
- >150M attribute values。

这说明论文的关键工程价值是**单模型覆盖大量属性组合**，避免每个属性单独训练、部署、监控。

---

## 3. 论文结果中最值得当前需求借鉴的指标思想

论文没有只追求整体 F1，而重点比较 **Recall@90P**：即在 precision 固定在 90% 的情况下，看 recall 能做到多高。

在两个 product type 上，MXT 相比 NER-MQMRC 的 Recall@90P 有明显提升；论文也做了三个 ablation：

1. Multi-PT vs Single-PT：多品类联合训练可以共享信息，而且更容易维护；
2. 去掉 Xception：性能下降，说明属性感知区域视觉特征有效；
3. 去掉 MAG、改成简单拼接：性能下降，说明门控式早期融合有效。

对当前 Spec，指标思想应该进一步推到极端：

> 我们不应该优化普通的 F1，也不应允许模型在所有样本上强行出答案；应优化 **Recall@VeryHighPrecision + Abstention Rate**。

例如内部评测建议至少同时报告：

```text
Precision@auto_accept
Recall@auto_accept
Coverage / Acceptance Rate
False Merge Count
Conflict Detection Recall
Unknown / Abstain Rate
```

其中最重要的是 `False Merge Count` 和自动放行集的 precision 下界。

---

## 4. MXT 原方案不能直接用于 reference 匹配的地方

### 4.1 自由生成会产生不可接受的“合理幻觉”

普通属性的近似值可能仍有业务价值，但 reference 是身份键：

```text
126500LN-0001
126500LN-0002
```

哪怕只差 `0001` / `0002`，都可能是不同配置。生成式 decoder 出错一个字符，会把“不确定”变成一个看起来非常确定的错误编号。

**改造：decoder 不允许任意生成。**

输出空间必须受以下候选集合约束：

```text
C = structured_field_candidates
  ∪ regex_candidates_from_title_desc
  ∪ OCR_candidates
  ∪ brand_catalog_candidates
```

模型只允许：

```text
choose one candidate from C
or ABSTAIN
```

如果确实需要生成新 reference，也只能进入 `UNKNOWN_REFERENCE_CANDIDATE` 队列，不能直接用于自动归并。

### 4.2 T5 tokenizer 对字母数字混合 identifier 不友好

论文自己指出 open-domain T5 tokenizer 对电商领域词切分不理想。腕表 reference 更极端：

```text
116610LN
M126500LN-0001
IW371605
03.9300.3620/01.I001
AB0136251B1A1
```

普通 subword tokenizer 可能把这些串切得非常碎，使模型把 reference 当“语言”处理，而不是精确 identifier。

建议：

- 保留原始字符串；
- 额外构造 char/byte-level encoder；
- 对候选 reference 使用字符级 embedding；
- 最终 canonicalization 直接在字符串层完成；
- 模型只负责候选打分，不负责重写 identifier。

### 4.3 视觉语义相似不能覆盖 reference 硬约束

同一系列不同 reference 的外观可以非常接近；相反，同一 reference 因角度、成色、表带、光线不同，图片差异也可能很大。

所以视觉分支不能产生：

```text
image_similarity_high => same_entity
```

只能产生：

```text
image evidence supports candidate reference
image evidence contradicts candidate reference
image evidence unavailable
```

### 4.4 Distant supervision 中的脏标签在 reference 场景更危险

论文生产实践也提到 catalog 中存在大量未归一化、junk attribute values，并通过 heuristic merge 和 tail trimming 清理。

reference 数据同样会有：

- 平台 SKU；
- 店铺自编码；
- 型号+配色码；
- 系列编号；
- 机芯编号；
- 表带编号；
- 配件适配型号；
- OCR 误读。

因此训练前必须做**编号角色分类**和**品牌级 normalization**，不能把所有“像型号的字符串”都当 reference。

---

# 5. 面向当前 Spec 的建议架构：MXT-R（Reference-first Multimodal Extraction & Resolution）

下面给出一个可直接落地的生产方案。核心不是“训练一个更大的匹配模型”，而是把任务拆成：

```text
数据接入
  ↓
字段/文本/图片证据抽取
  ↓
reference 候选生成
  ↓
reference 角色判定 + canonicalization
  ↓
候选 grounding / 冲突检测
  ↓
高精度自动接受 or ABSTAIN
  ↓
用 (brand_id, canonical_reference) 做严格 join
  ↓
簇级一致性审计
```

## 5.1 Layer 0：统一商品事件模型

三来源先落成统一 schema，不要一开始就 pairwise 比较。

建议最小数据结构：

```sql
product_record(
    record_id            bigint,
    source               varchar,
    source_product_id    varchar,
    title_raw            text,
    description_raw      text,
    brand_raw            varchar,
    category_raw         varchar,
    reference_raw        varchar null,
    structured_attrs     jsonb,
    image_urls           jsonb,
    captured_at          timestamp,
    payload_hash         varchar,
    version              bigint
)
```

所有后续抽取结果独立记录 provenance：

```sql
reference_evidence(
    record_id            bigint,
    candidate_raw        varchar,
    candidate_canonical  varchar,
    evidence_type        varchar,  -- structured/title/description/ocr/model/catalog
    evidence_location    varchar,  -- field/span/image_bbox
    evidence_score       float,
    role                 varchar,  -- reference/platform_sku/accessory_ref/serial/unknown
    extractor_version    varchar,
    created_at           timestamp
)
```

不要只存最终 reference；必须保留“为什么得到这个 reference”。这对 precision-first 的误匹配追踪非常关键。

---

## 5.2 Layer 1：廉价、高精度规则先行

模型之前先用 deterministic extraction：

### 结构化字段

如果来源明确提供 manufacturer reference / 型号字段，优先抽取，但仍不能盲信，因为来源可能把 SKU 填到型号字段。

### 文本候选

从 title / description 中使用品牌级规则抽取字母数字混合串：

```python
candidate_patterns = [
    r'\b[A-Z]{1,4}\d{3,}[A-Z0-9\-\./]*\b',
    r'\bM?\d{5,}[A-Z]{0,4}(?:-\d{3,4})?\b',
    ...
]
```

但 regex 只负责**候选召回**，不是最终 reference 判断。

### 图片 OCR

对以下区域优先 OCR：

- 表背；
- 保卡；
- 吊牌；
- 盒侧标签；
- 商品详情参数截图。

OCR 输出同样进入候选集合，不直接匹配。

此阶段的原则是：

> **召回可以高一点，错误候选后面再拒绝；最终自动合并阶段必须极其保守。**

---

## 5.3 Layer 2：MXT 思路改造成“reference candidate selector”

原论文：

```text
(question, product type, text, image) -> free-form attribute value
```

改成：

```text
(question=reference_number,
 source,
 brand,
 category,
 text,
 OCR,
 candidate_set,
 image)
    -> candidate_id | ABSTAIN
```

### 文本侧

输入建议：

```text
[ATTR] reference_number
[SOURCE] 腕表之家
[BRAND] rolex
[CATEGORY] watch
[TITLE] ...
[DESC] ...
[STRUCTURED] 型号=...
[OCR] image_2: 126500LN ...
[CANDIDATES]
C1=126500LN
C2=126500LN-0001
C3=116500LN
```

### 图像侧

保留 MXT 的两个思想：

1. 全局视觉 embedding 通过 gating 改变文本理解；
2. 属性条件 cross-attention 只关注和 reference 相关的图像区域。

工程上不一定需要原样使用 ResNet-152 + Xception。可以替换为现代视觉 encoder，但**保留“全局条件化 + 属性区域注意”这个架构原则**即可。

### 输出侧

不要自回归生成字符串，改成：

```text
softmax([C1, C2, ... Ck, ABSTAIN])
```

或者 pairwise 打分：

```text
score(record, candidate_reference)
```

然后要求：

```text
p_top1 >= tau
and p_top1 - p_top2 >= margin
and no_hard_conflict
```

才把候选晋级为 `trusted_reference`。

这一步把 MXT 的多模态理解能力保留下来，同时去掉最危险的自由生成。

---

## 5.4 Layer 3：Reference Role Classifier

在当前数据里最容易导致灾难性误匹配的并不是“少抽了一个 reference”，而是**把别的编号当成 reference**。

每个候选先分类：

```text
manufacturer_reference
platform_sku
seller_sku
serial_number
movement_caliber
accessory_reference
compatibility_reference
unknown_identifier
```

特征可以包括：

- 候选所在字段名；
- 候选左右文本窗口；
- 来源；
- 品牌；
- 字符串形态；
- 是否出现在品牌型号字典；
- 是否在多个商品中高频重复；
- OCR 所在图像类型；
- 是否伴随“适用/兼容/表带/机芯/序列号”等上下文词。

**只有 role=manufacturer_reference 的候选能进入自动匹配。**

---

## 5.5 Layer 4：Canonicalization 必须品牌感知，而且宁可少归一化

一个全局 `remove_non_alnum()` 会造成非常危险的过度归一化。

建议 canonicalization 分两层：

### Safe global normalization

```text
Unicode NFKC
trim whitespace
uppercase latin letters
normalize full-width chars
normalize obvious separator variants
```

### Brand-specific normalization

例如品牌字典配置：

```yaml
rolex:
  keep_suffix: true
  remove_prefix_m: conditionally
  preserve_variant_code: true

omega:
  preserve_dot_groups: true
  preserve_slash_suffix: true

breitling:
  preserve_all_alnum: true
```

必须保留：

- 能区分配置/版本的 suffix；
- 斜杠后的关键 variant；
- 可能影响实体身份的连字符段。

不要为了“提高 recall”把不同 reference 折叠成同一个 canonical 值。

建议同时保存：

```text
raw_reference
normalized_display_reference
canonical_reference
normalization_rule_version
```

---

## 5.6 Layer 5：Grounding 到品牌 reference dictionary

维护一个 `reference_registry`：

```sql
reference_registry(
    brand_id              bigint,
    canonical_reference   varchar,
    aliases               jsonb,
    family                varchar,
    model_name            varchar,
    known_variants        jsonb,
    evidence_count        int,
    status                varchar, -- trusted/provisional/blocked
    first_seen_at         timestamp,
    last_seen_at          timestamp
)
```

registry 来源可以是：

- 品牌官网/可信 catalog；
- 三源中高可信结构化字段的交集；
- 人工审核确认；
- 历史已稳定聚合的 reference。

### 自动接受策略

- 命中 `trusted registry`：可以继续自动判定；
- 新 reference：先进入 provisional，不能仅凭生成模型创建自动 merge key；
- 与品牌格式冲突：拒绝；
- 同一记录出现多个互斥 trusted reference：直接 ABSTAIN。

---

## 5.7 Layer 6：最终实体匹配不做“模糊相似”，而做 strict key join

当一条商品记录得到 `trusted_reference` 后，实体主键建议直接定义为：

```text
entity_key = (canonical_brand_id, canonical_reference)
```

跨来源匹配：

```sql
SELECT *
FROM source_a a
JOIN source_b b
  ON a.brand_id = b.brand_id
 AND a.trusted_reference = b.trusted_reference;
```

对于 100 万–1000 万记录，这比全量 pairwise matcher 简单得多：

- 无需 `O(N^2)`；
- 可以用 B-Tree / hash index；
- 持续增量时仅对新/变化记录重新抽取 reference；
- 归并复杂度近似线性。

索引：

```sql
CREATE INDEX idx_product_ref
ON normalized_product(brand_id, trusted_reference)
WHERE trusted_reference IS NOT NULL;
```

如果同一个 reference 对应多个来源记录，形成的是**同 reference 商品簇**，而不是依赖两两相似度边传播。

---

# 6. Precision-first 的硬闸门设计

Spec 写的是“绝对不能误匹配”，工程上无法用有限测试样本数学证明真正的绝对 0 错误，所以要把系统设计成：

> **模型只能提供候选证据；自动归并必须同时满足一组可解释的硬条件。**

建议自动接受条件：

```text
AUTO_ACCEPT(record, ref) iff
  brand_confident
  AND role(ref) == manufacturer_reference
  AND canonical_ref_is_valid
  AND evidence_has_independent_support
  AND no_reference_conflict
  AND no_accessory_context_conflict
  AND model_confidence >= τ_brand
  AND top1_margin >= δ_brand
  AND ref_registry_status == trusted
```

### 独立证据等级建议

**S 级**

- 来源结构化 manufacturer reference 字段；
- 品牌官方/可信 registry；
- 保卡/吊牌 OCR 清晰命中。

**A 级**

- 标题明确“型号/Ref/reference”上下文后的编号；
- 两个不同字段独立出现同编号。

**B 级**

- 普通文本 regex 候选；
- 图片 OCR 但无明确区域分类。

**C 级**

- 视觉/语义模型推断；
- 文本近似相似；
- 同系列外观相似。

自动匹配可以要求：

```text
(S)
or (A + A)
or (S + no conflict)
```

而 C 级**永远不能单独触发自动归并**。

---

# 7. 冲突优先于相似：必须有 Hard Negative / Veto 机制

普通实体匹配常把各种相似度加权求和：

```text
0.4 text + 0.3 image + 0.3 attributes
```

对当前 Spec 不合适。因为强负证据不应该被大量弱正证据抵消。

例如：

```text
标题很像 + 图片很像 + 同系列
BUT
reference = 126500LN-0001 vs 126500LN-0002
```

最终必须是：

```text
NOT_MATCH
```

所以建议采用 **Veto-first**：

```python
def decision(record, candidate_ref):
    if brand_conflict(record, candidate_ref):
        return REJECT
    if trusted_reference_conflict(record, candidate_ref):
        return REJECT
    if identifier_role_conflict(record, candidate_ref):
        return REJECT
    if accessory_context(record, candidate_ref):
        return REJECT
    if insufficient_evidence(record, candidate_ref):
        return ABSTAIN
    if calibrated_score(record, candidate_ref) < threshold:
        return ABSTAIN
    return ACCEPT
```

这比“综合分数超过 0.92 就同款”更契合 precision-first。

---

# 8. 图片应该怎么用：从“视觉相似”转向“reference 证据定位”

结合 MXT 的 attribute-aware image attention，建议图片管线分成：

```text
image
  ├─ image type classifier
  │    ├─ dial/front
  │    ├─ caseback
  │    ├─ warranty card
  │    ├─ hang tag
  │    ├─ box label
  │    └─ other
  │
  ├─ OCR / text detector
  │
  └─ visual encoder
        └─ attribute-conditioned attention
```

对 reference 来说优先级应是：

1. 保卡/吊牌/盒标上的 OCR；
2. 表背/表扣等刻字 OCR；
3. 视觉属性帮助判品牌/系列；
4. 全局视觉相似只做候选召回或冲突辅助。

### OCR 结果必须保留 bbox

```json
{
  "image_id": "...",
  "text": "126500LN",
  "bbox": [x1, y1, x2, y2],
  "ocr_confidence": 0.98,
  "image_type": "warranty_card"
}
```

这样后续模型可以学习“编号出现在保卡 reference 区域”与“编号出现在配件说明文字里”的差别。

---

# 9. 训练数据：几百个人工黄金样本怎么用最划算

用户可接受人工标注几百对。不要平均随机标，而应优先做 hard cases。

## 9.1 黄金集组成建议

例如 600 对：

```text
150 对：明确同 reference 正例
250 对：同品牌同系列、reference 很接近但不同的 hard negatives
100 对：配件/表带/盒证中出现兼容 reference 的负例
50 对：平台 SKU / seller SKU 冒充 reference
50 对：OCR 易混字符（0/O, 1/I, 5/S, 8/B）
```

为什么负例更多：当前业务损失函数明显是 asymmetric，false positive 的代价远大于 false negative。

## 9.2 标注对象不是只标 `match / non-match`

建议同时标：

```text
brand
true_reference
candidate_reference_role
reference_evidence_source
is_reference_present_in_text
is_reference_present_in_image
is_accessory
final_match_label
```

这样几百条数据可以同时用于：

- role classifier；
- candidate selector；
- 阈值校准；
- 错误分析；
- OCR confusion 修正。

## 9.3 Hard negative mining

线上最危险的负例不是随机两个商品，而是：

```text
same brand
same family
almost same title
almost same image
reference edit distance very small
```

例如按 canonical reference 的字符邻近关系构建难负例：

```text
Levenshtein(ref_a, ref_b) <= 2
AND brand_a == brand_b
AND ref_a != ref_b
```

模型训练和验收都必须重点覆盖这些样本。

---

# 10. 阈值与校准：不要一个全局 0.9

不同品牌、来源、字段质量差异很大：

- 腕表之家标题格式可能稳定；
- 某来源 reference 结构化字段可能较可靠；
- 另一个来源可能大量把 SKU 写进型号；
- Rolex 与 Omega 的 reference 语法完全不同。

所以阈值最好至少做到：

```text
τ(source, brand, evidence_pattern)
```

例如：

```yaml
腕表之家:
  rolex:
    structured+registry: 0.995
    title+registry: 0.999
    ocr_only: reject_auto

奢当家:
  omega:
    structured+title: 0.999
    model_only: reject_auto
```

关键不是概率值本身，而是**自动放行集的真实错误率**。

建议按每种 gate 输出 calibration report：

```text
accepted_count
reviewed_count
false_positive_count
precision_lower_bound
coverage
```

如果某个 gate 出现任何系统性误合并，先降低 coverage / 关闭 gate，而不是用其他特征补偿分数。

---

# 11. 增量架构：100 万–1000 万数据不需要每天全量重跑

对每条 record 计算：

```text
content_hash = hash(
    title,
    description,
    brand,
    structured_reference_fields,
    image_hashes
)
```

只有以下情况重新跑 extraction：

- 新商品；
- content_hash 变化；
- extractor_version 更新；
- normalization_rule_version 更新；
- reference_registry 对该品牌发生变化。

推荐事件流程：

```text
Crawler / DB Sync
    ↓
Kafka/Pulsar topic: product-upsert
    ↓
Normalizer Worker
    ↓
Rule Candidate Extractor
    ↓
OCR Worker (only when needed)
    ↓
MXT-R Candidate Selector
    ↓
Reference Gate
    ↓
Entity Index Upsert
```

批量历史回填可以使用 Spark/Flink/普通分片 worker；在线增量只需要消费变更事件。

如果当前团队不想引入 Kafka，一期也可以简单使用：

```text
PostgreSQL + task table + Python workers
```

等吞吐和实时性真的需要时再换消息队列。架构重点是幂等和版本化，而不是必须某个中间件。

---

# 12. 推荐的数据表与可解释审计链

最终实体结果：

```sql
resolved_product(
    record_id               bigint primary key,
    canonical_brand_id      bigint,
    trusted_reference       varchar,
    entity_key              varchar,
    decision                varchar, -- accept/abstain/reject
    decision_reason         jsonb,
    confidence              float,
    extractor_version       varchar,
    normalization_version   varchar,
    resolved_at             timestamp
)
```

实体簇：

```sql
reference_entity(
    entity_key              varchar primary key,
    canonical_brand_id      bigint,
    canonical_reference     varchar,
    source_count            int,
    member_count            int,
    conflict_state          varchar,
    updated_at              timestamp
)
```

一定要能回答：

```text
为什么这两条被判成同一个实体？
```

理想答案不是：

```text
模型相似度 0.982
```

而是：

```text
品牌均 canonicalized 为 Rolex；
A 的结构化型号字段 = M126500LN-0001；
B 标题明确型号 = 126500LN-0001；
两者经 Rolex rule v7 归一化为同一 canonical reference；
B 保卡 OCR 同时命中 126500LN；
无其他 trusted reference 冲突；
reference registry 状态=trusted；
因此按 strict reference key 合并。
```

这才是可审计的 precision-first 系统。

---

# 13. 直接可落地的 MVP

## Phase 1：不用训练模型，先建立 80% 的正确骨架

先做：

1. 三来源统一 schema；
2. brand normalization；
3. structured reference extraction；
4. regex candidate extraction；
5. identifier role rules；
6. brand-specific canonicalization；
7. `(brand_id, canonical_reference)` strict join；
8. conflict -> abstain；
9. 全量记录 provenance。

这一步很可能已经能覆盖一大批高置信样本，而且风险最低。

### MVP 自动匹配规则示例

```python
if structured_ref_is_valid \
   and registry_contains(brand, canonical_ref) \
   and not any_conflicting_ref(record):
    accept(canonical_ref)
elif explicit_title_ref \
     and registry_contains(brand, canonical_ref) \
     and second_independent_evidence(record, canonical_ref):
    accept(canonical_ref)
else:
    abstain()
```

---

## Phase 2：加入 OCR

只对以下记录跑 OCR：

```text
no trusted structured ref
OR text contains multiple candidate refs
OR title/description insufficient
```

不要对所有 1000 万商品的所有图片无差别 OCR，成本没有必要。

优先识别：

```text
warranty_card / hang_tag / label / caseback
```

得到独立证据后回到同一个 gate。

---

## Phase 3：训练 MXT-R selector，而不是生成器

训练样本单位：

```text
(record, candidate_reference, label)
```

输入：文本、候选 reference、OCR、图片 embedding、来源、品牌。

输出：

```text
belongs_to_current_product_reference: yes/no
```

或者候选集级：

```text
C1/C2/C3/ABSTAIN
```

训练时过采样同系列 near-reference hard negatives。

### 推荐 loss

除了 cross entropy，可增加 margin objective：

```text
L = CE(top_candidate)
  + λ * max(0, m - s_pos + s_hardneg)
```

使真正 reference 对相邻 hard negative 拉开 margin。

---

## Phase 4：人工复核闭环

人工队列不要随机排序，优先：

```text
high business value
× high model uncertainty
× high conflict severity
```

人工确认后回流：

```text
reference_registry
hard_negative_pool
role_classifier_training
brand_normalization_rules
threshold_calibration
```

即让人工主要用于提升未来自动化覆盖，而不是无限做人工匹配。

---

# 14. 代码级模块拆分建议

```text
reference_resolution/
├── ingest/
│   ├── schemas.py
│   └── source_adapters/
├── brand/
│   ├── normalizer.py
│   └── brand_registry.yaml
├── candidate/
│   ├── structured.py
│   ├── regex.py
│   ├── ocr.py
│   └── catalog.py
├── role/
│   ├── features.py
│   └── classifier.py
├── normalize/
│   ├── base.py
│   ├── rolex.py
│   ├── omega.py
│   └── breitling.py
├── multimodal/
│   ├── text_encoder.py
│   ├── vision_encoder.py
│   ├── fusion.py
│   └── candidate_selector.py
├── gate/
│   ├── veto_rules.py
│   ├── thresholds.py
│   └── resolver.py
├── registry/
│   └── reference_registry.py
├── entity/
│   ├── key.py
│   └── cluster_audit.py
├── eval/
│   ├── hard_negatives.py
│   ├── calibration.py
│   └── metrics.py
└── jobs/
    ├── backfill.py
    └── incremental.py
```

核心接口可以固定为：

```python
@dataclass
class ReferenceCandidate:
    raw: str
    canonical: str | None
    role: str
    source: str
    location: str
    confidence: float

@dataclass
class ResolutionResult:
    decision: Literal["ACCEPT", "ABSTAIN", "REJECT"]
    canonical_reference: str | None
    reasons: list[str]
    evidence: list[ReferenceCandidate]
    model_score: float | None
    version: str
```

最终 `resolve()` 必须允许返回 `ABSTAIN`，这是整个系统的第一等公民，而不是异常情况。

---

# 15. 推理伪代码

```python
def resolve_reference(record):
    brand = normalize_brand(record.brand_raw, record.title_raw)
    if not brand.confident:
        return abstain("brand_not_confident")

    candidates = []
    candidates += extract_structured_candidates(record)
    candidates += extract_text_candidates(record)

    if need_ocr(record, candidates):
        candidates += extract_ocr_candidates(record.images)

    candidates = deduplicate_candidates(candidates)
    candidates = classify_identifier_roles(candidates, record, brand)
    candidates = [c for c in candidates if c.role == "manufacturer_reference"]

    if not candidates:
        return abstain("no_reference_candidate")

    for c in candidates:
        c.canonical = canonicalize_reference(brand.id, c.raw)
        c.registry = lookup_reference_registry(brand.id, c.canonical)

    if trusted_conflict(candidates):
        return reject_or_abstain("conflicting_trusted_references")

    strong = deterministic_high_precision_candidate(candidates)
    if strong and gate_passes(record, strong):
        return accept(strong, "deterministic_evidence")

    scored = mxt_r_rank(record, candidates)
    best, second = top2(scored)

    if best.label == "ABSTAIN":
        return abstain("model_abstain")

    if not best.registry.trusted:
        return abstain("reference_not_grounded")

    if best.score < threshold(record.source, brand.id, best.evidence_pattern):
        return abstain("below_calibrated_threshold")

    if best.score - second.score < margin_threshold(brand.id):
        return abstain("insufficient_margin")

    if hard_veto(record, best):
        return reject_or_abstain("hard_conflict")

    return accept(best, "multimodal_candidate_selection")
```

跨来源实体构建：

```python
def entity_key(resolution, brand_id):
    if resolution.decision != "ACCEPT":
        return None
    return f"{brand_id}:{resolution.canonical_reference}"
```

不需要让 pairwise classifier 再覆盖这个结论。

---

# 16. 监控与回滚

生产里每次更新：

- OCR 模型；
- reference role classifier；
- normalization rule；
- reference registry；
- threshold；
- multimodal selector；

都可能改变实体归并结果。

因此必须版本化：

```text
extractor_version
ocr_version
role_model_version
normalization_version
registry_version
gate_version
```

并记录：

```text
old_entity_key -> new_entity_key
```

任何新版本上线前比较：

```text
new_accepts
lost_accepts
changed_reference
new_conflicts
cluster_merges
cluster_splits
```

尤其对 `cluster merge` 要做高优先级人工抽查，因为一次错误规则可能把大量商品合到同一 reference。

---

# 17. 评测集必须按“最容易误合并”的方式构造

不要只做随机 train/test split。

建议测试桶：

### Bucket A：同 brand，不同 family

相对简单。

### Bucket B：同 brand，同 family，不同 reference

关键 hard negative。

### Bucket C：reference 只差 1–2 个字符

最危险。

### Bucket D：标题出现配件兼容 reference

例如表带、盒证、配件描述包含主表型号。

### Bucket E：reference 只在图片/保卡中

测 OCR + multi-modal。

### Bucket F：多 reference 冲突

应该 ABSTAIN，而不是强行挑一个。

### Bucket G：全新 reference

测系统能否识别 unknown，而不是 hallucinate 成已有型号。

验收标准应按桶分别给 precision，不能被简单样本的高分平均掉。

---

# 18. 与 MXT 论文的“继承 / 修改 / 舍弃”清单

| MXT 设计 | 当前方案 | 原因 |
|---|---|---|
| 属性名作为 question | **继承** | `reference_number` 可作为显式任务条件 |
| product type + text context | **继承并扩展** | 加入 source / brand / structured attrs / OCR |
| ResNet 全局视觉条件化 | **继承思想** | 图片可帮助理解文本歧义 |
| MAG 门控融合 | **继承思想** | 比简单拼接更适合“视觉只修正语义” |
| Xception 属性感知区域 | **继承思想** | reference 应关注保卡、表背、标签等区域 |
| T5 自由生成 attribute value | **舍弃** | reference 不能允许 hallucination |
| greedy decoding | **舍弃** | 改 candidate selection + ABSTAIN |
| 单模型覆盖多 PT-attributes | **继承** | 可覆盖多品牌、多来源、多字段 |
| distant supervision | **部分继承** | catalog 可做弱标签，但必须清洗 SKU/脏值 |
| Recall@90P | **继承并提高标准** | 改成 ultra-high precision + coverage + false merge |
| value-absent image inference | **谨慎继承** | OCR/视觉可补文本缺失，但不能直接生成身份键 |

---

# 19. 最值得直接落地的一个方案

如果只选一条最实用路线，我建议：

> **先把整个系统建设成“Reference Extraction & Grounding Pipeline”，而不是“Product Matching Model”。**

具体定义：

```text
同款 = canonical_brand 相同
    AND canonical_reference 相同
    AND 两边 reference 均通过 high-precision trusted gate
```

其中 MXT 的角色是：

```text
多模态 evidence understanding
        ↓
reference candidate ranking / validation
```

而不是：

```text
直接判断两个商品是不是同款
```

这样能天然满足当前 Spec 的业务约束：

- **高 precision**：最终由 reference 硬相等收口；
- **字段稀疏**：文本、结构化字段、OCR、视觉可互补；
- **百万/千万级**：抽取后 hash/index join，不做全量 pairwise；
- **持续增量**：按 content hash/version 只重算变化记录；
- **可人工标注几百对**：重点用于 hard-negative、role、threshold；
- **可解释**：每一次合并都能追到具体 reference 证据；
- **可拒识**：不确定样本从设计上就允许不匹配。

---

# 20. 推荐实施顺序

## P0：建立黄金规则基线

- Brand canonicalization
- Structured reference extractor
- Text regex candidate extractor
- Identifier role rules
- Brand-specific reference canonicalizer
- Strict `(brand, reference)` join
- Conflict -> ABSTAIN

先量化：

```text
自动覆盖率是多少？
人工抽查是否出现误合并？
主要漏匹配原因是什么？
```

## P1：加入 OCR 证据

优先处理“文本没有 reference，但图片可能有保卡/标签”的商品。

## P2：加入 MXT-R candidate selector

只处理规则无法决定的长尾样本，不替代确定性规则。

## P3：建立 few-shot calibration + hard negative 评测

用几百黄金标签校准 gate，尤其测试 near-reference。

## P4：增量化 + 版本回溯

把历史回填和线上变更解耦，实现 extractor/version 可回滚。

---

# 21. 最终结论

这篇 MXT 论文最重要的启发，不是“用 T5 + ResNet + Xception 就能解决商品匹配”，而是三点：

1. **把稀疏商品数据中的关键信息恢复问题显式建模成属性任务**，而不是直接做黑盒 pairwise matching；
2. **图像应该在属性条件下参与文本理解和区域选择**，而不是简单使用全局图片相似度；
3. **生产模型必须能覆盖大量属性组合、支持高 precision 阈值、处理弱监督脏数据并可持续刷新。**

针对当前“同一 reference 才算同一商品”的 Spec，最合理的落地不是复制论文的自由生成，而是做以下关键改造：

```text
MXT free-form attribute generation
        ↓
constrained multimodal reference candidate selection
        ↓
role classification
        ↓
brand-aware canonicalization
        ↓
trusted registry grounding
        ↓
hard veto + abstention
        ↓
strict (brand, canonical_reference) join
```

这个方案能把多模态模型的能力放在它真正擅长的地方——**从脏、稀疏、多源证据中找出候选**；同时把最终实体归并放回到确定性、可审计、可拒识的 reference 规则上。这比直接训练一个“同款/不同款”分类器更符合“绝对不能误匹配、允许漏匹配”的业务约束，也更适合 100 万–1000 万级持续增量数据。
