# 提案：统一消息协议

> **这是一份设计提案，尚未实现。** 文中的消息格式、加密方式、接口名称都是设想，不是现有固件或客户端的接口。现在实际运行的蓝牙协议见 [蓝牙通信](../06-bluetooth.md)。

这份提案想回答一个问题：相框现在有三种和应用通信的方式（GATT 小消息、CoC、ABI 2.3 通用消息），以后可能还要加 Wi-Fi，能不能让它们共用**一套**会话、消息和传输事务，应用和客户端都只写一次业务逻辑？

适合读者：打算改动宿主协议、客户端协议核心或者 photoframe 传输层的开发者。

## 1. 要解决的问题

现在每加一种承载方式，就要在好几个地方各写一遍：

| 位置 | 现状 |
|---|---|
| 宿主 | GATT 管理命令、CoC 封套、通用消息各有一套帧格式和鉴权代码 |
| photoframe | 三个适配器分别接收图片命令，再交给共用的事务核心 |
| 电脑端（Go） | GATT 小消息和 CoC 各有一条发送路径 |
| 网页插件（TypeScript） | 自己又实现了一遍会话、签名和图片事务状态机 |

结果是：

- **同一个规则要在多处保持一致。** 例如“COMMIT 回执丢了只能查询、不能重发”，Go 和 TypeScript 各写一遍，任何一边写错都可能导致重复刷屏。
- **业务受限于最弱的通道。** 旧 GATT 每次只能带 106 字节图片，还要额外读一次回执，速度只有 CoC 的百分之一左右。
- **内容没有加密。** 现有会话只做 HMAC 签名，能防篡改和冒充，但空中的数据是明文。所以不能用它来传 Wi-Fi 密码，也就没法通过蓝牙给相框配网。
- **不能换通道接着传。** 一张图在 GATT 上传了一半，换到 CoC 只能从头再来。

第一步已经做了：photoframe 的图片事务提成了与传输无关的纯 C 核心，宿主加了 ABI 2.3 通用消息邮箱和 GATT 大消息适配器。本提案描述的是在此基础上的完整方案。

## 2. 目标与范围

目标：

- 应用只实现一次图片接收、校验和显示。
- GATT、CoC、Wi-Fi TCP/WebSocket 共用同一套会话、消息格式和事务。
- 客户端的协议核心只有一份：Go 实现，浏览器通过 Go 编译成的 WASM 复用；JavaScript 只保留 Web Bluetooth/WebSocket 驱动、界面和 WebCrypto 桥接。

第一版只做：

- 一个已授权的写入者、一个进行中的大文件事务；
- 可靠、有序的承载通道。

第一版不做：

- 多条链路并发拼包；
- 自己实现 UDP 重传；
- 跨重启的“恰好显示一次”保证。

网络发现、Wi-Fi 建链、证书配置、系统权限仍由连接管理器和各传输适配器负责。协议统一不会让浏览器获得原生系统才有的能力（例如浏览器打不开 CoC）。

## 3. 方案

### 3.1 分层

```mermaid
flowchart TB
    UI[网页或原生界面] --> Client[公共协议客户端<br/>Go，浏览器用 Go/WASM]
    Client --> Session[会话与消息层<br/>版本、身份、认证、序号、加密、错误]
    Session --> Transfer[事务层<br/>BEGIN / DATA / COMMIT / QUERY / CANCEL]
    Transfer --> Router[宿主路由与唯一 App Manager]
    Router --> Image[photoframe 图片处理]
    Router --> Package[应用包与管理处理]
    Session <--> GATT[GATT 消息适配器]
    Session <--> CoC[CoC 流适配器]
    Session <--> WiFi[TCP / WebSocket 适配器]
```

相框和客户端两端都实现会话层和事务层。固件里这两层是不依赖 NimBLE、lwIP 的纯 C 代码；客户端是 Go 代码。

### 3.2 传输适配器

每种承载方式只需要实现一个很小的接口：发一条完整消息、收一条完整消息、报告限制、关闭。

```go
type LinkLimits struct {
    MaxTxMessage, MaxRxMessage, RxBudgetBytes uint32
    ServerPush bool
}
type MessageLink interface {
    SendMessage(ctx context.Context, frame []byte) error
    ReceiveMessage(ctx context.Context) ([]byte, error)
    Limits() LinkLimits
    Close() error
}
```

适配器的约定：

- `SendMessage` 成功只表示适配器收下了整条消息，不表示设备已处理。
- `ReceiveMessage` 要么返回完整消息，要么返回明确的错误，从不交出半条。
- GATT 适配器在内部处理 MTU、分片、重组和读回复。上层看不到特征、MTU 和分片头。GATT 不能主动推送，所以报告 `ServerPush=false`，由公共客户端用 QUERY 等结果。
- CoC 和 TCP 是字节流，适配器先读公共帧头，再按声明长度读数据，处理短读、短写和 EOF。WebSocket 本身就是完整消息，仍要校验同一种帧头。
- 每条链路单独协商消息上限和接收预算。起点设想为 GATT 512 B、CoC 4 KiB、TCP/WebSocket 16 KiB，需要实测后确定。
- TCP 和 CoC 本身可靠，适配器不叠加逐包重传。适配器从不自行重发可能已经执行的 COMMIT。

### 3.3 公共消息格式

固定 24 字节头，逐字段按小端编码（不直接发送 C 结构体）：

| 字段 | 字节 | 含义 |
|---|---:|---|
| magic | 2 | 协议标识 `PP` |
| version | 1 | 线协议版本，和应用 ABI 版本无关 |
| kind | 1 | 握手、请求、响应、DATA、ACK、事件、关闭 |
| flags | 4 | 第一版必须为零 |
| session_id | 8 | 本次认证连接的随机会话标识 |
| sequence | 4 | 每个方向单调递增，溢出前必须重建会话 |
| payload_length | 4 | 消息体长度，分配内存前先检查 |

认证之后的消息都用 AEAD（带认证的加密）保护：帧头作为附加认证数据，消息尾部 16 字节认证标签。nonce 由 `session_id` 和 `sequence` 组成，两个方向用不同密钥；每次重建会话都生成新的随机数和密钥。

帧头、标签和事务字段都占空间，所以 512 字节的帧装不下 512 字节图片。实际每条 DATA 能带多少数据，由协商上限减去这些开销得出。

### 3.4 身份与会话

- 继续使用现有的配对 v2（物理按键批准）和设备 ID / CID / BID，Wi-Fi 不另设一套固定令牌。
- 每条新连接执行 HELLO → AUTH → READY。认证记录包含双方随机数、设备和客户端身份、BID、boot_id、连接绑定信息、选定的版本和能力摘要，用来防重放和防降级。
- 同一个配对关系可以授权多条通道，但每条通道都要重新认证，派生独立的方向密钥。IP 地址、SSID、“刚通过蓝牙告知的地址”都不算身份凭据。
- 浏览器里的配对密钥保持为不可导出的 `CryptoKey`，Go/WASM 核心通过密码服务接口调用 WebCrypto。原生端用各自的密钥存储实现同一接口。长期密钥不进入 URL、日志、设备网络配置或服务器插件配置。
- Wi-Fi 凭据必须在认证且加密的控制消息里交接。只做 HMAC 的现有会话不能用来传它。

### 3.5 统一事务

事务身份由五部分组成：`binding_id + boot_id + app_id + app_generation + transaction_id`。`transaction_id` 是每次用户意图的随机 128 位值，不能用图片 SHA 代替，因为同一张图可以有多次显示意图。

| 命令 | 作用 |
|---|---|
| BEGIN | 目标、事务 ID、内容类型、总长度、SHA-256；先做授权和资源准入 |
| DATA | 事务 ID、偏移、数据；只接受连续数据或内容一致的重复数据 |
| ACK | 应用已经消费并纳入校验的累计偏移，以及剩余接收预算；业务是否完成看最终回执 |
| COMMIT | 检查完整性后交给业务处理；发送端在发出前持久保存事务身份 |
| QUERY | 查询同一事务的目标代次、已收字节和最终结果 |
| CANCEL | 取消还没进入不可逆执行的事务 |

状态：RECEIVING → VERIFIED → EXECUTING → COMPLETED，另有 FAILED、CANCELLED、OUTCOME_UNKNOWN。

最终回执包含事务 ID、目标和代次、总长度、SHA、结果码和**完成范围**。图片的成功必须是 DISPLAYED（屏幕刷新并断电完成）；应用包暂存成功是 STAGED，不能被当成 DISPLAYED 或 ACTIVATED。

传输框架只管字节、授权、事务和调度。PNG 解码、六色校验、显示留在图片处理器里；ELF 校验和槽位管理留在应用包处理器里。

### 3.6 流控

消息大小、在途数据量、文件总大小是三个独立的限制。

- GATT 第一版窗口为 1（一问一答）。
- CoC/TCP 可以按接收预算批量发送，用累计 ACK 确认，块大小与 GATT 无关。
- credit 表示应用缓冲里真正可用的字节数，TCP 写进内核缓冲区不算应用已收到。
- 控制消息有独立的小预算，数据预算用完时仍然能 QUERY 和 CANCEL。
- 宿主每次只处理有限条消息就把控制权交还 App Manager，避免大流量饿死按键和管理操作。
- 稳态的 DATA/ACK 不读 Flash 应用槽、不重算目录、不逐包序列化状态 JSON。

### 3.7 断线与换通道

第一版不承诺“无感切换”，换通道是显式流程：

1. 停止在旧通道上发新的 DATA，在新通道上用原配对关系重新认证。
2. 用原 `transaction_id` 发 QUERY，核对 boot_id、应用代次、长度和 SHA。
3. 如果仍在 RECEIVING 且缓冲还在，领取新的写入者 epoch，从已确认的偏移继续。
4. 旧 epoch 排队的数据在执行前被 App Manager 拒绝。
5. COMMIT 已发出但回复丢失时，只允许 QUERY 原事务，不重新 BEGIN、不换 ID、不自动重刷。
6. 设备重启、事务丢失或回执被淘汰时，返回 OUTCOME_UNKNOWN，交给人处理。

事务的存活绑定到授权身份和应用生命周期，而不是某个 GATT 或 CoC 连接对象。链路断开后，接收中的事务只保留有限时间；应用卸载或配对撤销时立即失效。

## 4. 拟定接口

下面的名称只用来说明职责，正式签名在实现时再定。

### 客户端

| 接口组 | 拟定接口 | 约定 |
|---|---|---|
| 发现与能力 | `Discover`、`GetCapabilities`、`ListServices` | 返回设备身份、服务、协议版本、消息和队列限制；平台没有的能力不报 |
| 会话 | `Connect`、`Authenticate`、`Disconnect` | 用现有配对关系建立独立密钥和序号；连上不等于认证成功 |
| 请求响应 | `Call(service, method, request)` | 统一关联 ID、授权、错误、截止时间 |
| 传输 | `BeginTransfer`、`WriteData`、`CommitTransfer` | Begin 做资源准入；链路接受不能冒充应用确认；Commit 可能返回“处理中” |
| 确认与等待 | `QueryTransfer`、`WaitResult` | 结果带完成范围；图片只接受 DISPLAYED |
| 取消与恢复 | `CancelTransfer`、`ResumeTransfer` | 已进入显示的不能撤销；只恢复同一次开机、同一代次、同一授权下仍在接收的事务 |
| 换通道 | `Reconnect`、`AttachTransfer` | 新通道重新认证并领取写入 epoch |
| 事件 | `WatchTransfer`、`WatchState` | 统一的进度、完成、失败事件；不能推送的链路由客户端有限次查询 |
| 诊断 | `GetMetrics`、`GetLastError` | 分别统计发现、连接、认证、上传、执行耗时，不导出密钥 |

配对是独立的 `PairingService`，继续要求真人按键批准。Wi-Fi 建组、入网是连接管理器的平台操作。业务便利接口（`Image.Send`、`Apps.Install/Activate/Switch/Rollback`、`Device.Status`）都由上面的公共接口组合而成；`Image.Send` 只有拿到 DISPLAYED 回执才返回成功。

### 应用 SDK

现有 ABI 2.3 已经提供了 `app_host_channel_register_v23`、`info`、`recv`、`send`、`close` 和 `APP_EVENT_CHANNEL`（见 [应用接口参考](../05-app-api-reference.md)）。完整方案在此基础上扩展：

- `info` 额外报告已认证对端、配对和会话标识、可写预算；
- 注册时声明服务协议版本；
- 事务收尾交给公共事务层，应用只处理业务内容。

应用仍然在 App Manager 的协作式事件里运行，不保留宿主数据指针，不注册跨卸载的回调。

### 传输驱动

驱动只实现 `MessageLink` 和平台连接、释放。设备搜索、地址解析、Windows Wi-Fi 建组、ESP 入网由连接管理器编排。GATT UUID、ATT MTU、CoC PSM 和 credit、TCP 短读、WebSocket 分帧都不泄漏到图片或应用包服务。

## 5. 迁移步骤

| 现有代码 | 迁移方向 |
|---|---|
| `photopainter-host/main/ble_protocol.c` | 提取独立的会话/消息核心和命令路由，NimBLE 相关代码留在适配器 |
| `ai-quota-frame/internal/ble/protocol.go` | 提取通用协议客户端，原生和 Go/WASM 共用 |
| `ai-quota-frame/internal/ble/coc.go` | 复用公共封帧、密码和事务，只保留流 I/O |
| `ai-quota-frame/web/src/ble/session.ts`、`web/src/photoframe/sender.ts` | 迁到公共核心，JavaScript 只保留驱动和界面 |
| `photoframe/main/ble_image.c` | 去掉按传输方式分支的代码 |

现有 `APP_BLE_MAX_VALUE=128` 和 140 字节的 `app_ble_event_v2_t` 布局不能原地扩大，ABI 2.1 和 2.2 的接口原样保留。

建议顺序：

1. 把图片命令处理提成纯逻辑库，让现有适配器跑同一套测试。（已完成）
2. 提取公共会话/消息核心和 Go 客户端，建立跨语言的标准测试向量，先接模拟的 `MessageLink`。
3. 接入 GATT。新协议通过显式协商启用，能力不符就明确报错，不在出错后悄悄降级。
4. 加入 ABI 2.3 并更新 photoframe，让应用真正消费通用通道事件。（ABI 2.3 邮箱和 GATT 适配器已完成）
5. 加入 CoC、Wi-Fi TCP/WebSocket 适配器，复用同一套事务测试，最后做显式换通道恢复。

## 6. 验收设想

- 一组 C、Go、Go/WASM 共用的握手、加密、帧和事务标准向量。
- 同一业务用例在模拟的 GATT、CoC、TCP 适配器上得到一致的结果和最终回执。
- 故障注入：短读短写、重复或越界分片、队列满、认证重放、旧 epoch、COMMIT 回复丢失、应用切走、设备重启、回执被淘汰。
- 结果不确定时不触发自动重刷；缓存或断线状态要显示采集时间和 boot_id。
- 分别记录发现、连接、认证、上传、执行耗时，统计线上字节、DATA/ACK 数、排队时间、目录读取次数和内存峰值；稳态 DATA/ACK 期间 Flash 目录读取次数为零。
- 用同样的输入（约 60 KiB 的图片和一个应用包）做各通道基准。
- 先分别验收各平台的连续重连和传图，再验收换通道。

## 7. 未决问题

- **加密算法和握手细节。** AEAD 选哪种算法、HELLO/AUTH 的具体字节格式、能力摘要怎么算，都还没定。
- **各通道的消息上限。** GATT 512 B、CoC 4 KiB、TCP/WebSocket 16 KiB 只是起点，需要实测内存和吞吐后确定。
- **接收中事务的保留期限。** 链路断开后保留多久、占用的 PSRAM 怎么回收，需要和 60 秒接收空闲期限一起考虑。
- **Wi-Fi 的连接方式。** 相框入网（STA）、电脑建组（Windows GO/Legacy AP）还是普通路由器，各平台能力不同，属于连接策略，需要分别实测。
- **浏览器能力边界。** 浏览器没有 CoC，访问本地网络也受限制。纯网页还是配合本机组件，是客户端部署的选择。
- **Windows CoC。** Windows 原生客户端能否打开 CoC 需要实测。
- **和现有协议的共存期。** 旧客户端和 ABI 2.1/2.2 应用要继续工作多久，新旧协议在同一个连接上怎么协商。
