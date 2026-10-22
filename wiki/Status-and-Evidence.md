# 当前能力与验证证据

[返回首页](Home.md)

本页是 **2026-10-23 文档基线快照**，不是实时设备监控。Mac 功能会话在另一个分支继续开发；后续实现应更新接口文档和证据，不能仅凭这里的旧版本推断设备当前状态。

## 源码基线

| 仓库 | 已审查的功能提交 | 内容 |
|---|---|---|
| 宿主 | `b4709c0` | 常驻 BLE、ABI 2.1、兼容宿主 PNG 通道 |
| photoframe | `d2dd482` | 默认 ABI 2.0 相框、2.1 SDK 镜像和 echo 示例 |
| 电脑端 | `0e94fab` | BLE 发现/OTA/应用特征访问、旧宿主图片发送器及回执恢复 |

Wiki 与当前说明在各仓 `codex/framework-wiki`，仅改文档；不是已刷入固件的新版本。Mac 功能分支建议使用 `codex/photoframe-ble-input`，不得与文档分支互相覆盖。

## 已有证据

- 宿主 86 项主机测试通过，含实际协议 C 的 ASan/UBSan、真实 ELF 集成、manager 输入寿命、目录与有界启动恢复。
- 电脑端 Linux/Mac `make check` 和 Windows amd64 交叉构建通过；Mac 使用原生 CGO/Chrome。
- Windows 验证常驻发现、HMAC、重放拒绝、分块异常、断开清理、应用更新、ABI 2.1 echo、空闲过期及独立电源下完整 1 MiB 包传输。
- Mac 已验证 echo v1→v2→v1、签名回显和切出服务清理。
- 旧宿主 PNG 通道 Mac 离线测试通过：5 MiB 接收预约后断开释放、坏图拒绝、12915 字节图传输及物理完成、故意丢回复后严格一次 COMMIT/一次回执查询，没有设备重启。
- 正确主题的 59124 字节图片随后取得物理完成凭据，用户目测确认主题、三行、方向和颜色正常。

这些**不能替代新 APP 自行收图路线的验收**，也不等于完整 5 MiB 图片射频传输、所有断电点或长期耐久均已通过。

## 交接时的实机

宿主 `b4709c0`，原有 `photoframe v15 / ABI 1`，A/B 15/16；color-test 1/3 也是旧 ABI 1。源代码已经有 ABI 2 应用，不代表当时设备安装的包自动升级。只有一个 ABI 2 候选或全部旧 ABI 1 时，不能用按键验证多 APP 循环。

宿主 bin 1106048 字节，比之前 BLE 宿主多 1184 字节；静态 DIRAM 多 384 字节。旧在线与新离线 heap 不是受控同条件差值。Mac 测试 5 MiB 预约使用 PSRAM，断开后释放；具体峰值见原始记录。

交接时设备独立供电、没有电池和 SD；Mac/Windows pusher 均已停止以便下一阶段开发。后续功能会话可能改变这些状态，操作前重新检查。

## 详细记录

- [Windows BLE 与内存](https://github.com/M1nt-Ch0c0/photopainter-host/blob/codex/framework-wiki/docs/validation-ble.md)
- [Mac 离线图片与回执](https://github.com/M1nt-Ch0c0/photopainter-host/blob/codex/framework-wiki/docs/validation-ble-images.md)
- [电脑端部署与主题修正](https://github.com/M1nt-Ch0c0/ai-quota-frame/blob/codex/framework-wiki/docs/macos-deployment.md)
- [早期 ABI 2 运行层](https://github.com/M1nt-Ch0c0/photopainter-host/blob/codex/framework-wiki/docs/validation-runtime-v2.md)
- [APP 收图新路线的验收要求](BLE-Image-App-Roadmap.md)
