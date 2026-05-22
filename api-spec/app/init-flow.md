# 连接初始化与页面对应

> [← 返回目录](./README.md)

App 连接设备的完整代码流程：

```dart
class QZCameraClient {
  final QZHttpClient http = QZHttpClient();
  final QZEventSocket eventSocket = QZEventSocket();

  DeviceInfo? deviceInfo;
  StorageInfo? storageInfo;
  CameraStatus? cameraStatus;

  /// 完整连接流程
  Future<bool> connect() async {
    // [1] 检测设备
    if (!await DeviceDetector.isQZDeviceConnected()) {
      return false;
    }

    // [2] TCP 心跳（最先建立）
    final socketOk = await eventSocket.connect();
    if (!socketOk) return false;

    // [3] 并行读取基础状态
    final results = await Future.wait([
      http.get('/api/v1/device/info', fromData: (d) => DeviceInfo.fromJson(d)),
      http.get('/api/v1/device/storage', fromData: (d) => StorageInfo.fromJson(d)),
      http.get('/api/v1/camera/status', fromData: (d) => CameraStatus.fromJson(d)),
    ]);

    deviceInfo = (results[0] as ApiResponse<DeviceInfo>).data;
    storageInfo = (results[1] as ApiResponse<StorageInfo>).data;
    cameraStatus = (results[2] as ApiResponse<CameraStatus>).data;

    // [4] 连接 RTSP（在 PreviewPage 中启动）

    // [5] 异步加载菜单和文件列表（不阻塞连接）
    _loadMenusAsync();
    _loadFilesAsync();

    return true;
  }

  /// 断开连接
  Future<void> disconnect() async {
    await eventSocket.disconnect();
  }

  void dispose() {
    eventSocket.dispose();
    http.dispose();
  }
}
```

### 时序图

```
App                          设备
 │                            │
 ├─── TCP connect :9999 ─────►│  [1] 心跳通道
 │◄── OK ────────────────────┤
 ├─── S:100.0 (每500ms) ────►│
 │                            │
 ├─── GET /device/info ──────►│  [2] 基础信息（并行）
 ├─── GET /device/storage ───►│
 ├─── GET /camera/status ────►│
 │◄── JSON responses ────────┤
 │                            │
 ├─── RTSP connect :8554 ───►│  [3] 预览流
 │◄── video stream ──────────┤
 │                            │
 ├─── GET /settings/menus ───►│  [4] 菜单（异步）
 ├─── GET /media/files ──────►│  [5] 文件列表（异步）
 │◄── JSON responses ────────┤
 │                            │
 │  ✅ 连接完成，进入主界面     │
```

---

## 12. 页面与 API 对应关系

| 页面 | 需要调用的 API |
|---|---|
| **连接页** | `DeviceDetector` 检测网关 IP → TCP 心跳连接 |
| **预览页** | RTSP 播放 + `GET /camera/status` + `POST /camera/record/*` + `POST /camera/capture` |
| **相册页** | `GET /media/files` + `GET /media/thumbnail` + `GET /media/file` + `DELETE /media/file` |
| **设置页** | `GET /settings/menus` + `POST /settings/menu/value` + `POST /settings/wifi` + `POST /settings/datetime` |
| **设备信息页** | `GET /device/info` + `GET /device/storage` + `GET /device/battery` |

