# 已验证上游钉死

本页只记录链接和 SHA，不保存上游源码。升级时改这一页，并在专栏 README 同步。

## 相框固件

- 仓库：https://github.com/aitjcize/esp32-photoframe
- Commit：`bf0298263310c3fa023d42eca1e22f55948f1e50`
- 板型：`waveshare_photopainter_73`
- 预编译：https://github.com/aitjcize/esp32-photoframe/releases/download/v2.18.0/photoframe-firmware-waveshare_photopainter_73-merged.bin
- merged.bin SHA-256（ai-quota-frame 文档曾核对）：`41a680d59ae65f37ef581fd66568a988fd0e64469617651b1a1ec98e77fd30b3`

不 fork。要看结构或刷官方包，用上面的 commit / Release。定制相框行为以后开自己的仓库，不要改这份树再提上游。

## ELF 加载器

- 组件名：`espressif/elf_loader`
- 版本：`1.3.3`（组件登记页记录 archive `5d75f3f0dc499d9ed4b69284a3741187c2b75a70`）
- 引入：`idf.py add-dependency "espressif/elf_loader^1.3.3"`
- 源码浏览：https://github.com/espressif/esp-iot-solution/tree/master/components/elf_loader
- ESP32-S3 可在 PSRAM 执行 ELF；可选 `dlopen` / `dlsym`

不 fork。宿主工程当普通 IDF 组件依赖即可。加载器本身按官方包使用。

## ESP-IDF

- Commit：`5e6f53cdb31fe5708eae3f55af9737be2822db22`
- 说明：与上述相框固件交叉编译时测过的 6.0.3 附近 revision

编官方相框或以后的宿主时对齐这一颗，避免「随便装一份最新 IDF」。
