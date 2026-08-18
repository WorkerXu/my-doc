# Entity Matching with 7B LLMs: A Study on Prompting Strategies and Hardware Limitations

## 1. 本次选取与结论

- 原文：<https://ceur-ws.org/Vol-3931/paper4.pdf>
- 需求：Notion「调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）」
- 本次分析前已检查 `奢侈品调研/b/`，该论文此前未分析；已存在的 Ditto、DeepBlocker、GraLMatch、TransClean、pyJedAI、Confidence Classifiers 等结果不重复。

**核心结论：这篇论文最值得迁移的不是“用 7B LLM 直接判断两个商品是否相同”，而是两条 precision-first 原则：**

1. **缩小判定信息到最有辨识力的原子属性**。论文在商品场景里发现，以 `model number` 单属性构造 atomic domain-specific prompt，通常比把 product name、features、manufacturer、model number 全塞进 composite prompt 更稳。原因很直接：型号短、区分度高，而长标题和 feature 文本会引入噪声。
2. **对不稳定的 LLM 判定取交集而不是并集**。论文把 few-shot 示例顺序改成 TF/FT 两种提示，发现 LLM 存在明显 position bias；对 OpenHermes、Zephyr，只有两次提示都判 Match 的 intersection 策略，precision 的提升大于 recall 的损失。

但这篇论文也给出了对当前 Spec 非常重要的反证：**7B LLM 即使在最优 prompt 下也远达不到“绝对不能误匹配”的要求。**例如论文 Table 2 中：

- Orca2 + FT few-shot：Abt-Buy precision 0.768；Walmart-Amazon precision 0.420；
- Orca2 + atomic domain-specific：Abt-Buy precision 0.689；Walmart-Amazon precision 0.434；
- Zephyr + intersection few-shot：Abt-Buy precision 0.667；Walmart-Amazon precision 0.408。

因此，在本需求中 **LLM 不能拥有最终 Match 决策权**。最合适的定位是：

- 从标题/描述/OCR 中抽取 reference 候选；
- 判断字符串在上下文中扮演的“编号角色”（商品 reference、平台 SKU、店铺货号、兼容型号、序列号等）；
- 对规则已生成的候选做辅助冲突检测；
- 输出可拒识的结构化证据。

最终自动合并必须由 **canonical reference 的硬一致性 + 品牌约束 + 编号角色校验 + 冲突否决** 收口。

---

## 2. 论文技术实现拆解

### 2.1 问题框架：Filtering → Verification

论文沿用经典 ER/EM 的两阶段结构：

1. **Filtering / Blocking**：先从笛卡尔积中召回少量可能重复的候选对；
2. **Verification / Entity Matching**：对候选对做二分类，输出 True/False。

实验中 Blocking 使用 pyJedAI 的 kNN Join：

- Abt-Buy：字符 trigram，`k=4`；
- Walmart-Amazon：字符 4-gram，`k=2`；
- 都采用 stop-word removal、stemming、cosine similarity；
- 把 Blocking Recall 调到至少 90%，然后再让 LLM 验证候选。

这套结构适合“实体是否相同”的一般问题，但对当前 Spec 还可以进一步简化：**因为业务已经把“同一商品”严格定义成“同一 reference number”，reference 本身就是天然 blocking key / entity key。**只要 reference 被高精度抽出来并规范化，大多数记录根本不需要 pairwise LLM matcher。

### 2.2 7B 本地模型与量化

论文在 Ubuntu 22.04、i7-9700K、32GB RAM、GTX 1080 Ti 11GB 上，用 Ollama 运行 4-bit 量化的约 7B 模型。研究的关键不是某个具体模型，而是说明：

- 4-bit 量化可以让小模型在普通单卡上执行 EM；
- 运行成本可控，适合把 LLM 放到一个“疑难记录处理器”中，而不是对 1000 万级全量 pair 执行；
- 模型服从性差异很大：论文甚至发现 Llama 2 会对所有候选都回答 True，Mistral 会偏离 True/False 输出协议。

对本项目的启发是：**LLM 输出必须经过 schema 校验和规则校验，不能因为模型返回了一个合法 JSON 就相信语义正确。**模型升级也应被视作“抽取器版本变更”，需要离线回放黄金集后再发布。

### 2.3 Prompt 架构：generic / few-shot / domain-specific

论文比较了三类提示：

#### A. Generic zero-shot

只描述“给定两个记录，判断是否同一实体”，无示例。

优点是便宜、简单，但模型非常容易把“相似”误判成“相同”。论文观察到各模型普遍是 recall 高于 precision，false positive 明显。

#### B. Few-shot + TF/FT 双顺序

few-shot 放一个正例和一个负例，但作者特别测试：

- TF：正例在前，负例在后；
- FT：负例在前，正例在后。

同一候选对分别跑两次，然后：

- Union：任一次 True 就算 Match；
- Intersection：两次都是 True 才算 Match。

Intersection 的价值不是神奇提升模型，而是把 **“不稳定”转化为拒识信号**。这和当前 Spec 的目标一致：宁可不匹配，也不要误匹配。

#### C. Domain-specific：atomic vs composite

论文把商品匹配条件拆成：

- product name；
- features；
- manufacturer；
- model number。

Composite 同时考虑四项；Atomic 只使用 model number。实验中 Atomic 通常更好，作者解释为 model number 是短、干净、区分度强的值，而长文本属性会带入噪声。

对腕表二奢场景，这个结论应进一步强化：**reference 不是普通“特征之一”，而是业务定义本身。**因此模型不应学习“品牌、系列、尺寸、材质、图片很像，所以可能同款”的软逻辑去覆盖 reference 冲突。

---

## 3. 与当前 Spec 的关键差异

当前需求的目标不是最大化 F1，而是：

- 数据 100 万–1000 万级；
- 三个来源持续增量；
- 字段稀疏；
- reference 有时在独立字段，有时埋标题，有时只能从图片/OCR补足；
- **precision 优先到极致，允许大量 abstain；**
- “同一个商品” = **同一 reference number**。

因此，我建议把论文的通用 `candidate pair -> LLM True/False` 改造成：

> `record -> reference evidence extraction -> role validation -> canonicalization -> exact-key grouping -> conflict veto -> auto-link / abstain`

也就是说：**先解决“这条记录的 reference 到底是什么”，再解决跨源合并；不要直接让 LLM 判断两条商品是不是同款。**

这样有三个直接收益：

1. **复杂度从 pairwise 降到 key-based。**千万记录不需要两两比较；有 canonical reference 的记录可按 `(brand_id, canonical_reference)` 直接分桶。
2. **可解释性更强。**每次自动匹配都能回答“哪一个字段/标题 span/OCR 图证明 reference 是什么”。
3. **误匹配更容易被硬规则阻断。**只要两边 canonical reference 不一致，直接 Non-match；LLM 和视觉相似度都无权推翻。

---

## 4. 推荐直接落地的系统架构

```text
                     ┌─────────────────────────────┐
雷小安 ──────────────>│                             │
腕表之家 ────────────>│  1. Source Adapter / Ingest │──> Raw/Bronze
奢当家 ──────────────>│                             │
                     └──────────────┬──────────────┘
                                    │
                                    v
                     ┌─────────────────────────────┐
                     │ 2. Field Normalization      │
                     │ brand/title/ref/images/...  │
                     └──────────────┬──────────────┘
                                    │
                                    v
               ┌──────────────────────────────────────┐
               │ 3. Reference Evidence Extractor     │
               │ structured field / regex / LLM / OCR│
               └──────────────────┬───────────────────┘
                                  │ candidates + provenance
                                  v
               ┌──────────────────────────────────────┐
               │ 4. Number Role Classifier           │
               │ product_ref / SKU / serial / compat │
               └──────────────────┬───────────────────┘
                                  │
                                  v
               ┌──────────────────────────────────────┐
               │ 5. Brand-aware Canonicalizer        │
               │ lossless normalize + dictionary     │
               └──────────────────┬───────────────────┘
                                  │
                                  v
               ┌──────────────────────────────────────┐
               │ 6. Precision Gate                   │
               │ exact ref + brand + no conflict     │
               │ evidence intersection + abstention  │
               └───────────┬───────────────┬──────────┘
                           │               │
                     AUTO_MATCH         ABSTAIN
                           │               │
                           v               v
               ┌────────────────┐   ┌────────────────┐
               │ Entity Index   │   │ Review Queue   │
               │ (brand, ref)   │   │ hard cases     │
               └───────┬────────┘   └───────┬────────┘
                       │                    │ feedback
                       └────────────┬───────┘
                                    v
                         Gold set / rule versions
```

### 4.1 Source Adapter / Bronze 层

每条源记录先保留原始数据，不在入口阶段覆盖字段：

```json
{
  "source": "leixiaoan",
  "source_product_id": "...",
  "title_raw": "...",
  "reference_raw": "...",
  "brand_raw": "...",
  "attributes_raw": {},
  "image_urls": [],
  "ingested_at": "...",
  "source_updated_at": "..."
}
```

建议使用 Parquet + Iceberg/Delta/Hudi 之一保存批量历史版本；若目前只是小时/天级增量，**不必一开始引入复杂流系统**，定时 micro-batch 足够。后续若需要分钟级再加 Kafka/CDC。

### 4.2 Reference Evidence Extractor：一条记录产生多个“证据”，不是一个答案

不要只存 `reference = xxx`，而要存候选及来源：

```text
reference_evidence
- product_id
- extractor_type      # structured_field / regex / llm / image_ocr
- evidence_location   # field name / title span / image id
- raw_value
- canonical_candidate
- role                # product_reference / seller_sku / platform_id / serial / compatibility_ref / unknown
- brand_id
- confidence
- extractor_version
- created_at
```

推荐证据优先级：

1. 来源独立 reference 字段（但仍需做 role / 格式校验）；
2. 品牌/系列规则 + 标题 exact span；
3. LLM 从标题/描述抽取 exact span；
4. 图片 OCR（表背、保卡、吊牌等）；
5. 纯视觉相似度只能用于人工复核排序，不作为“同 reference”正证据。

### 4.3 LLM 不做 Pair Match，只做 constrained extraction / role classification

论文的“atomic model number”思路在这里可直接改造成一个强约束抽取器：

```text
SYSTEM
你是商品编号抽取器，不判断商品是否相似。
只能从输入原文中复制编号，不得补全、改写或猜测。

任务：抽取“当前被售商品本身”的 manufacturer reference/model number。
排除：
- 平台商品ID、店铺SKU、库存号
- 序列号/机芯序列
- 文中“适配/兼容/for”后面的其他商品型号
- 配件所适配主商品的 reference

如果证据不足或存在两个冲突候选，必须 abstain。

只返回 JSON：
{
  "reference_raw": string|null,
  "role": "product_reference"|"seller_sku"|"platform_id"|"serial"|"compatibility_reference"|"unknown",
  "evidence_span": string|null,
  "abstain": boolean
}
```

**强制要求 `reference_raw` 必须是输入文本的连续子串**。服务端收到结果后再做 substring check，模型若生成了原文不存在的编号，直接丢弃。这一步可以消灭最危险的一类“LLM 猜型号”错误。

### 4.4 Number Role Classifier：解决最容易产生灾难性 false positive 的问题

二奢和腕表里，长得像 reference 的字符串很多：

- 真正腕表 reference；
- 平台货号 / 店铺 SKU；
- 库存号；
- 序列号；
- 机芯号；
- 表带/配件写的“适配 126610LN”；
- 同标题列出的多个兼容型号。

如果只做正则抽取，很容易把“兼容型号”误当当前商品 reference。这比“没抽出来”危险得多。

因此我建议把角色分类独立成一个模型/规则模块，且它的输出只能影响“是否允许进入自动匹配”，不能直接创造匹配。

典型硬规则：

```text
if context contains [兼容, 适配, suitable for, for, replacement for]
   and candidate ref occurs in compatibility phrase:
       role = compatibility_reference
       forbid_auto_match = true
```

### 4.5 Brand-aware Canonicalizer：宁可少归一化，不做破坏性归一化

Reference 规范化的目标是消除格式差异，而不是“让相似字符串变一样”。建议分两层：

#### 通用 lossless normalize

- Unicode NFKC；
- trim；
- uppercase；
- 统一全角/半角；
- 统一常见 dash 字符；
- 连续空格压缩；
- 保留原始值与变换轨迹。

#### 品牌级 canonical rule

只对已有人工验证的品牌规则做安全映射，例如某品牌确定允许去掉某位置的空格或固定分隔符。**不要全局删除 `-`, `/`, `.`**，因为这些字符可能参与区分 reference。

建议返回：

```json
{
  "raw": " 126610 LN ",
  "canonical": "126610LN",
  "normalizer": "rolex/v3",
  "transformations": ["trim", "upper", "remove_verified_space"]
}
```

如果无法确定某个字符能否安全删除，保持原样并 abstain，不要“相似归一化”。

### 4.6 Precision Gate：最终自动合并规则

建议把“自动匹配”设计成显式规则引擎，而不是一个分数阈值。

#### Gate A：品牌必须一致

品牌先规范到 canonical brand id。品牌冲突直接 Non-match。

#### Gate B：reference canonical 必须完全一致

```text
left.canonical_reference == right.canonical_reference
```

只允许 exact equality；编辑距离、embedding 相似度不能在这个 Gate 中转成 Match。

#### Gate C：reference 必须被判定为“当前商品本身的 reference”

`role == product_reference` 才进入自动匹配。

#### Gate D：独立证据取交集（迁移论文 intersection 思路）

定义独立证据源：

- structured field；
- title/text extractor；
- OCR；
- brand reference dictionary。

可配置高精度规则，例如：

```text
AUTO_MATCH when
  brand_exact
  AND ref_exact
  AND role_is_product_reference
  AND no_conflict
  AND (
       trusted_structured_ref
       OR evidence_count_independent >= 2
  )
```

这里的 intersection 与论文 TF/FT 交集思想一致，但更适合生产：**不是让同一 LLM 重复投票，而是要求不同证据通道互相确认。**

#### Gate E：冲突具有否决权

以下任意一个条件触发，直接 abstain：

- structured reference 与标题 reference 不同；
- OCR 读出另一个高可信 reference；
- 同一条商品记录出现多个无法解释的 reference；
- 品牌与 reference 字典不兼容；
- 标题表明是配件/表带/盒证而不是腕表主体；
- reference 只存在于 compatibility 语境；
- normalizer 需要使用尚未验证的激进变换。

**冲突不能靠“多数票”覆盖。**precision-first 系统里，一个强冲突就应该阻断自动合并。

---

## 5. 为什么不建议照论文直接用 LLM 二分类

### 5.1 实验 precision 与业务要求不在一个量级

论文最佳配置在困难数据集上的 precision 只有约 0.3–0.43。即使在较干净的 Abt-Buy，最好也只有 0.768 左右。

如果 1000 万商品里自动产生 100 万条匹配边，哪怕 precision 99.9%，仍可能有约 1000 条误匹配；而论文结果离 99.9% 还很远。

所以 LLM classifier 更适合：

- `Reject / Review` 分流；
- hard-negative 检测；
- 证据抽取；
- role classification；

不适合成为最终 merge authority。

### 5.2 F1 不是这里的主指标

论文按 Precision / Recall / F1 比较模型，而当前需求应该把指标改成：

1. `AutoMatchPrecision`：自动合并样本的精确率；
2. `FalsePositiveCount`：黄金集和线上审计中出现的误合并数；
3. `AutoMatchCoverage`：多少记录能自动进入实体组；
4. `AbstainRate`；
5. `ReferenceExtractionPrecision`；
6. `ConflictRate`，按来源/品牌/规则版本拆分；
7. `HardNegativePrecision`：同系列相邻 reference、配件兼容型号等高风险样本上的 precision。

**第一阶段可以接受 coverage 很低，但 FP 必须为 0。**

### 5.3 “几百对黄金标签”无法统计意义上证明极端 precision

如果只标几百对，即使一个 false positive 都没观察到，也不能严格证明线上误匹配率接近 0。按常见的 “rule of three” 粗略理解：0 次错误、样本量 `n` 时，95% 置信层面的错误率上界大约还是 `3/n`。

因此几百对标签最适合用来：

- 找规则漏洞；
- 构造 hard negatives；
- 比较 normalizer / extractor 版本；
- 建立 regression suite；

而不是宣称“已经统计证明 99.99% precision”。上线应通过 shadow mode + 持续抽检逐步累积证据。

---

## 6. 训练与标注：把“标 pair”改成“标 record + hard pair”

Spec 允许人工标几百对。我建议拆成两类标签，信息密度更高。

### 6.1 Record-level reference labels（优先）

每条记录标：

```text
- brand_id
- reference_raw_span
- canonical_reference
- role
- is_watch_main_product
- conflict_reason(optional)
```

原因：一条 record label 能服务于任何跨源 pair；如果只标 pair，很多标签信息不能复用。

### 6.2 Hard-negative pairs

专门挑最危险的 pair：

- 同品牌、只差 1 个字符的 reference；
- 同系列不同尺寸；
- 同壳型但不同材质/盘面导致不同 reference；
- 表带/配件标题包含主表 reference；
- 平台 SKU 恰好长得像品牌 reference；
- 一边 reference 缺失、图片极像；
- OCR 容易混淆 `0/O`, `1/I`, `5/S`, `8/B`；
- 一条记录里同时出现两个 reference。

这些样本比随机负例重要得多，因为真实误合并往往发生在近邻 reference 上。

---

## 7. 增量处理与千万级规模

### 7.1 正常路径不做全量 pairwise

有可信 reference 的记录直接计算：

```text
entity_key = hash(brand_id + "|" + canonical_reference)
```

再通过 KV / SQL unique index 找已有 entity：

```sql
SELECT entity_id
FROM reference_entity_index
WHERE brand_id = ?
  AND canonical_reference = ?;
```

因此主路径复杂度接近 O(N) ingest + 索引查询，而不是 O(N²)。

### 7.2 只有 abstain 记录才进入候选检索

对没有足够 reference 证据的记录，可用：

- brand；
- series；
- title n-gram；
- 受限 reference candidate dictionary；
- image embedding；

召回少量候选，**但召回结果默认是 REVIEW CANDIDATE，而不是 Match**。

这时可以借鉴论文 Filtering → Verification 架构：先 kNN/blocking，再让 LLM/规则做“是否存在冲突”的二阶段验证。

### 7.3 推荐数据表

```text
product_record
- product_id
- source
- source_product_id
- source_updated_at
- brand_id
- title_raw
- raw_payload_uri
- pipeline_version

reference_evidence
- evidence_id
- product_id
- extractor_type
- raw_value
- canonical_candidate
- role
- evidence_location
- confidence
- extractor_version

product_resolution
- product_id
- resolved_brand_id
- resolved_reference
- resolution_status     # auto_resolved / review / unresolved
- resolution_reason
- rule_version

reference_entity
- entity_id
- brand_id
- canonical_reference
- created_at

entity_membership
- entity_id
- product_id
- source
- membership_status
- decision_reason
- decision_version

review_case
- case_id
- product_id(s)
- reason_code
- evidence_snapshot
- reviewer_result
- created_at
```

其中 `reference_entity(brand_id, canonical_reference)` 建唯一约束，避免同一 reference 被拆成多个实体。

---

## 8. 关键伪代码

```python
def resolve_product(record):
    brand = normalize_brand(record)
    evidences = []

    # 1) 独立证据通道
    evidences += extract_structured_reference(record)
    evidences += extract_reference_by_brand_rules(record)
    evidences += extract_reference_by_llm(record)      # exact-span only

    if needs_ocr(record, evidences):
        evidences += extract_reference_from_images(record)

    # 2) 编号角色
    evidences = classify_reference_roles(record, evidences)

    # 3) 品牌级安全规范化
    for e in evidences:
        e.canonical = canonicalize_reference(brand, e.raw_value)

    # 4) 冲突检查
    conflicts = find_strong_conflicts(evidences)
    if conflicts:
        return Abstain(reason="reference_conflict", evidence=evidences)

    valid = [
        e for e in evidences
        if e.role == "product_reference"
        and e.canonical is not None
        and brand_reference_is_valid(brand, e.canonical)
    ]

    if not valid:
        return Abstain(reason="no_valid_reference", evidence=evidences)

    # 5) 所有强证据必须指向同一个 canonical reference
    refs = {e.canonical for e in valid if is_strong(e)}
    if len(refs) != 1:
        return Abstain(reason="ambiguous_reference", evidence=evidences)

    ref = next(iter(refs))

    # 6) intersection-style precision gate
    if not (
        has_trusted_structured_ref(valid, ref)
        or independent_support_count(valid, ref) >= 2
    ):
        return Abstain(reason="insufficient_independent_evidence", evidence=evidences)

    return AutoResolved(brand_id=brand.id, canonical_reference=ref,
                        evidence=evidences)


def match_entities(left, right):
    # LLM / image similarity NEVER overrides these rules
    if left.status != "auto_resolved" or right.status != "auto_resolved":
        return "ABSTAIN"
    if left.brand_id != right.brand_id:
        return "NON_MATCH"
    if left.canonical_reference != right.canonical_reference:
        return "NON_MATCH"
    return "MATCH"
```

这段逻辑最大的特点是：**`MATCH` 分支里没有 LLM 分数。**LLM 只参与把原始数据变成可审计的 reference evidence。

---

## 9. 图片的正确角色

Spec 明确有图片。图片很有价值，但不建议把“视觉相似”作为正向自动合并条件，因为同系列不同 reference 的腕表可能几乎一模一样。

建议图片只做三件事：

1. **OCR reference**：表背、保卡、吊牌上的字母数字；
2. **冲突否决**：文本说 A reference，但图片高置信 OCR 为 B；
3. **人工复核排序**：在 abstain 候选中提高相似商品的优先级。

尤其要避免：

```text
text reference mismatch + image similarity high => MATCH
```

正确逻辑应是：

```text
text reference mismatch => NON_MATCH / ABSTAIN
image similarity cannot override
```

---

## 10. 上线阶段建议

### Phase 0：离线画像

先统计三源：

- reference 独立字段覆盖率；
- 标题中可 regex 直接提取比例；
- 每品牌 reference 格式；
- 一条记录多 reference 的比例；
- 配件/表带/盒证比例；
- OCR 可用图片比例。

输出品牌级规则优先级，不要一套正则打所有品牌。

### Phase 1：只上 deterministic high-precision baseline

只允许：

```text
brand exact
+ trusted reference field
+ safe canonicalization
+ exact reference equality
+ no conflict
```

先测“最保守基线”能覆盖多少数据。

### Phase 2：加入标题 reference extractor

- 规则 first；
- LLM 只对规则无法确定的记录执行；
- LLM 必须 exact-span extraction + abstain；
- 引入 number role classifier；
- 把标题证据和 structured field 做 intersection。

### Phase 3：加入 OCR

只在：

- 无 reference；
- 文本冲突；
- 高价值人工复核；

这几类记录上跑，避免对千万级全量图片做高成本推理。

### Phase 4：Shadow Mode

所有“自动 Match”先只产出影子结果，不真正合并；按品牌/来源/规则版本抽检。发现一次 FP，必须追溯：

- extractor；
- role classifier；
- canonicalizer；
- evidence gate；

把根因固化成 regression case 后再恢复放量。

### Phase 5：逐品牌放量

优先上线 reference 结构最稳定的品牌，再扩展到复杂品牌。precision gate 可以按品牌独立配置，不要求全局同一阈值。

---

## 11. 可观测性与审计

每个自动匹配边都必须保存 `reason_code`，例如：

```text
MATCH_STRUCTURED_EXACT
MATCH_STRUCTURED_TITLE_INTERSECTION
MATCH_TITLE_OCR_INTERSECTION
ABSTAIN_REFERENCE_CONFLICT
ABSTAIN_COMPATIBILITY_REFERENCE
ABSTAIN_MULTIPLE_REFERENCES
ABSTAIN_UNKNOWN_NUMBER_ROLE
ABSTAIN_UNSAFE_NORMALIZATION
NON_MATCH_BRAND_CONFLICT
NON_MATCH_REFERENCE_CONFLICT
```

Dashboard 至少按以下维度切：

- source pair；
- brand；
- extractor_version；
- normalizer_version；
- reason_code；
- 自动覆盖率；
- 人审 overturn rate；
- hard-negative FP；
- OCR 冲突率。

这会比只看一个全局 F1 更容易保障线上 precision。

---

## 12. 对论文方案的“保留 / 改造 / 放弃”

| 论文设计 | 当前项目处理 | 原因 |
|---|---|---|
| Filtering → Verification | **保留，但仅用于无可信 reference 的疑难记录** | 大部分有 reference 的数据可直接按 key 解析 |
| 7B 4-bit 本地 LLM | **保留为低成本抽取器/角色分类器** | 可处理长尾脏标题，但不承担最终 Match |
| Generic pairwise True/False | **放弃** | precision 不够，且“相似”容易被模型误当“相同” |
| Atomic model number prompt | **强烈保留并强化** | 与 Spec 的“同一 reference”业务定义完全一致 |
| Composite 多属性 prompt | **仅用于人工复核解释/冲突检测** | 其他属性会把相似变体推向 false positive |
| TF/FT two prompts | **可保留用于 LLM 稳定性检测** | 两次结果不一致直接 abstain |
| Union | **放弃** | 会增加 false positive |
| Intersection | **保留思想，升级为多证据交集** | structured/title/OCR 独立证据交集比同模型重复投票更可靠 |
| 以 F1 选最佳模型 | **放弃** | 当前业务应按 precision-first + coverage 评估 |

---

## 13. 最终可直接落地的 MVP

如果只做一个最小可用版本，我建议 2 周左右按以下顺序实现：

1. `brand_normalizer`：三源品牌统一；
2. `reference_evidence` 数据模型；
3. 结构化 reference 字段抽取；
4. 20–30 个主力品牌的 title regex / canonical rules；
5. `number_role` 规则：SKU、库存号、兼容型号、序列号；
6. `precision_gate`：品牌一致 + canonical reference exact + 无冲突；
7. `(brand_id, canonical_reference)` entity index；
8. review queue + reason_code；
9. 几百条 record-level gold set + hard-negative regression；
10. 最后再把 LLM 接到“规则无法确定”的小流量路径，做 exact-span reference 抽取，不改变最终 hard gate。

**第一版甚至可以完全不需要 pairwise matcher。**这反而更符合当前需求，因为业务定义已经给了一个非常强的实体键：reference number。

等 MVP 验证后，再加：

- OCR；
- LLM role classifier；
- 无 reference 记录的 blocking / retrieval；
- 人审反馈闭环；
- selective risk calibration。

---

## 14. 最终建议

这篇论文对当前系统最有价值的启示，可以浓缩成一句话：

> **越是 precision-first，越不应该让模型综合更多“相似性特征”自由判断；应该把问题压缩到 reference 这个原子属性，并对任何不稳定、冲突或证据不足的情况主动拒识。**

论文证明小 LLM 可以低成本参与商品 EM，也同时证明了“直接让 LLM 做 Match”会产生大量 false positive。因此建议最终架构采用：

**Reference-first + Evidence Intersection + Hard Exact Gate + Abstention**。

其中 LLM 是证据生产者，规则引擎才是自动合并裁决者。这样既能利用 LLM 处理字段稀疏、标题埋型号、长尾格式，又不会让模型语义相似度破坏“同一 reference”这一业务硬定义。