# HDV CAM 协议分析文档

基于 HDV CAM App（Android `app-newCam-release.apk` + iOS `HDV CAM 1.3.3.ipa`）逆向分析整理的完整协议技术文档。

## 文档索引

### 总览

| 文档 | 说明 |
|---|---|
| [app-newCam-release-technical-analysis.md](./app-newCam-release-technical-analysis.md) | **主入口** — App 架构、三套协议横向对比、iOS 对照分析 |

### QZ 协议族（重点）

| 文档 | 说明 |
|---|---|
| [qz-protocol-overview.md](./qz-protocol-overview.md) | 协议总览 — 网络形态、4 条链路分层、初始化顺序 |
| [qz-api-contract.md](./qz-api-contract.md) | API 合同 — 16 个命令码、请求模板、返回结构 |
| [qz-media-model.md](./qz-media-model.md) | 媒体与菜单模型 — 文件列表、菜单 XML、翻译资源、8 种工作模式 |
| [qz-replica-plan.md](./qz-replica-plan.md) | 复刻实施计划 — 阶段目标、验收标准、排期、任务拆分 |

### 其他协议族

| 文档 | 说明 |
|---|---|
| [mstar-protocol.md](./mstar-protocol.md) | MStar 协议 — CGI 接口、属性树 |
| [yz-protocol.md](./yz-protocol.md) | YZ 协议 — REST 接口、JSON 响应、curl 示例 |

### REST API 接口协议（开工用，拿到就干活）

| 文档 | 给谁看 |
|---|---|
| [qz-api-embedded.md](./qz-api-embedded.md) | **嵌入式工程师** — 你是 Server 端，按文档实现所有接口，附 curl 自测 |
| [qz-api-app.md](./qz-api-app.md) | **App 工程师** — 你是 Client 端，按文档调用接口，附 Dart 代码和 Mock 数据 |

### 逆向分析参考（原协议）

| 文档 | 说明 |
|---|---|
| [qz-interface-specification.md](./qz-interface-specification.md) | 原始 QZ 协议接口规范（命令码风格，供参考） |

### 开发指南

| 文档 | 适合读者 |
|---|---|
| [qz-embedded-engineer-guide.md](./qz-embedded-engineer-guide.md) | 嵌入式/固件工程师 — 原协议实现参考 |
| [qz-flutter-app-guide.md](./qz-flutter-app-guide.md) | Flutter App 工程师 — 原协议接入参考 |

## 三套协议族

HDV CAM App 内置 3 套设备协议，根据手机连接的热点网关 IP 自动切换：

| 协议 | 设备 IP | 风格 | 复杂度 |
|---|---|---|---|
| MStar | `192.168.1.1` | CGI 属性树 | 中 |
| **QZ** | `192.168.10.1` | 命令码 + XML + TCP/UDP | 高 |
| YZ | `192.168.169.1` | REST JSON | 低 |

## 怎么看

- **刚接触项目？** 从 [主文档](./app-newCam-release-technical-analysis.md) 开始，了解全貌
- **嵌入式要开工？** 看 [qz-api-embedded.md](./qz-api-embedded.md)，你是 Server 端
- **App 要开工？** 看 [qz-api-app.md](./qz-api-app.md)，你是 Client 端
- **排期派活？** 看 [复刻计划](./qz-replica-plan.md)
- **想快速抓包验证？** 先从 [YZ 协议](./yz-protocol.md) 开始，接口最直白
