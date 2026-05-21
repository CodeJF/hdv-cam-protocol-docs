# QZ 媒体与菜单模型

## 导航

### 主入口

- [主入口：app-newCam-release-technical-analysis.md](./app-newCam-release-technical-analysis.md)

### 兄弟文档

- [QZ 协议总览](./qz-protocol-overview.md)
- [QZ API 合同草案](./qz-api-contract.md)
- [QZ 复刻实施计划](./qz-replica-plan.md)

### 本页定位

- 适合相册、设置页、模型层实现
- 重点是文件列表、菜单 XML、翻译资源、当前值映射
- 不展开排期和团队协作

## 1. 文件链路概览

QZ 相册有两条数据来源：

1. 目录接口
- `action=dir`
- 直接返回 XML 格式媒体列表

2. 数据库接口
- 下载 `sunxi.db`（完整路径：`/tmp/data/.data/sqlite/sunxi.db`）
- 本地保存为 `DCF.db`
- 走数据库解析链

要高兼容复刻，最好两条都支持。

## 2. 媒体目录模型

目录接口当前可确认字段：

| 标签/属性 | 含义 |
|---|---|
| `<name>` | 文件完整路径 |
| `<size>` | 文件大小 |
| `<time>` | 拍摄/录制时间 |
| `<format time="...">` | 时长秒数 |

兼容示例：

```xml
<list>
  <file>
    <name>/mnt/DCIM/Normal/VID_0001.MP4</name>
    <size>12345678</size>
    <time>2026-05-20 17:30:00</time>
    <format time="60" />
  </file>
</list>
```

## 3. 媒体类型映射

| 目录类型 | property |
|---|---|
| 图片 | `Photo` |
| 普通视频 | `Normal` |
| 事件视频 | `Event` |
| 停车视频 | `Parking` |

扩展名识别：

- `jpg` -> 图片
- `mp4` / `mov` / `ts` -> 视频

建议：

- 原始路径统一返回 `/mnt/...`
- 缩略图路径按 `/thumb/...` 可访问

## 4. SD 卡与状态模型

### 4.1 SD 卡

```ts
type QzSdInfo = {
  hasCard: string
  total: number
  free: number
}
```

字段来源：

- `disk_status`
- `capacity`
- `free_space`

### 4.2 录像状态

```ts
type QzRecordStatus = {
  RecodStatus: string
}
```

约定：`"1"` 表示正在录像。

### 4.3 当前模式

```ts
type QzWorkMode = {
  curworkmodename: string
}
```

### 4.4 电池信息

```ts
type QzBatteryInfo = {
  // 字段待联调确认，命令码 0x7d9
}
```

## 5. 工作模式枚举

QZ 定义了 8 种工作模式，每种关联独立的 setting_keys 和分辨率键名：

| 模式 | 编号 | setting_keys | 分辨率键名 |
|---|---|---|---|
| NormalRecordeMode | 0 | `record_normal_setting_keys` | `Camera.Menu.NRecRes` |
| SlowRecordeMode | 1 | `record_slow_setting_keys` | `Camera.Menu.SRecRes` |
| LoopRecordeMode | 2 | `record_loop_setting_keys` | `Camera.Menu.LRecRes` |
| TimeLapseMode | 3 | `record_timelapse_setting_keys` | `Camera.Menu.TRecRes` |
| NormalCaptureMode | 4 | `photo_normal_setting_keys` | `Camera.Menu.NPhotoRes` |
| AutoCaptureMode | 5 | `photo_auto_setting_keys` | `Camera.Menu.APhotoRes` |
| ContinueCaptureMode | 6 | `photo_continue_setting_keys` | `Camera.Menu.CPhotoRes` |
| TimingCaptureMode | 7 | `photo_time_setting_keys` | `Camera.Menu.TPhotoRes` |

切换模式时，App 会用新模式对应的 `setting_keys` 数组名重新拉取菜单项。

## 6. 菜单定义模型

```ts
type QZMenu = {
  id: string
  index: number
  title: string
  sysKeys: boolean
  items: QZMenuItem[]
}

type QZMenuItem = {
  id: string
  title: string
  index: number
}
```

菜单不是简单 key/value，而是：

1. 菜单 ID 列表（来自 XML）
2. 菜单枚举值列表（来自翻译资源）
3. 当前值索引（来自 0x7d2/0x7d6）
4. 翻译资源（来自 zh-XX.xml）

## 7. 菜单 XML 结构

最小兼容结构：

```xml
<resources>
  <string-array name="video_setting_keys">
    <item>rec_resolution</item>
    <item>loop_record</item>
    <item>exposure</item>
  </string-array>

  <string-array name="system_setting_keys">
    <item>wifi_ssid</item>
    <item>wifi_password</item>
    <item>date_time</item>
  </string-array>

  <string-array name="record_normal_setting_keys">
    <item>rec_resolution</item>
    <item>loop_record</item>
  </string-array>

  <string-array name="photo_normal_setting_keys">
    <item>photo_resolution</item>
    <item>photo_quality</item>
  </string-array>
</resources>
```

规则：

- `item` 文本就是菜单 `id`
- 顺序就是菜单 `index`
- `system_setting_keys` -> `sysKeys = true`
- 每种工作模式对应一个 `<模式>_setting_keys` 数组

## 8. 翻译资源模型

```ts
type QZTranslation = {
  str: string
  value: string
  items: Array<{
    id: string
    title: string
  }>
}
```

两类关键项：

1. 菜单标题

```json
{
  "str": "rec_resolution",
  "value": "Resolution",
  "items": []
}
```

2. 枚举值列表

```json
{
  "str": "rec_resolution_array",
  "value": "",
  "items": [
    { "id": "1080p30", "title": "1080P 30FPS" },
    { "id": "4k30", "title": "4K 30FPS" }
  ]
}
```

## 9. 菜单当前值模型

```ts
type MenuCurrValue = {
  deviceId: number
  deviceName: string
  software: string
  info: Array<{
    index: number
    value: number
  }>
}
```

映射规则：

- `QZMenu.index == info[].index`
- `info[].value` 是当前枚举值索引

命令分组：

- `0x7d2`：非系统菜单当前值
- `0x7d6`：系统菜单当前值

## 10. 设备端推荐内部抽象

```ts
type QzMenuDef = {
  id: string
  index: number
  title: string
  sysKeys: boolean
  items: Array<{ id: string; title: string; index: number }>
  currentValueIndex: number
}
```

用这一份内部模型，可以一次性生成：

- `setting_keys.xml`
- 翻译资源
- `MenuCurrValue`

## 11. 开发提示

- 菜单、翻译、当前值三者必须同源，否则设置页会错位
- 媒体路径和缩略图路径最好统一规范，不要混多套前缀
- 如果暂时不实现 `sunxi.db`，先保证 `action=dir` 路径可用
- 切换工作模式后需要用对应模式的 setting_keys 重新加载菜单
- `sunxi.db` 完整路径为 `/tmp/data/.data/sqlite/sunxi.db`，不是 `/tmp/sunxi.db`
