# QZ REST API — App 端开发规范

> **版本**：v1.0.0 | **日期**：2026-05-22 | **状态**：已定稿，可开工  
> **你的角色**：你是 Client 端，负责调用所有 API 接口  
> **对应文档**：嵌入式端看 [embedded/](../embedded/)

---

## 全部接口一览

| 方法 | 路径 | 功能 | 优先级 |
|---|---|---|---|
| GET | `/api/v1/device/info` | 设备信息 | P0 |
| GET | `/api/v1/device/storage` | 存储卡信息 | P0 |
| GET | `/api/v1/device/battery` | 电池信息 | P1 |
| GET | `/api/v1/device/check` | 健康检查 | P1 |
| GET | `/api/v1/camera/status` | 相机状态（录像、模式） | P0 |
| POST | `/api/v1/camera/record/start` | 开始录像 | P0 |
| POST | `/api/v1/camera/record/stop` | 停止录像 | P0 |
| POST | `/api/v1/camera/capture` | 拍照 | P0 |
| POST | `/api/v1/camera/mode` | 切换工作模式 | P1 |
| POST | `/api/v1/camera/playback/enter` | 进入回放 | P1 |
| POST | `/api/v1/camera/playback/exit` | 退出回放 | P1 |
| POST | `/api/v1/camera/zoom` | 变焦 | P2 |
| GET | `/api/v1/media/files` | 文件列表 | P1 |
| GET | `/thumb/<path>.jpg` | 缩略图（静态 URL） | P1 |
| GET | `/api/v1/media/file` | 在线查看/下载文件 | P0 |
| DELETE | `/api/v1/media/file` | 删除文件 | P1 |
| GET | `/api/v1/settings/menus` | 获取菜单（含翻译和当前值） | P1 |
| POST | `/api/v1/settings/menu/value` | 修改菜单选项 | P1 |
| POST | `/api/v1/settings/wifi` | 设置 Wi-Fi | P1 |
| POST | `/api/v1/settings/datetime` | 同步时间 | P1 |
| POST | `/api/v1/settings/format` | 格式化 SD 卡 | P2 |
| POST | `/api/v1/settings/reset` | 恢复出厂设置 | P2 |

---

## 目录

### 基础架构

| 文档 | 内容 | 优先级 |
|---|---|---|
| [连接识别与响应格式](./connection.md) | Wi-Fi 网关检测、统一响应解析、HTTP 封装 | P0 |
| [TCP 心跳客户端](./tcp-heartbeat.md) | 事件监听（18 种事件）、完整 Dart 实现、UI 层示例 | P0 |
| [RTSP 预览接入](./rtsp-preview.md) | fijkplayer / media_kit 接入示例、预览 vs 回放对比 | P0 |

### REST API 接口

| 文档 | 路由前缀 | 接口数 | 优先级 |
|---|---|---|---|
| [设备信息](./device-info.md) | `/api/v1/device/*` | 4 个 | P0-P1 |
| [相机控制](./camera-control.md) | `/api/v1/camera/*` | 8 个 | P0-P2 |
| [媒体文件](./media-files.md) | `/api/v1/media/*` + `/thumb/*` | 7 节（列表、缩略图、查看图片、播放视频、下载、删除、相册页示例） | P0-P1 |
| [设置](./settings.md) | `/api/v1/settings/*` | 6 个 + 设置页完整示例 | P1-P2 |

### 附录

| 文档 | 内容 |
|---|---|
| [工作模式定义](./work-modes.md) | 8 种模式枚举、Dart 枚举、模式切换处理 |
| [连接初始化流程](./init-flow.md) | 完整连接代码、时序图、页面与 API 对应关系 |
| [Mock 数据集](./mock-data.md) | json-server 搭建、全部 Mock JSON（设备没好之前用这个自测） |
| [错误码与处理](./error-handling.md) | 错误码表、统一处理、网络超时、推荐 Flutter 依赖 |
