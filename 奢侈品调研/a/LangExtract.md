# LangExtract：把 LLM 限制为“有原文证据的 Reference 提取器”，为二奢同款匹配构建 fail-closed 主链路

## 0. 结论先行

本次从 `奢侈品文章调研.md` 中选取项目 **Google LangExtract** 做深入分析。

- 项目：<https://github.com/google/langextract>
- README：<https://github.com/google/langextract/blob/main/README.md>
- 核心抽取入口：<https://github.com/google/langextract/blob/main/langextract/extraction.py>
- 对齐 / grounding 实现：<https://github.com/google/langextract/blob/main/langextract/resolver.py>
- 文档分块与推理编排：<https://github.com/google/langextract/blob/main/langextract/annotation.py>
- 目标需求：<https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31>

分析前重新检查了 `奢侈品调研/a`，当前已有结果包括：

- `Ameli.md`
- `An Entity-Matching System Based on Multimodal Data for Two Major E-Commerce Stores in Mexico.md`
- `AnyMatch -- Efficient Zero-Shot Entity Matching with a Small Language Model.md`
- `ComEM.md`
- `Confidence Classifiers with Guaranteed Accuracy or Precision.md`
- `Conformal Selective Prediction with General Risk Control.md`
- `Deep Entity Matching with Pre-Trained Language Models.md`
- `DeepBlocker.md`
- `Fine-tuning large language models with contrastive margin ranking loss for selective entity matching in product data integration.md`
- `Large Scale Generative Multimodal Attribute Extraction for E-commerce Attributes.md`
- `MOON2.0 -- Dynamic Modality-balanced Multimodal Representation Learning for E-commerce Product Understanding.md`
- `Multi-Value-Product Retrieval-Augmented Generation for Industrial Product Attribute Value Identification.md`
- `PAM - Understanding Product Images in Cross Product Category Attribute Extraction.md`
- `Tailoring entity resolution for matching product offers.md`
- `TransClean - Finding False Positives in Multi-Source Entity Matching under Real-World Conditions via Transitive Consistency.md`
- `Using LLMs for the Extraction and Normalization of Product Attribute Values.md`
- `parts-distributor-sku-classifier.md`
- `pyJedAI.md`

`LangExtract` 尚未出现，因此本次继续执行。

当前 Spec 最重要的约束不是“找视觉/语义上相似的商品”，而是：

1. 雷小安、腕表之家、奢当家三源数据；
2. 规模约 100 万～1000 万条，且持续增量；
3. 业务定义明确：**同 reference number / 型号才算同商品**；
4. reference 可能在结构化字段，也可能藏在标题、描述甚至图片里；
5. **precision 极端优先，宁可拒识 / 漏匹配，也不能误匹配**；
6. 可以人工标注几百对黄金样本。

针对这个定义，我认为 LangExtract 最值得借鉴的不是“用 LLM 做 Entity Matching”，而是把 LLM 限制在更窄、也更安全的一层：

> **只让模型从商品文本里提出“疑似 manufacturer reference”，并要求该值必须能回指到原始文本的精确字符区间；任何无法从原文定位、只能靠模型知识猜出来的型号，一律不进入自动匹配主链路。**

最终生产架构应当是：

```text
三源原始商品
    |
    v
来源适配 / 字段语义识别
    |
    +--> 可信结构化 reference 字段 -----------+
    |                                        |
    +--> 确定性正则 / 品牌规则 --------------+----> Reference Evidence
    |                                        |
    +--> LangExtract 原文 grounding ----------+
    |                                        |
    +--> 图片 OCR（只做候选证据） ------------+
                                             |
                                             v
                              Evidence Gate / 冲突熔断
                                             |
                                             v
                                品牌感知 Canonicalizer
                                             |
                                             v
                           (brand_id, canonical_reference)
                                             |
                                             v
                               Reference Entity exact join
                                             |
                      +----------------------+------------------+
                      |                                         |
                  AUTO_MATCH                                ABSTAIN
             （硬证据完全一致）                     （缺失/冲突/模糊/未知）
```

核心原则只有一句：

> **LangExtract 负责“找证据”，Reference Entity 负责“定义身份”；LLM 永远没有权限直接宣布两条商品是同款。**

这比 pairwise LLM、embedding 相似度或多模态相似度更符合本需求的业务定义和风险偏好。

---

# 1. LangExtract 到底解决什么问题

LangExtract 是 Google 开源的 Python 信息抽取库。它让用户用自然语言 instruction + few-shot examples 定义抽取任务，然后由 LLM 输出结构化 extraction，并把 extraction 再映射回源文本中的位置。

README 把它的能力概括为：

- structured extraction；
- precise source grounding；
- few-shot schema guidance；
- 长文 chunking；
- 并行推理；
- 多次 extraction pass；
- JSONL / 可视化；
- Gemini / OpenAI / Ollama 等 provider。

对于本需求，真正关键的是 **source grounding**。

普通 LLM 抽取可能出现：

```text
输入标题：
“劳力士 黑水鬼 41mm 全套 2023年”

模型凭世界知识输出：
126610LN
```

这个答案即使“很可能对”，也不能进入当前系统，因为原文没有该 reference。

LangExtract 的输出对象 `Extraction` 除了：

```text
extraction_class
extraction_text
attributes
```

还会带：

```text
char_interval
alignment_status
```

如果模型生成的 extraction 无法在源文本找到，`char_interval` 可以为空。因此我们可以建立一个非常强的生产约束：

```text
没有原文字符证据 => 直接拒绝
```

这正好把 LLM 从“可能会猜”的生成器变成“候选提取器”。

---

# 2. 代码级拆解：LangExtract 的实际架构

截至本次分析，项目 `pyproject.toml` 标记版本为 `1.6.0`，Python 要求 `>=3.10`。核心模块边界很清晰：

```text
langextract/
  extraction.py       # 高层 extract API / 组件装配
  annotation.py       # 文档分块、batch、推理、合并
  prompting.py        # prompt 模板
  resolver.py         # 解析模型输出 + 和原文对齐
  prompt_validation.py# few-shot example 对齐预检
  factory.py          # provider/model factory
  providers/          # Gemini / OpenAI / Ollama provider
  core/data.py        # Document / Extraction / CharInterval 等
  chunking.py         # 文本分块
  io.py               # JSONL 等 I/O
```

`pyproject.toml` 还用 entry point 注册 provider：

```text
langextract.providers
  gemini -> GeminiLanguageModel
  ollama -> OllamaLanguageModel
  openai -> OpenAILanguageModel
```

并通过 import-linter 明确约束：

- `core` 不反向依赖 provider；
- provider 不依赖旧的 inference 高层；
- `core` 不依赖 annotation/chunking/prompting/resolver。

这说明它本身已经是一个比较干净的“核心数据结构 + 推理 provider + 编排 + resolver”分层架构，适合抽出其中的 grounded extraction 能力嵌到现有数据流水线里，而不需要把整个业务做成 LangExtract 项目形态。

## 2.1 `extract()`：高层装配入口

`langextract/extraction.py` 的 `extract()` 主要做几件事：

```text
1. 校验 few-shot examples / output schema
2. Prompt alignment 预检查
3. 根据 model/config 创建 language model
4. 创建 FormatHandler
5. 创建 Resolver
6. 创建 Annotator
7. annotate_text / annotate_documents
```

与本需求最相关的参数包括：

```python
max_char_buffer
batch_length
max_workers
extraction_passes
temperature
resolver_params
output_schema
prompt_validation_level
prompt_validation_strict
```

这意味着我们不用 fork LangExtract，就能把它调成“高 precision 模式”。

## 2.2 `Annotator`：chunk -> prompt -> infer -> resolve -> align

`annotation.py` 的单 pass 主流程非常直接：

```text
Document
  -> ChunkIterator
  -> make_batches_of_textchunk
  -> build_prompt
  -> language_model.infer(batch_prompts)
  -> resolver.resolve(model_output)
  -> resolver.align(extractions, source_chunk)
  -> AnnotatedDocument
```

它支持：

- batch_length 控制一批处理多少 chunk；
- max_workers 控制 provider 层并发；
- max_char_buffer 控制 chunk 大小；
- extraction_passes > 1 时重复抽取并合并非重叠结果。

对小说/病历等长文，多 pass 是为了 recall；但对本项目，商品 title/description 都很短，而且 `precision >> recall`，所以我建议：

```text
extraction_passes = 1
```

不要为了多找几个候选而增加模型随机性和成本。

## 2.3 `Resolver`：grounding 其实是“生成后对齐”

这是最需要看源码、也最容易被 README 的“precise grounding”一句话误导的部分。

`resolver.py` 不是强迫 LLM 在解码时逐字符复制原文，而是：

```text
LLM 先产生 extraction_text
        |
        v
Resolver 解析 JSON/YAML
        |
        v
WordAligner 尝试把 extraction_text 对齐回 source_text
```

对齐状态包括：

```text
MATCH_EXACT
MATCH_LESSER
MATCH_FUZZY
None
```

并且默认参数允许 fuzzy alignment：

```text
enable_fuzzy_alignment = True
fuzzy_alignment_threshold = 0.75
accept_match_lesser = True
```

这对一般信息抽取很友好，但 **不满足本 Spec 的 fail-closed 要求**。

例如模型输出的 reference 与原文只差一个字符，而 fuzzy alignment 仍可能把它挂到一个相近 span 上。对于人物、药物实体识别可能还能接受，对于腕表 reference 则完全不能接受：

```text
126610LN
126610LV
```

只差后缀，业务身份已经不同。

因此不能只写：

```python
if extraction.char_interval:
    accept()
```

生产必须额外收紧成：

```text
alignment_status == MATCH_EXACT
AND char_interval != None
AND source[start:end] == extraction_text
```

同时关闭 fuzzy 和 lesser。

这是本次分析中最重要的代码级结论。

## 2.4 `Extraction` 数据结构恰好适合做证据审计

`core/data.py` 中 `Extraction` 保留：

```text
extraction_class
extraction_text
char_interval
alignment_status
attributes
```

这意味着数据库里完全可以把：

```text
原始 title
抽出的 reference
字符起止位置
alignment 状态
模型版本
prompt 版本
validator 版本
```

一起落库。

之后任何一条自动 match 都可以反向解释：

```text
为什么匹配？
-> 因为两条 record 都绑定到 ref_entity Rolex/126610LN

为什么 record A 绑定到该 ref？
-> 因为 title[18:26] == "126610LN"

谁提出来的？
-> langextract extractor v3 / model xxx / prompt v5

它通过了什么规则？
-> MATCH_EXACT + Rolex reference grammar + registry hit
```

这是“绝对不能误匹配”场景里非常有价值的可审计性。

---

# 3. 不应该怎样使用 LangExtract

## 3.1 不让 LLM 直接做 pairwise `same / different`

不要把两条商品拼成：

```text
商品 A：...
商品 B：...
请判断是不是同款。
```

然后把 LLM 的 `same=true` 直接当 match。

原因：

1. 当前业务身份已经有明确键：reference；
2. pairwise LLM 会把系列、尺寸、颜色、外观相似等软特征混入；
3. 对 1000 万条数据，pairwise 候选数量和成本都更高；
4. 最危险的是 false positive 没有硬证据链。

## 3.2 不信任 LLM 生成的“品牌 / 型号含义”属性

LangExtract 的 `attributes` 可以来自模型推理，并不天然要求每个 attribute 都能逐字 grounding。

因此：

```text
extraction_text = "126610LN"  # 可以做硬证据
attributes.role = "manufacturer_reference" # 只能做 hint
```

真正的 identifier role 还要经过确定性 validator。

## 3.3 不把 `MATCH_FUZZY` 当可靠 reference

普通 NER 的 fuzzy 对齐与 reference identity 的风险模型不同。

生产规则建议：

```text
MATCH_EXACT  -> 才有机会继续
MATCH_LESSER -> reject
MATCH_FUZZY  -> reject
None         -> reject
```

## 3.4 不用模型世界知识补全缺失 reference

例如标题只有“黑水鬼”，即使模型知道典型 ref，也必须返回空，而不是补 `126610LN`。

Prompt 要明确写：

```text
如果 reference/model number 没有逐字出现在输入中，返回空。
严禁根据品牌、系列、尺寸、年份或常识猜测型号。
```

同时靠 literal span gate 做二次强制。

---

# 4. 建议的生产级 Reference-first 架构

## 4.1 Layer 0：来源适配器，不要一开始就 LLM

三个来源先做 source adapter，把字段按语义分类：

```text
manufacturer_reference_field
platform_sku_field
serial_number_field
title_field
description_field
brand_field
image_urls
unknown_identifier_field
```

这一步非常关键，因为不同平台的“货号 / 编号 / 商品编号 / 型号”含义可能完全不同。

比如：

```text
平台商品编号 = 938271
品牌型号 = 126610LN
```

如果不区分字段语义，任何 extractor 都可能把 `938271` 当 reference。

优先级应该是：

```text
可信 manufacturer_reference 结构化字段
    > 确定性标题规则
    > LangExtract grounded candidate
    > OCR candidate
    > 其他语义/视觉信号
```

## 4.2 Layer 1：确定性规则先吃掉最简单样本

如果 title 中已经有典型 reference pattern，先用品牌规则抽取，不要调用 LLM。

例如：

```text
brand = Rolex
pattern = 品牌专属 reference grammar
```

规则层优势：

- 便宜；
- 可复现；
- 可单测；
- 无 hallucination；
- 适合大规模 backfill。

LangExtract 应该主要覆盖：

- title/description 中有多个编号，需要判断哪个像 manufacturer ref；
- reference 周围上下文复杂；
- 中英混排；
- 文本结构不固定；
- 规则没有覆盖的新表达。

## 4.3 Layer 2：LangExtract 只做“原文 reference candidate 抽取”

建议 extraction class 只定义一个核心类型：

```text
manufacturer_reference_candidate
```

不要一上来让模型同时抽几十个属性。

Prompt 示例：

```text
从商品文本中抽取明确写出的品牌官方 reference/model number。

规则：
1. extraction_text 必须逐字来自输入；
2. 不允许猜测、补全、改写、纠错；
3. 平台 SKU、库存号、订单号、序列号、年份、尺寸、价格不是 reference；
4. 如果不能确定某个编号是 manufacturer reference，可以不抽；
5. 宁可漏掉，也不要误抽；
6. 保留原始标点、斜杠、连字符、前导零和字母后缀。
```

few-shot examples 必须特意加入 hard negative：

```text
“劳力士 126610LN 库存号 883921”
=> 只抽 126610LN

“商品编号 883921 劳力士 黑水鬼”
=> 不抽任何 reference

“OMEGA Ref. 210.30.42.20.01.001”
=> 精确抽出 210.30.42.20.01.001

“劳力士 黑水鬼 41mm”
=> 空，不允许根据知识补 reference
```

注意：上面的品牌示例只是说明 prompt 设计原则，真正 few-shot 应从三个来源的真实文本采样构建。

## 4.4 Layer 3：强制 Literal Grounding Gate

这是 LangExtract 输出后必须加的业务安全壳。

建议逻辑：

```python
from langextract.core import data as lxdata


def is_literal_exact(text: str, extraction) -> bool:
    ci = extraction.char_interval
    if ci is None:
        return False
    if extraction.alignment_status != lxdata.AlignmentStatus.MATCH_EXACT:
        return False
    if ci.start_pos is None or ci.end_pos is None:
        return False
    return text[ci.start_pos:ci.end_pos] == extraction.extraction_text
```

只要返回 False：

```text
不进入 canonicalization
不进入 exact join
不自动 match
```

即使未来 LangExtract 的 alignment 实现发生变化，这个最后的 Python substring equality 仍然是简单、透明、容易验证的安全闸。

---

# 5. 推荐的 LangExtract 配置

下面这份配置不是为了最高 recall，而是为了本 Spec 的 precision-first。

```python
import langextract as lx
from langextract import prompt_validation as pv
from langextract.core import data as lxdata

PROMPT = """
Extract only manufacturer-issued reference/model numbers that are explicitly
present in the input text.

Requirements:
- extraction_text must be copied verbatim from the input.
- Never infer, complete, correct, normalize, or guess a reference.
- Do not extract platform SKU, stock number, serial number, year, size, price,
  order number, or seller-specific code.
- If uncertain, return no extraction.
- Preserve punctuation, slashes, hyphens, leading zeros, and suffix letters.
"""

examples = [
    lx.data.ExampleData(
        text="劳力士 126610LN 全套 库存号 883921",
        extractions=[
            lx.data.Extraction(
                extraction_class="manufacturer_reference_candidate",
                extraction_text="126610LN",
            )
        ],
    ),
    lx.data.ExampleData(
        text="商品编号 883921 劳力士 黑水鬼 41mm",
        extractions=[],
    ),
]


def extract_ref(text: str):
    result = lx.extract(
        text_or_documents=text,
        prompt_description=PROMPT,
        examples=examples,
        model_id="YOUR_PINNED_MODEL_ID",
        temperature=0.0,
        extraction_passes=1,
        resolver_params={
            "enable_fuzzy_alignment": False,
            "accept_match_lesser": False,
            "exact_alignment_algorithm": "dp",
        },
        prompt_validation_level=pv.PromptValidationLevel.ERROR,
        prompt_validation_strict=True,
        show_progress=False,
        fetch_urls=False,
    )

    accepted = []
    for e in result.extractions or []:
        ci = e.char_interval
        if ci is None:
            continue
        if e.alignment_status != lxdata.AlignmentStatus.MATCH_EXACT:
            continue
        if ci.start_pos is None or ci.end_pos is None:
            continue
        if text[ci.start_pos:ci.end_pos] != e.extraction_text:
            continue
        accepted.append(e)

    return accepted
```

几个参数的理由：

### `temperature=0.0`

降低输出随机性。即使 provider 的确定性不是数学保证，也比高 temperature 更符合抽取任务。

### `extraction_passes=1`

多 pass 是 recall 技巧，本业务没有必要为了找更多 reference 候选而重复调用。

### `enable_fuzzy_alignment=False`

reference 差一个字符也可能是不同型号，不能 fuzzy。

### `accept_match_lesser=False`

禁止 extraction 比实际 source span 更长但仍被部分接受。

### `prompt_validation_level=ERROR + strict=True`

few-shot example 本身也必须严格对齐，避免用一个“示范中都不是原文拷贝”的 prompt 去训练模型行为。

### `fetch_urls=False`

LangExtract 的 `extract()` 默认把字符串当 literal text。源码明确警告，`fetch_urls=True` 会直接请求 URL，存在 SSRF 风险。当前业务输入就是抓取后的字段文本，不需要让 extractor 自己联网，保持 False 即可。

### `model_id` 必须 pin

不要生产环境永远使用“最新模型别名”。应记录：

```text
model_provider
model_id
extractor_version
prompt_version
examples_version
langextract_version
```

模型升级要走 shadow replay，而不是直接替换。

---

# 6. Reference 不是“抽到了字符串”就结束：还需要 Identifier Role Gate

真正的难点之一不是“文本里有没有编号”，而是：

```text
这个编号到底是 manufacturer reference，还是平台 SKU / 库存号 / 序列号？
```

因此建议把 LangExtract 输出称为：

```text
reference_candidate
```

而不是直接叫：

```text
reference
```

之后必须过 `IdentifierRoleValidator`。

Validator 的输入：

```text
source
field_name
brand_id
raw_text
candidate
candidate_span
附近上下文
```

Validator 的确定性规则可以包括：

```text
1. source-specific 字段白名单 / 黑名单
2. candidate 是否命中品牌 reference grammar
3. 是否命中平台 SKU grammar
4. 是否命中 serial number 特征
5. 是否命中 reference registry
6. 同记录是否存在另一个更高等级 reference 冲突
```

建议输出：

```text
MANUFACTURER_REFERENCE
PLATFORM_SKU
SERIAL_NUMBER
UNKNOWN_IDENTIFIER
CONFLICT
```

只有 `MANUFACTURER_REFERENCE` 才能继续。

不要让 LLM 的 `attributes={"role": ...}` 独立决定这个结果；LLM attribute 最多只能提供一个辅助 hint。

---

# 7. 品牌感知 Canonicalization：宁可少归一化，也不要产生碰撞

很多 Entity Matching 项目喜欢这样 normalize：

```text
uppercase
remove spaces
remove punctuation
keep alphanumeric only
```

对 reference identity 这是危险的。

本项目必须采用 **non-destructive canonicalization**。

推荐基础步骤：

```text
1. Unicode NFKC
2. trim 首尾空白
3. 英文字母 uppercase
4. 只统一明确等价的 Unicode 空白 / 连字符字符
5. 按品牌 parser 解析
6. 保留前导零
7. 保留字母 suffix
8. 保留可能有语义的 dot / slash / hyphen
```

明确禁止全局做：

```text
remove all punctuation
strip all letters
strip leading zero
只保留数字
模糊 edit-distance 合并
```

例如：

```text
126610LN != 126610LV
```

两者只差两个后缀字母，但绝对不能合并。

再例如类似：

```text
210.30.42.20.01.001
```

点号本身是 reference 的标准表达结构之一。即使业务最终决定生成某种无点 canonical key，也必须是 **品牌规则验证过的等价变换**，而不是全平台统一“删标点”。

建议每个 canonicalization 规则都带版本：

```text
canonicalizer_version = rolex_v3
canonicalizer_version = omega_v2
```

升级后做 collision diff：

```text
old_ref_entity -> new_ref_entity
```

只要发现多个旧 reference 被新规则压成一个 key，就默认阻断发布，先人工检查。

---

# 8. 建议建立 Reference Registry，而不是每次让 LLM 理解世界

可以从三源数据中的高可信结构化 reference 自动沉淀 registry：

```text
reference_registry
----------------------------------
brand_id
canonical_reference
raw_variants
source_count
record_count
first_seen_at
last_seen_at
trust_level
review_status
canonicalizer_version
```

它的作用不是定义所有腕表知识，而是给 title/description 抽取增加一个低风险验证条件。

例如：

```text
LangExtract 从 title 精确抽到 126610LN
AND brand = Rolex
AND registry 已有高可信 126610LN
=> 可以提升证据等级
```

如果抽到一个 registry 从未见过的 token：

```text
12661OLN
```

即使是 `MATCH_EXACT`，也先进入 `UNKNOWN_REFERENCE`，不要自动加入 Reference Entity。

之后如果它在另一个独立来源的可信 `型号/reference` 结构化字段中出现，或者人工确认，才升级 registry。

这样系统会随着数据积累越来越强，但不会让“第一次见到一个奇怪字符串”直接污染主键空间。

---

# 9. 证据等级：把“是否自动匹配”写成显式 policy

建议不要用一个 0~1 的黑盒 score 决定自动发布，而是使用离散 Evidence Grade。

## Grade S：最高可信

满足：

```text
source adapter 确认字段语义 = manufacturer reference
AND brand 可确定
AND canonicalizer 通过
AND 无冲突
```

可自动建立 `record -> reference_entity`。

## Grade A：高可信文本证据

满足：

```text
LangExtract / 确定性 extractor 找到候选
AND literal substring exact
AND alignment_status = MATCH_EXACT
AND IdentifierRoleValidator = MANUFACTURER_REFERENCE
AND brand grammar 通过
AND registry hit
AND 无冲突
```

可自动建立绑定。

## Grade B：有原文证据，但 registry 未知

```text
exact grounded
brand grammar 看起来合法
registry miss
```

只落 `ref_evidence`，默认 abstain。

## Grade C：OCR reference candidate

图片 OCR 抽到候选，但没有独立文本/结构化确认。

默认 abstain。

## Grade D：模糊 / 无 grounding / 角色不明

包括：

```text
MATCH_FUZZY
MATCH_LESSER
char_interval=None
LLM 推断出来但原文没有
编号角色不明
```

直接 reject。

这样主链路可以明确写成：

```text
AUTO_BIND only if grade in {S, A}
```

而不是：

```text
if score > 0.91
```

---

# 10. Exact Reference Entity：不要存大量 pairwise match edge

既然业务已经定义：

```text
同 reference = 同商品
```

最自然的数据模型不是：

```text
record_a, record_b, match_score
```

而是：

```text
record -> reference_entity
```

数据库可以这样设计。

```sql
CREATE TABLE product_record (
    id                 BIGINT PRIMARY KEY,
    source             TEXT NOT NULL,
    source_record_id   TEXT NOT NULL,
    brand_id           BIGINT,
    title              TEXT,
    description        TEXT,
    raw_payload        JSONB NOT NULL,
    content_hash       TEXT NOT NULL,
    ingested_at        TIMESTAMPTZ NOT NULL,
    UNIQUE (source, source_record_id)
);

CREATE TABLE ref_evidence (
    id                    BIGSERIAL PRIMARY KEY,
    record_id             BIGINT NOT NULL REFERENCES product_record(id),
    evidence_channel      TEXT NOT NULL,
    field_name            TEXT,
    raw_candidate         TEXT NOT NULL,
    char_start            INT,
    char_end              INT,
    alignment_status      TEXT,
    identifier_role       TEXT NOT NULL,
    canonical_reference   TEXT,
    evidence_grade        TEXT NOT NULL,
    accepted              BOOLEAN NOT NULL,
    reject_reason         TEXT,
    model_provider        TEXT,
    model_id              TEXT,
    prompt_version        TEXT,
    extractor_version     TEXT NOT NULL,
    validator_version     TEXT NOT NULL,
    canonicalizer_version TEXT,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ref_evidence_record
    ON ref_evidence(record_id);

CREATE INDEX idx_ref_evidence_canonical
    ON ref_evidence(canonical_reference)
    WHERE accepted = true;

CREATE TABLE reference_entity (
    id                    BIGSERIAL PRIMARY KEY,
    brand_id              BIGINT NOT NULL,
    canonical_reference   TEXT NOT NULL,
    canonicalizer_version TEXT NOT NULL,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (brand_id, canonical_reference)
);

CREATE TABLE record_reference (
    record_id             BIGINT PRIMARY KEY REFERENCES product_record(id),
    reference_entity_id   BIGINT NOT NULL REFERENCES reference_entity(id),
    evidence_grade        TEXT NOT NULL,
    bound_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_record_reference_entity
    ON record_reference(reference_entity_id);
```

跨源同款直接变成：

```sql
SELECT rr.reference_entity_id,
       array_agg(pr.id) AS records,
       array_agg(DISTINCT pr.source) AS sources
FROM record_reference rr
JOIN product_record pr ON pr.id = rr.record_id
GROUP BY rr.reference_entity_id
HAVING count(DISTINCT pr.source) >= 2;
```

复杂度从“候选 pairwise 比较”变成“先安全解析 key，再走索引 / hash exact join”。

对 100 万～1000 万记录，这是更符合问题结构的做法。

---

# 11. 冲突熔断：任何硬冲突都应该让记录进入 ABSTAIN

同一 record 可能同时有：

```text
结构化字段：126610LN
标题：126610LV
```

最危险的做法是：

```text
相信结构化字段，忽略标题
```

或：

```text
相信 LLM 分数更高的那个
```

正确做法是：

```text
两个高可信 manufacturer reference 不一致
=> REFERENCE_CONFLICT
=> 不绑定任何 reference_entity
=> 人工 / 后续规则处理
```

同理，如果：

```text
brand structured field = Rolex
reference grammar / registry 明确属于另一个 brand family
```

也应该 fail closed。

建议写成 invariant：

```text
一个 record 在任意时刻最多有一个 accepted manufacturer reference。
```

数据库发布逻辑执行前必须检查：

```text
count(distinct accepted canonical_reference) <= 1
```

大于 1 就不发布。

---

# 12. 图片怎么用：OCR 可以补证据，但图片相似度不能定义身份

Spec 里提到图片可用。

这里建议把图片能力拆成两类。

## 12.1 OCR Reference Candidate

流程：

```text
商品图片
  -> OCR
  -> token / bbox
  -> reference candidate
  -> brand grammar / registry validator
```

OCR 最有价值的场景是：

- 吊牌；
- 保卡；
- 表盒标签；
- 商品详情图中写出的型号。

但 OCR 独立输出默认只能是 Grade C，因为图片还可能包含：

- 平台库存标签；
- 店铺内部编码；
- 保卡序列号；
- 配件型号；
- 同图多件商品；
- OCR 单字符错误。

所以不要做：

```text
OCR 识别到 126610LN -> 自动 match
```

更安全的是：

```text
OCR ref == title exact-grounded ref
=> 增加审计信心

OCR ref == structured trusted ref
=> 增加审计信心

OCR 独立 ref
=> 候选 / 待复核
```

## 12.2 图像 embedding / CLIP 相似度

图像相似度可以：

- 给 abstain 队列排序；
- 找明显冲突；
- 帮人工复核；
- 辅助发现同图盗用 / 重复 listing。

但不能升级成 `same reference` 的唯一证据。

原因很简单：同 reference 的不同二手实物可以外观差异很大，不同 reference 的官方图又可能极其相似。

---

# 13. 百万到千万级怎么跑：把 LLM 放在“窄漏斗”里

如果 1000 万条全部调用一次 LLM，成本和吞吐都不理想，也没必要。

推荐漏斗：

```text
100% records
   |
   +--> trusted structured ref -> 直接 validator
   |
   +--> deterministic brand regex -> 直接 validator
   |
   +--> registry exact candidate -> 直接 validator
   |
   +--> unresolved textual records -> LangExtract
   |
   +--> still unresolved -> OCR / review / abstain
```

LangExtract 只处理前面规则吃不掉的部分。

## 13.1 按文本哈希缓存

很多商品标题模板重复。建立：

```text
extract_cache_key = sha256(
    source + field_name + brand_id + raw_text +
    prompt_version + model_id + validator_version
)
```

相同输入不重复推理。

## 13.2 先按 distinct text 处理

backfill 时可以先：

```sql
SELECT DISTINCT brand_id, title
FROM product_record
WHERE ...;
```

对 distinct 文本抽取后再回填记录映射。

## 13.3 LangExtract 的并发只是 worker 内部并发

LangExtract 自身支持：

```text
batch_length
max_workers
```

生产仍建议外部做任务队列：

```text
DB / CDC
   -> reference_extraction_queue
       -> extractor workers
           -> evidence store
               -> binder
```

每个任务 idempotent：

```text
(record_id, content_hash, extractor_version)
```

这样增量更新只重跑内容发生变化的 record。

## 13.4 不需要 O(N²) blocking

只要最终 accepted key 是：

```text
(brand_id, canonical_reference)
```

就可以用 B-tree / hash index exact lookup。

embedding ANN、blocking model、pairwise matcher 都只需留给研究 / review 辅助链路，而不是主链路。

---

# 14. 增量处理建议

每次来源数据新增/变更：

```text
UPSERT product_record
      |
      v
content_hash 是否变化？
   | no -> done
   | yes
      v
撤销旧的 unpublished/derived evidence
      |
      v
source adapter
      |
      v
structured/rule extractor
      |
      +-- 已得到 Grade S/A -> validator + bind
      |
      +-- unresolved -> LangExtract
                        |
                        v
                  literal exact gate
                        |
                        v
                 canonicalizer/registry
                        |
                        +-- Grade A -> bind
                        +-- B/C/D -> abstain
```

如果后续 registry 更新，不能偷偷把所有历史 Grade B 自动升级。建议生成可审计的 promotion job：

```text
candidate_evidence_id
old_grade
new_grade
registry_version
promotion_reason
```

然后再重新执行冲突检查。

---

# 15. 黄金样本怎么标：不要只标 pair 的 same / different

用户愿意标几百对，这是好资源，但如果只标：

```text
A 与 B：same / different
```

对本架构利用率不够高。

建议标注表至少包含：

```text
record_id
brand
manufacturer_reference（若可确定）
reference_raw_span
reference_field/channel
identifier_role
pair_label
hard_negative_reason
```

尤其要主动构造 hard negatives：

```text
同品牌、reference 只差 1 个字符
同系列但 reference 不同
同尺寸/颜色但 reference 不同
一个是平台 SKU，一个是 manufacturer ref
同前缀不同 suffix
OCR 单字符错误
标题一个 ref、结构化字段另一个 ref
```

这比随机采几百对更能验证 precision-first policy。

## 15.1 数据集建议拆分

```text
gold_dev
  -> 调 prompt / brand rule / canonicalizer

gold_holdout
  -> 最终只做发布门禁

production_audit
  -> 上线后持续随机抽 accepted auto-match
```

不要在同一批几百样本上反复调规则后再报告“100% precision”。

---

# 16. 应该看哪些指标

不要只报 F1。

## 16.1 Reference extraction precision

```text
accepted reference 中真正 manufacturer reference 的比例
```

按 source / brand / channel 分桶。

## 16.2 Auto-bind precision

```text
自动绑定到 reference_entity 的 record 中绑定正确的比例
```

这是最重要指标。

## 16.3 Cross-source match precision

```text
自动产生的跨源同款 pair / group 中真正同 reference 的比例
```

## 16.4 Abstention / coverage

```text
coverage = 自动成功绑定的 records / 总 records
```

coverage 是第二目标，不应该为了 coverage 牺牲 precision。

## 16.5 Conflict rate

```text
结构化 ref vs title ref 冲突率
text vs OCR 冲突率
同一 record 多 accepted ref 冲突率
```

这是发现源站字段质量问题的很好监控指标。

## 16.6 Canonicalization collision rate

任何 canonicalizer 变更都统计：

```text
多少 distinct raw validated refs -> 同一个 canonical_reference
```

如果出现非预期 many-to-one，阻断部署。

---

# 17. “零错误”应该怎样验收

需求说“绝对不能误匹配”。工程上不能用有限样本证明数学意义上的零错误，只能建立非常保守的机制并给出统计上界。

如果一批独立审计样本里观察到 0 个 false positive，经典的近似 “rule of three” 给出：

```text
95% 置信上界 ≈ 3 / n
```

也就是说：

```text
n = 300,  0 错 -> 上界约 1%
n = 3000, 0 错 -> 上界约 0.1%
```

因此“只有几百对黄金样本”可以用于开发和 hard-negative 验证，但不足以统计证明极低误匹配率。

更合理的上线策略是：

```text
1. 机制上只允许 S/A 硬证据自动发布
2. holdout hard negatives 必须 0 false positive
3. shadow 模式跑全量
4. 对 auto-match 持续抽样人工审计
5. 一旦发现任何 false positive，按 source/brand/rule version 熔断对应 policy
```

这里的优势是：由于每条结果都有 evidence span 和版本，出现一个错误后能快速追溯到底是：

```text
source field semantics 错
extractor 错
alignment 错
role validator 错
canonicalizer 错
registry 污染
```

而不是面对一个不可解释的 pairwise neural score。

---

# 18. 一个完整的 Binder Policy

建议把自动绑定条件写成代码级 policy，而不是散落在多个服务里。

伪代码：

```python
def decide_binding(record, evidences, registry):
    accepted = []

    for ev in evidences:
        if ev.identifier_role != "MANUFACTURER_REFERENCE":
            continue

        if ev.channel == "structured_trusted":
            if not ev.brand_grammar_valid:
                continue
            accepted.append(("S", ev))
            continue

        if ev.channel in {"title_rule", "langextract_text"}:
            if not ev.literal_exact:
                continue
            if ev.alignment_status not in {None, "match_exact"}:
                # 规则抽取可能没有 LangExtract alignment_status；
                # LangExtract 输出则必须 match_exact。
                continue
            if not ev.brand_grammar_valid:
                continue
            if not registry.contains(record.brand_id, ev.canonical_reference):
                continue
            accepted.append(("A", ev))
            continue

        # OCR / image-only 永不自动绑定

    refs = {ev.canonical_reference for _, ev in accepted}

    if len(refs) == 0:
        return Abstain("NO_HIGH_CONFIDENCE_REFERENCE")

    if len(refs) > 1:
        return Abstain("REFERENCE_CONFLICT")

    ref = next(iter(refs))
    grade = "S" if any(g == "S" for g, _ in accepted) else "A"

    return Bind(
        brand_id=record.brand_id,
        canonical_reference=ref,
        evidence_grade=grade,
    )
```

然后同款判定就非常简单：

```python
def same_product(a, b):
    if a.reference_entity_id is None or b.reference_entity_id is None:
        return False  # 实际返回 ABSTAIN 更好
    return a.reference_entity_id == b.reference_entity_id
```

注意：业务 API 最好不要只有二值 `True/False`，而是：

```text
MATCH
NO_MATCH
ABSTAIN
```

缺 reference 并不等于不同款，只是系统没有足够证据。

---

# 19. 为什么需要 `(brand_id, canonical_reference)`，而不是只用 reference 字符串

不同品牌可能存在相同形式的短编号，甚至同一个数字串。

所以 Reference Entity 唯一键建议至少是：

```text
(brand_id, canonical_reference)
```

如果未来发现某些品牌内部 reference namespace 还需要：

```text
collection / product_type / manufacturer namespace
```

可以再扩展，但不要现在用这些软属性去“救”模糊 reference。

品牌本身不确定时：

```text
ABSTAIN
```

不要先按 reference 字符串跨品牌 join，再用图像或标题相似度补品牌。

---

# 20. 与“Regex-only”方案相比，LangExtract 的真实增益是什么

如果需求只是：

```text
找到形如 \d{6}[A-Z]{2} 的 token
```

根本不需要 LangExtract。

LangExtract 有价值的地方是 **上下文中的 identifier role disambiguation**。

例如文本：

```text
“腕表之家商品编号 873221，型号 126610LN，附件编号 93211”
```

纯 regex 会得到多个数字 token；LangExtract 可以在 prompt/few-shot 指引下优先提出 `126610LN`。

但最后仍需 deterministic validator。

所以最佳组合不是：

```text
Regex vs LLM 二选一
```

而是：

```text
Regex / field semantics 负责 precision baseline
LangExtract 负责复杂上下文覆盖
literal grounding + validator 负责把 LLM 锁回 deterministic safety envelope
```

---

# 21. 与 embedding / multimodal matcher 相比

Embedding 的问题不是能力差，而是目标函数不对。

Embedding 非常擅长：

```text
看起来像不像
语义像不像
是不是同系列
```

但业务要的是：

```text
reference 是否相等
```

例如两个产品：

```text
同品牌
同系列
同尺寸
同材质
标题几乎一样
图片几乎一样
reference 只差 suffix
```

embedding 往往会给极高相似度，但业务上必须 No Match。

因此 embedding 可放在：

```text
abstain review ranking
hard negative mining
候选错误发现
图片重复检测
```

而不应放在最终 identity decision。

---

# 22. LangExtract 本身的几个生产风险与对应措施

## 风险 1：模型从 few-shot example 抄了一个 reference

LangExtract README 已明确提到模型可能把 example 内容抽到输入里。

措施：

```text
char_interval 必须存在
MATCH_EXACT
literal substring equality
```

抄来的 example reference 无法在 source 找到，直接拒绝。

## 风险 2：fuzzy grounding 把相近 reference 对齐上

措施：

```text
enable_fuzzy_alignment=False
accept_match_lesser=False
literal substring equality
```

## 风险 3：模型把 SKU 当 manufacturer ref

措施：

```text
source field semantics
few-shot hard negatives
brand grammar
platform SKU blocklist/grammar
reference registry
```

## 风险 4：LLM 输出 JSON 格式漂移

LangExtract 已有 FormatHandler / structured output 支持，并支持 Gemini/OpenAI 的 schema constraint。生产尽量启用 provider 支持的 structured output。

但即使 chunk parse 失败，结果也应该是：

```text
无 evidence -> abstain
```

而不是 retry 后自动放宽规则。

## 风险 5：模型升级导致抽取行为漂移

措施：

```text
pin model
记录版本
shadow replay gold + production sample
新旧 evidence diff
通过后再切换
```

## 风险 6：canonicalization 比 extractor 更危险

很多事故并不是抽错，而是：

```text
把两个本来不同的 raw ref normalize 成同一个 canonical key
```

措施：

```text
non-destructive rules
brand-specific parser
collision diff
canonicalizer versioning
```

## 风险 7：新品牌 / 新 reference family

不要让通用 LLM 自动扩展 policy。

措施：

```text
unknown brand/reference -> Grade B / abstain
积累样本 -> 人工批准 grammar/registry -> 再放开
```

---

# 23. 可直接落地的服务拆分

不需要做很多微服务，逻辑边界可以先拆成 5 个模块：

```text
1. source_adapter
   - 统一三源字段语义

2. reference_extractor
   - structured field extractor
   - deterministic regex extractor
   - LangExtract fallback
   - OCR adapter

3. reference_validator
   - literal grounding gate
   - identifier role rules
   - brand grammar
   - registry lookup
   - conflict detection

4. reference_canonicalizer
   - brand-specific canonicalization
   - versioned transformations

5. reference_binder
   - create/get Reference Entity
   - enforce one-record-one-ref invariant
   - exact join
   - emit MATCH / ABSTAIN
```

数据事件：

```json
{
  "record_id": 123,
  "source": "watch_home",
  "brand_id": 7,
  "field": "title",
  "candidate": "126610LN",
  "span": [12, 20],
  "channel": "langextract_text",
  "alignment": "match_exact",
  "literal_exact": true,
  "identifier_role": "MANUFACTURER_REFERENCE",
  "canonical_reference": "126610LN",
  "registry_hit": true,
  "evidence_grade": "A",
  "extractor_version": "langextract_v1",
  "validator_version": "ref_validator_v3",
  "canonicalizer_version": "rolex_v2"
}
```

做到这一层后，后续无论换 Gemini、OpenAI、本地模型，还是完全去掉 LLM，数据库和 identity 规则都不需要重构。

---

# 24. 建议的上线顺序

## 阶段 A：先建立无 LLM 的 Reference baseline

实现：

```text
source adapter
trusted structured ref
brand-aware canonicalizer
Reference Entity
exact join
conflict abstain
```

先知道“最硬证据能覆盖多少数据”。

## 阶段 B：加确定性 title extractor

从每个品牌/来源的真实数据中找高精度 pattern。

只扩大 coverage，不改 identity rule。

## 阶段 C：LangExtract shadow

对 baseline unresolved 的 record 跑 LangExtract，但先：

```text
只记录 ref_evidence
不自动 bind
```

拿黄金样本和人工 review 看：

```text
exact grounded candidate precision
identifier role error
registry miss 分布
```

## 阶段 D：只开放 Grade A 自动 bind

门槛必须全部满足：

```text
MATCH_EXACT
literal equality
brand grammar
registry hit
no conflict
```

## 阶段 E：OCR 只扩展 review candidate

先验证 OCR 错误模型，再决定未来是否有极少数可升级规则。

这个顺序的好处是，每个阶段只增加 coverage，不需要降低前一阶段的 precision policy。

---

# 25. 最值得写成单元测试 / Property Test 的不变量

## 25.1 Reference suffix 绝不能被吞

```python
assert canon("126610LN") != canon("126610LV")
```

## 25.2 未出现在原文的 extraction 永不接受

```python
assert not accept(text="黑水鬼 41mm", extracted="126610LN")
```

## 25.3 Fuzzy alignment 永不进入自动绑定

```python
assert grade(alignment="match_fuzzy") not in {"S", "A"}
```

## 25.4 同 record 两个高可信 ref 必须 abstain

```python
assert decide(["126610LN", "126610LV"]) == "REFERENCE_CONFLICT"
```

## 25.5 不同品牌不因相同 ref 字符串自动合并

```python
assert entity_key(brand_a, "1234") != entity_key(brand_b, "1234")
```

## 25.6 OCR-only 不自动 bind

```python
assert grade(channel="ocr_only") not in {"S", "A"}
```

## 25.7 Canonicalizer 版本升级不能静默制造 many-to-one collision

对所有历史 accepted refs 跑：

```text
old distinct refs -> new canonical refs
```

出现新增碰撞就 fail CI / deployment check。

---

# 26. 我会怎样利用那“几百对人工黄金样本”

如果只能标几百对，我不会主要拿去 fine-tune 一个 pairwise matcher，而会优先花在最能降低 false positive 的地方。

建议预算结构：

```text
约 1/3：真实正例，覆盖三源/主要品牌
约 1/3：near-reference hard negatives
约 1/3：identifier role / 冲突 / OCR / 新格式异常样本
```

每一对都顺便标出：

```text
正确 brand
正确 manufacturer reference
reference span
错误编号是什么角色
```

这样一份标注同时能测试：

```text
LangExtract extraction
role validator
canonicalizer
binder
最终 pair match
```

信息密度远高于单纯 same/different 标签。

---

# 27. 一个更严格的“自动匹配发布合同”

可以把 Spec 的业务要求写成以下 contract：

```text
系统只有在能够给 A 和 B 各自生成可审计的 manufacturer-reference 证据，
并且二者在同一 brand namespace 下的 canonical reference 完全一致时，
才允许发布 MATCH。

任何以下情况都返回 ABSTAIN，而不是猜测：
- 任一侧 reference 缺失；
- reference 只靠 LLM 世界知识推断；
- grounding 不是 exact；
- identifier role 不确定；
- canonicalizer 不支持该品牌 / family；
- registry 未验证；
- 同记录存在 reference 冲突；
- 品牌不确定；
- 仅有视觉相似；
- 仅有 embedding/文本语义相似；
- 仅有 OCR 候选。
```

这就是最贴合当前需求的 selective matching。

---

# 28. 最终推荐方案

如果现在就要落地，我推荐：

```text
               ┌──────────────────────────────┐
               │  雷小安 / 腕表之家 / 奢当家  │
               └──────────────┬───────────────┘
                              │
                        Source Adapter
                              │
                ┌─────────────┴─────────────┐
                │                           │
       Structured / Regex              unresolved text
                │                           │
                │                    Google LangExtract
                │                           │
                │                  exact source grounding
                │                           │
                └─────────────┬─────────────┘
                              │
                    IdentifierRoleValidator
                              │
                    Brand Reference Grammar
                              │
                     Reference Registry
                              │
                  Conflict / Evidence Gate
                              │
                 Brand-aware Canonicalizer
                              │
                    Reference Evidence DB
                              │
                 Reference Entity Binder
                              │
               exact (brand, reference) join
                              │
              ┌───────────────┴───────────────┐
              │                               │
           MATCH                           ABSTAIN
        Grade S / A                  everything else
```

LangExtract 在这个方案里不是 matcher，而是 **带证据的 reference extraction fallback**。

这是我认为它对当前 Spec 最有价值的落点。

原因可以压缩成五点：

1. **匹配定义清晰**：reference equality，本来就不需要泛化的语义 matcher；
2. **grounding 可审计**：reference 必须能回到原始 title/description 的字符位置；
3. **可 fail closed**：关闭 fuzzy/lesser，并加 literal substring gate；
4. **可扩展到千万级**：LLM 只跑 unresolved 漏斗，最终 exact index join，不做全量 pairwise；
5. **容易持续治理**：每条绑定都有 source、span、model/prompt/validator/canonicalizer version，可以回溯、熔断、重放。

最关键的实现细节不是“安装 `langextract` 然后调用 `lx.extract`”，而是要记住：

> **LangExtract 默认是通用高 recall 信息抽取工具；当前业务必须主动把它收紧成 high-precision extractor。尤其要关闭 fuzzy alignment、拒绝 MATCH_LESSER，并在最终再做一次源文本切片与 extraction_text 的逐字相等检查。**

只要守住这个边界，它就能非常自然地补上当前 Reference-first 系统最难的一块：**reference 藏在自然语言文本里，但我们又不允许 LLM 猜。**

---

# 29. 参考源码

- Google LangExtract：<https://github.com/google/langextract>
- README / Quick Start / Grounding：<https://github.com/google/langextract/blob/main/README.md>
- `extract()` 高层 API：<https://github.com/google/langextract/blob/main/langextract/extraction.py>
- `Resolver` / WordAligner / exact & fuzzy alignment：<https://github.com/google/langextract/blob/main/langextract/resolver.py>
- `Annotator` / chunking / batch / multi-pass：<https://github.com/google/langextract/blob/main/langextract/annotation.py>
- `Extraction` / `CharInterval` / `AlignmentStatus`：<https://github.com/google/langextract/blob/main/langextract/core/data.py>
- Provider / package architecture：<https://github.com/google/langextract/blob/main/pyproject.toml>
