# SC-Block：Supervised Contrastive Blocking within Entity Resolution Pipelines——面向跨源二奢/腕表 reference 匹配的技术分析与落地方案

> 分析对象：SC-Block: Supervised Contrastive Blocking within Entity Resolution Pipelines  
> 论文：https://arxiv.org/abs/2303.03132  
> 官方代码：https://github.com/wbsg-uni-mannheim/SC-Block  
> 本次代码分析基于官方仓库 `main` 当前提交：`41d1bb3c616e2f24ac877b38aa3205bdbad33771`  
> 对应需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 需求核心：100 万–1000 万级持续增量；“同一个商品”定义为**同一 reference number / 型号**；字段高度稀疏；图片可用；**precision 极端优先、允许漏匹配**。

---

## 0. 本次选题排除检查

执行前已检查 `奢侈品调研/d/` 当前目录，已有 30 篇分析，包括 DeepBlocker、pyJedAI、TransClean、GraLMatch、AnyMatch、Confidence Classifiers、Conformal Selective Prediction、Ameli、MOON2.0、parts-distributor-sku-classifier 等，但**没有 SC-Block**。

`奢侈品文章调研.md` 中 SC-Block 对应条目为：

> “专门解决实体解析 Blocking，使用监督对比表示与近邻搜索生成候选；论文按 99.5% pair completeness 构造管线，并在大规模商品基准上显著缩短运行时间，可作为百万—千万级三源数据第一层候选生成，避免全量笛卡尔积。”

因此本次选择 SC-Block，不重复此前 `d` 已分析对象。

---

## 1. 结论先行

SC-Block 最值得当前项目复用的，不是“用向量相似度判断同款”，而是下面三件事：

1. **把 Blocking 明确做成独立层**：先把千万级笛卡尔积压缩成每条记录很小的候选集合；
2. **用监督对比学习训练候选召回表示**：让同一实体/同一 reference 的记录靠近，不同实体远离；
3. **用 source-aware sampling 降低跨来源漏标导致的假负样本污染**，这对雷小安、腕表之家、奢当家三个来源尤其重要。

但是，SC-Block **绝不能直接成为当前需求的最终 matcher**。

当前业务定义非常强：

```text
同一个商品  <=>  同一品牌语义下的同一 canonical reference number
```

而 SC-Block 输出的是“向量空间里最像的 Top-K 邻居”。腕表中最危险的恰恰是：

```text
同品牌
+ 同系列
+ 外观近似
+ 标题近似
+ reference 只差一个后缀/数字
```

这种 pair 会天然在 embedding 空间很近，却必须判为**不同商品**。

因此我建议把 SC-Block 改造成一个更适合本需求的组件：

> **Reference Candidate Retriever：只召回“可能的 reference”，不直接产生 match edge。**

生产链路应是：

```text
高置信 reference 已抽到
    -> canonicalize
    -> brand_id + canonical_ref 严格等值
    -> 自动归组

reference 缺失/歧义
    -> SC-Block 风格候选 reference 召回
    -> 定向在标题/描述/OCR/结构化字段里找证据
    -> 证据足够才产出 verified canonical_ref
    -> 再走严格等值归组

仍无硬证据
    -> ABSTAIN / 人工复核
```

这能同时满足：

- 千万级可扩展；
- 能处理 reference 埋在标题或字段缺失；
- 图片可以参与；
- **embedding、视觉模型、LLM 都没有权限越过 reference 硬规则制造自动匹配。**

---

## 2. SC-Block 原论文到底解决什么问题

实体匹配若直接比较来源 A 和来源 B 的全部记录，复杂度是：

```text
|A| × |B|
```

两个来源各 100 万条就是 `10^12` 对；三个来源两两比较更不可接受。

因此标准 ER 管线拆成：

```text
Blocker
  -> 生成小候选集 C
  -> Matcher
  -> 最终匹配集合 M
```

SC-Block 的创新点是：

```text
记录序列化
  -> 监督对比学习训练 encoder
  -> 每条记录得到 embedding
  -> FAISS nearest-neighbor search
  -> Top-K candidate pairs
```

论文在评测完整 pipeline 时，会把候选集调到很高的 pair completeness，再交给更贵的 matcher。论文摘要报告：

- 以约 **99.5% pair completeness** 的候选集进入 matcher；
- 在较小基准上完整 pipeline 比其它 blocker 组合快约 **1.5–2×**；
- 大规模商品基准上，最佳 matcher 组合从约 **2.5 小时降至 18 分钟**，约 **8×**；
- blocker 的训练时间约 5 分钟，在该实验中能被后续节省的匹配时间明显摊薄。

这组结果证明的是：

> **更好的 Blocking 能显著减少 matcher 的工作量，同时基本不损伤最终 F1。**

它并不证明“embedding 相似度本身可以达到当前业务要求的零误匹配”。两者必须严格区分。

---

## 3. 官方仓库架构拆解

README 给出的完整实验链路是：

```text
01_prepare_datasets.sh
  -> 准备 query table / index table

02_preprocess_records_and_index_es.sh
  -> 预处理记录并写入 Elasticsearch

03_process_training_data.sh
  -> 生成对比学习训练数据

训练 contrastive model
  -> RoBERTa + supervised contrastive loss

04_load_data_into_faiss.sh
  -> embedding index table
  -> 写入 FAISS

05_run_strategy.sh
  -> blocker + matcher 完整实验
```

代码可按职责拆成：

```text
SC-Block/
├── src/finetuning/open_book/contrastive_pretraining/
│   ├── .../data/datasets.py       # correspondence graph、source-aware sampling、序列化训练数据
│   ├── .../models/loss.py         # SupConLoss / SimCLR / Barlow Twins
│   ├── .../models/modeling.py     # Transformer encoder、mean pooling、L2 normalize
│   └── .../run_pretraining...sh   # 训练参数
│
├── src/strategy/entity_serialization.py
│   └── [COL] attr [VAL] value 序列化
│
├── src/strategy/indexing/
│   ├── index_faiss_entity.py      # 批量 embedding + 多进程收集
│   └── faiss_collector.py         # FAISS IndexFlatIP / IndexFlatL2
│
└── src/strategy/retrieval/
    └── query_by_neural_entity.py  # encode query -> FAISS search -> 回查原始记录
```

这一层次划分很适合直接借鉴：

```text
Data preparation
    ↓
Representation learning
    ↓
Vector index
    ↓
Candidate retrieval
    ↓
Final decision
```

但官方仓库明显是**论文复现实验架构**，不是针对持续增量千万级线上服务设计，后面需要改造。

---

## 4. Record Serialization：字段感知输入，而不是裸拼字符串

论文采用 Ditto 风格的序列化：

```text
[COL] attribute_name [VAL] actual_attribute_value
```

例如：

```text
[COL] Name [VAL] Apple iPhone 13 mini
[COL] Color [VAL] green
```

官方 `EntitySerializer` 对 WDC product 使用：

```text
brand
name
price
pricecurrency
description
```

并且代码会截断 description。

这个思路对当前项目比 DeepBlocker 的“所有字段直接拼起来”更合适，因为字段角色得以保留。

### 4.1 腕表场景建议序列化

不建议直接使用论文的商品字段集合，而应变成：

```text
[BRAND] rolex
[SERIES] submariner
[TITLE] 劳力士 潜航者 黑水鬼 自动机械 41mm
[REF_RAW] 126610LN
[REF_ROLE] brand_reference
[MODEL_TEXT] 126610 ln
[DESC] ...
[OCR] ...
[SOURCE] leixiaoan
```

但这里必须分两种训练视图。

#### Full View

保留已知 reference，用于：

- 学习脏格式、别名、上下文；
- 测试有 reference 但格式不统一的召回。

#### Masked-Reference View

把显式 reference 字段和标题中已识别出的 reference token 遮掉：

```text
[REF_RAW] [MASK_REF]
[TITLE] 劳力士 潜航者 [MASK_REF] 黑水鬼 自动机械 41mm
```

原因非常重要：

如果训练标签本身就是 canonical reference，而输入里又直接包含 reference，模型很容易学成一个“reference 字符串复制器”。离线指标会非常漂亮，但线上真正需要它处理的是：

```text
reference 缺字段 / 埋标题未被规则抽到 / OCR 不完整
```

因此至少应有一部分 batch 强制 mask reference，逼 encoder 学品牌、系列、尺寸、机芯、材质、标题上下文等辅助信息。

---

## 5. Training Data Preparation：SC-Block 最值得复用的部分

### 5.1 correspondence graph

论文输入通常是 pairwise 正样本：

```text
record_A <-> record_B = match
```

但 supervised contrastive loss 更希望每条记录有一个实体 label，因此 SC-Block 先建图：

```text
节点 = record
正匹配 pair = edge
connected component = entity cluster
cluster_id = supervised contrastive label
```

官方代码 `datasets.py` 也是这样做的。

### 5.2 当前项目其实有更好的 label 来源

我们的业务定义已经规定：

```text
brand_id + canonical_ref
```

就是实体身份。

因此对于已有高置信 reference 的记录，根本不必先人工标 pair，可以直接构造弱监督 label：

```python
label = hash(brand_id + "|" + canonical_ref)
```

例如：

```text
雷小安      Rolex 126610LN   -> rolex|126610ln
腕表之家    劳力士 126610ln  -> rolex|126610ln
奢当家      ROLEX 126610-LN  -> rolex|126610ln
```

三条记录天然同 label。

这样可以从大量已有显式 reference 的商品自动生成训练集，把“几百个人工黄金 pair”主要用在：

- 规则验收；
- 极难相邻 reference；
- reference 角色歧义；
- 阈值/拒识校准；
- 模型错误分析。

这是比直接拿几百条 pair 训练 blocker 更高效的利用方式。

---

## 6. Source-Aware Sampling：三源数据中非常关键

SC-Block 特别处理了 contrastive learning 的一个真实问题：**漏标 match 会变成错误负样本。**

假设：

```text
雷小安 R1
腕表之家 R2
```

实际上是同 reference，但训练集中没有任何边把它们连接起来，那么普通 supervised contrastive batch 会把它们当成不同 label，并主动把 embedding 推远。

论文把这个问题称为 inter-label noise，并采用 source-aware sampling。

官方代码做法大意是：

1. 已知正匹配组成 connected components；
2. 已知 cluster 中记录保留；
3. 对没有已知 match 的 singleton，按来源拆开；
4. 分别构造 source-aware training dataset，降低“未知跨源真匹配同时出现在 batch 中却被当负样本”的概率。

### 6.1 三源版建议

把源码中的二源逻辑推广成：

```text
D_leixiaoan
D_xbiao
D_shedangjia
```

每个训练视图都包含：

```text
全部 verified multi-source ref clusters
+
仅当前 source 的 unresolved singleton
```

训练时轮换/混合三个 sampler。

不要像官方实现一样通过 record id 字符串里是否包含 `amazon`、`google`、`tablea`、`tableb` 来判断来源；生产数据必须有明确字段：

```text
source ∈ {LEIXIAOAN, XBIAO, SHEDANGJIA}
```

### 6.2 官方 connected component 实现不要照抄

`datasets.py` 里通过 `bucket_list` 手工维护集合：遇到 edge 后把新节点加进第一个命中的 bucket，然后 `break`。

实验数据上可能够用，但生产系统不应照搬，因为当一条边连接两个已经存在的 component 时，需要**真正合并两个集合**。

建议直接用：

```text
Disjoint Set Union / Union-Find
```

复杂度接近：

```text
O(E α(N))
```

且不会出现 component 合并不完整问题。

不过对本项目更进一步：最终自动 grouping 根本不需要模型图传递，直接用：

```text
brand_id + canonical_ref
```

作为 group key 即可，Union-Find 主要用于训练标签准备或人工确认边的离线整理。

---

## 7. Supervised Contrastive Encoder 的真实实现

官方 `modeling.py` 的核心非常直接：

```text
Transformer encoder
  -> token hidden states
  -> mean pooling
  -> L2 normalize
  -> SupConLoss
```

论文实验使用 `roberta-base`。

代码中的 `ContrastivePretrainModel`：

```python
output_left = encoder(...)
output_left = mean_pooling(...)

output_right = encoder(...)
output_right = mean_pooling(...)

output = cat(left, right)
output = normalize(output)
loss = SupConLoss(output, labels)
```

注意一个实现细节：仓库虽然定义了 `ContrastivePretrainHead`，但 SC-Block 实际 supervised contrastive 主模型并没有把 projection head 接进 forward；索引脚本同样明确：

```text
--with_projection=False
--dimensions=768
```

所以论文复现路径最终索引的是 **768 维 RoBERTa mean-pooled representation**。

### 7.1 SupCon loss

官方 `loss.py` 基于 Khosla et al. 的 Supervised Contrastive Loss：

- 同 label 样本为 positive；
- 不同 label 为 negative；
- 对 embedding 做相似度矩阵；
- temperature 缩放；
- 每个 anchor 同时拉近多个 positive、推远大量 negative。

论文配置：

```text
effective batch size = 1024
temperature          = 0.07
learning rate        = 5e-5
epochs               = 20
```

训练脚本还启用了：

```text
weight_decay = 0.01
warmup_ratio = 0.05
max_grad_norm = 1.0
fp16
gradient checkpointing
```

这些参数可以作为我们的第一版 baseline，但不应直接当最终最优参数。

---

## 8. 为什么腕表训练一定要加“同系列不同 reference”困难负例

普通 SupCon 只要 batch 足够大，会自然看到很多不同 label；但腕表业务最关键的 negative 不是随机负例，而是：

```text
同品牌 + 同系列 + 同尺寸 + 同颜色语义 + reference 极相似
```

例如：

```text
126610LN  vs 126610LV
```

这种 pair 对最终 precision 的威胁远大于：

```text
Rolex Submariner vs Cartier Tank
```

所以三源版训练 sampler 应主动加入 hard negative bucket：

```text
brand 相同
AND series 相同或标题 embedding 很近
AND canonical_ref 不同
AND ref edit distance 很小 / 共享长前缀
```

建议每个 anchor 的 batch 组成里明确保留：

```text
1~N 个同 reference positives
+ 多个同品牌相邻 reference hard negatives
+ 普通随机 negatives
```

这会让 blocker 学到：

> “语义很像”不代表“同 reference”。

即便如此，它依然只负责 candidate retrieval，最终也不能替代 reference exact gate。

---

## 9. FAISS 检索层：论文实现与千万级生产实现的差异

官方 `faiss_collector.py`：

```python
if similarity == 'f2':
    faiss.IndexFlatL2(dim)
else:
    faiss.IndexFlatIP(dim)
```

当使用 cosine similarity 时，encoder 先 L2 normalize，随后使用 `IndexFlatIP`，因此 inner product 等价于 cosine ranking。

官方检索路径：

```text
index table records
  -> GPU batch embedding
  -> Queue
  -> FaissIndexCollector
  -> IndexFlatIP
  -> faiss.write_index(...)

query record
  -> encoder
  -> index.search(vector, k)
  -> 得到 FAISS ids + score
  -> Elasticsearch 回查原始实体
```

`index_faiss_entity.py` 还做了多进程/GPU queue，把 embedding 生成和 FAISS 收集分开。

### 9.1 为什么 `IndexFlatIP` 不适合直接照搬到 1000 万记录

768 维 float32 向量，仅裸向量内存约：

```text
100 万条  ≈ 2.86 GiB
1000 万条 ≈ 28.61 GiB
```

这还没算：

- FAISS/服务进程开销；
- 原始记录；
- 多版本索引；
- 查询时 exact scan 的计算成本。

`IndexFlatIP` 是 exact nearest-neighbor，规模大时每次 query 都要扫大量向量。

因此生产实现建议：

```text
POC：FAISS HNSW / IVF
生产：按品牌 shard 的 FAISS HNSW/IVF-PQ，或可在线服务化的向量库
```

但最重要的优化不是换 ANN 算法，而是**不要索引 1000 万条 listing**。

---

## 10. 针对当前业务的关键改造：从 Record Index 变成 Reference Prototype Index

业务最终实体不是 listing，而是：

```text
brand + reference
```

一个 reference 可能在三个平台各出现几十、几百、几千条二手 listing。

如果把每条 listing 都放进 ANN：

- 索引巨大；
- 热门型号会占满 Top-K；
- 召回的是“记录”，后面还要再聚合成 reference；
- 增量更新频繁。

更合适的设计是：

```text
verified listings
    -> 按 canonical reference 分组
    -> 每个 reference 选少量 prototype / anchor
    -> ANN 只索引 prototype
```

### 10.1 prototype 选择

每个 reference 可以保存：

```text
每来源 1~3 条高质量代表记录
+
1 个全局 centroid（可选）
```

例如：

```text
rolex|126610ln
  ├── 雷小安 anchor 1
  ├── 腕表之家 anchor 1
  ├── 奢当家 anchor 1
  └── centroid
```

查询时：

```text
missing-ref record
  -> embedding
  -> Top-K prototype
  -> group by canonical_ref
  -> candidate reference list
```

这样 SC-Block 的输出直接变成：

```json
[
  {"ref": "126610LN", "score": 0.91},
  {"ref": "126610LV", "score": 0.89},
  {"ref": "124060",   "score": 0.83}
]
```

而不是几十条具体 listing。

### 10.2 为什么这比原论文结构更适合本项目

它把系统目标从：

```text
找一条最像的商品记录
```

改成：

```text
找少数最可能的 reference 候选
```

后者与业务身份定义完全一致，而且能显著压缩索引规模。

假设最终只保留 60 万个 768d prototype，裸 float32 向量约 **1.72 GiB**，已经比直接索引 1000 万 listing 小一个数量级；实际 reference 数量和 prototype 数应根据数据统计确定。

---

## 11. 推荐的完整生产架构

```text
             ┌──────────────────────┐
             │ 雷小安 / 腕表之家 / 奢当家 │
             └──────────┬───────────┘
                        │
                        ▼
              ┌──────────────────┐
              │  Ingest + Normalize │
              └────────┬─────────┘
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
 ┌─────────────────┐       ┌─────────────────┐
 │ Brand Normalizer │       │ Image/OCR Pipeline│
 └────────┬────────┘       └────────┬────────┘
          └────────────┬────────────┘
                       ▼
              ┌───────────────────┐
              │ Reference Extractor│
              │ + Role Classifier  │
              │ + Canonicalizer    │
              └────────┬──────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
  verified canonical_ref     missing / ambiguous
          │                         │
          ▼                         ▼
 ┌──────────────────┐    ┌────────────────────────┐
 │ Exact Ref Registry│    │ SC-Block Ref Retriever │
 │ brand + ref       │    │ brand-sharded ANN      │
 └────────┬─────────┘    └──────────┬─────────────┘
          │                          │
          │                   candidate refs
          │                          │
          │                          ▼
          │               ┌──────────────────────┐
          │               │ Targeted Ref Verifier│
          │               │ field/title/OCR      │
          │               └──────────┬───────────┘
          │                          │
          │               ┌──────────┴──────────┐
          │               │                     │
          │               ▼                     ▼
          │          verified ref            ABSTAIN
          │               │                /人工复核
          └───────────────┘
                          ▼
                  ┌─────────────────┐
                  │ Strict Match Gate│
                  │ exact ref only   │
                  └────────┬────────┘
                           ▼
                  match_group_id
```

核心原则：

> **SC-Block 只负责缩小“要验证哪些 reference”的空间，严格 Match Gate 才有最终写入匹配关系的权限。**

---

## 12. Reference 抽取与 canonicalization 应先于 SC-Block

SC-Block 不能解决“平台 SKU 被误认为品牌 reference”这类角色问题，所以 reference pipeline 应独立存在。

建议 `ref_evidence` 至少区分：

```text
brand_reference
platform_sku
seller_sku
serial_number
accessory_compatible_reference
ocr_candidate
unknown_code
```

### 12.1 canonicalization 原则

建议顺序：

```text
Unicode NFKC
-> trim
-> Latin uppercase
-> 品牌级 separator 规则
-> alias / prefix / suffix 规范
-> brand reference catalog 校验
```

非常重要：**不要做过度 normalization。**

禁止类似：

```text
把所有字母都删掉
把所有 leading zero 都删掉
只保留数字
把所有后缀都忽略
```

因为腕表 reference 中后缀经常具有身份意义。

例如：

```text
126610LN
126610LV
```

绝不能被规范成同一个值。

正确做法是品牌级规则：

```text
raw_ref       -> canonical_ref
126610-ln     -> 126610LN
126610 LN     -> 126610LN
126610LN      -> 126610LN
```

但：

```text
126610LV != 126610LN
```

---

## 13. 最终 Match Gate：必须是可审计的硬不变量

建议自动匹配只允许：

```python
def can_auto_match(a, b):
    return (
        a.brand_id == b.brand_id
        and a.ref_status == "VERIFIED"
        and b.ref_status == "VERIFIED"
        and a.canonical_ref == b.canonical_ref
        and not a.has_reference_conflict
        and not b.has_reference_conflict
    )
```

最终 group 可以直接定义：

```text
match_group_id = SHA256(brand_id + "|" + canonical_ref)
```

好处：

- 不依赖 pairwise 模型边；
- 不依赖图传递；
- 不会因为一条错误模型边把两个 cluster 整体污染；
- 可重算；
- 易审计；
- 对持续增量天然友好。

### 13.1 明确禁止的自动匹配条件

以下任何单一条件都不能自动产生 match：

```text
embedding cosine > 0.99
图片很像
标题很像
品牌 + 系列一样
LLM 说是同款
OCR 猜到一个相近型号
reference edit distance 很小
Top-1 ANN 很确定
```

它们最多提供：

```text
candidate
supporting evidence
review priority
```

最终必须回到 verified canonical reference。

---

## 14. SC-Block 在本系统里的在线执行方式

### 14.1 已有 verified reference

最快路径：

```text
record
-> brand normalize
-> ref extract
-> canonicalize
-> reference_registry lookup
-> exact group
```

不经过 ANN。

### 14.2 reference 缺失或歧义

```text
record
-> 构造 masked-reference serialization
-> bi-encoder embedding
-> 只查询该 brand 的 prototype ANN shard
-> Top-K reference candidates
-> candidate-specific verifier
```

candidate-specific verifier 不应该再让 LLM自由生成 reference，而应做**受限验证**：

```text
候选只有：
126610LN
126610LV
124060
```

然后检查：

- 标题原文是否存在等价形式；
- 描述里是否存在；
- 结构化型号字段是否存在；
- OCR 是否存在；
- 是否出现冲突 reference；
- 该字符串扮演的是品牌 reference 还是“兼容型号/平台 SKU”。

只有候选 reference 得到可解释硬证据后，才升级为 `VERIFIED`。

### 14.3 reference 完全不存在

即使 SC-Block Top-1 非常高，也默认：

```text
ABSTAIN
```

因为业务明确允许漏匹配，且“不误匹配”优先级远高于 coverage。

这种记录可以：

- 进入人工复核；
- 等后续图片 OCR；
- 等来源数据补充；
- 作为未匹配记录保留。

---

## 15. 图片应该怎么接入

图片在本项目里最有价值的不是“视觉同款直接 match”，而是**补 reference 证据**。

推荐优先级：

```text
表背刻字 OCR
保卡 OCR
吊牌 OCR
盒证 OCR
表盘文字 OCR
    >
视觉 embedding 同款检索
```

### 15.1 OCR 作为证据

例如 SC-Block 先召回：

```text
candidate_ref = 126610LN
```

然后 OCR 在表背/保卡识别到：

```text
M126610LN-0001
```

经过品牌规则规范化并识别为 brand reference，可成为很强的 supporting evidence。

### 15.2 视觉 embedding 的权限

视觉 embedding 只建议用于：

- 候选召回；
- Top-K 排序；
- 人工复核界面；
- 冲突检测。

不要让视觉相似度直接写 `match_group_id`。

腕表的相邻 reference 外观过于接近，这一点决定了视觉只能是辅助模态。

---

## 16. 建议的数据模型

### 16.1 `product_record`

```text
id
source
source_item_id
brand_raw
brand_id
title_raw
description_raw
structured_ref_raw
created_at
updated_at
```

### 16.2 `ref_evidence`

```text
record_id
evidence_type       # structured/title/description/ocr/manual
raw_value
normalized_value
role                # brand_reference/platform_sku/...
confidence
extractor_version
image_id nullable
created_at
```

### 16.3 `reference_registry`

```text
brand_id
canonical_ref
aliases
series
status
rule_version
```

唯一键：

```text
UNIQUE(brand_id, canonical_ref)
```

### 16.4 `record_resolution`

```text
record_id
canonical_ref nullable
ref_status          # VERIFIED / AMBIGUOUS / MISSING / CONFLICT
resolution_method   # explicit/title_regex/ocr/manual/...
rule_version
```

### 16.5 `ann_anchor`

```text
anchor_id
brand_id
canonical_ref
record_id
source
encoder_version
embedding_version
quality_score
```

### 16.6 `match_member`

```text
group_id
record_id
brand_id
canonical_ref
match_rule_version
```

`group_id` 只由 verified `brand_id + canonical_ref` 生成。

---

## 17. 三源 SC-Block 训练方案

### 17.1 自动弱标签

先从高置信数据构造：

```text
(label, record)
=
(hash(brand_id|canonical_ref), serialized_record)
```

只接受：

- 明确结构化 reference；或
- 标题/描述抽取后命中品牌 reference catalog 且角色明确；或
- 人工确认；或
- OCR 与文本/结构化证据一致。

不要把 SC-Block 自己预测出的 reference 再直接回流当训练真值，否则容易 self-reinforcement。

### 17.2 三源 source-aware sampler

```python
for source in SOURCES:
    dataset[source] = (
        all_verified_multi_source_clusters
        + unresolved_singletons_from(source)
    )
```

训练时交替取 batch。

### 17.3 reference masking

建议至少维护两种 augmentation：

```text
A. full
B. masked_ref
```

其中 masked_ref 对：

- structured ref；
- 标题里的 canonical ref alias；
- OCR 中明确 reference；

做 mask。

重点评估 `masked_ref -> reference candidate recall`。

### 17.4 hard negative mining

每天/每轮离线从以下 bucket 挖 hardest negatives：

```text
same brand
same series
same size/material/color terms
reference long-prefix overlap
small edit distance
ANN nearest but ref != ref
```

训练中提高这些 negative 的出现率。

### 17.5 数据切分不要随机 pair split

随机 pair split 很容易泄漏同一个 reference 的其它 listing。

建议至少做：

```text
Reference holdout split：整个 canonical_ref 只出现在 train 或 test 一侧
Source holdout split：验证跨平台泛化
Time split：旧数据训练，新增数据测试
Hard-neighbor split：专门测试相邻 reference
```

这比随机 F1 更能反映实际增量系统能力。

---

## 18. ANN 索引建议

论文使用 exact `IndexFlatIP`，我们的生产版建议：

### 第一层：brand shard

```text
Rolex index
Omega index
Cartier index
Patek index
...
```

品牌本身就是强业务约束，先做 shard 可以同时：

- 减少计算；
- 避免跨品牌相似型号字符串；
- 降低 false candidate；
- 便于独立更新热点品牌。

### 第二层：prototype ANN

每个 shard 只放 verified reference prototype。

初期可以：

```text
FAISS HNSW / IVF
```

如果需要：

- 在线 filter；
- 多副本；
- 持续增量；
- 服务化运维；

再替换为支持这些能力的向量服务。

接口一定要抽象：

```python
class ReferenceCandidateIndex:
    def add(self, anchors): ...
    def search(self, brand_id, vector, k): ...
    def rebuild(self, encoder_version): ...
```

这样底层从 FAISS 切换到其它 ANN，不影响上游业务逻辑。

---

## 19. 模型版本与索引版本必须绑定

SC-Block 官方仓库把模型名写进 FAISS 文件名，这个思路应保留并加强。

生产中至少记录：

```text
encoder_version
serialization_version
reference_rule_version
index_version
anchor_snapshot
```

因为 encoder 一旦升级，旧 embedding 和新 embedding 不能随意混用。

推荐发布模式：

```text
v1 encoder + v1 index 继续服务
后台构建 v2 index
离线验收 v2
原子切换 alias -> v2
保留 v1 可回滚
```

这比在原 index 上混写不同模型版本安全得多。

---

## 20. 直接可落地的 Resolver 伪代码

```python
def resolve(record):
    brand_id = normalize_brand(record)

    evidences = extract_reference_evidence(record, brand_id)
    resolution = resolve_verified_reference(evidences, brand_id)

    # Fast path: 已有硬 reference
    if resolution.status == "VERIFIED":
        return exact_group(
            brand_id=brand_id,
            canonical_ref=resolution.canonical_ref,
            evidences=evidences,
        )

    # Slow path: SC-Block 只负责召回 reference candidates
    query = serialize_for_blocker(
        record,
        brand_id=brand_id,
        mask_detected_reference=True,
    )
    vector = blocker_encoder.encode(query)

    candidates = reference_ann.search(
        brand_id=brand_id,
        vector=vector,
        k=20,
    )

    # 按 canonical_ref 聚合多个 prototype
    candidate_refs = aggregate_by_reference(candidates)

    for candidate_ref in candidate_refs:
        proof = verify_candidate_reference(
            record=record,
            brand_id=brand_id,
            candidate_ref=candidate_ref,
            sources=["structured", "title", "description", "ocr"],
        )

        if proof.is_verified and not proof.has_conflict:
            return exact_group(
                brand_id=brand_id,
                canonical_ref=candidate_ref,
                evidences=proof.evidences,
            )

    return Abstain(
        reason="no_verified_reference",
        candidate_refs=candidate_refs,
    )
```

这里没有任何：

```text
if cosine > threshold: match
```

这是刻意的。

---

## 21. 最小可行版本（建议实现顺序）

### Phase A：先完成无模型高精度主链

```text
brand normalization
reference role classification
reference extraction
brand-aware canonicalization
reference_registry
exact match_group_id
conflict / abstain
```

这一阶段已经能自动解决最安全的一批商品。

### Phase B：接入 SC-Block Reference Retriever

只处理：

```text
MISSING
AMBIGUOUS
```

训练数据使用已有 verified references 自动弱监督生成。

输出只写：

```text
candidate_reference
candidate_score
prototype_provenance
```

不写 match。

### Phase C：Candidate-specific Verifier

在 Top-K reference 约束下：

- 精确/品牌级 alias 搜索；
- OCR 二次识别；
- 编号角色判断；
- 冲突检测。

只有 verifier 把 candidate 升级为 `VERIFIED` 才进入 exact group。

### Phase D：图片增强

优先 OCR，再做视觉候选召回/人工复核辅助。

---

## 22. 评测体系

不能只看普通 Entity Matching F1。

### 22.1 Blocker 指标

```text
Reference Recall@K
Pair Completeness@K
候选 reference 数/query
p50/p95/p99 查询延迟
index size
新增写入吞吐
```

最重要测试：

```text
把真实 reference 从 query 文本中 mask 掉后，Recall@K 还有多少？
```

否则模型可能只是记住 reference token。

### 22.2 Final Resolver 指标

```text
Auto-match precision
Auto-match coverage
Abstain rate
Conflict rate
False positive count
人工复核命中率
```

业务既然“绝对不能误匹配”，验收必须优先看：

```text
FP = 0
```

然后才讨论 coverage。

### 22.3 黄金集必须刻意包含最危险样本

不要只随机抽几百 pair。

应至少覆盖：

```text
同系列不同 ref
reference 只差一个字符
同 reference 多种格式
平台 SKU 像 reference
配件标题包含兼容腕表 reference
标题有两个 reference
OCR 错一个字符
品牌别名
新增未见 reference
同图/极似图不同 reference
```

### 22.4 几百条黄金标签不能单独“证明”极端 precision

如果只抽几百个接受样本、恰好观察到 0 个错误，统计上仍不足以证明极低错误率。

因此本系统的高 precision 不能主要依赖：

```text
“模型在 300 条样本上 precision=100%”
```

而应依赖：

1. **可验证硬不变量**：brand + canonical reference exact equality；
2. 冲突即拒绝；
3. 模型只能召回；
4. 大规模持续抽样审计；
5. hard-case 黄金集。

这也是当前需求与普通 ML matching 项目最大的区别。

---

## 23. 对官方 SC-Block 代码的可复用与需要替换部分

### 可以直接参考

```text
[COL]/[VAL] 字段感知序列化思想
Supervised Contrastive Loss
mean pooling + L2 normalize
source-aware sampling
bi-encoder / retrieval 分层
FAISS candidate retrieval 接口
模型与索引版本绑定
blocking + matcher 分层评测
```

### 不建议直接照搬

```text
二源写死逻辑
通过 id 字符串判断来源
bucket_list 手写 connected component
IndexFlatIP 承担千万级在线索引
所有 listing 全量进 ANN
Elasticsearch id 与 FAISS row id 强耦合
模型相似度直接被理解为实体身份
随机 benchmark 指标代替 precision-first 验收
```

### 应改造成

```text
三源 source enum
Union-Find / 直接 reference label
Reference Prototype ANN
brand shard
Masked-reference training
hard negative mining
candidate-specific reference verification
strict exact match gate
ABSTAIN-first
```

---

## 24. 与现有 `d` 调研结果的关系

本篇与此前几篇可以形成一个完整分层：

```text
parts-distributor-sku-classifier
    -> 解决“这个编号到底是不是厂商 reference”

Using LLMs for Extraction and Normalization...
    -> reference 抽取与规范化

SC-Block（本篇）
    -> reference 缺失时，大规模召回少量候选 reference

DeepBlocker
    -> Blocking 的通用工程抽象

Ameli / 多模态方案
    -> 图片与细粒度属性辅助消歧

Confidence / Conformal Selective Prediction
    -> 若有模型判定环节，如何拒识和风险控制

TransClean / GraLMatch
    -> 多源匹配图的错误边/传递一致性审计
```

但当前需求最安全的主线仍应保持：

```text
reference evidence
    > candidate retrieval score
    > semantic similarity
    > visual similarity
```

---

## 25. 最终建议

如果现在就要定架构，我建议采用：

```text
“Reference Registry + Strict Exact Gate” 为主系统
“SC-Block-style Reference Candidate Retriever” 为 recall 插件
“OCR / 图像 / LLM” 为证据增强插件
“ABSTAIN” 为一等公民
```

其中 SC-Block 最适合承担：

> **对 reference 缺失/歧义记录，从百万—千万级商品世界里快速缩小到 5–20 个候选 reference。**

它不适合承担：

> **最终回答这两条记录是不是同一商品。**

最终身份判定始终应该由：

```text
same brand_id
AND
same verified canonical_ref
AND
no reference conflict
```

收口。

这样既真正复用了 SC-Block 在大规模 Blocking 上的优势，也不会把它的相似度模型能力误用成一个高风险的最终 matcher。

---

## 参考资料

1. SC-Block: Supervised Contrastive Blocking within Entity Resolution Pipelines  
   https://arxiv.org/abs/2303.03132

2. 官方 SC-Block 代码  
   https://github.com/wbsg-uni-mannheim/SC-Block

3. 官方仓库 README：复现流程、Elasticsearch/FAISS 依赖、实验脚本顺序  
   https://github.com/wbsg-uni-mannheim/SC-Block/blob/main/README.md

4. SC-Block supervised contrastive model 实现  
   `src/finetuning/open_book/contrastive_pretraining/src/contrastive/models/modeling.py`

5. SupConLoss 实现  
   `src/finetuning/open_book/contrastive_pretraining/src/contrastive/models/loss.py`

6. source-aware sampling / correspondence graph  
   `src/finetuning/open_book/contrastive_pretraining/src/contrastive/data/datasets.py`

7. FAISS index 构建  
   `src/strategy/indexing/index_faiss_entity.py`  
   `src/strategy/indexing/faiss_collector.py`

8. ANN 查询实现  
   `src/strategy/retrieval/query_by_neural_entity.py`

9. 当前需求 Spec  
   https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31
