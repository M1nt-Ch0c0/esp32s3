# CPM Web Bluetooth 相框插件

[返回首页](Home.md)

更新日期：2026-10-29。浏览器客户端和 CPA 原生资源插件维护于
`ai-quota-frame/web/` 与 `ai-quota-frame/plugins/photopainter/`。具体构建、
安装和本轮验证范围见对应仓库的 README；当前设备与后台状态见
[实测证据](Status-and-Evidence.md)。

## 当前实现与历史部署

用户已否定 0.1.0 的简化画面和手工设备令牌流程。当前页面直接复用
ai-quota-frame 的 Go 采集、usage、聚合、五种主题、原 HTML/CSS/皮肤、
量化/PNG 和内容指纹，通过共享 Go/WASM 引擎生成画面。默认与原生当前
配置一致：mint-skin、CLAUDE/CODEX/GROK、上海时区、15 分钟。

页面已改为配对浮窗和持久关系，设备令牌输入及常驻 GATT 提示已删除。
双方显示相同六位码后，由 GPIO4 功能键短按批准；长按 8 秒释放重置 owner。
重置请求可在刷新期间锁存，既有物理刷新完成后才清除关系，新工作立即
被拒绝。BOOT/电源键不参与。host 持有关系，APP 仍兼容 ABI 2.2。

**部署边界：** 插件 0.2.1 与早前原生客户端的安装属于夜间任务前的记录；
今晚 0.2.2、最终 host 和新原生客户端仅为离线候选。host 90/90 测试（0 跳过）
及构建通过。microSD 条件、刷写、真实按键、重启持久化、换绑和物理刷图
全部留到早上，**尚未实机验证**，不能把模拟 UI 当成已闭环。

最新使用与安全合同见 ai-quota-frame 的 `web/README.md`、
`plugins/photopainter/README.md` 及 host 的 `docs/ble-pairing-v2.md`。
0.1.0 的固定令牌和简化快照流程已经退役，使用说明以当前客户端 README 为准。


## 夜间发布与早上验收

新增离线候选 0.2.2，修复标签页锁能力缺失时的失败关闭及 1–5 平台限制，并补齐 CI 工具链。它尚未部署到线上；此前 0.2.1 的运行记录仍为历史基线。今晚不访问设备、不改线上服务器，按 [早上验收清单](../docs/MORNING-ACCEPTANCE.md) 再安装候选、刷写和验证。
