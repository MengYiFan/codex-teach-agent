# 教学课件智能生成系统文档入口

本目录把原来的融合总稿拆成面向不同读者的分层文档，避免产品、研发、测试、商业化团队都从一份 1800+ 行长文档开始阅读。

> 权威来源：本页指向的分层文档和状态为 `Accepted` 的 ADR 是当前唯一规范来源。历史融合总稿只用于追溯背景；发生冲突时不得以历史总稿覆盖分层文档。

## 5 分钟了解

- [产品总览](product/prd-overview.md)：理解产品定位、首版价值主张和教师心智。
- [MVP 范围](product/prd-mvp-scope.md)：确认 MVP-A、MVP-B、标准版、机构版边界。
- [用户可见语言映射](reference/user-visible-language-map.md)：确认教师端、学生端、管理端分别应该怎么说。

## 产品 / 设计必读

- [教师快速生成入口 PRD](product/prd-teacher-quick-start.md)
- [用户角色与核心旅程](product/prd-user-roles-and-journeys.md)
- [内容产物与审核质量](product/prd-content-outputs-and-quality.md)
- [商业化与增长指标](product/prd-commercialization-and-metrics.md)

## 前端 / 后端 / 算法必读

- [技术架构 TRD](technical/trd-architecture.md)
- [仓库目录与模块规范 TRD](technical/trd-repo-modules.md)
- [文件上传与解析 TRD](technical/trd-file-upload-and-parsing.md)
- [课程结构与内容块 TRD](technical/trd-course-ir-and-content-blocks.md)
- [内容产物生成与渲染 TRD](technical/trd-output-generation.md)
- [API 与数据契约 TRD](technical/trd-api-and-data-contracts.md)
- [快速生成状态机 TRD](technical/trd-quick-start-state-machine.md)
- [数据库 Schema TRD](technical/trd-database-schema.md)
- [AI 编排与渲染导出 TRD](technical/trd-ai-orchestration-and-rendering.md)
- [模型配置与 AI Gateway TRD](technical/trd-model-configuration.md)
- [教案广场与 Agent 后期能力 TRD](technical/trd-community-and-agent.md)
- [安全合规与测试策略 TRD](technical/trd-security-and-testing.md)

## 项目管理 / 测试必读

- [执行路线图](execution/execution-roadmap.md)
- [功能任务卡模板](execution/execution-task-card-template.md)
- [验收清单与风险登记](execution/execution-acceptance-and-risks.md)
- [原融合总稿拆分覆盖表](execution/execution-coverage-map.md)
- [MVP-A 测试与验收规范](testing/mvp-a-acceptance.md)

## 可执行契约

- [MVP-A OpenAPI 3.1](api/openapi/mvp-a.yaml)
- [API 维护说明](api/README.md)
- [测试与评估入口](testing/README.md)

## 参考资料

- [术语表](reference/glossary.md)
- [用户可见语言映射](reference/user-visible-language-map.md)
- [内容质量评分标准](reference/content-quality-rubric.md)
- [埋点事件字典](reference/event-tracking-dictionary.md)

## 历史总稿

- [PRD + TRD 融合总稿（只读归档）](teaching-content-generation-execution-plan.md)

> 维护规则：产品概念进 `product/`，技术实现进 `technical/`，任务拆解进 `execution/`，术语/指标/事件进 `reference/`，重大技术取舍进 `adr/`。接口变更必须同步 OpenAPI；验收口径变更必须同步 `testing/`。
