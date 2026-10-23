# 代码基线、Mac 进度与验证证据

[返回首页](Home.md)

更新日期：2026-10-24。四仓功能与文档协调合入默认 `main`；源码发布与实机验收分别记录。

## 主干源码

| 仓库 | 已包含的实现基线 | 正式入口 |
|---|---|---|
| 宿主 | `b4709c0`：常驻 BLE、ABI 2.1、A/B 管理和旧 PNG 兼容通道 | [main](https://github.com/M1nt-Ch0c0/photopainter-host/tree/main) |
| photoframe | `3b09a00`：APP BLE 收图；`e910153`：回滚与生命周期实测记录 | [应用协议](https://github.com/M1nt-Ch0c0/photoframe/blob/main/docs/ble-image-input.md) |
| 电脑端 | `3008eb0`：APP 服务发送；`93bbe07`：连接重试测试；`c61d1e2`：macOS 控制台 | [控制台说明](https://github.com/M1nt-Ch0c0/ai-quota-frame/blob/main/apps/console/README.md) |

2026-10-23 先单独发布 Wiki 文档；2026-10-24 按用户要求将本地功能代码、部署记录和文档合并。旧功能和文档分支保留历史，后续开发以 main 为共同基线。

## 本次合并验证

- 宿主 86 项 Python/C 测试全部通过（无跳过），覆盖实际 C 协议 ASan/UBSan、A/B、启动恢复和应用服务；ESP-IDF 构建使用固定提交。
- 相框原生三组 CTest、ABI 2.1 `.app.elf` 和兼容 `.so` 构建通过；SDK 头与导入表副本同宿主一致。
- 电脑端 Go 1.26.0 `make check` 通过：单元、竞态、vet、Chrome 渲染与原生 CGO 构建；SwiftUI 控制台及 Windows/Linux amd64 交叉构建通过。
- 本次只验证源码与构建，不操作设备、刷写或更改常驻服务配置。下列实体结果来自已提交的历史验收记录。

## APP 路线已有实机证据

正式记录见 [相框 APP 验收](https://github.com/M1nt-Ch0c0/photoframe/blob/main/docs/validation-ble-input-2026-10-23.md) 和 [Mac 部署](https://github.com/M1nt-Ch0c0/ai-quota-frame/blob/main/docs/macos-deployment.md)。

- 宿主仍为 `b4709c0`；APP 收图未新增宿主 ABI、SDK 或分区变更。客户端使用签名 APP 特征，不走旧 PNG opcode 15/16/17，不回退 HTTP，不抢回当前 APP。
- 无设备 IPv4 的安全测试验证 5 MiB PSRAM 预约、断线释放、取消、APP 自己的 60 秒超时和坏 PNG 拒绝。
- photoframe v17、v18 通过 BLE 写备用槽并激活，保留原 v15；v18→v15→v18 双向回滚有连续 uptime 与鉴权状态证据。
- APP 的 59124 字节 mint-skin 图片在 v18 测试中约 222.5 秒传完；丢弃 COMMIT 回复后约 247.5 秒恢复同请求完成回执，严格一次 COMMIT、一次 QUERY，窗口内未重启。
- 切出测试释放 5 MiB 接收缓冲，旧 APP 服务不可用，旧代次读写被拒绝；切回恢复有单项证据。
- 2026-10-24 控制台重新暂存并激活 v18；即时状态查询失败后未重复激活，独立鉴权确认 v18 / B、previous A、ready/confirmed=true。后续自动推图取得完成回执，用户已恢复自动推图。

## 仍待验收

完整生命周期测试尚未通过：一次失败于重新鉴权，另一次切换后 uptime 回退、reset reason=1，证明发生重启。单次成功的回滚、切出和恢复不能代替完整生命周期通过；复位码不能单独确定 USB 或供电是原因。

真实 GPIO 4 切换、最新 APP 画面的最终人工目测及长期稳定性仍待验证。历史独立供电证据不代表后续 USB 连线条件相同。最大 5 MiB 证据是预约与释放，不是完整 5 MiB 图片射频传输。源码合并不会改变这些结论。

## 保留的早期证据

旧宿主 PNG、Windows 1 MiB 包上传、ABI 2.1 echo 与早期运行层仍是兼容回归证据：

- [Windows BLE 与内存](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-ble.md)
- [旧宿主 Mac 离线图片与回执](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-ble-images.md)
- [早期 ABI 2 运行层](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-runtime-v2.md)
- [APP 路线与验收要求](BLE-Image-App-Roadmap.md)
