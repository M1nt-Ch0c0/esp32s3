# 代码、部署与实测状态

[返回首页](Home.md)

更新日期：2026-10-27。当前状态按源码、构建、实际部署和实体完成证据分别记录。历史测试通过不能代替本轮新代码回归。

## 当前实现与仓库

| 组件 | 当前能力 | 源码位置 |
|---|---|---|
| 宿主 | ABI 2.2、单前台 CoC、有界队列、既有 GATT/HTTP 管理及 A/B | [photopainter-host · f742a47](https://github.com/M1nt-Ch0c0/photopainter-host/commit/f742a47) |
| 相框 APP | photoframe v20，ABI 2.2 CoC 大块收图，保留 GATT 图片协议 | [photoframe · e373687](https://github.com/M1nt-Ch0c0/photoframe/commit/e373687) |
| 电脑后台 | Mac 原生 CoC、签名握手、连续发送、设备操作与维护 IPC | [ai-quota-frame · f8a617f](https://github.com/M1nt-Ch0c0/ai-quota-frame/commit/f8a617f) |
| 可视化控制台 | 独立 SwiftUI 仓库，0.4.0 / build 5，八页设备操作；原理教程已移除 | [photopainter-console · b1f02b2](https://github.com/M1nt-Ch0c0/photopainter-console/commit/b1f02b2) |
| 规范 | ABI、协议、维护、架构与证据 | esp32s3/wiki |

上表四个实现仓库的功能提交均已推送至各自 `main`；控制台为独立公开仓库。原 `codex/l2cap-coc-single-owner` 开发过程与更早的 ABI 2.1 基线保留在 Git 历史，当前入口不再依赖未发布工作区。Wiki 随本轮交付同步更新。

## 已完成部署与传输

- 更新前完成全部 16 MiB Flash 备份并与设备校验。宿主只写 factory 起点 `0x10000`；`0..0x10000` 与 `0x510000..0x1000000` 的 MD5 保持一致，配置、目录和 APP 数据区未随宿主刷写改变。
- 已刷宿主为 1116144 字节，SHA-256 `a46cfa7d1e378291999e1e648e5af26dfdaa8c956ed903cfc405a7539e27ec97`。
- photoframe v20 包为 80,352 字节，SHA-256 `7814051edb6129dee2506d4ef56c82a00940264177b0671261e328adf13377cb`。鉴权状态确认 ABI 2.2、runtime_ready/confirmed=true、CoC 已注册。
- 2026-10-27 09:02:59（UTC+8）最新版 Mac 后台已重启，`automatic=true`，恢复自动推图。独立控制台 0.4.0 / build 5 已安装并打开，提供八个操作页面；“工作原理”已删除。界面覆盖设备、连接、图片、应用通信、Wi-Fi/本地 SD 文件、USB 维护和诊断，具体破坏性维护入口不因界面可用而视为已经在实机执行。

### 首轮 CoC 图片

| 项目 | 实际观测 |
|---|---|
| 图片大小 | 67,509 字节 |
| 实际通道 | l2cap-coc |
| 发送阶段 | 2026-10-27 01:48:58.373 → 01:49:00.851（UTC+8） |
| 发送耗时 / 平均速率 | 2.477 秒 / 26.6 KiB/s |
| 最终显示完成回执 | 2026-10-27 01:49:21.649（UTC+8） |
| 结果 | APP 返回匹配请求的 complete/OK，物理刷新及最终 POWER_OFF 已完成 |

发送进度代表送入本地传输流的数据量，不能单独证明对端已收到。发送耗时不包含全部发现、建连、渲染和刷屏；最终业务回执才证明显示完成。这个结果是一轮成功观测，不是所有图片大小、距离和干扰条件下的性能保证。当前画面的方向、颜色和内容仍以人工目测为准。

### 最终自动推图与服务恢复

2026-10-27 09:02:59 重启最新后台并恢复自动推图后，又完成一轮真实推送：

| 项目 | 实际观测 |
|---|---|
| 自动更新 | automatic=true |
| 图片大小 / 实际通道 | 67,857 字节 / l2cap-coc |
| 发送阶段 | 09:03:22.990305 → 09:03:25.654106（UTC+8） |
| 后台记录的发送耗时 / 平均速率 | 2,663 ms / 约 24.9 KiB/s |
| 最终显示完成回执 | 09:03:46.269207（UTC+8） |
| 结果 | complete，uncertain=false |

这轮记录证明当前已部署后台恢复了自动采集/推图，并通过CoC取得物理完成回执；不只是凌晨的独立首轮结果。显示等待仍与发送耗时分开记录。

## 设备版本整理

2026-10-27 09:01:20（UTC+8）的鉴权状态确认：设备目录仅保留 `photoframe`，原 `color-test` 已移除；A/B 两槽版本均为 v20。当前运行 A 槽（active=0），B 槽为 previous=1，pending=-1，generation=3，ABI 2.2，ready/runtime_ready/confirmed 均为 true。设备没有保留旧 v19 可执行版本；A/B 的两份 v20 用于同版本恢复。

当前发布产物统一归档在工作区外的 `photopainter-local/releases/abi-2.2-v20`。分两批清理 85 项和 23 项旧构建目录、包及日志，共 108 项、2,876,963,697 字节（约 2.88 GB / 2.68 GiB）。当前 release、实际运行文件、源码、Git 历史、凭据和 Flash 恢复备份保留有效。

## 本轮验证范围

按用户敏捷开发要求，没有新增测试用例或运行全面测试套件。已完成宿主和 APP 的固定 IDF 编译、Mac 原生构建、包 manifest 检查、Flash 写入/保留区校验、鉴权状态以及实际 CoC 推图。

旧测试桩、ABI 次版本断言仍需在将来恢复系统测试时适配 2.2；不能拿旧套件的通过记录宣称本轮已全面回归。长时间耐久、各种断线/重连/切换组合和跨平台 CoC 尚无本轮完整验证。Linux/Windows 当前仍走既有 GATT 图片路径。

## 尚未解决的现场问题

此前曾发生 POWERON 重启和启动保护回退。2026-10-27 清晨再次出现 POWERON 并回退到 v19；接入稳定 USB 后已重新安装并统一为 v20。该恢复操作不证明已查明或修复复位根因。当前固件提供 uptime/reset_reason，但没有电池电压、剩余电量或 PMIC 关机原因遥测；不能据此确认没电或归因 USB。首次宿主更新后记录过 reset_reason=6，后续观测仍需按时间线判断，不能自行归因。CoC 成功不代表上述下线问题已修复。

## 历史证据入口

以下记录按标题日期和当时版本解释：

- 2026-10-24：各仓旧 ABI 2.1 功能和 Wiki 协调合入 main；当轮的宿主 86 项、APP 原生/构建、Go make check 和 SwiftUI 构建记录见各仓历史文档。
- [2026-10-23 APP GATT 收图与回执](https://github.com/M1nt-Ch0c0/photoframe/blob/main/docs/validation-ble-input-2026-10-23.md)：59,124 字节约 222.5 秒上传，丢 COMMIT 回复后查询原回执；部分生命周期检查失败，不能称为全项通过。
- [历史宿主 BLE / Windows 证据](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-ble.md)
- [历史 Mac 兼容图片通道](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-ble-images.md)
- [早期 ABI 2 运行层](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-runtime-v2.md)

旧产物清理不删除这些已提交的事实记录；旧记录不承担当前部署状态的职责。
