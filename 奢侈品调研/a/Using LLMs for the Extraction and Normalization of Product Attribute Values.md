# Using LLMs for the Extraction and Normalization of Product Attribute Values：面向跨源二奢腕表 Reference 的高精度抽取、规范化与实体链接方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选择论文与开源项目 **Using LLMs for the Extraction and Normalization of Product Attribute Values / WDC-PAVE** 进行深入分析。

- 论文：<https://arxiv.org/abs/2403.02130>
- 论文 PDF：<https://www.uni-mannheim.de/media/Einrichtungen/dws/Files_Research/Web-based_Systems/pub/ADBIS2024_Using_LLMs_for_the_Extraction_and_Normalization_of_Product_Attribute_Values.pdf>
- 官方代码：<https://github.com/wbsg-uni-mannheim/wdc-pave>
- 当前需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

当前 Spec 的核心不是普通的“相似商品匹配”，而是一个非常明确、并且对误判极端敏感的实体链接问题：

> **同一个商品 = 同一品牌下同一个 reference number；自动系统宁可漏掉，也绝不能把两个不同 reference 合并。**

WDC-PAVE 对这个问题最有价值的地方不是“用 GPT 替换正则”，而是把电商脏文本处理拆成了两个不同任务：

1. **Extraction**：从 title / description 中找出真正的属性值；
2. **Normalization**：把不同写法映射成统一形式。

论文示例中直接包含 `Part Number`，并展示了类似：

```text
raw:        6280-59-B21
normalized: 628059B21
```

这与腕表 reference 的现实问题高度同构：

```text
126610 LN
126610-LN
126610LN
```

可能是同一个 reference 的不同书写方式；但与此同时：

```text
126610LN
126610LV
```

只差两个字符，却绝不能合并。

因此，本次建议不是把 WDC-PAVE 原方案原样搬到生产，而是把它改造成一个 **Precision-First Reference Resolver**：

```text
雷小安 / 腕表之家 / 奢当家
          │
          ▼
   Source Adapter / 字段标准化
          │
          ▼
      品牌 Canonicalization
          │
          ▼
 ┌──────────────────────────────┐
 │ Reference Candidate Extraction│
 │ 1. 独立 reference 字段        │
 │ 2. 规则 / 字典 / regex        │
 │ 3. LLM constrained extraction │
 │ 4. OCR（只作为辅助证据）       │
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

最终生产规则保持极其简单：

> **只有两条商品记录都拿到 `VERIFIED_REFERENCE`，并且 `(brand_id, canonical_reference)` 完全相同，且不存在任何冲突证据时，才允许自动匹配。**

LLM、图片相似度、文本 embedding 都不能直接产生“同商品”结论；它们只能帮助找 reference、确认 reference，或者触发拒识。

这比直接训练 pairwise matcher 更符合当前 Spec，因为“是否同款”的业务真值本来就由 reference 定义，没有必要让模型重新学习一个更模糊的代理目标。

---

## 1. 为什么这篇论文适合当前需求

### 1.1 当前问题的真正瓶颈在上游，不在 pairwise matcher

如果三源数据每条记录都有准确的结构化字段：

```text
brand = Rolex
reference = 126610LN
```

那么匹配根本不需要神经网络：

```sql
WHERE brand_id = ?
  AND canonical_reference = ?
```

问题在于现实数据通常是：

```text
标题：劳力士 潜航者 黑水鬼 41mm 126610 LN 全套 2023
reference 字段：NULL
平台货号：LX202408019338
```

或者：

```text
标题：Rolex 126610LN 原装表带 / 适配黑水鬼
```

又或者：

```text
标题：欧米茄 海马 210.30.42.20.01.001 现货
```

所以系统的难点实际上是：

1. 从任意来源字段中找出 reference 候选；
2. 不把平台 SKU、鉴定号、序列号等误认成 reference；
3. 不把“兼容型号/提及型号”误认成当前商品 reference；
4. 对合法格式做安全的规范化；
5. 在无法确定时主动拒识。

WDC-PAVE 正好把 1 和 4 作为独立研究对象。

### 1.2 论文不是只有 extraction，还显式研究 normalization

论文把电商属性规范化分为多类操作，包括：

- Name Expansion；
- Generalization；
- Unit Conversion；
- String Wrangling；
- 大小写与标点处理等。

对于 reference，最相关的是 String Wrangling。

不过这里有一个关键差异：

> **WDC-PAVE 的属性规范化目标偏向语义等价；当前需求的 reference 规范化必须接近“无损、可证明等价”。**

例如颜色属性把 `Neon Lime Green` 归一到 `Green` 是合理的；但 reference 绝不能做类似模糊泛化。任何规则只要存在把两个真实 reference 压成同一个值的可能，就不能进入自动匹配路径。

因此我们应该借鉴它的“Extraction → Normalization”架构，而不是照搬所有 normalization 策略。

---

## 2. WDC-PAVE 的技术实现与代码架构

官方仓库：

<https://github.com/wbsg-uni-mannheim/wdc-pave>

README 中给出的工程入口很清晰：

```text
data/processed_datasets/
prompts/
pieutils/
scripts/
```

主要实验脚本包括：

```text
scripts/01_run_example_values_prompts.sh
scripts/02_run_prompts_with_training_data.sh
scripts/08_run_prompts_for_extraction_with_normalization.sh
scripts/10_run_prompts_for_normalization_multiple_attributes.sh
```

代码不是简单地拼一个 prompt 发给模型，而是已经形成了一个“数据预处理 → prompt 构造 → few-shot 检索 → LLM → 结构化校验 → 评估”的完整流水线。

### 2.1 Prompt orchestration

`prompts/08_extraction_with_normalization/8_few_shot_extraction_with_normalization.py` 的核心 prompt 大意是：

```text
Split the product {part} by whitespace.
Extract the valid attribute values from the product {part}
and normalize the attribute values according to the guidelines below.
Respond in JSON format.
Unknown attribute values should be marked as n/a.
Do not explain your answer.
Guidelines:
{guidelines}
```

它包含几个非常值得迁移的工程设计：

1. **输出被限定为 JSON**；
2. **明确允许 `n/a`**，不是强迫模型一定给答案；
3. **规范化规则作为 guideline 动态注入**；
4. **按 category 使用不同 attribute schema**；
5. `temperature=0`，减少不必要的随机性；
6. 模型输出再通过 Pydantic 做 schema validation。

这比“请帮我从标题抽型号”可靠得多。

### 2.2 Pydantic 不是格式美化，而是一道故障隔离层

代码会先按 category 创建对应的 Pydantic model：

```python
pydantic_models = create_pydanctic_models_from_known_attributes(
    task_dict['known_attributes']
)
```

LLM 响应拿到后：

```python
json_response = json.loads(response)
pred = pydantic_models[category](**json.loads(response))
```

如果 JSON 无法解析，会尝试修复；如果仍不满足 schema，则不产生合法 prediction。

这对我们的 reference 服务非常重要。

推荐进一步收紧 schema：

```json
{
  "brand": "Rolex",
  "reference_candidates": [
    {
      "raw": "126610 LN",
      "span": "...黑水鬼 126610 LN 全套...",
      "role": "current_product_reference",
      "source_field": "title"
    }
  ],
  "abstain": false
}
```

而不是只返回：

```json
{"reference": "126610LN"}
```

原因是**必须保留证据链**。模型如果输出了一个输入文本中根本不存在的 reference，后处理可以立刻拒绝。

### 2.3 Category-aware semantic few-shot retrieval

项目中的 `pieutils/search.py` 实现了 `CategoryAwareSemanticSimilarityExampleSelector`。

核心逻辑是：

```text
training examples
      │
      ▼
OpenAIEmbeddings
      │
      ▼
按 category 建立 FAISS vector store
      │
      ▼
对当前商品文本做 semantic search
      │
      ▼
top-k 相似 examples
      │
      ▼
作为 few-shot demonstrations 注入 prompt
```

并且支持：

```text
force_from_different_website = True
```

也就是可以避免从当前网站挑演示样本，让模型学习更具跨站泛化性的模式。

映射到三源腕表，可以改成：

```text
category  -> brand
website   -> source
```

例如处理雷小安的一条 Rolex 商品：

```text
优先检索：Rolex + 与当前标题最相似的已标注样本
限制：示例尽量来自腕表之家/奢当家，而不是当前来源
```

这样 few-shot 学到的是“跨来源写法如何解析成同一 reference”，而不是背某个平台模板。

### 2.4 FAISS 只用于找 demonstrations，不用于决定实体是否相同

这是一个容易误用的点。

WDC-PAVE 的向量检索是：

```text
寻找适合放进 prompt 的 few-shot 示例
```

而不是：

```text
embedding 距离近 => 两件商品相同
```

当前 Spec 也应该保持这个边界。

视觉 embedding、文本 embedding、FAISS/HNSW 都可以做：

- few-shot 示例检索；
- 人工复核候选排序；
- reference 缺失时的辅助信息召回；

但不能绕过 reference exact gate。

---

## 3. 论文结果对当前系统有什么启示

WDC-PAVE 数据集来自多网站商品页，包含 565 个 product offers、59 个网站、5 个品类以及数千个经过人工确认的 attribute-value 标注。

论文实验显示：

- GPT-4 在直接属性抽取上可以达到约 90% 级别的 F1；
- 增加 attribute example values 与语义相似 few-shot demonstrations 可以显著提升效果；
- extraction + normalization 可以一步完成；
- 但单独做 normalization 的表现还能更高；
- 对 string wrangling、name expansion 等操作，LLM 具备很强能力。

这给当前方案两个重要启示。

### 3.1 Few-shot 应用于“抽取”，不是用于“自动放行”

90% 多的 F1 对普通属性抽取已经很好，但对“绝不能误匹配”仍远远不够。

即使 precision 是 99%，在 1000 万条数据中也可能产生数量不可接受的错误。

因此 LLM 的角色必须限定为：

```text
候选生成器 / 证据提取器
```

而不是：

```text
最终 matcher
```

### 3.2 抽取与规范化最好拆开

论文实验本身说明 normalization 可以作为独立任务取得非常强的结果。

对腕表更应该拆开：

```text
LLM extraction
      ↓
raw reference candidate
      ↓
deterministic canonicalization
      ↓
registry validation
```

而不是：

```text
LLM：直接告诉我最终 canonical reference
```

拆开以后每一步都能审计，并且可以对每个品牌独立升级规则。

---

## 4. 当前 Spec 推荐的最终系统：Reference Resolver，而不是 Pair Matcher

### 4.1 核心数据模型

建议所有三源记录先统一为：

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

Reference 解析结果：

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
        "REGISTRY",
        "LLM"
    ]
    raw_value: str
    field_name: str | None
    span_start: int | None
    span_end: int | None
    role: str | None
    confidence: float | None
```

这样一条自动匹配记录可以完整回答：

> “为什么系统认为它是 126610LN？”

而不是只有一个不可解释的 0.997 分数。

---

## 5. 第一道：品牌先归一，再解析 reference

reference 不能脱离品牌直接比较。

因为不同品牌可能存在格式相似甚至完全相同的编号字符串。

先建立：

```text
brand_alias
------------------------------
劳力士       -> ROLEX
Rolex        -> ROLEX
ROLEX SA     -> ROLEX
欧米茄       -> OMEGA
Omega        -> OMEGA
```

然后任何 reference key 都必须是：

```text
(brand_id, canonical_reference)
```

而不是只用：

```text
canonical_reference
```

硬规则：

```python
if brand_id is None:
    return ABSTAIN
```

对 precision-first 系统来说，这是值得付出的 recall 损失。

---

## 6. 第二道：Reference Candidate Extraction

建议采用四级抽取，按照“可靠性”而不是“模型 sophistication”排序。

### 6.1 L0：独立结构化 reference 字段

如果来源本身提供独立 reference 字段：

```text
reference_no
model_number
ref
型号
```

先抽取它。

但结构化字段也不能直接信任，因为来源可能把：

- 序列号；
- 平台 SKU；
- 店铺货号；
- 鉴定编号；

错误塞进“型号”字段。

所以仍需 format + registry validation。

### 6.2 L1：品牌规则 / regex / dictionary

对高频品牌维护 reference pattern，例如概念上：

```yaml
ROLEX:
  extract_patterns:
    - "..."
  canonicalization:
    uppercase: true
    remove_spaces: true
    remove_hyphen_if_whitelisted: true

OMEGA:
  extract_patterns:
    - "..."
  canonicalization:
    preserve_dot_grouping: true
```

这里不建议在文档中硬编码一个“万能腕表型号 regex”，因为品牌格式差异太大，而且规则会迭代。

### 6.3 L2：LLM constrained extraction

只有 L0/L1 不能唯一确定时，再调用 LLM。

Prompt 不应该问：

```text
这是什么型号？
```

而应该约束为：

```text
你只能抽取输入文本中实际出现的字符串。
不要补全、猜测、改写或生成一个文本中不存在的 reference。

请识别：
1. 哪些字符串可能是腕表 reference；
2. 每个 candidate 在原文中的精确 span；
3. 它属于当前商品，还是 compatibility / accessory / comparison / mentioned-only；
4. 不确定时返回 abstain=true。
```

返回：

```json
{
  "candidates": [
    {
      "raw": "126610 LN",
      "role": "current_product_reference",
      "evidence": "黑水鬼 126610 LN 全套"
    }
  ],
  "abstain": false
}
```

服务端再做：

```python
assert candidate.raw in original_text
```

如果不在原文，直接丢弃。

这一步能消除一大类 LLM hallucination。

### 6.4 L3：OCR / 图片

当前 Spec 明确说有图片。

图片适合处理：

- 表背刻字；
- 保卡型号；
- 吊牌 reference；
- 发票/附件中的型号文本。

但图片不能直接根据“长得像”判断 reference。

建议：

```text
image
  ↓
OCR / document OCR
  ↓
reference candidate
  ↓
同样进入 role + canonicalization + registry validator
```

只有 OCR 与文本证据一致时提高可信度；视觉相似但 reference 不明确时仍然 ABSTAIN。

---

## 7. 第三道：Reference Role Validation 是防 false positive 的关键

只找出一个“像型号”的字符串还不够。

必须判断：

> 它是不是**当前售卖商品自己的 reference**？

建议 role 至少分：

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

例如：

```text
Rolex 126610LN 原装表带
```

应得到：

```json
{
  "raw": "126610LN",
  "role": "ACCESSORY_FOR_REFERENCE"
}
```

不能得到：

```json
{
  "reference": "126610LN"
}
```

高风险否决词可以做规则层：

```text
适配 / 适用 / compatible with / for
表带 / strap
盒 / box
保卡 / card
配件 / accessory
对比 / vs
```

但不能只靠关键词，因为“全套带盒证”中的“盒”并不表示商品是盒子。

所以更稳妥的是：

```text
规则快速过滤 + LLM role classification + 商品类目一致性检查
```

任何无法明确为 `CURRENT_PRODUCT_REFERENCE` 的候选都不能进入 VERIFIED。

---

## 8. 第四道：Brand-specific Canonicalization，禁止全局暴力清洗

这是本方案与直接照搬论文最重要的区别。

很诱人的写法是：

```python
canonical = re.sub(r'[^A-Z0-9]', '', raw.upper())
```

不推荐全局使用。

因为这样默认：

> 所有标点、空格、分隔符都没有语义。

这是未经证明的。

正确方法是：

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
        value = apply_dot_rules(value)

    return value
```

每个规则都需要：

```text
rule_version
brand
before
canonical
evidence_source
approved_by
```

例如：

```text
126610 LN -> 126610LN
```

只有在确认这种空格是排版差异、不会把两个合法 reference 合并时才进入 production rule。

### 8.1 canonicalization 的安全不变量

上线前对每个品牌做 collision test：

```python
for all_known_references in brand_registry:
    canon = canonicalize(ref)
    assert canon maps to exactly one reference entity
```

如果：

```text
ref_A != ref_B
canonicalize(ref_A) == canonicalize(ref_B)
```

则该规则不能自动上线。

这比单纯看 normalization F1 更适合“绝不能误匹配”的约束。

---

## 9. Reference Registry：把 matching 变成 entity linking

建议建立中心 Reference Registry：

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

每条商品不是先与另外千万条商品 pairwise 比较，而是独立完成：

```text
Product Record
    ↓
Reference Resolver
    ↓
Reference Entity
```

于是：

```text
雷小安 A -> ROLEX / 126610LN
腕表之家 B -> ROLEX / 126610LN
奢当家 C -> ROLEX / 126610LN
```

三者自然属于同一个 entity group。

这使复杂度从潜在的 pairwise 比较降为：

```text
O(N × extraction_cost) + O(1 / logN registry lookup)
```

非常适合 100 万–1000 万级持续增量。

---

## 10. 自动匹配 Gate：必须用硬规则收口

推荐最终 gate：

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

其中 `VERIFIED_REFERENCE` 的进入条件也要严格：

```python
VERIFIED_REFERENCE iff:
  brand is verified
  AND exactly one current-product reference remains
  AND canonicalization is collision-safe
  AND reference exists in trusted registry
  AND no explicit conflicting reference exists
  AND no accessory/compatibility role conflict exists
```

新发现但 registry 不存在的 reference：

```text
SOFT_REFERENCE / REVIEW
```

而不是自动创建 reference entity 后立即合并商品。

---

## 11. 冲突规则：负证据权重应高于正相似度

当前需求 precision 优先，因此推荐 asymmetrical decision policy：

```text
一个强冲突证据 > 多个弱相似证据
```

例如：

```text
title candidate      = 126610LN
structured ref field = 126610LV
```

无论图片多像，结果都应该是：

```text
CONFLICT / ABSTAIN
```

而不是平均两个证据分数。

推荐 hard veto：

```text
1. 两个可信独立字段给出不同 reference；
2. 同一商品存在多个无法解释的 reference；
3. 当前商品角色判定为配件；
4. 品牌与 reference registry 不一致；
5. canonicalization 出现多实体 collision；
6. LLM 输出 reference 不存在于原始输入证据；
7. OCR 与结构化字段发生明确冲突。
```

这类 gate 比训练一个 end-to-end score 更容易审计，也更符合业务风险。

---

## 12. 处理多 reference 标题

这是腕表场景最危险的 corner case 之一：

```text
126610LN / 116610LN 新老款对比
```

或者：

```text
适用 126610LN 126610LV 124060 表带
```

默认策略：

```python
if len(distinct_current_product_candidates) != 1:
    return ABSTAIN
```

除非存在更强证据：

```text
structured reference field = 126610LN
AND title 中 126610LN 被明确标注为本商品
AND 116610LN 只处在 comparison span
```

否则宁可不匹配。

---

## 13. 如何把 WDC-PAVE few-shot 检索改造成腕表专用版本

论文代码以 category 建 FAISS index。

腕表可改成：

```text
一级：brand
二级：source / language / title template
三级：hard-case type
```

训练样本 metadata：

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

示例检索：

```text
current item
   ↓
brand filter = ROLEX
   ↓
semantic retrieval top 20
   ↓
prefer different source
   ↓
hard-case diversity rerank
   ↓
3~5 examples into prompt
```

不要盲目把 top-10 最相似样本全塞进去，因为高度相似的邻近 reference 反而可能诱导模型 copy。

更稳的 few-shot 组成：

```text
1 个正常正例
1 个格式变体正例
1 个 accessory negative
1 个 multiple-reference abstain
1 个 platform-SKU negative
```

也就是 deliberately teach abstention。

---

## 14. Prompt 设计：必须把“不回答”当成一等输出

推荐 system contract：

```text
你是 reference evidence extractor，不是商品匹配器。
你只能引用输入中实际出现的 reference candidate。
禁止根据品牌、系列名称推测一个未出现的型号。
禁止补全残缺 reference。
如果存在多个无法消歧的 candidate，返回 abstain。
如果 reference 只描述配件兼容对象、对比对象或被提及对象，不得标记为 current product reference。
```

输出：

```json
{
  "brand_candidate": "ROLEX",
  "reference_candidates": [
    {
      "raw": "126610 LN",
      "evidence_field": "title",
      "evidence_text": "劳力士黑水鬼 126610 LN 全套",
      "role": "CURRENT_PRODUCT_REFERENCE"
    }
  ],
  "other_identifiers": [
    {
      "raw": "LX202408019338",
      "role": "PLATFORM_SKU"
    }
  ],
  "abstain": false,
  "reason": "single explicit current-product reference"
}
```

模型输出后的 validator：

```python
for c in output.reference_candidates:
    if c.raw not in source_field_text:
        reject(c)
```

再做 pattern 与 registry check。

---

## 15. 不建议让 LLM 做 authoritative normalization

虽然论文说明 LLM normalization 很强，但当前场景不能把它作为最终权威。

错误例：

```text
input:  126610L?
LLM:    126610LN
```

这在普通属性任务可能被视为“合理纠错”，在本需求里却是不可接受的猜测。

推荐分工：

```text
LLM：抽 raw span + role
规则：canonicalize
Registry：验证这个 canonical value 是否是合法实体
```

LLM 可以输出：

```text
normalization_suggestion
```

但只能用于人工复核或 offline rule mining，不能直接修改 canonical key。

---

## 16. 图片在系统中的正确位置

当前数据有图片，但不能把图片相似度提升为 reference 的替代品。

原因很简单：

```text
同系列不同 reference 的腕表外观可能极度相似
```

推荐图片用途：

### A. OCR evidence

```text
表背 / 保卡 / 吊牌图片
        ↓
       OCR
        ↓
reference-like token
        ↓
registry validation
```

### B. Conflict detection

如果文本写：

```text
126610LN
```

但保卡 OCR 高置信度识别出：

```text
126610LV
```

则：

```text
CONFLICT
```

### C. Review ranking

图片 embedding 可以帮助人工把最可能相关的候选排前，但不能产生 `VERIFIED_REFERENCE`。

---

## 17. 百万到千万规模的存储与索引设计

主索引完全可以很朴素：

```sql
CREATE INDEX idx_product_verified_reference
ON product_resolution (brand_id, canonical_reference)
WHERE status = 'VERIFIED_REFERENCE';
```

分组：

```sql
SELECT brand_id, canonical_reference, array_agg(product_id)
FROM product_resolution
WHERE status = 'VERIFIED_REFERENCE'
GROUP BY brand_id, canonical_reference;
```

因为 matching key 最终是 exact key，不需要对 1000 万条做全量 ANN pair retrieval。

向量数据库仅服务：

```text
few-shot retrieval / review assist
```

而不是主匹配索引。

---

## 18. 增量更新架构

建议每条 source record 计算：

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

并记录：

```text
extractor_version
normalizer_version
registry_version
prompt_version
ocr_version
```

处理 key：

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
       │
       ├─ LLM fallback
       │
       └─ OCR fallback
       ▼
Validator / Registry Linker
       │
       ├─ VERIFIED -> exact group index
       ├─ REVIEW   -> human queue
       └─ ABSTAIN  -> storage only
```

如果基础设施简单，MVP 不必一开始上 Kafka：

```text
PostgreSQL + Python worker + Redis/Celery
```

就足够。

数据增长后再替换为：

```text
Kafka / Flink / Spark / warehouse batch backfill
```

关键不是消息队列品牌，而是幂等与版本可追踪。

---

## 19. 建议的 PostgreSQL 最小表结构

```sql
CREATE TABLE product_record (
    id BIGSERIAL PRIMARY KEY,
    source TEXT NOT NULL,
    source_item_id TEXT NOT NULL,
    title TEXT,
    description TEXT,
    brand_raw TEXT,
    reference_raw TEXT,
    content_hash TEXT NOT NULL,
    source_payload JSONB NOT NULL,
    UNIQUE (source, source_item_id)
);

CREATE TABLE product_reference_resolution (
    product_id BIGINT PRIMARY KEY REFERENCES product_record(id),
    brand_id BIGINT,
    reference_entity_id BIGINT,
    canonical_reference TEXT,
    status TEXT NOT NULL,
    has_conflict BOOLEAN NOT NULL DEFAULT FALSE,
    evidence JSONB NOT NULL,
    decision_reason TEXT NOT NULL,
    extractor_version TEXT NOT NULL,
    normalizer_version TEXT NOT NULL,
    registry_version TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_resolution_match
ON product_reference_resolution(brand_id, canonical_reference)
WHERE status = 'VERIFIED_REFERENCE'
  AND has_conflict = FALSE;
```

这个表本身就可以直接支持跨三源聚合。

---

## 20. 人工标注几百对应该标什么

当前 Spec 允许几百对黄金标签。

不建议随机抽几百对，因为随机样本大部分太容易，无法约束 false positive。

应该主动构造 hard-negative benchmark。

至少覆盖：

### A. 同系列近邻 reference

```text
126610LN vs 126610LV
```

### B. reference 格式变体

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

### E. 平台 SKU 冒充型号

字母数字混合但实际是来源内部 ID。

### F. OCR 冲突

title 与保卡/表背 OCR 不一致。

### G. 新品牌 / 新格式

训练集从未出现的 reference pattern。

黄金集主要用于：

1. 评估抽取 precision；
2. 评估 role classification precision；
3. 评估 canonicalization collision；
4. 校准哪些品牌/规则能进入自动路径；
5. 建立 regression suite。

---

## 21. 指标不要只看 F1

当前需求的 dashboard 至少拆成：

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
```

最重要的不是整体 F1，而是：

```text
Auto-match bucket 中出现了多少 false positives？
```

发布门槛应按品牌、来源分别看。

例如某品牌规则还没有充分验证：

```text
只允许 SOFT_REFERENCE
```

而不是因为全局模型 F1 很高就让它自动匹配。

### 关于“几百条标签能否证明 99.99% precision”

需要特别提醒：几百条黄金标签不足以从统计上证明 99.99% 级别的 precision。

因此“极高 precision”不能只靠调一个模型概率阈值实现，必须依赖：

```text
业务真值定义 + deterministic gate + collision-safe normalization + registry + abstention
```

标签主要用于找漏洞、回归测试和扩展 coverage。

---

## 22. 推荐的两阶段上线方案

### Phase 1：不使用 LLM 也能上线的 Precision Core

先做：

```text
品牌归一化
独立 reference 字段
品牌规则抽取
collision-safe normalization
Reference Registry
exact matching
conflict gate
```

只覆盖最确定的记录。

目标不是 coverage，而是先构建“零误合并风格”的可信自动通道。

### Phase 2：WDC-PAVE 风格 LLM Extraction 扩 coverage

对 Phase 1 无法解析的商品：

```text
semantic few-shot retrieval
    ↓
LLM structured extraction
    ↓
span verification
    ↓
role validation
    ↓
normalization
    ↓
registry
```

LLM 的价值是把原本 ABSTAIN 的一部分记录推进到 VERIFIED，而不是改变 VERIFIED 的安全定义。

### Phase 3：OCR 与 active learning

对仍无法解析的高价值商品：

```text
OCR
人工复核
hard-case labeling
规则/示例库更新
```

把人工结果回流到：

```text
reference aliases
negative contexts
few-shot examples
brand policies
regression tests
```

---

## 23. 一套可直接实现的 Python 决策骨架

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
class Candidate:
    raw: str
    role: str
    source: str
    evidence: str

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

    # LLM must quote source text; hallucinated values cannot survive.
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
```

最终匹配：

```python
def same_product(a: Resolution, b: Resolution) -> bool:
    return (
        a.status == Status.VERIFIED
        and b.status == Status.VERIFIED
        and a.brand_id == b.brand_id
        and a.canonical_reference == b.canonical_reference
    )
```

业务逻辑非常容易单元测试。

---

## 24. 测试用例建议

```python
def test_space_variant_can_match():
    assert canonicalize("ROLEX", "126610 LN") == "126610LN"


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
    llm_candidate = "126610LN"
    assert llm_candidate not in text
    assert validate_llm_candidate(text, llm_candidate) is False


def test_normalizer_must_be_collision_free():
    refs = trusted_registry("OMEGA")
    canonical_values = [canonicalize("OMEGA", x) for x in refs]
    assert len(canonical_values) == len(set(canonical_values))
```

特别推荐把每次线上误判都转成永久 regression test。

---

## 25. 成本控制：LLM 不应该跑全部 1000 万条

WDC-PAVE 的实验代码会记录 token 与 API cost，这个思路值得保留。

生产上应分层：

```text
Tier 0：结构化字段 + registry exact
Tier 1：品牌 regex / alias rules
Tier 2：LLM extraction
Tier 3：OCR / manual review
```

只有 Tier 0/1 无法决策的部分进入 LLM。

还可以缓存：

```text
(input_hash, prompt_version, model_version) -> extraction_result
```

内容没变就不重复调用。

对于百万级历史回填：

```text
先规则跑全量
再只把 unresolved subset 发 LLM
```

这样成本和延迟都可控。

---

## 26. 为什么不建议直接训练“两个商品是否相同”的大模型

当前真值已经定义为：

```text
same reference
```

如果训练 pairwise matcher：

```text
商品 A 文本 + 图片
商品 B 文本 + 图片
      ↓
     model
      ↓
   same / different
```

模型很容易学习到：

```text
同系列 + 外观相似 + 标题相似 => same
```

但恰恰对腕表来说：

```text
同系列不同 reference
```

就是最危险的 hard negative。

所以应把问题改写成：

```text
A -> reference entity X
B -> reference entity X
```

而不是：

```text
A ~= B ?
```

这也是本次从 WDC-PAVE 得到的最重要架构启发：**优先解决 identifier extraction + normalization，再做 exact entity linking。**

---

## 27. 与已有调研方向的关系

现有调研里已经覆盖了 Blocking、Entity Matching、图清洗、selective classifier 等多个下游方向。

本方案补的是更上游的一层：

```text
“reference 到底是什么、在哪里、如何安全规范化？”
```

推荐整体组合关系：

```text
WDC-PAVE 风格 Reference Extraction
             ↓
Brand-specific Safe Normalization
             ↓
Reference Registry Entity Linking
             ↓
Exact-match auto group
             ↓
图一致性 / 冲突检测作为审计层
```

而不是把多个 matcher 的分数做加权平均。

---

## 28. 具体落地 API 设计

### 单条解析

```http
POST /v1/reference/resolve
```

Request：

```json
{
  "source": "leixiaoan",
  "source_item_id": "123456",
  "title": "劳力士黑水鬼 41mm 126610 LN 全套",
  "description": "...",
  "brand": "劳力士",
  "reference": null,
  "images": []
}
```

Response：

```json
{
  "brand_id": 1,
  "brand": "ROLEX",
  "canonical_reference": "126610LN",
  "status": "VERIFIED_REFERENCE",
  "evidence": [
    {
      "type": "TITLE",
      "raw": "126610 LN",
      "role": "CURRENT_PRODUCT_REFERENCE"
    },
    {
      "type": "REGISTRY",
      "raw": "126610LN",
      "role": "APPROVED_REFERENCE"
    }
  ],
  "decision_reason": "single current-product reference + approved registry hit"
}
```

### 批量处理

```http
POST /v1/reference/resolve-batch
```

仅负责提交任务，结果写数据库/对象存储。

### 查询同 reference 商品

```http
GET /v1/reference/{brand}/{reference}/products
```

内部只查 exact canonical key。

---

## 29. 人工复核界面最应该展示什么

不要只展示两张商品卡片和一个模型概率。

应该把证据拆开：

```text
品牌：ROLEX

结构化 reference：NULL
标题候选：126610 LN
描述候选：126610LN
OCR 候选：126610LN
平台 SKU：LX202408019338

Canonicalization：
126610 LN -> 126610LN
Rule: ROLEX_SPACE_RULE_V3

Registry：
ROLEX / 126610LN -> APPROVED

冲突：无
模型角色：CURRENT_PRODUCT_REFERENCE
```

人工的操作也应是：

```text
Approve reference
Reject candidate
Mark accessory
Mark platform SKU
Create/approve alias
```

这些动作可以直接反哺规则与 few-shot example store。

---

## 30. 监控与回滚

每次规则、模型、registry 更新后都可能影响历史结果，所以必须支持版本化。

建议每日指标：

```text
verified_count
new_reference_count
unknown_reference_count
abstain_rate
conflict_rate
manual_reject_rate
canonical_collision_count
LLM_non_grounded_output_count
```

如果某次发布后：

```text
manual_reject_rate अचानक上升
```

能够按：

```text
extractor_version / normalizer_version / prompt_version
```

迅速定位并回滚。

所有已自动合并的结果最好保存：

```text
match_decision_version
```

便于重新计算。

---

## 31. 推荐实施优先级

### P0：先建立不可绕过的安全核心

1. brand registry；
2. reference registry；
3. source field mapper；
4. canonicalization policy engine；
5. exact-match gate；
6. conflict / abstain 状态机；
7. audit evidence schema。

### P1：提高 reference coverage

1. brand-specific extractor；
2. WDC-PAVE 风格 structured LLM extraction；
3. semantic few-shot retrieval；
4. hard-negative / abstention demonstrations；
5. span grounding validator。

### P2：图像与反馈闭环

1. OCR；
2. 图像冲突验证；
3. human review UI；
4. active learning；
5. regression benchmark。

---

## 32. 最终建议

WDC-PAVE 证明了一个非常实用的方向：**LLM 可以高质量地从多网站电商脏文本里抽取并规范化属性，而且 few-shot、schema 描述、示例值和语义相似 demonstration 都会显著影响效果。**

但当前二奢/腕表需求不能停在“属性抽取 F1 很高”这一步。

要把它改造成 production-safe 系统，需要再加五层约束：

```text
1. Evidence Grounding
   每个 reference 必须能指回原始字段或 OCR 证据。

2. Role Validation
   reference 必须属于当前商品，不是兼容/配件/对比对象。

3. Brand-specific Collision-safe Normalization
   只做被证明安全的规范化，不做模糊纠错。

4. Trusted Reference Registry
   canonical reference 必须链接到明确实体。

5. Hard Abstention Gate
   任何冲突、多值、不确定都拒绝自动匹配。
```

因此最推荐的落地架构不是：

```text
LLM matcher + similarity threshold
```

而是：

```text
WDC-PAVE 风格抽取器
       +
安全规范化规则
       +
Reference Registry
       +
Exact Match Gate
       +
ABSTAIN / Human Review
```

在 100 万–1000 万级规模下，这个方案同时满足：

- **精度优先**：最终自动合并由可审计的 exact reference gate 决定；
- **字段稀疏**：LLM / OCR 可从非结构化字段补 reference 候选；
- **可扩展**：每条商品独立 entity linking，不做全量 pairwise 笛卡尔积；
- **持续增量**：content hash + pipeline version 支持幂等重跑；
- **可人工闭环**：几百条黄金标签优先覆盖 hard negatives；
- **可解释**：每个匹配都有 raw span、规则版本、registry entity 与 decision reason。

如果只能先实现一个版本，我建议第一版甚至先不追求 LLM 覆盖率：先把 `Reference Registry + safe canonicalization + exact-match gate + abstention` 做稳，然后把 WDC-PAVE 风格 LLM extractor 放在 unresolved records 上逐步扩大 coverage。

这最符合当前 Spec 中“**绝对不能误匹配，允许漏匹配**”的核心约束。