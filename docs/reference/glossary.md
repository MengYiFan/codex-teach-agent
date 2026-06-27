# 术语表

| 术语 | 非技术解释 | 技术解释 |
| --- | --- | --- |
| 课程结构 | 系统从课件中识别出的章节、知识点和重点难点 | CourseIR 的用户侧表达 |
| 课件识别结果 | 系统读懂课件后的文本、图片、页码和结构 | ParsedContent |
| 内容块 | 可单独编辑、重写、复用和审核的一小段教学内容 | ContentBlock |
| 来源页码 | 生成内容对应的原课件位置 | source_refs |
| 局部重写 | 只重写某一段、某道题或某个讲解提示 | regenerate section/block |
| 模型网关 | 统一调用不同大模型的后台服务 | ai-gateway |
| 生成任务 | 解析、生成、渲染等耗时任务的后台执行记录 | generation_tasks |
| 审核状态 | 内容是否已被教师确认或发布 | review_status |
| 版本 | 每次生成、编辑或重写后保存的可追溯记录 | content version |
| 快速生成会话 | 收集用途、文件/文本来源并展示生成进度的临时会话 | quick_start_session |
| 风险项 | 需要教师确认或系统修复的来源、置信度、题目结构问题 | review_finding |
| 导出清单 | 一次导出固定使用的输出及内容块版本集合 | output_manifest |
| 幂等键 | 防止重复点击或网络重试创建重复项目/任务的请求标识 | Idempotency-Key |
