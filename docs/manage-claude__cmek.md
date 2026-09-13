---
title: 客户管理的加密密钥
url: https://platform.claude.com/docs/zh-CN/manage-claude/cmek
description: 使用您控制的密钥对 Claude 工作区的静态数据进行加密。
---

```bash Learn more with the /claude-api skill in Claude Code
claude "/claude-api tell me about customer-managed encryption keys"
```

"Customer-managed encryption key"（客户管理的加密密钥），即 CMEK，允许您在自己的 [AWS KMS](https://aws.amazon.com/kms/)、[Google Cloud KMS](https://cloud.google.com/security/products/security-key-management) 或 [Azure Key Vault](https://azure.microsoft.com/en-us/products/key-vault) 中配置加密密钥，并让 Anthropic 使用它来加密某些工作区的静态数据。您保留对密钥的完全控制权，包括轮换、审计和撤销，并且 Anthropic 针对您的密钥执行的密钥操作会记录在您的云提供商的审计日志中。

CMEK 的使用是可选的。符合条件的组织可以**选择启用**客户管理的加密密钥，以替代 Anthropic 提供的默认加密。要激活 CMEK，请联系您的 Anthropic 客户团队。

<Warning>
  **启用 CMEK 是永久性的，并可能导致不可逆的数据丢失**

  启用 CMEK 是永久性的。Anthropic 不保留您密钥的任何副本，因此配置错误或密钥丢失可能会永久销毁您受 CMEK 保护的数据。如果您对任何步骤不确定，请在应用更改之前联系您的 Anthropic 代表。

  * **永久数据丢失：** 如果您的加密密钥被删除、被计划删除或其密钥材料被销毁，Anthropic 将无法恢复您的数据。
  * **标识符验证是强制性的：** 向错误或伪造的主体授予密钥访问权限可能会将您的数据暴露给未经授权的一方。请始终对照每个配置指南中发布的生产身份来验证 Anthropic 标识符。在 Claude Platform on AWS 上，该身份是 [AWS KMS 指南](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#claude-platform-on-aws)中发布的 AWS 服务主体。切勿信任通过电子邮件、聊天或任何入门引导渠道提供的标识符。
</Warning>

## 工作原理

只有组织管理员（在 Claude Platform 上；在 Claude Platform on AWS 上为 Admin 角色）或所有者和主要所有者（在 Claude Enterprise 上）可以配置 CMEK。在 Claude Platform 上，CMEK 的作用范围是每个工作区，并通过 Admin API 进行配置（在 Claude Platform on AWS 上，则在 Claude Console 中或通过 IAM 授权的外部密钥和工作区端点进行配置）。在 Claude Enterprise 上，CMEK 的作用范围是每个组织，并在 [claude.ai > 组织设置 > 数据和隐私](https://claude.ai/admin-settings/data-privacy-controls)中进行配置。在任一产品上，CMEK 保护的是您的密钥生效后写入的数据。现有数据（之前的聊天、文件和会话）仍使用 Anthropic 管理的密钥加密，不会使用您的密钥重新加密。

在 Claude Platform 上，Anthropic 建议在向新工作区发送任何请求之前将您的密钥附加到该工作区。如果您将密钥附加到已经在接收请求的工作区，您的密钥可能需要长达一天的时间才能生效。在此之前写入的数据与现有数据一样，使用 Anthropic 管理的密钥加密，不会重新加密。

CMEK 配置事件会出现在 [Compliance API 活动源](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)中。Anthropic 针对您的密钥执行的密钥操作（例如包装和解包数据密钥）不会出现在 Compliance API 中；它们会出现在您的云提供商的审计日志中。

Anthropic 从其标准公共 IP 范围调用您的密钥管理服务。如果您按 IP 限制对密钥管理服务的访问，请允许 [IP 地址](https://platform.claude.com/docs/zh-CN/api/ip-addresses)中列出的地址。在 Claude Platform on AWS 上，不要依赖基于 IP 的限制来保护您的密钥；请改用 [AWS KMS 指南](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#claude-platform-on-aws)中描述的密钥策略来限定访问范围。

## 前提条件

* 在将托管加密密钥的账户、项目或订阅中创建加密密钥和管理密钥访问的权限。
* 在 Claude Platform 上拥有 Claude Console 中的组织管理员角色（在 Claude Platform on AWS 上为 Admin 角色），或在 Claude Enterprise 上拥有所有者或主要所有者角色。
* 数据保留配置：对于 Claude Platform 和 Claude Enterprise，CMEK 均允许与[零数据保留（ZDR）](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)一起使用。

## 可用性和区域

除 Claude Platform on AWS（在本节末尾介绍）外，CMEK 目前仅在美国区域可用，所有加密操作均在美国区域处理。为获得最低的 "latency"（延迟），请选择靠近 Anthropic 美国基础设施的区域：

| 提供商          | 推荐区域                        |
| ------------ | --------------------------- |
| AWS          | `us-east-2`                 |
| Google Cloud | `us-central1`, `us-east5`   |
| Azure        | `northcentralus`, `eastus2` |

在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上，CMEK 仅支持 AWS KMS 密钥；无法注册 Google Cloud KMS 和 Azure Key Vault 密钥。上述区域建议在此不适用：密钥必须是与其所附加的工作区位于同一 AWS 账户和区域的单区域 KMS 密钥，并且其密钥策略必须向 AWS 服务主体而非 Anthropic 的 IAM 角色授予访问权限；请参阅[在 Claude Platform on AWS 上设置 CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#claude-platform-on-aws)。请在 Claude Console 中注册和附加密钥；外部密钥端点在 Claude Platform on AWS 上也可用，通过 [IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#encryption-keys)进行授权。没有单独的验证步骤：当您将密钥附加到工作区时，密钥会被隐式验证（附加调用会执行一轮加密/解密），因此密钥策略问题会在附加时而非注册时显现。

## CMEK 保护的内容

CMEK 涵盖的内容取决于您使用的产品。

### 使用 CMEK 密钥加密

**Claude Platform**

* 消息内容、文件和附件（包括随请求发送的内联附件和 Files API 上传的文件），以及 MCP 和工具配置。
* [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 数据，包括代理配置、环境、webhook，以及会话及其事件。

**Claude Enterprise**

* 聊天内容，包括技能、插件和 artifacts。
* 聊天附件和项目附件。
* CLI 上的 Claude Code，包括消息内容。
* Claude Desktop 中的 Cowork。
* 从用户机器上的会话捕获的 Compliance API [本地会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)。如果您的密钥无法使用，消息端点将返回 [503 Service Unavailable](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#local-sessions-temporarily-unavailable) 而非记录内容。会话元数据仍会列出。
* Office 代理。
* Claude in Chrome。

在两种产品上，备份和快照都会继承该密钥。

### 已禁用或已修改

启用 CMEK 后，某些功能会被关闭或大幅修改。此列表并非详尽无遗；请在启用 CMEK 之前与您的团队一起审阅。

**Claude Platform**

* Claude Console 中的 Playground 被禁用。
* Compliance API 中返回原始内容（例如提示、响应和文件）的部分被禁用。
* 其他测试版和研究预览功能可能不在 CMEK 的覆盖范围内。

**Claude Enterprise**

* 对话历史搜索被禁用。对话标题已加密，因此按标题或内容搜索不会返回任何结果。
* [项目知识搜索](https://support.claude.com/en/articles/11473015-retrieval-augmented-generation-rag-for-projects)（"retrieval-augmented generation"（检索增强生成），即 RAG）被禁用。项目知识会直接加载到每个对话的上下文中，而不是被索引和搜索。因此，与不使用 CMEK 相比，项目可使用的知识量可能大幅减少。超出可加载范围的知识将不会纳入对话。
* 某些分析功能会降级：claude.ai 技能和连接器的管理员分析（位于 claude.ai/analytics/usage 下以及通过 [Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api)）、Claude 智能报告（位于 claude.ai/analytics/insights 下）以及 Claude Code 贡献指标（位于 claude.ai/analytics/claude-code 下）。
* 审计日志导出被禁用。
* 用于临时文件交换的签名 URL 被禁用。这些 URL 支撑着 claude.ai 中的组织数据导出以及 Claude Code Remote 文件流程（例如屏幕截图更新）。

### 使用 Anthropic 密钥加密

这些功能仍然可用，但其数据不会使用您的密钥加密。您可以在**设置**中禁用任何不适合您使用场景的功能。

**Claude Platform**

* 非静态数据（例如缓存）以及 TTL 短于 24 小时的数据。
* 活动源、审计日志和遥测网络流量（例如 OTEL），以便客户即使在密钥被撤销的情况下也能保持合规。
* Claude Managed Agents [保管库凭据](https://platform.claude.com/docs/zh-CN/managed-agents/vaults)值，例如 OAuth 令牌和客户端密钥。这些值使用 Anthropic 管理的加密存储，仅可写入，且永远不会在 API 响应中返回。
* [用户配置文件](https://platform.claude.com/docs/zh-CN/api/beta/user_profiles)：`name`、`external_id` 和 `metadata` 字段使用 Anthropic 管理的加密存储，而非您的密钥。请勿在配置文件 `metadata` 中存储敏感个人数据。

**Claude Enterprise**

* Claude Code Desktop、网页版 Claude Code 以及 Claude in Slack。Anthropic 建议在管理控制台中禁用其中任何不适合您使用场景的功能。
* 测试版和研究预览功能可能不在 CMEK 的覆盖范围内，并且可能在 CMEK 组织中无法正常工作，例如 Claude Security 和 Claude Design。
* **设置** > **隐私**下的按需数据导出。
* [个人偏好 - Claude 指令部分](https://claude.ai/new#settings/general)以及 Cowork 全局指令。这些在账户级别设置，并在用户的所有组织之间共享。

在两种产品上，您组织中用户的账户数据（例如姓名、电子邮件地址和头像）不会使用您的密钥加密。

### 功能支持

启用 CMEK 后，以下 Claude Platform API 和工具会使用您的密钥存储静态数据：

| API                   | 工具和功能                                                 |
| --------------------- | ----------------------------------------------------- |
| Messages              | 网页搜索                                                  |
| Models                | 网页抓取                                                  |
| Files                 | 代码执行                                                  |
| Batch                 | Bash 工具                                               |
| Skills                | 文本编辑器工具                                               |
| Claude Managed Agents | MCP 连接器                                               |
|                       | 结构化输出（在 CMEK 组织中不适用于 Claude Fable 或 Claude Mythos 模型） |
|                       | Advisor 工具                                            |
|                       | 计算机使用                                                 |
|                       | 浏览器使用                                                 |
|                       | 上下文管理                                                 |

## 在您的密钥之外的有限保留

在三种狭义情况下，Anthropic 可能会使用 Anthropic 管理的加密保留特定记录：

* 法律要求 Anthropic 保留记录的情况（例如，根据 18 U.S.C. § 2258A 向 NCMEC 报告的材料）。
* 存在严重伤害的紧急风险（例如，CBRNE 武器开发、攻击性网络攻击或迫在眉睫的暴力威胁）。
* 违反 Anthropic [商业服务条款](https://www.anthropic.com/legal/commercial-terms)第 D.4 节或客户与 Anthropic 签订的其他适用协议中的同等条款。

除 [CSAM 筛查](https://support.claude.com/en/articles/9020328-csam-detection-and-reporting)外，保留需要人工审核员的明确决定，并遵循 Anthropic 的[商业数据保留政策](https://privacy.claude.com/en/articles/10023548-how-long-do-you-store-my-data)。对于每一次保留，都会生成相应的 [Compliance API 活动源](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)事件，其中包含传达保留目的的原因代码。详情请参阅 [CMEK 内容保留](https://platform.claude.com/docs/zh-CN/manage-claude/access-transparency#cmek-content-preservation)。安全筛查元数据（源自 Anthropic 自动安全扫描的记录，例如模式标识符和匹配指示符，而非对话内容）使用 Anthropic 管理的加密保留，并在密钥撤销后仍可读取。

## 限制

* **不可逆操作：** 一旦密钥附加到工作区，就无法分离或更换。在 Claude Platform 上，附加密钥还会锁定工作区的数据保留设置：您无法为该工作区关闭 30 天数据保留，而要恢复零数据保留则需要创建新工作区并将流量迁移到该工作区。在同一密钥内轮换密钥材料（例如 AWS KMS 自动轮换、Cloud KMS 轮换计划或 Azure Key Vault 轮换策略）受到透明支持，无需在 Anthropic 端进行任何更改。切换到*不同*的密钥需要使用新密钥创建新工作区并迁移您的数据。撤销或禁用密钥会使该工作区中所有受 CMEK 保护的数据永久无法访问，且没有回退路径。
* **无追溯加密：** CMEK 仅保护您的密钥生效后写入的数据（请参阅[工作原理](https://platform.claude.com/docs/zh-CN/manage-claude/cmek#how-it-works)）。
* **延迟：** 包装或解包数据密钥的操作需要往返您的密钥管理服务，这可能会给读取或写入静态数据的操作增加少量延迟。
* **撤销延迟：** 密钥撤销可能需要长达 1 小时（缓存 TTL）。在此窗口期间已在处理中的请求可能会继续成功。
* **KMS 费用：** CMEK 需要第三方密钥管理服务（AWS KMS、Google Cloud KMS 或 Azure Key Vault）中的密钥，这可能会产生由您的 KMS 提供商单独计费的费用。
* **网关后的 Claude Code 遥测：** 当 Claude Code 通过 LLM 网关或代理（自定义 `ANTHROPIC_BASE_URL`）连接时，CMEK 不适用于 Claude Code 的运行遥测。要关闭此遥测，请将 `DISABLE_TELEMETRY` 环境变量设置为 `1`，如 Claude Code 文档中[遥测服务](https://code.claude.com/docs/en/data-usage#telemetry-services)下所述。

## 配置您的提供商

请按照您所使用的密钥管理服务的指南进行操作。

<CardGroup cols={3}>
  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms" title="AWS KMS">
    创建一个 AWS KMS 密钥，其密钥策略向 Anthropic 授予访问权限，然后注册该密钥。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/cmek-google-cloud-kms" title="Google Cloud KMS">
    创建一个 Cloud KMS 加密密钥，向 Anthropic 的服务账户授予访问权限，然后注册该密钥。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/cmek-azure-key-vault" title="Azure Key Vault">
    创建一个 RSA 密钥，向 Anthropic 服务主体授予访问权限，然后注册并验证该密钥。
  </Card>
</CardGroup>
