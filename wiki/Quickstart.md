# 应用快速开始

[返回首页](Home.md)

本页先构建无显示的 lifecycle 示例，理解事件式应用，再进入显示或 BLE 开发。构建不访问设备；安装与激活属于后续独立步骤。

## 准备环境

保持 `photopainter-host`、`photoframe`、`ai-quota-frame`、`photopainter-console`、`esp32s3` 为兄弟目录，先看各仓 `git status`。控制台独立维护；当前 CoC APP 需要 ABI 2.2 宿主。按 [代码与部署状态](Status-and-Evidence.md) 核对实际提交和设备能力。

ESP-IDF 必须固定为 `5e6f53cdb31fe5708eae3f55af9737be2822db22`，Registry elf_loader 为未修改的 1.3.3。工程会校验 IDF 提交；已有其他版本时使用独立 checkout，不改动别人的工具链或 managed components。

```sh
. /absolute/path/to/pinned-esp-idf/export.sh
git -C "$IDF_PATH" rev-parse HEAD
# 在 photopainter-host 中，无串口参数，不会重置设备：
python3 tools/doctor.py
```

原生测试还需要 C 编译器、CMake、Ninja；宿主测试需要 OpenSSL 开发库。电脑端需要 `go.mod` 指定版本和 Chrome；macOS BLE 构建必须启用 CGO。

### 换电脑继续开发

五个仓库的当前实现均已进入 `main`，不依赖上一台电脑的 build 目录或未提交的临时脚本。设备已安装的宿主与 photoframe v20 会保留，换地点或换电脑不要求重新刷写/provision。

- 仅换地点、仍使用原 Mac：保持相框稳定供电、Mac 蓝牙开启及互联网可用；当前相框走 BLE，不要求它加入当地 Wi-Fi。
- 换新 Mac：克隆上述五仓，安装 Go、Chrome 和 Xcode Command Line Tools；宿主/APP 构建另需固定版本 ESP-IDF。当前 SwiftUI 控制台要求 Apple Silicon、macOS 14+；当前 CoC 客户端为 macOS 实现。
- 私有 `.env`、设备独立 token、Google 服务账号文件和必要绑定配置不在 Git 中，须通过私有方式迁移，权限保持 0600。配置中的文件路径和本机代理端口须按新电脑调整。
- 若源码不在默认兄弟目录，可在后台私有配置中设置 `PHOTOFRAME_HOST_REPO`，使 doctor、打包和 NVS 生成找到宿主工具；固定 IDF 可通过 `IDF_PATH` 指定。
- Windows/Linux 可以构建 ESP32 固件与 APP，桌面发送器保留 GATT 路径；当前 SwiftUI GUI 和已完成的 CoC 后端不能直接当成跨平台版本。

在新 Mac 的 `ai-quota-frame` 中执行 `make build`，再用 `python3 deploy/install-macos.py --binary dist/ai-quota-frame --env-file /private/path/frame.env --no-start` 安装后台与 USB 维护组件。在 `photopainter-console` 中执行 `sh build.sh` 构建控制台。

先暂停旧电脑自动推图，再从新控制台启动后台、读取鉴权设备状态并完成一次显示确认，避免两台电脑同时连接。不要把删除过的旧构建文件、Flash/NVS 备份或私有配置提交到新仓库。意外 POWERON 复位的根因仍未定位，迁移不等于该问题已解决。

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
| 默认 `photoframe/main/entry.c` | ABI 2.2 CoC 收图、PNG 解码、等待页与 RAM 帧恢复 |

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
