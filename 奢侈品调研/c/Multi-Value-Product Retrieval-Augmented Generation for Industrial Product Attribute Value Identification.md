# Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification：改造成 Reference-First 高精度腕表实体链接系统

> 分析人：c  
> 日期：2026-08-18  
> 目标需求：跨源二奢/腕表商品实体匹配（雷小安 × 腕表之家 × 奢当家）  
> 核心约束：100 万–1000 万级持续增量；字段高度稀疏；reference 可能位于结构化字段、标题或图片；“同一个商品”定义为**同一 reference number / 型号**；**precision 极端优先，绝不能误匹配，允许漏匹配**。

## 0. 选题、去重与结论先行

本次从 `奢侈品文章调研.md` 选取：

- 论文：**Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification**
- 简称：**MVP-RAG**
- EMNLP 2025 Industry Track：https://aclanthology.org/2025.emnlp-industry.147/
- arXiv：https://arxiv.org/abs/2509.23874
- 其属性值检索器依赖的 TACLR：https://arxiv.org/abs/2501.03835

`奢侈品文章调研.md` 对该论文的推荐理由是：把商品属性识别改成“检索增强生成”，重点处理未见属性值与级联错误；可先按品牌、系列和候选型号库检索，再只在受限候选中识别 reference，减少自由生成造成的误型号，适合腕表长尾型号持续增量。

执行前检查 `奢侈品调研/c/`，c 已经分析过：

1. `Ameli：Enhancing Multimodal Entity Linking with Fine-Grained Attributes.md`
2. `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
3. `pyJedAI.md`

因此 MVP-RAG 未被 c 分析过，满足去重要求。群内其他成员已经多次覆盖 DeepBlocker 等 Blocking 方向，所以本次选择 MVP-RAG，补足当前方案最关键但仍容易被低估的一层：**如何从脏、稀疏、不断变化的商品文本中，把 reference 解析成可用于严格实体链接的 canonical value。**

### 结论先行

MVP-RAG **不能原样作为最终匹配器上线**。论文在工业 PAVI 上表现很强，但其目标仍是整体属性识别 F1，而本需求要求“绝不能误匹配”；即便论文报告的高 precision，对自动合并而言也远远不够。

真正值得复用的是其结构：

```text
商品文本
  -> 候选标准值检索
  -> 同类可信商品示例检索
  -> 受上下文约束的值生成/判定
```

针对本需求，应改造成：

```text
listing
  -> 品牌/类别归一化
  -> reference 字面候选抽取
  -> Canonical Reference 候选检索（Top-K，小 K）
  -> 可信 reference exemplar 检索
  -> 受约束 Reference Resolver
  -> Strict Verification Gate
       ├─ AUTO_LINK
       ├─ REVIEW
       ├─ NEW_REFERENCE_PENDING
       └─ ABSTAIN
```

最重要的架构原则是：

> **Embedding、RAG、LLM 都只能帮助“找到/解释 reference 候选”，不能越权决定两个 listing 是同一商品。最终自动链接必须收口到可审计的 canonical reference 硬证据。**

并且不要再把问题建模为三源 listing 两两 pairwise matching，而应建模为：

```text
listing -> Canonical Reference Entity
```

例如：

```text
雷小安 listing A -> Rolex / 126610LN
腕表之家 listing B -> Rolex / 126610LN
奢当家 listing C   -> Rolex / 126610LN

=> 三条 listing 自动属于同一个 reference entity
```

这样来源数量增加时，不需要重新做所有 source pair 的笛卡尔组合。

---

## 1. MVP-RAG 到底解决了什么问题

MVP-RAG 面向 Product Attribute Value Identification（PAVI）。它不是简单从标题中抽一个 span，而是要输出**标准化后的属性值**。

这和 reference number 非常相似：

- 原始标题可能写 `126610 ln`、`126610LN`、`Ref. 126610-LN`；
- 平台可能把品牌货号、店铺 SKU、reference 混在一起；
- 标题可能不直接出现标准值，而是出现别名、缩写；
- 新 reference 会持续出现；
- 最终系统需要统一输出 canonical reference，例如 `126610LN`。

论文将 PAVI 建模成 retrieval-generation：

1. 商品标题 + 描述作为 query；
2. 标准属性值作为一个 corpus；
3. 历史同类商品作为另一个 corpus；
4. 先同时检索“候选值”和“相似商品”；
5. 再把两路检索结果放进 LLM prompt，生成标准属性值。

这比纯 NER 更适合 reference normalization，因为 NER 只擅长“文本里原样出现了什么”，而我们真正需要的是：

```text
raw mention -> canonical reference
```

---

## 2. MVP-RAG 技术架构拆解

## 2.1 第一层：Attribute Value Retrieval

论文的候选值检索沿用 TACLR 思路：

- query：`title + description`
- candidate value：标准 taxonomy 中的值
- 候选 value 不是裸字符串，而是带上下文编码，例如：

```text
A {category} with {attribute} being {value}
```

- item 与 value 进入共享 encoder；
- 用 cosine similarity 召回 Top-K；
- 只在对应 category/attribute 的候选值子空间中搜索，而不是全局乱搜。

TACLR 的两个设计尤其适合腕表 reference：

### 2.1.1 Taxonomy-aware hard negative sampling

TACLR 不用大量随机负例，而优先从**同 category + 同 attribute** 中抽 hard negatives。

对腕表最合理的迁移是：

```text
正例：Rolex 126610LN
难负例：Rolex 126610LV
难负例：Rolex 116610LN
难负例：Rolex 126613LN

而不是：
负例：Hermès 手袋 / 某个随机商品
```

因为真正会造成灾难性 false positive 的，恰恰是“同品牌、同系列、外观相似、型号只差 1–2 个字符”的邻近 reference。

### 2.1.2 Null value / 动态阈值

TACLR 为每个 category-attribute 引入显式 null value，并用 item 与 null embedding 的相似度作为动态阈值。

这比“全站统一 cosine > 0.8 就算匹配”合理得多。

迁移到本需求，可以把 `NULL_REFERENCE` / `UNKNOWN_REFERENCE` 作为显式候选：

```text
Rolex / Submariner 候选：
- 126610LN
- 126610LV
- 124060
- 116610LN
- UNKNOWN_REFERENCE
```

如果真实 reference 证据不足，应让 UNKNOWN 赢，而不是被迫从已有 reference 中猜一个。

**对 precision-first 系统来说，学会输出 UNKNOWN 比提高一点 Recall 更重要。**

---

## 2.2 第二层：Product Retrieval

MVP-RAG 同时检索同 category 的相似商品，论文使用通用文本表示模型计算向量相似度，然后把相似商品作为 few-shot 上下文。

迁移到腕表场景，不能直接把“历史上最相似的 listing”塞给 LLM，因为历史 listing 自己可能标错型号，会形成 RAG poisoning。

应改成只检索**可信 exemplar**：

```text
reference_exemplar
- canonical_ref_id
- source_listing_id
- normalized_brand
- title
- description
- verified_reference
- verification_source
- quality_level
- embedding
```

只有满足以下之一的记录才进入 exemplar index：

1. 官方/权威 catalog 明确 reference；
2. 人工双审确认；
3. 结构化 reference 字段与标题/图片 OCR 双证据一致；
4. 已通过长期审计的高可信来源。

普通历史自动预测结果不要反哺 exemplar index，否则错误会自我强化。

---

## 2.3 第三层：Attribute Value Generation

论文将以下信息拼入 prompt：

1. 任务定义；
2. 规则说明；
3. 相似商品；
4. 当前目标商品；
5. candidate attribute values。

再由微调后的 Qwen2.5-7B-Instruct 生成标准值。

原论文允许模型在某些场景下生成候选列表之外的新值，以处理 OOD。这对于一般属性补全很有价值，但对本需求是危险的：

> **一个 LLM 自由生成出来的 reference 绝不能直接触发 AUTO_LINK。**

因此需要将生成器改成“受约束解析器”：

```json
{
  "decision": "CANDIDATE | NEW_REFERENCE_PENDING | ABSTAIN",
  "canonical_reference": "126610LN | null",
  "raw_reference_mentions": ["126610 LN"],
  "evidence_spans": ["劳力士潜航者 126610 LN 黑盘"],
  "candidate_rank": 1,
  "conflicts": [],
  "reason_codes": ["TITLE_EXACT_NORMALIZED", "BRAND_MATCH"]
}
```

规则：

- `CANDIDATE` 时 `canonical_reference` **必须精确等于候选库中的一个值**；
- 文本里看到了 reference-like token，但 canonical catalog 没有对应值：`NEW_REFERENCE_PENDING`；
- 多个 reference 冲突、疑似配件适配型号、证据不足：`ABSTAIN`；
- 禁止模型把一个不存在于候选库的自由生成 reference 直接解释成已有 reference。

---

## 3. 论文实验对当前项目最有价值的信号

MVP-RAG 在 Xianyu-PAVI 上报告了明显优于基础分类/生成方案的结果，论文实验还给出了几个比总 F1 更值得迁移的观察。

### 3.1 Candidate K 不是越大越好

论文分析显示，候选值数量增加时，真实值 coverage 会继续上升，但总体 precision 会下降，F1 在较小候选数附近达到峰值。

这非常符合腕表场景：

> 对 precision-first，候选集应追求“小而覆盖”，不要为了 recall 把几十个相似型号一股脑交给 LLM。

首版建议：

```text
K_reference = 5 或 6
```

并按品牌/品类/系列先分区，再检索。

### 3.2 候选召回正确，生成质量会显著提升

论文中，当正确 attribute value 已经包含在候选列表中时，最终生成效果明显更好。

对应当前系统，工程投入优先级应该是：

```text
reference catalog 质量
> normalization/alias 质量
> candidate recall@K
> LLM 大小
```

不是先换一个更大的模型。

### 3.3 相似商品示例会帮助，也会污染

论文对错误 retrieved examples 的实验说明：少量错误示例存在时模型仍可能纠正，但当错误示例占据明显多数后会被带偏。

所以生产系统不能把 RAG 看成“总是加分”。必须加 exemplar quality gate、去重复、冲突检测和来源隔离。

---

## 4. MVP-RAG 哪些地方不能直接照搬

## 4.1 不能用 PAVI 的总体 precision 当自动合并门槛

论文在工业 PAVI 上达到很强的总体表现，但“90%+ precision”在属性补全里可能是优秀结果，在实体自动合并里却意味着灾难。

假设 1000 万条 listing 中只有 10% 进入自动链接：

```text
100 万次自动链接 × 1% FP = 1 万次错误实体合并
```

本需求显然不可接受。

因此模型分数只用于排序/拒识，最终 AUTO_LINK 需要独立的 strict gate，并用黄金集做 selective calibration。

## 4.2 不能允许自由生成 OOD reference 后自动落库

MVP-RAG 的生成能力正是为了缓解 OOD，但 reference 是身份键，不是普通描述属性。

正确做法：

```text
看到未知 reference-like token
    -> NEW_REFERENCE_PENDING
    -> 校验品牌规则 / 官方 catalog / 人工复核
    -> 写入 canonical_reference
    -> 重建/增量更新 candidate index
    -> 后续 listing 才可以链接
```

而不是：

```text
LLM 猜到一个新 reference
    -> 立刻把多个 listing 合并
```

## 4.3 不能只依赖文本

MVP-RAG 论文自身主要处理文本。当前需求明确有图片，因此我们可以增加：

- 表背刻字 OCR；
- 保卡/吊牌 OCR；
- 表盒标签 OCR；
- 图片中的 reference-like token；
- 视觉类别/系列作为冲突证据。

但图片不应该成为“看起来很像，所以同 reference”的充分条件。

**图像最安全的作用是：补充 reference 文字证据和做 contradiction veto。**

---

## 5. 推荐的目标架构：Reference-First MVP-RAG

```text
                        ┌──────────────────────┐
雷小安 ─┐               │ Canonical Reference  │
腕表之家 ├─ Ingestion ─▶ │ Catalog + Alias      │
奢当家 ─┘               └──────────┬───────────┘
                                   │
                                   ▼
                         ┌────────────────────┐
                         │ Normalization       │
                         │ brand/category/text │
                         └─────────┬──────────┘
                                   │
                ┌──────────────────┼─────────────────┐
                ▼                  ▼                 ▼
       structured ref       title/desc parser      OCR
                └──────────────────┼─────────────────┘
                                   ▼
                      ┌─────────────────────────┐
                      │ Literal Candidate Layer │
                      │ exact/alias/regex       │
                      └────────────┬────────────┘
                                   ▼
                      ┌─────────────────────────┐
                      │ Reference Retriever     │
                      │ TACLR-style dual encoder│
                      │ + ANN, Top-K 5~6        │
                      └────────────┬────────────┘
                                   │
                  ┌────────────────┴──────────────┐
                  ▼                               ▼
        Trusted Exemplar Retrieval        Candidate References
                  └────────────────┬──────────────┘
                                   ▼
                      ┌─────────────────────────┐
                      │ Constrained Resolver    │
                      │ LLM / reranker          │
                      └────────────┬────────────┘
                                   ▼
                      ┌─────────────────────────┐
                      │ Strict Verification Gate│
                      └────┬─────┬─────┬────────┘
                           │     │     │
                    AUTO_LINK REVIEW NEW_REF / ABSTAIN
```

---

## 6. Canonical Reference 数据模型

### 6.1 `canonical_reference`

```sql
CREATE TABLE canonical_reference (
    ref_id BIGSERIAL PRIMARY KEY,
    canonical_brand TEXT NOT NULL,
    canonical_reference TEXT NOT NULL,
    collection TEXT,
    family TEXT,
    category TEXT,
    status TEXT NOT NULL,            -- active/deprecated/pending
    catalog_version BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    UNIQUE (canonical_brand, canonical_reference)
);
```

“同一个商品”的业务 ID 直接变成：

```text
(canonical_brand, canonical_reference)
```

品牌字段也必须在 identity 中，否则不同品牌可能存在看起来一样的编号。

### 6.2 `reference_alias`

```sql
CREATE TABLE reference_alias (
    alias_id BIGSERIAL PRIMARY KEY,
    ref_id BIGINT NOT NULL,
    raw_alias TEXT NOT NULL,
    normalized_alias TEXT NOT NULL,
    alias_type TEXT NOT NULL,         -- punctuation/spacing/source_alias/ocr
    source TEXT,
    confidence_level TEXT NOT NULL,   -- gold/silver/weak
    UNIQUE (ref_id, normalized_alias)
);
```

例：

```text
126610LN
126610 LN
126610-LN
REF126610LN
Ref. 126610 LN
```

这些可以映射到同一个 canonical reference，但必须保留 raw form 和来源，便于审计。

### 6.3 `reference_exemplar`

只保存高可信示例，用于 Product Retrieval：

```sql
CREATE TABLE reference_exemplar (
    exemplar_id BIGSERIAL PRIMARY KEY,
    ref_id BIGINT NOT NULL,
    listing_id TEXT NOT NULL,
    title TEXT,
    description TEXT,
    quality_level TEXT NOT NULL,
    verified_by TEXT NOT NULL,
    embedding_version TEXT,
    UNIQUE (listing_id)
);
```

### 6.4 `reference_resolution`

```sql
CREATE TABLE reference_resolution (
    listing_id TEXT PRIMARY KEY,
    source TEXT NOT NULL,
    ref_id BIGINT,
    decision TEXT NOT NULL,
    raw_mentions JSONB,
    evidence JSONB,
    conflicts JSONB,
    candidate_scores JSONB,
    resolver_version TEXT NOT NULL,
    catalog_version BIGINT NOT NULL,
    model_version TEXT,
    created_at TIMESTAMP NOT NULL
);
```

每次自动链接都可以回答：

- 当时用了哪个 catalog；
- 哪个模型；
- 看到了哪些 reference mention；
- 哪些候选被排除；
- 为什么最终链接。

---

## 7. 具体流水线

## 7.1 Stage A：Normalization

先做确定性的 normalization，不要急着上模型。

```python
def normalize_reference(s: str) -> str:
    s = unicode_nfkc(s)
    s = s.upper()
    s = normalize_dash(s)
    s = remove_prefixes(s, ["REF", "REF.", "REFERENCE", "型号"])
    s = normalize_spaces(s)
    return brand_specific_rules(s)
```

注意：

> normalization 不能无脑删除所有标点。

某些品牌 reference 中 `-`、`.`、`/` 可能携带结构信息。应该按品牌建立 formatter/parser，而不是全局一个正则。

## 7.2 Stage B：Literal Candidate Layer

证据优先级：

1. 结构化 `reference/model` 字段；
2. 标题明确 reference mention；
3. 描述；
4. 图片 OCR；
5. 语义检索。

首先用 catalog alias 做 Aho-Corasick / trie / regex / inverted index exact lookup。

如果能得到唯一、高可信 exact canonical ref，很多请求完全不需要 LLM。

## 7.3 Stage C：Reference Retriever

对 exact/alias 无法唯一确定的 listing，再调用 TACLR-style retriever。

### Query 表示

```text
brand: Rolex
category: watch
series: Submariner
reference attribute
listing title: ...
description: ...
ocr tokens: ...
```

### Candidate 表示

```text
A Rolex watch with reference number 126610LN,
collection Submariner,
aliases [126610 LN, 126610-LN]
```

### 检索分区

优先：

```text
canonical_brand + category
```

能可靠识别 series 时再加 series，但不要因为 series 分类错而丢掉真实 reference。

### Top-K

首版建议 K=5~6。

对于 100 万–1000 万 listing，不做 listing 两两向量比较；reference catalog embedding 预计算，在线/批处理只做 listing -> reference ANN。

可用：

- 首版：PostgreSQL + pgvector HNSW，或 FAISS；
- catalog 达百万级、吞吐要求继续上涨时：Milvus/OpenSearch k-NN/独立 FAISS shard。

## 7.4 Stage D：Trusted Product Retrieval

只在同 brand/category 内检索高可信 exemplar，默认 3~4 个即可。

加入以下过滤：

```text
quality_level = gold
OR human_verified = true
OR strong_multi_evidence = true
```

并保证：

- 不让同一原始 listing 的重复抓取占满上下文；
- 不让单一来源占据全部 exemplar；
- reference 冲突的示例不入库；
- 新模型预测结果不能未经验证直接成为 RAG 教材。

## 7.5 Stage E：Constrained Reference Resolver

建议把模型任务从：

```text
“这个商品是什么型号？”
```

改成：

```text
“根据证据，在候选集合中选择一个 canonical reference；
如没有足够证据，请返回 UNKNOWN；
如果文本明确包含候选列表之外的新 reference-like token，返回 NEW_REFERENCE_PENDING；
禁止根据外观/语义相似度猜型号。”
```

输出必须走 JSON Schema / grammar constrained decoding。

### Prompt 关键规则

```text
1. Candidate reference 只用于帮助解析，不代表其中一定有正确答案。
2. 自动候选必须有当前 listing 自己的证据，不得只因为 retrieved exemplar 相似而选择。
3. 如果多个候选 reference 都可解释，ABSTAIN。
4. 如果标题、结构化字段、OCR 互相冲突，ABSTAIN。
5. 如果发现候选列表之外的明确 reference token，NEW_REFERENCE_PENDING。
6. 不得把“适配/兼容/表带适用”的 reference 当作当前商品 reference。
```

---

## 8. Strict Verification Gate：真正决定能不能自动链接

这是论文原方案最需要新增的一层。

### 8.1 强证据

可以定义：

```text
E1 = structured reference exact canonical match
E2 = title reference exact/alias canonical match
E3 = description reference exact/alias canonical match
E4 = OCR reference exact/alias canonical match
E5 = constrained resolver selected same candidate
```

### 8.2 冲突信号

```text
C1 = structured ref 与 title ref 不一致
C2 = title 出现两个不同 canonical ref
C3 = OCR 明确出现另一个 canonical ref
C4 = brand 不一致
C5 = 当前 listing 是表带/配件/盒证/适配件
C6 = top1/top2 reference 过近且没有字面证据区分
C7 = reference-like token 不在 catalog
```

### 8.3 建议状态机

```python
def resolve(listing):
    evidence = collect_evidence(listing)
    literal = literal_candidates(evidence)

    if has_hard_conflict(evidence, literal):
        return ABSTAIN

    # 最安全的 fast path
    if unique_exact_structured_ref(literal) and no_conflict(evidence):
        return AUTO_LINK

    candidates = retrieve_reference_candidates(listing, k=6)
    exemplars = retrieve_verified_exemplars(listing, k=4)
    model_out = constrained_resolver(listing, candidates, exemplars)

    if model_out.decision == "NEW_REFERENCE_PENDING":
        return NEW_REFERENCE_PENDING

    if model_out.decision != "CANDIDATE":
        return ABSTAIN

    chosen = model_out.canonical_reference

    # LLM 不能凭空新增 identity
    if chosen not in candidates:
        return ABSTAIN

    # 至少要求 listing 自身有可验证 reference 证据
    if not current_listing_has_reference_evidence(chosen, evidence):
        return ABSTAIN

    if contradiction_exists(chosen, evidence):
        return ABSTAIN

    if not calibrated_high_precision_gate(listing, chosen):
        return REVIEW

    return AUTO_LINK
```

关键点：

> **LLM 选中了候选，不等于可以链接；它还必须通过当前 listing 自身证据检查和独立 precision gate。**

---

## 9. 图片怎么用才安全

当前需求有图片，这是纯文本 MVP-RAG 的可补强点。

推荐分两层：

### 9.1 OCR evidence

优先 OCR：

- 表背；
- 保卡；
- 吊牌；
- 包装标签；
- 说明书/证书。

OCR 后复用同一套 reference normalizer + catalog lookup。

### 9.2 Visual evidence

通用 CLIP/视觉模型可以用于：

- 识别腕表 vs 配件；
- 识别大类/系列；
- 做冲突检查；
- 决定是否需要人工复核。

但不要定义：

```text
image similarity > 0.95 => same reference
```

因为同系列不同 reference 往往外观极近。视觉相似只能提高候选召回，不能单独成为 identity proof。

---

## 10. 千万级与持续增量如何落地

## 10.1 从 O(N²) 改成 O(N × TopK)

传统三源 pairwise：

```text
source A × source B
source A × source C
source B × source C
```

数据上涨后很快不可控。

Reference-first：

```text
每条 listing -> TopK canonical references
```

如果 catalog 有 M 个 reference，ANN 检索成本和 listing 数近似线性增长，不再构造 N² pair。

## 10.2 增量事件

```text
listing_created/updated
    -> normalize
    -> extract evidence
    -> exact lookup fast path
    -> uncertain only: vector retrieval
    -> uncertain only: RAG/LLM
    -> gate
    -> persist resolution + versions
```

### Fast path

大量拥有高质量结构化 reference 的 listing 可以直接走：

```text
normalize -> catalog exact lookup -> conflict check -> AUTO_LINK
```

只有灰区进入昂贵模型。

这比“1000 万条全跑 7B LLM”更合理。

## 10.3 Catalog 增量

新 reference 进入 catalog 时：

```text
1. 写 canonical_reference
2. 写 aliases
3. 生成 reference embedding
4. ANN index upsert
5. 将此前 NEW_REFERENCE_PENDING / ABSTAIN 中含相同 raw token 的 listing 重放
```

无需重新跑全量历史。

## 10.4 模型与数据版本

每个结果至少记录：

```text
catalog_version
normalizer_version
retriever_version
resolver_version
ocr_version
```

否则某次 catalog 或模型更新导致错误时无法回溯。

---

## 11. 几百对黄金标签怎么花最值

本需求允许人工标注几百对。不要随机抽几百条容易样本。

应该专门做 **hard-case gold set**。

### 11.1 正例

覆盖：

- reference 结构化字段存在；
- reference 只在标题；
- 只在描述；
- 只在图片 OCR；
- 不同空格/连字符/大小写；
- 平台别名；
- 新 reference；
- 字段缺失。

### 11.2 难负例

优先：

```text
同品牌 + 同系列 + 相邻 reference
同品牌 + 外观近似 + reference 不同
主商品 vs “适配某 reference”的配件
reference vs 平台 SKU
reference vs 店铺货号
标题 ref 与图片 ref 冲突
旧款/新款只差少数字符
```

这比随机负例更能测出真实 false positive 风险，也和 TACLR taxonomy-aware hard negative 的思想一致。

### 11.3 数据切分

不能随机把同一个 reference 的近重复 listing 同时放 train/test，否则指标虚高。

建议至少做：

```text
A. seen-reference test
B. unseen-reference / OOD test
C. hard-neighbor reference test
D. source-shift test
E. image/text conflict test
```

---

## 12. 评估指标：不要只看 F1

对这个业务，主指标应该是：

```text
1. AUTO_LINK Precision
2. False Merge Count
3. AUTO_LINK Coverage
4. Abstention Rate
5. Candidate Recall@K
6. Reference Exact Match Accuracy
7. NEW_REFERENCE_PENDING Precision
8. 各来源/品牌/系列 slice 的 precision
```

上线条件不要写成：

```text
F1 > 90%
```

而应该类似：

```text
AUTO_LINK precision 的保守置信下界达到目标；
且 hard-negative 集无不可接受 false merge；
否则继续 abstain/review。
```

这可以直接接上 c 前一次 `Confidence Classifiers with Guaranteed Accuracy or Precision` 的思路：把模型做成 selective predictor，而不是强迫每条 listing 都输出 reference。

---

## 13. 训练方案：首版不建议全参微调 7B

论文用 Qwen2.5-7B-Instruct 做 full-parameter fine-tuning，这在论文的大规模工业训练集上合理，但当前需求只有几百人工黄金样本，直接复制并不划算。

推荐顺序：

### Phase 1：无训练/弱监督

- brand/reference normalizer；
- alias catalog；
- exact/regex 候选；
- 通用 embedding + ANN；
- constrained prompt；
- 全部灰区先 REVIEW/ABSTAIN。

### Phase 2：Retriever 微调

用自动构造的高置信正例 + hard negatives 训练 dual encoder：

```text
positive:
listing -> verified canonical ref

negative:
同 brand / family 的其他 ref
```

### Phase 3：轻量 Resolver SFT/LoRA

只用人工复核过的边界样本训练：

```text
CANDIDATE
ABSTAIN
NEW_REFERENCE_PENDING
CONFLICT
```

目标不是让模型记住几万 reference，而是训练它**遵守拒识协议和证据规则**。

---

## 14. 可以直接实现的 MVP

### 服务划分

```text
1. catalog-service
   - canonical reference / alias CRUD
   - catalog version

2. evidence-service
   - text normalization
   - reference-like token extraction
   - OCR ingestion

3. retrieval-service
   - reference embedding index
   - trusted exemplar index

4. resolver-service
   - constrained LLM/reranker
   - structured JSON output

5. decision-service
   - deterministic strict gate
   - audit reason codes

6. review-console
   - human review
   - new reference approval
   - hard-negative feedback
```

### 首版技术栈建议

```text
Python + FastAPI / worker
PostgreSQL：catalog / alias / resolution metadata
pgvector HNSW 或 FAISS：reference candidate retrieval
对象存储：图片
OCR：已有能力优先；无则引入独立 OCR 服务
LLM serving：内部模型或 vLLM 部署的指令模型
ClickHouse（可选）：千万级 listing/evidence 分析与离线评估
Kafka/Pulsar（可选）：已有事件总线时接入增量任务
```

不要为了“架构完整”一开始就引入所有中间件。最小版本完全可以：

```text
Postgres + pgvector + batch worker + 一个 resolver service
```

先验证 precision 和 reference catalog 质量。

---

## 15. 两周级可交付路线

### 第 1 阶段：Canonical catalog + baseline

- 统一三源 brand；
- 建立一批腕表 canonical reference；
- reference normalizer；
- structured/title exact match；
- 输出 AUTO_LINK / ABSTAIN；
- 建 hard-negative gold set。

### 第 2 阶段：Retriever

- reference embeddings；
- brand/category 分区 ANN；
- Top-K=5~6；
- 测 candidate recall@K；
- 针对相邻 reference 加 hard negatives。

### 第 3 阶段：MVP-RAG 灰区解析

- verified exemplar index；
- constrained JSON resolver；
- NEW_REFERENCE_PENDING；
- 冲突 reason codes；
- 首先只进入 REVIEW，不放 AUTO_LINK。

### 第 4 阶段：Selective auto-link

- 用黄金集做阈值/风险校准；
- 仅打开最安全的一小部分自动链接；
- 记录所有 decision evidence；
- 按品牌逐步放量。

---

## 16. 最终推荐

MVP-RAG 对当前需求最值得参考的，不是“再加一个 LLM”，而是把 reference 识别从一次性生成，改造成**有候选空间、有可信示例、有拒识能力的检索增强解析问题**。

但最终生产设计必须比论文更保守：

```text
MVP-RAG 原始思想：
retrieval -> generation -> standardized value

当前需求应改成：
closed-world reference retrieval
    -> constrained resolver
    -> independent evidence verification
    -> selective auto-link / abstain
```

推荐最终业务不保存“listing A 与 listing B 匹配”的大量 pair，而保存：

```text
listing_id -> canonical_ref_id
```

这样可以同时解决：

- 三源跨平台实体对齐；
- 新来源持续接入；
- 百万到千万级扩展；
- reference 长尾与 OOD；
- 高 precision；
- 全链路可解释、可回滚、可重放。

一句话方案：

> **用 MVP-RAG/TACLR 做 reference 候选与规范化，用可信 exemplar 帮助灰区解析，用 `UNKNOWN/NEW_REFERENCE_PENDING` 承接 OOD，最后用独立的 strict evidence gate 决定 AUTO_LINK；模型负责“找答案”，规则与证据负责“允许合并”。**

---

## 17. 参考资料

1. MVP-RAG — Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification  
   https://aclanthology.org/2025.emnlp-industry.147/  
   https://arxiv.org/abs/2509.23874

2. TACLR — A Scalable and Efficient Retrieval-based Method for Industrial Product Attribute Value Identification  
   https://arxiv.org/abs/2501.03835

3. 本仓库候选调研清单：`奢侈品文章调研.md`

4. 本需求 Notion Spec：  
   https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31
