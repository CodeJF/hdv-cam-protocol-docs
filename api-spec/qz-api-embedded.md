# QZ REST API — 嵌入式端实现规范

> **版本**：v1.0.0 | **日期**：2026-05-22 | **状态**：已定稿，可开工  
> **你的角色**：你是 Server 端，负责实现所有 API 接口  
> **对应文档**：App 端看 [qz-api-app.md](./qz-api-app.md)

---

## 目录

- [1. 总览](#1-总览)
- [2. 网络与服务架构](#2-网络与服务架构)
- [3. 统一响应格式](#3-统一响应格式)
- [4. TCP 心跳服务（端口 9999）](#4-tcp-心跳服务端口-9999)
- [5. RTSP 预览服务（端口 8554）](#5-rtsp-预览服务端口-8554)
- [6. REST API — 设备信息](#6-rest-api--设备信息)
- [7. REST API — 相机控制](#7-rest-api--相机控制)
- [8. REST API — 媒体文件](#8-rest-api--媒体文件)
- [9. REST API — 设置](#9-rest-api--设置)
- [10. 工作模式定义](#10-工作模式定义)
- [11. 启动顺序](#11-启动顺序)
- [12. curl 自测命令集](#12-curl-自测命令集)
- [13. 错误码表](#13-错误码表)

---

## 1. 总览

### 你要实现什么

| 服务 | 端口 | 你要做的 |
|---|---|---|
| Wi-Fi AP | — | 建立热点，网关 `192.168.10.1` |
| TCP 心跳 | `9999` | 监听 TCP 连接，接收心跳，推送事件 |
| HTTP REST API | `8080` | 实现全部 REST 接口（本文档重点） |
| RTSP 预览 | `8554` | 提供实时视频流 |

### 技术选型建议

| 组件 | 推荐方案 |
|---|---|
| HTTP Server | `mongoose` / `libmicrohttpd` / `lighttpd` |
| JSON 生成 | `cJSON` / `jansson` |
| RTSP Server | `live555` / 自研 |
| TCP Server | 标准 `socket` + `select`/`epoll` |

---

## 2. 网络与服务架构

### 2.1 网络拓扑

```
┌──────────────────────┐         ┌──────────────┐
│     嵌入式相机（你）    │◄─WiFi──►│   手机 App   │
│   AP: 192.168.10.1    │  STA    │ 192.168.10.x │
│                        │         │              │
│  ┌─ TCP  :9999  心跳   │         │              │
│  ├─ HTTP :8080  REST   │         │              │
│  ├─ RTSP :8554  预览   │         │              │
│  └─ UDP  :49142 辅助   │         │              │
└──────────────────────┘         └──────────────┘
```

### 2.2 固定参数

| 参数 | 值 | 备注 |
|---|---|---|
| 设备 IP | `192.168.10.1` | 硬编码，不可更改 |
| DHCP 范围 | `192.168.10.100 ~ 200` | 建议值 |

---

## 3. 统一响应格式

**所有 REST API 都必须返回这个格式，无例外。**

### 成功

```json
{
  "code": 0,
  "msg": "ok",
  "data": { ... }
}
```

### 失败

```json
{
  "code": -1,
  "msg": "sd card not found",
  "data": null
}
```

### 规范

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | `number` | `0` 成功，非 `0` 失败 |
| `msg` | `string` | 成功时 `"ok"`，失败时为错误描述 |
| `data` | `object / array / null` | 业务数据，无数据时为 `null` |

**注意**：
- HTTP 状态码统一返回 `200`，用 `code` 区分业务成功/失败
- Content-Type 统一 `application/json; charset=utf-8`
- 所有数值用 `number` 类型，不要用字符串包数字

---

## 4. TCP 心跳服务（端口 9999）

**优先级：P0 — 不实现此服务，App 会判定设备离线**

### 4.1 你要做的

1. 在端口 `9999` 监听 TCP 连接
2. App 连上后每 `500ms` 发送 `S:100.0`（UTF-8 字符串，7 字节）
3. 你收到心跳后回复 `OK\n`（建议）
4. 维持连接不主动断开
5. 设备状态变化时，通过此连接推送事件

### 4.2 心跳协议

| 方向 | 内容 | 频率 |
|---|---|---|
| App → 设备 | `S:100.0` | 每 500ms |
| 设备 → App | `OK\n` | 收到心跳后回复 |

### 4.3 事件推送格式

**格式规则：每条事件为一行 JSON，以 `\n` 结尾。**

```
{"event":"<事件名>", ...附加字段}\n
```

- 一行一条事件，不要跨行
- 必须包含 `event` 字段
- 附加字段根据事件类型不同而不同
- `\n` 结尾（App 用换行符分割消息）

---

### 4.4 事件类型完整定义

#### 4.4.1 录像类事件

**`record_started` — 录像已开始**

触发时机：设备开始录像（收到 `POST /camera/record/start` 并执行成功后）

```json
{"event": "record_started", "mode": "NormalRecordeMode", "timestamp": "2026-05-22 14:30:00"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"record_started"` |
| `mode` | string | 当前工作模式名 |
| `timestamp` | string | 录像开始时间 `yyyy-MM-dd HH:mm:ss` |

---

**`record_stopped` — 录像已停止**

触发时机：设备停止录像（收到 `POST /camera/record/stop` 并执行成功后；或 SD 卡满自动停止）

```json
{"event": "record_stopped", "path": "/mnt/DCIM/Normal/VID_20260522_143000.MP4", "duration": 120, "size": 52428800}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"record_stopped"` |
| `path` | string | 保存的视频文件路径 |
| `duration` | number | 视频时长（秒） |
| `size` | number | 文件大小（字节） |

---

**`record_error` — 录像异常中断**

触发时机：录像过程中遇到错误（SD 卡写入失败、编码器异常等）

```json
{"event": "record_error", "reason": "sd_write_error"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"record_error"` |
| `reason` | string | 错误原因，取值见下表 |

reason 取值：

| 值 | 含义 |
|---|---|
| `sd_write_error` | SD 卡写入失败 |
| `sd_full` | SD 卡已满 |
| `encoder_error` | 编码器异常 |

---

#### 4.4.2 拍照类事件

**`capture_done` — 拍照完成**

触发时机：拍照成功并保存到 SD 卡后

```json
{"event": "capture_done", "path": "/mnt/DCIM/Photo/IMG_20260522_143500.jpg", "size": 3145728}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"capture_done"` |
| `path` | string | 照片文件路径 |
| `size` | number | 文件大小（字节） |

---

**`capture_error` — 拍照失败**

触发时机：拍照执行失败

```json
{"event": "capture_error", "reason": "sd_full"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"capture_error"` |
| `reason` | string | 错误原因：`sd_full` / `sd_write_error` / `sensor_error` |

---

#### 4.4.3 SD 卡类事件

**`sd_inserted` — SD 卡插入**

触发时机：检测到 SD 卡插入

```json
{"event": "sd_inserted", "totalMB": 127512, "freeMB": 127000}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"sd_inserted"` |
| `totalMB` | number | 总容量（MB） |
| `freeMB` | number | 剩余空间（MB） |

---

**`sd_removed` — SD 卡拔出**

触发时机：检测到 SD 卡被拔出

```json
{"event": "sd_removed"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"sd_removed"` |

---

**`sd_full` — SD 卡存储已满**

触发时机：SD 卡剩余空间不足（建议阈值：< 50MB）

```json
{"event": "sd_full", "freeMB": 12}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"sd_full"` |
| `freeMB` | number | 当前剩余空间（MB） |

---

**`sd_error` — SD 卡异常**

触发时机：SD 卡读写异常、文件系统损坏

```json
{"event": "sd_error", "reason": "fs_corrupt"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"sd_error"` |
| `reason` | string | `fs_corrupt`（文件系统损坏）/ `read_error`（读取失败）/ `write_error`（写入失败） |

---

#### 4.4.4 模式类事件

**`mode_changed` — 工作模式已切换**

触发时机：收到 `POST /camera/mode` 并成功切换后

```json
{"event": "mode_changed", "mode": "NormalCaptureMode", "modeIndex": 4}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"mode_changed"` |
| `mode` | string | 新模式名 |
| `modeIndex` | number | 新模式编号 0-7 |

---

#### 4.4.5 电池类事件

**`battery_changed` — 电量变化**

触发时机：电量变化超过 5% 时推送一次（不要每 1% 都推，太频繁）

```json
{"event": "battery_changed", "level": 75, "charging": false}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"battery_changed"` |
| `level` | number | 当前电量 0-100 |
| `charging` | boolean | 是否正在充电 |

---

**`battery_low` — 低电量警告**

触发时机：电量降至 **20%** 和 **10%** 时各推送一次

```json
{"event": "battery_low", "level": 10}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"battery_low"` |
| `level` | number | 当前电量 |

---

**`battery_exhausted` — 电量耗尽即将关机**

触发时机：电量降至 **5%** 以下，设备即将关机

```json
{"event": "battery_exhausted", "level": 3, "shutdownInSeconds": 30}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"battery_exhausted"` |
| `level` | number | 当前电量 |
| `shutdownInSeconds` | number | 预计多少秒后关机 |

---

#### 4.4.6 系统类事件

**`device_ready` — 设备就绪**

触发时机：App 建立 TCP 连接后，设备确认所有服务已启动

```json
{"event": "device_ready", "firmware": "V1.0.0", "deviceName": "QZ-CAM-001"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"device_ready"` |
| `firmware` | string | 固件版本 |
| `deviceName` | string | 设备名称 |

---

**`device_busy` — 设备忙**

触发时机：设备正在执行耗时操作（格式化、恢复出厂设置），暂时无法响应其他指令

```json
{"event": "device_busy", "action": "formatting"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"device_busy"` |
| `action` | string | 正在执行的操作：`formatting`（格式化）/ `resetting`（恢复出厂）/ `upgrading`（固件升级） |

---

**`device_idle` — 设备空闲（忙碌操作完成）**

触发时机：耗时操作完成后

```json
{"event": "device_idle", "action": "formatting"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"device_idle"` |
| `action` | string | 刚完成的操作 |

---

**`device_shutdown` — 设备即将关机**

触发时机：用户按了关机键、或电量耗尽即将关机

```json
{"event": "device_shutdown", "reason": "user_request", "delaySeconds": 3}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"device_shutdown"` |
| `reason` | string | `user_request`（用户操作）/ `battery_exhausted`（电量耗尽）/ `overheat`（过热保护） |
| `delaySeconds` | number | 多少秒后关机 |

---

**`file_deleted` — 文件已删除**

触发时机：收到 `DELETE /media/file` 并删除成功后；或循环录像覆盖旧文件时

```json
{"event": "file_deleted", "path": "/mnt/DCIM/Normal/VID_20260520_100000.MP4", "reason": "user_request"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 固定 `"file_deleted"` |
| `path` | string | 被删除的文件路径 |
| `reason` | string | `user_request`（用户删除）/ `loop_overwrite`（循环覆盖）/ `format`（格式化） |

---

### 4.5 事件类型汇总表

| event 值 | 分类 | 触发时机 | 附加字段 | 优先级 |
|---|---|---|---|---|
| `record_started` | 录像 | 开始录像 | `mode`, `timestamp` | P0 |
| `record_stopped` | 录像 | 停止录像 | `path`, `duration`, `size` | P0 |
| `record_error` | 录像 | 录像异常 | `reason` | P0 |
| `capture_done` | 拍照 | 拍照成功 | `path`, `size` | P0 |
| `capture_error` | 拍照 | 拍照失败 | `reason` | P0 |
| `sd_inserted` | SD 卡 | 卡插入 | `totalMB`, `freeMB` | P0 |
| `sd_removed` | SD 卡 | 卡拔出 | 无 | P0 |
| `sd_full` | SD 卡 | 卡满 | `freeMB` | P1 |
| `sd_error` | SD 卡 | 卡异常 | `reason` | P1 |
| `mode_changed` | 模式 | 模式切换 | `mode`, `modeIndex` | P1 |
| `battery_changed` | 电池 | 电量变化 | `level`, `charging` | P1 |
| `battery_low` | 电池 | 低电量 | `level` | P1 |
| `battery_exhausted` | 电池 | 即将关机 | `level`, `shutdownInSeconds` | P1 |
| `device_ready` | 系统 | 连接就绪 | `firmware`, `deviceName` | P0 |
| `device_busy` | 系统 | 设备忙 | `action` | P1 |
| `device_idle` | 系统 | 忙碌结束 | `action` | P1 |
| `device_shutdown` | 系统 | 即将关机 | `reason`, `delaySeconds` | P1 |
| `file_deleted` | 文件 | 文件删除 | `path`, `reason` | P1 |

---

### 4.6 推送时机对照（你收到 REST 请求后，什么时候推事件）

| REST 请求 | 推送什么事件 | 什么时候推 |
|---|---|---|
| `POST /camera/record/start` | `record_started` | 录像实际开始后 |
| `POST /camera/record/stop` | `record_stopped` | 视频文件保存完成后 |
| `POST /camera/capture` | `capture_done` 或 `capture_error` | 照片保存完成/失败后 |
| `POST /camera/mode` | `mode_changed` | 模式切换完成后 |
| `DELETE /media/file` | `file_deleted` | 文件删除后 |
| `POST /settings/format` | `device_busy` → `device_idle` | 格式化开始/结束时 |
| `POST /settings/reset` | `device_busy` → `device_shutdown` | 恢复出厂开始/即将重启时 |
| 无（硬件触发） | `sd_inserted` / `sd_removed` | SD 卡物理插拔时 |
| 无（硬件触发） | `battery_changed` / `battery_low` | 电量变化时 |
| 无（TCP 连接建立） | `device_ready` | App TCP 连上后立即推 |

### 4.7 连接参数

| 参数 | 值 |
|---|---|
| 端口 | `9999` |
| 编码 | UTF-8 |
| 最大连接数 | 1（同时只允许一个 App 连接） |

### 4.8 伪代码

```c
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
bind(server_fd, 9999);
listen(server_fd, 1);

int client_fd = accept(server_fd);

char buf[1024];
while (1) {
    int n = recv(client_fd, buf, sizeof(buf), 0);
    if (n > 0 && strncmp(buf, "S:100.0", 7) == 0) {
        send(client_fd, "OK\n", 3, 0);
    }
}
```

### 4.9 自测

```bash
# 用 nc 测试 TCP 心跳
nc -v 192.168.10.1 9999
# 输入 S:100.0 回车，应收到 OK
```

---

## 5. RTSP 预览服务（端口 8554）

**优先级：P0**

### 5.1 流地址

```
rtsp://192.168.10.1:8554/ch00
```

### 5.2 你要做的

| 要求 | 说明 |
|---|---|
| 编码格式 | H.264（必须支持） |
| 传输方式 | TCP interleaved |
| 预览分辨率 | 1080P 或以下 |
| 自动启动 | 设备上电后 RTSP 自动运行，不需要 HTTP 命令启动 |
| 延迟 | < 500ms 可接受，< 200ms 为优 |

### 5.3 自测

```bash
# 用 ffplay 测试 RTSP
ffplay rtsp://192.168.10.1:8554/ch00

# 用 VLC 测试
vlc rtsp://192.168.10.1:8554/ch00
```

---

## 6. REST API — 设备信息

Base URL：`http://192.168.10.1:8080`

---

### 6.1 获取设备信息

**优先级：P0**

```
GET /api/v1/device/info
```

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "deviceId": 1,
    "deviceName": "QZ-CAM-001",
    "model": "QZ-4K",
    "firmware": "V1.0.0",
    "serialNumber": "SN20260001"
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `deviceId` | number | 设备 ID |
| `deviceName` | string | 设备名称（用户可改） |
| `model` | string | 设备型号（出厂固定） |
| `firmware` | string | 固件版本号 |
| `serialNumber` | string | 设备序列号 |

**curl 自测：**

```bash
curl -s http://192.168.10.1:8080/api/v1/device/info | python3 -m json.tool
```

---

### 6.2 获取存储卡信息

**优先级：P0**

```
GET /api/v1/device/storage
```

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "inserted": true,
    "totalMB": 127512,
    "freeMB": 92341,
    "usedMB": 35171
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `inserted` | boolean | SD 卡是否插入 |
| `totalMB` | number | 总容量（MB） |
| `freeMB` | number | 剩余空间（MB） |
| `usedMB` | number | 已用空间（MB） |

**无卡时返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "inserted": false,
    "totalMB": 0,
    "freeMB": 0,
    "usedMB": 0
  }
}
```

**curl 自测：**

```bash
curl -s http://192.168.10.1:8080/api/v1/device/storage | python3 -m json.tool
```

---

### 6.3 获取电池信息

**优先级：P1**

```
GET /api/v1/device/battery
```

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "level": 85,
    "charging": false,
    "full": false
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `level` | number | 电量百分比 0-100 |
| `charging` | boolean | 是否正在充电 |
| `full` | boolean | 是否已充满 |

**curl 自测：**

```bash
curl -s http://192.168.10.1:8080/api/v1/device/battery | python3 -m json.tool
```

---

### 6.4 设备健康检查

**优先级：P1**

```
GET /api/v1/device/check
```

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "online": true,
    "uptime": 3600
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `online` | boolean | 设备是否正常工作 |
| `uptime` | number | 运行时长（秒） |

**curl 自测：**

```bash
curl -s http://192.168.10.1:8080/api/v1/device/check | python3 -m json.tool
```

---

## 7. REST API — 相机控制

---

### 7.1 获取相机状态

**优先级：P0**

```
GET /api/v1/camera/status
```

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "recording": false,
    "mode": "NormalRecordeMode",
    "modeIndex": 0,
    "rtspUrl": "rtsp://192.168.10.1:8554/ch00"
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `recording` | boolean | 是否正在录像 |
| `mode` | string | 当前工作模式名，见 [第 10 章](#10-工作模式定义) |
| `modeIndex` | number | 当前工作模式编号 0-7 |
| `rtspUrl` | string | RTSP 预览地址 |

**curl 自测：**

```bash
curl -s http://192.168.10.1:8080/api/v1/camera/status | python3 -m json.tool
```

---

### 7.2 开始录像

**优先级：P0**

**调用场景：** 用户在 App 预览页点"录像"按钮。

```
POST /api/v1/camera/record/start
```

请求体：无（或空 JSON `{}`）

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**你要做的事：**

1. 开始录像
2. 内部录像状态置为 `true`
3. 通过 TCP 9999 推送 `{"event": "record_started"}`

**已在录像时再次收到此请求：** 返回成功，不重复操作。

**错误情况返回：**

| 场景 | code | msg |
|---|---|---|
| SD 卡未插入 | -3 | sd card not found |
| SD 卡空间不足 | -4 | storage full |

**curl 自测：**

```bash
curl -s -X POST http://192.168.10.1:8080/api/v1/camera/record/start | python3 -m json.tool

# 验证状态
curl -s http://192.168.10.1:8080/api/v1/camera/status | python3 -m json.tool
# 应该看到 "recording": true
```

---

### 7.3 停止录像

**优先级：P0**

**调用场景：** 用户在 App 预览页点"停止"按钮，或切换工作模式前 App 自动调用。

```
POST /api/v1/camera/record/stop
```

请求体：无

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "path": "/mnt/DCIM/Normal/VID_20260522_143000.MP4",
    "duration": 120,
    "size": 52428800
  }
}
```

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `path` | string | 刚录好的视频文件完整路径 |
| `duration` | number | 视频时长（秒） |
| `size` | number | 文件大小（字节） |

**你要做的事：**

1. 停止录像
2. 保存视频文件到 `/mnt/DCIM/Normal/`（或当前模式对应目录）
3. **对 MP4 文件做 faststart 处理**（moov atom 移到文件头部，详见 8.3 节）
4. 生成缩略图到 `/mnt/DCIM/.thumbnails/` 目录
5. 内部录像状态置为 `false`
6. 通过 TCP 9999 推送 `{"event": "record_stopped", "path": "..."}`

**未在录像时收到此请求：** 返回成功，不做操作。

**curl 自测：**

```bash
curl -s -X POST http://192.168.10.1:8080/api/v1/camera/record/stop | python3 -m json.tool
```

---

### 7.4 拍照

**优先级：P0**

**调用场景：** 用户在 App 预览页点"拍照"按钮（需要处于拍照模式 4-7）。

```
POST /api/v1/camera/capture
```

请求体：无

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "path": "/mnt/DCIM/Photo/IMG_20260522_143500.jpg",
    "size": 2048000,
    "width": 4032,
    "height": 3024
  }
}
```

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `path` | string | 拍摄的照片完整路径 |
| `size` | number | 文件大小（字节） |
| `width` | number | 图片宽度（像素） |
| `height` | number | 图片高度（像素） |

**你要做的事：**

1. 执行拍照
2. 保存到 `/mnt/DCIM/Photo/IMG_yyyyMMdd_HHmmss.jpg`
3. 生成缩略图到 `/mnt/DCIM/.thumbnails/` 目录
4. 通过 TCP 9999 推送 `{"event": "capture_done", "path": "...", "size": ...}`

**错误情况返回：**

| 场景 | code | msg |
|---|---|---|
| 当前不在拍照模式 | -2 | wrong mode |
| SD 卡未插入 | -3 | sd card not found |
| SD 卡空间不足 | -4 | storage full |

**curl 自测：**

```bash
curl -s -X POST http://192.168.10.1:8080/api/v1/camera/capture | python3 -m json.tool
```

---

### 7.5 切换工作模式

**优先级：P1**

**调用场景：** 用户在 App 预览页切换录像/拍照/延时等模式。

```
POST /api/v1/camera/mode
Content-Type: application/json
```

#### 请求体

```json
{
  "mode": 4
}
```

| 字段 | 类型 | 必填 | 取值范围 | 说明 |
|---|---|---|---|---|
| `mode` | number | 是 | 0-7 | 目标模式编号，见 [第 10 章](#10-工作模式定义) |

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "mode": "NormalCaptureMode",
    "modeIndex": 4
  }
}
```

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `mode` | string | 切换后的工作模式名 |
| `modeIndex` | number | 切换后的工作模式编号 |

**你要做的事：**

1. 如果正在录像，**先停止录像**再切换
2. 切换到指定工作模式
3. 通过 TCP 9999 推送 `{"event": "mode_changed", "mode": "NormalCaptureMode", "modeIndex": 4}`

**curl 自测：**

```bash
# 切换到普通拍照模式
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"mode": 4}' \
  http://192.168.10.1:8080/api/v1/camera/mode | python3 -m json.tool
```

---

### 7.6 进入回放模式

**优先级：P1**

**调用场景：** App 从预览页进入相册页时调用。

> **为什么需要这个接口？** 相机在"预览模式"和"回放模式"下的硬件资源分配不同。预览模式时，传感器和编码器在持续工作（实时出 RTSP 流）。进入回放模式后，设备可以释放实时编码资源，把 CPU/内存让给文件读取（缩略图生成、视频 HTTP 在线播放等），响应会更快。HDV CAM 原版协议中这是一个 P0 命令（`cmd=0xbd9`），App 进相册前必须调用。

```
POST /api/v1/camera/playback/enter
```

请求体：无

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "mode": "playback"
  }
}
```

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `mode` | string | 固定返回 `"playback"` |

**你要做的事：**

1. 如果正在录像，**先停止录像**（保存当前文件）
2. 停止 RTSP 实时预览流（可选，减轻 CPU 负载）
3. 内部状态标记为"回放模式"
4. 准备好文件服务（确保 HTTP 文件接口可响应）

**已在回放模式时再次收到此请求：** 返回成功，不重复操作。

**curl 自测：**

```bash
curl -s -X POST http://192.168.10.1:8080/api/v1/camera/playback/enter | python3 -m json.tool
```

---

### 7.7 退出回放模式

**优先级：P1**

**调用场景：** App 从相册页返回预览页时调用。

> **为什么需要这个接口？** 退出回放后设备恢复实时编码，重新输出 RTSP 预览流。App 退出相册回到预览页时必须调用，否则预览画面可能黑屏。

```
POST /api/v1/camera/playback/exit
```

请求体：无

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "mode": "NormalRecordeMode",
    "modeIndex": 0
  }
}
```

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `mode` | string | 恢复后的工作模式名 |
| `modeIndex` | number | 恢复后的工作模式编号 |

**你要做的事：**

1. 退出回放模式
2. 恢复 RTSP 实时预览流
3. 恢复到进入回放前的工作模式
4. 内部状态恢复为正常工作模式

**未在回放模式时收到此请求：** 返回成功，不做操作。

**curl 自测：**

```bash
curl -s -X POST http://192.168.10.1:8080/api/v1/camera/playback/exit | python3 -m json.tool

# 退出后验证 RTSP 预览已恢复
curl -s http://192.168.10.1:8080/api/v1/camera/status | python3 -m json.tool
# 应该看到正常的 mode 和 rtspUrl
```

---

### 7.8 变焦控制

**优先级：P2**

**调用场景：** 用户在 App 预览页双指缩放或点击变焦按钮。

```
POST /api/v1/camera/zoom
Content-Type: application/json
```

#### 请求体

```json
{
  "level": 5
}
```

| 字段 | 类型 | 必填 | 取值范围 | 说明 |
|---|---|---|---|---|
| `level` | number | 是 | 0-10 | 变焦级别，0 为无缩放，10 为最大 |

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "level": 5
  }
}
```

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `level` | number | 当前实际变焦级别 |

**curl 自测：**

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"level": 5}' \
  http://192.168.10.1:8080/api/v1/camera/zoom | python3 -m json.tool
```

---

## 8. REST API — 媒体文件

这一章定义 App 如何浏览、**在线查看**、下载、删除相机上的照片和视频。

> **重要背景：** App 不是"先下载完再看"，而是**直接用你返回的 HTTP URL 在线播放视频**。播放器向你发 HTTP 请求，通过 Range 分段获取数据边下边播。所以你的 HTTP 文件服务必须正确支持 Range 请求和 `Accept-Ranges` 头，视频文件的 moov atom 必须在文件头部。详见 8.3 节。

---

### 8.1 获取文件列表

**优先级：P1**

```
GET /api/v1/media/files?type=video_normal&page=1&pageSize=20
```

#### 请求参数（Query String）

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `type` | string | 是 | 媒体类型，见下表 |
| `page` | number | 否 | 页码，从 1 开始，默认 1 |
| `pageSize` | number | 否 | 每页数量，默认 20 |

#### type 取值

| type 值 | 含义 | 存储目录 |
|---|---|---|
| `video_normal` | 普通视频 | `/mnt/DCIM/Normal/` |
| `video_event` | 事件视频（碰撞触发） | `/mnt/DCIM/Event/` |
| `video_parking` | 停车监控视频 | `/mnt/DCIM/Parking/` |
| `photo` | 照片 | `/mnt/DCIM/Photo/` |

#### 你要返回

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "total": 45,
    "page": 1,
    "pageSize": 20,
    "files": [
      {
        "path": "/mnt/DCIM/Normal/VID_20260522_143000.MP4",
        "name": "VID_20260522_143000.MP4",
        "size": 52428800,
        "time": "2026-05-22 14:30:00",
        "duration": 120,
        "width": 1920,
        "height": 1080
      },
      {
        "path": "/mnt/DCIM/Normal/VID_20260522_141500.MP4",
        "name": "VID_20260522_141500.MP4",
        "size": 26214400,
        "time": "2026-05-22 14:15:00",
        "duration": 60,
        "width": 1920,
        "height": 1080
      }
    ]
  }
}
```

#### 文件对象字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `path` | string | 是 | 文件完整路径 |
| `name` | string | 是 | 文件名 |
| `size` | number | 是 | 文件大小（字节） |
| `time` | string | 是 | 拍摄时间 `yyyy-MM-dd HH:mm:ss` |
| `duration` | number | 视频必填 | 时长（秒），照片不返回或返回 0 |
| `width` | number | 否 | 分辨率宽 |
| `height` | number | 否 | 分辨率高 |

#### 空列表返回

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "total": 0,
    "page": 1,
    "pageSize": 20,
    "files": []
  }
}
```

**curl 自测：**

```bash
# 获取普通视频列表
curl -s "http://192.168.10.1:8080/api/v1/media/files?type=video_normal&page=1&pageSize=20" | python3 -m json.tool

# 获取照片列表
curl -s "http://192.168.10.1:8080/api/v1/media/files?type=photo&page=1&pageSize=20" | python3 -m json.tool

# 获取事件视频列表
curl -s "http://192.168.10.1:8080/api/v1/media/files?type=video_event&page=1&pageSize=20" | python3 -m json.tool
```

---

### 8.2 获取缩略图

**优先级：P1**

> **设计思路：** 缩略图用静态 HTTP URL 直接访问，不走 REST API 接口。这样做的好处是 App 端可以直接把 URL 传给 `Image.network()` / `CachedNetworkImage` 等图片库，库自带缓存、并发加载、占位图等能力，不需要额外封装。HDV CAM 原版也是这么做的（`/thumb/mnt/DCIM/...`）。

#### URL 格式

```
http://192.168.10.1:8080/thumb/<文件路径去掉开头斜杠>.jpg
```

#### 对应关系（视频和照片都有缩略图）

| 原文件路径 | 缩略图 URL |
|---|---|
| `/mnt/DCIM/Normal/VID_20260522_143000.MP4` | `http://192.168.10.1:8080/thumb/mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg` |
| `/mnt/DCIM/Event/EVT_20260522_143000.MP4` | `http://192.168.10.1:8080/thumb/mnt/DCIM/Event/EVT_20260522_143000.MP4.jpg` |
| `/mnt/DCIM/Photo/IMG_20260522_143500.jpg` | `http://192.168.10.1:8080/thumb/mnt/DCIM/Photo/IMG_20260522_143500.jpg.jpg` |

**规则：** URL 路径 = `/thumb/` + 原文件 path 去掉开头 `/` + `.jpg`

#### 你要返回

| 响应头 | 值 |
|---|---|
| **Content-Type** | `image/jpeg` |
| **Content-Length** | 缩略图文件大小（字节） |
| **Cache-Control** | `public, max-age=86400`（建议，让 App 图片库缓存） |
| **Body** | JPEG 图片二进制数据 |

#### 缩略图规格

| 参数 | 值 |
|---|---|
| 格式 | JPEG |
| 建议尺寸 | 320 × 240（保持原始宽高比） |
| 建议质量 | 75%（平衡大小和清晰度） |
| 单张大小 | 10-30 KB |

#### 你要做的事

1. **录像停止时**：从视频文件提取一帧（建议第 1 秒），生成缩略图
2. **拍照完成时**：对原图做缩放，生成缩略图
3. **缩略图存储路径**：`/mnt/DCIM/.thumbnails/` 目录，命名为 `原文件名.jpg`
   - 视频：`/mnt/DCIM/.thumbnails/VID_20260522_143000.MP4.jpg`
   - 照片：`/mnt/DCIM/.thumbnails/IMG_20260522_143500.jpg.jpg`
4. **HTTP 路由映射**：收到 `/thumb/mnt/DCIM/Normal/VID_xxx.MP4.jpg` 请求时，读取 `/mnt/DCIM/.thumbnails/VID_xxx.MP4.jpg` 返回
5. **缩略图不存在时**：返回一张内置的默认占位图（灰色或带相机图标的 JPEG）

#### 路由实现伪代码

```c
// HTTP 请求：GET /thumb/mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg
// 提取路径：mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg
// 取文件名：VID_20260522_143000.MP4.jpg
// 读取文件：/mnt/DCIM/.thumbnails/VID_20260522_143000.MP4.jpg
// 返回 JPEG 二进制
```

**curl 自测：**

```bash
# 视频缩略图
curl -s -o thumb_video.jpg "http://192.168.10.1:8080/thumb/mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg"
ls -la thumb_video.jpg   # 应该 10-30KB

# 照片缩略图
curl -s -o thumb_photo.jpg "http://192.168.10.1:8080/thumb/mnt/DCIM/Photo/IMG_20260522_143500.jpg.jpg"
ls -la thumb_photo.jpg   # 应该 10-30KB

# 验证 Content-Type
curl -sI "http://192.168.10.1:8080/thumb/mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg" | grep -i content-type
# 期望：Content-Type: image/jpeg
```

---

### 8.3 在线查看/下载文件

**优先级：P0**

> **这个接口是整个相册功能的核心。** App 用这个接口在线查看图片、在线播放视频、下载文件到手机。不是"先下载完再看"，而是**边下边播**——App 播放器直接用这个 HTTP URL 播放视频，和看网页视频一样。

```
GET /api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `path` | string | 是 | 文件完整路径 |

#### 你要返回的响应头

```http
HTTP/1.1 200 OK
Content-Type: video/mp4
Content-Length: 52428800
Accept-Ranges: bytes
```

| 响应头 | 说明 |
|---|---|
| `Content-Type` | 根据文件类型返回：`.mp4` → `video/mp4`，`.jpg` → `image/jpeg`，`.mov` → `video/quicktime` |
| `Content-Length` | 文件总大小（字节） |
| `Accept-Ranges: bytes` | **必须返回**，告诉 App 播放器"我支持 Range 请求"，播放器看到这个头才会做在线播放 |
| Body | 文件二进制数据 |

#### 支持 Range 请求（在线播放和断点续传都靠它）

App 播放器在线播放视频时，不会一次请求整个文件，而是分段请求：

- 先请求文件开头的一小段（读 moov atom，获取视频元信息）
- 然后按播放进度分段请求后续数据
- 用户拖动进度条时，直接跳到对应字节位置请求

请求头：
```
Range: bytes=0-65535
```

响应（返回 206，不是 200）：
```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 0-65535/52428800
Content-Length: 65536
Accept-Ranges: bytes
```

**用户拖动进度条到中间位置时：**
```
Range: bytes=26214400-
```

响应：
```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 26214400-52428799/52428800
Content-Length: 26214400
Accept-Ranges: bytes
```

**你必须正确支持 Range 请求。** 如果不支持，App 端视频无法在线播放、无法拖动进度条、大文件下载也会失败。

#### Range 请求的三种格式（都要支持）

| 请求格式 | 含义 | 响应 |
|---|---|---|
| `Range: bytes=0-999` | 请求第 0~999 字节 | 206 + `Content-Range: bytes 0-999/总大小` |
| `Range: bytes=1000-` | 请求第 1000 字节到末尾 | 206 + `Content-Range: bytes 1000-末尾/总大小` |
| 无 Range 头 | 请求完整文件 | 200 + 完整文件 |

#### MP4 文件的 moov atom 必须在文件头部

> **这是在线播放能不能秒开的关键，非常重要。**

MP4 文件有一个元数据块叫 `moov atom`，记录了视频时长、关键帧索引、编码参数等。播放器必须先读到 moov 才能开始播放。

| moov 位置 | App 端播放行为 |
|---|---|
| **文件头部**（faststart） | 播放器请求前几十 KB 就拿到 moov，**1-2 秒内开始播放** ✅ |
| **文件尾部**（默认） | 播放器要先 Range 请求到文件末尾拿 moov，再回到开头请求数据，**慢很多** ❌ |

**录像保存 MP4 时，必须把 moov atom 放到文件前面。** 做法：

1. **方案 A（推荐）**：录像完成后做 faststart 后处理，把 moov 从文件尾部挪到头部。等价于 ffmpeg 的 `-movflags +faststart`
2. **方案 B**：录像编码器直接配置 moov 前置（如果芯片 SDK 支持）

**验证方法：**

```bash
# 用 ffprobe 检查 moov 位置（在电脑上测试录好的 MP4）
ffprobe -v quiet -show_entries format_tags=major_brand -show_entries stream=codec_type VID_test.MP4

# 用 AtomicParsley 查看（更直观）
AtomicParsley VID_test.MP4 -T

# 如果 moov 在 ftyp 后面（文件开头），说明是 faststart ✅
# 如果 moov 在 mdat 后面（文件末尾），需要处理 ❌

# 手动转换为 faststart（测试用）
ffmpeg -i input.mp4 -c copy -movflags +faststart output.mp4
```

#### 实现要点

1. 读取 `path` 参数指定的文件
2. 设置正确的 `Content-Type`（根据文件扩展名）
3. 设置 `Content-Length`（文件总大小或 Range 片段大小）
4. **始终返回 `Accept-Ranges: bytes` 头**
5. 如果请求头有 `Range`，解析字节范围，返回 `206 Partial Content` + `Content-Range` 头 + 对应的字节数据
6. 如果没有 `Range` 头，返回 `200 OK` + 完整文件
7. 如果文件不存在，返回 `{"code": -1, "msg": "file not found", "data": null}`
8. **录像文件保存时确保 moov atom 在文件头部**

**curl 自测：**

```bash
# 下载图片
curl -s -o photo.jpg "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Photo/IMG_20260522_143500.jpg"

# 下载视频
curl -s -o video.mp4 "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4"

# 测试 Range 请求 —— 请求前 64KB（模拟播放器读 moov）
curl -s -H "Range: bytes=0-65535" -o head.bin \
  "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4" \
  -w "\nHTTP状态码: %{http_code}\n"
# 期望输出：HTTP状态码: 206

# 测试 Range 请求 —— 请求中间位置（模拟用户拖动进度条）
curl -s -H "Range: bytes=26214400-26279935" -o mid.bin \
  "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4" \
  -w "\nHTTP状态码: %{http_code}\n"
# 期望输出：HTTP状态码: 206

# 验证 Accept-Ranges 头存在
curl -sI "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4" | grep -i accept-ranges
# 期望输出：Accept-Ranges: bytes
```

---

### 8.4 删除文件

**优先级：P1**

```
DELETE /api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `path` | string | 是 | 要删除的文件路径 |

#### 你要返回

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**你要做的事：**

1. 删除原始文件
2. 同时删除对应的缩略图
3. 文件不存在时返回错误

**curl 自测：**

```bash
curl -s -X DELETE "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4" | python3 -m json.tool
```

---

## 9. REST API — 设置

---

### 9.1 获取菜单（一次性返回全部）

**优先级：P1**

```
GET /api/v1/settings/menus?lang=zh-CN
```

#### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `lang` | string | 否 | 语言码，默认 `zh-CN` |

#### 语言码

| 语言 | 值 |
|---|---|
| 简体中文 | `zh-CN` |
| 繁体中文 | `zh-TW` |
| English | `en` |
| 韩语 | `ko` |
| 日语 | `ja` |

#### 你要返回

**重要：把菜单定义、翻译、当前值一次性返回，App 不需要多次请求。**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "currentMode": "NormalRecordeMode",
    "currentModeIndex": 0,
    "modeMenus": [
      {
        "id": "rec_resolution",
        "title": "分辨率",
        "index": 0,
        "currentValue": 2,
        "options": [
          { "index": 0, "id": "720p30", "title": "720P 30FPS" },
          { "index": 1, "id": "1080p30", "title": "1080P 30FPS" },
          { "index": 2, "id": "4k30", "title": "4K 30FPS" }
        ]
      },
      {
        "id": "loop_record",
        "title": "循环录像",
        "index": 1,
        "currentValue": 0,
        "options": [
          { "index": 0, "id": "off", "title": "关闭" },
          { "index": 1, "id": "1min", "title": "1分钟" },
          { "index": 2, "id": "3min", "title": "3分钟" },
          { "index": 3, "id": "5min", "title": "5分钟" }
        ]
      },
      {
        "id": "exposure",
        "title": "曝光补偿",
        "index": 2,
        "currentValue": 2,
        "options": [
          { "index": 0, "id": "n2", "title": "-2.0" },
          { "index": 1, "id": "n1", "title": "-1.0" },
          { "index": 2, "id": "0", "title": "0" },
          { "index": 3, "id": "p1", "title": "+1.0" },
          { "index": 4, "id": "p2", "title": "+2.0" }
        ]
      }
    ],
    "systemMenus": [
      {
        "id": "wifi_ssid",
        "title": "Wi-Fi 名称",
        "index": 0,
        "currentValue": -1,
        "options": [],
        "type": "input",
        "value": "QZ-CAM-001"
      },
      {
        "id": "wifi_password",
        "title": "Wi-Fi 密码",
        "index": 1,
        "currentValue": -1,
        "options": [],
        "type": "input",
        "value": "12345678"
      },
      {
        "id": "date_time",
        "title": "日期时间",
        "index": 2,
        "currentValue": -1,
        "options": [],
        "type": "datetime",
        "value": "2026-05-22 14:30:00"
      },
      {
        "id": "language",
        "title": "语言",
        "index": 3,
        "currentValue": 0,
        "options": [
          { "index": 0, "id": "zh-CN", "title": "简体中文" },
          { "index": 1, "id": "en", "title": "English" },
          { "index": 2, "id": "zh-TW", "title": "繁體中文" }
        ]
      },
      {
        "id": "format_card",
        "title": "格式化存储卡",
        "index": 4,
        "currentValue": -1,
        "options": [],
        "type": "action"
      },
      {
        "id": "factory_reset",
        "title": "恢复出厂设置",
        "index": 5,
        "currentValue": -1,
        "options": [],
        "type": "action"
      }
    ]
  }
}
```

#### 菜单项字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | 菜单唯一标识 |
| `title` | string | 菜单显示名称（已翻译） |
| `index` | number | 菜单序号 |
| `currentValue` | number | 当前选中的选项 index，`-1` 表示非选项类型 |
| `options` | array | 可选项列表（选项类型菜单才有） |
| `options[].index` | number | 选项序号 |
| `options[].id` | string | 选项标识 |
| `options[].title` | string | 选项显示名称（已翻译） |
| `type` | string | 可选，特殊菜单类型：`input`（输入框）、`datetime`（时间选择）、`action`（动作按钮） |
| `value` | string | 可选，`input`/`datetime` 类型的当前值 |

#### 关键设计说明

**为什么一次性返回？** 原协议需要 3 次请求（setting_keys.xml + 翻译 + 当前值）再在 App 端拼装。新设计由设备端直接组装好，App 端拿到就能直接渲染，减少请求次数和 App 端复杂度。

**curl 自测：**

```bash
curl -s "http://192.168.10.1:8080/api/v1/settings/menus?lang=zh-CN" | python3 -m json.tool
```

---

### 9.2 修改菜单选项值

**优先级：P1**

```
POST /api/v1/settings/menu/value
Content-Type: application/json

{
  "id": "rec_resolution",
  "value": 1
}
```

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | string | 菜单 ID（从菜单列表获取） |
| `value` | number | 新的选项 index |

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**你要做的事：** 修改对应设置项的当前值，使其立即生效。

**curl 自测：**

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"id": "rec_resolution", "value": 1}' \
  http://192.168.10.1:8080/api/v1/settings/menu/value | python3 -m json.tool
```

---

### 9.3 设置 Wi-Fi

**优先级：P1**

```
POST /api/v1/settings/wifi
Content-Type: application/json

{
  "ssid": "MyCamera",
  "password": "12345678"
}
```

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `ssid` | string | 否 | 新的 Wi-Fi 名称，不传则不修改 |
| `password` | string | 否 | 新的 Wi-Fi 密码，不传则不修改 |

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "ssid": "MyCamera",
    "password": "12345678",
    "reconnectRequired": true
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `reconnectRequired` | boolean | 是否需要重新连接 Wi-Fi |

**你要做的事：** 修改 AP 热点名称/密码。修改后客户端需要重新连接。

**curl 自测：**

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"ssid": "MyCamera", "password": "88888888"}' \
  http://192.168.10.1:8080/api/v1/settings/wifi | python3 -m json.tool
```

---

### 9.4 同步日期时间

**优先级：P1**

```
POST /api/v1/settings/datetime
Content-Type: application/json

{
  "datetime": "2026-05-22 14:30:00"
}
```

| 参数 | 类型 | 格式 | 说明 |
|---|---|---|---|
| `datetime` | string | `yyyy-MM-dd HH:mm:ss` | 手机当前时间 |

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**你要做的事：** 将设备系统时间设置为传入的时间值。

**curl 自测：**

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"datetime": "2026-05-22 14:30:00"}' \
  http://192.168.10.1:8080/api/v1/settings/datetime | python3 -m json.tool
```

---

### 9.5 格式化 SD 卡

**优先级：P2**

```
POST /api/v1/settings/format
```

请求体：无

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**你要做的事：** 格式化 SD 卡，删除所有媒体文件。

**curl 自测：**

```bash
curl -s -X POST http://192.168.10.1:8080/api/v1/settings/format | python3 -m json.tool
```

---

### 9.6 恢复出厂设置

**优先级：P2**

```
POST /api/v1/settings/reset
```

请求体：无

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**curl 自测：**

```bash
curl -s -X POST http://192.168.10.1:8080/api/v1/settings/reset | python3 -m json.tool
```

---

## 10. 工作模式定义

### 10.1 模式枚举

| 编号 | 模式名 | 中文 | 类型 |
|---|---|---|---|
| 0 | `NormalRecordeMode` | 普通录像 | 录像 |
| 1 | `SlowRecordeMode` | 慢动作 | 录像 |
| 2 | `LoopRecordeMode` | 循环录像 | 录像 |
| 3 | `TimeLapseMode` | 延时摄影 | 录像 |
| 4 | `NormalCaptureMode` | 普通拍照 | 拍照 |
| 5 | `AutoCaptureMode` | 自动拍照 | 拍照 |
| 6 | `ContinueCaptureMode` | 连拍 | 拍照 |
| 7 | `TimingCaptureMode` | 定时拍照 | 拍照 |

### 10.2 模式与菜单的关系

每种模式有独立的设置菜单列表。切换模式后，菜单内容会变化。`GET /api/v1/settings/menus` 返回的 `modeMenus` 会自动随当前模式变化。

### 10.3 文件存储目录

| 模式 | 存储目录 |
|---|---|
| 录像模式（0-3） | `/mnt/DCIM/Normal/` |
| 事件触发 | `/mnt/DCIM/Event/` |
| 停车监控 | `/mnt/DCIM/Parking/` |
| 拍照模式（4-7） | `/mnt/DCIM/Photo/` |

### 10.4 文件命名规范

| 类型 | 格式 | 示例 |
|---|---|---|
| 视频 | `VID_yyyyMMdd_HHmmss.MP4` | `VID_20260522_143000.MP4` |
| 事件视频 | `EVT_yyyyMMdd_HHmmss.MP4` | `EVT_20260522_143000.MP4` |
| 停车视频 | `PKG_yyyyMMdd_HHmmss.MP4` | `PKG_20260522_143000.MP4` |
| 照片 | `IMG_yyyyMMdd_HHmmss.jpg` | `IMG_20260522_143500.jpg` |

---

## 11. 启动顺序

设备上电后的服务启动顺序：

```
设备上电
  │
  ├─[1] 初始化硬件（Camera、SD Card、Wi-Fi 模块）
  │
  ├─[2] 启动 AP 热点
  │     └─ 网关: 192.168.10.1
  │     └─ DHCP: 192.168.10.100 ~ 200
  │
  ├─[3] 启动 TCP 心跳监听 :9999
  │
  ├─[4] 启动 HTTP REST API 服务 :8080
  │
  ├─[5] 启动 RTSP 预览服务 :8554
  │
  └─[6] 就绪，等待 App 连接
```

**关键：步骤 3（TCP 9999）必须在步骤 4（HTTP 8080）之前或同时启动。**

---

## 12. curl 自测命令集

把以下脚本保存为 `test.sh`，一键验证所有 P0 接口：

```bash
#!/bin/bash
BASE="http://192.168.10.1:8080"
OK=0
FAIL=0

test_api() {
  local desc=$1
  local method=$2
  local url=$3
  local body=$4
  
  echo -n "[$method] $desc ... "
  
  if [ "$method" = "GET" ]; then
    result=$(curl -s -w "\n%{http_code}" "$BASE$url")
  else
    if [ -n "$body" ]; then
      result=$(curl -s -w "\n%{http_code}" -X $method -H "Content-Type: application/json" -d "$body" "$BASE$url")
    else
      result=$(curl -s -w "\n%{http_code}" -X $method "$BASE$url")
    fi
  fi
  
  http_code=$(echo "$result" | tail -1)
  response=$(echo "$result" | sed '$d')
  
  if [ "$http_code" = "200" ]; then
    echo "OK ($http_code)"
    OK=$((OK+1))
  else
    echo "FAIL ($http_code)"
    echo "  Response: $response"
    FAIL=$((FAIL+1))
  fi
}

echo "===== P0: 基础 ====="
test_api "设备信息"      GET  "/api/v1/device/info"
test_api "存储卡信息"     GET  "/api/v1/device/storage"
test_api "相机状态"       GET  "/api/v1/camera/status"

echo ""
echo "===== P0: 录像 ====="
test_api "开始录像"       POST "/api/v1/camera/record/start"
sleep 2
test_api "录像状态确认"    GET  "/api/v1/camera/status"
test_api "停止录像"       POST "/api/v1/camera/record/stop"
sleep 1
test_api "停止后状态确认"  GET  "/api/v1/camera/status"

echo ""
echo "===== P0: 拍照 ====="
test_api "拍照"           POST "/api/v1/camera/capture"

echo ""
echo "===== P1: 媒体 ====="
test_api "普通视频列表"    GET  "/api/v1/media/files?type=video_normal&page=1&pageSize=20"
test_api "照片列表"        GET  "/api/v1/media/files?type=photo&page=1&pageSize=20"

echo ""
echo "===== P1: 设置 ====="
test_api "菜单列表"        GET  "/api/v1/settings/menus?lang=zh-CN"
test_api "切换模式"        POST "/api/v1/camera/mode" '{"mode": 4}'
sleep 1
test_api "模式确认"        GET  "/api/v1/camera/status"
test_api "切回录像"        POST "/api/v1/camera/mode" '{"mode": 0}'
test_api "同步时间"        POST "/api/v1/settings/datetime" '{"datetime": "2026-05-22 14:30:00"}'

echo ""
echo "===== 结果 ====="
echo "通过: $OK  失败: $FAIL"
```

---

## 13. 错误码表

| code | 含义 | 什么时候返回 |
|---|---|---|
| `0` | 成功 | 所有正常请求 |
| `-1` | 通用失败 | 未明确分类的错误 |
| `-2` | 参数错误 | 缺少必填参数、参数格式不对 |
| `-3` | SD 卡未插入 | 需要 SD 卡的操作（录像、拍照、格式化） |
| `-4` | SD 卡已满 | 空间不足 |
| `-5` | 文件不存在 | 下载/删除不存在的文件 |
| `-6` | 设备忙 | 正在格式化、正在恢复出厂设置等 |
| `-7` | 模式不支持 | 传入的 mode 编号不在 0-7 范围内 |

示例：

```json
{
  "code": -3,
  "msg": "sd card not inserted",
  "data": null
}
```
