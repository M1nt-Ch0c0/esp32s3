# 代码、部署与实测状态

[返回首页](Home.md)

更新日期：2026-10-29。当前状态按源码、构建、实际部署和实体完成证据分别记录。历史测试通过不能代替本轮新代码回归。

## 2026-10-29 夜间 main 交付（尚未实机验证）

最新夜间工作仅限离线源码、测试、构建、文档及 Git 发布，没有刷写、连接 BLE/串口、推图或修改线上 CPA/CPM。下文 0.2.1 部署和现场记录都是夜间禁令前的历史事实。

| 仓库 | main 提交 | 本轮结果 |
| --- | --- | --- |
| host | [f4453c6](https://github.com/M1nt-Ch0c0/photopainter-host/commit/f4453c646d820760102c6bcb11445d62362696f9) | 配对 v2、按键重置、撤权 drain/ACK、NVS fail-closed；90/90，0 跳过 |
| ai-quota-frame | [044256f](https://github.com/M1nt-Ch0c0/ai-quota-frame/commit/044256f4a0a52da0ae3550f862293fda07da166c) | 原生/浏览器配对、原生 Go/WASM 画面、0.2.2 插件、源鉴权修复、CI |
| photoframe | [ae194da](https://github.com/M1nt-Ch0c0/photoframe/commit/ae194dae683dcdea52854c08c271233ec4b96697) | ABI 2.2/CoC 回归补齐；不改业务 APP，不要求重装 v20 |
| console | [753505b](https://github.com/M1nt-Ch0c0/photopainter-console/commit/753505b3fbbe6e533c78dfb47927780e71d129de) | 0.4.1/build 6 配对界面；构建/签名通过，未安装 |

Web 32/32、UI 71/71、Go make check、迁移 3/3、Windows 交叉构建、APP 3 CTest + 4 入口 + 3 示例测试及固定 IDF 的 app/SO/host 构建通过。SDK header/imports 字节一致。GitHub Linux CI 记录见[晨间清单](../docs/MORNING-ACCEPTANCE.md)。这些软件结果不证明物理配对或显示成功。

提交后重建 host 为 1,132,800 字节，SHA-256 `014ce22bed5bc97564c9b6d192ede1daa705372cbd32dff9fc12c799c0343dc4`。最终交付包在 `photopainter-local/releases/pairing-v2-night-20261029/`；准确提交、大小/摘要由该包 manifest 记录，与早期候选分开。

**自动推图暂停。** 04:46:05（UTC+8）本地只读快照确认 `automatic=false`、operation 空、transfer idle、uncertain=false；今晚未修改。原生已安装的仍是早前 `0a270bee…` 客户端，线上仍以前序 0.2.1 记录为基线。0.2.2、控制台和新原生候选都没有在夜间安装。

[早上验收清单](../docs/MORNING-ACCEPTANCE.md)给出 microSD/Flash 条件、准确刷写命令、按键配对、页面/设备重启、换 owner、推图及线上安装/回滚。最终物理验收和线上实际加载留用户在场完成。

## 历史：0.2.1 部署与实机准备

用户后续明确要求完整复刻原生 ai-quota-frame，并改用物理批准/持久绑定，
因此 0.1.0 的简化 Canvas、快照-only 和手工设备令牌流程已退役。

- 当时插件 0.2.1 复用完整 Go 采集/usage/聚合/主题/量化/PNG，经 WASM 在
  浏览器执行。五主题 × 两场景已对照；四主题像素全同，mint-skin 仅有
  3 个字形灰阶阈值差，未以重画或手工补丁伪造一致。
- 浏览器配对/事务测试 30 项、隔离 UI 检查 68 项通过；完整 Go make check
  和 Windows 构建通过。界面由实际 Claude Code Opus 5.5 xhigh 设计后实现。
- Host 配对 v2：双方承诺/ECDH/HKDF、显示后功能键短按批准、NVS 持久 owner、
  8 秒长按锁存撤权、旧请求 drain/ACK、unbound tombstone、旧 token BLE/HTTP
  拒绝。89 项测试和独立审阅通过，不需要改业务 APP ABI。
- 当时的工作区固件候选 SHA-256
  `2f9f221766c9b3e308c0be34ac7d49ad79c9629b137ce38b9726113e644751d1`，
  **尚未刷入**。USB 身份已匹配，完整 16 MiB Flash 备份及设备摘要校验通过。
- 插件已热升级，所有服务器容器未重启，Qoder 不变，配置 inode 保留；
  新账号 subject 接口匿名 401、实际投影只含 sub，静态资源完整摘要吻合。
  安装备份 `/var/backups/cpa-stack/photopainter-v021-20261028T203014Z`。
- 新原生客户端已重载，采集 ready、6 账号、0 警告，自动推图保持暂停。

待用户确认 microSD 条件后，继续固件更新、真实按键配对、页面/设备重启后的
持久重连、功能键重置和新关系建立，以及图片完成回执和人工画面检查。
完整进度与产物位置见 [交付计划](../docs/pairing-and-plugin-parity.md)。

## 2026-10-29 现场恢复与后台修复

- BLE 可以发现并双向鉴权设备 `9c139eb063e8`。初始应用为故障空闲，
  `generation=0`、`loaded_app=-1`。只执行一次显式选择 photoframe 后，
  00:11:41 确认 v20 / slot 1、ABI 2.2、idle、runtime_ready/confirmed=true，
  GATT 两特征和 CoC 均已注册；uptime 连续增长，没有重启或刷写。
- 用户已目视确认恢复后的等待图片首屏、方向和颜色正常。现场表现与启动
  保护阻止自动加载吻合，但没有读取持久启动记录，不能断言最初失败或复位
  根因。宿主 NETWORK 事件覆盖启动错误的独立诊断缺陷已在源码修正，
  针对原生回归和固定 IDF 构建通过；该宿主修正未刷入设备。
- Wi-Fi 多次鉴权读取得到离线或连接中，IP 均为 `0.0.0.0`。应用恢复不等于
  Wi-Fi 已修复；本轮未重配 Wi-Fi。
- 额度采集失败由退役 Google 认证、失效本地代理，以及管理 POST 缺少匹配
  Origin 共同造成。客户端增加显式认证模式和只读 source-check；私有配置
  通过可信迁移工具原子保存，凭据和无关设置未输出或入库。
- 新后台于 00:29:42 重载，00:30:28 采集成功，6 个账号、0 警告。独立
  只读检查确认 4 个启用账号有新鲜结果；停用账号继续停用，额度耗尽仍
  按真实结果呈现。Go 单测、race、vet、macOS CGO 构建及迁移保护测试通过。
- 00:34:25 已恢复原 `automatic=true` 设置。随后一张 67,881 字节图片通过
  CoC 上传，发送阶段为 2,786 ms；最终回执未取得，原请求查询后进入
  `unknown` 并停止该图片的自动重试。不能将上传完成宣称为本轮实体显示
  成功，也不能据此认定最初设备问题已永久修复。
- CPM PhotoPainter 原生资源插件 0.1.0 已开发并安装。26 项网页协议/
  状态机测试、TS 构建、原生插件测试、C ABI/内嵌页面比对和精确 CPA
  隔离加载通过；真实 CPM 额度接口取得一个新鲜 Codex 窗口，旧窗口排除。
  Chrome 桌面/手机页面与 PNG→Go 校验通过，浏览器 BLE 物理推图尚未验收。
- 首次原子替换配置未触发插件加载，已回滚。获准暂停模型请求后，00:56:22
  仅重启 CPA 完成安装并恢复配置 inode 监听。生产菜单注册且有效，Manager
  本地资源 200 且页面摘要吻合；其他容器和 Qoder 不变。插件文件 SHA-256
  为 `a3b005e03399d13f935b3a7b05c78634f2a69fda2db4e9bf420133c3265aff45`。
  公网自动探测为 403/读超时或 200 部分响应，浏览器亦未取得完整加载
  证据；不能把本地资源验证替代公网验收。

上述修复已包含在顶部夜间 main 提交中；下面的 2026-10-27 矩阵和实测
记录保留为历史基线，不描述本夜间候选的部署状态。详细设备证据见宿主
`docs/validation-recovery-2026-10-29.md`，认证修复见客户端
`docs/source-auth-diagnostics.md`。

## 历史：2026-10-27 实现与仓库

| 组件 | 当前能力 | 源码位置 |
|---|---|---|
| 宿主 | ABI 2.2、单前台 CoC、有界队列、既有 GATT/HTTP 管理及 A/B | [photopainter-host · f742a47](https://github.com/M1nt-Ch0c0/photopainter-host/commit/f742a47) |
| 相框 APP | photoframe v20，ABI 2.2 CoC 大块收图，保留 GATT 图片协议 | [photoframe · e373687](https://github.com/M1nt-Ch0c0/photoframe/commit/e373687) |
| 电脑后台 | Mac 原生 CoC、签名握手、连续发送、设备操作与维护 IPC | [ai-quota-frame · f8a617f](https://github.com/M1nt-Ch0c0/ai-quota-frame/commit/f8a617f) |
| 可视化控制台 | 独立 SwiftUI 仓库，0.4.0 / build 5，八页设备操作；原理教程已移除 | [photopainter-console · b1f02b2](https://github.com/M1nt-Ch0c0/photopainter-console/commit/b1f02b2) |
| 规范 | ABI、协议、维护、架构与证据 | esp32s3/wiki |

上表四个实现仓库的功能提交均已推送至各自 `main`；控制台为独立公开仓库。原 `codex/l2cap-coc-single-owner` 开发过程与更早的 ABI 2.1 基线保留在 Git 历史，当前入口不再依赖未发布工作区。Wiki 随本轮交付同步更新。

## 历史：2026-10-27 部署与传输

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

## 历史：2026-10-27 设备版本整理

2026-10-27 09:01:20（UTC+8）的鉴权状态确认：设备目录仅保留 `photoframe`，原 `color-test` 已移除；A/B 两槽版本均为 v20。当前运行 A 槽（active=0），B 槽为 previous=1，pending=-1，generation=3，ABI 2.2，ready/runtime_ready/confirmed 均为 true。设备没有保留旧 v19 可执行版本；A/B 的两份 v20 用于同版本恢复。

当前发布产物统一归档在工作区外的 `photopainter-local/releases/abi-2.2-v20`。分两批清理 85 项和 23 项旧构建目录、包及日志，共 108 项、2,876,963,697 字节（约 2.88 GB / 2.68 GiB）。当前 release、实际运行文件、源码、Git 历史、凭据和 Flash 恢复备份保留有效。

## 历史：2026-10-27 验证范围

按用户敏捷开发要求，没有新增测试用例或运行全面测试套件。已完成宿主和 APP 的固定 IDF 编译、Mac 原生构建、包 manifest 检查、Flash 写入/保留区校验、鉴权状态以及实际 CoC 推图。

当时旧测试桩、ABI 次版本断言尚未适配 2.2，不能拿旧套件记录宣称当轮全面回归；该缺口已在 2026-10-29 夜间软件测试中补齐。长时间耐久、各种断线/重连/切换组合和跨平台 CoC 尚无本轮完整验证。Linux/Windows 当前仍走既有 GATT 图片路径。

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
