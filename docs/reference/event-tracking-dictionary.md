# 埋点事件字典

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
| `paywall_shown` | 展示升级提示 | trigger、plan_suggested |
| `subscription_started` | 用户开通套餐 | plan、billing_cycle |
