# zhiFaGuiRefer

Android AI 浏览器的参考、架构和迁移准备仓库。

当前只做源码分析、架构决策、权限边界、模块分类和验证准备，不包含可直接发布的产品实现，也不把多个第三方项目拼成一个单体。

集中审阅和删改请只打开 [整合审阅总稿](INTEGRATED-REVIEW-DRAFT.md)。该总稿已合并当前方案、历史扩展、参考源码、权限、记忆、数据库、技能和实验缺口；其他文档暂作为来源和回溯材料。

## 当前产品目标

第一阶段聚焦 **Android AI 浏览器**：用户和 AI 共享同一个持久化 `BrowserSession`，前台可观察和接管，后台可继续执行，重复流程可沉淀为经过验证的 Skill。

```text
任务
  -> BrowserSession
  -> 前台共享 WebView / 后台隐藏 WebView / Virtual Display / 外部浏览器
  -> DOM/可访问性观察 -> 结构化动作 -> 页面变化与结果验证
  -> 轨迹清洗 -> 参数化 -> 回放验证 -> Skill
```

## 唯一推荐阅读路径

| 顺序 | 文档 | 用途 |
|---|---|---|
| 1 | [整合审阅总稿](INTEGRATED-REVIEW-DRAFT.md) | 集中审阅全部当前方案、历史扩展、权限、记忆、数据库、Skill 和验证内容 |
| 2 | [AI 浏览器总体架构](architecture/ai-browser-overview.md) | 会话、前后台表面、执行权、观察、动作、验证和第一阶段边界 |
| 3 | [参考项目吸收计划](architecture/reference-absorption-plan.md) | 根工程候选、五类迁移策略、模块来源、删减项和实施顺序 |
| 4 | [技能契约](specs/skill-contract.md) | 参数、前置条件、成功判据、失效判据和回退 |
| 5 | [架构文档索引](architecture/README.md) 与 [决策文档索引](decision-records/README.md) | 当前架构、专题决策和历史材料 |
| 6 | [来源与证据索引](evidence/README.md) | 项目来源、源码文件、证据等级和验证缺口 |

## 当前架构结论

产品根工程采用“**新建 Android 产品壳 + Agentic WebView 浏览器核心**”，不把 Operit、ClawGUI、ClosePaw 或 Aries-AI 整体作为产品根工程。

| 层 | 当前选择 | 负责内容 |
|---|---|---|
| 产品壳 | 新建 `ai-browser` Android App | BrowserSession、聊天入口、前后台表面、执行权、用户接管和任务调度 |
| 浏览器核心 | Agentic WebView | WebView、DOM/可访问性观察、元素引用、结构化命令和页面错误 |
| Android 能力 | Operit 局部模块 | WebSession、文件、OCR、工具和工作流参考 |
| 后台执行 | 自有实现，参考 ClosePaw/Aries-AI | Virtual Display、Shizuku、截图、输入和生命周期 |
| 浏览器产品参考 | ZorvBrowser | GeckoView、标签页、Cookie、Storage、下载和审计 |
| 轨迹与技能 | KnowAct、ClawGUI-Skills、Ghost | SkillIR、蒸馏、验证、版本、失败修订和回放 |

前台与后台不是两个页面副本，而是同一 `BrowserSession` 的不同执行表面。每次切换都必须保存检查点、转移执行权并重新观察页面。

## 参考项目分工

| 项目 | 迁移定位 |
|---|---|
| Agentic WebView | 浏览器会话、DOM/可访问性观察和浏览器动作核心 |
| ZorvBrowser | GeckoView 浏览器、标签页、Cookie、Storage、下载和审计参考 |
| ClawGUI / ClawGUI-APP | Agent Runtime、模型适配、任务循环、轨迹和远程协议 |
| Operit | Android 工具、WebSession、文件、OCR、工作流和权限边界 |
| ClosePaw | 虚拟屏生命周期、Shizuku、输入、截图和资源清理 |
| Aries-AI / Ruto-GLM | 多版本显示/输入探测、帧分发和多 display 并发 |
| KnowAct / Ghost in the Droid | 轨迹编译、回放、Skill 包、校验和人工接管 |
| MobiAgent / Mobile-Agent-E / DroidAgent | 动作缓存、反思、探索和脚本化参考 |
| Zafiro | ToolRegistry、MCP、Python、Skill 和 Shell 安全策略 |

完整源码证据和模块取舍只维护在 [参考项目吸收计划](architecture/reference-absorption-plan.md) 和 [源码索引](evidence/source-index.yaml)，不在 README 重复展开。

## 五类迁移策略

1. **直接照搬再适配**：许可证、语言、边界和权限都允许时，复制局部模块并改接口；
2. **换语言重写**：保留算法、数据结构和行为契约，改写为 Kotlin 或独立服务；
3. **只能学习思路**：只迁移架构原则，不迁移运行时和基础设施；
4. **只吸收局部特性**：只迁移一个协议、算法或边界；
5. **删减后使用**：隔离 UI、实验、渠道、厂商宿主、模型和非目标依赖。

## 高内聚、低耦合规则

- `BrowserSession` 是浏览器状态唯一事实源；执行表面只持有会话引用；
- DOM、可访问性和结构化浏览器接口优先，截图/视觉/坐标只作为降级通道；
- 每个动作必须有前置条件、后置条件、超时、失败结果和证据；
- 用户接管优先于后台动作，动作边界必须可暂停和恢复；
- 第三方项目只能通过 Adapter/Provider 接入，不跨项目读取私有数据库或 UI 状态；
- 未经重复回放验证的轨迹只能是候选 Skill，不得覆盖推理路径。

## 相关全域助手研究

此前对手机个人文件、图片向量化、长期记忆、Android 权限和双核心全域助手的研究仍保留在决策文档中，作为浏览器未来扩展的支撑材料：

- [Android 资产权限矩阵](decision-records/003-android-asset-permission-matrix.md)
- [记忆系统选型](decision-records/004-frontier-memory-system-selection.md)
- [Android 数据库架构选型](decision-records/005-android-database-architecture-selection.md)
- [历史全域助手方案](decision-records/001-main-project-and-reference-modules.md)
- [历史源码整合清单](decision-records/002-source-review-and-final-integration-list.md)

这些材料不改变当前 AI 浏览器第一阶段边界。集中审阅请直接编辑 [整合审阅总稿](INTEGRATED-REVIEW-DRAFT.md)。

## 当前范围

已完成：参考源码核查、浏览器目标架构、根工程候选、模块吸收分类、技能契约和权限/记忆/数据库支撑决策。

尚未开始：产品代码、Android 真机 ROM 矩阵、性能评测和正式实验。实验计划见 [experiments/README.md](experiments/README.md)。

## 目录

- `architecture/`：当前 AI 浏览器架构和参考项目迁移方案
- `decision-records/`：当前与历史技术决策
- `evidence/`：源码来源、证据等级和核查范围
- `specs/`：跨实现的动作与技能契约
- `experiments/`：实验计划和结果
- `execution/`、`runtime/`、`memory/`、`skill-packaging/`：参考项目子模块
