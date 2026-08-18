# Entity Matching with 7B LLMs: A Study on Prompting Strategies and Hardware Limitations：对跨源二奢腕表 Reference 匹配的工程化启示与直接落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选择论文 **Entity Matching with 7B LLMs: A Study on Prompting Strategies and Hardware Limitations** 进行深入分析。

- 论文：<https://ceur-ws.org/Vol-3931/paper4.pdf>
- DOLAP 2025 论文集：<https://ceur-ws.org/Vol-3931/>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

当前 Spec 的关键约束是：

1. 三个来源：雷小安、腕表之家、奢当家；
2. 数据规模约 100 万～1000 万，并持续增量；
3. “同一个商品”定义为**同一 reference number / 型号**；
4. 字段稀疏，reference 可能有独立字段，也可能埋在标题或图片里；
5. **绝对不能误匹配，precision 优先到极致，允许漏匹配**；
6. 可用图片，可人工标注几百对黄金样本。

这篇论文最值得借鉴的并不是“用一个 7B LLM 直接判断两条商品是不是同一个”，而是它给出了两个非常重要的实验结论：

- **属性越多不一定越好。** 在商品 EM 中，仅使用 `model number` 的 atomic domain-specific prompt，通常比把 product name、features、manufacturer、model number 全部混在一起的 composite prompt 更有效，因为 model number 更短、更干净、更有区分度；
- **LLM 天生容易把候选判成 match，导致 false positive。** 论文中几乎所有策略都表现为 recall 明显高于 precision；即使是最好的 7B 模型和 atomic prompt，在更困难的数据集上 precision 仍只有约 0.43，完全不符合当前 Spec 的安全要求。

因此，当前需求不应采用：

```text
record A + record B
        ↓
       LLM
        ↓
   True / False
```

而应采用：

```text
每条商品记录独立解析
        ↓
提取自己的 reference 候选
        ↓
判断候选是不是“当前商品自身 reference”
        ↓
品牌级、collision-safe 规范化
        ↓
Reference Registry 校验
        ↓
得到 VERIFIED_REFERENCE
        ↓
仅对 VERIFIED_REFERENCE 做 exact join
```

最终自动匹配规则建议固定为：

> **只有两条记录都得到 `VERIFIED_REFERENCE`，并且 `(brand_id, canonical_reference)` 完全相同，同时不存在任何冲突证据，才能自动认定为同一商品；其他情况全部拒识或进入人工复核。**

LLM 在系统中最多承担三个角色：

- 从难文本中补充 reference 候选；
- 判断某个候选到底是当前商品自己的 reference，还是平台 SKU、配件适配型号、对比对象等；
- 作为冲突检测器提供“否决/拒识”信号。

**LLM 不应该拥有直接创建 match 的权限。**

---

## 1. 论文实际解决了什么问题

论文把 Entity Resolution 拆成经典的两阶段流程：

```text
Filtering / Blocking
       ↓
Candidate Pairs
       ↓
Verification / Entity Matching
       ↓
Match / Non-match
```

因为全量两两比较是二次复杂度，所以 Blocking 先压缩候选空间，Verification 再对候选对做判断。

论文只重点研究 Verification，也就是：

```text
f(r1, r2) -> {0, 1}
```

其中：

- `1` = 两条记录描述同一实体；
- `0` = 不同实体。

论文将这个二分类问题转成自然语言推理任务，让 LLM 只输出 `True` / `False`。

这个思路对普通商品 Entity Matching 是合理的，但对当前 Spec 只能部分借鉴，因为当前业务已经给出了一个非常强的“身份定义”：**同 reference 才是同商品**。这使得系统不必学习一个模糊的“商品相似性函数”，而可以转化为更安全的：

```text
record -> reference identity resolution
```

这是当前需求可以比论文做得更简单、也更可靠的根本原因。

---

## 2. 论文技术实现与架构

## 2.1 Blocking：先从百万级笛卡尔积缩成小候选集

论文使用 pyJedAI 的 kNN Join 做 Blocking：

```text
Raw records
   ↓
Text cleaning
- stop-word removal
- stemming
   ↓
character n-gram representation
   ↓
cosine similarity
   ↓
kNN Join
   ↓
Candidate pairs
```

两个实验数据集分别使用：

- Abt-Buy：character trigram，`k=4`；
- Walmart-Amazon：character four-gram，`k=2`。

论文把 Blocking 调到至少约 90% blocking recall，再让 LLM 处理剩下的候选对。

其候选压缩效果很明显：

- D1 原始笛卡尔积约 `1.16 × 10^6`，最终仅保留 4,345 个 candidate pairs；
- D2 原始笛卡尔积约 `5.64 × 10^7`，最终仅保留 5,163 个 candidate pairs。

这说明论文架构的工程核心之一并不是“LLM”，而是**先尽量便宜地减小候选空间**。

不过对当前 Spec 可以进一步优化：如果 identity 就是 reference，则大多数记录根本无需 kNN Blocking，可以直接把任务做成单条记录解析，然后按 exact key group/join，复杂度从 pairwise 直接降到近似 O(N)。

---

## 2.2 推理运行环境：7B + 4-bit 量化 + 普通单卡

论文实验栈：

```text
Python 3.12
Ollama 0.1.22
Ubuntu 22.04
Intel i7-9700K
32 GB RAM
NVIDIA GTX 1080 Ti 11 GB
7B LLM + 4-bit quantization
```

论文的意义在于证明：Entity Matching 不一定需要超大闭源模型；7B 量级模型经过 4-bit 量化后，可以在较普通硬件上完成推理。

这一点对当前项目有现实价值：

- 1000 万商品不适合全部调用昂贵云端大模型；
- reference 抽取的大部分 case 可以由规则/字典解决；
- 只把极少量 unresolved / ambiguous 记录送入本地 7B/8B 指令模型，可以显著降低成本；
- 模型可替换，不应让整个匹配系统与某一家大模型绑定。

---

## 2.3 三类 Prompt

论文比较三类核心 prompt。

### 2.3.1 Generic Zero-shot

基本形式类似：

```text
给你两个 record description，判断它们是不是同一实体。
只返回 True 或 False。
```

优点是完全不需要标注；缺点是模型很容易把“相似”误认为“相同”。

论文实验中几乎所有模型都出现：

```text
recall >> precision
```

也就是 match 放得过宽。

对于当前 Spec，这是最危险的行为模式。

### 2.3.2 Few-shot

在 prompt 中加入一个正例和一个负例：

```text
positive example
negative example
query pair
```

论文发现 example order 会造成明显 position bias，所以使用两种顺序：

```text
TF = True example -> False example
FT = False example -> True example
```

然后设计两个 ensemble 策略：

```text
Union:
TF=True OR FT=True -> Match

Intersection:
TF=True AND FT=True -> Match
```

论文结果显示，对 OpenHermes、Zephyr 等模型，Intersection 往往能显著提升 precision，代价是少量 recall。

这与当前需求的 precision-first 非常契合。

但是要特别注意：论文中的 Intersection 仍然是“两个概率性 LLM 判断的一致”，不是身份证据，因此它可以作为**拒识门**，不能成为最终自动匹配依据。

### 2.3.3 Domain-specific Prompt

论文在商品数据上定义四个属性：

1. Product Name
2. Features
3. Manufacturer
4. Model Number

然后比较两个版本。

#### Composite

把四个属性全部交给模型，让模型综合判断。

#### Atomic

只看：

```text
Model Number
```

论文明确说明选择 model number，是因为它具有：

- short
- distinctive
- clean

等特点。

实验中 atomic prompt 在绝大多数模型/数据集上优于 composite prompt，论文将原因归结为：其他长文本属性会引入额外噪声，而 model number 本身已经是强 identity signal。

这几乎就是当前腕表需求的实验佐证。

---

## 3. 论文结果里最重要的不是 F1，而是 False Positive 行为

论文 Table 2 的几个代表性数字：

### Orca2 / D1

```text
Zero-shot:
Precision 0.664
Recall    0.956
F1        0.784

FT Few-shot:
Precision 0.768
Recall    0.834
F1        0.799

Atomic domain-specific:
Precision 0.689
Recall    0.934
F1        0.793
```

### Orca2 / D2

```text
Zero-shot:
Precision 0.397
Recall    0.740
F1        0.517

FT Few-shot:
Precision 0.420
Recall    0.515
F1        0.463

Atomic domain-specific:
Precision 0.434
Recall    0.708
F1        0.538
```

D2 是更接近真实复杂电商数据的场景：

- 商品类型更杂；
- 文本更长；
- 缺失值更多；
- 真正 match 占比更低。

论文明确指出：7B LLM 倾向强调 recall、牺牲 precision，因此在低 match-rate 场景下 false positive 问题尤其严重。

这意味着：

> **如果把论文方案直接部署到当前二奢数据，最危险的情况不是漏掉同款，而是大量把外观相似、同系列、相邻型号商品错误合并。**

这恰好违反当前 Spec 的第一优先级。

因此应当吸收论文的 atomic / intersection 思路，同时舍弃“LLM 输出 True 就匹配”的决策方式。

---

## 4. 当前需求应该被重新定义为 Reference Identity Resolution

当前任务表面上是：

```text
跨源商品实体匹配
```

但既然业务已经规定：

```text
same product <=> same reference number
```

那么生产系统应该把问题改写为：

```text
record
  ↓
这个 record 自己的品牌是什么？
  ↓
它自己的 reference 是什么？
  ↓
这个 reference 是否可信且唯一？
  ↓
canonical identity key 是什么？
```

得到：

```text
identity_key = (brand_id, canonical_reference)
```

然后跨来源匹配只剩下：

```sql
A.brand_id = B.brand_id
AND A.canonical_reference = B.canonical_reference
```

如果业务确认 reference 在全球所有品牌间绝对唯一，可以只用 reference；但从 precision-first 风险控制看，默认使用 `(brand_id, canonical_reference)` 更安全。

---

## 5. 推荐的生产架构

```text
                    ┌────────────────────┐
                    │ 雷小安 / 腕表之家 / 奢当家 │
                    └─────────┬──────────┘
                              │
                              ▼
                 ┌────────────────────────┐
                 │ Source Adapter          │
                 │ 字段映射 / 原始数据保留  │
                 └─────────┬──────────────┘
                           │
                           ▼
                 ┌────────────────────────┐
                 │ Brand Canonicalization │
                 │ 劳力士/ROLEX -> brand_id│
                 └─────────┬──────────────┘
                           │
                           ▼
             ┌──────────────────────────────┐
             │ Reference Candidate Extractor│
             │ 1. 独立 reference 字段       │
             │ 2. brand regex / dictionary │
             │ 3. title / description span │
             │ 4. OCR token                 │
             │ 5. LLM fallback              │
             └────────────┬─────────────────┘
                          │
                          ▼
               ┌─────────────────────────┐
               │ Reference Role Validator│
               │ own ref / platform SKU  │
               │ compatible ref / serial │
               │ comparison ref / unknown│
               └──────────┬──────────────┘
                          │
                          ▼
              ┌───────────────────────────┐
              │ Brand-specific Canonicalizer│
              │ collision-safe normalization│
              └──────────┬────────────────┘
                         │
                         ▼
              ┌───────────────────────────┐
              │ Reference Registry         │
              │ canonical ref 是否存在？   │
              │ alias / pattern / family   │
              └──────────┬────────────────┘
                         │
                         ▼
           ┌───────────────────────────────┐
           │ Evidence & Conflict Gate       │
           │ 独立字段 vs title vs OCR 冲突？│
           │ 是否出现多个 mutually-exclusive ref│
           │ 是否是配件/表带/盒证？          │
           └──────────┬────────────────────┘
                      │
           ┌──────────┴───────────┐
           │                      │
           ▼                      ▼
 VERIFIED_REFERENCE            ABSTAIN
           │
           ▼
 exact key = (brand_id, canonical_reference)
           │
           ▼
 cross-source hash join / group by
           │
           ▼
       Entity Cluster
```

这套架构把概率模型关在 identity resolution 之前，让最终匹配是 deterministic 的。

---

## 6. 数据模型：不要只保存一个 `reference` 字符串

建议至少拆成四张逻辑表。

## 6.1 `product_record`

```sql
CREATE TABLE product_record (
    record_id          BIGINT PRIMARY KEY,
    source             VARCHAR(32) NOT NULL,
    source_item_id     VARCHAR(128) NOT NULL,
    title_raw          TEXT,
    description_raw    TEXT,
    brand_raw          VARCHAR(256),
    reference_raw      VARCHAR(256),
    category_raw       VARCHAR(256),
    image_urls         JSONB,
    payload_raw        JSONB,
    crawled_at         TIMESTAMP,
    updated_at         TIMESTAMP
);
```

原则：**原始值永远保留，不能只存清洗后结果。**

## 6.2 `reference_evidence`

一条 record 可以有多个候选：

```sql
CREATE TABLE reference_evidence (
    evidence_id          BIGSERIAL PRIMARY KEY,
    record_id            BIGINT NOT NULL,
    channel              VARCHAR(32) NOT NULL,
    raw_candidate        VARCHAR(256) NOT NULL,
    normalized_candidate VARCHAR(256),
    role                 VARCHAR(32),
    source_field         VARCHAR(64),
    text_span            TEXT,
    extractor_version    VARCHAR(64),
    model_version        VARCHAR(64),
    confidence           DECIMAL(8,6),
    created_at           TIMESTAMP
);
```

`channel` 示例：

```text
EXPLICIT_FIELD
TITLE_REGEX
DESCRIPTION_REGEX
OCR
LLM
MANUAL
```

`role` 示例：

```text
OWN_REFERENCE
PLATFORM_SKU
SHOP_SKU
SERIAL_NUMBER
COMPATIBLE_REFERENCE
COMPARISON_REFERENCE
ACCESSORY_TARGET_REFERENCE
UNKNOWN
```

这是 precision-first 系统中非常关键的一层。

因为标题：

```text
劳力士 126610LN 原装表带 适配 126610LV
```

如果只做“抓到所有像 reference 的字符串”，很容易错误合并。

真正的问题是：

> 这个编号是不是当前售卖主体自己的 identity？

---

## 6.3 `reference_registry`

```sql
CREATE TABLE reference_registry (
    brand_id              BIGINT NOT NULL,
    canonical_reference   VARCHAR(256) NOT NULL,
    series_id              BIGINT,
    family_id              BIGINT,
    display_reference      VARCHAR(256),
    aliases                JSONB,
    expected_pattern       VARCHAR(256),
    status                 VARCHAR(32) NOT NULL,
    source_provenance      JSONB,
    reviewed_at            TIMESTAMP,
    PRIMARY KEY (brand_id, canonical_reference)
);
```

`status`：

```text
VERIFIED
UNVERIFIED
DEPRECATED
CONFLICT
```

Reference Registry 可以来自：

- 腕表之家结构化型号页；
- 品牌官方目录；
- 可信历史数据；
- 人工确认。

自动匹配只允许使用 `VERIFIED` registry entry。

---

## 6.4 `record_reference_resolution`

```sql
CREATE TABLE record_reference_resolution (
    record_id             BIGINT PRIMARY KEY,
    brand_id              BIGINT,
    canonical_reference   VARCHAR(256),
    resolution_status     VARCHAR(32) NOT NULL,
    decision_reason       VARCHAR(128),
    evidence_snapshot     JSONB,
    resolver_version      VARCHAR(64),
    resolved_at           TIMESTAMP
);
```

`resolution_status`：

```text
VERIFIED_REFERENCE
AMBIGUOUS_REFERENCE
REFERENCE_CONFLICT
NO_REFERENCE
INVALID_REFERENCE
ACCESSORY_OR_NON_WATCH
MANUAL_REVIEW
```

最终 match engine 只读取：

```text
VERIFIED_REFERENCE
```

---

## 7. Reference 抽取：按“便宜且确定”到“昂贵且概率化”逐级降级

不要所有数据一上来就跑 LLM。

推荐：

```text
Tier 0: structured reference field
Tier 1: brand-specific exact pattern
Tier 2: registry alias lookup
Tier 3: OCR-supported extraction
Tier 4: constrained LLM extraction
Tier 5: manual review
```

## 7.1 Tier 0：独立字段

如果来源本身存在 reference 字段：

```text
reference = 126610LN
```

优先级最高，但仍然需要：

- 角色检查；
- 格式检查；
- Registry 检查；
- 与标题/OCR 的冲突检查。

原因是很多平台字段名虽然叫“型号”，实际可能存：

- 平台货号；
- 卖家 SKU；
- 系列名；
- 通用机芯编号；
- 多个 reference 拼接值。

## 7.2 Tier 1：品牌特定 Regex / Lexer

不要用一个全局正则识别所有品牌。

维护：

```python
REFERENCE_PATTERNS = {
    "rolex": [...],
    "omega": [...],
    "cartier": [...],
    "patek_philippe": [...],
}
```

品牌识别后再进入对应 lexer，能大幅降低把普通数字识别成 reference 的概率。

同时给每条抽取保留：

```text
raw span
field
offset
pattern_id
pattern_version
```

这样错误可以追溯。

## 7.3 Tier 2：Registry alias

例如：

```text
126610 LN
126610-LN
126610LN
```

如果 registry 已经确认三者是同一 canonical reference，可以直接归一。

但不要使用“编辑距离最小”自动纠错：

```text
126610LN
126610LV
```

虽然只差一个字符，却是不同型号。

所以：

```text
fuzzy similarity -> candidate only
never -> automatic canonical identity
```

---

## 8. Canonicalization 必须 collision-safe

最危险的实现之一是：

```python
ref = re.sub(r"[^A-Za-z0-9]", "", raw).upper()
```

这很方便，但如果不同 reference 的标点本身有语义，就可能造成 collision。

建议 canonicalizer 返回：

```python
CanonicalizationResult(
    raw="210.30.42.20.01.001",
    canonical="210.30.42.20.01.001",
    brand_id="omega",
    rule_id="omega-v3",
    reversible=True,
    registry_hit=True,
    collision_free=True,
)
```

原则：

1. 品牌特定；
2. 尽量可逆；
3. 必须通过 registry collision check；
4. 新规则上线前跑全量 collision audit；
5. 任何一对旧 canonical key 因新规则被合并，都必须人工审批。

可以维护：

```sql
SELECT canonical_reference, COUNT(DISTINCT raw_reference)
FROM reference_alias
GROUP BY canonical_reference;
```

但“多个 raw alias -> 一个 canonical”不一定有问题，关键是这些 alias 是否已经被验证为同一 reference。

---

## 9. 从论文 Atomic Prompt 改造成 Reference Role Validator

论文的 atomic prompt 思路是：

```text
只看 Model Number
```

当前项目不要让 LLM 判断整条 record pair 是否相同，而是把任务缩小为：

```text
给定：
- brand
- title / description 的局部上下文
- candidate reference

判断：
该 candidate 是否代表“当前售卖商品自己的 reference”？
```

推荐结构化输出：

```json
{
  "candidate": "126610LV",
  "role": "ACCESSORY_TARGET_REFERENCE",
  "is_own_reference": false,
  "has_conflict": false,
  "reason_code": "COMPATIBLE_WITH"
}
```

禁止模型输出任意新 reference：

```text
Candidate 必须来自上游 extractor 或 registry 候选集。
LLM 只能 classify / reject，不能自由生成 identity。
```

这会比“从全文自由生成 reference”安全很多。

---

## 10. 借鉴论文 Intersection：把 LLM 变成双重拒识门

论文指出 few-shot 对 example order 敏感，并通过 TF / FT 两种顺序做 intersection。

当前系统可以更进一步：

### Prompt A：正向确认

```text
这个 candidate 是否明确是当前商品自己的型号？
如果无法确定，返回 UNKNOWN。
```

### Prompt B：反向冲突检查

```text
这个 candidate 有没有证据表明它是：
- 适配型号
- 比较对象
- 平台 SKU
- 配件目标型号
- 其他商品型号
如果任何一种可能成立，返回 CONFLICT。
```

自动通过要求：

```text
A == OWN_REFERENCE
AND B == NO_CONFLICT
```

如果使用 few-shot，还可以再做：

```text
Prompt A(TF) AND Prompt A(FT)
```

形成多重 intersection。

但注意：

```text
LLM unanimity != verified identity
```

LLM 一致只是让记录进入下一层 Registry / conflict gate；不能绕过 exact identity 规则。

---

## 11. 图像的正确角色：提供独立证据，不做视觉相似即匹配

当前有图片，这是很重要的辅助信息。

推荐优先使用：

```text
图片
 ↓
OCR
 ↓
reference-like token
 ↓
品牌 pattern + registry validation
```

重点关注：

- 表背刻字；
- 保卡；
- 吊牌；
- 标签；
- 盒证贴纸；
- 发票/附件中的型号。

但图片相似度绝不能作为自动同款依据。

原因是腕表中最危险的 hard negative 就是：

```text
同系列
相邻 reference
同壳型
同表盘风格
不同机芯/材质/年份/尺寸
```

视觉 embedding 很可能把这些商品排得非常近。

因此图片更适合做：

```text
支持证据
冲突证据
人工复核辅助
```

例如：

```text
title -> 126610LN
OCR   -> 126610LV
```

此时应该：

```text
REFERENCE_CONFLICT -> ABSTAIN
```

而不是“选置信度高的那个”。

---

## 12. 最终 Decision Engine 应该是硬规则，而不是分数加权

不要设计：

```text
0.4 * text_similarity
+ 0.3 * image_similarity
+ 0.2 * llm_score
+ 0.1 * brand_score
> 0.85 -> match
```

因为这种线性/黑盒分数无法满足“绝对不能误匹配”。

建议使用明确 gate：

```python
def resolve(record):
    brand = resolve_brand(record)
    if brand.status != "VERIFIED":
        return ABSTAIN("BRAND_NOT_VERIFIED")

    candidates = extract_reference_candidates(record, brand)

    own_refs = []
    for c in candidates:
        if not valid_brand_pattern(c, brand):
            continue
        if role_validator(record, c) != "OWN_REFERENCE":
            continue
        canon = canonicalize(c, brand)
        if not canon.collision_free:
            continue
        if not registry.is_verified(brand.id, canon.value):
            continue
        own_refs.append(canon.value)

    own_refs = sorted(set(own_refs))

    if len(own_refs) == 0:
        return ABSTAIN("NO_VERIFIED_REFERENCE")

    if len(own_refs) > 1:
        return ABSTAIN("MULTIPLE_REFERENCES")

    if has_cross_channel_conflict(record, own_refs[0]):
        return ABSTAIN("REFERENCE_CONFLICT")

    return VERIFIED(brand.id, own_refs[0])
```

然后 match：

```python
def is_same_product(a, b):
    return (
        a.status == "VERIFIED_REFERENCE"
        and b.status == "VERIFIED_REFERENCE"
        and a.brand_id == b.brand_id
        and a.canonical_reference == b.canonical_reference
    )
```

这应该是系统唯一允许自动建边的函数。

---

## 13. 规模化：1000 万记录不需要做 1000 万 × 1000 万 Pair Matching

如果 identity 是 reference，最省资源的做法是：

```text
N records
 ↓
O(N) reference resolution
 ↓
(key, record_id)
 ↓
hash / sort / group by key
```

例如：

```sql
SELECT
    brand_id,
    canonical_reference,
    ARRAY_AGG(record_id)
FROM record_reference_resolution
WHERE resolution_status = 'VERIFIED_REFERENCE'
GROUP BY brand_id, canonical_reference;
```

然后检查每个 key 下有哪些来源：

```text
(ROLEX, 126610LN)
  - 雷小安: record 123
  - 腕表之家: record 456
  - 奢当家: record 789
```

这天然得到跨源 cluster，不需要做 pairwise classifier。

复杂度更接近：

```text
Extraction: O(N)
Sort / hash group: O(N) 或 O(N log N)
```

远优于：

```text
O(N²)
```

这也是当前 Spec 最大的架构简化机会。

---

## 14. 增量更新设计

三个来源持续增量时，推荐：

```text
Crawler / CDC
   ↓
Raw Event
   ↓
Source Adapter
   ↓
Reference Resolver
   ↓
resolution_key
   ↓
Upsert identity index
   ↓
cluster diff
```

一条商品变化后，不需要全局重跑。

可以只重算：

```text
record_id
旧 identity_key
新 identity_key
```

然后：

1. 从旧 cluster 删除；
2. 加入新 cluster；
3. 如果 status 变成 ABSTAIN，则撤销自动边；
4. 保存 decision version。

重要原则：**匹配结果必须可撤回。**

如果 canonicalization 或 registry 规则更新，系统应该知道哪些记录是由旧 resolver_version 产生的，从而支持定向重算。

---

## 15. 建议的存储与组件选型

不需要一开始搭非常重的架构。

### 第一阶段 / POC

```text
Python
Polars / DuckDB
PostgreSQL
对象存储中的原始 JSON / Parquet
OCR 服务
可替换的 7B/8B 本地模型服务
```

10M 数据做离线 backfill 时，Polars / DuckDB / Spark 任选其一均可；如果现有基础设施已经有 Spark，则直接复用。

PostgreSQL 负责：

- Reference Registry；
- rule/version metadata；
- manual review queue；
- verified resolution；
- lineage。

大体量事实表可以放 ClickHouse / 数据仓库，但不是第一天必须。

### 增量阶段

如果已有消息基础设施：

```text
Kafka
  ↓
Resolver workers
  ↓
PostgreSQL / ClickHouse
```

如果更新频率不是秒级，定时批处理就够，不必为了“实时”过早引入 Kafka/Flink。

---

## 16. Reference Registry 是整个系统的“安全锚点”

没有 Registry 时，系统只能判断：

```text
这个字符串“看起来像” reference
```

有 Registry 后可以判断：

```text
这个字符串是 Rolex 下一个已经确认存在的 reference
```

这两个安全等级差别非常大。

建议 Registry 不只存 canonical value，还存：

```text
brand
canonical_reference
aliases
series
family
valid_format
source provenance
first_seen
last_seen
manual verification
conflicting aliases
```

同时维护负向知识：

```text
known_platform_sku_pattern
known_serial_pattern
known_accessory_pattern
known_fake_reference_pattern
```

这会比不断调 LLM prompt 更有长期价值。

---

## 17. 几百对黄金标签应该怎么花

Spec 可以人工标注几百对。

不建议只随机标：

```text
same / not same
```

因为随机负样本大多数太简单，对 precision-first 帮助很小。

建议构建 hard-negative-first 标注集。

### 17.1 Hard Negative 类型

1. 同品牌、同系列、reference 只差一个字符；
2. 同外观、不同尺寸；
3. 同系列、不同材质后缀；
4. 标题出现“适配 XXX reference”的表带/配件；
5. 同一个 title 同时出现两个 reference；
6. 平台 SKU 很像 reference；
7. OCR 误读一个字符；
8. `0/O`、`1/I/L`、`5/S` 等易混字符；
9. 不同品牌出现相同数字串；
10. 标题和结构化 reference 字段冲突。

### 17.2 标签粒度

一条样本不仅标 `same/not-same`，还应标：

```json
{
  "brand": "ROLEX",
  "own_reference": "126610LN",
  "candidate_spans": [
    {"text": "126610LN", "role": "OWN_REFERENCE"},
    {"text": "126610LV", "role": "COMPARISON_REFERENCE"}
  ],
  "record_type": "WATCH",
  "resolution_status": "VERIFIED_REFERENCE"
}
```

这些标签可以同时训练/评估：

- candidate extractor；
- role validator；
- conflict detector；
- final resolver。

信息利用率远高于单纯 pair label。

---

## 18. 评测指标必须改成 Precision-First

不要以 F1 为发布门槛。

论文以 Precision / Recall / F1 为主要指标，这是学术 EM 的常规做法；当前业务应该把指标改成：

### 一级指标

```text
Auto-Match Precision
False Positive Count
```

### 二级指标

```text
Coverage / Auto-resolution rate
Abstain rate
Manual-review rate
Reference extraction precision
Reference role-classification precision
```

推荐 dashboard：

```text
verified_records
verified_reference_rate
cross_source_cluster_count
auto_match_edges
false_positive_audit_count
conflict_rate
multi_ref_rate
registry_miss_rate
ocr_conflict_rate
llm_fallback_rate
manual_queue_size
```

“绝对 0 false positive”在统计意义上不能靠有限样本证明，因此技术实现必须依赖 deterministic identity gate，而不是单纯依赖模型离线 precision。

---

## 19. 发布策略：宁可先只覆盖最简单的 30%～60%

第一版只自动处理：

```text
brand verified
AND reference candidate unique
AND registry hit
AND explicit field/title evidence一致
AND no accessory context
AND no OCR conflict
```

其他全部：

```text
ABSTAIN
```

等这个集合人工审计稳定后，再逐步放宽：

```text
Phase 1: explicit field exact
Phase 2: title regex exact
Phase 3: registry alias
Phase 4: OCR supporting evidence
Phase 5: LLM-assisted role validation
```

每一步只扩大 coverage，不改变最终 exact key 规则。

---

## 20. 直接可以落地的 P0 / P1 实施计划

## P0：1～2 周，先把安全骨架跑通

### Step 1：统一三源字段

产出：

```text
source_record schema
brand raw value map
reference raw field map
```

### Step 2：做 Brand Registry

至少覆盖 Top 品牌：

```text
raw brand -> brand_id
```

### Step 3：做 Reference Registry

优先导入高频品牌、已有明确型号的数据。

### Step 4：独立字段 + title regex

不用 LLM，先得到第一版：

```text
VERIFIED_REFERENCE
ABSTAIN
```

### Step 5：exact join

直接生成第一版跨来源 cluster。

### Step 6：人工抽样 500～2000 个自动 cluster

重点找 false positive。

如果出现任何一种误合并模式，就优先加 reject rule，而不是调高一个统一相似度阈值。

---

## P1：2～4 周，补齐难 case

### Step 1：Reference Role Validator

训练或 prompt 一个轻量模型，识别：

```text
OWN_REFERENCE
ACCESSORY_TARGET_REFERENCE
PLATFORM_SKU
UNKNOWN
```

### Step 2：加入 OCR

只抽取编号 token，不做视觉 same-product classifier。

### Step 3：加入论文式 Intersection

对于 LLM fallback：

```text
正向 role prompt
AND
反向 conflict prompt
```

两边都通过才继续。

### Step 4：人工 review queue

把以下 case 排高优先级：

```text
两候选冲突
registry miss
高频新 reference
跨来源高价值商品
同系列相邻 reference
```

### Step 5：反馈回流

人工确认后更新：

```text
registry alias
negative pattern
role classifier training set
brand rule
```

---

## 21. 一套可直接实现的 Resolver 伪代码

```python
from dataclasses import dataclass
from enum import Enum

class Status(str, Enum):
    VERIFIED = "VERIFIED_REFERENCE"
    ABSTAIN = "ABSTAIN"
    CONFLICT = "REFERENCE_CONFLICT"

@dataclass
class Resolution:
    status: Status
    brand_id: str | None = None
    reference: str | None = None
    reason: str | None = None


def resolve_reference(record, services) -> Resolution:
    brand = services.brand_resolver.resolve(record)
    if not brand.verified:
        return Resolution(Status.ABSTAIN, reason="BRAND_UNVERIFIED")

    candidates = services.extractor.extract(record, brand)

    accepted = []
    rejected = []

    for c in candidates:
        if not services.patterns.valid(brand.id, c.raw):
            rejected.append((c, "INVALID_PATTERN"))
            continue

        role = services.role_validator.classify(record, c)
        if role != "OWN_REFERENCE":
            rejected.append((c, role))
            continue

        canon = services.canonicalizer.normalize(brand.id, c.raw)
        if not canon.collision_safe:
            rejected.append((c, "CANONICALIZATION_RISK"))
            continue

        if not services.registry.is_verified(brand.id, canon.value):
            rejected.append((c, "REGISTRY_MISS"))
            continue

        accepted.append(canon.value)

    accepted = sorted(set(accepted))

    if len(accepted) == 0:
        return Resolution(Status.ABSTAIN, brand.id, reason="NO_VERIFIED_REF")

    if len(accepted) > 1:
        return Resolution(Status.CONFLICT, brand.id, reason="MULTIPLE_OWN_REFS")

    ref = accepted[0]

    if services.conflict_detector.has_conflict(record, brand.id, ref):
        return Resolution(Status.CONFLICT, brand.id, ref, "CROSS_CHANNEL_CONFLICT")

    return Resolution(Status.VERIFIED, brand.id, ref)
```

Match 逻辑保持不可扩展的硬约束：

```python
def can_auto_match(x: Resolution, y: Resolution) -> bool:
    if x.status != Status.VERIFIED:
        return False
    if y.status != Status.VERIFIED:
        return False

    return (
        x.brand_id == y.brand_id
        and x.reference == y.reference
    )
```

“不可扩展”是刻意的：不要让后续开发为了 coverage 在这个函数里加 image similarity、title similarity、LLM score 等旁路。

---

## 22. 建议增加一条系统级安全 invariant

在代码和数据层都定义：

```text
INVARIANT:
No automatic entity edge may exist
without a shared VERIFIED_REFERENCE identity key.
```

可以在数据库层做审计：

```sql
SELECT e.*
FROM entity_edge e
LEFT JOIN record_reference_resolution a
  ON e.left_record_id = a.record_id
LEFT JOIN record_reference_resolution b
  ON e.right_record_id = b.record_id
WHERE e.created_by = 'AUTO'
  AND NOT (
      a.resolution_status = 'VERIFIED_REFERENCE'
      AND b.resolution_status = 'VERIFIED_REFERENCE'
      AND a.brand_id = b.brand_id
      AND a.canonical_reference = b.canonical_reference
  );
```

这个查询必须永远返回 0 行。

这比“模型 precision 99.x%”更接近当前业务真正需要的安全保证。

---

## 23. 论文方案中不建议直接采用的部分

### 23.1 不要直接使用 pairwise LLM binary classifier 做最终匹配

论文已经证明，即使较优策略也有明显 false positive 倾向。

### 23.2 不要把更多字段塞给 LLM 期待更准

论文的 atomic model-number prompt 反而通常优于 composite prompt，说明 name/features 等噪声会干扰判断。

### 23.3 不要只看 F1

对当前系统：

```text
Precision 0.99 + Recall 0.95
```

都可能仍然不可接受，因为千万级数据下 1% false positive 是灾难。

### 23.4 不要依赖图片视觉相似度补 reference 不确定性

图像可以验证冲突，但不应越过 reference identity。

### 23.5 不要自动 fuzzy-correct reference

一个字符差异很可能就是另外一个真实型号。

---

## 24. 与论文相比，当前系统真正应该采用的“Atomic”定义

论文中的 atomic：

```text
pair A.model_number == pair B.model_number ?
```

当前系统更安全的 atomic 定义应该是：

```text
record
 ↓
唯一、可信、属于当前商品自身的 canonical reference
```

然后：

```text
same_product(A, B)
=
A.verified_reference_key == B.verified_reference_key
```

换句话说，论文告诉我们：**不要让模型被过多商品属性干扰。**

当前 Spec 可以进一步推进为：**连“商品相似性”本身都不要建模，只解析身份键。**

---

## 25. 最终推荐方案

如果现在就要开始实施，我建议按下面优先级：

```text
1. Brand Registry
2. Reference Registry
3. Brand-specific reference parser
4. Collision-safe canonicalizer
5. Reference role classifier
6. Conflict detector
7. VERIFIED_REFERENCE 状态机
8. exact key cross-source join
9. OCR supporting evidence
10. 7B/8B LLM fallback + intersection reject gate
```

其中 1～8 就能做出一版生产可用系统；9～10 主要提升 coverage。

论文的 7B LLM 并不是主引擎，而更适合放在第 10 层解决极少量难 case。

这会把整体成本和风险控制在一个非常合理的范围：

```text
大多数记录：规则 / registry -> 毫秒级
少数困难记录：OCR / LLM
所有自动匹配：exact key
所有不确定记录：ABSTAIN
```

对当前“precision 优先到极致”的业务约束，这是比直接复刻论文 EM classifier 更合适的落地路径。

---

## 26. 一句话总结

**这篇论文最重要的启示是：商品实体匹配中，短、干净、区分度高的 model number 比混合大量语义属性更可靠；但 7B LLM 仍然会产生大量 false positive。因此当前二奢腕表系统应把 LLM 从“匹配决策器”降级为“reference 候选/角色/冲突辅助器”，最终只允许经过 Registry 验证的 `(brand_id, canonical_reference)` exact match 自动建边，并对其他全部情况主动拒识。**
