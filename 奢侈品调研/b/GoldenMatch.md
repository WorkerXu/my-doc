# GoldenMatch：把“高吞吐匹配计算”与“可审计身份状态机”拆开，落成 Reference-First 腕表实体身份层

- 分析人：b
- 调研项目：GoldenMatch
- 项目地址：https://github.com/benseverndev-oss/goldenmatch
- 来源清单：`奢侈品文章调研.md`
- 对应需求：https://app.notion.com/p/Spec-3bf7b2a8538b812fba00fb258024ff31

## 1. 本次为什么选择 GoldenMatch

本次执行前先检查了 `奢侈品调研/b` 已有结果，目录中已经有 Ameli、AnyMatch、DeepBlocker、Ditto、GraLMatch、TransClean、catalog-forge、product-matcher、pyJedAI 等分析，但没有 `GoldenMatch.md`，因此本次选择 GoldenMatch 继续分析。

当前 Spec 的约束不是一般意义上的“相似商品匹配”，而是一个非常偏 **identity / master data** 的问题：

1. 数据来自雷小安、腕表之家、奢当家三个来源；
2. 规模约 100 万～1000 万，并持续增量；
3. “同一商品”严格定义为 **同一 reference number / 型号**；
4. reference 可能有结构化字段，也可能埋在标题、图片 OCR 中；
5. **误匹配成本极高，precision 优先，宁可漏配、拒识、人工复核**；
6. 可人工标注几百对黄金样本。

GoldenMatch 最值得研究的地方不是它的通用 fuzzy matcher，而是它把实体解析系统拆成了两个不同性质的引擎：

```text
Identity Compute Engine
  block -> score -> cluster
  高吞吐、批处理、Arrow/Rust、可替换执行后端

                 |
                 | ResolutionBatch + evidence
                 v

Identity Control Plane
  stable entity id
  source-record membership
  create / absorb / merge / split
  provenance
  conflict edge
  append-only event
  audit seal
  persisted incremental index
```

这正好对应本需求中两个经常被混在一起的问题：

- “怎样在千万级数据里快速找到值得比较的候选？”——计算问题；
- “一旦认定是同一型号，怎样稳定、可回滚、可审计地维护跨源实体身份？”——状态管理问题。

本次最重要的结论是：

> **不要把 GoldenMatch 的通用 fuzzy/概率聚类结果直接当成腕表同款真值；应该复用它的两引擎边界、持久候选索引、版本化 resolution batch、stable identity、证据边、冲突边和审计状态机，并把最终身份判据替换成 `canonical_brand + verified canonical_reference` 的确定性 Reference-First gate。**

也就是说，GoldenMatch 更适合作为本项目的 **Identity Platform 架构参考**，而不是“训练一个相似度模型然后调高阈值”的现成答案。

---

## 2. GoldenMatch 的核心架构：One Product, Two Engines

GoldenMatch 根 README 和 `context-network` 中把架构明确分为两个引擎。

### 2.1 Identity Compute Engine

职责是纯计算：

```text
raw/normalized records
  -> blocking
  -> pair scoring
  -> clustering
  -> resolution batch + evidence
```

特点：

- 批量边界偏 Arrow；
- 核心性能路径由 Rust authoritative kernel 承担；
- DataFusion、DuckDB、Ray、Spark 等执行后端可以替换；
- 单次调用尽量保持 stateless；
- 大规模场景重点优化候选压缩、分布式计算、内存上限。

项目 README 报告过 100M 行级别的 dedupe 运行，并将大规模分布式路径实现成 blocking-key shuffle、分布式连通分量等模式。这里不应该直接把这个 benchmark 等同于当前腕表数据上的性能保证，但它证明架构上已经把“计算吞吐”和“身份状态”拆开了。

### 2.2 Identity Control Plane

职责是 durable identity state：

```text
stable entity_id
source record ownership
create / absorb / merge / split
field provenance
same_as / conflicts_with evidence
append-only event timeline
config lineage
audit seals
persisted block-key index
```

默认可落 SQLite，生产可用 Postgres；重点不是列式计算，而是：

- 事务；
- 幂等；
- 稳定 ID；
- 可重放；
- 可追责；
- 可撤销；
- 变更历史。

这一层对当前 Spec 特别重要。因为如果仅输出一个每天重新计算的 `cluster_id`，下游会立刻遇到：

```text
今天 cluster 17 是谁？
明天重跑后 cluster 17 还一样吗？
新抓到一个 listing 后为什么并进旧实体？
哪一次规则升级改变了历史实体？
某条 reference 被发现抽错后，怎样拆掉错误合并？
价格、来源、图片等字段到底来自哪个 listing？
```

GoldenMatch 的答案是：**cluster 只是某次计算结果，entity 才是跨运行稳定的业务对象。**

这条边界应该直接继承到腕表系统。

---

## 3. Compute 侧值得借鉴什么：Blocking、Scoring、Clustering 的职责必须分开

GoldenMatch 的通用计算层支持 exact、fuzzy、概率模型、embedding 等多种 scorer，也支持 static、multi-pass、ANN、LSH、sorted-neighborhood 等 blocking。

对当前项目，最应该借鉴的不是“多上几个 scorer”，而是以下职责边界。

### 3.1 Blocking 只回答“谁值得比较”

Blocking 的目标是把：

```text
N x N
```

的比较问题压成一个很小的候选集合，而不负责最终身份决定。

腕表场景应该分两条候选路径。

#### 路径 A：已经有 verified reference

最理想情况不需要 fuzzy blocking：

```text
(canonical_brand_id, canonical_reference)
          |
          v
B-tree / hash / inverted index exact lookup
          |
          v
0 or 1 canonical entity
```

这是主路径。

#### 路径 B：reference 缺失或不可信

才使用：

```text
brand
+ title model-shaped token
+ char n-gram / BM25
+ OCR token
+ optional text/image embedding
        |
        v
Top-K possible references/entities
```

但输出仍然只是候选，不能直接合并。

### 3.2 Scoring 只能排候选，不应拥有最终身份权

GoldenMatch 的通用 matcher 可以产生 fuzzy/probabilistic score；对一般客户去重这是合理的。

腕表的身份定义却已经由业务明确给出：同一 reference。

因此这里应该把 score 的权限降级：

```text
score 可以：
- 排序人工 review 队列
- 帮 reference extractor 选择优先核验对象
- 找 hard negatives
- 做图片/标题冲突检查

score 不可以：
- 覆盖 reference 不一致
- 在 reference 缺失时直接 AUTO_MATCH
- 仅凭 0.999 相似度把两个相邻型号合并
```

### 3.3 通用 Union-Find / WCC 对本项目最大的风险是“传递污染”

GoldenMatch 的 `core/cluster.py` 使用 Union-Find，把高分 pair edge 形成连通分量；同时又有 MST、bottleneck、oversized cluster split 等机制检测弱桥。

这对普通 ER 很合理，但当前需求必须更保守。

例如：

```text
A reference = 116610LN
B reference = 缺失，标题与 A 很像
C reference = 126610LN

A --高分--> B --高分--> C
```

如果只做图连通：

```text
A, B, C -> 一个 cluster
```

这会造成不可接受的传递污染。

腕表版应设置不可被模型覆盖的 cluster invariant：

```text
一个 AUTO entity 中最多只能存在一个 verified canonical_reference
```

等价地：

```text
verified_ref(A) != verified_ref(B)
=> cannot-link(A, B)
=> 无论路径上有多少相似边，都禁止处于同一自动实体
```

因此 GoldenMatch 的 clustering 可用于候选簇和 review，但最终实体 materialization 前必须增加 **Reference Constraint Validator**。

---

## 4. ResolutionBatch：非常值得直接照搬的“计算 -> 身份状态”契约

GoldenMatch 不是让 compute 代码直接随手更新数据库，而是逐步把 compute -> control 的边界抽成 `ResolutionBatch`。

代码：

- `packages/python/goldenmatch/goldenmatch/identity/resolution_batch.py`
- `packages/python/goldenmatch/goldenmatch/identity/resolve.py`

`ResolutionBatch` 是 frozen dataclass，并带 `CONTRACT_VERSION`。其 metadata 包含：

```text
run_id
dataset
matchkey_name
source_pk_col
controller_snapshot
actor
weak_confidence_threshold
field_strategies
config_id / config_schema_version / config_json
flush_rows
contract_version
```

同时携带：

```text
clusters
cluster_frames
df
scored_pairs
pair_score_view
```

然后由：

```text
apply_batch(store, batch)
```

作为 compute -> control 的单一写入口。

这对当前项目很值得直接复用，因为“为什么这次把两个 listing 合起来”必须能重放。

腕表版建议定义：

```text
ReferenceResolutionBatch v1
├── contract_version
├── run_id
├── extractor_version
├── normalization_version
├── reference_catalog_version
├── source_records[]
├── identifier_claims[]
├── candidate_pairs[]
├── decisions[]
├── contradictions[]
├── actor
└── recorded_at
```

其中每个 decision 必须包含：

```text
record_id
candidate_entity_id
brand_id
reference_raw
reference_canonical
reference_role
reference_evidence
verdict: AUTO_MATCH | AUTO_CREATE | REJECT | REVIEW
reason_code
ruleset_version
```

这样后续规则升级时可以精确回答：

> 是 extractor 变了、canonicalization 变了、reference catalog 变了，还是 decision policy 变了？

而不是只留下一个不可解释的最终 cluster。

---

## 5. GoldenMatch 的增量方案：Control Plane 持久化 Blocking Index

这是 GoldenMatch 对当前 Spec 最有直接工程价值的部分之一。

代码：

- `identity/block_index.py`
- `identity/resolve.py::_resolve_via_index`
- `identity/store.py::identity_record_block_keys`

### 5.1 它解决的不是“怎样流式循环”，而是“新增记录不重扫历史全库”

早期所谓 incremental 仍会把历史 corpus 放在内存，再对新增 record 重 blocking。

GoldenMatch 后来明确选择：

```text
Control Plane owns persisted index
Compute queries index
```

形成双向 seam：

```text
new record
   |
   | stateless compute block keys
   v
persisted block index (control read)
   |
   | candidate record ids only
   v
compute score bounded candidates
   |
   v
control commit identity transition
```

`identity_record_block_keys` 表大致是：

```sql
(record_id, entity_id, block_key, pass_sig)
```

并对：

```text
(pass_sig, block_key)
entity_id
record_id
```

建索引。

`compute_record_block_keys()` 重用批处理 blocker 的 `_build_block_key_expr`，确保：

> 批处理时某记录落在哪个 block，增量时用同一条规则也会得到同一个 block key。

### 5.2 `_resolve_via_index` 的实际流程

代码中流程非常清楚：

```text
1. compute_record_block_keys(new_record)
2. store.candidates_by_block_keys(keys)
3. 只从 store 取这些 candidate payload
4. 构造 bounded candidate frame
5. _match_record_rows(new_record, candidate_frame)
6. 用 mini frame 调回 resolve_clusters
7. commit create / absorb / merge
8. 把新 record 自己的 block key 写回 persisted index
```

因此第 N+1 条新记录可以立刻找到前 N 条已进入 store 的记录，而不需要重新 materialize 百万/千万历史数据。

### 5.3 对腕表场景可以进一步简化并变得更可靠

当前业务有一个比通用 blocking 更强的 key：

```text
(canonical_brand_id, verified_canonical_reference)
```

所以主索引甚至不需要复杂 multi-pass：

```sql
CREATE UNIQUE INDEX idx_product_identity_key
ON product_entity(canonical_brand_id, canonical_reference)
WHERE status = 'active';
```

新增 listing：

```text
extract + verify reference
        |
        v
brand + canonical reference exact lookup
        |
  +-----+------+
  |            |
 exists      not exists
  |            |
ABSORB       CREATE
```

对 reference 不完整的长尾记录，再额外维护候选索引：

```text
brand + reference_prefix
brand + model token
brand + char-ngram
brand + OCR token
optional vector/image ANN
```

这等价于 GoldenMatch 的 persisted block index，但严格区分：

- **identity index**：可以触发自动身份绑定；
- **candidate index**：只能触发进一步验证。

这个权限拆分非常关键。

---

## 6. IdentityStore：GoldenMatch 把“实体结果”做成了状态机，而不是一个 CSV

代码：`packages/python/goldenmatch/goldenmatch/identity/store.py`

核心表包括：

```text
identity_nodes
source_records
evidence_edges
identity_events
audit_seals
identity_aliases
identity_record_block_keys
identity_relationships
identity_runs
```

### 6.1 `identity_nodes`：实体 ID 是稳定业务对象

GoldenMatch 的 `new_entity_id()` 生成 UUIDv7-shaped、时间有序 ID。

腕表版建议同样不要用：

```text
cluster_id = 12345
```

做下游主键，而使用：

```text
product_entity_id = UUIDv7
```

并单独把：

```text
canonical_brand_id
canonical_reference
```

作为实体的当前身份 claim。

这样未来 reference canonicalization 规则变化、人工 split/merge 时，下游仍有稳定对象可追踪。

### 6.2 `source_records`：listing 与 entity 的 membership 必须显式存

每条来源记录保存：

```text
record_id
source
source_pk
record_hash
entity_id
payload
first_seen_at
last_seen_at
```

这正适合三源商品：

```text
record_id = 雷小安:xxx
record_id = 腕表之家:yyy
record_id = 奢当家:zzz
```

一个 entity 可以有多条来源 listing，listing 的原始 payload 不应该被 golden record 覆盖掉。

### 6.3 `evidence_edges`：正证据和冲突证据分开

GoldenMatch 明确允许同一 pair 记录：

```text
same_as
conflicts_with
```

而不是所有信息压成一个 score。

腕表版建议把 edge kind 扩展成：

```text
REFERENCE_EXACT
REFERENCE_CONFLICT
BRAND_CONFLICT
IDENTIFIER_ROLE_CONFLICT
TITLE_SUPPORT
OCR_SUPPORT
IMAGE_NEAR_DUP
IMAGE_VISUAL_SIMILAR
MANUAL_ACCEPT
MANUAL_REJECT
```

其中只有少数 evidence kind 有自动 identity 权限。

例如：

```text
REFERENCE_EXACT + verified + same brand
=> eligible for AUTO_MATCH

IMAGE_VISUAL_SIMILAR
=> support only

REFERENCE_CONFLICT
=> hard veto
```

这比把所有特征放进一个加权总分更符合“可漏不可错”。

### 6.4 `identity_events`：所有身份改变都应该是 append-only 事件

腕表版至少应记录：

```text
ENTITY_CREATED
RECORD_ABSORBED
ENTITY_MERGED
ENTITY_SPLIT
REFERENCE_CLAIM_ADDED
REFERENCE_CLAIM_REVOKED
DECISION_OVERRIDDEN
MANUAL_ACCEPTED
MANUAL_REJECTED
CATALOG_VERSION_CHANGED
```

不能只更新当前状态而丢历史。

---

## 7. GoldenMatch 的 create / absorb / merge 思路怎样改造成腕表版

`identity/resolve.py` 会把一次运行的 cluster 与历史 identity 对齐，然后决定：

```text
create
absorb
merge
```

并 upsert record membership、evidence edge、event。

这种“计算结果 -> 状态迁移”方式应该保留，但腕表版需要加更强 precondition。

### 7.1 CREATE

条件：

```text
存在 verified manufacturer reference
AND canonical brand 已确定
AND 当前没有 active entity 使用同一 identity key
```

则创建：

```text
product_entity_id
identity_key = brand_id + canonical_reference
```

### 7.2 ABSORB

只有：

```text
record.brand_id == entity.brand_id
AND record.verified_reference == entity.canonical_reference
AND 没有其他 verified reference conflict
```

才允许把新 listing 加到旧 entity。

### 7.3 MERGE

通用 GoldenMatch 可以根据跨 cluster evidence 合并多个 identity。

腕表版必须更严格：

```text
entity A 与 entity B 只有在：
A.brand_id == B.brand_id
AND A.canonical_reference == B.canonical_reference
```

才允许自动 merge。

如果两个实体 reference 不同：

```text
绝不因为标题、图片、embedding、LLM 很像而 merge
```

### 7.4 SPLIT

人工或后续规则发现某 entity 内出现：

```text
两个 verified canonical references
```

就不是“低置信”，而是 **cluster invariant violation**，应立即：

```text
freeze entity
-> generate review task
-> split memberships
-> append SPLIT event
-> rebuild exact identity index
```

---

## 8. 审计：GoldenMatch 的 hash-chain 很适合高风险 identity 决策

代码：`identity/audit.py`

GoldenMatch 对 append-only event 又加了两层 tamper-evidence：

### 8.1 每个 event 自己做内容 SHA-256

事件的 immutable content 规范化后计算：

```text
entry_hash = sha256(canonical_event_content)
```

这样事后修改某个 event 字段会被检测到。

### 8.2 定期 seal 整段事件历史

`seal_audit_log()` 将 event hash 按 `event_id` 顺序做 left fold，并把当前 root 与上一 seal root 串起来。

可检测：

```text
内容修改
删除
插入
重排
sealed event 缺失
```

对当前需求未必一开始就需要做到密码学级审计，但至少应该直接采用它的设计原则：

> **身份决策不是普通缓存，而是高风险可追溯状态；必须保存 decision evidence 与版本历史。**

如果后续实体匹配影响价格分析、库存映射、交易决策，审计价值会更高。

---

## 9. 哪些 GoldenMatch 能力可以直接参考，哪些不能原样使用

### 9.1 建议直接借鉴

| GoldenMatch 设计 | 腕表系统对应实现 |
|---|---|
| Compute / Control Plane 分离 | 离线/批量候选计算与 durable product identity store 分开 |
| Versioned ResolutionBatch | 版本化 `ReferenceResolutionBatch` |
| Persisted block-key index | 持久 reference/candidate index，支持持续增量 |
| Stable UUIDv7 entity id | 稳定 `product_entity_id` |
| source_records | 三平台 listing membership |
| same_as / conflicts_with | 正证据与 hard contradiction 分离 |
| config lineage | extractor/canonicalizer/catalog/policy 版本可追踪 |
| append-only events | create/absorb/merge/split/revoke 历史 |
| audit seals | 可选高完整性审计 |
| bounded candidate frame | 新增数据只比较 touched candidates，不重扫全库 |
| batch + incremental 共用 resolver | 避免两套逻辑长期漂移 |

### 9.2 不建议原样使用

#### 1. 通用 fuzzy score 作为 AUTO_MATCH

不符合 reference-defined identity。

#### 2. 普通 Union-Find/WCC 直接 materialize entity

存在传递误合并风险。

#### 3. 一个共享 threshold 解决所有品牌、来源和证据类型

“reference exact”“OCR 猜测”“图片相似”不是同一种证据，不能用一个 score 统一授权。

#### 4. zero-config 自动优化作为最终身份策略

本需求更适合显式、白名单式、可审计规则；自动 config 可以帮助候选层，但不能改写 identity gate。

---

## 10. 直接可落地方案：Reference-First Golden Identity

建议把系统拆成六层。

```text
[1] Source Ingest
雷小安 / 腕表之家 / 奢当家
        |
        v
[2] Identifier Claim Extraction
结构化 reference -> title -> OCR -> optional LLM candidate
        |
        v
[3] Claim Validation & Canonicalization
brand / role / format / catalog / context validation
        |
        v
[4] Candidate Retrieval
exact identity index + bounded fuzzy/image candidates
        |
        v
[5] Deterministic Decision Gate
AUTO_MATCH / AUTO_CREATE / REJECT / REVIEW
        |
        v
[6] Identity Control Plane
stable entity + membership + evidence + events + review + audit
```

其中 **第 5 层拥有最终身份权**，而不是模型。

---

## 11. Identifier Claim：不要把一个字符串直接叫 reference

建议新增 `identifier_claim` 表/对象，每次从任何位置抽出的疑似编号都只是 claim：

```text
claim_id
record_id
raw_value
normalized_value
canonical_value
role
role_confidence
evidence_type
evidence_location
extractor_version
catalog_version
validation_status
created_at
```

`role` 至少区分：

```text
MANUFACTURER_REFERENCE
PLATFORM_LISTING_ID
MERCHANT_SKU
SERIAL_NUMBER
COMPATIBLE_REFERENCE
MODEL_FAMILY
UNKNOWN
```

只有：

```text
role == MANUFACTURER_REFERENCE
AND validation_status == VERIFIED
```

才能进入 identity gate。

### 11.1 为什么 role 必须单独建模

典型脏数据：

```text
标题：劳力士 116610LN 适配表带
```

抽取器可以非常准确地抽到 `116610LN`，但它并不是当前商品本体的 reference。

如果系统只有：

```text
reference_extracted = 116610LN
```

就会产生高置信误合并。

因此 reference extraction 和 reference ownership 是两件事。

---

## 12. Canonicalization：只允许“证明安全”的归一化

腕表 reference 是离散 identity token，不应该像自然语言一样 aggressive normalization。

建议两级规范化。

### 12.1 Safe Normalization

允许全品牌通用：

```text
Unicode NFKC
trim
uppercase
全角数字/字母 -> 半角
统一明显等价的空白字符
统一 Unicode dash 到标准 dash（但暂不删除）
```

### 12.2 Brand-Aware Canonicalization

只有经过测试的品牌规则才能：

```text
去某些 separator
补固定前缀
映射历史别名
处理品牌特定 reference 格式
```

所有规则必须有：

```text
rule_id
version
brand_scope
test_cases
approved_by
```

永久保留：

```text
reference_raw
reference_normalized_safe
reference_canonical
canonicalization_rule_id
```

核心原则：

> **宁可两个其实相同的 reference 没被折叠，也不要两个不同 reference 被 canonicalization 错折叠。**

---

## 13. Deterministic Decision Gate：建议直接写成状态机

核心不是一个 `match_score > 0.97`，而是一组有优先级的 gate。

### 13.1 AUTO_MATCH

建议第一版只允许：

```text
same canonical brand
AND both sides have VERIFIED MANUFACTURER_REFERENCE
AND canonical reference exact equal
AND no second verified conflicting reference
AND product role == WATCH_MAIN_ITEM
```

伪代码：

```python
def decide(record, entity):
    if record.brand.status != "VERIFIED":
        return REVIEW("brand_unverified")

    refs = verified_manufacturer_refs(record)
    if len(refs) != 1:
        return REVIEW("reference_not_unique")

    ref = refs[0]

    if entity.brand_id != record.brand_id:
        return REJECT("brand_conflict")

    if entity.canonical_reference != ref.canonical:
        return REJECT("reference_conflict")

    if has_hard_contradiction(record, entity):
        return REJECT("hard_contradiction")

    return AUTO_MATCH("verified_reference_exact")
```

### 13.2 AUTO_CREATE

```text
品牌已验证
+ 唯一 verified manufacturer reference
+ identity index 中不存在该 key
+ 当前商品不是配件/兼容件
```

则建立新 entity。

### 13.3 REJECT

一旦存在：

```text
verified brand conflict
verified reference conflict
identifier role conflict
current-item vs compatible-item conflict
```

直接拒绝。

### 13.4 REVIEW / ABSTAIN

以下情况全部不自动猜：

```text
reference 缺失
只从低质量图片 OCR 到一串编号
标题中有多个疑似 reference
reference catalog 查不到
品牌不确定
OCR 与标题冲突
标题很像但 reference 不同
图片极像但 reference 无法验证
```

“没有结论”是一个合法的一等输出。

---

## 14. 建议的数据模型：把 GoldenMatch IdentityStore 改造成 Product Identity Store

可以直接用 Postgres 实现第一版。

### 14.1 `product_entity`

```sql
product_entity(
  entity_id uuid primary key,
  canonical_brand_id bigint not null,
  canonical_reference text not null,
  status text not null,
  confidence_policy text,
  created_at timestamptz,
  updated_at timestamptz
)
```

并建立：

```sql
unique(canonical_brand_id, canonical_reference)
WHERE status = 'active'
```

### 14.2 `source_record`

```text
record_id
source
source_pk
entity_id nullable
payload_json
record_hash
first_seen_at
last_seen_at
```

### 14.3 `identifier_claim`

保存所有结构字段、标题、OCR、模型提出的编号 claim。

### 14.4 `resolution_decision`

```text
decision_id
record_id
candidate_entity_id
verdict
reason_code
ruleset_version
reference_claim_id
evidence_json
run_id
actor
created_at
```

### 14.5 `evidence_edge`

明确区分 support 与 contradiction。

### 14.6 `identity_event`

append-only 记录所有身份变化。

### 14.7 `reference_catalog`

```text
brand_id
canonical_reference
known_alias
family
format_rule_id
source
source_trust
valid_from / valid_to
```

### 14.8 `review_task`

```text
record_id
candidate_entity_ids
reason
priority
status
reviewer
verdict
```

---

## 15. 千万级规模下的增量路径

千万级不意味着每条记录都要跑大模型或向量检索。

### 15.1 绝大多数“有 verified reference”的记录是 O(1)/O(logN) 路径

```text
new record
 -> extract structured/title reference
 -> validate + canonicalize
 -> exact key lookup
 -> absorb/create
```

数据库索引足够承担这条身份主路径。

### 15.2 只有 unresolved tail 进入昂贵路径

```text
reference missing/ambiguous
 -> brand-scoped candidate retrieval
 -> Top-K reference/entity
 -> OCR/title/model verifier
 -> verified ?
      yes -> exact identity path
      no  -> review/abstain
```

这会把大模型、VLM、image embedding 的成本限制在少量困难记录上。

### 15.3 批处理与实时/微批应该共用同一个 Decision Gate

借鉴 GoldenMatch：

```text
batch resolver
incremental resolver
```

最后都调用同一个 identity transition 函数。

不能做成：

```text
离线一套规则
实时另一套规则
```

否则半年后必然出现语义漂移。

---

## 16. 图片应该怎么放进这个架构

当前 Spec 说图片可用，但“同一商品=同一 reference”决定了视觉不能拥有 identity authority。

建议把图片拆成三类能力。

### 16.1 OCR evidence

优先从：

```text
表背
保卡
吊牌
证书
包装标签
```

抽取 reference-shaped token。

输出仍是 `identifier_claim`，必须经过 role + catalog + context 验证。

### 16.2 Near-duplicate image evidence

pHash / image hash 可发现跨平台复用同一张商品图。

这是很强的辅助证据，但仍可能存在：

```text
搬图
盗图
官方素材复用
同系列模板图
```

不能替代 reference。

### 16.3 Visual retrieval

图片 embedding 可用于：

```text
在缺 reference 时召回可能的品牌/系列/reference 候选
```

最终仍要回到 reference verifier。

---

## 17. 几百对黄金样本应该花在哪里

本需求不是“只有几百标签，所以训练一个二分类器碰碰运气”。

几百条人工预算更适合用于风险最高的地方。

### 17.1 固定 holdout 验收集

重点覆盖：

```text
同品牌同系列、reference 只差 1 位
同 reference 不同写法
平台 SKU 像 reference
配件标题包含目标腕表 reference
多个 reference 同时出现
OCR O/0、I/1、S/5 混淆
品牌字段缺失或错误
标题极像但 reference 不同
图片极像但 reference 不同
```

### 17.2 Hard Negative 集

比随机负样本重要得多：

```text
116610LN vs 126610LN
同系列不同尺寸
同系列不同材质后缀
同外观不同 generation
兼容配件 vs 主表
```

### 17.3 Reference extractor / role classifier 的定向标注

人工应优先解决：

```text
“这串编号是不是当前售卖主体的 manufacturer reference？”
```

而不是只标：

```text
“这两个标题像不像？”
```

---

## 18. 评测指标：不能再以 F1 为发布主门槛

当前业务核心是 false positive 风险。

建议拆成四套指标。

### 18.1 Auto-match Precision

最重要：

```text
AUTO_MATCH 中真实同 reference 的比例
```

目标不是“高于 recall”，而是发布时必须接近可验证的极限。

### 18.2 False Positive Count / Rate

必须单独报告：

```text
总 FP
按品牌 FP
按来源对 FP
按 reference extractor 来源 FP
按 reason_code FP
```

不要把 FP 淹没在 F1 中。

### 18.3 Coverage / Abstain Rate

允许系统通过增加 abstain 来换 precision：

```text
auto_match_coverage
review_rate
abstain_rate
```

### 18.4 Candidate Recall

Blocking/ANN 单独评估：

```text
真实同 reference 是否进入 Top-K
```

candidate recall 高，不意味着可自动 merge；这是两层不同指标。

### 18.5 Cluster Invariant

生产硬门槛：

```text
active entity 内 verified canonical_reference 数量 > 1
```

必须恒为 0。

---

## 19. 上线前必须加的“硬不变量”

建议数据库/服务两层同时 enforce。

### Invariant A

```text
一个 active entity 只有一个 canonical_brand_id
```

### Invariant B

```text
一个 active entity 只有一个 verified canonical_reference
```

### Invariant C

```text
不同 verified canonical_reference 的记录不得被 AUTO_MERGE
```

### Invariant D

```text
candidate score 永远不能覆盖 REFERENCE_CONFLICT
```

### Invariant E

```text
没有 verified manufacturer reference 时，不允许仅凭图片/embedding AUTO_MATCH
```

### Invariant F

每一次 membership 修改都必须产生：

```text
resolution_decision + identity_event
```

没有 reason/version 的写入直接拒绝。

---

## 20. 推荐的服务/模块拆分

第一版不需要复杂微服务，单仓库 Python + Postgres 即可，但代码边界建议按 GoldenMatch 思路拆清楚。

```text
src/
  ingest/
    leixiaoan.py
    watchhome.py
    shedangjia.py

  reference/
    extract_structured.py
    extract_title.py
    extract_ocr.py
    role_classifier.py
    canonicalizer.py
    catalog.py
    validator.py

  candidate/
    exact_index.py
    lexical_index.py
    vector_index.py
    image_index.py

  decision/
    policy.py
    contradictions.py
    invariants.py
    reasons.py

  identity/
    store.py
    resolution_batch.py
    resolver.py
    events.py
    audit.py
    review.py

  evaluation/
    goldset.py
    precision_report.py
    hard_negative_report.py
    regression.py
```

关键依赖方向：

```text
candidate -> decision -> identity
```

不允许：

```text
candidate/model 直接写 entity membership
```

---

## 21. 一条新增记录的完整时序

以奢当家新增一条 listing 为例。

```text
1. Ingest
   source_record = 奢当家:123

2. Extract claims
   structured field -> "126610LN"
   title           -> "126610LN"
   OCR             -> "126610LN"

3. Role validation
   三路均指向当前主表
   role = MANUFACTURER_REFERENCE

4. Canonicalization
   brand = Rolex
   canonical_ref = 126610LN

5. Exact identity lookup
   key = (ROLEX, 126610LN)

6A. entity exists
   -> run hard contradiction check
   -> AUTO_MATCH
   -> append RECORD_ABSORBED
   -> membership points to entity

6B. entity does not exist
   -> AUTO_CREATE
   -> new UUIDv7 entity
   -> append ENTITY_CREATED

7. Persist evidence
   STRUCTURED_REFERENCE
   TITLE_REFERENCE
   OCR_REFERENCE

8. Persist resolution decision
   reason = VERIFIED_REFERENCE_EXACT
   ruleset/extractor/catalog versions

9. Update incremental indexes

10. Async/periodic audit + regression metrics
```

而如果 OCR 得到 `116610LN`、结构化字段是 `126610LN`：

```text
REFERENCE_CONFLICT
-> REVIEW
-> 不绑定任何旧 entity
```

这才符合可漏不可错。

---

## 22. 与 GoldenMatch 原始增量 resolver 相比，腕表版可以更简单

GoldenMatch `_resolve_via_index` 的通用逻辑是：

```text
block keys -> candidate records -> bounded frame -> score -> mini cluster -> resolve
```

腕表版主路径可压缩成：

```text
verified identity key -> entity lookup -> invariant check -> state transition
```

仅 unresolved tail 才保留通用路径：

```text
candidate index -> bounded candidates -> extract/verify -> exact identity key
```

这意味着在百万到千万级数据上，最昂贵的工作不会和总数据量线性绑定，而主要与：

```text
reference 缺失率
reference 冲突率
人工 review 覆盖率
```

相关。

这是比“全量 pairwise + 大模型 rerank”更符合当前业务约束的架构。

---

## 23. 推荐的分阶段落地顺序

### Phase 0：黄金集与数据审计

先做：

```text
reference 字段覆盖率
标题 reference 可抽率
编号角色污染率
品牌覆盖率
一条记录多 reference 比例
典型 OCR 错误分布
```

并构建固定 hard-negative holdout。

### Phase 1：纯确定性 Reference-First MVP

只支持：

```text
结构化 reference
+ 高精度 title regex/parser
+ safe canonicalization
+ brand scope
+ exact identity index
+ stable entity
+ decision/event log
```

不要急着上 embedding。

这版虽然 recall 不高，却最容易验证是否能实现“0 observed FP”。

### Phase 2：Reference Catalog + Role Classification

补：

```text
品牌格式规则
approved aliases
platform SKU / merchant SKU / compatibility role
reference ownership 判断
```

### Phase 3：OCR 辅助

只处理有价值图片，并把 OCR 当 claim，不直接 auto-match。

### Phase 4：Candidate Retrieval for Unresolved Tail

上 BM25/char n-gram/ANN/image retrieval，把候选送回 verifier。

### Phase 5：Active Learning / Selective Model

人工预算集中在：

```text
reference 边界冲突
role 分类
相邻型号 hard negatives
```

模型仍没有覆盖 hard veto 的权限。

### Phase 6：完整 Identity Control Plane

补齐：

```text
merge/split
review UI
config lineage
versioned ResolutionBatch
audit seals
rollback/replay
```

---

## 24. 技术栈建议

### 第一阶段

```text
Python
Polars / PyArrow
PostgreSQL
Parquet/Object Storage
FastAPI（如需 review/API）
```

### 批量重算

1000 万级可以先用：

```text
Polars / DuckDB
```

按品牌/reference 分区即可。

真的需要跨机扩展再考虑：

```text
Ray / Spark
```

GoldenMatch 的经验说明 compute backend 可以替换，不应该把 identity state 绑死在某个分布式框架里。

### 搜索层

优先：

```text
Postgres exact/B-tree
OpenSearch/Elasticsearch BM25 或字符字段
```

只有 unresolved tail 才需要 vector DB / FAISS 类索引。

---

## 25. 主要风险与防护

### 风险 1：错误 reference 被 exact match 放大

`exact` 并不天然安全；如果来源字段本身填错，错误会非常“确定”。

防护：

```text
编号角色分类
brand scope
多证据交叉校验
reference catalog
冲突隔离
来源可信度
```

### 风险 2：canonicalization 过度

把不同型号清洗成同一字符串是最危险的一类 bug。

防护：

```text
safe normalization 与 brand rule 分层
规则版本化
所有 canonical collision 进入测试
```

### 风险 3：传递合并

一条错误边污染整个簇。

防护：

```text
merge 前重新校验 cluster invariant
reference conflict = cannot-link
```

### 风险 4：模型逐渐获得“越权”

系统迭代中很容易出现：

```text
“这个模型 99.9% 了，就让它直接合并吧”
```

防护：

```text
Decision Policy 单独服务/模块
模型只输出 evidence/candidate
数据库写 membership 必须携带允许的 reason_code
```

### 风险 5：历史规则升级后结果不可解释

防护：

```text
ResolutionBatch version
ruleset version
catalog version
extractor version
identity_runs/event lineage
```

---

## 26. 对当前 Spec 的最终建议

GoldenMatch 给当前需求最有价值的启发不是一个新 matching algorithm，而是：

> **把“找到候选”和“维护身份”当成两个不同工程问题。**

当前项目应采用：

```text
Reference-First Matcher
        +
Durable Product Identity Control Plane
```

而不是：

```text
一个万能相似度模型
        +
阈值
        +
cluster_id
```

我建议第一版最终自动身份规则直接收紧为：

```text
AUTO_MATCH =
  canonical_brand 相同
  AND 唯一 verified manufacturer reference 存在
  AND canonical_reference 完全一致
  AND 无任何 hard contradiction
```

其他信息的权限为：

```text
标题相似度     -> 候选/排序
图片相似度     -> 候选/辅助证据
OCR            -> reference claim
LLM/VLM        -> claim/解释/复核建议
人工标注       -> 校准、hard negative、override
```

它们都不能覆盖：

```text
verified reference conflict
```

在这个前提下，再复用 GoldenMatch 的：

```text
版本化 compute->control handoff
persisted incremental index
stable UUID identity
source membership
same_as / conflicts_with evidence
create / absorb / merge / split state transition
append-only event
config lineage
audit seal
```

就能得到一个真正适合“百万～千万级、持续增量、可漏不可错”的二奢腕表实体身份系统。

## 27. 最小可执行 MVP

如果需要马上动手，我会先实现以下 8 个组件，而不是先训练模型：

```text
1. source_record 表
2. identifier_claim 表
3. brand canonicalizer
4. safe + brand-aware reference canonicalizer
5. reference role validator
6. (brand_id, canonical_reference) exact identity index
7. deterministic decision gate + hard veto
8. product_entity + decision/event audit log
```

先用几百条 hard-negative 黄金样本证明：

```text
AUTO_MATCH observed FP = 0
```

再逐步增加 OCR、BM25、embedding、VLM，目标只是在 **不破坏 precision 的前提下提高能够得到 verified reference 的覆盖率**。

这比从 fuzzy matcher 开始再不断调高阈值，更符合当前 Spec 的风险模型，也更容易在持续增量和千万级规模下长期维护。