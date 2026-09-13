---
title: 分析 API
url: https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api
description: 了解您的组织需要哪种分析 API 和 API 密钥，然后配置对 Claude Code 生产力指标或 Claude Enterprise 参与度和采用数据的访问权限。
---

Anthropic 提供两种分析 API，您使用哪一种取决于您的组织管理的是哪种 Claude 产品：

* **Claude Code Analytics API** 为使用 Claude Platform 的组织报告每日 Claude Code 生产力指标。它是 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 的一部分，使用 Admin API 密钥。
* **Claude Enterprise Analytics API** 为 Claude Enterprise 组织报告跨 Claude 产品（聊天、项目、Claude Code 等）的全组织范围的参与度、采用情况和成本数据。它使用在 claude.ai 中创建的 Analytics API 密钥。

这两种 API 使用不同的密钥类型，由不同的角色在不同的位置创建。本页介绍哪种 API 适合您的组织，以及如何创建正确的密钥。

## 您需要哪种 API？

| API                                 | 密钥类型                               | 创建位置                                                                                      | 谁可以创建 | 涵盖内容                                                     |
| ----------------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------- | ----- | -------------------------------------------------------- |
| **Claude Code Analytics API**       | Admin API 密钥（`sk-ant-admin01-...`） | [Claude Console > Settings > Admin keys](https://platform.claude.com/settings/admin-keys) | 组织管理员 | 每位用户的每日 Claude Code 指标：会话、代码行数、提交、拉取请求、工具接受情况，以及按模型估算的成本 |
| **Claude Enterprise Analytics API** | Analytics API 密钥                   | [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access)    | 主要所有者 | 全组织范围的参与度和采用情况（用户活动、活跃用户摘要、项目、技能和连接器使用情况），以及成本和使用报告      |

这些密钥类型不可互换：Admin API 密钥无法调用 Claude Enterprise Analytics API，Analytics API 密钥也无法调用 Admin API。两种 API 都出现在 [Admin API 参考](https://platform.claude.com/docs/zh-CN/api/admin)下，但它们是具有各自密钥类型的独立 API。如果您的组织同时使用 Claude Platform 和 Claude Enterprise，您可以配置两种密钥，并使用每种 API 获取其各自的数据。

<Note>
  您在寻找 API 使用量和成本数据而非产品分析数据？请参阅 [Usage and Cost API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api)，其中说明了适用于 Claude Console 和 Claude Enterprise 组织的正确途径。
</Note>

<Note>
  如果您希望在产品中而非以编程方式查看参与度和采用数据，请使用 claude.ai 中的[分析仪表板](https://claude.ai/analytics/activity)。对于治理和审计用例（单个用户操作、原始活动事件、对话内容），请参阅 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。
</Note>

## 获取 Claude Code Analytics API 的访问权限

Claude Code Analytics API 对所有有权访问 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 的组织开放，并且可免费使用。

<Steps>
  <Step title="创建 Admin API 密钥">
    按照[创建 Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys#create-a-key-for-a-claude-console-organization)中的步骤操作。
  </Step>

  <Step title="调用 API">
    在 `x-api-key` 标头中传递密钥：

    ```bash
    curl "https://api.anthropic.com/v1/organizations/usage_report/claude_code?starting_at=2025-09-08" \
      --header "anthropic-version: 2023-06-01" \
      --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
    ```
  </Step>
</Steps>

有关可用指标、请求参数和响应架构，请参阅 [Claude Code Analytics API 指南](https://platform.claude.com/docs/zh-CN/manage-claude/claude-code-analytics-api)和 [API 参考](https://platform.claude.com/docs/zh-CN/api/admin/usage_report/retrieve_claude_code)。

## 获取 Claude Enterprise Analytics API 的访问权限

Claude Enterprise Analytics API 对 Claude Enterprise 组织开放。参与度和采用数据在所有 Enterprise 计划中均可用。成本和使用端点适用于基于用量的 Enterprise 计划；对于基于席位的 Enterprise 计划，它们仅反映使用额度。

<Steps>
  <Step title="以主要所有者身份登录">
    只有组织的主要所有者才能启用 API 访问并创建 Analytics API 密钥。
  </Step>

  <Step title="启用 API 访问并创建密钥">
    前往 [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access) 并启用公共 API 访问，然后创建 Analytics API 密钥。密钥具有 `read:analytics` 作用域。复制显示的密钥并将其存储在您的密钥管理器中。
  </Step>

  <Step title="调用 API">
    在 `x-api-key` 标头中传递密钥。端点位于 `https://api.anthropic.com/v1/organizations/analytics/` 下。有关请求示例、参数和响应架构，请参阅 [Claude Enterprise Analytics API 参考](https://platform.claude.com/docs/zh-CN/api/admin/analytics)。
  </Step>
</Steps>

Claude Enterprise Analytics API 提供：

* **用户活动：** 每位用户的每日指标，涵盖聊天（对话、消息、项目、文件、artifacts）、Claude Code（会话、提交、拉取请求、代码行数、工具操作）以及其他 Claude 产品
* **活动摘要：** 组织级别的每日、每周和每月活跃用户、席位数量和待处理邀请
* **项目、技能和连接器使用情况：** 聊天项目、技能和连接器的采用情况细分
* **成本和使用报告：** 每位用户和组织级别随时间变化的令牌使用量和成本（基于用量的 Enterprise 计划）

有关端点详情、参数和响应架构，请参阅 [Claude Enterprise Analytics API 参考](https://platform.claude.com/docs/zh-CN/api/admin/analytics)。以下各节介绍适用于这些端点的数据新鲜度、指标定义和操作指南。

## 数据可用性和新鲜度

Claude Enterprise Analytics API 数据适用于 2026 年 1 月 1 日及之后的日期。

**参与度和采用端点**（用户活动、摘要、项目、技能、连接器）返回您指定日期的每日快照。某一天的数据会在次日 10:00 UTC 进行聚合，通常有 1 天的延迟。确切的新鲜度因查询而异，因此不要假设固定的延迟，而应检查错误响应：请求尚不可用的日期会返回 400 错误，并指明最近可用的日期。如果数据在远超典型延迟后仍不可用，通常表明 Anthropic 一侧的数据管道出现故障；如果缺口持续存在，请联系支持团队。

**成本和使用端点**遵循不同的新鲜度模型。数据通常在底层使用发生后四小时内可用，但可能需要长达 24 小时。随着延迟事件到达和对账运行，某一日期的数值可能在最多 30 天内被修订。如需发票级别的总计，请查询至少 30 天前的日期。

<Note>
  成本和使用响应包含 `data_refreshed_at` 时间戳。当省略 `ending_at`（默认为当前时间）时，响应会包含 `data_refreshed_at` 之后的一段不完整的尾部数据。为了在重复调用中获得稳定的结果，请将 `ending_at` 设置为等于或早于先前返回的 `data_refreshed_at` 的值。
</Note>

## 指标的定义方式

**活跃用户。** 如果满足以下任一条件，则用户在某一天被计为活跃：他们在 Claude 中发送了至少一条聊天消息；他们有至少一个与您的 Claude Enterprise 组织关联的、包含工具使用或 git 活动的 Claude Code 会话（本地或远程）；或者他们有至少一个包含工具使用或消息活动的 Cowork 会话。

**按产品划分的指标块。** 按产品划分的指标对象（例如用户活动记录上的 Office Agent 或 Cowork 指标）始终存在于每条记录中。未使用该产品的组织会看到全零值，而不是 `null`。

**连接器名称。** 连接器名称会跨来源进行规范化。例如，`Atlassian MCP server`、`mcp-atlassian` 和 `atlassian_MCP` 在连接器使用端点中都显示为 `atlassian`。

## 使用 API

**分页游标与发出它们的查询绑定。** 在成本和使用端点上，不要在序列中途更改查询参数：如果您更改 `products[]`、`group_by[]`、`order_by`、日期范围或任何筛选条件并传递旧游标，请求将返回 400 错误。要更改参数，请在不带游标的情况下从第一页重新开始。

**列表参数使用方括号表示法。** 为每个值重复该参数，例如 `products[]=chat&products[]=claude_code`。

**金额字段是以美分为单位的十进制字符串。** 货币金额以十进制字符串形式返回，例如 `"41280.000000"`（表示 $412.80）。要转换为美元，请将其解析为十进制数并除以 100。对于可能超过数百万美元的数值，请避免使用二进制浮点解析。

**速率限制在组织级别应用**，而非按密钥应用，此 API 中所有端点的默认限制为每分钟 60 个请求。如果这不足以满足您的用例，请联系您的 Anthropic 客户团队讨论调整限制。

## 已知限制

如果您的组织通过 Amazon Bedrock 使用 Claude Code，Claude Enterprise Analytics API 不会返回该使用情况的 Claude Code 活动。

## 后续步骤

<CardGroup cols={2}>
  <Card title="Claude Code Analytics API" href="https://platform.claude.com/docs/zh-CN/manage-claude/claude-code-analytics-api">
    使用 Admin API 密钥跟踪 Claude Code 会话、代码更改和工具使用情况。
  </Card>

  <Card title="Usage and Cost API" href="https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api">
    跟踪您组织的 API 令牌使用量和成本。
  </Card>

  <Card title="Claude Enterprise Analytics API 参考" href="https://platform.claude.com/docs/zh-CN/api/admin/analytics">
    参与度、采用情况和成本数据的端点参考。
  </Card>

  <Card title="设置 Compliance API" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access">
    审计和合规数据使用其自己的密钥类型。
  </Card>
</CardGroup>
