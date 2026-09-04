# 宿主 + 加载器 + 组件

## 为什么拆

整包 `merged.bin` 更换成本低（2MB USB/OTA），但业务和板级、IDF 编在一起，改一行推图也要重编整份框架。拆开之后：

- **宿主**：IDF、FreeRTOS、Wi-Fi、PhotoPainter HAL、`elf_loader`。尽量冻住。
- **组件**：相框或其它功能，编成 `.so` / `.app.elf`，从 SD 或分区加载。

这不是 Android 插件。宿主仍是唯一上电入口；组件不能脱离宿主单独刷。

## 启动

```text
ROM 下载模式 / 已刷宿主
    → IDF 启动
    → 连 Wi-Fi（SD config 或 NVS）
    → elf_loader 打开约定路径
         /storage/plugins/photoframe.so
         /storage/plugins/xxx.so
    → dlsym 约定符号，进入组件
```

组件崩溃不应毁掉 ROM 下载模式。宿主应在加载失败时停在可刷机、可看串口的状态，而不是花屏死循环。

## 计划中的组件契约（尚未实现）

每个组件导出稳定 C ABI，例如：

```c
int plugin_init(const plugin_host_t *host);
int plugin_start(void);
void plugin_stop(void);
```

`plugin_host_t` 只暴露宿主保证稳定的能力：日志、显示一块已打开的屏、HTTP 注册、重启。屏幕时序、PMIC、Wi-Fi 驱动不进组件。

ABI 一变，旧 `.so` 全部作废。这是拆分的主要代价，所以宿主 API 要比业务更保守。

## 和 ai-quota-frame 的边界

| 放哪 | 内容 |
|---|---|
| 主机仓库 | 额度采集、HTML 渲染、六色 PNG、`FRAME_ACCESS_TOKEN` |
| 本专栏宿主 | 芯片、加载器、板子、Wi-Fi |
| photoframe 组件 | `POST /api/push`、刷屏、设备推图码 |

主机不链进固件。组件不读 CLIProxyAPI。
