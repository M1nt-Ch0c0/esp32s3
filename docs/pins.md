# 钉死与参考

产品路径：**ESP-IDF + elf_loader + 本账号工程**。不把第三方相框固件当作运行时一层。

## ESP-IDF

- Commit：`5e6f53cdb31fe5708eae3f55af9737be2822db22`
- 说明：6.0.3 附近，编 ESP32-S3 时对齐这一颗

## ELF 加载器

- 组件：`espressif/elf_loader` **v1.3.3**
- 引入：`idf.py add-dependency "espressif/elf_loader^1.3.3"`
- 登记：https://components.espressif.com/components/espressif/elf_loader/versions/1.3.3
- 源码：https://github.com/espressif/esp-iot-solution/tree/master/components/elf_loader
- ESP32-S3 可在 PSRAM 跑 ELF；可选 `dlopen` / `dlsym`

不 fork。当普通 IDF 组件用。

## 可选参考（不是依赖）

[aitjcize/esp32-photoframe](https://github.com/aitjcize/esp32-photoframe) @ `bf0298263310c3fa023d42eca1e22f55948f1e50` 是另一份 IDF 应用。自己做宿主时**不必刷它、不必 fork 它**。

若要核对微雪 PhotoPainter 7.3 的引脚和 E6 时序，可以只读它的 `waveshare_photopainter_73` HAL，或对照微雪官方驱动。那是抄板级信息，不是把整份相框固件接进栈里。
