---
title: 设置 Compliance API
url: https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access
description: 为您的组织启用 Compliance API，然后创建 Compliance Access Key（具有限定范围的权限）或 Admin API 密钥，并了解应使用哪一种。
---

<Note>
  Claude Enterprise 组织和符合条件的独立 Claude Console 组织可以自助访问 Compliance API。本页介绍如何为您的组织启用 Compliance API 并创建 API 密钥。
</Note>

<Check>
  **所需角色：** 组织管理员（Claude Console），或主要所有者或组织所有者（claude.ai）。
</Check>

Compliance API 使用两种密钥类型，您创建哪一种取决于您的组织使用哪种 Claude 产品。主要所有者和组织所有者在 claude.ai 中创建 "Compliance Access Key"（合规访问密钥）；这些密钥可解锁完整的 Compliance API。主要所有者的密钥可以覆盖父组织下的每个组织；组织所有者的密钥仅覆盖其自己的组织。组织管理员在 Claude Console 中创建 "Admin API key"（管理 API 密钥）；这些密钥仅解锁 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)（活动源）。

## 您需要哪种密钥？

| 密钥类型                                           | 创建位置                                                                                      | 用途                                                                                                                       | 是否适用于 Compliance API？ |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| **Compliance Access Key** (`sk-ant-api01-...`) | [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access)    | Activity Feed、聊天、文件、项目、会话（在 Cowork 和 Claude Code 等应用中）、用户、组织元数据和组织设置                                                     | 是（所有端点）               |
| **Admin API 密钥** (`sk-ant-admin01-...`)        | [Claude Console > Settings > Admin keys](https://platform.claude.com/settings/admin-keys) | [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 和 Compliance API Activity Feed               | 仅 Activity Feed       |
| **Analytics API 密钥**                           | [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access)    | Claude Enterprise Analytics API（请参阅 [Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api)） | 否                     |
| **Claude API 密钥** (`sk-ant-api03-...`)         | [Claude Console > Settings > API keys](https://platform.claude.com/settings/keys)         | 通过 [Claude API](https://platform.claude.com/docs/zh-CN/api/overview) 调用 Claude 模型                                        | 否                     |

一个 Claude Enterprise 租户有一个**父组织**（parent organization），它为其下的每个工作负载组织集中管理身份、SSO 和 SCIM。这些工作负载组织是父组织的**关联组织**（linked organizations）。

<Warning>
  **Claude Enterprise 父组织不会出现在 Claude Console（`platform.claude.com`）中。** 父组织不承载任何工作负载，没有 Claude API 密钥，也没有 Admin API 密钥。请在 claude.ai 的 **Organization settings** 中创建 Compliance Access Key，而不是在 Claude Console 中。
</Warning>

## 设置 Compliance API

设置是一个流程：为您的组织启用 Compliance API，然后在 claude.ai 中创建 Compliance Access Key。Claude Console 组织则在启用后[创建 Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#create-an-admin-api-key)；Admin API 密钥仅能访问 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)。

<Warning>
  具有 `read:compliance_user_data` 的 Compliance Access Key 可以读取每个关联组织中的每个聊天、 文件、项目和会话记录，包括主要所有者未曾看过的内容。具有 `delete:compliance_user_data` 的密钥可以永久删除聊天、文件和 项目。请像对待生产数据库凭据一样对待 Compliance Access Key： 将它们存储在密钥管理器中，切勿存储在源代码控制或 SIEM 转发器配置中。
</Warning>

<Steps>
  <Step title="启用 Compliance API">
    在何处启用 Compliance API 取决于您的组织的设置方式：

    * **Claude Enterprise 组织：** 主要所有者在 [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access) 启用 Compliance API。启用在父组织级别进行，并级联到每个关联组织，包括 claude.ai 和 Claude Console。
    * **独立 Claude Console 组织：** 组织管理员在 [Claude Console > Settings > Security](https://platform.claude.com/settings/security) 打开 **Compliance API** 开关。符合条件的组织可自助启用，更改立即生效。如果 **Compliance API** 部分不可见，则说明您没有管理员角色，或者您的组织已关联到父组织（此时应从父组织启用 Compliance API），或者您的组织不符合自助启用的条件；如果您不确定属于哪种情况，请联系您的客户团队或 [Anthropic 支持](https://support.claude.com)。
    * **关联到父组织的 Claude Console 组织：** 在 Claude Console 中无需打开任何设置。请让您父组织的主要所有者在 claude.ai 中启用 Compliance API，或联系您的客户团队。

    <Warning>
      **关闭 Compliance API 会停止活动记录。** 组织管理员可以随时使用打开 Compliance API 的同一个 **Compliance API** 开关将其关闭。在 Compliance API 关闭期间，不会为您的组织记录任何活动事件，因此 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed) 不会收到新事件。如果您的组织已加入 [Access Transparency](https://platform.claude.com/docs/zh-CN/manage-claude/access-transparency)，关闭 Compliance API 也会停止 Access Transparency 事件的传送。Compliance API 关闭期间未记录的活动以后无法恢复。重新打开 Compliance API 会从该时间点起恢复记录；已记录的活动不会被删除。对于 Claude Enterprise 组织，claude.ai 中的 Compliance API 设置还控制[本地会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)（用户机器上的会话）的记录捕获：捕获在 Compliance API 启用时开始，在其关闭期间停止，并且在其关闭期间运行的会话的记录内容不会被捕获，以后也无法恢复。
    </Warning>

    独立 Claude Console 组织使用 Admin API 密钥而非 Compliance Access Key：启用后，请跳过其余步骤，改为[创建新的 Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#create-an-admin-api-key)。其余步骤用于配置 Compliance Access Key，这些密钥仅适用于属于 Claude Enterprise 租户的组织。
  </Step>

  <Step title="确定密钥的范围">
    密钥的访问权限在创建时设定。请确定密钥覆盖哪些组织：

    * **父组织**的密钥可以访问父组织下的每个组织。
    * **单个组织**的密钥只能访问该组织。
  </Step>

  <Step title="使用匹配的角色登录">
    登录 claude.ai。父组织的主要所有者可以创建任一范围的密钥。组织所有者只能创建限定于其自己组织的密钥。

    如果下一步中描述的 **API** 页面不可见，或者创建密钥时合规范围不可用，则说明您的角色无法创建 Compliance Access Key，或者您的组织尚未启用 Compliance API（请返回第一步）。
  </Step>

  <Step title="打开 API 设置">
    前往 [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access) 并找到 **Keys** 部分。
  </Step>

  <Step title="创建密钥">
    点击 **Create key**，为密钥命名，并从下表中选择一个或多个范围。点击 **Create**。

    | 范围                            | 授予权限                                                                            |
    | ----------------------------- | ------------------------------------------------------------------------------- |
    | `read:compliance_activities`  | 读取 Activity Feed。覆盖父组织的密钥可读取父组织及所有关联组织的事件。                                      |
    | `read:compliance_user_data`   | 读取用户聊天、消息、文件、项目、会话元数据和记录、组织用户以及群组成员                                             |
    | `delete:compliance_user_data` | 删除用户聊天、文件和项目                                                                    |
    | `read:compliance_org_data`    | 读取组织元数据（名称、类型、角色和群组）以及父组织下各组织当前生效的设置。用户列表和群组成员资格需要 `read:compliance_user_data`。 |

    选择您的集成所需的最小范围集：

    * 仅读取 Activity Feed 的审计管道只需要 `read:compliance_activities`。
    * 读取聊天和文件但从不删除它们的 eDiscovery 工具不需要 `delete:compliance_user_data`。
    * 如果您的工作流既读取又删除，请使用具有不同范围的**两个密钥**，这样泄露的读取密钥就无法删除数据。

    Compliance Access Key 的范围在创建后不可更改。要更改范围，请创建一个具有所需范围的新密钥，然后删除旧密钥。
  </Step>

  <Step title="复制并存储密钥">
    复制显示的密钥（以 `sk-ant-api01-` 开头）并将其存储在您的密钥管理器中。完整密钥仅显示一次。
  </Step>

  <Step title="导出密钥以用于本指南中的示例">
    将密钥设置为环境变量，以便本指南中的 shell 示例可以读取它：

    ```bash
    export ANTHROPIC_COMPLIANCE_ACCESS_KEY=sk-ant-api01-...
    ```
  </Step>
</Steps>

## 创建 Admin API 密钥

<Note>
  在 Admin API 密钥可以调用 Activity Feed 之前，必须已[为您的 Claude Console 组织启用](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api) Compliance API。
</Note>

按照[创建 Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys#create-a-key-for-a-claude-console-organization)中的步骤操作，然后将密钥设置为环境变量：

```bash
export ANTHROPIC_ADMIN_KEY=sk-ant-admin01-...
```

使用不同的变量名可以防止在您同时配置两种密钥时 Admin API 密钥覆盖 Compliance Access Key。本指南中的 cURL 示例从 `$ANTHROPIC_COMPLIANCE_ACCESS_KEY` 读取密钥；使用 Admin API 密钥调用 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed) 时，请替换为 `$ANTHROPIC_ADMIN_KEY`。

仅当创建密钥时组织已启用 Compliance API，Admin API 密钥才会带有 `read:compliance_activities` 范围；请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access#set-up-the-compliance-api)。它们无法被授予任何其他 Compliance API 范围，因此对 Activity Feed 以外的任何端点的调用都会返回 [403 Forbidden](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)。

有关同一密钥在管理您的 Claude Console 组织中的作用，请参阅 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)。

## 检查您的密钥范围

要检查您已有密钥的范围，请使用以下信号之一。

* **密钥前缀。** `sk-ant-admin01-` 是 Admin API 密钥（仅带有 `read:compliance_activities`，受上一节中启用时机的限制）。`sk-ant-api01-` 是 Compliance Access Key；其范围是您在创建时选择的子集。
* **设置界面。** 打开 [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access) 中的 **Keys** 部分，或 [Claude Console > Settings > Admin keys](https://platform.claude.com/settings/admin-keys) 中的 **Admin keys** 部分，并查看该密钥的 **Scopes** 列。
* **错误响应。** 超出密钥范围的调用会返回 403，消息格式为 `Missing required scopes. Got: [<scopes the key carries>] Needed: [<scopes the endpoint requires>]`。有关完整的错误目录，请参阅[处理 Compliance API 错误](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-errors#403-forbidden)。

```json
{
  "error": {
    "type": "permission_error",
    "message": "Missing required scopes. Got: ['read:compliance_activities'] Needed: ['read:compliance_user_data']"
  }
}
```

## 管理和轮换密钥

从创建 Compliance Access Key 的同一 **Keys** 表中删除它：前往 [claude.ai > Organization settings > API](https://claude.ai/admin-settings/api-access)。从 [Claude Console > Settings > Admin keys](https://platform.claude.com/settings/admin-keys) 删除 Admin API 密钥。

删除密钥在下一次请求时生效：没有宽限期。Compliance Access Key 不会自行过期。

要在不中断服务的情况下轮换密钥：

1. 创建一个具有相同范围的新密钥。
2. 更新您的集成以使用新密钥。
3. 验证集成使用新密钥能够成功运行。
4. 删除旧密钥。

轮换前存储的分页游标仍然有效：游标的范围限定于组织，而非密钥。

如果 Compliance Access Key 泄露，请立即删除它，审计 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed) 中由被泄露密钥产生的 `compliance_api_accessed` 活动，并轮换泄露密钥可能触及的任何下游凭据。传递 `activity_types[]=compliance_api_accessed` 以限定查询范围，然后在您的客户端中保留 `actor.type` 为 `api_actor` 且 `actor.api_key_id` 与被泄露密钥匹配的活动；有关 actor 架构，请参阅[了解 Activity 对象](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed#understand-the-activity-object)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="查询 Activity Feed" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed">
    使用任何具有 `read:compliance_activities` 的密钥读取组织范围的活动事件。
  </Card>

  <Card title="检索和删除聊天、文件和项目" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-content-data">
    使用具有 `read:compliance_user_data` 的 Compliance Access Key 检索 claude.ai 聊天、文件和项目，并使用 `delete:compliance_user_data` 删除它们。
  </Card>

  <Card title="检索会话记录" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions">
    使用具有 `read:compliance_user_data` 的 Compliance Access Key 列出您的用户在 Claude 应用和代理（例如 Cowork 和 Claude Code）中运行的会话，并检索其记录。
  </Card>
</CardGroup>
