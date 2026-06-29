# TRD：API 与数据契约

## 快速生成 API

```http
POST /api/quick-start/sessions
GET /api/quick-start/sessions/{session_id}
POST /api/quick-start/sessions/{session_id}/files
POST /api/quick-start/sessions/{session_id}/confirm-context
POST /api/quick-start/sessions/{session_id}/generate
```

## 创建会话请求示例

```json
{
  "intent": "teach_tomorrow",
  "preferred_outputs": ["study_notes", "quiz", "teacher_tips", "lesson_plan_draft"],
  "optional_context": {
    "subject": null,
    "grade": null,
    "duration_minutes": 45,
    "student_level": "normal",
    "teaching_scenario": "regular_class"
  },
  "client_trace_id": "qs_20260626_001"
}
```

## 创建会话响应示例

```json
{
  "session_id": "qss_001",
  "project_id": "proj_001",
  "status": "waiting_for_file",
  "created_draft_project": true,
  "next_action": "upload_file",
  "teacher_visible_message": "请上传 PPTX 或 PDF，我们会先生成可编辑初稿"
}
```

## 状态响应要求

状态响应必须同时包含内部状态和教师可见文案：

```json
{
  "session_id": "qss_001",
  "project_id": "proj_001",
  "status": "generating",
  "current_step": "generating_quiz",
  "teacher_visible_message": "学习要点已生成，随堂小测还在生成中",
  "progress_percent": 65,
  "available_outputs": ["study_notes"],
  "detected_context": {
    "subject": { "value": "物理", "confidence": 0.94, "needs_confirmation": false },
    "grade": { "value": "高一", "confidence": 0.72, "needs_confirmation": true },
    "duration_minutes": { "value": 45, "confidence": 0.6, "needs_confirmation": true }
  },
  "recoverable_errors": []
}
```

## 完整业务接口清单

### 项目接口

```http
POST /api/projects
GET /api/projects
GET /api/projects/{project_id}
PATCH /api/projects/{project_id}
DELETE /api/projects/{project_id}
```

### 文件接口

```http
POST /api/projects/{project_id}/files
GET /api/projects/{project_id}/files
GET /api/files/{file_id}/parsed-result
DELETE /api/files/{file_id}
```

### 生成任务接口

```http
POST /api/projects/{project_id}/generate
GET /api/tasks/{task_id}
POST /api/tasks/{task_id}/retry
POST /api/tasks/{task_id}/cancel
```

### 输出结果接口

```http
GET /api/projects/{project_id}/outputs
GET /api/outputs/{output_id}
PATCH /api/outputs/{output_id}
POST /api/outputs/{output_id}/regenerate
GET /api/outputs/{output_id}/download
```

### 模型配置接口

```http
POST /api/model-providers
GET /api/model-providers
PATCH /api/model-providers/{provider_id}
DELETE /api/model-providers/{provider_id}
POST /api/model-providers/{provider_id}/test
```

### 广场与 Agent 后期接口

```http
GET /api/community/resources
GET /api/community/resources/{resource_id}
POST /api/community/resources/{resource_id}/fork
POST /api/community/resources/{resource_id}/watch
DELETE /api/community/resources/{resource_id}/watch
GET /api/community/resources/{resource_id}/lineage
POST /api/agent/sessions
POST /api/agent/sessions/{session_id}/messages
GET /api/agent/sessions/{session_id}
```

## 数据落地原则

1. `quick_start_session` 只保存首次生成会话状态、用途、可选上下文和前端 trace ID。
2. 上传成功后自动创建 `projects` 草稿记录，后续所有文件、CourseIR 和输出仍归属项目，避免产生两套业务模型。
3. 解析、识别、生成、渲染仍进入 `generation_tasks` 或队列系统；前端通过 session 聚合查询进度。
4. 每次局部重写必须创建新的 ContentBlock 版本。
5. 所有模型调用记录 prompt_key、prompt_version、模型、token、成本、延迟和失败原因。
