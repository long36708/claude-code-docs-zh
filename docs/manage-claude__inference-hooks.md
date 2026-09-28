---
title: 推理钩子
url: https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks
description: 在推理继续之前，将每个受管控的提示发送到您组织的 AI 安全服务器，以获取允许或拒绝的裁决。
---

<Note>
  推理钩子目前处于 beta 阶段，面向 Claude Enterprise 组织提供。配置推理钩子需要 claude.ai 中的 `organization:manage` 权限，只有所有者（Owner）和主要所有者（Primary owner）角色拥有该权限；请参阅[配置推理钩子](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)。
</Note>

推理钩子让 Claude Enterprise 组织能够在推理运行之前，将每个受管控的提示路由到一个 AI 安全服务器——即由该组织或其安全供应商运营的 HTTPS 服务。当用户提交提示时，Anthropic 会将对话记录发送到您的 AI 安全服务器，并等待允许或拒绝的裁决；被拒绝的请求永远不会到达模型。安全与合规团队使用推理钩子来内联执行数据策略，而开发人员则构建用于评估每个请求的 AI 安全服务器。

由于钩子运行在 Anthropic 的服务器上——在请求离开客户端之后、模型运行之前——它会统一应用于每个受管控的请求，无需在用户设备上安装或部署任何内容。

目前唯一的钩子事件是 `prompt`，它在每个受管控的推理请求开始推理之前触发一次。响应侧的执行计划作为后续事件推出。

***

## 推理钩子的工作原理

1. 用户在受管控的界面上提交提示。
2. Anthropic 向您组织配置的 AI 安全服务器端点发送 HTTPS `POST` 请求。请求正文携带对话记录，并且一旦您的组织生成了签名密钥，每个请求都会按照 [Standard Webhooks](https://www.standardwebhooks.com/) 规范进行签名，以便您的服务器可以验证它来自 Anthropic。
3. 您的 AI 安全服务器评估内容，并在您组织配置的裁决超时时间内（默认为 5 秒）返回裁决。
4. 若为 `allow`，推理正常进行。若为 `deny`，请求将被拒绝，用户会看到一条由两部分组成的"被策略阻止"消息：首先是您的 AI 安全服务器在裁决的 `deny_reason` 字段中提供的针对该请求的原因，其后是您的管理员配置的常设消息（例如，应联系谁或在哪里申请例外）。如果您的管理员尚未配置，内置的默认消息会引导用户联系管理员。每次拒绝也会记录在您组织的[活动源](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)中。

下图追踪了一个示例（一个 Cowork 请求，其中 Claude 还调用了一个 O365 工具），以说明流程中哪些部分会被钩子拦截。被钩子拦截的点是图中的第 1 步和第 6 步，即提示到达和工具结果返回之时；每个拦截点都会触发与您的 AI 安全服务器之间的验证交互，如第 2–3 步和第 7–8 步所示。

![流程图：AI security server（AI 安全服务器）在推理继续之前验证提示和工具结果](https://platform.claude.com/docs/images/inference-hooks-flow.png)

裁决是一个小型 JSON 对象：`{"action": "allow"}` 允许请求继续，而拒绝则携带面向用户的原因。有关完整的裁决架构，请参阅[返回裁决](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#return-a-verdict)。

您的 AI 安全服务器看到的内容与用户看到的内容相同：对话记录文本、工具调用及其结果，以及从附件中提取的文本。它永远不会收到原始文件或图像字节、系统提示或 Anthropic 内部上下文。

推理钩子系统本身不会保留提示或响应内容的副本。它仅存储您的钩子配置以及有关钩子活动的元数据，例如裁决、时间戳和请求标识符。无论是否启用钩子，您使用的 Claude 产品都会按照其自身的数据保留规则存储提示和响应。例如，在 claude.ai 上被钩子阻止的消息仍会保留在对话中。

如果您的 AI 安全服务器无法访问、返回错误或未在超时时间内响应，则由您组织的故障处理设置决定结果：阻止请求，或允许请求在未经检查的情况下继续。由您的服务器导致的持续故障会触发 "circuit breaker"（熔断器）：Anthropic 将停止联系您的服务器，并对每个请求应用您的故障处理设置；一旦检测到您的服务器再次返回裁决，熔断器会自动重置。请参阅[熔断器](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration#circuit-breaker)。

执行可以按照您的节奏逐步推出，因此没有人需要在第一天就被阻止：影子模式在实时流量上观察裁决而不阻止任何内容，推出百分比检查所选比例的请求，而排除项则完全豁免所选角色的成员。请参阅[配置推理钩子](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)。

有关完整的请求和响应架构、签名验证以及运维细节，请参阅[开发集成](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint)。

***

## 在请求被拒绝后继续对话

每个请求都包含整个对话，因此被拒绝的消息会随之后的每条消息再次发送。如果您的 AI 安全服务器评估整个对话记录，它也会拒绝这些请求。要继续对话，用户需要从应用接下来发送的内容中移除被拒绝的内容，包括 Claude 会再次读取的任何文件。

具体步骤取决于所使用的应用：

* **claude.ai，包括 Claude Desktop 和移动应用。** 用户应编辑被拒绝的消息或更早的消息，而不是将更正后的副本作为新消息发送。在网页端和 Claude Desktop 中，除非用户移除附件，否则编辑操作会重新发送该消息的附件。新建聊天也可以。
* **Claude Code。** 用户运行 `/rewind` 并选择最初引入该内容的提示。如果系统询问，用户选择 **Restore conversation**，然后编辑或清除返回到输入框中的提示。`/clear` 会重新开始。请参阅[检查点](https://code.claude.com/docs/zh-CN/checkpointing)。
* **Cowork。** 如果被拒绝的消息是用户的最新消息，用户可以编辑该消息。编辑操作会重新发送附加的文件，而 **Restart from here** 会原样重新发送整条消息。如果该内容位于文件或更早的消息中，用户应选择 **New task**。
* **Claude Tag。** 在 Slack 中，用户首先编辑被拒绝的消息；如果该消息是回复，则改为将其删除。然后在 Claude 回答的位置单独发送 `@Claude !restart`：即在该线程中，或在频道的顶层。新会话会重新读取仍保留在 Slack 中的消息，因此必须先进行编辑或删除。请参阅 [`!restart` 命令](https://claude.com/docs/claude-tag/users/commands#restart-a-stuck-or-wrong-context-session)。

***

## 使用场景

* **"Data loss prevention"（数据丢失防护），即 DLP。** 将对话记录转发到您的 DLP 扫描器，并拒绝携带受监管或机密材料的提示。这是最常见的部署方式。
* **实时对话记录归档。** 在每份对话记录到达时进行归档并始终返回 `allow`，作为轮询 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 的一种基于推送的替代方案。
* **提示遥测。** 在使用发生的当下衡量您的组织如何使用 Claude。
* **策略引擎。** 在推理之前执行您自己的规则：模型允许列表、项目范围的限制或工作时间控制。

***

## 当前限制

* 附件以元数据和提取的文本表示。原始文件和图像字节永远不会被发送，因此仅含图像的内容（例如文档的屏幕截图）不会被检查。
* 裁决只有允许或拒绝。不支持重写或编辑（脱敏）提示。
* 平台组织（通过 Claude Platform 进行 API 访问）不在范围之内。

***

## 可用性

推理钩子面向 Claude Enterprise 组织提供。配置推理钩子需要 `organization:manage` 权限，只有所有者（Owner）和主要所有者（Primary owner）角色拥有该权限。

一个钩子即可管控您 Claude Enterprise 组织中 claude.ai、Cowork、Claude Code 和 Claude Tag 会话中的对话，无论它们运行在网页端、桌面或移动应用、CLI 还是 Slack 中。推理钩子在 Amazon Bedrock 或 Google Cloud 上不可用。

受管控的请求是指用户对话背后的推理请求。辅助请求（例如对话标题生成）不会发送到您的端点，并且系统提示和工具定义永远不会包含在所发送的内容中。语音模式不在覆盖范围内。

***

## 推理钩子与 Compliance API 的对比

这两项功能都服务于 Claude Enterprise 组织的安全、法务和合规团队。

|      | 推理钩子                    | Compliance API                |
| ---- | ----------------------- | ----------------------------- |
| 何时生效 | 内联，在推理运行之前              | 事后                            |
| 作用   | 实时允许或拒绝每个受管控的请求         | 检索活动、聊天、文件、项目、会话记录和用户，用于审计和导出 |
| 方向   | Anthropic 调用您的 AI 安全服务器 | 您调用 Anthropic 的 API           |

使用推理钩子在请求到达模型之前将其阻止，并使用 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 在事后审计所发生的情况。

***

## 本节内容

<CardGroup>
  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration" title="配置推理钩子">
    为您的组织启用推理钩子，设置并测试您的 AI 安全服务器，选择故障处理方式，并执行裁决。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint" title="开发推理钩子集成">
    用于构建 AI 安全服务器的请求和裁决架构、签名验证、运维语义以及集成模式。
  </Card>
</CardGroup>
