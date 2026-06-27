# TRD：技术架构

## 技术目标

1. 建立稳定的文件解析链路，支持 PPTX、PDF，后续扩展 DOCX、Markdown、图片 OCR。
2. 建立统一课程结构模型，作为多格式生成的中心数据模型。
3. 建立模型网关，适配 OpenAI Compatible、Azure OpenAI、Claude、Gemini、DeepSeek、通义千问、智谱、火山方舟、本地模型等。
4. 建立异步生成任务系统，保证长任务可追踪、可恢复、可重试。
5. 采用结构化 JSON 输出加 Schema 校验，降低大模型输出不可控风险。
6. 采用模板渲染生成 HTML/PDF/DOCX/PPTX，而不是让模型直接生成最终文件。

## 推荐模块边界

| 模块 | 责任 |
| --- | --- |
| `apps/web` | 教师端、学生分享页、管理端页面和交互 |
| `apps/api` | 业务 API、权限、会话、任务调度入口 |
| `apps/worker` | 文件解析、生成、渲染、导出等长任务 |
| `apps/agent` | AI 助手编排，不直接修改核心业务数据 |
| `packages/domain` | 核心领域类型与业务规则 |
| `packages/course-ir` | 课程结构、知识点、来源引用等数据模型 |
| `packages/parser` | PPTX/PDF/DOCX/Markdown/OCR 解析 |
| `packages/ai-gateway` | 唯一模型调用出口、成本、日志、重试 |
| `packages/generator` | Prompt、Schema、生成与修复策略 |
| `packages/renderer-*` | HTML、PDF、DOCX、PPTX 渲染 |
| `packages/storage` | 文件存储、对象存储、导出产物 |

## MVP 已选技术栈

MVP 使用 NestJS + Fastify、BullMQ + Redis、PostgreSQL 和 S3 兼容对象存储。详细理由、边界和重新评估条件见 [ADR 0005](../adr/0005-mvp-runtime-and-queue.md)。历史总稿中列出的 FastAPI、Celery、Temporal、Qdrant 等均不是当前 MVP 依赖。

## 架构原则

1. 教师端只面向用户语言，内部术语不直接暴露。
2. 所有模型调用必须经过 `packages/ai-gateway`。
3. 所有长任务必须可查询状态、可重试、可追溯。
4. 所有生成内容必须关联来源页码、Prompt 版本、模型和内容版本。
5. Agent 不直接写核心业务表，只通过受控工具和 API 触发动作。
6. HTTP 契约以 `docs/api/openapi/mvp-a.yaml` 为准，数据库契约以 migration 和数据库 TRD 为准。
