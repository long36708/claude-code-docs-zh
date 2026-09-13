---
title: 支出限额 API
url: https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api
description: 为每位 Claude Enterprise 成员设置支出限额，查看每位成员的支出限额继承自何处，并审核或处理成员提出的提高限额请求。
---

Spend Limits API（支出限额 API）让您可以为每位 Claude Enterprise 成员设置支出限额，查看每位成员的支出限额继承自何处，并审核或处理成员提出的提高限额请求。

如需按用户和按时间分桶的用量与成本*报告*，请参阅 [Analytics APIs](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api)。

<Check>
  **需要具有相应作用域的 Admin API 密钥**

  这些端点需要具有 `read:spend_limits` 作用域（用于 `GET` 端点）或 `write:spend_limits` 作用域（用于 `POST` 和 `DELETE` 端点）的 Admin API 密钥。请参阅[创建 Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys#create-a-key-for-a-claude-enterprise-organization)，了解您的主要所有者在何处创建密钥以及应选择哪些作用域。在每个请求的 `x-api-key` 标头中传递该密钥。
</Check>

<Note>
  支出限额 API 仅适用于 Claude Enterprise 组织。它不适用于 Claude Platform（Claude Console）组织。
</Note>

## 概述

该 API 在两种资源上公开了八个端点：

| 资源           | 端点                                                                                                                                                                                                                                                    | 用途                                       |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| **支出限额**     | `GET /v1/organizations/spend_limits/effective` `GET /v1/organizations/spend_limits/{spend_limit_id}` `POST /v1/organizations/spend_limits` `DELETE /v1/organizations/spend_limits/{spend_limit_id}`                                                   | 读取每位成员的有效支出限额和本期至今支出；设置或清除按用户的覆盖值。       |
| **支出限额提高请求** | `GET /v1/organizations/spend_limit_increase_requests` `GET /v1/organizations/spend_limit_increase_requests/{id}` `POST /v1/organizations/spend_limit_increase_requests/{id}/approve` `POST /v1/organizations/spend_limit_increase_requests/{id}/deny` | 列出成员提出的提高支出限额请求，并附带做出决定所需的上下文；批准或拒绝每个请求。 |

使用**支出限额**端点来回答"每位成员适用什么支出限额、它来自哪里、以及他们距离限额还有多远？"这类问题，并设置按用户的覆盖值。使用**支出限额提高请求**端点来处理成员提交的请求队列。

## 前提条件

* 您的组织必须使用 Claude Enterprise 计划。
* 您的组织必须已开启用量额度（usage credits）。您的主要所有者可以在 claude.ai 账单设置中开启。

## 快速开始

列出每位成员的有效月度支出限额和本期至今支出：

```bash cURL
curl "https://api.anthropic.com/v1/organizations/spend_limits/effective?limit=20" \
  --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

## 关键概念

### 支出限额层级

每位成员的支出都适用一个 **effective spend limit**（有效支出限额），它从一个由多个作用域级别组成的层级中解析得出。当成员没有按用户的覆盖值时，他们会继承为其所在组（如果您的组织使用基于组的限额）、其席位层级或组织范围默认值所配置的支出限额。组支出限额是按成员的默认值：每位继承该限额的成员都以其自身的支出为准进行限制，而不是共享的组预算。

读取 `GET /v1/organizations/spend_limits/effective` 会返回每位当前成员及其解析后的有效支出限额、该限额的解析来源（`source`）以及其本期至今支出。使用 `POST /v1/organizations/spend_limits` 设置按用户的覆盖值会将成员固定在特定的支出限额上，而不管他们原本会继承什么。删除覆盖值会使他们恢复到继承的支出限额（如果不存在继承限额，则保持无限制）。

每位成员所在行的 `source` 字段告诉您其支出限额是从哪个级别解析出来的：`user`（按用户的覆盖值）、`seat_tier`、`rbac_group` 或 `organization`。请将作用域类型视为开放集合；遇到未知值时应跳过处理而不是报错。

### 周期

`period`（周期）是执行支出限额并重置支出的循环窗口。支出限额由其 `(scope, period)` 对来标识。目前 `monthly` 是唯一支持的周期；月度支出在每个日历月第一天的 00:00 UTC 重置。请将 `period` 视为开放集合。

### 金额与货币

所有货币值均为字符串，以**组织账单货币的最小单位**表示（对于 USD 为美分）。例如，`"50000"` 表示 500.00 USD。请将其解析为十进制数并除以 100 以显示美元；对于较大的值，请避免使用二进制浮点数。

`amount` **可为空**。在成员的有效行中，`null` 表示**无限制**（没有支出限额），而 `"0"` 表示该成员不能在其计划所含用量之外使用 Claude。在已配置的支出限额行上（如 `GET /v1/organizations/spend_limits/{id}` 返回的），`null` 仅表示未设置数值型支出限额；请读取成员的有效行以区分无限制与仅限所含用量。

`period_to_date_spend` 是成员自当前 `period` 开始以来累计的支出，采用相同的最小单位格式；它可能包含小数部分（例如 `"41280.125"`）。如果支出读数暂时不可用，它可能显示为 `"0"`；请将其视为参考信息，而非事务性数据。

### 提高请求的生命周期

当成员在 claude.ai 中点击 **Request more usage**（请求更多用量）时，会创建一个 **spend limit increase request**（支出限额提高请求）。请求不会通过此 API 创建。请求的 `status` 为以下之一：

| 状态         | 含义                                                                                                  |
| ---------- | --------------------------------------------------------------------------------------------------- |
| `pending`  | 等待管理员处理。请求通常携带实时的 `spend_summary`，以便您在做决定时查看成员当前的有效支出限额和本期至今支出；如果无法计算，`spend_summary` 可能为 `null`。   |
| `approved` | 请求已以批准方式解决：要么管理员明确批准了它，要么另一项管理员操作提高了该成员的支出限额，要么 Anthropic 支持团队代表组织提高了支出限额。`spend_summary` 为 `null`。 |
| `denied`   | 管理员已拒绝。`spend_summary` 为 `null`。claude.ai 会自 `resolved_at` 起 30 天内隐藏该成员的请求按钮；管理员仍可随时直接提高该成员的支出限额。   |

`approved` 和 `denied` 均为终态。一位成员同一时间最多只有一个 `pending` 请求。

使用 `POST /v1/organizations/spend_limit_increase_requests/{id}/approve` 批准时，会写入与 `POST /v1/organizations/spend_limits` 所写入的相同的按用户支出限额行。直接设置支出限额**不会**转换待处理请求的状态；请使用批准端点来解决请求。

默认情况下，当成员的请求被批准或拒绝时，Anthropic 会向该成员发送电子邮件。在批准或拒绝时传递 `suppress_notification: true` 可抑制该邮件（例如，当您自己的系统会通知成员时）。

## 速率限制

所有八个端点共享一个按组织计算的速率限制，即**每分钟 60 个请求**。超出限制的请求将返回 **429 Too Many Requests**。

## 分页

`GET /v1/organizations/spend_limits/effective` 和 `GET /v1/organizations/spend_limit_increase_requests` 使用**不透明游标**进行分页。第一个请求返回最多 `limit` 行以及一个 `next_page` 游标；在下一个请求中将该游标原样作为 `page` 参数传递，并重复此过程直到 `next_page` 为 `null`。

\*\*不要在序列中途更改查询参数。\*\*游标与签发它们的筛选条件绑定。如果您更改了 `user_ids[]`、`period[]`、`status[]` 或 `actor_ids[]` 并传递旧游标，您将收到 400 错误，提示 *"cursor does not match current query parameters"*。请改为从第一页开始新的序列。

## 序列化列表参数

列表参数使用方括号表示法：为每个值重复带 `[]` 的参数名。

```text wrap
user_ids[]=user_01AbCdEfGh&user_ids[]=user_01JkLmNoPq
```

## 错误响应

错误响应遵循[错误](https://platform.claude.com/docs/zh-CN/api/errors)中记录的标准格式。联系支持团队时，请引用响应正文中的 `request_id`。

## 支出限额

### 列出每位成员的有效支出限额

`GET /v1/organizations/spend_limits/effective` 为每位当前成员返回一行，反映每位成员的有效支出限额、其在作用域层级中的 `source` 以及其 `period_to_date_spend`。需要 `read:spend_limits` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[列出有效支出限额](https://platform.claude.com/docs/zh-CN/api/admin/spend_limits/list_effective)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/spend_limits/effective?limit=20" \
  --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

```json
{
  "data": [
    {
      "scope": { "type": "user", "user_id": "user_01AbCdEfGh" },
      "actor": {
        "type": "user_actor",
        "user_id": "user_01AbCdEfGh",
        "name": "Jane Smith",
        "email_address": "jane@example.com",
        "deleted": false
      },
      "amount": "50000",
      "currency": "USD",
      "period": "monthly",
      "source": { "type": "seat_tier", "seat_tier": "enterprise_standard" },
      "spend_limit_id": "spl_01XyZaBcDeFgHiJkLmNoPq",
      "period_to_date_spend": "31402.5"
    }
  ],
  "next_page": "page_..."
}
```

### 获取单个支出限额

`GET /v1/organizations/spend_limits/{spend_limit_id}` 按 ID 返回一个已配置的支出限额。使用它来检查某个 `spend_limit_id` 字段所引用的行。需要 `read:spend_limits` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[检索支出限额](https://platform.claude.com/docs/zh-CN/api/admin/spend_limits/retrieve)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/spend_limits/spl_01AbCdEfGhIjKlMnOpQrSt" \
  --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

### 设置按用户的覆盖值

`POST /v1/organizations/spend_limits` 设置按用户的支出限额覆盖值。这是一个以 `(scope, period)` 为键的 upsert 操作：为已有限额的用户和周期设置限额会就地覆盖它。此端点仅接受 `scope.type: "user"`；席位层级、组和组织级别的默认值在 claude.ai 设置中配置。需要 `write:spend_limits` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[创建支出限额](https://platform.claude.com/docs/zh-CN/api/admin/spend_limits/create)。

```bash cURL
curl --request POST "https://api.anthropic.com/v1/organizations/spend_limits" \
  --header "content-type: application/json" \
  --header "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  --data '{"scope": {"type": "user", "user_id": "user_01AbCdEfGh"}, "amount": "75000"}'
```

```json
{
  "type": "spend_limit",
  "id": "spl_01RsTuVwXyZaBcDeFgHiJk",
  "created_at": "2026-05-11T10:02:44Z",
  "updated_at": "2026-05-11T10:02:44Z",
  "scope": { "type": "user", "user_id": "user_01AbCdEfGh" },
  "amount": "75000",
  "currency": "USD",
  "period": "monthly"
}
```

### 移除按用户的覆盖值

`DELETE /v1/organizations/spend_limits/{spend_limit_id}` 移除按用户的覆盖值，之后该成员将回退到任何继承的席位层级、组或组织默认值。席位层级、组和组织级别的行不能通过此端点删除。需要 `write:spend_limits` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[删除支出限额](https://platform.claude.com/docs/zh-CN/api/admin/spend_limits/delete)。

```bash cURL
curl --request DELETE "https://api.anthropic.com/v1/organizations/spend_limits/spl_01RsTuVwXyZaBcDeFgHiJk" \
  --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

## 支出限额提高请求

### 列出提高请求

`GET /v1/organizations/spend_limit_increase_requests` 列出请求，最新的排在最前。可按 `status[]`（`pending`、`approved`、`denied`）和 `actor_ids[]` 筛选。该列表不包含请求者已不再是组织成员的请求。需要 `read:spend_limits` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[列出支出限额提高请求](https://platform.claude.com/docs/zh-CN/api/admin/spend_limits/increase_requests/list)。

```bash cURL
curl --globoff "https://api.anthropic.com/v1/organizations/spend_limit_increase_requests?status[]=pending&limit=50" \
  --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

每个待处理请求都携带实时的 `spend_summary`，显示请求者当前的有效支出限额和本期至今支出，足以在无需单独查询的情况下做出决定。

### 获取单个提高请求

`GET /v1/organizations/spend_limit_increase_requests/{id}` 按 ID 返回一个请求。需要 `read:spend_limits` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[检索支出限额提高请求](https://platform.claude.com/docs/zh-CN/api/admin/spend_limits/increase_requests/retrieve)。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/spend_limit_increase_requests/slir_01AbCdEfGhIjKlMnOpQrSt" \
  --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

### 批准提高请求

`POST /v1/organizations/spend_limit_increase_requests/{id}/approve` 批准一个待处理请求：它以管理员提供的 `amount` 为请求者写入按用户的支出限额，并将请求转换为 `approved`。请求本身不携带所请求的金额；您在批准时提供新的支出限额。需要 `write:spend_limits` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[批准支出限额提高请求](https://platform.claude.com/docs/zh-CN/api/admin/spend_limits/increase_requests/approve)。

```bash cURL
curl --request POST "https://api.anthropic.com/v1/organizations/spend_limit_increase_requests/slir_01AbCdEfGhIjKlMnOpQrSt/approve" \
  --header "content-type: application/json" \
  --header "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  --data '{"amount": "75000", "suppress_notification": true}'
```

### 拒绝提高请求

`POST /v1/organizations/spend_limit_increase_requests/{id}/deny` 拒绝一个待处理请求。对 `denied` 状态是幂等的：拒绝一个已被拒绝的请求会返回 200 及现有资源。该端点会拒绝对已批准请求执行拒绝操作的尝试，以便自动化流程能够区分重试与冲突的决定。需要 `write:spend_limits` 作用域。

有关完整的参数详情和响应模式，请参阅 API 参考中的[拒绝支出限额提高请求](https://platform.claude.com/docs/zh-CN/api/admin/spend_limits/increase_requests/deny)。

```bash cURL
curl --request POST "https://api.anthropic.com/v1/organizations/spend_limit_increase_requests/slir_01AbCdEfGhIjKlMnOpQrSt/deny" \
  --header "content-type: application/json" \
  --header "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  --data '{"suppress_notification": true}'
```

## 示例工作流

其中一些工作流将支出限额 API 与 [Analytics APIs](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api) 的成本端点结合使用。Analytics 成本端点专为跨日期范围的组织级支出报告而设计。`GET /spend_limits/effective` 返回当前适用于每位成员的上限。先用 Analytics 进行一次扫描以发现需要关注哪些成员，然后用 `/effective` 读取他们当前的上限。

支出限额端点需要 `spend_limits` 作用域，而 Analytics 成本端点需要 `read:analytics`；有关如何配置访问权限，请参阅 [Analytics APIs](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api)。两者的所有货币值均为以最小单位（美分）表示的十进制字符串。两个 API 都使用不透明游标进行分页。请设置明确的 `limit` 并通过 `next_page` 翻页直到其为 `null`，以覆盖整个组织。

### 自动化提高请求审核流程

运行一个定时任务，获取待处理请求，应用您组织的批准策略，并解决每个请求。

1. 列出待处理请求：

   ```bash cURL
   curl --globoff "https://api.anthropic.com/v1/organizations/spend_limit_increase_requests?status[]=pending&limit=100" \
     --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
   ```

   每个请求都携带请求者的 `actor.user_id` 和实时的 `spend_summary`，其中包含其当前有效的 `amount` 和 `period_to_date_spend`，足以在无需单独查询的情况下做出决定。

2. 应用您的策略。例如，当成员当前的 `amount` 低于某个阈值时自动批准，并将较大的上限转交人工审核。

3. 解决每个请求。要批准，请提供新的上限：

   ```bash cURL
   curl --request POST "https://api.anthropic.com/v1/organizations/spend_limit_increase_requests/{id}/approve" \
     --header "content-type: application/json" \
     --header "x-api-key: $ANTHROPIC_ADMIN_KEY" \
     --data '{"amount": "75000", "suppress_notification": true}'
   ```

   要拒绝，请改为向 `.../{id}/deny` 发送 `POST`。当您自己的系统会通知请求者时，请传递 `suppress_notification: true`。

### 识别接近支出限额的成员

找出接近上限的成员，以便在他们被阻止之前提高上限。

1. 从 Analytics API 拉取每位成员的本月至今支出（每位成员一行，默认按支出从高到低排序）：

   ```bash cURL
   curl "https://api.anthropic.com/v1/organizations/analytics/user_cost_report?starting_at=2026-06-01T00:00:00Z&limit=1000" \
     --header "x-api-key: $ANALYTICS_API_KEY"
   ```

   每行携带 `actor.user_id`、`actor.email` 和 `amount`（成员的支出，以美分计）。通过 `next_page` 翻页以覆盖整个组织。

2. 对于支出最高的成员（或所有超过某个美元阈值的成员），分批获取有效上限：

   ```bash cURL
   curl --globoff "https://api.anthropic.com/v1/organizations/spend_limits/effective?user_ids[]=user_01Ab...&user_ids[]=user_01Cd...&limit=100" \
     --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
   ```

   每行以 `amount` 返回上限（`null` = 无限制，`"0"` = 仅限所含用量），同时返回 `period_to_date_spend`。

3. 对于每位上限为正数的成员，计算 `period_to_date_spend / amount`，并标记达到或超过您阈值（例如 80%）的成员。将 `"0"` 上限视为已达限额。服务器端没有针对此比率的筛选器。

4. 对被标记的成员采取行动：使用 `POST /v1/organizations/spend_limits` 提高上限，如果存在待处理的提高请求则批准它，或联系该成员。

### 查找用量快速变化的成员

找出支出环比上周大幅跃升的成员。

1. 从 Analytics API 拉取过去两周每位成员的每日成本：

   ```bash cURL
   curl "https://api.anthropic.com/v1/organizations/analytics/user_cost_report?starting_at=2026-06-09T00:00:00Z&ending_at=2026-06-23T00:00:00Z&bucket_width=1d&limit=1000" \
     --header "x-api-key: $ANALYTICS_API_KEY"
   ```

   设置 `bucket_width` 后，每位成员在有用量的每一天各占一行；通过 `next_page` 翻页以收集每位成员的完整序列。

2. 按 `actor.user_id` 对行进行分组。对于每位成员，分别汇总最近七天和之前七天的数据。标记最近一周超过前一周达到您所选倍数（例如三倍）的成员。最近几天的成本是暂定的，可能会被向上修正；为了进行可重复的比较，请将 `ending_at` 设置为不晚于先前返回的 `data_refreshed_at`（请参阅[数据可用性与新鲜度](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api#data-availability-and-freshness)）。

3. 对被标记的成员采取行动：使用 `POST /v1/organizations/spend_limits` 调整上限，或联系该成员。

### 在事件期间临时提高成员的支出限额

在事件处于开放状态时为事件响应人员提供工作空间：在事件开始时提高其支出上限，并在事件关闭后回滚。请以您的事件管理系统作为提高上限的门控条件，例如要求提供一个该成员已被分配到的有效事件 ID。

1. 读取成员当前的上限，并记录下来以便回滚：

   ```bash cURL
   curl --globoff "https://api.anthropic.com/v1/organizations/spend_limits/effective?user_ids[]=user_01AbCdEfGh&period[]=monthly" \
     --header "x-api-key: $ANTHROPIC_ADMIN_KEY"
   ```

2. 提高上限：

   ```bash cURL
   curl --request POST "https://api.anthropic.com/v1/organizations/spend_limits" \
     --header "content-type: application/json" \
     --header "x-api-key: $ANTHROPIC_ADMIN_KEY" \
     --data '{"scope": {"type": "user", "user_id": "user_01AbCdEfGh"}, "amount": "500000", "period": "monthly"}'
   ```

3. 如果响应人员在事件期间需要更广泛的访问权限，请预先配置一个事件响应人员组，其[自定义角色](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#custom-roles)授予该权限，并在事件持续期间将该成员加入其中：

   ```bash cURL
   curl --request POST "https://api.anthropic.com/v1/organizations/rbac_groups/rbac_group_01UvWxYzAbCdEfGhIjKlMn/members" \
     --header "content-type: application/json" \
     --header "x-api-key: $ANTHROPIC_ADMIN_KEY" \
     --data '{"user_id": "user_01AbCdEfGh"}'
   ```

   有关组端点，请参阅[用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management#groups)。

4. 当您的事件系统将事件标记为已关闭时，回滚这两项更改：恢复您在步骤 1 中记录的支出限额（如果该成员原本没有覆盖值，则使用 `DELETE /v1/organizations/spend_limits/{spend_limit_id}` 删除覆盖值），并使用 `DELETE /v1/organizations/rbac_groups/{group_id}/members/{user_id}` 将该成员从组中移除。

## 常见问题

### 直接设置支出限额会解决成员的待处理提高请求吗？

不会。`POST /v1/organizations/spend_limits` 会写入覆盖值，但不会触及待处理请求。请使用 `POST /v1/organizations/spend_limit_increase_requests/{id}/approve` 在一次调用中解决请求并写入覆盖值。

### 删除按用户的覆盖值后会发生什么？

该成员会回退到他们从层级中继承的任何值：其组、席位层级或组织默认值。如果任何级别都不存在默认值，则该成员无限制。

### 我可以通过此 API 设置席位层级或组织范围的默认值吗？

不可以。只有按用户的覆盖值可以通过此 API 写入。席位层级、组和组织级别的默认值在 claude.ai 组织设置中配置。

### 为什么活跃成员的 `period_to_date_spend` 有时显示为 `"0"`？

支出读数可能暂时不可用，在这种情况下该字段显示为 `"0"` 而不是报错。请将其视为参考信息。

## 另请参阅

<CardGroup cols={2}>
  <Card title="支出限额 API 参考" href="https://platform.claude.com/docs/zh-CN/api/admin/spend_limits">
    每个支出限额 API 端点的生成请求和响应模式。
  </Card>

  <Card title="支出限额提高请求 API 参考" href="https://platform.claude.com/docs/zh-CN/api/admin/spend_limits/increase_requests">
    提高请求端点的生成请求和响应模式。
  </Card>

  <Card title="Analytics APIs" href="https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api">
    面向 Claude Enterprise 的按用户和按时间分桶的用量与成本报告。
  </Card>
</CardGroup>
