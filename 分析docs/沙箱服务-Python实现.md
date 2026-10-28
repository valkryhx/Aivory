# Aivory 沙箱服务(sandbox-service)——Python 代码执行沙箱深度调研

> 本文档深入分析 `sandbox-service/` 这个 **Python 实现的服务**,它本身就是一个独立运行的 Python 服务。
> 这个事实对「把 Go 后端复刻成 Python」极为关键:**它就是复刻后端的"天然一部分"**,读者应优先考虑**原样复用**。
> 所有源码引用格式为 `app.py:行号`(指 `sandbox-service/app.py`)或 `Dockerfile.*:行号` / `docker-compose.yml:行号`。
> 版本基准:`v2.5.2-beta.12`。

---

## 目录

1. [服务定位与架构](#1-服务定位与架构)
2. [HTTP API 契约](#2-http-api-契约)
3. [代码执行引擎深度分析](#3-代码执行引擎深度分析)
4. [文件侧车与 /workspace 持久化](#4-文件侧车与-workspace-持久化)
5. [安全隔离设计](#5-安全隔离设计)
6. [与 Go 后端及前端的协作](#6-与-go-后端及前端的协作)
7. [Python 复刻要点](#7-python-复刻要点)

---

## 1. 服务定位与架构

### 1.1 一句话定位

`sandbox-service` 是一个**独立的、基于 FastAPI 的 Python sidecar 服务**:它不直接执行用户代码,而是**接管宿主机的 Docker daemon(挂载 `/var/run/docker.sock`),为每个会话(session)拉起一个锁定的 `aivory-sandbox` 容器,并用 `docker exec` 把用户代码送进容器执行**。

文档头部注释(`app.py:1-30`)把它描述为 `design.md §4.5` 的设计落地,并明确给出它使用的 HTTP 协议:

```
POST /sessions  -> {"session_id": "..."}
POST /exec      {session_id, code, timeout_ms}
                -> {"stdout", "stderr", "exit_code", "files":[{name,mime_type,data_base64}]}
POST /files     {session_id, path, data_base64}  -> {"ok": true}
POST /files/reset-inputs {session_id}
                -> {"ok": true, "session_gone": false}
                   or {"ok": false, "session_gone": true}
```

**它说的是 Go 后端(宿主的 HTTP 调用方)的语言**——Go 后端只需要把 `SANDBOX_BASE_URL` 指向这个服务,不改一行 Go 代码即可接入(`app.py:1-4`)。注释明确写了:`server/internal/sandbox/sandbox.go` 期望的正是这套协议。

### 1.2 运行时拓扑:runner 与 sidecar

三个容器镜像/角色,分工清晰:

| 角色 | 镜像 | 构建文件 | 作用 |
|------|------|----------|------|
| **runtime runner**(实时沙箱运行时) | `aivory-sandbox` | `Dockerfile.runner` | 每个 session 容器运行的**纯 Python 运行时镜像**,携带数据科学+文档生成+中文字体。容器本身只 `sleep infinity`,由 sidecar 用 `docker exec` 注入代码 |
| **control sidecar**(控制服务) | `aivory-sandbox-sidecar` | `Dockerfile.sidecar` | 即 `app.py` 本体。Go 后端调用它;它驱动 Docker daemon 拉起 runner 容器 |
| **image keepalive**(守护) | `aivory-sandbox` | (同一 runner 镜像) | 一个无标签的空闲容器,纯为占用镜像引用,防止 `docker image prune -a` 清掉运行时镜像导致下次会话冷拉 600MB 超时(`docker-compose.yml:125-131`) |

三者关系(`docker-compose.yml:56-135`):

```
┌──────────────┐  POST /sessions /exec /files  ┌───────────────────┐  docker exec   ┌──────────────────┐
│  Go 后端      │ ────────────────────────────► │  sidecar (app.py) │ ─────────────► │ session 容器      │
│ (server)      │   SANDBOX_BASE_URL           │  FastAPI :8000    │                │ aivory-sandbox   │
└──────────────┘                               └───────────────────┘                │  --network none  │
      ▲                                          │                                 └──────────────────┘
      │            /var/run/docker.sock (挂载)   ▼
      │                                    ┌──────────────────┐
      └─────── SANDBOX_BASE_URL 指向同一      │ 宿主 Docker daemon │
                                               └──────────────────┘
```

### 1.3 Web 框架:FastAPI(不是 Flask、不是自研)

`app.py:55-57`:
```python
from fastapi import Body, FastAPI, HTTPException
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field
```

- **Web 框架是 FastAPI**,配合 Pydantic(请求/响应模型校验)与 Uvicorn(ASGI 服务器)。
- 应用入口 `app = FastAPI(title="Aivory Sandbox Sidecar", version="1.0")`(`app.py:302`,`Dockerfile.sidecar:48` 用 `uvicorn app:app --host 0.0.0.0 --port 8000` 启动)。
- 这是**同步 def 处理器**(非 async def),因为核心逻辑是阻塞式 `subprocess` 调 `docker`;FastAPI 会在线程池里跑同步 handler,天然支持并发。

### 1.4 依赖清单

**sidecar 依赖** `requirements.txt`(仅 4 个,轻量):
```
fastapi==0.115.6
uvicorn[standard]==0.34.0
boto3==1.35.99      # S3 / S3-compatible (MinIO) 归档
oss2==2.19.1        # Aliyun OSS 归档
```
> boto3/oss2 是**懒加载**的(`app.py:1662,1701`)——没有配置任何存储后端时,sidecar 能不装也能启动。

**runner 运行时依赖** `runner-requirements.txt`(数据科学+文档生成,全部 `==` 精确锁定,可复现+供应链安全):
- 数据科学:numpy 1.26.4、pandas 2.2.2、scipy 1.13.1、scikit-learn 1.5.1、statsmodels 0.14.2
- 绘图:matplotlib 3.9.2、seaborn 0.13.2、plotly 5.24.1、kaleido 0.2.1、pillow、sympy、networkx
- 表格/PDF 读取:openpyxl、xlsxwriter、xlrd、pypdf
- 文档生成:python-pptx、python-docx、reportlab、weasyprint(HTML→PDF)
- 文本:tabulate、requests、lxml、beautifulsoup4、markdown、jinja2、pyyaml

> 头部注释(`requirements.txt:1-2`)明确区分:**sidecar 无需 runner 的那些包**,它们只在 `Dockerfile.runner` 里装。

### 1.5 配置项(全部 env 可覆盖,`app.py:59-200`)

几乎每个行为都是 env 可调的,这是该服务可复刻性的关键。核心配置:

| 环境变量 | 默认 | 说明 |
|---------|------|------|
| `SANDBOX_IMAGE` | `aivory-sandbox:latest` | runner 镜像 |
| `SANDBOX_MEMORY` / `SANDBOX_CPUS` / `SANDBOX_PIDS_LIMIT` | `2g` / `1` / `256` | 会话容器资源限制 |
| `SANDBOX_API_KEY` | 空 | **Bearer 鉴权密钥,缺失则拒绝启动(fail-closed)** |
| `SANDBOX_ALLOW_NO_AUTH` | 空 | 显式 `1/true/yes/on` 才关闭鉴权 |
| `SANDBOX_EXEC_TIMEOUT_CAP_MS` | `600000`(10min) | 单次 exec 硬上限 |
| `SANDBOX_DEFAULT_EXEC_TIMEOUT_MS` | `120000`(2min) | 调用方缺省时默认超时 |
| `SANDBOX_IDLE_TTL_SECONDS` | `1800`(30min) | 空闲回收 TTL |
| `SANDBOX_MAX_SESSIONS` / `MAX_CONCURRENT_EXECS` / `MAX_CONCURRENT_CREATES` | `16` / `4` / `2` | 并发闸门 |
| `SANDBOX_READ_ONLY_ROOTFS` | `1` | 根文件系统只读(防磁盘填满) |
| `SANDBOX_WORKSPACE_SIZE` / `SANDBOX_TMPFS_SIZE` | `512m` / `256m` | /workspace 与 /tmp tmpfs 上限 |
| `SANDBOX_MAX_OUTPUT_BYTES` | `32KB` | stdout/stderr 截断上限 |
| `SANDBOX_MAX_ARTIFACT_BYTES` | `20MB` | 单产文件上限 |
| `SANDBOX_MAX_FILES_PER_EXEC` / `MAX_TOTAL_ARTIFACT_BYTES` | `20` / `50MB` | 单次 exec 文件数与总量 |
| `SANDBOX_MAX_BODY_BYTES` / `MAX_CODE_BYTES` | `56MiB` / `1MiB` | 请求体/代码体积护栏 |
| `SANDBOX_SECCOMP_PROFILE` | 空(可选) | 会话容器 seccomp 配置文件 |
| `SANDBOX_DISK_SIZE` | 空(可选) | 可写层 quota(需 overlay2+prjquota) |
| `SANDBOX_LOCAL_STORAGE_DIR` | 空(可选) | 本地归档目录(默认存储后端) |
| `SANDBOX_MAX_ARCHIVE_BYTES` | `200MB` | 工作区归档上限 |
| `SANDBOX_STORAGE_DEFAULT_TTL` / `MAX_TTL` | `3600` / `86400` | presigned URL TTL |

---

## 2. HTTP API 契约

### 2.1 端点一览表

所有请求/响应均为 JSON,除特殊说明外 **POST body 携带 session_id**。除 `/healthz` 外全部要求 `Authorization: Bearer <API_KEY>`。

| 方法 | 路径 | 请求体 | 响应 | 用途 |
|------|------|--------|------|------|
| POST | `/sessions` | `{storage? , idle_ttl_sec? , archive_key?}` | `{"session_id": "..."}` | 创建会话(拉起一个锁定容器) |
| POST | `/exec` | `{session_id, code, timeout_ms?}` | `{"stdout","stderr","exit_code","files":[{name,mime_type,data_base64}],...}` | 执行一段 Python 代码 |
| POST | `/files` | `{session_id, path, data_base64}` | `{"ok": true}` | 上传文件进 /workspace |
| POST | `/files/get` | `{session_id, path}` | `{"data_base64": "..."}` | 从 /workspace 取回文件 |
| POST | `/files/list` | `{session_id}` | `{"files":[{"path","size"}],...}` | 列出 /workspace 下所有文件(管理用) |
| POST | `/files/reset-inputs` | `{session_id}` | `{"ok":bool,"session_gone":bool}` | 清空 uploads/skills,保留 outputs |
| POST | `/archives/files/list` | `{archive_key, session_id?, storage?}` | `{"archive_status","files"[...]}` | 浏览已归档工作区文件列表 |
| POST | `/archives/files/get` | `{archive_key, session_id?, path, storage?}` | `{"archive_status","data_base64"?}` | 取回归档工作区中的某个文件 |
| DELETE | `/sessions/{session_id}` | `{storage?, discard?=false, archive_key?}` | `{"ok": true}` | 销毁会话(默认先归档工作区) |
| DELETE | `/archives/{archive_key}` | `{storage?}` | `{"ok": true}` | 只删归档、不操作容器 |
| POST | `/storage/put` | `{key, data_base64, content_type?, expires_in?, storage?}` | `{"provider","key","url","expires_in"}` | 直接写对象存储(返回 presigned URL) |
| POST | `/storage/delete` | `{key, storage?}` | `{"ok":true,"key"}` | 从对象存储删除对象 |
| POST | `/storage/gc` | `{storage?, max_age_seconds}` | `{"deleted","scanned","freed_bytes"}` | 清理过期归档对象 |
| GET | `/healthz` | 无 | `{"ok","docker","image","image_ready"}` | 健康检查(无需鉴权) |

> 注意:`/sessions` 实际是 POST(创建),不再是文档注释里 Go 端历史版本的 GET 语义不一致问题——Go 端 `sandbox.go:304` 用 `POST /sessions` 创建。

### 2.2 请求模型(Pydantic,`app.py:722-793`)

```python
class ExecBody(BaseModel):
    session_id: str
    code: str = Field(max_length=MAX_CODE_BYTES)     # 1 MiB 字符上限
    timeout_ms: Optional[int] = None

class FilesBody(BaseModel):
    session_id: str
    path: str
    data_base64: str

class CreateBody(BaseModel):
    storage: Optional[dict[str, Any]] = None        # 自由格式存储块,原样转发
    idle_ttl_sec: Optional[int] = None              # 管理员可调的回收 TTL
    archive_key: Optional[str] = None               # 稳定归档键(会话 id)

class DeleteBody(BaseModel):
    storage: Optional[dict[str, Any]] = None
    discard: Optional[bool] = False                 # true=不归档且删旧归档
    archive_key: Optional[str] = None

class StoragePutBody(BaseModel):
    key: str
    data_base64: str
    content_type: Optional[str] = None
    expires_in: Optional[int] = None
    storage: Optional[dict[str, Any]] = None
```
要点:`storage` 块是 Go 端原样转发的自由字典(provider、prefix、s3_*、oss_*),sidecar 保持对具体字段无感知(`app.py:753-755`)。

### 2.3 鉴权模型

- **fail-closed 启动保护**(`app.py:310-326`):若 `SANDBOX_API_KEY` 为空且未显式设置 `SANDBOX_ALLOW_NO_AUTH=1`,直接 `sys.exit(1)` 拒绝启动。
- **常量时间比较中间件**(`app.py:2135-2140`):除 `/healthz` 外,所有请求用 `secrets.compare_digest` 比对 `Authorization: Bearer <key>`,防时序侧信道。
- **请求体大小中间件**(`app.py:2146-2162`):按 `Content-Length` 在读取前拒绝超大请求;`/storage/put` 用更宽松的 300MiB 上限,其余用 56MiB。注册在最后所以最外层先执行。

---

## 3. 代码执行引擎深度分析

### 3.1 执行流程(端到端)

`POST /exec`(`app.py:914-983`)的完整生命周期:

1. **校验**:`_valid_session` 校验 session_id 必须是 32 位十六进制(uuid4 hex,防注入容器名,`app.py:334-337`);按 UTF-8 字节数拒绝超大代码(`app.py:923-925`);超时被 clamp 到 `[1s, EXEC_TIMEOUT_CAP_MS]`(`app.py:927-929`)。
2. **快路径拒绝**:若 `<sid>` 正在拆毁(reaper/DELETE 持锁归档),直接 404 让 Go 端快速新建会话(`app.py:934-935`,L6)。
3. **上锁+占执行槽**:`_session_lock(sid)`(限单会话并发)+ `_exec_slots` 信号量(全局最多 4 个并发 exec,`app.py:251`)。占槽超时抛 429(`app.py:405-415`)。
4. **写代码进容器**:每个 cell 生成一个 `AIVORY_CELL_TOKEN`(uuid4 hex),把代码写到容器内 `/tmp/aivory-cell-<token>.py`(`app.py:940-945`)。**用 stdin 写**而非 argv,**避免参数长度限制和 shell 引号风险**。
5. **执行前快照 outputs**:`_snapshot_outputs`(`app.py:1445-1457`)记录 `/workspace/outputs` 下每个文件 `mtime|size`,作为"哪些是新产出"的判据。
6. **真正执行**:
   ```python
   exit_code, stdout, stderr = _run_exec_bounded(
       name,
       ["timeout", "--kill-after=2s", _timeout_arg(timeout_s), "python", cell_path],
       timeout=timeout_s + 15,
       env={"AIVORY_CELL_TOKEN": cell_token},
   )
   ```
   (`app.py:956-962`)
   - 容器内用 **coreutils `timeout`** 包裹 `python`,`--kill-after=2s` 保证超时后强杀;
   - 宿主侧 `_run_exec_bounded` 再加一层墙钟超时(`timeout_s + 15s`)并用 selector 非阻塞读 stdout/stderr。
7. **清理后台遗留进程**:`_kill_marked_processes`(`app.py:619-644`)扫描容器内 `/proc/[0-9]*` 的 `environ`,凡是继承了 `AIVORY_CELL_TOKEN=<token>` 的进程(包括 subprocess / double-fork / setsid 脱离进程组的)一律 `kill -9`。这是**关键深度**:只有 `timeout` 杀其直接子进程 + 宿主 `proc.kill()` 杀 `docker exec` 客户端,无法覆盖"自行脱离进程组的后台进程",它们会 reparent 到 PID 1 的空转并持续烧掉 cpu/mem/pids 预算直到 reaper。env 标记被 fork/exec/setsid/double-fork 继承,故能收割整棵进程树,而 PID 1(`sleep infinity` 无标记)永不被误杀。
8. **删除 cell 文件**,`exit_code==124` 时附加超时提示(`app.py:970-972`)。
9. **收集新产出文件** `_collect_new_files`(见 3.3),返回 `{"stdout","stderr","exit_code","files"}`。

**执行命令细节**:`timeout --kill-after=2s 120.000s python /tmp/aivory-cell-xxx.py`,工作目录是 `/workspace`(`-w` 参数,`app.py:825`),所以用户代码里 `open("x.csv","w")` 直接落在 /workspace。

### 3.2 超时 / 资源限制全栈三层

| 层 | 机制 | 位置 |
|----|------|------|
| 容器层 | `--memory`、`--memory-swap`、`--cpus`、`--pids-limit`、`--ulimit nofile` | `app.py:816-820` |
| 进程层 | 容器内 `timeout --kill-after=2s` | `app.py:958-959` |
| 宿主层 | `_run_exec_bounded` 墙钟 + selector 非阻塞读 | `app.py:564-616` |

**stdout/stderr 截断**:`_BoundedOutput`(`app.py:383-402`)把 stdout/stderr 分别限制在 `MAX_OUTPUT_BYTES`(默认 32KB),超了加 `... [truncated,...]` 尾注——**保证 sidecar 不会被一个 `print` 死循环 OOM**。宿主 `docker exec` 的 stdout/stderr 用 selectors 读,每块最大 8KB。

**为何要 selector 而非 read()**:`docker exec` 管道不关闭就不能阻塞式读完;selector 非阻塞轮询(0.2s 间隔,`app.py:129`)让宿主能**在中途检测到超时并 `proc.kill()`**,再等 grace 2s 合并退出码。

### 3.3 文件输出收集

`_collect_new_files`(`app.py:1460-1501`):
- 对比执行前后 `outputs` 快照,只取**发生变化(新增或 mtime/size 变化)**的文件;
- 逐项 base64:`docker exec <name> base64 -w0 <path>`(在容器内编码,避免原始字节经过 argv);
- MIME 类型用 `mimetypes.guess_type` 推断;
- **硬性护栏**:单文件 ≤ 20MB、每 exec ≤ 20 个文件、合计 ≤ 50MB、收集预算墙钟 ≤ 60s(`MAX_COLLECT_SECONDS`),超出的计为 skipped 并在 stderr 附加说明。这保证"一次输出海量大文件"不会让响应超出 Go 端 HTTP 客户端预算。

### 3.4 允许的依赖 / 包安装

- **预装固定镜像依赖**:见 1.4(runner-requirements.txt),数据科学 + 文档生成全套。
- **`/workspace` 会话内持久**:`/workspace` 是 tmpfs 且跨 exec 保留(`app.py:15-18`,桌面注释"pip-installed packages ... survive")。因此用户在代码里 `pip install xx`(若镜像有 pip 并可写 tmpfs)理论上可在会话内装包,但**容器无网络(`--network none`)**,所以装包只能用镜像已有/内置 wheel,无法访问公网 PyPI——**网络下载被隔离,公共镜像下载由 Go 后端代理**。
- **可选重型扩展**(`Dockerfile.runner:70-80` 注释):Playwright/Chromium(HTML 截图,~400MB)、LibreOffice(格式转换,~500MB),取消注释即可。

### 3.5 开机预热(matplotlib 中文字体)

`_warm_matplotlib` + `_MPL_WARM_PY`(`app.py:652-718`)在**每个新会话容器创建后**、用户代码运行前,以 `python -` 方式注入一段预热脚本:重写 CJK 版 `matplotlibrc` 到 `$HOME`(因为 `/home/sandbox` 是空 tmpfs,遮蔽了镜像 bake 的 rc),并强制构建 matplotlib 的 FontManager 缓存 + 一次真实渲染。这解决"第一张图中文是方框、第二张就好了"的经典 bug。**预热失败被吞掉、绝不放慢或阻塞会话创建**。

> 这是该服务"产品级打磨"的典型:它不只是"能跑代码",还包括中文字体、CJK 渲染、PPT/PDF 生成这些 Aivory 的具体业务需求。

---

## 4. 文件侧车与 /workspace 持久化

### 4.1 目录布局(容器内)

`app.py:197-200`:
```python
WORKSPACE  = "/workspace"
OUTPUTS_DIR = f"{WORKSPACE}/outputs"   # Python 产出物
UPLOADS_DIR = f"{WORKSPACE}/uploads"   # 用户/后端上传的输入
SKILLS_DIR  = f"{WORKSPACE}/skills"    # 技能(技能包)
```
权限:硬化的只读根文件系统下,/workspace 是可写 tmpfs 且 `mode=1777`,让非 root 的 sandbox 用户(uid 1000)能建文件(`app.py:837-841`)。

### 4.2 文件交换协议(/files 与 /files/get)

- **上传 `/files`**(`app.py:986-1033`):base64 解码 → 规范化路径必须落在 `/workspace` 下 → 双层防越狱:
  1. 字符串层 `_safe_under_workspace`(`normpath` 不能 `..` 逃逸,`app.py:1517-1520`);
  2. **现实路径校验 F9**(`app.py:1017-1024`):先 `mkdir -p` 父目录,再用容器内 `realpath -m` 解析真实路径(跟随符号链接),若真实位置脱离 /workspace 则拒绝。防止"`cat >` 跟随符号链接父目录(如 `/workspace/x -> /etc`)越权覆盖容器内宿主可见文件"。
  - 写文件走 `docker exec -i sh -c 'cat > <真实路径>'`,stdin 传字节。
  - 文件体积上限 `MAX_UPLOAD_BYTES`(默认 40MB)。
- **下载 `/files/get`**(`app.py:1124-1155`):同样的路径规范化 + realpath 校验;容器内 `base64 -w0 <path>` 取回。拒绝目录/缺失文件(404)。
- **列出 `/files/list`**(`app.py:1158-1194`):容器内 `find /workspace -maxdepth 6` 输出 NUL 分隔的 `size\tpath`,避免换行文件名伪造行;上限 500 条。

### 4.3 输入管理 `/files/reset-inputs`

`app.py:1036-1063`:清空 `uploads` 与 `skills`,**保留 `outputs` 与后端拉取的图片**。这是一条**固定路径的、非调用方控制的一般删除接口**(刻意如此,避免做成任意删除)。

### 4.4 workspace 归档机制(G2 稳定键)

这是**跨会话持久化**的核心设计,也是「复刻要点」中最值得注意的业务逻辑之一:

- **生命周期**:`/sessions` 创建时若带 `archive_key`(Go 端转发的是 **conversation id**),记录之(`app.py:891-892`);销毁或空闲回收时 `_archive_workspace` 把 /workspace tar 成 gzip,上传到对象存储键 `<prefix>/workspaces/<archive_key>.tgz`(`app.py:1913-1981`);下次同会话再跑时 `_restore_workspace` 下载并 untar 回 /workspace(`app.py:2001-2039`)。
- **稳定键的意义**(`app.py:217-227` 注释,G2 修复):每次 create 都会 `uuid4().hex` 生成**全新 session id**,但归档用 **conversation id** 作键——所以会话被回收再重建,`/workspace` 依然能按同一个 conversation id 恢复,**工作区跨会话 recycle 存活**。
- **归档边界**:`--exclude=./uploads --exclude=./skills`(`app.py:1924`),因为这两者是数据库权威、每次 Python 调用都会重新 stage;下载限 `MAX_ARCHIVE_BYTES`(200MB),解压用 `--no-same-owner --no-overwrite-dir --strip-components=1`(防所有权注入 / 覆盖既有 tmpfs 根)。
- **读取侧的安全**(`app.py:1209-1271`):`/archives/files/*` 用带字节数/成员数/墙钟三重边界的流式 tar 扫描器 `_ArchiveScanReader` + 覆写 `_proc_pax`/`_proc_gnulong` 的 `_ArchiveTarInfo`,**防 tar 炸弹与超长元数据注入**。

### 4.5 存储后端:local / S3 / Aliyun OSS

存档与 `/storage/*` 走统一抽象(`app.py:1544-2040`),boto3/oss2 懒加载、客户端按凭据元组缓存:

| provider | 判定有效 | 上传/下载/删除 | presigned URL |
|----------|----------|----------------|---------------|
| `local` | `SANDBOX_LOCAL_STORAGE_DIR` 是存在的目录 | 直接读写文件系统(`_local_path` 强约束于目录内) | **不支持**(抛出 400) |
| `s3` | `s3_bucket` | boto3(`endpoint_url` 支持 MinIO 等兼容存储,自动 path-style + SigV4) | `generate_presigned_url` |
| `aliyun_oss` | bucket+endpoint+两把 key | oss2 | `sign_url(... slash_safe=True)` |

- **Key 安全**:`_validate_object_key`(`app.py:1586-1591`)拒绝 `..`、绝对路径、NUL;`_local_path`(`app.py:1600-1610`)用 realpath 保引用防符号链接逃逸。
- **GC**:`/storage/gc`(`app.py:1359-1418`)只清扫 `<prefix>/workspaces/*.tgz` 与 `<prefix>/mineru/*` 两个已知前缀下**时间戳确定且早于 cutoff**的对象,时间戳读不到一律保留(fail-safe)。

---

## 5. 安全隔离设计

### 5.1 隔离核心:每个会话一个 locked-down Docker 容器

`/sessions` 创建容器的完整参数(`app.py:805-849`):

```python
args = [
    "run", "-d", "--name", name,
    "--label", LABEL, "--label", f"aivory.session_id={session_id}",
    # 强制无网络
    "--network", "none",
    "--memory", MEMORY, "--memory-swap", MEMORY,
    "--cpus", CPUS, "--pids-limit", PIDS_LIMIT,
    "--ulimit", f"nofile={NOFILE_ULIMIT}",
    "--init",                       # tini 作 PID1,收僵尸
    "--user", "1000:1000",          # 非 root,映射到镜像里的 sandbox 用户
    "--security-opt", "no-new-privileges",
    "--cap-drop", "ALL",            # 丢弃全部 capabilities
    "-w", WORKSPACE,
]
```
- **无网络**:`--network none` 是"不变量而非可选项"(`app.py:811-815 注释`)。用户 Python 完全没有网络命名空间路由;公共镜像下载由 Go 后端代理。这是**外联攻击面被清零**的关键。
- **非 root + 丢 cap**:`--user 1000:1000` + `--cap-drop ALL` + `no-new-privileges`。
- **只读根文件系统 + 有界 tmpfs**(F6,`app.py:832-842`):`--read-only`,只有 `/tmp`、`/home/sandbox`、`/workspace` 是可写的、size 有界的 tmpfs(`--tmpfs ... size=...`),**阻止磁盘填满 DoS**(disk-fill)。rootfs 只读也意味着用户无法动系统二进制。
- **可选 seccomp**(F1,`app.py:830-831`):设 `SANDBOX_SECCOMP_PROFILE` 时为会话容器 pin 一个自定义 seccomp 配置,进一步收窄内核攻击面(配合 docker 默认 profile)。
- **可选磁盘 quota**(`app.py:848-868`):`--storage-opt size=`(overlay2+prjquota),不支持时自动 retry 一次去掉该参数。

### 5.2 代码/文件层防御(纵深)

- **session id 严格校验**:只能 `[0-9a-f]{32}`(`app.py:334-337`),永不被内插进 shell/容器名。
- **代码体积上限**:1MiB(`app.py:924-925`)。
- **请求体上限**:Content-Length 前置拒绝(`app.py:2146-2162`)。
- **路径双层防逃逸**:字符串 normpath + 容器内 realpath 跟随符号链接(`app.py:1017-1024, 1140-1144`)。
- **归档读取防 tar 炸弹**(`app.py:1209-1271`)。
- **临时文件原子写**:本地存储写 `.tmp-<pid>-<tid>` 再 `os.replace`(`app.py:1742-1745`)。
- **归档键净化**:`_sanitize_archive_key` 限制 `[A-Za-z0-9_-]{1,128}`,防前缀/路径遍历注入(`app.py:493-500`)。
- **后台进程收割**:`AIVORY_CELL_TOKEN` env 标记收割脱离进程组的进程(`app.py:619-644`)。

### 5.3 控制面自身的风险与缓解(docker.sock = root)

`docker-compose.yml:16-54` 的 SECURITY NOTE 明确指出:**sidecar 挂载 `/var/run/docker.sock`,而该 socket 等同宿主机 root——任何能驱动它的人都可接管宿主(挂载宿主 FS、跑 privileged)**。这虽无法消除(架构需要),但缓解:

1. **`SANDBOX_API_KEY` 强制**:不设 key 拒绝启动(fail-closed);
2. **不要暴露到公网**:仅对内网/VPN 可达;
3. **强烈建议套 docker-socket-proxy**(compose 注释给出完整实例),把 socket 权限限制在 `CONTAINERS/POST/EXEC/IMAGES/VERSION` 几个方法;
4. **给会话容器 pin seccomp**。

> 复刻时若从"developer/single-host"升级到生产,这个模型本身要换(见第 7 章)。

### 5.4 运行约束小结

sidecar **以 root 运行**、直驱宿主 Docker daemon,因此 sidecar 容器本身(非会话容器)是宿主机信任边界;docker-compose 给 sidecar 自身也加了 `mem_limit: 1g` + `pids_limit: 512`(`docker-compose.yml:108-109`),防止内存归档缓冲撑爆。

---

## 6. 与 Go 后端及前端的协作

### 6.1 Go 端消费方式(简单提)

Go 端 `server/internal/sandbox/sandbox.go`(`${BaseURL}` 指向 sidecar):
- `POST {BaseURL}/sessions` → 创建(拿 `session_id`);
- `POST {BaseURL}/exec {session_id, code, timeout_ms}` → 执行拿 `stdout/stderr/exit_code/files`;
- `POST {BaseURL}/files`、`POST {BaseURL}/files/reset-inputs` → 写文件/重置输入;
- `DELETE {BaseURL}/sessions/{id}`(可带 `discard`)、`DELETE {BaseURL}/archives/{key}` → 销毁/清归档;
- `POST {BaseURL}/files/get`、`POST {BaseURL}/files/list`、`POST {BaseURL}/archives/files/{list,get}` → 管理/审查工具;
- `POST {BaseURL}/storage/gc` → 定时 GC(管理员的 `storage` 块原样下传)。
- 鉴权:`Authorization: Bearer <apiKey>`。会话失效(404 / `session_gone:true`)时 Go 端自动新建会话(`app.py:105 permises-1052` 描述的 ErrSessionGone sentinel)。

### 6.2 前端如何展示代码执行结果

后端把 `/exec` 返回的 `files`(含 `mime_type` 与 `data_base64`)与 stdout/stderr 一并回给前端渲染:
- **标准输出/错误** → 对话流里的代码块结果面板(纯文本/终端风格)。
- **`mime_type` 引发的产物** → 交给前端按类型分派渲染:图片(`image/png` 等)内联预览、表格/CSV、HTML(`text/html` 走 HTML 预览 iframe/紧凑渲染)、PDF、.pptx/.docx 等。产物可下载(前端层将 base64 转 blob)。

> (Go 前端这些展示逻辑在 Web 端 `src/` 侧;这里只交代契约——**前端不关心 sandbox 是 Go 还是 Python 调的,只消费统一的 `{stdout, stderr, exit_code, files}` 结构**。)

### 6.3 端到端流程(用户让 AI 跑代码 → 结果回前端)

```
① 用户在对话框输入 → AI 决定写代码
② Go 后端(server) 生成一段 Python → 调 SANDBOX_BASE_URL
③ sidecar(app.py): 若没有会话 → POST /sessions 创建锁定容器(含预热 matplotlib)
④ sidecar: POST /files 上传用户提供的输入文件(uploads)
⑤ sidecar: POST /exec {code} → docker exec 进容器:
       timeout ... python /tmp/aivory-cell-xxx.py
       - 容器内 /workspace 持久;pip 装过的包、中间文件保留
       - macOS/Windows 若无网络,Python 只能跑本地计算
⑥ sidecar 收集 /workspace/outputs 新产出 → base64 打包
⑦ sidecar 返回 {stdout, stderr, exit_code, files:[{name,mime_type,data_base64}]}
⑧ Go 后端(可先归档旧工作区)把结果转发给前端
⑨ 前端按 files[].mime_type 渲染(图片/HTML/表格/PDF 预览)+ 显示 stdout/stderr
⑩ 会话空闲超过 TTL → reaper 归档 /workspace 到对象存储并回收容器;
   下次同 conversation 再跑 → 从 <conv_id>.tgz 恢复工作区
```

---

## 7. Python 复刻要点

### 7.1 核心结论:原样复用,它就是新后端的天然一部分

这是本次调研最重要的结论。**`sandbox-service/app.py` 已经是用 Python(FastAPI)写的、与 Go 后端解耦的独立 HTTP 服务**,其唯一耦合面就是第 2 章的 HTTP 契约与 `Authorization: Bearer` 鉴权。

对于一个"把 Go 后端换 Python"的复刻工程:
- **把 Go 后端从 `sandbox.go` 换成 Python 客户端**:推理到 `POST /sessions → /exec → /files → ...`,完全不需要动 `sandbox-service` 的 app.py 一行;
- **`sandbox-service` 直接作为新服务保留**:它自己的 requirements.txt、Dockerfile.sidecar、docker-compose.yml 都不依赖 Go;
- 前端、部署拓扑、持久化归档逻辑全部透明——**HTTP 协议是新后端与沙箱之间唯一的技术边界**。

### 7.2 若你希望"零改动集成":直接保留现状

保留以下清单不动:
- `sandbox-service/app.py`(FastAPI sidecar);
- `Dockerfile.runner`(Python 运行时镜像)+ `runner-requirements.txt`;
- `Dockerfile.sidecar`(控制服务镜像)+ `requirements.txt`;
- `docker-compose.yml`(sidecar + keepalive + archives 卷)。

你的新 Python 后端只需:
```python
SANDBOX_BASE_URL = os.getenv("SANDBOX_BASE_URL")
SANDBOX_API_KEY  = os.getenv("SANDBOX_API_KEY")
# 用 httpx/requests 调:
#   POST {base}/sessions  -> session_id
#   POST {base}/files (上传输入)
#   POST {base}/exec {session_id, code, timeout_ms} -> stdout/stderr/exit_code/files
#   DELETE {base}/sessions/{id}
```

### 7.3 若你想把 sidecar 逻辑"内联"或重构进新后端

sidecar 的关键可复用模块(在新后端里可以直接抄这段逻辑):
- 会话生命周期 + `docker run` 安全参数(`app.py:805-911`);
- 超标量执行 `_run_exec_bounded` + `_BoundedOutput`(`app.py:564-616, 383-402`);
- 后台进程收割 `_kill_marked_processes`(`app.py:619-644`);
- 产物收集 `_collect_new_files`(`app.py:1460-1501`);
- 归档/恢复/GC、存储抽象(`app.py:1544-2040`);
- 空闲 reaper 线程(`app.py:2046-2088`)。

做一个"一键边车"或新后端内 `sandbox.py` 模块时,直接沿用这些函数。

### 7.4 若你想用 FastAPI 重写 / 微重构

- **直接就是 FastAPI**——无需从 Flask/自研迁移。若只想瘦身:
  - 可去掉强类型 Pydantic 而保留 dict,但保留 schema 校验更安全;
  - 可把 2162 行单文件拆分成 `api.py`(路由)、`docker_exec.py`(执行)、`storage.py`(后端抽象)、`reaper.py`(回收)、`config.py`(env)。
- **并发模型**:保持 def 同步 handler(依赖 `subprocess` 阻塞),不要盲改成 async 读管道;若需高并发可换 `asyncio` + `create_subprocess_exec` 但会改动大量代码——**不必要,单 host dev 模型下线程池已足够**。
- 关键改动区若重写,务必保留的防护:session id 正则、`_safe_under_workspace` + realpath 双层防越狱、`AIVORY_CELL_TOKEN` 收割、`MAX_*` 全部体积护栏、Content-Length 前置守卫。

### 7.5 隔离引擎:保持 Docker 还是升级?

**现状是 Docker 容器(非 gVisor/微VM)**。原代码注释(`app.py:28-29`)自己就说明了演进方向:"This is a DEV / single-host tool. For production swap `docker run` here for a gVisor / microVM / E2B backend——the Go side and this HTTP contract do not move."

| 方案 | 优势 | 代价 | 适合 |
|------|------|------|------|
| **保留 `docker run`(现状)** | 零改动、成熟、资源开销小 | docker.sock = root,依赖 API key + socket 代理 | 单机/团队自托管 |
| **gVisor(runsc)** | 更强的系统调用沙箱,`--runtime=runsc` 一行切换 | 装 runsc 到宿主、内核网络无 → 与前端 `--network none` 配合极佳 | 单机到生产过渡 |
| **Firecracker / microVM** | 硬件级隔离 | 需 VMM 管理、镜像转换、复杂度高 | 严格多租户 |
| **E2B / 外部沙箱 API** | 免运维、强隔离 | 依赖第三方、数据出网 | 无自托管约束 |
| **受限容器(仅 seccomp/apparmor 不换 runtime)** | 最轻 | 隔离弱于上面 | 信任代码的团队内部 |

**复刻建议**:先在 `app.py` 的 `docker run` 基础上,默认 `--runtime=runsc`(gVisor)加 `SANDBOX_SECCOMP_PROFILE`;如需更强再抽象出 `executor` 接口(gVisor/firecracker/E2B 三实现共用同一 HTTP contract)。

### 7.6 保持 HTTP 协议兼容 = 前端与后端零改动

关键纪律:**不要把任何协议字段改名/删减**。前端和 Go 后端依赖的契约字段:
- `/sessions` → `session_id`;
- `/exec` → `stdout`, `stderr`, `exit_code`, `files[]`(`name`, `mime_type`, `data_base64`);
- `/files` → `{path, data_base64}`(base64,非 multipart);
- `/files/reset-inputs` → `{ok, session_gone}`;
- DELETE `/sessions/{id}` body `{discard}`;
- `/storage/gc` → `{deleted, scanned, freed_bytes}`;
- `Authorization: Bearer` 鉴权 + `/healthz` 免鉴权。

**只要新后端继续通过这套 HTTP 契约对话,那么前端展示逻辑(按 `mime_type` 渲染)与旧 Go 后端迁移逻辑都完全无需改动。**

---

## 附:关键源码摘录速查

- **服务定位/协议**:`app.py:1-30`
- **全部环境变量配置**:`app.py:59-200`
- **FastAPI 入口**:`app.py:302`
- **Fail-closed 鉴权启动**:`app.py:310-326`
- **docker 封装与 session 校验**:`app.py:330-363`
- **有界输出缓冲**:`app.py:383-402`
- **会话锁/信号量**:`app.py:405-436`
- **创建会话(安全参数)**:`app.py:796-911`
- **执行 `/exec`**:`app.py:914-983`
- **超标量执行循环**:`app.py:564-616`
- **后台进程收割**:`app.py:619-644`
- **matplotlib 预热**:`app.py:652-718`
- **请求模型**:`app.py:722-793`
- **上传 `/files`(防越狱)**:`app.py:986-1033`
- **取回 `/files/get`**:`app.py:1124-1155`
- **归档 tar 扫描安全**:`app.py:1209-1271`
- **存储抽象与归档/恢复/GC**:`app.py:1544-2040`
- **空闲 reaper**:`app.py:2042-2088`
- **鉴权中间件**:`app.py:2135-2140`
- **体长守卫**:`app.py:2146-2162`
- **运行镜像**:`Dockerfile.runner`
- **sidecar 镜像**:`Dockerfile.sidecar`
- **部署与安全说明**:`docker-compose.yml`
