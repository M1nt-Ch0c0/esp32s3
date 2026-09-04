# ESP32-S3 开发专栏

本仓库只做**索引**。运行时栈是：

```text
ESP-IDF          框架（FreeRTOS、Wi-Fi、驱动）
  + elf_loader   官方组件，加载 .so / .app.elf
  + 自己的东西   宿主、相框、其它功能（本账号新仓库）
```

[aitjcize/esp32-photoframe](https://github.com/aitjcize/esp32-photoframe) 也是「IDF 上的一份应用」，不是系统层。自己做就不把它当底包，不 fork、不刷进产品路径。板级时序若要对照，可以当只读参考，见 [docs/pins.md](docs/pins.md)。

## 本账号

| 仓库 | 做什么 |
|---|---|
| [ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame) | 电脑侧：额度采集，渲 800×480 六色 PNG（不是固件） |

以后的宿主固件、相框组件、其它 ESP32-S3 工程在这张表加一行。

## 真正要钉死的上游（不 fork）

| 项目 | 钉死 | 链接 |
|---|---|---|
| ESP-IDF | [`5e6f53c`](https://github.com/espressif/esp-idf/tree/5e6f53cdb31fe5708eae3f55af9737be2822db22)（6.0.3 附近） | [源码树](https://github.com/espressif/esp-idf/tree/5e6f53cdb31fe5708eae3f55af9737be2822db22) |
| ELF 加载器 | [`espressif/elf_loader` v1.3.3](https://components.espressif.com/components/espressif/elf_loader/versions/1.3.3) | [组件登记](https://components.espressif.com/components/espressif/elf_loader/versions/1.3.3) · [源码](https://github.com/espressif/esp-iot-solution/tree/master/components/elf_loader) |

自己的宿主用 `idf.py add-dependency "espressif/elf_loader^1.3.3"` 引入加载器，不要拷一份加载器仓库。

## 安全

设备 API 只放在受信局域网。不要做公网端口映射。
