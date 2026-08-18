# De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search：把 MM-LTP 改造成腕表 Reference 归属验证与噪声否决层

- 分析人：b
- 调研文章：De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search
- 作者：Zhizhang Hu, Shasha Li, Ming Du, Arnab Dhua, Douglas Gray
- 会议：CVPR 2024 Workshop on Multimodal Learning and Applications
- CVF：https://openaccess.thecvf.com/content/CVPR2024W/MULA/html/Hu_De-noised_Vision-language_Fusion_Guided_by_Visual_Cues_for_E-commerce_Product_CVPRW_2024_paper.html
- 作者论文页：https://kevin-hu.com/publication/mula_2024/
- Amazon Science：https://www.amazon.science/publications/de-noised-vision-language-fusion-guided-by-visual-cues-for-e-commerce-product-search
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31

## 1. 为什么本轮选择这篇文章

当前 Spec 的业务约束是：

1. 雷小安、腕表之家、奢当家三源数据；
2. 100 万～1000 万级，持续增量；
3. “同一个商品”严格定义为 **同一 reference number / 型号**；
4. reference 可能在结构化字段，也可能埋在标题、描述甚至图片里；
5. 图片可用；
6. **绝对不能误匹配，precision 优先到极致，允许漏匹配**；
7. 可以人工标注几百对黄金标签。

在开始本轮分析前，我先检查了 `奢侈品调研/b/` 已有结果。此前已经分析过 AnyMatch、DeepBlocker、Ditto、GraLMatch、TransClean、pyJedAI、Confidence Classifier、Conformal Selective Prediction、MOON2.0、End-to-end multi-modal product matching、属性抽取/规范化、SKU/MPN 分类等方向，因此这些条目本轮全部排除。

本篇 **De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search** 尚未出现在 `b` 目录中，而且它与本项目一个非常危险但容易被忽略的问题高度相关：

> 标题中出现一个“看起来像 reference 的字符串”，并不意味着它就是当前售卖商品自己的 reference。

典型例子：

```text
劳力士表带 适配 116500LN / 126500LN
欧米茄盒证 适用于 310.30.42.50.01.001
某平台标题把库存号、商家 SKU、序列号、reference 混写
卖家为了 SEO 在标题里塞多个相邻型号
```

如果系统采取：

```text
标题抽 reference -> canonicalize -> exact join
```

就可能出现极其严重的 false positive。

原论文 MM-LTP 的核心不是“做商品匹配”，而是：

> 使用视觉线索学习哪些文本 token 真正与当前商品图像有关，并在多模态训练过程中在线抑制/剪除噪声 token。

这给当前项目提供了一个很适合的迁移方向：

> **不要把图片用于“长得像所以判同款”，而要把图片和上下文用于判断“这个 reference 候选到底是不是当前商品自己的 reference”。**

这与 precision-first 的目标比直接做视觉相似度匹配更一致。

---

## 2. 原论文解决的问题

电商多模态模型通常使用：

```text
商品图片 <-> 商品标题/描述
```

作为训练对。

问题是电商标题并不是干净的视觉描述。卖家会放入：

- 功能属性；
- SEO 关键词；
- 品牌宣传；
- 兼容范围；
- 非视觉规格；
- 与商品本体无关的附属信息。

因此一条标题可能有很多 token，但真正能被图像支持的只有一部分。

原论文举的核心问题可以抽象为：

```text
image = 当前商品视觉内容
text  = 视觉内容 + 非视觉信息 + 噪声
```

如果直接把完整 text 与 image 强行对齐，模型会被噪声监督污染。

论文提出：

**MM-LTP = MultiModal alignment-guided Learned Token Pruning**

它把原本用于降低 Transformer 计算量的 token pruning，改造成 **在线文本清洗机制**：让模型自己学出每层的 token 重要性阈值，弱化与图像对齐差的文本 token。

论文在基于 Amazon ESCI 扩展出的多模态电商数据上评测，覆盖约 71.7 万商品，训练集 637,511 商品、测试集 80,000 商品，总计约 104 万对数据。

最值得关注的结论不是整体 Recall，而是：

- MM-LTP 能同时接入 CLIP 这种隐式融合模型和 ALBEF 这种显式 cross-attention 模型；
- 在 ALBEF 中同时对 self-attention 和 cross-attention 做 pruning 时，R@1 从 51.68 提升到 57.06，提升 5.38 个百分点；
- 说明对电商脏文本先做视觉约束的去噪，确实能显著改善多模态表示。

---

## 3. MM-LTP 技术实现拆解

## 3.1 两类多模态架构都可接入

论文考虑两种常见结构。

### A. CLIP 类：隐式融合

```text
image -> image encoder -> image embedding
text  -> text encoder  -> text embedding

image embedding <-> text embedding
            contrastive loss
```

这里图文没有 cross-attention，所以 MM-LTP 从 text encoder 的 **self-attention** 中计算 token 重要性。

### B. ALBEF 类：显式融合

```text
image -> image encoder -------------------\
                                          cross-attention / fusion encoder
text  -> text encoder --------------------/
```

这里可以直接利用文本 token 对 image token 的 cross-attention 判断 token 的视觉相关性。

论文的工程价值在于：MM-LTP 不是绑定某个 backbone，而是对 attention matrix 做一个通用的 learned mask。

---

## 3.2 Token Importance：从 attention 中计算每个 token 的重要性

设 Query token 为：

```text
x = [x1, x2, ..., xm]
```

Key token 为：

```text
z = [z1, z2, ..., zk]
```

注意力矩阵为：

```text
Attn(x, z) = x Wq Wk^T z^T / sqrt(d)
```

最直接的做法是把一个 Query token 对所有 Key token、所有 head 的 attention 求平均：

```text
s(x_i) = average_h average_j Attn(x_i, z_j)
```

但论文认为所有 Key 平均会引入噪声，因此使用 **Key 侧 [CLS] token** 来汇总整体上下文。

最终 token importance 近似为：

```text
s(x_i) = (1 / H) * sum_h Attn_h(x_i, z_cls)
```

对于 cross-attention：

```text
z_cls = image [CLS]
```

对于 self-attention：

```text
z_cls = text [CLS]
```

论文的消融实验显示，这种“Key CLS + Query 非 CLS token”的计算方式优于对全部 Key/Query 做简单平均。

### 对腕表项目的启发

可以把标题：

```text
劳力士 原装橡胶表带 适配 Daytona 116500LN 黑色 20mm
```

拆成 token，并让视觉/上下文模型判断：

```text
劳力士         -> 商品上下文
原装           -> 弱视觉/营销词
橡胶表带       -> 强视觉：当前商品是表带
适配           -> 强关系词：后面的型号可能是 compatibility target
Daytona        -> 被适配对象
116500LN       -> reference candidate，但未必属于当前商品本体
黑色           -> 视觉属性
20mm           -> 视觉/尺寸属性
```

真正重要的不是“116500LN 是否能从图片中看出来”，而是：

> `116500LN` 在这个文本结构里扮演的是 **当前商品 reference**，还是 **被适配对象 reference**。

这是本项目应该比原论文多做的一层。

---

## 3.3 Learned Threshold：不同层自动学习不同剪枝阈值

论文没有人工固定 threshold，而是给每个 Transformer layer 一个可学习参数：

```text
τ_l
```

原因是浅层和深层 token importance 分布不同，且不同任务的数据噪声程度不同。

对第 `l` 层第 `i` 个 token：

```text
M_l(x_i) = sigmoid((s_l(x_i) - τ_l) / T)
```

其中：

```text
s_l(x_i) = token importance
τ_l      = learnable threshold
T        = temperature
```

当 `T` 很小时，该 sigmoid 接近二值 mask：

```text
s > τ -> M ≈ 1 -> 保留
s < τ -> M ≈ 0 -> 抑制
```

论文实验使用：

```text
T = 1e-4
```

因此它虽然在训练时可微，但行为已经非常接近 hard pruning。

---

## 3.4 Pruning Loss：让模型真的愿意丢掉噪声 token

如果只有 task loss，模型未必有动力剪 token。

因此论文增加 L1 pruning regularization：

```text
L_prune = average_layer ( ||M_l(x)||_1 / query_length_l )
```

总目标：

```text
L = L_model + λ * L_prune
```

论文实验：

```text
λ = 0.1
```

直观解释：

- `L_model` 要求模型把有用信息保住；
- `L_prune` 鼓励尽量少保留 token；
- 两者竞争后，剩下的是对当前 multimodal task 最重要的文本信息。

这本质上是一种 **可学习的信息瓶颈**。

---

## 3.5 原论文训练规模

论文实现信息：

```text
框架：PyTorch + Ray
GPU：8 × NVIDIA A100
vision encoder：ViT-B/16，12 层，约 86M 参数
CLIP text encoder：12 层，约 63M 参数
ALBEF text/fusion encoder：6 层，总模型约 124M 参数
```

CLIP：

```text
epoch > 100
batch size = 1360
optimizer = AdamW
weight decay = 0.02
learning rate：5e-6 -> warmup 2e-5 -> cosine decay 5e-6
```

MM-LTP：

```text
每层 threshold 线性初始化
最后一层 threshold = 0.01
T = 1e-4
λ = 0.1
```

这说明直接完整复现并不适合作为当前项目 MVP。

我们应该迁移 **机制**，而不是照搬训练规模。

---

## 4. 这篇论文不能直接照搬到腕表 Reference Matching

这是本次分析最重要的部分。

如果直接把 MM-LTP 理解成：

```text
和图片对不上 -> 删除文本 token
```

反而会伤害 reference matching。

### 4.1 Reference 经常天然“不可视觉化”

很多腕表主图中：

```text
116500LN
126610LV
IW500705
310.30.42.50.01.001
```

根本没有可读文本。

reference 可能只存在于：

- 标题；
- 保卡；
- 吊牌；
- 表背小字；
- 商品属性字段。

因此主图与 reference token 的视觉 attention 很低，并不能证明 reference 是噪声。

### 4.2 外观相似和同 reference 不是同一件事

腕表同系列相邻 reference 往往外观极其接近。

如果把视觉相似度用于最终 match：

```text
相似外观 -> 同款
```

会直接违反 Spec。

例如：

```text
同系列不同表径
同系列不同材质
同系列不同盘面
前代/后代 reference
同壳型不同机芯
```

视觉都可能高度相似。

### 4.3 论文优化的是 retrieval recall，不是 false-positive-zero

原论文评测指标是 Recall@K。

当前项目真正应该优化的是：

```text
Auto-Match Precision
False Positive per Million
Abstention / Review Coverage
```

所以 MM-LTP 只能作为局部组件，而不能作为最终 matching model。

---

# 5. 推荐的迁移方式：Reference Attribution Gate（RAGate）

建议不要把 MM-LTP 复制成一个“大一统商品 matcher”，而是改造成一个更窄的模块：

> **Reference Attribution Gate：判断一个 reference candidate 是否属于“当前售卖主体商品”。**

系统职责拆成：

```text
Reference Extraction
        ↓
Reference Role Classification
        ↓
Reference Attribution Gate   <- MM-LTP 思路主要落在这里
        ↓
Canonicalization
        ↓
Exact Reference Join
        ↓
Conflict Gate / Abstention
        ↓
Auto Match
```

最终是否匹配，仍然只由 canonical reference 的硬证据决定。

---

## 6. 端到端落地架构

```text
                   ┌──────────────────────────────┐
                   │ 雷小安 / 腕表之家 / 奢当家    │
                   └──────────────┬───────────────┘
                                  │
                                  v
                    ┌─────────────────────────┐
                    │ Raw Ingestion / Snapshot│
                    │ 保留原文、原图、来源时间 │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┴──────────────────┐
              │                                     │
              v                                     v
   ┌──────────────────────┐              ┌──────────────────────┐
   │ Text Reference Extract│              │ Image OCR / Crop OCR │
   │ field/title/desc      │              │ 表背/保卡/吊牌/标签   │
   └───────────┬──────────┘              └───────────┬──────────┘
               │                                     │
               └──────────────────┬──────────────────┘
                                  v
                   ┌──────────────────────────────┐
                   │ Reference Evidence Store      │
                   │ raw value + span + provenance │
                   └──────────────┬───────────────┘
                                  │
                                  v
                   ┌──────────────────────────────┐
                   │ Role Classifier              │
                   │ BRAND_REF / SKU / SERIAL /   │
                   │ COMPATIBLE_REF / OTHER       │
                   └──────────────┬───────────────┘
                                  │
                                  v
                   ┌──────────────────────────────┐
                   │ RAGate                       │
                   │ MM-LTP-inspired ownership    │
                   │ text context + visual support│
                   └──────────────┬───────────────┘
                                  │
                                  v
                   ┌──────────────────────────────┐
                   │ Brand-specific Canonicalizer │
                   │ raw -> canonical reference   │
                   └──────────────┬───────────────┘
                                  │
                                  v
                   ┌──────────────────────────────┐
                   │ Exact Reference Index        │
                   │ key=(brand_id, canonical_ref)│
                   └──────────────┬───────────────┘
                                  │
                                  v
                   ┌──────────────────────────────┐
                   │ Precision-first Policy Engine│
                   │ ACCEPT / REJECT / REVIEW     │
                   └──────────────┬───────────────┘
                                  │
                  ┌───────────────┼────────────────┐
                  v               v                v
             Auto Match        Reject         Human Review
```

---

# 7. 第一层：Reference Extraction 必须“多候选 + 保留 provenance”

不要只存一个：

```text
reference = "116500LN"
```

必须存成 evidence list：

```json
[
  {
    "raw": "116500LN",
    "normalized": "116500LN",
    "source_type": "structured_field",
    "field": "reference_no",
    "span": null,
    "extractor": "source_adapter_v3",
    "extract_confidence": 1.0
  },
  {
    "raw": "116500LN",
    "normalized": "116500LN",
    "source_type": "title",
    "field": "title",
    "span": [17, 25],
    "context": "橡胶表带适配 Daytona 116500LN",
    "extractor": "regex_rolex_v5",
    "extract_confidence": 0.92
  }
]
```

关键是：

> 同一个字符串，从结构化 reference 字段抽到，和从“适配 xxx”标题里抽到，证据等级完全不同。

---

# 8. 第二层：编号角色分类，先解决“它是什么编号”

建议所有候选先分类：

```text
BRAND_REFERENCE        品牌官方 reference / model number
PLATFORM_SKU           平台货号
SELLER_SKU             商家 SKU
SERIAL_NUMBER          独立序列号
COMPATIBLE_REFERENCE   配件适配的目标商品 reference
ORDER_OR_INVENTORY_ID  库存/订单编号
UNKNOWN_IDENTIFIER     未知编号
```

### 强烈建议使用规则优先

例如：

```text
"适配 116500LN"
"for 116500LN"
"compatible with 116500LN"
"表带 116500LN"
```

其中 `116500LN` 默认优先标记：

```text
COMPATIBLE_REFERENCE
```

除非另外有结构化字段/保卡 OCR 等强证据证明当前商品本体就是该型号。

### 为什么这一层非常重要

如果 role 不先解决，后面的 exact match 反而会放大错误：

```text
配件 A 标题出现 116500LN
腕表 B reference = 116500LN

字符串 exact 相等
=> 错误合并
```

因此本项目的真正风险并不是字符串相似度，而是 **reference ownership / role ambiguity**。

---

# 9. 第三层：把 MM-LTP 改成 Reference Attribution Gate

## 9.1 输入

针对每个 reference candidate 构造：

```text
文本输入：
  brand
  title
  description
  structured attrs
  candidate 前后窗口
  special marker: <REF>116500LN</REF>

图片输入：
  主图
  表盘图
  表背图
  保卡图
  吊牌图
  OCR crop
```

不要把所有图同权。

建议 image_type：

```text
MAIN_PRODUCT
DIAL
CASEBACK
WARRANTY_CARD
HANG_TAG
BOX
ACCESSORY
UNKNOWN
```

对 reference 归属而言：

```text
WARRANTY_CARD / HANG_TAG / CASEBACK OCR
```

通常比纯主图视觉 embedding 更有证据价值。

---

## 9.2 与原 MM-LTP 不同：Reference token 必须设为 protected token

原论文会把视觉弱相关 token 剪掉。

本项目不能这样做。

建议：

```text
candidate reference token = protected
```

即：

```text
M(reference_token) = 1
```

不允许仅凭 visual grounding 把 reference 自己删除。

MM-LTP 只用于清理它周围的上下文：

```text
营销词
SEO token
重复品牌词
无关功能词
被适配对象
其他型号
```

然后由清洗后的上下文判断 reference role。

这是一处关键改造。

---

## 9.3 Candidate Ownership Score

对每个 candidate `r` 计算：

```text
ownership_score(r)
```

建议融合以下特征：

```text
1. structured_field_trust
2. title_context_role_score
3. candidate_span_attention
4. image_text_alignment_score
5. OCR_exact_support
6. brand_reference_dictionary_hit
7. accessory_context_penalty
8. competing_reference_count
9. source_specific_field_quality
10. contradiction_score
```

可以先做 LightGBM / Logistic Regression，而不是直接训练大模型。

示意：

```text
p_owner = sigmoid(
    w1 * structured_ref
  + w2 * dictionary_hit
  + w3 * ocr_exact
  + w4 * title_primary_role
  + w5 * mm_ltp_context_score
  - w6 * accessory_context
  - w7 * conflicting_ref
  - w8 * platform_sku_pattern
)
```

重点：

> `p_owner` 只判断 candidate 是否属于当前商品，不直接判断两条商品是不是同款。

---

# 10. 第四层：Canonicalization 必须品牌化，禁止全局激进清洗

错误做法：

```python
ref = re.sub(r"[^A-Z0-9]", "", ref.upper())
```

这会让一些本来有意义的后缀、分段、版本标识被错误吞掉。

建议：

```text
global normalization
    ↓
brand-specific parser
    ↓
dictionary validation
    ↓
canonical reference
```

全局只做非常安全的变换：

```text
Unicode NFKC
trim
uppercase
统一全角/半角
统一显然等价的空白
```

其他规则按品牌配置。

例如：

```yaml
rolex:
  allow_remove_space: true
  allow_remove_dash: false
  preserve_suffix: true
  regex:
    - "^[0-9]{5,6}[A-Z]{0,3}$"

omega:
  allow_remove_space: true
  preserve_dot_groups: true

iwc:
  aliases:
    "IW 500705": "IW500705"
```

**canonicalizer 必须可版本化。**

数据库同时保留：

```text
raw_reference
canonical_reference
canonicalizer_version
```

未来规则升级可以重跑，不丢原始证据。

---

# 11. 第五层：千万级不要做全量 pairwise matching，直接 Exact Reference Index

既然业务定义已经明确：

```text
same product <=> same canonical reference
```

那最终主索引应该是：

```text
(brand_id, canonical_reference)
```

而不是 embedding ANN。

### PostgreSQL 示例

```sql
CREATE TABLE product_reference (
    product_id           BIGINT NOT NULL,
    source               SMALLINT NOT NULL,
    brand_id             BIGINT NOT NULL,
    raw_reference        TEXT NOT NULL,
    canonical_reference  TEXT NOT NULL,
    role                 SMALLINT NOT NULL,
    ownership_score      DOUBLE PRECISION NOT NULL,
    evidence_level       SMALLINT NOT NULL,
    conflict_flag        BOOLEAN NOT NULL DEFAULT FALSE,
    canonicalizer_ver    INTEGER NOT NULL,
    extractor_ver        INTEGER NOT NULL,
    created_at           TIMESTAMPTZ NOT NULL,
    PRIMARY KEY(product_id, canonical_reference)
);

CREATE INDEX idx_ref_exact
ON product_reference(brand_id, canonical_reference)
WHERE role = 1 AND conflict_flag = FALSE;
```

1000 万级 exact key lookup 对 PostgreSQL/分片 KV 来说并不困难。

如果吞吐继续上升，可以把：

```text
brand_id + canonical_reference -> entity_id
```

放到 RocksDB / Redis / Dynamo 风格 KV 中。

但没有必要为了“AI 感”先引入向量数据库做最终判定。

### Vector Search 的正确位置

只允许用于：

```text
缺 reference 的疑难记录
候选召回
人工 review 排序
潜在 dictionary alias 发现
```

不能产生自动 merge。

---

# 12. Precision-first Policy Engine

建议三态，而不是二分类：

```text
ACCEPT
REJECT
REVIEW
```

## 12.1 ACCEPT

自动合并必须满足硬条件。

第一版建议：

```python
def auto_match(a, b):
    if a.brand_id != b.brand_id:
        return "REJECT"

    if a.conflict_flag or b.conflict_flag:
        return "REVIEW"

    if not a.canonical_ref or not b.canonical_ref:
        return "REVIEW"

    if a.canonical_ref != b.canonical_ref:
        return "REJECT"

    if a.role != "BRAND_REFERENCE" or b.role != "BRAND_REFERENCE":
        return "REVIEW"

    if a.evidence_level < HIGH or b.evidence_level < HIGH:
        return "REVIEW"

    if a.ownership_score < TA or b.ownership_score < TB:
        return "REVIEW"

    return "ACCEPT"
```

### ACCEPT 核心原则

不是：

```text
model_score > 0.99
```

而是：

```text
相同 canonical reference
+ reference role 正确
+ reference 属于商品本体
+ 无冲突证据
+ 两边证据强度达标
```

模型只是其中一项 gate。

---

## 12.2 REJECT

明确冲突直接拒绝：

```text
brand 不同
canonical reference 不同
一个是商品本体，一个是配件适配 target
强 OCR/结构化字段出现不同 reference
```

不要让视觉相似度覆盖这些 hard negative。

---

## 12.3 REVIEW

以下全部进入 REVIEW：

```text
只有一边有 reference
reference 只来自 description
同标题出现多个 reference
商品类别疑似 ACCESSORY
OCR 与标题冲突
平台字段历史质量差
canonicalization 使用新/未知规则
ownership score 边界
```

允许漏匹配，所以 REVIEW 可以很多。

---

# 13. 最重要的防错设计：Cluster Invariant

不要让 pairwise edge 通过传递关系污染整个实体簇。

对每个最终实体簇强制：

```text
cluster_key = (brand_id, canonical_reference)
```

集群 invariant：

```text
同一个 cluster 只能存在一个 canonical_reference
```

任何新记录加入前必须验证：

```text
record.canonical_ref == cluster.canonical_ref
```

如果：

```text
A --match--> B
B --match--> C
```

但 C 的强 reference 与 cluster key 不同：

```text
直接禁止入簇
```

这比纯图聚类更适合当前 Spec。

---

# 14. Evidence Level 设计

建议为 reference 建立固定证据等级。

## Level 5：确定性强证据

```text
可信 source 的独立 structured reference 字段
+ 格式校验通过
+ 品牌 dictionary 命中
+ 无冲突
```

## Level 4：双重独立证据

```text
标题 reference
+ 保卡/吊牌/表背 OCR exact 一致
```

或：

```text
标题 reference
+ source 属性 reference 一致
```

## Level 3：高置信文本证据

```text
标题唯一 candidate
+ brand parser 命中
+ role = BRAND_REFERENCE
+ RAGate ownership 高
```

## Level 2：弱文本/OCR

```text
description only
单次低质量 OCR
```

## Level 1：视觉/语义猜测

```text
图片看起来像某 reference
embedding nearest neighbor
LLM 根据外观猜型号
```

### 自动匹配建议

MVP：

```text
只允许 Level >= 4 自动合并
```

运行稳定后再用黄金集验证是否能放宽到部分 Level 3。

Level 1 永远不得自动合并。

---

# 15. RAGate 的训练方案：不需要几万人工标签

用户可接受几百对黄金标签，这足够启动，但标签应该集中在 **最危险的 hard cases**。

建议人工标注 300～600 组，结构不是简单：

```text
pair -> match / non-match
```

而是尽量多标一层：

```json
{
  "product_id": 123,
  "candidate": "116500LN",
  "candidate_role": "COMPATIBLE_REFERENCE",
  "belongs_to_primary_product": false,
  "evidence": ["title_context"],
  "hard_reason": "accessory_compatible_with"
}
```

### hard negative 必须优先

不要随机采负例。

重点采：

```text
同品牌相邻 reference
同系列不同后缀
配件标题带目标腕表 reference
盒证/表带/表扣
平台 SKU 长得像 reference
序列号长得像 reference
一个标题多个 reference
OCR 误读 0/O、1/I、5/S、8/B
相同外观不同 reference
旧款/新款相邻型号
```

这些样本才决定 production precision。

---

# 16. Weak Supervision：从现有数据自动造训练信号

几百人工标签外，还可以自动生成弱标签。

### Positive candidate ownership

高可信规则：

```text
结构化 reference 字段 = 标题唯一 candidate
且 brand parser 通过
且不是 accessory category
```

### Negative candidate ownership

```text
"适配/兼容/for/compatible with" 后的 candidate
已知平台 SKU 字段
库存号字段
一条记录里和强 structured reference 不一致的其他 candidate
```

然后用人工黄金集做校准和最终 threshold 选择。

这比直接让大模型自由学习省得多。

---

# 17. 如果真正实现 MM-LTP-style 模型，推荐的小型改造

不建议从零训练 ALBEF。

可以用已有视觉/文本 encoder，新增轻量 gate。

## 17.1 模型结构

```text
image(s) -> frozen/small vision encoder -> visual tokens

text -> tokenizer
     -> mark candidate span
     -> text transformer

visual tokens + text tokens
     -> 2~4 layer cross-attention adapter
     -> token importance
     -> learned threshold mask
     -> context embedding

candidate span embedding
+ context embedding
+ structured features
     -> MLP
     -> ownership probability
```

只训练：

```text
cross-attention adapter
threshold τ
small MLP
```

可以 LoRA/adapter 化，成本远小于原论文。

---

## 17.2 修改 pruning loss

原论文会整体鼓励少 token。

本项目建议只对非 protected token 做 pruning：

```text
L_prune_nonref
```

总 loss：

```text
L = L_role
  + α * L_owner
  + β * L_prune_nonref
  + γ * L_contradiction
```

其中：

```text
L_role          candidate 编号角色分类
L_owner         是否属于当前商品本体
L_prune_nonref  清理无关上下文
L_contradiction OCR/structured/text 强证据冲突
```

Reference token 本身固定保留。

---

# 18. 图片应该怎么用：只做 Support / Veto，不做 Override

这是整个架构的安全边界。

## 图片可以做

```text
1. OCR 抽取 reference
2. 识别当前商品是“腕表本体”还是“表带/盒/扣/配件”
3. 验证标题里的材质、颜色、盘面等是否明显冲突
4. 判断 candidate 前后的视觉属性是否支持当前商品
5. 为人工复核排序
```

## 图片不可以做

```text
长得像 116500LN -> 自动赋值 116500LN
视觉 embedding 高 -> 覆盖 reference 冲突
两个图很像 -> 自动判同 reference
```

最终 reference 必须来自可审计的文本/OCR/结构字段。

---

# 19. OCR 策略：优先“证据图”，不是所有图一锅端

建议先做 image classifier：

```text
main_watch
caseback
warranty_card
hang_tag
box
accessory
other
```

然后对：

```text
caseback
warranty_card
hang_tag
```

做高分辨率 OCR。

输出也必须 evidence 化：

```json
{
  "value": "126610LV",
  "image_id": "...",
  "image_type": "WARRANTY_CARD",
  "bbox": [100, 200, 320, 250],
  "ocr_confidence": 0.97,
  "parser": "rolex_ref_v3"
}
```

不要只存最后字符串。

---

# 20. 数据表建议

## raw_product

```text
product_id
source
source_product_id
raw_json
raw_title
raw_description
category
crawl_time
version
```

## reference_evidence

```text
evidence_id
product_id
raw_value
normalized_value
source_type
field_name
span_start
span_end
image_id
ocr_bbox
extractor
extractor_version
extract_confidence
```

## reference_candidate

```text
product_id
candidate
role
role_score
ownership_score
brand_id
canonical_reference
canonicalizer_version
evidence_level
conflict_flag
```

## reference_entity

```text
entity_id
brand_id
canonical_reference
created_at
status
```

唯一约束：

```sql
UNIQUE (brand_id, canonical_reference)
```

## product_entity_link

```text
product_id
entity_id
decision
policy_version
decision_reason
decision_time
```

## review_case

```text
case_id
product_id
candidate
reason
model_snapshot
human_decision
reviewer
time
```

所有自动决策都要能解释：

```text
为什么自动合并
使用了哪些 evidence
由哪个 parser/model/policy 版本做出
```

---

# 21. 增量处理架构

如果当前抓取系统已经有批次文件，MVP 不必先上 Kafka。

```text
crawler batch
  -> raw object storage
  -> extraction workers
  -> reference evidence table
  -> policy engine
  -> exact index
```

当持续增量频率上升后再演进：

```text
Crawler
  -> Kafka / Pulsar
  -> Extraction workers
  -> OCR workers
  -> Role/RAGate workers
  -> Canonicalizer
  -> Reference Index
  -> Match Event
```

### 幂等键

```text
(source, source_product_id, source_version)
```

### 决策版本

```text
extractor_version
canonicalizer_version
ragate_version
policy_version
```

每次升级都可以定向 replay。

---

# 22. 线上决策必须生成 Reason Code

示例：

```text
ACCEPT_SAME_REF_STRUCTURED_STRUCTURED
ACCEPT_SAME_REF_STRUCTURED_OCR
ACCEPT_SAME_REF_DUAL_EVIDENCE
REJECT_REF_CONFLICT
REJECT_BRAND_CONFLICT
REVIEW_MULTI_REF_TITLE
REVIEW_ACCESSORY_COMPATIBLE_REF
REVIEW_OCR_TEXT_CONFLICT
REVIEW_LOW_OWNER_SCORE
```

这对后续 precision 审计非常重要。

不能只留一个：

```text
score = 0.997
```

---

# 23. 评测体系：不要以 F1 为主

建议核心指标：

```text
Auto-Accept Precision
False Positives / 1M auto decisions
Auto-Accept Coverage
Review Rate
Reference Extraction Precision
Reference Ownership Precision
Conflict Detection Recall
```

### 黄金集必须分桶

```text
structured-structured
structured-title
title-title
text-OCR
accessory
multi-reference title
same-series adjacent refs
OCR confusion
new brand/new source
```

否则总体 precision 会被大量简单样本稀释。

---

# 24. “几百黄金标签”能做什么，不能做什么

几百对标签足够：

```text
启动 role classifier
找明显 hard negative pattern
选择第一版 threshold
验证系统方向
```

但几百条 **不足以统计证明 99.99% precision**。

一个简单经验是：零错误样本下，95% 置信水平的错误率上界大约是：

```text
3 / N
```

因此：

```text
N=300  -> 约 1% 量级上界
N=3000 -> 约 0.1%
N=30000 -> 约 0.01%
```

所以真正的“绝不能误匹配”不能靠一次性黄金集保证，而要靠：

```text
硬规则
+ abstain
+ conflict gate
+ 持续抽样审计
+ hard-negative 回归集
```

模型 precision 只是其中一部分。

---

# 25. 上线后的持续审计

建议每个策略桶持续抽样：

```text
每天/每批次：

Level 5 auto accept -> 少量随机抽查
Level 4 auto accept -> 更高比例抽查
边界 ownership score -> 强制 review
新品牌/新 source -> 初期全部 review
新 canonicalizer version -> shadow mode
```

检测到一个 false positive 后，不只是修这一条，而是生成：

```text
counterexample
reason code
新 hard-negative rule
回归测试
```

然后 replay 同类历史数据。

---

# 26. 推荐 MVP：两周级工程方向，不复现整篇论文

第一版直接落地以下组件即可。

## Phase 0：只做确定性 Reference Join

```text
structured reference
-> safe canonicalizer
-> (brand, canonical_ref) exact join
-> cluster invariant
```

先拿到最高 precision 的 coverage。

## Phase 1：Title Reference Extraction + Role Rules

加入：

```text
标题 regex
品牌字典
适配/兼容/配件规则
平台 SKU 规则
multi-ref conflict
```

只有高证据记录自动放行。

## Phase 2：OCR Evidence

只优先做：

```text
保卡
吊牌
表背
```

OCR exact 支持可以把部分弱 title evidence 提升到 Level 4。

## Phase 3：轻量 RAGate

先不上完整 MM-LTP。

训练：

```text
candidate role classifier
candidate ownership classifier
```

输入：

```text
candidate context
source fields
category
image type
CLIP/视觉 embedding features
OCR evidence
```

模型先用 LightGBM / 小 Transformer 都可。

## Phase 4：MM-LTP-style Adapter

只有当 Phase 3 的 hard cases 仍然大量来自标题噪声时，再做：

```text
cross-attention adapter
learnable threshold
protected reference span
non-ref token pruning loss
```

这能避免过度工程化。

---

# 27. 可直接实现的 Reference 决策代码框架

```python
from dataclasses import dataclass
from enum import Enum

class Decision(str, Enum):
    ACCEPT = "ACCEPT"
    REJECT = "REJECT"
    REVIEW = "REVIEW"

@dataclass
class RefState:
    brand_id: str | None
    canonical_ref: str | None
    role: str | None
    evidence_level: int
    ownership_score: float
    conflict: bool
    is_accessory: bool


def decide(a: RefState, b: RefState) -> tuple[Decision, str]:
    if a.brand_id and b.brand_id and a.brand_id != b.brand_id:
        return Decision.REJECT, "REJECT_BRAND_CONFLICT"

    if a.conflict or b.conflict:
        return Decision.REVIEW, "REVIEW_INTERNAL_REF_CONFLICT"

    if not a.canonical_ref or not b.canonical_ref:
        return Decision.REVIEW, "REVIEW_MISSING_REFERENCE"

    if a.canonical_ref != b.canonical_ref:
        return Decision.REJECT, "REJECT_REF_CONFLICT"

    if a.is_accessory or b.is_accessory:
        return Decision.REVIEW, "REVIEW_ACCESSORY"

    if a.role != "BRAND_REFERENCE" or b.role != "BRAND_REFERENCE":
        return Decision.REVIEW, "REVIEW_REFERENCE_ROLE"

    if min(a.evidence_level, b.evidence_level) < 4:
        return Decision.REVIEW, "REVIEW_EVIDENCE_LEVEL"

    if min(a.ownership_score, b.ownership_score) < 0.995:
        return Decision.REVIEW, "REVIEW_LOW_OWNERSHIP"

    return Decision.ACCEPT, "ACCEPT_SAME_CANONICAL_REFERENCE"
```

注意：`0.995` 只是示例，不应该手拍上线。

真正阈值必须由黄金集和 hard-negative 回归集校准。

---

# 28. Candidate Extraction 的安全示例

```python
@dataclass
class Evidence:
    raw: str
    source_type: str
    role_hint: str | None
    context: str | None
    confidence: float

ACCESSORY_CUES = [
    "适配", "兼容", "表带", "表扣", "保护壳", "for", "compatible with", "fits"
]


def role_hint_from_context(context: str) -> str | None:
    s = context.lower()
    if any(cue in s for cue in ACCESSORY_CUES):
        return "COMPATIBLE_REFERENCE"
    return None
```

规则要 source/brand/category 化，并建立测试样本，而不是把所有逻辑塞进一条 regex。

---

# 29. 为什么这个方案比“直接用 LLM 判断两条是不是同款”更适合

LLM matcher 很容易在下面场景产生 false positive：

```text
品牌一样
系列一样
外观一样
年份相近
标题大部分词相同
只有 reference 后缀差一位
```

语言模型会天然倾向“语义相似”。

但当前定义不是语义相似，而是：

```text
canonical_ref(A) == canonical_ref(B)
```

所以更合理的 AI 分工是：

```text
AI：
  从脏数据里找 reference
  判断 reference 的角色
  判断 reference 是否属于当前商品
  提供冲突检测
  排序人工 review

规则引擎：
  canonical reference exact equality
  最终 ACCEPT
```

这正是本篇 MM-LTP 最值得迁移的地方：

> AI 用来抑制噪声，不用来推翻确定性 identifier。

---

# 30. 与此前 b 调研的组合方式

这篇文章不是替代之前的方案，而是补一块此前架构容易漏掉的“candidate ownership”层。

可以组合成：

```text
DeepBlocker / retrieval
    -> 只用于缺 reference 候选召回

属性抽取 / normalization
    -> 找出 reference candidate

parts-distributor-sku-classifier
    -> 编号 role

MM-LTP / 本篇
    -> candidate ownership + noisy context suppression

Confidence / Conformal selective prediction
    -> 对弱证据 gate 做 abstention 校准

TransClean / GraLMatch
    -> 离线发现图结构异常/错误边

最终：
    canonical reference exact policy
```

其中本篇负责：

```text
标题里出现了 reference
        ↓
这个 reference 真属于当前商品本体吗？
```

这是非常关键的一道防误匹配门。

---

# 31. 最终推荐方案

如果只保留一个结论：

> **不要把 MM-LTP 直接训练成“同款腕表 matcher”；把它改造成 Reference Attribution Gate，用视觉和上下文去噪、判断候选 reference 的归属，再让最终匹配严格收口到 `(brand, canonical_reference)` exact equality。**

推荐 production decision path：

```text
1. 多源原始数据不可变保存
2. 多位置抽取 reference candidate，保留 provenance
3. 编号 role 分类
4. RAGate 判断 candidate 是否属于当前商品
5. 品牌化 canonicalization
6. 冲突检测
7. (brand_id, canonical_reference) exact index
8. policy engine 只对高证据记录 ACCEPT
9. 其余全部 REVIEW/ABSTAIN
10. cluster 强制单 canonical reference invariant
11. 持续 hard-negative 审计
```

对于 100 万～1000 万级数据，这套架构的主要计算复杂度集中在 extraction/OCR；最终匹配本身只是 exact-key join，不需要 O(N²) pairwise matching。

这既利用了本文“视觉引导文本去噪”的技术价值，也避免了它对当前业务最危险的误用：

```text
视觉相似 ≠ 同 reference
视觉不可见 ≠ reference 是噪声
```

最终自动合并必须由可审计的 canonical reference 硬证据决定。

---

## 32. 建议第一批验收用例

上线前至少覆盖以下测试：

```text
[PASS] 两平台 structured reference 完全一致
[PASS] 一边 structured，一边标题唯一 reference + OCR 一致
[REJECT] 同品牌同系列但 reference 不同
[REVIEW] 标题出现两个 reference
[REVIEW] 表带标题写“适配 116500LN”
[REVIEW] 商品是盒证但标题包含腕表 reference
[REJECT] structured reference 与 title reference 冲突
[REVIEW] OCR 116500LN，标题 126500LN
[REVIEW] reference 只来自 description
[REVIEW] 视觉模型猜到型号但无文本/OCR证据
[REJECT] brand 不一致但 reference 字符串碰巧一样
[PASS] 品牌规则确认的安全 alias canonicalize 后一致
```

只要其中任何 accessory / adjacent-reference case 被自动 ACCEPT，都应该视为 blocking bug，而不是普通模型误差。

---

## 33. 结论

MM-LTP 原论文证明了一个对当前项目非常实用的原则：

> 电商文本中的 token 并不天然都属于商品本体；使用多模态对齐信息可以在线识别并抑制噪声。

但腕表 reference matching 比论文的视觉检索更严格。因此正确迁移不是“让视觉模型决定是不是同款”，而是：

```text
视觉 + 文本上下文
    -> 判断 reference candidate 的归属/角色/冲突

canonical reference exact equality
    -> 决定最终同款
```

从工程风险和上线成本看，我建议优先实现 **Reference Evidence Store + Role Classifier + Brand Canonicalizer + Exact Reference Index + 三态 Policy Engine**，随后再用几百条 hard-case 黄金标签训练轻量 RAGate。只有当轻量模型无法处理足够多的噪声标题时，再升级到 MM-LTP-style cross-attention adapter。

这样能最大限度满足 Spec 的核心要求：**宁可拒识，也不要误匹配。**
