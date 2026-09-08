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
