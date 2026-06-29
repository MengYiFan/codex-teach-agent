# TRD：数据库 Schema 草案

## 说明

本文件承接融合总稿中的核心表设计，作为后续迁移脚本和数据模型的起点。字段应随实现继续补充索引、约束、租户隔离和审计字段。

## 核心表

| 表 | 用途 |
| --- | --- |
| `projects` | 后台草稿/课程项目，教师端可表现为“这节课”或“备课草稿”。 |
| `project_files` | 上传文件记录和解析结果。 |
| `course_ir_versions` | CourseIR 版本。 |
| `generation_tasks` | 解析、生成、渲染等长任务。 |
| `generated_outputs` | 生成内容与导出文件。 |
| `model_providers` | 模型供应商配置。 |
| `model_configs` | 模型名称、能力和参数配置。 |
| `community_resources` | 后期公开或组织内共享资源。 |
| `resource_relations` | 复制、关注、谱系等资源关系。 |
| `agent_sessions` | Agent 会话。 |
| `model_call_logs` | 模型调用日志、成本和延迟。 |

## SQL 草案

```sql
CREATE TABLE projects (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  title VARCHAR(255) NOT NULL,
  subject VARCHAR(100),
  grade VARCHAR(100),
  duration_minutes INT,
  teaching_style VARCHAR(100),
  student_level VARCHAR(100),
  status VARCHAR(50),
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);

CREATE TABLE project_files (
  id UUID PRIMARY KEY,
  project_id UUID REFERENCES projects(id),
  file_name VARCHAR(255) NOT NULL,
  file_type VARCHAR(50),
  file_size BIGINT,
  storage_url TEXT,
  parse_status VARCHAR(50),
  parsed_result JSONB,
  error_message TEXT,
  created_at TIMESTAMP NOT NULL
);

CREATE TABLE course_ir_versions (
  id UUID PRIMARY KEY,
  project_id UUID REFERENCES projects(id),
  version INT NOT NULL,
  content JSONB NOT NULL,
  created_by VARCHAR(50),
  created_at TIMESTAMP NOT NULL
);

CREATE TABLE generation_tasks (
  id UUID PRIMARY KEY,
  project_id UUID REFERENCES projects(id),
  task_type VARCHAR(100),
  status VARCHAR(50),
  progress INT,
  current_step VARCHAR(100),
  error_message TEXT,
  started_at TIMESTAMP,
  finished_at TIMESTAMP,
  created_at TIMESTAMP NOT NULL
);

CREATE TABLE generated_outputs (
  id UUID PRIMARY KEY,
  project_id UUID REFERENCES projects(id),
  course_ir_version INT,
  output_type VARCHAR(50),
  title VARCHAR(255),
  content JSONB,
  file_url TEXT,
  version INT,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

## 模型与后期生态表

```sql
CREATE TABLE model_providers (
  id UUID PRIMARY KEY,
  owner_id UUID NOT NULL,
  owner_type VARCHAR(50) NOT NULL,
  display_name VARCHAR(100) NOT NULL,
  provider_type VARCHAR(50) NOT NULL,
  base_url TEXT,
  api_key_encrypted TEXT,
  organization_id TEXT,
  api_version TEXT,
  is_default BOOLEAN DEFAULT FALSE,
  enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);

CREATE TABLE model_configs (
  id UUID PRIMARY KEY,
  provider_id UUID REFERENCES model_providers(id),
  display_name VARCHAR(100),
  model_name VARCHAR(100) NOT NULL,
  capability JSONB,
  max_tokens INT,
  temperature FLOAT,
  top_p FLOAT,
  enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);

CREATE TABLE community_resources (
  id UUID PRIMARY KEY,
  source_project_id UUID REFERENCES projects(id),
  owner_id UUID NOT NULL,
  visibility VARCHAR(50) NOT NULL,
  title VARCHAR(255) NOT NULL,
  summary TEXT,
  subject VARCHAR(100),
  grade VARCHAR(100),
  tags JSONB,
  fork_count INT DEFAULT 0,
  watch_count INT DEFAULT 0,
  chat_count INT DEFAULT 0,
  status VARCHAR(50),
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);

CREATE TABLE resource_relations (
  id UUID PRIMARY KEY,
  resource_id UUID REFERENCES community_resources(id),
  related_project_id UUID REFERENCES projects(id),
  relation_type VARCHAR(50) NOT NULL,
  user_id UUID NOT NULL,
  created_at TIMESTAMP NOT NULL
);

CREATE TABLE agent_sessions (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  project_id UUID REFERENCES projects(id),
  resource_id UUID REFERENCES community_resources(id),
  intent VARCHAR(100),
  status VARCHAR(50),
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);

CREATE TABLE model_call_logs (
  id UUID PRIMARY KEY,
  user_id UUID,
  project_id UUID,
  provider VARCHAR(100),
  model VARCHAR(100),
  task_type VARCHAR(100),
  prompt_tokens INT,
  completion_tokens INT,
  estimated_cost DECIMAL,
  latency_ms INT,
  success BOOLEAN,
  error_message TEXT,
  created_at TIMESTAMP NOT NULL
);
```
