# 工作模式定义

> [← 返回目录](./README.md)

### 10.1 模式枚举

| 编号 | 模式名 | 中文 | 类型 |
|---|---|---|---|
| 0 | `NormalRecordeMode` | 普通录像 | 录像 |
| 1 | `SlowRecordeMode` | 慢动作 | 录像 |
| 2 | `LoopRecordeMode` | 循环录像 | 录像 |
| 3 | `TimeLapseMode` | 延时摄影 | 录像 |
| 4 | `NormalCaptureMode` | 普通拍照 | 拍照 |
| 5 | `AutoCaptureMode` | 自动拍照 | 拍照 |
| 6 | `ContinueCaptureMode` | 连拍 | 拍照 |
| 7 | `TimingCaptureMode` | 定时拍照 | 拍照 |

### 10.2 模式与菜单的关系

每种模式有独立的设置菜单列表。切换模式后，菜单内容会变化。`GET /api/v1/settings/menus` 返回的 `modeMenus` 会自动随当前模式变化。

### 10.3 文件存储目录

| 模式 | 存储目录 |
|---|---|
| 录像模式（0-3） | `/mnt/DCIM/Normal/` |
| 事件触发 | `/mnt/DCIM/Event/` |
| 停车监控 | `/mnt/DCIM/Parking/` |
| 拍照模式（4-7） | `/mnt/DCIM/Photo/` |

### 10.4 文件命名规范

| 类型 | 格式 | 示例 |
|---|---|---|
| 视频 | `VID_yyyyMMdd_HHmmss.MP4` | `VID_20260522_143000.MP4` |
| 事件视频 | `EVT_yyyyMMdd_HHmmss.MP4` | `EVT_20260522_143000.MP4` |
| 停车视频 | `PKG_yyyyMMdd_HHmmss.MP4` | `PKG_20260522_143000.MP4` |
| 照片 | `IMG_yyyyMMdd_HHmmss.jpg` | `IMG_20260522_143500.jpg` |

---

