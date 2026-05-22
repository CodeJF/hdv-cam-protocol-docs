# QZ 协议 Flutter App 开发指南

## 1. 文档目的

本文面向使用 Flutter 开发 QZ 兼容相机控制 App 的客户端工程师。内容基于 HDV CAM Android APK 和 iOS IPA 的逆向分析，涵盖协议接入、数据模型、UI 对接和推荐架构。

---

## 2. 协议总览

QZ 设备通过 Wi-Fi 热点直连，App 作为 STA 接入。通信分三条链路：

| 链路 | 地址 | 用途 |
|---|---|---|
| HTTP | `http://192.168.10.1:8082` | 命令控制、状态读取、资源文件 |
| RTSP | `rtsp://192.168.10.1:8554/ch00` | 实时预览视频流 |
| TCP Socket | `192.168.10.1:9999` | 设备事件推送、保活 |

设备识别依据：手机当前连接的 Wi-Fi 网关为 `192.168.10.1`。

---

## 3. 推荐架构

### 3.1 分层设计

```
┌──────────────────────────────────────────┐
│                 UI 层                      │
│  ConnectPage │ PreviewPage │ AlbumPage    │
│  SettingPage │ FileViewPage               │
├──────────────────────────────────────────┤
│              ViewModel / Provider          │
│  DeviceViewModel │ AlbumViewModel          │
│  SettingViewModel │ PreviewViewModel       │
├──────────────────────────────────────────┤
│            协议抽象层 (CameraClient)        │
│  ┌────────────────────────────────────┐  │
│  │ QZCameraClient implements          │  │
│  │   CameraClient                     │  │
│  └────────────────────────────────────┘  │
├──────────────────────────────────────────┤
│             传输层                         │
│  HttpService │ RtspPlayer │ TcpSocket     │
└──────────────────────────────────────────┘
```

### 3.2 统一相机接口

参考原生 App 的 `Case` 抽象模式，定义统一接口：

```dart
abstract class CameraClient {
  /// 连接设备
  Future<bool> connect();
  
  /// 断开连接
  Future<void> disconnect();
  
  /// 获取设备基础信息
  Future<DeviceInfo> getDeviceInfo();
  
  /// 获取 SD 卡信息
  Future<SdCardInfo> getSdCardInfo();
  
  /// 获取录像状态
  Future<bool> isRecording();
  
  /// 获取当前工作模式
  Future<WorkMode> getCurrentMode();
  
  /// 获取电量
  Future<BatteryInfo> getBatteryInfo();
  
  /// 开始录像
  Future<bool> startRecording();
  
  /// 停止录像
  Future<bool> stopRecording();
  
  /// 拍照
  Future<bool> capture();
  
  /// 切换工作模式
  Future<bool> switchMode(int modeIndex);
  
  /// 进入/退出回放模式
  Future<bool> setPlaybackMode(bool enter);
  
  /// 获取文件列表
  Future<MediaFileList> getMediaFiles(MediaType type, int page);
  
  /// 删除文件
  Future<bool> deleteFile(String filePath);
  
  /// 获取菜单定义
  Future<List<MenuGroup>> getMenus();
  
  /// 获取菜单当前值
  Future<MenuCurrentValues> getMenuValues(bool isSystem);
  
  /// 设置 Wi-Fi
  Future<bool> setWifiName(String ssid);
  Future<bool> setWifiPassword(String pwd);
  
  /// 同步时间
  Future<bool> syncTime(DateTime dateTime);
  
  /// 格式化 SD 卡
  Future<bool> formatCard();
  
  /// 获取实时流地址
  String getLiveStreamUrl();
  
  /// 事件流
  Stream<DeviceEvent> get eventStream;
}
```

---

## 4. 数据模型定义

### 4.1 设备信息

```dart
class DeviceInfo {
  final int deviceId;
  final String deviceName;
  final String software;
  final String status;
  
  factory DeviceInfo.fromJson(Map<String, dynamic> json) {
    return DeviceInfo(
      deviceId: _parseInt(json['deviceId']),
      deviceName: json['deviceName'] ?? '',
      software: json['software'] ?? '',
      status: json['status'] ?? '0',
    );
  }
}
```

### 4.2 SD 卡信息

```dart
class SdCardInfo {
  final bool hasCard;
  final double totalMB;
  final double freeMB;
  
  factory SdCardInfo.fromJson(Map<String, dynamic> json) {
    return SdCardInfo(
      hasCard: json['disk_status'] == '1',
      totalMB: double.tryParse(json['capacity'] ?? '0') ?? 0,
      freeMB: double.tryParse(json['free_space'] ?? '0') ?? 0,
    );
  }
}
```

### 4.3 录像状态

```dart
class RecordStatus {
  final bool isRecording;
  
  factory RecordStatus.fromJson(Map<String, dynamic> json) {
    return RecordStatus(
      isRecording: json['RecodStatus'] == '1',  // 注意：是 Recod 不是 Record
    );
  }
}
```

### 4.4 工作模式

```dart
enum WorkMode {
  normalRecord('NormalRecordeMode'),
  slowRecord('SlowRecordeMode'),
  loopRecord('LoopRecordeMode'),
  timeLapse('TimeLapseMode'),
  normalCapture('NormalCaptureMode'),
  autoCapture('AutoCaptureMode'),
  continueCapture('ContinueCaptureMode'),
  timingCapture('TimingCaptureMode');
  
  final String rawValue;
  const WorkMode(this.rawValue);
  
  static WorkMode fromString(String s) {
    return WorkMode.values.firstWhere(
      (m) => m.rawValue == s,
      orElse: () => WorkMode.normalRecord,
    );
  }
  
  bool get isRecordMode => [
    normalRecord, slowRecord, loopRecord, timeLapse
  ].contains(this);
  
  bool get isCaptureMode => !isRecordMode;
  
  /// 对应 setting_keys.xml 中的 array name
  String get settingKeysName {
    switch (this) {
      case normalRecord: return 'record_normal_setting_keys';
      case slowRecord: return 'record_slow_setting_keys';
      case loopRecord: return 'record_loop_setting_keys';
      case timeLapse: return 'record_timelapse_setting_keys';
      case normalCapture: return 'photo_normal_setting_keys';
      case autoCapture: return 'photo_auto_setting_keys';
      case continueCapture: return 'photo_burst_setting_keys';
      case timingCapture: return 'photo_time_setting_keys';
    }
  }

  /// Camera.Menu 分辨率 key（用于分辨率解析）
  String get resolutionMenuKey {
    switch (this) {
      case normalRecord: return 'Camera.Menu.NRecRes';
      case slowRecord: return 'Camera.Menu.SRecType';
      case loopRecord: return 'Camera.Menu.LRecRes';
      case timeLapse: return 'Camera.Menu.TLRecRes';
      case normalCapture: return 'Camera.Menu.NPhotoRes';
      case autoCapture: return 'Camera.Menu.APhotoRes';
      case continueCapture: return 'Camera.Menu.CPhotoRes';
      case timingCapture: return 'Camera.Menu.TPhotoRes';
    }
  }
}
```

### 4.5 媒体文件

```dart
enum MediaType {
  photo('Photo'),
  normalVideo('Normal'),
  eventVideo('Event'),
  parkingVideo('Parking');
  
  final String property;
  const MediaType(this.property);
}

class MediaFile {
  final String name;       // 完整路径 /mnt/DCIM/Normal/xxx.mp4
  final int size;          // 字节
  final DateTime time;     // 拍摄时间
  final int? duration;     // 视频时长（秒），图片为 null
  
  String get fileName => name.split('/').last;
  
  bool get isVideo => ['.mp4', '.mov', '.ts']
      .any((ext) => name.toLowerCase().endsWith(ext));
  
  /// 构造文件下载 URL
  String get downloadUrl =>
      'http://192.168.10.1:8082/file/media${name.startsWith('/') ? name : '/$name'}';
  
  /// 构造缩略图 URL
  String get thumbnailUrl =>
      'http://192.168.10.1:8082/thumb${name.startsWith('/') ? name : '/$name'}.jpg';
}

class MediaFileList {
  final List<MediaFile> files;
  final int totalCount;
  final bool hasMore;
}
```

### 4.6 菜单模型

```dart
class QZMenu {
  final String id;         // 如 "rec_resolution"
  final int index;         // 在 setting_keys 中的序号
  final String title;      // 显示标题（来自翻译资源）
  final bool sysKeys;      // 是否系统菜单
  final List<QZMenuItem> items;  // 枚举选项
  int currentValueIndex;   // 当前值（来自 0x7d2/0x7d6）
}

class QZMenuItem {
  final String id;         // 如 "1080p30"
  final String title;      // 如 "1080P 30FPS"
  final int index;         // 选项序号
}

class MenuCurrentValues {
  final int deviceId;
  final String deviceName;
  final String software;
  final List<MenuValueItem> info;
}

class MenuValueItem {
  final int index;         // 对应 QZMenu.index
  final int value;         // 当前选中的 QZMenuItem.index
}
```

### 4.7 电量信息

```dart
class BatteryInfo {
  final int level;         // 0-100
  final bool isFull;
  
  factory BatteryInfo.fromJson(Map<String, dynamic> json) {
    return BatteryInfo(
      level: int.tryParse(json['level'] ?? '0') ?? 0,
      isFull: json['full'] == '1',
    );
  }
}
```

### 4.8 设备事件

```dart
enum DeviceEventType {
  connected,
  disconnected,
  error,
  recordingStarted,
  recordingStopped,
  sdCardChanged,
  modeChanged,
}

class DeviceEvent {
  final DeviceEventType type;
  final String? rawData;
}
```

---

## 5. HTTP 协议层实现

### 5.1 QZ HTTP Client

```dart
class QZHttpClient {
  static const String _baseUrl = 'http://192.168.10.1:8082';
  final http.Client _client;
  
  /// 读状态命令
  Future<Map<String, dynamic>> getDeviceInfo(int cmd) async {
    final url = '$_baseUrl/api/getdeviceinfo/?custom=1&cmd=0x${cmd.toRadixString(16)}';
    final response = await _client.get(Uri.parse(url))
        .timeout(const Duration(seconds: 5));
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    }
    throw QZProtocolException('GET cmd=0x${cmd.toRadixString(16)} failed: ${response.statusCode}');
  }
  
  /// 写状态命令（带 par 参数）
  Future<Map<String, dynamic>> setDeviceInfoPar(int cmd, int par) async {
    final url = '$_baseUrl/api/setdeviceinfo/?custom=1&cmd=0x${cmd.toRadixString(16)}&par=$par';
    final response = await _client.post(Uri.parse(url))
        .timeout(const Duration(seconds: 5));
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    }
    throw QZProtocolException('POST cmd=0x${cmd.toRadixString(16)} failed');
  }
  
  /// 写状态命令（带 str 参数）
  Future<Map<String, dynamic>> setDeviceInfoStr(int cmd, String str) async {
    final url = '$_baseUrl/api/setdeviceinfo/?custom=1&cmd=0x${cmd.toRadixString(16)}&str=${Uri.encodeComponent(str)}';
    final response = await _client.post(Uri.parse(url))
        .timeout(const Duration(seconds: 5));
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    }
    throw QZProtocolException('POST cmd=0x${cmd.toRadixString(16)} failed');
  }
  
  /// 删除文件（同时带 par 和 str）
  Future<Map<String, dynamic>> deleteFile(String filePath) async {
    final url = '$_baseUrl/api/setdeviceinfo/?custom=1&cmd=0xfa3&par=0&str=${Uri.encodeComponent(filePath)}';
    final response = await _client.post(Uri.parse(url))
        .timeout(const Duration(seconds: 5));
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    }
    throw QZProtocolException('DELETE file failed');
  }
  
  /// 文件列表（返回 XML）
  Future<String> getFileList(String property, int from, int count) async {
    final url = '$_baseUrl/api/?action=dir&property=$property'
        '&format=all&from=$from&count=$count&backward=';
    final response = await _client.get(Uri.parse(url))
        .timeout(const Duration(seconds: 10));
    if (response.statusCode == 200) {
      return response.body;  // XML 字符串
    }
    throw QZProtocolException('dir failed');
  }
  
  /// 获取 setting_keys.xml
  Future<String> getSettingKeys() async {
    final url = '$_baseUrl/usr/share/minigui/res/lang/setting_keys.xml';
    final response = await _client.get(Uri.parse(url))
        .timeout(const Duration(seconds: 5));
    return response.body;
  }
  
  /// 获取翻译资源
  Future<String> getTranslation(String langCode) async {
    final url = '$_baseUrl/usr/share/minigui/res/lang/zh-$langCode.xml';
    final response = await _client.get(Uri.parse(url))
        .timeout(const Duration(seconds: 5));
    return response.body;
  }
}
```

### 5.2 命令码常量

```dart
class QZCommands {
  // 读取命令
  static const int deviceInfo = 0x7d1;       // 2001
  static const int menuValuesNormal = 0x7d2; // 2002
  static const int sdCardInfo = 0x7d4;       // 2004
  static const int recordStatus = 0x7d5;     // 2005
  static const int menuValuesSystem = 0x7d6; // 2006
  static const int workMode = 0x7d8;         // 2008
  static const int batteryLevel = 0x7d9;     // 2009
  static const int deviceCheck = 0xbcd;      // 3021
  
  // 写入命令
  static const int toggleRecord = 0x44c;     // 1100
  static const int capture = 0x44d;          // 1101
  static const int formatCard = 0x406;       // 1030
  static const int factoryReset = 0x407;     // 1031
  static const int setWifiName = 0xbbb;      // 3003
  static const int setWifiPassword = 0xbbc;  // 3004
  static const int setDate = 0xbbd;          // 3005
  static const int setTime = 0xbbe;          // 3006
  static const int enterPlayback = 0xbd9;    // 3033
  static const int switchMode = 0xbda;       // 3034
  static const int zoom = 0xbcc;             // 3020
  static const int deleteFile = 0xfa3;       // 4003
  static const int zoomDv = 0xfa4;           // 4004
}
```

---

## 6. XML 解析

### 6.1 文件列表解析

依赖：`xml: ^6.x`

```dart
import 'package:xml/xml.dart';

List<MediaFile> parseMediaList(String xmlString) {
  final document = XmlDocument.parse(xmlString);
  final files = <MediaFile>[];
  
  for (final fileElement in document.findAllElements('file')) {
    final name = fileElement.findElements('name').first.innerText;
    final size = int.tryParse(
      fileElement.findElements('size').first.innerText) ?? 0;
    final timeStr = fileElement.findElements('time').first.innerText;
    final time = DateTime.tryParse(timeStr) ?? DateTime.now();
    
    int? duration;
    final formatElements = fileElement.findElements('format');
    if (formatElements.isNotEmpty) {
      duration = int.tryParse(
        formatElements.first.getAttribute('time') ?? '') ?? 0;
    }
    
    files.add(MediaFile(
      name: name,
      size: size,
      time: time,
      duration: duration,
    ));
  }
  return files;
}
```

### 6.2 setting_keys.xml 解析

```dart
Map<String, List<String>> parseSettingKeys(String xmlString) {
  final document = XmlDocument.parse(xmlString);
  final result = <String, List<String>>{};
  
  for (final array in document.findAllElements('string-array')) {
    final name = array.getAttribute('name') ?? '';
    final items = array.findElements('item')
        .map((e) => e.innerText)
        .toList();
    result[name] = items;
  }
  return result;
}
```

### 6.3 翻译资源解析

翻译资源解析后需构造为 `QZTranslation` 列表。每个菜单 ID 对应两条记录：
- `str` = 菜单 ID → 标题翻译
- `str` = 菜单 ID + `_array` → 枚举值列表

```dart
class QZTranslation {
  final String str;
  final String value;
  final List<QZTranslationItem> items;
}

class QZTranslationItem {
  final String id;
  final String title;
}
```

---

## 7. RTSP 实时预览

### 7.1 推荐方案

| 方案 | 包名 | 优势 | 劣势 |
|---|---|---|---|
| `flutter_vlc_player` | `flutter_vlc_player` | 稳定，支持 RTSP | 包体大 |
| `fijkplayer` | `fijkplayer` | 基于 IJK（与原 App 同源） | 维护不活跃 |
| `media_kit` | `media_kit` | 基于 MPV，跨平台 | 相对较新 |
| `video_player` + 自定义 | — | 官方包 | 不原生支持 RTSP |

**推荐 `fijkplayer` 或 `media_kit`**，与原 App 的 IJKPlayer 技术栈最接近。

### 7.2 接入示例（fijkplayer）

```dart
import 'package:fijkplayer/fijkplayer.dart';

class PreviewController {
  final FijkPlayer _player = FijkPlayer();
  
  Future<void> startPreview() async {
    await _player.setOption(FijkOption.playerCategory, "mediacodec", 1);
    await _player.setOption(FijkOption.formatCategory, "rtsp_transport", "tcp");
    await _player.setOption(FijkOption.playerCategory, "packet-buffering", 0);
    await _player.setOption(FijkOption.playerCategory, "framedrop", 1);
    
    await _player.setDataSource(
      'rtsp://192.168.10.1:8554/ch00',
      autoPlay: true,
    );
  }
  
  Future<void> stopPreview() async {
    await _player.stop();
    await _player.reset();
  }
  
  void dispose() {
    _player.release();
  }
}
```

### 7.3 接入示例（media_kit）

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
  
  Future<void> startPreview() async {
    await _player.open(
      Media('rtsp://192.168.10.1:8554/ch00'),
    );
  }
  
  Future<void> stopPreview() async {
    await _player.stop();
  }
  
  void dispose() {
    _player.dispose();
  }
}
```

---

## 8. TCP 事件通道

### 8.1 实现

```dart
import 'dart:async';
import 'dart:io';
import 'dart:convert';

class QZEventSocket {
  Socket? _socket;
  Timer? _keepAliveTimer;
  final _eventController = StreamController<DeviceEvent>.broadcast();
  
  Stream<DeviceEvent> get eventStream => _eventController.stream;
  
  Future<bool> connect() async {
    try {
      _socket = await Socket.connect(
        '192.168.10.1',
        9999,
        timeout: const Duration(milliseconds: 5000),
      );
      
      _socket!.listen(
        (data) => _onData(data),
        onError: (e) => _onError(e),
        onDone: () => _onDisconnect(),
      );
      
      _startKeepAlive();
      _eventController.add(DeviceEvent(type: DeviceEventType.connected));
      return true;
    } catch (e) {
      return false;
    }
  }
  
  void _startKeepAlive() {
    _keepAliveTimer?.cancel();
    _keepAliveTimer = Timer.periodic(
      const Duration(milliseconds: 500),
      (_) => _sendKeepAlive(),
    );
  }
  
  void _sendKeepAlive() {
    try {
      _socket?.write('S:100.0');
    } catch (e) {
      _onError(e);
    }
  }
  
  void _onData(List<int> data) {
    final message = utf8.decode(data);
    // 解析设备推送的事件
    final event = _parseEvent(message);
    if (event != null) {
      _eventController.add(event);
    }
  }
  
  DeviceEvent? _parseEvent(String raw) {
    // 根据实际抓包结果解析
    // 目前已知的事件前缀模式待联调确认
    return DeviceEvent(type: DeviceEventType.connected, rawData: raw);
  }
  
  void _onError(dynamic error) {
    _eventController.add(DeviceEvent(type: DeviceEventType.error));
  }
  
  void _onDisconnect() {
    _eventController.add(DeviceEvent(type: DeviceEventType.disconnected));
  }
  
  Future<void> disconnect() async {
    _keepAliveTimer?.cancel();
    await _socket?.close();
    _socket = null;
  }
  
  void dispose() {
    disconnect();
    _eventController.close();
  }
}
```

---

## 9. 设备识别与连接

### 9.1 Wi-Fi 网关检测

```dart
import 'package:network_info_plus/network_info_plus.dart';

class DeviceDetector {
  static const String _qzGateway = '192.168.10.1';
  
  final _networkInfo = NetworkInfo();
  
  Future<bool> isQZDeviceConnected() async {
    final gateway = await _networkInfo.getWifiGatewayIP();
    return gateway == _qzGateway;
  }
}
```

### 9.2 连接流程

```dart
class QZCameraClient implements CameraClient {
  final QZHttpClient _http;
  final QZEventSocket _eventSocket;
  final PreviewController _preview;
  
  Future<bool> connect() async {
    // 1. 建立 TCP 事件通道
    final socketOk = await _eventSocket.connect();
    if (!socketOk) return false;
    
    // 2. 读取基础状态（并行请求）
    final results = await Future.wait([
      _http.getDeviceInfo(QZCommands.deviceInfo),    // 0x7d1
      _http.getDeviceInfo(QZCommands.recordStatus),  // 0x7d5
      _http.getDeviceInfo(QZCommands.sdCardInfo),    // 0x7d4
      _http.getDeviceInfo(QZCommands.workMode),      // 0x7d8
    ]);
    
    // 3. 解析设备信息
    _deviceInfo = DeviceInfo.fromJson(results[0]);
    _recordStatus = RecordStatus.fromJson(results[1]);
    _sdCardInfo = SdCardInfo.fromJson(results[2]);
    _workMode = WorkMode.fromString(results[3]['curworkmodename'] ?? '');
    
    // 4. 加载菜单（可异步）
    _loadMenus();
    
    return true;
  }
  
  @override
  String getLiveStreamUrl() => 'rtsp://192.168.10.1:8554/ch00';
}
```

---

## 10. 菜单系统组装

菜单的完整组装流程需要三步请求：

```dart
Future<List<QZMenu>> loadMenus() async {
  // Step 1: 获取 setting_keys.xml
  final keysXml = await _http.getSettingKeys();
  final keysMap = parseSettingKeys(keysXml);
  
  // Step 2: 获取翻译资源
  final langCode = _getLangCode();
  final transXml = await _http.getTranslation(langCode);
  final translations = parseTranslations(transXml);
  
  // Step 3: 获取当前值
  final normalValues = await _http.getDeviceInfo(QZCommands.menuValuesNormal);
  final systemValues = await _http.getDeviceInfo(QZCommands.menuValuesSystem);
  
  // 组装
  final menus = <QZMenu>[];
  
  // 当前模式对应的设置项
  final currentKeys = keysMap[_workMode.settingKeysName] ?? [];
  for (int i = 0; i < currentKeys.length; i++) {
    final menuId = currentKeys[i];
    final title = translations.findTitle(menuId);
    final items = translations.findItems('${menuId}_array');
    
    final currentValue = MenuCurrentValues.fromJson(normalValues)
        .info.firstWhere((v) => v.index == i, orElse: () => MenuValueItem(index: i, value: 0));
    
    menus.add(QZMenu(
      id: menuId,
      index: i,
      title: title,
      sysKeys: false,
      items: items,
      currentValueIndex: currentValue.value,
    ));
  }
  
  // 系统设置项
  final systemKeys = keysMap['system_setting_keys'] ?? [];
  for (int i = 0; i < systemKeys.length; i++) {
    final menuId = systemKeys[i];
    final title = translations.findTitle(menuId);
    final items = translations.findItems('${menuId}_array');
    
    final currentValue = MenuCurrentValues.fromJson(systemValues)
        .info.firstWhere((v) => v.index == i, orElse: () => MenuValueItem(index: i, value: 0));
    
    menus.add(QZMenu(
      id: menuId,
      index: i,
      title: title,
      sysKeys: true,
      items: items,
      currentValueIndex: currentValue.value,
    ));
  }
  
  return menus;
}

String _getLangCode() {
  final locale = Platform.localeName;
  if (locale.startsWith('zh_TW') || locale.startsWith('zh_Hant')) return 'TW';
  if (locale.startsWith('zh')) return 'CN';
  if (locale.startsWith('ko')) return 'KO';
  if (locale.startsWith('ru')) return 'RU';
  if (locale.startsWith('th')) return 'TI';
  if (locale.startsWith('sv')) return 'SV';
  if (locale.startsWith('ro')) return 'RO';
  if (locale.startsWith('pt')) return 'PT';
  if (locale.startsWith('pl')) return 'PL';
  if (locale.startsWith('nl')) return 'NL';
  if (locale.startsWith('ja')) return 'JP';
  return 'CN';  // 默认中文
}
```

---

## 11. 推荐 Flutter 依赖

```yaml
dependencies:
  # 网络
  http: ^1.2.0
  
  # XML 解析
  xml: ^6.5.0
  
  # Wi-Fi 检测
  network_info_plus: ^5.0.0
  
  # RTSP 播放（二选一）
  fijkplayer: ^0.11.0
  # media_kit: ^1.1.0
  # media_kit_video: ^1.1.0
  
  # 图片缓存（缩略图）
  cached_network_image: ^3.3.0
  
  # 状态管理
  provider: ^6.1.0  # 或 riverpod
  
  # 文件下载
  dio: ^5.4.0
  
  # 本地存储
  shared_preferences: ^2.2.0
  # sqflite: ^2.3.0  # 如需解析 sunxi.db
  
  # 权限
  permission_handler: ^11.3.0
```

---

## 12. 页面结构建议

| 页面 | 功能 | 对应原 App |
|---|---|---|
| `ConnectPage` | Wi-Fi 检测、设备连接 | `ConFrgm` |
| `PreviewPage` | RTSP 实时预览 + 拍照/录像控制 | `CamConQZAtv` |
| `AlbumPage` | 设备文件浏览（四个 Tab） | `AlbCamFrgm` |
| `FileViewPage` | 图片预览 / 视频播放 | — |
| `SettingPage` | 设备设置列表 | `CamSetFrgm` |
| `LocalAlbumPage` | 已下载到本地的文件 | `AlbLocalFrgm` |

---

## 13. 关键注意事项

1. **字段名大小写敏感**：`RecodStatus`（非 RecordStatus）、`curworkmodename`（全小写）
2. **数值都是字符串**：`disk_status`、`capacity`、`free_space`、`level` 等返回的都是字符串类型
3. **菜单三者必须对齐**：`setting_keys.xml` 的 index、翻译资源的 id、`MenuCurrValue` 的 index 三者一一对应
4. **HTTP 超时**：建议 5 秒，设备为嵌入式系统，响应可能较慢
5. **TCP 保活频率**：500ms 一次 `S:100.0`，丢失会导致设备判断断开
6. **RTSP 优先 TCP 传输**：设置 `rtsp_transport=tcp`，Wi-Fi 直连场景下 UDP 可能丢包
7. **文件路径**：所有路径以 `/mnt/DCIM/` 开头，不要截断
8. **并发请求**：初始化阶段可并行请求多个命令码，加快连接速度
