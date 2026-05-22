# 完整 Mock 数据集

> [← 返回目录](./README.md)

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
