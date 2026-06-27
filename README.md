# 教学课件智能生成系统

本仓库用于承载“教学课件智能生成系统”的产品、技术方案与后续工程实现。当前采用 Monorepo 结构，按职责拆为可独立部署应用、共享业务包、基础设施与文档。

## 目录结构

```text
apps/
  web/      教师端、学生端、教案广场与 Chat UI Web 应用
  api/      核心业务 API：项目、文件、任务、输出、广场、权限与审计
  agent/    Mastra Agent 服务：Chat、Tool、Workflow、Memory、RAG 与观测
  worker/   异步任务进程：解析、生成、渲染、导出、通知与重试
  app/      移动端 App：学生复习、watch 通知与轻量 Chat
packages/
  ui/             共享 UI、主题、图标与设计 token
  config/         TypeScript、Lint、格式化、测试和构建共享配置
  api-client/     Web/App/Agent 调用 API 的类型安全客户端
  domain/         用户、项目、文件、任务、权限、输出等领域模型
  course-ir/      CourseIR Schema、类型、校验、迁移与版本工具
  parser/         PPTX/PDF/DOCX/Markdown/OCR 解析能力
  ai-gateway/     模型供应商适配、路由、Key 管理、审计和成本统计
  agent-tools/    Mastra Tools 业务封装与权限/审计适配
  generator/      学习要点、测试卡、教案、HTML/PPT 结构生成
  renderer-html/  HTML 页面模板、静态打包与 PDF 导出入口
  renderer-ppt/   PPTX 模板、主题与 Slide JSON 渲染
  storage/        PostgreSQL、Redis、S3/MinIO、pgvector 访问封装
  community/      教案广场、fork/watch、评论、谱系与发布审核规则
infra/
  docker/      本地开发与镜像配置
  migrations/  数据库迁移脚本
  deploy/      部署、环境变量、Kubernetes/Terraform/Compose 配置
docs/
  adr/      架构决策记录
  api/      API 契约文档
  testing/  测试策略与验收用例
```

## 工作区约定

- 使用 `pnpm-workspace.yaml` 管理 `apps/*` 与 `packages/*`。
- 根目录 `tsconfig.base.json` 提供 TypeScript 基础配置。
- 每个应用和包都保留 `src/index.ts` 作为稳定入口，后续实现不得从其他模块导入内部路径。
- 业务写入统一经过 `apps/api`，`apps/agent` 和 `apps/worker` 通过受控 API 或队列触发写入。
- 模型调用统一经 `packages/ai-gateway`，避免供应商 SDK 与 API Key 分散在业务代码中。

## 合并冲突处理约定

- 合并或变基后先运行 `pnpm check:conflicts`，确保仓库中不存在 `<<<<<<<`、`=======`、`>>>>>>>` 等冲突标记。
- 文档类冲突优先保留最新的目录规范、模块边界和分阶段执行计划，再补齐被另一侧分支新增的业务说明。
- 工程骨架类冲突优先保留根工作区配置、每个 `apps/*` 与 `packages/*` 的 `package.json`、`tsconfig.json`、`.env.example` 和 `src/index.ts`，避免后续包无法被 workspace 识别。
- 如果代码 README 与产品/技术文档描述不一致，以 `docs/README.md` 指向的分层文档和已接受 ADR 为准，并回填模块 README。`docs/teaching-content-generation-execution-plan.md` 是历史归档，不再作为规范来源。
