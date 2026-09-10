# 手机个人文件与图片索引的 Android 权限矩阵

状态：整合前准备文档；只定义权限、授权范围和失效处理，不启动实现。

## 结论

个人文件、图片向量化和检索至少会遇到五类权限边界：媒体库读取、文件/目录选择、广泛文件访问、后台执行，以及向云端发送内容。它们不是同一种权限，不能用一个“存储权限已授权”的布尔值代替。

推荐默认路线：`Photo Picker/SAF` 获取用户明确选择的内容；需要全库相册索引时再请求分级媒体读取权限；只有“设备文件搜索”确实是产品核心且无法用 SAF/MediaStore 满足时，才提供 `MANAGE_EXTERNAL_STORAGE` 专项模式。

## 一、按资产类型的权限矩阵

| 资产和操作 | 推荐 API | 可能需要的权限 | 关键限制 |
|---|---|---|---|
| 用户临时选择图片/视频 | Android Photo Picker | 不需要媒体运行时权限 | 默认 URI 授权是临时的；长期索引要尝试持久化 URI，并处理撤销、移动、删除和最多 5000 个媒体授权 |
| 全图库图片/视频索引 | `MediaStore` | API 32 及以下通常为 `READ_EXTERNAL_STORAGE`；API 33+ 为 `READ_MEDIA_IMAGES`、`READ_MEDIA_VIDEO`；API 34+ 自建选择界面还要处理 `READ_MEDIA_VISUAL_USER_SELECTED` | 用户可能只授予部分照片；每次查询和恢复前台都要重新检查，不能缓存“全库已授权” |
| 音频、录音、音乐封面 | `MediaStore.Audio` | API 33+ `READ_MEDIA_AUDIO`；旧版本按 `READ_EXTERNAL_STORAGE` | 音频读取和麦克风录音是两回事；录音采集另需 `RECORD_AUDIO` |
| 用户选择单个文件 | `ACTION_OPEN_DOCUMENT` | 不需要存储运行时权限 | 通过返回的 `content://` URI 读取；跨重启要调用 `takePersistableUriPermission` |
| 用户选择目录及子文件 | `ACTION_OPEN_DOCUMENT_TREE` | 不需要存储运行时权限 | 只能访问用户选定目录；Android 11+ 不能选择内部存储根、可靠 SD 根、Download、`Android/data`、`Android/obb` |
| 应用自己创建的索引、缩略图和缓存 | `filesDir`、`getExternalFilesDir`、应用专属 MediaStore | 不需要存储权限 | 应用卸载后应用专属目录会被删除；敏感向量和 OCR 应优先放内部存储 |
| 其他应用创建的 Download 文件 | SAF | 通常不应只依赖媒体权限 | 对 `MediaStore.Downloads` 中非本应用创建的文件，优先使用 SAF |
| 设备共享存储的全文件扫描 | 直接路径、`MediaStore.Files` | `MANAGE_EXTERNAL_STORAGE` 特殊应用访问 | 需要跳转系统设置，不是普通运行时权限；仍不能访问其他应用的 `Android/data` 专属目录；应用商店审核和用户信任成本高 |
| 未脱敏照片 EXIF 地理位置 | `MediaStore` + EXIF | API 29+ `ACCESS_MEDIA_LOCATION`，并需运行时授权 | GPS 属于高敏感数据；默认只读取经过脱敏的位置信息或不读取位置 |
| 向量化期间保持进程运行 | WorkManager，必要时用户发起的前台任务 | 长任务需前台服务通知；API 34+ 声明对应 FGS 类型和权限；API 33+ 通知显示另需 `POST_NOTIFICATIONS` | Android 12+ 限制后台启动前台服务；Android 15+ `dataSync` 前台服务 24 小时内累计最多 6 小时 |

## 二、权限不是能力本身

以下能力与文件读取权限分开治理：

| 能力 | 典型授权 | 与资产索引的关系 |
|---|---|---|
| 虚拟屏截图、GUI 观察 | Accessibility Service；若使用 MediaProjection 还需用户投屏授权和对应前台服务类型 | 只在把 GUI 截图纳入个人资产或轨迹索引时启用，不因文件搜索默认申请 |
| Shizuku、Root、隐藏 API | 用户在 Shizuku/Root 工具中单独授权 | 不能替代 MediaStore/SAF 授权，也不能承诺可读其他应用私有目录 |
| 悬浮窗 | `SYSTEM_ALERT_WINDOW` 特殊应用访问 | 只用于进度、控制和人工确认 UI，不提供文件读取能力 |
| 长期后台扫描 | WorkManager、前台服务、设备厂商自启动/电池策略 | 这是调度和存活问题，不是扩大文件授权的理由 |
| 云端 Embedding、OCR 或 VLM | `INTERNET` 加用户数据外传同意 | `INTERNET` 本身不会弹权限框；必须单独显示上传范围、模型、费用、保留期和失败降级策略 |

## 三、Android 版本和授权状态

### API 28 及以下

读取其他应用创建的媒体通常需要 `READ_EXTERNAL_STORAGE`；修改媒体还可能需要 `WRITE_EXTERNAL_STORAGE`。端侧模型和索引文件应使用应用专属目录，避免依赖旧版广泛路径。

### API 29-32

Scoped Storage 默认生效。应用自己的 MediaStore 内容和应用专属目录不需要广泛存储授权；读取其他应用媒体通常需要 `READ_EXTERNAL_STORAGE`。`WRITE_EXTERNAL_STORAGE` 在 Android 11 及以上不再增加额外访问能力。

### API 33

图片、视频、音频拆成 `READ_MEDIA_IMAGES`、`READ_MEDIA_VIDEO`、`READ_MEDIA_AUDIO`。只需要用户一次性选择少量图片时，优先 Photo Picker，不要为了单次导入申请全库权限。

### API 34 及以上

自建图库界面必须处理 `READ_MEDIA_VISUAL_USER_SELECTED`。即使 `READ_MEDIA_IMAGES` 或 `READ_MEDIA_VIDEO` 返回已授予，实际也可能只是用户选择的部分媒体。索引器必须：

1. 在每次扫描前通过 `checkSelfPermission` 重新读取当前授权；
2. 在 `onResume`、设置返回和用户重新选择后刷新 MediaStore；
3. 给每条索引记录保存 `access_scope=full|partial|uri|app_owned`；
4. `partial` 扫描不得把“当前不可见”直接当成“用户已删除”；
5. 打开原始 URI 失败时标记为 `inaccessible`，等待重新授权或重新选择，而不是删除证据。

### API 35 及以上

如果用 `dataSync` 前台服务做长时间全库向量化，要实现超时停止和断点续跑；不要假设一个前台服务可以无限运行。大批量、用户主动触发的传输或处理应评估 WorkManager、用户发起数据传输任务或分段任务。

## 四、参考项目源码中的权限问题

| 项目和源码 | 已观察到的做法 | 整合时的结论 |
|---|---|---|
| `local-photo-search` `AndroidManifest.xml`、`MediaStoreScanner.kt` | 声明 `READ_EXTERNAL_STORAGE(maxSdk=32)`、`READ_MEDIA_IMAGES`、`READ_MEDIA_VISUAL_USER_SELECTED`；`hasImagePermission()` 将部分授权也视为可扫描，但没有返回 `full/partial` 状态 | 可复用分级权限和扫描入口；必须增加授权范围状态。其 `syncScanned()` 不能在 partial 结果下删除所有未出现记录 |
| `Operit` `AndroidManifest.xml` | 同时声明 `READ_EXTERNAL_STORAGE`、`WRITE_EXTERNAL_STORAGE`、`READ_MEDIA_AUDIO`、`MANAGE_EXTERNAL_STORAGE`、悬浮窗、前台服务、相机、录音、短信、位置等大量权限 | 不能整体复制 Manifest；按文件、媒体、OCR、GUI、语音、远程能力拆分按需声明和申请 |
| `Zafiro` `AndroidManifest.xml` | 声明图片/视频/部分媒体、`MANAGE_EXTERNAL_STORAGE`、无障碍和前台服务 | 可参考能力分组；不能把 `MANAGE_EXTERNAL_STORAGE` 当成普通索引前置条件 |
| `X-OmniClaw` `AlbumScanner.kt` | 以 MediaStore 增量扫描相册并保存 URI/元数据 | 适合与 partial 授权结合，但扫描游标必须与授权范围和可见性版本绑定 |
| `PocketSearch` `IndexService` | 使用平台媒体选择/相册 API，先同步 live IDs，再批量编码并清理陈旧记录 | 可吸收陈旧数据清理，但必须区分“用户删除”和“当前没有权限看到” |

## 五、最小权限方案

### 模式 A：用户选定内容

- 图片/视频：Photo Picker；
- 文档、代码、压缩包：`ACTION_OPEN_DOCUMENT` 或 `ACTION_OPEN_DOCUMENT_TREE`；
- 持久化 URI 授权；
- 索引表只处理已授权 URI；
- 不声明 `MANAGE_EXTERNAL_STORAGE`。

适合首次导入、隐私优先和应用商店分发，缺点是不能自动发现用户未选择的内容。

### 模式 B：相册全库索引

- API 32 及以下：按版本申请 `READ_EXTERNAL_STORAGE`；
- API 33+：按媒体类型申请 `READ_MEDIA_IMAGES/VIDEO/AUDIO`；
- API 34+：声明并处理 `READ_MEDIA_VISUAL_USER_SELECTED`；
- 每次扫描重新检查授权，并区分 full/partial；
- 只有确认 full 且用户启用“删除同步”时，才允许将未出现项判断为陈旧。

适合“找我的照片/视频/录音”，但需要清晰的授权解释、重新选择入口和索引可见性治理。

### 模式 C：设备文件搜索

- 先用 MediaStore + SAF 覆盖公共媒体和用户选择目录；
- 只有核心功能确实要求跨目录自动扫描时，才申请 `MANAGE_EXTERNAL_STORAGE`；
- 申请后仍需处理 `Android/data` 等不可访问目录、厂商限制和系统设置撤销；
- 将该模式做成显式高级开关，默认关闭。

适合文件管理器、备份、杀毒或明确的“设备文件搜索”产品，不适合作为普通聊天助手的默认权限。

## 六、索引数据合同必须记录授权状态

```text
AssetRecord
  id / sourceUri / mimeType / modifiedAt / contentHash
  accessScope: app_owned | uri | partial | full | inaccessible
  permissionSnapshot: media/photo-picker/tree grant identifiers
  visibleAt / lastOpenedAt / revokedAt
  parserVersion / embeddingModel / embeddingDimension
```

规则：

1. 权限状态不能只存一个永久布尔值；每次使用前重新向系统查询。
2. 向量可以保留用于恢复，但原始 URI 失效后必须禁止打开、分享和上传。
3. 用户撤回权限时，默认停止读取并标记不可访问；是否删除向量由用户设置决定。
4. partial 结果只允许增量更新可见集合，不允许执行全量删除同步。
5. 云端处理只上传用户明确允许的资产或派生内容，并记录模型、时间和撤回状态。

## 七、实施前验收清单

- [ ] API 28、32、33、34、35 至少覆盖一次授权和撤销测试；
- [ ] 全库、部分媒体、单 URI、目录 URI、权限拒绝和权限被系统重置均有状态；
- [ ] 用户删除、文件移动、URI 撤销和暂时不可见可以区分；
- [ ] OCR、Embedding、缩略图读取失败不会改变授权状态；
- [ ] 后台索引支持暂停、取消、断点、重试和通知；
- [ ] `MANAGE_EXTERNAL_STORAGE`、悬浮窗、无障碍、Shizuku 和云端上传均为独立能力开关；
- [ ] 检索结果始终返回真实 URI 和当前访问状态，不返回无法打开的“幽灵结果”。

## 官方依据

- [Android 14 部分照片和视频访问](https://developer.android.com/about/versions/14/changes/partial-photo-video-access)
- [Android 共享媒体文件](https://developer.android.com/training/data-storage/shared/media)
- [Storage Access Framework 文件和目录](https://developer.android.com/training/data-storage/shared/documents-files)
- [Photo Picker](https://developer.android.com/training/data-storage/shared/photo-picker)
- [所有文件访问](https://developer.android.com/training/data-storage/manage-all-files)
- [前台服务类型](https://developer.android.com/about/versions/14/changes/fgs-types-required)
- [Android 15 前台 dataSync 超时](https://developer.android.com/about/versions/15/behavior-changes-15)
- [后台任务和长任务](https://developer.android.com/develop/background-work/background-tasks/data-transfer-options)
