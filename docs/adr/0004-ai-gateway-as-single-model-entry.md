# ADR 0004：AI Gateway 作为唯一模型调用出口

## 状态

Accepted

## 背景

系统后续可能接入 OpenAI Compatible、Azure OpenAI、Claude、Gemini、DeepSeek、通义千问、智谱、火山方舟和本地模型。若各业务模块直接调用模型，会导致成本、日志、安全和重试策略分散。

## 决策

所有模型调用必须经过 `packages/ai-gateway`。

## 影响

- 统一管理模型供应商、Key、Base URL、重试、fallback 和限流。
- 统一记录 token、成本、延迟、失败原因、Prompt 版本。
- 普通教师端不暴露模型配置，管理员端集中配置。
