# 宿主服务参考

[返回首页](Home.md)

服务声明以[版本化 SDK](https://github.com/M1nt-Ch0c0/photopainter-host/blob/b4709c0/sdk/photopainter_app.h)为准。应用必须在宿主调用的有效事件内使用服务；它们不是任意线程都可调用的系统 API。

## 服务与资源上限

| 服务 | ABI | 关键规则 |
|---|---|---|
| `app_host_context_v2()` | 2.0+ | 本次入口 context，事件外无有效指针 |
| `app_host_complete_v2(result)` | 2.0+ | 恰好一次，匹配事件 ID/代次；可用 app_complete 辅助 |
| `app_host_display_v2(pixels, bytes)` | 2.0+ | 必须 384000 字节、0–5 逻辑像素；每事件最多一次；STOP 禁止 |
| `app_host_cache_put_v2(version, data, bytes)` | 2.0+ | 复制内容，单 APP 最多 512 KiB、总计 1 MiB；可失败 |
| `app_host_cache_get_v2(version, &bytes)` | 2.0+ | 按 APP ID 和内容版本隔离；借用到事件结束 |
| `app_host_timer_v2(id, interval_ms)` | 2.0+ | 每 APP 一个；非零最小 1000 ms；0 取消 |
| `app_host_monotonic_ms_v2()` | 2.0+ | 单调毫秒，用于本次开机的时限 |
| `app_host_wall_time_v2()` | 2.0+ | Unix 秒；未合理设置时为 0，不承诺掉电准确时间 |
| `app_host_ble_register_v2(service)` | 2.1 | 复制描述；每 APP 一个服务、最多四特征 |
| `app_host_ble_set_v2(index, data, bytes)` | 2.1 | 复制值，每值最多 128 字节，不保证通知被客户端消费 |
| `app_host_ble_unregister_v2()` | 2.1 | 撤销注册和事件；STOP 中可显式调用 |

完整 BLE 注册限制、事件处理和认证读写见 [应用 BLE](Application-BLE.md)。当前网络服务只通过 context 提供连接状态；没有 APP Wi-Fi socket、HTTP client、文件系统、SD 读写或凭据读取接口。

## 显示

APP 按从左上到右下的行顺序填充六色逻辑像素，宿主检查全部像素，再旋转打包和驱动板级显示。返回 APP_OK 表示硬件刷新和最终 POWER_OFF 等待成功；APP 自己仍须报告本事件结果。参见 [显示契约](Display-and-Images.md)。

## RAM 缓存与借用

缓存是可丢弃的宿主 RAM，不是持久化。APP 以自己的内容格式版本标识缓存；版本不匹配视为无缓存。其他应用的旧缓存可能被回收，设备重启后消失。

`cache_get` 返回借用指针，不可 free，也不可在下一事件继续使用。该事件一旦借用缓存，不能再 `cache_put` 替换缓存。`cache_put` 复制内容，调用返回后 APP 可释放自己的输入。bytes=0 用于清除自身缓存。

不要拿 512 KiB 帧缓存接口保存 5 MiB PNG 接收缓冲。相框适合缓存 384000 字节已显示的逻辑帧；缓存写入失败不应把一次已经成功的实体刷新改成“需要重刷”。

## 定时与时间

定时 tick 合并，不保证每个到期点都交付。事件较慢时根据单调时间计算实际经过时间，不用 tick 次数冒充真实时间。重设定时器替换前一设置；切出自动取消。STOP 允许 interval=0，禁止新设非零定时器。

墙钟可能未同步；离线时钟 APP 要明确显示“时间未设置”等状态。当前没有承诺 BLE 授时协议，不能把单调毫秒当作 Unix 时间。

## 堆内存与结果码

允许导入 malloc/calloc/realloc/free 等 SDK 列表中的函数，但没有独立 APP 内存配额或强制隔离。当前宿主配置使较大普通堆分配可使用 PSRAM；这不是“任意大小必定成功”或独立 PSRAM 分配 ABI 的保证。

每次检查分配失败，先验证长度再分配，STOP/错误/超时释放，保留显示解码峰值余量。不要为获得某个 IDF 函数而绕过导入白名单。系统内部函数即使符号存在，也不等于 APP 被授权导入。

| 值 | 结果 |
|---:|---|
| 0 | APP_OK |
| -1 | APP_ERR_ARGUMENT |
| -2 | APP_ERR_INPUT |
| -3 | APP_ERR_DIMENSIONS |
| -4 | APP_ERR_COLOR |
| -5 | APP_ERR_MEMORY |
| -6 | APP_ERR_DISPLAY |
| -7 | APP_ERR_TIMEOUT |
| -8 | APP_ERR_RUNTIME |
| -9 | APP_ERR_UNSUPPORTED |

这些是应用事件/服务结果，不等于 HTTP 状态或 BLE 传输错误码。业务协议应定义自己的错误映射，区分可恢复的坏请求与运行契约损坏。

依据：[服务实现](https://github.com/M1nt-Ch0c0/photopainter-host/blob/b4709c0/main/app_services.c)、[宿主配置](https://github.com/M1nt-Ch0c0/photopainter-host/blob/b4709c0/sdkconfig.defaults)。
