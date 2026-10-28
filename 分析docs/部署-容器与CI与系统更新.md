# Aivory 部署、容器与 CI 与系统更新调研

> 配套：《00-整体架构与技术栈.md》《99-复刻路线图-Python后端.md》。
> 本文档调研 **部署形态（Docker 镜像与 Compose）、CI 流水线、系统更新器（`updater/`，纯 Python 可复用）**，并给出新 Python 后端替换后的部署适配方案。

---

## 1. 镜像构建：单容器同源 SPA + API

`deploy/Dockerfile.app`（三段式、BuildKit）：

| Stage | 基础镜像 | 动作 |
|---|---|---|
| web | `node:22-bookworm-slim`（`--platform=$BUILDPLATFORM`） | `npm ci` + Vite `npm run build` → `dist`（前端在原生平台构建，避免 QEMU 模拟下 npm 崩溃） |
| build | `golang:1.26-bookworm`（`CGO_ENABLED=1`） | `go build ./cmd/api` → `aivory-api`（嵌入 SQLite 驱动需 CGO + C 工具链） |
| runtime | `debian:bookworm-slim` | 装 `ffmpeg chromium fonts-noto-cjk tzdata ca-certificates`；拷贝 API 二进制 + SPA `dist`；设 `STATIC_DIR=/app/web`、`UPLOAD_DIR`/`ARTIFACT_DIR`/`BACKUP_DIR`；监听 **8787**；healthcheck `GET /api/health`；`ENTRYPOINT aivory-api` |

**关键设计**：**Go 进程同时服务 SPA 与 `/api`（同源）**，`src/api/client.ts` 的 `API_BASE` 默认为同源 `/api`，无需构建期 URL。域名/HTTPS 反向代理部署需设 `ALLOWED_ORIGINS`。

> **Python 替换提示**：把 `build`/`runtime` 两阶段换成 Python 镜像即可：`python:3.12-slim` runtime + `uv`/`pip` 装 FastAPI；`STATIC_DIR` 由 FastAPI `StaticFiles` + 挂载提供（`/api` 路由 + `StaticFiles(directory=web)`）。CGO/SQLite 细节：Python 的 `pysqlite3` 无需 CGO，构建更轻。chromium 仍需要（PDF 导出）—— 换用 `playwright install chromium`。

---

## 2. 两种部署拓扑（docker-compose）

### 2.1 全功能生产（`docker-compose.prod.yml`）
服务与端口（`grep` 节点）：

| 服务 | 镜像/说明 |
|---|---|
| `postgres` | `postgres:16-alpine`（数据卷 pgdata） |
| `redis` | `redis:7-alpine`（redisdata） |
| `qdrant` | `qdrant/qdrant:v1.12.4`（qdrantdata） |
| `sandbox` | `aivory-sandbox-sidecar`：挂载宿主 `docker.sock`，为每个会话 `docker run` 锁定的 `aivory-sandbox` 运行时；env：`SANDBOX_MEMORY`(2g)、`SANDBOX_CPUS`、`SANDBOX_MAX_SESSIONS`(16)、`SANDBOX_EXEC_TIMEOUT_CAP_MS`(600000)、`SANDBOX_READ_ONLY_ROOTFS=1`、`SANDBOX_WORKSPACE_SIZE`(512m)、`SANDBOX_LOCAL_STORAGE_DIR`(workspace 归档卷 sandbox-archives) |
| `sandbox-image-keepalive` | 防止运行时镜像被 `docker image prune` 回收 |
| `updater` | `aivory-updater`（Python，见 §4） |
| `app` | `aivory-app`：端口 **`80:8787`**；env `AIVORY_LISTEN=:8787`、`SANDBOX_URL` 等 |

网络：内部 `internal` 网络隔离（仅 app 对外暴露 80）。

### 2.2 个人版（`docker-compose.personal.yml`）
仅 `app`、`sandbox`(+keepalive)、`updater`、归档卷。**单一 app 实例跑 SQLite**（`bun` 内嵌向量，无需 PG/Redis/Qdrant），零外部依赖。

### 2.3 外部依赖与 `ALLOWED_ORIGINS`
- 生产需 `POSTGRES_PASSWORD`、`REDIS_PASSWORD`、`JWT_SECRET` 强值；个人版仅 `JWT_SECRET`。
- 反向代理/HTTPS 部署必须设置 `ALLOWED_ORIGINS` 为精确浏览器 origin（CORS + Cookie SameSite 语义）。

---

## 3. CI 流水线（`.github/workflows/`）

| 工作流 | 职责 |
|---|---|
| `docker-images.yml` | 构建并推送 `aivory-app`（含前端+后端）镜像到 GHCR |
| `sandbox-image.yml` | 构建/推送 `aivory-sandbox`（运行时）与 `aivory-sandbox-sidecar` 镜像 |
| `desktop.yml` | 桌面客户端构建/测试/发布 |

镜像 tag 约定：`IMAGE_TAG`、`APP_IMAGE_TAG`、`SANDBOX_IMAGE_TAG` 支持分镜像指定版本；`registry=ghcr.io`、`owner=hjxwz123`（`updater.py` 中同名默认）。

---

## 4. 系统更新器（`updater/` —— 纯 Python，可复用）

`updater.py`（456 行，**纯 Python 标准库**，`http.server.ThreadingHTTPServer` + `urllib`，无第三方依赖）：

- **角色**：一个常驻小 HTTP 服务，从 **Admin → 更新控制** 接收指令，通过读写 `docker-compose.prod.yml` / `.env` 触发 `docker compose pull && up -d` 完成应用升级；也顺带更新「正在运行的捆绑 sandbox」源码。
- **关键机制**：
  - 监听 `AIVORY_UPDATER_LISTEN`（默认 `0.0.0.0:8790`）。
  - 鉴权：HS256 token，token 文件 `AIVORY_UPDATER_TOKEN_FILE`（`/shared/.aivory-update-token`），与应用容器共享密钥（Go 端 `system-update.ts` / `desktop-update.ts` 与更新器对接）。
  - 配置驱动：`AIVORY_UPDATER_COMPOSE_FILE`、`AIVORY_UPDATER_ENV_FILE`、`AIVORY_UPDATER_APP_SERVICE`（默认 `app`）。
  - 拉取指定 `IMAGE_TAG` 的镜像 → health 检查（`AIVORY_UPDATER_HEALTH_TIMEOUT_SECONDS` 默认 180s）→ 回滚保护。
  - `sandbox_sources.py`（48 行）：`source_fingerprint` 计算捆绑沙箱运行时源码指纹，用于决定是否需要重建/更新 sandbox 镜像。
- `test_updater.py`（278 行）：标准库 `unittest` 单测。

> **Python 替换提示**：`updater/` 与语言无关（只操作 compose/镜像），**可直接复用**驱动新的 Python 后端容器升级；无需重写。

---

## 5. 新的 Python 后端的部署适配（Compose 层）

把 `aivory-app`（Go）镜像替换为 `aivory-python-app`（FastAPI）后，`docker-compose` 的**外围服务不变**：

| 组件 | 是否需改 | 说明 |
|---|---|---|
| postgres/redis/qdrant | 否 | FastAPI 通过 `asyncpg`+`redis-py`+`qdrant-client` 连接 |
| sandbox (sidecar) | **否** | 已是 Python/FastAPI，HTTP 契约不变 |
| updater | 否 | 语言无关 |
| app 镜像 | **改** | 换成 Python 镜像；`STATIC_DIR` 改由 FastAPI `StaticFiles` 服务；env 命名尽量对齐 |
| 健康检查 | 改 | `/api/health` 由 FastAPI handler 提供（契约不变） |
| 端口 | 不变 | 容器内 8787（FastAPI/uvicorn `--port 8787`），对外 80 |

**关键纪律**：保持 `/api/health`、`ALLOWED_ORIGINS`、`JWT_SECRET`/`AIVORY_*` env 命名与契约一致，使 Compose 与更新器零改动。

---

## 6. 复刻部署顺序建议

1. **个人版先行**：单容器 FastAPI + SQLite + 内嵌向量（先不做 Qdrant/Redis），跑通 `/api/health` 与前端。
2. **加 PostgreSQL**：切换 `DATABASE_URL`，验证 SQLAlchemy 双方言。
3. **加 Redis + Qdrant**：启用缓存/pub-sub/异步任务/向量。
4. **接 sandbox compose 服务**：复用现有 sidecar。
5. **接 updater**：复用更新器驱动容器升级。
6. **CI**：复制三套 workflow，替换 build 命令为 Python（`uv build`/`docker build` FastAPI 镜像）。
