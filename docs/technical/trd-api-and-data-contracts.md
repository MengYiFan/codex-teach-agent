# TRD：API 与数据契约

## 契约来源与约定

- MVP-A 的机器可读 HTTP 契约以 [OpenAPI 3.1 文件](../api/openapi/mvp-a.yaml) 为准；本文件解释边界和跨接口规则。
- 基础路径为 `/api/v1`，JSON 字段使用 `snake_case`，时间使用 UTC RFC 3339，ID 使用 UUID。
- 除公开示例和健康检查外均需登录；API 从会话/JWT 获取 `actor_id`，不接受客户端传入用户 ID。
- 每个响应返回由服务端生成的 `X-Request-Id`，并与错误体 `request_id` 一致；客户端不得指定或覆盖该值。客户端自己的链路标识只通过受限的 `client_trace_id` 字段传入。
- 创建任务、重试、重写、导出等非幂等操作必须携带 `Idempotency-Key`。

## 快速生成

```http
POST /api/v1/quick-start/sessions
GET  /api/v1/quick-start/sessions/{session_id}
PATCH /api/v1/quick-start/sessions/{session_id}
GET  /api/v1/quick-start/sessions/{session_id}/files
POST /api/v1/quick-start/sessions/{session_id}/files
DELETE /api/v1/quick-start/sessions/{session_id}/files/{file_id}
POST /api/v1/quick-start/sessions/{session_id}/confirm-context
POST /api/v1/quick-start/sessions/{session_id}/generate
POST /api/v1/quick-start/sessions/{session_id}/retry
POST /api/v1/quick-start/sessions/{session_id}/cancel
```

创建会话请求：

```json
{
  "intent": "teach_tomorrow",
  "source_type": "file_with_text",
  "source_text": "高一物理牛顿第二定律，45 分钟，增加一个生活化导入",
  "preferred_outputs": ["teacher_tips", "lesson_plan_draft", "study_notes"],
  "optional_context": {
    "subject": "物理",
    "grade": "高一",
    "duration_minutes": 45,
    "student_level": "normal",
    "teaching_scenario": "regular_class"
  },
  "client_trace_id": "qs_20260627_001"
}
```

`source_type=file` 时 `source_text` 必须为空；`text` 和 `file_with_text` 时必须提供 10-2000 字。OpenAPI 使用条件 Schema 表达该约束，服务端必须再次校验。`preferred_outputs` 为空时按用途矩阵填充。创建会话不创建项目：

```json
{
  "session_id": "5b123e8e-790d-4f5c-bd0e-f5d6de60f18b",
  "project_id": null,
  "active_run_id": null,
  "latest_run_id": null,
  "status": "awaiting_source",
  "context_version": 1,
  "next_action": "upload_file",
  "expires_at": "2026-06-28T12:00:00Z",
  "teacher_visible_message": "请上传 PPTX 或 PDF"
}
```

文本模式创建后直接返回 `source_ready`。文件通过 `multipart/form-data` 上传；MVP 限单文件 50 MiB、单会话 1 个主文件。文件校验完成前状态为 `source_validating`。

会话处于 `awaiting_source` 或 `source_ready` 且没有活动 run 时，可以通过 `PATCH /quick-start/sessions/{id}` 修改用途、`preferred_outputs` 和文本说明。请求必须携带 `expected_context_version`；修改成功递增版本。`GET /quick-start/sessions/{id}/files` 用于刷新页面后恢复临时文件 ID、名称和校验状态。

`POST /quick-start/sessions/{id}/retry` 只接受最近 settled run 中可重试的失败 task ID，并一次性创建新的 retry run；已经成功的产物不会进入本轮。普通 `POST /tasks/{id}/retry` 用于块重写、导出等不属于 settled quick-start run 的独立任务。

项目创建前，quick-start `confirm-context` 更新会话上下文；项目创建后，该接口与项目 `confirm-context` 使用同一事务服务，更新 project context/version 并同步会话投影。会话清理后只能使用项目接口。两个入口都要求相同的 `expected_context_version` 和 `Idempotency-Key`。

生成请求必须在 `source_ready` 状态调用：

```http
POST /api/v1/quick-start/sessions/{session_id}/generate
Idempotency-Key: 7d10a6c0-74a6-47fd-aa78-4c1911819c5c
```

```json
{
  "context_version": 2
}
```

成功返回 `202 Accepted`，并原子创建项目和初始任务：

```json
{
  "session_id": "5b123e8e-790d-4f5c-bd0e-f5d6de60f18b",
  "project_id": "5965a453-a9e5-45f0-a8bb-66202c4d44cb",
  "run_id": "e2041f29-1828-47ad-93d8-f154fcad0f50",
  "task_id": "c9c6bd11-57ee-438a-887f-170a4bfc27a9",
  "status": "queued",
  "requested_at": "2026-06-27T12:01:00Z"
}
```

状态响应必须同时包含内部状态和教师可见信息：

```json
{
  "session_id": "5b123e8e-790d-4f5c-bd0e-f5d6de60f18b",
  "project_id": "5965a453-a9e5-45f0-a8bb-66202c4d44cb",
  "active_run_id": "e2041f29-1828-47ad-93d8-f154fcad0f50",
  "latest_run_id": "e2041f29-1828-47ad-93d8-f154fcad0f50",
  "status": "generating",
  "current_step": "generating_quiz",
  "teacher_visible_message": "学习要点已生成，随堂小测还在生成中",
  "progress_percent": 65,
  "available_outputs": ["study_notes"],
  "detected_context": {
    "grade": {
      "value": "高一",
      "source": "model_inference",
      "confidence": 0.72,
      "needs_confirmation": true
    }
  },
  "warnings": [],
  "recoverable_errors": []
}
```

状态流转、项目创建时机和失败策略见 [快速生成状态机](trd-quick-start-state-machine.md)。

## 项目、任务与输出

```http
GET    /api/v1/projects
GET    /api/v1/projects/{project_id}
PATCH  /api/v1/projects/{project_id}
POST   /api/v1/projects/{project_id}/confirm-context
DELETE /api/v1/projects/{project_id}
GET    /api/v1/deletion-operations/{operation_id}
GET    /api/v1/projects/{project_id}/files
GET    /api/v1/projects/{project_id}/text-sources
GET    /api/v1/files/{file_id}/parsed-result
GET    /api/v1/assets/{asset_id}/download

GET  /api/v1/projects/{project_id}/tasks
GET  /api/v1/projects/{project_id}/generation-runs
GET  /api/v1/generation-runs/{run_id}
GET  /api/v1/tasks/{task_id}
GET  /api/v1/tasks/{task_id}/attempts
POST /api/v1/tasks/{task_id}/retry
POST /api/v1/tasks/{task_id}/cancel

GET /api/v1/projects/{project_id}/outputs
GET /api/v1/outputs/{output_id}
GET /api/v1/outputs/{output_id}/versions
POST /api/v1/outputs/{output_id}/versions/{version}/restore
```

MVP-A 不提供空白项目创建接口；项目只由快速生成流程创建。`PATCH /projects/{id}` 只修改标题；课程上下文使用版本化 `confirm-context`，因此 quick-start session 清理后仍可选择 `future_only` 或创建 `regenerate_context` run。`DELETE /projects/{id}` 是幂等软删除并返回 `202` 及 `deletion_operation_id`；资源立即不可访问，客户端通过删除操作接口查询在线对象、模型明细和备份 tombstone 的清理状态。删除操作按 `requested_by_user_id` 独立鉴权，因此父项目不可见后仍可查询；相同幂等键重放返回原操作。

## 内容块编辑与重写

```http
GET   /api/v1/outputs/{output_id}/blocks
GET   /api/v1/content-blocks/{block_id}
PATCH /api/v1/content-blocks/{block_id}
DELETE /api/v1/content-blocks/{block_id}?expected_version={version}
POST  /api/v1/content-blocks/{block_id}/regenerate
GET   /api/v1/content-blocks/{block_id}/versions
POST  /api/v1/content-blocks/{block_id}/versions/{version}/restore
```

编辑请求使用乐观锁：

```json
{
  "expected_version": 3,
  "title": "生活中的惯性",
  "body": { "format": "rich_text", "content": [] },
  "change_summary": "修正例子并精简表述"
}
```

版本冲突返回 `409 VERSION_CONFLICT` 和当前版本号。编辑和重写永远创建新版本，不覆盖旧版本。重写请求：

```json
{
  "expected_version": 3,
  "action": "simplify",
  "instruction": "保留公式，换成高一学生熟悉的例子"
}
```

`action` 可取 `simplify`、`expand`、`change_example`、`change_questions`、`custom`。指令最大 1000 字。

删除内容块创建不可变 tombstone 版本，并创建不再引用该块的新 OutputVersion；历史输出仍可读取旧块版本。恢复请求必须携带 `expected_current_version`，防止覆盖请求者未见的新修改。恢复内容块会复制指定历史版本为新的未审核版本，而不是把当前指针改回旧版本，并清除父块的 `deleted_at`。整份输出恢复会按历史 manifest 为每个块创建新的未审核恢复版本，恢复当时已删除/存在的块集合，为当前多出的块创建 tombstone，并创建一个 `review_status=unreviewed` 的当前 OutputVersion。所有编辑、删除、确认、重写和恢复事务都必须先锁定所属 output，在锁内最新 manifest 上合并变更。

## 审核

```http
GET  /api/v1/projects/{project_id}/review-findings
POST /api/v1/review-findings/{finding_id}/resolve
POST /api/v1/content-blocks/{block_id}/confirm
```

MVP-A finding 类型为 `missing_source`、`low_confidence_context`、`invalid_question_structure`、`answer_format_mismatch`、`single_choice_not_unique`。`resolve` 的动作可为 `confirmed`、`fixed`、`dismissed`。所有请求使用 `expected_finding_version` 防止重复处理；块级 finding 还必须附带 `expected_block_version`，项目/上下文级 finding 则附带 `expected_context_version`。创建新块版本时，旧版本仍为 `open` 的 finding 转为 `superseded`，不再阻塞当前版本；上下文 finding 绑定 `context_version`。确认内容块会创建正文不变、`review_status=confirmed` 的新 ContentBlockVersion 和对应 OutputVersion；后续编辑、重写或恢复产生的新版本重新回到 `unreviewed`。OutputVersion 只有在清单中所有块版本已确认且没有当前 blocking finding 时才投影为 `confirmed`。

处理 finding 和确认内容是两个明确动作：`resolve` 只记录风险项处置，不自动改变块审核状态；存在当前版本的 open blocking finding 时，确认接口返回 `INVALID_STATE_TRANSITION`。教师在同一 UI 操作中同时选择“接受风险并确认内容”时，API 层先处理 finding，再用返回的当前版本确认；任一步冲突都刷新版本，不能伪装为单个非原子写入。

## 导出

```http
POST /api/v1/projects/{project_id}/exports
GET  /api/v1/exports/{export_id}
GET  /api/v1/exports/{export_id}/download
```

创建导出请求：

```json
{
  "format": "pdf",
  "output_versions": [
    { "output_id": "819c75f9-4967-4f75-b2d2-d94fa393dd48", "version": 2 }
  ],
  "template_key": "default_teacher_document"
}
```

创建返回 `202`。下载接口只在导出为 `succeeded` 时返回包含 `download_url` 和 `expires_at` 的下载票据，签名 URL 有效期不超过 10 分钟；数据库只保存 object key。支持 `pdf`、`docx`、`markdown`。

导出请求中的每个 OutputVersion 必须属于当前项目且互不重复。创建导出时在事务内解析并保存每个 OutputVersion 的 block manifest、CourseIR 版本和模板版本；后续编辑、恢复或审核不得改变该导出清单。

## 质量反馈与埋点

```http
POST /api/v1/projects/{project_id}/quality-feedback
POST /api/v1/events/batch
```

质量反馈固定到一组 `output_id + version`，使用 `direct_use`、`minor_edit`、`major_edit`、`unusable` 四档。一次幂等请求只产生一条反馈记录和服务端 `quality_feedback_submitted` 事件。

事件批量接口只接受事件字典中标记为“客户端”的事件名，最多 50 条；登录态可选，服务端从身份上下文或匿名会话生成不可逆 subject key。生成、导出、订阅等服务端事实事件不能由客户端提交。事件接收、去重、保留和指标口径见 [指标与埋点 TRD](trd-analytics-and-metrics.md)。

## 模型配置（管理员）

```http
POST   /api/v1/admin/model-providers
GET    /api/v1/admin/model-providers
PATCH  /api/v1/admin/model-providers/{provider_id}
DELETE /api/v1/admin/model-providers/{provider_id}
POST   /api/v1/admin/model-providers/{provider_id}/test
GET    /api/v1/admin/model-providers/{provider_id}/models
POST   /api/v1/admin/model-providers/{provider_id}/models
PATCH  /api/v1/admin/model-configs/{config_id}
DELETE /api/v1/admin/model-configs/{config_id}
```

这些接口要求 `model_admin` 权限。所有响应只返回凭据状态和末四位掩码，不返回可逆密文或完整 Key。

创建供应商/模型配置必须携带幂等键；更新和删除必须携带 `expected_version` 并写审计日志。凭据轮换与普通字段更新使用同一乐观锁，冲突不能静默覆盖另一名管理员的修改。

## MVP-B 预留接口

以下接口不进入 MVP-A OpenAPI，也不得作为阶段 1 的依赖：

```http
POST   /api/v1/projects/{project_id}/share-links
GET    /api/v1/projects/{project_id}/share-links
DELETE /api/v1/share-links/{share_id}
GET    /s/{public_token}
```

社区、Agent、fork/watch 接口属于后期生态，另行维护 OpenAPI，不与 MVP-A 契约混合。

## 错误响应

所有非 2xx 响应使用统一结构：

```json
{
  "error": {
    "code": "VERSION_CONFLICT",
    "message": "内容已被其他操作更新",
    "request_id": "req_01J1...",
    "retryable": false,
    "details": {
      "expected_version": 3,
      "current_version": 4
    }
  }
}
```

| HTTP | 错误码 | 场景 |
| --- | --- | --- |
| 400 | `VALIDATION_ERROR` | DTO、文件、字段或 Schema 不合法 |
| 401 | `AUTH_REQUIRED` | 未登录或登录态失效 |
| 403 | `FORBIDDEN` | 无资源或管理员权限 |
| 404 | `RESOURCE_NOT_FOUND` | 资源不存在或对当前用户不可见 |
| 409 | `INVALID_STATE_TRANSITION` | 当前状态不允许该动作 |
| 409 | `VERSION_CONFLICT` | 乐观锁版本冲突 |
| 409 | `IDEMPOTENCY_KEY_REUSED` | 同一键对应不同请求体 |
| 409 | `RESOURCE_BUSY` | 同一内容块已有活动重写任务 |
| 413 | `FILE_TOO_LARGE` | 超过 50 MiB |
| 415 | `UNSUPPORTED_FILE_TYPE` | 扩展名、MIME 或 magic bytes 不支持 |
| 422 | `SOURCE_REQUIRED` | 文件或文本来源缺失 |
| 422 | `FILE_UNREADABLE` | 文件损坏、加密或无法解析 |
| 429 | `QUOTA_EXCEEDED` | 套餐或速率额度不足 |
| 503 | `MODEL_UNAVAILABLE` | 所有允许的模型路由失败 |
| 503 | `TASK_QUEUE_UNAVAILABLE` | 任务暂时无法入队 |

`INVALID_STATE_TRANSITION` 的 `details` 必须包含 `current_state` 和 `allowed_actions`；`VERSION_CONFLICT` 必须包含 `expected_version` 和 `current_version`；`RESOURCE_BUSY` 必须包含活动任务 ID。OpenAPI 为这些错误定义有类型的 detail Schema，客户端不得依赖自由文本解析。

生产环境错误不得回传栈、SQL、对象存储路径、模型凭据或原始敏感内容。

## 幂等、分页和删除

1. `Idempotency-Key` 为 UUID，结果保存 24 小时；同一 actor + 规范化 route template + key 唯一，path ID 和请求体共同进入 fingerprint。
2. 可能增长的列表使用 cursor 分页：`?limit=20&cursor=...`，`limit` 范围 1-100。cursor 是不透明签名值并绑定 actor、route、过滤条件和排序锚点；项目按 `updated_at DESC, id DESC`，任务/finding/管理配置按 `created_at DESC, id DESC`，run 按 `requested_at DESC, id DESC`，版本按 `version DESC`，当前内容块按 `position ASC, id ASC`。篡改、过期或换过滤条件复用返回 `VALIDATION_ERROR`。MVP 上限明确为 1-5 项的文件、文本来源、输出和 task attempt 可直接返回有界数组。
3. 删除项目先标记 `deleted_at` 并创建 deletion operation/tombstone，立即禁止访问；对象文件和模型日志按数据保留策略异步清理。
4. 自动重试在原逻辑任务下创建不可变 attempt；用户手动重试创建引用原 task 的后继任务。不能清空原 attempt 错误或覆盖成功产物。

## 数据落地原则

1. `quick_start_sessions` 保存首次生成输入和聚合状态，不长期承载业务内容。
2. 接受生成请求时创建 `projects`，临时文件随后绑定项目，临时 `source_text` 复制为使用独立数据密钥的项目级文本来源；CourseIR、输出、内容块和导出均归属项目。
3. 每次编辑、重写、删除、确认和恢复都创建不可变 ContentBlockVersion；审核状态属于具体版本。内容块变更同时创建一个新的 OutputVersion 清单，使整份输出可以稳定预览、导出和恢复。父记录版本号只作为并发缓存指针，不是历史事实源。
4. 模型调用记录 Prompt、模型、token、成本、延迟、数据策略和失败原因，但不保存 API Key 或未脱敏的完整输入。
5. 发布、分享、模型配置和删除等高影响动作写入不可变审计日志。
