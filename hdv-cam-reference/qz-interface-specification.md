# QZ 统一接口协议规范

> **文档版本**：v1.0.0  
> **状态**：已定稿，可开工  
> **日期**：2026-05-22  
> **适用范围**：嵌入式固件工程师、App 客户端工程师  
> **使用方式**：双方按本文档独立开发，联调时以本文档为唯一对齐标准

---

## 目录

- [1. 文档说明](#1-文档说明)
- [2. 术语与约定](#2-术语与约定)
- [3. 网络基础](#3-网络基础)
- [4. 通信链路总览](#4-通信链路总览)
- [5. 链路 1：TCP 心跳通道（端口 9999）](#5-链路-1tcp-心跳通道端口-9999)
- [6. 链路 2：HTTP 控制通道（端口 8082）](#6-链路-2http-控制通道端口-8082)
- [7. 链路 3：RTSP 预览通道（端口 8554）](#7-链路-3rtsp-预览通道端口-8554)
- [8. 链路 4：UDP 辅助通道（端口 49142）](#8-链路-4udp-辅助通道端口-49142)
- [9. HTTP 接口规范 — 读取命令](#9-http-接口规范--读取命令)
- [10. HTTP 接口规范 — 写入/控制命令](#10-http-接口规范--写入控制命令)
- [11. HTTP 接口规范 — 文件系统](#11-http-接口规范--文件系统)
- [12. HTTP 接口规范 — 菜单与资源](#12-http-接口规范--菜单与资源)
- [13. 工作模式规范](#13-工作模式规范)
- [14. 数据类型与字段约定](#14-数据类型与字段约定)
- [15. 错误处理约定](#15-错误处理约定)
- [16. 初始化时序](#16-初始化时序)
- [17. 联调验证清单](#17-联调验证清单)
- [18. 完整 Mock 数据集](#18-完整-mock-数据集)
- [19. 风险与待确认项](#19-风险与待确认项)
- [20. 版本记录](#20-版本记录)

---

## 1. 文档说明

### 1.1 本文档是什么

这是嵌入式端和 App 端的**唯一接口合同**。双方按此文档独立开发，不需要等对方完成再开工。

### 1.2 怎么用

| 角色 | 怎么看 |
|---|---|
| **嵌入式工程师** | 你是服务端。按本文档实现所有接口的请求解析和响应生成。用 curl 自测。 |
| **App 工程师** | 你是客户端。按本文档构造请求和解析响应。用 Mock 数据自测。 |
| **联调时** | 以本文档版本号为准。字段名、字段类型、取值范围必须一致。 |

### 1.3 优先级标记

| 标记 | 含义 | 时间 |
|---|---|---|
| **P0** | 最小可演示，必须先做 | 第 1 周 |
| **P1** | 正式交互，补齐功能 | 第 2 周 |
| **P2** | 高兼容，可延后 | 第 3-4 周 |

---

## 2. 术语与约定

| 术语 | 含义 |
|---|---|
| 设备端 / 固件端 | 嵌入式相机，作为 Wi-Fi AP 和各服务的 **Server** 端 |
| App 端 / 客户端 | 手机 App，作为 Wi-Fi STA 和各服务的 **Client** 端 |
| 命令码 | HTTP 请求中 `cmd` 参数的十六进制值，如 `0x7d1` |
| par | 整型参数，跟在命令码后 |
| str | 字符串参数，跟在命令码后 |

### 字段值类型约定

**重要：除非特别标注，所有 JSON 响应中的数值字段均为 `string` 类型**（这是原协议的设计，不是 bug）。

```
正确：{ "disk_status": "1", "capacity": "127512.0" }
错误：{ "disk_status": 1, "capacity": 127512.0 }
```

### 字段名约定

字段名**大小写敏感**，必须严格匹配：

- `RecodStatus` — 不是 `RecordStatus`（原协议拼写）
- `curworkmodename` — 全小写
- `disk_status` — 下划线分隔
- `free_space` — 下划线分隔

---

## 3. 网络基础

### 3.1 网络拓扑

```
┌──────────────┐    Wi-Fi AP    ┌──────────────┐
│  嵌入式相机    │◄─────────────►│   手机 App    │
│ 192.168.10.1  │   STA 接入     │ 192.168.10.x │
└──────────────┘               └──────────────┘
```

### 3.2 固定参数

| 参数 | 值 | 说明 |
|---|---|---|
| 设备 IP | `192.168.10.1` | **硬编码，不可更改** |
| DHCP 范围 | `192.168.10.100 ~ 192.168.10.200` | 建议值 |
| SSID | 无强制要求 | App 通过网关 IP 识别设备，不依赖 SSID |

### 3.3 App 识别逻辑

App 通过当前 Wi-Fi 网关 IP 判断设备类型：

| 网关 IP | 协议 |
|---|---|
| `192.168.10.1` | **QZ**（本文档） |
| `192.168.1.1` | MStar |
| `192.168.169.1` | YZ |

---

## 4. 通信链路总览

QZ 协议包含 **4 条并行链路**，不是单一 HTTP：

| 链路 | 端口 | 协议 | 用途 | 优先级 |
|---|---|---|---|---|
| TCP 心跳 | `9999` | 原始 TCP Socket | 连接保活、事件推送 | **P0** |
| HTTP 控制 | `8082` | HTTP GET/POST | 命令控制、状态读取、资源文件 | **P0** |
| RTSP 预览 | `8554` | RTSP over TCP | 实时视频流 | **P0** |
| UDP 辅助 | `49142` | UDP Datagram | 设备发现广播 + 多机状态同步（App 纯接收） | P2 |

**关键依赖**：TCP 心跳是连接成功的前提。如果设备不监听 9999 端口，App 会判定连接失败，后续 HTTP 和 RTSP 均不会正常工作。

---

## 5. 链路 1：TCP 心跳通道（端口 9999）

**优先级：P0**

### 5.1 连接参数

| 参数 | 值 | 说明 |
|---|---|---|
| 地址 | `192.168.10.1:9999` | |
| 协议 | 原始 TCP Socket | 不是 HTTP，不是 WebSocket |
| 连接超时 | `5000ms` | App 等待连接的最大时间 |
| 接收缓冲区 | `1024` 字节 | App 端的读取缓冲区大小 |
| 编码 | UTF-8 | 所有数据均为 UTF-8 字符串 |

### 5.2 心跳协议

| 方向 | 内容 | 间隔 | 说明 |
|---|---|---|---|
| App → 设备 | `S:100.0` | 每 `500ms` | ASCII 字符串，7 个字节 |
| 设备 → App | 待联调确认 | — | 设备收到心跳后的回复格式需抓包确认 |

### 5.3 事件推送（设备 → App）

设备通过此 TCP 通道向 App 推送状态变化事件：

| 事件类型 | 触发场景 |
|---|---|
| `CameraEventMessage:connect` | TCP 连接建立成功 |
| `CameraEventMessage:break` | 连接断开 |
| `CameraEventMessage:Exception` | 异常发生 |
| `CameraEventMessage:clean` | 清理/重置 |

### 5.4 设备端实现要求

1. 监听 TCP 端口 `9999`，接受客户端连接
2. 接收并识别心跳字符串 `S:100.0`
3. 维持 TCP 连接不主动断开
4. 当设备状态变化时（录像开始/停止、SD 卡插拔等），通过此通道推送事件

### 5.5 App 端实现要求

1. 连接成功后立即启动 500ms 定时器发送 `S:100.0`
2. 监听接收数据，解析事件消息
3. 连接断开时通知 UI 层
4. 支持断线重连

---

## 6. 链路 2：HTTP 控制通道（端口 8082）

**优先级：P0**

### 6.1 基础信息

| 参数 | 值 |
|---|---|
| Base URL | `http://192.168.10.1:8082` |
| Content-Type（响应） | `application/json`（命令接口）/ `text/xml`（文件列表、资源文件） |
| 字符编码 | UTF-8 |
| 请求超时建议 | `5000ms`（普通命令）/ `10000ms`（文件列表） |

### 6.2 四种请求模式

#### 模式 A：读取状态（GET）

```http
GET /api/getdeviceinfo/?custom=1&cmd=<命令码>
```

用途：读取设备信息、SD 卡、录像状态、工作模式、菜单当前值等。

#### 模式 B：写入/控制（POST）

带整型参数：
```http
POST /api/setdeviceinfo/?custom=1&cmd=<命令码>&par=<整型值>
```

带字符串参数：
```http
POST /api/setdeviceinfo/?custom=1&cmd=<命令码>&str=<字符串值>
```

同时带两种参数：
```http
POST /api/setdeviceinfo/?custom=1&cmd=<命令码>&par=<整型值>&str=<字符串值>
```

用途：拍照、录像、设置 Wi-Fi、删除文件等。

#### 模式 C：动作/目录（GET）

```http
GET /api/?action=<动作名>&<参数键值对>
```

用途：文件列表查询。

#### 模式 D：静态资源（GET）

```http
GET /<资源路径>
```

用途：菜单 XML、翻译资源、数据库文件、缩略图等。

### 6.3 命令码传递格式

命令码在 URL 中以**十六进制字符串**传递：

```
正确：cmd=0x7d1
也可能：cmd=2001（十进制）
```

**设备端建议同时兼容十六进制和十进制格式。**

---

## 7. 链路 3：RTSP 预览通道（端口 8554）

**优先级：P0**

### 7.1 流地址

```
rtsp://192.168.10.1:8554/ch00
```

### 7.2 规范

| 参数 | 值 |
|---|---|
| 视频编码 | H.264（必须）/ H.265（可选） |
| 传输方式 | TCP interleaved |
| 预览分辨率 | 1080P 或以下 |
| 延迟要求 | < 500ms 可接受，< 200ms 为优 |

### 7.3 设备端要求

- RTSP 服务**必须随设备启动自动运行**
- **不需要任何 HTTP 命令来启动流**，App 直接连接即播放
- 断开后 App 会重新构造 URL 重连

### 7.4 App 端要求

- 进入预览页时立即连接 RTSP
- 使用 TCP 传输模式（`rtsp_transport=tcp`）
- 支持断线自动重连

---

## 8. 链路 4：UDP 辅助通道（端口 49142）

**优先级：P2（可延后）**

| 参数 | 值 |
|---|---|
| 端口 | `49142`（0xBFF6） |
| 协议 | UDP Datagram |
| 方向 | 设备 → App（设备广播，App 纯接收） |
| 用途 | 设备发现广播 + 多机状态同步 |

**逆向确认**：
- `CaseEventManagerQZ.java`：`new DatagramSocket(49142)` + `DSocket.receive(packet)`，App 仅接收
- `sendData()` 走 TCP `outStream.write()`，不走 UDP
- I 帧刷新由 Native 层（`libijkplayer.so`）通过 RTCP PLI（RFC 4585）自动处理，不走此端口
- 整个 `com.record.*` 包中未找到 I-frame / IDR / keyframe 相关代码

阶段 1 和阶段 2 可以不实现。App 主要依赖 HTTP + TCP 心跳工作。

---

## 9. HTTP 接口规范 — 读取命令

所有读取命令使用 **GET** 请求，URL 模板：
```
GET /api/getdeviceinfo/?custom=1&cmd=<命令码>
```

---

### 9.1 设备基础信息（0x7d1）

**优先级：P0**

#### 请求

```http
GET /api/getdeviceinfo/?custom=1&cmd=0x7d1
```

#### 响应

```json
{
  "deviceId": "1",
  "deviceName": "QZ Camera",
  "software": "V1.0.0",
  "status": "0"
}
```

#### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | string | 是 | 设备 ID |
| `deviceName` | string | 是 | 设备名称 |
| `software` | string | 是 | 固件版本号 |
| `status` | string | 是 | `"0"` 正常 |

#### 联调备注

此命令返回的完整字段集仍需联调校准，以上为最小必须字段。

---

### 9.2 SD 卡信息（0x7d4）

**优先级：P0**

#### 请求

```http
GET /api/getdeviceinfo/?custom=1&cmd=0x7d4
```

#### 响应

```json
{
  "disk_status": "1",
  "capacity": "127512.0",
  "free_space": "92341.0"
}
```

#### 字段说明

| 字段 | 类型 | 必填 | 取值范围 | 说明 |
|---|---|---|---|---|
| `disk_status` | string | 是 | `"0"` / `"1"` | `"0"` 无卡，`"1"` 有卡 |
| `capacity` | string | 是 | 浮点数字符串 | SD 卡总容量，单位 MB |
| `free_space` | string | 是 | 浮点数字符串 | 剩余空间，单位 MB |

#### 关键约定

- 容量值为**字符串类型的浮点数**（如 `"127512.0"`），不是 number
- App 端解析时需做 `parseFloat` / `double.tryParse`

---

### 9.3 录像状态（0x7d5）

**优先级：P0**

#### 请求

```http
GET /api/getdeviceinfo/?custom=1&cmd=0x7d5
```

#### 响应

```json
{
  "RecodStatus": "1"
}
```

#### 字段说明

| 字段 | 类型 | 必填 | 取值范围 | 说明 |
|---|---|---|---|---|
| `RecodStatus` | string | 是 | `"0"` / `"1"` | `"0"` 未录像，`"1"` 录像中 |

#### 关键约定

- 字段名是 **`RecodStatus`**，不是 `RecordStatus`。这是原协议的拼写，双方必须严格匹配。

---

### 9.4 当前工作模式（0x7d8）

**优先级：P0**

#### 请求

```http
GET /api/getdeviceinfo/?custom=1&cmd=0x7d8
```

#### 响应

```json
{
  "curworkmodename": "NormalRecordeMode"
}
```

#### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `curworkmodename` | string | 是 | 当前工作模式名，取值见 [13. 工作模式规范](#13-工作模式规范) |

#### 关键约定

- 字段名 **`curworkmodename`** 全小写
- 取值必须是 8 种工作模式名之一（见第 13 章）

---

### 9.5 电池信息（0x7d9）

**优先级：P1**

#### 请求

```http
GET /api/getdeviceinfo/?custom=1&cmd=0x7d9
```

#### 响应

```json
{
  "level": "85",
  "full": "0"
}
```

#### 字段说明

| 字段 | 类型 | 必填 | 取值范围 | 说明 |
|---|---|---|---|---|
| `level` | string | 是 | `"0"` ~ `"100"` | 电量百分比 |
| `full` | string | 是 | `"0"` / `"1"` | `"1"` 已充满 |

---

### 9.6 菜单当前值 — 非系统（0x7d2）

**优先级：P1**

#### 请求

```http
GET /api/getdeviceinfo/?custom=1&cmd=0x7d2
```

#### 响应

```json
{
  "deviceId": "1",
  "deviceName": "QZ Camera",
  "software": "V1.0.0",
  "info": [
    { "index": 0, "value": 2 },
    { "index": 1, "value": 0 },
    { "index": 2, "value": 1 }
  ]
}
```

#### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | string | 是 | 设备 ID |
| `deviceName` | string | 是 | 设备名称 |
| `software` | string | 是 | 固件版本号 |
| `info` | array | 是 | 菜单值数组 |
| `info[].index` | number | 是 | 对应 `setting_keys.xml` 中当前模式 setting_keys 数组的项序号 |
| `info[].value` | number | 是 | 当前选中的枚举值索引（对应翻译资源 `_array` 中的项序号） |

#### 映射关系

```
setting_keys.xml 中：
  record_normal_setting_keys → ["rec_resolution", "loop_record", "exposure"]
                                  index=0          index=1        index=2

0x7d2 响应中：
  info: [{ index: 0, value: 2 }]
  ↓
  表示 rec_resolution 当前选中第 3 个枚举值（index=2）

翻译资源 rec_resolution_array 中：
  items: ["720p30", "1080p30", "4k30"]
                                 ↑ value=2 → "4k30"
```

#### 关键约定

- `info` 数组中的 `index` 和 `value` 是 **number 类型**（不是 string）
- 此命令返回**非系统菜单**的当前值，对应当前工作模式的 `<mode>_setting_keys`

---

### 9.7 菜单当前值 — 系统（0x7d6）

**优先级：P1**

#### 请求

```http
GET /api/getdeviceinfo/?custom=1&cmd=0x7d6
```

#### 响应

格式与 0x7d2 完全一致，但返回的是**系统菜单**（`system_setting_keys`）的当前值。

---

### 9.8 设备检查（0xbcd）

**优先级：P1**

#### 请求

```http
GET /api/getdeviceinfo/?custom=1&cmd=0xbcd
```

#### 响应

```json
{
  "status": "0"
}
```

#### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `status` | string | 是 | `"0"` 设备正常在线 |

---

## 10. HTTP 接口规范 — 写入/控制命令

所有写入命令使用 **POST** 请求，URL 模板：
```
POST /api/setdeviceinfo/?custom=1&cmd=<命令码>&par=<值>
POST /api/setdeviceinfo/?custom=1&cmd=<命令码>&str=<值>
```

**通用成功响应**（除非接口另有说明）：

```json
{
  "status": "0"
}
```

---

### 10.1 开始/停止录像（0x44c）

**优先级：P0**

#### 开始录像

```http
POST /api/setdeviceinfo/?custom=1&cmd=0x44c&par=1
```

#### 停止录像

```http
POST /api/setdeviceinfo/?custom=1&cmd=0x44c&par=0
```

#### 参数说明

| 参数 | 类型 | 取值 | 说明 |
|---|---|---|---|
| `par` | int | `1` | 开始录像 |
| `par` | int | `0` | 停止录像 |

#### 设备端行为

- 收到 `par=1` 后开始录像，`0x7d5` 的 `RecodStatus` 应变为 `"1"`
- 收到 `par=0` 后停止录像，`RecodStatus` 应变为 `"0"`
- 建议通过 TCP 9999 通道推送录像状态变化事件

---

### 10.2 拍照（0x44d）

**优先级：P0**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0x44d&par=1
```

#### 参数说明

| 参数 | 类型 | 取值 | 说明 |
|---|---|---|---|
| `par` | int | `1` | 执行拍照 |
| `par` | int | `0` | 取消 |

#### 设备端行为

- 拍照完成后，新文件应出现在 `action=dir&property=Photo` 的列表中
- 文件命名建议：`IMG_yyyyMMdd_HHmmss.jpg`
- 存储路径：`/mnt/DCIM/Photo/`

---

### 10.3 进入/退出回放模式（0xbd9）

**优先级：P0**

#### 进入回放

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xbd9&par=1
```

#### 退出回放

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xbd9&par=0
```

#### 参数说明

| 参数 | 类型 | 取值 | 说明 |
|---|---|---|---|
| `par` | int | `1` | 进入回放模式 |
| `par` | int | `0` | 退出回放模式 |

---

### 10.4 设置 Wi-Fi 名称（0xbbb）

**优先级：P1**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xbbb&str=MyCamera
```

#### 参数说明

| 参数 | 类型 | 说明 |
|---|---|---|
| `str` | string | 新的 Wi-Fi SSID |

#### 设备端行为

- 修改 AP 热点名称
- 修改后手机需重新连接

---

### 10.5 设置 Wi-Fi 密码（0xbbc）

**优先级：P1**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xbbc&str=12345678
```

#### 参数说明

| 参数 | 类型 | 说明 |
|---|---|---|
| `str` | string | 新的 Wi-Fi 密码 |

#### 设备端行为

- 修改 AP 热点密码
- 修改后手机需重新连接

---

### 10.6 设置日期（0xbbd）

**优先级：P1**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xbbd&str=2026-05-22
```

#### 参数说明

| 参数 | 类型 | 格式 | 说明 |
|---|---|---|---|
| `str` | string | `yyyy-MM-dd` | 日期字符串 |

---

### 10.7 设置时间（0xbbe）

**优先级：P1**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xbbe&str=14:30:00
```

#### 参数说明

| 参数 | 类型 | 格式 | 说明 |
|---|---|---|---|
| `str` | string | `HH:mm:ss` | 时间字符串，24 小时制 |

---

### 10.8 切换工作模式（0xbda）

**优先级：P1**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xbda&par=4
```

#### 参数说明

| 参数 | 类型 | 取值范围 | 说明 |
|---|---|---|---|
| `par` | int | `0` ~ `7` | 工作模式编号，见 [13. 工作模式规范](#13-工作模式规范) |

#### 设备端行为

- 切换到指定工作模式
- 切换后 `0x7d8` 返回的 `curworkmodename` 应更新为对应模式名

#### App 端行为

- 切换模式后，App 会用新模式对应的 `setting_keys` 数组名重新请求 `0x7d2` 获取菜单当前值

---

### 10.9 删除文件（0xfa3）

**优先级：P1**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xfa3&par=0&str=/mnt/DCIM/Normal/VID_20260520_173000.MP4
```

#### 参数说明

| 参数 | 类型 | 说明 |
|---|---|---|
| `par` | int | 固定为 `0` |
| `str` | string | 要删除的文件完整路径 |

---

### 10.10 格式化 SD 卡（0x406）

**优先级：P2**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0x406&par=0
```

---

### 10.11 恢复出厂设置（0x407）

**优先级：P2**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0x407&par=0
```

---

### 10.12 变焦控制（0xbcc）

**优先级：P2**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xbcc&par=5
```

#### 参数说明

| 参数 | 类型 | 说明 |
|---|---|---|
| `par` | int | 变焦级别（具体范围需联调确认） |

---

### 10.13 变焦 DV（0xfa4）

**优先级：P2**

#### 请求

```http
POST /api/setdeviceinfo/?custom=1&cmd=0xfa4&par=5
```

---

## 11. HTTP 接口规范 — 文件系统

### 11.1 文件列表（action=dir）

**优先级：P1**

#### 请求

```http
GET /api/?action=dir&property=Normal&format=all&from=0&count=20&backward=
```

反向查询（从后往前）：
```http
GET /api/?action=reardir&property=Normal&format=all&from=0&count=20&backward=
```

#### 参数说明

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `action` | string | 是 | `dir`（正向）/ `reardir`（反向） |
| `property` | string | 是 | 媒体类型，见下表 |
| `format` | string | 是 | 固定 `all` |
| `from` | int | 是 | 起始偏移量，`page * 20` |
| `count` | int | 是 | 每页数量，固定 `20` |
| `backward` | string | 是 | 留空 |

#### property 取值

| 媒体类型 | property 值 | 对应存储目录 |
|---|---|---|
| 普通视频 | `Normal` | `/mnt/DCIM/Normal/` |
| 事件视频 | `Event` | `/mnt/DCIM/Event/` |
| 停车视频 | `Parking` | `/mnt/DCIM/Parking/` |
| 图片 | `Photo` | `/mnt/DCIM/Photo/` |

#### 响应（XML 格式）

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

#### XML 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `<name>` | string | 是 | 文件完整路径，必须以 `/mnt/` 开头 |
| `<size>` | string(int) | 是 | 文件大小，单位字节 |
| `<time>` | string | 是 | 拍摄时间，格式 `yyyy-MM-dd HH:mm:ss` |
| `<format time="N" />` | attribute | 视频必填 | 时长秒数，图片可省略 |

#### 空列表响应

```xml
<list>
</list>
```

---

### 11.2 缩略图

**优先级：P1**

#### 请求

```http
GET /thumb/mnt/DCIM/Normal/VID_20260520_173000.MP4.jpg
```

#### 规范

| 参数 | 值 |
|---|---|
| URL 前缀 | `/thumb/` |
| 路径规则 | 原文件路径去掉开头 `/`，末尾加 `.jpg` |
| 图片格式 | JPEG |
| 建议尺寸 | 320 × 240 |

#### 路径对应关系

| 原文件路径 | 缩略图 URL |
|---|---|
| `/mnt/DCIM/Normal/VID_xxx.MP4` | `/thumb/mnt/DCIM/Normal/VID_xxx.MP4.jpg` |
| `/mnt/DCIM/Photo/IMG_xxx.jpg` | `/thumb/mnt/DCIM/Photo/IMG_xxx.jpg.jpg` |

---

### 11.3 文件下载

**优先级：P1**

#### 请求

```http
GET /file/media/mnt/DCIM/Normal/VID_20260520_173000.MP4
```

#### 规范

| 参数 | 值 |
|---|---|
| URL 前缀 | `/file/media/` |
| 路径规则 | 原文件路径去掉开头 `/` |

---

### 11.4 数据库文件（sunxi.db）

**优先级：P2**

#### 请求

```http
GET /tmp/data/.data/sqlite/sunxi.db
```

#### 说明

- 完整路径**必须**是 `/tmp/data/.data/sqlite/sunxi.db`，不是 `/tmp/sunxi.db`
- App 下载后本地保存为 `DCF.db`，走 SQLite 解析链
- 阶段 1-2 可不实现，先保证 `action=dir` 路径可用

---

### 11.5 文件命名规范

| 类型 | 命名格式 | 示例 |
|---|---|---|
| 普通视频 | `VID_yyyyMMdd_HHmmss.MP4` | `VID_20260520_173000.MP4` |
| 事件视频 | `EVT_yyyyMMdd_HHmmss.MP4` | `EVT_20260520_173000.MP4` |
| 停车视频 | `PKG_yyyyMMdd_HHmmss.MP4` | `PKG_20260520_173000.MP4` |
| 图片 | `IMG_yyyyMMdd_HHmmss.jpg` | `IMG_20260520_173000.jpg` |

---

## 12. HTTP 接口规范 — 菜单与资源

### 12.1 setting_keys.xml

**优先级：P1**

#### 请求

```http
GET /usr/share/minigui/res/lang/setting_keys.xml
```

#### 响应

```xml
<resources>
  <!-- 各工作模式的设置项列表 -->
  <string-array name="record_normal_setting_keys">
    <item>rec_resolution</item>
    <item>loop_record</item>
    <item>exposure</item>
  </string-array>

  <string-array name="record_slow_setting_keys">
    <item>rec_resolution</item>
    <item>slow_rate</item>
  </string-array>

  <string-array name="record_loop_setting_keys">
    <item>rec_resolution</item>
    <item>loop_duration</item>
  </string-array>

  <string-array name="record_timelapse_setting_keys">
    <item>rec_resolution</item>
    <item>timelapse_interval</item>
  </string-array>

  <string-array name="photo_normal_setting_keys">
    <item>photo_resolution</item>
    <item>photo_quality</item>
  </string-array>

  <string-array name="photo_auto_setting_keys">
    <item>photo_resolution</item>
    <item>auto_interval</item>
  </string-array>

  <string-array name="photo_continue_setting_keys">
    <item>photo_resolution</item>
    <item>burst_count</item>
  </string-array>

  <string-array name="photo_time_setting_keys">
    <item>photo_resolution</item>
    <item>timer_delay</item>
  </string-array>

  <!-- 系统设置 -->
  <string-array name="system_setting_keys">
    <item>wifi_ssid</item>
    <item>wifi_password</item>
    <item>date_time</item>
    <item>language</item>
    <item>format_card</item>
    <item>factory_reset</item>
  </string-array>
</resources>
```

#### 解析规则

| 规则 | 说明 |
|---|---|
| `<item>` 文本 | 菜单 ID |
| `<item>` 顺序 | 菜单 index（从 0 开始） |
| `name="system_setting_keys"` | 标记为系统菜单（`sysKeys = true`） |
| `name="<mode>_setting_keys"` | 对应工作模式的设置项列表 |

#### 关键约定

- **设备端**：根据自身支持的功能生成此文件
- **App 端**：根据当前 `curworkmodename` 选取对应的 `<mode>_setting_keys` 数组解析菜单
- 切换工作模式后，App 会重新用新模式对应的 `setting_keys` 数组名拉取菜单项

---

### 12.2 翻译资源（zh-XX.xml）

**优先级：P1**

#### 请求

```http
GET /usr/share/minigui/res/lang/zh-CN.xml
```

#### 语言码映射

| 语言 | 码 | 文件名 |
|---|---|---|
| 简体中文 | `CN` | `zh-CN.xml` |
| 繁体中文 | `TW` | `zh-TW.xml` |
| 韩语 | `KO` | `zh-KO.xml` |
| 俄语 | `RU` | `zh-RU.xml` |
| 泰语 | `TI` | `zh-TI.xml`（注意非标准码） |
| 日语 | `JP` | `zh-JP.xml`（待确认） |
| 英文 | 回退 | 使用 `value` 字段的英文默认值 |

#### 响应结构

翻译资源被 App 解析为 JSON 数组，每个菜单 ID 产生**两条**记录：

**记录 1：菜单标题**

```json
{
  "str": "rec_resolution",
  "value": "分辨率",
  "items": []
}
```

**记录 2：枚举值列表**

```json
{
  "str": "rec_resolution_array",
  "value": "",
  "items": [
    { "id": "720p30", "title": "720P 30FPS" },
    { "id": "1080p30", "title": "1080P 30FPS" },
    { "id": "4k30", "title": "4K 30FPS" }
  ]
}
```

#### 命名规则

| 用途 | str 格式 | 示例 |
|---|---|---|
| 菜单标题 | `<菜单ID>` | `rec_resolution` |
| 枚举值列表 | `<菜单ID>_array` | `rec_resolution_array` |

#### 关键约定

- **菜单、翻译、当前值三者必须同源**
  - `setting_keys.xml` 中菜单 ID 的 index → `0x7d2` / `0x7d6` 的 `info[].index`
  - 翻译资源 `_array` 中项的 index → `0x7d2` / `0x7d6` 的 `info[].value`
- 三者任何一个不对齐，App 设置页会显示错乱

---

## 13. 工作模式规范

### 13.1 模式枚举表

| 编号 | 模式名 | 中文 | 类型 |
|---|---|---|---|
| `0` | `NormalRecordeMode` | 普通录像 | 录像 |
| `1` | `SlowRecordeMode` | 慢动作 | 录像 |
| `2` | `LoopRecordeMode` | 循环录像 | 录像 |
| `3` | `TimeLapseMode` | 延时摄影 | 录像 |
| `4` | `NormalCaptureMode` | 普通拍照 | 拍照 |
| `5` | `AutoCaptureMode` | 自动拍照 | 拍照 |
| `6` | `ContinueCaptureMode` | 连拍 | 拍照 |
| `7` | `TimingCaptureMode` | 定时拍照 | 拍照 |

### 13.2 模式 → setting_keys 映射

| 编号 | 模式名 | setting_keys 数组名 |
|---|---|---|
| 0 | `NormalRecordeMode` | `record_normal_setting_keys` |
| 1 | `SlowRecordeMode` | `record_slow_setting_keys` |
| 2 | `LoopRecordeMode` | `record_loop_setting_keys` |
| 3 | `TimeLapseMode` | `record_timelapse_setting_keys` |
| 4 | `NormalCaptureMode` | `photo_normal_setting_keys` |
| 5 | `AutoCaptureMode` | `photo_auto_setting_keys` |
| 6 | `ContinueCaptureMode` | `photo_continue_setting_keys` |
| 7 | `TimingCaptureMode` | `photo_time_setting_keys` |

### 13.3 模式 → 分辨率键名映射

| 编号 | 模式名 | Camera.Menu 分辨率键名 |
|---|---|---|
| 0 | `NormalRecordeMode` | `Camera.Menu.NRecRes` |
| 1 | `SlowRecordeMode` | `Camera.Menu.SRecRes` |
| 2 | `LoopRecordeMode` | `Camera.Menu.LRecRes` |
| 3 | `TimeLapseMode` | `Camera.Menu.TRecRes` |
| 4 | `NormalCaptureMode` | `Camera.Menu.NPhotoRes` |
| 5 | `AutoCaptureMode` | `Camera.Menu.APhotoRes` |
| 6 | `ContinueCaptureMode` | `Camera.Menu.CPhotoRes` |
| 7 | `TimingCaptureMode` | `Camera.Menu.TPhotoRes` |

### 13.4 模式切换流程

```
App 发送 POST cmd=0xbda&par=<新模式编号>
    │
    ▼
设备切换工作模式
    │
    ▼
App 读取 GET cmd=0x7d8 确认 curworkmodename 已更新
    │
    ▼
App 根据新模式的 setting_keys 数组名重新请求 0x7d2
    │
    ▼
App 刷新设置页菜单
```

---

## 14. 数据类型与字段约定

### 14.1 类型对照表

| 文档标记 | JSON 实际类型 | 示例 | 说明 |
|---|---|---|---|
| `string` | `string` | `"1"` | |
| `string(int)` | `string` | `"12345678"` | 内容为整数的字符串 |
| `string(float)` | `string` | `"127512.0"` | 内容为浮点数的字符串 |
| `number` | `number` | `2` | 只在 `info[].index` 和 `info[].value` 中使用 |
| `array` | `array` | `[...]` | |

### 14.2 字段名严格清单

以下字段名拼写已通过逆向确认，双方必须严格匹配：

| 字段名 | 出处 | 注意 |
|---|---|---|
| `RecodStatus` | 0x7d5 | 不是 RecordStatus |
| `curworkmodename` | 0x7d8 | 全小写 |
| `disk_status` | 0x7d4 | 下划线 |
| `capacity` | 0x7d4 | |
| `free_space` | 0x7d4 | 下划线 |
| `deviceId` | 0x7d1/0x7d2/0x7d6 | 驼峰 |
| `deviceName` | 0x7d1/0x7d2/0x7d6 | 驼峰 |
| `software` | 0x7d1/0x7d2/0x7d6 | |
| `level` | 0x7d9 | |
| `full` | 0x7d9 | |
| `NormalRecordeMode` | 0x7d8 | 注意 Recorde 拼写 |

---

## 15. 错误处理约定

### 15.1 HTTP 错误

| 场景 | 设备端返回 | App 端处理 |
|---|---|---|
| 命令成功 | HTTP 200 + `{"status": "0"}` | 正常处理 |
| 未知命令码 | HTTP 200 + `{"status": "-1"}` | 提示操作失败 |
| 参数缺失 | HTTP 400 | 提示参数错误 |
| 服务不可用 | HTTP 503 / 连接超时 | 提示设备离线，尝试重连 |

### 15.2 TCP 错误

| 场景 | 处理 |
|---|---|
| 连接超时（>5000ms） | App 判定设备未就绪，提示用户检查 Wi-Fi |
| 连接断开 | App 尝试重连，超过 3 次提示用户 |
| 心跳无响应 | App 继续发送心跳，由设备端决定是否断开 |

### 15.3 RTSP 错误

| 场景 | 处理 |
|---|---|
| 连接失败 | 等待 2 秒后重试，最多 3 次 |
| 播放中断 | 自动重连 |

---

## 16. 初始化时序

### 16.1 完整流程图

```
手机连接设备 Wi-Fi 热点
│
▼ App 检测网关 IP = 192.168.10.1
│
▼ 切换到 QZ 协议实现
│
├─[1] TCP 连接 192.168.10.1:9999           ← P0，必须最先
│     └─ 成功后启动 500ms 心跳定时器
│
├─[2] 并行读取基础状态（HTTP）              ← P0
│     ├─ GET cmd=0x7d1  设备基础信息
│     ├─ GET cmd=0x7d4  SD 卡信息
│     ├─ GET cmd=0x7d5  录像状态
│     └─ GET cmd=0x7d8  当前工作模式
│
├─[3] 构造并连接 RTSP                       ← P0
│     └─ rtsp://192.168.10.1:8554/ch00
│
├─[4] 加载菜单资源（HTTP）                   ← P1，可异步
│     ├─ GET setting_keys.xml
│     ├─ GET zh-CN.xml（翻译资源）
│     ├─ GET cmd=0x7d2（非系统菜单值）
│     └─ GET cmd=0x7d6（系统菜单值）
│
├─[5] 加载文件列表                           ← P1，可异步
│     └─ GET action=dir
│
└─[6] 进入主界面，预览画面显示
```

### 16.2 设备端启动顺序

```
设备上电
│
├─[1] 启动 AP 热点，网关 192.168.10.1
├─[2] 启动 TCP 监听 :9999
├─[3] 启动 HTTP 服务 :8082
├─[4] 启动 RTSP 服务 :8554
└─[5] （可选）启动 UDP 监听 :49142
```

**关键：TCP 9999 必须在 HTTP 8082 之前或同时就绪。**

---

## 17. 联调验证清单

### 17.1 设备端自测（curl）

```bash
# ===== P0 =====

# 1. TCP 心跳通道（用 nc 测试）
nc -v 192.168.10.1 9999
# 连接成功后输入 S:100.0 回车，观察是否保持连接

# 2. 设备基础信息
curl -s "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=0x7d1" | python3 -m json.tool

# 3. SD 卡信息
curl -s "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=0x7d4" | python3 -m json.tool

# 4. 录像状态
curl -s "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=0x7d5" | python3 -m json.tool

# 5. 当前工作模式
curl -s "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=0x7d8" | python3 -m json.tool

# 6. 开始录像
curl -s -X POST "http://192.168.10.1:8082/api/setdeviceinfo/?custom=1&cmd=0x44c&par=1" | python3 -m json.tool

# 7. 停止录像
curl -s -X POST "http://192.168.10.1:8082/api/setdeviceinfo/?custom=1&cmd=0x44c&par=0" | python3 -m json.tool

# 8. 拍照
curl -s -X POST "http://192.168.10.1:8082/api/setdeviceinfo/?custom=1&cmd=0x44d&par=1" | python3 -m json.tool

# 9. RTSP 预览
ffplay rtsp://192.168.10.1:8554/ch00

# ===== P1 =====

# 10. 文件列表
curl -s "http://192.168.10.1:8082/api/?action=dir&property=Normal&format=all&from=0&count=20&backward="

# 11. setting_keys.xml
curl -s "http://192.168.10.1:8082/usr/share/minigui/res/lang/setting_keys.xml"

# 12. 翻译资源
curl -s "http://192.168.10.1:8082/usr/share/minigui/res/lang/zh-CN.xml"

# 13. 菜单当前值
curl -s "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=0x7d2" | python3 -m json.tool
curl -s "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=0x7d6" | python3 -m json.tool

# 14. 电池
curl -s "http://192.168.10.1:8082/api/getdeviceinfo/?custom=1&cmd=0x7d9" | python3 -m json.tool

# 15. 切换工作模式
curl -s -X POST "http://192.168.10.1:8082/api/setdeviceinfo/?custom=1&cmd=0xbda&par=4" | python3 -m json.tool

# 16. 缩略图
curl -s -o thumb.jpg "http://192.168.10.1:8082/thumb/mnt/DCIM/Normal/VID_20260520_173000.MP4.jpg"
```

### 17.2 App 端自测（Mock Server）

App 端可以先用本地 Mock Server 替代设备，Mock 数据见 [第 18 章](#18-完整-mock-数据集)。

---

## 18. 完整 Mock 数据集

以下 Mock 数据供双方开发时使用：

### 18.1 cmd=0x7d1 设备信息

```json
{
  "deviceId": "1",
  "deviceName": "QZ-CAM-001",
  "software": "V1.0.0",
  "status": "0"
}
```

### 18.2 cmd=0x7d4 SD 卡

```json
{
  "disk_status": "1",
  "capacity": "127512.0",
  "free_space": "92341.0"
}
```

### 18.3 cmd=0x7d5 录像状态

未录像：
```json
{
  "RecodStatus": "0"
}
```

录像中：
```json
{
  "RecodStatus": "1"
}
```

### 18.4 cmd=0x7d8 工作模式

```json
{
  "curworkmodename": "NormalRecordeMode"
}
```

### 18.5 cmd=0x7d9 电池

```json
{
  "level": "85",
  "full": "0"
}
```

### 18.6 cmd=0x7d2 非系统菜单值

```json
{
  "deviceId": "1",
  "deviceName": "QZ-CAM-001",
  "software": "V1.0.0",
  "info": [
    { "index": 0, "value": 2 },
    { "index": 1, "value": 0 },
    { "index": 2, "value": 1 }
  ]
}
```

### 18.7 cmd=0x7d6 系统菜单值

```json
{
  "deviceId": "1",
  "deviceName": "QZ-CAM-001",
  "software": "V1.0.0",
  "info": [
    { "index": 0, "value": 0 },
    { "index": 1, "value": 0 },
    { "index": 2, "value": 0 },
    { "index": 3, "value": 0 }
  ]
}
```

### 18.8 cmd=0xbcd 设备检查

```json
{
  "status": "0"
}
```

### 18.9 写入命令通用成功

```json
{
  "status": "0"
}
```

### 18.10 文件列表（Normal）

```xml
<list>
  <file>
    <name>/mnt/DCIM/Normal/VID_20260522_143000.MP4</name>
    <size>52428800</size>
    <time>2026-05-22 14:30:00</time>
    <format time="120" />
  </file>
  <file>
    <name>/mnt/DCIM/Normal/VID_20260522_141500.MP4</name>
    <size>26214400</size>
    <time>2026-05-22 14:15:00</time>
    <format time="60" />
  </file>
  <file>
    <name>/mnt/DCIM/Normal/VID_20260521_100000.MP4</name>
    <size>104857600</size>
    <time>2026-05-21 10:00:00</time>
    <format time="300" />
  </file>
</list>
```

### 18.11 文件列表（Photo）

```xml
<list>
  <file>
    <name>/mnt/DCIM/Photo/IMG_20260522_143500.jpg</name>
    <size>3145728</size>
    <time>2026-05-22 14:35:00</time>
  </file>
  <file>
    <name>/mnt/DCIM/Photo/IMG_20260522_142000.jpg</name>
    <size>2621440</size>
    <time>2026-05-22 14:20:00</time>
  </file>
</list>
```

### 18.12 setting_keys.xml

```xml
<resources>
  <string-array name="record_normal_setting_keys">
    <item>rec_resolution</item>
    <item>loop_record</item>
    <item>exposure</item>
  </string-array>

  <string-array name="record_slow_setting_keys">
    <item>rec_resolution</item>
    <item>slow_rate</item>
  </string-array>

  <string-array name="record_loop_setting_keys">
    <item>rec_resolution</item>
    <item>loop_duration</item>
  </string-array>

  <string-array name="record_timelapse_setting_keys">
    <item>rec_resolution</item>
    <item>timelapse_interval</item>
  </string-array>

  <string-array name="photo_normal_setting_keys">
    <item>photo_resolution</item>
    <item>photo_quality</item>
  </string-array>

  <string-array name="photo_auto_setting_keys">
    <item>photo_resolution</item>
    <item>auto_interval</item>
  </string-array>

  <string-array name="photo_continue_setting_keys">
    <item>photo_resolution</item>
    <item>burst_count</item>
  </string-array>

  <string-array name="photo_time_setting_keys">
    <item>photo_resolution</item>
    <item>timer_delay</item>
  </string-array>

  <string-array name="system_setting_keys">
    <item>wifi_ssid</item>
    <item>wifi_password</item>
    <item>date_time</item>
    <item>language</item>
    <item>format_card</item>
    <item>factory_reset</item>
  </string-array>
</resources>
```

### 18.13 翻译资源 zh-CN.xml（Mock）

```json
[
  { "str": "rec_resolution", "value": "分辨率", "items": [] },
  { "str": "rec_resolution_array", "value": "", "items": [
    { "id": "720p30", "title": "720P 30FPS" },
    { "id": "1080p30", "title": "1080P 30FPS" },
    { "id": "4k30", "title": "4K 30FPS" }
  ]},
  { "str": "loop_record", "value": "循环录像", "items": [] },
  { "str": "loop_record_array", "value": "", "items": [
    { "id": "off", "title": "关闭" },
    { "id": "1min", "title": "1分钟" },
    { "id": "3min", "title": "3分钟" },
    { "id": "5min", "title": "5分钟" }
  ]},
  { "str": "exposure", "value": "曝光补偿", "items": [] },
  { "str": "exposure_array", "value": "", "items": [
    { "id": "n2", "title": "-2.0" },
    { "id": "n1", "title": "-1.0" },
    { "id": "0", "title": "0" },
    { "id": "p1", "title": "+1.0" },
    { "id": "p2", "title": "+2.0" }
  ]},
  { "str": "photo_resolution", "value": "照片分辨率", "items": [] },
  { "str": "photo_resolution_array", "value": "", "items": [
    { "id": "8m", "title": "8M" },
    { "id": "12m", "title": "12M" },
    { "id": "16m", "title": "16M" }
  ]},
  { "str": "photo_quality", "value": "照片质量", "items": [] },
  { "str": "photo_quality_array", "value": "", "items": [
    { "id": "normal", "title": "普通" },
    { "id": "fine", "title": "精细" },
    { "id": "superfine", "title": "超精细" }
  ]},
  { "str": "wifi_ssid", "value": "Wi-Fi名称", "items": [] },
  { "str": "wifi_password", "value": "Wi-Fi密码", "items": [] },
  { "str": "date_time", "value": "日期时间", "items": [] },
  { "str": "language", "value": "语言", "items": [] },
  { "str": "language_array", "value": "", "items": [
    { "id": "cn", "title": "简体中文" },
    { "id": "en", "title": "English" },
    { "id": "tw", "title": "繁體中文" }
  ]},
  { "str": "format_card", "value": "格式化存储卡", "items": [] },
  { "str": "factory_reset", "value": "恢复出厂设置", "items": [] }
]
```

---

## 19. 风险与待确认项

以下内容需要在联调时通过**真实抓包**确认，当前按文档实现即可：

| 项目 | 当前状态 | 风险等级 |
|---|---|---|
| TCP 心跳设备端回复格式 | 待抓包确认 | 中 |
| `0x7d1` 完整字段集 | 最小字段已确认，可能有更多字段 | 低 |
| UDP 报文格式 | 未确认 | 低（P2 才需要） |
| `sunxi.db` 表结构 | 未确认 | 低（P2 才需要） |
| 部分设置项的枚举值 | 待联调对齐 | 低 |
| `connect_getXML` 原始格式 | 待确认 | 中 |
| TCP 除心跳外的完整报文类型 | 部分确认 | 中 |
| 翻译资源原始 XML 格式 | App 解析为 JSON 数组已确认，原始 XML 结构待确认 | 中 |

### 处理原则

- 先按本文档实现
- 联调时以真实抓包为准
- 发现不一致的地方，更新本文档版本号

---

## 20. 版本记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0.0 | 2026-05-22 | 初始版本，覆盖全部 4 条链路 + 21 个命令码 + 菜单资源 + 文件系统 |
