# Aivory 桌面客户端(Electron)调研文档

> 调研目标仓库:`D:\git_codes\Aivory`,目录 `desktop/`
> 目的:为「后端换成 Python 复刻」评估是否要做桌面端、以及若要做需要满足哪些接口。
> 本文所有引用均为真实源码位置,格式为 `文件:行号`(路径相对仓库根 `D:\git_codes\Aivory\`)。

---

## 1. 结论速览

- **桌面客户端是一个"纯壳"**:Electron 壳只负责本地打包前端(HTML/JS/CSS)+ 把 `/api` 请求代理到远程 Aivory 服务器。**它不承载后端,也不内嵌任何 Node HTTP 服务器**。
- **前端的本地加载方式**:通过 Electron 自定义协议处理器把 `https://app.aivory.invalid/` 这个"假 origin"映射到本地 `web/` 目录(`local-web.cjs:7`),没有 HTTP listener、没有端口(`local-web.cjs:6-8`)。
- **桌面应用本身不发起 AI 推理**:所有推理、用户、订阅、知识库逻辑都在远程 Go 后端。桌面端 = 浏览器 + 代理 + 桌面集成(托盘、通知、更新、支付)。
- **复刻建议**:若目标是个人/SaaS 部署,**先只做 Web 版即可,桌面端完全可延后**。桌面端唯一的硬需求是后端暴露 `/api` HTTP 接口 + 4 个「desktop 专属接口」(浏览器登录 PKCE 流 + 更新清单 + 下载链接),见第 4 节「复刻要点」,这些在 Python 后端里只用约 3 个 handler + 2 个存储配置即可对上。
- 注意:原始后端是 **Go**(`server/internal/api/desktop_*.go`),不是 JS/Node。本文涉及的 server 端接口引用的是 Go 源码。

---

## 2. 桌面客户端架构

### 2.1 进程划分

| 进程 | 职责 | 关键文件 |
|------|------|----------|
| **主进程** (Electron Main) | 生命周期、窗口创建、托盘/菜单、连接状态、更新检查、浏览器登录、支付窗口、API 请求代理、音频桥、系统通知 | `desktop/main.cjs` |
| **预加载** (Preload) | 通过 `contextBridge` 暴露安全 API 给渲染进程,校验来源 | `desktop/preload.cjs`、`desktop/server-preload.cjs`、`desktop/offline-preload.cjs` |
| **渲染进程** (Renderer) | 即 Web 前端(Vite/React,打包进 `web/`),与 Web 版共用同一套前端源码 | `desktop/frontend.mjs`、`src/`(仓库根) |
| **可配置的 Web 前端来源** | 本地 `https://app.aivory.invalid/` 协议壳 或 远程服务器 | `desktop/local-web.cjs` |

文字图:

```
┌────────────────────────── Electron 桌面应用 ───────────────────────────┐
│                                                                        │
│  主进程 main.cjs                                                       │
│   ├─ 创建主窗口 BrowserWindow(加载本地打包前端)                        │
│   ├─ 自定义协议 https://app.aivory.invalid/ ──local-web.cjs──► 本地 web/
│   ├─ /api 请求代理(协议处理器拦截,转发到远程 baseUrl)                  │
│   ├─ ConnectionStatus 连接/离线管理 + offline 覆盖层                    │
│   ├─ UpdateChecker 更新检查(远程 /api/public/desktop-update)           │
│   ├─ BrowserAuth 浏览器登录(PKCE)                                      │
│   ├─ 支付窗口(openPayment)                                             │
│   ├─ ReplyNotifications 系统通知                                       │
│   └─ 托盘 / 菜单 / 单实例锁                                            │
│                    │  preload (contextBridge)                          │
│                    ▼                                                   │
│  渲染进程(打包的 React 前端,web/)  ←── 远程 Aivory 后端 /api  ──► Go Server │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.2 桌面前端与 Web 前端是同一份源码

- 前端构建入口是仓库根的 `vite.config.ts` + `src/`;桌面构建时用 `VITE_AIVORY_DESKTOP=true` 环境变量触发桌面专用行为(`desktop/frontend.mjs:12`)。
- 桌面构建输出到 `dist-desktop/`,再由 `prepare.mjs` 拷进应用包 `generated/app/web/`(`desktop/prepare.mjs:13,26-28`)。
- Web 版跑的是纯远程服务器页面;桌面版把前端文件打进安装包,**永远不加载服务器主页**,只把 `/api` 请求发给服务器(`desktop/README.md` 开篇)。这一点决定了桌面端对部署形态的关键影响:前端在客户端,后端只需要 API。

### 2.3 安全模型(值得复刻时参考)

- 所有窗口 `nodeIntegration:false, contextIsolation:true, sandbox:true, webSecurity:true`(`desktop/main.cjs:86-89,155,178-180`)。
- 预加载只暴露白名单方法;所有 `ipcMain.handle` 都先校验 `trusted(event)` / `localTransport(event)`,即必须来自主窗口主 frame 且 URL 必须是本地前端 origin(`desktop/main.cjs:126-140`)。
- 导航守卫:只在受信 origin 内允许,外部网址一律用系统浏览器打开(`desktop/main.cjs:203-222`、`policy.cjs:17-46`)。
- 权限:只允许音频(语音用的 media)、剪贴板写入、全屏(`policy.cjs:42-46`)。

---

## 3. 桌面版如何承载前后端(本地 vs 远程)——复刻是否做桌面端的关键

### 3.1 不内嵌后端,只代理 API

- 桌面端启动后,若配置了服务器地址 `baseUrl`,主窗口直接 `loadURL(APP_URL)`,即加载本地协议 `https://app.aivory.invalid/`(`desktop/main.cjs:8,224-233`;`local-web.cjs:7`)。
- 该 origin 由 `desktopSession.protocol.handle('https', createLocalHandler(...))` 接管(`desktop/main.cjs:265-274`),`createLocalHandler` 从磁盘 `web/` 目录读取静态文件(`local-web.cjs:92-109`),实现**离线本地加载前端**(无任何端口、无 HTTP 服务)。
- `/api` 路径命中时,协议处理器把请求转发到远程 `baseUrl`(`local-web.cjs:34-87`):剥离本地 header、设置远端 Origin 以通过 CSRF、`credentials:'include'`、透传流式 body(SSE 不缓冲,`local-web.cjs:64-66`),并把 `set-cookie` 留在远端 session 中(HTTP-only)。`serverUrl()` 负责把本地 URL 改写为远端 URL(`local-web.cjs:24-27`)。
- 结论:**桌面端零 Node 后端运行**,需要的只是一个"提供 `/api` 的远程 HTTP 服务"。这对 Python 后端是绝对利好——只要 Python 后端实现 `/api` REST 接口,桌面壳无需改就行。

### 3.2 服务器地址配置

- 来源优先级:打包默认值 `config.json.baseUrl` → 用户保存 `userData/server.json`(`desktop/server-config.cjs:5-31`,`desktop/main.cjs:382-384`)。
- 配置校验 `normalizeBaseUrl`:`http(s)://` 开头、纯 origin、无路径/query/hash/凭据(`desktop/policy.cjs:3-15`)。
- 首启无配置时弹出「服务器设置」窗口(`server.html`),让用户填地址(`desktop/main.cjs:76-98`)。
- 每个服务器用独立持久化 session partition `persist:aivory-<sha256(baseUrl)前16位>`(`desktop/main.cjs:262-263`),切换服务器即切换 cookie/storage 隔离区。

### 3.3 应用图标 / 托盘 / 单实例

- 单实例锁:`requestSingleInstanceLock()`(`desktop/main.cjs:351`);支持 `--quit` 命令行退出(`desktop/main.cjs:66-71,353`);`activate` 恢复窗口(`desktop/main.cjs:385`)。
- 托盘 + macOS Dock 菜单(`desktop/main.cjs:46-74`)。
- macOS 隐藏标题栏绘制自绘标题区 + 拖拽区(`desktop/window-chrome.cjs`)。

---

## 4. 离线与连接管理(ConnectionStatus)

- `desktop/connection.cjs` `ConnectionStatus` 类:
  - 通过主窗口 `did-fail-load` 检测加载失败(`connection.cjs:56-58`),`failed()` 显示离线覆盖层(`connection.cjs:108-111`)。
  - 用 `net.isOnline()` + 渲染进程 `navigator.onLine` 事件判断在线/离线(`connection.cjs:14,23-36`)。
  - 断线/不可达时,显示一个 `WebContentsView` 覆盖层 `offline.html`(离线页),每 10 秒自动重测(`connection.cjs:75-96`)。
  - 重连(`retry`):请求远端 `GET /api/public/needs-setup`(5 秒超时),`>=500` 视为服务器不可用(`connection.cjs:113-133`)。
  - 恢复后移除覆盖层并向渲染进程发 `desktop:connection-restored`(`connection.cjs:97-106`;preload 转成 `online`/`aivory:desktop-reconnected` 事件,`preload.cjs:43-46`)。
- **离线页依赖的接口**:`/api/public/needs-setup` —— Python 后端需提供(Web 版无此覆盖层,纯桌面功能)。

---

## 5. 认证与浏览器登录

### 5.1 桌面端内嵌浏览器登录(PKCE 流)

- 触发:`preload.cjs` 暴露 `loginInBrowser()`(桌面端"用浏览器登录"按钮)→ `desktop:browser-login` → `browserAuth.start()`(`desktop/main.cjs:142`)。
- 流程(`desktop/browser-auth.cjs:22-66`):
  1. 生成 verifier + sha256 challenge。
  2. `POST /api/auth/desktop/start { challenge }` → 返回 `request_id`(43 位 base64url)。
  3. 打开系统浏览器 `GET <baseUrl>/desktop/authorize?request_id=...`。
  4. 用户在其已登录浏览器上 approve/deny。
  5. 桌面端每 2 秒轮询 `POST /api/auth/desktop/token { request_id, verifier }`,直到 `authorized`。
  6. 成功后 `onAuthorized()` 恢复窗口并广播 `desktop:authorized`。
  - 5 分钟超时(`desktop/browser-auth.cjs:25`)。

### 5.2 服务端对应接口(Go 实现,供 Python 复刻)

| 接口 | 方法 | 说明 | 源码 |
|------|------|------|------|
| `/api/auth/desktop/start` | POST | 存 challenge 与 request_id(缓存 TTL 5min),返回 `request_id`,`expires_at`,限流 | `server/internal/api/desktop_auth_handlers.go:42-76`、路由 `server/internal/api/router.go:188` |
| `/api/auth/desktop/token` | POST | PKCE 校验 verifier,返回 `pending/authorized/denied`,签发 cookie | `desktop_auth_handlers.go:132-196`、`router.go:189` |
| `/api/auth/desktop/authorize` | GET/POST | 需登录;浏览器侧展示同意信息/提交授权,写入 grant | `desktop_auth_handlers.go:90-130`、`router.go:190-191` |

- 核心语义:PKCE 证明 + 一次性 grant,`CompareAndDelete` 防重放(`desktop_auth_handlers.go:146-152,159-162`)。

### 5.3 密码登录

- 桌面端主窗口走的是同一套 Web 密码登录(渲染进程内),不经过上面的 PKCE 桥。PKCE 桥只在「用浏览器外部授权」时用。

---

## 6. 自动更新机制

### 6.1 检查与“下载=打开安装包”

- 主进程 `UpdateChecker`(`desktop/updates.cjs`):
  - 启动后 15 秒 + 每 4 小时自动 `check()`(`desktop/main.cjs:324-325`),菜单/托盘可手动(`desktop/main.cjs:50,345`)。
  - `find()` → `GET /api/public/desktop-update`(Accept: application/json,UA `Aivory-Desktop/<version>`,10 秒超时,`updates.cjs:37-49`)。
  - 解析清单 `selectUpdate()`:需 `enabled===true`、`next > current`(semver)、按 `{os}_{arch}` 查 `downloads`(`updates.cjs:3-17`),平台映射 `darwin→macos`, `win32→windows`, `linux:linux`,架构限 `x64/arm64`(`updates.cjs:5-6`)。
- 找到新版本 → 弹本地对话框(手动提示或自动静默提示一次,`updates.cjs:26-35`)→ 用户点「更新」→ `shell.openExternal(update.url)`(`desktop/main.cjs:307-313`)。
- **更新是"全量替换安装包"**,不是增量补丁:桌面端自身**不下载、不校验、不安装**,只是把管理员配好的下载 URL 交给系统浏览器/默认下载器,再由用户运行安装包。安装动作交给 electron-builder 生成的安装器(NSIS/dmg/deb/AppImage)。

### 6.2 服务端清单与 admin 集成(Go 实现)

- **配置存哪**:`store.GetSetting(db, "desktop_update")`(`desktop_update_handlers.go:139-146`)。
- **公开接口** `GET /api/public/desktop-update`(`router.go:219`):
  - `enabled=false` 时返回 `{"enabled":false}`(`desktop_update_handlers.go:148-152`)。
  - 启用时返回完整 `desktopUpdateConfig`:`{enabled, source, version, downloads:{windows_x64|...}}`(`desktop_update_handlers.go:21-26,148-157`)。
  - 公开检查**不访问 GitHub**,只读管理员已发布的清单(`desktop_update_handlers.go:155-156`)。
- **admin 接口**(`router.go:703-704`):
  - `GET /api/admin/desktop-update`:读取配置 + 从 GitHub release 拉目录(system update catalog),给出 `{config, latest_version, update_available, releases[], check_error}`(`desktop_update_handlers.go:159-197`)。
  - `POST /api/admin/desktop-update/check`:强制刷新(`desktop_update_handlers.go:199-201`)。
  - 平台白名单:`windows/macos/linux × x64/arm64`(`desktop_update_handlers.go:15-19`)。官方源资产命名 `Aivory-<v>-<os>-<arch>.<ext>`(`desktop_update_handlers.go:85-115`)。
- **下载入口** `GET /api/public/desktop-download`(`router.go:220`,`desktop_download_handlers.go:37-46`):管理员配置的 `desktop_download` 设置,用于站点页面上"下载桌面端"按钮。
- 管理端 UI:仓库顶层 `updater/` 目录是**服务器自身系统更新器**,与桌面更新共用 `systemUpdate` 机制,但桌面更新只管"把桌面安装包 URL 发给客户端"。

> 注意:`desktop/updater/` 目录并不存在;任务描述中提到的 `desktop/updater/` 实际是仓库根的 `updater/`(Go 系统更新器,与桌面客户端无直接运行关系)。

---

## 7. 构建与分发(三平台)

### 7.1 构建脚本链

- `desktop/frontend.mjs`:用 Vite 构建前端到 `dist-desktop`(`VITE_AIVORY_DESKTOP=true, VITE_API_BASE=/api`)(`frontend.mjs:7-17`)。
- `desktop/prepare.mjs:prepareApp()`:
  - 读 `.env.local`/`.env` 的 `AIVORY_DESKTOP_BASE_URL`(`policy.cjs:1` + `prepare.mjs:15-17`)。
  - 拷贝所有 `.cjs` 主进程文件、离线/服务器页面、图标资源到 `generated/app/`(`prepare.mjs:20-24`)。
  - 拷贝 `web/`(打包前端,强制存在,`prepare.mjs:26-28`)。
  - 用 `@resvg/resvg-js` 生成 icon/tray 图标(`prepare.mjs:29-42`)。
  - 只把 `semver`、`ws` 两个运行时依赖装进包里(`prepare.mjs:43-49`)。
  - 写 `config.json{baseUrl}` 与 `package.json` 版本号(取根 package 版本,`prepare.mjs:50-63`)。
- `desktop/build.mjs`:buildFrontend → prepareApp → 调 `electron-builder --config electron-builder.cjs`(`build.mjs:9-18`)。
- `desktop/dev.mjs`:prepareApp → 用 electron 直接跑 `generated/app`(`dev.mjs:9-14`)。
- `desktop/electron-builder.cjs`:
  - `appId: com.aivory.desktop`,`productName: Aivory`,`asar:true`,`npmRebuild:false`(`electron-builder.cjs:5-14`)。
  - 打包内容 `files:['package.json','*.cjs','offline.*','server.*','config.json','assets/**/*','web/**/*']`(`electron-builder.cjs:12`)。
  - mac:`dmg`+`zip`,hardenedRuntime + entitlements(有签名则正式、无则 adhoc),支持 notarize;加麦克风用途描述(`electron-builder.cjs:17-29`)。
  - win:`nsis`(可改目录、桌面/开始菜单快捷方式)(`electron-builder.cjs:30-36`)。
  - linux:`AppImage`+`deb`(`electron-builder.cjs:37-41`)。
- 测试:`desktop/tests/*.test.mjs`,覆盖 api-requests / browser-auth / local-web / policy / reply-notifications / server-config / updates / window-lifecycle。

### 7.2 分发形态

- 全量安装包,每平台在原生 OS 上构建(`desktop/README.md`);GitHub Actions 提供 Windows / macOS(x64+arm64) / Linux 构建。
- 更新链路由 admin 配置清单指向 GitHub release(或自定义 URL),客户端只负责打开该 URL 让用户下载安装。

---

## 8. 其他主进程能力

- **API 请求取消/SSE 流**:`desktop/api-requests.cjs` —— 协议 handler 用 `x-aivory-desktop-request` header 关联请求,`ApiRequests` 用 AbortController 支持取消,含 30s 未命中自动清理;页面销毁时中止其请求(`api-requests.cjs:1-56`)。渲染进程通过 `preload.startApiRequest/abortApiRequest` 接入(`preload.cjs:39-40`)。
- **音频桥**:`desktop/audio-socket.cjs` —— 语音帧经窄桥发到主进程 `ws` 直连 `wss://<baseUrl>/api/audio/stream`(或 `ws://`,`audio-socket.cjs:36-44`),带 cookie + UA;有界 FIFO 缓冲(上限约 960KB,`audio-socket.cjs:2`)、单飞发送防堆积。渲染端 `preload.connectAudioSocket`(`preload.cjs:4-27`)。
- **系统通知(回复提醒)**:`desktop/reply-notifications.cjs` —— 窗口失焦/不可见时显示原生通知,点击恢复窗口并带目标定位(`reply-notifications.cjs:16-52`);由前端 `showReplyNotification` 触发(`main.cjs:146-147`),含清理(`clear`)、上限 32 条(`reply-notifications.cjs:41-46`)。
- **支付窗口**:`desktop:payment` 打开一个独立 BrowserWindow,支持 `redirect` 与 `form_post`(`main.cjs:148-164`),支付完成后重定向回 `/subscription` 并通知主窗口(`main.cjs:204-211`)。
- **多语言**:`desktop/locales.cjs` —— en/zh/zh-Hant/ja/fr 五套桌面壳文案,按系统 locale 选择(`locales.cjs:89-92`);连接状态页与服务器设置页都走它。

---

## 9. 复刻要点(对新 Python 后端)

### 9.1 是否先做桌面端?

**强烈建议:先只做 Web 版(个人部署),桌面端延后。** 理由:

1. 桌面端本质是「打包 Web 前端 + 代理 /api 」，功能价值在于免浏览器、托盘、原生通知、离线覆盖层,对核心产品(对话/AI/RAG/订阅)无增量;Web 版已覆盖全部业务。
2. 桌面端的前端本身就是仓库根同一套 React/vite 前端——Python 复刻时,前端可整体复用,只需保证 `VITE_API_BASE=/api` 与后端 `/api` 前缀一致,桌面壳几乎零改动即可接上。
3. 桌面端唯一需要 Python 后端额外实现的是少量 desktop 专属接口(见下),工作量占后端很小。

### 9.2 桌面客户端对后端的硬性接口依赖(Python 须提供)

| 需求来源 | 接口 | 说明 | 对应原实现 |
|----------|------|------|-----------|
| 连接重试 | `GET /api/public/needs-setup` | 探测服务器可用性;<500 视为可用 | `connection.cjs:119` |
| 更新检查 | `GET /api/public/desktop-update` | 返回清单 `{enabled, version, downloads:{windows_x64,macos_arm64,...}}`;enabled=false 也须返回 200 | `updates.cjs:37-49`、`desktop_update_handlers.go:148-157` |
| 更新下载展示 | `GET /api/public/desktop-download` | 返回 `{enabled, url}` | `desktop_download_handlers.go:37-46` |
| 浏览器登录 | `POST /api/auth/desktop/start`、`POST /api/auth/desktop/token`、`GET/POST /api/auth/desktop/authorize` | PKCE 桥,见 5.2 | `desktop_auth_handlers.go` |
| 前端渲染 | `/` 或 SPA 路由 | Web 版需要;桌面端不需要(前端在本地) | — |
| 音频 | `WS /api/audio/stream` | 语音;Web 版也需要同一套 | `audio-socket.cjs:36` |

- **更新协议注意**:客户端只消费清单并打开下载 URL,**不参与签名校验/增量补丁/安装**,Python 后端只需一个「读配置 → 返回 JSON」handler + admin 写配置的接口即可完整对接。GitHub release 目录拉取是 admin 侧可选项,非客户端硬依赖。

### 9.3 Python 后端需满足的通用约束

1. **API 前缀 `/api`**:桌面壳把 `VITE_API_BASE` 设为 `/api`,所有业务接口要挂在 `/api` 下(见 `frontend.mjs:12`,`local-web.cjs:34`)。
2. **CSRF / Origin 校验兼容**:桌面代理会把请求 `Origin` 设成远端服务器 origin(`local-web.cjs:41`),Python 端 CSRF 校验须接受同 origin / 合理放行 HTTPS 桌面壳。
3. **Cookie 会话**:桌面壳用 `credentials:'include'` 并依赖服务端 `set-cookie`(HTTP-only)维护会话(`local-web.cjs:38,43`);密码登录、PKCE token 换取都要能种 cookie。
4. **SSE/流式**:聊天与 `desktop-update` 之外的业务响应要支持流式透传,协议代理不做缓冲(`local-web.cjs:64-66`)。
5. **CORS/CAM**:Web 版需要正确处理跨域;桌面壳因是协议代理、同 origin 请求,对 CORS 更宽容,但生产建议 HTTPS。

### 9.4 复刻时的实施顺序建议

1. 先做 **Python 后端 + 现有 Web 前端**(个人部署,浏览器访问)。
2. 若要桌面体验:复用现有 `desktop/` 全套 Electron 壳(它是与前端/后端解耦的),只需:
   - 在 Python 后端新增上述 5 类 desktop 接口(约 `needs-setup` + `desktop-update` + `desktop-download` + PKCE 3 个 auth handler + 对应 admin 写配置);
   - 用 `AIVORY_DESKTOP_BASE_URL=<python后端>` 走 `desktop/build.mjs` 打出三平台安装包。
3. 桌面端更新可通过 Python 后端提供一个简单的 JSON 清单(指向自托管安装包或 GitHub release)即可,无需上 electron-updater 等重量方案。

---

## 附:关键文件清单

**主进程/壳**:`desktop/main.cjs`、`desktop/window-lifecycle.cjs`、`desktop/window-chrome.cjs`、`desktop/policy.cjs`、`desktop/locales.cjs`
**预加载**:`desktop/preload.cjs`、`desktop/server-preload.cjs`、`desktop/offline-preload.cjs`
**本地承载**:`desktop/local-web.cjs`、`desktop/server-config.cjs`
**离线与连接**:`desktop/connection.cjs`、`desktop/offline.html/.js/.css`
**认证**:`desktop/browser-auth.cjs`;服务端 `server/internal/api/desktop_auth_handlers.go`
**更新**:`desktop/updates.cjs`;服务端 `server/internal/api/desktop_update_handlers.go`、`desktop_download_handlers.go`,路由 `router.go:188-191,219-220,703-704`
**构建**:`desktop/prepare.mjs`、`desktop/frontend.mjs`、`desktop/build.mjs`、`desktop/dev.mjs`、`desktop/electron-builder.cjs`
**其他**:`desktop/api-requests.cjs`、`desktop/audio-socket.cjs`、`desktop/reply-notifications.cjs`
