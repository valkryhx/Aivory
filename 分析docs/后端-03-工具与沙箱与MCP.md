# Aivory 后端调研：Agent 工具注册表 + 沙箱 + MCP + SSE（复刻要点）

> 目标：为「Python/FastAPI 复刻」提供 Go 后端这几个子系统的深度源码级剖析。
> 覆盖目录：
> - `server/internal/tools/`（Registry、内置工具、日志诊断）
> - `server/internal/sandbox/`（Python 沙箱 HTTP 客户端）
> - `server/internal/mcp/`（MCP 客户端传输层）
> - `server/internal/sse/`（SSE 写入辅助）
> - `server/internal/msgcache/`（消息增量缓存）
> - `server/internal/tooldiagnostics/`（工具诊断采样）
> - `server/internal/toolnames/`（工具名常量）
> - API 侧：`server/internal/api/user_mcp_handlers.go`、`server/internal/api/admin_mcp_handlers.go`
> - LLM 编排侧：`server/internal/llm/tool_exec.go`、`server/internal/llm/orchestrator.go`、`server/internal/llm/types.go`
>
> 所有行号基于当前 `main` 分支源码，引用格式 `文件:行号`。

---

## 0. 一句话架构

Aivory 的 Agent 工具是一个「统一自建工具层」（design.md §4.2）：

```
Tool 接口（Name / Description / InputSchema / Execute）
   └─ Registry（模型感知的全局注册表：tools map + MCP bindings）
         ├─ List()      → [llm.ToolDef]（function-calling schema，供各 provider 使用）
         ├─ ListMCP()   → [llm.MCPToolDef]（把远程 MCP 工具折叠成合成 Function 名）
         └─ Run()       → (text, []llm.Citation, error)（执行 + 日志 + 诊断 包装）
         └─ runMCP()    → 走 mcp.Client → tools/call
```

- **编排器（orchestrator）不知道工具内部是什么**，它只把 `input []byte`（JSON）交给 `Tool.Execute`，拿回 `text + citations + err`。`server/internal/tools/registry.go:1-5` 注释明确：「The orchestrator does not know what the tools are — it just hands them inputs and gets back text + citations.」
- 本地工具与远程 MCP 工具在**同一个 Function 命名空间**里共存，靠 `mcpBindings` map 区分。

---

## 1. 工具注册表架构

### 1.1 Tool 接口（工具的"定义/契约"）

`server/internal/tools/registry.go:33-39`：

```go
// Tool is the contract every self-built tool implements.
type Tool interface {
	Name() string
	Description() string
	InputSchema() json.RawMessage
	Execute(ctx context.Context, input []byte, tc *llm.ToolContext) (text string, citations []llm.Citation, err error)
}
```

- `Name()`：工具名（如 `python_execute`、`aivory_web_search`）。
- `Description()`：给模型的自然语言说明（很长，属于 prompt 工程）。
- `InputSchema()`：返回 JSON Schema（`json.RawMessage`），即 function-calling schema，直接透传给各 LLM provider。
- `Execute(ctx, input []byte, tc *llm.ToolContext)`：执行，输入是 **JSON 字节**（每个工具自己 `json.Unmarshal`），输出是**文本**（工具结果是文本字符串）+ **引用列表**（`[]llm.Citation`）+ 错误。

> Python 复刻对应：`Tool` 接口 ≈ Pydantic 模型 + 函数装饰器（见第 7 节）。

### 1.2 Registry 结构体

`server/internal/tools/registry.go:41-67`：

```go
type Registry struct {
	mu      sync.RWMutex
	tools   map[string]Tool              // 本地内置工具：name -> Tool
	cfg     config.Config
	db      *sql.DB
	logger  *log.Logger
	sandbox sandbox.Service              // python_execute 依赖
	mcpBindings map[string]mcpBinding    // 合成 Function 名 -> 远程 MCP 绑定
	mcpClients  map[string]cachedMCPClient // 会话缓存
	mcpGenerations map[string]uint64     // 失效代数（防陈旧客户端）
	userMCPHTTPClient *http.Client
	toolLogOnce / toolLogs / toolLogMu / toolLogClosed / toolLogDone  // 工具日志异步写入
}
```

关键设计：
- `mcpBindings map[string]mcpBinding`：**模型可见的 Function 名 → 受信的服务 + 远端方法**。URL / headers 永远不进模型可控输入（`registry.go:50-51`）。
- `mcpBindings` 结构（`registry.go:69-86`）携带 `RuntimeFingerprint`（执行前重新读 DB 比对，跨进程也能关闭陈旧绑定窗口）、`WorkspaceID`/`OwnerID`（用户 MCP 的租户边界）。

### 1.3 工具如何注册

`NewRegistry` 把每个内置工具实例注册进去（`registry.go:140-164`）：

```go
func NewRegistry(db *sql.DB, cfg config.Config, logger *log.Logger) *Registry {
	r := &Registry{ ... }
	sb := newSettingsSandbox(db, cfg.SandboxBaseURL, cfg.SandboxAPIKey)   // 沙箱设置包装
	search := newSettingsSearcher(db, cfg.SearchProvider, cfg.SearchAPIKey, cfg.SearchBaseURL)
	r.Register(&webSearchTool{cfg: cfg, searcher: search})
	r.Register(&webFetchTool{db: db, cfg: cfg})
	r.Register(&fetchImageTool{sandbox: sb, logger: logger})
	r.Register(&pythonExecuteTool{sandbox: sb, uploadDir: cfg.UploadDir, artifactDir: cfg.ArtifactDir, logger: logger})
	r.Register(&imageGenerateTool{db: db, uploadDir: cfg.UploadDir, artifactDir: cfg.ArtifactDir, logger: logger})
	r.Register(&useSkillTool{db: db})
	r.Register(&saveMemoryTool{db: db})
	return r
}
```

`Register` 只是放进 map（`registry.go:167-171`）：

```go
func (r *Registry) Register(t Tool) {
	r.mu.Lock()
	r.tools[t.Name()] = t
	r.mu.Unlock()
}
```

### 1.4 列表：工具 JSON 描述（function calling schema）

`List` / `ListRegistered`（`registry.go:178-187`），内部走 `listRegistered`（`registry.go:205-231`）：

```go
func (r *Registry) listRegistered(includeUnavailable bool) []llm.ToolDef {
	sandboxEnabled := r.sandbox != nil && r.sandbox.Enabled()
	imageGenerationEnabled := r.ImageGenerationConfigured()
	promptOverrides := systemprompts.Load(r.db)
	...
	for _, t := range r.tools {
		if !includeUnavailable && t.Name() == "python_execute" && !sandboxEnabled { continue }
		if !includeUnavailable && t.Name() == "image_generate" && !imageGenerationEnabled { continue }
		description := promptOverrides.Text("tools.description." + t.Name())
		if description == "" { description = t.Description() }
		out = append(out, llm.ToolDef{
			Name: t.Name(), Description: description, InputSchema: t.InputSchema(),
		})
	}
	sort.Slice(out, func(i, j int) bool { return out[i].Name < out[j].Name })
	return out
}
```

- **运行时依赖不可用的工具会被隐藏**（`python_execute` 无沙箱、`image_generate` 无 image 模型），避免模型选一个跑不起来的操作。
- 描述可用系统提示词覆盖（`tools.description.<name>`）；`ListRegistered`（管理端能力页用）则保留所有工具以说明需配置什么。
- `ToolDef`（`server/internal/llm/types.go:255-260`）：

```go
type ToolDef struct {
	Name        string          `json:"name"`
	Description string          `json:"description"`
	InputSchema json.RawMessage `json:"input_schema"`
}
```

这就是最朴素的 **function calling schema**：`name` + `description` + `input_schema`（JSON Schema object）。

### 1.5 工具日志 / 诊断

工具执行日志由 `Registry.Run`（`server/internal/tools/call_logging.go:174-225`）统一包装：

- 每个调用先 `tooldiagnostics.Start(ctx, bodies)` 建立诊断采样器，捕获输入/输出/HTTP 请求细节并脱敏。
- `defer` 里根据结果写 `store.ToolCallLog`（工具名、用户/会话/消息/工作区/模型 ID、状态 `success/error/timeout/canceled/partial`、耗时、入参/出参截断、错误）。
- 通过异步 channel `enqueueToolLog`（`call_logging.go:35-84`）批量落库，队列满则丢弃并记日志。
- 受两个设置控制：`log_request_bodies`（是否记录请求体，默认 true）、`log_tool_errors_only`（是否只记错误，默认 true）。
- 另有 `LogToolFailure`（`call_logging.go:91-129`）记录编排器执行跟踪器发现但没走到 `Run` 的失败（重复请求、重复失败路由、无新证据的搜索等）。

`tooldiagnostics`（`server/internal/tooldiagnostics/diagnostics.go`）负责：
- 采样工具发出的 HTTP 请求（方法、URL、状态码、耗时、请求/响应体），并把 body 里出现的 secret 值（API key、token、password、authorization 等）替换成 `[redacted]`。
- `Collector.Sanitize`（`diagnostics.go:170-224`）递归处理 JSON、URL 脱敏、base64 二进制脱敏。
- 管理端据此展示「这一个工具调用到底对外发了什么、收到什么」且不泄露凭据。

> Python 复刻要点：日志/诊断是**横切关注点**，建议用 FastAPI 的依赖注入或装饰器统一包裹每个工具调用，脱敏逻辑复用同一套正则/规则。

---

## 2. 内置工具清单与能力

内置工具共 **7 个**（`tools/` 包内实现），外加可选的远程 **MCP 工具**（动态合成）与 **provider-hosted 官方工具**（由模型配置驱动，不属于本地 Registry）。

| 工具名 | 注册位置 | 用途 | InputSchema |
|---|---|---|---|
| `aivory_web_search` | `builtins.go:156`（`webSearchTool`） | 联网搜索（可批量、可搜图） | `registry.go` 里动态拼，见下 |
| `web_fetch` | `builtins.go:367`（`webFetchTool`） | 抓取网页正文 + 图片元数据，SSRF 防护 + Jina 兜底 | `{"type":"object","properties":{"url","urls","show_images"}}` |
| `fetch_image` | `builtins.go:768` | 下载公开图片进沙箱 `/workspace/downloads/` | `{"type":"object","properties":{"url","filename"},"required":["url"]}` |
| `python_execute` | `builtins.go:1186` | 在持久 Python 沙箱跑代码（数学/数据分析/图像编辑/绘图/文档编辑/生成下载文件） | `{"type":"object","properties":{"code":{"type":"string","minLength":1}},"required":["code"]}` |
| `image_generate` | `builtins.go:1757` | 图生图 / 文生图（Gemini / OpenAI 双通道） | `{"type":"object","properties":{"prompt","action","base_image","base_image_index","n","size"},"required":["prompt","action","base_image"]}` |
| `use_skill` | `builtins.go:3660` | 加载已注册技能的完整说明 | `{"type":"object","properties":{"name"},"required":["name"]}` |
| `save_memory` | `builtins.go:3752` | 持久化用户长期记忆 | `{"type":"object","properties":{"memory_text","slot","value"},"required":["memory_text"]}` |

### 2.1 web_search 的动态 schema（值得注意）

`webSearchTool.InputSchema`（`builtins.go:116-146`）是**运行时根据实际搜索后端动态生成**的：基础有 `query`/`queries`/`top_k`/`show_images`，再根据 Searcher 是否实现 `searcherWithOptions` 追加过滤参数；searxng 配置了结果数则删掉 `top_k` 避免模型覆盖管理员设置。这体现了 **schema 随可执行能力动态变化**的设计。

`toolnames` 包（`server/internal/toolnames/names.go:6-9`）持有稳定名字：

```go
const (
	AivoryWebSearch       = "aivory_web_search"
	LegacyAivoryWebSearch = "web_search"
)
```

### 2.2 image_generate（最复杂的内置工具）

`imageGenerateTool.Execute`（`builtins.go:1832-2163`）体现了工具分层鉴权 + 计费 + 配额 + 内容安全 + 多通道 fallback 的完整模式：
- 解析并严格校验输入（action/base_image/base_image_index、尺寸合规、n 上限）。
- `resolveImageModel`（`builtins.go:2185`）：用用户预选 image 模型或第一个启用 image 模型。
- **内容安全**：`moderateImagePrompt`（`builtins.go:2002`）对最终 provider prompt 做自查。
- **配额/计费**：每日图片限额（`checkDailyImageLimit`，`quota_ledger` 原子预留）、每模型 quota、`ImageBilling`（free→credits→block，`builtins.go:1941-1975`）。
- **双通道**：`geminiGenerateImages` / `openaiGenerateImages`（`builtins.go:2043-2050`），按 channel type 路由；主通道失败走 `resolveImageFallbackChannels` 逐个兜底（`builtins.go:2065-2082`）。
- **持久化**：`saveArtifact` 写 ArtifactDir + 记录 artifact 行 + `tc.OnArtifact` 回调触发 SSE `artifact` 事件（`builtins.go:1561-1623`）。

> Python 复刻：image_generate 的**分层鉴权/配额逻辑与工具本体解耦**，建议复刻时把「加载模型 → 配额预检 → 内容安全 → 调 provider → fallback → 计费/持久化」做成可复用的管线，装饰器只做 schema 定义。

---

## 3. 沙箱集成协议（HTTP 契约）

### 3.1 哲学：产品不内嵌运行时，只代理

`sandbox/sandbox.go:1-19` 注释明确：**不内嵌 Python 运行时，代理到一个现成的沙箱服务**（OpenTerminal / E2B / 任意暴露 HTTP 的 runner），通过切 `SANDBOX_BASE_URL` 换后端，代码其他部分不用动。

协议刻意保持极小（任何 runner 都能实现）：

```
POST {BaseURL}/sessions            -> {"session_id":"..."}
POST {BaseURL}/exec  {session_id, code, timeout_ms}
     -> {"stdout":"...","stderr":"...","exit_code":0,
         "files":[{"name":"plot.png","mime_type":"image/png","data_base64":"..."}]}
POST {BaseURL}/files/reset-inputs {session_id}
     -> {"ok":true,"session_gone":false}  或  {"ok":false,"session_gone":true}
```

- `files` 是运行中写入 `/workspace/outputs` 的产物。
- 配置了 `SANDBOX_API_KEY` 时附加 `Authorization: Bearer <key>`。
- 产物用 **base64** 在 wire 上传回。

### 3.2 Service 接口（契约全集）

`sandbox/sandbox.go:119-165` 定义了 `sandbox.Service` 接口：`Enabled / NewSession / Exec / PutFile / ResetInputs / GetFile / ListFiles / Release / ReleaseDiscard / PruneArchives`。

`HTTPSandbox`（`sandbox.go:167-187`）是默认实现，字段含 `BaseURL / APIKey / Storage *StorageConfig / ExecTimeout / IdleTTL / client`。

### 3.3 HTTP 方法实现（sidecar 协议）

统一 `do` / `doMethod`（`sandbox.go:254-287`）：POST/DELETE JSON，带 `Authorization: Bearer`，4xx/5xx 包装成 `HTTPError{StatusCode, Body}`，响应体用 `maxSandboxRespBytes`（默认 256MB）限流防 OOM。

各端点映射：

| Service 方法 | HTTP 调用 | 说明 |
|---|---|---|
| `NewSession(ctx, archiveKey)` | `POST /sessions`，body 可带 `storage`/`idle_ttl_sec`/`archive_key` | 创建会话，返回 `session_id`（`sandbox.go:290-311`） |
| `PutFile(ctx, sessionID, path, data)` | `POST /files` `{session_id, path, data_base64}` | 写入 sidecar workspace（`sandbox.go:315-321`） |
| `ResetInputs(ctx, sessionID)` | `POST /files/reset-inputs` | 清空 `/workspace/uploads` 与 `/workspace/skills` 并重建，返回 `session_gone`（`sandbox.go:326-339`） |
| `Exec(ctx, sessionID, code)` | `POST /exec` `{session_id, code, timeout_ms}` | 执行代码，返回 stdout/stderr/exit_code/files（`sandbox.go:342-371`） |
| `GetFile(ctx, sessionID, path)` | `POST /files/get` `{session_id,path}` | 取回 workspace 中某个文件（`sandbox.go:376-390`） |
| `ListFiles(ctx, sessionID)` | `POST /files/list` | 列出 `/workspace` 下文件（`sandbox.go:400-412`） |
| `Release(ctx, sessionID)` | `DELETE /sessions/{id}`（带 storage） | 主动销毁并**归档** workspace（`sandbox.go:415-424`） |
| `ReleaseDiscard(ctx, sessionID, archiveKey)` | `DELETE /sessions/{id}` body `{discard:true, storage, archive_key}`；session 为空则 `DELETE /archives/{archiveKey}` | 不归档销毁，并删除已有归档（「clear sandbox」真删除，`sandbox.go:429-444`） |
| `PruneArchives(ctx, maxAge)` | `POST /storage/gc` `{storage, max_age_seconds}` | 清理过期 workspace 归档（`sandbox.go:449-465`） |
| `ListArchivedFiles/GetArchivedFile` | `POST /archives/files/list`、`POST /archives/files/get` | 只读读取已归档 workspace（`archive_browser.go:78-86`） |

> 补充说明：沙箱 `app.py` 暴露的是**两组并存的端点**——(1) exec 会话管理走上面的 `POST /sessions`、`POST /exec`、`POST /files*`、`DELETE /sessions/{id}`、`DELETE /archives/{key}`（`sandbox.go` 的会话接口）；(2) **workspace 归档的持久化存储 sidecar** 另有 `POST /storage/put`、`POST /storage/delete`、`POST /storage/gc`、`POST /archives/files/list|get`（MinerU 摄取把文件经 `/storage/put` 写入对象存储、`/storage/delete` 清理，`archive_browser.go` 只读归档）。两者不互斥，都是 sandbox-service 真实协议（`app.py` 中均有对应路由）。

### 3.4 会话生命周期与「会话消失」恢复

- `ErrSessionGone`（`sandbox.go:42`）：sidecar 空闲回收器已删掉会话。`python_execute` 视为**预期恢复信号**：重建会话 + 重新 staging 权威输入 + 重试一次。
- `isSandboxSessionRecoverable`：基于 `HTTPError.StatusCode==404`（`archive_browser.go:28-44` 的 `IsSessionGone`，识别 `session not found` / `session not running`）。
- `python_execute.Execute`（`builtins.go:1198-1477`）完整流程：
  1. 读 `sandbox_id`（`store.GetConvProviderStateKey`）；
  2. 没有则 `NewSession(ctx, convID)`（archiveKey=会话id=会话ID，稳定键归档），并持久化 `sandbox_id`；持久化失败则 `releaseUntrackedSandbox` 释放，避免泄漏容器；
  3. `stageFiles`：先 `ResetInputs`，再把会话的上传区文件、图片生成产物、skill 资产逐个 `PutFile` 到 `/workspace/uploads|skills`；文件夹上传写 manifest `/workspace/aivory-uploads.txt`；
  4. `t.sandbox.Exec(sessionID, code)`；
  5. 产物 `saveArtifact`（Source=`ArtifactSourcePythonExecute`）+ `tc.OnArtifact` 触发 SSE；
  6. **截断输出** stdout/stderr 到 `pythonExecuteStdoutStderrTruncationCap`（默认 32KB，Pitfall A5 防刷上下文）；
  7. 返回文本给模型（含 `Produced files:` 列表，提示「文件已自动附加，用文件名引用，别用 sandbox:/workspace/outputs 路径」）。
- `contextEnded` / `abandonPythonSession`（`builtins.go:1532-1530`）：客户端取消时清掉 `sandbox_id`，避免后续请求排队在一个已无人消费的阻塞 handler 后。

### 3.5 每次调用按会话锁串行化

`lockConvSandbox`（`builtins.go:1169-1174`）用 `sync.Map` 按会话 id 持有一个 mutex，保证 `get→create→persist` 原子，防止一轮并发 `python_execute` 都看到空 `sandbox_id` 各自 NewSession 互相覆盖。

### 3.6 python_execute 启用条件 & 安全降级

- `Enabled()` 取决于 `BaseURL != ""`（`sandbox.go:252`），即 **配置了 `SANDBOX_BASE_URL`（或管理设置 `sandbox_base_url`）**。
- 沙箱配置来自**管理设置**，每次调用重读（`sandbox_settings.go:37-57`），键名：`sandbox_base_url`、`sandbox_api_key`、`sandbox_idle_ttl_sec`、`sandbox_exec_timeout_sec`，以及归档存储 `storage_provider`（s3/aliyun_oss/local）等。env 只是 boot 默认值。
- 未配置时 `python_execute` 走 **safe-mode**（`builtins.go:1206-1217`）：只对 `print(expr)` 做本地算术求值（`tryQuickArithmetic`，一个极小的 shunting-yard，`builtins.go:1642`），其余返回提示「Configure a sandbox URL + key in Admin settings」。
- `Registry.listRegistered` 在 `python_execute` 依赖不可用时直接**不列出**该工具；执行时 `checkCurrentToolAccess` 再拦一次（`registry.go:655-657`）。

---

## 4. MCP 支持机制

### 4.1 MCP 客户端传输层（`internal/mcp/`）

包注释：`mcp` 实现 **MCP Streamable HTTP client transport**（`mcp/types.go:1-2`）。

- `Config`（`client.go:32-38`）：`URL`（受信、管理员控制）+ `Headers`（**构造时拷贝，工具调用参数不能改**）+ `Timeout`（默认 30s）+ `MaxResponseBytes`（默认 8MB）+ 可注入 `HTTPClient`（用于传输策略/测试）。
- `NewClient`（`client.go:60-100`）校验 endpoint：必须是绝对 http/https、无 userinfo、**无 query、无 fragment**（防凭证泄漏到日志，`client.go:381-410`）；header 名/值校验，拒绝保留头（`accept/content-type/host/mcp-*` 等，`client.go:425-433`）。

**双协议发现**（`client.go:105-193`）：
1. **现代无状态协议** `server/discover`（`ModernProtocolVersion="2026-07-28"`），POST 带 `_meta.protocolVersion` + `Mcp-Method` 头。
2. 失败则**回退到传统 initialize 握手**（遍历 `2025-11-25 / 2025-03-26 / 2024-11-05` 三个版本），`initialize` → `notifications/initialized`（session 化）。
- `shouldTryLegacy`（`client.go:441-459`）：仅当错误码是 `-32600/-32601/-32602` 或 HTTP 4xx 中的特定几种才回退；认证错误不回退。
- 协议版本强制为 `YYYY-MM-DD` 格式（`normalizeProtocolVersion`，`client.go:195-221`），防敌意服务器把配置的凭据反射进 protocol 元数据。

**方法**：
- `ListTools(ctx)`（`client.go:225-272`）：`tools/list`，跟踪 cursor 分页（上限 `maxListPages=100`，累计响应字节受 `MaxResponseBytes` 限制）。
- `CallTool(ctx, name, arguments)`（`client.go:276-305`）：`tools/call` `{name, arguments}`，arguments 必须是 JSON object（`normalizeArguments`，`client.go:366-379`）。

**协议封装**（`transport.go`）：
- `rpcCall`（`transport.go:45-72`）：构造 JSON-RPC 2.0 请求 → `post` → 校验响应 id → 解析 result/error。
- `post`（`transport.go:89-145`）设置 `Mcp-Protocol-Version`、`Mcp-Session-Id`（session 化时）、`Mcp-Method`、`Mcp-Name` 头，接受 `application/json, text/event-stream`，**同时支持 JSON 与 SSE 响应体**（`decodeRPCResponse`，`transport.go:158-206`；`parseSSEData` 解析 `data:` 行）。
- `rpcNotify`（`transport.go:74-87`）：notification（无 id 的 JSON-RPC）。

**类型**（`types.go`）：
- `Tool`（`types.go:51-60`）：`Name/Title/Description/InputSchema/OutputSchema/Annotations/Meta/Icons`，JSON Schema 以 `json.RawMessage` 保留（无损传给 provider）。
- `CallResult`（`types.go:89-94`）：`Content []Content + StructuredContent + IsError + Meta`。
- `TextContent()`（`types.go:98-121`）：拼接 text / resource 文本，若只有 structuredContent 则返回其紧凑 JSON。
- `RPCError / HTTPError / ResponseTooLargeError`。

### 4.2 Registry 如何把 MCP 工具折叠成 Function

`ListMCP(_, userID, workspaceID)`（`registry.go:263-350`）返回当前可用的 MCP 工具定义：
- 读到管理员启用的 MCP 服务（`store.ListMCPServer(false)`）+ 用户的个人/活跃工作区服务（`store.ListUserMCPServersScoped`）。
- 工作区调用时校验 `WorkspacePolicy`（`AllowToolCalling`/`AllowMCP`/成员 `CanUseMCP`）、按服务器 `ToolDeniedByPolicy("mcp:"+id)` 过滤。
- `mergeMCPDefinitions`（`registry.go:352-455`）：把每个远端工具合成一个 `llm.MCPToolDef`，并写进 `mcpBindings[functionName]`。

**Function 命名**（模型可见，安全关键）`mcpFunctionName`（`registry.go:536-566`）：
- 保留区间 `[A-Za-z0-9_-]`、≤64 字节；
- 用 `sha256(serverID + "\0" + remoteName)` 的前 6 字节十六进制做后缀，防跨服务碰撞且不暴露 DB id；
- 名字形如 `mcp_<cleaned-name>_<hex>`；重名时递增 attempt。
- 声明全集身份 `mcpFunctionDeclarationIdentity`（`registry.go:471-478`）= 源身份长度+值 + runtime 指纹长度+值，把「Function 名」与「端点/凭据/协议/快照」绑定，配置一变就产生新名字、旧名字失效（防陈旧绑定重绑到新端点）。

**运行时指纹** `mcpRuntimeFingerprint`（`registry.go:1048-1069`）：对 `ID/URL/Headers/Enabled/ProtocolVersion/LastSyncedAt/DiscoveredTools/DisabledTools` 做 sha256，绑定/执行前比对，跨进程失效。

**失效与并发安全**：
- `InvalidateMCPServer(serverID)`（`registry.go:499-531`）：把 `mcpGenerations[serverID]` 递增代数，清掉该 server 的所有 bindings 与缓存客户端。代数机制关闭「失效回调未及时到达时旧客户端又被写回缓存」的窗口。
- `mcpClient`（`registry.go:1234-1294`）按 URL+headers+protocol+任意 lastSynced 组合缓存客户端；构造期间若代数变了则拒绝发布。

### 4.3 执行 MCP 工具（runMCP）

`runMCP`（`registry.go:813-898`）：
1. `loadRuntimeMCPServer` 加载权威行（管理员或用户表），并**重复** workspace / 组访问检查为这些声明。
2. 校验 `Enabled`、`ToolEnabled(binding.RemoteName)`、快照仍含该工具。
3. 取缓存客户端（`mcpClient`），网络调用前**再验一遍 state**（`verifyMCPClientState`，`registry.go:900-935`）：比对 binding 指纹、in-process 代数、DB 权威行。
4. `client.CallTool(ctx, RemoteName, input)`。
5. 失败时 `invalidateMCPClient` 只删当前实例（不留脏会话），错误经 `redactMCPError` 脱敏。
6. 结果：`text := TextContent()`；若 `IsError` 则当作工具错误；把 `content` 里的 text/resource（仅 http/https URL）转成 `llm.Citation`（`registry.go:868-893`）。

**凭据脱敏**：`redactMCPToolSnapshotMetadata` / `redactMCPXMLText` / `redactMCPJSONSnapshot`（`registry.go:1093-1216`）确保模型可见的 schema/名/描述里不出现配置的 header 值（如 Bearer token）；红色/掩码为 `[redacted]`。

**用户 MCP 的租户隔离**：
- `mcpFunctionSourceIdentity`（`registry.go:484-493`）：用户表用 `usermcp:<len>:<id>...`，防止管理员/用户表 id 复用。
- 用户 MCP 缓存 key 按 `userID + resourceWorkspaceID + executionWorkspaceID + serverID` 限定（`mcpUserCacheKey`，`registry.go:1316-1327`），远程 session 状态不能在用户/工作区之间串联。

### 4.4 管理员 / 用户如何添加 MCP 工具（API handler）

**用户 MCP**（`server/internal/api/user_mcp_handlers.go`）：
- `userMCPServerCreatePayload`（`user_mcp_handlers.go:24-32`）：`name/icon/description/url/headers/enabled/workspace_id`。
- 与 admin 共享元数据校验（`userMCPMetadataProxy` → `validateMCPServerMetadata`，`user_mcp_handlers.go:92-101`）。
- header 值是 **write-only**：响应里 `headers` 全部显示为掩码 `mcpHeaderMask`（`user_mcp_handlers.go:53-56`）。
- 发现结果 `discovered_tools` 经 `redactMCPToolSnapshot` 脱敏后才返回（`user_mcp_handlers.go:61`）。
- 工作区共享但成员无权时只给最小元数据（`userMCPServerListJSON`，`user_mcp_handlers.go:73-88`）。
- 选择/权限：`toolPolicyAllowsID`（permissions.go）按 **`usermcp:` 命名空间**校验；目录只对该命名空间列出可选。

**管理员 MCP**（`server/internal/api/admin_mcp_handlers.go`）：管理面板管理官方/租户级 MCP 服务——CRUD、启用/禁用、触发远端发现（`server/discover` 或 `initialize`）、存储 `discovered_tools` 快照、`disabled_tools` 清单、`last_error`、`last_synced_at`。发现是**显式的**（管理员/用户手动同步），聊天轮次不做远端 schema 发现（`registry.go:262` 注释）。

**注册/刷新流程**：创建或更新后写 `store.MCPServer` / `store.UserMCPServer` 行，并调用 `Registry.InvalidateMCPServer(serverID)`（改端点/凭据/启用态/快照/删除后）使内存中的合成 Function 失效 fail-closed。

> Python 复刻要点：MCP **客户端**（连外部 MCP server）直接用官方 `mcp` Python SDK（Streamable HTTP transport）。但 **Aivory 用的其实是"反转"模式：Aivory 是 client**，它把外部 MCP server 的工具伪装成本地 Function 数字（合成名）喂给主 LLM。复刻时：用 `mcp` SDK 建 `ClientSession`，`session.list_tools()` 同步快照入库，`session.call_tool()` 执行；合成 Function 命名与指纹校验逻辑照搬。

---

## 5. SSE / msgcache

### 5.1 SSE 写入辅助（`internal/sse/sse.go`）

`Writer` 包装 `http.ResponseWriter`，提供并发安全 `Send(event)`：
- `New`（`sse.go:25-39`）：校验底层支持 `http.Flusher`，设置 `content-type: text/event-stream`、`cache-control: no-cache, no-transform`、`connection: keep-alive`、`x-accel-buffering: no`，随即 `WriteHeader(200)+Flush`。
- `SendID(payload, eventName, id)`（`sse.go:49-70`）：写 `id:`（可选）、`event:`、`data: <json>`、空行，全部在 `mu` 锁内**串行化**（编排器从并发工具 goroutine + keep-alive goroutine 同时写，底层 ResponseWriter 非并发安全，必须加锁）。
- `Ping()`（`sse.go:73-81`）：写注释行 `: ping` 防代理超时。

```go
func (s *Writer) SendID(payload any, eventName, id string) error {
	...
	s.mu.Lock(); defer s.mu.Unlock()
	if id != ""   { s.w.Write([]byte("id: " + id + "\n")) }
	if eventName != "" { s.w.Write([]byte("event: " + eventName + "\n")) }
	s.w.Write([]byte("data: ")); s.w.Write(body); s.w.Write([]byte("\n\n"))
	s.flusher.Flush()
	return nil
}
```

### 5.2 事件类型枚举（`llm.SseEvent`）

on-the-wire 事件形状（`llm/types.go:280-317`）：`Source/Type/MessageID/Text/Name/ID/PartialJson/Input/Summary/URL/Title/Citation/StopReason/Usage/Message/Code/ToolID/Status/SourceCount/Credits/ThinkingMs` 等，全部 `lowercase, snake_case`。

从 provider 与编排器实际发出的事件（`Type` 字段取值）包括：
- `tool_start`（`Name`+`ID`+`Input`）：**每个工具调用先宣布**（`tool_exec.go:910`）。
- `tool_input`（`PartialJson`）：流式增量工具输入 JSON 片段（`anthropic_provider.go:1117`）。
- `tool_result`（`Name/ID/Summary/Status`）：工具结果展示（`anthropic_provider.go:543`、`openai_provider.go`，Summary 截断）。
- `artifact`：工具产出文件（沙箱输出 / 生成图片）已持久化（`OnArtifact` 触发，`orchestrator.go:4188-4230`）。
- `done`（带 `Usage`、`Credits`、可选 `ThinkingMs`）：回合结束。
- `delta`：文本增量。
- `verify` 系列：`verify_started/verify_finding/verify_done`（审计模式）。
- `tool_call` 等是**持久化 message block 的 Kind**（`UnifiedBlock`：`tool_call`/`tool_output`/`tool_result`/`artifact`/`document`/`attachment`），与 SSE 事件是两套东西。

### 5.3 消息增量缓存（`internal/msgcache/msgcache.go`）

`msgcache` 为 `store.ListMessages` 提供带版本的短缓存：
- `Bump(cache, convID)`（`msgcache.go:22-27`）：把该会话的版本号 `Incr`，**前缀无关**（`conv:ver:<convID>`），Redis 无需 SCAN/DEL。
- `ListMessages(ctx, cache, db, convID, leafID)`（`msgcache.go:32-55`）：
  - key = `conv:path:<convID>:<leafID>:<version>`，TTL `pathTTL=45s`；
  - version key TTL `versionKeyTTL=10min`；
  - 命中反序列化返回，未命中查 DB 后写缓存。
- 用途：active-path 水合、上下文组装等高频热读，靠 mutation 触发的版本 bump 保证有限陈旧（bounded staleness）。
- `versionKey`（`msgcache.go:57-59`）构造 `conv:ver:` key。

> Python 复刻要点：SSE 用 `sse-starlette` / FastAPI `StreamingResponse`（或 `sse-starlette` 的 `EventSourceResponse`），事件名枚举与 `SseEvent` JSON 字段照搬（这样前端不用改）。并发安全锁在 ASGI 下天然由事件循环串行化，但多 worker 时需注意。msgcache 用 Redis `INCR conv:ver:<conv>` + `GET/SET conv:path:...` 即可，算法照搬。

---

## 6. 工具调用生命周期（编排交互）

### 6.1 调用循环（tool-exec / runToolsConcurrent）

核心在 `server/internal/llm/tool_exec.go`：

```go
// tool_exec.go:906-942
func runToolsConcurrent(ctx context.Context, runner ToolRunner, calls []toolCallSpec, onEvent func(SseEvent)) []toolCallResult {
	results := make([]toolCallResult, len(calls))
	// 1) 先串行宣布所有 tool_start
	for _, c := range calls {
		onEvent(SseEvent{Type: "tool_start", Name: c.Name, ID: c.ID, Input: c.Input})
	}
	// 2) 信号量并发执行（maxConcurrentTools）
	for i, c := range calls {
		...
		go func(i int, c toolCallSpec) {
			defer func(){ if r := recover(); r != nil { results[i].Err = ... } }()
			out, cites, err := runner.Run(tooldiagnostics.WithCallID(ctx, c.ID), c.Name, c.Input)
			results[i] = toolCallResult{Output: out, Citations: cites, Err: err}
		}(i, c)
	}
	wg.Wait()
	return results
}
```

- 一轮里模型可发出**多个工具调用**，Aivory 用信号量**并发**执行（上限 `maxConcurrentTools`），每个调用在自己的 goroutine 内；panic 被 recover 并转成工具错误（防单工具崩溃拖垮整个 API 进程）。
- `ToolRunner` 接口由编排器实现；本地工具最终调到 `Registry.Run`，MCP 工具也走同一入口（由 `mcpBindings` 路由）。

### 6.2 参数与结果回传格式

- **入参**：provider 发出 tool_call → 编排器把原始 JSON（`Input`）直接传给 `Registry.Run(ctx, name, input, tc)`（`call_logging.go:174`）。`input` 是 JSON object 字节。
- **结果回填**：每个工具返回 `(text string, citations []llm.Citation, err error)`：
  - `text`：给模型的工具结果文本。
  - `citations`：跨源引用（`llm/types.go:56-73`：`ID/Index/Title/URL/Snippet/Source(web|kb|document)` + 可选的 `ImageURL/ThumbnailURL/ImageDisplay`），用于渲染引用与资料卡。
  - `err`：出错时 `toolCallResult.Err`；`llm.ToolUserError`（`tool_exec.go:369-371`）表示消息**可安全给模型看**的校验错误，其余错误在模型边界被替换为通用信息。
- 编排器用结果构造 `tool_output` 块与回填消息，再送回 provider 继续多轮 tool use（循环直到 `stop_reason` 非 tool_use 或到达预算）。
- 产出文件通过 `tc.OnArtifact(ArtifactRef)`（`orchestrator.go:194-196`）异步持久化并流 `artifact` SSE 事件（`ArtifactRef{Source/ID/Filename/URL/MimeType/Size}`，`types.go:331-338`）。

### 6.3 ToolContext（工具的执行上下文 / 鉴权载体）

`llm.ToolContext`（`orchestrator.go:120-209`）携带：`UserID/ConvID/MessageID/WorkspaceID/ModelID/DB`、`WorkspaceAccessCheck`（执行前重验工作区权限回调）、预算/计数字段、`BuiltinTools/SystemTools` 允许集、技能选择、图像模型与配额相关字段、`OnArtifact` 回调。`ToolContext.AllowsBuiltinTool`（`orchestrator.go:283`）是最终执行边界策略检查。

### 6.4 执行前/后的多层鉴权（安全边界）

`Registry.run`（`registry.go:593-638`）与 `checkCurrentToolAccess` / `checkWorkspaceToolAccess` / `checkMCPGroupAccess`（`registry.go:645-809`）形成多层防线：
1. 未知工具 / `tc` 空 / `ctx` 空 → 拒绝（防 panic、防绕过 MCP 私有端点）。
2. `AllowsBuiltinTool(name)`：只接受本轮声明的 Function（拒绝陈旧/主动投递的调用）。
3. 运行依赖检查（sandbox / image model）。
4. 工作区策略（AllowToolCalling / AllowMCP / AllowDrawing / AllowSkills / 服务器级 allowlist / 成员 CanUseMCP / CanUseSkills）。
5. 用户组工具策略（`builtin:<name>`、`mcp:<id>`、`usermcp:<id>` 的 ResourcePolicyAllows）。
6. 全局 `disabled_tools` 管理员禁用清单。
7. 图片/记忆/技能专属开关。
8. MCP 额外：loadRuntimeMCPServer 重读权威行（`registry.go:977-1040`）后**再次**跑工作区 + 组检查，并做指纹/代数比对。

---

## 7. Python 复刻要点（FastAPI）

### 7.1 工具定义：Pydantic + 函数装饰器

把 Go 的 `Tool` 接口（Name/Description/InputSchema/Execute）映射为：

```python
from pydantic import BaseModel, Field
from typing import Any

class ToolContext:
    """对应 llm.ToolContext，携带 user/conv/workspace/db/预算/on_artifact 回调等。"""
    ...

@dataclass
class ToolResult:
    text: str
    citations: list[Citation] = field(default_factory=list)

class ToolFn(Protocol):
    name: str
    description: str
    input_schema: dict          # JSON Schema（function calling schema）
    def execute(self, ctx: ToolContext, args: dict) -> ToolResult: ...
```

装饰器模式（推荐）：`@tool(name, description, **schema)` 自动把 Pydantic 模型转 JSON Schema 存到 `input_schema`，并把函数包成 `ToolFn`。模型调用时：

```python
class WebSearchInput(BaseModel):
    query: str = Field(description="One search query")
    queries: list[str] | None = Field(max_items=5, default=None)
    top_k: int = Field(10, ge=1, le=10)

@tool("aivory_web_search", "Search the public web ...", schema=WebSearchInput)
def web_search(ctx, args): ...
```

- `InputSchema()` 动态特性（web_search 随后端增删参数）复刻直接用 Pydantic 构造不同 schema 即可。
- 每个工具自己 `model_validate(args)`（对应 Go 的 `json.Unmarshal`），错误用 `ToolUserError(message)`（可安全回显给模型）。

### 7.2 调用循环

```python
async def run_tools(tools: ToolsRegistry, calls, ctx, emit):  # 并发，信号量
    for c in calls:
        emit(SseEvent(type="tool_start", name=c.name, id=c.id, input=c.input))
    sem = asyncio.Semaphore(MAX_CONCURRENT)
    results = await asyncio.gather(*[
        _run_one(sem, tools, c, ctx, emit) for c in calls
    ])
    return results
```

- 用 `asyncio.gather` + `Semaphore` 对应 Go 的信号量并发。
- 结果回传保持 `(text, citations, error)` 三元组；`done` 事件带 usage/credits。
- 多轮循环：把 `tool_start/tool_result/artifact` 块写回消息，送 provider 直到 `stop_reason`。

### 7.3 沙箱：保持同样 HTTP 协议（可复用现有 Python sandbox-service）

- **完全照搬 `sandbox.HTTPSandbox` 的 HTTP 契约**：`POST /sessions`、`POST /exec {session_id, code, timeout_ms}`、`POST /files {session_id,path,data_base64}`、`POST /files/reset-inputs`、`POST /files/get`、`POST /files/list`、`DELETE /sessions/{id}`、`DELETE /archives/{key}`、`POST /archives/files/list|get`、`POST /storage/gc`。
- 现在仓库已有 `沙箱服务-Python实现.md` 对应这个 sidecar，**FastAPI 后端作为 client 直接 HTTP 调用它即可**，无需重写沙箱。
- 关键行为复刻：
  - `sandbox_id` 存会话 `provider_state`；创建后持久化失败要释放会话。
  - `ResetInputs → PutFile(up中/uploads, /skills, manifest)` 的 staging 流程。
  - `ErrSessionGone`（404 + `session not found/running`）→ 重建会话 + 重新 staging + 重试一次。
  - 每次调用会话锁（Python 用 `asyncio.Lock` per conv 的 dict）。
  - 输出截断（32KB）、产物 `saveArtifact` + `artifact` 事件、已知错误脱敏。
- 启用条件：`sandbox_base_url` 管理设置非空 → `Enabled`；否则 `python_execute` safe-mode（只做 `print(expr)` 算术求值）。

### 7.4 MCP：用官方 `mcp` Python SDK

Aivory 是 **MCP client**（连外部 MCP server，把其工具伪装成本地 Function）。用官方 SDK：

```python
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async with streamablehttp_client(url, headers=headers, timeout=30) as (read, write, _):
    async with ClientSession(read, write) as sess:
        await sess.initialize()               # 对应 ensureDiscovered（含 server/discover / initialize 双协议）
        tools = await sess.list_tools()       # 显式发现，快照入库
        result = await sess.call_tool(name, arguments)  # 执行
```

- 发现是**显式/手动同步**的（管理员或用户触发），聊天轮次不做远端 schema 发现——把 `discovered_tools` 快照持久化，之后列表直接读库。
- 照搬安全层：合成 Function 名（`mcp_<clean>_<sha256前6字节>`，≤64、`[A-Za-z0-9_-]`）、RuntimeFingerprint、`InvalidateMCPServer` 代数、header 值 write-only + 全链路脱敏（`redactMCPText/JSON`）。
- 用户 MCP 租户隔离：按 `userID + workspace scope` 做缓存键与来源身份。
- 执行前多层鉴权（workspace policy、组工具策略、`disabled_tools`）照搬。

### 7.5 日志 / 诊断 / SSE / 缓存

- 工具日志：异步队列写 `tool_call_logs`；`log_request_bodies` / `log_tool_errors_only` 开关；HTTP 请求采样 + 脱敏（`tooldiagnostics` 逻辑可原样移植为正则 + JSON 树递归 scrub）。
- SSE：用 `sse-starlette`；事件名/字段保持 `SseEvent` 定义；`tool_start` 先串行宣布再并发执行。
- msgcache：Redis `INCR conv:ver:<conv>` + `SET conv:path:<conv>:<leaf>:<ver>`（TTL 45s / version 10min）——算法照搬。

### 7.6 复刻工作量排序建议

1. 先做 `ToolFn` + 装饰器 + `Registry.List/Run`（最小闭环，先内置 web_search/web_fetch）。
2. 接沙箱 HTTP client（复用现有 sidecar）。
3. 工具日志/诊断 + SSE 事件。
4. MCP client（官方 SDK）+ 合成 Function 命名与缓存。
5. 多层鉴权 + 配额/计费（image_generate 最后做）。

---

## 附：关键文件速查

| 文件 | 角色 |
|---|---|
| `server/internal/tools/registry.go` | Registry、MCP 折叠/缓存/指纹/脱敏 |
| `server/internal/tools/builtins.go` | 7 个内置工具实现 |
| `server/internal/tools/call_logging.go` | `Registry.Run` 日志包装、异步落库 |
| `server/internal/tools/sandbox_settings.go` | settingsSandbox（管理设置 → 沙箱配置） |
| `server/internal/tools/sandbox_folder_staging.go` 等 | 沙箱 staging 辅助 |
| `server/internal/tools/toolnames/names.go` | 工具名常量 |
| `server/internal/tools/tooldiagnostics/diagnostics.go` | 诊断采样/脱敏 |
| `server/internal/sandbox/sandbox.go` | HTTPSandbox、HTTP 协议、StorageConfig |
| `server/internal/sandbox/archive_browser.go` | 归档只读浏览、IsSessionGone |
| `server/internal/mcp/{client,transport,types}.go` | MCP Streamable HTTP client |
| `server/internal/sse/sse.go` | SSE Writer（并发安全） |
| `server/internal/msgcache/msgcache.go` | 消息版本化缓存 |
| `server/internal/api/{admin,user}_mcp_handlers.go` | MCP CRUD + 发现 + 脱敏 |
| `server/internal/api/tools_catalog_handlers.go` | 工具目录（可选工具列表） |
| `server/internal/llm/tool_exec.go` | 调用循环、ToolRunner、runToolsConcurrent、ToolUserError |
| `server/internal/llm/orchestrator.go` | ToolContext、OnArtifact、预算 |
| `server/internal/llm/types.go` | ToolDef/MCPToolDef/Citation/SseEvent/ArtifactRef |
