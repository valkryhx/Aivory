# Aivory 后端调研 04 —— RAG 知识库系统 · 向量存储 · 异步队列

> 调研对象:Go 后端 `server/internal/rag/` `server/internal/vector/` `server/internal/queue/`
> 目标:为「把 Go 后端换成 Python/FastAPI」的复刻工程提供可执行的源码级参考。
> 所有论断都给出 `server/internal/...:行号` 出处。本文按「摄取 → 检索 → 注入 → 引用 → 复刻」组织。

---

## 0. 一句话架构

Aivory 的 RAG 是一个**「解析 → 分块 → 嵌入 → 写库 → 异步检索 → 注入上下文 → 前端引用」**的完整链路,关键设计点:

- **文档解析本地优先、MinerU 云端兜底**(`parser.go`,延迟优先策略);Office/PDF 只要有可用文本层就本地即时解析,只有扫描件/无文本 PDF 才走付费云端 OCR。
- **small-to-big 层次分块**:父块(带标题面包屑、不嵌入)+ 子块(嵌入向量),检索命中子块后把**子块邻接窗口(或父块)**注入上下文,保证"搜小块、回大块"(`rag.go:2436`)。
- **异步摄取线程/队列双轨**:生产用 asynq/Redis 持久任务,开发用进程内队列兜底;二者之上统一用**DB 心跳 + 租约 + 崩溃恢复**保证不丢任务(`rag.go:226`、`320`)。
- **向量后端可插拔**:`VectorStore` 接口,生产 Qdrant(按维度一 collection),个人版内嵌 SQLite 精确余弦搜索,或禁用后走全文本注入(`vector/vector.go:62`)。
- **混合检索**:向量稠密腿 + 关键词稀疏腿 双路独立拉取,`fuseReciprocalRank` RRF 融合,再走 reranker(可选)、相似度阈值截断、邻接窗口扩展(`rag.go:1410`、`1992`)。
- **LLM 路由**:`RouteAndRetrieve` 先看 token 预算,小文档直接全文注入;否则交给 TaskLLM 决策 `none | retrieve | full_doc`,并提供多轮证据扩展 `RouteAndRetrieveIterative`(`rag.go:2929`、`3218`)。

---

## 1. RAG 架构流程(完整链路)

```
用户上传文档
   │
   ▼
HTTP API(not in scope)── 写 documents 行(status=pending)
   │  rag.Ingest(docID)                                rag.go:364
   ▼
enqueueIngest ── asynq(Redis,有租约/唯一锁) ── 或 ── 进程内队列
   │                                                                     rag.go:375
   ▼
runIngestWithRetries ── startIngestHeartbeat(DB 心跳)                    rag.go:481
   │         (3 次整体重试, 缓存 parse 避免重复付费 OCR)
   ▼
runPipeline                                                              rag.go:625
   ① 清理旧 chunk/向量(幂等)                    DeleteChunksByDocument + DeleteByDocument
   ② status → parsing
   ③ 解析 parseDocument(本地文本 / 本地 OfficeXML / 本地PDF文本层 / MinerU 云端OCR)   parser.go:96
   ④ status → embedding
   ⑤ 层次分块 chunkHierarchical(parent+children)                        rag.go:2436
   ⑥ 一次性批量嵌入 em.Embed(allChildren)                               rag.go:890
   ⑦ 事务写 chunks(parent+child) + upsert 向量                           rag.go:969 / 983
   ⑧ status → ready(chunk_count)
   ▼
检索期(对话时刻)
   RouteAndRetrieve / Retrieve
     ├─ token 预算 ≤ FullTextThreshold(8000) → 全文注入 fullTextSnippets
     ├─ 否则 TaskLLM 路由 none|retrieve|full_doc                           rag.go:2955
     │    retrieve → 每条改写 query 并行 Retrieve → RRF 融合
     ├─ 向量稠密腿(Search) + 关键词腿(SearchKeyword) + 关系库词法腿      rag.go:1696
     ├─ fuseReciprocalRank 融合                                          rag.go:1992
     ├─ 相似度阈值/固定 TopK 截断                                        rag.go:1421
     ├─ reranker 重排(可选)                                             reranker.go
     └─ child window(±1 邻接子块)/父块扩展 → Snippet 列表
   ▼
注入对话上下文(编排器拼 prompt, not in scope)
   ▼
前端渲染引用:每 Snippet 携带 ID/Index/Title/URL/Snippet/Source
   URL 约定:知识库 → kbdoc://<document_id>;对话 → doc://<document_id>      rag.go:1515
   Source 约定:知识库 → "kb";对话上传 → "document"                        rag.go:1503
```

---

## 2. 摄取流水线逐步解析

### 2.1 入口与幂等清理

`runPipeline`(`rag.go:625`)是整个摄取的主角:

```go
// rag.go:649
// Idempotent re-ingest: drop any chunks AND vectors from a previous… run BEFORE doing anything else
if err := store.DeleteChunksByDocument(ctx, s.db, docID); err != nil { return err }
if s.vec.Enabled() {
    if derr := s.vec.DeleteByDocument(ctx, docID); derr != nil { … }
}
if err := store.UpdateDocumentStatus(ctx, s.db, docID, "parsing", "", 0); … // rag.go:651-663
```

关键语义:**先清空再重写**,使重复摄取(崩溃重入、`RequeueIncomplete`、手动 Retry)不会产生重复 chunk/向量。DB chunk 行是真相,向量库只是搜索索引——因此即便 DB chunk 已清但 Qdrant 未删,也照删。

### 2.2 文档解析(文本提取决策)

`parseDocument`(`parser.go:96`)的决策规则:

- **文本类**(mime `text/*`、json/xml/csv/markdown)→ 本地 `os.ReadFile` 直接返回(`parser.go:107`)。
- **Office(docx/pptx)** → `extractOfficeXML` 读 zip 解 XML 文本;**只有提取不到有效文本(纯截图 pptx)才走 MinerU OCR**(`parser.go:164-177`)。docx 用 `word/document.xml`,pptx 用 `ppt/slides/slide*.xml`(`parser.go:275`)。
- **PDF** → 先 `inspectPDFTextLayerBounded` 快速探测文本层(`parser.go:180`),判定 `scanned`(扫描件)/`thin`(文本稀疏但含图)。**只有扫描/稀疏文本 PDF 才走 MinerU**;正常数字 PDF 走 `extractFullPDFText`(`pdf` 库,`parser.go:609`)。
- **其它(图片、legacy .doc/.ppt、html)** → 直接 MinerU。
- **无文本可提且 MinerU 未配置/失败** → 返回带 `document_parser_not_configured` 的占位文本 + `extracted=false`,`runPipeline` 据此把文档标为 `failed` **而不是** 塞一个垃圾 chunk(`parser.go:78`、`rag.go:749`)。

> 中文表格/代码块(`internal/rag/spreadsheet.go`)是独立通道:KB 下的 CSV/TSV/XLSX/XLSM 走 `SpreadsheetIndexText`;对话下的电子表格跳过摄取,留给代码沙箱 python_execute(`rag.go:686-700`)。

### 2.3 MinerU(数学/复杂 PDF/扫描件解析对接)

MinerU 云 API 对接在 `parser.go:653` `minerUExtractViaCloud`,四步:

1. **上传源文件到对象存储**:`PutFileDirect` 直接 S3/OSS 上传,key=`mineru/<short-id>/<safe-name>`,TTL 1 小时(`parser.go:677`、`mineruSourceTTLSeconds=3600`);上传完必删(即使轮询失败也要清理,`parser.go:693`)。
2. **提交任务** `POST /api/v4/extract/task`(`parser.go:736`):

```go
payload := map[string]any{
    "url":            fileURL,          // 预签名 S3/OSS 可访问 URL
    "model_version":  "pipeline",       // 项目要求固定 pipeline
    "is_ocr":         true,             // 项目要求固定 true,保证扫描 PDF
    "enable_formula": true,             // 数学公式(LaTeX)
    "enable_table":   true,             // 表格结构化
}
```

3. **轮询** `GET /api/v4/extract/task/{task_id}`,5s 间隔,上限 20 分钟;状态 `done|failed|pending|running|converting|waiting-file`(`parser.go:791`)。
4. **下载 zip 并解包**:取 `full.md` + `images/`,`![alt](images/foo.png)` 重写为 `![alt](mineru://foo.png)` 中间标记;随后 `stripMinerUMarkdownImages` 在进入分块/嵌入前剥离(**避免把 `mineru://` 文件名字符串污染向量和关键词**)(`parser.go:873`、`999`)。

安全要点:zip URL 来自 API 响应,故用 SSRF-safe 客户端(`netsafe.PrivateBlockClient`,`parser.go:868`),且只接受 http(s) scheme、zip 上限 500MiB(`parser.go:892`)。

特殊 chunk:`soleMineruImageMarker`(`rag.go:2755`)——一个刚好等于单个 `![caption](mineru://fname)` 的子块被分类为 `chunk_type="image_caption"` 并记录 `ImageRef`,混合散文的仍是 `text`。

### 2.4 分块(Chunking,small-to-big)

`chunkHierarchical`(`rag.go:2436`)+ 常量(`rag.go:2424`):

```go
childTargetChars  = 2000   // 子块目标字符(≈400-800 token)
parentTargetChars = 4800   // 父块(章节)目标字符
chunkOverlapChars = 250    // 相邻子块重叠(~12%)
```

层级拆解:

1. **按标题切章** `splitByHeadings`(`rag.go:2479`):匹配 ATX `#…######` 标题,维护标题深度栈,生成 `breadcrumb`(如 `"Manual > Setup > Networking"`)。
2. **保护原子单位** `splitProtectedAtoms`(`rag.go:2544`)+ `splitParagraphsAndTables`(`rag.go:2572`):围栏代码块(```` ``` ````)与整段管道表格 `|…|` 视为 **atomic**,绝不跨两个 chunk 拆分;数学块 `$$…$$` 和单图块同样保持整体。这正是"保护表格/代码"。
3. **合并成子块** `mergeAtomsIntoChildren`(`rag.go:2607`):累积 atom 到 ~target 字符;超长段落按句号(CJK+ASCII 句点)切句 `splitSentences`(`rag.go:2709`,支持 `。！？；. ! ? ;`)。
4. **重叠** `withOverlap`(`rag.go:2678`):每个子块(除首个)带着前一块的尾部(在词/标点边界切断,且不进代码/表格块)。
5. **面包屑前缀**:`"[" + breadcrumb + "]\n" + child`,让子块的语义(标题路径)进向量(`rag.go:2449`)。
6. **父块** = 截断到 `parentTargetChars` 的节正文,`ChunkType="text"`(`rag.go:2458`)。

嵌入前还有一个**超限子块再分割** `limitEmbeddingChildren`(`chunk_embedding.go:16`):受嵌入模型单输入 token 上限约束,把超大子块(OCR 巨段)按行/句边界的字节上限切分,并保留面包屑前缀,目的是"一输入一向量"契约不被破坏。

### 2.5 嵌入(Embedding,合法 provider 调用)

`runPipeline` 一次性把所有子块 `em.Embed(ctx, allChildren)`(内部按 ≤10 文本/请求 并发批次,`embedder_http.go:204`):

- **provider 解析优先级**(`resolveEmbedder`,`embedder_http.go:627`):
  1. 管理后台配置的 embedding 模型(`settings.embedding_model_id` → model+channel);
  2. `EMBEDDING_*` 环境变量;
  3. 内置本地 hash-bag embedder(256 维,始终可用,`featureVector` 把 token 哈希落到维度桶再 L2 归一,`rag.go:2382`)。
- **知识库嵌入锁定** `resolveEmbedderForKB`(`embedder_http.go:661`):KB 创建时锁定 `embedding_model_id`,`global` 设置变更**绝不**改写既有 KB 向量(否则同一 collection 维度错乱,检索静默退化)。
- **双协议**:OpenAI `/v1/embeddings` 与 DashScope 原生 `text-embedding`(`embedder_http.go:144`、`354`);带 `dimensions` 参数请求,若被 provider 拒绝(400)则**去掉 dim 重试**。
- **重试/背压**:`postEmbeddings` 指数退避+抖动,重试 TLS 超时/断连/429/5xx,4xx 直接失败;DashScope 有进程级并发槽上限(2,`embedder_http.go:183`)。
- **维度校准**:嵌入返回的实际维度 ≠ 配置维度时,自动修正 collection 维度并写回 `knowledge_bases.embedding_dim`(`rag.go:901`)。
- **用量计费**:`logEmbeddingUsage` 记录 embedding 花费(§8.3),本地 embedder 不记账(`embedder_http.go:22`)。
- **query 嵌入缓存**:进程内 10 分钟/4096 条 TTL 缓存,`embedQueriesCached`(`query_embeddings.go:56`)。

### 2.6 写入元数据与引用(chunk 行 + Payload)

子块/父块以预生成 ID 在**单事务批量写入** `CreateChunksBatch`(`rag.go:918-977`)——一次 commit 代替逐个 INSERT(旧版每 chunk 一次 fsync 是索引大文件的瓶颈)。写入维度由 `resolveEmbedder` 决定,存到 `chunks.embedding_model` 与 `chunks.embedding_dim`。

向量 payload(`vector/vector.go:17`)把"渲染引用所需全部元数据"内联进向量,**检索命中无需二次回库**:

```go
type Payload struct {
    ChunkID        string // chunk id
    DocumentID     string
    KBID           string
    ConversationID string
    ParentID       string
    ChunkType      string // text | image_caption | parent
    Seq            int    // 文档内顺序
    Content        string
    Filename       string
}
```

> **引用机制**:子块命中 → 用 `Seq`/`ParentID` 在关系库找到同父的邻接子块(`childWindow`,±1,`rag.go:1551`)拼成大片段注入;前端用 `Snippet.URL`(见 §5)渲染来源卡片。

### 2.7 摄取状态机与完整代码摘录

`documents.status` 流转:`pending → parsing → embedding → ready | failed`,
外加 `Failed` 时写 `error` 原因。核心循环(`rag.go:625-1004`)已在上文全程摘录,这里再强调两个门:

- **small-to-big 注入门**(`rag.go:791`):对话文档若整体 token ≤ `FullTextThreshold` 且是行门控文本(代码/配置)≤ 2000 行,则 `skipEmbed=true`,整个文档以文本直接注入、不向量化。
- **KB 文档总是嵌入**(`rag.go:842` 之后),因为 KB 是跨文档搜索索引。

---

## 3. 异步任务机制(队列 · worker · 恢复)

### 3.1 双轨队列抽象

`internal/queue/queue.go` 定义非常薄的 `Queue` 接口(`Enqueue(name, job)` / `Close`),注释明确:接口形态**刻意让未来 asynq/Kafka 实现无业务层改动地替换**(`queue.go:1-4`)。

- **进程内实现 `InProcess`**(`queue.go:31`):固定 worker 池(默认 8,`AIVORY_QUEUE_IN_PROCESS_WORKERS`)、有界缓冲(256)、`runJob` 带 **panic recovery**(PDF reader 解析恶意文件会 panic,必须 recover 保 worker 存活,`queue.go:92`)、单 job 30 分钟超时上限(必须覆盖 MinerU 20 分钟轮询,`queue.go:110`)。
- **生产 RAG 用 asynq/Redis**:`UseAsynq`(`rag.go:226`)起**两条 lane**:

```go
ragFastQueueName = "rag-fast"   // 文本/表格快件, concurrency=4
ragSlowQueueName = "rag"        // PDF/扫描件慢件,   concurrency=4
```

`ingestQueueNameForDocument`(`rag.go:430`):文本/表格走快 lane,二进制扫描件走慢 lane——**避免一个大 PDF 的 OCR 阻塞轻量文本作业**。

`enqueueIngest`(`rag.go:375`):asynq 可用则投 `asynq.NewTask("rag.ingest", {"doc_id"})`,带 `Queue`、`Timeout(75m)`、`MaxRetry(1 仅租约丢失)`、`Unique(80m TTL)` 唯一锁;asynq 失败即**降级到进程内队列**兜底。

### 3.2 worker 消费 + 心跳租约

- **asynq handler** `handleAsynqIngest`(`rag.go:437`):解 `doc_id` → `runIngestWithRetries`。业务/上游失败已在其内部处理并归档,故 handler 出错只包 `asynq.SkipRetry`(保留 asynq 重试给**租约丢失**,不把"3 次管线重试"再叠一层,`rag.go:453`)。
- **DB 心跳** `startIngestHeartbeat`(`rag.go:525`):每 30 秒 `TouchDocumentIngest` 刷新 `documents.ingest_updated_at`(`store/kbs.go:1283`),**只在非终态**写,避免迟到心跳让 ready/failed 复活。
- **向量写租约** `upsertDocumentVectors`(`rag.go:17`)+ `BeginDocumentVectorWrite`(`store/resource_lifecycle.go:67`):写向量前先 `UPDATE documents SET id=id WHERE id=?` 锁行 + 插 `document_vector_writes(write_id, document_id, expires_at)`;事务在网络 I/O 前结束,慢 Qdrant 不会占住 SQLite 写锁。删除侧以租约未过期作为"清理尚未安全"信号。

### 3.3 崩溃恢复(RequeueIncomplete / RunIngestRecovery / RequeueStaleIngests)

- **启动恢复** `RequeueIncomplete`(`rag.go:294`):`ListIncompleteDocuments` 取 `pending/parsing/embedding` 态文档(进程内队列不跨重启,否则会永久"indexing…"),逐个刷新心跳并 `Ingest` 重排。
- **持续看门狗** `RunIngestRecovery`(`rag.go:320`):启动先 `SweepDeletedDocumentVectors` + `RequeueIncomplete`,再每 1 分钟:
  - `SweepDeletedDocumentVectors` 处理遗留的已删文档清理;`RequeueStaleIngests` 认领心跳停滞的任务。
  `ClaimStaleIncompleteDocuments`(`store/kbs.go:1294`)是 **原子认领**:`UPDATE … SET status='pending' WHERE id=? AND status=? AND ingest_updated_at<=?`,多副本只一个能推进,防重复入队。
- **3 次重试 + parse 缓存** `runIngestWithRetries`(`rag.go:481`):`parseCache` 跨重试缓存已解析文本——**瞬时嵌入/DB 失败只重跑便宜的嵌入,绝不重复付费 MinerU OCR**。退避 = 3s×attempt。`finalizeIngestFailure`(`rag.go:554`)用**全新独立 context**(任务 context 已超时不可复用),清 chunk 删向量后把状态写 failed。

### 3.4 删除-向量联动

删除文档触发 DB trigger 写 `pending_vector_cleanup`(`store/resource_lifecycle.go:10`),`PendingVectorCleanup` 只挑"无活跃写租约 & 文档已不存在"的行清理,失败记 `attempts/last_error` 重试(`SweepDeletedDocumentVectors`)。

---

## 4. 向量存储抽象与两种后端对比

### 4.1 `VectorStore` 接口(`vector/vector.go:62`)

```go
type Store interface {
    Enabled() bool
    Upsert(ctx, dim int, points []Point) error
    Search(ctx, dim int, vector []float32, scope Scope, topK int) ([]Hit, error)
    SearchKeyword(ctx, dim int, query string, scope Scope, topK int) ([]Hit, error)  // 混合检索独立关键词腿
    ExistingChunkIDs(ctx, dim int, scope Scope) (map[string]bool, error)
    VectorChunkStatuses(ctx, dim int, scope Scope) (map[string]ChunkVectorStatus, error)
    AllVectorChunkStatuses(ctx, dim int) (map[string]ChunkVectorStatus, error)
    DeleteByDocument / DeleteByKB / DeleteByConversation
}
```

**`Scope` 隔离**(`vector.go:50`):`KBIDs + ConversationID + DocumentIDs`,语义为"kb_id ∈ KBIDs OR conversation_id = ConvID;DocumentIDs 提供时替换 Conversation 分支"。这是**租户/会话隔离**的唯一手段——检索时据此过滤。

**`Enabled()` 为 false**(Disabled 存储)时,RAG 层退化为**全上下文注入**(见 `rag.go:1263` fullContext 逻辑)。

### 4.2 Qdrant 实现(`vector/qdrant.go`)

- **按维度一 collection**:`collectionName(dim) = "aivory_c"+dim`(`qdrant.go:79`),前缀 `aivory_c` 用于枚举和租户隔离。
- **点 ID**:chunk 字符串 id → `uuid.NewSHA1(pointNamespace, chunkID)` 确定性 UUIDv5(`qdrant.go:27`、`196`),**重复摄取幂等 upsert**。
- **集合创建** `ensureCollection`(`qdrant.go:139`):首次见到某维度 `PUT /collections/{name} {vectors:{size, distance:"Cosine"}}`;并建 payload 索引 `kb_id/conversation_id/document_id`(keyword)+ `content`(multilingual text,供关键词腿)。Cosine 匹配 L2 归一化嵌入。
- **稠密检索** `Search`(`qdrant.go:227`):`POST /points/search {vector, limit:topK, filter:{should}}`。
- **关键词腿** `SearchKeyword`(`qdrant.go:271`):`POST /points/scroll` 带 `{filter:{should + must:{content match}}}`;**Qdrant text-match 只是 yes/no 谓词**,真正的类 BM25 排序由调用方(RRF)在内存完成(`qdrant.go:309`)。
- **存量一致性** `ExistingChunkIDs`/`VectorChunkStatuses`(`qdrant.go:322`/`372`):scroll 枚举,只带 payload/chunk_id**不带向量**,因关系库才是渲染真相;`HasVector` 校验向量非空用于未完成 restore 检测。
- **删除并发** `deleteByField`(`qdrant.go:487`):扫全部 collection(调用方不知文档落在哪个维度),`AIVORY_VECTOR_DELETE_CONCURRENCY=4` 并发删。
- **归档门** `qdrantArchiveGate`(`qdrant.go:37`):备份恢复锁全读写,恢复后 bump generation 使各客户端缓存失效(`LockQdrantArchive`)。
- **对话 fork 复制** `fork.go`:scroll 出源会话向量,按 `remap` 改 payload id 后 upsert(不重新嵌入)——版本分支的向量复制。

### 4.3 SQLite 内嵌实现(`vector/sqlite.go`)

个人单实例部署:无守护进程、无原生扩展,精确余弦搜索进程内完成。

- **存储**:表 `vector_points(chunk_id, dimension, embedding)`;embedding 为 **L2 归一化后的 float32 LE 字节** 序列(`encodeNormalizedVector`,`sqlite.go:407`);upsert `ON CONFLICT(chunk_id,dimension) DO UPDATE`(`sqlite.go:41`)。
- **检索** `Search`(`sqlite.go:67`):`JOIN chunks` 取 scope 内全部向量 → 内存 `dotProduct`(归一化后即余弦,`sqlite.go:457`)→ **exact topK**。
- **关键词腿** `SearchKeyword`(`sqlite.go:130`):**刻意不用 FTS5**,用内存词频 `sqliteKeywordScore`(`sqlite.go:477`,大小写归一、子串含 term +1),保证个人镜像 amd64/arm64 行为一致、无原生扩展依赖。
- **状态/清理**:`VectorChunkStatuses` 用 `length(embedding)==dim*4` 判向量是否为空(`sqlite.go:255`);删除用 `DELETE FROM vector_points WHERE chunk_id IN (SELECT … FROM chunks WHERE field=?)`(`sqlite.go:272`)。
- **维度受限**:SQLite 后端与 PostgreSQL 数据库不兼容(`config.go:235`)——嵌入向量必须活在同一个 SQLite 文件里。

### 4.4 对比

| 维度 | Qdrant | SQLite 内嵌 |
|---|---|---|
| 部署 | 独立守护进程(HTTP) | 无,进程内 |
| 相似度 | Cosine(集合级) | Cosine(L2 归一化后点积) |
| top-k | 近似 ANN | 精确全扫 |
| 维度隔离 | 一 collection/keep 维度 | 一 (chunk,dim) 行 |
| 关键词腿 | Qdrant text 索引 + 调用方 BM25-ish | 内存词频 |
| 适用 | 生产多实例 | 单实例个人版 |

### 4.5 `vector_backend` 配置选择(`config.go:209` + `cmd/api/main.go:109`)

```go
const ( VectorBackendAuto="auto"; VectorBackendQdrant="qdrant"; VectorBackendSQLite="sqlite"; VectorBackendDisabled="disabled" )
// config.go:219 ResolveVectorBackend
//   auto:   QDRANT_URL 非空 → qdrant,否则 disabled        (保持历史行为)
//   qdrant: 必须 QDRANT_URL,否则报错
//   sqlite: 必须 SQLite DATABASE_URL(拒绝 postgres)
//   disabled:强制禁用
```

`main.go` `switch vectorBackend` 分别 `SetVectorStore(vector.NewQdrant(cfg.QdrantURL,cfg.QdrantAPIKey))` / `vector.NewSQLite(db)` / 默认禁用。Redis 配置同开关 asynq(`main.go:123`)。

---

## 5. 检索注入协议(前端如何展示引用)

### 5.1 检索入口

- `Retrieve`(`rag.go:1180`):单对话,vector 不可用时 fail-open 到全文本/有界关键词。
- `RouteAndRetrieve`(`rag.go:2929`):先 token 预算路由。
- `RetrieveDocuments` / `RouteAndRetrieveDocuments` / `RouteAndRetrieveDocumentScope`:限制到当前 turn 的文档附件。
- `retrieveStrict`(`rag.go:1196`):KB 迭代 API 专用,索引故障必须报错而非伪装成空命中。

### 5.2 混合检索流程(`retrieve`/`searchScope`)

1. **scope 判定**(§4.1)+ `ValidateKBEmbeddingCompatibility`:同轮多 KB 必须同一嵌入模型/维度,否则报错(`rag.go:1293`)。
2. **query 嵌入**(缓存)→ 稠密腿 `vec.Search(dim, qVec, scope, 30)` + 关键词腿 `vec.SearchKeyword(…, 30)` + 关系库词法腿(仅未嵌入对话块/无索引兜底,`rag.go:1696`、`1608`)。
3. **融合 `fuseReciprocalRank`** 按 1/(rank+k),k=60 默认(`rag.go:1992`):向量排序腿 + 关键词排序腿各累加一次。
4. **截断**:固定 TopK(默认 8 或 cfg)或动态 TopK(cosine ≥ `SimThreshold` 0.5,`rag.go:1421`)。
5. **跨 KB 轮插** `interleaveKnowledgeBaseCandidates`(`rag.go:2054`,固定 TopK 模式)。
6. **rerank**(可选,`reranker.go`):独立 `GET/POST /v1/rerank`(OpenAI 兼容),TopN≤24 候选,5s 超时,失败回退 RRF 原序——**重排只是质量插件,绝不能因它阻断检索**(`reranker.go:122`)。
7. **邻接窗口扩展**:命中子块 → 同父 ±1 邻接块拼成 snippet(`childWindow`,`rag.go:1551`);无法关联则 `expandHit(parent, child)` 用父块包裹(`rag.go:2188`)。
8. 对话聚合超预算时 `limitSnippetsToTokenBudget`(总 token ≤ FullTextThreshold)。

### 5.3 返回给前端的 `Snippet`(引用协议)

```go
// rag.go:1137
type Snippet struct {
    ID      string `json:"id"`
    Index   int    `json:"index"`
    Title   string `json:"title"`    // 即 filename
    URL     string `json:"url"`
    Snippet string `json:"snippet"`  // 注入的文本
    Source  string `json:"source"`   // "kb" | "document"
}
```

- `Source`(`snippetSource`,`rag.go:1503`):KB 块 → `"kb"`;对话上传 → `"document"`。
- `URL`(`snippetDocumentURL`,`rag.go:1515`):KB → `"kbdoc://<document_id>"`;对话 → `"doc://<document_id>"`。注释明确:旧版本对话引用被持久化为 `source="kb"`+`doc://`,故**仅凭 Source 不能安全启用 KB-only 预览 UI**,必须同时看 URL scheme。
- `Index` 为 1 起连续序号。前端用这套协议套用"web-search 引用"同款组件渲染(`rag.go:1149` 注释)。

### 5.4 LLM 路由与多轮证据

- **路由 prompt** `buildRouterPrompt`(`rag.go:4225`):列出在作用域文档 metadata(可信)+ 最新用户消息 + `rag.router.rules` 系统提示;决策输出 `RouteDecision{strategy: none|retrieve|full_doc, document_ids, queries}`(`rag.go:2777`)。
- **TaskLLM 调用**:优先 `DocumentRouter.RouteDocuments`(类型化路径),否则 `task.RunJSON("task.router", …)`(`rag.go:3094`);12s 超时(`routerCallTimeout`),失败回退 plain retrieve。
- **full_doc** 超预算 → **map-reduce 摘要** `mapReduceSummarise`(`rag.go:4133`):按 ~6000 token/组、最多 8 组分批,每组 `rag.map_reduce.input` prompt 让 TaskLLM 出 ≤200 字符摘要,绝不跨文档合并进单条引用。
- **迭代证据扩展** `RouteAndRetrieveIterative`(`rag.go:3218`):首轮(命中检索判定足够)即返回;否则 `rag_evidence_judge` **判据模型**(`judgeIterativeEvidence`,`rag.go:3641`,`task.rag_evidence_judge`)给出 `sufficient? queries?`,用 follow-up 查询再检索扩证,最多 3 条新 query、24 候选,直至证据充分或无新片段。判据 prompt 构造于 `buildEvidenceJudgePrompt`(`rag.go:3660`)。状态机:`found | partial | no_hit | error`(`rag.go:2789`)。

---

## 6. 「Python 复刻要点」

> 目标:**保持与前端/编排器/管理后台的线协议完全一致**(状态机、Snippet JSON、URL/source 约定、MinerU API、Qdrant API),仅替换后端实现语言。复刻顺序建议从最易验证、最独立的层开始。

### 6.1 建议技术选型映射

| Go 实现 | Python 对应 | 说明 |
|---|---|---|
| `pdf`(ledongthuc) 本地 PDF 文本 | `pypdf` / `pdfplumber` | `pypdf` 轻量提文本层;`pdfplumber` 可提表格。**注意**:Go 版先探测文本层决定"本地 vs MinerU OCR"的分流逻辑要照搬 | 
| `extractOfficeXML`(docx/pptx zip 解 XML) | `python-docx` / `python-pptx` | 或 `markitdown`(微软官方,统一转 Markdown,正好喂给后段 Markdown 分块) |
| 电子表格 | `openpyxl`(xlsx) / `pandas`(csv) | 对齐 `spreadsheet.go` 的"行/列/sheet 上限"截断行为 |
| 传统 doc/ppt/扫描件 OCR | 复用 **MinerU 云 API**(协议同 Go) | 完全同构,**必须保留 `model_version=pipeline, is_ocr=true, enable_formula, enable_table`** |
| 通用 doc 提取兜底 | `unstructured` / `markitdown` | 可作为本地解析的补充;若完全自托管可选 |
| 分块 | `langchain_text_splitters` 或 **自研翻译** | Aivory 的"标题面包屑 + 原子表/代码 + 重叠 + JJK句切"语义独特,建议**自研**照抄 `rag.go:2436-2731` |
| 嵌入 client | `openai` SDK(OpenAI 兼容) + 本地 `sentence-transformers`/ollama | 保持 `EMBEDDING_MODEL/EMBEDDING_DIM/EMBEDDING_BASE_URL` 兼容;本地哈希 embedder 可保留做 dev fallback(等价 `aivory-local-embed`) |
| 异步任务 | `celery`(+redis broker) / `arq` / `RQ` | 见 6.3 |
| Qdrant 客户端 | `qdrant-client`(官方 Python) | 请求/路径/过滤结构直接对照 `qdrant.go` |
| SQLite 向量 | `sqlite-vec` / 自研 Python 精确余弦 | 若追求零外部依赖可自研(见 6.5) |

| ChromaDB/本地库(备选) | `chromadb` | 若要"无 Qdrant 守护进程"的本地向量可考虑,但必须自实现 Scope 过滤、关键词腿、维度隔离 |
| rerank | `sentence-transformers` 的 CrossEncoder 或本地 rerank HTTP | 对齐 `reranker.go` 的 OpenAI 兼容 `/v1/rerank` 协议 |
| TaskLLM(路由/判据/map-reduce) | 复用现有 LLM 编排 | 沿用 `task.router` / `task.rag_evidence_judge` 两种 task kind,输出 JSON 结构不变 |

> 注:不推荐把 `langchain` 全家桶当重依赖,`langchain_text_splitters`/`qdrant-client`/`openai` 都是单库轻量;Aivory 的协议字段(Snippet 的 `id/index/title/url/snippet/source`)是复刻的"引力中心"。

### 6.2 复刻顺序(建议)

**Phase 1 —— 读取与分块(无外部依赖,可单测)**
1. 解析层:`pypdf`(PDF 文本层探测)/`python-docx`/`python-pptx`(或 `markitdown` 统一转 Markdown)/`openpyxl`(电子表格)。照抄 `parseDocument` 的**本地 vs MinerU 分流决策树**(`parser.go:96`)——这是成本与延迟的关键。
2. 分块层:自研翻译 `chunkHierarchical` 全链路(`rag.go:2436-2731`):标题面包屑、原子表/代码、句切、重叠。单元测试对齐 Aivory 的 `chunk_token_test.go` / `tokenize_cjk_test.go` 语义。
3. 本地哈希 embedder(`rag.go:2382`),让**零配置即可端到端跑通**.

**Phase 2 —— 向量后端(先 SQLite 后 Qdrant)**
4. `vector_points` 表(或 `sqlite-vec`)自研精确余弦 + 关键词腿(`sqlite.go`)→ 最容易先复刻、无守护进程。
5. `qdrant` 后端:直接用 `qdrant-client`,方法签名对照 `Store` 接口(6 个查询/删除方法);实现**维度隔离 collection + Scope(should 过滤)+ `pointID` UUIDv5**。

**Phase 3 —— 异步摄取**
6. 队列抽象:定义类似 `Enqueue(name, job)` 的薄接口;生产接 Celery/arq,dev 用一个 `ThreadPoolExecutor`。
7. heartbeat+租约+恢复:在 DB 上实现 `ingest_updated_at` 心跳、`document_vector_writes` 写租约、`pending_vector_cleanup` 清理、`ClaimStaleIncompleteDocuments` 原子认领;起看门狗循环。

**Phase 4 —— 检索与路由(依赖前面各层)**
8. `searchScope`:query 嵌入(缓存)→ 稠密+关键词+词法 三腿 → RRF 融合 → 截断 → 邻接窗口。
9. `RouteAndRetrieve` + `RouteAndRetrieveIterative`:token 预算全文注入、TaskLLM 路由、证据判据、map-reduce。

**Phase 5 —— 管理/运维**
10. `vector_admin.go` 的向量审计/重建(`RebuildMissingVectors`)、`vector_lifecycle` 的清理与 fork 复制、rerank 插件开关。

### 6.3 Python 异步任务机制建议

Aivory 的"进程内队列兜底 + asynq 持久"双轨,以及 **DB 心跳/租约/恢复**是核心,不依赖某个具体队列产品:

- **Celery**:任务函数即 `parse+chunk+embed+upsert`,`acks_late + task_reject_on_worker_lost` 模拟 asynq 的租约语义;用 result backend 或 DB 表维护 `ingest_updated_at` 心跳。
- **arq**(asyncio 原生):轻量,`@cron` 可承担恢复看门狗,`job time limit` 等效 job 超时。推荐单实例 Python 复刻(fastapi + arq + qdrant/sqlite)。
- 关键不是选哪个 queue,而是**保留三层**:唯一性(唯一锁/队列去重)、租约(心跳+过期认领)、恢复(启动重排 + 周期扫描)。这三层 Aivory 都落在 DB,换语言不换协议。

**并发与超时映射**:
- 任务超时 —— Go `ingestTaskTimeout=75m`/`ingestPipelineTimeout=70m`(须覆盖 MinerU 20 分钟轮询)→ Python 设 70-75 分钟任务超时。
- worker 并发 —— `rag-fast=4`/`rag-slow=4`(避免一个大 OCR 饿死轻活)→ Celery/arq 分两条队列。
- 嵌入批并发 —— Go `embedConcurrency=4`、DashScope 全局槽 2(保护 provider)→ Python 用 `ThreadPoolExecutor(max_workers=4)` + 全局信号量。

### 6.4 协议一致性清单(前端/编排器强依赖)

| 协议项 | 值 | 出处 |
|---|---|---|
| `documents.status` | `pending/parsing/embedding/ready/failed` | `store/kbs.go:1264` |
| `Snippet.Source` | `"kb"` / `"document"` | `rag.go:1503` |
| `Snippet.URL` | `kbdoc://<doc>` / `doc://<doc>` | `rag.go:1515` |
| 错误码 | `document_parser_not_configured` | `parser.go:78` |
| Router 决策 | `RetrieveDecision.strategy ∈ {retrieve, full_doc, none}` | `rag.go:2777` |
| 迭代状态 | `found/partial/no_hit/error` | `rag.go:2789` |
| Task kind | `task.router`、`task.rag_evidence_judge` | `rag.go:3098`、`3650` |
| embedding 配置 | `EMBEDDING_MODEL/DIM/BASE_URL/API_KEY` + 管理表 `embedding_model_id` | `embedder_http.go:627` |
| MinerU API | `POST /api/v4/extract/task` + `GET /api/v4/extract/task/{id}` | `parser.go:736/791` |
| rerank 配置键 | `rag_rerank_enabled/api_url/api_key/model` | `reranker.go:52` |
| RAG knob | `rag_full_text_threshold=8000`、`rag_top_k=8`、`rag_similarity_threshold=0.5`、`rag_code_full_text_max_lines=2000` | `rag.go:2851` |
| 向量点 ID | `uuid5(ns, chunk_id)`,`ns=8f4d2c1a-1f2e-4b6a-9c3d-7a0b1c2d3e4f` | `qdrant.go:27` |
| collection 命名 | `<prefix>c<dim>` 前缀 `aivory_c`,相似度 Cosine | `qdrant.go:22/79/161` |

### 6.5 Python SQLite 向量(零外部守护进程方案)

对标 `sqlite.go`:
- 表:`vector_points(chunk_id, dimension, embedding BLOB)`;embedding 存 **L2 归一化后的 float32 LE 字节**,长度 = dim*4(`sqlite.go:407`)。
- 检索:单线程内 `SELECT` scope 行 → Python 点积 → topK(**不缩放、不排序出 top 后要截断**)。
- 关键词腿:内存词频 `sqliteKeywordScore`(大小写归一、`term in content +1`),**不自带 FTS5 依赖**。
- scope 过滤:直接翻译 `sqliteScopeClause`(`sqlite.go:360`):`kb_id IN (…) OR conversation_id=?`(DocumentIDs 存在时替代 Conversation 分支)。
- 若要"可扩展 + 无守护",可用 `sqlite-vec` 虚拟表做 ANN;但 Aivory 个人版用**精确扫描**即可,无需向量扩展。

### 6.6 Python Qdrant 后端要点

直接用 `qdrant-client`(sync 或 async):
- **保维度隔离**:每个 `dim` 一个 collection(`<prefix>c<dim>`),建 collection 时 `distance=Cosine`,并建 `kb_id/conversation_id/document_id` keyword 索引 + `content` text 索引(multilingual,供关键词腿)——照抄 `ensureCollection`(`qdrant.go:139`)。
- **点 ID**:`uuid.uuid5(NAMESPACE, chunk_id)` 保证幂等 upsert。
- **检索过滤**:`filter={"should":[{"key":"kb_id","match":{"value":..}},…]}`,稠密用 `search`,关键词腿用 `scroll + content match` 谓词,排序留给 RRF(`qdrant.go:271`)。
- **一致性守卫**:`VectorChunkStatuses`(scroll 校验 chunk 在库且向量非空)是"DB 是真相、向量库仅索引"的实现,检索前调用 `ensureVectorIndexComplete`(`rag.go:1864`)——若库损坏则 fallback 全文/关系检索。

### 6.7 复刻时的坑(从源码注释学到的教训)

1. **嵌入维度被 provider 忽略**:`embedOne` 去掉 `dimensions` 重试 + 收货后校准 `dim` 并回写 `knowledge_bases.embedding_dim`(`embedder_http.go:276`、`rag.go:901`)。不照做会因"错维向量进集合"被 Qdrant 拒收整批。
2. **Qdrant delete 要扫全部集合**:调用方不知 doc 落在哪个维度,`deleteByField` 遍历所有 `aivory_c*`(`qdrant.go:487`)。
3. **MinerU zip URL 是 SSRF 面**:只允许 http(s)、用 SSRF-safe client、zip ≤500MiB(`parser.go:868/892`)。
4. **任务 context 超时后不可复用**:`finalizeIngestFailure` 必用全新独立 context 写终态,否则行永远停在 `parsing/embedding`(`rag.go:554`)。
5. **embed 结果数与输入数必须一致**:少了就报错而非错位写向量(`embedder_http.go:293`)。
6. **线程池 worker 要 recover**:PDF reader 解析恶意文件会 panic,不 recover 会静默杀死 worker(`queue.go:92`)。
7. **交叉 KB 必须同模型同维度**:否则报错让编排器拆分(`rag.go:1293`)。
8. **rerank/路由失败只降级不阻断**:rerank 回退 RRF;路由失败回退 plain retrieve;证据判据缺 `sufficient` 字段报错但**不允许模型空证通过**(`rag.go:3324`)。
