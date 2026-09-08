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
