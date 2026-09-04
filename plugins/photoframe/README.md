# 组件：相框

业务：鉴权后的 `POST /api/push`、800×480 PNG/JPEG、走 processed-PNG 快路径刷 Spectra 6。

不包含：额度查询、Chromium 截图、CLIProxyAPI。那些在 [ai-quota-frame](https://github.com/M1nt-Ch0c0/ai-quota-frame)。

尚未实现。编成宿主可 `dlopen` 的模块后，改推图逻辑应只换本目录产物，不重编 IDF。
