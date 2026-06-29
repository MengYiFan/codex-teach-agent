# 用户可见语言映射

## 使用原则

- 教师端说教学任务语言，不说研发语言。
- 学生端说学习行为语言，不说平台语言。
- 管理端和研发文档可以保留内部术语，但必须和用户语言建立映射。

## 术语映射

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
| 项目 | 备课草稿 / 这节课 | 学习资料 | project |
| HTML 静态包 / ZIP | 在线预览 / 分享链接 | 打开学习页 | static export |

## 教师端禁用词

MVP 教师端不应出现：

- CourseIR
- Schema
- JSON
- 异步任务
- Queue
- Base URL
- API Key
- Token
- fork
- watch
- Agent Workflow
- RAG
- Tool
- Memory

## 教师端推荐词

MVP 教师端推荐使用：

- 上传课件
- 生成复习要点
- 生成随堂小测
- 生成讲解提示
- 生成教案初稿
- 修改这一段
- 换一种说法
- 换几道题
- 来源页码
- 需要确认
- 稍后查看
