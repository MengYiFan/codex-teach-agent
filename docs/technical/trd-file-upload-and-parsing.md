# TRD：文件上传与解析

## 目标

为教师上传的教学资料建立稳定、可追溯、可降级的解析链路。MVP 核心支持 PPTX 和 PDF，后续扩展 DOCX、Markdown、图片 OCR 和 ZIP 多文件上传。

## 文件格式范围

| 阶段 | 格式 | 策略 |
| --- | --- | --- |
| MVP | PPTX | 核心支持，提取页标题、正文、图片、表格、备注和页码顺序。 |
| MVP | PDF | 核心支持，优先文本型 PDF；扫描型 PDF 走 OCR 降级。 |
| 后续 | DOCX | 教案、讲义、说明文档。 |
| 后续 | Markdown | 结构化文本材料。 |
| 后续 | PNG/JPG | 图片课件、扫描件。 |
| 后续 | ZIP | 多文件打包上传。 |

## 上传验收

1. MVP 单文件上限 50 MiB、最多 200 页/幻灯片、每个快速会话一个主文件。
2. 同时校验扩展名、声明 MIME、magic bytes 和 OOXML/PDF 结构；任一不一致都拒绝。
3. 文件先写入私有临时对象路径并进行恶意文件扫描，通过后才进入 `source_ready`。
4. 系统记录原始文件名、标准媒体类型、大小、SHA-256、object key、扫描和解析状态；不保存长期签名 URL。
5. 空文件、加密文件、损坏文件、超限文件和不支持格式使用标准错误码与教师可理解提示。
6. 用户点击生成后，临时文件和项目在同一事务建立归属，再由 outbox 创建解析任务。

## PPTX 解析要求

1. 提取每页标题。
2. 提取正文文本。
3. 提取图片并保存为资源文件。
4. 提取表格内容。
5. 提取备注信息。
6. 保留页码顺序。

## PDF 解析要求

1. 优先处理文本型 PDF。
2. 扫描型 PDF 使用 OCR，并标记 OCR 置信度。
3. 提取页码、段落、标题、图片和表格。
4. 保留来源页码，便于结果追溯。

## ParsedContent 示例

```json
{
  "file_id": "file_001",
  "file_type": "pptx",
  "pages": [
    {
      "page_index": 1,
      "title": "牛顿第二定律",
      "blocks": [
        {
          "type": "text",
          "text": "物体的加速度与所受合外力成正比"
        },
        {
          "type": "image",
          "asset_id": "asset_001",
          "path": "s3://bucket/project/image001.png"
        }
      ],
      "notes": "可从生活实例导入"
    }
  ]
}
```

`page_index` 从 1 开始。每个 block 还应包含稳定 `block_index`；OCR block 必须包含 `confidence`。解析结果必须携带 `schema_version`，并通过 `packages/parser` 暴露的 Schema 校验。

示例中的 `path` 仅是 Worker 内部对象引用；对外 `GET /files/{id}/parsed-result` 只返回 `asset_id`，不返回 Bucket、object key 或签名 URL。需要预览资源时由独立鉴权下载接口签发短期票据。

## 降级策略

| 场景 | 处理 |
| --- | --- |
| 不支持格式 | 提示“请另存为 PDF/PPTX 后上传”。 |
| 扫描件 OCR 置信度低 | 生成结果标记“识别不清”，并提示教师手动补充。 |
| 图片/表格提取失败 | 保留页码和错误原因，不阻断文本内容生成。 |
| 部分页面失败 | 记录 page warning，允许跳过问题页、重新上传或手动补充；若所有请求产物仍成功，会话为 `succeeded`，只有至少一个请求产物因此失败时才为 `partial_success`。 |
| 文件加密或损坏 | 不自动重试，返回 `FILE_UNREADABLE`。 |
| 恶意文件扫描失败 | 隔离对象，不进入解析；扫描服务瞬时失败可自动重试 2 次。 |
| 页数超过 200 | 返回校验错误并提示拆分文件。 |
