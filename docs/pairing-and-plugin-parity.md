# 完整画面复用与物理配对交付记录

2026-10-29。用户要求插件完整复刻 ai-quota-frame，去掉固定推图令牌，以浏览器发现、六位码浮窗、设备按键批准和持久关系连接；功能键可以重置关系。**尚未实机验证，整体目标仍需早上验收。**

## 夜间任务完成范围

最新用户指令将实机验收和全部线上修改留到早上。今晚完成剩余软件、审阅修复、测试/构建、文档与各仓 main 推送；未刷写、连接 BLE/串口、推图、改 CPA/CPM 或重启容器，也没有等待用户回复。

[早上验收清单](MORNING-ACCEPTANCE.md)是操作入口，包含准确提交、最终候选、每步预期、失败回滚及自动推图安排。microSD 条件未确认时默认不刷写；不假设用户豁免。浏览器为默认最终 owner，原生客户端另作换绑验证。

## 已实现

- 原生 Go collector、usage、聚合、全部五主题、原 HTML/CSS/图标/皮肤、六色量化、PNG 和内容指纹通过 WASM 复用。默认 mint-skin、CLAUDE/CODEX/GROK、Asia/Shanghai、15 分钟。
- xAI subject 经 CPA 既有管理鉴权的最小投影接口提供，浏览器不下载完整 OAuth 文件；管理密钥只在当前页面内存中，不交给设备。
- 界面使用实际 Claude Code `claude-opus-5-5` / xhigh 提供的设计，接入发现、配对浮窗、六位码、持久关系和图像事务。设备令牌输入及常驻协议免责声明已删除。
- host 实现 P-256 双方 commitment/reveal、HKDF、六位码、显示完成后 GPIO4 短按批准、NVS 持久唯一 owner；客户端先保存候选关系再 READY。
- GPIO4 按住至少 8 秒并松开撤权。刷新期间可锁存：立即拒绝新工作，当前物理刷新及 borrowed job 完成、旧工作 drain/exact ACK 后才清除关系。保留 Wi-Fi/APP。
- ABI 2.2/PFI1 不变，无需为配对重装 photoframe v20。BLE token op1 拒绝，旧 HTTP 控制入口 410，不能用旧令牌绕过按键重置。
- 原生候选文件有进程间锁和身份复验，迟到确认不能复活已忘记关系。浏览器 CryptoKey 不可导出、strict IndexedDB、缺失 Web Locks 时失败关闭；未知图片只查询原请求，不自动再次 COMMIT。

## 审阅发现的问题与修复

pairing_audit 对撤权饥饿、旧请求跨 reset、副作用 admission 竞态和损坏 NVS 被误认为初次使用的阻断项已关闭；生产 C 状态机/manager 回归覆盖这些交错。原生存储跨进程竞争、pre-COMMIT 鉴权失败的外层推送重试、Web Locks 缺失和平台数量 1–5 边界也已修复。

## 离线证据

- host 90/90、0 跳过，包含真实 v20 ELF、临时 NVS 生成、生产配对/HTTP/manager 策略；固定 IDF 构建通过。
- Go make check、迁移 3/3、Windows 交叉构建通过；Web 32/32、UI 71/71、真实本机 C ABI 及 Linux ELF/导出检查通过。
- 五主题×两场景：四主题像素完全相同，mint-skin 各有 3 个字形灰阶阈值差（原灰度差不超过 2/255）；未手工改像素假装完全一致。
- photoframe 3 CTest、4 入口与 3 示例测试通过，app/SO 构建通过；SDK 头和 imports 镜像逐字节一致。
- Console 0.4.1/build 6 编译、签名检查通过，未安装或启动。
- 提交后 host 固件 SHA `014ce22bed5bc97564c9b6d192ede1daa705372cbd32dff9fc12c799c0343dc4`，未刷入。
- GitHub CI 初次遇到下载 Chrome 的 sandbox 权限问题；修正环境初始化并保持全部一致性断言，[后续 Linux CI 40772972073](https://github.com/M1nt-Ch0c0/ai-quota-frame/actions/runs/40772972073) 全项通过，包含 Linux 原生 C ABI/资源验证。

最终产物与 manifest：`../photopainter-local/releases/pairing-v2-night-20261029/`。各 main 提交见清单；不再依赖未提交工作区。固件、安装包、真实数据、凭据及备份均不进 Git。

## 现场与部署边界

夜间任务之前插件已到 0.2.1，备份 `/var/backups/cpa-stack/photopainter-v021-20261028T203014Z`；原生已安装后台 SHA `0a270bee82cb0d4e16724974206d3cdb51f498ff5b16a604e45ee4e14e0300e8`。这些不是夜间 0.2.2/新后台已部署的证据。CPA 8.0.45 watcher 绑定原 inode，后续升级不得原子覆盖配置；不能沿用此前一次性的暂停窗口重启服务。

旧固件完整 Flash 备份已在夜间禁令前成功，16,777,216 字节，SHA `056a2f81a56198fbac3fd9f7ef77fc416d6fcf77d96e748beae17b563602f339`，并与设备 MD5 比对。准确路径和恢复影响见晨间清单。设备没有刷入配对 v2，microSD 状态仍留现场确认。

自动推图在 04:46:05（UTC+8）本地只读快照中为暂停：automatic=false、operation=null、transfer idle、uncertain=false；今晚未切换。历史不确定 CoC 图片不能仅凭上传完成宣布成功。

## 尚待早上验收

- [ ] 插件 Linux CPA 隔离实际加载、线上热升级、资源/权限/其他容器不变。
- [ ] microSD 条件、当日精确端口、完整 Flash 备份及 factory-only 更新。
- [ ] 浏览器发现、真实六位码/功能键批准、刷新页面及重启后持久鉴权。
- [ ] 8 秒长按撤权、刷新中安全锁存、旧 owner 拒绝、新 owner 配对。
- [ ] 原生客户端/Console 安装与配对，浏览器/Go owner 交接。
- [ ] 一次实际推图的认证完成回执及内容/方向/颜色目视确认。
- [ ] 最终 owner 明确后决定自动发送；默认继续暂停。

这些验收未完成前不能宣称整套目标闭环，模拟 UI、测试 adapter 和绿色构建都不能代替实体操作。
