# REST API — 媒体文件

> [← 返回目录](./README.md)

这一章定义 App 如何浏览、**在线查看**、下载、删除相机上的照片和视频。

> **重要背景：** App 不是"先下载完再看"，而是**直接用你返回的 HTTP URL 在线播放视频**。播放器向你发 HTTP 请求，通过 Range 分段获取数据边下边播。所以你的 HTTP 文件服务必须正确支持 Range 请求和 `Accept-Ranges` 头，视频文件的 moov atom 必须在文件头部。详见下方 [在线查看/下载文件](#83-在线查看下载文件) 小节。

---

### 8.1 获取文件列表

**优先级：P1**

```
GET /api/v1/media/files?type=video_normal&page=1&pageSize=20
```

#### 请求参数（Query String）

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `type` | string | 是 | 媒体类型，见下表 |
| `page` | number | 否 | 页码，从 1 开始，默认 1 |
| `pageSize` | number | 否 | 每页数量，默认 20 |

#### type 取值

| type 值 | 含义 | 存储目录 |
|---|---|---|
| `video_normal` | 普通视频 | `/mnt/DCIM/Normal/` |
| `video_event` | 事件视频（碰撞触发） | `/mnt/DCIM/Event/` |
| `video_parking` | 停车监控视频 | `/mnt/DCIM/Parking/` |
| `photo` | 照片 | `/mnt/DCIM/Photo/` |

#### 你要返回

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

#### 文件对象字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `path` | string | 是 | 文件完整路径 |
| `name` | string | 是 | 文件名 |
| `size` | number | 是 | 文件大小（字节） |
| `time` | string | 是 | 拍摄时间 `yyyy-MM-dd HH:mm:ss` |
| `duration` | number | 视频必填 | 时长（秒），照片不返回或返回 0 |
| `width` | number | 否 | 分辨率宽 |
| `height` | number | 否 | 分辨率高 |

#### 空列表返回

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "total": 0,
    "page": 1,
    "pageSize": 20,
    "files": []
  }
}
```

**curl 自测：**

```bash
# 获取普通视频列表
curl -s "http://192.168.10.1:8080/api/v1/media/files?type=video_normal&page=1&pageSize=20" | python3 -m json.tool

# 获取照片列表
curl -s "http://192.168.10.1:8080/api/v1/media/files?type=photo&page=1&pageSize=20" | python3 -m json.tool

# 获取事件视频列表
curl -s "http://192.168.10.1:8080/api/v1/media/files?type=video_event&page=1&pageSize=20" | python3 -m json.tool
```

---

### 8.2 获取缩略图

**优先级：P1**

> **设计思路：** 缩略图用静态 HTTP URL 直接访问，不走 REST API 接口。这样做的好处是 App 端可以直接把 URL 传给 `Image.network()` / `CachedNetworkImage` 等图片库，库自带缓存、并发加载、占位图等能力，不需要额外封装。HDV CAM 原版也是这么做的（`/thumb/mnt/DCIM/...`）。

#### URL 格式

```
http://192.168.10.1:8080/thumb/<文件路径去掉开头斜杠>.jpg
```

#### 对应关系（视频和照片都有缩略图）

| 原文件路径 | 缩略图 URL |
|---|---|
| `/mnt/DCIM/Normal/VID_20260522_143000.MP4` | `http://192.168.10.1:8080/thumb/mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg` |
| `/mnt/DCIM/Event/EVT_20260522_143000.MP4` | `http://192.168.10.1:8080/thumb/mnt/DCIM/Event/EVT_20260522_143000.MP4.jpg` |
| `/mnt/DCIM/Photo/IMG_20260522_143500.jpg` | `http://192.168.10.1:8080/thumb/mnt/DCIM/Photo/IMG_20260522_143500.jpg.jpg` |

**规则：** URL 路径 = `/thumb/` + 原文件 path 去掉开头 `/` + `.jpg`

#### 你要返回

| 响应头 | 值 |
|---|---|
| **Content-Type** | `image/jpeg` |
| **Content-Length** | 缩略图文件大小（字节） |
| **Cache-Control** | `public, max-age=86400`（建议，让 App 图片库缓存） |
| **Body** | JPEG 图片二进制数据 |

#### 缩略图规格

| 参数 | 值 |
|---|---|
| 格式 | JPEG |
| 建议尺寸 | 320 × 240（保持原始宽高比） |
| 建议质量 | 75%（平衡大小和清晰度） |
| 单张大小 | 10-30 KB |

#### 你要做的事

1. **录像停止时**：从视频文件提取一帧（建议第 1 秒），生成缩略图
2. **拍照完成时**：对原图做缩放，生成缩略图
3. **缩略图存储路径**：`/mnt/DCIM/.thumbnails/` 目录，命名为 `原文件名.jpg`
   - 视频：`/mnt/DCIM/.thumbnails/VID_20260522_143000.MP4.jpg`
   - 照片：`/mnt/DCIM/.thumbnails/IMG_20260522_143500.jpg.jpg`
4. **HTTP 路由映射**：收到 `/thumb/mnt/DCIM/Normal/VID_xxx.MP4.jpg` 请求时，读取 `/mnt/DCIM/.thumbnails/VID_xxx.MP4.jpg` 返回
5. **缩略图不存在时**：返回一张内置的默认占位图（灰色或带相机图标的 JPEG）

#### 路由实现伪代码

```c
// HTTP 请求：GET /thumb/mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg
// 提取路径：mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg
// 取文件名：VID_20260522_143000.MP4.jpg
// 读取文件：/mnt/DCIM/.thumbnails/VID_20260522_143000.MP4.jpg
// 返回 JPEG 二进制
```

**curl 自测：**

```bash
# 视频缩略图
curl -s -o thumb_video.jpg "http://192.168.10.1:8080/thumb/mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg"
ls -la thumb_video.jpg   # 应该 10-30KB

# 照片缩略图
curl -s -o thumb_photo.jpg "http://192.168.10.1:8080/thumb/mnt/DCIM/Photo/IMG_20260522_143500.jpg.jpg"
ls -la thumb_photo.jpg   # 应该 10-30KB

# 验证 Content-Type
curl -sI "http://192.168.10.1:8080/thumb/mnt/DCIM/Normal/VID_20260522_143000.MP4.jpg" | grep -i content-type
# 期望：Content-Type: image/jpeg
```

---

### 8.3 在线查看/下载文件

**优先级：P0**

> **这个接口是整个相册功能的核心。** App 用这个接口在线查看图片、在线播放视频、下载文件到手机。不是"先下载完再看"，而是**边下边播**——App 播放器直接用这个 HTTP URL 播放视频，和看网页视频一样。

```
GET /api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `path` | string | 是 | 文件完整路径 |

#### 你要返回的响应头

```http
HTTP/1.1 200 OK
Content-Type: video/mp4
Content-Length: 52428800
Accept-Ranges: bytes
```

| 响应头 | 说明 |
|---|---|
| `Content-Type` | 根据文件类型返回：`.mp4` → `video/mp4`，`.jpg` → `image/jpeg`，`.mov` → `video/quicktime` |
| `Content-Length` | 文件总大小（字节） |
| `Accept-Ranges: bytes` | **必须返回**，告诉 App 播放器"我支持 Range 请求"，播放器看到这个头才会做在线播放 |
| Body | 文件二进制数据 |

#### 支持 Range 请求（在线播放和断点续传都靠它）

App 播放器在线播放视频时，不会一次请求整个文件，而是分段请求：

- 先请求文件开头的一小段（读 moov atom，获取视频元信息）
- 然后按播放进度分段请求后续数据
- 用户拖动进度条时，直接跳到对应字节位置请求

请求头：
```
Range: bytes=0-65535
```

响应（返回 206，不是 200）：
```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 0-65535/52428800
Content-Length: 65536
Accept-Ranges: bytes
```

**用户拖动进度条到中间位置时：**
```
Range: bytes=26214400-
```

响应：
```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 26214400-52428799/52428800
Content-Length: 26214400
Accept-Ranges: bytes
```

**你必须正确支持 Range 请求。** 如果不支持，App 端视频无法在线播放、无法拖动进度条、大文件下载也会失败。

#### Range 请求的三种格式（都要支持）

| 请求格式 | 含义 | 响应 |
|---|---|---|
| `Range: bytes=0-999` | 请求第 0~999 字节 | 206 + `Content-Range: bytes 0-999/总大小` |
| `Range: bytes=1000-` | 请求第 1000 字节到末尾 | 206 + `Content-Range: bytes 1000-末尾/总大小` |
| 无 Range 头 | 请求完整文件 | 200 + 完整文件 |

#### MP4 文件的 moov atom 必须在文件头部

> **这是在线播放能不能秒开的关键，非常重要。**

MP4 文件有一个元数据块叫 `moov atom`，记录了视频时长、关键帧索引、编码参数等。播放器必须先读到 moov 才能开始播放。

| moov 位置 | App 端播放行为 |
|---|---|
| **文件头部**（faststart） | 播放器请求前几十 KB 就拿到 moov，**1-2 秒内开始播放** ✅ |
| **文件尾部**（默认） | 播放器要先 Range 请求到文件末尾拿 moov，再回到开头请求数据，**慢很多** ❌ |

**录像保存 MP4 时，必须把 moov atom 放到文件前面。** 做法：

1. **方案 A（推荐）**：录像完成后做 faststart 后处理，把 moov 从文件尾部挪到头部。等价于 ffmpeg 的 `-movflags +faststart`
2. **方案 B**：录像编码器直接配置 moov 前置（如果芯片 SDK 支持）

**验证方法：**

```bash
# 用 ffprobe 检查 moov 位置（在电脑上测试录好的 MP4）
ffprobe -v quiet -show_entries format_tags=major_brand -show_entries stream=codec_type VID_test.MP4

# 用 AtomicParsley 查看（更直观）
AtomicParsley VID_test.MP4 -T

# 如果 moov 在 ftyp 后面（文件开头），说明是 faststart ✅
# 如果 moov 在 mdat 后面（文件末尾），需要处理 ❌

# 手动转换为 faststart（测试用）
ffmpeg -i input.mp4 -c copy -movflags +faststart output.mp4
```

#### 实现要点

1. 读取 `path` 参数指定的文件
2. 设置正确的 `Content-Type`（根据文件扩展名）
3. 设置 `Content-Length`（文件总大小或 Range 片段大小）
4. **始终返回 `Accept-Ranges: bytes` 头**
5. 如果请求头有 `Range`，解析字节范围，返回 `206 Partial Content` + `Content-Range` 头 + 对应的字节数据
6. 如果没有 `Range` 头，返回 `200 OK` + 完整文件
7. 如果文件不存在，返回 `{"code": -1, "msg": "file not found", "data": null}`
8. **录像文件保存时确保 moov atom 在文件头部**

**curl 自测：**

```bash
# 下载图片
curl -s -o photo.jpg "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Photo/IMG_20260522_143500.jpg"

# 下载视频
curl -s -o video.mp4 "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4"

# 测试 Range 请求 —— 请求前 64KB（模拟播放器读 moov）
curl -s -H "Range: bytes=0-65535" -o head.bin \
  "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4" \
  -w "\nHTTP状态码: %{http_code}\n"
# 期望输出：HTTP状态码: 206

# 测试 Range 请求 —— 请求中间位置（模拟用户拖动进度条）
curl -s -H "Range: bytes=26214400-26279935" -o mid.bin \
  "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4" \
  -w "\nHTTP状态码: %{http_code}\n"
# 期望输出：HTTP状态码: 206

# 验证 Accept-Ranges 头存在
curl -sI "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4" | grep -i accept-ranges
# 期望输出：Accept-Ranges: bytes
```

---

### 8.4 删除文件

**优先级：P1**

```
DELETE /api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4
```

#### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `path` | string | 是 | 要删除的文件路径 |

#### 你要返回

```json
{
  "code": 0,
  "msg": "ok",
  "data": null
}
```

**你要做的事：**

1. 删除原始文件
2. 同时删除对应的缩略图
3. 文件不存在时返回错误

**curl 自测：**

```bash
curl -s -X DELETE "http://192.168.10.1:8080/api/v1/media/file?path=/mnt/DCIM/Normal/VID_20260522_143000.MP4" | python3 -m json.tool
```

---
