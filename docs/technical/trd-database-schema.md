# TRD：数据库 Schema

## 适用范围

本 Schema 覆盖 MVP-A 的快速生成、文件、CourseIR、任务、输出、内容块版本、审核、导出、模型调用、幂等和审计。SQL migration 是最终执行来源；本文件用于约束表职责和关键约束。

约定：

1. PostgreSQL，主键使用 UUID，时间使用 `TIMESTAMPTZ`。
2. 业务删除默认软删除；对象存储和日志由清理任务按保留策略物理删除。
3. 数据库保存 `storage_key`，不保存长期签名 URL。
4. JSONB 内容进入数据库前必须通过版本化 Schema 校验。
5. 所有外键字段、状态查询字段和清理时间字段建立索引。

## 核心关系

```text
app_users
  ├─ quick_start_sessions ─ project_files(temp)
  └─ projects
       ├─ project_files
       ├─ course_ir_versions
       ├─ generation_tasks
       ├─ generated_outputs ─ output_versions
       │    └─ content_blocks ─ content_block_versions
       ├─ review_findings
       └─ exports

model_providers ─ model_configs
generation_tasks/model calls/audit logs 贯穿上述对象
```

## MVP-A SQL 基线

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE app_users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_subject VARCHAR(255) NOT NULL UNIQUE,
  display_name VARCHAR(100),
  role VARCHAR(30) NOT NULL DEFAULT 'teacher'
    CHECK (role IN ('teacher', 'model_admin', 'system_admin')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);

CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_user_id UUID NOT NULL REFERENCES app_users(id),
  source_session_id UUID,
  title VARCHAR(255) NOT NULL,
  subject VARCHAR(100),
  grade VARCHAR(100),
  duration_minutes INT CHECK (duration_minutes BETWEEN 5 AND 480),
  teaching_style VARCHAR(100),
  student_level VARCHAR(30)
    CHECK (student_level IN ('basic', 'normal', 'advanced')),
  status VARCHAR(30) NOT NULL DEFAULT 'draft'
    CHECK (status IN ('draft', 'ready', 'partial', 'failed', 'deleted')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);

CREATE TABLE quick_start_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES app_users(id),
  project_id UUID UNIQUE REFERENCES projects(id),
  intent VARCHAR(40) NOT NULL
    CHECK (intent IN ('teach_tomorrow', 'student_review', 'create_quiz', 'write_lesson_plan')),
  source_type VARCHAR(30) NOT NULL
    CHECK (source_type IN ('file', 'text', 'file_with_text')),
  source_text_encrypted BYTEA,
  source_text_sha256 CHAR(64),
  preferred_outputs JSONB NOT NULL,
  context JSONB NOT NULL DEFAULT '{}'::jsonb,
  context_version INT NOT NULL DEFAULT 1 CHECK (context_version >= 1),
  client_trace_id VARCHAR(100),
  status VARCHAR(40) NOT NULL
    CHECK (status IN (
      'awaiting_source', 'source_validating', 'source_ready', 'queued', 'parsing',
      'detecting_context', 'generating', 'reviewing_quality', 'partial_success',
      'succeeded', 'failed', 'cancelled', 'expired'
    )),
  current_step VARCHAR(100),
  progress_percent INT NOT NULL DEFAULT 0 CHECK (progress_percent BETWEEN 0 AND 100),
  warning_summary JSONB NOT NULL DEFAULT '[]'::jsonb,
  error_summary JSONB NOT NULL DEFAULT '[]'::jsonb,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (
    (source_type = 'file' AND source_text_encrypted IS NULL)
    OR (source_type IN ('text', 'file_with_text') AND source_text_encrypted IS NOT NULL)
  )
);

ALTER TABLE projects
  ADD CONSTRAINT projects_source_session_fk
  FOREIGN KEY (source_session_id) REFERENCES quick_start_sessions(id);

CREATE TABLE project_files (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  quick_start_session_id UUID REFERENCES quick_start_sessions(id) ON DELETE SET NULL,
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  original_file_name VARCHAR(255) NOT NULL,
  media_type VARCHAR(150) NOT NULL,
  extension VARCHAR(20) NOT NULL,
  file_size_bytes BIGINT NOT NULL CHECK (file_size_bytes BETWEEN 1 AND 52428800),
  sha256 CHAR(64) NOT NULL,
  storage_key TEXT NOT NULL UNIQUE,
  status VARCHAR(30) NOT NULL
    CHECK (status IN ('uploaded', 'scanning', 'ready', 'rejected', 'parsing', 'parsed', 'parse_failed')),
  page_count INT CHECK (page_count BETWEEN 1 AND 200),
  malware_scan_status VARCHAR(20) NOT NULL DEFAULT 'pending'
    CHECK (malware_scan_status IN ('pending', 'clean', 'infected', 'failed')),
  parsed_result JSONB,
  parsed_schema_version VARCHAR(30),
  parse_error_code VARCHAR(100),
  parse_error_detail TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ,
  CHECK (quick_start_session_id IS NOT NULL OR project_id IS NOT NULL)
);

CREATE TABLE course_ir_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  version INT NOT NULL CHECK (version >= 1),
  schema_version VARCHAR(30) NOT NULL,
  content JSONB NOT NULL,
  source_fingerprint CHAR(64) NOT NULL,
  created_by_type VARCHAR(20) NOT NULL
    CHECK (created_by_type IN ('model', 'user', 'system', 'migration')),
  created_by_user_id UUID REFERENCES app_users(id),
  generation_task_id UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (project_id, version)
);

CREATE TABLE generation_tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  quick_start_session_id UUID REFERENCES quick_start_sessions(id) ON DELETE SET NULL,
  parent_task_id UUID REFERENCES generation_tasks(id),
  root_task_id UUID REFERENCES generation_tasks(id),
  task_type VARCHAR(40) NOT NULL
    CHECK (task_type IN (
      'parse_file', 'build_course_ir', 'generate_output', 'review_quality',
      'regenerate_block', 'render_export', 'cleanup_project'
    )),
  target_type VARCHAR(40),
  target_id UUID,
  status VARCHAR(20) NOT NULL
    CHECK (status IN ('queued', 'running', 'retrying', 'succeeded', 'failed', 'cancelled')),
  attempt INT NOT NULL DEFAULT 1 CHECK (attempt >= 1),
  max_attempts INT NOT NULL CHECK (max_attempts BETWEEN 1 AND 5),
  progress_percent INT NOT NULL DEFAULT 0 CHECK (progress_percent BETWEEN 0 AND 100),
  current_step VARCHAR(100),
  payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  idempotency_key UUID,
  worker_job_id VARCHAR(255),
  error_code VARCHAR(100),
  error_message TEXT,
  retryable BOOLEAN NOT NULL DEFAULT FALSE,
  queued_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  started_at TIMESTAMPTZ,
  finished_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (project_id, task_type, idempotency_key)
);

ALTER TABLE course_ir_versions
  ADD CONSTRAINT course_ir_generation_task_fk
  FOREIGN KEY (generation_task_id) REFERENCES generation_tasks(id) ON DELETE SET NULL;

CREATE TABLE generated_outputs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  output_type VARCHAR(40) NOT NULL
    CHECK (output_type IN ('study_notes', 'quiz', 'teacher_tips', 'lesson_plan_draft')),
  title VARCHAR(255) NOT NULL,
  current_version INT NOT NULL DEFAULT 1 CHECK (current_version >= 1),
  status VARCHAR(20) NOT NULL
    CHECK (status IN ('generating', 'ready', 'failed')),
  review_status VARCHAR(20) NOT NULL DEFAULT 'unreviewed'
    CHECK (review_status IN ('unreviewed', 'confirmed')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ,
  UNIQUE (project_id, output_type)
);

CREATE TABLE output_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  output_id UUID NOT NULL REFERENCES generated_outputs(id) ON DELETE CASCADE,
  version INT NOT NULL CHECK (version >= 1),
  course_ir_version_id UUID NOT NULL REFERENCES course_ir_versions(id),
  schema_version VARCHAR(30) NOT NULL,
  snapshot JSONB NOT NULL,
  block_manifest JSONB NOT NULL DEFAULT '[]'::jsonb,
  change_summary VARCHAR(500),
  created_by_type VARCHAR(20) NOT NULL
    CHECK (created_by_type IN ('model', 'user', 'system', 'restore')),
  created_by_user_id UUID REFERENCES app_users(id),
  generation_task_id UUID REFERENCES generation_tasks(id) ON DELETE SET NULL,
  prompt_key VARCHAR(100),
  prompt_version VARCHAR(50),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (output_id, version)
);

CREATE TABLE content_blocks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  output_id UUID NOT NULL REFERENCES generated_outputs(id) ON DELETE CASCADE,
  stable_key VARCHAR(150) NOT NULL,
  block_type VARCHAR(40) NOT NULL
    CHECK (block_type IN ('note', 'quiz', 'answer', 'teacher_tip', 'lesson_plan_section')),
  position INT NOT NULL CHECK (position >= 0),
  current_version INT NOT NULL DEFAULT 1 CHECK (current_version >= 1),
  review_status VARCHAR(20) NOT NULL DEFAULT 'unreviewed'
    CHECK (review_status IN ('unreviewed', 'confirmed')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ,
  UNIQUE (output_id, stable_key)
);

CREATE TABLE content_block_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content_block_id UUID NOT NULL REFERENCES content_blocks(id) ON DELETE CASCADE,
  version INT NOT NULL CHECK (version >= 1),
  title VARCHAR(255),
  body JSONB NOT NULL,
  source_refs JSONB NOT NULL DEFAULT '[]'::jsonb,
  knowledge_point_ids JSONB NOT NULL DEFAULT '[]'::jsonb,
  reuse_policy VARCHAR(30) NOT NULL DEFAULT 'private'
    CHECK (reuse_policy IN ('private', 'project', 'organization', 'public')),
  change_summary VARCHAR(500),
  created_by_type VARCHAR(20) NOT NULL
    CHECK (created_by_type IN ('model', 'user', 'system', 'restore')),
  created_by_user_id UUID REFERENCES app_users(id),
  generation_task_id UUID REFERENCES generation_tasks(id) ON DELETE SET NULL,
  prompt_key VARCHAR(100),
  prompt_version VARCHAR(50),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (content_block_id, version)
);

CREATE TABLE review_findings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  output_id UUID REFERENCES generated_outputs(id) ON DELETE CASCADE,
  content_block_id UUID REFERENCES content_blocks(id) ON DELETE CASCADE,
  block_version INT,
  finding_type VARCHAR(50) NOT NULL
    CHECK (finding_type IN (
      'missing_source', 'low_confidence_context', 'invalid_question_structure',
      'answer_format_mismatch', 'single_choice_not_unique'
    )),
  severity VARCHAR(20) NOT NULL CHECK (severity IN ('info', 'warning', 'blocking')),
  status VARCHAR(20) NOT NULL DEFAULT 'open'
    CHECK (status IN ('open', 'confirmed', 'fixed', 'dismissed')),
  message TEXT NOT NULL,
  details JSONB NOT NULL DEFAULT '{}'::jsonb,
  source_refs JSONB NOT NULL DEFAULT '[]'::jsonb,
  resolved_by_user_id UUID REFERENCES app_users(id),
  resolution_note VARCHAR(500),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at TIMESTAMPTZ,
  CHECK (content_block_id IS NULL OR block_version IS NOT NULL)
);

CREATE TABLE exports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  requested_by_user_id UUID NOT NULL REFERENCES app_users(id),
  generation_task_id UUID REFERENCES generation_tasks(id) ON DELETE SET NULL,
  format VARCHAR(20) NOT NULL CHECK (format IN ('pdf', 'docx', 'markdown')),
  status VARCHAR(20) NOT NULL
    CHECK (status IN ('queued', 'rendering', 'succeeded', 'failed', 'expired')),
  output_manifest JSONB NOT NULL,
  template_key VARCHAR(100) NOT NULL,
  template_version INT NOT NULL CHECK (template_version >= 1),
  storage_key TEXT UNIQUE,
  file_name VARCHAR(255),
  file_size_bytes BIGINT CHECK (file_size_bytes >= 0),
  sha256 CHAR(64),
  error_code VARCHAR(100),
  error_message TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ
);

CREATE TABLE model_providers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  display_name VARCHAR(100) NOT NULL,
  provider_type VARCHAR(40) NOT NULL
    CHECK (provider_type IN ('openai_compatible', 'azure_openai', 'anthropic', 'gemini', 'local')),
  base_url TEXT,
  auth_type VARCHAR(30) NOT NULL
    CHECK (auth_type IN ('api_key', 'workload_identity', 'none')),
  credential_ciphertext BYTEA,
  credential_key_version VARCHAR(100),
  credential_last_four CHAR(4),
  enabled BOOLEAN NOT NULL DEFAULT TRUE,
  created_by_user_id UUID NOT NULL REFERENCES app_users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ,
  CHECK (
    (auth_type = 'api_key' AND credential_ciphertext IS NOT NULL)
    OR (auth_type <> 'api_key' AND credential_ciphertext IS NULL)
  )
);

CREATE TABLE model_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider_id UUID NOT NULL REFERENCES model_providers(id) ON DELETE CASCADE,
  display_name VARCHAR(100) NOT NULL,
  model_name VARCHAR(150) NOT NULL,
  capability JSONB NOT NULL,
  routing_priority INT NOT NULL DEFAULT 100,
  max_output_tokens INT CHECK (max_output_tokens > 0),
  temperature NUMERIC(4,3) CHECK (temperature BETWEEN 0 AND 2),
  enabled BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (provider_id, model_name)
);

CREATE TABLE model_call_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES app_users(id) ON DELETE SET NULL,
  project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
  generation_task_id UUID REFERENCES generation_tasks(id) ON DELETE SET NULL,
  model_config_id UUID REFERENCES model_configs(id) ON DELETE SET NULL,
  request_id VARCHAR(100) NOT NULL,
  task_type VARCHAR(100) NOT NULL,
  prompt_key VARCHAR(100) NOT NULL,
  prompt_version VARCHAR(50) NOT NULL,
  input_fingerprint CHAR(64),
  prompt_tokens INT CHECK (prompt_tokens >= 0),
  completion_tokens INT CHECK (completion_tokens >= 0),
  estimated_cost NUMERIC(14,6) CHECK (estimated_cost >= 0),
  currency CHAR(3),
  latency_ms INT CHECK (latency_ms >= 0),
  success BOOLEAN NOT NULL,
  error_code VARCHAR(100),
  error_message_redacted TEXT,
  data_policy VARCHAR(50) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE idempotency_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  actor_user_id UUID NOT NULL REFERENCES app_users(id),
  route_key VARCHAR(200) NOT NULL,
  idempotency_key UUID NOT NULL,
  request_fingerprint CHAR(64) NOT NULL,
  response_status INT,
  response_body JSONB,
  resource_type VARCHAR(50),
  resource_id UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at TIMESTAMPTZ NOT NULL,
  UNIQUE (actor_user_id, route_key, idempotency_key)
);

CREATE TABLE outbox_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_type VARCHAR(50) NOT NULL,
  aggregate_id UUID NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'published', 'failed')),
  attempt INT NOT NULL DEFAULT 0 CHECK (attempt >= 0),
  available_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  actor_user_id UUID REFERENCES app_users(id) ON DELETE SET NULL,
  actor_type VARCHAR(20) NOT NULL CHECK (actor_type IN ('user', 'worker', 'system')),
  action VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50) NOT NULL,
  resource_id UUID,
  project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
  request_id VARCHAR(100) NOT NULL,
  source_ip_hash CHAR(64),
  user_agent_hash CHAR(64),
  before_summary JSONB,
  after_summary JSONB,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 必需索引

```sql
CREATE INDEX idx_projects_owner_updated
  ON projects (owner_user_id, updated_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_quick_start_user_created
  ON quick_start_sessions (user_id, created_at DESC);
CREATE INDEX idx_quick_start_expiry
  ON quick_start_sessions (expires_at) WHERE status IN ('awaiting_source', 'source_ready');
CREATE INDEX idx_project_files_project ON project_files (project_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_project_files_session ON project_files (quick_start_session_id) WHERE project_id IS NULL;
CREATE INDEX idx_tasks_project_created ON generation_tasks (project_id, created_at DESC);
CREATE INDEX idx_tasks_runnable ON generation_tasks (status, queued_at)
  WHERE status IN ('queued', 'retrying');
CREATE INDEX idx_outputs_project ON generated_outputs (project_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_blocks_output_position ON content_blocks (output_id, position) WHERE deleted_at IS NULL;
CREATE INDEX idx_findings_project_status ON review_findings (project_id, status, severity);
CREATE INDEX idx_exports_project_created ON exports (project_id, created_at DESC);
CREATE INDEX idx_model_calls_project_created ON model_call_logs (project_id, created_at DESC);
CREATE INDEX idx_outbox_pending ON outbox_events (available_at) WHERE status = 'pending';
CREATE INDEX idx_audit_resource ON audit_logs (resource_type, resource_id, created_at DESC);
CREATE INDEX idx_idempotency_expiry ON idempotency_records (expires_at);
```

## 事务与并发规则

1. 接受快速生成时锁定 `quick_start_sessions FOR UPDATE`，在同一事务创建 project、更新临时文件归属、创建 root task、写 outbox 和幂等记录。
2. 更新内容块时以 `WHERE id = ? AND current_version = ?` 做乐观锁；插入 ContentBlockVersion、创建引用新块版本的 OutputVersion，并更新两个当前版本指针必须在同一事务。
3. `generated_outputs.current_version` 与 `content_blocks.current_version` 是缓存指针；不可变版本表才是历史来源。
4. 恢复历史版本会创建新版本，不把当前指针直接改回旧数字。
5. Worker 通过 task ID 和目标版本保证幂等；重复执行只能返回已有结果。
6. 审计日志不保存完整正文、文件内容、凭据或未经脱敏的模型输入。

## 项目初始标题

项目在点击生成时创建。如果尚未识别课程名，标题使用“未命名备课 YYYY-MM-DD HH:mm”；识别成功后仅在用户未编辑过标题时自动替换，避免覆盖教师输入。

## MVP-B 扩展：学生分享

以下表只在 MVP-B migration 中创建：

```sql
CREATE TABLE share_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  created_by_user_id UUID NOT NULL REFERENCES app_users(id),
  token_hash CHAR(64) NOT NULL UNIQUE,
  output_manifest JSONB NOT NULL,
  status VARCHAR(20) NOT NULL CHECK (status IN ('active', 'revoked', 'expired')),
  expires_at TIMESTAMPTZ NOT NULL,
  revoked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

只向用户返回一次原始随机 token，数据库仅保存 SHA-256/HMAC 结果；撤回后立即失效。

## 后期生态表

`organizations`、`community_resources`、`resource_relations`、`agent_sessions`、向量索引和评论举报不属于 MVP-A。它们应在对应阶段通过独立 ADR、权限模型和 migration 引入，不能先以无租户约束的草表进入生产数据库。
