# QZ REST API — App 端开发规范

> **版本**：v1.0.0 | **日期**：2026-05-22 | **状态**：已定稿，可开工  
> **你的角色**：你是 Client 端，负责调用所有 API 接口  
> **对应文档**：嵌入式端看 [qz-api-embedded.md](./qz-api-embedded.md)

---

## 目录

- [1. 总览](#1-总览)
- [2. 连接与识别](#2-连接与识别)
- [3. 统一响应格式](#3-统一响应格式)
- [4. TCP 心跳客户端（端口 9999）](#4-tcp-心跳客户端端口-9999)
- [5. RTSP 预览接入（端口 8554）](#5-rtsp-预览接入端口-8554)
- [6. REST API — 设备信息](#6-rest-api--设备信息)
- [7. REST API — 相机控制](#7-rest-api--相机控制)
- [8. REST API — 媒体文件（在线查看、下载、删除）](#8-rest-api--媒体文件在线查看下载删除)
- [9. REST API — 设置](#9-rest-api--设置)
- [10. 工作模式定义](#10-工作模式定义)
- [11. 连接初始化完整流程](#11-连接初始化完整流程)
- [12. 页面与 API 对应关系](#12-页面与-api-对应关系)
- [13. 完整 Mock 数据集](#13-完整-mock-数据集)
- [14. 错误码与处理](#14-错误码与处理)

---

## 1. 总览

### 你要对接什么

| 服务 | 地址 | 你要做的 |
|---|---|---|
| TCP 心跳 | `192.168.10.1:9999` | 连接后每 500ms 发送心跳，接收设备事件 |
| HTTP REST API | `http://192.168.10.1:8080` | 调用 REST 接口控制相机 |
| RTSP 预览 | `rtsp://192.168.10.1:8554/ch00` | 接入实时视频流 |

### 全部接口一览

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
| GET | `/api/v1/media/thumbnail` | 缩略图 | P1 |
| GET | `/api/v1/media/file` | 在线查看/下载文件 | P0 |
| DELETE | `/api/v1/media/file` | 删除文件 | P1 |
| GET | `/api/v1/settings/menus` | 获取菜单（含翻译和当前值） | P1 |
| POST | `/api/v1/settings/menu/value` | 修改菜单选项 | P1 |
| POST | `/api/v1/settings/wifi` | 设置 Wi-Fi | P1 |
| POST | `/api/v1/settings/datetime` | 同步时间 | P1 |
| POST | `/api/v1/settings/format` | 格式化 SD 卡 | P2 |
| POST | `/api/v1/settings/reset` | 恢复出厂设置 | P2 |

---

## 2. 连接与识别

### 2.1 设备识别

App 通过手机当前连接的 **Wi-Fi 网关 IP** 判断设备类型：

| 网关 IP | 协议 |
|---|---|
| `192.168.10.1` | QZ（本文档） |
| `192.168.1.1` | MStar |
| `192.168.169.1` | YZ |

### 2.2 检测代码（Flutter）

```dart
import 'package:network_info_plus/network_info_plus.dart';

class DeviceDetector {
  static const String qzGateway = '192.168.10.1';

  static Future<bool> isQZDeviceConnected() async {
    final info = NetworkInfo();
    final gateway = await info.getWifiGatewayIP();
    return gateway == qzGateway;
  }
}
```

---

## 3. 统一响应格式

**所有 REST API 都返回这个格式。**

```json
{
  "code": 0,
  "msg": "ok",
  "data": { ... }
}
```

### 解析基类（Dart）

```dart
class ApiResponse<T> {
  final int code;
  final String msg;
  final T? data;

  bool get isSuccess => code == 0;

  ApiResponse({required this.code, required this.msg, this.data});

  factory ApiResponse.fromJson(
    Map<String, dynamic> json,
    T Function(dynamic)? fromData,
  ) {
    return ApiResponse(
      code: json['code'] as int,
      msg: json['msg'] as String,
      data: json['data'] != null && fromData != null
          ? fromData(json['data'])
          : null,
    );
  }
}
```

### HTTP 基础封装（Dart）

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

class QZHttpClient {
  static const String baseUrl = 'http://192.168.10.1:8080';
  final http.Client _client = http.Client();

  /// GET 请求
  Future<ApiResponse<T>> get<T>(
    String path, {
    Map<String, String>? params,
    T Function(dynamic)? fromData,
  }) async {
    var uri = Uri.parse('$baseUrl$path');
    if (params != null) {
      uri = uri.replace(queryParameters: params);
    }
    final response = await _client.get(uri).timeout(Duration(seconds: 5));
    final json = jsonDecode(response.body);
    return ApiResponse.fromJson(json, fromData);
  }

  /// POST 请求
  Future<ApiResponse<T>> post<T>(
    String path, {
    Map<String, dynamic>? body,
    T Function(dynamic)? fromData,
  }) async {
    final uri = Uri.parse('$baseUrl$path');
    final response = await _client.post(
      uri,
      headers: {'Content-Type': 'application/json'},
      body: body != null ? jsonEncode(body) : null,
    ).timeout(Duration(seconds: 5));
    final json = jsonDecode(response.body);
    return ApiResponse.fromJson(json, fromData);
  }

  /// DELETE 请求
  Future<ApiResponse<T>> delete<T>(
    String path, {
    Map<String, String>? params,
    T Function(dynamic)? fromData,
  }) async {
    var uri = Uri.parse('$baseUrl$path');
    if (params != null) {
      uri = uri.replace(queryParameters: params);
    }
    final response = await _client.delete(uri).timeout(Duration(seconds: 5));
    final json = jsonDecode(response.body);
    return ApiResponse.fromJson(json, fromData);
  }

  void dispose() => _client.close();
}
```

---

## 4. TCP 心跳客户端（端口 9999）

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

---

## 5. RTSP 预览接入（端口 8554）

**优先级：P0**

### 5.1 流地址

```
rtsp://192.168.10.1:8554/ch00
```

### 5.2 推荐方案

| 方案 | 包名 | 推荐度 |
|---|---|---|
| fijkplayer | `fijkplayer` | ★★★ 基于 IJK，兼容性好 |
| media_kit | `media_kit` + `media_kit_video` | ★★★ 基于 MPV，跨平台 |
| flutter_vlc_player | `flutter_vlc_player` | ★★ 稳定但包体大 |

### 5.3 接入示例（fijkplayer）

```dart
import 'package:fijkplayer/fijkplayer.dart';

class PreviewController {
  final FijkPlayer _player = FijkPlayer();

  FijkPlayer get player => _player;

  Future<void> start() async {
    // 关键参数
    await _player.setOption(FijkOption.formatCategory, "rtsp_transport", "tcp");
    await _player.setOption(FijkOption.playerCategory, "packet-buffering", 0);
    await _player.setOption(FijkOption.playerCategory, "framedrop", 1);
    await _player.setOption(FijkOption.playerCategory, "mediacodec", 1);

    await _player.setDataSource(
      'rtsp://192.168.10.1:8554/ch00',
      autoPlay: true,
    );
  }

  Future<void> stop() async {
    await _player.stop();
    await _player.reset();
  }

  void dispose() {
    _player.release();
  }
}
```

### 5.4 接入示例（media_kit）

```dart
import 'package:media_kit/media_kit.dart';
import 'package:media_kit_video/media_kit_video.dart';

class PreviewController {
  late final Player _player;
  late final VideoController videoController;

  PreviewController() {
    _player = Player();
    videoController = VideoController(_player);
  }

  Future<void> start() async {
    await _player.open(Media('rtsp://192.168.10.1:8554/ch00'));
  }

  Future<void> stop() async {
    await _player.stop();
  }

  void dispose() {
    _player.dispose();
  }
}
```

### 5.5 在页面中使用

```dart
// fijkplayer
FijkView(player: _previewController.player)

// media_kit
Video(controller: _previewController.videoController)
```

### 5.6 RTSP 预览 vs 录像回放：两个不同的播放场景

| | 实时预览 | 录像回放 |
|---|---|---|
| **播放什么** | 摄像头当前画面（直播流） | 已录好的视频文件 |
| **数据源** | RTSP `rtsp://192.168.10.1:8554/ch00` | HTTP `http://192.168.10.1:8080/api/v1/media/file?path=...` |
| **协议** | RTSP over TCP | HTTP Range 请求 |
| **能拖进度条吗** | 不能，是直播 | 能，支持 seek |
| **有缓冲条吗** | 无（实时流） | 有（边下边播） |
| **用哪个播放器** | fijkplayer / media_kit（配 RTSP 参数） | media_kit / fijkplayer（配 HTTP URL） |
| **对应页面** | PreviewPage（主界面） | VideoPlayPage（相册→点击视频） |

> HDV CAM 原版 App 也是这么区分的：IJKPlayer 负责 RTSP 实时预览，ExoPlayer 负责 HTTP 录像回放。

---

## 6. REST API — 设备信息

Base URL：`http://192.168.10.1:8080`

---

### 6.1 获取设备信息

**优先级：P0**

```
GET /api/v1/device/info
```

#### 响应示例

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

#### 数据模型

```dart
class DeviceInfo {
  final int deviceId;
  final String deviceName;
  final String model;
  final String firmware;
  final String serialNumber;

  DeviceInfo.fromJson(Map<String, dynamic> json)
      : deviceId = json['deviceId'],
        deviceName = json['deviceName'],
        model = json['model'],
        firmware = json['firmware'],
        serialNumber = json['serialNumber'];
}
```

#### 调用示例

```dart
final resp = await http.get<DeviceInfo>(
  '/api/v1/device/info',
  fromData: (d) => DeviceInfo.fromJson(d),
);
if (resp.isSuccess) {
  print('设备名: ${resp.data!.deviceName}');
  print('固件版本: ${resp.data!.firmware}');
}
```

---

### 6.2 获取存储卡信息

**优先级：P0**

```
GET /api/v1/device/storage
```

#### 响应示例

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

#### 数据模型

```dart
class StorageInfo {
  final bool inserted;
  final int totalMB;
  final int freeMB;
  final int usedMB;

  StorageInfo.fromJson(Map<String, dynamic> json)
      : inserted = json['inserted'],
        totalMB = json['totalMB'],
        freeMB = json['freeMB'],
        usedMB = json['usedMB'];

  /// 已用百分比（0-100）
  double get usedPercent => totalMB > 0 ? (usedMB / totalMB * 100) : 0;

  /// 格式化显示
  String get totalDisplay => '${(totalMB / 1024).toStringAsFixed(1)} GB';
  String get freeDisplay => '${(freeMB / 1024).toStringAsFixed(1)} GB';
}
```

#### 调用示例

```dart
final resp = await http.get<StorageInfo>(
  '/api/v1/device/storage',
  fromData: (d) => StorageInfo.fromJson(d),
);
if (resp.isSuccess) {
  final storage = resp.data!;
  if (!storage.inserted) {
    showDialog('请插入 SD 卡');
  } else {
    print('剩余: ${storage.freeDisplay}');
  }
}
```

---

### 6.3 获取电池信息

**优先级：P1**

```
GET /api/v1/device/battery
```

#### 响应示例

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

#### 数据模型

```dart
class BatteryInfo {
  final int level;
  final bool charging;
  final bool full;

  BatteryInfo.fromJson(Map<String, dynamic> json)
      : level = json['level'],
        charging = json['charging'],
        full = json['full'];

  bool get isLow => level < 20;
}
```

#### 调用示例

```dart
final resp = await http.get<BatteryInfo>(
  '/api/v1/device/battery',
  fromData: (d) => BatteryInfo.fromJson(d),
);
if (resp.isSuccess && resp.data!.isLow) {
  showWarning('电量不足: ${resp.data!.level}%');
}
```

---

### 6.4 设备健康检查

**优先级：P1**

```
GET /api/v1/device/check
```

#### 响应示例

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

#### 调用示例

```dart
final resp = await http.get('/api/v1/device/check');
if (!resp.isSuccess) {
  showError('设备离线');
}
```

---

## 7. REST API — 相机控制

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

#### 调用示例

```dart
Future<void> startRecording() async {
  final resp = await http.post('/api/v1/camera/record/start');
  if (resp.isSuccess) {
    setState(() => _isRecording = true);
  } else {
    showError('录像启动失败: ${resp.msg}');
  }
}
```

---

### 7.3 停止录像

**优先级：P0**

```
POST /api/v1/camera/record/stop
```

无请求体。

#### 调用示例

```dart
Future<void> stopRecording() async {
  final resp = await http.post('/api/v1/camera/record/stop');
  if (resp.isSuccess) {
    setState(() => _isRecording = false);
  }
}
```

---

### 7.4 拍照

**优先级：P0**

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
    "path": "/mnt/DCIM/Photo/IMG_20260522_143500.jpg"
  }
}
```

#### 调用示例

```dart
Future<void> takePhoto() async {
  final resp = await http.post('/api/v1/camera/capture');
  if (resp.isSuccess) {
    final path = resp.data['path'];
    showSnackBar('拍照成功');
    // 可以立即用 path 获取缩略图预览
  }
}
```

---

### 7.5 切换工作模式

**优先级：P1**

```
POST /api/v1/camera/mode
Content-Type: application/json

{"mode": 4}
```

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
    // 切换模式后重新加载菜单
    await loadMenus();
  }
}
```

---

### 7.6 进入/退出回放模式

**优先级：P1**

```
POST /api/v1/camera/playback/enter    ← 进入
POST /api/v1/camera/playback/exit     ← 退出
```

无请求体。

#### 调用示例

```dart
// 进入回放（查看相册前调用）
await http.post('/api/v1/camera/playback/enter');

// 退出回放（返回预览时调用）
await http.post('/api/v1/camera/playback/exit');
```

---

### 7.7 变焦控制

**优先级：P2**

```
POST /api/v1/camera/zoom
Content-Type: application/json

{"level": 5}
```

#### 调用示例

```dart
Future<void> setZoom(int level) async {
  await http.post('/api/v1/camera/zoom', body: {'level': level});
}
```

---

## 8. REST API — 媒体文件（在线查看、下载、删除）

**这一章解决"如何查看相机里的视频和照片、如何下载到手机"的问题。**

> **核心思路：先看后下。** 用户点开就能直接看（图片在线加载、视频在线播放），觉得满意再下载到手机。不需要先下载完才能看。
>
> **原理：** HDV CAM 原版 App（Android 端）也是这么做的——用 ExoPlayer 直接播放设备 HTTP URL（`http://192.168.10.1:8082/file/media/...`），ExoPlayer 通过 HTTP Range 请求边下边播，所以点开视频很快就能播、还能看到缓冲进度条。我们的做法一样，只是 URL 换成了我们的 REST API 路径。

### 完整流程

```
App 打开相册页
     │
     ▼
[1] 获取文件列表  GET /api/v1/media/files?type=photo
     │
     ▼
[2] 加载缩略图    GET /api/v1/media/thumbnail?path=xxx
     │  （网格展示，每个文件一张小图）
     ▼
用户点击某个文件
     │
     ├─ 图片 → [3a] 在线查看原图
     │         Image.network(fileUrl) 直接从 HTTP 加载
     │         不用先下载到本地，支持缩放手势
     │
     └─ 视频 → [3b] 在线播放视频
               播放器直接用 HTTP URL 播放，边下边播
               支持拖动进度条（设备端通过 Range 请求跳转）
               1-2 秒内开始播放，有缓冲进度条
     │
     ▼
用户觉得满意，点"保存到手机"
     │
     └─ [4] 下载到本地  GET /api/v1/media/file?path=xxx
              用 Dio 下载到手机相册，显示下载进度
     │
用户长按删除
     │
     └─ [5] 删除文件  DELETE /api/v1/media/file?path=xxx
```

---

### 8.1 获取文件列表

**优先级：P1**

```
GET /api/v1/media/files?type=video_normal&page=1&pageSize=20
```

#### 参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `type` | string | 是 | 见下表 |
| `page` | number | 否 | 页码（从 1 开始），默认 1 |
| `pageSize` | number | 否 | 每页数量，默认 20 |

#### type 取值

| 值 | 含义 |
|---|---|
| `video_normal` | 普通视频 |
| `video_event` | 事件视频（碰撞触发） |
| `video_parking` | 停车监控视频 |
| `photo` | 照片 |

#### 响应示例

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

#### 数据模型

```dart
class MediaFile {
  final String path;
  final String name;
  final int size;
  final String time;
  final int? duration;  // 视频时长（秒），照片为 null
  final int? width;
  final int? height;

  MediaFile.fromJson(Map<String, dynamic> json)
      : path = json['path'],
        name = json['name'],
        size = json['size'],
        time = json['time'],
        duration = json['duration'],
        width = json['width'],
        height = json['height'];

  bool get isVideo => name.toLowerCase().endsWith('.mp4') ||
                      name.toLowerCase().endsWith('.mov');
  bool get isPhoto => name.toLowerCase().endsWith('.jpg');

  /// 缩略图 URL
  String get thumbnailUrl =>
    'http://192.168.10.1:8080/api/v1/media/thumbnail?path=${Uri.encodeComponent(path)}';

  /// 文件下载 URL
  String get fileUrl =>
    'http://192.168.10.1:8080/api/v1/media/file?path=${Uri.encodeComponent(path)}';

  /// 格式化文件大小
  String get sizeDisplay {
    if (size > 1024 * 1024 * 1024) return '${(size / 1024 / 1024 / 1024).toStringAsFixed(1)} GB';
    if (size > 1024 * 1024) return '${(size / 1024 / 1024).toStringAsFixed(1)} MB';
    return '${(size / 1024).toStringAsFixed(1)} KB';
  }

  /// 格式化时长
  String get durationDisplay {
    if (duration == null) return '';
    final m = duration! ~/ 60;
    final s = duration! % 60;
    return '${m.toString().padLeft(2, '0')}:${s.toString().padLeft(2, '0')}';
  }
}

class MediaFileList {
  final int total;
  final int page;
  final int pageSize;
  final List<MediaFile> files;

  MediaFileList.fromJson(Map<String, dynamic> json)
      : total = json['total'],
        page = json['page'],
        pageSize = json['pageSize'],
        files = (json['files'] as List).map((f) => MediaFile.fromJson(f)).toList();

  bool get hasMore => page * pageSize < total;
}
```

#### 调用示例

```dart
/// 获取普通视频列表
Future<MediaFileList> getVideoList({int page = 1}) async {
  final resp = await http.get<MediaFileList>(
    '/api/v1/media/files',
    params: {
      'type': 'video_normal',
      'page': '$page',
      'pageSize': '20',
    },
    fromData: (d) => MediaFileList.fromJson(d),
  );
  return resp.data!;
}

/// 获取照片列表
Future<MediaFileList> getPhotoList({int page = 1}) async {
  final resp = await http.get<MediaFileList>(
    '/api/v1/media/files',
    params: {
      'type': 'photo',
      'page': '$page',
      'pageSize': '20',
    },
    fromData: (d) => MediaFileList.fromJson(d),
  );
  return resp.data!;
}
```

---

### 8.2 获取缩略图

**优先级：P1**

```
GET /api/v1/media/thumbnail?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

返回 JPEG 图片二进制数据（不是 JSON）。

#### 在列表中显示缩略图

```dart
// 方式 1：直接用 Image.network
Image.network(
  file.thumbnailUrl,
  width: 120,
  height: 90,
  fit: BoxFit.cover,
  errorBuilder: (_, __, ___) => Icon(Icons.broken_image),
)

// 方式 2：用 cached_network_image 缓存（推荐）
CachedNetworkImage(
  imageUrl: file.thumbnailUrl,
  width: 120,
  height: 90,
  fit: BoxFit.cover,
  placeholder: (_, __) => CircularProgressIndicator(),
  errorWidget: (_, __, ___) => Icon(Icons.broken_image),
)
```

---

### 8.3 在线查看图片

**优先级：P1**

```
GET /api/v1/media/file?path=/mnt/DCIM/Photo/IMG_20260522_143500.jpg
```

返回图片二进制数据（`image/jpeg`）。直接用 `Image.network` 加载，不需要先下载到本地。

#### 全屏查看图片

```dart
class PhotoViewPage extends StatelessWidget {
  final MediaFile file;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(file.name),
        actions: [
          // 保存到手机按钮
          IconButton(
            icon: Icon(Icons.download),
            onPressed: () => _saveToPhone(context, file),
          ),
        ],
      ),
      body: InteractiveViewer(
        child: Image.network(
          file.fileUrl,  // 直接用 HTTP URL，在线加载
          fit: BoxFit.contain,
          loadingBuilder: (_, child, progress) {
            if (progress == null) return child;
            return Center(child: CircularProgressIndicator(
              value: progress.expectedTotalBytes != null
                  ? progress.cumulativeBytesLoaded / progress.expectedTotalBytes!
                  : null,
            ));
          },
        ),
      ),
    );
  }
}
```

---

### 8.4 在线播放视频

**优先级：P0**

> **这是相册功能最重要的能力。** 用户点开视频，1-2 秒内开始播放，有缓冲进度条，可以拖动进度条跳转。不需要等整个视频下载完。
>
> **原理：** 播放器直接用设备的 HTTP URL 作为数据源，通过 Range 请求分段获取数据，边下边播。HDV CAM 原版 App 中 Android 端用的是 ExoPlayer，我们用 `media_kit`（基于 MPV）或 `fijkplayer`（基于 IJK），原理一样。

```
播放器的数据源 URL:
http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 方案对比

| 方案 | 包名 | 优势 | 劣势 |
|---|---|---|---|
| **media_kit（推荐）** | `media_kit` | 基于 MPV，跨平台，支持 HTTP/RTSP，维护活跃 | 相对较新 |
| fijkplayer | `fijkplayer` | 基于 IJK（与 HDV CAM 原版同源） | 维护不活跃 |
| flutter_vlc_player | `flutter_vlc_player` | 稳定，支持 RTSP | 包体大 |

#### 用 media_kit 在线播放（推荐）

```dart
import 'package:media_kit/media_kit.dart';
import 'package:media_kit_video/media_kit_video.dart';

class VideoPlayPage extends StatefulWidget {
  final MediaFile file;
  const VideoPlayPage({required this.file});

  @override
  State<VideoPlayPage> createState() => _VideoPlayPageState();
}

class _VideoPlayPageState extends State<VideoPlayPage> {
  late final Player _player;
  late final VideoController _controller;

  @override
  void initState() {
    super.initState();
    _player = Player();
    _controller = VideoController(_player);

    // 直接用 HTTP URL 播放，不需要先下载
    _player.open(Media(widget.file.fileUrl));
  }

  @override
  void dispose() {
    _player.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.file.name),
        actions: [
          // 下载到手机
          IconButton(
            icon: Icon(Icons.download),
            onPressed: () => _saveToPhone(context, widget.file),
          ),
        ],
      ),
      body: Video(
        controller: _controller,
        // media_kit 内置了播放/暂停、进度条、全屏等控件
      ),
    );
  }
}
```

#### 用 fijkplayer 在线播放

```dart
import 'package:fijkplayer/fijkplayer.dart';

class VideoPlayPage extends StatefulWidget {
  final MediaFile file;
  const VideoPlayPage({required this.file});

  @override
  State<VideoPlayPage> createState() => _VideoPlayPageState();
}

class _VideoPlayPageState extends State<VideoPlayPage> {
  final FijkPlayer _player = FijkPlayer();

  @override
  void initState() {
    super.initState();
    _initPlayer();
  }

  Future<void> _initPlayer() async {
    // 开启硬解码
    await _player.setOption(FijkOption.playerCategory, "mediacodec", 1);
    // 减少缓冲延迟
    await _player.setOption(FijkOption.playerCategory, "packet-buffering", 0);
    await _player.setOption(FijkOption.playerCategory, "framedrop", 1);

    // 直接用 HTTP URL 播放
    await _player.setDataSource(
      widget.file.fileUrl,
      autoPlay: true,
    );
  }

  @override
  void dispose() {
    _player.release();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.file.name),
        actions: [
          IconButton(
            icon: Icon(Icons.download),
            onPressed: () => _saveToPhone(context, widget.file),
          ),
        ],
      ),
      body: FijkView(
        player: _player,
        // 内置播放控件（进度条、缓冲条、播放/暂停）
      ),
    );
  }
}
```

---

### 8.5 下载文件到手机

**优先级：P1**

> 用户在线看完觉得满意，点"保存到手机"按钮时才下载。图片和视频都走这个流程。

```
GET /api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 用 Dio 下载（支持进度显示）

```dart
import 'package:dio/dio.dart';
import 'package:path_provider/path_provider.dart';

class FileDownloader {
  final Dio _dio = Dio();

  /// 下载文件到本地
  Future<String> download(
    MediaFile file, {
    void Function(int received, int total)? onProgress,
  }) async {
    final dir = await getApplicationDocumentsDirectory();
    final savePath = '${dir.path}/${file.name}';

    await _dio.download(
      file.fileUrl,
      savePath,
      onReceiveProgress: onProgress,
    );

    return savePath;
  }
}
```

#### 下载按钮示例

```dart
Future<void> _saveToPhone(BuildContext context, MediaFile file) async {
  final downloader = FileDownloader();

  // 显示下载进度对话框
  showDialog(
    context: context,
    barrierDismissible: false,
    builder: (_) => _DownloadProgressDialog(
      file: file,
      downloader: downloader,
    ),
  );

  final localPath = await downloader.download(
    file,
    onProgress: (received, total) {
      final percent = (received / total * 100).toStringAsFixed(0);
      debugPrint('下载进度: $percent%');
    },
  );

  Navigator.pop(context); // 关闭进度对话框

  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(content: Text('已保存到: ${file.name}')),
  );
}
```

---

### 8.6 删除文件

**优先级：P1**

```
DELETE /api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 调用示例

```dart
Future<bool> deleteFile(MediaFile file) async {
  final resp = await http.delete(
    '/api/v1/media/file',
    params: {'path': file.path},
  );
  if (resp.isSuccess) {
    setState(() => _files.remove(file));
    return true;
  }
  showError('删除失败: ${resp.msg}');
  return false;
}
```

---

### 8.7 相册页完整示例

```dart
class AlbumPage extends StatefulWidget {
  @override
  State<AlbumPage> createState() => _AlbumPageState();
}

class _AlbumPageState extends State<AlbumPage> with SingleTickerProviderStateMixin {
  late TabController _tabController;
  final _types = ['video_normal', 'video_event', 'video_parking', 'photo'];
  final _titles = ['普通视频', '事件视频', '停车视频', '照片'];
  Map<String, MediaFileList?> _fileLists = {};

  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: 4, vsync: this);
    _loadFiles('video_normal');
  }

  Future<void> _loadFiles(String type) async {
    final resp = await http.get<MediaFileList>(
      '/api/v1/media/files',
      params: {'type': type, 'page': '1', 'pageSize': '20'},
      fromData: (d) => MediaFileList.fromJson(d),
    );
    if (resp.isSuccess) {
      setState(() => _fileLists[type] = resp.data);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('相册'),
        bottom: TabBar(
          controller: _tabController,
          tabs: _titles.map((t) => Tab(text: t)).toList(),
          onTap: (i) => _loadFiles(_types[i]),
        ),
      ),
      body: TabBarView(
        controller: _tabController,
        children: _types.map((type) {
          final list = _fileLists[type];
          if (list == null) return Center(child: CircularProgressIndicator());
          if (list.files.isEmpty) return Center(child: Text('暂无文件'));

          return GridView.builder(
            gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
              crossAxisCount: 3,
              childAspectRatio: 4 / 3,
            ),
            itemCount: list.files.length,
            itemBuilder: (_, i) {
              final file = list.files[i];
              return GestureDetector(
                onTap: () => _openFile(file),
                child: Stack(
                  fit: StackFit.expand,
                  children: [
                    // 缩略图
                    Image.network(file.thumbnailUrl, fit: BoxFit.cover),
                    // 视频时长标签
                    if (file.isVideo)
                      Positioned(
                        bottom: 4, right: 4,
                        child: Container(
                          padding: EdgeInsets.symmetric(horizontal: 4, vertical: 2),
                          color: Colors.black54,
                          child: Text(file.durationDisplay,
                            style: TextStyle(color: Colors.white, fontSize: 12)),
                        ),
                      ),
                  ],
                ),
              );
            },
          );
        }).toList(),
      ),
    );
  }

  void _openFile(MediaFile file) {
    if (file.isPhoto) {
      // 在线查看图片
      Navigator.push(context,
        MaterialPageRoute(builder: (_) => PhotoViewPage(file: file)));
    } else {
      // 在线播放视频（直接用 HTTP URL，不用先下载）
      Navigator.push(context,
        MaterialPageRoute(builder: (_) => VideoPlayPage(file: file)));
    }
  }
}
```

---

## 9. REST API — 设置

---

### 9.1 获取菜单

**优先级：P1**

```
GET /api/v1/settings/menus?lang=zh-CN
```

**这个接口一次性返回所有菜单数据：菜单定义 + 翻译 + 当前值。你不需要多次请求再拼装。**

#### 响应示例

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
          { "index": 1, "id": "en", "title": "English" }
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

#### 数据模型

```dart
class MenuOption {
  final int index;
  final String id;
  final String title;

  MenuOption.fromJson(Map<String, dynamic> json)
      : index = json['index'],
        id = json['id'],
        title = json['title'];
}

class MenuItem {
  final String id;
  final String title;
  final int index;
  int currentValue;
  final List<MenuOption> options;
  final String? type;    // "input", "datetime", "action", 或 null（选项类型）
  final String? value;   // input/datetime 的当前文本值

  MenuItem.fromJson(Map<String, dynamic> json)
      : id = json['id'],
        title = json['title'],
        index = json['index'],
        currentValue = json['currentValue'],
        options = (json['options'] as List).map((o) => MenuOption.fromJson(o)).toList(),
        type = json['type'],
        value = json['value'];

  /// 是否是选项类型（有下拉选择）
  bool get isOptionType => options.isNotEmpty;

  /// 当前选中的选项标题
  String? get currentOptionTitle {
    if (currentValue < 0 || currentValue >= options.length) return null;
    return options[currentValue].title;
  }
}

class MenuData {
  final String currentMode;
  final int currentModeIndex;
  final List<MenuItem> modeMenus;
  final List<MenuItem> systemMenus;

  MenuData.fromJson(Map<String, dynamic> json)
      : currentMode = json['currentMode'],
        currentModeIndex = json['currentModeIndex'],
        modeMenus = (json['modeMenus'] as List).map((m) => MenuItem.fromJson(m)).toList(),
        systemMenus = (json['systemMenus'] as List).map((m) => MenuItem.fromJson(m)).toList();
}
```

#### 调用示例

```dart
Future<MenuData> loadMenus() async {
  final resp = await http.get<MenuData>(
    '/api/v1/settings/menus',
    params: {'lang': 'zh-CN'},
    fromData: (d) => MenuData.fromJson(d),
  );
  return resp.data!;
}
```

---

### 9.2 修改菜单选项

**优先级：P1**

```
POST /api/v1/settings/menu/value
Content-Type: application/json

{"id": "rec_resolution", "value": 1}
```

#### 调用示例

```dart
Future<void> changeMenuValue(MenuItem menu, int newValue) async {
  final resp = await http.post(
    '/api/v1/settings/menu/value',
    body: {'id': menu.id, 'value': newValue},
  );
  if (resp.isSuccess) {
    setState(() => menu.currentValue = newValue);
  }
}
```

---

### 9.3 设置 Wi-Fi

**优先级：P1**

```
POST /api/v1/settings/wifi
Content-Type: application/json

{"ssid": "MyCamera", "password": "88888888"}
```

#### 响应示例

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "ssid": "MyCamera",
    "password": "88888888",
    "reconnectRequired": true
  }
}
```

#### 调用示例

```dart
Future<void> setWifi(String ssid, String password) async {
  final resp = await http.post(
    '/api/v1/settings/wifi',
    body: {'ssid': ssid, 'password': password},
  );
  if (resp.isSuccess && resp.data['reconnectRequired'] == true) {
    showDialog('Wi-Fi 已修改，请重新连接设备热点');
  }
}
```

---

### 9.4 同步时间

**优先级：P1**

```
POST /api/v1/settings/datetime
Content-Type: application/json

{"datetime": "2026-05-22 14:30:00"}
```

#### 调用示例

```dart
Future<void> syncTime() async {
  final now = DateTime.now();
  final formatted = '${now.year}-${_pad(now.month)}-${_pad(now.day)} '
      '${_pad(now.hour)}:${_pad(now.minute)}:${_pad(now.second)}';

  await http.post(
    '/api/v1/settings/datetime',
    body: {'datetime': formatted},
  );
}

String _pad(int n) => n.toString().padLeft(2, '0');
```

---

### 9.5 格式化 SD 卡

**优先级：P2**

```
POST /api/v1/settings/format
```

#### 调用示例

```dart
Future<void> formatCard() async {
  final confirm = await showConfirmDialog('确定要格式化 SD 卡？所有文件将被删除。');
  if (!confirm) return;

  final resp = await http.post('/api/v1/settings/format');
  if (resp.isSuccess) {
    showSnackBar('格式化完成');
  }
}
```

---

### 9.6 恢复出厂设置

**优先级：P2**

```
POST /api/v1/settings/reset
```

#### 调用示例

```dart
Future<void> factoryReset() async {
  final confirm = await showConfirmDialog('确定要恢复出厂设置？');
  if (!confirm) return;

  final resp = await http.post('/api/v1/settings/reset');
  if (resp.isSuccess) {
    showDialog('已恢复出厂设置，请重新连接设备');
  }
}
```

---

### 9.7 设置页完整示例

```dart
class SettingPage extends StatefulWidget {
  @override
  State<SettingPage> createState() => _SettingPageState();
}

class _SettingPageState extends State<SettingPage> {
  MenuData? _menuData;

  @override
  void initState() {
    super.initState();
    _loadMenus();
  }

  Future<void> _loadMenus() async {
    final data = await loadMenus();
    setState(() => _menuData = data);
  }

  @override
  Widget build(BuildContext context) {
    if (_menuData == null) return Center(child: CircularProgressIndicator());

    return ListView(
      children: [
        // 当前模式菜单
        _buildSection('当前模式设置', _menuData!.modeMenus),
        // 系统菜单
        _buildSection('系统设置', _menuData!.systemMenus),
      ],
    );
  }

  Widget _buildSection(String title, List<MenuItem> menus) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Padding(
          padding: EdgeInsets.all(16),
          child: Text(title, style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
        ),
        ...menus.map((menu) => _buildMenuItem(menu)),
      ],
    );
  }

  Widget _buildMenuItem(MenuItem menu) {
    // 选项类型 → 弹出选择器
    if (menu.isOptionType) {
      return ListTile(
        title: Text(menu.title),
        subtitle: Text(menu.currentOptionTitle ?? ''),
        onTap: () => _showOptionPicker(menu),
      );
    }
    // 输入类型
    if (menu.type == 'input') {
      return ListTile(
        title: Text(menu.title),
        subtitle: Text(menu.value ?? ''),
        onTap: () => _showInputDialog(menu),
      );
    }
    // 时间类型
    if (menu.type == 'datetime') {
      return ListTile(
        title: Text(menu.title),
        subtitle: Text(menu.value ?? ''),
        onTap: () => syncTime(),
      );
    }
    // 动作类型
    if (menu.type == 'action') {
      return ListTile(
        title: Text(menu.title),
        onTap: () => _executeAction(menu),
      );
    }
    return SizedBox.shrink();
  }

  void _showOptionPicker(MenuItem menu) {
    showModalBottomSheet(
      context: context,
      builder: (_) => ListView(
        shrinkWrap: true,
        children: menu.options.map((opt) => ListTile(
          title: Text(opt.title),
          trailing: opt.index == menu.currentValue ? Icon(Icons.check) : null,
          onTap: () {
            changeMenuValue(menu, opt.index);
            Navigator.pop(context);
          },
        )).toList(),
      ),
    );
  }
}
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

### 10.2 Dart 枚举

```dart
enum WorkMode {
  normalRecord(0, 'NormalRecordeMode', '普通录像'),
  slowRecord(1, 'SlowRecordeMode', '慢动作'),
  loopRecord(2, 'LoopRecordeMode', '循环录像'),
  timeLapse(3, 'TimeLapseMode', '延时摄影'),
  normalCapture(4, 'NormalCaptureMode', '普通拍照'),
  autoCapture(5, 'AutoCaptureMode', '自动拍照'),
  continueCapture(6, 'ContinueCaptureMode', '连拍'),
  timingCapture(7, 'TimingCaptureMode', '定时拍照');

  final int index;
  final String rawName;
  final String displayName;

  const WorkMode(this.index, this.rawName, this.displayName);

  bool get isRecordMode => index <= 3;
  bool get isCaptureMode => index >= 4;

  static WorkMode fromIndex(int i) =>
      WorkMode.values.firstWhere((m) => m.index == i, orElse: () => normalRecord);

  static WorkMode fromName(String name) =>
      WorkMode.values.firstWhere((m) => m.rawName == name, orElse: () => normalRecord);
}
```

### 10.3 模式切换后的处理

切换模式后，菜单内容会变化。你需要重新请求菜单：

```dart
Future<void> onModeChanged(int newModeIndex) async {
  // 1. 切换模式
  await http.post('/api/v1/camera/mode', body: {'mode': newModeIndex});

  // 2. 重新加载菜单（设备返回的 modeMenus 会自动变化）
  final menus = await loadMenus();
  setState(() => _menuData = menus);
}
```

---

## 11. 连接初始化完整流程

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

---

## 13. 完整 Mock 数据集

在设备端未完成前，你可以用以下 Mock 数据自测 App。

### 13.1 用 json-server 搭建 Mock

安装：
```bash
npm install -g json-server
```

创建 `mock-db.json`：

```json
{
  "device_info": {
    "code": 0,
    "msg": "ok",
    "data": {
      "deviceId": 1,
      "deviceName": "QZ-CAM-001",
      "model": "QZ-4K",
      "firmware": "V1.0.0",
      "serialNumber": "SN20260001"
    }
  },
  "device_storage": {
    "code": 0,
    "msg": "ok",
    "data": {
      "inserted": true,
      "totalMB": 127512,
      "freeMB": 92341,
      "usedMB": 35171
    }
  },
  "device_battery": {
    "code": 0,
    "msg": "ok",
    "data": {
      "level": 85,
      "charging": false,
      "full": false
    }
  },
  "camera_status": {
    "code": 0,
    "msg": "ok",
    "data": {
      "recording": false,
      "mode": "NormalRecordeMode",
      "modeIndex": 0,
      "rtspUrl": "rtsp://192.168.10.1:8554/ch00"
    }
  }
}
```

### 13.2 文件列表 Mock（普通视频）

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "total": 3,
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
      },
      {
        "path": "/mnt/DCIM/Normal/VID_20260521_100000.MP4",
        "name": "VID_20260521_100000.MP4",
        "size": 104857600,
        "time": "2026-05-21 10:00:00",
        "duration": 300,
        "width": 3840,
        "height": 2160
      }
    ]
  }
}
```

### 13.3 文件列表 Mock（照片）

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "total": 2,
    "page": 1,
    "pageSize": 20,
    "files": [
      {
        "path": "/mnt/DCIM/Photo/IMG_20260522_143500.jpg",
        "name": "IMG_20260522_143500.jpg",
        "size": 3145728,
        "time": "2026-05-22 14:35:00",
        "duration": 0,
        "width": 4032,
        "height": 3024
      },
      {
        "path": "/mnt/DCIM/Photo/IMG_20260522_142000.jpg",
        "name": "IMG_20260522_142000.jpg",
        "size": 2621440,
        "time": "2026-05-22 14:20:00",
        "duration": 0,
        "width": 4032,
        "height": 3024
      }
    ]
  }
}
```

### 13.4 菜单 Mock

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
          { "index": 1, "id": "en", "title": "English" }
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

---

## 14. 错误码与处理

### 14.1 错误码表

| code | 含义 | 你的处理 |
|---|---|---|
| `0` | 成功 | 正常处理 |
| `-1` | 通用失败 | 显示 `msg` 内容 |
| `-2` | 参数错误 | 检查请求参数 |
| `-3` | SD 卡未插入 | 提示用户插入 SD 卡 |
| `-4` | SD 卡已满 | 提示空间不足 |
| `-5` | 文件不存在 | 刷新文件列表 |
| `-6` | 设备忙 | 稍后重试 |
| `-7` | 模式不支持 | 检查模式编号 |

### 14.2 统一错误处理

```dart
void handleApiError(ApiResponse resp) {
  switch (resp.code) {
    case -3:
      showDialog('请插入 SD 卡后再试');
      break;
    case -4:
      showDialog('存储空间不足，请清理文件');
      break;
    case -5:
      showSnackBar('文件已不存在');
      refreshFileList();
      break;
    case -6:
      showSnackBar('设备忙，请稍后重试');
      break;
    default:
      showSnackBar('操作失败: ${resp.msg}');
  }
}
```

### 14.3 网络超时处理

```dart
try {
  final resp = await http.get('/api/v1/camera/status');
  // ...
} on TimeoutException {
  showDialog('连接超时，请检查 Wi-Fi 是否已连接到相机');
} on SocketException {
  showDialog('无法连接设备，请确认已连接相机热点');
}
```

### 14.4 推荐 Flutter 依赖

```yaml
dependencies:
  http: ^1.2.0                    # HTTP 请求
  dio: ^5.4.0                     # 文件下载（支持进度）
  network_info_plus: ^5.0.0       # Wi-Fi 网关检测
  fijkplayer: ^0.11.0             # RTSP 播放（二选一）
  # media_kit: ^1.1.0             # RTSP 播放（二选一）
  cached_network_image: ^3.3.0    # 缩略图缓存
  provider: ^6.1.0                # 状态管理
  path_provider: ^2.1.0           # 本地存储路径
  permission_handler: ^11.3.0     # 权限管理
```
