# 早上：原生客户端安装与回滚

本文件只记录早上在场执行的步骤；今晚没有执行下列安装、暂停、重载、恢复或设备命令。没有读取 `.env`、配对密钥、运行日志或 IPC，也没有修改仓库。**配对 v2 尚未实机验证。**

## 已核对的真实入口

- 仓库：`/Users/wcy/opensource/ai-quota-frame`。
- 当前运行文件目录：`~/Library/Application Support/ai-quota-frame`；应用：`~/Applications/PhotoPainter Renderer.app`；LaunchAgent：`~/Library/LaunchAgents/com.m1ntch0c0.ai-quota-frame.plist`。
- 只读核对当前 `start.sh`、应用 Info.plist 和 LaunchAgent 的非秘密字段：原生 app launcher 切入上述运行目录，再执行 `start.sh`；脚本私下加载同目录 `.env`，最后把所有参数交给 `ai-quota-frame`。启动器不把凭据放在 argv。
- **本机裸 `pp` 解析成 `/usr/bin/pp`，不是 PhotoPainter。** `~/.local/bin/pp` 不存在。验收清单必须写完整的 `python3 /Users/wcy/opensource/ai-quota-frame/deploy/pp.py ...`，不能写裸 `pp`。
- `deploy/pp.py` 固定追加 `ble` 前缀，因此仅用于 BLE 命令，不能调用 `console pause/resume`。`open` 的退出状态仅说明应用启动/等待结果，不能单靠其 exit code 认定设备操作成功；必须核对底层输出和认证完成回执。
- 当前安装后台的 SHA-256 是 `0a270bee82cb0d4e16724974206d3cdb51f498ff5b16a604e45ee4e14e0300e8`，大小 12027266 字节。它不是今晚最终构建。
- 今晚 macOS arm64/CGO 产物 `../photopainter-local/releases/pairing-v2-night-20261029/client/ai-quota-frame-darwin-arm64`：SHA-256 `9e4860a85aee1c231d2fad7b50fd393a531de2a290fa4a80d2bf22d5cb57e36c`，12043842 字节。以该交付目录的最终 `manifest.json` 为准。
- 当前 app launcher SHA-256：`3c2a94c58c63d9f54dfabdff06a01d73d1ca1353b69f3b8ed53226904b9e8a3d`，51904 字节。升级脚本会重新编译/签名该启动器，所以应一并备份 app。

## 自动推图的真实命令和边界

```sh
PP_RUNTIME="$HOME/Library/Application Support/ai-quota-frame"
"$PP_RUNTIME/ai-quota-frame" console pause
"$PP_RUNTIME/ai-quota-frame" console snapshot
# 只有最终决定原生端继续担任 owner 时，验收后才可执行：
"$PP_RUNTIME/ai-quota-frame" console resume
```

`console snapshot` 只读本地后台快照，不连接设备；`console status` 会入队并连接设备，不能混用。CLI 的 `console` 分支在配置/BLE初始化前返回，不必手工 source `.env`。

暂停成功要求返回没有 `error` 且 `state.automatic=false`。它仅暂停后续自动推图，不打断已经开始的显示。停止/升级前还须确认 `state.operation` 为空、`state.transfer.uncertain=false`，且没有 `rendering/connecting/uploading/displaying/recovering/unknown` 等进行中或未知状态。若结果未知，保留进程和原请求继续处理，不能通过重启清除；原生未知图片事务身份仍在进程内。

暂停会持久化到运行目录 `console/preferences.json`。该文件缺失时当前代码默认 `automatic=true`；因此不能只在启动之后再发 pause 来避免首次推图。安装器不会修改偏好文件，`--no-start` 会卸载并禁用 LaunchAgent，但不会把偏好值改为 false。后文要求先成功 pause，或在后台已确认停止时先写入 false。

`console resume` 会唤醒已有的最新待发快照，可能立即推图，不保证等下一轮定时刷新。**如果浏览器是最终 owner，原生端应保持暂停并保持 LaunchAgent 禁用/卸载，不执行 resume，也不重新启动原生常驻服务。** 暂停本身不会阻止手动 `pp.py` 命令。

本轮没有查询运行后台，所以不对“此刻自动推图开/关”新增事实断言。[早上验收清单](MORNING-ACCEPTANCE.md)应采用验收清单最近一次已确认的状态，并在早上操作前用 `console snapshot` 复核；若后台未连接，先确认它确实已停止，不为查看状态贸然启动。

## 早上升级：备份与安装（今晚未执行）

先完成上面的暂停与静止/确定状态检查；如果存在未知刷屏，停止下面流程。使用同一终端定义路径，`PP_BINARY` 指向最终交付包内的 macOS 产物。

```sh
PP_REPO=/Users/wcy/opensource/ai-quota-frame
PP_RUNTIME="$HOME/Library/Application Support/ai-quota-frame"
PP_BUNDLE="$HOME/Applications/PhotoPainter Renderer.app"
PP_PLIST="$HOME/Library/LaunchAgents/com.m1ntch0c0.ai-quota-frame.plist"
PP_DOMAIN="gui/$(id -u)"
PP_SERVICE="$PP_DOMAIN/com.m1ntch0c0.ai-quota-frame"
PP_BINARY=/Users/wcy/opensource/photopainter-local/releases/pairing-v2-night-20261029/client/ai-quota-frame-darwin-arm64
PP_BACKUP="$HOME/PhotoPainter-backups/native-$(date +%Y%m%d-%H%M%S)"
umask 077
mkdir -p "$PP_BACKUP/runtime/console"

# 已确认没有活动/未知事务后，固定为停止状态以得到一致备份。
launchctl disable "$PP_SERVICE"
if launchctl print "$PP_SERVICE" >/dev/null 2>&1; then
  launchctl bootout "$PP_SERVICE"
fi

for PP_NAME in ai-quota-frame usb-maintenance.py start.sh .env ble-device.json; do
  if [ -f "$PP_RUNTIME/$PP_NAME" ]; then
    cp -p "$PP_RUNTIME/$PP_NAME" "$PP_BACKUP/runtime/$PP_NAME"
  fi
done
if [ -f "$PP_RUNTIME/console/preferences.json" ]; then
  cp -p "$PP_RUNTIME/console/preferences.json" "$PP_BACKUP/runtime/console/preferences.json"
fi
if [ -d "$PP_RUNTIME/ble-bonds" ]; then
  ditto "$PP_RUNTIME/ble-bonds" "$PP_BACKUP/runtime/ble-bonds"
fi
ditto "$PP_BUNDLE" "$PP_BACKUP/PhotoPainter Renderer.app"
cp -p "$PP_PLIST" "$PP_BACKUP/com.m1ntch0c0.ai-quota-frame.plist"
shasum -a 256 "$PP_BACKUP/runtime/ai-quota-frame"
shasum -a 256 "$PP_BINARY"

python3 "$PP_REPO/deploy/install-macos.py" \
  --binary "$PP_BINARY" --env-file "$PP_RUNTIME/.env" --no-start
shasum -a 256 "$PP_RUNTIME/ai-quota-frame"
launchctl print-disabled "$PP_DOMAIN"
```

逐条执行；任何命令失败就停下，不继续。预期新旧二进制摘要分别与交付 manifest、上述旧版本摘要一致；安装提示 `installed but disabled`，`print-disabled` 中该 label 为 true。完整 app、旧后台、辅助脚本、偏好和私有配置都在仓库外的 0700 备份目录。不要打印或提交其中 `.env`、配对文件。

安装器支持的只有 `--binary`、`--env-file`、`--no-start`，没有 `--dry-run` 或 `--keep-paused`；它会 bootout、替换后台和 USB helper、重新生成 start.sh、编译/签名 app、注册 LaunchServices、更新 plist。指向当前运行 `.env` 的同一路径时不会复制替换其内容，只确保 0600 权限。它不更新仓库 `deploy/pp.py` 之外的 PATH wrapper。

默认 `ble-device.json`、`ble-bonds` 都位于该运行目录，支持 `PHOTOFRAME_BLE_BINDING_FILE` / `PHOTOFRAME_BLE_BOND_DIR` 自定义绝对路径。只读核查发现默认位置当前都不存在；不能据此断定没有自定义关系。本轮未打开配置确认 override。若此前明确设置过自定义位置，早上私下把相应文件/目录另行复制进受保护备份，不把密钥写到命令参数或聊天。

如果暂停设置文件原本不存在、pause 无法持久化，或者只安装了未运行过的后台，在**后台已确认停止**时先固定暂停（这段也仅留待早上）：

```sh
python3 - <<'PY'
import json, os
from pathlib import Path
directory = Path.home() / 'Library/Application Support/ai-quota-frame/console'
directory.mkdir(parents=True, exist_ok=True, mode=0o700)
directory.chmod(0o700)
temporary = directory / 'preferences.json.tmp'
fd = os.open(temporary, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
with os.fdopen(fd, 'w') as stream:
    json.dump({'automatic': False}, stream)
    stream.flush()
    os.fsync(stream.fileno())
temporary.chmod(0o600)
os.replace(temporary, directory / 'preferences.json')
PY
```

该偏好只对 macOS 的 BLE 常驻后台生效；本次应使用 BLE 配置，不启动历史 HTTP 模式。若浏览器为最终 owner，安装后保持 `--no-start` 的状态即可，不执行后面的原生启动。

## 原生 owner 的在场启动/测试分支

在暂停偏好已保存、浏览器自动发送关闭且已断开后，才能选原生端进行这条分支。独立 BLE CLI 可在 LaunchAgent 禁用时通过 app identity 执行，优先用它完成配对及单次推图验收：

```sh
python3 "$PP_REPO/deploy/pp.py" scan
python3 "$PP_REPO/deploy/pp.py" pair --device-id ACTUAL_12_HEX_ID --name 'PhotoPainter Mac'
python3 "$PP_REPO/deploy/pp.py" paired --device-id ACTUAL_12_HEX_ID
# 推图使用验收清单中明确选定的 800x480 六色 PNG；不能盲目重复执行。
python3 "$PP_REPO/deploy/pp.py" push --device-id ACTUAL_12_HEX_ID --file /absolute/path/frame.png
```

配对码一致后在设备 GPIO4 短按批准；`paired` 只报告本地关系摘要，不能代替实际鉴权。推图须见匹配的认证完成回执，再看屏幕方向/颜色。这里的设备动作必须由总清单统一编排，不能同时启动浏览器测试或后台推图。

确实保留原生为最终 owner 时，才启动已暂停的常驻后台，先检查状态、数据源，再明确恢复自动：

```sh
launchctl enable "$PP_SERVICE"
launchctl bootstrap "$PP_DOMAIN" "$PP_PLIST"
"$PP_RUNTIME/ai-quota-frame" console snapshot
# 确认 automatic=false、transport=ble；数据源与实体显示均验收完成后：
"$PP_RUNTIME/ai-quota-frame" console resume
"$PP_RUNTIME/ai-quota-frame" console snapshot
```

**若先测试原生、后交给浏览器，必须再次 pause、确认活动已结束、disable/bootout，再在设备端物理重置并在浏览器配对；之后不能恢复原生自动发送。** 恢复本地旧配对备份无法恢复设备已经撤销的 owner。

## 原生安装失败的回滚（默认保持停止）

在前述变量仍有效、并且没有活跃/未知设备事务的终端执行。不会恢复或覆盖设备 NVS，也不自动恢复旧 owner：

```sh
launchctl disable "$PP_SERVICE"
if launchctl print "$PP_SERVICE" >/dev/null 2>&1; then
  launchctl bootout "$PP_SERVICE"
fi
for PP_NAME in ai-quota-frame usb-maintenance.py start.sh; do
  cp -p "$PP_BACKUP/runtime/$PP_NAME" "$PP_RUNTIME/$PP_NAME.rollback"
  mv -f "$PP_RUNTIME/$PP_NAME.rollback" "$PP_RUNTIME/$PP_NAME"
done
mv "$PP_BUNDLE" "$PP_BACKUP/replaced-PhotoPainter Renderer.app"
ditto "$PP_BACKUP/PhotoPainter Renderer.app" "$PP_BUNDLE"
cp -p "$PP_BACKUP/com.m1ntch0c0.ai-quota-frame.plist" "$PP_PLIST"
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister \
  -f "$PP_BUNDLE"
shasum -a 256 "$PP_RUNTIME/ai-quota-frame"
```

预期后台摘要恢复为备份值，服务仍禁用/卸载。若重复回滚，先改 `replaced-PhotoPainter Renderer.app` 为新路径，不能覆盖前次留档。精确安装步骤未改变 `.env`，所以回滚默认保留现有配置；只有早上另有修改且明确要撤销时，才私下从备份恢复 `.env` 并设为 0600。不要自动恢复旧 `ble-bonds` 或自动推图偏好。旧后台与新宿主协议是否兼容必须另验，不能仅因进程能启动就 resume。
