# Query Brand Entity Linking in E-Commerce Search：从品牌实体链接迁移到“Reference-first”二奢腕表匹配的落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选择此前 `奢侈品调研/a` 尚未分析的论文：

**Query Brand Entity Linking in E-Commerce Search**

- 论文：<https://arxiv.org/abs/2502.01555>
- 作者：Dong Liu, Sreyashi Nag（Amazon）
- PECOS 官方实现：<https://github.com/amzn/pecos>
- PECOS 论文：<https://jmlr.org/papers/v23/21-0085.html>
- 当前需求 Spec：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

这篇论文非常适合当前需求，不是因为“品牌”与“腕表 Reference”完全相同，而是因为它解决了一个高度同构的工程问题：

> **从短、脏、上下文不足的文本中识别一个 mention，再把它链接到一个超大、长尾、持续变化的 canonical entity 空间；当名称存在别名、一对多歧义时，用侧信息消歧；同时将高精度的精确匹配与高覆盖的语义模型做分层融合。**

对当前 Spec，最重要的迁移结论是：

1. **Reference Number 必须先被建模成 canonical entity，而不是直接做跨平台 record-pair 二分类。** 三个平台的商品记录都先链接到 `reference_entity_id`，最终“是否同商品”退化为 `reference_entity_id` 是否相同。
2. **论文里的 `NER + Exact Lexical Match` 应成为主干，而不是 baseline。** 论文实验中，精确词典路线的 precision 明显高于纯语义 PECOS；当前业务又明确要求“绝对不能误匹配”，所以语义模型只能做召回/候选生成，不能拥有自动合并权限。
3. **论文的一对多歧义处理非常值得直接照搬：如果侧信息过滤后仍有多个 entity，不预测。** 对腕表就是：同一个规范化字符串若仍可能对应多个 Reference，直接 `ABSTAIN`，而不是挑最高分。
4. **把论文的 Product Type Filter 改造成 `Brand + ProductRole + Series/Family + Source` 约束。** “Role”尤其重要：标题中的字符串可能是腕表 Reference，也可能是平台 SKU、库存号、序列号、表带适配型号、盒证编号。
5. **把论文的 NO_ENTITY 类改成 `NO_REFERENCE / ABSTAIN`。** 系统必须把“不知道”作为一级输出，而不是异常。
6. **语义模型可用 PECOS/XMC，但只能在 exact path 失败后提供 Top-K Reference 候选。** 最终自动接受仍必须满足可审计的硬证据，例如 canonical reference 唯一一致、独立来源证据一致且无冲突。
7. **图片不是“像不像”的自动同款依据。** 图片 OCR 可作为 Reference 的第二证据源，也可作为冲突否决信号；视觉 embedding 最多做候选排序/人工复核辅助。

因此，我建议直接落地一个：

> **Reference Registry + Conservative Extractor + Source-scoped Exact Linker + Side-info Disambiguator + PECOS Candidate Generator + Hard-gated Decision Engine**

系统核心不是“训练一个更强的 matcher”，而是**建立一个有拒识能力、证据可追溯、语义模型无越权权限的 Reference Entity Linking 系统**。

---

## 1. 论文到底解决了什么问题

论文研究的是电商搜索中的 Brand Entity Linking。输入通常是很短的查询词，平均约 2.4 个词，文本不具备完整自然语言结构，同时品牌实体空间巨大、长尾明显、新品牌持续加入。

论文把问题拆成三类方案：

### 1.1 两阶段：NER + Exact Lexical Matching

第一阶段用 MetaTS-NER 从 query 中抽取 brand mention；第二阶段把 mention 放入静态词典查找 canonical brand entity。

词典并不是简单的 `brand_name -> entity`：

- 同一 entity 可以有多个 surface form，例如全称、简称、别名；
- 一些名称是 store-specific，因此 key 可以带 store tag；
- 相同名称可能映射到多个 brand entity，因此要额外消歧；
- 如果消歧后仍然不能唯一确定，论文选择**不做预测**，以保证高精度。

这一点与当前腕表需求几乎一一对应：

- 一个 Reference 可以有多个写法：空格、连字符、大小写、前后缀、品牌习惯写法；
- 某些平台把自己的 SKU、货号与 Reference 混在一起；
- 同一个短字符串在不同品牌/系列/来源中可能含义不同；
- 当证据不足时，业务允许漏匹配，因此应该明确拒识。

### 1.2 两阶段语义版：NER + M2E-PECOS

论文用 PECOS 替换 exact dictionary matcher，把抽取出来的 brand mention 直接分类到超大 brand entity 空间。

PECOS 的关键不是“深度语义”本身，而是它解决了 **extreme classification / extreme ranking**：

- 标签数很大；
- 标签是长尾分布；
- 很多标签训练样本很少；
- 需要低延迟 Top-K 推理。

PECOS 的核心流程可以概括为：

1. 用 label embedding 表示输出标签；
2. 对标签做层次聚类，形成 hierarchical label tree；
3. 训练递归 matcher，从树根逐层缩小候选空间；
4. 最后在很小的叶子候选集合里做排序。

PECOS 官方 X-Linear 实现支持稀疏 TF-IDF 特征、层次标签树和 C++ 快速推理，适合把几十万乃至更多 Reference 当作 label 空间。

但论文的实验结果给当前业务一个非常重要的警告：

> **语义匹配能提高 coverage，但会损失 precision。**

因此不能因为 PECOS 很快、很强，就把它变成最终裁决器。

### 1.3 一阶段：Q2E-PECOS

论文还提出直接把完整 query 分类到 brand entity，不再先做 NER。这解决了两阶段误差传递和 NER recall 上限，也能捕捉没有显式品牌字符串但由产品线暗示品牌的情况。

论文还专门增加了 `NO_ENTITY` 类，让模型可以对不含品牌意图的 query 输出“无品牌实体”。

迁移到腕表后，这一思想可以保留，但必须改成：

- `NO_REFERENCE`
- `AMBIGUOUS_REFERENCE`
- `CONFLICT`
- `CANDIDATE_ONLY`

而不是只有一个“有/无 Reference”的二值分类。

### 1.4 Fusion：精确结果优先于语义结果

论文最终把 exact path 与 PECOS path 并行运行：

- exact 能给结果时，优先 exact；
- exact 没覆盖到时，语义模型补 coverage。

实验中，`NER + Exact Lexical Match` 的 precision 在两个 store group 上分别达到约 **97.22% / 99.15%**，且在 85K 非品牌 query 上 false alarm 约 **1.177%**；语义方案覆盖更广，但误报更高。

对于普通搜索系统，这样的精度可能已经很好；但当前 Spec 的约束是“绝对不能误匹配”，所以还要再收紧一层：

> **论文里的 Fusion 是“精确优先”；当前系统应该是“精确硬门禁 + 语义仅候选”。**

也就是说，PECOS 不应像论文那样在 exact 缺失时直接补最终预测，而应只给候选 Reference，交给后续 hard validator。

---

## 2. 论文最值得迁移的五个技术点

## 2.1 Canonical Entity Registry，而不是 Pairwise Match

论文不是直接判断两个 query 是否同品牌，而是把 query 链接到 canonical brand entity。

对当前系统应采用同样结构：

```text
雷小安 record ─┐
腕表之家 record ├──> reference_entity_id
奢当家 record ─┘
```

最终：

```text
same_product(record_a, record_b)
    := record_a.reference_entity_id == record_b.reference_entity_id
```

这样可以把原本潜在的 O(N²) 跨源 pairwise 问题改成每条记录独立做一次 Entity Linking。

对 100 万～1000 万记录，这是架构上最重要的降维。

## 2.2 Source-scoped Alias

论文会给 store-specific surface form 加 store tag。

腕表数据同样应该支持：

```text
(source_id, brand_id, alias_norm) -> reference_entity_id
```

而不是只有：

```text
alias_norm -> reference_entity_id
```

原因包括：

- 某平台自定义字段可能把 `型号` 实际填成内部款号；
- 某来源标题模板可能固定加入前缀/后缀；
- 平台 SKU 格式可能恰好与某品牌 Reference 相似；
- 同一字符串在不同来源里的可靠性不同。

Source scope 可以显著降低跨平台误解释。

## 2.3 Side Information Disambiguation

论文用 Query Product Type 与 Brand 的已知 Product Type 集合做消歧。

当前可映射成以下硬过滤链：

```text
Brand
  -> ProductRole（腕表/表带/盒证/配件/零件）
    -> Series / Family
      -> Case Size / Material / Movement（可选）
        -> Reference candidate
```

其中优先级应是：

1. 品牌冲突：直接否决；
2. 商品角色冲突：直接否决；
3. 系列冲突：默认否决或人工；
4. 其他属性冲突：作为强负证据；
5. 图片视觉不相似：只能辅助，不直接决定 Reference。

如果硬过滤后仍有多个候选：

```text
ABSTAIN
```

不要 `argmax(score)`。

## 2.4 NO_ENTITY / Abstention 是产品能力，不是模型失败

论文显式增加 `NO_ENTITY`，说明它把“无可链接实体”当成合法标签。

当前系统更应该把拒识细分：

```text
NO_REFERENCE        没发现可信 Reference
AMBIGUOUS           有多个无法排除的候选
CONFLICT            不同证据源给出不同 Reference
CANDIDATE_ONLY      只有语义候选，没有硬证据
AUTO_LINK           满足自动链接硬门槛
```

这比简单输出 `matched=true/false` 更适合 precision-first。

## 2.5 Strong + Weak + Dictionary 三类训练数据

论文训练数据包含：

- brand-name2entity 字典数据；
- 人工 strong labels；
- 从搜索行为构造的 weak labels。

当前可以直接映射为：

### Dictionary / Synthetic Positive

从已确认的 Reference Registry 自动构造：

```text
Rolex | 126610LN
Rolex | 126610 LN
Rolex | Ref.126610LN
Rolex | 型号126610LN
...
```

注意：只生成**保证等价**的格式扰动，不允许随意删除所有标点。

### Strong labels

用户允许人工标注几百对，应该集中标注：

- 同系列相邻 reference；
- 只有一个字符不同；
- 标题出现“适配/for/compatible with”的 reference；
- SKU 与 reference 格式相似；
- OCR 容易混淆 `0/O`、`1/I`、`5/S`；
- 同名/别名品牌；
- 同一商品里同时出现当前款与历史款 reference。

这些 hard cases 比随机样本更值钱。

### Weak labels

可以来自：

- 平台结构化 `reference/model` 字段与标题抽取完全一致；
- 同一来源同一商品多次快照稳定出现同 Reference；
- 标题与 OCR 独立抽取一致；
- 已有人工确认 cluster 的高置信同义写法。

Weak labels 可以训练 candidate generator，但不要直接升级成自动合并规则。

---

# 3. 当前 Spec 与论文问题的关键差异

论文解决的是“品牌语义实体”，当前解决的是“Reference Identifier 实体”。这决定了不能机械照抄。

## 3.1 Reference 的字符串结构比品牌更强

Reference 本质上是厂商定义的 identifier，通常具有高信息密度的字母数字结构。

因此应该充分利用：

- 字符位置；
- 字符串长度；
- 合法字符集；
- 品牌特定 pattern；
- 前后缀语义；
- 分隔符规则；
- 系列内部邻近编号。

这类信号对品牌实体链接不一定重要，但对腕表 Reference 是最核心的 precision 信号。

## 3.2 “像”不能代表“相同”

同系列腕表可能：

```text
126610LN
126610LV
```

文本和图片都极其相似，但它们就是不同 Reference。

所以：

- 编辑距离很小不是正证据；
- embedding 很近不是正证据；
- 图片很像不是正证据；
- 同系列不是正证据。

这些最多用于构造 hard negative。

## 3.3 Reference normalization 必须保守

不能使用粗暴规则：

```python
re.sub(r"[^A-Z0-9]", "", ref)
```

然后把得到的字符串直接当成全局唯一 canonical key。

原因是不同品牌可能把 `/`、`.`、`-`、空格、后缀作为有意义结构。

正确方法是保留多层规范化结果：

```text
raw_ref
surface_norm
brand_rule_norm
canonical_ref
```

其中只有 `canonical_ref` 可以用于自动跨源合并。

`surface_norm` 只能用于候选召回。

---

# 4. 建议的目标架构

```text
┌─────────────────────────────────────────────────────────────┐
│  雷小安 / 腕表之家 / 奢当家：全量 + 增量                  │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Canonical Record Layer                                  │
│  source / source_id / title / attrs / images / raw payload  │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Evidence Extractors                                     │
│  structured field / title / description / image OCR         │
│  + identifier role classifier                               │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Conservative Normalizer                                 │
│  brand-aware parser + source-aware rules                    │
└──────────────┬──────────────────────────────┬───────────────┘
               │                              │
               ▼                              ▼
┌────────────────────────────┐   ┌────────────────────────────┐
│ 4A. Exact Alias Registry   │   │ 4B. PECOS Candidate Gen   │
│ source+brand+alias -> ref  │   │ text -> Top-K ref entity  │
└──────────────┬─────────────┘   └──────────────┬─────────────┘
               │                                │
               └──────────────┬─────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Side-info / Conflict Validator                          │
│  brand / role / series / source / OCR / structured evidence │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Decision Engine                                         │
│  AUTO_LINK / REVIEW / ABSTAIN / CONFLICT / NO_REFERENCE     │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Reference Entity Graph                                  │
│  record_id -> reference_entity_id                           │
└─────────────────────────────────────────────────────────────┘
```

这套架构最大特点是：

> **模型不直接修改实体关系，所有模型输出都要经过 Decision Engine。**

---

# 5. Reference Registry：系统真正的核心资产

建议建立以下核心表。

## 5.1 `reference_entity`

```sql
CREATE TABLE reference_entity (
    reference_entity_id BIGSERIAL PRIMARY KEY,
    brand_id            BIGINT NOT NULL,
    canonical_ref       TEXT NOT NULL,
    series_id           BIGINT NULL,
    product_role        TEXT NOT NULL DEFAULT 'WATCH',
    status              TEXT NOT NULL DEFAULT 'ACTIVE',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (brand_id, canonical_ref)
);
```

`brand_id` 必须参与唯一键。不要假设 Reference 字符串跨品牌全局唯一。

## 5.2 `reference_alias`

```sql
CREATE TABLE reference_alias (
    alias_id             BIGSERIAL PRIMARY KEY,
    reference_entity_id  BIGINT NOT NULL,
    brand_id             BIGINT NOT NULL,
    source               TEXT NULL,
    alias_raw            TEXT NOT NULL,
    alias_norm           TEXT NOT NULL,
    alias_type           TEXT NOT NULL,
    trust_level          SMALLINT NOT NULL,
    provenance           JSONB NOT NULL,
    enabled              BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_reference_alias_lookup
ON reference_alias(brand_id, source, alias_norm)
WHERE enabled = TRUE;
```

`alias_type` 推荐：

```text
CANONICAL
SAFE_FORMAT_VARIANT
SOURCE_SPECIFIC
HUMAN_CONFIRMED
OCR_CONFIRMED
MODEL_SUGGESTED
```

其中 `MODEL_SUGGESTED` 默认不能进入自动 exact path，必须人工升级后才能成为可信 alias。

## 5.3 `reference_evidence`

```sql
CREATE TABLE reference_evidence (
    record_id            BIGINT NOT NULL,
    evidence_type        TEXT NOT NULL,
    raw_value            TEXT,
    normalized_value     TEXT,
    source_field         TEXT,
    extractor_version    TEXT,
    confidence           DOUBLE PRECISION,
    bbox                  JSONB,
    image_id              TEXT,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

关键是保留 provenance。以后发现误规则时，可以反查是哪一个 extractor / alias / source field 导致的。

## 5.4 `record_reference_link`

```sql
CREATE TABLE record_reference_link (
    record_id             BIGINT PRIMARY KEY,
    reference_entity_id   BIGINT NULL,
    decision              TEXT NOT NULL,
    decision_reason       TEXT NOT NULL,
    decision_version      TEXT NOT NULL,
    evidence_snapshot     JSONB NOT NULL,
    decided_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`decision` 取值：

```text
AUTO_LINK
REVIEW
ABSTAIN
CONFLICT
NO_REFERENCE
```

---

# 6. Reference 抽取器：先识别“这是什么编号”，再识别“它是哪一个 Reference”

当前系统最容易犯的严重错误，不一定是抽不到 Reference，而是把别的编号当 Reference。

因此建议把任务分为两步：

```text
identifier span detection
        ↓
identifier role classification
        ↓
reference entity linking
```

## 6.1 Role 分类至少包含

```text
REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
INVENTORY_ID
ACCESSORY_COMPATIBLE_REFERENCE
CERTIFICATE_NUMBER
UNKNOWN_IDENTIFIER
```

特别是：

```text
“适配劳力士 126610LN 表带”
```

里面出现的 `126610LN` 并不是当前商品本身的 Reference，而是兼容对象。

如果只做字符串抽取 + exact match，会产生非常危险的 false positive。

## 6.2 结构化字段需要 field trust policy

对每个平台建立字段可靠性配置：

```yaml
source: leidangjia
fields:
  reference_no:
    role: REFERENCE
    trust: HIGH
  sku:
    role: PLATFORM_SKU
    trust: NEVER_REFERENCE
  title:
    role: FREE_TEXT
    trust: MEDIUM
```

不是所有名为 `model` / `型号` 的字段都自动可信。第一次接入来源时要抽样验证。

---

# 7. Conservative Normalizer

建议 normalization 分层。

```python
@dataclass
class NormalizedReference:
    raw: str
    surface_norm: str
    brand_rule_norm: str | None
    canonical_ref: str | None
    rules_applied: list[str]
```

## 7.1 `surface_norm`

只做明显安全的文本级规范化：

- Unicode NFKC；
- 全角转半角；
- 大小写统一；
- 首尾空白；
- 连续空白折叠；
- 常见 `Ref.` / `Reference` / `型号` 标签从 span 外移除。

不要立即删除所有 `- / .`。

## 7.2 `brand_rule_norm`

使用品牌专属 parser：

```python
def normalize_reference(brand_id, raw):
    x = safe_surface_normalize(raw)
    parser = brand_parsers.get(brand_id)
    if not parser:
        return NormalizedReference(raw, x, None, None, ["surface"])

    parsed = parser.parse(x)
    if not parsed.is_valid:
        return NormalizedReference(raw, x, parsed.norm, None, parsed.rules)

    return NormalizedReference(
        raw=raw,
        surface_norm=x,
        brand_rule_norm=parsed.norm,
        canonical_ref=parsed.canonical_if_known,
        rules_applied=parsed.rules,
    )
```

关键点：

> parser 可以产生“候选规范形式”，但只有 Registry 已确认的 canonical key 才能成为自动链接依据。

---

# 8. Exact Linker：把论文 baseline 升级为生产主链

论文的 exact lexical matcher 对当前需求应该成为最重要的自动通道。

建议 key：

```text
(brand_id, source, alias_norm)
```

查不到 source-specific 时再 fallback：

```text
(brand_id, NULL, alias_norm)
```

伪代码：

```python
def exact_link(record, evidence):
    brand_id = record.brand_id
    if brand_id is None:
        return Abstain("NO_CANONICAL_BRAND")

    refs = []

    for ev in evidence:
        if ev.role != "REFERENCE":
            continue

        candidates = alias_registry.lookup(
            brand_id=brand_id,
            source=record.source,
            alias_norm=ev.brand_rule_norm or ev.surface_norm,
        )

        refs.extend((ev, c) for c in candidates)

    return resolve_exact_candidates(record, refs)
```

## 8.1 唯一 exact candidate 也不能无脑通过

还要做：

```text
brand consistency
product role consistency
negative context check
source field trust
conflicting evidence check
```

例如标题：

```text
“原装表带 适配 126610LN”
```

即使 alias 唯一，也必须因为 `product_role=ACCESSORY` 而拒绝链接到腕表 Reference 实体。

---

# 9. Side-info Disambiguator：把论文 Product Type Filter 改造成多级硬过滤

论文在 brand name 映射多个 entity 时，用 Product Type 做过滤，并且过滤后仍多解则不预测。

当前可以实现：

```python
def hard_filter(record, candidates):
    out = candidates

    out = [c for c in out if c.brand_id == record.brand_id]

    if record.product_role:
        out = [c for c in out if c.product_role == record.product_role]

    if record.series_id:
        compatible = [c for c in out if c.series_id in (None, record.series_id)]
        if compatible:
            out = compatible

    if len(out) != 1:
        return None

    return out[0]
```

这里故意没有：

```python
return max(out, key=lambda c: c.score)
```

因为对当前业务，“多个候选中最高分”不满足安全要求。

---

# 10. PECOS：只做 Candidate Generator

## 10.1 为什么还值得用

当 Reference 不在明确结构化字段里，而是埋在标题、描述、OCR 甚至格式被污染时，exact path 会漏掉。

这时 PECOS 的优势是：

- label 空间可以很大；
- 长尾 label 可利用 label correlation；
- 支持稀疏特征；
- 层次树缩小搜索空间；
- 可低延迟 Top-K。

当前 label 定义：

```text
label := reference_entity_id
```

instance text 可拼接：

```text
[BRAND] Rolex
[SOURCE] leixiaoan
[ROLE] WATCH
[TITLE] ...
[REF_SPANS] 126610 ln ...
[SERIES] Submariner
[OCR] ...
```

但为了减少模型学到不稳定的营销语义，建议第一版 PECOS 输入以**标识符特征**为主：

- char 2~5 grams；
- word TF-IDF；
- brand token；
- source token；
- series token；
- role token；
- extractor 输出的 identifier spans。

不要一开始就把完整图片 embedding 和大模型 embedding 全塞进去。

## 10.2 X-Linear 就足够做第一版

PECOS 官方示例结构：

```python
from pecos.xmc.xlinear.model import XLinearModel
from pecos.xmc import Indexer, LabelEmbeddingFactory

label_feat = LabelEmbeddingFactory.create(Y, X)
cluster_chain = Indexer.gen(label_feat)
model = XLinearModel.train(X, Y, C=cluster_chain)
```

当前可以构造：

```text
X: record / mention 的稀疏 TF-IDF + char ngram feature
Y: reference_entity_id one-hot / multi-hot
```

预测：

```text
TopK(reference_entity_id, score)
```

然后交给 hard validator。

## 10.3 不允许 semantic-only AUTO_LINK

决策规则必须明确写进代码：

```python
if candidate.origin == "PECOS_ONLY":
    return REVIEW_OR_ABSTAIN
```

即使：

```text
score = 0.9999
```

也不能自动链接。

原因不是“不相信模型”，而是 score 不是业务定义的逻辑证明。

---

# 11. 图片怎么接入

当前 Spec 说有图片可用，但图片应以 evidence 方式加入，而不是把视觉相似度变成主 matcher。

## 11.1 OCR 是最有价值的图片通道

优先检测：

- 表背刻字；
- 保卡；
- 吊牌；
- 盒标；
- 发票/鉴定卡局部。

OCR 结果重新走：

```text
identifier extraction -> role classification -> normalization -> registry lookup
```

如果：

```text
标题 Reference == OCR Reference
```

这是两个独立 evidence channel 的一致性，可显著提高可接受度。

如果：

```text
标题 Reference != OCR Reference
```

默认进入：

```text
CONFLICT
```

而不是对二者做 score averaging。

## 11.2 Visual embedding 只做辅助

可用于：

- 人工 review 排序；
- 发现“型号字符串正确但商品角色明显不对”的异常；
- 候选召回；
- cluster QA。

不能用于：

```text
视觉非常相似 => AUTO_LINK
```

因为同系列相邻 Reference 往往就是高视觉相似 hard negative。

---

# 12. Decision Engine：真正决定能不能自动合并

建议用确定性规则，不要把所有特征再交给一个黑盒二分类器。

## 12.1 推荐决策表

| 场景 | 结果 |
|---|---|
| 高可信结构化字段给出 canonical ref，brand 一致，无冲突 | `AUTO_LINK` |
| 标题抽取 unique exact alias + source/brand/role 全通过 | `AUTO_LINK` 或按来源策略收紧 |
| 标题 exact + OCR exact 且一致 | `AUTO_LINK` |
| exact alias 一对多，硬过滤后唯一 | `AUTO_LINK` |
| exact alias 一对多，过滤后仍多解 | `ABSTAIN` |
| PECOS Top-1 很高但无 exact evidence | `REVIEW` |
| PECOS Top-K + 一个弱 exact 证据 | `REVIEW`，直到弱证据被确认规则化 |
| 不同 evidence 给出不同 ref | `CONFLICT` |
| 只发现 SKU / serial / compatible ref | `NO_REFERENCE` |
| 没有 Reference 证据 | `NO_REFERENCE` |

## 12.2 伪代码

```python
def decide(record, evidence, exact_candidates, pecos_candidates):
    conflicts = detect_conflicts(evidence)
    if conflicts:
        return Decision("CONFLICT", conflicts)

    exact = strict_exact_resolve(record, exact_candidates)
    if exact.is_unique and exact.all_hard_checks_pass:
        return Decision(
            "AUTO_LINK",
            reference_entity_id=exact.ref_id,
            reason=exact.audit_reason,
        )

    if exact.is_ambiguous:
        return Decision("ABSTAIN", reason="MULTIPLE_EXACT_CANDIDATES")

    if pecos_candidates:
        return Decision(
            "REVIEW",
            reason="SEMANTIC_CANDIDATE_ONLY",
            candidates=pecos_candidates[:5],
        )

    return Decision("NO_REFERENCE", reason="NO_VALID_REFERENCE_EVIDENCE")
```

---

# 13. 跨源“同商品”不再需要训练 Pair Matcher

一旦完成 record -> reference_entity_id：

```sql
SELECT a.record_id, b.record_id
FROM record_reference_link a
JOIN record_reference_link b
  ON a.reference_entity_id = b.reference_entity_id
WHERE a.decision = 'AUTO_LINK'
  AND b.decision = 'AUTO_LINK'
  AND a.source <> b.source;
```

甚至更推荐直接建立 Reference-centric view：

```text
reference_entity_id = 12345
  ├── 雷小安: record A
  ├── 雷小安: record B
  ├── 腕表之家: record C
  └── 奢当家: record D
```

这不仅解决跨源匹配，也自然支持：

- 同来源重复商品；
- 同 Reference 多卖家；
- 增量新记录挂载；
- Reference 级价格聚合；
- Reference 级图片 QA；
- Reference 级异常检测。

---

# 14. 增量架构

100 万～1000 万规模不需要全量笛卡尔匹配。

每条新增/变更 record 独立执行：

```text
ingest
  -> canonicalize
  -> extract evidence
  -> exact registry lookup
  -> optional PECOS TopK
  -> hard validation
  -> write link decision
```

复杂度主要与单条记录长度和 label retrieval 有关，而不是与总 record 数平方增长。

建议组件：

```text
Raw storage        : Object Storage / Parquet
Canonical records  : PostgreSQL / ClickHouse
Reference Registry : PostgreSQL
Exact hot index    : Redis / RocksDB / in-memory immutable map
PECOS model        : Python service + C++ predict path
OCR                : 独立异步 worker
Decision Engine    : 无状态 service / batch job
Audit analytics    : ClickHouse
```

如果已有数据基础设施，不必为了本方案强行更换数据库；关键是逻辑层分离。

---

# 15. 版本化是必须的

Reference 规则会不断修正，因此每个结果必须带版本：

```text
extractor_version
normalizer_version
alias_registry_version
pecos_model_version
decision_version
```

这样出现误匹配时，可以做到：

```text
找出所有被 normalizer v17 影响的 record
重新计算
回滚旧 link
```

而不是手工修数据。

---

# 16. 黄金标签怎么标最划算

用户只愿意标几百对，因此不要随机抽样。

优先从以下集合采样：

## 16.1 Adjacent Reference Hard Negatives

例如：

```text
126610LN vs 126610LV
116610LN vs 126610LN
```

用于验证系统是否会把“相似”误判成“相同”。

## 16.2 Identifier Role Hard Cases

```text
商品本身 Reference
兼容配件 Reference
平台 SKU
序列号
```

## 16.3 Normalization Hard Cases

验证：

```text
空格是否可删
连字符是否可删
斜杠是否有意义
后缀是否有意义
```

## 16.4 Cross-source Alias Cases

每个平台挑最常见但格式不同的 Reference 表达。

## 16.5 Conflict Cases

标题、结构化字段、OCR 给出不同编号的记录。

这些标签的价值远高于“随机选几百对看是否同款”。

---

# 17. 评估体系必须改成 precision-first

不要用普通 F1 作为上线主指标。

建议至少统计：

```text
AutoLink Precision
AutoLink Count
False Merge Count
Abstention Rate
Review Yield
Conflict Rate
Reference Extraction Precision
Role Classification Precision
Source-specific Precision
Brand-specific Precision
```

并额外维护：

```text
Hard Negative False Positive Rate
```

上线门槛可以定义为：

```text
任何黄金 hard-negative 上出现 false positive => 阻断发布
```

另外，“抽样中 0 个 false positive”不等于真实误匹配率为 0。上线审核应同时看置信区间和样本规模，不能只看点估计。

---

# 18. 为什么论文的 97%~99% precision 仍然不够

论文的 exact lexical pipeline 已经很强，但对当前约束仍不足。

原因包括：

1. 论文目标是搜索体验，允许一定 false alarm；
2. 当前“同一商品”定义完全绑定 Reference，一次误链接就会把跨源价格、库存、图片等后续数据全部污染；
3. Reference 字符串天然存在相邻型号 hard negatives；
4. 商品标题可能提到“兼容对象”的 Reference，而不是自身 Reference；
5. OCR 的单字符错误可能刚好落到另一个合法 Reference。

所以必须从“分类器 precision”升级成：

> **证据满足安全不变量才自动链接，否则拒识。**

---

# 19. 关键安全不变量

建议把以下 invariant 写入单元测试和 Decision Engine：

## Invariant 1

```text
不同 canonical_ref 永不因 embedding 相似而合并
```

## Invariant 2

```text
不同 brand_id 的 Reference 不自动合并
```

## Invariant 3

```text
semantic-only candidate 永不 AUTO_LINK
```

## Invariant 4

```text
多个 exact candidate 未唯一消歧时必须 ABSTAIN
```

## Invariant 5

```text
发现高可信冲突 evidence 时必须 CONFLICT
```

## Invariant 6

```text
ACCESSORY_COMPATIBLE_REFERENCE 不能成为当前商品 Reference
```

## Invariant 7

```text
模型输出不能直接写 canonical entity relation，必须经过 Decision Engine
```

---

# 20. 推荐直接落地的模块接口

```python
class BrandResolver:
    def resolve(self, record) -> BrandDecision: ...

class IdentifierExtractor:
    def extract(self, record) -> list[IdentifierEvidence]: ...

class IdentifierRoleClassifier:
    def classify(self, evidence) -> IdentifierRole: ...

class ReferenceNormalizer:
    def normalize(self, brand_id, evidence) -> NormalizedReference: ...

class ExactReferenceRegistry:
    def lookup(self, brand_id, source, alias_norm) -> list[ReferenceEntity]: ...

class ReferenceCandidateGenerator:
    def predict(self, record, evidence, topk=5) -> list[Candidate]: ...

class ReferenceValidator:
    def validate(self, record, evidence, candidates) -> ValidationResult: ...

class ReferenceDecisionEngine:
    def decide(self, validation) -> LinkDecision: ...
```

这样做的好处是将：

```text
extract
normalize
retrieve
validate
decide
```

彻底解耦，任何模型升级都不能绕过安全门。

---

# 21. 推荐的 API 输出

不要只返回一个 `reference` 字符串。

```json
{
  "record_id": "sjdj_123",
  "decision": "AUTO_LINK",
  "reference_entity_id": 88421,
  "canonical_ref": "126610LN",
  "brand_id": 12,
  "reason": "TITLE_EXACT_AND_OCR_EXACT_AGREE",
  "evidence": [
    {
      "type": "TITLE_REFERENCE",
      "value": "126610 LN",
      "normalized": "126610LN"
    },
    {
      "type": "IMAGE_OCR_REFERENCE",
      "value": "126610LN",
      "normalized": "126610LN"
    }
  ],
  "versions": {
    "normalizer": "refnorm-3",
    "registry": "registry-2026-08-18",
    "decision": "decision-5"
  }
}
```

人工复核结果则返回：

```json
{
  "decision": "REVIEW",
  "reason": "PECOS_ONLY",
  "candidates": [
    {"reference_entity_id": 88421, "score": 0.94},
    {"reference_entity_id": 88422, "score": 0.92}
  ]
}
```

---

# 22. 测试样例

## Case A：安全 exact

```text
brand = Rolex
title = 劳力士 潜航者 126610LN 全套
```

抽取：

```text
REFERENCE: 126610LN
```

Registry 唯一：

```text
Rolex + 126610LN -> ref_entity_1
```

输出：

```text
AUTO_LINK
```

## Case B：相邻型号不能 fuzzy merge

```text
record A: 126610LN
record B: 126610LV
```

即使文本/图片 embedding 0.99 相似：

```text
不同 canonical_ref => 不匹配
```

## Case C：配件污染

```text
title = 适配 Rolex 126610LN 黑色胶带
```

Role classifier：

```text
ACCESSORY_COMPATIBLE_REFERENCE
```

输出：

```text
NO_REFERENCE
```

## Case D：多解

```text
alias_norm = 1234
```

在同品牌下映射两个历史 Reference，series 也无法消歧：

```text
ABSTAIN
```

## Case E：OCR 冲突

```text
structured = 126610LN
ocr = 126610LV
```

输出：

```text
CONFLICT
```

而不是选择 structured score 更高的一边。

## Case F：PECOS 很自信但无硬证据

```text
Top1 = 126610LN, score=0.999
```

输出：

```text
REVIEW
```

---

# 23. 分阶段落地

## P0：只做 Registry + Exact + Abstain

先实现：

- canonical brand；
- Reference Registry；
- source-aware alias；
- conservative normalization；
- identifier role；
- exact linker；
- conflict detection；
- decision audit。

这一阶段就能覆盖大量高精度数据，并且最符合当前业务风险偏好。

## P1：加入 OCR 双证据

将图片中表背/卡证/标签的 Reference OCR 接入同一个 evidence pipeline。

核心目标不是提高视觉匹配率，而是：

```text
增加独立 Reference 证据
```

## P2：加入 PECOS Candidate Generator

只处理：

```text
exact path 未命中
```

用于：

- 人工 review 候选；
- 发现新的 alias；
- 发现 Registry 缺失 Reference；
- 辅助建设 hard-negative 数据集。

模型建议先用 X-Linear + char/word TF-IDF，不需要一开始上 Transformer。

## P3：主动学习闭环

从以下集合选人工样本：

```text
高分 PECOS 但 exact miss
多候选 exact
OCR / title conflict
source 新增格式
brand 新增长尾 Reference
```

人工确认后：

- 新 canonical entity -> Registry；
- 新安全别名 -> Alias；
- 新危险模式 -> negative rule；
- 新难例 -> gold set；
- 新 weak label -> candidate model training。

---

# 24. 不建议采用的方案

## 24.1 直接把所有字段拼接后做跨源向量相似度

问题：同系列不同 Reference 会成为最危险的 false positive。

## 24.2 直接用 LLM 判断“两条商品是不是同一个”

问题：不可建立足够稳定、可证明的高精度边界；对 1000 万级成本也不合理。

LLM 可以用于离线：

- 生成规则候选；
- 解释人工 review；
- 标注辅助；
- 发现 identifier role pattern。

不能作为最终 merge authority。

## 24.3 对 Reference 做全局 aggressive normalization

例如删除所有标点、前导零、后缀。

这会把合法不同 Reference 折叠到同一 key，是结构性 false positive 来源。

## 24.4 只用 pairwise matching

如果 A-B、B-C 判同，pairwise 图很容易通过传递关系把错误扩散到整个 cluster。

Reference-centric linking 直接让每条记录链接 canonical entity，风险更可控。

## 24.5 为了 coverage 给 semantic score 设置一个“足够高”的自动阈值

即使离线测试看起来不错，分布漂移后也可能产生新 false positive。

当前业务应该把 score 用于：

```text
candidate ordering
```

而不是：

```text
automatic identity proof
```

---

# 25. 与论文结果的最终对应关系

| 论文设计 | 当前系统迁移 |
|---|---|
| Brand Entity | Reference Entity |
| MetaTS-NER | Identifier Span + Role Extractor |
| Exact Lexical Match | Source-scoped Reference Alias Registry |
| Store Tag | Source scope |
| Product Type Filter | Brand + ProductRole + Series/Family Filter |
| M2E-PECOS | Mention/Identifier -> Top-K Reference |
| Q2E-PECOS | Record text -> Top-K Reference |
| NO_ENTITY | NO_REFERENCE / ABSTAIN |
| Fusion exact-first | Hard exact gate + semantic candidate-only |
| Human strong labels | 几百条 Reference hard cases |
| Weak engagement labels | 结构化字段/标题/OCR一致 weak labels |
| Brand entity prediction | record -> reference_entity_id |

---

# 26. 直接落地建议

如果现在就开始实现，我建议不要先训练任何大模型，而是按以下优先级：

```text
1. Reference Registry
2. 品牌规范化
3. Identifier Role
4. Conservative Normalization
5. Source-scoped Exact Alias
6. Ambiguity / Conflict / Abstain
7. record -> reference_entity_id
8. OCR evidence
9. PECOS Top-K candidate
10. 主动学习与 alias 回流
```

第一版最关键的成功标准不是“匹配了多少”，而是：

> **所有 AUTO_LINK 都能给出明确、可审计、可复现的 Reference 证据链。**

当 coverage 不够时，再让 PECOS、OCR、LLM、视觉模型逐层扩展“候选集合”；但这些能力都不应突破 Decision Engine 的硬约束。

这也是这篇论文对当前 Spec 最大的价值：

> 它证明了在超大、长尾 entity 空间里，**exact lexical linking + 侧信息消歧本身就是极强的高精度方案；语义 XMC 的正确位置是补 coverage，而不是取代 canonical identifier。**

对于“同一商品 = 同一 Reference”且 false positive 代价极高的二奢腕表场景，应进一步把这个思想贯彻为：

> **Reference-first、exact-first、evidence-first、abstention-by-default。**
