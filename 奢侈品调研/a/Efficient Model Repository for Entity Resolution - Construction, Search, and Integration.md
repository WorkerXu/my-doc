# MoRER：把“模型复用 + 分布漂移”改造成 Reference-first 二奢匹配的控制平面

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取论文 **Efficient Model Repository for Entity Resolution: Construction, Search, and Integration**（MoRER）做深入分析。

- 论文介绍页：<https://dbs.uni-leipzig.de/research/publications/efficient-model-repository-for-entity-resolution-construction-search-and>
- 论文 PDF：<https://openproceedings.org/2026/conf/edbt/paper-245.pdf>
- EDBT 2026，Victor Christen / Peter Christen
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

分析前已检查 `奢侈品调研/a` 当前已有结果，目录中已经包含 TransClean、GraLMatch、DeepBlocker、AnyMatch、ALMSER-GB、Conformal Selective Prediction、Confidence Classifiers、各种多模态属性抽取/商品匹配文章等；**本论文尚未出现在 `a` 目录**，因此本次继续执行。

当前 Spec 的核心约束是：

1. 雷小安、腕表之家、奢当家三个来源；
2. 数据规模 100 万～1000 万，并持续增量；
3. “同一个商品”业务定义为 **同一 reference number / 型号**，不是泛语义意义上的“看起来是同款”；
4. reference 字段高度稀疏，有时结构化、有时埋在标题/描述里，图片也可用；
5. **precision 极端优先，绝对不能误匹配，允许漏匹配**；
6. 可以人工标几百对黄金标签。

MoRER 的论文主张是：

> 多源 Entity Resolution 不应该每新增一个来源组合就从头训练一个 matcher；可以先比较不同 ER 任务的“相似度特征分布”，把相近的 ER 任务聚成簇，每簇维护一个分类模型。新任务来了以后先判断它属于哪个任务簇，再复用对应模型；如果分布漂移，再重聚类和补标训练数据。

这个思路对当前需求非常有价值，但**不能原样拿来作为最终匹配器**。

原因很简单：论文最终仍然在做 `pair -> Match / NonMatch` 分类，而当前业务已经明确给出了一个更强的身份不变量：

```text
同一商品 <=> 同一 canonical reference
```

因此当前项目最安全的主链路应该是：

```text
商品记录
  -> reference 候选抽取
  -> 品牌内保守规范化
  -> reference 证据聚合 / 冲突检测
  -> 唯一 reference_entity_id
  -> exact join
```

MoRER 更适合被改造成**控制平面（Control Plane）**，负责：

```text
任务分布画像
-> 任务聚类
-> 模型/规则路由
-> 漂移检测
-> 标注预算分配
-> 新任务触发重训/人工复核
```

而真正产生“同一商品”关系的**数据平面（Data Plane）**仍由：

```text
brand_id + canonical_reference + 安全证据
```

做硬收口。

我的直接落地建议可以概括成一句话：

> **用 MoRER 管“该用哪个 reference 抽取/验证模型、何时认为数据分布变了”，不要让 MoRER 决定“两个腕表是否同款”；最终自动合并只由已经安全解析到同一个 Reference Entity 的记录产生。**

---

# 1. 论文真正解决的问题

论文考虑的是典型 Multi-Source Entity Resolution（MS-ER）。

假设有数据源：

```text
D1, D2, D3, ... Dz
```

每两个来源之间都可能构成一个 ER problem：

```text
p1,2
p1,3
p2,3
...
```

每个候选 record pair 被转换成 similarity feature vector：

```text
w = [
  sim(title),
  sim(brand),
  sim(model_number),
  sim(price),
  ...
]
```

传统做法是：

```text
p1,2 -> 训练 M1,2
p1,3 -> 再训练 M1,3
p2,3 -> 再训练 M2,3
```

来源一多，ER 任务数近似按来源对数增长，标注和训练成本快速上升。

MoRER 的核心观察是：

> 不同来源对虽然是不同 ER 任务，但其 similarity feature 的统计分布可能非常相似；相似任务不需要独立模型，可以复用一个模型。

例如：

```text
D1 vs D2：
  title_sim 分布、brand_sim 分布、model_sim 分布

D1 vs D3：
  如果这些分布和 D1 vs D2 高度相似
  => 可能直接复用 M1,2
```

这本质上不是“比较两条商品”，而是：

```text
比较两个 Matching Task 本身是否相似
```

这是论文最值得当前项目借鉴的地方。

---

# 2. MoRER 的完整技术架构

论文 Figure 3 给出了 5 个阶段：

```text
1. Similarity Distribution Analysis
2. ER Problem Clustering
3. Model Generation
4. Process New ER Problems
5. Classification with appropriate model
```

可以拆成下面的系统。

```text
                 ┌────────────────────┐
                 │ Existing ER Tasks  │
                 │ p1,2 p1,3 ... pm,n │
                 └─────────┬──────────┘
                           │
                           v
              ┌──────────────────────────┐
              │ Distribution Profiler    │
              │ KS / WD / PSI / C2ST     │
              └───────────┬──────────────┘
                          │ task similarity
                          v
              ┌──────────────────────────┐
              │ ER Task Similarity Graph │
              │ node = ER task           │
              │ edge = distribution sim  │
              └───────────┬──────────────┘
                          │ Leiden
                          v
              ┌──────────────────────────┐
              │ Task Clusters C1...Ck    │
              └───────────┬──────────────┘
                          │ label budget
                          v
              ┌──────────────────────────┐
              │ Model per Cluster        │
              │ MC1 ... MCk              │
              └───────────┬──────────────┘
                          │
          new task -------┘
               │
               v
    distribution similarity / reclustering
               │
               v
        select/reuse/update model
```

## 2.1 输入不是原始字段，而是统一 similarity feature vector

论文并不强制某一种 Blocking 或编码器。它假设上游已经把候选对转换成统一 feature vector。

可以是：

```text
Jaccard(title)
Jaro-Winkler(name)
normalized numeric difference(price)
embedding cosine similarity
...
```

论文也明确提到：

- 可以先 Blocking 降低候选空间；
- 文本可用预训练语言模型生成 embedding；
- 如果不同来源属性不完全一致，可以把多个字段编码成统一 record embedding，再计算 embedding similarity。

这意味着 MoRER 本身更像：

```text
ER 模型生命周期 / 任务路由框架
```

而不是具体的 feature extractor。

---

# 3. 任务分布怎么比较：KS / Wasserstein / PSI / C2ST

MoRER 的关键不是逐 pair 比较，而是对每个 ER task 的 feature 分布做统计画像。

对 feature `f`：

```text
p1,2 -> d^f_1,2
p3,4 -> d^f_3,4
```

然后判断两个分布是否相近。

论文评估了四种方式。

## 3.1 Kolmogorov-Smirnov（KS）

比较两组样本累计分布函数 CDF 的最大差异。

直观上：

```text
KS 越小
=> 两个任务在该 feature 上越像
```

优点：

- 计算便宜；
- 不需要训练额外模型；
- 对单变量漂移比较直接。

对当前项目特别适合监控：

```text
reference_extractor_confidence
normalization_ops_count
ocr_confidence
ref_candidate_count
raw_ref_length
```

这些值的月度/来源/品牌分布有没有突然变化。

## 3.2 Wasserstein Distance

衡量把一个分布“搬运”成另一个分布需要多少成本。

它能捕获整体形状差异，但论文实验里也指出：不同数据集上效果并不稳定，有些情况下会把本应不同的任务判得过于相似。

因此当前项目不应只依赖 WD 做模型路由。

## 3.3 Population Stability Index（PSI）

把 feature 分桶，比较新旧任务各区间样本比例的变化。

论文实验中 PSI 在异构/噪声场景下表现相对稳定。

这个指标非常适合生产监控，因为可解释性强。

例如：

```text
腕表之家 / Rolex / title-only
过去 30 天：
  90% 商品只抽到 1 个 ref candidate

本周：
  35% 商品抽到 >= 3 个 ref candidate
```

即使最终模型精度暂时没明显下降，这已经说明：

```text
标题格式 / 抓取模板 / parser 输入分布发生变化
```

应该进入 drift 告警。

## 3.4 Classifier Two-Sample Test（C2ST）

把两个任务的数据合起来，训练一个分类器判断样本来自 Task A 还是 Task B。

如果分类器很容易区分：

```text
Task A 与 Task B 分布差很多
```

如果很难区分：

```text
两个任务可能属于同一分布
```

论文把 inverse F1 作为任务相似度。

C2ST 对高维 feature vector 更有价值，但计算比 KS/PSI 贵。

对本项目建议：

```text
在线监控：PSI + 少量 KS
离线重聚类：PSI / KS + C2ST
```

不需要每批数据都跑 C2ST。

---

# 4. 从任务相似度到“模型仓库”：ER Problem Graph + Leiden

论文把每个 ER task 当成一个图节点：

```text
G_P = (P, E)
```

节点：

```text
p1,2
p1,3
p2,3
...
```

边权：

```text
sim_p(task_i, task_j)
```

然后使用 **Leiden community detection** 对 ER task graph 聚类。

论文也尝试过 Girvan-Newman、Label Propagation，最终使用 Leiden，原因之一是它对大图的可扩展性和社区质量较好。

最终得到：

```text
C1 = {p1,2, p1,3, p4,7}
C2 = {p2,5, p3,5}
C3 = {...}
```

然后：

```text
一个 Cluster -> 一个 Matcher Model
```

也就是说：

```text
M_C1
M_C2
M_C3
```

而不是：

```text
每个 source pair -> 一个独立 model
```

这就是“Model Repository”的核心。

---

# 5. 标注预算怎么分：不是平均分，而是按 Cluster 分配

MoRER 很有价值的第二个思想，是**把有限人工标注预算分配给任务簇，而不是每个来源都平均标**。

论文设：

```text
b_tot = 总标签预算
b_min = 每个 cluster 最低预算
```

先保证每个 cluster 至少有 `b_min`，剩余预算再根据 cluster 中 feature vector 数量等信息按比例分配。

对当前需求可直接迁移。

用户只愿意标几百对，那么最浪费的做法是：

```text
雷小安 vs 腕表之家 100 对
雷小安 vs 奢当家   100 对
腕表之家 vs 奢当家 100 对
```

因为真正难度可能完全不平均。

更合理的是先按“风险任务”分布：

```text
Rolex / 标题 reference 清晰 / 结构化字段稳定
=> 少标

Omega / 点号 reference / 标题格式变化大
=> 多标

某来源图片 OCR / 多候选 reference / 配件污染严重
=> 多标

新品牌 / 新模板 / 新抓取字段
=> 多标
```

这和 MoRER 的 cluster-level budget allocation 完全一致。

---

# 6. 新任务来了以后怎么处理：sel_base 与 sel_cov

论文有两类策略。

## 6.1 sel_base：直接找最近 Cluster 复用模型

新 ER problem：

```text
p_new
```

计算它和现有 cluster 训练数据分布的相似度：

```text
sim(p_new, C1)
sim(p_new, C2)
...
```

选最相似 cluster：

```text
argmax sim
```

直接调用该 cluster 的模型。

这个路径适合：

```text
数据格式比较稳定
新批次和旧批次差异小
```

## 6.2 sel_cov：新任务进入图，重新聚类，并检查 Coverage

如果新任务分布可能产生 domain shift，MoRER 会：

```text
1. 把新 task 放入 ER problem graph
2. 和历史 task 建边
3. 重新 Leiden 聚类
4. 判断新 cluster 中有多少 feature vector 来自尚未训练过的新任务
```

论文定义 coverage ratio，大意是：

```text
新任务数据在 cluster 中占比越来越高
=> 旧模型开始“代表性不足”
=> 需要补标并更新模型
```

这是当前项目非常值得直接落地的部分。

因为二奢抓取一定会出现：

```text
网站改版
字段重命名
标题模板变化
新品牌上线
图片比例变化
OCR 质量变化
商家把 reference 改到描述或图片里
```

这些问题往往不是“模型突然坏了”，而是**输入任务分布变了**。

把它显式建模成 task drift，比只看线上 precision/recall 更早发现风险。

---

# 7. 为什么 MoRER 不能直接作为当前 Spec 的最终 Matcher

这一点必须明确。

论文在实验里优化/报告的是：

```text
Precision
Recall
F1
```

但当前业务不是普通 F1 问题，而是：

```text
False Positive 成本远高于 False Negative
```

论文 Table 4 中，在不同数据集/预算下，MoRER 组合方案的 precision 有大量 0.84、0.88、0.94、0.96 等结果。

这在普通 ER benchmark 中可以是不错的结果，但在当前业务定义下完全不能直接自动上线。

如果 precision = 0.96：

```text
每 100 个系统自动声明“同一商品”的结果里
理论上平均可能有 4 个错误
```

对于“绝对不能误匹配”的要求，这个风险量级不可接受。

更重要的是，当前业务有一个比统计分类更强的逻辑定义：

```text
同一商品 = 同一 reference
```

既然业务已经给了稳定 ID，系统就不应该让 title/image semantic similarity 去“投票覆盖”这个 ID。

因此我不建议：

```text
pair features
-> MoRER matcher
-> score > 0.9
-> Match
```

我建议：

```text
MoRER / ML / LLM
只负责：
  - reference 抽取辅助
  - reference 是否可信的风险估计
  - 数据分布路由
  - 冲突否决
  - 人工样本选择

最终 Match：
  accepted reference_entity_id 完全相同
```

---

# 8. 对当前需求的改造：Reference-first + MoRER Control Plane

完整架构建议分成数据平面和控制平面。

## 8.1 数据平面：唯一负责产生 Match

```text
                 ┌─────────────────────────┐
                 │ 雷小安 / 腕表之家 / 奢当家 │
                 └────────────┬────────────┘
                              v
                   ┌─────────────────────┐
                   │ Raw Product Storage │
                   └─────────┬───────────┘
                             v
                ┌──────────────────────────┐
                │ Reference Evidence Extractor│
                │ structured/title/desc/OCR│
                └───────────┬──────────────┘
                            v
                ┌──────────────────────────┐
                │ Brand-aware Normalizer   │
                │ conservative canonicalize│
                └───────────┬──────────────┘
                            v
                ┌──────────────────────────┐
                │ Evidence Aggregator      │
                │ agreement/conflict/risk  │
                └───────────┬──────────────┘
                            v
                ┌──────────────────────────┐
                │ Product -> Reference     │
                │ accepted/review/rejected │
                └───────────┬──────────────┘
                            v
                ┌──────────────────────────┐
                │ Reference Entity Table   │
                │ UNIQUE(brand, canonical) │
                └───────────┬──────────────┘
                            v
                     EXACT JOIN ONLY
```

核心 invariant：

```text
只有 product_reference_assignment.status = accepted
才允许进入自动匹配。
```

然后：

```sql
SELECT ...
FROM product_reference_assignment a
JOIN product_reference_assignment b
  ON a.reference_entity_id = b.reference_entity_id
WHERE a.status = 'accepted'
  AND b.status = 'accepted'
  AND a.source <> b.source;
```

这比千万级 record pair 全量比较更安全，也更便宜。

## 8.2 控制平面：MoRER 风格任务仓库

```text
                       ┌───────────────────┐
                       │ Task Profiler     │
                       │ PSI/KS/C2ST       │
                       └─────────┬─────────┘
                                 v
                       ┌───────────────────┐
                       │ Task Similarity   │
                       │ Graph + Leiden    │
                       └─────────┬─────────┘
                                 v
                    ┌────────────────────────┐
                    │ Task Cluster Registry  │
                    └──────────┬─────────────┘
                               v
                    ┌────────────────────────┐
                    │ Rule / Model Registry  │
                    │ cluster -> versions    │
                    └──────────┬─────────────┘
                               v
                   extraction / validation route
                               │
                               v
                    ┌────────────────────────┐
                    │ Drift / Coverage Check │
                    └──────────┬─────────────┘
                               │
                       label / retrain / review
```

控制平面永远不直接写：

```text
product A == product B
```

它只写：

```text
这个任务应该使用哪套 extractor/normalizer/validator
这批数据是否已经漂移
是否必须降低自动化等级
是否需要补标
```

---

# 9. 当前项目应该如何定义“Task”

如果完全按论文定义：

```text
Task = source_pair
```

当前只有三个来源，只有：

```text
雷小安 × 腕表之家
雷小安 × 奢当家
腕表之家 × 奢当家
```

任务太少，MoRER 的 task clustering 价值有限。

所以需要把 task 颗粒度细化。

我建议：

```text
Task = source_pair
     × brand_bucket
     × evidence_mode
     × time_window
```

例如：

```text
腕表之家 × 奢当家
Rolex
structured-title
2026-08
```

或者：

```text
雷小安 × 奢当家
Omega
ocr-title
2026-08
```

`evidence_mode` 可以定义为：

```text
S-S = 两边都有结构化 reference
S-T = 一边结构化，一边 title 抽取
T-T = 两边 title 抽取
T-O = title + image OCR
O-O = 两边都主要来自 OCR
MIX = 多通道冲突或组合
```

这样三个来源也可以自然形成几十到数百个 ER tasks。

最重要的是：

```text
任务差异真正来自“品牌 reference 规则 + 来源格式 + 证据通道 + 时间漂移”
```

而不只是来源名字。

---

# 10. 推荐的 Reference Evidence 数据结构

不要直接在 product 表上写一个 `reference` 字符串然后覆盖。

应该保存**证据级 lineage**。

```sql
CREATE TABLE reference_evidence (
    id                  BIGSERIAL PRIMARY KEY,
    product_id          BIGINT NOT NULL,
    source              TEXT NOT NULL,
    brand_id            BIGINT,

    evidence_channel    TEXT NOT NULL,
    -- structured_field / title_regex / description / ocr / llm_span

    raw_value           TEXT NOT NULL,
    normalized_candidate TEXT,

    extractor_name      TEXT NOT NULL,
    extractor_version   TEXT NOT NULL,
    normalizer_version  TEXT,

    extraction_confidence DOUBLE PRECISION,
    ocr_confidence      DOUBLE PRECISION,
    pattern_valid       BOOLEAN,
    dictionary_hit      BOOLEAN,
    role_is_reference   BOOLEAN,

    source_locator      JSONB,
    -- 例如字段名、标题 span、图片 id + bbox

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

好处：

1. 任何自动 Match 都可以回溯到原始证据；
2. parser/normalizer 升级后可以重新算，不覆盖历史；
3. 如果标题和图片冲突，可以显式看到；
4. 人工复核界面可以直接展示证据位置。

---

# 11. Reference Entity 不要用字符串直接 Join

建议有独立 reference dictionary / entity table。

```sql
CREATE TABLE reference_entity (
    id                  BIGSERIAL PRIMARY KEY,
    brand_id            BIGINT NOT NULL,
    canonical_reference TEXT NOT NULL,
    canonical_display   TEXT,
    grammar_version     TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'active',
    metadata            JSONB,

    UNIQUE (brand_id, canonical_reference)
);
```

为什么必须 `brand_id + canonical_reference`？

因为不同品牌可能存在形似甚至完全相同的型号串。

最终身份应该是：

```text
reference_entity_id
```

而不是裸字符串：

```text
"126610LN"
```

---

# 12. Product -> Reference Assignment 必须是一个状态机

```sql
CREATE TABLE product_reference_assignment (
    product_id           BIGINT PRIMARY KEY,
    reference_entity_id  BIGINT,

    status               TEXT NOT NULL,
    -- accepted / review / rejected / unresolved

    evidence_tier        TEXT,
    -- A / B / C / D

    risk_score           DOUBLE PRECISION,
    conflict_flags       JSONB,
    evidence_ids         BIGINT[],

    router_task_id       BIGINT,
    model_version        TEXT,
    normalizer_version   TEXT,

    decided_at           TIMESTAMPTZ,
    decided_by           TEXT
);
```

建议证据等级：

## Tier A：可直接自动接受

例如：

```text
结构化 reference 字段
+ 品牌已确认
+ brand-specific grammar 通过
+ reference dictionary 命中
+ 无任何冲突候选
```

## Tier B：双通道独立一致

例如：

```text
title 抽取 = 126610LN
图片 OCR   = 126610LN
brand      = Rolex
无第二个冲突 reference
```

可以自动接受，但仍要记录证据来源。

## Tier C：只有单一噪声通道

例如：

```text
只有标题 LLM/regex 抽到一个 ref
```

即使 validator 给出很高概率，也建议进入：

```text
review / abstain
```

而不是自动 Match。

## Tier D：只有模糊相似或视觉相似

例如：

```text
CLIP 看起来像某型号
标题只写“黑水鬼”
```

永远不能自动创建 Reference Assignment。

---

# 13. Reference 规范化必须“品牌内保守”，不能全局去字符

错误的 normalizer 往往比 matcher 更危险。

危险做法：

```python
canonical = re.sub(r'[^0-9A-Z]', '', raw.upper())
```

然后无条件认为：

```text
ABC-12-34
ABC1234
ABC 12.34
```

完全等价。

有些品牌 separator 可以安全忽略，有些 separator/后缀却是型号语义的一部分。

所以建议：

```text
Global normalization
  只做 Unicode NFKC、大小写、全半角等无歧义处理

Brand-specific normalization
  根据品牌 grammar/dictionary 做有限 alias mapping
```

例如：

```text
raw_value
 -> unicode_normalized
 -> whitespace_normalized
 -> brand parser
 -> dictionary alias resolution
 -> canonical_reference
```

必须保留：

```text
raw_value
canonical_reference
normalization_operations
normalizer_version
```

如果一次规范化做了太多 destructive operations：

```text
normalization_lossiness = high
```

应该主动降低自动接受等级。

---

# 14. “看起来像 reference 的字符串”还要先判断它是什么编号

二奢数据里很容易出现：

```text
平台商品 ID
店铺 SKU
库存号
订单号
证书号
机芯号
序列号
配件适配型号
真正品牌 reference
```

这些字符串经常长得都像字母数字编码。

因此抽取不应该是：

```text
找到疑似型号串 -> reference
```

而应该是：

```text
candidate span
 -> number-role classifier / rules
 -> brand reference?
 -> serial?
 -> store SKU?
 -> compatible-with reference?
```

尤其配件标题：

```text
“适配 Rolex 126610LN 表带”
```

这里出现 `126610LN` 并不意味着当前商品本身就是 Rolex 126610LN 腕表。

因此至少需要 feature：

```text
product_type_is_watch
accessory_flag
reference_context_role
compatibility_keyword_flag
```

这类 feature 非常适合交给 MoRER 的 task/model router 去适配不同来源模板。

---

# 15. MoRER 在本项目里的 Feature Vector 应该长什么样

论文原始 feature 更偏：

```text
simTitle
simBrand
simModel
simPrice
```

当前项目必须改成 **reference-risk feature vector**。

建议至少包含：

```text
# identity hard features
brand_equal
canonical_ref_equal

# evidence strength
left_structured_ref
right_structured_ref
left_title_ref
right_title_ref
left_ocr_ref
right_ocr_ref

# validation
left_pattern_valid
right_pattern_valid
left_dictionary_hit
right_dictionary_hit

# ambiguity
left_ref_candidate_count
right_ref_candidate_count
left_conflicting_ref_count
right_conflicting_ref_count

# normalization risk
left_normalization_ops_count
right_normalization_ops_count
left_lossy_normalization
right_lossy_normalization
raw_ref_edit_distance

# role/context
left_reference_role_score
right_reference_role_score
left_accessory_flag
right_accessory_flag

# multimodal consistency
left_text_ocr_agree
right_text_ocr_agree
image_text_conflict

# source / pipeline metadata
extractor_confidence_min
ocr_confidence_min
parser_version_group
```

模型的职责不要定义成：

```text
P(pair_is_same_product)
```

而应定义成更安全的问题：

```text
P(reference_assignment_is_wrong)
```

即：

```text
risk / veto model
```

最终逻辑：

```text
硬规则判断“有资格自动接受”
模型只能否决，不能把不满足硬规则的 pair 提升为 Match
```

这个方向对 precision-first 系统非常重要。

---

# 16. 推荐的自动匹配硬规则

伪代码：

```python
def auto_match(left, right):
    # 1. 两边都必须已经完成安全 reference assignment
    if left.assignment_status != "accepted":
        return False
    if right.assignment_status != "accepted":
        return False

    # 2. 品牌必须相同
    if left.brand_id != right.brand_id:
        return False

    # 3. 最终 canonical reference entity 必须完全相同
    if left.reference_entity_id != right.reference_entity_id:
        return False

    # 4. 任意一边有 reference 冲突都禁止自动发布
    if left.has_reference_conflict or right.has_reference_conflict:
        return False

    # 5. 风险模型只有 veto 权
    if max(left.risk_score, right.risk_score) >= RISK_VETO_THRESHOLD:
        return False

    return True
```

需要强调：

```text
title similarity
image similarity
price similarity
series similarity
```

全部不能绕过第 3 条。

---

# 17. 图片应该怎么用：只做“reference 证据”和“冲突否决”

当前有图片，这是优势，但不能把视觉相似当成身份主键。

推荐图片用途按价值排序：

## 17.1 OCR Reference

优先识别：

```text
表背
保卡
吊牌
证书
包装标签
```

流程：

```text
image
-> text region detection
-> OCR
-> candidate reference spans
-> brand grammar
-> reference dictionary
-> evidence record
```

OCR 输出必须带：

```text
image_id
bbox
raw_text
ocr_confidence
extractor_version
```

## 17.2 Text / OCR Cross-check

如果：

```text
标题 reference = A
图片 OCR reference = B
A != B
```

直接进入：

```text
conflict
=> 禁止自动 Match
```

这比让视觉模型给一个“90% 相似”更符合 precision-first。

## 17.3 视觉 embedding

CLIP / ViT / 多模态 embedding 最适合：

```text
人工复核排序
候选召回
异常检查
```

不能单独提供正向自动 Match 权限。

---

# 18. MoRER Task Profiler 在这里应该统计什么

每个 Task 保存一份 distribution profile，而不是保存全部 feature vector 做两两比较。

例如：

```json
{
  "task": "腕表之家|奢当家|Rolex|T-O|2026-08",
  "count": 48210,
  "features": {
    "ref_candidate_count": {
      "hist": [0.01, 0.78, 0.17, 0.04]
    },
    "pattern_valid": {
      "true_rate": 0.963
    },
    "dictionary_hit": {
      "true_rate": 0.941
    },
    "text_ocr_agree": {
      "true_rate": 0.884
    },
    "normalization_ops_count": {
      "hist": [0.61, 0.27, 0.09, 0.03]
    }
  }
}
```

然后 task similarity 基于 profile 算，而不是把千万条样本互相比较。

这样成本会低很多。

---

# 19. Task Similarity Graph 的生产实现

建议表结构：

```sql
CREATE TABLE er_task (
    id              BIGSERIAL PRIMARY KEY,
    source_left     TEXT NOT NULL,
    source_right    TEXT NOT NULL,
    brand_bucket    TEXT NOT NULL,
    evidence_mode   TEXT NOT NULL,
    window_start    DATE NOT NULL,
    window_end      DATE NOT NULL,
    feature_schema_hash TEXT NOT NULL,
    sample_count    BIGINT NOT NULL,
    profile_json    JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
CREATE TABLE er_task_similarity (
    task_a          BIGINT NOT NULL,
    task_b          BIGINT NOT NULL,
    ks_score        DOUBLE PRECISION,
    psi_score       DOUBLE PRECISION,
    c2st_score      DOUBLE PRECISION,
    combined_score  DOUBLE PRECISION NOT NULL,
    profile_version TEXT NOT NULL,
    PRIMARY KEY(task_a, task_b)
);
```

```sql
CREATE TABLE er_task_cluster (
    task_id         BIGINT PRIMARY KEY,
    cluster_id      BIGINT NOT NULL,
    cluster_version TEXT NOT NULL,
    assigned_at     TIMESTAMPTZ NOT NULL
);
```

初版不必实时跑 Leiden。

可以每天/每周 batch：

```text
1. 更新最近窗口 task profile
2. 计算新 task 到 cluster representative 的距离
3. 如果都很远，再构图重聚类
4. 生成新的 cluster_version
```

这比每条商品在线算图更合理。

---

# 20. 为什么我不建议照搬论文“用 feature 标准差当权重”

MoRER 对不同 feature 的任务分布相似度做加权时，会考虑 feature 的标准差，认为变化较大的 feature 更可能有判别力。

对通用 ER 有道理，但当前系统是安全系统，权重不能只靠统计量决定。

例如：

```text
price similarity 的方差很高
```

不代表它应该在“是否同 reference”的任务路由里获得高权重。

相反：

```text
reference_conflict_rate
normalization_lossiness
accessory_rate
text_ocr_disagreement
```

即使方差没那么高，也可能是强风险信号。

因此建议：

```text
combined_weight = domain_safety_weight × statistical_weight
```

例如：

```text
reference_conflict_rate       3.0
normalization_lossiness       2.5
accessory_rate                2.0
text_ocr_disagreement         2.0
reference_candidate_count     1.5
price                         0.2
visual_embedding_similarity   0.2
```

让业务 identity invariant 优先于统计相关性。

---

# 21. Model Registry 需要保存什么

```sql
CREATE TABLE er_model_registry (
    model_id                    BIGSERIAL PRIMARY KEY,
    cluster_id                  BIGINT NOT NULL,
    cluster_version             TEXT NOT NULL,

    model_type                  TEXT NOT NULL,
    model_uri                   TEXT NOT NULL,
    feature_schema_hash         TEXT NOT NULL,

    training_task_ids           BIGINT[] NOT NULL,
    training_label_set_version  TEXT NOT NULL,
    normalizer_version          TEXT NOT NULL,
    extractor_version           TEXT NOT NULL,

    decision_threshold          DOUBLE PRECISION,
    veto_threshold              DOUBLE PRECISION,

    offline_precision           DOUBLE PRECISION,
    offline_recall              DOUBLE PRECISION,
    hard_negative_fp_rate       DOUBLE PRECISION,

    status                      TEXT NOT NULL,
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

所有线上 assignment 都必须记录：

```text
model_id
cluster_version
normalizer_version
extractor_version
```

否则出现错匹配以后无法追溯。

---

# 22. 模型选择：建议 LightGBM / Logistic 做风险模型，而不是大 LLM 直接判 pair

这里不需要复杂模型起步。

当前 feature 大量是：

```text
boolean
count
confidence
agreement
conflict
```

传统 tabular classifier 足够：

```text
Logistic Regression
LightGBM
XGBoost
Random Forest
```

优势：

- 快；
- 可解释；
- 容易做 threshold；
- 训练/部署成本低；
- 对几百标注比大模型 fine-tune 更现实。

LLM 更适合：

```text
从脏标题/描述里抽 candidate span
```

不适合直接承担：

```text
A 与 B 是不是同一商品
```

尤其在型号只差 1～2 个字符时，语言模型的语义相似偏好反而可能是风险。

论文也在 Dexter（大量 model number，轻微文本差异会造成 non-match）上观察到预训练语言模型类方法可能受这种细粒度差异影响，这和腕表 reference 场景高度一致。

---

# 23. 几百对黄金标签怎么花最值

“几百对”不足以统计证明一个高度自动化 matcher 接近零 false positive，因此人工标签必须用在最危险的边界上，而不是随机采样。

建议构造四类集合。

## 23.1 Positive Gold

明确同一：

```text
同品牌
同 reference
不同来源
不同标题写法
不同图片
```

## 23.2 Hard Negative

最重要。

必须大量包含：

```text
同品牌
同系列
reference 只差 1 个字符
reference 前后代
相同俗称
外观高度相似
```

例如系统最怕的不是：

```text
Rolex vs Cartier
```

而是：

```text
同一系列相邻 reference
```

## 23.3 Extraction Negative

标题里出现某 reference，但商品并不是该 reference：

```text
表带
配件
盒证
“适配 xxx”
对比文案
多个 reference 共现
```

## 23.4 Drift / New Task Gold

只标新 cluster / 高 PSI / 高 conflict 的数据。

MoRER 的预算思想可以直接用于这里。

例如总共 400 对：

```text
100  稳定任务回归验证
200  hard negative / 冲突任务
100  新 cluster / 新品牌 / 新模板
```

不要平均分到所有品牌。

---

# 24. 评测指标也必须改：F1 不是第一指标

建议生产 Gate：

```text
1. Auto-match Precision
2. Hard-negative False Positive Rate
3. Abstention Rate
4. Accepted Reference Coverage
5. Conflict Rate
6. New-cluster / Drift Rate
```

尤其要单独统计：

```text
same_brand_same_family_adjacent_reference_fp_rate
```

而不是只看 overall F1。

测试集拆分也建议：

```text
按时间切
按品牌切
按来源模板切
```

不要只 random split，否则相同标题模板/相同 reference family 会泄漏到 train/test 两边，得到过度乐观结果。

---

# 25. 线上 Drift 规则

建议维护两套信号。

## 25.1 数据漂移

```text
PSI(feature) > threshold
KS p/value-distance abnormal
C2ST 能高准确率区分 new vs old
```

## 25.2 业务风险漂移

```text
reference_conflict_rate ↑
unknown_brand_rate ↑
ref_dictionary_miss_rate ↑
accessory_rate ↑
OCR/title disagreement ↑
normalization_lossy_rate ↑
auto_accept_rate 突然变化
```

出现这些情况时不要自动“再训练然后继续发”。

应该：

```text
1. 降级该 task 为 review-only
2. 抽 hard cases 给人工
3. 更新 normalizer/extractor/model
4. 离线验证
5. 再恢复自动接受
```

这比单纯持续在线学习更安全。

---

# 26. 10M 级数据如何避免笛卡尔积

如果按传统 pairwise：

```text
10M × 10M
```

显然不可行。

Reference-first 后，大多数数据不需要 pairwise matcher。

流程是：

```text
N 条商品
-> O(N) reference extraction
-> O(N log N) / indexed upsert 到 reference entity
-> group by reference_entity_id
```

真正的跨来源链接天然是：

```text
reference_entity_id
```

组里的记录。

数据库索引：

```sql
CREATE INDEX idx_assignment_reference
ON product_reference_assignment(reference_entity_id)
WHERE status = 'accepted';
```

如果数据 1000 万级，PostgreSQL 本身已经可以支撑这类索引和 Join；图片/OCR、批量 profile 计算可以放到异步 worker / Spark/Flink，不必一开始就上复杂分布式 Matching 服务。

---

# 27. 推荐的实际技术栈

## MVP / 1000 万级可维护版本

```text
PostgreSQL
  - product metadata
  - reference_entity
  - assignments
  - task/model registry

S3 / MinIO
  - raw JSON
  - images
  - OCR artifacts
  - model artifacts

Python workers
  - parsing
  - OCR
  - normalization
  - feature calculation
  - task profiling

FastAPI
  - assignment / review / evidence API

Redis + Celery 或 Kafka
  - 异步 OCR / reprocess

scikit-learn / LightGBM
  - risk model

networkx / igraph / leidenalg
  - task similarity graph clustering
```

如果未来持续扩到更大：

```text
Kafka -> Flink/Spark Structured Streaming
Iceberg/Delta Lake -> 历史分析和重算
```

但当前 1M～10M 没必要一开始做成复杂大数据平台。

---

# 28. 推荐的增量处理流程

每来一条新商品：

```text
1. 保存 raw record
2. brand canonicalization
3. 读取结构化 reference
4. title/description candidate extraction
5. 必要时异步 OCR
6. 生成 reference_evidence
7. brand-specific normalization
8. evidence aggregation
9. 冲突检查
10. assignment state machine
11. accepted -> reference_entity_id
12. exact join 生成跨源关联
13. 写 task feature sample
```

每小时/每天：

```text
1. 更新 task distribution profile
2. PSI / KS drift check
3. 高 drift task 标记
```

每周或任务明显变化时：

```text
1. 重新构建 task similarity graph
2. Leiden cluster
3. 比较 cluster version
4. coverage / drift 判断是否补标
5. 更新 model registry
```

---

# 29. 可以直接写成的 Router 逻辑

```python
class TaskRouter:
    def route(self, task_profile):
        # 快路径：先和 cluster representative 比
        candidates = []

        for cluster in active_clusters():
            score = task_similarity(
                task_profile,
                cluster.representative_profile,
                metrics=["psi", "ks"]
            )
            candidates.append((score, cluster))

        best_score, best_cluster = max(candidates)

        # 新分布：不允许强行复用
        if best_score < MIN_TASK_SIMILARITY:
            return RouteDecision(
                mode="review_only",
                reason="new_distribution",
                trigger_recluster=True,
                trigger_labeling=True,
            )

        # 已知 cluster 但 coverage/drift 超标
        if best_cluster.coverage_ratio > COVERAGE_LIMIT:
            return RouteDecision(
                mode="restricted",
                cluster=best_cluster,
                trigger_retrain=True,
            )

        return RouteDecision(
            mode="normal",
            cluster=best_cluster,
            model=best_cluster.active_model,
        )
```

关键点：

```text
找不到合适模型时，系统输出 abstain/review
而不是“选最接近的凑合用”
```

这一步比原论文更符合当前 precision-first。

---

# 30. 建议增加“Cluster Safety Contract”

普通 Model Registry 只存模型。

当前项目建议每个 cluster 再存一套安全合同：

```json
{
  "cluster_id": 12,
  "allowed_evidence_modes": ["S-S", "S-T", "T-O"],
  "required": {
    "brand_equal": true,
    "canonical_ref_equal": true,
    "pattern_valid": true
  },
  "forbidden": {
    "reference_conflict": true,
    "accessory_ambiguous": true,
    "lossy_normalization_high": true
  },
  "auto_accept_tiers": ["A", "B"],
  "fallback": "review"
}
```

也就是说：

```text
Model 不拥有最终授权
Safety Contract 才拥有
```

模型只在合同内部提供风险分。

---

# 31. 为什么这比“一个统一大模型”更适合当前项目

如果训练一个统一模型：

```text
所有品牌
所有来源
所有抽取通道
所有时间窗口
-> 一个 matcher
```

会有几个问题：

1. Rolex、Omega、Cartier 的 reference 结构差异很大；
2. 三个平台字段质量不同；
3. OCR-only 和 structured-reference 不是同一个难度；
4. 网站改版以后旧模型可能悄悄失效；
5. 单模型总体 F1 很高，也可能隐藏某个小品牌极高 FP。

MoRER 的任务聚类思路正好把这些差异显式建模：

```text
相似任务共用模型
不同任务分开
新分布先发现再复用
```

这比“一套 prompt / 一个 Transformer 走天下”更可控。

---

# 32. 论文自身限制，以及对本项目的影响

论文在限制部分明确指出：

1. 需要有足够数量的已解决 ER tasks；
2. 如果历史任务太少，模型仓库的 feature diversity 不够，新任务可能出现明显 domain shift；
3. 当前方法假设不同 ER task 存在 common feature space；
4. 论文主要处理 feature-space heterogeneity，对属性完全不一致的情况需要先构造统一 representation。

当前项目正好有两个风险。

## 风险 1：来源只有三个

解决：

```text
Task 细化为 source_pair × brand × evidence_mode × time_window
```

## 风险 2：字段非常稀疏

解决：

不要强行要求：

```text
每个平台都必须有 title/brand/model/price
```

而是统一成“证据型 feature schema”：

```text
channel_present
candidate_count
pattern_valid
conflict_count
ocr_agreement
...
```

字段缺失本身也是 feature。

---

# 33. 和当前 `a` 目录里已有方案怎么组合

这篇 MoRER 不应该替代已有思路，而应该把它们编排起来。

可以形成：

```text
Reference-first 主键体系
        │
        ├── 高精度 reference 抽取 / 规范化
        │
        ├── 图片 OCR / 多模态证据
        │
        ├── Risk / Selective Prediction
        │
        ├── TransClean / 图一致性安全审计
        │
        └── MoRER
             负责 task clustering
             model routing
             drift detection
             label budget allocation
```

换句话说：

```text
MoRER 是“模型仓库和任务路由器”
TransClean 是“匹配图安全审计器”
Reference Entity 是“最终业务身份主键”
```

三者职责不冲突。

---

# 34. 一个建议的 MVP 顺序

## Phase 1：先做 deterministic Reference Backbone

必须先完成：

```text
brand canonicalization
reference_evidence
brand-specific normalizer
reference_entity
assignment state machine
exact join
```

这个阶段即使没有 ML，也已经可以安全产生一部分高 precision 匹配。

## Phase 2：补 Risk Model + Review

```text
几百对黄金标签
hard negatives
reference role classifier
risk/veto model
人工复核页面
```

## Phase 3：再做 MoRER Control Plane

```text
task definition
distribution profile
PSI / KS
similarity graph
Leiden
model registry
router
```

## Phase 4：增量 Drift / Coverage

```text
new task detection
review-only automatic downgrade
active labeling queue
model version update
```

## Phase 5：图片 OCR / 多模态补强

不要一开始就让大视觉模型决定 Match。

先把图片变成：

```text
可追溯的 reference evidence
```

价值更高。

---

# 35. 最终推荐方案

如果现在要直接为 Spec 定技术路线，我建议采用下面这个最终架构：

```text
                 ┌─────────────────────────────┐
                 │        Raw Product          │
                 │ 雷小安 / 腕表之家 / 奢当家    │
                 └──────────────┬──────────────┘
                                v
                   ┌─────────────────────────┐
                   │ Reference Evidence      │
                   │ structured/title/OCR    │
                   └────────────┬────────────┘
                                v
                   ┌─────────────────────────┐
                   │ Brand-aware Canonicalizer│
                   └────────────┬────────────┘
                                v
                    ┌────────────────────────┐
                    │ Evidence Aggregation   │
                    │ conflict + risk + tier │
                    └────────────┬───────────┘
                                 v
                 ┌──────────────────────────────┐
                 │ Product Reference Assignment │
                 │ accepted / review / reject   │
                 └──────────────┬───────────────┘
                                v
                    ┌────────────────────────┐
                    │ Reference Entity       │
                    │ UNIQUE(brand, ref)     │
                    └────────────┬───────────┘
                                 v
                       EXACT REFERENCE JOIN
                                 │
                                 v
                         Cross-source Entity


        ───────────────── Control Plane ─────────────────

         feature/task samples
                 │
                 v
       Distribution Profiler
          PSI / KS / C2ST
                 │
                 v
       Task Similarity Graph
                 │
               Leiden
                 │
                 v
         Task Cluster Registry
                 │
                 v
          Rule/Model Registry
                 │
                 v
       Router + Drift + Coverage
                 │
                 v
      label / retrain / downgrade
```

最关键的工程原则只有四条：

1. **最终自动匹配只认 Reference Entity exact equality。**
2. **LLM/ML/图片只能帮助抽取、验证、否决、路由和人工排序，不能越权创造 reference equality。**
3. **找不到可信 reference 就 abstain，不为 recall 强行补全。**
4. **把来源/品牌/证据通道/时间窗口建模成 Task，用 MoRER 的分布聚类和 coverage 思想管理持续增量变化。**

---

# 36. 对论文的最终评价

MoRER 对当前需求最有价值的不是“它能把 ER F1 提高多少”，而是它给出了一个很实用的生产思想：

> **把多源 Matching 看成很多不断出现、分布不同的任务；不要假设一个模型永远适用。先判断新任务像不像过去的任务，再决定复用、重聚类还是补标。**

对于三源二奢/腕表系统，这个思想可以直接升级为：

```text
Reference-first Data Plane
+
MoRER-style Model/Rule Control Plane
```

这样同时解决：

```text
precision-first
字段稀疏
来源持续变化
品牌规则不同
图片/OCR加入
少量人工标签
模型版本治理
线上分布漂移
```

而且不会把最核心的“同一 reference”业务定义交给一个概率模型猜。

**推荐落地优先级：高。**

但推荐的是：

```text
MoRER 的任务聚类 / 模型仓库 / drift / label-budget 思想
```

而不是：

```text
直接把 MoRER 的 pairwise classifier 输出当最终同款关系。
```
