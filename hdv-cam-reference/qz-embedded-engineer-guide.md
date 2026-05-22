# QZ 设备端实现指南（嵌入式工程师版）

## 1. 文档目的

本文面向嵌入式软件工程师，说明如何在设备端实现一套兼容 HDV CAM App 的 QZ 协议栈。App 通过 **IP 地址** 识别设备类型，通过 **HTTP + RTSP + UDP** 三条链路控制设备。本文档所有接口均来自 APK/IPA 逆向分析，是 App 侧的真实调用行为。

---

## 2. 网络拓扑与基础要求

### 2.1 热点与网关

设备必须以 AP 模式工作，手机作为 STA 接入：

| 项目 | 值 |
|---|---|
| 设备 AP 网关 | `192.168.10.1` |
| DHCP 范围 | `192.168.10.100 ~ 192.168.10.200`（建议） |
| SSID 前缀 | 无强制要求，App 通过网关 IP 识别 |

App 的 `CSManager` 路由逻辑：
- `192.168.10.1` → 走 QZ 协议
- `192.168.1.1` → 走 MStar 协议
- `192.168.169.1` → 走 YZ 协议

**设备 IP 必须是 `192.168.10.1`，不可更改。** App 硬编码了此地址。

### 2.2 需要开放的服务端口

| 服务 | 端口 | 协议 | 用途 |
|---|---|---|---|
| HTTP API | `8082` | TCP | 命令控制、状态读取、资源文件 |
| RTSP 实时流 | `8554` | TCP | 实时预览 |
| 事件推送 | `9999`（0x270F） | TCP Socket | 设备→App 事件通知、保活 |
| UDP 辅助 | `49142`（0xBFF6） | UDP | 状态广播、辅助控制 |

---

## 3. HTTP API 服务（端口 8082）

### 3.1 URL 结构

所有 HTTP 请求基于 `http://192.168.10.1:8082`，分为四类：

#### 读状态（GET）
```
GET /api/getdeviceinfo/?custom=1&cmd=<cmd>
```

#### 写状态（POST）
```
POST /api/setdeviceinfo/?custom=1&cmd=<cmd>&par=<value>
POST /api/setdeviceinfo/?custom=1&cmd=<cmd>&str=<string>
```

#### 动作/目录（GET）
```
GET /api/?action=<action>&key1=val1&key2=val2
```

#### 静态资源（GET）
```
GET /usr/share/minigui/res/lang/setting_keys.xml
GET /usr/share/minigui/res/lang/zh-<LANG>.xml
GET /tmp/data/.data/sqlite/sunxi.db
GET /file/media/<path>
```

### 3.2 完整命令码表

以下是从 APK 逆向确认的全部命令码：

#### 读取命令（GET /api/getdeviceinfo/?custom=1&cmd=）

| 命令码 | 十进制 | 功能 | 返回关键字段 | 优先级 |
|---|---|---|---|---|
| `0x7d1` | 2001 | 基础设备信息 | 设备名、固件版本等 | P0 |
| `0x7d2` | 2002 | 非系统菜单当前值 | `MenuCurrValue` JSON | P1 |
| `0x7d4` | 2004 | SD 卡信息 | `disk_status`, `capacity`, `free_space` | P0 |
| `0x7d5` | 2005 | 录像状态 | `RecodStatus` | P0 |
| `0x7d6` | 2006 | 系统菜单当前值 | `MenuCurrValue` JSON | P1 |
| `0x7d8` | 2008 | 当前工作模式 | `curworkmodename` | P0 |
| `0x7d9` | 2009 | 电量信息 | `level`, `full` | P1 |
| `0xbcd` | 3021 | 设备状态检查 | `status` | P1 |

#### 写入命令（POST /api/setdeviceinfo/?custom=1&cmd=）

| 命令码 | 十进制 | 功能 | 参数 | 优先级 |
|---|---|---|---|---|
| `0x44c` | 1100 | 开始/停止录像 | `&par=1`开始 / `&par=0`停止 | P0 |
| `0x44d` | 1101 | 拍照 | `&par=1`拍照 / `&par=0`取消 | P0 |
| `0xbd9` | 3033 | 进入/退出回放 | `&par=1`进入 / `&par=0`退出 | P0 |
| `0x406` | 1030 | 格式化 SD 卡 | `&par=0` | P1 |
| `0x407` | 1031 | 恢复出厂设置 | `&par=0` | P2 |
| `0xbbb` | 3003 | 设置 Wi-Fi 名称 | `&str=<ssid>` | P1 |
| `0xbbc` | 3004 | 设置 Wi-Fi 密码 | `&str=<pwd>` | P1 |
| `0xbbd` | 3005 | 设置日期 | `&str=<yyyy-MM-dd>` | P1 |
| `0xbbe` | 3006 | 设置时间 | `&str=<HH:mm:ss>` | P1 |
| `0xbda` | 3034 | 切换工作模式 | `&par=<mode_index>` | P1 |
| `0xbcc` | 3020 | 变焦控制 | `&par=<value>` | P2 |
| `0xfa3` | 4003 | 删除文件 | `&par=0&str=<filepath>` | P1 |
| `0xfa4` | 4004 | 缩放/Zoom | `&par=<value>` | P2 |

### 3.3 响应格式

所有 HTTP 响应必须为合法 JSON，Content-Type 建议为 `application/json`。

#### 基础状态 0x7d1 响应

```json
{
  "deviceId": 1,
  "deviceName": "QZ Camera",
  "software": "V1.0.0",
  "status": "0"
}
```

#### SD 卡 0x7d4 响应

```json
{
  "disk_status": "1",
  "capacity": "127512.0",
  "free_space": "92341.0"
}
```

`disk_status`: `"1"` 有卡，`"0"` 无卡。  
`capacity` / `free_space`: 单位 MB，字符串类型。

#### 录像状态 0x7d5 响应

```json
{
  "RecodStatus": "1"
}
```

`"1"` = 录像中，`"0"` = 未录像。注意字段名是 `RecodStatus`（不是 Record）。

#### 当前工作模式 0x7d8 响应

```json
{
  "curworkmodename": "NormalVideo"
}
```

App 已确认的工作模式名：

| 模式名 | 含义 | 对应设置 Key |
|---|---|---|
| `NormalRecordeMode` | 普通录像 | `record_normal_setting_keys` |
| `SlowRecordeMode` | 慢动作录像 | `record_slow_setting_keys` |
| `LoopRecordeMode` | 循环录像 | `record_loop_setting_keys` |
| `TimeLapseMode` | 延时录像 | `record_timelapse_setting_keys` |
| `NormalCaptureMode` | 普通拍照 | `photo_normal_setting_keys` |
| `AutoCaptureMode` | 自动拍照 | `photo_auto_setting_keys` |
| `ContinueCaptureMode` | 连拍 | `photo_burst_setting_keys` |
| `TimingCaptureMode` | 定时拍照 | `photo_time_setting_keys` |

#### 电量 0x7d9 响应

```json
{
  "level": "85",
  "full": "0"
}
```

#### 菜单当前值 0x7d2 / 0x7d6 响应

```json
{
  "deviceId": 1,
  "deviceName": "QZ Camera",
  "software": "V1.0.0",
  "info": [
    { "index": 0, "value": 2 },
    { "index": 1, "value": 0 },
    { "index": 2, "value": 1 }
  ]
}
```

`index` 对应 `setting_keys.xml` 中菜单项的序号，`value` 是当前选中的枚举值索引。

#### 写入命令通用成功响应

```json
{
  "status": "0"
}
```

### 3.4 文件列表接口

#### 请求

```
GET /api/?action=dir&property=<Type>&format=all&from=<offset>&count=<pageSize>&backward=
```

也支持反向查询：
```
GET /api/?action=reardir&property=<Type>&format=all&from=<offset>&count=<pageSize>&backward=
```

#### property 取值

| 类型 | property 值 | 对应目录 |
|---|---|---|
| 普通视频 | `Normal` | `/mnt/DCIM/Normal/` |
| 事件视频 | `Event` | `/mnt/DCIM/Event/` |
| 停车视频 | `Parking` | `/mnt/DCIM/Parking/` |
| 图片 | `Photo` | `/mnt/DCIM/Photo/` |

#### 分页规则

- 默认每页 `20` 条（0x14）
- `from` = 页码 × 20
- `count` = 20

#### 响应格式（XML）

```xml
<list>
  <file>
    <name>/mnt/DCIM/Normal/VID_20260520_173000.MP4</name>
    <size>12345678</size>
    <time>2026-05-20 17:30:00</time>
    <format time="60" />
  </file>
  <file>
    <name>/mnt/DCIM/Normal/VID_20260520_172000.MP4</name>
    <size>23456789</size>
    <time>2026-05-20 17:20:00</time>
    <format time="120" />
  </file>
</list>
```

字段说明：
- `<name>`: 文件完整路径，必须以 `/mnt/` 开头
- `<size>`: 文件大小（字节）
- `<time>`: 格式 `yyyy-MM-dd HH:mm:ss`
- `<format time="N" />`: 视频时长秒数，图片可省略

### 3.5 缩略图服务

App 从文件列表中的路径衍生缩略图 URL。设备端应使 `/thumb/` 路径可访问。

建议映射：
- 原始文件：`http://192.168.10.1:8082/file/media/mnt/DCIM/Normal/VID_xxx.MP4`
- 缩略图：`http://192.168.10.1:8082/thumb/mnt/DCIM/Normal/VID_xxx.MP4.jpg`

缩略图建议规格：320×240 JPEG。

### 3.6 数据库接口

App 会下载 `sunxi.db` 作为相册的另一条数据源：

```
GET /tmp/data/.data/sqlite/sunxi.db
```

下载后 App 本地存为 `DCF.db`，用 SQLite 解析。此接口为 P2 优先级，可延后实现。

### 3.7 菜单与设置资源

#### setting_keys.xml

```
GET /usr/share/minigui/res/lang/setting_keys.xml
```

返回 XML：

```xml
<resources>
  <string-array name="record_normal_setting_keys">
    <item>rec_resolution</item>
    <item>loop_record</item>
    <item>exposure</item>
  </string-array>

  <string-array name="photo_normal_setting_keys">
    <item>photo_resolution</item>
    <item>photo_quality</item>
  </string-array>

  <string-array name="system_setting_keys">
    <item>wifi_ssid</item>
    <item>wifi_password</item>
    <item>date_time</item>
    <item>language</item>
  </string-array>
</resources>
```

App 根据当前 `curworkmodename` 选取对应的 `string-array`。  
`system_setting_keys` 对应 `sysKeys = true`。

模式 → setting_keys 映射（逆向确认）：

| 工作模式 | setting_keys array name |
|---|---|
| `NormalRecordeMode` | `record_normal_setting_keys` |
| `SlowRecordeMode` | `record_slow_setting_keys` |
| `LoopRecordeMode` | `record_loop_setting_keys` |
| `TimeLapseMode` | `record_timelapse_setting_keys` |
| `NormalCaptureMode` | `photo_normal_setting_keys` |
| `AutoCaptureMode` | `photo_auto_setting_keys` |
| `ContinueCaptureMode` | `photo_burst_setting_keys` |
| `TimingCaptureMode` | `photo_time_setting_keys` |

#### 翻译资源

```
GET /usr/share/minigui/res/lang/zh-<LANG>.xml
```

App 从翻译 XML 中解析出 `QZTranslation` 数组，语言码映射：

| 系统语言 | LANG |
|---|---|
| 简体中文 | `CN` |
| 繁体中文 | `TW` |
| 日语 | `JP`（待确认） |
| 韩语 | `KO` |
| 俄语 | `RU` |
| 泰语 | `TI`（注意非标准） |
| 其他 | 回退英文 |

翻译资源返回后 App 将其解析为以下结构：

```json
[
  {
    "str": "rec_resolution",
    "value": "Resolution",
    "items": []
  },
  {
    "str": "rec_resolution_array",
    "value": "",
    "items": [
      { "id": "1080p30", "title": "1080P 30FPS" },
      { "id": "4k30", "title": "4K 30FPS" }
    ]
  }
]
```

规则：
- `str` = 菜单 ID → 提供标题翻译
- `str` = 菜单 ID + `_array` → 提供枚举项列表

---

## 4. RTSP 实时流（端口 8554）

### 4.1 流地址

```
rtsp://192.168.10.1:8554/ch00
```

App 使用 **IJKPlayer**（基于 FFmpeg）播放 RTSP 流，支持的编码：
- 视频：H.264 / H.265
- 建议分辨率：1080P 或以下用于预览
- 传输方式：TCP interleaved（IJK 默认）

### 4.2 实现要求

- 设备启动后 RTSP 服务必须自动运行
- App 进入预览页时立即连接，**不需要额外 HTTP 命令启动流**
- 断开重连时 App 会重新构造 URL 并重新播放
- 延迟要求：< 500ms 可接受，< 200ms 为优

### 4.3 推荐实现

嵌入式平台可选：
- `live555` — 轻量，适合资源有限设备
- `GStreamer` RTSP Server — 功能丰富
- 自研基于 FFmpeg 的 RTSP Server

---

## 5. TCP 事件通道（端口 9999）

### 5.1 连接方式

App 使用 `java.net.Socket` 连接到设备 `192.168.10.1:9999`：
- 连接超时：5000ms（0x1388）
- 读取缓冲区：1024 字节（0x400）
- 保活间隔：500ms（0x1F4）

### 5.2 保活机制

App 连接成功后，会定时发送保活数据：

```
S:100.0
```

这是一个 ASCII 字符串，通过 TCP OutputStream 发送。设备收到后应回复确认（具体回复格式需抓包确认）。

### 5.3 事件推送

设备通过此 TCP 通道向 App 推送事件，App 接收后解析为字符串，分发给 `CaseEventManagerQZ` 处理。

事件消息类型（从代码推断）：
- `CameraEventMessage:connect` — 连接成功
- `CameraEventMessage:break` — 连接断开
- `CameraEventMessage:Exception` — 异常
- `CameraEventMessage:clean` — 清理

### 5.4 实现建议

设备端需要：
1. 在端口 9999 监听 TCP 连接
2. 接收 App 的保活字符串 `S:100.0`
3. 当状态变化（录像开始/停止、SD 卡变化等）时推送事件
4. 数据格式为 UTF-8 字符串

---

## 6. UDP 辅助通道（端口 49142）

### 6.1 连接方式

App 使用 `DatagramSocket` 绑定本地端口 `49142`（0xBFF6），接收设备广播的 UDP 数据包。

### 6.2 用途

- 设备状态广播
- 辅助保活
- 直播相关辅助控制信号

### 6.3 实现优先级

UDP 通道为 **P2 优先级**。在阶段 1 可以不实现，App 主要依赖 HTTP + TCP 事件通道工作。

---

## 7. 初始化链路时序

App 连接设备的完整时序：

```
手机连接设备 Wi-Fi 热点
        │
        ▼
App 检测网关 IP = 192.168.10.1
        │
        ▼
CSManager 切换到 QZ 协议实现
        │
        ├──► TCP 连接 192.168.10.1:9999（事件通道）
        │
        ├──► GET /api/getdeviceinfo/?custom=1&cmd=0x7d1  （基础信息）
        │
        ├──► GET /api/getdeviceinfo/?custom=1&cmd=0x7d5  （录像状态）
        │
        ├──► GET /api/getdeviceinfo/?custom=1&cmd=0x7d4  （SD 卡）
        │
        ├──► GET /api/getdeviceinfo/?custom=1&cmd=0x7d8  （当前模式）
        │
        ├──► 构造 RTSP URL: rtsp://192.168.10.1:8554/ch00
        │
        ├──► GET /usr/share/minigui/res/lang/setting_keys.xml
        │
        ├──► GET /usr/share/minigui/res/lang/zh-CN.xml
        │
        ├──► GET /api/getdeviceinfo/?custom=1&cmd=0x7d2  （非系统菜单值）
        │
        ├──► GET /api/getdeviceinfo/?custom=1&cmd=0x7d6  （系统菜单值）
        │
        └──► 开始 RTSP 播放
```

---

## 8. 设备端架构建议

### 8.1 服务模块划分

```
┌─────────────────────────────────────────────┐
│                  设备固件                      │
├─────────────┬───────────┬──────────┬────────┤
│  HTTP Server │ RTSP Server│ TCP Event│  UDP   │
│   :8082      │  :8554    │  :9999   │ :49142 │
├─────────────┴───────────┴──────────┴────────┤
│              命令处理器（Command Handler）      │
│    ┌──────┬──────┬──────┬──────┬──────┐     │
│    │录像  │拍照  │设置  │文件  │系统  │      │
│    └──────┴──────┴──────┴──────┴──────┘     │
├─────────────────────────────────────────────┤
│           硬件抽象层（HAL）                    │
│  ┌──────┬──────┬──────┬──────┬──────┐       │
│  │Camera│SD Card│Wi-Fi │编码器│GPIO  │       │
│  └──────┴──────┴──────┴──────┴──────┘       │
└─────────────────────────────────────────────┘
```

### 8.2 推荐技术栈

| 组件 | 推荐方案 |
|---|---|
| HTTP Server | `mongoose` / `libmicrohttpd` / `lighttpd` |
| RTSP Server | `live555` / 自研 |
| JSON 生成 | `cJSON` / `jansson` |
| XML 生成 | 模板字符串拼接即可 |
| TCP Server | 标准 socket + select/epoll |
| 数据库 | `sqlite3`（可选，P2） |

---

## 9. 分阶段实现计划

### 阶段 1：最小可演示（P0）

| 序号 | 任务 | 验收标准 |
|---|---|---|
| 1 | AP 热点，网关 `192.168.10.1` | 手机可连接 |
| 2 | HTTP 服务 `:8082` | curl 可访问 |
| 3 | `0x7d1` 基础信息 | 返回合法 JSON |
| 4 | `0x7d4` SD 卡 | 返回容量信息 |
| 5 | `0x7d5` 录像状态 | 返回 RecodStatus |
| 6 | `0x7d8` 当前模式 | 返回 curworkmodename |
| 7 | `0x44c` 录像开始/停止 | 状态切换正确 |
| 8 | `0x44d` 拍照 | 触发拍照 |
| 9 | `0xbd9` 进入/退出回放 | 模式切换 |
| 10 | RTSP `:8554/ch00` | IJK 可播放 |
| 11 | TCP `:9999` 事件通道 | App 可连接、保活正常 |

### 阶段 2：补齐交互（P1）

| 序号 | 任务 |
|---|---|
| 1 | `action=dir` 文件列表 |
| 2 | 缩略图服务 |
| 3 | `setting_keys.xml` |
| 4 | 翻译资源 `zh-CN.xml` |
| 5 | `0x7d2` / `0x7d6` 菜单当前值 |
| 6 | `0x7d9` 电量 |
| 7 | `0xbbb` / `0xbbc` Wi-Fi 设置 |
| 8 | `0xbbd` / `0xbbe` 时间同步 |
| 9 | `0xbda` 模式切换 |
| 10 | `0xfa3` 文件删除 |

### 阶段 3：高兼容（P2）

| 序号 | 任务 |
|---|---|
| 1 | `sunxi.db` 数据库 |
| 2 | UDP 事件通道 |
| 3 | `0x406` 格式化 |
| 4 | `0x407` 恢复出厂 |
| 5 | `0xbcc` / `0xfa4` 变焦 |

---

## 10. 联调测试脚本

以下 curl 命令可用于快速验证设备端实现：

```bash
# 基础信息
curl "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=2001"

# SD 卡
curl "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=2004"

# 录像状态
curl "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=2005"

# 当前模式
curl "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=2008"

# 开始录像
curl -X POST "http://192.168.10.1:8082/api/setdeviceinfo/?custom=1&cmd=1100&par=1"

# 停止录像
curl -X POST "http://192.168.10.1:8082/api/setdeviceinfo/?custom=1&cmd=1100&par=0"

# 拍照
curl -X POST "http://192.168.10.1:8082/api/setdeviceinfo/?custom=1&cmd=1101&par=1"

# RTSP 测试
ffplay rtsp://192.168.10.1:8554/ch00

# 文件列表
curl "http://192.168.10.1:8082/api/?action=dir&property=Normal&format=all&from=0&count=20&backward="

# setting_keys.xml
curl "http://192.168.10.1:8082/usr/share/minigui/res/lang/setting_keys.xml"

# TCP 事件通道测试
nc -v 192.168.10.1 9999
```

> **注意**：App 中命令码使用十六进制传递（如 `cmd=0x7d1`），但也有可能用十进制。建议设备端 **同时兼容两种格式**。实际抓包后以真实请求为准。

---

## 11. 注意事项

1. **字段名严格匹配**：`RecodStatus`（不是 RecordStatus）、`curworkmodename`（全小写）
2. **字符串类型数字**：`disk_status`、`capacity` 等都是字符串，不是数字
3. **菜单三者同源**：`setting_keys.xml`、翻译资源、`MenuCurrValue` 的 index 必须对齐
4. **RTSP 必须自启动**：不需要 HTTP 命令来启动流
5. **TCP 保活必须响应**：App 每 500ms 发送一次 `S:100.0`，设备需维持连接
6. **文件路径规范**：所有文件路径以 `/mnt/DCIM/` 开头
