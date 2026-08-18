# WDC Products: A Multi-Dimensional Entity Matching Benchmark

> 分析者：d  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 选取来源：`奢侈品文章调研.md` 中的 WDC Products 条目（https://arxiv.org/abs/2301.09521）  
> 论文：Ralph Peeters, Reng Chiz Der, Christian Bizer, *WDC Products: A Multi-Dimensional Entity Matching Benchmark*, EDBT 2024  
> 官方项目：https://github.com/wbsg-uni-mannheim/wdcproducts  
> Benchmark 页面：https://webdatacommons.org/largescaleproductcorpus/wdc-products/index.html

## 1. 为什么这篇值得用于当前 Spec

当前需求并不是普通的“相似商品搜索”，而是一个 **precision 极端优先、允许大量 abstain/漏匹配** 的跨源商品实体匹配系统。需求又明确规定：

- 同一个商品的定义是“同一参考号 / reference number / 型号”；
- 三个来源字段稀疏、脏、schema 不一致，reference 可能独立存在，也可能埋在标题或图片中；
- 数据规模 100 万到 1000 万，并持续增量；
- 不能误匹配；
- 有图片；
- 可以人工标注几百对作为黄金标签。

WDC Products 本身不是一套可以直接拿来上线的 matching 服务，它真正有价值的地方是：它把“一个 matcher 到底有没有在真实困难条件下可靠工作”拆成了三个可控维度，并给出一套可复现的数据构造与评测方式：

1. **corner cases 比例**：专门放大“看起来很像但其实不是同一商品”的 hard negative，以及“文本看起来差异很大但其实相同”的 hard positive；
2. **unseen entities 比例**：专门测试训练时没出现过的新商品；
3. **development set size**：专门测试少量标注时模型是否仍然稳定。

这三个维度和本需求几乎一一对应：

- 腕表最危险的是同系列、同外观、reference 只差 1～2 个字符的 false positive；
- 每天都会出现训练集中从未出现过的新 reference；
- 我们只有几百对黄金标签，不具备“大量人工标注兜底”的条件。

因此，WDC Products 最适合被借鉴为当前系统的 **测试集生成器 + hard-negative 生成策略 + 模型上线门禁**，而不是直接照搬成最终 matcher。

---

## 2. WDC Products 的核心设计

### 2.1 数据规模与来源

WDC Products 使用 2020 年 Common Crawl 中的 schema.org 商品数据。论文最终 benchmark 包含：

- 3,259 个电商站点；
- 11,715 条 product offers；
- 2,162 个真实商品实体；
- title、description、brand、price、priceCurrency 等属性。

底层原始 PDC2020 语料更大，论文描述其最初约有 9800 万条商品 offer，利用 MPN、GTIN、SKU 等商品标识符把 offer 聚成 cluster，之后再做清洗与 benchmark 构造。

这里和本需求最重要的共性是：**WDC 也没有一开始就把“文本相似”当真值，而是优先用商品 identifier 构造实体 cluster，再把文本模型用于评测/匹配。**

对当前需求来说，reference number 就应当扮演类似 MPN/GTIN 的“实体锚点”角色。

### 2.2 三维 Benchmark

WDC Products 在三个维度上分别取多个档位：

- corner-case：20% / 50% / 80%；
- unseen products：0% / 50% / 100%；
- development set：small / medium / large。

组合后形成 27 个可比的 benchmark variant。

这比“随机切一份 train/test 然后报 F1”强很多。普通随机切分在商品匹配里非常容易高估效果：同一个商品、甚至同一个卖家的相似模板文本可能同时出现在训练集和测试集，模型实际上学会了记忆，而不是学会识别新 reference。

WDC Products 明确按 offer 做互斥 split，避免同一条记录跨 train/validation/test 泄漏；unseen 维度进一步保证测试实体本身可以完全没出现在训练集。

### 2.3 Hard Negative 的构造方式

论文对 corner case 的定义非常适合腕表：

- hard positive：同一商品，但两个 offer 的文本表面差异很大；
- hard negative：不同商品，但文本表面高度相似。

WDC 先用 DBSCAN 把“相似产品”做粗分组，然后在组内为 seed product 找最相似但不同的 product cluster。为了避免 benchmark 被单一相似度算法“做题”，其 hard case 并不是固定使用一种距离，而是随机在多种相似度之间切换，包括：

- Cosine；
- DICE；
- Generalized Jaccard；
- fastText embedding similarity。

这点对腕表非常关键。我们的 hard negative 不能只用字符串编辑距离生成，否则 matcher 只需要反向适配这一个指标即可。应该混合多种难例来源：

- reference 编辑距离近；
- 同品牌同系列；
- 标题 BM25/embedding 近；
- 图片 embedding 近；
- OCR 结果近；
- 价格区间近；
- 卖家标题模板近。

### 2.4 代码层面的模型结构

官方 repo 不只提供 benchmark，还包含 R-SupCon / pair-wise / multi-class 的训练代码。

其 `src/contrastive/models/modeling.py` 的结构可以概括为：

```text
record text
   ↓
Transformer Encoder (AutoModel)
   ↓
mean pooling / pooler output
   ↓
L2 normalize
   ↓
Supervised Contrastive pre-training
   ↓
下游 pair-wise classifier 或 multi-class classifier
```

pair-wise classifier 对左右两条记录分别编码，再组合：

```text
L
R
|L - R|
L * R
```

官方实现支持如 `concat-abs-diff-mult`，即：

```text
[L, R, abs(L-R), L*R]
      ↓
Dropout
      ↓
Linear(hidden*4 -> 1)
      ↓
Sigmoid
```

训练使用 BCEWithLogitsLoss；对比学习部分使用标准 Supervised Contrastive Loss，temperature 默认可配置为 0.07。

代码中的 `preprocess_wdcproducts.py` 则体现了一个很朴素但非常生产化的原则：**输入先统一清洗、HTML unescape、空值处理、属性截断，然后才训练模型。** 例如 title、brand、description 分别有长度上限。

这个原则对三源二奢数据同样重要：不要直接把各站原始 JSON 扔给 LLM/Transformer，而是要先构造稳定的 canonical product record。

---

## 3. WDC Products 给当前需求的最重要结论

论文实验中，不同 benchmark variant 上先进 matcher 的 Top-F1 仍会有明显差异，论文报告大致在 0.64～0.89 区间；所有系统在 unseen entities 上都会显著下降。论文还观察到 supervised contrastive learning 相比 cross-encoder 在小标注条件下更数据高效。

这对当前需求有两个直接含义。

### 3.1 不应该把“模型概率高”当成自动合并依据

如果连 benchmark 上的强模型面对 unseen product 都会明显退化，那么生产环境里出现新品牌、新系列、新卖家模板时，单一分类器 score 更不能当“同一 reference”的最终真值。

当前业务又明确要求“绝对不能误匹配”，所以生产规则应该是：

> **模型可以帮助找候选、排序、识别疑难样本、触发拒绝，但不能推翻 reference 的硬证据。**

### 3.2 必须把 hard negative 做成一等公民

随机负样本对这个需求几乎没有价值。

例如：

```text
Rolex Submariner 126610LN
Rolex Submariner 126610LV
```

对于普通文本 embedding 来说二者极其相似，对图像 embedding 也可能非常接近，但按当前业务定义它们必须是不同实体。

因此，训练集和测试集都必须高比例包含这种“同品牌、同系列、相邻 reference”的负样本；系统上线门槛也应该主要看这类样本的 false positive，而不是看整体 F1。

---

# 4. 建议直接落地的整体架构

下面给出一个针对 100 万～1000 万级、持续增量、precision-first 的可直接实施方案。

## 4.1 总体架构

```text
雷小安 / 腕表之家 / 奢当家
          │
          ▼
┌──────────────────────┐
│  Raw Ingestion Layer  │
│ 原始 JSON / HTML / 图像 │
└──────────┬───────────┘
           ▼
┌────────────────────────────┐
│ Canonicalization / Parsing │
│ 品牌归一 / 类目 / 文本清洗 │
└──────────┬─────────────────┘
           ▼
┌───────────────────────────────┐
│ Reference Extraction Service  │
│ 字段 + 标题 + OCR + 候选字典 │
└──────────┬────────────────────┘
           ▼
┌─────────────────────────────┐
│ Reference Evidence Resolver │
│ 角色识别 / 冲突检测 / 置信级别 │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ Exact Reference Index       │
│ (brand_id, ref_canonical)   │
└──────────┬──────────────────┘
           │
      exact candidate
           ▼
┌─────────────────────────────┐
│ Safety Gate / Verifier      │
│ 属性冲突 + OCR + 图片辅助否决 │
└──────────┬──────────────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
AUTO_LINK     ABSTAIN
     │           │
     ▼           ▼
Entity Store   Review Queue
```

最核心的设计点是：**自动匹配主路径不是 ANN 相似度检索，而是 canonical reference 的 exact lookup。**

图片、文本 embedding、LLM 都放在“抽取、验证、拒绝、人工复核”链路中，而不是直接产生实体等价关系。

---

# 5. Canonical Product Record

先把三源数据统一成一个稳定 schema：

```json
{
  "record_id": "sdj:123456",
  "source": "shedangjia",
  "source_product_id": "123456",
  "category": "watch",
  "brand_raw": "劳力士",
  "brand_id": "rolex",
  "title_raw": "劳力士潜航者 126610LN 全套",
  "title_norm": "劳力士 潜航者 126610LN 全套",
  "description_norm": "...",
  "reference_raw_candidates": [
    {
      "value": "126610LN",
      "origin": "title",
      "span": [7, 15]
    }
  ],
  "reference_canonical": "126610LN",
  "reference_role": "PRODUCT_REFERENCE",
  "reference_confidence": "A",
  "image_urls": ["..."],
  "image_ocr_tokens": ["126610LN"],
  "ingested_at": "...",
  "parser_version": "ref-parser-2026-08-18-v1"
}
```

这里需要保留 `raw`、`norm`、`canonical` 三个层次，绝不能只留下最终 reference。

原因是后续误匹配调查必须回答：

- reference 原始文本是什么；
- 是哪个 extractor 找到的；
- 在标题第几个字符；
- 经过了哪些 normalization；
- 当时使用哪个 parser version；
- 有没有 OCR / 图片证据；
- 为什么被自动放行。

对于 precision-first 系统，**可审计性和模型本身同等重要。**

---

# 6. Reference 抽取与规范化

## 6.1 多通道抽取，不做自由生成

建议 reference extractor 返回 **候选集合**，而不是让 LLM 直接生成一个字符串：

```text
ExplicitFieldExtractor
TitleRegexExtractor
BrandDictionaryExtractor
DescriptionExtractor
ImageOCRExtractor
LLMSpanExtractor（最后兜底）
          │
          ▼
ReferenceCandidate[]
```

每个候选必须包含：

```text
value
source_channel
source_span / image_id
extractor_version
confidence
```

LLM 如果使用，也应该被约束成 span extraction：只能从输入文本中复制候选，不允许自由生成不存在的型号。

## 6.2 Canonicalization

建议 normalization 顺序固定且版本化：

```python
def canonicalize_reference(raw: str) -> str:
    s = unicode_nfkc(raw)
    s = s.upper().strip()
    s = normalize_dash(s)
    s = remove_zero_width_chars(s)
    s = normalize_spaces(s)
    s = brand_specific_normalize(s)
    return s
```

注意不要一开始就粗暴删除所有 `-`、`.`、`/`。

某些品牌中这些符号可能有语义。更稳妥的方式是：

```text
raw_reference
    ↓
generic normalization
    ↓
brand-aware grammar
    ↓
canonical reference
```

即不同品牌维护自己的 grammar / alias rules。

例如配置表：

```yaml
rolex:
  patterns:
    - '[0-9]{5,6}[A-Z]{0,3}'
  remove_chars: [' ']

omega:
  patterns:
    - '[0-9]{3}\.[0-9]{2}\.[0-9]{2}\.[0-9]{2}\.[0-9]{3}\.[0-9]{3}'
  preserve_chars: ['.']
```

不要把所有品牌混成一个“万能 regex”。

---

# 7. Reference Role Classification：防止把 SKU / 配件型号当主型号

这是最容易制造灾难性 false positive 的地方之一。

标题可能同时包含：

- 腕表 reference；
- 店铺 SKU；
- 平台商品 ID；
- 表带适配 reference；
- 机芯型号；
- 保卡编号；
- 盒子 / 配件型号。

因此抽到一个“长得像型号”的 token 以后不能立刻 exact match，而应先做 role classification：

```text
PRODUCT_REFERENCE
PLATFORM_SKU
SHOP_SKU
COMPATIBLE_REFERENCE
MOVEMENT_REFERENCE
SERIAL_NUMBER
ACCESSORY_REFERENCE
UNKNOWN
```

只有：

```text
reference_role == PRODUCT_REFERENCE
```

才能进入自动匹配链路。

role classifier 建议优先规则 + 少量分类模型：

```text
上下文关键词
+ 字段来源
+ 品牌 grammar
+ token 长度/结构
+ 页面 DOM label
+ LLM 分类（低优先级）
```

例如标题：

```text
适配劳力士 126610LN / 126610LV 原装风格表带
```

虽然出现两个合法 Rolex reference，但当前商品是“表带”，两个 reference 都应被标成 `COMPATIBLE_REFERENCE`，不能拿去做主商品 entity matching。

---

# 8. 自动匹配决策规则

建议把自动决策写成显式、可版本化的 policy，而不是“score > 0.93”。

## 8.1 Reference Evidence Level

可以定义：

### A 级

- 来源有结构化 reference 字段；
- 通过品牌 grammar；
- role=PRODUCT_REFERENCE；
- 和标题 / OCR 中至少一个独立通道一致；
- 没有冲突候选。

### B 级

- 没有结构化字段；
- title 中高确定性抽取；
- 命中品牌已知 reference dictionary；
- role=PRODUCT_REFERENCE；
- 没有其他冲突 reference。

### C 级

- 仅 title regex 找到，未命中字典；或
- 仅 OCR 找到；或
- LLM 抽取；或
- 同一 record 出现多个竞争候选。

## 8.2 建议策略

```python
def decide_match(left, right):
    if left.brand_id != right.brand_id:
        return "REJECT"

    if not left.reference_canonical or not right.reference_canonical:
        return "ABSTAIN"

    if left.reference_canonical != right.reference_canonical:
        return "REJECT"

    if left.reference_role != "PRODUCT_REFERENCE":
        return "ABSTAIN"

    if right.reference_role != "PRODUCT_REFERENCE":
        return "ABSTAIN"

    if has_reference_conflict(left) or has_reference_conflict(right):
        return "ABSTAIN"

    if is_accessory(left) or is_accessory(right):
        return "ABSTAIN"

    if min(left.ref_evidence_level, right.ref_evidence_level) >= LEVEL_B:
        return "AUTO_LINK"

    return "ABSTAIN"
```

注意：

**图片或 embedding 不应该把 `reference_canonical !=` 的 pair 从 REJECT 翻成 MATCH。**

如果业务定义就是 reference 相同，则 reference 冲突就是硬否决。

---

# 9. 图片应该怎样用

图片有用，但角色必须收敛。

## 9.1 OCR：强辅助

适合提取：

- 表背刻字；
- 保卡；
- 吊牌；
- 盒签；
- 证书。

OCR 输出进入 ReferenceCandidate 集合，再经过 brand grammar + role resolver。

### 建议 OCR 共识机制

一个 record 多张图时，不要只看单张 OCR：

```text
image_1 -> 126610LN
image_2 -> 126610LN
image_3 -> noise
```

如果两个独立图片都识别到同一 canonical reference，可以提升 evidence level。

反之：

```text
title -> 126610LN
OCR   -> 126610LV
```

必须降级为 `ABSTAIN`，不能自动匹配。

## 9.2 Image Embedding：只用于 hard-negative / review ranking

CLIP / SigLIP / DINOv2 等图片 embedding 可以用于：

1. 从同品牌商品里找“外观最像但 reference 不同”的 hard negative；
2. 人工复核队列排序；
3. 在 reference 相同但图像明显冲突时提供 veto signal。

不建议：

```text
图片很像 -> 自动认为是同一 reference
```

腕表同系列不同 reference 往往设计只差尺寸、材质、圈色、盘色或小字，视觉近邻恰恰是最危险的 false positive 来源。

---

# 10. 100 万～1000 万规模下如何避免 O(N²)

如果按照当前业务定义，reference 一旦高置信抽取成功，就根本不需要做全库 pair-wise comparison。

核心索引：

```sql
CREATE TABLE product_record (
    record_id           BIGINT PRIMARY KEY,
    source              SMALLINT NOT NULL,
    source_product_id   TEXT NOT NULL,
    brand_id            INT,
    ref_canonical       TEXT,
    ref_role            SMALLINT,
    ref_evidence_level  SMALLINT,
    parser_version      TEXT,
    updated_at          TIMESTAMP NOT NULL
);

CREATE INDEX idx_product_ref
ON product_record (brand_id, ref_canonical)
WHERE ref_canonical IS NOT NULL;
```

实体表：

```sql
CREATE TABLE product_entity (
    entity_id       BIGSERIAL PRIMARY KEY,
    brand_id        INT NOT NULL,
    ref_canonical   TEXT NOT NULL,
    created_at      TIMESTAMP NOT NULL,
    UNIQUE (brand_id, ref_canonical)
);
```

映射表：

```sql
CREATE TABLE entity_member (
    entity_id   BIGINT NOT NULL,
    record_id   BIGINT NOT NULL,
    decision    TEXT NOT NULL,
    policy_ver  TEXT NOT NULL,
    evidence    JSONB NOT NULL,
    PRIMARY KEY (entity_id, record_id)
);
```

### 增量匹配复杂度

新 record 到达：

```text
parse reference
   ↓
SELECT entity WHERE brand_id=? AND ref_canonical=?
   ↓
0 个：创建新 entity / 暂存
1 个：过 safety gate 后加入
>1 个：数据异常，进入审计
```

数据库索引查找接近 O(log N)，如果用 hash/KV 结构可接近 O(1)。

这样 1000 万记录依然完全可控，不需要生成 1000 万² 的 pair。

### 什么时候才需要 Blocking / ANN

只对：

```text
ref 缺失
ref 低置信
ref 冲突
```

的灰区数据做 candidate retrieval。

而 retrieval 的目的应是：

> 找“可能属于哪个已知 reference”的候选供 extractor / reviewer 判断，
> 而不是绕过 reference 规则直接建立 match。

---

# 11. 基于 WDC Products 构造“腕表版多维 Benchmark”

这是本篇最值得直接复制的部分。

建议建立内部 `WatchMatchBench`。

## 11.1 维度一：Hardness

不是简单 random negative，而是至少分：

```text
H0 random brand/category negative
H1 same brand negative
H2 same brand + same series negative
H3 reference edit-distance <= 2 negative
H4 title embedding TopK negative
H5 image embedding TopK negative
H6 OCR-confusable negative
H7 accessory / compatible-reference negative
```

生产上线时重点看 H3～H7。

### 典型 hard negative

```text
126610LN  vs 126610LV
116500LN  vs 116500  
310.30.42.50.01.001 vs 310.30.42.50.01.002
```

如果模型连这种都不能稳定拒绝，就不应承担自动匹配职责。

## 11.2 维度二：Unseen Reference

切分必须按 **reference/entity**，而不是按 record。

错误方式：

```text
同一个 126610LN 的雷小安记录 -> train
同一个 126610LN 的奢当家记录 -> test
```

这会发生实体泄漏。

正确方式：

```text
train references = A 集合
validation references = B 集合
test references = C 集合
A ∩ B ∩ C = ∅（用于 unseen 子集）
```

建议测试集明确标记：

```text
seen_reference = true / false
```

并分别报告 precision。

## 11.3 维度三：Label Budget

既然当前只有几百对标注，可以做：

```text
50 对
100 对
300 对
500 对
```

分别训练/校准，观察 hard-negative precision 是否稳定。

如果某方法只能在 5000+ 标签时工作，就不适合当前阶段。

---

# 12. 黄金标签应该怎么花

不要把几百个标注随机撒在全量 pair 上。

建议配比：

| 类型 | 占比建议 | 目的 |
|---|---:|---|
| 明确正样本 | 15% | 校验 extraction / normalization |
| 同品牌同系列 hard negative | 30% | 直接压 false positive |
| reference 编辑距离近 | 20% | 防止字符归一错误 |
| 配件/兼容型号 | 15% | 防 reference role 错判 |
| OCR/图片冲突 | 10% | 校验多模态策略 |
| 新品牌/新系列 unseen | 10% | 测分布外鲁棒性 |

最好把每条黄金数据记录成可复现原因，而不仅是 0/1：

```json
{
  "left_id": "...",
  "right_id": "...",
  "label": 0,
  "reason": "SAME_SERIES_DIFFERENT_REFERENCE",
  "left_ref_gold": "126610LN",
  "right_ref_gold": "126610LV"
}
```

这样才能做 failure taxonomy。

---

# 13. 评测指标：不要把 F1 当主指标

WDC Products 用 F1 比较 matcher，这对论文 benchmark 合理；但当前业务的目标函数不同。

生产门禁建议优先级：

```text
1. False Positive Count
2. Precision
3. Precision lower confidence bound
4. Hard-negative precision
5. Unseen-reference precision
6. Coverage / Auto-link rate
7. Recall
8. F1
```

核心指标可以定义为：

```text
AutoLinkPrecision
= TP_auto / (TP_auto + FP_auto)
```

以及：

```text
AutoLinkCoverage
= auto_link_records / all_records
```

precision-first 系统真正的 trade-off 是：

```text
Precision vs Coverage
```

而不是 Precision vs Recall。

### 一个重要的统计现实

几百对黄金标签无法“统计证明 99.99% precision”。

即使在 300 个 auto-link 样本里 0 个错误，也只能说明当前样本没看到错误，并不能证明真实错误率低于万分之一。

所以“绝对不能误匹配”不能靠模型阈值统计保证，而要靠：

- reference exact hard constraint；
- 冲突即拒绝；
- 不确定即 abstain；
- 线上审计；
- 高风险样本人工复核。

模型只提升 coverage，不负责突破安全边界。

---

# 14. 建议的 Hard-Negative Generator

借鉴 WDC 多 similarity 随机混合，而不是只有一种距离：

```python
def generate_hard_negatives(record, catalog, k=20):
    candidates = set()

    # 1. 同品牌同系列
    candidates |= catalog.same_brand_series(record, k)

    # 2. reference 字符串近邻
    candidates |= catalog.ref_edit_distance(record, max_dist=2, k=k)

    # 3. 文本 embedding 近邻
    candidates |= catalog.text_ann(record.title_norm, k=k)

    # 4. 图片 embedding 近邻
    candidates |= catalog.image_ann(record.images, k=k)

    # 5. BM25 lexical 近邻
    candidates |= catalog.bm25(record.title_norm, k=k)

    result = []
    for c in candidates:
        if c.brand_id != record.brand_id:
            continue
        if c.ref_gold == record.ref_gold:
            continue
        result.append((record, c))

    return diversify_by_reason(result)
```

这里 `diversify_by_reason` 很重要，防止 hard negative 全部来自某一类。

每次 parser/model 升级后都应该自动跑这个 hard suite。

---

# 15. 模型应该放在哪个位置

## 15.1 可用模型一：Reference Span Extractor

目标：

```text
title/description -> reference span candidate
```

更推荐 token classification / constrained extraction，而不是 binary pair matcher。

原因是业务真值本身就是 reference，先把中间变量抽准，系统可解释性最高。

## 15.2 可用模型二：Reference Role Classifier

目标：

```text
candidate + surrounding context -> PRODUCT_REFERENCE / SKU / COMPATIBLE_REFERENCE / ...
```

这是降低 false positive 非常高 ROI 的模型。

## 15.3 可用模型三：R-SupCon 风格 Pair Verifier

可以借鉴 WDC repo：

```text
canonical record text -> Transformer -> embedding
```

再用 supervised contrastive learning 让同 reference 靠近、不同 reference 尤其 hard negative 拉远。

但它的用途建议是：

```text
风险评分 / review ranking / veto
```

而不是自动创造 match。

### 输入建议

不要输入全部噪声字段，而是序列化 canonical attributes：

```text
[BRAND] rolex
[REFERENCE] 126610ln
[SERIES] submariner
[TITLE] ...
[CATEGORY] watch
```

### 对比学习样本

positive：

```text
同 canonical reference、不同来源
```

negative 优先：

```text
同 brand + 同 series + 不同 reference
```

这正是 supervised contrastive learning 最适合利用的结构。

---

# 16. 人工复核队列

所有不能自动确定的记录统一进入 `ABSTAIN`，而不是硬猜。

建议 review reason：

```text
NO_REFERENCE
MULTIPLE_REFERENCE_CANDIDATES
TITLE_OCR_CONFLICT
BRAND_CONFLICT
REFERENCE_ROLE_UNKNOWN
ACCESSORY_SUSPECTED
LOW_CONFIDENCE_REFERENCE
NEW_BRAND_PATTERN
```

队列排序可以用：

```text
业务价值 × 出现频率 × 可解决性 × 风险
```

复核结果除了决定单条记录，还应该回流：

```text
brand grammar
reference dictionary
role classifier
span extractor
hard-negative benchmark
```

即人工不是无限重复劳动，而是持续扩大确定性规则覆盖面。

---

# 17. 数据与服务拆分建议

## 17.1 在线链路

可以拆为：

```text
ref-parser-service
reference-index-service
matching-policy-service
review-service
```

### ref-parser-service

输入原始 canonical record，输出 ReferenceCandidate[]。

### reference-index-service

维护：

```text
(brand_id, ref_canonical) -> entity_id
```

### matching-policy-service

纯 deterministic policy + 小量 verifier signal，输出：

```text
AUTO_LINK / REJECT / ABSTAIN
```

### review-service

管理人工队列和 label 回流。

## 17.2 离线链路

```text
raw snapshot
   ↓
Spark / DuckDB / Polars batch parsing
   ↓
reference extraction backfill
   ↓
benchmark generator
   ↓
model training / threshold calibration
   ↓
policy regression test
```

1000 万级并不一定需要复杂大数据栈。如果单条文本不大，Polars/DuckDB + 对象存储就可以做很多批任务；数据继续扩大或图片特征计算很重时再上 Spark。

---

# 18. 版本与审计

建议所有自动决策保存：

```json
{
  "decision": "AUTO_LINK",
  "entity_id": 987,
  "brand_id": "rolex",
  "reference_canonical": "126610LN",
  "left_evidence_level": "A",
  "right_evidence_level": "B",
  "policy_version": "match-policy-v3",
  "parser_version": "ref-parser-v7",
  "verifier_version": "rsupcon-v2",
  "decision_at": "..."
}
```

以后 parser 规则变化时，可以精确找出哪些旧匹配需要重新评估。

这一点在持续增量系统里非常重要：否则新模型一上线，你无法知道历史 800 万条记录哪些受影响。

---

# 19. 上线门禁：直接借鉴 WDC 的多维思路

每次发布新的 parser / matcher / policy，都跑以下矩阵：

| 维度 | 档位 |
|---|---|
| hard-negative ratio | 20 / 50 / 80% |
| unseen reference | 0 / 50 / 100% |
| label budget | small / medium / full |
| source pair | 雷小安×腕表之家 / 雷小安×奢当家 / 腕表之家×奢当家 |
| reference origin | field / title / OCR / mixed |

门禁条件不要只写一个总 precision，而要至少要求：

```text
FP_auto == 0 on curated critical suite
precision_hard_negative >= target
precision_unseen >= target
no regression per source-pair
coverage 不得通过降低安全规则虚假提升
```

如果出现任一 hard false positive，发布直接阻断并加入永久 regression case。

---

# 20. 可以直接开始实施的 MVP

## Phase 1：不训练 matcher，先做 Reference-First

1. 三源 schema mapping；
2. 品牌 canonical dictionary；
3. 20～30 个主要品牌 reference grammar；
4. structured field + title reference extraction；
5. reference role classification 先规则化；
6. `(brand_id, ref_canonical)` exact entity index；
7. 冲突即 abstain；
8. 保存完整 evidence。

这一版往往已经能拿到很高 precision，并建立后续系统的稳定“骨架”。

## Phase 2：WatchMatchBench

用 300～500 对人工标签建立：

- hard negative；
- unseen reference；
- accessory confusion；
- OCR confusion；
- 三源 pair 分层。

以后所有版本都跑 regression。

## Phase 3：OCR + 图片

只做：

- OCR reference evidence；
- title/OCR 冲突否决；
- image hard-negative mining；
- review ranking。

不要让图片相似度独立建立 auto-link。

## Phase 4：R-SupCon / 小模型

当积累一定黄金数据后，再训练：

- span extractor；
- role classifier；
- contrastive verifier。

模型的成功指标不是“替代 exact rule”，而是：

```text
在 AutoLinkPrecision 不下降的前提下，提高 AutoLinkCoverage
```

---

# 21. 一个可操作的在线决策状态机

```text
                 ┌──────────────┐
                 │ 新商品记录     │
                 └──────┬───────┘
                        ▼
               是否抽到 reference？
                 │             │
                否             是
                 │             ▼
                 │       是否唯一候选？
                 │        │          │
                 │       否          是
                 │        │          ▼
                 │        │     role 是否 PRODUCT_REFERENCE？
                 │        │       │               │
                 │        │      否               是
                 │        │       │               ▼
                 │        │       │      brand grammar 是否通过？
                 │        │       │        │             │
                 │        │       │       否             是
                 │        │       │        │             ▼
                 │        │       │        │     是否有跨通道冲突？
                 │        │       │        │       │          │
                 │        │       │        │      是          否
                 │        │       │        │       │          ▼
                 └────────┴───────┴────────┴───────┴────> ABSTAIN
                                                            │
                                                      exact index lookup
                                                            │
                                                   ┌────────┴────────┐
                                                   ▼                 ▼
                                                有实体              无实体
                                                   │                 │
                                                AUTO_LINK        CREATE/WAIT
```

这个状态机最大的优点是每个 false positive 都能追到一个明确 policy branch，而不是只能看到“模型给了 0.97”。

---

# 22. 与 WDC Products 的区别：哪些不能照搬

## 22.1 不直接复制其 label 生成方式

WDC 的 benchmark cluster 初始依赖网页里的 MPN/GTIN/SKU，并且论文对标签做人工抽检仍观察到少量噪声。对当前需求来说，我们已经有更严格的业务定义：**same reference**。

因此内部 gold set 应由人工确认 canonical reference，而不是把来源自己的 identifier 无条件视为真值。

## 22.2 不以 F1 最大化作为生产目标

WDC 是比较 matcher 的科研 benchmark，F1 是合理指标；当前需求是 asymmetric loss：

```text
False Positive Cost >> False Negative Cost
```

因此模型和 policy 都应围绕 precision/coverage 调整。

## 22.3 不把 multi-class recognition 当默认线上方案

论文同时支持 pair-wise 和 multi-class。multi-class 适合“只识别一组已知产品”的场景，但当前系统持续出现新 reference，天然属于 open-world。

因此线上实体键仍建议是：

```text
brand + canonical reference
```

而不是固定分类头中的 class id。

---

# 23. 最终建议

WDC Products 对本需求最值得落地的不是某个单独模型，而是以下四点：

1. **按实体切分 unseen reference 测试集**：避免随机切分造成虚高；
2. **系统性 hard-negative mining**：同品牌同系列、reference 近邻、文本/图片近邻都要进入测试；
3. **少标注条件单独评估**：不假设未来有大量人工标签；
4. **把 benchmark 作为发布门禁**：每次 parser / model / policy 变更都必须过 hard/unseen/source 分层回归。

在生产架构上，则建议进一步收敛为：

> **Reference-first deterministic matching + 多通道 reference extraction + role classification + exact index + conflict abstention + multimodal veto/review + WDC-style multi-dimensional regression benchmark。**

对于“同一个商品 = 同一 reference”这个业务定义，这比做一个端到端“万能多模态 pair matcher”更直接、更便宜、更可解释，也更符合“宁可漏，不能错”的核心约束。

如果只能先做一版，我会优先落地：

```text
品牌归一
→ reference grammar / span extraction
→ reference role 分类
→ canonical reference exact index
→ 冲突拒识
→ WatchMatchBench hard-negative regression
```

等这条主链稳定以后，再用 R-SupCon / OCR / 图片模型提高灰区覆盖率，而不是反过来让模型定义实体身份。

---

## 24. 参考资料

- WDC Products paper: https://arxiv.org/abs/2301.09521
- WDC Products benchmark: https://webdatacommons.org/largescaleproductcorpus/wdc-products/index.html
- WDC Products GitHub: https://github.com/wbsg-uni-mannheim/wdcproducts
- WDC Product Data Corpus: https://webdatacommons.org/largescaleproductcorpus/v2/index.html
- 官方 repo 预处理代码：`src/processing/preprocess/preprocess_wdcproducts.py`
- 官方 repo 对比学习模型：`src/contrastive/models/modeling.py`
- 官方 repo SupCon loss：`src/contrastive/models/loss.py`
