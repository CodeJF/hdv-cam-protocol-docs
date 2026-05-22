# TCP 心跳客户端（端口 9999）

> [← 返回目录](./README.md)

**优先级：P0 — 连上设备后第一件事就是建立心跳**

### 4.1 协议

| 方向 | 内容 | 频率 |
|---|---|---|
| App → 设备 | `S:100.0`（UTF-8 字符串） | 每 500ms |
| 设备 → App | 事件 JSON（每行一条）| 状态变化时 |

### 4.2 事件类型完整定义

#### 录像类

| event | 含义 | 附加字段 | 你要做的 |
|---|---|---|---|
| `record_started` | 录像已开始 | `mode`, `timestamp` | 录像按钮切换为"停止"状态，显示录像时间 |
| `record_stopped` | 录像已停止 | `path`, `duration`, `size` | 录像按钮切换为"开始"状态 |
| `record_error` | 录像异常中断 | `reason` | 弹窗提示错误原因 |

`record_started` 示例：
```json
{"event": "record_started", "mode": "NormalRecordeMode", "timestamp": "2026-05-22 14:30:00"}
```

`record_stopped` 示例：
```json
{"event": "record_stopped", "path": "/mnt/DCIM/Normal/VID_20260522_143000.MP4", "duration": 120, "size": 52428800}
```

`record_error` 示例：
```json
{"event": "record_error", "reason": "sd_full"}
```

reason 取值：`sd_write_error`（写入失败）、`sd_full`（卡满）、`encoder_error`（编码器异常）

---

#### 拍照类

| event | 含义 | 附加字段 | 你要做的 |
|---|---|---|---|
| `capture_done` | 拍照成功 | `path`, `size` | 显示"拍照成功"提示，可预加载缩略图 |
| `capture_error` | 拍照失败 | `reason` | 弹窗提示错误原因 |

`capture_done` 示例：
```json
{"event": "capture_done", "path": "/mnt/DCIM/Photo/IMG_20260522_143500.jpg", "size": 3145728}
```

`capture_error` 示例：
```json
{"event": "capture_error", "reason": "sd_full"}
```

reason 取值：`sd_full`、`sd_write_error`、`sensor_error`（传感器异常）

---

#### SD 卡类

| event | 含义 | 附加字段 | 你要做的 |
|---|---|---|---|
| `sd_inserted` | SD 卡插入 | `totalMB`, `freeMB` | 刷新存储信息，隐藏"无卡"提示 |
| `sd_removed` | SD 卡拔出 | 无 | 显示"请插入 SD 卡"提示，禁用录像/拍照 |
| `sd_full` | SD 卡已满 | `freeMB` | 显示"存储空间不足"提示 |
| `sd_error` | SD 卡异常 | `reason` | 弹窗提示 SD 卡异常 |

`sd_inserted` 示例：
```json
{"event": "sd_inserted", "totalMB": 127512, "freeMB": 127000}
```

`sd_removed` 示例：
```json
{"event": "sd_removed"}
```

`sd_full` 示例：
```json
{"event": "sd_full", "freeMB": 12}
```

`sd_error` 示例：
```json
{"event": "sd_error", "reason": "fs_corrupt"}
```

reason 取值：`fs_corrupt`（文件系统损坏）、`read_error`、`write_error`

---

#### 模式类

| event | 含义 | 附加字段 | 你要做的 |
|---|---|---|---|
| `mode_changed` | 工作模式切换 | `mode`, `modeIndex` | 刷新模式 UI，重新加载菜单 |

示例：
```json
{"event": "mode_changed", "mode": "NormalCaptureMode", "modeIndex": 4}
```

---

#### 电池类

| event | 含义 | 附加字段 | 你要做的 |
|---|---|---|---|
| `battery_changed` | 电量变化 | `level`, `charging` | 更新电量图标 |
| `battery_low` | 低电量（20%/10%） | `level` | 显示低电量警告 |
| `battery_exhausted` | 即将关机（<5%） | `level`, `shutdownInSeconds` | 弹窗"设备即将关机"，保存状态 |

`battery_changed` 示例：
```json
{"event": "battery_changed", "level": 75, "charging": false}
```

`battery_low` 示例：
```json
{"event": "battery_low", "level": 10}
```

`battery_exhausted` 示例：
```json
{"event": "battery_exhausted", "level": 3, "shutdownInSeconds": 30}
```

---

#### 系统类

| event | 含义 | 附加字段 | 你要做的 |
|---|---|---|---|
| `device_ready` | 设备就绪 | `firmware`, `deviceName` | 确认连接成功，进入主界面 |
| `device_busy` | 设备忙 | `action` | 显示 loading，禁用操作按钮 |
| `device_idle` | 忙碌结束 | `action` | 隐藏 loading，恢复操作按钮 |
| `device_shutdown` | 即将关机 | `reason`, `delaySeconds` | 弹窗"设备即将关机"，断开连接 |

`device_ready` 示例：
```json
{"event": "device_ready", "firmware": "V1.0.0", "deviceName": "QZ-CAM-001"}
```

`device_busy` 示例（格式化中）：
```json
{"event": "device_busy", "action": "formatting"}
```

`device_idle` 示例（格式化完成）：
```json
{"event": "device_idle", "action": "formatting"}
```

`device_shutdown` 示例：
```json
{"event": "device_shutdown", "reason": "battery_exhausted", "delaySeconds": 3}
```

action 取值：`formatting`（格式化）、`resetting`（恢复出厂）、`upgrading`（固件升级）

reason 取值：`user_request`（用户操作）、`battery_exhausted`（电量耗尽）、`overheat`（过热保护）

---

#### 文件类

| event | 含义 | 附加字段 | 你要做的 |
|---|---|---|---|
| `file_deleted` | 文件已删除 | `path`, `reason` | 从文件列表中移除该文件 |

示例：
```json
{"event": "file_deleted", "path": "/mnt/DCIM/Normal/VID_20260520_100000.MP4", "reason": "loop_overwrite"}
```

reason 取值：`user_request`（用户删除）、`loop_overwrite`（循环覆盖）、`format`（格式化）

---

#### 事件汇总表

| event | 分类 | 含义 | 附加字段 | 你要做的 | 优先级 |
|---|---|---|---|---|---|
| `record_started` | 录像 | 开始录像 | `mode`, `timestamp` | 录像按钮切为"停止"，显示录像计时 | P0 |
| `record_stopped` | 录像 | 停止录像 | `path`, `duration`, `size` | 录像按钮切为"开始" | P0 |
| `record_error` | 录像 | 录像异常中断 | `reason` | 弹窗提示错误原因，恢复录像按钮 | P0 |
| `capture_done` | 拍照 | 拍照成功 | `path`, `size` | 显示"拍照成功"提示 | P0 |
| `capture_error` | 拍照 | 拍照失败 | `reason` | 弹窗提示错误原因 | P0 |
| `sd_inserted` | SD 卡 | SD 卡插入 | `totalMB`, `freeMB` | 刷新存储信息，隐藏"无卡"提示 | P0 |
| `sd_removed` | SD 卡 | SD 卡拔出 | 无 | 显示"请插入 SD 卡"，禁用录像/拍照 | P0 |
| `sd_full` | SD 卡 | SD 卡已满 | `freeMB` | 提示"存储空间不足" | P1 |
| `sd_error` | SD 卡 | SD 卡异常 | `reason` | 弹窗提示 SD 卡异常 | P1 |
| `mode_changed` | 模式 | 工作模式切换 | `mode`, `modeIndex` | 刷新模式 UI，重新加载菜单 | P1 |
| `battery_changed` | 电池 | 电量变化 | `level`, `charging` | 更新电量图标 | P1 |
| `battery_low` | 电池 | 低电量警告（20%/10%） | `level` | 显示低电量警告 | P1 |
| `battery_exhausted` | 电池 | 即将关机（<5%） | `level`, `shutdownInSeconds` | 弹窗"设备即将关机"，保存状态 | P1 |
| `device_ready` | 系统 | 设备就绪 | `firmware`, `deviceName` | 确认连接成功，进入主界面 | P0 |
| `device_busy` | 系统 | 设备忙（耗时操作中） | `action` | 显示 loading，禁用操作按钮 | P1 |
| `device_idle` | 系统 | 耗时操作完成 | `action` | 隐藏 loading，恢复操作按钮 | P1 |
| `device_shutdown` | 系统 | 设备即将关机 | `reason`, `delaySeconds` | 弹窗提示，断开连接 | P1 |
| `file_deleted` | 文件 | 文件已删除 | `path`, `reason` | 从文件列表移除该文件 | P1 |

### 4.3 完整实现（Dart）

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

class QZEventSocket {
  Socket? _socket;
  Timer? _heartbeatTimer;
  final _eventController = StreamController<Map<String, dynamic>>.broadcast();

  /// 事件流 — UI 层监听此流
  Stream<Map<String, dynamic>> get eventStream => _eventController.stream;

  /// 连接设备
  Future<bool> connect() async {
    try {
      _socket = await Socket.connect(
        '192.168.10.1',
        9999,
        timeout: Duration(milliseconds: 5000),
      );

      // 监听设备推送的事件
      _socket!
          .transform(utf8.decoder)
          .transform(LineSplitter())
          .listen(
            (line) => _onEvent(line),
            onError: (e) => _onError(e),
            onDone: () => _onDisconnect(),
          );

      // 启动心跳
      _startHeartbeat();
      return true;
    } catch (e) {
      return false;
    }
  }

  void _startHeartbeat() {
    _heartbeatTimer?.cancel();
    _heartbeatTimer = Timer.periodic(
      Duration(milliseconds: 500),
      (_) {
        try {
          _socket?.write('S:100.0');
        } catch (e) {
          _onError(e);
        }
      },
    );
  }

  void _onEvent(String line) {
    if (line.trim().isEmpty || line == 'OK') return;
    try {
      final event = jsonDecode(line) as Map<String, dynamic>;
      _eventController.add(event);
    } catch (e) {
      // 非 JSON 数据忽略（如心跳回复 "OK"）
    }
  }

  void _onError(dynamic error) {
    _eventController.add({'event': 'connection_error', 'error': '$error'});
  }

  void _onDisconnect() {
    _eventController.add({'event': 'disconnected'});
    // 尝试重连
    _reconnect();
  }

  Future<void> _reconnect() async {
    _heartbeatTimer?.cancel();
    await _socket?.close();
    _socket = null;

    for (int i = 0; i < 3; i++) {
      await Future.delayed(Duration(seconds: 2));
      if (await connect()) return;
    }
    _eventController.add({'event': 'reconnect_failed'});
  }

  Future<void> disconnect() async {
    _heartbeatTimer?.cancel();
    _heartbeatTimer = null;
    await _socket?.close();
    _socket = null;
  }

  void dispose() {
    disconnect();
    _eventController.close();
  }
}
```

### 4.4 Dart 事件模型

```dart
class DeviceEvent {
  final String event;
  final Map<String, dynamic> raw;

  DeviceEvent(this.raw) : event = raw['event'] ?? '';

  // 录像类
  String? get mode => raw['mode'];
  String? get timestamp => raw['timestamp'];
  String? get path => raw['path'];
  int? get duration => raw['duration'];
  int? get size => raw['size'];
  String? get reason => raw['reason'];

  // 电池类
  int? get level => raw['level'];
  bool? get charging => raw['charging'];
  int? get shutdownInSeconds => raw['shutdownInSeconds'];

  // 模式类
  int? get modeIndex => raw['modeIndex'];

  // SD 卡类
  int? get totalMB => raw['totalMB'];
  int? get freeMB => raw['freeMB'];

  // 系统类
  String? get action => raw['action'];
  String? get firmware => raw['firmware'];
  String? get deviceName => raw['deviceName'];
  int? get delaySeconds => raw['delaySeconds'];
}
```

### 4.5 UI 层监听示例

```dart
class PreviewPage extends StatefulWidget { ... }

class _PreviewPageState extends State<PreviewPage> {
  late QZEventSocket _eventSocket;

  @override
  void initState() {
    super.initState();
    _eventSocket = QZEventSocket();
    _eventSocket.connect();

    _eventSocket.eventStream.listen((raw) {
      final e = DeviceEvent(raw);
      switch (e.event) {
        // ---- 录像 ----
        case 'record_started':
          setState(() => _isRecording = true);
          break;
        case 'record_stopped':
          setState(() => _isRecording = false);
          break;
        case 'record_error':
          showError('录像异常: ${e.reason}');
          setState(() => _isRecording = false);
          break;

        // ---- 拍照 ----
        case 'capture_done':
          showSnackBar('拍照成功');
          break;
        case 'capture_error':
          showError('拍照失败: ${e.reason}');
          break;

        // ---- SD 卡 ----
        case 'sd_inserted':
          setState(() => _sdInserted = true);
          refreshStorageInfo();
          break;
        case 'sd_removed':
          setState(() => _sdInserted = false);
          showWarning('SD 卡已拔出');
          break;
        case 'sd_full':
          showWarning('存储空间不足，剩余 ${e.freeMB}MB');
          break;
        case 'sd_error':
          showError('SD 卡异常: ${e.reason}');
          break;

        // ---- 模式 ----
        case 'mode_changed':
          setState(() {
            _currentMode = e.mode;
            _currentModeIndex = e.modeIndex;
          });
          loadMenus(); // 重新加载菜单
          break;

        // ---- 电池 ----
        case 'battery_changed':
          setState(() => _batteryLevel = e.level);
          break;
        case 'battery_low':
          showWarning('电量不足: ${e.level}%');
          break;
        case 'battery_exhausted':
          showDialog('设备将在 ${e.shutdownInSeconds} 秒后关机');
          break;

        // ---- 系统 ----
        case 'device_ready':
          setState(() => _connected = true);
          break;
        case 'device_busy':
          showLoading('设备正在${_actionText(e.action)}...');
          break;
        case 'device_idle':
          hideLoading();
          showSnackBar('${_actionText(e.action)}完成');
          break;
        case 'device_shutdown':
          showDialog('设备即将关机: ${e.reason}');
          disconnect();
          break;

        // ---- 文件 ----
        case 'file_deleted':
          if (e.reason == 'loop_overwrite') {
            // 循环覆盖，静默刷新文件列表
            refreshFileList();
          }
          break;
      }
    });
  }

  String _actionText(String? action) {
    switch (action) {
      case 'formatting': return '格式化';
      case 'resetting': return '恢复出厂设置';
      case 'upgrading': return '固件升级';
      default: return '处理';
    }
  }
}
```
