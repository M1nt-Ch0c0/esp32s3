# ESP32-S3 开发专栏

该仓库用于链接和描述本人在 ESP32 平台上进行的开发工作，不用于产出任何构建结果。

| 仓库 | 说明 | AI / 开发入口 |
|---|---|---|
| [esp32s3](https://github.com/M1nt-Ch0c0/esp32s3) | 本专栏，索引其它仓库 | 从下列具体项目进入 |
| [ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame) | 主机渲 800×480 六色 PNG，供 PhotoPainter 显示额度 | [`AGENTS.md`](https://github.com/M1nt-Ch0c0/ai-quota-frame/blob/main/AGENTS.md) |
| [photopainter-host](https://github.com/M1nt-Ch0c0/photopainter-host) | PhotoPainter ESP32-S3 宿主固件，接收 PNG 并驱动 Spectra 6 屏幕 | [`AGENTS.md`](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/AGENTS.md) · [`develop-photopainter-stack`](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/.agents/skills/develop-photopainter-stack/SKILL.md) |
| [photoframe](https://github.com/M1nt-Ch0c0/photoframe) | 可由宿主动态加载的相框应用组件 | [`AGENTS.md`](https://github.com/M1nt-Ch0c0/photoframe/blob/main/AGENTS.md) |

## PhotoPainter 当前架构

四个仓库的职责、运行时架构图、构建依赖、A/B 回退流程及修改影响矩阵，见 **[架构总览](ARCHITECTURE.md)**。

远端额度与用量接口 → 本机 SSH 隧道 → `ai-quota-frame` 生图及主动推送 → `photopainter-host` → 独立 A/B ELF → Spectra 6。

- [首次部署、业务更新与回退](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs-module-slots.md)
- [macOS 常驻服务及 SSH 隧道](https://github.com/M1nt-Ch0c0/ai-quota-frame#macos-常驻运行)
- [2026-09-06 实机验证记录](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-2026-09-06.md)

宿主不再嵌入业务 ELF，业务更新无需重刷框架。首次成功实体刷新后确认候选，未确认重启或失败回退；相框驱动使用 ALDO4 供电。详细构建和维护约定以各项目文档为准，凭据及设备备份不入库。
