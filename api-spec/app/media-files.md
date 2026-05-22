# REST API — 媒体文件（在线查看、下载、删除）

> [← 返回目录](./README.md)

**这一章解决"如何查看相机里的视频和照片、如何下载到手机"的问题。**

> **核心思路：先看后下。** 用户点开就能直接看（图片在线加载、视频在线播放），觉得满意再下载到手机。不需要先下载完才能看。
>
> **原理：** HDV CAM 原版 App（Android 端）也是这么做的——用 ExoPlayer 直接播放设备 HTTP URL（`http://192.168.10.1:8082/file/media/...`），ExoPlayer 通过 HTTP Range 请求边下边播，所以点开视频很快就能播、还能看到缓冲进度条。我们的做法一样，只是 URL 换成了我们的 REST API 路径。

### 完整流程

```
App 打开相册页
     │
     ▼
[1] 获取文件列表  GET /api/v1/media/files?type=photo
     │
     ▼
[2] 加载缩略图    GET /api/v1/media/thumbnail?path=xxx
     │  （网格展示，每个文件一张小图）
     ▼
用户点击某个文件
     │
     ├─ 图片 → [3a] 在线查看原图
     │         Image.network(fileUrl) 直接从 HTTP 加载
     │         不用先下载到本地，支持缩放手势
     │
     └─ 视频 → [3b] 在线播放视频
               播放器直接用 HTTP URL 播放，边下边播
               支持拖动进度条（设备端通过 Range 请求跳转）
               1-2 秒内开始播放，有缓冲进度条
     │
     ▼
用户觉得满意，点"保存到手机"
     │
     └─ [4] 下载到本地  GET /api/v1/media/file?path=xxx
              用 Dio 下载到手机相册，显示下载进度
     │
用户长按删除
     │
     └─ [5] 删除文件  DELETE /api/v1/media/file?path=xxx
```

---

### 8.1 获取文件列表

**优先级：P1**

```
GET /api/v1/media/files?type=video_normal&page=1&pageSize=20
```

#### 参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `type` | string | 是 | 见下表 |
| `page` | number | 否 | 页码（从 1 开始），默认 1 |
| `pageSize` | number | 否 | 每页数量，默认 20 |

#### type 取值

| 值 | 含义 |
|---|---|
| `video_normal` | 普通视频 |
| `video_event` | 事件视频（碰撞触发） |
| `video_parking` | 停车监控视频 |
| `photo` | 照片 |

#### 响应示例

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "total": 45,
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
      }
    ]
  }
}
```

#### 数据模型

```dart
class MediaFile {
  final String path;
  final String name;
  final int size;
  final String time;
  final int? duration;  // 视频时长（秒），照片为 null
  final int? width;
  final int? height;

  MediaFile.fromJson(Map<String, dynamic> json)
      : path = json['path'],
        name = json['name'],
        size = json['size'],
        time = json['time'],
        duration = json['duration'],
        width = json['width'],
        height = json['height'];

  bool get isVideo => name.toLowerCase().endsWith('.mp4') ||
                      name.toLowerCase().endsWith('.mov');
  bool get isPhoto => name.toLowerCase().endsWith('.jpg');

  /// 缩略图 URL（静态 HTTP URL，直接给 Image.network / CachedNetworkImage 使用）
  String get thumbnailUrl {
    // path 格式：/mnt/DCIM/Normal/VID_xxx.MP4
    // 缩略图 URL：http://192.168.10.1:8080/thumb/mnt/DCIM/Normal/VID_xxx.MP4.jpg
    final trimmed = path.startsWith('/') ? path.substring(1) : path;
    return 'http://192.168.10.1:8080/thumb/$trimmed.jpg';
  }

  /// 文件下载 URL
  String get fileUrl =>
    'http://192.168.10.1:8080/api/v1/media/file?path=${Uri.encodeComponent(path)}';

  /// 格式化文件大小
  String get sizeDisplay {
    if (size > 1024 * 1024 * 1024) return '${(size / 1024 / 1024 / 1024).toStringAsFixed(1)} GB';
    if (size > 1024 * 1024) return '${(size / 1024 / 1024).toStringAsFixed(1)} MB';
    return '${(size / 1024).toStringAsFixed(1)} KB';
  }

  /// 格式化时长
  String get durationDisplay {
    if (duration == null) return '';
    final m = duration! ~/ 60;
    final s = duration! % 60;
    return '${m.toString().padLeft(2, '0')}:${s.toString().padLeft(2, '0')}';
  }
}

class MediaFileList {
  final int total;
  final int page;
  final int pageSize;
  final List<MediaFile> files;

  MediaFileList.fromJson(Map<String, dynamic> json)
      : total = json['total'],
        page = json['page'],
        pageSize = json['pageSize'],
        files = (json['files'] as List).map((f) => MediaFile.fromJson(f)).toList();

  bool get hasMore => page * pageSize < total;
}
```

#### 调用示例

```dart
/// 获取普通视频列表
Future<MediaFileList> getVideoList({int page = 1}) async {
  final resp = await http.get<MediaFileList>(
    '/api/v1/media/files',
    params: {
      'type': 'video_normal',
      'page': '$page',
      'pageSize': '20',
    },
    fromData: (d) => MediaFileList.fromJson(d),
  );
  return resp.data!;
}

/// 获取照片列表
Future<MediaFileList> getPhotoList({int page = 1}) async {
  final resp = await http.get<MediaFileList>(
    '/api/v1/media/files',
    params: {
      'type': 'photo',
      'page': '$page',
      'pageSize': '20',
    },
    fromData: (d) => MediaFileList.fromJson(d),
  );
  return resp.data!;
}
```

---

### 8.2 缩略图

**优先级：P1**

> **缩略图是静态 HTTP URL，不走 REST API 接口。** 视频和照片都有缩略图。URL 是固定规则拼出来的，不需要单独请求。直接把 URL 传给 Flutter 图片组件加载即可。

#### URL 规则

```
http://192.168.10.1:8080/thumb/<文件路径去掉开头斜杠>.jpg
```

| 原文件路径 | 缩略图 URL |
|---|---|
| `/mnt/DCIM/Normal/VID_20260522_143000.MP4` | `http://192.168.10.1:8080/thumb/mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg` |
| `/mnt/DCIM/Photo/IMG_20260522_143500.jpg` | `http://192.168.10.1:8080/thumb/mnt/DCIM/Photo/IMG_20260522_143500.jpg.jpg` |

`MediaFile` 数据模型里已经封装了这个规则，直接用 `file.thumbnailUrl` 即可。

#### 在列表中显示缩略图

```dart
// 方式 1：直接用 Image.network（简单场景）
Image.network(
  file.thumbnailUrl,   // 静态 HTTP URL，Flutter 直接加载
  width: 120,
  height: 90,
  fit: BoxFit.cover,
  errorBuilder: (_, __, ___) => Icon(Icons.broken_image),
)

// 方式 2：用 cached_network_image（推荐，自带磁盘缓存 + 内存缓存）
CachedNetworkImage(
  imageUrl: file.thumbnailUrl,
  width: 120,
  height: 90,
  fit: BoxFit.cover,
  placeholder: (_, __) => Container(
    color: Colors.grey[300],
    child: Icon(Icons.image, color: Colors.grey),
  ),
  errorWidget: (_, __, ___) => Icon(Icons.broken_image),
)
```

> **为什么用静态 URL 而不是 REST API 接口？**
> - `Image.network` / `CachedNetworkImage` 直接传 URL 就能用，不需要额外封装 HTTP 请求
> - `CachedNetworkImage` 会自动做磁盘缓存 + 内存缓存，相册页来回切换不用重复下载
> - 相册页 GridView 同时显示 20+ 张缩略图，图片库能自动并发加载、取消离屏请求
> - HDV CAM 原版也是这么做的：`/thumb/mnt/DCIM/...`，Android 端用 Glide 加载，iOS 端用 SDWebImage 加载

---

### 8.3 在线查看图片

**优先级：P1**

```
GET /api/v1/media/file?path=/mnt/DCIM/Photo/IMG_20260522_143500.jpg
```

返回图片二进制数据（`image/jpeg`）。直接用 `Image.network` 加载，不需要先下载到本地。

#### 全屏查看图片

```dart
class PhotoViewPage extends StatelessWidget {
  final MediaFile file;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(file.name),
        actions: [
          // 保存到手机按钮
          IconButton(
            icon: Icon(Icons.download),
            onPressed: () => _saveToPhone(context, file),
          ),
        ],
      ),
      body: InteractiveViewer(
        child: Image.network(
          file.fileUrl,  // 直接用 HTTP URL，在线加载
          fit: BoxFit.contain,
          loadingBuilder: (_, child, progress) {
            if (progress == null) return child;
            return Center(child: CircularProgressIndicator(
              value: progress.expectedTotalBytes != null
                  ? progress.cumulativeBytesLoaded / progress.expectedTotalBytes!
                  : null,
            ));
          },
        ),
      ),
    );
  }
}
```

---

### 8.4 在线播放视频

**优先级：P0**

> **这是相册功能最重要的能力。** 用户点开视频，1-2 秒内开始播放，有缓冲进度条，可以拖动进度条跳转。不需要等整个视频下载完。
>
> **原理：** 播放器直接用设备的 HTTP URL 作为数据源，通过 Range 请求分段获取数据，边下边播。HDV CAM 原版 App 中 Android 端用的是 ExoPlayer，我们用 `media_kit`（基于 MPV）或 `fijkplayer`（基于 IJK），原理一样。

```
播放器的数据源 URL:
http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 方案对比

| 方案 | 包名 | 优势 | 劣势 |
|---|---|---|---|
| **media_kit（推荐）** | `media_kit` | 基于 MPV，跨平台，支持 HTTP/RTSP，维护活跃 | 相对较新 |
| fijkplayer | `fijkplayer` | 基于 IJK（与 HDV CAM 原版同源） | 维护不活跃 |
| flutter_vlc_player | `flutter_vlc_player` | 稳定，支持 RTSP | 包体大 |

#### 用 media_kit 在线播放（推荐）

```dart
import 'package:media_kit/media_kit.dart';
import 'package:media_kit_video/media_kit_video.dart';

class VideoPlayPage extends StatefulWidget {
  final MediaFile file;
  const VideoPlayPage({required this.file});

  @override
  State<VideoPlayPage> createState() => _VideoPlayPageState();
}

class _VideoPlayPageState extends State<VideoPlayPage> {
  late final Player _player;
  late final VideoController _controller;

  @override
  void initState() {
    super.initState();
    _player = Player();
    _controller = VideoController(_player);

    // 直接用 HTTP URL 播放，不需要先下载
    _player.open(Media(widget.file.fileUrl));
  }

  @override
  void dispose() {
    _player.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.file.name),
        actions: [
          // 下载到手机
          IconButton(
            icon: Icon(Icons.download),
            onPressed: () => _saveToPhone(context, widget.file),
          ),
        ],
      ),
      body: Video(
        controller: _controller,
        // media_kit 内置了播放/暂停、进度条、全屏等控件
      ),
    );
  }
}
```

#### 用 fijkplayer 在线播放

```dart
import 'package:fijkplayer/fijkplayer.dart';

class VideoPlayPage extends StatefulWidget {
  final MediaFile file;
  const VideoPlayPage({required this.file});

  @override
  State<VideoPlayPage> createState() => _VideoPlayPageState();
}

class _VideoPlayPageState extends State<VideoPlayPage> {
  final FijkPlayer _player = FijkPlayer();

  @override
  void initState() {
    super.initState();
    _initPlayer();
  }

  Future<void> _initPlayer() async {
    // 开启硬解码
    await _player.setOption(FijkOption.playerCategory, "mediacodec", 1);
    // 减少缓冲延迟
    await _player.setOption(FijkOption.playerCategory, "packet-buffering", 0);
    await _player.setOption(FijkOption.playerCategory, "framedrop", 1);

    // 直接用 HTTP URL 播放
    await _player.setDataSource(
      widget.file.fileUrl,
      autoPlay: true,
    );
  }

  @override
  void dispose() {
    _player.release();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.file.name),
        actions: [
          IconButton(
            icon: Icon(Icons.download),
            onPressed: () => _saveToPhone(context, widget.file),
          ),
        ],
      ),
      body: FijkView(
        player: _player,
        // 内置播放控件（进度条、缓冲条、播放/暂停）
      ),
    );
  }
}
```

---

### 8.5 下载文件到手机

**优先级：P1**

> 用户在线看完觉得满意，点"保存到手机"按钮时才下载。图片和视频都走这个流程。

```
GET /api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 用 Dio 下载（支持进度显示）

```dart
import 'package:dio/dio.dart';
import 'package:path_provider/path_provider.dart';

class FileDownloader {
  final Dio _dio = Dio();

  /// 下载文件到本地
  Future<String> download(
    MediaFile file, {
    void Function(int received, int total)? onProgress,
  }) async {
    final dir = await getApplicationDocumentsDirectory();
    final savePath = '${dir.path}/${file.name}';

    await _dio.download(
      file.fileUrl,
      savePath,
      onReceiveProgress: onProgress,
    );

    return savePath;
  }
}
```

#### 下载按钮示例

```dart
Future<void> _saveToPhone(BuildContext context, MediaFile file) async {
  final downloader = FileDownloader();

  // 显示下载进度对话框
  showDialog(
    context: context,
    barrierDismissible: false,
    builder: (_) => _DownloadProgressDialog(
      file: file,
      downloader: downloader,
    ),
  );

  final localPath = await downloader.download(
    file,
    onProgress: (received, total) {
      final percent = (received / total * 100).toStringAsFixed(0);
      debugPrint('下载进度: $percent%');
    },
  );

  Navigator.pop(context); // 关闭进度对话框

  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(content: Text('已保存到: ${file.name}')),
  );
}
```

---

### 8.6 删除文件

**优先级：P1**

```
DELETE /api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 调用示例

```dart
Future<bool> deleteFile(MediaFile file) async {
  final resp = await http.delete(
    '/api/v1/media/file',
    params: {'path': file.path},
  );
  if (resp.isSuccess) {
    setState(() => _files.remove(file));
    return true;
  }
  showError('删除失败: ${resp.msg}');
  return false;
}
```

---

### 8.7 相册页完整示例

```dart
class AlbumPage extends StatefulWidget {
  @override
  State<AlbumPage> createState() => _AlbumPageState();
}

class _AlbumPageState extends State<AlbumPage> with SingleTickerProviderStateMixin {
  late TabController _tabController;
  final _types = ['video_normal', 'video_event', 'video_parking', 'photo'];
  final _titles = ['普通视频', '事件视频', '停车视频', '照片'];
  Map<String, MediaFileList?> _fileLists = {};

  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: 4, vsync: this);
    _loadFiles('video_normal');
  }

  Future<void> _loadFiles(String type) async {
    final resp = await http.get<MediaFileList>(
      '/api/v1/media/files',
      params: {'type': type, 'page': '1', 'pageSize': '20'},
      fromData: (d) => MediaFileList.fromJson(d),
    );
    if (resp.isSuccess) {
      setState(() => _fileLists[type] = resp.data);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('相册'),
        bottom: TabBar(
          controller: _tabController,
          tabs: _titles.map((t) => Tab(text: t)).toList(),
          onTap: (i) => _loadFiles(_types[i]),
        ),
      ),
      body: TabBarView(
        controller: _tabController,
        children: _types.map((type) {
          final list = _fileLists[type];
          if (list == null) return Center(child: CircularProgressIndicator());
          if (list.files.isEmpty) return Center(child: Text('暂无文件'));

          return GridView.builder(
            gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
              crossAxisCount: 3,
              childAspectRatio: 4 / 3,
            ),
            itemCount: list.files.length,
            itemBuilder: (_, i) {
              final file = list.files[i];
              return GestureDetector(
                onTap: () => _openFile(file),
                child: Stack(
                  fit: StackFit.expand,
                  children: [
                    // 缩略图
                    Image.network(file.thumbnailUrl, fit: BoxFit.cover),
                    // 视频时长标签
                    if (file.isVideo)
                      Positioned(
                        bottom: 4, right: 4,
                        child: Container(
                          padding: EdgeInsets.symmetric(horizontal: 4, vertical: 2),
                          color: Colors.black54,
                          child: Text(file.durationDisplay,
                            style: TextStyle(color: Colors.white, fontSize: 12)),
                        ),
                      ),
                  ],
                ),
              );
            },
          );
        }).toList(),
      ),
    );
  }

  void _openFile(MediaFile file) {
    if (file.isPhoto) {
      // 在线查看图片
      Navigator.push(context,
        MaterialPageRoute(builder: (_) => PhotoViewPage(file: file)));
    } else {
      // 在线播放视频（直接用 HTTP URL，不用先下载）
      Navigator.push(context,
        MaterialPageRoute(builder: (_) => VideoPlayPage(file: file)));
    }
  }
}
```
