# ESP32-S3 开发专栏

本仓库只做**索引**：链到本账号里所有基于 ESP32-S3 的仓库，以及已经核对、不准备改动的上游。这里不放可编译工程，也不 fork 别人的树。

自己写的新东西开新仓库，把链接加进下面的表。

## 本账号

| 仓库 | 做什么 |
|---|---|
| [ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame) | 主机：从 CLIProxyAPI 取额度，渲 800×480 六色 PNG，给 PhotoPainter 显示 |

以后的宿主固件、相框组件、其它板子应用，都在这张表里加一行。

## 上游（只读链接，不 fork）

这些按已验证 revision 使用，不改、不维护一份拷贝。

| 项目 | 钉死版本 | 链接 |
|---|---|---|
| 相框固件 [aitjcize/esp32-photoframe](https://github.com/aitjcize/esp32-photoframe) | commit [`bf029826`](https://github.com/aitjcize/esp32-photoframe/tree/bf0298263310c3fa023d42eca1e22f55948f1e50)（v2.18 已核对） | [源码树](https://github.com/aitjcize/esp32-photoframe/tree/bf0298263310c3fa023d42eca1e22f55948f1e50) · [PhotoPainter 7.3 预编译 merged.bin](https://github.com/aitjcize/esp32-photoframe/releases/download/v2.18.0/photoframe-firmware-waveshare_photopainter_73-merged.bin) |
| ELF 加载器 [espressif/elf_loader](https://components.espressif.com/components/espressif/elf_loader) | 组件 **v1.3.3**（支持 ESP32-S3 / PSRAM、`dlopen`） | [组件登记](https://components.espressif.com/components/espressif/elf_loader/versions/1.3.3) · [源码（esp-iot-solution）](https://github.com/espressif/esp-iot-solution/tree/master/components/elf_loader) · [说明](https://github.com/espressif/esp-iot-solution/tree/master/examples/elf_loader) |
| ESP-IDF | commit [`5e6f53c`](https://github.com/espressif/esp-idf/tree/5e6f53cdb31fe5708eae3f55af9737be2822db22)（测过 6.0.3） | [源码树](https://github.com/espressif/esp-idf/tree/5e6f53cdb31fe5708eae3f55af9737be2822db22) |

板型只认微雪 **ESP32-S3-PhotoPainter**（`waveshare_photopainter_73`，800×480 Spectra 6）。不要用 13.3 寸 E6 裸板的镜像。

钉死的 SHA 和刷写注意见 [docs/pins.md](docs/pins.md)。

## 打算怎么用（还没开自己的固件仓库）

```text
官方相框固件 @ bf029826     ← 先整包刷，验证屏幕 / Wi-Fi
espressif/elf_loader v1.3.3  ← 以后宿主用组件依赖引入，不 fork
ai-quota-frame               ← 画面仍在主机渲
```

加载器和官方相框目前都当成品引用。自己的「宿主 + 组件」工程出现后，在本页加链接即可。

## 安全

相框和设备 API 只放在受信局域网。不要做公网端口映射。
