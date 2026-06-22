# 教学课件智能生成系统执行文档（PRD + TRD 融合版）

## 1. 文档信息

| 项目 | 内容 |
| --- | --- |
| 文档名称 | 教学课件智能生成系统执行文档 |
| 文档类型 | PRD + TRD 融合执行方案 |
| 目标读者 | 产品、设计、前端、后端、算法、测试、运维、项目管理 |
| 核心目标 | 支持用户上传教学课件，自动生成教学 HTML 页面、上课 PPT、测试卡、学习要点、教案等教学资源，并支持配置模型和 API Key |
| 建议版本 | MVP → 标准版 → 机构版 |

## 2. 项目背景与目标

### 2.1 背景

教师或教研人员在备课过程中通常需要将已有课件整理成多种教学资源，例如课堂展示 PPT、学生学习要点、随堂测试题、教学网页、教案和作业材料。传统流程需要大量人工整理、改写、排版和校验，重复劳动较多。

本系统希望通过文档解析、内容结构化、大模型生成、模板渲染和人工编辑闭环，将“上传课件到生成多种教学资源”的流程产品化、标准化和可配置化。

### 2.2 产品目标

1. 用户上传 PPT、PDF、Word、Markdown、图片等教学资料后，系统自动解析内容。
2. 系统将课件内容结构化为统一课程中间表示，即 CourseIR。
3. 系统基于 CourseIR 自动生成教学 HTML 页面、上课 PPT、测试卡、学习要点、教案等输出。
4. 用户可以配置不同模型供应商、模型名称、Base URL 和 API Key。
5. 用户可以在线预览、编辑、局部重生成和导出最终结果。
6. 系统支持任务进度、失败重试、版本管理、成本统计和调用日志。

### 2.3 技术目标

1. 建立稳定的文件解析链路，支持 PPTX、PDF、DOCX、Markdown 的文本、图片、表格和结构提取。
2. 建立统一 CourseIR 数据结构，作为多格式生成的中心数据模型。
3. 建立模型网关，适配 OpenAI Compatible、Azure OpenAI、Claude、Gemini、DeepSeek、通义千问、智谱、火山方舟、本地模型等。
4. 建立异步生成任务系统，保证长任务可追踪、可恢复、可重试。
5. 采用结构化 JSON 输出加 Schema 校验，降低大模型输出不可控风险。
6. 采用模板渲染生成 HTML、PPTX、PDF、DOCX，而不是让模型直接生成最终文件。

## 3. 范围定义

### 3.1 MVP 范围

MVP 版本必须包含以下能力：

| 模块 | 功能 |
| --- | --- |
| 用户项目 | 创建课程项目、填写课程基础信息 |
| 文件上传 | 上传 PPTX、PDF 文件 |
| 文件解析 | 提取 PPTX、PDF 中的文本、图片、页结构 |
| CourseIR | 生成课程结构、章节、知识点、重点难点 |
| 模型配置 | 支持 OpenAI Compatible 模型配置和连通性测试 |
| 内容生成 | 生成学习要点、测试题、教学 HTML 页面、PPT 大纲或 PPTX |
| 结果预览 | 在线预览生成结果 |
| 导出下载 | 下载 HTML、PPTX、Markdown 或 ZIP |
| 任务系统 | 查看生成进度、失败原因和重试入口 |

### 3.2 标准版范围

标准版在 MVP 基础上增加：

1. DOCX、Markdown、图片 OCR 支持。
2. 教案、作业、课堂活动、思维导图生成。
3. 在线编辑器和局部重生成。
4. 多模板、多主题和学校风格配置。
5. 版本管理、历史对比和恢复。
6. 生成质量审校和题目答案一致性检查。

### 3.3 机构版范围

机构版在标准版基础上增加：

1. 多租户、组织、角色和权限管理。
2. 机构级模型池和 Key 管理。
3. 教材、考纲、题库、校本资源知识库。
4. 成本统计、额度限制、审计日志。
5. 班级、课程体系、教师协作和资源共享。
6. 私有化部署和本地模型接入。
7. 教案广场、资源 fork/watch、围绕教案的 Chat 助教和教学意图识别。

### 3.4 暂不包含范围

MVP 阶段暂不处理：

1. 复杂视频课件自动剪辑。
2. 全自动直播授课。
3. 与 LMS、教务系统的深度集成。
4. 高保真还原所有原始 PPT 动效。
5. 自动保证所有学科事实完全正确，仍需要人工确认。

## 4. 用户角色与使用场景

### 4.1 用户角色

| 角色 | 诉求 | 权限 |
| --- | --- | --- |
| 教师 | 快速生成上课材料和练习题 | 上传课件、生成内容、编辑导出 |
| 教研员 | 统一生成教案、题卡和标准资源 | 管理模板、审核资源、共享课程 |
| 机构管理员 | 控制模型、费用、权限和数据 | 管理用户、模型 Key、额度、审计 |
| 学生 | 查看学习要点和测试卡、在广场发现优质教案 | 只读访问教师发布内容、watch 公开资源、发起学习型 Chat |
| 资源贡献者 | 分享、维护和迭代公开教案 | 发布公开教案、查看 fork/watch/chat 反馈 |

### 4.2 核心用户故事

1. 作为教师，我希望上传已有课件后自动生成学习要点，以便快速发给学生复习。
2. 作为教师，我希望系统根据课件自动生成课堂测试题，以便进行随堂检测。
3. 作为教师，我希望生成可直接上课使用的 PPT，以便节省备课排版时间。
4. 作为教研员，我希望系统按统一模板生成教案，以便保证校内教学资源格式一致。
5. 作为管理员，我希望配置不同模型和 Key，以便根据成本和质量选择合适模型。
6. 作为教师或学生，我希望在「广场」查看其他用户公开分享的教案，以便复用优质教学资源。
7. 作为教师，我希望 fork 公开教案并基于本班情况二次改造，以便形成自己的授课版本。
8. 作为学生或教师，我希望 watch 感兴趣的教案，以便跟进后续更新、评论和衍生版本。
9. 作为学习者，我希望围绕某个教案发起 Chat，让 Agent 结合教案知识库识别我的意图并进行教学引导。

## 5. 总体业务流程

```text
创建课程项目
  → 填写课程信息
  → 上传课件文件
  → 文件解析
  → 内容清洗与结构化
  → 生成 CourseIR
  → 用户确认或修改 CourseIR
  → 选择输出内容
  → 模型生成结构化结果
  → Schema 校验与质量审校
  → 模板渲染 HTML/PPTX/PDF/DOCX
  → 在线预览与编辑
  → 导出、分享或发布到广场
  → 其他用户 fork / watch / chat
```

## 6. 产品功能需求

### 6.1 项目管理

#### 功能说明

用户可以创建、复制、删除、搜索和归档课程项目。

#### 字段要求

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| 课程名称 | 是 | 如“牛顿第二定律” |
| 学科 | 是 | 如物理、语文、数学 |
| 年级 | 是 | 如高一、七年级 |
| 课时 | 否 | 默认 45 分钟 |
| 教学风格 | 否 | 互动式、探究式、应试强化、项目式学习 |
| 学生水平 | 否 | 基础、普通、提高 |
| 输出类型 | 是 | HTML、PPT、测试卡、学习要点、教案等 |

#### 验收标准

1. 用户可以成功创建项目。
2. 项目列表展示项目名称、学科、年级、状态、更新时间。
3. 用户可以进入项目详情查看上传文件和生成结果。
4. 用户可以复制项目并保留输入文件与配置。

### 6.2 文件上传

#### 功能说明

支持用户上传教学资料，并对文件类型、大小和安全性进行校验。

#### MVP 支持格式

| 格式 | 说明 |
| --- | --- |
| PPTX | 核心支持格式 |
| PDF | 核心支持格式 |

#### 后续支持格式

| 格式 | 说明 |
| --- | --- |
| DOCX | 教案、讲义、说明文档 |
| Markdown | 结构化文本材料 |
| PNG/JPG | 图片课件、扫描件 |
| ZIP | 多文件打包上传 |

#### 验收标准

1. 文件上传成功后存储到对象存储。
2. 系统记录文件名称、类型、大小、存储地址、解析状态。
3. 非法格式、超大文件、空文件应给出明确错误提示。
4. 上传成功后自动创建解析任务。

### 6.3 文件解析

#### 功能说明

系统需要从课件中提取文本、图片、表格、页结构和备注信息。

#### PPTX 解析要求

1. 提取每页标题。
2. 提取正文文本。
3. 提取图片并保存为资源文件。
4. 提取表格内容。
5. 提取备注信息。
6. 保留页码顺序。

#### PDF 解析要求

1. 优先处理文本型 PDF。
2. 对扫描型 PDF 使用 OCR。
3. 提取页码、段落、标题、图片和表格。
4. 保留来源页码，便于结果追溯。

#### 统一解析结果示例

```json
{
  "file_id": "file_001",
  "file_type": "pptx",
  "pages": [
    {
      "page_index": 1,
      "title": "牛顿第二定律",
      "blocks": [
        {
          "type": "text",
          "text": "物体的加速度与所受合外力成正比"
        },
        {
          "type": "image",
          "asset_id": "asset_001",
          "path": "s3://bucket/project/image001.png"
        }
      ],
      "notes": "可从生活实例导入"
    }
  ]
}
```

### 6.4 CourseIR 课程中间表示

#### 功能说明

CourseIR 是系统的核心数据模型，用于承接课件解析结果，并驱动所有后续生成产物。

#### CourseIR 设计原则

1. 可读：方便人工编辑和审查。
2. 可扩展：支持不同学科、年级和输出类型。
3. 可追溯：知识点和章节应能关联原始文件页码。
4. 可校验：必须有 JSON Schema。
5. 可版本化：每次重要修改生成新版本。

#### CourseIR 示例

```json
{
  "course_title": "牛顿第二定律",
  "subject": "物理",
  "grade": "高一",
  "duration_minutes": 45,
  "teaching_style": "探究式",
  "objectives": [
    "理解牛顿第二定律的内容",
    "能够使用 F=ma 解决简单问题"
  ],
  "chapters": [
    {
      "id": "ch_1",
      "title": "情境导入",
      "estimated_minutes": 5,
      "source_refs": [
        {
          "file_id": "file_001",
          "page_index": 1
        }
      ],
      "knowledge_points": [
        {
          "id": "kp_1",
          "name": "力与运动状态变化",
          "description": "力是改变物体运动状态的原因",
          "difficulty": "easy",
          "importance": "medium"
        }
      ]
    }
  ]
}
```

### 6.5 教学 HTML 页面生成

#### 功能说明

系统根据 CourseIR 生成可在线浏览的教学页面。

#### 页面组成

1. 课程封面。
2. 章节导航。
3. 教学目标。
4. 知识点讲解。
5. 图文说明。
6. 互动测试题。
7. 学习总结。
8. 课后练习。

#### 技术要求

1. 模型输出页面结构 JSON。
2. 后端或前端模板负责渲染 HTML。
3. HTML 页面应支持响应式布局。
4. 不允许直接信任模型输出的任意脚本。
5. 支持导出静态 HTML 包。

### 6.6 上课 PPT 生成

#### 功能说明

系统根据 CourseIR 和教学风格生成 PPT 大纲，并渲染成 PPTX。

#### PPT 类型

| 类型 | 说明 |
| --- | --- |
| 教师授课版 | 内容完整，包含讲解提示 |
| 学生简版 | 内容精简，便于课堂展示 |
| 复习版 | 强调知识点、易错点和练习题 |

#### PPT JSON 示例

```json
{
  "slides": [
    {
      "layout": "title",
      "title": "牛顿第二定律",
      "subtitle": "高一物理"
    },
    {
      "layout": "formula",
      "title": "核心公式",
      "formula": "F = ma",
      "bullets": [
        "F 表示合外力",
        "m 表示质量",
        "a 表示加速度"
      ]
    }
  ]
}
```

### 6.7 测试卡生成

#### 功能说明

系统根据知识点生成题卡，支持题型、数量、难度和知识点覆盖率配置。

#### 支持题型

1. 单选题。
2. 多选题。
3. 判断题。
4. 填空题。
5. 简答题。
6. 案例分析题。

#### 题目字段

| 字段 | 说明 |
| --- | --- |
| question | 题干 |
| type | 题型 |
| options | 选项 |
| answer | 正确答案 |
| explanation | 解析 |
| difficulty | 难度 |
| knowledge_point_id | 关联知识点 |
| source_refs | 来源页码 |

#### 验收标准

1. 题目必须有答案和解析。
2. 单选题答案必须唯一。
3. 题目应覆盖主要知识点。
4. 题目难度应符合用户选择。
5. 题卡可以导出为 HTML、PDF、DOCX 或 Markdown。

### 6.8 学习要点生成

#### 功能说明

面向学生输出简洁、准确、可复习的学习材料。

#### 内容结构

1. 本课概览。
2. 核心知识点。
3. 重点公式或概念。
4. 易错点。
5. 记忆技巧。
6. 课后自测。

### 6.9 教案生成

#### 功能说明

面向教师生成标准教案。

#### 教案结构

1. 教学目标。
2. 教学重点。
3. 教学难点。
4. 教学准备。
5. 教学过程。
6. 课堂互动。
7. 板书设计。
8. 作业设计。
9. 教学反思。

### 6.10 模型配置

#### 功能说明

用户或管理员可以配置模型供应商、Base URL、API Key 和模型用途。

#### 配置字段

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| provider_type | 是 | openai_compatible、azure_openai、anthropic、gemini、local 等 |
| display_name | 是 | 用户可识别名称 |
| base_url | 是 | 模型服务地址 |
| api_key | 是 | 加密存储 |
| model_name | 是 | 实际调用模型名称 |
| capability | 是 | 文本、视觉、嵌入、JSON、函数调用等能力 |
| is_default | 否 | 是否默认模型 |

#### 安全要求

1. API Key 必须加密存储。
2. 前端不能回显完整 Key。
3. 调用时临时解密。
4. 必须记录调用日志，但不能记录完整敏感 Key。
5. 支持删除、禁用和连通性测试。

### 6.11 教案广场与资源协作

#### 功能说明

「广场」用于承载公开或组织内共享的教案、学习要点、测试卡和课堂活动方案，让教师、学生和教研员可以发现、复用、关注和讨论教学资源。

#### 核心能力

1. 资源发现：按学科、年级、教材版本、知识点、难度、热度、更新时间筛选教案。
2. 资源详情：展示教案摘要、适用场景、来源课件引用、生成版本、作者、更新时间、fork 数、watch 数和 chat 入口。
3. Fork：用户可基于公开教案创建自己的项目副本，并保留来源关系、版本号和引用声明。
4. Watch：用户可关注教案更新、评论、衍生版本和作者发布的新资源。
5. Chat：用户可围绕单个教案或教案集合发起对话，Agent 结合教案内容、CourseIR、测试题、学习要点和来源页码进行意图识别与教学引导。
6. 反馈闭环：支持点赞、收藏、纠错、评论、举报和教学适用性评分。

#### 权限要求

1. 公开资源可被登录用户查看、watch 和发起 Chat。
2. Fork 后的私有副本默认仅创建者可见，可再次选择发布到广场。
3. 组织资源可限制在学校、教研组或班级范围内可见。
4. 学生账号只能 fork 到个人学习空间，不能覆盖教师原始资源。

#### 验收标准

1. 用户可以在广场检索并打开公开教案详情。
2. 用户可以 fork 教案并在自己的项目列表中看到新副本。
3. 用户可以 watch 教案并收到更新提醒。
4. 用户可以从教案详情进入 Chat，Agent 能引用当前教案内容回答问题。
5. Fork、watch、chat、评论等行为均记录审计日志。

### 6.12 Agent 教学引导

#### 功能说明

Agent 面向教师备课、课堂组织和学生自学场景，基于 Chat + 教案知识库完成意图识别、教学引导、练习推荐和内容改写。

#### 推荐能力边界

1. 意图识别：区分备课、讲解、答疑、练习、纠错、复习、教研评价等意图。
2. 教案问答：优先基于 CourseIR、教案、测试卡、学习要点和来源页码回答。
3. 教学引导：针对学生回答进行追问、提示、分层解释和错因分析。
4. 教师助手：帮助教师改写导入语、生成课堂提问、调整难度和补充活动。
5. 结果沉淀：将高质量对话片段沉淀为教案备注、FAQ 或新的课堂活动。

#### 开发技术建议

1. Agent 编排层：如果主工程采用 TypeScript/Node，优先使用 Mastra；如果主工程采用 Python，再考虑 LangGraph、LlamaIndex Workflows；复杂度较低时也可使用自研轻量状态机，避免把业务流程完全写死在 Prompt 中。
2. 模型接入层：Mastra 负责 Agent、Tool、Workflow、Memory 和观测；业务侧仍保留 ai-gateway 作为统一模型配置、Key 管理、审计和成本统计入口，避免供应商凭证散落在 Agent 代码中。
3. 知识库检索：使用 PostgreSQL + pgvector 作为 MVP 默认方案；机构版可扩展到 Qdrant 或 Milvus。
4. 工具调用：将教案检索、CourseIR 查询、题目生成、局部重生成、版本保存、权限校验封装为 Tool。
5. 会话记忆：短期记忆存储在 Redis 或数据库会话表，长期记忆只沉淀经用户确认的教学偏好和高质量内容。
6. 安全护栏：所有 Agent 输出必须带来源优先级、权限校验和敏感内容过滤；涉及事实性知识时提示教师审核。

#### Mastra 落地建议

1. 推荐结论：可以使用 Mastra，尤其适合当前推荐技术栈中的 Next.js、React、NestJS/Node、TypeScript 方向。
2. 使用边界：Mastra 不替代业务后端、权限系统、CourseIR、内容审核、资源版本管理和渲染导出服务；它只负责 Agent 编排、工具调用、工作流、人机确认、记忆和观测。
3. 集成方式：将 Mastra 放在 `packages/agent` 或独立 `apps/agent` 中，通过内部 API 调用项目、广场、CourseIR、题库、文件和渲染服务。
4. 工具设计：把 `searchLessonPlan`、`getCourseIR`、`generateQuiz`、`regenerateSection`、`forkResource`、`watchResource`、`saveTeachingNote` 等能力封装为 Mastra Tools。
5. 工作流设计：把“意图识别 → 权限校验 → 检索 → 工具选择 → 生成回答 → 人工确认 → 写入版本”的关键链路实现为 Mastra Workflow；纯问答场景可使用 Agent + RAG。
6. 人机协同：教案发布、覆盖原教案、批量生成题目、向学生推送内容等高影响动作必须进入 human-in-the-loop 确认。
7. 可观测性：记录每次 Agent 调用的用户、资源、检索片段、工具调用、模型、token、延迟、成本、失败原因和人工反馈。
8. MVP 建议：先用 Mastra 实现“围绕单个教案 Chat + 引用来源 + 生成课堂提问/练习 + 保存为备注”，再扩展到多教案检索、自动 fork 改写和机构级知识库。


## 7. 技术架构设计

### 7.1 总体架构

```text
Web 前端
  ↓
API 服务
  ↓
任务队列
  ↓
文件解析服务
  ↓
CourseIR 结构化服务
  ↓
AI 编排与模型网关
  ↓
内容生成服务
  ↓
渲染导出服务
  ↓
数据库 / 对象存储 / 向量库
```

### 7.2 推荐技术栈

| 层级 | 推荐方案 |
| --- | --- |
| 前端 | Next.js、React、Tailwind CSS、shadcn/ui、Tiptap |
| 后端 | FastAPI 或 NestJS |
| Agent 编排 | Mastra（TypeScript 优先）、LangGraph/LlamaIndex Workflows（Python 优先） |
| 数据库 | PostgreSQL |
| 对象存储 | S3 或 MinIO |
| 缓存 | Redis |
| 异步任务 | Celery、BullMQ 或 Temporal |
| 向量库 | pgvector、Qdrant 或 Milvus |
| PPT 解析 | python-pptx、LibreOffice headless |
| PDF 解析 | PyMuPDF、pdfplumber、OCR |
| PPT 生成 | PptxGenJS 或 python-pptx |
| HTML 渲染 | React 模板或服务端模板 |

### 7.3 仓库目录与模块规范

建议采用 Monorepo，并按“可独立部署的应用放 `apps/`、可复用业务能力放 `packages/`、运行基础设施放 `infra/`、产品和技术文档放 `docs/`”的原则组织。

```text
apps/
  web/
    教师端、学生端和广场 Web 应用，建议使用 Next.js。
  api/
    核心业务 API，负责项目、文件、生成任务、权限、广场和计费等稳定业务能力。
  agent/
    Agent 服务，建议 TypeScript 技术栈使用 Mastra；负责 Chat、Tool、Workflow、Memory、RAG 和 Agent 观测。
  worker/
    异步任务进程，负责文件解析、内容生成、渲染导出、批量任务和失败重试。
  app/
    移动端 App，可选；建议使用 Expo/React Native，复用 packages 中的类型、API client 和 UI 基础组件。

packages/
  ui/
    Web/App 共享 UI 组件、主题、图标和设计 token。
  config/
    TypeScript、ESLint、Prettier、Tailwind、测试等共享配置。
  api-client/
    前端和 Agent 调用 API 的类型安全 SDK。
  domain/
    项目、教案、广场、用户、权限等领域模型和纯业务规则。
  course-ir/
    CourseIR Schema、类型定义、校验、迁移和版本管理。
  parser/
    PPTX、PDF、DOCX、Markdown、OCR 等文件解析能力。
  ai-gateway/
    模型供应商适配、模型路由、Key 解密、调用日志、成本统计和重试。
  agent-tools/
    Mastra Tools 的业务封装，如教案检索、CourseIR 查询、题目生成、局部重生成、fork/watch。
  generator/
    学习要点、测试卡、教案、HTML 结构、PPT 大纲等内容生成逻辑。
  renderer-html/
    HTML 页面模板、静态资源打包和 PDF 导出入口。
  renderer-ppt/
    PPTX 模板、主题和 Slide JSON 渲染。
  storage/
    数据库、对象存储、缓存、向量库访问封装。
  community/
    广场、fork、watch、评论、资源谱系和发布审核规则。

infra/
  docker/
    本地开发和部署镜像。
  migrations/
    数据库迁移脚本。
  deploy/
    环境变量模板、Kubernetes/Terraform/Compose 等部署配置。

docs/
  产品 PRD、技术设计、接口规范、架构决策记录和测试策略。
```

#### 目录职责边界

1. `apps/web` 只处理页面、路由、表单、编辑器、预览和用户交互，不直接访问数据库和对象存储。
2. `apps/api` 是业务写入入口，负责鉴权、权限、事务、审计、资源版本和对外 REST/RPC 接口。
3. `apps/agent` 不直接修改核心业务数据；需要写入教案、fork、watch、发布或推送时，必须调用 `apps/api` 暴露的受控接口。
4. `apps/worker` 执行长任务，不承载面向用户的同步请求；任务状态统一写入任务表并通过 API 查询。
5. `packages/domain` 和 `packages/course-ir` 不依赖具体框架，保持纯类型、Schema 和业务规则，便于 Web、API、Agent、Worker 复用。
6. `packages/ai-gateway` 是唯一模型调用出口；Agent、生成器和审校模块不得直接散落调用各模型供应商 SDK。
7. `packages/agent-tools` 只封装工具适配层，工具内部必须进行权限校验、输入 Schema 校验和审计记录。
8. `packages/storage` 只提供基础设施访问能力，不写复杂业务流程；复杂流程应放在 `apps/api` 或领域服务中。

#### 推荐依赖方向

```text
apps/web ───────→ packages/api-client ─────→ apps/api
apps/app ───────→ packages/api-client ─────→ apps/api
apps/agent ─────→ packages/agent-tools ────→ apps/api / packages/ai-gateway
apps/worker ────→ packages/parser / generator / renderer-* / storage
apps/api ───────→ packages/domain / course-ir / community / storage / ai-gateway
packages/* ─────→ packages/config / domain / course-ir，不反向依赖 apps/*
```

#### 命名与工程规范

1. 应用目录使用单数业务名：`web`、`api`、`agent`、`worker`、`app`。
2. 共享包使用短横线命名：`course-ir`、`ai-gateway`、`api-client`、`agent-tools`。
3. API 入参、Agent Tool 入参、CourseIR、生成结果均必须有 Zod 或 JSON Schema 校验。
4. 跨应用共享类型必须来自 `packages/*`，禁止从 `apps/*` 相互导入。
5. 每个 package 暴露稳定 `src/index.ts`，内部文件不作为跨包导入入口。
6. 环境变量按应用拆分：`apps/api/.env.example`、`apps/agent/.env.example`、`apps/worker/.env.example`，敏感 Key 不进入仓库。
7. 数据迁移、任务队列、对象存储 Bucket、向量索引等基础设施变更必须进入 `infra/` 并配套文档说明。
8. 后续如果选择 Python 解析服务，可放在 `apps/parser-service` 或 `packages/parser-python`，通过队列/API 与 Node 主工程解耦。

packages/storage
  数据库、对象存储和缓存访问

packages/agent
  Agent 状态机、工具调用、意图识别和教学引导

packages/community
  广场、fork、watch、评论和资源关系管理
```

## 8. 后端接口设计

### 8.1 项目接口

```http
POST /api/projects
GET /api/projects
GET /api/projects/{project_id}
PATCH /api/projects/{project_id}
DELETE /api/projects/{project_id}
```

### 8.2 文件接口

```http
POST /api/projects/{project_id}/files
GET /api/projects/{project_id}/files
GET /api/files/{file_id}/parsed-result
DELETE /api/files/{file_id}
```

### 8.3 生成任务接口

```http
POST /api/projects/{project_id}/generate
GET /api/tasks/{task_id}
POST /api/tasks/{task_id}/retry
POST /api/tasks/{task_id}/cancel
```

### 8.4 输出结果接口

```http
GET /api/projects/{project_id}/outputs
GET /api/outputs/{output_id}
PATCH /api/outputs/{output_id}
POST /api/outputs/{output_id}/regenerate
GET /api/outputs/{output_id}/download
```

### 8.5 模型配置接口

```http
POST /api/model-providers
GET /api/model-providers
PATCH /api/model-providers/{provider_id}
DELETE /api/model-providers/{provider_id}
POST /api/model-providers/{provider_id}/test
```

### 8.6 广场与 Agent 接口

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

## 9. 数据库设计

### 9.1 projects

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
```

### 9.2 project_files

```sql
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
```

### 9.3 course_ir_versions

```sql
CREATE TABLE course_ir_versions (
  id UUID PRIMARY KEY,
  project_id UUID REFERENCES projects(id),
  version INT NOT NULL,
  content JSONB NOT NULL,
  created_by VARCHAR(50),
  created_at TIMESTAMP NOT NULL
);
```

### 9.4 generation_tasks

```sql
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
```

### 9.5 generated_outputs

```sql
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

### 9.6 model_providers

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
```

### 9.7 model_configs

```sql
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
```

### 9.8 community_resources

```sql
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
```

### 9.9 resource_relations

```sql
CREATE TABLE resource_relations (
  id UUID PRIMARY KEY,
  resource_id UUID REFERENCES community_resources(id),
  related_project_id UUID REFERENCES projects(id),
  relation_type VARCHAR(50) NOT NULL,
  user_id UUID NOT NULL,
  created_at TIMESTAMP NOT NULL
);
```

### 9.10 agent_sessions

```sql
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
```

### 9.11 model_call_logs

```sql
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

## 10. AI 编排设计

### 10.1 模型调用原则

1. 大模型负责理解、规划、改写和生成结构化内容。
2. 系统负责解析文件、校验 JSON、渲染文件和权限控制。
3. 所有关键输出必须使用 JSON Schema 校验。
4. 长内容生成采用分章节生成和汇总生成结合方式。
5. 失败时支持重试、降级模型和人工介入。

### 10.2 生成流水线

```text
ParsedContent
  → Course Analysis Prompt
  → CourseIR Draft
  → CourseIR Validation
  → Output Planning
  → HTML JSON Generation
  → PPT JSON Generation
  → Quiz JSON Generation
  → Study Notes Generation
  → Quality Review
  → Rendering
```

### 10.3 模型路由

| 任务 | 推荐模型类型 |
| --- | --- |
| 课件理解 | 长上下文文本模型或多模态模型 |
| 图片理解 | 视觉模型 |
| CourseIR 生成 | 强推理文本模型 |
| 测试题生成 | 文本模型 |
| HTML 结构生成 | 文本模型 |
| PPT 大纲生成 | 强文本模型 |
| 嵌入检索 | Embedding 模型 |
| 审校检查 | 成本适中的文本模型 |

### 10.4 Agent 编排流程

```text
用户消息 + 当前教案上下文
  → 权限校验
  → 意图识别
  → 检索 CourseIR / 教案 / 测试卡 / 学习要点
  → 选择工具或生成策略
  → 输出教学引导
  → 记录来源、反馈和可沉淀内容
```

Agent 不应直接绕过业务系统修改最终教案；涉及写入教案、生成新题目、发布到广场等动作时，需要通过受控 Tool 调用并保留版本记录。

### 10.5 Prompt 模板管理

Prompt 应版本化，并记录：

1. prompt_key。
2. prompt_version。
3. 适用任务。
4. 输入变量。
5. 输出 Schema。
6. 示例输入输出。
7. 发布状态。

## 11. 渲染导出设计

### 11.1 HTML 渲染

HTML 渲染采用结构 JSON + 模板方式。

```text
HTMLPageJSON
  → React/Vue/服务端模板
  → 静态 HTML
  → assets 打包
```

### 11.2 PPTX 渲染

PPTX 渲染采用 Slide JSON + PPT 渲染器方式。

```text
SlideJSON
  → 主题模板
  → PptxGenJS/python-pptx
  → PPTX 文件
```

### 11.3 PDF / DOCX 导出

1. HTML 可通过 Playwright 或 Chromium 导出 PDF。
2. 学习要点和测试卡可通过 DOCX 模板导出 Word。
3. 所有导出文件可打包为 ZIP。

## 12. 安全、权限与合规

### 12.1 文件安全

1. 限制文件类型和大小。
2. 文件上传后进行病毒扫描或安全扫描。
3. 用户只能访问自己或组织授权的文件。
4. 对象存储使用私有 Bucket 和签名 URL。

### 12.2 模型 Key 安全

1. API Key 使用 KMS 或应用层密钥加密。
2. Key 不出现在日志中。
3. Key 在前端仅显示掩码。
4. 删除模型配置时同步删除密文。

### 12.3 内容安全

1. 对生成内容进行敏感词和不适宜内容检测。
2. HTML 输出需要净化，禁止任意脚本注入。
3. 对外分享链接需要权限和过期时间控制。

## 13. 质量保障与测试策略

### 13.1 单元测试

覆盖：

1. 文件类型识别。
2. CourseIR Schema 校验。
3. 模型配置加密解密。
4. Prompt 输入变量构造。
5. HTML/PPT 渲染器。

### 13.2 集成测试

覆盖：

1. 上传 PPTX 后生成 CourseIR。
2. 上传 PDF 后生成学习要点。
3. 配置模型后执行连通性测试。
4. 生成任务失败后重试。
5. 导出 ZIP 并校验文件完整性。

### 13.3 质量评估

建议建立人工或半自动评估集，指标包括：

1. 知识点召回率。
2. 教学目标准确性。
3. 题目答案正确率。
4. 难度匹配度。
5. 输出格式合规率。
6. 教师可用性评分。

## 14. 里程碑计划

### 14.1 第 1 阶段：MVP 骨架，2-3 周

1. 项目管理。
2. 文件上传。
3. PPTX/PDF 基础解析。
4. 模型配置。
5. 异步任务框架。

### 14.2 第 2 阶段：核心生成，2-3 周

1. CourseIR 生成。
2. 学习要点生成。
3. 测试卡生成。
4. 教学 HTML 生成。
5. 基础结果预览。

### 14.3 第 3 阶段：PPT 与导出，2 周

1. PPT JSON 生成。
2. PPTX 模板渲染。
3. PDF/DOCX/ZIP 导出。
4. 下载和分享。

### 14.4 第 4 阶段：编辑与质量，2-3 周

1. 在线编辑。
2. 局部重生成。
3. 版本管理。
4. AI 审校。
5. 题目质量检查。

### 14.5 第 5 阶段：机构能力，持续迭代

1. 多租户权限。
2. 教案广场、fork/watch 和资源谱系。
3. Chat Agent 与知识库 RAG。
4. 成本统计。
5. 审计日志。
6. 私有化部署支持。

## 15. 验收清单

### 15.1 MVP 验收

1. 用户可以创建项目并上传 PPTX/PDF。
2. 系统可以解析上传文件并生成结构化解析结果。
3. 系统可以生成 CourseIR。
4. 系统可以生成学习要点、测试卡和教学 HTML 页面。
5. 系统可以生成 PPT 大纲或 PPTX。
6. 用户可以配置 OpenAI Compatible 模型和 API Key。
7. 用户可以查看任务进度和失败原因。
8. 用户可以预览和下载生成结果。

### 15.2 技术验收

1. 关键输出均通过 JSON Schema 校验。
2. API Key 加密存储且日志不泄露敏感信息。
3. 异步任务支持失败重试。
4. 文件存储和结果存储可追溯。
5. 生成结果与原课件页码具备基本关联关系。

## 16. 风险与应对

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| 文件解析质量不稳定 | 影响 CourseIR 和后续生成 | 引入多解析器兜底，保留人工编辑入口 |
| 模型输出格式不稳定 | 渲染失败 | 使用 JSON Schema、重试和修复 Prompt |
| 生成内容存在知识错误 | 教学风险 | 增加审校、来源引用和教师确认流程 |
| PPT 排版不可控 | 用户体验差 | 使用固定模板和 Slide JSON 渲染 |
| API Key 泄露 | 安全风险 | 加密存储、日志脱敏、权限隔离 |
| 生成成本过高 | 商业不可控 | 模型路由、缓存、额度和成本统计 |
| 长课件超上下文 | 生成失败 | 分页切分、章节摘要、RAG 检索 |

## 17. 推荐落地原则

1. 优先实现 CourseIR，不要让每个输出类型直接依赖原始文件。
2. 优先让模型生成结构化 JSON，不要直接生成复杂最终文件。
3. 优先提供可编辑、可重生成能力，而不是追求一次生成完美。
4. 优先记录来源页码和版本，方便教师审核。
5. 优先支持 OpenAI Compatible 接口，后续扩展更多模型供应商。
6. 优先完成 MVP 的闭环体验，再扩展机构能力和知识库能力。

## 18. MVP 最小闭环定义

MVP 成功标准如下：

```text
用户上传一个 PPTX 或 PDF
  → 系统解析课件
  → 系统生成 CourseIR
  → 系统生成学习要点、测试卡、教学 HTML 页面和 PPTX
  → 用户在线预览
  → 用户下载 ZIP
```

只要该闭环稳定可用，就可以进入真实教师试用阶段，并基于反馈继续优化解析质量、模板质量和生成质量。
