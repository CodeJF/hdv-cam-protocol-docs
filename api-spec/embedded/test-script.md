# curl 自测命令集

> [← 返回目录](./README.md)

把以下脚本保存为 `test.sh`，一键验证所有 P0 接口：

```bash
#!/bin/bash
BASE="http://192.168.10.1:8080"
OK=0
FAIL=0

test_api() {
  local desc=$1
  local method=$2
  local url=$3
  local body=$4
  
  echo -n "[$method] $desc ... "
  
  if [ "$method" = "GET" ]; then
    result=$(curl -s -w "\n%{http_code}" "$BASE$url")
  else
    if [ -n "$body" ]; then
      result=$(curl -s -w "\n%{http_code}" -X $method -H "Content-Type: application/json" -d "$body" "$BASE$url")
    else
      result=$(curl -s -w "\n%{http_code}" -X $method "$BASE$url")
    fi
  fi
  
  http_code=$(echo "$result" | tail -1)
  response=$(echo "$result" | sed '$d')
  
  if [ "$http_code" = "200" ]; then
    echo "OK ($http_code)"
    OK=$((OK+1))
  else
    echo "FAIL ($http_code)"
    echo "  Response: $response"
    FAIL=$((FAIL+1))
  fi
}

echo "===== P0: 基础 ====="
test_api "设备信息"      GET  "/api/v1/device/info"
test_api "存储卡信息"     GET  "/api/v1/device/storage"
test_api "相机状态"       GET  "/api/v1/camera/status"

echo ""
echo "===== P0: 录像 ====="
test_api "开始录像"       POST "/api/v1/camera/record/start"
sleep 2
test_api "录像状态确认"    GET  "/api/v1/camera/status"
test_api "停止录像"       POST "/api/v1/camera/record/stop"
sleep 1
test_api "停止后状态确认"  GET  "/api/v1/camera/status"

echo ""
echo "===== P0: 拍照 ====="
test_api "拍照"           POST "/api/v1/camera/capture"

echo ""
echo "===== P1: 媒体 ====="
test_api "普通视频列表"    GET  "/api/v1/media/files?type=video_normal&page=1&pageSize=20"
test_api "照片列表"        GET  "/api/v1/media/files?type=photo&page=1&pageSize=20"

echo ""
echo "===== P1: 设置 ====="
test_api "菜单列表"        GET  "/api/v1/settings/menus?lang=zh-CN"
test_api "切换模式"        POST "/api/v1/camera/mode" '{"mode": 4}'
sleep 1
test_api "模式确认"        GET  "/api/v1/camera/status"
test_api "切回录像"        POST "/api/v1/camera/mode" '{"mode": 0}'
test_api "同步时间"        POST "/api/v1/settings/datetime" '{"datetime": "2026-05-22 14:30:00"}'

echo ""
echo "===== 结果 ====="
echo "通过: $OK  失败: $FAIL"
```

---

