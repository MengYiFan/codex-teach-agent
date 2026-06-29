# 教学课件智能生成系统执行文档（PRD + TRD 融合版）

> **拆分说明（2026-06-26）**：本文保留为 PRD + TRD 融合总稿。后续协作请优先阅读 [docs/README.md](README.md)，并在 `product/`、`technical/`、`execution/`、`reference/`、`adr/` 中维护对应职责的分层文档。

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

### 2.4 产品心智与体验原则

系统面向一线教师时，应优先呈现“我有一份课件，帮我快速变成可用教学材料”的心智，而不是让教师先理解项目、模型、任务、CourseIR、Schema 等技术概念。教师端的首要体验应是：

```text
上传课件
  → 选择用途：明天上课 / 发给学生复习 / 出随堂题 / 写教案
  → 系统自动识别课程信息并生成初稿
  → 教师局部修改或重生成
  → 导出、分享或进入课堂使用
```

体验设计原则：

1. 教师可见概念使用教学语言，例如“课程结构”“来源页码”“重新生成这一段”，避免直接暴露 CourseIR、Schema、异步任务等技术词。
2. 优先让用户看到可编辑初稿，再引导补充学科、年级、课时、学生水平等信息，降低首次使用门槛。
3. MVP 必须支持最小局部重生成能力，至少允许对学习要点、题目、导入语、讲解提示和教案片段进行重写。
4. 对普通教师默认隐藏模型配置；模型、Key、额度和成本优先作为管理员或私有化部署能力呈现。
5. 对学生侧输出不只提供资料下载，还应形成“学习要点 → 自测 → 错因解释 → 回看知识点 → 变式练习”的学习闭环。

### 2.5 红绿蓝多 Agent 深度审阅结论

本轮从红队、绿队、蓝队三个视角对当前需求进行审阅，结论是：长期平台愿景完整，但 MVP 仍需进一步回到教师真实心智，避免把工程术语、模型配置、任务系统和社区能力过早推到普通教师面前。首版应从“教学资源全链路平台”收敛为“10 分钟备课助手”：上传课件或输入一句需求后，快速生成可修改、可导出、可分享的学生复习要点、随堂小测、教师讲解提示和教案初稿。

| 视角 | 主要判断 | 必须调整 |
| --- | --- | --- |
| 红队：真实教师心智 | 老师更关心“明天能不能上课”，不关心项目、CourseIR、Schema、任务、Base URL | 教师端采用“上传即生成”，项目、任务、模型、CourseIR 均作为后台概念 |
| 绿队：产品化与商业化 | MVP 范围偏宽，付费价值和增长漏斗不够聚焦 | 首版聚焦学习要点、随堂小测、教师提示、导出/分享；默认托管模型和免费额度 |
| 蓝队：学习与复用 | 文档适合研发执行，但长文档学习门槛高，术语和复用模型需要显性化 | 增加阅读路径、术语映射、教学资源复用模型、内容块颗粒度和质量指标分层 |

#### 2.5.1 面向教师的产品主张

首版对外表达建议统一为：

```text
上传课件，3 分钟生成：
✓ 学生复习要点
✓ 随堂小测与答案解析
✓ 教师讲解提示
✓ 可编辑教案初稿
```

教师端默认流程：

```text
拖入 PPT/PDF 或输入一句备课需求
  → 选择用途：明天上课 / 发学生复习 / 出随堂小测 / 写教案
  → 系统自动识别课程名、学科、年级、课时和学生水平
  → 直接展示可编辑初稿
  → 教师按块修改、用口语指令重写或确认风险项
  → 导出 PDF/DOCX/Markdown 或生成学生分享链接
```

关键原则：

1. 不让普通教师先创建项目；上传或输入需求后自动创建草稿项目。
2. 不让普通教师配置模型；使用系统或机构预配置模型，并展示“剩余额度 / 本次预计消耗 / 升级套餐”。
3. 不让教师看到 CourseIR、Schema、异步任务、Base URL、Token、fork、watch 等技术词；必要时用教学语言替换。
4. 不把 HTML 当作首版核心产物；HTML 优先作为在线预览页和学生分享页承载。
5. 不追求一次生成完美；必须提供局部修改、局部重写、教师审核和版本保留。

#### 2.5.2 用户可见语言与内部术语映射

| 内部/研发术语 | 教师端说法 | 学生端说法 | 管理端/研发保留说法 |
| --- | --- | --- | --- |
| CourseIR | 课程结构 / 知识点大纲 | 本课知识结构 | CourseIR |
| ParsedContent | 课件识别结果 | 资料来源 | ParsedContent |
| Schema 校验 | 格式检查 / 内容完整性检查 | 内容是否完整 | JSON Schema Validation |
| 异步任务 / Queue | 正在生成 / 可稍后查看 | 正在准备学习材料 | generation_tasks / queue |
| source_refs | 来源页码 / 原课件位置 | 来自哪一页课件 | source_refs |
| 局部重生成 | 重新生成这一段 / 换一版 | 换一种解释 / 再出一题 | regenerateSection |
| Agent | AI 备课助手 / AI 助教 | AI 学伴 | Agent |
| fork | 复制为我的版本 | 保存到我的学习空间 | fork |
| watch | 关注更新 / 收藏 | 关注这份资料 | watch |
| Base URL / API Key / Token | 不在教师端展示 | 不展示 | 管理员模型配置 |

#### 2.5.3 MVP 分层与非目标

MVP 应拆成两层，避免一次性承诺标准版能力。

| 层级 | 目标 | 必须包含 | 暂不包含 |
| --- | --- | --- | --- |
| MVP-A：教师备课最小闭环 | 验证老师是否愿意上传课件并使用生成结果 | 上传 PPTX/PDF、自动识别课程信息、学习要点、随堂小测、教师讲解提示、教案初稿、在线预览、局部重写、PDF/DOCX/Markdown 导出 | PPTX 高质量生成、完整教学 HTML 产品、广场、学生账号、复杂 Agent 生命周期、普通教师模型配置 |
| MVP-B：上课包闭环 | 验证老师是否能直接用于第二天课堂 | PPT 大纲或基础 PPTX、教师备注、学生分享链接、小测答题与错因解释、教师审核面板 | 完整社区、机构知识库、多教案 RAG、移动端 App |

MVP 非目标：

1. 不做完整教案广场、公开社区和复杂 UGC 审核体系。
2. 不做高保真还原所有原始 PPT 动效和复杂排版。
3. 不要求普通教师配置模型、Key、Base URL 或模型能力。
4. 不做多模型复杂路由 UI。
5. 不做复杂多 Agent 协作流程；首版 Agent 只服务“生成初稿、局部重写、审核提示”。
6. 不做移动端 App 和完整学生账号体系；可先做免登录分享页。
7. 不做机构知识库批量导入；仅预留后续接口和权限模型。

#### 2.5.4 教师审核面板

为避免“教师确认”停留在原则层，结果页必须提供可扫描、可定位、可处理的审核面板：

1. 顶部摘要：本次生成有多少处需要确认、多少处低置信内容、多少道题建议复核答案。
2. 每个风险项展示 AI 生成内容、来源页码/原文片段、风险原因和建议操作。
3. 操作按钮使用教学语言：保留、修改、重新生成、删除、确认并发布给学生、保存为新版本。
4. 学生可见内容必须有状态：未审核、已教师确认、已发布给学生。
5. 高影响动作，例如发布、覆盖版本、批量推送学生，必须经过 human-in-the-loop 确认。

#### 2.5.5 教学资源复用模型

长期商业化不应只复用整份教案，而应复用教学内容块。建议统一资源生命周期：

```text
原始素材
  → 课件识别结果
  → 课程结构 / 知识点大纲
  → 教案 / 学习要点 / 随堂小测 / 教师提示 / 分享页
  → 内容块编辑与局部重写
  → 版本沉淀
  → 分享 / 复制为我的版本 / 关注更新
  → 学生错因、教师批注、评论和使用反馈
  → 反哺下一版与模板资产
```

内容块最小颗粒度建议：

| 颗粒度 | 示例 | 可编辑 | 可重写 | 可复用 | 可发布 |
| --- | --- | --- | --- | --- | --- |
| 课程 | 牛顿第二定律 | 是 | 是 | 是 | 是 |
| 章节 | 情境导入 | 是 | 是 | 是 | 是 |
| 知识点 | 合外力与加速度 | 是 | 是 | 是 | 是 |
| 教学活动 | 小车实验观察 | 是 | 是 | 是 | 是 |
| 题目 | 单选题 1 | 是 | 是 | 是 | 可选 |
| 解析 | 答案解析 | 是 | 是 | 是 | 可选 |
| 教师话术 | 导入语 / 讲解提示 | 是 | 是 | 是 | 可选 |
| 学生提示 | 错因解释 / 回看知识点 | 是 | 是 | 是 | 可选 |

每个内容块建议包含 `id`、`type`、`title`、`body`、`source_refs`、`knowledge_point_ids`、`version`、`reuse_policy` 和 `review_status`。

#### 2.5.6 商业化与增长指标

首版商业化建议采用“免费试用 → 教师 Pro → 机构版”的阶梯：

| 套餐 | 目标用户 | 计费方式 | 核心权益 |
| --- | --- | --- | --- |
| 免费试用 | 单个教师 | 免费额度 | 每月有限生成次数、基础导出、带品牌分享页 |
| 教师 Pro | 高频教师 | 月付 / 年付 | 更多生成次数、无水印、模板包、批量导出、更多局部重写 |
| 机构版 | 学校 / 教培机构 | 按席位 + 用量 / 年度授权 | 统一模型池、额度、审计、校本模板、组织资源库、私有化部署 |

MVP 必须埋点的增长漏斗：

```text
访问首页
  → 查看示例
  → 上传课件
  → 生成成功
  → 首次预览
  → 局部重写
  → 导出 / 分享
  → 注册 / 付费
  → 第二次复用
```

核心指标建议：

| 指标 | MVP 目标 |
| --- | --- |
| 上传到首版结果可预览时间 | ≤ 3 分钟 |
| 首次用户完成导出率 | ≥ 40% |
| 生成结果无需大改即可使用比例 | ≥ 60% |
| 用户完成一次局部重写比例 | ≥ 30% |
| 首次体验无需配置 API Key | 100% |
| 分享链接创建率 | 持续提升 |
| 学生自测完成率 | 持续提升 |
| 第二次复用率 | 持续提升 |

### 2.6 PRD + TRD 细节化写作要求

后续基于本文档生成执行文档、任务卡、接口文档或代码实现时，不能只保留方向性结论，必须同时写清“用户为什么需要、页面如何呈现、系统如何实现、异常如何处理、如何验收”。每个新增功能都应按以下粒度展开：

| 维度 | PRD 必须写清 | TRD 必须写清 | 生成执行文档时的产出 |
| --- | --- | --- | --- |
| 用户场景 | 谁在什么时间、拿什么素材、要完成什么教学任务 | 入口路由、权限、依赖服务、任务触发条件 | 用户故事、页面流程、接口调用链 |
| 输入信息 | 必填/选填字段、默认值、可推断字段、用户可修改字段 | DTO、Schema、字段来源、校验规则、存储位置 | 表单字段表、API request/response、数据库字段 |
| 系统判断 | 自动识别什么、哪些低置信度要追问、哪些可默认 | 置信度阈值、模型调用、规则引擎、fallback 策略 | 状态机、判定规则、错误码 |
| 输出内容 | 教师/学生实际看到什么、可编辑哪些块、如何导出/分享 | 输出 JSON Schema、内容块 ID、版本、来源引用、渲染模板 | 产物 Schema、模板规则、版本策略 |
| 交互状态 | loading、部分完成、失败、重试、审核、发布等状态文案 | task status、事件流、幂等键、重试策略、通知机制 | 状态流转图、埋点事件、异常处理清单 |
| 质量与安全 | 教师如何审核、学生能否看到、版权/隐私提示 | 敏感信息检测、权限校验、审计日志、模型日志脱敏 | 审核面板需求、安全用例、测试用例 |
| 验收标准 | 用户能完成什么闭环，成功体验是什么 | 自动化测试、集成测试、性能指标、可观测指标 | Definition of Done、测试脚本、监控指标 |

#### 2.6.1 需求详略优先级

1. **教师主链路必须最详细**：上传、识别、生成、预览、局部重写、审核、导出/分享，每一步都要有页面文案、字段、状态、错误处理和验收。
2. **AI 生成链路必须结构化**：任何模型生成都要写清输入变量、Prompt 版本、输出 Schema、校验、修复、重试和人工兜底。
3. **数据链路必须可追溯**：从原始文件页码到 CourseIR，再到内容块、导出文件和学生分享页，必须保留 ID、版本和来源引用。
4. **教师端与管理端必须分离**：教师端描述教学语言与使用流程；管理端描述模型配置、额度、成本、审计和安全。
5. **商业化能力必须可计量**：免费额度、套餐、分享水印、导出限制、模板包、机构席位和用量统计都要绑定埋点和计费口径。

#### 2.6.2 功能任务卡模板

后续拆分 issue 或开发任务时，建议每张任务卡使用以下模板：

```markdown
## 目标
- 本任务让哪个用户在什么场景下完成什么动作。

## 用户流程
1. 用户从哪个入口进入。
2. 用户看到哪些提示、按钮、默认值。
3. 用户提交后系统如何反馈进度。
4. 成功、部分成功、失败分别怎么展示。

## PRD 字段
| 字段 | 类型 | 必填 | 默认值 | 用户可改 | 系统可推断 | 说明 |

## TRD 契约
- API：method + path。
- Request DTO。
- Response DTO。
- 数据表/对象存储路径。
- 任务队列 payload。
- 幂等键。
- 错误码。

## AI 契约
- Prompt key/version。
- 输入变量。
- 输出 Schema。
- 校验和自动修复策略。
- 失败降级策略。

## 权限与安全
- 谁可以读。
- 谁可以写。
- 是否涉及学生数据、版权内容、模型 Key。
- 审计日志字段。

## 埋点与指标
- 事件名。
- 触发时机。
- 属性。
- 成功指标。

## 验收标准
- 正常路径。
- 异常路径。
- 性能/成本。
- 自动化测试或人工验收方式。
```

#### 2.6.3 示例：教师快速生成入口的执行级拆分

| 子任务 | PRD 输出 | TRD 输出 | 验收重点 |
| --- | --- | --- | --- |
| 首屏上传区 | 上传按钮、拖拽区、示例课件入口、用途卡片文案 | `POST /api/quick-start/sessions`、上传签名、文件类型校验 | 教师 1 分钟内能发起生成 |
| 课程信息识别 | 展示“我猜你是……”确认卡、低置信度提示 | 解析任务、课程识别模型调用、confidence 字段 | 不阻断首次生成，低置信度可后改 |
| 初稿生成 | 学习要点、随堂小测、教师提示、教案初稿四个 Tab | `generate-output` 任务、ContentBlock Schema、版本保存 | 能看到可编辑初稿和来源页码 |
| 局部重写 | “改简单点 / 换例子 / 换题 / 重新生成”快捷操作 | `POST /api/content-blocks/{id}/regenerate`、幂等键、Prompt key | 只影响选中块，不覆盖整份结果 |
| 教师审核 | 风险摘要、待确认项、来源片段、一键确认 | review findings 表、审核状态、审计日志 | 发布前必须可确认风险项 |
| 导出/分享 | PDF/DOCX/Markdown 导出、学生链接、有效期 | render 任务、share token、权限策略 | 导出可下载，分享可撤回 |

## 3. 范围定义

### 3.1 MVP 范围

MVP 版本必须包含以下能力：

| 模块 | 功能 |
| --- | --- |
| 教师快速生成入口 | 上传课件或输入一句备课需求后自动创建草稿项目，教师无需先理解“项目” |
| 文件上传 | 上传 PPTX、PDF 文件；图片 OCR 和 DOCX 文本提取作为试验能力或明确降级提示 |
| 文件解析 | 提取 PPTX、PDF 中的文本、图片、页结构 |
| CourseIR | 生成课程结构、章节、知识点、重点难点 |
| 模型配置 | 教师端默认使用系统或机构预配置模型；管理员端支持 OpenAI Compatible 模型配置和连通性测试 |
| 内容生成 | MVP-A 必须生成学习要点、随堂小测、教师讲解提示和教案初稿；HTML 作为在线预览/分享页，PPTX 作为 MVP-B 或增强输出 |
| 结果预览 | 在线预览生成结果，并支持最小局部编辑和重生成 |
| 导出下载 | 优先下载 Markdown、PDF、DOCX；ZIP、HTML 静态包、PPTX 根据模板成熟度逐步开放 |
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
10. 作为教师，即使我还没有现成教案，我也希望通过 Agent 从课件或零散素材开始生成教案，并经历确认、编辑、审校、发布、复用的完整生命周期。

## 5. 总体业务流程

```text
上传课件或创建课程项目
  → 系统自动识别课程信息，用户补充必要教学上下文
  → 文件解析
  → 内容清洗与结构化
  → 生成 CourseIR
  → 用户确认或修改课程结构
  → 选择输出用途：上课讲解 / 学生复习 / 随堂测试 / 标准教案
  → 模型生成结构化结果
  → Schema 校验与质量审校
  → 模板渲染 HTML/PPTX/PDF/DOCX
  → 在线预览、局部编辑与局部重生成
  → 导出、分享或发布到广场
  → 其他用户 fork / watch / chat
```

### 5.1 无现成教案时的 Agent 生成生命周期

当用户没有可复用教案时，系统应允许从课件、教材片段、教学目标或一句自然语言需求开始，由专门的“教案生成 Agent”创建教案并进入完整生命周期。该 Agent 不替代教师决策，而是负责补齐初稿、暴露待确认项、驱动审校和沉淀版本。

```text
输入起点：PPTX/PDF/Word/Markdown/图片/文本需求
  → Agent 识别教学意图、学科、年级、课时、学生水平和输出用途
  → 缺失信息追问或使用可解释默认值
  → 解析素材并生成 CourseIR 草稿
  → 生成教案草稿、学习要点、测试题和课堂活动建议
  → 自动标记不确定内容、来源页码和需要教师确认的假设
  → 教师预览、批注、局部重生成和确认
  → 质量审校：事实一致性、题目答案、难度匹配、课堂时间分配
  → 形成教案版本并可导出、分享、发布到广场或进入 Chat 使用
  → 后续基于课堂反馈、学生错题、评论、fork/watch 数据继续迭代
```

生命周期状态建议：

| 状态 | 说明 | 关键动作 |
| --- | --- | --- |
| draft_requested | 用户提出生成需求或上传素材 | 创建生成任务、记录输入来源 |
| context_collecting | Agent 收集或推断教学上下文 | 追问缺失信息、读取课程元数据 |
| ir_building | 构建 CourseIR 草稿 | 解析素材、抽取知识点、关联来源页码 |
| lesson_generating | 生成教案和配套资源 | 生成教案、学习要点、测试题、课堂活动 |
| teacher_reviewing | 教师审核与局部修改 | 批注、局部重生成、确认假设 |
| quality_reviewing | 系统质量审校 | 检查事实、答案、难度、时间分配和格式 |
| versioned | 形成可追溯版本 | 保存版本、记录 Prompt/模型/来源 |
| published_or_shared | 发布或分享 | 导出、分享链接、发布广场、推送学生 |
| iterating | 基于反馈持续迭代 | 根据课堂反馈、错题、评论和 fork 数据更新 |

关键验收标准：

1. 没有现成教案时，教师可以只上传课件或输入一句需求启动教案生成。
2. Agent 必须显式列出使用了哪些素材、哪些信息来自推断、哪些内容需要教师确认。
3. 生成教案必须关联 CourseIR、来源页码、版本号和质量审校结果。
4. 教师可以对任意章节、活动、题目、导入语或作业进行局部重生成。
5. 教案进入发布、分享、覆盖原版本、推送学生等高影响动作前，必须经过 human-in-the-loop 确认。

## 6. 产品功能需求

### 6.0 教师端快速生成入口

#### 功能说明

教师端首屏应优先呈现“上传课件，生成教学材料”，而不是“创建项目”。用户可以拖入 PPTX/PDF，或输入一句自然语言需求，例如“我要准备高一物理牛顿第二定律的随堂小测”。系统自动创建草稿项目、识别课程信息并生成可编辑初稿。

#### 首次使用流程

```text
上传课件或输入需求
  → 选择用途：明天上课 / 发学生复习 / 出随堂小测 / 写教案
  → 系统解析并自动识别课程信息
  → 直接生成低风险初稿
  → 在结果顶部展示“我猜你是：学科 / 年级 / 课时 / 学生水平，可修改”
  → 教师修改后局部重写或导出分享
```

#### 字段策略

| 字段 | 教师端策略 | 说明 |
| --- | --- | --- |
| 输出用途 | 首次唯一建议必选 | 用场景卡片替代表单字段 |
| 课程名称 | 自动识别，允许修改 | 低置信度时再追问 |
| 学科 | 自动识别，允许修改 | 不作为上传前置阻断 |
| 年级 | 自动识别或默认普通难度 | 低置信度时展示“可补充” |
| 课时 | 默认 45 分钟 | 可在结果页调整并局部重写 |
| 学生水平 | 默认普通 | 可用“基础 / 普通 / 提高”快速切换 |

#### 验收标准

1. 教师无需先创建项目即可上传课件并看到生成结果。
2. 上传后系统自动创建草稿项目并保存文件、识别结果和生成历史。
3. 教师首次体验不需要配置 API Key、Base URL 或模型名称。
4. 课程信息识别置信度低时，以一句自然语言追问或结果页提示处理，不用完整表单阻断。
5. 生成过程使用教师可理解文案，例如“正在解析课件”“已生成学习要点”“测试题还在生成中”。

#### 页面组成与交互细节

| 区域 | 默认展示 | 交互 | 异常/空状态 |
| --- | --- | --- | --- |
| 顶部价值说明 | “上传课件，生成复习要点、随堂小测和讲解提示” | 点击“看示例”进入示例项目 | 未登录用户可先看示例，导出时提示登录 |
| 上传区 | 支持拖拽 PPTX/PDF，显示大小限制和隐私提示 | 选择文件后立即校验类型、大小、是否空文件 | 不支持格式时提示“请另存为 PDF/PPTX 后上传” |
| 用途卡片 | 明天上课、发学生复习、出随堂小测、写教案 | 单选，默认选中“明天上课”；高级设置可展开 | 未选择时按钮不可提交并解释用途影响输出 |
| 高级设置 | 默认收起 | 可填写年级、课时、学生水平、模板偏好 | 可全部留空，系统使用可解释默认值 |
| 生成进度 | 解析课件、识别课程、生成初稿、质量检查 | 分阶段展示已完成产物，允许关闭页面稍后查看 | 失败时定位到文件页码或生成步骤 |
| 结果确认卡 | “我猜你是：学科/年级/课时/学生水平” | 修改后可选择“仅更新后续生成”或“重新生成全部” | 低置信字段用黄色提示，不阻塞预览 |

#### 教师端状态文案

| 内部状态 | 教师端文案 | 可操作动作 |
| --- | --- | --- |
| `uploading` | 正在上传课件 | 取消上传 |
| `parsing` | 正在识别课件内容，预计 1 分钟 | 稍后查看 |
| `course_detecting` | 正在识别课程信息 | 跳过并先生成 |
| `generating_notes` | 正在生成学生复习要点 | 查看已完成内容 |
| `generating_quiz` | 正在生成随堂小测和解析 | 查看学习要点 |
| `reviewing_quality` | 正在检查题目答案和来源页码 | 查看草稿 |
| `partial_success` | 部分内容已生成，少量页面识别不清 | 跳过问题页 / 重新上传 / 手动补充 |
| `failed` | 生成失败：说明原因和建议处理方式 | 重试 / 更换文件 / 联系支持 |

#### 前后端契约建议

快速生成入口可以由 API 创建一个 `quick_start_session`，再串联文件上传、解析、CourseIR 构建和输出生成任务。

```http
POST /api/quick-start/sessions
GET /api/quick-start/sessions/{session_id}
POST /api/quick-start/sessions/{session_id}/files
POST /api/quick-start/sessions/{session_id}/confirm-context
POST /api/quick-start/sessions/{session_id}/generate
```

请求示例：

```json
{
  "intent": "teach_tomorrow",
  "preferred_outputs": ["study_notes", "quiz", "teacher_tips", "lesson_plan_draft"],
  "optional_context": {
    "subject": null,
    "grade": null,
    "duration_minutes": 45,
    "student_level": "normal",
    "teaching_scenario": "regular_class"
  },
  "client_trace_id": "qs_20260626_001"
}
```

响应示例：

```json
{
  "session_id": "qss_001",
  "project_id": "proj_001",
  "status": "waiting_for_file",
  "created_draft_project": true,
  "next_action": "upload_file",
  "teacher_visible_message": "请上传 PPTX 或 PDF，我们会先生成可编辑初稿"
}
```

生成状态响应应同时包含教师可见状态和内部状态：

```json
{
  "session_id": "qss_001",
  "project_id": "proj_001",
  "status": "generating",
  "current_step": "generating_quiz",
  "teacher_visible_message": "学习要点已生成，随堂小测还在生成中",
  "progress_percent": 65,
  "available_outputs": ["study_notes"],
  "detected_context": {
    "subject": { "value": "物理", "confidence": 0.94, "needs_confirmation": false },
    "grade": { "value": "高一", "confidence": 0.72, "needs_confirmation": true },
    "duration_minutes": { "value": 45, "confidence": 0.6, "needs_confirmation": true }
  },
  "recoverable_errors": []
}
```

#### 任务与数据落地

1. `quick_start_session` 只保存首次生成会话状态、用途、可选上下文和前端 trace ID。
2. 上传成功后自动创建 `projects` 草稿记录，后续所有文件、CourseIR 和输出仍归属项目，避免产生两套业务模型。
3. 解析、识别、生成、渲染仍进入 `generation_tasks` 或队列系统；前端通过 session 聚合查询进度。
4. 每次局部重写必须创建新的 ContentBlock 版本，不能直接覆盖原内容。
5. 所有模型调用必须经过 `packages/ai-gateway`，记录 prompt_key、prompt_version、模型、token、成本、延迟和失败原因。

#### 埋点事件

| 事件名 | 触发时机 | 关键属性 |
| --- | --- | --- |
| `quick_start_viewed` | 用户打开快速生成页 | user_id、source、is_logged_in |
| `sample_project_opened` | 点击示例课件 | sample_subject、sample_grade |
| `quick_start_intent_selected` | 选择用途卡片 | intent |
| `file_upload_started` | 开始上传 | file_type、file_size |
| `file_upload_failed` | 上传失败 | reason、file_type、file_size |
| `first_preview_shown` | 首次可预览内容出现 | time_to_preview_ms、available_outputs |
| `context_confirmed` | 教师确认或修改课程信息 | changed_fields、confidence_before |
| `content_block_regenerated` | 局部重写完成 | block_type、prompt_action、latency_ms |
| `output_exported` | 导出文件成功 | output_type、file_format |
| `share_link_created` | 创建学生分享链接 | visibility、expires_in_days |

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

Agent 面向教师备课、课堂组织和学生自学场景，覆盖“无教案生成”“有教案问答”“教案迭代”“学生学习引导”四类链路。系统应至少包含教案生成 Agent 和教学引导 Agent：前者负责从课件或需求生成教案并驱动生命周期，后者基于已有教案、CourseIR、测试卡和学习要点进行 Chat、答疑、练习推荐和内容改写。

#### 推荐能力边界

1. 意图识别：区分备课、从零生成教案、讲解、答疑、练习、纠错、复习、教研评价等意图。
2. 无教案生成：当缺少教案时，教案生成 Agent 应从课件、教材片段或自然语言需求出发，补齐教学上下文、生成 CourseIR 和教案草稿。
3. 教案问答：已有教案时，教学引导 Agent 优先基于 CourseIR、教案、测试卡、学习要点和来源页码回答。
4. 教学引导：针对学生回答进行追问、提示、分层解释、错因分析和变式练习推荐。
5. 教师助手：帮助教师改写导入语、生成课堂提问、调整难度、补充活动和重排课堂时间。
6. 生命周期推进：Agent 需要驱动 draft_requested、context_collecting、ir_building、lesson_generating、teacher_reviewing、quality_reviewing、versioned、published_or_shared、iterating 等状态流转。
7. 结果沉淀：将高质量对话片段沉淀为教案备注、FAQ、新的课堂活动或下一版教案修改建议。

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
8. MVP 建议：先实现“无教案时从课件生成教案初稿 + 引用来源 + 教师确认 + 局部重生成 + 保存版本”，再实现“围绕单个教案 Chat + 生成课堂提问/练习 + 保存为备注”，最后扩展到多教案检索、自动 fork 改写和机构级知识库。


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

### 7.4 分阶段模块拆分与落地顺序

为了避免一开始就把所有能力耦合在单个应用中，建议按“先骨架、再闭环、再协作、再机构化”的节奏拆分模块。每个阶段都应形成可运行、可演示、可验收的最小纵切。

#### 7.4.1 第 0 阶段：工程骨架

| 模块 | 目录 | 核心职责 | 首批交付物 |
| --- | --- | --- | --- |
| Web 壳应用 | `apps/web` | 登录后工作台、项目列表、基础路由、布局和主题 | 首页、项目列表空状态、全局导航 |
| API 壳服务 | `apps/api` | 健康检查、统一错误、鉴权中间件、OpenAPI/RPC 约定 | `/health`、鉴权骨架、错误码规范 |
| Worker 壳进程 | `apps/worker` | 队列连接、任务注册、任务状态上报 | 示例任务、重试策略、任务日志 |
| 共享配置 | `packages/config` | TS、Lint、格式化、测试、构建配置 | 统一脚本和基础规范 |
| 领域模型 | `packages/domain` | 用户、项目、文件、输出、任务的基础类型 | 领域实体和枚举 |
| 存储访问 | `packages/storage` | 数据库、Redis、对象存储客户端封装 | 连接池、事务工具、Repository 基类 |

验收重点：本地可以启动 Web、API、Worker；提交代码前可以运行格式化、类型检查和基础测试。

#### 7.4.2 第 1 阶段：上传解析与 CourseIR 闭环

| 模块 | 目录 | 核心职责 | 首批交付物 |
| --- | --- | --- | --- |
| 项目管理 | `apps/api` + `packages/domain` | 创建课程项目、复制、归档、基础元数据 | 项目 CRUD、项目列表 |
| 文件上传 | `apps/web` + `apps/api` + `packages/storage` | 上传 PPTX/PDF、校验类型大小、写入对象存储 | 上传页、文件记录、解析任务创建 |
| 文件解析 | `apps/worker` + `packages/parser` | 提取文本、图片、页结构和来源页码 | PPTX/PDF parser、ParsedContent |
| CourseIR | `packages/course-ir` | 课程结构 Schema、版本、校验和迁移 | CourseIR JSON Schema、版本表适配 |
| AI 网关 | `packages/ai-gateway` | OpenAI Compatible 接入、Key 管理、调用日志 | 模型配置、连通性测试、调用审计 |

验收重点：上传一个 PPTX/PDF 后，可以异步解析并生成可查看、可校验、可追溯来源页码的 CourseIR。

#### 7.4.3 第 2 阶段：内容生成与预览导出

| 模块 | 目录 | 核心职责 | 首批交付物 |
| --- | --- | --- | --- |
| 内容生成 | `packages/generator` | 学习要点、测试卡、教案、HTML/PPT 结构 JSON | 分类型 Prompt、Schema 校验、质量检查 |
| HTML 渲染 | `packages/renderer-html` + `apps/web` | 教学页面预览、静态 HTML 打包、PDF 导出入口 | 预览页、主题模板、导出包 |
| PPT 渲染 | `packages/renderer-ppt` | Slide JSON 到 PPTX | PPT 模板、主题、版式映射 |
| 结果管理 | `apps/api` + `packages/storage` | 输出结果版本、下载、局部重生成入口 | 输出列表、详情、下载接口 |
| 在线编辑 | `apps/web` + `packages/ui` | 结果编辑、局部重生成按钮、版本对比入口 | 编辑器、保存、恢复 |

验收重点：用户可以从 CourseIR 生成学习要点、测试卡、教学 HTML、PPTX，并完成预览、编辑和下载。

#### 7.4.4 第 3 阶段：教案广场与资源协作

| 模块 | 目录 | 核心职责 | 首批交付物 |
| --- | --- | --- | --- |
| 广场领域 | `packages/community` | 资源发布、可见性、fork/watch、评论、资源谱系 | 发布规则、资源关系、审核状态 |
| 广场 API | `apps/api` | 资源列表、详情、fork、watch、评论、举报 | `/api/community/resources` 相关接口 |
| 广场 Web | `apps/web` | 资源发现、详情页、fork/watch 入口、作者主页 | 广场首页、筛选、详情 |
| 审计与通知 | `apps/api` + `apps/worker` | 行为日志、更新通知、举报处理 | 审计日志、站内通知 |

验收重点：用户可以发布教案到广场，其他用户可以检索、查看、fork、watch，并能看到资源来源和衍生关系。

#### 7.4.5 第 4 阶段：Agent 与 Chat 教学引导

| 模块 | 目录 | 核心职责 | 首批交付物 |
| --- | --- | --- | --- |
| Agent 服务 | `apps/agent` | Mastra Agent、Workflow、Memory、观测和会话编排 | 教案 Chat、意图识别、引用回答 |
| Agent 工具 | `packages/agent-tools` | 教案检索、CourseIR 查询、题目生成、局部重生成、fork/watch | Tool Schema、权限校验、审计记录 |
| 检索增强 | `packages/storage` + `packages/course-ir` | 向量索引、片段切分、来源引用、重排 | pgvector 索引、检索 API |
| Chat UI | `apps/web` | 教案详情页 Chat、教师助手、学生答疑 | 对话面板、引用卡片、操作确认 |
| 人机确认 | `apps/api` + `apps/agent` | 发布、覆盖、推送、批量生成等高影响动作确认 | confirmation token、审批记录 |

验收重点：用户能围绕单个教案发起 Chat，Agent 能基于教案和 CourseIR 回答、引用来源、生成练习，并在写入前要求确认。

#### 7.4.6 第 5 阶段：移动端与机构能力

| 模块 | 目录 | 核心职责 | 首批交付物 |
| --- | --- | --- | --- |
| 移动端 | `apps/app` | 学生查看学习要点、测试卡、watch 更新和轻量 Chat | 只读资源、消息提醒、答疑入口 |
| 多租户 | `apps/api` + `packages/domain` | 组织、学校、班级、角色、权限、额度 | 租户模型、RBAC、额度策略 |
| 机构知识库 | `packages/storage` + `apps/worker` | 教材、考纲、题库、校本资源入库和索引 | 批量导入、检索、权限隔离 |
| 运维治理 | `infra` | 环境、部署、监控、备份、安全策略 | 部署模板、告警、备份恢复 |

验收重点：支持学校/机构级资源、权限、模型池、成本统计和私有化部署基础能力。

#### 7.4.7 每个模块的标准交付清单

1. `README.md`：说明模块职责、边界、启动方式、依赖和常用命令。
2. `src/index.ts` 或服务入口：提供稳定导出，避免跨包导入内部实现。
3. `schema` 或 `types`：所有外部输入输出必须有类型和 Schema。
4. `tests`：核心业务规则、Schema 校验、工具函数和关键服务必须有测试。
5. `fixtures`：解析、生成、渲染和 Agent Tool 必须提供最小样例数据。
6. `observability`：API、任务、模型调用、Agent Tool 调用必须记录可追踪日志。
7. `security`：涉及文件、Key、权限、发布、写入和外部模型调用的模块必须定义安全检查点。



### 7.5 模块级目录拆分规范

本节把仓库目录进一步拆到后续可直接创建 issue、任务卡和迭代分支的粒度。原则是：`apps/*` 只组合产品流程和部署入口，`packages/*` 沉淀可复用能力；所有跨模块调用先通过公开入口、Schema 和 API 契约，不直接引用内部实现。

#### 7.5.1 `apps/web`：Web 前端应用

```text
apps/web/
  src/
    app/                 路由、布局、页面入口
    features/            按业务域组织的页面级功能
      projects/          项目列表、项目详情、创建/复制/归档
      upload/            文件上传、解析状态、失败重试入口
      editor/            CourseIR、教案、学习要点、测试卡编辑
      preview/           HTML/PPT/Markdown 预览和下载
      community/         广场列表、资源详情、fork/watch/comment
      chat/              教案 Chat、引用卡片、人机确认面板
    components/          当前应用专属组件
    hooks/               Web 专属 hooks
    lib/                 路由守卫、错误处理、埋点、权限 helper
    styles/              全局样式和主题挂载
```

边界要求：Web 只调用 `packages/api-client`，不直接访问数据库、对象存储、模型 SDK 或队列；复杂表单输入必须复用 `packages/domain`、`packages/course-ir` 或 API Schema。

#### 7.5.2 `apps/api`：核心业务 API

```text
apps/api/
  src/
    modules/
      auth/              登录态、用户上下文、RBAC/ABAC 策略入口
      projects/          项目 CRUD、复制、归档、成员关系
      files/             上传签名、文件记录、解析任务创建
      course-ir/         CourseIR 版本读取、保存、迁移和对比
      generation/        生成任务编排、输出结果、局部重生成入口
      model-providers/   模型供应商配置、Key 加密、连通性测试
      community/         发布、检索、fork、watch、评论、举报、谱系
      agent/             Agent session、message、confirmation 受控接口
      audit/             审计日志、行为追踪、合规导出
    common/              错误码、响应结构、中间件、日志、配置
    contracts/           OpenAPI/RPC 契约和 DTO Schema
```

边界要求：API 是业务写入的唯一同步入口；所有写入必须记录 `actor_id`、资源 ID、来源动作和审计日志；Agent 或 Worker 不能绕过 API 直接改核心业务状态。

#### 7.5.3 `apps/agent`：Mastra Agent 服务

```text
apps/agent/
  src/
    agents/              teacherAssistant、studentTutor、reviewAgent
    workflows/           intent-routing、lesson-chat、regenerate-with-confirm
    tools/               Mastra Tool 注册和 packages/agent-tools 适配
    memory/              会话摘要、短期记忆、用户确认偏好
    rag/                 检索策略、引用组装、重排和片段压缩
    guardrails/          权限、敏感内容、写入确认、来源优先级
    observability/       trace、tool call、token/cost/latency 记录
```

边界要求：`apps/agent` 负责编排和对话体验，不拥有业务数据库写入逻辑；写入教案、fork/watch、发布、推送、保存备注等动作必须调用 `apps/api` 的受控接口，并在高影响动作前生成 confirmation token。

#### 7.5.4 `apps/worker`：异步任务进程

```text
apps/worker/
  src/
    queues/              队列连接、任务名称、重试/死信策略
    jobs/
      parse-file/        文件解析任务
      build-course-ir/   CourseIR 生成和校验任务
      generate-output/   学习要点、测试卡、教案、HTML/PPT 结构生成
      render-output/     HTML/PPTX/PDF/DOCX/ZIP 渲染导出
      index-resource/    资源向量索引和重建
      notify/            watch 更新、任务完成、举报处理通知
    runtime/             worker 启停、并发控制、心跳和日志
```

边界要求：Worker 只处理可重试、可恢复的长任务；任务输入必须是轻量 ID 和版本号，不传递大 JSON；任务结果写入必须幂等，并更新任务进度、错误原因和可重试状态。

#### 7.5.5 核心 `packages/*` 拆分

| 包 | 建议子目录 | 第一批能力 | 不应包含 |
| --- | --- | --- | --- |
| `packages/domain` | `entities/`、`value-objects/`、`policies/`、`events/` | Project、File、Output、Task、UserRole、Permission 等纯领域模型 | Web 路由、数据库客户端、模型 SDK |
| `packages/course-ir` | `schema/`、`validators/`、`migrations/`、`fixtures/` | CourseIR JSON Schema、版本迁移、来源引用校验 | 具体 UI 编辑器和 API controller |
| `packages/parser` | `pptx/`、`pdf/`、`docx/`、`markdown/`、`ocr/`、`normalizers/` | ParsedContent 标准结构、页码/图片/表格提取 | 业务项目状态写入 |
| `packages/ai-gateway` | `providers/`、`routing/`、`credentials/`、`logging/`、`schemas/` | OpenAI Compatible 接入、模型路由、Key 解密、调用审计 | 页面 Prompt、Agent Workflow |
| `packages/generator` | `prompts/`、`planners/`、`schemas/`、`quality/` | 学习要点、测试卡、教案、HTML/PPT JSON 生成 | 文件解析和最终文件渲染 |
| `packages/renderer-html` | `templates/`、`themes/`、`exporters/` | HTML 预览结构渲染、静态包、PDF 导出适配 | 模型调用和业务权限判断 |
| `packages/renderer-ppt` | `layouts/`、`themes/`、`assets/`、`exporters/` | Slide JSON 到 PPTX、主题和版式映射 | CourseIR 生成逻辑 |
| `packages/agent-tools` | `schemas/`、`tools/`、`adapters/`、`audit/` | searchLessonPlan、getCourseIR、generateQuiz、regenerateSection、fork/watch | Mastra Agent 主流程和 UI |
| `packages/community` | `policies/`、`lineage/`、`ranking/`、`moderation/` | 发布可见性、fork/watch 规则、谱系、热度排序、审核 | API 路由和前端页面 |
| `packages/storage` | `db/`、`repositories/`、`object-store/`、`cache/`、`vector/` | PostgreSQL、Redis、S3/MinIO、pgvector 基础访问 | 复杂业务编排和权限策略 |
| `packages/api-client` | `generated/`、`clients/`、`errors/` | 类型安全 API SDK、错误映射、请求上下文 | 业务页面状态和数据库访问 |
| `packages/ui` | `components/`、`tokens/`、`icons/`、`editor-kit/` | 通用 UI、主题 token、基础编辑组件 | 业务数据获取 |

#### 7.5.6 模块任务卡模板

后续每个模块进入开发前，建议用同一模板拆任务：

1. **目标**：本迭代要让哪个用户流程可演示。
2. **输入/输出契约**：API DTO、Tool Schema、队列 payload、数据库表或 JSON Schema。
3. **依赖模块**：只列公开入口，例如 `@tcg/course-ir` 或 `/api/projects/{id}`。
4. **数据与权限**：需要读写的数据表、对象存储路径、角色限制和审计字段。
5. **失败与重试**：错误码、幂等键、重试策略、人工介入入口。
6. **测试夹具**：最小 PPTX/PDF、CourseIR fixture、Prompt fixture、渲染 fixture。
7. **验收方式**：页面路径、API 请求、任务日志、导出文件或 Agent 对话样例。


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
用户消息 + 当前教案上下文或原始课件素材
  → 权限校验
  → 意图识别：生成教案 / 教案问答 / 学生答疑 / 局部改写 / 练习推荐
  → 判断是否已有可用教案
  → 无教案：启动教案生成 Agent，收集上下文、构建 CourseIR、生成教案草稿和配套资源
  → 有教案：检索 CourseIR / 教案 / 测试卡 / 学习要点
  → 选择工具或生成策略
  → 输出教学引导或待确认的教案变更
  → 记录来源、假设、反馈和可沉淀内容
```

Agent 不应直接绕过业务系统修改最终教案；涉及写入教案、生成新题目、发布到广场、覆盖版本、推送学生等动作时，需要通过受控 Tool 调用、保留版本记录，并在高影响动作前要求教师确认。无教案生成链路中的每次推断、追问、素材引用、审校结果和人工确认都应进入可审计日志。

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

### 12.4 教育数据与版权合规

1. 上传内容应自动检测学生姓名、手机号、成绩、班级等敏感信息，并在发布或分享前提示教师处理。
2. 默认分享链接应支持有效期、访问范围、撤回和访问记录；面向学生的免登录链接只采集最小必要数据。
3. 发布到广场或组织资源库前必须进行版权和隐私检查，教材、教辅、校本资料需要标注来源和授权范围。
4. 学生 Chat 记录默认不公开，不用于公开广场排序；用于教学改进前应进行匿名化或获得授权。
5. 对外发布、覆盖原版本、批量推送学生等动作必须经过教师或机构管理员确认。
6. 公开资源需保留 AI 生成声明、教师审核状态、来源引用和纠错入口。

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

建议建立人工或半自动评估集，并将质量指标拆成教学质量、工程质量和产品转化三组。

#### 教学质量指标

1. 知识点覆盖率。
2. 先备知识识别准确率。
3. 教学目标可测量性。
4. 重点难点匹配度。
5. 易错点覆盖率。
6. 题目答案正确率。
7. 题目解析有效性。
8. 学生自测后的回看路径完整度。
9. 教师可用性评分。

#### 工程质量指标

1. Schema 通过率。
2. 渲染成功率。
3. 导出成功率。
4. 局部重写成功率。
5. 来源页码覆盖率。
6. Agent 引用准确率。
7. 任务重试成功率。
8. 平均生成成本。

#### 产品转化指标

1. 首页访问到上传转化率。
2. 上传到首次预览成功率。
3. 上传到首版结果可预览时间。
4. 预览后局部重写比例。
5. 导出完成率。
6. 分享链接创建率。
7. 学生自测完成率。
8. 教师第二次复用率。
9. 升级点击率和付费转化率。

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

#### MVP-A：教师备课最小闭环验收

1. 教师无需先创建项目，可以上传 PPTX/PDF 或输入一句备课需求。
2. 系统可以解析上传文件并生成结构化识别结果，内部形成课程结构草稿（CourseIR）。
3. 系统可以自动识别课程名称、学科、年级、课时和学生水平，并允许教师在结果页修改。
4. 系统可以生成学习要点、随堂小测、答案解析、教师讲解提示和教案初稿。
5. 用户可以在线预览、按内容块编辑，并用“改简单点”“换几道基础题”“重新生成这一段”等口语指令局部重写。
6. 用户可以导出 PDF、DOCX 或 Markdown。
7. 教师首次体验无需配置 OpenAI Compatible 模型、API Key、Base URL 或模型名称。
8. 生成过程展示教师可理解的进度和失败原因，例如“正在解析课件”“第 12 页识别不清，可跳过或重新上传”。

#### MVP-B：上课包闭环验收

1. 系统可以生成 PPT 大纲或基础 PPTX，并包含教师备注、课堂提问页和练习页。
2. 教师可以生成学生分享链接，学生无需登录即可查看学习要点并完成一次 3-5 题自测。
3. 学生自测后可以看到答案解析、错因解释和回看知识点。
4. 教师可以在审核面板看到待确认项、低置信内容、题目答案复核建议和来源页码。
5. 发布给学生、覆盖版本、批量推送等高影响动作必须经过教师确认。

#### MVP 增强但不阻塞真实试用

1. 教学 HTML 独立页面。
2. 高质量 PPTX 模板渲染。
3. 完整广场、fork/watch、评论和复杂 Chat。
4. 多模型供应商复杂路由 UI。

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
用户上传一个 PPTX 或 PDF，或输入一句“我要准备某节课”的自然语言需求
  → 系统解析素材并自动识别课程信息
  → 系统生成 CourseIR
  → 如果没有现成教案，教案生成 Agent 生成教案初稿和待确认假设
  → 系统生成学习要点、随堂测试题和教师讲解提示
  → 用户在线预览、局部编辑和局部重生成
  → 系统完成质量审校并保存可追溯版本
  → 用户导出 PDF/Markdown/DOCX/ZIP，或分享给学生
```

只要该闭环稳定可用，就可以进入真实教师试用阶段，并基于反馈继续优化解析质量、模板质量、生成质量、教案生命周期和学生学习闭环。教学 HTML、PPTX、广场、fork/watch 和复杂 Agent 协作应在该闭环验证后逐步扩展。
