# 技能（Skill）与 MCP 分级：公共可用 / 用户私有 / 工作区共享

> 场景：Aivory 支持多用户，技能（Skills）与 MCP Server 是**可执行的资产**（skill 含 instructions + 工具配置，MCP 是可连接的远程工具网关）。它们必须回答：**哪些 skill/MCP 是全平台公共可用？哪些是某用户私有的？哪些是某个工作区成员共享的？管理员如何统一治理？**
> 本文档回答：**本项目对 skill 与 mcp 是否分成了「公共可用 / 用户私有」等不同级别？分级由哪些数据字段与权限谓词决定？**
>
> 源码基准：Aivory v2.5.2-beta.12（Go 后端）。配套：《多用户写删除操作隔离分析》《后端-03-工具与沙箱与MCP》《agent-subagent流程与多用户隔离分析》。

---

## 0. 一句话结论（先给答案）

**分了。skill 与 mcp 都有清晰的「三级/多级」可见性与治理模型：**

| 级别 | skill | mcp | 谁能写 | 谁能用 | 判定字段 |
|---|---|---|---|---|---|
| **① 平台级（公共）** | 内置/公开技能（catalog） | 全局 MCP（channel） | **仅 admin** | 全体（受用户组 policy 限制） | admin 路由 `/api/admin/skills`、`/api/admin/mcp` |
| **② 用户私有** | 个人技能 | 个人 MCP Server | **仅 owner** | 仅 owner | `user_id` + `workspace_id=''` |
| **③ 工作区共享** | 工作区技能 | （同 skills 模式） | **owner + 有 `can_create_skills` 的成员** | 工作区成员（受 member 权限位） | `user_id + workspace_id` 复合键 |

- **区分「公共」与「私有」的最小单位是 `workspace_id`**：`workspace_id=''` ⇒ 用户/平台私有；`workspace_id` 非空 ⇒ 某个工作区成员共享。
- **区分「平台 vs 个人」的加工机关是路由中间件**：`/api/admin/*` 用 `requireAdmin`，`/api/me/*` 用 `requireAuth`。
- **「能不能用」「成员的 instructions 会不会被别的成员看到」** 由**用户组 policy + workspace 能力位**双重治理，删/改由 `CanManage` 决定。

---

## 1. 数据模型：两种「私有/共享」载体

### 1.1 技能（user_skills）

- 表 `user_skills`（用户在 `internal/store/store.go:341` 迁移加了 `workspace_id`）：
  - `user_id`：拥有者。
  - `workspace_id`：空 ⇒ 用户私有；非空 ⇒ 该工作区成员共享（`store.go:341` `ALTER TABLE user_skills ADD COLUMN workspace_id`）。
- 另有 `skills`（平台 catalog / admin 维护）、`user_prompts`（同 skills 模式，`store.go:342` 同样有 `workspace_id`）。
- **注意**：`user_id` + `workspace_id` 构成**复合归属键**，不是只靠 `user_id`。workspace 技能仍记录 `user_id`（创建者），但可被同工作区的授权成员使用。

### 1.2 MCP Server（user_mcp_servers + 全局 mcp）

- 用户级：`server/internal/store/user_mcp_servers.go` 的 `user_mcp_servers` 带 `user_id + workspace_id`，同 skills 分私有/共享。
- 平台级：`/api/admin/mcp`（`requireAdmin`）管理的全局 MCP Server 列表，全平台可用。
- 模型绑定：`model_skills` / channel 模型与 skill/MCP 的绑定关系决定「哪个模型切到哪些工具」。

---

## 2. 路由分级：`requireAuth` vs `requireAdmin` vs `/api/me`

从 `server/internal/api/router.go` 抽出的分级对照：

```go
// —— 平台公共 / catalog（登录即可读）
mux.handle("GET", "/api/skills",             requireAuth(d, listSkillsPublicHandler))   // :330
mux.handle("GET", "/api/library/catalog",    requireAuth(d, listLibraryCatalogHandler)) // :333

// —— 用户私有 / 工作区共享（自助 CRUD，带 workspace_id 派发）
mux.handle("GET",    "/api/me/skills",       requireAuth(d, listMySkillsHandler))         // :335
mux.handle("POST",   "/api/me/skills",       requireAuth(d, createMySkillHandler))        // :336
mux.handle("POST",   "/api/me/skills/from-catalog", requireAuth(d, copySkillFromCatalogHandler)) // :337 (把公共 skill 复制成自己的)
mux.handle("PATCH",  "/api/me/skills/:id",   requireAuth(d, updateMySkillHandler))        // :338
mux.handle("DELETE", "/api/me/skills/:id",   requireAuth(d, deleteMySkillHandler))        // :339
mux.handle("GET/POST/PATCH/DELETE", "/api/me/mcps[/:id[/test|/sync]]", requireAuth(...))  // :347-352

// —— 管理（仅 admin）
mux.handle("GET/POST/PATCH/DELETE[/:id]",
           "/api/admin/mcp[/:id[/test|/sync]]", requireAdmin(d, ...))                     // :535-540
mux.handle("GET/POST/PATCH/DELETE[/:id]/assets/reorder",
           "/api/admin/skills", requireAdmin(d, ...))                                     // :593-600
// ↓ v2.5.2-beta.12 之后新增（导入功能，仍为 requireAdmin，不改变分级模型）
mux.handle("POST", "/api/admin/skills/import-archive", requireAdmin(d, importSkillArchiveAdmin)) // :596
mux.handle("POST", "/api/admin/skills/import-remote",  requireAdmin(d, importSkillRemoteAdmin))  // :597
mux.handle("GET/POST/PATCH[/:id]/reorder", "/api/admin/prompts", requireAdmin(d, ...))    // :601-605
```

> **行号说明**：以上行号取自当前 `main`（`33dddcef`）。相对文档最初的 `v2.5.2-beta.12` 快照，`/api/skills` 之前的行号整体 **+4**，原因是 `Deps` 结构体新增了 `SkillImportHTTPClient *http.Client` 字段（`router.go:63-66`）；`/api/admin/skills` 之后的行号再 **+2**，即两个新增的 import 端点。


**分级语义**：
- `requireAdmin`（`middleware.go`）＝ 平台级治理，只有管理员能增删改全局 skill/MCP/prompt。
- `requireAuth` + `/api/me/*` ＝ 普通用户自助管理**自己的**私有/工作区资产。
- `/api/skills`、`/api/library/catalog` ＝ **只读的公共/目录视图**，登录用户都能浏览、可 `copySkillFromCatalog` **复制成自己的**（`library_handlers.go:638`）。

---

## 3. 谁能读、谁能用：用户组 policy + workspace 能力位

### 3.1 用户私有技能的「能否用」由用户组 policy 控制

`listMySkillsHandler`（`library_handlers.go:468-518`）：

```go
// 个人私有技能：受用户组 policy 约束
if workspaceID == "" {
    if !store.UserSkillPolicyAllows(permissions.Skills, *skill) { ... 过滤掉 ... }
}
```

- `permissions.Skills` 是当前用户组的 skill 策略；`UserSkillPolicyAllows` 判断该用户组是否允许使用某一技能。
- **不能用的成员拿不到他人/共享技能的 instructions**（`library_handlers.go` 在返回前按 policy 过滤，避免把 instructions 泄露给无权用户）。

### 3.2 工作区共享技能由 workspace 成员能力位治理

- 成员权限列（`store.go:299-304` 附近）：`can_create_skills` / `can_use_skills` / `can_create_skills_prompts`。
- 工作区查询统一带 `libraryWorkspaceReadPredicate` / `libraryWorkspaceCapabilityPredicate`（见 `user_mcp_servers.go`、`library_handlers.go`）：**只有「对该工作区有对应能力位」的成员才查得到、用得上**。
- 改/删由 `current.CanManage`（`library_handlers.go:608` 的 `deleteMySkillHandler`）决定 —— owner 或被授予 manager 能力的成员可管理，普通成员只读/仅用。

> **关键**：分级不仅是「可见性」，还包括「执行授权」。共享技能/全局 MCP 的**工具调用**在 `Orchestrator` 里也要经过 `registry` 的 `allowlist`（见 `registry_allowlist_test.go`、`registry_user_mcp_test.go`）——工具没被允许给当前模型/用户组，就不会被派发。

---

## 4. 一张 mermaid 收束分级

```mermaid
flowchart TB
    subgraph C1[level 1 平台级 / admin]
        A1[/api/admin/skills/] --> A2[全局技能 catalog]
        A3[/api/admin/mcp/]   --> A4[全局 MCP Server]
        A5[/api/admin/prompts/] --> A6[全局 prompts]
        A7[模型绑定 model_skills/channel] --> A8[哪个模型切哪些公共工具]
    end

    subgraph C2[level 2 用户私有 workspace_id='']
        B1[/api/me/skills/] --> B2[个人技能 owner-only]
        B3[/api/me/mcps/]   --> B4[个人 MCP owner-only]
        B5[用户组 policy UserSkillPolicyAllows] -->|过滤| B2
    end

    subgraph C3[level 3 工作区共享 workspace_id 非空]
        D1[user_skills/user_mcp_servers 带 workspace_id] --> D2[成员共享]
        D3[member 能力位 can_create_skills/can_use_skills] -->|可见/可用| D2
        D4[CanManage] -->|改/删| D2
        D5[libraryWorkspaceReadPredicate<br/>libraryWorkspaceCapabilityPredicate] --> D3
    end

    C1 -.公开/复制.-> C2  ;; 公共可 copy 成私有
    C2 -.升级/共享.-> C3
```

---

## 5. 对「Python 后端复刻」的要点

1. **路由分级照搬**：`admin` 前缀用 `Depends(require_admin)`；`/api/me/*` 用 `Depends(get_current_user)`；`/api/skills`、`/api/library/catalog` 只读公开目录。
2. **表结构带 `workspace_id`**：`user_skills`、`user_mcp_servers`、`user_prompts` 都要有 `workspace_id TEXT NOT NULL DEFAULT ''`；**归属判定用 `user_id + workspace_id` 复合键**，空串=私有，非空=工作区共享。
3. **两级授权谓词**：
   - 用户组 policy：`user_group_policy_allows(permissions.skills, skill)` —— 私有技能「能否用」。
   - workspace 能力位：`can_create_skills / can_use_skills / can_create_skills_prompts`，查询加 `library_workspace_read_predicate(capability)` 过滤。
4. **改/删用 `can_manage`**：`get_user_skill_scoped(id, user_id, workspace_id)` 返回 `can_manage`，不满足即 403（对齐 `deleteMySkillHandler`）。
5. **公共目录可复制**：`POST /api/me/skills/from-catalog` 把平台 catalog skill 复制成用户的（`copySkillFromCatalogHandler`），Python 端实现 `POST /me/skills/from-catalog`。
6. **工具执行端再过滤**：`Orchestrator` 派发工具前经 registry allowlist（`registry_allowlist`），全局/共享 MCP 工具也要按「模型 + 用户组」判定是否可派发，避免跨用户/跨模型误用。
7. **隔离底线**：任何 `get_skill/get_mcp` 都带 `user_id(+workspace_id)` 过滤；**不能用的用户看不到他人 instructions**（返回前按 policy 过滤，避免泄露 prompt 内容）。
8. **导入功能（5.5 节）不影响分级模型**：两个 import 端点都是 `requireAdmin`，只产出「预览 + 已暂存资产」，**不写 `skills` 表**；最终仍走 `createSkillAdmin` → `validateSkillAssets` 的同一套校验。

---

## 5.5 技能导入（`admin_skill_import.go`）—— v2.5.2-beta.12 之后新增

> 这是文档最初基线之后新增的模块（761 行）。它在**不改变上述三级权限模型**的前提下，给管理员提供「从 zip / URL 导入技能」的入口。

### 5.5.1 两个端点

| 端点 | 方法 | 入参 | 鉴权 |
|---|---|---|---|
| `/api/admin/skills/import-archive` | POST | `multipart/form-data`，字段 `file`（zip） | `requireAdmin` |
| `/api/admin/skills/import-remote` | POST | JSON `{"url": "..."}` | `requireAdmin` |

两者返回同一 wire 结构 `skillImportPreview`（`admin_skill_import.go:55`）：

```json
{ "name": "...", "description": "...", "instructions": "...",
  "assets": [ ... ], "warnings": ["...", "..."],
  "name_exists": false, "source_summary": "..." }
```

### 5.5.2 关键设计：只预览，不落库

`importSkillArchiveAdmin`（`:528`）与 `importSkillRemoteAdmin`（`:660`）都遵循：

1. **解析全部成功才写盘**（all-or-nothing，`:67` 注释）——任何一个资产校验失败即整体报错。
2. 资产经 `persistSkillImportAssets`（`:462`）暂存，复用与手动上传相同的 skill-assets 管线。
3. **不写 `skills` 表**；返回预览后由前端填表，保存仍走 `createSkillAdmin` 及其 `validateSkillAssets` 复核（`admin_skill_import.go:26-29` 注释明确说明这是为了防止导入绕过手写技能的既有保证）。

> 复刻要点：Python 端必须保留「导入 = 预览」这个语义。若图省事让 import 直接创建记录，就会绕过 `validateSkillAssets`，与手动创建路径产生行为差异。

### 5.5.3 限额（`admin_skill_import.go:31-44`）

| 常量 | 值 | 用途 |
|---|---|---|
| `maxSkillImportZipBytes` | 64 MiB | 上传/下载的 zip 本体上限 |
| `maxSkillImportExtractBytes` | 100 MiB | 解压后总字节上限（防 zip bomb） |
| `maxSkillImportEntries` | 500 | 参与处理的条目数上限 |
| `maxSkillImportMarkdownBytes` | 1 MiB | 远程拉取的单个 `SKILL.md` 上限 |
| `skillImportHTTPTimeout` | 45 s | 单次远程抓取（含重定向）超时 |

### 5.5.4 SKILL.md 解析（`:137` `parseSkillMarkdown`）

解析 Agent Skills 规范里 `SKILL.md` 的子集：

- 开头 YAML frontmatter，取 `name` 与 `description`；
- 标量支持单行，也支持 `|`（literal）与 `>`（folded）**块标量**（`foldYAMLBlockLines`，`:113`）；
- 正文即 `instructions`；
- 不认识的键（`license`、`allowed-tools`、`metadata` 等）忽略。
- 与前端 `parseSkillDocument` 保持一致，但**额外支持多行 description**。

### 5.5.5 zip 处理与路径安全

- `cleanZipEntryPath`（`:219`）：以虚拟根 clean，`..` 被吸收，空结果（目录项）拒绝。
- `skipZipEntryName`（`:229`）：过滤 macOS 打包噪音与 dotfile。
- `stripTopDir`（`:245`）：GitHub archive 会把内容包在 `<repo>-<ref>/` 里，剥掉这一层。
- 资产保留子目录结构（如 `scripts/office/helpers/__init__.py`），配合 `fileguard.SanitizeRelPath`（见《后端-03》）双重把关。

### 5.5.6 URL 形态（`resolveSkillImportURL`，`:581`）

- 只接受 `http`/`https`；
- `github.com` / `www.github.com` 的仓库页、子目录页、blob 页会**重写**为 archive/raw 等价地址；
- 其他 URL 原样抓取，再按后缀 / `Content-Type` / magic bytes（`looksLikeZip`，`:653`）判定是 zip 还是纯 `SKILL.md`。
- HTTP 客户端走 **`netsafe.SafeClient(skillImportHTTPTimeout)`**（`:647`）——SSRF 加固，禁止内网地址。`Deps.SkillImportHTTPClient`（`router.go:63`）仅供测试注入。

### 5.5.7 复刻检查清单

- [ ] 两个端点均 `require_admin`
- [ ] 返回 `skillImportPreview` 八个字段（含 `name_exists` 用于前端提示重名）
- [ ] all-or-nothing：解析失败不留半成品文件
- [ ] **不写 skills 表**，保存仍走 create + validate
- [ ] 五项限额齐全，zip bomb 防护
- [ ] zip 路径清洗 + 子目录保留
- [ ] GitHub URL 重写规则
- [ ] 远程抓取走 SSRF 安全的 HTTP 客户端

---

## 6. 参考源码

- 路由分级：`server/internal/api/router.go:330-339, 347-352, 535-540, 593-605`
- 迁移（workspace_id）：`server/internal/store/store.go:341-342`
- 用户组 policy / 能力位：`library_handlers.go:468`（listMySkillsHandler）、`UserSkillPolicyAllows`、`workspace_members` 的 `can_create_skills/can_use_skills`
- workspace 谓词：`user_mcp_servers.go`（`libraryWorkspaceReadPredicate`/`libraryWorkspaceCapabilityPredicate`）
- 改删：`library_handlers.go:608`（deleteMySkillHandler，`CanManage`）
- 公共目录复制：`library_handlers.go:638`（copySkillFromCatalogHandler）
- 工具派发过滤：`internal/tools/registry_allowlist_test.go`、`registry_user_mcp_test.go`
- 技能导入：`server/internal/api/admin_skill_import.go`（见第 5.5 节）
- 配套：《多用户写删除操作隔离分析》《后端-03-工具与沙箱与MCP》《agent-subagent流程与多用户隔离分析》《99-复刻路线图-Python后端》
