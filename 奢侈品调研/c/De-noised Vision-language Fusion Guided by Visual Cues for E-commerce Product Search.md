# De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search

> 分析人：c  
> 论文：Zhizhang Hu, Shasha Li, Ming Du, Arnab Dhua, Douglas Gray, **De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search**, CVPR Workshops 2024, pp. 1986–1996  
> CVF：https://openaccess.thecvf.com/content/CVPR2024W/MULA/html/Hu_De-noised_Vision-language_Fusion_Guided_by_Visual_Cues_for_E-commerce_Product_CVPRW_2024_paper.html  
> 作者 PDF：https://kevin-hu.com/publication/mula_2024/mula_2024.pdf  
> Amazon Science：https://www.amazon.science/publications/de-noised-vision-language-fusion-guided-by-visual-cues-for-e-commerce-product-search  
> DOI：10.1109/CVPRW63382.2024.00204  
> 调研清单：`奢侈品文章调研.md`  
> 目标需求：Notion《调研计划 Spec：跨源二奢/腕表商品实体匹配系统（雷小安 × 腕表之家 × 奢当家）》  
> 业务硬定义：**“同一个商品” = 同一 reference number / 型号**；100 万–1000 万级、持续增量、字段稀疏、有图片；**precision 极端优先，绝不能误匹配，允许漏匹配**。

---

## 1. 选题与去重

执行前检查 `奢侈品调研/c/`，已存在多篇实体匹配、reference 抽取、Blocking、多模态商品理解和 selective prediction 分析；本篇标题 **De-noised Vision-language Fusion Guided by Visual Cues for E-commerce Product Search** 不在已有文件中，因此可继续分析。

`奢侈品文章调研.md` 对该论文的推荐点是：电商标题中存在大量营销词、功能词、兼容型号、配件描述等不能从图片中得到支持的噪声文本，论文用视觉对齐信号在线剪除低价值 token，使图文训练更干净。这个问题与二奢腕表非常同构：卖家标题中经常同时出现系列、材质、成色、配件、店铺 SKU、兼容型号甚至其他商品的 reference，直接拿完整标题做 embedding 或 pairwise matching 很容易产生 false positive。

但对当前 Spec 有一个必须先强调的反直觉结论：

> **论文的“视觉引导文本去噪”不能直接作用于 reference 最终判定。**
>
> reference number 往往并不会在商品主图里“可视化”：例如正面表盘照片看不出 `126610LN`，但这个字符串恰恰是当前业务判定“是否同一商品”的最高优先级硬证据。如果照搬论文，让视觉注意力把“不可从图片解释”的 token 全部降权，reference token 反而可能被当作噪声剪掉。

所以本文最终建议不是“用 MM-LTP 替代 reference 规则”，而是做成：

> **Reference Hard Channel + Protected MM-LTP Soft Channel + Precision Gate**

即 reference 抽取/规范化走独立硬证据旁路，MM-LTP 只负责给候选召回、商品语义表征、图文冲突检测和人工复核排序降噪；最终自动 MATCH 仍必须由 canonical reference 的可追溯证据闭环决定。

---

## 2. 论文到底解决什么问题

### 2.1 电商图文训练数据不是天然“对齐”的

论文指出，通用视觉语言模型常默认一条 image-text pair 中，文本大部分都能被图片解释，但电商标题不是这样。商家会在标题中塞入尽可能多的属性，包含：

- 功能性但不可视属性；
- 营销词；
- 材质/规格冗余；
- 搜索 SEO 词；
- 不一定属于主商品视觉主体的描述。

因此“整条标题 ↔ 图片”的粗暴对齐会让模型学习到错误关系。

对应到腕表数据，典型标题可能类似：

```text
劳力士 Rolex 潜航者 黑水鬼 41mm 126610LN 全套 2023 保卡 盒证齐全
支持置换 回收 寄卖 店铺编号 LX-883921
```

这里至少存在四类信息：

```text
真正决定实体：126610LN
视觉语义：劳力士 / 潜航者 / 黑色表盘 / 41mm
交易语义：全套 / 2023 / 盒证齐全
平台或店铺噪声：支持置换 / 回收 / 寄卖 / LX-883921
```

如果全部 token 等权送入跨模态训练，图像编码器会被迫尝试解释“寄卖”“回收”“店铺编号”等内容，降低细粒度视觉表征质量。

### 2.2 论文核心思想：让模型自己学习“哪些词值得看”

论文提出 **MultiModal alignment-guided Learned Token Pruning（MM-LTP）**。

它并不是先离线跑一个规则清洗器，而是在模型训练时直接根据 attention 得到每个文本 token 的重要度，学习一个阈值，把不重要 token 的表示软置零。

因此流程是：

```text
原始商品标题
    │
    ▼
Text Encoder / Fusion Encoder
    │
    ├─ attention score matrix
    │
    ▼
计算每个 text token importance
    │
    ▼
learnable threshold τ_l
    │
    ▼
soft binary mask M_l(token)
    │
    ├─ 保留视觉相关 token
    └─ 抑制低对齐/噪声 token
    │
    ▼
继续多模态训练 / alignment / retrieval
```

它的优点是：文本清洗规则不是人工固定，且每一层可以学不同阈值。

---

## 3. MM-LTP 技术架构深入拆解

## 3.1 同时兼容两类多模态架构

论文特意验证了两种常见结构：

### A. CLIP 类：隐式融合

```text
Image ──> Vision Encoder ──> image embedding
                              │
                              ├─ contrastive alignment loss
                              │
Text  ──> Text Encoder   ──> text embedding
```

CLIP 没有显式 image-text cross-attention，因此 MM-LTP 从 **Text Encoder 的 self-attention** 中估计 token 重要度。

### B. ALBEF 类：显式融合

```text
Image ──> Vision Encoder ─────────────┐
                                      ▼
Text  ──> Text Encoder ──> Fusion Encoder (cross-attention)
                                      │
                                      ▼
                              multimodal objective
```

ALBEF 有文本 query 对图像 key/value 的 cross-attention，因此可以直接利用跨模态 attention 来判断“某个文本 token 是否真正与图片相关”。

论文结果也表明：**有显式 cross-attention 的 ALBEF 上，token pruning 带来的收益更明显。**

这点对腕表落地很重要：如果我们只是做候选召回，CLIP/SigLIP 一类 dual encoder 足够便宜；如果要做“这段标题到底是不是被这张表图支持”的细粒度冲突验证，则 cross-attention 模型更合适。

---

## 3.2 Token importance：从 attention matrix 提取重要度

设 Query token 序列为 `x`，Key token 序列为 `z`，attention score：

```text
Attn(x, z) = x Wq Wk^T z^T / sqrt(d)
```

最直接的方法，是对每个 query token 在所有 attention head、所有 key token 上求平均：

```text
s(x_i) = (1/H) * (1/k) * Σ_h Σ_j Attn(x_i, z_j)
```

但论文认为这样会把 Key 中的噪声也平均进去，所以进一步使用 Key 的聚合 token（论文写作 `[CLS]`）作为更稳定的参照：

```text
s(x_i) = (1/H) * Σ_h Attn(x_i, z_0)
```

直觉是：

- cross-attention 时，图像聚合 token 汇总了整体视觉概念；
- self-attention 时，文本聚合 token 汇总了整体语义；
- 单独看聚合 token 比把所有 key token 一起平均更不容易被局部噪声影响。

论文消融实验验证了这种 importance 计算方式稳定优于多种替代设置。

### 工程实现需要注意一个细节

论文用 `[CLS]` 泛指聚合位置，但不同 backbone 的真实 pooled token 并不一样。

例如 HuggingFace 标准 CLIP 文本分支并不是 BERT 式 `[CLS]` 池化语义，通常以 EOT 等位置构造最终表示。落地时不能机械写死 `token_index = 0`，应该针对 backbone 明确：

```python
pooled_index = model_adapter.get_pool_token_index(input_ids)
```

而不是假设所有模型都有同一种 `[CLS]`。

---

## 3.3 Learnable threshold：每层自己学习剪多少 token

静态阈值很难通用，因为：

- 不同品牌标题噪声比例不同；
- 不同来源标题风格不同；
- Transformer 深层和浅层的语义粒度不同；
- CLIP self-attention 与 ALBEF cross-attention 的 score 分布不同。

因此论文把每层阈值 `τ_l` 直接设为可学习参数。

第 `l` 层、第 `i` 个 token 的 soft mask：

```text
M_l(x_i) = sigmoid((s_l(x_i) - τ_l) / T)
```

其中 `T` 是 temperature。

当 `T` 很小时：

```text
s > τ  -> mask ≈ 1
s < τ  -> mask ≈ 0
```

但仍可反向传播，因此训练时可以学习“这个 token 应该留还是删”。

论文实验中 `T = 1e-4`，因此近似硬二值，但训练仍可微。

实际执行并不需要从计算图里真的删除 token；可以先把表示乘 mask：

```python
hidden = hidden * mask.unsqueeze(-1)
```

这样实现最简单，也不会立即破坏 sequence position 和已有模型结构。

---

## 3.4 Pruning loss：主动鼓励模型丢弃低价值 token

如果只依赖原模型任务 loss，模型可能发现“全部保留最安全”，于是阈值学不会真正 pruning。

因此论文加了一个 L1 风格的剪枝正则：

```text
L_prune = (1/N) * Σ_l ||M_l(x)||_1 / d_l^Q
```

总目标：

```text
L = L_model + λ * L_prune
```

论文实验取 `λ = 0.1`。

这个目标会形成一个拉扯：

```text
L_model      希望保留对任务有用的信息
L_prune      希望尽量少保留 token
```

最后模型倾向只保留对多模态 alignment 真正有贡献的文本内容。

---

## 3.5 论文训练规模与实现参数

论文基于 Amazon ESCI 扩展出多模态数据：

```text
总商品数：717,511
训练商品：637,511
测试商品：80,000
训练 image-text pairs：858,231
测试 image-image pairs：186,084
```

每个商品包含：

- title；
- main image；
- 0–10 张 auxiliary images。

模型：

```text
CLIP:
  Vision: ViT-B/16, 12 layers, 86M
  Text:   12-layer Transformer, 63M

ALBEF:
  Vision: ViT-B/16
  Text + Fusion: 6-layer Transformer, total 124M
```

训练环境：

```text
8 × NVIDIA A100
PyTorch
Ray distributed
```

CLIP 训练参数包括：

```text
epochs > 100
batch_size = 1360
AdamW
weight_decay = 0.02
lr: 5e-6 -> warmup 2e-5 -> cosine decay 5e-6
T = 1e-4
lambda_prune = 0.1
```

这意味着论文证明的是“大规模领域微调中 MM-LTP 有效”，而不是说当前项目必须复制这套算力。

当前项目反而有一个优势：虽然人工黄金 cross-source pair 只有几百对，但**每条商品自己的 title-image 天然就是弱监督 image-text pair**。百万级抓取数据足以先做自监督/弱监督领域适配，几百对黄金标签只需要用于最终 match gate、阈值校准和 hard negative 验证。

---

## 4. 论文实验结果说明了什么

### 4.1 CLIP：在 self-attention 上 pruning 也有效

第一阶段从 open-domain CLIP 微调到电商域：

```text
               R@1    R@5    R@10
Off-the-shelf   42.56  51.29  56.62
CLIP-FT1        53.59  63.24  69.03
CLIP-FT1wTP     55.21  65.13  70.97
```

MM-LTP 相对普通领域微调继续提升约 1.6–1.9 个百分点。

说明即便没有显式图文 cross-attention，文本 self-attention 里的 token importance 也能提供有用的去噪信号。

### 4.2 ALBEF：显式图文交互更适合做去噪

第一阶段：

```text
                    R@1    R@5    R@10
Off-the-shelf       38.60  47.40  53.03
ALBEF-FT1           51.68  62.27  68.68
ALBEF-FT1wTP        54.59  65.09  71.36
ALBEF-FT1wTP-All    57.06  67.54  73.74
```

同时在 Text self-attention + Fusion cross-attention 上应用 pruning 时，相对普通 fine-tune 的 R@1 提升 5.38 个百分点。

这说明“文本内部语义去噪 + 图文显式对齐去噪”叠加效果最好。

### 4.3 Grad-CAM 给了可解释证据

论文观察到：

- `valve`、`dog`、`handle` 等视觉可描述词，会对图像局部形成集中 attention；
- 品牌或无法直接视觉映射的词会产生更分散的 attention。

这恰好揭示本需求最大的风险：

> **reference、年份、机芯编号等业务关键 identifier 可能在主图里没有强视觉区域，但业务上绝不能被视为“无关词”。**

因此论文的机制要借，但业务语义必须覆盖原论文的“纯视觉相关性”目标。

---

## 5. 为什么不能直接拿 MM-LTP 做腕表同款终判

### 5.1 “视觉不重要”与“实体不重要”不是一回事

MM-LTP 优化的是：

```text
哪些词有助于 image-text alignment / retrieval
```

当前业务真正需要优化的是：

```text
两条记录能否被证明拥有同一 canonical reference
```

二者不等价。

例如：

```text
A: Rolex Daytona 116500LN white dial
B: Rolex Daytona 126500LN white dial
```

两者外观、品牌、系列、颜色都很相似，visual embedding 很可能非常接近；但 reference 不同，必须 `NO_MATCH`。

反过来：

```text
A: 126610LN
B: 劳力士潜航者 黑水鬼（图片里看不到编号）
```

即使 reference token 对视觉 attention 很低，只要它来自可信结构化字段/标题直接抽取，它仍然是最高等级证据。

### 5.2 主图不能证明 reference

表背刻字、保卡、吊牌可能显示 reference；但大多数正面商品图只表现：

- 表盘布局；
- 表圈颜色；
- 表壳形状；
- logo；
- 指针/刻度。

这些最多支持“外观一致/冲突”，通常不能单独证明 reference。

因此视觉模型只应该有两种权限：

```text
1. 候选召回 / 排序
2. 冲突否决 / 降低自动化等级
```

不应该拥有：

```text
“视觉很像 -> 自动 MATCH”
```

### 5.3 论文指标是 Recall@K，不是 precision-first entity resolution

论文关注视觉检索 R@1/R@5/R@10，而当前系统要求“绝不能误匹配”。

所以不能用论文中 Recall 提升直接推断实体匹配可上线；必须重新设计 precision-first gate 和 hard-negative 评测。

---

## 6. 直接可落地的改造：Protected MM-LTP

## 6.1 两条文本通路彻底分开

推荐架构：

```text
                    ┌──────────────────────────┐
Raw title/fields ──>│ Hard Identifier Channel  │──> reference evidence
                    │ 不做视觉 token pruning    │
                    └──────────────────────────┘
                              │
                              │
                              ▼
                    Canonical Reference Resolver

Raw title + images ─────────────────────────────────────────┐
                                                            ▼
                    ┌──────────────────────────┐
                    │ Protected MM-LTP Channel │──> semantic/image embedding
                    │ 图文去噪 + 软证据          │──> visual conflict score
                    └──────────────────────────┘
```

### Hard Identifier Channel 负责

- reference candidate extraction；
- 编号角色分类；
- brand/model/series 结构化；
- reference canonicalization；
- provenance；
- 冲突 detection。

### Protected MM-LTP Channel 负责

- 删除营销/交易/SEO 噪声；
- 生成更干净的文本/图像 embedding；
- image-text consistency；
- same-series hard negative 排序；
- 人工复核 explanation。

最终 MATCH 只由 Hard Channel + Gate 决定。

---

## 6.2 对 reference span 做强制保护

如果确实要在同一 Transformer 中加入 MM-LTP，必须加入 `protected_mask`：

```python
soft_mask = sigmoid((importance - tau[layer]) / temperature)

# protected token 永远保留
final_mask = protected_mask + (1.0 - protected_mask) * soft_mask

hidden = hidden * final_mask.unsqueeze(-1)
```

`protected_mask = 1` 的 token 至少包括：

- reference extractor 找出的候选 span；
- 高可信品牌词；
- 机芯/系列里用于区分变体的关键 token；
- OCR 直接读出的 identifier 映射 token（如果拼接进文本）。

更稳妥的第一版甚至可以不用同流保护，而是保持 **raw hard-channel 和 pruned semantic-channel 双流**，避免视觉训练误伤 identifier。

---

## 6.3 reference 证据必须带 provenance

不要只保存：

```json
{"reference": "126610LN"}
```

应保存：

```json
{
  "raw": "126610LN",
  "canonical": "126610LN",
  "brand_id": "rolex",
  "source_type": "title_span",
  "source_field": "title",
  "char_start": 17,
  "char_end": 25,
  "role": "manufacturer_reference",
  "extractor": "regex+brand_dict+classifier_v3",
  "confidence": 0.998,
  "accepted": true,
  "conflicts": []
}
```

图像 OCR 同理：

```json
{
  "canonical": "126610LN",
  "source_type": "image_ocr",
  "image_role": "warranty_card",
  "bbox": [x1, y1, x2, y2],
  "ocr_text": "126610LN",
  "accepted": true
}
```

这样最终 MATCH 能回答：

> 为什么这两条记录被合并？

而不是只有一个不可解释模型分数。

---

## 7. 面向 Spec 的完整生产架构

```text
雷小安 / 腕表之家 / 奢当家
        │
        ▼
┌──────────────────────┐
│ Source Adapters       │
│ raw snapshot + delta  │
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ Normalization Layer  │
│ brand/title/field/img │
└──────────┬───────────┘
           ├────────────────────────────┐
           ▼                            ▼
┌──────────────────────┐      ┌────────────────────────┐
│ Reference Hard Path  │      │ Media Understanding    │
│ regex/dict/LLM/NER   │      │ image role + OCR       │
│ role classification  │      │ image hash/quality     │
└──────────┬───────────┘      └──────────┬─────────────┘
           │                             │
           └──────────────┬──────────────┘
                          ▼
                ┌──────────────────────┐
                │ Reference Resolver   │
                │ canonical + conflict │
                └──────────┬───────────┘
                           │
         ┌─────────────────┴──────────────────┐
         ▼                                    ▼
┌──────────────────────┐            ┌────────────────────────┐
│ Exact Ref Index      │            │ Protected MM-LTP       │
│ brand::canonical_ref │            │ semantic / visual vec  │
└──────────┬───────────┘            │ consistency / veto     │
           │                        └──────────┬─────────────┘
           └─────────────────┬────────────────┘
                             ▼
                   ┌──────────────────────┐
                   │ Precision Gate       │
                   │ MATCH/NO/ABSTAIN     │
                   └──────────┬───────────┘
                              ▼
                   ┌──────────────────────┐
                   │ Entity / Audit Store │
                   │ decision + evidence  │
                   └──────────────────────┘
```

---

## 8. 百万到千万级不能做全量 pairwise

三个来源如果各有百万级记录，直接笛卡尔积不可接受。

当前业务定义其实给了一个极强的 Blocking key：reference。

### 8.1 第一层：Exact Reference Index

一旦某条记录有 accepted canonical reference：

```text
key = brand_id + "::" + canonical_reference
```

例如：

```text
rolex::126610LN
omega::31030425001002
```

把所有来源记录直接挂到倒排桶：

```text
ReferenceBucket
  key
  -> source A record ids
  -> source B record ids
  -> source C record ids
```

这是主路径，复杂度近似 `O(N)` 构建 + `O(1)/O(logN)` 查桶，不需要两两比较。

### 8.2 第二层：Reference Unknown / Ambiguous 队列

只有以下记录进入 expensive path：

- reference 缺失；
- 抽出多个冲突 reference；
- reference 疑似 OCR 错一位；
- 型号看似像平台 SKU；
- 长尾品牌字典不存在。

候选召回可用：

```text
brand exact
+ series blocking
+ reference char n-gram / edit distance
+ MM-LTP text embedding ANN
+ image embedding ANN
```

但召回结果只是候选，不会自动 MATCH。

### 8.3 不要把模糊 reference “纠正”成最近 reference 后直接合并

例如：

```text
OCR: 12661OLN
候选字典最近：126610LN
```

系统可以生成：

```text
candidate_reference = 126610LN
```

但 provenance 必须仍是 `ocr_fuzzy_candidate`，不能偷偷升级成 `accepted_exact_reference`。

只有额外直接证据（标题、结构化字段、另一张保卡/表背 OCR）确认后才进入自动 MATCH。

---

## 9. Precision Gate：把“模型”降级成软证据

建议所有跨源 pair 最终只输出三态：

```text
MATCH
NO_MATCH
ABSTAIN
```

不要只输出概率。

参考伪代码：

```python
def decide(a, b):
    # 1. 品牌冲突直接否决
    if a.brand_id and b.brand_id and a.brand_id != b.brand_id:
        return NO_MATCH("brand_conflict")

    # 2. 两侧必须都有可追溯、已接受 reference 证据
    if not a.ref.accepted or not b.ref.accepted:
        return ABSTAIN("reference_not_proven")

    # 3. canonical reference 不同，绝不靠视觉相似度挽救
    if a.ref.canonical != b.ref.canonical:
        return NO_MATCH("reference_conflict")

    # 4. 任意一侧出现多个互相冲突 reference
    if a.ref.conflicts or b.ref.conflicts:
        return ABSTAIN("multi_reference_conflict")

    # 5. 编号角色可疑，例如平台 SKU / 配件兼容型号
    if a.ref.role != "manufacturer_reference" or b.ref.role != "manufacturer_reference":
        return ABSTAIN("reference_role_uncertain")

    # 6. 视觉只能做 veto，不能单独创造 MATCH
    if visual_conflict(a, b) >= VISUAL_CONFLICT_THRESHOLD:
        return ABSTAIN("visual_conflict")

    return MATCH("same_canonical_reference_with_no_conflict")
```

这里的原则非常重要：

> **软模型只允许把 MATCH 降级成 ABSTAIN，不允许把缺失硬证据的 ABSTAIN 升级成 MATCH。**

这样才能满足 precision-first。

---

## 10. MM-LTP 在这个系统里最值得做的 4 件事

## 10.1 商品语义 embedding 去噪

原始标题：

```text
劳力士潜航者126610LN黑水鬼41mm全套支持置换高价回收门店现货
```

经过 protected pruning 后希望接近：

```text
劳力士 潜航者 126610LN 黑水鬼 41mm
```

而不是让：

```text
全套 / 支持置换 / 高价回收 / 门店现货
```

主导 embedding。

这个 embedding 用于：

- unresolved record candidate retrieval；
- 同系列 hard negative mining；
- 人工 Review 排序。

## 10.2 视觉冲突检测

当两条记录 reference 相同，但：

- 一张图明显是劳力士；
- 另一张图明显是欧米茄；
- 或表盘布局完全不同；

视觉分支可以发出 conflict signal，迫使最终 Gate `ABSTAIN`。

注意这里的目标不是“图像证明 same reference”，而是：

```text
发现硬字段可能被错误抽取/污染
```

## 10.3 image-text consistency 发现脏记录

对单条商品内部计算：

```text
image <-> cleaned title
```

如果一致性极低，标记：

```text
record_quality = suspicious
```

常见原因可能包括：

- 抓错图；
- 标题串行错位；
- 配件图当主图；
- reference 来自兼容对象；
- 卖家复用模板标题。

这种记录不参与自动 MATCH，直接降级。

## 10.4 为人工复核提供 token 级解释

保留每层/最终 token importance，可以把审核界面展示成：

```text
[劳力士] 高
[潜航者] 高
[126610LN] protected
[黑水鬼] 高
[支持置换] 低
[高价回收] 低
[店铺编号 LX-883921] 低/role=store_sku
```

比只给一个 0.93 similarity 更有可解释性。

---

## 11. reference 抽取本身应该怎么做

MM-LTP 不能代替 identifier extraction。

推荐多路并行：

```text
structured field parser
        │
regex / brand pattern library
        │
NER / span classifier
        │
LLM constrained extraction
        │
image OCR（表背/保卡/吊牌）
        │
        ▼
Reference Candidate Set
        │
        ▼
Role Classifier
        │
        ├─ manufacturer_reference
        ├─ seller_sku
        ├─ platform_item_id
        ├─ compatible_reference
        ├─ movement/caliber
        └─ unknown_identifier
```

### 关键：先判“编号是什么角色”，再 canonicalize

否则很容易把：

```text
平台商品号
店铺库存号
保卡序列号
机芯号
配件兼容型号
```

误当成 reference。

### canonicalization 也必须 brand-aware

只做安全变换：

```text
Unicode normalize
trim
统一大小写
统一明确无语义的 separator
```

不要全局做：

```text
删除所有 - / . / 空格
删除 suffix
只保留数字
```

因为不同品牌的 suffix、分隔符可能有实际型号含义。

建议：

```python
canonical = brand_adapter[brand_id].normalize_reference(raw)
```

而不是一个全品牌通用 regex。

---

## 12. 训练方案：几百人工标签也能用

论文虽然用了 70 多万商品，但当前项目不需要先获得几十万人工 match 标签。

### 12.1 MM-LTP 领域适配：用天然 image-title pair

每条已有商品记录天然提供：

```text
(product title, product main image)
(product title, auxiliary image)
```

这部分可以从百万级抓取数据自动产生，用来学习：

- 哪些标题 token 与图像相关；
- 哪些卖家模板词可剪；
- 腕表领域的细粒度视觉 embedding。

不依赖跨源黄金 pair。

### 12.2 黄金标签集中到真正危险的 hard negatives

几百对人工标注不要随机抽。

优先覆盖：

```text
同品牌 + 同系列 + reference 只差 1 位
同 reference 主体 + suffix 不同
同壳型不同尺寸
同表盘色不同 reference
新旧代外观接近
配件标题出现腕表 reference
兼容型号列表
平台 SKU 长得像 reference
OCR O/0, I/1, B/8 混淆
多个 reference 同时出现在标题
结构化字段与标题冲突
图片与文本明显冲突
```

这些才会决定线上 false positive。

### 12.3 训练/验证数据禁止随机 pair split

应该按 reference / 系列分组切分，至少保留：

- seen series / unseen reference；
- unseen series；
- 新来源；
- 时间增量批次。

否则模型可能只是记住 reference 或标题模板，离线指标虚高。

---

## 13. 一个可以直接实现的 Protected MM-LTP 模块

伪代码：

```python
class ProtectedTokenPruner(nn.Module):
    def __init__(self, num_layers, init_tau, temperature=1e-4):
        super().__init__()
        self.tau = nn.Parameter(torch.tensor(init_tau))
        self.temperature = temperature

    def forward(
        self,
        hidden_states,
        attentions,
        pooled_key_index,
        protected_mask,
        layer_idx,
    ):
        # attentions: [B, H, Q, K]
        # 取聚合 key token 得到 query token importance
        score = attentions[:, :, :, pooled_key_index].mean(dim=1)  # [B, Q]

        soft_mask = torch.sigmoid(
            (score - self.tau[layer_idx]) / self.temperature
        )

        # reference / brand 等硬 token 永不被剪
        final_mask = protected_mask + (1.0 - protected_mask) * soft_mask
        final_mask = final_mask.clamp(0.0, 1.0)

        pruned_hidden = hidden_states * final_mask.unsqueeze(-1)
        prune_loss = final_mask.mean()

        return pruned_hidden, prune_loss, final_mask
```

训练：

```python
loss = model_task_loss + lambda_prune * prune_loss
```

生产中同时输出：

```text
embedding
image_text_consistency
per_token_importance
pruned_tokens
protected_tokens
```

方便审计。

### 第一版不建议真的减少 token sequence length

软置零已经能验证收益；真实删 token 会引入：

- position index 变化；
- KV cache/attention mask 改造；
- batch 内动态 shape；
- ONNX/TensorRT 导出复杂度。

在当前需求中，MM-LTP 的首要价值是“降噪”而不是推理提速，因此第一版保留固定长度最稳妥。

---

## 14. 数据层建议

至少保留以下实体。

### 14.1 `product_record`

```text
record_id
source
source_item_id
title_raw
fields_raw
brand_id
created_at
updated_at
raw_version
```

### 14.2 `reference_observation`

```text
observation_id
record_id
raw_value
canonical_value
brand_id
role
provenance_type
provenance_locator
extractor_version
confidence
accepted
conflict_group
```

### 14.3 `media_observation`

```text
record_id
image_id
image_role
phash
ocr_text
ocr_boxes
image_embedding_version
quality_score
```

### 14.4 `match_decision`

```text
left_record_id
right_record_id
decision             # MATCH / NO_MATCH / ABSTAIN
canonical_reference
reason_code
evidence_json
model_versions
rule_version
created_at
```

### 14.5 `reference_entity`

```text
brand_id
canonical_reference
entity_id
member_record_ids
```

如果业务定义始终是“同一 reference = 同一商品”，最终 cluster 完全可以围绕 `(brand_id, canonical_reference)` 构建，而不是让 clustering 算法自由学习实体边界。

---

## 15. 增量更新怎么做

每次新抓取/字段变化：

```text
1. 生成 record version
2. 重跑受影响字段的 reference extraction
3. 重跑新增图片 OCR / embedding
4. 得到新的 accepted canonical reference
5. 查询 exact reference index
6. 执行 Precision Gate
7. 写 match_decision + audit evidence
8. 只有 reference 证据变化时才重新计算实体归属
```

不要每次全量重跑所有 pair。

如果模型版本升级：

```text
soft evidence 可以批量重算
hard reference provenance 不丢失
历史 match decision 可审计回放
```

这样可以安全迭代 MM-LTP，而不会让模型升级偷偷改变业务实体定义。

---

## 16. 推荐技术栈

技术栈不限，可以优先选择成熟、可审计组件。

### 数据与索引

```text
Raw / history: S3 / MinIO + Parquet/Iceberg
Operational DB: PostgreSQL
Text/reference search: OpenSearch / PostgreSQL trigram
Vector ANN: FAISS / Milvus / Qdrant
```

### Pipeline

```text
Batch/backfill: Spark / Ray
Incremental: Kafka/Pulsar + worker
Orchestration: Airflow / Temporal
```

### 模型

```text
Reference extractor: regex + brand dictionaries + small encoder/LLM constrained JSON
OCR: PaddleOCR / PP-OCR / cloud OCR 做 ensemble（关键图片人工抽检）
Semantic baseline: CLIP/SigLIP
Cross-modal verifier: ALBEF/BLIP-style cross-attention
Protected MM-LTP: PyTorch custom module
Serving: Triton / TorchServe / ONNX Runtime
```

重点不是选哪个数据库，而是保证：

```text
hard evidence 和 soft evidence 分层
每个 MATCH 都可回溯到原始 reference 证据
soft model 没有越权自动合并能力
```

---

## 17. 上线顺序：先规则闭环，再让 MM-LTP 进入 shadow

### Phase A：Reference-first baseline

先完成：

- brand normalization；
- reference extraction；
- role classifier；
- brand-aware canonicalization；
- exact reference index；
- MATCH/NO_MATCH/ABSTAIN gate；
- audit log。

没有视觉模型也能先得到一个极高 precision 的 baseline。

### Phase B：MM-LTP Shadow Mode

MM-LTP 只输出：

```text
cleaned semantic embedding
image-text consistency
visual conflict score
token importance
```

但不影响 MATCH，只记录它是否能提前发现人工确认的脏数据。

### Phase C：MM-LTP 获得“否决权”

当 hard-negative 验证充分后，只允许：

```text
MATCH -> ABSTAIN
```

例如视觉冲突极高、图片与标题不一致时自动降级人工复核。

### Phase D：用于 unresolved candidate ranking

对 reference 缺失记录，用 MM-LTP embedding 找最可能的候选 reference/商品，供：

- OCR 二次定位；
- 人工 review；
- 主动学习采样。

仍然不允许仅凭相似度自动 MATCH。

---

## 18. 评测指标必须重做，不能照搬论文 Recall@K

建议分四层。

### A. Reference extraction

```text
Exact reference precision
Exact reference recall
Role classification precision
Conflict detection recall
```

其中 `manufacturer_reference` precision 最重要。

### B. Candidate retrieval

```text
Recall@10 / Recall@50
```

只考察 unresolved path，不代表最终自动化质量。

### C. Auto-match Gate

主指标：

```text
Auto-match precision
False positive count
Abstain rate
Coverage
```

在 precision-first 约束下，应允许 coverage 很低。

### D. MM-LTP soft verifier

```text
Hard-negative conflict recall
Clean title token retention
Noise token suppression
Reference token preservation = 100%
Brand token preservation
Image-text dirty-record detection AUCPR
```

特别加入：

> **Reference Token Preservation Rate 必须是 100%**

因为 Protected MM-LTP 的第一原则就是绝不能因为视觉弱对齐把 identifier 剪掉。

---

## 19. 风险清单

### 风险 1：主图不包含 reference 信息

解决：视觉只做 soft evidence；表背/保卡/吊牌单独分类与 OCR。

### 风险 2：视觉相似的不同 reference 被误合并

解决：canonical reference 不同直接 `NO_MATCH`；视觉无权覆盖。

### 风险 3：reference 字符串被 MM-LTP 当噪声

解决：Hard Channel 独立；或 protected token mask 强制保留。

### 风险 4：配件标题含腕表 reference

例如“适配 126610LN 表带”。

解决：编号 role classifier；识别 `compatible_reference`，不能升级为当前商品 reference。

### 风险 5：店铺 SKU 长得像型号

解决：来源字段 schema + 编号 role classifier + brand reference pattern dictionary。

### 风险 6：OCR 把 0/O、1/I、8/B 读错

解决：保留 raw OCR，不做无证据纠正；模糊候选只能进入 review/二次 OCR。

### 风险 7：相同 reference 的图片可能完全不同

二手商品拍摄角度、光照、配件数量差异很大。

解决：视觉冲突阈值必须非常保守；只识别明显冲突，不要求视觉必须高相似。

### 风险 8：几百对标签无法证明“绝对零误匹配”

解决：不把“模型置信度”当证明。安全性来自硬业务规则、可追溯 reference 证据和 abstain 机制；黄金集用于寻找规则漏洞，而不是用统计分数宣称绝对保证。

---

## 20. 与已有 `c` 调研的组合关系

这篇 MM-LTP 最适合做已有方案中的“多模态软证据去噪层”。

可以组合为：

```text
DeepBlocker / pyJedAI
    -> 大规模候选生成框架

Using LLMs for Extraction / PAM / Large-scale Attribute Extraction
    -> reference/属性抽取

parts-distributor-sku-classifier
    -> identifier role 分类，防止 SKU 冒充 reference

Ameli / MOON2.0 / 本篇 MM-LTP
    -> 多模态候选与图文一致性

AnyMatch / Ditto 类 Matcher
    -> unresolved hard-case 软判别

Confidence / Conformal Selective Prediction
    -> MATCH/ABSTAIN 阈值与风险控制

TransClean / GraLMatch
    -> 多源图一致性审计、防止错误边污染实体簇
```

本篇独特贡献是：

> **在进入多模态 embedding / verifier 之前，先让视觉对齐信号帮助过滤标题噪声；但针对 reference-first 业务必须增加 identifier protection。**

---

## 21. 最小可落地版本（推荐）

如果现在就开始实现，不建议先训练完整 ALBEF + MM-LTP。

### MVP-1

```text
1. source schema normalization
2. brand dictionary
3. reference regex/span extractor
4. identifier role classifier
5. brand-aware canonicalizer
6. exact reference inverted index
7. three-state Precision Gate
8. audit evidence store
```

### MVP-2

引入现成 CLIP/SigLIP 生成 image/text embedding，先测：

```text
同 reference image similarity 分布
不同 reference hard negative 分布
image-text consistency 分布
```

只做 shadow。

### MVP-3

在文本分支加入 Protected MM-LTP：

```text
reference spans protected
brand spans protected
营销词/交易词允许 pruning
```

用百万级 title-image 自监督微调。

### MVP-4

若确实观察到“cross-attention 比 dual encoder 更能发现标题脏词/图文冲突”，再引入 ALBEF/BLIP-style verifier，只跑 top-K unresolved / suspicious records，避免全量高成本推理。

这条路线比“一上来训练完整多模态 matcher”更符合当前 precision-first 和工程性价比。

---

## 22. 最终结论

MM-LTP 论文值得参考的不是“token pruning 可以提 Recall”这一表面结果，而是一个更有工程价值的原则：

> **电商标题不是可靠的自然语言描述，必须在建模时显式处理“哪些文本真的属于当前商品”。**

对于跨源腕表实体匹配，这个原则尤其重要，因为标题中可能混入兼容型号、店铺 SKU、配件 reference、营销词和交易信息。

但当前 Spec 又比论文多了一条更严格的业务定义：

> **同一商品 = 同一 reference。**

因此最合理的落地不是直接复制 MM-LTP，而是改成 **Reference Hard Channel + Protected MM-LTP Soft Channel + Precision Gate**：

1. reference 抽取/角色识别/规范化独立存在，永不被视觉剪枝；
2. MM-LTP 清理标题噪声，改善候选检索、图文一致性和冲突识别；
3. image/semantic similarity 永远不能覆盖 reference 冲突；
4. 自动 MATCH 必须要求两侧都有 accepted canonical reference 且完全一致；
5. 视觉和模型只拥有“否决/降级为 ABSTAIN”的权限；
6. reference 缺失、冲突或角色不明时宁可漏匹配，也不自动猜。

如果按照这个改造执行，MM-LTP 能成为现有 entity resolution 系统里非常合适的一层：**它不是裁判，而是负责把软证据变干净、把危险样本暴露出来，让 reference 这个硬裁判更少被脏数据欺骗。**
