# HDV CAM Protocol Documentation

HDV CAM App (Android `app-newCam-release.apk` + iOS `HDV CAM 1.3.3.ipa`) reverse engineering and protocol analysis documentation.

## Documents

| Document | Description |
|---|---|
| [app-newCam-release-technical-analysis.md](./app-newCam-release-technical-analysis.md) | Main overview - App architecture, protocol comparison, iOS analysis |
| [qz-protocol-overview.md](./qz-protocol-overview.md) | QZ protocol overview - network topology, 4-link architecture, initialization |
| [qz-api-contract.md](./qz-api-contract.md) | QZ API contract - 16 command codes, request/response formats |
| [qz-media-model.md](./qz-media-model.md) | QZ media & menu model - file listing, XML menus, translations |
| [qz-replica-plan.md](./qz-replica-plan.md) | QZ replica plan - phased implementation, acceptance criteria, task list |
| [mstar-protocol.md](./mstar-protocol.md) | MStar protocol - CGI interfaces, property tree |
| [yz-protocol.md](./yz-protocol.md) | YZ protocol - REST API, JSON responses, curl examples |
| [qz-embedded-engineer-guide.md](./qz-embedded-engineer-guide.md) | Guide for embedded/firmware engineers |
| [qz-flutter-app-guide.md](./qz-flutter-app-guide.md) | Guide for Flutter app developers |

## Protocol Families

The HDV CAM app supports 3 device protocol families:

| Protocol | Device IP | Style | Complexity |
|---|---|---|---|
| MStar | `192.168.1.1` | CGI property tree | Medium |
| **QZ** | `192.168.10.1` | Command code + XML + TCP/UDP | High |
| YZ | `192.168.169.1` | REST JSON | Low |

## Quick Start

- New to the project? Start with [app-newCam-release-technical-analysis.md](./app-newCam-release-technical-analysis.md)
- Implementing QZ firmware? See [qz-embedded-engineer-guide.md](./qz-embedded-engineer-guide.md)
- Building Flutter app? See [qz-flutter-app-guide.md](./qz-flutter-app-guide.md)
- Need API details? See [qz-api-contract.md](./qz-api-contract.md)
