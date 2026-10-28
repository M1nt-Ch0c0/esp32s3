# 早上验收清单：配对 v2 与完整相框插件

2026-10-29。**尚未实机验证。** 今晚完成的是源码、离线验证、构建和各仓 main 发布；没有刷写、连接设备蓝牙、推图、改线上 CPA/CPM 或重启容器。下面所有设备、安装及线上命令都留到用户在场时执行。

**自动推图当前保持暂停。** 最近一次只读本地快照为 `2026-10-29T04:46:05+08:00`：`automatic=false`、`operation=null`、transfer=`idle`、`uncertain=false`、source=`ready`。这不是此刻设备在线证明。今晚没有切换该设置，也没有安装新客户端。早上先复核，再操作。

## 1. 本次交付与验收顺序

建议顺序：暂停并确认空闲 → 核对备份/产物 → 在线插件隔离加载及升级 → 仅刷 host → 浏览器配对与推图 → 重启/重置/换 owner → 决定最终自动发送端。先以浏览器作为 owner；原生端另作交接测试，最后再换回浏览器。这样符合本次插件优先目标。

| 仓库 | main 交付提交 | 内容 |
| --- | --- | --- |
| photopainter-host | `f4453c646d820760102c6bcb11445d62362696f9` | 物理配对、撤权、NVS、防重放及旧入口退役 |
| ai-quota-frame | `044256f4a0a52da0ae3550f862293fda07da166c` | 原生配对、共享 Go/WASM、Claude Code 设计界面、CPA 插件 0.2.2、数据源修复 |
| photoframe | `ae194dae683dcdea52854c08c271233ec4b96697` | ABI 2.2/CoC 兼容测试与文档；业务 APP 不变 |
| photopainter-console | `753505b3fbbe6e533c78dfb47927780e71d129de` | 0.4.1 / build 6，配对状态、六位码和本地忘记 |
| esp32s3 | 本文所在 main 提交 | 跨仓协议说明、状态与验收/回滚步骤 |

最终离线包在：

```text
/Users/wcy/opensource/photopainter-local/releases/pairing-v2-night-20261029/
```

这个目录与早期 `pairing-v2-20261029` 候选分开。`manifest.json` 记录源码提交、测试边界、产物大小/摘要；`SHA256SUMS` 用于逐文件校验。安装只使用本目录，或使用另有完整验证记录的同一源码重建产物，不混用摘要。

| 交付文件 | 用途 / SHA-256 |
| --- | --- |
| `host/photopainter_host.bin` | 仅写 factory，1,132,800 字节；`014ce22bed5bc97564c9b6d192ede1daa705372cbd32dff9fc12c799c0343dc4` |
| `client/ai-quota-frame-darwin-arm64` | 原生后台；`9e4860a85aee1c231d2fad7b50fd393a531de2a290fa4a80d2bf22d5cb57e36c` |
| `client/ai-quota-frame-windows-amd64.exe` | 交叉构建，未做 Windows 运行验收；摘要见 manifest |
| `plugin/linux/amd64/photopainter-v0.2.2.so` | Linux CI 验证产物，11,500,432 字节；`4307bcd790941cd3b0be3f36cfa0919f7a7898b2febf4f646f8f40a9f693b02b` |
| `console/PhotoPainter Console.app` 及 zip | 控制台候选；签名和 zip 摘要见 manifest |
| `reference-app/photoframe.app.elf`、`photoframe.so` | 兼容性构建留档，不要求重新安装 v20，更不能刷 `photoframe.bin` |
| `rollback/photopainter_host-abi2.2-v1-auth.bin` | 先前已部署 host；`a46cfa7d1e378291999e1e648e5af26dfdaa8c956ed903cfc405a7539e27ec97` |

本轮离线结果：host 90/90（0 跳过）；Go `make check`（单测、Chrome 对照、race、vet、CGO 构建）通过；Windows 交叉构建通过；迁移测试 3/3；Web 32/32；UI 71/71；APP 3 CTest + 4 入口测试 + 3 示例测试；固定 IDF 的 host/APP/SO 与 SwiftUI 构建通过。SDK 头文件及 54 项 imports 镜像一致，v20 ELF 的 41 个动态导入全部合法。

GitHub [Linux CI 40772972073](https://github.com/M1nt-Ch0c0/ai-quota-frame/actions/runs/40772972073) 已通过：沙箱初始化、全部 Web/Chrome/UI 检查、固定 Debian 12 builder 的原生测试与构建、Linux C ABI/精确内嵌资源验证和产物上传。首次运行的 Chrome sandbox 权限问题已修复。CI 未加载生产 CPA，也未接触设备。

浏览器与原生使用同一采集、usage、聚合、HTML/CSS/皮肤及量化代码。五主题×两场景对照：四主题逐像素相同；mint-skin 各有 3 个字形灰阶跨阈值像素差，不宣称所有平台逐字节一致。界面按实际 Claude Code Opus 5.5 xhigh 方案接入。

## 2. 开始前：暂停、备份条件与产物校验

- [ ] 保持稳定 USB 供电。确认手边是目标相框 `9c139eb063e8`，没有串口监视器或另一个蓝牙客户端占用。
- [ ] 复核自动发送已暂停。仅对已经在运行的后台执行下面本地 IPC 命令；后台已禁用/停止则保持停止，不为读取状态重新启动。

```sh
PP_RUNTIME="$HOME/Library/Application Support/ai-quota-frame"
"$PP_RUNTIME/ai-quota-frame" console pause
"$PP_RUNTIME/ai-quota-frame" console snapshot
```

预期 `state.automatic=false`、没有 operation、没有活动传输且 `uncertain=false`。`snapshot` 只读本地状态；`console status` 会连接设备，不能替代这里的命令。暂停不打断已开始的刷新。若结果未知，保留原发送器处理原请求，不以重启、USB 复位或再次 COMMIT 清掉未知状态。

- [ ] 浏览器自动发送关闭并断开设备。在确认原任务结束后，禁用/卸载原生常驻任务，避免与直接 USB helper 和浏览器争用：

```sh
PP_SERVICE="gui/$(id -u)/com.m1ntch0c0.ai-quota-frame"
launchctl disable "$PP_SERVICE"
if launchctl print "$PP_SERVICE" >/dev/null 2>&1; then
  launchctl bootout "$PP_SERVICE"
fi
```

- [ ] 确认 microSD：无卡则记录“无卡”；有卡则在空闲后关机取卡，制作完整镜像并校验，或者由用户在场明确豁免。**未知卡状态默认不刷写**。Flash 备份不包含 SD；不格式化卡，不猜测 `/dev/diskN`。完成后保持原卡内容，重新插卡/供电。
- [ ] 校验本地交付包：

```sh
PP_RELEASE=/Users/wcy/opensource/photopainter-local/releases/pairing-v2-night-20261029
cd "$PP_RELEASE"
shasum -a 256 -c SHA256SUMS
codesign --verify --deep --strict 'console/PhotoPainter Console.app'
```

预期全部 `OK` 且签名校验返回 0。任何不一致先停止安装，重新从对应提交构建并生成新清单；不能修改校验值掩盖不一致。

需要拉取源码时，各仓先检查无本地冲突，再 `git pull --ff-only origin main`，并对照上表。固定 IDF 必须为 `5e6f53cdb31fe5708eae3f55af9737be2822db22`，Registry elf_loader 1.3.3 未修改。构建命令见各仓 README；构建输出的 flash 示例不是本次刷写方法。

## 3. 插件升级：先隔离，再线上

- [ ] 按[插件安装与回滚步骤](MORNING-PLUGIN-UPGRADE.md)核对当前服务、上传到全新 staging，执行 Linux C ABI 验证和 `--network none` CPA 隔离加载。隔离实例使用空账号和测试配置，不挂载生产凭据。
- [ ] 隔离加载成功后，备份当前线上配置/插件及容器状态，仅添加 0.2.2 库并热加载。**CPA 配置原 inode 必须保留**，不原子 rename 配置、不自动重启 CPA 或整栈。
- [ ] 认证插件列表显示 0.2.2、菜单“发送到相框”存在；三个资源摘要匹配；匿名 subject 接口 401；真实 xAI 查询仅投影 subject；Qoder 和所有既有容器 ID/PID/启动时间保持原状。

预期官方 CPM 前端可打开新插件，默认 mint-skin、CLAUDE/CODEX/GROK、上海时区、15 分钟。数据源输入的是现有 **CPM 管理密钥**，只用于当前页同源采集；不是新生成的设备令牌，也不传给相框。不要把密钥放进命令参数、截图或验收记录。

若隔离验证失败，不进入生产步骤。若热加载失败，保留旧库并按补充步骤撤销本插件变更；监听器问题留现场维护窗口处理。线上历史基线为 0.2.1，备份路径 `/var/backups/cpa-stack/photopainter-v021-20261028T203014Z`，早上仍须复核实际状态，不能直接覆盖整份旧配置。

## 4. 宿主更新：备份后仅写 factory

- [ ] 按[USB 维护步骤](MORNING-USB-UPGRADE.md)用 `usb-ports` 重新枚举端口，再用 `usb-inspect` 验证交付固件。不要沿用旧 `/dev/cu.usbmodem1101`；helper 写入前还会核对 ESP32-S3 芯片和 MAC。
- [ ] `usb-backup` 保存完整 16 MiB 并与设备 MD5 比对。记录返回的 `backup_manifest` 和 SHA，不打开/上传原始 Flash。备份也会复位设备。
- [ ] 在同一终端中，按补充步骤设置 `PP_PYTHON`、`PP_TOOL`、`PP_EXPECTED_ID`、`PP_BACKUPS`、`PP_PORT` 和 `PP_HOST_BIN` 后执行：

```sh
PP_RELEASE_SHA=014ce22bed5bc97564c9b6d192ede1daa705372cbd32dff9fc12c799c0343dc4
"$PP_PYTHON" "$PP_TOOL" --action usb-flash \
  --expected-id "$PP_EXPECTED_ID" --output-root "$PP_BACKUPS" \
  --port "$PP_PORT" --baud 115200 \
  --path "$PP_HOST_BIN" --digest "$PP_RELEASE_SHA"
```

预期 `complete=true,operation=usb-flash`。helper 再做完整备份，确认分区布局，只写 `0x10000` factory；校验固件和 factory 以外区域不变，保留 Wi-Fi/NVS/APP。末行 `digest` 是操作前完整 Flash 的摘要，**不是**固件摘要。不要运行全量 `idf.py flash` 或 `usb-provision`。

既有可信备份为 `~/Library/Application Support/ai-quota-frame/console/usb-backups/20261029T025640-fqcrhobf/manifest.json`，16,777,216 字节，SHA `056a2f81a56198fbac3fd9f7ef77fc416d6fcf77d96e748beae17b563602f339`；它是今晚禁止设备操作之前的历史备份。晨间新备份优先用于本次恢复。

若写入失败先看 helper 的 `restored` 结果；不连续重试。若新固件功能异常，优先对 `rollback/photopainter_host-abi2.2-v1-auth.bin` 使用同一 factory-only 流程及对应 SHA。只有明确要恢复整个快照时才用 `usb-restore --path .../manifest.json --digest ...`；全量恢复会覆盖 Wi-Fi、配对、目录、APP 及快照后的变化。旧鉴权固件与 v2 客户端不自动兼容，回滚后自动发送继续暂停。

## 5. 浏览器首次配对与原版画面

在支持 Web Bluetooth 的桌面 Chrome/Edge 和 HTTPS 插件页面验收；先允许浏览器系统蓝牙权限。手机界面测试不等于 iOS Safari 支持 Web Bluetooth。

| 操作 | 预期结果 | 失败时处理 |
| --- | --- | --- |
| 点击添加相框，系统选择器选 `PP-XXXX`；结合已核对设备身份确认目标 | 打开页面配对浮窗，双方协商六位码 | 未发现时检查电源、浏览器权限和其他客户端占用；不重新 provision |
| 等相框实际显示配对画面，核对两边六位数字 | 页面提示可以按功能键；屏幕数字与网页一致 | 不一致不批准，取消并记录；不要反复按键 |
| 短按 **GPIO4 功能键** | NVS 保存成功，页面显示已配对/已连接 | 显示未完成前的短按不批准；丢批准回复先重连已有候选，勿急着删除关系 |
| 输入数据源管理密钥并生成预览 | 原生同款 800×480 六色画面，平台、额度、usage、主题一致；无设备令牌输入 | 采集失败不发送旧图；检查数据源鉴权，勿修改设备凭据 |
| 发送一次当前预览，保持自动发送关闭 | 先传输、再显示；最终收到匹配请求 ID、长度、SHA 和字节数的认证完成回执 | 上传 100% 不算完成；未知状态只查原请求，不盲目再次 COMMIT |
| 看实际屏幕 | 内容与预览一致、方向和六色正常，无裁切/错位 | 记录照片和安全错误信息；保留回执身份，停止自动发送 |

BOOT 是 GPIO0，电源键是 GPIO5，都不是配对功能键。电子纸断电仍保留图像，旧画面不能证明设备在线。浏览器走 GATT，不能把早前 Mac CoC 的几秒上传速度作为浏览器验收阈值。

若 host 已连接但 APP 不 ready，先记录状态；不要为配对重新安装 v20。只能在明确确认应用启动保护/当前槽状态后，使用已配对原生 owner 的显式 `switch --app photoframe` 分支恢复，并重新进行 owner 交接。不要同时启动另一个客户端争用。

## 6. 持久关系、重置与不同客户端交接

- [ ] 在没有未完成图片时刷新插件页面，再选同一设备：不再按键配对即可鉴权。管理密钥按设计可能需要重新输入，它与设备关系是两件事。
- [ ] 断开浏览器，在没有刷新/未知事务时正常重启设备，重新连接：已保存 owner 仍可使用，不重新配对；Wi-Fi 和原 photoframe APP 保留。
- [ ] 长按 **GPIO4 至少 8 秒后松开**：旧 owner 被撤销，重连鉴权失败；旧配对记录不能让设备继续信任它。页面“移除相框”只删本地记忆，不能代替设备重置。
- [ ] 再次添加并核对数字、短按批准：建立新关系。原 owner 即使保留旧记录也不能控制设备。
- [ ] 刷新中重置专项放在普通发送全部通过之后：只发一张测试图，在显示期间长按并松开。预期立即拒绝新工作，当前物理刷新及借用输入完整结束后才持久清除关系；不得断电打断刷新。旧会话可能失去最终回执，这个专项应记录为“结果未知”而非伪造成功，不重发同一 COMMIT。保留原事务直到人工确认并明确结束恢复。

需要验证 Go 客户端时，按[原生安装与回滚步骤](MORNING-NATIVE-UPGRADE.md)备份现有 app/后台/配置，使用 `--no-start` 安装夜间产物。确认暂停偏好保存；该参数本身不写 `automatic=false`。关闭浏览器自动发送和连接，完成设备物理重置后执行：

```sh
PP_REPO=/Users/wcy/opensource/ai-quota-frame
python3 "$PP_REPO/deploy/pp.py" pair --device-id 9c139eb063e8 --name 'PhotoPainter Mac'
python3 "$PP_REPO/deploy/pp.py" paired --device-id 9c139eb063e8
python3 "$PP_REPO/deploy/pp.py" status --device-id 9c139eb063e8
# 从浏览器下载已校验的当前预览，填入绝对路径；只执行一次。
python3 "$PP_REPO/deploy/pp.py" push --device-id 9c139eb063e8 --file /absolute/path/frame.png
```

本机裸 `pp` 是 `/usr/bin/pp`，不是 PhotoPainter；以上必须调用完整脚本。`paired` 仅显示本地摘要，`status` 才实际鉴权。命令启动成功不是推图成功，仍需匹配完成回执和目视检查。原生与浏览器是两个 owner；测试结束后关闭原生连接、物理重置并在浏览器重新配对，不能靠复制旧关系绕过重置。

## 7. 控制台候选与最终自动发送

控制台 0.4.1 也留在早上安装。先在菜单中退出旧控制台，再备份并安装（当前安装位置已只读核对为 `~/Applications/PhotoPainter Console.app`）：

```sh
PP_CONSOLE="$HOME/Applications/PhotoPainter Console.app"
PP_CONSOLE_BACKUP="$HOME/PhotoPainter-backups/console-$(date +%Y%m%d-%H%M%S)"
umask 077
mkdir -p "$PP_CONSOLE_BACKUP"
mv "$PP_CONSOLE" "$PP_CONSOLE_BACKUP/PhotoPainter Console.app"
ditto "$PP_RELEASE/console/PhotoPainter Console.app" "$PP_CONSOLE"
codesign --verify --deep --strict "$PP_CONSOLE"
/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' \
  "$PP_CONSOLE/Contents/Info.plist"
```

预期版本 0.4.1/build 6。若选择原生 owner 分支，可在已确认持久暂停后启动后台，再打开控制台检查 pair-status、配对进度六位码、state=4 才提示按键，以及本地 forget 的影响说明。若最终是浏览器 owner，保持原生服务停用；控制台显示未连接后台是正确状态。UI 安装失败时退出新 app，然后回滚，自动推图保持暂停：

```sh
mv "$PP_CONSOLE" "$PP_CONSOLE_BACKUP/failed-PhotoPainter Console.app"
ditto "$PP_CONSOLE_BACKUP/PhotoPainter Console.app" "$PP_CONSOLE"
codesign --verify --deep --strict "$PP_CONSOLE"
```

最终选择如下，只启用一种：

| 最终 owner | 自动发送安排 |
| --- | --- |
| 浏览器（本清单默认） | 原生保持暂停且服务禁用；完成所有验收后，用户明确开启页面“自动发送”。浏览器关闭后不会继续后台发送；需保持页面可运行 |
| 原生 Mac | 关闭浏览器自动发送并断开；物理重置后原生配对。按原生指南先启动暂停状态，确认数据源与单次刷图通过，再执行 `console resume` |
| 尚有失败/未知结果 | 两端都暂停，不用重启清掉未知事务，不恢复自动发送 |

`console resume` 可能立即发送已排队的最新画面，并非一定等待 15 分钟。验收结束前默认仍为暂停。

## 8. 验收记录

逐项填写日期、执行人、源码提交/产物 SHA、实际端口/设备 ID、Flash 和 SD 备份位置、插件实际版本与资源摘要、容器未变记录、配对码是否一致（不必保存码）、重启后重连、重置后旧 owner 拒绝、图片最终回执和目视结果。最后明确写“自动发送仍暂停”或“由哪一个 owner 开启”。不记录密钥、配对 root、完整账号文件或原始 Flash 内容。

只有软件、线上加载、物理配对/重置/持久化和图像回执/目视全部通过后，才将整体目标改为完成；本夜间交付不替代这些验收。
