# REST API — 相机控制

> [← 返回目录](./README.md)

---

### 7.1 获取相机状态

**优先级：P0**

```
GET /api/v1/camera/status
```

#### 响应示例

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

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `recording` | boolean | 是否正在录像，`true` 表示录像中 |
| `mode` | string | 当前工作模式名称，如 `NormalRecordeMode`，见 [工作模式定义](./work-modes.md) |
| `modeIndex` | number | 当前工作模式编号 0-7，0-3 为录像模式，4-7 为拍照模式 |
| `rtspUrl` | string | RTSP 实时预览地址，如 `rtsp://192.168.10.1:8554/ch00` |

#### 数据模型

```dart
class CameraStatus {
  final bool recording;
  final String mode;
  final int modeIndex;
  final String rtspUrl;

  CameraStatus.fromJson(Map<String, dynamic> json)
      : recording = json['recording'],
        mode = json['mode'],
        modeIndex = json['modeIndex'],
        rtspUrl = json['rtspUrl'];

  bool get isRecordMode => modeIndex <= 3;
  bool get isCaptureMode => modeIndex >= 4;
}
```

#### 调用示例

```dart
final resp = await http.get<CameraStatus>(
  '/api/v1/camera/status',
  fromData: (d) => CameraStatus.fromJson(d),
);
if (resp.isSuccess) {
  final status = resp.data!;
  print('正在录像: ${status.recording}');
  print('当前模式: ${status.mode}');
}
```

---

### 7.2 开始录像

**优先级：P0**

**什么时候调用：** 用户在预览页点"录像"按钮。需要当前处于录像模式（modeIndex 0-3）。

```
POST /api/v1/camera/record/start
```

无请求体。

#### 响应示例

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

`data` 为 `null`，成功与否看 `code`。

#### 错误情况

| code | msg | 说明 |
|---|---|---|
| -3 | sd card not found | SD 卡未插入 |
| -4 | storage full | SD 卡空间不足 |

#### 调用示例

```dart
Future<void> startRecording() async {
  final resp = await http.post('/api/v1/camera/record/start');
  if (resp.isSuccess) {
    setState(() => _isRecording = true);
  } else if (resp.code == -3) {
    showError('请插入 SD 卡');
  } else if (resp.code == -4) {
    showError('SD 卡已满，请清理文件');
  } else {
    showError('录像启动失败: ${resp.msg}');
  }
}
```

---

### 7.3 停止录像

**优先级：P0**

**什么时候调用：** 用户点"停止"按钮，或 App 需要切换模式 / 进入相册前自动调用。

```
POST /api/v1/camera/record/stop
```

无请求体。

#### 响应示例

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

> 未在录像时调用此接口，返回成功但 `data` 为 `null`。

#### 调用示例

```dart
Future<void> stopRecording() async {
  final resp = await http.post('/api/v1/camera/record/stop');
  if (resp.isSuccess) {
    setState(() => _isRecording = false);
    if (resp.data != null) {
      final path = resp.data['path'];
      final duration = resp.data['duration'];
      showSnackBar('录像已保存，时长 ${duration}s');
    }
  }
}
```

---

### 7.4 拍照

**优先级：P0**

**什么时候调用：** 用户在预览页点"拍照"按钮。需要当前处于拍照模式（modeIndex 4-7）。

```
POST /api/v1/camera/capture
```

无请求体。

#### 响应示例

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

#### 错误情况

| code | msg | 说明 |
|---|---|---|
| -2 | wrong mode | 当前不在拍照模式 |
| -3 | sd card not found | SD 卡未插入 |
| -4 | storage full | SD 卡空间不足 |

#### 调用示例

```dart
Future<void> takePhoto() async {
  final resp = await http.post('/api/v1/camera/capture');
  if (resp.isSuccess) {
    final path = resp.data['path'];
    showSnackBar('拍照成功');
    // path 可以直接拼缩略图 URL 预览
  } else if (resp.code == -2) {
    showError('请先切换到拍照模式');
  }
}
```

---

### 7.5 切换工作模式

**优先级：P1**

**什么时候调用：** 用户在预览页切换录像/拍照/延时等模式。

```
POST /api/v1/camera/mode
Content-Type: application/json
```

#### 请求体

```json
{"mode": 4}
```

| 字段 | 类型 | 必填 | 取值范围 | 说明 |
|---|---|---|---|---|
| `mode` | number | 是 | 0-7 | 目标模式编号，见 [工作模式定义](./work-modes.md) |

#### 响应示例

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

> 如果正在录像，设备会先自动停止录像再切换模式。

#### 调用示例

```dart
Future<void> switchMode(int modeIndex) async {
  final resp = await http.post(
    '/api/v1/camera/mode',
    body: {'mode': modeIndex},
  );
  if (resp.isSuccess) {
    setState(() {
      _currentMode = resp.data['mode'];
      _currentModeIndex = resp.data['modeIndex'];
    });
    // 切换模式后菜单项会变，重新加载
    await loadMenus();
  }
}
```

---

### 7.6 进入/退出回放模式

**优先级：P1**

> **什么时候调用？** App 从预览页进入相册页时调用 `enter`，从相册页返回预览页时调用 `exit`。
>
> **为什么需要？** 相机硬件资源有限。预览模式下传感器和编码器在实时工作（输出 RTSP 流），进入回放模式后设备释放编码资源给文件服务（缩略图加载、视频在线播放会更流畅）。退出回放后设备恢复 RTSP 预览流。HDV CAM 原版协议中这是必须调用的命令（`cmd=0xbd9`），**不调用的话进相册可能导致设备响应变慢或预览黑屏**。

```
POST /api/v1/camera/playback/enter    ← 进入回放（进相册前）
POST /api/v1/camera/playback/exit     ← 退出回放（回预览时）
```

无请求体。

#### 返回示例

**进入回放：**
```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "mode": "playback"
  }
}
```

**退出回放：**
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

| 字段 | 类型 | 说明 |
|---|---|---|
| `mode` | string | 进入时固定 `"playback"`；退出时返回恢复后的工作模式名 |
| `modeIndex` | number | 退出时返回恢复后的工作模式编号（进入时无此字段） |

#### 调用示例

```dart
class AlbumPage extends StatefulWidget {
  @override
  State<AlbumPage> createState() => _AlbumPageState();
}

class _AlbumPageState extends State<AlbumPage> {
  @override
  void initState() {
    super.initState();
    // 进入相册页时，通知设备进入回放模式
    http.post('/api/v1/camera/playback/enter');
    _loadFiles();
  }

  @override
  void dispose() {
    // 离开相册页时，通知设备退出回放模式，恢复 RTSP 预览
    http.post('/api/v1/camera/playback/exit');
    super.dispose();
  }

  // ...
}
```

---

### 7.7 变焦控制

**优先级：P2**

**什么时候调用：** 用户在预览页双指缩放或点击变焦按钮。

```
POST /api/v1/camera/zoom
Content-Type: application/json
```

#### 请求体

```json
{"level": 5}
```

| 字段 | 类型 | 必填 | 取值范围 | 说明 |
|---|---|---|---|---|
| `level` | number | 是 | 0-10 | 变焦级别，0 为无缩放，10 为最大 |

#### 响应示例

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

#### 调用示例

```dart
Future<void> setZoom(int level) async {
  final resp = await http.post('/api/v1/camera/zoom', body: {'level': level});
  if (resp.isSuccess) {
    setState(() => _zoomLevel = resp.data['level']);
  }
}
```
