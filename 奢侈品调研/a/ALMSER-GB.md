# ALMSER-GB：Graph-boosted Active Learning for Multi-Source Entity Resolution 深入分析与三源二奢落地方案

> 分析对象：ALMSER-GB（Graph-boosted Active Learning for Multi-Source Entity Resolution）  
> 项目：https://github.com/wbsg-uni-mannheim/ALMSER-GB  
> 论文：Graph-boosted Active Learning for Multi-Source Entity Resolution（ISWC 2021）  
> 对应需求：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）

## 0. 结论先行

ALMSER-GB 最值得复用的不是它最终如何“判定两个实体相同”，而是它在**多源实体解析 + 极少人工标签**条件下，如何利用多源 correspondence graph 找到最值得人工标注的冲突样本，并把三个来源之间的标签信息复用起来。

对于当前需求，建议采用一个明确的“双平面”架构：

1. **生产决策平面（Decision Plane）**：只允许经过品牌约束、reference 抽取、reference 规范化、冲突检查后的 `canonical_reference` 硬一致性产生自动 MATCH；无法证明时一律 ABSTAIN/REVIEW。
2. **学习与审计平面（Learning / Audit Plane）**：借鉴 ALMSER-GB 的图冲突检测、委员会分歧、多数据源任务覆盖和主动学习，把有限的几百个人工标签花在最容易产生 false positive 的边界样本上。

也就是说：

> **复用 ALMSER-GB 的“选什么去标”和“如何利用三源图发现冲突”，不要照搬“图连通即可视为匹配”的生产判定语义。**

原因很直接：本需求已经定义“同一个商品 = 同一个 reference number”，且“绝对不能误匹配”。ALMSER-GB 中 `nx.has_path()` 的传递式图推断和 graph pseudo-label augmentation，一旦存在一条 false-positive 边，就可能把错误沿连通分量传播；这与 precision-first 目标冲突。

---

## 1. 当前需求与 ALMSER-GB 的对应关系

当前需求具有以下特点：

- 三个来源：雷小安、腕表之家、奢当家；
- 数据量约 100 万～1000 万，且需要持续增量更新；
- “同一个商品”的定义不是视觉相似，也不是同系列，而是**同一参考号 / reference number / 型号**；
- 字段稀疏，reference 有时是结构化字段，有时埋在标题中；
- 图片可用；
- precision 优先到极致，可以牺牲 recall；
- 可以人工标注几百对黄金样本。

ALMSER-GB 正好解决了其中一个关键子问题：**多源实体解析中，如何用有限标注预算选择最有信息量的 pair**。它把每个来源组合视作不同的 matching task，例如本项目天然存在：

- 雷小安 ↔ 腕表之家；
- 雷小安 ↔ 奢当家；
- 腕表之家 ↔ 奢当家。

如果后续增加第 4、第 5 个来源，pairwise task 数量还会快速上升，ALMSER 的跨任务复用价值会更明显。

---

## 2. ALMSER-GB 原项目的代码架构

项目不是一个完整“从原始商品到实体簇”的产品系统，而是建立在**已经生成 pair-level feature vectors**之上的主动学习实验框架。核心代码主要位于：

- `code/ALMSER.py`：核心状态、分类器训练、图增强训练；
- `code/ALMSER_EXP.py`：主动学习循环、query strategy、候选选择；
- `code/graphutils.py`：correspondence graph 构造、bridge/min-cut 处理；
- `code/learningutils.py`：RandomForest / GBoost / SVM / DT / LogisticRegression 等分类器；
- `code/scoreaggregation.py`：无监督相似度聚合与阈值；
- `code/ALMSER_CC_split.ipynb`：按连通分量切分 pool/test，避免图传递导致数据泄漏；
- `code/MatchingTaskProfiling.ipynb`：不同 source-pair matching task 的密度、corner case、重要特征画像。

项目 README 给出的主流程是：准备 feature vector 文件 → 设置查询预算和 query strategy → 加载训练/测试 pair → 运行 ALMSER → 记录 Precision / Recall / F1。

### 2.1 数据模型

`ALMSER.__init__` 接收 `feature_vector_train` 与 `feature_vector_test`。除了特征列外，代码显式维护：

- `source`、`target`：一对记录的节点 ID；
- `source_id`、`target_id`、`pair_id`；
- `datasource_pair`：该 pair 来自哪两个数据源；
- `agg_score`：无监督聚合相似度；
- `unsupervised_label`：无监督阈值得到的初始标签；
- `label`：黄金标签。

因此它的输入已经是假设“候选对生成 + 特征计算”完成之后的结果。

### 2.2 Bootstrap：先用无监督标签启动模型

初始化阶段先执行：

```python
model = getClassifier(self.classifier_name, n_estimators=10, warm_start=True)
self.learning_models['boot_all'] = model.fit(
    self.feature_vector.drop(['label','source','target'], axis=1),
    self.unlabeled_set['unsupervised_label']
)
```

也就是说，在没有人工标签时先用无监督标签训练初始模型。

如果 `bootstrap=True`，`bootstrap_labeled_set()` 还会针对每一个 `datasource_pair`：

- 找 `agg_score` 最大的 pair，当正样本；
- 找 `agg_score` 最小的 pair，当负样本；
- 将这些样本加入 labeled set。

这是一种典型的“极值 pseudo-label 冷启动”。在普通 ER 场景里有用，但对于本需求不能直接照搬，原因见后文。

### 2.3 分类器与委员会分歧

主模型默认使用 RandomForest。代码还建立了一个异构 committee：

- 当前主动学习模型；
- Decision Tree；
- Gradient Boosting；
- Logistic Regression；
- SVM。

`calculate_disagreement()` 比较多个模型投票结果，用委员会分歧作为 query informativeness 的一种信号。

这一点非常适合迁移：几百条人工标签不应该平均随机抽，而应该优先标注模型之间最不一致、最容易产生 false positive 的 pair。

### 2.4 correspondence graph

`graphutils.constructGraphFromWeightedPredictions()` 是项目最有价值的部分之一。

它大致按以下规则构图：

1. 模型预测为 positive 的 pair 作为图边；
2. 边权使用模型预测置信度 `pre_proba`；
3. 人工确认 positive 的边加入图，并设置很高的 capacity；
4. 人工确认 false 的边被移除；
5. 可删除某些 bridge；
6. 如果一个人工确认 negative 的 pair 在图中仍然有路径连接，则对该连通分量执行 `nx.minimum_cut()`，删除 cut 上的边，使这两个节点断开。

图中的推断标签是：

```python
nx.has_path(G, source, target)
```

即两个节点只要在同一个连通分量中，就被图推断为 match。

### 2.5 Graph-boosted Active Learning

在 `ALMSER_EXP.update_criteria_scores()` 中，核心链路是：

```text
当前模型预测
  ↓
构建 correspondence graph
  ↓
图传递推断 graph_inferred_label
  ↓
比较 predicted_label 与 graph_inferred_label
  ↓
把不一致的 pair 设为高 informativeness
  ↓
优先交给 oracle / 人工标注
```

`almser_gb` 主要关注：

- 模型判 match，但图推断不 match；
- 模型判不 match，但图传递后认为 match。

如果图和模型之间没有冲突，它退化到 committee disagreement。

这实际上是一种很合理的 **“结构约束 vs 局部分类器”冲突采样**。

### 2.6 图增强 pseudo-label

项目还会把部分“小连通分量”里的图推断标签当成补充训练数据，训练 `boost_graph` 模型：

```python
small_cc_data, small_cc_labels = \
    self.get_feature_vector_from_unlabeled_data(
        self.unlabeled_set[self.unlabeled_set.graph_cc_size <= self.count_sources]
    )

data_for_boost = pd.concat([small_cc_data, all_data])
labels_for_boost = np.concatenate((small_cc_labels, all_labels))
```

这能够扩大训练集，但对于“绝对不能误匹配”的本需求，风险很高：图只要有一条错边，就可能制造多个错误 pseudo-label。

### 2.7 Connected Component Split 防泄漏

README 特别说明 `ALMSER_CC_split` 不是随机 pair split，而是考虑完整图的 connected components，防止 pool 与 test 因传递关系泄漏。

这个设计应直接保留。实体解析里如果同一个 reference 的记录分别落入 train/test，随机切 pair 很容易让测试指标虚高。

---

## 3. ALMSER-GB 哪些设计不能直接用于当前需求

### 3.1 默认目标仍是 F1，而不是“零 false positive”

`learningutils.py` 中模型调参和交叉验证大量以 F1 为 scorer，实验输出也是 Precision / Recall / F1。

本需求的目标函数不同：

```text
第一目标：尽可能保证自动接受集合中没有误匹配
第二目标：在满足第一目标后，再提高自动覆盖率 / recall
```

因此不能用“F1 最优阈值”作为生产阈值。

### 3.2 图传递是最大的生产风险

假设：

```text
A(ref=126610) --误边--> B(ref=126610LN)
B --正确边--> C(ref=126610LN)
```

如果图判定标准是 `has_path(A, C)`，那么一条误边就能把整个簇污染。

腕表尤其危险，因为：

- 同系列不同 reference 外观极相似；
- 型号通常只有一两个字母/数字的差异；
- 标题可能把适配型号、系列号、机芯号、库存 SKU 混在一起；
- 图片相似度对近似变体很高。

所以**模型边不能参与生产 cluster 的传递闭包**。

### 3.3 `agg_score` 最大值当正样本不满足 precision-first

ALMSER 的 bootstrap 假设每个 source-pair 中最高相似 pair 很可能是 positive。

在本项目中仍可能出现：

- 同系列不同 reference 的两个商品正好非常相似；
- 表带、附件标题包含适配手表型号；
- 复制标题或图片造成高相似；
- 平台 SKU 被误当型号。

因此无监督极值只能作为“高优先级人工候选”，不能自动进入正标签集合。

### 3.4 “连通分量节点数 ≤ 来源数”不是可靠的 clean component 条件

ALMSER 用 source count 来判定部分小连通分量是否适合 graph-boost。

但二奢数据可能一个来源内就有多个记录属于同一个 reference：

- 不同卖家；
- 不同成色；
- 不同年份；
- 重复抓取或上下架重新发布。

因此实体簇大小与来源数没有一一对应关系。

### 3.5 RandomForest 的概率不是生产置信保证

项目使用 `predict_proba()` 作为图边权。但“模型概率 0.99”不等于“真实误匹配概率小于 1%”，尤其是在来源/品牌分布漂移时。

在当前需求中，模型概率更适合作为：

- review queue 排序；
- hard-negative mining；
- 冲突报警；

而不是最终自动 MATCH 门槛。

### 3.6 原项目不解决 reference 抽取本身

ALMSER 假设 pair feature 已经存在，而本需求真正的第一关键问题是：

> 如何从结构化字段、标题、图片 OCR 中找出“真正属于当前商品的 reference”，并规范成 canonical reference。

因此需要在 ALMSER 之前增加专门的 Reference Resolution 层。

---

## 4. 推荐落地架构：Reference-first + ALMSER-inspired Active Learning

## 4.1 总体架构

```text
                 ┌────────────────────────────┐
                 │ 雷小安 / 腕表之家 / 奢当家 │
                 └──────────────┬─────────────┘
                                │
                         Incremental Ingest
                                │
                 ┌──────────────▼─────────────┐
                 │  Brand Canonicalization    │
                 │ 品牌别名 / 中英文名统一      │
                 └──────────────┬─────────────┘
                                │
                 ┌──────────────▼─────────────┐
                 │ Reference Evidence Extract │
                 │ 字段 / 标题 / OCR / 型号库   │
                 └──────────────┬─────────────┘
                                │
                 ┌──────────────▼─────────────┐
                 │ Reference Canonicalization │
                 │ 品牌级规范化 + role 判定     │
                 └──────────────┬─────────────┘
                                │
               ┌────────────────┴─────────────────┐
               │                                  │
     VERIFIED canonical ref                 unresolved / conflict
               │                                  │
   Hash/Index exact lookup                 Candidate Retrieval
               │                                  │
        Hard Rule Engine                   pair features/model
               │                                  │
       MATCH / NO_MATCH                   ALMSER-style risk graph
                                                  │
                                          Human Review Queue
                                                  │
                                          Gold labels / rules
                                                  │
                                  extractor/model/risk strategy 更新
```

核心原则：**模型只能帮系统找到“应该看哪里”，不能凭自身相似度把一对记录升级成自动 MATCH。**

---

## 5. Reference Evidence：先解决“这个字符串到底是不是 reference”

建议把每次 reference 提取都保存成独立 evidence，而不是直接覆盖成一个字段。

### 5.1 Evidence 来源

优先级建议：

1. 平台结构化 reference/model 字段；
2. 标题中符合品牌 reference grammar 的候选；
3. 品牌型号库 / reference catalog 命中；
4. 图片 OCR：表背刻字、保卡、吊牌、证书；
5. 小模型/NER/LLM 抽取结果，仅作候选，不独立决定 MATCH。

数据结构：

```text
reference_evidence
- record_id
- raw_value
- normalized_candidate
- evidence_type       # structured/title/ocr/catalog/model
- source_field
- span_or_image_id
- role                # product_ref / compatible_ref / sku / serial / movement / unknown
- confidence
- parser_version
- conflict_flag
```

### 5.2 必须做“编号角色分类”

一个看起来像型号的字母数字串未必是商品 reference。

需要区分：

- 品牌 reference；
- 平台内部货号；
- 店铺 SKU；
- 序列号 serial；
- 机芯编号；
- 配件“适配型号”；
- 盒证上其他编号。

例如标题“适配 Rolex 126610LN 原装表带”里出现 `126610LN`，并不意味着当前商品就是 reference 126610LN 的腕表。

因此抽取阶段的任务不是简单 regex，而是：

```text
candidate extraction → role classification → canonicalization → evidence fusion
```

---

## 6. Canonical Reference：品牌级规范化，而不是全局暴力清洗

建议 canonical key：

```text
(brand_id, canonical_reference)
```

基础规范化可以统一：

- Unicode NFKC；
- 全角/半角；
- 大小写；
- Unicode dash 到 ASCII dash；
- 首尾空格；
- 品牌允许的分隔符变体。

但不要写一个全局规则“删掉所有空格、点、横线后比较”。因为有些品牌中标点、后缀、字母本身可能区分不同 reference。

推荐：

```python
canonicalize(brand_id, raw_reference, rule_version)
```

由品牌级策略决定：

- 哪些分隔符可以忽略；
- 哪些后缀必须保留；
- 是否需要补前导零；
- 大小写是否敏感；
- 哪些格式是无效的；
- 是否能与官方/可信 catalog 对齐。

所有规则必须版本化，以便错误规则出现后回放历史决策。

---

## 7. 百万～千万级 Candidate Generation：禁止全量笛卡尔积

如果有 1000 万记录，跨源全量 pairwise 比较不可行。

### 7.1 VERIFIED reference 记录

直接建立倒排/哈希索引：

```text
reference_index
PRIMARY KEY / INDEX: (brand_id, canonical_reference)
VALUE: record_id, source, evidence_status
```

新记录到达时：

```text
brand + verified canonical_reference
    ↓
O(1)/O(logN) lookup
    ↓
只比较同 key 的记录
```

这类流量应成为系统的主路径。

### 7.2 未解析 / 冲突记录

只对 unresolved tail 做候选召回：

- brand blocking；
- series/family blocking；
- title char-ngram / BM25；
- reference token edit-distance；
- catalog lookup；
- 图像 embedding top-K；
- OCR token retrieval。

候选召回的目标是高 recall，不负责最终自动匹配。

每个 unresolved record 只保留 top-K（例如几十个量级，由离线实验确定），再进入 pair feature 与主动学习层。

---

## 8. Production Decision Engine：三态输出而不是二分类

生产决策必须是：

```text
MATCH
NO_MATCH
REVIEW / ABSTAIN
```

推荐最小规则：

```python
def decide(a, b):
    if hard_brand_conflict(a, b):
        return NO_MATCH

    if a.ref.verified and b.ref.verified:
        if a.ref.canonical != b.ref.canonical:
            return NO_MATCH

        if has_hard_contradiction(a, b):
            return REVIEW

        return MATCH

    return REVIEW
```

### 8.1 自动 MATCH 的必要条件

至少同时满足：

- canonical brand 相同；
- 两侧都有 `VERIFIED` reference；
- canonical reference 精确一致；
- reference 的 role 都是“当前商品 reference”，不是兼容型号/SKU/serial；
- 不存在其他硬冲突证据。

### 8.2 自动 NO_MATCH

可以高置信拒绝的场景：

- canonical brand 不同；
- 两侧 VERIFIED reference 明确不同；
- 一侧明确是配件/附件且另一侧是腕表主体；
- 有硬证据证明 reference role 不同。

### 8.3 模型权限边界

强烈建议采用单向权限：

```text
模型可以：MATCH 候选 → 降级为 REVIEW
模型不可以：REVIEW → 升级为 MATCH
```

这样无论模型、图片 embedding 或 LLM 多么自信，都不会绕过 reference 硬证据。

---

## 9. 把 ALMSER 的图从“决策图”改成“风险图”

这是本方案最关键的改造。

### 9.1 节点与边类型

节点：商品记录。

边应显式带类型：

```text
HARD_MATCH       # verified canonical ref exact match
REVIEW_POSITIVE  # 人工确认同 reference
REVIEW_NEGATIVE  # 人工确认不同 reference
MODEL_CANDIDATE  # 模型认为相似，仅用于风险/候选
```

生产聚类只允许：

```text
HARD_MATCH + REVIEW_POSITIVE
```

参与连通性。

`MODEL_CANDIDATE` 绝不进入生产传递闭包。

### 9.2 图的用途

风险图主要做四件事：

1. 找出模型与硬 reference 规则的冲突；
2. 找出一个模型边如果被接受会连接两个不同 verified-reference 簇的危险情况；
3. 找出人工 negative pair 却在候选图中仍有大量路径的异常区域；
4. 给 active learning 排序。

ALMSER 中的 minimum-cut 也可以保留，但语义从：

> “自动删除边修复匹配图”

改成：

> “定位导致两个人工 negative 节点连通的最可疑 cut edges，送入人工复核”。

这样 minimum-cut 成为**错误定位工具**，而不是自动修改生产真值的工具。

---

## 10. ALMSER-inspired 主动学习策略：从“不确定性”升级为“误匹配风险”

原版 `almser_gb` 优先查询模型预测与 graph inference 的分歧样本。

当前场景可以定义更贴近 precision-first 的风险分数：

```python
risk_score = (
    5.0 * hard_reference_conflict
    + 3.0 * would_merge_different_verified_ref_clusters
    + 2.0 * reference_ambiguity
    + 1.5 * model_vs_rule_disagreement
    + 1.0 * committee_disagreement
    + 0.5 * source_pair_undercoverage
)
```

具体优先级建议：

### P0：可能制造 false positive 的样本

- 模型预测 match，但两边 verified reference 不同；
- 一个候选边将连接两个不同 canonical_reference 簇；
- title reference 与结构化字段 reference 冲突；
- OCR reference 与标题/字段冲突；
- 同系列相邻 reference，文本和图片又高度相似。

### P1：reference 角色不明确

- 标题同时出现商品 reference 和“适配型号”；
- SKU 与 reference 格式相似；
- 标题出现多个 reference；
- 表带、盒证、附件等可能借用腕表型号。

### P2：模型委员会强分歧

延续 ALMSER 的 DT / GBoost / Logistic / SVM / 主模型 committee disagreement。

### P3：数据源覆盖不足

如果标注集中在“雷小安 ↔ 腕表之家”，应适度提高另外两个 source pair 的采样权重，防止模型只熟悉某一平台组合。

---

## 11. 人工标注界面不要只问“是否相同”

为了让几百个标签产生最大价值，建议标注结果结构化：

```text
same_reference: true / false / unknown
left_reference: ...
right_reference: ...
left_evidence_type: field/title/ocr/catalog
right_evidence_type: ...
conflict_reason:
  - adjacent_reference
  - compatible_accessory
  - platform_sku
  - multiple_reference_candidates
  - OCR_error
  - brand_alias_error
  - other
```

这样一份人工标注同时可以服务：

- reference extractor；
- role classifier；
- canonicalization rule；
- pair classifier；
- risk query strategy；
- 规则回归测试。

比单一布尔 `match/non-match` 信息量大很多。

---

## 12. 黄金集与评测：必须防止图泄漏

直接借鉴 ALMSER 的 Connected Component Split。

不要随机切 pair，因为同一 reference 的多个节点可能通过另一条边泄漏到 train/test 两边。

推荐按照以下单位切分：

- canonical reference family / connected component；
- brand；
- 时间批次（额外做未来增量测试）；
- 来源组合。

黄金集重点覆盖：

1. 同系列不同 reference 的 hard negatives；
2. reference 只差一个字符的 hard negatives；
3. 标题 reference 与字段冲突；
4. 配件标题中的适配 reference；
5. OCR 错字符（O/0、I/1、S/5 等）；
6. 新型号/长尾型号；
7. 三个平台字段缺失模式；
8. 随机正常样本，用于估计分布覆盖。

### 指标顺序

第一层指标：

- `false_positive_count_auto_match`；
- `precision_auto_match`；
- `auto_match_coverage`；
- `abstain_rate`；
- 各 evidence type 的 reference extraction precision。

第二层指标：

- recall；
- F1；
- review queue hit rate；
- 每新增一个人工标签带来的错误减少量。

对于自动 MATCH 路径，任何发现的 false positive 都应该触发：

```text
定位 rule/parser/model version
→ 回放受影响记录
→ 降级/回滚规则
→ 增加回归样本
```

而不是仅仅把整体 F1 当作仍然“可接受”。

---

## 13. 增量处理流程

单条新商品进入系统时：

```text
1. normalize brand
2. extract all reference evidence
3. classify evidence roles
4. canonicalize reference candidates
5. detect intra-record reference conflicts
6. if exactly one VERIFIED canonical ref:
      query reference_index
      run hard contradiction checks
      produce MATCH / NO_MATCH / REVIEW
   else:
      retrieve top-K candidates
      build pair features
      compute risk_score
      enqueue REVIEW if valuable
7. persist decision + evidence + rule/model versions
```

主动学习不需要阻塞主数据流。它负责不断改善 unresolved tail 的处理能力和规则质量。

---

## 14. 推荐数据表

### `product_record`

```text
record_id
source
source_item_id
brand_raw
brand_id
category
title
raw_reference
raw_payload_uri
created_at
updated_at
```

### `reference_evidence`

```text
id
record_id
raw_value
canonical_candidate
evidence_type
role
confidence
parser_version
conflict_flag
```

### `reference_resolution`

```text
record_id
brand_id
canonical_reference
status            # VERIFIED / AMBIGUOUS / MISSING / CONFLICT
resolution_reason
rule_version
```

### `reference_index`

```text
brand_id
canonical_reference
record_id
source
```

### `match_edge`

```text
left_record_id
right_record_id
edge_type          # HARD_MATCH / REVIEW_POSITIVE / REVIEW_NEGATIVE / MODEL_CANDIDATE
decision
reason
model_version
rule_version
created_at
```

### `review_queue`

```text
pair_id
risk_score
risk_reasons
priority
state
created_at
```

### `gold_label`

```text
pair_id
same_reference
left_reference
right_reference
conflict_reason
reviewer
created_at
```

---

## 15. 工程技术栈建议

技术栈不是本问题的核心，可以保持简单、可审计：

- Python：reference parser、规则引擎、特征生成、active learning；
- PostgreSQL：决策元数据、规则版本、review queue；
- ClickHouse / Parquet：大规模离线 pair/features/评测日志；
- Polars / Spark：千万级批处理；
- OpenSearch / Elasticsearch：未解析记录的文本 top-K retrieval；
- FAISS / Milvus / pgvector（可选）：图片/文本 embedding 候选召回；
- NetworkX：黄金集、小规模风险子图调试；
- 图规模进一步增大后可改为基于表/邻接索引的增量冲突检测，不必为了“图”强上图数据库。

生产主路径是 `(brand_id, canonical_reference)` 索引查询，因此规模主要是数据库索引问题，不应变成千万级全图 pair matching 问题。

---

## 16. 可直接实现的代码模块划分

```text
luxury_matcher/
├── ingest/
│   ├── adapters.py
│   └── schema.py
├── brand/
│   └── canonicalize.py
├── reference/
│   ├── extract_structured.py
│   ├── extract_title.py
│   ├── extract_ocr.py
│   ├── role_classifier.py
│   ├── canonicalize.py
│   └── resolve.py
├── matching/
│   ├── exact_index.py
│   ├── hard_rules.py
│   ├── candidate_retriever.py
│   └── pair_features.py
├── active_learning/
│   ├── committee.py
│   ├── risk_graph.py
│   └── query_strategy.py
├── review/
│   ├── queue.py
│   └── labels.py
├── evaluation/
│   ├── cc_split.py
│   ├── precision_gate.py
│   └── regression_cases.py
└── jobs/
    ├── incremental_resolve.py
    └── retrain_review_model.py
```

### `hard_rules.py`

```python
from enum import Enum

class Decision(str, Enum):
    MATCH = "MATCH"
    NO_MATCH = "NO_MATCH"
    REVIEW = "REVIEW"


def decide_pair(left, right):
    if left.brand_id != right.brand_id:
        return Decision.NO_MATCH, "brand_conflict"

    lref = left.reference_resolution
    rref = right.reference_resolution

    if lref.status == "VERIFIED" and rref.status == "VERIFIED":
        if lref.canonical_reference != rref.canonical_reference:
            return Decision.NO_MATCH, "verified_reference_conflict"

        if left.is_accessory != right.is_accessory:
            return Decision.REVIEW, "product_role_conflict"

        return Decision.MATCH, "verified_reference_exact"

    return Decision.REVIEW, "insufficient_reference_evidence"
```

### `query_strategy.py`

```python
def score_for_review(pair):
    score = 0.0
    reasons = []

    if pair.verified_reference_conflict:
        score += 5.0
        reasons.append("verified_reference_conflict")

    if pair.would_merge_different_ref_clusters:
        score += 3.0
        reasons.append("dangerous_cluster_merge")

    if pair.reference_ambiguity:
        score += 2.0
        reasons.append("reference_ambiguity")

    if pair.model_vs_rule_disagreement:
        score += 1.5
        reasons.append("model_rule_disagreement")

    score += pair.committee_disagreement
    score += 0.5 * pair.source_pair_undercoverage

    return score, reasons
```

这就是把 ALMSER 的 `disagreement_graph_pred + datasource_pair_frequency + committee disagreement` 改造成 precision-risk oriented query strategy。

---

## 17. ALMSER 原设计到本需求的映射

| ALMSER-GB 组件 | 原作用 | 本需求建议 |
|---|---|---|
| `datasource_pair` | 区分不同来源组合 | 原样保留，三个 source-pair 独立统计覆盖率 |
| `agg_score` | 无监督相似度与 bootstrap | 只用于候选排序/人工采样，不直接赋 positive 标签 |
| `unsupervised_label` | 冷启动模型 | 可用于训练弱监督候选模型，但不得形成自动 MATCH |
| RandomForest `predicted_label` | pair match 判别 | review/risk signal |
| committee disagreement | 主动学习 | 直接保留 |
| correspondence graph | 传递匹配 | 改成风险审计图 |
| `graph_inferred_label` | 作为结构推断标签 | 改成冲突信号，不作为生产真值 |
| `disagreement_graph_pred` | 找模型-图冲突 | 直接保留思想，升级为 model-rule/cluster conflict |
| graph pseudo-label / `boost_graph` | 自动扩充训练集 | 默认禁用；如实验，只能在离线模型训练中使用且不得影响硬门槛 |
| minimum-cut | 断开 negative pair | 用于定位可疑边并送审，不自动修改生产真值 |
| task relatedness | 跨来源迁移 | 可保留，后续来源扩展时价值更大 |
| connected-component split | 防止 train/test 泄漏 | 强烈保留 |

---

## 18. 分阶段落地顺序

### 阶段 A：先做 deterministic reference baseline

- 三源 schema 对齐；
- brand canonicalization；
- structured/title reference extraction；
- brand-aware canonicalization；
- exact reference index；
- 三态 hard-rule engine；
- 建立第一版黄金集和 false-positive regression cases。

先验证“只靠 reference 硬证据”能覆盖多少数据。这一步通常能快速给出 production-safe baseline。

### 阶段 B：处理 unresolved tail

- BM25 / ngram / catalog top-K 候选；
- OCR evidence；
- pair feature；
- review queue。

### 阶段 C：引入 ALMSER-inspired 主动学习

- source-pair coverage；
- committee disagreement；
- model-vs-rule disagreement；
- risk graph；
- dangerous cluster merge detection；
- 每轮把最危险的几十对送审。

### 阶段 D：再考虑多模态模型

图片更适合作为：

- unresolved 候选召回；
- OCR reference 来源；
- 硬冲突/人工复核辅助证据。

不要让图片相似度直接覆盖 reference 硬规则。

---

## 19. 上线验收门槛

建议至少满足：

1. 每一个自动 MATCH 都能回溯到 `brand + canonical_reference + evidence`；
2. 不存在“模型分高所以自动 MATCH”的路径；
3. reference parser/canonicalizer 均有 version；
4. 测试集按 reference/component 隔离，避免泄漏；
5. hard negative 中重点包含同系列相邻 reference；
6. 一旦出现自动路径 false positive，可以定位到具体 parser/rule version 并回放；
7. unresolved case 默认 REVIEW，而不是强迫模型二分类；
8. 监控以 false positive 和 auto-match precision 为第一指标，F1 只做辅助指标。

---

## 20. 最终建议

ALMSER-GB 与当前三源需求的匹配度很高，但正确的采用方式是“拆开用”：

- **采用**：多 source-pair 建模、committee disagreement、graph-vs-model conflict、主动学习、task coverage、connected-component split、minimum-cut 的冲突定位思想；
- **改造**：correspondence graph 只作为审计/采样图；
- **禁用生产语义**：预测边的传递闭包、graph pseudo-label 直接扩张实体簇、F1 最优阈值自动放行；
- **新增主干**：brand-aware reference extraction → role classification → canonicalization → exact reference index → hard-rule 3-way decision。

如果只保留一句落地原则：

> **让 reference 规则负责“能不能自动合并”，让 ALMSER-GB 式主动学习负责“下一条最值得人看的数据是什么”。**

这样既能利用几百个人工标签持续提高覆盖率，又不会因为模型或图的一次误判破坏“绝对不能误匹配”的核心约束。

## 参考

- ALMSER-GB 项目：https://github.com/wbsg-uni-mannheim/ALMSER-GB
- README：https://github.com/wbsg-uni-mannheim/ALMSER-GB/blob/master/README.md
- 核心主动学习实现：https://github.com/wbsg-uni-mannheim/ALMSER-GB/blob/master/code/ALMSER.py
- 实验 / Query Strategy：https://github.com/wbsg-uni-mannheim/ALMSER-GB/blob/master/code/ALMSER_EXP.py
- 图实现：https://github.com/wbsg-uni-mannheim/ALMSER-GB/blob/master/code/graphutils.py
- 无监督 score aggregation：https://github.com/wbsg-uni-mannheim/ALMSER-GB/blob/master/code/scoreaggregation.py
