# Entity Matching with 7B LLMs: A Study on Prompting Strategies and Hardware Limitations

## 1. 调研对象与结论先行

- 论文：**Entity Matching with 7B LLMs: A Study on Prompting Strategies and Hardware Limitations**
- 作者：Ioannis Arvanitis-Kasinikos, George Papadakis
- 会议：DOLAP 2025
- 原文：https://ceur-ws.org/Vol-3931/paper4.pdf
- 对应需求：跨源二奢/腕表商品实体匹配，数据量 100 万–1000 万，持续增量；“同一个商品”定义为**同一 reference number / 型号**；precision 极端优先，允许漏匹配；字段稀疏，reference 可能在结构化字段、标题甚至图片中。

这篇论文最值得迁移的不是“用 7B LLM 直接做商品 pair matching”，而是下面三个结论：

1. **只用 model number 的 atomic domain-specific prompt，通常优于把 product name、features、manufacturer、model number 全部塞进去的 composite prompt。** 对当前需求而言，这正好支持“reference 是唯一主判据，其他属性最多做否决证据”的架构。
2. **few-shot 的样例顺序会导致 position bias；对两个顺序分别推理并取交集，可以显著提高 precision。** 这可以直接迁移成 reference 提取/角色判定的保守双判机制。
3. **7B LLM 在商品 EM 上总体表现为 recall 高于 precision，容易产生 false positive。** 因而不能把 LLM 的“语义相似”结果作为自动合并依据；在本需求里，LLM 应只负责把脏文本变成候选 reference 证据，并且必须允许 abstain（拒识）。

因此建议将最终系统设计为：

> **Reference Extraction & Validation → Canonicalization → Exact Key Match → Conflict Veto → Abstain / Manual Review**

而不是：

> 文本/图片 embedding → 相似度 → LLM 判断是否同款。

后者会天然违背“绝对不能误匹配”的约束。

---

## 2. 论文技术实现拆解

### 2.1 总体流程：Filtering → Verification

论文沿用实体解析常见的两阶段框架：

1. **Filtering / Blocking**：先从笛卡尔积中召回少量候选 pair。
2. **Verification / Entity Matching**：再判断候选 pair 是否是同一实体。

论文在两个商品数据集上使用 pyJedAI 的 kNN Join 做 blocking：

- Abt-Buy：k=4，字符 trigram，多集表示，cosine similarity。
- Walmart-Amazon：k=2，字符 four-gram，多集表示，cosine similarity。
- blocking recall 调到至少约 90%。

论文之后才把候选 pair 送给 LLM。

这个设计对通用商品 EM 是合理的，但对当前腕表 Spec 可以进一步简化：**当 reference 已被高可信提取后，根本不需要对全量商品做语义近邻 blocking，只需要对 `(brand_id, canonical_reference)` 做精确索引/Join。** 向量检索只应该服务于“reference 未确认”的人工复核或候选提示，而不能产生自动 match edge。

### 2.2 模型与部署

论文实验环境：

- Python 3.12
- Ollama 0.1.22
- Ubuntu 22.04
- Intel i7-9700K
- 32 GB RAM
- GTX 1080 Ti 11 GB VRAM
- 7B 级模型
- 4-bit quantization

论文尝试多种 7B/8B 开源模型，最终重点比较 Orca2、OpenHermes、Zephyr 等。核心工程点是：**4-bit 量化使商品 EM 可以在单张消费级 GPU 上运行。**

迁移到当前系统时，不必绑定论文中的具体模型。重要的是保留它的部署形态：

- 小模型本地部署；
- 结构化输出；
- 低成本重复推理；
- 只处理少量“reference 不确定记录”，而非 1000 万记录全量 pair inference。

这样 LLM 成本可被压缩到总数据的很小比例。

### 2.3 Prompt 设计

论文比较三类策略。

#### A. 通用 Zero-shot

输入两个 record，要求输出 True / False。

问题：模型容易把“相似”理解成“同一”，造成大量 false positive。

#### B. Few-shot

加入一对正例和一对负例。论文发现示例顺序会影响结果：

- TF：先 True 示例，再 False 示例；
- FT：先 False 示例，再 True 示例。

针对顺序偏置，论文定义：

- Union：TF 或 FT 任意一个判 True 就接受；
- Intersection：TF 和 FT 都判 True 才接受。

对 OpenHermes、Zephyr，Intersection 通常以较小的 recall 损失换来明显 precision 提升。

这个思路非常适合本需求，但需要把判定任务从“两个商品是否相同”改成更窄的原子任务：

> “字符串 X 是否是当前售卖腕表本体的品牌 reference，而不是平台 SKU、库存号、兼容型号、配件适用型号或其他编号？”

然后对两个 few-shot 顺序各跑一次，只在两个结果都同意且证据一致时，才接受该 reference。

#### C. Domain-specific Prompt

论文对商品设置四个字段：

- product name
- features
- manufacturer
- model number

又比较两种版本：

- Composite：四个字段一起比较；
- Atomic：**只用 model number。**

多数情况下 Atomic 更好。论文解释是 model number 短、区分度高、噪声少，而长标题和 features 会把额外噪声引入决策。

这与腕表需求高度一致：

- 品牌、系列、材质、尺寸、机芯、图片外观都可能“很像”；
- 相邻 reference 外观甚至完全近似；
- 只有 reference 本身满足需求定义中的身份条件。

所以系统不应训练一个“多模态综合相似度大于阈值即同款”的自动合并器，而应训练/实现一个**reference 证据系统**。

---

## 3. 论文结果对当前需求的直接启示

论文最关键的负面结果其实比正面结果更重要：7B LLM 在两个商品数据集上普遍是 **recall 明显高于 precision**，也就是更容易多报 match。

例如论文 Table 2 中，Orca2 在 Walmart-Amazon 上：

- Zero-shot precision 约 0.397；
- Atomic domain-specific precision 约 0.434；
- 即便是最好配置，也远达不到“绝对不能误匹配”。

因此：

> **LLM pair matcher 不应该进入自动 merge 的信任边界。**

可以进入信任边界的是：

1. 独立 reference 证据；
2. 品牌内 canonical reference 严格一致；
3. 没有任何冲突证据；
4. reference 角色已确认是“售卖商品本体 reference”；
5. 必要时多路独立证据交叉一致。

换句话说，LLM 最适合做的是“把非结构化内容变成结构化候选”，而不是“替业务定义做最终裁决”。

---

## 4. 推荐直接落地架构

### 4.1 数据流

```text
雷小安 / 腕表之家 / 奢当家
        │
        ▼
[Raw Offer Ingestion]
        │
        ▼
[字段标准化 + 品牌归一]
        │
        ▼
[Reference Candidate Extractor]
   ├─ 结构化字段
   ├─ 标题/描述规则
   ├─ 图片 OCR
   └─ LLM 原子抽取（仅疑难）
        │
        ▼
[Reference Role Validator]
   ├─ brand reference
   ├─ platform SKU
   ├─ seller/internal ID
   ├─ compatibility/reference of another product
   ├─ accessory reference
   └─ unknown
        │
        ▼
[Brand-specific Canonicalizer]
        │
        ▼
[Evidence Aggregator + Conflict Veto]
        │
   ┌────┴─────────────┐
   │                  │
可信唯一 reference   不确定/冲突
   │                  │
   ▼                  ▼
Exact Key Index      Abstain / 人工复核
(brand_id, ref)       │
   │                  └─ 回流规则/少量训练数据
   ▼
跨源 exact join
   │
   ▼
Entity Cluster / Match Edge
```

### 4.2 为什么这个架构比“先召回候选 pair 再 LLM 判断”更适合

数据规模 100 万–1000 万，但真正的业务 identity key 已经明确是 reference。

若一个 record 能可靠得到：

```text
brand_id = rolex
canonical_reference = 126610LN
```

则跨源匹配可以退化为数据库中极便宜、可解释、可审计的精确 join：

```sql
SELECT a.id, b.id
FROM source_a a
JOIN source_b b
  ON a.brand_id = b.brand_id
 AND a.canonical_reference = b.canonical_reference
WHERE a.reference_status = 'trusted'
  AND b.reference_status = 'trusted';
```

这比对千万记录构造 embedding、ANN、pairwise LLM 判断更安全，也更便宜。

---

## 5. Reference 抽取层设计

### 5.1 每条记录不要只存一个 reference 字符串

建议先存“证据事件”，再生成最终 assignment。

```sql
CREATE TABLE reference_evidence (
    id BIGSERIAL PRIMARY KEY,
    offer_id BIGINT NOT NULL,
    source VARCHAR(32) NOT NULL,
    evidence_type VARCHAR(32) NOT NULL,
    raw_value TEXT NOT NULL,
    normalized_candidate TEXT,
    span_text TEXT,
    role VARCHAR(32),
    confidence NUMERIC(6,5),
    extractor_version VARCHAR(64),
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

`evidence_type` 示例：

- `structured_field`
- `title_regex`
- `description_regex`
- `ocr_caseback`
- `ocr_card`
- `ocr_tag`
- `llm_title`
- `llm_ocr_validation`

这样后续所有自动匹配都可以追溯“为什么认为它是这个 reference”。

### 5.2 证据优先级

建议默认优先级：

1. 平台明确标注且已验证字段语义的 reference/model 字段；
2. 品牌特定 pattern 命中的标题片段；
3. 保卡/吊牌 OCR；
4. 表背 OCR；
5. 描述文本；
6. LLM 推断。

LLM 不能覆盖更高优先级的冲突证据。

### 5.3 只抽“候选”，不让模型自由生成

LLM 输入中必须要求：

- reference 必须能在输入文本/OCR 中找到原始 span；
- 禁止模型根据品牌知识猜型号；
- 找不到则返回 null；
- 多个候选全部返回，不自行选一个。

建议输出：

```json
{
  "candidates": [
    {
      "value": "126610LN",
      "span": "劳力士潜航者 126610LN",
      "role": "product_reference",
      "reason": "explicit token after model family"
    }
  ]
}
```

如果输出的 `value` 无法从原始文本做字符级回溯，则直接丢弃。

---

## 6. 从论文 Intersection Few-shot 迁移出的“双判交集”

论文证明 few-shot 存在 position bias，因此本系统可以针对**reference role validation**做两次独立推理。

### Prompt A：正例在前

- 正例：标题中的 `126610LN` 是售卖腕表本体 reference；
- 负例：`适用 126610LN 的表带` 中 `126610LN` 是兼容对象 reference，不是当前商品 reference。

### Prompt B：负例在前

相同例子，顺序反转。

只有：

```text
A.role == product_reference
AND B.role == product_reference
AND A.value == B.value
```

才把 LLM 证据记为 positive。

任何不一致：

```text
role = unknown
status = abstain
```

注意：这仍然不能单独触发自动跨源 merge，它只是在 reference 证据层提高 precision。

---

## 7. Reference Role Validator：当前项目最值得新增的一层

实际二奢数据里，最危险的问题往往不是“没抽到 reference”，而是**抽到了一个看起来像 reference 的编号，但它不是当前售卖商品的 reference**。

建议明确分类：

```text
PRODUCT_REFERENCE
PLATFORM_SKU
SELLER_INTERNAL_ID
COMPATIBILITY_REFERENCE
ACCESSORY_REFERENCE
SERIAL_NUMBER
CALIBER_NUMBER
UNKNOWN
```

必须重点防以下 hard negatives：

### 7.1 配件兼容型号

```text
适配 Rolex 126610LN 黑水鬼表带
```

如果当前商品是表带，`126610LN` 不能把它与腕表本体合并。

### 7.2 盒证/附件

```text
126610LN 原装盒 / 保卡 / 吊牌
```

标题中存在正确腕表 reference，但商品实体不是腕表。

### 7.3 平台 SKU / 商家货号

平台字段可能存在：

```text
货号：126610LN-82937
SKU：126610LN
```

仅凭字符串像 reference 不能认为它就是品牌 reference。

### 7.4 一条记录出现多个 reference

例如卖家写：

```text
126610LN / 116610LN 同款配件
```

自动选择一个风险极高，应直接 abstain。

### 7.5 机芯号与 reference 混淆

腕表描述中机芯编号也常为字母数字串，必须单独分类，不能进入 reference key。

---

## 8. Canonicalization：宁可少归一，不要过度归一

目标不是把所有相似字符串压成一样，而是只消除明确无语义差异的格式噪声。

建议 canonicalizer 按品牌配置：

```python
def canonicalize_reference(brand_id, raw):
    x = unicode_nfkc(raw)
    x = x.strip().upper()
    x = normalize_known_spaces(x)
    x = apply_brand_rules(brand_id, x)
    return x
```

不要默认做下面这些危险操作：

- 全局删除所有 `-`、`.`、`/`；
- 删除所有前导零；
- 模糊编辑距离合并；
- 把 `O` 和 `0`、`I` 和 `1` 自动互换；
- 只保留数字；
- reference prefix 截断。

如果某品牌确实存在等价格式，例如官方/平台经常把某处分隔符写法混用，应维护**品牌级确定性 rewrite rule**，并有版本号和回归测试。

---

## 9. 最终 Trusted Reference 状态机

建议每条商品得到一个状态，而不是一个裸字符串。

```text
NO_REFERENCE
CANDIDATE
CONFLICTED
TRUSTED_SINGLE
TRUSTED_MULTI_NOT_ALLOWED
REJECTED_ROLE
```

自动匹配只允许：

```text
status == TRUSTED_SINGLE
```

`TRUSTED_SINGLE` 的最低条件建议：

1. 品牌已 canonicalize；
2. 只有一个 canonical reference；
3. role = PRODUCT_REFERENCE；
4. 至少一个高信任证据，或两个独立中信任证据一致；
5. 没有冲突 reference；
6. 商品类型不是已识别的配件/盒证等非腕表类别；
7. reference 格式通过品牌 pattern / 字典校验（如果可用）。

示例保守规则：

```python
trusted = (
    brand_id is not None
    and len(unique_refs) == 1
    and product_reference_votes >= 1
    and no_conflict
    and not accessory_like
    and role_confidence >= threshold
)
```

门槛宁可偏高。

---

## 10. 跨源匹配与聚类

当两个商品都拥有 `TRUSTED_SINGLE` reference 时：

```text
match_key = brand_id + ':' + canonical_reference
```

所有来源相同 key 的记录进入一个实体簇。

建议保存显式 edge：

```sql
CREATE TABLE match_edge (
    left_offer_id BIGINT NOT NULL,
    right_offer_id BIGINT NOT NULL,
    match_key TEXT NOT NULL,
    decision VARCHAR(16) NOT NULL,
    reason_code VARCHAR(64) NOT NULL,
    pipeline_version VARCHAR(64) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (left_offer_id, right_offer_id)
);
```

自动 edge 的 `reason_code` 只允许类似：

```text
EXACT_TRUSTED_REFERENCE
```

不建议存在：

```text
HIGH_SEMANTIC_SIMILARITY
IMAGE_SIMILARITY_HIGH
LLM_SAYS_SAME
```

这些只能作为人工复核辅助字段，不能作为自动 match reason。

---

## 11. 图片应该如何使用

Spec 明确有图片可用，但图片最适合做**证据增强和冲突否决**，不是身份主键。

推荐顺序：

1. OCR 保卡、吊牌、表背、发票/标签；
2. 从 OCR 抽 reference 候选；
3. 与标题/结构化字段交叉验证；
4. 视觉 embedding 只用于：
   - 人工复核排序；
   - 找“外观极像但 reference 不同”的 hard negatives；
   - 检测商品是否疑似表带/盒子/附件；
   - 不用于覆盖 reference 冲突。

例如：

```text
A.reference = 126610LN
B.reference = 126610LV
image_similarity = 0.98
```

最终仍必须判 **NOT MATCH**。

这是腕表数据中非常重要的安全规则。

---

## 12. 增量架构

100 万–1000 万规模不需要为了“精确 reference join”引入复杂向量基础设施。

推荐生产组件：

```text
Crawler / Importer
      │
      ▼
Object Storage（原图、原始 JSON）
      │
      ▼
Queue / Incremental Job
      │
      ├─ normalize worker
      ├─ OCR worker
      ├─ reference extractor
      └─ LLM validator（仅疑难）
      │
      ▼
PostgreSQL / OLTP metadata
      │
      ├─ reference_evidence
      ├─ reference_assignment
      ├─ match_edge
      └─ review_task
      │
      ▼
Exact Match Worker
      │
      ▼
Entity Cluster
```

10M 级别的 `(brand_id, canonical_reference)` B-Tree/Hash 索引完全可管理。若离线批量重算，可用 Spark/DuckDB/ClickHouse 做 join；但线上增量不需要 ANN。

增量商品到达时流程：

```python
def process_offer(offer):
    normalized = normalize_offer(offer)
    evidence = extract_reference_evidence(normalized)
    assignment = resolve_reference(evidence)

    save_assignment(assignment)

    if assignment.status != 'TRUSTED_SINGLE':
        enqueue_review_if_needed(offer, assignment)
        return

    peers = exact_lookup(
        brand_id=assignment.brand_id,
        canonical_reference=assignment.reference,
        exclude_source=offer.source,
    )

    for peer in peers:
        if peer.status == 'TRUSTED_SINGLE':
            create_exact_match_edge(offer, peer)
```

---

## 13. 人工标注几百对应该标什么

Spec 可以接受几百对黄金标签。不要随机抽几百对普通商品 pair；随机样本会被大量容易负例浪费。

建议把标注预算放到最危险的 hard cases：

1. 同品牌、同系列、相邻 reference；
2. reference 只在标题中；
3. reference 只在图片 OCR 中；
4. 一个标题多个 reference；
5. 表带/盒证/配件出现腕表 reference；
6. 平台 SKU 与品牌 reference 形状相似；
7. O/0、I/1、S/5 OCR 混淆；
8. reference 缺一位/多一位；
9. 同 reference 但品牌不同；
10. 同外观但 reference 不同。

并且标签对象最好不是单纯 pair match，而是拆成：

```text
offer_id
brand_label
reference_span
canonical_reference
reference_role
product_type
safe_to_auto_match
```

这样同一批人工数据可以同时训练/评估抽取器、角色分类器和最终 gate。

---

## 14. Precision-first 评估方案

不能只看 F1。

当前业务最重要的指标是：

```text
Auto-Match Precision
False Positive Count
Coverage / Auto-Match Rate
Abstain Rate
```

建议分层评估：

### Layer A：reference extraction precision

抽出的 reference 是否正确存在、是否属于当前商品本体。

### Layer B：trusted gate precision

被标成 `TRUSTED_SINGLE` 的记录，其 canonical reference 是否正确。

### Layer C：match precision

自动生成的 `EXACT_TRUSTED_REFERENCE` edge 是否真的满足业务定义。

### Layer D：coverage

有多少记录最终可以自动匹配。

业务目标应该类似：

```text
先把 Layer C precision 做到可接受的极高水平，
再逐步提升 Layer D coverage。
```

不能为了覆盖率放松自动 merge gate。

### 一个需要特别注意的统计问题

如果几百条黄金样本上“一个 false positive 都没有”，并不等于真实 precision 已经被证明达到 99.9% 甚至更高。

例如 300 个被接受样本里 0 个错误，按经典 `rule of three`，95% 置信下真实错误率的上界仍大约是 1%。

所以几百条人工标签适合：

- 设计规则；
- 找 hard negative；
- 训练/校准抽取器；
- 早期回归测试。

但若要对“极低误匹配率”建立统计信心，后续需要持续积累**被自动接受的样本**进行抽检，尤其要对高风险品牌、来源和新分布单独统计。

---

## 15. 可直接实现的 LLM 原子任务

### 15.1 任务 1：reference span extraction

输入：标题/描述/OCR 文本。

输出：原文 span，不允许知识补全。

```text
你只抽取文本中实际出现的、可能作为当前售卖商品品牌 reference/model number 的字母数字串。
不得根据品牌知识猜测、补齐或改写不存在的型号。
如果没有，返回空数组。
如果有多个，全部返回。
```

### 15.2 任务 2：reference role classification

```text
给定商品类型、标题上下文和候选编号，判断编号角色：
PRODUCT_REFERENCE / PLATFORM_SKU / SELLER_INTERNAL_ID /
COMPATIBILITY_REFERENCE / ACCESSORY_REFERENCE /
SERIAL_NUMBER / CALIBER_NUMBER / UNKNOWN。

只输出 JSON。
不确定必须输出 UNKNOWN。
```

### 15.3 任务 3：冲突解释

当结构化字段与 OCR/标题不一致时，LLM 可以生成“冲突摘要”给人工审核，但不能决定 override：

```text
structured_ref = 126610LN
ocr_ref = 126610LV
```

系统行为：

```text
status = CONFLICTED
no auto match
```

LLM 只帮助解释冲突来源。

---

## 16. 建议的 MVP

### Phase 1：纯规则 + exact key

先不接 LLM，完成：

- 品牌归一；
- 三个平台 reference 字段盘点；
- 标题 regex 候选；
- brand-specific canonicalizer；
- `reference_evidence` / `reference_assignment` 表；
- `TRUSTED_SINGLE` gate；
- `(brand_id, reference)` exact join；
- 人工审核页。

目标：快速得到一个 precision 很高但 coverage 不高的基线。

### Phase 2：OCR

增加：

- 保卡/吊牌/表背 OCR；
- OCR reference candidate；
- 文本/OCR 一致性验证。

目标：提升字段缺失记录的 coverage。

### Phase 3：7B 小模型 atomic validator

借鉴论文：

- 只处理规则无法确认的候选；
- 使用原子 prompt；
- 双顺序 few-shot；
- 取 intersection；
- 不允许自由生成 reference；
- 保留 abstain。

目标：增加 `TRUSTED_SINGLE` 数量，但不降低 precision。

### Phase 4：主动学习 / hard-negative 回流

人工审核结果自动沉淀：

```text
new regex
brand canonical rule
SKU pattern deny-list
accessory phrase pattern
few-shot hard negative
```

重点优化“最容易误匹配”的边界，而不是平均 F1。

---

## 17. 与论文方案的关键改造点

| 论文做法 | 当前需求的改造 |
|---|---|
| LLM 对 candidate pair 判 True/False | LLM 只做 reference 抽取和 role validation |
| Blocking 后做语义验证 | trusted reference 后直接 exact join |
| 以 F1 为主要指标 | auto-match precision / false positive 优先 |
| Atomic prompt 比 composite 更优 | 把 reference 提升为唯一正向 identity key |
| Few-shot 受 position bias | 两个顺序分别跑，取 intersection |
| 模型 recall 高、precision 低 | LLM 不进入最终自动 merge 信任边界 |
| 4-bit 7B 可在单 GPU 运行 | 仅疑难记录调用本地小模型，成本进一步降低 |

---

## 18. 最终建议

这篇论文给当前项目最重要的启示不是“7B LLM 足以做实体匹配”，而恰恰是：

> **即使在较干净的商品数据上，小 LLM 也明显倾向多报 match；当业务要求 precision 极致优先时，应把 LLM 从最终 match decision 中移出去，只保留它擅长的非结构化信息抽取能力。**

当前 Spec 最适合的生产方案是：

```text
结构化字段 / 标题 / OCR
      ↓
候选 reference
      ↓
reference 角色验证
      ↓
品牌级保守 canonicalization
      ↓
唯一且无冲突？ ── 否 → abstain / review
      │ 是
      ↓
TRUSTED_SINGLE
      ↓
(brand_id, canonical_reference) exact match
      ↓
跨源实体簇
```

在这个架构中：

- **reference 是唯一正向匹配证据；**
- 品牌、标题、图片、系列等只用于抽取、验证和否决；
- 任何冲突都让系统拒识；
- LLM 只在“reference 是否属于当前商品本体”这类原子问题上工作；
- few-shot 顺序偏差通过双判交集压制；
- 自动匹配结果拥有可追溯 evidence chain；
- 后续新增来源和品牌时，只需新增 source adapter、品牌规则和少量 hard-negative 标注，不需要重建一个黑盒大模型 matcher。

这是一个可以直接从 MVP 演进到千万级增量生产的 precision-first 路线。
