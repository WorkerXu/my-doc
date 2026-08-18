# Entity Matching with 7B LLMs：A Study on Prompting Strategies and Hardware Limitations

> 分析人：d  
> 原论文：Ioannis Arvanitis-Kasinikos, George Papadakis, *Entity Matching with 7B LLMs: A Study on Prompting Strategies and Hardware Limitations*, DOLAP 2025.  
> 论文：https://ceur-ws.org/Vol-3931/paper4.pdf  
> 对应需求：[调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）](https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31)

## 1. 结论先行

这篇论文值得参考，但**不能照搬成“让 7B LLM 直接判断两个商品是不是同款”**。

论文最有价值的三点是：

1. **商品匹配里，Model Number 单一属性（atomic prompt）常常比把名称、features、manufacturer、model number 全部塞给模型更有效。** 原因不是 LLM 更聪明，而是 model number 本身短、区分度高、噪声小；其它长文本属性会把模型带偏。
2. **Few-shot 的样例顺序会产生明显 position bias。** 论文通过把正负样例顺序互换，分别得到 TF / FT 两次判断，再取 intersection，只接受两次都判 True 的 pair，以牺牲 recall 换 precision。
3. **即使做了上述优化，7B LLM 仍普遍“recall 高、precision 低”，容易 false positive。** 在更脏、更稀疏的 Walmart-Amazon 数据集上，论文里 Orca2 + atomic domain-specific prompt 的 precision 约为 0.434，远远不符合当前 Spec 的“绝对不能误匹配”。

因此，针对当前需求，我建议把论文的核心思想改造成一个 **Reference-First / Abstain-by-Default** 架构：

- LLM 不负责最终“是不是同一个商品”的判定；
- LLM 只负责辅助解决“这段文本里的哪个字符串才是品牌参考号、它是不是当前商品自身 reference”这类抽取/角色判别问题；
- 最终自动匹配条件是：`canonical_brand_id + canonical_reference` **严格一致**，并且 reference 的证据链通过高精度规则；
- 任何歧义、冲突、疑似配件、兼容型号、平台 SKU、OCR 不确定，都 **abstain（拒判）**；
- 图片只做独立补证或冲突否决，不允许用视觉“很像”覆盖 reference 不一致。

这个方案和 Spec 的定义完全一致：**“同一个商品”就是同一 reference number / 型号；precision 极端优先，允许漏配。**

---

## 2. 论文到底做了什么

### 2.1 整体框架：Filtering → Verification

论文沿用经典 Entity Resolution 两阶段结构：

```text
原始数据 A × 原始数据 B
        │
        ▼
   Filtering / Blocking
   把笛卡尔积压缩为候选 pair
        │
        ▼
   Verification / Entity Matching
   用 LLM 对每个候选 pair 输出 True / False
```

论文明确只研究 Verification，但实验前仍然用 pyJedAI 做 blocking。

这点对当前 100 万–1000 万量级非常重要：即使最终判定规则很简单，也绝不能做全量跨源笛卡尔积。三个来源如果各有百万级商品，pair 数量会迅速进入 10^12 级别。

不过当前需求比普通 EM 更有利：**最终实体 key 已经被业务定义成 reference**。因此可以进一步把 pairwise matching 改成 key-based grouping，大幅减少“候选 pair”概念本身。

---

### 2.2 论文的 Blocking 实现

论文实验中使用 pyJedAI 0.1.6 的 kNN-Join 做候选生成：

- Abt-Buy：
  - 原始笛卡尔积约 `1.16 × 10^6`
  - 生成 4,345 个 candidate pairs
  - blocking recall = 0.924
  - blocking precision = 0.229
  - `k = 4`
  - 文本做 stop-word removal + stemming
  - 字符三元组（character trigrams）
  - cosine similarity
- Walmart-Amazon：
  - 原始笛卡尔积约 `5.64 × 10^7`
  - 生成 5,163 个 candidate pairs
  - blocking recall = 0.910
  - blocking precision = 0.150
  - `k = 2`
  - 字符四元组（character four-grams）
  - cosine similarity

论文的调参目标是：**blocking recall 至少 90%，在此前提下最大化 blocking precision**。

对当前项目，建议把这个思路保留在“reference 缺失或抽取失败的待复核候选生成”里，但不要作为主路径。主路径应直接通过 canonical reference 分桶。

---

## 3. Prompt 设计：真正值得借鉴的部分

### 3.1 基础 zero-shot

论文最简单的 prompt，本质是：

```text
给你两个 record，判断是不是同一个实体。
只能回答 True / False。
```

这种通用 prompt 最大的问题是模型会自行综合大量弱证据：名称相似、品牌相同、描述类似、规格接近，都可能让它输出 True。

对腕表场景，这是危险的：

- Rolex Submariner 同系列不同 reference 外观极接近；
- 同一个系列可能只有末尾 1–2 位数字不同；
- 表带、表盒、表扣、保卡、零件标题可能带着主表 reference；
- 商家会在标题里写“适配 116610LN / 126610LN”；
- 复刻、改装、替换件也可能完整包含正品 reference。

因此 zero-shot pairwise “same product?” 不应进入自动合并链路。

---

### 3.2 Few-shot + 样例顺序偏差

论文构造两种 few-shot prompt：

- `TF`：先放正例，再放负例；
- `FT`：先放负例，再放正例。

同一个候选 pair 分别请求两次。

随后有两种聚合：

```text
Union:
    TF == True OR FT == True

Intersection:
    TF == True AND FT == True
```

论文发现，对 OpenHermes 和 Zephyr，intersection 往往能显著提高 precision，而且 precision 的提升幅度大于 recall 的损失。

这背后是一条对当前 Spec 很重要的工程原则：

> 当目标极端偏向 precision 时，不要追求“一个强模型给一个高分”，而应要求多个具有扰动差异的判定器一致同意。

但这里不能机械照搬成“两次 LLM 都说 match 就合并”，因为论文最终 precision 仍远低于要求。

更安全的迁移方式是：**把 intersection 用在 reference 证据提取层，而不是最终 match 层。**

例如：

```text
Prompt A：先给“主商品 reference”正例，再给“兼容型号/平台 SKU”负例
Prompt B：交换样例顺序

只有 A、B 都把某个 token 判定为 PRIMARY_REFERENCE，
才允许这个 token 进入 canonicalization。
```

最终是否同款，仍然由 canonical reference exact equality 决定。

---

### 3.3 Atomic domain-specific prompt

论文把商品属性拆成四类：

1. product name
2. features
3. manufacturer
4. model number

然后比较：

- Composite：四个条件全部写进 prompt；
- Atomic：只使用 model number。

论文结果大多数情况下 Atomic 更好，并明确解释：

- model number 短；
- distinctive；
- 干净；
- 能减少 product name 等长文本噪声。

这是本论文对当前项目最直接的启发。

当前 Spec 已经比论文更进一步：业务上直接规定“同 reference 就是同商品”。因此，我们没必要让模型再去“学习”这一判定逻辑。

**正确做法是把难题从 Entity Matching 转成 Reference Extraction + Reference Normalization + Reference Validation。**

---

## 4. 为什么论文方法不能直接上线

### 4.1 论文 precision 与 Spec 不在一个数量级

论文 Table 2 的典型结果：

| 模型 / Prompt | 数据集 | Precision | Recall | F1 |
|---|---:|---:|---:|---:|
| Orca2 + atomic domain-specific | Abt-Buy | 0.689 | 0.934 | 0.793 |
| Orca2 + atomic domain-specific | Walmart-Amazon | 0.434 | 0.708 | 0.538 |
| Orca2 + FT few-shot | Abt-Buy | 0.768 | 0.834 | 0.799 |
| OpenHermes + intersection few-shot | Abt-Buy | 0.683 | 0.718 | 0.700 |
| Zephyr + intersection few-shot | Walmart-Amazon | 0.408 | 0.761 | 0.531 |

对于普通 benchmark，这些结果可以讨论 F1；但对于当前系统，这类 precision 基本没有上线意义。

如果每天自动产生 10 万条匹配：

- precision 99% → 约 1,000 条错配；
- precision 99.9% → 约 100 条错配；
- precision 99.99% → 约 10 条错配。

所以“绝对不能误匹配”要求的系统形态，不应该是“训练一个更高 F1 的 matcher”，而应该是：

> **只自动处理可被硬证据证明的 subset，其余全部拒判。**

---

### 4.2 7B LLM 天然有 false-positive 倾向

论文明确观察到所有测试模型大多呈现：

```text
Recall >> Precision
```

也就是倾向于把候选 pair 判为匹配。

此外，论文预实验里：

- Llama 2 甚至对所有候选都输出 True；
- Mistral 有时不遵循输出格式，会解释、拒答，而不是稳定返回 True/False。

这说明即便模型参数固定，prompt 固定，也不能把 LLM 的单次输出当作数据库约束。

---

### 4.3 数据越脏，LLM 越差

论文在 Walmart-Amazon 上明显比 Abt-Buy 差，作者归因包括：

- missing values 更多；
- record 更长；
- 商品类型更复杂；
- 真匹配占比更低；
- 7B 模型 attention window / 表达能力有限。

这和三源二奢数据非常接近：字段稀疏、标题营销化、来源 schema 不一致、reference 有时埋在文本里。

因此必须先**结构化、抽取、降噪**，不能把 raw record 直接交给 LLM。

---

## 5. 建议落地架构：Reference-First Entity Resolution

## 5.1 总体架构

```text
┌────────────────────────────────────────────────────────────┐
│                 三源 Raw Product Records                  │
│      雷小安 / 腕表之家 / 奢当家，持续增量                │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 1. Source Adapter / Schema Unification                     │
│ source_id, product_id, title, brand, model, desc, images   │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 2. Brand Canonicalization                                  │
│ Rolex / 劳力士 / ROLEX → canonical_brand_id               │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 3. Reference Candidate Extraction                          │
│ A. structured field                                        │
│ B. title / description regex + dictionary                  │
│ C. image OCR                                                │
│ D. LLM role classifier (only ambiguous cases)              │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 4. Reference Validation + Canonicalization                 │
│ brand-specific grammar / alias / checksum-like constraints │
│ primary vs accessory vs compatible vs seller-SKU           │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 5. Evidence Ledger                                         │
│ 保存每个 reference 的 span/source/confidence/conflict      │
└─────────────────────────┬──────────────────────────────────┘
                          │
             ┌────────────┴─────────────┐
             │                          │
             ▼                          ▼
┌───────────────────────┐   ┌───────────────────────────────┐
│ HIGH-CONFIDENCE REF   │   │ AMBIGUOUS / MISSING REF       │
│ exact key grouping    │   │ candidate retrieval / review  │
└───────────┬───────────┘   └───────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────────────────────────┐
│ 6. Hard Match Gate                                         │
│ same canonical_brand_id                                    │
│ AND same canonical_reference                               │
│ AND no conflict                                            │
│ AND reference role = PRIMARY_PRODUCT                       │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 7. Entity Group / Cross-source Cluster                     │
│ entity_key = hash(brand_id, canonical_reference)           │
└────────────────────────────────────────────────────────────┘
```

核心变化是：**不再把每个商品 pair 当作一次二分类问题，而是先给每条 record 生成一个经过审计的 entity key。**

这会同时解决规模与 precision 两个问题。

---

## 5.2 数据模型

建议至少保留以下三张核心表。

### product_record

```sql
CREATE TABLE product_record (
    source              VARCHAR(32) NOT NULL,
    source_product_id   VARCHAR(128) NOT NULL,
    raw_brand           TEXT,
    raw_title           TEXT,
    raw_model           TEXT,
    raw_description     TEXT,
    image_urls          JSONB,
    ingested_at         TIMESTAMP NOT NULL,
    payload_hash        VARCHAR(64) NOT NULL,
    PRIMARY KEY (source, source_product_id)
);
```

### reference_evidence

```sql
CREATE TABLE reference_evidence (
    source              VARCHAR(32) NOT NULL,
    source_product_id   VARCHAR(128) NOT NULL,

    canonical_brand_id  BIGINT,

    raw_reference       VARCHAR(256),
    canonical_reference VARCHAR(256),

    evidence_type       VARCHAR(32) NOT NULL,
    -- STRUCTURED_FIELD / TITLE / DESCRIPTION / OCR / LLM

    reference_role      VARCHAR(32) NOT NULL,
    -- PRIMARY_PRODUCT / COMPATIBLE_MODEL / ACCESSORY_FOR /
    -- PLATFORM_SKU / SELLER_SKU / UNKNOWN

    extractor_version   VARCHAR(64) NOT NULL,
    confidence          NUMERIC(8, 6),
    source_span         TEXT,
    image_id            TEXT,
    created_at          TIMESTAMP NOT NULL
);
```

### resolved_product_key

```sql
CREATE TABLE resolved_product_key (
    source              VARCHAR(32) NOT NULL,
    source_product_id   VARCHAR(128) NOT NULL,

    canonical_brand_id  BIGINT NOT NULL,
    canonical_reference VARCHAR(256) NOT NULL,

    decision            VARCHAR(16) NOT NULL,
    -- AUTO / REVIEWED / ABSTAIN

    decision_reason     TEXT NOT NULL,
    resolver_version    VARCHAR(64) NOT NULL,
    resolved_at         TIMESTAMP NOT NULL,

    PRIMARY KEY (source, source_product_id)
);

CREATE INDEX idx_resolved_product_entity
ON resolved_product_key(canonical_brand_id, canonical_reference)
WHERE decision IN ('AUTO', 'REVIEWED');
```

跨源匹配不需要跑模型：

```sql
SELECT canonical_brand_id,
       canonical_reference,
       array_agg(jsonb_build_object(
           'source', source,
           'product_id', source_product_id
       )) AS records
FROM resolved_product_key
WHERE decision IN ('AUTO', 'REVIEWED')
GROUP BY canonical_brand_id, canonical_reference
HAVING count(DISTINCT source) >= 2;
```

这一步本质是 hash/group-by，复杂度远低于 pairwise matching。

---

## 6. Reference 抽取：从“自由生成”改为“候选 + 验证”

### 6.1 第一优先级：结构化字段

如果来源有明确 `reference/model/reference_number` 字段：

1. 不直接信任；
2. 先判断它是不是平台自己的 SKU；
3. 通过品牌-specific grammar / 已知 reference 字典校验；
4. 保留 raw value 与 canonical value；
5. 如果结构化字段和标题抽取冲突，拒判而不是“选一个更像的”。

高 precision 系统里，**冲突本身就是信号。**

---

### 6.2 第二优先级：规则候选生成

不要让 LLM 从整段标题自由生成 reference。

先用便宜、确定性的规则找 candidate spans：

```python
REF_TOKEN = re.compile(r"(?<![A-Z0-9])[A-Z0-9][A-Z0-9./-]{3,20}(?![A-Z0-9])")
```

然后结合品牌 grammar 过滤。

示意：

```python
def extract_reference_candidates(brand, text):
    tokens = generic_ref_tokens(text.upper())
    grammar = BRAND_REFERENCE_GRAMMAR[brand]
    return [t for t in tokens if grammar.maybe_valid(t)]
```

品牌 grammar 不一定只是一条 regex，可以是：

```python
class ReferenceGrammar:
    allowed_patterns: list[Pattern]
    deny_patterns: list[Pattern]
    known_prefixes: set[str]
    min_len: int
    max_len: int
    normalization_rules: list[Rule]
```

---

### 6.3 第三优先级：LLM 做“角色分类”，不要做“自由抽取”

把 LLM 任务改成：

> 给定品牌、标题、已由规则抽出的候选 token，判断每个 token 的语义角色。

例如输入：

```json
{
  "brand": "ROLEX",
  "title": "劳力士黑水鬼表带 适配116610LN 126610LN 原装扣",
  "candidates": ["116610LN", "126610LN"]
}
```

输出只能是：

```json
{
  "116610LN": "COMPATIBLE_MODEL",
  "126610LN": "COMPATIBLE_MODEL"
}
```

而不是：

```json
{"reference": "116610LN"}
```

这能防止 LLM 把“适配对象 reference”误当作“当前商品 reference”。

---

## 7. 把论文的 TF / FT intersection 改造成高精度 Reference Role Gate

论文的 intersection 思想可以直接迁移：

### Prompt A

- 示例 1：当前商品自身 reference → PRIMARY_PRODUCT
- 示例 2：适配型号 → COMPATIBLE_MODEL
- 示例 3：店铺货号 → SELLER_SKU

### Prompt B

相同示例，但完全倒序。

对于同一个 candidate token：

```python
role_a = llm_role_prompt_a(record, token)
role_b = llm_role_prompt_b(record, token)

if role_a == role_b == "PRIMARY_PRODUCT":
    role = "PRIMARY_PRODUCT"
else:
    role = "UNKNOWN"
```

这比论文直接：

```text
True ∩ True => Match
```

安全得多，因为最终 Match 还有下一层 hard rule。

可以进一步加入第三个独立判定器：

```text
Rule classifier
LLM prompt A
LLM prompt B
```

只有：

```text
Rule != CONTRADICTION
AND A == PRIMARY_PRODUCT
AND B == PRIMARY_PRODUCT
```

才让 reference 进入自动匹配池。

---

## 8. Canonicalization：只做“可逆、安全”的归一化

这是最容易把高 precision 系统做坏的地方。

### 8.1 可以做的 normalization

通常安全：

- trim；
- Unicode NFKC；
- 英文字母 uppercase；
- 明确等价的全角/半角字符；
- 品牌已验证的 separator 规则；
- OCR 中有确定性证据时的字符修正。

例如：

```text
" 126610LN " → "126610LN"
"126610-LN"  → 只有品牌字典明确两者等价时才可变为 "126610LN"
```

### 8.2 不应该默认做的 normalization

危险：

- 无条件删掉所有 `- / .`；
- 无条件去掉前导 0；
- `O ↔ 0`、`I ↔ 1` 自动互换；
- 模糊编辑距离后“选最近型号”；
- 只因为字符串 90% 相似就归一到同一个 reference。

原则：

> **canonicalization 必须由品牌知识证明“这两种写法是同一个 reference 的格式变体”，而不是由相似度猜出来。**

建议维护：

```sql
reference_alias(
    canonical_brand_id,
    alias,
    canonical_reference,
    alias_type,
    evidence,
    approved_by,
    version
)
```

只有已审核 alias 才进入自动归一化。

---

## 9. 最终自动匹配 Gate

建议把自动匹配写成非常保守、可测试的纯函数。

```python
def resolve_key(record) -> ResolveResult:
    brand = canonicalize_brand(record)
    if not brand.is_high_confidence:
        return Abstain("brand_uncertain")

    evidences = collect_reference_evidence(record, brand)

    primary = [
        e for e in evidences
        if e.role == "PRIMARY_PRODUCT"
        and e.canonical_reference is not None
        and e.trust_level >= TRUST_HIGH
    ]

    refs = {e.canonical_reference for e in primary}

    if len(refs) == 0:
        return Abstain("reference_missing")

    if len(refs) > 1:
        return Abstain("reference_conflict")

    ref = next(iter(refs))

    if has_negative_context(record, ref):
        return Abstain("reference_in_negative_or_compatibility_context")

    if looks_like_accessory(record):
        return Abstain("accessory_risk")

    return Auto(
        brand_id=brand.id,
        canonical_reference=ref,
        reason="high_confidence_primary_reference"
    )
```

跨源同款：

```python
def same_product(a, b):
    if a.decision != "AUTO" or b.decision != "AUTO":
        return False

    return (
        a.canonical_brand_id == b.canonical_brand_id
        and a.canonical_reference == b.canonical_reference
    )
```

这里没有模型 similarity threshold。

---

## 10. 图片怎么用

Spec 明确说有图片。

图片最适合做两件事：

### 10.1 OCR 补 reference

优先 OCR：

- 保卡；
- 吊牌；
- 表背；
- 盒贴；
- 证书；
- 品牌标签。

但 OCR 结果仍然走：

```text
candidate extraction
→ brand grammar
→ role validation
→ canonicalization
→ conflict gate
```

OCR 不直接产生 match。

### 10.2 Conflict veto

如果文本 reference = A，但图中高可信 OCR reference = B：

```text
A != B => ABSTAIN
```

而不是让“文本模型”和“视觉模型”加权投票。

对 precision-first 系统，图片的最大价值往往是**否决**，不是“加一点 similarity 分”。

---

## 11. reference 缺失时怎么办

这里必须坚持业务定义。

如果记录没有足够可信的 reference，哪怕：

- 品牌完全一致；
- 系列一致；
- 标题高度相似；
- 图片几乎一模一样；
- 价格接近；

也不应自动合并。

可以做 candidate retrieval 给人工：

```text
brand exact
+ series exact/alias
+ title char n-gram / BM25
+ image embedding
+ candidate reference dictionary
```

输出 Top-K：

```json
[
  {"candidate_reference": "126610LN", "evidence": [...]},
  {"candidate_reference": "116610LN", "evidence": [...]}
]
```

但状态必须是：

```text
NEEDS_REVIEW
```

不能写入自动 entity group。

---

## 12. 规模设计：1M–10M + 持续增量

### 12.1 为什么不用全量 pairwise

如果三个源分别约 300 万、300 万、300 万：

```text
A × B + A × C + B × C
≈ 27 × 10^12 candidate pairs
```

任何 LLM matcher 都不可行。

Reference-First 方案把过程变成：

```text
每条 record 独立解析一次 reference
O(N)

随后按 (brand, reference) 建索引 / group-by
O(N log N) 或近似 O(N)
```

增量数据只需要解析新/变更 record，不必重跑历史 pair。

---

### 12.2 推荐技术栈

一个务实版本：

```text
Batch / Backfill:
Python + Polars / Spark

Online / Incremental Worker:
Python + FastAPI worker / Celery / Kafka consumer

Storage:
PostgreSQL（10M 量级足够做 key 索引与审计元数据）
对象存储/Parquet 保存 raw snapshot

OCR:
PaddleOCR / 云 OCR / 现有视觉模型

LLM:
本地 7B/8B quantized model，仅处理 ambiguous subset
或线上结构化输出模型

Metrics:
Prometheus + Grafana
```

如果抓取是批量定时任务，不一定一开始就上 Kafka。

可以先：

```text
crawler table
→ incremental SQL scan by updated_at
→ resolver workers
→ resolved_product_key
```

只有吞吐和延迟真的成为问题，再引入消息队列。

---

## 13. 增量处理与幂等

用 `payload_hash` 检测内容变化：

```python
if incoming.payload_hash == stored.payload_hash:
    skip()
else:
    rerun_reference_resolution()
```

每次解析保存：

```text
extractor_version
resolver_version
prompt_version
reference_alias_version
```

这样更新规则后可以定向重跑：

```sql
SELECT *
FROM product_record p
JOIN resolved_product_key r USING (source, source_product_id)
WHERE r.resolver_version < '2026-08-18-v3';
```

如果一个商品原来 AUTO，后来发现冲突，应支持撤销 entity key：

```text
AUTO → ABSTAIN
```

而不是只增不减。

---

## 14. 黄金标签怎么用最划算

Spec 允许人工标注几百对。

不建议把几百对主要拿去训练一个 pairwise matcher。

更有效的是做 **hard-case calibration set**：

### 标签桶

1. 同 reference、不同格式；
2. 同系列、相邻 reference；
3. 配件标题里出现主表 reference；
4. “适配/兼容”多个 reference；
5. 平台 SKU 长得像品牌 reference；
6. OCR `O/0, I/1, B/8` 混淆；
7. 标题 reference 与结构化字段冲突；
8. 品牌别名/翻译；
9. 同 reference 但信息极稀疏；
10. reference 缺失但图像相似。

其中 hard negatives 比随机 negatives 更重要。

---

## 15. precision 的统计验收：几百条标签能证明到什么程度

“绝对不能错”无法仅靠有限测试集数学证明。

如果希望在 95% 置信度下，观察到 0 个 false positive 后，把真实 false-positive rate 上界压到：

- 1% 以下，大约需要 299 个无误样本；
- 0.1% 以下，大约需要 2,995 个无误样本；

近似来自：

```text
(1 - p)^n <= 0.05
```

所以“几百对黄金标签”适合：

- 找规则漏洞；
- 选择阈值；
- 验证是否能达到约 99% 级别的统计下界；

但不足以证明 99.99%。

因此工程上应该把风险控制主要建立在**硬约束 + 拒判**上，而不是寄希望于测试集里恰好没碰到 false positive。

---

## 16. 推荐的线上指标

不要只监控 F1。

至少监控：

```text
auto_match_count
abstain_rate
reference_extraction_rate
reference_conflict_rate
structured_vs_title_conflict_rate
ocr_vs_text_conflict_rate
accessory_risk_rate
manual_review_precision
posthoc_false_positive_count
new_reference_rate
unknown_brand_rate
```

最关键的是：

```text
AUTO_MATCH_FALSE_POSITIVE
```

只要发现一条，就应触发：

1. 回溯该规则版本产生的所有 AUTO 结果；
2. 查找同模式 record；
3. 临时加 deny rule；
4. 将相关 bucket 降级为 ABSTAIN；
5. 修复后再恢复。

---

## 17. 论文中 7B LLM 的硬件启发

论文实验环境：

- Ubuntu 22.04.1；
- Intel i7-9700K；
- 32GB RAM；
- NVIDIA GTX 1080 Ti 11GB；
- Python 3.12；
- Ollama 0.1.22；
- 7B 级模型；
- 4-bit quantization。

说明 reference role classifier 完全可以在较低成本 GPU 上本地部署。

但当前系统不应该“每个候选 pair 调一次 LLM”。

应该只把 LLM 用在小比例 ambiguous records：

```text
1000 万 records
假设 85% structured/rule 直接解决
10% 明确缺失直接 abstain
仅 5% 进入 LLM role classification
=> 50 万次 record-level 调用
```

并且同一条 record 只解析一次，结果缓存；不随跨源 pair 数增长。

---

## 18. 可直接落地的服务拆分

```text
brand-normalizer
    输入 raw_brand
    输出 canonical_brand_id

reference-candidate-extractor
    输入 brand + text fields
    输出 candidate spans

reference-role-classifier
    输入 candidate + local context
    输出 PRIMARY_PRODUCT / COMPATIBLE / SKU / UNKNOWN

reference-canonicalizer
    输入 brand + raw ref
    输出 canonical ref / invalid

image-reference-worker
    输入 images
    OCR → evidence

resolver
    聚合 evidence
    输出 AUTO / ABSTAIN

entity-index
    key = (brand_id, canonical_reference)
    挂载各来源 record
```

服务之间尽量传结构化 evidence，而不是反复传整段商品文本。

---

## 19. API 设计示例

### Resolve 单条商品

```http
POST /v1/resolve-reference
```

```json
{
  "source": "leidangjia",
  "source_product_id": "123456",
  "brand": "劳力士",
  "title": "劳力士潜航者 126610LN 黑盘 全套",
  "model": null,
  "description": "...",
  "images": ["..."]
}
```

返回：

```json
{
  "decision": "AUTO",
  "canonical_brand_id": 101,
  "canonical_reference": "126610LN",
  "evidence": [
    {
      "type": "TITLE",
      "raw": "126610LN",
      "role": "PRIMARY_PRODUCT"
    }
  ],
  "conflicts": [],
  "resolver_version": "2026-08-18-v1"
}
```

歧义场景：

```json
{
  "decision": "ABSTAIN",
  "reason": "multiple_reference_candidates_in_compatibility_context",
  "evidence": [
    {"raw": "116610LN", "role": "COMPATIBLE_MODEL"},
    {"raw": "126610LN", "role": "COMPATIBLE_MODEL"}
  ]
}
```

---

## 20. Prompt 示例：严格 reference role 分类

System：

```text
你不是商品匹配器。
你的唯一任务是判断候选编号在当前商品文本中的语义角色。

允许标签：
- PRIMARY_PRODUCT：当前售卖商品自身的品牌型号/reference
- COMPATIBLE_MODEL：当前商品适配/兼容的其它商品型号
- ACCESSORY_FOR：当前商品是配件，编号指向被配套主商品
- PLATFORM_SKU：平台/店铺内部货号
- UNKNOWN：证据不足

禁止根据“看起来像型号”推断 PRIMARY_PRODUCT。
如果存在“适配、兼容、for、适用于、替换”等关系，优先判 COMPATIBLE_MODEL 或 ACCESSORY_FOR。
只能返回 JSON。
```

User：

```json
{
  "brand": "ROLEX",
  "candidate": "126610LN",
  "title": "原装表带 适配劳力士 126610LN 116610LN",
  "model_field": null
}
```

预期：

```json
{
  "role": "ACCESSORY_FOR",
  "confidence": "high"
}
```

然后使用论文的 intersection 思想，把正负样例顺序互换再请求一次；两次结果不一致就 UNKNOWN。

---

## 21. 防止“同系列相邻 reference”误并的专项规则

腕表最危险的 false positive 就是相邻型号。

建议把 reference exact equality 放在所有 similarity 之前：

```python
if ref_a and ref_b and ref_a != ref_b:
    return DEFINITE_NON_MATCH
```

注意：这里必须是**安全 canonicalization 之后**的 ref。

例如：

```text
116610LN != 126610LN
```

即使：

```text
brand = Rolex
series = Submariner
color = black
image cosine = 0.99
```

也绝不能合并。

视觉模型只能在 ref 缺失时帮助人工找 candidate，不能覆盖这个规则。

---

## 22. 多源聚类：不要让错误边传递

传统多源 ER 常见做法：

```text
A matches B
B matches C
=> A/B/C 一个 cluster
```

但这种传递会放大一条 false-positive edge。

当前需求可以避免这个问题：

```text
cluster key = (canonical_brand_id, canonical_reference)
```

只有拥有相同硬 key 的 record 才进入同 cluster。

这意味着无需做复杂的 correlation clustering，且不会因为某一条模糊相似边污染整簇。

---

## 23. 与论文方法的逐项映射

| 论文 | 当前项目改造 |
|---|---|
| Filtering / Blocking | 主路径改成 brand+reference 分桶；仅对 ref 缺失记录做候选召回 |
| Pairwise LLM True/False | 禁止作为最终自动 match |
| Atomic model-number prompt | 升级为 reference-first 体系 |
| Composite attributes | 只用于辅助识别 reference 语义角色，不参与最终 key |
| TF / FT | 用于 reference role classifier 的 prompt perturbation |
| Intersection | 两次都判 PRIMARY_PRODUCT 才接纳该 evidence |
| 7B 4-bit LLM | 只处理 ambiguity subset，可低成本本地部署 |
| F1 优化 | 改为 precision + abstain coverage 优化 |
| model outputs match | 最终输出改成 auditable entity key |

---

## 24. MVP 分阶段实施

### Phase 0：两天内可完成的 baseline

1. 三源字段 mapping；
2. 品牌 canonicalization；
3. 统计已有 structured model/reference 字段覆盖率；
4. 对 structured reference 做 NFKC + uppercase + trim；
5. 仅对 `brand + exact raw/canonical ref` 严格一致的商品聚类；
6. 人工抽查 false positive。

这一步就可以得到一个“极高 precision、低 recall”的 baseline。

### Phase 1：标题 reference 抽取

1. 建每个品牌 candidate regex；
2. 收集已知 reference 字典；
3. 标记兼容词、配件词、平台 SKU 上下文；
4. title/structured 同 ref 才提高 trust；
5. 冲突直接 abstain。

### Phase 2：LLM ambiguity resolver

1. 做 PRIMARY_PRODUCT / COMPATIBLE / ACCESSORY / SKU role 分类；
2. TF / FT 双 prompt intersection；
3. 只处理规则不能决定的 candidate；
4. LLM 结果不能单独触发 AUTO。

### Phase 3：Image OCR

1. 先识别高价值图片类型；
2. OCR reference；
3. 与文本 evidence 交叉验证；
4. 图片冲突触发 abstain。

### Phase 4：黄金标签 + 风险控制

1. 几百条 hard cases；
2. 每个规则 bucket 独立测 precision；
3. 只上线零 false-positive bucket；
4. 收集线上人工复核回流；
5. 定期扩大验证集。

---

## 25. 建议的上线策略

不要设置一个统一：

```text
score > 0.95 => match
```

而是分“可证明的规则桶”：

### Bucket A：最安全

```text
同 brand
+ 两边 structured reference 都存在
+ canonical reference exact equal
+ 无冲突
```

先上线。

### Bucket B：较安全

```text
一边 structured ref
+ 另一边 title ref
+ exact equal
+ title ref 通过 PRIMARY role
+ 无配件/兼容上下文
```

独立验证后上线。

### Bucket C：多证据

```text
title ref
+ OCR ref
+ 两者一致
+ exact equal with other source
```

验证后上线。

### Bucket D：只有 LLM 推断

```text
永不自动上线，保持 REVIEW / ABSTAIN
```

这样可以逐桶扩大 coverage，而不破坏总体 precision。

---

## 26. 最终建议

如果只从这篇论文拿一个结论，我会拿这个：

> **不要让丰富属性把最关键的 identifier 证据淹没。**

对当前系统，reference 不是普通 feature，而是业务定义的实体 identity key。

因此最合理的架构不是：

```text
文本 + 图片 + 属性
→ 多模态 embedding / LLM
→ similarity score
→ threshold
→ match
```

而是：

```text
文本/结构化字段/图片
→ 找 reference 候选
→ 判断候选是不是“当前商品自己的 reference”
→ 品牌级安全 canonicalization
→ 冲突检测
→ 得到可信 (brand, reference)
→ exact key grouping
```

LLM 只解决“不结构化数据变成可信 key”过程中的局部语义难题。

这比直接做 pairwise LLM Entity Matching 更符合：

- 100万–1000万规模；
- 持续增量；
- 字段稀疏；
- reference 可能埋在标题；
- 可用图片；
- 几百条黄金标签；
- precision 极端优先、允许漏配。

**推荐直接以 Reference-First Resolver 作为 MVP 主线，而把论文里的 7B LLM + atomic prompt + intersection 作为 reference role validation 的辅助组件，而不是最终 matcher。**

---

## 参考资料

1. Entity Matching with 7B LLMs: A Study on Prompting Strategies and Hardware Limitations  
   https://ceur-ws.org/Vol-3931/paper4.pdf
2. pyJedAI  
   https://github.com/AI-team-UoA/pyJedAI
3. 当前需求 Spec  
   https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31
