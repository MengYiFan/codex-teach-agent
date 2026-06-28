# MVP-A 测试与验收规范

## 发布判定

发布候选版本必须同时满足：

1. P0/P1 功能、权限、幂等和数据生命周期用例全部通过。
2. OpenAPI 校验、数据库 migration 和固定 fixture 契约测试通过。
3. 性能、可靠性和内容质量达到本文件阈值。
4. 不存在未关闭的高危安全问题；中危问题必须有负责人和上线前处置日期。

任何一次测试都记录 commit、环境、模型配置 ID、Prompt 版本、fixture 版本和执行时间，保证结果可复现。

## 基准输入

性能基线只使用以下输入，超出范围仍需可用但不受 3 分钟 SLO 约束：

| 类型 | 基准范围 |
| --- | --- |
| PPTX | ≤ 30 MiB、≤ 100 页、文本为主、图片 ≤ 200 张、无宏和密码 |
| 文本 PDF | ≤ 30 MiB、≤ 100 页、可提取文本 |
| 扫描 PDF | ≤ 20 MiB、≤ 30 页、300 DPI 以内；单独统计 OCR 性能 |
| 文本需求 | 10-1000 字，至多 5 个明确约束 |
| 默认输出 | 按用途矩阵生成；最多 3 个默认产物 |

测试环境固定 API/Worker 规格、Worker 并发、模型路由和网络区域。冷启动与热运行分别报告，不混合计算。

## 性能与可靠性

| 指标 | 定义 | 门槛 |
| --- | --- | --- |
| 首次有意义预览 | 从 generation run 的 `requested_at` 到首个默认产物完整内容块持久化、通过 Schema 且可由 API 读取的 `first_meaningful_output_ready_at`；均为服务端时间 | 热运行 P50 ≤ 90 秒，P95 ≤ 180 秒；冷启动 P95 ≤ 240 秒 |
| 请求可用率 | 请求获得至少一个可预览产物 | ≥ 98% |
| 完整成功率 | 所有请求产物成功 | ≥ 95% |
| 导出成功率 | 合法导出请求在一次自动重试内成功 | ≥ 99% |
| API 延迟 | 不含上传和长任务的读写 API | P95 ≤ 500 ms |
| 重复副作用 | 重放相同幂等请求产生重复项目/版本/导出 | 0 |

至少执行 100 个生成请求；少于 100 个时只报告观测值，不宣称达到百分位 SLO。

可用率、完整成功率、取消、依赖失败和重复请求的分母及去重规则以 [指标、埋点与质量反馈 TRD](../technical/trd-analytics-and-metrics.md) 为准；不得在测试报告中临时排除失败样本。

## 内容质量数据集

首版至少 60 个经授权样本：

| 维度 | 最低覆盖 |
| --- | --- |
| 学科 | 语文、数学、英语、物理、化学、生物、历史、地理至少各 5 个 |
| 年级 | 小学高年级、初中、高中各不少于 15 个 |
| 输入 | PPTX ≥ 25、文本 PDF ≥ 15、扫描 PDF ≥ 5、文本需求 ≥ 15 |
| 难度 | 基础、普通、提高均有覆盖 |
| 异常 | 低置信上下文、缺页码、图表、公式、长文件、损坏/加密文件 |

按 [内容质量评分标准](../reference/content-quality-rubric.md) 评审。每个发布候选至少评审固定核心集 30 个；模型、Prompt、解析器或 CourseIR Schema 有实质变化时运行完整集。

## P0 主链路用例

1. 登录教师以 PPTX 创建会话，上传通过后生成，获得默认产物、来源页码并成功导出三种格式。
2. 登录教师只输入文本需求，不上传文件即可生成；不存在 `waiting_for_file` 阻断。
3. 同一生成请求重放三次，只创建一个 project 和一个 root task。
4. 内容块编辑创建新版本；旧版本可读；版本冲突不会覆盖他人修改。
5. 局部重写只修改目标块，输出其他块的版本保持不变。
6. 部分产物失败时成功产物可预览和导出，状态为 `partial_success`。
7. 项目 A 的用户不能读取、猜测下载或修改项目 B 及其 file、parsed asset、text source、task、output、历史版本和导出。
8. 日志、trace、错误响应和数据库均不出现明文 API Key、Authorization、签名 URL 或敏感输入原文。
9. 刷新未生成的文件会话后可重新取得文件 ID/状态并继续生成；修改用途使用版本锁且不会丢失上下文更新。
10. 删除、确认和从内容块/整份输出历史版本恢复都会创建新版本，导出旧 OutputVersion 仍产生原内容。

## P1 异常与恢复用例

1. 扩展名伪装、MIME 不匹配、空文件、>50 MiB、>200 页、加密 PDF、损坏 PPTX 被正确拒绝。
2. zip bomb、路径穿越、恶意文件扫描命中后不进入解析。
3. 模型 429/5xx、超时、队列短暂不可用按策略重试，耗尽后返回受控错误。
4. Worker 在写入前、写入后和确认队列前崩溃，重放不产生重复版本。
5. 导出期间内容发生新编辑，导出仍固定到请求时的版本清单。
6. 低置信上下文产生 finding；确认后只关闭对应版本的 finding。
7. 临时会话 24 小时后过期并清理对象；删除项目立即不可访问并进入清理队列。
8. OpenAPI 中每种 4xx/5xx 错误至少有一个契约测试。
9. 两个不同内容块并发保存时，最终 OutputVersion 同时包含两次更新；同一块并发仍返回 `VERSION_CONFLICT`。
10. `partial_success` 和 `failed` 显式重试会创建新 generation run，旧 run/task 保持不变，session/project 聚合状态最终一致。
11. 项目级 finding 不要求 block version；块级 finding 使用目标版本锁，不能处理已经过期的版本。
12. 删除项目返回可查询 operation；在线对象七天内清理，失败进入死信，备份恢复后 tombstone 重新执行。
13. 重写/上下文重生成运行期间发生用户编辑时，过期 Worker 结果返回 `VERSION_CONFLICT`，不能移动 current 指针或覆盖用户版本。
14. 当前块存在 open blocking finding 时不能直接确认；处理 finding 后确认创建新版本，旧 finding 保留审计且不再阻塞新版本。

## 安全测试

| 类别 | 用例 |
| --- | --- |
| 越权 | 顺序/随机 UUID 枚举、跨用户 project/file/output/export 访问 |
| 上传 | 双扩展名、伪 MIME、宏、嵌套压缩、超大图片、畸形 PDF/PPTX |
| 渲染 | Prompt 注入、脚本标签、事件属性、危险 URL、Markdown HTML |
| 凭据 | 管理员与教师权限、Key 掩码、日志脱敏、轮换和删除 |
| 模型端点 | loopback、云元数据、DNS 重绑定、重定向到私网、未批准端口和允许的本地模型 allowlist |
| 资源消耗 | 大页数、高压缩比、超长文本、并发上传和并发重写 |

## 埋点验收

使用一个测试用户走完访问、登录、文件与文本来源、生成成功/失败、预览、编辑、重写、审核和导出。验证：

1. 事件顺序与服务端事实一致，成功事件不由客户端伪造。
2. `event_id` 重放不会重复计数。
3. 漏斗能计算来源就绪、生成、预览、导出和第二次复用。
4. 属性不包含禁止采集内容。
5. 客户端提交服务端事实事件被拒绝；匿名转登录、迟到事件和窗口边界按指标 TRD 得到固定结果。
6. 质量反馈固定到 OutputVersion 清单，可计算首次导出前评分和人工编辑字符比例。

## 验收记录模板

```markdown
- Release/commit:
- Environment:
- OpenAPI version:
- DB migration version:
- Model config ID:
- Prompt versions:
- Fixture set version:
- Functional/security result:
- Performance P50/P95 and sample size:
- Quality score and hard-failure rate:
- Known risks, owner, deadline:
- Decision: pass / fail
```
