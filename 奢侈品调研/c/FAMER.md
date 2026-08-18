# FAMER：把多源相似图与增量修复架构改造成 Reference-First 高精度腕表实体匹配系统

> 分析人：c  
> 目标需求：跨源二奢/腕表商品实体匹配（雷小安 × 腕表之家 × 奢当家）  
> 核心约束：100 万–1000 万级持续增量；字段稀疏；reference 可能来自结构化字段、标题或图片；“同一个商品”定义为**同一 reference number / 型号**；**precision 极端优先，绝不能误匹配，允许漏匹配**；可人工标注几百对黄金样本。

## 0. 选题、去重与结论先行

本次从 `奢侈品文章调研.md` 选取：

- 项目：**FAMER（FAst Multi-source Entity Resolution system）**
- 项目页：https://dbs.uni-leipzig.de/research/projects/famer
- 核心论文：**Scalable Matching and Clustering of Entities with FAMER**
- 论文页：https://dbs.uni-leipzig.de/research/publications/scalable-matching-and-clustering-of-entities-with-famer
- 增量实体解析论文：https://dbs.uni-leipzig.de/research/publications/incremental-multi-source-entity-resolution-for-knowledge-graph-completion
- CLIP 聚类论文：https://dbs.uni-leipzig.de/research/publications/using-link-features-for-entity-clustering-in-knowledge-graphs
- 公开代码：https://git.informatik.uni-leipzig.de/dbs/FAMER

执行前重新核对 `奢侈品调研/c/`，其中已有 ALMSER-GB、Ameli、AnyMatch、DeepBlocker、pyJedAI、TransClean、GraLMatch、多模态属性抽取、置信度/选择性预测等分析，**没有 `FAMER.md`，也没有以 FAMER 为主体的分析**，因此满足“每次分析前排除已分析文章”的要求。

### 结论先行

FAMER 对当前需求最有价值的不是“把它的模糊匹配 + 聚类直接部署”，而是以下四个工程思想：

1. **多源先统一成一个可计算的数据/图模型，而不是为每一对来源维护独立 matcher**；
2. **Blocking → Pairwise Linking → Similarity Graph → Clustering/Repair** 的分层，让候选生成、证据计算、聚类决策各自可评测；
3. **分布式并行执行 + Block Split**，可处理大规模与倾斜数据；
4. **Incremental ER + 局部 Repair**，新数据到来时不必全量重算历史实体。

但当前业务的定义比通用 ER 强得多：

> “同一个商品”不是“文本/图片很像”，而是“同一个 canonical reference number”。

因此建议不要建设“listing 两两模糊匹配系统”，而是建设：

> **Listing → Canonical Reference Entity 的高精度实体链接系统**。

FAMER 原来的 `source-consistency` 思想应改成更符合业务的：

> **reference-consistency：一个自动放行的实体簇只能对应一个 canonical `(brand_id, reference)`；任何高可信 reference 冲突都必须阻断合并并进入 REVIEW/ABSTAIN。**

推荐最终架构可命名为 **RefGraph / Reference-First Entity Resolution**：

```text
三源原始 listing
    │
    ▼
Source Adapter / Field Role Normalizer
    │
    ▼
Reference Evidence Extractor
(structured field / title / OCR / catalog evidence)
    │
    ▼
Conservative Canonicalizer + Reference Catalog Validator
    │
    ├─────────────┐
    ▼             ▼
Exact Ref Join   Conflict / Missing Ref
    │             │
    ▼             ▼
AUTO_ACCEPT     REVIEW / ABSTAIN
    │             │
    └──────┬──────┘
           ▼
Versioned Evidence Graph + Resolution Audit Log
           │
           ▼
Incremental Local Repair / Re-evaluation
```

**核心安全原则：**

- 模糊文本相似度、embedding、图片相似度、LLM/VLM 都只能用于**候选发现、reference 抽取辅助或冲突检测**；
- 它们不能越过 canonical reference 的严格验证直接产生 `AUTO_ACCEPT`；
- 对 reference 不确定、来源角色不清楚、存在冲突、只凭视觉相似的记录一律拒识；
- precision-first 的关键不是把阈值调到 0.99，而是**把“自动合并”变成一个需要满足可审计硬证据条件的状态机**。

---

# 1. FAMER 原始系统解决什么问题

FAMER 面向的是 Multi-source Entity Resolution：多个数据源中存在表示同一现实实体的不同记录，需要在大数据规模上把它们链接并聚成实体簇。

官方项目把流程概括为两大块：

1. 构建多源实体的 **Similarity Graph**；
2. 在相似图上执行 **Clustering / Repair**。

其完整工作流可抽象为：

```text
Multiple Sources
      │
      ▼
Preprocessing / Schema Homogenization
      │
      ▼
Blocking
      │
      ▼
Pairwise Comparison + Similarity Computation
      │
      ▼
Match Classification
      │
      ▼
Similarity Graph  SG=(V,E)
      │
      ▼
Clustering
(CC / Star / Center / MergeCenter / CLIP / SplitMerge ...)
      │
      ▼
Cluster Repair / Fusion / Evaluation
```

FAMER 使用 Apache Flink 做并行执行，并与 Gradoop property graph / Flink Gelly 图计算能力结合。官方项目还进一步支持增量实体解析：已有结果保存在 clustered similarity graph 中，新实体到达后被分配到已有簇或新建簇；更复杂的 repair 路径会只重聚类受影响的一部分历史图。

这一点与当前三源持续抓取非常同构：**新数据持续进来，但不应该每次对 1000 万历史商品重新跑一次全量实体匹配。**

---

# 2. FAMER 的技术实现与架构拆解

## 2.1 Preprocessing：先把不同来源变成统一 Property Graph

FAMER 的公开 Wiki 显示，预处理层可以：

- 读取 Gradoop `LogicalGraph` 或 `GraphCollection`；
- 将任意多个来源组合成一个 `GraphCollection`；
- 给 vertex 增加必需的 `graphLabel` 来标识数据来源；
- 对来源间语义相同但字段名不同的 property 做 rename；
- 把多个 property 合并成新的 property；
- 直接加载 benchmark 和 perfect mapping / perfect clustering 进行评测。

例如不同来源可能分别有：

```text
source A: model_no
source B: ref
source C: 型号
```

FAMER 的思路不是让后续 matching 代码分别知道每个平台，而是在 preprocessing 阶段先变成统一属性。

### 对当前需求的启发

当前三源更需要的是**字段角色归一化**，而不仅是字段名归一化：

```text
raw field
  -> semantic role
     - manufacturer_reference
     - platform_sku
     - seller_sku
     - source_item_id
     - serial_number
     - series
     - title
     - compatible_reference
```

这里非常关键，因为电商标题里“像型号的字符串”并不一定是当前商品的 manufacturer reference。例如：

- 平台自己的商品 ID；
- 店铺 SKU；
- 表带/配件标题中的“适配 126610LN”；
- 盒证上另外一件商品的号码；
- 序列号；
- reference 的父系列或缩写。

如果只是统一成一个 `model_no` 字段，后续 exact match 也可能产生灾难性误配。

所以应借鉴 FAMER 的 source adapter 边界，但把它升级成：

```text
Source Adapter
  = schema mapping
  + field-role classification
  + provenance preservation
```

每一个 reference 候选都必须保留“从哪里来的、原始文本是什么、哪个 extractor 产生、哪个版本规则判断”的 provenance。

---

## 2.2 Blocking：把不可能比较的 pair 提前砍掉

FAMER 核心论文中，Blocking 用于避免多源记录的全笛卡尔积。

它支持类似：

- Standard Blocking；
- Sorted Neighborhood；
- 单 pass / multi-pass；
- 针对大 block 数据倾斜的 Block Split。

标准思想是：先按 blocking key 把可能相同的实体放到一个小块中，再只在 block 内计算 pairwise similarity。

### Block Split 的工程意义

大规模电商数据非常容易产生 hot block，例如只按：

```text
brand=Rolex
```

就会产生一个巨大块，使某一个 worker 承担异常高的比较量。FAMER 的 Block Split 思路是把大 block 拆到多个处理节点，从而减少分布式倾斜。

### 但当前需求可以比 FAMER 更进一步

因为业务定义是 exact reference，已抽取到可信 reference 的 listing 根本不需要 pairwise blocking：

```text
key = (canonical_brand_id, canonical_reference)
```

直接 hash / index lookup 即可。

也就是说：

```text
已解析 reference 的主路径： O(N) key resolution
未解析 reference 的疑难路径： Blocking / ANN top-K / soft retrieval
```

这比“所有数据都先做模糊 Blocking”更安全也更便宜。

推荐分桶策略：

```text
Tier 0: exact canonical reference key
Tier 1: canonical brand + validated reference prefix/pattern
Tier 2: canonical brand + series + model token
Tier 3: brand + text/image embedding top-K（仅 REVIEW 候选）
```

Tier 越往下，证据越软，**绝不能自动提升为同款事实**。

---

## 2.3 Pairwise Comparison / Match Classification：FAMER 把“边”显式化

Blocking 后，FAMER 会对候选 pair 的属性进行相似度计算，支持多个字符串相似函数，再根据配置把 pair 分类成 matching link，并附带组合 similarity。

最终形成：

```text
SG = (V, E)

V = entity records
E = candidate/matching links
edge.similarity ∈ [0,1]
```

这一层设计值得保留：**不要只保存“最终 MATCH/NO_MATCH”，要保存证据边及其来源。**

但在当前需求里，建议把一个含义模糊的 `similarity` edge 拆开：

```text
CANDIDATE_SIMILAR_TO
SUPPORTS_REFERENCE
CONFLICTS_REFERENCE
EXTRACTED_REFERENCE
RESOLVES_TO
```

其中只有 `RESOLVES_TO` 是业务实体链接结果，其余只是 evidence。

这样可避免非常危险的语义混淆：

```text
视觉相似 0.997
≠
同一个 reference
```

以及：

```text
标题相似 0.99
≠
同一个 reference
```

---

## 2.4 Similarity Graph：FAMER 最值得借鉴的“可审计中间层”

FAMER 把 pairwise matching 的输出保存成图而不是直接做不可解释的最终合并。

对当前系统，这个思路可以变成一个 **Evidence Graph**：

### 节点

```text
Listing
ReferenceEntity
Brand
Series（可选）
EvidenceArtifact（可选：图片/OCR chunk）
```

### 边

```text
Listing --RESOLVES_TO--> ReferenceEntity
Listing --HAS_CANDIDATE--> ReferenceEntity
Listing --HAS_EVIDENCE--> EvidenceArtifact
EvidenceArtifact --SUPPORTS--> ReferenceEntity
EvidenceArtifact --CONFLICTS--> ReferenceEntity
ReferenceEntity --BELONGS_TO--> Brand
```

### 最关键的图约束

自动接受的 `RESOLVES_TO` 必须是单值的：

```text
∀ listing l:
count(high_confidence RESOLVES_TO reference) <= 1
```

一个逻辑“同款簇”的所有 listing 必须满足：

```text
canonical_brand_id 相同
AND
canonical_reference 完全相同
```

若出现：

```text
structured field -> 126610LN
OCR             -> 126610LV
```

不能通过“多数票”或“整体相似度”自动选择一个，而应该直接：

```text
state = CONFLICT_REVIEW
```

这就是把 FAMER 的“图修复”改成 precision-first 的“证据冲突修复”。

---

## 2.5 Clustering：Connected Components 为什么不能直接用于本需求

FAMER 评测多种 clustering。最直观的基线是 Connected Components：只要通过相似边连通，就进入同一个 cluster。

但它在 precision-first 场景中存在致命风险：

```text
A -- B  （错误边）
B -- C
C -- D
```

一条 false-positive edge 就可能把两个真实实体簇连起来。

腕表尤其危险：相邻 reference 的外观和标题往往高度类似，例如只是后缀、材质、尺寸、盘面不同。如果把“视觉/文本相似”作为 equivalence edge 再做传递闭包，误合并会从 pair 级放大为 cluster 级。

### 推荐替代

最终业务分组不要由 graph connectivity 决定，而由 canonical key 决定：

```text
entity_id = hash(canonical_brand_id, canonical_reference)
```

图聚类只用于：

- 找冲突；
- 找疑似错误边；
- 组织人工复核；
- 发现可能遗漏的 reference alias；
- 影响范围分析；
- 局部 repair。

**聚类算法不是事实来源，reference 才是事实来源。**

---

## 2.6 SplitMerge：把“先拆错簇，再谨慎合并”的思想迁移过来

FAMER 的 SplitMerge 大致流程是：

1. 先基于 connected components 获得初始簇；
2. 检测 source conflict；
3. 优先去除冲突下较弱的 link；
4. 根据簇内相似度继续 split；
5. 生成 cluster representative；
6. 对满足条件的簇再 merge。

其中一个关键机制是维护 cluster 对应的 source 集合；如果 merge 会让来源冲突，则阻止或删除较弱边。

### 迁移到当前需求：Source-Consistency → Reference-Consistency

原算法关心：

```text
一个 clean source 在一个 cluster 中不能出现两个 entity
```

本业务更应该关心：

```text
一个 accepted cluster 中不能出现两个不同的 canonical reference
```

因此可以定义：

```python
def cluster_is_safe(cluster):
    refs = {
        e.canonical_reference
        for e in cluster.high_trust_reference_evidence
        if e.canonical_reference is not None
    }
    return len(refs) <= 1
```

一旦出现两个可信 reference：

```text
{126610LN, 126610LV}
```

优先动作不是“选择分数更高的边继续 merge”，而是：

```text
1. freeze automatic merge
2. split by canonical reference
3. mark bridging soft edges as suspicious
4. re-evaluate listings lacking hard ref
5. send unresolved bridge records to review
```

这比一般 cluster repair 更保守，但与“绝不能误匹配”一致。

---

## 2.7 CLIP：Link Strength 很适合做冲突边排序，但不应成为最终判据

FAMER 的 CLIP（Clustering based on Link Priority）不仅看 entity similarity，还引入 link strength / link degree 等边特征，并可用于构建或修复 cluster。论文给出了 Apache Flink 的并行实现。

这给当前项目一个实用启发：

当发生冲突时，可以给 evidence edge 建立明确的优先级：

```text
P0 结构化 manufacturer reference + 品牌型号库验证
P1 标题明确 reference + 位置/上下文角色验证 + 型号库验证
P2 多张图片 OCR 一致 + 型号库验证
P3 单图 OCR / 低质量标题 extraction
P4 embedding / image similarity / LLM guess
```

但需要强调：

> 高优先级 evidence 可以帮助判断“哪条软边最可能错”，不代表 P4 累积很多条就能推翻 P0。

因此不能简单 weighted sum：

```text
0.7 * image + 0.2 * title + 0.1 * ref
```

应该使用有 veto 的规则：

```text
hard reference conflict => reject/review
```

而不是被大量软证据淹没。

---

## 2.8 Apache Flink + Gradoop/Gelly：FAMER 的可扩展执行方式

FAMER 使用 Apache Flink 做分布式处理；相关研究中图被组织为 Gradoop logical graph，并在需要图迭代时转换成 Flink Gelly graph，通过 Scatter-Gather 一类顶点中心迭代执行 clustering，再转换回 Gradoop。

这套实现体现了两个很重要的工程原则：

1. **算法逻辑和数据规模解耦**：blocking、linking、clustering 都可以在分区数据上并行；
2. **把图状态作为显式中间产物**：方便继续 clustering、repair、evaluation。

### 当前项目是否应该直接沿用 Gradoop/Gelly？

不建议把“旧版 FAMER + Gradoop/Gelly”当成生产依赖直接搬运。

原因不是算法思想失效，而是：

- FAMER 的公开项目研究期较早；
- 当前业务最终判定是 reference exact identity，不需要为了主路径引入重型通用图聚类栈；
- 生产系统更需要 CDC、流式增量、幂等写、可审计状态与版本化 reprocess；
- 图适合作为 evidence/audit/repair 视图，而不是主交易存储。

更合理的是：

> **保留 Flink 式并行/增量思想，重写 Reference-First 领域逻辑。**

---

# 3. FAMER Incremental ER：最适合当前需求的部分

FAMER 后续增量工作处理的是：

```text
已有 clustered similarity graph
+
来自已有/新增 source 的一批新 entity
```

两类策略：

### Base incremental

```text
新 entity
 -> 找相似已有 cluster
 -> 加入 cluster
 or
 -> 新建 cluster
```

### Repair incremental

不是只机械把新记录塞入旧簇，而是允许重聚类一部分受影响的历史 graph，以降低插入顺序依赖并纠正已有错误。

这对持续抓取的三源数据非常重要：

- 新品牌/新 reference 上线；
- 某平台字段结构变化；
- reference 词典补充 alias；
- OCR 模型升级；
- 过去 `UNRESOLVED` 的 listing 获得新证据；
- 过去已接受的 reference 被发现解析规则存在 bug。

传统 append-only matcher 的问题是：一旦历史判错，后续数据只会继续依附旧错误。

### 推荐迁移成“局部可重算”的 Resolution Engine

每个 resolution 不能只有最终值：

```text
listing_id -> reference_id
```

还应保存：

```text
decision_version
rule_version
extractor_version
catalog_version
evidence_ids
decision_state
decision_reason
created_at
superseded_by
```

这样当规则版本变化时，不需要全表 1000 万重跑，而是通过 impact index 找到：

```text
受该规则影响的 brand
受该 alias 影响的 ref pattern
由该 OCR model 产生的 evidence
处于某种 conflict reason 的 listing
```

然后只 replay 受影响集合。

这就是 FAMER “recluster a portion of the existing graph” 在本业务最值得直接落地的变体。

---

# 4. 为什么不能原样把 FAMER 当最终匹配器

## 4.1 当前需求不是一般的“相似实体”定义

FAMER 面向通用实体解析，通常需要组合名字、地址、属性等模糊证据。

当前需求却给出了明确等价关系：

```text
same product ⇔ same canonical reference number
```

因此最强基线其实不是复杂模型，而是：

```text
extract reference
-> normalize conservatively
-> validate role + brand
-> exact join
```

只要 reference 证据可靠，继续做文本/图片 pairwise matching 反而增加出错面。

---

## 4.2 Source-consistency 不是当前业务正确的不变量

FAMER/CLIP/SplitMerge 的很多设计特别利用“clean source”概念：一个真实实体簇中不应有来自同一 source 的多个实体。后续研究也扩展到 clean/dirty source 组合，但 source provenance 仍是重要约束维度。

二手电商却可能在同一平台同时存在很多**同一个 reference** 的不同 listing：

```text
腕表之家 listing 1 -> Rolex 126610LN
腕表之家 listing 2 -> Rolex 126610LN
```

按照本需求的定义，它们应该被视为同一商品实体类别，即便它们是不同实物/卖家 listing。

所以：

```text
同源出现多个 listing != 冲突
```

真正的冲突是：

```text
同一个 accepted semantic cluster 出现不同 canonical reference
```

因此必须改成 reference-consistency。

---

## 4.3 Pairwise 模糊边 + 传递性会把一次误判放大

最危险的情况：

```text
Rolex 126610LN  ~  126610LV
126610LV        ~  126610LN 某脏标题
```

如果 soft similarity 被当成“same entity”边，再做 Connected Components，cluster precision 可能比 pair precision 更差。

**本需求宁可留下孤立点，也不能允许一条可疑边污染整个簇。**

---

## 4.4 Fusion/Cluster Representative 不适合丢弃底层证据

FAMER 的增量体系可选择 Fusion，把 cluster 表示成单一 representative，以提高后续处理效率。

当前项目可以有 `ReferenceEntity` 作为代表，但不能只保留融合结果并丢掉 listing/evidence：

```text
ReferenceEntity
  <- listing A (structured ref)
  <- listing B (title ref)
  <- listing C (OCR ref)
```

必须保留所有原始证据，因为：

- 要审计为什么自动合并；
- 要在 extractor/rule 升级时局部重算；
- 要发现某个来源字段系统性污染；
- 要撤销历史错误决策。

所以可以“逻辑融合”，不能“证据销毁”。

---

# 5. 直接可落地方案：RefGraph / Reference-First Entity Resolution

## 5.1 总体架构

```text
┌───────────────────────────────────────────────────────────────┐
│ Sources                                                       │
│ 雷小安 / 腕表之家 / 奢当家 / future sources                  │
└───────────────┬───────────────────────────────────────────────┘
                │ CDC / batch snapshots
                ▼
┌───────────────────────────────────────────────────────────────┐
│ 1. Ingestion + Source Adapter                                 │
│ raw snapshot / schema map / field role / provenance           │
└───────────────┬───────────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────────┐
│ 2. Reference Evidence Extraction                              │
│ structured -> title parser -> OCR (gated) -> optional VLM     │
└───────────────┬───────────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────────┐
│ 3. Conservative Canonicalizer + Brand Reference Catalog       │
│ role validate / grammar validate / exact canonical lookup     │
└───────────────┬───────────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────────┐
│ 4. Decision Gateway                                            │
│ AUTO_ACCEPT / REVIEW / UNRESOLVED / CONFLICT                  │
│ hard conflict has veto; soft evidence cannot overrule ref      │
└───────────────┬───────────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────────┐
│ 5. Resolution Store + Evidence Graph + Audit Log               │
└───────────────┬───────────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────────┐
│ 6. Incremental Local Repair                                    │
│ replay affected refs/brands/rules/models only                  │
└───────────────────────────────────────────────────────────────┘
```

---

# 6. 数据模型：把“事实、候选、证据、决策”分开存

下面是一套可直接落地的关系模型；图数据库不是必需，先用 PostgreSQL/ClickHouse/Iceberg 也能实现 evidence graph 的逻辑关系。

## 6.1 `reference_entity`

```sql
CREATE TABLE reference_entity (
    reference_id        BIGSERIAL PRIMARY KEY,
    canonical_brand_id  BIGINT NOT NULL,
    canonical_reference VARCHAR(128) NOT NULL,
    series              VARCHAR(256),
    model_name          VARCHAR(256),
    status              VARCHAR(32) NOT NULL DEFAULT 'ACTIVE',
    catalog_version     BIGINT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (canonical_brand_id, canonical_reference)
);
```

这个表是真正的 semantic entity registry。

---

## 6.2 `listing_snapshot`

```sql
CREATE TABLE listing_snapshot (
    listing_pk          BIGSERIAL PRIMARY KEY,
    source              VARCHAR(32) NOT NULL,
    source_item_id      VARCHAR(256) NOT NULL,
    snapshot_version    BIGINT NOT NULL,
    title               TEXT,
    raw_payload_uri     TEXT NOT NULL,
    image_manifest_uri  TEXT,
    ingested_at         TIMESTAMPTZ NOT NULL,
    UNIQUE(source, source_item_id, snapshot_version)
);
```

原始 payload 建议 immutable 存 OSS/S3/Iceberg，关系库保存索引。

---

## 6.3 `reference_evidence`

```sql
CREATE TABLE reference_evidence (
    evidence_id          BIGSERIAL PRIMARY KEY,
    listing_pk           BIGINT NOT NULL,
    channel              VARCHAR(32) NOT NULL,
    -- STRUCTURED / TITLE / OCR / CATALOG / VLM / MANUAL

    semantic_role        VARCHAR(64) NOT NULL,
    -- MANUFACTURER_REFERENCE / PLATFORM_SKU / SERIAL / COMPATIBLE_REF ...

    raw_value            TEXT NOT NULL,
    raw_span             TEXT,
    candidate_reference  VARCHAR(128),
    canonical_brand_id   BIGINT,
    canonical_reference  VARCHAR(128),

    role_confidence       DOUBLE PRECISION,
    extraction_confidence DOUBLE PRECISION,
    catalog_validated     BOOLEAN NOT NULL DEFAULT FALSE,

    extractor_name        VARCHAR(64) NOT NULL,
    extractor_version     VARCHAR(64) NOT NULL,
    rule_version          VARCHAR(64) NOT NULL,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

注意这里保存的是 evidence，不是最终事实。

---

## 6.4 `listing_reference_resolution`

```sql
CREATE TABLE listing_reference_resolution (
    listing_pk          BIGINT PRIMARY KEY,
    reference_id        BIGINT,
    state               VARCHAR(32) NOT NULL,
    -- AUTO_ACCEPT / REVIEW / UNRESOLVED / CONFLICT / REJECTED

    decision_reason     VARCHAR(128) NOT NULL,
    decision_version    BIGINT NOT NULL,
    rule_version        VARCHAR(64) NOT NULL,
    catalog_version     BIGINT NOT NULL,
    accepted_evidence_ids BIGINT[],
    conflicting_evidence_ids BIGINT[],
    decided_at          TIMESTAMPTZ NOT NULL,
    reviewer_id         VARCHAR(128)
);
```

一个 listing 只有在 `AUTO_ACCEPT` 时才允许给下游暴露确定的 `reference_id`。

---

## 6.5 `resolution_audit_log`

```sql
CREATE TABLE resolution_audit_log (
    audit_id            BIGSERIAL PRIMARY KEY,
    listing_pk          BIGINT NOT NULL,
    old_state           VARCHAR(32),
    new_state           VARCHAR(32) NOT NULL,
    old_reference_id    BIGINT,
    new_reference_id    BIGINT,
    trigger_type        VARCHAR(64) NOT NULL,
    -- NEW_DATA / RULE_CHANGE / CATALOG_CHANGE / MODEL_CHANGE / MANUAL

    trigger_id          VARCHAR(256),
    decision_version    BIGINT NOT NULL,
    payload             JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

这张表让“历史错误被修复”成为正常操作，而不是灾难性数据库更新。

---

# 7. Reference Canonicalization：宁可少归一，也不能错误碰撞

reference number 常见格式变化：

```text
126610LN
126610 LN
126610-LN
126610ln
```

但不同品牌 reference 的标点、后缀、斜杠、点号可能有真实语义。因此不能使用一个激进的全局规则：

```python
re.sub(r'[^A-Z0-9]', '', ref)
```

这可能把两个不同 reference 错误折叠成同一个 key。

推荐三层规范化：

## Level A：无损 lexical normalize

```text
Unicode NFKC
trim
统一大小写
标准化全角/半角
标准化 dash code point
压缩重复 whitespace
```

## Level B：品牌规则 normalize

只有被品牌语法白名单证明安全时，才：

```text
允许删除某些空格
允许等价 dash
允许已验证 alias
```

## Level C：catalog resolution

必须把结果映射到品牌 reference catalog 中的**唯一 canonical value**。

如果：

```text
normalized token -> 0 candidate
```

则 `UNRESOLVED`；

如果：

```text
normalized token -> >1 candidates
```

则 `REVIEW`；

只有：

```text
normalized token -> exactly 1 validated canonical ref
```

才具备自动接受的必要条件。

### Canonicalization 的最重要测试指标

不是普通准确率，而是：

```text
collision_count =
不同真实 reference 被 normalize 成同一个 canonical key 的数量
```

对 production `AUTO_ACCEPT` 规则，目标应是**已验收集合中 collision = 0**。

---

# 8. Evidence Extractor：按成本与可信度逐层升级

## 8.1 第一层：结构化字段

如果某来源存在明确的 manufacturer reference 字段，并且经过字段角色确认：

```text
structured_ref
 -> canonicalizer
 -> catalog validate
```

这是最便宜、最强的路径。

但上线前必须抽样确认该字段不是：

- 平台货号；
- 店铺 SKU；
- serial number；
- SPU/SKU 混用；
- 兼容型号。

---

## 8.2 第二层：标题抽取

标题中抽 reference 时，不只做 regex，应同时判断**编号角色上下文**。

例：

```text
“劳力士 潜航者 126610LN 全套”
```

可能是当前商品 reference。

但：

```text
“适配劳力士 126610LN 黑色表带”
```

这里的 126610LN 是 compatible reference，不是被售商品 reference。

因此 extractor 至少输出：

```json
{
  "raw": "126610LN",
  "role": "MANUFACTURER_REFERENCE",
  "role_confidence": 0.998,
  "context": "..."
}
```

而不是只输出字符串。

---

## 8.3 第三层：图片 OCR

只对以下记录启动 OCR：

```text
没有可靠结构化/标题 ref
OR
文本 evidence 冲突
OR
人工 review 需要独立证据
```

优先 OCR：

- 表背刻字；
- 保卡；
- 吊牌；
- 证书；
- 清晰型号区域。

OCR 结果仍然只是 candidate：

```text
OCR token
 -> format grammar
 -> brand catalog
 -> cross-evidence check
```

需特别处理：

```text
O ↔ 0
I ↔ 1
B ↔ 8
S ↔ 5
“-” / “.” / “/”
```

不能用模糊纠错后直接 AUTO_ACCEPT 相邻 reference。

---

## 8.4 第四层：VLM / LLM

推荐只做：

- 提出 reference candidate；
- 标注哪个文本 span / 图片区域支持 candidate；
- 判断一个编号更像 manufacturer reference 还是 SKU；
- 对 REVIEW 页面生成证据摘要。

禁止：

```text
VLM: “看起来像 126610LN”
=> AUTO_ACCEPT
```

模型必须输出可验证 candidate，而最终 decision 由 deterministic catalog/rule gateway 控制。

---

# 9. Decision Gateway：从“相似度阈值”升级成有否决权的状态机

推荐决策伪代码：

```python
from enum import Enum

class State(str, Enum):
    AUTO_ACCEPT = "AUTO_ACCEPT"
    REVIEW = "REVIEW"
    UNRESOLVED = "UNRESOLVED"
    CONFLICT = "CONFLICT"


def resolve_listing(listing, evidences, catalog):
    credible = []
    soft = []

    for ev in evidences:
        if ev.semantic_role != "MANUFACTURER_REFERENCE":
            continue

        candidate = conservative_normalize(
            brand=listing.brand,
            raw=ev.raw_value,
        )

        match = catalog.lookup_exact(listing.brand, candidate)

        if match.is_unique and ev.catalog_validated:
            ev.reference_id = match.reference_id

            if ev.channel in {"STRUCTURED", "TITLE", "OCR", "MANUAL"} \
               and passes_channel_policy(ev):
                credible.append(ev)
            else:
                soft.append(ev)

    hard_refs = {ev.reference_id for ev in credible}

    # veto: 两个可信 reference 互相冲突，绝不投票自动选一个
    if len(hard_refs) > 1:
        return Decision(
            state=State.CONFLICT,
            reference_id=None,
            reason="CREDIBLE_REFERENCE_CONFLICT",
        )

    if len(hard_refs) == 1:
        ref_id = next(iter(hard_refs))

        if any(explicitly_contradicts(ev, ref_id) for ev in evidences):
            return Decision(
                state=State.REVIEW,
                reference_id=None,
                reason="CONTRADICTORY_EVIDENCE",
            )

        return Decision(
            state=State.AUTO_ACCEPT,
            reference_id=ref_id,
            reason="UNIQUE_VALIDATED_REFERENCE",
        )

    # 软 evidence 只能形成候选，不能升级为事实
    if soft:
        return Decision(
            state=State.REVIEW,
            reference_id=None,
            reason="SOFT_REFERENCE_CANDIDATE_ONLY",
        )

    return Decision(
        state=State.UNRESOLVED,
        reference_id=None,
        reason="NO_VALIDATED_REFERENCE",
    )
```

### 关键点

不是：

```text
score > 0.99 => match
```

而是：

```text
满足一组允许自动放行的证据条件
AND
没有任何硬冲突
=> AUTO_ACCEPT
```

这能显著减少模型置信度失真造成的 false positive。

---

# 10. 多源“聚类”实际上可以退化成 Canonical Reference Join

只要 Listing 已经被安全解析为：

```text
(reference_id = 12345)
```

那么实体簇就是：

```sql
SELECT reference_id, array_agg(listing_pk)
FROM listing_reference_resolution
WHERE state = 'AUTO_ACCEPT'
GROUP BY reference_id;
```

不需要再跑 generic clustering。

这件事非常重要：

> FAMER 帮我们理解“多源 ER 应如何分层、并行、修复”，但 Spec 给出的 reference 定义让最终聚类问题大幅简化。

也就是说，复杂度应该投入到：

1. reference 是否真的是 manufacturer reference；
2. 是否正确抽取；
3. 是否安全 canonicalize；
4. 是否存在冲突；
5. 历史决策能否被局部修复。

而不是投入到“两个看起来很像的 listing 到底是不是同款”的黑盒分类器。

---

# 11. Flink 增量拓扑：直接对应 FAMER 的可扩展执行思想

若生产数据是持续增量，推荐 Kafka + Flink（或同类流式框架）实现。

```text
Kafka: listing_changes
       │
       ▼
[Source Adapter]
       │
       ▼
[Field Role + Brand Normalize]
       │
       ▼
[Structured/Title Ref Extractor]
       │
       ├── reliable ref ───────────────┐
       │                               │
       └── missing/conflict -> OCR ────┤
                                       ▼
                             [Catalog Exact Lookup]
                                       │
                                       ▼
                               [Decision Gateway]
                              /        |         \
                             /         |          \
                     AUTO_ACCEPT    REVIEW     UNRESOLVED
                         │              │            │
                         ▼              ▼            ▼
                     resolution     review topic   unresolved
                         │
                         ▼
                  audit/change log
```

## 11.1 幂等键

```text
(source, source_item_id, snapshot_version)
```

同一 snapshot 重放不能产生重复 decision。

## 11.2 Exactly-once / 可恢复

Flink checkpoint 只负责计算状态；最终数据库写入仍应使用：

- idempotent upsert；
- decision_version；
- expected previous version / optimistic lock。

避免 checkpoint 恢复导致覆盖更新。

## 11.3 State 不要存整个 reference cluster

热门 reference 可能对应海量 listing，如果全部在一个 keyed state 中维护完整 list，会产生 hot state。

推荐：

```text
Flink state：小状态
- 当前 catalog/version cache
- rule metadata
- per-key counters / compact metadata

Persistent store：大成员集合
- listing -> reference mapping
- evidence rows
```

如果必须按 reference 做聚合，也可将：

```text
(reference_id, shard_id)
```

作为物理分区键；语义实体仍然是一个 reference_id。

---

# 12. Incremental Local Repair：把 FAMER 的局部重聚类改成“影响范围重判”

建议定义 Repair Trigger：

```text
RULE_CHANGED
CATALOG_ALIAS_CHANGED
BRAND_MAPPING_CHANGED
EXTRACTOR_MODEL_CHANGED
OCR_MODEL_CHANGED
MANUAL_CORRECTION
SOURCE_SCHEMA_CHANGED
```

每种 trigger 都能计算受影响集合。

例如 catalog 新增：

```text
Rolex: "126610 LN" -> "126610LN"
```

只需要找：

```sql
SELECT DISTINCT listing_pk
FROM reference_evidence
WHERE canonical_brand_id = :rolex
  AND candidate_reference IN ('126610 LN', '126610LN')
  AND rule_version < :new_version;
```

然后 replay 这些 listing。

### Repair 的安全状态迁移

允许：

```text
UNRESOLVED -> AUTO_ACCEPT
REVIEW     -> AUTO_ACCEPT
AUTO_ACCEPT -> REVIEW
AUTO_ACCEPT -> CONFLICT
AUTO_ACCEPT(ref A) -> AUTO_ACCEPT(ref B)  # 需要强审计/人工或明确规则修复
```

特别是最后一种不能静默覆盖，必须记录：

- old ref；
- new ref；
- 什么规则触发；
- 哪些 evidence 发生变化；
- 下游哪些聚合需要补偿更新。

这就是把图 repair 真正变成生产级“可撤销决策”。

---

# 13. Candidate Retrieval：只服务于“reference 缺失尾部”

当结构化字段/标题/OCR 都没有可靠 reference 时，才需要模糊检索。

候选可以来自：

```text
BM25(title)
text embedding
image embedding
brand + series + attributes
ANN top-K
```

但输出必须是：

```text
Candidate Reference IDs
```

而不是：

```text
candidate listing pairs
```

更推荐：

```text
listing -> top-K canonical reference entities
```

这会把候选空间从 N 个 listing 压到通常更小、更稳定的 reference catalog。

后续只做：

```text
candidate evidence verification
```

而不是 pairwise “像不像”。

---

# 14. Hard Negative 设计：专门围绕“相邻 reference”构造

几百条人工黄金标签不应随机均匀抽样，应该优先覆盖最容易导致 false positive 的边界。

推荐 hard negatives：

## 14.1 同系列相邻型号

```text
reference 只差 1 个字符
reference 只差后缀
不同材质后缀
不同尺寸
不同盘面/表圈
新老代相邻型号
```

## 14.2 编号角色混淆

```text
manufacturer ref vs platform SKU
manufacturer ref vs seller SKU
manufacturer ref vs serial number
当前商品 ref vs compatible product ref
```

## 14.3 OCR 混淆

```text
0/O
1/I/l
5/S
8/B
```

## 14.4 文本/图片“极像但不同 ref”

这类样本是视觉模型和 embedding 最容易自信误判的区域。

## 14.5 跨来源字段污染

例如某平台的“型号”字段实际上存系列名或内部款号。

这些样本比随机 easy negatives 更能压低实际 false positive。

---

# 15. 黄金标签怎么花：不要拿几百对去证明“无限接近 100% precision”

几百对标签足够做：

- 字段角色验收；
- reference extraction error taxonomy；
- hard negative 调参；
- channel 放行策略；
- 人工 review 规范；
- active learning 启动；
- 分 source/brand 的初始 sanity check。

但如果要声称极高 precision，几百个零错误样本在统计上并不强。

如果抽查 `n` 个自动接受样本且观察到 0 个 false positive，则 95% 单侧上界近似：

```text
FP_rate_upper = 1 - 0.05^(1/n)
```

当 `n=300` 时，上界约 1%，也就是说只能支持 precision 下界约 99%，远不能统计证明 99.9%+。

若想在 0 error 条件下把 95% 上界压到约 0.1%，需要接近 3000 个独立验收样本。

因此系统不能依赖一句“模型在几百对上 100% precision”作为安全保证。

正确策略是：

> **用 deterministic reference semantics 把错误空间结构性收窄，再用持续抽检统计验证。**

---

# 16. 评测指标：F1 不能做主指标

## 16.1 最重要：Auto-Accept Precision

```text
P(auto-accepted pair/group truly same canonical ref)
```

并且要同时报告置信下界，而不是只报点估计。

## 16.2 Auto-Accept Coverage

```text
AUTO_ACCEPT listing / all listing
```

precision 达标后再优化 coverage。

## 16.3 Reference Extraction Precision / Recall

按 channel 分开：

```text
structured
标题
OCR
VLM
```

不能混成一个总分，否则低质量 channel 会被高质量 channel 掩盖。

## 16.4 Canonicalization Collision Rate

```text
不同真实 ref -> 同 canonical ref
```

这是最危险的系统性错误之一。

## 16.5 Conflict Rate

```text
credible evidence 指向多个 reference 的比例
```

按 source / brand 追踪，可以发现上游字段质量问题。

## 16.6 Review Yield

人工 review 中：

```text
真正发现系统风险的比例
```

避免 review 队列充满 easy cases。

## 16.7 Drift Metrics

分来源、品牌、时间窗口：

```text
extract rate
unknown ref rate
conflict rate
auto-accept rate
manual overturn rate
```

任何突变都应触发 threshold/rule freeze，而不是自动放宽。

---

# 17. 人工审核界面应该展示什么

不要给审核员一个黑盒：

```text
模型说 0.984 MATCH，是否确认？
```

应该展示：

```text
商品 A / 当前 listing
- 来源 / 原始 ID
- 原始标题
- 原始结构化 reference 字段
- reference 候选及其原文位置
- 每张图片 OCR 结果和区域
- canonicalization 过程
- catalog 命中详情
- soft candidate refs
- 冲突证据

候选 canonical reference
- brand
- canonical ref
- series/model
- known aliases
- 相邻 reference hard negatives

系统原因码
- CREDIBLE_REFERENCE_CONFLICT
- PLATFORM_SKU_SUSPECTED
- OCR_ONLY
- UNKNOWN_CATALOG_REFERENCE
- AMBIGUOUS_NORMALIZATION
```

审核结果要反哺：

- alias；
- field-role rule；
- OCR confusion rule；
- hard negative set；
- review prioritization。

---

# 18. 存储与基础设施建议

一个不依赖单一厂商的落地组合：

## Raw / History

```text
OSS/S3 + Iceberg/Delta/Hudi
```

用途：

- 原始 payload；
- snapshot；
- replay；
- rule/model 升级后的离线重算。

## Streaming

```text
Kafka + Flink
```

用途：

- 持续增量；
- source adapter；
- extractor orchestration；
- decision；
- repair trigger。

## Transactional metadata

```text
PostgreSQL
```

用途：

- canonical reference catalog；
- resolution state；
- audit；
- manual review。

## Large analytical evidence

```text
ClickHouse / Iceberg
```

用途：

- 全量 evidence；
- 指标；
- 漂移分析；
- 大范围 impact query。

## Images

```text
OSS/S3
```

只保存 URI/hash/metadata 到结构化库。

## Vector Retrieval（可选）

```text
FAISS / Milvus / pgvector / OpenSearch kNN
```

只服务 REVIEW candidate retrieval，不作为自动事实库。

## Graph DB（可选，不是第一阶段必需）

如果后期 review/impact/debug 强依赖图查询，再增加 Neo4j/JanusGraph 等；第一阶段完全可以用关系表表达边。

---

# 19. 计算复杂度：为什么 Reference-First 才适合 1000 万规模

若直接对三个来源做 pairwise 商品匹配，最坏会接近：

```text
O(N^2)
```

即便 Blocking 后仍有大量候选 pair。

Reference-First 主路径：

```text
每条 listing：
extract -> normalize -> hash/index exact lookup
```

总体接近：

```text
O(N)
```

只有比例 `p` 的 unresolved tail 进入 top-K retrieval：

```text
O(N * p * K)
```

其中 `K` 可以严格限制。

OCR/VLM 成本也可以用 gating 控制：

```text
C_total =
N * C_text
+ N * p_ocr * C_ocr
+ N * p_vlm * C_vlm
```

目标不是让所有 1000 万记录跑最贵模型，而是不断降低：

```text
p_ocr
p_vlm
p_review
```

同时不牺牲 auto-accept precision。

---

# 20. 对热门品牌/型号的数据倾斜：借鉴 FAMER Block Split，但不要制造大 pair block

FAMER 的 Block Split 是为巨大 block 并行比较。

在 RefGraph 中，热门 reference 也可能有几十万 listing，但它们已经共享一个确定 `reference_id`，不需要在这些 listing 间两两比较。

因此：

```text
不要：
Rolex 126610LN block 内全 pair compare

要：
每条 listing 独立 resolve -> reference_id
```

只有统计/聚合时才处理大 group。

如果一个 reference 成为流式 hot key：

```text
physical key = (reference_id, hash(listing_id) % S)
semantic key = reference_id
```

可用 shard 分散负载。

这比 Block Split 更适合本业务，因为根本不产生 quadratic comparison。

---

# 21. 图片怎么用才不会破坏 precision-first

图片最适合三个角色：

## A. OCR evidence

从表背/保卡/吊牌提取 reference。

## B. Conflict detector

例如文本说某 reference，但图片关键属性明显属于另一个变体，则降低自动放行资格，送 REVIEW。

## C. Candidate retrieval

没有 reference 时，从 reference catalog/已验真图片中找 top-K 候选。

### 图片不应该做的事

```text
image_similarity > threshold
=> same product
```

腕表同系列不同 reference 的视觉相似度可能极高，所以视觉只能“提出/否决候选”，不能覆盖 reference identity。

---

# 22. 失败模式与安全策略

| 失败模式 | 危险 | 推荐处理 |
|---|---|---|
| 平台 SKU 被当 reference | 极高 | field-role classifier + source allowlist；不确定则拒识 |
| 标题出现“适配某型号” | 极高 | context role extraction；compatible ref 禁止 AUTO_ACCEPT |
| 同系列相邻 reference | 极高 | exact canonical equality；相似度永不作为等价条件 |
| OCR 把 0/O、1/I 混淆 | 高 | 生成多个 candidate，若非唯一 catalog hit 则 REVIEW |
| 激进删除标点导致 ref collision | 极高 | brand-specific conservative normalize + collision test |
| 图像外观高度相似 | 高 | 只做 candidate/反证，不做最终匹配 |
| LLM 自信 hallucinate reference | 极高 | candidate 必须回指原文/OCR/catalog；无法验证则不接受 |
| 一条错边产生传递闭包 | 极高 | final group by reference_id，不做 soft-edge equivalence closure |
| 上游字段语义改版 | 高 | schema/version drift monitor + source-specific freeze |
| 历史规则 bug | 高 | versioned evidence + local repair + reversible audit |

---

# 23. 逐阶段上线方案

## Phase 0：数据审计 + Canonical Reference Registry

目标：先知道三个来源到底有哪些字段、哪些字段可信。

产物：

```text
brand registry
reference catalog
source field-role map
reference grammar/rule pack
hard-negative seed set
```

验收：

- 每个来源抽样确认 reference 字段语义；
- 不允许未知字段直接进入 auto-accept。

---

## Phase 1：纯文本 deterministic MVP

只支持：

```text
structured reference
+ title reference
+ exact catalog validation
```

暂时不做图片、不做 embedding、不做 LLM。

这一步通常就能覆盖最干净的数据，而且最容易获得很高 precision。

上线条件：

- 黄金集中 auto-accept 无已知 FP；
- canonicalization collision = 0；
- 每个 accept 都有可解释 reason/evidence。

---

## Phase 2：OCR 独立证据

只处理：

```text
text unresolved
text conflict
```

新增：

- image quality gate；
- OCR candidate extraction；
- catalog verify；
- OCR confusion handling。

默认策略：

```text
OCR-only candidate -> REVIEW
```

等数据证明某些图片类型/品牌 OCR 足够可靠后，再为特定路径开放自动放行，而不是全局开放。

---

## Phase 3：Soft Candidate Retrieval

增加：

```text
BM25 / embedding / image retrieval
```

目标只提升 review efficiency / extraction recall。

任何 soft retrieval 命中都不改变：

```text
AUTO_ACCEPT 必须由 reference evidence policy 决定
```

---

## Phase 4：Flink 增量 + Local Repair

将离线批处理变成：

```text
new listing -> incremental resolution
rule/catalog update -> impact replay
```

实现：

- event version；
- idempotent upsert；
- audit log；
- replay topic；
- repair dashboard。

---

## Phase 5：主动学习与高风险 review

将有限人工预算优先给：

```text
reference 冲突
相邻型号
角色不清晰编号
OCR ambiguity
新 source / 新 brand drift
人工 overturn 高发规则
```

而不是随机标注大量 easy negatives。

---

# 24. 推荐的 API 边界

## 24.1 单 listing 解析

```http
POST /v1/reference/resolve
```

输入：

```json
{
  "source": "watch_home",
  "source_item_id": "123",
  "title": "...",
  "attributes": {},
  "images": []
}
```

输出：

```json
{
  "state": "AUTO_ACCEPT",
  "reference": {
    "brand_id": 10,
    "reference_id": 987,
    "canonical_reference": "126610LN"
  },
  "reason": "UNIQUE_VALIDATED_REFERENCE",
  "decision_version": 42,
  "evidence": [
    {
      "channel": "TITLE",
      "raw": "126610LN",
      "role": "MANUFACTURER_REFERENCE",
      "catalog_validated": true
    }
  ]
}
```

## 24.2 查询一个 reference 的跨源 listing

```http
GET /v1/references/{reference_id}/listings
```

只默认返回：

```text
state=AUTO_ACCEPT
```

REVIEW/UNRESOLVED 不应该混入事实结果。

## 24.3 Repair

```http
POST /v1/repair/replay
```

输入可以按：

```text
brand
reference
rule_version
extractor_version
source
listing ids
```

指定影响范围。

---

# 25. 与 FAMER 的逐项映射

| FAMER 原设计 | 当前需求建议 | 是否保留 |
|---|---|---|
| 多源 GraphCollection | 统一 listing schema + provenance | 保留思想 |
| graphLabel 标识来源 | `source` + field-role provenance | 保留并增强 |
| Blocking | exact ref key / brand-aware candidate retrieval | 保留但主路径简化 |
| Block Split | hot-key/sharded state；避免大 pair block | 改造 |
| Pairwise similarity | evidence/candidate score | 仅软路径保留 |
| Match classification | Decision Gateway 状态机 | 重写 |
| Similarity Graph | Evidence Graph | 强烈保留思想 |
| Connected Components | canonical reference group-by | 不用于最终事实 |
| Source consistency | Reference consistency | 核心替换 |
| SplitMerge | reference conflict split + local repair | 保留思想 |
| CLIP link priority | evidence trust priority / conflict ordering | 保留思想 |
| Cluster representative | Canonical Reference Entity | 保留，但不丢 evidence |
| Incremental clustering | incremental resolution | 强烈保留 |
| Reclustering partial graph | impact-based replay / repair | 强烈保留 |
| Fusion | 逻辑汇总 | 可保留，不物理删除原始证据 |
| Flink distributed execution | Kafka + Flink stream/batch | 推荐 |
| Gradoop/Gelly | 可选，非主路径依赖 | 不必直接采用 |

---

# 26. 与现有调研方向的组合位置

FAMER 解决的是“多源、图、聚类、并行、增量 repair”这一工程骨架，它适合与已有几类方法组合，而不是互相替代：

```text
Reference extraction / normalization
        │
        ▼
FAMER-inspired distributed candidate/evidence architecture
        │
        ▼
Reference-consistency Decision Gateway
        │
        ▼
Selective / abstention policy
        │
        ▼
Transitive conflict / graph repair audit
```

其中最终真值仍然只有一个：

```text
canonical reference
```

这样可以避免“模型 A 认为像、模型 B 认为很像、图片也像，所以就当同款”的证据堆叠陷阱。

---

# 27. 最终推荐架构原则

可以把整个方案压缩成 10 条 production rule：

1. **不要把问题建模成 listing-listing 全量匹配；建模成 listing → canonical reference entity。**
2. **reference 是 identity key；文本/图片相似度只是 search/evidence。**
3. **先做字段角色识别，防止 platform SKU / compatible ref 冒充 manufacturer reference。**
4. **canonicalization 必须保守、品牌感知、可回溯；禁止激进字符串折叠。**
5. **AUTO_ACCEPT 要由允许的强证据路径触发，不由一个通用概率阈值触发。**
6. **任何可信 reference 冲突都有 veto 权，直接 REVIEW/CONFLICT。**
7. **同款集合按 reference_id group-by，不做 soft similarity 的传递闭包。**
8. **保留完整 evidence graph / audit log，任何历史决策都能解释、撤销、重算。**
9. **采用 FAMER 的 incremental repair 思想，只 replay 受规则/模型/catalog 变化影响的局部数据。**
10. **precision 达标之后再优化 coverage；OCR/VLM/ANN 逐层只服务于 unresolved tail。**

---

# 28. 最终方案判断

**FAMER 值得参考，但不建议原样直接落地。**

它最适合贡献：

- 多源统一数据模型；
- Blocking / 候选压缩；
- 显式相似/证据图；
- 分布式执行；
- cluster repair；
- incremental ER。

而当前业务应把它的通用“相似实体聚类”改造成更严格的：

> **Reference-First Evidence Resolution + Reference-Consistency Repair**。

这样既能利用 FAMER 面向大规模多源增量数据的成熟架构思想，又不会继承“模糊边被聚类放大”的 precision 风险。

对于雷小安 × 腕表之家 × 奢当家这类 100 万–1000 万级持续增量场景，第一版完全可以先上线：

```text
Source Adapter
+ Structured/Title Reference Extractor
+ Conservative Canonicalizer
+ Canonical Reference Catalog
+ Exact Join
+ Conflict Veto
+ Audit Log
```

然后再按 unresolved tail 的实际比例逐步加：

```text
OCR
→ ANN candidate retrieval
→ VLM/LLM assistant
→ active review
```

整个演进过程中都不改变最终规则：

> **只有经过可审计证据证明的 canonical reference 才能产生自动同款事实；无法证明就拒识。**

这比把一个 generic entity matcher 的 F1 提高几个点，更符合 Spec 中“绝对不能误匹配”的核心目标。

---

# 29. 参考资料

1. FAMER 项目主页  
   https://dbs.uni-leipzig.de/research/projects/famer

2. Saeedi, Nentwig, Peukert, Rahm. **Scalable Matching and Clustering of Entities with FAMER**. CSIMQ, 2018.  
   https://dbs.uni-leipzig.de/research/publications/scalable-matching-and-clustering-of-entities-with-famer

3. Saeedi, Peukert, Rahm. **Using Link Features for Entity Clustering in Knowledge Graphs** (CLIP, ESWC 2018 Best Research Paper).  
   https://dbs.uni-leipzig.de/research/publications/using-link-features-for-entity-clustering-in-knowledge-graphs

4. Saeedi, Peukert, Rahm. **Incremental Multi-source Entity Resolution for Knowledge Graph Completion**. ESWC 2020.  
   https://dbs.uni-leipzig.de/research/publications/incremental-multi-source-entity-resolution-for-knowledge-graph-completion

5. FAMER GitLab  
   https://git.informatik.uni-leipzig.de/dbs/FAMER

6. FAMER PreProcessing Configuration Wiki（Gradoop GraphCollection、graphLabel、property rename/combine）  
   https://git.informatik.uni-leipzig.de/dbs/FAMER/-/wikis/Home/Configuration/PreProcessing-Configuration-(JSON)
