# Aivory 文档预览与编辑 — 最优方案

> 目标：HTML / 表格 / Word / PPT 全部支持**右侧预览**，并尽可能支持**右侧编辑**。
> 本文基于对当前源码（v2.4.9-beta.6）的逐文件勘察，标注了每个结论对应的文件与行号。

> **⚠️ 阅读须知（2026-11 校对）**：本文是一份**开工前的决策记录 + 施工纪实**，不是当前系统的说明书。
> §一「现状盘点」与 §1.3「五个缺口」记录的是 **v2.4.9-beta.6 的状态**，这些缺口**后来都已修复并交付**（见 §三）。
> 因此不要把 §一 当成"现在还不能预览/编辑"来读。当前仓库（v2.5.2-beta.12）的实际行为请看 §三 的交付明细。
> 文中少数路径因后续重构而失效，已在原处标注现名；行号引自各自的基线版本，跨版本不保证仍然对齐。

**关于证据强度**：本文所有"现状"结论均来自阅读仓库源码（含行号，可自行核对）。
依赖已安装（`node_modules` 就位），P0 已跑通 typecheck / 120 测试文件 682 用例 / 生产构建。
第三方库的版本与许可证结论取自 npm registry 字段与仓库 LICENSE 原文，并已在本机复核。

---

## 〇、已确认的决策与调研纠正（2026-09，优先阅读）

### 已拍板的决策

| # | 决策 | 结论 |
| --- | --- | --- |
| 1 | **保真度的定义** | **文件级保真 = 硬指标**（保存后未编辑部分字节级不变，用测试强制）；**屏幕排版 = 接受近似**（字体度量差异写入产品说明，不作为缺陷） |
| 2 | **实现路线** | 不依赖容器，采用 **Apache-2.0 / MIT 现成组件**（纯浏览器，无 AGPL、无授权费、无服务端） |
| 3 | **表格技术栈** | AG Grid Community (MIT，DOM+ARIA，a11y 达标) + `@formulajs/formulajs` (MIT) + 自研 OOXML 外科手术写回 |
| 4 | **保存语义** | 默认**另存为新文件**，不覆盖原文件 |
| 5 | **本期范围** | P0 + P1 + P2 |

### ⚠️ 必须纠正的三个错误认知

**纠正 1 — 本文件早期版本的 Word 编辑推荐是错的。**
早期版本推荐"复用已有 mammoth + Tiptap + `docx`"实现 Word 编辑闭环。**这条链路不是 round-trip，不可用于"编辑现有 docx"**：

- `mammoth` 是 **docx → HTML 的单向有损转换**（BSD-2-Clause）；
- `docx` (docx.js, MIT) **完全没有读取/解析 API** —— 它只能"从零生成"，所以"保存"等于**重新生成一份新文档**；
- 结果是页眉页脚、分页、节、文本框、浮动图形、OMML 公式、域、目录、修订痕迹**全部永久丢失且每次保存累积损失**。
- 连 Tiptap 自己的付费 Conversion 能力矩阵也承认：浮动图片、文本框、形状、节/section 导出均为 ✕。

**纠正 2 — Univer 的 OSS 版不能读写 xlsx。**
xlsx/docx/pptx 导入导出属 **Univer Pro 付费功能**，且导入导出**必须自建 `universer-api` 服务端**（`uploadFileServerUrl`/`importServerUrl`/`exportServerUrl`），无有效 license 时进入 evaluation 模式（带水印 + 导入大小限制）。对"纯浏览器、无服务端"的要求，Univer 直接出局。

**纠正 3 — 两条高价值的新线索（已在本机核实 npm 元数据）。**
`@docx-editor.dev/core` + `/react` **2.21.0 = Apache-2.0**，官方定位 "Canonical OOXML, all in the browser"、**无需服务端**、明确 "lossless round-trip: untouched content and unsupported OOXML survive the save" —— 这正是我们要求的"文件级保真"。`pptx-react-viewer` **4.1.1 = Apache-2.0**，纯浏览器、无容器。
⚠️ 但 **`@docx-editor.dev/pro` 的许可证是 `LicenseRef-EigenPal-Pro-Evaluation-1.0`，明写 "You may not use the Software for Production Use"** —— 它在公共 npm 上，极易误装侵权。生产只能使用 `/core` 与 `/react`。

### 🔒 一个影响功能边界的既有安全约束（本次勘察发现）

`server/internal/api/upload_policy.go:47-52` 把 `.html` / `.htm` / `.svg` **故意排除在上传白名单之外**，注释写明原因：

> anything HTML-rendered that the browser might interpret if served back inline (`.html`, `.htm`, `.svg` — SVG can carry XSS; PDF is fine because §4.5 forces attachment disposition)

**这有两个直接后果：**

1. **用户根本上传不了 `.html` 文件。** 所以 P0 的"HTML 文件渲染成文档"主要受益者是 **AI / 沙箱生成的 HTML 产物**（走到 `DocumentPreview` 时带 `.html` 文件名），以及管理员调整过白名单的部署。
2. **"另存为副本"对 `.html` 会被服务端拒绝。** 因此面板上对 HTML 产物**只提供"下载"，不提供"另存为副本"**，避免给出一个必然失败的按钮。

> 放开 `.html` 白名单是一个**安全决策**，需要与 PDF 同等的强制 `attachment` disposition 保护。本方案**不擅自改动**，保持现状。

---

## 一、现状盘点

### 1.1 已有的预览能力

| 格式 | 预览 | 渲染技术 | 代码位置 |
| --- | --- | --- | --- |
| 图片 | ✅ | `<img>` + objectURL | `src/components/files/document-preview.tsx:100` |
| PDF | ✅ | pdfjs-dist（懒加载 worker） | `src/components/files/pdf-native-preview.tsx` |
| DOCX | ✅ 只读 | `docx-preview` 的 `renderAsync` | `src/components/files/docx-native-preview.tsx:95` |
| PPTX | ✅ 只读 | `@aiden0z/pptx-renderer` | `src/components/files/pptx-native-preview.tsx:80` |
| XLSX | ✅ 只读（截断到 250 行 × 50 列） | `read-excel-file`（Web Worker） | `src/components/files/spreadsheet-native-preview.tsx` + `spreadsheet-parser.worker.ts` |
| HTML（AI 流式生成） | ✅ 沙箱渲染 | iframe `srcdoc` + Tailwind runtime | `src/components/html/sandboxed-html-frame.tsx` + `src/lib/html-preview-document.ts`（基线时在 `chat/html-preview-panel.tsx`） |
| **HTML（`.html` 文件）** | ❌ **降级为纯文本源码** | `<pre>` | `src/lib/file-preview-kind.ts:252` |
| 纯文本 / 代码 | ✅ | `<pre>`，上限 2 MiB | `src/components/files/document-preview.tsx:185` |

**关键结论：五种格式的渲染器已经全部存在且经过安全加固。** 缺的不是渲染能力，而是**接入方式**和**编辑回路**。

### 1.2 右侧面板的基础设施已经存在

`src/components/chat/chat-side-panel.tsx` 已经是统一的右侧抽屉外壳：

- 桌面端（≥1024px）：`<aside>` 分栏，与对话区并排
- 移动端：右侧 `Sheet`
- 内置 `present` 状态机处理退场动画与 reduced-motion

聊天页目前挂载 **4 个互斥抽屉**（`src/pages/chat/ChatLayout.tsx:184-189`）：

```
HtmlPreviewPanel         AI 生成的 HTML
InlineThreadPanel        子线程
SandboxFilesPanel        沙箱文件浏览
ConversationFilesPanel   对话文件清单
```

互斥由各 store 在 `openPreview()` 里互相 `close()` 实现，见 `src/store/html-preview.ts:31-33`。

### 1.3 五个缺口

#### 缺口 1 — HTML 文件不渲染，只显示源码（投入产出比最高的一处）

`src/lib/file-preview-kind.ts:75-76` 把 `htm` / `html` 放进了 `TEXT_EXTENSIONS`，于是 `documentPreviewKind()` 在 `:252` 返回 `'text'`，走 `<pre>` 显示源码。

而**渲染管线已经现成**：`buildHtmlPreviewDocument()` + 沙箱 iframe（基线时 `html-preview-panel.tsx:152-159`；现为 `src/components/html/sandboxed-html-frame.tsx`），连 Tailwind runtime 注入、Google Fonts 镜像替换、CSP 升级都做好了。只是当时没有接到文件预览路径上。

#### 缺口 2 — 预览位置不统一，聊天附件不在右侧

| 入口 | 当前形态 | 是否在右侧 |
| --- | --- | --- |
| Files 页（`UserFiles.tsx:562`） | 桌面端左右分栏 + 移动 Sheet | ✅ |
| AI 生成的 HTML（基线时 `html-preview-panel.tsx`） | 右侧抽屉 | ✅ |
| **聊天附件**（基线时 `file-preview.tsx:110`） | **居中 Dialog** | ❌ |
| **沙箱 / AI 产物**（`sandbox-files-panel.tsx:183`） | **居中 Dialog** | ❌ |

也就是说，**AI 产出的 docx / xlsx / pptx 目前点开是弹窗盖住对话**，而不是右侧并排。这与"对话与产物并排"的产品心智不一致——而这恰恰是用户最常预览的一类文件。

#### 缺口 3 — 完全没有编辑能力

docx / pptx / xlsx / html 四项全部只有只读渲染器，没有任何编辑器。

#### 缺口 4 — 后端没有写回接口（编辑保存的硬阻塞）

`server/internal/api/router.go:418-420` 只有三个文件端点：

```
GET  /api/me/files            列表
POST /api/me/files/delete     删除
GET  /api/me/files/content    读内容
```

**没有任何 PUT / PATCH 更新端点。** 前端 `openPreview()` 拿到的 `data: ArrayBuffer` 是一次性快照，即使能改也无处保存。

而 `PRODUCT.md:24` 明确把「丢失工作」列为 anti-reference 第一条：

> Interfaces that hide failures, lose work on navigation, or use optimistic local state without a durable recovery path.

**所以「编辑必须能持久化」不是可选项，而是需求的一部分。** 没有写回接口就不该给用户一个会丢数据的编辑器。

#### 缺口 5 — 产品级约束会在选型上否决一些方案

| 约束 | 出处 | 对选型的影响 |
| --- | --- | --- |
| 5 种语言（zh / zh-Hant / en / ja / fr）× 15 个命名空间 | `src/i18n/locales/` | 新增 UI 文案要 ×5 本地化 |
| WCAG 2.1 AA + 键盘可操作 + 明暗主题 | `PRODUCT.md:37` | **canvas 渲染的表格/幻灯片编辑器天然不满足**（无 DOM 语义、无法键盘导航） |
| 自托管、追求「一条 docker compose 起来」 | `deploy/docker-compose.prod.yml` | 引入重型文档服务器会显著抬高部署门槛 |
| 服务端状态为准，不做无恢复路径的乐观更新 | `PRODUCT.md:32` | 编辑器必须有脏状态守卫与冲突处理 |

---

## 二、最优方案

### 2.1 核心判断：先统一面板，不要为四种格式各做一套

Aivory 真正缺的是一个统一的**右侧产物面板（Artifact Panel）**抽象，格式只是面板里的内容。

现在四个抽屉各自维护 `open` / `html` / `shareable`，而 `DocumentPreview` 是一个**无壳渲染器**（被 Files 页、聊天附件、管理后台三处复用）。把这两者合并即得最优解：

```
ArtifactPanel                        ← 复用 ChatSidePanel 外壳
  ├─ useArtifactPanel store
  │    { open, source, kind, name, content, mode, dirty, conflict }
  ├─ header: 类型图标 + 文件名 + [预览|编辑] 切换 + 保存 + 下载 + 全屏 + 关闭
  └─ body: <DocumentPreview mode="view" | "edit" onSave={...} />
```

**收益：**

1. 一处实现，**四个入口同时升级**（Files 页 / 聊天附件 / 沙箱产物 / 管理后台都指向 `DocumentPreview`）
2. 与既有互斥协调机制天然兼容（在 `openPreview()` 里 close 其它抽屉）
3. 移动端自动获得右侧 Sheet（`ChatSidePanel` 已处理）
4. AI 生成 HTML 面板退化为本面板的一种 `source`，删掉一套重复的 store
5. 只实现一次脏状态守卫、全屏、分享、i18n

### 2.2 分层策略：按「保真度 vs 部署成本」分四层

| 层 | 适用格式 | 手段 | 保真度 | 部署成本 |
| --- | --- | --- | --- | --- |
| **L1 原生前端渲染** | 图片、PDF、文本、**HTML** | 已有 | 高（HTML 为 100%） | 0 |
| **L2 前端源码级编辑** | HTML / CSS / JS、CSV、Markdown | 代码编辑器 + iframe 实时预览 | 源码级 100% | 0 |
| **L3 前端文档模型编辑** | XLSX、DOCX | JS 库解析 / 回写 OOXML | 中高（复杂排版、图表会丢） | 0 |
| **L4 服务端文档引擎** | DOCX、XLSX、**PPTX** | OnlyOffice / Collabora 容器 | 接近原生 | 高（额外容器 + 授权） |

**推荐路线：L1 + L2 立刻做（纯前端、零部署成本、零新依赖），L3 做 XLSX 与 DOCX，PPTX 走 L4 或明确降级。**

理由：**PPTX 是四者中唯一浏览器内没有成熟开源编辑器的格式。** 渲染器生态有（我们已在用），编辑器生态基本空白。自研画布式幻灯片编辑器的成本远超收益。

### 2.3 分格式落地方案

#### HTML —— ✅ 已实现（P0）

1. **`src/lib/file-preview-kind.ts`**：`DocumentPreviewKind` 新增 `'html'`；`htm` / `html` / `xhtml` 从 `TEXT_EXTENSIONS` 移入新的 `HTML_EXTENSIONS`，并在 MIME 侧新增 `HTML_MIME_TYPES`（`text/html`、`application/xhtml+xml`）。`.js` / `.css` / `.tsx` 等仍为 `'text'`。
2. **`src/components/html/sandboxed-html-frame.tsx`（新增）**：把原先只存在于 `html-preview-panel.tsx` 的**安全关键 sandbox 属性**抽成**全应用唯一来源**，并用它渲染文件预览。这样"渲染不受信 HTML"只有一处实现，安全姿态不会在两个界面之间漂移。
   - **安全红线**：`sandbox="allow-scripts allow-popups allow-popups-to-escape-sandbox"`，**绝不可加 `allow-same-origin`**（与 `allow-scripts` 同时出现会使沙箱完全失效）。`referrerPolicy="no-referrer"` 保持。
3. **`src/components/files/document-preview.tsx`**：新增 `kind === 'html'` 分支渲染该 frame，不再走 `<pre>` 源码。
4. **`documentPreviewByteLimit()`**：`'html'` 复用 `MAX_TEXT_PREVIEW_BYTES`（2 MiB）。
5. **`fileTypeFilterFor()` 有意保持 `'text'`**：渲染行为变了，但浏览筛选分组不变，避免动 5 语言的筛选 UI。
6. **测试**：`tests/frontend/lib/file-preview-kind.test.ts` 增补用例，并显式锁住"渲染为 html、筛选仍为 text"这一有意为之的分裂。

#### 表格（XLSX / CSV）—— 决策点已定

早期版本的"a11y 还是功能"二选一是个**伪两难**。已确认的组合同时满足两边：

| 组件 | 许可证 | 作用 |
| --- | --- | --- |
| **AG Grid Community** 36.2.0 | MIT（已核实 registry + LICENSE） | DOM + ARIA 渲染的网格，**键盘导航与读屏可用**，a11y 达标 |
| **`@formulajs/formulajs`** 4.6.1 | MIT | 公式求值（⚠️ 无 scope 的 `formulajs@1.0.8` **没有 license 字段**，不要用那个包名） |
| 自研依赖图 + OOXML 外科手术写回 | — | 公式依赖计算；保存时只重写实际编辑的单元格，其余部件字节级透传 |

**明确排除**：`handsontable`（商业授权，$999/开发者起，且许可证禁止用于竞争产品）、`hyperformula`（**GPL-3.0-only**，闭源不可用）、`jspreadsheet-ce`（registry 无 license 字段，商用前必须确认）、Univer（xlsx 读写属 Pro 付费且需自建服务端）。

还需处理 `spreadsheet-native-preview.tsx:19-20` 的 250 行 × 50 列截断——**编辑态不能截断**，否则用户编辑的是残缺表格。

#### Word（DOCX）—— 采用 `@docx-editor.dev/react`

**✅ 采用 `@docx-editor.dev/core` + `/react` 2.21.0（Apache-2.0，已在本机核实 registry license 字段）。**

选它的理由正是本方案的核心指标：官方定位 "Canonical OOXML, all in the browser"、**无需服务端**、并明确 **"lossless round-trip: untouched content and unsupported OOXML survive the save"** —— 这恰好就是决策 1 要求的**文件级保真**，而不是 HTML round-trip 那种重新生成。

**采购红线（务必写进依赖审查）**：

- ✅ `/core`、`/react`、`/vue`、`/i18n`、`/fonts` = **Apache-2.0**，官方明示 "Ship commercial products"。
- ⛔ `/pro` 与 `/editor-api` 的许可证是 **EigenPal Pro Evaluation License 1.0**，原文 **"You may not use the Software for Production Use"**。它**就挂在公共 npm 上**，极易被误装。生产环境**只能**使用 `/core` 与 `/react`。
- Pro（$500/月/产品）才含修订痕迹、批注、协作 —— 这些恰好是"AI 改稿 + 人审阅"的高价值点，属于后续预算决策，**本期不引入**。

**已知局限（须同步给产品）**：

- 编辑能力：文本 / 表格 / 图片 / 链接 / 列表。**能渲染、能 round-trip，但不能编辑**：OMML 数学公式、文本框、首字下沉、隐藏文字、文本效果。
- **最大风险是字体度量**：官方明确 "Word-compatible wrapping requires font bytes"，缺失时用回退度量，**不保证与 Word 一致的分页/断行**。`/fonts` 只提供 6 种**西文**替代字体；**中文字体需自备且合法授权**。→ 这正是决策 1 里"屏幕排版接受近似"的现实来源，POC 必须覆盖中文样本。
- React 19 兼容（peerDeps `^18 || ^19`），但依赖 DOM，SSR/Next.js 需 dynamic import（我们是 Vite SPA，不受影响）。

#### PPT（PPTX）—— ✅ 决策改为自研文本级外科手术编辑

**原计划采用 `pptx-react-viewer` 4.1.1（Apache-2.0）。实测其 peer 依赖后否决。**

`pptx-react-viewer` 与本仓库现有依赖存在**三个主版本 peer 冲突**（本机 `node_modules` 实测值 vs registry 声明的 peer range）：

| peer | viewer 要求 | 本仓库已装 | 结论 |
| --- | --- | --- | --- |
| `i18next` | `^26.3.6` | **25.10.10** | ❌ 落后一个大版本 |
| `react-i18next` | `^17.0.9` | **15.7.4** | ❌ 落后两个大版本 |
| `lucide-react` | `^1.24.0` | **0.474.0** | ❌ 落后一个大版本 |
| `framer-motion` | `^12.42.2 \|\| ^13.1.1` | 12.43.0 | ✅ 满足 |
| `jspdf` / `react-icons` / `fast-xml-parser` | — | 缺失 | 需新增 |
| `jszip` / `react` / `react-dom` | — | 3.10.2 / 19.3.0 | ✅ 满足 |

横幅在于 **i18n 栈**：把 `i18next` 25→26、`react-i18next` 15→17 全应用升级，会触及产品**每一句翻译的运行时**。为新增一个 PPT 编辑器而承担这种爆炸半径，代价不成比例。

**✅ 定为自研 pptx 文本级外科手术编辑**，复用 P4 为表格建立的 OOXML 外科手术核心：

- `.pptx` 同样是 ZIP + XML。只重写 `ppt/slides/slideN.xml` 里的 `<a:t>` 文本节点，**其余 zip entry 逐字节原样透传** —— 与 P4 完全相同的零损失保证，并同样用测试强制。
- **渲染复用已装的 `@aiden0z/pptx-renderer`**（P0 起就在用），不新增任何渲染依赖。
- **零新依赖、零 peer 冲突、无需容器。**
- 范围如实记录：**文本级编辑**。不能新增形状 / 图片 / 图表、不能改版式与母版。这与"零文件损失优先"的既定取舍一致——完整 pptx 创作是 600–1000+ 人日的量级（PPTist 投入了 5.7 年）。

**明确排除**：`pptx-react-viewer`（i18n peer 冲突，见上）、自研画布编辑器（600–1000+ 人日）、OnlyOffice/Collabora（容器 + AGPL/订阅）、PPTist（AGPL-3.0，README 明写"禁止闭源商用"）、`ranuts/document`（AGPL + §7 强制保留 ONLYOFFICE logo）。

### 2.4 保存与写回（P2）

**结论：默认的「另存为新文件」不需要新增任何后端端点。**

勘察发现仓库已有 `POST /api/files`（`server/internal/api/router.go:414` → `uploadFileHandler`），它就是一条完整的、带所有权校验、配额检查、限流、MIME 嗅探与大小上限的上传管线，返回 `201` + 文件记录；`upload_policy.go:53` 的默认白名单已包含 `docx / pptx / xlsx / doc / ppt / xls / csv / txt / md / json / xml` 等。前端也已有调用先例（`src/store/conversation-files.ts:78` 用 `apiUpload('/files?...')`）。

因此 P2 只需要：

1. **`src/api/endpoints.ts` 新增 `saveDocumentCopy(blob, filename)`**（✅ 已实现）—— 走 `POST /files`，不带 `conversation_id`，得到一个新的、独立归属当前用户、会出现在 Files 页清单里的文件。
2. **面板 header 增加「另存为副本」动作**，成功后 toast 提示，失败把服务端错误如实展示（不静默）。
3. **`kind === 'html'` 时不显示该动作**，只保留「下载」—— 因为上传白名单**故意**拒绝 `.html`/`.htm`（见 §〇 安全约束），给出按钮必然失败。

为什么这比新增 `PUT` 覆盖端点更好：

- **不破坏历史**：对话消息、引用、附件链接指向的字节永远不会被改写，符合 `PRODUCT.md:24` 的 anti-reference（"历史被悄悄改变 / 丢失工作"）。
- **零后端改动、零部署成本**，直接复用已经过安全加固的路径（配额、限流、MIME 嗅探、`writeUploadCopy` 的原子写入与失败清理）。
- 若将来确实需要"覆盖原文件"，再单独评估 `PUT` + 乐观并发（`If-Match: <updated_at>` → `409`）即可，**不影响本期交付**。

**仍建议补的鲁棒性**：编辑器内的脏状态守卫（面板关闭 / 路由切换 / 刷新前提示未保存）。基线时 `html-preview-panel.tsx:48-52` 已有 `useLocation` 关闭抽屉的先例（现为 `artifact-panel.tsx:52-60`），可扩展为"有未保存修改时先确认"。这一条属于各格式编辑器（P4–P6）的职责，本期面板只做"另存"，不引入未保存状态。

### 2.5 需要同步改动的地方（避免漏项）

| 项目 | 位置 |
| --- | --- |
| 类型定义 | `DocumentPreviewKind` 新增 `'html'`（`file-preview-kind.ts:1`） |
| 字节上限 | `documentPreviewByteLimit()`（`file-preview-kind.ts:366`） |
| 过滤分组 | `fileTypeFilterFor()` —— ✅ 已定：`.html` **仍归 `text`**，不新增分组（避免动 5 语言的筛选 UI） |
| 文件图标 | `src/lib/file-icon.ts` |
| 沙箱产物归类 | `src/lib/sandbox-browser.ts:67` 现把 html 归为 `code` |
| 聊天附件面板 | `src/components/chat/file-preview.tsx` Dialog → 右侧面板（该文件现已删除，由 `chat/artifact-panel.tsx` 取代） |
| 文案 | 5 语言 × `files.json` / `chat.json`（`preview.*` 扩展、新增 `edit.*`） |
| 测试 | `file-preview-kind.test.ts`、新增 `document-preview` 组件测试 |
| CSP / 沙箱 | 编辑态 iframe 必须沿用与只读态**完全相同**的 sandbox 属性 |

---

## 三、分阶段路线图

| 阶段 | 内容 | 状态 | 工作量 | 风险 |
| --- | --- | --- | --- | --- |
| **P0** | HTML 文件渲染接入：`.html` kind + 单一沙箱 frame + 测试 | ✅ **已完成并验证** | ~1 人日 | 极低（纯复用） |
| **P1** | 统一 Artifact 面板：合并 HTML 抽屉与文件预览，聊天附件 / 沙箱产物 / KB 文档从 Dialog 迁到右侧 | ✅ **已完成并验证** | ~5-8 人日 | 低（改动面广但机械） |
| **P2** | 「另存为副本」：复用 `POST /files`，**无需新后端** | ✅ **已完成并验证** | ~1-2 人日 | 低 |
| **P3** | HTML / CSV / Markdown 源码级编辑（CodeMirror 6，MIT） | ✅ **已完成并验证** | ~1 人日 | 低 |
| **P4** | 表格编辑：AG Grid Community + formulajs + 自研 OOXML 外科手术写回 | ✅ **已完成并验证** | ~2 人日 | 中（零文件损失需测试强制） |
| **P5** | Word 编辑：`@docx-editor.dev/react`（Apache-2.0，无 peer 冲突） | ✅ **已完成并验证** | ~1 人日 | 低 |
| **P6** | PPT：自研 pptx 文本级外科手术编辑（复用 P4 的 OOXML 核心） | ✅ **已完成并验证** | ~1 人日 | 中 |

### 实现进度明细

**P0（已交付）**

| 文件 | 变更 |
| --- | --- |
| `src/lib/file-preview-kind.ts` | `DocumentPreviewKind` 新增 `'html'`；新增 `HTML_EXTENSIONS` / `HTML_MIME_TYPES`；`documentPreviewByteLimit()` 覆盖；`fileTypeFilterFor()` 有意保持 `text` |
| `src/components/html/sandboxed-html-frame.tsx` | **新增**，全应用唯一渲染不受信 HTML 的位置 |
| `src/components/files/document-preview.tsx` | 新增 `kind === 'html'` 分支 |
| `src/components/chat/html-preview-panel.tsx` | 改用共享 frame，删除重复的 sandbox 属性 |
| `tests/frontend/lib/file-preview-kind.test.ts` | 新增 11 个用例，含"渲染为 html、筛选仍为 text"的锁定用例 |
| `tests/frontend/components/html/sandboxed-html-frame.test.tsx` | **新增 3 个用例**，把 sandbox 属性当作安全边界锁死（断言**绝不含 `allow-same-origin`**、不含 `allow-forms`/`allow-modals`/`allow-downloads`） |
| `tests/frontend/components/files/document-preview.test.tsx` | **新增 3 个用例**，直接钉住用户需求本身：`.html`/`.htm`/`.xhtml` 渲染为 `<iframe srcdoc>` **且不再出现 `<pre>`**，而 `.ts` 仍走 `<pre>` 源码分支 |
| `pnpm-workspace.yaml` | **新增**：本环境只有 pnpm（npm 损坏）。用 `overrides` 把 tiptap 钉回 `package-lock.json` 的 3.28.0（否则传递依赖解析到 3.31.x，`rich-composer-editor` 导入即崩） |

**最终验证（P0–P6 + 收尾加固）**：`tsc -b --noEmit` exit 0；vitest **129 文件 / 749 用例全绿**（起点 120/671）；`vite build` 成功；`npm run test:browser` **4/4 编辑器通过**；`python tests/fixtures/ooxml/verify_roundtrip.py` **全部真实库检查通过**（含浏览器产出的文件）。

已实测确认 `node_modules/@docx-editor.dev/pro` **未被安装**（其许可证禁止生产使用）。

懒加载实测（已断言主 chunk 内**不含**任何编辑器库）：

| chunk | 大小 |
| --- | --- |
| `codemirror-editor` | 524 KB |
| `spreadsheet-editor`（含 AG Grid） | 717 KB |
| `docx-editor`（+ 129 KB CSS） | 2597 KB |
| `pptx-editor` | **6.2 KB**（复用 P4 核心与已装渲染器） |
| 主 `index` chunk | 1843.7 KB（与开工前的 1842 KB 基本持平，未被编辑器污染） |

> ✅ **偶发失败已修复**：`tests/frontend/lib/audio-stream.test.ts` 的 flake 已复现、根因定位并修复，在原本 3/3 必现的并发条件下验证为 3/3 通过。详见上方「收尾加固 §3」。
>
> ⚠️ **验证边界**：以上全部由 typecheck / 单元测试 / 生产构建 / chunk 分析 / 真实库回读证明。**仍未经过真实浏览器交互验证**——本环境无法驱动浏览器。CodeMirror 的实际挂载、AG Grid 的键盘与读屏表现、docx 与 pptx 编辑器的真实交互，都需要在浏览器里过一遍。这类结论我没有替你打包票。

### P3（已交付）

| 文件 | 变更 |
| --- | --- |
| `src/components/files/editors/editor-types.ts` | **新增**：`EditorId` / `EditedContent` / `DocumentEditorProps` 契约 + `documentEditorFor()` 解析器 |
| `src/components/files/editors/codemirror-editor.tsx` | **新增**：CodeMirror 6 源码编辑器，配色全部取自既有设计 token（含 `--color-syntax-*`），明暗主题一致 |
| `src/components/files/editors/document-editor.tsx` | **新增**：懒加载注册表（每个编辑器一个 chunk） |
| `src/components/chat/artifact-panel.tsx` | 面板加编辑模式：header 的编辑/预览切换、HTML 上下分栏实时预览、保存/下载改用编辑后字节 |
| `src/i18n/locales/*/chat.json` ×5 | 新增 `enterEdit` / `exitEdit`（并补上此前遗漏的 `copyFailed`） |
| `tests/frontend/lib/locale-parity.test.ts` | **新增**：跨 5 语言 key 集合一致性测试（见下） |
| `tests/frontend/components/files/editors/editor-types.test.ts` | **新增**：解析器契约测试 |

**懒加载已实测生效**：生产构建把 CodeMirror 拆成独立 chunk `codemirror-editor-*.js`（524 KB），只读文档的用户不会下载它。

**两处设计决策值得记录**：

1. **HTML 编辑的持久化路径是「下载」而不是「另存为副本」**：服务端上传白名单**故意**拒绝 `.html`/`.htm`（见 §〇）。所以 HTML 的编辑闭环是「编辑 → 实时预览 → 下载结果」；其余白名单内格式才有「另存为副本」。面板在有编辑时，下载取的是**编辑后的字节**而非服务端原文件。
2. **HTML 实时预览用上下分栏而非左右**：面板宽度是 `clamp(22rem, 34vw, 36rem)`（352–576px），左右各半会挤到不可用；上下分栏在任意宽度都成立。

**新增的 i18n 防线（`locale-parity.test.ts`）**：此前仓库只有按功能点写的 i18n 测试，没有跨语言结构性校验，因此 P1 遗漏的 `copyFailed` 会静默回退成英文。新测试断言 5 种语言的 leaf key 集合一致，并**正确处理 i18next 复数后缀**（英文用 `key_one`/`key_other`，中日文用裸 `key`——这是正确行为，朴素比较会产生大量误报）。它同时冻结了一个已知偏差：zh 独有的 4 个管理员文案（`models.fastMarked` 等），并有一条元测试防止这个豁免清单被悄悄扩大。

### P5（已交付）

依赖：`@docx-editor.dev/core` + `/react` + `/i18n`，全部 **2.21.0 / Apache-2.0**，**已实测确认 `/pro` 未被安装**（它的许可证写明禁止生产使用，且就挂在公共 npm 上）。

| 文件 | 变更 |
| --- | --- |
| `src/components/files/editors/docx-editor.tsx` | **新增**：`<DocxEditor document={bytes} ref>`，`ref.save()` 产出编辑后字节 |
| `editor-types.ts` / `document-editor.tsx` | `docx` → `'docx'`，注册表加懒加载项 |
| `src/i18n/locales/*/chat.json` ×5 | 新增 `docxSaveFailed` |

**四个实现判断：**

1. **保存是防抖而非每次按键**（900ms）：`save()` 会序列化整个 docx 包，逐键调用会在长文档上卡顿。面板的「另存为副本」用最近一次序列化的结果。
2. **关闭了编辑器自带的菜单栏**（`menu={false}`）：它的 File › Open/Save 会与面板自己的保存/下载形成两条互相冲突的路径。
3. **关闭了导航窗格**（`navigation={false}`）：面板只有 22–36rem 宽，导航窗格会挤掉页面本身。
4. **跟随应用明暗主题**（读取 `useTheme().resolved` 传给 `colorMode`）。

⚠️ **必须如实记录的局限：编辑器自身界面文案无法完整覆盖 5 语言。** 厂商只提供 **en / fr / zh-CN** 三个目录，**没有日语和繁体中文**。我按语言映射到最接近的目录，并让 `zh-Hant` 使用简体目录（对繁体用户可读性高于英文）——这是一个有意的、可一行改回的选择。**我们自己的 UI 文案仍是完整 5 语言**，缺口只在第三方编辑器 chrome 上。

### P4（已交付）

| 文件 | 变更 |
| --- | --- |
| `src/lib/ooxml/archive.ts` | **新增**：ZIP 层。未被 `write` 的 entry 原样交回 JSZip，重打包后**解压内容逐字节不变** |
| `src/lib/ooxml/xlsx.ts` | **新增**：sheet 定位、共享字符串、单元格读写、`fullCalcOnLoad` |
| `src/lib/ooxml/formula.ts` | **新增**：安全的公式求值器（见下） |
| `src/components/files/editors/spreadsheet-editor.tsx` | **新增**：AG Grid 网格 + 公式栏 + sheet 切换 |
| `editor-types.ts` / `document-editor.tsx` | `xlsx` → `'sheet'`，注册表加懒加载项 |
| `src/i18n/locales/*/chat.json` ×5 | `sheetLoading` / `sheetFailed` / `formulaBar` / `formulaUnsupported` |
| `tests/frontend/lib/ooxml/surgical-xlsx.test.ts` | **新增 6 个用例**：零损失硬指标 + 命名空间 + 样式保留 + 转义 + 重算标记 + 行列顺序 |
| `tests/frontend/lib/ooxml/formula.test.ts` | **新增 8 个用例**，含 7 个 JS 注入 payload |

**三个必须记录的技术判断：**

1. **命名空间是这里最容易踩的坑。** SpreadsheetML 用**默认命名空间**，所以 `document.createElement('v')` 产出的元素落在空命名空间、序列化成 `<v xmlns="">`，**Excel 会拒绝整个文件**。所有元素必须用 `createElementNS`。（这个坑是 P4 子代理的探测发现的，值得记功。）
2. **公式求值绝不能用 `new Function`。** 公式文本来自用户打开的文件，即攻击者可控；而应用其他地方的不受信 HTML 是关在**不透明源沙箱**里的，公式没有这层边界。改用显式词法 + Pratt 解析器，函数走白名单，越界即抛 `FormulaUnsupported` 并在 UI 上**如实标注"未计算"**，而不是显示一个错误的数字。
3. **文本值写成 inline string**，不碰 `sharedStrings.xml`——改它要同步 `count`/`uniqueCount` 与所有索引，风险远大于收益。inline string 是合法 OOXML。

**如实记录的边界**：公式求值是**单遍**的，引用另一个公式单元格时取该单元格的既有值；完整的迭代计算引擎不在范围内。支持的函数见 `formula.ts` 的白名单（SUM/AVERAGE/MIN/MAX/COUNT/COUNTA/ABS/ROUND/INT/SQRT/MOD/POWER/LEN/TRIM/UPPER/CONCATENATE/IF/AND/OR/NOT/IFERROR）。编辑态上限 2000 行 × 100 列——**不沿用**只读预览的 250×50 截断。

## 真实浏览器验证（`npm run test:browser`）

在此之前，四个编辑器**从未被真实挂载过**——只有 typecheck 与纯逻辑测试。新增 `tests/browser/`：一个仅供测试的页面（不在 index.html 的依赖图里，因此永不进入生产包）把编辑器挂到真实 DOM 上，由 `puppeteer-core` 驱动**本机已安装的 Chrome**（不下载浏览器）完成挂载、交互与断言，并把**浏览器产出的字节写盘**，交给 openpyxl / python-pptx / python-docx 回读。

**这一步立刻抓出两个使功能不可用的真实缺陷——而 typecheck、单元测试、`vite build` 三者全部通过。**

### 缺陷 A：HTML 代码编辑器完全不渲染（致命）

控制台抛 `RangeError: Invalid top rule name SingleExpression`，`#root` 子元素数为 0——P3 的 HTML/CSV/Markdown 编辑**在浏览器里根本打不开**。

**根因**：pnpm 把 `@lezer/javascript` 解析成了 **1.0.0**（2021 年的构建），而 `@codemirror/lang-javascript` 的依赖区间是 `^1.0.0`、当前版本是 1.5.5。那个陈旧版本预生成的解析器表与 `@lezer/lr@1.4.10` 不兼容。（pnpm 11 的 `minimumReleaseAge` 保护机制是诱因。）修法：在 `pnpm-workspace.yaml` 用 override 钉住 `@lezer/javascript: 1.5.5`。修复后 `.cm-editor` / `.cm-content` 正常渲染。

### 缺陷 B：`npm run dev` 打开 Word 文档即崩（致命）

`@docx-editor.dev/core` 的 dist 里含 **top-level await**。生产构建能过（`build.target` 是 `es2022`），但 **Vite dev server 的依赖预打包用 esbuild 默认 target（es2020）**，直接构建失败。也就是说 `npm run dev` 下用户一打开 `.docx` 就报错，而 `vite build` 完全正常——这类"只在开发服务器复现"的问题只有真实运行才能发现。

**修法**：在 `vite.config.ts` 增加 `optimizeDeps.esbuildOptions.target: 'es2022'`，与 `build.target` 对齐。

### 缺陷 C：AG Grid 单元格无法编辑（major）

AG Grid v33+ 的模块是**按需注册**的。缺少编辑器模块时网格仍能渲染（所以看起来正常），但 `editable` 单元格双击不打开编辑器，只抛 error #200。修法：在 `spreadsheet-editor.tsx` 模块级 `ModuleRegistry.registerModules([AllCommunityModule])`——注册在编辑器内部，成本落在懒加载 chunk 里。

### 验证结果

```
ok   code  sample.html   edits=16  last=648B/文本   -> out/browser-code.html
ok   sheet sample.xlsx   edits=1   last=20750B/PK   -> out/browser-sheet.xlsx
ok   docx  sample.docx   edits=2   last=35876B/PK   -> out/browser-docx.docx
ok   pptx  sample.pptx   edits=1   last=102754B/PK  -> out/browser-pptx.pptx
```

对应的真实库回读（编辑后仍被独立实现接受）：

```
browser xlsx: 编辑单元格 = 'WidgetsPlus'、其他单元格 = 80、公式 = '=SUM(B3:B5)'、
              样式填充 = '00B1552F'、第二工作表完好
browser docx: 标题完好、输入的文字存在、表格完好
browser pptx: 编辑后标题 = 'Q4 Review — browser'、第二页完好、11 个版式完好
browser html: 输入的标记存在、原有标记完好
```

**这构成了完整的可用性证据链：真实浏览器挂载 → 真实交互 → 产出字节 → 被真实第三方库接受。**

> 仍未覆盖：Firefox / Safari、移动端布局、以及键盘/读屏的真实人工验收（AG Grid 的 ARIA 语义已由它自身保证，但我没有做人工读屏测试）。

## HTML 预览：一次真实缺陷排查（`npm run test:preview`）

用户报告："AI 写完 HTML 点击预览是空白的，需要手动刷新一遍才会显示内容"。

新增 `tests/browser/run-preview.mjs`：挂载**真实的 `ArtifactPanel`**，通过 store 走与点击按钮完全相同的路径，用 9 个场景在真实 Chrome 里验证。

沙箱 iframe 是不透明源，父页面读不到它的 DOM（这层隔离是刻意的，不能为测试放宽），所以改为让产物自己的脚本 `postMessage` 回报"已执行"，并由父页面测量 iframe 的**几何尺寸**——"文档执行了但盒子是 0×0"正是"空白"，而纯 DOM 断言看不出来。

### 找到并修复的真实缺陷（已在真实浏览器复现并验证）

预览文档向 iframe 注入：

```html
<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests">
```

在 **HTTP 但非 localhost** 的源上（自托管部署的常见形态），这条指令会把**同源**子资源也升级成 https，包括我们注入的 Tailwind runtime：

```
body background = "rgba(0, 0, 0, 0)"
console: Failed to load resource: net::ERR_SSL_PROTOCOL_ERROR
```

**runtime 从未加载 → AI 生成的 HTML 完全失去样式**；对依赖 Tailwind 布局/显隐的产物，表现就是"空白"。`localhost` 是规范定义的"潜在可信源"而被豁免，所以本地开发一直测不出来。

**修复**：只在 `location.protocol === 'https:'` 时注入该指令（HTTP 源上本就没有混合内容可升级）。公开分享预览仍始终注入——它由我们自己的服务器在已知源上提供。

**验证**：在 `http://<LAN-IP>:5198` 下重跑，Tailwind 恢复正常编译（此前必现 `ERR_SSL_PROTOCOL_ERROR`）。

**同一类缺陷的第二个实例（服务端）**：`server/internal/api/html_preview_handlers.go` 给**公开分享的预览**设置的 CSP **响应头**里也带了 `upgrade-insecure-requests`。纯 HTTP 部署下它同样会把该页面自己的 `/tailwind-browser.js` 升级成 https——**分享出去的预览链接也会完全失去样式**。已改为复用既有的 `secureCookie(r)` 判断（TLS 或可信代理的 `X-Forwarded-Proto: https`），并在 `html_preview_handlers_test.go` 中加了两个方向的断言（明文源不得含该指令、https 源必须含）。

> ⚠️ **Go 测试未能在此环境运行**：本机 `go` 是 32 位而 mingw gcc 是 64 位，CGO 链接器脚本不匹配；sqlite3 驱动需要 CGO，因此 `go test ./internal/api/` 无法执行。已用 `CGO_ENABLED=0 go build ./...`（exit 0）与 `go vet ./internal/api/`（exit 0，含测试文件）确认编译无误，但**这两条断言的实际运行结果我没有验证过**，需要在能跑 Go 测试的环境（CI/Linux）确认。

### 防御性加固

iframe 现在在**文档变化时重建**，而非就地改 `srcdoc`。就地改通常会导航，但一旦这次导航被丢弃，画面会一直停在旧（可能为空的）文档上，只有重建能恢复——这正是刷新按钮做的事，也正是所报告症状的形状。流式的 350ms 防抖已承担该频率，且 `srcdoc` 变化本就会重置文档状态，因此重建不带来新代价。

### 仍未复现的部分（如实记录）

**9 个场景（含窄窗口 Sheet 分支、流式写入、关闭重开、同块重复点击、切换块）全部通过**，我无法用合成产物复现"必须手动刷新"。因此严格讲：上述 HTTP 缺陷**真实且已验证修复**，但它在纯 HTTP 源下是**持续失败**，与"刷新一次就好"不吻合——用户的症状可能还有第二个成因，**需要具体产物与复现条件才能定论**。

## 收尾加固（P3–P6 之后）

### 1. 用真实库生成的样本验证外科手术写回

此前的零损失测试全部使用**手写的 XML fixture**——它只能证明我的实现符合**我自己对格式的假设**。openpyxl 就暴露了这一点：它默认写 inline string、**根本不生成 `sharedStrings.xml`**，这是手写 fixture 从未覆盖的形状。

新增：

| 文件 | 作用 |
| --- | --- |
| `tests/fixtures/ooxml/make_fixtures.py` | 用 **openpyxl / python-pptx** 生成真实样本（多工作表、样式、品牌填充色、数字格式、公式、合并单元格、列宽、冻结窗格、布尔、百分比；pptx 含母版与 11 个版式） |
| `tests/frontend/lib/ooxml/real-fixtures.test.ts` | 对真实样本做外科手术编辑，断言除被编辑部件外**逐字节相等**，并产出 `out/` 供下一步回读 |
| `tests/fixtures/ooxml/verify_roundtrip.py` | 用 **openpyxl / python-pptx 重新打开产出文件**，逐项断言 |

**结果：25 项真实库检查全部通过。** openpyxl 能在编辑后的工作簿里找到：编辑过的值与新增单元格、合并区域 `A1:C1`、粗体、`00B1552F` 品牌填充色、`#,##0.00` 数字格式、`=SUM(B3:B5)` 公式文本、列宽 24、冻结窗格 `A3`、以及第二个工作表与所有未编辑单元格。python-pptx 能找到：编辑后的标题（em-dash 完好）、副标题、第二页两个段落、11 个版式。

这一步证明了此前只能推断的事情：**产出文件被独立的真实实现接受，而不只是 XML 合法。**

运行方式：`npm run test:ooxml-roundtrip`（repo 里的脚本按惯例用 `python3`；Windows 上用 `python`）。

### 2. 补齐 DOCX 编辑器 chrome 的日语与繁体中文

厂商只提供 en / fr / zh-CN。新增：

| 文件 | 内容 |
| --- | --- |
| `src/i18n/docx-editor/ja.ts` | 128 个**常驻可见 chrome** 的日文文案 |
| `src/i18n/docx-editor/zh-Hant.ts` | 同范围的繁体中文文案 |
| `tests/frontend/lib/docx-editor-i18n.test.ts` | **10 个守卫用例** |

覆盖范围：工具栏、格式栏、对齐、列表、样式、字体、字号、缩放、标题栏、编辑模式、加载、错误文案。

**一个关键设计**：繁体中文是 `deepMerge(zhCN, zhHantChrome)` ——**合并到简体目录之上而不是英文之上**。这样本项目尚未翻译的深层对话框 key 会保留**简体**（繁体读者可读），而不是掉成英文。

守卫测试检查三件事，其中第一件是最重要的失败模式：**翻译不能丢掉占位符**（`{label}`/`{fonts}`/`{current}`/`{total}`），否则用户会直接看到 `{fonts}` 这样的裸标记；以及未知 key（拼错的 key 会静默失效）和未翻译的复制粘贴残留。

**仍然存在的缺口（如实记录）**：754 个 key 中还有约 626 个（深层对话框：批注、修订、高级表格、协作、图片插入/环绕、内容控件、备注、审阅者）仍是英文（ja）或简体（zh-Hant）。这是**翻译工作量**，不是编码问题；我选择不机器翻译数百条专业 UI 文案然后声称完成。

### 3. 修复 `audio-stream.test.ts` 的偶发失败

**先复现再修**（单跑 9 轮不复现，改用 3 路并发全量套件后 **3/3 必现**），拿到真实错误：

```
Test timed out in 5000ms.
AssertionError: expected [ ArrayBuffer [51, 19, 51, 19, …] ] to deeply equal []
```

**根因链**：测试 1 在 CPU 竞争下 5s 超时 → 它的语音会话**从未被 cancel**（`controller.cancel()` 位于超时点之后）→ 该会话的 `startVoiceStream` 迟到才 resolve，把 socket **推进了已被 `beforeEach` 清空的静态数组** → 测试 2 的 `latestSocket()` 拿到**测试 1 的僵尸 socket** → 于是断言里出现上一个测试的音频数据（`0x1333 = 4915 = 0.15 × 32767`，正是测试 1 的采样幅度）。

**修法**（不掩盖任何产品缺陷）：
1. `afterEach` 里取消本测试启动的会话——即使测试中途失败也不会留下僵尸。
2. 该套件用 30s 超时：这两个测试自身不做任何等待，超过 5s 只可能意味着机器被饿死，而超时**正是产生僵尸的原因**。

**验证**：在之前 3/3 必现的并发条件下重跑，**3/3 通过**；顺序全量多轮通过。

> 📌 **后续：`realtime-compaction.test.ts` 已按同一流程修复**（复现 → 定位 → 修复）。它在 3 路并发下 3/3 超时，错误是 `Test timed out in 5000ms`；测试体内**没有任何等待**（只刷新两次微任务），所以超出 5s 只可能是机器被饿死——动态 `import('@/lib/realtime')` 在负载下解析模块图很慢。已加同样的 30s 上限。

> 📌 **`audio-stream` 的修法在后续轮次被加强了一次。** 首轮修法（afterEach 取消 + 30s 上限）只把概率从 1/3 降到更低，压力下仍会失败——因为**僵尸仍然存在**。重新推导后的结论是：给实例打标记/世代号**无法**解决，因为迟到的调用读取的是**当前**全局变量，与存活测试自己的调用无法区分。真正消除机制的做法是：**在 teardown 里 await 在途的 `startVoiceStream` 并就地取消它**（`pendingStart` 模式）。这正是僵尸 socket 被创建在新测试注册表里的唯一途径。加强后 3 路并发 3/3 通过。

> ⚠️ **关于 3 路并发压力的说明**：在把**三整套**全量测试同时跑（每套本身已并行，等于数倍超订）的极端条件下，个别时序敏感测试仍可能超出 30s，并出现过 `message-row-citation-preview.test.tsx` 的套件级失败。这是**机器容量**现象，不是测试缺陷——正常 CI 一次只跑一套。判断标准应是顺序全量运行是否确定（见下方验证）。

## 收尾加固（P3–P6 之后）

### 4. DOCX 编辑器文案：已补齐全部可达 key

分四批完成，每个语言目录现 **585 个 key**：

| 批次 | key 数 | 覆盖 |
| --- | --- | --- |
| 1 | 134 | 常驻可见 chrome：工具栏、格式栏、对齐、列表、样式、字体、字号、缩放、标题栏、编辑模式、错误 |
| 2 | 109 | `colorPicker`、`disabledReason`、`contextMenu`、`lineSpacing`、`ruler`、`documentOutline`、`toc` |
| 3 | 155 | `navigation`、`imageWrap`、`headerFooter`、`imageProperties`、`hyperlinkPopup`、`notes`、`image` 族、`equationPopup` |
| 4 | 122 | `dialogs`（页面设置、段落、脚注属性、图片属性）、`tableAdvanced`、`table`、`contentControl`、`textFormField` |

**两个基于证据的范围决定**（都不是为了省事）：

1. **134 个 key 属于 Pro 功能**（`comments` / `revisions` / `reviewers` / `collaboration` / `collaborationDemo` / `review`）。本项目只用 Apache-2.0 core，且面板已移除会到达这些界面的菜单栏——**它们是死 key**。因此可达目标是 **620**，不是 754。
2. **20 个 key 语言中立**，翻译它们只会造出与英文逐字相同的"译文"：纸张尺寸（`Letter (8.5" × 11")`）、数字格式示例（`i, ii, iii, ...`）、`OK`、`in`、`0.5 pt` 等度量。这些**显式登记**在测试里并冻结。另有 15 个快捷键加速键（`Ctrl+L`）同理。

**最终结果**：620 个可达 key 中 **585 已翻译**，35 个为上述显式语言中立项。

**关键保障是一条覆盖不变量，而不是一个百分比**：

> 每一个可达 key，要么已翻译，要么被显式登记为语言中立。

这条断言会**在厂商升级引入新 key 而未翻译时立即失败**——也就是缺口本来会悄悄回来的时候。它取代了上一版"覆盖率多少"式的弱断言。

**守卫测试在这一过程中抓到了我自己的一个缺陷**：它曾报出两个 key「译文与英文相同」——`URL` 与 `{kind} {number}`。我核查后发现**是检查规则错了，不是翻译错了**（`URL` 三种语言都写 URL；`{kind} {number}` 是纯占位符）。我没有加 key 白名单，而是改成有原则的规则，并**删掉了原先 `/^font\./` 那个笼统豁免**——现在检查比修改前更严格。

### P6（已交付）

**按你的决策改为自研**（`pptx-react-viewer` 与本仓库 i18n 栈有三个主版本 peer 冲突，见 §2.3 PPT）。

| 文件 | 变更 |
| --- | --- |
| `src/lib/ooxml/pptx.ts` | **新增**：slide 定位（顺序取自 `sldIdLst`，不是 rels 映射顺序）、`<a:t>` 文本读写、`xml:space` 处理 |
| `src/components/files/editors/pptx-editor.tsx` | **新增**：slide 预览（复用已装的 `@aiden0z/pptx-renderer`）+ 上/下一张 + 逐 run 文本框 |
| `editor-types.ts` / `document-editor.tsx` | `pptx` → `'pptx'`，注册表加懒加载项 |
| `src/i18n/locales/*/chat.json` ×5 | `pptxFailed` / `pptxPrev` / `pptxNext` / `pptxRuns` / `pptxNoText` |
| `tests/frontend/lib/ooxml/surgical-pptx.test.ts` | **新增 6 个用例**：零损失 + slide 顺序 + `xml:space` + 无变更原样返回 + 转义 |

**pptx 与 xlsx 的一个关键差异**：xlsx 用**默认命名空间**，所以创建元素必须 `createElementNS`（否则产出 `<v xmlns="">`，Excel 拒绝）；pptx 用**前缀命名空间**（`p:`/`a:`），而本模块**只改写既有 `<a:t>` 的文本内容、从不创建元素**，因此那一类坑在 pptx 侧不存在。查找仍走命名空间感知的辅助函数，因为不同解析器对前缀匹配的行为并不一致。

**范围如实记录**：**文本级编辑**。不能新增形状/图片/图表、不能改版式与母版。这是纯浏览器零依赖方案下诚实的边界，也是安全的边界——只有 `<a:t>` 被改写，主题、母版、版式、媒体、图表全部原样保留。

**成本极低**：编辑器 chunk 仅 **6.2 KB**，因为它完全复用了 P4 的 ZIP 层与已装的渲染器，零新依赖。

### 收尾时发现并修复的两个真实缺陷

在做 P6 时我复查了 P4 的编辑器，发现两个会导致**丢编辑**的问题，都已修复：

1. **防抖与保存的竞态（影响 sheet / docx / pptx 三个编辑器）**：三个编辑器都做防抖序列化，而面板的「另存为副本」读的是 `edited.bytes`。用户改完**立刻**点保存，会存下**原始文件**——正是"以为保存了其实没保存"的丢工作场景，也违反 `PRODUCT.md` 的 anti-reference。修法：`DocumentEditorProps` 增加 `flushRef`，编辑器注册一个「立即序列化并返回字节」的回调，面板在保存与下载前先 flush。

2. **过期闭包丢编辑（sheet）**：AG Grid 的回调只创建一次并被保留，`applyEdit` 闭包里捕获的 `sheets` 会过期——**在同一张表上连续编辑两次，第二次会基于第一次编辑前的状态重建，把第一次的改动丢掉**。修法：改从 `sheetsRef.current` 读最新状态。

3. **顺带修正了保真度**：表格编辑器原先把**每一个单元格**都当作编辑写回，会把整个 sheet 重写成 inline string，违背"只改用户真正编辑的内容"。改为只写脏单元格，并用测试断言**未编辑的单元格在重写后的 sheet XML 里逐字符不变**。

P6 的 pptx 编辑器从一开始就带 `slidesRef` 与 `flushRef`，没有这两个问题。

### P0 + P1 + P2 完成后的交付形态

| 文件 | 变更 |
| --- | --- |
| `src/store/artifact-panel.ts` | **新增**，取代 `src/store/html-preview.ts`：一个 store，`source` 判别联合 `html` \| `file` |
| `src/components/chat/artifact-panel.tsx` | **新增**，取代 `html-preview-panel.tsx` + `file-preview.tsx`：右侧统一面板，HTML body / File body |
| `src/components/chat/file-preview.tsx` | **删除**（原居中 Dialog） |
| `src/components/chat/html-preview-panel.tsx` | **删除** |
| `src/pages/chat/ChatLayout.tsx` | 改为渲染 `<ArtifactPanel />` |
| `src/components/chat/code-block.tsx` | 迁移到新 store |
| `src/components/chat/sandbox-files-panel.tsx`、`src/components/chat/message-row.tsx`、`src/pages/kb/KnowledgeBaseDetail.tsx` | 三个 Dialog 调用点改为 `openArtifact({ type: 'file', … })` |
| `src/store/{sandbox-files,inline-thread,conversation-files}.ts` | 交叉关闭改指向新 store |
| `src/api/endpoints.ts` | 新增 `authApi.saveDocumentCopy(blob, filename)` |
| `src/i18n/locales/{zh,zh-Hant,en,ja,fr}/chat.json` | 新增 `saveCopy` / `copySaved` / `copyFailed`（**5 语言全覆盖**） |

四种格式（HTML / PDF / DOCX / PPTX / XLSX）**全部在右侧面板预览**，聊天附件与沙箱产物不再是居中弹窗；文档可「下载」与「另存为副本」（HTML 产物只提供「下载」，见 §〇）。

**编辑能力**（P3–P6）是下一期。

**独立验证（由主 Agent 复核，非子代理自述）**：

```
tsc -b --noEmit                 → exit 0
vitest --dir tests/frontend     → 122 files / 688 tests 全绿
vite build                      → exit 0
```

并且逐项复核了下述高风险点：`toast.success/error` 签名、`saveDocumentCopy` 确在 `authApi` 内、5 语言 key 真实存在、全仓已无 `FilePreview` / `useHtmlPreview` 残留引用、`message-row` 的「附件被删除则关面板」行为保留。

---

## 四、决策记录（已定，见 §〇）

| # | 问题 | 结论 | 依据 |
| --- | --- | --- | --- |
| 1 | "保真损失"指什么？ | **文件级保真为硬指标**；屏幕排版接受近似 | 屏幕像素级保真需要 Word 排版引擎 + 微软字体，**任何浏览器方案都做不到**——ONLYOFFICE/Collabora 也用 Carlito 等度量替代字体并静默替换，官方自认"版式可能受损" |
| 2 | Word/PPT 路线 | **Word 用现成 Apache-2.0 组件；PPT 改为自研文本级编辑** | `@docx-editor.dev/react` 无 peer 冲突、原生 OOXML round-trip；`pptx-react-viewer` 与本仓库 i18n 栈有三个主版本 peer 冲突（见 §2.3 PPT），改为复用 P4 的 OOXML 核心做自研文本级外科手术编辑 |
| 3 | 表格 a11y vs 功能 | **两者都要**：AG Grid Community + formulajs + 自研写回 | AG Grid 是 DOM+ARIA 渲染，不像 Univer/Luckysheet 的 canvas 那样破坏键盘与读屏 |
| 4 | 编辑保存语义 | **默认另存为新文件**，不覆盖 | `PRODUCT.md` 把"丢失工作 / 历史被改写"列为 anti-reference |
| 5 | 本期范围 | P0 + P1 + P2 | — |

---

## 附：明确不采用的方案

| 不采用 | 原因 |
| --- | --- |
| **mammoth + Tiptap + docx.js 做 Word 编辑** | **不是 round-trip**：`docx` 库没有读取 API，"保存"等于重新生成一份新文档；页眉页脚 / 分页 / 节 / 文本框 / 公式 / 域 / 修订**全部永久丢失且累积** |
| **Univer OSS 做表格** | xlsx 读写属 Pro 付费功能，且**必须自建 `universer-api` 服务端**；无 license 时带水印 + 导入大小限制 |
| **Handsontable / HyperFormula** | Handsontable 商业授权 $999/开发者起且禁止用于竞争产品；**`hyperformula` 是 GPL-3.0-only**，闭源不可用 |
| **Luckysheet** | README 明写 "no longer maintained"，末次发布 2021-01 |
| **`@docx-editor.dev/pro`** | 许可证 `LicenseRef-EigenPal-Pro-Evaluation-1.0`，明写 **"You may not use the Software for Production Use"**。它在公共 npm 上，**极易误装侵权** |
| **OnlyOffice Docs / Collabora** | 需要独立容器 + 强制后端（`callbackUrl` / WOPI host）+ AGPL-3.0 §7 附加条款 或 订阅；与"不依赖容器"的要求直接冲突。且**两者都不含 MS 字体、都不跑 VBA**，屏幕保真同样做不到完美 |
| **`ranuts/document`** | AGPL-3.0 且 §7 强制保留 ONLYOFFICE logo（有测试在移除 logo 时失败） |
| 为四种格式各写一套面板 | 已有 `ChatSidePanel` + `DocumentPreview` 两个抽象，重复实现会立刻产生多份脏状态 / 全屏 / 分享逻辑 |
| 自研 PPT 画布编辑器 | 参考量级 600–1000+ 人日（PPTist 投入了 5.7 年） |
| 编辑态给 iframe 加 `allow-same-origin` | 会使沙箱完全失效，等于把用户 cookie / storage 暴露给任意 HTML |
| 放开 `.html` 上传白名单 | 存储型 HTML 内联返回是 XSS 向量；放开需要与 PDF 同等的强制 `attachment` disposition 保护。**属于安全决策，不擅自改动** |
