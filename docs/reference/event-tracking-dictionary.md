# 埋点事件字典

## 通用规则

每个事件都包含：

| 字段 | 说明 |
| --- | --- |
| `event_id` | 客户端 UI 事件由客户端生成 UUID；业务事实事件由服务端生成 UUID；全局用于去重 |
| `event_name` | 下表中的固定事件名 |
| `occurred_at` | 客户端 UTC 时间；服务端另记录接收时间 |
| `visit_session_id` | 产品访问会话，不等同 quick-start session |
| `quick_start_session_id` | 快速生成会话 ID，可空 |
| `generation_run_id` | 一次生成或显式重生成 run ID，可空；性能和可靠性去重单位 |
| `subject_key` | 服务端根据登录用户或匿名 ID 生成的不可逆 HMAC；客户端不能直接指定 |
| `anonymous_id` | 未登录浏览器 ID，仅在接收时用于生成 subject HMAC，不原样落入指标仓库；登录时用于合法归因，不跨设备追踪 |
| `source` | 首页、示例、历史项目等入口 |
| `app_version` | 前端版本/commit |
| `experiment_ids` | 实验分组，不得包含个人信息 |

埋点不得包含文件名、文件正文、备课需求原文、学生信息、Prompt、API Key、签名 URL 或自由输入。失败原因使用受控枚举 `error_code`。

客户端事件为页面查看、示例打开、登录提示、意图选择、上传开始、首次实际渲染、付费墙展示等 UI 事实。来源就绪、生成、内容保存、审核处理、导出和订阅结果均为服务端事件；客户端提交这些事件名必须被拒绝。接收、保留和指标计算以 [指标、埋点与质量反馈 TRD](../technical/trd-analytics-and-metrics.md) 为准。

## MVP-A 事件

| 事件名 | 触发时机 | 关键属性 |
| --- | --- | --- |
| `quick_start_viewed` | 打开快速生成页 | `is_logged_in` |
| `sample_project_opened` | 打开示例 | `sample_subject`、`sample_grade` |
| `auth_prompt_shown` | 点击真实生成但未登录 | `trigger` |
| `auth_completed` | 登录完成并回到快速入口 | `method`、`return_source` |
| `quick_start_intent_selected` | 选择用途 | `intent` |
| `source_text_submitted` | 文本需求校验成功 | `character_count_bucket`、`has_optional_context` |
| `file_upload_started` | 开始上传 | `file_type`、`file_size_bucket` |
| `file_upload_completed` | 文件安全和格式检查通过 | `file_type`、`file_size_bucket`、`page_count_bucket`、`duration_ms` |
| `file_upload_failed` | 上传或检查失败 | `error_code`、`file_type`、`file_size_bucket` |
| `generation_requested` | 点击开始生成且 API 接受 | `intent`、`source_type`、`requested_outputs` |
| `generation_started` | 首个 Worker 任务开始 | `source_type`、`queue_wait_ms` |
| `first_meaningful_output_ready` | 首个默认产物完整内容块已持久化并可读取 | `time_to_ready_ms`、`output_type`、`source_type` |
| `generation_partial_succeeded` | 至少一个产物成功、至少一个失败 | `available_outputs`、`failed_outputs`、`duration_ms` |
| `generation_succeeded` | 所有请求产物成功 | `available_outputs`、`duration_ms`、`model_route_class` |
| `generation_failed` | 无可用产物 | `error_code`、`failed_step`、`retryable`、`duration_ms` |
| `first_preview_shown` | 首个默认产物首次实际渲染 | `time_from_ready_to_shown_ms`、`output_type`、`source_type` |
| `context_confirmed` | 确认或修改上下文 | `changed_fields`、`apply_mode`、`confidence_bucket_before` |
| `content_block_edited` | 手工编辑保存成功 | `block_type`、`previous_version`、`new_version` |
| `content_block_regeneration_requested` | 发起局部重写 | `block_type`、`action` |
| `content_block_regenerated` | 局部重写完成 | `block_type`、`action`、`latency_ms` |
| `review_finding_resolved` | 处理风险项 | `finding_type`、`severity`、`resolution` |
| `quality_feedback_submitted` | 版本化质量反馈保存成功 | `usability_rating`、`output_types`、`before_first_export` |
| `output_export_requested` | 创建导出任务 | `file_format`、`output_types` |
| `output_exported` | 导出成功 | `file_format`、`output_types`、`latency_ms` |
| `output_export_failed` | 导出失败 | `file_format`、`error_code`、`retryable` |
| `project_revisited` | 创建 24 小时后再次打开项目 | `days_since_created_bucket` |
| `second_generation_started` | 同一用户开始第二个项目生成 | `days_since_first_generation_bucket` |
| `paywall_shown` | 展示升级提示 | `trigger`、`plan_suggested` |
| `subscription_started` | 开通套餐成功 | `plan`、`billing_cycle`、`currency` |

`source_text_submitted`、`file_upload_completed/failed`、`generation_*`、`first_meaningful_output_ready`、内容保存、审核、质量反馈、导出结果和订阅事件必须由服务端发送；客户端事件只用于 UI 行为，防止把页面点击误判为成功。

## MVP-B 事件

| 事件名 | 触发时机 | 关键属性 |
| --- | --- | --- |
| `share_link_created` | 分享链接创建成功 | `expires_in_days`、`output_types` |
| `share_link_revoked` | 教师撤回分享 | `age_hours_bucket` |
| `shared_page_viewed` | 学生打开有效分享页 | `share_id_hash`、`is_first_view` |
| `student_quiz_completed` | 学生完成自测 | `question_count`、`score_bucket`、`duration_bucket` |

## 漏斗口径

| 漏斗步骤 | 事件 | 窗口 |
| --- | --- | --- |
| 访问 | `quick_start_viewed` | 起点 |
| 激活意向 | `quick_start_intent_selected` | 访问后 24 小时 |
| 来源就绪 | `file_upload_completed` 或 `source_text_submitted` | 访问后 24 小时 |
| 生成接受 | `generation_requested` | 来源就绪后 24 小时 |
| 首次价值 | `first_meaningful_output_ready` | 请求后 30 分钟 |
| 编辑参与 | `content_block_edited` 或 `content_block_regenerated` | 首次预览后 7 天 |
| 导出 | `output_exported` | 首次预览后 7 天 |
| 第二次复用 | `second_generation_started` 或 `project_revisited` | 首次生成后 30 天 |

分母使用去重 subject；性能和可靠性指标使用去重 `generation_run_id`。测试、员工和自动化账号必须通过服务端账号属性排除。7 天、30 天窗口均按首次价值的服务端 UTC 时间计算，边界采用左闭右开区间。
