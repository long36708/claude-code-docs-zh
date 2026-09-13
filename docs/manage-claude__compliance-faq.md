---
title: Compliance API 常见问题
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq
description: 关于 Compliance API 访问、作用域、保留期限和集成的常见问题解答。
---

<Note>
  要启用 Compliance API，请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。
</Note>

## 访问与作用域

<AccordionGroup>
  <Accordion title="谁可以启用 Compliance API？">
    对于 Claude Enterprise 组织，主要所有者在 [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access) 中启用 Compliance API，启用状态会从父组织级联到每个关联组织。对于符合条件的独立 Claude Console 组织（即没有父组织的组织），组织管理员在 [Claude Console > Settings > Security](https://platform.claude.com/settings/security) 中启用它。关联到父组织的 Claude Console 组织不会自行启用 Compliance API；它由父组织启用。有关步骤，请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)。
  </Accordion>

  <Accordion title="在 Claude Console 中启用 Compliance API 后，我可以将其关闭吗？">
    可以。对于独立的 Claude Console 组织，组织管理员可以在 [Claude Console > Settings > Security](https://platform.claude.com/settings/security) 中关闭 **Compliance API** 开关，这与开启它的位置相同。当 Compliance API 处于关闭状态时，不会为您的组织记录任何活动事件，因此 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)（活动源）不会收到新事件。如果您的组织已注册 [Access Transparency](https://platform.claude.com/docs/zh-CN/manage-claude/access-transparency)（访问透明度），关闭 Compliance API 也会停止 Access Transparency 事件的传送。Compliance API 关闭期间未记录的活动以后无法恢复。重新开启 Compliance API 会从该时间点起恢复记录；已记录的活动不会被删除。
  </Accordion>

  <Accordion title="关闭 Compliance API 会删除已捕获的事件吗？">
    不会。关闭 Compliance API 会停止记录新的活动事件，但不会删除在其开启期间已捕获的事件。记录会从 Compliance API 重新开启的时间点起恢复。
  </Accordion>

  <Accordion title="在 Claude Console 中关闭 Compliance API 的操作会被记录在某处吗？">
    会。当在 Claude Console 中关闭（或重新开启）Compliance API 时，该更改会作为 `org_compliance_api_settings_updated` 活动记录在 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed) 中，因此您的审计跟踪会显示谁在何时更改了该设置。此活动是记录停止的一个例外：即使 Compliance API 关闭期间不记录其他任何活动，禁用操作本身仍会被记录。
  </Accordion>

  <Accordion title="为什么在创建 Admin API 密钥时，我的父组织没有出现在 Claude Console 中？">
    这是预期行为。Claude Enterprise 父组织集中管理所有关联组织的身份；它不承载工作负载，也根本不会出现在 Claude Console 中。Claude Console 只会显示关联在父组织之下的 Claude Console 组织。

    要调用 Compliance API，您需要改为创建以下两种密钥类型之一：

    * \*\*如需完整的 Compliance API 访问权限（[Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed) 以及聊天、文件、项目、会话、用户、组织元数据和组织设置），\*\*由父组织的主要所有者（或组织所有者，用于仅限其自身组织的密钥）在 claude.ai 中创建 [Compliance Access Key](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)（合规访问密钥）。
    * \*\*如仅需 Activity Feed 访问权限，\*\*由您的 Claude Console 组织中的组织管理员在 Claude Console 中创建 [Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#create-an-admin-api-key)。该组织必须已启用 Compliance API，并且管理员必须在 Compliance API 处于启用状态时创建 Admin API 密钥，该密钥才会携带 `read:compliance_activities` 作用域。
  </Accordion>

  <Accordion title="我可以将常规的 Claude API 密钥用于 Compliance API 吗？">
    不可以。Claude API 密钥（`sk-ant-api03-...`）用于对 Claude API 上的 Claude 模型调用进行身份验证；它不能对 `/v1/compliance/*` 的调用进行身份验证。Compliance API 仅接受 Compliance Access Key（`sk-ant-api01-...`）和 Admin API 密钥（`sk-ant-admin01-...`）。有关完整的对应关系，请参阅[您需要哪种密钥？](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#which-key-do-you-need)。
  </Accordion>

  <Accordion title="为什么我的 Admin API 密钥在聊天或文件端点上返回 403？">
    Admin API 密钥携带固定的 `read:compliance_activities` 作用域，该作用域仅授权访问 Activity Feed。其他所有 Compliance API 端点都需要只有在 claude.ai 中创建的 Compliance Access Key 才能携带的作用域。使用 Admin API 密钥调用内容或目录端点会返回 403，并指明该端点系列所需的作用域：聊天、文件、项目、项目附件、会话、用户和组成员需要 `read:compliance_user_data`，组织、角色、组和有效组织设置需要 `read:compliance_org_data`。例如，列出聊天会返回以下响应。

    ```json Response
    {
      "error": {
        "type": "permission_error",
        "message": "Missing required scopes. Got: ['read:compliance_activities'] Needed: ['read:compliance_user_data']"
      }
    }
    ```

    要访问内容端点，您的父组织的主要所有者（或组织所有者，仅限其自身组织）必须[创建 Compliance Access Key](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)，并携带 `read:compliance_user_data`（删除操作还需 `delete:compliance_user_data`），或者为组织、角色、组和有效设置端点携带 `read:compliance_org_data`。独立的 Claude Console 组织（即没有父组织的组织）无法创建 Compliance Access Key，因此无法使用内容端点；它只能查询 Activity Feed。有关完整的按端点分类的目录，请参阅[处理 Compliance API 错误](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)。
  </Accordion>
</AccordionGroup>

## 数据覆盖范围与保留期限

<AccordionGroup>
  <Accordion title="Activity Feed 可以追溯到多久以前？">
    Activity Feed 保留 6 年的组织活动，新事件在发生后 1 分钟内即可查询。该活动源最多只能追溯到您的组织首次启用 Compliance API 的时间点：记录不具有追溯性，启用之前的活动不会被回填。Activity Feed 的保留期限独立于您组织的内容保留策略：聊天、文件和项目内容遵循为您的组织配置的保留规则（默认为无限期），除非用户提前将其删除。
  </Accordion>

  <Accordion title="Activity Feed 是否包含提示或消息内容？">
    不包含。Activity Feed 记录谁在何时做了什么（身份验证、聊天创建、文件上传、项目更改、管理操作以及类似的资源事件），但不会捕获聊天或消息中的提示文本或模型响应。

    要检索消息正文和文件内容，请使用携带 `read:compliance_user_data` 的 Compliance Access Key 调用聊天、消息和文件端点。同一密钥和作用域可通过[本地会话端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)检索用户机器上的会话（例如 Cowork 和 Claude Code 会话）的转录记录，并通过[远程会话端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-remote-sessions)检索云端 Cowork 会话的转录记录。这些端点仅提供 Claude Enterprise 内容；Claude Console 工作负载以及使用 API 密钥进行身份验证的 Claude API 工作负载会通过 Activity Feed 公开管理和资源事件，但不会通过 Compliance API 公开提示文本或模型响应。
  </Accordion>

  <Accordion title="Cowork、Claude Code、Claude Science 和 Claude for Microsoft 365 会话是否会出现在 Compliance API 中？">
    会。在用户机器上运行的 Claude Desktop 中的 Cowork 会话、Claude Code 会话（在终端、Claude Desktop 或 IDE 扩展中）、Claude Science 桌面应用中的会话，以及 Excel、PowerPoint、Word 和 Outlook 中的 Claude for Microsoft 365 会话，在用户使用其 Claude Enterprise 账户登录期间都会被捕获，并可通过[本地会话端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)获取。在 claude.ai 网页版或移动版上启动的 Cowork 会话在 Anthropic 管理的云端环境中运行，可通过[远程会话端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-remote-sessions)获取。每个系列都有一个返回会话元数据的列表端点和一个返回会话转录记录（用户提示、助手响应以及工具调用和结果）的消息端点。本地系列还增加了第三个端点，用于检索单个会话的元数据。所有这些端点都使用您现有的携带 `read:compliance_user_data` 的 Compliance Access Key；无需新的密钥或作用域。

    本地会话在其请求到达 Claude API 时被捕获，因此设备上无需安装任何内容，而从未到达 API 的设备端活动不会被捕获。使用 Claude Console API 密钥进行身份验证的 Claude Code 会话、通过第三方云平台（Amazon Bedrock、Google Cloud 或 Microsoft Foundry）运行的 Claude Code 会话，以及网页版 Claude Code 均不会被捕获。网页版 Claude Code 同样在 Anthropic 管理的云端环境中运行，但它不是远程会话；远程会话端点仅返回 Cowork 会话。启用了 [HIPAA 就绪](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#hipaa-readiness)的组织不会获得本地会话数据，并且适用[零数据保留（ZDR）](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#zero-data-retention-zdr-scope)的会话会被排除。

    本地和远程会话端点对于 Cowork 和 Claude Code 会话是稳定的；对 Claude Science 和 Claude for Microsoft 365 会话的覆盖处于测试阶段。
  </Accordion>

  <Accordion title="会话转录记录包含哪些内容？">
    本地和远程会话转录记录都包含用户提示、助手响应以及工具调用和结果。对于本地会话（在用户机器上），记录的是 Claude 被要求做什么以及它返回了什么，而不是设备上发生了什么。

    | 数据         | 本地会话（在用户机器上）                                                                                                                                                             | 远程会话（在云端）                                                                                                                                                                |
    | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
    | 用户提示       | 是；以 `text` 块形式返回。                                                                                                                                                        | 是；以 `text` 块形式返回。                                                                                                                                                        |
    | 助手响应       | 是；仅文本输出。                                                                                                                                                                 | 是；仅文本输出。                                                                                                                                                                 |
    | 工具调用和结果    | 是；每个 `tool_use` 输入和 `tool_result` 中的每个 `text` 条目默认截断为 10,000 字节（可应请求提高至每个约 1 MiB）。                                                                                       | 是；每个 `tool_use` 输入和 `tool_result` 中的每个 `text` 条目默认截断为 10,000 字节（可应请求提高至每个约 1 MiB）。                                                                                       |
    | 文件内容和文件名   | 是；Claude 通过工具读取的文本会出现在转录记录中，并受相同的截断限制。图像、PDF 以及其他二进制或结构化内容仅以占位符 `text` 块形式出现。文件名出现在工具调用的输入和输出中。                                                                          | 是；文件内容和文件名通过工具调用的输入和输出出现在转录记录中（仅文本；其他内容被省略）。                                                                                                                             |
    | Artifacts  | 是；生成的内容出现在转录记录中的工具调用输入内。                                                                                                                                                 | 是；生成的内容出现在转录记录中的工具调用输入内。                                                                                                                                                 |
    | 技能         | 是；当客户端将技能内容作为消息内容发送时，技能内容会出现，且不会与其他用户文本区分开来。                                                                                                                             | 是；技能内容出现在转录记录中。                                                                                                                                                          |
    | 会话元数据      | 是；所有者（`user.id` 和电子邮件地址）、组织、工作区、`product_surface`、`created_at` 和 `updated_at`，来自列表和检索端点。本地会话不携带 `status`。                                                                | 是；所有者、组织、状态、时间戳和 `product_surface`，来自列表端点。                                                                                                                               |
    | 思考块        | 否。                                                                                                                                                                       | 否。                                                                                                                                                                       |
    | 图像和其他非文本内容 | 否；每个图像、PDF 或其他二进制或结构化块以占位符 `text` 块形式出现（例如 `[image content not shown]`），且 `truncated` 设置为 `true`。原始文件字节永远不会被返回。                                                          | 否；非文本块被省略，原始文件字节永远不会被返回。                                                                                                                                                 |
    | 令牌用量、成本和延迟 | 否；令牌用量和成本可通过 [Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api#get-access-to-the-claude-enterprise-analytics-api) 获取。 | 否；令牌用量和成本可通过 [Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api#get-access-to-the-claude-enterprise-analytics-api) 获取。 |

    有关端点和参数，请参阅[用户机器上的会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)和[云端会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-remote-sessions)。
  </Accordion>

  <Accordion title="会话覆盖范围与 Cowork 和 Claude Code 的 OpenTelemetry 日志记录（OTEL）相比如何？">
    [Cowork 的 OpenTelemetry 日志记录](https://support.claude.com/en/articles/14477985-monitor-claude-cowork-activity-with-opentelemetry)和 [Claude Code 监控](https://code.claude.com/docs/en/monitoring-usage)与会话端点有所重叠，但满足不同的需求：OTEL 在活动发生时将逐事件遥测数据流式传输到您运行的基础设施，而 Compliance API 让您可以在事后从 Anthropic 检索保留的逐会话转录记录。OTEL 也可以捕获提示和响应，但 Anthropic 建议使用 Compliance API 来检索 Cowork 和 Claude Code 会话的内容。有关比较本地会话、远程会话和 OTEL 的表格，请参阅[检索会话转录记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions)的简介。

    OTEL 事件和 Compliance API 记录共享组织和用户标识符，因此您可以将它们关联起来。
  </Accordion>

  <Accordion title="已删除的内容可以通过 Compliance API 恢复吗？">
    不可以。通过 Compliance API 执行的删除是即时、永久且不可恢复的。用户在 claude.ai 中删除的聊天内容同样不可恢复：Compliance API 仍会返回该聊天及其消息，并填充 `deleted_at`，但不会返回其内容。请在内容仍可用时提取您需要保留的任何内容（用于法律保留或归档）。有关何时将内容导出到您自己的归档，请参阅[规划内容保留](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns#plan-content-retention)。
  </Accordion>

  <Accordion title="Compliance API 不捕获哪些内容？">
    Compliance API 有已知的覆盖边界：Activity Feed 记录资源事件但不记录提示或响应文本；Claude Console 工作负载以及使用 API 密钥进行身份验证的 Claude API 工作负载完全不公开消息内容；被您的保留策略移除、被用户在 claude.ai 中删除或通过 Compliance API 硬删除的内容不可恢复。有关完整的覆盖边界和传送契约，请参阅[传送保证与完整性](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns#delivery-guarantees-and-completeness)。

    会话转录记录也有其自身的边界。本地会话仅在其请求到达 Claude API 时被捕获，因此从未到达 API 的设备端活动不会被捕获。使用 Claude Console API 密钥进行身份验证的 Claude Code 会话、通过第三方云平台（Amazon Bedrock、Google Cloud 或 Microsoft Foundry）运行的 Claude Code 会话，以及网页版 Claude Code 同样不会被捕获；启用了 HIPAA 就绪的组织不会获得本地会话数据；适用零数据保留的会话会被排除。任何会话转录记录（无论本地还是远程）都不包含思考块或工具定义。使用[客户管理的加密密钥](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)的组织照常接收本地会话转录记录。当密钥无法使用时，消息端点会返回 [503 Service Unavailable](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-sessions-temporarily-unavailable) 而非转录内容，而会话元数据仍会被列出。
  </Accordion>
</AccordionGroup>

## 集成与分页

<AccordionGroup>
  <Accordion title="如何将 Compliance API 记录与我的 SIEM 关联？">
    通过 `actor.user_id`、`actor.email_address`、`actor.ip_address`、`actor.user_agent` 和 `created_at` 将 `Activity` 记录与您的 SIEM 关联。有关关联键表和使用模式，请参阅[设计您的合规集成](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns#correlate-with-your-siem)。
  </Accordion>

  <Accordion title="一个客户可以在一个父组织下拥有多个组织吗？">
    可以。一个 Claude Enterprise 父组织可以拥有多个关联组织，包括 claude.ai 组织和 Claude Console 组织的混合（例如，分开的生产和预发布 Claude Console 组织）。身份、SSO 和 SCIM 在父组织范围内共享；计费、成员、项目和 API 密钥对每个组织保持独立。Compliance API 的启用在父组织级别进行并级联到所有关联组织，而覆盖父组织并携带 `read:compliance_org_data` 的 Compliance Access Key 可以通过 `GET /v1/compliance/organizations` 枚举父组织下的每个组织。
  </Accordion>

  <Accordion title="活动是否按顺序返回？我如何检测何时已追上实时数据？">
    活动按从新到旧的顺序返回，`created_at` 相同时按活动 ID 排序。要追上进度，请通过 `before_id` 向前遍历页面，直到 `has_more` 为 `false`；该最终响应的 `first_id` 即为您的新游标，此时您已到达当前时间点。完整的循环（包括初始回填以及游标持久化的安全条件）请参阅[游标驱动的增量读取](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns#cursor-driven-incremental-reads)。
  </Accordion>

  <Accordion title="如何获取沙盒来测试 Compliance API？">
    如果仅测试 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)，您不需要 Claude Enterprise 组织：组织管理员可以在符合条件的独立 Claude Console 测试组织上[启用 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)，并使用新的 Admin API 密钥查询该活动源。如果该组织的 Security 设置中看不到 **Compliance API** 部分，则该组织不符合自助启用的条件。

    要测试每个端点，请设置一个 Claude Enterprise 沙盒组织，并将其与同一父组织下的 Claude Console 组织关联。这样沙盒既可以测试 Activity Feed（通过 Admin API 密钥），也可以测试聊天、文件、项目和会话端点（通过 Compliance Access Key）。

    1. \*\*配置 Claude Enterprise 组织。\*\*联系您的 Anthropic 代表以设置 Claude Enterprise 沙盒组织。对于现有的 Claude Enterprise 组织，主要所有者可以[直接在 claude.ai 中启用 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)。
    2. \*\*创建 Claude Console 组织。\*\*使用相同的电子邮件地址在 `platform.claude.com` 上自行创建一个 Claude Console 组织。
    3. \*\*关联两个组织。\*\*以 Claude Enterprise 组织的主要所有者身份登录，前往 [claude.ai > Organization settings > Identity and access](https://claude.ai/admin-settings/identity)，并使用 **Merge Organizations** 将两者关联到一个共享的父组织下。

    关联完成后，请按照[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access) 创建密钥并开始查询。测试组织使用与生产组织相同的启用流程。
  </Accordion>
</AccordionGroup>
