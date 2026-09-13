---
title: 处理 Compliance API 错误
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors
description: 按 HTTP 状态码整理的每条 Compliance API 错误消息，包含原因和修复方法。
---

<Note>
  要启用 Compliance API，请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。
</Note>

本页列出了每个已记录的 Compliance API 端点返回的响应消息、原因以及修复方法。

Compliance API 以标准的 [Anthropic 错误格式](https://platform.claude.com/docs/zh-CN/api/errors)返回错误：一个非 2xx 状态码、一个 `request-id` 响应头，以及一个 JSON 正文，其中包含带有 `type` 和 `message` 的 `error` 对象。当您向支持团队上报问题时，请附上 `request-id` 响应头的值。

```json
{
  "error": {
    "type": "authentication_error",
    "message": "The API key provided is invalid or has been revoked."
  }
}
```

在本页中，本地会话（local sessions）运行在用户的机器上，远程会话（remote sessions）运行在云端；请参阅[检索会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions)。

请根据 `error.type` 进行匹配，而不是根据消息字符串。消息足够稳定，可以复制到运行手册中，但可能会随时间改写措辞；而 type 值是 API 契约的一部分。本地会话端点有少数已记录的例外情况，其中共享同一 type 的响应需要通过消息来区分；每种情况都会在适用之处加以说明。

下表让您一目了然地了解是否应重试。随后的每个部分都会展示逐字的错误正文和修复方法。

| 状态                                                                                                                            | 是否重试？                | 何时                                                                                                                                                                                                                                                                                                                                                                                                               |
| ----------------------------------------------------------------------------------------------------------------------------- | -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)                     | 否                    | 修正请求后重新发送。                                                                                                                                                                                                                                                                                                                                                                                                       |
| [401 Unauthorized](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#401-unauthorized)                   | 否                    | 修正或轮换密钥，然后重新发送。                                                                                                                                                                                                                                                                                                                                                                                                  |
| [403 Forbidden](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)                         | 否                    | 添加缺失的作用域或使用正确的密钥类型，然后重新发送。                                                                                                                                                                                                                                                                                                                                                                                       |
| [404 Not Found](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#404-not-found)                         | 通常否                  | 资源已被删除或从未存在；将其从您的队列中移除。例外情况：在本地会话端点上，消息 `Local sessions are not available.`（在每次调用时返回，包括列表调用）表示这些端点当前对您的父组织不可用，而不是某个会话已消失；请保留您队列中的 ID，并参阅[未找到本地会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-session-not-found)。仍处于 `pending` 状态的远程会话在启动之前，其 messages 端点会返回 404；请参阅[未找到远程会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#remote-session-not-found)。 |
| [409 Conflict](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#409-conflict)                           | 否                    | 请求与资源的当前状态冲突；解决冲突（例如分离子资源），然后重试。                                                                                                                                                                                                                                                                                                                                                                                 |
| [429 Too Many Requests](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#429-too-many-requests)         | 是，在 `retry-after` 之后 | 等待 `retry-after` 中的秒数，然后重试；不要推进您的游标。                                                                                                                                                                                                                                                                                                                                                                             |
| [500 Internal Server Error](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#500-internal-server-error) | 取决于 `x-should-retry` | 重试前检查 `x-should-retry` 响应头。                                                                                                                                                                                                                                                                                                                                                                                      |
| [502, 503, 504, 529](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#500-internal-server-error)        | 是，使用退避               | 瞬时错误；使用指数退避重试。例外情况：某些本地会话 503 不是瞬时的。请参阅[本地会话暂时不可用](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-sessions-temporarily-unavailable)。                                                                                                                                                                                                                                               |

## 400 Bad Request

请求在语法上有效，但包含服务器拒绝的参数。修正该参数后重试。

### 无效的时间戳格式

**Type：** `invalid_request_error`

```text wrap
The `created_at.gte` parameter contains an invalid timestamp format. Timestamps must be provided in RFC 3339 format e.g., "2024-03-01T00:00:00Z". Got "2024-01-01".
```

**原因：** 某个 `created_at.*` 或 `updated_at.*` 值（`.gte`、`.gt`、`.lte`、`.lt`）无法解析为日期时间。消息会指出失败的参数名称，并回显所发送的值。

**修复：** 发送包含时间和时区的完整 RFC 3339 时间戳，例如 `2024-03-01T00:00:00Z` 或 `2024-03-01T00:00:00+00:00`。

本地会话列表（`GET /v1/compliance/apps/sessions/local`）在同时提供两个时间边界且 `created_at.lt` 不严格晚于 `created_at.gte` 时，也会返回 400 `invalid_request_error`。正文内容为：

```text wrap
created_at.lt must be strictly after created_at.gte.
```

发送一个晚于 `created_at.gte` 的 `created_at.lt`，或省略其中一个边界。

### 无效的 limit

**Type：** `invalid_request_error`

```text wrap
The limit parameter must be between 1 and 1000, inclusive. Got 1500.
```

**原因：** `limit` 查询参数超出了可接受的范围。消息中指出的边界反映了所调用的特定端点的最大值。

**修复：** 发送端点可接受范围内的 `limit`。每个列表端点都有自己的 `limit` 范围；请参阅相应 [Compliance API 参考](https://platform.claude.com/docs/zh-CN/api/compliance)页面上的参数约束。

会话记录端点（`GET /v1/compliance/apps/sessions/local/{session_id}/messages` 和 `GET /v1/compliance/apps/sessions/remote/{session_id}/messages`）以相同方式验证其截断参数：`tool_use_input_max_bytes` 和 `tool_result_max_bytes` 各自接受一个正的字节数或 `-1`（服务器最大值），因此诸如 `0` 之类的值会返回相同的 400 `invalid_request_error`。

### 无效的分页 ID

**Type：** `invalid_request_error`

```text wrap
Invalid `after_id`. No activity found for `after_id` "activity_invalid123"
```

**原因：** `after_id` 或 `before_id` 游标无法解码为不透明游标，也无法解析为活动 ID。

**修复：** 将分页游标视为不透明字符串。始终复制上一页返回的 `first_id` 或 `last_id` 值；当 `has_more` 为 `false` 时停止。不要根据对象 ID 构造游标。

目录、项目和会话端点（组织、用户、角色、角色权限、群组、群组成员、项目、项目附件、本地和远程会话以及会话消息）使用不透明的 `page` 令牌进行分页，而不是 `after_id` 和 `before_id`。同样的建议适用：原样传递上一响应中的 `next_page` 值，并在 `has_more` 为 `false` 时停止（或者，在不返回 `has_more` 的会话端点上，当 `next_page` 为 `null` 时停止）。格式错误的 `page` 令牌会返回与格式错误的 `after_id` 或 `before_id` 相同的 400 `invalid_request_error`。

两个分页的本地会话端点（列表端点和 messages 端点）对于任何无法解码的 `page` 值都会返回以下 400 `invalid_request_error`，例如在您存储后被截断或篡改的令牌，或由不同端点或在不同父组织下签发的令牌。在本地会话 messages 端点（`GET /v1/compliance/apps/sessions/local/{session_id}/messages`）上，每个 `page` 游标还绑定到签发它时所针对的会话和 `order`，因此为不同会话或排序顺序签发的游标会返回相同的正文：

```text wrap
The page parameter is not a valid cursor for this request.
```

messages 端点上的游标还会在遍历（walk，即对各页的一次完整遍历）开始 24 小时后过期。过期的游标会返回：

```text wrap
The page cursor has expired. Restart the walk without a page parameter; results will reflect the current retention boundary.
```

对于第一个正文，请将上一响应中未经修改的 `next_page` 值重新发送到签发它的端点和会话。对于过期的游标，请在不带 `page` 参数的情况下重新开始；新的遍历反映其开始时生效的保留边界，因此在此期间已超出保留期的消息将不再返回（请参阅[检索本地会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-a-local-session-transcript)）。

## 401 Unauthorized

`x-api-key` 请求头缺失或与已知密钥不匹配。作用域错误的有效密钥会改为返回 [403 Forbidden](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)。

### 无效的 API 密钥

**Type：** `authentication_error`

```text wrap
The API key provided is invalid or has been revoked.
```

**原因：** `x-api-key` 中的密钥不存在、已被删除或已被禁用。缺失或为空的 `x-api-key` 请求头会返回相同的正文，因此请同时检查您的密钥存储和该密钥的撤销状态。

**修复：** 确认密钥值，检查它是否未在 claude.ai（Compliance Access Keys）或 Claude Console（Admin API keys）中被删除，并确认它已启用。请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。

## 403 Forbidden

`x-api-key` 中的密钥有效，但不具备端点所需的作用域。逐字消息会列出密钥所具备的作用域（`Got:`）和端点所需的作用域（`Needed:`），因此您无需重新检查 Claude Console 或 claude.ai 即可确认密钥所具备的作用域。Compliance Access Key 的作用域在创建后不可更改，因此每个作用域不足的修复方法都会指引您创建新密钥，而不是编辑现有密钥。独立的 Claude Console 组织（没有父组织的组织）无法创建 Compliance Access Key，因此需要该密钥的修复方法不适用于它；它只能查询 Activity Feed。

### 作用域不足：Activity Feed

**Type：** `permission_error`

```text wrap
Missing required scopes. Got: ['read:compliance_user_data'] Needed: ['read:compliance_activities']
```

**原因：** 使用了不带 `read:compliance_activities` 的密钥调用 `GET /v1/compliance/activities`。导致此错误的常见途径有两种：

* Compliance Access Key（`sk-ant-api01-...`）在创建时未包含 `read:compliance_activities` 作用域。
* Claude Console Admin API key（`sk-ant-admin01-...`）是在组织未启用 Compliance API 时创建的。在 Compliance API 未启用时创建的密钥不具备该作用域；请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)。

**修复：** Compliance Access Key 的作用域在创建后不可更改。创建一个包含 `read:compliance_activities` 的新密钥，或使用 Claude Console Admin API key。有关 Admin API key 在何种条件下具备此作用域，请参阅[您需要哪种密钥？](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#which-key-do-you-need)。

### 作用域不足：组织数据

**Type：** `permission_error`

```text wrap
Missing required scopes. Got: ['read:compliance_user_data'] Needed: ['read:compliance_org_data']
```

**原因：** 使用了不带 `read:compliance_org_data` 的密钥调用组织、角色、群组或有效设置端点。导致此错误的常见途径有两种：

* Compliance Access Key（`sk-ant-api01-...`）在创建时未包含 `read:compliance_org_data` 作用域。
* 使用了 Claude Console Admin API key（`sk-ant-admin01-...`）。Admin API key 仅具备 `read:compliance_activities`，无法读取组织元数据。

**修复：** [创建一个新的 Compliance Access Key](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)，并选中 `read:compliance_org_data`。Admin API key 无法读取组织元数据；必须使用 Compliance Access Key。

### 已停用的作用域：组织设置

**Type：** `permission_error`

```text wrap
Missing required scopes. Got: ['read:compliance_org_settings'] Needed: ['read:compliance_org_data']
```

**原因：** `read:compliance_org_settings` 作用域已于 2026 年 6 月 30 日停用。`GET /v1/compliance/organizations/{organization_id}/settings` 现在需要 `read:compliance_org_data`，与其他组织端点相同的作用域，而已停用的作用域不再授权任何操作。仅具备 `read:compliance_org_settings` 的 Compliance Access Key 在每次调用设置端点时都会返回此错误，即使该密钥在停用之前可以正常工作。创建密钥时无法再选择或授予已停用的作用域。

**修复：** Compliance Access Key 的作用域在创建后不可更改。[创建一个新的 Compliance Access Key](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)，并选中 `read:compliance_org_data`，更新您的集成以使用它，然后删除旧密钥。已具备 `read:compliance_org_data` 的密钥不受此次停用的影响。

### 作用域不足：用户数据

**Type：** `permission_error`

```text wrap
Missing required scopes. Got: ['read:compliance_activities'] Needed: ['read:compliance_user_data']
```

**原因：** 使用了不带 `read:compliance_user_data` 的密钥调用聊天、消息、文件、项目、会话、组织用户或群组成员端点。导致此错误的常见途径有两种：

* Compliance Access Key（`sk-ant-api01-...`）在创建时未包含 `read:compliance_user_data` 作用域。
* 使用了 Claude Console Admin API key（`sk-ant-admin01-...`）。Admin API key 仅具备 `read:compliance_activities`，且无法被授予 `read:compliance_user_data`，因此它们无法调用聊天、文件、项目、项目附件、会话、用户或群组成员端点。

**修复：** 使用在 claude.ai 中创建并选中 `read:compliance_user_data` 的 [Compliance Access Key](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)。如果该请求确实应仅限于 Activity Feed，请改为将 Admin API key 指向 `GET /v1/compliance/activities`。

### 作用域不足：删除

**Type：** `permission_error`

```text wrap
Missing required scopes. Got: ['read:compliance_user_data'] Needed: ['delete:compliance_user_data']
```

**原因：** 使用了不带 `delete:compliance_user_data` 的 Compliance Access Key 调用聊天、文件或项目上的 `DELETE` 端点。

**修复：** [创建一个新的 Compliance Access Key](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)，并选中 `delete:compliance_user_data`。删除作用域与 `read:compliance_user_data` 是分开的，以便只读审计密钥无法删除内容。

## 404 Not Found

端点已解析，但资源 ID 不存在或已被删除。Compliance API 的删除是立即且永久的，因此对先前已知 ID 返回 404 通常意味着内容已通过 Compliance API 删除调用被硬删除，或被保留策略移除。会话端点增加了两种情况。在本地会话端点上，当这些端点对您的父组织不可用时，每次调用（包括列表调用）都会返回一条单独的 404 消息 `Local sessions are not available.`；它不依赖于会话 ID，并且可能是暂时的。请参阅[未找到本地会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-session-not-found)。在远程会话端点上，仍在配置中的会话（`status` 为 `pending`）尚无记录，因此其 messages 端点在会话启动之前会返回 404。请参阅[未找到远程会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#remote-session-not-found)。每个修复方法中引用的活动类型字符串（例如 `claude_chat_created`）是您可以传递给 Activity Feed `activity_types[]` 过滤器的值；有关所有支持的值，请参阅[查询合规活动](https://platform.claude.com/docs/zh-CN/api/compliance/activities/list)。

### 未找到聊天

**Type：** `not_found_error`

```text wrap
Chat claude_chat_01H5CWunD7RpVJ5bHa8RCkja not found.
```

**原因：** 路径中的聊天 ID 与可通过 Compliance API 读取的聊天不匹配。该聊天可能已通过先前的 Compliance API 调用被硬删除，或被您组织的保留策略移除，也可能属于调用密钥无法读取的组织。用户在 claude.ai 中删除的聊天不会返回 404；它们仍然可读，`deleted_at` 已填充，但不包含其消息内容。

**修复：** 对照最近的 `claude_chat_created` 或 `claude_chat_viewed` 活动确认聊天 ID。如果活动是最近的而读取仍然失败，则该聊天已被硬删除（通过此 API 或因保留策略到期），或属于您密钥作用域之外的组织。

### 未找到文件

**Type：** `not_found_error`

```text wrap
No file found with provided id, or it has already been deleted.
```

**原因：** 文件 ID 不存在或已被删除。此错误同时适用于聊天附件文件（`claude_file_...`）和项目文件。

**修复：** 对照最近的 `claude_file_uploaded` 或 `claude_file_deleted` 活动进行核对。如果文件已被删除，则二进制内容已不存在；活动记录会在 6 年保留窗口内保留在 feed 中。

### 未找到项目

**Type：** `not_found_error`

```text wrap
No project is found with the provided id.
```

**原因：** 项目 ID 不存在或已被删除。

**修复：** 对照最近的 `claude_project_created` 或 `claude_project_deleted` 活动进行核对。即使项目本身已不存在，Activity Feed 仍会继续公开该项目的生命周期事件。

### 未找到项目文档

**Type：** `not_found_error`

```text wrap
No project document found with provided id, or it has already been deleted.
```

**原因：** 项目文档 ID 不存在或已被删除。此错误适用于文本项目文档（`claude_proj_doc_...`），不适用于项目文件。

**修复：** 使用 `GET /v1/compliance/apps/projects/{project_id}/attachments` 列出当前附件。如果文档缺失，则它已被删除；如果您只需要元数据，可通过 `claude_project_document_uploaded` 活动记录检索。

### 未找到本地会话

**Type：** `not_found_error`

```text wrap
Local session not found.
```

**原因：** 传递给 `GET /v1/compliance/apps/sessions/local/{session_id}` 或 `GET /v1/compliance/apps/sessions/local/{session_id}/messages` 的会话 ID 与可通过 Compliance API 读取的本地会话不匹配。在以下情况下，两个端点都会返回这同一条消息而不区分原因：该 ID 不是您的密钥可读取的组织中的会话（包括属于另一个父组织的 ID）、该会话从未存在、该会话适用[零数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#zero-data-retention-zdr-scope)，或该会话的所有活动都已超出适用于运行它的组织的保留期。`Local session not found.` 响应没有瞬时形式，因为本地会话没有配置中（`pending`）状态；请对比[未找到远程会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#remote-session-not-found)，其中 `pending` 会话在启动之前会返回 404。不是格式正确的 `clls_` 标识符的会话 ID 会改为返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。

当本地会话端点本身对您的父组织不可用时，这些端点（包括列表端点）会返回一条不同的 404 消息 `Local sessions are not available.`。该响应不依赖于会话 ID；客户侧的任何密钥、作用域或设置都无法改变它，并且它可能是暂时的。两种响应都带有 `not_found_error` type；区分它们的是消息文本。

**修复：** 对照 `GET /v1/compliance/apps/sessions/local` 确认会话 ID；请参阅[用户机器上的会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)。如果该会话不再出现在列表中，则其内容已超出保留期（或该会话因其他原因不再位于您的密钥可读取的组织中），其记录无法检索；请将该 ID 从您的队列中移除。如果每次调用（包括列表调用）都返回 `Local sessions are not available.`，请保留您队列中的会话 ID，并在下一次计划运行时重试；如果该响应持续存在，请联系您的 Anthropic 代表并附上 `request-id` 响应头。

### 未找到远程会话

**Type：** `not_found_error`

```text wrap
Remote session not found.
```

**原因：** 传递给 `GET /v1/compliance/apps/sessions/remote/{session_id}/messages` 的会话 ID 与可通过 Compliance API 读取的会话记录不匹配。这发生在以下情况：会话 ID（`cse_...`）不存在或会话已被删除、会话属于您的密钥无法读取的组织，或会话的 `status` 仍为 `pending`：pending 会话尚无记录，因此 messages 端点在会话启动之前会返回 404。不是格式正确的 `cse_` 标识符的会话 ID 会改为返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。

**修复：** 对照 `GET /v1/compliance/apps/sessions/remote` 确认会话 ID 及其 `status`；请参阅[云端会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-remote-sessions)。如果会话为 `pending`，请在其离开该状态后重试。如果该会话不再出现在列表中，则它已被删除，其记录无法检索。

### 未找到组织、角色或群组

**Type：** `not_found_error`

```text wrap
The "ce86b5f3-7c16-48b3-a9f3-e1d2c4b8a0f1" organization does not exist or the requester is not authorized to access it.
```

组织、角色和群组端点以标准错误格式返回 404 `not_found_error`。组织消息会指出 `org_uuid`；角色和群组消息是通用的（`Role not found.`、`Group not found.`）。当路径 ID（`org_uuid`、`role_id` 或 `group_id`）不存在或不再属于调用密钥可读取的树时，会发生这种情况。

**原因：** 路径中的 ID 与可通过 Compliance API 读取的记录不匹配。角色和群组可以被删除，组织可以从父树中取消关联。

**修复：** 对照相应的列表端点验证 ID，并对照 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed) 中最近的组织、角色或群组活动进行核对。

### 组织设置不可用

**Type：** `not_found_error`

```text wrap
organization `91012d09-e48b-438e-a489-1bebfd8fa6f9` not found in this organization's hierarchy
```

**原因：** `GET /v1/compliance/organizations/{organization_id}/settings` 在三种情况下返回此 404，这三种情况有意共享相同的正文，以便响应不会泄露某个组织是否存在：`organization_id` 不是您父组织的关联组织之一、该值不是有效的 UUID，或设置端点尚未为您的父组织启用。

**修复：** 对照[列出组织](https://platform.claude.com/docs/zh-CN/api/compliance/organizations/list)验证 ID。如果已知有效的组织 ID 仍返回 404，则设置端点尚未为您的父组织启用；请联系您的 Anthropic 代表。

## 409 Conflict

请求格式正确且已授权，但与资源的当前状态冲突。

### 项目有附加的聊天

**Type：** `conflict_error`

```text wrap
The "claude_proj_01KGp4eZNug9ri4kE35RSppq" project cannot be deleted as it has chats attached to it. Delete or detach all chats, and try deleting the project again.
```

**原因：** 对仍有聊天附加的项目调用了 `DELETE /v1/compliance/apps/projects/{project_id}`。

**修复：** 使用 `GET /v1/compliance/apps/chats?user_ids[]={user_id}&project_ids[]={project_id}` 列出项目的聊天（`project_ids[]` 过滤器需要至少一个 `user_ids[]` 值；请通过[列出组织用户](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data#list-organization-users)枚举 ID），使用 `DELETE /v1/compliance/apps/chats/{claude_chat_id}` 逐一删除，然后重试项目删除。

## 429 Too Many Requests

对 Compliance API 的请求限制为**每个[父组织](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api#how-the-compliance-api-works)每分钟 600 个请求**。该限制是一个在父组织下所有密钥（Compliance Access Keys 以及所有关联组织的 Admin API keys）之间以及所有 `/v1/compliance/*` 端点之间共享的预算；远程会话端点在此之上还有第二个请求预算。对于没有父组织的独立 Claude Console 组织，相同的预算适用于该组织本身，并在其 Admin API keys 之间共享。如果您的集成需要更高的限制，请联系您的 Anthropic 代表。

一旦您的 API 密钥通过身份验证，Compliance API 响应就会通过标准的[速率限制响应头](https://platform.claude.com/docs/zh-CN/api/rate-limits#response-headers)报告共享预算，以便您的客户端可以主动限流，而不是等待 429：

* `anthropic-ratelimit-requests-limit` 是每分钟的请求预算。
* `anthropic-ratelimit-requests-remaining` 是当前窗口中剩余的预算。
* `anthropic-ratelimit-requests-reset` 是窗口重置并恢复全部预算时的 RFC 3339 时间戳。

429 响应还带有一个 `retry-after` 响应头，其中包含发送下一个请求之前需要等待的秒数。该值可能在 `anthropic-ratelimit-requests-reset` 之外包含一个小的安全余量；请遵循 `retry-after`。

```http
HTTP/1.1 429 Too Many Requests
date: Tue, 21 Apr 2026 14:38:02 GMT
retry-after: 25
anthropic-ratelimit-requests-limit: 600
anthropic-ratelimit-requests-remaining: 0
anthropic-ratelimit-requests-reset: 2026-04-21T14:38:25Z
```

```json
{
  "error": {
    "type": "rate_limit_error",
    "message": "Compliance API rate limit of 600 requests per minute per parent organization has been exceeded. Retry after the time indicated by the retry-after header. Quote the request-id response header when contacting Anthropic support."
  }
}
```

**原因：** 您的父组织（或独立 Claude Console 组织）在 1 分钟窗口内，跨所有共享其预算的密钥，向 `/v1/compliance/*` 发送了超过 600 个请求，或者耗尽了远程会话端点的第二个请求预算（本节稍后介绍）。

**修复：** 等待 `retry-after` 响应头中的秒数，然后重试。如果该响应头不存在（例如被中间层剥离），请回退到指数退避（从 1 秒开始，加倍直至 60 秒）。不要在 429 时推进您的分页游标：失败的请求未返回任何数据，因此上一个成功页面的游标仍然正确。

身份验证失败的请求（缺失或无法识别的密钥，或使用 Claude API 密钥而非 Compliance Access Key 或 Admin API key）会在速率限制器之前被拒绝，不消耗配额。缺少端点所需作用域的有效密钥会在返回 403 之前消耗一个配额单位。

[本地会话端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)仅计入共享限制。[远程会话端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-remote-sessions)在此之上还有第二个请求预算，与共享限制一样以您的父组织为键。来自该预算的 429 带有一个始终为 `1` 的 `retry-after` 响应头（这是最小等待时间，而非实际重置时间）；该响应上的任何 `anthropic-ratelimit-*` 响应头描述的是共享限制而非此预算，因此如果 429 重复出现，请进行指数退避。

如果您按计划轮询 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)，请将您的总请求速率（跨所有密钥、关联组织和并发工作进程）控制在共享限制以下。观察 `anthropic-ratelimit-requests-remaining`，以便在达到限制之前减速。有关在窗口轮询和游标驱动摄取之间进行选择，请参阅[设计您的合规集成](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns#choose-a-feed-consumption-pattern)。

## 500 Internal Server Error

当失败是确定性的时，来自 Compliance API 的 500 会带有 `x-should-retry: false` 响应头。Anthropic SDK 会自动遵循此响应头。如果您使用对每个 5xx 都重试的通用 HTTP 重试库，请在 `x-should-retry` 为 `false` 时抑制重试；重试此错误在每次尝试时都会以相同方式失败。

不带 `x-should-retry: false` 响应头的 500 是瞬时的：请使用指数退避重试（从 1 秒开始，加倍直至 60 秒）。502、503、504 和 529 响应同样适用。例外是接下来介绍的一小部分本地会话 503，它们取决于组织的设置或加密密钥，而非负载。有关平台范围的重试语义，请参阅[错误](https://platform.claude.com/docs/zh-CN/api/errors)。

### 本地会话暂时不可用

**Type：** `overloaded_error`

```text wrap
The local-sessions index is temporarily unavailable. Try again shortly.
```

```text wrap
Captured content is temporarily unavailable. Try again shortly.
```

```text wrap
The local-sessions index cannot currently evaluate retention overrides for this page. Try again later.
```

**原因：** [本地会话端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)返回带有以下正文之一的 503。三者共享 `overloaded_error` type，因此这是本页上少数需要通过消息文本而非 `error.type` 来区分情况的错误之一：

* `index is temporarily unavailable` 正文表示会话列表因负载或后端状况而短暂不可用。这是瞬时的。
* `Captured content` 正文表示某个会话的记录内容目前无法返回。这通常也是瞬时的。在使用[客户管理的加密密钥](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)的组织中，messages 端点还会对每个包含您的密钥无法解密的内容的页面返回此正文，例如因为您禁用、撤销或销毁了该密钥，或因为无法访问该密钥。在这种情况下，只要密钥无法使用，错误就会持续存在。两种情况下的消息文本相同，因此表明密钥是原因的唯一信号是该错误在该组织中持续重复出现。无法使用的密钥永远不会被报告为 `not_captured`。
* `retention overrides` 正文表示适用于所请求范围内一个或多个会话的保留或数据处理设置尚无法评估。在检索和 messages 端点上，它显示为 `for this session` 而非 `for this page`。它取决于运行该会话的组织的数据和设置，而非负载，并且可能持续较长时间。

**修复：** 按如下方式处理每个正文：

* 对于两个 `Try again shortly.` 正文，请使用指数退避重试，并且不要推进您的 `page` 游标，因为失败的请求未返回任何数据。
* 如果 `Captured content` 正文在使用客户管理密钥的组织的 messages 端点上持续重复出现，请将其视为持久性错误：停止遍历该组织的记录，并在您的密钥管理服务中检查密钥状态。其他关联组织中的记录以及所有地方的会话元数据均不受影响。如果您在之后的运行中重试，请在不带 `page` 的情况下重新开始每个会话的遍历，因为 messages 页面游标会在遍历的第一页之后 24 小时过期。
* 对于 `Try again later.` 正文，不要保持遍历处于打开状态等待其清除。在列表端点上，要么稍后通过不带 `page` 参数重新开始来重试（超过 24 小时的列表页面令牌仍被接受，但会根据当前保留边界重新评估，因此搁置的遍历可能会跳过会话），要么缩小 `created_at.gte` 和 `created_at.lt` 窗口直到请求成功，并在之后的运行中单独导出跳过的范围。在检索和 messages 端点上，跳过该会话 ID，继续进行其余的导出，并在之后的运行中重试该会话。messages 页面游标会在遍历的第一页之后 24 小时过期，因此当您返回该会话时，请在不带 `page` 的情况下重新开始该会话的遍历。

如果这些情况中的任何一种在多次运行中重复出现，请联系您的 Anthropic 代表并附上 `request-id` 响应头。对于客户管理密钥的情况，仅当您的密钥可用而错误仍持续时才这样做。

对于服务范围的事件，请查看 [status.anthropic.com](https://status.anthropic.com)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="Compliance API 常见问题" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq">
    有关访问、作用域、保留和集成的常见问题。
  </Card>

  <Card title="错误" href="https://platform.claude.com/docs/zh-CN/api/errors">
    平台范围的错误目录和重试语义。
  </Card>
</CardGroup>
