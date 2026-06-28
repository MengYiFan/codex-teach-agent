# 测试夹具规范

每个 fixture 目录包含：

- 输入文件或文本需求。
- `manifest.json`：fixture ID、来源授权、学科、年级、页数、是否扫描件、预期知识点和允许用途。
- 预期 `ParsedContent`、CourseIR 最小断言、文件/文本 `source_refs` 和输出结构断言。
- 预期 generation run/task 状态、幂等重放结果及服务端事件序列；并发 fixture 还要声明基线版本和预期冲突。
- 不包含真实姓名、手机号、学号、成绩、学校内部资料或未授权教材全文。

文件命名使用稳定 fixture ID，不使用真实用户和学校名称。二进制文件更新时必须同步 SHA-256。
