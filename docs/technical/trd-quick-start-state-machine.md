# TRD：快速生成状态机

## 目标与边界

快速生成会话负责收集用途、文件或文本需求、可选上下文，并聚合项目创建和生成进度。它不是第二套项目模型：只有用户明确点击“开始生成”后才创建一个项目草稿。

MVP-A 要求用户登录后才能创建会话。未登录用户只能读取公开示例，不允许上传真实文件或创建项目。

## 核心对象

| 对象 | 生命周期 | 说明 |
| --- | --- | --- |
| `quick_start_session` | 创建后最多保留 24 小时；生成后保留 30 天用于排障 | 保存输入意图、临时来源、上下文和聚合状态 |
| `project` | 首次接受生成请求时创建 | 后续文件、CourseIR、输出、版本均归属项目 |
| `generation_run` | 每次初次生成、失败产物重试或上下文重生成各创建一轮；随项目保留 | 固化本轮上下文快照并聚合请求产物、根任务和最终结果；同一会话最多一轮活动 run |
| `generation_task` | 每个长步骤一个或多个任务 | 初次/重生成的解析、识别、生成、审校任务属于一个 run；后续块重写、导出和清理任务通过目标资源独立关联 |

`expires_at` 在首次生成前表示会话可操作截止时间（创建后 24 小时）；接受首次生成后更新为聚合诊断记录的清理时间（30 天后）。项目和项目来源不受该字段控制。

## 来源模式

| 模式 | 校验 | 进入 `source_ready` 的条件 |
| --- | --- | --- |
| `file` | PPTX/PDF，扩展名、MIME、magic bytes 一致，≤ 50 MiB，≤ 200 页/幻灯片 | 文件上传完成并通过安全、格式和基本可读性检查 |
| `text` | 去除首尾空白后 10-2000 个 Unicode 字符 | 文本需求保存成功 |
| `file_with_text` | 同时满足上述两类校验 | 文件检查通过且文本需求有效 |

## 会话状态

“本轮终态”表示当前 generation run 不再自动推进，但用户可以通过明确动作创建新 run；只有 `expired` 是不可恢复的会话终态。

| 状态 | 状态性质 | 教师端文案 | 允许动作 |
| --- | --- | --- | --- |
| `awaiting_source` | 可编辑 | 请上传课件或描述备课需求 | 上传、填写需求、删除会话 |
| `source_validating` | 活动 | 正在检查资料 | 查询、取消上传 |
| `source_ready` | 可编辑 | 资料已准备好 | 修改用途/上下文、开始生成 |
| `queued` | 活动 | 已加入生成队列 | 查询、取消 |
| `parsing` | 活动 | 正在识别课件内容 | 查询、取消、稍后查看 |
| `detecting_context` | 活动 | 正在识别课程信息 | 查询、稍后查看 |
| `generating` | 活动 | 正在生成教学材料 | 查询、查看已完成产物、取消未开始步骤 |
| `reviewing_quality` | 活动 | 正在检查格式、答案和来源 | 查询、查看草稿 |
| `partial_success` | 本轮终态 | 部分内容已生成 | 查看、重试失败产物、导出成功产物、修改上下文后重生成 |
| `succeeded` | 本轮终态 | 教学材料已生成 | 查看、编辑、重写、恢复、导出、修改上下文后重生成 |
| `failed` | 本轮终态 | 生成失败 | 查看原因、按可恢复性重试 |
| `cancelled` | 会话终态 | 已取消生成 | 复制输入并创建新会话 |
| `expired` | 会话终态 | 临时会话已过期 | 创建新会话 |

`partial_success` 表示至少一个请求产物可用，且至少一个产物失败；单页解析失败但所有请求产物可用时，会话仍可为 `succeeded`，同时返回 warning。

run 聚合只看本轮 `requested_outputs`：存在 queued/running/retrying 子任务时不得 settled；全部请求产物可用为 `succeeded`；至少一个可用且至少一个任务耗尽/死信为 `partial_success`；没有任何可用产物且所有任务已终止为 `failed`。非请求的历史产物不用于把本轮失败提升为部分成功。

## 状态流转

```text
awaiting_source
  → source_validating → source_ready
  → source_validating → awaiting_source（来源校验失败，可修复）

source_ready
  → queued → parsing（文件模式）→ detecting_context
  → queued → detecting_context（文本模式）
  → generating → reviewing_quality
  → succeeded | partial_success | failed

partial_success/failed
  → queued（显式重试失败产物，创建新 generation_run）

succeeded/partial_success
  → queued（确认新上下文并选择 regenerate_affected，创建新 generation_run）

任一活动状态 → cancelled
awaiting_source/source_ready 超过 24 小时 → expired
```

`cancelled`、`expired` 不允许原会话重新进入活动状态。禁止的流转必须返回 `INVALID_STATE_TRANSITION`，并包含当前状态、允许动作和 `request_id`。

## Session、run、task 与 project 聚合

1. 初次生成、失败产物重试和 `regenerate_affected` 都创建新的 `generation_run`；run 保存请求的输出类型、上下文版本及不可变上下文快照。清理 quick-start session 只把 run 的来源关联置空，不删除项目 run/task 历史。
2. run 下包含一个 `generation_run` 根任务以及解析、CourseIR、产物、审校等子任务。自动重试只增加 task attempt；用户手动重试创建后继任务并保留原失败任务。
3. 同一项目、同一会话最多一个活动 run。旧 run 和任务保持不可变，状态查询返回可空的 `active_run_id` 和始终指向最近一轮的 `latest_run_id`。
4. 会话状态由活动 run 聚合；没有活动 run 时展示最近 settled run。任务完成不能绕过聚合事务直接把会话标为成功。
5. project 状态映射固定为：创建事务内为 `draft`；活动 run 为 `generating`；会话 `succeeded` 为 `ready`；`partial_success` 为 `partial`；无任何可用产物的 `failed` 为 `failed`。取消后有可用产物则为 `partial`，否则回到 `draft`。
6. 子任务结果、run 最终状态、session 聚合状态和 project 状态必须在同一数据库事务内推进；重复消费同一结果不得重复推进。

## 项目创建与幂等

1. `POST /quick-start/sessions` 只创建会话，不创建项目。
2. 文件先写入 `tmp/quick-start/{session_id}/{file_id}`；文本需求直接保存在会话中。
3. `POST /quick-start/sessions/{id}/generate` 必须携带 `Idempotency-Key`。
4. API 在同一事务中锁定会话、验证 `source_ready`、创建项目、把临时文件归属迁移到项目、把文本需求复制为项目级加密文本来源、创建 generation run 和首个根任务，并记录幂等结果。
5. 同一用户、同一 session、同一幂等键的重复请求返回原 `project_id` 和 `task_id`；不同 payload 复用同一键返回 `IDEMPOTENCY_KEY_REUSED`。
6. 创建项目成功而队列投递失败时，outbox 记录负责重投；不得创建第二个项目。
7. 显式重试或上下文重生成锁定会话，验证没有活动 run，创建新 run/root task/outbox，并更新聚合状态；相同幂等键只返回原 run 和任务。

## 进度计算

进度只用于用户反馈，不作为任务完成依据：

| 步骤 | 权重 |
| --- | --- |
| 文件解析（文本模式跳过） | 25% |
| 课程信息识别与 CourseIR | 20% |
| 默认产物生成 | 45%，按请求产物均分 |
| 基础质量检查与持久化 | 10% |

状态响应必须包含 `active_run_id`、`available_outputs`、`warnings` 和 `recoverable_errors`。每个可恢复错误至少包含 `task_id`、`output_type` 或文件定位信息，使前端可以发起明确重试。前端每 2 秒轮询；页面隐藏后退避到 10 秒。SSE/推送不属于 MVP-A。

## 失败与重试

| 错误类别 | 自动重试 | 用户动作 |
| --- | --- | --- |
| 模型/网络瞬时失败 | 最多 2 次，指数退避并加随机抖动 | 自动重试耗尽后手动重试失败产物 |
| 队列/Worker 瞬时失败 | 最多 3 次 | 稍后重试 |
| 文件损坏、加密或格式不符 | 不重试 | 更换文件 |
| OCR/单页提取失败 | 不阻断其他页面 | 跳过、补充文本或更换文件 |
| Schema 校验失败 | 自动修复 1 次，再换低风险降级模板 1 次 | 查看部分结果或重试 |
| 权限、配额、内容安全拒绝 | 不自动重试 | 登录、升级、修改输入或联系管理员 |

自动重试在同一逻辑任务下创建新的 task attempt；用户手动重试创建带 `retry_of_task_id` 的后继任务。两类重试都不得覆盖已经成功的内容版本。自动重试耗尽后任务进入 `dead_lettered`，记录最后错误并允许受控人工重放。

## 上下文确认

课程名、学科、年级、课时和学生水平分别保存 `value`、`source`、`confidence` 和 `needs_confirmation`。首次生成时复制到项目 `context + context_version`，后续项目确认接口继续维护；run 固化当轮快照。阈值默认如下：

- `confidence >= 0.85`：直接展示，可修改。
- `0.60 <= confidence < 0.85`：标记“请确认”，不阻塞初稿。
- `confidence < 0.60`：字段留空或使用明确标注的产品默认值，不把推断值当成事实。

修改上下文时，用户必须选择：

1. `future_only`：只影响后续生成与重写。
2. `regenerate_affected`：创建新任务重生成受影响产物；旧版本保留。

`regenerate_affected` 只允许在没有活动 run 时执行，并原子创建新 run。`future_only` 不改变会话运行状态。

项目尚未创建时只允许 `future_only`（即更新首次生成将使用的上下文）；对尚不存在的产物请求 `regenerate_affected` 返回 `INVALID_STATE_TRANSITION`。

## 并发规则

1. 每个项目和会话最多有一个活动 generation run，包括初次生成、失败重试和上下文重生成。
2. 同一内容块最多有一个活动的重写任务；其他请求返回 `RESOURCE_BUSY`。
3. 编辑使用 `expected_version` 乐观锁；版本不匹配返回 `VERSION_CONFLICT` 并返回当前版本号。
4. 取消只影响尚未提交的模型调用和队列步骤，已经成功持久化的产物仍保留。
5. 不同内容块可以并发编辑，但创建整份 OutputVersion 前必须锁定所属 `generated_output`，基于锁内最新 manifest 分配下一版本，避免丢失另一个块的更新。
6. Worker 写结果时锁定逻辑任务并使用 task/result 唯一键；结果版本提交和任务成功状态在同一事务完成，队列 ACK 只能发生在事务提交后。
7. 重写或上下文重生成任务在 payload 中固定 `base_output_version` 和目标块版本。Worker 提交时若当前版本已变化，返回受控 `VERSION_CONFLICT` 并保留用户当前版本，不自动把过期模型结果设为 current；教师可基于最新版本重新发起。

BullMQ `jobId` 使用稳定的 `{task_id}:{attempt_number}`；outbox 重投同一 attempt 不能创建第二个队列任务。租约恢复或 Worker 重启仍由数据库 task/attempt 和结果唯一键决定是否执行或直接返回既有结果。
