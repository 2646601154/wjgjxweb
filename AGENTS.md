# AGENTS.md

## 项目

纯前端 Android 工具箱：通过 WebUSB + Tango（`@yume-chan/adb` 2.6.4）直连手机执行 ADB 命令，xterm.js 提供终端，另带一个 BAT 语法子集的脚本解释器。中文 UI，无后端、无构建、无依赖安装。

- 分离版：`index.html` → `js/app.js`（原生 ES modules）
- 单文件版：`docs/index.html`（内联全部 CSS/JS/vendor，供离线下载与 GitHub Pages 在线体验）
- `server.cjs`：仅本地静态服务器，Node 标准库，监听 `127.0.0.1:8124`

## 运行与验证

```bash
node server.cjs        # http://127.0.0.1:8124 ，PORT 环境变量可改端口
```

- 必须走 HTTP/HTTPS；`file://` 下 ES 模块和 WebUSB 都不可用。`server.cjs` 返回 `Cache-Control: no-store`，改完刷新即可。
- 没有 npm、package.json、lockfile、测试、lint、CI。不要擅自引入构建链或依赖。
- 无自动化验证手段：需要 Chrome/Edge + 开启 USB 调试的真机。改设备端命令后要实机验证，无法验证就明确说明，不要假装测过。
- 设备能力随 ROM/Android 版本差异很大，`app.js` 里大量逻辑是降级与失败兜底，改动时保留这些分支。

## 双版本同步（最容易踩的坑）

`docs/index.html` 是手工维护的 AIO，没有任何生成脚本。改 `index.html`、`css/`、`js/` 后必须手工镜像过去，否则在线版与源码漂移：

- 样式：`css/xterm.css` + `css/styles.css` 内联进 `<style>`
- vendor：`js/vendor/xterm.js`、`addon-fit.js` 以 UMD 内联
- Tango 包：在 `docs/index.html:259` 的 `<script type="importmap">` 中以 `data:text/javascript;base64,...` 内联
- 业务代码：一个 `<script type="module">`，按 `js/core.js` → `js/batch.js` → `js/transport.js` → `js/terminal.js` → `js/app.js` 顺序拼接，并去掉 `import` / `export` 行

同步后至少检查：

```bash
grep -n "const WJ = \|const WJBatch = \|const WJTransport = \|const WJTerminal = " docs/index.html
```

各文件的改动也要在两处核对关键字符串（新增的 toast/错误文案很好用）。

## 代码地图

- `js/core.js`：`WJ` 命名空间 + `Service`。所有 ADB 命令、输出解析、参数校验都在这里。
  - `WJ.VERSION` 在 `js/core.js:7`，`WJ.defaults` 在 `js/core.js:8`，`WJ.settings()` 校验在 `js/core.js:32`
  - 校验/工具：`pkg`、`number`、`integer`、`quote`、`checkedResult`、`zipStore`、`crc32`
  - 注意 `WJ.settings` 会强制 `connectionMethod='usb'`，并丢弃未知/类型不符的键
- `js/transport.js`：`WJTransport`。懒加载 vendor、WebUSB 连接、`Driver`（命令执行 / 终端 / 同步 push / 控制通道）、TCP 与 server 桥接。`manager.requestDevice()` 需要用户手势，新增连接入口时别把它放到失去激活的长异步链后面。
- `js/terminal.js`：`WJTerminal.Controller`。xterm 会话生命周期、Ctrl+C 中断、Ctrl+Shift+C 复制、80ms 防抖 resize。
- `js/batch.js`：`WJBatch.Runner`。只解释 BAT 子集，**不执行 cmd.exe**，最终只把 adb 命令交给 `Service.execute`；不支持的 Windows 语法要报行号。
- `js/app.js`：DOM 装配、状态机、`actions` 映射（`js/app.js:78`）、`refreshPage`（`js/app.js:70`）、连接 UI 同步 `syncConnectionUi`（`js/app.js:28`）。

## 常见改动怎么落

- **加按钮/操作**：`index.html` 写 `data-action="foo"`，在 `js/app.js:78` 的 `actions` 里加 `foo`。
- **加页面**：导航按钮加 `data-page="foo"`，加 `<section class="page" id="page-foo">`，在 `WJ.pages` 注册，并在 `refreshPage` 的 `switch` 里加刷新分支。
- **加设置项**：`WJ.defaults`（`js/core.js:8`）加默认值 → `WJ.settings` 加约束 → 页面控件加 `data-setting` → `readSettings` 对应处理。设置、脚本、历史存于 `localStorage`，键前缀 `wj-webadb:`（启动时会把旧 `gj-webadb:` 键迁移过来）；ADB 密钥存于 IndexedDB `Tango`。
- **设备控件禁用**：给元素加 `data-device`，断连时由 `syncConnectionUi` 统一禁用，不用自己写禁用逻辑。
- **版本号**：`WJ.VERSION`、`index.html` 侧栏与关于页、`docs/index.html`，三处一起改。

## 硬约束

- 所有中文文案保持现有口吻；代码是高度压缩的单行风格，改哪行就按原风格写，不做无关格式化/重排。
- 拼 shell 参数必须经过 `WJ.quote` / `WJ.pkg` / `WJ.number` 等校验，不能直接拼接用户输入；命令结果统一用 `WJ.checkedResult` 判断失败。
- `js/vendor/` 是 vendored ESM/UMD，不要手改。升级 Tango/xterm 时同步 `js/vendor/`、`docs/index.html` 内联副本以及 `LICENSE-Tango.txt` / `LICENSE-xterm.txt`。
- 原创代码为 GPL-3.0-only，源文件保留 SPDX 头；提交信息沿用简短中文。
- 不要假设 root；降级、失败提示、确认弹窗都是产品的一部分。
- 原作者是 **晨钟酱Official（CzJam）**，原项目为 `CzJam/gjgjxweb`；本仓库只是改名部署版。界面、文档和元信息不得把本仓库维护者写成原作者，改动时保留署名、原项目链接与 `NOTICE.md`。

## 已知文档偏差

- `README.md` 目录树写的是 `assets/favicon.svg`，实际文件是 `assets/favicon.ico`。
- 项目已从 `gjgjxweb` 改名为 `wjgjxweb`，README/在线体验/项目地址应指向 `2646601154`；作者 `CzJam` 的 GitHub/B站等个人链接不要改。
