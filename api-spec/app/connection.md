# 连接识别与响应格式

> [← 返回目录](./README.md)

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

