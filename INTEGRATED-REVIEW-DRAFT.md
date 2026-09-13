# zhiFaGuiRefer 整合审阅总稿

状态：待用户审阅和删改。

本文把仓库当前的 AI 浏览器方案、全域助手扩展方案、参考项目源码结论、权限矩阵、记忆选型、数据库选型、技能契约和实验缺口合并为一份文档。原专题文档暂时保留，本文只作为集中审阅稿；用户确认后再反向精简专题文档。

## 1. 项目定位

### 1.1 当前第一阶段目标

第一阶段实现 Android AI 浏览器，而不是先实现通用手机 AI 助手。核心能力是：

- 持久化浏览器会话；
- 前台共享 WebView；
- 后台隐藏 WebView、Virtual Display 或外部浏览器执行表面；
- 用户观察、暂停、接管和恢复；
- DOM/可访问性优先，视觉 GUI 作为降级；
- 动作后置条件和页面结果验证；
- 成功轨迹清洗、参数化、回放和 Skill 晋升。

### 1.2 未来全域助手方向

浏览器稳定后，扩展到手机 GUI、个人文件和图片检索、长期记忆、远程电脑、端侧/云端模型、工作流和跨设备协同。全域扩展属于后续能力，不改变浏览器第一阶段的核心边界。

### 1.3 当前状态 `S0`

- 已有多个参考仓库和源码快照；
- 当前仓库只有研究文档，没有产品代码；
- 尚未完成真机 ROM、性能、功耗和中文召回实验；
- 参考项目许可证、语言、运行时和权限边界不同，不能整体拼接。

### 1.4 目标状态 `Sg`

```text
用户任务
  -> 创建或恢复 BrowserSession
  -> 选择前台共享或后台执行表面
  -> DOM/结构化动作
  -> 页面变化与结果验证
  -> 失败时重观察、回退或切换视觉/设备后端
  -> 记录 Episode 和检查点
  -> 去冗余、参数化、生成 SkillIR
  -> 相同会话和新会话回放验证
  -> candidate -> validated -> promoted
```

### 1.5 理想方向 `IFR`

用户只描述目标，不需要理解 WebView、Virtual Display、CDP、模型或权限细节。系统自动选择成本最低且可验证的执行表面；重复任务优先执行已经验证的 Skill；用户始终可以看到、暂停、接管或恢复任务。

### 1.6 第一阶段非目标

- 通用手机 App 自动化平台；
- 远程电脑和多设备编排；
- Android Chrome 插件；
- 完整长期记忆、知识图谱和训练闭环；
- 对所有厂商 ROM 的无差别兼容；
- 复杂终端、社交、短信、日历、语音和系统管理工具。

## 2. 根工程决策

### 2.1 当前推荐方案

采用“**新建 Android 产品壳 + Agentic WebView 浏览器核心**”。

```text
产品根工程：product/ai-browser Android App
浏览器核心：browser/agentic-webview
后台增强：ClosePaw / Aries-AI / Ruto-GLM 适配器
Agent 与技能：ClawGUI-APP / KnowAct / ClawGUI-Skills / Ghost 适配器
Android 工具：Operit 局部模块适配器
```

产品第一性对象是 `BrowserSession`，不是 `PhoneAgent`，也不是某个虚拟屏实现。

### 2.2 根工程候选取舍

| 候选 | 优势 | 主要缺陷 | 当前结论 |
|---|---|---|---|
| 新建 Android 产品壳 | 可以围绕 BrowserSession、执行权和用户接管建立干净边界 | 需要吸收 Agent、轨迹、Skill 和 Shizuku 基础设施 | 产品根工程 |
| Agentic WebView | WebView、会话协议、元素引用、结构化命令和失败模型边界清楚 | 不是完整 Agent Runtime | 浏览器核心首选 |
| ClawGUI | 任务、会话、Provider、Episode、远程渠道和 GUI Agent 完整 | Android 浏览器、权限和本地服务不足 | Agent/控制面参考 |
| Operit | WebSession、工具、文件、OCR、工作流、向量和 Android 服务最多 | 单体大、权限和工具耦合重 | Android 能力参考，不作浏览器根 |
| ClosePaw | Virtual Display 生命周期、Shizuku、输入、截图和清理成熟 | 缺少浏览器会话和完整产品壳 | 后台执行内核参考 |
| ZorvBrowser | GeckoView、标签页、Cookie、Storage、下载和审计完整 | 需要重新接入 Agent Runtime | 浏览器替代内核参考 |
| AIOPE、Eta、OpenMinis、RikkaHub | 产品形态、系统 Agent、工具或沙箱有局部价值 | 目标、许可证或运行时不匹配 | 只吸收局部思路 |

### 2.3 与旧全域助手方案的关系

旧文档曾提出 `ClawGUI + Operit` 双核心总工程：ClawGUI 负责控制面，Operit 负责 Android 运行面。该方案仍适合作为未来全域扩展，但第一阶段浏览器不直接继承两个项目的产品壳，而是先建立 BrowserSession 和 Agentic WebView 的独立边界。

## 3. 浏览器执行架构

### 3.1 BrowserSession

`BrowserSession` 是唯一状态源，至少保存：

- 会话 ID、任务 ID、当前执行权；
- 标签页、页面 URL、Cookie、Storage 和导航历史；
- 页面 DOM/可访问性快照和页面版本；
- 动作历史、检查点、验证结果和失败原因；
- 当前执行表面和可用能力；
- Skill 引用、模型快照和审计事件。

前台和后台只是不同执行表面，不是两个页面副本。切换时只转移执行权和会话状态，不复制 Cookie、DOM 或页面状态。

### 3.2 执行表面

| 表面 | 主要用途 | 进入条件 | 退出条件 |
|---|---|---|---|
| `ForegroundWebView` | 用户可见、可观察和可接管 | 用户打开会话或请求接管 | 用户切后台、锁屏或主动让出 |
| `HiddenWebView` | 后台 DOM/JS 执行 | 页面无需系统级输入 | 需要真实输入、渲染或后台限制触发 |
| `VirtualDisplay` | 隔离渲染、截图和定向输入 | DOM 不能完成或需要隔离 | 任务结束、权限失效或资源回收 |
| `ExternalBrowser` | Chrome/CDP 或独立浏览器 | 用户明确授权或 WebView 不足 | 连接断开、权限撤回或任务完成 |

### 3.3 执行权状态机

```text
Idle
  -> ForegroundOwned
  -> BackgroundOwned
  -> HandoffPending
  -> ForegroundOwned / BackgroundOwned
  -> Paused / Failed / Completed
```

约束：

1. 同一 `BrowserSession` 同时只有一个动作执行者；
2. 用户接管优先级最高，后台动作必须在动作边界停止；
3. 转移前写入检查点，转移后重新获取页面快照；
4. 表面失效进入 `Paused` 或 `Failed`，不能静默切换继续操作。

### 3.4 观察优先级

```text
DOM / 可访问性树 / 结构化浏览器接口
  > 页面脚本、网络状态和结构化结果
  > WebView 截图或 CDP
  > Virtual Display 截图和视觉模型
  > 坐标点击和滑动
```

坐标是最后降级路径，不能作为默认动作协议。

### 3.5 统一 Action IR

```text
ActionIR
  actionType
  targetLocator
  parameters
  preconditions
  postconditions
  timeout
  retryPolicy
  evidenceRefs
  source
```

模型输出必须先转换为 `ActionIR`，执行器不能直接执行自然语言或未经校验的坐标。

## 4. 验证与技能闭环

### 4.1 动作验证

```text
观察 -> 选择动作 -> 执行
                 -> 页面变化检查
                 -> 后置条件检查
                 -> 证据记录
       失败 -> 重观察 / 回退 / 请求接管
```

“调用成功”不等于“任务成功”。至少需要页面变化、目标元素状态、结构化结果或业务结果证据。

### 4.2 轨迹到 Skill

```text
Episode
  -> 过滤读屏和无效工具
  -> 删除等待、重复点击和无效观察
  -> 提取参数、前置条件和成功判据
  -> 生成 SkillIR
  -> 相同会话回放
  -> 新会话回放
  -> candidate -> validated -> promoted
```

未经设备验证、结果验证和重复回放的轨迹只能进入候选区。

### 4.3 Skill 必备字段

- Skill ID、版本、适用站点/页面特征；
- 输入参数和默认值；
- 前置条件、定位器和动作步骤；
- 中间检查点、成功判据和失效判据；
- 失败回退、人工接管条件和权限要求；
- 来源 Episode、验证设备、模型版本和审计记录。

## 5. 参考项目迁移分类

### 5.1 可以直接照搬，再做适配

| 模块 | 来源 | 适配要求 |
|---|---|---|
| 浏览器协议和 WebView 边界 | Agentic WebView `browser-api`、`browser-webview`、`agent-tools` | 接入 BrowserSession、执行权和页面版本 |
| Skill 执行合同 | Ghost `Action`、`Workflow`、`Element` | 保留前后置条件、回滚、重试和定位降级链 |
| 轨迹蒸馏边界 | Ghost `trace_to_steps.py` | 保留 actuating-tool allow-list，后续接参数化和验证 |
| Skill 元数据和校验 | Ghost `skill.yaml`、`validate_skill.py` | 增加权限、来源、验证证据和兼容矩阵 |
| 虚拟屏生命周期 | ClosePaw `VirtualDisplayPlatform`、`VdLifecycleArbiter` | 接入自有 `DeviceBackend`，补 ROM/API 探测 |
| 控制面数据合同 | ClawGUI `TaskSession`、`Episode`、`VerificationResult`、`SkillPipeline` | 作为中立领域合同，不把 Android 类型带入控制面 |
| Skill 验证和修订 | ClawGUI-Skills `verifier.py`、`evolution.py` | 保留信息隔离、版本快照、失败案例和受限修订 |

### 5.2 换语言或运行时重写

| 模块 | 原实现 | 重写方向 |
|---|---|---|
| Agentic WebView 与产品壳连接 | Android/Python 混合参考 | Kotlin Android 产品层 + 中立 BrowserSession 合同 |
| 轨迹编译和去冗余 | KnowAct Python、DroidAgent Python | Kotlin 端轻编译器，复杂模型修订保留服务端 |
| 动作树和缓存 | MobiAgent Python/Torch | Kotlin 状态—动作—任务索引，缓存只产生候选 |
| 反思和 Shortcut | Mobile-Agent-E Python | Kotlin 会话对象 + 持久化候选，不能只存提示词 |
| 外部浏览器控制 | Zorv ACI/AIDL、CDP | Kotlin Binder/HTTP Provider，保留浏览器动作合同 |
| 端侧模型服务 | clip-as-service Python | Kotlin `ModelProvider`，云端保留批量/流式服务 |

### 5.3 只能学习思路

| 来源 | 学习内容 | 不直接搬入 |
|---|---|---|
| deep-student | 强类型上下文、取消传播、VFS-RAG、审批 | Rust/Tauri 学习工具链和业务范围过重 |
| EagleRAG | 文本/视觉双管线、引用回链、多租户治理 | Celery、Milvus、Redis、MinIO 服务拓扑 |
| RAG-Anything | 多模态文档解析和关系组织 | 服务端运行环境和重型基础设施 |
| ColPali | 视觉文档多向量和 late interaction | 手机默认运行的模型与存储成本 |
| ClawGUI-RL/Eval | 训练、评测和 Infer-Judge-Metric 闭环 | Android 生产包中的训练依赖 |
| AppAgentX | Page-Element-Action 图和结构化模板化 | Neo4j/Pinecone 生产依赖和字符串完成判定 |
| Eta/OpenMinis | 系统级权限路由、沙箱和渐进式 Skill | Root/LSPosed、GPL 或非商业运行边界 |

### 5.4 只吸收一两点特性

| 来源 | 吸收点 |
|---|---|
| Aries-AI | 输入签名探测、OpenGL 帧分发、IME 焦点、启动后迁移 |
| Ruto-GLM | `session -> displayId -> Job`、SurfaceControl 镜像、API 33 分支 |
| X-OmniClaw | 精确停止、上下文压缩、相册游标、摘要隐私、deeplink |
| MobiAgent | ActTree、相似任务召回、目标元素变化检测 |
| Mobile-Agent-E | Manager/Operator/Reflector 分工、Tips/Shortcut 反思 |
| Zafiro | ToolRegistry、MCP/Python、Skill 按需加载、Shell 规则 |
| Qwen3/WeMM | 多模态统一向量、Reranker、MRL 维度裁剪 |
| Qdrant/sqlite-vec | payload 过滤、命名向量、SQLite 内嵌向量字段 |

### 5.5 删减后使用

| 来源 | 删除/隔离 | 保留 |
|---|---|---|
| Operit | 市场、示例、非目标工具、重复 UI、分散 ROM 特判 | WebSession、文件、OCR、文档、工作流和权限适配器 |
| ClawGUI | RL、Eval、非目标渠道和生产无关 UI | Agent、Session、Provider、Episode 和远程协议 |
| ClosePaw | 产品聊天 UI 和业务入口 | Virtual Display、Shizuku、生命周期、输入和截图 |
| Aries-AI | 主 UI、重复输入实现、API 30-33 不可用入口 | 帧分发、IME、任务迁移和探测思路 |
| Ruto-GLM | Compose 聊天层、固定指令格式、未完成 IME | Display/Job/Input 低层能力 |
| Zafiro | Xposed 宿主、厂商语音接管和产品 UI | ToolRegistry、MCP、Python、Skill Runtime |
| DroidAgent | DROIDBOT 实验基础设施、固定尺寸和坐标假设 | 离线探索和轨迹种子 |

## 6. Android 个人文件、图片和其它 App 文件

### 6.1 目标能力

```text
授权
  -> 文件/相册扫描
  -> MIME 和元数据识别
  -> 文本解析/OCR/ASR/视觉摘要
  -> 文本、图片、音频、视频向量
  -> 全文、标量、向量和跨模态召回
  -> 原始 URI、页码、时间戳或区域证据
  -> 查看、复制、编辑、分享、发送或继续交给 Agent
```

### 6.2 权限和访问边界

| 内容 | 能否读取 | 方式 |
|---|---|---|
| 其它 App 写入共享相册/视频/音频 | 可以 | MediaStore + 对应 `READ_MEDIA_*` |
| 公共目录普通文件 | 可以 | SAF 选择文件或目录 |
| 对方主动分享的文件 | 可以 | `content:// URI`、FileProvider、ACTION_SEND |
| 其它 App 导出的 Provider/云接口 | 可以 | 按接口授权 |
| 其它 App 内部目录 | 普通应用不可以 | Android 沙箱隔离 |
| 其它 App `Android/data`、`Android/obb` | Android 11+ 通常不可以 | Scoped Storage 和 SAF 限制 |
| 其它 App 私有数据库、缓存、密钥和令牌 | 不可以 | 没有公开授权接口 |

`MANAGE_EXTERNAL_STORAGE` 只能扩大共享存储访问，不能解锁其它 App 私有沙箱和 `Android/data`。

### 6.3 资产权限状态

```text
sourceKind: media_store | saf_uri | provider | cloud | app_private
accessScope: app_owned | uri | partial | full | inaccessible
ownerPackage
grantPersisted
lastAccessCheckAt
revokedAt
```

规则：

1. Android 14 部分照片授权下，当前看不到不能当成已删除；
2. URI 撤销、文件移动或删除后，派生向量可以保留，但禁止打开、分享和上传；
3. 权限必须每次使用前重新检查，不能只读 DataStore 中的历史布尔值；
4. `MANAGE_EXTERNAL_STORAGE`、无障碍、Shizuku、Root、悬浮窗和云端上传必须是独立能力开关；
5. 检索结果必须返回来源 App、授权范围和当前可访问状态。

### 6.4 端侧索引方案

| 能力 | 主要来源 | 当前处理 |
|---|---|---|
| 相册增量扫描 | local-photo-search、X-OmniClaw | 直接吸收 MediaStore 游标和删除同步，但区分 partial 与删除 |
| 图片向量 | local-photo-search | 作为端侧索引骨架，保留模型版本、失败重试和后端探测 |
| 图文编码与过滤 | PocketSearch | Kotlin 重写 MNN/ONNX、schema 版本和标量过滤 |
| OCR/Office/PDF | Operit | 拆为 AssetParser Adapter |
| 图片摘要/隐私 | X-OmniClaw | 只保存派生摘要和证据引用，不替代原图向量 |
| 文件路由和混合检索 | semantic-finder/EagleRAG | 吸收 parser、chunk、RRF、VisualEncoder 路由 |
| 高质量多模态模型 | Qwen3-VL-Embedding/WeMM | 云端或 GPU 按需启用 |

## 7. 记忆与数据库

### 7.1 五层事实源

```text
L0 工作记忆       当前会话、最近观察、待确认动作
L1 个人长期记忆   偏好、稳定事实、联系人、跨会话经验
L2 资产记忆       文件/图片/视频/音频/文档解析、向量和证据
L3 知识记忆       主题页、实体页、引用图和冲突记录
L4 技能记忆       已验证 SkillIR、参数、前置条件和失败修订
```

`Episode`、ActTree 和最近成功路径是过程记录或候选缓存，不是事实记忆。只有通过验证、去重、冲突检查和权限检查后才能写入长期记忆或 Skill。

### 7.2 MemoryRouter

1. 日期、数量、文件属性、权限、版本和状态先走结构化查询；
2. 全文、关键词、文本向量、图片向量和关系召回并行执行；
3. 用 RRF 或可解释加权融合，质量不足时进入深度路径；
4. 深度路径回读原始资产或完整 Episode，并返回 `EvidenceRef`；
5. 所有写回都先进入候选区，经抽取、去重、冲突、权限和版本检查后提交。

### 7.3 Android 数据库最小方案

```text
Android: Room -> SQLite/FTS5 -> ObjectBox HNSW
Cloud:   PostgreSQL + pgvector + relation tables
```

- Room/SQLite 保存资产、解析结果、证据、关系、任务、权限和同步事件；
- FTS5 保存文件名、OCR、转写、摘要和正文；
- ObjectBox 只保存端侧向量和 HNSW，可删除重建；
- 不把 Asset、Memory、Entity、Episode 和 Skill 合并为一张事实表；
- 端侧不默认部署 Neo4j、Graphiti、Qdrant 或 Milvus；
- 向量必须绑定 `asset_part_id`、模态、模型版本、维度和量化信息。

## 8. 高内聚、低耦合约束

### 8.1 依赖方向

```text
ControlPlane -> Contracts -> CapabilityRouter
                              -> BrowserAdapter
                              -> DeviceAdapter
                              -> AssetIndexAdapter
                              -> SkillAdapter
                              -> ModelProvider
```

### 8.2 规则

- 核心合同不依赖 Android、Shizuku、数据库、模型 SDK 或第三方项目；
- 第三方实现只能进入 Adapter/Provider；
- 每类数据只有一个事实来源；
- UI 不直接操作设备、数据库或文件系统；
- 模型不直接调用设备 API，不直接修改 Session、Asset 或 Skill 状态；
- Skill 只调用 `Action`/`Capability`，不直接调用隐藏 API；
- ROM 特判集中在设备能力探测和适配器；
- 任何跨边界调用都携带 `request_id`、`trace_id`、`task_id`、`session_id`、`contract_version`、`deadline`、`cancel_token`、`status`、`error_code` 和 `evidence_refs`。

### 8.3 数据所有权

| 数据 | 唯一所有者 |
|---|---|
| BrowserSession、任务和执行权 | 产品控制器/SessionRepository |
| Android 权限和设备能力 | AndroidRuntime/DeviceAdapter |
| 原始文件和 URI | 系统存储或用户授权 Provider |
| 解析结果和资产索引 | AssetRepository/AssetIndex |
| 向量 | VectorIndex |
| Episode 和验证证据 | EpisodeStore/VerificationStore |
| Skill 版本和晋升状态 | SkillStore/SkillLifecycle |
| 模型配置和用量 | ModelProvider/UsageRepository |

## 9. 兼容、权限和验证门槛

### 9.1 必须验证的 Android 表面

- API 28、32、33、34、35；
- 前台共享 WebView、后台隐藏 WebView、Virtual Display、外部 CDP；
- DOM、可访问性、截图、视觉和坐标降级链；
- ClosePaw、Aries-AI、Ruto-GLM 的显示创建、输入、截图、任务迁移和清理；
- ColorOS、MIUI、鸿蒙、三星等 ROM 的权限、后台存活和系统服务反射。

### 9.2 必须验证的 Skill 闭环

- 轨迹是否能正确过滤只读工具和无效步骤；
- 等待、重复点击和探索动作是否被安全去冗余；
- 参数提取是否泄露账号、密码、验证码或本地路径；
- 相同会话和新会话回放是否都通过；
- 失败是否能分类为定位、规划、权限、执行、表面或结果错误；
- 只有设备验证、结果验证和重复回放通过后才能晋升。

### 9.3 必须验证的资产检索

- full/partial/URI/拒绝/撤销权限状态；
- 用户删除、文件移动、URI 失效和暂时不可见的区分；
- OCR、Embedding、缩略图和向量库失败时的重试与恢复；
- 中文图文召回、跨模态召回、证据定位、延迟和电量；
- 模型升级和向量维度变化后的受控重建。

## 10. 实施顺序

### P0：先冻结核心边界

1. `BrowserSession`、`ExecutionSurface`、`ExecutionLease`、`ActionIR`；
2. 前台 WebView 的 DOM/可访问性观察和结构化动作；
3. 页面变化、后置条件、失败回退和用户接管；
4. Episode、检查点、轨迹 allow-list 和 Skill 候选状态。

### P1：建立可运行闭环

1. 后台隐藏 WebView；
2. ClosePaw/自有 Virtual Display Adapter；
3. Ghost Action/Workflow 合同和 KnowAct 验证状态；
4. SkillPackage、版本、失败案例和重复回放；
5. 最小 Android 权限和本地持久化。

### P2：补 Android 和个人数据能力

1. Operit 文件、OCR、文档和工作流 Adapter；
2. MediaStore/SAF 资产索引；
3. local-photo-search 图片向量；
4. X-OmniClaw 会话/相册/摘要；
5. PocketSearch 图文编码和过滤。

### P3：全域扩展

1. ClawGUI 控制面和远程电脑；
2. PowerMem 长期记忆和 Handoff；
3. Qwen3/WeMM、EagleRAG、Reranker 和服务端索引；
4. 多设备、云端算力 API、远程渠道和跨设备资产查询；
5. 训练评测和规模化部署。

## 11. 当前待审阅问题

请在本文上直接删改或批注以下决策：

- [ ] 第一阶段是否只保留 AI 浏览器，还是同时保留手机个人文件检索 MVP？
- [ ] 产品根工程是否确认“新建 Android 壳 + Agentic WebView”，不再把 Operit/ClawGUI 作为产品壳？
- [ ] 后台执行首选自有实现还是先隔离复用 ClosePaw？
- [ ] 是否允许 Aries-AI 仅作独立进程/行为参考，不复制其 AGPL 文件？
- [ ] Skill 的正式格式是否以 SkillIR + SkillPackage 为准？
- [ ] Android 资产索引是否默认只用 Photo Picker/SAF，还是提供全库媒体索引开关？
- [ ] 是否接受端侧 Room + FTS5 + ObjectBox HNSW 的数据库方案？
- [ ] 哪些全域助手能力必须进入浏览器第一阶段，哪些明确延期？

## 12. 原文档映射

| 本文部分 | 原文档 |
|---|---|
| 产品目标、浏览器表面和执行权 | `architecture/ai-browser-overview.md` |
| 根工程和参考项目迁移矩阵 | `architecture/reference-absorption-plan.md` |
| 全域助手、资产系统和源码长版推演 | `decision-records/001-main-project-and-reference-modules.md` |
| 源码复核、五类分类和双核心方案 | `decision-records/002-source-review-and-final-integration-list.md` |
| Android 文件和其它 App 权限 | `decision-records/003-android-asset-permission-matrix.md` |
| 记忆分层和 MemoryRouter | `decision-records/004-frontier-memory-system-selection.md` |
| Android 数据库和索引选型 | `decision-records/005-android-database-architecture-selection.md` |
| 通用动作和技能契约 | `architecture/overview.md`、`specs/skill-contract.md` |
| 项目、源码文件和证据等级 | `evidence/source-index.yaml` |
| 实验计划 | `experiments/README.md` |

## 13. 源码参考总表

| 项目 | 角色 | 证据 | 当前迁移定位 | 本地核查范围 |
|---|---|---|---|---|
| Aries-AI | 虚拟屏与输入执行 | L1 源码 | 学习/局部吸收；不直接复制完整引擎 | `ShizukuVirtualDisplayEngine.kt`、`VirtualAsyncInputInjector.kt`、`VirtualDisplayController.kt` |
| ClosePaw | 虚拟屏生命周期 | L1 源码 | 执行层直接底座候选 | `VirtualDisplayPlatform.kt`、`VdLifecycleArbiter.kt`、`ShizukuDisplayTransport.kt`、`VirtualDisplayInputInjector.kt` |
| Operit | Android 工具和 WebSession | L1 源码 | 局部 Adapter，删减后使用 | `VirtualDisplayManager.kt`、`MemoryRepository.kt`、`VectorIndexManager.kt`、`ToolPermissionSystem.kt` |
| ClawGUI | 控制面和 Agent | L1 源码 | 运行时与协议参考 | `contracts.py`、`tracer.py`、`device_factory.py`、Skill verifier/evolution |
| ClawGUI-Skills | Skill 生命周期 | L1 源码 | 验证、失败修订、版本和审计底座 | `schema.py`、`verifier.py`、`evolution.py` |
| KnowAct | 轨迹编译和晋升 | L1 源码 | `SkillIR`、状态合同和 `validate/promote` 底座 | `trajectory_codegen.py`、`state_contract.py`、`shortcut_validation.py` |
| Ghost in the Droid | Skill 执行和轨迹蒸馏 | L1 源码 | Action/Workflow/allow-list 直接吸收 | `trace_to_steps.py`、`base.py`、`checkpoint.py`、`auto_creator.py`、Skill 校验器 |
| MobiAgent | 动作树和缓存 | L1 源码 | ActTree、候选缓存和目标变化检测 | `action_cache/tree.py`、`action.py`、`reranker.py` |
| Mobile-Agent-E | 计划、执行、反思 | L1 源码 | Manager/Operator/Reflector 思路重写 | `agents.py`、`inference_agent_E.py`、`controller.py` |
| DroidAgent | App 探索和脚本 | L1 源码 | 离线探索和 Skill 种子 | `app_state.py`、WorkingMemory/TaskMemory、`make_script.py` |
| AppAgentX | 页面—元素—动作图 | L1 源码 | 只吸收结构化模板化和关系模型 | `explor_auto.py`、`State.py`、`graph_db.py`、`data_storage.py`、`chain_evolve.py` |
| Ruto-GLM | 多虚拟屏并发 | L1 源码 | display/job 并发和输入构造局部吸收 | `RutoAiTasker.kt`、`InputManagerService.kt`、`DisplayManagerServiceStub.kt` |
| Zafiro | Tool/MCP/Python Runtime | L1 源码 | ToolRegistry、Skill 和 Shell 策略局部吸收 | `ToolRegistry.kt`、`ToolManager.kt`、`SkillFileRepository.kt`、`ShellCommandSafetyPolicy.kt` |
| X-OmniClaw | Android 会话和个人资产 | L1 源码 | 相册、摘要、会话、压缩和安装局部吸收 | `MemoryIndex.kt`、`SessionManager.kt`、`MessageCompactor.kt`、`SkillInstaller.kt` |
| PowerMem | Memory/Handoff 生命周期 | L1 源码 | 记忆合同、版本和交接参考 | `protocols.py`、Memory/Handoff service |
| EagleRAG | 多模态路由和视觉编码 | L1 源码 | VisualEncoder、内容路由和证据字段 | `router.py`、`visual_encoder.py`、视觉索引 |
| semantic-finder | 文件解析和混合检索 | L1 源码 | parser/chunk/RRF/MRL 局部吸收 | `parsers.py`、`chunker.py`、`hybrid_search.py`、`reranker.py`、watcher |
| local-photo-search | Android 图片索引 | L1 源码 | 端侧图片索引骨架 | `MediaStoreScanner.kt`、`PhotoIndexStore.kt`、`ImageEmbeddingIndexer.kt`、`VectorStore.kt` |
| PocketSearch | MNN 图文搜图 | L1 源码 | Kotlin 重写编码、过滤和 schema 迁移 | `clip_service.dart`、`vector_store.dart`、`index_service.dart`、`search_service.dart` |
| TIDY | ONNX/Room 图片检索基线 | L1 源码 | 只保留编码和数据结构思路 | `ORTImageViewModel.kt`、`ImageEmbeddingRepository.kt`、`ImageEmbeddingDao.kt` |
| clip-as-service | 远程 Embedding 服务 | L1 源码 | ModelProvider 批量/流式接口参考 | ONNX/Torch executor、`clip_model.py`、client API |

## 14. 待补规范

当前已有 [技能契约](specs/skill-contract.md)，但以下跨模块合同仍需在开始产品整合前冻结：

| 合同 | 必须解决的问题 |
|---|---|
| `BrowserSession` | 页面、Cookie、标签、检查点、执行权和表面状态的唯一事实源 |
| `ExecutionSurface` | 前台 WebView、后台 WebView、Virtual Display、外部浏览器的能力和切换条件 |
| `ExecutionLease` | 用户接管、暂停、恢复、取消和并发互斥 |
| `ActionIR` | 动作类型、定位器、参数、前后置条件、证据、超时和回退 |
| `Episode` | 观察、动作、模型输出、工具结果、页面差异、权限和验证证据 |
| `SkillIR/SkillPackage` | 候选、验证、晋升、失败修订、版本和安装格式 |
| `Asset/Evidence` | 原始 URI、授权范围、派生内容、页码/时间戳/区域和失效状态 |
| `TaskCommand/TaskEvent` | 产品壳、浏览器核心、Android 后端和未来控制面的幂等通信 |

## 15. 审阅方式

建议只在本文做第一轮删改，使用以下标记：

- `[保留]`：进入下一版规范；
- `[删除]`：不再维护；
- `[延期]`：保留结论但不进入第一阶段；
- `[待验证]`：需要源码补证或真机实验；
- `[冲突]`：与其他章节的目标、根工程或权限结论不一致。

确认本文后，再把结果同步回专题文档，避免继续在多份长文档中并行修改。
