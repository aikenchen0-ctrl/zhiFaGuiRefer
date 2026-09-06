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

## 目录

- `references/`：项目定位、源码索引和证据记录
- `architecture/`：分层架构、执行链路和边界
- `specs/`：统一动作、轨迹、技能、验证和回退规范
- `experiments/`：最小复现实验记录
- `evidence/`：来源、许可证和事实核验
- `decision-records/`：技术选型决策

## 许可证说明

本仓库文档采用 MIT。第三方项目保持原许可证，不随本仓库重新授权。Aries-AI 为 AGPL-3.0，直接复用代码前必须单独进行许可证评估。

