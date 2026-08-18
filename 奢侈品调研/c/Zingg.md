# Zingg：大规模实体解析架构分析与奢侈品商品匹配落地方案

> 分析者：c  
> 日期：2026-08-18  
> 项目：Zingg（`zinggAI/zingg`）  
> 关联需求：Notion Spec `3bf7b2a8538b812fba00fb258024ff31`

## 1. 结论先行

Zingg 很适合作为本需求的**大规模候选生成、灰区匹配、主动学习和人工审核辅助层**，但不应该直接承担“最终自动合并”的唯一判定职责。

原因是需求的关键约束不是一般意义上的 F1，而是：

1. 数据规模约 100 万～1000 万，不能做全量 O(N²) 两两比较；
2. 多来源字段格式不同、字段缺失，reference number 还可能藏在 title 中；
3. 同一 reference number 代表同一商品；
4. **绝对不能误匹配，可以牺牲召回率。**

Zingg 默认是概率式实体解析：通过 blocking 缩小候选集合，用字段相似度模型为候选对打分，再按阈值和聚类得到实体簇。它的 `EXACT` match type 只是“给分类器一个很强的 exact-match 特征”，官方文档明确说明：即使某个 `EXACT` 字段不同，记录仍然可能因为其他字段而被模型判为 match。因此，直接使用“Zingg score >= threshold => merge”不能满足“已知不同 reference number 绝不能合并”的业务约束。

建议采用如下双通道架构：

```text
                         ┌──────────────────────────┐
Source A / B / C ... --->│ Bronze / Raw Product    │
                         └────────────┬─────────────┘
                                      │
                                      v
                         ┌──────────────────────────┐
                         │ Normalize + Ref Extract  │
                         │ brand / title / ref      │
                         └────────────┬─────────────┘
                                      │
                         ┌────────────┴─────────────┐
                         │                          │
                         v                          v
              ┌────────────────────┐     ┌──────────────────────┐
              │ GOLD deterministic │     │ GRAY uncertain lane  │
              │ identity-key lane  │     │ Zingg blocking/model │
              └─────────┬──────────┘     └──────────┬───────────┘
                        │                           │
                        v                           v
                  AUTO_MATCH                 CANDIDATE / REVIEW
                        │                           │
                        └────────────┬──────────────┘
                                     v
                         ┌──────────────────────────┐
                         │ canonical_product /     │
                         │ match_edge / audit log  │
                         └──────────────────────────┘
```

核心原则是：**确定性 evidence 决定自动合并；Zingg 只扩大召回、排序灰区候选，不允许覆盖硬冲突。**

---

## 2. 为什么选 Zingg

在《奢侈品文章调研.md》的候选项目中，Zingg 是一个直接面向 Entity Resolution / Record Linkage 的开源框架，并且与当前 Spec 的工程问题高度重合：

- 面向百万级以上数据而不是单机 pairwise matching；
- 原生支持 Spark 分布式执行；
- 同时有 `match`（单数据集去重）与 `link`（多数据源关联）流程；
- 使用 blocking 避免全量笛卡尔积；
- 支持主动学习，人只需标注少量困难样本；
- 支持 `EXACT`、`FUZZY`、`TEXT`、`NUMERIC`、`NULL_OR_BLANK` 等字段匹配特征；
- 模型、训练数据、输出均可持久化；
- Enterprise 版本还提供增量 identity graph、稳定 Zingg ID、blocking 验证、解释与 diff 等治理能力。

但本需求比典型 ER 更严格：**precision 优先级远高于 recall，且 reference number 是强业务身份键。** 因此最佳方式不是照搬 Zingg，而是利用其可扩展候选生成和学习能力，并在其前后加业务硬约束。

---

## 3. Zingg 技术实现与架构拆解

### 3.1 运行时：Spark 分布式数据处理

Zingg 的 Python 程序本质上是 PySpark 程序，CLI / Python API 最终驱动 Spark 执行。官方示例通过：

```python
args = Arguments()
args.setFieldDefinition(fieldDefs)
args.setModelId("100")
args.setZinggDir("models")
args.setNumPartitions(4)
args.setData(inputPipe)
args.setOutput(outputPipe)

options = ClientOptions([ClientOptions.PHASE, "findTrainingData"])
zingg = Zingg(args, options)
zingg.initAndExecute()
```

把字段定义、输入输出 Pipe、模型目录、分区数和执行 phase 组合起来。

对当前 100 万～1000 万商品规模而言，这个设计的价值是计算能以 Spark partition/shuffle 的方式扩展，而不是在一个 Python 进程里维护 N×N 相似度矩阵。

### 3.2 Blocking：先找“值得比较”的候选，而不是两两比较

Zingg 在分类前先做 blocking。官方 glossary 给出的目标是把实际候选比较空间压缩到全量 pair 空间的极小比例（文档给出的典型描述为 0.05%～1%）。

这解决了商品匹配最重要的规模问题：

```text
10,000,000 records
全量 pair ≈ 5e13
                 ↓ blocking
有限候选对
                 ↓ similarity model
match / no-match
```

Zingg 的 `train` phase 会从人工标注数据中同时学习 blocking 和 similarity model；Enterprise 还有 `verifyBlocking`，用于检查已知正样本是否被 blocking 放进同一个候选空间。

对于奢侈品场景，可以进一步比通用 blocking 更保守：

- 第一层必须先按 canonical brand 分区；
- reference 已知时优先走 identity-key exact lookup，不进入模型候选；
- reference 缺失时再对同 brand、同 product family/category 的数据执行 Zingg blocking；
- 对每条记录限制 top-k 候选，避免热门标题形成超大 block。

### 3.3 Match Type：字段相似度是特征，不等于业务规则

Zingg 支持多种 match type，例如：

- `EXACT`：完全一致时产生强 exact 信号；
- `FUZZY`：容忍拼写、缩写、格式差异；
- `TEXT`：适合长文本的词重叠；
- `NUMERIC`：抽取数字比较；
- `NUMERIC_WITH_UNITS`：适合带单位的商品规格；
- `NULL_OR_BLANK`：显式建模缺失值；
- `DONT_USE`：字段保留在输出中但不参与匹配。

一个字段还能配置多个 match type。

**这里有一个对本需求非常重要的实现细节：**Zingg 官方对 `EXACT` 的定义不是硬约束，而是“给 classifier 提供 exact-match signal”。也就是说：

```text
reference A != reference B
```

即使 reference 被配置成 `EXACT`，概率模型仍可能因为 title、brand、description 等其他字段很像而输出 match。

所以 `reference number 不同 => NO_MATCH` 必须放在 Zingg 模型之外做 hard veto，不能只依赖字段权重。

### 3.4 主动学习闭环

社区版核心 phase 是：

```text
findTrainingData
      ↓
label
      ↓
train
      ↓
match / link
```

其中：

- `findTrainingData`：扫描数据并挑选信息量高、值得人工判断的候选对；
- `label`：人工标成 Match / No Match / Uncertain；
- `train`：从标注数据构建 blocking 和 similarity 模型并持久化；
- `match`：对一个数据集做实体聚类；
- `link`：关联两个或更多独立来源的数据，输出仍保留来源信息。

这比随机标注大量 pair 更适合当前项目，因为真正难的是“长得非常像但不是同一 reference”的 hard negative。

奢侈品商品匹配的训练集应刻意强化这类样本：

```text
同 brand + title 极相似 + reference 不同          => NO_MATCH
同 brand + 同 reference + title 格式差异很大      => MATCH
同 brand + reference 一边缺失 + title 高相似      => UNCERTAIN / REVIEW
不同 brand + 型号数字相同                         => NO_MATCH
```

### 3.5 概率式匹配与聚类

Zingg 默认流程会学习不同字段的相似度权重，给候选 pair 产生匹配分数，再把达到阈值的关系转为 cluster。文档还描述了 transitive closure / graph clustering：例如 A-B、B-C 形成匹配关系后，可形成同一实体簇。

这个机制对客户去重很有价值，但在“绝不能错并”的商品匹配里存在额外风险：

```text
A(ref=123) --match--> B(ref missing) --match--> C(ref=456)
```

若仅按传递闭包，A 与 C 可能通过 B 被连接，即使两端存在明确 reference 冲突。

因此本项目必须在形成 canonical cluster 前增加 **cluster invariant**：

> 一个自动合并的商品 cluster 内，不允许同时出现两个不同的已验证 `(brand_id, ref_norm)`。

任何违反 invariant 的候选簇必须 quarantine，而不是自动闭包。

### 3.6 Community 与 Enterprise 的相关差异

从本需求角度，重要差异包括：

- Community 的 `Z Cluster` 不是跨运行稳定 ID；
- Enterprise 有持久化 `Zingg ID` 和 identity graph；
- Enterprise 有 `runIncremental`，可把新增/变更记录并入已有 identity graph，而不全量重跑；
- Enterprise 有 `verifyBlocking`、`explainOutput`、`diff` 等治理能力；
- Enterprise 还有 deterministic matching 能力。

即使不采购 Enterprise，也可以用 Community 版完成灰区候选匹配；稳定 canonical ID、增量 identity index 和确定性 reference 合并建议直接由我们自己的数据层实现，这样也不会把最关键的业务正确性绑定到模型或商业版本。

---

## 4. 面向 Spec 的直接落地架构

## 4.1 数据分层

建议至少维护四层：

```text
bronze_product        原始来源数据，完全不改写
       ↓
normalized_product    标准化品牌/标题/reference，并保留解析证据
       ↓
match_edge            每一条跨源匹配/拒绝/待审边，以及判定证据
       ↓
canonical_product     最终稳定商品实体
```

核心表可以设计为：

```sql
normalized_product(
  source                 string,
  source_product_id      string,
  brand_raw              string,
  brand_id               string,
  title_raw              string,
  title_norm             string,
  ref_raw                string,
  ref_norm               string,
  ref_origin             string,   -- STRUCTURED / TITLE / NONE
  ref_parser_version     string,
  ref_confidence         double,
  attributes_json        string,
  ingested_at            timestamp
)
```

```sql
identity_index(
  brand_id               string,
  ref_norm               string,
  canonical_product_id   string,
  state                  string,   -- ACTIVE / QUARANTINE
  first_seen_at          timestamp,
  last_seen_at           timestamp,
  rule_version           string
)
```

```sql
match_edge(
  left_source            string,
  left_id                string,
  right_source           string,
  right_id               string,
  decision               string,   -- AUTO_MATCH / NO_MATCH / REVIEW
  decision_reason        string,
  evidence_json          string,
  model_score            double,
  rule_version           string,
  created_at             timestamp
)
```

所有自动决策都保存 evidence 与 rule version，后续才可以解释和回滚。

---

## 4.2 Brand 先 canonicalize

reference number 通常只有在品牌命名空间内才安全。例如纯数字/短型号可能跨品牌重复。

因此第一身份键不是 `ref_norm`，而是：

```text
identity_key = (canonical_brand_id, ref_norm)
```

品牌标准化至少包括：

```text
"Louis Vuitton" / "LOUIS VUITTON" / "LV" -> louis_vuitton
"Saint Laurent" / "YSL"                  -> saint_laurent
```

但 alias 映射本身也必须版本化。不能因为 fuzzy brand similarity 就直接跨品牌自动匹配。

---

## 4.3 Reference Number 抽取：把“title 里可能有 reference”变成结构化证据

建议采用“两级 parser”：

### 一级：结构化字段

如果来源直接提供 reference number：

1. Unicode NFKC；
2. trim；
3. 大小写标准化；
4. 去除确认无语义的显示空格；
5. **保留 leading zero**；
6. 不默认删除 `-`、`/`、`.` 等符号，除非品牌规则确认它们只是展示格式；
7. 通过 brand-specific regex/validator 验证。

### 二级：title 提取

只有在结构化 reference 缺失时，才从 title 做提取：

```text
brand_id
  ↓
select brand-specific parser
  ↓
regex/token candidates
  ↓
validator
  ↓
unique candidate ?
    yes -> ref_origin=TITLE
    no  -> ref_norm=NULL + REVIEW
```

不要使用一个“全品牌通用的任意数字串 regex”，因为尺寸、年份、容量、价格、系列号都可能伪装成型号。

推荐每个品牌 parser 输出：

```json
{
  "ref_norm": "M45947",
  "origin": "TITLE",
  "parser": "louis_vuitton_v3",
  "confidence": 0.999,
  "span": [18, 24],
  "pattern_id": "LV_ALPHA_NUM_01"
}
```

这让匹配证据可审核、可回放。

---

## 4.4 三态而不是二态：AUTO_MATCH / NO_MATCH / REVIEW

为满足 precision-first，不要强迫所有 pair 都得到 match/no-match。

推荐规则矩阵：

| 条件 | 决策 |
|---|---|
| canonical brand 相同，双方结构化 ref 均有效且 `ref_norm` 完全一致 | `AUTO_MATCH` |
| canonical brand 相同，一边结构化 ref，一边 title 高置信 parser 抽取，且完全一致、无 collision | `AUTO_MATCH` |
| canonical brand 相同，双方都仅由 title 抽取到相同 ref | `REVIEW`，初期不要自动合并 |
| 双方都有有效 ref，但 `ref_norm` 不同 | `NO_MATCH`，硬否决 |
| canonical brand 明确不同 | `NO_MATCH` |
| ref 缺失，仅 Zingg 高分 | `REVIEW` |
| ref 有冲突，即使 Zingg=0.99999 | `NO_MATCH` |

这直接把“允许牺牲召回率”转化为系统行为。

---

## 4.5 Golden lane：确定性 exact join

大部分拥有可靠 reference 的记录根本不需要进入 ML。

PySpark 示意：

```python
from pyspark.sql import functions as F

valid = products.filter(
    F.col("brand_id").isNotNull() &
    F.col("ref_norm").isNotNull()
)

left = valid.alias("l")
right = valid.alias("r")

auto = (
    left.join(
        right,
        (F.col("l.brand_id") == F.col("r.brand_id")) &
        (F.col("l.ref_norm") == F.col("r.ref_norm")) &
        (F.col("l.source") != F.col("r.source")),
        "inner"
    )
    .filter(
        (F.col("l.ref_origin") == "STRUCTURED") |
        (F.col("r.ref_origin") == "STRUCTURED")
    )
)
```

生产实现还要补三类 guard：

1. `(source, brand_id, ref_norm)` 内部重复/冲突检测；
2. 一个候选 cluster 中出现多个有效 ref 时 quarantine；
3. brand parser / normalization 版本变更后重新验证受影响 edge。

更高效的线上/增量方式不是 self join，而是直接查 `identity_index`：

```text
new product
   ↓ normalize/extract
(brand_id, ref_norm)
   ↓ keyed lookup
found     -> attach existing canonical_product_id
not found -> create canonical_product_id / gray lane
```

这接近 O(N) 数据扫描 + keyed lookup，远小于 O(N²)。

---

## 5. Zingg 在本方案中的正确位置

### 5.1 只处理 Gray lane

Gray lane 包括：

- 两边都没有 reference；
- 只有一边有 reference，另一边 title parser 无法确定；
- title/description 很像，但证据不足以自动合并；
- parser 新规则上线前需要发现潜在漏匹配样本。

### 5.2 推荐字段配置

概念配置可为：

```python
brand = FieldDefinition("brand_id", "string", MatchType.EXACT)
title = FieldDefinition("title_norm", "string", MatchType.FUZZY)
desc = FieldDefinition("description_norm", "string", MatchType.TEXT)
ref = FieldDefinition("ref_norm", "string", MatchType.EXACT)
source_id = FieldDefinition("source_product_id", "string", MatchType.DONT_USE)
```

对于高缺失字段，可再结合 `NULL_OR_BLANK`。

但这里的 `ref EXACT` 只是辅助模型区分 gray 样本；**最终 hard veto 仍在外层 rule engine。**

### 5.3 Hard negative 比普通负样本重要

训练集不要让模型只看到“明显不同”的负样本。真正决定线上 precision 的是 near-duplicate hard negatives：

```text
Brand: Rolex
Title A: Submariner Date 41mm Black
Title B: Submariner Date 41mm Black
Ref A: 126610LN
Ref B: 126610LV
=> NO_MATCH
```

这类数据能训练模型在 reference 缺失时更加谨慎，也能用于评估 blocking 是否错误地把近似产品全部聚到一起。

### 5.4 模型输出只变成候选，不直接变成 canonical identity

建议输出策略：

```text
Zingg pair score
  >= review_high_threshold  -> REVIEW_HIGH
  >= review_low_threshold   -> REVIEW_LOW
  else                      -> ignore
```

不要：

```text
score >= 0.95 -> AUTO_MATCH
```

除非未来某一种极窄 evidence pattern 已经在独立验证集、对抗集和生产 shadow 流量上长期保持 0 false positive，并且即使提升也仍通过 reference conflict veto。

---

## 6. 防止“传递误并”的 Cluster Invariant

Zingg 的图聚类能力很强，但本需求需要额外约束 cluster。

对每个 canonical cluster 维护：

```text
validated_identity_keys = distinct(brand_id, ref_norm)
```

自动 cluster 必须满足：

```text
count(validated_identity_keys) <= 1
```

如果发现：

```text
(louis_vuitton, M45947)
(louis_vuitton, M45948)
```

同时出现在一个 cluster，即使它们通过多个缺失 ref 的中间节点相连，也必须：

1. 阻止 merge；
2. 标记冲突 edge；
3. 把相关 gray nodes 送人工审核；
4. 保存模型分数和路径用于调试。

这样可以阻断最危险的 transitive false merge。

---

## 7. 100 万～1000 万规模下的复杂度设计

建议各阶段复杂度如下：

### 7.1 Normalize / Reference Extract

每条记录独立处理：

```text
O(N)
```

适合 Spark map/UDF（优先普通 Spark SQL expression；品牌 parser 复杂时再用 Pandas UDF）。

### 7.2 Deterministic identity lane

按：

```text
hash(brand_id, ref_norm)
```

分区或建立 lookup index，通过 hash/shuffle join 或数据库 keyed lookup 完成，避免笛卡尔积。

### 7.3 Gray lane

先做严格 business blocking，再进入 Zingg blocking：

```text
brand
  + optional category/product family
  + bounded candidate generation
```

并设置 block size 上限与 top-k，防止例如“CHANEL BAG”这类热门短标题产生巨大候选块。

### 7.4 存储

批处理建议使用 Parquet/Delta/Iceberg 一类列式表；`identity_index` 可落在支持 keyed merge/upsert 的表中。增量数据只需要查询相关 identity key 和局部 gray candidates，无需每天全量重算。

---

## 8. 增量匹配方案

每天新增/变更商品的处理流程：

```text
CDC / new batch
    ↓
normalize + extract ref
    ↓
identity_index lookup
    ├─ exact validated key exists
    │      ↓
    │   attach canonical id
    │
    ├─ new validated key
    │      ↓
    │   create canonical id
    │
    └─ ref missing / ambiguous
           ↓
       Zingg gray candidate generation
           ↓
       REVIEW
```

这与 Zingg Enterprise 的 `runIncremental` 思路一致，但我们的 identity key 和 audit edge 自己维护，因此即使使用 Community 版本也能做到业务层增量。

---

## 9. Precision-first 验证与上线门槛

“绝不能误匹配”无法靠一个模型 probability 声明来保证，必须转成工程控制面。

### 9.1 分证据层评估

至少分别统计：

```text
P1: structured ref == structured ref
P2: structured ref == title-extracted ref
P3: title-extracted == title-extracted
P4: no ref, Zingg candidate
```

不要只报告全局 precision，否则高质量 P1 会掩盖低质量 P3/P4。

### 9.2 对抗测试集

必须覆盖：

- 同品牌、同标题、不同 reference；
- reference 只差 1 个字符；
- leading zero；
- `-`、`/`、`.` 是否有语义；
- title 中年份被错当 reference；
- 42mm / 30ml / 18K 等规格数字；
- 同数字串跨品牌复用；
- brand alias 映射错误；
- 一条记录抽出多个候选 reference；
- title 截断；
- OCR/Unicode 相似字符（O/0、I/1 等）——默认不能自动纠错成相同 ref。

### 9.3 自动规则升级门槛

建议：

- P1 可以最早上线自动匹配；
- P2 只有 brand-specific parser 在独立验证集零误报后才能开启；
- P3 初期始终人工审核；
- P4 永远只作为候选，除非未来出现新的确定性证据。

### 9.4 审计与回滚

每条自动 edge 记录：

```text
rule_version
normalizer_version
parser_version
raw evidence
normalized evidence
decision_reason
```

规则升级后如发现问题，可定位并删除特定版本产生的 edge，而不是整库重跑和人工排查。

---

## 10. 推荐的最小可落地版本（MVP）

### Phase 0：先解决 80% 最确定的问题

只做：

1. canonical brand；
2. structured reference normalization；
3. `(brand_id, ref_norm)` exact identity index；
4. reference conflict hard veto；
5. audit edge。

这一步不依赖 ML，风险最低，可以最快上线。

### Phase 1：支持 title 内 reference

按品牌逐个实现 parser：

1. regex/tokenizer；
2. validator；
3. parser version；
4. collision dashboard；
5. structured-vs-title shadow 验证。

只有验证通过的 parser 才能进入 P2 自动匹配。

### Phase 2：接入 Zingg Gray lane

只对 ref 缺失/不确定数据使用：

1. `findTrainingData`；
2. 标注 hard positives / hard negatives；
3. `train`；
4. `link` 多来源候选；
5. 将输出写到 REVIEW queue，而非 canonical merge。

### Phase 3：持续学习

人工 review 结果反哺 Zingg：

```text
review decisions
      ↓
marked training pairs
      ↓
retrain / compare model
      ↓
shadow evaluation
      ↓
new candidate ranking model
```

注意：学习系统可以优化“谁值得审核”，但不能取消硬业务 invariant。

---

## 11. Zingg 与本方案的职责映射

| 能力 | Zingg 原生能力 | 本项目采用方式 |
|---|---|---|
| 分布式计算 | Spark/PySpark | 直接利用 |
| 候选缩减 | Blocking model | Gray lane 使用 |
| 多源关联 | `link` | 用于来源间候选发现 |
| 字段相似度 | EXACT/FUZZY/TEXT/... | Gray lane 特征 |
| 主动学习 | findTrainingData/label/train | 专门挖 hard cases |
| 自动模型阈值 | 概率匹配 | 不作为自动 merge 依据 |
| 图聚类 | cluster/transitive closure | 外加 ref cluster invariant |
| 稳定实体 ID | Enterprise Zingg ID | 建议自建 canonical ID/index |
| 增量 | Enterprise runIncremental | 可参考；业务层也可自建 |
| Explain/Diff | Enterprise | 有预算时用于治理，非 MVP 必需 |
| reference 冲突硬否决 | 默认 EXACT 不保证 | 必须自建 rule engine |

---

## 12. 一个可直接实现的 Decision Engine

伪代码：

```python
def decide(a, b):
    # 0. brand 是第一道边界
    if a.brand_id and b.brand_id and a.brand_id != b.brand_id:
        return "NO_MATCH", "BRAND_CONFLICT"

    # 1. 已知 reference 冲突拥有最高优先级
    if a.ref_valid and b.ref_valid and a.ref_norm != b.ref_norm:
        return "NO_MATCH", "REFERENCE_CONFLICT"

    # 2. 最强证据：结构化 reference 完全一致
    if (
        a.brand_id == b.brand_id
        and a.ref_valid and b.ref_valid
        and a.ref_norm == b.ref_norm
        and a.ref_origin == "STRUCTURED"
        and b.ref_origin == "STRUCTURED"
    ):
        return "AUTO_MATCH", "STRUCTURED_REFERENCE_EXACT"

    # 3. 结构化 vs title 高置信 parser
    if same_valid_ref(a, b) and exactly_one_structured(a, b):
        if title_side_parser_is_promoted(a, b) and not has_collision(a, b):
            return "AUTO_MATCH", "STRUCTURED_TITLE_REFERENCE_EXACT"
        return "REVIEW", "TITLE_REFERENCE_NOT_PROMOTED"

    # 4. 只有 title-derived reference：保守审核
    if same_valid_ref(a, b):
        return "REVIEW", "TITLE_ONLY_REFERENCE"

    # 5. 其余情况 Zingg 只做候选排序
    score = zingg_candidate_score(a, b)
    if score >= REVIEW_THRESHOLD:
        return "REVIEW", "ZINGG_GRAY_CANDIDATE"

    return "NO_MATCH", "INSUFFICIENT_EVIDENCE"
```

cluster 合并前再做：

```python
def can_union(cluster_a, cluster_b):
    keys = validated_identity_keys(cluster_a) | validated_identity_keys(cluster_b)
    return len(keys) <= 1
```

这两层足以阻止“模型高分误并”和“传递闭包误并”两个最危险路径。

---

## 13. 最终建议

对于该 Spec，推荐的技术决策不是“使用 Zingg 取代规则”，而是：

> **Deterministic Identity Key + Zingg Gray-Lane Entity Resolution**

即：

1. `(canonical_brand_id, validated_ref_norm)` 是自动身份主键；
2. structured reference 优先；
3. title extraction 必须品牌化、可解释、版本化；
4. 不同已知 reference 是 hard NO_MATCH；
5. Zingg 用于 ref 缺失场景的 blocking、候选排序和主动学习；
6. Zingg 的概率分数不能覆盖 reference 冲突；
7. canonical cluster 始终执行“最多一个 validated identity key”的 invariant；
8. 所有自动 edge 都可审计和回滚；
9. 以召回率换精度：拿不准就 REVIEW，不自动合并。

这种方案同时利用了 Zingg 在百万级实体解析上的工程能力，又把 Spec 最核心的“零误匹配优先”落实为确定性系统约束，而不是寄希望于模型阈值。

---

## 14. 本次阅读的 Zingg 实现/文档入口

以下均来自 `zinggAI/zingg` 仓库：

- `README.md`
- `examples/febrl/FebrlExample.py`：Python API / Spark 执行示例
- `docs/reference/cli-command-reference.md`：`findTrainingData`、`label`、`train`、`match`、`link`、`runIncremental` 等 phase
- `docs/zingg-concepts/how-zingg-learns/match-types/README.md`：EXACT/FUZZY/TEXT/NUMERIC/NULL_OR_BLANK 等 match type 语义
- `docs/zingg-concepts/concept-glossary.md`：blocking、active learning、probabilistic matching、graph clustering、identity graph、Zingg ID 等架构说明
- `common/client/src/main/java/zingg/common/client/MatchTypes.java`：match type 定义入口
- `python/zingg/client.py`：Python client 入口

分析基于 2026-08-18 阅读到的仓库版本/文档。