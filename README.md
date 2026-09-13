# Android AI 浏览器参考与设计仓库

本仓库只服务一个主目标：设计支持**前台共享与后台执行**的 Android AI 浏览器。

## 当前目标

- 前台使用共享 WebView，用户可以随时观察、暂停和接管。
- 后台使用隐藏 WebView、Virtual Display 或独立浏览器后端继续执行。
- 前后台切换依赖持久化 `BrowserSession` 和执行权转移，不复制页面。
- DOM、可访问性树和结构化浏览器接口优先；视觉 GUI 作为降级通道。
- 每个动作都要有页面变化、后置条件或结果状态验证。
- 高频成功轨迹经过清洗、参数化和回放验证后，才能晋升为 Skill。
- 外部 Chrome/CDP 和扩展是可选适配器，不是 Android Chrome 插件基础依赖。

本仓库是研究、选型和迁移参考，不是当前可直接发布的完整 Android 产品。

## 唯一推荐阅读路径

1. [AI 浏览器总体架构](architecture/ai-browser-overview.md)：明确会话、表面、执行权和动作分层。
2. [参考项目吸收计划](architecture/reference-absorption-plan.md)：说明哪些内容照搬、重写、吸收或删除。
3. [技能契约](specs/skill-contract.md)：定义参数、前置条件、成功判据、失效判据和回退。
4. [决策记录索引](decision-records/README.md)：区分当前有效决策、支撑决策和历史方案。
5. [来源与证据索引](evidence/source-index.yaml)：记录项目来源、许可证和事实边界。

## 参考项目分层

| 层 | 主参考 | 吸收范围 |
|---|---|---|
| 浏览器核心 | [Agentic WebView](https://github.com/shanthropic/agentic-webview) | WebView 工具、DOM/可访问性观察、浏览器动作抽象 |
| 浏览器产品参考 | [ZorvBrowser](https://github.com/Quor-a/ZorvBrowser)、[Operit](https://github.com/Moole123/operit) | WebSession、前后台浏览器能力和 Android 工具边界 |
| 后台执行 | [ClosePaw](https://github.com/imoonkey/closepaw)、[Aries-AI](https://github.com/ZG0704666/Aries-AI) | Virtual Display、Shizuku、生命周期、截图和输入隔离 |
| 轨迹与技能 | [MobiAgent](https://github.com/IPADS-SAI/MobiAgent)、[Mobile-Agent-E](https://github.com/X-PLUG/MobileAgent/tree/main/Mobile-Agent-E)、[KnowAct](https://github.com/HITsz-TMG/KnowAct) | 轨迹检索、去冗余、参数化、验证和技能晋升 |
| 探索与封装 | [DroidAgent](https://github.com/coinse/droidagent)、[Ghost in the Droid](https://github.com/ghost-in-the-droid/android-agent) | 自动探索、脚本生成、技能注册和版本管理 |

其余项目保留在对应目录中，作为局部证据或对比样本，不代表根工程选择。

## 目录边界

- `architecture/`：AI 浏览器架构和参考项目迁移方案。
- `specs/`：跨实现的动作、轨迹、技能和验证契约。
- `decision-records/`：有日期和结论边界的技术决策。
- `references/`：项目定位、阅读入口和源码索引。
- `evidence/`：来源、许可证、版本和事实核验。
- `execution/`、`browser/`、`runtime/`、`memory/`、`skill-generation/`：第三方参考项目，保持上游边界。

## 当前实施顺序

1. 固定 `BrowserSession`、表面和执行权状态机。
2. 以 Agentic WebView 建立 DOM 优先的浏览器动作层。
3. 接入前台共享 WebView 与后台隐藏表面切换。
4. 接入 Virtual Display/CDP 等执行适配器并验证失败回退。
5. 建立轨迹清洗、SkillIR、回放验证和版本晋升闭环。

## 许可证边界

本仓库新增文档采用 MIT。第三方项目保留原许可证；参考不等于允许复制代码，移植前必须单独核对许可证、隐私、权限和后台生命周期约束。
