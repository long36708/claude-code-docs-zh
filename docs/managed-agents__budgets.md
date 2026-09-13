---
title: 会话预算
url: https://platform.claude.com/docs/zh-CN/managed-agents/budgets
description: 使用按公开标价费率强制执行的硬性美元预算来限制会话的支出。
---

会话预算（session budget）是您在[创建会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)时设置的可选硬性支出上限。平台会持续按公开标价费率对会话消耗的所有内容进行计价（即会话的**标价成本**，list cost），并在该成本达到预算后停止发出新的模型请求。越过上限时正在进行中的请求仍会完成，因此最终的标价成本可能会[略微超出预算](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#when-a-session-reaches-its-budget)。达到预算的会话会暂停并进入[空闲](https://platform.claude.com/docs/zh-CN/managed-agents/session-operations#session-statuses)状态，而不是终止；更改或移除预算会自动恢复其工作。部署（deployment）接受相同的预算，并将其应用于它们启动的每个会话；请参阅[部署上的预算](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#budgets-on-deployments)。

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

## 在创建会话时设置预算

创建会话时传入可选的 `budget` 字段：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  session=$(curl -sS --fail-with-body https://api.anthropic.com/v1/sessions \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d @- <<EOF
  {
    "agent": "$AGENT_ID",
    "environment_id": "$ENVIRONMENT_ID",
    "budget": {
      "type": "limit",
      "max_list_cost": {"amount": "125", "currency": "USD"}
    }
  }
  EOF
  )
  SESSION_ID=$(jq -r '.id' <<< "$session")
  ```

  ```bash CLI
  # 请保持金额带引号，以便作为字符串而非数字发送。
  SESSION_ID=$(ant beta:sessions create \
    --agent "$AGENT_ID" \
    --environment-id "$ENVIRONMENT_ID" \
    --budget '{type: limit, max_list_cost: {amount: "125", currency: USD}}' \
    --transform id --raw-output)
  ```

  ```python Python
  session = client.beta.sessions.create(
      agent=agent.id,
      environment_id=environment.id,
      budget={
          "type": "limit",
          "max_list_cost": {"amount": "125", "currency": "USD"},
      },
  )
  print(session.id, session.budget.max_list_cost.amount)  # sesn_01... 125
  ```

  ```typescript TypeScript
  const session = await client.beta.sessions.create({
    agent: agent.id,
    environment_id: environment.id,
    budget: {
      type: "limit",
      max_list_cost: { amount: "125", currency: "USD" }
    }
  });
  console.log(session.id, session.budget?.max_list_cost.amount); // sesn_01... 125
  ```

  ```csharp C#
  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = agent.ID,
      EnvironmentID = environment.ID,
      Budget = new()
      {
          Type = BetaManagedAgentsBudgetLimitType.Limit,
          MaxListCost = new() { Amount = "125", Currency = BetaCurrency.Usd },
      },
  });
  Console.WriteLine($"{session.ID} {session.Budget?.MaxListCost.Amount}");  // sesn_01... 125
  ```

  ```go Go
  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent: anthropic.BetaSessionNewParamsAgentUnion{
  		OfString: anthropic.String(agent.ID),
  	},
  	EnvironmentID: environment.ID,
  	Budget: anthropic.BetaManagedAgentsBudgetLimitParam{
  		Type: anthropic.BetaManagedAgentsBudgetLimitTypeLimit,
  		MaxListCost: anthropic.BetaMonetaryAmountParam{
  			Amount:   "125",
  			Currency: anthropic.BetaCurrencyUsd,
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(session.ID, session.Budget.MaxListCost.Amount) // sesn_01... 125
  ```

  ```java Java
  var session = client.beta().sessions().create(SessionCreateParams.builder()
      .agent(agent.id())
      .environmentId(environment.id())
      .budget(BetaManagedAgentsBudgetLimit.builder()
          .type(BetaManagedAgentsBudgetLimit.Type.LIMIT)
          .maxListCost(BetaMonetaryAmount.builder()
              .amount("125")
              .currency(BetaCurrency.USD)
              .build())
          .build())
      .build());
  IO.println(session.id() + " " + session.budget().orElseThrow().maxListCost().amount());  // sesn_01... 125
  ```

  ```php PHP
  $session = $client->beta->sessions->create(
      agent: $agent->id,
      environmentID: $environment->id,
      budget: [
          'type' => 'limit',
          'max_list_cost' => ['amount' => '125', 'currency' => 'USD'],
      ],
  );
  echo "{$session->id} {$session->budget->maxListCost->amount}\n"; // sesn_01... 125
  ```

  ```ruby Ruby
  session = client.beta.sessions.create(
    agent: agent.id,
    environment_id: environment.id,
    budget: {
      type: "limit",
      max_list_cost: {amount: "125", currency: "USD"}
    }
  )
  puts "#{session.id} #{session.budget.max_list_cost.amount}" # sesn_01... 125
  ```
</CodeGroup>

`budget` 对象有两个字段：

* `type` 始终为 `"limit"`。
* `max_list_cost` 是上限本身：`amount` 是以字符串形式书写、不带前导零的整数美分数（`"125"` 表示 $1.25，`"50"` 表示 50 美分），且必须大于零。诸如 `"25.00"` 之类的小数形式会被拒绝。该金额是字符串而非数字，因此永远不会对其应用浮点舍入。`currency` 是大写的 ISO-4217 货币代码；`USD` 是唯一支持的货币。

预算只能在创建会话时附加。向没有预算的现有会话添加预算会被拒绝并返回 400 错误。已设预算的会话的上限可以随时[更改](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#change-the-budget)或[移除](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#remove-the-budget)。

## 标价成本的计量方式

平台会持续按公开标价费率对会话消耗的内容进行计价：

* **模型令牌**，按每个所服务模型的标价计算
* **网页搜索**，每 1,000 次搜索 $10
* **会话运行时间**，每小时 $0.08

这一累计美元总额就是会话的**标价成本**，预算正是与它进行比较。标价成本并非您的合同价格：如果您的组织已协商折扣，会话会在标价总额达到上限时达到上限，而您的实际计费支出可能低于该上限。

强制执行使用精确的、未经舍入的标价成本。会话及其事件上报告的 `list_cost` 数值为整数美分，四舍五入到最接近的美分，因此报告的数值可能与强制执行所用的精确金额相差最多半美分（上下皆有可能）。

## 当会话达到其预算时

上限在模型请求之间强制执行，而不是在请求进行中。在每次模型请求之前，平台会检查会话已消耗的标价成本，一旦该总额达到上限，每个线程都会在其下一次请求之前暂停。使总额越过上限的那个请求是在会话仍低于上限时被接纳的，并会运行至完成，因此已暂停会话记录的 `list_cost` 会等于或略微超过 `max_list_cost`：上限为 `"50"`（50 美分）的会话可能以 `"53"` 的 `list_cost` 暂停。这是预期行为，而非计费错误，且超出量以每个线程一次模型请求为界。请将预算视为对新工作的约束，而非精确的停止点，并在设定上限时考虑这一次请求的余量。

达到预算的会话会进入空闲状态，其 `stop_reason` 为 `budget_reached`；它不会被终止，其历史记录和沙箱会像任何其他空闲会话一样被保留。在[事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)上，您将依次看到：

1. 每个线程暂停时，一个 `stop_reason` 为 `budget_reached` 的 `session.thread_status_idle` 事件。
2. 一个 [`session.usage`](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#monitor-spend) 事件，包含会话的累计用量和标价成本。
3. 一个 `stop_reason` 为 `budget_reached` 的 `session.status_idle` 事件。用量事件始终紧接在此空闲事件之前。

如果某个线程的最后一次请求既越过了上限又完成了其轮次，则该线程会在其自身的 `session.thread_status_idle` 事件上报告 `end_turn`，而会话仍报告 `budget_reached`；请将会话级别的 `stop_reason` 视为会话在其预算处暂停的信号。

### 达到上限时接受的事件

当会话达到或超过其预算时，它只接受用于结算已在进行中的工作的事件：

* `user.tool_confirmation`
* `user.tool_result`
* `user.custom_tool_result`
* `user.interrupt`

任何会启动新工作的事件（例如 `user.message`）都会被拒绝，并返回列出此列表的 400 错误。已结算的结果会被记录，而不会触发新的模型请求；会话保持在其预算处暂停。

在会话于其预算处暂停（所有线程均在上限处暂停）时发送的 `user.interrupt` 会被接受并忽略：它不会出现在事件列表中，也不会改变任何内容。请更改或移除预算以继续。

## 恢复达到预算的会话

通过会话更新来更改或移除预算。被接受的更新会自动恢复会话已暂停的工作；无需客户端进一步操作。

### 更改预算

使用新的 `max_list_cost` 更新会话。新值可以高于或低于当前上限，但必须严格大于会话已消耗的标价成本；否则更新会被拒绝并返回 400 错误：`budget.max_list_cost must be greater than the session's consumed list cost`。由于会话暂停时已消耗成本通常[略微超过旧上限](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#when-a-session-reaches-its-budget)，请以会话报告的 `usage.list_cost` 为基础设定新值，而不是以旧的 `max_list_cost` 为基础。请将其设置为比该数值高一美分或更多：报告的值经过舍入，可能略低于检查所用的精确已消耗成本。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -sS --fail-with-body "https://api.anthropic.com/v1/sessions/$SESSION_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "budget": {
        "type": "limit",
        "max_list_cost": {"amount": "500", "currency": "USD"}
      }
    }'
  ```

  ```bash CLI
  ant beta:sessions update \
    --session-id "$SESSION_ID" \
    --budget '{type: limit, max_list_cost: {amount: "500", currency: USD}}'
  ```

  ```python Python
  updated_session = client.beta.sessions.update(
      session.id,
      budget={
          "type": "limit",
          "max_list_cost": {"amount": "500", "currency": "USD"},
      },
  )
  print(updated_session.budget.max_list_cost.amount)  # 500
  ```

  ```typescript TypeScript
  const updatedSession = await client.beta.sessions.update(session.id, {
    budget: {
      type: "limit",
      max_list_cost: { amount: "500", currency: "USD" }
    }
  });
  console.log(updatedSession.budget?.max_list_cost.amount); // 500
  ```

  ```csharp C#
  var updatedSession = await client.Beta.Sessions.Update(session.ID, new()
  {
      Budget = new()
      {
          Type = BetaManagedAgentsBudgetLimitType.Limit,
          MaxListCost = new() { Amount = "500", Currency = BetaCurrency.Usd },
      },
  });
  Console.WriteLine(updatedSession.Budget?.MaxListCost.Amount);  // 500
  ```

  ```go Go
  updatedSession, err := client.Beta.Sessions.Update(ctx, session.ID, anthropic.BetaSessionUpdateParams{
  	Budget: anthropic.BetaManagedAgentsBudgetLimitParam{
  		Type: anthropic.BetaManagedAgentsBudgetLimitTypeLimit,
  		MaxListCost: anthropic.BetaMonetaryAmountParam{
  			Amount:   "500",
  			Currency: anthropic.BetaCurrencyUsd,
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(updatedSession.Budget.MaxListCost.Amount) // 500
  ```

  ```java Java
  var updatedSession = client.beta().sessions().update(session.id(), SessionUpdateParams.builder()
      .budget(BetaManagedAgentsBudgetLimit.builder()
          .type(BetaManagedAgentsBudgetLimit.Type.LIMIT)
          .maxListCost(BetaMonetaryAmount.builder()
              .amount("500")
              .currency(BetaCurrency.USD)
              .build())
          .build())
      .build());
  IO.println(updatedSession.budget().orElseThrow().maxListCost().amount());  // 500
  ```

  ```php PHP
  $updatedSession = $client->beta->sessions->update(
      $session->id,
      budget: [
          'type' => 'limit',
          'max_list_cost' => ['amount' => '500', 'currency' => 'USD'],
      ],
  );
  echo "{$updatedSession->budget->maxListCost->amount}\n"; // 500
  ```

  ```ruby Ruby
  updated_session = client.beta.sessions.update(
    session.id,
    budget: {
      type: "limit",
      max_list_cost: {amount: "500", currency: "USD"}
    }
  )
  puts updated_session.budget.max_list_cost.amount # 500
  ```
</CodeGroup>

### 移除预算

将 `budget` 设置为 `null` 以完全移除上限。会话已暂停的工作会恢复，且产生的 `session.updated` 事件携带设置为 `null` 的 `budget`。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -sS --fail-with-body "https://api.anthropic.com/v1/sessions/$SESSION_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{"budget": null}'
  ```

  ```bash CLI
  ant beta:sessions update --session-id "$SESSION_ID" --budget null
  ```

  ```python Python
  unbudgeted_session = client.beta.sessions.update(session.id, budget=None)
  print(unbudgeted_session.budget)  # None
  ```

  ```typescript TypeScript
  const unbudgetedSession = await client.beta.sessions.update(session.id, { budget: null });
  console.log(unbudgetedSession.budget); // null
  ```

  ```csharp C#
  // 赋值为 null 会发送显式的 null；不设置 Budget 则会省略该字段。
  var unbudgetedSession = await client.Beta.Sessions.Update(session.ID, new() { Budget = null });
  Console.WriteLine(unbudgetedSession.Budget is null);  // True: the session no longer has a budget
  ```

  ```go Go
  // 零值的 Budget 会从请求中省略；param.NullStruct（来自
  // github.com/anthropics/anthropic-sdk-go/packages/param）会发送显式的 null。
  unbudgetedSession, err := client.Beta.Sessions.Update(ctx, session.ID, anthropic.BetaSessionUpdateParams{
  	Budget: param.NullStruct[anthropic.BetaManagedAgentsBudgetLimitParam](),
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(unbudgetedSession.JSON.Budget.Valid()) // false: the session no longer has a budget
  ```

  ```java Java
  // 空的 Optional 会发送显式的 null；不设置 budget 则会省略该字段。
  var unbudgetedSession = client.beta().sessions().update(session.id(), SessionUpdateParams.builder()
      .budget(Optional.empty())
      .build());
  IO.println(unbudgetedSession.budget().isPresent());  // false: the session no longer has a budget
  ```

  ```php PHP
  // update(budget: null) 会省略该字段，因此通过原始客户端发送显式的 null。
  $unbudgetedSession = $client->beta->sessions->raw
      ->update($session->id, ['budget' => null])
      ->parse();
  echo json_encode($unbudgetedSession->budget), "\n"; // null
  ```

  ```ruby Ruby
  unbudgeted_session = client.beta.sessions.update(session.id, budget: nil)
  p unbudgeted_session.budget # nil
  ```
</CodeGroup>

<Warning>
  移除会话的预算是单向操作：预算已被移除的会话无法再被赋予新的预算。若要保留会话的上限，请改为更改预算。
</Warning>

## 监控支出

会话对象携带其 `budget` 以及一个包含所跟踪支出的 `usage` 对象：`usage.list_cost` 是会话已消耗的标价成本，`usage.active_seconds` 是其运行时成本计价所依据的运行时间。对于在 `budget_reached` 处暂停的会话，预计 `usage.list_cost` 会等于或略微超过 `max_list_cost`：[越过上限的请求](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#when-a-session-reaches-its-budget)在暂停之前已完成。会话级别的 `active_seconds` 对并发线程的重叠活动只计一次。线程检索响应在线程自身的 `usage` 上携带相同的两个字段，按线程计价。每线程数值独立舍入，且不包含会话的运行时间成本，因此它们的总和不会精确等于会话的 `list_cost`；预算强制执行所依据的是会话级数值。

`session.usage` 事件是会话累计用量和所跟踪标价成本的快照。它携带会话的令牌总数、`list_cost`、`active_seconds`、`server_tool_use` 请求计数（`web_search_requests`，按每次请求计入标价成本；以及 `web_fetch_requests`，其值为 `0`，因为网页抓取请求不按次收费且不计量），以及会话 `budget` 的回显，若会话没有预算则为 `null`。它出现在事件列表和会话流中。无论停止原因是什么，会话都会在进入空闲状态之前立即发出一个该事件，因此达到预算的会话总会在预算达到的空闲事件之前立即发出一个该事件。

要从流和会话对象中读取用量，请参阅[跟踪用量](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#tracking-usage)。

## 多智能体会话中的预算

[多智能体](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)会话拥有一个在其所有线程之间共享的单一预算；没有每线程上限。每个线程的消耗按其自身所服务的模型计价，且随着共享上限的达到，各线程独立暂停。[顾问](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#give-the-session-an-advisor)咨询计入同一预算，按顾问模型的费率计价。一个线程可能在 `budget_reached` 处暂停，而另一个线程则完成其进行中的请求。

待处理的询问优先于上限：若会话中一个线程在等待 `requires_action`，另一个线程在 `budget_reached` 处暂停，则会话级别报告 `requires_action`。待处理的请求仍需要回答，而回答它属于预算不会阻止的[结算事件](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#events-accepted-at-the-cap)。

## 部署上的预算

[部署](https://platform.claude.com/docs/zh-CN/managed-agents/scheduled-deployments)在您创建或更新它时接受相同的 `budget` 对象：

```json
{
  "budget": {
    "type": "limit",
    "max_list_cost": { "amount": "2000", "currency": "USD" }
  }
}
```

上限会被复制到部署启动的每个会话上，因此它分别约束每次运行，而不是部署的累计支出。更改部署的预算适用于部署此后启动的会话，而不适用于已在运行的会话。与会话不同，部署的预算可以用 `null` 清除，并在之后重新设置。请参阅[为每次运行设置预算](https://platform.claude.com/docs/zh-CN/managed-agents/scheduled-deployments#set-a-budget-on-each-run)。

## 没有标价的模型

预算只能跟踪平台能够计价的消耗。如果创建的已设预算会话的智能体，或其[多智能体名册](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)上的任何智能体或顾问，使用了没有公开标价的模型，则该创建会被拒绝并返回 400 错误，说明该模型没有可用的标价。

如果已设预算会话的用量中出现了没有标价的模型，预算将无法再计量会话的支出：会话可能以 `budget_reached` 的 `stop_reason` 暂停，且更改预算会被拒绝。请移除预算以恢复会话。

## 错误参考

与预算相关的请求在以下情况下会被拒绝：

| 条件                                                                                                                                                 | 状态  |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| 在会话达到或超过其预算时发送了启动工作的事件（例如 `user.message`）；错误会列出[接受的结算事件](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#events-accepted-at-the-cap) | 400 |
| 预算被设置为等于或低于会话已消耗标价成本的值                                                                                                                             | 400 |
| 向创建时没有预算的会话添加预算，或在移除后重新添加                                                                                                                          | 400 |
| `amount` 不是整数美分（例如 `"25.00"`）、为零或负数，或 `currency` 不是 `USD`                                                                                          | 400 |
| 已设预算的创建引用了[没有公开标价](https://platform.claude.com/docs/zh-CN/managed-agents/budgets#models-without-a-list-price)的模型                                   | 400 |

<Note>
  会话预算是针对单个会话的、以美元计（以美分书写）的硬性上限，由平台强制执行。它们不同于 Messages API 的[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)，后者是建议性的、以令牌计量的预算，供模型在单个智能体循环内进行自我调节。
</Note>
