---
title: API 与数据保留
url: https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention
description: 了解 Anthropic 的 API 及相关功能如何保留数据，包括有关零数据保留（ZDR）和 HIPAA 就绪 API 访问的信息。
---

本页面涵盖 Claude API（`api.anthropic.com`）、Claude Platform on AWS 以及 [Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)，在这些场景中 Anthropic 是数据处理者。在 Amazon Bedrock 和 Google Cloud's Agent Platform 上，云提供商是数据处理者；请参阅这些平台的数据保留和合规文档以了解其对应的控制措施。

Anthropic 为 Claude API 提供两种数据处理安排：["zero data retention"（零数据保留），即 ZDR](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#zero-data-retention-zdr-scope) 和 [HIPAA 就绪](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#hipaa-readiness)。[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)列出了每种安排所涵盖的 API 功能。有关这些安排之外的 Anthropic 标准保留政策，请参阅[商业数据保留政策](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)和[消费者数据保留政策](https://privacy.claude.com/en/articles/10023548-how-long-do-you-store-my-data)。

## Anthropic 如何处理数据保留

不同的 API 和功能有不同的存储需求。如果某项功能不需要存储客户的提示或响应，则它可能符合 ZDR 资格。如果某项功能必须进行存储，Anthropic 会在以下承诺下设计尽可能小的保留范围：

* 未经您明确许可，保留的数据绝不会用于模型训练。
* 仅保留功能运行在技术上所必需的内容。对话内容（您的提示和 Claude 的输出）默认不会被保留；例外情况是[受管辖模型（Covered Models）](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)，它们要求 30 天的保留期。
* 保留的数据会按照最短的实际可行 "time to live"（生存时间），即 TTL 进行清除，并且 Anthropic 致力于让客户能够控制数据的保留时长。保留了哪些内容，以及在适用特定 TTL 时的保留时长，均记录在各功能的页面上。

有几种保留模式不属于本页所述的 ZDR 和 HIPAA 安排。通过 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 可访问的数据遵循其自身的保留模式。[Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed) 保留数据 6 年。来自 claude.ai 的聊天、文件和项目内容遵循您的组织在 [claude.ai > Organization settings > Data and privacy](https://claude.ai/admin-settings/data-privacy-controls) 中设置的保留政策。[本地会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)（来自用户机器上的会话，例如 Cowork 和 Claude Code 等应用中的会话）默认存储 6 年，或者在设置了有限的自定义对话保留期时，按您组织的自定义对话保留期存储（即同一 claude.ai 设置）。[远程会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-remote-sessions)（云端的 Cowork）保留 6 年。Compliance API 不会捕获 ZDR 生效的本地会话，也不会捕获已启用 HIPAA 就绪的组织的任何本地会话。

## 零数据保留（ZDR）

在 ZDR 安排下，Anthropic 在返回 API 响应后不会静态存储客户的提示或响应。要为您的组织申请 ZDR，请联系 [Anthropic 销售团队](https://claude.com/contact-sales)。ZDR 按组织启用；每个新组织都需要由您的客户团队单独启用 ZDR，且启用不会自动扩展到同一账户下的其他组织。

### ZDR 涵盖的范围

* **Claude Messages 和 Token Counting API：** ZDR 适用于这些端点中[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)所列的符合资格的功能。依托于 `/v1/messages` 但在表中标记为"否"的功能（例如代码执行）不在涵盖范围内。
* **Claude Code：** 当 Claude Code 与来自商业组织（即受 Anthropic 商业服务条款约束的组织，区别于消费者 Claude 账户）的 API 密钥一起使用，或通过已启用 ZDR 的 Claude Enterprise 使用时，ZDR 适用。如果在 Claude Code 中启用了指标日志记录，则使用统计等生产力数据不受 ZDR 约束，可能会被保留。完整详情请参阅 [Claude Code ZDR 文档](https://code.claude.com/docs/en/zero-data-retention)。
* **Claude Platform on AWS：** [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 遵循与第一方 Claude API 相同的数据保留政策。ZDR 可按需提供；请联系您的 Anthropic 客户代表以启用。

### ZDR 不涵盖的范围

* **Claude Console：** Claude Console 中的任何使用，包括 playground。
* **Claude Managed Agents：** Claude Managed Agents 是有状态资源；会话记录会一直保留，直到您将其删除。
* **Claude 消费者产品：** Claude Free、Pro 和 Max 计划，包括这些计划的客户使用 Claude 的网页、桌面或移动应用或 Claude Code 的情况。
* **Claude Teams 和 Claude Enterprise 产品界面：** 这些界面不符合 ZDR 资格。例外情况是通过已启用 ZDR 的 Claude Enterprise 使用的 Claude Code；请参阅 [ZDR 涵盖的范围](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#what-zdr-covers)。
* **Claude for Excel：** 目前不符合 ZDR 资格。
* **Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5 和 Claude Mythos 5：** 这些模型要求 30 天的数据保留，除非获得 Anthropic 明确授权，否则在 ZDR 下不可用。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。
* **第三方集成：** 由第三方网站、工具或其他集成处理的数据不在涵盖范围内，尽管其中一些可能提供类似的服务。请审查每项服务的数据处理做法。
* **"Cross-Origin Resource Sharing"（跨源资源共享），即 CORS：** 具有 ZDR 安排的组织不支持 CORS。要从基于浏览器的应用程序发起 API 调用，请通过后端代理服务器路由请求。有关代理模式和 API 密钥处理，请参阅 [API 安全指南](https://platform.claude.com/docs/zh-CN/api/overview)。
* **被标记的内容和法律保留：** 请参阅[无论何种安排均适用的保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#retention-regardless-of-arrangement)。

<Note>
  有关哪些产品和功能符合 ZDR 资格的最新信息，请参阅您的合同条款或联系您的 Anthropic 客户代表。
</Note>

## HIPAA 就绪

Claude API 为处理 "protected health information"（受保护的健康信息），即 PHI 的组织提供 HIPAA 就绪的集成支持。在签署 BAA 并启用 HIPAA 的组织中，您可以使用受支持的 API 功能处理 PHI，同时支持您组织的 HIPAA 合规。符合资格的组织可以直接在 Claude Console 中审阅并签署 BAA，并启用 HIPAA 就绪。与 ZDR 相比，HIPAA 就绪应用了一套更广泛的隐私和安全保障措施（在 PHI 的整个生命周期内对其进行保护的加密、访问控制和审计日志记录），而不是要求立即删除。如果您的组织处理 PHI，则应使用 HIPAA 就绪这一安排；您无需同时使用 ZDR。有关每种安排涵盖哪些功能，请参阅[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)。

<Note>
  本页面涵盖 Claude API 的 HIPAA 就绪。有关涵盖 Claude Enterprise 和配置要求的完整 HIPAA 实施指南，请参阅 [Anthropic Trust Center](https://trust.anthropic.com/resources)。
</Note>

### HIPAA 就绪涵盖的范围

* **Claude API：** HIPAA 就绪适用于 Claude API（`api.anthropic.com`）中[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)所列的符合资格的功能。

### HIPAA 就绪不涵盖的范围

* **Claude 消费者产品：** Claude Free、Pro 和 Max 计划。
* **Claude Console：** 通过 Claude Console 界面的使用（支持从 Console 设置中启用 HIPAA 就绪；通过 Console 处理 PHI 不在涵盖范围内）。
* **合作伙伴运营的平台：** Amazon Bedrock 和 Google Cloud's Agent Platform。请参阅这些平台的合规文档。
* **Claude Platform on AWS 和 Microsoft Foundry：** 这些平台上不提供 HIPAA 就绪。
* **第三方集成：** 由连接到您应用程序的外部工具或服务处理的数据。
* **Claude Code：** Claude Code 不在 HIPAA 就绪的涵盖范围内。
* **Beta 功能：** 处于 beta 阶段的功能通常不在 BAA 的涵盖范围内，除非在[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)中明确列为符合资格。
* **被标记的内容和法律保留：** 请参阅[无论何种安排均适用的保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#retention-regardless-of-arrangement)。

### PHI 处理指南

受保护的健康信息（PHI）包括任何可识别个人身份的健康信息。在 Claude API 的语境中，PHI 通常出现在消息内容（提示和 Claude 的响应）、附加文件（图像、PDF）以及与消息内容相关的文件名或元数据中。根据 BAA，以下字段预期不包含 PHI：工作区名称、用户信息（姓名、电子邮件、电话号码）、账单数据和支持工单。

当使用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)或带有 `strict: true` 的工具时，API 会将 JSON schema 编译为语法，这些语法与消息内容分开缓存。这些缓存的 schema 不享有与提示和响应相同的 PHI 保护。**请勿在 JSON schema 定义中包含 PHI。** 此限制适用于 schema 属性名称、`enum` 值、`const` 值和 `pattern` 正则表达式。患者特定信息应仅出现在消息内容中，在那里它受到 HIPAA 保障措施的保护。

### HIPAA 错误处理

您签署的 BAA 是确定哪些功能在涵盖范围内的官方权威依据。API 也会自动强制执行这些限制。当启用 HIPAA 的组织发送包含不符合资格功能的请求时，API 会返回 `400` 错误，以防止意外使用您的 BAA 未涵盖的功能：

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "The requested features are not available for HIPAA-regulated organizations without Zero Data Retention: code_execution."
  }
}
```

错误消息会列出在请求中检测到的不符合资格的功能；请移除它们并重试。"without Zero Data Retention" 这一短语是 API 自身的措辞，不会改变解决方法。[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)中"详情"列注明不会被阻止的客户端工具会被接受，但仍不在 HIPAA 就绪的涵盖范围内。

### HIPAA 就绪入门

有两种方式可以设置 HIPAA 就绪的 API 访问。大多数组织可以使用 Anthropic 的标准 BAA 直接在 Claude Console 中启用；需要协商 BAA 的组织应与其客户团队合作。

#### 在 Console 中启用（标准 BAA）

<Steps>
  <Step title="打开您组织的隐私设置">
    在 [Claude Console > Settings > Privacy](https://platform.claude.com/settings/privacy) 中，拥有 HIPAA 管理权限的组织管理员会看到一个 **HIPAA compliance** 卡片。如果您的组织符合资格但您没有看到启用选项，请让组织管理员完成这些步骤。
  </Step>

  <Step title="审阅并签署 BAA">
    下载 Business Associate Agreement（业务伙伴协议）和 HIPAA 实施指南，然后以您组织的授权法律代表身份接受该协议。每个步骤在您下载前一份文档后才可用，并且您的启用将绑定到您所下载的确切 BAA 版本。
  </Step>

  <Step title="启用立即生效">
    一旦您接受，HIPAA 就绪控制措施即会应用于您的组织。一旦为您的组织启用了 HIPAA 就绪，该配置即为永久性的，管理员无法禁用。API 会自动强制执行功能限制，对使用不符合资格功能的请求返回错误。有关该错误和客户端工具例外情况，请参阅 [HIPAA 错误处理](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#hipaa-error-handling)。
  </Step>
</Steps>

#### 联系销售（自定义 BAA）

如果您的组织需要协商或自定义的 BAA，或者您的组织无法使用自助启用，请联系 [Anthropic 销售团队](https://claude.com/contact-sales)。Anthropic 将签署 BAA 并为您的组织启用 HIPAA 就绪。

#### 使用符合资格的功能进行构建

无论您使用哪种途径，请在[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)中确认哪些功能受支持，并针对限制 PHI 出现位置的功能查阅 [PHI 处理指南](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#phi-handling-guidelines)。有关详细的配置和合规要求，请参阅 [HIPAA 实施指南](https://trust.anthropic.com/resources)。

<Warning>
  HIPAA 就绪在组织级别强制执行。如果您同时需要 HIPAA 就绪和通用 API 访问，请为每种用途使用单独的组织。
</Warning>

## 特定模型的数据保留要求

Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5 和 Claude Mythos 5 被指定为受管辖模型（Covered Models）（请参阅[受管辖模型支持文章](https://support.claude.com/en/articles/15425695)），并要求 30 天的数据保留；因此，除非获得 Anthropic 明确授权，否则它们均不提供 ZDR。在 Claude API 上，如果组织的数据保留配置不满足此要求，则向 Claude Fable 5 发出的请求会返回 `400 invalid_request_error`：

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "In order to access this model, your organization or workspace must have data retention enabled."
  }
}
```

30 天数据保留要求适用于提供受管辖模型的所有场景。在 Claude API（包括 Claude Platform on AWS）上，由 Anthropic 处理保留的数据。在 Amazon Bedrock 和 Google Cloud's Agent Platform 上，保留的数据留存在您的云提供商环境中；请查阅各平台的文档以了解启用步骤。

### 为工作区启用 30 天保留

具有 ZDR 安排的组织可以通过仅为特定工作区启用 30 天保留，使这些模型在该工作区中可用。组织中的其他工作区保持零数据保留。

<Steps>
  <Step title="打开工作区的隐私控制">
    在 [Claude Console > Settings > Workspaces](https://platform.claude.com/settings/workspaces) 中，选择该工作区并打开其 **Privacy controls** 选项卡。
  </Step>

  <Step title="开启 30 天数据保留">
    为该工作区启用 30 天数据保留设置。
  </Step>

  <Step title="验证">
    现在，从此工作区向受管辖模型发出的请求将会成功。没有覆盖设置的工作区继续遵循组织默认设置。
  </Step>
</Steps>

## 功能资格

下表列出了哪些 Claude API 功能符合 ZDR 和 HIPAA 就绪安排的资格。

每个资格列使用三个值：

* **是：** 该功能在该安排下完全符合资格。对于 ZDR，"是"还假定您使用的是不要求 30 天数据保留的模型；无论功能资格如何，[受管辖模型](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)在 ZDR 下均不可用。
* **是（有条件）：** 您的提示和 Claude 的输出不会被存储，但会短暂保留一个有限的技术产物（在"详情"列中注明）以使该功能正常运行。有关约束这些功能的承诺，请参阅 [Anthropic 如何处理数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#how-anthropic-approaches-data-retention)。
* **否：** 该功能不符合资格。在 HIPAA 就绪下，API 会阻止包含"否"功能的请求并返回 `400` 错误，除非该功能的"详情"列另有说明。在 ZDR 下，API **不会**阻止这些功能；使用其中某项功能即表示您选择就该特定数据脱离您的 ZDR 安排，并适用该功能自身记录的保留政策。对于 ZDR 标记为"否"的功能通常是有状态的（它们存储作业、文件或容器状态），这就是它们无法实现零保留的原因。

| 功能                                                                                                        | 端点                                             | ZDR 资格                                         | HIPAA 资格                           | 详情                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| --------------------------------------------------------------------------------------------------------- | ---------------------------------------------- | ---------------------------------------------- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)                    | `/v1/messages`                                 | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             |                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)                                | `/v1/messages`                                 | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             |                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [Advisor 工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool)               | `/v1/messages`（带 `advisor` 工具）                 | <Eligible>是</Eligible>                         | <Eligible status="no">否</Eligible> | Advisor 模型输出在 API 响应中返回；响应之后服务器端不存储任何内容。                                                                                                                                                                                                                                                                                                                                                                                                |
| [Agent skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview)             | `/v1/messages`（带 `skills`）/ `/v1/skills`       | <Eligible status="no">否</Eligible>             | <Eligible status="no">否</Eligible> | Skill 数据按标准政策保留。请参阅 [Agent skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview#data-retention)。                                                                                                                                                                                                                                                                                                       |
| [Bash 工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/bash-tool)                     | `/v1/messages`（带 `bash` 工具）                    | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | 在您的环境中执行的客户端工具。                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)                          | `/v1/messages/batches`                         | <Eligible status="no">否</Eligible>             | <Eligible status="no">否</Eligible> | 29 天保留；需要异步存储。请参阅[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#data-retention)。                                                                                                                                                                                                                                                                                                                       |
| [浏览器使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)                | `/v1/messages`（带 `browser` 工具集）                | <Eligible>是</Eligible>                         | <Eligible status="no">否</Eligible> | 客户端工具。Anthropic 不运行浏览器操作，也不会在标准 API 处理之外保留页面内容。不在 HIPAA 就绪的涵盖范围内；包含浏览器使用工具的请求不会被阻止。请参阅[浏览器使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool#data-retention)。                                                                                                                                                                                                                                        |
| [缓存诊断](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics)                        | `/v1/messages`（带 `diagnostics`）                | <Eligible status="qualified">是（有条件）</Eligible> | <Eligible status="no">否</Eligible> | 您的提示和 Claude 的输出不会被存储。会短暂保留由加密哈希和令牌计数估算组成的指纹，以便与下一个请求进行比较。请参阅[缓存诊断](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics#data-retention)。                                                                                                                                                                                                                                                                         |
| [引用](https://platform.claude.com/docs/zh-CN/build-with-claude/citations)                                  | `/v1/messages`                                 | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             |                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)                   | `/v1/agents`、`/v1/sessions`、`/v1/environments` | <Eligible status="no">否</Eligible>             | <Eligible status="no">否</Eligible> | 会话是有状态资源；记录会一直保留，直到您将其删除。适用于所有 Managed Agents 子功能，包括[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)。                                                                                                                                                                                                                                                                                               |
| [代码执行](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)              | `/v1/messages`（带 `code_execution` 工具）          | <Eligible status="no">否</Eligible>             | <Eligible status="no">否</Eligible> | 容器数据最多保留 30 天。请参阅[代码执行](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#data-retention)。                                                                                                                                                                                                                                                                                                           |
| [计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)               | `/v1/messages`（带 `computer` 工具集或工具）            | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | 客户端工具，屏幕截图和文件在您的环境中捕获和存储，而非由 Anthropic 存储。请参阅[计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#data-retention)。                                                                                                                                                                                                                                                                                |
| [上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)                         | `/v1/messages`（带 `context_management`）         | <Eligible>是</Eligible>                         | <Eligible status="no">否</Eligible> | 上下文编辑（工具使用清除和思考清除）实时应用。                                                                                                                                                                                                                                                                                                                                                                                                                 |
| [上下文管理（压缩）](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)                          | `/v1/messages`（带 `context_management`）         | <Eligible>是</Eligible>                         | <Eligible status="no">否</Eligible> | 服务器端压缩结果通过 API 响应以无状态方式返回并往返传递。                                                                                                                                                                                                                                                                                                                                                                                                         |
| [数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)                               | `/v1/messages`（带 `inference_geo`）              | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             |                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)                                 | `/v1/messages`（带 `effort`）                     | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             |                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)                                | `/v1/messages`（带 `speed: "fast"`）              | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | 相同的 Messages API 端点，推理速度更快。无论速度设置如何，ZDR 均适用。                                                                                                                                                                                                                                                                                                                                                                                            |
| [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)                               | `/v1/files`                                    | <Eligible status="no">否</Eligible>             | <Eligible status="no">否</Eligible> | 文件会一直保留，直到被明确删除或达到其配置的过期时间。请参阅 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files#data-retention)。                                                                                                                                                                                                                                                                                                              |
| [细粒度工具流式传输](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/fine-grained-tool-streaming) | `/v1/messages`                                 | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             |                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)                          | `/v1/messages`（带 `mcp_servers`）                | <Eligible status="no">否</Eligible>             | <Eligible status="no">否</Eligible> | 数据按标准政策保留。请参阅 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#data-retention)。                                                                                                                                                                                                                                                                                                                          |
| [MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview)                    | `/v1/tunnels`                                  | <Eligible status="no">否</Eligible>             | <Eligible status="no">否</Eligible> | 研究预览。有关数据流边界和子处理者详情，请参阅 [MCP 隧道安全](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/security)。                                                                                                                                                                                                                                                                                                                       |
| [记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)                      | `/v1/messages`（带 `memory` 工具）                  | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | 客户端记忆存储，由您控制数据保留。                                                                                                                                                                                                                                                                                                                                                                                                                       |
| [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages)            | `/v1/messages`                                 | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | 用于生成 Claude 响应的标准 API 调用。                                                                                                                                                                                                                                                                                                                                                                                                               |
| [对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)     | `/v1/messages`（带 `role: "system"` 消息）          | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | Messages API 的请求形态能力；对话中途系统消息流经标准推理路径，响应之后服务器端不存储任何内容。                                                                                                                                                                                                                                                                                                                                                                                  |
| [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)                            | `/v1/messages`                                 | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | HIPAA 资格适用于通过 Messages API 内联发送的 PDF，而非通过 Files API 发送的 PDF。                                                                                                                                                                                                                                                                                                                                                                            |
| [程序化工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling)     | `/v1/messages`（带 `code_execution` 工具）          | <Eligible status="no">否</Eligible>             | <Eligible status="no">否</Eligible> | 基于代码执行容器构建；数据最多保留 30 天。请参阅[程序化工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling#data-retention)。                                                                                                                                                                                                                                                                                         |
| [提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)                           | `/v1/messages`                                 | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | 您的提示和 Claude 的输出不会被存储。KV 缓存表示和加密哈希在缓存 TTL 期间保存在内存中，并在过期后立即删除。请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#data-retention)。                                                                                                                                                                                                                                                                         |
| [搜索结果](https://platform.claude.com/docs/zh-CN/build-with-claude/search-results)                           | `/v1/messages`（带 `search_results` 来源）          | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             |                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)                      | `/v1/messages`                                 | <Eligible status="qualified">是（有条件）</Eligible> | <Eligible>是</Eligible>             | 您的提示和 Claude 的输出不会被存储。仅缓存 JSON schema，自上次使用起最多 24 小时。这也涵盖[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)（工具上的 `strict: true`），它使用相同的语法管道。JSON schema 定义中不得包含 PHI；请参阅 [PHI 处理指南](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#phi-handling-guidelines)。请参阅[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs#data-retention)。 |
| [文本编辑器工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/text-editor-tool)              | `/v1/messages`（带 `text_editor` 工具）             | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | 在您的环境中执行的客户端工具。                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)                                   | `/v1/messages`（带 `thinking`）                   | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             |                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)                           | `/v1/messages/count_tokens`                    | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | 在发送请求之前计算令牌数。                                                                                                                                                                                                                                                                                                                                                                                                                           |
| [工具搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)                 | `/v1/messages`（带 `tool_search` 工具）             | <Eligible>是</Eligible>                         | <Eligible status="no">否</Eligible> | 由 Anthropic 执行的服务器端工具；请求中的工具定义在每次调用时于内存中进行搜索，响应之后不存储任何内容。                                                                                                                                                                                                                                                                                                                                                                               |
| [网页抓取](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)                   | `/v1/messages`（带 `web_fetch` 工具）               | <Eligible>是</Eligible>                         | <Eligible status="no">否</Eligible> | 抓取的网页内容在 API 响应中返回。[动态过滤](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool#dynamic-filtering)不符合 ZDR 或 HIPAA 资格。网站发布者可能会根据其自身政策保留请求数据（例如抓取的 URL 和请求元数据）。                                                                                                                                                                                                                                                 |
| [网页搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)                  | `/v1/messages`（带 `web_search` 工具）              | <Eligible>是</Eligible>                         | <Eligible>是</Eligible>             | 实时网页搜索结果在 API 响应中返回。[动态过滤](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool#dynamic-filtering)不符合 ZDR 或 HIPAA 资格。                                                                                                                                                                                                                                                                                       |

## 无论何种安排均适用的保留

即使已有 ZDR 或 HIPAA 安排，Anthropic 仍可能在法律要求的情况下，或在数据被 Anthropic 的自动化信任与安全系统标记的情况下保留数据。因此，如果某个聊天或会话被标记，Anthropic 可能会将输入和输出保留最长 2 年。

## 常见问题

<AccordionGroup>
  <Accordion title="我如何知道我的组织是否有 ZDR 安排？">
    请查看您的合同条款或联系您的 Anthropic 客户代表，以确认您的组织是否已有 ZDR 安排。
  </Accordion>

  <Accordion title="我可以在我的 ZDR 安排下使用符合 ZDR 资格（有条件）的功能吗？">
    可以。这些功能保留的是一组最少的、有文档记录的技术数据，而不是您的提示或 Claude 的输出。有关"是（有条件）"的含义，请参阅[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)的图例；有关约束这些功能的承诺，请参阅 [Anthropic 如何处理数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#how-anthropic-approaches-data-retention)。
  </Accordion>

  <Accordion title="如果我在 ZDR 下使用标记为&#x22;否&#x22;的功能会怎样？">
    不会有任何机制阻止该请求。对于 ZDR 标记为"否"的功能本质上是有状态的：Batch API 存储您的作业，Files API 存储您的文件，代码执行在持久化容器中运行。这些功能的数据按该功能记录的政策保留。使用它们即表示您选择就该特定数据脱离您的 ZDR 安排。
  </Accordion>

  <Accordion title="我可以请求删除不符合 ZDR 资格的功能中的数据吗？">
    请联系您的 Anthropic 客户代表，讨论非 ZDR 功能的删除选项。
  </Accordion>

  <Accordion title="HIPAA 就绪与 ZDR 有何不同？">
    ZDR 防止客户数据在 API 响应返回后被静态存储。HIPAA 就绪涉及一套更广泛的隐私和安全保障措施，在 PHI 的整个生命周期内对其进行保护，包括加密、访问控制和审计日志记录。在 HIPAA 就绪下，数据可以在这些保障措施到位的情况下被保留，而不是要求立即删除。这两种安排涵盖不同的功能集；请参阅[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)。
  </Accordion>

  <Accordion title="如果我有 HIPAA 就绪，我还需要 ZDR 吗？">
    不需要。HIPAA 就绪的 API 访问被设计为处理 PHI 的组织的 ZDR 替代方案。启用 HIPAA 就绪后，您可以访问受支持的 API 功能，同时保持 HIPAA 所要求的隐私和安全保护。
  </Accordion>

  <Accordion title="如果我在 HIPAA 下使用不符合资格的功能会怎样？">
    API 会返回类型为 `invalid_request_error` 的 `400` 错误，但[功能资格表](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)中"详情"列注明不会被阻止的客户端工具除外（这些工具会被接受，但仍不在 HIPAA 就绪的涵盖范围内）。错误消息会指明哪些功能不可用。请从您的请求中移除这些功能并重试。请参阅 [HIPAA 错误处理](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#hipaa-error-handling)。
  </Accordion>

  <Accordion title="我可以将同一个组织用于 HIPAA 和非 HIPAA 工作负载吗？">
    不可以。HIPAA 就绪在组织级别强制执行，并会自动阻止不符合资格的功能（表中"详情"列注明的客户端工具是例外：它们不会被阻止，但仍不在 HIPAA 就绪的涵盖范围内）。对于不需要 HIPAA 就绪的工作负载，请使用单独的组织。
  </Accordion>

  <Accordion title="我如何申请 HIPAA 就绪的 API 访问？">
    符合资格的组织可以通过审阅并签署 Anthropic 的标准 BAA，直接在 [Claude Console > Settings > Privacy](https://platform.claude.com/settings/privacy) 中启用 HIPAA 就绪；请参阅 [HIPAA 就绪入门](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#getting-started-with-hipaa-readiness)。如果您的组织需要协商 BAA，或者您的组织无法使用自助启用，请联系 [Anthropic 销售团队](https://claude.com/contact-sales)。
  </Accordion>

  <Accordion title="这是否适用于 Amazon Bedrock 或 Google Cloud？">
    不适用。本页所述的 ZDR 和 HIPAA 安排适用于 Claude API，在该场景中 Anthropic 是数据处理者。在 Bedrock 和 Google Cloud 上，云提供商是数据处理者；请参阅这些平台的数据保留和合规政策以了解其对应的控制措施。
  </Accordion>

  <Accordion title="Claude Platform on AWS 是否符合 ZDR 或 HIPAA 就绪资格？">
    Claude Platform on AWS 遵循与第一方 Claude API 相同的数据保留政策。ZDR 可按需提供；请联系您的 Anthropic 客户代表以启用。Claude Platform on AWS 上不提供 HIPAA 就绪。详情请参阅 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)。
  </Accordion>

  <Accordion title="Claude Code 是否符合 ZDR 资格？">
    Claude Code 通过两种途径符合 ZDR 资格：

    * **API 密钥：** 与来自商业组织的按量付费 API 密钥一起使用的 Claude Code
    * **Claude Enterprise：** 通过已为组织启用 ZDR 的 Claude Enterprise 使用的 Claude Code

    ZDR 按组织启用。每个新组织都需要由您的客户团队单独启用 ZDR。ZDR 不会自动应用于同一账户下创建的新组织。

    此外，如果您在 Claude Code 中启用了指标日志记录，则生产力数据（例如使用统计）不受 ZDR 约束，可能会被保留。

    有关 Claude Enterprise 上 Claude Code 的 ZDR 完整详情，包括被禁用的功能以及如何申请启用，请参阅 [Claude Code ZDR 文档](https://code.claude.com/docs/en/zero-data-retention)。
  </Accordion>

  <Accordion title="Claude for Excel 是否支持 ZDR？">
    不支持，Claude for Excel 目前不符合 ZDR 资格。
  </Accordion>

  <Accordion title="我如何申请 ZDR？">
    要申请 ZDR 安排，请联系 [Anthropic 销售团队](https://claude.com/contact-sales)。
  </Accordion>
</AccordionGroup>

## 相关资源

* [隐私政策](https://www.anthropic.com/legal/privacy)
* [结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)
* [提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)
* [批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)
* [Files API 参考](https://platform.claude.com/docs/zh-CN/api/files/upload)
* [Trust Center](https://trust.anthropic.com/resources)
