# REST API — 相机控制

> [← 返回目录](./README.md)

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
| `mode` | string | 当前工作模式名，见 [工作模式定义](./work-modes.md) |
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
3. **对 MP4 文件做 faststart 处理**（moov atom 移到文件头部，详见 [在线查看/下载文件](./media-files.md#83-在线查看下载文件)）
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
| `mode` | number | 是 | 0-7 | 目标模式编号，见 [工作模式定义](./work-modes.md) |

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

