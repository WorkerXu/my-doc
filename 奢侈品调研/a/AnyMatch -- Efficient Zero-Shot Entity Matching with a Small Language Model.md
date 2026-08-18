# AnyMatch -- Efficient Zero-Shot Entity Matching with a Small Language Model：面向二奢腕表 Reference Entity Linking 的落地分析

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取 **AnyMatch -- Efficient Zero-Shot Entity Matching with a Small Language Model** 进行深入分析；在执行前已检查 `奢侈品调研/a/`，当前标题尚未被 `a` 分析过。

- 论文：<https://arxiv.org/abs/2409.04073>
- 官方实现：<https://github.com/Jantory/anymatch>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

AnyMatch 的核心价值不是“把 GPT-2 换成一个便宜的大模型”，而是提供了一套非常适合**新增来源 / 新品牌 / 新数据分布快速迁移**的数据组织方法：

1. 把 Entity Matching 统一建模为 sequence classification；
2. 从多个已知数据集迁移，在完全未见的目标数据集上 zero-shot 推理；
3. 用 AutoML 找出传统模型容易错的**困难正例**，优先放入训练集；
4. 把 record-level 样本拆出 attribute-level 样本，让模型学习“字段值之间怎样才算一致”；
5. 控制正负样本比例，避免大规模候选集中负例淹没正例；
6. 使用只有约 124M 参数的 GPT-2 作为 matcher，大幅降低推理成本。

这些思想对“雷小安 × 腕表之家 × 奢当家”的持续增量匹配很有价值，但**官方 AnyMatch 不能原样作为最终自动合并器**。

原因很直接：当前 Spec 已经定义：

> 同一个商品 = 同一个 reference number / 型号，并且绝对不能误匹配，precision 极端优先，允许漏匹配。

而 AnyMatch 官方实现最终仍然是一个普通二分类器，训练和 early stopping 优化的是 F1，推理使用 `argmax` 直接输出 0/1，并没有：

- reference 硬约束；
- 冲突否决；
- probability calibration；
- abstain / review 状态；
- precision 下界控制；
- 对 identifier 角色的明确建模。

因此本方案的核心改造是：

> **保留 AnyMatch 的“跨数据集迁移 + hard-example selection + attribute-level augmentation + 小模型低成本推理”，但把它降级为 Reference Entity Linking 流程中的候选复核器 / 难例排序器；最终自动放行仍由 reference 的确定性证据决定。**

推荐最终架构不是：

```text
商品 A + 商品 B
      ↓
AnyMatch
      ↓
match / unmatch
```

而是：

```text
商品记录
   ↓
Reference Evidence Extraction
   ↓
Canonical Reference Registry 候选召回
   ↓
硬规则冲突过滤
   ↓
AnyMatch-style Small Matcher（只处理疑难候选）
   ↓
Precision-first Decision Gate
   ├── AUTO_LINK
   ├── REVIEW
   └── REJECT
   ↓
Reference Entity ID
```

两条商品记录最终是否“相同”，只需要判断是否链接到同一个经过验证的 `reference_entity_id`。

---

## 1. AnyMatch 解决的问题

传统 Entity Matching 往往要求目标数据集本身有一批人工标注 pair。例如，要匹配两个商品表，先标几百或几千对“同一商品 / 不同商品”，然后为这一对数据源单独训练 matcher。

AnyMatch 关注的是更难的 **zero-shot entity matching**：

- 目标数据集以前没见过；
- 目标数据集没有标注 pair；
- 甚至推理时不依赖 column name / column type；
- 但可以利用其他已经有标签的数据集做 transfer learning。

论文的形式化过程可以概括为：

```text
多个历史标注 EM 数据集
D_transfer(1), D_transfer(2), ..., D_transfer(m)
          │
          ├── record-level sampling
          ├── hard positive selection
          ├── label balancing
          └── attribute-level augmentation
          │
          ▼
      D_fine-tune
          │
          ▼
   fine-tune GPT-2
          │
          ▼
  一个通用 matcher f
          │
          ▼
未见目标数据 D_target
无需为该目标数据重新标注即可推理
```

这和当前业务的“持续新增来源 / 品牌 / 批次”非常契合：我们不希望每来一个新数据源都重新从零训练一个 matcher。

但需要注意：AnyMatch 解决的是**通用 EM 泛化**，当前业务解决的是更窄、更强约束的 **reference identity linking**。后者反而可以利用业务定义构造比通用 EM 更安全的系统。

---

## 2. AnyMatch 的核心技术架构

### 2.1 将 Entity Matching 转成 Sequence Classification

论文和代码都把左右两条记录的值序列化成文本，然后交给语言模型做二分类。

官方默认序列化 `mode1` 的形式类似：

```text
Record A is <p>COL value1, COL value2, COL value3</p>.
Record B is <p>COL value1, COL value2, COL value3</p>.
Given the attributes of the two records, are they the same?
```

其中：

- `COL` 只标识“这里开始是一个字段值”；
- `<p>...</p>` 把一条 record 包起来；
- 缺失值统一写成 `N/A`；
- 默认不把字段名传给模型。

官方 `utils/data_utils.py::df_serializer()` 实现了 4 种 serialization mode。

最重要的设计点是：**AnyMatch 故意降低对 schema name 的依赖**。论文消融实验甚至显示，在其 zero-shot benchmark 下，把实际 column name 加入 prompt 反而使平均 F1 略降，作者将其解释为字段名会让模型更容易绑定到已知 schema，从而降低跨数据集迁移性。

对腕表项目的启发是：

- “平台字段名”不能成为核心语义，例如 `goods_no`、`model`、`款号`、`货号` 在不同平台含义可能不同；
- 但当前业务不能完全照搬“无字段名”，因为 **reference、platform_sku、seller_sku、series、material 的风险等级完全不同**；
- 所以生产系统应该采用“**稳定语义角色**”而不是“原始平台字段名”。

例如序列化为：

```text
ROLE brand VAL Rolex
ROLE ref_explicit VAL 126610LN
ROLE title VAL 劳力士潜航者黑水鬼 41mm 126610LN
ROLE series VAL Submariner
ROLE source VAL xiaoleian
```

而不是：

```text
COL Rolex, COL 126610LN, COL 劳力士潜航者..., COL Submariner
```

这样既避免绑定某个平台 schema，又把 reference 这种高风险字段的“角色”显式保留下来。

### 2.2 小模型：GPT-2 Sequence Classification

官方 `model.py` 默认使用：

```python
GPT2ForSequenceClassification.from_pretrained('gpt2')
```

论文使用约 124M 参数的 GPT-2；同时提供 T5、BERT 作为消融对照。

论文报告 AnyMatch 在 9 个 benchmark 上的平均预测质量仅比当时基于 GPT-4 的 MatchGPT 低约 4.4%，但模型参数少约四个数量级，按论文当时的价格假设推理成本低 3899 倍。

在论文的 4×A100 吞吐实验中，GPT-2 版本达到约 69 万 token/s，而对比的大模型方案低得多。这个实验不应该直接换算成我们的生产 QPS，因为：

- 硬件不同；
- 输入长度不同；
- benchmark 比真实中文二奢字段更简单；
- 我们最终还需要候选生成、reference extraction 和规则决策。

但它说明一个非常关键的工程原则：

> **只要把大规模候选控制住，小型 matcher 可以作为一个成本可接受的批量复核层；没有必要把每个候选都送给闭源大 LLM。**

### 2.3 Leave-One-Dataset-Out Transfer Learning

官方 `loo.py` 固定使用 9 个数据集，并在实验中每次 leave one dataset out：

```python
dataset_names = [
  'abt', 'amgo', 'beer', 'dbac', 'dbgo',
  'foza', 'itam', 'waam', 'wdc'
]
```

例如测试 `wdc` 时，就用其余 8 个数据集生成训练数据；这模拟“目标数据从未见过”的 zero-shot 场景。

其思想可以直接改造成腕表领域版本：

```text
已覆盖品牌 / 来源 / 时间段
        ↓
构造 transfer tasks
        ↓
留出一个品牌或来源作为 unseen domain
        ↓
验证 matcher 是否能迁移
```

我们不应只做随机 train/test split。真正有业务价值的测试应该包括：

- leave-one-source-out：训练不见“奢当家”，只在测试时出现；
- leave-one-brand-out：训练不见某品牌，新品牌直接上线；
- time-split：训练只使用某时间点以前的数据，测试新抓取批次；
- unseen-reference split：测试 reference 在训练集中从未出现。

如果随机切分，同一个 reference 或相似标题模板很可能同时出现在 train/test，得出的 F1 会过于乐观。

---

## 3. AnyMatch 最值得复用的部分：困难样本选择

### 3.1 AutoML Filter 的真实逻辑

论文提出用简单 AutoML 模型找“难例”。官方实现位于：

```text
utils/data_utils.py::automl_filter()
```

流程不是“选全部模型预测错误的 pair”，而是更具体：

1. 如果某个 transfer dataset 少于 1200 pair，全部保留；
2. 如果大于 1200：
   - 读取 AutoML 对训练集的预测；
   - 找出 **label=1 但 AutoML 判错的正例**；
   - 正例最多保留约 400；
   - 如果困难正例不足，再从正确预测的正例补足；
   - 随机采样两倍数量的负例；
   - 最终约形成 `1 positive : 2 negative` 的 record-level 训练集。

论文算法描述也是同样思路：优先把普通模型难以识别的 matching pair 纳入 fine-tuning。

这对腕表业务非常有用，但我们的“难例”应该反过来强化 **false-positive boundary**。

官方 AnyMatch 更关注：

> 为什么一个真实 match 很难被普通模型认出来？

当前业务更应该关注：

> 为什么两个不同 reference 看起来特别像，容易被模型误合并？

因此建议改造成 **Hard Positive + Hard Negative 双向挖掘**：

#### Hard Positive

同一 canonical reference，但：

- 一个来源写 `126610LN`，另一个写 `126610 LN`；
- reference 只出现在标题 / OCR；
- 中文名、英文名混合；
- 字段严重缺失；
- 标题顺序、营销词差异大；
- 同 reference 的不同成色、年份、附件描述。

#### Hard Negative

不同 canonical reference，但：

- 只差 1 个字符；
- 同品牌同系列同尺寸；
- 外观几乎一样；
- 一个是前代 reference、一个是后代 reference；
- 标题里同时出现多个型号；
- 配件标题写“适配 126610LN”，但商品本身不是 126610LN；
- 平台 SKU 长得像品牌 reference；
- OCR 将 `0/O`、`1/I`、`5/S` 混淆。

在“绝不能误匹配”的业务里，**Hard Negative 比普通随机负例更重要**。

### 3.2 用现有黄金数据自动生成大量训练 pair

Spec 允许人工标几百对，但实际上我们可以利用 reference 的硬事实生成更多“弱监督但高可信”样本。

当一条商品已有高可信 reference 字段：

```text
brand = Rolex
canonical_reference = 126610LN
reference_source = explicit_field
reference_confidence = trusted
```

则可以自动构造：

#### 高可信正例

```text
(source=A, brand=Rolex, ref=126610LN)
(source=B, brand=Rolex, ref=126610LN)
=> positive
```

前提：两边 reference 都来自可信字段或人工验证源。

#### 高价值困难负例

```text
Rolex 126610LN  vs  Rolex 126610LV
Rolex 116610LN  vs  Rolex 126610LN
```

或者从 reference registry 中做 edit-distance / token-neighbor 采样。

这样人工的几百对黄金标签可以集中用于：

- reference 抽取歧义；
- 标题中多个型号角色判断；
- OCR 冲突；
- 新品牌格式；
- 模型最高分但硬规则不允许放行的 pair。

而不是浪费在显然相同或显然不同的随机 pair 上。

---

## 4. Attribute-Level Augmentation 如何迁移到腕表

AnyMatch 的第二个关键创新是 attribute-level training examples。

官方会从 record pair 中拆出单字段值 pair。例如一个完整匹配 record：

```text
name_l, brand_l, price_l, date_l
name_r, brand_r, price_r, date_r
```

会额外生成：

```text
name_l  vs name_r
brand_l vs brand_r
price_l vs price_r
...
```

官方代码 `read_multi_attr_data()` 还会：

- 按 attribute 分组；
- 每个 attribute 内正负样本配平；
- 最多采样 800 条；
- 再使用和 record-level 一致的 serializer。

论文消融显示，把 attribute-level 样本和 record-level 样本混合训练，对 zero-shot 泛化有明显帮助。

### 4.1 腕表领域不要直接复用“record 标签 = attribute 标签”

这里有一个非常重要的风险。

一个 record pair 是同一 reference，并不意味着每个属性都“相同”：

```text
同一 reference
price: 70000 vs 82000      # 完全可以不同
condition: 95新 vs 9成新   # 完全可以不同
year: 2021 vs 2023         # 完全可以不同
accessories: 单表 vs 全套  # 完全可以不同
```

所以不能简单把 record-level positive 标签复制给每个业务字段。

更安全的做法是按字段定义任务类型：

### Identity attributes

强身份字段：

- `canonical_reference`
- `brand`
- 某些品牌特定 reference family / caliber 约束

这些字段适合训练“是否一致 / 是否冲突”。

### Descriptive attributes

描述字段：

- series
- model_name
- case_size
- material
- dial_color

这些只能做辅助一致性 / 冲突检测。

### Instance attributes

实例级变化字段：

- price
- condition
- seller
- stock status
- accessories
- listing time

这些**不能用来定义 reference identity**。

因此建议将 AnyMatch 的 attribute-level augmentation 改为多个辅助任务：

```text
reference_equivalence
brand_equivalence
series_consistency
model_name_consistency
identifier_role
accessory_mention_detection
```

最终 classifier 只消费这些结构化证据，而不是把所有字段相似性平均掉。

---

## 5. 官方实现的代码级审查：不能原样生产的点

### 5.1 推理使用 argmax，不支持 precision-first

官方 `utils/train_eval.py` 中对 GPT/BERT 的预测是：

```python
logits = output[:2][1]
pred = logits.argmax(axis=-1)
```

评估只计算：

```text
F1
Accuracy
```

训练 early stopping 也以 validation F1 为最佳模型标准。

这与当前业务目标直接冲突。

假设：

```text
threshold 0.50: precision 95%, recall 92%
threshold 0.995: precision 99.99%, recall 45%
```

通用 benchmark 可能更喜欢前者，但我们的业务一定选择后者，甚至还要加硬规则。

生产版必须返回：

```text
p_match
p_nonmatch
model_version
```

并支持：

```text
AUTO_LINK / REVIEW / REJECT
```

而不是 `argmax -> bool`。

### 5.2 训练目标和 early stopping 也必须从 F1 改掉

建议最少同时监控：

```text
precision@auto_accept
recall@auto_accept
false_positive_count
coverage
PR-AUC
hard-negative FPR
unseen-reference FPR
```

对于上线 gate，建议使用：

```text
在 validation hard set 上：
false_positive_count == 0
并且 lower confidence bound(precision) >= 业务目标
```

如果样本量不足以证明统计下界，则自动降级为 REVIEW，而不是放宽阈值。

### 5.3 官方 serializer 丢失字段语义，不适合 identifier 风险控制

`mode1` 只输出 `COL value`，故意忽略字段名。

对普通 zero-shot EM，这是提高 schema transferability 的设计；对当前任务却可能把：

```text
reference = 126610LN
platform_sku = 126610LN
compatible_model = 126610LN
```

当成等价证据。

生产版应先做 canonical role mapping：

```text
原始字段名
   ↓
Source Adapter
   ↓
稳定 semantic role
```

例如：

```text
ref_explicit
platform_sku
seller_sku
model_name
series
brand
ocr_text
compatibility_text
```

模型看到的是 role，不是平台原始字段名。

### 5.4 GPT-2 英文 tokenizer 不适合直接处理中文二奢文本

官方实验主要针对公开 benchmark，并默认 `gpt2` / `bert-base-uncased`。

我们的标题很可能包含：

```text
劳力士 潜航者 黑水鬼 41毫米 126610LN 全套附件
```

直接照搬英文 GPT-2 tokenizer 会带来：

- 中文 tokenization 效率差；
- 序列更长；
- 中文领域词迁移能力不足；
- reference 周围上下文识别效果不稳定。

因此“AnyMatch”应理解为一种训练架构，而不是必须使用 GPT-2。

生产上应换成支持中文/多语、参数规模仍较小的 sequence classifier，并使用我们自己的 domain transfer data 重新 fine-tune。

### 5.5 长序列处理策略太粗

官方 `data.py` 在样本 token 数超过 `max_len` 时直接过滤掉，而不是字段优先级截断。

训练脚本常用：

```text
max_len = 350
```

这会导致长 record 的样本整个消失。

当前业务应做 field-aware truncation：

优先级：

```text
ref evidence
> brand
> model / series
> title 中 reference 周边窗口
> OCR reference 周边窗口
> material / size
> 其他长描述
```

不能因为商品描述很长，就把含有关键 reference 的样本丢掉。

### 5.6 `load_model()` 的 BERT 分支不真正尊重传入模型名

官方 `model.py`：

```python
elif 'bert' in base_model:
    model = BertMatcher('bert-base-uncased')
```

也就是说即使调用者传入另一个包含 `bert` 的模型名，实际仍被固定成 `bert-base-uncased`。

如果我们想替换中文/多语 encoder，必须重构 model factory，而不能只改 CLI 参数。

### 5.7 AnyMatch 本身不是 Blocking 系统

论文明确把 matcher 视为可以接在 blocked candidate set 后面的组件。

即使 matcher 很便宜，也不能做：

```text
10,000,000 × 10,000,000
```

全笛卡尔 pair inference。

百万到千万规模首先必须解决 candidate generation；AnyMatch 只负责候选集后处理。

---

## 6. 当前业务应从 Pairwise Matching 重构为 Reference Entity Linking

Spec 已经给了一个极强业务定义：

> 同一商品 = 同一 reference number / 型号。

这意味着最合理的数据模型不是构造大量：

```text
product A ↔ product B
```

而是建立一个全局 Reference Registry：

```text
product A -> ref_entity_123
product B -> ref_entity_123
product C -> ref_entity_456
```

然后：

```text
same_product(A, B)
= A.reference_entity_id == B.reference_entity_id
```

### 6.1 Reference Registry

建议表结构：

```sql
reference_entity (
    ref_entity_id           bigint primary key,
    brand_id                bigint not null,
    canonical_reference     varchar not null,
    strict_key              varchar not null,
    family_key              varchar,
    series                  varchar,
    model_name              varchar,
    status                  varchar,
    created_at              timestamp,
    updated_at              timestamp,
    unique (brand_id, strict_key)
)
```

其中：

- `canonical_reference`：展示用标准形式；
- `strict_key`：只做保守、可证明安全的规范化；
- `family_key`：可用于候选召回，但**不能用于自动合并**。

例如可以有两套 normalization：

```text
raw:       "126610 LN"
strict:    "126610LN"
family:    "126610LN"
```

但对于某些品牌，如果 `-`、`.`、空格具有真实型号语义，就不能简单删除。Normalization 必须按品牌配置和测试。

### 6.2 Reference Evidence 独立建模

每个“发现的 reference”都不要直接覆盖商品字段，而应保存 provenance：

```sql
reference_evidence (
    evidence_id             bigint primary key,
    product_id              bigint not null,
    raw_value               varchar not null,
    canonical_candidate     varchar,
    evidence_type           varchar not null,
    field_name              varchar,
    source_position         varchar,
    extractor_version       varchar,
    confidence              double,
    role                    varchar,
    is_conflicted           boolean,
    created_at              timestamp
)
```

`evidence_type` 示例：

```text
EXPLICIT_FIELD
TITLE_REGEX
TITLE_MODEL
OCR_BACKCASE
OCR_CARD
OCR_TAG
MANUAL
REFERENCE_CATALOG
```

`role` 示例：

```text
PRODUCT_REFERENCE
PLATFORM_SKU
SELLER_SKU
COMPATIBLE_REFERENCE
ACCESSORY_REFERENCE
UNKNOWN_IDENTIFIER
```

这样系统能回答：

> “为什么这条商品被链接到了 126610LN？”

而不是只有一个黑盒概率。

---

## 7. 可直接落地的生产架构

推荐整体架构：

```text
              ┌───────────────────────┐
              │ 雷小安 / 腕表之家 / 奢当家 │
              └──────────┬────────────┘
                         │ ingest
                         ▼
              ┌───────────────────────┐
              │ Raw Product Store     │
              │ 原始字段 + 图片 + 版本     │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │ Source Adapter        │
              │ 字段角色统一 / brand normalize│
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │ Reference Extractor   │
              │ field/title/OCR/role  │
              └──────────┬────────────┘
                         ▼
              ┌───────────────────────┐
              │ Evidence Resolver     │
              │ strict normalize      │
              │ conflict detection    │
              └─────┬─────────┬───────┘
                    │         │
          trusted exact       ambiguous
                    │         │
                    ▼         ▼
           ┌────────────┐  ┌──────────────────┐
           │ Exact Link │  │ Candidate Retrieval│
           └─────┬──────┘  │ Reference Registry │
                 │         └────────┬─────────┘
                 │                  ▼
                 │         ┌──────────────────┐
                 │         │ AnyMatch-style   │
                 │         │ Small Matcher    │
                 │         └────────┬─────────┘
                 │                  ▼
                 └────────────►┌────────────────┐
                              │ Decision Gate  │
                              │ rules+calibrate │
                              └───┬────┬───────┘
                                  │    │
                        AUTO_LINK │    │ REVIEW/REJECT
                                  ▼    ▼
                         Reference Link   Human Queue
                                  │
                                  ▼
                           Entity Cluster
```

---

## 8. Fast Path：大多数记录不要调用模型

如果两边都有可信的 explicit reference，则最安全也最便宜的办法仍然是 exact link。

### 8.1 自动链接条件

建议至少满足：

```text
1. brand 已规范化且一致
2. reference evidence role = PRODUCT_REFERENCE
3. strict canonical reference 唯一
4. reference 在 registry 中存在或通过可信流程新建
5. 当前 record 内没有第二个冲突的高可信 PRODUCT_REFERENCE
6. 没有 accessory / compatibility 语义
```

满足后：

```text
product -> reference_entity_id
```

不需要 AnyMatch。

这一步应该覆盖尽可能多的数据，因为：

- 速度最快；
- 可解释；
- 可审计；
- 最容易保证 precision。

### 8.2 为什么不是“reference 字符串相似”

例如：

```text
126610LN
126610LV
```

编辑距离极小，但业务上是不同 reference。

所以：

```text
string similarity
embedding similarity
visual similarity
```

都不能代替最终 strict reference equality。

---

## 9. Slow Path：AnyMatch 应该放在哪里

AnyMatch 最适合处理以下情况：

### Case A：一边 reference 明确，另一边只在标题中疑似出现

```text
A.ref_explicit = 126610LN
B.title = "劳力士潜航者黑水鬼41毫米 126610 LN 全套"
```

流程：

```text
B title extractor
 -> candidate 126610LN
 -> role check
 -> candidate ref entity
 -> AnyMatch / auxiliary verifier
 -> Decision Gate
```

### Case B：OCR 提取 reference，但 OCR 有轻微噪声

```text
OCR raw = 1266I0LN
candidate registry:
  126610LN
  126610LV
```

AnyMatch 可以结合：

- 标题；
- 品牌；
- 系列；
- OCR 周边文本；
- 候选 reference metadata；

给候选排序。

但如果没有任何确定性证据恢复到唯一 reference，最终应该 REVIEW，而不是因为视觉/文本“很像”就自动合并。

### Case C：新平台字段语义未知

例如新平台出现：

```text
code = 126610LN
```

它究竟是 reference 还是平台 SKU？

可以用 AnyMatch-style transfer model 做 identifier role classification，先确定字段/字符串角色，再进入 reference linking。

### Case D：新品牌刚接入，规则库还不成熟

AnyMatch 的 zero-shot / transfer learning 思路在这里最有价值：

- 已有品牌形成 transfer set；
- 新品牌没有大量人工 pair；
- 模型先做候选排序与 review prioritization；
- 人工复核反馈逐步沉淀成品牌规则和黄金集；
- 达到安全阈值以后再扩大自动覆盖。

---

## 10. Precision-first Decision Gate

这是整个方案与 AnyMatch 官方实现最大的差别。

### 10.1 三态而不是二态

最终决策必须是：

```text
AUTO_LINK
REVIEW
REJECT
```

而不是：

```text
MATCH
UNMATCH
```

### 10.2 推荐规则

伪代码：

```python
def decide(product, candidate_ref, evidence, model_score):
    # 1. 硬冲突直接拒绝
    if evidence.has_trusted_brand_conflict:
        return REJECT

    if evidence.has_two_distinct_trusted_product_refs:
        return REVIEW

    if evidence.has_explicit_ref_conflict(candidate_ref):
        return REJECT

    if evidence.reference_role in {
        'PLATFORM_SKU',
        'SELLER_SKU',
        'COMPATIBLE_REFERENCE',
        'ACCESSORY_REFERENCE'
    }:
        return REJECT

    # 2. 最强路径：可信 reference exact
    if evidence.has_trusted_exact_product_reference(candidate_ref):
        return AUTO_LINK

    # 3. 弱证据只能进入保守模型路径
    if evidence.has_unique_weak_reference(candidate_ref):
        if model_score >= REVIEW_HIGH_THRESHOLD:
            return REVIEW
        return REJECT

    return REJECT
```

如果业务真的把“绝对不能误匹配”理解为字面意义的 **zero tolerated false positive**，则建议：

> 模型分数永远不能单独产生 `AUTO_LINK`；模型只负责把疑难记录排序给人工，或作为 veto 信号。

如果后续业务接受“统计上极低风险”的自动化，则可以增加第二条自动路径，但必须经过独立 calibration + risk control，不能直接使用 0.5。

---

## 11. 把 AnyMatch 的训练数据策略改造成 WatchMatch

可以做一个内部小模型，暂称 `WatchMatch`，训练思想继承 AnyMatch。

### 11.1 Transfer Task 的组织

不要仅按“平台 A × 平台 B”拆任务，建议多个维度：

```text
source pair
brand
time window
reference family
extraction channel
```

形成诸如：

```text
Rolex / 雷小安 × 腕表之家
Omega / 雷小安 × 奢当家
Cartier / 腕表之家 × 奢当家
OCR-only samples
title-only samples
explicit-ref samples
```

### 11.2 Record-level 样本

序列化建议：

```text
Record A:
ROLE brand VAL ROLEX
ROLE ref_candidate VAL 126610LN
ROLE ref_source VAL TITLE
ROLE title VAL 劳力士潜航者黑水鬼41mm 126610LN
ROLE series VAL SUBMARINER

Record B:
ROLE brand VAL ROLEX
ROLE ref_candidate VAL 126610LN
ROLE ref_source VAL CATALOG
ROLE model_name VAL Submariner Date
ROLE series VAL SUBMARINER

Question: Do both records support the exact same product reference?
```

注意问题应从泛化的：

```text
are they the same?
```

改为：

```text
do both records support the exact same product reference?
```

让训练目标与业务定义一致。

### 11.3 Attribute-level 样本

建议只生成经过任务定义的 attribute pair：

```text
[reference]
126610 LN  <-> 126610LN       positive equivalence
126610LN   <-> 126610LV       negative equivalence

[brand]
劳力士      <-> Rolex          positive
Rolex      <-> Tudor           negative

[identifier role]
"型号126610LN" -> PRODUCT_REFERENCE
"货号LX1234"  -> PLATFORM_SKU
"适配126610LN表带" -> COMPATIBLE_REFERENCE
```

### 11.4 AutoML Filter 的业务版

建议先训练一个廉价 baseline：

```text
LightGBM / logistic / rules
```

特征：

```text
strict_ref_equal
ref_edit_distance
brand_equal
series_equal
model_name_similarity
ref_source_type
num_ref_candidates
has_accessory_terms
has_compatibility_terms
ocr_confidence
```

然后：

1. 选 baseline 误判的 positive；
2. 选 baseline 高置信误判的 negative；
3. 特别扩大 reference 近邻 hard negative；
4. 与普通样本混合训练小模型。

这样才真正把 AnyMatch 的“困难样本优先”迁移到 precision-first 场景。

---

## 12. 人工几百对黄金标签应该怎么花

不建议随机抽 500 pair。

建议分层：

```text
100 对：不同 reference 但极相似（单字符/后缀/代际差异）
100 对：标题多 reference / accessory / compatibility
80 对：OCR reference 歧义
80 对：reference 缺失、靠多个弱字段判断
60 对：新品牌 / 新来源
40 对：跨语言、缩写、别名
40 对：模型最高置信度的疑似 false positive
```

实际数量可以按数据分布调整。

黄金集必须保存“原因标签”：

```text
DIFFERENT_REFERENCE
ACCESSORY_MENTION
PLATFORM_SKU_CONFUSION
OCR_CONFUSION
BRAND_CONFLICT
INSUFFICIENT_EVIDENCE
TRUE_SAME_REFERENCE
```

这样不仅能算指标，还能定位下一轮系统应该修 extractor、normalizer 还是 matcher。

---

## 13. 评测设计：不能只看 F1

AnyMatch 论文为了通用 benchmark 主要报告 F1，这是合理的；当前系统需要完全不同的上线指标。

### 13.1 核心指标

```text
Auto-link Precision
Auto-link False Positive Count
Auto-link Coverage
Review Rate
Reject Rate
Reference Extraction Precision
Reference Extraction Coverage
```

其中最重要的是：

```text
Auto-link False Positive Count = 0
```

在经过专门设计的 hard test set 上尤其如此。

### 13.2 分桶指标

必须按以下维度拆开：

```text
source
brand
reference evidence type
new vs seen reference
new vs seen brand
OCR vs title vs explicit field
reference length/pattern
candidate count
```

一个平均 99.99% precision 的模型，可能在某个新品牌上只有 95%，这在本需求中仍然不可上线。

### 13.3 对抗测试集

至少建立：

```text
126610LN vs 126610LV
116610LN vs 126610LN
型号 vs 平台 SKU
腕表 vs 同型号表带/配件
标题出现两个 reference
OCR 单字符混淆
品牌冲突但 reference 字符串相同
同系列同外观不同 reference
```

这类 test case 才是业务真正的“死亡测试”。

---

## 14. 图片应该如何接入

AnyMatch 本身是文本 / 结构化 record matcher，不处理图片。

当前 Spec 有图片，因此建议图片只承担两类职责。

### 14.1 OCR 作为 reference evidence

对：

- 表背；
- 保卡；
- 吊牌；
- 盒标；

做 OCR，抽取 identifier candidate。

OCR 输出必须保存：

```text
raw OCR token
bounding box
image id
OCR confidence
candidate canonical reference
```

如果 OCR 与标题 reference 一致，证据增强；如果冲突，进入 REVIEW。

### 14.2 视觉相似只做 veto / review aid

不要使用：

```text
图片很像 => 同 reference
```

同系列腕表不同 reference 视觉可能极近。

视觉更适合：

```text
文本声称 Rolex，但图片明显不是 Rolex -> veto
reference 候选说绿色表圈，但图像明显黑色 -> conflict signal
```

即“视觉可以帮助否决或人工复核”，不能越权替代 reference identity。

---

## 15. 百万～千万级的扩展方案

### 15.1 不做 product-product 全量匹配

如果有 1000 万商品，两两 pair 是不可行的。

核心优化是：

```text
product -> reference registry
```

而不是：

```text
product -> every other product
```

Reference Entity 数量通常远小于商品记录数。

### 15.2 分层候选召回

建议按成本从低到高：

#### Tier 0：trusted exact hash join

```text
(brand_id, strict_ref_key)
```

O(1) / indexed lookup。

#### Tier 1：brand + reference token inverted index

用于轻微格式噪声。

#### Tier 2：brand + series + identifier fuzzy retrieval

只召回少量 reference candidates，例如 Top-5。

#### Tier 3：AnyMatch rerank

只对 Tier 1/2 的少量疑难候选运行小模型。

如果 90% 记录可以通过 exact path 处理，模型实际只处理剩余 10%；若每条疑难记录只产生 3～5 个 reference candidate，推理规模将从不可接受的笛卡尔积降到线性量级。

这里的 90% 只是架构示意，真实比例需要在三源数据上统计后确定。

---

## 16. 增量处理

系统要支持持续新增商品，不应定期全量重跑。

建议每条 product 有：

```text
raw_hash
normalizer_version
extractor_version
registry_version
matcher_version
decision_policy_version
```

增量流程：

```text
新/变更商品
  ↓
raw_hash changed?
  ├─ no -> skip
  └─ yes
       ↓
normalize + extract
       ↓
evidence changed?
       ↓
重新 link 当前商品
       ↓
写入 decision + reason + version
```

Reference Registry 变化时，只需要重放受该 reference family 影响的记录，而不是 1000 万全量。

### 16.1 决策必须可重放

保存：

```sql
match_decision (
    decision_id,
    product_id,
    candidate_ref_entity_id,
    decision,
    reason_code,
    hard_rule_result,
    model_score,
    matcher_version,
    policy_version,
    evidence_snapshot_id,
    created_at
)
```

当以后发现某条 normalization rule 有 bug，可以精确定位受到影响的历史 link。

---

## 17. 推荐的直接落地版本

### Phase 1：先不用 AnyMatch，建立可解释的 Reference Core

实现：

```text
Source Adapter
Brand Normalizer
Reference Normalizer
Reference Registry
Evidence Store
Exact Linker
Conflict Detector
Review Queue
```

目标：

- 明确当前多少商品有 explicit reference；
- 明确 title 能高精度抽多少；
- 建立基准 precision；
- 收集 hard cases。

### Phase 2：实现 AnyMatch-style WatchMatch

训练数据来源：

```text
可信 exact link 自动生成正例
reference 近邻生成 hard negative
人工 review 结果作为 gold
历史多个品牌 / 来源作为 transfer datasets
```

模型目标：

```text
候选 reference 是否得到当前商品证据支持
```

不是泛化的“两个商品是不是一样”。

### Phase 3：上线为 REVIEW 排序器

先只用于：

```text
候选排序
人工队列优先级
冲突解释
hard-example mining
```

不产生自动合并。

### Phase 4：只开放被证明安全的自动路径

只有当某一证据桶：

```text
source=X
brand=Y
evidence_type=TITLE_EXTRACTED
rule_version=Z
```

在足够大的独立 hard set 上持续零 false positive，才考虑把这一桶从 REVIEW 升级到 AUTO_LINK。

这比设置一个全局模型阈值安全得多。

---

## 18. 推荐代码模块

可以按如下方式拆服务：

```text
watch_match/
├── adapters/
│   ├── leixiaoan.py
│   ├── watchhome.py
│   └── shedangjia.py
├── normalization/
│   ├── brand.py
│   ├── reference.py
│   └── rules/
├── extraction/
│   ├── explicit_field.py
│   ├── title_reference.py
│   ├── identifier_role.py
│   └── ocr_reference.py
├── registry/
│   ├── reference_repository.py
│   └── candidate_retrieval.py
├── matcher/
│   ├── serializer.py
│   ├── model.py
│   ├── hard_example_mining.py
│   ├── calibration.py
│   └── inference.py
├── decision/
│   ├── hard_rules.py
│   ├── policy.py
│   └── reason_codes.py
├── review/
│   └── queue.py
└── evaluation/
    ├── hard_cases.py
    ├── precision_report.py
    └── drift_report.py
```

AnyMatch 官方仓库里最值得概念复用的映射：

```text
AnyMatch                         WatchMatch
-----------------------------------------------------------
df_serializer()              -> semantic-role serializer
automl_filter()              -> hard pos/neg miner
read_multi_attr_data()       -> role-aware auxiliary samples
loo.py                       -> leave-source/brand/time-out eval
GPT2ForSequenceClassification -> small multilingual classifier
train_eval.py F1 early stop  -> precision/coverage risk-aware trainer
argmax inference             -> calibrated tri-state decision support
```

---

## 19. 一个端到端示例

输入：

```json
{
  "source": "奢当家",
  "source_item_id": "S123",
  "brand": "劳力士",
  "title": "劳力士 潜航者 黑水鬼 41mm 型号126610 LN 全套",
  "reference": null,
  "images": ["..."]
}
```

### Step 1：Brand normalize

```text
劳力士 -> ROLEX
```

### Step 2：Title reference extraction

```text
raw candidate = "126610 LN"
strict candidate = "126610LN"
role = PRODUCT_REFERENCE
```

### Step 3：Registry lookup

```text
(ROLEX, 126610LN)
 -> ref_entity_id = 90001
```

### Step 4：Conflict check

标题中没有第二个高可信 reference；没有“适配 / 表带 / 配件”等角色冲突。

### Step 5：模型复核

序列化商品证据与 reference entity 元数据，得到：

```text
p_support = 0.997
```

注意这个分数本身仍不等于自动合并资格。

### Step 6：Decision Gate

如果当前 policy 规定：

```text
TITLE_REFERENCE 只能 REVIEW
```

则：

```text
decision = REVIEW
reason = TITLE_ONLY_REFERENCE
```

人工确认后：

```text
product.reference_entity_id = 90001
```

该人工结果进入 gold set；当此证据桶积累足够独立样本且持续零 false positive，未来才可考虑升级成自动路径。

---

## 20. 最终建议

AnyMatch 对当前项目最有价值的不是 GPT-2 本身，而是下面四个思想：

1. **Transfer Learning**：把历史品牌 / 来源积累成一个可迁移的小型 matcher，新来源不必从零开始；
2. **Hard Example Selection**：训练资源集中到真正困难的边界样本；
3. **Attribute-Level Augmentation**：让模型学习字段值层面的关系，而不是只记完整 record；
4. **Small Model Inference**：疑难候选可以大批量本地推理，不依赖昂贵 LLM API。

但对当前 Spec，应该明确反对“直接把 AnyMatch 的二分类结果作为商品合并结果”。官方实现：

- 以 F1 做 early stopping；
- 以 argmax 做最终 0/1 推理；
- 没有 abstention；
- 不区分 reference 与平台 SKU 等 identifier role；
- 不处理图片；
- 也不负责 blocking。

因此最推荐的落地方式是：

> **Reference Entity Linking 作为主系统，AnyMatch-style 小模型作为 Slow Path；规则决定“能不能自动链接”，模型主要决定“疑难候选哪个更值得相信 / 更值得人工看”。**

如果只能优先实现一条链路，建议按以下顺序：

```text
1. Reference Registry
2. Strict Normalization
3. Evidence Provenance + Identifier Role
4. Exact Link + Conflict Gate
5. Review Queue
6. Hard-case Gold Set
7. AnyMatch-style WatchMatch
8. Calibration / Risk Control
9. OCR 与视觉辅助证据
```

这条路线最大化利用了业务定义本身带来的确定性，同时吸收 AnyMatch 在跨域泛化和低成本难例建模方面的优势，最符合“千万级、持续增量、字段稀疏、允许漏匹配但绝不能误匹配”的约束。

---

## 参考

- Zeyu Zhang, Paul Groth, Iacer Calixto, Sebastian Schelter. **AnyMatch -- Efficient Zero-Shot Entity Matching with a Small Language Model**. arXiv:2409.04073. <https://arxiv.org/abs/2409.04073>
- AnyMatch official implementation: <https://github.com/Jantory/anymatch>
- `model.py`: <https://github.com/Jantory/anymatch/blob/main/model.py>
- `utils/data_utils.py`: <https://github.com/Jantory/anymatch/blob/main/utils/data_utils.py>
- `utils/train_eval.py`: <https://github.com/Jantory/anymatch/blob/main/utils/train_eval.py>
- `loo.py`: <https://github.com/Jantory/anymatch/blob/main/loo.py>
