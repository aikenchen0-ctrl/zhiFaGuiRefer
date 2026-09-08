# 主工程与高价值参考模块决策

状态：整合前准备基线。后续参考项目分析、模块取舍和验证结论持续更新到本文。

## 目标

构建以 Android 为第一运行端、可扩展到电脑、浏览器和云端的全域 AI 助手。系统至少包含：任务与会话、上下文、长期记忆、Skill、GUI Agent、虚拟显示、远程电脑、端侧与云端模型、工作流，以及个人数字资产的多模态理解、索引、检索和后续执行。

## 主工程决定

选择 `Operit` 作为产品根工程。

理由：它已经具备原生 Android 应用壳、会话运行时、模型 Provider、端侧模型、工具注册、MCP、工作流、定时任务、文件工具、OCR、Office/PDF 转换、向量记忆、Shower 和虚拟屏入口。以它为根工程，可以保留最多现成能力，并减少 Python 与 Android 双宿主造成的状态、权限和生命周期重复。

其他主工程候选的定位：

| 项目 | 定位 | 不作为根工程的原因 |
|---|---|---|
| `ClawGUI` | 全域控制面、远程渠道、跨设备协议、GUI 模型和评测参考 | Android 文件、权限、后台服务和本地工具仍需大量补齐 |
| `X-OmniClaw` | 端侧会话、上下文、相册记忆和行为记录参考 | 工具、文档处理、虚拟屏和工程闭环不如 Operit 完整 |
| `ClosePaw` | Android 虚拟显示和执行内核 | 缺少全域助手的模型、工具、记忆和跨设备控制面 |

## 目标架构

```text
Operit Android 根工程
├── Task Control
│   ├── Task / Session / Run / Step
│   ├── Context / Memory / Skill
│   └── Permission / Audit / Model Router
├── Capability Runtime
│   ├── Android GUI / Virtual Display
│   ├── File / Browser / Shell / MCP
│   ├── Remote Computer Agent
│   └── Cloud Compute API
├── Skill Compiler
│   ├── Episode / SkillIR
│   ├── Candidate / Validate / Promote
│   └── Version / Failure Revision / Replay
├── Personal Asset Intelligence
│   ├── Asset Discovery / Parser / OCR / ASR
│   ├── Text / Image / Audio / Video Embedding
│   ├── Hybrid Retrieval / Rerank / Provenance
│   └── Open / Copy / Edit / Share / Send
└── Evaluation
    ├── ROM / API Compatibility
    ├── GUI Task Success
    ├── Skill Replay Success
    └── Retrieval Recall / Precision / Latency
```

## 模块吸收总表

“照搬来改”表示以指定源码为实现起点；“吸收思路”表示只迁移协议、算法或数据格式，不复制整个工程。

| 模块需求 | 根模块或照搬来改 | 吸收思路 | 主要取舍 |
|---|---|---|---|
| Android 产品壳 | Operit `app` | X-OmniClaw 前台服务、状态和诊断 | 复用率最高，但必须拆分大单体和权限边界 |
| 任务与会话 | Operit `ChatRuntimeHolder`、聊天和前台服务 | ClawGUI/nanobot Session；X-OmniClaw 会话隔离和精确停止 | 统一为 `Task -> Run -> Step`，避免聊天会话直接承载执行状态 |
| 模型接入 | Operit `llmprovider`、MNN、llama | ClawGUI Model Adapter；X-OmniClaw Agent/VLM/STT 分离 | 保留端侧和 API 双路径，输出统一结构化结果 |
| 工具和工作流 | Operit Tool Registration、Workflow、Terminal、Browser、File、MCP | Zafiro Tool Registry、Python、Skill | 工具按能力包注册，权限不再由工具自行扩张 |
| 虚拟显示 | ClosePaw `VirtualDisplayPlatform` 和 Shizuku transports | Aries OpenGL、输入签名、IME、任务迁移；Ruto 多 display 会话；Operit Shower AIDL | 用 ClosePaw 生命周期替换当前薄封装，保留 Shower 服务边界 |
| GUI Agent | ClawGUI/GUIClaw 的设备协议和模型动作适配 | X-OmniClaw 观察-推理-执行-验收；Zafiro 无障碍树 | GUI Agent 只调用统一 `DeviceBackend`，不得直接依赖具体 ROM 实现 |
| Episode 轨迹 | ClawGUI tracer 和 KnowAct trajectory recorder | X-OmniClaw 无障碍事件、Operit 工具调用日志 | 保留动作前后状态、截图、工具结果、模型快照和验证证据 |
| 轨迹编译 | KnowAct `trajectory_codegen.py`、`data.py`、`flat.py`、`state_contract.py` | DroidAgent 探索、MobiAgent 动作树 | Python 编译器先作为独立服务接入，稳定后再决定是否移植 Kotlin |
| 验证与晋升 | KnowAct `shortcut_validation.py`、`deeplink.py` | ClawGUI verifier；X-OmniClaw Success Monitor | 只有设备验证、结果验证和重复回放通过后才能 promote |
| Skill 生命周期 | ClawGUI-Skills `schema/package/store/verifier/evolution` | KnowAct 状态契约；X-OmniClaw SkillLock；PowerMem Experience/Skill | `SkillIR` 是中间格式，`SkillPackage` 是正式存储格式 |
| 长期记忆 | Operit MemoryRepository 作为 Android 存储入口 | PowerMem 提取、融合、时间衰减、Experience/Skill；X-OmniClaw 混合检索 | 用户事实、任务经验、程序技能和证据分库存储 |
| 远程电脑 | GUIClaw `DeviceBackend` 和 Windows backend | ClawGUI Gateway/Channel；Operit HTTP bridge | 新增能力协商、心跳、断线恢复、取消和授权范围 |
| 训练评测 | ClawGUI-Eval/RL 独立部署 | KnowAct 设备验证日志、ROM 矩阵 | 不进入手机生产运行时 |

## 五类实施清单

### 1. 可以直接照搬，再做接口适配

这些模块边界清晰、依赖相对局部，适合作为第一批实现起点。直接照搬不等于不测试，复制后必须替换包名、错误类型、存储路径和权限入口。

| 模块需求 | 直接照搬来源 | 需要改动 | 优势 | 风险 |
|---|---|---|---|---|
| Android 相册增量扫描 | X-OmniClaw `AlbumScanner`、`AlbumImageRecord` | 改为 Operit AssetRecord；接入 MediaStore 权限和任务队列 | 已处理时间戳+媒体 ID 游标 | 只覆盖 MediaStore，不覆盖其他应用私有目录 |
| 图片视觉摘要与隐私过滤 | X-OmniClaw `ImageMemorySummarizer`、`ImageMemoryPrivacyFilter` | 将摘要写入统一资产表，不再只写 Markdown | 已有图片压缩、VLM 调用和敏感字段过滤 | 摘要不是原图向量，不能替代图像检索 |
| 文档 OCR 与格式转换 | Operit `OCRUtils`、`DocumentConversionUtil` | 拆出独立 `AssetParser` 接口，限制大文件和超时 | PDF、Office、图片已有实现 | 当前与 Operit 工具和 UI 依赖较深 |
| Android 向量检索入口 | Operit `MemoryRepository`、`VectorIndexManager` | 从“记忆”改为通用资产索引，增加资产类型和来源字段 | 已有 HNSW、语义搜索和调试查询 | 需处理向量维度变化和索引重建 |
| 技能包安全安装 | X-OmniClaw `SkillInstaller`、`SkillLockManager` | 接入正式 SkillPackage 和签名/来源字段 | 已有 ZIP 路径校验、版本和哈希记录 | 不能直接信任外部 Skill |
| GUI 设备接口 | KnowAct `DeviceBackend`、动作和观察数据类 | 改为 Kotlin/IPC 可序列化协议 | 设备后端职责已经清晰 | Python 协议不能直接被 Android 调用 |

### 2. 可以用另一种语言重写后符合需求

这些模块的算法或数据结构有价值，但原实现语言、运行时或依赖不适合 Operit Android 根工程。

| 模块需求 | 原实现 | 重写方案 | 保留内容 | 取舍 |
|---|---|---|---|---|
| 轨迹转 SkillIR | KnowAct Python `trajectory_codegen.py`、`data.py` | Kotlin 实现核心规范化；复杂编译保留 Python 服务 | 动作参数、状态契约、截图和去重规则 | Kotlin 端易部署；Python 端迭代更快 |
| 轨迹去冗余 | KnowAct Python `flat.py`、`_merger.py` | Kotlin 实现候选删除、动作合并和回放接口 | 语义相似度、序列相似度和状态约束 | 删除动作必须真实回放，不能只按字符串相似度 |
| 端侧图片检索 | PocketSearch Dart `clip_service`、`index_service`、`vector_store` | Kotlin + MNN/ONNX + Zvec/ObjectBox | MobileCLIP 编码、HNSW、标量过滤、进度和 Agent API | 跨语言后需重新验证模型输入、ABI 和内存 |
| ClawGUI 设备后端 | ClawGUI Python `DeviceBackend`、Windows/ADB/HDC | Kotlin 端实现 Android Provider；电脑端保留 Python/独立服务 | observe、execute、preflight、取消和错误合同 | 统一协议降低跨设备耦合，但增加 IPC |
| ClawGUI Skill Runtime | ClawGUI-Skills Python | Kotlin 端运行 SkillPackage；Python 保留生成和复杂修订服务 | metadata 检索、版本、失败案例、审计 | 端侧轻量，服务端保留模型驱动修订 |
| 全文件监听与索引 | Semantic Finder Python watcher/ingest | Kotlin WorkManager + ContentObserver + SAF；桌面端保留原实现 | 文件监听、双向量、ANN+BM25+RRF | 移动端事件不完整，必须有周期校正扫描 |
| 多模态文档服务 | EagleRAG Python | Operit 端采用轻量解析器；云端用 EagleRAG | 文本/视觉分路、证据回链和 MCP | 手机不运行 Milvus、Redis、MinIO 全套基础设施 |

### 3. 只能学习思路，不能直接作为实现底座

这些项目的核心价值是架构或模型思想，直接搬入会把不匹配的基础设施、训练假设或运行时一起带进来。

| 项目 | 学习内容 | 不直接照搬的原因 |
|---|---|---|
| EagleRAG | 文本/视觉双管线、RRF、图扩展、引用溯源、MCP | 服务端微服务和基础设施过重 |
| RAG-Anything | 文本、图片、表格、公式的统一文档关系 | 解析链和运行环境偏服务器 |
| ColPali | 页面级视觉多向量和 OCR-free 文档检索 | 多向量存储和模型推理成本高 |
| PixelRAG | 视觉切片、页面渲染和视觉证据 | 更适合复杂文档服务，不是手机相册索引 |
| PowerMem | 记忆提取、融合、时间衰减、Experience/Skill 双层 | 记忆策略需与 Operit Session/Memory 模型重合并 |
| WeMM-Embedding | 文本、图片、视频和视觉文档统一向量空间 | 先验证模型服务和向量维度，再决定是否生产使用 |
| Qwen3-VL-Embedding | 高精度跨模态 Embedding 和 Reranker 配合 | 端侧模型体量和推理成本高 |

### 4. 只吸收一两点特性

| 来源 | 只吸收的特性 | 不吸收的部分 |
|---|---|---|
| Aries-AI | 输入签名探测、任务启动后 display 校验/迁移 | 不复制其 API 34 反射路径和整套显示引擎 |
| Ruto-GLM | `session -> displayId -> Job` 的并发绑定 | 不复制其消息框架和固定指令格式 |
| Ente | 图片预处理、增量索引、设备状态调度 | 不复制照片同步、账号和整套 UI |
| Immich | Asset 元数据、OCR/CLIP 分工、时间/地点/人脸过滤 | 不复制服务端 PostgreSQL/VectorChord 部署 |
| Zafiro | 工具按需加载、MCP/Skill/Python 扩展 | 不复制其厂商语音助手接管和整套 Agent Runtime |
| Operit Shower | 独立 AIDL 特权服务、截图/输入服务边界 | 不把 Shower 与所有业务工具继续耦合 |
| Jina CLIP 服务 | 图片/文本统一编码 API 和批处理 | 不将云端服务协议当作手机本地存储协议 |
| ObjectBox | Android HNSW 和对象/向量同库 | 不在未完成数据迁移评估前替换现有存储 |

### 5. 删减后可以使用

以下不是删除功能，而是先隔离非核心路径，待依赖扫描和回归测试通过后再移出默认构建。

| 来源 | 可删减内容 | 保留内容 | 预期收益 |
|---|---|---|---|
| Operit | `dragonbones`、`fbx`、`mmd` 等非 Agent 核心展示模块 | `terminal`、`mnn`、`llama`、`quickjs`、`showerclient` | 减少 APK 体积、构建时间和初始化负担 |
| Operit | 市场展示、示例工具、重复 UI 适配 | 模型配置、工具注册、会话、工作流 | 降低产品逻辑和工具注册耦合 |
| Operit | 默认捆绑的高权限能力 | 按能力启用 Root、Shizuku、无障碍、文件和虚拟屏 | 缩小权限面，便于 ROM 诊断 |
| Operit | 重复的旧记忆格式和单用途索引入口 | 通用 AssetRecord、统一 Memory/Skill API | 避免多套数据格式并存 |
| X-OmniClaw | 只写行为 Markdown 的旧路径 | 事件采集和 SkillIR 转换 | 防止“记录”直接绕过验证晋升 |
| ClawGUI | RL 训练运行时和评测 UI 的生产依赖 | Agent、Provider、Episode、远程渠道 | 减少手机端资源和部署复杂度 |
| EagleRAG | Milvus/Redis/MinIO 的手机端依赖 | 文本/视觉分路、引用字段、MCP 合同 | 保留方法，不引入重型基础设施 |

## 根工程优劣势

### 选择 Operit 的优势

1. Android 原生宿主已经存在，权限、前台服务、模型、工具和 UI 不需要重新搭建。
2. 已有文件、OCR、Office/PDF、向量记忆、Shower、工作流和定时任务，和目标需求重合度最高。
3. `minSdk=26`，可以覆盖更多 Android 设备，再通过能力探测决定功能是否启用。
4. Kotlin 原生整合 X-OmniClaw、ClosePaw、PocketSearch 的 Android 逻辑，少一层跨运行时通信。

### 选择 Operit 的代价

1. 工程规模大，工具注册、权限和业务 UI 耦合，第一阶段必须先拆边界。
2. 已有 ROM 问题记录较多，虚拟屏、无障碍、文件和截图能力必须建立逐 ROM 证据。
3. 现有向量记忆主要围绕 Memory 对象，改成全域 Asset Index 会涉及数据迁移。
4. GUI Agent、轨迹编译和 Skill 晋升不是完整闭环，需要接入 KnowAct 和 ClawGUI。

### 为什么不选 ClawGUI 为根工程

ClawGUI 的全域控制面、远程渠道、模型适配和 GUI Agent 较强，但 Android 原生文件权限、文档解析、后台服务、Shower 和本地工具要重新接入。它应作为 Operit 的控制面、远程渠道、模型和评测参考，而不是产品宿主。

## 开始改造前的基线

本地工作分支：`AutoRefer/operit` -> `zhifagui-integration`。

第一轮只建立接口和观测基线，不删除生产代码：

1. 为当前 Operit 的聊天、工具、工作流、文件、记忆、Shower 和虚拟屏建立行为清单。
2. 定义 `TaskSession`、`AssetRecord`、`Episode`、`SkillIR`、`VerificationResult` 和统一错误类型。
3. 用适配器包裹旧模块，保持现有调用入口不变。
4. 新增资产扫描、轨迹编译和 Skill 晋升时，先写失败测试，再实现最小路径。
5. 只有替代路径通过回归测试和 ROM/设备验证后，才删除旧实现。

## 高内聚、低耦合架构约束

### 模块边界

每个模块只拥有一种主要变化原因，并且只拥有自己负责的数据写入权。第三方项目只能进入 `adapter` 或 `provider` 边界，不能直接修改核心领域对象。

```text
domain
  Task / Session / Run / Step
  Asset / Episode / SkillIR / Verification

application
  TaskOrchestrator / CapabilityRouter / Policy

capability
  device / asset / memory / skill / model / remote

adapter
  Operit / ClosePaw / Aries / X-OmniClaw / ClawGUI / PocketSearch

infrastructure
  database / vector store / file store / queue / network

presentation
  Android UI / Web UI / remote channel
```

### 允许的依赖方向

```text
presentation -> application -> domain
capability -> domain
adapter -> capability + domain
infrastructure -> domain contracts
```

禁止反向依赖：

- `domain` 不依赖 Android、Shizuku、数据库、网络、模型 SDK 或 UI；
- `application` 不直接调用 `VirtualDisplay`、`MediaStore`、MNN、向量数据库或第三方 API；
- `skill` 不直接读写设备文件和截图，必须通过 `AssetStore`；
- `model` 不直接改变 Session、Skill 或 Asset 状态；
- `ui` 不直接操作 Shower、数据库和文件系统；
- 任一第三方适配器不允许引用另一第三方适配器的内部类。

### 核心端口

核心模块只依赖以下接口：

```text
TaskRepository
SessionRepository
AssetSource
AssetParser
EmbeddingProvider
VectorIndex
TextIndex
DeviceBackend
FrameSource
InputTransport
SkillCompiler
SkillVerifier
SkillStore
ModelProvider
RemoteAgentTransport
```

Operit、ClosePaw、Aries、X-OmniClaw、PocketSearch 和 ClawGUI 都通过这些端口接入。替换某个实现时，核心模块不应感知实现来自哪个项目。

### 数据所有权

| 数据 | 唯一所有者 | 其他模块访问方式 |
|---|---|---|
| 任务和会话状态 | `TaskRepository` / `SessionRepository` | 查询、事件和命令 |
| 文件资产元数据 | `AssetRepository` | `AssetId` 查询 |
| 原始文件和缩略图 | `AssetStore` | URI、流和受控导出 |
| 文本/OCR/转写索引 | `TextIndex` | 查询接口 |
| 图片/音频/视频向量 | `VectorIndex` | 向量查询接口 |
| 设备显示和输入 | `DeviceBackend` | 会话作用域命令 |
| Skill 版本和晋升状态 | `SkillStore` | SkillIR 和版本 API |
| 模型配置和用量 | `ModelProvider` / `UsageRepository` | 推理请求和统计事件 |

禁止多个模块直接写同一文件、表或索引。跨模块更新通过事务、命令或事件完成。

### 事件与异步任务

扫描、OCR、Embedding、视频抽帧、Skill 修订和远程执行均属于异步任务，统一使用：

```text
JobId
TaskId
SessionId
source_version
status
progress
retry_count
error_code
evidence_refs
```

任务必须可暂停、取消、恢复和重试；不能由 UI 协程直接持有长时间资源。后台任务完成后发布领域事件，查询方通过 Repository 获取最终状态。

### 第三方适配器隔离

| 适配器 | 只允许暴露 | 不得泄漏 |
|---|---|---|
| ClosePaw | `DeviceBackend`、`FrameSource`、`InputTransport` | 内部 Binder、隐藏 API 反射类 |
| Aries | API 版本探测、GL 帧结果、任务迁移结果 | `VirtualDisplayConfig.Builder` 的无条件路径 |
| X-OmniClaw | 相册记录、会话记录、记忆查询、Skill 文档 | 自己的全局 Prompt 和工具注册表 |
| PocketSearch | 图片向量、索引进度、Top-K 结果 | Flutter、Dart 和 Zvec 内部对象 |
| ClawGUI | Provider、Episode、远程设备和 Skill 编译服务 | nanobot 内部 Session 和渠道状态 |
| Operit Shower | 截图、视频、输入和显示会话 | 业务工具和 UI 状态 |
| EagleRAG | 文档解析、视觉块、引用和检索结果 | Milvus、Redis、MinIO 内部连接 |

### 组合根和依赖注入

只允许在应用启动组合根创建具体实现：

```text
OperitApplication
  -> repositories
  -> infrastructure
  -> adapters
  -> capability services
  -> application orchestrator
  -> UI / remote endpoints
```

业务类通过构造函数接收接口，不使用跨模块静态单例。需要全局生命周期的对象由应用容器管理，并明确关闭顺序。

### 高内聚低耦合验收

每次引入参考模块前必须回答：

1. 该代码只解决一个领域问题吗？
2. 它的输入输出能否用稳定接口描述？
3. 是否把第三方类型泄漏到核心模块？
4. 是否拥有清晰的数据写入边界？
5. 是否能单独替换、模拟和测试？
6. 失败是否通过类型化错误返回？
7. 是否能在没有 Android UI 和真实设备时测试核心逻辑？
8. 删除该适配器是否只影响一个能力包？

若第 3、4、8 项任一答案为“否”，先拆适配器和数据边界，再进行功能吸收。

## 直接实施顺序

```text
Operit 行为基线
  -> TaskSession / AssetRecord / Episode / SkillIR 合同
  -> ClosePaw 虚拟屏适配
  -> X-OmniClaw 相册和会话适配
  -> PocketSearch 图片向量适配
  -> Operit OCR/文档解析拆包
  -> KnowAct 编译、验证和 promote
  -> ClawGUI-Skills / PowerMem 生命周期
  -> ClawGUI 远程电脑和云端模型
  -> 删除被替代的旧路径
```

## 个人数字资产模块

正式名称：全域数字资产多模态理解、语义检索与任务执行。

### 推荐实现边界

```text
MediaStore / SAF / Workspace / Remote Source
  -> AssetRecord
  -> Parser / OCR / ASR / Visual Encoder
  -> Metadata + FTS + Modality Vectors
  -> Query Router
  -> Multi-recall + RRF
  -> Reranker
  -> Asset URI + Evidence
  -> Agent Action
```

不要把全部资产强制塞进单一向量空间。效果优先时，图片、文本、音频、视频和视觉文档使用专用编码器，在查询阶段执行多路召回与融合。

### 照搬来改

| 能力 | 来源 | 直接参考文件或模块 |
|---|---|---|
| Android 相册扫描 | X-OmniClaw | `AlbumScanner.kt`、`GalleryMemoryWorkflow.kt` |
| OCR 和文档转换 | Operit | `OCRUtils.kt`、`DocumentConversionUtil.kt` |
| Android 文档向量检索 | Operit | `MemoryRepository.kt`、`VectorIndexManager.kt` |
| 端侧照片向量管线 | PocketSearch | `clip_service.dart`、`index_service.dart`、`vector_store.dart`、`search_service.dart` |
| 媒体资产模型和过滤 | Immich | asset、search、machine-learning 的数据和任务设计 |
| 端侧照片预处理 | Ente | MobileCLIP 索引、图片预处理、增量任务、查询过滤 |
| 全文件监听和双向量 | Semantic Finder | watcher、ingest、retrieval、storage |
| 文件虚拟层 | DeepStudent | VFS、Blob、SQLite、LanceDB、OCR adapters |
| 服务端双管线 | EagleRAG | Knowhere + PixelRAG、文本/视觉集合、证据回链、MCP |

### 模型与检索组件

| 场景 | 首选 | 补充 |
|---|---|---|
| Android 低延迟搜图 | MobileCLIP-S1/S2 + MNN | Chinese-CLIP 或量化 Jina CLIP 做中文对比 |
| 高效果文字搜图 | Jina CLIP v2 | Qwen3-VL-Embedding、WeMM-Embedding |
| 截图和复杂图片 | Qwen3-VL-Embedding | OCR 文本向量 + VLM 精排 |
| PDF 和视觉文档 | ColPali/PixelRAG | RAG-Anything、Knowhere、Operit OCR |
| 音频 | ASR 文本向量 + 专用音频向量 | omni-retrieval 的统一多模态路线 |
| 视频 | 镜头切分 + 抽帧 + ASR | 视频向量和时间戳级结果 |
| Android 向量库 | Zvec/ObjectBox/SQLite-vec 对比验证 | 不在实测前锁定单一实现 |
| 服务端向量库 | Qdrant/Milvus | seekdb 用于向量、全文和标量统一查询 |

### 必须保存的证据

```text
asset_id
source_uri
mime_type
path_or_album
created_at / modified_at
content_hash / perceptual_hash
parser_version / embedding_model / embedding_dimension
ocr_text / transcript / visual_summary
page / timestamp / region
permission_scope / sensitivity
```

检索结果必须返回原始 URI、命中信号、页码或时间戳和模型版本，不能只返回模型生成的名称或摘要。

## 高价值参考项目

| 项目 | 本地目录 | 价值 | 采用方式 |
|---|---|---|---|
| Operit | `AutoRefer/operit` | Android 产品、工具、文档、向量、工作流、Shower | 根工程 |
| ClosePaw | `AutoRefer/closepaw` | 虚拟屏生命周期、清理、输入和策略 | 执行内核 |
| Aries-AI | `AutoRefer/Aries-AI` | GL 帧管线、输入签名、IME、任务迁移 | 适配器参考 |
| Ruto-GLM | `AutoRefer/Ruto-GLM` | 多 display 会话和定向输入 | 并发执行参考 |
| ClawGUI | `AutoRefer/ClawGUI` | 控制面、渠道、模型、Episode、Skill、评测 | 控制面和协议参考 |
| KnowAct | `AutoRefer/KnowAct-shallow` | 轨迹编译、状态契约、验证晋升 | 编译器底座 |
| X-OmniClaw | `AutoRefer/X-OmniClaw` | 端侧会话、上下文、相册记忆、行为记录 | 端侧能力参考 |
| Zafiro | `AutoRefer/zafiro` | 工具、MCP、Python、Skill | Runtime 参考 |
| PowerMem | `AutoRefer/PowerMem` | 记忆提取、融合、衰减、Experience/Skill | 记忆生命周期参考 |
| PocketSearch | `AutoRefer/PocketSearch` | MNN + MobileCLIP + Zvec、Android 性能数据、Agent API | 端侧搜图底座候选 |
| Ente | `AutoRefer/ente-research` | MobileCLIP、增量索引、图片预处理和组合过滤 | 照片工程参考 |
| Immich | `AutoRefer/immich-research` | 媒体资产、OCR、CLIP、人脸、路径和过滤 | 媒体数据模型参考 |
| Semantic Finder | `AutoRefer/semantic-finder-research2` | 全文件监听、媒体双向量、ANN+BM25+RRF | 全文件索引参考 |
| DeepStudent | `AutoRefer/deep-student` | VFS、Blob、SQLite、LanceDB、OCR、Skill/MCP | 统一资产层参考 |
| EagleRAG | `AutoRefer/EagleRAG` | 文本/视觉双管线、融合、引用、MCP | 服务端架构参考 |
| Knowhere | `AutoRefer/Knowhere-research` | 文档层级、结构化块和可追溯证据 | 文档解析参考 |
| PixelRAG | `AutoRefer/PixelRAG-research2` | 网页/PDF 视觉渲染、切片和视觉索引 | 视觉文档参考 |
| RAG-Anything | `AutoRefer/RAG-Anything` | 文本、图片、表格、公式联合处理 | 多模态文档参考 |
| ColPali | `AutoRefer/colpali` | OCR-free 视觉页面多向量检索 | 高精度实验参考 |
| Jina CLIP 服务 | `AutoRefer/clip-as-service` | 图片和文本统一编码服务 | 服务封装参考 |
| OpenCLIP | `AutoRefer/open_clip` | CLIP 模型加载、训练和推理 | 模型基线 |
| MobileCLIP | `AutoRefer/ml-mobileclip-research` | 移动端图文检索模型 | 端侧模型基线 |
| Qwen3-VL-Embedding | `AutoRefer/Qwen3-VL-Embedding` | 图片、截图、视频、混合模态向量和精排 | 高效果模型候选 |
| WeMM-Embedding | `AutoRefer/WeMM-Embedding` | 文本、图片、视频和视觉文档统一向量 | 高效果模型候选 |
| Qdrant | `AutoRefer/qdrant` | 多向量、过滤、融合和服务端检索 | 服务端向量库候选 |
| sqlite-vec | `AutoRefer/sqlite-vec` | SQLite 内嵌向量搜索 | 端侧数据库候选 |
| seekdb | `AutoRefer/seekdb` | 向量、全文、标量和 Agent 数据统一 | 服务端数据库候选 |
| omni-retrieval | `AutoRefer/omni-retrieval` | 文本、图片、音频、视频统一离线检索 | 全模态实验参考 |

## 第一轮删减候选

仅在依赖扫描和回归测试建立后执行：

1. 将 Operit 的 `dragonbones`、`fbx`、`mmd` 等非核心展示模块从默认构建中隔离。
2. 将市场、示例、开发工具和业务 UI 从核心 Agent Runtime 中拆出。
3. 将大规模工具注册表拆成 capability packages，按需加载。
4. 将 Root、Shizuku、无障碍、文件和虚拟屏权限按能力声明，不再默认捆绑。
5. 保留 `terminal`、`mnn`、`llama`、`quickjs`、`showerclient`，直至替代实现通过回归验证。

## 实施顺序

1. 固定 Operit 主分支和现有行为基线，建立独立整合分支。
2. 定义 `TaskSession`、`DeviceBackend`、`AssetRecord`、`Episode`、`SkillIR`、`VerificationResult`。
3. 用 ClosePaw 生命周期替换虚拟显示核心，保留 Operit Shower 作为特权服务。
4. 接入 KnowAct 轨迹编译和验证晋升。
5. 接入 ClawGUI-Skills 和 PowerMem 的技能/记忆生命周期。
6. 建立个人数字资产增量索引，先接 Operit OCR 和 PocketSearch 图片向量。
7. 接入远程电脑 Agent、ClawGUI 渠道和云端模型路由。
8. 建立 ROM、GUI、检索和 Skill 回放评测矩阵，再开始删除旧实现。

## 验证门槛

进入下一阶段前必须满足：

- Android 版本和 ROM 能力探测可复现；
- 虚拟屏创建、输入、截图、任务迁移和清理均有明确结果；
- GUI 动作有执行后验证，失败能分类；
- Skill 未通过设备、结果和重复回放验证不得晋升；
- 资产检索返回可追溯 URI、页码或时间戳；
- Embedding 模型和维度变更能触发受控重建；
- 远程设备支持取消、重连和权限撤销；
- 高风险动作保留人工确认和审计记录。

## 待验证决策

1. PocketSearch 的 Zvec 与 Operit 现有 HNSW、ObjectBox、SQLite-vec 的 Android 性能对比。
2. MobileCLIP、Jina CLIP v2、Qwen3-VL-Embedding、WeMM 的中文召回率和端侧成本。
3. KnowAct Python 编译器采用本地服务、Chaquopy 还是 Kotlin 移植。
4. ClosePaw 接入 Operit Shower 后的生命周期和 Binder 边界。
5. ClawGUI Gateway 与 Operit HTTP bridge 的远程设备协议合并方式。
