---
title: 梦境
url: https://platform.claude.com/docs/zh-CN/managed-agents/dreams
description: 让 Claude 回顾过去的会话，以整理智能体的记忆并发掘新的洞见。
---

<Tip>
  "Dreaming"（梦境）是一项研究预览功能。[申请访问权限](https://claude.com/form/claude-managed-agents)即可试用。
</Tip>

智能体在工作时会写入其 [memory stores（记忆存储）](https://platform.claude.com/docs/zh-CN/managed-agents/memory)，但这些写入是局部且增量的：经过许多会话之后，记忆存储会积累重复项、相互矛盾的内容以及过时的条目。

**Dreams（梦境）** 让 Claude 来清理这些问题。一次梦境会读取现有的记忆存储以及过去的会话记录，然后生成一个新的、重新组织过的记忆存储：重复项被合并，过时或相互矛盾的条目被替换为最新值，并发掘出新的洞见。

输入存储永远不会被修改，因此您可以审查输出结果，如果不满意可以将其丢弃。

<Note>
  梦境端点受 `dreaming-2026-04-21` beta 请求头控制；单独使用 `managed-agents-2026-04-01` 请求头并不能获得梦境的访问权限。本页中的梦境端点示例会同时发送这两个请求头；会话和记忆存储调用只需要 `managed-agents-2026-04-01`。SDK 会自动设置这些请求头。
</Note>

## 工作原理

一个 **dream（梦境）** 是一个异步作业，它接收：

* 一个预先存在的 **memory store（记忆存储）：** Claude 对其进行验证、去重和重新组织的存储，以及
* 1 到 100 个 **sessions（会话）：** Claude 从中挖掘模式和洞见并融入输出的过去会话记录。

梦境会生成另一个 **output memory store（输出记忆存储）**，与输入分开。在梦境开始 `running` 后不久，一旦工作流克隆了输入存储，输出存储 ID 就会出现在梦境的 `outputs[]` 中；处于 `running` 状态的梦境可能会短暂地报告一个空的 `outputs[]`。

## 创建梦境

<CodeGroup>
  ```bash cURL
  dream=$(curl -s https://api.anthropic.com/v1/dreams \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01,dreaming-2026-04-21" \
    -H "content-type: application/json" \
    --data @- <<EOF
  {
    "inputs": [
      { "type": "memory_store", "memory_store_id": "$store_id" },
      { "type": "sessions", "session_ids": ["$session_a", "$session_b"] }
    ],
    "model": "claude-opus-4-8",
    "instructions": "Focus on coding-style preferences; ignore one-off debugging notes."
  }
  EOF
  )
  dream_id=$(jq -r '.id' <<< "$dream")
  echo "$dream_id"  # drm_01...
  ```

  ```bash CLI
  dream_id=$(ant beta:dreams create --transform id --raw-output <<YAML
  inputs:
    - type: memory_store
      memory_store_id: $store_id
    - type: sessions
      session_ids: [$session_a, $session_b]
  model: claude-opus-4-8
  instructions: Focus on coding-style preferences; ignore one-off debugging notes.
  YAML
  )
  ```

  ```python Python
  dream = client.beta.dreams.create(
      inputs=[
          {"type": "memory_store", "memory_store_id": store_id},
          {"type": "sessions", "session_ids": [session_a, session_b]},
      ],
      model="claude-opus-4-8",
      instructions="Focus on coding-style preferences; ignore one-off debugging notes.",
  )
  print(dream.id)  # drm_01...
  ```

  ```typescript TypeScript
  let dream = await client.beta.dreams.create({
    inputs: [
      { type: "memory_store", memory_store_id: storeId },
      { type: "sessions", session_ids: [sessionA, sessionB] },
    ],
    model: "claude-opus-4-8",
    instructions: "Focus on coding-style preferences; ignore one-off debugging notes.",
  });
  console.log(dream.id); // drm_01...
  ```

  ```csharp C#
  var dream = await client.Beta.Dreams.Create(new()
  {
      Inputs =
      [
          new BetaDreamMemoryStoreInput
          {
              Type = BetaDreamMemoryStoreInputType.MemoryStore,
              MemoryStoreID = storeID,
          },
          new BetaDreamSessionsInput
          {
              Type = BetaDreamSessionsInputType.Sessions,
              SessionIds = [sessionA, sessionB],
          },
      ],
      Model = "claude-opus-4-8",
      Instructions = "Focus on coding-style preferences; ignore one-off debugging notes.",
  });
  Console.WriteLine(dream.ID);  // drm_01...
  ```

  ```go Go
  dream, err := client.Beta.Dreams.New(ctx, anthropic.BetaDreamNewParams{
  	Inputs: []anthropic.BetaDreamInputUnionParam{
  		anthropic.BetaDreamInputParamOfMemoryStore(storeID),
  		anthropic.BetaDreamInputParamOfSessions([]string{sessionA, sessionB}),
  	},
  	Model: anthropic.BetaDreamModelParamsUnion{
  		OfString: anthropic.String("claude-opus-4-8"),
  	},
  	Instructions: anthropic.String("Focus on coding-style preferences; ignore one-off debugging notes."),
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(dream.ID) // drm_01...
  ```

  ```java Java
  var dream = client.beta().dreams().create(
      DreamCreateParams.builder()
          .addMemoryStoreInput(storeId)
          .addSessionsInput(List.of(sessionA, sessionB))
          .model("claude-opus-4-8")
          .instructions("Focus on coding-style preferences; ignore one-off debugging notes.")
          .build()
  );
  IO.println(dream.id());  // drm_01...
  ```

  ```php PHP
  $dream = $client->beta->dreams->create(
      inputs: [
          ['type' => 'memory_store', 'memory_store_id' => $storeId],
          ['type' => 'sessions', 'session_ids' => [$sessionA, $sessionB]],
      ],
      model: 'claude-opus-4-8',
      instructions: 'Focus on coding-style preferences; ignore one-off debugging notes.',
  );
  echo "{$dream->id}\n"; // drm_01...
  ```

  ```ruby Ruby
  dream = client.beta.dreams.create(
    inputs: [
      {type: "memory_store", memory_store_id: store_id},
      {type: "sessions", session_ids: [session_a, session_b]}
    ],
    model: "claude-opus-4-8",
    instructions: "Focus on coding-style preferences; ignore one-off debugging notes."
  )
  puts dream.id # drm_01...
  ```
</CodeGroup>

梦境的输入包括预先存在的记忆存储和一个会话数组。所选模型运行梦境流水线。在研究预览期间，支持 `claude-opus-5`、`claude-fable-5`、`claude-opus-4-8`、`claude-opus-4-7`、`claude-sonnet-5` 和 `claude-sonnet-4-6`。您可以选择性地传入 `instructions` 来引导梦境过程。请参阅[使用指令进行引导](https://platform.claude.com/docs/zh-CN/managed-agents/dreams#steer-with-instructions)。

响应是完整的 `dream` 资源，其中 `status: "pending"`：

```json
{
  "type": "dream",
  "id": "drm_01AbCDefGhIjKlMnOpQrStUv",
  "status": "pending",
  "inputs": [
    { "type": "memory_store", "memory_store_id": "memstore_01Hx..." },
    { "type": "sessions", "session_ids": ["sesn_01...", "sesn_02..."] }
  ],
  "outputs": [],
  "model": { "id": "claude-opus-4-8" },
  "instructions": "Focus on coding-style preferences; ignore one-off debugging notes.",
  "session_id": null,
  "created_at": "2026-04-29T17:04:10Z",
  "ended_at": null,
  "archived_at": null,
  "usage": {
    "input_tokens": 0,
    "output_tokens": 0,
    "cache_read_input_tokens": 0,
    "cache_creation_input_tokens": 0
  },
  "error": null
}
```

<Tip>
  如果您只有会话记录而没有现有存储，请先[创建一个空的记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory#create-a-memory-store)，并将其作为 `memory_store` 输入传入。
</Tip>

### 使用指令进行引导

可选的 `instructions` 字段用于引导梦境流水线合成的内容。它会在整个流水线中生效：哪些内容需要仔细阅读、哪些内容需要合并或丢弃，以及如何组织输出存储的结构。

请将 `instructions` 用于高层次的合成指导，例如关注领域（"关注编码风格偏好"）、需要原样保留的内容，或您希望在整个存储中应用的输出约定。该流水线是对输入的一次合成处理，而不是应用于存储文本的编辑器，因此针对特定行的命令式指令（"将句子 X 改为 Y"、"修正 Z 部分中的计数"）通常不会产生任何变化。若要对单条记忆进行有针对性的编辑，请直接在输出存储上使用 [Memory Stores API](https://platform.claude.com/docs/zh-CN/managed-agents/memory#view-and-edit-memories)。

## 跟踪进度

梦境以异步方式运行，通常需要几分钟到几小时，具体取决于输入会话记录的数量。通过 ID 轮询梦境以检查状态：

<CodeGroup>
  ```bash cURL
  while true; do
    dream=$(curl -s "https://api.anthropic.com/v1/dreams/$dream_id" \
      -H "x-api-key: $ANTHROPIC_API_KEY" \
      -H "anthropic-version: 2023-06-01" \
      -H "anthropic-beta: managed-agents-2026-04-01,dreaming-2026-04-21")
    status=$(jq -r '.status' <<< "$dream")
    echo "status=$status input_tokens=$(jq -r '.usage.input_tokens' <<< "$dream")"
    [[ "$status" == "pending" || "$status" == "running" ]] || break
    sleep 10
  done
  ```

  ```bash CLI
  ant beta:dreams retrieve --dream-id "$dream_id"
  ```

  ```python Python
  while dream.status in ("pending", "running"):
      time.sleep(10)
      dream = client.beta.dreams.retrieve(dream.id)
      print(f"status={dream.status} input_tokens={dream.usage.input_tokens}")
  ```

  ```typescript TypeScript
  while (dream.status === "pending" || dream.status === "running") {
    await sleep(10_000);
    dream = await client.beta.dreams.retrieve(dream.id);
    console.log(`status=${dream.status} input_tokens=${dream.usage.input_tokens}`);
  }
  ```

  ```csharp C#
  while (dream.Status.Value() is BetaDreamStatus.Pending or BetaDreamStatus.Running)
  {
      await Task.Delay(TimeSpan.FromSeconds(10));
      dream = await client.Beta.Dreams.Retrieve(dream.ID);
      Console.WriteLine($"status={dream.Status.Raw()} input_tokens={dream.Usage.InputTokens}");
  }
  ```

  ```go Go
  for dream.Status == anthropic.BetaDreamStatusPending || dream.Status == anthropic.BetaDreamStatusRunning {
  	time.Sleep(10 * time.Second)
  	dream, err = client.Beta.Dreams.Get(ctx, dream.ID, anthropic.BetaDreamGetParams{})
  	if err != nil {
  		panic(err)
  	}
  	fmt.Printf("status=%s input_tokens=%d\n", dream.Status, dream.Usage.InputTokens)
  }
  ```

  ```java Java
  while (dream.status().equals(BetaDreamStatus.PENDING)
          || dream.status().equals(BetaDreamStatus.RUNNING)) {
      Thread.sleep(10_000);
      dream = client.beta().dreams().retrieve(dream.id());
      IO.println("status=" + dream.status() + " input_tokens=" + dream.usage().inputTokens());
  }
  ```

  ```php PHP
  while (in_array($dream->status, [BetaDreamStatus::PENDING->value, BetaDreamStatus::RUNNING->value], true)) {
      sleep(10);
      $dream = $client->beta->dreams->retrieve($dream->id);
      echo "status={$dream->status} input_tokens={$dream->usage->inputTokens}\n";
  }
  ```

  ```ruby Ruby
  while %i[pending running].include?(dream.status)
    sleep 10
    dream = client.beta.dreams.retrieve(dream.id)
    puts "status=#{dream.status} input_tokens=#{dream.usage.input_tokens}"
  end
  ```
</CodeGroup>

### 生命周期

| `status`    | 含义                                |
| ----------- | --------------------------------- |
| `pending`   | 梦境已成功创建并排队。                       |
| `running`   | 流水线正在处理。`usage` 会随着工作进展而更新。       |
| `completed` | 已成功完成。`outputs[]` 的值即为新的记忆存储。     |
| `failed`    | 梦境运行以错误结束。输出记忆存储保持原样，保留失败前已写入的内容。 |
| `canceled`  | 梦境运行已取消。输出记忆存储保持原样。               |

### 观察流水线运行

一旦梦境处于 `running` 状态，其 `session_id` 字段就会指向运行该流水线的底层 [session（会话）](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)。您可以流式传输该会话的 [events（事件）](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)，以实时观察梦境正在读取和写入的内容。当梦境达到终止状态时，该会话会被归档（而非删除），因此会话记录在之后仍然可用。

## 使用输出

当 `status` 达到 `completed` 时，`outputs[]` 中的 `memory_store` 条目引用的是一个已完全填充的存储。它是您工作区中的一个普通记忆存储。您可以使用 [Memory Stores API](https://platform.claude.com/docs/zh-CN/managed-agents/memory#view-and-edit-memories) 或在 Console 中审查它，然后选择：

* **利用它：** 将其作为 `memory_store` 资源附加到未来的会话中，以替代（或配合）输入记忆存储，或者
* **丢弃它：** [删除该记忆存储](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/delete)或[归档该记忆存储](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/archive)。

<CodeGroup>
  ```bash cURL
  # dream 结束后，memory_store 输出保存重建后的存储
  output_store_id=$(jq -r 'first(.outputs[] | select(.type == "memory_store")).memory_store_id' <<< "$dream")

  curl -s https://api.anthropic.com/v1/sessions \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    --data @- <<EOF
  {
    "agent": "$agent_id",
    "environment_id": "$environment_id",
    "resources": [
      { "type": "memory_store", "memory_store_id": "$output_store_id" }
    ]
  }
  EOF
  ```

  ```bash CLI
  output_store_id=$(ant beta:dreams retrieve --dream-id "$dream_id" --format json |
    jq -r 'first(.outputs[] | select(.type == "memory_store")).memory_store_id')

  ant beta:sessions create <<YAML
  agent: $agent_id
  environment_id: $environment_id
  resources:
    - type: memory_store
      memory_store_id: $output_store_id
  YAML
  ```

  ```python Python
  # 梦境结束后，输出中保存着重建的记忆存储
  output_store_id = next(
      output.memory_store_id for output in dream.outputs if output.type == "memory_store"
  )

  session = client.beta.sessions.create(
      agent=agent_id,
      environment_id=environment_id,
      resources=[
          {"type": "memory_store", "memory_store_id": output_store_id},
      ],
  )
  ```

  ```typescript TypeScript
  // dream 结束后，输出中保存着重建的内存存储
  const output = dream.outputs.find((entry) => entry.type === "memory_store");
  const outputStoreId = output!.memory_store_id;

  await client.beta.sessions.create({
    agent: agentId,
    environment_id: environmentId,
    resources: [
      { type: "memory_store", memory_store_id: outputStoreId },
    ],
  });
  ```

  ```csharp C#
  var output = dream.Outputs.FirstOrDefault(entry => entry.Type == "memory_store");
  if (output is { MemoryStoreID: var outputStoreID })
  {
      await client.Beta.Sessions.Create(new()
      {
          Agent = agentID,
          EnvironmentID = environmentID,
          Resources =
          [
              new BetaManagedAgentsMemoryStoreResourceParam
              {
                  Type = BetaManagedAgentsMemoryStoreResourceParamType.MemoryStore,
                  MemoryStoreID = outputStoreID,
              },
          ],
      });
  }
  ```

  ```go Go
  for _, output := range dream.Outputs {
  	if output.Type != "memory_store" {
  		continue
  	}
  	outputStoreID := output.MemoryStoreID

  	session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  		Agent: anthropic.BetaSessionNewParamsAgentUnion{
  			OfString: anthropic.String(agentID),
  		},
  		EnvironmentID: environmentID,
  		Resources: []anthropic.BetaSessionNewParamsResourceUnion{{
  			OfMemoryStore: &anthropic.BetaManagedAgentsMemoryStoreResourceParam{
  				MemoryStoreID: outputStoreID,
  			},
  		}},
  	})
  	if err != nil {
  		panic(err)
  	}
  	fmt.Println(session.ID)
  	break
  }
  ```

  ```java Java
  var output = dream.outputs().stream()
      .filter(entry -> entry.type().equals(BetaDreamOutput.Type.MEMORY_STORE))
      .findFirst();
  if (output.isPresent()) {
      var outputStoreId = output.get().memoryStoreId();

      var session = client.beta().sessions().create(
          SessionCreateParams.builder()
              .agent(agentId)
              .environmentId(environmentId)
              .addMemoryStoreResource(outputStoreId)
              .build()
      );
  }
  ```

  ```php PHP
  $matches = array_filter($dream->outputs, fn($output) => $output->type === 'memory_store');
  $output = $matches ? reset($matches) : null;
  if ($output !== null) {
      $session = $client->beta->sessions->create(
          agent: $agentId,
          environmentID: $environmentId,
          resources: [
              ['type' => 'memory_store', 'memory_store_id' => $output->memoryStoreID],
          ],
      );
  }
  ```

  ```ruby Ruby
  output = dream.outputs.find { it.type == :memory_store }
  if output
    client.beta.sessions.create(
      agent: agent_id,
      environment_id: environment_id,
      resources: [
        {type: "memory_store", memory_store_id: output.memory_store_id}
      ]
    )
  end
  ```
</CodeGroup>

梦境本身永远不会删除或修改其输入。在 `failed` 或 `canceled` 状态下，输出存储会保留部分内容，以便您检查停止前生成的内容；如果不需要，请通过 Memory Stores API 将其清理。

<Warning>
  当梦境处于 `pending` 或 `running` 状态时，400 保护机制适用于归档梦境本身，而不适用于其存储。在运行过程中归档或删除*输入*记忆存储（或删除输入会话）将导致梦境失败，并返回 `input_memory_store_unavailable` 或 `input_session_unavailable`。
</Warning>

## 取消梦境

取消操作会立即将处于 `pending` 或 `running` 状态的梦境转为 `canceled`。取消一个已经处于 `canceled` 状态的梦境是幂等的空操作；取消处于 `completed` 或 `failed` 状态的梦境会返回 400。

<Note>
  取消之后，梦境的 `usage` 字段可能会在进行中的工作逐步结束期间继续更新几秒钟。如果您需要最终计数，请轮询梦境直到 `usage` 稳定下来。
</Note>

<CodeGroup>
  ```bash cURL
  curl -s -X POST "https://api.anthropic.com/v1/dreams/$dream_id/cancel" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01,dreaming-2026-04-21"
  ```

  ```bash CLI
  ant beta:dreams cancel --dream-id "$dream_id"
  ```

  ```python Python
  client.beta.dreams.cancel(dream.id)
  ```

  ```typescript TypeScript
  await client.beta.dreams.cancel(dream.id);
  ```

  ```csharp C#
  await client.Beta.Dreams.Cancel(dream.ID);
  ```

  ```go Go
  dream, err = client.Beta.Dreams.Cancel(ctx, dream.ID, anthropic.BetaDreamCancelParams{})
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().dreams().cancel(dream.id());
  ```

  ```php PHP
  $client->beta->dreams->cancel($dream->id);
  ```

  ```ruby Ruby
  client.beta.dreams.cancel(dream.id)
  ```
</CodeGroup>

## 归档梦境

归档操作会在已达到终止状态（`completed`、`failed` 或 `canceled`）的梦境上设置 `archived_at`；`status` 保持不变。已归档的梦境会从默认的列表响应中排除，但仍可通过 ID 读取。归档一个已经归档的梦境是幂等的空操作。归档处于 `pending` 或 `running` 状态的梦境会返回 400；请先取消它。不支持取消归档。

<CodeGroup>
  ```bash cURL
  curl -s -X POST "https://api.anthropic.com/v1/dreams/$dream_id/archive" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01,dreaming-2026-04-21"
  ```

  ```bash CLI
  ant beta:dreams archive --dream-id "$dream_id"
  ```

  ```python Python
  client.beta.dreams.archive(dream.id)
  ```

  ```typescript TypeScript
  await client.beta.dreams.archive(dream.id);
  ```

  ```csharp C#
  await client.Beta.Dreams.Archive(dream.ID);
  ```

  ```go Go
  dream, err = client.Beta.Dreams.Archive(ctx, dream.ID, anthropic.BetaDreamArchiveParams{})
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().dreams().archive(dream.id());
  ```

  ```php PHP
  $client->beta->dreams->archive($dream->id);
  ```

  ```ruby Ruby
  client.beta.dreams.archive(dream.id)
  ```
</CodeGroup>

归档梦境不会影响其输出记忆存储；请通过 [Memory Stores API](https://platform.claude.com/docs/zh-CN/managed-agents/memory#view-and-edit-memories) 单独管理它。

## 列出梦境

返回工作区中所有未归档的梦境，按最新优先排序。使用 `limit`（默认 20，最大 100）和 `page` 游标进行分页。传入 `include_archived=true` 以包含已归档的梦境。

<CodeGroup>
  ```bash cURL
  curl -s "https://api.anthropic.com/v1/dreams?limit=20" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01,dreaming-2026-04-21"
  ```

  ```bash CLI
  ant beta:dreams list --limit 20
  ```

  ```python Python
  for listed_dream in client.beta.dreams.list(limit=20):
      print(listed_dream.id, listed_dream.status)
  ```

  ```typescript TypeScript
  for await (const listedDream of client.beta.dreams.list({ limit: 20 })) {
    console.log(listedDream.id, listedDream.status);
  }
  ```

  ```csharp C#
  var page = await client.Beta.Dreams.List(new() { Limit = 20 });
  await foreach (var listed in page.Paginate())
  {
      Console.WriteLine($"{listed.ID} {listed.Status.Raw()}");
  }
  ```

  ```go Go
  dreams := client.Beta.Dreams.ListAutoPaging(ctx, anthropic.BetaDreamListParams{
  	Limit: anthropic.Int(20),
  })
  for dreams.Next() {
  	listed := dreams.Current()
  	fmt.Println(listed.ID, listed.Status)
  }
  if err := dreams.Err(); err != nil {
  	panic(err)
  }
  ```

  ```java Java
  for (var listedDream : client.beta().dreams().list(
      DreamListParams.builder().limit(20).build()
  ).autoPager()) {
      IO.println(listedDream.id() + " " + listedDream.status());
  }
  ```

  ```php PHP
  foreach ($client->beta->dreams->list(limit: 20)->pagingEachItem() as $dream) {
      echo "{$dream->id} {$dream->status}\n";
  }
  ```

  ```ruby Ruby
  client.beta.dreams.list(limit: 20).auto_paging_each do
    puts "#{it.id} #{it.status}"
  end
  ```
</CodeGroup>

## 错误

以下是可能出现的梦境错误的非完整列表。

| `error.type`                      | 出现时机                       |
| --------------------------------- | -------------------------- |
| `timeout`                         | 流水线超出了其运行时间预算。             |
| `internal_error`                  | 未分类的流水线故障。                 |
| `memory_store_org_limit_exceeded` | 在流水线配置工作存储时，您的组织达到了记忆存储上限。 |
| `input_memory_store_too_large`    | 输入记忆存储超出了流水线的大小限制。         |
| `input_memory_store_unavailable`  | 输入记忆存储在梦境创建后被归档或删除。        |
| `input_session_unavailable`       | 某个输入会话在梦境创建后被删除。           |

## 计费

梦境按您所选模型的标准 API 令牌费率计费；资源上的 `usage` 会报告精确的总量。成本大致随输入会话的数量和长度线性增长。请先从一小批会话开始，待您对整理质量满意后再扩大规模。

## 限制

| 限制                | 值                                                                                                          |
| ----------------- | ---------------------------------------------------------------------------------------------------------- |
| 每个梦境的会话数          | 100                                                                                                        |
| `instructions` 长度 | 4,096 个字符                                                                                                  |
| 支持的模型             | `claude-opus-5`、`claude-fable-5`、`claude-opus-4-8`、`claude-opus-4-7`、`claude-sonnet-5`、`claude-sonnet-4-6` |

在此功能处于研究预览阶段期间，梦境创建适用默认速率限制。如果您需要更高的限制，请[联系支持团队](https://support.claude.com)。
