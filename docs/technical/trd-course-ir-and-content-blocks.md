# TRD：CourseIR 与 ContentBlock

## CourseIR 定位

CourseIR 是内部统一课程中间表示。教师端不得直接出现 CourseIR，应展示为“课程结构”或“知识点大纲”。

## CourseIR 必需字段

| 字段 | 说明 |
| --- | --- |
| `id` | 课程结构 ID |
| `title` | 课程标题 |
| `subject` | 学科 |
| `grade` | 年级 |
| `duration_minutes` | 课时 |
| `student_level` | 学生水平 |
| `chapters` | 章节结构 |
| `knowledge_points` | 知识点 |
| `source_refs` | 来源页码和原文片段 |
| `confidence` | 自动识别置信度 |
| `version` | 版本号 |
| `schema_version` | JSON Schema 版本，例如 `1.0.0` |

`source_refs` 是带 `source_kind` 的联合类型：

- 文件来源：`source_kind=file`、`file_id`、从 1 开始的 `page_index`、可选 `quote` 和 `confidence`。
- 文本来源：`source_kind=source_text`、项目级 `text_source_id`、可选从 0 开始且左闭右开的 `char_start/char_end`、`quote` 和 `confidence`。

接受生成请求时，文本模式和文件补充说明必须从临时会话复制为项目级加密文本来源；会话记录清理不能破坏项目版本的来源追溯。CourseIR Schema 发布后，同一主版本只增加可选字段；破坏性变化必须提供迁移函数和 fixture。

## ContentBlock 定位

ContentBlock 是可编辑、可重写、可复用、可审核的最小教学内容单位。

## ContentBlock 必需字段

| 字段 | 说明 |
| --- | --- |
| `id` | 内容块 ID |
| `type` | 内容块类型，如 note、quiz、answer、teacher_tip、lesson_plan_section |
| `title` | 标题 |
| `body` | 正文 |
| `source_refs` | 来源页码 |
| `knowledge_point_ids` | 关联知识点 |
| `version` | 版本 |
| `reuse_policy` | 复用策略 |
| `review_status` | 当前版本的审核投影；历史事实保存在 ContentBlockVersion |

MVP-A 的 `review_status` 只使用 `unreviewed`、`confirmed`；`published` 从 MVP-B 引入。正文使用带 `format` 的 JSON 对象，不允许同一字段混用纯文本、Markdown 和富文本文档。

## 版本策略

1. 每次生成、编辑、局部重写、删除、确认和恢复都创建新的 ContentBlockVersion，不能直接覆盖原内容或只修改父记录状态。
2. ContentBlockVersion 保存正文、来源、Prompt/任务、`review_status` 和可选 tombstone；编辑、重写和恢复后的新版本默认 `unreviewed`。
3. 导出和分享必须记录使用的 OutputVersion；OutputVersion 保存 ContentBlock ID + version 清单、CourseIR 版本和审核投影。
4. 发布给学生前必须检查清单内每个具体版本的 `review_status` 及 blocking finding。
5. 编辑使用 `expected_version` 乐观锁；恢复旧版本复制内容并创建新版本，不把指针回拨。
6. 内容块变更锁定所属 output，基于锁内最新 manifest 创建下一 OutputVersion；不同块并发修改不得丢失彼此结果。
