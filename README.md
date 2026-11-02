# PhotoPainter

PhotoPainter 是一块 7.3 寸六色电子纸相框（微雪 ESP32-S3 PhotoPainter），我们给它写了一套自己的固件和配套软件，用来在相框上显示 AI 编程订阅的额度看板：Claude、Codex、Grok 等账号还剩多少额度、什么时候重置、最近几天用了多少 token。

相框本身不联网取数据。电脑上的后台程序或手机浏览器负责采集数据、生成画面，再通过蓝牙推送到相框。相框固件分成两层：常驻的**宿主**负责蓝牙、配对、存储和应用管理；具体显示什么由可以单独安装和更新的**应用**决定。

这个仓库只放文档。代码在下面几个仓库里：

| 仓库 | 是什么 |
|---|---|
| [photopainter-host](https://github.com/M1nt-Ch0c0/photopainter-host) | 相框上的宿主固件 |
| [photoframe](https://github.com/M1nt-Ch0c0/photoframe) | 相框应用：接收图片并显示 |
| [ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame) | 电脑端程序和 CPM 网页插件：采集额度、生成画面、推送到相框 |
| [photopainter-console](https://github.com/M1nt-Ch0c0/photopainter-console) | macOS 上的设备控制台 |

## 从哪里开始读

| 你想… | 先读 |
|---|---|
| 了解这个项目 | [项目介绍](docs/01-overview.md) |
| 把相框用起来 | [上手使用](docs/02-getting-started.md) |
| 搞懂相框内部怎么运作 | [设备是怎么工作的](docs/03-how-the-device-works.md) |
| 给相框写一个应用 | [编写一个应用](docs/04-writing-an-app.md)，再查[应用接口参考](docs/05-app-api-reference.md) |
| 开发蓝牙客户端或应用通信 | [蓝牙通信](docs/06-bluetooth.md) |
| 用电脑端、控制台或网页插件 | [电脑端、控制台与网页插件](docs/07-clients.md) |
| 刷固件、备份或排查问题 | [维护与排错](docs/08-maintenance.md) |
| 知道现在部署了什么、有哪些已知问题 | [当前状态](docs/09-status.md) |
| 查一个不认识的词 | [术语表](docs/10-glossary.md) |
| 修改这些文档 | [文档写作规范](docs/11-writing-guide.md) |

还没实现、正在讨论的设计放在 [docs/proposals](docs/proposals/)。
