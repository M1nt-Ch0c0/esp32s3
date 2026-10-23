# Mac 本地接力提示词：把 BLE 收图实现到 photoframe APP

> 历史交接快照（2026-10-23）。2026-10-24 用户已授权四仓提交并合入 main，本文中的分支安排、禁止合并及“尚未开始”等状态仅反映交接当时；当前实现与剩余验收见 [状态页](wiki/Status-and-Evidence.md)。

请在当前 Mac 上继续 PhotoPainter 项目，完成下面的任务；不要只给计划。
本文件记录到 2026-10-23，用户最新的架构决定优先于旧文档中已经实现的宿主专用 PNG 通道。

## 先读：两个会话正在并行，必须遵守的工作分工

用户已让原会话专门整理 Wiki，让你在 Mac 本地继续开发。**不要让两个会话重复修改同一条分支或互相覆盖文件。**

| 会话 | 工作范围 | 分支与工作区 |
|---|---|---|
| 你：Mac 本地开发会话 | photoframe ABI 2.1 收图、Mac 客户端、必要的最小通用宿主扩展审计、测试与已授权的应用 OTA 实测 | `/Users/wcy/opensource/`，从当前代码建立独立功能分支，例如各仓 `codex/photoframe-ble-input` |
| 原会话：文档会话 | 开发者 Wiki、框架/APP/电脑端职责整理、历史文档标注和 GitHub 文档发布 | 在 Linux 工作区使用各仓 `codex/framework-wiki`；不会继续同步或修改你的 Mac 工作区、服务配置或设备 |

协作规则：

1. 先检查当前 HEAD、分支和所有未提交修改，再建立功能分支。不要 checkout、merge 或 pull 文档会话的 `codex/framework-wiki` 来进行功能开发，不要向该分支推送。
2. 交接时 Mac 留有上一轮同步的未提交文档草稿。这些不是你新增的功能代码；保留它们，明确选择文件提交，不要 `git add -A` 将全部草稿混进功能提交，也不要 `reset --hard`、`clean` 或批量覆盖。
3. 功能实现所需的 SDK 头文件/导入列表副本、打包器、C 校验器和测试由你在功能分支同步修改。不要为了避开文档会话而漏改这些必须一致的代码。
4. 若你确实新增或改变 ABI、GATT、应用协议、错误码或资源限制，在功能分支记录**实际实现的规范及测试证据**；相关协议参考文档可做必要的最小更新。不要重写整个 Wiki、其他仓库 README 或本接力提示词。文档会话基于已确认旧接口编写 Wiki，待功能完成后按你的实际变更协调合并。
5. 提交说明/最终交接给出修改的接口、文件、提交号、已测与未测项，特别标出与旧 Wiki 基线不同的内容。后续由用户决定合并顺序；可能出现的 README/协议文档冲突需人工按最终实现解决，不采用整文件覆盖。
6. 不合并主分支，不 force-push，不删除原宿主 PNG 实现与回归测试。Mac 开发不依赖文档分支完成，不需要等待原会话。

以下正文中的版本和状态是本次交接快照，开工仍要重新核对。此后的文档发布可能让 GitHub 的文档分支更新，但不会自动更新你的功能代码或代表实机发生变化。

## 用户最新要求（本次真正要做的事）

宿主应提供通用 Wi-Fi/BLE 通信、鉴权、应用 OTA、ELF 热加载和应用切换框架。
**相框 APP 通过已有 ABI 2.1 BLE 接口实现收图**：分块接收、整图校验、传输状态、
断线处理、显示完成回执都优先放进 `photoframe` APP；Mac 客户端访问该 APP 的 BLE 服务。
鉴权与硬件显示仍使用宿主接口，所有应用工作仍走唯一 App Manager。

- 用户接受先按键切到相框 APP 再推图；切出后该 APP 的收图服务必须停止并清理资源。
- 发现设备和 Wi-Fi/BLE 应用 OTA 必须由宿主常驻提供，不能依赖相框 APP 运行。
- **暂停继续扩展宿主专用 PNG 协议，暂停串口刷写宿主。**先核对实机和 ABI，优先通过现有 BLE 应用 OTA 安装新相框。
- 若已有 ABI 确实缺少必要能力，先指出具体缺项、代码证据和最小的通用宿主扩展方案；不要默默增加专用 PNG 宿主功能。
- 保留已完成的有效代码、测试和回退能力，不删除旧通道、不降级宿主，不破坏 HTTP、401/409、A/B、启动保护或 ABI 1/2 兼容。
- 在这台 Mac 上完成无相框 Wi-Fi 的发现、推图、应用更新实测，最终恢复正确主题的常驻服务。

## 工作区、必读文件与当前源码

Mac 兄弟仓库在 `/Users/wcy/opensource/`：

| 仓库 | 已提交实现基线 |
|---|---|
| `photopainter-host` | `b4709c028e6006cb97303b5b4129f83925788e82` |
| `ai-quota-frame` | `0e94faba026641ac90ce8d5f4cb9f002692bcf1a` |
| `photoframe` | `d2dd48214cbbccaca93e3a32e7ab5bd623f9dbfb` |
| `esp32s3` | `6d2d027136cc84eda82d872a5ce630532792b11e`，只保存文档 |

最初接力时已将 Mac 四仓放在 `codex/ble-image-push` 分支，并同步本轮**未提交的文档及验收记录**。原会话随后另开 `codex/framework-wiki`，只在原工作区整理文档；你的功能开发请按上面的并行分工新建分支。
首先运行各仓 `git status`、核对 HEAD，保留这些修改；不要 reset/clean 或把它们当作无关垃圾。
功能代码基线已提交并推送 GitHub。可基于当前内容创建 `codex/photoframe-ble-input` 等新分支，
各仓按阶段提交，说明动机。下面列出的待实现 APP 路线尚未开始写代码。

开工前阅读：

1. `photopainter-host/AGENTS.md`、`README.md`。
2. `photopainter-host/.agents/skills/develop-photopainter-stack/SKILL.md` 及相关 references。
3. 宿主 `docs-runtime-v2.md`、`docs-module-slots.md`、`docs-multi-apps.md`。
4. `photoframe/AGENTS.md`、`ai-quota-frame/AGENTS.md`。
5. `photopainter-host/docs/ble-protocol.md`、`docs/validation-ble-images.md`、`docs/validation-ble.md`。
6. `ai-quota-frame/docs/ble.md`、`docs/macos-deployment.md` 和本仓 `ARCHITECTURE.md`。

旧架构图描述已经安装的宿主图片通道，不代表用户同意继续沿该路线扩展；本文件开头的新要求是接下来要达到的目标。

## 实机状态：不要把宿主版本、应用版本和页面主题混为一谈

- ESP32-S3 PhotoPainter N16R8、800×480 Spectra 6。稳定设备 ID `9c139eb063e8`。
- 位于 Mac 附近，**独立 USB 电源、无电池、无 microSD、不连接 Mac 数据线**。
- BLE 测试期间 IP 为 `0.0.0.0`，无需给相框配 Wi-Fi。Mac 采集额度仍需要互联网。
- 当前宿主代码 **`b4709c0`**，bin 大小 **1106048 字节**，SHA-256：
  `17a546d0022f8a6b3fcab48bd22b3c38cd95e954b8dbf83f2581d62d6786ef63`。
- 宿主已经支持常驻发现、双向 HMAC、应用 A/B OTA、ABI 2.1 自定义 GATT，也包含先前实验的专用 PNG opcode 15/16/17。
  不要为了改路线先刷回旧固件。
- 最新 BLE 状态确认：**运行 `photoframe v15`，payload ABI 1，bank 0 / A，ready/runtime_ready/confirmed 均为 true**。
  它一直是原有应用，没有被降级；目前尚未安装 ABI 2.1 相框。
- `photoframe` A/B 版本为 **15/16**；`color-test` A/B 为 **1/3**，也是旧 ABI 1。
  测试用 `ble-echo` 已移除。不要未经核对覆盖活动槽；新相框版本号须高于现有包（例如 17，先查真实状态）。
- ABI 1 不支持自主 START，GPIO 4 切换跳过 ABI 1。新相框应使用 ABI 2.1；若验证真实按键切换，需另一个已确认 ABI 2 应用。
  不要声称当前仅两个旧 ABI 1 应用已验证了按键切换。
- 状态 JSON 顶层 `abi=1` 是旧管理封套，判断运行应用应看 `payload_abi` 和目录能力；不要因此误判宿主不支持 ABI 2.1。

宿主之前已经按用户授权更新过：完整备份 16 MiB Flash、分块校验设备摘要并保存 SHA-256，
只写 `0x10000`，factory 之外摘要保持一致。备份指针在
`~/.local/state/photopainter/ble-image-backup-location.txt`，写入元数据在
`~/.local/state/photopainter/ble-image-artifacts/flash-image-result.json`。
这些不是要求再次刷写。**禁止打开或打印 Flash/NVS 备份、secrets.env 或任何令牌日志。**

## 已有 ABI 2.1：先查实现，不要臆断缺能力

重点阅读宿主 `sdk/photopainter_app.h`、`sdk/abi2_imports.inc`、`main/app_ble.c`、
`main/app_services.c`、`main/app_manager.c`、`main/ble_transport.c`，以及
`photoframe/main/entry.c`、`components/photoframe/src/png_decode.c`、`examples/ble-echo/`。

已经确认的接口：

- `APP_MANIFEST_BLE` 声明 ABI 2.1；旧 `APP_MANIFEST` 仍是 2.0。
- `app_host_ble_register_v2` / `app_host_ble_set_v2` / `app_host_ble_unregister_v2`。
- 每应用 1 个服务、最多 4 个特征、每值最多 **128 字节**；宿主复制描述和值，不保留 ELF 回调指针。
- `APP_EVENT_BLE` 包含认证连接、断开、写入和错误事件；CONNECTED 表示宿主已鉴权。
- 宿主协议 op 13/14 已能带运行代次、服务版本和索引进行签名的应用特征读写。
  Mac 现有 `ble app-read` / `ble app-write` 已经实测；应复用其鉴权和路由机制。
- `app_host_display_v2` 已支持活动事件（禁止 STOP、每事件最多一次显示），同步等待最终 POWER_OFF。
  它为显示延长看门狗预算；普通事件预算 5 秒。不要让接收循环长时间阻塞单次事件。
- 有宿主计时器、单调时间、可失败的 RAM 帧缓存，以及 allowlist 中的 `malloc/calloc/realloc/free`。
  需要核对大缓冲的 PSRAM 分配实际行为；不能直接导入未允许的 ESP-IDF 蓝牙、任务或硬件 API。
- ABI 未导出通用 SHA API不等于必须扩宿主：先评估 APP 链入可许可、可测试的 SHA-256 实现及资源成本。

上述只是初步核对，不是已经完成“零宿主变更可行”的全面审计。特别检查：生命周期清理、
同步显示回执读取、事件错误处理、连接断开时序、128 字节分块和校验内存。发现缺口后先报告。

## 实现与验收建议

1. 相框 APP 定义有版本的应用级协议，至少涵盖开始、分块、进度、取消、校验/提交、结果查询。
   使用图片长度、SHA-256、请求 ID，明确重复/乱序/冲突分块规则和上限。复用既有 PNG 解码器，
   完整验证 800×480、非隔行、六色、不透明、无尾随数据后才调用宿主显示。
2. 接收状态和内存归 APP，断开/超时/STOP 必须释放；切出后服务和旧代次请求失效。
   明确与 HTTP PNG INPUT、计时事件、应用切换并发的处理，不能绕开 App Manager。
3. 应用级完成回执只在物理刷新和 POWER_OFF 成功后生成。回复丢失时，Mac 查询原请求，不能盲目重复刷屏。
   切出、重启、回执丢失会使结果不确定，客户端必须明确停止该图片的自动重试，不能假装永久 exactly-once。
4. Mac 的 BLE 图片发送改为访问相框 APP 服务，保留原单工作线程、最新快照、去重和错误节流。
   相框未运行时明确返回不可用/忙碌，不自动抢回应用、不回退 HTTP，不继续使用宿主 PNG opcode 15/16/17 作为新主路径。
5. 原 `internal/ble/image.go`、`image_test.go`、`hardware_image_test.go` 中的完整性、丢回复、取消和回执测试值得复用，
   但须改为测试 APP 服务，不能把旧宿主通道通过当作新架构通过。保留旧实现和回归测试。
6. 编译新相框 `.app.elf`，按原 `tools/module.py package` 格式打包；通过 BLE 暂存备用槽、显式激活并确认。
   保留旧 active 版本作为回退。不做宿主整机 OTA、不动分区表、不 provision。
7. Mac 上无设备 Wi-Fi 验证发现、应用安装/更新/回滚、APP 分块收图、坏图拒绝、断线清理、
   丢回执后查询、切出服务消失和切回恢复。实体成功必须有可信完成回执加人工目测。
8. 在功能分支记录实际协议、必要的 ABI/SDK 修改和实测证据；报告固件/应用大小、内部 RAM 和 PSRAM。按顶部的并行约定协调 Wiki/架构文档，避免和原会话重复重写。
   完成验收后再启动 Mac 常驻服务。长任务持续给中文进度，不反复询问已经授权的应用 OTA/实机测试。

## Mac 服务和配置：务必保留正确页面样式

当前 Mac LaunchAgent **未加载且禁用**，`ai-quota-frame` 进程数为 0；Windows 旧计划任务、看门狗及推图进程也已停用。
用户改路线时我们没有重新启动后台推图。保持停止，避免测试竞争。

- 仓库程序：`/Users/wcy/opensource/ai-quota-frame/dist/ai-quota-frame`。
- 运行目录：`~/Library/Application Support/ai-quota-frame`。
- 应用身份：`~/Applications/PhotoPainter Renderer.app`。
- LaunchAgent：`com.m1ntch0c0.ai-quota-frame`，进程存活检查，不新增监听端口。
- 已安装程序 SHA-256：`8c9251a94b7564465ccf897c78faafd9ad4ae3f825d9d50b1cf5f439645f1bc9`。
- 可信启动器私下加载运行目录的 `.env`；令牌已配置，不需用户重新提供，不得打印或放入命令参数。
- `PHOTOFRAME_PUSH_TRANSPORT=ble`，设备 ID 已配置，推图 URL 为空。
- **必须保留 `FRAME_THEME=mint-skin`、`DISPLAY_PROVIDERS=claude:CLAUDE,codex:CODEX,xai:GROK`、
  `REFRESH_INTERVAL=15m`、`TZ=Asia/Shanghai`。**第一次测试曾因迁移遗漏主题显示旧 classic 页面；
  现已纠正并经用户确认“已恢复，显示正常”，不要再次改回默认主题或行顺序。
- Mac 数据源使用已配置的 HTTPS 网关和 Google 服务账号文件；无需迁移密钥或新配源站 SSH 隧道。
- 取数需要此机已有代理 `127.0.0.1:7890`；运行环境已设代理。BLE 设备通信不经过代理。
- 系统默认 shell 是 fish；自动化用明确的 `/bin/sh` 或 Python subprocess，避免照搬 Bash 语法。

通过应用身份执行 CLI，保持现有 macOS 蓝牙权限：

```sh
open -n -W "$HOME/Applications/PhotoPainter Renderer.app" --args ble status
open -n -W "$HOME/Applications/PhotoPainter Renderer.app" --args ble install --file /absolute/path/photoframe-v17.pkg
open -n -W "$HOME/Applications/PhotoPainter Renderer.app" --args ble activate --app photoframe
```

`open` 的退出码仅表示启动器结果，不能单独当作 BLE 操作成功；需要捕获 CLI 的安全输出并核对签名状态。
不要打开日常服务日志，它可能含源端敏感内容。维护状态优先看 launchctl、进程和完成文件元数据。
`last-frame.png` 只在服务确认物理成功后保存；测试 PNG 文件本身不证明已经显示。

## 工具链与已通过的旧路线测试

- Go `/opt/homebrew/bin/go`，项目固定 `GOTOOLCHAIN=go1.26.0`；macOS BLE 必须启用 CGO。
- Chrome `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`。
- ESP-IDF 必须为 `5e6f53cdb31fe5708eae3f55af9737be2822db22`，elf_loader 必须是未修改的 Registry 1.3.3。
  Mac 已有 esptool Python 环境不代表完整交叉编译工具链已核验；先运行 doctor、检查 IDF 提交和子模块，再构建。
- Linux/Mac `make check`、Windows amd64 交叉构建、宿主 86 项测试（实际 C ASan/UBSan、真实 ELF 集成）均已通过。
- 旧宿主图片通道在 Mac 完成无 Wi-Fi 测试：5 MiB PSRAM 预约后断开释放、坏 PNG 拒绝、
  12915 字节图 8.9 秒传完、丢弃完成回复后 41.4 秒取得原回执，严格一次 COMMIT，没有重启。
- ABI 2.1 echo 已在 Mac 完成 v1→v2→v1、鉴权回显和切出清理，测试应用已移除。
- 随后通过旧通道推送 59124 字节正确 mint-skin 图片，取得物理完成凭据，用户目测确认正常。
  **这些证明现有能力有效，不证明本次 APP 自己收图的新路线已经完成。**
- 当前宿主比前一 BLE 版本增加 1184 字节，静态 DIRAM 增加 384 字节；空闲内部 RAM 约 163 KB、PSRAM 约 8.28 MB。
  详细取样在宿主 `docs/validation-ble-images.md`。

最后，当前用户只要求生成这份接力提示词，因此尚未开始改 APP 收图实现，也没有因新路线再刷写设备。
请从核对版本、阅读 ABI 和建立最小可测试的 APP 收图方案开始，按上述授权继续完成开发与 Mac 实测。
