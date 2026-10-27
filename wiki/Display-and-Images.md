# 显示与图片契约

[返回首页](Home.md)

宿主提供通用的逻辑帧显示服务；PNG 解析、相框业务协议、页面和恢复策略属于 APP。电脑端负责把额度等数据渲染为兼容图片。分层变化不改变硬件安全时序和显示完成的定义。

## ABI 2 的显示输入

`app_host_display_v2()` 接收 800×480、共 384000 字节的行优先逻辑帧。坐标从左上开始，每像素一个索引：

| 索引 | 颜色 | RGB |
|---:|---|---|
| 0 | 黑 | `#000000` |
| 1 | 白 | `#ffffff` |
| 2 | 黄 | `#ffff00` |
| 3 | 红 | `#ff0000` |
| 4 | 蓝 | `#0000ff` |
| 5 | 绿 | `#00ff00` |

全部像素在硬件 I/O 前验证。宿主板级服务完成唯一一次旋转和 E6 颜色打包，APP 不自己倒转像素、不操作 GPIO/SPI/I²C。每事件最多调用一次显示，STOP 禁止显示。

## 相框的 PNG 输入

现有 decoder 接受最大 5 MiB、恰好 800×480、非隔行、所有像素完全不透明且属于上表六色的 PNG。必须完整验证结构、解码、验色和拒绝尾随数据，再开始面板操作。伪 PNG 签名或 SHA 相同但内容无效，都不能通过验证。

解码产生逻辑帧后调用宿主显示；成功后可缓存逻辑帧。缓存可失败，不能把缓存失败变成重刷理由。启动时有匹配 RAM 缓存则恢复，否则 APP 自己显示等待页。新 APP 收图协议应复用该 decoder，不复制一套较弱验证。

## 什么才算完成

物理完成包括 E6 刷新和最终 POWER_OFF 的 BUSY 等待。HTTP 兼容输入只有在此后返回 200；现有宿主 BLE 图片通道也在此后产生鉴权完成结果。当前photoframe的GATT和CoC业务回执均采用同样完成条件。

上传到 100%、SHA 校验通过、包暂存成功、GATT 写入或CoC入队成功、APP 已加载都不是物理显示完成。APP BLE 写入的通用成功也不自动证明绘图；客户端需核对该 APP 协议定义的请求 ID、结果和完成状态。

回复丢失时查询原请求，禁止将“未收到回复”直接当作“面板未刷新”。设备/APP 重启或切出导致凭据丢失时停止自动重刷。实机验收还要人工确认内容、方向和颜色，电子纸断电保留旧图不能证明设备在线。

## 兼容边界

ABI 1 保留 APP 内的旧驱动；ABI 2 使用宿主板级驱动。两条路径由统一调度串行运行，不能并发驱动屏幕。固定引脚、电源与 E6 时序的改变需要框架级审查，不属于调整 APP 收图协议的范围。

相框收图命令和客户端使用方法放在 [APP 文档](https://github.com/M1nt-Ch0c0/photoframe/blob/main/README.md)和[电脑端文档](https://github.com/M1nt-Ch0c0/ai-quota-frame/blob/main/docs/ble.md)。框架 README 不再以推图业务命令作为入门流程。

依据：[decoder](https://github.com/M1nt-Ch0c0/photoframe/blob/d2dd482/components/photoframe/src/png_decode.c)、[默认 APP](https://github.com/M1nt-Ch0c0/photoframe/blob/d2dd482/main/entry.c)、[板级显示](https://github.com/M1nt-Ch0c0/photopainter-host/tree/b4709c0/components/photopainter_board)。
