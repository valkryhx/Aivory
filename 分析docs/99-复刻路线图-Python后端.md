# Aivory 复刻路线图（后端换成 Python）

> 这份文档是整套调研的**行动指南**。它把前面各模块调研（00–06、沙箱、前端、桌面）的结论，组织成一份可执行的、把 Go 后端替换成 Python 的复刻计划。
>
> 配套文档见 `00-整体架构与技术栈.md` 的「文档索引」表。

---

## 0. 一句话结论

**复刻的最大红利是：前端（React）几乎可以完全复用。** 只要新的 Python 后端在 **HTTP 端点路径、请求/响应 JSON、JWT 鉴权、SSE 事件流** 这四层与原 Go 后端协议一致，`src/` 前端无需改动即可对接。因此复刻的主线不是「重写前端」，而是「用 Python 重写 server/ 的协议面与业务」，并且把 `sandbox-service` 这个已是 Python 的服务**原样保留复用**。

---

## 1. 技术选型对照表（Go → Python）

| 职责 | 原 Go 实现 | Python 复刻建议 | 理由 |
|---|---|---|---|
| HTTP 路由/中间件 | `net/http` + 自研 mux | **FastAPI**（依赖注入、Pydantic、SSE 支持） | 类型安全、异步、`Depends` 做鉴权 |
| 关系库访问 | `pgx/v5` + SQLite 驱动 + 手写 SQL | **SQLAlchemy 2.0 (async)** 或 **SQLModel** | 双后端兼容、类型映射 |
| 迁移 | 启动时幂等重放 schema.sql | **Alembic**（基线 + 增量） | 版本化，替代原 Migrate |
| 缓存/PubSub | Redis / 内存 Cache 接口 | `redis-py`（aioredis）或 `aiocache` | 保持 Redis 可选降级 |
| 异步任务 | asynq(Redis) / 进程内队列 | **arq** 或先做进程内 `asyncio.Queue` 任务 | 个人版可零依赖 |
| 向量库 | Qdrant / SQLite 内嵌 | `qdrant-client` 或 `sqlite-vec` / `chromadb` | 个人版可内嵌 |
| LLM Provider | 自研适配 | **自研 + Pydantic**（统一 OpenAI 兼容协议） | 结构化输出用 tool calling / JSON mode |
| 结构化输出 | TaskLLM + typesafe | Pydantic + provider JSON mode / tool use | 对齐原 TaskLLM |
| JWT | `golang-jwt/v5` | `pyjwt` 或 `python-jose` | — |
| Passkey(WebAuthn) | `go-webauthn` | `fido2` / `webauthn` (python) | — |
| OAuth | 自研 | `authlib` | — |
| SMTP | `mail` 包 | `aiosmtplib` / `smtplib` | — |
| Stripe | `stripe-go` | `stripe` (官方 Python SDK) | — |
| 对象存储 | S3/阿里OSS/本地 | `boto3` / `oss2` / 本地磁盘 | — |
| PDF 导出 | chromedp(Headless Chrome) | `playwright` 或 `weasyprint` | — |
| 代码沙箱 | 外部 HTTP 调用 | **复用现有 `sandbox-service`（Python/FastAPI）** | 已是 Python，直接保留 |
| 配置 | 环境变量 | `pydantic-settings` | — |
| 日志 | `log`/`slog` | `structlog` 或 `logging` | — |

---

## 2. 建议的新后端项目目录结构

```
aivory-python-backend/
├── pyproject.toml / requirements.txt
├── alembic/                      # 数据库迁移（替代 go server/migrations）
│   ├── versions/
│   └── env.py（双方言：postgresql / sqlite）
├── app/
│   ├── main.py                   # FastAPI 入口 + lifespan 装配（对应 go main.go）
│   ├── core/                     # config(pydantic-settings)、logging、envcfg
│   │   ├── config.py
│   │   └── security.py           # JWT、密码、2FA、captcha
│   ├── db/                       # SQLAlchemy engine/session、models
│   │   ├── base.py
│   │   ├── session.py
│   │   └── models/               # 每张核心表一个 model（继承原 schema）
│   ├── api/                      # 路由 + handler（对应 internal/api）
│   │   ├── deps.py               # 依赖注入：get_current_user 等
│   │   ├── middleware.py         # 限流、日志、CORS
│   │   └── routers/
│   │       ├── auth.py oauth.py  convo.py  messages.py
│   │       ├── workspaces.py projects.py files.py kbs.py
│   │       ├── admin/ payments.py share.py
│   │       └── ...
│   ├── services/                 # 业务层
│   │   ├── llm/                  # provider、orchestrator、task_llm、流式
│   │   ├── tools/                # 工具注册表 + 调用循环
│   │   ├── rag/                  # 摄取/检索/引用/异步
│   │   ├── vector/               # VectorStore 抽象(qdrant/sqlite)
│   │   ├── sandbox_client/       # 调 sandbox-service 的 httpx 客户端
│   │   ├── payment/  mail/  storage/  cache/  pdf/
│   │   └── ...
│   ├── schemas/                  # Pydantic 请求/响应模型（对齐原 JSON）
│   ├── crud/                     # SQLAlchemy 查询函数
│   └── sse.py                    # SSE 事件流封装 + 消息缓存/停止流协调
├── sandbox-service/              # 【保留】现有 Python 沙箱，原样复用
└── tests/
```

对应关系速查：

| Go | Python |
|---|---|
| `cmd/api/main.go` | `app/main.py`（lifespan 装配） |
| `internal/api/router.go` | `app/api/routers/*.py` + 主 router |
| `internal/api/middleware.go` | `app/api/middleware.py` + FastAPI `Depends` |
| `internal/llm` | `app/services/llm/` |
| `internal/tools` | `app/services/tools/` |
| `internal/rag` + `vector` | `app/services/rag/` + `app/services/vector/` |
| `internal/store` + `migrations` | `app/db/` + `alembic/` |
| `sandbox-service` | 原样保留 |
| `desktop/` | 先不做桌面，部署重点放 Web + Docker |

---

## 3. 分阶段实施路线（由易到难，逐阶段可验证）

### 阶段 0：环境与协议基线
- 建立 FastAPI 骨架 + `pydantic-settings` 配置 + SQLAlchemy + Alembic。
- **先跑通 `/api/health` 与一个不需要鉴权的端点**，并让原前端 `src/api/client.ts` 能连上（配 `VITE_` / baseURL）。
- 目标：前端白屏能加载，认证端点可返回正确结构的错误。

### 阶段 1：认证 + 用户 + 工作区（最小闭环）
- 迁移用户/会话/token/工作区/成员相关表。
- 实现：注册、登录、JWT 签发/刷新、登出、`/api/me`、工作区 CRUD。
- **用原前端跑通登录 → 进入主界面**。
- 「Python 复刻要点」细节见 `后端-01-API路由与认证授权.md`。

### 阶段 2：对话与 LLM 编排（产品核心）
- 会话/消息/分支表迁移。
- 实现 Provider 注册表 + Orchestrator + SSE 流式（delta/thinking/tool_logs/done）。
- 实现 TaskLLM（结构化输出）用于标题生成等。
- **关键验收**：原前端能发消息、看到流式增量输出、能停止生成、对话出现自动标题。事件协议必须与 `后端-02-LLM编排与流式.md` 中记录的一致。

**⚠️ 停止生成跨实例送达（Python 实现检查项，源自《停止生成跨实例精准送达机制分析.md》）**：
- Redis Stream 事件流 `gen:{msg_id}` + 吊销墓碑 `gen:{msg_id}:revoked`，原样移植三个 **Lua 原子脚本**（先 `EXISTS` 查墓碑 → `XADD`/`XRANGE (id +`/`DEL`），**不要**用「GET 再 XADD」两条命令。
- stop 端点（`POST /conversations/:id/stop`）三级选择器：`generation_id` / `message_id` / 无选择器；**任何 stop 前先 `get_conversation(id, user_id)` 校验所有权**。
- 发布侧 `set(scopedStopIntentKey, ttl) + publish(topic)`；订阅侧**先 `psubscribe` 再查该持久意图标记**（弥合「停止先到、生成未订阅」竞态）。
- `ctx` 取消后**只放行 `done(stopped)`**，丢弃 provider 缓冲的迟到 delta（FastAPI `CancelledError`/`asyncio`）。
- 跟随者三重兜底：吊销广播 + 墓碑回读 + DB 状态轮询（`asyncio` 定时 `status != streaming`）。
- 个人版可先只做**进程内**版本（互斥锁 + 内存流），语义与 Redis 版对等，生产再切 Redis。

### 阶段 3：工具 + 沙箱 + RAG（可选的增强闭环，供应链价值最高）
- 工具注册表 + 调用循环；`sandbox-client` 对接**现有沙箱服务**（复刻时可直接复用其 HTTP 协议与容器）。
- RAG：文档上传 → 解析（pypdf/python-docx/python-pptx）→ 分块 → 嵌入 → 向量检索 → 引用注入。异步用 arq（或阶段 2 内先做进程内队列）。
- 详情见 `后端-03-工具与沙箱与MCP.md`、`后端-04-RAG知识库与向量存储.md`、`沙箱服务-Python实现.md`。

### 阶段 4：文件、办公文档、管理后台、计费（完整度）
- 对象存储抽象 + 上传/下载/预览（`后端-06`）。
- 管理后台：Channels/Models 配置、成员、限额（对应 `/api/admin/*` 一大批端点）。
- 订阅/积分/支付：Stripe webhook 等（`后端-06`）。
- 办公文档：前端已能生成 PPTX/DOCX，后端主要是文件存取 + AI PPT 网关转发（`前端-03`）。

### 阶段 5：横切加固 & 部署
- 限流、验证码、SSRF 防护、上传安全（`后端-01`/`后端-06`）。
- 部署：Docker Compose（适配个人 SQLite + 生产 PG/Redis/Qdrant），复用现有 `deploy/` 的思路。
- 桌面客户端、自动更新可以**延后或不做**（先出 Web 版）。

---

## 4. 必须保持的协议兼容清单（前端零改动的前提）

> 前方若相应对应模块文档已产出，细节以该文档为准；此清单是总纲。

1. **HTTP 端点路径与方法**：`POST /api/auth/login`、`POST /api/auth/refresh`、`GET /api/me`、`POST /api/conversations/:id/messages`、`GET /api/events`（实时事件流）等 —— 见 `src/api/endpoints.ts` + `后端-01`。
2. **鉴权**：请求头 `Authorization: Bearer <access_token>`；刷新用 refresh token；过期/刷新语义与原一致。
3. **JSON 字段命名**：后端 JSON 字段用 camelCase，`src/api/types.ts` 定义的接口字段名必须原样匹配（Pydantic 可用 `alias` 或直接同名）。
4. **SSE 事件格式**：事件名（`delta`、`thinking`、`tool_logs`、`done`、`conversation.updated`…）与每个事件负载的 JSON schema —— 见 `后端-02` 与 `前端-02`。
5. **资源 ID 格式**：`前缀_12位hex`（`co_`、`msg_`、`kb_`…）—— 见 `后端-05`。
6. **错误响应结构**：错误 body 的字段形态（`{error: ...}` 等）。
7. **实时事件（events）**：跨设备实时推送（通知、会话更新）所用的订阅通道。

### 兼容策略建议
- **先写一份「协议契约」**（从 `src/api/types.ts`、`endpoints.ts`、`后端-01/02` 提取），作为 Pydantic schema 与测试的依据。
- **用原前端 + 新后端做集成测试**（可复用仓库 `tests/browser/*.mjs` 的 Playwright 用例），每阶段一个冒烟集。

---

## 5. 各模块复刻要点速查（交叉引用）

| 模块 | 核心要点 | 详细文档 |
|---|---|---|
| 整体架构 | 前端可复用；后端服务端驱动协议面 | `00-整体架构与技术栈.md` |
| API 路由与认证 | FastAPI Depends 鉴权；JWT/2FA/Passkey/OAuth 选型 | `后端-01` |
| LLM 编排 | Provider 抽象、Orchestrator 状态机、SSE、TaskLLM | `后端-02` |
| 工具与沙箱 | 工具注册表、sandbox-client 复用现有 Python 沙箱、MCP | `后端-03` |
| RAG 与向量 | 解析/分块/嵌入/检索、异步队列、Qdrant/SQLite | `后端-04` |
| 数据模型 | SQLAlchemy + Alembic 基线迁移、PG/SQLite、短 ID、settings KV | `后端-05` |
| 支付/邮件/横切 | Stripe、积分账本、SMTP、S3/OSS、PDF、缓存、安全 | `后端-06` |
| 沙箱服务 | **已是 Python/FastAPI，原样复用** | `沙箱服务-Python实现.md` |
| 前端架构 | 状态管理、API 客户端、端点契约 | `前端-01` |
| 前端聊天 | SSE 消费、分支、停止生成 | `前端-02` |
| 前端办公/KB | OOXML 生成在前端、后端存文件与服务 | `前端-03` |
| 桌面 Electron | 可延后；Web 优先 | `桌面客户端-Electron.md` |

---

## 6. 风险与难点

1. **LLM 编排的状态机**是最复杂、最易踩坑的部分（工具循环、上下文压缩、停止流协商、跨进程事件）。建议把它当作独立子项目，先用一个模型 + 一个工具跑通。
2. **PG/SQLite 双后端**：SQLAlchemy 会吸收大部分方言差异，但要注意 `JSONB` vs `JSON`、全文/`like` 检索性能、事务行为差异。
3. **异步任务**：原 Go 用 asynq（Redis）也可降级进程内。Python 若想零依赖个人版，先做进程内 `asyncio.Queue` worker；生产再切 arq/Celery（注意进程内队列不跨节点）。
4. **协议边界反推**：前端对不少行为的假设（如 `GET /api/events` 长连接、SSE 逗号分隔/格式）藏在源码里，**逐个验证**，别凭想象。
5. **计费/支付**：涉及真实钱款，先做「积分消耗记账」逻辑与隔离，Stripe 接入放最后且要测试 webhook 签名。
6. **安全**：限流、验证码、SSRF、上传校验、生产 JWT 守卫——原项目考虑非常细，复刻时不要砍掉。
7. **规模**：后端约 400+ 个 Go 文件、441 条路由。**不要试图一次性全部搬**，按第 3 节阶段推进，每阶段都能跑。

---

## 7. 里程碑验收标准

| 里程碑 | 可运行验证 |
|---|---|
| M1 协议基线 | 原前端连接新后端，登录页加载、`/api/health` ok |
| M2 认证闭环 | 前端能注册/登录/退出，`/api/me` 返回用户 |
| M3 对话闭环 | 前端能对话、流式渲染、停止生成、自动标题、对话分支渲染 |
| M4 工具+沙箱 | 前端发起需要工具的请求，沙箱出结果回填，tool_logs 展示 |
| M5 RAG | 前端上传文档入库，检索到引用并展示来源 |
| M6 完整度 | 管理后台可配 Channel/Model；文件上传/预览；计费记账 |
| M7 部署 | Docker Compose 双配置（个人 SQLite / 生产 PG+Redis+Qdrant）跑通 |

---

## 8. 结语

Aivory 是一个「前端（React）+ 后端（Go）+ 沙箱（Python/FastAPI）+ 桌面（Electron）」的四层自托管 AI 工作台。复刻时 **前端与沙箱是资产**（沙箱本身已是 Python），真正的工作量集中在 **用 Python 重写 Go 后端并保持协议兼容**。按「认证 → 对话 → 工具/沙箱/RAG → 文件/办公/管理/计费 → 加固/部署」的顺序逐步落地，每一阶段都能用原前端做集成验收，风险可控。
