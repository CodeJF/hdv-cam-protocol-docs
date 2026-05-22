# REST API — 设备信息

> [← 返回目录](./README.md)

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

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `deviceId` | number | 设备 ID，每台设备唯一 |
| `deviceName` | string | 设备名称，用户可通过设置修改，也是 Wi-Fi 热点名 |
| `model` | string | 设备型号，出厂固定（如 `QZ-4K`） |
| `firmware` | string | 固件版本号（如 `V1.0.0`） |
| `serialNumber` | string | 设备序列号，出厂固定 |

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

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `inserted` | boolean | SD 卡是否已插入，`false` 时其他字段都为 0 |
| `totalMB` | number | SD 卡总容量（MB），如 127512 表示约 128 GB |
| `freeMB` | number | SD 卡剩余可用空间（MB） |
| `usedMB` | number | SD 卡已用空间（MB） |

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

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `level` | number | 电量百分比，0-100，如 85 表示 85% |
| `charging` | boolean | 是否正在充电（USB 供电中） |
| `full` | boolean | 电池是否已充满（charging=true 且 level=100 时为 true） |

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

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `online` | boolean | 设备是否正常工作，用于判断设备是否死机或异常 |
| `uptime` | number | 设备已运行时长（秒），如 3600 表示开机 1 小时 |

#### 调用示例

```dart
final resp = await http.get('/api/v1/device/check');
if (!resp.isSuccess) {
  showError('设备离线');
}
```

---
