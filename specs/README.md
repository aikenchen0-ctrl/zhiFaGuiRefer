# 规范索引

规范文档只定义跨项目稳定合同，不绑定某个第三方实现。

## 当前规范

- [Android 操作技能契约](skill-contract.md)：Skill 标识、版本、参数、前置条件、定位、动作、成功判据、失效判据和回退。
- 浏览器第一阶段还需要 `BrowserSession`、`ExecutionSurface`、`ExecutionLease` 和 `ActionIR` 合同，具体边界见 [`../architecture/ai-browser-overview.md`](../architecture/ai-browser-overview.md)。

## 待补规范

- `Task / Session / Run / Step`；
- `DeviceBackend / FrameSource / InputTransport`；
- `Asset / AssetPart / Evidence / Embedding`；
- `Episode / SkillIR / VerificationResult`；
- `TaskCommand / TaskEvent / AssetQuery / AssetResult` 跨 ClawGUI 与 Operit 合同。

这些规范在开始产品整合前冻结，第三方项目只能实现规范，不能反向修改核心领域对象。
