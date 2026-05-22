# REST API — 设置

> [← 返回目录](./README.md)

---

### 9.1 获取菜单（一次性返回全部）

**优先级：P1**

```
GET /api/v1/settings/menus?lang=zh-CN
```

#### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `lang` | string | 否 | 语言码，默认 `zh-CN` |

#### 语言码

| 语言 | 值 |
|---|---|
| 简体中文 | `zh-CN` |
| 繁体中文 | `zh-TW` |
| English | `en` |
| 韩语 | `ko` |
| 日语 | `ja` |

#### 你要返回

**重要：把菜单定义、翻译、当前值一次性返回，App 不需要多次请求。**

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
          { "index": 1, "id": "en", "title": "English" },
          { "index": 2, "id": "zh-TW", "title": "繁體中文" }
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

#### 菜单项字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | 菜单唯一标识 |
| `title` | string | 菜单显示名称（已翻译） |
| `index` | number | 菜单序号 |
| `currentValue` | number | 当前选中的选项 index，`-1` 表示非选项类型 |
| `options` | array | 可选项列表（选项类型菜单才有） |
| `options[].index` | number | 选项序号 |
| `options[].id` | string | 选项标识 |
| `options[].title` | string | 选项显示名称（已翻译） |
| `type` | string | 可选，特殊菜单类型：`input`（输入框）、`datetime`（时间选择）、`action`（动作按钮） |
| `value` | string | 可选，`input`/`datetime` 类型的当前值 |

#### 关键设计说明

**为什么一次性返回？** 原协议需要 3 次请求（setting_keys.xml + 翻译 + 当前值）再在 App 端拼装。新设计由设备端直接组装好，App 端拿到就能直接渲染，减少请求次数和 App 端复杂度。

**curl 自测：**

```bash
curl -s "http://192.168.10.1:8080/api/v1/settings/menus?lang=zh-CN" | python3 -m json.tool
```

---

### 9.2 修改菜单选项值

**优先级：P1**

```
POST /api/v1/settings/menu/value
Content-Type: application/json

{
  "id": "rec_resolution",
  "value": 1
}
```

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | string | 菜单 ID（从菜单列表获取） |
| `value` | number | 新的选项 index |

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**你要做的事：** 修改对应设置项的当前值，使其立即生效。

**curl 自测：**

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"id": "rec_resolution", "value": 1}' \
  http://192.168.10.1:8080/api/v1/settings/menu/value | python3 -m json.tool
```

---

### 9.3 设置 Wi-Fi

**优先级：P1**

```
POST /api/v1/settings/wifi
Content-Type: application/json

{
  "ssid": "MyCamera",
  "password": "12345678"
}
```

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `ssid` | string | 否 | 新的 Wi-Fi 名称，不传则不修改 |
| `password` | string | 否 | 新的 Wi-Fi 密码，不传则不修改 |

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "ssid": "MyCamera",
    "password": "12345678",
    "reconnectRequired": true
  }
}
```

#### 返回字段（data）

| 字段 | 类型 | 说明 |
|---|---|---|
| `ssid` | string | 修改后的 Wi-Fi 热点名称 |
| `password` | string | 修改后的 Wi-Fi 密码 |
| `reconnectRequired` | boolean | 是否需要 App 重新连接 Wi-Fi（修改了 ssid 或 password 时为 `true`） |

**你要做的事：** 修改 AP 热点名称/密码。修改后客户端需要断开并重新连接新热点。

**curl 自测：**

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"ssid": "MyCamera", "password": "88888888"}' \
  http://192.168.10.1:8080/api/v1/settings/wifi | python3 -m json.tool
```

---

### 9.4 同步日期时间

**优先级：P1**

```
POST /api/v1/settings/datetime
Content-Type: application/json

{
  "datetime": "2026-05-22 14:30:00"
}
```

| 参数 | 类型 | 格式 | 说明 |
|---|---|---|---|
| `datetime` | string | `yyyy-MM-dd HH:mm:ss` | 手机当前时间 |

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**你要做的事：** 将设备系统时间设置为传入的时间值。

**curl 自测：**

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"datetime": "2026-05-22 14:30:00"}' \
  http://192.168.10.1:8080/api/v1/settings/datetime | python3 -m json.tool
```

---

### 9.5 格式化 SD 卡

**优先级：P2**

```
POST /api/v1/settings/format
```

请求体：无

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**你要做的事：** 格式化 SD 卡，删除所有媒体文件。

**curl 自测：**

```bash
curl -s -X POST http://192.168.10.1:8080/api/v1/settings/format | python3 -m json.tool
```

---

### 9.6 恢复出厂设置

**优先级：P2**

```
POST /api/v1/settings/reset
```

请求体：无

**你要返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**curl 自测：**

```bash
curl -s -X POST http://192.168.10.1:8080/api/v1/settings/reset | python3 -m json.tool
```

---
