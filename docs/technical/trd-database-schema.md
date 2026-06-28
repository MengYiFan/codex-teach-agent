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
       ├─ project_files ─ project_assets
       ├─ project_text_sources
       ├─ course_ir_versions
       ├─ generation_runs ─ generation_tasks ─ generation_task_attempts
       ├─ generated_outputs ─ output_versions ─ output_version_blocks
       │    └─ content_blocks ─ content_block_versions
       ├─ review_findings
       ├─ exports
       ├─ quality_feedback
       └─ deletion_operations

model_providers ─ model_configs
analytics_events/model calls/audit logs 贯穿上述对象
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
  title_source VARCHAR(20) NOT NULL DEFAULT 'placeholder'
    CHECK (title_source IN ('placeholder', 'detected', 'user')),
  subject VARCHAR(100),
  grade VARCHAR(100),
  duration_minutes INT CHECK (duration_minutes BETWEEN 5 AND 480),
  teaching_style VARCHAR(100),
  student_level VARCHAR(30)
    CHECK (student_level IN ('basic', 'normal', 'advanced')),
  context JSONB NOT NULL DEFAULT '{}'::jsonb,
  context_version INT NOT NULL DEFAULT 1 CHECK (context_version >= 1),
  status VARCHAR(30) NOT NULL DEFAULT 'draft'
    CHECK (status IN ('draft', 'generating', 'ready', 'partial', 'failed', 'deleted')),
  version INT NOT NULL DEFAULT 1 CHECK (version >= 1),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ,
  CHECK (jsonb_typeof(context) = 'object'),
  CHECK ((status = 'deleted') = (deleted_at IS NOT NULL))
);

CREATE TABLE quick_start_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES app_users(id),
  project_id UUID UNIQUE REFERENCES projects(id) ON DELETE SET NULL,
  intent VARCHAR(40) NOT NULL
    CHECK (intent IN ('teach_tomorrow', 'student_review', 'create_quiz', 'write_lesson_plan')),
  source_type VARCHAR(30) NOT NULL
    CHECK (source_type IN ('file', 'text', 'file_with_text')),
  source_text_encrypted BYTEA,
  source_text_key_version VARCHAR(100),
  source_text_hmac CHAR(64),
  source_text_hmac_key_version VARCHAR(100),
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
    (
      source_type = 'file'
      AND source_text_encrypted IS NULL
      AND source_text_key_version IS NULL
      AND source_text_hmac IS NULL
      AND source_text_hmac_key_version IS NULL
    )
    OR (
      source_type IN ('text', 'file_with_text')
      AND source_text_encrypted IS NOT NULL
      AND source_text_key_version IS NOT NULL
      AND source_text_hmac IS NOT NULL
      AND source_text_hmac_key_version IS NOT NULL
    )
  ),
  CHECK (jsonb_typeof(preferred_outputs) = 'array'),
  CHECK (jsonb_array_length(preferred_outputs) BETWEEN 1 AND 4),
  CHECK (jsonb_typeof(context) = 'object')
);

ALTER TABLE projects
  ADD CONSTRAINT projects_source_session_fk
  FOREIGN KEY (source_session_id) REFERENCES quick_start_sessions(id) ON DELETE SET NULL;

CREATE TABLE project_files (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  quick_start_session_id UUID REFERENCES quick_start_sessions(id) ON DELETE CASCADE,
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
  CHECK ((quick_start_session_id IS NOT NULL) <> (project_id IS NOT NULL))
);

CREATE TABLE project_text_sources (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  source_role VARCHAR(20) NOT NULL CHECK (source_role IN ('primary', 'supplement')),
  source_text_ciphertext BYTEA NOT NULL,
  data_key_version VARCHAR(100) NOT NULL,
  source_text_hmac CHAR(64) NOT NULL,
  fingerprint_key_version VARCHAR(100) NOT NULL,
  character_count INT NOT NULL CHECK (character_count BETWEEN 10 AND 2000),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ,
  UNIQUE (project_id, source_role)
);

CREATE TABLE project_assets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  source_file_id UUID NOT NULL REFERENCES project_files(id) ON DELETE CASCADE,
  page_index INT NOT NULL CHECK (page_index BETWEEN 1 AND 200),
  asset_type VARCHAR(20) NOT NULL CHECK (asset_type IN ('image', 'table_snapshot')),
  media_type VARCHAR(100) NOT NULL,
  storage_key TEXT NOT NULL UNIQUE,
  file_size_bytes BIGINT NOT NULL CHECK (file_size_bytes > 0),
  sha256 CHAR(64) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
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
  model_config_id UUID,
  provider_type_snapshot VARCHAR(40)
    CHECK (provider_type_snapshot IN ('openai_compatible', 'azure_openai', 'anthropic', 'gemini', 'local')),
  model_name_snapshot VARCHAR(150),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (project_id, version),
  CHECK (
    created_by_type <> 'model'
    OR (
      generation_task_id IS NOT NULL
      AND provider_type_snapshot IS NOT NULL
      AND model_name_snapshot IS NOT NULL
    )
  )
);

CREATE TABLE generation_tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  quick_start_session_id UUID REFERENCES quick_start_sessions(id) ON DELETE SET NULL,
  generation_run_id UUID,
  parent_task_id UUID REFERENCES generation_tasks(id),
  root_task_id UUID REFERENCES generation_tasks(id),
  retry_of_task_id UUID REFERENCES generation_tasks(id),
  task_type VARCHAR(40) NOT NULL
    CHECK (task_type IN (
      'generation_run', 'parse_file', 'build_course_ir', 'generate_output', 'review_quality',
      'regenerate_block', 'render_export', 'cleanup_project'
    )),
  target_type VARCHAR(40)
    CHECK (target_type IN ('content_block', 'output', 'export', 'file', 'project')),
  target_id UUID,
  output_type VARCHAR(40)
    CHECK (output_type IN ('study_notes', 'quiz', 'teacher_tips', 'lesson_plan_draft')),
  status VARCHAR(20) NOT NULL
    CHECK (status IN ('queued', 'running', 'retrying', 'succeeded', 'failed', 'dead_lettered', 'cancelled')),
  attempt_count INT NOT NULL DEFAULT 0 CHECK (attempt_count >= 0),
  max_attempts INT NOT NULL CHECK (max_attempts BETWEEN 1 AND 5),
  progress_percent INT NOT NULL DEFAULT 0 CHECK (progress_percent BETWEEN 0 AND 100),
  current_step VARCHAR(100),
  payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  idempotency_key UUID,
  error_code VARCHAR(100),
  error_message TEXT,
  retryable BOOLEAN NOT NULL DEFAULT FALSE,
  queued_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  started_at TIMESTAMPTZ,
  finished_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (project_id, task_type, idempotency_key),
  CHECK (
    task_type <> 'regenerate_block'
    OR (target_type = 'content_block' AND target_id IS NOT NULL)
  ),
  CHECK (
    task_type NOT IN ('generation_run', 'parse_file', 'build_course_ir', 'generate_output', 'review_quality')
    OR generation_run_id IS NOT NULL
  ),
  CHECK (task_type <> 'generate_output' OR output_type IS NOT NULL)
);

CREATE TABLE generation_runs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  quick_start_session_id UUID REFERENCES quick_start_sessions(id) ON DELETE SET NULL,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  run_kind VARCHAR(30) NOT NULL
    CHECK (run_kind IN ('initial', 'retry_failed_outputs', 'regenerate_context')),
  retry_of_run_id UUID REFERENCES generation_runs(id),
  context_version INT NOT NULL CHECK (context_version >= 1),
  context_snapshot JSONB NOT NULL,
  requested_outputs JSONB NOT NULL,
  status VARCHAR(30) NOT NULL
    CHECK (status IN ('queued', 'running', 'succeeded', 'partial_success', 'failed', 'cancelled')),
  root_task_id UUID UNIQUE,
  requested_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  first_meaningful_output_ready_at TIMESTAMPTZ,
  settled_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (jsonb_typeof(requested_outputs) = 'array'),
  CHECK (jsonb_array_length(requested_outputs) BETWEEN 1 AND 4),
  CHECK (jsonb_typeof(context_snapshot) = 'object')
);

ALTER TABLE generation_tasks
  ADD CONSTRAINT generation_tasks_run_fk
  FOREIGN KEY (generation_run_id) REFERENCES generation_runs(id) ON DELETE CASCADE;

ALTER TABLE generation_runs
  ADD CONSTRAINT generation_runs_root_task_fk
  FOREIGN KEY (root_task_id) REFERENCES generation_tasks(id) ON DELETE SET NULL;

ALTER TABLE quick_start_sessions
  ADD COLUMN active_generation_run_id UUID REFERENCES generation_runs(id) ON DELETE SET NULL,
  ADD COLUMN latest_generation_run_id UUID REFERENCES generation_runs(id) ON DELETE SET NULL;

CREATE TABLE generation_task_attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  task_id UUID NOT NULL REFERENCES generation_tasks(id) ON DELETE CASCADE,
  attempt_number INT NOT NULL CHECK (attempt_number >= 1),
  worker_job_id VARCHAR(255) NOT NULL UNIQUE,
  status VARCHAR(20) NOT NULL
    CHECK (status IN ('running', 'succeeded', 'failed', 'cancelled')),
  error_code VARCHAR(100),
  error_message TEXT,
  started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  finished_at TIMESTAMPTZ,
  UNIQUE (task_id, attempt_number)
);

ALTER TABLE course_ir_versions
  ADD CONSTRAINT course_ir_generation_task_fk
  FOREIGN KEY (generation_task_id) REFERENCES generation_tasks(id);

CREATE TABLE generated_outputs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  output_type VARCHAR(40) NOT NULL
    CHECK (output_type IN ('study_notes', 'quiz', 'teacher_tips', 'lesson_plan_draft')),
  title VARCHAR(255) NOT NULL,
  current_version INT NOT NULL DEFAULT 1 CHECK (current_version >= 1),
  status VARCHAR(20) NOT NULL
    CHECK (status IN ('generating', 'ready', 'failed')),
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
  review_status VARCHAR(20) NOT NULL DEFAULT 'unreviewed'
    CHECK (review_status IN ('unreviewed', 'confirmed')),
  change_summary VARCHAR(500),
  created_by_type VARCHAR(20) NOT NULL
    CHECK (created_by_type IN ('model', 'user', 'system', 'restore')),
  created_by_user_id UUID REFERENCES app_users(id),
  generation_task_id UUID REFERENCES generation_tasks(id),
  model_config_id UUID,
  provider_type_snapshot VARCHAR(40)
    CHECK (provider_type_snapshot IN ('openai_compatible', 'azure_openai', 'anthropic', 'gemini', 'local')),
  model_name_snapshot VARCHAR(150),
  prompt_key VARCHAR(100),
  prompt_version VARCHAR(50),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (output_id, version),
  CHECK (
    created_by_type <> 'model'
    OR (
      generation_task_id IS NOT NULL
      AND provider_type_snapshot IS NOT NULL
      AND model_name_snapshot IS NOT NULL
      AND prompt_key IS NOT NULL
      AND prompt_version IS NOT NULL
    )
  )
);

CREATE TABLE content_blocks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  output_id UUID NOT NULL REFERENCES generated_outputs(id) ON DELETE CASCADE,
  stable_key VARCHAR(150) NOT NULL,
  block_type VARCHAR(40) NOT NULL
    CHECK (block_type IN ('note', 'quiz', 'answer', 'teacher_tip', 'lesson_plan_section')),
  position INT NOT NULL CHECK (position >= 0),
  current_version INT NOT NULL DEFAULT 1 CHECK (current_version >= 1),
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
  body JSONB,
  source_refs JSONB NOT NULL DEFAULT '[]'::jsonb,
  knowledge_point_ids JSONB NOT NULL DEFAULT '[]'::jsonb,
  reuse_policy VARCHAR(30) NOT NULL DEFAULT 'private'
    CHECK (reuse_policy IN ('private', 'project')),
  review_status VARCHAR(20) NOT NULL DEFAULT 'unreviewed'
    CHECK (review_status IN ('unreviewed', 'confirmed')),
  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
  change_summary VARCHAR(500),
  created_by_type VARCHAR(20) NOT NULL
    CHECK (created_by_type IN ('model', 'user', 'system', 'restore')),
  created_by_user_id UUID REFERENCES app_users(id),
  generation_task_id UUID REFERENCES generation_tasks(id),
  model_config_id UUID,
  provider_type_snapshot VARCHAR(40)
    CHECK (provider_type_snapshot IN ('openai_compatible', 'azure_openai', 'anthropic', 'gemini', 'local')),
  model_name_snapshot VARCHAR(150),
  prompt_key VARCHAR(100),
  prompt_version VARCHAR(50),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (content_block_id, version),
  CHECK ((is_deleted AND body IS NULL) OR (NOT is_deleted AND body IS NOT NULL)),
  CHECK (
    created_by_type <> 'model'
    OR (
      generation_task_id IS NOT NULL
      AND provider_type_snapshot IS NOT NULL
      AND model_name_snapshot IS NOT NULL
      AND prompt_key IS NOT NULL
      AND prompt_version IS NOT NULL
    )
  )
);

CREATE TABLE output_version_blocks (
  output_version_id UUID NOT NULL REFERENCES output_versions(id) ON DELETE CASCADE,
  content_block_version_id UUID NOT NULL REFERENCES content_block_versions(id),
  position INT NOT NULL CHECK (position >= 0),
  PRIMARY KEY (output_version_id, position),
  UNIQUE (output_version_id, content_block_version_id)
);

CREATE TABLE review_findings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  output_id UUID REFERENCES generated_outputs(id) ON DELETE CASCADE,
  content_block_id UUID REFERENCES content_blocks(id) ON DELETE CASCADE,
  block_version INT,
  context_version INT CHECK (context_version >= 1),
  finding_type VARCHAR(50) NOT NULL
    CHECK (finding_type IN (
      'missing_source', 'low_confidence_context', 'invalid_question_structure',
      'answer_format_mismatch', 'single_choice_not_unique'
    )),
  severity VARCHAR(20) NOT NULL CHECK (severity IN ('info', 'warning', 'blocking')),
  status VARCHAR(20) NOT NULL DEFAULT 'open'
    CHECK (status IN ('open', 'confirmed', 'fixed', 'dismissed', 'superseded')),
  version INT NOT NULL DEFAULT 1 CHECK (version >= 1),
  message TEXT NOT NULL,
  details JSONB NOT NULL DEFAULT '{}'::jsonb,
  source_refs JSONB NOT NULL DEFAULT '[]'::jsonb,
  resolved_by_user_id UUID REFERENCES app_users(id),
  resolution_note VARCHAR(500),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at TIMESTAMPTZ,
  CHECK (
    (content_block_id IS NOT NULL AND block_version IS NOT NULL AND context_version IS NULL)
    OR (content_block_id IS NULL AND block_version IS NULL AND context_version IS NOT NULL)
  ),
  CHECK (status <> 'dismissed' OR resolution_note IS NOT NULL)
);

CREATE TABLE exports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  requested_by_user_id UUID NOT NULL REFERENCES app_users(id),
  generation_task_id UUID REFERENCES generation_tasks(id) ON DELETE SET NULL,
  format VARCHAR(20) NOT NULL CHECK (format IN ('pdf', 'docx', 'markdown')),
  status VARCHAR(20) NOT NULL
    CHECK (status IN ('queued', 'rendering', 'succeeded', 'failed', 'expired')),
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

CREATE TABLE export_output_versions (
  export_id UUID NOT NULL REFERENCES exports(id) ON DELETE CASCADE,
  output_version_id UUID NOT NULL REFERENCES output_versions(id),
  position INT NOT NULL CHECK (position >= 0),
  PRIMARY KEY (export_id, position),
  UNIQUE (export_id, output_version_id)
);

CREATE TABLE quality_feedback (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  submitted_by_user_id UUID NOT NULL REFERENCES app_users(id),
  usability_rating VARCHAR(20) NOT NULL
    CHECK (usability_rating IN ('direct_use', 'minor_edit', 'major_edit', 'unusable')),
  before_first_export BOOLEAN NOT NULL,
  version INT NOT NULL DEFAULT 1 CHECK (version >= 1),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  superseded_at TIMESTAMPTZ,
  UNIQUE (project_id, submitted_by_user_id, version)
);

CREATE TABLE quality_feedback_outputs (
  quality_feedback_id UUID NOT NULL REFERENCES quality_feedback(id) ON DELETE CASCADE,
  output_version_id UUID NOT NULL REFERENCES output_versions(id),
  PRIMARY KEY (quality_feedback_id, output_version_id)
);

CREATE TABLE deletion_operations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id),
  requested_by_user_id UUID NOT NULL REFERENCES app_users(id),
  cleanup_task_id UUID REFERENCES generation_tasks(id) ON DELETE SET NULL,
  status VARCHAR(30) NOT NULL
    CHECK (status IN ('queued', 'running', 'online_deleted', 'completed', 'failed', 'dead_lettered')),
  online_objects_deleted_at TIMESTAMPTZ,
  model_details_deleted_at TIMESTAMPTZ,
  backup_tombstone_recorded_at TIMESTAMPTZ,
  error_code VARCHAR(100),
  requested_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ NOT NULL,
  UNIQUE (project_id)
);

CREATE TABLE analytics_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL UNIQUE,
  event_name VARCHAR(100) NOT NULL,
  event_source VARCHAR(20) NOT NULL CHECK (event_source IN ('client', 'server')),
  subject_key CHAR(64),
  visit_session_id UUID,
  quick_start_session_id UUID REFERENCES quick_start_sessions(id) ON DELETE SET NULL,
  generation_run_id UUID REFERENCES generation_runs(id) ON DELETE SET NULL,
  project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
  properties JSONB NOT NULL DEFAULT '{}'::jsonb,
  event_fingerprint CHAR(64) NOT NULL,
  occurred_at TIMESTAMPTZ NOT NULL,
  received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at TIMESTAMPTZ NOT NULL
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
  version INT NOT NULL DEFAULT 1 CHECK (version >= 1),
  created_by_user_id UUID NOT NULL REFERENCES app_users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ,
  CHECK (
    (
      auth_type = 'api_key'
      AND (
        (deleted_at IS NULL AND credential_ciphertext IS NOT NULL)
        OR (deleted_at IS NOT NULL AND credential_ciphertext IS NULL)
      )
    )
    OR (auth_type <> 'api_key' AND credential_ciphertext IS NULL)
  ),
  CHECK (
    provider_type NOT IN ('openai_compatible', 'azure_openai', 'local')
    OR base_url IS NOT NULL
  )
);

CREATE TABLE model_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider_id UUID NOT NULL REFERENCES model_providers(id) ON DELETE CASCADE,
  display_name VARCHAR(100) NOT NULL,
  model_name VARCHAR(150) NOT NULL,
  capability JSONB NOT NULL,
  routing_priority INT NOT NULL DEFAULT 100 CHECK (routing_priority BETWEEN 1 AND 1000),
  max_output_tokens INT CHECK (max_output_tokens > 0),
  temperature NUMERIC(4,3) CHECK (temperature BETWEEN 0 AND 2),
  enabled BOOLEAN NOT NULL DEFAULT TRUE,
  version INT NOT NULL DEFAULT 1 CHECK (version >= 1),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (provider_id, model_name)
);

ALTER TABLE course_ir_versions
  ADD CONSTRAINT course_ir_model_config_fk
  FOREIGN KEY (model_config_id) REFERENCES model_configs(id) ON DELETE SET NULL;
ALTER TABLE output_versions
  ADD CONSTRAINT output_versions_model_config_fk
  FOREIGN KEY (model_config_id) REFERENCES model_configs(id) ON DELETE SET NULL;
ALTER TABLE content_block_versions
  ADD CONSTRAINT block_versions_model_config_fk
  FOREIGN KEY (model_config_id) REFERENCES model_configs(id) ON DELETE SET NULL;

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
  deduplication_key VARCHAR(255) NOT NULL UNIQUE,
  status VARCHAR(20) NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'published', 'retrying', 'dead_lettered')),
  attempt INT NOT NULL DEFAULT 0 CHECK (attempt >= 0),
  max_attempts INT NOT NULL DEFAULT 5 CHECK (max_attempts BETWEEN 1 AND 20),
  last_error_code VARCHAR(100),
  last_error_message TEXT,
  locked_at TIMESTAMPTZ,
  locked_by VARCHAR(100),
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
  ON projects (owner_user_id, updated_at DESC, id DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_quick_start_user_created
  ON quick_start_sessions (user_id, created_at DESC);
CREATE INDEX idx_quick_start_expiry
  ON quick_start_sessions (expires_at);
CREATE INDEX idx_project_files_project ON project_files (project_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_project_files_session ON project_files (quick_start_session_id) WHERE project_id IS NULL;
CREATE INDEX idx_project_text_sources_project ON project_text_sources (project_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_project_assets_file_page ON project_assets (source_file_id, page_index) WHERE deleted_at IS NULL;
CREATE INDEX idx_tasks_project_created ON generation_tasks (project_id, created_at DESC, id DESC);
CREATE INDEX idx_runs_project_requested ON generation_runs (project_id, requested_at DESC, id DESC);
CREATE INDEX idx_tasks_runnable ON generation_tasks (status, queued_at)
  WHERE status IN ('queued', 'retrying');
CREATE UNIQUE INDEX idx_one_active_run_per_session
  ON generation_runs (quick_start_session_id)
  WHERE quick_start_session_id IS NOT NULL AND status IN ('queued', 'running');
CREATE UNIQUE INDEX idx_one_active_run_per_project
  ON generation_runs (project_id)
  WHERE status IN ('queued', 'running');
CREATE UNIQUE INDEX idx_one_active_block_regeneration
  ON generation_tasks (target_id)
  WHERE task_type = 'regenerate_block' AND status IN ('queued', 'running', 'retrying');
CREATE UNIQUE INDEX idx_output_version_task_result
  ON output_versions (output_id, generation_task_id) WHERE generation_task_id IS NOT NULL;
CREATE UNIQUE INDEX idx_block_version_task_result
  ON content_block_versions (content_block_id, generation_task_id) WHERE generation_task_id IS NOT NULL;
CREATE UNIQUE INDEX idx_course_ir_task_result
  ON course_ir_versions (project_id, generation_task_id) WHERE generation_task_id IS NOT NULL;
CREATE UNIQUE INDEX idx_export_task_result
  ON exports (generation_task_id) WHERE generation_task_id IS NOT NULL;
CREATE INDEX idx_outputs_project ON generated_outputs (project_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_output_versions_history ON output_versions (output_id, version DESC);
CREATE INDEX idx_blocks_output_position ON content_blocks (output_id, position, id) WHERE deleted_at IS NULL;
CREATE INDEX idx_block_versions_history ON content_block_versions (content_block_id, version DESC);
CREATE INDEX idx_findings_project_status
  ON review_findings (project_id, status, created_at DESC, id DESC);
CREATE INDEX idx_exports_project_created ON exports (project_id, created_at DESC);
CREATE INDEX idx_model_calls_project_created ON model_call_logs (project_id, created_at DESC);
CREATE INDEX idx_model_configs_provider_created ON model_configs (provider_id, created_at DESC, id DESC);
CREATE INDEX idx_model_providers_created ON model_providers (created_at DESC, id DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_outbox_pending ON outbox_events (available_at)
  WHERE status IN ('pending', 'retrying');
CREATE INDEX idx_audit_resource ON audit_logs (resource_type, resource_id, created_at DESC);
CREATE INDEX idx_idempotency_expiry ON idempotency_records (expires_at);
CREATE INDEX idx_deletion_status ON deletion_operations (status, requested_at);
CREATE INDEX idx_analytics_subject_time ON analytics_events (subject_key, occurred_at);
CREATE INDEX idx_analytics_expiry ON analytics_events (expires_at);
```

## 事务与并发规则

1. 接受快速生成时锁定 `quick_start_sessions FOR UPDATE`，在同一事务创建 project、generation run、root task、更新临时文件归属、写 outbox 和幂等记录，并把会话 `expires_at` 更新为 30 天后的诊断清理时间。
2. 显式重试或上下文重生成锁定会话（存在时）和 project；活动 run 的部分唯一索引阻止同一项目并行创建第二轮。
3. 更新内容块时先锁定 `generated_outputs FOR UPDATE`，再按 UUID 顺序锁定涉及的内容块；锁内重新读取最新 manifest 和 `expected_version`，插入 ContentBlockVersion、创建下一 OutputVersion 并更新缓存指针。
4. 更新项目标题使用 `WHERE id = ? AND version = ?`，成功后递增 project version 并设置 `title_source=user`；自动识别标题只能在 `title_source=placeholder` 且 project version 匹配时更新。课程上下文只能通过确认接口更新，在同一事务递增 `context_version`，同步查询投影字段，并按 `apply_mode` 决定是否创建 run。
5. `generated_outputs.current_version` 与 `content_blocks.current_version` 是缓存指针；不可变版本表才是历史来源。审核状态从具体版本投影，不单独修改父记录。
6. 删除创建 `is_deleted=true` 的 tombstone 版本并从新 manifest 移除该块；恢复复制历史内容为新的 `unreviewed` 版本，不能把当前指针直接回拨。
7. Worker 锁定逻辑 task，使用 task/result 唯一索引检查已有结果，并核对 payload 中的 `base_output_version`/目标块版本；版本已推进时以 `VERSION_CONFLICT` 结束，不移动 current 指针。在同一事务提交任务产物和成功状态；数据库提交后才能 ACK。重复执行返回已有结果，不再次发出业务事件。
8. outbox publisher 使用 `FOR UPDATE SKIP LOCKED` 和有限租约认领事件；进程崩溃后过期租约可重放。每次队列投递创建不可变 task attempt；自动重试耗尽后 task/outbox 进入 `dead_lettered`，保存最后错误并通过受控操作重放。
9. 处理 finding 时使用 `WHERE id = ? AND version = ?` 锁定 finding；块级 finding 还必须匹配当前 block version，项目级 finding 匹配 context version而不虚构 block version。创建新块/上下文版本时，旧版本 open finding 递增版本并转为 `superseded`，同时写审计日志。
10. migration 必须实现可延迟 constraint trigger：project asset 与 source file 属于同一 project；run 的 `retry_of_run_id` 属于同一 project；task 的 parent/root/retry 属于同一 project 且生成子任务属于同一 run；`output_version_blocks` 只能引用同一 output 的块版本；`export_output_versions` 和 `quality_feedback_outputs` 只能引用同一 project 的 OutputVersion。API 权限校验不能替代数据库关系约束。
11. 创建导出或质量反馈时，在同一事务写父记录和版本关联行；后续 API 返回的 manifest 从关联表投影，不保存第二份可漂移 JSON 清单。
12. task/outbox payload 只保存资源 ID、目标版本、操作类型和受控参数，不复制文件、完整 CourseIR、内容正文、文本来源或凭据；Worker 按 ID 在权限边界内读取。
13. 审计日志和 analytics properties 不保存完整正文、文件内容、凭据或未经脱敏的模型输入。

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
  status VARCHAR(20) NOT NULL CHECK (status IN ('active', 'revoked', 'expired')),
  expires_at TIMESTAMPTZ NOT NULL,
  revoked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE share_link_output_versions (
  share_link_id UUID NOT NULL REFERENCES share_links(id) ON DELETE CASCADE,
  output_version_id UUID NOT NULL REFERENCES output_versions(id),
  position INT NOT NULL CHECK (position >= 0),
  PRIMARY KEY (share_link_id, position),
  UNIQUE (share_link_id, output_version_id)
);
```

只向用户返回一次原始随机 token，数据库仅保存 HMAC-SHA-256 结果；撤回后立即失效。MVP-B migration 同样使用 constraint trigger 保证分享清单中的 OutputVersion 属于 `share_links.project_id`。

## 后期生态表

`organizations`、`community_resources`、`resource_relations`、`agent_sessions`、向量索引和评论举报不属于 MVP-A。它们应在对应阶段通过独立 ADR、权限模型和 migration 引入，不能先以无租户约束的草表进入生产数据库。
