---
title: 处理 Compliance API 错误
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors
description: 按 HTTP 状态码列出的 Compliance API 错误响应，以及每种错误的原因和修复方法。
---

<Note>
  要启用 Compliance API，请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。
</Note>

本页按 HTTP 状态码列出常见的 Compliance API 错误响应，以及每种错误的原因和修复方法。

Compliance API 以标准的 [Anthropic 错误格式](https://platform.claude.com/docs/zh-CN/api/errors)返回错误：一个非 2xx 状态码、一个 `request-id` 响应头，以及一个 JSON 正文，其中包含带有 `type` 和 `message` 的 `error` 对象。当您向支持团队上报问题时，请附上 `request-id` 响应头的值。

```json
{
  "error": {
    "type": "permission_error",
    "message": "Missing required scopes. Got: ['read:compliance_activities'] Needed one of: ['read:compliance_user_data', 'read:org_audit']"
  }
}
```

在本页中，本地会话（local sessions）运行在用户的机器上，远程会话（remote sessions）运行在云端；请参阅[检索会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions)。

请根据 HTTP 状态码和 `error.type` 进行匹配，而不是根据消息字符串。消息内容足够稳定，可以复制到运维手册（runbook）中，但可能会随时间调整措辞；状态码和类型值则属于 API 契约的一部分。少数共享相同状态码和类型的响应需要通过消息来区分；这些情况会在相应位置特别说明。

下表让您一目了然地了解是否应重试。随后的每个部分都会展示逐字的错误正文和修复方法。

| 状态                                                                                                                            | 是否重试？                | 何时                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ----------------------------------------------------------------------------------------------------------------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)                     | 否                    | 修正请求；如果消息表明 Compliance API 未启用，则启用它，然后重新发送。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| [401 Unauthorized](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#401-unauthorized)                   | 否                    | 密钥无法识别、已被停用或已过期；重新启用或替换密钥，然后重新发送。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| [403 Forbidden](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)                         | 否                    | 添加缺失的权限范围或使用正确的密钥类型，然后重新发送。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| [404 Not Found](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#404-not-found)                         | 通常否                  | 如果消息中指明了某个资源，则表示该资源已被删除或从未存在；请将其从您的队列中移除。仅为 `Not found` 的消息表示请求未通过身份验证（或路径不存在），而不是资源已不存在；请参阅[请求未通过身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#request-not-authenticated)。会话端点还有另外两种情况：在本地会话端点上，消息 `Local sessions are not available.`（每次调用都会返回，包括列表调用）表示这些端点当前对您的父组织不可用，而不是会话已不存在；请保留您队列中的 ID，并参阅[未找到本地会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-session-not-found)。仍处于 `pending` 状态的远程会话在启动之前，其消息端点会返回 404；请参阅[未找到远程会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#remote-session-not-found)。 |
| [409 Conflict](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#409-conflict)                           | 否                    | 请求与资源的当前状态冲突；解决冲突（例如解除子资源的关联），然后重试。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| [429 Too Many Requests](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#429-too-many-requests)         | 是，在 `retry-after` 之后 | 等待 `retry-after` 中指定的秒数，然后重试；不要推进您的游标。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| [500 Internal Server Error](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#500-internal-server-error) | 取决于 `x-should-retry` | 重试前请检查 `x-should-retry` 响应头。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| [502, 503, 504, 529](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#500-internal-server-error)        | 是，使用退避               | 暂时性错误；使用指数退避进行重试。例外：某些本地会话 503 错误并非暂时性的。请参阅[本地会话暂时不可用](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-sessions-temporarily-unavailable)。                                                                                                                                                                                                                                                                                                                                                                                                                             |

## 400 Bad Request

请求在语法上有效，但服务器拒绝了某个参数，或者该组织未启用 Compliance API。请修正消息中指出的原因，然后重新发送。

### Compliance API 未启用

**Type：** `invalid_request_error`

```text wrap
Compliance API is not enabled for this organization
```

**原因：** 密钥有效，但该密钥所属的组织或父组织未启用 Compliance API。在启用该 API 之前，每个端点都会返回此响应；如果管理员关闭了该 API，也会再次返回此响应。

**修复：** 按照[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api) 启用 Compliance API，然后重新发送请求。

### 未知的查询参数

**类型：** `invalid_request_error`

```text wrap
Unknown query parameter: 'created_at[gte]'. Did you mean 'created_at.gte'?
```

**原因：** 请求中包含了端点未定义的查询参数；Compliance API 会拒绝无法识别的参数，而不是忽略它们。消息会指出该参数的名称；对于相近的错误写法（例如用方括号表示法代替点号，或缺少 `[]` 后缀），消息还会建议正确的参数名称。

**修复：** 使用端点对应的 [Compliance API 参考](https://platform.claude.com/docs/zh-CN/api/compliance)页面中列出的参数名称。范围过滤器使用点号表示法（例如 `created_at.gte`），数组过滤器需要 `[]` 后缀（例如 `activity_types[]`），分页参数则根据端点不同为 `after_id`、`before_id` 或 `page`。

### 无效的参数值

**类型：** `invalid_request_error`

```text wrap
limit: Input should be less than or equal to 1000
```

```text wrap
created_at.gte: Input should be a valid datetime or date, invalid character in year
```

```text wrap
activity_types[].0: Input is not one of the permitted values.
```

**原因：** 某个查询参数的值未通过验证。消息以参数名称开头（对于数组参数，后面会跟上元素的位置），然后说明未满足的约束条件。上面展示了三种常见情况：`limit` 超过了端点的最大值（消息中的数字即该端点的最大值）；`created_at.*` 或 `updated_at.*` 的值无法解析为日期或时间戳；`activity_types[]` 的值不是受支持的活动类型。

**修复：** 修正消息中指出的参数。每个列表端点都有各自的 `limit` 范围；请参阅相应 [Compliance API 参考](https://platform.claude.com/docs/zh-CN/api/compliance)页面中的参数约束。请以 RFC 3339 格式发送时间戳，并带有明确的 UTC 偏移量，例如 `2024-03-01T00:00:00Z` 或 `2024-03-01T00:00:00+00:00`；本地会话列表会拒绝不带偏移量的时间戳（`created_at.gte: Input should have timezone info`）。有关受支持的 `activity_types[]` 值，请参阅[查询合规活动](https://platform.claude.com/docs/zh-CN/api/compliance/activities/list)。

本地会话列表（`GET /v1/compliance/apps/sessions/local`）在同时提供两个时间边界且 `created_at.lt` 不严格晚于 `created_at.gte` 时，也会返回 400 `invalid_request_error`。正文内容为：

```text wrap
created_at.lt must be strictly after created_at.gte.
```

发送一个晚于 `created_at.gte` 的 `created_at.lt`，或省略其中一个边界。

会话记录端点（`GET /v1/compliance/apps/sessions/local/{session_id}/messages` 和 `GET /v1/compliance/apps/sessions/remote/{session_id}/messages`）以相同方式验证其截断参数：`tool_use_input_max_bytes` 和 `tool_result_max_bytes` 均接受正的字节数或 `-1`（服务器最大值），因此诸如 `0` 之类的值会返回相同的 400 `invalid_request_error`。

### 无效的分页游标

**Type：** `invalid_request_error`

```text wrap
Invalid activity_id format: 'activity_invalid123'
```

```text wrap
Invalid pagination cursor for 'after_id'
```

**原因：** 某个 "pagination cursor"（分页游标）无法解码。在 Activity Feed 上，如果 `after_id` 或 `before_id` 的值既不是 API 签发的游标，也不是格式正确的活动 ID，则返回第一个正文，其中会回显所发送的值。在聊天和聊天消息端点上，如果 `after_id` 或 `before_id` 的值无法解码，则返回第二个正文，其中会指出该参数的名称。

**修复：** 将分页游标视为不透明字符串。始终复制上一页返回的 `first_id` 或 `last_id` 值；当 `has_more` 为 `false` 时停止。不要根据对象 ID 构造游标。

目录、项目和会话端点（组织、用户、角色、角色权限、组、组成员、项目、项目附件、本地和远程会话以及会话消息）使用不透明的 `page` 令牌进行分页，而不是 `after_id` 和 `before_id`。同样的建议也适用：原样传递上一个响应中的 `next_page` 值，并在 `has_more` 为 `false` 时停止（对于不返回 `has_more` 的会话端点，则在 `next_page` 为 `null` 时停止）。格式错误的 `page` 令牌会返回与格式错误的 `after_id` 或 `before_id` 相同的 400 `invalid_request_error`，但消息内容因端点而异。

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

请求携带的 Compliance Access Key（`sk-ant-api01-...`）或 Admin API 密钥（`sk-ant-admin01-...`）未能通过身份验证。如果请求未携带密钥，或携带的是其他类型的密钥，则除组织设置端点外，所有端点都会改为返回 [404 Not Found](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#request-not-authenticated)；而权限范围不正确的有效密钥会返回 [403 Forbidden](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)。

### API 密钥无效、已停用或已过期

**Type：** `authentication_error`

```text wrap
API key is invalid.
```

```text wrap
API key has been deactivated.
```

```text wrap
API key has expired.
```

**原因：** `API key is invalid.` 表示发送的值与任何可用的密钥都不匹配，例如因为该值在存储时被截断或修改。`API key has been deactivated.` 表示该密钥已被禁用或删除。`API key has expired.` 表示 Admin API 密钥已超过其过期日期；Compliance Access Key 在创建时不设置过期日期。

**修复：** 对于 `API key is invalid.`，请将客户端发送的值与创建密钥时您存储的密钥值进行比较；完整的密钥值只会显示一次，因此如果您存储的副本有误，请创建新密钥。对于 `API key has been deactivated.`，如果密钥只是被禁用，请重新启用它：Compliance Access Key 在 [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access) 中启用，Admin API 密钥在 [Claude Console > Settings > Admin keys](https://platform.claude.com/settings/admin-keys) 中启用；已删除的密钥无法恢复。对于已删除或已过期的密钥，请创建新密钥并更新您的集成以使用该密钥，具体请参阅[管理和轮换密钥](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#manage-and-rotate-keys)。

## 403 Forbidden

`x-api-key` 中的密钥有效，但不具备端点所接受的权限范围（scope）。原样消息会列出该密钥具备的权限范围（`Got:`）以及端点接受的权限范围（读取端点上为 `Needed one of:`，表示具备所列任意一个权限范围即可；删除端点上为 `Needed:`），因此您无需重新查看 Claude Console 或 claude.ai 即可确认密钥具备哪些权限范围。在读取端点上，接受列表中还包括 `read:org_audit`，这是一个只读审计权限范围，涵盖所有 Compliance API 读取端点；请参阅[为 Claude Enterprise 密钥选择权限范围](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys#choose-scopes-for-a-claude-enterprise-key)。Compliance Access Key 的权限范围在创建后不可更改，因此每个权限范围不足的修复方法都会指导您创建新密钥，而不是编辑现有密钥。独立的 Claude Console 组织（没有父组织的组织）无法创建 Compliance Access Key，因此需要此类密钥的修复方法不适用于它；它只能查询 Activity Feed。

### 作用域不足：Activity Feed

**Type：** `permission_error`

```text wrap
Missing required scopes. Got: ['read:compliance_user_data'] Needed one of: ['read:compliance_activities', 'read:org_audit']
```

**原因：** 使用了不带 `read:compliance_activities` 的密钥调用 `GET /v1/compliance/activities`。导致此错误的常见途径有两种：

* Compliance Access Key（`sk-ant-api01-...`）在创建时未包含 `read:compliance_activities` 作用域。
* Claude Console Admin API key（`sk-ant-admin01-...`）是在组织未启用 Compliance API 时创建的。在 Compliance API 未启用时创建的密钥不具备该作用域；请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)。

**修复：** Compliance Access Key 的作用域在创建后不可更改。创建一个包含 `read:compliance_activities` 的新密钥，或使用 Claude Console Admin API key。有关 Admin API key 在何种条件下具备此作用域，请参阅[您需要哪种密钥？](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#which-key-do-you-need)。

### 作用域不足：组织数据

**Type：** `permission_error`

```text wrap
Missing required scopes. Got: ['read:compliance_user_data'] Needed one of: ['read:compliance_org_data', 'read:org_audit']
```

**原因：** 使用了不带 `read:compliance_org_data` 的密钥调用组织、角色、群组或有效设置端点。导致此错误的常见途径有两种：

* Compliance Access Key（`sk-ant-api01-...`）在创建时未包含 `read:compliance_org_data` 作用域。
* 使用了 Claude Console Admin API key（`sk-ant-admin01-...`）。Admin API key 仅具备 `read:compliance_activities`，无法读取组织元数据。

**修复：** [创建一个新的 Compliance Access Key](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)，并选中 `read:compliance_org_data`。Admin API key 无法读取组织元数据；必须使用 Compliance Access Key。

### 已停用的作用域：组织设置

**Type：** `permission_error`

```text wrap
Missing required scopes. Got: ['read:compliance_org_settings'] Needed one of: ['read:compliance_org_data', 'read:org_audit']
```

**原因：** `read:compliance_org_settings` 作用域已于 2026 年 6 月 30 日停用。`GET /v1/compliance/organizations/{organization_id}/settings` 现在需要 `read:compliance_org_data`，与其他组织端点相同的作用域，而已停用的作用域不再授权任何操作。仅具备 `read:compliance_org_settings` 的 Compliance Access Key 在每次调用设置端点时都会返回此错误，即使该密钥在停用之前可以正常工作。创建密钥时无法再选择或授予已停用的作用域。

**修复：** Compliance Access Key 的作用域在创建后不可更改。[创建一个新的 Compliance Access Key](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)，并选中 `read:compliance_org_data`，更新您的集成以使用它，然后删除旧密钥。已具备 `read:compliance_org_data` 的密钥不受此次停用的影响。

### 作用域不足：用户数据

**Type：** `permission_error`

```text wrap
Missing required scopes. Got: ['read:compliance_activities'] Needed one of: ['read:compliance_user_data', 'read:org_audit']
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

如果 404 的消息中指明了某个资源或资源类型，则表示路径中的 ID 不存在或已被删除。Compliance API 的删除操作是即时且永久的，因此对先前已知的 ID 返回 404 通常意味着内容已不存在：已通过 Compliance API 删除调用被硬删除、已被保留策略移除，或者（对于文件和工件）已随其所属聊天被用户在 claude.ai 中删除。消息仅为 `Not found` 的 404 则不同：它表示请求未通过身份验证（或路径不存在），任何端点（包括列表端点）都可能返回此响应；请参阅[请求未通过身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#request-not-authenticated)。会话端点还有另外两种情况。在本地会话端点上，当这些端点对您的父组织不可用时，每次调用（包括列表调用）都会返回另一条 404 消息 `Local sessions are not available.`；该响应与会话 ID 无关，并且可能是暂时的。请参阅[未找到本地会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-session-not-found)。在远程会话端点上，仍在预配中的会话（`status` 为 `pending`）尚无会话记录，因此在会话启动之前，其消息端点会返回 404。请参阅[未找到远程会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#remote-session-not-found)。各修复方法中引用的活动类型字符串（例如 `claude_chat_created`）是可以传递给 Activity Feed `activity_types[]` 过滤器的值；有关所有受支持的值，请参阅[查询合规活动](https://platform.claude.com/docs/zh-CN/api/compliance/activities/list)。

### 请求未通过身份验证

**类型：** `not_found_error`

```text wrap
Not found
```

**原因：** 请求未携带 Compliance API 接受的凭据：未发送 API 密钥，或者密钥既不是 Compliance Access Key（`sk-ant-api01-...`）也不是 Admin API 密钥（`sk-ant-admin01-...`），例如 Claude API 密钥（`sk-ant-api03-...`）。其状态、类型和消息与路径不存在时相同，且与端点或任何资源 ID 无关，因此诸如 `GET /v1/compliance/activities` 之类的列表端点也会返回此正文。唯一的例外是 `GET /v1/compliance/organizations/{organization_id}/settings`，它对此类请求返回 401 `authentication_error`。在所有端点上，未能通过身份验证的 Compliance Access Key 或 Admin API 密钥都会改为返回 [401 Unauthorized](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#401-unauthorized)。

**修复：** 在 `x-api-key` 请求头中发送密钥，并检查其前缀：Compliance API 仅接受 `sk-ant-api01-...` 和 `sk-ant-admin01-...` 密钥；请参阅[您需要哪种密钥？](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#which-key-do-you-need)。如果请求头和密钥都正确，但某个路径仍返回 `Not found` 而其他路径正常，请对照 [Compliance API 参考](https://platform.claude.com/docs/zh-CN/api/compliance)检查该路径。

### 未找到聊天

**Type：** `not_found_error`

```text wrap
Chat conversation not found: 'claude_chat_01H5CWunD7RpVJ5bHa8RCkja'
```

**原因：** 路径中的聊天 ID 与可通过 Compliance API 读取的聊天不匹配。该聊天可能已通过先前的 Compliance API 调用被硬删除，或被您组织的保留策略移除，也可能属于调用密钥无法读取的组织。用户在 claude.ai 中删除的聊天不会返回 404；它们仍然可读，`deleted_at` 已填充，但不包含其消息内容。

**修复：** 对照最近的 `claude_chat_created` 或 `claude_chat_viewed` 活动确认聊天 ID。如果活动是最近的而读取仍然失败，则该聊天已被硬删除（通过此 API 或因保留策略到期），或属于您密钥作用域之外的组织。

### 未找到文件

**Type：** `not_found_error`

```text wrap
File not found: 0d3b8f72-6c1e-4a59-b2de-7f4c9a1e5b60
```

**原因：** 该文件 ID 在您的密钥可读取的组织中不存在，或者该文件已被删除。在 claude.ai 中删除聊天也会删除附加到该聊天的文件，尽管聊天本身仍会出现在列表中。消息通过文件的底层 UUID 来标识文件，而不是请求中发送的 `claude_file_...` ID。元数据、内容和删除端点都会返回此正文；它同时适用于聊天附件文件（`claude_file_...`）和项目文件。

**修复：** 对照最近的 `claude_file_uploaded` 或 `claude_file_deleted` 活动进行核对。随聊天一起删除的文件没有 `claude_file_deleted` 活动，因此还需检查该聊天的 `claude_chat_deleted` 活动。如果文件已被删除，则其二进制内容已不存在；活动记录会在 6 年保留期内保留在 Feed 中。

### 未找到生成的文件或工件

**类型：** `not_found_error`

```text wrap
Generated file not found: 'claude_gen_file_01TbR8wAcCeFhJkLnPqStUvX'
```

```text wrap
Generated file content not found: 'claude_gen_file_01TbR8wAcCeFhJkLnPqStUvX'
```

```text wrap
Artifact version not found: 'claude_artifact_version_01KmNpQrSt3UvWxYz5AbCdEfG'
```

**原因：** 路径中的 ID 与任何可通过 Compliance API 读取的工具生成文件或工件版本都不匹配。生成文件的元数据端点返回第一个正文，内容端点返回第二个正文，两个工件端点都返回第三个正文。生成的文件和工件会随其所在的聊天一起被删除，包括用户在 claude.ai 中删除该聊天的情况。

**修复：** 使用[获取聊天消息](https://platform.claude.com/docs/zh-CN/api/compliance/apps/chats/messages/list)查找该 ID 所属的聊天。如果该聊天的 `deleted_at` 已被填充，或者某个 `claude_chat_deleted` 活动指明了该聊天，则内容已不存在；请将该 ID 从您的队列中移除。否则，请对照该聊天消息中的 `generated_files` 和 `artifacts` 数组确认该 ID。

### 未找到项目

**Type：** `not_found_error`

```text wrap
No project is found with the provided id.
```

```text wrap
No project found with provided id, or it has already been deleted.
```

**原因：** 该项目 ID 不存在或已被删除。项目详情、附件和协作者端点返回第一个正文；`DELETE /v1/compliance/apps/projects/{project_id}` 返回第二个正文。

**修复：** 对照最近的 `claude_project_created` 或 `claude_project_deleted` 活动进行核对。即使项目本身已不存在，Activity Feed 仍会继续公开该项目的生命周期事件。

### 未找到项目文档

**Type：** `not_found_error`

```text wrap
No project document found with the provided id.
```

```text wrap
No project document found with the provided id, or it has already been deleted.
```

**原因：** 该项目文档 ID 不存在或已被删除。文档内容和元数据端点返回第一个正文；`DELETE /v1/compliance/apps/projects/documents/{document_id}` 返回第二个正文。此错误适用于文本项目文档（`claude_proj_doc_...`），不适用于项目文件。

**修复：** 使用 `GET /v1/compliance/apps/projects/{project_id}/attachments` 列出当前附件。如果文档缺失，则说明它已被删除；如果您只需要元数据，可以通过 `claude_project_document_uploaded` 活动记录检索。活动记录会显示谁上传了该文档、何时上传以及上传到哪个项目，但不包含其名称。

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

请求格式正确且已获授权，但与资源的当前状态冲突。正文带有 `invalid_request_error` 类型，400 响应也使用该类型，因此请通过 409 状态码而不是 `error.type` 来识别冲突。

### 项目有附加的聊天

**类型：** `invalid_request_error`

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

* `index is temporarily unavailable` 响应体表示由于负载或后端状况，会话列表暂时不可用。这是暂时性的。
* `Captured content` 响应体表示当前无法返回某个会话的对话记录内容。这通常也是暂时性的。在使用[客户管理的加密密钥](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)的组织中，如果某个页面包含您的客户管理密钥无法解密的内容（例如因为您禁用、撤销或销毁了该密钥，或者该密钥无法访问），messages 端点也会针对每个这样的页面返回此响应体。在这种情况下，只要该密钥无法使用，该错误就会一直存在。无论哪种情况，消息文本都相同，因此判断密钥是原因的唯一信号是该错误在该组织中反复出现。不可用的密钥永远不会被报告为 `not_captured`。
* `retention overrides` 响应体表示适用于所请求范围内一个或多个会话的保留或数据处理设置暂时无法评估。在 retrieve 和 messages 端点上，其内容为 `for this session`，而不是 `for this page`。它取决于运行该会话的组织的数据和设置，而非负载，并且可能会持续较长时间。

**修复：** 按如下方式处理每个正文：

* 对于两个 `Try again shortly.` 正文，请使用指数退避重试，并且不要推进您的 `page` 游标，因为失败的请求未返回任何数据。
* 如果 `Captured content` 正文在使用客户管理密钥的组织的 messages 端点上持续重复出现，请将其视为持久性错误：停止遍历该组织的记录，并在您的密钥管理服务中检查密钥状态。其他关联组织中的记录以及所有地方的会话元数据均不受影响。如果您在之后的运行中重试，请在不带 `page` 的情况下重新开始每个会话的遍历，因为 messages 页面游标会在遍历的第一页之后 24 小时过期。
* 对于 `Try again later.` 正文，不要保持遍历处于打开状态等待其清除。在列表端点上，要么稍后通过不带 `page` 参数重新开始来重试（超过 24 小时的列表页面令牌仍被接受，但会根据当前保留边界重新评估，因此搁置的遍历可能会跳过会话），要么缩小 `created_at.gte` 和 `created_at.lt` 窗口直到请求成功，并在之后的运行中单独导出跳过的范围。在检索和 messages 端点上，跳过该会话 ID，继续进行其余的导出，并在之后的运行中重试该会话。messages 页面游标会在遍历的第一页之后 24 小时过期，因此当您返回该会话时，请在不带 `page` 的情况下重新开始该会话的遍历。

如果上述任何情况在多次运行中反复出现，请联系您的 Anthropic 代表，并提供 `request-id` 响应头。对于客户管理密钥的情况，仅当该密钥可用时错误仍然持续，才需要这样做。

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
