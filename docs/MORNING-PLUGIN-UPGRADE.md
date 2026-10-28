# 早上：插件 0.2.2 安装、升级与回滚

本文件只依据本地源码、文档和候选清单整理，**本轮未执行任何 SSH、服务器或容器命令**。下列联网/容器命令全部留到用户早上在场时。0.2.2 尚未部署、尚未实机验证；0.2.1 为此前线上历史基线，早上仍要复核。

## 已有可信本地工具与产物

源仓库 `/Users/wcy/opensource/ai-quota-frame`；最终交付目录 `/Users/wcy/opensource/photopainter-local/releases/pairing-v2-night-20261029`。先读[总验收清单](MORNING-ACCEPTANCE.md)。

- `plugins/photopainter/build.sh`：生成网页和共享 Go/WASM，再执行原生构建。
- `plugins/photopainter/build-native.sh`：要求三个生成资源存在，在**本机目标架构**上运行 Go 测试并构建。
- `plugins/photopainter/verify-native.py`：调用真实 C ABI，检查版本、仅 management_api 能力、三个资源与构建字节完全一致、错误路径拒绝。
- 交付目录 `plugin/linux/amd64/photopainter-v0.2.2.so`：Linux/amd64 候选，精确 SHA-256 以交付 `manifest.json` 与 `SHA256SUMS` 为准；不要使用早期 dist 候选摘要。
- 交付 `manifest.json` 明确 `deployed=false`、`hardware_validated=false`；Linux C ABI 与实际 CPA 加载分别记录。

最终包选用 [Linux CI 40772972073](https://github.com/M1nt-Ch0c0/ai-quota-frame/actions/runs/40772972073) 的固定 Debian 12 构建，已在 Linux 执行真实 C ABI/资源检查；实际 CPA 加载仍留晨间验证，再升级生产。交付 .so 为 11,500,432 字节，SHA-256 `4307bcd790941cd3b0be3f36cfa0919f7a7898b2febf4f646f8f40a9f693b02b`。它与 Go 1.27.3 生成的网页引擎配套，不混用本地 Zig 的旧资源。

## 1. 本地核对与 staging

```sh
PP_RELEASE=/Users/wcy/opensource/photopainter-local/releases/pairing-v2-night-20261029
cd "$PP_RELEASE"
shasum -a 256 -c SHA256SUMS

tar -czf /tmp/photopainter-v0.2.2-staging.tgz \
  plugin tools/verify-plugin-native.py manifest.json

# 以下只在早上执行。用 mktemp 返回的新目录，不覆盖旧 staging。
plugin_stage=$(ssh -o BatchMode=yes root@cpa-direct.m1ntch0c0.com \
  'mktemp -d /var/tmp/photopainter-v022.XXXXXX')
case "$plugin_stage" in /var/tmp/photopainter-v022.*) ;; *) exit 1 ;; esac
scp /tmp/photopainter-v0.2.2-staging.tgz \
  "root@cpa-direct.m1ntch0c0.com:$plugin_stage/source.tgz"
printf '%s\n' "$plugin_stage"
```

预期：本地库哈希完全一致。后续服务器命令中的 `STAGE` 应设为上述新目录，不能借用含真实配置/账号的旧目录。

生成资源的预期哈希：

- HTML：`3925ac9762b43b9744344684ea8559e5bec83356d7169d94f95aed096355678a`（82,445 B）
- gzip WASM：`537912ba76ab08f734ff118cdaf938fb25cbd81655ac7d6288f4a9821e215a31`（4,419,135 B）
- wasm_exec.js：`0c949f4996f9a89698e4b5c586de32249c3b69b7baadb64d220073cc04acba14`（16,992 B）

## 2. Linux C ABI 与 network-none CPA 隔离加载

以下在服务器 root shell 中、仅测试 staging 运行。第一步重新核对当前生产镜像、UID/GID和工作目录；历史为 CPA 8.0.45、987:989、`/var/lib/cliproxyapi`，不把历史值当成当前已核实。

```sh
STAGE=/var/tmp/photopainter-v022.REPLACE_WITH_ACTUAL
cd "$STAGE"
tar -xzf source.tgz
sha256sum plugin/linux/amd64/photopainter-v0.2.2.so
python3 tools/verify-plugin-native.py \
  "$STAGE/plugin/linux/amd64/photopainter-v0.2.2.so" \
  "$STAGE/plugin/panel.html"
docker inspect cpa-cliproxyapi \
  --format '{{.Image}} {{.Config.Image}} {{.Config.User}} {{.Config.WorkingDir}}'
```

预期：C ABI 输出 `version=0.2.2`、`abi=1`、`schema=7` 和三个匹配资源。若装载/GLIBC/导出失败，停止，不复制到生产。

在版本/UID仍与上述基线相符时，创建**独立测试配置与空账号目录**。不得挂载生产配置、auth、数据库、token或后端目录。

```sh
mkdir -p isolated/plugins/linux/amd64 isolated/auth
chmod 755 "$STAGE" "$STAGE/isolated"
chown 987:989 isolated/auth
chmod 700 isolated/auth
install -o 987 -g 989 -m 644 \
  plugin/linux/amd64/photopainter-v0.2.2.so \
  isolated/plugins/linux/amd64/photopainter-v0.2.2.so
cat > isolated/config.yaml <<'YAML'
host: "127.0.0.1"
port: 8317
auth-dir: "/stage/isolated/auth"
remote-management:
  allow-remote: false
  secret-key: "photopainter-isolated-test-key-not-production"
  disable-control-panel: true
api-keys: ["isolated-test-key-not-for-model-requests"]
plugins:
  enabled: true
  dir: "/stage/isolated/plugins"
  configs:
    photopainter:
      enabled: true
      priority: 0
YAML
chown 987:989 isolated/config.yaml
chmod 600 isolated/config.yaml
plugin_test_image=$(docker inspect cpa-cliproxyapi --format '{{.Image}}')
plugin_test_name=photopainter-v022-acceptance
# 该名字若已存在，先核对来源；不要直接删掉不认识的容器。
docker run -d --name "$plugin_test_name" --network none --user 987:989 \
  --mount "type=bind,src=$STAGE,dst=/stage" \
  --entrypoint /CLIProxyAPI/CLIProxyAPI "$plugin_test_image" \
  -config /stage/isolated/config.yaml -local-model
plugin_test_pid=$(docker inspect "$plugin_test_name" --format '{{.State.Pid}}')
nsenter -t "$plugin_test_pid" -n python3 - "$STAGE" <<'PY'
from pathlib import Path
import hashlib,json,sys,time,urllib.request,urllib.error
root=Path(sys.argv[1]); opener=urllib.request.build_opener(urllib.request.ProxyHandler({}))
def get(path,auth=False):
 headers={'Authorization':'Bearer photopainter-isolated-test-key-not-production'} if auth else {}
 try:
  with opener.open(urllib.request.Request('http://127.0.0.1:8317'+path,headers=headers),timeout=15) as r:return r.status,r.read(32*1024*1024)
 except urllib.error.HTTPError as e:return e.code,b''
for attempt in range(60):
 try:
  if get('/')[0]==200: break
 except OSError: pass
 time.sleep(0.5)
else: raise RuntimeError('isolated CPA did not become ready')
code,body=get('/v0/management/plugins',True);assert code==200
entry=next(p for p in json.loads(body)['plugins'] if p['id']=='photopainter')
assert entry['registered'] and entry['effective_enabled'] and entry['metadata']['version']=='0.2.2'
assert any(m['path']=='/v0/resource/plugins/photopainter/panel' for m in entry['menus'])
resources={}
for name,file in [('panel','panel.html'),('frame.wasm','frame.wasm.gz'),('wasm_exec.js','wasm_exec.js')]:
 code,body=get('/v0/resource/plugins/photopainter/'+name)
 assert code==200 and body==(root/'plugin'/file).read_bytes()
 resources[name]={'bytes':len(body),'sha256':hashlib.sha256(body).hexdigest()}
assert get('/v0/management/plugins/photopainter/account-subject?name=missing.json')[0]==401
assert get('/v0/management/plugins/photopainter/account-subject?name=missing.json',True)[0]==404
print(json.dumps({'isolated_loaded':True,'version':'0.2.2','resources':resources,'anonymous_subject':401,'missing_subject':404}))
PY
# 留下输出/失败日志后，仅清理本次确认由自己创建的隔离实例。
docker stop "$plugin_test_name"
docker rm "$plugin_test_name"
```

预期：注册、菜单、版本、三个完整资源摘要都通过，匿名subject接口401，未知账号404。由于是空auth目录，这一步不证明真实账号subject投影；该项要在生产已有授权下只读核对，不能用假账号成功替代。

## 3. 生产热升级：受控步骤，不提供“盲覆盖全部配置”一键命令

现有仓库**没有已经验证的通用生产安装/回滚脚本**。`build.sh`和`verify-native.py`都不是部署器。先读服务器当前 `/AGENTS.md`、`/opt/server-ops/PHOTOPAINTER-PLUGIN.md`，核实真实目录/UID/已加载版本，保存 root-only 时间戳备份：

- `/etc/cliproxyapi/config.yaml`原字节、inode、owner/mode和SHA。
- 已加载旧库（历史应为0.2.1）与Qoder的SHA、路径。
- 所有现有容器ID、PID、启动时间、重启次数。

只有上述检查通过，才把新文件放入扫描目录；不要覆盖旧库，也不要复用已有同名文件：

```sh
# 路径/UID须由前一步当前状态核实，以下是已有部署的历史路径。
test ! -e /var/lib/cliproxyapi/plugins/linux/amd64/photopainter-v0.2.2.so
install -o 987 -g 989 -m 644 \
  "$STAGE/plugin/linux/amd64/photopainter-v0.2.2.so" \
  /var/lib/cliproxyapi/plugins/linux/amd64/.photopainter-v022.installing
mv /var/lib/cliproxyapi/plugins/linux/amd64/.photopainter-v022.installing \
   /var/lib/cliproxyapi/plugins/linux/amd64/photopainter-v0.2.2.so
```

在 **CPM 的插件管理**中只保存 PhotoPainter 的配置变化以触发重载（例如把其priority从当前值调高1并记录前后值）。若必须脚本改YAML，必须使用审阅过的AST定位，只替换 `plugins.configs.photopainter.priority` 标量；验证解析结果仅此一项变化、保存前原文件未被他人改动，然后以同一打开文件 `r+b` 写入/截断/flush/fsync。**不得 `mv`/`os.replace`/原子rename配置文件。** 上面的 `mv` 只针对新库文件，可以使用。

现有版本选择已在前序隔离热升级验证：扫描目录中新版本库可替换旧活动版本。但早上必须以认证插件列表的 `metadata.version`、实际path和资源摘要证明真正选中了0.2.2，不能仅靠文件名猜测。

必须核对：

1. `/v0/management/plugins`中photopainter registered/effective_enabled、version0.2.2、menu正确；Qoder仍registered。
2. 本地CPA、Manager入口及公网的三个完整资源摘要匹配；gzip响应比较压缩文件，浏览器/Node自动解压后应比较原WASM清单的raw hash。
3. 匿名管理subject仍401；使用现有管理凭据只读查询真实xAI时只返回`sub`，不输出具体值或任何凭据。
4. 容器ID/PID/启动时间/重启次数不变，Qoder文件不变，配置解析仅预期项变化且inode不变。
5. 若重载未生效，保留证据并停止。不能为完成安装擅自重启CPA/整栈；需要用户在场确认维护窗口后再处理watcher/重启。

真实管理密钥由受信程序从现有私有配置消费，不写进shell参数、URL或日志；不要使用 `curl -H "Bearer $(cat ...)"`。

## 4. 回滚

- 先确认是本次0.2.2导致的问题，并再次保存当前状态和后续差异。
- 将**哈希仍为本次候选**的新库移出扫描目录，放到本次root-only备份中；保留旧0.2.1库。
- 只还原本次PhotoPainter配置标量变化，且确认该标量仍是本次写入的值。其他配置/凭据有后续修改时全部保留。
- 同样保留配置inode，触发热加载；认证列表必须重新选中原版本，原页面摘要恢复，所有其他服务保持不变。
- 如果当前配置文件完整SHA仍等于本次“after SHA”，才可把备份原字节写回**同一inode**；否则不能整文件恢复。
- 不恢复数据库、不清空账户、不换凭据、不重启整栈；未确认图片事务不因回滚成为“已取消”。

不能安全自动化的部分：未知的当前配置结构/后来变更、是否有在途模型请求、失效watcher恢复、真实业务页面和设备结果的判断。现有仓库没有可靠处理这些情况的部署器，因此应保留人工维护窗口和逐项判据，不能生成一个无条件全配置覆盖/自动重启的脚本冒充闭环。

这些步骤没有安排任何自动重启。初次安装若未启用插件，须先按服务器当前规范仅增加 PhotoPainter 对应配置，保留原 inode；当前已有 0.2.1 的部署走上面的升级分支。线上步骤与真实硬件验收都留到用户在场。
