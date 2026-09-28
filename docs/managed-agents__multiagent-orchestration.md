---
title: 多智能体编排
url: https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration
description: 在单个会话中协调多个智能体。
featureMetadata:
  topic:
    title: Managed Agents
    url: https://platform.claude.com/docs/en/managed-agents/overview
  status: beta
  betaHeader: managed-agents-2026-04-01
---

"Multiagent orchestration"（多智能体编排）让一个智能体能够与其他智能体协作完成复杂的工作。各智能体可以在各自隔离的上下文中并行运行，这有助于提升输出质量，也可以缩短完成时间。

不确定多智能体方案是否适合您的问题？请参阅[何时使用多智能体系统（以及何时不使用）](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)。

## 工作原理

所有智能体共享同一个沙箱、文件系统和 [vault 凭证](https://platform.claude.com/docs/zh-CN/managed-agents/vaults)，但每个智能体都在自己的 **session thread**（会话线程）中运行，这是一个上下文隔离的事件流，拥有自己的对话历史。协调者在 **primary thread**（主线程）中报告活动（主线程与会话级[事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)相同）；当协调者委派工作时，会在运行时生成额外的线程。

线程是持久的：协调者可以向之前调用过的智能体发送后续消息，而该智能体会保留其先前所有轮次的内容。

每个智能体使用自己的配置：模型、系统提示、工具、MCP 服务器和技能。会话级[智能体配置覆盖](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#override-agent-configuration-for-a-session)是例外；它们适用于协调者及其 `self` 副本。工具、MCP 服务器和上下文不共享。

### 委派什么

多智能体协调最适合复杂任务，这类任务要么需要跨多种界面开展工作，要么由多个范围明确的子任务共同服务于一个总体目标。

效果良好的模式：

* **并行化：** 同时分发相互独立的子任务（搜索多个来源、分析不同的文件），并由协调者综合结果。
* **专业化：** 将任务路由到具有领域专注型系统提示和工具的智能体，例如安全智能体或文档智能体，而不是让单个智能体承载所有能力。
* **升级：** 针对一部分复杂子任务，咨询能力更强的智能体或模型。

## 配置协调者

在[定义智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)时，设置 `multiagent` 来声明协调者可以委派的智能体名册：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  coordinator=$(curl -fsS https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d @- <<EOF
  {
    "name": "Engineering Lead",
    "model": "claude-opus-5-5",
    "system": "You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent.",
    "tools": [
      {
        "type": "agent_toolset_20260401"
      }
    ],
    "multiagent": {
      "type": "coordinator",
      "agents": [
        {"type": "agent", "id": "$REVIEWER_AGENT_ID"},
        {"type": "agent", "id": "$TEST_WRITER_AGENT_ID"}
      ]
    }
  }
  EOF
  )
  ```

  <CodeGroupItem>
    ```bash CLI
    ant apply engineering-lead.md reviewer.md test-writer.md
    ```

    <File filename="engineering-lead.md">
      ```markdown
      ---
      name: Engineering Lead
      model: claude-opus-5-5
      tools:
        - type: agent_toolset_20260401
      multiagent:
        type: coordinator
        agents: # paths: ant apply substitutes {type: agent, id, version}
          - ./reviewer.md
          - ./test-writer.md
      ---

      You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent.
      ```
    </File>

    <File filename="reviewer.md">
      ```markdown
      ---
      name: reviewer
      model: claude-haiku-4-5
      ---

      You are a code reviewer.
      ```
    </File>

    <File filename="test-writer.md">
      ```markdown
      ---
      name: test-writer
      model: claude-haiku-4-5
      ---

      You write unit tests.
      ```
    </File>
  </CodeGroupItem>

  ```python Python
  coordinator = client.beta.agents.create(
      name="Engineering Lead",
      model="claude-opus-5-5",
      system="You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent.",
      tools=[
          {"type": "agent_toolset_20260401"},
      ],
      multiagent={
          "type": "coordinator",
          "agents": [
              {"type": "agent", "id": reviewer_agent.id},
              {"type": "agent", "id": test_writer_agent.id},
          ],
      },
  )
  ```

  ```typescript TypeScript
  const coordinator = await client.beta.agents.create({
    name: "Engineering Lead",
    model: "claude-opus-5-5",
    system:
      "You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent.",
    tools: [{ type: "agent_toolset_20260401" }],
    multiagent: {
      type: "coordinator",
      agents: [
        { type: "agent", id: reviewerAgent.id },
        { type: "agent", id: testWriterAgent.id },
      ],
    },
  });
  ```

  ```csharp C#
  var coordinator = await client.Beta.Agents.Create(new()
  {
      Name = "Engineering Lead",
      Model = BetaManagedAgentsModel.ClaudeOpus5_5,
      System = "You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent.",
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = BetaManagedAgentsAgentToolset20260401ParamsType.AgentToolset20260401,
          },
      ],
      Multiagent = new BetaManagedAgentsMultiagentParams
      {
          Type = BetaManagedAgentsMultiagentParamsType.Coordinator,
          Agents = [reviewerAgent.ID, testWriterAgent.ID],
      },
  });
  ```

  ```go Go
  coordinator, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name:   "Engineering Lead",
  	Model:  anthropic.BetaManagedAgentsModelConfigParams{ID: anthropic.BetaManagedAgentsModelClaudeOpus5_5},
  	System: anthropic.String("You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent."),
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  		OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  			Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  		},
  	}},
  	Multiagent: anthropic.BetaManagedAgentsMultiagentParams{
  		Type: anthropic.BetaManagedAgentsMultiagentParamsTypeCoordinator,
  		Agents: []anthropic.BetaManagedAgentsMultiagentRosterEntryParamsUnion{
  			{OfString: anthropic.String(reviewerAgent.ID)},
  			{OfString: anthropic.String(testWriterAgent.ID)},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var coordinator = client.beta().agents().create(
      AgentCreateParams.builder()
          .name("Engineering Lead")
          .model(BetaManagedAgentsModel.CLAUDE_OPUS_5_5)
          .system("You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent.")
          .addTool(
              BetaManagedAgentsAgentToolset20260401Params.builder()
                  .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
                  .build()
          )
          .multiagent(BetaManagedAgentsMultiagentParams.builder()
              .type(BetaManagedAgentsMultiagentParams.Type.COORDINATOR)
              .addAgent(BetaManagedAgentsAgentParams.builder()
                  .type(BetaManagedAgentsAgentParams.Type.AGENT)
                  .id(reviewerAgent.id())
                  .build())
              .addAgent(BetaManagedAgentsAgentParams.builder()
                  .type(BetaManagedAgentsAgentParams.Type.AGENT)
                  .id(testWriterAgent.id())
                  .build())
              .build())
          .build()
  );
  ```

  ```php PHP
  $coordinator = $client->beta->agents->create(
      name: 'Engineering Lead',
      model: 'claude-opus-5-5',
      system: 'You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent.',
      tools: [
          ['type' => 'agent_toolset_20260401'],
      ],
      multiagent: [
          'type' => 'coordinator',
          'agents' => [
              ['type' => 'agent', 'id' => $reviewerAgent->id],
              ['type' => 'agent', 'id' => $testWriterAgent->id],
          ],
      ],
  );
  ```

  ```ruby Ruby
  coordinator = client.beta.agents.create(
    name: "Engineering Lead",
    model: "claude-opus-5-5",
    system: "You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent.",
    tools: [
      {type: "agent_toolset_20260401"}
    ],
    multiagent: {
      type: "coordinator",
      agents: [
        {type: "agent", id: reviewer_agent.id},
        {type: "agent", id: test_writer_agent.id}
      ]
    }
  )
  ```
</CodeGroup>

`multiagent.agents` 可以接受以下任意形式：

* `{"type": "agent", "id": agent.id}` 通过 ID 引用先前创建的 `agent`。如果未指定 `version`，该引用会固定到创建协调者时该智能体的最新版本。
* `{"type": "agent", "id": agent.id, "version": agent.version}` 固定到特定的智能体版本。
* `{"type": "self"}` 允许协调者生成自身的副本。如果会话是使用[智能体配置覆盖](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#override-agent-configuration-for-a-session)创建的，这些覆盖同样适用于这些副本；通过 ID 引用的名册条目不受影响。
* `{"type": "advisor", "model": "<model id>"}` 为会话的主线程提供一个顾问，主线程可以在轮次进行中咨询它。每个名册最多只能有一个顾问条目。请参阅[为会话提供顾问](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#give-the-session-an-advisor)。

在 [`ant apply`](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/apply) 智能体文件（即 CLI 选项卡）中，名册条目也可以是另一个智能体文件的路径，例如 `./reviewer.md`。Apply 会先创建该智能体，再将路径替换为固定版本的 `{"type": "agent", "id": ..., "version": ...}` 引用。

协调者的配置（包括其 `multiagent.agents` 名册）会在创建或更新协调者时生成快照。被引用的智能体会固定在当时解析出的版本上，不会自动获取其定义的后续更新。如需委派给被引用智能体的较新版本，请[更新协调者](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#update-an-agent)，使其名册引用该版本。

协调者只能向下委派一层智能体。如果引用的智能体本身带有 `multiagent.agents` 名册，创建或更新请求会因验证错误而失败。`multiagent.agents` 中最多可列出 20 个不同的智能体，但协调者可以调用每个智能体的多个副本。

当智能体固定了[推理地理区域](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)（即[智能体定义](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)中的 `model.inference_geo`）时，协调者和每个名册成员的固定值必须全部设为同一个值，或者全部不设置。不匹配的名册会被拒绝并返回 400 验证错误。这一检查既发生在保存智能体时，也发生在[会话创建覆盖](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#override-agent-configuration-for-a-session)更改任一固定值时。

### 为会话提供顾问

`multiagent.agents` 中的顾问条目会为会话的主线程提供一个 **advisor**（顾问）。顾问是一个模型，主线程可以在轮次进行中向它寻求策略性指导，例如规划方法、摆脱困境，或在完成前审查工作。该条目只有两个字段：`type` 和 `model`：

```bash cURL
curl -fsS https://api.anthropic.com/v1/agents \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: managed-agents-2026-04-01" \
  -H "content-type: application/json" \
  -d '{
    "name": "Backend engineer",
    "model": "claude-sonnet-5",
    "system": "You implement backend features end to end. Consult the advisor before major backend design decisions.",
    "multiagent": {
      "type": "coordinator",
      "agents": [
        {"type": "advisor", "model": "claude-opus-5-5"}
      ]
    }
  }'
```

一个名册最多只能包含一个顾问条目，并可与其他任何形式的名册条目并存。该条目占用保留的名册名称 `anthropic.advisor`。如果名册中既有顾问条目，又有一个名称恰好为 `anthropic.advisor` 的成员，该名册会被拒绝并返回 400 验证错误。在响应中，无论提交时顾问条目位于什么位置，它都会出现在名册的最后。

顾问模型必须达到最低能力要求，且智能体自身模型的能力不得高于其顾问；能力相当的模型可以配对。无效的配对会在保存智能体时被拒绝并返回 400 验证错误。有效配对请参阅顾问工具的[模型兼容性](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool#model-compatibility)表。

顾问也可以作为 [Messages API 上的服务器工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool)使用。Managed Agents 中的顾问在配置和交付方式上有所不同：名册条目没有 `max_uses`、`max_tokens` 或 `caching` 字段，建议通过线程事件送达，而不是通过 `advisor_tool_result` 块。

#### 咨询如何运作

每次咨询都作为一个由平台生成、名为 `anthropic.advisor` 的线程运行，该线程在咨询完成时自行终止，建议则以 `agent.thread_message_received` 事件的形式送达主线程。一次咨询会发出标准的线程事件，以保留名称 `anthropic.advisor` 标识（线程生命周期事件将其作为 `agent_name` 携带，建议送达事件将其作为 `from_agent_name` 携带），通常按以下顺序：

1. `session.thread_created`
2. `session.thread_status_running`
3. `agent.thread_message_received`（建议）
4. `session.thread_status_idle`（`stop_reason: end_turn`）
5. `session.thread_status_terminated`

咨询不会发出 `agent.tool_use` 事件，会话的事件流上也不会出现 `agent.thread_message_sent` 事件，因为咨询输入由平台组合而非由智能体发送。如果您列出顾问线程自身的事件，建议也会以 `agent.thread_message_sent` 事件的形式出现在那里。建议送达（事件 3）不保证在顾问线程的 idle 和 terminated 事件之前到达，因此不要将这些事件视为建议已送达的信号。

您的客户端能否读取建议取决于顾问模型的策略，这与 Messages API 顾问工具上的[结果变体](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool#result-variants)划分一致。在那里返回明文结果的顾问模型，在这里会以可读文本内容的形式送达建议；在那里返回已编辑结果的顾问模型，在这里会在每个客户端界面上以 `[{"type": "redacted"}]` 占位符作为消息内容送达，而智能体本身仍会在服务器端读取完整建议。在前面的示例中，Claude Opus 5 是一个返回已编辑结果的顾问，因此您的客户端看到的是占位符，而智能体读取的是完整建议；如果您希望建议在事件流上可读，请改为选择 Claude Opus 4.8 作为顾问。顾问的思考过程永远不会被呈现。客户端不能自行发送 `redacted` 块；包含此类块的事件会被拒绝并返回 400 验证错误。

失败或被中断的咨询永远不会导致智能体的轮次失败：智能体会在收到一条咨询失败的通用通知后继续。咨询期间的会话级 `user.interrupt` 会终止顾问线程且不送达任何建议；带有顾问线程 `session_thread_id` 的 `user.interrupt` 仅放弃该次咨询。

#### 顾问线程

顾问不是名册智能体：它对协调者的 `list_agents` 工具不可见，不能通过 `send_to_agent` 向其发送消息，并且只有会话的主线程可以咨询它。名册智能体不能。

顾问线程不受并发线程限制的约束。它们会出现在会话的[线程列表](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#threads)中，其 `agent` 设置为与配置完全一致的顾问形式（`{"type": "advisor", "model": ...}`），`parent_thread_id` 设置为主线程。

顾问侧的提示缓存是自动的；无需任何配置。咨询按顾问模型的费率计费，其令牌会出现在顾问线程的用量以及会话的用量总计中。

#### 移除顾问

要移除顾问，请使用不再包含顾问条目的名册[更新智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#update-an-agent)。如果顾问是名册中唯一的条目，请通过设置 `"multiagent": null` 完全清空名册。

## 创建会话

创建一个引用协调者的会话。协调者会根据需要委派给其名册中的智能体。

<CodeGroup>
  ```bash cURL
  session=$(curl -fsSL https://api.anthropic.com/v1/sessions \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d @- <<EOF
  {
    "agent": "$COORDINATOR_ID",
    "environment_id": "$ENVIRONMENT_ID"
  }
  EOF
  )
  SESSION_ID=$(jq -r '.id' <<< "$session")
  ```

  ```bash CLI
  ant beta:sessions create \
    --agent "$COORDINATOR_ID" \
    --environment-id "$ENVIRONMENT_ID"
  ```

  ```python Python
  session = client.beta.sessions.create(
      agent=coordinator.id,
      environment_id=environment.id,
  )
  ```

  ```typescript TypeScript
  const session = await client.beta.sessions.create({
    agent: coordinator.id,
    environment_id: environment.id,
  });
  ```

  ```csharp C#
  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = coordinator.ID,
      EnvironmentID = environment.ID,
  });
  ```

  ```go Go
  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent: anthropic.BetaSessionNewParamsAgentUnion{
  		OfString: anthropic.String(coordinator.ID),
  	},
  	EnvironmentID: environment.ID,
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var session = client.beta().sessions().create(SessionCreateParams.builder()
      .agent(coordinator.id())
      .environmentId(environment.id())
      .build());
  ```

  ```php PHP
  $session = $client->beta->sessions->create(
      agent: $coordinator->id,
      environmentID: $environment->id,
  );
  ```

  ```ruby Ruby
  session = client.beta.sessions.create(
    agent: coordinator.id,
    environment_id: environment.id
  )
  ```
</CodeGroup>

## 将智能体连接到 MCP 服务器

MCP 服务器的作用域是智能体（每个智能体定义声明自己的服务器和工具），而 vault 凭证的作用域是会话（创建会话时传入的 `vault_ids` 适用于所有线程）。这对您的集成有两点影响：

* 要对 MCP 服务器进行身份验证，请为所有智能体用到的每个 MCP 服务器都提供一个 vault 凭证。
* 要限制某个智能体的访问权限，请在其智能体定义中只声明它需要的服务器。

创建会话时，[智能体配置覆盖](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#override-agent-configuration-for-a-session)可以替换协调者及其 `self` 副本的 MCP 服务器。

创建声明了 GitHub MCP 服务器的研究员智能体，以及将工作委派给研究员的协调者：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  research_agent_id=$(curl --fail-with-body -sS "$BASE/v1/agents" "${H[@]}" --data @- <<'EOF' | jq -er '.id'
  {
    "name": "researcher",
    "model": "claude-haiku-4-5",
    "mcp_servers": [{"type": "url", "name": "github", "url": "https://api.githubcopilot.com/mcp/"}],
    "tools": [{"type": "mcp_toolset", "mcp_server_name": "github"}]
  }
  EOF
  )

  coordinator_id=$(curl --fail-with-body -sS "$BASE/v1/agents" "${H[@]}" --data @- <<EOF | jq -er '.id'
  {
    "name": "coordinator",
    "model": "claude-opus-5-5",
    "tools": [{"type": "agent_toolset_20260401"}],
    "multiagent": {
      "type": "coordinator",
      "agents": [{"type": "agent", "id": "$research_agent_id"}]
    }
  }
  EOF
  )
  ```

  <CodeGroupItem>
    ```bash CLI
    ant apply coordinator.md researcher.md
    ```

    <File filename="coordinator.md">
      ```markdown
      ---
      name: coordinator
      model: claude-opus-5-5
      tools:
        - type: agent_toolset_20260401
      multiagent:
        type: coordinator
        agents: # path: ant apply substitutes {type: agent, id, version}
          - ./researcher.md
      ---
      ```
    </File>

    <File filename="researcher.md">
      ```markdown
      ---
      name: researcher
      model: claude-haiku-4-5
      mcp_servers:
        - type: url
          name: github
          url: https://api.githubcopilot.com/mcp/
      tools:
        - type: mcp_toolset
          mcp_server_name: github
      ---
      ```
    </File>
  </CodeGroupItem>

  ```python Python
  research_agent = client.beta.agents.create(
      name="researcher",
      model="claude-haiku-4-5",
      mcp_servers=[
          {"type": "url", "name": "github", "url": "https://api.githubcopilot.com/mcp/"},
      ],
      tools=[{"type": "mcp_toolset", "mcp_server_name": "github"}],
  )

  coordinator = client.beta.agents.create(
      name="coordinator",
      model="claude-opus-5-5",
      tools=[{"type": "agent_toolset_20260401"}],
      multiagent={
          "type": "coordinator",
          "agents": [{"type": "agent", "id": research_agent.id}],
      },
  )
  ```

  ```typescript TypeScript
  const researchAgent = await client.beta.agents.create({
    name: "researcher",
    model: "claude-haiku-4-5",
    mcp_servers: [
      { type: "url", name: "github", url: "https://api.githubcopilot.com/mcp/" },
    ],
    tools: [{ type: "mcp_toolset", mcp_server_name: "github" }],
  });

  const coordinator = await client.beta.agents.create({
    name: "coordinator",
    model: "claude-opus-5-5",
    tools: [{ type: "agent_toolset_20260401" }],
    multiagent: {
      type: "coordinator",
      agents: [{ type: "agent", id: researchAgent.id }],
    },
  });
  ```

  ```csharp C#
  var researchAgent = await client.Beta.Agents.Create(new()
  {
      Name = "researcher",
      Model = BetaManagedAgentsModel.ClaudeHaiku4_5,
      McpServers =
      [
          new()
          {
              Type = BetaManagedAgentsUrlMcpServerParamsType.Url,
              Name = "github",
              Url = "https://api.githubcopilot.com/mcp/",
          },
      ],
      Tools =
      [
          new BetaManagedAgentsMcpToolsetParams
          {
              Type = BetaManagedAgentsMcpToolsetParamsType.McpToolset,
              McpServerName = "github",
          },
      ],
  });

  var coordinator = await client.Beta.Agents.Create(new()
  {
      Name = "coordinator",
      Model = BetaManagedAgentsModel.ClaudeOpus5_5,
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = BetaManagedAgentsAgentToolset20260401ParamsType.AgentToolset20260401,
          },
      ],
      Multiagent = new()
      {
          Type = BetaManagedAgentsMultiagentParamsType.Coordinator,
          Agents =
          [
              new BetaManagedAgentsAgentParams
              {
                  Type = BetaManagedAgentsAgentParamsType.Agent,
                  ID = researchAgent.ID,
              },
          ],
      },
  });
  ```

  ```go Go
  researcher, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name:  "researcher",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{ID: anthropic.BetaManagedAgentsModelClaudeHaiku4_5},
  	MCPServers: []anthropic.BetaManagedAgentsURLMCPServerParams{{
  		Type: anthropic.BetaManagedAgentsURLMCPServerParamsTypeURL,
  		Name: "github",
  		URL:  "https://api.githubcopilot.com/mcp/",
  	}},
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  		OfMCPToolset: &anthropic.BetaManagedAgentsMCPToolsetParams{
  			Type:          anthropic.BetaManagedAgentsMCPToolsetParamsTypeMCPToolset,
  			MCPServerName: "github",
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }

  coordinator, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name:  "coordinator",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{ID: anthropic.BetaManagedAgentsModelClaudeOpus5_5},
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  		OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  			Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  		},
  	}},
  	Multiagent: anthropic.BetaManagedAgentsMultiagentParams{
  		Type: anthropic.BetaManagedAgentsMultiagentParamsTypeCoordinator,
  		Agents: []anthropic.BetaManagedAgentsMultiagentRosterEntryParamsUnion{{
  			OfBetaManagedAgentsAgents: &anthropic.BetaManagedAgentsAgentParams{
  				Type: anthropic.BetaManagedAgentsAgentParamsTypeAgent,
  				ID:   researcher.ID,
  			},
  		}},
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var researcher = client.beta().agents().create(
      AgentCreateParams.builder()
          .name("researcher")
          .model(BetaManagedAgentsModel.CLAUDE_HAIKU_4_5)
          .addMcpServer(BetaManagedAgentsUrlMcpServerParams.builder()
              .name("github")
              .type(BetaManagedAgentsUrlMcpServerParams.Type.URL)
              .url("https://api.githubcopilot.com/mcp/")
              .build())
          .addTool(BetaManagedAgentsMcpToolsetParams.builder()
              .type(BetaManagedAgentsMcpToolsetParams.Type.MCP_TOOLSET)
              .mcpServerName("github")
              .build())
          .build()
  );

  var coordinator = client.beta().agents().create(
      AgentCreateParams.builder()
          .name("coordinator")
          .model(BetaManagedAgentsModel.CLAUDE_OPUS_5_5)
          .addTool(BetaManagedAgentsAgentToolset20260401Params.builder()
              .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
              .build())
          .multiagent(BetaManagedAgentsMultiagentParams.builder()
              .type(BetaManagedAgentsMultiagentParams.Type.COORDINATOR)
              .addAgent(BetaManagedAgentsAgentParams.builder()
                  .type(BetaManagedAgentsAgentParams.Type.AGENT)
                  .id(researcher.id())
                  .build())
              .build())
          .build()
  );
  ```

  ```php PHP
  $researchAgent = $client->beta->agents->create(
      name: 'researcher',
      model: 'claude-haiku-4-5',
      mcpServers: [
          ['type' => 'url', 'name' => 'github', 'url' => 'https://api.githubcopilot.com/mcp/'],
      ],
      tools: [
          ['type' => 'mcp_toolset', 'mcp_server_name' => 'github'],
      ],
  );

  $coordinator = $client->beta->agents->create(
      name: 'coordinator',
      model: 'claude-opus-5-5',
      tools: [
          ['type' => 'agent_toolset_20260401'],
      ],
      multiagent: [
          'type' => 'coordinator',
          'agents' => [
              ['type' => 'agent', 'id' => $researchAgent->id],
          ],
      ],
  );
  ```

  ```ruby Ruby
  research_agent = client.beta.agents.create(
    name: "researcher",
    model: "claude-haiku-4-5",
    mcp_servers: [
      {type: "url", name: "github", url: "https://api.githubcopilot.com/mcp/"}
    ],
    tools: [
      {type: "mcp_toolset", mcp_server_name: "github"}
    ]
  )

  coordinator = client.beta.agents.create(
    name: "coordinator",
    model: "claude-opus-5-5",
    tools: [
      {type: "agent_toolset_20260401"}
    ],
    multiagent: {
      type: "coordinator",
      agents: [
        {type: "agent", id: research_agent.id}
      ]
    }
  )
  ```
</CodeGroup>

然后使用存有 GitHub 凭证的 vault 创建会话：

<CodeGroup>
  ```bash cURL
  session_id=$(curl --fail-with-body -sS "$BASE/v1/sessions" "${H[@]}" --data @- <<EOF | jq -er '.id'
  {
    "agent": "$coordinator_id",
    "environment_id": "$environment_id",
    "vault_ids": ["$vault_id"]
  }
  EOF
  )
  echo "$session_id"
  ```

  ```bash CLI
  session_id=$(ant beta:sessions create \
    --agent "$coordinator_id" \
    --environment-id "$environment_id" \
    --vault-id "$vault_id" \
    --transform id --raw-output)
  echo "$session_id"
  ```

  ```python Python
  session = client.beta.sessions.create(
      agent=coordinator.id,
      environment_id=environment.id,
      vault_ids=[vault.id],
  )
  print(session.id)
  ```

  ```typescript TypeScript
  const session = await client.beta.sessions.create({
    agent: coordinator.id,
    environment_id: environment.id,
    vault_ids: [vault.id],
  });
  console.log(session.id);
  ```

  ```csharp C#
  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = coordinator.ID,
      EnvironmentID = environment.ID,
      VaultIds = [vault.ID],
  });
  Console.WriteLine(session.ID);
  ```

  ```go Go
  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent: anthropic.BetaSessionNewParamsAgentUnion{
  		OfString: anthropic.String(coordinator.ID),
  	},
  	EnvironmentID: environment.ID,
  	VaultIDs:      []string{vault.ID},
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(session.ID)
  ```

  ```java Java
  var session = client.beta().sessions().create(SessionCreateParams.builder()
      .agent(coordinator.id())
      .environmentId(environment.id())
      .vaultIds(List.of(vault.id()))
      .build());
  IO.println(session.id());
  ```

  ```php PHP
  $session = $client->beta->sessions->create(
      agent: $coordinator->id,
      environmentID: $environment->id,
      vaultIDs: [$vault->id],
  );
  echo "{$session->id}\n";
  ```

  ```ruby Ruby
  session = client.beta.sessions.create(
    agent: coordinator.id,
    environment_id: environment.id,
    vault_ids: [vault.id]
  )
  puts session.id
  ```
</CodeGroup>

在此示例中，只有研究员声明了 GitHub MCP 服务器，因此协调者无权访问该服务器。会话的 `vault_ids` 会为研究员的线程提供 GitHub 凭证。

<Tip>
  如果声明服务器后，智能体的 MCP 调用仍无法通过身份验证，请确认凭证的 `mcp_server_url` 与智能体的 `mcp_servers[].url` 指向同一服务器。两个 URL 在匹配前都会经过规范化处理：scheme 和主机名转为小写，并去除默认端口和末尾斜杠。因此，主机名大小写、默认端口或末尾斜杠的差异不会影响匹配；但路径、子域名或非默认端口不同则会导致匹配失败。
</Tip>

## 线程

**会话级事件流**（`/v1/sessions/{session_id}/events/stream`）被视为**主线程**，包含所有线程全部活动的精简视图。您看不到子智能体的完整活动，但可以看到它们工作的开始和结束，以及诸如工具权限请求之类的阻塞事件。

**会话线程**是您深入查看特定智能体活动的地方。

会话的 `status` 是所有智能体活动的聚合；如果至少有一个线程处于 `running`，则整个会话状态也为 `running`。

[会话预算](https://platform.claude.com/docs/zh-CN/managed-agents/budgets)是会话所有线程共享的单一上限。达到上限时，各线程独立暂停，每个线程的成本按该线程自身所用的模型定价。

<Note>
  最多支持 25 个并发线程。协调者可以调用名册中单个智能体的多个副本，从而创建与一个 `agent` 关联的多个线程。[顾问](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#give-the-session-an-advisor)咨询线程不受此限制。
</Note>

<Tabs>
  <Tab title="列出线程">
    按如下方式列出与会话关联的所有线程：

    <CodeGroup>
      ```bash cURL
      curl -fsS "https://api.anthropic.com/v1/sessions/$SESSION_ID/threads" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" \
        | jq -r '.data[] | "[\(.agent.name)] \(.status)"'
      ```

      ```bash CLI
      ant beta:sessions:threads list --session-id "$SESSION_ID"
      ```

      ```python Python
      for thread in client.beta.sessions.threads.list(session.id):
          print(f"[{thread.agent.name}] {thread.status}")
      ```

      ```typescript TypeScript
      for await (const thread of client.beta.sessions.threads.list(session.id)) {
        const name = thread.agent.type === "agent" ? thread.agent.name : "advisor";
        console.log(`[${name}] ${thread.status}`);
      }
      ```

      ```csharp C#
      await foreach (var thread in (await client.Beta.Sessions.Threads.List(session.ID)).Paginate())
      {
          Console.WriteLine($"[{thread.Agent.Name}] {thread.Status}");
      }
      ```

      ```go Go
      threads := client.Beta.Sessions.Threads.ListAutoPaging(ctx, session.ID, anthropic.BetaSessionThreadListParams{})
      for threads.Next() {
      	thread := threads.Current()
      	fmt.Printf("[%s] %s\n", thread.Agent.Name, thread.Status)
      }
      if err := threads.Err(); err != nil {
      	panic(err)
      }
      ```

      ```java Java
      for (var thread : client.beta().sessions().threads().list(session.id()).autoPager()) {
          var name = thread.agent().isAgent() ? thread.agent().asAgent().name() : "advisor";
          IO.println("[" + name + "] " + thread.status());
      }
      ```

      ```php PHP
      foreach ($client->beta->sessions->threads->list($session->id)->pagingEachItem() as $thread) {
          echo "[{$thread->agent->name}] {$thread->status}\n";
      }
      ```

      ```ruby Ruby
      client.beta.sessions.threads.list(session.id).auto_paging_each do |thread|
        puts "[#{thread.agent.name}] #{thread.status}"
      end
      ```
    </CodeGroup>

    完整列表包含主线程。主线程的 `parent_thread_id` 为 null。
  </Tab>

  <Tab title="中断会话线程">
    发送带有 `session_thread_id` 的 `user.interrupt` 以停止特定线程。省略 `session_thread_id` 会中断会话中每个未归档的线程，包括主线程。

    <CodeGroup>
      ```bash cURL
      curl -fsS "https://api.anthropic.com/v1/sessions/$SESSION_ID/events?beta=true" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" \
        -H "content-type: application/json" \
        -d "{\"events\": [{\"type\": \"user.interrupt\", \"session_thread_id\": \"$THREAD_ID\"}]}"
      ```

      ```bash CLI
      ant beta:sessions:events send \
        --session-id "$SESSION_ID" \
        --event "{type: user.interrupt, session_thread_id: $THREAD_ID}"
      ```

      ```python Python
      client.beta.sessions.events.send(
          session.id,
          events=[{"type": "user.interrupt", "session_thread_id": thread.id}],
      )
      ```

      ```typescript TypeScript
      await client.beta.sessions.events.send(session.id, {
        events: [{ type: "user.interrupt", session_thread_id: thread.id }],
      });
      ```

      ```csharp C#
      await client.Beta.Sessions.Events.Send(session.ID, new()
      {
          Events =
          [
              new BetaManagedAgentsUserInterruptEventParams
              {
                  Type = BetaManagedAgentsUserInterruptEventParamsType.UserInterrupt,
                  SessionThreadID = thread.ID,
              },
          ],
      });
      ```

      ```go Go
      if _, err := client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
      	Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
      		OfUserInterrupt: &anthropic.BetaManagedAgentsUserInterruptEventParams{
      			Type:            anthropic.BetaManagedAgentsUserInterruptEventParamsTypeUserInterrupt,
      			SessionThreadID: anthropic.String(thread.ID),
      		},
      	}},
      }); err != nil {
      	panic(err)
      }
      ```

      ```java Java
      client.beta().sessions().events().send(
          session.id(),
          EventSendParams.builder()
              .addEvent(BetaManagedAgentsUserInterruptEventParams.builder()
                  .type(BetaManagedAgentsUserInterruptEventParams.Type.USER_INTERRUPT)
                  .sessionThreadId(thread.id())
                  .build())
              .build());
      ```

      ```php PHP
      $client->beta->sessions->events->send(
          $session->id,
          events: [
              ['type' => 'user.interrupt', 'session_thread_id' => $thread->id],
          ],
      );
      ```

      ```ruby Ruby
      client.beta.sessions.events.send_(
        session.id,
        events: [{type: "user.interrupt", session_thread_id: thread.id}]
      )
      ```
    </CodeGroup>

    对于阻塞在 `requires_action` 上的子线程，中断会以错误工具结果（"Tool execution was interrupted before completion. Please retry."）关闭每个待处理的工具调用，并直接重新发出带有 `stop_reason: end_turn` 的 `session.thread_status_idle`；不会对模型进行采样。对于已处于 `idle` 的线程，中断是空操作。
  </Tab>

  <Tab title="归档会话线程">
    可选择在会话线程完成工作后将其归档。这会在 25 个线程的限制中释放一个线程名额。

    <CodeGroup>
      ```bash cURL
      curl -fsS -X POST "https://api.anthropic.com/v1/sessions/$SESSION_ID/threads/$THREAD_ID/archive" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01"
      ```

      ```bash CLI
      ant beta:sessions:threads archive \
        --session-id "$SESSION_ID" \
        --thread-id "$THREAD_ID"
      ```

      ```python Python
      archived = client.beta.sessions.threads.archive(thread.id, session_id=session.id)
      print(archived.status, archived.archived_at)
      ```

      ```typescript TypeScript
      const archived = await client.beta.sessions.threads.archive(thread.id, {
        session_id: session.id,
      });
      console.log(archived.status, archived.archived_at);
      ```

      ```csharp C#
      var archived = await client.Beta.Sessions.Threads.Archive(thread.ID, new() { SessionID = session.ID });
      Console.WriteLine($"{archived.Status} {archived.ArchivedAt}");
      ```

      ```go Go
      archived, err := client.Beta.Sessions.Threads.Archive(ctx, thread.ID, anthropic.BetaSessionThreadArchiveParams{
      	SessionID: session.ID,
      })
      if err != nil {
      	panic(err)
      }
      fmt.Println(archived.Status, archived.ArchivedAt)
      ```

      ```java Java
      var archived = client.beta().sessions().threads().archive(
          thread.id(),
          ThreadArchiveParams.builder()
              .sessionId(session.id())
              .build());
      IO.println(archived.status() + " " + archived.archivedAt().orElseThrow());
      ```

      ```php PHP
      $archived = $client->beta->sessions->threads->archive($thread->id, sessionID: $session->id);
      echo "{$archived->status} {$archived->archivedAt->format(DATE_ATOM)}\n";
      ```

      ```ruby Ruby
      archived = client.beta.sessions.threads.archive(thread.id, session_id: session.id)
      puts "#{archived.status} #{archived.archived_at}"
      ```
    </CodeGroup>

    仅当线程处于 `idle` 时归档才会成功。停留在 `requires_action` 上的线程视为空闲，可以直接归档；只有正在运行的线程必须先被中断：

    <CodeGroup>
      ```bash cURL
      # 中断该线程，然后将其归档
      curl -fsS "https://api.anthropic.com/v1/sessions/$SESSION_ID/events?beta=true" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" \
        -H "content-type: application/json" \
        -d "{\"events\": [{\"type\": \"user.interrupt\", \"session_thread_id\": \"$THREAD_ID\"}]}"

      curl -fsS -X POST "https://api.anthropic.com/v1/sessions/$SESSION_ID/threads/$THREAD_ID/archive" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01"
      ```

      ```bash CLI
      ant beta:sessions:events send \
        --session-id "$SESSION_ID" \
        --event "{type: user.interrupt, session_thread_id: $THREAD_ID}"

      ant beta:sessions:threads archive \
        --session-id "$SESSION_ID" \
        --thread-id "$THREAD_ID"
      ```

      ```python Python
      client.beta.sessions.events.send(
          session.id,
          events=[{"type": "user.interrupt", "session_thread_id": thread.id}],
      )
      archived = client.beta.sessions.threads.archive(thread.id, session_id=session.id)
      print(archived.status, archived.archived_at)
      ```

      ```typescript TypeScript
      await client.beta.sessions.events.send(session.id, {
        events: [{ type: "user.interrupt", session_thread_id: thread.id }],
      });
      const archived = await client.beta.sessions.threads.archive(thread.id, {
        session_id: session.id,
      });
      console.log(archived.status, archived.archived_at);
      ```

      ```csharp C#
      await client.Beta.Sessions.Events.Send(session.ID, new()
      {
          Events =
          [
              new BetaManagedAgentsUserInterruptEventParams
              {
                  Type = BetaManagedAgentsUserInterruptEventParamsType.UserInterrupt,
                  SessionThreadID = thread.ID,
              },
          ],
      });
      archived = await client.Beta.Sessions.Threads.Archive(thread.ID, new() { SessionID = session.ID });
      Console.WriteLine($"{archived.Status} {archived.ArchivedAt}");
      ```

      ```go Go
      if _, err := client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
      	Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
      		OfUserInterrupt: &anthropic.BetaManagedAgentsUserInterruptEventParams{
      			Type:            anthropic.BetaManagedAgentsUserInterruptEventParamsTypeUserInterrupt,
      			SessionThreadID: anthropic.String(thread.ID),
      		},
      	}},
      }); err != nil {
      	panic(err)
      }

      archived, err := client.Beta.Sessions.Threads.Archive(ctx, thread.ID, anthropic.BetaSessionThreadArchiveParams{
      	SessionID: session.ID,
      })
      if err != nil {
      	panic(err)
      }
      fmt.Println(archived.Status, archived.ArchivedAt)
      ```

      ```java Java
      client.beta().sessions().events().send(
          session.id(),
          EventSendParams.builder()
              .addEvent(BetaManagedAgentsUserInterruptEventParams.builder()
                  .type(BetaManagedAgentsUserInterruptEventParams.Type.USER_INTERRUPT)
                  .sessionThreadId(thread.id())
                  .build())
              .build());

      archived = client.beta().sessions().threads().archive(
          thread.id(),
          ThreadArchiveParams.builder()
              .sessionId(session.id())
              .build());
      IO.println(archived.status() + " " + archived.archivedAt().orElseThrow());
      ```

      ```php PHP
      $client->beta->sessions->events->send(
          $session->id,
          events: [['type' => 'user.interrupt', 'session_thread_id' => $thread->id]],
      );
      $archived = $client->beta->sessions->threads->archive($thread->id, sessionID: $session->id);
      echo "{$archived->status} {$archived->archivedAt->format(DATE_ATOM)}\n";
      ```

      ```ruby Ruby
      client.beta.sessions.events.send_(
        session.id,
        events: [{type: "user.interrupt", session_thread_id: thread.id}]
      )
      archived = client.beta.sessions.threads.archive(thread.id, session_id: session.id)
      puts "#{archived.status} #{archived.archived_at}"
      ```
    </CodeGroup>
  </Tab>
</Tabs>

### 主线程事件

这些事件在 `/v1/sessions/{session_id}/events/stream` 的主线程上呈现多智能体活动。消息方向类事件的命名是相对于它们所出现的线程流而言的：`agent.thread_message_received` 表示有一条消息从另一个线程到达此线程，`agent.thread_message_sent` 表示此线程发送了一条消息。例如，协调者委派的任务会以 `agent.thread_message_received` 事件的形式到达子线程自己的流。

| 类型                                 | 描述                                                                                 |
| ---------------------------------- | ---------------------------------------------------------------------------------- |
| `session.thread_created`           | 创建了一个线程。包含 `session_thread_id` 和 `agent_name`。                                     |
| `session.thread_status_running`    | 一个线程开始活动。                                                                          |
| `session.thread_status_idle`       | 与该线程关联的智能体正在等待输入。包含一个 `stop_reason`，指示智能体停止的原因。                                    |
| `session.thread_status_terminated` | 一个线程已被归档或遇到终止性错误。                                                                  |
| `agent.thread_message_received`    | 在主线程上，某个智能体向协调者发送了报告或问题。包含 `from_session_thread_id`、`from_agent_name` 和 `content`。 |
| `agent.thread_message_sent`        | 在主线程上，协调者向另一个智能体发送了任务或后续消息。包含 `to_session_thread_id`、`to_agent_name` 和 `content`。  |

顾问咨询会以保留名称 `anthropic.advisor` 发出同样的这些线程事件（在线程生命周期事件上作为 `agent_name`，在建议送达事件上作为 `from_agent_name`）；事件顺序请参阅[为会话配置顾问](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#give-the-session-an-advisor)。

### 会话线程事件

关键事件会被代理到主线程。不过，您可能仍希望调查特定智能体的推理和工具调用。为此，请流式传输或列出关联会话线程的事件。

每个会话线程在 `/v1/sessions/{session_id}/threads/{thread_id}/stream` 都有自己的事件流，并且它接受与会话级流相同的 `event_deltas[]` 参数，因此您可以在模型生成时预览子智能体的文本。一个连接只预览它正在读取的线程：子线程的预览永远不会出现在会话级流上，因此若要实时观察子智能体，请打开它自己的线程流。有关选择启用、累积和对账预览的信息，请参阅[预览会话线程事件](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#preview-session-thread-events)。

<Tabs>
  <Tab title="流式传输会话线程事件">
    <CodeGroup>
      ```bash cURL
      curl -fsSN "https://api.anthropic.com/v1/sessions/$SESSION_ID/threads/$THREAD_ID/stream?beta=true" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" |
        while IFS= read -r line; do
          [[ $line == data:* ]] || continue
          json=${line#data: }
          case $(jq -r '.type' <<<"$json") in
            agent.message)
              printf '%s' "$(jq -j '.content[] | select(.type == "text") | .text' <<<"$json")"
              ;;
            session.thread_status_idle)
              break
              ;;
          esac
        done
      ```

      ```bash CLI
      ant beta:sessions:threads:events stream \
        --session-id "$SESSION_ID" \
        --thread-id "$THREAD_ID"
      ```

      ```python Python
      with client.beta.sessions.threads.events.stream(
          thread.id,
          session_id=session.id,
      ) as stream:
          for event in stream:
              match event.type:
                  case "agent.message":
                      for block in event.content:
                          if block.type == "text":
                              print(block.text, end="")
                  case "session.thread_status_idle":
                      break
      ```

      ```typescript TypeScript
      const stream = await client.beta.sessions.threads.events.stream(thread.id, {
        session_id: session.id,
      });

      loop: for await (const event of stream) {
        switch (event.type) {
          case "agent.message":
            for (const block of event.content) {
              if (block.type === "text") {
                process.stdout.write(block.text);
              }
            }
            break;
          case "session.thread_status_idle":
            break loop;
        }
      }
      ```

      ```csharp C#
      await foreach (var evt in client.Beta.Sessions.Threads.Events.StreamStreaming(thread.ID, new() { SessionID = session.ID }))
      {
          if (evt.Value is BetaManagedAgentsAgentMessageEvent message)
          {
              foreach (var block in message.Content)
              {
                  if (block.Type == "text")
                  {
                      Console.Write(block.Text);
                  }
              }
          }
          else if (evt.Value is BetaManagedAgentsSessionThreadStatusIdleEvent)
          {
              break;
          }
      }
      ```

      ```go Go
      	stream := client.Beta.Sessions.Threads.Events.StreamEvents(ctx, thread.ID, anthropic.BetaSessionThreadEventStreamParams{
      		SessionID: session.ID,
      	})
      	defer stream.Close()

      loop:
      	for stream.Next() {
      		event := stream.Current()
      		switch event.Type {
      		case "agent.message":
      			for _, block := range event.AsAgentMessage().Content {
      				if block.Type == "text" {
      					fmt.Print(block.Text)
      				}
      			}
      		case "session.thread_status_idle":
      			break loop
      		}
      	}
      	if err := stream.Err(); err != nil {
      		panic(err)
      	}
      ```

      ```java Java
      try (var streamResponse = client.beta().sessions().threads().events().streamStreaming(
          thread.id(),
          EventStreamParams.builder().sessionId(session.id()).build()
      )) {
          loop:
          for (var event : (Iterable<BetaManagedAgentsStreamSessionThreadEvents>) streamResponse.stream()::iterator) {
              switch (event.type().value()) {
                  case AGENT_MESSAGE -> {
                      for (var block : event.asAgentMessage().content()) {
                          block.text().ifPresent(textBlock -> IO.print(textBlock.text()));
                      }
                  }
                  case SESSION_THREAD_STATUS_IDLE -> {
                      break loop;
                  }
              }
          }
      }
      ```

      ```php PHP
      $stream = $client->beta->sessions->threads->events->streamStream(
          $thread->id,
          sessionID: $session->id,
      );

      foreach ($stream as $event) {
          switch (true) {
              case $event instanceof \Anthropic\Beta\Sessions\Events\ManagedAgentsAgentMessageEvent:
                  foreach ($event->content as $block) {
                      if ($block instanceof \Anthropic\Beta\Sessions\Events\ManagedAgentsTextBlock) {
                          echo $block->text;
                      }
                  }
                  break;
              case $event instanceof \Anthropic\Beta\Sessions\Events\ManagedAgentsSessionThreadStatusIdleEvent:
                  break 2;
          }
      }
      ```

      ```ruby Ruby
      client.beta.sessions.threads.events.stream_events(thread.id, session_id: session.id).each do |event|
        case event
        when Anthropic::Beta::Sessions::BetaManagedAgentsAgentMessageEvent
          event.content.each do |block|
            print block.text if block.is_a?(Anthropic::Beta::Sessions::BetaManagedAgentsTextBlock)
          end
        when Anthropic::Beta::Sessions::BetaManagedAgentsSessionThreadStatusIdleEvent
          break
        end
      end
      ```
    </CodeGroup>
  </Tab>

  <Tab title="列出会话线程事件">
    列出所有过去的会话线程事件以获取完整历史。

    <CodeGroup>
      ```bash cURL
      curl -fsS "https://api.anthropic.com/v1/sessions/$SESSION_ID/threads/$THREAD_ID/events" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" \
        | jq -r '.data[] | "[\(.type)] \(.processed_at)"'
      ```

      ```bash CLI
      ant beta:sessions:threads:events list \
        --session-id "$SESSION_ID" \
        --thread-id "$THREAD_ID"
      ```

      ```python Python
      for event in client.beta.sessions.threads.events.list(
          thread.id,
          session_id=session.id,
      ):
          print(f"[{event.type}] {event.processed_at}")
      ```

      ```typescript TypeScript
      for await (const event of client.beta.sessions.threads.events.list(thread.id, {
        session_id: session.id,
      })) {
        console.log(`[${event.type}] ${event.processed_at}`);
      }
      ```

      ```csharp C#
      var page = await client.Beta.Sessions.Threads.Events.List(thread.ID, new() { SessionID = session.ID });
      await foreach (var evt in page.Paginate())
      {
          Console.WriteLine($"[{evt.Type}] {evt.ProcessedAt}");
      }
      ```

      ```go Go
      pager := client.Beta.Sessions.Threads.Events.ListAutoPaging(ctx, thread.ID, anthropic.BetaSessionThreadEventListParams{
      	SessionID: session.ID,
      })
      for pager.Next() {
      	event := pager.Current()
      	fmt.Printf("[%s] %s\n", event.Type, event.ProcessedAt)
      }
      if err := pager.Err(); err != nil {
      	panic(err)
      }
      ```

      ```java Java
      for (var event : client.beta().sessions().threads().events().list(
              thread.id(),
              EventListParams.builder().sessionId(session.id()).build()
          ).autoPager()) {
          var type = event._json().orElseThrow() instanceof JsonObject json
              ? json.values().get("type").asStringOrThrow()
              : "unknown";
          var processedAt = event.processedAt().map(OffsetDateTime::toString).orElse("pending");
          IO.println("[" + type + "] " + processedAt);
      }
      ```

      ```php PHP
      foreach (
          $client->beta->sessions->threads->events->list(
              $thread->id,
              sessionID: $session->id,
          )->pagingEachItem() as $event
      ) {
          echo "[{$event->type}] {$event->processedAt->format(DATE_RFC3339)}\n";
      }
      ```

      ```ruby Ruby
      client.beta.sessions.threads.events.list(
        thread.id,
        session_id: session.id
      ).auto_paging_each do |event|
        puts "[#{event.type}] #{event.processed_at}"
      end
      ```
    </CodeGroup>
  </Tab>
</Tabs>

### 工具权限和自定义工具

如果子智能体需要从您的客户端获取某些内容，例如执行工具调用的[权限](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#tool-confirmation)或[自定义工具的结果](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#handling-custom-tool-calls)，相应事件会被同步发布到**主线程**，并通过 `session_thread_id` 标识其来源会话线程。在 `always_ask` 策略下，或在 [`auto`](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies#let-the-server-evaluate-each-call-with-auto) 策略下服务器无法做出判定时，工具调用需要您的许可。

```json
{
  "type": "session.thread_status_idle",
  "id": "sevt_01ABC...",
  "session_thread_id": "sth_01DEF...",
  "agent_name": "code-reviewer",
  "stop_reason": {
    "type": "requires_action",
    "event_ids": ["sevt_01XYZ..."]
  }
}
```

发送 `user.tool_confirmation`（带 `tool_use_id`）或 `user.custom_tool_result`（带 `custom_tool_use_id`）；服务器会自动将响应路由到正确的线程。

在 `auto` 策略下，您的 `user.message` 事件可能促使服务器允许原本会拒绝的调用。子智能体线程中的任何内容都不会被视为您的意图：您的客户端不会在该线程中发布消息，协调者发给子智能体的消息也不算数。当服务器在 `auto` 策略下拒绝某个调用时，不会同步发布任何内容：该事件及错误工具结果只会出现在子智能体自己的[线程流](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#session-thread-events)上，子智能体会继续运行。

以下示例扩展了[工具确认处理程序](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#tool-confirmation)以路由回复。同样的模式也适用于 `user.custom_tool_result`。

<CodeGroup>
  ```bash cURL
  while IFS= read -r event_id; do
    jq -n --arg id "$event_id" \
      '{events: [{type: "user.tool_confirmation", tool_use_id: $id, result: "allow"}]}' |
      curl -fsS "https://api.anthropic.com/v1/sessions/$SESSION_ID/events?beta=true" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" \
        -H "content-type: application/json" \
        -d @-
  done < <(jq -r '.stop_reason.event_ids[]' <<<"$data")
  ```

  ```bash CLI
  # 此工作流不适合转换为一次性的 shell 命令。
  # 请改用此代码组中的某个 SDK 示例。
  ```

  ```python Python
  for event_id in stop.event_ids:
      client.beta.sessions.events.send(
          session.id,
          events=[
              {
                  "type": "user.tool_confirmation",
                  "tool_use_id": event_id,
                  "result": "allow",
              }
          ],
      )
  ```

  ```typescript TypeScript
  for (const eventId of stop.event_ids) {
    await client.beta.sessions.events.send(session.id, {
      events: [
        {
          type: "user.tool_confirmation",
          tool_use_id: eventId,
          result: "allow",
        },
      ],
    });
  }
  ```

  ```csharp C#
  foreach (var eventId in requiresAction.EventIds)
  {
      await client.Beta.Sessions.Events.Send(session.ID, new()
      {
          Events =
          [
              new BetaManagedAgentsUserToolConfirmationEventParams
              {
                  Type = BetaManagedAgentsUserToolConfirmationEventParamsType.UserToolConfirmation,
                  ToolUseID = eventId,
                  Result = BetaManagedAgentsUserToolConfirmationEventParamsResult.Allow,
              },
          ],
      });
  }
  ```

  ```go Go
  for _, eventID := range stopReason.EventIDs {
  	params := anthropic.BetaManagedAgentsUserToolConfirmationEventParams{
  		Type:      anthropic.BetaManagedAgentsUserToolConfirmationEventParamsTypeUserToolConfirmation,
  		ToolUseID: eventID,
  		Result:    anthropic.BetaManagedAgentsUserToolConfirmationEventParamsResultAllow,
  	}
  	if _, err := client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
  		Events: []anthropic.BetaManagedAgentsEventParamsUnion{{OfUserToolConfirmation: &params}},
  	}); err != nil {
  		panic(err)
  	}
  }
  ```

  ```java Java
  for (var eventId : pendingToolUseIds) {
      client.beta().sessions().events().send(
          session.id(),
          EventSendParams.builder()
              .addEvent(BetaManagedAgentsUserToolConfirmationEventParams.builder()
                  .toolUseId(eventId)
                  .result(BetaManagedAgentsUserToolConfirmationEventParams.Result.ALLOW)
                  .build())
              .build()
      );
  }
  ```

  ```php PHP
  foreach ($event->stopReason->eventIDs as $eventId) {
      $client->beta->sessions->events->send($session->id, events: [[
          'type' => 'user.tool_confirmation',
          'tool_use_id' => $eventId,
          'result' => 'allow',
      ]]);
  }
  ```

  ```ruby Ruby
  event_ids.each do |event_id|
    client.beta.sessions.events.send_(session.id, events: [{
      type: "user.tool_confirmation",
      tool_use_id: event_id,
      result: "allow"
    }])
  end
  ```
</CodeGroup>
