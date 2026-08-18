# DeepBlocker：把深度 Blocking 改造成 Reference-First 的千万级腕表候选召回层

> 分析人：c  
> 目标需求：跨源二奢/腕表商品实体匹配（雷小安 × 腕表之家 × 奢当家）  
> 核心约束：100 万–1000 万级持续增量；字段稀疏；reference 可能在结构化字段、标题或图片中；“同一个商品”定义为**同一 reference number / 型号**；**precision 极端优先，绝不能误匹配，允许漏匹配**。

## 0. 选题、去重与结论先行

本次从 `奢侈品文章调研.md` 选取：

- 项目：**DeepBlocker**
- GitHub：https://github.com/qcri/DeepBlocker
- 对应论文：Deep Learning for Blocking in Entity Matching: A Design Space Exploration（VLDB 2021）
- 需求 Spec：https://app.notion.com/p/3bf7b2a8538b812fba00fb258024ff31

执行前重新检查 `奢侈品调研/c/`，c 已经分析过：

1. `Ameli：Enhancing Multimodal Entity Linking with Fine-Grained Attributes.md`
2. `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
3. `Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification.md`
4. `TransClean：Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency.md`
5. `parts-distributor-sku-classifier.md`
6. `pyJedAI.md`

`DeepBlocker` 尚未被 c 分析，因此满足去重要求。

### 结论先行

DeepBlocker **非常值得借鉴，但绝不能直接作为最终匹配器上线**。

它最值得复用的不是某个具体神经网络，而是这条工程抽象：

```text
Tuple Embedding
    -> Vector Pairing / Top-K Retrieval
    -> Candidate Set
```

对当前需求，应该改造成：

```text
原始三源 listing
    -> 字段/编号角色归一化
    -> Reference Evidence Extraction
    -> Reference Exact Lookup（优先）
    -> Reference-Aware Blocking / ANN（只给疑难样本找候选）
    -> Strict Evidence Verifier
    -> AUTO_LINK / REJECT / ABSTAIN
```

其中最重要的一条原则是：

> **DeepBlocker 只能负责“不要漏掉可能正确的候选”，不能负责“证明两个商品相同”。最终自动合并必须由 canonical reference 的高置信证据决定。**

这与需求的定义完全一致：既然“同一个商品”被定义为“同一 reference number”，就不应该把视觉相似、标题相似或 embedding 相似本身提升成最终真值。

---

## 1. DeepBlocker 实际在做什么

DeepBlocker 是一个面向 Entity Matching 的 Blocking 项目。Blocking 的目的不是直接判定 match，而是把原本的笛卡尔积比较压缩成一个尽量高召回的小候选集。

仓库主流程在 `deep_blocker.py`：

```text
left_df / right_df
    -> validate_columns()
    -> preprocess_datasets()
    -> tuple_embedding_model.preprocess(all_text)
    -> get_tuple_embedding(left)
    -> get_tuple_embedding(right)
    -> vector_pairing_model.index(right_embeddings)
    -> vector_pairing_model.query(left_embeddings)
    -> topK_neighbors_to_candidate_set()
```

`DeepBlocker` 构造函数只接收两个核心组件：

```python
DeepBlocker(tuple_embedding_model, vector_pairing_model)
```

这个边界设计很好，因为“如何表示一条商品记录”和“如何从大量向量中找 Top-K 候选”被彻底解耦。

对本项目来说，可以保留这个接口思想，但替换掉默认实现：

```text
TupleEmbeddingModel
    DeepBlocker 默认：SIF / AutoEncoder / CTT / Hybrid
    本项目：ReferenceAwareTupleEmbedding

VectorPairingModel
    DeepBlocker 默认：ExactTopKVectorPairing
    本项目：BrandShardedANNPairing + lexical/reference index
```

---

## 2. 源码架构深入拆解

## 2.1 预处理：把多个字段直接拼成 `_merged_text`

`deep_blocker.py` 的 `preprocess_datasets()` 会：

1. 只保留 `cols_to_block`；
2. NaN 填空字符串；
3. 全部转成字符串；
4. 除 `id` 外的所有字段用空格连接成 `_merged_text`；
5. 后续 embedding 只处理这个字符串。

示例脚本 `main.py` 对 Amazon-Google 使用：

```python
cols_to_block = ["title", "manufacturer", "price"]
```

这对通用商品 Blocking 很自然，但对腕表 reference 是一个明显风险：

```text
brand / reference / platform_sku / title / series / price
```

这些字段不是等价 token。

尤其 `reference` 和平台自有 SKU 都可能是字母数字串，如果无类型地拼在一起，模型无法知道某个编号到底代表：

- 品牌官方 reference；
- 平台货号；
- 店铺 SKU；
- 订单/库存 ID；
- 配件兼容型号；
- 当前商品自己的型号。

所以本项目不能直接使用 `_merged_text`，必须先做**字段角色归一化**。

推荐中间结构：

```python
NormalizedListing(
    listing_id,
    source,
    canonical_brand_id,
    product_type,
    title_normalized,
    structured_reference_candidates,
    title_reference_candidates,
    ocr_reference_candidates,
    platform_sku,
    series,
    material,
    movement,
    image_embeddings,
)
```

然后 Blocking 使用的是经过角色筛选的字段，而不是原始字段大拼接。

---

## 2.2 Tuple Embedding：DeepBlocker 的核心设计空间

`tuple_embedding_models.py` 把 tuple embedding 做成抽象接口：

```python
class ABCTupleEmbedding:
    def preprocess(self, list_of_tuples): ...
    def get_tuple_embedding(self, list_of_tuples): ...
    def get_word_embedding(self, list_of_words): ...
```

当前仓库实现了几类方法。

### AverageEmbedding

最简单：FastText token embedding 求均值。

优点是便宜、无监督；缺点是会把不同 token 的精确身份“平均掉”。

对腕表尤其危险，例如：

```text
Rolex Submariner 126610LN
Rolex Submariner 126610LV
```

两条文本除了最后两个字符几乎完全相同，语义 embedding 很可能非常接近，但业务上是不同 reference，**绝不能自动合并**。

因此这类 embedding 只能做候选召回。

### SIFEmbedding

SIF 会统计 token 频率，对高频词降权，并可移除第一主成分。

其价值在于弱化诸如：

```text
全新 / 二手 / 正品 / 男表 / 自动机械 / 热卖 / 专柜
```

这类高频营销词。

对本项目可以借鉴 SIF 的思想，但不应把 reference token 也按普通频率权重处理。reference 是业务主键型证据，应单独进入 lexical/reference 通道。

### AutoEncoderTupleEmbedding

流程是：

```text
SIF 300d
    -> AutoEncoder
    -> 150d latent embedding
```

`dl_models.py` 的 AutoEncoder 是一个两层 encoder + 对称 decoder，以 MSE 重建输入。

这属于无监督压缩，适合把商品文本表示压缩后用于近邻检索。

但它优化的是“重建表示”，不是“保留 reference 的逐字符判别力”。因此不能期待它天然分开同系列、只差一个后缀的 reference。

### CTTTupleEmbedding

CTT 的关键思路更值得借鉴：项目会自动构造 synthetic pairs。

正例：

- 从同一 tuple 随机删除一部分 token，作为扰动后的同实体样本。

负例：

- 当前 tuple 与随机选择的另一 tuple 配对。

随后使用 Siamese summarizer：

```text
t1 -> shared MLP -> z1
                         -> abs(z1-z2) -> linear -> sigmoid
 t2 -> shared MLP -> z2
```

这是一种自监督地让模型学习“哪些扰动应该仍然相似”的方法。

### HybridTupleEmbedding

Hybrid 基本等于：

```text
AutoEncoder embedding
    + CTT synthetic pair training
```

也就是先做无监督压缩，再用 synthetic pair 训练 Siamese 结构。

---

## 2.3 一个值得注意的当前源码问题：CTT/Hybrid 训练出的 summarizer 没有真正用于最终 retrieval embedding

在当前仓库代码中：

- `CTTTupleEmbedding.preprocess()` 会训练 `self.ctt_model`；
- `HybridTupleEmbedding.preprocess()` 也会训练 `self.ctt_model`；
- 但是两个类的 `get_tuple_embedding()` 并没有调用 `self.ctt_model.get_tuple_embedding(...)`。

当前 CTT 返回的是底层 SIF embedding：

```python
embedding_matrix = torch.tensor(
    self.sif_embedding_model.get_tuple_embedding(list_of_tuples)
).float()
return embedding_matrix
```

Hybrid 返回的是 AutoEncoder embedding，而不是 CTT 的 `siamese_summarizer` 输出。

也就是说，按当前 GitHub 实现，CTT/Hybrid 中专门训练的 Siamese summarizer 并没有进入 Top-K retrieval 的实际向量。

这说明这个仓库更适合作为**研究原型/架构参考**，而不是直接复制到生产。

如果在本项目中采用同类训练方式，必须有单元测试验证：

```text
训练前 embedding
训练后 embedding
retrieval 真正使用的 embedding
```

三者的调用链要明确，不允许“模型训练了但线上没用到”。

---

## 2.4 Synthetic Pair 对腕表领域需要重写

DeepBlocker 当前：

```text
positive = 同一商品随机删 token
negative = 随机另一商品
```

随机负例对于腕表 reference 任务太容易。

真正危险的 false positive 通常是：

```text
同品牌 + 同系列 + 极近 reference
同品牌 + 外观近似 + 不同尺寸
同一基础型号 + 不同后缀
整表 vs 表带/表盒/配件
商品标题中出现“适配某型号”
平台 SKU 长得像品牌 reference
```

因此应该把 synthetic/hard negative 改成：

```text
Rolex 126610LN  vs  Rolex 126610LV
Rolex 116610LN  vs  Rolex 126610LN
Omega 210.30... vs  同系列邻近 reference
整表 reference vs “适配 reference 的表带”
同 source 的平台 sku vs 品牌 reference
```

正例扰动也要更谨慎：

允许扰动：

- 空格、大小写；
- `-`、`.`、`/` 等格式差异；
- 营销词增删；
- 品牌中英文别名；
- 标题顺序变化。

默认不允许把 reference 本体随机改字符，然后仍标正例。

如果为了“reference 缺失时也能召回”而生成 reference-masked view，也必须明确它只训练 Blocking，不参与最终 AUTO_LINK。

---

## 2.5 Vector Pairing：默认 ExactTopK 是最大规模瓶颈

`vector_pairing_models.py` 的 `ExactTopKVectorPairing` 实现非常直接：

```python
all_pair_cosine_similarity_matrix = 1 - distance.cdist(
    query_embeddings,
    indexed_embeddings,
    metric="cosine"
)

topK_indices_each_row = np.argsort(
    -all_pair_cosine_similarity_matrix
)[:, :K]
```

这意味着它显式计算：

```text
N_left × N_right
```

完整相似度矩阵。

复杂度近似：

```text
时间：O(N × M × D)
内存：O(N × M)
```

如果两边各 100 万条，单是相似度矩阵就有 10^12 个元素，根本不可行，更不用说 1000 万级。

因此生产实现必须替换 Vector Pairing 层。

推荐：

```text
Brand/Category hard shard
    -> lexical/reference index
    -> ANN vector index
```

ANN 可以使用：

- FAISS HNSW / IVF 系列；
- hnswlib；
- Milvus / Qdrant 等独立向量服务；
- 若已有搜索基础设施，也可以把 lexical + vector 放入同一个检索系统。

这里 ANN 的目标是**高召回候选生成**，不是最终判定，所以近似检索本身不会破坏 precision；它最多影响 recall，而最终 precision 由下游 Reference Gate 控制。

---

## 2.6 Candidate Set 的 ID 处理也不能直接照搬

`blocking_utils.topK_neighbors_to_candidate_set()` 把：

```text
DataFrame row index
```

直接当成：

```text
ltable_id / rtable_id
```

虽然 `DeepBlocker.validate_columns()` 强制要求有 `id` 列，但候选输出实际上没有把原始 `id` 映射回来。

研究数据集的 `id` 往往正好和行号一致，所以不容易暴露问题；生产数据中一定不能这样做。

必须显式维护：

```python
left_pos_to_listing_id
right_pos_to_reference_id
```

ANN 返回内部 position 后，再安全映射回不可变业务 ID。

否则经过过滤、重排、分区、增量合并后，行号变化会造成灾难性的错误关联。

---

## 2.7 DeepBlocker 的评估方式反而很值得保留

`compute_blocking_statistics()` 只评估两个 Blocking 指标：

```text
recall = 真匹配中有多少被候选集覆盖
cssr   = candidate_count / (left_count × right_count)
```

这是正确的职责划分：Blocking 不应该拿最终 precision 来定义自己。

本项目也应严格分层评估：

### Candidate/Blocking 层

- candidate recall@K；
- 每条 listing 的候选数 p50/p95/p99；
- candidate reduction ratio；
- 索引构建时间；
- 增量写入吞吐；
- query latency。

### Final Decision 层

- AUTO_LINK precision；
- false positive 数量；
- abstain rate；
- hard-negative false positive rate；
- 各 source / brand / field 的错误分布。

> Blocking 宁可多给几个候选；最终 Gate 宁可大量 abstain，也不能为了 recall 放宽自动合并条件。

---

# 3. 为什么 DeepBlocker 原版不能直接用于当前需求

可以概括成 7 个问题。

## 3.1 它解决的是“Blocking”，不是“同 reference 证明”

Embedding 相似只能说明“值得比较”，不能说明“是同一 reference”。

## 3.2 字段拼接会丢失编号角色

`reference`、`platform_sku`、兼容型号不应处于同一语义层。

## 3.3 FastText/语义 embedding 会过度平滑邻近型号

对自然语言这是优点，对 reference 是风险。

## 3.4 随机负例太简单

生产风险来自同系列邻近 reference，而不是随机跨品类商品。

## 3.5 ExactTopK 无法支撑百万–千万级

必须使用分片和 ANN/lexical retrieval。

## 3.6 当前候选 ID 使用行号

生产必须映射真实业务 ID。

## 3.7 当前训练/索引方式不是增量服务架构

`block_datasets()` 会把左右表拼起来重新 preprocess / 训练，再生成 embedding 和索引。持续增量场景需要：

```text
稳定模型版本
+ 持久化 canonical reference index
+ 新 listing 只做增量 embedding/query
+ 参考库变更时局部更新索引
```

而不是每批数据重新训练全部模型。

---

# 4. 推荐落地：Reference-First Deep Blocking Architecture

## 4.1 不要做三平台全量 pairwise matching

最重要的架构变化是把问题从：

```text
雷小安 listing ↔ 腕表之家 listing
雷小安 listing ↔ 奢当家 listing
腕表之家 listing ↔ 奢当家 listing
```

改成：

```text
任意来源 listing
    -> Canonical Reference Entity
```

Canonical Reference Entity 的唯一键建议为：

```text
(canonical_brand_id, canonical_reference)
```

例如：

```text
(Rolex, 126610LN)
(Omega, 210.30.42.20.01.001)
```

于是跨源是否为“同一个商品”变成：

```text
listing A -> ref_entity_123
listing B -> ref_entity_123
=> 同一 reference entity
```

新增第四、第五个平台时，不需要增加新的 pairwise matcher。

---

## 4.2 总体架构

```text
                    ┌─────────────────────────┐
三源 Raw Listing -->│  Source Adapter         │
                    │  schema/field mapping   │
                    └───────────┬─────────────┘
                                v
                    ┌─────────────────────────┐
                    │ Normalization           │
                    │ brand/type/text/id role │
                    └───────────┬─────────────┘
                                v
                    ┌─────────────────────────┐
                    │ Reference Evidence      │
                    │ structured/title/OCR   │
                    └───────────┬─────────────┘
                                |
               ┌────────────────┴────────────────┐
               v                                 v
      ┌───────────────────┐             ┌────────────────────┐
      │ Exact Ref Lookup  │             │ Fallback Blocking  │
      │ hash/index        │             │ lexical + ANN      │
      └─────────┬─────────┘             └──────────┬─────────┘
                └────────────────┬─────────────────┘
                                 v
                    ┌─────────────────────────┐
                    │ Strict Evidence Verifier│
                    │ conflict veto + abstain │
                    └───────────┬─────────────┘
                                v
              ┌─────────────────┼──────────────────┐
              v                 v                  v
          AUTO_LINK           REJECT            ABSTAIN
              |                                    |
              v                                    v
     Canonical Entity Store                 Human Review Queue
```

---

# 5. 第一级：Reference Exact Lookup 应该吃掉大部分“容易样本”

如果某条 listing 已经有可靠的结构化 reference，就没有必要先跑向量检索。

推荐建立：

```text
reference_dictionary[
    canonical_brand_id,
    canonical_reference
] -> reference_entity_id
```

流程：

```text
raw reference
 -> Unicode NFKC
 -> uppercase
 -> brand-specific punctuation normalization
 -> preserve meaningful prefix/suffix
 -> canonical_reference
 -> exact lookup
```

### 特别注意：不要“过度 normalize”

错误做法：

```text
把所有标点全部删掉
把所有 0 前缀去掉
把 O 和 0 自动互换
把 I/1/L 自动互换
把 LN/LV 之类后缀忽略
```

这些操作很可能把不同 reference 合并。

正确做法是保留：

```text
raw_value
normalized_value
normalization_rule_version
brand_id
source_field
confidence
```

任何模糊纠错都应该产生“候选”，而不是直接覆盖 canonical value。

---

# 6. 第二级：Reference Evidence Extractor

对 reference 不在结构化字段中的商品，需要从多个证据通道提取。

建议至少四路：

```text
E1: structured_reference
E2: title_reference
E3: description/reference-table text
E4: image OCR reference
```

每个证据记录：

```python
ReferenceEvidence(
    listing_id,
    source,
    channel,
    raw_value,
    normalized_value,
    brand_id,
    role,          # own_reference / compatible_reference / sku / unknown
    confidence,
    extractor_version,
)
```

这里一定要结合之前 `parts-distributor-sku-classifier` 调研里的编号角色思想：

> **先判断“这个编号是什么”，再判断“这个编号是多少”。**

例如：

```text
“适配劳力士 126610LN 的表带”
```

抽到 `126610LN` 并不代表当前商品本身是 126610LN。

所以要有：

```text
product_type = strap/accessory
role(reference) = compatible_reference
```

最终直接 veto 自动链接到整表实体。

---

# 7. 第三级：DeepBlocker 风格的 Fallback Blocking

只有这些 listing 进入 fallback：

```text
- 没抽到高置信 reference；
- reference 有多个候选；
- 字段冲突；
- reference 不在 canonical catalog；
- 新品牌/新系列；
- OCR/标题证据不完整。
```

这能避免 1000 万条记录全部做 ANN。

## 7.1 新的 Tuple Embedding 不应是单一 `_merged_text`

推荐实现：

```python
class ReferenceAwareTupleEmbedding:
    def encode(self, listing):
        return {
            "semantic_vec": encode_title_series_material(listing),
            "ref_char_vec": encode_reference_char_ngrams(listing),
            "brand_id": listing.canonical_brand_id,
            "product_type": listing.product_type,
        }
```

实践中甚至不必强行拼成一个向量，可以做多通道检索：

```text
Channel A: canonical reference exact index
Channel B: reference character n-gram / edit candidate index
Channel C: title/series semantic ANN
Channel D: image embedding ANN（只辅助）
```

最后做 candidate union，再进入严格验证。

这种设计比“所有字段压成一个 150d 向量”更适合 identifier-centric 任务。

---

## 7.2 Reference lexical index 比 semantic ANN 更优先

对于：

```text
126610LN
126610-LN
126610 LN
M126610LN-0001
```

字符级、brand-specific 规则和 lexical retrieval 通常比自然语言 embedding 更直接。

建议候选优先级：

```text
P0 exact canonical reference
P1 reference alias / punctuation-equivalent
P2 char n-gram / constrained edit candidate
P3 semantic ANN fallback
P4 visual ANN fallback
```

Semantic/visual 越靠后，权限越低。

---

## 7.3 新 Vector Pairing：BrandShardedANNPairing

接口可以沿用 DeepBlocker：

```python
class BrandShardedANNPairing:
    def index(self, reference_entities): ...
    def query(self, listing_embeddings, top_k=10): ...
```

内部：

```text
brand_id -> ANN index
```

若品牌高置信：

```text
只查一个 brand shard
```

若品牌不确定：

```text
查 top-2 brand shard
最终禁止仅凭 ANN 自动 link
```

这样同时获得：

- 更小索引；
- 更低 latency；
- 更少跨品牌相似型号误召回；
- 更容易按品牌独立调参、重建和回滚。

---

# 8. 最终决策必须是 Strict Evidence Gate，而不是 embedding threshold

建议三态：

```text
AUTO_LINK
REJECT
ABSTAIN
```

不要强迫每条数据二分类。

一个可直接落地的保守规则：

```python
def decide(listing, candidate_ref):
    evidences = high_conf_reference_evidences(listing)

    if trusted_brand_conflicts(listing, candidate_ref):
        return REJECT

    if has_high_conf_conflicting_references(evidences):
        return ABSTAIN

    own_refs = [e for e in evidences if e.role == "own_reference"]

    if not own_refs:
        return ABSTAIN

    if any(e.normalized_value != candidate_ref.reference for e in own_refs):
        return REJECT

    if two_independent_channels_agree(own_refs, candidate_ref.reference):
        return AUTO_LINK

    if one_whitelisted_high_reliability_field_agrees(own_refs, candidate_ref.reference):
        return AUTO_LINK

    return ABSTAIN
```

### 为什么要求“独立证据”

例如：

```text
结构化 reference = 126610LN
标题抽取 reference = 126610LN
```

如果两个值来自同一个原始字段的复制，就不能算两路独立证据。

真正独立可以是：

```text
结构化字段 + 标题
标题 + OCR
结构化字段 + OCR
```

并且必须先做来源/字段可靠性审计。

### 单证据何时允许 AUTO_LINK

只允许对明确白名单字段：

```text
(source, field, brand) -> reliability policy
```

例如某来源的品牌官方 reference 结构化字段经人工抽查长期稳定，才可以进入单证据自动放行；否则默认 abstain。

几百条黄金标签不足以“统计证明 99.99% precision”，所以不能把小样本校准结果包装成绝对保证。正确做法是：

- 规则本身极端保守；
- hard-negative 定向测试；
- 对单字段建立白名单；
- 持续人工抽检；
- 发现一次系统性误合并即可降级该规则。

---

# 9. 图片如何使用：只做辅助证据，不越权

需求明确有图片，因此图片应该用，但权限要受控。

## 9.1 OCR 优先于视觉相似

最有价值的图片通常是：

- 表背刻字；
- 保卡；
- 吊牌；
- 证书；
- 型号标签；
- 表盘局部文字。

OCR 如果识别出 reference，可以进入 `ReferenceEvidence`。

但 OCR 也会混淆：

```text
0/O
1/I/L
5/S
8/B
```

所以 OCR 模糊纠错只能生成候选，不应直接改成 canonical reference。

## 9.2 Visual Embedding 的权限

视觉 embedding 可以：

- 在没有 reference 时找“可能是哪一系列”；
- 给人工复核排序；
- 对文本候选做冲突提示；
- 召回 OCR 可能漏掉的候选 reference。

视觉 embedding 不可以：

```text
“长得很像” => AUTO_LINK
```

因为同系列不同 reference 往往视觉高度相似。

---

# 10. 千万级规模下的实际计算策略

设：

```text
N = listing 数量
R = canonical reference entity 数量
K = fallback 每条候选数
f = 进入 fallback 的 listing 比例
```

不要构造：

```text
N × N
```

甚至也不要默认构造：

```text
N × R
```

推荐成本结构：

```text
高置信 reference listing：O(1) / O(log R) exact lookup
疑难 listing：f × N × ANN(top-K)
最终验证：约 f × N × K
```

例如仅作为容量估算，若：

```text
N = 10,000,000
f = 10%
K = 10
```

则疑难候选边约：

```text
10,000,000 × 10% × 10 = 10,000,000 pairs
```

这比任意全量 pairwise 都小很多，而且 `f` 应通过 Reference Extractor 持续压低。

### 关键点

系统性能优化的第一优先级不是把 ANN 做得多快，而是：

> **尽可能多地让 listing 通过可靠 reference 直接命中 canonical entity，根本不进入 ANN。**

---

# 11. 增量更新架构

DeepBlocker 原始流程是 batch 训练 + batch blocking，当前需求需要改成版本化增量服务。

推荐拆成：

```text
1. Ingestion
2. Normalize
3. Extract Evidence
4. Exact Resolve
5. Fallback Candidate Retrieval
6. Verify
7. Persist Decision
8. Audit / Review feedback
```

每条 listing 存：

```text
source_record_version
normalizer_version
extractor_version
index_version
verifier_version
decision_version
```

这样规则更新后可以只重跑受影响的数据。

### 幂等键

建议：

```text
(source, source_listing_id, source_updated_at/content_hash)
```

避免重复抓取造成重复实体。

### Canonical Reference 变化

新增 reference：

```text
写数据库
 -> 更新 exact dictionary
 -> 增量插入对应 brand ANN shard
```

品牌规则变化：

```text
只重建该 brand 的 normalization/index
```

不用全局重跑。

---

# 12. 推荐数据模型

## 12.1 `canonical_reference`

```text
reference_entity_id
canonical_brand_id
canonical_reference
reference_aliases
series
product_type
status
version
```

唯一约束：

```text
UNIQUE(canonical_brand_id, canonical_reference)
```

## 12.2 `normalized_listing`

```text
listing_id
source
source_listing_id
canonical_brand_id
product_type
normalized_title
platform_sku
raw_payload_hash
normalizer_version
```

## 12.3 `reference_evidence`

```text
listing_id
channel
role
raw_value
normalized_value
confidence
extractor_version
```

## 12.4 `candidate_edge`

```text
listing_id
reference_entity_id
candidate_source      # exact / lexical / semantic_ann / visual_ann
rank
score
index_version
```

## 12.5 `match_decision`

```text
listing_id
reference_entity_id
decision              # AUTO_LINK / REJECT / ABSTAIN
reason_codes
verifier_version
created_at
```

`reason_codes` 必须可审计，例如：

```text
REF_EXACT_STRUCTURED
REF_TITLE_AGREE
REF_OCR_AGREE
CONFLICTING_REF
ACCESSORY_COMPATIBLE_REF
BRAND_CONFLICT
NO_REFERENCE_EVIDENCE
ANN_ONLY_NOT_ALLOWED
```

---

# 13. 可以直接实现的 `DeepBlocker` 风格接口

建议保留它的模块化思想：

```python
class CandidateGenerator:
    def retrieve(self, listing) -> list[Candidate]:
        ...

class ExactReferenceGenerator(CandidateGenerator):
    ...

class LexicalReferenceGenerator(CandidateGenerator):
    ...

class SemanticANNGenerator(CandidateGenerator):
    ...

class VisualANNGenerator(CandidateGenerator):
    ...

class CandidateUnion:
    def retrieve(self, listing):
        candidates = []
        for generator in self.generators:
            candidates.extend(generator.retrieve(listing))
        return dedupe_and_rank(candidates)

class StrictReferenceVerifier:
    def decide(self, listing, candidates):
        ...
```

这比把所有逻辑塞进一个 LLM prompt 或一个二分类模型更安全，因为：

- 每层可以单独回放；
- 每层有独立指标；
- 任何模型异常都可以降级；
- Exact reference 路径永远可以保留；
- 自动合并规则可被审计。

---

# 14. 黄金标签应该怎么用

用户可接受几百对人工标注，这些标签不应随机浪费在“明显正例/明显负例”上。

优先构造 hard set：

```text
40% 同品牌同系列邻近 reference
20% 标题有兼容型号/配件
15% 平台 SKU 与 reference 混淆
10% reference 格式差异
10% OCR 混淆
5% 新品牌/新来源异常
```

另外准备少量正常样本作为 sanity check。

### Blocking 标签用途

评估：

```text
candidate recall@K
```

目标是尽量保证正确 reference 进入候选集。

### Verifier 标签用途

重点评估：

```text
AUTO_LINK false positive = 0（在当前测试集内）
```

并单独统计：

```text
ABSTAIN rate
```

不要为了降低 abstain 强行调低阈值。

### 主动标注

后续新增标注应优先抽：

- verifier 最接近 AUTO_LINK 边界的样本；
- 新品牌；
- 新来源；
- 新 normalization rule 命中的样本；
- 模型/规则发生分歧的样本。

---

# 15. 生产监控必须按 source / brand 分桶

总体 precision 很容易掩盖局部灾难。

至少监控：

```text
AUTO_LINK count
ABSTAIN rate
manual overturn rate
reference conflict rate
structured/title/OCR agreement rate
unknown reference rate
candidate recall on sampled labels
```

分组：

```text
source
brand
product_type
extractor_version
verifier_version
```

例如某平台字段语义发生变化，把 `reference` 从官方型号改成内部商品号，总体指标可能短时间看不出，但该 source 的 conflict rate 会突然上升，应立即关闭对应白名单规则。

---

# 16. 故障与降级策略

precision-first 系统最重要的是“坏了以后宁可少匹配”。

## ANN 服务不可用

```text
Exact Reference 路径继续工作；
疑难样本全部 ABSTAIN，进入待处理队列。
```

## OCR/视觉模型不可用

```text
结构化/标题证据继续；
不能补足证据的样本 ABSTAIN。
```

## 新 extractor 版本异常

```text
回滚 extractor_version；
重放受影响 listing；
旧 decision 保留审计记录。
```

## 品牌规则冲突

```text
禁止 AUTO_LINK；
按 brand 降级；
不影响其他品牌。
```

这比一个端到端模型崩坏后整体不可解释要安全得多。

---

# 17. DeepBlocker 哪些部分可以直接借，哪些必须丢

## 可以借

### 1. `Tuple Embedding` / `Vector Pairing` 解耦

这是最有价值的设计。

### 2. Self-supervised Blocking

没有大规模标签时，用无监督/自监督学习候选表示是合理的，但要换成领域 hard negatives。

### 3. Top-K Candidate Set 思想

下游只验证小候选集。

### 4. Blocking 单独评估 recall + reduction

不要把 Blocking 和最终 Matching 指标混为一谈。

### 5. 模块可替换

Exact、lexical、ANN、visual 可以独立升级。

## 必须丢/重写

### 1. `_merged_text`

改成 typed field/evidence schema。

### 2. FastText 直接承担 reference 判别

只能做语义辅助。

### 3. Random negative

改成同品牌同系列 hard negative。

### 4. `ExactTopKVectorPairing`

改成分片 ANN / lexical index。

### 5. DataFrame row index 作为 ID

改成不可变业务 ID 映射。

### 6. 每批重新训练全部数据

改成版本化模型 + 持久索引 + 增量更新。

### 7. embedding 相似度直接变成 match

明确禁止；最终必须 Reference Evidence Gate。

---

# 18. 最小可落地版本（MVP）

建议不是先上深度模型，而是按风险从低到高落地。

## Phase A：Canonical Reference + Exact Resolver

先实现：

```text
brand normalization
reference normalization
identifier role
canonical reference table
exact lookup
strict conflict rules
```

这层通常会产生最高精度、最低成本的一批结果。

## Phase B：Lexical Fallback

加入：

```text
reference char n-gram
brand shard
constrained edit candidates
alias dictionary
```

仍然不允许 fuzzy candidate 自动 link，必须经过 evidence verifier。

## Phase C：DeepBlocker 风格 ANN

加入：

```text
ReferenceAwareTupleEmbedding
BrandShardedANNPairing
Top-K union
candidate recall benchmark
```

目标是解决 reference 缺失、标题脏、字段稀疏。

## Phase D：OCR / Visual

最后加入图片，优先 OCR reference，其次视觉召回。

## Phase E：Selective/Confidence Gate

如果规则之外还要上分类模型，应只做：

```text
REJECT / ABSTAIN 辅助
```

或在已经满足 reference 基本一致的样本中进一步过滤风险；不能让模型绕过 reference 规则放行。

---

# 19. 对当前 Spec 的最终推荐方案

如果现在就要做一个能真正上线的版本，我建议：

```text
                    Canonical Reference Catalog
                              ^
                              |
Raw Listing -> Normalizer -> Evidence Extractor
                              |
            +-----------------+------------------+
            |                                    |
   exact reference found                 uncertain / missing
            |                                    |
     Exact Ref Lookup                   Deep Blocking
            |                         lexical + ANN
            +-----------------+------------------+
                              |
                       Evidence Verifier
                              |
              +---------------+---------------+
              |               |               |
          AUTO_LINK         REJECT          ABSTAIN
```

其中 DeepBlocker 的位置非常明确：

> **它是 `uncertain / missing` 分支中的 Candidate Generator，而不是系统中心。**

这能同时满足：

- **100 万–1000 万级**：避免全量笛卡尔积，Exact + shard + ANN；
- **持续增量**：Reference Catalog 与 ANN shard 可增量更新；
- **字段稀疏**：标题/描述/OCR/视觉多路候选；
- **Reference 埋在标题**：Evidence Extractor 专门处理；
- **有图片**：OCR/视觉仅作为辅助；
- **几百标注**：重点用于 hard-negative gate 与分层评测；
- **绝不能误匹配**：最终由 Reference Evidence + conflict veto + ABSTAIN 收口。

---

# 20. 最终结论

DeepBlocker 对这个项目最重要的价值不是“深度学习”三个字，而是把大规模实体匹配中最昂贵的问题拆成了：

```text
Representation
    + Retrieval
```

并用 Top-K Candidate Set 避免全量比较。

但当前腕表需求比通用 Entity Matching 更严格，因为业务真值已经被明确规定为 reference number。基于这个定义，系统应进一步拆成：

```text
Candidate Retrieval = 可以模糊、可以机器学习、可以 ANN
Final Decision       = 必须严格、可解释、可拒识、reference-first
```

因此推荐采用“**DeepBlocker-inspired Blocking + Canonical Reference Linking + Strict Evidence Gate**”三层架构：

1. DeepBlocker 思想解决规模和候选召回；
2. Canonical Reference Entity 消除三源 pairwise 组合爆炸；
3. Strict Evidence Gate 把 precision 风险牢牢锁在最终自动合并之前。

如果只能保留一句落地原则：

> **让 embedding 负责找候选，让 reference 负责定生死；任何证据冲突或仅靠相似度成立的候选，一律 ABSTAIN。**
