---
title: 检索会话记录
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions
description: 列出您的用户在 Claude 应用和智能体（例如 Claude Cowork 和 Claude Code）中运行的会话，并通过 Compliance API 检索其会话记录。
---

<Note>
  本页上的端点仅对 Claude Enterprise 组织可用，且处于 beta 阶段。它们使用与[聊天、文件和项目端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data)相同的 Compliance Access Key 和 `read:compliance_user_data` 作用域；无需新的密钥、作用域、设置或客户端更新。请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。
</Note>

<Check>
  **所需作用域：** Compliance Access Key 上的 `read:compliance_user_data`。

  **前提条件：** 在组织范围内列出会话无需任何前提条件。若要将远程会话列表（云端会话）筛选到特定用户，您需要从[列出组织用户](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data#list-organization-users)获取用户 ID；本地会话列表没有用户筛选器。
</Check>

本页上的端点将您的用户在 Claude 应用和智能体（目前为 Cowork 和 Claude Code）中运行的会话记录，从您的 Claude Enterprise 组织公开给合规审查人员。每个会话是与 Claude 的一次单独对话；其 "transcript"（会话记录）是该对话中用户提示、助手响应以及工具调用和结果的序列。这些端点支持 "eDiscovery"（电子取证）导出和 "data loss prevention"（数据丢失防护），即 DLP 的执行。

Compliance API 根据会话的运行位置将其分为两个端点系列：用于在用户机器上运行的会话的本地会话端点，以及用于在 Anthropic 托管环境中于云端运行的会话的远程会话端点。两个系列均为只读，且均不对 Admin API 密钥（`sk-ant-admin01-...`）开放：使用 Admin API 密钥进行身份验证的调用会返回 [403 Forbidden](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)。

下表将每个产品及其运行位置映射到返回其会话的端点系列，以及在响应中标识它们的 `product_surface` 值。随着覆盖范围的扩大，产品会被添加到此表中。

| 产品及其运行位置                                                 | 端点系列                                          | `product_surface` |
| -------------------------------------------------------- | --------------------------------------------- | ----------------- |
| Claude Desktop 中的 Cowork，在用户机器上运行                        | 本地会话端点（`/v1/compliance/apps/sessions/local`）  | `cowork`          |
| 终端、Claude Desktop 或 IDE 扩展中的 Claude Code，在用户机器上运行        | 本地会话端点                                        | `claude_code`     |
| 在 claude.ai 网页版或移动版上启动的 Cowork 会话，在 Anthropic 托管环境中于云端运行 | 远程会话端点（`/v1/compliance/apps/sessions/remote`） | `cowork_remote`   |

本地会话的捕获与您的组织是否启用了 Compliance API 相关联，并在用户使用其 Claude Enterprise 账户登录期间适用。会话端点不会返回以下内容：

* 使用 Claude Console API 密钥进行身份验证的 Claude Code 会话，或通过第三方云平台（例如 Amazon Bedrock、Google Cloud 或 Microsoft Foundry）运行的 Claude Code 会话。
* 网页版 Claude Code。它同样在 Anthropic 托管环境中于云端运行，但它不是远程会话；远程会话端点仅返回 Cowork 会话。
* 启用了 [HIPAA 就绪](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#hipaa-readiness)的组织中的本地会话。不会捕获任何本地会话数据，因此本地会话端点不会为这些组织返回任何会话。
* [零数据保留（ZDR）](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#zero-data-retention-zdr-scope)生效的本地会话。这些会话会从列表结果中排除，且检索端点和消息端点对它们返回 404。

Anthropic 建议使用 Compliance API 来检索 Cowork 和 Claude Code 会话的内容。下表将[本地会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)和[远程会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-remote-sessions)与基于 OpenTelemetry 的替代方案进行比较，即 [Cowork 的 OpenTelemetry 日志记录](https://support.claude.com/en/articles/14477985-monitor-claude-cowork-activity-with-opentelemetry)和 [Claude Code 监控](https://code.claude.com/docs/en/monitoring-usage)。

|                      | 本地会话（在用户机器上）                                                                                                                                                     | 远程会话（在云端）                                                                                                                                                        | OpenTelemetry 日志记录                            |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| 交付方式                 | 拉取：通过 HTTPS 查询和导出                                                                                                                                                | 拉取：通过 HTTPS 查询和导出                                                                                                                                                | 推送：流式传输到您的 OTLP 收集器                           |
| 设置                   | 使用您现有的 Compliance Access Key 即可                                                                                                                                  | 使用您现有的 Compliance Access Key 即可                                                                                                                                  | 管理员配置 OTLP 端点和内容捕获设置                          |
| 基础设施                 | Anthropic 托管                                                                                                                                                     | Anthropic 托管                                                                                                                                                     | 您运行收集器和存储                                     |
| ID 前缀                | `clls_`                                                                                                                                                          | `cse_`                                                                                                                                                           | 不适用                                           |
| `product_surface` 值  | `cowork`、`claude_code`                                                                                                                                           | `cowork_remote`                                                                                                                                                  | 不适用                                           |
| 保留期                  | 默认 6 年，或在设置了有限期限时采用您组织的自定义对话保留期；由 Anthropic 持有                                                                                                                   | 6 年，由 Anthropic 持有                                                                                                                                               | 您的基础设施，您的策略                                   |
| 用户提示和助手响应            | 是                                                                                                                                                                | 是                                                                                                                                                                | 是，受内容捕获设置约束                                   |
| 工具输入                 | 默认每个输入截断为 10,000 字节；可按请求提高至约 1 MiB                                                                                                                               | 默认每个输入截断为 10,000 字节；可按请求提高至约 1 MiB                                                                                                                               | 截断的摘要                                         |
| 工具结果内容               | 默认每个文本条目截断为 10,000 字节；可按请求提高至约 1 MiB                                                                                                                             | 默认每个文本条目截断为 10,000 字节；可按请求提高至约 1 MiB                                                                                                                             | 大小和成功与否等元数据；Claude Code 还可以通过可选的、有大小上限的设置捕获内容 |
| 文件内容                 | 是，通过会话记录中的工具调用（仅文本；其他内容显示为占位符）                                                                                                                                   | 是，通过会话记录中的工具调用（仅文本；其他内容被省略）                                                                                                                                      | 文件路径；Claude Code 还可以通过可选的、有大小上限的设置捕获内容        |
| 主机和设备元数据（终端类型、工作区路径） | 否                                                                                                                                                                | 否                                                                                                                                                                | 是                                             |
| 令牌用量和成本              | 否；可通过 [Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api#get-access-to-the-claude-enterprise-analytics-api) 获取 | 否；可通过 [Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api#get-access-to-the-claude-enterprise-analytics-api) 获取 | 是                                             |

## 用户机器上的会话（本地会话）

本地会话在用户使用其 Claude Enterprise 账户登录期间于用户机器上运行：目前包括 Claude Desktop 中的 Cowork，以及终端、Claude Desktop 或 IDE 扩展中的 Claude Code。

Compliance API 通过三个端点公开本地会话：`GET /v1/compliance/apps/sessions/local` 列出会话元数据，`GET /v1/compliance/apps/sessions/local/{session_id}` 检索单个会话的元数据，`GET /v1/compliance/apps/sessions/local/{session_id}/messages` 返回单个会话的会话记录。这三个端点都需要 `read:compliance_user_data` 作用域，并且仅计入共享的 Compliance API 速率限制；它们不受适用于远程会话端点的第二个请求预算的约束。请参阅 [429 Too Many Requests](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#429-too-many-requests)。如果您的父组织无法使用本地会话，这三个端点都会返回 404，并附带消息 `Local sessions are not available.`（请参阅[未找到本地会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-session-not-found)）；当会话列表或已捕获内容暂时不可用时，它们会返回 503（请参阅[本地会话暂时不可用](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-sessions-temporarily-unavailable)）。

对于本地会话，Anthropic 会在每个对话的请求到达 Claude API 时在服务器端记录该对话；设备上不会安装任何内容，除了客户端已经发送到 Claude API 的请求之外，不会收集任何其他内容。本地会话记录显示的是 Claude 被要求做什么以及它返回了什么，而不是设备上发生了什么。文件和网络活动仅通过会话记录中的工具调用和工具结果可见，因此从未到达 API 的活动（例如，会话从未发送的本地文件）不会被捕获。

在使用[客户管理的加密密钥](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)的组织中，本地会话照常列出并可检索，但目前不会返回会话记录内容；每条消息返回时其内容会被标记为不可用（有关此类消息的标记方式，请参阅[检索本地会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-a-local-session-transcript)）。

列表端点为您的密钥可读取的每个关联组织返回会话元数据，不含会话记录内容。与远程会话列表不同，它没有组织或用户筛选器：请使用 `created_at.gte` 和 `created_at.lt` 参数在时间上限定结果范围。两者都接受带有必需 UTC 偏移量的 RFC 3339 时间戳，当两者都提供时，`created_at.lt` 必须严格晚于 `created_at.gte`，否则请求会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。第三个时间筛选器 `updated_at.gte` 按最后活动时间而非首次活动时间限定范围：它返回最后一次推理调用在给定时间或之后的会话，并可与 `created_at` 筛选器组合使用，而不改变排序或分页。如本节后文所述，可使用它来轮询自上一次遍历以来活跃的会话。新会话和消息在短暂的处理延迟后出现在结果中，通常在几分钟内；会话刚开始后立即缺失并不一定意味着未被捕获。以下请求列出自给定日期以来创建的会话。

```bash cURL
curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/apps/sessions/local" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --data-urlencode "created_at.gte=2026-07-01T00:00:00Z" \
  --data-urlencode "limit=100"
```

```json Response
{
  "data": [
    {
      "type": "compliance_local_session",
      "id": "clls_01HxKpLmNoPqRsTuVwXyZaBc",
      "organization_uuid": "9a1e0000-0000-0000-0000-000000000000",
      "workspace_id": "wrkspc_01SvYKoWVRVHoEbwESNvzYdR",
      "user": {
        "id": "user_01GpKpLmNoPqRsTuVwXyZaBc",
        "email_address": "engineer@example.com"
      },
      "product_surface": "cowork",
      "created_at": "2026-07-09T14:02:11Z",
      "updated_at": "2026-07-09T14:02:38Z"
    },
    {
      "type": "compliance_local_session",
      "id": "clls_01HyLqMnOpQrStUvWxYzAbCd",
      "organization_uuid": "9a1e0000-0000-0000-0000-000000000000",
      "workspace_id": null,
      "user": {
        "id": "user_01HqRsTuVwXyZaBcDeFgHiJk",
        "email_address": null
      },
      "product_surface": "claude_code",
      "created_at": "2026-07-08T09:15:43Z",
      "updated_at": "2026-07-08T09:52:10Z"
    }
  ],
  "next_page": "page_AAEfQx7mPdLkq9Rt2VwHbZk"
}
```

结果按 `created_at` 以逆时间顺序（最新的在前）排序，并列项按固定的服务器端顺序排列，每个响应最多返回 `limit` 个结果（默认 100，最大 500）。该端点仅使用 `page` 和 `next_page` 令牌向前分页（请参阅[分页结果](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#paginate-results)）：在下一个请求中将响应的 `next_page` 值作为 `page` 查询参数传回，并在 `next_page` 为 `null` 时停止。响应没有 `has_more` 字段。请在开始列表遍历后的 24 小时内完成遍历；较旧的列表游标仍会被接受，但会根据当前的保留边界重新评估，因此其最早保留活动即将超出保留期的会话可能会被跳过。

在每个会话对象中，`user.id` 始终被设置，并且在账户删除后仍然保留；当用户的账户已被删除或用户不再是您的密钥可读取的组织的成员时，`user.email_address` 为 `null`。当会话未与工作区关联时，`workspace_id` 为 `null`。一个本地会话对应一个客户端会话 ID：在客户端中开始新对话或清除其上下文会开始一条新的会话记录。请将 `id` 值视为不透明字符串；其格式可能会在不另行通知的情况下更改。

本地会话携带 `updated_at` 但不携带 `status`：本地会话没有服务器端生命周期状态，其可见性由保留期控制。本地会话被捕获为客户端在会话期间进行的一系列 Claude API 调用（推理调用），保留期分别适用于每个被捕获的调用。`created_at` 是会话最早保留的调用的时间戳，`updated_at` 是其最后一次调用的时间戳，均为 UTC。随着较旧的调用超出保留期，`created_at` 会相应前移，一旦会话中的每个调用都已过期，该会话将不再返回；`updated_at` 跟踪最近的调用，在此之前不受影响。由于 `created_at` 可能在多次运行之间发生变化，因此在随时间重新遍历列表时请按 `id` 去重。为了在会话增加消息时保持会话记录最新，请使用 `updated_at.gte` 筛选器进行轮询，并使连续的窗口相互重叠。在列表端点上，`updated_at` 是一个下界：对于在页面或 `created_at.lt` 窗口边界处仍然活跃的会话，它可能会暂时滞后于会话的真实最后活动时间，并且新调用只有在前文提到的短暂处理延迟之后才可查询。由于这种滞后，请将每次运行的 `updated_at.gte` 设置为比上一次运行的开始时间早几分钟，而不是恰好设置为上一次运行的时间。设置为恰好上一次时间的边界会悄无声息地永久丢弃在那一刻其最后一次调用仍在索引中的会话，因为一旦边界前移超过该调用，之后的任何运行都不会再返回它。对返回的会话按 `id` 去重，重新获取其会话记录，并对消息按 `id` 去重。检索会话或其消息始终反映确切的最新保留调用，因此作为扩大重叠范围的替代方案，也可以定期对较旧窗口执行一次对账遍历，以求双重保险。

该列表是根据会话活动元数据构建的，因此它可能包含会话记录内容未被捕获的会话，例如在您的组织开始捕获之前运行的会话（可追溯到您的保留期允许的最早时间）；此类会话的会话记录会返回每条消息，并将其内容标记为不可用（请参阅[检索本地会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-a-local-session-transcript)）。

已捕获的本地会话内容默认自捕获起存储 6 年。如果运行该会话的组织在 [claude.ai > Organization settings > Data and privacy](https://claude.ai/admin-settings/data-privacy-controls) 中设置了有限的自定义对话保留期，则改为适用该期限，无论它比默认值短还是长；当组织配置了多个自定义保留期时，适用最短的那个。对该设置的更改以两种不同的方式生效：设置一经更改，端点就会停止返回早于组织当前期限的活动；而每条已捕获的消息按其被捕获时生效的期限存储，因此之后延长期限不会恢复已经过期的内容。

若要直接获取单个会话的元数据，请将其 ID 传递给 `GET /v1/compliance/apps/sessions/local/{session_id}`。响应与列表端点返回的会话对象相同，没有信封，也没有会话记录内容。格式错误的会话 ID 会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。单一的 [404 Not Found](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#404-not-found) 涵盖响应不加区分的四种情况：会话不在您的密钥可读取的组织中（包括另一个父组织下的会话）、会话不存在、零数据保留对其生效，或其中的每个调用都已超出保留期。

`product_surface`（字符串或 `null`）标识创建会话的产品：`cowork` 表示在用户机器上的 Claude Desktop 中运行的 Cowork 会话，`claude_code` 表示 Claude Code 会话。随着覆盖范围的扩大，会出现新的值。

<Note>
  **构建向前兼容的处理程序。** 透传无法识别的 `product_surface` 值，并忽略您的处理程序不期望的字段，以便在新的产品界面发布时您的集成能够继续工作。
</Note>

### 检索本地会话记录

消息端点返回会话的会话记录，该记录由已捕获的 Claude API 调用重建而成：用户提示、助手文本、工具调用以及工具结果的文本部分，除大小截断外均按发送时的原样返回。该内容中的 URL、凭据或个人数据不会被遮蔽，因此请将会话记录视为敏感内容。会话记录会省略或替换以下内容：

* 思考块永远不会被包含。
* 请求的 "system prompt"（系统提示）永远不会被返回。一条内容为 `[system prompt content not shown]` 的标记消息代替它（通常每个会话一次；没有已捕获内容的会话不携带标记）。
* 工具定义和 MCP 服务器配置不属于会话记录的一部分。
* 图像、PDF 以及其他二进制或结构化块不会被返回。每个此类块显示为一个内容为 `[<block type> content not shown]`（例如 `[image content not shown]`）的 `text` 块，且 `truncated` 设置为 `true`。工具结果中的非文本项会被替换为一个 `[N non-text item(s) not shown]` 条目，且该工具结果块的 `truncated` 为 `true`。
* `text` 块上的引用元数据会被省略，受影响的块携带设置为 `true` 的 `truncated`。

`CLAUDE.md` 等项目指令文件显示为普通的用户角色内容。技能内容在客户端将其作为消息内容发送时出现，并且不与其他用户文本区分。有关覆盖范围摘要，请参阅 [Compliance API 常见问题](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq#data-coverage-and-retention)；有关将本地会话与远程会话和 OpenTelemetry 日志记录进行比较的表格，请参阅本页面的简介。

```bash cURL
session_id="clls_01HxKpLmNoPqRsTuVwXyZaBc"

curl --fail-with-body -sS \
  "https://api.anthropic.com/v1/compliance/apps/sessions/local/$session_id/messages" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY"
```

```json Response
{
  "session": {
    "type": "compliance_local_session",
    "id": "clls_01HxKpLmNoPqRsTuVwXyZaBc",
    "organization_uuid": "9a1e0000-0000-0000-0000-000000000000",
    "workspace_id": "wrkspc_01SvYKoWVRVHoEbwESNvzYdR",
    "user": {
      "id": "user_01GpKpLmNoPqRsTuVwXyZaBc",
      "email_address": null
    },
    "product_surface": "cowork",
    "created_at": "2026-07-09T14:02:11Z",
    "updated_at": "2026-07-09T14:02:38Z"
  },
  "data": [
    {
      "type": "compliance_local_session_message",
      "id": "clsm_01J4KpLmNoPqRsTuVwXyZaBa",
      "role": "user",
      "model": null,
      "created_at": "2026-07-09T14:02:11Z",
      "provenance": {
        "type": "synthetic_marker"
      },
      "content": [
        {
          "type": "text",
          "text": "[system prompt content not shown]",
          "truncated": true
        }
      ]
    },
    {
      "type": "compliance_local_session_message",
      "id": "clsm_01J4KpLmNoPqRsTuVwXyZaBc",
      "role": "user",
      "model": null,
      "created_at": "2026-07-09T14:02:11Z",
      "provenance": null,
      "content": [
        {
          "type": "text",
          "text": "Fix the failing test in tests/auth_test.py",
          "truncated": false
        }
      ]
    },
    {
      "type": "compliance_local_session_message",
      "id": "clsm_01J4KpLmNoPqRsTuVwXyZaBd",
      "role": "assistant",
      "model": "claude-opus-5",
      "created_at": "2026-07-09T14:02:11Z",
      "provenance": null,
      "content": [
        {
          "type": "text",
          "text": "I'll read the test file first.",
          "truncated": false
        },
        {
          "type": "tool_use",
          "id": "toolu_01AbCdEfGhIjKlMnOpQrSt",
          "name": "Read",
          "input": "{\"file_path\":\"tests/auth_test.py\"}",
          "truncated": false
        }
      ]
    },
    {
      "type": "compliance_local_session_message",
      "id": "clsm_01J4KpLmNoPqRsTuVwXyZaBe",
      "role": "user",
      "model": null,
      "created_at": "2026-07-09T14:02:38Z",
      "provenance": null,
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01AbCdEfGhIjKlMnOpQrSt",
          "name": "Read",
          "is_error": false,
          "content": [
            {
              "type": "text",
              "text": "def test_login_expiry():\n    ..."
            }
          ],
          "truncated": false
        }
      ]
    },
    {
      "type": "compliance_local_session_message",
      "id": "clsm_01J4KpLmNoPqRsTuVwXyZaBf",
      "role": "assistant",
      "model": "claude-opus-5",
      "created_at": "2026-07-09T14:02:38Z",
      "provenance": null,
      "content": [
        {
          "type": "text",
          "text": "The test was asserting on a stale expiry timestamp. I've updated it.",
          "truncated": false
        }
      ]
    }
  ],
  "next_page": null
}
```

响应在分页的 `data` 数组旁嵌入了一个 `session` 信封。此示例中的第一条记录是代替请求系统提示的标记；其 `provenance` 将在本节后面描述。在此端点上，`user.email_address` 始终为 `null`：消息端点不解析电子邮件地址，因此此处的 `null` 并不意味着用户的账户已被删除。若要将会话归属到某个电子邮件地址，请将 `user.id` 与[列表端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)或检索端点（`GET /v1/compliance/apps/sessions/local/{session_id}`）进行关联。

消息默认按最早的在前返回；传递 `order=desc` 可反转顺序。分页使用与列表端点相同的 `page`/`next_page` 方案，`limit` 默认为 100，最大为 1,000。当响应达到其大小限制时，页面可能会提前结束，因此消息数少于 `limit` 的页面并不意味着您已到达末尾；请继续分页直到 `next_page` 为 `null`。页面游标绑定到签发它们时所属的会话和排序顺序，并且一次遍历的游标在其第一页之后 24 小时过期：过期的游标会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)，告知您在不带 `page` 参数的情况下重新开始，重新开始的遍历反映当前的保留边界。为不同会话或 `order` 签发的游标同样返回 400，作为无效游标处理。

每条消息携带一个 `role`（`user` 或 `assistant`）和一个由 `text`、`tool_use` 和 `tool_result` 块组成的 `content` 数组。它还携带一个 `model`：在从 Claude API 捕获的助手轮次上，这是为该轮次提供服务的模型；在用户消息以及任何设置了 `provenance` 的助手消息上，它为 `null`，因为客户端声明的历史记录和合成标记不是由模型生成的，而对于不可用的内容，提供服务的模型是未知的。`text` 块携带 `text` 和 `truncated`。`tool_use` 块携带 `id`、`name`、`input` 和 `truncated`，其中 `input` 是 JSON 编码的字符串而不是对象。`tool_result` 块携带 `tool_use_id`、`name`、`is_error`、一个由 `text` 条目组成的 `content` 数组以及 `truncated`。MCP 工具调用和结果，以及大多数服务器工具调用和结果，都被规范化为这些相同的 `tool_use` 和 `tool_result` 形状；任何其他块类型都显示为 `[<block type> content not shown]` 占位符。消息 `id` 在该轮次被保留期间是稳定的。从同一推理调用重建的每条消息都携带该调用的时间戳，因此连续的消息通常共享一个 `created_at` 值；请保留返回的顺序，而不是按时间戳重新排序。

每条消息还携带一个 `provenance` 字段，描述其内容的捕获方式。对于由 Claude API 捕获的已验证内容（这是常见情况），`provenance` 为 `null`。否则它是一个对象，其 `type` 标记例外情况：

* `content_unavailable` 表示内容无法返回。`content` 数组为空，`provenance.reason` 说明原因。`not_captured` 表示该轮次没有可用内容；它并不证明没有存储任何记录，因为被存储端访问策略扣留的内容也以相同的原因报告（例如，在使用[客户管理的加密密钥](https://platform.claude.com/docs/zh-CN/manage-claude/cmek#disabled-or-modified)的组织中），并且在其他方面已被捕获的会话中的个别轮次可能因其他数据处理原因而不可用，并携带相同的原因。`client_aborted` 表示客户端在响应完成之前关闭了连接或取消了请求，因此该轮次的响应未被捕获；任何已经流式传输到客户端的部分输出都不包含在内，并且此原因仅适用于助手角色的轮次。`cmek_key_revoked` 保留用于在您组织的客户管理密钥不可用（例如已撤销）时使用该密钥加密的内容；目前不会返回该值，因此请为向前兼容而处理它。`retention_elapsed` 表示内容已超出保留期。`oversize` 表示单条消息超出了每条消息的大小限制；该消息仍会返回，但 `content` 数组为空。
* `client_asserted` 标记由客户端作为对话历史提供且无法与已捕获的响应匹配的助手消息；其作者身份未经验证。
* `synthetic_marker` 标记由端点本身生成的记录，例如代替系统提示的标记。当客户端在会话中途重写或压缩其对话历史时（例如，在上下文压缩之后），会话记录会在该点插入一条标记消息，并继续显示客户端发送的新内容；当您的组织设置了有限的保留期时，重写的历史本身会被扣留（第二个标记会注明这一点），并且仅显示最新的用户轮次及其后续内容。

标记消息和客户端声明的消息以一个带方括号的说明性 `text` 块开头，并标记为 `truncated: true`，例如 `[system prompt content not shown]`。请将这些记录视为存在但不可用或未经验证，而不是缺失，并容忍无法识别的 `provenance` 类型和原因。

有两个参数限制每个工具块返回的字节数：`tool_use_input_max_bytes` 和 `tool_result_max_bytes`，两者默认均为 10,000 字节。传递 `-1` 表示服务器最大值（每个字符串约 1 MiB）；`0` 会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)，超过最大值的值会被限制为最大值。被任一上限截断的字符串会在字符边界处截断，并附加一个带内后缀（例如 `…[truncated; pass tool_result_max_bytes=-1 for the server max]`），其块携带 `"truncated": true`。因此，被截断的 `tool_use` `input` 不再是有效的 JSON，所以请仅从未截断的块中解析工具输入（或提高上限并重新获取）。`text` 类型的块始终受限于相同的约 1 MiB 的服务器最大值；没有参数可以提高它，达到该限制的 `text` 块同样携带 `"truncated": true`。

会话记录内容遵循[用户机器上的会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)中描述的保留期。当会话的开头已超出保留期时，会话记录以单个 `reason` 为 `retention_elapsed` 的 `content_unavailable` 占位符开头，随后是保留的消息。当会话中的每个调用都已过期时，消息端点会返回 [404 Not Found](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#404-not-found)，与对您的密钥无法读取的组织中的会话、不存在的会话以及零数据保留生效的会话的处理方式相同。格式错误的会话 ID 会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。

## 云端会话（远程会话）

在 claude.ai 网页版或移动版上启动的 Cowork 会话在 Anthropic 托管环境中于云端运行。Compliance API 通过两个端点公开这些远程会话：`GET /v1/compliance/apps/sessions/remote` 列出会话元数据，`GET /v1/compliance/apps/sessions/remote/{session_id}/messages` 返回单个会话的会话记录。两者都需要 `read:compliance_user_data` 作用域，并且都计入共享的 Compliance API 速率限制以及这些端点特有的第二个请求预算；请参阅 [429 Too Many Requests](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#429-too-many-requests)。

列表端点默认为组织范围：省略 `organization_ids[]` 可包含您的密钥可读取的每个 claude.ai 组织，或传递最多 500 个值以缩小范围。若要改为将列表限定到特定用户，请传递 1–10 个 `user_ids[]` 值（从[列出组织用户](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data#list-organization-users)获取 ID）；该筛选器匹配会话的所属用户，因此只要设置了 `user_ids[]`，智能体拥有的会话就会被排除。使用 `created_at` 范围参数（`gte`、`gt`、`lt`、`lte`，RFC 3339 格式）在时间上限定结果范围。没有 `updated_at` 筛选器。以下请求列出自给定日期以来创建的会话。

```bash cURL
curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/apps/sessions/remote" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --data-urlencode "created_at.gte=2026-06-01T00:00:00Z" \
  --data-urlencode "limit=100"
```

```json Response
{
  "data": [
    {
      "id": "cse_01WpQrStUvXyZaBcDeFgHjK6",
      "organization_uuid": "91012d09-e48b-438e-a489-1bebfd8fa6f9",
      "user": {
        "id": "user_01XyDMpzjS89pFZXqSFUBDr6",
        "email_address": "user@example.com"
      },
      "agent_id": null,
      "started_by_user": null,
      "status": "active",
      "created_at": "2026-07-01T17:04:05Z",
      "updated_at": "2026-07-01T18:00:41Z",
      "product_surface": "cowork_remote",
      "claude_project_id": "claude_proj_01KGp4eZNug9ri4kE35RSppq"
    },
    {
      "id": "cse_01TkNpRsUvWxYzAbCdEfGhJ4",
      "organization_uuid": "91012d09-e48b-438e-a489-1bebfd8fa6f9",
      "user": null,
      "agent_id": "cagt_01MnPqRsTuVwXyZaBcDeFgH8",
      "started_by_user": {
        "id": "user_01XyDMpzjS89pFZXqSFUBDr6",
        "email_address": "user@example.com"
      },
      "status": "archived",
      "created_at": "2026-06-28T09:15:22Z",
      "updated_at": "2026-06-28T09:47:10Z",
      "product_surface": "cowork_remote",
      "claude_project_id": null
    }
  ],
  "next_page": "page_AAEfMk93cXpYdGxrZXk"
}
```

结果按 `created_at` 以逆时间顺序（最新的在前）排序，每个响应最多返回 `limit` 个结果（默认 100，最大 500）。该端点使用 `page` 和 `next_page` 令牌分页（请参阅[分页结果](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#paginate-results)）：在下一个请求中将响应的 `next_page` 值作为 `page` 查询参数传回，并在 `next_page` 为 `null` 时停止。

会话由用户或智能体拥有，绝不会同时由两者拥有。对于用户拥有的会话，`user` 携带所有者的 ID 和电子邮件地址（当用户不再是您的密钥可读取的组织的成员时，`email_address` 为 `null`），且 `agent_id` 为 `null`。对于智能体拥有的会话（例如计划任务），`user` 为 `null`，`agent_id` 携带智能体的 ID（前缀 `cagt_`），`started_by_user` 标识发起运行的人员，例如通过启动计划任务；在用户拥有的会话上，`started_by_user` 为 `null`。

`claude_project_id` 是会话所属的 claude.ai [项目](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data#retrieve-projects-and-attachments)的 ID（前缀 `claude_proj_`），当会话不在项目中时为 `null`。

`status` 为 `pending`、`active`、`paused`、`archived` 或 `failed` 之一。会话在配置期间为 `pending`；`pending` 会话尚无会话记录，在配置完成之前，消息端点对其返回 404。已删除的会话永远不会被返回。

`product_surface`（字符串或 `null`）标识创建会话的产品。该端点目前仅返回 `product_surface` 为 `cowork_remote` 的会话：在 claude.ai 网页版或移动版上启动的 Cowork 会话。

<Note>
  **构建向前兼容的处理程序。** 透传无法识别的 `status` 和 `product_surface` 值，并忽略您的处理程序不期望的字段，以便在新的状态和产品界面发布时您的集成能够继续工作。
</Note>

### 检索远程会话记录

消息端点返回会话的会话记录：用户提示、助手响应以及工具调用和结果。思考块和图像不包含在内。有关覆盖范围摘要，请参阅 [Compliance API 常见问题](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq#data-coverage-and-retention)；有关将远程会话与本地会话和 Cowork 的 OpenTelemetry 日志记录进行比较的表格，请参阅本页面的简介。

```bash cURL
session_id="cse_01WpQrStUvXyZaBcDeFgHjK6"

curl --fail-with-body -sS \
  "https://api.anthropic.com/v1/compliance/apps/sessions/remote/$session_id/messages" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY"
```

```json Response
{
  "session": {
    "id": "cse_01WpQrStUvXyZaBcDeFgHjK6",
    "organization_uuid": "91012d09-e48b-438e-a489-1bebfd8fa6f9",
    "user": {
      "id": "user_01XyDMpzjS89pFZXqSFUBDr6",
      "email_address": null
    },
    "agent_id": null,
    "started_by_user": null,
    "status": "active",
    "created_at": "2026-07-01T17:04:05Z",
    "updated_at": "2026-07-01T18:00:41Z",
    "product_surface": "cowork_remote",
    "claude_project_id": null
  },
  "data": [
    {
      "id": "csev_01HjKmNpQrStUvWxYzAbCdE2",
      "role": "user",
      "created_at": "2026-07-01T17:04:05Z",
      "content": [
        {
          "type": "text",
          "text": "Summarize the customer feedback in the attached spreadsheet.",
          "truncated": false
        }
      ],
      "sent_by_user_id": null,
      "content_unavailable": false
    },
    {
      "id": "csev_01BcDeFgHjKmNpQrStUvWxY4",
      "role": "assistant",
      "created_at": "2026-07-01T17:04:06Z",
      "content": [
        {
          "type": "text",
          "text": "I'll start by reading the spreadsheet...",
          "truncated": false
        }
      ],
      "sent_by_user_id": null,
      "content_unavailable": false
    }
  ],
  "next_page": null
}
```

响应在分页的 `data` 数组旁嵌入了一个 `session` 信封。在此端点上，信封的 `user.email_address`、`started_by_user` 和 `claude_project_id` 始终设置为 `null`；请改为从列表端点获取这些值。

消息默认按最早的在前返回；传递 `order=desc` 可反转顺序。分页使用与列表端点相同的 `page`/`next_page` 方案，`limit` 默认为 100，最大为 1,000。当响应达到其大小限制时，页面可能会提前结束，因此消息数少于 `limit` 的页面并不意味着您已到达末尾；请继续分页直到 `next_page` 为 `null`。

每条消息携带一个 `role`（`user` 或 `assistant`）和一个由 `text`、`tool_use` 和 `tool_result` 块组成的 `content` 数组。消息的 `created_at` 值是提交时间戳：连续的消息可能共享一个时间戳或略有倒置，因此请保留返回的顺序，而不是按 `created_at` 重新排序。在智能体拥有的会话上，当某条用户消息可归属时，`sent_by_user_id` 记录发送该消息的用户；否则为 `null`，包括所有助手消息。当消息的内容完全无法返回时（例如，超出大小限制），该消息携带设置为 `true` 的 `content_unavailable`。

有两个参数限制每个工具块返回的字节数：`tool_use_input_max_bytes` 和 `tool_result_max_bytes`，两者默认均为 10,000 字节。传递 `-1` 表示服务器最大值（每个字符串约 1 MiB）；`0` 会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。被任一上限截断的块携带 `"truncated": true`，被截断的 `tool_use` 输入不再是有效的 JSON，所以请仅从未截断的块中解析工具输入（或提高上限并重新获取）。

对于 `pending` 会话、不存在或已被删除的会话，以及您的密钥无法读取的组织中的会话，消息端点会返回 [404 Not Found](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#404-not-found)。

## 保留与删除

会话端点为只读；本地和远程会话无法通过 Compliance API 删除。本地会话记录默认保留 6 年，或在设置了有限期限时采用您组织的自定义对话保留期，如[用户机器上的会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)中所述。远程会话记录保留 6 年。有关这些期限与 Anthropic 其他保留安排的关系，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="检索和删除聊天、文件和项目" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data">
    使用相同的 Compliance Access Key 访问 claude.ai 聊天内容、文件附件和项目。
  </Card>

  <Card title="Compliance API 常见问题" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq#data-coverage-and-retention">
    逐字段总结会话记录包含的内容，以及其他常见问题。
  </Card>

  <Card title="处理 Compliance API 错误" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors">
    逐字的错误负载以及每个错误的修复方法。
  </Card>

  <Card title="API 参考" href="https://platform.claude.com/docs/zh-CN/api/compliance/apps">
    Compliance API 的端点路径、参数和响应模式。
  </Card>
</CardGroup>
