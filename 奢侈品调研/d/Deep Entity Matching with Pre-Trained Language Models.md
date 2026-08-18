# Deep Entity Matching with Pre-Trained Language Models（Ditto）：面向跨源二奢/腕表 reference-first 匹配的技术分析与落地方案

> 分析对象：Deep Entity Matching with Pre-Trained Language Models（Ditto）  
> 论文：https://arxiv.org/abs/2004.00584  
> 官方实现：https://github.com/megagonlabs/ditto  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 需求核心：100 万–1000 万级持续增量商品；“同一个商品”定义为**同一 reference number / 型号**；字段稀疏；reference 可能埋在标题；有图片；**precision 极端优先，允许漏匹配**；可人工标注几百对。

---

## 1. 结论先行

Ditto 值得参考，但**不能原样作为最终实体匹配器**。

它最有价值的地方是把异构商品记录序列化为统一文本，并使用预训练 Transformer 对候选 pair 做二分类：

```text
record A + record B
      ↓
COL / VAL 序列化
      ↓
Transformer cross-encoder
      ↓
[CLS] / 首 token representation
      ↓
Linear( hidden_size -> 2 )
      ↓
match probability
```

这种结构特别适合当前三源数据的几个现实问题：字段不齐、字段名不一致、标题噪声大、同一信息在不同来源里可能落在不同字段。

但当前业务已经给出一个比普通 Entity Matching 更强的身份定义：

> **同一实体 ⇔ 同一品牌语义下的同一 canonical reference number。**

因此推荐的生产架构不是“让 Ditto 学会什么叫同一块表”，而是：

1. **先抽取 reference 候选，并保留证据来源；**
2. **做 brand-aware reference canonicalization；**
3. **只有 canonical reference 一致的记录才进入自动合并候选；**
4. Ditto 只做二次验证/冲突否决，识别“虽然字符串里出现了同一 reference，但其实是配件、兼容型号、店铺 SKU、错误 OCR 或上下文不一致”；
5. reference 不一致时，**无论 Ditto 分数多高都不得匹配**；
6. reference 缺失或证据冲突时，默认 `ABSTAIN`，不自动合并。

也就是说，Ditto 在本项目中的正确定位是：

```text
Reference-first deterministic matcher
              +
High-precision semantic veto / verifier
```

而不是 generic semantic matcher。

更重要的是：一旦 identity key 已经变成 `(brand_id, canonical_reference)`，千万级实体解析就不再需要大规模 pairwise 比较。绝大多数数据可以直接通过倒排索引分组，整体复杂度从潜在的 O(N²) 降到接近 O(N)。模型只处理小部分疑难样本。

---

## 2. Ditto 原始方法解决什么问题

Ditto 将 Entity Matching 视为一个**文本 pair 分类问题**。

官方 README 给出的商品序列化形式为：

```text
COL title VAL microsoft visio standard 2007 version upgrade
COL manufacturer VAL microsoft
COL price VAL 129.95
```

完整训练样本是：

```text
<entry_1> \t <entry_2> \t <label>
```

其中 label 为：

```text
0 = no-match
1 = match
```

与传统“逐字段相似度 + 人工 feature”相比，Ditto 的核心优势是让 Transformer 自己学习跨字段、跨顺序、跨缺失模式的联合特征。

对当前三源数据，这个设计确实有意义。例如同一 reference 在不同平台可能分别表现为：

```text
雷小安：
COL title VAL 劳力士迪通拿黑陶 116500LN 全套

腕表之家：
COL brand VAL Rolex
COL reference VAL 116500LN
COL series VAL Cosmograph Daytona

奢当家：
COL title VAL ROLEX 宇宙计型迪通拿 黑盘
COL model VAL M116500LN-0002
```

只要统一序列化，模型不要求三个来源有完全一致的 schema。

论文摘要还报告：普通 BERT/DistilBERT/RoBERTa fine-tuning 已可显著提升 EM 效果；三类优化（domain knowledge、summarization、data augmentation）可继续提升；并在一个 789K × 412K 的真实大规模任务上取得较高 F1。

但这里必须强调：**论文目标主要是提高 F1，不是保证接近零 false positive。** 当前需求的优化目标完全不同，所以方法要重新裁剪。

---

## 3. 官方代码架构拆解

Ditto 的轻量实现核心模块非常小：

```text
ditto_light/
├── dataset.py       # pair 读取、tokenize、padding
├── ditto.py         # Transformer + Linear 分类器、训练、阈值评估
├── knowledge.py     # domain knowledge 注入
├── summarize.py     # TF-IDF 摘要
└── augment.py       # MixDA 数据增强
```

整体可以拆成四层：

```text
输入层
  ↓
序列化 / preprocessing
  ↓
Transformer pair encoder
  ↓
2-class classifier
  ↓
threshold decision
```

### 3.1 `DittoDataset`：pair 直接交给 tokenizer

`dataset.py` 中每条数据读取：

```python
s1, s2, label = line.strip().split('\t')
```

然后：

```python
x = tokenizer.encode(
    text=left,
    text_pair=right,
    max_length=max_len,
    truncation=True
)
```

这意味着它用的是典型 cross-encoder 结构：左右记录同时进入同一个 Transformer，让 self-attention 直接建模跨记录 token 交互。

优点：

- 对“116500LN”这种稀疏但关键的 token，可以直接形成跨侧对齐；
- 对字段缺失、顺序变化、标题与结构化字段错位比较鲁棒；
- 训练只需要 pair label，不需要手工设计几十个相似度 feature。

缺点：

- cross-encoder 单 pair 成本比 bi-encoder 高；
- 绝对不适合 1000 万记录做全量笛卡尔比较；
- 必须先有 blocking / deterministic key。

因此本项目一定要把它放在**候选裁剪之后**。

### 3.2 `DittoModel`：Transformer 首 token + 二分类 Linear

核心模型几乎就是：

```python
enc = self.bert(x1)[0][:, 0, :]
return self.fc(enc)
```

即：

```text
Transformer hidden state at token 0
        ↓
Linear(hidden_size, 2)
        ↓
softmax(match / no-match)
```

没有复杂的字段级网络、图网络或多头决策器。

这对我们的启发是：**不必为了三源腕表专门设计非常复杂的神经网络。** 真正决定 precision 的关键反而是：

- 输入里是否明确标记 reference；
- reference 是否被正确 canonicalize；
- hard negative 是否覆盖相邻型号；
- 是否允许模型绕过 reference 冲突；
- threshold / abstention 规则是否 precision-first。

### 3.3 MixDA：在 hidden representation 上混合原样本与增强样本

当启用 data augmentation 时，代码同时编码原 pair 和增强 pair：

```python
enc1 = ...
enc2 = ...
aug_lam = np.random.beta(alpha_aug, alpha_aug)
enc = enc1 * aug_lam + enc2 * (1.0 - aug_lam)
```

然后再分类。

`augment.py` 支持：

```text
del
swap
drop_col
append_col
drop_token
drop_sym
drop_same
...
```

这很适合普通 EM 的鲁棒性训练，但**不能直接照搬到 reference-first 任务**。

例如：

```text
COL reference VAL 116500LN
```

如果 `drop_col` 把 reference 整列删掉，但 label 仍然是 match，模型就会被鼓励“即使看不到型号，也根据品牌/系列/外观语义判 match”。这正是当前系统最不希望学到的行为。

所以本项目必须做 **reference-aware augmentation**：

```text
禁止删除：
- verified reference token
- reference provenance
- brand
- product_type

允许扰动：
- 营销词
- 成色描述
- 价格
- 店铺文案
- 字段顺序
- 无关长描述
```

并额外构造大量“只差一点但必须 no-match”的 hard negatives。

### 3.4 Domain Knowledge：原版 ID 规则不能直接用于腕表 reference

`ProductDKInjector` 会：

- 用 spaCy NER 标记 PRODUCT / NUM；
- 对数字做归一化；
- 对长度 >= 7 且包含数字的 token 加 `ID` 标记。

这套规则对一般商品有帮助，但腕表 reference 有两个危险点。

第一，reference 不一定 >= 7 字符：

```text
5711/1A
15202ST
114060
16233
```

第二，更危险的是把 identifier 当普通数字归一化。例如某些 reference 中的：

```text
00123
01.0240
116500-LN
```

identifier 的前导零、分隔符、字母后缀都可能有身份意义，不能经过 float 式数字标准化。

因此要新增专用 tag：

```text
REF_RAW
REF_CANON
REF_SOURCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
```

并明确：

> **reference token 永远按 identifier 处理，禁止走普通 numeric normalization。**

### 3.5 Summarization：TF-IDF 截断需要加入 reference 保护

Ditto 的 summarizer 通过 TF-IDF 保留高权重 token，使长文本控制在 `max_len` 内。

但腕表场景不能单纯按 TF-IDF 排序。低频不等于身份关键，高频也不等于无用；更重要的是 reference 可能很短，而且只出现一次。

推荐改成：

```text
Protected tokens（永远保留）
  - brand
  - reference raw/canonical
  - reference 周围上下文窗口
  - product_type
  - series/model
  - OCR 中的 reference span

Optional tokens（按 TF-IDF / rule 截断）
  - 长描述
  - 营销词
  - 配送说明
  - 成色套话
  - 店铺固定模板
```

另外，原版 `summarize.py` 构建 IDF 时会读取 train、valid、test。生产实现中建议只基于 train/历史无标签语料建立统计，避免离线评测信息泄漏。

### 3.6 原版阈值策略与本项目目标相反

`evaluate()` 会遍历：

```python
for th in np.arange(0.0, 1.0, 0.05):
```

并选择**验证集 F1 最高**的 threshold。

这是当前任务最需要改掉的地方。

因为：

```text
F1 最优 ≠ precision 最优
```

如果增加一些 false positive 能换来更多 recall，F1 甚至可能变高；但当前业务明确不能接受这种 trade-off。

生产策略应该是：

```text
reference hard rule
    AND
no contradiction
    AND
verifier_score >= very_high_threshold
    ELSE ABSTAIN
```

模型阈值不再用 F1 选择，而是以**可接受 false-positive 上限**为目标，并保留大量拒识。

---

## 4. 最关键的架构调整：不要做“全量 pair matching”，改成 reference entity assignment

当前需求其实可以重新定义为：

```text
每条商品记录
    ↓
识别它属于哪个 canonical reference entity
```

而不是：

```text
任意两条商品记录
    ↓
逐对判断是否相同
```

这两种建模方式在千万级规模上差别巨大。

### 4.1 推荐实体主键

最终 identity key：

```text
(brand_id, canonical_reference)
```

例如：

```text
(ROLEX, 116500LN)
(AP, 15202ST.OO.1240ST.01)
(PATEK_PHILIPPE, 5711/1A-010)
```

如果业务确认某些品牌 reference 需要更复杂作用域，再扩展为：

```text
(brand_id, reference_namespace, canonical_reference)
```

### 4.2 匹配从 O(N²) 变成近似 O(N)

如果每条记录都能得到一个 verified canonical reference，则只需要：

```sql
SELECT *
FROM product_reference_assignment
WHERE brand_id = ?
  AND canonical_reference = ?;
```

新增记录直接加入现有 entity group。

不需要：

```text
新记录 × 全库 1000 万条
```

甚至不需要 ANN。

Ditto 只对低置信候选做验证，因此 GPU 成本可以压缩几个数量级。

---

## 5. 推荐的生产架构

```text
┌──────────────────────────────────────────────┐
│ 雷小安 / 腕表之家 / 奢当家 source adapters │
└─────────────────────┬────────────────────────┘
                      ↓
┌──────────────────────────────────────────────┐
│ Raw Product Store                            │
│ source_item_id / raw fields / images/version │
└─────────────────────┬────────────────────────┘
                      ↓
┌──────────────────────────────────────────────┐
│ Brand Normalizer                             │
│ Rolex / 劳力士 / ROLEX -> brand_id          │
└─────────────────────┬────────────────────────┘
                      ↓
┌──────────────────────────────────────────────┐
│ Reference Candidate Extractor                │
│ 1. structured field                          │
│ 2. title / description parser                │
│ 3. OCR from image                            │
│ 4. optional LLM extractor                    │
└─────────────────────┬────────────────────────┘
                      ↓
┌──────────────────────────────────────────────┐
│ Reference Canonicalizer                      │
│ brand-aware + identifier-safe normalization  │
└─────────────────────┬────────────────────────┘
                      ↓
┌──────────────────────────────────────────────┐
│ Evidence Resolver                            │
│ conflict detection / provenance / confidence │
└───────────────┬──────────────────────────────┘
                ↓
        verified ref candidate?
          /              \
        no                yes
        ↓                  ↓
     ABSTAIN       exact reference index lookup
                           ↓
                 candidate entity / anchor
                           ↓
                Ditto high-precision verifier
                           ↓
                ┌──────────┴──────────┐
                ↓                     ↓
             accept                ABSTAIN
                ↓
        Reference Entity Index
```

关键原则：

> **模型没有“越权匹配”权限。它只能阻止或拒识，不能把两个 reference 不同的记录强行判成同一实体。**

---

## 6. Reference 抽取与证据模型

每个 reference 候选不能只存一个字符串，必须保留完整 provenance：

```json
{
  "record_id": "sdj:123456",
  "brand_id": "ROLEX",
  "raw_ref": "M116500LN-0002",
  "canonical_ref": "116500LN",
  "evidence_type": "title",
  "span": "M116500LN-0002",
  "confidence": 0.97,
  "extractor_version": "ref-parser-v3",
  "image_id": null,
  "bbox": null
}
```

`evidence_type` 至少区分：

```text
structured_reference_field
structured_model_field
title_regex
description_regex
ocr_caseback
ocr_card
ocr_tag
llm_extraction
manual
```

这样线上出现误匹配时可以回答：

```text
为什么这个商品被归到 116500LN？
```

而不是只看到一个不可解释的 0.996 模型分数。

---

## 7. Reference canonicalization：宁可少归一化，不要过归一化

precision-first 场景里 canonicalization 是最危险的基础模块之一。

推荐分两层：

### 7.1 `retrieval_ref`

用于把明显的格式差异召回到一起，例如：

```text
116500-LN
116500 LN
116500LN
```

可以统一成：

```text
116500LN
```

### 7.2 `canonical_ref`

用于真正身份判定，应尽量保留品牌规则中有区分意义的部分：

```text
5711/1A-010
5711/1A-011
```

不能因为只差后缀就合并。

因此 canonicalizer 必须是：

```text
brand-aware
versioned
reversible enough for audit
```

而不是一个全局 `re.sub('[^A-Z0-9]', '', ref)`。

建议每次归一化输出：

```json
{
  "raw": "M116500LN-0002",
  "retrieval_ref": "116500LN0002",
  "canonical_ref": "116500LN",
  "rule_id": "rolex-official-prefix-suffix-v2",
  "rule_version": 2
}
```

只有经过已验证品牌规则的 canonical 值才拥有自动匹配资格。

---

## 8. Ditto 在本项目中的正确输入设计

不推荐把三个来源所有原始字段直接粗暴拼起来。

建议明确序列化：

```text
COL source VAL leidian
COL product_type VAL watch
COL brand VAL ROLEX
COL title VAL 劳力士 迪通拿 黑陶 116500LN 全套
COL reference_raw VAL 116500LN
COL reference_canonical VAL 116500LN
COL reference_source VAL title_regex
COL series VAL Daytona
COL condition VAL 95new
```

另一侧：

```text
COL source VAL xxxxx
COL product_type VAL watch
COL brand VAL ROLEX
COL title VAL 宇宙计型迪通拿 黑盘 116500LN
COL reference_raw VAL M116500LN-0002
COL reference_canonical VAL 116500LN
COL reference_source VAL structured_reference_field
COL series VAL Cosmograph Daytona
```

其中以下字段必须视为 protected：

```text
brand
product_type
reference_raw
reference_canonical
reference_source
```

这样模型学习的不是“两个标题听起来像不像同一块表”，而是：

> 在 reference 已经一致的前提下，是否存在足以否决该 identity assignment 的上下文冲突？

---

## 9. 重点训练样本：hard negatives 比普通随机负例重要得多

随机负例非常容易：

```text
Rolex Submariner vs Cartier Tank
```

这类样本对提高 high-precision 几乎没有价值。

真正需要人工标注的负例应该来自同品牌、同系列、外观相近的边界：

```text
116500LN vs 126500LN
126610LN vs 126610LV
5711/1A-010 vs 5711/1A-011
15400ST vs 15500ST
```

以及最容易造成 false positive 的业务噪声：

```text
标题中写“适配 116500LN”的表带
盒证/配件商品引用主体腕表型号
店铺内部 SKU 看起来像 reference
OCR 把 0 / O、1 / I、5 / S 识别错
平台型号字段实际是系列名
标题里同时出现多个兼容型号
卖家文案“同款 116500LN”但商品不是该 reference
```

几百对黄金标签建议不要平均随机抽，而要**定向覆盖 hard cases**。

一个可执行的第一版标注预算：

```text
100 对：明确正例，同 reference 跨来源表达差异大
250 对：hard negative，同品牌/同系列/相邻 reference
100 对：reference 角色错误（SKU、配件兼容号、OCR、多个候选）
50  对：字段极稀疏 / 缺失 / 冲突，标记为 abstain benchmark
```

总计约 500 对即可启动第一轮，但这 500 对主要用于**找错误模式和调拒识策略**，不应被理解为足以统计证明“万分之一误匹配率”。

如果想对 99.9%+ precision 做可靠统计验证，需要远多于几百个已自动接受样本的审计量；所以第一版的极高 precision 主要依赖 deterministic reference gate，而不是模型概率本身。

---

## 10. Reference-aware 数据增强

原 Ditto 的 MixDA 思路可以保留，但增强算子必须重写。

### 10.1 可以做的正例增强

```text
大小写变化
116500ln -> 116500LN

安全分隔符变化
116500-LN -> 116500 LN

字段顺序变化
brand/reference/title -> title/brand/reference

删除无关营销文案
“全套附件齐全支持鉴定” -> 删除

来源常见别名变化
劳力士 -> ROLEX
```

### 10.2 不应该做的增强

```text
删除 reference
删除 reference suffix
把 126500LN 改成 116500LN 后仍保留 positive label
把多个 reference 候选压成一个
把 identifier 当数字四舍五入
```

### 10.3 必须主动生成的负例

可从真实 canonical reference 库生成候选：

```text
same brand + same series + nearest edit distance
same prefix + different suffix
same digits + different letters
old generation + new generation
```

例如：

```python
hard_negative_pool = index.query(
    brand_id=brand,
    series=series,
    ref_prefix=ref[:4]
)
```

这些 hard negatives 对 precision 的价值远高于普通 random negative sampling。

---

## 11. 决策规则：模型只做 veto，不做 override

推荐把最终决策写成明确、可单元测试的规则，而不是 `score > 0.5`。

```python
def decide(record_a, record_b, verifier_score):
    # 1. 品牌必须一致
    if record_a.brand_id != record_b.brand_id:
        return "NO_MATCH", "BRAND_CONFLICT"

    # 2. 两侧都必须有 verified reference
    if not record_a.verified_ref or not record_b.verified_ref:
        return "ABSTAIN", "REFERENCE_MISSING"

    # 3. canonical reference 不一致，绝对禁止模型 override
    if record_a.canonical_ref != record_b.canonical_ref:
        return "NO_MATCH", "REFERENCE_CONFLICT"

    # 4. 商品类型冲突，拒绝
    if not compatible_product_type(record_a, record_b):
        return "ABSTAIN", "PRODUCT_TYPE_CONFLICT"

    # 5. 同 reference 后，Ditto 只做语义/上下文冲突过滤
    if verifier_score < HIGH_PRECISION_THRESHOLD:
        return "ABSTAIN", "VERIFIER_LOW_CONFIDENCE"

    return "MATCH", "REFERENCE_EXACT_AND_VERIFIED"
```

如果两侧 reference 都来自高可信结构化字段，甚至可以跳过 Ditto，直接按 key 分组；模型资源集中给：

```text
reference 来自标题
reference 来自 OCR
同一条记录出现多个候选
疑似配件/兼容型号
结构化字段与标题冲突
```

---

## 12. 不要保存 pair edge 为主，保存 reference assignment 为主

传统 entity matching 常建：

```text
A -> B match edge
A -> C match edge
B -> C match edge
```

但本项目更适合保存：

```text
record A -> entity(ROLEX, 116500LN)
record B -> entity(ROLEX, 116500LN)
record C -> entity(ROLEX, 116500LN)
```

推荐核心表：

### `product_record`

```text
id
source
source_item_id
raw_payload
brand_id
title
product_type
updated_at
content_hash
```

唯一键：

```text
(source, source_item_id)
```

### `reference_evidence`

```text
id
record_id
raw_ref
retrieval_ref
canonical_ref
evidence_type
span_or_bbox
confidence
extractor_version
created_at
```

### `reference_assignment`

```text
record_id
brand_id
canonical_ref
decision            # VERIFIED / ABSTAIN / CONFLICT
verifier_score
reason_code
rule_version
model_version
updated_at
```

索引：

```sql
CREATE INDEX idx_reference_entity
ON reference_assignment(brand_id, canonical_ref)
WHERE decision = 'VERIFIED';
```

### `reference_entity`

```text
entity_id
brand_id
canonical_ref
first_seen_at
last_seen_at
```

唯一键：

```text
(brand_id, canonical_ref)
```

这套 schema 天然支持持续增量和审计。

---

## 13. 增量更新架构

每条商品更新可以走幂等 pipeline：

```text
source item changed
      ↓
content_hash changed?
      ↓ yes
re-run brand/ref extraction
      ↓
canonicalize
      ↓
resolve evidence
      ↓
assignment changed?
      ↓
update entity membership
```

建议为每个处理阶段记录版本：

```text
brand_normalizer_version
reference_extractor_version
canonicalizer_version
ocr_version
verifier_model_version
decision_rule_version
```

当品牌规则修复时，可以精确找出需要重跑的记录，而不是全库不可控重算。

1000 万级数据不需要一开始就上非常复杂的流计算平台。第一版用：

```text
PostgreSQL / MySQL + 对象存储 + batch worker / queue
```

就足够验证业务闭环。

吞吐扩大后再替换为：

```text
Kafka / Pulsar
+ worker pool
+ analytical warehouse
```

核心匹配逻辑不变，因为 identity lookup 本质上是 exact key lookup。

---

## 14. 图片应该怎么用

当前有图片，但图片不应该成为同 reference 的替代定义。

最安全、最有价值的图片用途是**提取 reference 证据**：

```text
表背刻字 OCR
保卡 OCR
吊牌 OCR
盒贴 OCR
机芯/壳号局部 OCR
```

推荐流程：

```text
image
  ↓
region / OCR
  ↓
reference candidate
  ↓
brand-aware canonicalizer
  ↓
与 title / structured reference 交叉校验
```

证据融合：

```text
structured_ref == title_ref == OCR_ref
    -> very high confidence

structured_ref == title_ref, OCR missing
    -> high confidence

title_ref == OCR_ref, structured missing
    -> high confidence, 可进入 Ditto verifier

structured_ref != title_ref
    -> ABSTAIN

title_ref != OCR_ref
    -> ABSTAIN / 人工复核
```

视觉 embedding 可以用于辅助判断“这是腕表本体还是表带/盒子/配件”，但**不能因为两张表看起来一样就跨过 reference 规则直接合并**。

同系列不同 reference 外观非常接近，这一点在腕表场景尤其危险。

---

## 15. 现代化重写 Ditto 时需要修的工程点

官方仓库是研究代码，生产落地不建议原样部署。

### 15.1 tokenizer padding / attention mask

轻量实现手工用 `0` padding：

```python
x12 = [xi + [0] * (maxlen - len(xi)) for xi in x12]
```

模型调用时又没有显式传 `attention_mask`：

```python
self.bert(x1)
```

这对不同 tokenizer 的 pad token 不够稳健，尤其当 backbone 从 DistilBERT 换成 RoBERTa/XLM-R 时。

生产版应直接使用 HuggingFace tokenizer：

```python
tokenizer(
    left,
    right,
    padding=True,
    truncation=True,
    return_tensors="pt"
)
```

并传：

```text
input_ids
attention_mask
token_type_ids（模型支持时）
```

### 15.2 中文/中英混合 backbone

当前三源标题以中文为主，并含大量英文品牌名和字母数字 identifier。建议第一版直接测试：

```text
xlm-roberta-base
```

而不是固定老版本英文 DistilBERT。

但 backbone 不是最重要变量；在 precision-first 任务中，reference gate 与 hard-negative 设计通常比把 base 模型换大更关键。

### 15.3 inference batch

只处理疑难候选后，可以用普通 GPU batch inference：

```text
batch 64 / 128
fp16 / bf16
```

不需要把 1000 万全部过 cross-encoder。

### 15.4 输出必须带 reason code

不要只有：

```json
{"score": 0.9987}
```

而要：

```json
{
  "decision": "MATCH",
  "reason": "REFERENCE_EXACT_AND_VERIFIED",
  "canonical_ref": "116500LN",
  "reference_evidence": ["title_regex", "ocr_card"],
  "verifier_score": 0.9987,
  "model_version": "ditto-xlm-r-v3",
  "rule_version": "match-policy-v5"
}
```

这对误匹配追责、规则回滚和人工复核非常重要。

---

## 16. High-precision threshold 的选择

原版 Ditto 用最大 F1 阈值，这里应换成 precision-first 校准。

建议在验证集上把候选按模型分数排序，寻找满足目标 precision 的最低阈值，但必须注意：

```text
几百个验证样本不足以证明极端高 precision
```

所以不要做：

```text
验证集 100 个 accept 全对
=> 宣称线上 precision = 100%
```

正确策略是：

1. deterministic reference exact match 负责主要安全边界；
2. Ditto 作为 veto，只会减少自动匹配，不会制造 reference 冲突匹配；
3. 对自动 accept 持续做随机人工 audit；
4. 按来源、品牌、reference pattern 分桶看 precision；
5. 新品牌、新规则、新 OCR 版本先 shadow；
6. 发生任何 false positive 时，优先定位 `reason_code + evidence + version`，而不是直接调一个全局阈值。

如果后续需要统计意义上的风险控制，可以把 Ditto score 再接现有的 selective / conformal calibration 层；但这仍然不能替代 reference hard gate。

---

## 17. 评测集不能随机切分

普通随机 train/test 很容易把相同 reference 或相似模板同时放到训练集和测试集，导致结果虚高。

建议至少做四套评测：

### A. `seen-reference / seen-source`

验证常规线上数据。

### B. `unseen-reference`

测试新型号进入时能否保持 precision。

### C. `cross-source shift`

例如：

```text
train: 雷小安 + 腕表之家
validation/test: 奢当家
```

验证来源模板变化。

### D. `hard-negative benchmark`

只放：

```text
same brand
same series
near reference
accessory mention
seller SKU
OCR confusion
```

最终上线最看重 D，不应该只看总体 F1。

指标建议：

```text
Auto-match precision        # 第一优先级
False positive count        # 直接看绝对数
Coverage / auto-match rate  # 第二优先级
Abstain rate
Reference extraction precision
Reference conflict rate
Per-brand precision
Per-source precision
```

Recall/F1 可以记录，但不能主导阈值选择。

---

## 18. 第一版可直接落地的 MVP

### Phase 0：纯 deterministic baseline

不训练模型，先完成：

```text
1. brand canonicalization
2. structured reference extraction
3. title regex / grammar extraction
4. reference canonicalization
5. exact index grouping
6. conflict -> abstain
```

这一步很可能已经能高精度覆盖相当一部分数据。

### Phase 1：建立 reference evidence 标注

抽样 300–500 个困难案例，重点标：

```text
reference 是否真属于当前商品
是否是平台 SKU
是否是兼容/配件引用
canonicalization 是否正确
跨源 pair 是否应该属于同 reference entity
```

### Phase 2：训练 Ditto-style verifier

建议：

```text
backbone: xlm-roberta-base
input: structured COL/VAL pair
loss: cross entropy
training: hard negatives > random negatives
augmentation: reference-aware only
```

模型只进入：

```text
title/OCR ref
多候选
字段冲突边界
```

### Phase 3：图片 OCR

只为 reference 提供额外证据，不做视觉相似度直接 match。

### Phase 4：shadow + audit

新 verifier 先不改变实体归属，只记录：

```text
old decision
new decision
score
reason
```

人工审查差异后再放量。

---

## 19. 推荐服务接口

### Reference extraction

```http
POST /reference/extract
```

输入：

```json
{
  "source": "sdj",
  "brand": "劳力士",
  "title": "劳力士迪通拿黑陶 116500LN 全套",
  "attrs": {},
  "images": []
}
```

输出：

```json
{
  "brand_id": "ROLEX",
  "candidates": [
    {
      "raw": "116500LN",
      "canonical": "116500LN",
      "evidence": "title_regex",
      "confidence": 0.99
    }
  ],
  "status": "RESOLVED"
}
```

### Entity assignment

```http
POST /entity/assign
```

输出：

```json
{
  "entity_id": "ROLEX#116500LN",
  "decision": "VERIFIED",
  "reason": "REFERENCE_EXACT_AND_VERIFIED"
}
```

如果冲突：

```json
{
  "entity_id": null,
  "decision": "ABSTAIN",
  "reason": "REFERENCE_EVIDENCE_CONFLICT"
}
```

### Ditto verifier

```http
POST /verifier/pair
```

只接受已经通过 reference precondition 的 pair：

```json
{
  "left_record_id": "lxa:1",
  "right_record_id": "wbzj:2",
  "required_canonical_ref": "116500LN"
}
```

如果两条记录 canonical reference 不同，API 在进入模型前直接拒绝请求。

---

## 20. 一个重要的产品边界：缺 reference 时不要“猜成同款”

这是本项目最容易被通用 EM 模型带偏的地方。

假设：

```text
A: 劳力士 迪通拿 黑陶 黑盘 全套 2020
B: Rolex Daytona 116500LN black dial full set
```

Ditto 很可能会给较高相似度。

但如果 A 没有可信 reference 证据，当前业务定义下最安全答案是：

```text
ABSTAIN
```

而不是：

```text
MATCH 116500LN
```

因为相同系列可能存在极相近的 reference，文本甚至图片都不足以替代 reference 本身。

如果后续业务希望“推测 reference”，那应该单独定义一个：

```text
reference inference task
```

其输出仍需独立置信度和证据审计，不能混在实体匹配模型里偷偷完成。

---

## 21. 与当前 Spec 的最终映射

| Spec 约束 | Ditto 原始能力 | 推荐改造 |
|---|---|---|
| 100万–1000万级 | 只负责 candidate pair matching | 用 `(brand, canonical_ref)` exact index，模型只跑疑难记录 |
| 持续增量 | 可离线 inference | 幂等 assignment pipeline + versioned extractor/model |
| reference 才定义同实体 | 通用语义 match | reference hard gate，模型不能 override |
| 字段高度稀疏 | 很适合 COL/VAL 序列化 | 保留并显式标记 reference provenance |
| reference 埋在标题 | 可利用上下文 | 先抽取 candidate，再由 verifier 做上下文确认 |
| 绝不能误匹配 | 原版优化 F1 | high-precision threshold + veto-only + abstain |
| 有图片 | 原版主要文本 | 图片用于 OCR/reference 证据与商品类型冲突检测 |
| 只有几百标签 | Ditto 具备较好 label efficiency | 标签全部集中在 hard negatives 和 reference role 错误 |

---

## 22. 最终建议

如果需要现在就落地，我建议不要先训练一个“万能匹配模型”，而按下面顺序实施：

```text
第一优先级：
brand normalization
+ reference extraction
+ canonicalization
+ exact entity index
+ conflict abstention

第二优先级：
Ditto-style XLM-R cross-encoder
只验证 reference 已一致但上下文存在风险的候选

第三优先级：
image OCR -> reference evidence

第四优先级：
持续 hard-negative mining
+ 人工 audit
+ selective calibration
```

Ditto 真正值得借鉴的是：

1. 用简单统一的 `COL / VAL` 形式吸收异构 schema；
2. 用预训练 LM 做候选级 cross-encoder；
3. 用 domain knowledge 强调 identifier；
4. 用 hard augmentation 提升难例判别；
5. 通过 blocking 将 matcher 与大规模候选生成解耦。

但当前项目必须反过来限制它：

> **reference 是身份事实，模型只是辅助证据。**

在这个原则下，Ditto 可以成为一个很有效的“误匹配保险丝”；如果让它直接负责最终 match，则会把当前最核心的 precision-first 约束重新退化成普通 F1 优化问题。

---

## 参考

- Yuliang Li, Jinfeng Li, Yoshihiko Suhara, AnHai Doan, Wang-Chiew Tan. *Deep Entity Matching with Pre-Trained Language Models*. https://arxiv.org/abs/2004.00584
- Ditto official repository: https://github.com/megagonlabs/ditto
- Core model implementation: https://github.com/megagonlabs/ditto/blob/master/ditto_light/ditto.py
- Dataset serialization/tokenization: https://github.com/megagonlabs/ditto/blob/master/ditto_light/dataset.py
- Domain knowledge injection: https://github.com/megagonlabs/ditto/blob/master/ditto_light/knowledge.py
- Summarization: https://github.com/megagonlabs/ditto/blob/master/ditto_light/summarize.py
- Data augmentation: https://github.com/megagonlabs/ditto/blob/master/ditto_light/augment.py
