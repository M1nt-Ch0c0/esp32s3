# 宿主 BLE 传输与鉴权

[返回首页](Home.md) · [应用 BLE 开发](Application-BLE.md)

这是宿主常驻通道的开发参考。应用自身的业务协议另行版本化；相框 APP 协议已在 Mac 本地实现，正式原件和代码尚未推送；不能把下面的宿主包上传命令照搬成 APP 收图规范。完整字节契约见宿主[协议原件](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/ble-protocol.md)。

## 发现和 GATT

NimBLE 只做外设，支持一个连接。常驻服务 UUID 为 `7b9d0001-6c82-4f31-9a48-6f5061696e74`；四个特征共用相同后缀。

| 前缀 | 特征 | 语义 |
|---|---|---|
| `7b9d0002` | Discovery | 公共只读发现记录 |
| `7b9d0003` | Challenge | 当前连接的新随机 nonce |
| `7b9d0004` | Request | 有响应写入，承载分片的签名命令 |
| `7b9d0005` | Response | 读取完整响应，支持 ATT 长读 |

广播包含服务 UUID；扫描响应的 manufacturer data 使用实验 company ID `0xffff`，之后为 20 字节记录：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 1 | 协议版本 1 |
| 1 | 1 | 0 离线、1 连接中、2 IPv4 就绪、3 Wi-Fi 配置无效 |
| 2 | 6 | 工厂 Wi-Fi MAC 派生的稳定 ID，客户端显示为 12 位十六进制 |
| 8 | 4 | 网络序 IPv4，未就绪为 0 |
| 12 | 8 | SSID 字节 SHA-256 的前 8 字节，离线/配置无效为 0 |

网络状态变化立即更新内存记录并安排广播更新。连接期间 GATT 记录更新，断开后广播最新内容。发现数据不含令牌、密码或 SSID 明文；它未经认证，不能据此信任设备或把 Bearer 发往广播声称的 IP。

## 双向 HMAC

当前无配对、bonding 或链路加密。共享设备令牌为 32–128 个无空白可打印 ASCII 字节，必须使用强随机值，不能复用管理/OAuth/Wi-Fi 密钥。

设 D 为 6 字节设备 ID，S 为服务端 32 字节 nonce，C 为客户端 32 字节 nonce，字符串域为不含 NUL 的 ASCII：

1. 客户端检查 ID，读取 S，生成 C。
2. 发 `01 || C || HMAC(token, "pp-ble-v1/client" || D || S || C)`。
3. 双方计算 `K = HMAC(token, "pp-ble-v1/session" || D || S || C)`。
4. 宿主以 op=1、seq=0 返回签名发现记录，客户端验签后才信任它。

每连接 nonce 只允许一次鉴权尝试，失败后重新连接。后续请求：

```text
op:u8 || sequence:u32 || payload || HMAC(K, "pp-ble-v1/request" || preceding_bytes)
```

响应：

```text
op:u8 || sequence:u32 || code:u16 || payload || HMAC(K, "pp-ble-v1/response" || preceding_bytes)
```

所有整数小端，标签 32 字节。seq 从 1 开始，严格递增且不回绕。未鉴权/格式错误可以收到无签名的 7 字节拒绝，绝不能把它当成功或设备身份证据。响应特征单字节 0 表示工作尚未完成，不是上传成功。

## 分片和常驻操作

一次 GATT write 前置 `offset:u16, total_frame_size:u16`，随后至少一个帧字节。完整逻辑帧最多 512 字节；最小 ATT MTU 23 时也可分片。允许相同片段重复，拒绝空洞、冲突、越界及执行期间的新写入。一次只允许一个未完成命令。

| op | 常驻操作 | 说明 |
|---:|---|---|
| 1 / 2 | 鉴权 / 签名发现 | bootstrap 与已认证记录 |
| 3 | 状态、应用列表 | offset=0 冻结快照，之后分页读取 |
| 4–8 | 包 BEGIN / CHUNK / COMMIT / ABORT / PROGRESS | 写备用槽；COMMIT 成功只表示暂存 |
| 9–12 | activate / switch / rollback / remove | 复用唯一安装和调度状态机 |
| 13 / 14 | 应用特征读 / 写 | 携带 generation、service_revision、index，再附应用数据 |
| 15–17 | 历史宿主 PNG 能力 / 开始 / 结果 | 当前兼容实现，暂停扩展；不作为新 APP 协议 |

APP ID 编码为 32 字节 NUL 填充。包传输额外携带整包 SHA-256，包内仍有 ELF payload SHA 校验。会话支持 60 秒空闲、两小时总寿命；正在运行的同步工作持有数据直到完成。断开后的包传输可以整包重传，不承诺跨连接持久续传。

## 错误和结果

| code | 含义 |
|---:|---|
| 0 | 该命令成功，具体是否完成业务取决于命令契约 |
| 1 / 2 | 未授权或 MAC/seq 无效 / 设备令牌未配置 |
| 3 / 4 / 5 | 忙 / 参数或命令无效 / 偏移冲突 |
| 6 / 7 / 8 | 整包摘要错误 / 内存不足 / 目录、代次或传输状态错误 |
| 9 / 10 / 11 | 包被拒绝 / 存储或应用执行失败 / 会话过期 |
| 12 / 13 / 14 | 历史 PNG 通道：坏图 / 显示失败 / 设备启动代次改变 |

传输错误、APP 结果码和 APP 自身协议错误需分别处理。激活或显示回复丢失时，先查真实状态/原回执，不重新发送有副作用的命令。HMAC 保证鉴权与完整性，图片、包、状态仍可被监听；详见 [安全边界](Security.md)。
