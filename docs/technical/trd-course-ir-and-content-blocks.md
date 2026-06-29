# TRD：CourseIR 与 ContentBlock

## CourseIR 定位

CourseIR 是内部统一课程中间表示。教师端不得直接出现 CourseIR，应展示为“课程结构”或“知识点大纲”。

## CourseIR 建议字段

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

## ContentBlock 定位

ContentBlock 是可编辑、可重写、可复用、可审核的最小教学内容单位。

## ContentBlock 建议字段

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
| `review_status` | 未审核、已确认、已发布 |

## 版本策略

1. 每次局部重写必须创建新的 ContentBlock 版本，不能直接覆盖原内容。
2. 导出和分享必须记录使用的内容版本。
3. 发布给学生前必须检查 `review_status`。
