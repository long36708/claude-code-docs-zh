---
title: 定义您的智能体
url: https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup
description: 创建可复用、带版本的智能体配置。
---

"Agent"（智能体）是一种可复用、带版本的配置，用于定义角色设定和能力。它将模型、系统提示、工具、MCP 服务器和技能打包在一起，共同决定 Claude 在会话期间的行为方式。

只需将智能体作为可复用资源创建一次，之后每次[启动会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)时通过 ID 引用它即可。智能体带有版本，便于在大量会话中进行管理。

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

## 智能体配置字段

| 字段            | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`        | 必填。智能体的人类可读名称。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `model`       | 必填。驱动该智能体的 Claude [模型](https://platform.claude.com/docs/zh-CN/models/overview)。接受模型 ID 字符串或对象，例如 `{"id": "claude-opus-5"}`。支持 Claude 4.5 及更高版本的模型。对象形式还接受 `speed`、`effort` 和 `inference_geo` 字段；请参阅[创建智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#create-an-agent)下的提示、[Effort 级别](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#effort-levels)以及[固定推理地理区域](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#pin-the-inference-geo)。 |
| `system`      | 定义智能体行为和角色设定的 [system prompt](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#give-claude-a-role)（系统提示）。系统提示不同于[用户消息](https://platform.claude.com/docs/zh-CN/managed-agents/reference#event-types)，后者应描述要完成的工作。                                                                                                                                                                                                                        |
| `tools`       | 智能体可用的工具。组合了[预构建智能体工具](https://platform.claude.com/docs/zh-CN/managed-agents/tools)、[MCP 工具](https://platform.claude.com/docs/zh-CN/managed-agents/mcp-connector)和[自定义工具](https://platform.claude.com/docs/zh-CN/managed-agents/tools#custom-tools)。                                                                                                                                                                                                                                               |
| `mcp_servers` | 提供标准化第三方能力的 [MCP 服务器](https://platform.claude.com/docs/zh-CN/managed-agents/mcp-connector)。                                                                                                                                                                                                                                                                                                                                                                                                        |
| `skills`      | 通过渐进式披露提供领域特定上下文的[技能](https://platform.claude.com/docs/zh-CN/managed-agents/skills)。                                                                                                                                                                                                                                                                                                                                                                                                               |
| `multiagent`  | 协调者声明，列出该智能体可以委派任务的智能体。请参阅[多智能体编排](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)。                                                                                                                                                                                                                                                                                                                                                                                |
| `description` | 对智能体功能的描述。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `metadata`    | 供您自行跟踪使用的任意键值对。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |

您还可以为单个会话覆盖 `model`、`system`、`tools`、`mcp_servers` 和 `skills`，而无需更改智能体。在按会话的 `model` 覆盖中设置的 `effort` 级别不会生效，并且由于该覆盖会完整替换智能体的 `model` 对象，使用 `model` 覆盖创建的会话将以模型的默认 effort 级别运行；若要以特定 effort 级别运行，请在智能体上设置 `effort`，并且不要为该会话覆盖 `model`。请参阅[为会话覆盖智能体配置](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#override-agent-configuration-for-a-session)。

## 创建智能体

以下示例定义了一个编码智能体，它使用 Claude Opus 5 并可访问预构建的智能体工具集。该工具集允许智能体编写代码、读取文件、搜索网络等。有关支持的工具的完整列表，请参阅[智能体工具参考](https://platform.claude.com/docs/zh-CN/managed-agents/tools)。

这些示例使用 curl、`ant` CLI 或某个 SDK。如果您尚未完成设置，[快速入门](https://platform.claude.com/docs/zh-CN/managed-agents/quickstart#install-the-cli)涵盖了安装和客户端设置。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  agent=$(curl -fsSL https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "name": "Coding Assistant",
      "model": "claude-opus-5",
      "system": "You are a helpful coding agent.",
      "tools": [{"type": "agent_toolset_20260401"}]
    }')

  AGENT_ID=$(jq -r '.id' <<< "$agent")
  AGENT_VERSION=$(jq -r '.version' <<< "$agent")
  ```

  <MultiFileExample language="cli" label="CLI">
    ```bash CLI
    agent=$(ant beta:agents create --format json < coding-assistant.agent.yaml)

    AGENT_ID=$(jq -r '.id' <<< "$agent")
    ```

    <File filename="coding-assistant.agent.yaml">
      ```yaml
      name: Coding Assistant
      model:
        id: claude-opus-5
      system: You are a helpful coding agent.
      tools:
        - type: agent_toolset_20260401
      ```
    </File>
  </MultiFileExample>

  ```python Python
  agent = client.beta.agents.create(
      name="Coding Assistant",
      model="claude-opus-5",
      system="You are a helpful coding agent.",
      tools=[
          {"type": "agent_toolset_20260401"},
      ],
  )
  ```

  ```typescript TypeScript
  const agent = await client.beta.agents.create({
    name: "Coding Assistant",
    model: "claude-opus-5",
    system: "You are a helpful coding agent.",
    tools: [{ type: "agent_toolset_20260401" }],
  });
  ```

  ```csharp C#
  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Coding Assistant",
      Model = BetaManagedAgentsModel.ClaudeOpus5,
      System = "You are a helpful coding agent.",
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = "agent_toolset_20260401",
          },
      ],
  });
  ```

  ```go Go
  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "Coding Assistant",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: anthropic.BetaManagedAgentsModelClaudeOpus5,
  	},
  	System: anthropic.String("You are a helpful coding agent."),
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  		OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  			Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var agent = client.beta().agents().create(
      AgentCreateParams.builder()
          .name("Coding Assistant")
          .model(BetaManagedAgentsModel.CLAUDE_OPUS_5)
          .system("You are a helpful coding agent.")
          .addTool(
              BetaManagedAgentsAgentToolset20260401Params.builder()
                  .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
                  .build()
          )
          .build()
  );
  ```

  ```php PHP
  $agent = $client->beta->agents->create(
      name: 'Coding Assistant',
      model: 'claude-opus-5',
      system: 'You are a helpful coding agent.',
      tools: [
          BetaManagedAgentsAgentToolset20260401Params::with(
              type: 'agent_toolset_20260401',
          ),
      ],
  );
  ```

  ```ruby Ruby
  agent = client.beta.agents.create(
    name: "Coding Assistant",
    model: "claude-opus-5",
    system_: "You are a helpful coding agent.",
    tools: [{type: "agent_toolset_20260401"}]
  )
  ```
</CodeGroup>

响应会回显您的配置，并添加 `id`、`type`、`version`、`created_at`、`updated_at` 和 `archived_at` 字段，同时用默认值填充您省略的 `model` 字段（例如 `effort`）。`version` 从 1 开始，每当更新改变了智能体时递增。

```json
{
  "id": "agent_01HqR2k7vXbZ9mNpL3wYcT8f",
  "type": "agent",
  "name": "Coding Assistant",
  "model": {
    "id": "claude-opus-5",
    "effort": { "type": "high" },
    "speed": "standard"
  },
  "system": "You are a helpful coding agent.",
  "description": null,
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": {
        "permission_policy": { "type": "always_allow" }
      }
    }
  ],
  "skills": [],
  "mcp_servers": [],
  "multiagent": null,
  "metadata": {},
  "version": 1,
  "created_at": "2026-04-03T18:24:10.412Z",
  "updated_at": "2026-04-03T18:24:10.412Z",
  "archived_at": null
}
```

工具集上的 `default_config` 显示其默认[权限策略](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies) `always_allow`，除非您另行配置，否则将应用该策略。

<Tip>
  要将 Claude Opus 5 或 Claude Opus 4.8 与[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)配合使用，请以对象形式传递 `model`，例如：`{"id": "claude-opus-5", "speed": "fast"}`。请参阅快速模式页面的[支持的模型](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#supported-models)。
</Tip>

<Tip>
  要设置模型的 effort 级别，请以对象形式传递 `model`，例如：`{"id": "claude-opus-5", "effort": "high"}`。`effort` 字段接受级别字符串（`low`、`medium`、`high`、`xhigh` 或 `max`）或诸如 `{"type": "high"}` 的对象。有关每个级别的作用，请参阅 [Effort 级别](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#effort-levels)。
</Tip>

### 固定推理地理区域

与 `speed` 和 `effort` 一样，`inference_geo`（推理地理区域）通过 `model` 的对象形式设置：以对象形式传递 `model`，并在 `id` 旁设置 `inference_geo`。该字段接受 `"us"` 或 `"global"`。未设置时，每个模型请求在被处理时遵循工作区的默认推理地理区域。有关工作区级别的地理区域控制和定价，请参阅[数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)。

以下示例将智能体固定到美国推理，并打印响应的 `model` 对象中回显的 `inference_geo` 值：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  agent=$(curl -fsSL https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "name": "Geo-pinned assistant",
      "model": {"id": "claude-opus-5", "inference_geo": "us"},
      "system": "You are a helpful assistant."
    }')

  echo "Inference geo: $(jq -r '.model.inference_geo' <<< "$agent")"
  ```

  <MultiFileExample language="cli" label="CLI">
    ```bash CLI
    agent=$(ant beta:agents create --format json < geo-pinned.agent.yaml)

    echo "Inference geo: $(jq -r '.model.inference_geo' <<< "$agent")"
    ```

    <File filename="geo-pinned.agent.yaml">
      ```yaml
      name: Geo-pinned assistant
      model:
        id: claude-opus-5
        inference_geo: us
      system: You are a helpful assistant.
      ```
    </File>
  </MultiFileExample>

  ```python Python
  agent = client.beta.agents.create(
      name="Geo-pinned assistant",
      model={
          "id": "claude-opus-5",
          "inference_geo": "us",
      },
      system="You are a helpful assistant.",
  )

  print(f"Inference geo: {agent.model.inference_geo}")
  ```

  ```typescript TypeScript
  const agent = await client.beta.agents.create({
    name: "Geo-pinned assistant",
    model: { id: "claude-opus-5", inference_geo: "us" },
    system: "You are a helpful assistant.",
  });

  console.log(`Inference geo: ${agent.model.inference_geo}`);
  ```

  ```csharp C#
  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Geo-pinned assistant",
      Model = new BetaManagedAgentsModelConfigParams
      {
          ID = BetaManagedAgentsModel.ClaudeOpus5,
          InferenceGeo = "us",
      },
      System = "You are a helpful assistant.",
  });

  Console.WriteLine($"Inference geo: {agent.Model.InferenceGeo}");
  ```

  ```go Go
  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "Geo-pinned assistant",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID:           anthropic.BetaManagedAgentsModelClaudeOpus5,
  		InferenceGeo: anthropic.String("us"),
  	},
  	System: anthropic.String("You are a helpful assistant."),
  })
  if err != nil {
  	panic(err)
  }

  fmt.Printf("Inference geo: %s\n", agent.Model.InferenceGeo)
  ```

  ```java Java
  var agent = client.beta().agents().create(
      AgentCreateParams.builder()
          .name("Geo-pinned assistant")
          .model(
              BetaManagedAgentsModelConfigParams.builder()
                  .id(BetaManagedAgentsModel.CLAUDE_OPUS_5)
                  .inferenceGeo("us")
                  .build()
          )
          .system("You are a helpful assistant.")
          .build()
  );

  IO.println("Inference geo: " + agent.model().inferenceGeo().orElseThrow());
  ```

  ```php PHP
  $agent = $client->beta->agents->create(
      name: 'Geo-pinned assistant',
      model: BetaManagedAgentsModelConfigParams::with(
          id: 'claude-opus-5',
          inferenceGeo: 'us',
      ),
      system: 'You are a helpful assistant.',
  );

  echo "Inference geo: {$agent->model->inferenceGeo}\n";
  ```

  ```ruby Ruby
  agent = client.beta.agents.create(
    name: "Geo-pinned assistant",
    model: {id: "claude-opus-5", inference_geo: "us"},
    system_: "You are a helpful assistant."
  )

  puts "Inference geo: #{agent.model.inference_geo}"
  ```
</CodeGroup>

`inference_geo` 固定值会在保存智能体时、从其创建会话时以及会话处理的每一轮中，对照工作区的 [`allowed_inference_geos`](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency#workspace-level-restrictions) 进行验证。如果工作区允许列表收窄导致某个固定值不再被允许，则无法从该智能体创建新会话，且正在运行的会话会拒绝后续轮次；固定值永远不会被豁免，因为工作区依赖它们来满足合规和数据驻留要求。

在不支持地理推理固定的模型上设置 `inference_geo` 会返回 400 错误；有关支持的模型，请参阅[模型可用性](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency#model-availability)。在 `multiagent` 配置中，协调者的固定值与每个名册成员的固定值必须全部设置为相同的值，或全部不设置；请参阅[多智能体编排](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)。若要稍后更改或清除固定值，请更新智能体的 `model` 对象；提供不含 `inference_geo` 的 `model` 会将其清除，如[更新语义](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#update-semantics)中所述。

## 更新智能体

当配置发生变化时，更新智能体会生成一个新版本。`version` 字段是可选的：提供它可实现乐观并发控制（不匹配时返回 409），省略它则无条件应用更新（最后写入者获胜）。对已归档智能体的更新会被拒绝。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  updated_agent=$(curl -fsSL "https://api.anthropic.com/v1/agents/$AGENT_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d @- <<EOF
  {
    "version": $AGENT_VERSION,
    "system": "You are a helpful coding agent. Always write tests."
  }
  EOF
  )

  echo "New version: $(jq -r '.version' <<< "$updated_agent")"
  ```

  <MultiFileExample language="cli" label="CLI">
    ```bash CLI
    ant beta:agents update --agent-id "$AGENT_ID" < coding-assistant.agent.yaml
    ```

    <File filename="coding-assistant.agent.yaml">
      ```yaml
      name: Coding Assistant
      model:
        id: claude-opus-5
      system: You are a helpful coding agent. Always write tests.
      tools:
        - type: agent_toolset_20260401
      ```
    </File>
  </MultiFileExample>

  ```python Python
  updated_agent = client.beta.agents.update(
      agent.id,
      version=agent.version,
      system="You are a helpful coding agent. Always write tests.",
  )

  print(f"New version: {updated_agent.version}")
  ```

  ```typescript TypeScript
  const updatedAgent = await client.beta.agents.update(agent.id, {
    version: agent.version,
    system: "You are a helpful coding agent. Always write tests.",
  });

  console.log(`New version: ${updatedAgent.version}`);
  ```

  ```csharp C#
  var updatedAgent = await client.Beta.Agents.Update(agent.ID, new()
  {
      Version = agent.Version,
      System = "You are a helpful coding agent. Always write tests.",
  });

  Console.WriteLine($"New version: {updatedAgent.Version}");
  ```

  ```go Go
  updatedAgent, err := client.Beta.Agents.Update(ctx, agent.ID, anthropic.BetaAgentUpdateParams{
  	Version: anthropic.Int(agent.Version),
  	System:  anthropic.String("You are a helpful coding agent. Always write tests."),
  })
  if err != nil {
  	panic(err)
  }

  fmt.Printf("New version: %d\n", updatedAgent.Version)
  ```

  ```java Java
  var updatedAgent = client.beta().agents().update(
      agent.id(),
      AgentUpdateParams.builder()
          .version(agent.version())
          .system("You are a helpful coding agent. Always write tests.")
          .build()
  );

  IO.println("New version: " + updatedAgent.version());
  ```

  ```php PHP
  $updatedAgent = $client->beta->agents->update(
      $agent->id,
      version: $agent->version,
      system: 'You are a helpful coding agent. Always write tests.',
  );

  echo "New version: {$updatedAgent->version}\n";
  ```

  ```ruby Ruby
  updated_agent = client.beta.agents.update(
    agent.id,
    version: agent.version,
    system_: "You are a helpful coding agent. Always write tests."
  )

  puts "New version: #{updated_agent.version}"
  ```
</CodeGroup>

上述示例提供了来自创建响应的 `version`，因此只有在您读取智能体之后没有其他操作更改过它时，更新才会生效。若要无条件应用更新，请在请求中省略 `version`：

<CodeGroup defaultLanguage="cURL">
  ```bash cURL
  updated_agent=$(curl -fsSL "https://api.anthropic.com/v1/agents/$AGENT_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "description": "Writes and reviews code."
    }')

  echo "New version: $(jq -r '.version' <<< "$updated_agent")"
  ```
</CodeGroup>

### 更新语义

* **`version`** 是可选的，提供时必须至少为 1。提供时，如果它与智能体的当前版本不匹配，请求将返回 409，即使您发送的字段已与存储的值一致也是如此；请重新读取智能体并重试。省略时，更新将无条件应用，最近一次更新会静默替换任何并发更新，且不会向任一调用方返回错误。对于交互式调用方，建议默认提供 `version`；而省略它则适合声明式应用循环，例如同步已签入的智能体定义的 CI 作业，此时由该循环拥有智能体。

* **省略的字段会被保留。** 您只需包含想要更改的字段。

* **标量字段**（`model`、`system`、`name`、`description`）会被新值替换。`system` 和 `description` 可通过传递 `null` 清除。`model` 和 `name` 是必填项，无法清除。在您提供的 `model` 对象中，`effort` 是唯一的例外：如果模型 `id` 未更改，省略 `effort` 会保持存储的 effort 级别不变。如果您更改了模型 `id`，省略的 `effort` 会重置为新模型的默认值。其他 `model` 字段会随对象一起被替换：提供不含 `inference_geo` 的 `model` 会清除智能体的推理地理区域固定值。

* **数组字段**（`tools`、`mcp_servers`、`skills`）会被新数组完全替换。若要完全清除某个数组字段，请传递 `null` 或空数组。

* **`multiagent`** 会被整体替换，包括其 `agents` 名册。传递 `null` 可将其清除。

* **Metadata** 在键级别进行合并。您提供的键会被添加或更新。您省略的键会被保留。若要删除特定键，请将其值设置为 `null`。

* **无操作检测。** 如果更新相对于当前版本没有产生任何变化，则不会创建新版本，并返回现有版本。

* **协调者名册不会被更新。** 在其 `multiagent.agents` 名册中引用此智能体的协调者会保留在协调者创建或上次更新时固定的版本，即使该引用省略了 `version`。若要委派给新版本，请[更新协调者](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#configure-the-coordinator)，使其名册引用新版本。

## 智能体生命周期

| 操作       | 行为                            |
| -------- | ----------------------------- |
| **更新**   | 当配置发生变化时生成新的智能体版本。            |
| **列出版本** | 返回完整的版本历史，以便您跟踪随时间的变化。        |
| **归档**   | 使智能体变为只读。新会话无法引用它，但现有会话会继续运行。 |

### 列出版本

获取完整的版本历史，以跟踪智能体随时间的变化。结果是分页的，SDK 示例会自动获取每一页。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -fsSL "https://api.anthropic.com/v1/agents/$AGENT_ID/versions" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    | jq -r '.data[] | "Version \(.version): \(.updated_at)"'
  ```

  ```bash CLI
  ant beta:agents:versions list --agent-id "$AGENT_ID"
  ```

  ```python Python
  for version in client.beta.agents.versions.list(agent.id):
      print(f"Version {version.version}: {version.updated_at.isoformat()}")
  ```

  ```typescript TypeScript
  for await (const version of client.beta.agents.versions.list(agent.id)) {
    console.log(`Version ${version.version}: ${version.updated_at}`);
  }
  ```

  ```csharp C#
  var versions = await client.Beta.Agents.Versions.List(agent.ID);
  await foreach (var version in versions.Paginate())
  {
      Console.WriteLine($"Version {version.Version}: {version.UpdatedAt:O}");
  }
  ```

  ```go Go
  iter := client.Beta.Agents.Versions.ListAutoPaging(ctx, agent.ID, anthropic.BetaAgentVersionListParams{})
  for iter.Next() {
  	version := iter.Current()
  	fmt.Printf("Version %d: %s\n", version.Version, version.UpdatedAt.Format(time.RFC3339))
  }
  if err := iter.Err(); err != nil {
  	panic(err)
  }
  ```

  ```java Java
  for (var version : client.beta().agents().versions().list(agent.id()).autoPager()) {
      IO.println("Version " + version.version() + ": " + version.updatedAt());
  }
  ```

  ```php PHP
  foreach ($client->beta->agents->versions->list($agent->id)->pagingEachItem() as $version) {
      echo "Version {$version->version}: {$version->updatedAt->format(DateTimeInterface::ATOM)}\n";
  }
  ```

  ```ruby Ruby
  client.beta.agents.versions.list(agent.id).auto_paging_each do |agent_version|
    puts "Version #{agent_version.version}: #{agent_version.updated_at.iso8601}"
  end
  ```
</CodeGroup>

### 归档智能体

归档会使智能体变为只读，且无法撤销。现有会话会继续运行，但新会话无法引用该智能体。响应会将 `archived_at` 设置为归档时间戳。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  archived=$(curl -fsSL -X POST "https://api.anthropic.com/v1/agents/$AGENT_ID/archive" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01")

  echo "Archived at: $(jq -r '.archived_at' <<< "$archived")"
  ```

  ```bash CLI
  ant beta:agents archive --agent-id "$AGENT_ID"
  ```

  ```python Python
  archived = client.beta.agents.archive(agent.id)

  print(f"Archived at: {archived.archived_at.isoformat()}")
  ```

  ```typescript TypeScript
  const archived = await client.beta.agents.archive(agent.id);
  console.log(`Archived at: ${archived.archived_at}`);
  ```

  ```csharp C#
  var archived = await client.Beta.Agents.Archive(agent.ID);
  Console.WriteLine($"Archived at: {archived.ArchivedAt:O}");
  ```

  ```go Go
  archived, err := client.Beta.Agents.Archive(ctx, agent.ID, anthropic.BetaAgentArchiveParams{})
  if err != nil {
  	panic(err)
  }
  fmt.Printf("Archived at: %s\n", archived.ArchivedAt.Format(time.RFC3339))
  ```

  ```java Java
  var archived = client.beta().agents().archive(agent.id());
  IO.println("Archived at: " + archived.archivedAt().orElseThrow());
  ```

  ```php PHP
  $archived = $client->beta->agents->archive($agent->id);

  echo "Archived at: {$archived->archivedAt->format(DateTimeInterface::ATOM)}\n";
  ```

  ```ruby Ruby
  archived = client.beta.agents.archive(agent.id)
  puts "Archived at: #{archived.archived_at.iso8601}"
  ```
</CodeGroup>

## 后续步骤

<CardGroup cols={2}>
  <Card title="工具" icon="tool" href="https://platform.claude.com/docs/zh-CN/managed-agents/tools">
    配置您的智能体可用的工具。
  </Card>

  <Card title="技能" icon="graduation-cap" href="https://platform.claude.com/docs/zh-CN/managed-agents/skills">
    为您的智能体附加可复用的、基于文件系统的专业知识，以支持领域特定的工作流。
  </Card>

  <Card title="启动会话" icon="play" href="https://platform.claude.com/docs/zh-CN/managed-agents/sessions">
    创建会话以运行您的智能体并开始执行任务。
  </Card>

  <Card title="参考" icon="book" href="https://platform.claude.com/docs/zh-CN/managed-agents/reference">
    Claude Managed Agents 的事件类型、自托管 worker CLI 标志、支持的 MCP 服务器类型、速率限制和品牌指南。
  </Card>
</CardGroup>
