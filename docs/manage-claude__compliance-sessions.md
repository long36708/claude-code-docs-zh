---
title: 检索会话对话记录
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions
description: 通过 Compliance API 列出您的用户在 Claude 应用和智能体（例如 Claude Cowork 和 Claude Code）中运行的会话，并检索其对话记录。
---

<Note>
  本页上的端点仅适用于 Claude Enterprise 组织。本地和远程会话端点对 Cowork、Claude Code 和 Claude for Microsoft 365 会话的支持已稳定；对 Claude Science 和 Claude in Chrome 会话的支持处于 beta 阶段。这些端点使用与[聊天、文件和项目端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data)相同的 Compliance Access Key 和 `read:compliance_user_data` 范围；无需新的密钥、范围、设置或客户端更新。请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。
</Note>

<Check>
  **所需范围：** Compliance Access Key 上的 `read:compliance_user_data`。

  **前提条件：** 在整个组织范围内列出会话无需任何前提条件。要将远程会话列表（云端会话）筛选到特定用户，您需要从[列出组织用户](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data#list-organization-users)获取用户 ID；本地会话列表没有用户筛选器。
</Check>

本页面上的端点向合规审查人员公开您的 Claude Enterprise 组织中用户在 Claude 应用和智能体（目前包括：Cowork、Claude Code、Claude Science、Claude for Microsoft 365 和 Claude in Chrome）中运行的会话的"transcript"（对话记录）。每个会话是与 Claude 的一次对话；其对话记录是该对话中用户提示、助手响应以及工具调用和结果的序列。这些端点支持"electronic discovery"（电子取证），即 eDiscovery 导出，以及"data loss prevention"（数据防泄漏），即 DLP 策略执行。

Compliance API 根据会话的运行位置将其分为两个端点系列：用于用户机器上会话的本地会话端点，以及用于在 Anthropic 托管环境中于云端运行的会话的远程会话端点。两个系列都是只读的，且都不对 Admin API 密钥（`sk-ant-admin01-...`）开放：使用 Admin API 密钥进行身份验证的调用会返回 [403 Forbidden](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)。

下表将每个产品及其运行位置映射到返回其会话的端点系列，以及在响应中标识这些会话的 `product_surface` 值。随着覆盖范围的扩大，会有更多产品添加到此表中。

| 产品及其运行位置                                                                                                 | 端点系列                                          | `product_surface`                                                                                                        |
| -------------------------------------------------------------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Claude Desktop 中的 Cowork，在用户机器上运行                                                                        | 本地会话端点（`/v1/compliance/apps/sessions/local`）  | `cowork`                                                                                                                 |
| 终端、Claude Desktop 或 IDE 扩展中的 Claude Code，在用户机器上运行                                                        | 本地会话端点                                        | `claude_code`                                                                                                            |
| Claude Science 桌面应用，在用户机器上运行                                                                             | 本地会话端点                                        | `claude_science`                                                                                                         |
| Claude for Microsoft 365（适用于 Excel、PowerPoint、Word 和 Outlook 的 Claude 加载项），在 Microsoft 365 桌面或 Web 应用中运行 | 本地会话端点                                        | `office_agents/excel`、`office_agents/powerpoint`、`office_agents/word` 或 `office_agents/outlook`（未识别应用时为 `office_agents`） |
| Claude in Chrome（浏览器扩展的内置聊天），在用户机器上运行                                                                    | 本地会话端点                                        | `claude_in_chrome`                                                                                                       |
| 在 claude.ai 网页版或移动端启动的 Cowork 会话，在 Anthropic 托管环境中于云端运行                                                  | 远程会话端点（`/v1/compliance/apps/sessions/remote`） | `cowork_remote`                                                                                                          |

本地会话的捕获取决于您的组织是否启用了 Compliance API，并在用户使用其 Claude Enterprise 账户登录时生效。会话端点不返回以下内容：

* 使用 Claude Console API 密钥进行身份验证的 Claude Code 会话，或通过第三方云平台（例如 Amazon Bedrock、Google Cloud 或 Microsoft Foundry）运行的 Claude Code 会话。
* [Claude Code 云端会话](https://code.claude.com/docs/zh-CN/claude-code-on-the-web)，这些会话在云基础设施上运行，而不是在用户机器上运行。尽管两者都在云端运行，但这些云端会话并不是远程会话；远程会话端点仅返回 Cowork 会话。
* 在启用了 [HIPAA 就绪](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#hipaa-readiness)的组织中的本地会话。这些组织不会捕获任何本地会话数据，因此本地会话端点不会为这些组织返回任何会话。
* 适用["zero data retention"（零数据保留），即 ZDR](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#zero-data-retention-zdr-scope) 的本地会话。这些会话会从列表结果中排除，检索端点和消息端点会对其返回 404。

Anthropic 建议使用 Compliance API 检索会话内容。下表将[本地会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)和[远程会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-remote-sessions)与适用于 Cowork 和 Claude Code 的基于 OpenTelemetry 的替代方案（[Cowork 的 OpenTelemetry 日志记录](https://support.claude.com/en/articles/14477985-monitor-claude-cowork-activity-with-opentelemetry)和 [Claude Code 监控](https://code.claude.com/docs/zh-CN/monitoring-usage)）进行了比较。

|                      | 本地会话（在用户机器上）                                                                                                                                                     | 远程会话（在云端）                                                                                                                                                        | OpenTelemetry 日志记录                              |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| 交付方式                 | 拉取：通过 HTTPS 查询和导出                                                                                                                                                | 拉取：通过 HTTPS 查询和导出                                                                                                                                                | 推送：以流式传输方式发送到您的 OTLP 收集器                        |
| 设置                   | 使用您现有的 Compliance Access Key 即可                                                                                                                                  | 使用您现有的 Compliance Access Key 即可                                                                                                                                  | 管理员配置 OTLP 端点和内容捕获设置                            |
| 基础设施                 | 由 Anthropic 托管                                                                                                                                                   | 由 Anthropic 托管                                                                                                                                                   | 由您运行收集器和存储                                      |
| ID 前缀                | `clls_`                                                                                                                                                          | `cse_`                                                                                                                                                           | 不适用                                             |
| `product_surface` 值  | `cowork`、`claude_code`、`claude_science`、`claude_in_chrome`，以及以 `office_agents` 开头的值                                                                              | `cowork_remote`                                                                                                                                                  | 不适用                                             |
| 保留期                  | 默认 6 年；如果您的组织设置了有限的自定义对话保留期，则采用该保留期；由 Anthropic 保存                                                                                                               | 6 年，除非用户提前删除会话；由 Anthropic 保存                                                                                                                                    | 您的基础设施，您的策略                                     |
| 用户提示和助手响应            | 是                                                                                                                                                                | 是                                                                                                                                                                | 是，取决于内容捕获设置                                     |
| 工具输入                 | 默认每个输入截断为 10,000 字节；可按请求提高至约 1 MiB                                                                                                                               | 默认每个输入截断为 10,000 字节；可按请求提高至约 1 MiB                                                                                                                               | 截断的摘要                                           |
| 工具结果内容               | 默认每个文本条目截断为 10,000 字节；可按请求提高至约 1 MiB                                                                                                                             | 默认每个文本条目截断为 10,000 字节；可按请求提高至约 1 MiB                                                                                                                             | 大小和成功状态等元数据；Claude Code 还可以通过一个可选的、有大小上限的设置捕获内容 |
| 文件内容                 | 是，通过对话记录中的工具调用（仅文本；其他内容显示为占位符）                                                                                                                                   | 是，通过对话记录中的工具调用（仅文本；其他内容被省略）                                                                                                                                      | 文件路径；Claude Code 还可以通过一个可选的、有大小上限的设置捕获内容        |
| 主机和设备元数据（终端类型、工作区路径） | 否                                                                                                                                                                | 否                                                                                                                                                                | 是                                               |
| 令牌使用量和成本             | 否；可通过 [Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api#get-access-to-the-claude-enterprise-analytics-api) 获取 | 否；可通过 [Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api#get-access-to-the-claude-enterprise-analytics-api) 获取 | 是                                               |

## 用户机器上的会话（本地会话）

本地会话在用户使用其 Claude Enterprise 账户登录时在其机器上运行：目前包括 Claude Desktop 中的 Cowork、Claude Code（在终端、Claude Desktop 或 IDE 扩展中）、Claude Science 桌面应用、Claude for Microsoft 365（在 Excel、PowerPoint、Word 和 Outlook 中）以及 Claude in Chrome 浏览器扩展。

Compliance API 通过三个端点公开本地会话：`GET /v1/compliance/apps/sessions/local` 列出会话元数据，`GET /v1/compliance/apps/sessions/local/{session_id}` 检索单个会话的元数据，`GET /v1/compliance/apps/sessions/local/{session_id}/messages` 返回单个会话的对话记录。这三个端点都需要 `read:compliance_user_data` 范围，并且仅计入共享的 Compliance API "rate limit"（速率限制）；它们不受适用于远程会话端点的第二个请求配额的约束。请参阅 [429 Too Many Requests](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#429-too-many-requests)。如果您的父组织无法使用本地会话，这三个端点都会返回 404，并附带消息 `Local sessions are not available.`（请参阅[未找到本地会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-session-not-found)）；当会话列表或捕获的内容暂时不可用时，它们会返回 503（请参阅[本地会话暂时不可用](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-sessions-temporarily-unavailable)）。

对于本地会话，Anthropic 会在每个对话的请求到达 Claude API 时在服务器端记录该对话；设备上不会安装任何内容，除了客户端本来就会发送到 Claude API 的请求之外，也不会收集任何其他内容。本地会话对话记录显示的是 Claude 被要求做什么以及它返回了什么，而不是设备上发生了什么。文件和网络活动只能通过对话记录中的工具调用和工具结果看到，因此从未到达 API 的活动（例如，会话从未发送的本地文件）不会被捕获。

在使用[客户管理的加密密钥](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)的组织中，本地会话对话记录使用您的客户管理密钥加密，并照常返回。当该密钥无法使用时（例如，因为您禁用或撤销了它，或者因为无法访问它），消息端点会针对受影响的页面返回 [503 Service Unavailable](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-sessions-temporarily-unavailable)，而不是对话记录内容。这些消息永远不会被报告为 `not_captured`（请参阅[检索本地会话对话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-a-local-session-transcript)）。列出会话和检索会话元数据不受影响。

列表端点会为您的密钥可以读取的每个关联组织返回会话元数据，不包含对话记录内容。与远程会话列表不同，它没有组织或用户筛选器：请使用 `created_at.gte` 和 `created_at.lt` 参数在时间上限定结果。两者都接受带有必需 UTC 偏移量的 RFC 3339 时间戳；当同时提供两者时，`created_at.lt` 必须严格晚于 `created_at.gte`，否则请求会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。第三个时间筛选器 `updated_at.gte` 按最后活动时间而非首次活动时间进行限定：它返回最后一次推理调用时间等于或晚于给定时间的会话，并且可以与 `created_at` 筛选器组合使用，而不会改变排序或分页。您可以使用它来轮询自上一轮以来处于活动状态的会话，如本节后面所述。新会话和消息会在短暂的处理延迟（通常在几分钟内）后出现在结果中；会话在启动后立即缺失并不一定意味着未被捕获。以下请求列出自给定日期以来创建的会话。

```bash cURL
curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/apps/sessions/local" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --header "anthropic-version: 2023-06-01" \
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

结果按 `created_at` 以逆时间顺序（最新的在前）排序，相同值按固定的服务器端顺序排列，每个响应最多返回 `limit` 个结果（默认 100，最大 500）。该端点仅支持使用 `page` 和 `next_page` 令牌向前分页（请参阅[对结果进行分页](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#paginate-results)）：在下一个请求中将响应的 `next_page` 值作为 `page` 查询参数传回，并在 `next_page` 为 `null` 时停止。响应中没有 `has_more` 字段。请在开始列表遍历后的 24 小时内完成遍历；较旧的列表游标仍会被接受，但会根据当前的保留边界重新评估，因此最早保留活动即将超出保留期的会话可能会被跳过。

在每个会话对象中，`user.id` 始终会被设置，并且在账户删除后仍然保留；当用户的账户已被删除或用户不再是您的密钥可以读取的组织的成员时，`user.email_address` 为 `null`。当会话未与工作区关联时，`workspace_id` 为 `null`。一个本地会话对应一个客户端会话 ID：在客户端中开始新对话或清除其上下文，都会开始一个新的会话条目。对于 Claude Science，列表还可能包含该应用自身后台工作的单独会话（例如，为对话命名；在较新的应用版本中还包括其审阅者和委派轨道），而在较旧的应用版本中，部分后台工作会作为额外消息出现在对话自身的记录中。跨越某些应用更新而持续进行的 Claude Science 对话会显示为两个会话。这些行为均属预期。请将 `id` 值视为不透明字符串；其格式可能会在不另行通知的情况下更改。

对于 Claude for Microsoft 365，在加载项中删除对话仅在客户端上生效，因此不会反映在 API 中：本地会话没有 `deleted_at` 字段，会话会一直保留在列表中，直到因保留期到期而被移除。

本地会话带有 `updated_at`，但没有 `status`：本地会话没有服务器端生命周期状态，其可见性由保留期决定。本地会话被捕获为客户端在会话期间发出的一系列 Claude API 调用（推理调用），保留期分别应用于每个捕获的调用。`created_at` 是会话中最早保留的调用的时间戳，`updated_at` 是其最后一次调用的时间戳，两者均为 UTC。随着较早的调用超过保留期，`created_at` 会相应推后；一旦会话中的每个调用都已过期，该会话将不再被返回；`updated_at` 跟踪最近的调用，在此之前不受影响。由于 `created_at` 在不同运行之间可能会变化，因此当您随时间重新遍历列表时，请按 `id` 去重。为了在会话新增消息时保持对话记录最新，请使用 `updated_at.gte` 筛选器进行轮询，并使连续的时间窗口相互重叠。在列表端点上，`updated_at` 是一个下限：对于在页面或 `created_at.lt` 窗口边界处仍处于活动状态的会话，它可能会暂时落后于会话真实的最后活动时间，并且新调用只有在前面提到的短暂处理延迟之后才能被查询到。由于存在这种滞后，请将每次运行的 `updated_at.gte` 设置为比上一次运行的开始时间早几分钟，而不是恰好设置为上一次运行的时间。如果将边界恰好设置为上一次的时间，那么最后一次调用在那一刻仍在建立索引的会话会被静默且永久地遗漏，因为一旦边界越过该调用，之后的任何运行都不会再返回它。请按 `id` 对返回的会话去重，重新获取其对话记录，并按 `id` 对消息去重。检索会话或其消息始终反映确切的最新保留调用，因此对较早的时间窗口定期进行对账，是比扩大重叠范围更彻底的替代方案。

该列表基于会话活动元数据构建，因此可能包含对话记录内容未被捕获的会话，例如在您的组织开始捕获之前运行的会话（最早可追溯到您的保留期所允许的范围）；此类会话的对话记录会返回每条消息，并将其内容标记为不可用（请参阅[检索本地会话对话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-a-local-session-transcript)）。

捕获的本地会话内容默认自捕获之日起存储 6 年。如果运行该会话的组织在 [claude.ai > Organization settings > Data and privacy](https://claude.ai/admin-settings/data-privacy-controls) 中设置了有限的自定义对话保留期，则改为适用该期限，无论它比默认值短还是长；当组织配置了多个自定义保留期时，适用最短的那个。对该设置的更改会以两种不同的方式生效：一旦设置更改，端点就会立即停止返回早于组织当前期限的活动；而每条捕获的消息则按其被捕获时生效的期限进行存储，因此之后延长期限并不会恢复已经过期的内容。

要直接获取单个会话的元数据，请将其 ID 传递给 `GET /v1/compliance/apps/sessions/local/{session_id}`。响应与列表端点返回的会话对象相同，没有外层封装，也不包含对话记录内容。格式错误的会话 ID 会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。同一个 [404 Not Found](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#404-not-found) 涵盖四种情况，响应不会对其加以区分：会话不在您的密钥可以读取的组织中（包括其他父组织下的会话）、会话不存在、会话适用零数据保留，或者会话中的每个调用都已超过保留期。

`product_surface`（字符串或 `null`）标识创建会话的产品：`cowork`（用户机器上 Claude Desktop 中的 Cowork）、`claude_code`（Claude Code）、`claude_science`（Claude Science）、`claude_in_chrome`（Claude in Chrome 浏览器扩展的内置聊天），或 `office_agents/excel`、`office_agents/powerpoint`、`office_agents/word` 和 `office_agents/outlook` 之一（Claude for Microsoft 365，按应用区分；未识别应用时仅为 `office_agents`）。随着覆盖范围的扩大，会出现新的值。

<Note>
  **构建向前兼容的处理程序。** 请透传无法识别的 `product_surface` 值，并忽略处理程序未预期的字段，以便在新的产品界面发布时您的集成仍能正常工作。
</Note>

### 检索本地会话对话记录

消息端点返回会话的对话记录，该记录根据捕获的 Claude API 调用重建：包括用户提示、助手文本、工具调用以及工具结果的文本部分，除大小截断外，均按发送时的原样返回。该内容中的 URL、凭据或个人数据不会被屏蔽，因此请将对话记录视为敏感信息。对话记录会省略或替换以下内容：

* 思考块永远不会被包含。
* 请求的系统提示永远不会被返回。一条内容为 `[system prompt content not shown]` 的标记消息会代替它（通常每个会话一次；没有捕获内容的会话不带此标记）。
* 工具定义和 MCP 服务器配置不属于对话记录的一部分。
* 图像、PDF 以及其他二进制或结构化块不会被返回。每个此类块都显示为一个内容为 `[<block type> content not shown]` 的 `text` 块（例如 `[image content not shown]`），并且 `truncated` 设置为 `true`。工具结果中的非文本项，例如[网页搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)结果或[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)的输出，会被替换为一个 `[N non-text item(s) not shown]` 条目，并且该工具结果块的 `truncated` 为 `true`。对应的工具调用（其 `input` 中包含搜索查询或代码）仍会被返回。
* `text` 块上的引用元数据（例如基于网络搜索结果的回答上的来源引用）会被省略。文本本身会被返回，并且该块的 `truncated` 设置为 `true`。

诸如 `CLAUDE.md` 之类的项目指令文件显示为普通的用户角色内容。当客户端将技能内容作为消息内容发送时，技能内容会出现在对话记录中，并且不会与其他用户文本区分开。有关覆盖范围摘要，请参阅 [Compliance API 常见问题](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq#data-coverage-and-retention)；有关将本地会话与远程会话和 OpenTelemetry 日志记录进行比较的表格，请参阅本页面的简介部分。

```bash cURL
session_id="clls_01HxKpLmNoPqRsTuVwXyZaBc"

curl --fail-with-body -sS \
  "https://api.anthropic.com/v1/compliance/apps/sessions/local/$session_id/messages" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --header "anthropic-version: 2023-06-01"
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
      "model": "claude-opus-5-5",
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
      "model": "claude-opus-5-5",
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

响应在分页的 `data` 数组旁嵌入了一个 `session` 封装对象。此示例中的第一条记录是代替请求系统提示的标记；其 `provenance` 将在本节后面介绍。在此端点上，`user.email_address` 始终为 `null`：消息端点不解析电子邮件地址，因此此处的 `null` 并不表示用户的账户已被删除。要将会话归属到某个电子邮件地址，请将 `user.id` 与[列表端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)或检索端点（`GET /v1/compliance/apps/sessions/local/{session_id}`）的结果进行关联。

默认情况下，消息按从旧到新的顺序返回；传递 `order=desc` 可反转顺序。分页使用与列表端点相同的 `page`/`next_page` 方案，`limit` 默认为 100，最大为 1,000。当响应达到其大小限制时，页面可能会提前结束，因此消息数少于 `limit` 的页面并不意味着您已到达末尾；请继续分页，直到 `next_page` 为 `null`。页面游标绑定到签发时所对应的会话和排序顺序，并且一次遍历的游标会在其第一页之后 24 小时过期：过期的游标会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)，提示您在不带 `page` 参数的情况下重新开始，重新开始的遍历会反映当前的保留边界。为其他会话或其他 `order` 签发的游标也会作为无效游标返回 400。

每条消息都带有一个 `role`（`user` 或 `assistant`）和一个由 `text`、`tool_use` 和 `tool_result` 块组成的 `content` 数组。它还带有一个 `model`：对于从 Claude API 捕获的助手轮次，这是处理该轮次的模型；对于用户消息以及任何设置了 `provenance` 的助手消息，它为 `null`，因为客户端声明的历史记录和合成标记并非由模型生成，而对于不可用的内容，处理它的模型是未知的。`text` 块带有 `text` 和 `truncated`。`tool_use` 块带有 `id`、`name`、`input` 和 `truncated`，其中 `input` 是 JSON 编码的字符串，而不是对象。`tool_result` 块带有 `tool_use_id`、`name`、`is_error`、一个由 `text` 条目组成的 `content` 数组以及 `truncated`。MCP 工具调用和结果，以及大多数服务器工具调用和结果，都会被规范化为相同的 `tool_use` 和 `tool_result` 结构；任何其他块类型都显示为 `[<block type> content not shown]` 占位符。在轮次被保留期间，消息 `id` 保持稳定。从同一推理调用重建的每条消息都带有该调用的时间戳，因此连续的消息通常共享相同的 `created_at` 值；请保留返回的顺序，而不要按时间戳重新排序。

每条消息还带有一个 `provenance` 字段，描述其内容是如何被捕获的。对于由 Claude API 捕获的已验证内容（这是常见情况），`provenance` 为 `null`。否则，它是一个对象，其 `type` 标记了例外情况：

* `content_unavailable` 表示内容无法返回。`content` 数组为空，`provenance.reason` 说明原因。`not_captured` 表示该轮次没有可用内容。它并不能证明没有存储任何记录：Anthropic 的数据处理策略不向 Compliance API 提供的内容也会以相同的原因报告，在其他部分已被捕获的会话中因此类原因而不可用的个别轮次也是如此。不可用的客户管理密钥是唯一的例外，它会改为返回 [503 Service Unavailable](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-sessions-temporarily-unavailable)。`client_aborted` 表示客户端在响应完成之前关闭了连接或取消了请求，因此该轮次的响应未被捕获；已经流式传输到客户端的任何部分输出都不包括在内，并且此原因仅适用于助手角色的轮次。`cmek_key_revoked` 保留用于标记使用您组织的客户管理密钥加密、且该密钥不可用（例如已被撤销）的内容。目前不会返回此原因，因为不可用的密钥会改为产生 503，但为了向前兼容，请对其进行处理。`retention_elapsed` 表示内容已超过保留期。`oversize` 表示单条消息超出了每条消息的大小上限；该消息仍会被返回，但 `content` 数组为空。
* `client_asserted` 标记由客户端作为对话历史提供、且无法与捕获的响应匹配的助手消息；其作者身份未经验证。
* `synthetic_marker` 标记由端点本身生成的记录，例如代替系统提示的标记。当客户端在会话中途重写或压缩其对话历史时（例如，在上下文压缩之后），对话记录会在该位置插入一条标记消息，并继续显示客户端发送的新内容。当您的组织设置了有限的保留期且该新内容包含助手消息时，对话记录会隐去新内容中直至并包括其最后一条助手消息的部分（由第二条标记说明这一点），并且仅显示该位置之后的用户消息，随后是会话的其余部分。

标记消息和客户端声明的消息以一个带方括号的说明性 `text` 块开头，该块标记为 `truncated: true`，例如 `[system prompt content not shown]`。请将这些记录视为存在但不可用或未经验证，而不是缺失，并容忍无法识别的 `provenance` 类型和原因。

有两个参数限制每个工具块返回的字节数：`tool_use_input_max_bytes` 和 `tool_result_max_bytes`，两者默认均为 10,000 字节。传递 `-1` 可使用服务器最大值（每个字符串约 1 MiB）；`0` 会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)，超过最大值的值会被限制为最大值。被任一上限截断的字符串会在字符边界处截断，并附加一个带内后缀（例如 `…[truncated; pass tool_result_max_bytes=-1 for the server max]`），其所在块带有 `"truncated": true`。因此，被截断的 `tool_use` `input` 不再是有效的 JSON，所以请仅从未截断的块中解析工具输入（或提高上限并重新获取）。`text` 类型的块始终以相同的服务器最大值（约 1 MiB）为上限；没有参数可以提高该上限，达到该上限的 `text` 块也带有 `"truncated": true`。

Claude Science 通过其 `repl` 工具运行的代码来调用连接器（MCP 服务器），而不是作为单独命名的工具调用，因此 Claude Science 对话记录中没有以连接器命名的块。每个连接器调用都出现在 `repl` `tool_use` 块的 `input` 中的代码里（例如，一个 `host.mcp("<server>", "<tool>", ...)` 调用），而连接器输出仅在该代码将其打印出来时才会出现在对应的 `tool_result` 中。Cowork 和 Claude Code 会话则不同：它们以各自的 `mcp__<server>__<tool>` 名称调用每个连接器工具，该名称即为 `tool_use` 块的 `name`。要监控 Claude Science 会话中的连接器使用情况，请解析 `input` 字符串并匹配其中包含的代码，而不是匹配工具名称。对于这些会话，请传递 `tool_use_input_max_bytes=-1`，以便较长的代码输入以服务器最大值为上限返回，而不会在连接器调用出现之前就在 10,000 字节的默认值处被截断。

对话记录内容遵循[用户机器上的会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)中所述的保留期。当会话的开头部分已超过保留期时，对话记录会以单个 `reason` 为 `retention_elapsed` 的 `content_unavailable` 占位符开头，随后是保留的消息。当会话中的每个调用都已过期时，消息端点会返回 [404 Not Found](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#404-not-found)，与对您的密钥无法读取的组织中的会话、不存在的会话以及适用零数据保留的会话的处理方式相同。格式错误的会话 ID 会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。

## 云端会话（远程会话）

在 claude.ai 网页版或移动端启动的 Cowork 会话在云端由 Anthropic 管理的环境中运行。Compliance API 通过两个端点公开这些远程会话：`GET /v1/compliance/apps/sessions/remote` 列出会话元数据，`GET /v1/compliance/apps/sessions/remote/{session_id}/messages` 返回单个会话的对话记录。两者都需要 `read:compliance_user_data` 范围，并且都会计入共享的 Compliance API "rate limit"（速率限制），同时还会计入这些端点专用的第二个请求配额；请参阅 [429 Too Many Requests](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#429-too-many-requests)。

列表端点默认使用组织范围：省略 `organization_ids[]` 即可包含您的密钥可读取的所有 claude.ai 组织，或传入最多 500 个值以缩小范围。若要改为将列表限定到特定用户，请传入 1–10 个 `user_ids[]` 值（可从[列出组织用户](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data#list-organization-users)获取这些 ID）；该筛选器匹配会话的所属用户，因此只要设置了 `user_ids[]`，由代理拥有的会话就会被排除。使用 `created_at` 范围参数（`gte`、`gt`、`lt`、`lte`，采用 RFC 3339 格式）在时间上限定结果。不存在 `updated_at` 筛选器。以下请求列出自给定日期以来创建的会话。

```bash cURL
curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/apps/sessions/remote" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --header "anthropic-version: 2023-06-01" \
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

结果按 `created_at` 以时间倒序（最新的在前）排列，每个响应最多返回 `limit` 条结果（默认 100，最大 500）。该端点使用 `page` 和 `next_page` 令牌进行 "pagination"（分页）（请参阅[对结果进行分页](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#paginate-results)）：在下一个请求中将响应的 `next_page` 值作为 `page` 查询参数传回，当 `next_page` 为 `null` 时停止。

会话由用户或代理之一拥有，绝不会同时由两者拥有。对于用户拥有的会话，`user` 包含所有者的 ID 和电子邮件地址（当该用户不再是您的密钥可读取的组织的成员时，`email_address` 为 `null`），且 `agent_id` 为 `null`。对于代理拥有的会话（例如计划任务），`user` 为 `null`，`agent_id` 包含代理的 ID（前缀为 `cagt_`），而 `started_by_user` 标识发起该运行的人员，例如通过启动计划任务发起；在用户拥有的会话中，`started_by_user` 为 `null`。

`claude_project_id` 是会话所属的 claude.ai [项目](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data#retrieve-projects-and-attachments)的 ID（前缀为 `claude_proj_`），当会话不属于任何项目时为 `null`。

`status` 为 `pending`、`active`、`paused`、`archived` 或 `failed` 之一。会话在预配期间处于 `pending` 状态；`pending` 会话尚无对话记录，在预配完成之前，消息端点会对其返回 404。已删除的会话永远不会被返回。

`product_surface`（字符串或 `null`）标识创建该会话的产品。该端点目前仅返回 `product_surface` 为 `cowork_remote` 的会话：即在 claude.ai 网页版或移动端启动的 Cowork 会话。

<Note>
  **构建向前兼容的处理程序。** 对于无法识别的 `status` 和 `product_surface` 值，请直接透传，并忽略处理程序未预期的字段，以便在新的状态和产品界面发布后，您的集成仍能正常工作。
</Note>

### 检索远程会话对话记录

消息端点返回会话的对话记录：用户提示、助手响应，以及工具调用和结果。不包含思考块和图像。有关覆盖范围摘要，请参阅 [Compliance API 常见问题](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq#data-coverage-and-retention)；有关远程会话与本地会话以及 Cowork 的 OpenTelemetry 日志记录的对比表，请参阅本页的简介部分。

```bash cURL
session_id="cse_01WpQrStUvXyZaBcDeFgHjK6"

curl --fail-with-body -sS \
  "https://api.anthropic.com/v1/compliance/apps/sessions/remote/$session_id/messages" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --header "anthropic-version: 2023-06-01"
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

响应在分页的 `data` 数组旁嵌入了一个 `session` 信封。在此端点上，该信封中的 `user.email_address`、`started_by_user` 和 `claude_project_id` 始终设置为 `null`；请改为从列表端点获取这些值。

消息默认按从旧到新的顺序返回；传入 `order=desc` 可反转顺序。分页使用与列表端点相同的 `page`/`next_page` 方案，`limit` 默认为 100，最大为 1,000。当响应达到其大小限制时，页面可能会提前结束，因此消息数少于 `limit` 的页面并不意味着您已到达末尾；请持续分页，直到 `next_page` 为 `null`。

每条消息都带有一个 `role`（`user` 或 `assistant`）以及一个由 `text`、`tool_use` 和 `tool_result` 块组成的 `content` 数组。消息的 `created_at` 值是提交时间戳：连续的消息可能共享同一时间戳或出现轻微的顺序颠倒，因此请保留返回的顺序，而不要按 `created_at` 重新排序。在代理拥有的会话中，当可以确定归属时，`sent_by_user_id` 会记录发送某条用户消息的用户；否则为 `null`，所有助手消息上也均为 `null`。当消息的内容完全无法返回时（例如超出大小限制），该消息会带有设置为 `true` 的 `content_unavailable`。

有两个参数用于限制每个工具块返回的字节数：`tool_use_input_max_bytes` 和 `tool_result_max_bytes`，两者默认均为 10,000 字节。传入 `-1` 表示使用服务器最大值（每个字符串约 1 MiB）；传入 `0` 会返回 [400 Bad Request](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#400-bad-request)。被任一上限截断的块会带有 `"truncated": true`，而被截断的 `tool_use` 输入不再是有效的 JSON，因此请仅从未截断的块中解析工具输入（或提高上限后重新获取）。

对于 `pending` 会话、不存在或已删除的会话，以及位于您的密钥无法读取的组织中的会话，消息端点会返回 [404 Not Found](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#404-not-found)。

## 保留与删除

会话端点是只读的；无法通过 Compliance API 删除本地和远程会话。本地会话对话记录默认保留 6 年，如果您的组织设置了有限的自定义对话保留期，则按该保留期保留，详见[用户机器上的会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)。远程会话对话记录保留 6 年，除非用户提前删除该会话。用户删除会话后，远程会话端点将不再返回该会话，并且其对话记录无法通过 Compliance API 恢复。要了解这些期限与 Anthropic 其他保留安排之间的关系，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="检索和删除聊天、文件和项目" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data">
    使用同一个 Compliance Access Key 访问 claude.ai 聊天内容、文件附件和项目。
  </Card>

  <Card title="Compliance API 常见问题" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq#data-coverage-and-retention">
    逐字段汇总会话对话记录所包含的内容，以及其他常见问题。
  </Card>

  <Card title="处理 Compliance API 错误" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors">
    原样的错误负载及每种错误的修复方法。
  </Card>

  <Card title="API 参考" href="https://platform.claude.com/docs/zh-CN/api/compliance/apps">
    Compliance API 的端点路径、参数和响应架构。
  </Card>
</CardGroup>
