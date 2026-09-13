---
title: 定时部署
url: https://platform.claude.com/docs/zh-CN/managed-agents/scheduled-deployments
description: 使用 Claude API 创建和管理部署：按周期性 cron 计划运行智能体并查看其运行历史。
---

**定时部署**（scheduled deployment）允许[智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)自主启动[会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)，从而按可预测的节奏完成任务。您可以使用 Deployments API（Claude API 的一部分）来创建和管理部署。

有关发布背景以及各团队按计划运行的任务示例，请参阅博客文章 [Claude Managed Agents 中的定时部署和保管库](https://claude.com/blog/whats-new-in-claude-managed-agents)。

<Note>
  所有 Managed Agents API 请求都需要 `managed-agents-2026-04-01` beta 请求头。SDK 会自动设置该 beta 请求头。
</Note>

## 创建定时部署

创建部署时，除了 `schedule` 之外，您还需要传入执行所需的[会话配置](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)。

* 部署需要[智能体配置](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)和[环境配置](https://platform.claude.com/docs/zh-CN/managed-agents/environments)，并可选择性地接受[文件](https://platform.claude.com/docs/zh-CN/managed-agents/files)、[GitHub](https://platform.claude.com/docs/zh-CN/managed-agents/github)、[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory)和[保管库](https://platform.claude.com/docs/zh-CN/managed-agents/vaults)。以[自托管环境](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)为目标的部署可以附加记忆存储；`file` 和 `github_repository` 资源则需要云环境。Claude Console 的部署表单目前不为自托管环境提供记忆存储选项；请改为通过 API 或 SDK 附加它们。
* 部署还需要至少一个初始事件（`user.message` 或 `user.define_outcome`），用于启动每个会话的工作。
* 在 `schedule` 中，您需要定义一个 cron `expression` 和一个 `timezone`。支持的最大粒度为分钟级。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  DEPLOYMENT_ID=$(
    curl --fail-with-body -sS "https://api.anthropic.com/v1/deployments?beta=true" \
      -H "x-api-key: $ANTHROPIC_API_KEY" \
      -H "anthropic-version: 2023-06-01" \
      -H "anthropic-beta: managed-agents-2026-04-01" \
      -H "content-type: application/json" \
      -d @- <<EOF | jq -er '.id'
  {
    "name": "Weekly compliance scan",
    "agent": "$AGENT_ID",
    "environment_id": "$ENVIRONMENT_ID",
    "initial_events": [
      {"type": "user.message", "content": [{"type": "text", "text": "Run the weekly compliance scan."}]}
    ],
    "schedule": {
      "type": "cron",
      "expression": "0 20 * * 5",
      "timezone": "America/New_York"
    }
  }
  EOF
  )
  ```

  ```bash CLI
  DEPLOYMENT_ID=$(ant beta:deployments create <<YAML | jq -er '.id'
  name: Weekly compliance scan
  agent: $AGENT_ID
  environment_id: $ENVIRONMENT_ID
  initial_events:
    - type: user.message
      content:
        - type: text
          text: Run the weekly compliance scan.
  schedule:
    type: cron
    expression: "0 20 * * 5"
    timezone: America/New_York
  YAML
  )
  ```

  ```python Python
  deployment = client.beta.deployments.create(
      name="Weekly compliance scan",
      agent=agent.id,
      environment_id=environment.id,
      initial_events=[
          {
              "type": "user.message",
              "content": [{"type": "text", "text": "Run the weekly compliance scan."}],
          },
      ],
      schedule={
          "type": "cron",
          "expression": "0 20 * * 5",
          "timezone": "America/New_York",
      },
  )
  ```

  ```typescript TypeScript
  const deployment = await client.beta.deployments.create({
    name: "Weekly compliance scan",
    agent: agent.id,
    environment_id: environment.id,
    initial_events: [
      {
        type: "user.message",
        content: [{ type: "text", text: "Run the weekly compliance scan." }],
      },
    ],
    schedule: {
      type: "cron",
      expression: "0 20 * * 5",
      timezone: "America/New_York",
    },
  });
  ```

  ```csharp C#
  var deployment = await client.Beta.Deployments.Create(new()
  {
      Name = "Weekly compliance scan",
      Agent = agent.ID,
      EnvironmentID = environment.ID,
      InitialEvents =
      [
          new BetaManagedAgentsUserMessageEventParams
          {
              Type = BetaManagedAgentsUserMessageEventParamsType.UserMessage,
              Content =
              [
                  new BetaManagedAgentsTextBlock
                  {
                      Type = BetaManagedAgentsTextBlockType.Text,
                      Text = "Run the weekly compliance scan.",
                  },
              ],
          },
      ],
      Schedule = new BetaManagedAgentsScheduleParams
      {
          Type = BetaManagedAgentsScheduleParamsType.Cron,
          Expression = "0 20 * * 5",
          Timezone = "America/New_York",
      },
  });
  ```

  ```go Go
  deployment, err := client.Beta.Deployments.New(ctx, anthropic.BetaDeploymentNewParams{
  	Name:          "Weekly compliance scan",
  	Agent:         anthropic.BetaDeploymentNewParamsAgentUnion{OfString: anthropic.String(agent.ID)},
  	EnvironmentID: environment.ID,
  	InitialEvents: []anthropic.BetaManagedAgentsDeploymentInitialEventParamsUnion{{
  		OfUserMessage: &anthropic.BetaManagedAgentsUserMessageEventParams{
  			Type: anthropic.BetaManagedAgentsUserMessageEventParamsTypeUserMessage,
  			Content: []anthropic.BetaManagedAgentsUserMessageEventParamsContentUnion{{
  				OfText: &anthropic.BetaManagedAgentsTextBlockParam{
  					Type: anthropic.BetaManagedAgentsTextBlockTypeText,
  					Text: "Run the weekly compliance scan.",
  				},
  			}},
  		},
  	}},
  	Schedule: anthropic.BetaManagedAgentsScheduleParams{
  		Type:       anthropic.BetaManagedAgentsScheduleParamsTypeCron,
  		Expression: "0 20 * * 5",
  		Timezone:   "America/New_York",
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var deployment = client.beta().deployments().create(
      DeploymentCreateParams.builder()
          .name("Weekly compliance scan")
          .agent(agent.id())
          .environmentId(environment.id())
          .addInitialEvent(
              BetaManagedAgentsUserMessageEventParams.builder()
                  .type(BetaManagedAgentsUserMessageEventParams.Type.USER_MESSAGE)
                  .addTextContent("Run the weekly compliance scan.")
                  .build()
          )
          .schedule(
              BetaManagedAgentsScheduleParams.builder()
                  .type(BetaManagedAgentsScheduleParams.Type.CRON)
                  .expression("0 20 * * 5")
                  .timezone("America/New_York")
                  .build()
          )
          .build()
  );
  ```

  ```php PHP
  $deployment = $client->beta->deployments->create(
      name: 'Weekly compliance scan',
      agent: $agent->id,
      environmentID: $environment->id,
      initialEvents: [
          [
              'type' => 'user.message',
              'content' => [['type' => 'text', 'text' => 'Run the weekly compliance scan.']],
          ],
      ],
      schedule: [
          'type' => 'cron',
          'expression' => '0 20 * * 5',
          'timezone' => 'America/New_York',
      ],
  );
  ```

  ```ruby Ruby
  deployment = client.beta.deployments.create(
    name: "Weekly compliance scan",
    agent: agent.id,
    environment_id: environment.id,
    initial_events: [
      {
        type: "user.message",
        content: [{type: "text", text: "Run the weekly compliance scan."}]
      }
    ],
    schedule: {
      type: "cron",
      expression: "0 20 * * 5",
      timezone: "America/New_York"
    }
  )
  ```
</CodeGroup>

响应中包含一个部署对象，其 `schedule.upcoming_runs_at` 字段已填充接下来的触发时间，以便您确认计划设置正确。

```json
{
  "id": "depl_01xyz",
  "status": "active",
  "paused_reason": null,
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York",
    "last_run_at": null,
    "upcoming_runs_at": [
      "2026-05-09T00:00:00Z",
      "2026-05-16T00:00:00Z",
      "2026-05-23T00:00:00Z"
    ]
  }
}
```

即将运行的时间戳反映的是所配置的精确计划。但是，为了分散负载，实际执行会应用抖动（jitter），幅度最多为两次运行间隔的 15%，最小为 5 秒，最大为 9 分钟。

每个组织最多支持 **1,000 个定时部署**。如需更多，请联系 Anthropic 支持团队。

有关完整参数和响应模式，请参阅[创建部署参考文档](https://platform.claude.com/docs/zh-CN/api/beta/deployments/create)。

### Cron 和时区语义

* **表达式：** 标准 POSIX cron（`minute hour day-of-month month day-of-week`）。您可以在 [Claude Console](https://platform.claude.com/workspaces/default/deployments) 中生成并验证这些 cron 表达式。
* **时区：** IANA 时区标识符（例如 `"America/Los_Angeles"`）。
* **夏令时（DST）：** Cron 计划使用字面挂钟时间匹配，因此 `America/New_York` 时区中的 `"0 20 * * *"` 会在当地时间晚上 8:00 触发，无论当前生效的是 EST 还是 EDT。

<Note>
  在夏令时开始（时钟拨快）当天不存在的挂钟时间（例如凌晨 2 点）不会被触发。在夏令时结束（时钟拨慢）当天出现两次的挂钟时间会触发两次。如果无法接受遗漏或重复执行，请将计划安排在当地时间凌晨 1–3 点窗口之外，或使用 UTC。
</Note>

### 为每次运行设置预算

在创建或更新部署时传入可选的 `budget` 对象。它的结构与[会话预算](https://platform.claude.com/docs/zh-CN/managed-agents/budgets)相同。部署会将该上限复制到它启动的每个会话上，因此预算是分别约束每次运行，而不是作为跨运行的累计上限：上限为 `"2000"` 的部署在每次运行中最多可花费约 20 美元。

由部署启动的会话与任何其他设有预算的会话行为完全相同：当其自身的标价成本[达到上限](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#when-a-session-reaches-its-budget)时，会话会以 `budget_reached` 暂停。更改部署的预算仅适用于之后启动的运行；已在运行的会话会保留其启动时的上限，您可以[通过会话本身进行更改](https://platform.claude.com/docs/zh-CN/managed-agents/session-operations#updating-the-session-budget)。与会话预算不同，部署的预算可以通过 `"budget": null` 移除，并在之后重新设置。

以下示例为现有部署设置预算：

```bash cURL
curl --fail-with-body -sS "https://api.anthropic.com/v1/deployments/$DEPLOYMENT_ID?beta=true" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: managed-agents-2026-04-01" \
  -H "content-type: application/json" \
  -d @- <<'EOF'
{
  "budget": {
    "type": "limit",
    "max_list_cost": {"amount": "2000", "currency": "USD"}
  }
}
EOF
```

## 部署运行

部署可能因多种原因而触发失败：例如，`environment` 资源已被归档，或会话创建受到速率限制。每次执行部署的尝试都会生成一条**部署运行**（deployment run）记录，使您能够独立于会话生命周期来跟踪成功与失败。

成功的部署会生成活动会话，成功的部署运行包含关联的 `session_id`。要跟踪会话的生命周期，请通过[事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)或 [webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks) 跟踪会话事件。部署生命周期的变更以及每次定时运行的结果也会作为 webhook 事件发送，列于[支持的事件类型](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks#supported-event-types)的"部署事件"和"部署运行事件"选项卡中。

按如下方式列出某个部署的所有部署运行：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl --fail-with-body -sS "https://api.anthropic.com/v1/deployment_runs?beta=true&deployment_id=$DEPLOYMENT_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"
  ```

  ```bash CLI
  ant beta:deployment-runs list --deployment-id "$DEPLOYMENT_ID"
  ```

  ```python Python
  for run in client.beta.deployment_runs.list(
      deployment_id=deployment.id,
  ):
      print(run.created_at, run.session_id or run.error.type)
  ```

  ```typescript TypeScript
  for await (const run of client.beta.deploymentRuns.list({
    deployment_id: deployment.id,
  })) {
    console.log(run.created_at, run.session_id ?? run.error?.type);
  }
  ```

  ```csharp C#
  var runs = await client.Beta.DeploymentRuns.List(
      new() { DeploymentID = deployment.ID }
  );
  await foreach (var run in runs.Paginate())
  {
      // Error 联合类型直接暴露 .Message；在添加通用 .Type 访问器之前，
      // 判别器需从 .Json 中读取。
      var outcome = run.SessionID ?? run.Error!.Json.GetProperty("type").GetString();
      Console.WriteLine($"{run.CreatedAt} {outcome}");
  }
  ```

  ```go Go
  runs := client.Beta.DeploymentRuns.ListAutoPaging(ctx, anthropic.BetaDeploymentRunListParams{
  	DeploymentID: anthropic.String(deployment.ID),
  })
  for runs.Next() {
  	run := runs.Current()
  	if run.SessionID != "" {
  		fmt.Println(run.CreatedAt.Format(time.RFC3339), run.SessionID)
  	} else {
  		fmt.Println(run.CreatedAt.Format(time.RFC3339), run.Error.Type)
  	}
  }
  if err := runs.Err(); err != nil {
  	panic(err)
  }
  ```

  ```java Java
  for (var run : client.beta().deploymentRuns().list(
          DeploymentRunListParams.builder()
              .deploymentId(deployment.id())
              .build()).autoPager()) {
      // Error 联合类型尚未暴露通用的 .type()/.message()
      // 访问器；.toString() 包含两者。
      IO.println(run.createdAt() + " "
          + run.sessionId().orElseGet(() -> run.error().orElseThrow().toString()));
  }
  ```

  ```php PHP
  foreach ($client->beta->deploymentRuns->list(
      deploymentID: $deployment->id,
  )->pagingEachItem() as $run) {
      $outcome = $run->sessionID ?? $run->error->type;
      echo "{$run->createdAt->format(DATE_ATOM)} {$outcome}\n";
  }
  ```

  ```ruby Ruby
  client.beta.deployment_runs.list(
    deployment_id: deployment.id
  ).auto_paging_each do
    puts "#{it.created_at} #{it.session_id || it.error.type}"
  end
  ```
</CodeGroup>

您还可以筛选出带有错误的部署运行：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl --fail-with-body -sS "https://api.anthropic.com/v1/deployment_runs?beta=true&deployment_id=$DEPLOYMENT_ID&has_error=true" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"
  ```

  ```bash CLI
  ant beta:deployment-runs list --deployment-id "$DEPLOYMENT_ID" --has-error
  ```

  ```python Python
  for run in client.beta.deployment_runs.list(
      deployment_id=deployment.id,
      has_error=True,
  ):
      print(run.created_at, run.error.type, run.error.message)
  ```

  ```typescript TypeScript
  for await (const run of client.beta.deploymentRuns.list({
    deployment_id: deployment.id,
    has_error: true,
  })) {
    console.log(run.created_at, run.error?.type, run.error?.message);
  }
  ```

  ```csharp C#
  var failedRuns = await client.Beta.DeploymentRuns.List(
      new() { DeploymentID = deployment.ID, HasError = true }
  );
  await foreach (var failedRun in failedRuns.Paginate())
  {
      var error = failedRun.Error!;
      var errorType = error.Json.GetProperty("type").GetString();
      Console.WriteLine($"{failedRun.CreatedAt} {errorType} {error.Message}");
  }
  ```

  ```go Go
  failedRuns := client.Beta.DeploymentRuns.ListAutoPaging(ctx, anthropic.BetaDeploymentRunListParams{
  	DeploymentID: anthropic.String(deployment.ID),
  	HasError:     anthropic.Bool(true),
  })
  for failedRuns.Next() {
  	failedRun := failedRuns.Current()
  	fmt.Println(failedRun.CreatedAt.Format(time.RFC3339), failedRun.Error.Type, failedRun.Error.Message)
  }
  if err := failedRuns.Err(); err != nil {
  	panic(err)
  }
  ```

  ```java Java
  for (var run : client.beta().deploymentRuns().list(
          DeploymentRunListParams.builder()
              .deploymentId(deployment.id())
              .hasError(true)
              .build()).autoPager()) {
      IO.println(run.createdAt() + " " + run.error().orElseThrow());
  }
  ```

  ```php PHP
  foreach ($client->beta->deploymentRuns->list(
      deploymentID: $deployment->id,
      hasError: true,
  )->pagingEachItem() as $run) {
      echo "{$run->createdAt->format(DATE_ATOM)} {$run->error->type} {$run->error->message}\n";
  }
  ```

  ```ruby Ruby
  client.beta.deployment_runs.list(
    deployment_id: deployment.id,
    has_error: true
  ).auto_paging_each do
    puts "#{it.created_at} #{it.error.type} #{it.error.message}"
  end
  ```
</CodeGroup>

失败的运行包含一个 `error`，其 `type` 描述了会话创建被拒绝的原因（例如 `environment_archived_error`、`agent_archived_error` 或 `session_rate_limited_error`）。有关所有筛选参数和响应模式，请参阅[列出部署运行参考文档](https://platform.claude.com/docs/zh-CN/api/beta/deployment_runs/list)。

```json
{
  "type": "deployment_run",
  "id": "drun_01abc124",
  "deployment_id": "depl_01xyz",
  "trigger_context": { "type": "schedule", "scheduled_at": "2026-05-09T00:00:00Z" },
  "session_id": null,
  "error": {
    "type": "environment_archived_error",
    "message": "environment `env_01abc` is archived"
  },
  "agent": { "type": "agent", "id": "agent_01ghi789", "version": 3 },
  "created_at": "2026-05-09T00:00:01Z"
}
```

要按 ID 检索单个运行，请调用 [`GET /v1/deployment_runs/{deployment_run_id}`](https://platform.claude.com/docs/zh-CN/api/beta/deployment_runs/retrieve)。[`deployment_run` webhook 事件](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks#supported-event-types)在其 `data.id` 中携带运行 ID。

## 管理部署生命周期

每次生命周期变更都会发出一个 [webhook 事件](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks#supported-event-types)，因此您无需轮询即可对部署的暂停、取消暂停或归档做出响应；请参阅"部署事件"选项卡。

**暂停**（Pause）会从此刻起抑制定时触发；先前部署运行中正在运行的会话会继续执行。暂停期间仍允许通过 `run` 端点进行手动运行。暂停会将 `paused_reason` 设置为 `{"type": "manual"}`；取消暂停会将其清除。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl --fail-with-body -sS -X POST "https://api.anthropic.com/v1/deployments/$DEPLOYMENT_ID/pause?beta=true" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"
  ```

  ```bash CLI
  ant beta:deployments pause --deployment-id "$DEPLOYMENT_ID"
  ```

  ```python Python
  client.beta.deployments.pause(deployment.id)
  ```

  ```typescript TypeScript
  await client.beta.deployments.pause(deployment.id);
  ```

  ```csharp C#
  await client.Beta.Deployments.Pause(deployment.ID);
  ```

  ```go Go
  if _, err := client.Beta.Deployments.Pause(ctx, deployment.ID, anthropic.BetaDeploymentPauseParams{}); err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().deployments().pause(deployment.id());
  ```

  ```php PHP
  $client->beta->deployments->pause($deployment->id);
  ```

  ```ruby Ruby
  client.beta.deployments.pause(deployment.id)
  ```
</CodeGroup>

**取消暂停**（Unpause）会从下一个计划时间点恢复计划。错过的触发不会被补执行。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl --fail-with-body -sS -X POST "https://api.anthropic.com/v1/deployments/$DEPLOYMENT_ID/unpause?beta=true" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"
  ```

  ```bash CLI
  ant beta:deployments unpause --deployment-id "$DEPLOYMENT_ID"
  ```

  ```python Python
  client.beta.deployments.unpause(deployment.id)
  ```

  ```typescript TypeScript
  await client.beta.deployments.unpause(deployment.id);
  ```

  ```csharp C#
  await client.Beta.Deployments.Unpause(deployment.ID);
  ```

  ```go Go
  if _, err := client.Beta.Deployments.Unpause(ctx, deployment.ID, anthropic.BetaDeploymentUnpauseParams{}); err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().deployments().unpause(deployment.id());
  ```

  ```php PHP
  $client->beta->deployments->unpause($deployment->id);
  ```

  ```ruby Ruby
  client.beta.deployments.unpause(deployment.id)
  ```
</CodeGroup>

**归档**（Archive）与**暂停**不同，它是终结性的：计划会终止，且部署无法再被修改。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl --fail-with-body -sS -X POST "https://api.anthropic.com/v1/deployments/$DEPLOYMENT_ID/archive?beta=true" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"
  ```

  ```bash CLI
  ant beta:deployments archive --deployment-id "$DEPLOYMENT_ID"
  ```

  ```python Python
  client.beta.deployments.archive(deployment.id)
  ```

  ```typescript TypeScript
  await client.beta.deployments.archive(deployment.id);
  ```

  ```csharp C#
  await client.Beta.Deployments.Archive(deployment.ID);
  ```

  ```go Go
  if _, err := client.Beta.Deployments.Archive(ctx, deployment.ID, anthropic.BetaDeploymentArchiveParams{}); err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().deployments().archive(deployment.id());
  ```

  ```php PHP
  $client->beta->deployments->archive($deployment->id);
  ```

  ```ruby Ruby
  client.beta.deployments.archive(deployment.id)
  ```
</CodeGroup>

### 失败行为

会话创建的速率限制响应会立即记录为一次 `session_rate_limited_error` 运行，不会重试；计划会在下一个计划时间点再次尝试。会话内部底层 API 调用的速率限制由会话自身处理。

如果部署的智能体已被归档，则该部署会在同一操作中自动归档。如果智能体已被删除，下一次定时触发会检测到智能体缺失并自动归档该部署。在这两种情况下都不会记录部署运行。如果智能体引用的子智能体已被归档，下一次触发会记录一次失败的运行，其 `error.type: "agent_archived_error"`，并且部署会自动暂停，以便您更新智能体后恢复。其他不可恢复的会话创建错误（例如环境或保管库已归档）的行为方式相同：触发会记录一次失败的运行，并且部署会自动暂停。部署的 `paused_reason.error.type` 与失败运行的 `error.type` 一致。

## 触发手动运行

要在计划之外运行部署，请调用 [`run` 端点](https://platform.claude.com/docs/zh-CN/api/beta/deployments/run)。这会立即创建一个会话，并写入一条 `trigger_context.type: "manual"` 的部署运行记录。这使您可以在正式启用计划之前测试部署。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl --fail-with-body -sS -X POST "https://api.anthropic.com/v1/deployments/$DEPLOYMENT_ID/run?beta=true" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"
  ```

  ```bash CLI
  ant beta:deployments run --deployment-id "$DEPLOYMENT_ID"
  ```

  ```python Python
  run = client.beta.deployments.run(deployment.id)
  ```

  ```typescript TypeScript
  const run = await client.beta.deployments.run(deployment.id);
  ```

  ```csharp C#
  var manualRun = await client.Beta.Deployments.Run(deployment.ID);
  ```

  ```go Go
  manualRun, err := client.Beta.Deployments.Run(ctx, deployment.ID, anthropic.BetaDeploymentRunParams{})
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var run = client.beta().deployments().run(deployment.id());
  ```

  ```php PHP
  $run = $client->beta->deployments->run($deployment->id);
  ```

  ```ruby Ruby
  run = client.beta.deployments.run(deployment.id)
  ```
</CodeGroup>
