# Android AI 浏览器根工程与参考模块吸收计划

> 状态：准备阶段设计文档
>
> 日期：2026-09-09
>
> 目标：在不把多个全能助手直接拼接成单体工程的前提下，确定一个可维护的 Android 根工程，并明确各参考项目中哪些代码可以直接复用、哪些需要换语言重写、哪些只能学习方法、哪些只吸收局部特性、哪些经过删减后可以进入产品。

## 1. 目标与边界

### 1.1 目标状态

本项目首先实现一个 Android AI 浏览器，而不是一个带浏览器功能的手机 AI 助手。浏览器会话、页面观察、元素操作、用户接管、前后台切换和浏览器技能是产品主线；手机系统工具、跨 App GUI、终端和外部消息渠道只能作为辅助执行后端。

系统以持久化浏览器会话为核心：前台通过 WebView 向用户展示并允许人工接管，后台通过隐藏 WebView、虚拟屏或独立浏览器后端继续执行任务。前后台切换不依赖页面复制，而依赖会话状态保存和执行权转移。浏览器采用 DOM/结构化操作为主、视觉 GUI 为降级通道，并通过页面变化、后置条件和结果状态验证每次动作。高频成功流程可被记录、去冗余、参数化并晋升为带前置条件、后置条件、失败恢复和版本信息的浏览器 Skill。系统可选支持外部 Chrome/CDP 和浏览器扩展，但不将 Android Chrome 插件作为基础依赖。

### 1.2 当前状态 `S0`

- 已有多个参考仓库和源码快照，尚无产品代码。
- 当前工作区是研究和选型仓库，不是可发布的 Android 应用。
- 已确认的关键参考层：
  - 浏览器核心：Agentic WebView、ZorvBrowser、OperitAI WebSession、AIOPE Browser。
  - 后台执行：Aries-AI、ClosePaw、Eta。
  - Agent/GUI 降级：ClawGUI-APP、MobiAgent、X-OmniClaw。
  - 轨迹和技能：MobiAgent AgentRR、Mobile-Agent-E、ClawGUI-Skills、AppAgent-Claw。
  - 系统级能力：Eta、OpenPhone。
- 许可证并不统一，直接复制代码存在法律和分发边界。

### 1.3 目标状态 `Sg`

目标产品至少能完成以下闭环：

```text
任务输入
  -> 浏览器会话检索或创建
  -> 前台共享或后台执行模式选择
  -> DOM/结构化动作
  -> 页面变化与结果验证
  -> 失败时切换 GUI/视觉执行
  -> 记录轨迹和检查点
  -> 去冗余、参数化、技能验证
  -> 技能入库并在后续任务中复用
```

### 1.4 理想方向 `IFR`

用户不需要关心 WebView、虚拟屏、CDP、无障碍或模型切换，只需说明目标。系统自动选择成本最低且可验证的执行面；用户随时可以看到、暂停、接管或恢复任务。重复任务不再逐步调用视觉模型，而是优先执行已验证技能。

### 1.5 非目标

- 不在第一阶段实现通用手机 AI 助手、定制 Android 系统或跨 App 自动化平台。
- 不在第一阶段支持桌面浏览器、iOS 浏览器或外部消息渠道。
- 不把 Chrome Android 扩展兼容作为基础依赖；扩展只作为未来浏览器插件能力评估。
- 不把终端、SSH、语音、社交、日历、短信、文件和系统控制功能带入浏览器主链。
- 不把未经验证的轨迹直接晋升为可执行技能。

## 2. 核心矛盾与设计原则

### 2.1 核心参数矛盾 `PC`

浏览器需要同时满足两个互相拉扯的要求：

1. 用户要看到并随时接管同一个页面。
2. AI 要在后台独立执行而不占用用户前台。

如果用两个独立浏览器，会出现 Cookie、页面状态和标签页分叉；如果只用一个前台 WebView，后台运行时用户不能继续使用浏览器。因此系统必须把“浏览器会话”和“浏览器显示表面”分离。

### 2.2 统一原则

1. `BrowserSession` 是唯一状态源，显示表面只是它的挂载方式。
2. 用户操作和 AI 操作必须串行化，不能依赖“同时点击不会冲突”的假设。
3. DOM、元素引用和结构化动作优先；截图和视觉模型作为降级路径。
4. 每个动作都必须有前置条件、后置条件和失败结果。
5. 技能是经过验证的执行资产，不是模型的一段自由文本。
6. 参考项目按模块吸收，不能按仓库整体复制。
7. 许可证、数据边界和权限边界在选型阶段就必须显式记录。

## 3. 根工程选择

### 3.1 候选根工程

| 候选 | 优势 | 主要缺陷 | 结论 |
|---|---|---|---|
| 新建 `ai-browser` Android 壳 | 可以围绕浏览器会话、前后台表面和用户接管建立干净边界 | 需要吸收 Agent、轨迹、技能和 Shizuku 基础设施 | **推荐产品根工程** |
| `Agentic WebView` | Android 浏览器 SDK，已有会话协议、元素引用、失败模型、导航策略和 Compose 绑定，Apache-2.0 | 不是完整聊天 Agent，也没有手机后台调度和 GUI 降级 | **推荐浏览器核心** |
| `ZorvBrowser` | GeckoView 受控浏览器，DOM、标签页、Cookie、Storage、审计和 ACI 工具完整 | 缺少完整 Agent Runtime，仍需与产品壳整合 | 浏览器能力参考 |
| `ClawGUI-APP` | Kotlin/Compose、手机端 Shizuku、双 Agent、轨迹记录、覆盖层、模型适配、Apache-2.0 | 浏览器不是主线，当前偏 ADB/截图 Agent | 只吸收 Agent/轨迹模块 |
| Aries-AI | Virtual Display、Shizuku、输入定向、焦点隔离和后台截图已经成型 | AGPL-3.0；浏览器共享会话和技能生命周期不足 | 只吸收执行思路，或在兼容许可证下单独隔离 |
| OperitAI | 功能最全，包含 WebSession、工作流、记忆、Skill、虚拟显示和多权限通道 | 规模大、边界复杂；许可证和依赖较重；容易带入大量非目标功能 | 参考 WebSession 和工作流，不作为根工程 |
| ZorvAI + ZorvBrowser | ACI 跨应用协议，GeckoView 受控浏览器，工具契约完整 | 根工程和浏览器分属不同仓库，产品仍偏早期 | 吸收浏览器协议和 ACI 思路 |
| Eta | 系统级 Agent、后台内置浏览器、Root/Linux、用户接管和 Skill | Root + LSPosed；PolyForm Noncommercial；系统兼容成本高 | 只学习系统级能力路由和后台浏览器思路 |
| AIOPE | 聊天、浏览器、工具、RAG、工作流和多 Agent 一体化 | BSL-1.1，不适合作为可修改分发的根工程 | 只作产品形态参考 |
| RikkaHub Agent | 后台任务、80+ 工具、审批、浏览器和远程渠道 | AGPL-3.0；GUI 技能编译不是核心 | 只吸收工具治理和任务调度 |
| OpenMinis | 本地沙箱、Skill、记忆和跨平台 Agent | GPLv3；Android 浏览器自动化不是核心 | 只吸收渐进式 Skill 加载和沙箱边界 |
| 新建空白工程 | 架构和许可证最干净 | 需要重写浏览器会话、Agent、轨迹、技能和 UI | **实际采用的产品壳** |

### 3.2 根工程决策

第一阶段采用“两层根工程”决策：

```text
产品根工程：新建 product/ai-browser Android App
浏览器核心：browser/agentic-webview
```

产品根工程只负责聊天入口、浏览器会话、前台 WebView、后台执行表面、用户接管、Agent 调度、轨迹和技能库。浏览器核心负责页面观察、元素引用、结构化命令、生命周期、导航策略和结构化失败。

ClawGUI-APP 不再作为产品根工程，而作为 Agent Runtime、Shizuku、覆盖层、模型适配和轨迹记录的参考来源。这样产品的第一性对象是 `BrowserSession`，而不是 `PhoneAgent`。

理由：

- Agentic WebView 采用 Apache-2.0，适合成为浏览器协议与 WebView 宿主基础。
- 新壳可以围绕 BrowserSession 设计，不被现有手机 Agent 的 ADB/截图假设绑架。
- ClawGUI-APP 的 PhoneAgent、TraceRecorder、MemoryStore、Overlay 和 Shizuku 代码可以按模块吸收，而不把非浏览器功能带入根工程。
- ZorvBrowser 的 GeckoView、标签页、Cookie、Storage、等待、审计和页面脚本可以作为浏览器能力补充。
- 该结构天然支持未来将前台共享 WebView、后台隐藏 WebView、Virtual Display 和 CDP 作为多个执行表面接入同一会话。

### 3.3 根工程不直接继承的部分

产品根工程不直接继承以下模块：

- ClawGUI-RL 的训练集群、在线强化学习和多 GPU 调度。
- ClawGUI-Eval 的六套基准评测运行器。
- Feishu、QQ、Telegram 等外部消息通道。
- 完整 Linux/SSH/远程服务器工具。
- iOS、HarmonyOS 和桌面执行后端。
- 与浏览器无关的复杂角色卡、绘图、媒体、手机社交和通用知识库功能。
- ClawGUI-APP 的整套聊天产品界面；只移植 Agent Runtime 接口和浏览器需要的运行能力。

这些内容保留在参考仓库中，等核心浏览器闭环稳定后再按需求引入。

## 4. 复用分类标准

### 4.1 可以直接照搬

同时满足以下条件才可直接照搬：

- 许可证允许复制、修改和分发。
- 语言和运行环境与根工程一致，或无需引入额外运行时。
- 模块边界独立，接口可以保留。
- 不携带未审查的权限扩大、网络访问或数据上传逻辑。

### 4.2 可以换语言重写

保留数据结构、算法和行为契约，但将 Python、JavaScript、Rust 或主机端实现重写为 Kotlin/Android 原生模块。适用于 AgentRR、Mobile-Agent-E、DroidAgent 等研究代码。

### 4.3 只能学习思路

项目依赖定制系统、Root、厂商 Hook、特殊模型或不兼容许可证，无法合理移植代码，只记录其系统边界、数据流和决策逻辑。

### 4.4 只能吸收一两点特性

项目整体与目标无关，但有少数明确组件可迁移，例如权限分级、条件等待、页面快照或 Token 压缩。

### 4.5 删减后可用

保留目标模块，删除渠道、平台、界面、模型或基础设施扩展，使其满足 Android 单设备、浏览器任务和可验证回放的范围。

## 5. 参考项目吸收矩阵

### 5.1 ClawGUI / ClawGUI-APP

来源：[ZJU-REAL/ClawGUI](https://github.com/ZJU-REAL/ClawGUI)

| 模块 | 分类 | 处理方式 |
|---|---|---|
| `clawgui-app/app/src/main/kotlin/com/clawgui/ng/runtime/AgentRuntime.kt` | 直接照搬 | 保留运行时接口，将浏览器任务作为新的 Runtime 能力 |
| `runtime/phone/PhoneAgent.kt` | 直接照搬后修改 | 保留循环、模型适配、停止和接管；改为输出统一 Action IR |
| `runtime/phone/actions/ActionHandler.kt` | 直接照搬后修改 | 保留动作解析和失败反馈；加入浏览器动作、页面版本和后置条件 |
| `runtime/trace/TraceRecorder.kt`、`TraceStore.kt` | 直接照搬后修改 | 保留轨迹持久化；增加 DOM 快照、页面差异、会话 ID 和动作来源 |
| `runtime/phone/memory/MemoryStore.kt` | 直接照搬后修改 | 保留轻量检索；将普通记忆和可执行技能分离 |
| `runtime/overlay/*` | 直接照搬后删减 | 保留运行状态、询问和接管；暂时移除与产品无关的展示 |
| `runtime/shizuku/wadb/*` | 直接照搬后修改 | 保留 Shizuku 引导和 ADB 连接；加入 Virtual Display 能力探测 |
| `runtime/phone/model/adapters/*` | 直接照搬后删减 | 先保留 AutoGLM、Qwen-VL、UI-TARS；其余按评测需要加入 |
| RL、Eval、Feishu、媒体和多渠道模块 | 删减 | 第一阶段不进入 Android 产品根工程 |

### 5.2 Agentic WebView

来源：[shanthropic/agentic-webview](https://github.com/shanthropic/agentic-webview)

| 模块 | 分类 | 处理方式 |
|---|---|---|
| `browser-api` | 直接照搬 | 作为浏览器会话、观察、命令、错误和事件的协议基础 |
| `browser-webview` | 直接照搬后修改 | 作为前台和后台 WebView 宿主；加入会话所有权和表面挂载 |
| `browser-compose` | 直接照搬 | 作为前台浏览器 UI 绑定层 |
| `agent-tools` | 直接照搬 | 作为 JSON Schema 工具定义和调度器 |
| `integrations/jsonrpc` | 直接照搬后按需启用 | 为外部 Agent 或调试工具提供协议接入 |
| `integrations/koog` | 暂缓 | 只有根工程选择 Koog 时才启用 |
| `web-runtime` | 直接照搬并审查 | 保留 DOM 观察和元素引用逻辑，审查脚本注入和敏感信息处理 |
| website、演示和发布脚本 | 删减 | 不进入 Android 产品 |

这是浏览器底层的首选来源，因为它把 WebView、协议、页面状态和 Agent 工具分开，且 Apache-2.0。

### 5.3 ZorvBrowser

来源：[Quor-a/ZorvBrowser](https://github.com/Quor-a/ZorvBrowser)

| 模块 | 分类 | 处理方式 |
|---|---|---|
| GeckoView 浏览器宿主 | 只能学习思路 | 不与 Agentic WebView 同时作为第一内核；先保留为替代内核方案 |
| `browser_action` | 吸收特性 | 参考稳定 ID、CSS 选择器、click/type/scroll/select |
| `browser_wait` | 直接吸收思路 | 加入条件等待和网络空闲判断 |
| `browser_snapshot` / `browser_restore` | 吸收特性 | 转换为 BrowserSession 检查点，不承诺完整页面回滚 |
| Cookie、Storage、标签页、下载和截图 | 吸收特性 | 逐项加入 BrowserSession 能力矩阵 |
| ACI/AIDL 控制协议 | 换语言重写 | 用 Kotlin Binder 或本地接口重写，避免绑定 ZorvAI 主工程 |
| `inject_touch`、任意 HTTP 和抓包能力 | 暂缓 | 默认关闭，经过权限和安全审查后再加入 |

### 5.4 Aries-AI

来源：[ZG0704666/Aries-AI](https://github.com/ZG0704666/Aries-AI)

许可证：AGPL-3.0。

| 模块 | 分类 | 处理方式 |
|---|---|---|
| `VirtualDisplayController.kt` | 只能学习思路或隔离复用 | 研究显示创建、销毁、生命周期；默认采用自有实现 |
| `vdiso/ShizukuVirtualDisplayEngine.kt` | 只能学习思路 | 研究 Shizuku 权限和虚拟屏绑定，不直接复制到非 AGPL 根工程 |
| `input/VirtualAsyncInputInjector.kt` | 换语言重写 | 以自有接口重写显示定向输入和输入队列 |
| `core/cache/ScreenshotManager.kt` 等 | 吸收特性 | 参考截图节流、非黑帧检测、缓存和覆盖层保护 |
| `VirtualScreenPreviewOverlay.kt` | 直接照搬后删减 | 若许可证路径允许，保留预览交互；否则自有实现 |
| Aries 聊天、语音、遥测和完整 UI | 删减 | 不进入根工程 |

重要决策：如果产品根工程保持 Apache-2.0 或闭源兼容路径，Aries-AI 只能作为行为参考或独立进程服务，不能将 AGPL Kotlin 文件直接复制进来。

### 5.5 OperitAI

来源：[Moole123/operit](https://github.com/Moole123/operit)

| 模块 | 分类 | 处理方式 |
|---|---|---|
| `WebSessionBrowserHost` | 换语言重写或局部吸收 | 参考浏览器宿主、生命周期和最小化状态 |
| `BrowserToolSupport` | 换语言重写 | 参考浏览器工具与聊天 Agent 的连接方式 |
| `BrowserPageExecutionSupport` | 吸收特性 | 参考页面脚本、等待、下载和结果提取 |
| `WebSessionHistoryStore` | 直接吸收思路 | 转化为 BrowserSession 历史与检查点存储 |
| `WorkflowExecutor` / `WorkflowScheduler` | 删减后可用 | 只保留单设备定时任务和可恢复队列 |
| `VirtualDisplayManager` | 只能学习思路 | 不把 Operit 的显示层与 Aries/自有实现混合 |
| Ubuntu、SSH、角色卡、绘图、媒体和大量工具 | 删减 | 第一阶段移除 |

OperitAI 适合作为产品功能参考，不适合作为根工程；其模块可以帮助补足 AIOPE/ClawGUI 中缺少的 WebSession 生命周期。

### 5.6 MobiAgent / AgentRR

来源：[IPADS-SAI/MobiAgent](https://github.com/IPADS-SAI/MobiAgent)

许可证：Apache-2.0。

| 模块 | 分类 | 处理方式 |
|---|---|---|
| `agent_rr/action_cache/tree.py` | 换语言重写 | Kotlin 实现 UI 状态—动作—任务关联的 ActTree |
| `ActionTreeNode.try_find_shortcuts()` | 换语言重写 | 保留重复前缀、分叉节点和快捷动作发现算法 |
| `ActionTreeNodeFuzzy` | 换语言重写 | 使用本地嵌入或轻量文本匹配进行任务检索 |
| `target_elem_changed()` | 直接吸收思路 | 加入元素内容、区域和页面版本变化校验 |
| AgentRR 训练嵌入和重排序 | 删减 | 第一阶段用规则、BM25 或轻量嵌入替代训练模型 |

AgentRR 是动作复用层，不是浏览器内核；必须通过 Action IR 连接到 WebView 和 GUI 执行器。

### 5.7 Mobile-Agent-E

来源：[X-PLUG/MobileAgent/Mobile-Agent-E](https://github.com/X-PLUG/MobileAgent/tree/main/Mobile-Agent-E)

| 模块 | 分类 | 处理方式 |
|---|---|---|
| `InfoPool` | 换语言重写 | 将任务、动作历史、结果和未来任务建模为 Kotlin 数据对象 |
| `ExperienceReflectorShortCut` | 换语言重写 | 参考成功轨迹反思和 Shortcut 候选生成 |
| `ExperienceReflectorTips` | 吸收特性 | 将错误经验作为非执行性提示保存 |
| Shortcut 的前置条件与原子动作序列 | 直接吸收数据结构 | 迁移为 Skill Candidate Schema |
| ADB 坐标执行器 | 删减 | 改为 Web 元素引用、相对区域和 GUI 后端动作 |
| 独立 Python 推理脚本 | 删减 | 不放进 Android 根工程 |

### 5.8 ClawGUI-Skills

来源：[ClawGUI-Skills](https://github.com/ZJU-REAL/ClawGUI/tree/master/clawgui-skills)

| 模块 | 分类 | 处理方式 |
|---|---|---|
| `meta_info.json`、`plan.md`、`backup.md`、`recover.md` | 直接吸收结构 | 转为 Kotlin 序列化的技能包格式 |
| `trace / reuse / evolve` 三种模式 | 直接吸收思路 | 作为 SkillRuntime 的状态机 |
| `failure_examples/` | 直接吸收结构 | 保存失败轨迹和可复用教训 |
| 受限文件修订 | 换语言重写 | 改为只允许更新技能目录，不允许修改产品源代码 |
| 版本、运行和编辑日志 | 直接吸收 | 进入 SkillAuditStore |
| Python PhoneAgent 集成 | 换语言重写 | 用 Kotlin AgentRuntime 接口替换 |

这是技能生命周期的主参考，比单纯把动作序列写入 Markdown 更可审计。

### 5.9 AppAgentX 与 AppAgent

来源：[AppAgentX](https://github.com/Westlake-AGI-Lab/AppAgentX)、[AppAgent](https://github.com/TencentQQGYLab/AppAgent)

| 模块 | 分类 | 处理方式 |
|---|---|---|
| 探索阶段与部署阶段分离 | 只能学习思路 | 作为“探索资产”和“运行时技能”分层依据 |
| 元素文档和应用知识库 | 吸收特性 | 作为 AppProfile 和 BrowserSiteProfile 的语义层 |
| Neo4j/Pinecone 组合 | 只能学习思路 | 第一阶段不引入外部数据库 |
| 重复动作演化为高级动作 | 只能学习思路 | 由 AgentRR + Skill Candidate 实现 |
| 原有 Python/ADB 运行链 | 删减 | 不作为 Android 根工程 |

两者适合提供概念和数据模型，不应被当成可直接嵌入的完整运行时。

### 5.10 DroidAgent

来源：[coinse/droidagent](https://github.com/coinse/droidagent)

| 模块 | 分类 | 处理方式 |
|---|---|---|
| 自主探索器 | 换语言重写或独立工具 | 第一阶段保留为离线探索工具，不放入运行时 |
| exploration history | 直接吸收数据结构 | 转为统一轨迹格式 |
| UIAutomator2 脚本生成 | 只能学习思路 | 先生成 Action IR，再由 Android 后端执行 |
| DROIDBOT 测试基础设施 | 删减 | 不进入产品 APK |

DroidAgent 的脚本生成只适合作为离线技能种子和回归测试来源，不能直接作为共享 WebView 的执行协议。

### 5.11 Ghost in the Droid

来源：[ghost-in-the-droid/android-agent](https://github.com/ghost-in-the-droid/android-agent)

许可证：MIT。

| 模块 | 分类 | 处理方式 |
|---|---|---|
| `skill.yaml` | 直接吸收结构 | 转为技能元数据模型 |
| `elements.yaml` | 直接吸收结构 | 转为站点或应用元素定位器集合 |
| `actions/` | 换语言重写 | 改为 Kotlin Action IR 和执行器 |
| `workflows/` | 换语言重写 | 改为可组合的 Skill Step |
| BFS App Explorer | 只能学习思路 | 作为浏览器站点探索或 App 探索的离线任务 |
| Skill Hub、iOS、手机农场和 Vue Dashboard | 删减 | 第一阶段移除 |

### 5.12 Eta、OpenPhone、OpenMinis、RikkaHub Agent、AIOPE

这些项目的主要用途是提供系统级和产品级参考，不作为第一阶段代码底座。

| 项目 | 只吸收的部分 | 明确不复制的部分 |
|---|---|---|
| [Eta](https://github.com/Mangi-11/Eta) | 后台内置浏览器、用户接管、系统能力路由、渐进式 Skill、运行恢复 | Root/LSPosed Hook、厂商助手接管、PolyForm 代码 |
| [OpenPhone](https://github.com/secondly-com/OpenPhone) | 系统级 Agent、Watcher、Heartbeat、审批、审计和持续任务 | 定制 Android/LineageOS、特权系统组件和商业限制代码 |
| [OpenMinis](https://github.com/OpenMinis/OpenMinis) | 按需加载 Skill、本地沙箱、跨会话记忆 | GPLv3 沙箱整体、iOS 运行时和非目标工具 |
| [RikkaHub Agent](https://github.com/ExTV/rikkahub-agent) | 后台调度、工具开关、审批、内置浏览器和会话恢复 | AGPL 根工程、80+ 非浏览器工具和完整聊天 UI |
| [AIOPE](https://github.com/XNet-NGO/aiope) | 共享 WebView 产品形态、自动运行、模型按任务路由、浏览器工具命名 | BSL 源码、完整 UI、网关、RAG 和大量工具 |

## 6. AI 浏览器根工程模块清单

### 6.1 必须保留或新增

| 模块 | 责任 | 第一来源 |
|---|---|---|
| `browser-session` | URL、标签页、Cookie、Storage、页面版本、任务检查点；产品核心状态源 | Agentic WebView、ZorvBrowser |
| `browser-surface` | 前台 WebView 显示、用户触摸和人工接管；产品首要用户表面 | Agentic WebView、AIOPE |
| `browser-background` | 隐藏 WebView、虚拟屏或独立 Chromium 后端；同一会话的后台表面 | Aries-AI、Eta、ClosePaw |
| `handoff-controller` | AI、用户、系统三类控制权转移 | ClawGUI-APP、Eta |
| `browser-observer` | DOM、元素、正文、截图、页面差异和网络空闲 | Agentic WebView、ZorvBrowser |
| `action-ir` | 统一表达 DOM、GUI、虚拟屏和脚本动作 | MobiAgent、DroidAgent、Ghost |
| `dom-executor` | 元素引用、CSS、表单、滚动、脚本和等待 | Agentic WebView、ZorvBrowser |
| `gui-executor` | 截图、无障碍、视觉定位和降级动作 | ClawGUI-APP、Aries-AI |
| `virtual-display-executor` | Shizuku 显示创建、输入定向和截图 | Aries-AI 思路，自有实现 |
| `agent-runtime` | 浏览器任务规划、模型调用、工具循环、停止和恢复 | ClawGUI-APP、ZorvAI |
| `trace-recorder` | 记录观察、动作、结果、耗时、错误和截图 | ClawGUI-APP、ClawGUI-Skills |
| `skill-engine` | 检索、候选生成、验证、晋升、版本和失效 | ClawGUI-Skills、Mobile-Agent-E |
| `action-memory` | ActTree、任务相似度、动作缓存和回放 | MobiAgent AgentRR |
| `verification` | 前置条件、后置条件、页面差异和结果确认 | Agentic WebView、AppAgent-Claw |
| `recovery` | 超时、元素失效、用户接管、进程恢复和 GUI 降级 | ClawGUI-APP、ZorvBrowser、Eta |
| `security-policy` | 域名、脚本、Cookie、文件、敏感操作和审批策略 | Agentic WebView、Aartiq、RikkaHub Agent |
| `storage` | Room、技能包、轨迹、审计和加密配置 | ClawGUI-APP |

### 6.2 可选模块

- `chrome-cdp-bridge`：接管用户真实 Chrome 或 Termux Chromium。
- `extension-host`：加载受控脚本或扩展兼容层。
- `site-profile`：保存站点级元素语义、流程和登录态提示。
- `exploration-runner`：离线探索网页或 App 并生成技能种子。
- `scheduler`：定时和事件触发的浏览器任务。
- `remote-channel`：Feishu、Telegram 或其他外部入口，不能反向定义浏览器会话模型。
- `local-llm`：端侧模型和低成本观察器。

## 7. 应从根工程删掉什么

第一阶段以新建 AI 浏览器壳和 Agentic WebView 为底座时，删除或暂缓：

1. ClawGUI-RL 全部训练环境、在线强化学习和奖励模型。
2. ClawGUI-Eval 的多基准、多 GPU 和论文复现实验代码。
3. Feishu 通道和其他外部消息渠道，先保留本地浏览器聊天入口。
4. 不参与浏览器执行的媒体处理、角色卡、图片生成、手机社交和远程服务器功能。
5. iOS、HarmonyOS、桌面和云端设备后端。
6. 除 AutoGLM、Qwen-VL、UI-TARS 外暂时不用的模型适配器。
7. 完整 Linux Shell；仅保留构建和调试所需的最小命令能力。
8. 用户画像和通用对话记忆中与浏览器任务无关的字段。
9. 复杂的多 Agent DAG；第一阶段只保留一个浏览器规划 Agent 和一个浏览器执行 Agent。
10. 未经验证的自动 Skill 晋升；默认只生成候选，不自动启用。

不能删除：浏览器会话状态、前台 WebView、后台表面、停止和接管、轨迹记录、动作解析、页面验证、错误反馈和模型适配接口。Shizuku 引导、覆盖层和 GUI 降级属于第二层能力，不能先于浏览器核心。

## 8. 需要新增什么

### 8.1 BrowserSession 数据

```text
session_id
surface_mode
active_tab_id
tabs
cookies_ref
storage_ref
current_url
document_version
page_fingerprint
control_owner
checkpoint_id
task_status
last_verified_result
```

### 8.2 Action IR

```text
action_id
session_id
backend
operation
target_ref
arguments
precondition
postcondition
document_version
timeout_ms
retry_policy
fallback_action
risk_level
```

### 8.3 Skill 包

```text
skill_id
name
intent
scope
input_schema
preconditions
steps
checkpoints
success_condition
recovery_plan
failure_examples
confidence
version
last_verified_at
invalidation_rules
```

### 8.4 控制权状态机

```text
AI_ACTIVE
USER_REQUESTED_TAKEOVER
USER_ACTIVE
PAUSE_PENDING
PAUSED
RESUME_CHECK
RECOVERY_REQUIRED
COMPLETED
FAILED
```

状态转换必须写入审计日志。用户接管后，AI 不得继续发送点击、输入、导航或脚本动作。

## 9. 技能生成与复用流程

```text
成功任务轨迹
  -> 轨迹规范化
  -> 去除等待、重复观察和无效点击
  -> 识别稳定目标和可变参数
  -> 生成 Skill Candidate
  -> 静态检查
  -> 沙盒或测试账号回放
  -> 后置条件验证
  -> 通过后晋升
  -> 失败后记录 failure_examples
```

技能晋升最低条件：

- 至少两次独立成功回放，或一次人工确认加一次自动验证。
- 可检测的前置条件和后置条件。
- 输入参数有明确 Schema。
- 不依赖失效坐标或临时文本。
- 失败时能够停止或回退到 GUI Agent。
- 支付、发消息、删除、发布和权限修改等高风险末步默认需要确认。

## 10. 优劣分析

### 10.1 新建 AI 浏览器产品壳

优势：

- 第一性对象是 `BrowserSession`，不会被手机助手的工具数量和 ADB 任务模型带偏。
- 可以把前台 WebView、后台 WebView、Virtual Display 和 CDP 设计为同一会话的可替换表面。
- 浏览器权限、Cookie、脚本、页面观察和用户接管可以在同一个边界内治理。
- 产品壳不继承任何不必要的聊天、终端、社交和系统工具。

代价：

- 需要自己实现产品入口、会话状态、后台生命周期和基础 Agent 编排。
- 需要吸收 ClawGUI-APP 的轨迹、覆盖层和 Shizuku 代码，而不能整体复制。
- 第一阶段开发量比直接套用全能助手更集中，但更符合目标。

### 10.2 采用 Agentic WebView 为浏览器核心

优势：

- 已经把 WebView、观察、元素引用、命令、生命周期和错误边界拆开。
- 元素引用按文档和帧作用域管理，页面更新后能显式报告失效。
- 有导航策略、敏感信息脱敏、Shadow DOM/同源 iframe 和结构化工具接口。
- Apache-2.0，适合作为浏览器核心直接接入和继续修改。

代价：

- 不提供完整的聊天 Agent、后台任务和 Android 任务恢复。
- 需要补充持久化 BrowserSession、前后台表面和 Skill 生命周期。
- WebView 仍受 Android 进程、Cookie 和站点安全策略限制。

### 10.3 直接以 OperitAI 为根工程

优势：功能多，WebSession、Workflow、Memory 和悬浮窗较完整。

代价：功能面过大，删除成本高；浏览器、终端、Agent 和插件边界复杂；许可证和第三方依赖需要重新审查。它适合作为产品参考，不适合作为第一根工程。

### 10.4 直接以 Aries-AI 为根工程

优势：虚拟屏和 Shizuku 执行最贴近后台需求。

代价：AGPL-3.0；浏览器共享 WebView、技能系统和完整任务状态不足。直接复制会把许可证和架构耦合同时带入，不建议作为默认路径。

### 10.5 直接以 ZorvBrowser 为根工程

优势：浏览器工具契约完整，DOM、标签页、Cookie、Storage 和审计边界清晰。

代价：缺少任务记忆、GUI 降级和 Shizuku 虚拟屏。它适合作为浏览器执行子模块，不适合作为完整 AI 浏览器产品根工程。

### 10.6 结论

不选择 OperitAI、Aries-AI、ZorvBrowser 或 ClawGUI-APP 作为整仓根工程。产品采用“新建浏览器壳 + Agentic WebView 核心 + 选择性吸收”的路径，其他仓库只提供受限模块或设计参考。

## 11. 推荐的吸收顺序

### 阶段 0：AI 浏览器壳

- 新建 `product/ai-browser` Android 工程。
- 只建立浏览器入口、会话仓储、前台 WebView、聊天任务面板和最小设置页。
- 固定 Android、Kotlin、Compose、Room 和 Agentic WebView 边界。
- 加入许可证与第三方声明清单。

### 阶段 1：浏览器会话

- 将 Agentic WebView 的 `browser-api`、`browser-webview` 和 `agent-tools` 作为第一批核心依赖接入。
- 实现 BrowserSession、标签页、页面版本和前台 WebView。
- 实现用户触摸和 AI 工具的动作锁。

### 阶段 2：后台表面

- 先实现隐藏 WebView 后台运行。
- 再实现 Shizuku Virtual Display 后端。
- 最后评估独立 Chromium/CDP 后端。
- 任何后端都必须返回统一 ActionResult。

### 阶段 3：浏览器执行闭环

- 实现 DOM/结构化动作、条件等待、页面变化、后置条件和结果验证。
- 引入最小模型适配器，仅支持一个稳定的浏览器 Agent 模型。
- DOM/元素操作失败时，再引入视觉 GUI 降级；此时只吸收 ClawGUI-APP 的截图和动作解析模块。

### 阶段 4：浏览器轨迹与技能

- 只记录浏览器观察、元素引用、动作、页面差异和后置结果。
- 移植 AgentRR 的最小 ActTree 和任务匹配。
- 移植 Mobile-Agent-E 的 Shortcut 候选生成，但限定为浏览器动作。
- 移植 ClawGUI-Skills 的 `trace / reuse / evolve` 生命周期。
- 加入 AppAgent-Claw 风格的录制、参数化、分层定位和后置验证。

### 阶段 5：后台和浏览器增强

- 吸收 ZorvBrowser 的条件等待、标签页、Cookie、Storage、快照和审计。
- 实现同一 BrowserSession 的隐藏 WebView 后台表面。
- 再实现 Shizuku Virtual Display 表面和用户接管预览。
- 最后评估 Chrome/CDP、浏览器扩展和远程控制；这些不阻塞 AI 浏览器 MVP。

## 12. 最终清单

### 12.1 直接复制或近似复制

- ClawGUI-APP 的 `AgentRuntime` 接口、轨迹仓储、覆盖层、Shizuku 引导和基础模型适配中的必要部分；不复制其 Android 应用壳和完整聊天 UI。
- Agentic WebView 的浏览器协议、WebView 宿主、Compose 绑定、工具 Schema 和结构化错误。
- ZorvBrowser 中许可证允许且边界清晰的浏览器数据结构，需逐文件保留声明。
- ClawGUI-Skills 的技能包字段、模式和审计结构，改为 Kotlin 序列化。

### 12.2 换语言重写

- MobiAgent AgentRR 的 ActTree、模糊任务匹配和快捷动作发现。
- Mobile-Agent-E 的 InfoPool、Shortcut 生成和经验反思。
- DroidAgent 的 exploration history 到 Action IR 的转换。
- Ghost in the Droid 的 `skill.yaml`、`elements.yaml`、动作和工作流模型。
- Aries-AI 的 Virtual Display 输入注入、截图节流和焦点隔离。
- OperitAI 的 WebSession 生命周期、页面执行和工作流调度。

### 12.3 只能学习思路

- AppAgentX 的图记忆和动作演化思想。
- AppAgent 的探索/部署分离和应用文档生成。
- Eta 的系统级能力路由、后台浏览器和用户接管。
- OpenPhone 的系统特权 Agent、Watcher、Heartbeat 和审计模型。
- OpenMinis 的渐进式 Skill 加载和本地沙箱边界。

### 12.4 只吸收局部特性

- ZorvBrowser：条件等待、稳定元素 ID、快照、Cookie、Storage、审计。
- AIOPE：共享 WebView 产品形态、自动连续运行和浏览器工具命名。
- RikkaHub Agent：工具开关、审批、后台任务和恢复诊断。
- ClosePaw：真实 Chrome CDP、浏览器脚本、虚拟屏和运行状态浮层。
- Aartiq：风险分级、计划解释和执行前审批。

### 12.5 删除或暂缓

- 所有桌面、iOS、HarmonyOS、手机农场和云端设备模块。
- RL 训练、论文评测和多 GPU 服务。
- 与浏览器任务无关的终端、SSH、媒体、绘图、角色卡和大量社交渠道。
- 未验证的自动技能晋升、任意 JavaScript、任意 HTTP、任意抓包和高风险自动提交。

## 13. 验证指标

第一阶段不以“工具数量”作为成功标准，而以以下指标验证：

| 指标 | 目标 |
|---|---|
| 前台共享会话切换 | 切换后 URL、标签页和页面状态保持 |
| 后台执行连续性 | 退出浏览器表面后任务继续运行 |
| 用户接管 | 接管后 AI 动作立即停止，恢复前重新验证页面 |
| DOM 动作成功率 | 在测试站点和常见 SPA 中稳定执行 |
| GUI 降级成功率 | DOM 失败时能进入视觉执行并返回结果 |
| 技能回放成功率 | 验证设备上连续成功，失败可回退 |
| Token 节省 | 重复任务相较冷启动减少模型调用和页面上下文 |
| 误操作率 | 高风险末步默认不自动执行 |
| 进程恢复 | 被系统杀死后能恢复到最近检查点 |
| 许可证完整性 | 每个复制模块均有来源、版本和声明记录 |

## 14. 当前结论

产品根工程选择：

```text
新建 product/ai-browser Android App
```

浏览器核心：

```text
Agentic WebView
```

浏览器能力补充：

```text
ZorvBrowser + OperitAI WebSession
```

后台执行：

```text
自有隐藏 WebView
  -> 自有 Shizuku Virtual Display
  -> 可选 Chromium/CDP
```

浏览器任务 Agent：

```text
最小 AgentRuntime
  + ClawGUI-APP PhoneAgent 的必要部分
  + 单一浏览器模型适配器
```

轨迹和技能：

```text
MobiAgent AgentRR
  + Mobile-Agent-E Shortcut
  + ClawGUI-Skills 生命周期
  + AppAgent-Claw 录制与验证
```

手机 GUI 和系统级能力不是本项目主线，只作为浏览器失败时的有限降级后端。核心边界是：

```text
浏览器会话
  -> 前台/后台表面
  -> Action IR
  -> DOM/GUI/Virtual Display 后端
  -> 轨迹记录
  -> 技能验证与复用
```

只有浏览器主链稳定后，才值得增加 Chrome/CDP、扩展兼容、手机 GUI 降级、远程渠道和系统级入口。

## 15. 下一轮实施入口

1. 新建 `product/ai-browser` 最小 Android 浏览器壳。
2. 将 Agentic WebView 作为唯一第一阶段浏览器核心接入。
3. 定义 `BrowserSession`、`Action IR`、`ActionResult` 和 `Skill` 的 Kotlin 数据结构。
4. 实现前台共享 WebView、用户接管和 AI 动作锁。
5. 完成 DOM/结构化操作、页面变化和后置条件验证。
6. 以隐藏 WebView 完成第一个后台浏览器任务闭环。
7. 移植 AgentRR 的最小 ActTree，验证浏览器动作缓存。
8. 加入 Skill Candidate 的人工确认和回放验证。
9. 再评估 Shizuku Virtual Display、视觉 GUI、Chrome/CDP 和扩展能力。

本文件只确定根工程、模块吸收边界和实施顺序，不代表已经完成代码迁移或许可证法律审查。
