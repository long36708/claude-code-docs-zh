---
title: 创建 Admin API 密钥
url: https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys
description: 为您的 Claude Console 或 Claude Enterprise 组织创建 Admin API 密钥。
---

Admin API 密钥用于对本指南 **Admin** 部分中的每个 API 进行身份验证：[Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)、[Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api)、[Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api)、[Spend Limits API](https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api)、[Usage and Cost API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api) 以及 [Rate Limits API](https://platform.claude.com/docs/zh-CN/manage-claude/rate-limits-api)。您无需为每个 API 单独创建密钥。唯一的例外是 Admin API 的服务账户（service-account）、联合身份颁发者（federation-issuer）和联合身份规则（federation-rule）端点，它们仅接受具有 `org:admin` 作用域的 OAuth bearer 令牌。请参阅[获取 OAuth bearer 令牌](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#oauth-bearer-token)。

在何处创建密钥取决于您的组织使用的是哪款 Claude 产品。

## 您需要哪种密钥？

| 您的组织                                                      | 创建密钥的位置                                                                                   | 密钥前缀                 | 谁可以创建                                                                                                         | 适用于                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Claude Console**（Claude Platform，`platform.claude.com`） | [Claude Console > Settings > Admin keys](https://platform.claude.com/settings/admin-keys) | `sk-ant-admin01-...` | 具有 **admin** 角色的组织成员                                                                                          | [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)、[Usage and Cost API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api)、[Rate Limits API](https://platform.claude.com/docs/zh-CN/manage-claude/rate-limits-api)、[Claude Code Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/claude-code-analytics-api)，以及 Compliance API 的 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)                               |
| **Claude Enterprise**（`claude.ai`）                        | [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access)    | `sk-ant-api01-...`   | 父组织的**主要所有者（primary owner）**（所有关联组织）。\*\*组织所有者（organization owner）\*\*可以创建仅携带 Compliance API 作用域、且仅限于其自身组织的密钥 | [用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management)（Admin API 的成员、邀请和群组端点）、[Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api)、[Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api) 以及 [Spend Limits API](https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api)，具体取决于您选择的[作用域](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys#choose-scopes-for-a-claude-enterprise-key) |

在一个组织中创建的密钥不能用于管理另一个组织。如果您的公司同时使用 Claude Console 和 Claude Enterprise，请在每个组织中各创建一个密钥。

## 为 Claude Console 组织创建密钥

<Steps>
  <Step title="以组织管理员身份登录">
    只有具有 **admin** 角色的组织成员才能创建 Admin API 密钥。请参阅[组织角色和权限](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#organization-roles-and-permissions)。
  </Step>

  <Step title="打开 Admin keys 设置">
    前往 [Claude Console > Settings > Admin keys](https://platform.claude.com/settings/admin-keys)。
  </Step>

  <Step title="创建密钥">
    点击 **Create key**，为其命名，选择[密钥过期时间](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-expiration)，然后点击 **Create**。Claude Console 密钥没有可选择的作用域；每个密钥都拥有对所有接受 Admin API 密钥的端点的完全访问权限（本页顶部提到的服务账户和联合身份端点不接受 Admin API 密钥）。
  </Step>

  <Step title="复制并存储密钥">
    复制显示的密钥（以 `sk-ant-admin01-` 开头）并将其存储在您的密钥管理器中。完整密钥仅显示一次。
  </Step>
</Steps>

## 为 Claude Enterprise 组织创建密钥

<Steps>
  <Step title="以主要所有者或组织所有者身份登录">
    Claude Enterprise 父组织的**主要所有者**可以创建能够访问每个关联组织的密钥，或仅限于单个组织的密钥。**组织所有者**可以创建仅具有 Compliance API 作用域、且仅限于其自身组织的密钥。
  </Step>

  <Step title="打开 API 设置">
    前往 [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access) 并找到 **Keys** 部分。
  </Step>

  <Step title="点击 + Create key">
    为密钥命名，并从[作用域表](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys#choose-scopes-for-a-claude-enterprise-key)中选择您需要的作用域。主要所有者可以在单个密钥上组合来自不同 API 的作用域（例如 `read:analytics` 和 `read:spend_limits`）。
  </Step>

  <Step title="复制并存储密钥">
    复制显示的密钥（以 `sk-ant-api01-` 开头）并将其存储在您的密钥管理器中。完整密钥仅显示一次。
  </Step>
</Steps>

## 为 Claude Enterprise 密钥选择作用域

创建 Claude Enterprise 密钥时，请选择您计划调用的 API 所需的每个作用域（scope）。作用域在创建时即固定；若要稍后添加作用域，请创建新密钥。

| 要调用……                                                                                                                                                                                                                                                                                                                                | 选择这些作用域                       |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------- |
| Admin API [用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management)：列出和查找成员及邀请；读取自定义角色及其权限                                                                                                                                                                                                                        | `read:members`                |
| Admin API [用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management)：更改成员角色、移除成员、创建和撤回邀请                                                                                                                                                                                                                           | `write:members`               |
| Admin API [用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management)：读取群组及其成员                                                                                                                                                                                                                                      | `read:rbac_groups`            |
| Admin API [用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management)：创建、重命名和删除群组；添加和移除群组成员；在创建邀请时分配群组                                                                                                                                                                                                              | `write:rbac_groups`           |
| [Spend Limits API](https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api)：读取成员的有效支出限额和提额请求                                                                                                                                                                                                                           | `read:spend_limits`           |
| [Spend Limits API](https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api)：设置或清除每用户支出限额；批准或拒绝提额请求                                                                                                                                                                                                                     | `write:spend_limits`          |
| [Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api)：参与度、采用率、成本和使用情况报告                                                                                                                                                                                                              | `read:analytics`              |
| [Compliance API Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)：组织范围的活动事件                                                                                                                                                                                                              | `read:compliance_activities`  |
| [Compliance API 聊天、文件和项目端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data)以及 [Compliance API 会话端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions)：读取聊天、文件、项目、会话记录和[组织用户](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data#list-organization-users) | `read:compliance_user_data`   |
| [Compliance API 聊天、文件和项目端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data)：删除聊天、文件和项目                                                                                                                                                                                                                 | `delete:compliance_user_data` |
| [Compliance API 组织端点](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-org-data)：读取组织元数据和有效设置                                                                                                                                                                                                                         | `read:compliance_org_data`    |
| Admin API [用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management)读取端点以及每个 Compliance API 读取端点，使用单一只读作用域（用于安全审计集成；不包括 Spend Limits 或 Analytics API）                                                                                                                                                              | `read:org_audit`              |

必须先为您的组织启用 Compliance 和 Analytics API，才能使用具有这些作用域的密钥。请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api) 和[获取 Claude Enterprise Analytics API 的访问权限](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api#get-access-to-the-claude-enterprise-analytics-api)。

## 使用密钥

在每个请求的 `x-api-key` 标头中传递密钥。有关完整的请求示例，请参阅各 API 的文档。

超出密钥作用域的调用会返回 `403 Forbidden`，并附带一条消息，列出该密钥拥有的作用域以及该端点所需的作用域。

## 后续步骤

<CardGroup cols={2}>
  <Card title="Admin API" href="https://platform.claude.com/docs/zh-CN/manage-claude/admin-api">
    管理组织成员、工作区和 API 密钥。
  </Card>

  <Card title="Spend Limits API" href="https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api">
    为您的 Claude Enterprise 组织设置每成员支出限额并审核提额请求。
  </Card>

  <Card title="Analytics API" href="https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api">
    报告 Claude Code 生产力或 Claude Enterprise 参与度和采用率。
  </Card>

  <Card title="Compliance API" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api">
    审计活动并检索或删除整个组织中的用户内容。
  </Card>
</CardGroup>
