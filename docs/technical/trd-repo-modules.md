# TRD：仓库目录与模块规范

## Monorepo 原则

采用 Monorepo，并按“可独立部署的应用放 `apps/`、可复用业务能力放 `packages/`、运行基础设施放 `infra/`、产品和技术文档放 `docs/`”组织。

## 推荐目录

```text
apps/
  web/      教师端、学生端、管理端和后期广场 Web 应用
  api/      核心业务 API，负责权限、项目、文件、任务、模型配置和审计
  agent/    Agent 服务，负责 Chat、Tool、Workflow、Memory、RAG 和观测
  worker/   异步任务进程，负责解析、生成、渲染、导出和失败重试
  app/      移动端 App，可选后期能力

packages/
  ui/             共享 UI、主题、图标和设计 token
  config/         TS/ESLint/Prettier/Tailwind/测试共享配置
  api-client/     类型安全 API SDK
  domain/         项目、教案、资源、用户、权限等领域模型
  course-ir/      CourseIR Schema、校验、迁移、fixtures
  parser/         PPTX、PDF、DOCX、Markdown、OCR 解析
  ai-gateway/     模型供应商适配、路由、Key、日志、成本、重试
  agent-tools/    Agent Tools 的业务封装
  generator/      学习要点、测试卡、教案、HTML/PPT JSON 生成逻辑
  renderer-html/  HTML 页面模板、静态资源和 PDF 导出入口
  renderer-ppt/   PPTX 模板、主题、Slide JSON 渲染
  storage/        数据库、对象存储、缓存、向量库访问封装
  community/      广场、复制、关注、评论、谱系、发布审核规则

infra/
  docker/
  migrations/
  deploy/
```

## 职责边界

1. `apps/web` 只处理页面、路由、表单、编辑器、预览和用户交互，不直接访问数据库、对象存储或模型 SDK。
2. `apps/api` 是业务写入入口，负责鉴权、权限、事务、审计、资源版本和对外 REST/RPC 接口。
3. `apps/agent` 不直接修改核心业务数据；写入教案、发布、推送等动作必须调用 `apps/api` 受控接口。
4. `apps/worker` 执行长任务，不承载面向用户的同步请求；任务状态统一写入任务表并通过 API 查询。
5. `packages/domain` 和 `packages/course-ir` 不依赖具体框架，保持纯类型、Schema 和业务规则。
6. `packages/ai-gateway` 是唯一模型调用出口；Agent、生成器和审校模块不得直接调用模型供应商 SDK。
7. `packages/agent-tools` 只封装工具适配层，工具内部必须进行权限校验、输入 Schema 校验和审计记录。
8. `packages/storage` 只提供基础设施访问能力，不写复杂业务流程。

## 推荐依赖方向

```text
apps/web ───────→ packages/api-client ─────→ apps/api
apps/agent ─────→ packages/agent-tools ────→ apps/api
apps/worker ────→ packages/parser/generator/renderer/storage
packages/generator ─→ packages/course-ir + packages/ai-gateway
packages/renderer-* ─→ packages/course-ir + packages/domain
```

## 模块标准交付清单

每个模块进入开发前应具备：

1. README：说明责任、边界和不应包含什么。
2. 类型和 Schema：DTO、事件、队列 payload、JSON Schema。
3. fixtures：最小输入、正常输出、错误输出。
4. tests：单元测试和关键集成测试。
5. observability：日志、指标、trace 字段。
6. security：权限、审计、脱敏、幂等和重试规则。
