---
title: 速率限制 API
url: https://platform.claude.com/docs/zh-CN/manage-claude/rate-limits-api
description: 使用速率限制 API 以编程方式查询您组织的 API 速率限制。
---

<Tip>
  **Admin API 不适用于个人账户。** 要与团队成员协作并添加成员，请在 **Console → Settings → Organization** 中设置您的组织。
</Tip>

速率限制 API（Rate Limits API）提供对为您的组织及其工作区配置的 "rate limit"（速率限制）的编程访问。这与 Claude Console 中[速率限制](https://platform.claude.com/settings/limits)页面上显示的信息相同。

使用此 API 可以：

* **保持网关和代理同步：** 在启动时和按计划读取您当前的限制，而不是硬编码那些在 Anthropic 调整后会产生偏差的值。
* **支持内部告警：** 将来自[使用量和成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api) 的使用数据与您配置的限制进行比较。
* **审计工作区配置：** 验证工作区覆盖设置是否与您的配置自动化所期望的一致。

<Check>
  **需要 Admin API 凭证。** 这些端点属于 Admin API 的一部分。您可以使用 [Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)、具有 `org:admin` 作用域的 OAuth 令牌，或未限定于某个工作区的个人或服务账户密钥来访问它们；工作区 API 密钥无法使用。详情请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#authentication)。
</Check>

本页面上的 SDK 和 CLI 示例构造默认客户端，该客户端从 `ANTHROPIC_API_KEY` 环境变量中读取 Admin API 密钥。SDK 将这些端点公开为 `client.beta.organization.rate_limits` 和 `client.beta.organization.workspaces.rate_limits`；Python、TypeScript、C#、Go 和 Java 的 list 方法返回一个会自动为您跟随 `next_page` 的迭代器，而 PHP、Ruby 和 curl 示例只读取一页。

## 快速开始

列出为您的组织配置的速率限制：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/rate_limits" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:rate-limits list
  ```

  ```python Python
  client = anthropic.Anthropic()

  rate_limits = client.beta.organization.rate_limits.list()

  for group in rate_limits:
      models = f" ({', '.join(group.models)})" if group.models else ""
      print(f"{group.group_type}{models}")
      for limit in group.limits:
          print(f"  {limit.type}: {limit.value}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const rateLimits = await client.beta.organization.rateLimits.list();

  for await (const group of rateLimits) {
    const models = group.models ? ` (${group.models.join(", ")})` : "";
    console.log(`${group.group_type}${models}`);
    for (const limit of group.limits) {
      console.log(`  ${limit.type}: ${limit.value}`);
    }
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var rateLimits = await client.Beta.Organization.RateLimits.List();

  await foreach (var group in rateLimits.Paginate())
  {
      var models = group.Models is null ? "" : $" ({string.Join(", ", group.Models)})";
      Console.WriteLine($"{group.GroupType.Raw()}{models}");
      foreach (var limit in group.Limits)
      {
          Console.WriteLine($"  {limit.Type}: {limit.Value}");
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  rateLimits := client.Beta.Organization.RateLimits.ListAutoPaging(context.Background(), anthropic.BetaOrganizationRateLimitListParams{})

  for rateLimits.Next() {
  	group := rateLimits.Current()
  	models := ""
  	if len(group.Models) > 0 {
  		models = fmt.Sprintf(" (%s)", strings.Join(group.Models, ", "))
  	}
  	fmt.Printf("%s%s\n", group.GroupType, models)
  	for _, limit := range group.Limits {
  		fmt.Printf("  %s: %d\n", limit.Type, limit.Value)
  	}
  }
  if err := rateLimits.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var rateLimits = client.beta().organization().rateLimits().list();

  for (var group : rateLimits.autoPager()) {
      var models = group.models()
          .map(modelIds -> " (" + String.join(", ", modelIds) + ")")
          .orElse("");
      IO.println(group.groupType().asString() + models);
      for (var limit : group.limits()) {
          IO.println("  " + limit.type() + ": " + limit.value());
      }
  }
  ```

  ```php PHP
  $client = new Client();

  $rateLimits = $client->beta->organization->rateLimits->list();

  foreach ($rateLimits->data as $group) {
      $models = $group->models ? ' (' . implode(', ', $group->models) . ')' : '';
      echo "{$group->groupType}{$models}\n";
      foreach ($group->limits as $limit) {
          echo "  {$limit->type}: {$limit->value}\n";
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  rate_limits = client.beta.organization.rate_limits.list

  rate_limits.data.each do |group|
    models = group.models ? " (#{group.models.join(", ")})" : ""
    puts "#{group.group_type}#{models}"
    group.limits.each do |limit|
      puts "  #{limit.type}: #{limit.value}"
    end
  end
  ```
</CodeGroup>

## 组织速率限制

`/v1/organizations/rate_limits` 端点返回在组织级别应用于 Messages API 及其支持资源的速率限制。其他产品（例如 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)）的限制不包括在内。

### 关键概念

* **速率限制组：** 响应中的每个条目代表一个速率限制组。模型速率限制被分组，以便多个模型版本共享同一组限制，其他组则涵盖诸如 Message Batches API、Files API、Token Counting API、agent skills 和网页搜索工具等资源。
* **`group_type`：** 标识该条目涵盖哪一类限制。有关值的列表，请参阅[按组类型筛选](https://platform.claude.com/docs/zh-CN/manage-claude/rate-limits-api#filtering-by-group-type)。
* **`models` 列表：** 对于 `model_group` 条目，`models` 字段列出计入该组限制的每个模型 ID 和别名。使用此列表可查找任意模型字符串属于哪个组。对于其他组类型，`models` 为 `null`。
* **`limits` 列表：** 每个组都带有一个 `{type, value}` 对的列表。`type` 字段标识限制器（例如 `requests_per_minute`、`input_tokens_per_minute` 或 `output_tokens_per_minute`），`value` 是配置的限制值。有关每个限制器如何计量和执行，请参阅[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)。

有关完整的参数详情和响应模式，请参阅[组织速率限制 API 参考](https://platform.claude.com/docs/zh-CN/api/admin/rate_limits/list)。

### 列出所有组织速率限制

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/rate_limits" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:rate-limits list
  ```

  ```python Python
  client = anthropic.Anthropic()

  rate_limits = client.beta.organization.rate_limits.list()

  for group in rate_limits:
      models = f" ({', '.join(group.models)})" if group.models else ""
      print(f"{group.group_type}{models}")
      for limit in group.limits:
          print(f"  {limit.type}: {limit.value}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const rateLimits = await client.beta.organization.rateLimits.list();

  for await (const group of rateLimits) {
    const models = group.models ? ` (${group.models.join(", ")})` : "";
    console.log(`${group.group_type}${models}`);
    for (const limit of group.limits) {
      console.log(`  ${limit.type}: ${limit.value}`);
    }
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var rateLimits = await client.Beta.Organization.RateLimits.List();

  await foreach (var group in rateLimits.Paginate())
  {
      var models = group.Models is null ? "" : $" ({string.Join(", ", group.Models)})";
      Console.WriteLine($"{group.GroupType.Raw()}{models}");
      foreach (var limit in group.Limits)
      {
          Console.WriteLine($"  {limit.Type}: {limit.Value}");
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  rateLimits := client.Beta.Organization.RateLimits.ListAutoPaging(context.Background(), anthropic.BetaOrganizationRateLimitListParams{})

  for rateLimits.Next() {
  	group := rateLimits.Current()
  	models := ""
  	if len(group.Models) > 0 {
  		models = fmt.Sprintf(" (%s)", strings.Join(group.Models, ", "))
  	}
  	fmt.Printf("%s%s\n", group.GroupType, models)
  	for _, limit := range group.Limits {
  		fmt.Printf("  %s: %d\n", limit.Type, limit.Value)
  	}
  }
  if err := rateLimits.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var rateLimits = client.beta().organization().rateLimits().list();

  for (var group : rateLimits.autoPager()) {
      var models = group.models()
          .map(modelIds -> " (" + String.join(", ", modelIds) + ")")
          .orElse("");
      IO.println(group.groupType().asString() + models);
      for (var limit : group.limits()) {
          IO.println("  " + limit.type() + ": " + limit.value());
      }
  }
  ```

  ```php PHP
  $client = new Client();

  $rateLimits = $client->beta->organization->rateLimits->list();

  foreach ($rateLimits->data as $group) {
      $models = $group->models ? ' (' . implode(', ', $group->models) . ')' : '';
      echo "{$group->groupType}{$models}\n";
      foreach ($group->limits as $limit) {
          echo "  {$limit->type}: {$limit->value}\n";
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  rate_limits = client.beta.organization.rate_limits.list

  rate_limits.data.each do |group|
    models = group.models ? " (#{group.models.join(", ")})" : ""
    puts "#{group.group_type}#{models}"
    group.limits.each do |limit|
      puts "  #{limit.type}: #{limit.value}"
    end
  end
  ```
</CodeGroup>

```json
{
  "data": [
    {
      "type": "rate_limit",
      "group_type": "model_group",
      "models": ["claude-opus-5"],
      "limits": [
        { "type": "requests_per_minute", "value": 4000 },
        { "type": "input_tokens_per_minute", "value": 10000000 },
        { "type": "output_tokens_per_minute", "value": 800000 }
      ]
    },
    {
      "type": "rate_limit",
      "group_type": "model_group",
      "models": [
        "claude-opus-4-5",
        "claude-opus-4-5-20251101",
        "claude-opus-4-6",
        "claude-opus-4-7",
        "claude-opus-4-8"
      ],
      "limits": [
        { "type": "requests_per_minute", "value": 4000 },
        { "type": "input_tokens_per_minute", "value": 10000000 },
        { "type": "output_tokens_per_minute", "value": 800000 }
      ]
    },
    {
      "type": "rate_limit",
      "group_type": "batch",
      "models": null,
      "limits": [{ "type": "enqueued_batch_requests", "value": 500000 }]
    }
  ],
  "next_page": null
}
```

### 查找特定模型的限制

将任意模型 ID 或别名作为 `model` 查询参数传入，即可仅返回包含该模型的条目：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/rate_limits?model=claude-opus-5" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:rate-limits list --model claude-opus-5
  ```

  ```python Python
  client = anthropic.Anthropic()

  rate_limits = client.beta.organization.rate_limits.list(model="claude-opus-5")

  for group in rate_limits:
      models = f" ({', '.join(group.models)})" if group.models else ""
      print(f"{group.group_type}{models}")
      for limit in group.limits:
          print(f"  {limit.type}: {limit.value}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const rateLimits = await client.beta.organization.rateLimits.list({ model: "claude-opus-5" });

  for await (const group of rateLimits) {
    const models = group.models ? ` (${group.models.join(", ")})` : "";
    console.log(`${group.group_type}${models}`);
    for (const limit of group.limits) {
      console.log(`  ${limit.type}: ${limit.value}`);
    }
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var rateLimits = await client.Beta.Organization.RateLimits.List(new()
  {
      Model = "claude-opus-5"
  });

  await foreach (var group in rateLimits.Paginate())
  {
      var models = group.Models is null ? "" : $" ({string.Join(", ", group.Models)})";
      Console.WriteLine($"{group.GroupType.Raw()}{models}");
      foreach (var limit in group.Limits)
      {
          Console.WriteLine($"  {limit.Type}: {limit.Value}");
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  rateLimits := client.Beta.Organization.RateLimits.ListAutoPaging(context.Background(), anthropic.BetaOrganizationRateLimitListParams{
  	Model: anthropic.String(anthropic.ModelClaudeOpus5),
  })

  for rateLimits.Next() {
  	group := rateLimits.Current()
  	models := ""
  	if len(group.Models) > 0 {
  		models = fmt.Sprintf(" (%s)", strings.Join(group.Models, ", "))
  	}
  	fmt.Printf("%s%s\n", group.GroupType, models)
  	for _, limit := range group.Limits {
  		fmt.Printf("  %s: %d\n", limit.Type, limit.Value)
  	}
  }
  if err := rateLimits.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.beta.organization.ratelimits.RateLimitListParams;
  import com.anthropic.models.messages.Model;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = RateLimitListParams.builder()
          .model(Model.CLAUDE_OPUS_5.asString())
          .build();
      var rateLimits = client.beta().organization().rateLimits().list(params);

      for (var group : rateLimits.autoPager()) {
          var models = group.models()
              .map(modelIds -> " (" + String.join(", ", modelIds) + ")")
              .orElse("");
          IO.println(group.groupType().asString() + models);
          for (var limit : group.limits()) {
              IO.println("  " + limit.type() + ": " + limit.value());
          }
      }
  }
  ```

  ```php PHP
  use Anthropic\Messages\Model;

  $client = new Client();

  $rateLimits = $client->beta->organization->rateLimits->list(
      model: Model::CLAUDE_OPUS_5->value,
  );

  foreach ($rateLimits->data as $group) {
      $models = $group->models ? ' (' . implode(', ', $group->models) . ')' : '';
      echo "{$group->groupType}{$models}\n";
      foreach ($group->limits as $limit) {
          echo "  {$limit->type}: {$limit->value}\n";
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  rate_limits = client.beta.organization.rate_limits.list(model: Anthropic::Model::CLAUDE_OPUS_5)

  rate_limits.data.each do |group|
    models = group.models ? " (#{group.models.join(", ")})" : ""
    puts "#{group.group_type}#{models}"
    group.limits.each do |limit|
      puts "  #{limit.type}: #{limit.value}"
    end
  end
  ```
</CodeGroup>

如果模型字符串不匹配任何组，该端点将返回 404 错误。`model` 参数仅在组织端点上受支持；工作区端点不接受该参数。

## 工作区速率限制

`/v1/organizations/workspaces/{workspace_id}/rate_limits` 端点返回为单个工作区配置的速率限制覆盖设置。

响应仅包含覆盖设置，因此其中缺失的任何内容均继承自组织：

* `data` 中不存在的组完全没有工作区覆盖设置。该工作区继承该组的组织级限制（并非无限制）。
* 在存在的组内，`limits[]` 中不存在的限制器类型没有针对该限制器的工作区覆盖设置。该工作区继承其组织值。
* 对于每个存在的限制器，`org_limit` 是同一限制器的组织级值；如果组织没有为该限制器类型配置限制，则为 `null`。

有关完整的参数详情和响应模式，请参阅[工作区速率限制 API 参考](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/rate_limits/list)。

<Tip>
  要获取您组织的工作区 ID，请使用[列出工作区](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/list)端点，或在 [Claude Console](https://platform.claude.com/settings/workspaces) 中查找。默认工作区不能设置速率限制覆盖，因此在此端点上没有条目；请使用组织端点读取其限制。
</Tip>

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/workspaces/wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ/rate_limits" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:workspaces:rate-limits list \
    --workspace-id wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ
  ```

  ```python Python
  client = anthropic.Anthropic()

  rate_limits = client.beta.organization.workspaces.rate_limits.list(
      "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  )

  for group in rate_limits:
      models = f" ({', '.join(group.models)})" if group.models else ""
      print(f"{group.group_type}{models}")
      for limit in group.limits:
          print(f"  {limit.type}: {limit.value}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const rateLimits = await client.beta.organization.workspaces.rateLimits.list(
    "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  );

  for await (const group of rateLimits) {
    const models = group.models ? ` (${group.models.join(", ")})` : "";
    console.log(`${group.group_type}${models}`);
    for (const limit of group.limits) {
      console.log(`  ${limit.type}: ${limit.value}`);
    }
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var rateLimits = await client.Beta.Organization.Workspaces.RateLimits.List(
      "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  );

  await foreach (var group in rateLimits.Paginate())
  {
      var models = group.Models is null ? "" : $" ({string.Join(", ", group.Models)})";
      Console.WriteLine($"{group.GroupType.Raw()}{models}");
      foreach (var limit in group.Limits)
      {
          Console.WriteLine($"  {limit.Type}: {limit.Value}");
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  rateLimits := client.Beta.Organization.Workspaces.RateLimits.ListAutoPaging(
  	context.Background(),
  	"wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
  	anthropic.BetaOrganizationWorkspaceRateLimitListParams{},
  )

  for rateLimits.Next() {
  	group := rateLimits.Current()
  	models := ""
  	if len(group.Models) > 0 {
  		models = fmt.Sprintf(" (%s)", strings.Join(group.Models, ", "))
  	}
  	fmt.Printf("%s%s\n", group.GroupType, models)
  	for _, limit := range group.Limits {
  		fmt.Printf("  %s: %d\n", limit.Type, limit.Value)
  	}
  }
  if err := rateLimits.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var rateLimits = client.beta().organization().workspaces().rateLimits()
      .list("wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ");

  for (var group : rateLimits.autoPager()) {
      var models = group.models()
          .map(modelIds -> " (" + String.join(", ", modelIds) + ")")
          .orElse("");
      IO.println(group.groupType().asString() + models);
      for (var limit : group.limits()) {
          IO.println("  " + limit.type() + ": " + limit.value());
      }
  }
  ```

  ```php PHP
  $client = new Client();

  $rateLimits = $client->beta->organization->workspaces->rateLimits->list(
      workspaceID: 'wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ',
  );

  foreach ($rateLimits->data as $group) {
      $models = $group->models ? ' (' . implode(', ', $group->models) . ')' : '';
      echo "{$group->groupType}{$models}\n";
      foreach ($group->limits as $limit) {
          echo "  {$limit->type}: {$limit->value}\n";
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  workspace_id = "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  rate_limits = client.beta.organization.workspaces.rate_limits.list(workspace_id)

  rate_limits.data.each do |group|
    models = group.models ? " (#{group.models.join(", ")})" : ""
    puts "#{group.group_type}#{models}"
    group.limits.each do |limit|
      puts "  #{limit.type}: #{limit.value}"
    end
  end
  ```
</CodeGroup>

```json
{
  "data": [
    {
      "type": "workspace_rate_limit",
      "group_type": "model_group",
      "models": ["claude-opus-5"],
      "limits": [
        { "type": "requests_per_minute", "value": 1000, "org_limit": 4000 },
        { "type": "input_tokens_per_minute", "value": 500000, "org_limit": 10000000 }
      ]
    },
    {
      "type": "workspace_rate_limit",
      "group_type": "model_group",
      "models": [
        "claude-opus-4-5",
        "claude-opus-4-5-20251101",
        "claude-opus-4-6",
        "claude-opus-4-7",
        "claude-opus-4-8"
      ],
      "limits": [
        { "type": "requests_per_minute", "value": 1000, "org_limit": 4000 },
        { "type": "input_tokens_per_minute", "value": 500000, "org_limit": 10000000 }
      ]
    }
  ],
  "next_page": null
}
```

## 按组类型筛选

两个端点都接受一个可选的 `group_type` 查询参数，用于将响应限制为单一类别：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/rate_limits?group_type=batch" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:rate-limits list --group-type batch
  ```

  ```python Python
  client = anthropic.Anthropic()

  rate_limits = client.beta.organization.rate_limits.list(group_type="batch")

  for group in rate_limits:
      models = f" ({', '.join(group.models)})" if group.models else ""
      print(f"{group.group_type}{models}")
      for limit in group.limits:
          print(f"  {limit.type}: {limit.value}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const rateLimits = await client.beta.organization.rateLimits.list({ group_type: "batch" });

  for await (const group of rateLimits) {
    const models = group.models ? ` (${group.models.join(", ")})` : "";
    console.log(`${group.group_type}${models}`);
    for (const limit of group.limits) {
      console.log(`  ${limit.type}: ${limit.value}`);
    }
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Organization.RateLimits;

  AnthropicClient client = new();

  var rateLimits = await client.Beta.Organization.RateLimits.List(new()
  {
      GroupType = GroupType.Batch
  });

  await foreach (var group in rateLimits.Paginate())
  {
      var models = group.Models is null ? "" : $" ({string.Join(", ", group.Models)})";
      Console.WriteLine($"{group.GroupType.Raw()}{models}");
      foreach (var limit in group.Limits)
      {
          Console.WriteLine($"  {limit.Type}: {limit.Value}");
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  rateLimits := client.Beta.Organization.RateLimits.ListAutoPaging(context.Background(), anthropic.BetaOrganizationRateLimitListParams{
  	GroupType: anthropic.BetaOrganizationRateLimitListParamsGroupTypeBatch,
  })

  for rateLimits.Next() {
  	group := rateLimits.Current()
  	models := ""
  	if len(group.Models) > 0 {
  		models = fmt.Sprintf(" (%s)", strings.Join(group.Models, ", "))
  	}
  	fmt.Printf("%s%s\n", group.GroupType, models)
  	for _, limit := range group.Limits {
  		fmt.Printf("  %s: %d\n", limit.Type, limit.Value)
  	}
  }
  if err := rateLimits.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.beta.organization.ratelimits.RateLimitListParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = RateLimitListParams.builder()
          .groupType(RateLimitListParams.GroupType.BATCH)
          .build();
      var rateLimits = client.beta().organization().rateLimits().list(params);

      for (var group : rateLimits.autoPager()) {
          var models = group.models()
              .map(modelIds -> " (" + String.join(", ", modelIds) + ")")
              .orElse("");
          IO.println(group.groupType().asString() + models);
          for (var limit : group.limits()) {
              IO.println("  " + limit.type() + ": " + limit.value());
          }
      }
  }
  ```

  ```php PHP
  use Anthropic\Beta\Organization\RateLimits\RateLimitListParams\GroupType;
  // ...

  $client = new Client();

  $rateLimits = $client->beta->organization->rateLimits->list(
      groupType: GroupType::BATCH,
  );

  foreach ($rateLimits->data as $group) {
      $models = $group->models ? ' (' . implode(', ', $group->models) . ')' : '';
      echo "{$group->groupType}{$models}\n";
      foreach ($group->limits as $limit) {
          echo "  {$limit->type}: {$limit->value}\n";
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  rate_limits = client.beta.organization.rate_limits.list(group_type: :batch)

  rate_limits.data.each do |group|
    models = group.models ? " (#{group.models.join(", ")})" : ""
    puts "#{group.group_type}#{models}"
    group.limits.each do |limit|
      puts "  #{limit.type}: #{limit.value}"
    end
  end
  ```
</CodeGroup>

有效值为 `model_group`、`batch`、`token_count`、`files`、`skills` 和 `web_search`。

## 分页

两个端点都接受 `page` 查询参数并返回 `next_page` 字段。目前响应始终为单页，因此 `next_page` 为 `null`。请基于 `next_page` 进行循环，这样当响应增长时，您的客户端无需更改即可正确分页。

## 常见问题

### 哪些模型字符串会出现在 `models` 列表中？

计入该组的每个模型 ID 和别名，包括带日期的 ID（例如 `claude-sonnet-4-5-20250929`）和不带日期的别名（例如 `claude-sonnet-4-5`）。查找您传递给 Messages API 的任意模型字符串，您都会在恰好一个 `model_group` 条目中找到它。

### 如果工作区响应中缺少某个组，这意味着什么？

该工作区对该组没有覆盖设置，并继承组织级限制。查询组织端点即可查看继承的值。

### 我可以使用此 API 更新速率限制吗？

不可以。要设置工作区速率限制，请在 [Claude Console](https://platform.claude.com/settings/workspaces) 中打开该工作区并使用**速率限制**选项卡。

## 另请参阅

* [速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)
* [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)
* [Admin API 参考](https://platform.claude.com/docs/zh-CN/api/admin)
* [工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces)
* [使用量和成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api)
