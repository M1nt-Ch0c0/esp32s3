# ESP32-S3 固件专栏

账号下没有 `M1nt-Ch0c0/esp32s3/photoframe` 这种嵌套仓库。本仓库就是专栏入口，路径对应：

```text
github.com/M1nt-Ch0c0/esp32s3                 专栏（本仓库）
github.com/M1nt-Ch0c0/esp32s3/tree/main/host  宿主固件 + ELF 加载器
github.com/M1nt-Ch0c0/esp32s3/tree/main/plugins/photoframe  组件：相框
github.com/M1nt-Ch0c0/esp32s3/tree/main/plugins/xxx         以后的组件
```

额度画面主机仍在独立仓库：[ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame)。本专栏只管设备侧：一份冻得住的宿主，业务以组件形式加载，降低整包重刷成本。

## 目标结构

```text
Flash 里的宿主（少改）
  ESP-IDF + 板级 HAL + Wi-Fi + elf_loader
       │  启动后加载
       ▼
  组件 1  photoframe   收图 / 刷屏 / 鉴权
  组件 2  xxx          以后再加
```

宿主用乐鑫 [`espressif/elf_loader`](https://components.espressif.com/components/espressif/elf_loader)（`dlopen` / `esp_elf_*`）。组件编成 `.so` 或 `.app.elf`，放到 SD 或专用分区，不必每次改业务都重链整份 IDF。

当前状态：**专栏和目录骨架已建，宿主与组件尚未实现。** 先把边界定死，再写代码。

## 目录

| 路径 | 角色 | 更换频率 |
|---|---|---|
| [`host/`](host/) | 最小 IDF 应用、加载器、PhotoPainter 板级、Wi-Fi | 低 |
| [`plugins/photoframe/`](plugins/photoframe/) | 相框：推图、刷屏、设备 API | 中 |
| [`plugins/xxx/`](plugins/xxx/) | 预留的下一个组件 | — |
| [`docs/architecture.md`](docs/architecture.md) | 宿主 / 组件契约 | — |

## 相关仓库

- [ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame)：主机渲 800×480 六色 PNG，现网仍可 URL 拉取；推图组件就绪后改为 `POST` 到设备。
- 上游参考（只读，不 fork 进本专栏）：[aitjcize/esp32-photoframe](https://github.com/aitjcize/esp32-photoframe) @ `bf029826`

## 安全

只在受信局域网使用。不要做公网端口映射。SD 上的 Wi-Fi 密码和组件文件视为明文，按物理介质保护。
