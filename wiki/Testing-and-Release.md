# 测试与发布准则

[返回首页](Home.md)

验证按改动影响选择：能在主机重现的解析、校验和状态机先原生测试；构建通过不能替代实机，上传成功不能替代显示完成。只改文档时检查链接、声明和示例即可，不为文档重刷设备。

## 2026-10-27 本轮范围

用户明确要求敏捷开发，不新增测试用例、不运行全面测试。本轮采用编译、设备备份/写入校验、鉴权状态和真实 CoC 推图作为已记录证据。旧测试桩与 ABI 次版本断言尚未同步到 2.2；不得引用旧测试通过记录宣称新代码已全面回归。以下命令是后续恢复系统验证时的参考，不是本轮已执行清单。

## 本地命令

宿主：

```sh
python3 -m unittest discover -s tools -p 'test_*.py'
# 可选真实 ELF 集成，使用已构建的实际路径
PHOTOFRAME_TEST_ELF=/absolute/path/photoframe.app.elf \
  python3 -m unittest discover -s tools -p 'test_*.py'
# 固定 IDF 环境中，源代码/构建配置确有变化时
idf.py build
```

应用：

```sh
cmake -S tests -B tests/build
cmake --build tests/build
ctest --test-dir tests/build --output-on-failure
python3 -m unittest discover -s tests -p 'test_*.py'
python3 -m unittest discover -s examples/color-test -p 'test_*.py'
```

电脑端：

```sh
GOTOOLCHAIN=go1.26.0 make check
# Windows 目标还需交叉构建；Mac 使用原生 CGO 构建 CoreBluetooth
GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./cmd/ai-quota-frame
```

Chrome 缺失会使部分渲染测试跳过；报告时说明，不把跳过写成通过。普通测试不应打开串口、蓝牙或刷新屏幕；硬件测试需要显式 opt-in 和对应授权。

## 按层覆盖

| 层 | 必测内容 |
|---|---|
| ABI / loader | manifest、ID、允许导入、未知版本、旧 APP 兼容、一次结果、错误代次 |
| manager / 目录 | 接纳、忙状态、借用输入寿命、A/B 中断、确认前重启、有界启动回退 |
| APP 生命周期 | START、缓存缺失、STOP 清理、切回重建、事件限时与无后台回调 |
| BLE / CoC | 错误关系、匿名拒绝、承诺/六位码/物理批准、持久化失败、重置撤权、HMAC 篡改/重放、分片、过期、资源释放、旧服务/session 拒绝和有界队列背压 |
| APP 业务协议 | 分块、长度/摘要/CRC、错误映射、断线/超时、重复提交、回执恢复 |
| 显示 | 完整验色在 I/O 前、失败不覆盖缓存、最终 POWER_OFF 后成功 |
| 客户端 | 单 worker、最新快照、取消、去重、节流、结果不确定时不盲目重试、BLE 无 HTTP 回退 |

涉及 ABI/硬件驱动时构建需要维护的 APP/SO 配置；未改驱动的普通 APP 业务改动不应触发宿主固件更新。

## 实机规则

测试前停自动推送和看门狗竞争；准确识别设备。打开 ESP32 原生 USB 串口可能重置设备，不用串口测在线。应用 OTA 先查目录与旧 active，显式暂存和激活。设备无 Wi-Fi 的验收记录真实 BLE IP 状态，不拿电脑联网代替设备离线证明。

显示测试要有可信物理完成结果加人工目测。结果不确定时保留第一份证据，查询原状态；禁止反复发送同一显示请求“试试看”。改变供电可能影响稳定性，历史电脑 USB 复位现象见 [实测证据](Status-and-Evidence.md)。

串口刷宿主是另一个范围：需要授权、16 MiB 完整备份和 SHA-256；有 SD 时按项目要求备份。Wiki 整理不构成新的硬件授权。

## 发布记录

每个仓库在独立分支提交，说明问题、最终行为及验证。记录宿主提交、APP ID/发布版本/ABI、SDK版本、客户端提交、产物 SHA-256 和配置依赖的名称，不记录实际凭据。

接口变更同步 SDK 镜像、导入/导出、Python/C 校验器、协议与测试；APP 业务协议放 APP 仓库。将实际通过、跳过、模拟和未测场景写清楚，更新 Wiki 的能力表及代码链接。发布包和备份分开保存，不能把私有配置塞进构建产物或 Git。
