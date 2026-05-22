# 运动相机项目 — 协议文档

## 仓库结构

```
├── api-spec/               ← 我们的接口协议（开发对接用这个）
│   ├── qz-api-embedded.md      嵌入式工程师看这个
│   └── qz-api-app.md           App 工程师看这个
│
├── hdv-cam-reference/      ← HDV CAM 逆向分析（参考资料，不直接用）
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
└── README.md               ← 你正在看的这个
```

---

## 开工看这里

### 嵌入式工程师

你是 Server 端，实现所有 REST API 接口。

**看这个文档：** [api-spec/qz-api-embedded.md](./api-spec/qz-api-embedded.md)

包含：
- 全部 22 个 REST API 接口的请求/响应格式
- TCP 心跳服务实现
- RTSP 预览服务要求
- curl 自测命令 + 一键验证脚本
- 错误码表

### App 工程师

你是 Client 端，调用所有 REST API 接口。

**看这个文档：** [api-spec/qz-api-app.md](./api-spec/qz-api-app.md)

包含：
- 全部 22 个 REST API 的调用方式
- 每个接口的 Dart 数据模型和调用示例
- 相册浏览、查看图片、下载视频的完整流程
- TCP 心跳客户端实现
- RTSP 预览接入（fijkplayer / media_kit）
- 完整页面代码示例（相册页、设置页）
- Mock 数据集（设备没好之前用这个自测）

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
| GET | `/api/v1/media/files` | 文件列表 |
| GET | `/api/v1/media/thumbnail` | 缩略图 |
| GET | `/api/v1/media/file` | 下载文件 |
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
