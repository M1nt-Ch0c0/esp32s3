# 宿主固件 + ELF 加载器

计划：ESP-IDF 最小应用，依赖 `espressif/elf_loader`，板型仅 `waveshare_photopainter_73`。

尚未放入可编译工程。实现时应钉死 IDF revision，并复用已核对的 PhotoPainter HAL，不重写 E6 驱动。

加载约定（草案）：

- SD：`/plugins/*.so` 或 `*.app.elf`
- 失败：串口打印原因，不刷脏屏，保持可进下载模式
