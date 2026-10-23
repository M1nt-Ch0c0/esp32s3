# PhotoPainter 应用框架开发者 Wiki

PhotoPainter 使用常驻宿主加载独立应用：宿主管通信、鉴权、资源与更新，应用决定业务和画面。同一时刻执行一个应用，切换时停止旧应用并释放其资源。可以把它理解为一套小型应用运行框架；底层调度仍由 ESP-IDF/FreeRTOS 提供，应用没有独立进程或内存保护。

**发布状态：Wiki 与配套功能代码均在各仓 main。** 开发前核对 [代码基线与验证进度](Status-and-Evidence.md)；源码合入不代表剩余实机验收已经通过。

## 开发前先看什么

| 你的任务 | 阅读顺序 |
|---|---|
| 第一次开发应用 | [架构与边界](Architecture.md) → [ABI 与版本](ABI-and-Versioning.md) → [快速开始](Quickstart.md) → [生命周期](Lifecycle-and-Scheduling.md) |
| 使用显示、定时或缓存 | [宿主服务](Host-Services.md) → [显示契约](Display-and-Images.md) |
| 开发应用 BLE 服务 | [宿主 BLE 协议](BLE-Protocol.md) → [应用 BLE 开发](Application-BLE.md) → [安全边界](Security.md) |
| 更新或发布应用 | [打包、OTA 与回退](Packages-and-Recovery.md) → [测试与发布准则](Testing-and-Release.md) |
| 继续相框 BLE 收图 | [已批准的 APP 收图路线](BLE-Image-App-Roadmap.md) → [当前能力与证据](Status-and-Evidence.md) |
| 排查现场问题 | [维护与故障排查](Operations.md) → [术语与常见问题](FAQ.md) |
| 修改框架或规范 | [兼容性规则](ABI-and-Versioning.md) → [文档维护规则](Documentation.md) |

## 哪些内容已经实现

| 能力 | 状态 |
|---|---|
| 五个应用、各自 A/B、单 ELF 前台运行、试运行和启动保护 | 已实现 |
| ABI 2 生命周期、宿主显示/缓存/定时/时间服务 | 已实现 |
| 常驻 BLE 发现、双向 HMAC、应用包上传和管理 | 已实现，Windows / Mac 有实机证据 |
| ABI 2.1 应用自定义 BLE 服务和切出清理 | 已实现，echo 示例有 Mac 实机证据 |
| 宿主专用 BLE PNG 通道 | 已验证的兼容实现；暂停继续扩展 |
| photoframe APP 自己分块收图和回执、Mac 使用 APP 服务 | 已合入 main 并通过部分实测；待剩余验收 |
| APP 通用 Wi-Fi socket / HTTP client ABI | 当前 SDK 未提供；不能当作现有服务使用 |
| 宿主整机 OTA、应用后台任务、多进程隔离 | 当前不支持；宿主整机 OTA 不在项目范围 |

“已实现”表示仓库中有代码，不代表所有硬件场景都通过；各项验证范围见 [证据页](Status-and-Evidence.md)。

## 开发中的共同约定

1. 新应用优先使用 ABI 2；使用 BLE 服务时声明 2.1，业务版本号单独管理。
2. 应用只调用允许导入的宿主服务，在事件内完成工作，所有入口恰好报告一次结果。
3. 输入、事件和缓存借用指针不得跨事件保存；STOP 释放应用自有内存。
4. 安装先暂存备用槽，再显式激活；成功接收不等于确认版本，更不等于物理刷新。
5. 通信错误后的重试必须考虑操作可能已完成，不能盲目重放显示和激活。
6. 应用协议属于应用；设备发现、鉴权、应用安装和调度属于宿主。
7. 接口与规范一起变更，并区分实现、设计提案、测试结果和现场快照。

代码入口：[宿主](https://github.com/M1nt-Ch0c0/photopainter-host/tree/main)、[应用](https://github.com/M1nt-Ch0c0/photoframe/tree/main)、[电脑端](https://github.com/M1nt-Ch0c0/ai-quota-frame/tree/main)。完整导航见 [侧栏](_Sidebar.md)；临时 Mac 接力记录见 [交接文档](../MAC_BLE_APP_HANDOFF.md)，它不代替正式接口规范。
