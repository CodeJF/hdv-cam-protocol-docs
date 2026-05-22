# QZ REST API — 嵌入式端实现规范

> **版本**：v1.0.0 | **日期**：2026-05-22 | **状态**：已定稿，可开工  
> **你的角色**：你是 Server 端，负责实现所有 API 接口  
> **对应文档**：App 端看 [app/](../app/)

---

## 目录

### 基础架构

| 文档 | 内容 | 优先级 |
|---|---|---|
| [网络架构与响应格式](./architecture.md) | 网络拓扑、固定参数、统一 JSON 响应格式 | P0 |
| [TCP 心跳服务](./tcp-heartbeat.md) | 端口 9999，心跳协议，事件推送（18 种事件） | P0 |
| [RTSP 预览服务](./rtsp-preview.md) | 端口 8554，H.264 实时视频流 | P0 |

### REST API 接口

| 文档 | 路由前缀 | 接口数 | 优先级 |
|---|---|---|---|
| [设备信息](./device-info.md) | `/api/v1/device/*` | 4 个 | P0-P1 |
| [相机控制](./camera-control.md) | `/api/v1/camera/*` | 8 个 | P0-P2 |
| [媒体文件](./media-files.md) | `/api/v1/media/*` + `/thumb/*` | 4 个 | P0-P1 |
| [设置](./settings.md) | `/api/v1/settings/*` | 6 个 | P1-P2 |

### 附录

| 文档 | 内容 |
|---|---|
| [工作模式定义](./work-modes.md) | 8 种工作模式、文件存储目录、命名规范 |
| [启动顺序](./boot-sequence.md) | 设备上电后的服务启动时序 |
| [curl 自测脚本](./test-script.md) | 一键验证所有 P0 接口的 bash 脚本 |
| [错误码表](./error-codes.md) | 全部错误码定义 |
