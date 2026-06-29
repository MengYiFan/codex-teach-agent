# ADR 0001：采用 Monorepo 分层结构

## 状态

Accepted

## 背景

教学课件智能生成系统同时包含 Web、API、Worker、Agent、解析、生成、渲染、存储、共享类型等模块。若分散到多个仓库，早期会增加联调、版本和契约维护成本。

## 决策

采用 Monorepo，并按 `apps/`、`packages/`、`infra/`、`docs/` 分层组织。

## 影响

- 前后端和算法共享领域类型、Schema、fixtures。
- 文档与代码同仓维护，方便任务卡和接口契约同步。
- 需要明确包边界，避免所有模块互相依赖。
