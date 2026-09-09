# 源码复核后的根工程与模块迁移清单

状态：整合前准备文档；只记录源码分析和迁移边界，不启动整合开发。

本文基于 `AutoRefer` 中已浅克隆或本地保存的源码，重点核对了类、接口、数据结构、任务生命周期和存储边界。`001-main-project-and-reference-modules.md` 保留为早期 Android 优先方案；本文是当前“全域 AI 助手”目标下的更新决策。

## 一、根工程选择

### 方案 A：ClawGUI 单根工程

`ClawGUI` 作为唯一根工程，`clawgui-agent` 承担任务循环、会话、模型 Provider、Episode、远程渠道和设备后端协议。

优势：跨设备、远程电脑、云端模型和 Agent 控制面最完整，Python 适合快速演进轨迹编译、Skill 修订、模型路由和评测。

代价：Android 原生文件权限、OCR、Office、虚拟屏、Shower、端侧模型和后台服务都要重新接入；Python/Kotlin 之间需要维护 IPC、取消、错误和状态合同。

### 方案 B：Operit 单根工程

`Operit` 作为唯一 Android 根工程，保留会话、工具、模型、文件、OCR、文档转换、向量、工作流、Shower 和后台服务。

优势：Android 端落地最快，现有文件和文档能力最多，和 ClosePaw、X-OmniClaw、PocketSearch 的 Kotlin 模块整合成本最低。

代价：跨设备和远程电脑控制面不足；`ToolRegistration.kt`、`MemoryRepository.kt`、权限和 UI 耦合较重；继续扩展会把 Android 单体变成全域系统的瓶颈。

### 方案 C：ClawGUI + Operit 双核心总工程，推荐

总工程采用单一仓库或统一版本编排，但保留两个独立运行面：

```text
ClawGUI ControlPlane
  任务、全局会话、模型路由、远程渠道、电脑/浏览器 Agent、Skill 生命周期
          |
          | 中立合同：Task / Device / Asset / Episode / Skill / Evidence
          |
Operit AndroidRuntime
  Android 权限、文件资产、解析/OCR/向量索引、GUI、虚拟屏、Shower、端侧模型
```

这是“一个总工程、两个边界明确的根运行时”，不是把两个项目的内部代码互相引用。

优势：

- ClawGUI 保留全域控制面，Operit 保留 Android 资产能力，覆盖目标最完整；
- 资产索引和 Android 权限由 Operit 内聚管理，跨设备任务和云端能力由 ClawGUI 内聚管理；
- 手机可独立运行，电脑/云端可作为增强节点；
- 替换 Android 执行实现或远程控制实现时，另一侧只依赖合同，不感知内部类；
- 训练、评测、云端 RAG 和端侧运行时可以分离部署。

代价：

- 两个运行时需要维护版本、连接、取消、重试和状态一致性；
- 必须建立中立合同包，不能让 Python dataclass 或 Kotlin data class 直接成为跨边界事实源；
- 需要处理离线手机、断网恢复、重复提交、事件顺序和权限撤回；
- 构建、发布和调试链路比单根工程复杂。

### 双核心的事实源划分

| 数据或能力 | 唯一事实源 | 对侧访问方式 |
|---|---|---|
| 全局任务、跨设备 Session、模型路由、远程渠道 | ClawGUI ControlPlane | 合同 API、事件和命令 |
| Android DeviceSession、权限和 ROM 能力 | Operit AndroidRuntime | 能力注册和状态上报 |
| 手机原始文件、URI、解析状态、向量和全文索引 | Operit Asset Intelligence | `AssetId`、证据和受控内容流 |
| 跨设备资产目录和检索任务 | ClawGUI Index Gateway | 远程查询，不复制手机原始路径 |
| Skill 版本、晋升和审计 | ClawGUI SkillLifecycle | Operit 只保存已安装运行版本 |
| Android GUI、虚拟屏、输入、截图和动作结果 | Operit Device Runtime | `DeviceBackend` 命令和结果 |
| 电脑/浏览器执行 | ClawGUI Remote Providers | 同一 `DeviceBackend` 合同 |

### 当前结论

选择 **方案 C：ClawGUI + Operit 双核心总工程**。`ClawGUI` 是总控制面根，`Operit` 是 Android 运行根；`ClosePaw` 为虚拟屏执行内核，`KnowAct` 为轨迹编译和验证闸门，`ClawGUI-Skills` 与 `PowerMem` 提供 Skill/Memory 生命周期，`X-OmniClaw`、`PocketSearch`、`Ente`、`Immich` 和 `Operit` 提供个人数字资产能力。

如果当前阶段只允许维护一个可安装应用，先发布 Operit AndroidRuntime；如果目标是全域产品和跨设备编排，必须按方案 C 设计接口，不能把 ClawGUI 或 Operit 的内部状态直接合并。

### “Operit + ClawGUI”作为总根工程的优劣详表

#### 优势

1. **能力覆盖完整**：Operit 已有 Android 文件、OCR、Office/PDF、向量、Shower、工作流和端侧模型；ClawGUI 已有任务控制、远程渠道、电脑/鸿蒙/iOS 后端、模型适配、Episode 和评测。
2. **符合资产系统主线**：资产发现、解析、索引和 Android 权限在 Operit 内聚；跨设备查询、任务编排和模型路由在 ClawGUI 内聚。
3. **运行模式可分离**：手机断网时仍可使用 Operit 本地能力；联网时由 ClawGUI 提供复杂规划、云端模型和远程设备协同。
4. **替换成本可控**：替换 ClosePaw、PocketSearch、Jina、Qwen、Qdrant 或某个解析器时，只影响对应 Adapter/Provider，不要求修改全局任务模型。
5. **便于效果优化**：端侧索引可以快速响应，服务端可以使用更大的多模态模型、Reranker、ColPali 或 EagleRAG 做高精度检索。
6. **便于验证和观测**：手机端负责产生原始证据，控制面负责跨设备汇总、评测、审计和失败分析。

#### 代价

1. **双运行时复杂度**：Kotlin AndroidRuntime 与 Python ControlPlane 需要版本、心跳、取消、重试和事件顺序合同。
2. **状态一致性问题**：全局 TaskSession、Android DeviceSession、AssetIndex 和 Skill 安装版本不能重复存储或互相覆盖。
3. **部署形态增加**：需要同时支持单机 Android、Android+电脑控制面、云端控制面三种模式。
4. **数据传输成本**：跨设备检索可能需要传输缩略图、OCR、向量结果或原始文件，必须由策略决定传输粒度。
5. **调试链路变长**：一个失败可能发生在 Android 权限、解析器、模型、向量库、IPC 或远程设备，必须有统一 `trace_id` 和错误码。
6. **重复能力风险**：两个项目都已有 Session、Memory、Provider 和 Tool 概念，若不先确定事实源，会出现双重记忆、双重模型配置和重复任务执行。

#### 权责分配

| 能力 | ClawGUI ControlPlane | Operit AndroidRuntime |
|---|---|---|
| 全局任务 | 创建、拆解、跨设备编排 | 执行本机子任务并上报状态 |
| 会话 | 跨渠道和跨设备主会话 | Android DeviceSession 和本地执行上下文 |
| 模型 | 云端/电脑模型路由、用量和策略 | 端侧 MNN/Llama/VLM/Embedding Provider |
| 资产 | 跨设备资产目录和查询编排 | MediaStore/SAF、原始 URI、解析和本地索引 |
| 检索 | 多设备结果融合、Reranker 和证据汇总 | 本地 FTS、向量和权限过滤 |
| Skill | SkillIR、版本、晋升、审计和分发 | 已安装 Skill 的执行和本地状态 |
| Android 操作 | 下发 `DeviceBackend` 命令 | 无障碍、Virtual Display、Shower、输入和截图 |
| 远程电脑 | Gateway、Channel、Remote Agent | 不承担 |
| 数据权限 | 全局策略和跨设备授权 | Android 系统权限和本地可读范围 |

#### 必须采用的边界合同

```text
ControlPlane -> TaskCommand -> AndroidRuntime
AndroidRuntime -> TaskEvent -> ControlPlane
AndroidRuntime -> AssetQuery -> AssetResult
ControlPlane -> ModelRequest -> ModelProvider
AndroidRuntime -> EvidenceRef -> ControlPlane
```

合同必须包含：`request_id`、`trace_id`、`task_id`、`session_id`、`device_id`、`asset_id`、`contract_version`、`deadline`、`cancel_token`、`status`、`error_code` 和 `evidence_refs`。

#### 三种部署形态

| 形态 | 说明 | 适用阶段 | 主要风险 |
|---|---|---|---|
| Android 单机 | Operit 内嵌 ControlPlane Adapter，手机本地完成任务和资产检索 | P0/P1 原型和离线场景 | 跨设备能力有限，模型和索引资源受手机约束 |
| 手机 + 电脑控制面 | ClawGUI 运行在电脑，Operit 作为 Android Provider | P1/P2 主力开发形态 | IPC、网络断开和状态同步 |
| 云端控制面 + 多设备 | ClawGUI 服务端统一调度多个 Operit、电脑和浏览器节点 | P2/P3 全域场景 | 权限、数据上传、计费和多租户隔离 |

#### 高内聚判断

方案 C 只有在以下条件同时满足时才成立：

1. ClawGUI 不直接读 Android 文件路径，只通过 `AssetId` 和 `EvidenceRef` 获取结果。
2. Operit 不直接修改 ClawGUI 的全局 Session、Skill 版本和模型路由状态。
3. Task、Asset、Episode、Skill、Evidence 采用中立合同，不直接共享 Python/Kotlin 类。
4. 本地索引和远程索引允许不同实现，但必须绑定同一 `asset_id` 和版本信息。
5. 任一侧停止或断网时，另一侧可以得到明确的 `paused`、`cancelled`、`expired` 或 `failed` 状态。

#### 总根工程的最终判断

如果“总根工程”指一个统一代码仓库，采用方案 C；如果“根运行时”指一个实际安装包，Android 端采用 Operit，控制面采用 ClawGUI；如果强行把两者合并成一个进程和一套全局状态，短期启动简单，长期会违背高内聚、低耦合目标。

### 第一阶段实际改造根工程

当前要在一份仓库上开始删减、修改和增加时，固定：

```text
仓库：E:\autoZhifa\AutoRefer\operit
分支：zhifagui-integration
角色：Android 端数字资产系统的第一阶段实现根
```

选择 Operit 作为第一阶段代码根，不等于放弃方案 C，而是先在 Android 运行面完成资产系统主链：

```text
Operit
  -> AssetRecord / AssetSource / AssetParser
  -> OCR / 文档解析 / 图片向量 / 本地索引
  -> 混合检索 / 证据返回 / 资产动作
```

ClawGUI 在这一阶段只作为外部控制面和协议参考，暂不复制进 Operit；后续通过 `TaskCommand`、`TaskEvent`、`AssetQuery`、`AssetResult` 和 `EvidenceRef` 接入远程控制面。

#### 第一阶段允许的改造范围

1. 在 Operit 内新增 `domain/asset`、`capability/asset`、`infrastructure/index` 和 `adapter` 包。
2. 将 `MemoryRepository` 中的文档分块、Embedding、向量索引和查询逻辑逐步提取到资产索引端口。
3. 将 `ToolRegistration.kt` 中的文件、OCR、文档、搜索工具拆为独立能力注册器。
4. 将 `VirtualDisplayManager.kt` 保留为旧适配器，虚拟屏核心接入 ClosePaw 前不删除现有入口。
5. 接入 X-OmniClaw 相册扫描、PocketSearch 图片索引、Operit OCR 和文档解析，所有实现都通过 `AssetSource`、`AssetParser`、`EmbeddingProvider`、`VectorIndex` 和 `TextIndex`。

#### 第一阶段禁止的改造范围

1. 不把 ClawGUI 的 nanobot、Web UI、RL 或 Eval 整体复制进 Operit。
2. 不把 ClosePaw、Aries、X-OmniClaw 或 PocketSearch 的应用 UI 整体复制进 Operit。
3. 不在未建立行为基线、依赖扫描和回归测试前删除 Operit 旧实现。
4. 不让资产索引模块直接依赖 GUI Agent、Shower、具体模型或具体向量数据库。

#### 第一阶段完成标志

只有同时满足以下条件，才进入下一阶段接入 ClawGUI 控制面：

- Android 授权范围内的图片、媒体、文档、文本和代码可以登记为 `AssetRecord`；
- OCR、文档解析、图片向量和全文索引可以独立运行并记录版本；
- 增量扫描、失败重试、索引重建和权限撤回可观测；
- 文字、图片和组合条件能够返回原始 URI、来源和证据位置；
- 资产结果可以交给 Operit 工具继续查看、复制、编辑、分享或发送；
- 旧工具和新资产接口可以并行运行，且核心领域层不依赖第三方实现。

### 一点五、记忆系统路线与端云分工

#### 记忆不是一张表

全域助手至少需要五类记忆，必须分库存储、分生命周期和分事实源：

| 记忆层 | 记录内容 | 事实源 | 典型检索 | 生命周期 |
|---|---|---|---|---|
| 工作记忆 | 当前会话、最近观察、待确认动作、临时上下文 | Operit/ClawGUI 当前会话 | 精确键、时间、会话 | 分钟到数小时 |
| 个人长期记忆 | 用户偏好、稳定事实、联系人习惯、任务经验 | ClawGUI `MemoryService`，端侧保留受控镜像 | 关键词、向量、关系、时间衰减 | 月到年 |
| 资产记忆 | 文件、图片、视频、音频和文档的元数据、解析结果、向量、证据位置 | Operit `Asset Intelligence` | FTS、字段过滤、模态向量、RRF | 与资产同生命周期 |
| 知识记忆 | 从资料编译出的主题页、实体页、引用图和冲突记录 | 云端或用户指定的 LLMWiki 适配器 | 全文、链接图、向量、引用 | 持续维护、可回溯 |
| 技能记忆 | 已验证的操作流程、参数、前置条件、失败修订 | ClawGUI `SkillLifecycle` | Skill 名称、能力标签、设备条件 | 版本化、可晋升/回滚 |

动作树缓存、最近成功轨迹和 GUI 探索记录属于执行缓存或 Episode，不得直接当作长期记忆；只有经过结果验证、去冗余和 promote 才能进入技能记忆。

#### 可选技术路线

1. **端侧单体路线**：SQLite/FTS5 或 ObjectBox 保存工作记忆、资产元数据和小规模向量；适合离线、低延迟和单设备，但模型、索引大小和跨设备同步能力受限。实现参考 `Operit MemoryRepository`、`X-OmniClaw MemoryIndex` 和 `PocketSearch`。
2. **云端记忆服务路线**：将长期记忆、关系图、时间衰减、冲突融合和跨设备查询集中到 PowerMem 类服务；适合多设备和大模型，但需要网络、账号、成本和数据上传策略。
3. **LLMWiki 编译路线**：原始来源进入 `raw`，LLM 增量维护 Markdown 主题/实体页、交叉链接、引用和冲突；查询优先读已编译知识，避免每次从原文重新推理。微软 `llmwiki` 明确采用这种“持久 Wiki 介于原始来源和查询之间”的架构；`lucasastorian/llmwiki` 还提供 MCP、图谱和来源回链。它适合知识记忆与可审计上下文，不适合承载实时资产状态或设备动作。
4. **分层混合路线（推荐）**：端侧维护工作记忆和资产索引，控制面维护长期记忆与技能，LLMWiki 作为知识源适配器；查询由 `MemoryRouter` 按问题类型并行召回，再用统一证据合同合并。

#### 端侧职责

- 在用户授权范围内发现 MediaStore、SAF、应用共享目录和已连接设备资产；保存原始 URI、哈希、权限状态和增量游标。
- 执行轻量解析、缩略图、OCR、音频转写和端侧视觉/文本向量；记录模型版本、解析版本和失败原因。
- 维护本地 FTS、字段过滤和小规模 ANN；断网时完成查询、证据定位和设备内查看/复制/编辑/分享等动作。
- 对上传内容执行用户可配置的范围、脱敏和采样策略；权限撤回时立即停止读取并使索引失效。
- 保存当前会话所需的短期上下文和设备能力，不保存跨设备全局事实的第二份权威副本。

#### 云端/控制面职责

- 维护跨设备任务、全局 Session、长期个人记忆、Skill 版本和审计日志；向手机下发幂等命令并接收事件。
- 承担大文件、长视频、复杂 PDF/Office、重 OCR/VLM、批量重嵌入、跨模态 Reranker 和全局图检索。
- 执行记忆抽取、去重、冲突检测、时间衰减、Experience 到 Skill 的晋升，以及离线评测和模型路由。
- 运行 LLMWiki/PowerMem 类知识编译服务，保留来源、引用、版本和可回滚快照；对手机只返回必要的知识片段和证据。
- 负责跨设备索引联邦和结果融合，不直接假设另一设备的本地路径可访问。

#### 统一边界和数据流

```text
手机原始资产 -> Operit AssetIndex -> 本地证据/向量
        |             |
        | 授权投影    | AssetQuery / EvidenceRef
        v             v
云端索引网关 -> MemoryRouter -> PowerMem/LLMWiki/SkillLifecycle
                                      |
                                      v
                              TaskCommand -> 手机或电脑执行
```

合同至少包含 `AssetId`、`MemoryId`、`Scope`、`SourceRef`、`EvidenceRef`、`ModelVersion`、`EmbeddingVersion`、`Freshness` 和 `PermissionState`。端侧与云端可以各自缓存，但只能有一个事实源；网络断开、重复提交、权限撤回和版本冲突必须有显式状态。

#### LLMWiki 的纳入结论

纳入，但定位为 **KnowledgeMemory Adapter**，不是根工程、不是手机资产数据库，也不是 Agent 的即时会话存储。优先吸收：来源到主题/实体页的增量编译、Markdown 可读可审计格式、页面间链接、引用回链、冲突标记、MCP 查询接口和会话记忆分离。删除或改造：VS Code/桌面 UI、固定文件夹假设、仅关键词的小规模索引和未经验证的后台自动改写。所有写入先经过 `SourceRef`、版本和验证队列，避免 LLM 直接覆盖原始资产或已确认事实。

#### 本阶段决策

总工程仍采用 `ClawGUI + Operit` 双核心：Operit 是手机资产和端侧记忆运行根，ClawGUI 是长期记忆、Skill 和跨设备控制面；PowerMem 提供长期记忆策略，LLMWiki 提供知识编译适配器，X-OmniClaw/Operit/PocketSearch 提供端侧索引实现。先定义 `MemoryRouter`、`AssetIndex` 和 `KnowledgeSource` 合同，再选择具体数据库或模型；不得让任何单一项目的 Memory 表成为全域事实源。

## 二、五类迁移策略

“直接照搬”指以源码模块为实现起点后改接口、包名、错误和存储；“吸收”只迁移协议、算法、数据字段或边界；所有迁移都必须经过统一合同和设备验证。

### 1. 可以直接照搬，再做适配

| 模块需求 | 源码起点 | 必改项 | 优势与风险 |
|---|---|---|---|
| GUI 任务循环和 Provider | `ClawGUI/clawgui-agent/phone_agent`、`nanobot` | 接入统一 `TaskSession`、取消和审计 | 控制面现成；需隔离渠道和模型私有状态 |
| 虚拟屏生命周期 | `ClosePaw` `VirtualDisplayPlatform`、Shizuku transport | 接入 `DeviceBackend`，补 API/ROM 能力探测 | 生命周期和清理边界清晰；仍受 ROM 二级屏限制 |
| 相册增量扫描 | `X-OmniClaw/agent/memory/gallery/AlbumScanner.kt` | 改为统一 `AssetRecord` 和权限事件 | MediaStore 游标可复用；不覆盖应用私有目录 |
| 端侧图片向量索引 | `local-photo-search/.../MediaStoreScanner.kt`、`PhotoIndexStore.kt`、`ImageEmbeddingIndexer.kt` | 接入统一 `AssetRecord`、任务队列和模型 Provider | 已有增量、失败重试、模型版本和 QNN/NNAPI/CPU 选择；需补中文模型和权限撤回 |
| 图片摘要与隐私过滤 | `X-OmniClaw/ImageMemorySummarizer.kt`、`PrivacyFilter` | 摘要写入 `AssetStore`，保留原始 URI 引用 | 可快速建立“找文件”能力；摘要不能替代原图向量 |
| 轨迹状态合同 | `KnowAct/state_contract.py`、`data.py` | 转为统一 `Episode`/`SkillIR` 字段 | 前置条件、动作和证据明确；需适配 Kotlin/IPC |
| Skill 包和版本校验 | `ClawGUI-Skills/schema.py`、`package.py`、`verifier.py` | 与端侧安装格式和签名字段统一 | 版本、失败案例和审计完整；不能跳过回放验证 |
| 记忆数据合同 | `PowerMem/src/powercontext/.../protocols.py`、`models.py` | 去除 Server/CLI 宿主，接入主仓储 | 来源、证据、版本冲突有明确边界；需统一数据所有权 |
| 文件路由和视觉编码协议 | `EagleRAG/eagle_rag/ingest/router.py`、`visual_encoder.py` | 替换 Milvus/Celery 为 Provider/Index 接口 | 文本/视觉分路清楚；服务端实现不能直接放手机 |
| 文件解析和混合检索 | `semantic-finder/ingest`、`retrieval/hybrid_search.py` | 替换 LanceDB/桌面 watcher | 解析、分块、BM25+ANN+RRF 可复用；移动端需周期校正扫描 |
| 远程 Embedding Provider | `clip-as-service` `client.encode/index/search/rank`、ONNX/Torch executor | 改为统一 `ModelProvider`，增加超时和隐私策略 | 批量/流式/多后端边界清晰；不引入 Jina/DocArray 运行时 |

### 2. 可换语言重写后使用

| 模块 | 原实现 | 重写落点 | 保留内容 | 取舍 |
|---|---|---|---|---|
| 轨迹编译和去冗余 | KnowAct Python `trajectory_codegen.py`、`flat.py` | Kotlin 端轻编译器，复杂修订保留 Python 服务 | 参数化、动作合并、状态约束和回放 | 端侧可离线；服务端迭代更快 |
| 端侧图文索引 | PocketSearch Dart `clip_service`、`index_service` | Kotlin + MNN/ONNX + 轻量向量库 | MobileCLIP、HNSW、标量过滤、增量索引 | 需重新验证模型输入、ABI、内存和中文召回 |
| 文件监听和摄取 | semantic-finder Python watcher/ingest | Kotlin `ContentObserver` + WorkManager + SAF | 去抖、hash、批量写入、失败重试 | 移动事件不完整，必须保留周期扫描 |
| 本地记忆索引 | PowerMem Python Core | Kotlin/SQLite/向量库 Adapter | Memory、Citation、Handoff、版本冲突 | 端侧低延迟，但策略编排可留在控制面 |
| 设备后端 | ClawGUI Python `DeviceBackend` | Android Kotlin Provider；电脑端保留 Python | observe、execute、preflight、cancel、error | 统一协议降低耦合，但增加 IPC |
| Skill Runtime | ClawGUI-Skills Python | Kotlin 运行 `SkillPackage` | 检索、版本、失败案例和审计 | 模型驱动修订仍放控制面 |

### 3. 只能学习思路

| 来源 | 只学习的内容 | 不能作为底座的原因 |
|---|---|---|
| `deep-student` | 强类型上下文、取消传播、VFS-RAG、审批和迁移 | Rust/Tauri 学习工具链过重，业务边界不匹配 |
| `EagleRAG` 服务端拓扑 | Celery、插件、引用回链、多租户治理 | Milvus、Redis、MinIO 不属于 Android 核心 |
| `ColPali` | 视觉文档多向量和 late interaction | 向量数量、推理和存储成本高 |
| `RAG-Anything` | PDF、图片、表格、公式联合解析 | 服务端运行环境和依赖过重 |
| `ClawGUI-RL/Eval` | 训练、评测和 Infer-Judge-Metric 闭环 | 不进入手机生产运行时 |
| `omni-retrieval` | 跨文本、图片、音频、视频统一检索 | 模型和资源成本不适合手机默认开启 |

### 4. 只吸收一两点特性

| 来源 | 吸收点 | 不吸收内容 |
|---|---|---|
| `Aries-AI` | 输入签名探测、OpenGL 双路帧、启动后迁移、IME 焦点 | 不复制 API 34-only 反射路径和完整显示引擎 |
| `Ruto-GLM` | `session -> displayId -> Job`、SurfaceControl 镜像、输入坐标换算 | 不复制聊天 UI、固定 marker 指令和未完成 IME |
| `X-OmniClaw` | 精确停止、图像摘要、deeplink/Intent、引用快照 | 不让 Markdown 行为记录直接晋升 Skill |
| `MobiAgent` | ActTree、动作缓存、历史路径排序 | 缓存只产生候选，不能绕过验证 |
| `Zafiro` | ToolRegistry、MCP、Python、Skill 按需加载 | 不复制 Xposed/厂商语音助手宿主 |
| `Qwen3-VL-Embedding`、`WeMM` | 统一多模态向量、Reranker、MRL 维度裁剪 | 2B/8B 默认不放在普通手机 |
| `Qdrant`、`sqlite-vec` | payload 过滤、命名向量、SQLite 内嵌向量字段 | 不直接引入完整服务端或未验证 JNI/ABI |

### 5. 删减后可以使用

| 来源 | 删除或隔离 | 保留并改造 |
|---|---|---|
| `Operit` | UI 展示、非目标工具、重复权限、分散 ROM 特判、重复记忆入口 | OCR、文档转换、文件工具、Shower、向量和工作流 Adapter |
| `ClosePaw` | 产品 UI 和与主业务绑定的入口 | 虚拟屏状态机、Shizuku、Binder 清理、错误可观测性 |
| `Aries-AI` | 主 UI、重复输入实现、API 30-33 不可用入口 | 帧分发、IME、任务迁移和能力探测思路 |
| `Ruto-GLM` | Compose 聊天层、模型管理 UI、固定协议 | Display/Job/Input 低层执行能力 |
| `Zafiro` | Xposed 宿主、厂商语音接管、产品 UI | 工具注册、MCP、Python、Skill Runtime |
| `TIDY` | Fragment/UI、主线程索引循环和固定英文 CLIP 模型 | ONNX 图文编码、Room 数据结构和跳过截图规则 |
| `ClawGUI` | RL、评测和非目标渠道的生产依赖 | Agent、Provider、Session、Episode、远程协议 |

## 三、源码核查新增证据

本节只记录本轮读取到的实际实现，结论优先于项目 README 的功能宣称。

| 项目与实际源码 | 源码事实 | 迁移判定 |
|---|---|---|
| `MobiAgent/agent_rr/action_cache/tree.py`、`action.py` | `ActionTree` 支持 exact/fuzzy 两种任务匹配；节点按动作合并任务，`try_find_shortcuts` 从共享路径生成 2~3 步 Shortcut；`target_elem_changed` 用裁剪图像 SSIM 或 OmniParser 判断目标是否变化；Qwen3 Embedder/Reranker 参与召回 | 只吸收动作树、缓存候选和目标变化检测；不直接复制 Python/Torch 状态，候选必须进入统一验证和晋升流程 |
| `MobileAgent/Mobile-Agent-E/MobileAgentE/agents.py`、`inference_agent_E.py` | `InfoPool` 把计划、感知、动作历史、结果、错误、子目标和未来任务集中在内存；Manager、Operator、ActionReflector、Tips/Shortcut Reflector 和 Retriever 由提示词串联；Shortcut 仅写入内存字典，执行依赖 ADB 和固定等待时间 | 只吸收“计划—执行—反思—Tips/Shortcut”闭环；重写为 `TaskSession` 和持久化 Skill 候选，不把提示词产物当作已验证技能 |
| `DroidAgent/droidagent/app_state.py`、`memories/working_memory.py`、`memories/task_memory.py` | 以静态 `AppState` 保存活动、GUI 状态、临时 Toast；WorkingMemory 区分 ACTION/OBSERVATION/CRITIQUE；TaskMemory 将任务评估、反思和可复现动作写入存储 | 吸收工作记忆分层和动作前后证据；删除全局静态状态，改为会话作用域并接入 `Episode` |
| `DroidAgent/scripts/make_script.py` | 将实验 `task_execution_history` 转为 UIAutomator2 Python 脚本；定位优先 text/content-desc/resource-id，最后退回坐标；脚本写入固定包名和原始屏幕宽高，并提供 activity 等待 | 删减实验数据、固定屏幕尺寸和坐标回退后使用；补前置/后置验证、密文参数和尺寸归一化 |
| `Ghost-in-the-Droid/gitd/skills/trace_to_steps.py` | 通过 actuating-tool allow-list 从聊天工具轨迹蒸馏可回放步骤；按 `tool_id` 或顺序匹配结果；读屏和元工具会被丢弃；明确把参数化和剪枝留给后续 review/commit | 直接照搬“allow-list + 轨迹蒸馏”边界；后续必须接入参数提取、冗余删除、设备验证和结果验证 |
| `Ghost-in-the-Droid/gitd/skills/base.py` | `Element` 按 content-desc/text/resource-id/class/坐标降级查找；`Action.run` 执行 precondition、execute、postcondition、rollback 和重试；`Workflow` 负责唤醒、返回主页、启动应用和弹窗清理 | 直接照搬执行合同和定位链，改为 `DeviceBackend`；坐标只能作为最后降级路径 |
| `Ghost-in-the-Droid/gitd/skills/checkpoint.py`、`auto_creator.py` | checkpoint 支持人工 resume/abort、屏幕条件自动恢复、超时和可注入轮询；BFS 探索保存 XML 结构 hash、截图、元素和转移，并限制深度/状态数 | 直接吸收 checkpoint 和 BFS 状态图；接入统一取消、权限和 ROM 能力探测 |
| `Ghost-in-the-Droid/registry/*.yaml`、`registry/scripts/validate_skill.py` | Skill 元数据含版本、包名、导出动作/工作流、弹窗检测、默认参数和测试设备；校验器检查字段、YAML、Python 语法和危险调用 | 直接吸收 SkillPackage 字段和校验器；加入来源、权限、验证证据和版本兼容矩阵 |
| `AppAgentX/data/State.py`、`explor_auto.py` | LangGraph State 同时保存截图、页面 JSON、动作、工具结果、错误和 fallback 标志；完成判定先让 LLM 生成标准，再对最近截图做字符串包含式 `yes/complete` 判断；`data/graph_db.py` 用 Neo4j 建 Page-Element-Action 图 | 只能学习状态图和“标准生成—判定”分层；不复制 LLM 字符串判定、Neo4j 生产依赖和全局配置 |
| `AppAgentX/data/data_storage.py`、`vector_db.py`、`chain_evolve.py` | 轨迹落成 Page/Element/Action 图，元素图像用 ResNet50 向量写入远程向量库；链路模板化由 LLM 输出 Pydantic 结构并写 Neo4j | 只吸收元素—动作—页面关系和结构化输出；端侧改用统一 Asset/SkillStore，结果必须有设备证据 |
| `closepaw/.../VirtualDisplayPlatform.kt`、`VdLifecycleArbiter.kt` | 用 `Stopped/Running/Draining/Broken` 状态和运行租约串行化启停；Binder 死亡进入 Broken；停止前排空操作、移除任务、释放 display 和 ImageReader | 直接照搬生命周期状态机；需把 API/ROM 能力探测放到 Adapter，不能把 Android 类型带入控制面 |
| `closepaw/.../ShizukuDisplayTransport.kt`、`ShizukuInputTransport.kt` | API 33 分支反射 `VirtualDisplayConfig.Builder`，API 31-32 有两种旧签名和 packageName 重试；保存 `IVirtualDisplayCallback` token；输入反射 `injectInputEvent` 并检查布尔结果 | 直接照搬签名探测和回退框架；`VirtualDisplayConfig.Builder` 在低版本的隐藏/公开边界必须实测，不能仅按 SDK 常量判断 |
| `closepaw/.../VirtualDisplayInputInjector.kt`、`ShizukuRuntimeGateway.kt` | 反射 `InputEvent.setDisplayId` 后读回验证；失败时执行 `input -d` shell；长按/滑动取消时发送 `ACTION_CANCEL`；通过 HiddenApiBypass 解锁隐藏 API | 直接吸收“探测—验证—回退—取消”链；隐藏 API 只能封装在设备适配器，保留审计和失败原因 |
| `operit/.../VirtualDisplayManager.kt` | 以进程单例保存一个 `VirtualDisplay`、一个 `ImageReader` 和一个 displayId；创建时使用物理屏尺寸减状态栏；释放和截图有异常兜底 | 只能删减后使用；不能作为多虚拟屏底座，需移除单例、补生命周期租约和完整尺寸策略 |
| `operit/.../MemoryRepository.kt`、`VectorIndexManager.kt` | ObjectBox 保存 Memory/DocumentChunk，文件名带 profile 和向量维度；关键词、标签、语义、关系边权重和 RRF 混合排序；Embedding 可调用云服务 | 直接吸收检索字段和维度迁移逻辑；存储和 UI 图模型必须拆到 `MemoryAdapter`，不能让 Asset、Memory、Skill 共用一张事实表 |
| `operit/.../ToolPermissionSystem.kt`、`ToolRegistration.kt` | DataStore 保存全局/单工具 ALLOW、ASK、FORBID；ASK 通过 Overlay 等待用户；工具注册集中且同时处理 UI 可见性、代理调用和权限检查 | 吸收权限等级和人工确认合同；删除 UI 状态副作用，统一由控制面做策略和审计 |
| `ClawGUI/clawgui-agent/phone_agent/integration/contracts.py` | 不可变 `TaskSession`、`Episode`、`EpisodeStep`、`VerificationResult` 和 `SkillPipeline`，明确 candidate/validated/promoted/deprecated | 直接作为总控制面的领域合同；Android/第三方实现只能实现 Adapter，不修改合同 |
| `ClawGUI/clawgui-agent/phone_agent/device_factory.py` | ADB/HDC/iOS 通过延迟加载模块共享同一设备操作接口，但保留全局 factory | 吸收设备能力接口；重写为会话作用域 Provider，禁止全局设备类型污染并发任务 |
| `ClawGUI/clawgui-skills/clawgui_skills/verifier.py`、`evolution.py` | Verifier 只读取脱敏轨迹和有限截图；无模型时按重复动作/定位错误给出反馈；修订通过受限文件工具、版本快照和失败案例写回 | 直接吸收验证信息边界、受限修订和版本快照；仍需接入真机结果和 ROM 证据 |
| `Aries-AI/.../ShizukuVirtualDisplayEngine.kt`、`VirtualAsyncInputInjector.kt` | 虚拟屏创建会扫描 `createVirtualDisplay` 方法，输入会探测 2/3 参数 `injectInputEvent`；但 `buildVirtualDisplayConfig` 无论设备版本都反射 `VirtualDisplayConfig.Builder`，输入注入为异步 best-effort 且吞掉结果 | 只吸收候选签名扫描、帧分发和 IME 处理；API 34 Builder 缺口、输入结果和低版本回退必须重写 |
| `Ruto-GLM/.../RutoAiTasker.kt`、`InputManagerService.kt`、`DisplayManagerServiceStub.kt` | 会话状态完成后按 `displayId` 建独立 `Job`；API 33 调 `injectInputEventToTarget`，旧版本调 `injectInputEvent`；显示通过 `ConcurrentHashMap` 管理，但 ImageReader 释放和输入返回值不完整 | 吸收 display/job 隔离和输入坐标构造；删除聊天产品层，补资源释放、注入结果和 ROM 探测 |
| `zafiro/libs/okia/.../ToolRegistry.kt`、`agent-runtime/.../ToolManager.kt`、`app/.../SkillFileRepository.kt` | 工具描述与执行器分离，支持 Local/MCP/Python；Skill 文件有路径解析、启用状态、冲突和导入；Shell 有命令规则、锁定状态和人工确认 | 直接吸收工具/Skill 注册合同和 Shell 安全策略；不复制 Xposed 宿主和 UI，改接控制面权限 |
| `X-OmniClaw/.../MemoryIndex.kt`、`SessionManager.kt`、`MessageCompactor.kt`、`SkillInstaller.kt` | SQLite+FTS5+逐条余弦向量混合检索；JSONL 会话索引和写锁；上下文压缩有超时、质量审计和回滚；Skill 安装有 ZIP 路径检查、SHA-256 和 lock 文件 | 直接吸收文件/会话/安装的局部实现；大规模向量检索、权限和版本事实源必须统一到控制面合同 |
| `local-photo-search/.../MediaStoreScanner.kt`、`PhotoIndexStore.kt`、`ImageEmbeddingIndexer.kt` | 使用 API 33/34 分级媒体权限；以 `date_modified + media_id` 增量扫描；SQLite 保存 pending/indexed/failed、重试、模型版本和向量偏移；编码器按 QNN/NNAPI/CPU 探测；索引器缓存会话并支持协程取消 | 直接照搬端侧图像索引骨架；补全文件类型、中文/多语模型、权限撤回和统一 `AssetRecord` |
| `local-photo-search/.../VectorStore.kt`、`TextEmbeddingSession.kt`、`ModelRegistry.kt` | 向量文件使用 mmap 连续读取；查询 Embedding 有内存+SQLite 缓存；模型目录通过 `model.json` 动态发现，模型文件不打包进 APK | 直接吸收轻量向量存储、模型注册和维度合同；需增加原子提交、崩溃恢复和模型签名 |
| `PocketSearch/lib/services/clip_service.dart`、`vector_store.dart` | MNN 同时加载图像/文本 CLIP 并保持 512 维共享空间；Zvec schema 带版本文件，升级时删除旧库重建；标量过滤字段写入哨兵值，避免缺失字段绕过过滤 | 重写为 Kotlin/MNN 或 ONNX Provider；吸收 schema 迁移、过滤字段完整性和模型常驻策略 |
| `PocketSearch/lib/services/index_service.dart`、`search_service.dart`、`query_rewriter.dart` | 先同步相册 ID 删除陈旧记录，再用缩略图规避 HEIC 解码失败；批量编码后定期 optimize；查询重写失败回退原词，并支持日期/地理过滤 | 删掉 Flutter UI 后使用索引流程和查询合同；中文语义重写、隐私开关和端侧网络降级需重写 |
| `tidy-ocr/.../ORTImageViewModel.kt`、`ImageEmbeddingRepository.kt`、`ImageEmbeddingDao.kt` | 以 ONNX `visual_quant` 对 MediaStore 图片逐张编码，Room 保存 `id/date/embedding`，跳过 Screenshots；索引循环在 ViewModel 启动 | 只能作为最小 ONNX+Room 基线；必须移到后台队列、增加失败状态、增量 hash 和模型版本 |

## 四、按模块需求确定底座和吸收清单

| 模块需求 | 直接照搬底座 | 吸收模块 | 优势 | 主要代价和验收点 |
|---|---|---|---|---|
| 全域任务、会话、上下文 | ClawGUI Agent/Session | PowerMem Handoff、X-OmniClaw 精确停止、deep-student 取消传播 | 跨设备编排最完整 | 统一任务状态、取消、恢复和审计 |
| Android GUI 执行 | ClawGUI `DeviceBackend` 合同 | ClosePaw 生命周期、Aries/Ruto 帧和输入、Zafiro 无障碍 | 设备后端可替换 | 每个 ROM 验证显示、输入、截图、清理 |
| 多虚拟屏并发 | ClosePaw 执行内核 | Ruto display/job 绑定、Aries 帧分发 | 并发任务隔离 | API 30-33、厂商二级屏和资源释放 |
| 文件、图片和文档 | local-photo-search 图像索引 | Operit OCR/Office、X-OmniClaw 摘要隐私、EagleRAG 路由 | 端侧可用并可扩展云端 | 权限、中文模型、索引增量和证据回链 |
| 长期记忆和交接 | PowerMem 协议 | X-OmniClaw MemoryIndex、semantic-finder RRF/MRL | 来源、版本和检索边界清楚 | 不允许多套事实源和跨模块直写数据库 |
| 轨迹自动编译 | KnowAct `SkillIR`/状态合同 | DroidAgent 探索、MobiAgent ActTree、X-OmniClaw 行为事件 | 轨迹可参数化和回放 | 去冗余后必须重新执行并验证结果 |
| Skill 生成、失败修订、版本 | ClawGUI-Skills | KnowAct `validate/promote`、PowerMem Experience、X-OmniClaw Lock | 生命周期和质量闸门完整 | 只允许 `raw/candidate/validated/promoted/deprecated` |
| 端侧/云端模型 | ClawGUI Provider | X-OmniClaw VLM/STT、EagleRAG VisualEncoder、clip-as-service | 可按隐私、延迟、费用和算力路由 | 超时、降级、用量和数据上传审计 |
| 远程电脑 Agent | ClawGUI Gateway/Channel | KnowAct Windows backend、Zafiro MCP、Operit Shell | 手机和电脑共用能力路由 | 心跳、重连、取消、授权撤回 |
| 训练与评测 | ClawGUI-RL/Eval | KnowAct 验证日志、ROM 兼容矩阵 | 生产与实验分离 | 不把训练依赖打入 Android 包 |

## 五、迁移优先级与退出条件

| 优先级 | 先迁移的模块 | 原因 | 进入下一阶段的条件 |
|---|---|---|---|
| P0 | ClawGUI 合同、ClosePaw 生命周期、Ghost Action/Workflow/trace-to-steps、KnowAct 验证状态 | 这些模块决定任务、设备、轨迹和 Skill 的稳定边界 | 合同冻结；启停、输入、截图、清理和 candidate/promote 流程均可观测 |
| P1 | Operit OCR/文档/权限 Adapter、X-OmniClaw MediaStore/Session、local-photo-search 图像索引、Zafiro Tool/MCP | 直接形成 Android 端文件、会话和工具能力 | 权限可撤回；索引可增量重建；工具可取消；第三方替换不改控制面 |
| P2 | Aries/Ruto 多虚拟屏增强、PowerMem Memory/Handoff、semantic-finder/EagleRAG 混合检索、远程电脑 | 提升兼容性、记忆质量和跨设备能力，但依赖边界较多 | ROM/API 矩阵通过；存储和 IPC 合同稳定；失败可降级 |
| P3 | Qwen3/WeMM、ColPali、Qdrant、RL/Eval、跨模态统一检索 | 主要提升效果或规模，不是最小可用链路 | 通过召回、延迟、费用、隐私和资源评测后按需启用 |

## 六、高内聚、低耦合约束

依赖方向固定为：

```text
ControlPlane -> Contracts -> CapabilityRouter
                              -> DeviceAdapter
                              -> DataIndexAdapter
                              -> SkillAdapter
                              -> ModelProvider
```

约束：

1. 第三方项目只能进入 `adapter` 或 `provider`，核心领域对象不依赖第三方包名、私有类和私有数据库。
2. 任务、设备、资产、向量、Skill 版本和模型用量各自只有一个事实来源。
3. 模型只输出结构化意图和动作，不直接调用 Android API；Skill 只调用 `Action`/`Capability`。
4. ROM 特判统一放在能力探测和设备适配器，不散落到 Agent、Skill 和 UI。
5. 缓存和历史路径只能产生候选，设备验证、结果验证和重复回放决定是否晋升。

替换 `ClosePaw`、`Operit`、`Aries`、`Ruto` 或 `X-OmniClaw` 时，控制面、`SkillIR` 和资产合同不应修改。

## 七、当前准备状态和后续顺序

已完成：

- 已浅克隆并登记主要参考项目；
- 已核对上述项目的实际源码入口、核心类和依赖边界；
- 已形成五类迁移策略、根工程取舍和模块清单；
- 已识别 API/ROM、权限、生命周期、索引维度和模型资源约束。

后续只在用户明确开始开发后执行：

1. 固定 ClawGUI 控制面和 Operit Android Adapter 的接口合同；
2. 为 ClosePaw/Aries/Ruto 建立 API 与 ROM 兼容实验矩阵；
3. 为 Asset、Memory、Episode、SkillIR 和 Verification 建立最小数据合同；
4. 先接入可观测适配器，再逐个替换实现；
5. 通过回归、真机和检索评测后，才删除 Operit 或其他旧路径。
