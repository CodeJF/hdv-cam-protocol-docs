# TCP 心跳服务（端口 9999）

> [← 返回目录](./README.md)

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
