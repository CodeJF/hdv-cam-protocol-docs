# QZ 协议总览

## 导航

### 主入口

- [主入口：app-newCam-release-technical-analysis.md](./app-newCam-release-technical-analysis.md)

### 兄弟文档

- [QZ API 合同草案](./qz-api-contract.md)
- [QZ 媒体与菜单模型](./qz-media-model.md)
- [QZ 复刻实施计划](./qz-replica-plan.md)
- [MStar 协议族](./mstar-protocol.md)
- [YZ 协议族](./yz-protocol.md)

### 本页定位

- 适合先看全貌
- 适合确认网络形态、协议分层、初始化链路
- 不负责命令码和字段级合同细节

## 1. 目标

本文只回答 4 个问题：

- `QZ` 设备在网络上长什么样
- App 如何识别并接入 `QZ`
- `QZ` 的实时链路由哪些协议组成
- 复刻端最先应该做哪些能力

## 2. 基础画像

| 项目 | 值 |
|---|---|
| 设备默认地址 | `192.168.10.1` |
| HTTP 控制 | `http://192.168.10.1:8082` |
| RTSP 预览 | `rtsp://192.168.10.1:8554/ch00` |
| TCP 事件 | `192.168.10.1:9999` |
| UDP 辅助 | `192.168.10.1:49142` |
| App 识别依据 | 手机当前连接的热点网关 IP |

结论：

- 要复刻 `QZ`，设备地址必须做成 `192.168.10.1`
- 不建议先做成其它 `192.168.10.x`

## 3. 协议分层

`QZ` 不是单一 HTTP 协议，而是 **4 条链路**并存：

### 3.1 HTTP（端口 8082）

- 控制命令（拍照、录像、设置）
- 状态读取（SD 卡、录像状态、工作模式、电量）
- 菜单/翻译/XML 资源
- 文件列表和数据库下载

### 3.2 RTSP（端口 8554）

- 实时预览主链路
- 固定入口：`rtsp://192.168.10.1:8554/ch00`

### 3.3 TCP Socket（端口 9999）

- **App 连接后立即建立**，是 P0 依赖
- 每 500ms 发送 `S:100.0` 作为心跳保活
- 连接超时 5000ms，接收缓冲区 1024 字节
- 事件类型：`CameraEventMessage:connect/break/Exception/clean`
- 如果设备不监听 9999，App 可能判定连接失败

### 3.4 UDP（端口 49142）

- `DatagramSocket` 监听端口 `49142`（0xBFF6）
- **逆向确认**：App 端是**纯接收通道**（`DSocket.receive(packet)`），App 不通过此端口向设备发数据
- 设备向 `192.168.10.255:49142` 广播，用于设备发现 + 多机状态同步
- `sendData()` 走的是 TCP `outStream.write()`，不是 UDP
- I 帧刷新不走此端口，由 Native 层（`libijkplayer.so` / FFmpeg）通过 RTCP PLI（RFC 4585）在 RTSP 协商端口自动处理

## 4. App 侧关键实现线索

### Android 端

核心类：

- `com.record.QzCS.QZCs` - 命令发送、状态管理、模式切换
- `com.record.QzCS.QZPr` - HTTP URL 构造、所有请求模板
- `com.record.QzCS.CaseEventManagerQZ` - TCP/UDP 事件管理

### iOS 端

关键方法名：

- `v536_Cam_PostRequestWithUrl` / `v536_Cam_GetRequestWithUrl` - HTTP 请求
- `connect_getXML` - 菜单 XML 获取
- `send_3008` / `getCurrSetting_2002` / `getCurrSetting_2002_2006` - 命令码操作
- `changeLiveStreamWithParamStr` - 直播流控制
- `handle_udp_rtsp` / `startUDP` - UDP 链路（实际为设备发现广播接收，非 RTSP 控制）
- `keepConnectActon` / `startTimer_keepConnect` - TCP 保活

两端都是"协议层封装 + 页面层调用"，不是把业务写死在页面里。

## 5. 初始化链路

App 接入 QZ 设备的完整顺序：

1. 手机连到设备热点
2. App 判断当前热点网关为 `192.168.10.1`
3. `CSManager` 切换到 `QZCs` 协议实现
4. **建立 TCP Socket 到 9999**，开始 500ms 心跳
5. 读取基础状态：
   - `0x7d1` - 设备基础信息
   - `0x7d4` - SD 卡信息
   - `0x7d5` - 录像状态
   - `0x7d8` - 当前工作模式
   - `0x7d9` - 电池信息
   - `0xbcd` - 设备检查
6. 构造 RTSP 地址：`rtsp://192.168.10.1:8554/ch00`
7. 初始化设置/相册：
   - 下载 `setting_keys.xml`
   - 下载翻译资源 `zh-XX.xml`
   - 读取菜单当前值 `0x7d2` / `0x7d6`
   - 文件列表 `action=dir` 或下载 `sunxi.db`

## 6. 工作模式

QZ 定义了 8 种工作模式：

| 模式名 | 内部编号 | setting_keys |
|---|---|---|
| `NormalRecordeMode` | 0 | `record_normal_setting_keys` |
| `SlowRecordeMode` | 1 | `record_slow_setting_keys` |
| `LoopRecordeMode` | 2 | `record_loop_setting_keys` |
| `TimeLapseMode` | 3 | `record_timelapse_setting_keys` |
| `NormalCaptureMode` | 4 | `photo_normal_setting_keys` |
| `AutoCaptureMode` | 5 | `photo_auto_setting_keys` |
| `ContinueCaptureMode` | 6 | `photo_continue_setting_keys` |
| `TimingCaptureMode` | 7 | `photo_time_setting_keys` |

每种模式对应独立的 `Camera.Menu.<Type>Res` 分辨率键名和 `setting_keys` XML 数组。

## 7. 复刻最小能力

### P0 最小可演示

1. 热点与网关 `192.168.10.1`
2. HTTP 服务 `:8082`（`GET /api/getdeviceinfo` + `POST /api/setdeviceinfo`）
3. **TCP 监听 `:9999`**，接受心跳 `S:100.0`
4. RTSP 预览 `:8554/ch00`
5. 基础状态（`0x7d1` / `0x7d4` / `0x7d5` / `0x7d8`）
6. 核心控制（拍照 `0x44d`、录像 `0x44c`、回放 `0xbd9`）

### P1 正式设备

- `action=dir` 文件列表
- 缩略图
- `setting_keys.xml` + 翻译资源
- `0x7d2` / `0x7d6` 菜单当前值
- Wi-Fi / 时间设置

### P2 高兼容

- `sunxi.db`
- UDP 设备发现广播 + 多机状态同步
- 完整枚举值对齐

## 8. 复刻优先顺序

1. `8082` HTTP 骨架
2. `9999` TCP 心跳监听
3. `0x7d1` / `0x7d4` / `0x7d5` / `0x7d8`
4. `8554/ch00` RTSP
5. `0x44c` / `0x44d` / `0xbd9`
6. `action=dir`
7. `setting_keys.xml`
8. `0x7d2` / `0x7d6`
9. `sunxi.db`
10. UDP 设备发现广播 + 多机状态同步

## 9. 当前确定与不确定

### 已高置信确定

- 地址、端口、RTSP 入口
- TCP 事件通道端口 9999 和心跳协议 `S:100.0`
- UDP 端口 49142
- 命令码驱动模式
- 菜单 XML 依赖
- 8 种工作模式完整枚举
- `sunxi.db` 路径：`/tmp/data/.data/sqlite/sunxi.db`

### 仍需联调/抓包确认

- UDP 广播报文字段格式
- `sunxi.db` 完整表结构
- 部分设置项的最终枚举语义
- `connect_getXML` 的完整原始文件格式
- TCP 事件通道除心跳外的完整报文类型
