# TRD：快速生成状态机

## 目标与边界

快速生成会话负责收集用途、文件或文本需求、可选上下文，并聚合项目创建和生成进度。它不是第二套项目模型：只有用户明确点击“开始生成”后才创建一个项目草稿。

MVP-A 要求用户登录后才能创建会话。未登录用户只能读取公开示例，不允许上传真实文件或创建项目。

## 核心对象

| 对象 | 生命周期 | 说明 |
| --- | --- | --- |
| `quick_start_session` | 创建后最多保留 24 小时；生成后保留 30 天用于排障 | 保存输入意图、临时来源、上下文和聚合状态 |
| `project` | 首次接受生成请求时创建 | 后续文件、CourseIR、输出、版本均归属项目 |
| `generation_task` | 每个长步骤一个或多个任务 | 解析、识别、生成、审校、导出均可独立重试 |

## 来源模式

| 模式 | 校验 | 进入 `source_ready` 的条件 |
| --- | --- | --- |
| `file` | PPTX/PDF，扩展名、MIME、magic bytes 一致，≤ 50 MiB，≤ 200 页/幻灯片 | 文件上传完成并通过安全、格式和基本可读性检查 |
| `text` | 去除首尾空白后 10-2000 个 Unicode 字符 | 文本需求保存成功 |
| `file_with_text` | 同时满足上述两类校验 | 文件检查通过且文本需求有效 |

## 会话状态

| 状态 | 是否终态 | 教师端文案 | 允许动作 |
| --- | --- | --- | --- |
| `awaiting_source` | 否 | 请上传课件或描述备课需求 | 上传、填写需求、删除会话 |
| `source_validating` | 否 | 正在检查资料 | 查询、取消上传 |
| `source_ready` | 否 | 资料已准备好 | 修改用途/上下文、开始生成 |
| `queued` | 否 | 已加入生成队列 | 查询、取消 |
| `parsing` | 否 | 正在识别课件内容 | 查询、取消、稍后查看 |
| `detecting_context` | 否 | 正在识别课程信息 | 查询、稍后查看 |
| `generating` | 否 | 正在生成教学材料 | 查询、查看已完成产物、取消未开始步骤 |
| `reviewing_quality` | 否 | 正在检查格式、答案和来源 | 查询、查看草稿 |
| `partial_success` | 是 | 部分内容已生成 | 查看、重试失败产物、导出成功产物 |
| `succeeded` | 是 | 教学材料已生成 | 查看、编辑、重写、导出 |
| `failed` | 是 | 生成失败 | 查看原因、按可恢复性重试 |
| `cancelled` | 是 | 已取消生成 | 复制输入并重新开始 |
| `expired` | 是 | 临时会话已过期 | 创建新会话 |

`partial_success` 表示至少一个请求产物可用，且至少一个产物失败；单页解析失败但所有请求产物可用时，会话仍可为 `succeeded`，同时返回 warning。

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

任一非终态 → cancelled
awaiting_source/source_ready 超过 24 小时 → expired
```

禁止的流转必须返回 `INVALID_STATE_TRANSITION`，并包含当前状态、允许动作和 `request_id`。

## 项目创建与幂等

1. `POST /quick-start/sessions` 只创建会话，不创建项目。
2. 文件先写入 `tmp/quick-start/{session_id}/{file_id}`；文本需求直接保存在会话中。
3. `POST /quick-start/sessions/{id}/generate` 必须携带 `Idempotency-Key`。
4. API 在同一事务中锁定会话、验证 `source_ready`、创建项目、把临时文件归属迁移到项目、创建首个任务，并记录幂等结果。
5. 同一用户、同一 session、同一幂等键的重复请求返回原 `project_id` 和 `task_id`；不同 payload 复用同一键返回 `IDEMPOTENCY_KEY_REUSED`。
6. 创建项目成功而队列投递失败时，outbox 记录负责重投；不得创建第二个项目。

## 进度计算

进度只用于用户反馈，不作为任务完成依据：

| 步骤 | 权重 |
| --- | --- |
| 文件解析（文本模式跳过） | 25% |
| 课程信息识别与 CourseIR | 20% |
| 默认产物生成 | 45%，按请求产物均分 |
| 基础质量检查与持久化 | 10% |

状态响应必须包含 `available_outputs`、`warnings` 和 `recoverable_errors`。前端每 2 秒轮询；页面隐藏后退避到 10 秒。SSE/推送不属于 MVP-A。

## 失败与重试

| 错误类别 | 自动重试 | 用户动作 |
| --- | --- | --- |
| 模型/网络瞬时失败 | 最多 2 次，指数退避并加随机抖动 | 自动重试耗尽后手动重试失败产物 |
| 队列/Worker 瞬时失败 | 最多 3 次 | 稍后重试 |
| 文件损坏、加密或格式不符 | 不重试 | 更换文件 |
| OCR/单页提取失败 | 不阻断其他页面 | 跳过、补充文本或更换文件 |
| Schema 校验失败 | 自动修复 1 次，再换低风险降级模板 1 次 | 查看部分结果或重试 |
| 权限、配额、内容安全拒绝 | 不自动重试 | 登录、升级、修改输入或联系管理员 |

所有重试创建新的 task attempt，但不得覆盖已经成功的内容版本。

## 上下文确认

课程名、学科、年级、课时和学生水平分别保存 `value`、`source`、`confidence` 和 `needs_confirmation`。阈值默认如下：

- `confidence >= 0.85`：直接展示，可修改。
- `0.60 <= confidence < 0.85`：标记“请确认”，不阻塞初稿。
- `confidence < 0.60`：字段留空或使用明确标注的产品默认值，不把推断值当成事实。

修改上下文时，用户必须选择：

1. `future_only`：只影响后续生成与重写。
2. `regenerate_affected`：创建新任务重生成受影响产物；旧版本保留。

## 并发规则

1. 每个会话最多有一个活动的初次生成请求。
2. 同一内容块最多有一个活动的重写任务；其他请求返回 `RESOURCE_BUSY`。
3. 编辑使用 `expected_version` 乐观锁；版本不匹配返回 `VERSION_CONFLICT` 并返回当前版本号。
4. 取消只影响尚未提交的模型调用和队列步骤，已经成功持久化的产物仍保留。
