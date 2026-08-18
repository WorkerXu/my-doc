# Product Matching RAG Pipeline：技术实现深拆与二奢腕表 Reference-First 落地方案

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **Product Matching RAG Pipeline / product-matching-rag** 做深入分析。

- 调研条目：<https://github.com/Abhisheksasidharann/product-matching-rag>
- 项目 README：<https://github.com/Abhisheksasidharann/product-matching-rag/blob/main/README.md>
- 目标需求：<https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31>

分析前已读取 `奢侈品调研/a` 当前目录，确认此前结果中**没有** `Product Matching RAG Pipeline.md` / `product-matching-rag.md`，因此本次不是重复分析。

这个项目与当前 Spec 非常接近：都是脏、稀疏、多来源商品数据，先高召回找候选，再用严格判别器过滤 false positive。项目当前代码的主链路是：

```text
WDC 商品语料
  -> 流式清洗 / 标准化
  -> SentenceTransformer embedding
  -> FAISS Top-K retrieval
  -> LLM 对候选做 match / no-match
  -> TP/FP/FN/TN 评测
```

但当前业务有一个比通用 product matching 更强的定义：

> **同一个商品 = 同一个 reference number / 型号。**

并且：

> **误匹配不可接受，precision 极端优先，允许 abstain / 漏匹配。**

所以这个项目最值得借鉴的是“**召回与判定解耦**”的架构，不应照搬“LLM 作为最终匹配器”。

推荐直接改造成：

```text
三源商品记录
  -> 数据标准化
  -> 品牌规范化
  -> Reference 候选抽取（结构化字段 / 标题 / 描述 / OCR）
  -> Reference 角色判别（商品自身 reference vs 配件兼容型号 vs 平台 SKU）
  -> Canonical Reference 解析
        |
        |-- 强证据且唯一 --> Exact Reference Join --> 自动 ACCEPT
        |
        `-- reference 缺失/冲突 --> 高召回 Candidate Retrieval
                                    -> 证据验证器（规则/LLM/图像）
                                    -> 只产出 reference 候选或 REVIEW
                                    -> 不直接越权合并

最终实体主键：
    (brand_id, canonical_reference)
```

一句话：

> **把 product-matching-rag 的 FAISS + LLM 从“最终匹配系统”降级为“缺失 reference 时的候选发现与证据补全系统”，把最终自动合并权交给可审计的 Canonical Reference Gate。**

---

## 1. 为什么这个项目值得参考

当前 Spec：

- 来源：雷小安、腕表之家、奢当家；
- 数据量 100 万～1000 万级；
- 持续增量；
- 字段稀疏；
- reference 有时为独立字段，有时埋在标题；
- 有图片；
- 可人工标注几百对黄金样本；
- precision 优先，宁可漏掉也不能误合并。

`product-matching-rag` 正好把通用商品匹配中的两个不同目标拆开：

1. **Retrieval：** 找到“可能是同一商品”的小候选集；
2. **Reasoning：** 对高度相似但细节不同的候选做严格排除。

这个分层非常重要。

对腕表尤其如此。同系列商品可能：

- 标题几乎一致；
- 图片外观极其接近；
- 品牌、系列、尺寸、机芯都相同；
- 但 reference 只差一位或一个后缀；
- 按当前业务定义必须判为不同商品。

因此不能让 embedding 相似度或视觉相似度直接触发 merge。

正确结构应当始终是：

```text
高召回层：可以模糊
最终决策层：必须确定、可解释、可拒识
```

---

## 2. 项目当前真实代码架构

README 给出的整体架构是：

```text
[WDC Corpus (1.3M+ Offers)]
       │
       ├─ filter_clusters.py
       ├─ build_catalog_split.py
       ├─ normalize_offers.py
       ├─ sample_catalog.py
       ├─ build_embeddings.py
       ├─ retrieve.py
       └─ match_with_llm.py
```

核心代码：

- `scripts/build_catalog_split.py`
- `scripts/normalize_offers.py`
- `scripts/build_embeddings.py`
- `scripts/retrieve.py`
- `scripts/match_with_llm.py`

源码：

- <https://github.com/Abhisheksasidharann/product-matching-rag/blob/main/scripts/build_catalog_split.py>
- <https://github.com/Abhisheksasidharann/product-matching-rag/blob/main/scripts/normalize_offers.py>
- <https://github.com/Abhisheksasidharann/product-matching-rag/blob/main/scripts/build_embeddings.py>
- <https://github.com/Abhisheksasidharann/product-matching-rag/blob/main/scripts/retrieve.py>
- <https://github.com/Abhisheksasidharann/product-matching-rag/blob/main/scripts/match_with_llm.py>

项目当前不是一个在线微服务，而是一个**离线脚本流水线原型**。它的价值主要在架构思路和验证闭环，而不是生产部署形态。

---

## 3. `build_catalog_split.py`：流式拆分和 out-of-core 思路

该脚本对 gzip JSONL 做两遍扫描：

```text
Pass 1:
  stream rows
  -> Counter(cluster_id)

Pass 2:
  cluster size == 1 -> singletons
  cluster 第一次出现 -> catalog_a
  cluster 后续出现 -> catalog_b
```

它没有把全部商品记录一次性加载进 DataFrame，而是逐行读取，适合百万级原始语料。

### 值得借鉴

当前三源数据也应该优先使用：

```text
JSONL / Parquet / Object Storage
+ streaming reader
+ batch transform
```

而不是先把 1000 万记录塞进单机 Pandas。

对于持续增量，更进一步应该改成事件化：

```text
source crawler
  -> raw_offer event
  -> normalize worker
  -> reference extractor
  -> resolver
  -> decision event
```

### 不能原样照搬

项目用 `Counter(cluster_id)` 和 `seen_clusters` 保存在单机内存里。WDC demo 可以，但生产持续增量时不应依赖“全局重新扫两遍”。

生产应该使用稳定实体表和幂等键：

```text
offer(source, source_offer_id) UNIQUE
reference_entity(brand_id, canonical_reference) UNIQUE
```

新数据到达后只做局部解析和局部链接。

---

## 4. `normalize_offers.py`：当前清洗层做了什么

代码主要做四件事：

1. 用 regex 排除车辆、房产等 off-domain 记录；
2. 清洗价格字符串；
3. 标题开头若重复品牌则移除品牌；
4. 构造：

```text
composite_text = brand | cleaned_title | truncated_description
```

其中 description 最多保留 200 字符。

如果完全没有 usable text，则直接 reject，不进入 embedding index。

### 对当前业务有价值的点

**先做确定性清洗，再做 embedding。**

腕表场景应该把这一层扩成：

```text
Raw Normalization
  - Unicode / 全半角统一
  - HTML / 空白清洗
  - 品牌别名规范化
  - source SKU / platform ID 字段识别
  - 标题中的编号候选扫描
  - 描述中的编号候选扫描
  - OCR 文本清洗
  - 配件/表带/盒证等商品角色识别
```

### 一个很重要的代码现实

README 描述里有“attribute extraction”的产品定位，但当前仓库代码中**没有独立的结构化 Model Number / MPN / Reference extractor 模块**。

现在实际依赖的是：

```text
composite_text
  -> embedding
  -> LLM 阅读自然语言
```

这对普通商品 demo 可以，但对当前 Spec 不够。

因为当前需求已经明确规定 reference 是最终身份语义，那么 reference 就不应该只是 prompt 里的一个潜在线索，而应成为一等数据结构。

---

## 5. `build_embeddings.py`：SentenceTransformer + FAISS 的实现

项目使用：

```python
SentenceTransformer("all-MiniLM-L6-v2")
```

批量编码：

```python
batch_size = 64
normalize_embeddings = True
```

然后：

```python
index = faiss.IndexFlatIP(dim)
index.add(embeddings)
```

因为向量已经 L2 normalize，Inner Product 等价于 cosine similarity。

同时它把每个向量对应的完整 metadata 放进一个 JSON `id_map`：

```text
vector position
  -> id
  -> cluster_id
  -> title
  -> composite_text
  -> brand
  -> price
```

### 优点

工程非常清晰：

```text
向量索引只解决 ANN / 相似检索
业务字段放 metadata map
```

这比把所有逻辑糅在模型里要好。

### 对 1000 万级数据的限制

`IndexFlatIP` 是精确暴力向量搜索。

概念上每个 query 都要和索引内全部向量算相似度：

```text
复杂度约 O(N * d)
```

50K demo 很合适，10M 持续增量则不应作为默认方案。

生产建议：

```text
第一层：brand/category/reference-pattern blocking
第二层：ANN index（HNSW / IVF 类）
第三层：规则 / 轻模型精排
```

尤其腕表场景有一个天然强 blocking key：`brand_id`。

没有理由让 Rolex 商品和 Cartier / Omega 全库做向量近邻。

可进一步按：

```text
brand_id
series_family（如果可解析）
reference_prefix / token signature
product_type
```

做 shard。

### `id_map` 的生产化问题

当前代码把 metadata 全部放到一个 JSON 数组再一次性载入内存。

生产更合适：

```text
vector_id -> metadata row
```

放在：

- 数据库；
- KV store；
- columnar store；
- 或按批次 mmap / Parquet lookup。

向量索引中只存最小 ID，不存一份巨大重复业务字段 JSON。

---

## 6. `retrieve.py`：召回层设计

`retrieve.py` 的逻辑：

```text
query.composite_text
  -> embedding
  -> FAISS search top 10
  -> position 映射到 metadata
  -> 看 ground-truth cluster_id 是否出现在 Top-K
```

README 报告：

```text
500 queries
Recall@10 = 0.896
```

这说明：

> **retrieval 只应该承担“候选不要漏太多”的责任，而不是最终同款判断。**

项目本身也承认 embedding 会把非常接近但实际不同的 variant 拉得很近。

这是和腕表场景高度同构的地方。

### 当前实现的一个评测注意点

代码先构造：

```python
indexed_clusters = set(entry["cluster_id"] for entry in id_map)
```

然后如果 query 的 ground-truth cluster 根本不在 sample index 中，会跳过该 query。

所以这个 Recall@K 本质是在测：

```text
“真匹配已知存在于索引中”条件下的 retrieval recall
```

它不是一个覆盖线上所有记录的 end-to-end match rate。

当前项目自己的 README 也说明 50K sampling 会制造真匹配不在 sample 中的 artifact。

对于我们的生产评测必须把这几种情况拆开：

```text
A. 真 reference 在候选库中，是否成功召回
B. 真 reference 根本不存在，系统是否正确 abstain
C. reference 存在但字段缺失，是否能恢复
D. 非商品 reference / compatibility reference 是否被错误当成自身型号
```

---

## 7. `match_with_llm.py`：LLM 终判器的具体实现

项目流程：

```text
FAISS retrieve 10
  -> 只取前 5 个
  -> Claude
  -> JSON:
       match_found
       matched_candidate
       confidence
       evidence
```

System Prompt 明确要求：

- 颜色不同不算同品；
- size 不同不算同品；
- pack quantity 不同不算同品；
- seller / price 不同仍可以是同一 SKU；
- 用 brand、model number、specs 提供 evidence。

这就是项目 precision-first 的核心。

README 基于 50 query 的端到端实验报告：

```text
Precision = 1.00
Recall    = 0.87
F1        = 0.93
```

这个方向是对的：宁可 false negative，也不要 false positive。

但不能把 50 条 query 的“0 FP”解释为已达到生产级绝对安全。

### 7.1 当前 LLM 层的几个风险

#### 风险 1：`confidence` 没有参与真正决策

虽然 prompt 返回：

```text
high / medium / low
```

但代码中：

```python
llm_says_match = llm_result.get("match_found", False)
```

只要 `match_found=true` 就进入正匹配逻辑。

也就是说：

```text
match=true + confidence=low
```

依然会被当成 match。

对“绝不能误匹配”的 Spec 不可接受。

#### 风险 2：只做 JSON parse，没有严格 schema validation

代码用：

```python
json.loads(text)
```

但没有进一步校验：

- `matched_candidate` 是否 integer；
- 是否属于实际展示给模型的 1..5；
- `confidence` 是否允许值；
- match_found 与 matched_candidate 是否逻辑一致；
- evidence 是否真的包含 reference 证据。

生产必须使用严格 schema validator。

#### 风险 3：API / JSON 错误被转换成 `match_found=False`

这对 precision 是保守的，但会混淆：

```text
业务 no-match
系统错误
模型不可解析
rate limit
```

生产必须是四态以上：

```text
ACCEPT
REJECT
ABSTAIN
ERROR
REVIEW
```

系统故障不能伪装成业务否定。

#### 风险 4：LLM 在最终身份定义上拥有过大权力

项目针对通用商品“是否 interchangeable”。

我们这里不是。

我们的业务定义已经给出确定规则：

```text
same product iff same reference number
```

所以 LLM 最终应该回答的不是：

```text
“它们是不是同一个商品？”
```

而是：

```text
“从这条记录中，你能否提取/确认属于商品自身的 reference？证据是什么？”
```

这样模型只做**证据抽取**，不做**实体定义**。

---

## 8. 最核心改造：从 Pairwise Match 变成 Reference Entity Resolution

通用 product matching 通常是：

```text
record A + record B
  -> matcher
  -> same / different
```

当前业务更适合：

```text
record
  -> reference resolver
  -> reference_entity_id
```

所有解析到同一个 `reference_entity_id` 的记录自然归组。

### 推荐实体主键

```text
reference_entity_id = hash(brand_id + canonical_reference)
```

数据库层：

```sql
UNIQUE (brand_id, canonical_reference)
```

这样：

- 不需要三源之间每两条都比较；
- 不需要依赖 pairwise transitivity；
- 新增第四个来源也不需要重新设计；
- 持续增量只需把新 record resolve 到已有 reference entity；
- 删除一条错误链接不会污染整张 connected component 图。

---

## 9. 生产架构：可以直接落地的版本

```text
                       ┌────────────────────┐
雷小安 ---------------->                    |
腕表之家 -------------->  Raw Offer Store   |
奢当家 ---------------->                    |
                       └─────────┬──────────┘
                                 |
                                 v
                    ┌────────────────────────┐
                    | Normalize / Brand Map  |
                    └────────────┬───────────┘
                                 |
                                 v
                    ┌────────────────────────┐
                    | Reference Candidate    |
                    | Extraction             |
                    | - structured field     |
                    | - title                |
                    | - description          |
                    | - image OCR            |
                    └────────────┬───────────┘
                                 |
                                 v
                    ┌────────────────────────┐
                    | Reference Role         |
                    | Classification         |
                    | own_ref / compat_ref   |
                    | seller_sku / unknown   |
                    └────────────┬───────────┘
                                 |
                                 v
                    ┌────────────────────────┐
                    | Canonicalizer          |
                    | brand-aware rules      |
                    └────────────┬───────────┘
                                 |
                ┌────────────────┴────────────────┐
                |                                 |
       high-confidence unique              missing/conflict
                |                                 |
                v                                 v
     ┌─────────────────────┐          ┌─────────────────────┐
     | Exact Reference     |          | Candidate Retrieval |
     | Entity Lookup       |          | lexical + ANN + OCR |
     └──────────┬──────────┘          └──────────┬──────────┘
                |                                |
                |                                v
                |                     ┌─────────────────────┐
                |                     | Evidence Verifier   |
                |                     | rules / LLM / image |
                |                     └──────────┬──────────┘
                |                                |
                └──────────────┬─────────────────┘
                               v
                    ┌────────────────────────┐
                    | Decision Engine        |
                    | ACCEPT/REVIEW/REJECT   |
                    | ABSTAIN/ERROR          |
                    └────────────┬───────────┘
                                 |
                 ┌───────────────┴───────────────┐
                 v                               v
       Reference Membership                Human Review
          + Audit Log                        Queue
```

---

## 10. 数据模型建议

### 10.1 `raw_offer`

```sql
raw_offer(
    offer_id            bigint primary key,
    source              varchar not null,
    source_offer_id     varchar not null,
    raw_json            jsonb not null,
    title_raw           text,
    description_raw     text,
    brand_raw           text,
    reference_raw       text,
    image_urls          jsonb,
    crawled_at          timestamp,
    payload_hash        varchar,
    unique(source, source_offer_id)
)
```

原始数据必须不可破坏地保存，后续 parser / rule 升级可以重放。

### 10.2 `normalized_offer`

```sql
normalized_offer(
    offer_id             bigint primary key,
    brand_id             bigint,
    brand_confidence     numeric,
    title_normalized     text,
    description_clean    text,
    product_type         varchar,
    is_accessory         boolean,
    normalize_version    varchar,
    updated_at           timestamp
)
```

### 10.3 `reference_candidate`

一条商品可以有多个候选 reference，不要过早只保留一个字符串。

```sql
reference_candidate(
    candidate_id         bigint primary key,
    offer_id             bigint not null,
    raw_value            varchar not null,
    canonical_value      varchar,
    source_type          varchar not null,
    source_location      varchar,
    role                 varchar,
    extractor            varchar,
    confidence           numeric,
    evidence             jsonb,
    parser_version       varchar
)
```

`source_type` 示例：

```text
structured_field
title_regex
description_regex
ocr_backcase
ocr_card
llm_extract
```

`role` 示例：

```text
OWN_REFERENCE
COMPATIBLE_REFERENCE
SELLER_SKU
PLATFORM_ID
SERIAL_NUMBER
UNKNOWN
```

这个 role 非常关键。

因为配件标题可能写：

```text
适用 XXX 型号
兼容 XXX reference
```

如果只看到一个“像 reference 的字符串”就 exact join，会制造灾难性 false positive。

### 10.4 `reference_entity`

```sql
reference_entity(
    reference_entity_id   bigint primary key,
    brand_id              bigint not null,
    canonical_reference   varchar not null,
    display_reference     varchar,
    reference_family      varchar,
    status                varchar,
    created_at            timestamp,
    unique(brand_id, canonical_reference)
)
```

### 10.5 `offer_reference_membership`

```sql
offer_reference_membership(
    offer_id              bigint primary key,
    reference_entity_id   bigint,
    decision              varchar not null,
    decision_reason       varchar not null,
    confidence_tier       varchar,
    evidence_snapshot     jsonb,
    rule_version          varchar,
    model_version         varchar,
    decided_at            timestamp
)
```

建议 `decision` 不做 boolean：

```text
ACCEPT
REVIEW
REJECT
ABSTAIN
ERROR
```

---

## 11. Reference 抽取不要一步到位，应该是“候选生成 → 角色判别 → canonicalize”

### 11.1 候选生成

顺序建议：

```text
1. 原站独立 reference/model 字段
2. 已知品牌模板 regex
3. title 中编号候选
4. description 中编号候选
5. 图片 OCR
6. LLM constrained extraction
```

每个结果都带 provenance。

例如：

```json
{
  "raw_value": "210.30.42.20.01.001",
  "source_type": "title_regex",
  "source_location": "title[18:35]",
  "role": "OWN_REFERENCE",
  "confidence": 0.98
}
```

### 11.2 Canonicalization 必须 brand-aware

错误做法：

```text
所有型号都统一删除所有符号
```

因为某些品牌的：

- 点号；
- 连字符；
- 空格；
- 字母后缀；
- 大小写；

可能具有语义。

应该保存两份：

```text
raw_reference
canonical_reference
```

canonicalizer 还必须版本化：

```text
canonicalizer_version
```

### 11.3 多证据一致优先

最高置信自动解析可以要求：

```text
structured reference
  == title extracted reference
```

或：

```text
title reference
  == OCR backcase reference
```

这种“两个独立证据一致”比单一大模型 confidence 有意义得多。

---

## 12. 自动匹配决策矩阵

推荐先从极保守规则上线。

| 情况 | 决策 |
|---|---|
| 两侧 brand 冲突 | REJECT |
| 两侧都有高置信 OWN_REFERENCE，canonical 完全一致 | ACCEPT |
| 两侧都有高置信 OWN_REFERENCE，但 canonical 不同 | REJECT |
| 一侧独立字段 reference，另一侧标题抽取 reference，brand 一致且 canonical 一致 | ACCEPT |
| title 抽取与 OCR 独立证据一致 | 可 ACCEPT，建议先离线验证后开启 |
| 只有 embedding 高相似 | ABSTAIN / REVIEW |
| 只有图片高相似 | ABSTAIN / REVIEW |
| LLM 说 same，但 reference 冲突 | REJECT |
| LLM 提取到候选 reference，但没有足够独立证据 | REVIEW |
| 发现 compatibility 语义 | 不允许作为 OWN_REFERENCE 自动合并 |
| 解析出多个互相冲突 reference | REVIEW / ABSTAIN |

最重要的一条：

```text
reference mismatch has veto power
```

即使：

```text
title similarity = 0.99
image similarity = 0.995
LLM says match
```

只要双方明确的 canonical reference 不同，就直接 REJECT。

---

## 13. 如何改造原项目的 Retrieval 层

原项目：

```text
all corpus
  -> MiniLM
  -> IndexFlatIP
  -> top 10
```

推荐：

```text
Query Offer
   |
   +-- brand known? -------- yes --> brand shard
   |                              |
   |                              +-- ref tokens known?
   |                              |      -> lexical / prefix candidates
   |                              |
   |                              `-- semantic ANN top K
   |
   `-- brand unknown
          -> brand resolver first
          -> unresolved records do not global-merge
```

### 多路候选 union

```text
C = C_reference_prefix
  ∪ C_title_lexical
  ∪ C_text_embedding
  ∪ C_ocr_reference
  ∪ C_image_embedding
```

然后做 deterministic pruning：

```text
brand conflict -> drop
product/accessory role conflict -> drop
strong reference conflict -> drop
```

最后才把小候选集送给 verifier。

### 为什么图片应该是辅助通道

腕表同系列变体视觉非常接近。

图片最适合：

- OCR 表背 reference；
- OCR 保卡 / 吊牌；
- 找相似候选；
- 为人工审核展示证据；
- 在文本冲突时提供反证。

不适合：

```text
image similarity > threshold -> 自动同 reference
```

---

## 14. 如何改造原项目的 LLM 层

原 prompt 问：

```text
Which candidate is the SAME product?
```

建议改成两个窄任务。

### Task A：Reference Extraction

输入：

```text
brand
raw title
description
structured fields
OCR snippets
```

输出强 schema：

```json
{
  "candidates": [
    {
      "reference": "...",
      "role": "OWN_REFERENCE|COMPATIBLE_REFERENCE|SELLER_SKU|UNKNOWN",
      "evidence": ["title", "ocr_backcase"],
      "exact_span": "..."
    }
  ],
  "abstain": true
}
```

模型不能直接输出：

```text
match=true
```

### Task B：Conflict Explanation

当规则层看到：

```text
candidate A = ref X
candidate B = ref Y
```

让 LLM 判断：

```text
X/Y 是否只是格式差异？
是否是配件兼容型号？
是否有明确文本说明其中一个不是当前商品型号？
```

LLM 输出“证据”和“建议”，最终 decision engine 再决定。

### 结构化输出必须严格校验

伪代码：

```python
result = llm_extract(payload)
validated = ReferenceExtractionSchema.validate(result)

if not validated:
    return ERROR

if result.abstain:
    return ABSTAIN

for candidate in result.candidates:
    if candidate.role != "OWN_REFERENCE":
        continue
    verify_candidate(candidate)
```

---

## 15. 关键 Decision Engine 伪代码

```python
def resolve_offer(offer):
    normalized = normalize(offer)

    brand = resolve_brand(normalized)
    if not brand.is_high_confidence:
        return Decision("ABSTAIN", "brand_unresolved")

    refs = extract_reference_candidates(normalized, brand)
    own_refs = [r for r in refs if r.role == "OWN_REFERENCE"]

    # 1. 多个强 reference 冲突，拒绝自动解析
    strong_refs = unique(
        r.canonical for r in own_refs if r.confidence_tier == "STRONG"
    )
    if len(strong_refs) > 1:
        return Decision("REVIEW", "conflicting_strong_references")

    # 2. 唯一强 reference，直接 resolve reference entity
    if len(strong_refs) == 1:
        canonical_ref = strong_refs[0]
        entity = lookup_reference_entity(brand.id, canonical_ref)
        if entity:
            return Decision(
                "ACCEPT",
                "strong_exact_reference",
                entity_id=entity.id,
                evidence=own_refs,
            )

        # 可按业务策略新建 reference entity，但必须记录来源
        entity = create_candidate_reference_entity(
            brand.id, canonical_ref, evidence=own_refs
        )
        return Decision(
            "ACCEPT",
            "new_strong_reference",
            entity_id=entity.id,
            evidence=own_refs,
        )

    # 3. 无强 reference，只做候选发现
    candidates = retrieve_candidates(normalized, brand)

    # 4. 强冲突候选先 veto
    candidates = [
        c for c in candidates
        if not has_strong_reference_conflict(normalized, c)
    ]

    if not candidates:
        return Decision("ABSTAIN", "no_reference_evidence")

    # 5. verifier 只能补证据，不能直接 merge
    evidence = verify_candidates(normalized, candidates)
    recovered_ref = derive_reference_from_verified_evidence(evidence)

    if recovered_ref.is_strong_and_unique:
        entity = lookup_reference_entity(brand.id, recovered_ref.value)
        if entity:
            return Decision(
                "ACCEPT",
                "recovered_exact_reference",
                entity_id=entity.id,
                evidence=evidence,
            )

    return Decision("REVIEW", "insufficient_reference_evidence")
```

这段逻辑保证：

> 模糊模型最终必须把证据“收敛”为一个 canonical reference，才能自动进入同一实体。

---

## 16. 增量架构

当前项目是离线脚本：重新读文件、重新建 sample、重新建 index。

生产应该做成：

```text
new_offer
  -> idempotent normalize
  -> reference extraction
  -> exact entity lookup
  -> optional candidate lookup
  -> decision
```

### 事件建议

```text
offer.raw.created
offer.normalized
reference.extracted
reference.resolved
match.review.requested
match.decision.created
```

### 幂等

每个 stage 至少带：

```text
source
source_offer_id
payload_hash
parser_version
model_version
rule_version
```

同一数据重复抓取不会重复建实体。

### 重放

canonicalizer 或 extractor 升级后：

```text
SELECT offers
WHERE parser_version < new_version
```

重新解析即可，不需要重新抓取原站。

---

## 17. 100 万～1000 万级性能设计

### 17.1 Fast Path 应处理绝大部分可确定样本

最便宜的路径：

```text
brand_id + canonical_reference
  -> indexed key lookup
```

这是数据库索引问题，不是 AI 问题。

如果能把大量记录 reference 可靠抽出来，则绝大多数自动匹配无需 vector search 和 LLM。

### 17.2 Candidate Retrieval 只服务长尾

只有这些数据进入重模型：

```text
reference 缺失
reference 冲突
标题埋型号
OCR 才有型号
新增品牌规则尚未覆盖
```

这会把计算成本从：

```text
每一条都向量 + LLM
```

降低到：

```text
少数疑难记录才向量 / OCR / LLM
```

### 17.3 ANN 不要默认全局 `IndexFlatIP`

推荐：

```text
brand shard
  -> ANN top 50
  -> lexical / field constraints
  -> top 5~20 verifier candidates
```

具体 ANN backend 可以替换，系统接口只依赖：

```text
search(vector, filters, k)
```

不要让业务逻辑绑死某个向量库。

### 17.4 Batch embedding

原项目 build 阶段已经使用 batch=64，这是对的。

生产异步 embedding worker 也应该聚合批次，而不是每条商品单独 RPC / 单独推理。

---

## 18. 黄金标签应该怎样花

只有几百对人工标签时，不应该随机抽样。

应该优先标 hard negatives：

```text
同品牌
同系列
标题高度相似
图片高度相似
reference 只差一位/一个后缀
配件标题包含目标手表 reference
同一 reference 不同格式
reference 埋在描述而非字段
OCR 与标题冲突
```

建议黄金集至少含四种集合：

```text
G1: exact-reference positives
G2: near-reference hard negatives
G3: missing-reference recoverable positives
G4: accessories / compatibility traps
```

训练用途不是唯一目标，更重要的是：

- 校准自动 ACCEPT 规则；
- 评估 parser；
- 评估 candidate recall；
- 评估 false merge；
- 回归测试每次规则升级。

---

## 19. 评测指标：不要只看 F1

原项目给 Precision / Recall / F1，是合理 baseline。

当前业务更应该拆成：

### 19.1 Reference Extraction

```text
Reference exact accuracy
Reference candidate recall
OWN_REFERENCE role precision
```

### 19.2 Candidate Retrieval

```text
Recall@10
Recall@50
```

前提是：真 reference 确实存在于目标 source。

### 19.3 自动合并

最核心：

```text
Auto-Merge Precision
False Merge Per Million Decisions
Auto-Merge Coverage
Abstain Rate
Review Rate
```

### 19.4 高风险分桶

指标必须按：

```text
brand
source pair
product type
reference present/missing
accessory/non-accessory
new vs old reference
time slice
```

分桶。

否则整体 99.99% 可能掩盖某一个品牌规则发生灾难性错误。

---

## 20. 为什么原项目的 100% Precision 不能直接当上线依据

README 的 100% Precision 是一个很好的原型信号，但实验只有 50 queries。

生产目标是百万到千万级，并且错误代价非常高。

需要特别防止：

```text
50 条测试 0 FP
  !=
线上千万条几乎不会 FP
```

而且项目测试是在：

- 50K sample；
- retrieval top-K；
- query 必须可在 sampled index 中找到相关 cluster 的条件；
- 单次 LLM；

下完成的。

所以可把它解释为：

> **证明“召回 + 严格 verifier”这个方向可行。**

不能解释为：

> **证明当前代码已经满足金融级/零误合并生产 SLA。**

---

## 21. 人工审核页面应该展示什么

REVIEW 不应该只给“模型说 0.93”。

审核页建议左右对比：

```text
Source / URL
Brand
Title
Structured reference
Extracted references
Reference role
OCR reference
Candidate entity canonical reference
Reference exact/mismatch status
Title diff
Image
LLM evidence（若有）
Rule trace
```

审核操作：

```text
Confirm reference
Reject candidate
Mark compatibility reference
Mark seller SKU
Correct brand
Create new canonical reference
```

审核结果直接回流 `reference_candidate` 和规则评测集。

---

## 22. 可直接实施的服务边界

### `BrandResolver`

```text
input: raw brand/title/source
output: brand_id + confidence + evidence
```

### `ReferenceExtractor`

```text
input: normalized offer + OCR
output: reference_candidate[]
```

### `ReferenceCanonicalizer`

```text
input: brand_id + raw_reference
output: canonical_reference + rule_version
```

### `ReferenceResolver`

```text
input: brand_id + canonical_reference
output: reference_entity_id / not_found
```

### `CandidateService`

```text
input: normalized offer + brand filter
output: candidate_reference_entity[]
```

### `EvidenceVerifier`

```text
input: offer + candidates
output: structured evidence only
```

### `DecisionEngine`

```text
input: deterministic evidence + verifier evidence
output: ACCEPT / REJECT / REVIEW / ABSTAIN / ERROR
```

### `ReviewService`

人工闭环。

### `AuditStore`

保存每个决定为什么发生。

---

## 23. 审计日志必须是一等公民

每次自动 ACCEPT 至少记录：

```json
{
  "offer_id": 123,
  "reference_entity_id": 456,
  "decision": "ACCEPT",
  "reason": "strong_exact_reference",
  "brand_id": 8,
  "canonical_reference": "...",
  "evidence": [
    {
      "type": "structured_field",
      "value": "..."
    },
    {
      "type": "title_extract",
      "value": "..."
    }
  ],
  "parser_version": "ref-parser-v3",
  "rule_version": "decision-v5",
  "model_version": null
}
```

以后发现规则错误，可以精准查询：

```text
所有 decision-v5 + 某 brand + 某 parser pattern 的 ACCEPT
```

并批量回滚/重算。

---

## 24. 最小可落地版本

### Phase 1：Reference Fast Path

先不做复杂 AI。

完成：

```text
raw schema
brand normalization
structured reference ingestion
title regex candidate extraction
brand-aware canonicalization
reference_entity table
exact reference join
audit log
```

这一阶段就可以解决大量明确样本。

### Phase 2：高风险 hard-negative 防线

增加：

```text
reference role classifier
accessory / compatibility detection
conflict rules
review queue
```

目标：压低 false positive，而不是追 recall。

### Phase 3：Candidate Retrieval

借鉴本项目：

```text
composite text
batch embedding
brand-sharded ANN
Top-K candidates
```

只给 reference 缺失样本使用。

### Phase 4：OCR / LLM Evidence

增加：

```text
image OCR
constrained reference extraction
conflict explanation
```

仍不允许模型仅凭“语义很像”自动 merge。

### Phase 5：Active Learning

用审核结果持续完善：

```text
brand rules
reference regex
role classifier
hard negative set
model calibration
```

---

## 25. 对原项目哪些保留，哪些替换

| 原项目组件 | 建议 | 原因 |
|---|---|---|
| streaming JSONL | 保留思想 | 适合大规模离线/回放 |
| data normalization | 保留并增强 | 是 embedding / extraction 基础 |
| composite_text | 保留 | 适合 fallback retrieval |
| SentenceTransformer | 可保留为 baseline | 只承担候选召回 |
| `IndexFlatIP` | 小规模实验保留，生产替换 | 10M 全扫描不经济 |
| metadata JSON id_map | 替换 | 生产用 DB/KV/columnar lookup |
| Top-K retrieval | 保留 | 清晰分离 recall 与 decision |
| LLM same-product 决策 | 替换 | 当前业务定义由 reference 决定 |
| LLM evidence | 保留并加强 | 很适合处理缺失/冲突证据 |
| 50 query eval | 只作 smoke test | 无法证明极低 FP |
| TP/FP/FN/TN | 保留 | 但增加 coverage/abstain/FPM |

---

## 26. 最终推荐方案

当前需求不应该建设成：

```text
三源全量 pairwise similarity
  -> embedding / image / LLM
  -> threshold
  -> merge
```

应该建设成：

```text
三源 Offer
  -> Reference Extraction
  -> Brand-aware Canonicalization
  -> Reference Entity Resolution
  -> Exact Reference Membership
```

只有 reference 缺失时才进入：

```text
Candidate Retrieval
  -> OCR / LLM / image evidence
  -> recover a canonical reference
  -> exact resolve
```

关键原则按优先级排列：

1. **业务实体不是“相似商品”，而是 `(brand, canonical_reference)`。**
2. **Reference 一致是自动合并正证据，Reference 冲突具有 veto 权。**
3. **Embedding / 图片只负责候选和证据，不拥有最终 merge 权。**
4. **LLM 从 matcher 改为 reference/evidence extractor。**
5. **允许大量 ABSTAIN / REVIEW，换极高 precision。**
6. **所有自动 ACCEPT 都必须有 provenance、rule version、可回放审计。**
7. **增量处理围绕 stable reference entity，不做全量重新 pairwise matching。**
8. **用几百条黄金标签重点覆盖 hard negatives，而不是平均随机抽样。**

---

## 27. 与 `product-matching-rag` 的最终关系

这个项目可以直接作为以下模块的 prototype：

```text
normalize_offers.py
  -> fallback normalization baseline

build_embeddings.py
  -> batch embedding baseline

retrieve.py
  -> candidate retrieval baseline

match_with_llm.py
  -> 改造成 EvidenceVerifier / ReferenceExtractor
```

也就是说，不需要推倒重来。

最小改造路径是：

```text
原：
retrieve -> LLM same/not-same -> match

改：
extract/canonicalize reference
   |-- exact -> ACCEPT
   `-- missing -> retrieve -> LLM extract evidence
                         -> canonical reference
                         -> exact resolve / REVIEW
```

这样既复用了原项目最有价值的 RAG 候选架构，又严格满足当前 Spec 的 reference-first 与 precision-first 约束。

**推荐优先落地这一版。**
