# 错误码与处理

> [← 返回目录](./README.md)

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
