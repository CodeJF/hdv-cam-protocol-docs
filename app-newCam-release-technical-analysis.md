# HDV CAM 技术分析文档

## 导航

### 全文目录

- [1. 文档目的](#1-文档目的)
- [2. 分析对象](#2-分析对象)
- [3. 结论摘要](#3-结论摘要)
- [4. 总体架构](#4-总体架构)
- [5. 页面结构](#5-页面结构)
- [6. 主要功能清单](#6-主要功能清单)
- [7. 功能实现方式](#7-功能实现方式)
- [8. 三套协议族对比](#8-三套协议族对比)
- [9. iOS 端对照分析](#9-ios-端对照分析)
- [10. 关键数据结构](#10-关键数据结构)
- [11. 本地存储与事件模型](#11-本地存储与事件模型)
- [12. 第三方库](#12-第三方库)
- [13. 权限分析](#13-权限分析)
- [14. 工程设计评价](#14-工程设计评价)
- [15. 深挖建议](#15-深挖建议)
- [16. 面向工程师的直接结论](#16-面向工程师的直接结论)
- [附：关键类速查](#附关键类速查)

### 协议族子文档

每套协议族有独立的详细文档，主文档只做横向对比和总览：

| 协议 | 子文档 | 适合场景 |
|---|---|---|
| QZ | [QZ 协议总览](./qz-protocol-overview.md) | 网络形态、协议分层、初始化链路 |
| QZ | [QZ API 合同草案](./qz-api-contract.md) | 命令码、请求模板、返回结构 |
| QZ | [QZ 媒体与菜单模型](./qz-media-model.md) | 文件列表、菜单 XML、翻译资源 |
| QZ | [QZ 复刻实施计划](./qz-replica-plan.md) | 排期、阶段目标、验收、团队分工 |
| MStar | [MStar 协议族](./mstar-protocol.md) | CGI 接口、属性树、请求模板 |
| YZ | [YZ 协议族](./yz-protocol.md) | REST 接口、返回结构、初始化链路 |

### 专项指南

| 文档 | 适合读者 |
|---|---|
| [嵌入式工程师指南](./qz-embedded-engineer-guide.md) | 固件/设备端开发 |
| [Flutter App 工程师指南](./qz-flutter-app-guide.md) | App 客户端开发 |

## 1. 文档目的

本文基于 `app-newCam-release.apk` 和 `HDV CAM 1.3.3.ipa` 的逆向分析整理。目标读者是 Android / iOS / 设备互联 / 音视频方向开发工程师。

本文回答：

- App 总体架构是什么
- 主要功能有哪些
- 三套设备协议各自长什么样
- iOS 端和 Android 端是否一致
- 复刻/抓包/二次开发应该从哪里下手

## 2. 分析对象

### Android

| 项目 | 值 |
|---|---|
| 文件 | `app-newCam-release.apk` |
| 包名 | `com.record.dv` |
| App 名称 | `HDV CAM` |
| 版本号 | `1.18.20260325` |
| minSdk / targetSdk | `24` / `35` |

### iOS

| 项目 | 值 |
|---|---|
| 文件 | `HDV CAM 1.3.3.ipa` |
| Bundle ID | `com.haijixing.hdv8k` |
| 版本 | `1.3.3`（Build `202605121451`） |
| 最低系统 | iOS 12.0 |
| 开发者 | 丹 罗 / wikico |

## 3. 结论摘要

这是一个"运动相机 / 行车记录仪 Wi-Fi 直连控制 App"，内置 **3 套设备协议族**：

| 协议 | 设备 IP | 风格 | 复杂度 |
|---|---|---|---|
| MStar | `192.168.1.1` | CGI 属性树 | 中 |
| QZ | `192.168.10.1` | 命令码 + XML + TCP/UDP | 高 |
| YZ | `192.168.169.1` | REST JSON | 低 |

App 根据手机当前连接的热点网关 IP 推断协议族，再切换到对应实现。

三套协议暴露的上层能力一致：

- 连接设备、实时预览、拍照/录像/锁定
- 电量/SD 卡/录制时长
- 参数菜单/设置/分辨率
- 媒体文件浏览/下载/删除
- Wi-Fi 改名改密、时间同步、重启
- 变焦/切换镜头

上层 UI 和 ViewModel 是统一的，差异封装在各协议族的 `Case` 子类和 `Pr` 协议访问类里。

**iOS 与 Android 在协议族划分、设备接入方式、主要功能域上完全一致**，已通过 IPA 中 ATS 例外域（`192.168.1.1` / `192.168.10.1` / `192.168.169.1`）和 QZ 专属页面资源双重确认。

## 4. 总体架构

### 4.1 分层结构

```
┌─────────────────────────────────────────┐
│  UI 层                                   │
│  MainActivity / ConFrgm / AlbLocalFrgm  │
│  CamConAtv / CamConQZAtv / CamConYZAtv  │
├─────────────────────────────────────────┤
│  ViewModel 层                            │
│  ConFrgmVmd / CamConAtvVmd / ...        │
├─────────────────────────────────────────┤
│  统一设备抽象层                           │
│  Case / CsFghj / CSManager              │
├─────────────────────────────────────────┤
│  协议实现层                              │
│  MSCs+MSPr / QZCs+QZPr / YZCs+YZPr     │
├─────────────────────────────────────────┤
│  基础设施层                              │
│  Wi-Fi / HTTP / IJK / ExoPlayer / Hawk  │
└─────────────────────────────────────────┘
```

### 4.2 统一设备抽象

核心是 `Case` 抽象层，统一维护：

- `cameraId` / `cameraNum` / `currentMode`
- `liveStreamId` / `supportWorkMode`
- `cameraResolutionMap`
- 事件观察者列表

上层页面不直接依赖具体协议，通过 `Case` 暴露的统一能力工作。

### 4.3 协议路由

`CSManager` 是协议切换中心：

| 设备 IP | 芯片枚举 | 实现类 |
|---|---|---|
| `192.168.1.1` | MStar | `com.record.MstCS.MSCs` |
| `192.168.10.1` | QZ | `com.record.QzCS.QZCs` |
| `192.168.169.1` | YZ | `com.record.YzCS.YZCs` |

App 面向的是三类固定热点型设备，不是云设备。

## 5. 页面结构

启动入口：`StartActivity` -> `MainActivity`

`MainActivity` 底部三块：

- 连接页 `ConFrgm`
- 本地相册页 `AlbLocalFrgm`
- 关于页 `AboutFrgm`

连接成功后进入对应控制页：`CamConAtv` / `CamConQZAtv` / `CamConYZAtv`。

## 6. 主要功能清单

- 设备 Wi-Fi 连接 / 扫码连接 / 手动连接
- 实时预览
- 拍照 / 录像开始停止 / 锁定视频 / 变焦
- 切换工作模式（普通录像/慢动作/循环/延时/普通拍照/自动/连拍/定时共 8 种）
- 获取电量/SD 卡/录制时长/分辨率/参数菜单
- 设置参数值 / Wi-Fi 名称密码 / 时间同步 / 重启设备
- 浏览设备端媒体 / 下载 / 删除 / 上传
- 播放视频/图片预览 / 本地相册管理 / 收藏

## 7. 功能实现方式

### 7.1 设备连接

`ConFrgm` -> `ConFrgmVmd`：检查 Wi-Fi -> 判断热点 IP -> `CSManager.set2CsInt()` 切协议 -> `dvConnect` -> 跳转控制页。

依赖"手机先连设备热点"的工作流，不是局域网扫描自动发现。

### 7.2 实时预览

IJK Player 为主：`GeneralIJKLiveStreamPlayer` / `IjkPlayerView`。

协议层返回 RTSP 流地址，交给 IJK 播放。三套协议都使用 RTSP。ExoPlayer 用于本地/回放场景。

### 7.3 拍照/录像/锁定

统一方法 `capture` / `record` / `lock`，封装在各 `Cs` 类中。ViewModel 调统一语义，实际命令由协议层处理。

### 7.4 参数菜单和设置

- `getMenuList` / `getMenuValues` / `getParameterValue` / `setParameterValue`
- QZ 协议还包含 `QZMenu` / `QZMenuItem` / `QZTranslation`，使用 Jsoup 解析 XML 资源

### 7.5 相册/文件管理

统一暴露 `getMediaFiles` / `getFileDb` / `uploadCamFile` / `deleteFile`。

设备媒体统一落到 `MediaJsdsd` 对象，上层相册页围绕此对象工作。

## 8. 三套协议族对比

### 8.1 横向对比表

| 维度 | MStar | QZ | YZ |
|---|---|---|---|
| 设备 IP | `192.168.1.1` | `192.168.10.1` | `192.168.169.1` |
| HTTP 入口 | `/cgi-bin/Config.cgi?` | `:8082/api/` | `/app/<cmd>?` |
| 请求风格 | CGI query | 命令码 `cmd=0x???` | REST `?param=&value=` |
| 响应格式 | 属性树/CGI | JSON + XML 混合 | 统一 JSON 外壳 |
| RTSP | `rtsp://.../liveRTSP/av` | `rtsp://...:8554/ch00` | 动态获取 |
| 菜单系统 | `Camera.Menu.*` 属性 | XML + 翻译资源 + 命令码 | `getparamvalue/items` |
| 媒体索引 | 下载 `DCF.db` | `action=dir` 或 `sunxi.db` | `getfilelist` JSON |
| 保活/事件 | UDP | TCP(9999) + UDP(49142) | 无明显证据 |
| 已确认命令数 | ~17 | **16** | ~30 |
| 复杂度 | 中 | **高** | **低** |

### 8.2 QZ 协议族

详见：[QZ 协议总览](./qz-protocol-overview.md) / [QZ API 合同](./qz-api-contract.md) / [QZ 媒体模型](./qz-media-model.md) / [QZ 复刻计划](./qz-replica-plan.md)

核心类：`QZCs` / `QZPr` / `CaseEventManagerQZ`

特点：

- 4 条链路并存：HTTP(8082) + RTSP(8554) + TCP(9999) + UDP(49142)
- 命令码驱动，16 个已确认命令（8 个读 + 8 个写控制）
- 菜单系统是 XML + 翻译资源 + 当前值三件套
- 8 种工作模式，每种关联独立的 setting_keys
- TCP 心跳（`S:100.0`，500ms）是连接成功的前提

### 8.3 MStar 协议族

详见：[MStar 协议族](./mstar-protocol.md)

核心类：`MSCs` / `MSPr` / `CaseEventManager`

特点：

- CGI 风格 `http://<ip>/cgi-bin/Config.cgi?<action>`
- 属性树访问（`Camera.Menu.*`、`Net.WIFI_AP.*`）
- 媒体索引靠下载 `DCF.db` 本地解析
- 存在 UDP 端口监听

### 8.4 YZ 协议族

详见：[YZ 协议族](./yz-protocol.md)

核心类：`YZCs` / `YZPr`

特点：

- 最接近标准 REST API：`http://<ip>/app/<cmd>?k=v`
- 统一 JSON 外壳 `{ "result": 0, "info": {...} }`
- 接口最规则、参数名最可读
- RTSP 地址通过 `getmediainfo` 动态获取

### 8.5 可开发性排序

1. **YZ** - 接口最规则，返回结构最接近 JSON API，适合优先复刻/抓包
2. **MStar** - CGI 风格清晰，但属性树和 DCF.db 增加复杂度
3. **QZ** - 混合 XML/命令码/TCP/UDP/数据库，工程成本最高

## 9. iOS 端对照分析

### 9.1 一致性结论

iOS 端（`HDV CAM 1.3.3`，Bundle ID `com.haijixing.hdv8k`）与 Android 端：

**一致的部分：**

- 协议族划分（MStar / QZ / YZ）
- 内网热点型接入方式
- 主要功能域
- RTSP + UDP + HTTP 混合传输模式
- 设备设置/相册/下载/删除/格式化业务流

**不同的部分：**

| 维度 | Android | iOS |
|---|---|---|
| UI 技术栈 | Kotlin + AndroidX | UIKit + nib/storyboard |
| 播放链路 | IJKPlayer + FFmpeg | AVFoundation / AVKit / VideoToolbox |
| 本地数据层 | Hawk KV | FMDB (SQLite) |
| 图片加载 | Glide | SDWebImage |
| JSON 映射 | Gson | MJExtension |

### 9.2 iOS 协议层线索

`func.list` 暴露了关键协议方法：

- `v536_Cam_PostRequestWithUrl` / `v536_Cam_GetRequestWithUrl` - HTTP 请求层
- `connect_getXML` / `send_3008` / `getCurrSetting_2002_2006` - QZ 命令码
- `changeLiveStreamWithParamStr` / `handle_udp_rtsp` - 直播流
- `startUDP` / `keepConnectActon` / `sendPingWithData` - 保活/事件
- `refreshMstartRtspWithSuccess` - MStar RTSP 刷新

iOS 端同样是"协议层封装 + 页面层调用"，不是把业务写死在页面里。

### 9.3 QZ 专属页面

iOS 包内有 QZ 独立 nib 资源：

- `HJ09X04QZ_AlbumController` / `HJ09X04QZ_SettingCell` / `HJ09X04QZ_SettingDetailCell` / `HJ09X04QZ_SettingHeadView` / `HJ09X04QZ_CamProgressView`

说明 QZ 在 iOS 端是一级支持方案，有专属相册页、设置页和控制视图。

### 9.4 额外发现

- `_ftp._tcp` Bonjour 声明：iOS 端预留了 FTP 发现能力
- 包内含 OCR/车牌/车辆识别模型：暗示兼容行车记录仪/双录产品
- iOS 文案覆盖 Dash Camera 和 Motion Camera 两类产品

### 9.5 对开发的实际意义

- 设备协议层：以 Android 侧整理的协议文档为准
- iOS 客户端架构：不需要强行模仿原包结构
- IPA 更多用于确认产品范围、页面范围和实现方向

## 10. 关键数据结构

### 10.1 通用媒体对象 `MediaJsdsd`

| 字段 | 类型 | 说明 |
|---|---|---|
| `ctime` | long | 创建时间 |
| `duration` | int | 时长 |
| `emr` | boolean | 紧急/锁定标记 |
| `fileName` | String | 文件名 |
| `name` | String | 展示名 |
| `path` | String | 原始路径 |
| `pathPlayOrdown` | String | 播放/下载路径 |
| `size` | long | 文件大小 |
| `thumbUrl` | String | 缩略图地址 |
| `type` | int | 媒体类型 |

只要设备端能产出 `MediaJsdsd` 列表，上层相册 UI 可直接复用。

### 10.2 分辨率对象 `RsITJfs`

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | String | 名称 |
| `index` | String | 内部索引 |
| `size` | String | 尺寸 |
| `rate` | String | 帧率/码率 |

### 10.3 工作模式枚举 `CsWmJk`

UI 模式：

- `CASE_MODE_VIDEO` / `VIDEO_SLOW` / `VIDEO_LOOP` / `VIDEO_TIMELAPSE`
- `CASE_MODE_PHOTO` / `PHOTO_TIMER` / `PHOTO_BLURT` / `PHOTO_AUTO`
- `CASE_MODE_SETUP` / `CASE_MODE_STORAGE`

设备内部编号：`MODE0_VIDEO` ~ `MODE7_PHOTO_TIMER`

App 已统一抽象"UI 模式"和"设备底层模式号"的映射。

## 11. 本地存储与事件模型

使用 `Hawk` 作为轻量 KV 持久化，已确认项：`privacy_agreed` / `appVersionCode` / `colectList`。

事件驱动：`AppEvent` / `AlbumEventManager` / `Case` 观察者列表，用于设备状态刷新、下载完成通知、相册刷新、页面联动。

## 12. 第三方库

### Android

| 库 | 用途 |
|---|---|
| Kotlin + AndroidX | 基础框架 |
| IJKPlayer + FFmpeg | 实时预览 |
| ExoPlayer | 本地/回放播放 |
| Glide + OkHttp | 图片加载 |
| Hawk | KV 存储 |
| Huawei ScanKit | 扫码 |
| Tencent Bugly | 崩溃监控 |
| uCrop | 图片裁剪 |
| FileDownloader | 文件下载 |

Native 库：`libijkplayer.so` / `libijkffmpeg.so` / `libAVAPIs.so` / `libIOTCAPIs.so` / `libipcamera.so` / `libaw_net_client.so`

`libIOTCAPIs.so` / `libAVAPIs.so` 意味着集成过 P2P 音视频能力，但主业务仍以本地 Wi-Fi 热点控制为核心。

### iOS

| 库 | 用途 |
|---|---|
| UIKit + nib | UI 框架 |
| AVFoundation / VideoToolbox | 播放/解码 |
| FMDB | SQLite |
| SDWebImage | 图片加载 |
| MJExtension | JSON/Model 映射 |
| IQKeyboardManager | 键盘管理 |
| ScanKitFrameWork | 扫码 |

## 13. 权限分析

### Android

- Wi-Fi/网络：连接热点、监听变化
- 位置：Wi-Fi 扫描兼容要求
- 存储：下载文件、本地相册
- 相机：扫码连接
- 前台服务：后台下载

### iOS

- 本地网络：直连设备
- 定位：读取当前 Wi-Fi SSID
- 照片：保存到系统相册
- 相机：扫码

## 14. 工程设计评价

**优点：**

- 协议和 UI 分层清楚
- 通过 `Case` 抽象统一上层调用
- 多协议共存但页面能力尽量统一
- 设备媒体/模式/分辨率有统一对象

**风险点：**

- 命名混淆严重，可维护性差
- 三套协议耦合在同一个包内
- 设备协议大量硬编码 IP 和命令
- QZ 混合 HTTP + TCP + UDP + RTSP + XML，排障复杂

## 15. 深挖建议

### 想复刻设备协议

优先顺序：YZ > MStar > QZ。原因：YZ 接口最直白，MStar 的 CGI 次之，QZ 混合链路最复杂。

### 想复刻 App 主要功能

优先入口：

- 连接：`ConFrgmVmd`
- 协议切换：`CSManager`
- 抽象能力：`Case`
- 预览：`GeneralIJKLiveStreamPlayer`
- 参数页：`CamSetFrgmVmd`
- 相册：`AlbCamFrgmVmd` / `AlbCamPageFrgmVmd`

### 想抓包

1. 手机连设备热点
2. 针对设备网段做 HTTP 抓包
3. 观察 RTSP 建连
4. 同步抓 UDP/TCP 流量
5. 重点验证：预览、录像、拍照、文件列表、下载、改 Wi-Fi

## 16. 面向工程师的直接结论

1. 这不是单协议 App，而是 **多协议设备壳**
2. 上层复用的关键抽象是 `Case`，不要直接从 Activity 下手
3. 设备识别靠当前连接热点 IP，不靠云端
4. 实时预览主通路是 `RTSP + IJKPlayer`（Android）/ `RTSP + AVFoundation`（iOS）
5. 文件与设置控制主通路是本地 HTTP
6. `YZ` 最适合作为第一批复刻或抓包样本
7. 新增设备支持的最合理方式是新增一个 `Case + Pr` 协议族
8. iOS 端和 Android 端协议层完全一致，开发协议层文档可共用

---

## 附：关键类速查

### App 和入口

- `com.record.dv.apC.AllContext`
- `com.record.dv.MainActivity`
- `com.record.dv.atv.StartActivity`

### 连接与控制

- `com.record.dv.fragment.ConFrgm`
- `com.record.dv.viewmodel.ConFrgmVmd`
- `com.record.dv.viewmodel.CamConAtvVmd`
- `com.record.dv.fragment.CamSetFrgm`

### 协议抽象

- `com.record.dvCS.Case`
- `com.record.dvCS.CsFghj`
- `com.record.dvCS.CsWmJk`
- `com.record.dvCSManager.CSManager`

### 协议实现

- `com.record.MstCS.MSCs` / `com.record.MstCS.MSPr`
- `com.record.QzCS.QZCs` / `com.record.QzCS.QZPr`
- `com.record.YzCS.YZCs` / `com.record.YzCS.YZPr`

### 媒体和菜单

- `com.record.dvCS.dvcsBean.MediaJsdsd`
- `com.record.dvCS.dvcsBean.RsITJfs`
- `com.record.QzCS.QZMenu` / `QZMenuItem` / `QZTranslation`

### 播放器

- `com.record.dvCS.dvcsPer.ijk.GeneralIJKLiveStreamPlayer`
- `com.record.dvCS.dvcsPer.ijk.IjkPlayerView`

### iOS 关键方法名

- `v536_Cam_PostRequestWithUrl` / `v536_Cam_GetRequestWithUrl`
- `connect_getXML` / `send_3008` / `getCurrSetting_2002_2006`
- `changeLiveStreamWithParamStr` / `handle_udp_rtsp`
- `startUDP` / `keepConnectActon`
- `pushToMSCameraController` / `pushToPreviewViewController`
