# 代码基线、Mac 进度与验证证据

[返回首页](Home.md)

更新日期：2026-10-23。本页区分 GitHub 公开功能代码、Mac 本地开发与已完成的验收，不把文档发布等同功能代码已合并。

## 文档发布在哪里

开发者 Wiki 和各仓新 README 已单独发布到默认 `main`。功能代码仍在自己的开发分支；本次 main 提交只含 Markdown，没有把 BLE 或 APP 功能代码并入主分支。`codex/framework-wiki` 保留首次文档审查记录。

## GitHub 已公开的功能基线

| 仓库 | 功能提交 | 对应代码分支 |
|---|---|---|
| 宿主 | `b4709c0` | [codex/ble-image-push](https://github.com/M1nt-Ch0c0/photopainter-host/tree/codex/ble-image-push) |
| photoframe SDK / echo | `d2dd482` | [codex/ble-discovery-app-ota](https://github.com/M1nt-Ch0c0/photoframe/tree/codex/ble-discovery-app-ota) |
| 旧宿主通道客户端 | `0e94fab` | [codex/ble-image-push](https://github.com/M1nt-Ch0c0/ai-quota-frame/tree/codex/ble-image-push) |

这些基线支持已公开的宿主 BLE 与 ABI 2.1 示例；不能只从旧 main 源码构建，便假定具有同一能力。SDK、导入和协议按所选源码提交核对。

## 最新已核实的 Mac 本地进展

通过只读 SSH 检查源码及已提交验收 Markdown，确认 Mac 的 `codex/photoframe-ble-input` 分支已有：

| 部分 | Mac 本地提交 | 结果 |
|---|---|---|
| photoframe APP 收图 | `3b09a00`，验证记录 `bef884a` | 已实现 ABI 2.1 分块、APP 内 SHA-256、传输状态、清理和完成回执 |
| Mac 客户端 | `3008eb0`，测试 `6c28675` | 新主路径使用签名 APP 特征读写，保留旧实现及回归 |
| 宿主 | 仍为 `b4709c0` | 未刷写，未修改分区、SDK、导入表或 Registry 组件 |

GitHub 只读查验未见 `codex/photoframe-ble-input` 远端分支，因此以上新功能当时仍是 **Mac 本地已提交、尚未推送**。正式应用协议原件在 Mac 的 `photoframe/docs/ble-image-input.md`，验证记录在 `docs/validation-ble-input-2026-10-23.md`；公开后应链接对应提交，不伪造已发布的代码链接。

## 新 APP 路线已经验证的项目

- 新 `photoframe v17 / ABI 2.1` 通过 BLE 写入备用 B 槽，保留原 v15 为 previous；激活回复超时后查签名状态确认，没有重复激活。
- APP 实际使用已有宿主服务，没有必要新增宿主 ABI。新客户端不使用旧宿主 PNG opcode 15/16/17，不回退 HTTP，也不抢回当前 APP。
- Mac APP 原生 3 组 CTest 及 ASan/UBSan、宿主 86 项测试、Go unit/race/vet/Chrome/native CGO 构建通过；旧 ABI 1 APP 与 SO 仍可构建。
- 无设备 IPv4 的安全测试通过：5 MiB PSRAM 预约、部分接收后断开释放、显式取消、APP 自己的 60 秒空闲超时、坏 PNG 拒绝。
- 新 APP 的 59124 字节 mint-skin 图在 219.8 秒传完；故意丢 COMMIT 回复后，只 QUERY 同一 ID，245.5 秒恢复匹配 ID/长度/SHA 的 complete/OK 回执。严格一次 COMMIT、一次 QUERY，显示测试窗口内未重启。

该回执证明 APP 经同步显示完成最终 POWER_OFF；旧主题图的用户确认不能替代新 APP 这次测试的最终目测。

## 尚未收尾的项目

显示测试后，后续状态确认发生了 POWERON 重启。记录要求核实独立供电和 USB 连线；没有打开串口，也未重复回滚。USB 曾被 doctor 枚举，不能照抄交接时的独立供电条件声称本次也已证明。

待完成/复核：切出服务与旧代次失效、切回恢复、应用回滚、真实 GPIO 4 切换、最终人工目测及供电稳定性。相关测试代码存在，不等于所有实机项目已通过。记录中后台服务仍禁用；未看到编译/测试进程不能证明 Mac 会话仍在自动工作，也不能证明任务已全部完成。

最大 5 MiB 的证据是预约和释放，不是完整 5 MiB 图片射频传输或长期耐久。功能分支发布及最终验收完成后，再更新这里的结果和正式 APP 协议链接。

## 保留的早期证据

旧宿主 PNG 通道、Windows 1 MiB 包上传、ABI 2.1 echo 和早期运行层仍是有效兼容回归证据，但不能替代 APP 新路线验收：

- [Windows BLE 与内存](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-ble.md)
- [旧宿主 Mac 离线图片与回执](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-ble-images.md)
- [早期 ABI 2 运行层](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-runtime-v2.md)
- [APP 路线与验收要求](BLE-Image-App-Roadmap.md)
