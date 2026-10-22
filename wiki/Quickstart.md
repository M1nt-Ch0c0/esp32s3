# 应用快速开始

[返回首页](Home.md)

本页先构建无显示的 lifecycle 示例，理解事件式应用，再进入显示或 BLE 开发。构建不访问设备；安装与激活属于后续独立步骤。

## 准备环境

保持 `photopainter-host`、`photoframe`、`ai-quota-frame`、`esp32s3` 为兄弟目录，检出所需开发分支并先看 `git status`。不要只克隆旧 `main` 后假定已经具备 ABI 2.1。

ESP-IDF 必须固定为 `5e6f53cdb31fe5708eae3f55af9737be2822db22`，Registry elf_loader 为未修改的 1.3.3。工程会校验 IDF 提交；已有其他版本时使用独立 checkout，不改动别人的工具链或 managed components。

```sh
. /absolute/path/to/pinned-esp-idf/export.sh
git -C "$IDF_PATH" rev-parse HEAD
# 在 photopainter-host 中，无串口参数，不会重置设备：
python3 tools/doctor.py
```

原生测试还需要 C 编译器、CMake、Ninja；宿主测试需要 OpenSSL 开发库。电脑端需要 `go.mod` 指定版本和 Chrome；macOS BLE 构建必须启用 CGO。

## 最小应用入口

应用提供 `main()`，每次调用处理一个事件后返回。不要写 `app_main()` 加永久循环，也不要创建任务。

```c
#include "photopainter_app.h"

APP_MANIFEST("my-app", "My first app", 0, 0);
static uint32_t ticks;

int main(int argc, char **argv) {
    (void)argc;
    (void)argv;
    const app_context_v2_t *c = app_host_context_v2();
    if (!c) return APP_ERR_RUNTIME;

    int result = APP_OK;
    switch (c->event) {
    case APP_EVENT_START:
        ticks = 0;
        result = app_host_timer_v2(1, 1000);
        break;
    case APP_EVENT_TIMER:
        ++ticks;
        break;
    case APP_EVENT_STOP:
        result = app_host_timer_v2(1, 0);
        break;
    case APP_EVENT_INPUT:
        result = APP_ERR_UNSUPPORTED;
        break;
    default:
        break;
    }
    app_complete(c, result,
        c->event == APP_EVENT_START && result == APP_OK ? APP_READY : 0);
    return result;
}
```

这是完整可编译的入口，但不是完整工程。APP 无显示要求，成功 START 报告 APP_READY 即可。显示应用必须声明 `APP_REQUIRES_DISPLAY`，并在 START 完成一次真实显示。

## 使用已维护的工程示例

在 `photoframe/examples/lifecycle` 中：

```sh
idf.py -B build-app -DIDF_TARGET=esp32s3 build
```

得到 `build-app/lifecycle.app.elf`。新建 APP 时可在 `photoframe/examples/` 下复制该目录，保留相对 SDK 路径，修改 manifest 的 ID/名称、CMake 的 `project()` 与 `project_elf()`，不要复制构建产物或忽略的配置。此示例的故障注入选项 `LIFECYCLE_STALL_ON_BOOT` 必须保持 OFF。

其他可复用入口：

| 示例 | 用途 |
|---|---|
| `examples/lifecycle` | 无显示、定时与 STOP |
| `examples/color-test` | START 自主首屏、宿主显示接口 |
| `examples/ble-echo` | ABI 2.1 注册、写入事件、设置回显与注销 |
| 默认 `photoframe/main/entry.c` | PNG 解码、等待页与 RAM 逻辑帧恢复 |

SDK 副本必须与宿主一致：

```sh
# 在 photopainter-host 中
cmp sdk/photopainter_app.h ../photoframe/components/app_sdk/include/photopainter_app.h
cmp sdk/abi2_imports.inc ../photoframe/components/app_sdk/abi2_imports.inc
```

## 打包与检查

在 `photopainter-host` 中：

```sh
python3 tools/module.py package --app lifecycle --version 1 \
  --input ../photoframe/examples/lifecycle/build-app/lifecycle.app.elf \
  --output /tmp/lifecycle-v1.pkg
```

打包器会检查 ELF、导入及 manifest，自动选择 ABI 2/format 2。先运行 [原生验证](Testing-and-Release.md)，再按 [OTA 流程](Packages-and-Recovery.md) 安装。构建目录里的普通固件 `.bin` 是构建辅助产物，不能当宿主刷入。

工程依据：[lifecycle](https://github.com/M1nt-Ch0c0/photoframe/tree/d2dd482/examples/lifecycle)、[APP 构建辅助](https://github.com/M1nt-Ch0c0/photoframe/blob/d2dd482/components/app_sdk/app_elf.cmake)。
