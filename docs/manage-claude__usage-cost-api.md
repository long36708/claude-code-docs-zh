---
title: 用量和成本 API
url: https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api
description: 使用 Usage & Cost Admin API 以编程方式访问您组织的 API 用量和成本数据。
---

<Tip>
  **Admin API 不适用于个人账户。** 要与团队成员协作并添加成员，请在 **Console → Settings → Organization** 中设置您的组织。
</Tip>

Usage & Cost Admin API 提供对您组织历史 API 用量和成本数据的编程化和细粒度访问。这些数据类似于 Claude Console 中[用量](https://platform.claude.com/usage)和[成本](https://platform.claude.com/cost)页面提供的信息。

此 API 使您能够更好地监控、分析和优化您的 Claude 实现：

* **准确的用量跟踪：** 获取精确的令牌计数和用量模式，而不是仅依赖响应令牌计数
* **成本对账：** 为财务和会计团队将内部记录与 Anthropic 账单进行匹配
* **产品性能和改进：** 监控产品性能，同时衡量系统更改是否改善了它，或设置警报
* **[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)优化：** 优化[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)或特定提示等功能，以充分利用您分配的容量。
* **高级分析：** 执行比 Console 中可用的更深入的数据分析

<Check>
  **需要 Admin API 凭证。** 这些端点属于 Admin API 的一部分。您可以使用 [Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)、具有 `org:admin` 作用域的 OAuth 令牌，或未限定于某个工作区的个人或服务账户密钥来访问它们；工作区 API 密钥无法使用。详情请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#authentication)。
</Check>

Claude Enterprise 组织使用 Analytics API 密钥配合不同的 API；请参阅[您需要哪个 API？](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api#which-api-do-you-need)。

<Note>
  **AWS 上的 Claude Platform：** 编程化的 Usage 和 Cost API 端点目前不可用。请改为在 Claude Console 的 **Usage** 和 **Cost** 页面查看用量和成本数据。
</Note>

## 您需要哪个 API？

Anthropic 通过两个 API 提供成本和用量报告，具体取决于您的组织管理哪个 Claude 产品：

| 您的组织                             | API                                                                                                   | 密钥类型                                                                                                                                  |
| -------------------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Claude Console (Claude Platform) | 本页描述的 Usage 和 Cost Admin API                                                                          | Admin API 密钥 (`sk-ant-admin01-...`) 或其他 [Admin API 凭证](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#authentication) |
| Claude Enterprise (claude.ai)    | [Claude Enterprise Analytics API](https://platform.claude.com/docs/zh-CN/api/admin/analytics) 成本和用量端点 | Analytics API 密钥                                                                                                                      |

Claude Enterprise 父组织不会出现在 Claude Console 中，也不携带 Admin API 密钥，因此对它们而言，Analytics API 密钥是访问此数据的唯一途径。请参阅 [Analytics APIs](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api) 了解如何创建每种密钥类型以及 Claude Enterprise 成本数据适用于哪些计划。

## 合作伙伴解决方案

领先的可观测性平台提供即用型集成，用于监控您的 Claude API 用量和成本，无需编写自定义代码。这些集成提供仪表板、警报和分析，帮助您有效管理 API 用量。

<CardGroup cols={3}>
  <Card title="CloudZero" icon="chart" href="https://docs.cloudzero.com/docs/connections-anthropic">
    用于跟踪和预测成本的云智能平台
  </Card>

  <Card title="Datadog" icon="chart" href="https://docs.datadoghq.com/integrations/anthropic/">
    具有自动追踪和监控的 LLM 可观测性
  </Card>

  <Card title="Grafana Cloud" icon="chart" href="https://grafana.com/docs/grafana-cloud/monitor-infrastructure/integrations/integration-reference/integration-anthropic/">
    无代理集成，通过开箱即用的仪表板和警报实现轻松的 LLM 可观测性
  </Card>

  <Card title="Harness" icon="chart" href="https://developer.harness.io/docs/cloud-cost-management/provider-integrations/ai-providers/anthropic/">
    用于云和 AI 成本管理的 FinOps 平台
  </Card>

  <Card title="Honeycomb" icon="polygon" href="https://docs.honeycomb.io/integrations/anthropic-usage-monitoring/">
    通过 OpenTelemetry 进行高级查询和可视化
  </Card>

  <Card title="Vantage" icon="chart" href="https://docs.vantage.sh/connecting_anthropic">
    用于 LLM 成本和用量可观测性的 FinOps 平台
  </Card>
</CardGroup>

## 快速开始

获取您组织过去 7 天的每日用量：

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2025-01-08T00:00:00Z&\
ending_at=2025-01-15T00:00:00Z&\
bucket_width=1d" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

<Tip>
  **为集成设置 User-Agent 标头**

  如果您正在构建集成，请设置您的 User-Agent 标头，以帮助 Anthropic 了解用量模式：

  ```text wrap
  User-Agent: YourApp/1.0.0 (https://yourapp.com)
  ```
</Tip>

## Usage API

使用 `/v1/organizations/usage_report/messages` 端点跟踪您组织的令牌消耗，并按模型、工作区和服务层级进行详细细分。

### 关键概念

* **时间桶：** 以固定间隔（`1m`、`1h` 或 `1d`）聚合用量数据
* **令牌跟踪：** 测量未缓存输入、缓存输入、缓存创建和输出令牌
* **过滤和分组：** 按 API 密钥、工作区、模型、服务层级、上下文窗口、[数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)或速度（beta）进行过滤，并按这些维度对结果进行分组
* **服务器工具使用：** 跟踪服务器端工具（如网络搜索）的使用情况

有关完整的参数详情和响应架构，请参阅 [Usage API 参考](https://platform.claude.com/docs/zh-CN/api/admin-api/usage-cost/get-messages-usage-report)。

### 基本示例

#### 按模型的每日用量

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2025-01-01T00:00:00Z&\
ending_at=2025-01-08T00:00:00Z&\
group_by[]=model&\
bucket_width=1d" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

#### 带过滤的每小时用量

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2025-01-15T00:00:00Z&\
ending_at=2025-01-15T23:59:59Z&\
models[]=claude-opus-5&\
service_tiers[]=batch&\
context_window[]=0-200k&\
bucket_width=1h" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

#### 按 API 密钥和工作区过滤用量

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2025-01-01T00:00:00Z&\
ending_at=2025-01-08T00:00:00Z&\
api_key_ids[]=apikey_01Rj2N8SVvo6BePZj99NhmiT&\
api_key_ids[]=apikey_01ABC123DEF456GHI789JKL&\
workspace_ids[]=wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ&\
workspace_ids[]=wrkspc_01XYZ789ABC123DEF456MNO&\
bucket_width=1d" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

<Tip>
  要检索您组织的 API 密钥 ID，请使用 [List API Keys](https://platform.claude.com/docs/zh-CN/api/admin-api/apikeys/list-api-keys) 端点。

  要检索您组织的工作区 ID，请使用 [List Workspaces](https://platform.claude.com/docs/zh-CN/api/admin-api/workspaces/list-workspaces) 端点，或在 Claude Console 中查找您组织的工作区 ID。
</Tip>

#### 数据驻留

通过使用 `inference_geo` 维度对用量进行分组和过滤，跟踪您的[数据驻留控制](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)。这对于验证您组织的地理路由非常有用。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2026-02-01T00:00:00Z&\
ending_at=2026-02-08T00:00:00Z&\
group_by[]=inference_geo&\
group_by[]=model&\
bucket_width=1d" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

您还可以过滤到特定地理位置。有效值为 `global`、`us` 和 `not_available`：

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2026-02-01T00:00:00Z&\
ending_at=2026-02-08T00:00:00Z&\
inference_geos[]=us&\
group_by[]=model&\
bucket_width=1d" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

<Note>
  2026 年 2 月之前发布的模型（早于 Claude Opus 4.6 和 Claude Sonnet 4.6）不支持 `inference_geo` 请求参数，因此它们的用量报告在此维度上返回 `"not_available"`。您可以在 `inference_geos[]` 中使用 `not_available` 作为过滤值来定位这些模型。
</Note>

#### 快速模式（研究预览）

通过使用 `speed` 维度进行分组和过滤，跟踪[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)用量。这对于监控标准模式与快速模式的用量非常有用。

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2026-02-01T00:00:00Z&\
ending_at=2026-02-08T00:00:00Z&\
group_by[]=speed&\
group_by[]=model&\
bucket_width=1d" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: fast-mode-2026-02-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

您还可以过滤到特定速度。有效值为 `standard` 和 `fast`：

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2026-02-01T00:00:00Z&\
ending_at=2026-02-08T00:00:00Z&\
speeds[]=fast&\
group_by[]=model&\
bucket_width=1d" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: fast-mode-2026-02-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

<Note>
  `speeds[]` 过滤器和 `speed` group\_by 值都需要 `fast-mode-2026-02-01` beta 标头。
</Note>

### 时间粒度限制

| 粒度   | 默认限制  | 最大限制     | 用例      |
| ---- | ----- | -------- | ------- |
| `1m` | 60 个桶 | 1,440 个桶 | 实时监控    |
| `1h` | 24 个桶 | 168 个桶   | 每日模式    |
| `1d` | 7 个桶  | 31 个桶    | 每周/每月报告 |

## Cost API

使用 `/v1/organizations/cost_report` 端点检索以美元为单位的服务级成本细分。

### 关键概念

* **货币：** 所有成本以美元计，以最低单位（美分）的十进制字符串报告
* **成本类型：** 跟踪令牌用量、网络搜索和代码执行成本
* **分组：** 按工作区或描述对成本进行分组以获得详细细分。按 `description` 分组时，响应包括解析后的字段，如 `model` 和 `inference_geo`
* **时间桶：** 仅支持每日粒度（`1d`）

有关完整的参数详情和响应架构，请参阅 [Cost API 参考](https://platform.claude.com/docs/zh-CN/api/admin-api/usage-cost/get-cost-report)。

<Warning>
  Priority Tier 成本使用不同的计费模型，不包含在成本端点中。请改为通过用量端点跟踪 Priority Tier 用量。
</Warning>

### 基本示例

```bash cURL
curl "https://api.anthropic.com/v1/organizations/cost_report?\
starting_at=2025-01-01T00:00:00Z&\
ending_at=2025-01-31T00:00:00Z&\
group_by[]=workspace_id&\
group_by[]=description" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

## 分页

两个端点都支持对大型数据集进行分页：

1. 发出初始请求。
2. 如果 `has_more` 为 `true`，请在下一个请求中使用 `next_page` 值。
3. 继续直到 `has_more` 为 `false`。

```bash cURL
# 第一个请求
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2025-01-01T00:00:00Z&\
ending_at=2025-01-31T00:00:00Z&\
limit=7" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"

# 响应包含："has_more": true, "next_page": "page_xyz..."

# 带分页的下一个请求
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2025-01-01T00:00:00Z&\
ending_at=2025-01-31T00:00:00Z&\
limit=7&\
page=page_xyz..." \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

## 常见用例

在 [Claude Cookbook](https://platform.claude.com/cookbook) 中探索详细的实现：

* **每日用量报告：** 跟踪令牌消耗趋势
* **成本归因：** 按工作区分配费用以进行分摊
* **缓存效率：** 测量和优化提示缓存
* **预算监控：** 为支出阈值设置警报
* **CSV 导出：** 为财务团队生成报告

## 常见问题

### 数据的新鲜度如何？

用量和成本数据通常在 API 请求完成后 5 分钟内出现，但延迟有时可能更长。

### 推荐的轮询频率是多少？

该 API 支持每分钟轮询一次以供持续使用。对于短时突发（例如，下载分页数据），可以接受更频繁的轮询。对于需要频繁更新的仪表板，请缓存结果。

### 如何跟踪代码执行用量？

代码执行成本出现在成本端点中，在描述字段下归类为 `Code Execution Usage`。代码执行不包含在用量端点中。

### 如何跟踪 Priority Tier 用量？

在用量端点中按 `service_tier` 过滤或分组，并查找 `priority` 值。Priority Tier 成本在成本端点中不可用。

### playground 用量会发生什么？

来自 Claude Console 中 playground（以及之前的旧版 Workbench）的 API 用量不与 API 密钥关联，因此即使按该维度分组，`api_key_id` 也将为 `null`。

### 默认工作区如何表示？

归因于默认工作区的用量和成本的 `workspace_id` 值为 `null`。

### 如何获取 Claude Code 的每用户成本细分？

使用 [Claude Code Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/claude-code-analytics-api)，它提供每用户的估计成本和生产力指标，而不会受到按许多 API 密钥细分成本的性能限制。对于使用许多密钥的一般 API 用量，请使用 [Usage API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api#usage-api) 跟踪令牌消耗作为成本代理。

## 另请参阅

使用 Usage 和 Cost API 为您的用户提供更好的体验、管理成本并保护您的速率限制。了解有关其他一些功能的更多信息：

* [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)
* [Admin API 参考](https://platform.claude.com/docs/zh-CN/api/admin)
* [Analytics APIs](https://platform.claude.com/docs/zh-CN/manage-claude/analytics-api) - 您的组织需要哪个分析 API 和密钥类型
* [定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)
* [提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) - 通过缓存优化成本
* [批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing) - 批量请求享受 50% 折扣
* [速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits) - 了解用量层级
* [Rate Limits API](https://platform.claude.com/docs/zh-CN/manage-claude/rate-limits-api) - 读取您配置的速率限制
* [数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency) - 控制推理地理位置
