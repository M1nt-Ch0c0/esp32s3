# 应用接口参考

这一页列出应用和宿主之间的全部接口：ABI 版本、清单、入口与事件、每个宿主服务的参数和限制、时间预算、结果报告规则和版本兼容规则。写应用时把它当手册查。

第一次写应用请先看 [编写一个应用](04-writing-an-app.md)。应用怎样被加载、确认和回退，见 [设备是怎么工作的](03-how-the-device-works.md)。蓝牙线上协议（客户端那一侧怎么发）见 [蓝牙通信](06-bluetooth.md)。

接口以宿主仓库的 `sdk/photopainter_app.h` 和 `sdk/abi2_imports.inc` 为准，实现在 `main/photoframe_plugin.c`（事件派发）、`main/app_services.c`（服务）、`main/app_manager.c`（调度）、`main/app_ble.c`、`main/app_coc.c`、`main/app_channel.c`。

## 基本模型

- **ABI**（应用二进制接口）是已编译的应用和宿主之间的约定：能调用哪些符号、结构体怎么排列、事件怎么传、指针能用多久。
- 应用是一个 Xtensa ELF32 动态 ELF，和宿主共享地址空间。同一时间只有一个应用在运行（前台应用）。
- 宿主所有对应用的调用都在同一个任务（App Manager）里串行执行，一次一个事件。应用自己不能创建任务或回调。
- 每个事件调用一次应用的 `int main(int argc, char **argv)`。应用通过 `app_host_context_v2()` 取得事件描述，处理完调用 `app_complete()` 报告结果，然后返回。
- 静态变量在同一次加载内跨事件保留。每次加载（启动、切换、激活）都会重新从 ELF 初始化。
- 这套约束让应用行为可预期，但它不是安全沙箱：应用代码能读写宿主内存。

## ABI 版本

主版本固定为 2。次版本由应用通过清单宏声明，宿主据此决定允许哪些导入、发送哪些事件。

| 版本 | 声明方式 | 新增内容 |
|---|---|---|
| 2.0 | `APP_MANIFEST(...)` | 生命周期事件 START / INPUT / TIMER / NETWORK / STOP；显示、缓存、定时器、单调时间、墙上时间服务；网络状态（随上下文提供） |
| 2.1 | `APP_MANIFEST_BLE(...)` | 一个应用自定义 GATT 服务（最多 4 个特征，每个值最多 128 字节）；BLE 事件（6） |
| 2.2 | `APP_MANIFEST_COC(...)` | 前台应用独占一个 L2CAP CoC 通道，非阻塞收发记录；COC 事件（7） |
| 2.3 | `APP_MANIFEST_CHANNEL(...)` | 与传输方式无关的消息邮箱；CHANNEL 事件（8）；在 GATT 上可传最大约 4 KiB 的逻辑消息 |

每个版本都保留前面版本的结构布局和语义，只增加内容。宿主 SDK 本身是 2.3（`APP_ABI_MAJOR 2u`、`APP_ABI_MINOR 3u`）。

另有一个旧的 ABI 1：没有清单，由宿主把 PNG 交给应用自带的驱动画屏。它只为兼容保留，新应用不要使用。

## 清单

清单是 ELF 里唯一的只读段 `.app_manifest`，固定 128 字节：

```c
typedef struct {
    uint32_t magic, size, abi_major, abi_minor, inputs, flags;
    char id[32], name[64];
    uint32_t reserved[2];
} app_manifest_v2_t;
```

不要手写这个结构，用四个宏之一，每个 ELF 只能用一个：

```c
APP_MANIFEST        (ID, NAME, INPUTS, FLAGS);   /* 2.0 */
APP_MANIFEST_BLE    (ID, NAME, INPUTS, FLAGS);   /* 2.1 */
APP_MANIFEST_COC    (ID, NAME, INPUTS, FLAGS);   /* 2.2 */
APP_MANIFEST_CHANNEL(ID, NAME, INPUTS, FLAGS);   /* 2.3 */
```

| 字段 | 规则 |
|---|---|
| `magic` / `size` | `0x32505041` / 128，宏自动填写 |
| `abi_major` / `abi_minor` | 2 / 0–3，由宏决定 |
| `id` | 1–31 个字符，只能是小写字母、数字、`_`、`-`。它是应用在设备上的唯一标识，必须和包头里的 ID 一致 |
| `name` | 1–63 个可打印 ASCII 字符，显示在客户端里 |
| `inputs` | `0`，或 `APP_INPUT_PNG`（1）表示接收 INPUT 事件里的 PNG |
| `flags` | `0`，或 `APP_REQUIRES_DISPLAY`（1）表示 START 必须成功显示首屏才算启动成功 |
| `reserved` | 必须为 0 |

任何未知的 `inputs`、`flags` 位或更高的次版本都会被拒绝。

## 入口与上下文

### 上下文

```c
const app_context_v2_t *app_host_context_v2(void);
```

只在事件处理期间返回有效指针；在事件之外调用返回 `NULL`。结构在 ELF32 上 44 字节：

| 字段 | 含义 |
|---|---|
| `size` | 结构大小，44 |
| `abi_major` / `abi_minor` | 2 / 应用清单声明的次版本 |
| `event` | 事件编号，见下表 |
| `event_id` | 本次调用的编号，每次调用递增 |
| `generation` | 加载代次。每加载一次应用加 1，同一次加载内不变 |
| `input_type` | INPUT 事件为 `APP_INPUT_PNG`，其他事件为 0 |
| `data_size` / `data` | 本事件附带的数据及其长度；没有数据时为 0 / `NULL` |
| `timer_id` | TIMER 事件里是应用设置定时器时给的 ID |
| `network_connected` | Wi-Fi 当前是否已连接（1/0），每个事件都有 |

`data` 是**借用指针**：只在本次 `main()` 返回前有效。需要保留的内容要复制到应用自己的内存里。

### 结果

```c
typedef struct {
    uint32_t size, event_id, generation;
    int32_t status;
    uint32_t flags;
} app_result_v2_t;                                   /* 20 字节 */
void app_host_complete_v2(const app_result_v2_t *result);

static inline void app_complete(const app_context_v2_t *c, int32_t status, uint32_t flags);
```

用 `app_complete(c, status, flags)` 即可，它会填好 `size`、`event_id`、`generation`。

## 事件

| 编号 | 常量 | 最低版本 | 何时触发 | `data` |
|---:|---|---|---|---|
| 1 | `APP_EVENT_START` | 2.0 | 应用刚被加载：开机、激活新版本、切换到这个应用、回滚后重新加载 | 无 |
| 2 | `APP_EVENT_INPUT` | 2.0，且清单声明 `APP_INPUT_PNG` | 宿主自带的兼容图片通道收到一张完整 PNG，并且这个应用正在运行 | PNG 字节，最多 5 MiB；`input_type = APP_INPUT_PNG` |
| 3 | `APP_EVENT_TIMER` | 2.0 | 应用设置的定时器到期 | 无；看 `timer_id` |
| 4 | `APP_EVENT_NETWORK` | 2.0 | 宿主的 Wi-Fi 连上或断开（只在配置了 Wi-Fi 时发生） | 无；看 `network_connected` |
| 5 | `APP_EVENT_STOP` | 2.0 | 应用即将被卸载：切到别的应用、激活或回滚导致重新加载 | 无 |
| 6 | `APP_EVENT_BLE` | 2.1 | 应用注册的 GATT 服务有连接变化、写入或错误 | `app_ble_event_v2_t`，140 字节 |
| 7 | `APP_EVENT_COC` | 2.2 | 应用打开的 CoC 通道有连接变化、可读、可写或错误 | `app_coc_event_v22_t`，24 字节 |
| 8 | `APP_EVENT_CHANNEL` | 2.3 | 应用注册的消息通道有连接变化、收到消息或错误 | `app_channel_event_v23_t`，24 字节 |

补充说明：

- **功能键不产生事件。** 短按功能键时宿主给当前应用发 STOP，再加载下一个应用并发 START。
- 只有 START 成功并报告了 `APP_READY` 之后，宿主才会发送 TIMER、NETWORK、BLE、COC、CHANNEL 事件。
- 次版本不够的应用不会收到对应事件。
- 处理 BLE/COC/CHANNEL 事件时，先检查 `data` 非空且 `data_size` 等于对应结构的大小。
- 当前客户端发图使用应用自己的协议（见 [蓝牙通信](06-bluetooth.md)），INPUT 事件只在宿主的兼容图片通道里出现。不收 PNG 的应用在 INPUT 里返回 `APP_ERR_UNSUPPORTED` 即可。

## 结果报告规则

每个事件必须**恰好调用一次** `app_host_complete_v2()`（或 `app_complete()`）。`main()` 的返回值不会传给宿主。

结果被认为有效，需要同时满足：

- `size` 为 20，`event_id` 和 `generation` 与本次上下文一致；
- `status` 是 `APP_OK`（0）或 `APP_ERR_*`（−1 到 −9）之一；
- `flags` 只能是 0 或 `APP_READY`，并且 `APP_READY` 只能出现在 START 里。

漏调、重复调用或结果无效，本次事件都按 `APP_ERR_RUNTIME` 处理。

| 值 | 常量 | 一般含义 |
|---:|---|---|
| 0 | `APP_OK` | 成功 |
| −1 | `APP_ERR_ARGUMENT` | 参数错误 |
| −2 | `APP_ERR_INPUT` | 输入内容错误或此时不能接受输入 |
| −3 | `APP_ERR_DIMENSIONS` | 图片尺寸不对 |
| −4 | `APP_ERR_COLOR` | 有六色以外的像素 |
| −5 | `APP_ERR_MEMORY` | 内存不足 |
| −6 | `APP_ERR_DISPLAY` | 面板硬件操作失败 |
| −7 | `APP_ERR_TIMEOUT` | 面板忙等待超时 |
| −8 | `APP_ERR_RUNTIME` | 运行契约被破坏 |
| −9 | `APP_ERR_UNSUPPORTED` | 不支持这个操作或输入 |

CoC 和 CHANNEL 的 I/O 状态码（`APP_COC_AGAIN` 等）是正数，**不能**直接传给 `app_complete()`，否则结果无效。

### 宿主怎样处理结果

| 事件 | 成功的条件 | 失败时宿主怎么做 |
|---|---|---|
| START | `APP_OK` + `APP_READY`；声明了 `APP_REQUIRES_DISPLAY` 的还要本事件里显示成功 | 卸载应用。试运行中的新版本会被取消，回到原版本；开机时由启动保护决定是否回退 |
| INPUT | `APP_OK`，并且本事件里显示成功（成功但没显示也算 `APP_ERR_RUNTIME`） | 结果为 −6、−7、−8 时：试运行中则回滚；不在试运行且为 −8 则卸载应用 |
| TIMER / NETWORK / BLE / COC / CHANNEL | 任意 `APP_OK` 或 `APP_ERR_*` 都会记录下来 | 只有 `APP_ERR_RUNTIME` 会让宿主卸载应用，其他错误码只记录 |
| STOP | `APP_OK` | 应用照样被卸载，但宿主把这次卸载算作失败，后面的加载不会进行：试运行会被取消并回到原版本；普通切换会停在没有应用运行的状态 |

应用被卸载后设备处于“没有应用运行”的故障空闲状态（状态里的 `phase` 为 `fault`），仍可以通过蓝牙管理。

## 时间预算与调度

| 项目 | 限制 |
|---|---|
| 单个事件的逻辑处理 | 5 秒。从事件开始计时 |
| 一次显示调用 | 650 秒。调用期间宿主临时放宽；返回后重新给 5 秒 |
| 超时后果 | 宿主看门狗让设备重启（不会尝试卸载正在执行的代码）。如果发生在应用启动期间，下次开机会触发启动保护 |

调度规则：

- 宿主空闲时按 NETWORK → TIMER → BLE → COC → CHANNEL 的顺序挑一个待处理事件调用，然后再看下一轮。网络变化之后如果定时器也到期，会紧接着调用 TIMER。
- 部分事件在客户端请求里同步投递：客户端写应用的 GATT 特征时，宿主立即派发 BLE 写入事件；客户端通过 CHANNEL 发消息时，宿主立即派发 CHANNEL 事件并取回回复（见 [CHANNEL](#消息通道-channelabi-23)）。
- 宿主正在显示配对码、等待用户按键确认时，暂停所有应用事件，确认或超时后恢复。
- 上传、刷新、切换期间宿主处于忙状态，新的工作要等当前工作结束。

所以应用要把长任务拆开：一次事件只处理一块数据，剩下的等下一个事件。

## 宿主服务

所有服务只能在宿主调用应用的事件处理期间使用。下面每节列出函数、参数、限制和指针生命周期。

### 显示

```c
int32_t app_host_display_v2(const uint8_t *pixels, uint32_t bytes);
```

| 项目 | 规则 |
|---|---|
| 缓冲 | 必须正好 `APP_FRAME_BYTES`（800×480 = 384000）字节 |
| 排列 | 逻辑坐标，从左上角开始逐行，每字节一个像素 |
| 颜色 | `APP_BLACK` 0、`APP_WHITE` 1、`APP_YELLOW` 2、`APP_RED` 3、`APP_BLUE` 4、`APP_GREEN` 5 |
| 次数 | 每个事件最多一次；STOP 里不能显示 |
| 校验 | 宿主先检查全部像素，再碰硬件。旋转 180° 和面板编码由宿主完成 |
| 返回 | 刷新完成、面板断电等待结束后返回 `APP_OK`，大约 20 秒 |
| 指针 | 宿主只在调用期间读取 `pixels`，返回后应用可以释放 |

| 返回值 | 原因 |
|---|---|
| `APP_ERR_ARGUMENT` | `pixels` 为空或长度不对 |
| `APP_ERR_COLOR` | 有大于 5 的像素值 |
| `APP_ERR_RUNTIME` | 不在事件内、在 STOP 里、本事件已经显示过、或宿主正在执行配对重置 |
| `APP_ERR_MEMORY` / `APP_ERR_DISPLAY` / `APP_ERR_TIMEOUT` | 面板驱动内存不足 / 硬件操作失败 / 忙等待超时 |

### 缓存

```c
int32_t        app_host_cache_put_v2(uint32_t version, const uint8_t *data, uint32_t bytes);
const uint8_t *app_host_cache_get_v2(uint32_t version, uint32_t *bytes);
```

缓存是宿主内存里的一块可丢失的存储，按应用 ID 隔离。切走应用再切回来通常还在；设备重启、删除应用或被别的应用挤掉后消失。不要把它当持久存储。

| 项目 | 规则 |
|---|---|
| 容量 | 每个应用一份，最多 512 KiB；所有应用合计最多 1 MiB |
| 回收 | 空间不够时，先回收其他应用里最久没用的缓存；自己的不会被挤掉 |
| `version` | 应用自定义的内容格式号。`get` 时版本不一致，宿主丢弃这份缓存并返回 `NULL` |
| `put` | 复制 `data`，返回后应用可以释放原缓冲。`bytes = 0` 清除自己的缓存；超过 512 KiB 时清除自己的缓存并返回 `APP_ERR_MEMORY` |
| `get` | 返回宿主内部缓冲的指针，把长度写入 `*bytes`；没有缓存时返回 `NULL`、`*bytes = 0` |
| 借用 | `get` 返回的指针只读，不能 `free`，只在本事件内有效。同一个事件里调用过 `get` 后，`put` 返回 `APP_ERR_ARGUMENT` |

`put` 失败（`APP_ERR_MEMORY`）是正常情况，应用应当容忍，不要因此把已经成功的显示判为失败。

### 定时器

```c
int32_t app_host_timer_v2(uint32_t timer_id, uint32_t interval_ms);
```

| 项目 | 规则 |
|---|---|
| 数量 | 每个应用一个。再次调用会替换前一个设置 |
| 间隔 | 非零时至少 1000 ms，否则返回 `APP_ERR_ARGUMENT`。`interval_ms = 0` 取消 |
| 触发 | 到期后发 TIMER 事件，`timer_id` 原样带回；周期性重复，直到取消 |
| 合并 | 应用或宿主忙时错过的多次到期会合并成一次，下一次到期从这次触发时重新计算。需要真实经过的时间请用单调时间计算 |
| STOP | 可以取消，不能设非零间隔 |
| 清理 | 应用被卸载时宿主自动取消 |

### 时间

```c
uint64_t app_host_monotonic_ms_v2(void);
uint64_t app_host_wall_time_v2(void);
```

| 函数 | 返回 |
|---|---|
| `app_host_monotonic_ms_v2` | 本次开机以来的毫秒数，单调递增。不在事件内调用返回 0 |
| `app_host_wall_time_v2` | Unix 秒。系统时钟早于 2024-01-01 时（也就是没有设置过时间）返回 0 |

宿主没有授时协议，断电后墙上时间不保证准确。需要显示时间的应用，要处理返回 0 的情况。

### 网络状态

没有单独的函数。每个事件的上下文里都有 `network_connected`，Wi-Fi 状态变化时宿主发 NETWORK 事件。多次变化会合并，只交付最新状态。

宿主不给应用提供 socket、HTTP 客户端、Wi-Fi 凭据或文件系统。

### 应用 GATT 服务（ABI 2.1）

应用可以注册一个自定义 GATT 服务，让已配对的客户端读写。宿主复制服务描述和特征值，替应用回答读取请求；应用只在写入、连接变化和错误时收到事件。客户端怎样读写这些特征，见 [蓝牙通信](06-bluetooth.md)。

```c
int32_t app_host_ble_register_v2(const app_ble_service_v2_t *service);
int32_t app_host_ble_set_v2(uint32_t characteristic, const uint8_t *data, uint32_t bytes);
int32_t app_host_ble_unregister_v2(void);
```

```c
typedef struct {
    uint8_t uuid[16];
    uint32_t properties, max_bytes;
} app_ble_characteristic_v2_t;

typedef struct {
    uint32_t size, count;                 /* size = sizeof(app_ble_service_v2_t) = 120 */
    uint8_t uuid[16];
    app_ble_characteristic_v2_t characteristics[APP_BLE_MAX_CHARACTERISTICS];
} app_ble_service_v2_t;

typedef struct {
    uint32_t type, characteristic, data_size;
    uint8_t data[APP_BLE_MAX_VALUE];
} app_ble_event_v2_t;                     /* 140 字节 */
```

**注册规则**

| 项目 | 规则 |
|---|---|
| 次数 | 每次加载注册一次；注销后可以再注册。重复注册返回 `APP_ERR_RUNTIME` |
| 特征数 | `count` 为 1–4，按数组下标 0–3 引用 |
| UUID | 16 字节，按通常书写顺序排列（不是 NimBLE 的反序）。不能全为 0；不能以 `7b 9d` 开头（宿主保留）；不能是蓝牙 SIG 基础 UUID（`xxxxxxxx-0000-1000-8000-00805f9b34fb`）；特征 UUID 互不相同，也不能等于服务 UUID |
| `properties` | `APP_BLE_READ`(1)、`APP_BLE_WRITE`(2)、`APP_BLE_NOTIFY`(4) 的组合；至少要有 READ 或 WRITE；NOTIFY 必须同时有 READ |
| `max_bytes` | 1–128（`APP_BLE_MAX_VALUE`） |
| 发布 | 异步完成。如果无线层注册失败，应用会收到 `APP_BLE_ERROR` 事件 |
| 指针 | 宿主复制描述，调用返回后 `service` 可以释放；宿主不保留任何指向 ELF 的指针 |

**设置值**：`app_host_ble_set_v2(index, data, bytes)` 把值复制到宿主，`bytes` 不超过该特征的 `max_bytes`。之后客户端读到的就是这个值；带 NOTIFY 的特征会通知已订阅的客户端。

**事件**（`APP_EVENT_BLE`，`data` 指向 `app_ble_event_v2_t`）：

| `type` | 含义 |
|---|---|
| `APP_BLE_CONNECTED`(1) | 一个**已通过配对鉴权**的客户端连上了。注册时如果已有鉴权连接，也会收到一次 |
| `APP_BLE_DISCONNECTED`(2) | 鉴权连接断开 |
| `APP_BLE_WRITTEN`(3) | 客户端写入了 `characteristic` 号特征，内容在 `data[0..data_size)` |
| `APP_BLE_ERROR`(4) | 无线层注册失败 |

宿主为应用排队最多 8 个 BLE 事件，队列满时客户端的写入会收到“忙”。连接变化时队列里还没处理的写入会被丢弃。

**调用时机**：`register`、`set` 不能在 STOP 里调用；`unregister` 任何事件里都可以。应用被卸载时宿主自动注销服务、清空事件。

### CoC 通道（ABI 2.2）

L2CAP CoC（面向连接的信道）是蓝牙上一条比 GATT 快得多的数据通道。前台应用可以独占一个 CoC 通道，按“记录”收发数据。目前只有 macOS 原生客户端使用 CoC。线上格式和鉴权见 [蓝牙通信](06-bluetooth.md)。

```c
int32_t app_host_coc_open_v22(const app_coc_config_v22_t *config);
int32_t app_host_coc_info_v22(app_coc_info_v22_t *info);
int32_t app_host_coc_recv_v22(uint32_t session, void *dst, uint32_t capacity, uint32_t *received);
int32_t app_host_coc_send_v22(uint32_t session, const void *src, uint32_t bytes);
int32_t app_host_coc_close_v22(void);
```

```c
typedef struct { uint32_t size, rx_sdu_bytes, rx_slots, tx_slots; } app_coc_config_v22_t;          /* 16 字节 */
typedef struct { uint32_t size, state, psm, session, rx_payload_max, tx_payload_max; } app_coc_info_v22_t; /* 24 字节 */
typedef struct { uint32_t size, type, session, rx_pending, tx_free; int32_t status; } app_coc_event_v22_t; /* 24 字节 */
```

**打开**

| 参数 | 范围 | 说明 |
|---|---|---|
| `size` | `sizeof(app_coc_config_v22_t)` | |
| `rx_sdu_bytes` | 128–4096 | 每条接收记录的缓冲大小。应用可收的最大载荷是它减 38 字节（鉴权封套开销） |
| `rx_slots` / `tx_slots` | 1–8 | 接收 / 发送队列的记录数 |

宿主为队列分配 `rx_slots × rx_sdu_bytes + tx_slots × 4096` 字节。相框应用用的是 4096 / 4 / 4，约 32 KiB。已经打开时再打开返回 `APP_COC_BUSY`。

**查询**：`info.state` 为 0 未打开、1 等待连接、2 已连接；`psm` 固定为 `0x81`；`session` 是当前连接编号（每次新连接都不同，0 表示没有连接）；`rx_payload_max` / `tx_payload_max` 是双方能接受的最大记录载荷。调用前把 `info.size` 设为结构大小。

**收发**

- `recv` 一次取出一条完整记录。缓冲不够时返回 `APP_COC_TOO_SMALL`，把需要的长度写进 `*received`，记录保留在队列里。
- `send` 把记录复制进发送队列，成功后源缓冲可以复用。`APP_COC_OK` 只代表已入队，不代表对端已收到或处理完。
- 两者都不阻塞。`session` 必须是当前连接的编号，旧编号返回 `APP_COC_STALE`。

| I/O 状态 | 值 | 含义 |
|---|---:|---|
| `APP_COC_OK` | 0 | 成功 |
| `APP_COC_AGAIN` | 1 | 接收队列空 / 发送队列满，稍后再试 |
| `APP_COC_CLOSED` | 2 | 当前没有连接 |
| `APP_COC_STALE` | 3 | `session` 不是当前连接 |
| `APP_COC_TOO_SMALL` | 4 | 接收缓冲太小 |
| `APP_COC_BUSY` | 5 | 已经打开 |

参数或调用时机错误时返回负数的 `APP_ERR_*`（`APP_ERR_ARGUMENT`、`APP_ERR_RUNTIME`、`APP_ERR_MEMORY`）。

**事件**（`APP_EVENT_COC`）：`type` 是位集合，可能同时有多位：

| 位 | 含义 |
|---|---|
| `APP_COC_CONNECTED`(1) | 新连接建立。应用应当丢掉上一个连接遗留的接收状态 |
| `APP_COC_DISCONNECTED`(2) | 连接断开，队列已清空 |
| `APP_COC_RX_READY`(4) | 有记录可读，`rx_pending` 是条数 |
| `APP_COC_TX_READY`(8) | 发送队列有空位，`tx_free` 是空位数 |
| `APP_COC_ERROR`(16) | 连接因错误断开，`status` 是错误码 |

就绪事件会合并：一次事件可能对应多条记录。每次事件只处理有限几条（相框应用每次最多四条）然后返回，剩下的等下一次事件。

**生命周期**：断开连接保留打开状态，下一个客户端可以直接连；`close`、切走应用或卸载会释放队列。STOP 里只能调用 `info` 和 `close`。

### 消息通道 CHANNEL（ABI 2.3）

CHANNEL 是一个与传输方式无关的消息邮箱：应用注册一个逻辑服务 ID，之后按“一条请求、一条回复”的方式和一个已鉴权的对端交换完整消息。应用看不到 ATT MTU、GATT 特征、CoC PSM 之类的细节。目前宿主通过 GATT 承载它。客户端一侧的命令和分片方式见 [蓝牙通信](06-bluetooth.md)。

```c
int32_t app_host_channel_register_v23(const app_channel_config_v23_t *config);
int32_t app_host_channel_info_v23(app_channel_info_v23_t *info);
int32_t app_host_channel_recv_v23(uint32_t session, void *dst, uint32_t capacity, uint32_t *received);
int32_t app_host_channel_send_v23(uint32_t session, const void *src, uint32_t bytes);
int32_t app_host_channel_close_v23(void);
```

```c
typedef struct {
    uint32_t size;
    uint8_t service[16];
    uint32_t max_message_bytes, max_reply_bytes;
} app_channel_config_v23_t;                                                        /* 28 字节 */
typedef struct { uint32_t size, state, session, rx_message_max, tx_message_max; } app_channel_info_v23_t;  /* 20 字节 */
typedef struct { uint32_t size, type, session, rx_pending, tx_free; int32_t status; } app_channel_event_v23_t; /* 24 字节 */
```

**注册**

| 参数 | 范围 | 说明 |
|---|---|---|
| `service` | 16 字节，不能全为 0 | 应用自定义的逻辑服务 ID，客户端据此确认对面是哪个应用 |
| `max_message_bytes` | 1–4096（`APP_CHANNEL_MAX_MESSAGE`） | 应用能接收的最大消息 |
| `max_reply_bytes` | 1–256（`APP_CHANNEL_MAX_REPLY`） | 应用回复的最大长度 |

宿主在注册时分配两块缓冲。已注册时再注册返回 `APP_CHANNEL_BUSY`。经由 GATT 时，客户端实际可发的消息上限是 `min(max_message_bytes, 4047)`，因为 4096 字节的逻辑请求里还要放协议头和鉴权字段。

**查询**：`info.state` 为 0 未注册、1 等待对端、2 已连接；`session` 是当前会话编号（不重复使用）；`rx_message_max` / `tx_message_max` 是注册时的两个上限。

**收发规则**

- 同一时间只有一个对端、一条在途请求。上一条消息没取走或上一条回复没送出时，新消息会被拒绝，客户端收到“忙”。
- `recv` 取出整条消息；缓冲不够时返回 `APP_CHANNEL_TOO_SMALL`，需要的长度写进 `*received`，消息保留。
- `send` 复制一条回复，长度不超过 `max_reply_bytes`；上一条回复还没送出时返回 `APP_CHANNEL_AGAIN`。成功只代表已复制，不代表对端已收到。
- **回复要在同一个 CHANNEL 事件里发。** 客户端发来一条消息时，宿主立即派发带 `APP_CHANNEL_RX_READY` 的 CHANNEL 事件，事件返回后马上取回复交给客户端。应用在这个事件里没有调用 `send`，客户端这次请求就会失败。

I/O 状态与 CoC 同名同值：`APP_CHANNEL_OK` 0、`AGAIN` 1、`CLOSED` 2（没有会话或对端失去授权）、`STALE` 3（`session` 过期或通道未注册）、`TOO_SMALL` 4、`BUSY` 5。参数或调用时机错误返回负数 `APP_ERR_*`。

**事件**（`APP_EVENT_CHANNEL`）：`type` 是位集合：

| 位 | 含义 |
|---|---|
| `APP_CHANNEL_CONNECTED`(1) | 对端打开了会话 |
| `APP_CHANNEL_DISCONNECTED`(2) | 会话关闭，或对端失去授权（例如配对被重置） |
| `APP_CHANNEL_RX_READY`(4) | 有一条消息待读，`rx_pending` 为 1 |
| `APP_CHANNEL_TX_READY`(8) | 上一条回复已送出，`tx_free` 为 1 |
| `APP_CHANNEL_ERROR`(16) | 错误 |

**生命周期**：注册跟随本次加载。`close`、切走或卸载会释放缓冲，旧的会话编号随之失效。STOP 里只能调用 `info` 和 `close`。

### 允许导入的函数

应用 ELF 里所有未定义的符号都必须在 `sdk/abi2_imports.inc` 里。打包工具和宿主用同一份清单检查，不在清单里的包会被拒绝。

| 类别 | 函数 |
|---|---|
| 宿主服务 | 上面各节列出的 `app_host_*` 函数。`app_host_ble_*` 要求次版本 ≥ 1，`app_host_coc_*` ≥ 2，`app_host_channel_*` ≥ 3 |
| 内存 | `malloc` `calloc` `realloc` `free` |
| 内存与字符串 | `memcpy` `memmove` `memset` `memcmp` `strlen` `strcmp` `strncmp` `strncpy` `strchr` `strrchr` `strcpy` |
| 其他 C 库 | `setjmp` `longjmp` `abort` `floor` `pow` `fprintf` `fputc` `fputs` `fread` `stderr` |
| 编译器辅助 | `__bswapsi2` 以及一组双精度浮点辅助函数（`__adddf3` `__subdf3` `__muldf3` `__divdf3` `__floatsidf` `__floatunsidf` `__fixdfsi` `__fixunsdfsi` `__ledf2` `__gedf2` `__extendsfdf2` `__truncdfsf2`） |

清单里没有 `printf`、`snprintf`、`strstr`、`memchr` 等函数，需要时自己实现。没有任务、GPIO、SPI、I²C 或系统回调接口。

`malloc` 的大块分配（超过 512 字节）会优先使用 PSRAM。应用没有单独的内存配额，每次分配都要检查失败。

## 借用指针与资源生命周期

| 对象 | 有效期 | 谁释放 |
|---|---|---|
| `app_host_context_v2()` 返回的上下文 | 本次 `main()` 返回前 | 宿主 |
| 上下文里的 `data`（PNG、BLE/COC/CHANNEL 事件） | 本次 `main()` 返回前 | 宿主 |
| `app_host_cache_get_v2()` 返回的指针 | 本次事件内，只读 | 宿主 |
| 传给 display、cache_put、ble_register、ble_set、coc_send、channel_send 的缓冲 | 宿主只在调用期间读取或复制 | 应用，调用返回后即可释放 |
| 应用 `malloc` 的内存 | 直到应用 `free` | 应用，在 STOP 里释放 |
| 定时器、GATT 服务、CoC 通道、CHANNEL 注册及其排队事件 | 本次加载 | 宿主在卸载时自动取消 |
| 缓存内容 | 跨加载保留，直到重启、被回收、删除应用或应用自己清除 | 宿主 |

宿主从不保存指向应用 ELF 的指针或回调，应用卸载后宿主不会再碰它的代码和数据。

## 包格式

`tools/module.py package` 生成的 `.pkg` 是：4096 字节包头 + ELF。

| 字段 | 内容 |
|---|---|
| `magic` | `0x31464C45` |
| `format` | 2（带应用 ID） |
| `abi` | 2（有清单时）；1（旧应用） |
| `length` | ELF 字节数 |
| `version` | 发布号，`--version` 指定 |
| `sha256` | ELF 的 SHA-256 |
| 应用 ID | 32 字节，NUL 补齐，必须等于清单里的 `id` |

整包不超过 1 MiB，也就是一个应用槽的大小。宿主收到包时会重新检查包头、SHA-256、ELF 结构、清单和导入清单。

## 版本兼容规则

1. **宿主向下兼容。** 支持 2.3 的宿主能运行声明 2.0、2.1、2.2、2.3 的应用，旧应用不用重新编译。
2. **旧宿主拒绝新应用。** 宿主只接受次版本不高于自己的应用；更高的会在上传时被拒绝，不会被加载执行。所以要先升级宿主固件，再安装需要新版本的应用。
3. **导入和声明必须匹配。** 用了 `app_host_ble_*` 必须声明至少 2.1，`app_host_coc_*` 至少 2.2，`app_host_channel_*` 至少 2.3。打包工具和宿主都会检查。声明更高的次版本但不用新服务是允许的。
4. **事件按声明投递。** 应用只会收到它声明的次版本所包含的事件。
5. **布局冻结。** 已发布的结构体（上下文 44 字节、结果 20 字节、清单 128 字节、BLE 服务 120 字节、BLE 事件 140 字节、CoC 结构各 16/24/24 字节、CHANNEL 结构 28/20/24 字节）不会改变。头文件里的 `_Static_assert` 会在编译时检查这些大小。新能力只通过新的次版本、新函数和新事件加入。
6. **SDK 两份保持一致。** 宿主 `sdk/` 是原件，`photoframe/components/app_sdk/` 是副本，内容必须完全相同。
7. **不要绕过检查。** 手改包头、自己定义“看起来一样”的结构体，都会让宿主和应用的约定失效。
