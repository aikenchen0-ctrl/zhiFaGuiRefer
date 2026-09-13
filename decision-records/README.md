# 决策记录索引

决策记录按“当前有效、支撑材料、历史材料”组织。文件编号不代表严格时间顺序；`004` 和 `005` 已统一用于当前专题决策。

## 当前有效

当前产品是 Android AI 浏览器，架构入口在 `architecture/`；本目录只记录会影响根工程、权限、数据持久化或参考项目取舍的决策。

| 文档 | 状态 | 用途 |
|---|---|---|
| [Android AI 浏览器根工程与参考模块吸收计划](../architecture/reference-absorption-plan.md) | 当前 | 产品根工程、模块分类、删减项和实施顺序 |
| [Android 资产权限矩阵](003-android-asset-permission-matrix.md) | 支撑 | 手机文件、相册、SAF、其它 App 文件和后台索引边界 |
| [前沿记忆系统选型](004-frontier-memory-system-selection.md) | 支撑 | 浏览器历史、轨迹和长期记忆的分层方案 |
| [Android 数据库架构选型](005-android-database-architecture-selection.md) | 支撑 | 端侧会话、轨迹和索引持久化方案 |

## 历史与扩展材料

| 文档 | 状态 | 说明 |
|---|---|---|
| [001 主工程与参考模块](001-main-project-and-reference-modules.md) | 历史 | 面向全域 Android 助手，保留资产系统和模块源码推演 |
| [002 源码复核与集成清单](002-source-review-and-final-integration-list.md) | 历史 | 保留全域助手双核心方案和源码证据；当前浏览器目标以架构文档为准 |

## 阅读规则

1. 先读 [AI 浏览器总体架构](../architecture/ai-browser-overview.md)，确定第一阶段边界；
2. 再读 [参考项目吸收计划](../architecture/reference-absorption-plan.md)，确定哪些内容可迁移；
3. 涉及权限、数据库或记忆时，分别阅读对应专题，不把它们合并成一个大模块；
4. 需要源码定位时回到 [源码证据索引](../evidence/source-index.yaml)，不要只依据项目 README；
5. 新增决策前先确认是否真的改变根工程、权限、数据模型、依赖边界或实施顺序。
