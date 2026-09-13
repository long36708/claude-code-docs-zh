---
title: 迁移
url: https://platform.claude.com/docs/zh-CN/managed-agents/migration
description: 将基于 Messages API 或 Claude Agent SDK 构建的现有智能体迁移到 Claude Managed Agents。
---

Claude Managed Agents 用托管基础设施取代您手写的智能体循环。本页介绍从基于 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 构建的自定义循环或从 [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview) 迁移时会发生哪些变化。

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

## 从 Messages API 智能体循环迁移

如果您通过在 `while` 循环中调用 `messages.create`、自行运行工具调用并将结果追加到对话历史来构建智能体，那么这些代码中的大部分都可以去掉。

### 您不再需要管理的内容

| 之前                                                   | 之后                                                     |
| ---------------------------------------------------- | ------------------------------------------------------ |
| 您维护对话历史数组，并在每一轮中将其传回。                                | 会话在服务器端存储历史。发送事件，接收事件。                                 |
| 您遍历 `tool_use` 内容块，运行每个工具，然后带着 `tool_result` 消息回到循环。 | 预构建工具在沙箱内自动运行。您只需通过 `agent.custom_tool_use` 事件处理自定义工具。 |
| 您自行配置沙箱来运行智能体生成的代码。                                  | 会话沙箱负责处理代码执行、文件操作和 bash。                               |
| 您决定循环何时结束。                                           | 当智能体没有更多事情要做时，会话会发出 `session.status_idle`。             |

### 代码对比

**之前**（Messages API 循环，简化版）：

<CodeGroup>
  ```python Python
  messages = [{"role": "user", "content": task}]
  while True:
      response = client.messages.create(
          model="claude-opus-5",
          max_tokens=1024,
          messages=messages,
          tools=tools,
      )
      messages.append({"role": "assistant", "content": response.content})
      if response.stop_reason == "end_turn":
          break
      for block in response.content:
          if block.type == "tool_use":
              result = execute_tool(block.name, block.input)
              messages.append(
                  {
                      "role": "user",
                      "content": [
                          {
                              "type": "tool_result",
                              "tool_use_id": block.id,
                              "content": result,
                          }
                      ],
                  }
              )
  ```

  ```typescript TypeScript
  const messages: Anthropic.MessageParam[] = [{ role: "user", content: task }];
  while (true) {
    const response = await client.messages.create({
      model: "claude-opus-5",
      max_tokens: 1024,
      messages,
      tools
    });
    messages.push({ role: "assistant", content: response.content });
    if (response.stop_reason === "end_turn") {
      break;
    }
    for (const block of response.content) {
      if (block.type === "tool_use") {
        const result = executeTool(block.name, block.input);
        messages.push({
          role: "user",
          content: [
            {
              type: "tool_result",
              tool_use_id: block.id,
              content: result
            }
          ]
        });
      }
    }
  }
  ```

  ```csharp C#
  List<MessageParam> messages = [new() { Role = Role.User, Content = task }];
  while (true)
  {
      var response = await client.Messages.Create(new()
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 1024,
          Messages = messages,
          Tools = tools,
      });
      messages.Add(new()
      {
          Role = Role.Assistant,
          Content = new([.. response.Content.Select(block => new ContentBlockParam(block.Json))]),
      });
      if (response.StopReason == StopReason.EndTurn)
      {
          break;
      }
      foreach (var block in response.Content)
      {
          if (block.Value is ToolUseBlock toolUse)
          {
              var result = ExecuteTool(toolUse.Name, toolUse.Input);
              messages.Add(new()
              {
                  Role = Role.User,
                  Content = new([new ToolResultBlockParam { ToolUseID = toolUse.ID, Content = result }]),
              });
          }
      }
  }
  ```

  ```go Go
  messages := []anthropic.MessageParam{
  	anthropic.NewUserMessage(anthropic.NewTextBlock(task)),
  }
  for {
  	response, err := client.Messages.New(ctx, anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeOpus5,
  		MaxTokens: 1024,
  		Messages:  messages,
  		Tools:     tools,
  	})
  	if err != nil {
  		log.Fatal(err)
  	}
  	messages = append(messages, response.ToParam())
  	if response.StopReason == anthropic.StopReasonEndTurn {
  		break
  	}
  	for _, block := range response.Content {
  		if toolUse, ok := block.AsAny().(anthropic.ToolUseBlock); ok {
  			result := executeTool(toolUse.Name, toolUse.Input)
  			messages = append(messages, anthropic.NewUserMessage(
  				anthropic.NewToolResultBlock(toolUse.ID, result, false),
  			))
  		}
  	}
  }
  ```

  ```java Java
  var messages = new ArrayList<MessageParam>();
  messages.add(MessageParam.builder()
      .role(MessageParam.Role.USER)
      .content(task)
      .build());
  while (true) {
      var response = client.messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .messages(messages)
          .tools(tools)
          .build());
      messages.add(response.toParam());
      if (StopReason.END_TURN.equals(response.stopReason().orElse(null))) {
          break;
      }
      for (var block : response.content()) {
          block.toolUse().ifPresent(toolUse -> {
              var result = executeTool(toolUse.name(), toolUse._input());
              messages.add(MessageParam.builder()
                  .role(MessageParam.Role.USER)
                  .contentOfBlockParams(List.of(
                      ContentBlockParam.ofToolResult(ToolResultBlockParam.builder()
                          .toolUseId(toolUse.id())
                          .content(result)
                          .build())))
                  .build());
          });
      }
  }
  ```

  ```php PHP
  $messages = [['role' => 'user', 'content' => $task]];
  while (true) {
      $response = $client->messages->create(
          model: 'claude-opus-5',
          maxTokens: 1024,
          messages: $messages,
          tools: $tools,
      );
      $messages[] = ['role' => 'assistant', 'content' => $response->content];
      if ($response->stopReason === 'end_turn') {
          break;
      }
      foreach ($response->content as $block) {
          if ($block->type === 'tool_use') {
              $result = executeTool($block->name, $block->input);
              $messages[] = [
                  'role' => 'user',
                  'content' => [
                      [
                          'type' => 'tool_result',
                          'tool_use_id' => $block->id,
                          'content' => $result,
                      ],
                  ],
              ];
          }
      }
  }
  ```

  ```ruby Ruby
  messages = [{ role: "user", content: task }]
  loop do
    response = client.messages.create(
      model: "claude-opus-5",
      max_tokens: 1024,
      messages: messages,
      tools: tools
    )
    messages << { role: "assistant", content: response.content }
    break if response.stop_reason == :end_turn
    response.content.each do |block|
      next unless block.type == :tool_use
      result = execute_tool(block.name, block.input)
      messages << {
        role: "user",
        content: [
          {
            type: "tool_result",
            tool_use_id: block.id,
            content: result
          }
        ]
      }
    end
  end
  ```
</CodeGroup>

**之后**（Claude Managed Agents）：

<CodeGroup>
  ```bash cURL
  agent=$(
    curl --fail-with-body -sS "https://api.anthropic.com/v1/agents?beta=true" \
      -H "x-api-key: ${ANTHROPIC_API_KEY}" \
      -H "anthropic-version: 2023-06-01" \
      -H "anthropic-beta: managed-agents-2026-04-01" \
      --json '{
        "name": "Task Runner",
        "model": "claude-opus-5",
        "tools": [{"type": "agent_toolset_20260401"}]
      }'
  )
  agent_id=$(jq -r '.id' <<< "${agent}")

  session_id=$(
    curl --fail-with-body -sS "https://api.anthropic.com/v1/sessions?beta=true" \
      -H "x-api-key: ${ANTHROPIC_API_KEY}" \
      -H "anthropic-version: 2023-06-01" \
      -H "anthropic-beta: managed-agents-2026-04-01" \
      --json "$(jq -n --argjson a "${agent}" --arg env "${environment_id}" \
        '{agent: {type: "agent", id: $a.id, version: $a.version}, environment_id: $env}')" \
    | jq -r '.id'
  )

  # 在后台打开 SSE 流，然后发送用户消息。
  stream_log=$(mktemp)
  curl --fail-with-body -sS -N \
    "https://api.anthropic.com/v1/sessions/${session_id}/events/stream?beta=true" \
    -H "x-api-key: ${ANTHROPIC_API_KEY}" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    > "${stream_log}" &
  stream_pid=$!

  curl --fail-with-body -sS \
    "https://api.anthropic.com/v1/sessions/${session_id}/events?beta=true" \
    -H "x-api-key: ${ANTHROPIC_API_KEY}" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    --json "$(jq -n --arg text "${task}" \
      '{events: [{type: "user.message", content: [{type: "text", text: $text}]}]}')" \
    > /dev/null

  # 等待会话进入空闲状态。grep 在首次匹配时退出，并且
  # 通过进程替换读取意味着 shell 不会等待
  # tail（前台的 `tail -f | grep -m1` 管道会挂起：tail
  # 只会在下一次写入时终止，而流空闲后不会再有写入）。
  grep -m1 '"session.status_idle"' <(tail -f -n +1 "${stream_log}") > /dev/null

  kill "${stream_pid}" 2>/dev/null || true
  ```

  <MultiFileExample language="cli" label="CLI">
    ```bash CLI
    { read -r _ agent_id; read -r _ agent_version; } < <(ant beta:agents create \
      --transform '{id,version}' --format yaml < task-runner.agent.yaml)

    session_id=$(ant beta:sessions create \
      --agent "{type: agent, id: $agent_id, version: $agent_version}" \
      --environment-id "$environment_id" \
      --transform id --raw-output)

    # 先打开流，然后发送用户消息
    exec {stream}< <(ant beta:sessions:events stream \
      --session-id "$session_id" \
      --transform type --raw-output)

    ant beta:sessions:events send \
      --session-id "$session_id" \
      --event "{type: user.message, content: [{type: text, text: \"$task\"}]}" \
    > /dev/null

    # 等待会话进入空闲状态（grep 在首次匹配时退出）
    grep -m1 -x 'session.status_idle' <&"$stream" > /dev/null
    exec {stream}<&-
    ```

    <File filename="task-runner.agent.yaml">
      ```yaml
      name: Task Runner
      model: claude-opus-5
      tools:
        - type: agent_toolset_20260401
      ```
    </File>
  </MultiFileExample>

  ```python Python
  agent = client.beta.agents.create(
      name="Task Runner",
      model="claude-opus-5",
      tools=[{"type": "agent_toolset_20260401"}],
  )

  session = client.beta.sessions.create(
      agent={"type": "agent", "id": agent.id, "version": agent.version},
      environment_id=environment.id,
  )

  with client.beta.sessions.events.stream(session.id) as stream:
      client.beta.sessions.events.send(
          session.id,
          events=[{"type": "user.message", "content": [{"type": "text", "text": task}]}],
      )
      for event in stream:
          if event.type == "session.status_idle":
              break
  ```

  ```typescript TypeScript
  const agent = await client.beta.agents.create({
    name: "Task Runner",
    model: "claude-opus-5",
    tools: [{ type: "agent_toolset_20260401" }]
  });

  const session = await client.beta.sessions.create({
    agent: { type: "agent", id: agent.id, version: agent.version },
    environment_id: environment.id
  });

  const stream = await client.beta.sessions.events.stream(session.id);

  await client.beta.sessions.events.send(session.id, {
    events: [
      {
        type: "user.message",
        content: [{ type: "text", text: task }]
      }
    ]
  });

  for await (const event of stream) {
    if (event.type === "session.status_idle") {
      break;
    }
  }
  ```

  ```csharp C#
  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Task Runner",
      Model = BetaManagedAgentsModel.ClaudeOpus5,
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = "agent_toolset_20260401",
          },
      ],
  });

  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = new BetaManagedAgentsAgentParams
      {
          Type = "agent",
          ID = agent.ID,
          Version = agent.Version,
      },
      EnvironmentID = environment.ID,
  });

  var stream = client.Beta.Sessions.Events.StreamStreaming(session.ID);

  await client.Beta.Sessions.Events.Send(session.ID, new()
  {
      Events =
      [
          new BetaManagedAgentsUserMessageEventParams
          {
              Type = "user.message",
              Content = [new BetaManagedAgentsTextBlock { Type = "text", Text = task }],
          },
      ],
  });

  await foreach (var streamEvent in stream)
  {
      if (streamEvent.Value is BetaManagedAgentsSessionStatusIdleEvent)
      {
          break;
      }
  }
  ```

  ```go Go
  	agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  		Name: "Task Runner",
  		Model: anthropic.BetaManagedAgentsModelConfigParams{
  			ID: anthropic.BetaManagedAgentsModelClaudeOpus5,
  		},
  		Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  			OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  				Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  			},
  		}},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  		Agent: anthropic.BetaSessionNewParamsAgentUnion{
  			OfBetaManagedAgentsAgents: &anthropic.BetaManagedAgentsAgentParams{
  				Type:    anthropic.BetaManagedAgentsAgentParamsTypeAgent,
  				ID:      agent.ID,
  				Version: anthropic.Int(agent.Version),
  			},
  		},
  		EnvironmentID: environment.ID,
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	stream := client.Beta.Sessions.Events.StreamEvents(ctx, session.ID, anthropic.BetaSessionEventStreamParams{})
  	defer stream.Close()

  	_, err = client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
  		Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
  			OfUserMessage: &anthropic.BetaManagedAgentsUserMessageEventParams{
  				Type: anthropic.BetaManagedAgentsUserMessageEventParamsTypeUserMessage,
  				Content: []anthropic.BetaManagedAgentsUserMessageEventParamsContentUnion{{
  					OfText: &anthropic.BetaManagedAgentsTextBlockParam{
  						Type: anthropic.BetaManagedAgentsTextBlockTypeText,
  						Text: task,
  					},
  				}},
  			},
  		}},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	for stream.Next() {
  		event := stream.Current()
  		if event.Type == "session.status_idle" {
  			break
  		}
  	}
  	if err := stream.Err(); err != nil {
  		log.Fatal(err)
  	}
  ```

  ```java Java
      var agent = client.beta().agents().create(
          AgentCreateParams.builder()
              .name("Task Runner")
              .model(BetaManagedAgentsModel.CLAUDE_OPUS_5)
              .addTool(
                  BetaManagedAgentsAgentToolset20260401Params.builder()
                      .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
                      .build()
              )
              .build()
      );

      var session = client.beta().sessions().create(
          SessionCreateParams.builder()
              .agent(
                  BetaManagedAgentsAgentParams.builder()
                      .type(BetaManagedAgentsAgentParams.Type.AGENT)
                      .id(agent.id())
                      .version(agent.version())
                      .build()
              )
              .environmentId(environment.id())
              .build()
      );

      try (var stream = client.beta().sessions().events().streamStreaming(session.id())) {
          client.beta().sessions().events().send(
              session.id(),
              EventSendParams.builder()
                  .addEvent(
                      BetaManagedAgentsUserMessageEventParams.builder()
                          .type(BetaManagedAgentsUserMessageEventParams.Type.USER_MESSAGE)
                          .addTextContent(task)
                          .build()
                  )
                  .build()
          );
          stream.stream()
              .takeWhile(event -> !event.isSessionStatusIdle())
              .forEach(_ -> {});
      }
  ```

  ```php PHP
  $agent = $client->beta->agents->create(
      name: 'Task Runner',
      model: 'claude-opus-5',
      tools: [
          BetaManagedAgentsAgentToolset20260401Params::with(
              type: 'agent_toolset_20260401',
          ),
      ],
  );

  $session = $client->beta->sessions->create(
      agent: BetaManagedAgentsAgentParams::with(
          type: 'agent',
          id: $agent->id,
          version: $agent->version,
      ),
      environmentID: $environment->id,
  );

  $stream = $client->beta->sessions->events->streamStream($session->id);

  $client->beta->sessions->events->send(
      $session->id,
      events: [
          [
              'type' => 'user.message',
              'content' => [['type' => 'text', 'text' => $task]],
          ],
      ],
  );

  foreach ($stream as $event) {
      if ($event->type === 'session.status_idle') {
          break;
      }
  }
  ```

  ```ruby Ruby
  agent = client.beta.agents.create(
    name: "Task Runner",
    model: "claude-opus-5",
    tools: [{type: "agent_toolset_20260401"}]
  )

  session = client.beta.sessions.create(
    agent: {type: "agent", id: agent.id, version: agent.version},
    environment_id: environment.id
  )

  stream = client.beta.sessions.events.stream_events(session.id)
  client.beta.sessions.events.send_(
    session.id,
    events: [{type: "user.message", content: [{type: "text", text: task}]}]
  )
  stream.each do
    break if it.type == :"session.status_idle"
  end
  ```
</CodeGroup>

### 您仍然可以控制的内容

* **系统提示和模型：** 字段相同，现在位于智能体定义上。
* **自定义工具：** 仍然使用 JSON Schema 声明。执行方式从内联处理改为响应 `agent.custom_tool_use` 事件。请参阅[会话事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)。
* **网页搜索和网页抓取设置：** 相同的 `allowed_domains`、`blocked_domains`、`max_content_tokens` 和 `user_location` 字段，现在只需在智能体工具集 `configs` 数组的 `web_search` 和 `web_fetch` 条目上设置一次，而不是在每个请求上设置。`max_uses`、`citations` 和 `cache_control` 字段不可用。请参阅[限制网页搜索和网页抓取的域名](https://platform.claude.com/docs/zh-CN/managed-agents/tools#restrict-web-search-and-web-fetch-domains)。
* **上下文：** 您仍然可以通过系统提示、[文件资源](https://platform.claude.com/docs/zh-CN/managed-agents/files)或[技能](https://platform.claude.com/docs/zh-CN/managed-agents/skills)注入上下文。

## 从 Claude Agent SDK 迁移

如果您使用 [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview) 进行构建，那么您已经在使用智能体、工具和会话这些概念。区别在于它们运行的位置：SDK 在您运营的进程中运行，而 Managed Agents 在 Anthropic 的基础设施中运行。迁移的大部分工作是将 SDK 配置对象映射到其 API 端的等价物。

### 发生的变化

| Agent SDK                                        | Managed Agents                                                                                                                                                             |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 每次运行时构造 `ClaudeAgentOptions(...)`                | 调用一次 `client.beta.agents.create(...)`；智能体在服务器端持久化并进行版本管理。请参阅[智能体设置](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)。                                    |
| `async with ClaudeSDKClient(...)` 或 `query(...)` | `client.beta.sessions.create(...)`，然后发送和接收[事件](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)。                                                |
| 由 SDK 自动分派的 `@tool` 装饰函数                         | 在智能体上声明为 `{"type": "custom", ...}`；您的客户端处理 `agent.custom_tool_use` 事件并以 `user.custom_tool_result` 回复。请参阅[工具](https://platform.claude.com/docs/zh-CN/managed-agents/tools)。 |
| 内置工具在您的进程中针对您的文件系统运行                             | `{"type": "agent_toolset_20260401"}` 在会话沙箱内针对 `/workspace` 运行相同的工具。                                                                                                        |
| `cwd`、`add_dirs` 指向本地路径                          | 将[文件](https://platform.claude.com/docs/zh-CN/managed-agents/files)作为会话资源上传或挂载。                                                                                             |
| `system_prompt` 和 `CLAUDE.md` 层级结构               | 智能体上的单个 `system` 字符串。每次更改智能体的更新都会生成一个新的服务器端版本；将会话固定到特定版本，即可在无需部署的情况下升级或回滚。请参阅[智能体设置](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)。                   |
| 在一处配置和认证 `mcp_servers`                           | 在智能体上声明服务器；通过会话上的 [Vault](https://platform.claude.com/docs/zh-CN/managed-agents/vaults) 提供凭据。                                                                              |
| `permission_mode`、`can_use_tool`                 | 按工具设置的 [`permission_policy`](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies)；为 `always_ask` 工具发送 `user.tool_confirmation` 事件。                   |

### 代码对比

**之前**（Agent SDK）：

<CodeGroup exclude="shell, csharp, go, java, php, ruby">
  ```python Python
  from claude_agent_sdk import (
      ClaudeAgentOptions,
      ClaudeSDKClient,
      create_sdk_mcp_server,
      tool,
  )


  @tool("get_weather", "Get the current weather for a city.", {"city": str})
  async def get_weather(args: dict) -> dict:
      return {"content": [{"type": "text", "text": f"{args['city']}: 18°C, clear"}]}


  options = ClaudeAgentOptions(
      model="claude-opus-5",
      system_prompt="You are a concise weather assistant.",
      mcp_servers={
          "weather": create_sdk_mcp_server("weather", "1.0", tools=[get_weather])
      },
  )

  async with ClaudeSDKClient(options=options) as agent:
      await agent.query("What's the weather in Tokyo?")
      async for msg in agent.receive_response():
          print(msg)
  ```

  ```typescript TypeScript
  import { createSdkMcpServer, query, tool } from "@anthropic-ai/claude-agent-sdk";
  import { z } from "zod";

  const getWeather = tool(
    "get_weather",
    "Get the current weather for a city.",
    { city: z.string() },
    async (args) => ({
      content: [{ type: "text", text: `${args.city}: 18°C, clear` }]
    })
  );

  for await (const message of query({
    prompt: "What's the weather in Tokyo?",
    options: {
      model: "claude-opus-5",
      systemPrompt: "You are a concise weather assistant.",
      mcpServers: {
        weather: createSdkMcpServer({ name: "weather", version: "1.0", tools: [getWeather] })
      }
    }
  })) {
    console.log(message);
  }
  ```
</CodeGroup>

**之后**（Managed Agents）：

<CodeGroup exclude="shell">
  ```python Python
  from anthropic import Anthropic

  client = Anthropic()

  agent = client.beta.agents.create(
      name="weather-agent",
      model="claude-opus-5",
      system="You are a concise weather assistant.",
      tools=[
          {
              "type": "custom",
              "name": "get_weather",
              "description": "Get the current weather for a city.",
              "input_schema": {
                  "type": "object",
                  "properties": {"city": {"type": "string"}},
                  "required": ["city"],
              },
          }
      ],
  )
  environment = client.beta.environments.create(
      name="weather-env",
      config={"type": "cloud", "networking": {"type": "unrestricted"}},
  )

  session = client.beta.sessions.create(
      agent={"type": "agent", "id": agent.id, "version": agent.version},
      environment_id=environment.id,
  )


  def get_weather(city: str) -> str:
      return f"{city}: 18°C, clear"


  with client.beta.sessions.events.stream(session.id) as stream:
      client.beta.sessions.events.send(
          session.id,
          events=[
              {
                  "type": "user.message",
                  "content": [{"type": "text", "text": "What's the weather in Tokyo?"}],
              }
          ],
      )
      for event in stream:
          if event.type == "agent.message":
              print(
                  "".join(block.text for block in event.content if block.type == "text")
              )
          elif event.type == "agent.custom_tool_use":
              result = get_weather(**event.input)
              client.beta.sessions.events.send(
                  session.id,
                  events=[
                      {
                          "type": "user.custom_tool_result",
                          "custom_tool_use_id": event.id,
                          "content": [{"type": "text", "text": result}],
                      }
                  ],
              )
          elif (
              event.type == "session.status_idle"
              and event.stop_reason
              and event.stop_reason.type == "end_turn"
          ):
              break
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";

  const client = new Anthropic();

  const agent = await client.beta.agents.create({
    name: "weather-agent",
    model: "claude-opus-5",
    system: "You are a concise weather assistant.",
    tools: [
      {
        type: "custom",
        name: "get_weather",
        description: "Get the current weather for a city.",
        input_schema: {
          type: "object",
          properties: { city: { type: "string" } },
          required: ["city"]
        }
      }
    ]
  });
  const environment = await client.beta.environments.create({
    name: "weather-env",
    config: { type: "cloud", networking: { type: "unrestricted" } }
  });

  const session = await client.beta.sessions.create({
    agent: { type: "agent", id: agent.id, version: agent.version },
    environment_id: environment.id
  });

  function getWeather({ city }: Record<string, unknown>): string {
    return `${city}: 18°C, clear`;
  }

  const stream = await client.beta.sessions.events.stream(session.id);

  await client.beta.sessions.events.send(session.id, {
    events: [
      {
        type: "user.message",
        content: [{ type: "text", text: "What's the weather in Tokyo?" }]
      }
    ]
  });

  for await (const event of stream) {
    if (event.type === "agent.message") {
      for (const block of event.content) {
        if (block.type === "text") {
          console.log(block.text);
        }
      }
    } else if (event.type === "agent.custom_tool_use") {
      const result = getWeather(event.input);
      await client.beta.sessions.events.send(session.id, {
        events: [
          {
            type: "user.custom_tool_result",
            custom_tool_use_id: event.id,
            content: [{ type: "text", text: result }]
          }
        ]
      });
    } else if (event.type === "session.status_idle" && event.stop_reason?.type === "end_turn") {
      break;
    }
  }
  ```

  ```csharp C#
  using System.Text.Json;

  using Anthropic.Models.Beta.Agents;
  using Anthropic.Models.Beta.Environments;
  using Anthropic.Models.Beta.Sessions;
  using Anthropic.Models.Beta.Sessions.Events;

  AnthropicClient client = new();

  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "weather-agent",
      Model = BetaManagedAgentsModel.ClaudeOpus5,
      System = "You are a concise weather assistant.",
      Tools =
      [
          new BetaManagedAgentsCustomToolParams
          {
              Type = "custom",
              Name = "get_weather",
              Description = "Get the current weather for a city.",
              InputSchema = new()
              {
                  Properties = new Dictionary<string, JsonElement>
                  {
                      ["city"] = JsonSerializer.SerializeToElement(new { type = "string" }),
                  },
                  Required = ["city"],
              },
          },
      ],
  });
  var environment = await client.Beta.Environments.Create(new()
  {
      Name = "weather-env",
      Config = new BetaCloudConfigParams
      {
          Networking = new BetaUnrestrictedNetwork(),
      },
  });

  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = new BetaManagedAgentsAgentParams
      {
          Type = "agent",
          ID = agent.ID,
          Version = agent.Version,
      },
      EnvironmentID = environment.ID,
  });

  static string GetWeather(string city) => $"{city}: 18°C, clear";

  using var stream = await client.Beta.Sessions.Events.WithRawResponse.StreamStreaming(session.ID);

  await client.Beta.Sessions.Events.Send(session.ID, new()
  {
      Events =
      [
          new BetaManagedAgentsUserMessageEventParams
          {
              Type = "user.message",
              Content = [new BetaManagedAgentsTextBlock { Type = "text", Text = "What's the weather in Tokyo?" }],
          },
      ],
  });

  await foreach (var streamEvent in stream.Enumerate())
  {
      if (streamEvent.Value is BetaManagedAgentsAgentMessageEvent message)
      {
          Console.WriteLine(string.Concat(message.Content.Select(block => block.Text)));
      }
      else if (streamEvent.Value is BetaManagedAgentsAgentCustomToolUseEvent toolUse)
      {
          var result = GetWeather(toolUse.Input["city"].GetString()!);
          await client.Beta.Sessions.Events.Send(session.ID, new()
          {
              Events =
              [
                  new BetaManagedAgentsUserCustomToolResultEventParams
                  {
                      Type = "user.custom_tool_result",
                      CustomToolUseID = toolUse.ID,
                      Content =
                      [
                          new BetaManagedAgentsTextBlock
                          {
                              Type = "text",
                              Text = result,
                          },
                      ],
                  },
              ],
          });
      }
      else if (streamEvent.Value is BetaManagedAgentsSessionStatusIdleEvent idle
          && idle.StopReason?.Value is BetaManagedAgentsSessionEndTurn)
      {
          break;
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()
  ctx := context.Background()

  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "weather-agent",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: anthropic.BetaManagedAgentsModelClaudeOpus5,
  	},
  	System: anthropic.String("You are a concise weather assistant."),
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  		OfCustom: &anthropic.BetaManagedAgentsCustomToolParams{
  			Type:        anthropic.BetaManagedAgentsCustomToolParamsTypeCustom,
  			Name:        "get_weather",
  			Description: "Get the current weather for a city.",
  			InputSchema: anthropic.BetaManagedAgentsCustomToolInputSchemaParam{
  				Properties: map[string]any{
  					"city": map[string]any{"type": "string"},
  				},
  				Required: []string{"city"},
  			},
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }
  environment, err := client.Beta.Environments.New(ctx, anthropic.BetaEnvironmentNewParams{
  	Name: "weather-env",
  	Config: anthropic.BetaEnvironmentNewParamsConfigUnion{
  		OfCloud: &anthropic.BetaCloudConfigParams{
  			Networking: anthropic.BetaCloudConfigParamsNetworkingUnion{
  				OfUnrestricted: &anthropic.BetaUnrestrictedNetworkParam{},
  			},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }

  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent: anthropic.BetaSessionNewParamsAgentUnion{
  		OfBetaManagedAgentsAgents: &anthropic.BetaManagedAgentsAgentParams{
  			Type:    anthropic.BetaManagedAgentsAgentParamsTypeAgent,
  			ID:      agent.ID,
  			Version: anthropic.Int(agent.Version),
  		},
  	},
  	EnvironmentID: environment.ID,
  })
  if err != nil {
  	panic(err)
  }

  getWeather := func(city string) string {
  	return fmt.Sprintf("%s: 18°C, clear", city)
  }

  stream := client.Beta.Sessions.Events.StreamEvents(ctx, session.ID, anthropic.BetaSessionEventStreamParams{})
  defer stream.Close()

  _, err = client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
  	Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
  		OfUserMessage: &anthropic.BetaManagedAgentsUserMessageEventParams{
  			Type: anthropic.BetaManagedAgentsUserMessageEventParamsTypeUserMessage,
  			Content: []anthropic.BetaManagedAgentsUserMessageEventParamsContentUnion{{
  				OfText: &anthropic.BetaManagedAgentsTextBlockParam{
  					Type: anthropic.BetaManagedAgentsTextBlockTypeText,
  					Text: "What's the weather in Tokyo?",
  				},
  			}},
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }

  loop:
  for stream.Next() {
  	event := stream.Current()
  	switch event.Type {
  	case "agent.message":
  		for _, block := range event.AsAgentMessage().Content {
  			if block.Type == "text" {
  				fmt.Println(block.Text)
  			}
  		}
  	case "agent.custom_tool_use":
  		toolUse := event.AsAgentCustomToolUse()
  		result := getWeather(toolUse.Input["city"].(string))
  		if _, err := client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
  			Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
  				OfUserCustomToolResult: &anthropic.BetaManagedAgentsUserCustomToolResultEventParams{
  					Type:            anthropic.BetaManagedAgentsUserCustomToolResultEventParamsTypeUserCustomToolResult,
  					CustomToolUseID: toolUse.ID,
  					Content: []anthropic.BetaManagedAgentsUserCustomToolResultEventParamsContentUnion{{
  						OfText: &anthropic.BetaManagedAgentsTextBlockParam{
  							Type: anthropic.BetaManagedAgentsTextBlockTypeText,
  							Text: result,
  						},
  					}},
  				},
  			}},
  		}); err != nil {
  			panic(err)
  		}
  	case "session.status_idle":
  		idle := event.AsSessionStatusIdle()
  		if _, ok := idle.StopReason.AsAny().(anthropic.BetaManagedAgentsSessionEndTurn); ok {
  			break loop
  		}
  	}
  }
  if err := stream.Err(); err != nil {
  	panic(err)
  }
  ```

  ```java Java
  import java.util.Map;
  import java.util.function.Function;

  import com.anthropic.models.beta.agents.AgentCreateParams;
  import com.anthropic.models.beta.agents.BetaManagedAgentsCustomToolInputSchema;
  import com.anthropic.models.beta.agents.BetaManagedAgentsCustomToolParams;
  import com.anthropic.models.beta.agents.BetaManagedAgentsModel;
  import com.anthropic.models.beta.environments.BetaCloudConfigParams;
  import com.anthropic.models.beta.environments.BetaUnrestrictedNetwork;
  import com.anthropic.models.beta.environments.EnvironmentCreateParams;
  import com.anthropic.models.beta.sessions.BetaManagedAgentsAgentParams;
  import com.anthropic.models.beta.sessions.SessionCreateParams;
  import com.anthropic.models.beta.sessions.events.BetaManagedAgentsStreamSessionEvents;
  import com.anthropic.models.beta.sessions.events.BetaManagedAgentsUserCustomToolResultEventParams;
  import com.anthropic.models.beta.sessions.events.BetaManagedAgentsUserMessageEventParams;
  import com.anthropic.models.beta.sessions.events.EventSendParams;

  var client = AnthropicOkHttpClient.fromEnv();

  var agent = client.beta().agents().create(AgentCreateParams.builder()
      .name("weather-agent")
      .model(BetaManagedAgentsModel.CLAUDE_OPUS_5)
      .system("You are a concise weather assistant.")
      .addTool(BetaManagedAgentsCustomToolParams.builder()
          .type(BetaManagedAgentsCustomToolParams.Type.CUSTOM)
          .name("get_weather")
          .description("Get the current weather for a city.")
          .inputSchema(BetaManagedAgentsCustomToolInputSchema.builder()
              .properties(BetaManagedAgentsCustomToolInputSchema.Properties.builder()
                  .putAdditionalProperty("city", JsonValue.from(Map.of("type", "string")))
                  .build())
              .addRequired("city")
              .build())
          .build())
      .build());
  var environment = client.beta().environments().create(EnvironmentCreateParams.builder()
      .name("weather-env")
      .config(BetaCloudConfigParams.builder()
          .networking(BetaUnrestrictedNetwork.builder().build())
          .build())
      .build());

  var session = client.beta().sessions().create(SessionCreateParams.builder()
      .agent(BetaManagedAgentsAgentParams.builder()
          .type(BetaManagedAgentsAgentParams.Type.AGENT)
          .id(agent.id())
          .version(agent.version())
          .build())
      .environmentId(environment.id())
      .build());

  Function<String, String> getWeather = city -> city + ": 18°C, clear";

  try (var stream = client.beta().sessions().events().streamStreaming(session.id())) {
      client.beta().sessions().events().send(
          session.id(),
          EventSendParams.builder()
              .addEvent(BetaManagedAgentsUserMessageEventParams.builder()
                  .type(BetaManagedAgentsUserMessageEventParams.Type.USER_MESSAGE)
                  .addTextContent("What's the weather in Tokyo?")
                  .build())
              .build());

      for (var event : (Iterable<BetaManagedAgentsStreamSessionEvents>) stream.stream()::iterator) {
          if (event.isAgentMessage()) {
              for (var block : event.asAgentMessage().content()) {
                  block.text().ifPresent(textBlock -> IO.println(textBlock.text()));
              }
          } else if (event.isAgentCustomToolUse()) {
              var toolUse = event.asAgentCustomToolUse();
              var city = toolUse.input()._additionalProperties().get("city").asStringOrThrow();
              var result = getWeather.apply(city);
              client.beta().sessions().events().send(
                  session.id(),
                  EventSendParams.builder()
                      .addEvent(BetaManagedAgentsUserCustomToolResultEventParams.builder()
                          .type(BetaManagedAgentsUserCustomToolResultEventParams.Type.USER_CUSTOM_TOOL_RESULT)
                          .customToolUseId(toolUse.id())
                          .addTextContent(result)
                          .build())
                      .build());
          } else if (event.isSessionStatusIdle()
              && event.asSessionStatusIdle().stopReason().isEndTurn()) {
              break;
          }
      }
  }
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Beta\Agents\BetaManagedAgentsCustomToolInputSchema;
  use Anthropic\Beta\Agents\BetaManagedAgentsCustomToolParams;
  use Anthropic\Beta\Sessions\BetaManagedAgentsAgentParams;

  $client = new Client();

  $agent = $client->beta->agents->create(
      name: 'weather-agent',
      model: 'claude-opus-5',
      system: 'You are a concise weather assistant.',
      tools: [
          BetaManagedAgentsCustomToolParams::with(
              type: 'custom',
              name: 'get_weather',
              description: 'Get the current weather for a city.',
              inputSchema: BetaManagedAgentsCustomToolInputSchema::with(
                  properties: ['city' => ['type' => 'string']],
                  required: ['city'],
              ),
          ),
      ],
  );
  $environment = $client->beta->environments->create(
      name: 'weather-env',
      config: ['type' => 'cloud', 'networking' => ['type' => 'unrestricted']],
  );

  $session = $client->beta->sessions->create(
      agent: BetaManagedAgentsAgentParams::with(
          type: 'agent',
          id: $agent->id,
          version: $agent->version,
      ),
      environmentID: $environment->id,
  );

  function getWeather(string $city): string
  {
      return "{$city}: 18°C, clear";
  }

  $stream = $client->beta->sessions->events->streamStream($session->id);

  $client->beta->sessions->events->send(
      $session->id,
      events: [
          [
              'type' => 'user.message',
              'content' => [['type' => 'text', 'text' => "What's the weather in Tokyo?"]],
          ],
      ],
  );

  foreach ($stream as $event) {
      if ($event->type === 'agent.message') {
          foreach ($event->content as $block) {
              if ($block->type === 'text') {
                  echo $block->text . "\n";
              }
          }
      } elseif ($event->type === 'agent.custom_tool_use') {
          $result = getWeather($event->input['city']);
          $client->beta->sessions->events->send(
              $session->id,
              events: [
                  [
                      'type' => 'user.custom_tool_result',
                      'custom_tool_use_id' => $event->id,
                      'content' => [['type' => 'text', 'text' => $result]],
                  ],
              ],
          );
      } elseif ($event->type === 'session.status_idle' && $event->stopReason?->type === 'end_turn') {
          break;
      }
  }
  $stream->close();
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::Client.new

  agent = client.beta.agents.create(
    name: "weather-agent",
    model: "claude-opus-5",
    system_: "You are a concise weather assistant.",
    tools: [
      {
        type: "custom",
        name: "get_weather",
        description: "Get the current weather for a city.",
        input_schema: {
          type: "object",
          properties: {city: {type: "string"}},
          required: ["city"]
        }
      }
    ]
  )
  environment = client.beta.environments.create(
    name: "weather-env",
    config: {type: "cloud", networking: {type: "unrestricted"}}
  )

  session = client.beta.sessions.create(
    agent: {type: "agent", id: agent.id, version: agent.version},
    environment_id: environment.id
  )

  def get_weather(city)
    "#{city}: 18°C, clear"
  end

  stream = client.beta.sessions.events.stream_events(session.id)
  client.beta.sessions.events.send_(
    session.id,
    events: [{type: "user.message", content: [{type: "text", text: "What's the weather in Tokyo?"}]}]
  )

  stream.each do |event|
    case event.type
    when :"agent.message"
      event.content.each do |block|
        puts block.text if block.type == :text
      end
    when :"agent.custom_tool_use"
      result = get_weather(event.input[:city])
      client.beta.sessions.events.send_(
        session.id,
        events: [
          {
            type: "user.custom_tool_result",
            custom_tool_use_id: event.id,
            content: [{type: "text", text: result}]
          }
        ]
      )
    when :"session.status_idle"
      break if event.stop_reason&.type == :end_turn
    end
  end
  ```
</CodeGroup>

智能体和环境只需创建一次，即可在多个会话中重复使用。工具函数仍然在您的进程中运行；区别在于您需要读取 `agent.custom_tool_use` 事件并显式发送结果，而不是由 SDK 为您分派。

### 转移到您客户端的功能

由 Anthropic 运行智能体循环的代价是，SDK 原本自动处理的一些事项变成了您客户端的职责。

| SDK 功能                          | Managed Agents 方式                                                                                |
| ------------------------------- | ------------------------------------------------------------------------------------------------ |
| 计划模式                            | 先运行一个仅做规划的会话，然后运行第二个会话来执行该计划。                                                                    |
| 输出样式、斜杠命令                       | 在发送 `user.message` 之前或接收 `agent.message` 之后在您的客户端中应用。                                            |
| `PreToolUse` / `PostToolUse` 钩子 | 您的客户端在响应之前已经能看到每个 `agent.custom_tool_use` 事件；将逻辑放在那里。对于内置工具，请使用 `permission_policy: always_ask`。 |
| `max_turns`                     | 在客户端统计轮次。                                                                                        |

## 迁移清单

1. [创建一个环境](https://platform.claude.com/docs/zh-CN/managed-agents/environments)，包含您的智能体所需的网络和运行时。
2. 将您的系统提示和工具选择移植到[智能体定义](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)中。
3. 用 [`sessions.create`](https://platform.claude.com/docs/zh-CN/managed-agents/sessions) 和 [`sessions.events.stream`](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming) 替换您的循环。
4. 对于智能体读取的任何本地文件，通过 [Files API](https://platform.claude.com/docs/zh-CN/managed-agents/files) 上传并将其挂载为 `resources`。
5. 对于任何自定义工具处理程序，将执行移入您的事件循环中，作为对 `agent.custom_tool_use` 事件的响应。
6. 在将生产流量指向新流程之前，先用测试会话进行验证。

## 在模型版本之间迁移

当新的 Claude 模型发布时，迁移 Claude Managed Agents 集成通常只需更改一个字段：更新[智能体定义](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)上的 `model`，该更改将在您创建的下一个会话中生效。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -sS --fail-with-body "https://api.anthropic.com/v1/agents/$AGENT_ID?beta=true" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    --json "$(jq -n --argjson version "$AGENT_VERSION" '{version: $version, model: "claude-opus-5"}')"
  ```

  <MultiFileExample language="cli" label="CLI">
    ```bash CLI
    ant beta:agents update --agent-id "$AGENT_ID" < agent.yaml
    ```

    <File filename="agent.yaml">
      ```yaml
      name: Task Runner
      model: claude-opus-5
      system: You are a task automation agent. Complete the task you are given end to end.
      tools:
        - type: agent_toolset_20260401
      ```
    </File>
  </MultiFileExample>

  ```python Python
  client.beta.agents.update(
      agent.id,
      version=agent.version,
      model="claude-opus-5",
  )
  ```

  ```typescript TypeScript
  await client.beta.agents.update(agent.id, {
    version: agent.version,
    model: "claude-opus-5"
  });
  ```

  ```csharp C#
  await client.Beta.Agents.Update(agent.ID, new()
  {
      Version = agent.Version,
      Model = BetaManagedAgentsModel.ClaudeOpus5,
  });
  ```

  ```go Go
  _, err = client.Beta.Agents.Update(ctx, agent.ID, anthropic.BetaAgentUpdateParams{
  	Version: agent.Version,
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: anthropic.BetaManagedAgentsModelClaudeOpus5,
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().agents().update(
      agent.id(),
      AgentUpdateParams.builder()
          .version(agent.version())
          .model(BetaManagedAgentsModel.CLAUDE_OPUS_5)
          .build()
  );
  ```

  ```php PHP
  $client->beta->agents->update(
      $agent->id,
      version: $agent->version,
      model: 'claude-opus-5',
  );
  ```

  ```ruby Ruby
  client.beta.agents.update(
    agent.id,
    version: agent.version,
    model: "claude-opus-5"
  )
  ```
</CodeGroup>

[Messages API 迁移指南](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)中记录的大多数模型级行为变化不需要您采取任何行动：

* **请求参数变化**（`max_tokens` 默认值、`thinking` 配置）由 Claude Managed Agents 运行时处理。这些字段不会在智能体定义上公开。
* **助手消息预填充**在基于事件的会话模型中不存在，因此在较新模型上移除该功能不会产生任何影响。
* **工具参数 JSON 转义**在您收到 `agent.custom_tool_use` 事件之前已由运行时解析。您看到的是结构化数据，而不是原始字符串。

Messages API 指南中的行为描述（模型有哪些不同的表现）仍然适用。迁移步骤（如何更改您的请求代码）则不适用。
