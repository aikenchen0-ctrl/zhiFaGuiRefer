# AI 浏览器总体架构

本文是本仓库的当前架构入口。它只描述 Android AI 浏览器，不把全域手机助手、远程电脑和个人资产检索列为第一阶段目标。

## 1. 目标状态

系统围绕一个持久化 `BrowserSession` 工作。会话保存页面、Cookie、标签、任务上下文、动作历史和当前执行权；前台与后台只是不同执行表面，不是两个页面副本。

```text
用户/模型任务
      |
任务控制器 -> BrowserSession -> 执行权协调器
                              |               |
                        前台共享表面       后台执行表面
                        WebView             隐藏 WebView / Virtual Display / CDP
                              \               /
                               观察、动作、验证
                                      |
                              轨迹与 Skill 编译
```

## 2. 执行表面

| 表面 | 作用 | 进入条件 | 退出条件 |
|---|---|---|---|
| `ForegroundWebView` | 用户可见、可观察、可接管 | 用户打开会话或请求接管 | 用户切后台、锁屏或主动让出执行权 |
| `HiddenWebView` | 同一会话的后台网页执行 | 页面需要 DOM/JS 操作且无需系统级输入 | 页面需要真实输入、渲染或后台限制触发 |
| `VirtualDisplay` | 隔离渲染、截图和定向输入 | 页面或目标流程不能由 DOM 完成 | 任务结束、权限失效或资源回收 |
| `ExternalBrowser` | Chrome/CDP 或独立浏览器后端 | 用户明确授权或 WebView 能力不足 | 连接断开、权限撤回或任务完成 |

表面切换只转移执行权和会话状态，不复制 DOM、Cookie 或页面截图。切换必须产生可审计事件，并在新表面重新观察后继续执行。

## 3. 执行权状态机

```text
Idle
  -> ForegroundOwned
  -> BackgroundOwned
  -> HandoffPending
  -> ForegroundOwned / BackgroundOwned
  -> Paused / Failed / Completed
```

约束：

1. 同一 `BrowserSession` 同时只有一个动作执行者。
2. 用户接管拥有最高优先级，后台动作必须在动作边界停止。
3. 执行权转移前写入检查点，转移后重新获取页面快照。
4. 任何表面失效都进入 `Paused` 或 `Failed`，不能静默换表面继续点击。

## 4. 观察和动作优先级

观察通道按可靠性排序：

1. DOM、可访问性树和结构化浏览器接口。
2. 页面脚本、网络状态和结构化结果。
3. WebView 截图或浏览器调试协议。
4. Virtual Display 截图与视觉模型。
5. 坐标点击和滑动只作为最后降级通道。

动作必须使用统一 `Action IR`，至少包含动作类型、目标定位、参数、前置条件、后置条件、超时、重试策略和证据引用。执行器不得直接依赖模型输出的坐标或自然语言。

## 5. 验证闭环

每个动作都经过以下闭环：

```text
观察 -> 选择动作 -> 执行 -> 页面变化检查 -> 后置条件检查 -> 记录证据
                         | 失败
                         v
                 重观察 / 回退 / 请求接管
```

成功不能只由“动作调用返回成功”决定。至少需要一个页面变化、结构化状态、目标元素状态或业务结果证据。

## 6. 轨迹到 Skill

```text
Episode
  -> 去除等待、重复点击和无效观察
  -> 抽取参数、前置条件和成功判据
  -> 生成 SkillIR
  -> 在相同会话和新会话中回放
  -> candidate -> validated -> promoted
```

Skill 必须包含：适用站点或页面特征、输入参数、前置条件、动作步骤、成功判据、失效判据、失败回退、权限要求、版本和证据。未经重复回放验证的轨迹只能作为候选，不得自动覆盖推理路径。

## 7. 第一阶段边界

必须实现：

- 单会话前台共享 WebView。
- 前台与后台隐藏 WebView 的状态保持和执行权转移。
- DOM/结构化动作与视觉降级动作的统一接口。
- 页面变化、后置条件和失败回退。
- 轨迹记录、最小去冗余和 Skill 候选验证。

暂不作为基础依赖：

- Android Chrome 插件。
- 全域手机应用自动化。
- 远程电脑控制。
- 复杂长期记忆和训练闭环。
- 对所有厂商 ROM 的无差别兼容。

## 8. 参考项目边界

- `Agentic WebView`：浏览器核心和 DOM/工具抽象主参考。
- `ZorvBrowser`、`Operit`：浏览器会话、Android 工具和产品形态参考。
- `ClosePaw`、`Aries-AI`：后台表面、Virtual Display、Shizuku 和生命周期参考。
- `MobiAgent`、`Mobile-Agent-E`、`KnowAct`：轨迹、状态契约和 Skill 编译参考。
- `DroidAgent`、`Ghost in the Droid`：探索、脚本化和技能封装参考。

具体取舍见[参考项目吸收计划](reference-absorption-plan.md)，不要从各项目 README 的功能列表直接推导根工程。
