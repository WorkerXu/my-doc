# Efficient Model Repository for Entity Resolution: Construction, Search, and Integration

## 0. 结论先行

本文选择调研清单中的 **Efficient Model Repository for Entity Resolution: Construction, Search, and Integration（MoRER）**。在开始前已检查 `奢侈品调研/b`，未发现同标题或 MoRER 的既有分析。

MoRER 的核心思想不是再训练一个“更强的 pairwise matcher”，而是把多源实体匹配拆成很多 **ER task（来源对之间的匹配任务）**，先比较这些任务的特征分布，再把分布相近的任务聚成簇；每个簇只维护一个分类模型，新增来源时先判断新任务属于哪个簇，再复用已有模型，出现分布漂移时才重聚类、补标注、重训。

这个思路与当前 Spec 的“多来源、持续增量、少量黄金标签”高度契合，但 **不能原样用 MoRER 的分类结果做最终商品合并**，原因有三点：

1. 当前需求把“同一个商品”明确定义为 **同一 reference number / 型号**，这实际上比通用 ER 更强、更可解释，应该优先做 reference 抽取、规范化与严格等值归并，而不是让模型从标题/图片相似度推断“像不像同款”。
2. Spec 要求 **绝对不能误匹配，precision 优先到极致**。MoRER 论文优化的是整体 Precision/Recall/F1，并非零误合并；在 WDC-computer 等数据上部分配置的 precision 只有 0.84–0.95，不能直接成为自动合并器。
3. 当前只有雷小安、腕表之家、奢当家三个来源，如果把一个 source-pair 当一个 ER task，一共只有 3 个跨源任务，MoRER 的“任务聚类/模型仓库”价值不大。要落地，必须把 task 粒度改成 **来源对 × 品牌/系列 × 字段覆盖模式 × 抽取器版本 × 时间窗口**，把 MoRER 从“pair matcher”改造成 **分布感知的策略/模型路由器 + 漂移检测器**。

因此建议直接落地成下面的架构：

> **Reference-first 硬规则做最终裁决；MoRER 的“任务分布分析 → 图聚类 → 模型/策略复用 → 漂移重聚类”用于选择 reference 抽取器、验证器和阈值策略。最终只输出 MATCH / NON_MATCH / ABSTAIN 三态，任何模型都无权越过 reference 硬证据把 ABSTAIN 升级成 MATCH。**

这样既保留 MoRER 对多源增量和少标注的优势，又能满足“宁可漏，不可错”的生产要求。

---

## 1. 调研对象

- 论文：**Efficient Model Repository for Entity Resolution: Construction, Search, and Integration**
- 作者：Victor Christen, Peter Christen
- 会议：EDBT 2026
- 论文页：https://dbs.uni-leipzig.de/research/publications/efficient-model-repository-for-entity-resolution-construction-search-and
- PDF：https://openproceedings.org/2026/conf/edbt/paper-245.pdf
- 官方代码：https://github.com/vicolinho/Model_Reuse_ER
- 当前需求 Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31

论文解决的问题是：多源 ER 中，数据源不断增加后，来源对数量快速增长，而每个来源对都单独标注、训练一个分类器会越来越贵；同时不同来源对的 similarity feature distribution 差异很大，直接把所有历史任务合并成一个大训练集也不一定合理。

MoRER 因此维护一个“ER Model Repository”：

```text
多个已解决 ER task
      │
      ├── 计算 similarity feature distribution
      │
      ├── 比较 task 与 task 的分布相似度
      │
      ▼
ER task similarity graph
      │
      ▼
Leiden community detection
      │
      ▼
多个 task cluster
      │
      ├── 每簇分配少量标注预算
      ├── Active Learning / Supervised
      ▼
每簇一个 classifier
      │
      ▼
ER Model Repository
      │
      ├── 新 task -> 选择最相似 cluster -> 复用模型
      └── 漂移明显 -> 重聚类 + 补标 + 重训
```

论文在三个多源数据集上评估：Dexter 有 23 个来源、276 个 ER problem、约 110 万 candidate pairs；WDC-computer 有 12 个 ER problem；Music 有 20 个 ER problem。代码实现基于 Python 3.12、scikit-learn、NetworkX，同时使用 `igraph + leidenalg` 做 Leiden 聚类。

---

## 2. MoRER 的核心技术实现

## 2.1 ER task 不是“记录”，而是一整批来源对候选

论文定义一个 ER problem / ER task `p(k,l)` 为来源 `Dk` 和 `Dl` 之间的候选记录对集合。每个候选 pair 被转换成 similarity feature vector：

```text
w = [sim_title, sim_brand, sim_model, sim_price, ...]
```

例如：

```text
A1-B1 -> [0.95, 1.00, 1.00, 1.00]
A2-B2 -> [1.00, 1.00, 0.50, 1.00]
A3-B4 -> [0.20, 0.00, 0.00, 0.50]
```

MoRER 真正比较的是：

> “两个来源对任务内部的 feature 分布是否相似？”

而不是直接比较某两个商品。

这一点非常关键。它把“模型复用”从 record-level 转成 task-level：如果 `雷小安 × 腕表之家 / Rolex` 与 `雷小安 × 奢当家 / Rolex` 的候选 feature 分布高度类似，就可以共享一套抽取/验证策略；如果某个新品牌的数据分布明显不同，就不应该硬套旧模型。

---

## 2.2 Similarity Distribution Analysis

MoRER 对 ER task 做分布比较，论文和代码支持多种统计方法。

### 2.2.1 Kolmogorov–Smirnov（KS）

对每个 feature 单独比较经验 CDF，判断两个任务在该 feature 上是否可能来自相同分布。

代码 `morer/reuse/statistic/statistical_tests.py` 中直接使用：

```python
from scipy.stats import ks_2samp
ks_stat, p_value = ks_2samp(col_array_1, col_array_2)
```

论文默认配置也把 KS 作为主要分布测试之一。

### 2.2.2 Wasserstein Distance

比较两组 feature 分布之间的“搬运距离”。代码直接调用：

```python
wasserstein_distance(col_array_1, col_array_2)
```

Wasserstein 对“整体分布位置发生变化”较直观，但论文实验指出其结果对数据集/AL 配置更敏感。

### 2.2.3 PSI（Population Stability Index）

PSI 适合检测 covariate shift。代码会先将 `-1` 等缺失占位转换，再计算并缩放 PSI。论文实验认为 PSI 在噪声或多样数据上较稳定。

### 2.2.4 Classifier Two-Sample Test（C2ST）

多变量版本不是逐列比较，而是把“来自 task A / task B”作为二分类标签，训练 XGBoost：

```text
X = similarity feature vectors
Y = task_A / task_B
```

如果分类器非常容易区分 A/B，说明两个 task 分布差异大；如果很难区分，则分布相似。

代码中使用 `XGBClassifier + 5-fold cross validation`，并用 F1/accuracy 作为区分能力指标。

### 2.2.5 feature 权重

MoRER 不简单平均所有 feature，而是用 feature 的标准差近似其判别力：

```python
all_sims = np.vstack(linkage_problems_numpy_arrays)
weights = np.std(all_sims, axis=0)
```

直觉是：一个 feature 如果几乎永远是常数，就不该决定两个 task 是否相似；变化更有区分度的 feature 权重更高。

这对当前腕表场景可以直接借鉴，但 feature 必须重做，不能照搬通用 title/price similarity。

---

## 2.3 ER Problem Similarity Graph

得到 task 两两相似度后，MoRER 构图：

```text
vertex = 一个 ER task
edge   = 两个 task 的分布相似度
weight = avg_similarity
distance = 1 - weight
```

官方代码 `morer/reuse/utils.py` 的 `create_graph()` 就是这个逻辑。

之后用 Leiden 做 community detection。`graph_clustering.py` 会先把 NetworkX graph 转成 igraph，然后：

```python
partition = leidenalg.find_partition(
    g,
    leidenalg.ModularityVertexPartition,
    n_iterations=-1,
    weights='weight'
)
```

作者也实现了 Louvain、Girvan-Newman、Label Propagation 作为替代，但论文选择 Leiden，主要因为：

- 能找出 weakly connected component 中内部连接更强的子群；
- 扩展到较多 ER task 时效率较好；
- 比“所有任务只用一个统一模型”更能保留来源异质性。

对当前需求，真正值得抄的不是 Leiden 本身，而是这个“先根据运行数据自动发现 task family，再绑定策略”的思想。

---

## 2.4 每个 cluster 只维护一个模型

聚类后，每个 cluster `Ci` 生成一个 `MCi`。

论文支持：

- Supervised
- Bootstrap Active Learning
- Almser Active Learning

官方代码默认：

```text
model_generation = bootstrap
min_budget = 50
total_budget = 1000
batch_size = 5
bootstrap k = 100
```

`model_generation.py` 里会先给每个 cluster 一个最低标注预算，再按 cluster 内 candidate vector 数量做比例分配。

一个重要工程细节是：

> **训练数据来自 cluster 内多个相似 ER task，而不是只取 cluster leader。**

代码会把同簇 task 的 pair/vector 合并，再用 Active Learning 挑样本；最终用 `active_learning_solution.train_model()` 训练分类器。论文实验主要以 Random Forest 类模型为传统 ML 基线。

这与当前“只愿意标几百对”的条件非常合适：黄金标签应该优先投到“高风险分布簇”里，而不是平均随机抽样。

---

## 2.5 新 task 的模型搜索

MoRER 有两类策略。

### `sel_base`

假设没有明显 domain shift：

1. 新 task 计算 similarity feature distribution；
2. 与每个 cluster 的训练数据分布比较；
3. 选择最相似 cluster；
4. 直接复用该 cluster 的 classifier。

官方代码的非 recluster 模式就是这个方向：

```text
new task
  -> determine_best_cluster(...)
  -> selected_task / selected model
  -> classify()
```

### `sel_cov`

处理持续增量和分布漂移：

1. 新 task 加入 task similarity graph；
2. 重新做 Leiden clustering；
3. 看 cluster 的成员关系是否变化；
4. 计算未被训练数据覆盖的新 feature vector 占比 `coverage ratio`；
5. 超过阈值 `t_cov` 时补标、重训。

论文测试了 `t_cov = 0.1 / 0.25 / 0.5`。官方 README 默认 `unsolved_ratio=0.25`。

核心公式可理解成：

```text
coverage_ratio = 新/未覆盖 vector 数量 / 当前 cluster 全部 vector 数量
```

代码 `main.py` 中会：

```python
covered_ratio = unsolved_lps_count / (unsolved_lps_count + solved_lps_count)
if covered_ratio > UNSOLVED_RATIO:
    # allocate new labeling budget
    # select new training data
    # retrain model
```

这正适合持续爬取的数据：新品牌、标题模板变化、平台改版、OCR 模型升级都可能触发分布漂移。

---

## 2.6 官方代码结构与真实工程流程

官方仓库的主要目录：

```text
morer/
  reuse/
    main.py
    graph_clustering.py
    model_generation.py
    model_selection.py
    utils.py
    incremental/
    statistic/
      statistical_tests.py
      psi_computation.py
record_linkage/
baseline/
datasets/
data/linkage_problems/
```

`main.py` 基本就是论文 Figure 3 的可运行版本：

```text
Step 1 读取/构建 linkage problem
Step 2 对 solved task 做 distribution test
Step 3 建 task similarity graph
Step 4 Leiden clustering
Step 5 cluster 内做 active learning + model generation
Step 6 新 task 选择模型
Step 7 可选 reclustering + retrain
Step 8 evaluation
```

README 给出的运行示例：

```bash
python ./morer/reuse/main.py \
  -d ./datasets/dexter/DS-C0/SW_0.3 \
  -l ./data/linkage_problems/dexter/ \
  -mg bootstrap \
  -tb 1000
```

带增量重聚类：

```bash
python ./morer/reuse/main.py \
  -d ./datasets/dexter/DS-C0/SW_0.3 \
  -l ./data/linkage_problems/dexter/ \
  -mg bootstrap \
  -tb 1000 \
  -rec True \
  -uns_ratio 0.25
```

---

## 2.7 官方代码是研究原型，不应直接上线

代码非常适合作为算法参考，但有明显 research prototype 特征：

- `graph_clustering.py` 中存在本机绝对路径 `sys.path.append('/Users/...')`；
- 多处用 `eval(task_string)` 还原 task tuple，不适合生产环境；
- `main.py` 是脚本式流程，状态主要放 Python dict / pickle / CSV；
- 新 task 处理、重聚类、重训都在单进程流程里完成；
- 没有版本化 model registry、feature schema registry、审计日志和回滚；
- 评价逻辑面向 F1，而不是“自动放行零误匹配”；
- task similarity 的阈值、缺失值处理、统计检验都有不少实验代码痕迹，需要重写测试。

因此建议 **借思想，不 fork 后直接改**。

---

## 3. 论文结果对当前 Spec 的真实启示

## 3.1 优势：很适合“持续加入新数据源/新分布”

论文最有价值的结论不是某个模型的 F1，而是：

> 当 ER task 很多时，把相似 task 先聚成簇，再在簇内做训练数据选择和模型复用，可以明显减少 Active Learning 搜索空间与训练成本。

论文中，MoRER 的 statistical analysis / clustering / model selection 额外开销相对较小，在不同数据集上是秒级到百秒级；相比 Almser 直接在更大相似图上做 AL，部分配置能获得数倍速度提升。

对 100 万–1000 万级历史商品，真正要避免的是“全量 candidate pair + 一个全局模型”的做法。

---

## 3.2 局限：它没有解决“绝对不能误匹配”

论文 Table 4 的部分典型 precision：

```text
Dexter / MoRER+Almser   ~0.96
WDC / MoRER+Almser      0.84 ~ 0.95
Music / MoRER+Almser    ~0.98
```

这些数字在通用 ER benchmark 上不错，但对于当前业务的“绝对不能误匹配”并不够。

如果 1000 万条记录中真正参与自动合并的 pair 达到百万级，即使 99.9% precision 也可能产生大量误合并；更重要的是，**几百条人工黄金标签根本无法统计上证明 99.99% 或 100% precision**。

所以“绝不能误匹配”不能靠 ML 指标保证，必须靠：

1. 更强的业务定义；
2. 硬证据；
3. 强拒识；
4. 任何冲突直接否决；
5. 全链路审计。

当前 Spec 已经给了最关键的业务定义：**same product = same reference**。这应该成为系统最高优先级 invariant。

---

## 3.3 局限：三个来源直接套 MoRER，task 太少

三个来源只有：

```text
雷小安 × 腕表之家
雷小安 × 奢当家
腕表之家 × 奢当家
```

如果 task 就定义到这里，只有 3 个节点，做 Leiden 几乎没有意义。

因此建议把 task 改成：

```text
TaskProfile =
  source_pair
× canonical_brand / brand_family
× field_coverage_profile
× extraction_channel
× parser_version
× time_window
```

例如：

```text
(雷小安, 腕表之家, Rolex, explicit_ref, parser_v3, 2026-W33)
(雷小安, 腕表之家, Rolex, title_only, parser_v3, 2026-W33)
(雷小安, 腕表之家, Cartier, title+ocr, parser_v2, 2026-W33)
(腕表之家, 奢当家, Omega, title_only, parser_v3, 2026-W34)
```

这样 task graph 才能表达真实的“分布族群”和“漂移”。

---

## 4. 针对当前 Spec 的建议架构：RefRoute

可以把落地方案称为 **RefRoute：Reference-first + Distribution-aware Routing**。

总体架构：

```text
                    ┌────────────────────┐
三源原始商品数据 ──> │ Raw / Normalized   │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │ Brand Normalizer   │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │ Reference Extractor│
                    │ field/title/OCR    │
                    └─────────┬──────────┘
                              │
                    reference observations
                              │
            ┌─────────────────┴─────────────────┐
            ▼                                   ▼
┌──────────────────────┐            ┌────────────────────────┐
│ Reference Canonicalizer│          │ Task Profile Builder   │
│ role + format + catalog│          │ distributions / drift  │
└───────────┬──────────┘            └────────────┬───────────┘
            │                                    │
            │                         task similarity graph
            │                                    │
            │                                    ▼
            │                         ┌────────────────────────┐
            │                         │ Leiden Cluster / Router│
            │                         └────────────┬───────────┘
            │                                      │
            │                           extractor/policy/model
            │                                      │
            └──────────────────┬───────────────────┘
                               ▼
                    ┌──────────────────────┐
                    │ Precision Gate       │
                    │ MATCH/NON/ABSTAIN    │
                    └──────────┬───────────┘
                               │
                    ┌──────────┴───────────┐
                    ▼                      ▼
              exact ref index        review queue
                    │                      │
                    └──────────┬───────────┘
                               ▼
                        label / feedback
```

MoRER 负责中间右侧“Task Profile → Cluster → Policy/Model Routing”，**左侧 reference-first 链路才是最终判同的主线**。

---

## 5. 第一步：先把 reference 从“字符串”升级成结构化证据

不能把标题里任何“像型号”的字符串都当 reference，否则最容易出现 false positive。

建议记录多条 `reference_observation`：

```text
record_id
source
brand_id
raw_value
canonical_value
role
channel
extractor_version
confidence
span
image_id
is_conflicting
created_at
```

其中 `role` 至少分：

```text
PRODUCT_REFERENCE       品牌官方 reference / 型号
PLATFORM_SKU            平台货号
SELLER_SKU              商家内部货号
SERIAL_NUMBER           单只序列号
ACCESSORY_COMPAT_REF     配件“适用于 xxx 型号”
UNKNOWN_IDENTIFIER      暂不能判定
```

这一步比“提高文本相似度”重要得多。

### 5.1 evidence channel

建议区分证据来源：

```text
EXPLICIT_FIELD
TITLE_REGEX
DESCRIPTION_REGEX
OCR_DIAL
OCR_CASEBACK
OCR_CARD
MANUAL
CATALOG_LOOKUP
```

生产决策时对 channel 做不同信任级别。

例如：

```text
explicit reference field + brand format valid
    > title regex
    > OCR from warranty card
    > generic OCR
```

模型可帮助抽取，但必须把“抽到了什么、从哪里抽到、角色是什么”保留下来，不能只保存一个最终字符串。

---

## 6. Reference Canonicalization：宁可少归一，不可过归一

建议 canonical key：

```text
(canonical_brand_id, canonical_reference)
```

但 canonicalization 只能做 **可证明等价** 的变化：

```text
trim
unicode normalization
uppercase
delete approved whitespace
normalize approved separators
brand-specific alias mapping
```

例如：

```text
"126 334"     -> "126334"
"126-334"     -> 仅当 Rolex 规则确认后 -> "126334"
"IW 5007-05"  -> 按 IWC 专用规则规范
```

不能做：

```text
编辑距离很近 -> 当成同 reference
只差一个字符 -> 自动纠正
数字前导零随意去掉
跨品牌共用一套 strip-all-punctuation 规则
```

因为腕表最危险的 hard negative 恰恰就是：

```text
126300 vs 126334
116500LN vs 126500LN
同系列不同尺寸/材质/盘面 reference
```

外观可能很接近，字符串也可能只差一位，但业务定义上就是不同商品实体。

---

## 7. 最终匹配不做 N²：用 reference inverted index

既然 same product 定义为 same reference，历史全量不应该做跨源笛卡尔 pairwise matching。

### 7.1 历史批处理

对每条 record：

```text
normalize brand
-> extract reference observations
-> canonicalize
-> precision gate
-> emit verified canonical key
```

然后：

```sql
CREATE INDEX idx_ref_entity
ON product_record (canonical_brand_id, canonical_reference)
WHERE decision_state = 'VERIFIED_REFERENCE';
```

相同 key 的三源记录自然聚合到同一 reference entity。

复杂度从潜在的：

```text
O(N²)
```

降到接近：

```text
O(N * extraction_cost) + O(N log G)
```

其中 `G` 是 reference group 数量。

### 7.2 增量

每来一条新记录：

```text
parse -> canonical reference -> index lookup -> attach to entity
```

无需重新跑历史 pairwise matcher。

图片、LLM、embedding 只处理 reference 缺失/冲突的少数 ABSTAIN，不进入全量主链路。

---

## 8. MoRER 思路如何改造成 Task Profile Repository

## 8.1 Task Profile 的 feature 不应是“商品相似度”本身

推荐每个时间窗口/品牌/来源组合计算下面的分布：

### reference 抽取质量

```text
explicit_ref_rate
reference_extracted_rate
ambiguous_reference_rate
multi_reference_rate
reference_conflict_rate
unknown_identifier_rate
accessory_compatible_ref_rate
catalog_hit_rate
```

### 字符串形态

```text
reference_length
num_digit_ratio
num_alpha_ratio
separator_count
prefix_pattern
suffix_pattern
brand_format_valid
```

### 数据稀疏度

```text
has_title
has_brand
has_model_field
has_reference_field
has_description
has_images
has_ocr
```

### 候选验证分布

```text
brand_equal
reference_exact_equal
reference_role_conflict
series_equal
price_ratio
image_similarity
text_similarity
missingness_bitmap
```

其中：

> `reference_exact_equal` 是硬证据；text/image similarity 只能用于 veto、review priority 或辅助抽取，不能替代 reference equality。

---

## 8.2 Task Profile 相似度

第一版无需复制论文所有统计方法，建议：

```text
连续数值 feature：PSI + KS
类别/布尔 feature：Jensen-Shannon / PSI on bins
整体分布：可选 C2ST
```

然后做：

```text
profile_similarity = weighted average
```

权重不要完全照搬论文的“全局标准差”，建议手工把风险相关 feature 权重拉高：

```text
reference_conflict_rate         HIGH
multi_reference_rate            HIGH
accessory_compatible_ref_rate   HIGH
brand_format_valid              HIGH
catalog_hit_rate                HIGH
image_similarity                LOW
price_ratio                     LOW
```

这样 task router 会优先感知真正会导致 false merge 的漂移。

---

## 8.3 Cluster 对应的不是“最终 matcher”，而是 Policy Bundle

每个 cluster 维护：

```text
policy_bundle_id
brand_ruleset_version
reference_extractor_version
identifier_role_classifier_version
ocr_pipeline_version
validator_model_version
precision_gate_version
review_sampling_policy
```

例如：

```text
Cluster C1: Rolex + explicit field heavy
  -> regex_v7
  -> role_classifier_v3
  -> OCR disabled by default
  -> strict exact field equality

Cluster C2: Cartier + title-only
  -> title extractor v5
  -> catalog constrained parser
  -> multi-ref -> ABSTAIN

Cluster C3: mixed title + OCR
  -> OCR card/caseback
  -> require 2-channel agreement
```

这比“每簇一个二分类 matcher”更符合当前业务。

---

## 9. Precision Gate：把“绝不能错”落实成程序 invariant

建议最终输出：

```text
MATCH
NON_MATCH
ABSTAIN
```

### 9.1 自动 MATCH 的最小条件

第一版可定义：

```text
1. canonical_brand 已确认
2. 两条记录都存在唯一 PRODUCT_REFERENCE
3. canonical_reference 严格一致
4. reference format 对该品牌合法
5. 不存在第二个冲突 PRODUCT_REFERENCE
6. 不是 accessory / parts / box / strap / compatibility listing
7. 当前 Task Profile 未被 drift quarantine
```

如果 reference 来自风险更高的渠道，可再加：

```text
TITLE_REGEX -> 必须 catalog hit
OCR -> 必须与 title/field 中另一独立证据一致，或人工确认
```

### 9.2 自动 NON_MATCH

```text
同品牌 + 两边均有可信 PRODUCT_REFERENCE + reference 不相等
```

即可直接判 NON_MATCH。

### 9.3 ABSTAIN

以下全部拒识：

```text
reference missing
多个候选 reference
品牌不确定
explicit field 与 title/OCR 冲突
只出现 accessory compatible reference
只得到近似 reference
当前 parser/profile 发生异常漂移
模型置信度高但没有 hard reference evidence
```

**ABSTAIN 不是失败，而是 precision-first 系统的核心能力。**

---

## 10. 模型在系统里的权限边界

建议明确一个原则：

> **模型只能“补证据、归类、否决、排序”，不能凭相似度创造 reference 等价关系。**

模型可做：

```text
brand normalization
identifier role classification
reference span extraction
accessory detection
OCR post-correction under catalog constraints
review priority
conflict detection
```

模型不可做：

```text
126300 ~ 126334 -> MATCH
两个外观相似图片 -> MATCH
标题很像但 reference 缺失 -> MATCH
LLM 说“应该是同款” -> MATCH
```

这种权限设计比单纯调一个高阈值安全得多。

---

## 11. 一个容易误合并的典型例子

记录 A：

```text
来源：腕表之家
标题：劳力士日志型 126334 蓝盘
reference field: 126334
```

记录 B：

```text
来源：某二奢
标题：适配 Rolex Datejust 126334 的原装表带
reference field: null
```

如果系统只做：

```text
brand + title similarity + detected number
```

两条很可能被错误合并。

正确做法：

```text
A: 126334 -> PRODUCT_REFERENCE
B: 126334 -> ACCESSORY_COMPAT_REF
B entity_type -> ACCESSORY
```

因此直接 NON_MATCH / ABSTAIN。

这个例子说明：**编号角色分类比“字符串有没有抽出来”更重要。**

---

## 12. 增量漂移：这是 MoRER 最值得直接借鉴的部分

假设某周奢当家改了标题模板，从：

```text
ROLEX 126334 日志型 蓝盘
```

变成：

```text
表带/盒证/配件适配 126334 / 126300 / 126333 ...
```

reference extractor 仍然能抽出数字，但：

```text
multi_reference_rate ↑
accessory_compatible_ref_rate ↑
reference_conflict_rate ↑
catalog hit pattern 改变
```

Task Profile 与历史 cluster 的 PSI/KS 会明显变化。

建议处理：

```text
new profile
 -> not similar to trusted cluster
 -> drift_flag = true
 -> quarantine policy
 -> 所有新记录 MATCH 权限降级为 ABSTAIN
 -> 抽 20~50 个 hard cases 人工标注
 -> 更新 extractor/role classifier
 -> profile 稳定后恢复自动放行
```

这比“发现线上误合并以后再回滚”安全很多。

---

## 13. 黄金标签怎么花最值

Spec 允许几百对人工标注。不要随机抽。

建议 300~500 条先按风险分配：

```text
40% hard negative
    - 同系列不同 reference
    - 只差 1 个字符/数字
    - 旧款/新款 reference 很接近

20% identifier-role hard cases
    - 平台 SKU
    - 商家货号
    - 序列号
    - 配件适配型号

20% extraction ambiguity
    - 标题多个型号
    - reference 藏在描述
    - OCR 多候选

20% positive formatting variants
    - 空格/连字符/大小写
    - 品牌别名
    - 同 reference 不同来源写法
```

并且按：

```text
brand × source × channel × task cluster
```

做分层，避免所有黄金标签都来自 Rolex 标题字段完整的简单样本。

Active Learning 也必须改成 **false-positive oriented**：优先标“系统最想自动 MATCH、但证据存在轻微冲突”的候选，而不是只追求整体 uncertainty。

---

## 14. 数据表建议

### 14.1 `product_record`

```sql
record_id              bigint primary key
source                  text
source_product_id       text
raw_title               text
raw_brand               text
canonical_brand_id      bigint
entity_type             text
parser_version          text
created_at              timestamptz
updated_at              timestamptz
```

### 14.2 `reference_observation`

```sql
observation_id          bigint primary key
record_id               bigint
raw_value               text
canonical_value         text
role                    text
channel                 text
extractor_version       text
confidence              double precision
is_format_valid         boolean
catalog_hit             boolean
is_conflicting          boolean
```

### 14.3 `reference_entity`

```sql
entity_id               bigint primary key
canonical_brand_id      bigint
canonical_reference     text
unique(canonical_brand_id, canonical_reference)
```

### 14.4 `record_entity_link`

```sql
record_id               bigint
entity_id               bigint
state                   text -- MATCH / ABSTAIN
reason_code             text
policy_bundle_id        bigint
evidence_snapshot       jsonb
created_at              timestamptz
```

### 14.5 `task_profile`

```sql
profile_id              bigint primary key
source_a                text
source_b                text
brand_family            text
field_profile           text
parser_version          text
time_window             text
feature_schema_version  text
metrics                  jsonb
cluster_id              bigint
drift_state             text
```

### 14.6 `policy_bundle`

```sql
policy_bundle_id        bigint primary key
cluster_id              bigint
brand_ruleset_version   text
extractor_version       text
role_model_version      text
validator_version       text
gate_version            text
status                  text
```

---

## 15. 可直接落地的服务划分

第一版不需要重微服务，可以做 4 个逻辑模块：

```text
1. ingest-worker
2. reference-service
3. task-profile-worker
4. review/admin
```

### ingest-worker

负责：

```text
三源数据读入
raw snapshot
字段统一
图片/附件地址持久化
```

### reference-service

负责：

```text
brand normalize
reference candidate extraction
identifier role classification
brand-specific canonicalization
catalog validation
precision gate
reference index lookup
```

### task-profile-worker

按小时/天/周聚合：

```text
TaskProfile features
KS / PSI
build task graph
Leiden clustering
policy routing
new drift alerts
```

### review/admin

展示：

```text
ABSTAIN cases
conflicting reference evidence
new unseen reference pattern
cluster drift
parser/model version
人工标签
```

---

## 16. 技术栈建议

### MVP

```text
Python 3.12
FastAPI
PostgreSQL
Polars
scipy
scikit-learn
igraph + leidenalg
Redis（可选）
S3/OSS（原始快照/图片/OCR 结果）
```

### 数据量继续增长后

10M 量级 PostgreSQL 仍可以承担 canonical reference 索引，但 profile/event analytics 可单独放：

```text
ClickHouse
```

如果增量抓取频率高，再加：

```text
Kafka / Redpanda
```

不建议一开始就引入向量数据库。当前主键就是 reference；embedding 只服务于 ABSTAIN 辅助分析，不是主索引。

---

## 17. 推荐的代码模块

```text
src/
  ingest/
    adapters/
      leixiaoan.py
      xcar_watch.py
      shedangjia.py

  reference/
    brand_normalizer.py
    candidate_extractor.py
    role_classifier.py
    canonicalizer.py
    catalog.py
    decision_gate.py

  profile/
    feature_builder.py
    distribution.py
    graph.py
    clustering.py
    router.py
    drift.py

  models/
    registry.py
    validator.py

  review/
    sampler.py
    service.py

  jobs/
    backfill.py
    incremental.py
    rebuild_profiles.py
```

MoRER 官方代码可以映射参考：

```text
statistical_tests.py -> profile/distribution.py
graph_clustering.py  -> profile/clustering.py
model_selection.py   -> profile/router.py
model_generation.py  -> models/registry.py
main.py              -> jobs/rebuild_profiles.py
```

但实现应重新写，不要把官方脚本直接生产化。

---

## 18. 决策代码骨架

```python
from enum import Enum

class Decision(str, Enum):
    MATCH = "MATCH"
    NON_MATCH = "NON_MATCH"
    ABSTAIN = "ABSTAIN"


def decide(a, b, policy):
    # 1. brand 必须可信
    if not a.brand_verified or not b.brand_verified:
        return Decision.ABSTAIN, "BRAND_UNVERIFIED"

    if a.brand_id != b.brand_id:
        return Decision.NON_MATCH, "BRAND_CONFLICT"

    # 2. 只接受 PRODUCT_REFERENCE
    ra = a.unique_product_reference()
    rb = b.unique_product_reference()

    if ra is None or rb is None:
        return Decision.ABSTAIN, "REFERENCE_MISSING_OR_AMBIGUOUS"

    # 3. 任何冲突 reference 直接拒绝自动匹配
    if a.has_reference_conflict or b.has_reference_conflict:
        return Decision.ABSTAIN, "REFERENCE_CONFLICT"

    # 4. 品牌格式校验
    if not ra.is_format_valid or not rb.is_format_valid:
        return Decision.ABSTAIN, "INVALID_REFERENCE_PATTERN"

    # 5. task profile 漂移时禁止自动放行
    if policy.quarantined:
        return Decision.ABSTAIN, "PROFILE_DRIFT"

    # 6. reference 是最终业务定义
    if ra.canonical_value != rb.canonical_value:
        return Decision.NON_MATCH, "REFERENCE_DIFFERENT"

    # 7. 高风险渠道要求额外证据
    if policy.require_two_channel_confirmation:
        if not (a.has_independent_confirmation(ra) and
                b.has_independent_confirmation(rb)):
            return Decision.ABSTAIN, "INSUFFICIENT_EVIDENCE"

    return Decision.MATCH, "EXACT_VERIFIED_REFERENCE"
```

这段逻辑体现一个关键边界：

> **模型不会被调用来决定“两个不同 reference 是否其实一样”。**

---

## 19. Task Profile / MoRER 路由代码骨架

```python
@dataclass
class TaskProfile:
    task_id: str
    features: dict[str, np.ndarray]
    parser_version: str
    policy_bundle_id: str | None = None


def profile_similarity(a: TaskProfile, b: TaskProfile) -> float:
    scores = []
    weights = []

    for name in RISK_FEATURES:
        x = a.features[name]
        y = b.features[name]

        psi = calc_psi(x, y)
        ks = ks_2samp(x, y).statistic

        sim = 1.0 - combine_distance(psi, ks)
        scores.append(sim)
        weights.append(RISK_WEIGHTS[name])

    return np.average(scores, weights=weights)


def rebuild_policy_clusters(profiles):
    g = ig.Graph()
    # add profile nodes
    # only add high-similarity edges
    # run leiden
    # bind / reuse policy bundle
    # new isolated profile => quarantine + review
```

与论文不同，建议 graph 只连接：

```text
同品牌族 / 相近品牌族
相同或相关来源对
近期窗口
```

不要对几千个 profile 做无条件全连接两两比较。随着 profile 增长，可以只取 kNN 或候选 blocking，把 task-level 图也控制住。

---

## 20. 历史 100 万–1000 万数据的处理顺序

建议顺序：

```text
Phase 1: brand normalize
Phase 2: explicit ref field ingest
Phase 3: deterministic title extractor
Phase 4: brand-specific canonicalization + catalog validation
Phase 5: strict verified key group
Phase 6: multi-ref / conflict / missing -> ABSTAIN
Phase 7: OCR only for ABSTAIN high-value records
Phase 8: optional model-assisted extraction
```

不要一开始把所有图片都跑重型视觉模型。因为大量记录可能已经有 reference 字段或标题可抽，图片只应该补 long tail。

---

## 21. 对图片的建议

Spec 明确有图片，但在 same-reference 定义下，图片最适合：

```text
OCR 表盘/表背/保卡 reference
识别 accessory vs watch
冲突 veto
人工审核辅助
```

不建议：

```text
CLIP similarity > threshold -> 自动 MATCH
```

理由：同系列不同 reference 的腕表视觉差异可能极小，视觉模型最容易把“非常像”误解释成“同 reference”。

图片只能提高 reference evidence 的覆盖率，不能改变最终 identity key。

---

## 22. 线上监控指标

不要只看 F1。生产指标建议：

### 最优先

```text
false_auto_merge_count
verified_auto_match_precision
match_with_conflicting_reference_count
```

### 覆盖

```text
auto_match_rate
abstain_rate
reference_extracted_rate
catalog_hit_rate
```

### 漂移

```text
reference_conflict_rate
multi_reference_rate
unknown_identifier_rate
PSI by TaskProfile
new_pattern_rate
cluster_switch_rate
```

### 人审

```text
review_confirm_match_rate
review_false_positive_rate
hard_negative_hit_rate
```

需要特别注意：几百个标签不能证明“零 FP”，所以 acceptance 应同时包含硬规则审计和专门的 red-team hard-negative suite。

---

## 23. 测试集应该故意制造“很像但绝不能合并”的样本

必须覆盖：

```text
同品牌同系列不同 reference
同 reference 不同分隔符
reference 只差一位
reference 前后缀不同
多个 reference 同时出现在标题
配件适配 reference
盒证/表带/零件 listing
平台 SKU 恰好长得像 reference
serial number 像 reference
explicit field 与 title 冲突
OCR 与 explicit field 冲突
同 reference 但品牌错误
brand alias 正确/错误映射
```

验收不要问“F1 到多少”，先问：

> **这些 hard negative 有没有任何一条被自动 MATCH？**

只要有，当前 gate 就不能上线。

---

## 24. 分阶段落地方案

## Phase 0：数据审计（1~2 天）

输出：

```text
三源 schema map
reference field coverage
brand coverage
标题中 reference 命中率
multi-reference 比例
SKU/serial/accessory 混淆样本
Top 20 品牌 reference 格式
```

目标是先知道 80% 数据能否靠确定性规则处理。

## Phase 1：Reference-first MVP（3~5 天）

完成：

```text
brand normalize
Top 品牌 reference parser
identifier role
canonical reference key
strict decision gate
Postgres inverted index
MATCH/NON_MATCH/ABSTAIN
```

这一阶段就可以安全处理大量显式 reference 数据。

## Phase 2：MoRER-style Task Profile（2~4 天）

完成：

```text
profile feature aggregation
PSI / KS
profile similarity graph
Leiden clustering
cluster -> policy bundle
profile drift quarantine
```

此时 MoRER 的核心价值开始体现。

## Phase 3：少量黄金标签 + Active Learning（2~4 天）

标 300~500 条 hard cases，训练：

```text
identifier role classifier
accessory detector
reference span validator
conflict detector
```

这些模型仍然只有“辅助/否决”权限。

## Phase 4：OCR / 图片补覆盖

只处理：

```text
高价值 ABSTAIN
reference missing
标题不可解析
有保卡/表背图
```

避免图片模型成为全量成本中心。

---

## 25. 与 MoRER 原论文的对应关系

| MoRER | 当前方案 |
|---|---|
| ER problem | source pair × brand × field profile × parser version × time window |
| similarity feature vector | reference/evidence/candidate risk features |
| distribution test | PSI / KS / 可选 C2ST |
| ER problem graph | Task Profile similarity graph |
| Leiden cluster | 相似数据分布的 policy family |
| cluster classifier | extractor/role model/validator + gate policy bundle |
| `sel_base` | 复用最近 trusted policy bundle |
| `sel_cov` | 新 profile 入图，漂移则 quarantine / 重聚类 |
| active learning budget | hard-negative 人工标注预算 |
| retraining | 更新 extractor/role/validator，不改变 reference identity 定义 |

这基本保留了 MoRER 的系统设计，但把“模型做 identity 判定”改成“模型服务于 reference 证据生产与风险控制”。

---

## 26. 最终建议

如果只从 MoRER 里选一件事直接落地，我会选：

> **建立 Task Profile Repository，把来源/品牌/字段覆盖/解析器版本/时间窗口的运行分布做成可聚类、可路由、可漂移检测的对象。**

如果只从当前 Spec 里选一条最高优先级原则，我会选：

> **最终自动合并只能由“可信 canonical reference 严格相等”触发，所有模糊相似度都只能补证据或否决。**

最终推荐架构不是“MoRER matcher”，而是：

```text
Reference Extraction
    + Identifier Role Classification
    + Brand-specific Canonicalization
    + Exact Reference Index
    + Precision Gate / Abstain
    + MoRER-style Task Profile Routing
    + Drift Quarantine
    + Hard-negative Active Learning
```

这套方案能同时满足：

- 三源异构字段；
- 100 万–1000 万历史数据；
- 持续增量；
- 少量人工标签；
- 图片可辅助；
- precision-first；
- 可解释、可审计；
- 新品牌/新模板分布变化时不把风险静默带入线上。

它比“直接训练一个商品匹配模型”更贴合当前业务定义，也更接近可以真正上线的系统。