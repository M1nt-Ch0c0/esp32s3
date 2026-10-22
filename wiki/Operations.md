# 维护与故障排查

[返回首页](Home.md)

先区分数据源、电脑渲染、BLE/HTTP 传输、APP 生命周期和物理显示，避免通过刷新屏幕或打开串口探测在线。

## 安全的检查顺序

1. 看主机进程/LaunchAgent，确认没有两个 pusher 或看门狗竞争。
2. BLE 扫描只检查发现，不信任身份；进一步查询必须鉴权。
3. 读取签名应用目录和当前状态，判断 loaded、runtime_ready、confirmed、pending、trial。
4. 核对 APP 服务 generation/revision。非当前应用、旧代次、未注册或发布失败都可能不可用。
5. 只有在输入已验证、操作已授权且不会产生重复刷新时才执行业务请求。

`payload_abi` 是当前 APP 的 ABI；顶层 `abi=1` 是管理封套。`display_completed` 表示最近事件，不是持久化显示历史。`last_result` 也可能来自后续网络/管理事件，不能单独推翻已有请求完成凭据。

## Mac

使用原生 CGO/CoreBluetooth，应用身份为 `PhotoPainter Renderer.app`。可信启动器加载用户私有运行配置，不在命令行传令牌。

```sh
open -n -W "$HOME/Applications/PhotoPainter Renderer.app" --args ble status
launchctl print gui/$(id -u)/com.m1ntch0c0.ai-quota-frame
# 开始独立实机操作前停止后台 pusher
launchctl bootout gui/$(id -u)/com.m1ntch0c0.ai-quota-frame
```

`open` 返回 0 只表示启动，操作成功需读 CLI 的安全结果或签名状态。Bluetooth 权限由系统设置管理。服务曾以 `--no-start` 安装时会禁用，完成对应验收后需显式 enable/bootstrap。不要把“进程在运行”误当成设备业务已成功。

迁移电脑时保留 FRAME_THEME、DISPLAY_PROVIDERS、TZ 和 REFRESH_INTERVAL。主题由电脑渲染器选择；页面变旧并不说明 APP ABI 或宿主被降级。现场私有配置和路径不应复制进通用 Wiki。

## Windows

看门狗只检查进程存在。停止计划任务可能只结束外层 wrapper，还要确认看门狗与子进程已退出。不能恢复旧的 `:8787/healthz` 轮询，否则可能强杀正常渲染、遗留浏览器或丢失操作结果。

## 常见症状

| 观察 | 可能含义与下一步 |
|---|---|
| 电子纸仍有图 | 只证明保留画面，继续检查鉴权连接 |
| BLE 可发现但鉴权失败 | 检查设备 ID、独立令牌、会话是否需要重建，不打印令牌 |
| 同网仍无法 HTTP | 检查地址、路由和客户端隔离；发现不会建立 IP 路由 |
| IP `0.0.0.0` | 无 IPv4，不妨碍常驻 BLE 或合格的 APP BLE 服务 |
| 应用写入返回 busy/state | 可能有上传/显示、应用已切出、旧代次或队列满；不紧密重试 |
| PNG 被拒绝 | 先原生验证尺寸、CRC、隔行、透明度、六色和尾随字节 |
| 管理或显示回复丢失 | 查询状态/原回执，禁止直接重发变更命令 |
| USB 消失且 uptime 变小 | 排查复位与供电；开串口本身也可能造成复位 |

历史电脑 USB 供电出现过 POWERON 复位，独立供电下通过过长包传输；这不足以确定故障部件或证明长期耐久。具体数值和环境见 [验证记录](Status-and-Evidence.md)。

设备 HTTP 的未授权请求可用于无显示的可达性检查；实际 POST /api/push 等业务操作和兼容错误码见[兼容参考](https://github.com/M1nt-Ch0c0/photopainter-host/blob/codex/framework-wiki/docs/legacy-image-input.md)。不要为排查网络问题重复 provision 或覆盖 NVS。
