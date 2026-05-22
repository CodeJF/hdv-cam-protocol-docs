# QZ API 合同草案

## 导航

### 主入口

- [主入口：app-newCam-release-technical-analysis.md](./app-newCam-release-technical-analysis.md)

### 兄弟文档

- [QZ 协议总览](./qz-protocol-overview.md)
- [QZ 媒体与菜单模型](./qz-media-model.md)
- [QZ 复刻实施计划](./qz-replica-plan.md)

### 本页定位

- 适合设备端和协议层开发直接对接口
- 重点是命令码、请求模板、最小返回结构
- 不展开媒体解析和项目排期

## 1. 固定前提

| 项目 | 值 |
|---|---|
| 设备地址 | `192.168.10.1` |
| HTTP Host | `http://192.168.10.1:8082` |
| 控制根路径 | `/api/` |
| TCP 事件端口 | `9999` |
| RTSP 端口 | `8554` |

## 2. 请求模板

### 2.1 读状态

```http
GET /api/getdeviceinfo/?custom=1&cmd=<cmd><suffix>
```

### 2.2 写状态

```http
POST /api/setdeviceinfo/?custom=1&cmd=<cmd><suffix>
```

### 2.3 动作/目录

```http
GET /api/?action=<action>&k1=v1&k2=v2
```

### 2.4 资源文件

```http
GET /usr/share/minigui/res/lang/setting_keys.xml
GET /usr/share/minigui/res/lang/zh-<LANG>.xml
GET /tmp/<name>
GET /tmp/data/.data/sqlite/sunxi.db
```

## 3. 已确认命令码

### 3.1 读状态命令

| 功能 | 请求 | 备注 |
|---|---|---|
| 设备基础信息 | `cmd=0x7d1` | 连接初始读取 |
| 菜单当前值（非系统） | `cmd=0x7d2` | 配合 setting_keys.xml |
| SD 卡信息 | `cmd=0x7d4` | 容量/剩余/状态 |
| 录像状态 | `cmd=0x7d5` | `RecodStatus` |
| 菜单当前值（系统） | `cmd=0x7d6` | 配合 system_setting_keys |
| 当前工作模式 | `cmd=0x7d8` | `curworkmodename` |
| 电池信息 | `cmd=0x7d9` | 电量状态 |
| 设备检查 | `cmd=0xbcd` | 设备在线/能力确认 |

### 3.2 写/控制命令

| 功能 | 请求 | 备注 |
|---|---|---|
| 开始/停止录像 | `cmd=0x44c&par=<0/1>` | `1`=开始 `0`=停止 |
| 拍照 | `cmd=0x44d&par=<0/1>` | |
| 格式化卡 | `cmd=0x406&par=0` | |
| 恢复默认 | `cmd=0x407&par=0` | |
| 进入/退出回放 | `cmd=0xbd9&par=<0/1>` | |
| 设置 Wi-Fi 名称 | `cmd=0xbbb&str=<ssid>` | |
| 设置 Wi-Fi 密码 | `cmd=0xbbc&str=<pwd>` | |
| 设置日期 | `cmd=0xbbd&str=<yyyy-MM-dd>` | |
| 设置时间 | `cmd=0xbbe&str=<HH:mm:ss>` | |
| 变焦 | `cmd=0xbcc&par=<level>` | |
| 设置工作模式 | `cmd=0xbda&par=<mode>` | 模式编号 0-7 |
| 删除文件 | `cmd=0xfa3&par=0&str=<path>` | |
| 变焦 DV | `cmd=0xfa4&par=<level>` | |

## 4. 最小返回约束

### 4.1 通用 JSON 响应

QZ 基础响应不强依赖统一外壳。最小要求：

- 返回合法 JSON object
- 顶层字段可被直接拍平为 `map`

最小成功示例：

```json
{
  "status": "0"
}
```

### 4.2 SD 卡信息（0x7d4）

```json
{
  "disk_status": "1",
  "capacity": "127512.0",
  "free_space": "92341.0"
}
```

### 4.3 录像状态（0x7d5）

```json
{
  "RecodStatus": "1"
}
```

### 4.4 当前工作模式（0x7d8）

```json
{
  "curworkmodename": "NormalVideo"
}
```

## 5. 文件列表接口

目录接口模板：

```http
GET /api/?action=dir&property=<Type>&format=all&from=<from>&count=<count>&backward=
```

`property` 取值：

| 类型 | property |
|---|---|
| 图片 | `Photo` |
| 普通视频 | `Normal` |
| 事件视频 | `Event` |
| 停车视频 | `Parking` |

分页规则：每页 `20`，`from = page * 20`。

## 6. RTSP 合同

固定入口：

```text
rtsp://192.168.10.1:8554/ch00
```

此链路必须在 App 进入预览页时可立即播放。

## 7. TCP 事件通道

| 项目 | 值 |
|---|---|
| 端口 | `9999`（0x270F） |
| 连接超时 | `5000ms`（0x1388） |
| 心跳间隔 | `500ms`（0x1F4） |
| 心跳内容 | `S:100.0` |
| 接收缓冲区 | `1024` 字节（0x400） |

App 连接设备后立即建立 TCP Socket，每 500ms 发送 `S:100.0`。

事件类型：

- `CameraEventMessage:connect` - 连接成功
- `CameraEventMessage:break` - 连接断开
- `CameraEventMessage:Exception` - 异常
- `CameraEventMessage:clean` - 清理

**设备端必须监听 9999 端口**，否则 App 可能判定连接失败。

## 8. sunxi.db 路径

完整下载 URL：

```text
http://192.168.10.1:8082/tmp/data/.data/sqlite/sunxi.db
```

App 下载后本地保存为 `DCF.db`，走数据库解析链。

## 9. 工作模式编号

| 模式 | 编号 | 说明 |
|---|---|---|
| NormalRecordeMode | 0 | 普通录像 |
| SlowRecordeMode | 1 | 慢动作 |
| LoopRecordeMode | 2 | 循环录像 |
| TimeLapseMode | 3 | 延时摄影 |
| NormalCaptureMode | 4 | 普通拍照 |
| AutoCaptureMode | 5 | 自动拍照 |
| ContinueCaptureMode | 6 | 连拍 |
| TimingCaptureMode | 7 | 定时拍照 |

通过 `cmd=0xbda&par=<编号>` 切换。

## 10. 风险点

- 文档中的请求模板已经足够启动开发，但不是最终抓包原文
- `0x7d1` 的完整字段仍需联调校准
- 如果只做 HTTP 而不做 TCP/RTSP，App 可能无法正常连接
- TCP 心跳通道是连接成功的前提条件
