# GraLMatch：把多源 Pairwise Match 的“传递污染”改造成二奢 Reference-first 的图安全审计层

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取尚未在 `奢侈品调研/a` 分析过的论文：

**GraLMatch: Matching Groups of Entities with Graphs and Language Models**

- 论文：<https://arxiv.org/abs/2406.15015>
- 论文代码：<https://github.com/FernandoDeMeer/GraLMatch>
- EDBT 2025 论文版本对应的公开实现位于上述 GitHub 仓库
- 当前需求 Spec：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

分析前已检查 `奢侈品调研/a` 当前已有结果，包含 Ameli、TransClean、DeepBlocker、pyJedAI、AutoBlock、Ditto、AnyMatch、MOON2.0、Conformal Selective Prediction 等，但**没有 GraLMatch**，因此本次继续执行。

当前 Spec 的核心不是一般意义上的“两个商品是不是很像”，而是：

> **只有同一 reference number / 型号，才定义为同一个商品。**

并且还有四个决定架构方向的约束：

1. 雷小安、腕表之家、奢当家三个来源，100 万～1000 万级记录，持续增量；
2. reference 字段高度稀疏，可能埋在标题/描述，甚至要从图片 OCR；
3. **precision 极端优先，宁可漏掉也不能误合并**；
4. 可以人工标几百对黄金样本，但不可能靠海量人工标注兜底。

GraLMatch 对这个需求最有价值的不是 DistilBERT 本身，而是它揭示了一个生产系统里非常危险、但很容易被忽略的问题：

```text
pairwise precision 很高
        !=
最终实体分组一定安全
```

只要一条 false-positive edge 把两个正确组件连接起来，连通性就会把错误通过传递关系扩散到整个组件。

论文采用：

```text
Blocking
  -> Pairwise Matcher
  -> Prediction Graph
  -> Minimum Edge Cut / Edge Betweenness Cleanup
  -> Entity Groups
```

来切掉最可能造成“跨簇污染”的边。

但对当前腕表需求，**不能直接照搬 GraLMatch 的完整算法**。最关键的原因是：

> GraLMatch 的核心停止条件之一是“组件大小不要超过数据源数”，适合“一个真实实体在每个 source 最多一条记录”的场景；而二奢业务按 reference 定义实体后，同一个 `Rolex 126610LN` 可以在雷小安、腕表之家、奢当家各出现几百甚至几千条 listing。组件大本身完全正常。

因此推荐的最终方案是：

```text
                  ┌──────────────────────────┐
                  │ Reference Evidence Layer │
                  │ 字段/标题/描述/OCR/目录库 │
                  └─────────────┬────────────┘
                                │
                                v
                  ┌──────────────────────────┐
                  │ Canonical Reference      │
                  │ brand + canonical_ref    │
                  └─────────────┬────────────┘
                                │
                    exact equality only
                                │
                                v
             ┌────────────────────────────────────┐
             │ Reference Entity / Safe Assignment │
             └────────────────┬───────────────────┘
                              │
               ┌──────────────┴──────────────┐
               v                             v
        自动发布同款关系                Suspicious Graph
                                      仅做安全审计
                                             │
                         ┌───────────────────┴──────────────────┐
                         │ conflict anchor / bridge / min-cut   │
                         │ betweenness / hard-negative review   │
                         └───────────────────┬──────────────────┘
                                             v
                                         拒识/人工复核
```

一句话概括：

> **让 reference identity 决定“能不能合并”，让 GraLMatch 图算法决定“哪里可能已经被错误证据污染、哪里最值得优先复核”。**

这比“用大模型直接判断两个商品是不是同款”更符合当前 Spec 的绝对 precision-first 目标。

---

# 1. GraLMatch 解决的到底是什么问题

传统 Entity Matching 常被建模成二分类：

```text
f(record_i, record_j) -> Match / NoMatch
```

论文指出，生产里的最终结果通常不是孤立 pair，而是实体组。

假设模型得到：

```text
A --match-- B --match-- C
```

即使模型从来没有显式计算 `A-C`，如果系统用 connected component 表示一个实体，那么最终已经隐式声明：

```text
A == B == C
=> A == C
```

此时一条错边的破坏力不是“错一个 pair”，而可能是把两个本来正确的实体簇完全粘在一起。

例如腕表场景：

```text
簇 1：Rolex 126610LN

A1 ---- A2 ---- A3
              \
               X   <- 一条错误边
              /
B1 ---- B2 ---- B3

簇 2：Rolex 126610LV
```

若系统按连通分量直接输出，错误的 `X` 会让两个 reference 变成一个超大组件。

这就是 GraLMatch 所强调的 **transitive pollution / transitive false matches**。

当前 Spec 对这类问题尤其敏感，因为 `126610LN` 与 `126610LV`：

- 品牌一样；
- 系列一样；
- 文本高度相似；
- 视觉也高度相似；
- reference 只差很少字符；

却在业务定义上必须严格判为不同实体。

所以，**同系列邻近 reference hard negative** 恰恰是当前系统最应该重点防守的边界。

---

# 2. 论文原始架构

论文的完整方法可以拆成四层。

## 2.1 Blocking：先缩小候选空间

如果有 `N` 条记录，全量 pair 数是 `O(N^2)`，百万级以后不可接受。

GraLMatch 先做 Blocking，只把“有可能相同”的记录组成候选 pair。

论文/代码中主要有三类 blocking：

### 2.1.1 ID Overlap

金融数据里用 ISIN、CUSIP、VALOR、SEDOL 等 identifier overlap 生成候选。

对当前需求可以直接映射成：

```text
reference field overlap
canonical reference overlap
OCR reference overlap
标题 reference candidate overlap
```

但注意：

> 在当前业务里 reference 不是“召回特征之一”，而是最终实体 identity。

因此如果 reference 已经被高可信抽取并规范化，就没有必要再交给 pairwise 模型决定 Match；应该直接进入 exact identity path。

### 2.1.2 Token Overlap

论文代码把记录 token 化后构建稀疏矩阵：

```text
records x vocabulary
```

然后用 sparse dot product 计算 token overlap，并且把同 source 的记录 mask 掉，再取 top-N。

代码位于：

- `datainc_code/src/matching/matcher.py`
- `get_tknzd_records_and_overlap_indicators`
- `get_top_overlap_idx`

这个实现用 `scipy.sparse.csr_matrix`，避免直接建立稠密 record-token 矩阵。

当前业务里可以保留这个思路，但用途建议改为：

```text
reference 不明确的记录
        ↓
通过品牌 / 系列 / title token / embedding
召回可能的 canonical reference
        ↓
只做“候选 reference 检索”
而不是直接建立自动 Match edge
```

### 2.1.3 依赖上游实体结果做下一层 Blocking

GraLMatch 在 securities 场景中会先利用 company matching，再根据 matched issuer 缩小 securities 的候选。

这是一个很值得迁移的工程思想：

> **先解决更稳定的上位实体，再用上位实体约束下位实体。**

腕表可以做成：

```text
Brand canonicalization
        ↓
Collection / family
        ↓
Reference candidate retrieval
        ↓
Reference verification
```

品牌不一致时，后面直接禁止跨品牌 reference 合并。

---

# 3. Pairwise Matcher：论文怎么做

GraLMatch 的论文实验使用 Transformer pairwise matcher，公开实现里包含 DistilBERT variant 和 DITTO。

DistilBERT 路径本质是：

```text
record_left + record_right
        ↓
tokenizer
        ↓
DistilBERT sequence classifier
        ↓
softmax(match / non-match)
        ↓
match probability
```

对应代码：

- `datainc_code/src/models/pytorch_model.py`
- `datainc_code/src/models/run_transformer_training.py`

训练部分就是标准 PyTorch fine-tuning：

```text
forward
 -> classification loss
 -> backward
 -> gradient clipping
 -> optimizer.step
 -> scheduler.step
```

测试阶段取类别 1 的 softmax 概率作为 `prediction_proba`。

论文非常值得关注的结论是：

> 在 entity group matching 中，pairwise recall 很高但 precision 稍差，可能比 precision 更高但 recall 稍低的模型产生更差的最终分组。

原因很简单：

```text
一个 false negative
通常只是少一条边

一个 false positive
可能把两个组件合并
并制造 O(|C1| * |C2|) 个隐式错误关系
```

这与当前 Spec “precision 优先到极致”完全一致。

但我的建议是：

> **不要把这个 Pairwise Matcher 放在最终自动合并开关上。**

当前业务有一个比“语义是否相似”更强的业务定义：`reference equality`。

因此 pairwise model 最多只承担：

1. 候选 reference 排序；
2. hard-negative 识别；
3. OCR/标题抽取冲突时的复核排序；
4. 人工队列优先级；
5. 作为“否决/怀疑信号”，不能单独作为“自动合并信号”。

---

# 4. GraLMatch Graph Cleanup 的技术实现

这是论文最值得复用的部分。

## 4.1 先把高分 pair 变成图

公开代码的 `generate_matches_graph` 非常直接：

```python
positive_matches_df = pairwise_matches_preds[
    pairwise_matches_preds['prob'] > threshold
]

matches_graph = nx.from_pandas_edgelist(
    positive_matches_df,
    'lid',
    'rid',
    ['prob', 'match_type']
)
```

默认 threshold 是：

```text
0.999
```

也就是说它一开始就非常保守。

对当前需求，这个思路正确，但建议进一步改为：

```text
只有“高可信 reference assignment”可以形成 publish graph edge

pairwise ML probability > 0.999
只能进入 suspicious/candidate graph
不能进入 publish graph
```

---

## 4.2 Minimum Edge Cut

如果一个 connected component 非常大，GraLMatch 先找 minimum edge cut：

```text
删除最少的边
让当前 component 被切开
```

直觉上，如果两个内部很稠密的正确实体组，只被少量 false-positive bridge 连起来，那么 minimum cut 很容易把 bridge 找出来。

公开代码使用：

```python
nx.minimum_edge_cut(subgraph)
```

然后直接删除这些边。

论文原算法用 `gamma` 控制什么时候采用 Minimum Edge Cut；仓库实现则把第一阶段写成：

```text
while component_size > 5 * num_of_datasources:
    对最大 component 做 minimum edge cut
```

这部分**不能直接用于腕表 reference entity**，原因后面会详细说明。

---

## 4.3 Edge Betweenness Centrality

第二阶段对仍然过大的 component 计算 edge betweenness：

```text
某条边被多少条最短路径经过
```

连接两个“社区”的 bridge edge 往往有高 betweenness。

公开代码：

```python
betweenness_centrality = nx.edge_betweenness_centrality(subgraph)
max_edge = max(betweenness_centrality, key=betweenness_centrality.get)
matches_graph.remove_edge(*max_edge)
```

原实现反复执行，直到组件大小不超过 `num_of_datasources`。

这个算法非常适合：

```text
找“最可疑的桥”
```

但它不是业务真值。

因此当前系统应把它降级为：

```text
suspicion score / review priority
```

而不是：

```text
betweenness 最大 => 一定是假边 => 自动删除
```

---

## 4.4 最后补全传递边

GraLMatch 清理完组件后，会把同一 component 内缺失的 transitive edge 补齐。

公开代码 `generate_transitive_matches_graph` 会遍历 component 里的 node pairs，并补成 complete graph。

这符合论文“实体组里所有节点代表同一实体”的定义。

但在百万级商品系统里，生产上不应该真的把一个 reference 下所有 listing 展开成两两 edge。

假设：

```text
Rolex 126610LN 有 50,000 条历史 listing
```

完整 clique 需要：

```text
50000 * 49999 / 2
≈ 12.5 亿条 pair
```

完全没必要。

正确的生产数据结构应是：

```text
record_id -> reference_entity_id
```

查询“同款商品”时：

```sql
SELECT *
FROM product_reference_assignment
WHERE reference_entity_id = ?
```

而不是物化所有 pair。

这也是本次落地方案和论文实现之间最重要的工程差异之一。

---

# 5. GraLMatch 为什么对当前需求有价值

## 5.1 它证明了“pairwise 指标漂亮”不代表分组安全

论文实验明确展示：

- pairwise 阶段即使 precision 看起来很好；
- 一旦把连通分量里的隐式 transitive matches 算进去；
- 少量 false positive 可以让 group-level precision 大幅下降。

对腕表系统，这意味着上线验收不能只看：

```text
pairwise precision / recall / F1
```

还必须看：

```text
component contamination
reference conflict component count
anchor purity
跨-reference 错误连通率
```

---

## 5.2 它把“错边的破坏力”显式建模了

普通 matcher 只知道：

```text
score(A, B)
```

Graph Cleanup 会额外问：

```text
如果接受 A-B，整个 component 会发生什么？
```

这非常适合做 production guardrail。

例如：

```text
A: strong ref = 126610LN
B: unknown
C: strong ref = 126610LV

A -- B -- C
```

即使：

```text
score(A,B)=0.9996
score(B,C)=0.9997
```

整个组件出现两个 mutually-exclusive strong reference anchor：

```text
126610LN != 126610LV
```

对当前业务这已经构成**确定性冲突**，应该直接熔断，而不是再讨论相似度。

---

# 6. 为什么不能直接复制 GraLMatch

## 6.1 最大问题：`mu = number_of_sources` 在当前业务不成立

论文明确说明，GraLMatch 的 group-size 假设适合：

```text
每个真实实体
在每个 data source 最多一个 record
```

因此论文会把期望最大实体组大小 `mu` 设为数据源数。

当前只有三个来源，如果照搬：

```text
mu = 3
```

那么任何包含 4 条以上 listing 的同 reference 组件都会被算法强行切开。

这是错误的。

当前业务的实体是：

```text
reference entity
```

不是：

```text
某个物理单品在三个平台各出现一次
```

同一个 reference 在同平台出现大量 listing 是正常情况。

所以**组件大小不能作为错误判据**。

---

## 6.2 论文 Graph Cleanup 会“无条件切边”，当前系统不能这样做

Minimum cut / betweenness 是结构启发式，不理解：

```text
reference evidence
brand compatibility
OCR provenance
structured field trust
seller SKU vs manufacturer reference
```

因此生产系统必须让业务硬约束优先：

```text
hard semantic conflict
    >
graph structural suspicion
    >
ML similarity score
```

---

## 6.3 NetworkX + CSV 流程不适合作为 1000 万级主数据面

仓库实现大量使用：

- pandas DataFrame；
- CSV 中间文件；
- NetworkX connected components；
- Python 循环；
- 逐 component betweenness；

非常适合研究实验和局部图，但不适合作为千万级全量在线主链路。

论文也指出 edge-cut / betweenness 清理超大 component 会变慢，因此做了 pre-cleanup。

生产化应该：

1. **主 identity 用 hash/index，不用 graph**；
2. 图只保留 suspicious records；
3. graph cleanup 只跑局部小组件；
4. 大批量聚合用 Spark/Flink/SQL，不用 NetworkX 处理全库；
5. 只有小规模异常 component 才下沉到 NetworkX / igraph。

---

# 7. 建议的生产架构：Reference-first + Graph Safety Auditor

## 7.1 数据层模型

推荐至少有 5 张核心逻辑表。

### 7.1.1 `product_record`

```sql
CREATE TABLE product_record (
    record_id           BIGINT PRIMARY KEY,
    source              VARCHAR(32) NOT NULL,
    source_record_id    VARCHAR(128) NOT NULL,
    source_updated_at   TIMESTAMP,
    title               TEXT,
    description         TEXT,
    structured_brand    VARCHAR(128),
    structured_ref      VARCHAR(256),
    image_urls          JSON,
    raw_payload         JSON,
    ingest_version      BIGINT NOT NULL,
    UNIQUE(source, source_record_id)
);
```

只保存原始事实，不在这里覆盖抽取结果。

---

### 7.1.2 `reference_evidence`

每一次发现 reference 候选，都落一条可追溯证据。

```sql
CREATE TABLE reference_evidence (
    evidence_id         BIGINT PRIMARY KEY,
    record_id           BIGINT NOT NULL,
    extractor           VARCHAR(64) NOT NULL,
    extractor_version   VARCHAR(64) NOT NULL,
    evidence_source     VARCHAR(32) NOT NULL,
    raw_value           VARCHAR(256),
    normalized_value    VARCHAR(256),
    brand_id            BIGINT,
    role                VARCHAR(32),
    confidence          DECIMAL(8,6),
    text_span_start     INT,
    text_span_end       INT,
    image_id            VARCHAR(256),
    status              VARCHAR(32),
    created_at          TIMESTAMP NOT NULL
);
```

`evidence_source` 建议限定：

```text
STRUCTURED_FIELD
TITLE_REGEX
DESCRIPTION_REGEX
TITLE_MODEL
DESCRIPTION_MODEL
IMAGE_OCR_CASEBACK
IMAGE_OCR_CARD
IMAGE_OCR_TAG
CATALOG_LOOKUP
HUMAN
```

`role` 至少区分：

```text
PRODUCT_REFERENCE
PLATFORM_SKU
SELLER_SKU
ACCESSORY_COMPATIBLE_REFERENCE
SERIAL_NUMBER
ORDER_ID
UNKNOWN_IDENTIFIER
```

这是防止误匹配的关键。

例如标题：

```text
“适配劳力士 126610LN 的表带，货号 B12345”
```

里面出现了 `126610LN`，但它不是当前售卖对象的腕表 reference。

如果只做正则抽取，很容易产生灾难性 false positive。

---

### 7.1.3 `reference_entity`

```sql
CREATE TABLE reference_entity (
    reference_entity_id BIGINT PRIMARY KEY,
    brand_id            BIGINT NOT NULL,
    canonical_reference VARCHAR(128) NOT NULL,
    collection_id       BIGINT,
    status              VARCHAR(32) NOT NULL,
    created_at          TIMESTAMP NOT NULL,
    UNIQUE(brand_id, canonical_reference)
);
```

真正的实体 ID 是：

```text
(brand_id, canonical_reference)
```

不能只用 reference string，因为不同品牌可能出现相同/相似编号。

---

### 7.1.4 `product_reference_assignment`

```sql
CREATE TABLE product_reference_assignment (
    record_id            BIGINT PRIMARY KEY,
    reference_entity_id  BIGINT,
    decision             VARCHAR(32) NOT NULL,
    decision_reason      VARCHAR(128) NOT NULL,
    decision_version     VARCHAR(64) NOT NULL,
    evidence_count       INT NOT NULL,
    strong_evidence_cnt  INT NOT NULL,
    conflict_count       INT NOT NULL,
    decided_at           TIMESTAMP NOT NULL
);
```

`decision` 建议只有：

```text
SAFE_ASSIGNED
ABSTAIN
CONFLICT
HUMAN_CONFIRMED
REVOKED
```

不要设计一个“LOW_CONFIDENCE_ASSIGNED”然后仍然自动发布。

---

### 7.1.5 `reference_audit_edge`

这张表只保存 suspicious graph，不保存全量同 reference pair。

```sql
CREATE TABLE reference_audit_edge (
    left_record_id       BIGINT NOT NULL,
    right_record_id      BIGINT NOT NULL,
    edge_type            VARCHAR(32) NOT NULL,
    score                DECIMAL(8,6),
    left_ref_entity_id   BIGINT,
    right_ref_entity_id  BIGINT,
    conflict_mask        BIGINT NOT NULL,
    status               VARCHAR(32) NOT NULL,
    created_at           TIMESTAMP NOT NULL,
    PRIMARY KEY(left_record_id, right_record_id, edge_type)
);
```

只放这些边：

```text
MODEL_HIGH_SIMILARITY_BUT_REF_CONFLICT
OCR_TEXT_DISAGREEMENT
MULTI_REF_CANDIDATES
BRAND_REF_CONFLICT
HARD_NEGATIVE_NEIGHBOR
HUMAN_REVIEW_RELATION
LEGACY_MATCH_EDGE
```

这样 graph 规模会比全量记录小几个数量级。

---

# 8. Reference 抽取与规范化：主链路怎么做

## 8.1 第一层：结构化字段

优先级最高：

```text
平台原生 reference/model 字段
```

但即使字段名叫 `model`，也不要默认可信，先做来源级 schema audit。

每个 source 建配置：

```yaml
source: leixiaoan
fields:
  reference:
    path: product.reference
    trust: HIGH
  title:
    path: product.title
```

来源字段先统一到 canonical schema。

---

## 8.2 第二层：确定性字符串抽取

每个品牌维护 reference grammar。

不要用一个全局正则匹配所有字母数字串。

例如概念上：

```python
BRAND_PATTERNS = {
    "rolex": [...],
    "omega": [...],
    "cartier": [...],
}
```

每个命中都保留：

```text
raw span
normalized candidate
前后文
extractor version
```

而不是只落最终字符串。

---

## 8.3 第三层：角色分类

对于每个“像 reference 的字符串”，判断它是不是：

```text
当前商品本身的 reference
```

建议先做规则，再做小模型/LLM fallback。

### 规则可以覆盖的明显负例

```text
适配 / compatible with / for / suitable for
表带 / 表盒 / 配件 / 镜面 / 表扣
货号 / SKU / 商品编号 / 编号
```

如果候选 reference 出现在上述上下文，默认：

```text
ABSTAIN / NOT_PRODUCT_REFERENCE
```

而不是自动认定。

---

## 8.4 第四层：OCR

图片只作为 reference 证据来源之一。

建议图片区分：

```text
表背
保卡
吊牌
证书
表盘
普通商品图
```

OCR reference 的可信度要按图片类型分层。

例如：

```text
保卡 reference 区域 OCR
    >
表背刻字 OCR
    >
普通图片自由 OCR
```

但即使 OCR 很强，也不要让它越过已有的强冲突。

例如：

```text
structured_ref = 126610LV
OCR = 126610LN
```

应进入：

```text
CONFLICT
```

而不是选一个更高分的覆盖另一个。

---

# 9. Canonicalization：只能做“可证明等价”的规范化

这是 precision-first 系统非常容易做过头的地方。

允许的规范化通常包括：

```text
trim
统一大小写
全角半角
明确允许的分隔符差异
品牌目录中已验证的 formatting alias
```

例如：

```text
AB-1234
AB 1234
ab1234
```

是否等价不能靠全局规则拍脑袋，必须按品牌/系列 catalog 验证。

不建议：

```text
编辑距离小于 2 就合并
前缀一样就合并
去掉所有字母后数字一样就合并
视觉很像就合并
```

因为邻近 reference 恰恰是腕表最危险的 hard negative。

推荐 API：

```python
def canonicalize_reference(
    brand_id: int,
    raw_reference: str,
    catalog_version: str,
) -> CanonicalizationResult:
    ...
```

返回：

```json
{
  "status": "EXACT_CATALOG_ALIAS",
  "canonical_reference": "126610LN",
  "rule_id": "rolex-separator-normalization-v3"
}
```

如果目录里没有证明：

```text
UNKNOWN
```

宁可 abstain。

---

# 10. 自动 Assignment 决策规则

我建议第一版甚至不要训练 pairwise matcher，先把自动发布规则做得极保守。

伪代码：

```python
def assign_reference(record, evidences, catalog):
    # 1. 排除不是 PRODUCT_REFERENCE 的 identifier
    product_refs = [
        e for e in evidences
        if e.role == "PRODUCT_REFERENCE"
    ]

    # 2. 只保留可安全 canonicalize 的候选
    canon = [
        canonicalize(e, catalog)
        for e in product_refs
        if e.confidence >= evidence_threshold(e.evidence_source)
    ]

    canon = [x for x in canon if x.status in SAFE_STATUSES]

    # 3. 没有可靠 reference => 拒识
    if not canon:
        return ABSTAIN("NO_STRONG_REFERENCE")

    # 4. 出现多个不同 canonical reference => 冲突
    unique_refs = set(x.canonical_reference for x in canon)
    if len(unique_refs) != 1:
        return CONFLICT("MULTIPLE_STRONG_REFERENCES")

    # 5. 品牌不确定/冲突 => 拒识
    if not brand_is_safe(record, product_refs):
        return CONFLICT("BRAND_CONFLICT")

    # 6. 若强负证据存在，不能自动放行
    if has_strong_negative_evidence(record, product_refs):
        return CONFLICT("STRONG_NEGATIVE_EVIDENCE")

    # 7. 唯一 reference 才允许 SAFE_ASSIGNED
    ref = next(iter(unique_refs))
    return SAFE_ASSIGNED(brand=record.brand_id, reference=ref)
```

自动同款关系只来自：

```text
SAFE_ASSIGNED(A).reference_entity_id
    ==
SAFE_ASSIGNED(B).reference_entity_id
```

没有第二条自动 Match 规则。

---

# 11. 如何真正借鉴 GraLMatch：Reference Conflict Graph

这里是本次分析最核心的落地点。

## 11.1 图里不要只有 record node

建议加 **Reference Anchor Node**：

```text
Record Nodes:
R1, R2, R3...

Reference Anchor Nodes:
REF:ROLEX:126610LN
REF:ROLEX:126610LV
```

边分两大类：

```text
assignment edge
candidate/similarity edge
```

示例：

```text
REF:126610LN
      |
      | strong structured reference
      |
     R1 ----- R2 ----- R3
                    similarity
                        |
                        |
                  REF:126610LV
```

如果一个 suspicious component 同时触达两个不同的 strong reference anchor，直接定义：

```text
ANCHOR_CONFLICT
```

这是比“组件太大”更适合当前业务的停止/切分条件。

---

## 11.2 当前业务的 Graph Invariants

建议把以下规则写成不可绕过的 invariant。

### Invariant 1：一个自动发布组件最多一个 strong reference anchor

```text
count(distinct strong_reference_anchor) <= 1
```

否则整个异常子图不自动合并。

### Invariant 2：一个 record 最多一个 SAFE reference assignment

```text
record_id -> exactly 0 or 1 safe reference entity
```

### Invariant 3：不同 canonical reference 永不自动传递

即使：

```text
text similarity = 0.99999
image similarity = 0.99999
```

只要：

```text
canonical_ref_A != canonical_ref_B
```

自动 Match 必须是 false。

### Invariant 4：同一个 reference 可以有任意数量、任意 source 分布的 listing

因此绝不能使用：

```text
component_size <= 3
```

作为正确性条件。

### Invariant 5：Graph edge 不能升级 reference identity

```text
similarity edge
不能把 unknown record 自动“继承”为邻居 reference
```

除非有独立 reference evidence。

这条可以直接阻断大部分 transitive pollution。

---

# 12. 把 Minimum Cut 改成“冲突锚点分离”

论文是：

```text
component 太大
 -> min edge cut
```

当前系统建议改成：

```text
component 中存在 >= 2 个互斥 reference anchor
 -> 寻找最小代价 cut
 -> 目标是把不同 reference anchors 分离
```

也就是从：

```text
size-driven cut
```

改成：

```text
semantic-conflict-driven cut
```

而且 edge 需要有成本。

建议 edge cost：

```text
strong structured ref edge       cost = +infinity
human confirmed edge             cost = +infinity
catalog exact alias edge          cost = very high
OCR card evidence edge            cost = high
model similarity edge             cost = low
weak title embedding edge         cost = very low
legacy heuristic edge             cost = very low
```

目标：

> 尽量切掉弱的 candidate/similarity edge，而不要切掉强 reference evidence。

可以用 weighted min-cut：

```python
cut_value, (left, right) = nx.minimum_cut(
    graph,
    source=anchor_a,
    target=anchor_b,
    capacity="cut_cost",
)
```

对于多个冲突 anchor，可以：

1. 先按 anchor pair 计算最小割；
2. 合并候选 cut edges；
3. 根据 edge trust 从低到高删除；
4. 每次删除后重新检测 anchor conflict；
5. 冲突消失即停止。

注意：

> 自动删除只针对 `candidate/similarity/legacy` 边；强 reference assignment 不自动删，强证据互相冲突时直接进入人工复核。

---

# 13. Edge Betweenness 在这里怎么用

不建议把 betweenness 当删除真值。

更合适的是形成 review score：

```text
review_score =
    w1 * edge_betweenness
  + w2 * anchor_conflict_path_count
  + w3 * reference_disagreement
  + w4 * model_uncertainty
  + w5 * source_risk
  + w6 * evidence_role_risk
```

重点优先审：

```text
连接两个不同 reference anchor 的路径上
betweenness 很高
且 edge 本身不是强 evidence
```

这样的边高度可能是 false-positive bridge。

这正是 GraLMatch 最应该迁移的部分。

---

# 14. 1M～10M 数据规模下怎么实现

## 14.1 不要建立全量商品 pair graph

主路径复杂度应接近：

```text
O(N) extraction
+
O(N log K) candidate lookup
+
O(N) exact reference assignment
```

而不是：

```text
O(N^2)
```

---

## 14.2 推荐组件

第一版可以非常务实：

```text
对象存储 / Data Lake
    保存 raw snapshot、OCR 中间产物

PostgreSQL / MySQL
    保存 reference_entity / assignment / evidence 元数据

OpenSearch / Elasticsearch
    做 title/brand/reference candidate retrieval

Redis
    缓存 canonical reference dictionary / hot aliases

Kafka（如果已有）
    增量 change event

Python workers
    regex / role classifier / OCR / canonicalizer

Spark / Flink（全量回刷和大批计算）
    只在数据量和 SLA 需要时引入

NetworkX / igraph
    仅处理 suspicious connected components
```

不建议为了“用了图算法”就上 Neo4j 做全库 identity 主存储。

主实体映射本质是：

```text
record_id -> reference_entity_id
```

关系库和 KV/index 已经非常高效。

---

## 14.3 增量流程

每条新增/更新记录：

```text
1. ingest raw record
2. brand canonicalization
3. reference evidence extraction
4. role classification
5. safe canonicalization
6. assignment decision
7. exact reference entity lookup/upsert
8. graph conflict audit（只在必要时）
9. publish SAFE_ASSIGNED
10. 其余进入 abstain / review queue
```

关键是幂等：

```text
(source, source_record_id, ingest_version)
```

同一版本重复跑结果必须一致。

---

# 15. 增量变更与撤销机制

持续爬取意味着一个 listing 可能后续修改标题、字段或图片。

所以 assignment 必须版本化，不能只有最终状态。

建议保留：

```text
assignment_version
extractor_version
catalog_version
input_record_version
```

如果新版本出现：

```text
旧：SAFE_ASSIGNED -> 126610LN
新：strong field -> 126610LV
```

不能直接覆盖。

流程应是：

```text
SAFE_ASSIGNED
    ↓
REVOKED
    ↓
CONFLICT
    ↓
从自动同款索引移除
    ↓
局部 graph audit
```

并记录事件：

```text
MATCH_REVOKED_DUE_TO_REFERENCE_CHANGE
```

这是 production 系统比论文离线实验必须多出来的一层。

---

# 16. 几百对黄金标签应该怎么花

不要随机标几百对。

随机样本大部分太容易，不能验证 precision-first 系统。

建议黄金集至少 70% 都放 hard negatives：

```text
同品牌
同系列
reference 仅 1～2 字符差异
外观高度相似
标题包含多个型号
配件标题包含适配型号
结构化字段与标题冲突
OCR 与文本冲突
品牌别名/跨语言
旧型号/新型号格式差异
同 reference 的合法多 listing
```

标签不仅标：

```text
Match / NoMatch
```

还建议标：

```json
{
  "brand": "Rolex",
  "product_reference": "126610LN",
  "reference_role": "PRODUCT_REFERENCE",
  "evidence_source": "TITLE",
  "should_auto_assign": true,
  "hard_negative_reason": null
}
```

这样几百条标签可以同时服务：

- reference extractor；
- identifier role classifier；
- canonicalization 规则；
- 自动发布阈值；
- 图异常优先级；
- regression test。

---

# 17. 评估指标：不要只看 F1

## 17.1 核心上线指标

建议按优先级：

### P0：Auto-match False Positive Count

```text
自动发布结果中的误匹配数量
```

目标：

```text
0（在验证集/持续抽检中）
```

### P0：Cross-reference Contamination Rate

```text
一个自动实体中包含 >1 个可信 canonical reference 的比例
```

目标：

```text
0
```

### P0：Strong Anchor Conflict Auto-Publish Count

出现多个 strong anchor 仍被自动发布的数量。

目标：

```text
0
```

### P1：Coverage / Auto-assignment Rate

```text
SAFE_ASSIGNED / all records
```

这个可以慢慢优化。

### P1：Abstain Rate

允许高。

### P1：Human Review Yield

人工队列里真正发现问题的比例。

### P2：Recall

不是第一上线门槛。

---

## 17.2 Group-level 指标

借鉴 GraLMatch，建议加：

```text
reference cluster purity
```

可以定义：

```text
purity(cluster)
=
cluster 内占比最高 canonical reference 的记录数
/
cluster 内有可信 reference 的记录数
```

自动发布 cluster 必须：

```text
purity = 1.0
```

但注意：如果很多记录没有可信 reference，不能因为“没有发现冲突”就推断正确。

所以需要同时报告：

```text
anchor_coverage(cluster)
```

---

# 18. 不能把“几百条无错”误认为统计上已经证明 99.99% precision

当前需求说“绝对不能误匹配”。工程上必须把这理解为：

```text
使用硬规则、可解释证据、拒识和人工复核
把 false positive 风险压到业务可接受的极低水平
```

不能只靠：

```text
在 300 个样本里没看到错误
=> 线上一定 99.99%
```

样本量远不足以证明这种极高 precision。

所以系统安全性要来自：

1. identity 业务定义简单明确：reference equality；
2. 自动合并只接受唯一 canonical reference；
3. 所有模糊场景 abstain；
4. 强证据冲突硬熔断；
5. 图一致性作为第二道审计；
6. 高频 hard-negative 回归；
7. 线上持续抽检和版本回滚。

---

# 19. 推荐的服务拆分

如果要直接进入工程实现，我建议拆成下面 7 个逻辑模块。

## 19.1 `brand-normalizer`

输入：

```json
{
  "source": "xxx",
  "brand_raw": "劳力士 / Rolex / ROLEX"
}
```

输出：

```json
{
  "brand_id": 101,
  "brand_canonical": "Rolex",
  "status": "SAFE"
}
```

---

## 19.2 `reference-extractor`

输出多个 evidence，不输出最终 identity：

```json
{
  "record_id": 123,
  "candidates": [
    {
      "raw": "126610LN",
      "source": "TITLE_REGEX",
      "role": "PRODUCT_REFERENCE",
      "confidence": 0.999
    }
  ]
}
```

---

## 19.3 `reference-canonicalizer`

只做白名单式、catalog-aware 规范化。

---

## 19.4 `reference-decision-engine`

把 evidence 聚合为：

```text
SAFE_ASSIGNED
ABSTAIN
CONFLICT
```

这是自动发布的唯一闸门。

---

## 19.5 `reference-entity-registry`

负责：

```text
(brand_id, canonical_reference)
 -> reference_entity_id
```

并维护 reference alias / catalog version。

---

## 19.6 `graph-safety-auditor`

只接收 suspicious records / legacy edges。

核心检查：

```text
anchor conflict
bridge edge
weighted min-cut
betweenness
source inconsistency
```

输出：

```text
ALLOW
QUARANTINE
REVIEW
REVOKE_CANDIDATE_EDGE
```

它不能返回：

```text
CREATE_MATCH_BECAUSE_SIMILAR
```

---

## 19.7 `review-console`

人工界面一屏至少展示：

```text
三源原始字段
标题高亮 reference span
结构化 reference
OCR 图片 crop
canonicalization 规则
冲突 reference anchors
图上 bridge edge
相邻 hard-negative reference
历史 assignment 版本
```

人工的价值应该集中在最难的 1%～5%，而不是替系统做普通 exact join。

---

# 20. Graph Safety Auditor 的参考伪代码

```python
from collections import defaultdict
import networkx as nx

STRONG_EDGE_TYPES = {
    "STRUCTURED_REF",
    "HUMAN_CONFIRMED_REF",
    "CATALOG_EXACT_ALIAS",
}

CUTTABLE_EDGE_TYPES = {
    "MODEL_SIMILARITY",
    "TEXT_RETRIEVAL",
    "IMAGE_SIMILARITY",
    "LEGACY_MATCH",
}


def audit_component(g: nx.Graph):
    anchors = {
        n for n, d in g.nodes(data=True)
        if d.get("node_type") == "REFERENCE_ANCHOR"
        and d.get("trust") == "STRONG"
    }

    if len(anchors) <= 1:
        return {
            "decision": "NO_ANCHOR_CONFLICT",
            "review_edges": []
        }

    review_edges = set()
    anchors = list(anchors)

    for i in range(len(anchors)):
        for j in range(i + 1, len(anchors)):
            a, b = anchors[i], anchors[j]

            if not nx.has_path(g, a, b):
                continue

            # weighted min cut: weak edge cheap to cut
            cut_value, partition = nx.minimum_cut(
                g,
                a,
                b,
                capacity="cut_cost"
            )

            left, right = partition

            for u in left:
                for v in g.neighbors(u):
                    if v in right:
                        edge = g[u][v]
                        if edge["edge_type"] in CUTTABLE_EDGE_TYPES:
                            review_edges.add(tuple(sorted((u, v))))

    # Betweenness 只用于排序，不作为真值
    bc = nx.edge_betweenness_centrality(g, weight=None)

    ranked = sorted(
        review_edges,
        key=lambda e: bc.get(e, bc.get((e[1], e[0]), 0)),
        reverse=True,
    )

    return {
        "decision": "ANCHOR_CONFLICT",
        "review_edges": ranked,
    }
```

生产版需要：

- 限制 component 大小；
- 超大组件先按 brand/reference anchor 分区；
- 只在 candidate graph 上跑；
- 加 timeout；
- 结果持久化；
- 记录 graph snapshot/version；
- 人工决策回流 edge trust。

---

# 21. Candidate Retrieval 怎么做才不会越权

对于没有 reference 的 record，可以用：

```text
品牌
系列
title BM25
char n-gram
multilingual embedding
image embedding
```

召回 top-K reference candidates。

但是返回结构必须是：

```json
{
  "record_id": 123,
  "candidate_references": [
    {"reference": "126610LN", "score": 0.94},
    {"reference": "126610LV", "score": 0.92}
  ]
}
```

而不是：

```json
{
  "matched_record_id": 456
}
```

接下来还必须找独立 reference evidence。

如果不能证明唯一 reference：

```text
ABSTAIN
```

这是“召回”和“身份判定”必须分离的地方。

---

# 22. 图片在系统里的正确位置

当前 Spec 明确“有图片可用”。

图片非常有价值，但不应该定义 identity。

推荐图片用途：

1. OCR reference；
2. 判断 OCR 字符串是否位于保卡/表背等可信区域；
3. 发现文本 reference 与视觉不一致；
4. 人工复核排序；
5. 候选 reference retrieval。

不推荐：

```text
image similarity > threshold
=> 自动 Match
```

因为：

```text
同系列不同 reference 外观可能极其接近
```

在 precision-first 场景，图片更适合成为：

```text
negative / conflict evidence
```

而不是最终 positive identity evidence。

---

# 23. LLM / SLM 应该放在哪里

推荐：

```text
高召回抽取器 / 角色分类器 / 解释器
```

不推荐：

```text
最终 Match judge
```

可以设计 JSON schema：

```json
{
  "brand": "Rolex",
  "identifiers": [
    {
      "value": "126610LN",
      "type": "REFERENCE",
      "role": "CURRENT_PRODUCT",
      "evidence_text": "..."
    }
  ]
}
```

但是 LLM 输出必须再次经过：

```text
schema validation
brand catalog validation
canonicalization
conflict check
```

LLM 自己不能创建 canonical reference entity。

---

# 24. 最值得从 GraLMatch 代码直接复用的部分

## 可以直接参考

### 24.1 Pipeline 分层

```text
blocking -> scoring -> graph -> cleanup -> groups
```

虽然主身份逻辑要改，但分阶段 pipeline 很清晰。

### 24.2 Graph 数据模型

```text
record node + edge attributes(prob, match_type)
```

可以扩展成：

```text
edge_type
trust
cut_cost
source
extractor_version
conflict_mask
```

### 24.3 Minimum Cut / Betweenness 的实现思路

NetworkX 代码足够作为异常组件 MVP。

### 24.4 中间结果可审计

原代码会保存：

```text
pairwise_matches_preds.csv
pre_cleanup_transitive_matches.csv
post_graph_cleanup_matches.csv
graph_cleanup_deleted_edges.csv
```

生产版不该继续用 CSV 做主存储，但“保留每一步中间决策”这个理念必须保留。

---

# 25. 不应该直接复用的部分

## 25.1 `component_size > num_sources` 就切

必须删除这个业务假设。

## 25.2 `component_size > 5 * num_sources` 做 min-cut

也不能用。

改成：

```text
存在互斥 strong reference anchors
才触发 semantic conflict cut
```

## 25.3 Pairwise model 直接生成最终正边

当前需求不建议。

## 25.4 补全 clique 的 transitive edges

千万级业务不物化。

使用：

```text
reference_entity_id
```

表达传递闭包。

## 25.5 CSV / pandas / NetworkX 全库运行

只保留给实验或 suspicious subgraph。

---

# 26. 第一版 MVP：两周左右可验证的路径

这里给一个不依赖复杂训练、可以很快跑起来的版本。

## Phase 1：只做 Reference-first Exact Identity

### Day 1-3

统一三源 schema：

```text
source
source_record_id
brand
title
description
structured_reference
images
```

### Day 3-5

实现：

```text
brand normalizer
brand-specific reference regex
safe canonicalizer
```

### Day 5-7

落表：

```text
reference_evidence
reference_entity
product_reference_assignment
```

先只自动接受：

```text
structured reference
或
标题唯一 reference + catalog exact validation
```

其余全部 abstain。

---

## Phase 2：加 Graph Safety Auditor

### Day 8-10

建立 suspicious edges：

```text
旧系统已有 match edge
高文本相似但 ref 不同
OCR 与文本冲突
同系列邻近 ref
```

### Day 10-12

实现：

```text
strong anchor conflict detector
weighted min-cut
betweenness review priority
```

### Day 12-14

做 review queue + hard-negative regression。

第一版不用训练复杂 matcher，也已经可以验证核心架构正确性。

---

# 27. 第二版：把少量黄金标签用于“最有价值的模型”

在 MVP 稳定后，我最建议训练的不是 pairwise Match model，而是：

```text
Identifier Role Classifier
```

任务：

```text
标题/描述里的一个字母数字串
到底是：

CURRENT_PRODUCT_REFERENCE
ACCESSORY_COMPATIBLE_REFERENCE
PLATFORM_SKU
SERIAL
OTHER
```

为什么它比 Match model 更值？

因为当前 identity 已经由 reference equality 定义。

真正困难的是：

```text
“这个字符串到底是不是当前商品的 reference？”
```

这一步做好以后，后面的匹配反而只是数据库 exact join。

几百条人工标注正好够做一个小型高精度分类器/规则+模型混合器。

---

# 28. 第三版：主动学习只挑最危险的边

利用 GraLMatch 的图思想，人工样本不要随机选。

优先级可以定义：

```text
P0：不同 strong reference anchor 之间的 bridge edge
P1：同系列 adjacent reference，高视觉/文本相似
P2：structured field vs OCR 冲突
P3：多个 reference candidate 且 role 不清晰
P4：新增来源/新增品牌的 out-of-distribution record
```

每次人工确认后，回流：

```text
reference role rules
canonical alias table
extractor training set
graph edge trust
regression corpus
```

这样有限标注会持续提高整个系统，而不是只提高一个 pairwise classifier 的 F1。

---

# 29. 上线前必须做的 Failure Injection

建议主动构造错误数据，测试系统是否“宁可不匹配”。

至少覆盖：

```text
1. reference 最后一位改一个字符
2. title 同时出现两个 reference
3. 配件标题含腕表 reference
4. structured_ref 和 OCR ref 冲突
5. 品牌错误但 reference 相似
6. 同一 reference 不同格式
7. OCR 把 O/0、I/1、S/5 混淆
8. 历史旧 listing 更新为另一 reference
9. legacy match edge 把两个 reference component 连起来
10. 1000 条合法同 reference listing 形成大组件
```

第 10 条尤其重要：

> 要证明我们已经彻底移除了 GraLMatch 原算法里“不适合当前业务的 group size 假设”。

---

# 30. 生产 SLO / Safety Gate

自动发布前，每条 assignment 至少经过：

```text
[PASS] brand resolved
[PASS] exactly one canonical reference
[PASS] canonicalization rule is whitelisted
[PASS] no strong evidence conflict
[PASS] identifier role is CURRENT_PRODUCT_REFERENCE
[PASS] no incompatible anchor conflict
[PASS] extractor/catalog versions recorded
```

任何一项失败：

```text
ABSTAIN / REVIEW
```

上线策略建议：

```text
shadow mode
 -> 只记录不发布
 -> hard-negative audit
 -> 5% 品牌灰度
 -> 只开放最稳定品牌
 -> 扩大 coverage
```

不要先追求高覆盖率。

---

# 31. 与 TransClean 的区别和互补

`a` 目录已经分析过 TransClean。两者都研究多源 EM 的 false positive，但侧重点不同。

### GraLMatch

更偏：

```text
pairwise prediction
 -> group graph
 -> graph structural cleanup
```

重点在：

```text
Minimum Edge Cut
Edge Betweenness
component size / entity group matching
```

### TransClean

更偏：

```text
transitive consistency
negative transitive predictions
用不一致性定位 false positives
并可把标注回流继续训练
```

当前业务建议组合：

```text
Reference anchor conflict
        +
Transitive consistency
        +
Weighted min-cut / betweenness
```

但三者都只能是 **Safety Auditor**，不能取代 reference identity。

---

# 32. 最终推荐架构

```text
┌──────────────────────────────────────────────────────────────┐
│                        三源商品数据                           │
│              雷小安 / 腕表之家 / 奢当家                      │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              v
┌──────────────────────────────────────────────────────────────┐
│ 1. Source Adapter / Raw Store                                │
│    原始字段、版本、图片、source record id                     │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              v
┌──────────────────────────────────────────────────────────────┐
│ 2. Brand Normalization                                      │
│    alias -> canonical brand_id                              │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              v
┌──────────────────────────────────────────────────────────────┐
│ 3. Reference Evidence Extraction                            │
│    structured field / regex / role classifier / OCR / LLM   │
│    每条证据保留 provenance                                   │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              v
┌──────────────────────────────────────────────────────────────┐
│ 4. Safe Canonicalizer                                       │
│    brand-aware catalog whitelist                            │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              v
┌──────────────────────────────────────────────────────────────┐
│ 5. Decision Engine                                          │
│                                                              │
│    unique strong ref + no conflict -> SAFE_ASSIGNED          │
│    ambiguous                    -> ABSTAIN                    │
│    multiple strong refs         -> CONFLICT                  │
└───────────────────────┬───────────────────────┬──────────────┘
                        │                       │
              SAFE_ASSIGNED              ABSTAIN/CONFLICT
                        │                       │
                        v                       v
┌──────────────────────────────┐      ┌────────────────────────┐
│ 6. Reference Entity Registry │      │ 7. Candidate Retrieval │
│ (brand, canonical_reference) │      │    + Review Queue      │
└──────────────┬───────────────┘      └───────────┬────────────┘
               │                                  │
               │                                  v
               │                    ┌────────────────────────────┐
               │                    │ 8. Graph Safety Auditor    │
               │                    │ anchor conflict            │
               │                    │ weighted min-cut           │
               │                    │ betweenness                │
               │                    │ transitive consistency     │
               │                    └─────────────┬──────────────┘
               │                                  │
               v                                  v
┌──────────────────────────────┐      ┌────────────────────────┐
│ 9. Same-reference Query      │      │ 10. Human Feedback     │
│ reference_entity_id exact    │      │ rules/model/catalog    │
└──────────────────────────────┘      └────────────────────────┘
```

---

# 33. 最终方案判断

如果当前团队要我只给一个最优先落地决策，我会选择：

> **先不上通用 Pairwise Entity Matching 模型。先把三源记录统一为 `record -> canonical reference entity` 的保守解析系统；只有唯一、高可信且无冲突的 reference 才自动 assignment；然后把 GraLMatch 的图清洗思想改造成“冲突锚点安全审计”，专门抓 false-positive bridge 和 legacy 污染。**

这有几个直接收益：

1. **严格贴合业务定义**：同 reference 才同商品；
2. **可解释**：每个自动匹配都有明确 reference evidence；
3. **天然 precision-first**：模糊就 abstain；
4. **千万级可扩展**：identity 是 hash/index exact join，不是全量 pairwise；
5. **图算法用在正确的位置**：只审计异常组件，不处理全库；
6. **增量友好**：新增/修改 record 只重算局部 assignment；
7. **人工标签利用率高**：集中标注 reference role 和最危险 hard negatives；
8. **可逐步引入 OCR/LLM/多模态**：它们都作为 evidence，而不会越权定义实体。

GraLMatch 最重要的启示可以压缩成一句话：

> **最终实体关系的风险不是 pairwise error rate，而是错误边进入连通图以后造成的传播半径。**

在当前需求里，再加一句业务化改造：

> **最安全的办法不是努力让 Match 模型更会猜，而是让 reference 成为不可越过的身份边界，让图算法专门负责发现谁试图跨过这条边界。**

---

# 34. 参考资料

1. GraLMatch: Matching Groups of Entities with Graphs and Language Models  
   <https://arxiv.org/abs/2406.15015>

2. GraLMatch 官方代码  
   <https://github.com/FernandoDeMeer/GraLMatch>

3. 关键实现：`datainc_code/src/matching/matcher.py`  
   <https://github.com/FernandoDeMeer/GraLMatch/blob/main/datainc_code/src/matching/matcher.py>

4. 图生成与传递边实现：`datainc_code/src/data/full_data_utils.py`  
   <https://github.com/FernandoDeMeer/GraLMatch/blob/main/datainc_code/src/data/full_data_utils.py>

5. DistilBERT/PyTorch 训练与推理：`datainc_code/src/models/pytorch_model.py`  
   <https://github.com/FernandoDeMeer/GraLMatch/blob/main/datainc_code/src/models/pytorch_model.py>

6. 当前需求 Spec  
   <https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>
