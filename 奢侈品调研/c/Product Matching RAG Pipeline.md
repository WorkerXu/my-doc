# Product Matching RAG Pipeline：把“向量召回 + LLM 判断”改造成 Reference-First 的零误匹配腕表实体系统

> 分析人：c  
> 目标需求：跨源二奢/腕表商品实体匹配（雷小安 × 腕表之家 × 奢当家）  
> 核心约束：100 万–1000 万级持续增量；字段高度稀疏；reference 可能在结构化字段、标题或图片中；“同一个商品”定义为**同一 reference number / 型号**；**precision 极端优先，绝不能误匹配，允许漏匹配**。

## 0. 选题、去重与结论先行

本次从 `奢侈品文章调研.md` 选取：

- 项目：**Product Matching RAG Pipeline / product-matching-rag**
- GitHub：`https://github.com/Abhisheksasidharann/product-matching-rag`
- 调研清单中的描述：2026 年 WDC Product Matching RAG 项目，使用向量召回、属性信息和严格 LLM 判别，并重点处理相近变体误匹配。
- 需求 Spec：`https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31`

执行前重新检查 `奢侈品调研/c/`。当前已经存在的分析包括：

1. `Ameli：Enhancing Multimodal Entity Linking with Fine-Grained Attributes.md`
2. `AnyMatch – Efficient Zero-Shot Entity Matching with a Small Language Model.md`
3. `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
4. `Conformal Selective Prediction with General Risk Control.md`
5. `DeepBlocker.md`
6. `Efficient Model Repository for Entity Resolution：Construction, Search, and Integration.md`
7. `End-to-end multi-modal product matching in fashion e-commerce.md`
8. `Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration.md`
9. `GraLMatch：Matching Groups of Entities with Graphs and Language Models.md`
10. `How to Fix a Broken Confidence Estimator：Evaluating Post-hoc Methods for Selective Classification with Deep Neural Networks.md`
11. `Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes.md`
12. `MOON2.0：Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding.md`
13. `Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification.md`
14. `PAM：Understanding Product Images in Cross Product Category Attribute Extraction.md`
15. `Progressive Fine-Tuning for Cost-Effective Structured Attribute Generation in E-commerce.md`
16. `Query Brand Entity Linking in E-Commerce Search.md`
17. `Tailoring entity resolution for matching product offers.md`
18. `TransClean：Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency.md`
19. `Using LLMs for the Extraction and Normalization of Product Attribute Values.md`
20. `parts-distributor-sku-classifier.md`
21. `pyJedAI.md`

`Product Matching RAG Pipeline` 尚未被 c 分析，因此满足去重要求。

### 结论先行

这个项目**非常适合当工程原型参考，但不适合原样作为当前 Spec 的最终匹配器**。

它最值得复用的是下面的分层：

```text
脏商品数据
  -> Normalize
  -> Dense Embedding
  -> FAISS Top-K Retrieval
  -> LLM Strict Verification
  -> Match / No Match
```

这比直接做全量 pairwise matching 合理很多：召回阶段负责缩小搜索空间，昂贵的推理只看少量候选。

但是当前需求有一个比通用商品匹配强得多的定义：

> **同一个商品 = 同一个 reference number / 型号。**

所以真正应该落地的架构不是“让 LLM 判断是不是同一商品”，而是：

```text
三源原始 listing
  -> 品牌/字段归一化
  -> Reference Candidate Extraction
       ├─ 结构化 reference 字段
       ├─ 标题/描述规则 + 品牌 reference 词典
       ├─ 图片 OCR
       └─ 受约束 LLM（只抽取，不自由生成）
  -> Reference Role Classification
       ├─ PRIMARY_REFERENCE
       ├─ COMPATIBLE_REFERENCE
       ├─ PLATFORM_SKU
       ├─ SELLER_SKU
       └─ UNKNOWN
  -> Brand-Specific Canonicalization
  -> Evidence Ledger
  -> Exact Reference Index (brand_id, canonical_reference)
  -> Strict Decision Gate
       ├─ AUTO_LINK
       ├─ ABSTAIN / REVIEW
       └─ NO_REFERENCE
```

**向量、LLM 和图片都只能帮助“找到/验证 reference 证据”，不能越权成为最终同款真值。**

这是本次分析最核心的落地结论。

---

## 1. 原项目解决什么问题

项目使用 Web Data Commons Product Corpus。README 描述的数据量在 130 万以上商品 offer，特点和当前三源二奢数据很接近：

- 标题和描述噪声大；
- brand 等属性经常缺失；
- 没有统一 UPC/GTIN；
- 相近 variant 极难区分；
- 需要先从百万级数据召回小候选集，再做更严格判断。

项目把商品 A 作为 query，把商品 B 建成向量索引，先通过 SentenceTransformer + FAISS 找 Top-K，再让 LLM 看 query 与候选的 brand、title、price、文本详情，输出最终 match/no-match。

README 的整体流程是：

```text
[WDC Corpus]
   |
   +-> filter_clusters.py
   |
   +-> build_catalog_split.py
   |      +-> catalog_a.jsonl
   |      +-> catalog_b.jsonl
   |
   +-> normalize_offers.py
   |
   +-> sample_catalog.py
   |
   +-> build_embeddings.py
   |      +-> SentenceTransformer
   |      +-> FAISS IndexFlatIP
   |      +-> metadata JSON
   |
   +-> retrieve.py
   |      +-> Top-K candidate retrieval
   |
   +-> match_with_llm.py
          +-> Top-5 candidates
          +-> Claude structured JSON decision
          +-> TP / FP / FN / TN
```

代码量很小，适合当一个可以快速复刻的 PoC，而不是完整生产系统。

---

## 2. 源码架构深入拆解

## 2.1 `build_catalog_split.py`：先把笛卡尔积问题变成 Catalog Retrieval

它对原始 gzip JSONL 做两遍流式扫描：

```text
Pass 1:
    stream rows
    -> Counter(cluster_id)

Pass 2:
    cluster size == 1 -> singletons.jsonl
    第一次遇到 cluster -> catalog_a.jsonl
    后续同 cluster -> catalog_b.jsonl
```

源码的重要工程点是：

- 不把 130 万行一次性载入内存；
- 只在 RAM 中保存 `cluster_id -> count` 和已见 cluster set；
- 其余数据直接流式写 JSONL。

这个模式可以直接迁移到 1000 万级：**大文件处理优先用 streaming / chunk，不要先做 pandas 全量载入。**

但对于当前需求，不应该把 `cluster_id` 的构造继续交给 pairwise matcher。因为业务已经告诉我们 cluster 的自然主键是什么：

```text
cluster_key = brand_id + canonical_reference
```

如果 reference 已经可靠抽出，那么一个商品不需要和其他 1000 万条记录逐对比较，直接 O(1)/O(logN) exact lookup 即可。

换句话说：原项目用 Retrieval 来发现“谁可能和我相同”，而当前系统应该首先尝试直接回答“这个 listing 属于哪个 reference entity”。

---

## 2.2 `normalize_offers.py`：通用商品 normalization 对腕表 reference 不够严格

这个脚本做了：

1. 两个正则排除 vehicle / real-estate；
2. price 清洗为 float；
3. title 如果以 brand 开头，则移除重复 brand；
4. 拼：

```text
brand | clean_title | description[:200]
```

作为 `composite_text`；
5. 没有可用文本时直接 reject。

源码中的核心构造基本等价于：

```python
parts = []
if brand:
    parts.append(brand)
if title:
    parts.append(title)
if description:
    parts.append(description[:200])
composite_text = " | ".join(parts)
```

这个设计对通用语义 embedding 很合理，但对 reference-first 系统有三个风险。

### 风险 1：把所有 token 放进一条语义文本，会稀释 identifier

腕表标题可能是：

```text
劳力士 潜航者 126610LN 黑水鬼 全套 2022 年 95 新
```

真正决定同款的是：

```text
126610LN
```

而不是“黑水鬼”“全套”“95 新”。

通用 dense embedding 会把系列、材质、营销词都揉到一个向量中，相邻 reference 极容易非常接近。

### 风险 2：description 只取前 200 字符

如果 reference 埋在长描述后面，它根本不会进入 embedding，也不会进入后续 LLM prompt 的有效上下文。

当前需求应该先做**全字段 reference scan**，再决定哪些文本送 embedding；不能反过来靠 embedding 希望模型自己注意到编号。

### 风险 3：price 对“同 reference”不是身份主键

二奢价格受成色、附件、年份、卖家、渠道影响很大。price 更适合：

- 异常检测；
- 人工审核提示；
- fraud / 数据质量信号。

不应成为自动 identity merge 的正向证据。

---

## 2.3 `build_embeddings.py`：一个干净的 SentenceTransformer + FAISS 原型

项目使用：

```python
SentenceTransformer("all-MiniLM-L6-v2")
```

批大小：

```python
BATCH_SIZE = 64
```

并指定：

```python
normalize_embeddings=True
```

所以向量先做 L2 normalization，然后使用：

```python
faiss.IndexFlatIP(dim)
```

内积就等价于 cosine similarity。

同时它保存一个 `id_map`：

```json
{
  "position": 123,
  "id": "...",
  "cluster_id": "...",
  "title": "...",
  "composite_text": "...",
  "brand": "...",
  "price": 123.0,
  "priceCurrency": "..."
}
```

因此向量位置可以映射回商品元数据，后续 LLM 不需要重新扫描 JSONL。

### 对当前需求有用的思想

这个“**向量索引与业务 metadata 分离**”的接口值得保留：

```text
ANN index:
    vector -> offer_id

Metadata store:
    offer_id -> normalized fields / evidence / raw payload
```

### 不能直接照搬的地方

`IndexFlatIP` 是 exact brute-force search。对于 5 万样本很好，但 100 万–1000 万、持续增量时会出现：

- 每个 query 都需要扫描大量向量；
- 全量 `id_map.json` 放内存越来越重；
- 更新/删除/版本管理缺乏生产语义；
- 索引本身无法承担业务主数据和审计职责。

更关键的是：**只要 reference 已经抽出，根本不应该走 ANN。**

因此当前项目的索引应该拆为两层：

```text
Tier 1 — Primary exact index
    (brand_id, canonical_reference) -> reference_entity / offers

Tier 2 — Fallback retrieval index
    unresolved offer -> likely reference / likely candidate offers
```

Tier 2 可以使用 pgvector HNSW、OpenSearch kNN、FAISS HNSW/IVF 等，但它只服务：

- reference 缺失；
- reference 抽取冲突；
- 人工审核候选；
- reference 词典候选检索。

**Tier 2 的结果永远不能直接 AUTO_LINK。**

---

## 2.4 `retrieve.py`：Top-K 召回只解决“看谁”，不解决“是不是”

`retrieve.py` 的关键逻辑非常简单：

```python
q_vec = model.encode([query_text], normalize_embeddings=True)
scores, indices = index.search(q_vec, TOP_K)
```

`TOP_K = 10`。

它再用 ground-truth `cluster_id` 检查 Top-10 是否包含正确 cluster，README 给出的 retrieval-only 结果为：

```text
Recall@10 = 0.896
```

这个指标回答的是：

> 真正同一商品有没有进入候选集？

它完全不回答：

> Top-10 里这些长得很像的商品是否会被错误合并？

对当前 Spec，召回层指标仍然要保留，但要改名和分层：

```text
Reference Extraction Recall
Reference Candidate Recall@K
Hard-Negative Candidate Coverage
```

最终 AUTO_LINK 的 precision 必须单独测，不能和 retrieval recall 混在一起。

---

## 2.5 `match_with_llm.py`：项目真正的“高 precision”来源

源码设置：

```text
RETRIEVAL_K = 10
LLM_K = 5
NUM_QUERIES = 50
```

即：

```text
FAISS Top-10
   -> 截取前 5
   -> LLM
```

System prompt 定义：

```text
same product = identical item customer would consider interchangeable
不同 color / size / pack quantity 不是同商品
不同 seller / price 可以是同商品
```

并要求 JSON：

```json
{
  "match_found": true,
  "matched_candidate": 1,
  "confidence": "high",
  "evidence": "..."
}
```

Prompt 会给 LLM：

- query title；
- brand；
- price；
- composite text；
- 每个候选的同类字段；
- FAISS similarity。

项目最有价值的点在这里：**embedding 只负责相似召回，LLM 专门检查 variant 差异。**

比如：

```text
8GB RAM
64GB RAM
```

dense embedding 很接近，但 LLM 可以因为容量冲突拒绝。

这个“第二阶段 verifier”思想可以迁移到腕表，不过 verifier 的职责必须改变。

原项目：

```text
LLM -> 决定 SAME / NOT SAME
```

当前需求应该改成：

```text
LLM -> 抽取/解释 reference 证据
Deterministic Gate -> 决定 AUTO_LINK / ABSTAIN
```

原因很简单：LLM 可能看漏后缀、把 seller SKU 当型号、忽略“适配/兼容”语义，也可能在输出 `confidence=high` 时依然判断错。既然业务要求“绝不能误匹配”，最终判断不能建立在开放式语义推理上。

---

## 3. 项目声称的 100% Precision 为什么不能直接外推

README 报告：

```text
50-query end-to-end evaluation
Precision = 1.00
Recall = 0.87
F1 = 0.93
```

这个结果说明原型方向有价值，但不能理解为“生产可做到零误匹配”。源码里有几个很重要的评测边界。

## 3.1 只有 50 个 end-to-end query

0 FP / 50 并不构成“绝不误匹配”的统计保证。

对于一个真正 precision-first 的上线系统，应该单独审计**所有被系统自动接受的正例**。

如果某个版本人工抽检 `n` 个 AUTO_LINK，观察到 0 个错误，那么在简单 Bernoulli 假设下，单侧 95% 下界大致需要：

```text
至少 299 个零错误样本   -> 才能把 precision 95% 下界推到约 99%
至少 2,995 个零错误样本 -> 才能推到约 99.9%
至少 29,956 个零错误样本 -> 才能推到约 99.99%
```

所以“几百对黄金标签”很适合：

- 发现规则 bug；
- 校准抽取器；
- 构造 hard negatives；
- 比较版本。

但不能证明一个开放式 LLM matcher 达到 99.99% precision。

这也是为什么当前方案必须把安全性主要建立在**业务定义 + 硬证据 gate**上，而不是统计置信度上。

---

## 3.2 评测主动跳过“索引中没有真匹配”的 query

`retrieve.py` 和 `match_with_llm.py` 都先建立：

```python
indexed_clusters = set(...)
```

然后：

```python
if query_cluster not in indexed_clusters:
    continue
```

这意味着只有“索引里确实存在同 cluster 商品”的 query 才会进入评测。

生产中最危险的一类 false positive 恰恰是：

```text
当前来源出现一个商品
另一来源根本没有这个 reference
但系统硬从相似候选中选了一个
```

原项目没有用足够的 singleton / no-counterpart query 正面压力测试这个场景。

当前系统测试集必须显式包含：

```text
NO_MATCH_AVAILABLE
```

并把这类 query 的 false-positive rate 作为一级指标。

---

## 3.3 `sample_catalog.py` 并没有真正“保持完整 cluster”

README 写的是 50k sample 保持 ground-truth clusters intact，但实际源码是：

```python
random.sample(matching_rows, 50_000)
```

它按 row 随机采样，不是按整个 cluster 原子采样。

不过它通过 `indexed_clusters` 过滤保证被评测的 A query 至少有一个同 cluster B row 在样本里。

这对于快速 PoC 没问题，但意味着：

- 结果是一个条件评测；
- 不是自然线上流量分布；
- 不代表 130 万全量下的真实 precision/recall。

---

## 3.4 LLM 的 `confidence` 没有真正用于 gate

虽然 JSON 返回：

```text
high / medium / low
```

但源码最终只看：

```python
llm_result["match_found"]
```

所以理论上：

```json
{"match_found": true, "confidence": "low"}
```

也会算匹配。

对当前需求，这个语义完全不够安全。

我们的 AUTO_LINK gate 不应该是一个浮动概率阈值，而应该是多个强约束的 conjunction。

---

## 4. 对当前 Spec，最重要的架构转变：从 Pair Matching 改成 Reference Resolution

当前业务定义已经把问题大幅简化了。

普通 entity matching 是：

```text
record A + record B
    -> model
    -> same / different
```

当前需求本质应该建模为：

```text
record
    -> resolve canonical reference
    -> reference entity
```

于是跨源匹配自然变成：

```text
雷小安 listing ---------+
                       |
腕表之家 listing --------+--> Canonical Reference Entity
                       |
奢当家 listing ----------+
```

而不是维护三个两两 matcher：

```text
雷小安 <-> 腕表之家
腕表之家 <-> 奢当家
雷小安 <-> 奢当家
```

这种 reference-centric 架构有四个巨大好处：

1. 不存在 O(N²) pair explosion；
2. 三源、多源天然统一；
3. 新来源接入只需要做 schema / reference extraction 适配；
4. cluster 不依赖模型传递关系，避免一条错边污染整簇。

---

## 5. 直接可落地的目标架构

```text
                     +----------------------+
                     | 雷小安 / 腕表之家 / 奢当家 |
                     +----------+-----------+
                                |
                                v
                    +------------------------+
                    | 1. Raw Offer Ingestion |
                    | source + source_item_id|
                    +-----------+------------+
                                |
                                v
                    +------------------------+
                    | 2. Schema Normalizer   |
                    | brand/title/ref/images |
                    +-----------+------------+
                                |
                                v
              +----------------------------------------+
              | 3. Reference Evidence Extraction       |
              |                                        |
              | A. structured ref                      |
              | B. title/desc regex + dictionary       |
              | C. image OCR                           |
              | D. constrained LLM extraction          |
              +-------------------+--------------------+
                                  |
                                  v
              +----------------------------------------+
              | 4. Reference Role Classifier           |
              | PRIMARY / COMPATIBLE / SKU / UNKNOWN   |
              +-------------------+--------------------+
                                  |
                                  v
              +----------------------------------------+
              | 5. Brand-specific Canonicalizer        |
              | raw_ref -> canonical_ref               |
              +-------------------+--------------------+
                                  |
                                  v
              +----------------------------------------+
              | 6. Evidence Aggregator / Conflict Gate |
              +-------------------+--------------------+
                                  |
                  +---------------+----------------+
                  |                                |
                  v                                v
       +------------------------+       +-------------------------+
       | high-confidence binding|       | unresolved / conflict   |
       +-----------+------------+       +------------+------------+
                   |                                  |
                   v                                  v
       +------------------------+       +-------------------------+
       | 7. Exact Ref Index     |       | 8. Fallback Retrieval   |
       | brand + canonical_ref  |       | ANN / lexical / image   |
       +-----------+------------+       +------------+------------+
                   |                                  |
                   v                                  v
       +------------------------+       +-------------------------+
       | AUTO_LINK cluster      |       | REVIEW / ABSTAIN        |
       +------------------------+       +-------------------------+
```

核心原则：

> **只有左边路径可以 AUTO_LINK；右边路径无论语义多相似，都不能直接自动合并。**

---

## 6. Reference Extraction：应该是系统的核心能力

## 6.1 证据源优先级

建议每一次 reference 抽取都不是只保存一个字符串，而是保存 evidence。

### Tier A：结构化强证据

例如来源明确提供：

```text
reference_number
model_ref
型号
参考编号
```

但接入前必须先确认字段语义，避免某个平台的“型号”其实是内部 SKU。

保存：

```json
{
  "raw_value": "126610LN",
  "evidence_type": "STRUCTURED_FIELD",
  "field": "reference_number",
  "role": "PRIMARY_REFERENCE",
  "extractor_version": "lxan-v3"
}
```

### Tier B：标题/描述中的确定性抽取

流程不要直接：

```text
title -> LLM -> reference
```

而应该：

```text
brand detection
  -> brand-specific token generator
  -> known-reference dictionary validation
  -> context role check
```

例如：

```text
Rolex 126610LN
Omega 310.30.42.50.01.001
AP 15500ST.OO.1220ST.01
```

品牌 reference 形态差异很大，所以应该维护品牌级 pattern，不要一个全局正则吃所有品牌。

### Tier B/C：图片 OCR

图片的价值不是“看起来像同一只表”，而是：

- 表背刻字；
- 保卡 reference；
- 吊牌；
- 证书；
- 表盘可见型号相关文本。

推荐流程：

```text
image
 -> OCR
 -> candidate token extraction
 -> brand reference dictionary / pattern validation
 -> evidence
```

视觉相似度可以帮助找候选图片，但不能证明同 reference。

### Tier C：受约束 LLM

LLM 最适合处理：

- 标题里多个编号，判断哪个像主商品 reference；
- 中英文混杂上下文；
- “适配 XX / compatible with XX”否定上下文；
- 从 OCR 噪声候选中解释哪个更可信。

但是必须采用**substring-constrained extraction**。

LLM 输出：

```json
{
  "candidate": "126610LN",
  "role": "PRIMARY_REFERENCE",
  "source_span": "126610LN",
  "reason_code": "TITLE_EXPLICIT_MODEL_REF"
}
```

服务端再验证：

```text
candidate 必须真实存在于 title/description/OCR 文本
```

如果 LLM 输出一个原文不存在的 reference，直接丢弃。

这样可以阻止自由生成/幻觉进入 canonical key。

---

## 7. 最容易造成 false positive 的地方：Reference Role

“标题里出现同一个 reference”并不等于“当前卖的商品就是这个 reference”。

典型危险标题：

```text
适配 Rolex 126610LN 的表带
126610LN 兼容表扣
劳力士 126610LN 原装盒
适用于 116610 / 126610 的保护膜
店铺 SKU: 126610LN-8891
```

如果只做 token exact match，会产生灾难性误匹配。

所以必须引入 role：

```text
PRIMARY_REFERENCE
COMPATIBLE_REFERENCE
ACCESSORY_TARGET_REFERENCE
PLATFORM_SKU
SELLER_SKU
SERIAL_NUMBER
UNKNOWN
```

并规定：

```text
只有 PRIMARY_REFERENCE 可以生成 AUTO_LINK key。
```

### 中文负上下文词典

可以先用规则覆盖高风险词：

```text
适用
适配
兼容
for
配件
表带
表扣
保护膜
表盒
盒子
说明书
附件
同款风格
替换
replacement
compatible
fits
```

规则命中时不要直接判负，而是把 reference role 降级到 `COMPATIBLE_REFERENCE` / `UNKNOWN`，进入人工或模型复核。

这比“相似度够高就合并”安全得多。

---

## 8. Canonicalization：宁可少归一，也不能过归一

reference normalization 是 precision-first 系统最容易被低估的风险。

错误做法：

```text
删掉所有空格、点、横杠、斜杠
只保留字母数字
```

因为不同品牌的分隔符和 suffix 可能有语义。

正确做法分两层。

## 8.1 全局一定安全的字符标准化

例如：

```text
Unicode NFKC
全角 -> 半角
英文字母 uppercase
trim 首尾空白
标准化明显的 Unicode dash 字符
```

保留原值：

```text
raw_reference
```

同时生成：

```text
normalized_reference
```

## 8.2 Brand-Specific Canonicalization

建立：

```python
canonicalize(reference, brand_id)
```

不同品牌使用不同规则。

例如某品牌已验证：

```text
"310.30.42.50.01.001"
```

点号属于标准 reference 结构，就不应该被通用逻辑随意删除。

### Alias 也必须显式可审计

如果确认：

```text
ABC-123 == ABC123
```

不要写一个全局“去横杠”规则，而应该进入：

```text
reference_alias
```

表：

```text
brand_id
raw_pattern
canonical_reference
rule_source
approved_by
rule_version
```

未知归一化一律 ABSTAIN。

---

## 9. 最终 AUTO_LINK Gate：不要用单一 similarity threshold

推荐初版只允许最保守规则上线。

伪代码：

```python
def decide_auto_link(lhs, rhs):
    if lhs.brand_id != rhs.brand_id:
        return ABSTAIN("BRAND_CONFLICT")

    if not lhs.primary_reference or not rhs.primary_reference:
        return ABSTAIN("REFERENCE_MISSING")

    if lhs.primary_reference != rhs.primary_reference:
        return REJECT("REFERENCE_CONFLICT")

    if lhs.reference_role != "PRIMARY_REFERENCE":
        return ABSTAIN("LHS_REFERENCE_ROLE_UNSAFE")

    if rhs.reference_role != "PRIMARY_REFERENCE":
        return ABSTAIN("RHS_REFERENCE_ROLE_UNSAFE")

    if lhs.has_conflicting_reference_evidence:
        return ABSTAIN("LHS_CONFLICT")

    if rhs.has_conflicting_reference_evidence:
        return ABSTAIN("RHS_CONFLICT")

    if lhs.is_accessory_or_compatibility_listing:
        return ABSTAIN("LHS_ACCESSORY_RISK")

    if rhs.is_accessory_or_compatibility_listing:
        return ABSTAIN("RHS_ACCESSORY_RISK")

    if not lhs.reference_evidence_is_strong:
        return ABSTAIN("LHS_WEAK_EVIDENCE")

    if not rhs.reference_evidence_is_strong:
        return ABSTAIN("RHS_WEAK_EVIDENCE")

    return AUTO_LINK
```

注意这里没有：

```text
cosine_similarity > 0.95
LLM confidence == high
image_similarity > 0.98
```

这些最多只能作为：

```text
review priority
candidate retrieval
conflict signal
```

而不是 identity proof。

---

## 10. 更进一步：不要真的做 pairwise link，直接绑定 Reference Entity

数据库中建立：

```text
reference_entity
----------------
id
brand_id
canonical_reference
status
created_at
```

唯一约束：

```sql
UNIQUE (brand_id, canonical_reference)
```

listing 解析成功后直接：

```text
offer -> reference_entity_id
```

于是“同一个商品”天然是：

```sql
SELECT *
FROM offer_reference_binding
WHERE reference_entity_id = ?;
```

cluster ID 也可以稳定生成：

```text
sha256(brand_id || "\0" || canonical_reference)
```

这样无需：

- pairwise union-find；
- match graph 反复合并；
- A-B、B-C、A-C 三套关系；
- LLM 传递性修复。

只要 reference entity 本身正确，跨 N 个来源都天然一致。

---

## 11. 推荐数据模型

### 11.1 `offer`

```sql
CREATE TABLE offer (
    id                 BIGSERIAL PRIMARY KEY,
    source             SMALLINT NOT NULL,
    source_item_id     TEXT NOT NULL,
    brand_id           BIGINT,
    title              TEXT,
    description        TEXT,
    structured_ref_raw TEXT,
    image_urls         JSONB,
    raw_payload        JSONB NOT NULL,
    content_hash       BYTEA NOT NULL,
    source_updated_at  TIMESTAMPTZ,
    ingested_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source, source_item_id)
);
```

### 11.2 `reference_evidence`

```sql
CREATE TABLE reference_evidence (
    id                   BIGSERIAL PRIMARY KEY,
    offer_id             BIGINT NOT NULL,
    brand_id             BIGINT,
    raw_value            TEXT NOT NULL,
    canonical_candidate  TEXT,
    evidence_type        TEXT NOT NULL,
    evidence_location    TEXT,
    role                 TEXT NOT NULL,
    rule_id              TEXT,
    extractor_version    TEXT NOT NULL,
    confidence           DOUBLE PRECISION,
    is_deterministic     BOOLEAN NOT NULL,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 11.3 `reference_entity`

```sql
CREATE TABLE reference_entity (
    id                  BIGSERIAL PRIMARY KEY,
    brand_id            BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    state               TEXT NOT NULL DEFAULT 'ACTIVE',
    UNIQUE (brand_id, canonical_reference)
);
```

### 11.4 `offer_reference_binding`

```sql
CREATE TABLE offer_reference_binding (
    offer_id             BIGINT PRIMARY KEY,
    reference_entity_id  BIGINT,
    decision             TEXT NOT NULL,
    reason_code          TEXT NOT NULL,
    decision_version     TEXT NOT NULL,
    evidence_snapshot    JSONB NOT NULL,
    decided_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`decision` 建议只允许：

```text
AUTO_LINK
REVIEW
NO_REFERENCE
REJECTED_CONFLICT
```

不要在数据库里只有一个 `matched=true/false`，否则无法知道“不匹配”和“无法判断”的区别。

---

## 12. 100 万–1000 万级扩展策略

## 12.1 主路径根本不需要百万级向量检索

如果 reference 抽取成功，核心查询是：

```sql
SELECT id
FROM reference_entity
WHERE brand_id = ?
  AND canonical_reference = ?;
```

这是普通 B-tree / hash-friendly exact lookup。

1000 万商品真正应该扩的是：

- ingestion throughput；
- OCR throughput；
- reference extraction workers；
- metadata storage；
- evidence versioning；
- review queue。

而不是把 1000 万商品全量 pairwise 向量搜索作为主路径。

## 12.2 Fallback ANN 按 brand 分片

如果要保留原项目的 embedding 召回思想，建议：

```text
brand shard
  -> unresolved candidates only
  -> HNSW / IVF / OpenSearch kNN
```

不要跨所有品牌全局 Top-K。

理由：

- 品牌冲突本来就是硬否决；
- 分片减少候选量；
- 相同数字型 reference 跨品牌撞号很常见；
- 更容易为不同品牌维护不同 tokenizer / regex / reference dictionary。

---

## 13. 持续增量更新怎么做

每次来源抓到新增/变更商品：

```text
UPSERT offer(source, source_item_id)
  -> compare content_hash
  -> unchanged: skip
  -> changed/new:
       normalize
       extract reference evidence
       classify role
       canonicalize
       decision gate
       update binding
```

### 版本化必须保留

对每条绑定保存：

```text
extractor_version
canonicalizer_version
decision_version
```

以后规则升级时可以：

```text
只重跑受影响品牌 / 受影响规则 / REVIEW 状态商品
```

而不是每次重跑 1000 万条。

### cluster 变更也可确定性处理

旧绑定：

```text
A -> REF_OLD
```

规则升级后：

```text
A -> REF_NEW
```

只需要更新 binding；无需去图里拆边、重新算 connected components。

---

## 14. 图片应该怎么使用

当前 Spec 明确“有图片可用”，但图片最安全的价值不是 visual same-item classification，而是**发现 reference 的文字证据**。

优先顺序建议：

```text
1. 保卡/证书 OCR
2. 吊牌 OCR
3. 表背刻字 OCR
4. 其他图像 OCR
5. 视觉商品类别/附件识别
6. 视觉相似检索（仅 review candidate）
```

### OCR 不能单独 AUTO_LINK

OCR 容易出现：

```text
O <-> 0
I <-> 1
S <-> 5
B <-> 8
8 <-> 3
```

所以 OCR reference 应经过：

```text
OCR token
 -> brand pattern
 -> known reference dictionary
 -> edit candidates
 -> textual cross evidence
```

如果 OCR 得到 `12661OLN`，而品牌词典只有 `126610LN`，可以产生 review candidate，但不要静默 autocorrect 后直接合并。

---

## 15. LLM 在最终系统中的正确位置

原项目把 LLM 放在：

```text
candidate -> final identity decision
```

建议改为三个低风险角色。

### 角色 A：Reference span extraction

只返回原文子串：

```text
{"reference_span":"126610LN"}
```

服务端验证 span 存在。

### 角色 B：Reference role classification

输入：

```text
标题 + 候选编号 + 候选上下文
```

输出：

```text
PRIMARY / COMPATIBLE / SKU / UNKNOWN
```

但只有经过离线 precision 验证的高置信规则才可提升到 AUTO_LINK 路径，否则仍 review。

### 角色 C：人工审核解释器

对 REVIEW 任务生成：

```text
- 两侧 reference 证据
- 冲突字段
- OCR 图像证据
- 为什么无法自动放行
```

提高人工审核效率。

### 不建议的角色

```text
“这两条看起来是不是同一只表？”
```

作为最终自动决策。

---

## 16. 几百对黄金标签应该怎么花

用户允许人工标注几百对。不要随机抽几百对普通商品，信息密度太低。

建议黄金集强制包含 hard slices。

### Slice 1：同品牌同系列、reference 只差 1–2 个字符

例如：

```text
ABC123A
ABC123B
```

这是最重要的 false-positive 压力测试。

### Slice 2：配件/兼容标题

```text
for 126610LN
适配 126610LN
126610LN 表带
```

用于验证 role classifier。

### Slice 3：平台 SKU 与 reference 混淆

标题同时出现：

```text
reference + store_sku + serial
```

### Slice 4：多 reference 标题

例如：

```text
兼容 116610 / 126610
```

系统必须 ABSTAIN。

### Slice 5：OCR 混淆

专门包含：

```text
0/O, 1/I, 5/S, 8/B
```

### Slice 6：同 reference 字符串跨品牌碰撞

这用于证明为什么 exact key 必须是：

```text
brand_id + canonical_reference
```

而不能只用 reference 字符串。

### Slice 7：真实 no-match query

另一来源根本没有该 reference。

这是原 `product-matching-rag` 评测最不足的部分之一。

---

## 17. 推荐评测指标

不要只看 F1。

当前业务应该按以下优先级：

```text
P0  AUTO_LINK False Positive Count
P0  AUTO_LINK Precision
P0  Hard-Negative Precision
P0  No-Match Query False Positive Rate

P1  Reference Extraction Precision
P1  Reference Role Classification Precision
P1  Canonicalization Error Count

P2  AUTO_LINK Coverage
P2  Abstention Rate
P2  Reference Extraction Recall
P2  Review Queue Size

P3  Fallback Retrieval Recall@K
P3  OCR Candidate Recall
```

目标不是最大化整体 F1，而是：

```text
在可接受覆盖率下，让 AUTO_LINK 的误匹配趋近于 0。
```

如果某个新规则把 coverage 从 40% 提到 55%，但出现一个无法解释的 FP，应立即回滚该规则，而不是因为 F1 上升就上线。

---

## 18. 推荐上线策略：先硬规则闭环，再逐层吃掉 ABSTAIN

### Milestone 1：Reference-Exact MVP

只做：

```text
三源 schema mapping
brand normalization
结构化 reference
标题强规则 reference
brand-specific canonicalization
exact reference entity index
AUTO_LINK / ABSTAIN
```

不做向量，不做 LLM 最终判断。

这个版本应该先回答两个问题：

1. 多少商品已经可以靠确定性 reference 自动绑定？
2. 误匹配主要来自哪些 reference 角色/归一化问题？

### Milestone 2：Reference Dictionary + Title Context

增加：

```text
品牌 reference registry
标题 token candidate
PRIMARY vs COMPATIBLE role
高风险词上下文
```

目标是扩大结构化字段缺失时的覆盖率。

### Milestone 3：Image OCR

只处理没有可靠 reference 或存在冲突的 listing。

输出仍然进入相同 evidence ledger，不单独创造匹配逻辑。

### Milestone 4：Fallback Retrieval

此时再引入原项目最值得借鉴的：

```text
embedding -> Top-K -> strict verifier
```

但用途是：

```text
给 unresolved listing 找可能的 canonical reference / review candidate
```

而不是直接自动合并。

### Milestone 5：LLM Assisted Review / Extraction

把 LLM 放到最难的尾部样本，严格保留 abstain。

---

## 19. 可以直接复用原项目哪些代码思想

### 可直接复用 1：Streaming JSONL 处理

适合百万级输入，避免大 dataframe。

### 可直接复用 2：向量与 metadata 分离

```text
vector position -> offer_id -> metadata store
```

只是生产上 metadata 应进数据库，而不是巨大 JSON。

### 可直接复用 3：两阶段架构

```text
cheap retrieval -> expensive verifier
```

很适合 unresolved tail。

### 可直接复用 4：结构化 LLM JSON 输出

比自然语言结论容易审计和集成。

### 可直接复用 5：明确区分 retrieval 与 matching 指标

必须分别测候选 recall 和最终 precision。

---

## 20. 必须替换掉原项目哪些设计

| 原项目设计 | 当前需求建议 |
|---|---|
| `composite_text` 作为主要 identity 表征 | 独立 Reference Evidence Pipeline |
| `all-MiniLM-L6-v2` 全量主召回 | exact reference index 为主；ANN 只 fallback |
| `IndexFlatIP` 50k 原型 | exact DB index + brand-sharded ANN |
| LLM 决定 same product | deterministic reference gate 决定 AUTO_LINK |
| generic “interchangeable” 定义 | exact canonical reference 定义 |
| price 作为 prompt 信息 | price 只作为异常/审核信号 |
| 50-query LLM 评测 | hard-negative + no-match + AUTO_LINK audit |
| 跳过没有真匹配的 query | 必须保留真实 unmatched query |
| confidence 仅展示 | decision state + reason code + abstention |
| metadata JSON | DB + evidence ledger + versioning |
| 两 catalog pair matching | canonical reference entity 多源绑定 |
| 无图片 | OCR reference evidence |
| 无增量版本语义 | idempotent upsert + extractor/version replay |

---

## 21. 一个最小服务接口设计

### `POST /resolve-reference`

输入：

```json
{
  "source": "watchhome",
  "source_item_id": "123",
  "brand": "Rolex",
  "title": "劳力士潜航者 126610LN 黑水鬼",
  "reference": null,
  "description": "...",
  "images": ["..."]
}
```

输出：

```json
{
  "brand_id": 42,
  "canonical_reference": "126610LN",
  "decision": "AUTO_LINK",
  "reason_code": "TITLE_REFERENCE_DICTIONARY_EXACT",
  "evidence": [
    {
      "type": "TITLE_TOKEN",
      "raw_value": "126610LN",
      "role": "PRIMARY_REFERENCE",
      "deterministic": true
    }
  ]
}
```

### 如果有歧义

```json
{
  "canonical_reference": null,
  "decision": "REVIEW",
  "reason_code": "MULTIPLE_REFERENCE_CANDIDATES",
  "evidence": [
    {"raw_value": "116610LN", "role": "UNKNOWN"},
    {"raw_value": "126610LN", "role": "UNKNOWN"}
  ]
}
```

最重要的是：

> **系统宁可输出 REVIEW，也不要猜一个 reference。**

---

## 22. 示例：为什么 Reference-First 比原始 RAG 更安全

假设三条数据：

```text
A 雷小安
Rolex Submariner 126610LN 黑水鬼 全套

B 腕表之家
劳力士 潜航者 126610LV 绿水鬼

C 奢当家
劳力士 126610LN 2022 年 95 新
```

embedding 很可能得到：

```text
sim(A, B) = 很高
sim(A, C) = 很高
```

原项目把 B/C 都交给 LLM 判断。

Reference-First 直接得到：

```text
A -> Rolex / 126610LN
B -> Rolex / 126610LV
C -> Rolex / 126610LN
```

于是：

```text
A == C
A != B
```

不需要模型解释。

再看配件：

```text
D
适配 Rolex 126610LN 的第三方橡胶表带
```

简单 exact token 会错误得到：

```text
D -> 126610LN
```

加入 Reference Role 后：

```text
D reference token = 126610LN
role = COMPATIBLE_REFERENCE
product_kind = ACCESSORY
```

所以：

```text
D -> ABSTAIN / REJECT AUTO_LINK
```

这才符合“绝不能误匹配”。

---

## 23. 代码层面的推荐模块边界

```text
src/
  ingestion/
    leixiaoan.py
    watchhome.py
    shedangjia.py

  normalize/
    brand.py
    text.py
    source_schema.py

  reference/
    candidate_generator.py
    structured_extractor.py
    title_extractor.py
    ocr_extractor.py
    llm_extractor.py
    role_classifier.py
    canonicalizer.py
    brand_rules/
      rolex.py
      omega.py
      ap.py
      patek.py
      ...

  decision/
    evidence_aggregator.py
    auto_link_gate.py
    reason_codes.py

  retrieval/
    lexical.py
    vector.py

  storage/
    offer_repository.py
    reference_repository.py
    evidence_repository.py

  review/
    queue.py
    audit_sampler.py
```

核心接口：

```python
extract_reference_evidence(offer) -> list[ReferenceEvidence]
resolve_primary_reference(evidences) -> ReferenceResolution
bind_reference_entity(resolution) -> BindingDecision
```

这样以后替换 OCR、LLM、embedding，都不会改变最终 decision contract。

---

## 24. 一个必须坚持的系统不变量

建议把下面几条写成数据库/服务层 invariants，而不只是产品约定。

### Invariant 1

```text
AUTO_LINK 必须有 canonical_reference。
```

### Invariant 2

```text
同一 cluster 内所有 listing 的 brand_id 与 canonical_reference 必须完全一致。
```

### Invariant 3

```text
COMPATIBLE_REFERENCE / PLATFORM_SKU / UNKNOWN 不可创建 AUTO_LINK。
```

### Invariant 4

```text
任何 conflicting primary reference evidence -> ABSTAIN。
```

### Invariant 5

```text
模型分数、图片相似度、LLM confidence 不可单独升级成 AUTO_LINK。
```

### Invariant 6

```text
每个 AUTO_LINK 必须可回放：能恢复当时使用的 raw evidence、规则版本和 decision version。
```

做到这些，系统即使未来不断加入更强模型，也不容易因为模型迭代而突破 precision 安全边界。

---

## 25. 最终建议

`product-matching-rag` 给当前需求最大的启发不是“用 Claude 做商品匹配”，而是：

> **把大规模商品匹配拆成便宜召回与昂贵严格验证两阶段。**

但是因为本项目已经明确：

```text
same product == same reference number
```

所以可以比通用 RAG pipeline 做得更简单、更可靠：

```text
通用 RAG：
semantic retrieval
  -> LLM decides identity

当前项目：
reference resolution
  -> exact reference entity binding
  -> semantic/LLM only for unresolved tail
```

因此我建议的直接落地优先级是：

```text
1. source schema + brand normalization
2. deterministic reference extraction
3. reference role classification
4. brand-specific canonicalization
5. exact (brand, canonical_reference) entity index
6. strict AUTO_LINK / ABSTAIN gate
7. evidence ledger + audit
8. image OCR
9. fallback ANN retrieval
10. constrained LLM extraction/review
```

最终目标不是“让模型越来越敢匹配”，而是：

> **让系统在证据不足时越来越善于拒绝；只有 reference 身份证据充分、无冲突、角色明确时才自动合并。**

这条路线既保留了 `product-matching-rag` 在大规模候选召回上的工程价值，又避开了它把 LLM 作为最终 identity judge、评测样本过小、缺少真实 unmatched query、缺少图片/reference 专用处理等不适合当前 Spec 的部分。

对雷小安 × 腕表之家 × 奢当家三源数据，最推荐的生产主架构不是“百万商品做语义配对”，而是**“每条 listing 解析到 canonical reference entity，再按 reference entity 自然聚合”**。这也是在“precision 优先到极致、允许漏匹配”约束下最稳健、最可审计、也最易扩展到千万级持续增量的方案。