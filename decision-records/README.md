# 决策记录索引

决策记录按编号记录研究结论，但不是平行的产品入口。当前产品架构以 [`architecture/ai-browser-overview.md`](../architecture/ai-browser-overview.md) 为准。

## 当前仍有支撑价值

| 文档 | 状态 | 用途 |
|---|---|---|
| [003 Android 资产权限矩阵](003-android-asset-permission-matrix.md) | 支撑 | 只有涉及手机文件、相册或跨应用内容时阅读 |
| [004 前沿记忆系统选型](004-frontier-memory-system-selection.md) | 支撑 | 规划浏览器历史、轨迹和长期记忆时阅读 |
| [005 Android 数据库架构选型](005-android-database-architecture-selection.md) | 支撑 | 规划端侧会话、轨迹和索引持久化时阅读 |

## 历史方案

| 文档 | 状态 | 说明 |
|---|---|---|
| [001 主工程与参考模块](001-main-project-and-reference-modules.md) | 历史 | 面向全域 Android 助手，不能直接作为 AI 浏览器根工程结论 |
| [002 源码复核与集成清单](002-source-review-and-final-integration-list.md) | 历史 | 保留源码证据和迁移记录，部分根工程结论已被 AI 浏览器目标取代 |

## 决策规则

1. 新功能先检查 AI 浏览器总体架构是否允许。
2. 与浏览器会话、执行权和动作验证有关的结论，优先写入 `architecture/` 或 `specs/`。
3. 只有影响根工程、权限、数据持久化或参考项目取舍的结论才新增决策记录。
4. 旧决策不删除，用状态标明是否仍然有效，避免多个 README 互相覆盖。
