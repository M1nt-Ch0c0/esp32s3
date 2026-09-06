# PhotoPainter 仓库职责与架构

当前系统由四个仓库组成：`esp32s3` 提供索引和总览，三个实现仓库分别负责电脑端应用、设备框架和设备显示模块。显示模块独立打包、独立更新；它通过约定的 ABI 在设备框架内运行。

## 1. 仓库分别做什么

| 仓库 | 运行位置 | 主要职责 | 主要产物 |
|---|---|---|---|
| [esp32s3](https://github.com/M1nt-Ch0c0/esp32s3) | 文档与开发入口 | 仓库索引、整体架构、部署文档导航 | Markdown 文档，不参与构建或设备运行 |
| [ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame) | 本地电脑，当前部署为 macOS | 从远端采集额度和用量；用 Chrome 排版并生成六色 PNG；调度、去重、排队和主动推送；提供常驻服务及 SSH 隧道安装器 | Go 程序、800×480 PNG、macOS LaunchAgent 和本地启动器 |
| [photopainter-host](https://github.com/M1nt-Ch0c0/photopainter-host) | ESP32-S3 | 启动、NVS 配网、Wi-Fi、HTTP 鉴权、请求缓冲；ELF 包校验、A/B 槽管理、加载、确认与回退；维护固定宿主 ABI | 框架固件 `photopainter_host.bin`；电脑端管理工具 `tools/module.py` |
| [photoframe](https://github.com/M1nt-Ch0c0/photoframe) | ESP32-S3，由框架加载 | 完整 PNG 解码与六色校验；像素旋转/打包；AXP2101 电源、GPIO/SPI、Spectra 6 E6 刷新与关断 | 部署使用 `photoframe.app.elf`；另产出 `photoframe.so`，当前宿主不加载它 |

额度接口、页面布局和刷新计划属于 `ai-quota-frame`；网络接入及模块生命周期属于 `photopainter-host`；图像解码和真实面板操作属于 `photoframe`。因此“更换额度页面”和“升级显示驱动”是两条不同的更新路径。

## 2. 运行时架构图

```mermaid
flowchart TB
    subgraph Remote[远端服务器]
        API["CLIProxyAPI / CPAMP<br/>额度与用量接口"]
    end
    subgraph PC[本地电脑]
        Tunnel["SSH 隧道<br/>仅监听 loopback"]
        App["ai-quota-frame<br/>采集 → 排版 → 六色 PNG → 调度推送"]
        Admin["tools/module.py<br/>模块打包、上传、激活与回退"]
    end
    subgraph Device[ESP32-S3 PhotoPainter]
        Host["photopainter-host<br/>Wi-Fi / HTTP / 鉴权 / ELF 生命周期"]
        Slots["Flash：ELF A、ELF B<br/>独立 NVS 状态日志"]
        Module["photoframe.app.elf<br/>当前选中的显示模块"]
        Panel["AXP2101 ALDO4 / SPI<br/>Spectra 6 面板"]
    end
    API -->|接口响应经隧道传回| Tunnel
    Tunnel --> App
    App -->|"POST /api/push：PNG + Bearer"| Host
    Admin -.->|"/api/module：包与管理命令"| Host
    Host -->|校验、写入、选择和恢复| Slots
    Slots -.->|择一读取并重定位| Module
    Host -->|通过固定 ABI 调用| Module
    Module -->|完整验图后才操作硬件| Panel
    Module -->|最终关断等待完成后报告结果| Host
    Host -->|刷屏成功才返回 HTTP 200| App
```

实线描述取数、推图和显示路径；虚线描述模块管理与加载路径。图中远端到本机的箭头表示数据返回方向，连接由本机主动发起。`esp32s3` 是文档索引，不在运行链路内。

当前 macOS 部署有两个独立常驻任务：一个维护 SSH 隧道，一个运行采集/生图/推送程序。设备只接收 PNG 和模块包，不连接远端额度接口，也不需要远端管理密钥。

## 3. 耦合关系

### 构建与发布：三个实现仓库独立

```mermaid
flowchart LR
    PCSource["ai-quota-frame 源码"] --> Go["Go 构建"] --> PCBinary["电脑端程序"]
    HostSource["photopainter-host 源码"] --> HostBuild["固定版本 ESP-IDF"] --> Firmware["框架固件 .bin"]
    ModuleSource["photoframe 源码"] --> ModuleBuild["同一固定版本 ESP-IDF"] --> ELF["业务 .app.elf"]
    ELF --> Pack["tools/module.py package"] --> Package["带 ABI、版本号及 SHA-256 的模块包"]
```

框架构建不会调用 `photoframe` 的构建，也不把业务 ELF 嵌入固件。三个仓库放在同级目录只是开发便利，不是框架编译的前提。第一次完整部署需要同时准备框架与种子模块；之后 ABI 兼容的模块更新通过网络写入备用槽，无需重刷框架。

`esp-idf` 是工具链和 SDK 依赖，不是第五个业务仓库。设备端两仓固定使用提交 `5e6f53cdb31fe5708eae3f55af9737be2822db22`，并使用未修改的 Registry `elf_loader` 1.3.3。电脑端运行还需要 Chrome/Chromium；远端部署通过 SSH 隧道访问接口。

### 运行与兼容：依赖明确的接口契约

| 两端 | 耦合点 | 兼容要求 |
|---|---|---|
| 远端接口 ↔ `ai-quota-frame` | API 地址、鉴权、返回数据结构 | 接口变化时调整电脑端采集器 |
| `ai-quota-frame` ↔ `photopainter-host` | HTTP、独立 Bearer 令牌、PNG 输入和响应状态 | 原始 `image/png`，最大 5 MiB；只有推图 200 表示实体刷新及最终关断等待完成 |
| `ai-quota-frame` ↔ `photoframe` | 经宿主传递的图像格式 | 800×480、非隔行、完全不透明、精确六色；格式变化需同时调整生成端和解码端 |
| `photopainter-host` ↔ `photoframe` | ELF 格式、ABI 版本、导入符号、结果码和调用生命周期 | 当前 ABI 1；模块所需符号必须由宿主提供；每次调用必须报告结果 |
| `photoframe` ↔ 硬件 | GPIO、I²C/SPI、电源与 E6 时序 | 对应当前 PhotoPainter/Spectra 6 硬件，屏幕电源为 ALDO4 |

宿主与模块之间有三个 PNG/结果桥接函数，以及内存、FreeRTOS、GPIO、I²C、SPI 等系统导出。固定导出表由 `photopainter-host/main/host_abi.c` 维护；新增宿主尚未提供的导入不能仅更新业务 ELF。

构建和更新已经解耦，执行环境仍然耦合：业务 ELF 与宿主共享同一设备的内存、任务和硬件资源，**不是独立进程或安全沙箱**。模块出现崩溃可能导致设备重启。A/B 负责候选版本生命周期与重启恢复，不能隔离任意原生代码故障。推图和模块操作共用锁，避免模块执行时被切换。

## 4. A/B 更新与回退

```mermaid
flowchart TD
    Baseline["运行已确认模块"] --> Stage["校验新包并写入备用槽<br/>保留当前已确认槽"]
    Stage --> Trial["显式激活候选<br/>先持久化 trial 标记，再加载"]
    Trial --> Good["有效图片完成实体刷新<br/>最终 POWER_OFF 等待成功"]
    Good --> Confirm["确认新版本<br/>原活动槽成为 previous"]
    Trial --> Bad["加载失败、运行错误<br/>或确认前设备重启"]
    Bad --> Restore["恢复已确认模块<br/>清除候选状态"]
    Trial --> Invalid["输入 PNG 被拒绝"]
    Invalid --> Remain["不确认候选<br/>等待下一次有效输入或回退"]
```

两个槽属于同一个框架固件，不是两份整机固件，也不会并行运行两个显示模块。只有一个槽被选中执行。新上传会占用备用槽，因此旧 `previous` 版本可能被覆盖；当前已确认版本仍保留。首次种子 A 是初始化基线，必须做真实刷屏验证后才能作为可用回退依据。手动 `rollback` 可取消待激活/试运行候选，或切回上一已确认版本。

## 5. 修改某项功能时，需要动哪些仓库

| 想改什么 | 主要修改位置 | 部署影响 |
|---|---|---|
| 额度来源、账号展示、页面布局、刷新间隔 | `ai-quota-frame` | 更新电脑端程序或配置，不改设备代码 |
| PNG 解码、像素处理、面板时序、电源设置 | `photoframe` | ABI 兼容时只更新模块包 |
| Wi-Fi、鉴权、HTTP、槽位日志、加载与回退策略 | `photopainter-host` | 更新框架固件，保留现有模块和状态分区 |
| 新增宿主导出、修改桥接函数签名或结果码 | `photopainter-host` + `photoframe` | 联合设计兼容方案并验证；旧模块可能不再兼容，不能直接套用普通模块更新流程 |
| 改分辨率、六色约束或 PNG 契约 | `ai-quota-frame` + `photoframe`，按需调整宿主限制 | 同时验证生成、协议校验与解码路径 |
| 改说明、导航和架构图 | `esp32s3` 及对应仓库文档 | 无运行时影响 |

## 6. 配置边界与详细文档

Wi-Fi 凭据与推图令牌保存在设备 NVS；远端管理凭据只保存在电脑端受保护的本地配置中。模块仓库不保存任何凭据。模块管理与推图共用设备端鉴权和端口，管理响应 200 不等于屏幕刷新成功。源码仓库不包含真实密钥、NVS 镜像、Flash 备份或含凭据日志。

- [首次迁移、包格式、槽位状态与命令](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs-module-slots.md)
- [macOS 常驻部署与远端隧道](https://github.com/M1nt-Ch0c0/ai-quota-frame#macos-常驻运行)
- [组件接口、输入限制与硬件依据](https://github.com/M1nt-Ch0c0/photoframe)
- [实机验证记录](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/docs/validation-2026-09-06.md)
- [开发流程与约束](https://github.com/M1nt-Ch0c0/photopainter-host/blob/main/.agents/skills/develop-photopainter-stack/SKILL.md)
