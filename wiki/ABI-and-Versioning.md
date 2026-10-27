# ABI 与版本管理

[返回首页](Home.md)

ABI（Application Binary Interface）定义已编译 APP 如何与宿主互相调用：符号名、数据结构布局、入口与事件、参数生命周期、返回结果。这是本项目维护的接口约定。

## 不同的版本分别表示什么

| 名称 | 例子 | 作用 |
|---|---|---|
| 宿主版本 | 源码提交与 bin 摘要 | 确认实际框架代码；工作区构建不能仅看旧 HEAD |
| APP 发布版本 | `photoframe v20` | 同一 APP 不同发布产物的标签 |
| ABI 版本 | 1、2.0、2.1、2.2 | 该 APP 使用的调用规则与所需服务 |
| 包格式 | format 1 / 2 | 包头布局与应用身份封装 |
| BLE 传输版本 | 发现记录 version 1 | 无线帧和鉴权的兼容版本 |
| APP 自身协议版本 | 由 APP 定义 | 例如相框 PFI1 命令和回执格式 |

不能用其中一个数字代替另一个。提高 APP 发布版本不会自动升级 ABI；改变 ABI 宏也不会替你重写旧入口。

## 已支持的 ABI

| ABI | 入口规则 | 显示职责 | 应用 BLE |
|---|---|---|---|
| 1 | 旧 PNG 输入桥接及结果回调 | 旧 APP 保留自己的板级驱动 | 无 ABI 2.1 服务 |
| 2.0 | START / INPUT / TIMER / NETWORK / STOP，显式结果 | APP 生成逻辑帧，宿主驱动硬件 | 不可导入 BLE 服务 |
| 2.1 | 保持 2.0 布局，增加 BLE 事件和三个受管导入 | 同 2.0 | 注册、设置值、注销 |
| 2.2 | 保持旧布局，增加 CoC 就绪事件 | 同 2.0 | 单前台所有权 CoC，非阻塞记录收发 |

新应用选 2.0；需要自定义 BLE 时选 2.1。旧 ABI 1 继续兼容，功能键跳过它，因为它没有自主 START 首屏。

需要 CoC 的 APP 使用 `APP_MANIFEST_COC` 声明 2.2；photoframe v20 已迁移。CoC 导入要求 minor >= 2；所有者自动绑定前台运行代次，断线保留注册、切出释放。不会扩大旧 `APP_BLE_MAX_VALUE` 或改变 140 字节旧 BLE 事件。见 [CoC 接口](Application-CoC.md) 与宿主 `docs/abi-2-2-coc.md`。2026-10-27 已部署并取得真实 CoC 显示回执，范围见状态页。

## 版本在哪里声明

ABI 2 ELF 内必须有唯一、只读、128 字节的 `.app_manifest`；包头记录 ABI 主版本和发布版本，manifest 补充 ABI 次版本、APP 身份、输入能力及首屏要求。

```c
// ABI 2.0：即使使用新版 SDK，这个宏也继续声明 2.0。
APP_MANIFEST("my-app", "My app", 0, 0);

// 另一个应用若使用 BLE，改用此宏；不能同时声明两个 manifest。
APP_MANIFEST_BLE("my-ble-app", "My BLE app", 0, 0);

// 需要 CoC 的应用声明 2.2；每个 ELF 只选一个 manifest。
APP_MANIFEST_COC("my-coc-app", "My CoC app", 0, 0);
```

ID 为 1–31 个小写字母、数字、下划线或连字符；名称为最多 63 字节的可打印 ASCII。包头 ID、manifest ID 和安装目标必须一致。保留字段为 0；未知能力与不支持的次版本拒绝。

ELF32 上 context/result/manifest 大小分别为 44/20/128 字节。应用使用固定宽度字段和 SDK 声明，不能自己复制一份“看起来相同”的结构或更改排列。

## 导入与兼容性校验

`photopainter-host/sdk/abi2_imports.inc` 是 ABI 2 导入允许列表。Python 打包器和 C 校验器都检查 manifest 与导入。GATT 导入要求 minor ≥ 1，CoC 导入要求 minor ≥ 2；旧宿主在执行不兼容代码之前拒绝新包。

SDK 原件在宿主 `sdk/`，应用仓库的 `components/app_sdk/` 保存同内容副本。修改 SDK 时同时同步头文件、导入列表、导出表、两套校验器和相关测试；不要只改一个宏或包头来绕过检查。

## 框架演进规则

1. 优先在已有服务上实现应用业务协议，尤其是相框收图。
2. 新通用能力需要说明旧接口无法完成什么、调用上下文、上限、所有权及切出清理规则。
3. 向后兼容扩展应保留旧布局、旧声明和旧行为，并通过显式新声明选择；不能让旧 APP 意外使用新导入。
4. 不兼容结构或语义变更需要新版本及迁移策略，不能复用旧版本号偷偷修改。
5. 发布检查覆盖 ABI 1、2.0、2.1、2.2、未知版本拒绝及缺少/多余导入等组合；本轮实际执行范围另见状态页。

管理 JSON 顶层 `abi=1` 是历史管理封套字段。当前 APP 的完整 ABI 看 `payload_abi` 与 `payload_abi_minor`（旧宿主可能缺少次版本）；`ready` 只表示已加载，完整就绪看 `runtime_ready` 和 `confirmed`。

实现原件为宿主 `sdk/photopainter_app.h`、`sdk/abi2_imports.inc`、`main/app_manifest.c`、`tools/app_manifest.py`。对应源码发布状态见 [证据页](Status-and-Evidence.md)，不以旧 ABI 2.1 固定提交代替当前接口。
