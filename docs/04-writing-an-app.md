# 编写一个应用

这一页带你从零写一个相框应用，并把它装到相框上。适合第一次写应用的开发者，你需要会写 C，大致知道 ESP-IDF（乐鑫的 ESP32 开发框架）是什么。

读完你会知道应用的代码结构，以及怎么构建、打包、安装和调试。应用怎样被加载、A/B 槽怎样切换，原理在[设备是怎么工作的](03-how-the-device-works.md)；接口的完整规则在[应用接口参考](05-app-api-reference.md)。

## 应用是什么

相框上的**应用**是一个独立编译的 ELF 文件，由宿主固件在运行时加载。它和宿主共用同一块内存，没有自己的进程和线程。

应用是**事件驱动**的。每当发生一件和应用有关的事，比如启动、定时器到期、即将被切走，宿主就调用一次应用的 `main()`。应用在 `main()` 里处理这一个事件，向宿主报告结果，然后返回。应用里没有 ESP-IDF 常见的 `app_main()`，也没有一直运行的循环。

```mermaid
sequenceDiagram
    participant H as 宿主
    participant A as 应用 main()
    H->>A: START（加载后第一次调用）
    A->>H: app_host_display_v2() 画首屏
    A->>H: app_complete(OK, APP_READY)
    Note over H: 首屏成功 + READY → 确认这个版本
    H->>A: TIMER（定时器到期）
    A->>H: app_complete(OK)
    Note over H: 用户短按功能键，切到下一个应用
    H->>A: STOP
    A->>H: app_complete(OK)
    Note over H: 取消定时器等资源，卸载 ELF
```

功能键由宿主处理。短按功能键时，宿主给当前应用发 STOP 并卸载它，再加载下一个应用并发 START。所以应用感知到的是“被切走”和“被切回来”这两件事。

## 准备环境

### 仓库

把下面两个仓库克隆成兄弟目录（同一个父目录下）：

| 仓库 | 用途 |
|---|---|
| `photopainter-host` | 宿主固件、SDK 原件（`sdk/`）、打包工具（`tools/module.py`） |
| `photoframe` | 相框应用和示例应用；`components/app_sdk/` 里有 SDK 副本和构建辅助脚本 |

安装到设备还需要 `ai-quota-frame`（命令行）或 CPM 网页插件，见 [电脑端、控制台与网页插件](07-clients.md)。

### 固定版本的 ESP-IDF

宿主和应用都必须用同一个版本的 ESP-IDF 构建，版本固定到某一个 Git 提交：

| 依赖 | 版本 | 在哪里固定 |
|---|---|---|
| ESP-IDF | v6.0.3，提交 `5e6f53cdb31fe5708eae3f55af9737be2822db22` | 各工程的 `CMakeLists.txt` 开头会检查 `$IDF_PATH` 指向的提交，不一致就报错 |
| `espressif/elf_loader` | 1.3.3，来自乐鑫组件仓库，原样使用 | `main/idf_component.yml` 里写 `^1.3.3`，`dependencies.lock` 锁定为 1.3.3 |

如果电脑上已经有别的 IDF 版本，可以另外检出一份固定版本，和原有的工具链分开放。

```sh
. /path/to/esp-idf/export.sh
git -C "$IDF_PATH" rev-parse HEAD     # 应输出 5e6f53cd...
```

在 `photopainter-host` 里运行 `python3 tools/doctor.py` 可以检查开发环境。它默认不打开串口，所以设备不会复位。

### SDK 头文件

应用只需要包含一个头文件 `photopainter_app.h`。它的原件在 `photopainter-host/sdk/`，`photoframe/components/app_sdk/` 里放着一份内容完全相同的副本。改了一边就要同步另一边，可以这样检查两边是否一致：

```sh
# 在 photopainter-host 中
cmp sdk/photopainter_app.h ../photoframe/components/app_sdk/include/photopainter_app.h
cmp sdk/abi2_imports.inc   ../photoframe/components/app_sdk/abi2_imports.inc
```

## 现成的示例

`photoframe/examples/` 里有三个可以直接构建的示例，建议先读一遍：

| 示例 | 演示内容 | ABI |
|---|---|---|
| `lifecycle` | 不画屏。START 时设一个 1 秒的定时器，每次 TIMER 计数并写缓存，STOP 时取消定时器 | 2.0 |
| `color-test` | START 时生成六色竖条并显示 | 2.0 |
| `ble-echo` | 注册一个自定义 GATT 服务，把写入的值回显到可读特征 | 2.1 |

默认的相框应用 `photoframe/main/entry.c` 是一个完整的大例子。它会解码 PNG、显示等待页、从内存缓存恢复上一张图，并支持 GATT、CoC、CHANNEL 三种蓝牙收图方式（ABI 2.3）。

下面的教程以 `lifecycle` 的工程结构为蓝本，写一个会画屏的应用 `hello-frame`。

## 第一步：建工程目录

在 `photoframe/examples/` 下新建 `hello-frame`，结构如下：

```text
photoframe/examples/hello-frame/
├── CMakeLists.txt
└── main/
    ├── CMakeLists.txt
    ├── idf_component.yml
    └── entry.c
```

放在 `examples/` 下，就能直接用相对路径引用 `../../components/app_sdk` 和 `../../sdkconfig.app.defaults`。放到别处也可以，把这些路径改对就行。

顶层 `CMakeLists.txt`（照抄 `lifecycle`，改工程名）：

```cmake
cmake_minimum_required(VERSION 3.16)
execute_process(COMMAND git -C "$ENV{IDF_PATH}" rev-parse HEAD
    OUTPUT_VARIABLE IDF_REV OUTPUT_STRIP_TRAILING_WHITESPACE COMMAND_ERROR_IS_FATAL ANY)
if(NOT IDF_REV STREQUAL "5e6f53cdb31fe5708eae3f55af9737be2822db22")
    message(FATAL_ERROR "Use the stack's pinned ESP-IDF")
endif()
set(SDKCONFIG "${CMAKE_BINARY_DIR}/sdkconfig")
set(SDKCONFIG_DEFAULTS "${CMAKE_CURRENT_LIST_DIR}/../../sdkconfig.app.defaults")
set(EXTRA_COMPONENT_DIRS "${CMAKE_CURRENT_LIST_DIR}/../../components/app_sdk")
set(PHOTOFRAME_ABI "2")
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(hello_frame)
if(NOT IDF_TARGET STREQUAL "esp32s3")
    message(FATAL_ERROR "This example only supports ESP32-S3 PhotoPainter")
endif()
set(ELF_COMPONENTS "")
include("${CMAKE_CURRENT_LIST_DIR}/../../components/app_sdk/app_elf.cmake")
include(elf_loader)
project_elf(hello_frame)
```

`project_elf()` 来自 elf_loader 组件，它把工程链接成宿主可以加载的 ELF。IDF 构建时还会顺带生成一个普通固件，`app_elf.cmake` 给这个固件补上一个空的 `app_main`，让它能编译通过。这个普通固件用不上。

`main/CMakeLists.txt`：

```cmake
idf_component_register(SRCS "entry.c" REQUIRES app_sdk)
```

`main/idf_component.yml`：

```yaml
dependencies:
  idf: "==6.0.3"
  espressif/elf_loader: "^1.3.3"
```

如果从 `lifecycle` 复制目录，也一并复制 `dependencies.lock`，这样 elf_loader 会锁定在 1.3.3。`build*` 目录是构建产物，只复制源文件即可。

## 第二步：写应用代码

`hello-frame` 做四件事：

- **START**：画首屏（白底、黑边框、中间一个彩色方块），设一个 1 小时的定时器，报告 `APP_READY`。
- **TIMER**：换一种方块颜色重画，并把当前页号写进宿主缓存。切走再切回来时，START 从缓存恢复页号。
- **INPUT**：这个应用不接收 PNG，返回“不支持”。
- **STOP**：取消定时器。

`main/entry.c`：

```c
/* SPDX-License-Identifier: MIT */
#include <stdlib.h>
#include <string.h>
#include "photopainter_app.h"

/* ID、显示名、输入能力（0 = 不收 PNG）、标志（需要首屏） */
APP_MANIFEST("hello-frame", "Hello frame", 0, APP_REQUIRES_DISPLAY);

#define STATE_VERSION 1u                 /* 缓存内容的格式版本 */
#define REDRAW_MS (60u * 60u * 1000u)    /* 1 小时 */

static uint32_t page;  /* 每次加载 ELF 都会重新初始化 */

static void draw(uint8_t *frame, uint32_t n) {
    static const uint8_t accents[] = {APP_RED, APP_YELLOW, APP_GREEN, APP_BLUE};
    uint8_t accent = accents[n % 4];
    memset(frame, APP_WHITE, APP_FRAME_BYTES);
    for (uint32_t y = 0; y < APP_FRAME_HEIGHT; y++) {
        for (uint32_t x = 0; x < APP_FRAME_WIDTH; x++) {
            int border = x < 16 || x >= APP_FRAME_WIDTH - 16 ||
                         y < 16 || y >= APP_FRAME_HEIGHT - 16;
            int block = x >= 300 && x < 500 && y >= 140 && y < 340;
            if (border) frame[y * APP_FRAME_WIDTH + x] = APP_BLACK;
            else if (block) frame[y * APP_FRAME_WIDTH + x] = accent;
        }
    }
}

static int32_t show(uint32_t n) {
    uint8_t *frame = malloc(APP_FRAME_BYTES);
    if (!frame) return APP_ERR_MEMORY;
    draw(frame, n);
    int32_t r = app_host_display_v2(frame, APP_FRAME_BYTES);
    free(frame);
    return r;
}

int main(int argc, char **argv) {
    (void)argc;
    (void)argv;
    const app_context_v2_t *c = app_host_context_v2();
    if (!c) return APP_ERR_RUNTIME;

    int32_t r = APP_OK;
    switch (c->event) {
    case APP_EVENT_START: {
        uint32_t bytes = 0;
        const uint8_t *saved = app_host_cache_get_v2(STATE_VERSION, &bytes);
        page = 0;
        if (saved && bytes == sizeof(page)) memcpy(&page, saved, sizeof(page));
        r = show(page);
        if (r == APP_OK) r = app_host_timer_v2(1, REDRAW_MS);
        break;
    }
    case APP_EVENT_TIMER:
        page++;
        r = show(page);
        if (r == APP_OK)
            (void)app_host_cache_put_v2(STATE_VERSION, (const uint8_t *)&page, sizeof(page));
        break;
    case APP_EVENT_INPUT:
        r = APP_ERR_UNSUPPORTED;
        break;
    case APP_EVENT_STOP:
        r = app_host_timer_v2(1, 0);
        break;
    default:
        break;
    }
    app_complete(c, r, c->event == APP_EVENT_START && r == APP_OK ? APP_READY : 0);
    return r;
}
```

逐段解释：

**清单。** `APP_MANIFEST(...)` 在 ELF 里放一段固定格式的数据，叫清单，告诉宿主这个应用的 ID、名字、ABI 版本和能力。

- ID 由 1–31 个小写字母、数字、`_` 或 `-` 组成。安装、切换、删除应用时都用它来指定应用。
- `APP_REQUIRES_DISPLAY` 表示 START 必须成功画出首屏，这个版本才算启动成功。
- `APP_MANIFEST` 声明的是 ABI 2.0。要用蓝牙服务时，换成 `APP_MANIFEST_BLE`、`APP_MANIFEST_COC` 或 `APP_MANIFEST_CHANNEL`，见[清单](05-app-api-reference.md#清单)。

**取上下文。** `app_host_context_v2()` 返回本次事件的描述，包括事件类型 `event`、事件编号、加载次数和附带的数据。这个指针只在本次 `main()` 返回前有效。

**画屏。** 显示缓冲有 800×480 = 384000 字节，每个字节是一个像素，从左上角开始逐行排列，取值是 `APP_BLACK`、`APP_WHITE` 等六个颜色常量。应用按逻辑坐标画就行，旋转和面板编码由宿主完成。这么大的缓冲，`malloc` 会从 PSRAM（外部扩展内存）分配。

`app_host_display_v2()` 要等电子纸刷新完、面板断电后才返回，一次大约 20 秒。

**报告结果。** 每个事件都必须**恰好调用一次** `app_complete()`。宿主只看 `app_complete()` 报告的结果，因为 elf_loader 不把 `main()` 的返回值传给宿主。`APP_READY` 标志只在 START 成功时带上。

**缓存。** `app_host_cache_put_v2()` 把数据复制到宿主内存里，每个应用的缓存互相隔开。切到别的应用再切回来，缓存通常还在；设备重启后就清空了。`STATE_VERSION` 是你自己定义的内容格式号。以后改了存储格式就换个号，旧内容会自动作废。

**STOP。** 宿主在卸载应用前发 STOP。这个例子只需要取消定时器。

- 宿主会自动取消定时器、蓝牙注册这类由宿主管理的资源。应用自己 `malloc` 的内存，要在 STOP 里 `free`。
- STOP 里只做清理：显示和设置新定时器都会失败。
- STOP 要报告 `APP_OK`，否则这次切换会失败。

### 几条硬规则

- 每个事件的处理时间上限是 **5 秒**，显示调用期间宿主会临时放宽。超时后宿主会重启设备，所以耗时的工作要拆到多个事件里做。
- 每个事件最多显示一次。
- 应用只能调用 SDK 允许的函数，列表在 `sdk/abi2_imports.inc`。像 `snprintf`、`printf`、`strstr` 都不在列表里，用了会打包失败。完整列表见[允许导入的函数](05-app-api-reference.md#允许导入的函数)。
- 硬件和系统资源都由宿主管理，应用通过宿主服务使用它们。FreeRTOS 任务、回调注册、GPIO、SPI、I²C 这些接口不对应用开放。

## 第三步：构建 ELF

```sh
cd photoframe/examples/hello-frame
idf.py -B build-app -DIDF_TARGET=esp32s3 build
```

产物是 `build-app/hello_frame.app.elf`，文件名取自 `project_elf()` 的参数。构建目录里还会出现一个普通的 `.bin` 固件，它只是构建副产品，安装时只用 `.app.elf`。

## 第四步：打包成 .pkg

打包工具在宿主仓库里。它先检查 ELF 的格式（必须是 Xtensa 架构的 32 位动态 ELF），读取清单，核对用到的函数都在允许列表里，最后加上 4096 字节的包头：

```sh
cd photopainter-host
python3 tools/module.py package --version 1 \
  --input ../photoframe/examples/hello-frame/build-app/hello_frame.app.elf \
  --output /tmp/hello-frame-1.pkg
```

成功时输出一行 JSON，包含文件路径、字节数、版本和 SHA-256。

- `--version` 是你自己定的发布号，32 位无符号整数，默认 1。宿主不要求递增，但每版用不同的号码，查看槽位时才分得清是哪一版。
- 带清单的 ELF 会打成 ABI 2 安装包，应用 ID 取自清单。`--app` 参数可以省略；写了的话，必须和清单里的 ID 一致。
- 安装包连同包头最大 1 MiB，所以 ELF 最大是 1 MiB 减 4 KiB。

不连相框也可以先在本地检查安装包：

```sh
ai-quota-frame ble inspect --file /tmp/hello-frame-1.pkg
```

## 第五步：装到设备上

安装分两步：

1. **上传**：把安装包写进这个应用当前没在用的槽，这个版本成为“待激活”。
2. **激活**：宿主加载新版本并发 START，首屏画成功后确认这个版本。原来的版本留作回退。

槽位规则见[应用目录与 A/B 槽](03-how-the-device-works.md#应用目录与-ab-槽)。

相框同一时间只认一个主人，可能是一台电脑的后台，也可能是一个浏览器。用哪个工具安装，取决于现在谁是主人。换主人的方法见[上手使用](02-getting-started.md#一台相框只认一个主人)。

### 方式一：网页插件的开发者工具

在 CPM 网页插件页面，展开相框卡片下方的“开发者工具”：

1. 点 **读取槽位**，查看每个应用两个槽里的版本和标记。
2. 点 **选择安装包**，选 `hello-frame-1.pkg`。页面会先在本机检查安装包，检查通过才能上传。
3. 点 **上传并激活**。上传完成后宿主加载新应用，屏幕会刷出 `hello-frame` 的首屏。

其他按钮（仅上传、激活待定版本、切换、回滚、删除）的作用见[开发者工具](07-clients.md#开发者工具)。

### 方式二：`ai-quota-frame` 命令行

电脑是主人时用命令行。命令默认连接本机最近配对的相框，也可以用 `--device-id 001122334455` 指定：

```sh
ai-quota-frame ble install  --file /tmp/hello-frame-1.pkg   # 上传到非当前槽
ai-quota-frame ble activate --app hello-frame               # 试运行并确认
ai-quota-frame ble status                                   # 查看状态 JSON
```

`install` 只把安装包写进槽里，还要再执行 `activate` 才会运行。`activate` 会等 START 和首屏都完成后才返回。

之后可以用这些命令管理：

```sh
ai-quota-frame ble switch   --app photoframe    # 切到另一个已安装的应用
ai-quota-frame ble rollback --app hello-frame   # 回到可回退槽的版本
ai-quota-frame ble remove   --app hello-frame   # 删除（只能删不在运行的应用）
```

### 用功能键切换

装好之后，短按功能键会按顺序切到下一个已确认版本的应用。待激活的版本只能通过“激活”命令启动。只装了一个应用时，短按没有反应。

## 第六步：更新

改完代码后，用新的 `--version` 重新构建、打包，再“上传并激活”一次。新版本写进另一个槽，激活成功后旧版本留作回退。

## 调试

### 看状态

`ai-quota-frame ble status` 输出的状态是一段 JSON，开发者工具读到的也是它。常用字段（`payload_abi`、`runtime_ready`、`confirmed`、`phase`、`last_result`、`runtime_diagnostic` 等）的含义见[读懂设备状态](08-maintenance.md#读懂设备状态)。写应用时还常看这几个字段：

| 字段 | 含义 |
|---|---|
| `display_completed` | 最近一个事件里是否真的完成了一次显示 |
| `generation` | 加载次数，每加载一次应用加 1 |
| `cache_bytes` / `free_psram` | 缓存占用的字节数、剩余的 PSRAM |

`runtime_diagnostic` 里的阶段名按启动顺序依次是 `slots`、`services`、`boot_guard`、`slot_read`、`elf_init`、`elf_relocate`、`app_start`、`confirm`、`ready`。如果上次启动停在 `app_start`，通常是 START 卡住或崩溃了。

### 看串口日志

应用可以用 `fprintf(stderr, ...)` 打日志，`fprintf`、`fputs`、`stderr` 都在允许列表里。日志从相框的 USB 串口输出。宿主每次加载应用成功时，会打印 `app_runtime: ready bank=... slot=... ABI=... version=...`。

打开 USB 串口可能让设备复位。如果复位发生在应用启动期间，会触发[启动保护](03-how-the-device-works.md#启动保护)。

### 在电脑上测试入口

不用相框也能在电脑上测试 `main()` 的逻辑，步骤如下：

1. 用 `-Dmain=app_entry` 把 `entry.c` 编译成电脑上的目标文件；
2. 自己写几个假的 `app_host_*` 函数；
3. 依次构造 START、TIMER、STOP 的上下文，调用 `app_entry`。

`photoframe/examples/ble-echo/test_entry.c` 就是这样做的，构建规则在 `photoframe/tests/CMakeLists.txt` 里。

## 常见错误

| 现象 | 原因 | 处理 |
|---|---|---|
| CMake 报 `Use the stack's pinned ESP-IDF` | `$IDF_PATH` 不是固定的 IDF 提交 | 切到固定版本的 IDF 再 `export.sh` |
| 打包报 `invalid ELF section or ABI 2 manifest/import` | 可能有三种原因：用了允许列表外的函数；用了蓝牙函数，但清单声明的 ABI 次版本不够；清单字段不合法 | 换掉不允许的函数；按用到的服务选对清单宏；检查 ID 格式 |
| 打包报 `expected Xtensa ET_DYN app ELF` 或 `expected bounded ELF32` | 传的不是 `*.app.elf`（比如 `.bin`）或超过 1 MiB | 用 `build-app/<名字>.app.elf` |
| 打包报 `manifest and package app ID differ` | `--app` 和清单里的 ID 不一样 | 去掉 `--app` 或改成一致 |
| 激活失败，设备回到原来的版本 | START 没报告 `APP_READY`，或声明了 `APP_REQUIRES_DISPLAY` 却没成功显示 | 检查 START 的每条分支都调用了 `app_complete`，并且首屏确实画了 |
| 应用运行一段时间后被卸载，`phase` 变成 `fault` | 某个事件的结果被当成 `APP_ERR_RUNTIME`。常见原因是漏调或多调了 `app_complete`，或者把 `APP_CHANNEL_AGAIN` 这类正数状态码传给了它 | 每个分支恰好调用一次 `app_complete`，状态只用 `APP_OK` 或 `APP_ERR_*` |
| 从这个应用切走时失败，之后没有应用在运行 | STOP 报告了错误码 | STOP 里只做清理，并报告 `APP_OK` |
| `app_host_display_v2` 返回 `APP_ERR_RUNTIME` | 同一个事件里第二次显示，在 STOP 里显示，或配对重置正在进行 | 一个事件只显示一次，并且放在 STOP 以外的事件里 |
| `app_host_display_v2` 返回 `APP_ERR_COLOR` | 有像素值大于 5 | 只用六个颜色常量 |
| `app_host_display_v2` 返回 `APP_ERR_ARGUMENT` | 缓冲不是 384000 字节 | 长度用 `APP_FRAME_BYTES` |
| `app_host_timer_v2` 返回 `APP_ERR_ARGUMENT` | 间隔小于 1000 ms，或在 STOP 里设了非零间隔 | 间隔至少 1000 ms；STOP 里只传 0 取消 |
| `app_host_cache_put_v2` 返回 `APP_ERR_ARGUMENT` | 同一个事件里先调用了 `cache_get` | 在另一个事件里写缓存 |
| 设备反复重启或回退到旧版本 | 某个事件超过 5 秒没返回，看门狗让设备重启；如果发生在启动期间，启动保护会回退 | 缩短单个事件的工作量；回退规则见[启动保护](03-how-the-device-works.md#启动保护) |
| 短按功能键没反应 | 宿主正忙（传图、刷新、写包、切换中），或只有一个可切换的应用 | 等当前操作结束再按 |
| 新应用上传失败 | 设备上已经有 5 个应用 | 先 `remove` 一个不在运行的应用 |

## 下一步

- 想让手机或电脑直接和应用通信：先读[应用通信的三种方式](06-bluetooth.md#8-应用通信的三种方式)选一种，再查[应用接口参考](05-app-api-reference.md#应用-gatt-服务abi-21)里对应的函数。可以从 `examples/ble-echo` 起步。
- 想接收图片：参考 `photoframe/main/entry.c` 和 `main/ble_image.c`。它们用同一套图片事务代码，同时支持三种传输方式。
