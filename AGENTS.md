# AGENTS.md — DesktopNaotu 项目指南

本文件为 AI 编码代理（以及新加入的开发者）提供本项目的结构、命令与约定说明。请用中文交流；项目源码注释与文档均以中文为主。

## 1. 项目概述

**DesktopNaotu（桌面版脑图）** 是基于百度脑图（kityminder）的 Electron 桌面应用，让用户在无网络环境下使用思维导图工具。核心能力：本地 `.km` 文件的打开/保存/自动保存、拖拽打开、关联打开、多语言（en / zh-CN / zh-TW / de）、偏好设置、日志与备份。

- 仓库：<https://github.com/NaoTu/DesktopNaotu>
- 版本号规则：`Major.Minor.Patch.Build`（见 `app/src/version.ts`）
- 许可证：以根目录 `LICENSE` 文件（GPL-2.0）为准；`package.json` 中写的 "MIT" 是历史遗留不一致，勿照抄。

## 2. 技术栈

| 层 | 技术 |
| --- | --- |
| 运行时 | Electron ^22.3.27（**最后一个支持 Win7/8.1 的版本线**；`nodeIntegration: true`、`contextIsolation: false`、`sandbox: false`；remote 能力走 `@electron/remote` 2.x，见 5.1） |
| 语言 | TypeScript ~4.9，`strict: true`，`noImplicitAny: true` |
| 构建系统 | Gulp 3.9.1（`gulpfile.js`）+ `gulp-tsc` 编译 TS 到 `dist/`（gulp-tsc 实测兼容 TS 4.9 编译） |
| 前端依赖 | **Bower**（非 npm）：AngularJS ~1.4.7、kityminder-editor / kityminder-core（NaoTu 的 fork，git 源）、Bootstrap、CodeMirror、jQuery、bootbox、marked、kity |
| 日志 | winston 3（`app/src/core/logger.ts`，单例；需锁定 `winston@3.3.3 + logform@2.2.0 + @dabh/diagnostics@2.0.2`，新版依赖链在旧 Node/Chromium 下崩溃） |
| 打包 | electron-packager ^17.1.2（`npm run pack*`，`--electron-version=22.3.27` 与 dependencies 保持一致，输出到仓库外的 `../OutApp`）；`build.php` 为服务端按 git tag 自动发布的脚本 |

⚠️ Gulp 3 与 Node 12+ 存在 `ReferenceError: primordials is not defined` 兼容问题，构建前请阅读 README「如何编译」一节（安装 `graceful-fs@4` 补丁，或用 nvm 切换到 Node 10/12）。`@types/node` 需用 16.x（匹配 Electron 22 内置的 Node 16）。

## 3. 目录结构

```
app/
├── src/                    # 全部 TypeScript 源码（编译输出到 dist/）
│   ├── main.ts             # 主进程入口：创建窗口、全局快捷键、close 拦截、ipcMain
│   ├── index.ts            # 渲染进程入口（编译为 dist/index.js，由 index.html 加载）
│   ├── define.ts           # 全局常量：路径、外部 URL、状态类型、语言类型、扩展名
│   ├── version.ts          # 版本号数组 [major, minor, patch, build]（gulp 自动递增）
│   ├── globals.d.ts        # 渲染层全局变量声明：$, jQuery, angular, minder, editor, bootbox, kityminder
│   ├── core/               # 基础服务层
│   │   ├── conf.ts         # naotu.config.json 配置管理（naotuConf 单例，含版本升级迁移）
│   │   ├── i18n.ts         # I18n.__("key") 多语言查找（en 为回退）
│   │   ├── logger.ts       # winston 日志单例（可按配置写入 userData/logs/）
│   │   ├── io.ts           # 文件读写（readText/writeText/readJson/writeJson）
│   │   ├── path.ts         # userData 目录推导：naotu.config.json、/logs、/backup
│   │   └── exec.ts         # 子进程执行辅助
│   ├── lib/                # 业务逻辑层
│   │   ├── base.ts         # naotuBase 状态机单例：当前文件路径、状态(none/opening/closing/saving)、已存/已改计数
│   │   ├── menu.ts         # 渲染进程应用菜单（naotuMenu.render()）
│   │   ├── file.ts         # openKm/saveKm/拖拽打开
│   │   ├── dialog.ts       # 打开/保存/另存为/导出对话框
│   │   ├── window.ts       # 新建/关闭/克隆文件、窗口最大化最小化
│   │   ├── exit.ts         # monitorExitRequest：退出前未保存确认
│   │   ├── recently.ts     # 最近文件记录与自动保存
│   │   ├── minder.ts       # 全局 minder（kityminder 实例）的 get/set 封装
│   │   ├── help.ts         # 关于/许可证/检查更新/退出
│   │   ├── electron.ts     # 主进程对话框的 remote 封装（showFileName 等）
│   │   └── edit.ts         # undo/redo
│   ├── plugin/
│   │   └── plugin-manager.ts  # 插件管理器（骨架，runPlugins 多为 TODO）
│   └── ui/
│       ├── pref_dialog.ts + pref_dialog.seg.html  # 偏好设置对话框
│       └── shortcut.ts     # 快捷键说明对话框（内嵌 HTML 模板字符串）
├── static/                 # index.html（唯一页面）与应用图标
└── style/                  # main.css
locale/                     # i18n JSON：en / zh_CN / zh_TW / de
dist/                       # gulp 构建产物（gitignore，package.json "main" 指向 dist/main.js）
doc/                        # 开发文档：Dev.md(开发日志)、CompileLog.md、History.md、Help.md、Future.md
screenshot/                 # 各平台截图
build.php                   # 服务端发布脚本（按 git tag 构建多平台包）
gulpfile.js / tsconfig.json / bower.json / .bowerrc
.vscode/                    # launch.json：Debug Main Process（附加渲染进程 9222 端口）
```

## 4. 常用命令

```bash
# —— 环境搭建 ——
npm install -g gulp bower       # 全局工具
npm install                     # electron-installer-debian 已入 optionalDependencies，Windows 无需 --force
bower install                   # git 源已改用 https://...git 形式，无需 git config 协议重写
npm install @types/node@16.x    # 匹配 Electron 22 内置的 Node 16

# —— 构建 & 运行 ——
gulp                            # 等价 npm start：clean → 递增版本号 → 编译 TS → 拷贝 css/html/locale/vendor
npm run demo                    # electron . 运行应用

# —— 调试 ——
# VSCode 中 F5（launch.json: Debug Main Process），渲染进程走 Ctrl+Shift+D 切换 DevTools

# —— 打包（输出到 ../OutApp，不在仓库内）——
npm run packwin32               # win32 ia32
npm run packwin64               # win32 x64
npm run packmacos               # darwin x64
npm run packlinux               # linux x64
npm run packlinuxarm            # linux arm64

# —— 发布自动化（服务端）——
php build.php                   # 按 git tag 逐版本构建并归档到下载目录
```

没有测试框架、没有 lint 配置文件（仅 gulpfile 内联 jshint，且 `copy-js` 任务已被注释）；验证手段是 `gulp` 构建通过 + `npm run demo` 手工验证。

## 5. 架构要点

### 5.1 双进程与 IPC 协议

- **主进程** `app/src/main.ts`（编译为 `dist/main.js`，即 package.json 的 `main`）：创建 BrowserWindow（最小 700×700，记忆上窗口尺寸）、注册 `Ctrl+Shift+D` 切换 DevTools。GPU 硬件加速保持默认开启（Chromium 108 合成栈正常）；若个别老显卡/驱动机器出现整窗黑屏或灰蒙蒙，可在 app ready 前加 `app.disableHardwareAcceleration()` 规避（Chromium 66 时代的已知问题，内核升级后已移除该补丁）。
- **remote 的正确用法（Electron 14+ 约定）**：主进程 `main.ts` 里 `remoteInitialize()` 一次 + 每个窗口 `remoteEnable(webContents)`；渲染进程统一 `import * as remote from "@electron/remote"`（**不能**再从 `"electron"` import remote，也**不能在主进程 require `"@electron/remote"`**——它会直接抛错）。
- **主/渲染共用模块的约束**：`core/conf.ts`、`core/path.ts` 两端都会加载，其中的 remote 访问必须按进程惰性 `require("@electron/remote")`（见两文件内的 `getApp()`/`getAppInstance()`），不能静态 import。
- **渲染进程** `app/src/index.ts`（编译为 `dist/index.js`，由 `app/static/index.html` 加载）：初始化 AngularJS 模块 `kityminderDemo`（依赖 `kityminderEditor`）、渲染菜单、拖拽打开、退出监听。
- **IPC 消息**（字符串约定，改名需两端同步）：
  - 主 → 渲染：`mainWindow.webContents.send("action", "exit")` —— 窗口 close 时触发。
  - 渲染 → 主：`ipcRenderer.sendSync("reqaction", "exit")` —— 确认退出后真正退出。
- **退出流程**：用户点关闭 → 主进程保存窗口尺寸、发送 `action/exit` 并 `preventDefault` → 渲染进程 `exit.ts` 检查 `naotuBase.HasSaved()`，未保存则 bootbox 确认 → 回发 `reqaction/exit` → 主进程 `safeExit = true; app.quit()`。

### 5.2 渲染层运行时

`index.html` 依序加载 `js/jquery.min.js`、`js/vendor.js`（gulp 把所有 bower 库 concat+uglify 成单文件）、`index.js`。AngularJS + kityminder-editor 提供 `<kityminder-editor>` 标签；脑图实例通过全局变量 `minder`（见 `globals.d.ts`）访问，`lib/minder.ts` 提供 get/set 封装。`index.ts` 中 `minder.on("contentchange")` 驱动自动保存与脏标记。

### 5.3 状态与配置

- `naotuBase`（`lib/base.ts`）：单例状态机。当前文件路径（set 时自动写入最近记录）、操作状态 `none | opening | closing | saving`、`_changedNum/_savedNum` 计数判断是否已保存。**打开/保存文件前后必须正确 setState，否则会误触发自动保存**。
- `naotuConf`（`core/conf.ts`）：`userData/naotu.config.json`。`upgrade()` 在启动时做字段级迁移（旧版本号 → 新模板）。扩展配置时：在 `NaotuConfig` 类加字段 → `getTemplate()` 加默认值 → `Deserialization()`/`upgrade()` 加迁移 → 递增 `define.ts` 的 `sConfigVersion`。
- 用户数据目录（`core/path.ts`）：`userData/naotu.config.json`、`userData/logs/`、`userData/backup/`。

### 5.4 版本号自动递增（重要）

`gulpfile.js` 的 `build-version` 任务在**每次 `gulp` 构建**时：读取 `app/src/version.ts` → 第 4 位（build）+1 → 重写 `version.ts`，并把前 3 位同步写回 `package.json` 的 `version`。

- **不要手动改 `package.json` 的 version**；发版只改 `app/src/version.ts` 的前 3 位，build 位交给构建。
- `define.ts` 的 `sAboutText`（关于对话框）在构建时通过 import 拿到新版本。

### 5.5 国际化（i18n）

- 文案 key 存于 `locale/{en,zh_CN,zh_TW,de}.json`，代码中用 `I18n.__("key")`；缺失 key 原样返回，`en.json` 是文件级回退。
- key 命名约定：`m*` 顶层菜单、`mi*` 菜单项、`s*` 提示字符串、`d*` 对话框（`dPreferences_*`、`dShortcut_*` 等前缀分组）、`c*` 按钮。
- **新增任何用户可见文案，四个语言文件都要加**（至少 en + zh_CN，其余语言可先复制英文）。
- 语言选择存于配置文件的 `locale` 字段（`Languages = "en" | "zh-CN" | "zh-TW" | "de"`）。

## 6. 代码约定

- **注释/标识符语言**：注释用中文，与现有代码一致；接口/类名英文，语序沿用 C# 风格习惯（如 `OnEdited`、`getXxx/setXxx`）。
- **TypeScript**：`strict` + `noImplicitAny` 开启，不要引入 `any` 逃逸（渲染层与 bower 库交互处例外，已有 `globals.d.ts` 声明）。模块用 commonjs import，不用 ES default import 语法糖。
- **单例模式**：`logger`、`naotuConf`、`naotuBase` 均为「类 + 静态 instance + 导出实例」的单例，新服务遵循同一模式。
- **UI 对话框**：统一用 bootbox（`bootbox.alert/confirm/dialog`）；偏好设置类复杂对话框放 `app/src/ui/`，HTML 片段用 `.seg.html`（由 `copy-html-vendor` 拷到 `dist/ui/`）。
- **Electron API**：渲染进程直接 `import { remote } from "electron"` 使用主进程能力（旧架构，勿引入 contextBridge/preload 改造，除非用户明确要求升级）。
- **不要提交**：`dist/`、`node_modules/`、`bower_components/`、`OutApp/`、`package-lock.json`（已在 .gitignore）。
- **依赖变更**：前端库走 `bower.json`（注意 `resolutions` 锁了 angular ~1.4.7、marked 0.6.2；git 源必须写 `https://...git` 全称——`git://` 已被 GitHub 停用，不带 `.git` 的 https 地址会被 bower 当作 tarball 源而无法解析 `#target`）；Node 侧走 `package.json`，锁版本用 `overrides`。

## 7. 常见坑

1. **Node 版本**：gulp 3.9.1 在高版本 Node 上报 `primordials is not defined`。已通过 `package.json` 的 `overrides` 把全树 graceful-fs 锁到 4.x 根治；若清理 node_modules 重装后仍出现，检查 overrides 是否被移除。
2. **构建即升版本**：任何一次 `gulp` 都会改写 `version.ts` 和 `package.json`，产生意外的 git diff——提交前检查，或接受它作为 build 号推进。
3. **`dist/` 才是运行目录**：Electron 加载的是 `dist/`，改了 `app/src` 必须重新 `gulp`（无 watch 编译 TS 的任务；`watch` 任务只做 livereload）。打包脚本 ignore 了 `app/` 与 `src/`，产物只带 `dist/`。
4. **electron 版本已统一**：`dependencies` 与 `pack*` 脚本均为 22.3.27（历史双轨已消除）。再升级注意：Electron 23+ 不支持 Win7/8.1；Electron 14+ 必须用 `@electron/remote`（本项目已迁移）。
5. **IPC 字符串协议**：`"action"/"exit"`、`"reqaction"/"exit"` 是主/渲染进程的隐式契约，重构时全局搜索两端。
6. **全局变量**：渲染层直接使用 `minder`、`bootbox`、`$` 等全局（`globals.d.ts` 声明），新代码不要重复 import 或重新声明。
7. **打包脚本的 `--ignore` 必须锚定**：ignore 是对相对路径做正则测试。曾因无锚点的 `"src"` 把 `node_modules/@electron/remote/dist/src/**` 全部剔除，导致打包版窗口永不创建（无报错、无崩溃日志，极难排查）。正确写法：`--ignore="^/(\.git|screenshot|doc|src|app|bower_components|electron-packager)(/|$)"`——只匹配顶层目录。
8. **`sIndexUrl` 必须用 `pathToFileURL`**：`path.join` 在 Windows 产生反斜杠，与 `file://` 拼接出的混合路径在 asar + 高版本 Chromium 下会挂起（渲染进程无法启动、无任何报错）。已在 `define.ts` 用 `pathToFileURL(join(__dirname, "index.html")).href` 修正，勿改回。
9. **依赖树已用 lock + overrides 固化**：`package-lock.json` 已入库；`overrides` 锁定 `graceful-fs@4`、`logform@2.2.0`、`@dabh/diagnostics@2.0.2`、`colorspace@1.1.2`——新版 winston 依赖链中的 `@so-ric/colorspace` 使用 `Object.hasOwn`，在旧内核（Chromium 93 以下）直接崩溃，勿移除 overrides。
10. **打包版启动慢是正常现象**：Defender 等会对重新打包的未签名 exe 实时扫描，首次启动可能挂起 30~90 秒，不要误判为死锁。启动异常可查 `userData/logs/main-crash.log`（main.ts 的 dumpCrash 同步落盘）。
11. **electron-installer-debian 在 optionalDependencies**：该包声明只支持 darwin/linux，放 devDependencies 会让 Windows 的 `npm install` 直接失败（被迫 `--force`）。入 optional 后 Windows 安装自动跳过，Linux 打 deb 时正常安装。
