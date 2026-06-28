# TRD：指标、埋点与质量反馈

## 目标与边界

本文件定义 MVP-A 产品事件的接收、去重、事实来源、保留和指标计算。事件只记录受控枚举和数值，不保存文件名、课件正文、备课需求、学生信息、Prompt、凭据、签名 URL 或其他自由输入。

## 事件来源

| 来源 | 事件 | 规则 |
| --- | --- | --- |
| 浏览器 | 页面查看、意图选择、首次实际渲染、编辑点击等 UI 事实 | 通过 `POST /events/batch`；客户端生成 UUID `event_id` |
| API/Worker | 来源就绪、生成接受/开始/完成、导出结果、质量反馈等业务事实 | 在业务事务内写 outbox；服务端生成 UUID `event_id` |
| 离线评估 | 质量评分、硬失败和评分者一致性 | 写版本化评估结果，不通过客户端事件上报 |

客户端提交服务端事实事件名返回 `VALIDATION_ERROR`。服务端事件的 `occurred_at` 使用业务事务时间；客户端事件同时保存客户端时间和服务端 `received_at`，漏斗排序优先使用服务端时间，UI 时延使用同一 trace 的单调计时或受控测试记录。

## 接收与去重

1. 批量接口每次最多 50 条、请求体不超过 256 KiB，并按 IP、匿名会话和登录用户限速。
2. 登录态可选；服务端从用户 ID或匿名 ID 通过带版本的 HMAC 生成 `subject_key`，不信任客户端传入 `user_id`。
3. `event_id` 全局唯一。相同 ID 和相同 fingerprint 返回已接收；相同 ID、不同内容返回 `IDEMPOTENCY_KEY_REUSED`。
4. 属性严格按事件字典 Schema 白名单过滤，未知字段拒绝而不是落库。
5. 服务端业务事件使用 outbox 投递；消费者按 `event_id` 幂等，重复发布不能重复计数。
6. 原始事件保留 180 天；删除账号后移除可关联 subject key。只有分组样本量至少 20 的匿名聚合可长期保留。
7. 批量接收是原子的：任一事件 Schema、隐私或 ID fingerprint 校验失败时整批拒绝，不产生部分写入；同内容重复事件不算失败，在 `duplicate_count` 中返回。

## 关键标识

| 字段 | 说明 |
| --- | --- |
| `event_id` | 事件去重 UUID |
| `subject_key` | 服务端生成的不可逆用户/匿名主体标识 |
| `visit_session_id` | 产品访问会话，不等同 quick-start session |
| `quick_start_session_id` | 快速生成会话 ID，可空 |
| `generation_run_id` | 一次初次生成或显式重生成 run ID；性能和可靠性去重单位 |
| `project_id` | 仅在受控业务事件内部关联，不能作为监控指标标签 |
| `occurred_at` / `received_at` | 事件发生与服务端接收 UTC 时间 |
| `app_version` | 前端 commit 或服务版本 |

## 指标分母和窗口

1. “首次成功生成用户”指用户第一次 run 达到 `succeeded` 或 `partial_success` 且至少一个请求产物可预览；同一用户只进入一次分母。
2. 7 天导出率：上述用户中，在首次可预览时间后 7×24 小时内至少有一次 `output_exported` 的去重用户比例。
3. 7 天局部重写率：上述用户中，在同一窗口至少有一次 `content_block_regenerated` 的去重用户比例；手工编辑不计入该指标。
4. 第二次复用率：首次可预览后 30 天内创建第二个 generation run，或创建 24 小时后重新访问原项目的去重用户比例。
5. 请求可用率：生产环境被 API 接受的去重 generation run 中，最终至少一个请求产物可用的比例。用户在首个 Worker 开始前主动取消的 run 单独报告，不进入分母；依赖失败和超时进入分母。
6. 完整成功率：同一分母中所有请求产物均成功且没有 blocking finding 的比例。
7. 导出成功率：合法且被接受的去重导出请求中，在一次自动重试内成功的比例；用户取消不进入分母。
8. 测试、员工和自动化主体由服务端账号属性排除，不能依赖客户端标记。

## 性能时间戳

服务端在接受生成时记录 generation run 的 `requested_at`，在首个默认产物的一个完整内容块通过 Schema、持久化并可由 API 读取时，以 `UPDATE ... WHERE first_meaningful_output_ready_at IS NULL` 原子记录该时间并写同一事务 outbox 事件。并发完成多个产物时只能产生一个首次 ready 事件。发布 SLO 使用二者差值，不依赖客户端时钟。`first_preview_shown` 仅衡量 UI 实际展示体验，另行报告从 ready 到 shown 的延迟。

冷启动、热运行、输入 fixture、模型路由、Worker 并发和网络区域必须作为测试运行元数据记录，但不得作为高基数在线指标标签。

## 质量反馈

1. 教师或受控评审者提交 `direct_use`、`minor_edit`、`major_edit`、`unusable`，反馈固定到明确 OutputVersion 清单。`before_first_export` 由服务端按该项目首个被接受导出请求的时间计算，客户端不能指定。
2. 服务端保存反馈记录并发送 `quality_feedback_submitted`；客户端不能直接发送该成功事件。
3. 同一主体、同一项目、同一版本清单可以更新反馈，但每次更新保留审计版本；产品指标使用首次导出前最后一次有效反馈。
4. 质量评分、字符编辑比例和多产物聚合公式见 [内容质量评分标准](../reference/content-quality-rubric.md)。

## 验收查询

每个指标必须提供版本化查询或等价数据测试，固定输入事件可得到固定结果。至少覆盖：重复 `event_id`、迟到事件、匿名转登录、`partial_success`、首任务前取消、跨 7/30 天窗口边界和账号删除后的主体解绑。
