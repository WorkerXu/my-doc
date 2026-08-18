# GraLMatch：Matching Groups of Entities with Graphs and Language Models

## 1. 结论先行

本文分析的条目来自 `奢侈品文章调研.md`：

- 论文：**GraLMatch: Matching Groups of Entities with Graphs and Language Models**
- arXiv：https://arxiv.org/abs/2406.15015
- 论文代码：https://github.com/FernandoDeMeer/GraLMatch
- EDBT 2025

针对 Notion Spec《跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》的约束——100 万到 1000 万记录、持续增量、字段稀疏、reference 可能埋在标题、图片可用、**同一商品严格定义为同一 reference number、绝对不能误匹配、precision 极端优先**——GraLMatch 最值得借鉴的不是“用 DistilBERT 判断两条商品是否相同”，而是下面这条工程原则：

> **不能只看 pairwise precision；一条 false-positive 边可能通过传递闭包把两个正确实体簇整体污染。最终系统必须显式管理图上的传递风险，并把高风险边在进入实体簇之前拦掉。**

但是 GraLMatch 的原始 Graph Cleanup 不能原样照搬：论文默认一个实体组通常“每个数据源最多一条记录”，因此把最终连通分量大小 `μ` 设置成数据源数量；二手商品场景中，同一个 reference 在雷小安/腕表之家/奢当家中的任一来源都可能同时出现大量商品记录，同 reference 的簇大小天然不受 3 个来源限制。论文自己也明确指出：当实体组大小高度异质时，这套按组大小切图的算法并不理想。

**因此本需求的直接落地方案应为：**

1. 把 **reference 抽取与规范化** 作为主任务，而不是训练一个自由度很高的“是否同商品”pairwise 模型。
2. 自动合并只允许基于 `brand_id + verified_reference_strict` 的确定性实体键。
3. 图片、文本相似度、LLM、embedding、跨源传递信息只负责：
   - 找 reference 候选；
   - 提升召回；
   - 发现冲突；
   - 对可疑记录排序；
   - **绝不单独拥有自动合并权限**。
4. 借鉴 GraLMatch 建一个独立的 **Graph Audit / Graph Guardrail**：任何软边连接两个不同的已验证 reference 实体时直接触发冲突；对“桥边”使用 edge betweenness / minimum cut 等图信号做高优先级复核。
5. 生产主实体关系不要依靠“软边的传递闭包”生成，而应依靠 reference 实体键；这样从架构上消除“一条错边污染整个簇”的根因。

一句话概括：**GraLMatch 应被改造成二奢系统的防错图层，而不是主匹配器。**

---

## 2. 为什么 GraLMatch 与本需求高度相关

传统实体匹配一般把问题拆成独立 pair：`A-B` 是不是同一个实体、`B-C` 是不是同一个实体。问题是只要最终要产出“实体组”，pairwise 预测天然会产生传递关系：如果模型判断 `A≈B`、`B≈C`，工程上通常会把 `A/B/C` 归到一个 connected component，即使模型从未直接判断 `A≈C`。

GraLMatch 的核心观察是：

- pairwise 模型即使看起来 precision 很高；
- 数据规模越大、来源越多；
- 候选边越多；
- 只要出现极少数 false positive；
- 错边就可能把两个本来正确的稠密实体簇连接起来；
- 传递闭包会放大单条错误，造成大量“隐式 false positive”。

这与腕表场景几乎同构。例如：

```text
雷小安 A: Rolex Submariner 126610LN
腕表之家 B: 劳力士 潜航者 126610LN
奢当家 C: Rolex Submariner 126610LV
```

如果文本/图像模型因为 `126610LN` 与 `126610LV` 外观、标题、系列都极其相近，错误地给出：

```text
A --match--> B
B --match--> C   # 错边
```

一个普通 union-find / connected-components 流程会立即得到 `{A,B,C}`，随后所有簇内 pair 都被隐式视作同实体。对“reference 必须完全相同”的业务定义而言，这种错误不可接受。

因此本项目真正需要优化的不是平均 F1，而是：

- **Auto-link Precision**：自动放行的边是否近乎零误报；
- **Cluster Contamination Rate**：一个实体簇是否混入过不同 verified reference；
- **False Bridge Rate**：是否存在连接两个 reference 实体的错误桥边；
- **Abstain Coverage**：宁愿多少数据进入拒识/人工队列，也不要自动误合并。

---

## 3. 论文技术架构拆解

GraLMatch 的端到端流程分四层：

```text
Raw Multi-source Records
        │
        ▼
     Blocking
        │
        ▼
Pairwise Match / NoMatch Model
        │
        ▼
Match Graph G=(V,E)
        │
        ▼
GraLMatch Graph Cleanup
        │
        ▼
Connected Components / Entity Groups
```

其中：

- `V`：记录；
- `E`：pairwise 模型预测为 Match 的边；
- connected component：传递意义下的实体组；
- Graph Cleanup：在形成最终实体组之前删除可能是 false positive 的边。

### 3.1 Blocking

论文强调大规模 EM 不可能计算所有 `n choose 2` pair，因此必须先 blocking。论文使用三类 blocking：

1. **ID Overlap**
   - 对证券使用 ISIN/CUSIP/VALOR/SEDOL 等 identifier overlap；
   - 这是低召回、高精度候选源。

2. **Token Overlap**
   - 记录 tokenize 后，选跨数据源 token overlap 最大的 top-N；
   - 目的是找到没有 identifier overlap、但文本接近的候选。

3. **Issuer Match**
   - 先匹配公司，再借助已匹配发行人生成证券候选；
   - 本质是利用上游实体关系作为下游 blocking 信号。

官方代码 `datainc_code/src/matching/matcher.py` 里 Token Overlap 并不是朴素全量字符串两两比较：它先把 token 集构造成 `scipy.sparse.csr_matrix` 的 record-token 指示矩阵，然后用稀疏矩阵乘法得到 overlap，并过滤同来源记录，再取 top-k。这个实现思路说明：**候选召回层必须用倒排/稀疏索引，而不是笛卡尔积。**

对本项目可直接映射为：

```text
Blocking A: brand + strict reference exact
Blocking B: brand + compact/search reference
Blocking C: brand + series + title token / embedding top-k
Blocking D: OCR extracted reference candidate
Blocking E: 已知 reference catalog 邻域（hard negative 候选）
```

但只有 Blocking A 有资格直接走自动实体键；B/C/D/E 都只能进入候选/审计链路。

### 3.2 Pairwise Matcher

论文用 DistilBERT 做二分类，输入两个 record，输出 `Match / NoMatch`。选择 DistilBERT 而不是大 LLM 的原因是规模：论文实际测试发现 Llama2 7B 约需 7 秒/候选 pair，百万级 pair 会使推理时间不可接受。

官方实现里：

- 模型预测结果保存为 `pairwise_matches_preds.csv`；
- 保留概率 `prob`；
- 通过 `match_type` 记录该边来自哪一种 blocking；
- `pre_cleanup` 默认使用很高的概率阈值（代码默认 `threshold=0.999`）生成 match graph。

这点对本项目非常重要：**每条边必须有 provenance**，至少要知道它是：

- 明确 reference 字段；
- 标题抽取；
- OCR；
- catalog 命中；
- embedding/LLM；
- 人工确认。

否则图清洗只能看到一个无语义的 score，无法针对“硬证据边”和“软猜测边”使用不同策略。

### 3.3 Graph Cleanup：Minimum Edge Cut + Edge Betweenness

GraLMatch 处理的是 match graph `G=(V,E)`。

其直觉是：false positive 往往是两个稠密真实实体簇之间极少数的“桥”。因此它使用两个经典图算法：

#### Minimum Edge Cut

找最少的一组边，使 connected component 被切开。

优势：

- 能快速把异常大的 component 拆开；
- 论文与代码都把它用于第一阶段大簇粗切。

#### Edge Betweenness Centrality

对边 `e` 定义：

```text
c_B(e) = Σ(s,t) σ(s,t|e) / σ(s,t)
```

如果一条边处于很多节点对最短路径上，它很可能是连接两个子簇的桥，因此 betweenness 高。

论文算法大致是：

```text
C = connected_components(G)

while largest_component_size > γ:
    cut = minimum_edge_cut(largest_component)
    remove(cut)

while largest_component_size > μ:
    e = edge_with_max_betweenness(largest_component)
    remove(e)

return connected_components(G)
```

论文将：

- `γ`：决定何时先用更快的 minimum edge cut；
- `μ`：最终最大 entity group 大小；
- 在论文目标数据中，`μ = 数据源数量`。

官方代码实现得更具体：

- component 大于 `5 * num_of_datasources` 时反复 `nx.minimum_edge_cut`；
- component 大于 `num_of_datasources` 时反复删除 `nx.edge_betweenness_centrality` 最大的边；
- 最后再把每个清理后的 connected component 补成传递闭包；
- 删除边会保留 `lid/rid/prob/match_type` 形成审计日志。

这套设计的最有价值部分不是参数，而是**“删除边必须可审计”**这一点。对零误匹配系统来说，所有自动 merge/reject 都应该保留 evidence 与 decision reason。

### 3.4 Pre-cleanup

图算法在超大 component 上会很慢。论文指出 Minimum Edge Cut 和 Edge Betweenness 对单个 component 都有大约 `O(mn)` 级别的代价，所以先做 pre-cleanup。

CompanyMatcher 的官方代码在 component 超过 `10 * num_of_datasources` 时，会优先删除 `text_match` 类型的边，也就是先牺牲低可信软边，保留 identifier 产生的强边。

这其实非常适合本需求，应该进一步强化为：

```text
Hard Reference Edge > Verified OCR Edge > Extracted Span Edge > Text/Image Similarity Edge
```

图发生冲突时，永远先移除低证据等级边，而不是简单根据全局概率阈值删边。

---

## 4. 论文实验中最值得本项目吸收的结果

论文有一个非常反直觉、但对本项目极关键的观察：**模型在普通 test split 上的 pairwise 指标很好，不代表在 blocking 产生的困难候选上也好。**

原因是训练/普通测试中的 negative 常常是随机负例，而生产 blocking 产生的 negative 恰恰是最像正例的 hard negatives。

在 Synthetic Companies 的端到端结果里：

- DistilBERT-15K 的 blocking candidate pair precision 高于 DistilBERT-ALL；
- 尽管 15K 模型 recall 更低，但 Graph Cleanup 后最终 group matching F1 更高；
- 作者据此强调：**大规模 entity group matching 中，pairwise precision 是决定性因素**。

这对腕表 reference 任务的标注策略有直接影响：几百对黄金标签不应该随机抽，而应该集中标：

- 同品牌、同系列、仅 reference 尾缀不同；
- 同 reference family、尺寸/材质/表圈不同；
- 标题包含“适用/兼容某型号”的表带、配件、盒证；
- 平台 SKU 长得像品牌 reference；
- OCR 把 `0/O`、`1/I`、`5/S`、`8/B` 混淆；
- reference 中 `/`、`.`、`-`、空格差异；
- 同图不同 reference、相似图不同 reference；
- 一个标题出现多个 reference 候选。

这比给模型大量简单 random negatives 更有价值。

---

## 5. GraLMatch 不能直接照搬的地方

### 5.1 `μ = 数据源数` 不成立

论文的关键先验是：一个实体组在一个来源里通常只有一条记录，所以实体组上限≈数据源数。

本项目恰恰相反：

```text
reference = 126610LN
```

同一个平台里可能同时存在几十、几百条二手在售/历史商品。三来源不代表 cluster size <= 3。

如果直接把 `μ=3`，系统会错误地把完全正确的同 reference 商品簇切碎。

### 5.2 “connected component = 实体”对零误报业务太危险

GraLMatch 最后仍然把清理后的 component 补成完整图，即接受剩余边的传递闭包。

本项目不应该这样做。只要存在软边：

```text
verified_ref_1 --soft--> unknown --soft--> verified_ref_2
```

如果 `verified_ref_1 != verified_ref_2`，无论 soft score 多高都不能通过传递闭包归并。

### 5.3 reference 是业务定义，不需要自由学习“实体相似性”

Spec 已经定义：“同一个商品”就是同一 reference number。

因此 pairwise matcher 最终回答“是不是同商品”其实绕远了。更稳的分解是：

```text
商品记录
  -> reference candidate extraction
  -> reference role classification
  -> reference canonicalization
  -> reference validation
  -> deterministic entity key
```

机器学习应该学习“这段字符串是不是当前商品的 reference”，而不是学习“这两个商品看起来是不是同款”。

### 5.4 图算法应从 merge engine 改成 anomaly detector

在本需求中更安全的策略是：

- 实体键负责合并；
- 图负责找异常；
- soft edge 永远不允许提升为实体真值；
- 图算法只产生 `incident`、`review_priority`、`veto`，不直接产生跨 reference union。

---

## 6. 直接可落地的目标架构

```text
                    ┌──────────────────────────┐
                    │ 雷小安 / 腕表之家 / 奢当家 │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │  Raw Ingestion / Version │
                    └────────────┬─────────────┘
                                 │
                                 ▼
        ┌──────────────────────────────────────────────┐
        │ Reference Extraction                         │
        │ explicit field / title span / OCR / catalog │
        └──────────────────────┬───────────────────────┘
                               │
                               ▼
        ┌──────────────────────────────────────────────┐
        │ Reference Resolver                           │
        │ role classify + strict/search norm + verify │
        └───────────┬──────────────────────┬───────────┘
                    │ VERIFIED             │ AMBIG/CONFLICT/MISSING
                    ▼                      ▼
       ┌───────────────────────┐     ┌──────────────────────┐
       │ Deterministic Entity  │     │ Candidate / Review   │
       │ brand + strict_ref    │     │ Graph                │
       └───────────┬───────────┘     └──────────┬───────────┘
                   │                             │
                   ▼                             ▼
       ┌───────────────────────┐     ┌──────────────────────┐
       │ Auto-link             │     │ GraLMatch-inspired   │
       │ Precision-first       │     │ Graph Guardrail      │
       └───────────┬───────────┘     └──────────┬───────────┘
                   │                             │
                   └──────────────┬──────────────┘
                                  ▼
                       ┌─────────────────────┐
                       │ Audit / Review / GT │
                       └─────────────────────┘
```

核心是维护两张“逻辑图”：

### Hard Entity Graph

只有确定证据：

```text
record -> (brand_id, verified_reference_strict)
```

它甚至不需要真的跑 NetworkX；可以直接通过数据库唯一键/哈希键建立 reference entity。

### Review Graph

包含：

- `strict exact`；
- `search_norm exact`；
- reference edit-distance 邻居；
- title/description similarity；
- image similarity；
- OCR candidate；
- LLM candidate；
- 人工反馈。

GraLMatch-inspired 算法只在这张图上跑，目标是**发现不一致和桥边**，而不是自动决定实体。

---

## 7. Reference 抽取与规范化：生产系统的真正核心

### 7.1 不要只存一个 `ref_norm`

建议至少存：

```text
ref_raw
ref_strict_norm
ref_search_norm
ref_role
ref_evidence_type
ref_evidence_span
ref_confidence
normalizer_version
extractor_version
```

### 7.2 strict_norm 与 search_norm 必须分离

错误做法：

```python
ref.upper().replace('-', '').replace('/', '').replace('.', '').replace(' ', '')
```

这是高风险的，因为某些品牌 reference 的分隔符可能有语义，过度规范化会把不同 reference 压成同一个字符串。

建议：

#### strict_norm：自动匹配使用

- Unicode NFKC；
- trim；
- 大小写统一；
- 只应用**品牌级已验证的等价规则**；
- 默认保留 `/ . -` 等可能有意义的符号；
- 不做模糊替换；
- 不做编辑距离自动纠错。

#### search_norm：候选召回使用

- 可以去常见分隔符；
- 可以做 compact form；
- 可以处理 OCR 常见混淆；
- 只能用于找 candidate，不能直接 auto-link。

这样：

```text
strict_norm equal -> 有机会自动放行
search_norm equal -> 只代表“值得比较”
```

### 7.3 抽取模型必须是 extractive，不要自由生成

LLM/RAG 的输出应限制为：

```json
{
  "candidate": "126610LN",
  "source": "title",
  "start": 18,
  "end": 26,
  "role": "product_reference"
}
```

模型应该从原字段中**指认 span**，而不是凭知识生成一个看起来合理的型号。否则“幻觉 reference”会变成极危险的 false positive 来源。

### 7.4 reference role classification

这是本需求很容易忽视的一层。

标题：

```text
劳力士潜航者风格 适配 126610LN 表带
```

里面的 `126610LN` 并不是当前商品 reference，而是兼容对象。如果只做 regex + exact match，会把表带并入手表实体。

因此每个 candidate 必须分类：

```text
product_reference
compatible_reference
mentioned_reference
platform_sku
shop_sku
internal_id
serial_number
unknown_identifier
```

自动实体键只接受 `product_reference`。

---

## 8. 自动匹配决策规则

推荐用明确状态机，而不是一个统一相似度阈值。

### 8.1 Reference 状态

```text
VERIFIED
AMBIGUOUS
CONFLICT
MISSING
INVALID
```

### 8.2 Auto-link 最小规则

```python
def can_auto_link(a, b):
    return (
        a.brand_id == b.brand_id
        and a.ref_state == "VERIFIED"
        and b.ref_state == "VERIFIED"
        and a.ref_strict_norm == b.ref_strict_norm
        and not has_hard_conflict(a, b)
    )
```

其中 `has_hard_conflict` 至少检查：

- 明确品牌冲突；
- 明确 reference 字段与标题 reference 冲突；
- product/reference role 冲突；
- catalog 显示该 reference 不属于该品牌；
- 同一记录出现两个互斥高置信 product reference；
- 人工负标注。

### 8.3 任何下面条件都必须 abstain

```text
ref 缺失
只有 image similarity
只有 title embedding similarity
只有 LLM 判断“应该是同款”
只有 search_norm 相同但 strict_norm 不同
OCR 候选与标题候选冲突
一个标题出现多个 product_reference 且无法唯一决策
```

---

## 9. GraLMatch-inspired Graph Guardrail 设计

### 9.1 Edge Schema

每条候选边都应保存：

```text
edge_id
left_record_id
right_record_id
edge_type
score
left_verified_entity_id
right_verified_entity_id
evidence_json
model_version
created_at
decision
review_status
```

建议 `edge_type`：

```text
HARD_REF_EXACT
FIELD_TITLE_CONSENSUS
OCR_TITLE_CONSENSUS
CATALOG_VALIDATED
SEARCH_REF_EQUAL
REF_EDIT_NEIGHBOR
TEXT_SIMILAR
IMAGE_SIMILAR
LLM_MATCH
MANUAL_POSITIVE
MANUAL_NEGATIVE
```

### 9.2 证据等级

```text
L5 MANUAL_POSITIVE
L4 VERIFIED strict reference
L3 双通道一致抽取（字段+标题 / 标题+OCR）
L2 单通道 reference candidate
L1 文本/图像相似
L0 无证据
```

冲突时优先移除低等级边；不要单纯使用一个概率 score。

### 9.3 最重要的图不变量

#### Invariant A：一个自动实体只能有一个 verified strict reference

```text
count(distinct verified_reference_strict) == 1
```

一旦 >1：

```text
cluster_status = CONFLICT
禁止新增 auto-link
产生 P0 incident
```

#### Invariant B：soft edge 不得跨两个不同 verified entity 自动 union

如果：

```text
entity(a) = Rolex:126610LN
entity(b) = Rolex:126610LV
```

即使：

```text
image_score = 0.9999
text_score = 0.9999
LLM = MATCH
```

也只能记为：

```text
candidate_conflict / hard_negative
```

#### Invariant C：传递信息只用于审计，不提升证据等级

```text
A hard-match B
B soft-match C
```

不能推出：

```text
A hard-match C
```

### 9.4 Bridge Detection

对 Review Graph 中出现以下情况的 component 才触发图算法：

- 包含 >=2 个不同 verified entity；
- component 异常膨胀；
- 一条新边把两个大 component 连起来；
- 同一 ref family 内出现大量近似 reference；
- soft edge 占比异常高。

对这些小范围 component：

1. 计算 bridges / articulation structure；
2. 计算 edge betweenness；
3. 必要时 minimum edge cut；
4. 结合 evidence level 排序；
5. 输出可疑边，不直接合并。

建议 suspicious score：

```text
suspicious(edge) =
    w1 * normalized_betweenness
  + w2 * is_cross_verified_reference
  + w3 * low_evidence_level
  + w4 * ref_string_conflict
  + w5 * source_role_conflict
  + w6 * component_size_jump
```

其中 `is_cross_verified_reference` 应是硬 veto，不只是普通加权特征。

### 9.5 为什么不要在 1000 万节点全局跑 NetworkX

论文代码使用 NetworkX，适合论文实验和中小 component，但不应该在整个 1000 万商品图上周期性跑全局 betweenness/min-cut。

正确做法：

- 先按品牌分区；
- 再按 reference/search_ref family 分区；
- 绝大多数 VERIFIED 记录直接通过实体键归档，不进入 graph algorithm；
- 只有 CONFLICT/AMBIGUOUS 形成的局部 component 进入 Graph Guardrail；
- 单 component 较大时再换 `networkit` / `igraph` / `rustworkx` 或分布式实现。

也就是说图算法是“异常慢路径”，不是“所有数据的主路径”。

---

## 10. 数据库模型建议

### 10.1 原始记录

```sql
CREATE TABLE product_record (
    id                  BIGSERIAL PRIMARY KEY,
    source              TEXT NOT NULL,
    source_item_id      TEXT NOT NULL,
    raw_json             JSONB NOT NULL,
    title                TEXT,
    brand_raw            TEXT,
    image_urls           JSONB,
    fetched_at           TIMESTAMPTZ NOT NULL,
    row_version          BIGINT NOT NULL,
    UNIQUE(source, source_item_id, row_version)
);
```

### 10.2 Reference 候选

```sql
CREATE TABLE reference_candidate (
    id                   BIGSERIAL PRIMARY KEY,
    record_id            BIGINT NOT NULL,
    brand_id             BIGINT,
    ref_raw              TEXT NOT NULL,
    ref_strict_norm      TEXT,
    ref_search_norm      TEXT,
    ref_role             TEXT NOT NULL,
    evidence_type        TEXT NOT NULL,
    evidence_span        JSONB,
    confidence           DOUBLE PRECISION,
    extractor_version    TEXT NOT NULL,
    normalizer_version   TEXT NOT NULL,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 10.3 最终 Reference 实体

```sql
CREATE TABLE reference_entity (
    id                   BIGSERIAL PRIMARY KEY,
    brand_id             BIGINT NOT NULL,
    ref_strict_norm      TEXT NOT NULL,
    status               TEXT NOT NULL DEFAULT 'ACTIVE',
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(brand_id, ref_strict_norm)
);
```

这张表的唯一约束就是系统最重要的“实体键”。

### 10.4 Record -> Entity

```sql
CREATE TABLE record_entity_link (
    record_id            BIGINT PRIMARY KEY,
    entity_id            BIGINT NOT NULL,
    decision_type        TEXT NOT NULL, -- AUTO / MANUAL
    evidence_snapshot    JSONB NOT NULL,
    resolver_version     TEXT NOT NULL,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 10.5 Candidate Edge / Graph Incident

```sql
CREATE TABLE match_candidate_edge (
    id                   BIGSERIAL PRIMARY KEY,
    left_record_id       BIGINT NOT NULL,
    right_record_id      BIGINT NOT NULL,
    edge_type            TEXT NOT NULL,
    score                DOUBLE PRECISION,
    evidence_json        JSONB NOT NULL,
    model_version        TEXT,
    decision             TEXT NOT NULL,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE graph_incident (
    id                   BIGSERIAL PRIMARY KEY,
    incident_type        TEXT NOT NULL,
    severity             TEXT NOT NULL,
    component_key        TEXT,
    edge_ids             JSONB,
    entity_ids           JSONB,
    reason_json          JSONB NOT NULL,
    status               TEXT NOT NULL DEFAULT 'OPEN',
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 11. 增量处理方案

100 万到 1000 万规模下，不能每来一批数据就重新全量 entity matching。

新记录的路径应当是：

```python
def process_record(record):
    # 1. 标准化品牌
    brand = resolve_brand(record)

    # 2. 多通道抽 reference candidate
    candidates = extract_candidates(record)

    # 3. role + catalog + evidence consistency
    resolved = resolve_reference(brand, candidates)

    # 4. 高精度状态机
    if resolved.state == "VERIFIED":
        entity = upsert_reference_entity(
            brand_id=brand.id,
            ref_strict_norm=resolved.strict_ref,
        )
        auto_link(record, entity, evidence=resolved.evidence)
        emit_audit_edges(record, resolved)
        return

    # 5. 其余全部拒识进入慢路径
    enqueue_candidate_generation(record, resolved)
```

慢路径：

```python
def review_path(record):
    candidates = retrieve_by_search_ref_text_image(record)
    edges = score_candidates(record, candidates)

    for edge in edges:
        if connects_different_verified_refs(edge):
            create_graph_incident(edge, "CROSS_VERIFIED_REFERENCE")
            continue

        save_review_edge(edge)

    run_local_graph_guardrail(record.brand_id, affected_component_ids(edges))
```

最关键的是：**新记录即使 soft candidate 与已有某个实体很像，也不能因为 connected component 自动入簇。**

---

## 12. 图片应该如何使用

Spec 明确有图片可用。图片很有价值，但在 reference-first 业务中用途必须受限。

推荐顺序：

### 用途 A：OCR 找 reference

优先处理：

- 表背刻字；
- 保卡；
- 吊牌；
- 盒贴；
- 机芯/壳号附近文本。

OCR 输出同样进入 `reference_candidate`，并保留 bbox/图片 ID。

### 用途 B：冲突否决

例如标题提取 `126610LN`，OCR 高置信得到 `126610LV`，不能让图像模型“平均一下”。应该直接：

```text
ref_state = CONFLICT
ABSTAIN
```

### 用途 C：人工复核排序

对 reference 缺失的数据，可以通过 CLIP/SigLIP/DINO 等召回视觉近邻，让人工更快判断；但 visual nearest neighbor 不能拥有 auto-link 权限。

### 不建议

```text
image embedding cosine > 0.99 -> 自动判同 reference
```

腕表同系列不同 reference 往往视觉极近，这正是 false positive 最危险的来源之一。

---

## 13. 黄金标签如何标才值钱

Spec 可接受人工标几百对。参考 GraLMatch 对 hard candidate 的实验，建议不要随机抽样。

建议黄金集分层：

### Positive

- 跨三源相同 reference、不同写法；
- reference 只存在标题；
- reference 只存在图片；
- 中英文品牌/系列名变化；
- 历史抓取版本变化。

### Hard Negative

优先级更高：

- 只差 1 个字符的 reference；
- 同 family 不同材质/尺寸/表圈；
- `LN/LV`、不同尾缀等；
- 配件标题引用目标手表 reference；
- 平台货号/库存号冒充 reference；
- OCR 易混字符；
- 多 reference 标题；
- 高图像相似但不同 reference；
- 高文本相似但不同 reference。

### Cluster-level Gold

不要只标 pair，还要手工建立几十个“小实体簇”，用于测试：

```text
一个 false edge 是否会污染整个 cluster
```

这正是 GraLMatch 指出的传统 pairwise benchmark 漏掉的问题。

---

## 14. 评测指标

不要只报 Precision / Recall / F1。

### 14.1 Auto-link Precision

只统计系统真正自动入实体的记录。

目标不是 `99%+`，而是尽可能逼近人工抽检中的零误报。

### 14.2 Selective Coverage

```text
coverage = 自动处理记录数 / 总记录数
```

precision-first 系统允许 coverage 低，然后随着规则、catalog、抽取器成熟逐步提升。

### 14.3 Cluster Contamination Rate

```text
contaminated_cluster =
  一个 auto entity 内出现 >=2 个 ground-truth reference
```

这是最重要的 group-level 安全指标之一。

### 14.4 Cross-reference Soft Edge Rate

统计模型/embedding/图像生成的软边有多少跨不同 verified reference，可持续挖 hard negatives。

### 14.5 Graph Bridge Incident Rate

统计新增数据是否频繁制造连接两个已验证实体的大桥边，可快速发现模型漂移或新来源字段语义变化。

### 14.6 Reference Extraction Metrics

分解评估：

```text
span precision
role precision
strict normalization precision
verified-reference precision
```

尤其要监控 `product_reference` role precision。

---

## 15. 上线安全机制

### 15.1 Shadow Mode

新 extractor / normalizer / graph rule 先只算结果不写 `record_entity_link`，与当前生产实体做 diff。

### 15.2 Version Everything

必须保存：

```text
extractor_version
normalizer_version
catalog_version
resolver_version
model_version
graph_rule_version
```

如果某个规则产生误匹配，可以精确找到受影响记录并重算。

### 15.3 Never Mutate Evidence

raw 字段、原图、OCR 原文、抽取 span、模型输出全部保留；canonical value 是派生数据，不覆盖 source truth。

### 15.4 Fail Closed

服务异常、catalog 不可用、模型超时、OCR 冲突时：

```text
ABSTAIN
```

而不是 fallback 到宽松相似度规则。

### 15.5 人工修正形成强负边

人工确认两个记录不同 reference 后，应该形成 `MANUAL_NEGATIVE`，未来任何 soft model 都不得覆盖。

---

## 16. 实施优先级

### P0：先实现不用 ML 也能安全工作的骨架

- source ingestion；
- brand canonicalization；
- strict/search 双 reference normalization；
- explicit field + title regex/span extractor；
- reference role；
- `reference_entity` 唯一键；
- VERIFIED / AMBIGUOUS / CONFLICT / MISSING 状态机；
- auto-link only on verified strict key；
- audit log。

完成这一层后系统已经能产生第一批**高 precision、低 coverage**结果。

### P1：加入候选与 Graph Guardrail

- search_ref / title / image 候选；
- candidate edge provenance；
- cross-verified-ref hard veto；
- local component；
- bridge / edge betweenness；
- incident queue；
- hard negative mining。

### P2：提升 reference coverage

- OCR；
- extractive LLM；
- brand-specific reference catalog；
- few-shot/小模型 role classifier；
- 人工反馈闭环。

### P3：规模化

- 增量流/批处理；
- 分区索引；
- candidate retrieval 服务；
- 只对异常局部 component 跑图算法；
- 监控分布漂移与 graph incident。

---

## 17. 与 GraLMatch 原实现的映射表

| GraLMatch | 二奢/腕表落地 |
|---|---|
| record node | 商品记录 |
| identifier overlap | verified reference strict exact |
| token overlap blocking | title/search_ref/embedding candidate retrieval |
| DistilBERT pair classifier | reference span/role/辅助 candidate scorer |
| match_type | evidence_type / edge_type |
| threshold=0.999 | 不采用单一全局阈值；使用证据状态机 |
| connected component | Review Graph 局部候选簇，不直接等于实体 |
| minimum edge cut | 异常大 component 的可疑桥定位 |
| edge betweenness | soft bridge review priority |
| μ = #sources | **舍弃**，商品同 ref 簇大小不受来源数限制 |
| transitive closure | **不允许软边自动传递为实体真值** |
| graph cleanup deleted edges | graph_incident + audited rejected edge |
| high pairwise precision | verified reference / role precision 优先 |

---

## 18. 最小可用实现伪代码

```python
class ResolutionState(str, Enum):
    VERIFIED = "VERIFIED"
    AMBIGUOUS = "AMBIGUOUS"
    CONFLICT = "CONFLICT"
    MISSING = "MISSING"
    INVALID = "INVALID"


def resolve_reference(record, brand, candidates, catalog):
    product_refs = [
        c for c in candidates
        if c.role == "product_reference"
    ]

    if not product_refs:
        return ResolutionState.MISSING, None

    # 只看 strict canonical value；search value 不能直接参与 auto-link
    strict_values = set(c.strict_norm for c in product_refs if c.strict_norm)

    if len(strict_values) > 1:
        return ResolutionState.CONFLICT, None

    strict_ref = next(iter(strict_values))

    if not catalog.is_valid(brand.id, strict_ref):
        return ResolutionState.INVALID, None

    # 必须达到可审计的证据规则，而非统一 neural score
    if has_sufficient_independent_evidence(product_refs):
        return ResolutionState.VERIFIED, strict_ref

    return ResolutionState.AMBIGUOUS, strict_ref


def resolve_entity(record, brand, resolution):
    state, strict_ref = resolution

    if state != ResolutionState.VERIFIED:
        return abstain(record, state)

    entity = db.upsert_reference_entity(
        brand_id=brand.id,
        ref_strict_norm=strict_ref,
    )

    assert not has_entity_reference_conflict(entity)

    return db.auto_link(
        record_id=record.id,
        entity_id=entity.id,
        evidence_snapshot=build_evidence(record),
    )
```

Graph Guardrail：

```python
def inspect_edge(edge):
    left_entity = verified_entity(edge.left_record_id)
    right_entity = verified_entity(edge.right_record_id)

    if left_entity and right_entity:
        if left_entity.id != right_entity.id:
            # 最强规则：跨 verified reference 的 soft edge 永不 union
            create_incident(
                edge=edge,
                incident_type="CROSS_VERIFIED_REFERENCE",
                severity="P0",
            )
            reject_soft_union(edge)
            return

    save_review_edge(edge)


def audit_component(component):
    verified_entities = distinct_verified_entities(component)

    if len(verified_entities) <= 1:
        return

    # GraLMatch 思路：找把多个可信子簇连起来的桥
    bc = edge_betweenness(component.soft_graph)
    bridges = graph_bridges(component.soft_graph)

    suspicious = rank_edges(
        component,
        betweenness=bc,
        bridges=bridges,
        evidence_level=True,
        reference_conflict=True,
    )

    enqueue_manual_review(suspicious)
```

---

## 19. 最终建议

如果严格按 Spec 的业务定义，最稳妥的架构不是“训练一个更强的多模态 matcher，把阈值调到 0.9999”，而是：

> **把 reference number 变成一等公民，用可追溯的抽取/角色判断/品牌级规范化得到 verified reference；用 `(brand, verified_reference)` 作为唯一自动实体键；所有文本、图片、LLM 与图算法都围绕这个键做候选发现、冲突发现和拒识。**

GraLMatch 给这个方案补上的关键一环是：**必须以 group/graph 的视角评价与防御 false positive，而不能只看单条 pair 的分数。**

最适合直接复用的思想与代码结构包括：

- blocking 后再做昂贵推理；
- 给每条边保存 `match_type/evidence_type`；
- 高置信边与低置信边分层；
- 大 component 先剔除低可信边；
- minimum edge cut / edge betweenness 找桥边；
- 所有删除/拒绝边保留审计日志；
- 评测必须包含 transitive/group-level 污染；
- 大规模下 precision 比 recall 更决定最终实体质量。

需要明确舍弃/改造的部分包括：

- 不采用 `μ = 数据源数量`；
- 不把 connected component 直接当最终商品实体；
- 不允许 soft edge 的传递闭包获得自动合并权限；
- 不把自由 pairwise matcher 放在 reference 规则之上。

这套改造后，GraLMatch 从一个“多源实体成组算法”变成了一个非常适合本项目的 **Reference-first Entity Resolution + Graph Safety Layer**：既保留大规模候选召回与多模态扩展空间，又把“绝对不能误匹配”落实成了数据库约束、状态机、图不变量和 fail-closed 机制，而不是寄希望于模型分数足够高。
