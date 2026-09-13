# 参考项目索引

本目录只说明“项目在整合中的角色”。具体源码文件、远程地址、固定提交和证据等级统一维护在 [`../evidence/source-index.yaml`](../evidence/source-index.yaml)。

## 当前角色

| 角色 | 项目 | 结论 |
|---|---|---|
| 浏览器核心 | Agentic WebView、ZorvBrowser | WebView、DOM/可访问性、标签页、Cookie、Storage 和浏览器动作 |
| 产品运行时参考 | ClawGUI、ClawGUI-APP | 任务循环、模型适配、会话、轨迹、覆盖层和远程协议 |
| Android 能力 | Operit | WebSession、文件、OCR、工具、工作流和权限边界 |
| 后台执行 | ClosePaw、Aries-AI、Ruto-GLM | Virtual Display、Shizuku、输入、截图、生命周期和多 display |
| 轨迹与 Skill | KnowAct、Ghost in the Droid、DroidAgent、MobiAgent、Mobile-Agent-E | 编译、蒸馏、探索、动作缓存和反思 |
| Runtime 扩展 | Zafiro | ToolRegistry、MCP、Python、Skill 和 Shell 安全策略 |
| 资产与记忆扩展 | PowerMem、X-OmniClaw、local-photo-search、PocketSearch | 作为未来全域扩展的支撑材料，不进入浏览器第一阶段核心 |
| 概念或服务端参考 | AppAgentX、EagleRAG、ColPali、Qdrant、clip-as-service | 只迁移结构、算法或 Provider 边界 |

## 证据等级

- `L1`：已读取本地官方源码；
- `L2`：官方文档、论文或项目说明；
- `L3`：本地复现实验；
- `L4`：基于证据的推断；
- `L5`：待确认。

当前大多数项目已达到 `L1` 源码核查，但 ROM、性能、功耗和中文召回仍属于 `L3/L5`，不能把静态源码结论当成设备兼容承诺。

## 子模块与本地研究目录

仓库内 Git 子模块用于固定少量核心参考项目；`AutoRefer` 中的其他目录用于扩大源码检索范围，不自动成为本仓库子模块。是否纳入子模块要在确认提交稳定、目录边界和更新成本后单独决策。

