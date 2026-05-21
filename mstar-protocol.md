# MStar 协议族

## 导航

### 主入口

- [主入口：app-newCam-release-technical-analysis.md](./app-newCam-release-technical-analysis.md)

### 兄弟文档

- [YZ 协议族](./yz-protocol.md)
- [QZ 协议总览](./qz-protocol-overview.md)
- [QZ API 合同草案](./qz-api-contract.md)
- [QZ 媒体与菜单模型](./qz-media-model.md)
- [QZ 复刻实施计划](./qz-replica-plan.md)

### 本页定位

- 适合 MStar 设备端开发和客户端协议层实现
- 重点是 CGI 接口、属性树、请求模板
- MStar 偏传统 CGI 风格，比 QZ 简单但比 YZ 复杂

## 1. 基础画像

| 项目 | 值 |
|---|---|
| 默认 IP | `192.168.1.1` |
| HTTP Host | `http://192.168.1.1/cgi-bin/Config.cgi?` |
| RTSP 预览 | `rtsp://<ip>/liveRTSP/av` |
| 数据库 | `http://<ip>/tmp/DCF.db` |
| 核心类（Android） | `com.record.MstCS.MSCs` / `com.record.MstCS.MSPr` / `com.record.MstCS.CaseEventManager` |

## 2. 协议特点

- 典型 CGI 参数风格，不是 REST JSON
- 属性树结构（`Camera.Menu.*`、`Net.WIFI_AP.*`）
- 设备媒体索引靠下载 `DCF.db` 再本地解析
- 存在 UDP 端口监听（状态通知/事件回调/心跳）

## 3. 请求模板

```text
GET http://<ip>/cgi-bin/Config.cgi?<action>&k1=v1&k2=v2
```

内部是"命令 + 参数数组"的组合，最终以 query string 拼接。

## 4. 主要能力

`MSCs` 包含以下明确方法：

| 方法 | 说明 |
|---|---|
| `capture()` | 拍照 |
| `record()` | 录像 |
| `lock()` | 锁定视频 |
| `syncTime()` | 时间同步 |
| `set2WiFiName()` | 设置 Wi-Fi 名称 |
| `set2WiFiPwd()` | 设置 Wi-Fi 密码 |
| `getMediaFiles()` | 获取媒体文件列表 |
| `uploadCamFile()` | 上传文件到设备 |
| `getResolution()` | 获取分辨率 |
| `getPrefers()` | 获取偏好设置 |
| `getDeviceStatus()` | 获取设备状态 |
| `getSdInfo()` | 获取 SD 卡信息 |
| `getDianLiang()` | 获取电量 |
| `zoomDv()` | 变焦 |
| `restartDevice()` | 重启设备 |

## 5. 已确认的属性/接口映射

| App 方法 | 命令/属性 |
|---|---|
| `capture()` | `capture` |
| `format()` | `format` |
| `getMenuValues()` | `Camera.Menu.*` |
| `getParameterValue(name)` | `property=<name>` + `get` |
| `getRecordingTime()` | `CurDuration` |
| `getResolution(video)` | `Camera.Menu.VideoRes` / `Camera.Menu.ImageRes` |
| `getSdcardInfo()` | `Camera.Menu.CardInfo.*` |
| `getWifiInfo()` | `Net.WIFI_AP.*` |
| `getWifiName()` | `Net.WIFI_AP.SSID` |
| `getWifiPws()` | `Net.WIFI_AP.CryptoKey` |
| `isRecording()` | `Camera.Preview.MJPEG.status.record` |
| `lock()` | `Video` + `lock` |
| `playback(true/false)` | `Playback` + `enter/exit` |
| `record(on)` | `recordon` / `recordoff` |
| `setDate()` | `TimeSettings` |
| `setParameterValue(name,val)` | `property=<name>&value=<val>` + `set` |
| `setWifiName()` | `Net.WIFI_AP.SSID` |
| `setWifiPassword()` | `Net.WIFI_AP.CryptoKey` |
| `setWorkMode()` | `sysworkmode` |
| `switchCamera()` | `property=Camera.Preview.Source.1.Camid&value=<id>` + `setcamid` |
| `getLiveStreamUrl()` | `rtsp://.../liveRTSP/av` |

## 6. 文件能力

MStar 的媒体索引不靠列表接口，而是：

1. 下载 `http://<ip>/tmp/DCF.db`
2. App 端解析 SQLite 获取媒体数据

同时 `MSPr` 中有 `&format=all&from=` 模式，说明也存在部分文件分页拉取能力。

## 7. 额外说明

- `CaseEventManager` 中存在 UDP 端口监听逻辑
- HTTP 负责控制/查询，UDP 负责状态通知、事件回调或心跳
- MStar 是三套协议里复杂度居中的一套
