# Android 端数据库架构选型

日期：2026-09-13  
状态：设计决策，尚未进入源码整合。

## 一、决策摘要

采用“一个权威事实库，多个可重建索引”的最小架构：

```text
Android: Room -> SQLite/FTS5 -> ObjectBox HNSW
Cloud:   PostgreSQL + pgvector + relation tables
```

- `Room` 是 Android 数据访问层，底层仍是 SQLite；不直接在业务层使用原始 SQLite API。
- Room/SQLite 保存唯一事实：资产、解析结果、证据、关系、任务、权限和同步事件。
- SQLite FTS5 保存全文和关键词索引；不再默认同时维护 AppSearch，避免两套全文事实。
- ObjectBox 只保存端侧向量及 HNSW 索引；向量损坏或模型升级时可以删除重建。
- 云端首版使用 PostgreSQL、内置全文检索和 pgvector；只有真实规模或图查询瓶颈出现后才增加 Qdrant/Graphiti。

Room 官方定位是 SQLite 的抽象层，提供编译期 SQL 检查、DAO、事务和迁移支持，并建议优先使用 Room 而不是直接操作 SQLite API。[Room 官方文档](https://developer.android.com/training/data-storage/room)  Room 支持 FTS3、FTS4 和 FTS5。[Room 全文检索](https://developer.android.com/training/data-storage/room/defining-data)

## 二、Android 端最小组件

| 组件 | 唯一职责 | 是否首版采用 |
|---|---|---|
| Room + SQLite | 权威实体、事务、迁移和关系数据 | 是 |
| Room FTS5 | 文件名、OCR、转写、摘要和正文检索 | 是 |
| ObjectBox HNSW | 图片、文本、音频、视频向量近邻检索 | 是 |
| AppSearch | 系统级或大规模全文派生索引 | 暂缓，性能不足时再启用 |
| sqlite-vec | SQLite 内嵌向量实验适配器 | 暂缓，不能作为首版前提 |
| Qdrant Edge | 端侧独立向量引擎 | 暂缓，待 Beta 稳定和真机验证 |

ObjectBox 已提供 Java/Kotlin 的 HNSW 向量索引，适合 Android 端低延迟检索。[ObjectBox Vector Search](https://docs.objectbox.io/on-device-vector-search)

AppSearch 是 Android 官方的高性能本地全文搜索方案，支持结构化 Schema、过滤和部分向量搜索；但首版引入它会增加第二套全文索引、迁移和一致性维护，因此仅作为可替换 `TextIndex` 适配器。[AppSearch](https://developer.android.com/develop/ui/views/search/appsearch)

## 三、权威数据模型

原始文件和媒体保留在文件系统、MediaStore 或 SAF 授权 URI 中，数据库只保存引用和派生数据：

```text
Asset          original URI, hash, type, size, timestamps, permission state
AssetPart      page, paragraph, frame, audio segment
Representation OCR, transcript, summary, structured extraction
Embedding      asset part, modality, model and vector version, dimension, vector
Entity         person, place, app, project, concept
Relation       subject, predicate, object, time and confidence
Evidence       source ref, byte range, page, timestamp, bounding box
IndexJob       parser, model, status, retry count and error
ChangeLog      operation id, device sequence, operation and tombstone
```

`Asset`、`Memory`、`Entity`、`Episode` 和 `Skill` 不共用一张事实表。全文、向量和图关系都必须能通过 `asset_id` 或 `evidence_id` 回链原始内容。

Embedding 表必须支持多模态和多版本，不在 `Asset` 中固定 `image_vector`、`text_vector` 等列：

```text
asset_part_id
modality
model_id
model_version
embedding_version
dimension
quantization
vector
```

## 四、端侧知识图谱

Android 端不部署 Neo4j、完整 Graphiti 或其他重量级图数据库，只使用 Room 关系表：

```text
Entity(id, type, canonical_name)
Relation(id, subject_id, predicate, object_id, confidence, source_ref, evidence_ref)
Event(id, occurred_at, ingested_at, valid_from, valid_to)
```

端侧关系用于轻量关联、过滤和证据回链；云端再将关系投影到 Graphiti 或其他图引擎，处理实体消歧、时间关系、多跳查询和跨设备合并。

每条由模型推断的关系必须区分 `EXTRACTED`、`INFERRED` 和 `AMBIGUOUS`，并保留来源、证据、模型版本和人工修正记录。

## 五、端云同步

不复制数据库文件，不把向量库当作事实源，采用事件同步和索引重建：

```text
Room ChangeLog -> sync gateway -> PostgreSQL -> pgvector/full text/graph projections
```

同步事件至少包含：

```text
op_id
device_id
sequence
asset_id
content_hash
operation
tombstone
permission_scope
parser_version
model_version
embedding_version
```

规则：

1. 手机是原始文件、URI、权限和本地证据的事实源。
2. 云端是跨设备任务、长期个人记忆和全局关系的事实源。
3. 全文、向量和图投影均可删除后重建。
4. 删除、权限撤回和重新授权都必须产生可追踪事件。
5. 使用 `op_id`、设备序号、内容哈希和墓碑记录保证幂等与冲突处理。

## 六、云端首版

云端首版只部署 PostgreSQL + pgvector：

- PostgreSQL 保存跨设备资产投影、长期记忆、任务、权限、同步事件和关系表。
- PostgreSQL 全文检索处理精确词、OCR、转写和结构化字段。
- pgvector 提供文本、图片、音频和视频向量的 HNSW/IVFFlat 检索。
- 一个 SQL 事务可以同时应用权限过滤、字段过滤、全文排序和向量召回。

`pgvector` 已支持 HNSW、IVFFlat、向量过滤和迭代索引扫描，足以覆盖首版服务端规模。[pgvector 官方仓库](https://github.com/pgvector/pgvector)

只有出现以下瓶颈才增加专用服务：

| 触发条件 | 增加组件 | 仍保留的事实源 |
|---|---|---|
| 向量规模、吞吐或隔离要求超过 pgvector | Qdrant | PostgreSQL |
| 时间图、多跳关系和实体演化复杂 | Graphiti/图数据库 | PostgreSQL 关系和事件 |
| 端侧向量性能或跨平台需求明确 | Qdrant Edge | Room/SQLite |
| Android 全文检索实测不足 | AppSearch | Room/SQLite |

## 七、未来 AI 和 RAG 兼容性

不把系统命名或设计成单一 RAG 数据库。未来检索器可能组合：

```text
exact + lexical + sparse + dense + multi-vector + temporal + graph + tool + evidence readback
```

所有访问都通过稳定接口：

```text
AssetStore
TextIndex
VectorIndex
GraphStore
EvidenceReader
MemoryRouter
```

更换模型、向量库、图引擎、长上下文模型或 Agent 查询策略时，只替换适配器，不修改资产和证据合同。摘要、向量和图关系永远是派生数据，原始文件和结构化解析结果必须可回读。

## 八、性能和演进约束

1. 先测量再增加组件：以真实手机资产集测试 FTS 查询、向量召回、索引构建、耗电、内存和恢复时间。
2. 分离写入和索引：事务先写 Room，再异步更新 FTS、向量和图投影，失败可重试。
3. 控制向量版本：模型或维度变化生成新索引，双索引校验后再切换，不能覆盖旧版本。
4. 控制端侧规模：端侧优先保存热数据和必要向量，冷数据和重模型处理交给云端。
5. 保持可降级：向量不可用时仍能使用全文、字段和关系检索；云端不可用时端侧仍能查询和执行设备动作。

## 九、最终决策

```text
Android  = Room + SQLite/FTS5 + ObjectBox HNSW
Cloud    = PostgreSQL + pgvector + relation tables
Deferred = AppSearch, sqlite-vec, Qdrant Edge, Qdrant, Graphiti
```

这套方案只保留一个端侧权威数据库和一个云端权威数据库，避免全文库、向量库和图数据库互相重复存储事实；同时通过适配器保留未来接入专用向量引擎、时间图和新型检索模型的空间。
