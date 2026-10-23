# ESP32-S3 开发专栏

**main 已包含配套功能代码与文档。** 常驻 BLE/ABI 2.1、相框 APP 收图、电脑端发送器及 macOS 控制台已同步到各仓主干；实机剩余验收见 [代码基线与进度](wiki/Status-and-Evidence.md)。

该仓库用于链接和描述本人在 ESP32 平台上进行的开发工作，不用于产出任何构建结果。

| 仓库 | 说明 | AI / 开发入口 |
|---|---|---|
| [esp32s3](https://github.com/M1nt-Ch0c0/esp32s3) | 本专栏，索引其它仓库 | 从下列具体项目进入 |
| [ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame) | 主机渲 800×480 六色 PNG，供 PhotoPainter 显示额度 | [`AGENTS.md`](https://github.com/M1nt-Ch0c0/ai-quota-frame/blob/main/AGENTS.md) |
| [photopainter-host](https://github.com/M1nt-Ch0c0/photopainter-host) | PhotoPainter 常驻通信、鉴权、应用 OTA/加载/切换与宿主服务框架 | [`AGENTS.md`](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/AGENTS.md) · [`develop-photopainter-stack`](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/.agents/skills/develop-photopainter-stack/SKILL.md) |
| [photoframe](https://github.com/M1nt-Ch0c0/photoframe) | 可由宿主动态加载的相框应用组件 | [`AGENTS.md`](https://github.com/M1nt-Ch0c0/photoframe/blob/main/AGENTS.md) |

## PhotoPainter 当前架构

**[PhotoPainter 应用框架开发者 Wiki](wiki/Home.md)** 是统一开发入口：涵盖 ABI、生命周期、宿主服务、应用 BLE、打包/OTA/回退、测试和发布规范。开始写 APP 时先读 [快速开始](wiki/Quickstart.md) 和 [应用 BLE](wiki/Application-BLE.md)。

开发者 Wiki 与功能代码已协调合入各仓 `main`；规范与实测边界见 [能力与证据](wiki/Status-and-Evidence.md)。GitHub 原生 Wiki 尚未初始化，本仓库 `wiki/` 是正式可维护原件。

四个仓库的职责、运行时架构图、构建依赖、A/B 回退流程及修改影响矩阵，见 **[架构总览](ARCHITECTURE.md)**。

远端 CPA 与 CMP → Google 登录网关（服务账号 ID Token）→ `ai-quota-frame` 生图及主动推送 → `photopainter-host` → 独立 A/B ELF → Spectra 6。

- [首次部署、业务更新与回退](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs-module-slots.md)
- [应用运行层（ABI 2）与功能键](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs-runtime-v2.md)
- [电脑端数据源与常驻运行](ARCHITECTURE.md#电脑端数据源与常驻运行)：网关鉴权、Windows 看门狗与 macOS 服务
- [2026-09-06 实机验证记录](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-2026-09-06.md)

宿主不再嵌入业务 ELF，业务更新无需重刷框架。ABI 2 支持自主首屏和 GPIO 4 短按切换，在 START 就绪及所需首屏后确认；旧 ABI 1 保持首次推图确认。宿主板级服务使用 ALDO4 供电。电脑端不再需要 SSH 隧道或保存管理密钥，渲染用的 Chrome 进程随服务一起回收。详细构建和维护约定以各项目文档为准，凭据及设备备份不入库。
