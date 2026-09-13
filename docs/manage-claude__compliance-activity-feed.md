---
title: 查询活动源
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed
description: 检索、筛选和分页浏览您组织的 Compliance API 活动源。
---

<Note>
  要启用 Compliance API，请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。
</Note>

<Check>
  **所需作用域：** Compliance Access Key 或 Admin API 密钥上的 `read:compliance_activities`。

  携带此作用域的 Compliance Access Key（`sk-ant-api01-...`）和 Admin API 密钥（`sk-ant-admin01-...`）都可以调用活动源。请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)，了解每种密钥类型在何种条件下携带该作用域。
</Check>

"Activity Feed"（活动源）记录您组织范围内的身份验证、聊天、文件、项目、管理和平台活动，并按时间倒序返回。活动在发生后 1 分钟内即可查询，并保留 6 年。记录不具有追溯性：从您的组织首次启用 Compliance API 时才开始记录，启用之前的活动不会被回填。

```bash cURL
curl --fail-with-body -sS \
  "https://api.anthropic.com/v1/compliance/activities?limit=1" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY"
```

```json Response
{
  "data": [
    {
      "id": "activity_01XyDMpzjS89pFZXqSFUBDr6",
      "created_at": "2026-04-10T08:09:10Z",
      "organization_id": "org_01Wv6QeBcDfGhJkLmNpQrSt8",
      "organization_uuid": "abcdef01-2345-6789-abcd-ef0123456789",
      "actor": {
        "type": "user_actor",
        "email_address": "user@example.com",
        "user_id": "user_01TuVwXyZaBcDeFgH2JkLmN4",
        "ip_address": "192.0.2.34",
        "user_agent": "Mozilla/5.0..."
      },
      "type": "claude_chat_created",
      "claude_chat_id": "claude_chat_01XyDMpzjS89pFZXqSFUBDr6",
      "claude_project_id": "claude_proj_01KGp4eZNug9ri4kE35RSppq"
    }
  ],
  "has_more": true,
  "first_id": "activity_01XyDMpzjS89pFZXqSFUBDr6",
  "last_id": "activity_01XyDMpzjS89pFZXqSFUBDr6"
}
```

## 筛选活动

可按组织、操作者、活动类型进行筛选，或使用带点的子参数 `created_at.gte`、`.gt`、`.lte` 和 `.lt` 按 `created_at` 时间窗口进行筛选。有关每个参数的类型和可接受的值，请参阅 [API 参考](https://platform.claude.com/docs/zh-CN/api/compliance/activities/list)。

可重复参数使用数组方括号查询语法：为每个值分别传递一次 `activity_types[]=...`、`actor_ids[]=...` 或 `organization_ids[]=...`。

```bash cURL
curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/activities" \
  --data-urlencode "activity_types[]=claude_file_uploaded" \
  --data-urlencode "activity_types[]=claude_chat_created" \
  --data-urlencode "created_at.gte=2026-04-01T00:00:00Z" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY"
```

活动源会产生数百种不同的活动类型。有关 `activity_types[]` 接受的完整值列表，请参阅 API 参考中的[查询合规活动](https://platform.claude.com/docs/zh-CN/api/compliance/activities/list)。

## 分页浏览结果

活动按从新到旧的顺序返回，`created_at` 相同时按活动 ID 排序，每个响应最多返回 `limit` 条结果（默认 100，最大 5,000）。有关完整的响应架构，请参阅 [API 参考](https://platform.claude.com/docs/zh-CN/api/compliance/activities/list)。

Compliance API 根据端点系列使用两种分页方案：

| 端点系列                          | 排序顺序            | 方案   | 参数                                                |
| ----------------------------- | --------------- | ---- | ------------------------------------------------- |
| 活动                            | 从新到旧            | 游标   | `after_id`、`before_id`（以 `first_id`、`last_id` 返回） |
| 聊天和聊天消息                       | 从旧到新            | 游标   | `after_id`、`before_id`（以 `first_id`、`last_id` 返回） |
| 组织、项目、项目附件、用户、角色、角色权限、群组、群组成员 | 因端点而异           | 页面令牌 | `page`（以 `next_page` 返回）                          |
| 本地和远程会话及会话消息                  | 会话从新到旧；消息默认从旧到新 | 页面令牌 | `page`（以 `next_page` 返回）                          |

文件不分页：它们按 ID 单独检索。

分页游标和页面令牌是不透明字符串：请原样传回。它们的内部格式并不稳定，对其进行解析会在没有通知的情况下失效。每个请求中只能设置 `after_id` 或 `before_id` 之一，两种方案都会返回 `has_more`，以便您知道何时停止。会话端点（本地和远程）是例外：它们返回 `next_page` 而不返回 `has_more`，因此当 `next_page` 为 `null` 时停止。

要分页浏览活动：

* 将响应的 `last_id` 作为 `after_id` 传递，以按结果顺序前进到下一页。由于活动按从新到旧排序，下一页包含更早的条目。
* 将 `first_id` 作为 `before_id` 传递，以返回上一页。
* 当 `has_more` 为 `false` 时停止。

游标参数决定翻页方向；端点的排序顺序决定时间方向。在这里，同一个 `after_id` 参数会获取更早的活动。聊天按从旧到新排序；有关那里的游标语义，请参阅[检索和删除聊天、文件和项目](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data)。

<Note>
  **游标在重试时可以安全地重复使用。** 来自已成功返回页面的游标或页面令牌仍然有效；失败的请求（5xx、超时、网络错误）不会推进您的位置。请使用相同的游标重试相同的请求。只有在您存储了游标所指向之前的页面后，才移动到下一个游标。

  在较长的暂停期间，本地会话端点上的页面令牌是例外。在[本地会话消息端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-a-local-session-transcript)上，一次遍历的 `page` 令牌在其第一页之后 24 小时过期（一次遍历是指对各页面的一次完整浏览），因此请在该时间窗口内完成或恢复，或者不带 `page` 参数重新开始。在[本地会话列表](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)上，较旧的 `page` 令牌仍会被接受，但会根据当前的保留边界重新评估，并可能跳过某些会话，因此也请在 24 小时内完成列表遍历。
</Note>

```bash cURL
# 获取第一页（最新活动优先）并记录其末尾游标。
last_id=$(curl --fail-with-body -sS \
  "https://api.anthropic.com/v1/compliance/activities?limit=2" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" | jq -er '.last_id')

# 原样传回游标以获取下一页（更早的）数据。
curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/activities" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --data-urlencode "limit=2" \
  --data-urlencode "after_id=${last_id}"
```

生产环境中的**回填**循环通过依据 `has_more` 和 `last_id` 驱动迭代来分页浏览更早的活动：

1. 从您存储的游标开始（或省略 `after_id` 以从头开始）。
2. 使用 `after_id=<last_id>` 逐页浏览，直到 `has_more` 为 `false`。
3. 只有在您存储了最终 `last_id` 所涵盖的每一页之后，才持久化该 `last_id`。

```text
cursor = stored_cursor
loop:
  if cursor is not null:
    page = GET /v1/compliance/activities?after_id={cursor}&limit=100
  else:
    page = GET /v1/compliance/activities?limit=100
  store(page.data)
  if page.last_id is not null:
    cursor = page.last_id
  if not page.has_more: break
persist(cursor)
```

## 了解 Activity 对象

`data` 中的每个条目都是一个 Activity，具有以下顶层结构：

| 字段                  | 类型            | 描述                                                                                                                                                             |
| ------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                | string        | 活动的唯一标识符。                                                                                                                                                      |
| `created_at`        | RFC 3339 字符串  | 活动发生的时间。                                                                                                                                                       |
| `organization_id`   | string 或 null | 活动发生的组织，对于不与组织关联的事件（登录、登出、Compliance API 调用）为 `null`。                                                                                                          |
| `organization_uuid` | string 或 null | 与 `organization_id` 作用范围相同，以 UUID 表示。                                                                                                                          |
| `actor`             | Actor 联合类型    | 执行该活动的人或事物。请参阅下面的操作者表。                                                                                                                                         |
| `type`              | string        | 活动类型，例如 `claude_chat_created`。                                                                                                                                 |
| *其他字段*              | 不定            | 特定于类型的字段，例如聊天事件上的 `claude_chat_id` 或文件事件上的 `filename`。有关每种类型的字段列表，请参阅 API 参考中的[查询合规活动](https://platform.claude.com/docs/zh-CN/api/compliance/activities/list)。 |

`actor` 字段是一个可辨识联合类型。`type` 辨识符告诉您存在哪些其他字段：

| `actor.type`                 | 出现时机                                                                                                                   | 关键字段                                                                                      |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `user_actor`                 | 已登录的 claude.ai 或 Claude Console 用户执行了该操作。                                                                              | `email_address`、`user_id`、`ip_address`、`user_agent`                                       |
| `api_actor`                  | 某个请求使用客户签发的 API 密钥调用了 Claude API 或 Compliance API。对于 Compliance Access Key 和 Admin API 密钥，Compliance API 调用都会产生此操作者类型。 | `api_key_id`、`ip_address`、`user_agent`                                                    |
| `admin_api_key_actor`        | 组织管理员使用 Admin API 密钥管理用户、邀请、工作区或 API 密钥。                                                                               | `admin_api_key_id`、`ip_address`、`user_agent`                                              |
| `unauthenticated_user_actor` | 在登录完成之前发生的操作，例如 `sso_login_initiated`。                                                                                 | `unauthenticated_email_address`、`ip_address`、`user_agent`                                 |
| `anthropic_actor`            | Anthropic 对该组织执行了操作，例如通过内部工具。                                                                                          | `email_address`（始终为 `null`；为与 `user_actor` 保持结构一致而存在，因为 Anthropic 操作人员不以个人电子邮件表示）         |
| `scim_directory_sync_actor`  | 身份提供商（例如 Okta、Microsoft Entra ID 或 JumpCloud）通过 SCIM 目录同步推送了更改。                                                        | `workos_event_id`、`directory_id`、`idp_connection_type`（可为空；例如 `OktaSCIMV2`、`AzureSCIMV2`） |

`claude_*_viewed` 活动表示某个 Claude 应用加载了内容，而不是某个人查看了它。每当 Claude 应用从 Anthropic 的服务器加载聊天、文件或项目时，都会记录 `claude_chat_viewed`、`claude_file_viewed` 和 `claude_project_viewed` 等类型。重复加载不会去重。网页、桌面和移动应用在不同时刻加载内容，有时在后台加载，并且可以在不加载的情况下显示缓存副本。因此，这些活动的计数因平台而异，并且不对应于发送的消息数或查看的屏幕数。

<Note>
  **构建向前兼容的处理程序。** 透传无法识别的 `type` 和 `actor.type` 值，并忽略处理程序未预期的字段，这样当新的活动类型发布时，您的集成仍能继续工作。
</Note>

## 后续步骤

<CardGroup cols={2}>
  <Card title="API 参考" href="https://platform.claude.com/docs/zh-CN/api/compliance/activities/list">
    `GET /v1/compliance/activities` 的完整请求和响应架构，包括每个受支持的 `activity_types[]` 值。
  </Card>

  <Card title="检索和删除聊天、文件和项目" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data">
    查询和删除您在活动源中找到的活动所对应的底层内容（需要 Compliance Access Key）。
  </Card>

  <Card title="设计您的合规集成" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns">
    选择轮询或批量消费模式，并规划 SIEM 关联。
  </Card>

  <Card title="处理 Compliance API 错误" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors">
    完整的错误目录。
  </Card>
</CardGroup>
