# 应用接口参考

这一页列出应用和宿主之间的全部接口，写应用时当手册查。内容包括 ABI 版本、清单、事件、结果报告规则、时间预算、每个宿主服务的参数和限制，以及版本兼容规则。

第一次写应用请先看[编写一个应用](04-writing-an-app.md)。应用怎样被加载、确认和回退，见 [设备是怎么工作的](03-how-the-device-works.md)。蓝牙线上协议（客户端那一侧怎么发）见 [蓝牙通信](06-bluetooth.md)。

接口声明在宿主仓库的 `sdk/photopainter_app.h`，允许调用的函数列在 `sdk/abi2_imports.inc`。宿主一侧的实现分布在这些文件里：

| 文件 | 内容 |
|---|---|
| `main/photoframe_plugin.c` | 加载应用、派发事件 |
| `main/app_services.c` | 显示、缓存、定时器、时间服务 |
| `main/app_manager.c` | 应用管理任务和调度 |
| `main/app_ble.c`、`main/app_coc.c`、`main/app_channel.c` | 三种蓝牙通信服务 |

## 基本模型

- **ABI**（应用二进制接口）是已编译的应用和宿主之间的约定：能调用哪些符号、结构体怎么排列、事件怎么传、指针能用多久。
- 应用是 Xtensa 架构的 32 位动态 ELF，和宿主共用同一个地址空间。同一时间只有一个应用在运行，叫前台应用。
- 宿主对应用的所有调用都在同一个任务里依次执行，这个任务叫应用管理任务（App Manager），一次只处理一个事件。应用代码只在这些调用里运行。
- 每个事件调用一次应用的 `int main(int argc, char **argv)`。应用通过 `app_host_context_v2()` 取得事件描述，处理完调用 `app_complete()` 报告结果，然后返回。
- 静态变量在同一次加载内跨事件保留。每次加载（启动、切换、激活）都会重新从 ELF 初始化。
- 这些规则让应用的行为可以预期。应用代码能读写宿主内存，所以只安装你信任的应用。

## ABI 版本

主版本固定为 2。次版本由应用通过清单宏声明，宿主据此决定允许哪些导入、发送哪些事件。

| 版本 | 声明方式 | 新增内容 |
|---|---|---|
| 2.0 | `APP_MANIFEST(...)` | START、INPUT、TIMER、NETWORK、STOP 五种事件；显示、缓存、定时器和时间服务；每个事件的上下文里带网络状态 |
| 2.1 | `APP_MANIFEST_BLE(...)` | 应用可以注册一个自定义 GATT 服务，最多 4 个特征，每个值最多 128 字节；新增 BLE 事件 |
| 2.2 | `APP_MANIFEST_COC(...)` | 前台应用可以独占一条 L2CAP CoC 通道收发数据；新增 COC 事件 |
| 2.3 | `APP_MANIFEST_CHANNEL(...)` | 通用消息通道，应用不用关心底层是哪种蓝牙传输；在 GATT 上一条消息最大约 4 KiB；新增 CHANNEL 事件 |

每个新版本都完整保留旧版本的结构布局和含义，只增加新内容。宿主 SDK 本身是 2.3（`APP_ABI_MAJOR 2u`、`APP_ABI_MINOR 3u`）。

宿主还兼容一种旧的 ABI 1 应用：没有清单，由应用自带的驱动画屏。新应用请使用 ABI 2。

## 清单

清单是 ELF 里一个名为 `.app_manifest` 的只读段，固定 128 字节，每个 ELF 只能有一个：

```c
typedef struct {
    uint32_t magic, size, abi_major, abi_minor, inputs, flags;
    char id[32], name[64];
    uint32_t reserved[2];
} app_manifest_v2_t;
```

用下面四个宏之一声明清单，宏会填好所有固定字段：

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

`inputs` 和 `flags` 只能使用上表列出的位，次版本不能高于宿主支持的版本，否则安装时会被拒绝。

## 入口与上下文

### 上下文

```c
const app_context_v2_t *app_host_context_v2(void);
```

在事件处理期间调用，返回本次事件的描述；在事件之外调用返回 `NULL`。结构体大小 44 字节：

| 字段 | 含义 |
|---|---|
| `size` | 结构大小，44 |
| `abi_major` / `abi_minor` | 2 / 应用清单声明的次版本 |
| `event` | 事件编号，见下表 |
| `event_id` | 本次调用的编号，每次调用递增 |
| `generation` | 加载次数。每加载一次应用加 1，同一次加载内不变 |
| `input_type` | INPUT 事件为 `APP_INPUT_PNG`，其他事件为 0 |
| `data_size` / `data` | 本事件附带的数据及其长度；没有数据时为 0 / `NULL` |
| `timer_id` | TIMER 事件里是应用设置定时器时给的 ID |
| `network_connected` | Wi-Fi 当前是否已连接（1/0），每个事件都有 |

`data` 是**借用指针**：内存属于宿主，只在本次 `main()` 返回前有效。需要保留的内容要复制到应用自己的内存里。

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

- 功能键由宿主处理。短按功能键时，宿主给当前应用发 STOP，再加载下一个应用并发 START。
- START 成功并报告了 `APP_READY` 之后，宿主才开始发送 TIMER、NETWORK、BLE、COC、CHANNEL 事件。
- 应用只收到它声明的次版本所包含的事件。
- 处理 BLE、COC、CHANNEL 事件时，先检查 `data` 非空，并且 `data_size` 等于对应结构体的大小。
- 现有客户端用应用自己的蓝牙协议发图（见[图片事务](06-bluetooth.md#7-图片事务)），INPUT 事件只来自宿主保留的兼容图片通道。不收 PNG 的应用在 INPUT 里返回 `APP_ERR_UNSUPPORTED` 即可。

## 结果报告规则

每个事件必须**恰好调用一次** `app_host_complete_v2()`（或 `app_complete()`）。`main()` 的返回值不会传给宿主。

有效的结果要同时满足：

- `size` 为 20，`event_id` 和 `generation` 与本次上下文一致；
- `status` 是 `APP_OK`（0）或 `APP_ERR_*`（−1 到 −9）之一；
- `flags` 只能是 0 或 `APP_READY`，并且 `APP_READY` 只能出现在 START 里。

漏调、重复调用或结果无效时，宿主把本次事件当成 `APP_ERR_RUNTIME`。

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
| −8 | `APP_ERR_RUNTIME` | 应用违反了本页的调用规则 |
| −9 | `APP_ERR_UNSUPPORTED` | 不支持这个操作或输入 |

CoC 和 CHANNEL 收发函数返回的状态码（`APP_COC_AGAIN` 等）是正数。传给 `app_complete()` 的状态只能是上表里的值，所以要先把这些状态码换成 `APP_OK` 或某个 `APP_ERR_*`。

### 宿主怎样处理结果

| 事件 | 成功的条件 | 失败时宿主怎么做 |
|---|---|---|
| START | 报告 `APP_OK` 并带 `APP_READY`；声明了 `APP_REQUIRES_DISPLAY` 的，还要在本事件里显示成功 | 卸载应用。如果是试运行，回到原版本；如果是开机，由启动保护决定是否回退 |
| INPUT | 报告 `APP_OK`，并且本事件里显示成功。报告成功却没显示，也按 `APP_ERR_RUNTIME` 处理 | 结果为 −6、−7、−8 时：试运行中则回滚；不在试运行时，只有 −8 会让宿主卸载应用 |
| TIMER、NETWORK、BLE、COC、CHANNEL | 任何合法的状态码都会被记录 | `APP_ERR_RUNTIME` 会让宿主卸载应用，其他错误码只记录 |
| STOP | `APP_OK` | 应用照样被卸载，但这次切换算作失败。试运行会被取消并回到原版本；普通切换停在没有应用运行的状态 |

应用被卸载后，相框进入故障空闲状态：没有应用在运行，状态里的 `phase` 为 `fault`，但仍可以通过蓝牙管理。

## 时间预算与调度

| 项目 | 限制 |
|---|---|
| 单个事件 | 5 秒，从事件开始计时 |
| 一次显示调用 | 650 秒。显示返回后，重新给 5 秒 |
| 超时后果 | 宿主的看门狗重启整台设备。如果发生在应用启动期间，下次开机会触发启动保护 |

调度规则：

- 宿主空闲时，按 NETWORK、TIMER、BLE、COC、CHANNEL 的顺序挑一个待处理事件调用应用，然后进入下一轮。网络变化之后如果定时器也到期，会紧接着调用 TIMER。完整的优先顺序见[三条硬规则](03-how-the-device-works.md#三条硬规则)。
- 有两种事件在客户端请求里立即派发。客户端写应用的 GATT 特征时，宿主立即派发 BLE 写入事件。客户端通过 CHANNEL 发消息时，宿主立即派发 CHANNEL 事件，并在事件返回后取回复（见 [CHANNEL](#消息通道-channelabi-23)）。
- 宿主显示配对码、等待用户按键确认期间，暂停所有应用事件，确认或超时后恢复。
- 上传、刷新、切换期间宿主处于忙状态，新的工作要等当前工作结束。

所以应用要把长任务拆开：一次事件只处理一块数据，剩下的留给下一个事件。

## 宿主服务

所有服务都只在事件处理期间可用。下面每节列出函数、参数、限制，以及指针的有效期。

### 显示

```c
int32_t app_host_display_v2(const uint8_t *pixels, uint32_t bytes);
```

| 项目 | 规则 |
|---|---|
| 缓冲 | 必须正好 `APP_FRAME_BYTES`（800×480 = 384000）字节 |
| 排列 | 逻辑坐标，从左上角开始逐行，每字节一个像素 |
| 颜色 | `APP_BLACK` 0、`APP_WHITE` 1、`APP_YELLOW` 2、`APP_RED` 3、`APP_BLUE` 4、`APP_GREEN` 5 |
| 次数 | 每个事件最多一次，STOP 以外的事件里才能显示 |
| 校验 | 宿主先检查全部像素，再操作屏幕。旋转 180° 和面板编码由宿主完成 |
| 返回 | 刷新完成、面板断电后返回 `APP_OK`，大约 20 秒 |
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

缓存是宿主内存里的一块临时存储，每个应用的缓存互相隔开。切走应用再切回来，缓存通常还在；设备重启、删除应用，或者空间被别的应用回收后，缓存就没了。需要长期保存的数据，请让客户端重新发送。

| 项目 | 规则 |
|---|---|
| 容量 | 每个应用一份，最多 512 KiB；所有应用合计最多 1 MiB |
| 回收 | 空间不够时，宿主回收其他应用里最久没用的缓存，正在写入的应用自己的缓存保留 |
| `version` | 应用自定义的内容格式号。`get` 时版本不一致，宿主丢弃这份缓存并返回 `NULL` |
| `put` | 复制 `data`，返回后应用可以释放原缓冲。`bytes = 0` 清除自己的缓存；超过 512 KiB 时清除自己的缓存并返回 `APP_ERR_MEMORY` |
| `get` | 返回宿主内部缓冲的指针，把长度写入 `*bytes`；没有缓存时返回 `NULL`、`*bytes = 0` |
| 借用 | `get` 返回的指针只读，内存由宿主管理，只在本事件内有效。同一个事件里调用过 `get` 后，`put` 返回 `APP_ERR_ARGUMENT` |

`put` 返回 `APP_ERR_MEMORY` 是正常情况。应用照常报告本次事件的结果即可，比如显示已经成功，就报告 `APP_OK`。

### 定时器

```c
int32_t app_host_timer_v2(uint32_t timer_id, uint32_t interval_ms);
```

| 项目 | 规则 |
|---|---|
| 数量 | 每个应用一个。再次调用会替换前一个设置 |
| 间隔 | 非零时至少 1000 ms，否则返回 `APP_ERR_ARGUMENT`。`interval_ms = 0` 取消 |
| 触发 | 到期后发 TIMER 事件，`timer_id` 原样带回；周期性重复，直到取消 |
| 合并 | 应用或宿主忙时错过的多次到期，会合并成一次 TIMER 事件，下一次到期从这次触发时重新计算。需要知道实际过了多久，请用单调时间计算 |
| STOP | 只能传 0 取消定时器 |
| 清理 | 应用被卸载时宿主自动取消 |

### 时间

```c
uint64_t app_host_monotonic_ms_v2(void);
uint64_t app_host_wall_time_v2(void);
```

| 函数 | 返回 |
|---|---|
| `app_host_monotonic_ms_v2` | 本次开机以来的毫秒数，单调递增。不在事件内调用返回 0 |
| `app_host_wall_time_v2` | 当前时间，Unix 秒数。系统时钟早于 2024-01-01 时返回 0，表示时间还没设置过 |

宿主自己不对时，断电后时间可能不准。需要显示时间的应用，要处理返回 0 的情况。

### 网络状态

没有单独的函数。每个事件的上下文里都有 `network_connected`，Wi-Fi 状态变化时宿主发 NETWORK 事件。多次变化会合并，只交付最新状态。

宿主给应用的网络能力只有这个连接状态。

### 应用 GATT 服务（ABI 2.1）

应用可以注册一个自定义 GATT 服务，让已配对的客户端读写。宿主保存服务描述和特征值的副本，客户端读取时由宿主直接回答。应用只在客户端写入、连接变化和出错时收到事件。客户端怎样读写这些特征，见[应用通信的三种方式](06-bluetooth.md#8-应用通信的三种方式)。

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
| UUID | 16 字节，按通常的书写顺序排列（和 NimBLE 内部的字节顺序相反）。服务和各个特征的 UUID 要互不相同、不全为 0。以 `7b 9d` 开头的 UUID 留给宿主，蓝牙标准组织的基础 UUID（`xxxxxxxx-0000-1000-8000-00805f9b34fb`）也不能用 |
| `properties` | `APP_BLE_READ`(1)、`APP_BLE_WRITE`(2)、`APP_BLE_NOTIFY`(4) 的组合。至少要有 READ 或 WRITE；带 NOTIFY 时必须同时带 READ |
| `max_bytes` | 1–128（`APP_BLE_MAX_VALUE`） |
| 发布 | 异步完成。如果无线层注册失败，应用会收到 `APP_BLE_ERROR` 事件 |
| 指针 | 宿主复制描述，调用返回后 `service` 可以释放 |

**设置值**：`app_host_ble_set_v2(index, data, bytes)` 把值复制到宿主，`bytes` 不超过该特征的 `max_bytes`。之后客户端读到的就是这个值；带 NOTIFY 的特征会通知已订阅的客户端。

**事件**（`APP_EVENT_BLE`，`data` 指向 `app_ble_event_v2_t`）：

| `type` | 含义 |
|---|---|
| `APP_BLE_CONNECTED`(1) | 一个**已通过配对鉴权**的客户端连上了。注册时如果已有鉴权连接，也会收到一次 |
| `APP_BLE_DISCONNECTED`(2) | 鉴权连接断开 |
| `APP_BLE_WRITTEN`(3) | 客户端写入了 `characteristic` 号特征，内容在 `data[0..data_size)` |
| `APP_BLE_ERROR`(4) | 无线层注册失败 |

宿主最多为应用排队 8 个 BLE 事件，队列满时客户端的写入会收到“忙”。连接变化时，队列里还没处理的写入会被丢弃。

**调用时机**：`register` 和 `set` 在 STOP 以外的事件里调用，`unregister` 在任何事件里都可以调用。应用被卸载时，宿主自动注销服务、清空事件。

### CoC 通道（ABI 2.2）

L2CAP CoC（面向连接的信道）是蓝牙上一条比 GATT 快得多的数据通道。前台应用可以独占一条 CoC 通道，以“记录”为单位收发数据，一条记录就是一段完整的数据。目前只有 macOS 原生客户端使用 CoC。空中的数据格式和鉴权方式见[蓝牙通信](06-bluetooth.md#三种承载方式)。

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
| `rx_sdu_bytes` | 128–4096 | 每条接收记录的缓冲大小。宿主要在每条记录外加 38 字节的鉴权信息，所以应用能收的最大数据量是这个值减 38 |
| `rx_slots` / `tx_slots` | 1–8 | 接收 / 发送队列的记录数 |

宿主为队列分配 `rx_slots × rx_sdu_bytes + tx_slots × 4096` 字节。相框应用用的参数是 4096、4、4，约 32 KiB。通道已经打开时再调用 `open`，返回 `APP_COC_BUSY`。

**查询**：调用前把 `info.size` 设为结构体大小。返回的字段含义如下：

| 字段 | 含义 |
|---|---|
| `state` | 0 未打开，1 等待连接，2 已连接 |
| `psm` | 通道号，固定为 `0x81` |
| `session` | 当前连接的编号，每次新连接都不同，0 表示没有连接 |
| `rx_payload_max` / `tx_payload_max` | 接收和发送方向上，一条记录最多能带多少字节数据 |

**收发**

- `recv` 一次取出一条完整记录。缓冲不够时返回 `APP_COC_TOO_SMALL`，把需要的长度写进 `*received`，记录保留在队列里。
- `send` 把记录复制进发送队列，成功后源缓冲可以复用。`APP_COC_OK` 表示记录已经进入发送队列；客户端是否处理完，要靠应用自己的协议确认。
- 两者都立即返回。`session` 必须是当前连接的编号，旧编号返回 `APP_COC_STALE`。

| I/O 状态 | 值 | 含义 |
|---|---:|---|
| `APP_COC_OK` | 0 | 成功 |
| `APP_COC_AGAIN` | 1 | 接收队列空 / 发送队列满，稍后再试 |
| `APP_COC_CLOSED` | 2 | 当前没有连接 |
| `APP_COC_STALE` | 3 | `session` 不是当前连接 |
| `APP_COC_TOO_SMALL` | 4 | 接收缓冲太小 |
| `APP_COC_BUSY` | 5 | 已经打开 |

参数错误、调用时机错误或内存不足时，返回负数的 `APP_ERR_ARGUMENT`、`APP_ERR_RUNTIME` 或 `APP_ERR_MEMORY`。

**事件**（`APP_EVENT_COC`）：`type` 的每一位代表一种情况，可能同时有多位：

| 位 | 含义 |
|---|---|
| `APP_COC_CONNECTED`(1) | 新连接建立。应用应当丢掉上一个连接遗留的接收状态 |
| `APP_COC_DISCONNECTED`(2) | 连接断开，队列已清空 |
| `APP_COC_RX_READY`(4) | 有记录可读，`rx_pending` 是条数 |
| `APP_COC_TX_READY`(8) | 发送队列有空位，`tx_free` 是空位数 |
| `APP_COC_ERROR`(16) | 连接因错误断开，`status` 是错误码 |

多条记录到达时，宿主可能只发一次事件。应用每次事件只处理有限几条（相框应用每次最多四条），然后返回，剩下的留给下一次事件。

**生命周期**：连接断开后，通道仍然保持打开，下一个客户端可以直接连上。调用 `close`、切走应用或卸载应用时，宿主释放队列。STOP 里只能调用 `info` 和 `close`。

### 消息通道 CHANNEL（ABI 2.3）

CHANNEL 是一个通用的消息通道。应用注册一个服务 ID，之后和一个已鉴权的客户端按“一条请求、一条回复”的方式交换完整消息。底层用哪种蓝牙传输、每次能写多少字节，都由宿主处理，应用不用关心。目前宿主通过 GATT 传输它。客户端一侧的命令见[通用消息命令](06-bluetooth.md#通用消息命令op-2427)。

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
| `service` | 16 字节，不全为 0 | 应用自定义的服务 ID，客户端靠它确认对面是哪个应用 |
| `max_message_bytes` | 1–4096（`APP_CHANNEL_MAX_MESSAGE`） | 应用能接收的最大消息 |
| `max_reply_bytes` | 1–256（`APP_CHANNEL_MAX_REPLY`） | 应用回复的最大长度 |

宿主在注册时分配收发两块缓冲。已经注册时再注册，返回 `APP_CHANNEL_BUSY`。通过 GATT 传输时，客户端一条消息最多 `min(max_message_bytes, 4047)` 字节，因为 4096 字节的请求里还要放协议头和签名。

**查询**：`info.state` 为 0 未注册、1 等待客户端、2 已连接。`session` 是当前会话的编号，每个会话的编号都不同。`rx_message_max` 和 `tx_message_max` 是注册时设的两个上限。

**收发规则**

- 同一时间只有一个客户端、一条未完成的请求。上一条消息还没取走，或上一条回复还没送出时，新消息会被拒绝，客户端收到“忙”。
- `recv` 取出整条消息。缓冲不够时返回 `APP_CHANNEL_TOO_SMALL`，把需要的长度写进 `*received`，消息留在通道里。
- `send` 复制一条回复，长度不超过 `max_reply_bytes`。上一条回复还没送出时返回 `APP_CHANNEL_AGAIN`。返回成功表示回复已经复制到宿主。
- **回复要在同一个 CHANNEL 事件里发。** 客户端发来一条消息时，宿主立即派发带 `APP_CHANNEL_RX_READY` 的 CHANNEL 事件，事件返回后马上取走回复交给客户端。应用必须在这个事件里调用 `send`，否则客户端这次请求失败。

收发状态码和 CoC 的取值一一对应：`APP_CHANNEL_OK` 0、`AGAIN` 1、`CLOSED` 2（没有会话，或客户端失去授权）、`STALE` 3（`session` 过期或通道未注册）、`TOO_SMALL` 4、`BUSY` 5。参数或调用时机错误时返回负数的 `APP_ERR_*`。

**事件**（`APP_EVENT_CHANNEL`）：`type` 的每一位代表一种情况：

| 位 | 含义 |
|---|---|
| `APP_CHANNEL_CONNECTED`(1) | 客户端打开了会话 |
| `APP_CHANNEL_DISCONNECTED`(2) | 会话关闭，或客户端失去授权（例如配对被重置） |
| `APP_CHANNEL_RX_READY`(4) | 有一条消息待读，`rx_pending` 为 1 |
| `APP_CHANNEL_TX_READY`(8) | 上一条回复已送出，`tx_free` 为 1 |
| `APP_CHANNEL_ERROR`(16) | 错误 |

**生命周期**：注册只在本次加载内有效。调用 `close`、切走应用或卸载应用时，宿主释放缓冲，旧的会话编号随之失效。STOP 里只能调用 `info` 和 `close`。

### 允许导入的函数

应用 ELF 调用的外部函数，都必须列在 `sdk/abi2_imports.inc` 里。打包工具和宿主用同一份列表检查，用到列表外函数的安装包会被拒绝。

| 类别 | 函数 |
|---|---|
| 宿主服务 | 上面各节列出的 `app_host_*` 函数。`app_host_ble_*` 要求声明的次版本至少为 1，`app_host_coc_*` 至少为 2，`app_host_channel_*` 至少为 3 |
| 内存 | `malloc` `calloc` `realloc` `free` |
| 内存与字符串 | `memcpy` `memmove` `memset` `memcmp` `strlen` `strcmp` `strncmp` `strncpy` `strchr` `strrchr` `strcpy` |
| 其他 C 库 | `setjmp` `longjmp` `abort` `floor` `pow` `fprintf` `fputc` `fputs` `fread` `stderr` |
| 编译器辅助 | `__bswapsi2` 以及一组双精度浮点辅助函数（`__adddf3` `__subdf3` `__muldf3` `__divdf3` `__floatsidf` `__floatunsidf` `__fixdfsi` `__fixunsdfsi` `__ledf2` `__gedf2` `__extendsfdf2` `__truncdfsf2`） |

`printf`、`snprintf`、`strstr`、`memchr` 这类函数不在列表里，需要时自己实现。

超过 512 字节的 `malloc` 会优先从 PSRAM 分配。宿主不给应用单独划分内存额度，所以每次分配都要检查是否失败。

## 借用指针与资源生命周期

| 对象 | 有效期 | 谁释放 |
|---|---|---|
| `app_host_context_v2()` 返回的上下文 | 本次 `main()` 返回前 | 宿主 |
| 上下文里的 `data`（PNG、BLE/COC/CHANNEL 事件） | 本次 `main()` 返回前 | 宿主 |
| `app_host_cache_get_v2()` 返回的指针 | 本次事件内，只读 | 宿主 |
| 传给 display、cache_put、ble_register、ble_set、coc_send、channel_send 的缓冲 | 宿主只在调用期间读取或复制 | 应用，调用返回后即可释放 |
| 应用 `malloc` 的内存 | 直到应用 `free` | 应用，最晚在 STOP 里释放 |
| 定时器、GATT 服务、CoC 通道、CHANNEL 注册，以及它们排队的事件 | 本次加载 | 宿主在卸载时自动取消 |
| 缓存内容 | 跨加载保留，直到重启、被回收、删除应用或应用自己清除 | 宿主 |

宿主只保存应用数据的副本，所以应用卸载后，宿主不会再访问它的代码和数据。

## 包格式

`tools/module.py package` 生成的 `.pkg` 由 4096 字节的包头和后面的 ELF 组成。包头字段如下：

| 字段 | 内容 |
|---|---|
| `magic` | `0x31464C45` |
| `format` | 2（带应用 ID） |
| `abi` | 2（有清单时）；1（旧应用） |
| `length` | ELF 字节数 |
| `version` | 发布号，`--version` 指定 |
| `sha256` | ELF 的 SHA-256 |
| 应用 ID | 32 字节，NUL 补齐，必须等于清单里的 `id` |

整包最大 1 MiB，也就是一个应用槽的大小。宿主收到安装包时，会重新检查包头、SHA-256、ELF 结构、清单和用到的函数。

## 版本兼容规则

1. **宿主向下兼容。** 支持 2.3 的宿主能运行声明 2.0、2.1、2.2、2.3 的应用，旧应用不用重新编译。
2. **宿主只接受次版本不高于自己的应用。** 次版本更高的应用在上传时就会被拒绝。所以要先升级宿主固件，再安装需要新 ABI 的应用。
3. **用到的函数和声明的版本必须匹配。** 用了 `app_host_ble_*` 要声明至少 2.1，`app_host_coc_*` 至少 2.2，`app_host_channel_*` 至少 2.3。打包工具和宿主都会检查。声明了较高的次版本、但不用对应的新服务，也是可以的。
4. **事件按声明投递。** 应用只会收到它声明的次版本所包含的事件。
5. **结构体布局固定。** 已发布的结构体大小保持不变，头文件里的 `_Static_assert` 会在编译时检查：

   | 结构体 | 大小（字节） |
   |---|---|
   | 上下文、结果、清单 | 44、20、128 |
   | BLE 服务、BLE 事件 | 120、140 |
   | CoC 配置、信息、事件 | 16、24、24 |
   | CHANNEL 配置、信息、事件 | 28、20、24 |

   新能力通过新的次版本、新函数和新事件加入。
6. **SDK 两份保持一致。** 宿主 `sdk/` 是原件，`photoframe/components/app_sdk/` 是副本，内容必须完全相同。
7. **只用 SDK 里的定义。** 包头由打包工具生成，结构体直接用头文件里的定义，这样宿主和应用的约定才能保持一致。
