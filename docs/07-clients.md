# 电脑端、控制台与网页插件

这一页讲相框的三种控制端由什么组成，各有哪些命令和按钮。三种控制端是：

- 电脑上的 `ai-quota-frame` 程序，包括命令行和常驻后台；
- macOS 上的图形控制台 `photopainter-console`；
- 挂在 CPM 管理中心里的网页插件。

使用者可以把这一页当作命令和配置手册，开发者可以从这里了解各部分的分工。想先推上第一张图，请看[上手使用](02-getting-started.md)。蓝牙协议本身在[蓝牙通信](06-bluetooth.md)里讲。

## 整体结构

所有控制端做的事情都一样，分三步：

1. **采集**：从 CLIProxyAPI（简称 CPA，一个代理多家 AI 服务的程序）的管理接口读取各账号的额度。如果还配置了 cpa-manager-plus（简称 CPAMP，CPA 的用量统计工具），也会读取今天和最近 7 天的 token 用量。
2. **渲染**：把额度数据填进 HTML 模板，排版成 800×480 的画面，再量化成相框能显示的六色 PNG。
3. **推送**：通过蓝牙把 PNG 发给相框上的 `photoframe` 应用，等它刷新完电子纸，并发回签名的完成回执。

采集和渲染是同一套 Go 代码，编译成两种形态：

```mermaid
flowchart TB
    subgraph Go["ai-quota-frame 的 Go 代码"]
        S[sources / quota / usage<br/>采集额度和用量]
        R[dashboard<br/>HTML 模板、主题、图标]
        Q[frame<br/>六色量化、PNG 编码]
        S --> R --> Q
    end
    Go -->|原生编译| N["ai-quota-frame daemon<br/>用无头 Chrome 排版 HTML"]
    Go -->|编译成 WASM| W["网页插件<br/>用访问者的浏览器排版 HTML"]
    N -->|Go 写的蓝牙客户端<br/>Mac 用 CoC，Windows 和 Linux 用 GATT| F[相框 photoframe 应用]
    W -->|TypeScript 写的 Web Bluetooth 客户端<br/>GATT| F
```

- **原生程序**：`ai-quota-frame daemon` 在电脑上常驻。它用无头 Chrome（没有窗口、在后台运行的 Chrome）把 HTML 排成图片，再用 Go 写的蓝牙客户端推送。
- **浏览器**：`cmd/web-frame` 把采集、模板和量化代码编译成 WebAssembly（WASM）。浏览器排版同一份 HTML，页面里的 TypeScript 代码再通过 Web Bluetooth 推送。

两边共用模板和量化代码，所以同样的数据生成的画面一致。

相框同一时间只认一个主人，见[上手使用](02-getting-started.md#一台相框只认一个主人)。

## ai-quota-frame 命令行

`ai-quota-frame` 通过子命令完成各项工作。不带参数运行时，它只打印帮助。

| 子命令 | 作用 |
|---|---|
| `daemon` | 启动常驻服务，循环采集、渲染、推送。配置全部从环境变量读取 |
| `ble` | 通过蓝牙配对、查看和管理相框。只需要蓝牙，Chrome、数据源密钥和后台都用不到 |
| `console` | 在 macOS 上通过本机 socket 向正在运行的后台发请求 |
| `source-check` | 只检查一次数据源能否正常采集，然后退出 |
| `help` | 打印帮助 |

### `ble` 子命令

```text
ai-quota-frame ble <命令> [--device-id ID] [--app APP] [--file 文件] [--name 名称] [--characteristic N] [--text 文本]
```

通用参数：

| 参数 | 默认值 | 说明 |
|---|---|---|
| `--device-id` | 环境变量 `PHOTOFRAME_DEVICE_ID`，否则用最近一次配对的设备 | 12 位十六进制设备 ID |
| `--app` | `photoframe` | 应用 ID，用于推图和应用管理命令 |
| `--file` | 无 | `push` 用的 PNG，或 `install`/`inspect` 用的应用包 |
| `--name` | `PhotoPainter desktop` | 配对时显示在相框上的控制端名字 |
| `--characteristic` | `0` | `app-read`/`app-write` 的应用特征序号，0–3 |
| `--text` | 空 | `app-write` 写入的 UTF-8 文本，最多 128 字节 |

命令列表：

| 命令 | 作用 | 会改变相框状态吗 |
|---|---|---|
| `scan` | 扫描 10 秒，列出附近相框的设备 ID 和广播信息（不需要配对） | 否 |
| `pair` | 和相框配对：打印六位码，等你在相框上短按功能键 | 是，建立主人关系 |
| `paired` | 显示本机保存的配对摘要（不含密钥） | 否 |
| `pair-status` | 读取相框公开的配对状态 | 否 |
| `forget` | 删除本机保存的配对。相框上的配对关系要用长按功能键清除 | 只改本机 |
| `info` | 鉴权连接后输出设备信息（ID、协议版本、IP、Wi-Fi 名称摘要） | 否 |
| `status` | 读取完整的签名状态：当前应用、各应用槽、版本、复位原因等 | 否 |
| `push` | 推一张 PNG 并等待完成回执 | 刷新屏幕 |
| `inspect` | 只在本地检查应用包的格式、应用 ID、版本、ABI，不连接相框 | 否 |
| `install` | 把安装包上传到该应用的备用槽，上传后**不运行** | 写备用槽 |
| `activate` | 试运行刚上传的待激活版本，启动成功后确认 | 切换应用 |
| `switch` | 把指定应用切到前台运行（使用它已确认的版本） | 切换应用 |
| `rollback` | 回到可回退槽的版本 | 切换版本 |
| `remove` | 删除一个非运行中的应用 | 删除应用 |
| `app-read` / `app-write` | 读写当前应用自定义的 GATT 特征（见[应用接口参考](05-app-api-reference.md)） | 取决于应用 |

`bind` 和 `unbind` 是 `pair` 和 `forget` 的别名。

几条行为说明：

- `push` 先在本地检查 PNG：不超过 5 MiB、800×480、非隔行、完全不透明、只含六种颜色、CRC 校验正确、文件末尾没有多余数据。检查通过才连接相框。
- `push` 要求相框正在运行 `photoframe`，否则报收图服务不可用。需要时先用 `switch` 切过去。
- `install` 和 `push` 的超时是 90 分钟，给很慢的连接留足时间；其他命令是 11 分钟。
- `activate`、`switch`、`rollback`、`remove` 的回复丢失时，程序重新连接并读一次签名状态，确认命令是否已经生效，命令本身只发一次。超时时，程序会提示你先查 `status`。
- 结果以 JSON 输出到标准输出，进度和提示输出到标准错误。

想先生成一张画面再手动推送，可以用 `cmd/render-frame`。它读取同样的环境变量，采集一次，把 PNG 写到 `--output` 指定的文件（默认 `frame.png`），不连接相框。

```sh
set -a; . ./.env; set +a
go run ./cmd/render-frame --output frame.png
./dist/ai-quota-frame ble push --file frame.png
```

### `console` 子命令

这个子命令只在 macOS 上可用。macOS 上的后台用蓝牙推送时，会在 `~/Library/Application Support/ai-quota-frame/console/bridge.sock` 打开一个 Unix socket，只有当前用户能访问。`console` 子命令和 macOS 控制台都通过它给后台发请求。

```text
ai-quota-frame console snapshot|status|refresh|pause|resume|switch|activate|rollback|cancel-trial|inspect|install [应用或文件] [摘要]
ai-quota-frame console request '<JSON>'
```

常用的几个：

| 命令 | 作用 |
|---|---|
| `snapshot` | 返回后台的当前状态，包括是否在自动推送、最近一次采集、最近一次发送的进度和结果。这个命令**不连接相框** |
| `status` | 排队连接相框读取签名状态，结果在之后的 `snapshot` 里 |
| `refresh` | 立即采集一次额度 |
| `pause` / `resume` | 暂停或恢复自动推送，设置会持久保存 |

请求先进入队列，由后台按顺序执行。命令立即返回排队后的状态，执行结果要再用 `snapshot` 查看。后台没在运行时，命令返回“后台服务未连接”。

### `source-check` 子命令

```sh
set -a; . ./.env; set +a
./dist/ai-quota-frame source-check            # 按配置检查
./dist/ai-quota-frame source-check --static   # 强制用管理密钥模式检查
```

它采集一次额度和用量，然后输出一段 JSON，内容包括：鉴权模式、各密钥是否已配置、`quota_ok`、`usage_ok`、账号数量和状态统计、有问题的服务商、耗时。输出里没有密钥、账号名和额度数值，可以放心贴给别人看。

这个命令只采集数据，不渲染画面，也不连接相框。采集失败时退出码为 1。

## 配置项

`daemon`、`render-frame` 和 `source-check` 从环境变量读配置。仓库里的 `.env.example` 是模板，复制成 `.env` 后填写。

### 数据源

| 变量 | 默认值 | 说明 |
|---|---|---|
| `CLIPROXY_BASE_URL` | `http://127.0.0.1:8317` | CPA 管理接口地址 |
| `SOURCE_AUTH_MODE` | 自动判断 | `management-key` 表示用管理密钥；`google-id-token` 用于管理接口放在 Google 登录网关后面的情况 |
| `CLIPROXY_MANAGEMENT_KEY` | 无 | CPA 管理密钥，`management-key` 模式必填 |
| `CPAMP_BASE_URL` | 设置了 `CPAMP_ADMIN_KEY` 时为 `http://127.0.0.1:18317` | 可选，cpa-manager-plus 地址，用于显示 token 用量 |
| `CPAMP_ADMIN_KEY` | 无 | cpa-manager-plus 管理密钥；设置了 `CPAMP_BASE_URL` 就必填 |
| `GOOGLE_SERVICE_ACCOUNT_FILE` | 无 | `google-id-token` 模式的服务账号私钥文件，权限应为 0600 |
| `GOOGLE_ID_TOKEN_AUDIENCE` | 无 | `google-id-token` 模式下网关的 `client_id` |

不写 `SOURCE_AUTH_MODE` 时，程序这样选择：只配了服务账号文件，就用 `google-id-token`；否则用 `management-key`。两种凭据都配了时，程序报错，请写明 `SOURCE_AUTH_MODE`。

### 相框与推送

| 变量 | 默认值 | 说明 |
|---|---|---|
| `PHOTOFRAME_PUSH_TRANSPORT` | `ble` | 推送方式，填 `ble` |
| `PHOTOFRAME_DEVICE_ID` | 最近一次配对的设备 | 指定推给哪台相框，必须已经和它配对过 |
| `PHOTOFRAME_BLE_BINDING_FILE` | 用户配置目录下的 `ai-quota-frame/ble-device.json` | 可选，改设备选择文件的位置（绝对路径） |
| `PHOTOFRAME_BLE_BOND_DIR` | 上面文件旁边的 `ble-bonds/` | 可选，改配对密钥目录的位置（绝对路径） |
| `LAST_FRAME_PATH` | 无 | 可选，每次确认显示成功后把 PNG 另存到这里 |

### 画面

| 变量 | 默认值 | 说明 |
|---|---|---|
| `FRAME_THEME` | `classic` | `classic`、`arcade`、`pop`、`mint-skin`，写 `mint` 等同于 `mint-skin` |
| `DISPLAY_PROVIDERS` | `codex,xai,kimi` | 要显示的服务商，1–5 行，格式见下 |
| `TZ` | `Asia/Shanghai` | 画面上时间使用的时区 |

`DISPLAY_PROVIDERS` 的格式：

- 用逗号分隔，每一项占画面上的一行，写成 `ID[+ID...][:显示名]`。
- 可用的 ID 是 `codex`、`claude`（也可写 `anthropic`）、`antigravity`、`xai`（也可写 `grok`）、`kimi`。
- 显示名最长 12 个字符，省略时用下表里的默认名称。
- 每个服务商只能出现一次。

### 采集节奏

| 变量 | 默认值 | 说明 |
|---|---|---|
| `REFRESH_INTERVAL` | `5m` | 采集间隔，最少 `1m` |
| `REQUEST_TIMEOUT` | `20s` | 单个请求超时，1 秒到 2 分钟 |
| `MAX_CONCURRENCY` | `4` | 同时查询的账号数，1–32 |
| `PASSIVE_MAX_AGE` | `24h` | 实时查询失败时，可以改用之前缓存的数据，这里设定缓存最多能用多久 |
| `DEMO_MODE` | `false` | 设为 `true` 时只用演示数据 |
| `CHROME_BIN` | 自动查找 | Chrome/Chromium 的路径 |

### 主题与服务商

| 主题 | 网页名称 | 说明 |
|---|---|---|
| `mint-skin` | 薄巧 | 插画背景。3 行以内正常显示；4–5 行时每行压缩成一行额度条，5 行时不显示重置时间 |
| `classic` | 经典 | 基础布局 |
| `arcade` | 街机 | 像素画装饰 |
| `pop` | 波普 | 像素画装饰 |

| 服务商 ID | 画面默认名称 |
|---|---|
| `codex` | CODEX（ChatGPT） |
| `claude` | CLAUDE |
| `antigravity` | ANTIGRAVITY |
| `xai` | GROK |
| `kimi` | KIMI |

所有主题最终都量化成黑、白、红、黄、绿、蓝六种颜色。图标和装饰按整像素绘制，所以电子纸上的画面干净、没有噪点。

## 后台是怎么推图的

- 每次采集成功，后台把最新的数据交给专门的推送线程。队列只保留最新一份，还没发出的旧数据会被替换。
- 采集失败时，屏幕保留上一张图。
- 画面内容和上次一样时，跳过这次推送。
- 推送前，后台先读相框状态，确认相框正在运行 `photoframe`，并且收图服务已经就绪。切换应用要由人来操作。
- 推送按[图片事务](06-bluetooth.md#7-图片事务)进行：上传、提交、等完成回执。提交后连接断开时，后台只查询原来那次请求的结果。查不到结果（比如相框重启了），就把这张图标为“结果未知”，停止对它的自动重试，等下一次采集出新画面。
- 没有配对关系时，后台直接报“需要配对”。

## 各平台差异

| 平台 | 蓝牙实现 | 图片通道 | 构建方式 |
|---|---|---|---|
| macOS | CoreBluetooth，需要 CGO | 相框应用支持时走 CoC，约 25 KiB/s；否则走 GATT | `make build`，默认启用 CGO，需要 Xcode Command Line Tools |
| Windows | WinRT 蓝牙接口 | GATT，每块 106 字节 | `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./cmd/ai-quota-frame` |
| Linux | BlueZ，通过 D-Bus 调用 | GATT，每块 106 字节 | `make build`，静态编译 |

- CoC 和 GATT 的区别见[三种承载方式](06-bluetooth.md#三种承载方式)。Windows 和 Linux 客户端每发一块 106 字节的数据都要等相框回复，所以比 Mac 慢很多。
- macOS 上的蓝牙功能需要 CGO，构建时保持默认的 `BUILD_CGO_ENABLED=1`。
- macOS 按应用授予蓝牙权限，所以常驻运行要用 `deploy/install-macos.py` 安装，命令行操作用 `deploy/pp.py`，见[上手使用](02-getting-started.md#4-在-mac-上装成常驻服务)。
- `console` 子命令和 macOS 控制台只在 macOS 上可用。其他平台上，停止服务就能暂停自动推送。

### Mac 上的文件位置

| 内容 | 位置 |
|---|---|
| 程序、`.env`、`start.sh`、USB 维护脚本 | `~/Library/Application Support/ai-quota-frame/` |
| 配对关系 | 同目录下的 `ble-device.json` 和 `ble-bonds/` |
| 控制接口 socket、暂停设置、USB 备份 | 同目录下的 `console/` |
| 应用身份 | `~/Applications/PhotoPainter Renderer.app` |
| LaunchAgent | `~/Library/LaunchAgents/com.m1ntch0c0.ai-quota-frame.plist` |
| 日志 | `~/Library/Logs/photopainter-render.log`、`photopainter-render.error.log` |

`install-macos.py` 有三个参数：

| 参数 | 说明 |
|---|---|
| `--env-file` | 必填，配置文件路径 |
| `--binary` | 程序路径，默认 `dist/ai-quota-frame` |
| `--no-start` | 安装文件，但让服务保持禁用 |

每次升级后台都要重新运行它。它会停掉旧服务、替换文件，重新生成应用和 LaunchAgent。

## macOS 控制台

`photopainter-console` 是一个独立的 SwiftUI 应用（`PhotoPainter Console.app`），系统要求见[版本与已知限制](09-versions-and-limits.md#组件版本)。它通过上面的 Unix socket，把所有操作交给正在运行的后台执行，自己不连蓝牙。

```mermaid
flowchart LR
    C[PhotoPainter Console] -->|Unix socket 请求| B[ai-quota-frame daemon]
    P[自动推图] --> Q
    B --> Q[唯一的设备操作队列]
    Q -->|蓝牙| F[相框]
    Q -->|USB 串口| U[相框 USB 维护]
```

后台只有一个设备操作队列。控制台的手动操作和自动推图都在这个队列里排队，一次只做一件事，所以同一时间只有一个操作在连相框。控制台要求后台正在运行，后台没运行时，界面会提示未连接。

控制台的页面：

| 页面 | 能做什么 |
|---|---|
| 运行总览 | 自动推送开关、立即采集、当前通道（GATT 或 CoC）、发送进度和速率、最近一次确认显示的画面 |
| 设备与应用 | 查看完整设备状态和各应用两个槽的版本；检查、安装、激活、切换、回滚、删除应用；结束试运行；使用指定槽的版本；把 ELF 打包成安装包 |
| 连接与发现 | 扫描、配对（显示六位码进度）、查看配对状态、忘记本机配对、启停后台服务 |
| 图片推送 | 检查自选 PNG，确认后单次推送，显示实际通道和结果 |
| 应用通信 | 读写当前应用的 0–3 号特征，支持文本、十六进制和 Base64 |
| Wi-Fi 配置 | 编辑 SD 卡或本地私有文件里的 `wifi.json`（增删、调整优先级） |
| USB 维护 | 列出串口、完整备份 Flash、更新宿主、从备份恢复、读取串口启动日志、生成和写入 NVS 配置。详见[维护与排错](08-maintenance.md) |
| 诊断 | 检查本机环境、诊断设备、列出每项功能是否可用及原因、导出诊断信息 |

几个细节：

- 安装应用或推图前，控制台先让后台检查文件并算出摘要。你确认后，后台在执行前再核对一次摘要，文件变了就停止。
- 有图片的结果还没确定时，切换应用、安装、结束试运行和 USB 写入都暂时不可用，以免丢掉那次结果。
- “使用指定槽的版本”会自动选择合适的命令。想用的版本是当前版本、待激活版本或可回退版本时，控制台分别用切换、激活或回滚完成。如果目录里已经不再记录这个槽，控制台会请你选择同一应用、同一版本的安装包，重新上传后激活。
- USB 写操作开始前，后台会自动暂停自动推送。

构建方法：在 `photopainter-console` 仓库里运行 `sh build.sh`，产物是 `dist/PhotoPainter Console.app`。

## CPM 网页插件

### 它是什么

CPM 是 CPA 的网页管理中心。网页插件是一个 CPA 插件，用 Go 编写，编译成 Linux 动态库（`.so`），由 CPA 加载。插件在服务器上只做两件事：

1. 提供三个静态文件：页面 `panel`、`frame.wasm`（gzip 压缩）和 `wasm_exec.js`，路径在 `/v0/resource/plugins/photopainter/` 下。CPM 的插件菜单里会因此出现“发送到相框”。
2. 提供一个只读接口 `GET /v0/management/plugins/photopainter/account-subject?name=...`，访问时需要 CPM 的管理鉴权。它只返回 xAI 账号的 `subject` 字段，采集 Grok 额度时要用到。

采集、渲染、配对和推图全部在访问页面的那个浏览器里完成，插件沿用 CPA 现有的端口。

```mermaid
sequenceDiagram
    participant B as 浏览器页面
    participant C as CPA/CPM 服务器
    participant F as 相框
    B->>C: 加载页面和 WASM
    B->>C: 用 CPM 的管理密钥读取额度（同源请求）
    B->>B: WASM 生成六色 PNG
    B->>F: Web Bluetooth：鉴权、上传、提交
    F-->>B: 签名的显示完成回执
```

### 数据源和存储

- 页面优先使用 CPM 在同一个网站下保存的登录信息，也就是浏览器本地存储里的 `cli-proxy-auth`。在 CPM 登录过，就不用再填密钥。CPM 保存的登录指向别的服务器时，页面不使用它。
- 没有保存的登录时，可以在“数据源”里手动填管理密钥，并选择是否让这个浏览器记住。记住的密钥按 CPM 的格式做了简单混淆后保存，并没有加密。“清除手动配置”只清除插件自己的配置，CPM 的登录保持不变。
- 和相框的配对密钥保存在浏览器的 IndexedDB 里，是不可导出的 WebCrypto 密钥。清除浏览器数据会丢失配对，之后要在相框上长按重置，再重新配对。
- 管理密钥只用于向同一个网站读取额度，只保存在浏览器里，相框和服务器都拿不到。

### 页面功能

| 区域 | 按钮或设置 | 作用 |
|---|---|---|
| 顶部 | 数据源 | 查看或设置管理密钥；可以导入 `ai-quota-frame` 导出的快照 JSON，或者直接载入一张 PNG |
| 相框画面 | 更新画面 | 采集最新额度并生成预览 |
| | 显示设置 | 主题、显示的服务（1–5 个）、时区、更新间隔 |
| | 下载 PNG | 下载当前预览，和发送给相框的是同一份文件 |
| 相框 | 添加 / 添加相框 | 选择设备并配对 |
| | 连接 / 断开 | 用已有配对连接或断开 |
| | 打开相框应用 | 相框前台不是相框应用时出现，点击后让相框切到 `photoframe` |
| | 相框信息 | 设备标识、配对标识、配对时间、相框应用版本，以及“设备运行诊断”（设备状态里不涉及隐私的字段） |
| | 移除此相框 | 删除本浏览器里保存的配对，5 秒内再点一次确认。相框上的主人关系保持不变 |
| 上次发送未确认 | 核对结果 / 核对画面后清除 | 见[维护与排错](08-maintenance.md#发送结果未知) |
| 定时推送 | 启用、检查间隔（1–1440 分钟）、保持屏幕唤醒 | 页面在前台时按间隔自动采集和发送 |
| 发送状态 | 发送到相框 / 取消发送 | 发送当前预览；上传阶段可以取消 |

页面的使用步骤见[上手使用](02-getting-started.md#方式一浏览器网页插件)。

页面根据相框应用的 ABI 版本选择传输方式。相框应用支持 ABI 2.3 时，页面用 GATT 大消息传图；只支持到 ABI 2.2 时，页面改用旧的小消息接口，速度慢很多。两种方式的速度见[三种承载方式](06-bluetooth.md#三种承载方式)。设备运行诊断里的 `payload_abi_minor` 就是相框应用的 ABI 次版本号。

### 开发者工具

相框卡片下方的“开发者工具”默认折叠。它用浏览器已有的配对关系直接管理相框上的应用，不需要电脑。槽的规则见[应用目录与 A/B 槽](03-how-the-device-works.md#应用目录与-ab-槽)。

| 按钮 | 作用 |
|---|---|
| **读取槽位** | 读取签名状态，列出每个应用的 A/B 槽版本和标记：当前、可回退、待激活、运行中、试运行 |
| **选择安装包** | 在本地检查 `.pkg`：大小不超过 1 MiB，包头的标识、格式、ABI 和应用 ID 正确，ELF 的 SHA-256 一致，内容是 Xtensa ELF。检查通过才能上传 |
| **上传并激活** | 上传到备用槽，然后激活。应用重新启动，启动成功后确认新版本 |
| **仅上传** | 只上传到备用槽，标记为待激活，暂不运行 |
| 应用（下拉框） | 选择下面四个按钮要操作的应用 |
| **激活待定版本** | 激活该应用已上传的待激活版本（宿主的 ACTIVATE） |
| **切换到此应用** | 把该应用切到前台，运行它已确认的版本（SWITCH） |
| **回滚** | 回到该应用可回退槽的版本（ROLLBACK） |
| **删除此应用** | 删除该应用（REMOVE），只能删非运行中的应用。点一次后按钮变成“确认删除”，5 秒内再点才执行 |

使用规则：

- 有未确认的图片、正在配对或正在发送时，这些按钮不可用。
- 每个改变状态的命令只发一次。回复丢失时，页面重新连接并读取签名状态，请你按槽位核对结果。
- 想让两个槽都是同一个版本，就用同一个安装包“上传并激活”两次。

### 构建插件

需要 Node 22.18+、Go 1.26+、Chrome/Chromium，以及目标平台的 C 编译器。

```sh
# 网页部分的测试和检查
cd web
npm ci --ignore-scripts
npm run check                 # 单元测试 + 构建
node tools/check-frame.mjs    # 与原生渲染对照
node tools/check-ui.mjs       # 隔离浏览器里的界面检查

# 构建完整插件
cd ..
./plugins/photopainter/build.sh
```

`build.sh` 做两件事：

1. 构建网页，把 `panel.html`、`frame.wasm.gz`、`wasm_exec.js` 复制到 `plugins/photopainter/`；
2. 调用 `build-native.sh` 运行 Go 测试，编译出 `plugins/photopainter/dist/photopainter-v<版本>.so`。

插件的版本号写在 `plugins/photopainter/main.go` 的 `const version` 里，网页的版本号写在 `web/package.json` 里。

检查工具需要 Chrome，找不到时会报错，可以用 `CHROME_BIN` 指定路径。这些检查只用本地测试数据。

`.so` 要和运行 CPA 的服务器平台相同（通常是 Linux amd64），并且和服务器的 glibc 版本兼容。推荐先在本地构建好网页资源，再在和服务器发行版相同的 Go 容器里，断开网络运行 `./build-native.sh`。在 macOS 上可以用 Zig 交叉编译出 Linux 产物，它的测试要在 Linux 上运行。

构建后，可以核对 `.so` 导出的 C 接口和内嵌的网页文件是否与网页构建产物一致：

```sh
python3 plugins/photopainter/verify-native.py \
  plugins/photopainter/dist/photopainter-v<版本>.so web/dist/panel.html
```

### 发布到 CPA

1. 先备份 CPA 的配置文件和现有插件。
2. 把新的 `.so` 放进 CPA 配置项 `plugins.dir` 所指目录下的 `linux/amd64/` 子目录，文件权限 0644，确保运行 CPA 的用户能读取。每个版本用新的文件名，旧版本留着方便回滚。
3. 在 CPA 配置里启用 `plugins.configs.photopainter`。CPA 检测到配置变化后，会自动扫描并加载新版本，不用重启。
4. 修改配置文件时直接在原文件上写入。CPA 通过原文件监听配置变化，如果用“写临时文件再重命名”的方式替换文件，CPA 就收不到之后的修改，只能重启。
5. 发布后逐项核对：
   - CPM 插件菜单里出现“发送到相框”；
   - 页面能打开，显示的版本正确；
   - 不带管理鉴权访问 `account-subject` 接口，返回 401；
   - 其他插件工作正常。

**停用插件**：把它的 `enabled` 设为关闭，确认菜单和页面路径都已消失。

**回滚插件**：撤下新的 `.so`，只撤销这个插件相关的配置改动，然后等 CPA 自动重新加载。其他配置保持当前内容，以免覆盖别人后来做的修改。浏览器里还没确认的图片记录在回滚后仍然保留，按[发送结果未知](08-maintenance.md#发送结果未知)处理。
