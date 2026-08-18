# PAM：把商品图片中的 OCR / 视觉证据变成 Reference 抽取的安全辅助通道

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取论文 **PAM: Understanding Product Images in Cross Product Category Attribute Extraction** 做深入分析。

- 论文：<https://arxiv.org/abs/2106.04630>
- PDF：<https://arxiv.org/pdf/2106.04630>
- DOI：<https://doi.org/10.1145/3447548.3467164>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

分析前已先检查 `奢侈品调研/a` 当前已有结果，并排除已分析文章/项目：

- `Ameli.md`
- `AnyMatch -- Efficient Zero-Shot Entity Matching with a Small Language Model.md`
- `ComEM.md`
- `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
- `DeepBlocker.md`
- `Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration.md`
- `Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes.md`
- `Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification.md`
- `Tailoring entity resolution for matching product offers.md`
- `TransClean - Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency.md`
- `Using LLMs for the Extraction and Normalization of Product Attribute Values.md`
- `parts-distributor-sku-classifier.md`
- `pyJedAI.md`

`PAM` 尚未出现，因此本次继续执行分析。

当前 Spec 的核心不是泛化的“两个商品看起来是不是同一个”，而是：

> **只有同一品牌下的同一 reference number / 型号，才能判定为同一商品实体；precision 极端优先，允许漏匹配，但不能误匹配。**

PAM 最值得借鉴的不是“拿多模态 Transformer 直接做 Match / NoMatch”，而是以下三个工程思想：

1. **不要只看商品标题。** 图片中的 OCR 文本经常补充结构化字段和标题缺失的信息；
2. **不要自由生成属性值。** 应该让输出受一个受控候选集合约束；
3. **候选集合必须受商品类别约束。** 对本需求更应进一步改造成受 `brand + reference registry` 约束。

因此，本项目最合适的落地方式是把 PAM 改造成一个 **Reference Evidence Extractor（型号证据抽取器）**：

```text
商品记录
  ├─ 结构化 reference 字段
  ├─ 标题 / 描述文本
  └─ 商品图片
        └─ OCR + 版面位置

            ↓

Reference Candidate Extraction
            ↓
Brand-aware Canonicalization / Registry Lookup
            ↓
Evidence Ledger（保留每条证据来源）
            ↓
Precision-first Resolver
  ├─ exact agreement -> ACCEPT
  ├─ high-trust conflict -> CONFLICT
  ├─ 单路或模糊证据 -> ABSTAIN / REVIEW
  └─ 不允许模型相似度越过硬规则
            ↓
reference_entity_id
            ↓
三源按 reference_entity_id exact join
```

**核心结论：PAM 适合解决“reference 在图片里、标题里、OCR 里怎么找出来”的问题，不适合直接决定“两个腕表是不是同款”。最终同款判定仍应由 canonical reference exact equality 收口。**

---

# 1. PAM 论文到底解决什么问题

PAM 的任务是电商商品属性值抽取。

给定：

- 商品类别；
- 商品标题/文本；
- 商品图片；
- 指定属性，例如 Brand、Item Form；

模型输出该属性的值。

与纯文本方案不同，PAM 同时利用三种信息：

```text
Product Text
OCR Tokens inside Image
Visual Object Regions
```

论文在 Amazon 商品数据上观察到：大量网页结构化属性缺失时，图片仍然包含可恢复的信息；文中统计指出，在其研究的 30 个属性中，超过 20% 的缺失属性值只能从商品图片中识别出来。

这与当前腕表场景高度同构：

```text
标题：劳力士潜航者 黑水鬼 自动机械 男表
结构化 reference：空
图片：表背 / 保卡 / 吊牌 / 柜台标签上出现 126610LN
```

如果只做标题 NER，可能永远拿不到 reference；图片 OCR 是真正有价值的第二证据通道。

论文还强调图片有第二个作用：**cross validation**。

即文本中出现多个可能值时，图片可以帮助判断哪个才属于当前商品。

这对二奢比普通电商更重要，因为标题/描述中经常同时出现：

- 当前商品 reference；
- 同系列比较型号；
- “适配 XX 型号”的配件型号；
- 盒证/表带/附件型号；
- 商家自己的 SKU；
- 库存号、流水号、内部 ID。

因此，Reference 抽取不能等价于：

```text
regex 找到一个像型号的字符串 -> 当作 reference
```

而应该变成：

```text
候选编号抽取 -> 编号角色识别 -> 品牌 reference 字典验证 -> 多证据交叉确认
```

---

# 2. PAM 的技术架构

## 2.1 整体：多模态 Sequence-to-Sequence

PAM 使用一个 Transformer 形式的 encoder / decoder。

论文的主要输入是：

```text
1. Product Text
2. OCR Tokens
3. Visual Objects
4. Previously Decoded Tokens
```

这些不同模态先被映射到同一维度，然后拼成一条 embedding sequence，送进 Multi-Modal Transformer。

这样做的关键点是：

> 任意一个文本 token 都可以直接 attention 到 OCR token 或图片对象区域；OCR token 也能结合自身所在位置和视觉区域来判断语义。

这比“标题模型 + OCR 模型各自跑完最后拼分数”更强，因为跨模态关联发生在 Transformer 内部。

论文实现参数：

- Transformer：4 层；
- Attention Heads：12；
- Hidden / embedding dimension：768；
- batch size：128；
- Adam；
- base learning rate：`1e-4`；
- max decoding steps：10；
- 训练 24,000 iterations。

对于今天要落地的腕表 reference 抽取，这个具体模型规模已经不是最重要的部分；真正值得保留的是 **“多模态证据进入同一受约束解码空间”** 这一设计。

---

## 2.2 文本编码：BERT 前三层

论文将商品文本 token 输入 BERT，使用前 3 层输出作为 text embedding，并在训练中 fine-tune。

PAM 实验中最多使用：

```text
100 product text tokens
```

在腕表场景里，可以把文本输入拆成带字段标签的序列，而不要把所有信息无脑拼接：

```text
[TITLE] 劳力士 Rolex 潜航者型 126610LN 黑盘
[DESC] 全套 2024 年 ...
[STRUCTURED_REF] null
[PLATFORM_SKU] SDJ-882931
[BRAND] Rolex
```

这种字段标签非常重要，因为：

```text
126610LN 出现在 STRUCTURED_REF
```

和：

```text
126610LN 出现在“兼容/对比/附件说明”
```

不能拥有同样的可信度。

---

## 2.3 图片编码：Faster R-CNN + Region Feature

PAM 使用 Faster R-CNN，backbone 是 ResNet-101，并预训练于 Visual Genome。

对于每个检测到的图片对象，模型保留：

```text
visual region feature
+
bounding box coordinates
```

论文最多取 10 个 image objects。

这个设计对 reference 本身不是最关键，因为腕表的外观视觉相似度 **不能作为 reference 相同的正证据**：同系列不同 reference 往往外观极其相似。

但图片区域仍有两个实际用途：

1. 区分 OCR 字符出现在什么物体上；
2. 给 OCR token 提供版面/位置上下文。

例如：

```text
“126610LN” 位于保卡型号栏
```

和：

```text
“126610LN” 位于背景海报 / 搜索截图 / 聊天截图
```

可靠性完全不同。

所以在当前项目中，视觉模型应该从“认表款”降级为“帮助理解 OCR 来源区域”。

---

## 2.4 OCR 编码：文字 + 字符形态 + OCR 区域视觉 + 坐标

这是 PAM 对当前需求最有价值的部分。

论文中的 OCR token 不是只用 OCR 后的字符串，而是组合：

```text
OCR text embedding
+ character-level feature
+ OCR region visual feature
+ bounding box / location
```

其中：

- OCR 文本使用 fastText；
- OOV / morphology 可通过 subword 缓解；
- 额外使用 PHOC 字符特征增强字符级鲁棒性；
- OCR token 对应图片区域继续用 Faster R-CNN 提取视觉特征；
- bounding box 位置也编码进去；
- 最终线性投影到统一 embedding dimension。

论文最多处理：

```text
100 OCR tokens
```

Reference number 恰恰是 OCR 最容易遇到 OOV 的类型：

```text
126610LN
5711/1A-011
RM 11-03
IW500705
Q4148420
```

这些字符串不能依赖普通词表语义。

因此落地时应该进一步加强为：

```text
OCR raw string
+ character n-gram
+ normalized char pattern
+ bounding box
+ image role
+ OCR confidence
+ brand-specific regex hit
+ reference registry exact hit
```

而不是只依赖语言模型 embedding。

---

# 3. PAM 最关键的设计：受约束的 Pointer / Candidate Decoder

普通生成模型会自由生成 token。

对 reference 这是危险的：

```text
126610LN
```

如果模型“生成”成：

```text
126610LV
```

从自然语言角度只差两个字符，但从业务上已经是完全不同实体。

PAM 没有让 decoder 完全自由生成，而是让每个 decoding step 从三个受控来源选 token：

```text
1. Product Text token
2. OCR token
3. External Attribute Vocabulary token
```

然后比较 decoder state 与每个候选 token 的表示，选择得分最高者。

这个设计对当前系统应改造成更严格的 **Reference Candidate Pointer**：

```text
允许输出的 reference 只能来自：

A. 结构化 reference 字段中实际出现的候选
B. 标题/描述中实际出现的候选
C. OCR 中实际识别到的候选
D. 已知品牌 Reference Registry 中的 canonical entity
```

但 D 不能像自然语言生成那样凭模型猜测；必须满足可解释映射，例如：

```text
OCR raw = "12661OLN"      # OCR 把 0 识别成 O
registry candidate = "126610LN"

可以建立 fuzzy candidate
但不能直接 ACCEPT
只能进入 REVIEW / SECOND_PASS_OCR
```

也就是说：

> PAM 的 constrained decoding 值得保留；PAM 的“近似语言得分后直接选最大值”不应照搬到 reference 自动合并。

---

# 4. Dynamic Vocabulary：从“商品类别词表”升级为“品牌 Reference Registry”

PAM 发现，多个商品类别共用一个大输出词表效果不好。

因此它针对每个：

```text
(product category, attribute type)
```

维护动态词表，只让模型在当前类别相关的属性值词中选择。

论文实验中，去掉 dynamic vocabulary 后，Item Form 的结果从：

```text
PAM:                  P 91.3 / R 75.3 / F1 82.5
w/o dynamic vocab:    P 89.1 / R 69.5 / F1 78.1
```

说明缩小候选空间同时改善了 precision 和 recall。

对当前项目，应该把这个思想升级成：

```text
Dynamic Vocabulary
        ↓
Brand-aware Reference Registry
```

例如：

```text
brand_id = Rolex

允许候选：
126610LN
126610LV
124060
116610LN
116610LV
...
```

而不是让一个 Rolex 商品去全库几百万 reference 中搜索。

建议实体表：

```sql
reference_entity(
    reference_entity_id bigint primary key,
    brand_id bigint not null,
    canonical_reference varchar not null,
    reference_family varchar,
    normalized_pattern varchar,
    status varchar not null,
    provenance jsonb,
    unique (brand_id, canonical_reference)
)
```

注意：

> `canonical_reference` 的规范化必须是 **品牌感知** 的，不能对所有品牌统一“删掉全部空格、斜杠、连字符”。

因为某些符号可能是 reference 语义的一部分。

可以做安全的第一层：

```text
Unicode NFKC
trim
uppercase
统一全角/半角
统一明显 OCR 空白
```

进一步规范化规则必须进入：

```text
brand_reference_normalization_rule
```

并保留 raw value，保证可追溯。

---

# 5. PAM 中“edit distance”思想必须谨慎改造

PAM 在 token selection 时加入 edit-distance similarity，解决领域专有词无法被通用 embedding 正确理解的问题。

对于自然语言属性，这是合理的。

但对于腕表 reference，**edit distance 绝不能成为自动匹配的正证据**。

典型困难负例：

```text
126610LN
126610LV
```

两者非常相似，但业务上明确不同。

再如：

```text
5711/1A-010
5711/1A-011
```

只差最后一位，也可能代表不同版本。

所以建议把 fuzzy matching 限定为：

```text
候选召回 / OCR 纠错提示 / 人工复核排序
```

禁止：

```text
fuzzy_score > threshold -> 自动绑定 reference_entity
```

安全规则应是：

```text
fuzzy = 只产生 candidate
exact verified evidence = 才能产生 ACCEPT
```

这也是本项目和普通商品属性抽取最重要的区别。

---

# 6. Product Category Multi-task：在本项目里改造成 Brand / Product-role 条件化

PAM 进一步让 decoder 先输出 product category，再输出 attribute value：

```text
sunscreen -> spray
sunscreen -> alba botanica
```

类别作为 target prefix 参与 loss，迫使模型学到“类别和属性值之间的相关性”。

论文还实验了单独的 category classification auxiliary task，两种方式表现接近。

对于腕表系统，可以改成两类辅助任务：

```text
Task A: brand classification / normalization
Task B: product role classification
Task C: reference candidate extraction
```

其中 product role 建议至少包括：

```text
WATCH
STRAP
BOX
CARD
ACCESSORY
PART
OTHER
```

原因是二奢页面可能卖的是：

```text
“适配 Rolex 126610LN 的表带”
```

这时标题和图片都可能出现 `126610LN`，但它并不意味着当前商品 reference 是 `126610LN`。

因此实际约束可以写成：

```text
只有 product_role = WATCH
才允许 reference candidate 进入自动绑定主链路

STRAP / ACCESSORY / PART 中出现的 reference
默认视为 target / compatible reference
不得绑定当前商品
```

这可以和已经分析过的 `parts-distributor-sku-classifier` 的“编号角色分类”思想组合起来。

---

# 7. 论文实验结果给当前项目的一个重要警告

PAM 在 61,308 个样本、14 个商品类别上实验：

- 56,843 train / validation；
- 4,465 held-out testing；
- Item Form 大约 20 个值；
- Brand 超过 100 个值。

其核心结果：

### Item Form

```text
PAM text-only   P=94.5  R=60.1  F1=73.4
PAM multimodal  P=91.3  R=75.3  F1=82.5
```

多模态显著增加 recall，但 precision 反而下降。

### Brand

```text
PAM text-only   P=81.2  R=78.4  F1=79.8
PAM multimodal  P=86.6  R=83.5  F1=85.1
```

在 Brand 上，多模态三项都提升。

这说明：

> **“加图片/加 OCR”不会天然提升 precision。它提升的是信息量；最终 precision 是否增加取决于你如何使用这些信息。**

当前 Spec 不能使用“模型融合以后概率更高，所以自动 Match”的策略。

正确方式应该是：

```text
多模态增加 Evidence Coverage
硬规则负责 Final Acceptance
```

换句话说：

```text
OCR 是证据源，不是裁判。
视觉模型是辅助证据，不是 identity rule。
```

---

# 8. Ablation 对本项目的直接启示

PAM 对 Item Form 的模态消融：

```text
w/o text   P=79.9  R=63.4  F1=70.7
w/o image  P=88.7  R=72.1  F1=79.5
w/o OCR    P=82.0  R=69.4  F1=75.1
full PAM   P=91.3  R=75.3  F1=82.5
```

OCR 被拿掉后 precision 从 91.3 掉到 82.0，说明 OCR 不只是补 recall，在与其他模态联合时也可能帮助排除错误值。

论文同时指出：纯视觉 object feature 相比文本/OCR更容易有噪声，而且“外观相似”跨类别未必代表同一语义。

这对腕表更应该强化：

```text
图片外观 embedding：
- 可以用于“找可能需要 OCR 的图”
- 可以用于人工审核 UI 排序
- 可以用于冲突告警
- 不可以单独证明 reference 相同
```

尤其不要做：

```text
CLIP / VLM image similarity > 0.95
=> same reference
```

同系列不同 reference 的外观近似正是 false positive 高发区。

---

# 9. 推荐落地架构：Reference Evidence Pipeline

## 9.1 总体架构

```text
                   ┌────────────────────┐
                   │ 雷小安 / 腕表之家 / 奢当家 │
                   └─────────┬──────────┘
                             │
                    Raw Product Ingestion
                             │
              ┌──────────────┴──────────────┐
              │                             │
        Text Evidence                  Image Evidence
              │                             │
   structured ref / title / desc      image role detect
              │                             │
   conservative candidate regex           OCR
              │                             │
              └──────────────┬──────────────┘
                             │
                   Reference Candidates
                             │
             Brand-aware Normalization
                             │
                 Reference Registry Lookup
                             │
                     Evidence Ledger
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
       ACCEPT             CONFLICT           ABSTAIN
          │                  │                  │
reference_entity_id     review queue       review / wait
          │
       Exact Join
          │
  cross-source same-reference groups
```

这比 pairwise matching 更符合业务定义，因为复杂度从：

```text
N x M pair comparison
```

变成：

```text
record -> reference_entity
```

然后利用索引做 exact join。

---

# 10. Evidence Ledger：必须保留“为什么得到这个 reference”

建议不要在商品表只存一个：

```text
reference = 126610LN
```

而是保存所有证据。

```sql
reference_evidence(
    evidence_id bigint primary key,
    product_id bigint not null,
    brand_id bigint,

    evidence_type varchar not null,
    -- STRUCTURED_FIELD / TITLE / DESCRIPTION / OCR / MANUAL

    source_field varchar,
    image_id bigint,
    image_role varchar,
    bbox jsonb,

    raw_value varchar not null,
    normalized_value varchar,
    candidate_reference_entity_id bigint,

    extractor varchar,
    extractor_version varchar,
    confidence numeric,

    role varchar,
    -- SELF_REFERENCE / COMPATIBLE_REFERENCE / SKU / SERIAL / UNKNOWN

    created_at timestamp not null
)
```

最终 assignment 单独存：

```sql
product_reference_assignment(
    product_id bigint primary key,
    reference_entity_id bigint,
    status varchar not null,
    -- ACCEPTED / ABSTAIN / CONFLICT / REVIEW

    rule_id varchar,
    evidence_ids bigint[],
    resolver_version varchar,
    updated_at timestamp not null
)
```

这样每次出现误判，都可以直接回答：

```text
为什么这个商品被挂到 126610LN？
```

并快速定位是：

- OCR 错；
- 品牌归一化错；
- 标题抽取错；
- 编号角色错；
- registry 数据错；
- resolver rule 错。

这对“绝对不能误匹配”的系统比一个 end-to-end 黑盒分数更重要。

---

# 11. Precision-first Resolver：直接可落地的规则

建议 Resolver 不输出二分类概率，而输出：

```text
ACCEPTED
ABSTAIN
CONFLICT
REVIEW
```

## Rule A：结构化 Reference + Registry exact hit

```text
brand 已确认
AND structured reference exact canonical hit
AND role = SELF_REFERENCE
=> ACCEPTED
```

这是最强路径。

---

## Rule B：标题与 OCR 双路 exact agreement

```text
brand 已确认
AND title candidate == OCR candidate
AND 两者 canonicalize 后 exact same reference_entity
AND product_role = WATCH
=> ACCEPTED
```

这就是 PAM “文本 + OCR cross validation” 在本项目最直接的用法。

---

## Rule C：结构化字段与 OCR 一致

```text
structured candidate == OCR candidate
=> ACCEPTED
```

对于卖家结构化字段偶尔脏写的场景，图片形成独立确认。

---

## Rule D：单路证据

```text
只有 title 或只有 OCR 一个来源命中
=> ABSTAIN
```

即使模型置信度很高，也不要在第一版自动合并。

因为业务允许漏匹配。

---

## Rule E：高可信证据冲突

```text
structured = 126610LN
OCR        = 126610LV
```

直接：

```text
CONFLICT
```

不能用模型做加权平均，也不能“选分数高的那个”。

冲突商品应进入人工复核，或者等待后续新图片/新字段。

---

## Rule F：Fuzzy near-reference

```text
OCR = 12661OLN
registry nearest = 126610LN
```

只能：

```text
REVIEW / SECOND_PASS_OCR
```

禁止自动修正后 ACCEPT。

可以重新：

- 对原图做更高分辨率 OCR；
- crop 型号区域；
- 多 OCR engine 投票；
- 用 VLM 只做字符转录；
- 人工确认。

---

# 12. Reference Canonicalization：不要做危险的“字符串洗白”

推荐 canonicalization 分层执行。

## Level 0：安全标准化

可直接执行：

```text
Unicode NFKC
trim
uppercase
连续空白压缩
全角数字/字母转半角
```

## Level 1：品牌规则

例如某品牌确认：

```text
"RM11-03"
"RM 11-03"
```

是同一个 reference 表示形式，才允许映射。

规则必须带：

```text
brand_id
rule_version
source
approved_by
```

## Level 2：模糊候选

如：

```text
0 / O
1 / I
5 / S
8 / B
```

只能用于 OCR candidate recovery，不改变 canonical reference。

最终自动绑定仍要求：

```text
resolved candidate -> registry exact entity
```

---

# 13. OCR Pipeline 应针对“型号区域”而不是普通场景 OCR

PAM 的实验还比较了不同 OCR 组件：OCR 能识别到更多有效 token 时，下游端到端效果明显更好。

腕表数据建议建立 image role：

```text
CASEBACK
WARRANTY_CARD
HANG_TAG
DIAL
BUCKLE
BOX_LABEL
INVOICE
GENERAL_PRODUCT_IMAGE
UNKNOWN
```

优先级可以是：

```text
保卡型号栏 / 吊牌型号栏 / 盒标
    > 表背刻字
    > 商品主图可见文字
    > 背景文字
```

OCR 服务输出不要只保留字符串，而要保留：

```json
{
  "text": "126610LN",
  "confidence": 0.994,
  "bbox": [312, 221, 488, 260],
  "image_role": "WARRANTY_CARD",
  "crop_id": "...",
  "ocr_engine": "...",
  "ocr_version": "..."
}
```

Resolver 才能知道：

```text
同样的 126610LN
来自保卡型号栏
```

比：

```text
来自商品截图背景
```

更可信。

---

# 14. 不建议完全复刻 PAM 的地方

## 14.1 不要把 reference 当自然语言属性自由生成

reference 是 identifier，不是普通词。

生成模型最常见的问题是：

```text
hallucination
近似拼写
字符替换
```

这些对 reference 都是不可接受的。

所以：

```text
生成 -> 改成抽取 / 指针 / registry constrained retrieval
```

---

## 14.2 不要让视觉相似度成为自动 Match 依据

腕表同系列不同 reference 的 hard negative 非常多。

视觉可以做：

```text
supporting evidence
conflict detection
image role classification
OCR region detection
```

但不能做：

```text
visual similarity => same reference
```

---

## 14.3 不要把 edit distance 当 identity similarity

自然语言属性的拼写近似和 reference 的一字符差异不是一个问题。

对于 reference：

```text
一字符差异默认就是不同实体
```

除非存在品牌官方 alias / 格式规则。

---

## 14.4 不要以 F1 为主优化目标

PAM 是普通属性抽取任务，F1 是合理指标。

当前需求明确 precision-first，应把线上优化目标改为：

```text
1. False Merge Count
2. Accepted Assignment Precision
3. Auto-match Precision
4. Coverage / Recall
```

顺序不能反。

宁可：

```text
自动覆盖 40%，precision 极高
```

也不要：

```text
自动覆盖 95%，但偶尔把 126610LN / 126610LV 合并
```

---

# 15. 百万～千万级数据如何跑

主链路不需要做三源全量 pairwise 比较。

每条新增/更新记录只需运行一次 reference resolution：

```text
O(N)
```

然后：

```sql
SELECT ...
FROM product_reference_assignment a
JOIN product_reference_assignment b
  ON a.reference_entity_id = b.reference_entity_id
WHERE a.source <> b.source
  AND a.status = 'ACCEPTED'
  AND b.status = 'ACCEPTED';
```

工程组件可以拆成：

```text
Ingestion Service
   ↓
Text Candidate Extractor
   ↓
Image/OCR Worker Pool
   ↓
Evidence Store
   ↓
Reference Resolver
   ↓
Reference Entity Store
   ↓
Cross-source Join Materializer
```

持久化建议：

```text
关系数据库：
- reference_entity
- assignment
- resolver rule/version

对象存储：
- 原始图片
- OCR crop

分析型存储：
- 大规模 evidence / audit log
- 规则效果统计
```

不建议把向量数据库放在主一致性路径上。

向量检索如果使用，只做：

```text
召回候选 / review assistant
```

最后仍回到 exact canonical reference。

---

# 16. 增量更新策略

三个来源持续更新时，不应每天全量重跑。

建议每条商品记录维护：

```text
content_hash
text_hash
image_hash_set
extractor_version
resolver_version
```

触发条件：

```text
文本变化 -> 重跑 text evidence
图片新增/变化 -> 只重跑新增图片 OCR
reference registry 更新 -> 只重跑受影响 brand
resolver 规则更新 -> 重算 assignment，不必重做 OCR
OCR 模型升级 -> 可按品牌/冲突队列分批回灌
```

这样 extraction 和 resolution 解耦。

这也是 Evidence Ledger 的另一个好处：

> 模型升级时不需要重新抓数据，只需基于历史 evidence 或重跑必要模块。

---

# 17. 几百对黄金标签怎么用最划算

Spec 允许人工标注几百对。

不建议把这几百对主要拿去训练一个端到端 Match / NoMatch 模型。

更值得标：

## 17.1 Reference candidate role

例如：

```text
126610LN -> SELF_REFERENCE
126610LN -> COMPATIBLE_REFERENCE
SDJ-882931 -> PLATFORM_SKU
8R35-01A0 -> MOVEMENT / CASE CODE
```

## 17.2 OCR hard cases

重点标：

```text
0/O
1/I
5/S
8/B
破折号/斜杠
反光
弧形刻字
低分辨率
```

## 17.3 Same-family hard negatives

黄金集必须故意包含：

```text
126610LN vs 126610LV
116610LN vs 126610LN
5711/1A-010 vs 5711/1A-011
```

不要随机抽样，因为随机 negative 太容易，无法验证“绝不误匹配”。

## 17.4 Conflict resolution

人工重点处理：

```text
structured ref != title ref
structured ref != OCR ref
title ref != OCR ref
```

这些样本对提升系统 precision 的价值远高于随机商品。

---

# 18. 推荐评测集

至少拆成以下桶：

```text
A. structured reference clean
B. reference only in title
C. reference only in OCR
D. title + OCR agree
E. title + OCR conflict
F. structured + OCR conflict
G. same-family near reference hard negative
H. accessory mentions watch reference
I. seller SKU looks like reference
J. unseen / long-tail reference
```

每个桶都统计：

```text
assignment precision
assignment coverage
conflict rate
abstain rate
false merge count
```

线上真正重要的是：

```text
accepted_reference_precision
```

以及最终：

```text
cross_source_auto_match_precision
```

而不是整体 F1。

---

# 19. 可以直接实现的 Resolver 伪代码

```python
def resolve_reference(product, evidences, registry):
    # 1. 非腕表主体商品，默认不自动绑定腕表 reference
    if product.role != "WATCH":
        return Assignment(status="ABSTAIN", reason="NON_WATCH_ROLE")

    # 2. 只保留当前品牌下 registry 可识别的候选
    candidates = []
    for e in evidences:
        if e.role != "SELF_REFERENCE":
            continue
        entity = registry.exact_lookup(
            brand_id=product.brand_id,
            normalized_reference=e.normalized_value,
        )
        if entity:
            candidates.append((entity, e))

    if not candidates:
        return Assignment(status="ABSTAIN", reason="NO_EXACT_REFERENCE_EVIDENCE")

    by_entity = group_by_reference_entity(candidates)

    # 3. 两个高可信 canonical reference 冲突，直接熔断
    high_trust_entities = {
        entity_id
        for entity_id, group in by_entity.items()
        if has_high_trust_evidence(group)
    }

    if len(high_trust_entities) > 1:
        return Assignment(status="CONFLICT", reason="HIGH_TRUST_REFERENCE_CONFLICT")

    # 4. 结构化 reference 独立确认
    for entity_id, group in by_entity.items():
        if has_verified_structured_ref(group):
            return Assignment(
                status="ACCEPTED",
                reference_entity_id=entity_id,
                reason="VERIFIED_STRUCTURED_REFERENCE",
                evidence_ids=evidence_ids(group),
            )

    # 5. 文本 + OCR 两个独立模态 exact agreement
    for entity_id, group in by_entity.items():
        if has_title_or_description(group) and has_high_quality_ocr(group):
            return Assignment(
                status="ACCEPTED",
                reference_entity_id=entity_id,
                reason="TEXT_OCR_EXACT_AGREEMENT",
                evidence_ids=evidence_ids(group),
            )

    # 6. 单路证据即使模型分高也拒识
    return Assignment(status="ABSTAIN", reason="INSUFFICIENT_INDEPENDENT_EVIDENCE")
```

重点不在伪代码本身，而在这三条原则：

```text
exact > fuzzy
agreement > score
abstain > risky merge
```

---

# 20. 与已经分析方案的组合方式

PAM 不应该取代此前方案，而应该放到更明确的位置。

## 与 parts-distributor-sku-classifier

```text
PAM：从文本 / OCR 找编号
SKU classifier：判断编号角色
```

组合后解决：

```text
“找到了 126610LN，但它到底是不是当前商品 reference？”
```

---

## 与 Using LLMs for Extraction and Normalization

```text
LLM / extractor：找候选、处理复杂语境
PAM OCR：补图片证据
Reference Registry：最终 canonicalize
```

LLM 不拥有最终写入 canonical reference 的权限。

---

## 与 DeepBlocker

当 reference 无法解析时，DeepBlocker 可以生成疑难候选供 review。

但：

```text
blocking candidate != match
```

如果最终没有 reference 硬证据，仍然 abstain。

---

## 与 Confidence Classifier

Confidence / selective classification 可以用于：

```text
哪些 evidence 值得自动进入 ACCEPT
```

但最好的 confidence features 应以：

```text
证据类型
证据独立性
OCR image role
registry exact hit
冲突数量
```

为主，而不是通用语义相似度。

---

## 与 TransClean

PAM 在前：

```text
找 reference evidence
```

TransClean 在后：

```text
审计已经形成的跨源 reference component 是否出现异常冲突
```

推荐完整链路：

```text
PAM-style multimodal evidence extraction
        ↓
role classification
        ↓
reference registry exact resolution
        ↓
exact join
        ↓
TransClean-style graph safety audit
```

形成双保险。

---

# 21. 第一版可以不训练大模型

虽然 PAM 是深度多模态模型，但当前项目第一版完全可以先做一个更可控的工程版本：

```text
结构化字段抽取
+
品牌感知 regex
+
OCR
+
图片 role
+
Reference Registry
+
exact evidence agreement rules
```

先测：

```text
仅靠这些硬证据能覆盖多少商品？
```

因为业务允许漏匹配，如果这一版已经达到较高 coverage，就没必要让大模型进入自动决策路径。

模型真正应该投入的区域是：

```text
ABSTAIN / CONFLICT queue
```

用来：

- 找 OCR candidate；
- 判断编号 role；
- 选择更值得人工看的图片；
- 给人工复核提供候选；
- 学习品牌特定格式。

这能把 ML 风险限制在“辅助发现证据”，而不是“直接合并实体”。

---

# 22. 建议的数据流状态机

每条商品记录可以维护：

```text
RAW
  ↓
EVIDENCE_EXTRACTED
  ↓
REFERENCE_RESOLVED
  ├─ ACCEPTED
  ├─ ABSTAIN
  ├─ CONFLICT
  └─ REVIEW
```

只有：

```text
REFERENCE_RESOLVED.ACCEPTED
```

允许进入自动跨源 join。

再进一步，`ACCEPTED` 可以记录 acceptance tier：

```text
T0: verified structured reference
T1: structured + OCR exact agreement
T2: title + OCR exact agreement
T3: manual verified
```

生产上可以先只发布：

```text
T0 / T1 / T3
```

等离线验证充分后再考虑 T2。

这比一个统一 `confidence=0.997` 更容易治理。

---

# 23. 最值得从 PAM 继承的四个东西

最终可以把 PAM 对当前需求的价值压缩成四点。

## 1. OCR 是一等公民

不要把图片只变成视觉 embedding。

对于腕表 reference，图片里的文字往往比“图片看起来像什么表”更重要。

## 2. 多模态的目标是 cross-validation

图片不是为了增加一个相似度分数，而是验证文本里的 reference 是否真的属于当前商品。

## 3. 输出空间必须受控

从自由 generation 改成：

```text
text/OCR candidate + brand reference registry
```

## 4. 条件化候选空间

从 PAM 的：

```text
category-specific vocabulary
```

改成：

```text
brand-specific reference registry
```

这既降低搜索空间，也降低跨品牌误匹配风险。

---

# 24. 最终推荐方案

如果直接为当前 Spec 落地，我建议把 PAM 的思想实现为以下生产架构：

```text
                    Reference Registry
                           ▲
                           │ exact lookup
                           │
雷小安 ─┐                  │
腕表之家 ├─> Product Evidence Pipeline ──> Precision Resolver
奢当家 ─┘       │                      │
               ├─ structured ref       ├─ ACCEPT
               ├─ title / desc         ├─ ABSTAIN
               ├─ OCR                  ├─ CONFLICT
               ├─ image role           └─ REVIEW
               └─ number role
                                              │
                                              ▼
                                     reference_entity_id
                                              │
                                              ▼
                                     exact cross-source join
                                              │
                                              ▼
                                      graph safety audit
```

并坚持以下不可越权原则：

```text
1. visual similarity 永远不能直接判同款
2. fuzzy reference 永远不能直接 ACCEPT
3. LLM 生成的 reference 永远不能直接写 canonical entity
4. 任意高可信 reference 冲突必须熔断
5. 只有 canonical reference exact equality 才能自动跨源合并
6. 证据不足就 abstain
```

这套方案比直接复刻 PAM 更适合二奢/腕表，因为它把 PAM 擅长的“从图文中发现属性证据”保留下来，同时把它不适合 identifier resolution 的自由度全部收紧。

**一句话总结：**

> PAM 的真正价值，是证明“图片 OCR + 文本 + 受约束候选空间”可以显著改善电商属性抽取；在当前三源腕表系统里，应把它改造成 `Reference Evidence Layer`，让 OCR 帮我们找到和验证 reference，但最终只有 `brand-aware canonical reference exact match` 才拥有实体合并权。