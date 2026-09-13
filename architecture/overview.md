# 通用 GUI Agent 执行链

本页补充 Android GUI 执行背景。当前产品架构、浏览器会话和前后台执行表面以 [`ai-browser-overview.md`](ai-browser-overview.md) 为准。

## 能力层

- **控制面**：任务、会话、模型路由、执行权、审计和 Skill 生命周期；
- **观察层**：DOM、可访问性树、页面结构、截图和设备状态；
- **执行层**：WebView 动作、无障碍、ClosePaw 虚拟屏、Shizuku、输入、截图和远程设备；
- **技能层**：Episode、SkillIR、验证、晋升、失败修订、版本和回放；
- **评测层**：页面结果、动作成功率、Skill 回放、ROM/API、延迟和成本。

## 统一执行链

```text
用户任务
  -> BrowserSession 与 Skill/Memory 检索
  -> 前置条件、执行权和权限检查
  -> 浏览器或设备后端执行动作
  -> 页面/屏幕/文件证据观察
  -> 结果验证
  -> 成功返回，或记录 Episode 并重新规划
```

## 统一动作边界

Aries-AI 的输入注入、Ruto-GLM 的 display 定向动作、MobiAgent 的动作缓存、DroidAgent 的 UIAutomator2 脚本和 Ghost 的 Action/Workflow，均先转换为统一 `Action`/`SkillStep`，再交给设备执行器。

```text
截图 + UI 树 -> StateSnapshot
StateSnapshot + Action -> EpisodeStep
EpisodeStep 序列 -> SkillIR 候选
SkillIR -> 设备验证 + 结果验证 + 重复回放
验证通过 -> SkillPackage；失败 -> 诊断、修订或回退
```

## 依赖方向

```text
ControlPlane -> Contracts -> CapabilityRouter
                              -> DeviceAdapter
                              -> AssetIndexAdapter
                              -> SkillAdapter
                              -> ModelProvider
```

核心合同不依赖 Android、Shizuku、数据库、模型 SDK 或第三方项目；ROM 特判、隐藏 API 和资源释放只能存在于设备适配器。

