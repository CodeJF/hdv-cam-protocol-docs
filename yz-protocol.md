# YZ 协议族

## 导航

### 主入口

- [主入口：app-newCam-release-technical-analysis.md](./app-newCam-release-technical-analysis.md)

### 兄弟文档

- [MStar 协议族](./mstar-protocol.md)
- [QZ 协议总览](./qz-protocol-overview.md)
- [QZ API 合同草案](./qz-api-contract.md)
- [QZ 媒体与菜单模型](./qz-media-model.md)
- [QZ 复刻实施计划](./qz-replica-plan.md)

### 本页定位

- 适合 YZ 设备端开发和客户端协议层实现
- 重点是接口表、请求模板、返回结构、初始化链路
- YZ 是三套协议里最接近标准 REST API 的一套

## 1. 基础画像

| 项目 | 值 |
|---|---|
| 默认 IP | `192.168.169.1` |
| HTTP Host | `http://192.168.169.1/app` |
| RTSP 预览 | 动态获取，通过 `getmediainfo` 返回 |
| 核心类（Android） | `com.record.YzCS.YZCs` / `com.record.YzCS.YZPr` |

## 2. 协议特点

YZ 是三套协议里最"REST/CGI 风格"的一套：

- 基于 `http://<ip>/app/<command>?k=v`
- 路径清晰、参数名可读
- 统一 JSON 响应外壳
- 适合优先抓包和快速复刻

## 3. 统一响应模型

所有接口共用一个外壳：

```json
{
  "result": 0,
  "info": { ... }
}
```

解析规则：

- `result == 0` 表示成功
- `result != 0` 表示失败
- `info` 是 JSON Object，字段拍平为 `Map<String, String>`

对应 TypeScript 模型：

```ts
type YzResponse = {
  result: number
  info?: Record<string, string>
}
```

## 4. 请求模板

```text
GET http://<ip>/app/<command>?k1=v1&k2=v2
```

所有请求都是 GET，参数通过 query string 传递。

## 5. 已确认接口表

| App 方法 | HTTP 请求 |
|---|---|
| `capture()` | `snapshot` |
| `deleteFile(path)` | `deletefile?file=<path>` |
| `format()` | `sdformat` |
| `getBatteryInfo()` | `getbatteryinfo` |
| `getCameraInfo()` | `getcamerainfo` |
| `getCapability()` | `capability` |
| `getDeviceAttr()` | `getdeviceattr` |
| `getFileList(folder,start,end)` | `getfilelist?folder=<folder>&start=<start>&end=<end>` |
| `getGpsFileList()` | `getgpsfilelist` |
| `getLockVideoStatus()` | `getlockvideostatus` |
| `getMenuValues()` | `getparamvalue?param=all` |
| `getParamItems(name)` | `getparamitems?param=<name>` |
| `getParameterValue(name)` | `getparamvalue?param=<name>` |
| `getProductInfo()` | `getproductinfo` |
| `getRecordingTime()` | `getrecduration` |
| `getSdInfo()` | `getsdinfo` |
| `getSupportLiveViewUrl()` | `getmediainfo` |
| `getThumbnailUrl(path)` | `getthumbnail?file=<path>` |
| `getWorkMode()` | `getparamvalue?param=work_mode` |
| `isRecording()` | `getrecduration`，读 `duration` |
| `lock()` | `lockvideo` |
| `playback(enter/exit)` | `playback?param=enter/exit` |
| `record(on/off)` | `setparamvalue?param=rec&value=<0/1>` |
| `reset()` | `reset` |
| `setDate(timestamp)` | `setsystime` + `date=<formatted>` |
| `setParameterValue(name,val)` | `setparamvalue?param=<name>&value=<val>` |
| `setTimezone(offset)` | `settimezone?timezone=<offset>` |
| `setWifiName(ssid)` | `setwifi?wifissid=<ssid>` |
| `setWifiPassword(pwd)` | `setwifi?wifipwd=<pwd>` |
| `setWorkMode(mode)` | `setparamvalue?param=work_mode&value=<mode>` |
| `settingEnter()` | `setting?param=enter` |
| `settingExit()` | `setting?param=exit` |
| `switchCamera(id)` | `setparamvalue?param=switchcam&value=<id>` |
| `wifiReboot()` | `wifireboot` |

## 6. 字段级返回结构

### 6.1 `getsdinfo`

```json
{
  "result": 0,
  "info": {
    "status": "2",
    "total": "127512.0",
    "free": "92341.0"
  }
}
```

`status` 到 App 三态标志的映射：

| 设备 status | App hasCardFlag |
|---|---|
| `0` / `1` / `4` / `10` / `11` / `12` / `13` | `"0"` |
| `3` | `"3"` |
| 其它 | `"1"` |

### 6.2 `getmediainfo`

```json
{
  "result": 0,
  "info": {
    "rtsp": "rtsp://192.168.169.1/liveRTSP/av1"
  }
}
```

`rtsp` 字段是实时预览的关键，App 直接用它构造播放器地址。

### 6.3 `getfilelist`

```json
{
  "result": 0,
  "info": [
    {
      "folder": "loop",
      "files": [
        {
          "name": "/DCIM/100MEDIA/VID_0001.MP4",
          "type": 1,
          "size": 12345,
          "createtime": 1710000000,
          "duration": 60
        }
      ]
    }
  ]
}
```

字段语义：

| 字段 | 类型 | 说明 |
|---|---|---|
| `folder` | String | 分组名：`loop` / `event` / `emr` / `park` |
| `name` | String | 设备端完整路径 |
| `type` | int | `1` = 图片，其它 = 视频 |
| `size` | long | 单位大概率是 KB，App 会乘 1024 |
| `createtime` | long | 单位秒，App 会乘 1000 |
| `duration` | int | 视频时长 |

消费规则：

- `name` 为空时忽略
- `folder == "emr"` 时标记为紧急视频
- 最终列表倒序排序

### 6.4 媒体分类映射

| 上层 tag | 内部 mediaType | 请求 folder |
|---|---|---|
| `event` | `1` | `event` |
| `loop` | `2` | `loop` |
| `emr` | `3` | `emr` |
| `park` | `4` | `park` |
| 默认 | `2` | 空（全量） |

分页规则：每页 30 条，`start = page * 30`，`end = start + 29`。

## 7. 初始化链路

`YZCs.connectTo()` 的启动顺序：

1. `getProductInfo()` - 基础设备信息
2. `getSupportLiveViewUrl()` - 获取 RTSP 地址
3. `getSdInfo()` - SD 卡状态
4. `getParamItems()` - 全部参数项和参数值

第 2 步返回值直接写入 `liveStreamUrl`。

## 8. 面向开发的请求示例

```bash
# 读取所有参数
curl "http://192.168.169.1/app/getparamvalue?param=all"

# 读取单个参数
curl "http://192.168.169.1/app/getparamvalue?param=work_mode"

# 读取参数候选项
curl "http://192.168.169.1/app/getparamitems?param=rec_resolution"

# 设置参数
curl "http://192.168.169.1/app/setparamvalue?param=work_mode&value=0"

# 开始录像
curl "http://192.168.169.1/app/setparamvalue?param=rec&value=1"

# 停止录像
curl "http://192.168.169.1/app/setparamvalue?param=rec&value=0"

# 拍照
curl "http://192.168.169.1/app/snapshot"

# 进入回放
curl "http://192.168.169.1/app/playback?param=enter"

# 退出回放
curl "http://192.168.169.1/app/playback?param=exit"

# 获取文件列表
curl "http://192.168.169.1/app/getfilelist?folder=/DCIM&start=0&end=29"

# 删除文件
curl "http://192.168.169.1/app/deletefile?file=/DCIM/100MEDIA/VID_0001.MP4"

# 获取缩略图
curl "http://192.168.169.1/app/getthumbnail?file=/DCIM/100MEDIA/IMG_0001.JPG"

# 设置 Wi-Fi
curl "http://192.168.169.1/app/setwifi?wifissid=HDV_CAM_NEW"
curl "http://192.168.169.1/app/setwifi?wifipwd=12345678"

# 切换镜头
curl "http://192.168.169.1/app/setparamvalue?param=switchcam&value=1"

# SD 卡信息
curl "http://192.168.169.1/app/getsdinfo"

# RTSP 地址
curl "http://192.168.169.1/app/getmediainfo"
```

## 9. 推荐客户端抽象

```ts
interface YzCameraClient {
  getAllParams(): Promise<Record<string, string>>
  getParam(name: string): Promise<Record<string, string>>
  setParam(name: string, value: string): Promise<boolean>
  getFileList(folder: string, start: number, end: number): Promise<YzFileList>
  deleteFile(path: string): Promise<boolean>
  getThumbnailUrl(path: string): string
  getLiveRtspUrl(): Promise<string>
  startRecord(): Promise<boolean>
  stopRecord(): Promise<boolean>
  capture(): Promise<boolean>
}
```

## 10. 开发接入顺序

1. `getparamvalue?param=all` - 确认设备在线、响应格式正确
2. `getmediainfo` - 确认 RTSP 能力
3. `getfilelist` - 验证媒体目录访问
4. `setparamvalue` - 先试无风险项如 `work_mode`
5. `setparamvalue?param=rec&value=1/0` - 录像控制
6. `setwifi` - 最后改 Wi-Fi，避免把设备连丢
