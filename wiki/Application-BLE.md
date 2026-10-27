# 应用 GATT 小消息开发规范

[返回首页](Home.md) · [ABI](ABI-and-Versioning.md) · [宿主协议](BLE-Protocol.md)

应用定义自己的服务与业务协议，宿主拥有无线栈、鉴权、GATT 发布、事件队列及调度。常驻发现和 OTA 服务不属于任何 APP，切出 APP 时不能停止它们。

大块连续数据使用 [ABI 2.2 CoC](Application-CoC.md)。本页的 128 字节限制只属于 ABI 2.1 特征接口，不限制整个蓝牙栈或 CoC。

## 注册与资源

使用 `APP_MANIFEST_BLE` 声明 2.1。在 START 中调用 `app_host_ble_register_v2()`，用 `app_host_ble_set_v2()` 设置初始值，成功后报告 APP_READY。描述和值由宿主复制；不得把 ELF 回调指针交给宿主。

| 项目 | 限制 |
|---|---|
| 每 APP 服务数 | 1，注销后可重新注册 |
| 每服务特征数 | 1–4 |
| 单特征值 | 1–128 字节上限；实际值可为空 |
| properties | READ、WRITE；NOTIFY 必须同时 READ |
| UUID | 自定义非零 128-bit，描述按常规打印顺序保存 |
| 保留命名空间 | 宿主 `7b9d` 前缀与 Bluetooth SIG UUID；禁止冲突和重复 |
| 待处理事件 | 最多 8 项；客户端必须应对忙状态 |

注册返回的是描述校验/复制结果；实际发布异步进行。遇到发布失败将收到 APP_BLE_ERROR。不要假定 `register` 返回后 OS 已完成服务发现或缓存刷新。

## 事件处理

`APP_EVENT_BLE` 的 `context.data` 指向 `app_ble_event_v2_t`，先检查指针和 `data_size == sizeof(app_ble_event_v2_t)`，再读 type/characteristic/data_size/data。值的字节数还必须符合本应用协议；文本数据未承诺以 NUL 结束。

| type | 含义 | 建议处理 |
|---|---|---|
| APP_BLE_CONNECTED | 宿主已认证的会话 | 初始化连接相关状态；不把它当无认证射频连接 |
| APP_BLE_DISCONNECTED | 认证会话结束 | 丢弃未提交的连接私有数据，按协议保留必要结果 |
| APP_BLE_WRITTEN | 某索引收到复制后的数据 | 解析一条有边界的命令，更新可读状态 |
| APP_BLE_ERROR | 异步服务发布错误 | 清理/注销并报告故障，不能继续声称可用 |

连接边界会清除尚未交付的旧写入并合并边界事件，不能依赖每次射频变化都精确计数。已开始的应用事件不会因为断开被抢占。所有 APP 事件均由 manager 串行执行。

## 认证读写

推荐通过宿主常驻通道的 op 13/14 访问应用特征。客户端先鉴权，从状态取得当前 APP、generation 和 service_revision，再提交索引和数据。旧代次、旧服务版本及切出后的请求被拒绝。

**应用特征的直接 write 也必须使用宿主完整的分片签名帧，不能写裸文本。**原始 GATT read/notify 不带应用层 HMAC，严格校验响应的客户端应使用签名 op 13 读值。通知只是提示：可能被 MTU 长度截断，不能用未验签通知作为显示完成凭据。

现有 echo 的开发客户端（令牌通过可信启动器加载）：

```sh
ai-quota-frame ble app-write --app ble-echo --characteristic 0 --text hello
ai-quota-frame ble app-read --app ble-echo --characteristic 0
```

省略设备 ID 的前提是已有环境配置或绑定。Mac 需通过其已授权的应用身份执行 CLI，见 [维护页](Operations.md)。

## 业务协议必须写清楚

每个应用协议至少定义版本、字节序、命令/响应编码、长度边界、错误码和最大资源。不要将 128 字节值上限误认为整幅图片上限；旧 GATT 路径需要 APP 分块；当前大数据使用 CoC，不能把小消息接口的逐次往返当作高吞吐通道。

可恢复的业务拒绝和宿主事件执行失败要分开。例如坏分块可以由 APP 发布有请求 ID 的错误状态，处理事件本身按契约完成；APP_ERR_RUNTIME 留给运行约定损坏，不能拿它表示普通“忙”。具体结果映射必须在该 APP 的协议里明确。

## STOP 与回执

APP 在 STOP 中释放自有缓冲、取消传输，可显式注销服务；宿主在卸载前取消受管服务和未交付事件。旧 ELF 卸载后不得留下指针。物理 GATT 数据库异步变更期间，旧代次请求已经失效。

如果业务会显示，回执必须在 `app_host_display_v2()` 返回 APP_OK 后才进入完成状态，并允许同一 APP 生命周期内重连查询。切出或重启可能丢失回执，协议必须明确结果不确定如何处理；不宣称永久 exactly-once。

可运行示例：[ble-echo 源码](https://github.com/M1nt-Ch0c0/photoframe/blob/d2dd482/examples/ble-echo/main/entry.c)。限制与实现：[app_ble.c](https://github.com/M1nt-Ch0c0/photopainter-host/blob/b4709c0/main/app_ble.c)、[GATT 传输](https://github.com/M1nt-Ch0c0/photopainter-host/blob/b4709c0/main/ble_transport.c)。
