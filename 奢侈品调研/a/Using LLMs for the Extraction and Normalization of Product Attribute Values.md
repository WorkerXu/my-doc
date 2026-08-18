# Using LLMs for the Extraction and Normalization of Product Attribute Values：面向跨源二奢腕表 Reference 的高精度抽取、规范化与实体链接方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选择论文与开源项目 **Using LLMs for the Extraction and Normalization of Product Attribute Values / WDC-PAVE** 进行深入分析。

- 论文：<https://arxiv.org/abs/2403.02130>
- 论文 PDF：<https://www.uni-mannheim.de/media/Einrichtungen/dws/Files_Research/Web-based_Systems/pub/ADBIS2024_Using_LLMs_for_the_Extraction_and_Normalization_of_Product_Attribute_Values.pdf>
- 官方代码：<https://github.com/wbsg-uni-mannheim/wdc-pave>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

当前 Spec 的业务真值非常明确：**同一商品就是同一品牌下同一 reference number**，而且 precision 优先到极致，可以接受漏匹配。由此决定了最合适的架构并不是再训练一个“商品 A 与商品 B 是否相同”的黑盒 matcher，而是把每条商品独立解析并链接到一个可信的 Reference Entity。

WDC-PAVE 最值得借鉴的是把电商脏文本处理拆为：

1. **Extraction**：从 title / description 中提取属性值；
2. **Normalization**：把不同表面写法规范到统一值。

论文示例直接包含 `Part Number`，并展示类似：

```text
raw:        6280-59-B21
normalized: 628059B21
```

这与腕表 reference 高度同构：

```text
126610 LN
126610-LN
126610LN
```

可能只是书写差异；但：

```text
126610LN
126610LV
```

虽然非常相似，却绝不能合并。

因此建议将论文方案改造成一个 **Precision-First Reference Resolver**：

```text
雷小安 / 腕表之家 / 奢当家
          │
          ▼
   Source Adapter / 字段标准化
          │
          ▼
      Brand Canonicalization
          │
          ▼
 ┌──────────────────────────────┐
 │ Reference Candidate Extraction│
 │ 1. 独立 reference 字段        │
 │ 2. 规则 / 字典 / regex        │
 │ 3. LLM constrained extraction │
 │ 4. OCR（辅助证据）             │
 └──────────────────────────────┘
          │
          ▼
   Reference Role Validation
  “这是当前商品自己的型号吗？”
          │
          ▼
 Brand-specific Canonicalizer
          │
          ▼
    Reference Registry 查验
          │
          ▼
 Conflict / Accessory / Multi-ref Gate
          │
     ┌────┴────┐
     │         │
 VERIFIED    ABSTAIN
     │
     ▼
 exact key = (brand_id, canonical_reference)
     │
     ▼
跨源记录按 exact key 聚合
```

最终自动匹配规则应保持非常简单：

> **只有两条记录都处于 `VERIFIED_REFERENCE`，且 `(brand_id, canonical_reference)` 完全相同，并且没有冲突证据时，才允许自动匹配。**

LLM、文本 embedding、图片相似度都不能直接产生“同商品”结论；它们只负责补充候选、证据或触发拒识。

---

## 1. 为什么 WDC-PAVE 正好击中当前问题

如果三源数据都稳定提供：

```text
brand = Rolex
reference = 126610LN
```

那么匹配只需要 exact key：

```sql
WHERE brand_id = ?
  AND canonical_reference = ?
```

真正困难的是上游数据经常类似：

```text
标题：劳力士 潜航者 黑水鬼 41mm 126610 LN 全套
reference 字段：NULL
平台货号：LX202408019338
```

或者：

```text
Rolex 126610LN 原装表带 / 适配黑水鬼
```

或者：

```text
欧米茄 海马 210.30.42.20.01.001 现货
```

因此系统真正要解决的是：

1. 从不同来源字段中找到 reference 候选；
2. 不把平台 SKU、序列号、鉴定号误当成 reference；
3. 不把“兼容型号、配件适配对象、对比对象”误当成当前商品 reference；
4. 对合法格式做**无损且 collision-safe** 的规范化；
5. 无法唯一确定时主动 `ABSTAIN`。

WDC-PAVE 正好对 1 和 4 给出了完整方法与代码基线。

---

## 2. WDC-PAVE 的实际代码架构

官方仓库：<https://github.com/wbsg-uni-mannheim/wdc-pave>

README 中可以看到主要模块：

```text
data/processed_datasets/
prompts/
pieutils/
scripts/
```

相关实验入口包括：

```text
scripts/01_run_example_values_prompts.sh
scripts/02_run_prompts_with_training_data.sh
scripts/08_run_prompts_for_extraction_with_normalization.sh
scripts/10_run_prompts_for_normalization_multiple_attributes.sh
```

这套代码并不是简单“拼 prompt -> 调模型”，而是：

```text
Dataset Preprocessing
       ↓
Task / Attribute Schema
       ↓
Normalization Guidelines
       ↓
Category-aware Few-shot Retrieval
       ↓
Chat Prompt
       ↓
LLM
       ↓
JSON Parse + Pydantic Validation
       ↓
Evaluation / Cost Accounting
```

### 2.1 Prompt Contract

`prompts/08_extraction_with_normalization/8_few_shot_extraction_with_normalization.py` 的核心任务约束大意为：

```text
Extract valid attribute values from the product text,
normalize them according to guidelines,
respond in JSON,
mark unknown values as n/a,
do not explain the answer.
```

它有六个可直接迁移的工程点：

1. 输出固定为 JSON；
2. 明确允许 `n/a`，即模型可以拒答；
3. normalization guideline 动态注入；
4. 不同 category 使用不同 attribute schema；
5. 模型使用 `temperature=0`；
6. 输出再经过 Pydantic 校验。

对当前项目，建议把输出 schema 收紧成带证据的候选：

```json
{
  "brand": "ROLEX",
  "reference_candidates": [
    {
      "raw": "126610 LN",
      "source_field": "title",
      "evidence_text": "黑水鬼 126610 LN 全套",
      "role": "CURRENT_PRODUCT_REFERENCE"
    }
  ],
  "abstain": false
}
```

而不是只输出：

```json
{"reference": "126610LN"}
```

原因是 production 系统必须能验证：模型给出的字符串是否真的存在于输入证据中。

### 2.2 Pydantic 是故障隔离层

代码会根据已知属性动态生成 Pydantic model，再把 LLM JSON 填入 model。如果 JSON 解析失败会尝试修复；如果 schema validation 仍失败，则不会形成合法 prediction。

这一思想应该进一步强化：

```text
LLM output
   ↓
JSON schema
   ↓
Pydantic
   ↓
Evidence grounding
   ↓
Role validator
   ↓
Reference normalizer
```

任何一步失败都 `ABSTAIN`，而不是尝试“猜一个最可能答案”。

### 2.3 按品类做语义相似 few-shot 检索

`pieutils/search.py` 中的 `CategoryAwareSemanticSimilarityExampleSelector` 会：

```text
训练样本
   ↓
OpenAIEmbeddings
   ↓
按 category 建 FAISS Vector Store
   ↓
对当前文本 semantic search
   ↓
取 top-k 相似标注样本
   ↓
作为 few-shot demonstrations
```

它还支持过滤相同网站的样本，这一点对三源数据非常有价值。

腕表场景可以映射为：

```text
category -> brand
website  -> source
```

例如处理雷小安 Rolex 商品时，优先从同品牌、但来自腕表之家或奢当家的已标注 hard cases 中检索 demonstrations，降低模型只记住某个平台标题模板的风险。

### 2.4 FAISS 的正确用途

WDC-PAVE 的 FAISS 是用来找 few-shot demonstrations，不是把向量距离当成商品匹配真值。

当前系统也应该坚持：

```text
向量检索 = 辅助
reference exact key = 最终业务真值
```

文本 embedding、视觉 embedding 可以用于：

- few-shot 示例检索；
- 人工复核排序；
- reference 缺失时的辅助候选召回；

但绝不能绕过 reference gate。

---

## 3. 论文实验对当前系统的启示

WDC-PAVE benchmark 包含来自多个网站、多个商品类别的人工确认属性值。论文实验表明：

- GPT-4 在直接属性抽取上可以达到约 90% 级别 F1；
- attribute example values 与语义相似 few-shot demonstrations 能显著提升效果；
- extraction + normalization 可以一步完成；
- 单独 normalization 的表现还可以更高；
- string wrangling 等 identifier-like 规范化操作尤其适合 LLM。

但这恰好说明：**即使属性抽取表现很高，也不能让 LLM 直接负责自动合并。**

假设 precision 只有 99%，放到千万级数据仍可能产生不可接受的误匹配。因此 LLM 应定位为：

```text
Candidate Generator / Evidence Extractor
```

而不是：

```text
Final Matcher
```

并且建议把 extraction 与 authoritative normalization 拆开：

```text
LLM extraction
      ↓
raw reference span
      ↓
deterministic brand-specific canonicalization
      ↓
trusted registry validation
```

而不是让 LLM直接生成最终 canonical reference。

---

## 4. 推荐数据模型

三源商品统一为：

```python
class ProductRecord:
    source: str
    source_item_id: str
    title: str | None
    description: str | None
    brand_raw: str | None
    reference_raw: str | None
    category_raw: str | None
    images: list[str]
    source_fields: dict
    content_hash: str
```

解析结果：

```python
class ReferenceResolution:
    product_id: str
    brand_id: int | None
    raw_candidates: list[str]
    canonical_reference: str | None
    status: Literal[
        "VERIFIED_REFERENCE",
        "SOFT_REFERENCE",
        "CONFLICT",
        "NO_REFERENCE",
        "ABSTAIN"
    ]
    evidence: list[Evidence]
    extractor_version: str
    normalizer_version: str
    registry_version: str
    decision_reason: str
```

Evidence：

```python
class Evidence:
    source_type: Literal[
        "STRUCTURED_FIELD",
        "TITLE",
        "DESCRIPTION",
        "OCR",
        "LLM",
        "REGISTRY"
    ]
    raw_value: str
    field_name: str | None
    evidence_text: str | None
    role: str | None
```

每个自动结论都必须能回答：

> “系统为什么认为这是这个 reference？”

而不是只留下一个概率分数。

---

## 5. 第一层：Brand Canonicalization

reference 不能脱离品牌直接比较。建议首先建立：

```text
劳力士       -> ROLEX
Rolex        -> ROLEX
ROLEX SA     -> ROLEX
欧米茄       -> OMEGA
Omega        -> OMEGA
```

所有匹配 key 都必须是：

```text
(brand_id, canonical_reference)
```

而不是只有 reference 字符串。

Precision-first 硬规则：

```python
if brand_id is None:
    return ABSTAIN
```

---

## 6. 第二层：Reference Candidate Extraction

推荐四级抽取，按可信度排序，而不是按模型复杂度排序。

### 6.1 L0：独立结构化 reference 字段

如果来源提供：

```text
reference_no / model_number / ref / 型号
```

优先使用，但仍需校验，因为平台 SKU、序列号、鉴定号可能被错误塞进“型号”字段。

### 6.2 L1：品牌规则 / regex / dictionary

高频品牌维护独立 policy：

```yaml
ROLEX:
  uppercase: true
  remove_spaces: true
  extract_patterns: [...]

OMEGA:
  uppercase: true
  preserve_dot_grouping: true
  extract_patterns: [...]
```

不要使用一个“万能 reference regex”覆盖所有品牌。

### 6.3 L2：LLM constrained extraction

只有 L0/L1 无法唯一确定时再调用 LLM。

Prompt 必须明确：

```text
你是 reference evidence extractor，不是商品匹配器。
只能抽取输入中真实存在的字符串。
禁止补全、纠错、猜测或生成输入中不存在的 reference。
需要区分当前商品型号与 compatibility / accessory / comparison / mentioned-only。
存在多个无法消歧候选时返回 abstain=true。
```

返回：

```json
{
  "candidates": [
    {
      "raw": "126610 LN",
      "role": "CURRENT_PRODUCT_REFERENCE",
      "evidence": "黑水鬼 126610 LN 全套"
    }
  ],
  "abstain": false
}
```

服务端必须做 grounding：

```python
assert candidate.raw in original_text
```

模型输出如果不在输入原文中，立即拒绝。

### 6.4 L3：OCR

图片可用于：

- 表背刻字；
- 保卡型号；
- 吊牌 reference；
- 发票或附件中的编号。

流程应是：

```text
image
  ↓
OCR
  ↓
reference-like token
  ↓
同一套 role / normalization / registry validation
```

图片“长得像”不能直接产生 reference；OCR 也只是证据之一。

---

## 7. 第三层：Reference Role Validation

这是防止 false positive 的关键。

标题中出现一个 reference，不代表它属于当前售卖商品。例如：

```text
Rolex 126610LN 原装表带
```

这里 `126610LN` 应标记为：

```text
ACCESSORY_FOR_REFERENCE
```

而不是 `CURRENT_PRODUCT_REFERENCE`。

建议 role 枚举：

```text
CURRENT_PRODUCT_REFERENCE
COMPATIBLE_WITH_REFERENCE
ACCESSORY_FOR_REFERENCE
COMPARISON_REFERENCE
MENTIONED_ONLY
PLATFORM_SKU
SERIAL_NUMBER
UNKNOWN_ID
```

高风险词：

```text
适配 / 适用 / compatible with / for
表带 / strap
配件 / accessory
对比 / vs
```

但不能仅靠关键词，因为“全套带盒证”中的“盒”并不表示商品是盒子。推荐：

```text
规则快速过滤
    +
LLM role classification
    +
商品类目一致性校验
```

只有明确为 `CURRENT_PRODUCT_REFERENCE` 的候选才有资格进入 VERIFIED。

---

## 8. 第四层：Brand-specific Collision-safe Canonicalization

这是不能直接照搬通用 string wrangling 的地方。

不推荐全局规则：

```python
canonical = re.sub(r'[^A-Z0-9]', '', raw.upper())
```

因为它默认所有标点都没有语义，可能把两个合法 reference 压成同一个值。

推荐：

```python
def canonicalize(brand_id, raw):
    policy = policies[brand_id]
    value = unicode_nfkc(raw).strip()

    if policy.uppercase:
        value = value.upper()

    if policy.remove_spaces:
        value = remove_spaces(value)

    if policy.hyphen_equivalences:
        value = apply_whitelisted_hyphen_rules(value)

    if policy.dot_rules:
        value = apply_brand_specific_dot_rules(value)

    return value
```

每条规则必须版本化并保存来源：

```text
rule_version
brand
before
canonical
evidence_source
approved_by
```

上线前做 collision test：

```python
for ref in all_known_refs:
    canon = canonicalize(ref)
    assert canon maps to exactly one reference entity
```

只要出现：

```text
ref_A != ref_B
canonicalize(ref_A) == canonicalize(ref_B)
```

该规则就不能进入自动路径。

---

## 9. Reference Registry：把 Pair Matching 变成 Entity Linking

建议中心 registry：

```sql
CREATE TABLE reference_entity (
    id BIGSERIAL PRIMARY KEY,
    brand_id BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (brand_id, canonical_reference)
);

CREATE TABLE reference_alias (
    id BIGSERIAL PRIMARY KEY,
    reference_entity_id BIGINT NOT NULL REFERENCES reference_entity(id),
    raw_alias TEXT NOT NULL,
    normalized_alias TEXT NOT NULL,
    provenance TEXT NOT NULL,
    approved BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (reference_entity_id, raw_alias)
);
```

每条商品独立链接：

```text
雷小安 A   -> ROLEX / 126610LN
腕表之家 B -> ROLEX / 126610LN
奢当家 C   -> ROLEX / 126610LN
```

三者自然属于同一 reference entity。

这样不需要在 1000 万条记录上做全量 pairwise 笛卡尔积。主流程是每条记录一次解析，再通过 `(brand_id, canonical_reference)` 的 B-tree/hash 索引查询；数据库 lookup 通常为 O(log M)（B-tree）或近似 O(1)（hash/KV）。

---

## 10. 自动 Match Gate：硬规则收口

```python
def can_auto_match(a, b) -> bool:
    if a.status != "VERIFIED_REFERENCE":
        return False
    if b.status != "VERIFIED_REFERENCE":
        return False
    if a.brand_id != b.brand_id:
        return False
    if a.canonical_reference != b.canonical_reference:
        return False
    if a.has_conflict or b.has_conflict:
        return False
    return True
```

`VERIFIED_REFERENCE` 本身也必须是严格状态：

```text
brand 已验证
AND 恰好一个 current-product reference
AND canonicalization collision-safe
AND reference 命中 trusted registry
AND 没有显式冲突 reference
AND 没有 accessory/compatibility 冲突
```

新发现但 registry 不存在的 reference：

```text
SOFT_REFERENCE / REVIEW
```

而不是自动创建实体并立即合并。

### 10.1 负证据高于弱相似证据

例如：

```text
title candidate      = 126610LN
structured ref field = 126610LV
```

无论图片多像，都应该：

```text
CONFLICT / ABSTAIN
```

推荐 hard veto：

1. 两个可信字段给出不同 reference；
2. 同一商品存在多个无法解释的 reference；
3. 商品角色是配件；
4. 品牌与 reference registry 不一致；
5. canonicalization 产生 collision；
6. LLM reference 没有原文 grounding；
7. OCR 与结构化字段发生明确冲突。

---

## 11. 多 Reference 标题的默认策略

危险例子：

```text
126610LN / 116610LN 新老款对比
```

或者：

```text
适用 126610LN 126610LV 124060 表带
```

默认：

```python
if len(distinct_current_product_candidates) != 1:
    return ABSTAIN
```

除非存在更强、可解释的结构化证据，否则宁可不匹配。

---

## 12. WDC-PAVE Few-shot 如何改造成腕表专用版本

WDC-PAVE 以 category 建 FAISS index；腕表可改为：

```text
一级：brand
二级：source / language
三级：hard-case type
```

训练样本：

```json
{
  "brand": "ROLEX",
  "source": "腕表之家",
  "input": "...",
  "reference_raw": "126610 LN",
  "reference_canonical": "126610LN",
  "role": "CURRENT_PRODUCT_REFERENCE",
  "hard_case": "SPACE_VARIANT"
}
```

检索：

```text
current item
   ↓
brand filter
   ↓
semantic retrieval top-N
   ↓
优先不同 source
   ↓
hard-case diversity rerank
   ↓
3~5 demonstrations
```

few-shot 不应全是正例，建议固定加入：

```text
1 个正常正例
1 个格式变体正例
1 个 accessory negative
1 个 multiple-reference abstain
1 个 platform-SKU negative
```

也就是显式教模型“什么时候应该不回答”。

---

## 13. 百万到千万规模的存储与增量架构

主索引：

```sql
CREATE INDEX idx_product_verified_reference
ON product_reference_resolution (brand_id, canonical_reference)
WHERE status = 'VERIFIED_REFERENCE'
  AND has_conflict = FALSE;
```

向量数据库只服务 few-shot retrieval / review assist，不承担主匹配。

每条 source record 计算：

```text
content_hash = hash(
    title,
    description,
    brand_raw,
    reference_raw,
    image_urls,
    relevant_source_fields
)
```

并保存：

```text
extractor_version
normalizer_version
registry_version
prompt_version
ocr_version
```

幂等处理 key：

```text
(source, source_item_id, content_hash, pipeline_version)
```

流水线：

```text
Crawler / Existing DB
       │
       ▼
Change Detector
       │
       ▼
Reference Extraction Queue
       │
       ├─ cheap rules
       ├─ LLM fallback
       └─ OCR fallback
       ▼
Validator / Registry Linker
       │
       ├─ VERIFIED -> exact group index
       ├─ REVIEW   -> human queue
       └─ ABSTAIN  -> storage only
```

MVP 可以从：

```text
PostgreSQL + Python worker + Redis/Celery
```

开始。规模扩大后再替换为 Kafka/Flink/Spark。关键是幂等、版本可追踪、可重跑，而不是一开始选择复杂基础设施。

---

## 14. 人工黄金标签应该优先标 Hard Negatives

Spec 允许几百对人工黄金标签，不建议随机抽样。随机数据大部分太容易，无法发现 precision 风险。

优先构造：

### A. 同系列近邻 reference

```text
126610LN vs 126610LV
```

### B. 格式等价变体

```text
126610 LN vs 126610LN
```

### C. 配件提及 reference

```text
腕表 126610LN
vs
适配 126610LN 的表带
```

### D. 多 reference 标题

```text
126610LN / 116610LN 对比
```

### E. 平台 SKU 冒充 reference

### F. OCR 与 title 冲突

### G. 新品牌、新格式、训练集未见 reference pattern

黄金集用于：

```text
reference_candidate_precision
role_classification_precision
canonicalization_collision_test
verified_reference_precision
regression test
```

需要强调：几百条标签不足以从统计上“证明”99.99% precision，因此极高 precision 不能主要依赖概率阈值，而必须依赖 deterministic gate、registry、collision-safe rules 和 abstention。

---

## 15. 指标不要只看 F1

推荐 dashboard：

```text
reference_candidate_precision
reference_candidate_recall
verified_reference_precision
verified_reference_coverage
auto_match_precision
auto_match_coverage
abstain_rate
conflict_rate
manual_review_rate
canonical_collision_count
```

发布门槛按品牌、来源拆分。

最重要的问题是：

> **Auto-match bucket 中到底出现了多少 false positives？**

某品牌规则证据不够，就只允许输出 `SOFT_REFERENCE`，不能因为全局 F1 很高而自动放行。

---

## 16. 可直接落地的 Python 决策骨架

```python
from dataclasses import dataclass
from enum import Enum

class Status(str, Enum):
    VERIFIED = "VERIFIED_REFERENCE"
    SOFT = "SOFT_REFERENCE"
    CONFLICT = "CONFLICT"
    NO_REFERENCE = "NO_REFERENCE"
    ABSTAIN = "ABSTAIN"

@dataclass
class Resolution:
    brand_id: int | None
    canonical_reference: str | None
    status: Status
    reason: str


def resolve_reference(record, brand_registry, reference_registry, policies):
    brand_id = brand_registry.resolve(record.brand_raw, record.title)
    if brand_id is None:
        return Resolution(None, None, Status.ABSTAIN, "brand_unresolved")

    candidates = []
    candidates += extract_structured_candidates(record)
    candidates += extract_rule_candidates(record, policies[brand_id])

    if not has_single_safe_candidate(candidates):
        candidates += extract_llm_candidates(record, brand_id)

    # LLM 输出必须能回指原文证据。
    candidates = [
        c for c in candidates
        if evidence_contains_raw_candidate(record, c)
    ]

    current_refs = [
        c for c in candidates
        if c.role == "CURRENT_PRODUCT_REFERENCE"
    ]

    if has_explicit_conflict(current_refs):
        return Resolution(brand_id, None, Status.CONFLICT, "reference_conflict")

    distinct = distinct_raw_references(current_refs)
    if len(distinct) != 1:
        return Resolution(brand_id, None, Status.ABSTAIN, "not_exactly_one_reference")

    raw = distinct[0]
    canonical = canonicalize_safely(brand_id, raw, policies[brand_id])
    if canonical is None:
        return Resolution(brand_id, None, Status.ABSTAIN, "unsafe_normalization")

    entity = reference_registry.get(brand_id, canonical)
    if entity is None or entity.status != "APPROVED":
        return Resolution(brand_id, canonical, Status.SOFT, "unknown_reference")

    if has_any_veto(record, candidates, entity):
        return Resolution(brand_id, canonical, Status.CONFLICT, "veto_evidence")

    return Resolution(brand_id, canonical, Status.VERIFIED, "verified_exact_reference")


def same_product(a: Resolution, b: Resolution) -> bool:
    return (
        a.status == Status.VERIFIED
        and b.status == Status.VERIFIED
        and a.brand_id == b.brand_id
        and a.canonical_reference == b.canonical_reference
    )
```

这套决策非常容易做单元测试和审计。

推荐 regression tests：

```python
def test_neighbor_reference_must_not_match():
    a = resolve("ROLEX", "126610LN")
    b = resolve("ROLEX", "126610LV")
    assert same_product(a, b) is False


def test_accessory_reference_must_abstain():
    x = resolve_title("适配 Rolex 126610LN 原装风格表带")
    assert x.status != Status.VERIFIED


def test_multiple_refs_must_abstain():
    x = resolve_title("126610LN / 116610LN 新老款对比")
    assert x.status == Status.ABSTAIN


def test_llm_hallucination_must_be_rejected():
    text = "劳力士黑水鬼 41mm"
    candidate = "126610LN"
    assert validate_llm_candidate(text, candidate) is False
```

线上每出现一个误判，都转成永久 regression case。

---

## 17. LLM 成本控制

不应让 LLM 跑全部 1000 万条。

分层：

```text
Tier 0：结构化字段 + registry exact
Tier 1：品牌 regex / alias rules
Tier 2：LLM extraction
Tier 3：OCR / manual review
```

只有前两层无法决策的记录进入 LLM。

缓存：

```text
(input_hash, prompt_version, model_version) -> extraction_result
```

历史回填先规则全量跑一次，只把 unresolved subset 发给模型，能大幅降低成本。

---

## 18. 推荐上线顺序

### Phase 1：Precision Core

先做：

```text
Brand Registry
Reference Registry
Source Field Mapper
Brand-specific Canonicalization
Exact-match Gate
Conflict / Abstain State Machine
Evidence Audit Schema
```

这一版即使 coverage 不高，也已经能安全产生一批自动匹配结果。

### Phase 2：WDC-PAVE 风格 LLM Extraction

对 Phase 1 无法解析的记录：

```text
Semantic Few-shot Retrieval
        ↓
LLM Structured Extraction
        ↓
Span Grounding
        ↓
Role Validation
        ↓
Safe Normalization
        ↓
Registry Validation
```

LLM 的作用是扩大 VERIFIED coverage，而不是改变 VERIFIED 的安全标准。

### Phase 3：OCR + Human Feedback

```text
OCR
 ↓
人工复核
 ↓
Hard-case labels
 ↓
Rule / Alias / Few-shot Example / Regression Suite 回流
```

---

## 19. 最终判断

WDC-PAVE 证明了：LLM 可以高质量地从多网站电商脏文本中抽取并规范化属性，而且 schema、example values、few-shot demonstrations、语义相似示例检索都会显著影响效果。

但当前二奢/腕表系统不能停在“属性抽取 F1 很高”。为了满足“绝对不能误匹配”，必须额外加五层安全约束：

```text
1. Evidence Grounding
   reference 必须回指原始字段或 OCR 证据。

2. Role Validation
   必须证明 reference 属于当前商品，而不是配件、兼容或对比对象。

3. Brand-specific Collision-safe Normalization
   只做被证明安全的等价变换，不做模糊纠错。

4. Trusted Reference Registry
   canonical reference 必须链接到明确、可信的实体。

5. Hard Abstention Gate
   任何冲突、多值、不确定都拒绝自动匹配。
```

因此推荐的生产架构不是：

```text
LLM Matcher + Similarity Threshold
```

而是：

```text
WDC-PAVE 风格 Reference Extractor
       +
Safe Canonicalization
       +
Reference Registry
       +
Exact Match Gate
       +
ABSTAIN / Human Review
```

这套方案同时满足：

- **Precision-first**：最终自动合并由 exact reference gate 决定；
- **字段稀疏**：LLM/OCR 可从非结构化字段补候选；
- **百万到千万规模**：每条记录独立 entity linking，不做全量 pairwise；
- **持续增量**：content hash + pipeline version 支持幂等重跑；
- **人工标签有限**：把几百条黄金样本集中在 hard negatives；
- **可解释可审计**：每次决策保留 raw span、规则版本、registry entity、decision reason。

如果只实现一个第一版，建议甚至先不追求 LLM 覆盖率：**优先把 `Reference Registry + safe canonicalization + exact-match gate + abstention` 做稳，再把 WDC-PAVE 风格 LLM extractor 放在 unresolved records 上逐步扩大 coverage。**

这最符合当前 Spec 的核心要求：**绝对不能误匹配，允许漏匹配。**