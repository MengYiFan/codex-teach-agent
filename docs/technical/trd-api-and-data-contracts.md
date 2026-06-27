# TRD：API 与数据契约

## 契约来源与约定

- MVP-A 的机器可读 HTTP 契约以 [OpenAPI 3.1 文件](../api/openapi/mvp-a.yaml) 为准；本文件解释边界和跨接口规则。
- 基础路径为 `/api/v1`，JSON 字段使用 `snake_case`，时间使用 UTC RFC 3339，ID 使用 UUID。
- 除公开示例和健康检查外均需登录；API 从会话/JWT 获取 `actor_id`，不接受客户端传入用户 ID。
- 每个响应返回 `X-Request-Id`；客户端可以传入该头用于链路追踪，但服务端必须校验格式并防止覆盖内部 trace。
- 创建任务、重试、重写、导出等非幂等操作必须携带 `Idempotency-Key`。

## 快速生成

```http
POST /api/v1/quick-start/sessions
GET  /api/v1/quick-start/sessions/{session_id}
POST /api/v1/quick-start/sessions/{session_id}/files
DELETE /api/v1/quick-start/sessions/{session_id}/files/{file_id}
POST /api/v1/quick-start/sessions/{session_id}/confirm-context
POST /api/v1/quick-start/sessions/{session_id}/generate
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

`source_type=file` 时 `source_text` 可空；`text` 和 `file_with_text` 时为 10-2000 字。`preferred_outputs` 为空时按用途矩阵填充。创建会话不创建项目：

```json
{
  "session_id": "5b123e8e-790d-4f5c-bd0e-f5d6de60f18b",
  "project_id": null,
  "status": "awaiting_source",
  "next_action": "upload_file",
  "expires_at": "2026-06-28T12:00:00Z",
  "teacher_visible_message": "请上传 PPTX 或 PDF"
}
```

文本模式创建后直接返回 `source_ready`。文件通过 `multipart/form-data` 上传；MVP 限单文件 50 MiB、单会话 1 个主文件。文件校验完成前状态为 `source_validating`。

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
  "task_id": "c9c6bd11-57ee-438a-887f-170a4bfc27a9",
  "status": "queued"
}
```

状态响应必须同时包含内部状态和教师可见信息：

```json
{
  "session_id": "5b123e8e-790d-4f5c-bd0e-f5d6de60f18b",
  "project_id": "5965a453-a9e5-45f0-a8bb-66202c4d44cb",
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
DELETE /api/v1/projects/{project_id}
GET    /api/v1/projects/{project_id}/files
GET    /api/v1/files/{file_id}/parsed-result

GET  /api/v1/projects/{project_id}/tasks
GET  /api/v1/tasks/{task_id}
POST /api/v1/tasks/{task_id}/retry
POST /api/v1/tasks/{task_id}/cancel

GET /api/v1/projects/{project_id}/outputs
GET /api/v1/outputs/{output_id}
GET /api/v1/outputs/{output_id}/versions
POST /api/v1/outputs/{output_id}/versions/{version}/restore
```

MVP-A 不提供空白项目创建接口；项目只由快速生成流程创建。`DELETE /projects/{id}` 是软删除并触发异步数据清理。

## 内容块编辑与重写

```http
GET   /api/v1/outputs/{output_id}/blocks
GET   /api/v1/content-blocks/{block_id}
PATCH /api/v1/content-blocks/{block_id}
POST  /api/v1/content-blocks/{block_id}/regenerate
GET   /api/v1/content-blocks/{block_id}/versions
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

## 审核

```http
GET  /api/v1/projects/{project_id}/review-findings
POST /api/v1/review-findings/{finding_id}/resolve
POST /api/v1/content-blocks/{block_id}/confirm
```

MVP-A finding 类型为 `missing_source`、`low_confidence_context`、`invalid_question_structure`、`answer_format_mismatch`、`single_choice_not_unique`。`resolve` 的动作可为 `confirmed`、`fixed`、`dismissed`，必须附带当前内容版本。

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

生产环境错误不得回传栈、SQL、对象存储路径、模型凭据或原始敏感内容。

## 幂等、分页和删除

1. `Idempotency-Key` 为 UUID，结果保存 24 小时；同一 actor + route + key 唯一。
2. 列表使用 cursor 分页：`?limit=20&cursor=...`，`limit` 范围 1-100。
3. 删除项目先标记 `deleted_at`，立即禁止访问；对象文件和模型日志按数据保留策略异步清理。
4. 重试必须引用原 task，但创建新 attempt；不能清空原错误或覆盖成功产物。

## 数据落地原则

1. `quick_start_sessions` 保存首次生成输入和聚合状态，不长期承载业务内容。
2. 接受生成请求时创建 `projects`，临时文件随后绑定项目；CourseIR、输出、内容块和导出均归属项目。
3. 每次编辑、重写、恢复都创建不可变版本；当前版本由父记录指针标识。
   内容块变更同时创建一个新的 OutputVersion 清单，使整份输出可以稳定预览、导出和恢复。
4. 模型调用记录 Prompt、模型、token、成本、延迟、数据策略和失败原因，但不保存 API Key 或未脱敏的完整输入。
5. 发布、分享、模型配置和删除等高影响动作写入不可变审计日志。
