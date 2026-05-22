# REST API — 设置

> [← 返回目录](./README.md)

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

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `currentMode` | string | 当前工作模式名称，如 `NormalRecordeMode` |
| `currentModeIndex` | number | 当前工作模式编号 0-7 |
| `modeMenus` | array | 当前模式下的设置菜单（如分辨率、循环录像），切换模式后菜单项会变 |
| `systemMenus` | array | 系统菜单（Wi-Fi、时间、语言、格式化等），所有模式通用 |

**菜单项字段：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | 菜单唯一标识，如 `rec_resolution`、`wifi_ssid` |
| `title` | string | 菜单显示名称，已按 `lang` 参数翻译 |
| `index` | number | 菜单排序序号 |
| `currentValue` | number | 当前选中的选项 index，`-1` 表示非选项类型（如输入框、按钮） |
| `options` | array | 可选项列表，每项有 `index`、`id`、`title`。选项类型菜单才有值，其他为空数组 |
| `type` | string? | 可选，特殊类型：`input`=输入框（Wi-Fi名/密码），`datetime`=时间选择，`action`=按钮（格式化/重置）。省略表示普通选项类型 |
| `value` | string? | 可选，`input`/`datetime` 类型的当前文本值 |

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

#### 请求体字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 是 | 菜单项唯一标识，从菜单列表的 `id` 字段获取 |
| `value` | number | 是 | 新的选项 index，对应菜单项 `options` 数组中的 `index` 值 |

返回 `data: null`，成功与否看 `code`。

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

**什么时候调用：** 用户在设置页修改相机 Wi-Fi 热点名称或密码。修改后手机需要重新连接新热点。

```
POST /api/v1/settings/wifi
Content-Type: application/json

{"ssid": "MyCamera", "password": "88888888"}
```

#### 请求体字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `ssid` | string | 否 | 新的 Wi-Fi 热点名称，不传则不修改 |
| `password` | string | 否 | 新的 Wi-Fi 密码，不传则不修改 |

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

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `ssid` | string | 修改后的 Wi-Fi 热点名称 |
| `password` | string | 修改后的 Wi-Fi 密码 |
| `reconnectRequired` | boolean | 是否需要手机重新连接 Wi-Fi，修改了名称或密码时为 `true` |

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

**什么时候调用：** App 连接设备后自动调用一次，把手机当前时间同步给相机，确保录像/拍照的时间戳准确。

```
POST /api/v1/settings/datetime
Content-Type: application/json

{"datetime": "2026-05-22 14:30:00"}
```

#### 请求体字段

| 字段 | 类型 | 必填 | 格式 | 说明 |
|---|---|---|---|---|
| `datetime` | string | 是 | `yyyy-MM-dd HH:mm:ss` | 手机当前时间，如 `2026-05-22 14:30:00` |

返回 `data: null`，成功与否看 `code`。

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

