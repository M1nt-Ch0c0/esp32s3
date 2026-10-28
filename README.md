# ESP32-S3 开发专栏

**配对 v2 尚未实机验证。** 新版浏览器插件复用原 Go 数据/画面，并使用功能键确认和持久配对。今晚发布源码与离线产物；刷写、按键和推图按 [早上验收清单](docs/MORNING-ACCEPTANCE.md) 在用户在场时执行。

PhotoPainter 由常驻宿主、独立相框 APP、电脑后台和 macOS 控制台组成。当前宿主提供 **ABI 2.2 单所有权 L2CAP CoC**，photoframe v20 已实机运行；一次 67,509 字节图片发送耗时 2.477 秒，并取得显示完成回执。源码提交、设备版本与验证范围见 [当前状态](wiki/Status-and-Evidence.md)。

本仓库只维护文档和导航，不产出固件或应用构建文件。

| 仓库 | 负责内容 | 开发入口 |
|---|---|---|
| [esp32s3](https://github.com/M1nt-Ch0c0/esp32s3) | Wiki、架构、跨仓规范 | [Wiki](wiki/Home.md) |
| [photopainter-host](https://github.com/M1nt-Ch0c0/photopainter-host) | ESP32-S3 通信、鉴权、A/B、ELF 加载、调度和板级显示 | `AGENTS.md`、`sdk/photopainter_app.h` |
| [photoframe](https://github.com/M1nt-Ch0c0/photoframe) | 相框 APP、PNG 校验、图片协议和显示回执 | `AGENTS.md`、`docs/ble-image-input.md` |
| [ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame) | 数据采集、渲染、GATT/CoC 发送和串行设备后台 | `AGENTS.md`、`docs/app-ble-images.md` |
| [photopainter-console](https://github.com/M1nt-Ch0c0/photopainter-console) | 独立 SwiftUI 设备操作控制台 | `AGENTS.md`、`README.md` |

## 当前链路

CPA/CPM → HTTPS 管理密钥鉴权 → `ai-quota-frame` 生成 800×480 六色 PNG → GATT 鉴权与 CoC 传输 → `photoframe` 校验与显示 → Spectra 6。保留 Google 网关的环境可显式选择 Google ID Token 模式；当前 Manager 已不接受该模式替代管理密钥。

浏览器端另提供独立的 [CPM 相框插件](wiki/Web-Bluetooth-Plugin.md)：使用访问页面的电脑蓝牙，通过现有 GATT 协议手动推图，不依赖本机原生助手。

控制台通过本机受权限保护的 Unix socket 调用后台的唯一设备队列，不建立第二条竞争的 BLE 操作链。控制台面向实际操作与诊断；开发原理、ABI 和协议统一维护在 Wiki。

- [架构与职责](ARCHITECTURE.md)
- [ABI 与版本](wiki/ABI-and-Versioning.md)
- [CoC 应用接口](wiki/Application-CoC.md)
- [设备操作与诊断](wiki/Operations.md)
- [打包、OTA 与恢复](wiki/Packages-and-Recovery.md)
- [部署与实测证据](wiki/Status-and-Evidence.md)

设备同一时刻只有一个前台 APP。GATT 保留发现、鉴权和应用安装；CoC 归当前 APP 独占，切出回收。ABI 2.1 小消息继续兼容；配对 v2 的旧 token HTTP 控制入口已退役为 410；应用内容升级与宿主固件更新是不同操作。

`wiki/` 是本项目正式可维护的 Wiki 原件。旧 [Mac 交接](MAC_BLE_APP_HANDOFF.md) 仅保留历史索引，不作为当前任务或授权来源。凭据、Flash/NVS/SD 备份和历史构建产物不入库。
