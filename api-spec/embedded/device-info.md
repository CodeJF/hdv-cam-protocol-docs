# REST API — 设备信息

> [← 返回目录](./README.md)

Base URL：`http://192.168.10.1:8080`

---

### 6.1 获取设备信息

**优先级：P0**

```
GET /api/v1/device/info
```

**你要返回：**

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

| 字段 | 类型 | 说明 |
|---|---|---|
| `deviceId` | number | 设备 ID |
| `deviceName` | string | 设备名称（用户可改） |
| `model` | string | 设备型号（出厂固定） |
| `firmware` | string | 固件版本号 |
| `serialNumber` | string | 设备序列号 |

**curl 自测：**

```bash
curl -s http://192.168.10.1:8080/api/v1/device/info | python3 -m json.tool
```

---

### 6.2 获取存储卡信息

**优先级：P0**

```
GET /api/v1/device/storage
```

**你要返回：**

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

| 字段 | 类型 | 说明 |
|---|---|---|
| `inserted` | boolean | SD 卡是否插入 |
| `totalMB` | number | 总容量（MB） |
| `freeMB` | number | 剩余空间（MB） |
| `usedMB` | number | 已用空间（MB） |

**无卡时返回：**

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "inserted": false,
    "totalMB": 0,
    "freeMB": 0,
    "usedMB": 0
  }
}
```

**curl 自测：**

```bash
curl -s http://192.168.10.1:8080/api/v1/device/storage | python3 -m json.tool
```

---

### 6.3 获取电池信息

**优先级：P1**

```
GET /api/v1/device/battery
```

**你要返回：**

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

| 字段 | 类型 | 说明 |
|---|---|---|
| `level` | number | 电量百分比 0-100 |
| `charging` | boolean | 是否正在充电 |
| `full` | boolean | 是否已充满 |

**curl 自测：**

```bash
curl -s http://192.168.10.1:8080/api/v1/device/battery | python3 -m json.tool
```

---

### 6.4 设备健康检查

**优先级：P1**

```
GET /api/v1/device/check
```

**你要返回：**

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

| 字段 | 类型 | 说明 |
|---|---|---|
| `online` | boolean | 设备是否正常工作 |
| `uptime` | number | 运行时长（秒） |

**curl 自测：**

```bash
curl -s http://192.168.10.1:8080/api/v1/device/check | python3 -m json.tool
```

---

