# TypeSafe 决策客户端

Aivory 的 Go 后端通过 `server/internal/typesafe` 调用 TypeSafe 的
`POST /v1/systemone`。统一入口为 `TaskLLM.RunDecision`，输入 `state + questions`，
输出按问题 ID 索引的结构化答案。现已接入文件路由（文档使用策略与全文覆盖的相关文档选择）、工具路由、记忆去重、记忆冲突裁决和内容审核。

## 管理员配置首批策略

1. 在渠道管理中新建 `typesafe` 渠道，填写服务端 API Key。默认地址为 `https://api.typesafe.ai/v1`。
2. 拉取渠道模型列表，或手动创建模型。TypeSafe 渠道的模型会保存为 `decision` 类型；建议使用固定发布版本，例如 `jev-1.13.0`。输入价格由管理员维护，当前新建决策模型默认每百万输入 token 为 0.042 美元，部署时请核对最新价格。
3. 在 `/admin/settings/model-policy` 的对应策略中选择 Jev 并保存。模型和渠道均须启用，渠道须配置密钥。
4. 内容审核还需在需要保护的聊天模型上启用审核，并选择“模型审核”模式。审核分类、关键词和拦截文案仍在内容审核设置页维护；两个页面共用同一个审核模型设置。

| 策略设置 | 原语 | 当前行为与回退 |
| --- | --- | --- |
| `file_route_model_id` | Choice + Noul | Choice 判断 `none` / `retrieve` / `full_doc`，confidence < 0.8 时回到原问题检索。全文覆盖时逐文档 Noul 多选，概率 > 0.15 的候选保留；无候选入选或请求失败时回到检索。 |
| `tool_route_model_id` | Choice | 判断 `search_only` / `full_tools`；只有选择前者且 confidence ≥ 0.8 时缩小工具范围。出错或不确定保留完整工具。原有 URL、附件、技能、继续调用和小工具集规则优先执行。 |
| `memory_dedup_model_id` | Choice | 从已有记忆候选及 `none` 中选择；confidence ≥ 0.9 才合并。完全相同的文本先在本地去重；低置信度或失败不做语义合并。 |
| `memory_adjudicate_model_id` | Choice | 为每条旧记忆返回 `keep` / `stale` / `no_conflict` / `unknown_current`；confidence < 0.8 视为当前事实不明确。失败沿用原有保守冲突处理。 |
| `moderation_model_id` | Noul | 每个违规类别一个独立问题，一次请求。任意概率 ≥ 0.85 拦截；全部 ≤ 0.15 放行；其余情况或服务故障回到关键词检查。 |

这些阈值是初始业务配置，尚未通过真实中文样本标定；confidence 并非答案正确率保证。审核回退沿用现有机制：未命中关键词时会放行。

五项可以独立选择普通聊天模型或 Jev。记忆提取、聊天、标题、检索词改写和全文摘要等生成任务仍使用聊天模型。决策模型不进入公开聊天模型列表，也不支持流式输出、视觉、工具调用或 Deep Research。通用客户端同时支持 Choice、Score、Noul；首批流程按语义使用 Choice 和 Noul，没有人为加入 Score。

管理员策略调用直接读取数据库中的渠道密钥、模型版本和输入价格，**所有 TypeSafe 配置均通过管理员页面保存到数据库，不读取环境变量**。禁用、改密钥和修改价格会在后续调用生效。每次调用沿用每日 token 配额预留及实际用量结算；账本按管理员模型 ID 归集，诊断用量关联渠道 ID，日志记录请求版本及实际服务版本。策略调用默认最多 10 秒、不重试，工具路由遵守更短的现有路由超时。

文件路由在管理员页面仍为“文件路由模型”，同一选择控制文档使用策略和全文覆盖的相关文档多选。Jev 根据可访问范围内的文档 ID、文件名、是否本轮附件及索引状态判断，不接收整篇文档。小文件直接注入、当前附件及明确文件名优先等已有本地规则仍先执行。超过 128 个文档候选时直接回到普通检索，避免截断候选后误判；需要基于正文判断的情况仍由检索和回答流程处理。

Jev 的 `retrieve` 使用用户原问题，在原有授权范围内进行片段检索，不生成改写词，也不缩小知识库范围。`full_doc` 的候选 ID 会再次经现有范围校验，随后沿用全文注入／分段摘要流程；摘要和后续证据判断仍由普通聊天任务模型执行。

## 配置存储

| 管理员页面 | 数据库存储 | 配置内容 |
| --- | --- | --- |
| 渠道管理 | `channels` | TypeSafe 渠道类型、API 地址、API Key、启用状态 |
| 模型管理 | `models` | Jev 请求模型名称／版本、输入价格、启用状态 |
| 模型策略 | `settings` | 文件路由、工具路由、记忆去重、冲突裁决及审核所选的模型 ID |

保存后在后续调用生效，无需重启后端。未配置可用决策模型时不会发起 TypeSafe 请求。
超时和重试沿用服务端调用策略：默认 10 秒、不重试；工具路由使用更短预算，其他调用可通过 Options.Timeout 设置本次预算。

## 通用调用

在持有 `*llm.TaskLLM` 的后端业务中调用。`modelID` 为管理员选定的数据库模型 ID：

```go
result, err := task.RunDecision(ctx, modelID, typesafe.Request{
    State: map[string]any{
        "message": userText,
        "capabilities": []string{"web", "code", "file"},
    },
    Questions: map[string]typesafe.Question{
        "scope": typesafe.NewChoice(
            "Which tool scope does the message require?",
            map[string]any{
                "search_only": "Conversation, writing, or a focused web search",
                "full_tools": "Code execution, file work, or multi-step investigation",
            },
        ),
        "needs_file": typesafe.NewNoul(
            "Does the message ask to read, create or modify a file?", nil,
        ),
        "complexity": typesafe.NewScore(
            "How many dependent stages does the requested work involve?",
            []any{"One self-contained step", "A few dependent steps", "A broad investigation"},
        ),
    },
}, typesafe.Options{
    Timeout: 3 * time.Second,
    Metadata: typesafe.Metadata{
        Purpose: "task.tool_route",
        UserID: userID,
        ConversationID: conversationID,
        MessageID: messageID,
        WorkspaceID: workspaceID,
    },
})
if err != nil {
    // 由具体业务选择原模型、规则或其他回退；不要采用 result 中的判断。
    return err
}
scope := result.Answers["scope"]
selected := *scope.Choice
confidence := *scope.Confidence
needsFileProbability := *result.Answers["needs_file"].Noul
complexity := *result.Answers["complexity"].Score
```

通用调用和五个策略使用同一个入口，统一校验可用性、预留配额及记录消耗。`Metadata` 只用于本地统计和日志，
不会发送给 TypeSafe。Purpose 默认 `task.decision`，显式值必须以 `task.` 开头。

- State 可以是字符串、JSON 对象或数组，也支持可序列化为这些形状的 Go struct、
  typed map、slice 和 `json.RawMessage`。
- Instructions 和各 criteria 描述支持字符串、对象、数组或 null。
- Choice 的 criteria 是候选名称到描述的 map，支持 1–255 个候选。
- Score 的 criteria 是 2–10 个等级的有序数组；返回值范围是 0 到等级数减一。
- Noul criteria 可省略，或提供 `true` / `false` 描述；答案是 yes 的概率，没有 confidence。
- `Answer` 的标量字段使用指针，区分“字段缺失”与合法的零值。只有 `err == nil` 才能采用答案。
- API 问题 ID 用于映射结果，不参与模型推理；问题含义必须写在 instructions 中。
- 同一请求中的问题相互独立，不会看到其他问题的答案。业务阈值不放进客户端。

也可以直接创建 `typesafe.New(typesafe.Config{...})` 并调用 `Evaluate`，用于不依赖
TaskLLM 的服务或测试；直接客户端默认不重试，可注入 HTTPClient、Logger 和 Recorder。
应用业务应优先使用 TaskLLM 入口，以统一持久化用量记录。

## 超时与错误

HTTP 请求、响应读取和重试等待共用总 deadline；调用方已有更早 deadline 时优先生效。
Options.Timeout 只覆盖本次调用；零值沿用客户端配置。

429、529、502、503、504 可重试，使用带抖动的指数退避，尊重 Retry-After 秒数或 HTTP 日期。
401/403、400/422、格式错误、模型版本错误不重试。网络中断可能已发生计费，不自动重放；
500 也不自动重放。调用方取消或超时后立即结束请求/等待，不再发出下一次 HTTP 请求。
所有重定向均拒绝跟随，避免转发密钥或重放 POST。

`typesafe.KindOf(err)` 返回稳定错误类别，包括：

`disabled`、`configuration`、`validation`、`authentication`、`rate_limit`、`overloaded`、
`http`、`transport`、`timeout`、`canceled`、`invalid_response`、`model_version`、`recording`。

通过 `errors.As` 读取 `*typesafe.Error` 的 HTTP 状态和请求 ID；通过 `errors.Is` 检测
`context.Canceled`、`context.DeadlineExceeded` 和 `llm.ErrTaskBillingRecord`。
上游错误正文可能包含输入数据，不会进入返回错误或日志。

默认响应上限 8 MiB，可在客户端 Config 中修改。客户端检查问题与答案 ID、类型、必填值、
概率范围和分布、Choice 候选、Score 等级与 legend，以及非负 token 用量。无效答案不会作为
可用结果返回，但已解析出的有效用量仍保留并记录。

## 模型版本

应用入口使用数据库模型的 `request_id`，忽略 Request.Model 覆盖值。建议管理员配置固定版本；也支持显式选择 `jev-latest` / `jev-preview`。固定模型请求必须返回同一
版本，否则返回 `model_version` 错误。别名默认接受实际返回的版本，也可以通过
Options.ExpectedModel 锁定预期发布版本。ExpectedModel 不得与固定请求版本冲突。

Response 同时提供 RequestedModel 和 Model。日志也记录两者，便于检查别名漂移。
没有自动静默降级到其他模型。业务应在升级后重新验证自己的判断阈值。

## 用量、费用与日志

- TaskLLM Recorder 将服务端实际报告的 token 写入 `billing_usage` 和 `usage_logs`；
  成功记录由现有数据库机制同步到 `usage_stats`。ModelID 使用管理员配置的数据库模型 ID；诊断记录关联渠道 ID，实际服务版本由响应和日志保留。
- 按管理员保存的模型输入价格计算费用，输出 token 记录但不收费。新建模型默认价格来自本次读取的
  TypeSafe Models 文档；升级模型前应核对并配置对应价格。
- 回答无效或版本不符但用量有效时，仍写入持久化消耗账本，并记录错误诊断行。
- 使用模型别名时仍按该数据库模型的配置价格记账；管理员应在服务商调价或升级版本时更新价格。没有收到有效 usage 的失败请求标为未知用量，不估造 token 或费用。
- `billing_usage` 沿用现有美元微单位精度；极低的单次费用可能被舍入，精确 token 与诊断费用
  仍保留。所有 TaskLLM.RunDecision 调用均接入原有每日 token 配额准入与结算。
- Recorder 使用独立、有界的 context，在调用取消后仍能保存已知消耗；默认最多额外等待 3 秒。
  Config.RecordTimeout 可调整。因此 inference timeout 不包含这段记账时间。
- 记账失败会返回 `recording` 错误，且不会因此再次调用模型。系统任务无 UserID 时，沿用现有
  存储行为：消耗进入账本，不伪造用户创建诊断行。
- 默认日志包含用途、本地关联 ID、请求/服务模型、耗时、尝试次数、HTTP 状态、错误类别和用量。
  不记录 state、问题、答案、请求头、密钥或上游错误正文。private 消息的 conversation ID 被移除。
- 直接客户端的 Stats() 是线程安全的进程内累计统计；重启归零。持久记录以数据库为准。

## 验证与参考

```bash
cd server
go test -race ./internal/typesafe
go test ./internal/llm -run TestDecision -count=1
```

测试使用本地 HTTP 服务，不需要真实密钥，覆盖混合问题、结构化字段、合法零值、版本漂移、
429/529 重试、Retry-After、取消/超时、响应限制、敏感日志、并发统计和 SQLite 持久化记账。
这些测试验证客户端行为，不代表中文分类效果已通过真实模型评估。

官方资料：[索引](https://docs.typesafe.ai/llms.txt)、[HTTP API](https://docs.typesafe.ai/api)、
[结构化字段](https://docs.typesafe.ai/primitives/advanced)、[模型与价格](https://docs.typesafe.ai/models)、
[置信度](https://docs.typesafe.ai/confidence)。
