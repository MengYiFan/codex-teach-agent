# ADR 0005：MVP 主运行时、API 与任务队列技术栈

## 状态

Accepted（2026-06-27）

## 背景

仓库已经采用 pnpm + TypeScript Monorepo，并存在 `apps/api`、`apps/worker`、`packages/*` 的 TypeScript 骨架。若 MVP 同时保留 FastAPI/NestJS、Celery/BullMQ/Temporal 等多套选择，会增加类型契约、部署、观测和维护成本。

## 决策

MVP 使用以下主栈：

| 层级 | 决策 |
| --- | --- |
| Web | Next.js + React；只通过生成的 `packages/api-client` 调用 API |
| API | NestJS + Fastify adapter；OpenAPI 3.1 是 HTTP 契约来源 |
| Worker | Node.js + BullMQ；Redis 承载队列、延迟重试和短期状态 |
| 数据库 | PostgreSQL；SQL migration 是 Schema 来源，`packages/storage` 提供 repository |
| 对象存储 | S3 兼容私有 Bucket；数据库只保存 object key，不保存长期签名 URL |
| 模型调用 | `packages/ai-gateway`；业务模块不得直接依赖供应商 SDK |
| 文件解析 | `packages/parser` 提供统一接口；需要 Python/LibreOffice/OCR 时以隔离进程或独立 Worker adapter 接入 |

MVP-A 不引入 Temporal、独立向量数据库、Agent 服务或移动端运行时。出现以下任一条件时再单独提交 ADR：

1. 工作流需要跨天等待、补偿事务或大量人工审批，BullMQ 无法清晰表达。
2. Node 解析链路无法达到验收集要求，需要独立 Python 解析服务。
3. 单 PostgreSQL/Redis 部署无法满足已测量的容量或隔离目标。

## 影响

- API、Worker、共享 Schema 使用同一 TypeScript 类型体系。
- 队列任务必须具备幂等键、有限重试、指数退避和死信处理。
- Worker 可以写任务结果、输出版本和技术状态，但用户发起的业务动作、权限判断和审计入口仍由 API 控制。
- 后期 Agent、社区、向量检索能力不得提前成为 MVP-A 的运行依赖。

## 被否决方案

- MVP 同时维护 NestJS 与 FastAPI：契约和部署重复。
- MVP 直接采用 Temporal：当前流程尚不需要其运维和建模复杂度。
- 所有解析逻辑强制 TypeScript：PPTX/OCR 的具体实现应由质量评估决定，而不是语言偏好决定。
