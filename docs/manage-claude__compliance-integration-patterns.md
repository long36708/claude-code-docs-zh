---
title: 设计您的合规集成
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns
description: 在轮询与游标驱动的 Activity Feed 消费方式之间进行选择，将 Compliance API 事件与您的 SIEM 关联，并规划数据保留。
---

<Note>
  要启用 Compliance API，请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。
</Note>

<Check>
  **所需作用域：** Compliance Access Key 或 Admin API 密钥上的 `read:compliance_activities`。
</Check>

一个生产级的 Compliance API 集成需要做出三项设计选择：如何消费 Activity Feed（活动源）、其输出如何与您的"security information and event management"（安全信息与事件管理）系统，即 SIEM 相关联，以及活动和内容的长期副本存放在何处。这些选择与端点本身无关；本页帮助您评估其中的权衡。

本页假设您已阅读以下页面：

* [查询 Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)，其中定义了本文通篇引用的参数和分页约定。
* [检索和删除聊天、文件和项目](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data)，其中定义了聊天、文件和项目端点，以及[规划内容保留](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns#plan-content-retention)中引用的 `deleted_at` 语义。
* [检索会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions)，其中定义了本地和远程会话端点。

## 选择源消费模式

Activity Feed 支持两种消费模式：以 `created_at.gte` 和 `created_at.lt` 为边界的周期性窗口轮询（window polling），以及游标驱动的增量读取（cursor-driven incremental reads）——从一次响应中持久化一个游标，并在下一次请求中传递它。两者返回相同的 `Activity` 对象；区别在于您的客户端在调用之间持久化的状态。

两种模式共享以下约束：

* 活动在发生后 1 分钟内即可查询，并保留 6 年。记录不具有追溯性：它从您的组织首次启用 Compliance API 时开始，启用之前的活动不会被回填。
* 每页的最大 `limit` 为 5,000。
* 游标值是不透明的字符串，您不得对其进行解析。
* 每个[父组织](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api#how-the-compliance-api-works)的请求限制为每分钟 600 次，该限制在所有密钥、所有关联组织以及所有 `/v1/compliance/*` 端点之间共享；与本地会话端点不同，远程会话端点在此之上还有第二个请求预算。有关响应头和重试约定，请参阅 [429 Too Many Requests](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#429-too-many-requests)。

| 模式        | 适用场景                                                      |
| --------- | --------------------------------------------------------- |
| 窗口轮询      | 您的管道按固定计划运行，您偏好无状态的工作进程，并且可以容忍重放或重叠的窗口                    |
| 游标驱动的增量读取 | 您希望活动发生与管道摄取之间的延迟最低，希望避免重新读取已经读完的页面，并且有一个持久的位置可以在运行之间保存游标 |

### 窗口轮询

将 `created_at.lt` 设置为至少 1 分钟之前，以便窗口中的每个活动都已可查询。使用 `created_at.gte` 作为下界，`created_at.lt` 作为上界，使连续的窗口无缝拼接、既无间隙也无重叠；将上一个窗口的 `lt` 值重用为下一个窗口的 `gte`。

```bash cURL
curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/activities" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --data-urlencode "created_at.gte=2026-04-20T07:00:00Z" \
  --data-urlencode "created_at.lt=2026-04-20T08:00:00Z" \
  --data-urlencode "limit=5000"
```

当响应中 `has_more: true` 时，表示该窗口包含多于一页的活动。您可以在窗口内分页，将响应的 `last_id` 作为下一次请求的 `after_id` 传递（当 `has_more` 为 `false` 时停止），或者选择更小的时间窗口。完整约定请参阅[分页结果](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#paginate-results)。

即使拼接得很干净，在其窗口关闭后才被索引的活动也永远不会出现在后续窗口中。请基于活动 `id` 去重，并且要么将每个新窗口加宽，使其与前一个窗口重叠几分钟，要么运行一个周期性的对账过程，重新查询较早的窗口。

<Warning>
  `created_at.lt` 边界过于接近当前时间会静默且永久地丢弃延迟索引的活动：一旦 `created_at.gte` 越过它们，后续任何窗口都无法恢复它们。请将 1 分钟可查询性这一数字视为文档记载的索引延迟，而非一个宽松的建议。
</Warning>

### 游标驱动的增量读取

```bash cURL
first_id="activity_01XyDMpzjS89pFZXqSFUBDr6"  # first_id from a previous response

curl --fail-with-body -sS -G \
  "https://api.anthropic.com/v1/compliance/activities" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY" \
  --data-urlencode "limit=5000" \
  --data-urlencode "before_id=$first_id"
```

逐页读取直到 `has_more` 为 `false`，然后持久化最终响应中的 `first_id`，并在下次运行时将其原样作为 `before_id` 传递，以检索比已保存游标更新的活动。若要反方向遍历以进行回填，请改为持久化 `last_id` 并将其作为 `after_id` 传递。有关游标与页面令牌的完整参考及重试语义，请参阅[分页结果](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#paginate-results)。

生产环境中的\*\*追赶（catch-up）\*\*循环通过 `has_more` 和 `first_id` 驱动迭代，获取自上次轮询以来记录的活动：

```text
cursor = stored_cursor
loop:
  page = GET /v1/compliance/activities?before_id={cursor}&limit=5000
  store(page.data)
  if page.first_id is not null:
    cursor = page.first_id
  if not page.has_more: break
persist(cursor)
```

游标在密钥轮换后仍然有效；请参阅[管理和轮换密钥](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#manage-and-rotate-keys)。

<Warning>
  每一页都与您传递的游标相邻：循环朝着当前时间向前推进，每次一页。当 `has_more` 为 `true` 时，不要将单个响应视为已追赶完毕。仅在 `has_more` 为 `false` 之后才持久化游标；未获取的页面是位于本次响应的 `first_id` 与当前时间之间的较新页面，在您完成循环或再次运行之前，它们将保持未读状态。
</Warning>

## 与您的 SIEM 关联

每个 `Activity` 都携带可与您 SIEM（Splunk、Datadog、Microsoft Sentinel、Cribl 或类似系统）中已有事件进行关联的字段：

| Compliance API 字段     | 关联目标                 |
| --------------------- | -------------------- |
| `actor.user_id`       | 您的身份提供商的稳定用户标识符      |
| `actor.email_address` | 当稳定 ID 不可用时使用目录电子邮件  |
| `actor.ip_address`    | 网络、VPN 和终端日志         |
| `actor.user_agent`    | 终端和设备清单，以及发出请求的客户端应用 |
| `created_at`          | 跨任意来源的时间窗口关联         |

当 `actor.type` 为 `user_actor` 时，`actor.user_id` 和 `actor.email_address` 存在。`actor.ip_address` 和 `actor.user_agent` 在某些 actor 类型上不存在，例如 `anthropic_actor` 和 `scim_directory_sync_actor`。在读取这些字段之前请先检查判别字段。`user_id` 是用户账户的稳定、不透明标识符：它在每个 Compliance API 端点和活动负载中保持一致，并且在用户的电子邮件或显示名称更改时不会改变。请使用 `user_id` 而非 `email_address` 作为主要关联键。

对 Compliance API 本身的调用会产生 `compliance_api_accessed` 活动。请将这些活动与其他活动类型一同摄取，以便您的 SIEM 记录谁在何时查询了合规数据。传递 `activity_types[]=compliance_api_accessed` 以限定查询范围，然后在您的客户端中，从每个 `actor.type` 为 `api_actor` 的活动中读取 `actor.api_key_id`，以将访问归因到特定的 Compliance Access Key 或 Admin API 密钥。

## 规划内容保留

五个保留期限决定了您之后可以检索的内容：

| 数据                       | 保留时长                             | 控制方                                   |
| ------------------------ | -------------------------------- | ------------------------------------- |
| Activity Feed 记录         | 6 年                              | Anthropic                             |
| 聊天、文件和项目内容               | 您组织的 claude.ai 保留策略，除非用户更早将其删除   | 您的组织                                  |
| 本地会话记录（用户机器上的会话）         | 默认 6 年，或在设置了有限期限时采用您组织的自定义对话保留期限 | 默认由 Anthropic 控制；当您的组织设置自定义期限时由您的组织控制 |
| 远程会话记录（云端会话）             | 6 年                              | Anthropic                             |
| 通过 Compliance API 硬删除的内容 | 不保留；删除立即生效且永久                    | `DELETE` 端点的调用方                       |

要了解 Claude Platform 其他部分如何处理保留，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。

请按如下方式在"导出并归档"与"按需 API 检索"之间做出决定：

* 如果您的法律保留或审计期限对活动元数据或会话记录的要求超过 6 年，请在摄取时将 Activity Feed 页面和会话记录导出到您自己的归档中。
* 如果您的内容保留策略短于您的电子取证（eDiscovery）期限，请在保留窗口到期之前导出聊天和文件内容；Compliance API 无法返回已被保留策略移除的内容。本地会话记录同样如此，当设置了有限期限时，它们遵循您组织的自定义对话保留期限，即使该期限短于 6 年。一旦设置更改，本地会话端点会立即停止返回早于您组织当前期限的消息，并且之后延长期限也不会恢复已过期的记录，因此请导出任何您必须保留超过该期限的记录。
* 如果您必须在用户于 claude.ai 中删除聊天内容后仍保留该内容（例如，处于法律保留之下），请在摄取时将聊天、文件和 artifact 内容导出到您自己的归档中；Compliance API 无法返回用户已删除的内容。
* 如果某个工作流可能发出 Compliance API 硬删除（例如 DLP 强制执行），请先检索并归档目标内容。硬删除之后没有恢复窗口。

在所有其他情况下，请依赖直接的 API 检索，避免维护并行副本。

### 交付保证与完整性

请将 Activity Feed 视为\*\*至少一次（at-least-once）\*\*交付：正确分页的遍历会至少返回每个活动一次，但在部分失败后的重试可能会重新交付您已存储的活动。请基于活动的 `id` 字段去重。

列表端点不返回 `total_count` 字段或校验和。要证明一次导出运行是完整的，请记录：

* 起始游标和终止 `last_id`。
* 导出的记录数量。
* 运行时间戳和最后一页的 `request-id`。

活动数量不是完整性检查。`claude_*_viewed` 活动类型（例如 `claude_chat_viewed`）遵循各应用的加载模式（请参阅[了解 Activity 对象](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#understand-the-activity-object)）。某个时段有聊天消息但没有 `claude_chat_viewed` 活动，本身并不表示数据缺失。请改为依赖遍历以及[窗口轮询](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns#window-polling)中描述的重叠或对账过程。

内容端点（聊天、文件、项目、项目附件，以及本地和远程会话记录）仅提供 Claude Enterprise 数据。Activity Feed 呈现整个组织范围内的管理和资源事件。Compliance API 不包括：

* 来自 Claude Console 的提示文本或模型响应，或来自使用 API 密钥认证的 Claude API 工作负载的提示文本或模型响应。
* 本地会话中从未发送给 Anthropic 的设备端活动，例如 Claude 未读取的本地文件。
* 使用 Claude Console API 密钥认证、通过第三方云平台（Amazon Bedrock、Google Cloud 或 Microsoft Foundry）运行、或在网页版 Claude Code 中运行的 Claude Code 使用情况。
* 来自已启用 [HIPAA 就绪](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#hipaa-readiness)的组织的本地会话，以及[零数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#zero-data-retention-zdr-scope)生效的本地会话。
* 会话记录中的思考块，以及图像或其他二进制内容（记录仅包含用户提示、助手响应和工具活动；本地会话记录在省略二进制内容的位置显示一个占位 `text` 块）。
* claude.ai 以提取文本形式存储的聊天附件的原始文件，例如某些 Word、PowerPoint 和 PDF 上传（文件内容端点返回提取的文本；请参阅[检索文件和 artifact](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data#retrieve-files-and-artifacts)）。
* 本地会话的系统提示（由一条标记消息代替）。
* 会话记录（本地或远程）中的工具定义和 MCP 服务器配置，以及本地会话记录中 `text` 块上的引用元数据。
* 其[客户管理的加密密钥](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)当前无法使用的组织中的本地会话记录内容。这些请求返回 [503 Service Unavailable](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-sessions-temporarily-unavailable)，而会话元数据仍会被列出。
* 被您组织的保留策略移除的内容。
* 用户在 claude.ai 中删除的聊天的内容（这些聊天仍会被列出，并填充 `deleted_at`）。
* 通过 Compliance API 硬删除的内容。

有关 Compliance API 捕获和不捕获哪些内容的更多信息，请参阅 [Compliance API 常见问题](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq#data-coverage-and-retention)。

为保证监管链（chain of custody），请将导出的记录与来源元数据一同存储：源端点、查询参数、运行时间戳，以及每条记录的内容哈希。

## 后续步骤

<CardGroup cols={2}>
  <Card title="查询 Activity Feed" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed">
    筛选参数、分页以及 `Activity` 对象架构。
  </Card>

  <Card title="检索和删除聊天、文件和项目" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data">
    聊天、文件和项目端点，包括硬删除。
  </Card>

  <Card title="检索会话记录" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions">
    列出您的用户在 Claude 应用和智能体（例如 Cowork 和 Claude Code）中运行的会话，并检索其记录。
  </Card>
</CardGroup>
