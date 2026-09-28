---
title: Claude Code 分析 API
url: https://platform.claude.com/docs/zh-CN/manage-claude/claude-code-analytics-api
description: 通过 Claude Code 分析 Admin API，以编程方式访问您组织的 Claude Code 使用分析和生产力指标。
---

<Tip>
  **Admin API 不适用于个人账户。** 要与团队成员协作并添加成员，请在 **Console → Settings → Organization** 中设置您的组织。
</Tip>

Claude Code 分析 Admin API 提供对 Claude Code 用户每日汇总使用指标的编程访问，使组织能够分析开发者生产力并构建自定义仪表板。与基础的[分析仪表板](https://platform.claude.com/claude-code)相比，此 API 提供更多详细信息，同时又没有 OpenTelemetry 集成那样复杂。

此 API 可帮助您更好地监控、分析和优化 Claude Code 的采用情况：

* **开发者生产力分析：** 跟踪使用 Claude Code 的会话数、添加/删除的代码行数、提交数以及创建的拉取请求数
* **工具使用指标：** 监控不同 Claude Code 工具（Edit、MultiEdit、Write、NotebookEdit）的接受率和拒绝率
* **成本分析：** 查看按 Claude 模型细分的预估成本和 "token"（令牌）使用量
* **自定义报告：** 导出数据，为管理团队构建高管仪表板和报告
* **使用论证：** 提供指标，以便在内部论证并扩大 Claude Code 的采用

<Check>
  **需要 Admin API 凭证。** 这些端点属于 Admin API 的一部分。您可以使用 [Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)、具有 `org:admin` 作用域的 OAuth 令牌，或未限定于某个工作区的个人或服务账户密钥来访问它们；工作区 API 密钥无法使用。详情请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#authentication)。
</Check>

<Note>
  **Claude Platform on AWS：** Claude Code 分析 API 目前不可用。请改为在 Claude Console 的 **Usage** 页面上查看 Claude Code 使用情况。
</Note>

<Note>
  **Claude Enterprise 组织：** claude.ai 用户的 Claude Code 活动由 Claude Enterprise 分析 API 报告，该 API 使用分析 API 密钥而非 Admin API 密钥。请参阅[分析 API](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api)，了解您的组织需要哪种 API 和密钥类型。
</Note>

## 快速入门

获取您组织在特定日期的 Claude Code 分析数据：

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/claude_code?\
starting_at=2025-09-08&\
limit=20" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

<Tip>
  **为集成设置 User-Agent 标头**

  如果您正在构建集成，请设置 User-Agent 标头，以帮助 Anthropic 了解使用模式：

  ```text wrap
  User-Agent: YourApp/1.0.0 (https://yourapp.com)
  ```
</Tip>

## Claude Code 分析 API

使用 `/v1/organizations/usage_report/claude_code` 端点跟踪整个组织的 Claude Code 使用情况、生产力指标和开发者活动。

### 关键概念

* **每日汇总：** 返回由 `starting_at` 参数指定的单日指标
* **用户级数据：** 每条记录代表一个用户在指定日期的活动
* **生产力指标：** 跟踪会话、代码行数、提交、拉取请求和工具使用情况
* **令牌和成本数据：** 监控按 Claude 模型细分的使用量和预估成本
* **"Cursor-based pagination"（基于游标的分页）：** 使用不透明游标进行稳定分页，以处理大型数据集
* **数据新鲜度：** 为保证一致性，指标最多有 1 小时的延迟

有关完整的参数详情和响应架构，请参阅 [Claude Code 分析 API 参考](https://platform.claude.com/docs/zh-CN/api/beta/organization/usage_report/retrieve_claude_code)。

### 基本示例

#### 获取特定日期的分析数据

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/claude_code?\
starting_at=2025-09-08" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

#### 使用分页获取分析数据

```bash cURL
# 第一个请求
curl "https://api.anthropic.com/v1/organizations/usage_report/claude_code?\
starting_at=2025-09-08&\
limit=20" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"

# 后续请求，使用响应中返回的 cursor
curl "https://api.anthropic.com/v1/organizations/usage_report/claude_code?\
starting_at=2025-09-08&\
page=page_MjAyNS0wNS0xNFQwMDowMDowMFo=" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

### 请求参数

| 参数            | 类型      | 必需 | 描述                              |
| ------------- | ------- | -- | ------------------------------- |
| `starting_at` | string  | 是  | YYYY-MM-DD 格式的 UTC 日期；仅返回该单日的指标 |
| `limit`       | integer | 否  | 每页记录数（默认：20，最大：1000）            |
| `page`        | string  | 否  | 来自上一个响应 `next_page` 字段的不透明游标令牌  |

### 可用指标

每条响应记录包含单个用户在单日内的以下指标：

#### 维度

* **date：** RFC 3339 格式的日期（UTC 时间戳）
* **actor：** 执行 Claude Code 操作的用户或 API 密钥（带有 `email_address` 的 `user_actor`，或带有 `api_key_name` 的 `api_actor`）
* **organization\_id：** 组织 UUID
* **customer\_type：** 客户账户类型（`api` 表示 API 客户，`subscription` 表示 Pro/Team 客户）
* **terminal\_type：** 使用 Claude Code 的终端或环境类型（例如 `vscode`、`iTerm.app`、`tmux`）

#### 核心指标

* **num\_sessions：** 此 actor 发起的不同 Claude Code 会话数
* **lines\_of\_code.added：** Claude Code 在所有文件中添加的代码总行数
* **lines\_of\_code.removed：** Claude Code 在所有文件中删除的代码总行数
* **commits\_by\_claude\_code：** 通过 Claude Code 的提交功能创建的 git 提交数
* **pull\_requests\_by\_claude\_code：** 通过 Claude Code 的 PR 功能创建的拉取请求数

#### 工具操作指标

按工具类型细分的工具操作接受率和拒绝率：

* **edit\_tool.accepted/rejected：** 用户接受/拒绝的 Edit 工具提议数
* **multi\_edit\_tool.accepted/rejected：** 用户接受/拒绝的 MultiEdit 工具提议数
* **write\_tool.accepted/rejected：** 用户接受/拒绝的 Write 工具提议数
* **notebook\_edit\_tool.accepted/rejected：** 用户接受/拒绝的 NotebookEdit 工具提议数

#### 模型细分

对于使用的每个 Claude 模型：

* **model：** Claude 模型标识符（例如 `claude-opus-5`）
* **tokens.input/output：** 此模型的输入和输出令牌数
* **tokens.cache\_read/cache\_creation：** 此模型与缓存相关的令牌使用量
* **estimated\_cost.amount：** 此模型的预估成本，以美分（USD）计
* **estimated\_cost.currency：** 成本金额的货币代码（目前始终为 `USD`）

### 响应结构

API 以以下格式返回数据：

```json
{
  "data": [
    {
      "date": "2025-09-08T00:00:00Z",
      "actor": {
        "type": "user_actor",
        "email_address": "developer@company.com"
      },
      "organization_id": "dc9f6c26-b22c-4831-8d01-0446bada88f1",
      "customer_type": "api",
      "terminal_type": "vscode",
      "core_metrics": {
        "num_sessions": 5,
        "lines_of_code": {
          "added": 1543,
          "removed": 892
        },
        "commits_by_claude_code": 12,
        "pull_requests_by_claude_code": 2
      },
      "tool_actions": {
        "edit_tool": {
          "accepted": 45,
          "rejected": 5
        },
        "multi_edit_tool": {
          "accepted": 12,
          "rejected": 2
        },
        "write_tool": {
          "accepted": 8,
          "rejected": 1
        },
        "notebook_edit_tool": {
          "accepted": 3,
          "rejected": 0
        }
      },
      "model_breakdown": [
        {
          "model": "claude-opus-5-5",
          "tokens": {
            "input": 100000,
            "output": 35000,
            "cache_read": 10000,
            "cache_creation": 5000
          },
          "estimated_cost": {
            "currency": "USD",
            "amount": 113
          }
        }
      ]
    }
  ],
  "has_more": false,
  "next_page": null
}
```

## 分页

对于拥有大量用户的组织，API 支持基于游标的 "pagination"（分页）：

1. 发出初始请求，可选择提供 `limit` 参数。
2. 如果响应中的 `has_more` 为 `true`，请在下一个请求中使用 `next_page` 值。
3. 持续请求，直到 `has_more` 为 `false`。

游标编码了最后一条记录的位置，即使有新数据到达，也能确保分页稳定。每个分页会话都保持一致的数据边界，以确保您不会遗漏或重复记录。

## 常见用例

* **高管仪表板：** 创建高层级报告，展示 Claude Code 对开发速度的影响
* **AI 工具比较：** 导出指标，将 Claude Code 与 Copilot 和 Cursor 等其他 AI 编码工具进行比较
* **开发者生产力分析：** 跟踪个人和团队随时间变化的生产力指标
* **成本跟踪和分配：** 监控支出模式，并按团队或项目分配成本
* **采用情况监控：** 识别哪些团队和用户从 Claude Code 中获得了最大价值
* **ROI 论证：** 提供具体指标，以便在内部论证并扩大 Claude Code 的采用

## 常见问题

### 分析数据的新鲜度如何？

Claude Code 分析数据通常在用户活动完成后 1 小时内出现。为确保分页结果一致，响应中仅包含超过 1 小时的数据。

### 我可以获取实时指标吗？

不可以，此 API 仅提供每日汇总指标。如需实时监控，请考虑使用 [OpenTelemetry 集成](https://code.claude.com/docs/zh-CN/monitoring-usage)。

### 数据中如何识别用户？

用户通过 `actor` 字段以两种方式识别：

* **`user_actor`：** 包含通过 OAuth 进行身份验证的用户的 `email_address`（最常见）
* **`api_actor`：** 包含使用 API 密钥进行身份验证的用户的 `api_key_name`

`customer_type` 字段表示使用量来自 `api` 客户（按需付费 API）还是 `subscription` 客户（Pro/Team 计划）。

### 数据保留期是多久？

历史 Claude Code 分析数据会被保留，并可通过 API 访问。此数据没有指定的删除期限。

### 支持哪些 Claude Code 部署？

此 API 仅跟踪 Claude API 上的 Claude Code 使用情况。通过 [Amazon Bedrock 中的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)、[Microsoft Foundry 中的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)、[Google Cloud 上的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 或 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 产生的使用量不包括在内。

### 使用此 API 需要多少费用？

对于所有有权访问 Admin API 的组织，Claude Code 分析 API 均可免费使用。

### 如何计算工具接受率？

对于每种工具类型，工具接受率 = `accepted / (accepted + rejected)`。例如，如果 edit 工具显示 45 次接受和 5 次拒绝，则接受率为 90%。

### 日期参数使用什么时区？

所有日期均为 UTC。`starting_at` 参数应采用 YYYY-MM-DD 格式，表示该日的 UTC 午夜。

## 另请参阅

Claude Code 分析 API 可帮助您了解并优化团队的开发工作流程。了解更多相关功能：

* [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)
* [Admin API 参考](https://platform.claude.com/docs/zh-CN/api/beta/organization)
* [Claude Code 分析仪表板](https://platform.claude.com/claude-code)
* [使用量和成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api) - 跟踪所有 Anthropic 服务的 API 使用情况
* [合规 API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) - 检索审计和活动数据
* [身份和访问管理](https://code.claude.com/docs/en/iam)
* [使用 OpenTelemetry 监控使用情况](https://code.claude.com/docs/zh-CN/monitoring-usage)，用于自定义指标和告警
