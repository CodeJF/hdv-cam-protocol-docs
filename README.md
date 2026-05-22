# 运动相机项目 — 协议文档

## 仓库结构

```
├── api-spec/                  ← 我们的接口协议（开发对接用这个）
│   ├── embedded/                  嵌入式工程师看这个目录
│   │   ├── README.md                  导航目录
│   │   ├── architecture.md            网络架构 + 响应格式
│   │   ├── tcp-heartbeat.md           TCP 心跳服务（18 种事件推送）
│   │   ├── rtsp-preview.md            RTSP 预览服务
│   │   ├── device-info.md             设备信息接口（4 个）
│   │   ├── camera-control.md          相机控制接口（8 个）
│   │   ├── media-files.md             媒体文件接口（列表、缩略图、在线播放、删除）
│   │   ├── settings.md                设置接口（6 个）
│   │   ├── work-modes.md              工作模式 + 文件存储 + 命名
│   │   ├── boot-sequence.md           启动顺序
│   │   ├── test-script.md             curl 自测脚本
│   │   └── error-codes.md             错误码表
│   │
│   └── app/                       App 工程师看这个目录
│       ├── README.md                  导航目录 + 接口一览
│       ├── connection.md              连接识别 + 响应格式 + HTTP 封装
│       ├── tcp-heartbeat.md           TCP 心跳客户端（Dart 实现 + UI 示例）
│       ├── rtsp-preview.md            RTSP 预览（fijkplayer / media_kit）
│       ├── device-info.md             设备信息接口（4 个）
│       ├── camera-control.md          相机控制接口（8 个）
│       ├── media-files.md             媒体文件（在线播放 + 下载 + 相册页示例）
│       ├── settings.md                设置接口 + 设置页示例
│       ├── work-modes.md              工作模式定义（Dart 枚举）
│       ├── init-flow.md               初始化流程 + 页面对应
│       ├── mock-data.md               Mock 数据集
│       └── error-handling.md          错误码与处理
│
├── hdv-cam-reference/         ← HDV CAM 逆向分析（参考资料，不直接用）
│   ├── app-newCam-release-technical-analysis.md
│   ├── qz-protocol-overview.md
│   ├── qz-api-contract.md
│   ├── qz-media-model.md
│   ├── qz-replica-plan.md
│   ├── qz-interface-specification.md
│   ├── qz-embedded-engineer-guide.md
│   ├── qz-flutter-app-guide.md
│   ├── mstar-protocol.md
│   └── yz-protocol.md
│
└── README.md                  ← 你正在看的这个
```

---

## 开工看这里

### 嵌入式工程师

你是 Server 端，实现所有 REST API 接口。

**入口：** [api-spec/embedded/README.md](./api-spec/embedded/README.md)

按优先级建议的阅读顺序：
1. [网络架构与响应格式](./api-spec/embedded/architecture.md) — 先了解整体架构
2. [TCP 心跳服务](./api-spec/embedded/tcp-heartbeat.md) — P0，App 靠这个判断设备在线
3. [RTSP 预览服务](./api-spec/embedded/rtsp-preview.md) — P0，实时视频流
4. [设备信息](./api-spec/embedded/device-info.md) + [相机控制](./api-spec/embedded/camera-control.md) — 核心接口
5. [媒体文件](./api-spec/embedded/media-files.md) — 文件列表、缩略图、在线播放（Range 请求）
6. [设置](./api-spec/embedded/settings.md) — 菜单、Wi-Fi、时间同步
7. [curl 自测脚本](./api-spec/embedded/test-script.md) — 开发完跑一遍验证

### App 工程师

你是 Client 端，调用所有 REST API 接口。

**入口：** [api-spec/app/README.md](./api-spec/app/README.md)

按优先级建议的阅读顺序：
1. [连接识别与响应格式](./api-spec/app/connection.md) — HTTP 封装、响应解析
2. [TCP 心跳客户端](./api-spec/app/tcp-heartbeat.md) — P0，事件监听
3. [RTSP 预览接入](./api-spec/app/rtsp-preview.md) — P0，实时预览
4. [设备信息](./api-spec/app/device-info.md) + [相机控制](./api-spec/app/camera-control.md) — 核心接口
5. [媒体文件](./api-spec/app/media-files.md) — 相册、在线播放视频、下载
6. [设置](./api-spec/app/settings.md) — 菜单渲染、设置页示例
7. [Mock 数据集](./api-spec/app/mock-data.md) — 设备没好之前用这个自测

---

## 接口协议概览

| 服务 | 端口 | 说明 |
|---|---|---|
| TCP 心跳 | `9999` | App 连上后每 500ms 发心跳，设备推送事件 |
| HTTP REST API | `8080` | 全部控制/查询/设置接口，统一 JSON |
| RTSP 预览 | `8554` | 实时视频流 `rtsp://192.168.10.1:8554/ch00` |

| 方法 | 接口 | 功能 |
|---|---|---|
| GET | `/api/v1/device/info` | 设备信息 |
| GET | `/api/v1/device/storage` | 存储卡 |
| GET | `/api/v1/device/battery` | 电池 |
| GET | `/api/v1/camera/status` | 录像状态 + 模式 |
| POST | `/api/v1/camera/record/start` | 开始录像 |
| POST | `/api/v1/camera/record/stop` | 停止录像 |
| POST | `/api/v1/camera/capture` | 拍照 |
| POST | `/api/v1/camera/mode` | 切换模式 |
| POST | `/api/v1/camera/playback/enter` | 进入回放 |
| POST | `/api/v1/camera/playback/exit` | 退出回放 |
| GET | `/api/v1/media/files` | 文件列表 |
| GET | `/thumb/<path>.jpg` | 缩略图 |
| GET | `/api/v1/media/file` | 在线查看/下载文件 |
| DELETE | `/api/v1/media/file` | 删除文件 |
| GET | `/api/v1/settings/menus` | 菜单设置 |
| POST | `/api/v1/settings/wifi` | Wi-Fi 设置 |
| POST | `/api/v1/settings/datetime` | 时间同步 |

---

## HDV CAM 参考资料

`hdv-cam-reference/` 文件夹是对 HDV CAM App（Android + iOS）的逆向分析文档。我们的接口协议参考了这些分析结果，但做了以下优化：

| 对比项 | HDV CAM 原协议 | 我们的新协议 |
|---|---|---|
| 接口风格 | `cmd=0x7d1` 命令码 | REST API `/api/v1/device/info` |
| 数据格式 | JSON + XML 混合 | 全部 JSON |
| 文件列表 | XML 返回 | JSON 返回 |
| 菜单系统 | 3 次请求拼装 | 1 次请求全部返回 |
| HTTP 端口 | 8082 | 8080 |
| 响应格式 | 无统一外壳 | 统一 `{code, msg, data}` |

如果需要了解原协议细节或对比参考：

| 文档 | 说明 |
|---|---|
| [主分析文档](./hdv-cam-reference/app-newCam-release-technical-analysis.md) | App 架构、三套协议对比 |
| [QZ 协议总览](./hdv-cam-reference/qz-protocol-overview.md) | 4 条链路、初始化流程 |
| [QZ 原始接口](./hdv-cam-reference/qz-interface-specification.md) | 原命令码风格的完整接口定义 |
| [QZ 媒体模型](./hdv-cam-reference/qz-media-model.md) | 文件列表、菜单 XML、工作模式 |
| [QZ 复刻计划](./hdv-cam-reference/qz-replica-plan.md) | 阶段目标、排期、任务拆分 |
