# parts-distributor-sku-classifier：从“编号角色分类”到跨源二奢 reference 确定性实体解析

> 分析人：c  
> 原项目：https://github.com/pcbng/parts-distributor-sku-classifier  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 本文结论优先级：**precision-first / 宁可拒识，不可误合并**

## 1. 为什么选这个项目

当前需求表面上是“跨来源商品实体匹配”，但 Spec 已经把“同一个商品”定义得非常强：**同一 reference number / 型号**。因此真正困难的部分不是训练一个 pairwise matcher 判断两条商品像不像，而是先回答：

1. 一条商品记录里出现的字母数字串，究竟是不是品牌 reference；
2. 如果不是，它是平台 SKU、商家库存号、序列号，还是标题中提到的“兼容/适配其他型号”；
3. 如果有多个疑似 reference，哪一个才属于当前商品本体；
4. 只有把上述问题高精度解决，跨源匹配才能安全地退化成确定性的 reference join。

`parts-distributor-sku-classifier` 恰好研究了同构问题：电子元器件中，制造商的 MPN（Manufacturer Part Number）和分销商自己的 SKU 都长得像“型号”，但语义完全不同。项目用字符级 LSTM 将字符串分为：

- Manufacturer Part Number（真正的制造商型号）；
- Mouser SKU（平台编号）；
- Digi-Key SKU（平台编号）。

这对二奢/腕表很有直接启发：**不要看到“像型号”的字符串就拿来做 exact match，先做 identifier role classification。**

---

## 2. 原项目技术实现拆解

### 2.1 数据流

项目的数据和模型文件非常简单：

```text
mpn_mouser_digikey.csv
        │
        ├─ partnum: 原始编号字符串
        └─ class: 0=MPN / 1=Mouser SKU / 2=Digi-Key SKU
        │
        ▼
字符词典 char_dictionary.json
        │
        ▼
字符 ID 序列 + post padding
        │
        ▼
Embedding(32)
        │
        ▼
LSTM(32, dropout=0.2, recurrent_dropout=0.2)
        │
        ▼
Dense(3, softmax)
        │
        ▼
类别概率
```

训练完成后分别保存：

- `trained_model_layers.json`：模型结构；
- `trained_model_weights.h5`：权重；
- `char_dictionary.json`：字符映射；
- `cleaned_training_data.json`：清洗后的训练数据。

从仓库里的模型 JSON 可以直接看到：`input_dim=52`、Embedding 维度 32、LSTM hidden size 32、输出 3 类 softmax。原始字符编号从 1 开始，因此 0 被自然留作 padding；按模型 input_dim 推断，训练词表覆盖约 51 个非 padding 字符。

### 2.2 字符级编码为什么适合“编号角色”

项目没有把 `SN74LVC541APWR`、`296-8521-1-ND` 当自然语言，而是直接按字符建模：

```python
unique_chars = set()
for s in df['partnum'].values:
    unique_chars |= set(c for c in s)

partnum_dict = {c: i+1 for i, c in enumerate(unique_chars)}
df['x'] = df['partnum'].map(lambda s: [partnum_dict[c] for c in s])
```

然后使用全数据中的最大字符串长度做 `post padding`。

这种表示非常适合 SKU/reference：区分它们的往往不是词义，而是：

- 固定前缀/后缀；
- 字母和数字交替方式；
- `-`、`/`、`.` 等分隔符位置；
- 长度分布；
- 某来源特有的编号模板。

也就是说，模型学习的是 **identifier morphology（编号形态）**。

### 2.3 训练方式

项目随机按行切 80% train / 20% validation：

```python
np.random.seed(20181203)
df['dataset'] = np.random.choice(
    ['train', 'val'],
    size=len(df),
    replace=True,
    p=[0.80, 0.20]
)
```

模型：

```python
model = Sequential()
model.add(Embedding(len(partnum_dict)+1, 32))
model.add(LSTM(32, dropout=0.2, recurrent_dropout=0.2))
model.add(Dense(3, activation='softmax'))

model.compile(
    loss='categorical_crossentropy',
    optimizer='adam',
    metrics=['accuracy']
)
```

训练配置：

- batch size = 32；
- epoch = 7；
- 训练样本 11,344；
- 验证样本 2,789；
- 总样本约 14,133。

Part 1 最后一个 epoch 的 validation accuracy 约 98.49%。Part 2 重新载入模型后报告 overall accuracy 99.01%，按类统计为：

- MPN：97.49%；
- Mouser SKU：99.51%；
- Digi-Key SKU：100%。

注意 Part 2 在“重新载入并 evaluate”时把 loss 写成了 `binary_crossentropy`，与 3 类 softmax 的训练 loss 不一致；这不会改变已经保存的权重和前向预测，但如果继续训练则不应照搬，生产实现应统一为多类交叉熵或明确设计的 selective/risk loss。

---

## 3. 原项目真正可迁移的架构思想

### 3.1 最重要：先判断“这个编号是什么”，再判断“两个商品是不是同一个”

如果直接做：

```text
record A title 中抽到 ABC123
record B title 中抽到 ABC123
=> match
```

很容易出现以下灾难性误合并：

- 两个平台都把同一种格式的自有 SKU 放进标题；
- 标题写“适配 Rolex XXX / for XXX”，抽到的是被适配手表的型号；
- 表带、表盒、保卡等配件标题中带有腕表 reference；
- 序列号、库存号、鉴定号被误判成 reference；
- 一个商品描述里出现多个 reference，只取了第一个；
- OCR 把表背/保卡上的其他编号误当 reference。

原项目提示我们应在 reference extraction 与 entity resolution 之间增加一层：

```text
Identifier Role Classification
```

即先把每个候选字符串分类成“它扮演的角色”。

### 3.2 但不能直接照搬 LSTM 结果

原模型 accuracy 很高，但对当前 Spec 仍远远不够，原因如下。

#### A. MPN 类 97.49% 不符合 precision-first

当前系统中 BRAND_REFERENCE 相当于正类。即使只有 1% 左右误判，在百万级数据上也可能产生大量错误实体键。

#### B. 原模型只有 closed-set 三分类，没有 UNKNOWN / ABSTAIN

真实二奢数据里会遇到训练中从未见过的编号类型。生产系统必须允许：

```text
UNKNOWN
NEEDS_REVIEW
CONFLICT
```

而不是强迫每个字符串三选一。

#### C. 字符串形态只能判断“像什么”，不能判断“属于谁”

例如一个标题出现：

```text
原装表带 适配 XXX123 / XXX124
```

`XXX123` 在形态上可能 100% 像品牌 reference，但它并不是当前商品（表带）的 reference。必须把**上下文 ownership**纳入判断。

#### D. 原字符表没有 OOV 设计

原代码通过 `partnum_dict[c]` 直接查字符。线上遇到训练词表之外的新字符会失败。生产模型必须显式保留：

```text
PAD=0
UNK=1
```

并在清洗层统一 Unicode、全角字符和异常标点。

#### E. 随机行切分容易高估泛化

SKU/型号往往有很强的家族模板。如果同一前缀/系列的近邻样本同时进入 train 和 validation，模型很容易“记格式”。更可靠的评测应按以下维度做 group holdout：

- 品牌；
- reference family / 前缀族；
- 来源；
- 时间批次；
- 新增平台。

这比随机 80/20 更接近持续增量场景。

---

# 4. 针对当前 Spec 的核心改造：不要做全量 pairwise matcher

Spec 已定义“同商品 = 同 reference”。因此推荐把问题重写为：

```text
跨源 Entity Matching
        ↓
每条 record 的 canonical reference 高精度解析
        ↓
生成确定性 entity_key
```

最终实体键：

```text
entity_key = hash(canonical_brand_id + "\x1f" + canonical_reference)
```

之所以必须加入 `brand_id`，是为了避免不同品牌出现相同/相近编号时发生跨品牌碰撞。

最终匹配逻辑甚至不需要 ML：

```python
MATCH(a, b) = (
    a.entity_key is not None
    and b.entity_key is not None
    and a.entity_key == b.entity_key
)
```

**ML 只负责帮助把 record 安全地解析为 entity_key；ML 不直接拥有“合并实体”的权限。**

这会把 100 万–1000 万级数据的 O(N²) 配对问题降为 O(N) 的解析 + O(1)/O(logN) 的 key lookup/upsert。

---

# 5. 可直接落地的生产架构

## 5.1 总体链路

```text
雷小安 ─┐
腕表之家 ├─> Raw Product Ingest
奢当家 ─┘
              │
              ▼
     Source Field Normalizer
              │
              ▼
      Identifier Candidate Miner
   ┌──────────┼───────────┐
   │          │           │
结构化字段    标题/描述     图片 OCR
   │          │           │
   └──────────┼───────────┘
              ▼
 Identifier Role Classifier
              │
              ▼
 Context Ownership Verifier
              │
              ▼
 Conservative Canonicalizer
              │
              ▼
 Brand/Reference Verifier
              │
              ▼
   Conflict / Negative Evidence Gate
        ┌─────┴─────┐
        │           │
     VERIFIED     ABSTAIN
        │           │
        ▼           ▼
 entity_key      Review Queue
        │
        ▼
 Deterministic Cross-source Join
```

## 5.2 第一层：Source Field Normalizer

不要一上来把三个平台所有字段 flatten 成一坨文本。首先建立来源字段语义配置：

```yaml
source: lei_xiao_an
fields:
  title: ...
  brand: ...
  suspected_reference: ...
  source_sku: ...
  description: ...
  images: ...
```

对每个来源明确哪些字段是：

- 平台商品 ID；
- 平台 SKU；
- 商家库存号；
- 品牌；
- 声称的 reference/model；
- 标题/描述；
- 图片。

如果某来源确实有语义稳定的“型号”结构化字段，这个字段的证据权重应远高于从自由文本猜出的字符串。

## 5.3 第二层：Identifier Candidate Miner

从每条记录中抽取所有疑似 identifier，不要只抽一个。

候选来源：

1. 结构化 reference/model 字段；
2. 标题；
3. 描述；
4. 属性键值；
5. 图片 OCR。

每个候选都要带完整 provenance：

```json
{
  "record_id": "...",
  "raw_token": "ABC-123",
  "origin": "title",
  "field": "title",
  "span": [12, 19],
  "left_context": "型号",
  "right_context": "全套",
  "image_id": null
}
```

**不要丢掉上下文。** 后面判断“这个 reference 是否属于当前商品”时必须使用。

---

# 6. Identifier Role Classifier：原项目的直接升级版

## 6.1 建议标签体系

相比原项目的三类，目标系统至少需要：

```text
BRAND_REFERENCE_SELF       当前商品自身的品牌 reference
PLATFORM_SKU               平台 SKU/货号
SELLER_INVENTORY_ID        商家库存号
SERIAL_NUMBER              序列号/唯一件号
COMPATIBLE_REFERENCE       “适配/兼容”的其他商品 reference
ACCESSORY_TARGET_REFERENCE 配件所对应主商品的 reference
OTHER_IDENTIFIER           其他编号
UNKNOWN                    无法判断
```

这里最重要的区分是：

```text
“它像品牌 reference”
≠
“它就是当前商品自己的 reference”
```

## 6.2 模型输入

保留原项目最有价值的字符形态分支，但增加上下文：

```text
token chars ─> Char Encoder ─────┐
                                 │
left/right context ─> Text Encoder ─┤
                                 ├─> Role logits
source_id ─> Embedding ──────────┤
field_type ─> Embedding ─────────┤
brand_id ─> Embedding ───────────┘
```

### 一个足够实用的轻量实现

字符侧没必要上大模型：

```text
Embedding(32)
 -> 1D-CNN(64, kernel 3/5) 或 BiGRU(64)
 -> token_shape_embedding
```

上下文可以先用：

- character/word n-gram + LightGBM；或
- 小型中文/多语 encoder；
- 甚至 source-specific 规则特征。

第一版最推荐 **规则 + char model + context classifier 的 ensemble**，原因是可解释、便于设置 veto。

## 6.3 规则优先于模型

例如某来源平台 SKU 有稳定格式时：

```python
if source == S and PLATFORM_SKU_REGEX.fullmatch(token):
    return PLATFORM_SKU, confidence=1.0, reason="source_rule"
```

这种确定性负规则不应该交给神经网络“重新判断”。

同理，如果上下文明确是：

```text
适配 / 兼容 / for / replacement / 表带适用
```

则候选即便像 reference，也应打上强负证据，禁止直接产生实体键。

---

# 7. Reference Canonicalization：必须“保守”，不能暴力清洗

常见错误实现是：

```python
normalize = re.sub(r'[^A-Z0-9]', '', token.upper())
```

这对 precision-first 很危险，因为有些品牌/系列的分隔符、后缀或子型号可能具有语义。

推荐分两级。

## 7.1 全局安全规范化

只做几乎不会改变型号语义的操作：

- Unicode NFKC；
- ASCII 大小写统一；
- trim；
- 全角字母数字转半角；
- 连续空白折叠；
- 明确等价的 Unicode dash 映射为标准 `-`。

## 7.2 brand-specific canonicalizer

每个品牌单独维护：

```text
allowed_pattern
separator_policy
known_alias_rules
meaningful_suffix_policy
```

**没有经过品牌规则确认的情况下，不要全局删除 `- / .`，也不要自动去掉后缀。**

规范化输出应该同时保留：

```json
{
  "raw": "...",
  "normalized_safe": "...",
  "canonical": "...",
  "canonicalizer_version": "brand-x-v7"
}
```

这样后续规则升级可以追溯和重算。

---

# 8. Reference Verifier：把“可能是”升级成“允许落实体键”

一个候选只有通过 verifier 才能从 `CANDIDATE` 变成 `VERIFIED_REFERENCE`。

推荐证据等级：

## Level A：强证据

- 来源已验证过语义的结构化 model/reference 字段；
- canonical reference 命中可信品牌 reference catalog；
- 两个独立字段给出相同 reference 且无冲突。

## Level B：组合证据

- 标题中抽取；
- role classifier 极高置信；
- 上下文明确表示“型号/Ref/model”；
- brand-specific grammar 校验通过；
- 无 SKU/serial/accessory 冲突。

## Level C：辅助证据

- 图片 OCR；
- 视觉相似；
- embedding 相似度。

**Level C 只能 corroborate，不能单独触发自动实体合并。**

尤其是图片：它最适合做“冲突否决”或人工复核辅助，而不是绕过 reference 规则直接认同款。

---

# 9. Precision-first 的最终决策门

建议把系统输出设计为三态，而不是二态：

```text
VERIFIED
REJECTED
ABSTAIN
```

对于一条 record，只有同时满足以下条件时才生成 `entity_key`：

```python
brand_is_verified
AND exactly_one_canonical_reference
AND reference_role == BRAND_REFERENCE_SELF
AND reference_evidence >= required_level
AND no_conflicting_verified_reference
AND no_platform_sku_conflict
AND no_serial_conflict
AND no_accessory_or_compatibility_conflict
```

伪代码：

```python
def resolve_entity_key(record):
    brand = resolve_brand(record)
    if not brand.verified:
        return Abstain("brand_not_verified")

    candidates = mine_identifiers(record)
    evidences = []

    for c in candidates:
        role = classify_identifier_role(c, record)
        norm = canonicalize(brand.id, c.raw_token)
        verification = verify_reference(
            brand=brand,
            candidate=c,
            role=role,
            canonical=norm,
            record=record,
        )
        evidences.append(verification)

    verified_refs = unique(
        e.canonical_reference
        for e in evidences
        if e.status == "VERIFIED_REFERENCE"
    )

    if len(verified_refs) == 0:
        return Abstain("reference_missing")

    if len(verified_refs) > 1:
        return Abstain("reference_conflict")

    if has_strong_negative_evidence(evidences):
        return Abstain("negative_evidence")

    ref = verified_refs[0]
    return Verified(
        entity_key=sha256(f"{brand.id}\x1f{ref}"),
        canonical_reference=ref,
    )
```

跨源匹配随后变成：

```sql
SELECT *
FROM product_records
WHERE entity_key = :entity_key;
```

不需要让模型在两个商品之间再做一次“像不像”的主观判断。

---

# 10. 冲突优先：任何正证据都不能覆盖强负证据

这是当前需求与普通 Entity Matching 最大的不同。

传统 matcher 往往：

```text
title 很像 + image 很像 + brand 一样
=> score 0.98
=> match
```

这里应该反过来：

```text
只要发现两个不同的 VERIFIED reference
=> 无条件不自动合并
```

建议维护 veto 规则：

| 情况 | 自动决策 |
|---|---|
| 两条记录 canonical reference 完全相同且各自已 VERIFIED | 可合并 |
| 同品牌，但 reference 不同 | 不合并 |
| 一边有 VERIFIED reference，另一边只有视觉相似 | ABSTAIN |
| 标题 ref 与 OCR ref 冲突 | ABSTAIN |
| 同记录出现两个高可信不同 ref | ABSTAIN |
| reference 只出现在“适配/兼容”上下文 | 不作为实体键 |
| 只有图片 embedding 相似 | 只能召回/人工复核 |

---

# 11. 图片应该放在什么位置

Spec 明确说有图片，因此图片有价值，但要把权限放对。

推荐三种用途：

## 11.1 OCR 找 reference 候选

对保卡、吊牌、表背、证书等图片 OCR，把识别出的编号送入同一套 identifier pipeline。

## 11.2 交叉验证

例如：

```text
标题：REF123
OCR：REF123
```

这会增强 reference 证据。

反之：

```text
标题：REF123
OCR：REF128
```

应触发 conflict，而不是计算两个分数后“多数表决”。

## 11.3 疑难记录召回

当文本缺失 reference 时，可用视觉 embedding 找“可能同系列/同款”的候选，提供给人工或后续 OCR；但视觉近邻不能自动写入 entity_key。

原因很简单：腕表同系列不同 reference 可能视觉上极为接近，而 Spec 又明确要求不能误匹配。

---

# 12. 数据表设计建议

## 12.1 product_record

```sql
product_record(
  id,
  source,
  source_item_id,
  content_hash,
  brand_raw,
  brand_id,
  title,
  attributes_json,
  description,
  image_urls,
  updated_at
)
```

唯一键：

```text
(source, source_item_id)
```

## 12.2 identifier_evidence

```sql
identifier_evidence(
  id,
  record_id,
  raw_token,
  normalized_token,
  canonical_token,
  origin,
  field_name,
  context,
  role,
  role_score,
  verification_status,
  positive_evidence_json,
  negative_evidence_json,
  model_version,
  rule_version,
  canonicalizer_version,
  created_at
)
```

## 12.3 entity_assignment

```sql
entity_assignment(
  record_id,
  brand_id,
  canonical_reference,
  entity_key,
  decision,
  decision_reason,
  decision_version,
  created_at,
  updated_at
)
```

核心索引：

```text
INDEX (brand_id, canonical_reference)
INDEX (entity_key)
```

这样百万/千万量级不是“全局相似度搜索”问题，而是普通的键查询问题。

---

# 13. 几百条人工标签应该怎么花

用户只能接受几百对黄金标签，因此不能把它们平均随机抽样浪费掉。

## 13.1 先用弱监督造大量 role labels

可从现有结构化字段自动产生相对干净的训练数据：

- 已知平台 SKU 字段 → `PLATFORM_SKU`；
- 已知 source_item_id → `PLATFORM/SOURCE_ID`；
- 高可信 model/reference 字段 → `BRAND_REFERENCE_SELF`；
- 明确 serial 字段 → `SERIAL_NUMBER`。

人工标签主要用于 hard cases。

## 13.2 黄金集必须刻意包含 hard negatives

推荐重点标：

- 同品牌同系列、只差 1–2 字符的相邻 reference；
- SKU 中包含真正 reference 的情况；
- 标题中出现“适配/兼容/for”的 reference；
- 配件标题引用主表 reference；
- 同记录出现多个 reference；
- OCR `0/O`、`1/I`、`5/S` 等混淆；
- 分隔符差异；
- 新来源/新品牌未见格式；
- 结构化字段与标题冲突。

这类样本对减少 false positive 的价值远高于随机简单样本。

## 13.3 评测不要只看 accuracy / F1

至少输出：

```text
Auto-accept precision
Auto-accept coverage
False-positive count
Abstain rate
Conflict rate
Precision by source
Precision by brand
Precision by evidence level
```

重点指标应是：

```text
P(correct entity_key | system auto-accepted)
```

而不是总体 classification accuracy。

---

# 14. “绝对不能误匹配”在统计上意味着什么

几百条黄金标签无法统计证明 99.99% 甚至更高 precision。

如果测试中看到 0 个错误，经典近似“rule of three”告诉我们：95% 置信下错误率上界约为 `3/n`。

例如：

- n=300 且 0 错误，仍只能得到约 1% 级错误率上界；
- n=1000 且 0 错误，约 0.3% 上界；
- 想仅凭抽样把错误率上界压到 0.01% 量级，需要远多于几百样本。

因此当前项目不能依赖“模型分数很高”来满足绝对安全要求。正确做法是：

1. 用确定性 reference 身份作为最终合并依据；
2. 模型只提供候选/角色/上下文证据；
3. 边界案例全部 abstain；
4. 用强负规则 veto；
5. 上线后持续抽样审计 auto-accepted 集合。

这和原项目最大的升级点就在这里：**从普通三分类器变成带证据、冲突和拒识的安全解析系统。**

---

# 15. 增量数据与 1000 万规模怎么处理

这个方案天然适合持续增量。

## 15.1 每条记录独立解析

新记录进入时：

```text
normalize
 -> mine identifiers
 -> classify roles
 -> verify reference
 -> compute entity_key
 -> upsert assignment
```

无需和历史 1000 万条记录逐对比较。

## 15.2 content hash 避免重复计算

对会影响 reference 的字段计算 hash：

```text
brand + title + attrs + description + OCR-versioned-image-text
```

未变化则不重跑解析。

## 15.3 所有结果版本化

存：

```text
model_version
rule_version
canonicalizer_version
reference_catalog_version
```

规则更新时可以只重放受影响品牌/来源/版本的数据，而不是全库硬重算。

## 15.4 批量回填

1000 万记录完全可以按：

```text
source / brand / date partition
```

批量跑向量化推理，然后 bulk upsert entity key。因为没有笛卡尔积，瓶颈会主要落在文本/OCR 和存储 IO，而不是实体 pair 数量。

---

# 16. 推荐的最小可落地版本

## Phase A：先不用神经网络也能上线安全版

先实现：

1. 三来源 field mapping；
2. source SKU / source ID 明确排除规则；
3. identifier candidate miner；
4. 保守 canonicalizer；
5. brand + reference exact entity key；
6. conflict → abstain；
7. 人工 review queue；
8. 所有证据可追踪。

这一步就能把“误把平台货号当型号”的高危路径堵住。

## Phase B：加入原项目启发的 char role classifier

训练：

```text
BRAND_REFERENCE_SELF
PLATFORM_SKU
SELLER_INVENTORY_ID
SERIAL_NUMBER
OTHER/UNKNOWN
```

模型作用：

- 自动过滤明显平台/库存编号；
- 给未知格式风险打分；
- 发现来源规则没覆盖的新 SKU 模板。

**模型不直接产生 match。**

## Phase C：加入 context ownership verifier

专门解决：

```text
“这个字符串是一个 reference”
但
“它是不是当前商品自己的 reference？”
```

这是防止配件、兼容型号、描述引用导致误合并的关键层。

## Phase D：OCR 作为独立证据通道

OCR 候选仍进入相同 role + canonicalization + conflict pipeline，避免形成第二套不可控逻辑。

---

# 17. 建议的线上决策状态机

```text
RAW
 │
 ▼
CANDIDATE_EXTRACTED
 │
 ├─ role=SKU/SERIAL/OTHER ─────> REJECTED_IDENTIFIER
 │
 ├─ role=UNKNOWN ──────────────> ABSTAIN
 │
 ▼
REFERENCE_LIKE
 │
 ├─ ownership uncertain ───────> ABSTAIN
 │
 ├─ brand grammar fail ────────> ABSTAIN/REJECT
 │
 ▼
REFERENCE_VERIFIED
 │
 ├─ conflicting verified ref ──> CONFLICT
 │
 ▼
ENTITY_KEY_ASSIGNED
```

每一步都必须记录 `reason_code`，例如：

```text
STRUCTURED_REFERENCE_FIELD
SOURCE_SKU_PATTERN
ACCESSORY_CONTEXT
CATALOG_HIT
OCR_CORROBORATED
MULTI_REFERENCE_CONFLICT
UNKNOWN_IDENTIFIER_FORMAT
```

这样人工能理解系统为什么拒识，也方便后续统计规则效果。

---

# 18. 与 c 目录已有调研结果的组合方式

本项目和此前 c 已分析的几类方法是互补关系，而不是重复：

- `Ameli`：更适合多模态候选/细粒度属性辅助；
- `Multi-Value-Product RAG`：更适合受限候选值检索，辅助 reference extraction；
- `Confidence Classifiers with Guaranteed Accuracy or Precision`：可用于给 role/context classifier 增加 selective acceptance；
- `TransClean`：可在已经形成跨源实体图后做 false-positive 审计；
- `pyJedAI`：可以作为 blocking/ER 实验框架。

而 `parts-distributor-sku-classifier` 补上的是一个更靠前、非常关键的安全层：

```text
“先确定编号语义角色，再让 reference 进入 ER。”
```

推荐生产链路组合为：

```text
候选编号抽取
  -> 编号角色分类（本文）
  -> reference 受限检索/规范化
  -> selective verifier
  -> deterministic entity_key
  -> 多源图一致性审计
```

---

# 19. 最终推荐方案

如果要为当前 Spec 直接选一个主方案，我建议不是“训练一个商品对二分类模型”，而是下面这套 **Reference Evidence Resolver**：

```text
                    ┌─────────────────┐
                    │ Structured attrs │
                    └────────┬────────┘
                             │
Title/Description ───────────┼────> Candidate Miner
                             │
Image OCR ───────────────────┘
                             │
                             ▼
                 Identifier Role Router
              (rules + char model + context)
                             │
                             ▼
                 Conservative Canonicalizer
                             │
                             ▼
                 Brand/Reference Verifier
                             │
                       ┌─────┴─────┐
                       │           │
                   VERIFIED     ABSTAIN
                       │
                       ▼
       entity_key = hash(brand_id, canonical_ref)
                       │
                       ▼
             exact cross-source aggregation
```

### 自动合并的唯一原则

> **只有每条记录自己的 reference 都被独立验证，且 `(brand_id, canonical_reference)` 完全一致，系统才允许自动归入同一实体。**

其他任何信息——标题相似、图片相似、价格接近、系列相同、LLM 判断“看起来一样”——都只能作为辅助证据、候选召回或冲突信号，不能覆盖 reference 不一致，也不能单独触发自动合并。

这既继承了原项目“识别编号身份”的最有价值思想，也把它改造成适合当前 100 万–1000 万持续增量、字段稀疏、多模态且 **false positive 成本极高** 的生产架构。

---

## 参考源码

- 项目 README：https://github.com/pcbng/parts-distributor-sku-classifier/blob/master/README.md
- Part 1（数据清洗/字符编码/LSTM 训练）：https://github.com/pcbng/parts-distributor-sku-classifier/blob/master/parts-distributor-sku-classifier-part-1.ipynb
- Part 2（加载模型/分类别评估）：https://github.com/pcbng/parts-distributor-sku-classifier/blob/master/parts-distributor-sku-classifier-part-2-explore.ipynb
- 模型结构：https://github.com/pcbng/parts-distributor-sku-classifier/blob/master/data/trained_model_layers.json
