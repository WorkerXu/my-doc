# ALMSER-GB：把图增强主动学习改造成“零误合并优先”的跨源 Reference 审核系统

> 分析人：c  
> 目标需求：跨源二奢/腕表商品实体匹配（雷小安 × 腕表之家 × 奢当家）  
> 核心约束：100 万–1000 万级持续增量；字段稀疏；reference 可能在结构化字段、标题或图片中；“同一个商品”定义为**同一 reference number / 型号**；**precision 极端优先，绝不能误匹配，允许漏匹配**。

## 0. 选题、去重与结论先行

本次从 `奢侈品文章调研.md` 选取：

- 项目：**ALMSER-GB**
- 全称：**Graph-boosted Active Learning for Multi-Source Entity Resolution**
- GitHub：https://github.com/wbsg-uni-mannheim/ALMSER-GB
- 论文：Anna Primpeli, Christian Bizer, *Graph-boosted Active Learning for Multi-Source Entity Resolution*, ISWC 2021

执行前重新检查 `奢侈品调研/c/`，当前 c 已经分析过 Ameli、AnyMatch、Confidence Classifiers、Conformal Selective Prediction、DeepBlocker、TransClean、pyJedAI、parts-distributor-sku-classifier 等项目/论文，但**没有 ALMSER-GB**，因此满足“每次分析前排除已经分析过条目”的要求。

### 结论先行

ALMSER-GB 与当前需求最契合的部分，并不是它最终训练出的 Entity Matching 分类器，而是下面两个机制：

1. **多源 correspondence graph**：把三个来源的 pairwise 匹配关系放在同一张图里，通过传递关系、负边约束、连通分量发现潜在错误。
2. **Graph-boosted Active Learning**：人工标签不随机花，而优先标“模型判断和图结构冲突”的 pair，让几百条黄金标签更集中地消灭 false positive / hard negative。

但原版 ALMSER 的默认目标仍然是提升 F1，而且会把模型预测正边放入图，再利用图推断产生额外训练标签。对本需求“绝对不能误匹配”的约束来说，这一做法风险过高：**一条错的软正边可能通过图传递污染整个连通分量**。

所以建议不是“原样部署 ALMSER”，而是做一个 **ALMSER-Guarded** 版本：

```text
ALMSER 原版：
模型预测 -> correspondence graph -> 图推断标签 -> 增强训练 -> 匹配

推荐：
Reference 证据抽取/规范化
    -> 硬规则安全合并图 G_truth
    -> 模型软候选图 G_review
    -> 图冲突/风险采样
    -> 人工黄金标签
    -> 改进抽取器/审核器

最终自动合并只允许：
Canonical Brand 一致
AND Canonical Reference 严格一致
AND 无强冲突证据
AND 通过组件级安全约束
```

换句话说：

> **ALMSER 用来决定“下一条最值得让人看什么”，而不是决定“哪两条数据可以自动合并”。**

这是把它落到当前 Spec 最重要的一次架构改造。

---

## 1. ALMSER-GB 原项目到底解决什么问题

ALMSER 面向的是 **Multi-Source Entity Resolution**：不是只处理 A 表与 B 表，而是多个来源之间存在多个 matching task，例如三个来源会形成：

```text
雷小安 <-> 腕表之家
雷小安 <-> 奢当家
腕表之家 <-> 奢当家
```

传统做法往往把三个任务独立训练、独立采样，这会浪费多源之间已经存在的传递信息。ALMSER 的核心思想是：

```text
如果 A1 ~ B7
且 B7 ~ C3
那么 A1 和 C3 的关系就不应完全独立看待。
```

因此项目把所有候选 record pair 统一投影为 correspondence graph，再让图结构参与：

- 主动学习 query selection；
- 识别模型可能的 false positive / false negative；
- 产生部分图推断标签；
- 给学习器补充训练数据；
- 在多个 source-pair matching task 之间复用标注价值。

这与当前三源腕表场景非常同构，尤其适合“只有几百条人工黄金标签”的限制。

---

## 2. 从源码拆解 ALMSER 的实现架构

项目主要代码位于 `code/`：

```text
ALMSER.py
ALMSER_EXP.py
ALMSER_utils.py
graphutils.py
learningutils.py
ALMSER_log.py
datautils.py
MatchingTaskProfiling.ipynb
ALMSER_CC_split.ipynb
ALMSER_Experiments.ipynb
```

它不是一个 production service，而是论文复现实验代码：Python 3.7 + pandas + scikit-learn + NetworkX + Notebook。

### 2.1 输入：先 Blocking，再把候选 pair 变成 feature vector

README 和实验 Notebook 都表明，ALMSER 并不负责对所有记录做笛卡尔积；它消费的是前置 Blocking 之后的候选 pair feature vector。

每个 pair 除了特征，还保留：

```text
source
source_id
target
target_id
pair_id
datasource_pair
agg_score
unsupervised_label
label
```

`datasource_pair` 让模型知道这个 pair 来自哪个来源组合，例如：

```text
leixiaoan_watchhome
leixiaoan_shedangjia
watchhome_shedangjia
```

这对当前业务很重要，因为三个平台的数据风格和字段可靠性不一样，不能假设同一阈值在所有 source pair 上都同样可靠。

### 2.2 冷启动：无监督标签 + 每个来源对的极值样本

`ALMSER.__init__()` 会先用 `unsupervised_label` 训练一个 bootstrap 模型：

```python
model = getClassifier(..., n_estimators=10, warm_start=True)
model.fit(feature_vector, unsupervised_label)
```

然后 `bootstrap_labeled_set()` 会对每个 `datasource_pair`：

- 选择 `agg_score` 最大的一条作为正向 bootstrap；
- 选择 `agg_score` 最小的一条作为负向 bootstrap；
- 放进 labeled set；
- 再开始主动学习。

这个设计是“少标注 ER”的经典冷启动技巧，但**不应该直接照搬到当前腕表业务**。

原因是：在 reference-first 场景，最高字符串相似度也可能是同系列不同 reference，例如只差一个字符的近邻型号；把“最高相似”直接当真阳性，与“绝不能误匹配”目标冲突。

推荐替代为：

```text
正 bootstrap：只来自已验证的 canonical reference exact-equality
负 bootstrap：优先来自同品牌、同系列、reference 仅一位不同的 hard negative
```

也就是说，冷启动本身仍可保留，但标签来源必须从“相似度极值”改成“可证明的 reference 证据”。

### 2.3 Base learner：全局模型 + 多来源 pair 模型

`learningutils.py` 提供：

- RandomForest
- GradientBoosting
- SVM
- DecisionTree
- LogisticRegression

`ALMSER.py` 会维护：

- `boot_all`：bootstrap/warm-start 模型；
- `all`：全部已标注数据上的主模型；
- `all_simple`：轻量版本；
- `boost_graph`：加入图推断训练数据后的模型；
- 某些策略下还会按 source-pair/task group 训练多个模型。

这说明 ALMSER 的核心不是一个特殊神经网络，而是**主动学习 + 图结构 + 多任务采样策略**。因此迁移到当前项目时，不必保留 RandomForest；可以把 classifier 换成 LightGBM/XGBoost/小型 cross-encoder，但图增强采样思想保持不变。

### 2.4 Correspondence Graph：模型正预测边 + 人工正边 + 人工负约束

`graphutils.py::constructGraphFromWeightedPredictions()` 是整个项目最关键的实现。

它大致做四件事：

1. 把模型预测为 match 的 pair 加入 NetworkX graph；
2. 边权使用预测概率 `pre_proba`；
3. 人工标注 positive edge 加入图，权重人为设得极高；
4. 人工标注 negative pair 不允许连接。

抽象后是：

```text
record = node
predicted match = soft weighted edge
manual positive = very strong edge
manual negative = cannot-connect constraint
```

这比独立 pair classifier 多出一个非常重要的能力：**可以发现 pairwise 模型看不到的全局冲突**。

### 2.5 Negative constraint + minimum cut：用人工负例切断错误传递路径

如果人工明确标记：

```text
A != C
```

但当前 graph 中仍然存在：

```text
A -- B -- C
```

那么 `graphutils.py` 会在当前 connected component 内调用 NetworkX `minimum_cut`，根据边权切掉一组代价最低的边，使 A 与 C 不再连通。

这一点对腕表特别有价值。

假设：

```text
雷小安 A：Rolex 126610LN
腕表之家 B：标题只写“劳力士潜航者黑水鬼”
奢当家 C：Rolex 116610LN
```

如果视觉/标题相似模型错误地让：

```text
A ~ B
B ~ C
```

人工只需确认一次 `A != C`，图约束就能提示中间至少有一条软边有问题。

但是当前项目应把 minimum-cut 用于**定位可疑软边**，而不是自动重写真实 entity component。后面会给出生产版拆法。

### 2.6 Graph inferred label：传递连通关系作为辅助标签

原版 ALMSER 使用：

```python
nx.has_path(G, source, target)
```

判断两个节点是否在图上连通，从而生成 `graph_inferred_label`。

随后会计算：

```text
graph_cc_size
model predicted_label
graph_inferred_label
disagreement_graph_pred
```

其中最关键的主动学习信号就是：

```text
模型说 MATCH，但图说 NO_MATCH
或
模型说 NO_MATCH，但图说 MATCH
```

这些 pair 比随机样本更值得人标。

### 2.7 Graph-boosted training：用“小型干净连通分量”补训练标签

`update_learning_models_per_ds()` 中，如果已经有图推断标签，会抽取 `graph_cc_size <= count_sources` 的小 connected component，把这些图推断 label 与人工 labeled data 拼起来，训练 `boost_graph` 模型。

原设计意图是：

```text
少量人工标签
+ 图中相对干净的弱标签
=> 扩大训练集
```

这也是 ALMSER label efficiency 的来源之一。

不过对当前 Spec，这里必须降权：

> 图推断标签最多只能作为 weak label 训练“候选排序器/风险模型”，不能成为自动合并事实。

### 2.8 Query strategy：前 20 次先不用图，再专找图-模型冲突

`ALMSER_EXP.py::simplify_qs()` 有一个很实用的工程细节：当使用图策略时，最开始约 20 次迭代仍然退回普通 disagreement strategy，因为图太早时不稳定。

随后 `get_informativeness_score()` 对 `almser_gb` 的主逻辑是：

```text
若存在 graph vs model disagreement：
    优先采样这些冲突 pair
否则：
    fallback 到 heterogeneous committee disagreement
```

Committee 会组合多种分类器预测，利用模型分歧寻找不确定样本。

这个策略非常值得直接移植到三源腕表黄金标签分配中。

---

## 3. 为什么原版 ALMSER 不能直接作为当前系统的最终 matcher

## 3.1 原版优化 F1，本需求优化的是“近乎零 false positive”

论文框架最终还是普通二分类 ER：

```text
MATCH / NON-MATCH
```

而当前需求的判定定义更强：

```text
MATCH <=> 同一 canonical reference
```

这意味着我们根本不应该让一个相似度分类器拥有最终裁决权。

例如：

```text
Rolex 116610LN
Rolex 126610LN
```

名称、系列、外观、图片都极其相似，但 reference 不同，根据 Spec 必须是 NON-MATCH。

因此最终业务规则应是“证明同一 reference”，不是“模型觉得像”。

## 3.2 一条错误软边可能产生传递污染

假设模型误加一条：

```text
A(ref=116610LN) -- B(ref=126610LN)
```

如果 B 又与其他 126610LN 节点连接，那么图连通性可能把 A 带进错误 component。

对追求 recall/F1 的论文实验，这种风险可以靠后续 AL 修复；但生产环境里一次错误自动合并可能就是不可接受的数据事故。

## 3.3 NetworkX 全局图不适合 1000 万级持续增量

原项目是实验代码，graph 在单机内存里用 NetworkX 构造。对于 100 万–1000 万 listing：

- 不应每批重建全球图；
- 不应对所有模型候选边做全局 `minimum_cut`；
- 不应把主业务状态仅存在 pandas DataFrame / Notebook。

生产版应改成：

```text
硬合并组件：DSU / component table
软候选/负约束：数据库 edge table
局部冲突审计：只拉取单个小 component 到 NetworkX
```

## 3.4 论文的“clean component weak labels”不能升级成业务真值

对本项目最危险的误用是：

```text
图看起来干净
=> 图推断 match
=> 自动合并
```

不应该这么做。

“clean component”最多可以帮助模型训练、风险排序、发现漏抽 reference；自动 merge 仍必须经过 reference hard gate。

---

## 4. 先重新定义问题：不是 Pair Matching，而是 Reference Entity Linking

当前 Spec 已经把“同一个商品”定义成“同一个 reference”。因此最合适的数据模型不是：

```text
listing A <-> listing B ?
```

而是：

```text
listing A -> Canonical Reference Entity
listing B -> Canonical Reference Entity
```

例如：

```text
雷小安 #8731
  -> brand=Rolex
  -> canonical_ref=126610LN

腕表之家 #9921
  -> brand=Rolex
  -> canonical_ref=126610LN

奢当家 #3022
  -> brand=Rolex
  -> canonical_ref=126610LN
```

只要三个 listing 都被**可靠地链接到相同 `(brand_id, canonical_ref)`**，才形成同一 reference entity。

这样能把最难、最危险的问题从：

```text
千万级 pair similarity classification
```

变成：

```text
每条 listing 的 reference evidence extraction + normalization + verification
```

这也解释了 ALMSER 在新架构中的角色：它主要帮助我们更高效地标注**抽取/规范化/冲突审核的困难样本**。

---

# 5. 推荐生产架构：ALMSER-Guarded

## 5.1 总体结构

```text
                   ┌─────────────────────┐
                   │ 雷小安 / 腕表之家 / 奢当家 │
                   └──────────┬──────────┘
                              │
                       Ingestion / CDC
                              │
                              v
                   ┌─────────────────────┐
                   │ Normalized Record Store │
                   └──────────┬──────────┘
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             v                v                v
       Structured Field   Title Extractor   Image OCR
       Role Resolver      + Brand Grammar  + Ref Detector
             │                │                │
             └────────────────┼────────────────┘
                              v
                   ┌─────────────────────┐
                   │ Reference Evidence Set │
                   └──────────┬──────────┘
                              │
                       Canonicalizer
                              │
                              v
                   ┌─────────────────────┐
                   │ Precision Hard Gate │
                   └──────┬────────┬─────┘
                          │        │
                    SAFE MATCH   ABSTAIN
                          │        │
                          v        v
                       G_truth   G_review
                          │        │
                          │   ALMSER 风险采样
                          │        │
                          │        v
                          │     人工审核
                          │        │
                          └────< 黄金标签回流
```

系统从架构上把两张图分开，是防止误匹配的关键：

### `G_truth`：业务事实图

只允许：

- 人工确认 positive；
- 经过严格 canonical reference gate 的 deterministic positive。

它负责真实 entity component。

### `G_review`：模型软候选图

允许：

- 文本 matcher 高分；
- 图像相似；
- OCR 相似；
- 模型推断；
- 不完整 reference 候选。

它**永远不直接触发 merge**，只服务：

- active learning；
- 冲突检测；
- 候选排序；
- 人工审核。

这相当于给 ALMSER 增加一条生产安全边界。

---

# 6. Reference Evidence：把“型号字符串”变成可审计证据

## 6.1 不要只存一个 `reference` 字段

每条 listing 应保存多个候选和来源：

```json
{
  "record_id": 123,
  "brand_id": "rolex",
  "evidences": [
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "type": "STRUCTURED_REFERENCE_FIELD",
      "role": "BRAND_REFERENCE",
      "confidence": 0.999
    },
    {
      "raw": "126610LN",
      "canonical": "126610LN",
      "type": "TITLE_RULE",
      "role": "BRAND_REFERENCE",
      "confidence": 0.995
    }
  ]
}
```

为什么必须存 evidence set：

- 平台字段名叫“型号”不代表一定是品牌 reference；
- 标题里的编号可能是平台 SKU、店铺货号、机芯号、尺寸、年份；
- 配件标题可能写“适用 126610LN”，但卖的是表带；
- OCR 可能把 `0/O`、`1/I`、`5/S` 识错。

因此要先做**编号角色识别**，再做 reference 匹配。

## 6.2 Evidence type 建议

```text
STRUCTURED_REFERENCE_FIELD_VERIFIED
TITLE_BRAND_PATTERN
DESCRIPTION_BRAND_PATTERN
OCR_CASEBACK
OCR_CARD
OCR_TAG
OCR_DIAL
CATALOG_LOOKUP
LLM_EXTRACTED
PLATFORM_SKU
SHOP_SKU
UNKNOWN_IDENTIFIER
```

只有前面少数强证据可进入 hard gate；LLM 输出默认只作为 candidate，不作为事实。

---

# 7. Canonical Reference Normalizer：这是系统真正的核心

需要维护按品牌版本化的 normalization profile，而不是写一个全局 `replace('-', '')`。

## 7.1 推荐接口

```python
canonicalize_ref(
    brand_id: str,
    raw_value: str,
    evidence_type: str,
    product_type: str,
) -> RefCandidate
```

输出：

```python
RefCandidate(
    raw="126610 ln",
    canonical="126610LN",
    valid=True,
    rule_version="rolex-v7",
    role="BRAND_REFERENCE",
    warnings=[],
)
```

## 7.2 规范化步骤

通用层只做低风险操作：

```text
Unicode NFKC
trim
统一大小写
统一全角/半角
去除明确的展示空白
```

是否删除 `- / .` 等符号必须按品牌 profile 决定。

原因是不同品牌的 reference 结构不同，过度 normalization 会把本来不同的编号碰撞成同一个 key。

## 7.3 Brand-specific validator

每个品牌维护：

```text
合法字符集
长度区间
前缀/后缀规则
分段结构
已知 canonical reference catalog
已知 alias
已知高风险近邻
```

如果 canonicalization 后不符合品牌规则：

```text
ABSTAIN
```

不要为了提高 recall 强行猜。

---

# 8. Precision Hard Gate：真正决定能否自动合并

推荐把最终判断写成纯规则服务，而不是 `model_score > 0.99`。

## 8.1 最小安全条件

两条 listing 自动判同 reference 至少要求：

```text
1. canonical_brand_id 一致
2. product_type 都确认是主体商品，不是表带/盒证/配件
3. 两侧都存在高可信 canonical_reference
4. canonical_reference 严格相等
5. 任一侧不存在另一个同等级强证据指向不同 reference
6. component-level invariant 校验通过
```

伪代码：

```python
def safe_match(a, b):
    if a.brand_id is None or a.brand_id != b.brand_id:
        return "REJECT"

    if a.product_type != "WATCH" or b.product_type != "WATCH":
        return "ABSTAIN"

    ra = best_hard_reference(a)
    rb = best_hard_reference(b)

    if ra is None or rb is None:
        return "ABSTAIN"

    if ra.canonical != rb.canonical:
        return "REJECT"

    if has_strong_reference_conflict(a) or has_strong_reference_conflict(b):
        return "ABSTAIN"

    if not component_union_is_safe(a, b, ra.canonical):
        return "ABSTAIN"

    return "MATCH"
```

这里没有图像相似阈值，没有大模型“觉得是同款”，没有模糊字符串兜底。

这正是 Spec 的 precision-first 约束应该长成的代码形态。

---

# 9. 用 ALMSER 思路重做 Active Learning

## 9.1 人工标签应该花在哪里

几百条黄金标签不应该随机抽。

最应该标的是：

```text
同品牌、同系列、reference 只差 1 个字符
reference 在标题中但字段缺失
结构化字段与标题 reference 冲突
OCR 与标题冲突
主体表 vs 表带/盒证/配件
同一 reference 多种书写格式
不同来源对字段可靠性差异大的样本
模型高置信 MATCH 但 hard gate REJECT 的样本
图传递会导致 reference 冲突的边
```

这些都直接针对 false positive。

## 9.2 Pair feature vector

可以给每个 review candidate 建立：

```text
brand_exact
reference_exact
reference_edit_distance
reference_prefix_equal
reference_suffix_equal
reference_length_equal
reference_source_trust_a
reference_source_trust_b
structured_vs_title_consistency_a
structured_vs_title_consistency_b
ocr_exact
ocr_conflict
product_type_equal
accessory_keyword_a
accessory_keyword_b
series_equal
title_embedding_similarity
image_embedding_similarity
price_ratio
year_conflict
size_conflict
source_pair
```

注意：

- `reference_exact` 是主要硬特征；
- embedding / image similarity 只能影响“是否值得审核”，不能提高为自动 merge 证据；
- `source_pair` 必须保留，因为不同平台噪声机制不同。

## 9.3 从 ALMSER 继承的 query score

可以实现：

```text
query_score =
    w1 * graph_model_disagreement
  + w2 * false_positive_risk
  + w3 * boundary_uncertainty
  + w4 * underrepresented_source_pair
  + w5 * reference_conflict_severity
```

其中：

### `graph_model_disagreement`

复用 ALMSER 的核心：

```text
pair model 认为 MATCH
但 reference/graph invariant 认为不可连接
=> 高优先级审核
```

这类样本非常可能就是危险 false positive。

### `false_positive_risk`

对“模型高分但硬规则冲突”的样本加权最高。

### `underrepresented_source_pair`

确保三种 source pair 都有足够黄金样本，而不是人工预算被数据量最大的来源对吃光。

## 9.4 前若干轮不要依赖图

直接借鉴 `simplify_qs()`：最初图结构还很稀疏时，先用：

```text
source-pair stratified uncertainty
+ hard-negative sampling
+ rule-conflict sampling
```

等每个来源对都有正/负 seed 后，再启动 graph disagreement。

这里不必机械照搬“20”这个常数，应该变成配置：

```text
min_gold_per_source_pair
min_positive_per_source_pair
min_negative_per_source_pair
```

满足后才进入 graph-boosted phase。

---

# 10. 两张图的生产实现

## 10.1 `G_truth`：不能被模型污染

边类型：

```text
HARD_POSITIVE_REFERENCE
HARD_POSITIVE_MANUAL
HARD_NEGATIVE_REFERENCE_CONFLICT
HARD_NEGATIVE_MANUAL
```

只有 HARD_POSITIVE 可以进入 entity component。

## 10.2 `G_review`：允许软边

边类型：

```text
SOFT_TEXT_SIMILARITY
SOFT_IMAGE_SIMILARITY
SOFT_OCR_CANDIDATE
SOFT_MODEL_MATCH
SOFT_PARTIAL_REFERENCE
```

每条边带：

```text
model_version
score
features_snapshot
reason_codes
created_at
```

## 10.3 Component invariant

每一个真实 component 必须满足：

```text
component 内所有 hard canonical reference 完全一致
```

可进一步保存：

```text
component.canonical_brand_id
component.canonical_reference
component.reference_rule_version
```

任何待 union 的新节点，如果携带不同 reference：

```text
拒绝 union + 创建 REVIEW_CONFLICT
```

这比事后靠模型置信度发现错误安全得多。

---

# 11. 用 DSU / Component Table 替代全局 NetworkX

千万级规模不建议维护一个 NetworkX 全球图。

真实合并图本质上只需要动态连通性，可以使用：

- Disjoint Set Union（Union-Find）；或
- 数据库 `entity_component_id`；或
- KV/关系库中的 component parent。

复杂图算法只在冲突局部执行。

## 11.1 Safe Union

```python
def union_if_safe(node_a, node_b):
    ca = find_component(node_a)
    cb = find_component(node_b)

    if ca.id == cb.id:
        return ca.id

    if ca.brand_id != cb.brand_id:
        raise Quarantine("brand conflict")

    if ca.canonical_ref != cb.canonical_ref:
        raise Quarantine("reference conflict")

    if explicit_negative_exists(ca.id, cb.id):
        raise Quarantine("negative constraint")

    return union(ca, cb)
```

这个 invariant 让误匹配变成“结构上无法发生”，而不是“希望模型不要犯错”。

## 11.2 Minimum-cut 只用于局部软图审核

当存在：

```text
manual/hard negative: A != C
soft graph path: A ~ B ~ C
```

拉取 A/C 所在的小型 `G_review` 局部子图，再运行 minimum-cut，输出：

```text
最值得复核/删除的 soft edges
```

不要对 `G_truth` 自动执行 cut。

人工确认后：

- 若是 soft edge 错：标为 `REJECTED`；
- 若是 reference extraction 错：修正 evidence；
- 若是规则问题：进入 brand rule regression case。

这就是把 ALMSER 的 minimum-cut 从“自动图修复”改成“可解释的审核建议器”。

---

# 12. 数据库 Schema 建议

## 12.1 `product_record`

```sql
CREATE TABLE product_record (
    id BIGINT PRIMARY KEY,
    source SMALLINT NOT NULL,
    source_item_id TEXT NOT NULL,
    content_hash TEXT NOT NULL,
    raw_title TEXT,
    raw_brand TEXT,
    canonical_brand_id BIGINT,
    product_type TEXT,
    raw_payload JSONB,
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ,
    UNIQUE (source, source_item_id)
);
```

## 12.2 `reference_evidence`

```sql
CREATE TABLE reference_evidence (
    id BIGINT PRIMARY KEY,
    record_id BIGINT NOT NULL,
    raw_value TEXT NOT NULL,
    canonical_value TEXT,
    evidence_type TEXT NOT NULL,
    identifier_role TEXT NOT NULL,
    confidence DOUBLE PRECISION,
    rule_version TEXT,
    is_hard BOOLEAN NOT NULL DEFAULT FALSE,
    conflict_group TEXT,
    metadata JSONB
);
```

高频索引：

```sql
(canonical_brand_id, canonical_value)
(record_id, is_hard)
```

## 12.3 `reference_entity`

```sql
CREATE TABLE reference_entity (
    id BIGINT PRIMARY KEY,
    canonical_brand_id BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    normalizer_version TEXT NOT NULL,
    status TEXT NOT NULL,
    UNIQUE (canonical_brand_id, canonical_reference)
);
```

## 12.4 `match_edge`

```sql
CREATE TABLE match_edge (
    left_record_id BIGINT NOT NULL,
    right_record_id BIGINT NOT NULL,
    edge_type TEXT NOT NULL,
    status TEXT NOT NULL,
    score DOUBLE PRECISION,
    model_version TEXT,
    reason_codes JSONB,
    created_at TIMESTAMPTZ,
    PRIMARY KEY (left_record_id, right_record_id, edge_type)
);
```

`edge_type` 至少区分：

```text
HARD_POSITIVE
HARD_NEGATIVE
SOFT_CANDIDATE
REJECTED
```

## 12.5 `entity_component`

```sql
CREATE TABLE entity_component (
    component_id BIGINT PRIMARY KEY,
    canonical_brand_id BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    state TEXT NOT NULL,
    member_count BIGINT NOT NULL,
    version BIGINT NOT NULL
);
```

## 12.6 `review_task`

```sql
CREATE TABLE review_task (
    id BIGINT PRIMARY KEY,
    left_record_id BIGINT,
    right_record_id BIGINT,
    task_type TEXT,
    priority DOUBLE PRECISION,
    reason_codes JSONB,
    model_snapshot JSONB,
    status TEXT,
    reviewer_label TEXT,
    created_at TIMESTAMPTZ,
    reviewed_at TIMESTAMPTZ
);
```

这张表就是 ALMSER active learner 的线上落点。

---

# 13. 千万级 Candidate Generation：不要全量 pairwise

既然最终条件是 canonical reference equality，最优候选生成优先级应该是：

## Tier 0：Hard exact key

```text
(canonical_brand_id, canonical_reference)
```

直接倒排/哈希查找。

复杂度接近：

```text
O(N)
```

而不是 `O(N²)`。

## Tier 1：Reference candidate retrieval

用于 reference 不完整/有 OCR 噪声的 review 场景：

```text
brand + reference prefix
brand + reference ngram
brand + catalog alias
```

只产生少量候选，不自动 merge。

## Tier 2：文本/图像 embedding

仅对 reference 抽取失败的尾部数据启用：

```text
brand filter
-> title embedding ANN
-> image embedding ANN
-> top-K review candidates
```

结果进入 `G_review`，不进入 `G_truth`。

这能把最昂贵的多模态计算限制在疑难长尾，而不是对千万级全量数据跑。

---

# 14. 图片应该怎么用

Spec 说有图片，但图片不能越权替代 reference。

优先顺序建议：

## 第一用途：OCR 提取 reference

重点图像：

```text
表背
保卡
吊牌
证书
包装标签
```

OCR 产出必须再经过 brand-specific reference validator。

## 第二用途：冲突否决

例如文本抽到 `126610LN`，但图片 OCR 明确读到 `116610LN`：

```text
ABSTAIN + REVIEW
```

而不是“文本分更高所以忽略 OCR”。

## 第三用途：审核排序

image embedding 可以帮助找到视觉近邻，特别适合构造 hard negative：

```text
外观极像但 reference 不同
```

这类样本正好是主动学习最该标的数据。

## 不推荐用途

```text
image similarity > 0.99 => MATCH
```

腕表同系列不同 reference 外观可能非常接近，这和 Spec 定义直接冲突。

---

# 15. 如何把几百条黄金标签用到最值

建议不把人工标签只定义成 pair `MATCH/NO_MATCH`，而是同时采集错误类型：

```text
SAME_REFERENCE
DIFFERENT_REFERENCE
WRONG_BRAND
REFERENCE_EXTRACTION_ERROR
PLATFORM_SKU_MISCLASSIFIED
ACCESSORY_NOT_WATCH
OCR_ERROR
AMBIGUOUS
```

这样一条人工反馈可以同时改：

- pair model；
- identifier role classifier；
- title extractor；
- brand normalizer；
- OCR post-processing；
- review sampling policy。

比只训练一个二分类器的数据利用率高得多。

## 15.1 黄金标签优先队列

建议比例由线上错误分布动态调整，但初始可以按风险桶组织：

```text
A. model=MATCH && hard_rule=REJECT/ABSTAIN
B. graph path creates reference conflict
C. same brand + one-character-different reference
D. structured field vs title/OCR conflict
E. source-pair under-covered samples
F. random audit sample
```

A/B/C 优先级最高，因为它们与 false positive 直接相关。

---

# 16. 模型在系统里应该承担什么角色

## 16.1 可以做

模型适合：

```text
reference span extraction
identifier role classification
product_type classification
candidate ranking
review priority scoring
OCR correction candidate generation
soft edge prediction
```

## 16.2 不应该做

模型不应该直接输出：

```text
MERGE
```

最终 merge 应由 deterministic gate + component invariant 决定。

## 16.3 推荐模型选择

第一版不需要大模型：

- 结构化/标题抽取：规则 + CRF/小型 token classifier；
- pair risk scorer：LightGBM / XGBoost；
- 文本向量：小型中文/多语 embedding；
- OCR：成熟 OCR 模型；
- 图片：CLIP 类 embedding 仅做弱证据；
- LLM：仅处理低频疑难抽取并要求结构化输出，默认 weak evidence。

ALMSER 的价值就在于它对 classifier 本身不强绑定，可以渐进替换。

---

# 17. 增量数据处理流程

每条新增/更新 listing：

```text
1. 幂等 ingestion
2. brand canonicalization
3. product_type / identifier role 识别
4. 结构化字段 reference 提取
5. title reference 提取
6. 必要时 OCR
7. 建立 evidence set
8. canonical reference resolution
9. hard gate
10a. 足够强 -> 查 reference_entity -> safe union
10b. 不足 -> 生成 G_review candidate
11. component invariant audit
12. 异常进入 review_task
```

幂等键建议：

```text
(source, source_item_id, content_hash)
```

如果内容没变化，不重新跑昂贵模型。

如果 normalization rule / model 升级，通过：

```text
rule_version / model_version
```

触发定向重算，而不是全库无条件回刷。

---

# 18. 服务拆分建议

```text
source-adapter
brand-resolver
identifier-role-resolver
reference-extractor
reference-normalizer
ocr-worker
candidate-index
precision-gate
graph-audit-service
active-learning-sampler
review-api
review-ui
evaluator
```

### 存储建议

第一版就能落地的组合：

```text
PostgreSQL：canonical reference、record state、review、hard edge/component
对象存储：原图/OCR 中间产物
Redis：短期任务/缓存（可选）
消息队列：Kafka/RabbitMQ/Celery 任选其一
```

如果事件量和分析量继续上升：

```text
ClickHouse：大规模 evidence / candidate / audit 日志分析
OpenSearch：reference fuzzy retrieval / debug search
FAISS/Milvus：只服务 tail candidate retrieval
```

不建议第一版因为“1000 万”就直接堆复杂向量数据库；exact reference index 本身非常便宜，先把 hardest problem 放在 extraction 和 safety gate 上。

---

# 19. 评测指标必须从 F1 改成 Precision-First

原 ALMSER 主要报告 Precision / Recall / F1；本项目上线门槛要改。

## 19.1 核心指标

```text
Auto-Merge Precision
False Positive Count
False Positive Rate
Coverage / Auto-Merge Rate
Abstain Rate
Reference Extraction Precision
Reference Extraction Coverage
```

其中优先级：

```text
Auto-Merge Precision >>> Coverage
```

## 19.2 分桶评测

必须按：

```text
brand
source pair
reference evidence type
product type
新旧批次
是否 OCR
是否 title-only
是否出现 conflict
```

分别观察。

一个全局 99.99% 可能掩盖某个新来源/品牌只有 96%。

## 19.3 Golden set 要故意包含 hard negative

不能随机切分。

必须人为加入：

```text
同系列不同 reference
一字符差异 reference
同图/官图复用
配件引用主体表型号
平台 SKU 像 reference
OCR 0/O、1/I、5/S 混淆
标题多个 reference
不同年代相邻型号
```

否则评测很容易虚高。

---

# 20. 发布安全机制

## 20.1 三态输出，而不是二态

```text
MATCH
NON_MATCH
ABSTAIN
```

系统有权说“不知道”。

Spec 已明确允许漏匹配，因此 ABSTAIN 是设计特性，不是失败。

## 20.2 Shadow Mode

新规则/新模型先只写：

```text
proposed_decision
```

不改变正式 component。

与当前 hard gate 对比，积累黄金审计后再升级。

## 20.3 Kill Switch

按：

```text
brand
source
source_pair
normalizer_version
model_version
```

都应该能一键关闭 auto-merge，只退回 review。

## 20.4 可回滚 lineage

任何 merge 必须知道：

```text
哪条 evidence
哪个 normalizer version
哪个 rule
哪个人工 label
哪个 component version
```

这样发生错误时可以定向拆分/重算。

---

# 21. ALMSER 原源码到生产模块的映射

| ALMSER 原模块 | 原作用 | 当前建议落点 |
|---|---|---|
| `unsupervised_label` | 冷启动弱标签 | reference hard rules / verified seeds |
| `agg_score` | bootstrap 极值 | 不再直接判正；用于 review sampling |
| `datasource_pair` | 多来源 task 标识 | 保留，做 source-pair calibration / coverage |
| `learning_models['all']` | 主 pair classifier | risk/review scorer |
| `constructGraphFromWeightedPredictions` | 构造 correspondence graph | 只构造 `G_review` |
| manual positive edge | 强正边 | `G_truth` HARD_POSITIVE |
| manual negative | 图约束 | HARD_NEGATIVE / cannot-link |
| `minimum_cut` | 修复冲突路径 | 局部 soft-edge 审核建议 |
| `graph_inferred_label` | 弱标签 | 只允许训练 soft scorer，不做 merge truth |
| clean component augmentation | 扩充训练集 | weak training labels，且需 reference consistency |
| graph-model disagreement | 主动学习 query | **直接保留，作为核心 review priority** |
| task transfer/group | 多来源复用 | source-pair/品牌 task sampling |
| NetworkX global graph | 实验图 | DSU + DB component + local NetworkX |

---

# 22. 可以直接实施的第一版方案

如果现在开始落地，我建议按下面顺序，不需要先做复杂端到端 matcher。

## Milestone A：Reference-first deterministic baseline

交付：

```text
brand resolver
identifier role resolver
reference normalizer
reference evidence table
reference_entity table
precision hard gate
```

先只自动处理：

```text
结构化 reference 明确
+ 品牌明确
+ canonical reference exact match
+ 无冲突
```

这一阶段就可以吃掉最安全的一批数据，而且 precision 最可控。

## Milestone B：标题抽取 + 主动学习审核

加入：

```text
title reference extractor
hard-negative generator
review_task
ALMSER-style disagreement sampler
```

让几百条黄金标签优先修正：

```text
字段角色误判
标题多型号
同系列相邻型号
来源字段差异
```

## Milestone C：G_truth / G_review 双图

加入：

```text
DSU/component service
hard negative constraints
soft graph
local min-cut audit
component reference invariant
```

重点不是“提高 recall”，而是多一道防错保险。

## Milestone D：OCR / 图片

只对 reference 缺失或冲突 listing 启动：

```text
OCR
image candidate retrieval
visual hard-negative mining
```

图片永远不直接放行。

## Milestone E：模型迭代

随着黄金标签积累，再训练：

```text
identifier role classifier
reference span extractor
pair risk scorer
source-pair calibrated review model
```

模型性能提高只会让：

```text
更多样本从 ABSTAIN 进入“可验证证据”
```

而不会放松最终 hard gate。

---

# 23. 关键伪代码：把“误匹配不允许发生”写进架构

## 23.1 Resolve Reference

```python
def resolve_reference(record):
    evidences = []

    evidences += extract_verified_structured_fields(record)
    evidences += extract_title_candidates(record)

    if not has_sufficient_hard_evidence(evidences):
        evidences += extract_ocr_candidates(record.images)

    normalized = [
        canonicalize(record.brand_id, e)
        for e in evidences
    ]

    hard = [e for e in normalized if e.valid and e.is_hard]

    if conflicting_hard_references(hard):
        return Abstain("HARD_REFERENCE_CONFLICT", evidences=normalized)

    best = choose_single_hard_reference(hard)
    if best is None:
        return Abstain("NO_HARD_REFERENCE", evidences=normalized)

    return ResolvedReference(best.canonical, evidences=normalized)
```

## 23.2 Upsert to Reference Entity

```python
def link_record(record):
    ref = resolve_reference(record)

    if ref.is_abstain:
        enqueue_review(record, ref.reason)
        return

    entity = get_or_create_reference_entity(
        brand_id=record.brand_id,
        canonical_ref=ref.canonical,
    )

    if component_invariant_safe(record, entity):
        attach_to_entity(record, entity)
    else:
        quarantine(record, "COMPONENT_INVARIANT_CONFLICT")
```

## 23.3 Active Learning Review Selection

```python
def review_priority(pair):
    return (
        5.0 * pair.model_match_but_hard_rule_reject
        + 4.0 * pair.graph_reference_conflict
        + 3.0 * pair.one_char_reference_difference
        + 2.0 * pair.structured_title_ocr_conflict
        + 1.5 * pair.model_uncertainty
        + 1.0 * pair.source_pair_undercoverage
    )
```

这比“找最不确定样本”更符合业务：直接按潜在 false-positive 损失排序。

---

# 24. ALMSER 对当前需求最值得保留的三个思想

## 24.1 多源不能只独立做 pairwise

三源之间存在结构信息。

即使最终 match 由 reference hard gate 决定，图仍能发现：

```text
某个 listing 被不同来源证据拉向两个 reference
```

这类异常非常适合做审核告警。

## 24.2 人工标签应采“结构冲突”，不是随机样本

在几百条预算下，ALMSER 最大贡献是 query strategy。

对于 precision-first 项目，最有价值的 query 不是 entropy 最大，而是：

```text
最可能造成错误自动合并的边
```

## 24.3 图推断适合增强“学习”，不适合直接增强“事实”

这一点必须和原论文做主动区分。

我们可以让图推断标签训练：

```text
candidate ranker
risk scorer
identifier-role model
```

但业务事实仍只有：

```text
可验证 reference evidence
+ manual gold
```

---

# 25. 最终推荐

**推荐采用 ALMSER-GB 的多源图增强主动学习思想，但不要直接部署其 matcher/全局图推断机制。**

针对当前 Spec，最优落地形态是：

```text
Canonical Reference Registry
+ Evidence-first Reference Extraction
+ Precision Hard Gate
+ G_truth / G_review 双图隔离
+ DSU Component Invariant
+ ALMSER-style Graph Disagreement Active Learning
+ Human Review
+ OCR/Image 仅做辅助证据
```

最终自动合并的核心公式应非常简单：

```text
AUTO_MATCH =
    SAME_CANONICAL_BRAND
    AND SAME_CANONICAL_REFERENCE
    AND HARD_EVIDENCE_ON_BOTH_SIDES
    AND NO_STRONG_CONFLICT
    AND COMPONENT_INVARIANT_SAFE
```

其余所有模型能力——文本相似、图片相似、LLM、图传递、弱标签、cross-encoder——都只能帮助系统：

```text
找候选
找冲突
找最值得人工标的样本
```

而不能越过这一公式。

这样既能利用 ALMSER 在“多源 + 少量标签”上的核心优势，又把最危险的图传递 false positive 从业务真值路径中彻底隔离，才真正满足“允许漏匹配，但绝不能误匹配”的需求。

---

## 参考资料

- ALMSER-GB GitHub：https://github.com/wbsg-uni-mannheim/ALMSER-GB
- `README.md`：项目运行方式、实验配置与代码组织
- `code/ALMSER.py`：bootstrap、学习模型、图增强训练、source-pair/task transfer
- `code/ALMSER_EXP.py`：active learning loop、query strategy、graph-model disagreement sampling
- `code/graphutils.py`：correspondence graph、bridge removal、negative constraint、minimum cut
- `code/ALMSER_utils.py`：query strategy criteria、disagreement score
- `code/learningutils.py`：RandomForest/SVM/GBDT 等基础 learner 与跨 task transfer evaluation
- 论文：Anna Primpeli, Christian Bizer, *Graph-boosted Active Learning for Multi-Source Entity Resolution*, ISWC 2021
