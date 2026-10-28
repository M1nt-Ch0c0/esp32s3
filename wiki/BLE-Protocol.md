# 宿主 BLE 传输与配对

[返回首页](Home.md) · [配对 v2 原件](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/ble-pairing-v2.md)

**当前源码为配对 v2，尚未实机验证。** 宿主负责发现、物理批准、持久 owner、会话鉴权和应用管理；photoframe 的 PFI1 图片业务版本仍为 1，APP ABI 仍为 2.2。

## 发现

NimBLE 只做外设，同一时刻一个连接。常驻服务 UUID 为 `7b9d0001-6c82-4f31-9a48-6f5061696e74`，发现/Challenge/Request/Response 特征前缀依次为 0002/0003/0004/0005，共用后缀。名称为 `PP-XXXX`（稳定 ID 后四位）；新客户端也能识别历史 `PPBLE`。

扫描响应 manufacturer data 使用 company ID `0xffff`，随后为 20 字节记录：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 1 | 协议版本 2；旧 v1 设备不能进入新配对流程 |
| 1 | 1 | Wi-Fi：0 离线、1 连接中、2 IPv4 就绪、3 配置无效 |
| 2 | 6 | 工厂 Wi-Fi MAC 派生的稳定设备 ID |
| 8 | 4 | 网络序 IPv4，未就绪为零 |
| 12 | 8 | SSID SHA-256 前八字节，离线/配置无效为零 |

广播和公开 PairInfo 都不是鉴权证据，不包含协商密钥或 Wi-Fi 密码。

## 物理配对与重连

双方用 P-256、commit/reveal、HKDF 和六位短认证码建立关系。客户端必须先严格持久保存候选，再以 MAC 证明就绪；宿主实际完成配对码显示后，才接受 GPIO4 功能键短按。NVS 保存成功后才报告 bound。首次 namespace 不存在或有效 unbound tombstone 表示未绑定；已有 namespace 中记录缺失/损坏会失败关闭。

关系保存后，BondAuth 用新 Challenge 和双方随机数派生本次会话密钥，不再填写设备令牌。批准回复丢失时使用已保存候选认证恢复。每台相框一个 owner；浏览器和原生程序是不同 owner，同一客户端可记住多台设备。

功能键长按至少八秒并松开会锁存重置请求，立即阻止新请求/新显示；已开始的物理刷新完整结束，worker 归还借用输入和 admission 并确认 drain 后，才清除 owner。旧请求/旧会话不能跨越 reset 执行。重置不删除 Wi-Fi 或已安装 APP；BOOT/电源键不参与。

| op | 操作 |
|---:|---|
| 18 | PairInfo（公开状态） |
| 19 / 20 | PairCommit / PairReveal |
| 21 / 22 | 带证明的 PairPoll / PairCancel |
| 23 | BondAuth |
| 1 | v2 永久拒绝旧固定 token 鉴权 |

精确字节、签名域、长度、超时和测试向量以宿主协议原件为准。本方案是应用层配对，不是 OS SMP bond；现有业务数据保持 HMAC 完整性契约，不宣称链路加密。

## 已鉴权操作与分片

正常签名请求/响应保留原 wire layout 和 `pp-ble-v1/request`、`pp-ble-v1/response` 域名，密钥来源变为 BondAuth。序号从 1 严格递增，不回绕。无签名错误不能建立信任；Response 单字节零只表示 pending。

每次有响应写入使用 `offset:u16le,total:u16le,fragment`，逻辑帧最大 512 字节。一次只允许一个完整请求在途；浏览器使用最大 20 字节的保守写片段。

| op | 操作 |
|---:|---|
| 2 / 3 | 签名发现 / 冻结分页状态 |
| 4–8 | 应用包 BEGIN/CHUNK/COMMIT/ABORT/PROGRESS |
| 9–12 | activate/switch/rollback/remove |
| 13 / 14 | 受管 APP 特征读/写，钉住 generation/revision/index |
| 15–17 | 历史宿主 PNG 兼容操作，不作为新 APP 的业务协议 |

应用包暂存成功不等于激活；上传完成不等于物理刷新。图片恢复必须钉住设备 ID、配对 BID、APP generation/revision 和 request ID；换绑、重启或缺失回执时不能自动重画。CoC 见 [数据通道](Application-CoC.md)，PFI1 见 [APP 原件](https://github.com/M1nt-Ch0c0/photoframe/blob/main/docs/ble-image-input.md)。
