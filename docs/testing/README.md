# 测试与评估

- [MVP-A 测试与验收规范](mvp-a-acceptance.md)
- [内容质量评分标准](../reference/content-quality-rubric.md)
- [安全与测试策略](../technical/trd-security-and-testing.md)
- `fixtures/`：只存脱敏、自建或获授权的最小测试材料及 manifest，不提交真实教师或学生数据。

CI 至少执行类型检查、单元测试、OpenAPI 校验、契约测试、数据库 migration up/down、冲突标记检查和固定 fixture 的端到端 smoke test。生成质量和性能基线可在受控环境按发布候选版本执行。
