# 原融合总稿拆分覆盖表

本表用于检查 `teaching-content-generation-execution-plan.md` 中的重要内容是否已迁入分层文档，避免 PRD/TRD 拆分后遗漏。

历史总稿不再是规范来源；本表只用于迁移审计，不表示两套文档需要保持双向同步。

| 原总稿章节 | 拆分后的维护位置 | 说明 |
| --- | --- | --- |
| 1 文档信息 | `docs/README.md` | 入口、读者路径和维护规则。 |
| 2.1-2.5 背景、目标、心智、红绿蓝结论 | `product/prd-overview.md`、`reference/user-visible-language-map.md` | 产品定位、教师心智、语言映射、MVP 收敛。 |
| 2.6 PRD + TRD 写作要求 | `execution/execution-task-card-template.md` | 功能任务卡模板和拆 issue 规范。 |
| 3 范围定义 | `product/prd-mvp-scope.md` | MVP-A、MVP-B、标准版、机构版、后期生态边界。 |
| 4 用户角色与场景 | `product/prd-user-roles-and-journeys.md` | 教师、教研员、管理员、学生旅程。 |
| 5 总体业务流程 / Agent 生命周期 | `technical/trd-ai-orchestration-and-rendering.md`、`technical/trd-community-and-agent.md` | 生成流水线、Agent 编排、后期能力边界。 |
| 6.0 教师快速生成入口 | `product/prd-teacher-quick-start.md`、`technical/trd-quick-start-state-machine.md`、`technical/trd-api-and-data-contracts.md` | 页面流程、状态机、字段策略、快速生成 API。 |
| 6.1 项目管理 | `technical/trd-api-and-data-contracts.md`、`technical/trd-database-schema.md` | 作为后台草稿和项目表维护，不作为教师首屏。 |
| 6.2-6.3 文件上传与解析 | `technical/trd-file-upload-and-parsing.md` | 文件格式、解析要求、ParsedContent 示例。 |
| 6.4 CourseIR | `technical/trd-course-ir-and-content-blocks.md` | CourseIR、ContentBlock、版本策略。 |
| 6.5-6.9 HTML/PPT/测试卡/学习要点/教案 | `product/prd-content-outputs-and-quality.md`、`technical/trd-output-generation.md` | 产物结构、字段、MVP 与后期边界。 |
| 6.10 模型配置 | `technical/trd-model-configuration.md`、`docs/adr/0004-ai-gateway-as-single-model-entry.md` | 管理员模型配置、Key 安全、模型网关。 |
| 6.11-6.12 广场与 Agent | `technical/trd-community-and-agent.md` | 后期生态能力，不进入 MVP-A 主线。 |
| 7 技术架构 / 技术栈 / 仓库目录 | `technical/trd-architecture.md`、`technical/trd-repo-modules.md`、`docs/adr/0001-monorepo-structure.md` | 架构、技术栈、模块职责和依赖方向。 |
| 8 后端接口设计 | `technical/trd-api-and-data-contracts.md`、`api/openapi/mvp-a.yaml` | 快速生成 API、错误契约和可执行 OpenAPI。 |
| 9 数据库设计 | `technical/trd-database-schema.md` | 核心表、用途和 SQL 草案。 |
| 10 AI 编排设计 | `technical/trd-ai-orchestration-and-rendering.md`、`technical/trd-community-and-agent.md` | 模型调用原则、路由、Prompt、Agent。 |
| 11 渲染导出设计 | `technical/trd-ai-orchestration-and-rendering.md`、`technical/trd-output-generation.md` | HTML/PPTX/PDF/DOCX 渲染原则。 |
| 12 安全权限合规 | `technical/trd-security-and-testing.md` | 文件、Key、内容、版权、学生数据。 |
| 13 测试策略 / 质量评估 | `technical/trd-security-and-testing.md`、`technical/trd-analytics-and-metrics.md`、`reference/content-quality-rubric.md`、`testing/mvp-a-acceptance.md` | 测试类型、评分标准、数据集、事件与性能口径。 |
| 14 里程碑 | `execution/execution-roadmap.md` | 阶段 0-4。 |
| 15 验收清单 | `execution/execution-acceptance-and-risks.md` | MVP-A、MVP-B、技术验收。 |
| 16 风险与应对 | `execution/execution-acceptance-and-risks.md` | 范围、术语、质量、成本、权限风险。 |
| 17 推荐落地原则 / 18 MVP 最小闭环 | `product/prd-mvp-scope.md`、`execution/execution-roadmap.md` | MVP 硬边界和落地顺序。 |

## 检查结论

原总稿的重要主题均有分层维护位置。是否达到开发就绪状态，仍以 OpenAPI、数据库 migration、测试夹具和任务卡是否同步为准，不能只凭本覆盖表判断。
