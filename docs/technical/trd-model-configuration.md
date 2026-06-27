# TRD：模型配置与 AI Gateway

## 目标

普通教师不配置模型；管理员和私有化部署场景通过统一模型配置能力管理供应商、Key、模型能力、额度、审计和成本。

## 配置字段

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `provider_type` | 是 | openai_compatible、azure_openai、anthropic、gemini、local 等 |
| `display_name` | 是 | 管理员可识别名称 |
| `base_url` | 条件必填 | `openai_compatible`、私有化和自定义 Azure endpoint 必填；使用供应商默认端点时可空 |
| `auth_type` | 是 | `api_key`、`workload_identity` 或 `none` |
| `api_key` | 条件必填 | `auth_type=api_key` 时必填并加密；本地模型或工作负载身份不要求 |
| `model_name` | 是 | 实际调用模型名称 |
| `capability` | 是 | 文本、视觉、嵌入、JSON、函数调用等能力 |
| `is_default` | 否 | 是否默认模型 |

## 安全要求

1. API Key 必须加密存储。
2. 前端不能回显完整 Key。
3. 调用时临时解密。
4. 日志不能记录完整敏感 Key。
5. 支持删除、禁用和连通性测试。
6. 密文必须记录密钥版本，支持轮换；数据库和日志不能保存可用的明文凭据。
7. 连通性测试使用最小请求，不携带教师课件或学生数据。

## AI Gateway 职责

1. 统一模型供应商适配。
2. 统一模型路由、fallback、重试和限流。
3. 统一记录 prompt_key、prompt_version、模型、token、成本、延迟和失败原因。
4. 统一进行日志脱敏和审计。
5. 为机构版提供模型池、额度限制和成本统计基础。
6. 在发送外部模型前执行数据策略：敏感数据检测、必要脱敏、供应商允许列表和调用目的记录。

## 模型路由建议

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
