---
title: Compliance API
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api
description: 以编程方式访问您组织的 Claude 活动、聊天、文件、项目、Claude 应用中的会话以及用户，用于合规、审计和治理。
---

Compliance API（合规 API）为 Claude Enterprise 和 Claude Console 客户提供对其组织 Activity Feed（活动源）的编程访问。对于 Claude Enterprise 组织，它还涵盖每个关联组织中的用户、角色和组目录；每个组织当前生效的有效设置；claude.ai 组织中的底层聊天、文件和项目；以及 Cowork、Claude Code、Claude Science 和 Claude for Microsoft 365 会话。安全、法务和合规团队使用它来审计活动、检索或删除内容，并将事件输送到下游工具中。

<Note>
  有两种密钥类型可以解锁 Compliance API。**Compliance Access Key**（合规访问密钥，在 claude.ai 中创建）可以访问所有端点，而 **Admin API key**（管理员 API 密钥，在 Claude Console 中创建）只能访问 Activity Feed。有关完整的密钥类型比较，请参阅[您需要哪种密钥？](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#which-key-do-you-need)。
</Note>

以下调用返回您组织中最近的活动事件。任何具有 `read:compliance_activities` 作用域的密钥都可以发起此调用。要创建密钥并授予其该作用域，请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。

```bash cURL
curl --fail-with-body -sS \
  "https://api.anthropic.com/v1/compliance/activities?limit=1" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY"
```

成功的响应会返回一个 JSON 对象，其中包含 `data`（一个 `Activity` 记录数组）、`has_more`、`first_id` 和 `last_id`：

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

***

## Compliance API 的工作原理

每个端点都位于 `https://api.anthropic.com` 上的 `/v1/compliance/*` 下，并通过 `x-api-key` 标头进行身份验证。要配置密钥，请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。

Activity Feed（`GET /v1/compliance/activities`）可供任何携带 `read:compliance_activities` 作用域的密钥使用；有关筛选器、分页和完整的 `Activity` 对象，请参阅[查询 Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)。其余端点需要携带相关作用域的 Compliance Access Key。

一个 Claude Enterprise 租户拥有一个父组织（集中管理身份的顶级容器），以及两种类型的关联组织：claude.ai 组织（用户在其中聊天和存储内容）和 Claude Console 组织（用户在其中管理 Claude API 工作负载）。对于覆盖父组织的密钥，目录端点（组织、用户、角色和组）会返回来自任一类型的每个关联组织的数据。内容端点（聊天、文件、项目、项目附件和会话）仅提供 Claude Enterprise 数据。聊天、文件和项目端点返回 claude.ai 的聊天、文件和项目。会话端点返回用户机器上的 Cowork、Claude Code、Claude Science 和 Claude for Microsoft 365 会话（本地会话）的记录，这些记录是在用户使用其 Claude Enterprise 账户登录时捕获的。它们还返回在 claude.ai 网页端或移动端启动的 Cowork 会话的记录，这些会话在 Anthropic 管理的环境中于云端运行（远程会话）。独立的 Claude Console 组织（没有父组织的组织）不属于 Claude Enterprise 租户；它使用 Admin API key，并且只能查询 Activity Feed。

所有 `/v1/compliance/*` 端点共享每个父组织每分钟 600 个请求的 "rate limit"（速率限制）（对于独立的 Claude Console 组织，则按每个组织计算）。本地会话端点仅计入该共享限制，而远程会话端点在此基础上还有第二个请求预算。有关响应标头和重试约定，请参阅 [429 Too Many Requests](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#429-too-many-requests)。

***

## Compliance API 与相关功能的比较

有几个相邻功能与 Compliance API 存在重叠；以下是如何选择。

### 导出审计日志

审计日志导出是 [claude.ai > Organization settings > Data and privacy](https://claude.ai/admin-settings/data-privacy-controls) 中的一项独立功能，允许所有者和主要所有者下载组织事件的 CSV 文件。它比 Compliance API 的范围窄得多：回溯窗口有上限、仅支持 CSV 下载，并且无法访问聊天、文件或项目内容。对于持续的编程使用，请统一采用 Compliance API。

### Analytics API

Anthropic 提供两个分析 API：Claude Enterprise Analytics API 和 [Claude Code Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/claude-code-analytics-api)。两者都为 IT、FinOps 和平台团队返回汇总的使用量和成本数据，而 Compliance API 则为安全、法务和合规团队返回逐事件记录。这两个 API 系列回答不同的问题，使用不同的密钥，并且分别配置。

### OpenTelemetry 日志记录

[Cowork 的 OpenTelemetry 日志记录](https://support.claude.com/en/articles/14477985-monitor-claude-cowork-activity-with-opentelemetry)和 [Claude Code 监控](https://code.claude.com/docs/en/monitoring-usage)会在活动发生时将逐事件遥测数据（包括令牌、成本和主机元数据）流式传输到您运行的收集器，而 Compliance API 则按请求从 Anthropic 返回保留的逐会话记录，并可与您现有的 Compliance Access Key 配合使用。OpenTelemetry 日志记录也可以捕获提示和响应，但 Anthropic 建议使用 Compliance API 来检索 Cowork 和 Claude Code 会话的内容。有关比较本地会话、远程会话和 OpenTelemetry 日志记录的表格，请参阅[检索会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions)的简介。

### 推理钩子

[推理钩子](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks)（测试版）以内联方式运行：您组织的 AI 安全服务器会在推理之前接收每个受管控的提示，并可以实时拒绝它，而 Compliance API 则在事后检索记录并返回更丰富的数据，例如组织设置和完整的非文本文件。

***

## 本节内容

<CardGroup>
  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access" title="设置 Compliance API">
    为您的组织启用 Compliance API，然后创建 Compliance Access Key（具有限定作用域的权限）或 Admin API key，并了解应使用哪一种。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed" title="查询 Activity Feed">
    检索、筛选和分页浏览共享的 Activity Feed。两种密钥类型均支持。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data" title="检索和删除聊天、文件和项目">
    读取聊天内容、文件和项目附件；按需删除聊天、文件和项目。需要 Compliance Access Key。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions" title="检索会话记录">
    列出您的用户在 Claude 应用和代理（例如 Cowork 和 Claude Code）中运行的会话，并检索其记录。需要 Compliance Access Key。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data" title="列出组织、用户、角色、组和设置">
    枚举关联组织、成员、角色和目录组，并读取每个组织的有效设置。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-integration-patterns" title="设计您的合规集成">
    选择活动源消费模式，规划 SIEM 关联，并确定您的保留策略。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors" title="处理 Compliance API 错误">
    Compliance API 返回的每个 400、401、403、404、409、429 和 5xx 响应，以及各自的修复方法。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/api/compliance" title="API 参考">
    每个 Compliance API 调用的端点路径、参数和响应架构。
  </Card>

  <Card href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-faq" title="Compliance API 常见问题">
    有关密钥、作用域、可用性和集成的常见问题解答。
  </Card>
</CardGroup>
