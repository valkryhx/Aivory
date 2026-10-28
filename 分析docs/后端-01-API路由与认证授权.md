# Aivory 后端调研 01：HTTP API 路由层 + 认证授权系统

> 目标：深度复刻 Aivory 的 Go 后端（后端换成 Python/FastAPI）。本文聚焦 **Go 后端 HTTP API 路由层 + 认证授权系统**，为 FastAPI 复刻提供协议/字段/时序级参考。
> 源码根：`D:\git_codes\Aivory\server`
> 涉及包：`internal/api`（router/middleware/handlers）、`internal/auth`（JWT/TOTP）、`internal/oauth`（OAuth 提供商）、`internal/store/refresh_tokens.go`（会话持久化）。

---

## 0. 架构总览

- 后端是一个**单二进制 Go 进程**，使用**标准库 `net/http` + 手写微型 mux**（`internal/api/mux.go`），刻意不引入 gin/echo 等路由框架，减少依赖面。
- 所有路由统一在 `NewRouter(d Deps) http.Handler`（`router.go:145`）中注册，共 **441 条** `mux.handle(...)` 调用（`grep -c` 验证）。
- 依赖通过 `Deps` 结构体（`router.go:35`）注入：DB（`*sql.DB`）、Cache（Redis/内存抽象）、Auth（JWT service）、Queue、Mailer、LLM Provider、RAG、Passkey service、Logger 等。
- **鉴权模型**：短时效 access JWT（cookie `auth_token` 或 `Authorization: Bearer`）+ 长时效 refresh JWT（HttpOnly cookie `refresh_token`，Path 限定 `/api/auth`），配合数据库中 `refresh_tokens` 表做会话持久化、旋转与撤销。
- **分层中间件链**（从外到内）：`errorResponseLoggingMiddleware` → `recoverMiddleware` → `spaHandler`(可选) → `corsMiddleware` → `siteAccessMiddleware` → 自定义 mux → 路由级包装器(`rateLimitedIP`/`requireAuth`/`requireAdmin`)。

---

## 1. API 路由地图

### 1.1 路由注册与 mux 工作机制

自定义 mux（`mux.go:36-108`）：
- `mux.handle(method, pattern, handler)`：`pattern` 使用 `:name` 路径参数语法，拆分为段数组。
- `matchPath`：逐段匹配；`:` 前缀段为参数，写入 `r.Context()`（`pathCtxKey`），处理器用 `pathParam(r, "name")` 读取。
- **首次匹配原则**（`ServeHTTP` 逐条线性扫描）：静态路由需注册在 `/:id` 前，否则会被吞（如 `/api/admin/models/reorder` 必须早于 `/api/admin/models/:id`，见 `router.go:537-540`、`router.go:601-602` 注释）。
- 方法不匹配记下 pattern 后继续扫描，最终 404 JSON `{"error":"not found"}`。
- 请求命中后调用 `auditHTTPRequest(d, w, r, handler)`（在 mux 层做审计，见 4.4）。
- `writeJSON`/`writeError` 统一 JSON 响应；`writeError` 对 5xx 落到客户端只返回 `"internal server error"`，真实错误仅写日志（`mux.go:117-132`）。
- **请求体上限**：`decodeJSON` 用 `MaxBytesReader` 限制 4MB（`mux.go:137`，`jsonRequestBodySizeCap` 可 `AIVORY_API_JSON_REQUEST_BODY_SIZE_CAP` 覆盖）；必须为合法 UTF-8。

### 1.2 按资源分组的端点

> 方法/路径/用途/是否鉴权。441 条不逐条罗列，按资源分组概括 + 标出代表性关键端点。鉴权列：🔓=公开、🔒=requireAuth、🛡=requireAdmin、⏱=额外 IP 限流。

#### A. 认证核心（`/api/auth/*`）— 公开凭据面，全部强制 IP 限流
| 方法/路径 | 用途 | 鉴权 |
|---|---|---|
| POST /api/auth/register | 注册（默认 role=user），可发邮件验证码 | 🔓⏱ 5/60s |
| POST /api/auth/login | 密码登录；返回 access/refresh cookie；启 2FA 时返回 ticket | 🔓⏱ 10/60s |
| POST /api/auth/login/2fa | 用 ticket + TOTP 码完成登录 | 🔓⏱ 10/60s |
| POST /api/auth/passkey/begin | WebAuthn 免密登录挑战（断言开始） | 🔓⏱ 10/60s |
| POST /api/auth/passkey/verify | WebAuthn 免密登录验证 | 🔓⏱ 10/60s |
| POST /api/auth/logout | 登出，旋转/撤销 refresh token | 🔓⏱ 30/60s |
| POST /api/auth/refresh | 用 refresh cookie 换新 token 对 | 🔓⏱ 30/60s |
| POST /api/auth/session | 浏览器启动会话探测（200 而非 401） | 🔓⏱ 30/60s |
| GET/POST /api/auth/desktop/start·token | 桌面客户端 OAuth 式授权码换取 token | 🔓⏱ |
| GET/POST /api/auth/desktop/authorize | 桌面授权信息/授权 | 🔒 |
| POST /api/auth/verify-email | 6 位邮箱验证码激活账户 | 🔓⏱ 10/300s |
| POST /api/auth/send-code | 重发验证码（purpose=verify/reset） | 🔓⏱ 3/60s |
| POST /api/auth/forgot-password | 发送重置邮件链接 | 🔓⏱ 5/900s |
| POST /api/auth/reset-password | 用重置 token 改密 | 🔓⏱ 5/60s |
| GET /api/auth/sessions | 列出当前用户活跃会话 | 🔒 |
| POST /api/auth/sessions/revoke-others | 撤销其它会话 | 🔒 |
| POST /api/auth/sessions/:jti/revoke | 按 jti 撤销某会话 | 🔒 |
| GET /api/auth/oauth/:id/start | 开始 OAuth 授权码流（302 跳提供商） | 🔓⏱ 20/60s |
| GET/POST /api/auth/oauth/:id/callback | OAuth/Apple 回调 | 🔓⏱ 20/60s |
| GET /api/auth/oauth/handoff | 跨域回调单次 token 换取（设本域 cookie） | 🔓⏱ 20/60s |

#### B. 公开面（`/api/public/*` 及 misc）— 无鉴权
| 方法/路径 | 用途 |
|---|---|
| GET /api/health | 健康检查 |
| GET /api/public/signup-open | 是否开放注册 + 是否要求滑块验证码 |
| GET /api/public/captcha、POST /api/public/captcha/verify | 滑块拼图验证码：发挑战 / 校验得单次 pass token |
| GET /api/public/needs-setup、POST /api/setup | 首次部署探测 / 创建首个管理员 |
| GET /api/public/oauth-providers | 启用的 OAuth 提供商列表（无密钥） |
| GET /api/public/auth-policy | 公共认证策略（密码/passkey 开关、入口模式、默认提供商） |
| GET /api/public/legal-config、/api/public/site-status | 法律配置 / 站点关闭状态 |
| GET /api/public/user-groups、/api/credit-packages | 会员层级/积分包公开定价 |
| GET /api/public/shared/:token 等 | 公开会话分享快照（token 在路径） |
| GET /api/payments/webhooks/:channelId | 支付回调（按渠道签名鉴权） |
| GET /api/icons/:filename、/api/avatars/:filename | 公共图标/头像（wrap，无鉴权但文件名是随机 hex） |
| GET /api/public/maps/* 、/api/public/desktop-* | 地图 geocode/proxy、桌面更新 |

#### C. 当前用户面（`/api/me/*`）
| 方法/路径 | 用途 |
|---|---|
| GET/PATCH/DELETE /api/me | 个人信息读/改/删（删为异步用户删除） |
| GET /api/me/usage、/credits、/settings、/upload-policy | 用量/积分/设置/上传策略 |
| PATCH /api/me/password、POST /api/me/password/set | 改密 / 首次设置密码 ⏱ 5/60s |
| POST /api/me/avatar | 头像上传 |
| POST /api/me/2fa/setup·enable·disable | 2FA TOTP 设置/启用/禁用 ⏱ 10/300s |
| GET /api/me/passkeys、POST /api/me/passkeys/begin·finish、DELETE /api/me/passkeys/:id | Passkey 设备自助管理 ⏱ 10/300s |
| GET /api/me/identities、DELETE /api/me/identities、POST /api/me/identities/:id/link | 第三方身份绑定/解绑/开始绑定 |
| GET /api/me/memories 等 | 记忆 CRUD（requireMemoryHandler） |
| POST /api/me/redeem | 兑换码 ⏱ 10/60s |
| GET /api/me/skills、/prompts、/mcps、/images、/files、/storage、/ppt/* | 技能/提示词/MCP/图片/文件/PPT 子资源 |

#### D. 会话/聊天（`/api/conversations`、`/api/messages`）
| 方法/路径 | 用途 |
|---|---|
| GET/POST/DELETE /api/conversations | 列表/新建/清空全部 |
| POST /api/conversations/archive-all、/import、/draft-scope | 归档/导入/草稿作用域切换 |
| GET/PATCH/DELETE /api/conversations/:id | 会话 CRUD、reorder、compact |
| GET/POST /api/conversations/:id/messages | 消息列表/发送（SSE 流式） |
| GET /api/conversations/:id/messages/:msgId/stream、PATCH、DELETE | 流式/编辑/删除 |
| POST /api/conversations/:id/stop、/regenerate、/fork、/inline-threads | 停止/重生成/分叉/内联线程 |
| POST /api/conversations/:id/export/pdf、/share、/api/shared/:token/clone | PDF 导出/分享/克隆 |
| GET /api/events | SSE 实时事件流（requireAuth） |
| GET /api/models、/tools、/image-models、/skills、/search、/private-chat | 模型/工具/搜索/私聊 |

#### E. 文件/知识库（`/api/files`、`/api/kbs`、`/api/documents`、`/api/artifacts`）
- 文件上传走 `requireCapabilityHandler(errFileUploadGroupPermission, AllowFileUpload, ...)`。
- KB 所有路由套 `requireKnowledgeBaseHandler`。包含 documents 上传/列表/重试/重命名/删除、shares 增删、share-candidates、workspace-members。
- 资源下载点 `GET /api/files/:id`、`GET /api/documents/:id/content`、`GET /api/artifacts/:id` 走 cookie 鉴权（`requestSignatureExempt` 放行 img 类请求）。

#### F. 工作区/项目（`/api/workspaces`、`/api/projects`）
- workspace：CRUD、profile/icon、members（改权限/角色/踢人）、leave、invite（rotate/list/create/revoke）、transfer、policy、announcement、usage、audit log。
- 邀请链接 `/api/workspaces/join/:token`（GET 探测 + POST 加入）对未知 token 统一 404，192-bit token 空间 + IP 限流防扫。
- project：CRUD、documents（上传受 requireKnowledgeBase + 文件能力）。

#### G. 管理面（`/api/admin/*`）— requireAdmin
- onboarding、channels（模型渠道）、models（CRUD/reorder/quotas/skills/fast）、model-tags、tools/builtins、mcp。
- user-groups、credit-packages、payment-channels、payment-methods、payment-orders。
- users（CRUD/ban/unban/password/email/role/2fa disable、drill-down conversations/projects/memories/login-history/images/kbs）、conversations、domains、workspaces。
- resources、usage、tool-logs、audit-logs、analytics、overview、feedback、files。
- oauth-providers（CRUD/prepare/reorder）、settings、system-prompts、email/preview。
- aippt、backup（export/import/archives/import-jobs）、config export/import、vectors、redeem-codes、icons。
- `requireAdmin` 内部先 `requireAuth` 再判 `u.Role == "admin"`（`middleware.go:166`）。

---

## 2. 中间件链与请求生命周期

### 2.1 外层组装顺序（`router.go:754-758`）

```go
var handler http.Handler = corsMiddleware(d.Config.AllowedOrigins, siteAccessMiddleware(d, mux))
if d.Config.StaticDir != "" {
    handler = spaHandler(d.Config.StaticDir, handler)   // 同进程托管前端 SPA
}
return errorResponseLoggingMiddleware(d.Logger, recoverMiddleware(handler))
```

请求进入顺序（从外到内）：
1. **errorResponseLoggingMiddleware**（`response_logging.go:274`）——最外层，用 `responseLogWriter` 包装 ResponseWriter，记录非 2xx 的 method/route/status/duration/reason；成功响应体和 SSE/文件下载**不缓冲**。为实现 SSE 的 `Flusher`、WebSocket 的 `Hijacker`、HTTP/2 的 `Pusher`、文件的 `ReaderFrom`，它按底层 ResponseWriter 能力动态选择组合包装类型（`responseLogF/H/P/R` 组合矩阵，`response_logging.go:212`）。
2. **recoverMiddleware**（`router.go:815`）——捕获 handler panic，记录堆栈，回 500，不崩进程。
3. **spaHandler**（`router.go:771`）——仅当配置了 `STATIC_DIR`（单容器部署前后端同源）。非 `/api/*` 路径：存在的真实文件按 `Cache-Control` 策略服务（`/assets/*` 永久缓存、`index.html/version.json/sw.js` 禁止缓存、public 文件 must-revalidate）；缺失的 `/assets/*` 回 404（不吞掉旧 chunk）；其它未知路径回退 `index.html`（SPA 路由）。`path.Clean` 防目录穿越。
4. **corsMiddleware**（`router.go:830`）——**回显式 CORS**：只有 `Origin` 精确命中 `AllowedOrigins` 配置才回 `Access-Control-Allow-Origin: <origin>` + `Allow-Credentials: true`（credentials 模式不能用 `*`）。`Allow-Headers` 明确列出签名字段等自定义头；`Allow-Methods: GET,POST,PATCH,PUT,DELETE,OPTIONS`；`Max-Age: 600s`。OPTIONS 直接回 204。
5. **siteAccessMiddleware**（`site_access.go:94`）——站点整体关闭开关（`site_access` 设置）。`/api/auth/*`（除 register）、支付回调和一批 public 端点豁免；管理员在关闭时放行；普通用户回 503 `{error:"site_closed", closed:true, message}` + `Retry-After: 30`。
6. **自定义 mux** —— 命中路由 → `auditHTTPRequest` → 路由级包装器（见下）。

### 2.2 路由级包装器（核心鉴权时序）

每个路由的鉴权通过**包装器函数组合**表达（`router.go` 中反复出现）：

- `wrap(d, h)`：无鉴权直接转发（`middleware.go:55`）。
- `rateLimitedIP(d, scope, perWindow, window, next)`：先做 per-IP 限流，超限回 429，再进 next（`middleware.go:347`）。常与 `requireAuth`/`wrap` 叠加。
- `requireAuth(d, h)`：完整 access-token 校验（`middleware.go:73`）。
- `requireAdmin(d, h)`：requireAuth + role 校验 + 管理员审计（`middleware.go:166`）。
- `requireCapabilityHandler(denied, allowed, h)`：校验用户组权限（如 AllowFileUpload / AllowSharing / AllowKnowledgeBases / AllowVoiceTranscription / AllowConversationExport），见 `permissions.go:256`，通常嵌套在 requireAuth 之后。
- `requireKnowledgeBaseHandler` / `requireMemoryHandler`：KB 聚合开关 / 记忆聚合开关 + 用户组权限。
- `aiPPTAuthorized`：AI PPT 功能开关。

### 2.3 requireAuth 内部校验序列（`middleware.go:73-162`）

1. **CSRF 门**（`csrfOK`）：对携带 `auth_token` cookie 的状态改变请求（非 GET/HEAD/OPTIONS），校验 `Origin` 属同源或在 AllowedOrigins 白名单，否则 403 `cross-site request blocked`。Bearer-only（无 cookie）豁免（`middleware.go:187-196`）。
2. **读 token**：`readAccessToken` 优先 `Authorization: Bearer`，其次 `auth_token` cookie（`middleware.go:242`）。
3. **解析 access JWT**：`d.Auth.ParseAccess`（`auth.go:163`），失败回 401。
4. **可选请求签名**：若配置 `RequestSignaturesRequired` 且非豁免路径，校验 v2 请求签名（见 5.3），失败 403。
5. **DB 权威状态校验**：`store.GetUserAuthStateForSession(uid, sessionID)` 取最新 role/status/token_ver/session_active；`Status != "active"` 回 403 `account_blocked`；`TV != claims.TV` 或 `!SessionActive` 回 401 `session_expired`。
6. **取用户**：`cachedAuthUser`（带缓存），失败 401。**关键点**：`Role/Status/TokenVer` 三个授权输入字段强制以 DB 读取值覆盖缓存值（`middleware.go:127-129`）。
7. workspace 路径注入审计上下文（`X-Request-ID`）。
8. **初始密码策略**：`!user.HasPassword` 且非初始密码设置请求时，若策略要求 OAuth 初始密码，回 412 `initial_password_required`。
9. **在线状态**：缓存节流（每分钟一次）后台 `store.TouchLastSeen`。
10. **域名访问控制**：`enforceDomainAccess`（企业域名白名单）。
11. 将 user 写入 context，调 handler。

### 2.4 客户端 IP 解析（防绕过，`middleware.go:258-315`）

- `clientIP`：仅当直连对端是 loopback/私有网段（即反向代理）才信任 `X-Forwarded-For`/`X-Real-IP`。
- **XFF 取右至左**：从最右往左找第一个非可信代理的 IP（防客户端伪造 `X-Forwarded-For: <spoofed>` 绕过限流）。
- `isTrustedPeer`：loopback + RFC-1918/RFC-4193 私有段。

### 2.5 审计（auditHTTPRequest）

- `auditHTTPRequest`（在 mux 命中处调用）与 `prepareAdminAudit`（requireAdmin）负责事件审计，`admin_audit_middleware.go` 承载实现；admin_audit_logs 提供审计日志查询/导出/删除端点。

---

## 3. JWT Token 模型详解

### 3.1 密钥派生（`auth.go:100-104`）

一个主 secret 经 HMAC-SHA256 按用途派生独立签名密钥，做到**用途隔离**（access 签发的 token 无法用 refresh 校验器通过）：

```go
func deriveSigningKey(master []byte, purpose string) []byte {
    h := hmac.New(sha256.New, master)
    _, _ = h.Write([]byte("aivory/jwt/" + purpose))
    return h.Sum(nil)
}
```
用途：`"access"`（accessKey）、`"refresh"`（refreshKey）、`"request-proof"`（requestKey，请求签名用，独立于 JWT）。

### 3.2 Access Token Claims（`auth.go:31-38`）

```go
type Claims struct {
    jwt.RegisteredClaims                 // sub=uid, exp, iat, jti
    UID       string `json:"uid"`
    Role      string `json:"role"`
    TV        int    `json:"tv"`         // token version（数据库比对）
    SessionID string `json:"sid,omitempty"`
    TokenUse  string `json:"token_use"`  // "access"
}
```
- 签发（`IssueAccessForSession`，`auth.go:109`）：`Header.typ = "at+jwt"`，HS256，jti=uuid，`sid` 绑定稳定会话族。
- 解析（`ParseAccess`，`auth.go:163`）：强制 `WithValidMethods([HS256])`、`WithExpirationRequired()`、`WithIssuedAt()`；校验 `typ=="at+jwt"`、`token_use=="access"`、`sub==uid`、`uid/sid/jti/iat` 非空。

### 3.3 Refresh Token Claims（`auth.go:41-46`）

```go
type RefreshClaims struct {
    jwt.RegisteredClaims         // sub=uid, exp, iat, jti
    UID      string `json:"uid"`
    TV       int    `json:"tv"`
    TokenUse string `json:"token_use"`  // "refresh"
}
```
- 签发（`IssueRefresh`，`auth.go:139`）：`Header.typ = "rt+jwt"`，jti 即会话记录主键，写库时持久化。
- 解析（`ParseRefresh`，`auth.go:185`）：同样限制 HS256/含 typ/token_use，`sub==uid`。

### 3.4 TTL（`auth.go:206-209`，由配置注入 `auth.New(secret, accessTTL, refreshTTL, cache)`）
- `AccessTTL()` / `RefreshTTL()` 取配置；短 access + 长 refresh，access 过期用 refresh 旋转。

### 3.5 会话/登出/撤销语义

**会话表**：`refresh_tokens` 表，主键 `jti`，含 `session_id, user_id, expires_at, revoked, created_at, user_agent, ip, location, last_seen`。`session_id` 缺省回退为 `jti`，同一设备跨旋转保持稳定 → "会话族"。

**旋转（`store.RotateRefreshToken`，`refresh_tokens.go:25`）**——同一事务内：
1. `UPDATE users SET token_ver=token_ver WHERE id=? AND status='active' AND token_ver=?`——获得用户行写锁，与改密串行化；不匹配则 `ErrInvalidRefreshToken`。
2. 查旧 JTI（未撤销未过期）：查不到且其为已撤销/未过期（说明已赢旋转竞争）→ **判定重放**，撤销整个 `session_id` 会话族，返回 `ErrRefreshTokenReplay`（`refresh_tokens.go:68-90`）。
3. 撤销旧 JTI、插入新 JTI 行（`session_id` 继承）。

**登出（`auth_handlers.go:589`）**：`logoutHandler` 先 `csrfRefreshOK`，解析 refresh cookie 并 `RevokeUserSession(jti)`，再清两个 cookie。

**撤销**：
- `RevokeUserSession(db, uid, jti)`：单会话撤销（`misc.go:2113`）。
- 改密/重置 → token_ver +1 → 所有该用户旧 access/refresh 因 `TV` 不匹配全部失效。
- 会话管理端点：`GET /api/auth/sessions` 列活跃会话、`POST revoke-others`、`POST /:jti/revoke`（当前族由 access-token `sid` 或 refresh cookie 推导，浏览器/Bearer 语义一致）。

**Cookie 细节**（`auth_handlers.go:912-1033`）：
- `auth_token`：HttpOnly、Path `/`、SameSite=Lax、Secure 视连接（`secureCookie` 判断 TLS 或受信代理 `X-Forwarded-Proto`）。
- `refresh_token`：HttpOnly、**Path `/api/auth`**（只随 auth 端点发送）、SameSite=Lax。
- 清 cookie 时按 `auth_token` 的 `/`、`/api`、`/api/auth` 三种旧 Path 变体都清，防旧 cookie 遮蔽。

---

## 4. 认证处理器时序

### 4.1 注册（`registerHandler`，`auth_handlers.go:241`）
流程：要求密码登录开启 → 校验邮箱/密码 → **首次用户保护**（无用户时 409 要求走 setup）→ 域名白名单 → `signup_open` 开关 → 读 `email_verification_required` → **滑块验证码门**（`register_captcha_required` 时消费单次 pass token）→ 邮箱查重 → **per-IP 每日注册配额**（`register_ip_daily_limit`，`regquota:<ip>:<date>` 计数器，失败回滚）→ `store.HashPassword`（bcrypt）→ 插入（`pending` 或 `active`）。pending 则后台异步发 6 位验证码（缓存 `verify:<email>`，TTL 10min），返回 `{verification_required:true, email, retry_after}`；active 则 `finaliseSession`。
- **防邮箱探测**：发码、忘记密码对存在/不存在邮箱返回相同响应并消耗相同冷却窗口（`reserveEmailSend`，`auth_handlers.go:43`，`mailcooldown:<purpose>:<recipient>` 缓存计数 120s）。
- **验证码防爆破**：`codeMatches` 用 `subtle.ConstantTimeCompare`，`registerCodeFailure` 计数达 5 次即删除（烧毁）验证码（`auth_handlers.go:80-90`）。

### 4.2 登录（`loginHandler`，`auth_handlers.go:515`）+ 2FA
1. `requirePasswordLoginEnabled` → 解码 → 可选 `login_captcha_required` 滑块门（先于账号查询，是最廉价的拒绝）。
2. 查用户；**不存在时用 `dummyPasswordHash`（cost-10 bcrypt）做等时比较**并返回与"密码错"相同的 401，防账号枚举（`auth_handlers.go:96,548`）。
3. 密码用 `store.CheckPassword`（bcrypt）比对；通过后才揭示账户状态（pending → 403 需验证邮箱；非 active → 403 被封）。
4. **2FA 门**：若 `user.TotpEnabled`，不发会话，改发**短时票据** `issueTwofaTicket`（随机 16 字节 hex，缓存 `2fa:<ticket>`，TTL 5min），返回 `{totp_required:true, ticket}`。
5. 否则 `finaliseLoginSession`（记录登录历史：password）。

**登录 2FA（`login2faHandler`，`twofa_handlers.go:144`）**：
- ticket 来源：请求体，或 OAuth 流的 `aivory_2fa` HttpOnly cookie（Path `/api/auth`）。
- 读缓存 `2fa:<ticket>` → 校验用户 active/totp_enabled/token_ver、OAuth 时校验 `OAuthProviderCallbackGuard`。
- 校验 TOTP：`VerifyTotp`（±1 step 容忍），失败时 `2fa:fail:<ticket>` 计数，达 5 次**烧毁票据**；成功用 `SetNX("2fa:used:<uid>:<code>", 90s)` 防重放，`CompareAndDelete` 消费票据。
- 完成后 `finaliseLoginSession`（password_2fa 或 oauth_2fa）。

### 4.3 忘记/重置密码（`password_reset.go`）
- `forgotPasswordHandler`：要求密码登录开启 → 校验/归一化邮箱 → `reserveEmailSend` 冷却（120s）→ 存在则发重置邮件。响应统一 `{ok:true, retry_after}`。
- 重置链接：`origin + "/reset-password?lang=...#token=" + token`（token 43 字符 base64url，32 字节随机）。重置授权用 `reset-link:<sha256(token)>` 缓存存 `passwordResetGrant{user_id, token_ver, expires_at}`（TTL 10min）。
- `resetPasswordHandler`：校验 token 格式正则与缓存授权、读当前密码、核对 `token_ver` → bcrypt 新密码 → `CompareAndDelete` 消费 → `UpdateUserPasswordIfCurrent`（并发改密防护）→ token_ver 提升 → 广播 `user:<id>:kill` 让在线会话下线 → 回 `{ok:true}`。
- `passwordResetOrigin` 只信任 `OAUTH_CALLBACK_BASE_URL` 或 `AllowedOrigins`，拒绝任意 Host/Origin/转发头（防 open-redirect）。

### 4.4 刷新/会话（`refreshHandler`/`sessionHandler`，`auth_handlers.go:691,719`）
- 共用 `refreshSession`（`auth_handlers.go:626`）：读 `refresh_token` cookie → `ParseRefresh` → 查用户（active、token_ver 匹配）→ `IssueRefresh` → `RotateRefreshToken`（含重放检测）→ `IssueAccessForSession`。
- `refreshHandler` 失败回 **401**（显式刷新契约）；`sessionHandler`（浏览器探测）失败则**清 cookie 回 200** `{authenticated:false, auth_policy}`，避免登录页 DevTools 出现预期 401。
- 均返回 `request_signing_key`（由新 access 的 `sid` 派生）与 `expires_at`。

---

## 5. 2FA / Passkey / OAuth 时序

### 5.1 TOTP 2FA（`totp.go`，`twofa_handlers.go`）
- 纯 Go 实现 RFC 6238：SHA-1、6 位、30s step，无外部依赖——**Google Authenticator/Authy/1Password 默认兼容**。
- `GenerateTotpSecret`（160-bit 熵 base32）；`TotpURI` 生成 `otpauth://totp/Issuer:Account?...`；`VerifyTotp` 容忍 ±1 step。
- 自管理端点：`/api/me/2fa/setup`（生成未启用 secret，存库）→ `enable`（验证 code + `SetNX` 防重放后置 `TotpEnabled`）→ `disable`（需当前 code）。
- 登录集成见 4.2；OAuth 登录也走 2FA（见 5.3）。

### 5.2 Passkey / WebAuthn 免密（`passkey_handlers.go` + `passkey_service.go`）
- 库：**go-webauthn/webauthn**。`PasskeyService` 接口把 go-webauthn 依赖隔离到 `passkey_service.go` 单文件，方便测试 fake。
- **登录流程（无用户名 discoverable）**：
  1. `POST /api/auth/passkey/begin`（公开）→ `BeginDiscoverableLogin`（空 allowList + `UserVerificationRequired`）→ 返回 `{ticket, options}`，挑战 session 缓存 `pkauth:<key>`（随机 key，TTL 5min）。
  2. `POST /api/auth/passkey/verify` → 按 rawID/userHandle 查凭据归属 → `FinishLogin` 验证签名/挑战/绑定/成功计数 → `CompareAndDelete` 消费票据 → `TouchPasskey` 更新计数与标志 → `finaliseLoginSession`（passkey）。
- **注册流程（自服务）**：`/api/me/passkeys/begin`（需登录，`ResidentKeyRequired` + `UserVerificationRequired` + `WithExclusions` 排除已有凭据）→ 缓存 `pkreg:<uid>` → `/finish` 创建凭据入库 → `/delete` 删除。上限 `maxPasskeysPerUser=10`。
- **RP 身份**：`passkeyRPOrigin` 优先用 `Origin` 头（兼容 TLS 终结代理），校验 scheme=HTTPS（localhost 可 http）、host 非 IP，RPID=hostname。
- **安全**：票据失败计数达 5 烧毁（同 2FA）、`passkeyBodyLimit=64KB`、服务不可用时 503 `passkey_login_disabled`。
- 凭据表存储 `credential_id, public_key, sign_count, authenticator_flags, flags_known`，一次性登录不额外要求 TOTP（passkey 本身是完整认证因子）。

### 5.3 OAuth / 社交登录（`oauth.go` + `oauth_handlers.go`）
- 提供商种类：内置 **Google、GitHub、Apple、WeCom（企业微信）** + 管理的 **OIDC** 与 **OAuth 2.0** 通用提供商。`Resolve` 把内置提供商钉到官方信任/协议端点，不允许自定义 host 覆盖（防信任根漂移）。
- **PKCE**：Google/OIDC/OAuth2 用 S256；GitHub classic 与 Apple 只用 state。
- **ID token 独立验证**（`oidc_token.go`）：签名/issuer/aud/azp/寿命/nonce 都独立验；GitHub 与 generic OAuth2 通过固定 HTTPS userinfo 端点解析身份。
- **回调安全**：
  - state 随机 24/32 字符，服务端缓存 `oauth:state:<state>`（TTL 10min），存 `oauthFlowState{verifier, nonce, origin, signup_ip, captcha, browser_binding, link_*}`。
  - **浏览器绑定**：`aivory_oauth_<sha256(state)[:16]>` HttpOnly cookie 存 binding（form_post 时 SameSite=None+Secure），回调时 `subtle.ConstantTimeCompare` 校验 —— 防登录状态被绑定到别的浏览器。
  - **跨域 handoff**：站点多域名、提供商只注一个 redirect_uri → 流固定落在权威域 A，回调后若用户起始域不同（且在 OAuthReturnOrigins 白名单）则铸一次性 token（缓存 `oauth:handoff:<tok>`，60s 单次）302 回该域 `/api/auth/oauth/handoff`，由 handoff 端点本域设 cookie。`allowedReturnOrigin` 为 open-redirect 门。
  - **身份绑定（link）**：`POST /api/me/identities/:id/link`（需登录，镜像 start 但把 `LinkUserID/TokenVer/SessionID` 存 state），回调走同一 `/api/auth/oauth/:id/callback`，走 BIND 分支（不建会话、不挑战 2FA，冲突检测）。
- **登录收尾**（`completeOAuthLoginWithGuard`，`oauth_handlers.go:197`）：账户状态门 → 若 TOTP 开启，铸 OAuth 2FA 票据并写入 `aivory_2fa` HttpOnly cookie（Path `/api/auth`，TTL 300s），302 到登录页 `?twofa=1`；否则 `issueSessionCookiesWithOAuthGuard`（含 `OAuthProviderCallbackGuard` 绑定）→ 记录 oauth 登录 → 302 `/`。
- **提供商回调保护**：`store.NewOAuthProviderCallbackGuard` / `ValidateOAuthProviderCallbackGuard` 确保回调时提供商 row 未被管理变更过（并发/篡改防御）。
- **subject 命名空间**（`SubjectNamespace`，`oauth.go:540`）：把信任域（kind/client_id/issuer/jwks/auth/token/userinfo 等字段）哈希进命名空间前缀，防止提供商种类/端点切换后旧身份被误继承。

### 5.4 滑块拼图验证码（`captcha.go`）
- 服务端用 `image/png` 渲染 280×160 背景 + 缺口 + 拼块，随机 gap 位于右侧 2/3。客户端返回**滑动比例（0-1 分数，分辨率无关）** + 拖动轨迹点。
- 校验：位置误差 `capTol≈0.025`、轨迹时长 180ms~20s、点数上限 80、挑战绑定、**单次使用**。通过则发放单次 **pass token**（按 purpose=login/register）。
- 挑战缓存：`captchaChallengeCacheTTL=2min`；`consumeCaptchaPass` 消费 pass token（登录/注册/OAuth 共用）。

---

## 6. 限流与安全策略

### 6.1 限流原语（`middleware.go:325-355`）
- `rateLimitIP`：**固定窗口 per-IP**，计数用 `d.Cache.Incr(key, window)`（Redis 共享，跨副本生效）。key：`rl:ip:<scope>:<ip>:<path>`。超限回 429 `errRateLimited`。
- `rateLimitUser`：**per-USER**，key：`rl:u:<scope>:<userID>`（用于耗敏的已认证操作，如文档上传）。
- `reserveConcurrentGen`：每用户**并发生成槽位**（默认 cap=3，可 `max_concurrent_generations` 覆盖），`gen:active:<uid>` Incr，超限 Decr 回退并拒绝（并发聊天生成限制）。

### 6.2 各接口限流预算（`router.go:69-142`，均可用 `AIVORY_API_RATE_LIMIT_*` 覆盖）
| 接口 scope | 预算（次/窗口） |
|---|---|
| auth register | 5 / 60s |
| auth login | 10 / 60s |
| login 2fa | 10 / 60s |
| passkey | 10 / 60s |
| logout | 30 / 60s |
| refresh / session | 30 / 60s |
| verify-email | 10 / 300s |
| send-code | 3 / 60s |
| forgot-password | 5 / 900s |
| reset-password | 5 / 60s |
| captcha issue/verify | 20 / 60s |
| first-run setup | 10 / 60s |
| oauth start/callback/handoff | 20 / 60s |
| password change/set | 5 / 60s |
| identity link start | 20 / 60s |
| 2fa setup/enable/disable | 10 / 300s |
| redeem code | 10 / 60s |
| payment checkout/resume | 15 / 60s |
| payment webhook | 600 / 60s |
| workspace join | 30 / 60s |
| public shared conversation | 60 / 60s |
| shared assets (files/artifacts) | 240 / 60s |
| map geocode / map sdk | 60/300 / 60s |
| private-chat | 30 / min |
| conversation pdf | 12 / min |

另有：**per-IP 每日注册配额** `register_ip_daily_limit`（`auth_handlers.go:322`，失败回滚）；**每日 token 配额** `daily_token_limit`（`middleware.go:439`，基于持久化计费账本）。

### 6.3 v2 请求签名（`middleware.go:464-629`）
可选启用（`RequestSignaturesRequired`）。浏览器为每个已认证请求附加头：
`X-Req-Ts`、`X-Req-Nonce`、`X-Req-Token`（MAC，base64 严格解码、长度 32）、`X-Device-Id`、`X-Req-Content-SHA256`。

HMAC 消息（以 `\x00` 连接）：
```
v2 \x00 ts \x00 nonce \x00 METHOD \x00 canonicalTarget \x00 deviceId \x00 payloadDigest \x00 sha256(rawToken)
```
- key：由会话 `RequestSigningKey(sessionID)` 派生（HMAC(requestKey, "aivory/request-proof/session/"+sid)，base64url），**与 JWT 签名密钥分离，且不出现在 access token 内**（复制 Authorization 不足以铸签名）。
- 双重 HMAC：先用 `requestKey` 签 `ts/3600`（小时键）作 base，再签消息。
- 时间窗：±60s~+300s（可配）防重放；nonce 用 `Cache.SetNX(reqnonce:<uid>:<sid>:<nonce>, ttl)` **原子消费**，跨副本字节级重放即拒绝。
- 豁免（GET 原生资源，浏览器无法附加自定义头）：音频 WebSocket、`/api/files/:id`、`/api/artifacts/:id`、`/api/documents/:id/content`、conversation/admin sandbox file、`/api/me/ppt/resource`、`/api/me/files/content`（`requestSignatureExempt`，`middleware.go:598`）。

### 6.4 其它安全措施
- **CSRF 双层**：`csrfOK`（cookie-auth 变更请求）+ `csrfRefreshOK`（仅 refresh cookie 的 /auth/refresh、/logout、/auth/session 探测）。同源比较含 scheme；`Sec-Fetch-Site: cross-site`（无 Origin）视为攻击。
- **bcrypt 密码** + 不存在邮箱走 dummy hash 等时比较。
- **验证码/票据烧毁**：发码、2FA、passkey 失败达阈值即删除授权。
- **TOTP 重放**：`SetNX("2fa:used:<uid>:<code>", 90s)`。
- **请求体 4MB** 上限、错误日志不外泄内部细节（`mux.go:117-132`）。
- **CORS 回显式** + 明确允许签名字段。
- `security_config.go`、`domain_access.go`（企业域名白名单）、`admin_audit_middleware.go`（审计/离线审计日志）。
- `clientIP` 防 XFF 伪造；OAuth 端点经 `ValidateHTTPSProviderEndpoint` + DNS-rebinding-safe client + 禁重定向（`oauth.go:116-122`）。

---

## 7. Python 复刻要点（FastAPI）

> 目标是**协议/字段/时序等价**。下表 Go → Python 对应。

### 7.1 技术栈映射
| 职责 | Go 原实现 | Python/FastAPI 建议 |
|---|---|---|
| Web 框架 | `net/http` + 手写 mux | **FastAPI**（Starlette router，路径参数比 mux 更强，天然支持 `:id` → `/{id}`） |
| 路由注册 | `mux.handle(method, path, handler)` | `@app.api_route(path, methods=[...])` / `APIRouter` 按资源分组 |
| 依赖注入 | `Deps` 结构体注入 `NewRouter(d)` | **FastAPI `Depends()`**（`get_db`、`get_current_user`、`get_current_admin`、`require_capability`） |
| Access JWT | golang-jwt/v5 HS256 | **pyjwt**（或 python-jose），HS256，`typ`/`token_use`/claims 完全对齐 |
| 密码哈希 | bcrypt（store.HashPassword/CheckPassword） | **passlib[bcrypt]**（或 argon2） |
| TOTP | 自研 `auth/totp.go` (RFC 6238) | **pyotp**（或自实现，参数：SHA1/6位/30s） |
| WebAuthn | go-webauthn/webauthn | **fido2**（`Fido2Server`、`RelyingParty`；无用户名 discoverable → `begin_discoverable_login`） |
| OAuth/OIDC | 自研 `internal/oauth` | **authlib**（`OAuth2Client`/`OIDCClient`，PKCE/state/nonce/JWKS 校验） |
| 限流 | Cache.Incr 固定窗口 | 自研 `RateLimitMiddleware` + Redis INCR（或 slowapi），key 格式对齐 |
| 缓存 | `cache.Cache` 抽象（Redis/内存） | **Redis**（shared）或内存回退 |
| HTTP 客户端 | 自研（DNS-rebinding-safe） | `httpx` + 定制 transport（鉴权/禁重定向） |

### 7.2 必须保持的协议/字段（前端依赖，勿改）
- **请求**：`POST /api/auth/login|register|refresh|session|logout|…` 的 JSON body 字段（email/password/name/locale/captcha_token/phone 等）。
- **响应 JSON 字段（前端硬依赖）**：
  - `authResp`：`{user, access_token, request_signing_key, expires_at}`（`auth_handlers.go:213`）。
  - `authSessionResp`：`{authenticated, user?, access_token?, request_signing_key?, expires_at?, auth_policy?}`。
  - 登录 2FA 门：`{totp_required:true, ticket}`。
  - 未授权：`{"error": "..."}`；限流：429 + `Retry-After` 头；站点关闭：503 `{error:"site_closed", closed:true, message}`。
  - `publicAuthPolicy`字段：`password_login_enabled, passkey_login_enabled, entry_mode, default_provider, oauth_initial_password_policy, oauth_auto_provision_enabled, providers`（`auth_policy.go:49`）。
- **Cookie**：`auth_token`（HttpOnly, Path `/`, Lax, Secure 视连接）、`refresh_token`（HttpOnly, **Path `/api/auth`**, Lax）；`aivory_2fa`、`aivory_oauth_*`。
- **限流 Headers/参数名**：`Retry-After`；`retry_after` 字段。
- **JWT claims**：`uid, role, tv, sid, token_use, sub, exp, iat, jti`；`typ`(at+jwt/rt+jwt)；HS256。
- **会话后缀**：access token 的 `sid` 绑定数据库会话族；`token_ver` 必须与 DB 强校验。
- **请求签名头**：`X-Req-Ts / X-Req-Nonce / X-Req-Token / X-Device-Id / X-Req-Content-SHA256`（若启用 RequestSignaturesRequired 则必须实现，否则前端所有浏览器请求被 403）。
- **WebAuthn options**：`{ticket, options}`（登录）、`{session, ...}` 缓存的挑战结构——options 是标准 `publicKey` 对象，须由 fido2 生成与校验。

### 7.3 FastAPI 鉴权依赖实现建议
```python
# 语义对应 requireAuth (middleware.go:73)
async def get_current_user(request: Request, db=Depends(get_db),
                           auth: AuthService = Depends(get_auth)) -> User:
    token = read_access_token(request)              # Bearer or auth_token cookie
    if not token: raise HTTPException(401, "auth_required")
    claims = auth.parse_access(token)               # pyjwt, 校验 typ/token_use/sub==uid
    # 请求签名校验（若启用）——见 6.3
    state = get_user_auth_state_for_session(db, claims.uid, claims.session_id)
    if state.status != "active": raise HTTPException(403, "account_blocked")
    if state.token_ver != claims.tv: raise HTTPException(401, "session_expired")
    user = get_cached_user(db, claims.uid)
    user.role, user.status, user.token_ver = state.role, state.status, state.token_ver
    return user

# get_current_admin = 依赖 get_current_user 后校验 role=="admin"
# require_capability(perm) = 依赖 get_current_user 后校验用户组权限
```
- 用 `Depends` 实现与 Go 相同的关键点：**授权输入字段（role/status/token_ver）必须覆盖缓存值**；CSRF 检查放在状态变更依赖里。
- `get_current_user_optional` 用于桌面授权等场景。

### 7.4 复刻清单（按文件）
| Go 文件 | FastAPI 对应 |
|---|---|
| `router.go` | `app/main.py` + `api/routers/*.py`（按 auth/me/conversations/workspaces/admin/kbs/… 拆 APIRouter） |
| `mux.go` | FastAPI 原生路由（无需自写 mux）；保留首次匹配顺序注意点已无） |
| `middleware.go` | `core/deps.py`（get_current_user/admin/capability）+ `middleware/*.py`（CORS、site_access、限流、请求日志） |
| `auth/auth.go` | `core/auth.py`（JWT 签发/解析、密钥派生、RequestSigningKey） |
| `auth/totp.go` | `core/totp.py`（或 pyotp） |
| `auth_handlers.go` / `twofa_handlers.go` / `password_reset.go` | `api/routers/auth.py` + `services/` |
| `passkey_handlers.go` / `passkey_service.go` | `api/routers/passkey.py` + `services/passkey.py`（fido2） |
| `oauth.go` / `oidc_token.go` / `oauth_handlers.go` | `api/routers/oauth.py` + `services/oauth.py`（authlib） |
| `captcha.go` | `services/captcha.py`（Pillow 绘拼图 + 轨迹校验） |
| `refresh_tokens.go` | `services/sessions.py`（旋转/重放检测 SQL） |
| `permissions.go` | `core/permissions.py`（require_capability / require_knowledge_base） |
| `auth_policy.go` / `site_access.go` / `domain_access.go` | `core/policy.py` |

### 7.5 复刻时的高风险/易错点
1. **请求签名（v2）**若启用必须一字不差复刻 HMAC 消息拼接、小时键、nonce 原子消费 —— 否则浏览器一个都不通过。
2. **refresh token 旋转 + 重放检测**：必须事务化（锁用户行、字面 `token_ver=token_ver` 取锁、重放撤销整个会话族）。用 SQLAlchemy 事务 + `SELECT ... FOR UPDATE` 或行级锁模拟。
3. **token_ver 强校验**：改密/重置/撤销必须 bump，且 access 校验必须查 DB 状态（不能只信 JWT 自身）。
4. **防枚举/等时**：不存在邮箱登录走 dummy bcrypt、发码/忘记密码统一响应统一冷却。
5. **CORS**：回显式 + credentials，允许签名字段，OPTIONS 204。
6. **per-IP 限流 + XFF 取右至左**（防伪造）必须保留。
7. **OAuth**：state/nonce/PKCE/浏览器绑定 cookie/跨域 handoff/open-redirect 门/subject 命名空间/提供商回调 guard——遗漏任一都破坏安全。
8. **Cookie SameSite/Path**：`refresh_token` Path=`/api/auth` 是前端与安全双重约定，别改。

---

## 附录：关键文件索引
- `server/internal/api/router.go` — 441 条路由、限流预算、中间件组装、CORS、SPA
- `server/internal/api/mux.go` — 手写微型路由
- `server/internal/api/middleware.go` — requireAuth/requireAdmin/CSRF/限流/请求签名/clientIP
- `server/internal/api/response_logging.go` — 响应日志 / ResponseWriter 能力矩阵
- `server/internal/api/auth_handlers.go` — 注册/登录/刷新/会话/登出/cookie
- `server/internal/api/twofa_handlers.go` — 2FA 设置与登录票据
- `server/internal/api/passkey_handlers.go`、`passkey_service.go` — WebAuthn
- `server/internal/api/password_reset.go` — 忘记/重置
- `server/internal/api/oauth_handlers.go` — OAuth 回调/跨域/绑定
- `server/internal/oauth/oauth.go`、`oidc_token.go` — 提供商/ID token 验证
- `server/internal/api/captcha.go` — 滑块验证码
- `server/internal/api/auth_policy.go`、`permissions.go`、`site_access.go`、`domain_access.go`
- `server/internal/auth/auth.go`、`totp.go` — JWT/TOTP
- `server/internal/store/refresh_tokens.go` — 会话旋转/重放检测
