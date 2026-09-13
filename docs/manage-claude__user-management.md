---
title: 用户管理
url: https://platform.claude.com/docs/zh-CN/manage-claude/user-management
description: 使用 Admin API 管理您的 Claude Enterprise 组织中的人员：列出成员并更改角色、发送和撤回邀请、管理群组，以及读取自定义角色。
---

本页介绍如何使用 [Admin API](https://platform.claude.com/docs/zh-CN/api/admin) 以编程方式管理您的 **Claude Enterprise**（claude.ai）组织中的人员：列出成员并按电子邮件地址查找成员、更改成员的角色、移除成员、发送和撤回邀请、管理您企业的群组及其成员资格，以及读取您组织的自定义角色。对于 Claude Console（Claude Platform）组织，请参阅 [Claude Console 的 Admin API 指南](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)。

<Note>
  群组和自定义角色请求不需要 `anthropic-beta: ce-user-management-2026-07-13` [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)。仍然发送该标头的请求会被接受，且行为完全相同。
</Note>

## 您的组织可以使用哪些端点？

Admin API 是位于 `https://api.anthropic.com/v1/organizations/` 下的一组端点。Claude Console 和 Claude Enterprise 组织使用[不同的密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)进行身份验证，并且各自可以访问这些端点的不同子集：

| 端点                                                                                                                                                                                                                                                                                                                                                                                                           | Claude Console（Claude Platform）                                                       | Claude Enterprise（claude.ai） |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- | ---------------------------- |
| [成员](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#members)和[邀请](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#invites)                                                                                                                                                                                                                                        | 可用；请参阅 [Admin API 指南](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) | 可用（本页）                       |
| [群组](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#groups)                                                                                                                                                                                                                                                                                                                            | 不可用                                                                                   | 可用（本页）                       |
| [自定义角色](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#custom-roles)                                                                                                                                                                                                                                                                                                                   | 不可用                                                                                   | 可用，只读（本页）                    |
| [支出限额](https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api)                                                                                                                                                                                                                                                                                                                                | 不可用                                                                                   | 可用                           |
| [工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces)、[API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#api-keys)、[用量和成本报告](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api)、[速率限制](https://platform.claude.com/docs/zh-CN/manage-claude/rate-limits-api)，以及 [Admin API 指南](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)中的其他端点 | 可用                                                                                    | 不可用                          |

成员和邀请端点对两种组织类型是相同的；本页记录它们在 Claude Enterprise 中的行为，包括 Claude Enterprise 的[组织角色](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#organization-roles)。群组和自定义角色端点仅适用于 Claude Enterprise。

<Check>
  **需要具有作用域的 Admin API 密钥**

  这些端点需要具有以下作用域的 Admin API 密钥：`read:members` 作用域（成员和邀请的 `GET` 端点，以及所有自定义角色端点；没有单独的角色作用域）、`write:members` 作用域（成员和邀请的 `POST` 和 `DELETE` 端点）、`read:rbac_groups` 作用域（群组的 `GET` 端点），或 `write:rbac_groups` 作用域（群组的 `POST` 和 `DELETE` 端点）。携带 `read:org_audit` 作用域（用于安全审计集成的只读作用域）的密钥也可以调用本页上的每个 `GET` 端点以及 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 的读取端点。请参阅[创建 Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys#create-a-key-for-a-claude-enterprise-organization)，了解您的主要所有者在何处创建密钥以及应选择哪些作用域。在每个请求的 `x-api-key` 标头中传递该密钥。成员和邀请请求还需要 `anthropic-version: 2023-06-01` 标头，如示例所示；群组和自定义角色请求则不需要。
</Check>

## 概述

本页涵盖五种资源：

| 资源        | 端点                                                                                                                                                                                                                        | 用途                                   |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| **成员**    | `GET /v1/organizations/users` `GET /v1/organizations/users/{user_id}` `POST /v1/organizations/users/{user_id}` `DELETE /v1/organizations/users/{user_id}`                                                                 | 列出组织的成员或按电子邮件查找某个成员；更改成员的角色；移除成员。    |
| **邀请**    | `POST /v1/organizations/invites` `GET /v1/organizations/invites` `GET /v1/organizations/invites/{invite_id}` `DELETE /v1/organizations/invites/{invite_id}`                                                               | 邀请某人加入组织、跟踪邀请的状态，并在邀请被接受之前撤回它。       |
| **群组**    | `GET /v1/organizations/rbac_groups` `GET /v1/organizations/rbac_groups/{group_id}` `POST /v1/organizations/rbac_groups` `POST /v1/organizations/rbac_groups/{group_id}` `DELETE /v1/organizations/rbac_groups/{group_id}` | 读取您企业的群组以及附加到每个群组的自定义角色；创建、重命名和删除群组。 |
| **群组成员**  | `GET /v1/organizations/rbac_groups/{group_id}/members` `POST /v1/organizations/rbac_groups/{group_id}/members` `DELETE /v1/organizations/rbac_groups/{group_id}/members/{user_id}`                                        | 读取群组的成员；添加和移除成员。                     |
| **自定义角色** | `GET /v1/organizations/rbac_roles` `GET /v1/organizations/rbac_roles/{role_id}` `GET /v1/organizations/rbac_roles/{role_id}/permissions`                                                                                  | 读取您组织的自定义角色以及每个角色授予的权限。              |

自定义角色及其群组附加关系在 [claude.ai 组织设置](https://claude.ai/admin-settings)中管理；API 可以读取它们，但无法更改它们。

## 快速开始

列出组织的成员，最新的排在最前：

```bash cURL
curl "https://api.anthropic.com/v1/organizations/users?limit=20" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01"
```

```json
{
  "data": [
    {
      "type": "user",
      "id": "user_01AbCdEfGhIjKlMnOpQrSt",
      "email": "jane@example.com",
      "name": "Jane Smith",
      "role": "user",
      "added_at": "2026-06-12T09:14:03Z"
    }
  ],
  "has_more": false,
  "first_id": "user_01AbCdEfGhIjKlMnOpQrSt",
  "last_id": "user_01AbCdEfGhIjKlMnOpQrSt"
}
```

## 关键概念

### 组织角色

每个成员恰好拥有一个组织角色。读取操作会将成员的角色返回为以下五个值之一：

| 角色                 | 含义                        |
| ------------------ | ------------------------- |
| `user`             | 标准成员。                     |
| `managed`          | 其权限通过附加到其所属群组的自定义角色授予的成员。 |
| `owner`            | 组织所有者。                    |
| `membership_admin` | 可以管理组织成员的成员。              |
| `primary_owner`    | 组织的主要所有者。恰好只有一个。          |

API 只能在创建邀请和更新角色时分配 `user` 和 `managed` 角色。管理角色（`owner`、`membership_admin` 和 `primary_owner`）在 claude.ai 组织设置中分配，持有这些角色的成员无法通过此 API 修改或移除。

### 成员和邀请

一个人通过接受邀请（或在已配置的情况下通过您组织的单点登录）成为成员。创建邀请会发送一封邀请电子邮件；此后该邀请的状态为 `pending`，直到收件人接受（`accepted`）或其由服务器分配的 `expires_at` 过期（`expired`）。只有 `pending` 状态的邀请可以撤回。要更改待处理邀请的电子邮件地址或角色，请撤回它并创建一个新的邀请。

如果您组织的计划从有限的已购买席位池中分配成员，则待处理的邀请会占用一个席位。创建邀请端点不接受席位或层级参数：席位会自动从有可用名额的最低层级分配。在没有空闲席位时创建邀请会失败并返回 400 错误，而不会购买席位。撤回邀请、让其过期或稍后移除该成员都会将席位归还到池中。

### 群组和角色

群组将成员与自定义角色关联起来（"role-based access control"（基于角色的访问控制），即端点路径和作用域名称中的 `rbac`）。群组由您的整个企业（父组织及其下的每个组织）拥有，而不是由单个组织拥有，因此群组作用域（`read:rbac_groups` 和 `write:rbac_groups`）需要为所有关联组织创建的密钥。每个群组都带有一个 `source_type`：在 claude.ai 中创建的群组为 `direct`，由您的身份提供商配置的群组为 `scim`。群组的 `roles` 字段列出附加到该群组的自定义角色的 ID；可使用[自定义角色端点](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#custom-roles)将它们解析为名称和权限。请注意，角色目录是按组织划分的，而群组是企业范围的，因此获取属于您企业中其他组织的角色时，您的密钥会收到 404。当角色数据暂时不可用时，该字段为 `null`（而不是 `[]`），因此请重试以区分降级读取与没有角色的群组。

## 速率限制

Admin API 端点共享每个组织**每分钟 100 个请求**的限制；邀请创建则有其自己的限制，为**每小时 1,200 个请求**。超过限制的请求会返回 **429 Too Many Requests**。

## 分页

成员和邀请列表使用基于 ID 的分页：传递 `limit`（默认 20，最大 1000）以及 `before_id` 或 `after_id` 中的至多一个，并使用每个响应的 `first_id` 和 `last_id` 字段进行翻页，直到 `has_more` 为 `false`。群组和自定义角色列表则使用**不透明游标**：将响应的 `next_page` 值原样作为下一个请求的 `page` 参数传递，直到 `next_page` 为 `null`。

## 错误响应

错误响应遵循[错误](https://platform.claude.com/docs/zh-CN/api/errors)中记录的标准格式。

## 成员

### 列出成员

`GET /v1/organizations/users` 返回组织的成员，最近添加的排在最前。按 `email` 筛选可查找特定成员；匹配不区分大小写，并容许同一地址的常见变体（例如，`jane+hiring@example.com` 匹配 `jane@example.com`）。需要 `read:members` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[列出用户](https://platform.claude.com/docs/zh-CN/api/admin/users/list)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/users?email=jane@example.com" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01"
```

### 获取成员

`GET /v1/organizations/users/{user_id}` 按 ID 返回一个成员。需要 `read:members` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[获取用户](https://platform.claude.com/docs/zh-CN/api/admin/users/retrieve)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/users/user_01AbCdEfGhIjKlMnOpQrSt" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01"
```

### 更改成员的角色

`POST /v1/organizations/users/{user_id}` 将成员的角色设置为 `user` 或 `managed`。持有管理角色（`owner`、`membership_admin` 或 `primary_owner`）的成员无法通过此端点更改，并且无法分配管理角色；两者都会返回 400，并在 claude.ai 组织设置中管理。如果您组织的身份提供商管理角色（高级 SSO 或高级 SCIM 配置），角色更新会返回 400。需要 `write:members` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[更新用户](https://platform.claude.com/docs/zh-CN/api/admin/users/update)。

```bash cURL
curl -X POST "https://api.anthropic.com/v1/organizations/users/user_01AbCdEfGhIjKlMnOpQrSt" \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"role": "managed"}'
```

### 移除成员

`DELETE /v1/organizations/users/{user_id}` 将成员从组织中移除，并将其占用的任何已购买席位归还到组织的池中。持有管理角色的成员无法通过此端点移除，并且如果您的身份提供商管理成员资格（SCIM），移除操作会返回 400。需要 `write:members` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[移除用户](https://platform.claude.com/docs/zh-CN/api/admin/users/delete)。

```bash cURL
curl -X DELETE "https://api.anthropic.com/v1/organizations/users/user_01AbCdEfGhIjKlMnOpQrSt" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01"
```

```json
{
  "type": "user_deleted",
  "id": "user_01AbCdEfGhIjKlMnOpQrSt"
}
```

## 邀请

### 创建邀请

`POST /v1/organizations/invites` 发送一封邀请电子邮件，并返回带有服务器分配的 `expires_at` 的邀请。`role` 必须为 `user` 或 `managed`。如果该电子邮件地址已存在待处理的邀请，或该地址已属于某个成员，请求会返回 400 并指明现有资源。其身份提供商自动配置用户（JIT 或 SCIM）的组织无法通过 API 创建邀请。需要 `write:members` 作用域。

在从有限席位池中分配成员的计划中，邀请会自动从有可用名额的最低层级占用一个席位；API 不接受层级参数。如果没有空闲席位，请求会失败并返回 400 错误，而不会购买席位。请通过组织的计划管理添加席位后重试。

可选的 `rbac_group_ids` 字段列出在成员接受邀请时要分配给他们的群组（按带 `rbac_group_` 前缀的 ID）。传递非空的 `rbac_group_ids` 还要求密钥携带 `write:rbac_groups` 作用域，因为群组分配可能会授予附加到该群组角色的权限。

有关完整的参数详情和响应模式，请参阅 API 参考中的[创建邀请](https://platform.claude.com/docs/zh-CN/api/admin/invites/create)。

```bash cURL
curl -X POST "https://api.anthropic.com/v1/organizations/invites" \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "email": "newhire@example.com",
    "role": "managed",
    "rbac_group_ids": ["rbac_group_01UvWxYzAbCdEfGhIjKlMn"]
  }'
```

```json
{
  "type": "invite",
  "id": "invite_01QrStUvWxYzAbCdEfGhIj",
  "email": "newhire@example.com",
  "role": "managed",
  "invited_at": "2026-07-06T16:20:11Z",
  "expires_at": "2026-07-27T16:20:11Z",
  "accepted_at": null,
  "status": "pending",
  "rbac_group_ids": ["rbac_group_01UvWxYzAbCdEfGhIjKlMn"]
}
```

### 列出邀请

`GET /v1/organizations/invites` 返回组织的邀请，最近的排在最前，涵盖 `pending`、`accepted` 和 `expired` 状态；没有状态筛选器。需要 `read:members` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[列出邀请](https://platform.claude.com/docs/zh-CN/api/admin/invites/list)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/invites?limit=20" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01"
```

### 获取邀请

`GET /v1/organizations/invites/{invite_id}` 按 ID 返回一个邀请。需要 `read:members` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[获取邀请](https://platform.claude.com/docs/zh-CN/api/admin/invites/retrieve)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/invites/invite_01QrStUvWxYzAbCdEfGhIj" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01"
```

### 撤回邀请

`DELETE /v1/organizations/invites/{invite_id}` 撤回 `pending` 状态的邀请，使邀请电子邮件中的链接失效。撤回 `accepted` 状态的邀请会返回 400（请改为移除该成员）；撤回 `expired` 状态的邀请会返回 400。需要 `write:members` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[删除邀请](https://platform.claude.com/docs/zh-CN/api/admin/invites/delete)。

```bash cURL
curl -X DELETE "https://api.anthropic.com/v1/organizations/invites/invite_01QrStUvWxYzAbCdEfGhIj" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01"
```

## 群组

您的企业直接创建的群组——无论是在 [claude.ai 组织设置](https://claude.ai/admin-settings)中还是通过此 API 创建（`source_type: "direct"`）——支持本节中的每个端点。由您的身份提供商配置的群组（`source_type: "scim"`）可以读取但不能修改：重命名或删除 SCIM 群组，或更改其成员资格，都会返回 400，因为它由您的身份提供商拥有。与成员和邀请请求不同，群组请求不需要 `anthropic-version` 标头。

### 列出群组

`GET /v1/organizations/rbac_groups` 返回您企业的群组，包括由身份提供商管理的（`scim`）群组。需要 `read:rbac_groups` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[列出群组](https://platform.claude.com/docs/zh-CN/api/admin/rbac_groups/list)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/rbac_groups?limit=20" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

```json
{
  "data": [
    {
      "type": "rbac_group",
      "id": "rbac_group_01UvWxYzAbCdEfGhIjKlMn",
      "name": "Engineering",
      "source_type": "direct",
      "roles": ["rbac_role_01CdEfGhIjKlMnOpQrStUv"],
      "created_at": "2026-03-18T10:01:42Z",
      "updated_at": "2026-05-02T08:55:09Z"
    }
  ],
  "has_more": false,
  "next_page": null
}
```

### 获取群组

`GET /v1/organizations/rbac_groups/{group_id}` 按 ID 返回一个群组。需要 `read:rbac_groups` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[获取群组](https://platform.claude.com/docs/zh-CN/api/admin/rbac_groups/retrieve)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/rbac_groups/rbac_group_01UvWxYzAbCdEfGhIjKlMn" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

### 创建群组

`POST /v1/organizations/rbac_groups` 使用给定的 `name`（1–255 个字符）创建一个没有角色或成员的群组。需要 `write:rbac_groups` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[创建群组](https://platform.claude.com/docs/zh-CN/api/admin/rbac_groups/create)。

```bash cURL
curl -X POST "https://api.anthropic.com/v1/organizations/rbac_groups" \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -d '{"name": "Engineering"}'
```

```json
{
  "type": "rbac_group",
  "id": "rbac_group_01UvWxYzAbCdEfGhIjKlMn",
  "name": "Engineering",
  "source_type": "direct",
  "roles": [],
  "created_at": "2026-07-09T18:00:00Z",
  "updated_at": "2026-07-09T18:00:00Z"
}
```

### 重命名群组

`POST /v1/organizations/rbac_groups/{group_id}` 更新群组。`name` 是此端点唯一可以更改的字段。需要 `write:rbac_groups` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[更新群组](https://platform.claude.com/docs/zh-CN/api/admin/rbac_groups/update)。

```bash cURL
curl -X POST "https://api.anthropic.com/v1/organizations/rbac_groups/rbac_group_01UvWxYzAbCdEfGhIjKlMn" \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -d '{"name": "Platform Engineering"}'
```

### 删除群组

`DELETE /v1/organizations/rbac_groups/{group_id}` 删除群组。其成员仍然是各自组织的成员，但会失去该群组所附加角色的权限，并且群组[支出限额](https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api)（如果存在）将不再适用于他们。需要 `write:rbac_groups` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[删除群组](https://platform.claude.com/docs/zh-CN/api/admin/rbac_groups/delete)。

```bash cURL
curl -X DELETE "https://api.anthropic.com/v1/organizations/rbac_groups/rbac_group_01UvWxYzAbCdEfGhIjKlMn" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

```json
{
  "id": "rbac_group_01UvWxYzAbCdEfGhIjKlMn",
  "type": "rbac_group_deleted"
}
```

### 列出群组的成员

`GET /v1/organizations/rbac_groups/{group_id}/members` 返回群组的成员（每个成员带有其 `user_id` 和电子邮件），最早的排在最前。仅返回您企业各组织的当前成员，因此当 `has_more` 为 `true` 时，某一页可能包含少于 `limit` 个条目。需要 `read:rbac_groups` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[列出群组成员](https://platform.claude.com/docs/zh-CN/api/admin/rbac_groups/members/list)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/rbac_groups/rbac_group_01UvWxYzAbCdEfGhIjKlMn/members?limit=100" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

```json
{
  "data": [
    {
      "type": "rbac_group_member",
      "group_id": "rbac_group_01UvWxYzAbCdEfGhIjKlMn",
      "user_id": "user_01AbCdEfGhIjKlMnOpQrSt",
      "email": "jane@example.com",
      "created_at": "2026-04-07T12:30:00Z"
    }
  ],
  "has_more": false,
  "next_page": null
}
```

### 向群组添加成员

`POST /v1/organizations/rbac_groups/{group_id}/members` 按 `user_id` 将组织成员添加到群组。该用户必须已经是您企业某个组织的成员（否则请求返回 404），并且添加已在群组中的人会返回 400。对于 `scim` 群组，成员资格在您的身份提供商中管理，此请求会返回 400。要为尚未加入的人分配群组，请改为在[创建邀请](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#create-an-invite)时使用 `rbac_group_ids`。需要 `write:rbac_groups` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[添加群组成员](https://platform.claude.com/docs/zh-CN/api/admin/rbac_groups/members/create)。

```bash cURL
curl -X POST "https://api.anthropic.com/v1/organizations/rbac_groups/rbac_group_01UvWxYzAbCdEfGhIjKlMn/members" \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -d '{"user_id": "user_01AbCdEfGhIjKlMnOpQrSt"}'
```

```json
{
  "type": "rbac_group_member",
  "group_id": "rbac_group_01UvWxYzAbCdEfGhIjKlMn",
  "user_id": "user_01AbCdEfGhIjKlMnOpQrSt",
  "email": "jane@example.com",
  "created_at": "2026-07-09T18:00:00Z"
}
```

### 从群组中移除成员

`DELETE /v1/organizations/rbac_groups/{group_id}/members/{user_id}` 将成员从群组中移除；他们仍然是其组织的成员。如果该用户不是群组的成员，请求返回 404；对于 `scim` 群组则返回 400，因为其成员资格在您的身份提供商中管理。需要 `write:rbac_groups` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[移除群组成员](https://platform.claude.com/docs/zh-CN/api/admin/rbac_groups/members/delete)。

```bash cURL
curl -X DELETE "https://api.anthropic.com/v1/organizations/rbac_groups/rbac_group_01UvWxYzAbCdEfGhIjKlMn/members/user_01AbCdEfGhIjKlMnOpQrSt" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

```json
{
  "group_id": "rbac_group_01UvWxYzAbCdEfGhIjKlMn",
  "user_id": "user_01AbCdEfGhIjKlMnOpQrSt",
  "type": "rbac_group_member_deleted"
}
```

## 自定义角色

自定义角色通过 API 是只读的：这些端点列出您组织的自定义角色（在 [claude.ai 组织设置](https://claude.ai/admin-settings)中定义或由 Anthropic 配置）以及每个角色授予的权限。自定义角色读取使用 `read:members` 作用域（没有单独的角色作用域），并可使用组织级密钥：与群组端点不同，它们不需要为所有关联组织创建的密钥，并且返回的目录是您组织自己的目录。

### 列出角色

`GET /v1/organizations/rbac_roles` 返回您组织的自定义角色。需要 `read:members` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[列出角色](https://platform.claude.com/docs/zh-CN/api/admin/rbac_roles/list)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/rbac_roles?limit=20" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

```json
{
  "data": [
    {
      "type": "rbac_role",
      "id": "rbac_role_01CdEfGhIjKlMnOpQrStUv",
      "name": "Engineering base",
      "created_at": "2026-03-18T10:01:42Z",
      "updated_at": "2026-05-02T08:55:09Z"
    }
  ],
  "has_more": false,
  "next_page": null
}
```

### 获取角色

`GET /v1/organizations/rbac_roles/{role_id}` 按 ID 返回一个角色。需要 `read:members` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[获取角色](https://platform.claude.com/docs/zh-CN/api/admin/rbac_roles/retrieve)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/rbac_roles/rbac_role_01CdEfGhIjKlMnOpQrStUv" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

### 列出角色的权限

`GET /v1/organizations/rbac_roles/{role_id}/permissions` 返回角色的权限。每个权限将一个 `resource`（它所适用的对象：组织的产品功能、连接器工具、连接器 OAuth 作用域、单个连接器或所有连接器）与一个 `action`（它在该资源上授予的内容）配对。未为您的组织启用的功能对应的行会被省略，因此当 `has_more` 为 `true` 时，某一页可能包含少于 `limit` 行。需要 `read:members` 作用域。

有两个 `action` 值需要特别注意：action 为 `capability_access_all`（所有产品功能）或 `capability_access_all_ga`（所有稳定的产品功能，即所有未标记为 beta 或研究预览的功能）的 `organization` 权限是一种全面授权（既不涵盖模型访问，也不涵盖带 `permission_` 前缀的管理面板权限），并以该单行列出而不展开。当您统计某个角色授予的内容时，请将全面授权行视为涵盖其变体所描述的所有内容，而不仅仅是其他行中列出的功能。

有关完整的参数详情和响应模式，请参阅 API 参考中的[列出角色权限](https://platform.claude.com/docs/zh-CN/api/admin/rbac_roles/permissions/list)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/rbac_roles/rbac_role_01CdEfGhIjKlMnOpQrStUv/permissions?limit=20" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

```json
{
  "data": [
    {
      "type": "rbac_role_permission",
      "resource": {
        "type": "organization",
        "organization_id": "12345678-1234-5678-1234-567812345678"
      },
      "action": "capability_access_all_ga"
    },
    {
      "type": "rbac_role_permission",
      "resource": {
        "type": "connector_tool",
        "connector_id": "mcpsrv_01WxYzAbCdEfGhIjKlMnOp",
        "tool_name": "search_tickets"
      },
      "action": "use"
    }
  ],
  "has_more": false,
  "next_page": null
}
```

## 示例工作流

### 为离职员工办理离职

1. 按电子邮件查找该成员：

   ```bash cURL
   curl "https://api.anthropic.com/v1/organizations/users?email=departing@example.com" \
     -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
     -H "anthropic-version: 2023-06-01"
   ```

2. 使用响应中的 `id`，通过 `DELETE /v1/organizations/users/{user_id}` 移除他们。他们的席位（如果有）会归还到池中。

3. 如果此人尚未加入，查找不会返回任何成员；请改为列出邀请并撤回其 `pending` 状态的邀请。

### 审计群组成员资格

1. 列出群组并记录每个群组的 `id`、`name` 和 `roles`。

2. 对于每个带有敏感角色的群组，分页遍历 `GET /v1/organizations/rbac_groups/{group_id}/members`，并将成员电子邮件与您身份提供商的名册进行比较。

3. 使用 `DELETE /v1/organizations/rbac_groups/{group_id}/members/{user_id}` 移除不应再留在群组中的成员。对于 `scim` 群组，请改为在您的身份提供商中进行更改。

有关将群组成员资格与临时提高支出限额相结合的工作流，请参阅 Spend Limits API 页面上的[在事件期间临时提高成员的支出限额](https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api#temporarily-raise-a-members-spend-limit-during-an-incident)。

## 常见问题

### 这是与 Admin API 不同的 API 吗？

不是。成员和邀请端点与 Claude Console 组织使用的 `/v1/organizations/` 端点相同；本页记录它们在 Claude Enterprise 中的行为。群组和自定义角色端点是同一 API 的一部分，仅适用于 Claude Enterprise 组织。[可用性表](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#which-endpoints-can-your-organization-use)显示了每种组织类型可以调用哪些端点。

### 我可以通过 API 分配所有者或成员管理员角色吗？

不可以。API 仅在创建邀请和更新角色时分配 `user` 和 `managed`。管理角色在 claude.ai 组织设置中分配，持有这些角色的成员无法通过 API 修改或移除。

### 我可以通过 API 创建或修改群组吗？

可以，使用 `write:rbac_groups` 作用域：创建、重命名和删除群组，以及添加或移除其成员。API 无法更改两样东西：由您的身份提供商配置的群组（`source_type: "scim"`），其名称和成员资格由身份提供商拥有；以及自定义角色，它们在 claude.ai 组织设置中管理（API 可以[读取它们](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#custom-roles)）。

### 未接受的邀请会占用席位吗？

在具有有限席位池的计划中，会：`pending` 状态的邀请会占用一个席位。撤回邀请或让其过期会释放该席位。在没有席位池的计划中，邀请不占用任何资源。

### 我的组织使用单点登录。哪些操作可用？

如果您的身份提供商自动配置用户（JIT 或 SCIM），邀请创建会返回 400。如果它管理角色（高级 SSO 或高级 SCIM 配置），角色更新会返回 400。如果它管理成员资格（SCIM 配置），成员移除会返回 400。读取操作在任何情况下都可用。

### 当创建 Admin API 密钥的人离开时，该密钥会怎样？

密钥会继续有效。Admin API 密钥的作用域是组织，而不是个人用户，并且在 claude.ai 中创建的密钥不会过期。将创建者从组织中移除或通过您的身份提供商取消其配置会终止他们自己的访问权限，但不会终止他们创建的密钥。降低他们的角色也不会改变这些密钥：每个密钥都会以其原始作用域保持有效。当您为创建过 Admin API 密钥的人办理离职时，请在 [claude.ai > 组织设置 > API](https://claude.ai/admin-settings/api-access) 的 **Keys** 部分删除这些密钥并创建替代密钥。

## 另请参阅

<CardGroup cols={2}>
  <Card title="创建 Admin API 密钥" href="https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys">
    您的主要所有者在何处创建具有作用域的密钥以及应选择哪些作用域。
  </Card>

  <Card title="Compliance API" href="https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api">
    审计活动并检索或删除整个组织中的用户内容。
  </Card>

  <Card title="Analytics API" href="https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api">
    面向 Claude Enterprise 的按用户和按时间分段的用量和成本报告。
  </Card>

  <Card title="Spend Limits API" href="https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api">
    设置每个成员的支出限额并审核提额请求。
  </Card>
</CardGroup>
