---
title: 检索和删除聊天、文件及项目
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data
description: 通过 Compliance API 访问 claude.ai 组织的聊天内容、文件附件和项目。
---

<Note>
  本页面上的端点仅对 Claude Enterprise 组织可用。它们用于检索和删除 claude.ai 的聊天、文件和项目；Cowork 和 Claude Code 等应用中的会话记录请参阅[检索会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions)。请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。
</Note>

<Check>
  **所需作用域：** Compliance Access Key 上的 `read:compliance_user_data`。删除端点还需要 `delete:compliance_user_data`。

  **前提条件：** 在组织范围内列出聊天无需前提条件。若要将聊天列表筛选到特定用户，您需要从[列出组织用户](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data#list-organization-users)获取用户 ID。本页面上的其他端点直接接受资源 ID。
</Check>

本页面上的端点向合规审查人员公开 Claude Enterprise 的聊天内容、文件上传、项目和项目附件。它们支持 "eDiscovery"（电子取证）导出、"data loss prevention"（数据丢失防护），即 DLP 的执行，以及账户删除响应。聊天、文件和项目内容的保留时长取决于您组织的保留策略。当用户在 claude.ai 中删除聊天时，其消息内容、附加文件、工具生成的文件和 artifacts 会随之一并删除。Compliance API 仍会列出该聊天，其 `deleted_at` 字段已填充且 `name` 为空，并返回不含内容的消息。已被硬删除的聊天（通过 Compliance API 本身删除，或在组织的保留窗口到期后删除）无法检索。

这两个作用域仅授予在 claude.ai 中创建的 Compliance Access Key（`sk-ant-api01-...`）；请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access) 以配置一个。`read:compliance_user_data` 作用域涵盖检索操作；`delete:compliance_user_data` 仅在使用删除端点时需要。聊天、文件、项目和附件端点不对 Admin API 密钥（`sk-ant-admin01-...`）开放；使用 Admin API 密钥进行身份验证的调用会返回 [403 Forbidden](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)。

本页面上的端点有两种分页方式；完整参考请参阅[分页结果](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#paginate-results)。每个章节都会注明适用哪种方案。

## 检索聊天和消息

使用[列出聊天](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/list)分页浏览聊天元数据，然后使用[获取聊天消息](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/messages/list)获取单个聊天的完整消息内容。

聊天列表端点默认为组织范围：省略 `user_ids[]` 即可包含您父组织下的所有聊天。添加 `order_by=updated_at` 可按最后更新时间排序。这种组合是导出聊天并保持导出内容最新的推荐方式，因为一个分页循环即可获取每个用户的新聊天、已修改的聊天以及在 claude.ai 中已删除的聊天，而无需先枚举用户。以下请求列出自指定日期以来更新过的聊天。

```bash cURL
curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/apps/chats" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --data-urlencode "order_by=updated_at" \
  --data-urlencode "updated_at.gte=2025-06-01T00:00:00Z" \
  --data-urlencode "limit=100"
```

```json Response
{
  "data": [
    {
      "id": "claude_chat_01H5CWunD7RpVJ5bHa8RCkja",
      "name": "Product Requirements Discussion",
      "created_at": "2026-04-10T08:09:10Z",
      "updated_at": "2026-04-10T09:10:11Z",
      "deleted_at": null,
      "href": "https://claude.ai/chat/abcdef01-2345-6789-abcd-ef0123456789",
      "model": "claude-opus-5",
      "organization_uuid": "91012d09-e48b-438e-a489-1bebfd8fa6f9",
      "project_id": "claude_proj_01KGp4eZNug9ri4kE35RSppq",
      "user": {
        "id": "user_01XyDMpzjS89pFZXqSFUBDr6",
        "email_address": "user@example.com"
      }
    }
  ],
  "has_more": true,
  "first_id": "eyJrIjogInVwZGF0ZWRfYXQiLCAidCI6ICIyMDI2LTA0LTEwVDA5OjEwOjExKzAwOjAwIiwgImlkIjogImFiY2RlZjAxLS4uLiJ9",
  "last_id": "eyJrIjogInVwZGF0ZWRfYXQiLCAidCI6ICIyMDI2LTA0LTEwVDA5OjEwOjExKzAwOjAwIiwgImlkIjogImFiY2RlZjAxLS4uLiJ9"
}
```

结果按 `order_by` 字段升序排序，最旧的在前，相同值时按 `id` 排序。分页使用[分页结果](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#paginate-results)中描述的标准 `first_id`/`last_id`/`has_more` 游标字段。若要向前遍历到较新的聊天，请在下一次请求中将响应的 `last_id` 作为 `after_id` 传回。

这种向前遍历也是您在多次运行之间保持导出内容最新的方式：持久化保存最后一页的 `last_id`，并在下次运行时将其作为 `after_id` 从该处继续。由于列表按 `updated_at` 排序，在您保存的游标之后发生变化的聊天会重新出现在游标前方，因此每次增量运行都会返回全新的聊天以及此后在 claude.ai 中被修改或删除的旧聊天。请以聊天 `id` 为键对结果进行幂等处理，以应对这些重复出现的情况。返回时 `deleted_at` 已填充的聊天已没有可获取的内容，因此应将其视为已删除而非已更新。

这些组织范围的查询有一些限制。游标是不透明的且与排序键绑定，因此在一个 `order_by` 值下签发的 `after_id` 在另一个值下会被拒绝并返回 400 错误。时间筛选边界也必须与排序键匹配：`updated_at.*` 边界应与 `order_by=updated_at` 搭配，`created_at.*` 边界应与默认的 `order_by=created_at` 搭配。不支持使用 `before_id` 向后分页，且 `project_ids[]` 筛选器不可用。完整的筛选器参考请参阅[列出聊天](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/list)。

若要将列表范围限定到特定用户（例如，对指定保管人的法律保留），请传入 1–10 个 `user_ids[]` 值。从[列出组织用户](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data#list-organization-users)获取这些 ID。按用户筛选的查询始终按 `created_at` 排序（传入 `order_by=updated_at` 会返回 400 错误），并同时支持 `after_id` 和 `before_id`。按 `project_ids[]` 筛选仅在这种按用户筛选的形式下可用。将 `user_ids[]` 与任何 `updated_at.*` 边界组合使用已被弃用，并将在 2026-09-22 之后被拒绝并返回 400 错误；若要按更新时间保持保管人集合最新，请运行不带 `user_ids[]` 的组织范围 `order_by=updated_at` 遍历，并从其结果中选出保管人的聊天，而将按用户筛选的列表保留用于按 `created_at` 排序的导出。

```bash cURL
curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/apps/chats" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --data-urlencode "user_ids[]=user_01XyDMpzjS89pFZXqSFUBDr6" \
  --data-urlencode "created_at.gte=2025-06-01T00:00:00Z" \
  --data-urlencode "limit=100"
```

列表响应仅包含聊天元数据。若要获取实际的聊天内容、附加文件和内联 artifacts（Claude 在聊天中生成的结构化文档），请针对每个聊天 ID 继续调用消息端点：

```bash cURL
chat_id="claude_chat_01H5CWunD7RpVJ5bHa8RCkja"

curl --fail-with-body -sS \
  "https://api.anthropic.com/v1/compliance/apps/chats/$chat_id/messages" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY"
```

消息端点返回聊天的元数据以及按 `created_at` 排序的 `chat_messages` 数组。省略 `limit` 时，完整的消息集会在一个响应中返回；传入 `limit`、`after_id` 或 `before_id` 可对非常长的聊天进行分页。该端点还接受 `created_at.*` 和 `updated_at.*` 范围边界（`gt`、`gte`、`lt`、`lte`）以及 `order` 参数（`asc` 或 `desc`）。完整的参数列表请参阅[获取聊天消息](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/messages/list)。对于用户消息，`created_at` 是消息发送的时间；对于助手消息，它是 Claude 完成生成该消息的时间。每条消息都包含其文本内容，以及（如果存在）任何上传的文件（通常在用户消息上）、任何工具生成的文件，以及助手生成或更新的任何 artifacts（通常在助手消息上）：

```json Response
{
  "id": "claude_chat_01H5CWunD7RpVJ5bHa8RCkja",
  "name": "Product Requirements Discussion",
  "created_at": "2026-04-10T08:09:10Z",
  "updated_at": "2026-04-10T09:10:11Z",
  "deleted_at": null,
  "href": "https://claude.ai/chat/abcdef01-2345-6789-abcd-ef0123456789",
  "model": "claude-opus-5",
  "organization_uuid": "91012d09-e48b-438e-a489-1bebfd8fa6f9",
  "project_id": "claude_proj_01KGp4eZNug9ri4kE35RSppq",
  "user": {
    "id": "user_01XyDMpzjS89pFZXqSFUBDr6",
    "email_address": "user@example.com"
  },
  "chat_messages": [
    {
      "id": "claude_chat_msg_01VnBPkLmtj7YdW5QrXKEA8c",
      "role": "user",
      "created_at": "2026-04-10T08:09:10Z",
      "content": [
        {
          "type": "text",
          "text": "Can you help me draft requirements for our new dashboard feature?"
        }
      ],
      "files": [
        {
          "id": "claude_file_01UaT9wBcDfGhJkLmNpQrSv7",
          "filename": "dashboard_mockup_v1.pdf",
          "mime_type": "application/pdf",
          "size_bytes": 482133,
          "md5": "56367e4d2705cc9c025ad07424e944f0",
          "created_at": "2026-04-10T08:09:10Z"
        }
      ]
    },
    {
      "id": "claude_chat_msg_01M8tFcHwbQ2kY6NpEjRZv4D",
      "role": "assistant",
      "created_at": "2026-04-10T08:09:11Z",
      "content": [
        {
          "type": "text",
          "text": "I'd be happy to help you draft requirements for your dashboard feature..."
        }
      ],
      "generated_files": [
        {
          "id": "claude_gen_file_01TbR8wAcCeFhJkLnPqStUvX",
          "filename": "requirements_summary.csv",
          "mime_type": "text/csv",
          "size_bytes": 2048,
          "md5": "89968669461d95416549937168269d6b"
        }
      ],
      "artifacts": [
        {
          "id": "claude_artifact_01HqRsTuVwXyZa2BcDeFgH4J",
          "version_id": "claude_artifact_version_01KmNpQrSt3UvWxYz5AbCdEfG",
          "title": "Dashboard Requirements Draft",
          "artifact_type": "text/markdown"
        }
      ]
    }
  ],
  "has_more": false,
  "first_id": "eyJtc2dfdXVpZCI6ICIwZjcwYjA2Ni0uLi4ifQ==",
  "last_id": "eyJtc2dfdXVpZCI6ICJhNGUwYjE3Mi0uLi4ifQ=="
}
```

在给定消息上，`files`、`generated_files` 和 `artifacts` 各自都可能为 `null`。`files` 是用户附加到消息的文件和文本附件（例如 PDF、图像、电子表格、文档和粘贴的文本），以 claude.ai 存储它们的形式呈现。`generated_files` 是助手在对话过程中通过工具使用创建的二进制文件（例如 PDF、电子表格或幻灯片）。`artifacts` 是助手在其响应中生成或更新的带版本的文档（例如代码或 markdown）；一个 artifact 可以在同一聊天的多个助手轮次中被修订，每次修订都会以同一 artifact `id` 下的新 `version_id` 出现。将每个条目的 `id`（对于 artifacts 则为 `version_id`）传给[检索文件和 artifacts](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data#retrieve-files-and-artifacts) 中对应的内容端点即可下载。

## 检索文件和 artifacts

文件和 artifacts 按 ID 下载，而不是独立列出。这些 ID 来自[检索聊天和消息](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data#retrieve-chats-and-messages)中的聊天消息端点（每条消息上的 `files`、`generated_files` 和 `artifacts` 数组），或者对于项目级上传，来自[项目附件端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data#retrieve-projects-and-attachments)。

请选择与您的 ID 类型和所需数据相匹配的端点。同一个文件内容端点同时服务于聊天文件和项目文件。

| 您拥有                            | 您想要               | 使用此端点                                                                                                        |
| ------------------------------ | ----------------- | ------------------------------------------------------------------------------------------------------------ |
| `claude_file_*` ID             | 文件的内容             | [下载文件内容](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/files/download)                    |
| `claude_file_*` ID             | 仅文件的元数据           | [获取文件元数据](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/files/retrieve)                   |
| `claude_gen_file_*` ID         | 工具生成文件的二进制内容      | [下载 Claude 生成的文件](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/generated_files/download) |
| `claude_gen_file_*` ID         | 仅工具生成文件的元数据       | [获取生成文件的元数据](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/generated_files/retrieve)      |
| `claude_artifact_version_*` ID | 一个 artifact 版本的文本 | [下载 artifact 内容](https://platform.claude.com/docs/zh-CN/api/compliance/apps/artifacts/download)              |
| `claude_artifact_version_*` ID | 仅 artifact 版本的元数据 | [获取 artifact 元数据](https://platform.claude.com/docs/zh-CN/api/compliance/apps/artifacts/retrieve)             |
| `claude_proj_doc_*` ID         | 项目文档的纯文本内容        | [获取项目文档内容](https://platform.claude.com/docs/zh-CN/api/compliance/apps/projects/documents/retrieve)           |
| `claude_proj_doc_*` ID         | 仅项目文档的元数据         | [获取项目文档元数据](https://platform.claude.com/docs/zh-CN/api/compliance/apps/projects/documents/metadata)          |

文件内容端点以分块二进制响应的形式流式传输 claude.ai 为该文件存储的内容。该内容并不总是与用户上传的文件完全相同。图像可能以处理后的副本而非上传的原始字节提供。某些附加到聊天的文档（例如 Word 文件、PowerPoint 文件和部分 PDF）以 claude.ai 从中提取的文本形式存储。对于这些文档，端点会以原始文件名返回提取的文本，而原始文档无法通过 Compliance API 获取。`size_bytes` 和 `md5` 字段描述的是存储的内容而非上传的文件。文件名和 `mime_type` 仍可能标示上传文档的格式。请根据返回的字节而非其名称或声明的类型来识别文件格式。

响应包含以下标头：

* `Content-Disposition: attachment; filename*=utf-8''<percent-encoded filename>` 以 RFC 5987 扩展形式携带原始上传文件名。扩展形式用于所有文件名，而不仅限于非 ASCII 文件名。
* `Content-Type` 携带为存储内容记录的 MIME 类型，对于以提取文本形式存储的文档，它仍可能标示原始文档格式。
* `Content-MD5` 携带所提供字节的 MD5 摘要，按 RFC 1864 的规定进行 base64 编码。
* `Transfer-Encoding: chunked` 始终会设置。

```bash cURL
file_id="claude_file_01UaT9wBcDfGhJkLmNpQrSv7"

curl --fail-with-body -sS -OJ \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  "https://api.anthropic.com/v1/compliance/apps/chats/files/$file_id/content"
```

`-OJ` 标志告诉 curl 以 `Content-Disposition` 中的文件名保存响应，即用户上传的原始文件名。

artifact 内容端点返回一个 artifact 版本的文本正文。请传入助手消息 `artifacts` 数组中某个条目的 `version_id`，而不是 artifact 的稳定 `id`。artifact 的每个新版本都有自己的 `version_id`，Compliance API 会提供该版本的精确字节。

## 检索项目和附件

项目将相关聊天与自定义指令、知识库内容以及附加的文件或文本文档捆绑在一起。Compliance API 公开项目元数据、项目详情以及属于某个项目的附件列表。

* [列出项目](https://platform.claude.com/docs/zh-CN/api/compliance/apps/projects/list)
* [获取项目详情](https://platform.claude.com/docs/zh-CN/api/compliance/apps/projects/retrieve)
* [列出项目附件](https://platform.claude.com/docs/zh-CN/api/compliance/apps/projects/attachments/list)
* [获取项目文档内容](https://platform.claude.com/docs/zh-CN/api/compliance/apps/projects/documents/retrieve)

项目结果按创建日期升序排序。附件结果按 `created_at` 升序排序，相同值时按 `id` 排序。项目列表和附件列表响应使用不透明的 `next_page` 页面令牌进行分页，而不是聊天和 Activity Feed 所使用的 `first_id`/`last_id` 游标。请在下一次请求中将该令牌作为 `page` 查询参数传回。

### 项目文件与项目文档

项目附件有两种不同的形态，通过每个条目上的 `type` 判别字段来识别：

`type` 为 `project_file` 的条目是文件上传（PDF、图像、电子表格），其 ID 以 `claude_file_` 开头；使用[下载文件内容](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/files/download)下载它们。`type` 为 `project_doc` 的条目是纯文本文档（始终为 `text/plain`），其 ID 以 `claude_proj_doc_` 开头，包括 Word 文件等在添加到项目时由 claude.ai 转换为文本的文档；使用[获取项目文档内容](https://platform.claude.com/docs/zh-CN/api/compliance/apps/projects/documents/retrieve)获取它们。

遍历附件列表的使用方必须根据 `type` 进行分支，并为每个条目调用对应的内容端点。以下请求列出一页附件；通过将 `next_page` 作为 `page` 参数传回进行分页，直到 `has_more` 为 `false`。

```bash cURL
project_id="claude_proj_01KGp4eZNug9ri4kE35RSppq"

curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/apps/projects/$project_id/attachments" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY"
```

```json Response
{
  "data": [
    {
      "id": "claude_file_01UaT9wBcDfGhJkLmNpQrSv7",
      "created_at": "2026-04-10T08:09:10Z",
      "filename": "dashboard_mockup_v1.pdf",
      "mime_type": "application/pdf",
      "size_bytes": 482133,
      "md5": "56367e4d2705cc9c025ad07424e944f0",
      "type": "project_file"
    },
    {
      "id": "claude_proj_doc_01YnT8sBcWvUtXzQpMkRfDgH",
      "created_at": "2026-04-10T08:09:11Z",
      "filename": "requirements.md",
      "mime_type": "text/plain",
      "type": "project_doc"
    }
  ],
  "has_more": false,
  "next_page": null
}
```

## 删除内容

<Warning>
  每次成功的删除都是永久且立即生效的。没有恢复窗口。
</Warning>

Compliance API 为聊天、文件、项目文档和整个项目公开了硬删除端点。硬删除的聊天无法恢复，并且此后不再出现在列表响应中。

* [删除聊天](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/delete)：同时移除该聊天的消息以及附加到这些消息的任何文件。
* [删除文件](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/files/delete)：同时处理聊天文件和项目文件。
* [删除项目文档](https://platform.claude.com/docs/zh-CN/api/compliance/apps/projects/documents/delete)：按 ID 移除单个项目文档。
* [删除项目](https://platform.claude.com/docs/zh-CN/api/compliance/apps/projects/delete)：请参阅[删除项目前先分离聊天](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data#detach-chats-before-deleting-a-project)。

这四个端点都需要 `delete:compliance_user_data` 作用域，该作用域在创建 Compliance Access Key 时与读取作用域分开授予。

以下请求删除一个聊天。相同的模式适用于其他删除端点；只有 URL 不同。

```bash cURL
# 警告：此操作将永久删除该聊天、其所有消息
# 以及任何附加文件。删除立即生效且无法撤销。它
# 需要 `delete:compliance_user_data` 权限范围，该范围在创建 Compliance Access Key 时
# 与 `read:compliance_user_data` 分开授予。
# 运行此操作前，请确保您已获得明确授权。

chat_id="claude_chat_01H5CWunD7RpVJ5bHa8RCkja"

curl --fail-with-body -sS -X DELETE \
  "https://api.anthropic.com/v1/compliance/apps/chats/$chat_id" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY"
```

```json Response
{
  "id": "claude_chat_01H5CWunD7RpVJ5bHa8RCkja",
  "type": "claude_chat_deleted"
}
```

每次成功的删除都会返回一个包含 `id` 和 `type` 判别字段的小型确认信封。聊天端点返回 `claude_chat_deleted`；在将删除视为已确认之前，请检查 `type` 字段。其他端点返回的确切 `type` 值请参阅各删除端点 [API 参考](https://platform.claude.com/docs/zh-CN/api/compliance/apps)页面上的响应模式。

### 删除项目前先分离聊天

当仍有任何聊天附加到项目时，该项目无法删除。API 会返回 409 及以下正文：

```json
{
  "error": {
    "type": "conflict_error",
    "message": "The \"claude_proj_01KGp4eZNug9ri4kE35RSppq\" project cannot be deleted as it has chats attached to it. Delete or detach all chats, and try deleting the project again."
  }
}
```

若要解决此问题，请使用 `GET /v1/compliance/apps/chats?user_ids[]={user_id}&project_ids[]={project_id}` 列出该项目的聊天（`project_ids[]` 筛选器需要至少一个 `user_ids[]` 值；通过[列出组织用户](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data#list-organization-users)枚举 ID），使用 `DELETE /v1/compliance/apps/chats/{claude_chat_id}` 删除每一个聊天（或在 claude.ai 中将其移出项目），然后重试项目删除。

## 后续步骤

<CardGroup cols={2}>
  <Card title="API 参考" href="https://platform.claude.com/docs/zh-CN/api/compliance/apps">
    每个聊天、文件、项目和 artifact 端点的完整请求和响应模式。
  </Card>

  <Card title="检索会话记录" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions">
    列出您的用户在 Claude 应用和智能体（例如 Cowork 和 Claude Code）中运行的会话，并检索其记录。
  </Card>

  <Card title="列出组织、用户、角色、群组和设置" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data">
    枚举与本页面上的聊天和项目相关联的人员和团队。
  </Card>
</CardGroup>
