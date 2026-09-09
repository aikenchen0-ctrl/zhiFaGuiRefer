# 源码复核后的根工程与模块迁移清单

状态：整合前准备文档；只记录源码分析和迁移边界，不启动整合开发。

本文基于 `AutoRefer` 中已浅克隆或本地保存的源码，重点核对了类、接口、数据结构、任务生命周期和存储边界。`001-main-project-and-reference-modules.md` 保留为早期 Android 优先方案；本文是当前“全域 AI 助手”目标下的更新决策。

## 一、根工程选择

### 方案 A：ClawGUI 作为总根工程，推荐

`ClawGUI` 作为控制面和服务端根工程，`clawgui-agent` 承担任务循环、会话、模型 Provider、Episode、远程渠道和设备后端协议。

优势：

- 与全域目标最匹配，已有任务、会话、记忆、Episode、模型 API、远程渠道和多设备抽象；
- Python 控制面适合快速演进轨迹编译、Skill 修订、模型路由和评测；
- 远程电脑、浏览器、Android 和云端模型可以统一为 `DeviceBackend`、`ModelProvider` 和 `CapabilityRouter`；
- 训练、评测和生产运行时可以保持边界，不把实验依赖带入手机。

代价：

- 必须新增或接入 Android 原生文件权限、OCR、虚拟屏、Shower、端侧模型和后台服务；
- Python/Kotlin 之间要维护 IPC、取消、错误和状态合同；
- 手机离线运行需要额外的轻量端侧 Runtime。

### 方案 B：Operit 作为 Android 产品根工程

`Operit` 作为 Android 宿主，保留其会话、工具、模型、文件、OCR、文档转换、向量、工作流、Shower 和后台服务。

优势：

- Android 权限、前台服务、端侧模型和本地文件能力已有实际代码；
- 与 `ClosePaw`、`X-OmniClaw`、`PocketSearch` 的 Kotlin/Android 模块整合成本较低；
- 可以优先形成单机可用产品。

代价：

- 工具注册、权限、UI、模型和业务状态耦合较重，必须先拆分；
- 跨设备、远程电脑、复杂会话和 Skill 编译需要重新建立控制面；
- ROM 特判、虚拟屏和本地向量索引会成为 Android 单体的长期维护负担。

### 当前结论

选择 **方案 A：ClawGUI 为总根工程**。`Operit` 降级为 Android 能力底座和适配器来源，`ClosePaw` 为虚拟屏执行内核，`KnowAct` 为轨迹编译和验证闸门，`ClawGUI-Skills` 与 `PowerMem` 提供 Skill/Memory 生命周期，`X-OmniClaw`、`local-photo-search` 和 `Operit` 提供个人数据能力。

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
