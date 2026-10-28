# 早上：宿主备份、刷写与回滚

**尚未实机验证。** 本补充来自源码和本地 ELF 的离线检查；今晚没有执行 USB helper 的任何 action，没有枚举/打开串口、连接 BLE、推图或访问线上服务器。先完成[总验收清单](MORNING-ACCEPTANCE.md)中的暂停、microSD 和现场检查。自动推图最近的确认结果记录在总清单中。

## 1. ABI 镜像与真实 ELF 的离线证据

核查时 host HEAD：`f4453c646d820760102c6bcb11445d62362696f9`。

| 内容 | 两份路径 | 字节数 | SHA256 / 结果 |
| --- | --- | ---: | --- |
| SDK header | `photopainter-host/sdk/photopainter_app.h`；`photoframe/components/app_sdk/include/photopainter_app.h` | 7205 | 两者完全一致：`7611277cf022d08864dee4f6b5c7eeaeb6c44d8b6597831fe3a0393f87f4cf6a` |
| ABI 2 imports | `photopainter-host/sdk/abi2_imports.inc`；`photoframe/components/app_sdk/abi2_imports.inc` | 1488 | 两者完全一致：`35394593517623456b049bc78abcc9bb12ab59e7d2eb02ae5b6d9af4ab28ff66`；54 个唯一 allowlist imports |
| 已有 v20 ELF | `/Users/wcy/opensource/photopainter-local/releases/abi-2.2-v20/photoframe.app.elf` | 76256 | `22d4946bbeb53de904232ea136ad8c4a7f40b866493176cbe03105c938264741` |

实际路径是 `abi2_imports.inc`，不存在预想的 `app_imports.txt`。使用 host 的 `tools/app_manifest.py` 只读解析已有 v20 ELF：`id=photoframe`、ABI `2.2`、inputs `1`、flags `1`；41 个未定义动态导入全部在 SDK allowlist 内，没有越界导入。此前完整 host 测试 90/90 通过、0 skipped，已包括真实 ELF 的 C validator；日志 `/tmp/photopainter-host-night-final-tests.log`。

这些结果证明源码镜像和 ELF 导入兼容，不证明新配对固件已在实机正常运行。

## 2. 早上操作前条件

以下命令**只在早上用户在场时执行**。例子以这台 Mac 的路径为准。

1. 核对正式验收清单中的提交、最终固件路径和最终 SHA256。不要沿用提交前的 `2f9f2217…` 候选摘要；Git 派生 `PROJECT_VER` 可能令提交后构建不同。
2. 暂停自动推图；确认没有正在上传/显示的任务，也没有 `transfer.uncertain=true`。若结果未知，只能由原发送器查询原请求；不要通过停止/重启发送器或 USB 复位来清掉未知状态。
3. 关闭浏览器的设备连接，关闭其他 BLE 客户端和串口监视器。若自动后台本来已禁用且未运行，保持该状态，不要为了 USB 维护启动它。
4. 已安装 microSD 时先备份卡内容；若本次不备份，记录用户在场时的明确豁免。完整 Flash 备份不包含 microSD。
5. 正常升级只写 factory。不要执行 `idf.py flash`、通用全量 esptool 刷写配方或 `usb-provision`。

若后台当前运行，以下是本地 IPC 的暂停和快照命令；快照不会刷新设备状态：

```bash
PP_CLIENT="$HOME/Library/Application Support/ai-quota-frame/ai-quota-frame"
"$PP_CLIENT" console pause
"$PP_CLIENT" console snapshot
```

预期 `state.automatic=false`、`state.transfer.uncertain=false`、没有 `state.operation`，且没有 uploading/displaying 等进行中状态。暂停不打断已接纳的物理显示。确认已有工作结束后，使用直接 helper 的维护方式须停用后台：

```bash
launchctl disable "gui/$(id -u)/com.m1ntch0c0.ai-quota-frame"
launchctl bootout "gui/$(id -u)/com.m1ntch0c0.ai-quota-frame"
```

`bootout` 对本来就未加载的服务会报“找不到服务”；先按正式清单核对服务/进程状态，不要把该提示当作需要重新启动。不要关闭原发送器来掩盖未知结果。下面 USB 步骤不包含恢复自动推图命令。

## 3. 重新识别端口、校验候选并备份

如果机内有 microSD，先在设备空闲且没有未知事务时正常关机并取卡，用读卡器连接 Mac。下面只读卡制作镜像，不格式化或恢复卡。先 `diskutil list external physical`，根据容量和读卡器身份核对真实整盘；不要把示例 `diskN` 原样执行，更不能选系统盘：

```bash
diskutil list external physical
# 以下两个变量仅在核对真实整卡身份后填写，N 不是分区号。
PP_SD_DISK=/dev/diskN
PP_SD_RAW=/dev/rdiskN
PP_SD_BACKUP="$HOME/PhotoPainter-backups/sd-$(date +%Y%m%d-%H%M%S)"
umask 077
mkdir -p "$PP_SD_BACKUP"
diskutil info "$PP_SD_DISK"
diskutil unmountDisk "$PP_SD_DISK"
sudo dd if="$PP_SD_RAW" of="$PP_SD_BACKUP/microsd.img" bs=4m
sudo chmod 600 "$PP_SD_BACKUP/microsd.img"
sudo shasum -a 256 "$PP_SD_BACKUP/microsd.img"
sudo shasum -a 256 "$PP_SD_RAW"
diskutil eject "$PP_SD_DISK"
```

预期镜像大小等于卡的整盘字节数，两次 SHA-256 相同；镜像保存在仓库外私有目录。读取失败或摘要不同就停止刷写、保留原卡，修复读卡条件后重新备份。不要显示/上传镜像内容。退出读卡器后把原卡装回相框再供电。若明确无卡则记“无卡”；未做备份必须记录用户现场明确豁免，不能自行推断。

建议在 Bash 执行，避免管道吞掉 helper 的失败退出状态。`PP_RELEASE_SHA` 已填入最终产物清单中的值；`PP_PORT` 必须来自当天重新枚举的结果。

```bash
set -o pipefail
umask 077
PP_PYTHON=/Users/wcy/.espressif/python_env/idf6.0_py3.12_env/bin/python
PP_TOOL=/Users/wcy/opensource/ai-quota-frame/deploy/usb-maintenance.py
PP_EXPECTED_ID=9c139eb063e8
PP_BACKUPS="$HOME/Library/Application Support/ai-quota-frame/console/usb-backups"
PP_HOST_BIN=/Users/wcy/opensource/photopainter-local/releases/pairing-v2-night-20261029/host/photopainter_host.bin
PP_RELEASE_SHA=014ce22bed5bc97564c9b6d192ede1daa705372cbd32dff9fc12c799c0343dc4

"$PP_PYTHON" "$PP_TOOL" --action usb-ports \
  --expected-id "$PP_EXPECTED_ID" --output-root "$PP_BACKUPS"
```

`9c139eb063e8` 是此前记录的设备 ID，须与眼前的设备重新核对。`usb-ports` 只枚举端口，不打开串口；返回的条目有 `port`、`description`、`device_id`、`matches_expected`。不要复用昨天的 `/dev/cu.usbmodem1101`。多个候选或 USB 序列号缺失时，先识别实体设备；不猜选端口。真正进行 backup/flash/restore 时 helper 还会读取芯片 MAC，要求 ESP32-S3 且 MAC 精确匹配，不匹配则不写设备。

```bash
PP_PORT='/dev/cu.usbmodem替换为刚刚确认的端口'

"$PP_PYTHON" - "$PP_HOST_BIN" "$PP_RELEASE_SHA" <<'PY'
import hashlib, pathlib, re, sys
candidate, expected = pathlib.Path(sys.argv[1]), sys.argv[2]
assert re.fullmatch(r"[0-9a-f]{64}", expected), "先填最终产物 SHA256"
assert hashlib.sha256(candidate.read_bytes()).hexdigest() == expected, "固件与最终产物清单不一致"
print("Final host artifact SHA256 verified")
PY

"$PP_PYTHON" "$PP_TOOL" --action usb-inspect \
  --expected-id "$PP_EXPECTED_ID" --output-root "$PP_BACKUPS" \
  --path "$PP_HOST_BIN"

"$PP_PYTHON" "$PP_TOOL" --action usb-backup \
  --expected-id "$PP_EXPECTED_ID" --output-root "$PP_BACKUPS" \
  --port "$PP_PORT" --baud 115200
```

上面步骤逐条执行；任一步失败就不要执行后续写入。`usb-inspect` 是本地文件解析，预期 `complete=true,type=host-firmware,digest=<最终固件 SHA256>`，检查 ESP32-S3 镜像且 32～0x500000 字节。`usb-backup` 会打开串口、复位设备并读取完整 16 MiB，可能耗时数分钟；预期末行 JSON `complete=true,operation=usb-backup,bytes=16777216`，并给出 `backup_manifest`、`backup_directory`、`digest`。把这些非秘密元数据记入验收记录。

备份目录中 `flash.bin` 是敏感数据；不要打开、打印、上传或提交。helper 已核对长度及设备 MD5，生成 `manifest.json`，设置目录 0700，Flash、manifest、维护日志 0600。`maintenance.log` 也按敏感文件处理。仅校验文件存在或摘要不能替代 helper 的设备匹配检查。

## 4. 只升级宿主 factory

重新检查设备端口仍有效，确保完整备份已完成后执行：

```bash
"$PP_PYTHON" "$PP_TOOL" --action usb-flash \
  --expected-id "$PP_EXPECTED_ID" --output-root "$PP_BACKUPS" \
  --port "$PP_PORT" --baud 115200 \
  --path "$PP_HOST_BIN" --digest "$PP_RELEASE_SHA"
```

helper 会再做一份操作前的完整备份，并核对实际分区表：factory 为 `0x10000/0x500000`，NVS 为 `0x9000/0x6000`。随后只写候选固件到 `0x10000` 并校验；另用设备 MD5 核对 `[0,0x10000)` 和 `[0x510000,0x1000000)` 未改变。不会覆盖 bootloader、分区表、NVS 或应用数据分区。

预期末行 `complete=true,operation=usb-flash`，之后设备复位。**这一末行的 `digest` 仍是本次写入前完整 Flash 备份的 SHA256，不是新固件 SHA256。** 保留该次 `backup_manifest`；不得把成功刷写或屏幕保留的旧画面当作配对/显示验收成功。

新固件后的 GPIO4、配对、重启持久化、浏览器推图和颜色方向确认，继续正式清单。普通 factory 更新保留 Wi-Fi 和应用；配对重置也应保留 Wi-Fi 和应用。重新生成并写整个 NVS 会清除绑定，故不是此升级步骤。

## 5. 失败处理和回滚

若工具报错，先记录末行 JSON，不反复写入。`complete=false,restored=true` 表示已自动恢复本次操作前内容；`restored=false` 可能是尚未开始写，也可能是回滚失败，必须结合 `error` 和是否进入 write 阶段判断。无法确认时保持 USB 连接，保留备份，不盲目重试。

固件兼容性问题优先用已保存的旧 host `.bin` 做 factory-only 回滚：先 `usb-inspect --path <旧host.bin>` 核对其 SHA，再将上面的 `usb-flash` 命令换为旧文件及其 digest。该路径保留当前 NVS/应用。不要把 16 MiB `flash.bin` 当作 host 固件传给 `usb-flash`。

只有明确要恢复整机快照时，使用下面流程。将变量替换为第 3 或第 4 步保存的**目标备份清单**；推荐传 `manifest.json`，不能传目录。也接受同目录的准确文件名 `flash.bin`，但旁边必须有匹配的 `manifest.json`。

```bash
PP_RESTORE_MANIFEST='/绝对路径/所选备份目录/manifest.json'
PP_RESTORE_SHA='替换为所选完整Flash备份的64位SHA256'

"$PP_PYTHON" "$PP_TOOL" --action usb-inspect \
  --expected-id "$PP_EXPECTED_ID" --output-root "$PP_BACKUPS" \
  --path "$PP_RESTORE_MANIFEST"

# 上条必须返回 complete=true,type=backup,digest 与 PP_RESTORE_SHA 完全一致。
"$PP_PYTHON" "$PP_TOOL" --action usb-restore \
  --expected-id "$PP_EXPECTED_ID" --output-root "$PP_BACKUPS" \
  --port "$PP_PORT" --baud 115200 \
  --path "$PP_RESTORE_MANIFEST" --digest "$PP_RESTORE_SHA"
```

备份 manifest 要求 `schema=photopainter-usb-backup-v1`、`device_id` 与目标一致、同目录 `flash.bin` 恰好 16777216 字节且 SHA 匹配。restore 自己先备份当前整机，再写所选 16 MiB 到地址 0 并校验；**会覆盖 Wi-Fi/NVS、配对、目录和应用，丢弃快照后变化**。恢复旧 token 固件后，v2 客户端不会自动降级；客户端版本/配置必须按对应旧版本恢复，保持自动推图暂停，重新确认所有权。

## 6. 通过本机 console 队列的替代方式

如果正式验收选择使用已更新且确认暂停的后台，可通过相同串行队列调用维护；无需同时执行上面的直接 helper 方式。已安装 helper 路径为 `~/Library/Application Support/ai-quota-frame/usb-maintenance.py`，应与本次产物一致。不要为使用这个入口而贸然启动本来禁用的自动服务。

```bash
"$PP_CLIENT" console request '{"action":"usb-ports"}'
"$PP_CLIENT" console request '{"action":"usb-backup","port":"/dev/cu.usbmodem替换为当天端口","confirmed":true}'
"$PP_CLIENT" console snapshot
```

`usb-flash`/`usb-restore` 的 JSON 另带 `path`、`digest`、`confirmed:true`。后台绑定的 device ID 必须正确；不把另一个 device_id 字段当成当前绑定的覆盖方式。队列收到请求不代表维护完成：持续查看 snapshot 的 `operation_result_action`、`operation_result_at`、`operation_result.complete` 和 `operation_error`。非只读 USB action 自动暂停推图，并拒绝 `transfer.uncertain=true`；执行使用与 BLE 相同的 gate。

直接 helper 没有后台队列/自动暂停逻辑，也没有 `--confirmed` 参数。所有 action 都要求 `--expected-id`、`--output-root`；flash/restore 还必须带匹配的 `--digest`。默认115200，支持230400/460800；稳定性优先保留115200。会打开串口的 backup/flash/restore/provision 在结束时都会尝试 hard reset；ports/inspect 不打开串口。

验收结束前不要自动恢复常驻推图。浏览器和 Go 客户端是不同 owner；最终只选择一个，通过物理配对完成归属后，再由用户明确恢复该 owner 的自动任务。
