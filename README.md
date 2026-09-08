# zhiFaGuiRefer

Android 非接口化 GUI Agent 技术参考仓库。

本仓库面向技术选型、架构拆解、实验复现和后续实现迁移，不直接复制第三方项目源码，也不承诺当前阶段提供完整可运行的 Android 产品。

## 两条核心主线

1. **虚拟屏与 GUI Agent 运行框架**：以 Aries-AI 为执行层主参考，研究 Shizuku、Virtual Display、定向输入、焦点隔离、后台截图和无障碍操作。
2. **轨迹采集与技能封装**：研究如何把高频 GUI 操作转换为可参数化、可验证、可失效检测和可回退的 Android 操作技能。

## 总体链路

```text
任务 -> 技能检索 -> 前置条件检查 -> 技能回放 -> 结果验证
                                      | 成功
                                      v
                                    返回结果
                                      |
                                      | 失败
                                      v
                           GUI Agent 重新观察与推理
                                      v
                              Virtual Display 执行
                                      v
                                轨迹记录
                                      v
                               候选技能生成
                                      v
                             真实回放验证后入库
```

## 参考项目分工

| 层级 | 项目 | 参考内容 |
|---|---|---|
| 执行层 | [Aries-AI](https://github.com/ZG0704666/Aries-AI) | Virtual Display、Shizuku、输入定向、焦点隔离、后台截图 |
| Agent Runtime | [zafiro](https://github.com/niki914/zafiro) | 原生 Agent、工具、Skill、MCP、权限管理 |
| 动作记忆 | [MobiAgent](https://github.com/IPADS-SAI/MobiAgent) | AgentRR、ActTree、历史动作检索、回放和状态校验 |
| 技能生成 | [Mobile-Agent-E](https://github.com/X-PLUG/MobileAgent/tree/main/Mobile-Agent-E) | Tips、Shortcuts、成功轨迹反思和自演化 |
| 离线探索 | [DroidAgent](https://github.com/coinse/droidagent) | App 探索、轨迹记录、UIAutomator2 脚本生成 |
| 技能封装 | [Ghost in the Droid](https://github.com/ghost-in-the-droid/android-agent) | 元素、动作、工作流、注册和版本管理 |
| 概念参考 | [AppAgentX](https://github.com/Westlake-AGI-Lab/AppAgentX) | 重复轨迹演化为高级动作 |

## 源码复核后的当前根工程结论

面向“全域 AI 助手”，当前推荐 **ClawGUI 作为总根工程和控制面**；`Operit` 作为 Android 文件、OCR、文档、Shower 和端侧工具能力底座，`ClosePaw` 作为虚拟屏执行内核。`KnowAct`、`ClawGUI-Skills`、`PowerMem`、`X-OmniClaw`、`local-photo-search` 等只按适配器和 Provider 接入。

旧的 Android 优先方案仍保留在 [`decision-records/001-main-project-and-reference-modules.md`](decision-records/001-main-project-and-reference-modules.md) 作为历史记录；基于实际源码复核后的五类迁移策略、模块需求、直接照搬清单、吸收清单和取舍，统一以 [`decision-records/002-source-review-and-final-integration-list.md`](decision-records/002-source-review-and-final-integration-list.md) 为准。

## 整合注意

- `Aries-AI`：补齐 API 30-33 的旧 Virtual Display 接口，并验证 EGL、IME、任务迁移和输入返回值；ROM 声明必须以真机证据为准。
- `ClosePaw`：优先复用生命周期、Binder 清理、敏感应用阻断和错误可观测性；隐藏 API 保留版本探测。
- `Operit`：Shower 功能宽，但 ColorOS、MIUI、鸿蒙已有 UI、无障碍和文件访问问题；权限按功能拆分并逐 ROM 验证。
- `Zafiro`：主要提供 Accessibility、Root/Shizuku、Python、MCP、Skill 和 Xposed，不是系统 Virtual Display 实现；厂商语音助手接管需单独适配。
- 所有参考：区分功能声明、源码实现和设备实测；移植前检查许可证、权限/数据边界、后台生命周期和失败清理。

## 整合底座与模块取舍

目标不是拼接项目，而是建立“任务控制面 + 多执行适配器”。推荐以 `ClosePaw` 的执行内核为底座，按层吸收其他项目：

| 领域 | 底座或来源 | 吸收内容 | 约束与边界 |
|---|---|---|---|
| 虚拟屏执行 | `ClosePaw` | 生命周期、会话租约、并发控制、输入/截图、任务清理、错误状态 | 补齐 API 30-33 适配；不能假设所有 ROM 支持二级屏 |
| 显示与输入适配 | `Aries-AI` | OpenGL 双路帧分发、输入签名探测、任务迁移、IME 焦点处理 | `VirtualDisplayConfig` 仅走 API 34+；输入必须返回可验证结果 |
| Agent Runtime | `Zafiro` | 无障碍树、工具注册、Skill、MCP、Python、Root/Shizuku | 作为上层运行时，不承担 Virtual Display 底座 |
| 特权服务与工具 | `Operit` | Shower AIDL、截图/视频、浏览器、终端、文件、工作流和调度 | 独立进程、按能力拆分权限；不整体引入大单体和 ROM 特判 |
| 轨迹与技能 | `MobiAgent`、`Mobile-Agent-E`、`DroidAgent`、`Ghost in the Droid` | 轨迹记录、检索、反思、脚本生成、技能注册和版本验证 | 必须统一为可参数化、可验证、可失效检测的 Skill |

全域能力由新增控制面统一管理：

```text
任务/会话控制面
  上下文、记忆、会话、Skill、权限、审计、模型路由
        |
能力路由层
  Android GUI | Virtual Display | 远程电脑 | Shell/MCP | 云端 API
        |
执行内核
  ClosePaw 生命周期 + Aries 适配与帧管线
```

必须支持端侧模型与云端 API 的统一接入，并按隐私、网络、延迟、算力和费用动态路由。GUI Agent 的探索结果应进入“GUI 轨迹编译与技能沉淀”流程：

```text
自主探索 -> 完成验证 -> 轨迹去冗余 -> 提取参数/前置条件 -> 生成脚本或 Skill -> 回放测试 -> 版本化入库
```

推荐顺序：先稳定 `ClosePaw + API 适配`，再接入 Aries 帧管线；随后接入 Zafiro Runtime 和 Operit 独立服务；最后建设远程电脑、长期记忆、云端模型路由和跨设备协同。这样牺牲部分早期功能数量，换取执行链可验证、失败可诊断、ROM 适配可扩展。

## 根工程选择与模块整合清单

### 两种根工程

| 方案 | 根工程 | 适用目标 | 优势 | 代价 |
|---|---|---|---|---|
| A | [ClawGUI](https://github.com/ZJU-REAL/ClawGUI) 的 `clawgui-agent` | 全域控制面和跨设备服务参考 | 已有任务循环、会话、记忆、Episode、模型 API、远程渠道和多设备后端 | 需要接入 Android 原生服务，并处理 Python/Kotlin 边界 |
| B | [Operit](https://github.com/AAswordman/Operit) Android 应用 | Android 为第一运行端的全域助手 | 原生工具、Shower、OCR、文档转换、向量记忆、模型、工作流和后台服务最完整 | 必须拆分大单体、权限和 ROM 特判，并补跨设备协议 |

早期方案曾选择 B（Operit 作为 Android 产品根工程）。该结论已被基于源码复核的 `002-source-review-and-final-integration-list.md` 更新：当前总根工程选择 A（ClawGUI 控制面），Operit 降级为 Android 能力适配器，ClosePaw 继续承担虚拟屏执行内核。

### 模块需求与来源

“照搬来改”表示复制指定模块作为实现起点，再改成统一接口；“吸收”表示迁移算法、接口或数据格式，不复制整个项目。

| 模块需求 | 照搬来改 | 吸收模块 | 优势 | 代价与边界 |
|---|---|---|---|---|
| 全域任务、会话、远程渠道 | `ClawGUI/clawgui-agent/nanobot` 的 agent、session、channel、provider | X-OmniClaw 多会话停止；Operit 工作流 | 控制面和 API 已有基础 | 需统一跨设备任务协议 |
| 设备能力抽象 | `KnowAct/GUIClaw/guiclaw/interfaces.py` 的 `DeviceBackend` | ClawGUI 的 Android/HDC/iOS；ClosePaw Android | 后端可插拔 | 需统一观察、动作、取消和错误 |
| GUI Agent 循环 | `ClawGUI/clawgui-agent/phone_agent/agent.py` | X-OmniClaw 观察-推理-执行-验收；Zafiro 工具路由 | 模型适配和 Episode 完整 | 与 ADB、提示词耦合较深 |
| Android 虚拟屏 | ClosePaw `VirtualDisplayPlatform`、Shizuku Transport | Aries OpenGL、输入签名、IME、任务迁移；Operit Shower 捕获 | 生命周期和清理可靠 | API 30-33、厂商 ROM 需单独适配 |
| 手机文件和相册索引 | X-OmniClaw `AlbumScanner`、`GalleryMemoryWorkflow`、`MemoryIndex` | Operit 文档解析、OCR、HNSW | 已有增量扫描和混合检索 | 当前图片主要先转视觉摘要和文本向量，原图向量需后续增加 |
| 文档、图片和 OCR | Operit `DocumentConversionUtil`、`OCRUtils` | X-OmniClaw 图片隐私过滤和 VLM 摘要 | PDF、DOCX、表格和 OCR 覆盖广 | 权限、耗时和大文件资源需限制 |
| Episode 轨迹 | ClawGUI `tracer.py` | X-OmniClaw 无障碍事件；KnowAct 状态契约 | 支持回放、训练和证据追踪 | 图片、文本和账号信息必须脱敏 |
| 轨迹编译 | KnowAct `trajectory_codegen.py`、`data.py`、`flat.py`、`state_contract.py` | DroidAgent 探索；X-OmniClaw 行为记录；Ghost 工作流格式 | 参数化、状态前置和动作固定化 | 去冗余必须删除动作后重新回放验证 |
| 快捷动作晋升 | KnowAct `shortcut_validation.py`、`deeplink.py` | X-OmniClaw Intent/deeplink 行为克隆 | `candidate -> validated -> promoted` 边界清晰 | 需真机和视觉双重验证 |
| Skill 生成和失败修订 | ClawGUI-Skills `schema.py`、`package.py`、`verifier.py`、`evolution.py` | KnowAct 状态契约；X-OmniClaw `SkillLockManager` | 有失败案例、版本快照和审计 | 要统一 SkillIR 与端侧安装格式 |
| 记忆和检索 | X-OmniClaw `MemoryIndex.kt`；ClawGUI `memory_store.py` | MobiAgent 动作树和路径缓存 | 支持全文、向量和历史路径 | 不能让缓存绕过验证 |
| Agent Runtime | Zafiro `ToolRegistry`、MCP、Python、Skill | Operit 工具注册和工作流；X-OmniClaw 工具路由 | 扩展能力强 | 权限必须由 ClawGUI 控制面统一治理 |
| 远程电脑 | KnowAct GUIClaw Windows/desktop backend | ClawGUI Gateway/Channel；Operit Shell/文件 | 与手机共用设备抽象 | 需心跳、重连、取消和授权 |
| 端侧与云端模型 | ClawGUI Provider、ModelClient、模型适配器 | X-OmniClaw VLM/STT 分离；ClawGUI-Eval OpenAI 兼容后端 | 支持本地、远程和云端路由 | 需处理费用、隐私、超时和模型格式差异 |
| 训练与评测 | ClawGUI-RL、ClawGUI-Eval 独立运行 | KnowAct 验证日志；DroidAgent/MobiAgent 轨迹 | 形成数据和指标闭环 | 不进入手机生产运行时 |

### 文件和图片语义检索

该能力统一称为“手机个人数据多模态索引与语义检索”：

```text
授权 -> 文件/相册扫描 -> 文本解析/OCR -> 图片视觉摘要 -> 文本/图像向量 -> 混合检索 -> 文件证据 -> 继续执行
```

第一阶段采用“文件内容、OCR、图片摘要、文件名和时间的文本混合检索”；第二阶段再增加 CLIP/SigLIP 类图像向量，支持以图搜图和相似图片搜索。检索结果必须返回原始 URI、缩略图、命中原因、时间、来源和可执行动作，不能只返回模型生成的文件名。

### 统一数据和状态

`KnowAct Skill/SkillStep` 作为编译中间表示，`ClawGUI-Skills SkillPackage` 作为正式技能存储，`X-OmniClaw SKILL.md` 作为端侧安装格式。三者必须通过显式转换器连接：

```text
Episode -> SkillIR -> Candidate -> Validated -> Promoted -> SkillPackage
```

正式状态只允许：`raw`、`candidate`、`validated`、`promoted`、`deprecated`。未经设备验证、结果验证和重复回放的轨迹，不得晋升为正式 Skill。

### 实施顺序

1. 固定 Operit 行为基线并拆分核心能力接口。
2. 用 ClosePaw 生命周期强化 Android 执行内核，保留 Shower 服务边界。
3. 接入 KnowAct 轨迹编译、状态契约和 `validate/promote`。
4. 接入 ClawGUI-Skills 版本、失败修订和审计。
5. 接入 X-OmniClaw、PocketSearch 和 Operit 的个人数字资产能力。
6. 接入 PowerMem 记忆生命周期和 Zafiro Runtime。
7. 接入 ClawGUI 远程渠道、电脑后端和模型路由。
8. 最后接入 ClawGUI-RL/Eval 的离线训练评测闭环。

### Ruto-GLM：Android 多虚拟屏并行参考

[Ruto-GLM](https://github.com/iamr0s/Ruto-GLM) 已浅克隆到 `AutoRefer/Ruto-GLM`。它不改变根工程选择，但对 Android 执行层有较高参考价值：

- 以会话绑定 `displayId`，为每个虚拟屏任务创建独立协程，适合多虚拟屏并行和单任务停止；
- 同时提供 `DisplayManager` 虚拟屏、`SurfaceControl` 镜像显示、`ImageReader` 和 PixelCopy 截图路径；
- 提供按 display 定向的点击、滑动、按键和文本输入；
- 通过 API 版本分支调用 `injectInputEventToTarget` 或 `injectInputEvent`；
- 提供低层动作运行时和 AutoGLM 指令解析，可作为 `SkillIR` 到 Android 原子动作的适配参考；
- 使用 LangChain4j 接入 OpenAI 兼容模型和 Gemini，并支持流式响应。

整合边界：

1. 吸收 Ruto 的“会话 -> display -> 独立任务 Job”并发模型，补入 `TaskSession` 和 `DeviceSession`。
2. 吸收 `SurfaceControl` 镜像作为 ClosePaw 的可选帧后端，不替换 ClosePaw 生命周期状态机。
3. 吸收其输入事件构造、坐标换算和 API 33 分支；必须补 ROM 签名探测、注入结果返回和失败诊断。
4. 吸收其低层动作注册思想，将 `click`、`swipe`、`text`、`launch`、`wait` 映射到统一 `Action`/`SkillStep`，不直接采用固定 marker 字符串作为正式技能格式。

限制：Ruto 目前主要是 Android 执行和会话消息框架，没有完整的长期记忆、文件/图片语义索引、轨迹编译、Skill 晋升和跨设备控制面；其隐藏 API、任务迁移验证、资源释放和输入结果处理仍需纳入真机兼容测试。它应作为 `ClosePaw + Aries` 的执行层参考，不作为 `ClawGUI` 或 `Operit` 根工程替代品。

## 目录

- `references/`：项目定位、源码索引和证据记录
- `architecture/`：分层架构、执行链路和边界
- `specs/`：统一动作、轨迹、技能、验证和回退规范
- `experiments/`：最小复现实验记录
- `evidence/`：来源、许可证和事实核验
- `decision-records/`：技术选型决策

## 如何阅读

按执行链阅读：先看 `architecture/overview.md`，理解“技能检索 -> 回放校验 -> GUI Agent 推理 -> 轨迹记录 -> 候选技能验证”的总流程；再看 `specs/skill-contract.md`，理解技能的参数、前置条件、成功判据、失效判据和回退规则。

按参考项目阅读：

- `execution/aries-ai/`：先看其 `README.md` 和技术文档，重点关注 Virtual Display、Shizuku、截图分发和输入注入。
- `runtime/zafiro/`：先看 `README.md`，再阅读 `agent-runtime/`、`app/` 和 `libs/okia/`，重点关注 Agent、工具、Skill、MCP 和权限边界。
- `memory/mobiagent/`：重点阅读 `agent_rr/` 和 ActTree 相关说明，关注状态、动作、任务关联及回放。
- `skill-generation/mobile-agent-e/`：阅读 `README.md`、`inference_agent_E.py` 和 `data/`，关注 Tips、Shortcuts 和成功轨迹演化。
- `exploration/droidagent/`：阅读探索流程和 `script/`，关注轨迹到 UIAutomator2 脚本的转换；脚本需再转换为统一动作表示。
- `skill-packaging/ghost-in-the-droid/`：阅读 `skill.yaml`、`elements.yaml`、`actions/` 和 `workflows/`，关注技能注册、版本和验证。
- `concepts/appagentx/`：作为概念参考阅读记忆和动作序列演化，不将其视为本仓库的完整运行时。

按研究问题阅读：

1. 想研究后台隔离：`execution/aries-ai/` -> `architecture/overview.md`。
2. 想研究动作复用：`memory/mobiagent/` -> `specs/skill-contract.md`。
3. 想研究技能生成：`skill-generation/mobile-agent-e/` -> `skill-packaging/ghost-in-the-droid/`。
4. 想研究离线采集：`exploration/droidagent/` -> 统一动作表示设计。

子模块默认处于上游固定提交。克隆后使用 `git clone --recurse-submodules`，或在已克隆目录执行 `git submodule update --init --recursive`。

## 许可证说明

本仓库文档采用 MIT。第三方项目保持原许可证，不随本仓库重新授权。Aries-AI 为 AGPL-3.0，直接复用代码前必须单独进行许可证评估。
