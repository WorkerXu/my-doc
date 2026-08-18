# Using LLMs for the Extraction and Normalization of Product Attribute Values：面向二奢腕表 Reference-First 匹配系统的实现分析

## 1. 分析对象与结论

本次选择的对象是：

- 论文：**Using LLMs for the Extraction and Normalization of Product Attribute Values**，Alexander Brinkmann、Nick Baumann、Christian Bizer，ADBIS 2024 / arXiv:2403.02130
- 论文：https://arxiv.org/abs/2403.02130
- 官方实现：https://github.com/wbsg-uni-mannheim/wdc-pave
- WDC-PAVE 项目页：https://webdatacommons.org/structureddata/wdc-pave/

在分析前已检查 `奢侈品调研/b`，该论文/项目此前未被我分析；已有的 Ameli、Ditto、DeepBlocker、pyJedAI、TransClean、GraLMatch、parts-distributor-sku-classifier 等结果均已排除。

### 对当前 Spec 最重要的结论

当前 Spec 把“同一个商品”定义为**同一 reference number / 型号**。在这个定义下，真正困难的问题并不是一个通用的 pairwise Entity Matching 分类器，而是：

1. 从三个来源稀疏、异构、噪声文本中**找出真正属于当前商品的 reference**；
2. 对 reference 做**不丢语义的规范化**；
3. 判断这个编号到底是品牌 reference、平台 SKU、店铺货号、序列号还是配件适配型号；
4. 当证据不足或冲突时**拒识**；
5. 只有 reference 被高可信解析后，才使用 `(brand_id, reference_id)` 做严格等值归并。

因此，WDC-PAVE 最值得借鉴的不是“让 GPT-4 直接决定两个商品是不是同一个”，而是它的这套工程结构：

> **结构化 Schema + 属性级规范化规则 + 相似案例检索式 few-shot + JSON 强约束输出 + n/a 拒识 + exact-match 评估。**

把这套方法改造成腕表 reference 专用抽取器后，可以直接嵌到百万到千万级数据流水线中；最终匹配层则退化为便宜、确定、易审计的等值 Join。

我的建议是实现一个 **Reference-First PAVE（下文简称 RF-PAVE）**：

```text
雷小安 / 腕表之家 / 奢当家
        │
        ▼
原始记录 + 图片落库
        │
        ▼
品牌规范化 ──► 编号角色识别
        │
        ▼
Reference 候选抽取
  ├─ 独立字段
  ├─ 品牌规则/正则
  ├─ LLM PAVE fallback
  └─ OCR/VLM 文字证据
        │
        ▼
品牌专用、可逆规范化
        │
        ▼
Reference Catalog 校验 + 冲突检测
        │
        ├─ ACCEPT：形成 canonical reference_id
        ├─ REVIEW：人工复核
        └─ ABSTAIN：不匹配
        │
        ▼
严格键：(brand_id, reference_id)
        │
        ▼
跨来源聚合 / 增量更新
```

这比“先做海量 pairwise 候选，再训练一个 matcher”更符合当前 Spec 的 precision-first 约束。

---

## 2. 论文解决了什么问题

论文研究的是 Product Attribute Value Extraction and Normalization（PAVE）：从商品标题和描述里抽取结构化属性值，并把不同表述规范到统一形式。

一个与当前需求几乎同构的论文示例是：

```text
HP – 6280-59-B21 - 3TB 3G SATA 7.2K rpm
LFF (3.5-inch) Midline 1yr Hard Drive
```

抽取后：

```json
{
  "Manufacturer": "HP",
  "Product Type": "Hard Drive",
  "Rotational Speed": "7.2K",
  "Part Number": "6280-59-B21"
}
```

规范化后：

```json
{
  "Manufacturer": "Hewlett-Packard",
  "Product Type": "Storage Solutions",
  "Rotational Speed": "7200",
  "Part Number": "628059B21"
}
```

这里的 **Part Number** 与腕表的 **Reference Number** 高度对应：两者都是决定商品型号身份的字母数字标识符，而且经常混入连字符、空格、括号或营销文本。

论文的 WDC-PAVE benchmark 包含：

- 565 个商品 offer；
- 来自 59 个不同网站；
- 4,687 个经过人工核验的 attribute-value pair；
- 37 个属性；
- 45% 的属性在某个商品上是 `n/a`；
- 同时提供“原始抽取值”和“规范化值”。

这个数据设定对我们的意义很大：**字段缺失是正常情况，`n/a` 不是失败，而是系统必须拥有的合法输出。** 当前 Spec 的“允许漏匹配、绝不能误匹配”，本质上要求系统把 abstention 做成一级能力。

---

## 3. 论文/代码的技术架构

### 3.1 六块式 Prompt 结构

论文把 Prompt 拆成最多六个 building block：

1. Role Description；
2. Task Description；
3. Task Input；
4. Task Output；
5. Demonstration Input；
6. Demonstration Output。

其中 Role Description 不只是普通自然语言，而会注入目标 Schema、属性解释和 example values。代码通过 Pydantic 动态创建模型，再把模型转换成 JSON Schema，从而约束模型“应该抽哪些字段、字段是什么类型”。

对腕表任务不应给模型几十个通用属性，而应使用一个极窄 Schema，例如：

```json
{
  "brand": "Rolex",
  "reference_candidates": [
    {
      "raw_value": "126610LN",
      "source_field": "title",
      "evidence_text": "劳力士潜航者 126610LN 全套",
      "identifier_role": "brand_reference"
    }
  ],
  "abstain": false
}
```

窄 Schema 可以显著减少模型自由发挥空间。

### 3.2 代码中的强结构化输出

官方实现 `prompts/08_extraction_with_normalization/8_few_shot_extraction_with_normalization.py` 的关键路径是：

1. 从任务定义加载已知属性；
2. 创建 Pydantic 模型；
3. 构造 ChatPromptTemplate；
4. 调用 LLM，`temperature=0`；
5. 尝试直接 `json.loads`；
6. 用 Pydantic 校验字段；
7. JSON 不合法时再走修复解析；
8. 仍失败则不产出有效 prediction；
9. 最后按 exact-match 计算 precision / recall / F1。

这一点非常适合当前需求：**LLM 输出不是事实，只是一个待验证的结构化候选。**

对于 RF-PAVE，我建议再加三层硬校验：

```text
LLM JSON
  │
  ├─ Schema Validation
  │
  ├─ Evidence Span Validation
  │     raw_value 必须能在原始 title/description 中定位
  │
  └─ Reference Catalog Validation
        规范化后的值必须符合品牌规则，最好能映射到 catalog ref_id
```

任何一层失败都不能自动进入匹配键。

### 3.3 类别感知的相似案例检索

代码 `pieutils/search.py` 中实现了 `CategoryAwareSemanticSimilarityExampleSelector`：

- 每个 category 单独构建 FAISS 向量索引；
- 使用 OpenAI Embeddings 对训练样本编码；
- 对新商品检索语义最接近的 top-k 示例；
- 把这些示例动态加入 few-shot prompt；
- 还支持 `force_from_different_website`，即尽量从不同网站选择 demonstration，避免来源模板泄漏。

论文也明确说明，few-shot demonstration 是通过 embedding + cosine similarity 选取最相似的训练商品，而不是固定随机样本。

这可以直接改成：

```text
原实现 category
      ↓
RF-PAVE 检索分区 = brand_id + product_type + identifier_context
```

例如 Rolex 的 title 只从 Rolex 已标注样本中取 demonstration；如果是“表带/配件/盒证”场景，则从 identifier role hard cases 中取样，不要从 Omega、Cartier 的规则体系中检索示例。

进一步可加入来源约束：

```text
输入来自 雷小安
few-shot 优先从 腕表之家 + 奢当家 取相似示例
```

这能降低模型只学习某个平台固定文案格式的风险。

### 3.4 规范化不是一个通用函数，而是属性级规则

`pieutils/pieutils.py` 的 `get_normalization_guidelines_from_csv` 会从配置表中按 category / attribute 加载 normalization instruction。论文把规范化拆成四类：

- Name Expansion；
- Generalization；
- Unit Conversion；
- String Wrangling。

对于当前需求，真正相关的是 **String Wrangling**，但必须做一项关键改造：

> 论文对 Part Number 可以把 `6280-59-B21` 直接变成 `628059B21`；腕表 reference 不应该默认做全局“删除所有非字母数字字符”的有损规范化。

因为某些品牌/系列中：

- 后缀代表材质、表盘或版本；
- 点号、斜杠、短横线的位置可能参与官方编号语义；
- `126610LN` 与 `126610LV` 只有很小差别但绝不能合并；
- “配件适配 126610LN”里的 reference 不属于当前售卖主体。

所以 RF-PAVE 需要两层规范化：

```text
raw_reference
    │
    ├─ safe_normalized
    │   只做 Unicode、大小写、全半角、空白、明确等价符号转换
    │
    └─ catalog_canonical
        只有命中品牌规则/别名表后才能映射为官方 canonical ref
```

禁止使用一个全品牌共享的 `re.sub('[^A-Za-z0-9]', '')` 作为最终 identity key。

---

## 4. 实验结果对方案的启示

论文的结果不应该直接被解释成“GPT-4 91% F1，因此可以直接自动匹配”。恰恰相反，这组结果说明 LLM 必须被放在受控的位置。

### 4.1 Direct Extraction

论文中 GPT-4：

- zero-shot：74.40 F1；
- 10 个 example values：79.65 F1；
- 10 example values + 3 demonstrations：88.94 F1；
- 10 example values + 10 demonstrations：90.54 F1。

论文最佳 PLM baseline AVEQA 为 80.83 F1。

结论是：**示例和语义相近 demonstration 对小样本很有效，但 90% 左右的 F1 对“绝不能误匹配”仍远远不够。**

### 4.2 Extraction + Normalization

GPT-4 在 10 example values + 5 demonstrations 时达到约 91.32 F1。值得注意的是 5 个 demonstrations 已基本达到 10 个 demonstrations 的效果，而 prompt 更短。

对线上系统的启示是：

- 不要把整套标注集塞进 prompt；
- 每次动态检索 3～5 个最相关 hard cases 更经济；
- few-shot 的职责是帮助提取，而不是提升最终匹配权限。

### 4.3 先抽取、后规范化通常更稳

论文的 normalization-only 任务明显优于“边抽取边规范化”，GPT-4 在 10 examples + 5 demonstrations 时约 96.06 F1，10 demonstrations 约 96.21 F1。

论文还指出，当 value 已经先被抽出来后，单位转换等复杂 normalization 的表现会明显变好。

因此对 RF-PAVE，我不建议一个 Prompt 同时做：

```text
读全文 → 判断哪个编号是 reference → 改写成 canonical → 判是否同款
```

而要拆成多个可审计阶段：

```text
候选抽取 → role 分类 → 安全规范化 → catalog 映射 → final gate
```

每一阶段都有可保存的中间证据。

---

## 5. 当前 Spec 的问题应该重新建模

Spec 给出的关键约束是：

- 100 万～1000 万级；
- 三个来源持续增量；
- reference 可能是独立字段，也可能埋在标题；
- precision 极端优先，允许漏；
- 有图片；
- 可标几百对黄金样本。

如果“同一个商品”严格定义成同一 reference，那么 pairwise 相似度模型其实应该从主路径里拿掉。

### 5.1 主问题不是 Match，而是 Resolve Reference

定义：

```text
resolve(record) ->
  ACCEPT(brand_id, reference_id, evidence)
  REVIEW(candidates, conflicts)
  ABSTAIN(reason)
```

最终匹配变成：

```text
record_a == record_b
iff
record_a.status == ACCEPT
and record_b.status == ACCEPT
and record_a.brand_id == record_b.brand_id
and record_a.reference_id == record_b.reference_id
```

不需要一个概率 matcher 再决定一次。

### 5.2 为什么一定要包含 brand_id

不同品牌可能存在相同或高度相似的型号字符串，因此不要把 reference string 当全局主键。

推荐最终 identity key：

```text
(brand_id, catalog_reference_id)
```

如果 catalog 还没建立完整，则退化成：

```text
(brand_id, safe_normalized_reference)
```

但第二种只能在严格规则通过后使用。

---

## 6. 可直接落地的 RF-PAVE 架构

## 6.1 组件拆分

### A. Source Adapter

每个来源统一映射为：

```json
{
  "source": "leixiaoan",
  "source_item_id": "...",
  "brand_raw": "劳力士",
  "reference_field_raw": null,
  "title": "劳力士 潜航者 126610LN ...",
  "description": "...",
  "category": "watch",
  "image_urls": [],
  "updated_at": "...",
  "payload_hash": "..."
}
```

必须保存原文，后续所有抽取都能追溯到 raw payload。

### B. Brand Resolver

先把：

```text
劳力士 / ROLEX / Rolex / rolex
```

统一到稳定 `brand_id`。

品牌不确定时不要进入 reference 自动归并。

### C. Deterministic Reference Candidate Extractor

优先级：

1. 平台独立 reference 字段；
2. 品牌专用正则；
3. 通用 identifier token 候选；
4. LLM fallback。

独立字段并不等于可信：还要判断它是不是平台 SKU/内部货号。

### D. LLM PAVE Fallback

只处理以下情况：

- 没有独立字段；
- 规则找到多个候选；
- 标题中 reference 和适配型号混杂；
- 规则无法确定 identifier role。

LLM 不生成 canonical reference，只返回**原文中真实存在的候选和证据 span**。

### E. Image Evidence Extractor

图片不要先做 CLIP 相似度匹配，因为同系列不同 reference 外观可能极其接近。

优先做：

1. 表背 OCR；
2. 保卡 OCR；
3. 吊牌 OCR；
4. 盒标 OCR。

输出仍然只是 candidate evidence。

如果 OCR reference 与文本 reference 冲突，状态必须从 ACCEPT 降为 REVIEW，而不是做“多数投票自动合并”。

### F. Canonicalizer

输入：

```text
brand_id + raw_reference
```

输出：

```json
{
  "safe_normalized": "126610LN",
  "canonical_reference": "126610LN",
  "reference_id": "rolex:126610LN",
  "rule_id": "rolex-v3",
  "catalog_hit": true
}
```

规范化规则必须 versioned。

### G. Decision Gate

最终 gate 是系统最重要的模块。

推荐状态机：

```text
EXTRACTED
  │
  ├─ candidate conflict ──────────────► REVIEW
  │
  ├─ no candidate ────────────────────► ABSTAIN
  │
  ├─ identifier_role != brand_reference ► ABSTAIN/REVIEW
  │
  ├─ brand unknown ───────────────────► REVIEW
  │
  ├─ canonicalization unsafe ─────────► REVIEW
  │
  ├─ catalog invalid ─────────────────► REVIEW
  │
  └─ all hard gates pass ─────────────► ACCEPT
```

### H. Reference Key Index

只对 ACCEPT 的记录写：

```text
brand_id + reference_id -> cluster_id
```

新增记录是 O(1)/O(log N) 查索引，不再做 N×M pairwise comparison。

---

## 6.2 推荐的数据表

### `product_record`

```sql
CREATE TABLE product_record (
  id BIGSERIAL PRIMARY KEY,
  source TEXT NOT NULL,
  source_item_id TEXT NOT NULL,
  brand_raw TEXT,
  title TEXT,
  description TEXT,
  payload JSONB NOT NULL,
  payload_hash TEXT NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL,
  UNIQUE (source, source_item_id)
);
```

### `reference_evidence`

```sql
CREATE TABLE reference_evidence (
  id BIGSERIAL PRIMARY KEY,
  record_id BIGINT NOT NULL REFERENCES product_record(id),
  extractor TEXT NOT NULL,
  source_location TEXT NOT NULL,
  raw_value TEXT NOT NULL,
  evidence_text TEXT,
  identifier_role TEXT,
  extractor_version TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### `reference_resolution`

```sql
CREATE TABLE reference_resolution (
  record_id BIGINT PRIMARY KEY REFERENCES product_record(id),
  brand_id BIGINT,
  raw_reference TEXT,
  safe_normalized_reference TEXT,
  reference_id BIGINT,
  status TEXT NOT NULL,
  decision_reason TEXT NOT NULL,
  canonicalizer_version TEXT NOT NULL,
  catalog_version TEXT,
  resolved_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_resolution_identity
ON reference_resolution (brand_id, reference_id)
WHERE status = 'ACCEPT';
```

### `reference_catalog`

```sql
CREATE TABLE reference_catalog (
  reference_id BIGSERIAL PRIMARY KEY,
  brand_id BIGINT NOT NULL,
  canonical_reference TEXT NOT NULL,
  aliases JSONB NOT NULL DEFAULT '[]',
  pattern_version TEXT,
  verified BOOLEAN NOT NULL DEFAULT false,
  UNIQUE (brand_id, canonical_reference)
);
```

---

## 6.3 LLM 输出 Schema

建议不要让模型直接返回一个字符串，而是要求：

```json
{
  "brand": "Rolex",
  "candidates": [
    {
      "raw_value": "126610LN",
      "identifier_role": "brand_reference",
      "source_field": "title",
      "evidence_text": "潜航者 126610LN 全套",
      "reason": "型号紧邻系列名，且不是库存号/序列号"
    }
  ],
  "has_conflict": false,
  "abstain": false
}
```

服务端必须再执行：

```python
assert candidate.raw_value in normalized_source_text
assert candidate.identifier_role == "brand_reference"
```

不允许模型通过“知识记忆”补出一个原文里不存在的型号并直接被接受。

对于 OCR 候选，`source_field` 必须是明确的 `image_ocr:<image_id>`，也不能假装来自文本。

---

## 7. 自动接受策略：宁可不匹配，也不能错合并

建议至少定义四档证据，但只有最高档进入自动 ACCEPT。

### Tier A：可自动接受

满足全部：

- brand 已确定；
- reference 候选唯一；
- candidate 是 `brand_reference`；
- 值来自结构化字段或原始文本真实 span；
- 品牌专用 safe normalization 成功；
- 命中已验证 reference catalog；
- 不存在文本/OCR/其他字段冲突。

### Tier B：高可信但建议先 REVIEW

例如：

- title 中唯一 reference；
- 品牌规则完全匹配；
- 尚未进入 catalog；
- 没有其他冲突证据。

人工确认后可以自动回灌到 catalog/alias table。

### Tier C：仅 LLM 找到

- 规则没找到；
- LLM 在原文中定位到 candidate；
- 尚缺 catalog 支持。

默认 REVIEW，不自动合并。

### Tier D：仅图片视觉相似 / OCR 模糊候选

默认 ABSTAIN 或 REVIEW，永不因为“看起来像同款”自动合并。

---

## 8. Reference 规范化的安全策略

规范化是此系统最容易制造 silent false positive 的地方。

### 8.1 允许的通用安全操作

可直接做：

- Unicode NFKC；
- 全角转半角；
- trim；
- 连续空白折叠；
- 拉丁字母统一大写；
- 明确的 Unicode 横线变体统一。

### 8.2 不应该全局做的操作

不能无条件：

- 删除所有 `/`；
- 删除所有 `.`；
- 删除所有 `-`；
- 删除全部后缀；
- 模糊编辑距离纠错；
- 用相邻 reference 自动补全。

### 8.3 品牌专用 alias

正确方式：

```text
raw A ─┐
raw B ─┼─► verified alias rule ─► canonical reference_id
raw C ─┘
```

alias 必须带：

- brand_id；
- rule/source；
- 人工或权威来源验证状态；
- 生效版本。

这样规则升级可以重放，发生错误也可以审计是哪条 alias 导致。

---

## 9. Few-shot 黄金样本怎么用最划算

Spec 允许人工标注几百对，应该优先标“最可能制造 false positive”的 hard cases，而不是随机均匀采样。

建议 300～500 个高价值样本分配为：

### 9.1 Reference 抽取 hard cases

覆盖：

- 同标题出现多个编号；
- reference + 库存 SKU；
- reference + 序列号；
- 本体 + 表带适配型号；
- 本体 + 盒证编号；
- 中文/英文混排；
- 特殊分隔符；
- 同系列相邻 reference。

### 9.2 Identifier role hard negatives

特别标：

```text
brand_reference
platform_sku
seller_sku
serial_number
accessory_compatible_reference
unknown_identifier
```

### 9.3 跨源检索示例

few-shot selector 使用：

```text
brand_id 先过滤
↓
embedding 检索 title/description 语义近邻
↓
排除同来源模板
↓
选 3～5 个最相似 hard cases
```

这正是 WDC-PAVE `CategoryAwareSemanticSimilarityExampleSelector` 的生产化改造版。

### 9.4 样本优先用于“抽取器”，不是 pair matcher

只要 reference 能被可靠 resolve，两个商品是否匹配已经由严格 key 决定。把标注预算拿去训练通用 pairwise matcher 反而浪费，并可能引入概率误合并。

---

## 10. 评估指标也要改成 precision-first

不要以整体 F1 作为上线主指标。

至少拆成：

```text
1. Reference candidate recall
2. Canonical reference exact accuracy
3. Auto-accept precision   <-- 第一主指标
4. Auto-accept coverage
5. Review rate
6. Abstain rate
7. Conflict detection recall
8. 每个品牌/来源的 false-positive 数
```

### 10.1 自动接受集必须单独统计

例如：

```text
所有记录 100,000
  ├─ ACCEPT 55,000
  ├─ REVIEW 10,000
  └─ ABSTAIN 35,000
```

系统应优先优化：

```text
precision(ACCEPT) -> 逼近 100%
```

而不是为了覆盖率把 REVIEW/ABSTAIN 强行塞进 ACCEPT。

### 10.2 “测试集 0 个错误”不等于真正保证零误匹配

如果只测几百条，即使观察到 0 个 false positive，也不足以统计上证明生产流量里的错误率极低。

可使用简单的 rule-of-three 做 sanity check：若 N 个独立 accepted case 中 0 error，95% 置信下误差率上界约为 `3/N`。

这意味着想把误差率上界压到 0.1% 附近，需要大约 3000 个零错误 accepted 样本。当前“几百对黄金标签”更适合做模型/规则开发，生产上线后还应持续抽样审计。

---

## 11. 百万～千万规模下如何跑

因为最终不做 pairwise 全连接比较，规模问题会简单很多。

### 11.1 历史回填

推荐：

```text
Object Storage / DB
  │
  ▼
批量 Worker（Ray / Spark / Python multiprocessing 均可）
  │
  ├─ brand resolve
  ├─ deterministic extraction
  ├─ only hard cases -> LLM queue
  ├─ OCR queue
  └─ canonicalize + gate
  │
  ▼
PostgreSQL reference index
```

真正需要 LLM 的记录应该被压缩到少数 hard cases，不能对 1000 万条记录全部调用大模型。

### 11.2 增量处理

每条记录保存：

```text
payload_hash
extractor_version
canonicalizer_version
catalog_version
```

只有以下变化才重跑：

- 商品文本/独立字段变化；
- 图片变化；
- 规则版本变化；
- catalog 增补了相关品牌/reference；
- 模型版本显式升级。

### 11.3 数据库选型

对 100 万～1000 万记录，只要 final lookup 是组合 B-tree key，PostgreSQL 足够作为第一版 truth store，不必为了这个量级先引入复杂分布式图数据库。

可以使用：

- PostgreSQL：主数据、reference catalog、最终 identity index；
- pgvector / FAISS：few-shot 示例检索；
- S3/OSS：图片和 raw payload；
- Redis / queue：异步 OCR/LLM；
- Kafka：只有当持续增量吞吐已经需要时再引入。

---

## 12. 推荐的核心决策代码

第一版甚至可以非常明确地写成：

```python
from dataclasses import dataclass
from enum import Enum

class Status(str, Enum):
    ACCEPT = "ACCEPT"
    REVIEW = "REVIEW"
    ABSTAIN = "ABSTAIN"

@dataclass
class Resolution:
    status: Status
    brand_id: str | None = None
    reference_id: str | None = None
    reason: str = ""


def resolve_reference(record, catalog, extractors):
    brand = resolve_brand(record)
    if not brand.is_verified:
        return Resolution(Status.REVIEW, reason="brand_unresolved")

    evidences = []

    # 1. 强证据优先
    evidences += extract_structured_reference(record, brand)
    evidences += extract_by_brand_rules(record, brand)

    # 2. 只有不唯一时才调用 LLM
    if not has_unique_brand_reference(evidences):
        evidences += extract_with_llm_pave(record, brand)

    # 3. 图片只补证据，不直接决定 identity
    if needs_image_evidence(evidences):
        evidences += extract_image_ocr(record.images, brand)

    # 4. 冲突直接拦截
    candidates = unique_brand_reference_candidates(evidences)
    if len(candidates) == 0:
        return Resolution(Status.ABSTAIN, reason="no_reference")
    if len(candidates) > 1:
        return Resolution(Status.REVIEW, reason="reference_conflict")

    raw_ref = candidates[0]

    # 5. 只允许品牌专用安全规范化
    normalized = safe_normalize_reference(brand.id, raw_ref)
    if not normalized.safe:
        return Resolution(Status.REVIEW, reason="unsafe_normalization")

    # 6. Catalog 是最终强约束
    ref = catalog.lookup(brand.id, normalized.value)
    if ref is None or not ref.verified:
        return Resolution(Status.REVIEW, reason="catalog_miss")

    # 7. 其他证据若与最终 reference 冲突，也不能 ACCEPT
    if has_conflicting_evidence(evidences, ref):
        return Resolution(Status.REVIEW, reason="evidence_conflict")

    return Resolution(
        Status.ACCEPT,
        brand_id=brand.id,
        reference_id=ref.id,
        reason="verified_reference"
    )
```

最终聚合只需要：

```sql
SELECT brand_id, reference_id, array_agg(record_id)
FROM reference_resolution
WHERE status = 'ACCEPT'
GROUP BY brand_id, reference_id;
```

---

## 13. Prompt 的生产化版本

WDC-PAVE 的 Prompt 设计可以精简成以下 reference 专用模板：

### System

```text
你是商品标识符抽取器，不是商品匹配器。
只提取输入文本中真实出现的编号。
不得根据品牌知识补写输入中未出现的型号。
如果无法确定哪个编号是当前售卖腕表本体的品牌 Reference Number，必须 abstain。
```

### Brand-specific schema/instruction

```text
Brand: Rolex
Target identifier role: brand_reference
Do not return:
- seller SKU
- platform inventory number
- serial number
- accessory compatibility reference
- warranty/card internal number
```

### Few-shot

动态检索 3～5 个同品牌、不同来源的相似 hard cases。

### Output

```json
{
  "candidates": [
    {
      "raw_value": "...",
      "identifier_role": "brand_reference|seller_sku|platform_sku|serial_number|accessory_compatible_reference|unknown",
      "evidence_text": "..."
    }
  ],
  "abstain": true,
  "conflict": false
}
```

### 服务端验证

LLM 的 `evidence_text` 和 `raw_value` 必须做原文 span check。只有 `brand_reference` 才进入后续 canonicalizer。

---

## 14. 图片怎么用才符合 precision-first

当前有图片，这是优势，但使用方式非常关键。

### 不推荐

```text
image embedding similarity 高
→ 两条记录自动判同一 reference
```

腕表同系列不同 reference 常只有颜色、材质、尺寸或细小部件差异，视觉近似很容易制造高置信 false positive。

### 推荐

```text
图片
  ├─ 表背 OCR
  ├─ 保卡 OCR
  ├─ 盒标 OCR
  └─ 吊牌 OCR
       │
       ▼
reference candidate evidence
```

图片可以做到两件事：

1. 文本缺 reference 时恢复候选；
2. 文本已有 reference 时做冲突否决。

即使 OCR 得到一个 reference，也应继续走 brand rule + catalog gate。

---

## 15. 与 WDC-PAVE 原实现相比，需要修改什么

| WDC-PAVE | 当前项目应改成 |
|---|---|
| category-specific attributes | brand-specific reference schema |
| 通用 PAVE | reference 专用抽取 |
| String Wrangling 可直接删除符号 | 品牌专用、可逆、安全规范化 |
| F1 为核心评估 | auto-accept precision 为第一指标 |
| LLM 直接输出规范化 attribute | LLM 只提 raw candidate，canonicalizer 独立 |
| FAISS category top-k demo | brand + role + source-aware top-k demo |
| n/a | ABSTAIN / REVIEW 一级状态 |
| 单次实验脚本 | versioned service + queue + replay |
| exact-match benchmark | final `(brand_id, reference_id)` exact key |

---

## 16. 第一阶段可直接落地的 MVP

建议不要先训练模型，先用两周左右能验证核心假设的 MVP：

### Phase 1：数据与规则

1. 三源统一 schema；
2. 建品牌 canonical 表；
3. 统计各平台已有 reference 独立字段覆盖率；
4. 为 Top 10～20 品牌写 reference pattern；
5. 建 `reference_evidence` 和 `reference_resolution` 表；
6. 用规则先跑一轮，统计 UNIQUE / CONFLICT / NONE。

### Phase 2：Catalog

从高可信独立字段和人工确认样本建立：

```text
brand_id + canonical_reference + aliases
```

先覆盖头部品牌和头部 reference。

### Phase 3：LLM PAVE fallback

只把 `NONE` 和 `MULTIPLE_CANDIDATES` 送 LLM；接入：

- JSON Schema；
- 3～5 个品牌内 semantic few-shot；
- evidence span check；
- role 分类；
- abstain。

### Phase 4：图片 OCR

只处理：

- 无文本 reference；
- 文本冲突；
- 高价值待人工 review。

### Phase 5：Strict Join

只对 `ACCEPT` 写 identity index，并产出三源 cluster。

---

## 17. 上线 Gate 建议

自动合并策略建议固定成代码/配置，而不是“模型置信度 > 0.95”。

例如第一版：

```yaml
auto_accept:
  brand_verified: true
  unique_reference_candidate: true
  identifier_role: brand_reference
  source_span_verified: true
  safe_normalization: true
  catalog_hit: true
  catalog_verified: true
  conflicting_evidence: false
  llm_only_allowed: false
  image_similarity_only_allowed: false
```

这样任何人都能解释为什么一条数据被自动合并，也能通过配置收紧策略。

---

## 18. 这篇论文最值得继承、以及最不能照搬的地方

### 最值得继承

1. **Schema 驱动抽取**：减少模型自由度；
2. **属性级 normalization guideline**：适合改成品牌 reference rule；
3. **semantic few-shot retrieval**：几百条标注可以被高效复用；
4. **不同网站 demonstration**：非常适合三来源迁移；
5. **Pydantic/JSON 校验**：输出必须可程序化验证；
6. **n/a**：明确承认“不知道”是合法结果；
7. **exact match 评估**：reference 不能用模糊相似代替。

### 最不能照搬

1. 不能因为论文 90%+ F1 就让 LLM 做 final matcher；
2. 不能把 Part Number 的“去掉所有非字母数字”直接用于全部腕表品牌；
3. 不能优化整体 F1 而牺牲 auto-accept precision；
4. 不能把图像外观相似当成 reference 身份证明；
5. 不能把 LLM 自己生成的 canonical 型号当事实；
6. 不能在 reference 缺失时为了 recall 强行匹配。

---

## 19. 最终推荐方案

对于当前 Spec，我建议最终系统不是：

```text
文本 + 图片 → Entity Matching Model → same / different
```

而是：

```text
文本/独立字段/图片 OCR
        ↓
Reference Candidate Extraction
        ↓
Identifier Role Classification
        ↓
Brand-aware Safe Normalization
        ↓
Verified Reference Catalog
        ↓
ACCEPT / REVIEW / ABSTAIN
        ↓
仅 ACCEPT：严格 (brand_id, reference_id) Join
```

WDC-PAVE/论文中的 LLM + semantic few-shot 技术放在 `Reference Candidate Extraction` 这个局部步骤里非常合适；其余步骤尽可能使用确定性规则、数据表和硬 gate。

这套设计的最大优势是：

- **把模型错误限制在候选层，而不是让模型错误直接变成误合并；**
- **可以利用几百条黄金样本，却不依赖大规模训练；**
- **1000 万级也无需全量 pairwise 比较；**
- **持续增量时只需要 resolve 新/变更记录，然后做索引查找；**
- **每次匹配都能回答“这个 reference 从哪里来、经过哪条规则、为什么被自动接受”；**
- **当证据不足时系统天然拒识，和 precision-first 目标一致。**

如果只选一个可以最快开始实现的切入点，我会优先做：

> **`reference_evidence` / `reference_resolution` 两张表 + Top 品牌 deterministic extractor + verified catalog + strict gate。**

LLM PAVE 和 OCR 都作为后续提高 coverage 的插件，而不是第一天就放到最终匹配判定里。
