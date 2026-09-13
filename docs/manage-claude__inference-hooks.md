---
title: 推理钩子
url: https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks
description: 在推理继续之前，将每个受管控的提示发送到您组织的 AI 安全服务器，以获取允许或拒绝的裁决。
---

<Note>
  Inference hooks（推理钩子）目前处于 beta 阶段，面向 Claude Enterprise 组织提供。配置它们需要在 claude.ai 中拥有 `organization:manage` 权限，内置的 Admin、Owner 和 Primary owner 角色均持有该权限；请参阅[配置推理钩子](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)。
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

下图追踪了一个示例（一个 Cowork 请求，其中 Claude 还调用了一个 O365 工具），以说明流程中哪些部分被钩住。被钩住的点是图中的步骤 1 和步骤 5，即提示到达和工具结果返回之处；每一处都会引发与您的 AI 安全服务器之间的验证交换，如步骤 2 和步骤 6 所示。

![流程图：AI security server（AI 安全服务器）在推理继续之前同时验证 prompt（提示）和 tool result（工具结果）](https://platform.claude.com/docs/images/inference-hooks-flow.svg)

裁决是一个小型 JSON 对象：`{"action": "allow"}` 允许请求继续，而拒绝则携带面向用户的原因。有关完整的裁决架构，请参阅[返回裁决](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#return-a-verdict)。

您的 AI 安全服务器看到的内容与用户看到的相同：对话记录文本、工具调用及其结果，以及从附件中提取的文本。它永远不会收到原始文件或图像字节、系统提示或 Anthropic 内部上下文。Anthropic 不会将提示或响应内容作为推理钩子的一部分进行存储；它仅记录有关钩子活动的元数据，例如裁决、时间戳和请求标识符。

如果您的 AI 安全服务器无法访问、返回错误或未在超时时间内响应，则由您组织的故障处理设置决定结果：阻止该请求，或允许其在未经检查的情况下继续。

执行可以按照您的节奏逐步推出，因此没有人需要在第一天就被阻止：影子模式在实时流量上观察裁决而不阻止任何内容，推出百分比检查所选比例的请求，而排除项则完全豁免所选角色的成员。请参阅[配置推理钩子](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)。

有关完整的请求和响应架构、签名验证以及运维细节，请参阅[开发集成](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint)。

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

推理钩子面向 Claude Enterprise 组织提供。配置它们需要 `organization:manage` 权限，内置的 Admin、Owner 和 Primary owner 角色持有该权限，任何被授予该权限的自定义角色也同样持有。

一个钩子即可管控您的 Claude Enterprise 组织中跨 claude.ai、Cowork 和 Claude Code 会话的对话，无论它们运行在网页端、桌面应用还是 CLI 中。推理钩子在 Amazon Bedrock 或 Google Cloud 上不可用。

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
