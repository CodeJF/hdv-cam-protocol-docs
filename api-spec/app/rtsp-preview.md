# RTSP 预览接入（端口 8554）

> [← 返回目录](./README.md)

**优先级：P0**

### 5.1 流地址

```
rtsp://192.168.10.1:8554/ch00
```

### 5.2 推荐方案

| 方案 | 包名 | 推荐度 |
|---|---|---|
| fijkplayer | `fijkplayer` | ★★★ 基于 IJK，兼容性好 |
| media_kit | `media_kit` + `media_kit_video` | ★★★ 基于 MPV，跨平台 |
| flutter_vlc_player | `flutter_vlc_player` | ★★ 稳定但包体大 |

### 5.3 接入示例（fijkplayer）

```dart
import 'package:fijkplayer/fijkplayer.dart';

class PreviewController {
  final FijkPlayer _player = FijkPlayer();

  FijkPlayer get player => _player;

  Future<void> start() async {
    // 关键参数
    await _player.setOption(FijkOption.formatCategory, "rtsp_transport", "tcp");
    await _player.setOption(FijkOption.playerCategory, "packet-buffering", 0);
    await _player.setOption(FijkOption.playerCategory, "framedrop", 1);
    await _player.setOption(FijkOption.playerCategory, "mediacodec", 1);

    await _player.setDataSource(
      'rtsp://192.168.10.1:8554/ch00',
      autoPlay: true,
    );
  }

  Future<void> stop() async {
    await _player.stop();
    await _player.reset();
  }

  void dispose() {
    _player.release();
  }
}
```

### 5.4 接入示例（media_kit）

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

  Future<void> start() async {
    await _player.open(Media('rtsp://192.168.10.1:8554/ch00'));
  }

  Future<void> stop() async {
    await _player.stop();
  }

  void dispose() {
    _player.dispose();
  }
}
```

### 5.5 在页面中使用

```dart
// fijkplayer
FijkView(player: _previewController.player)

// media_kit
Video(controller: _previewController.videoController)
```

### 5.6 RTSP 预览 vs 录像回放：两个不同的播放场景

| | 实时预览 | 录像回放 |
|---|---|---|
| **播放什么** | 摄像头当前画面（直播流） | 已录好的视频文件 |
| **数据源** | RTSP `rtsp://192.168.10.1:8554/ch00` | HTTP `http://192.168.10.1:8080/api/v1/media/file?path=...` |
| **协议** | RTSP over TCP | HTTP Range 请求 |
| **能拖进度条吗** | 不能，是直播 | 能，支持 seek |
| **有缓冲条吗** | 无（实时流） | 有（边下边播） |
| **用哪个播放器** | fijkplayer / media_kit（配 RTSP 参数） | media_kit / fijkplayer（配 HTTP URL） |
| **对应页面** | PreviewPage（主界面） | VideoPlayPage（相册→点击视频） |

> HDV CAM 原版 App 也是这么区分的：IJKPlayer 负责 RTSP 实时预览，ExoPlayer 负责 HTTP 录像回放。
