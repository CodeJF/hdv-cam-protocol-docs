# 工作模式定义

> [← 返回目录](./README.md)

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
