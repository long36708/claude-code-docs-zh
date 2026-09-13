---
title: 对话中途的系统消息与工具变更
url: https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages
description: 在对话进行到一半时更改系统指令或工具可用性，而不会使其之前的缓存前缀失效。
---

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

系统指令通常位于顶层 `system` 字段中，排在对话中所有消息之前。这个位置非常适合 [prompt caching](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)（提示缓存）："system prompt"（系统提示）是稳定前缀的一部分，因此后续轮次会命中缓存。但对于那些您在会话进行到一半时才发现需要的指令来说，这个位置并不理想，因为编辑顶层 `system` 字段会改变提示的最开头部分，并使其后所有内容的缓存失效。

对话中途的系统消息弥补了这一缺口。您可以在对话中新指令开始变得相关的位置追加一条 `{"role": "system"}` 消息，而不是编辑顶层 `system` 字段。缓存前缀保持不变，因此下一个请求仍然可以从缓存中读取它，而新指令仍然作为系统指令被应用，而不是作为普通的用户文本。

<Note>
  对话中途的系统消息可在 Claude API、[Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 和 [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 上使用。

  此功能可在 Claude Fable 5.1、[Claude Mythos 5.1](https://anthropic.com/glasswing)、Claude Fable 5、[Claude Mythos 5](https://anthropic.com/glasswing)、Claude Opus 4.8 和 Claude Opus 5 上使用。对话中途的系统消息不需要 beta 标头。此功能在 Claude Sonnet 5 上不可用。在该模型上请改用顶层 `system` 字段。

  [对话中途的工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes)处于 beta 阶段，需要 `mid-conversation-tool-changes-2026-07-01` beta 标头。它们可在相同的模型上使用，支持 Claude API、Amazon Bedrock 和 Google Cloud。

  [轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)（`clear_at`）处于 beta 阶段，需要 `mid-conversation-system-clear-at-2026-08-21` beta 标头，支持的模型和平台与对话中途的系统消息相同。
</Note>

## 对话中途的工具变更

`tools` 数组在经过哈希处理的请求前缀中的位置甚至比顶层 `system` 字段更靠前，因此编辑它会使整个对话的[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)失效。对话中途的工具变更是对话中途系统消息在工具方面的对应功能。您无需在对话的整个生命周期内固定工具列表，而是可以在轮次之间更改向模型提供哪些工具：预先在 `tools` 中声明完整的工具集，然后使用 `tool_addition` 和 `tool_removal` 块从对话中的某个特定位置起向模型提供某个工具或将其撤回。`tools` 数组本身永远不会改变，因此缓存前缀保持完整。

`tool_addition` 和 `tool_removal` 是 `role: "system"` 消息的 `content` 数组中的内容块，它们可以与同一消息中的 `text` 块混合使用。该消息遵循与任何对话中途系统消息相同的放置规则（请参阅[限制](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)），并且变更从对话中的该位置起生效。每个块的 `tool` 字段引用一个工具而不是定义一个工具：`{"type": "tool_reference", "name": "..."}` 指定在请求的 `tools` 数组中声明的工具，而 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)工具可以通过 `mcp_tool_reference`（`server_name` 和 `name`）单独引用，或通过 `mcp_toolset_reference`（`server_name`）作为整个工具集引用。引用未在 `tools` 中声明的名称会返回 400 错误。

在 `tools` 中声明的每个工具从对话开始时就会提供给模型，除非它是以 `defer_loading: true` 声明的，这会使其保持隐藏状态，直到某个 `tool_addition` 块将其显现出来。`tool_addition` 也可以重新提供先前被 `tool_removal` 撤回的工具。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: mid-conversation-tool-changes-2026-07-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "tools": [
        {
          "name": "get_weather",
          "description": "Get the current weather for a location.",
          "input_schema": {
            "type": "object",
            "properties": {
              "location": {"type": "string", "description": "City name"}
            },
            "required": ["location"]
          }
        }
      ],
      "messages": [
        {
          "role": "user",
          "content": "Say OK."
        },
        {
          "role": "system",
          "content": [
            {
              "type": "tool_removal",
              "tool": {"type": "tool_reference", "name": "get_weather"}
            }
          ]
        }
      ]
    }'
  ```

  ```bash CLI
  ant beta:messages create --beta mid-conversation-tool-changes-2026-07-01 \
    --transform 'content.#(type=="text").text' --raw-output <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  tools:
    - name: get_weather
      description: Get the current weather for a location.
      input_schema:
        type: object
        properties:
          location:
            type: string
            description: City name
        required:
          - location
  messages:
    - role: user
      content: Say OK.
    - role: system
      content:
        - type: tool_removal
          tool:
            type: tool_reference
            name: get_weather
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      betas=["mid-conversation-tool-changes-2026-07-01"],
      # 完整的工具集在一开始就已声明且永不更改，因此
      # 缓存的前缀保持完整。
      tools=[
          {
              "name": "get_weather",
              "description": "Get the current weather for a location.",
              "input_schema": {
                  "type": "object",
                  "properties": {
                      "location": {"type": "string", "description": "City name"},
                  },
                  "required": ["location"],
              },
          },
      ],
      messages=[
          {
              "role": "user",
              "content": "Say OK.",
          },
          # 从此处起撤回 get_weather。该块通过名称引用
          # 该工具而非编辑 `tools`，因此先前的轮次保持
          # 字节级一致，缓存仍然命中。
          {
              "role": "system",
              "content": [
                  {
                      "type": "tool_removal",
                      "tool": {"type": "tool_reference", "name": "get_weather"},
                  },
              ],
          },
      ],
  )

  for block in response.content:
      if block.type == "text":
          print(block.text)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    betas: ["mid-conversation-tool-changes-2026-07-01"],
    // 完整的工具集在一开始就已声明且永不更改，因此
    // 缓存的前缀保持完整。
    tools: [
      {
        name: "get_weather",
        description: "Get the current weather for a location.",
        input_schema: {
          type: "object",
          properties: {
            location: {
              type: "string",
              description: "City name"
            }
          },
          required: ["location"]
        }
      }
    ],
    messages: [
      { role: "user", content: "Say OK." },
      // 从此处起撤回 get_weather。该块通过名称引用
      // 该工具，而不是编辑 `tools`，因此之前的轮次保持
      // 字节级一致，缓存仍然命中。
      {
        role: "system",
        content: [
          {
            type: "tool_removal",
            tool: { type: "tool_reference", name: "get_weather" }
          }
        ]
      }
    ]
  });

  for (const block of response.content) {
    if (block.type === "text") {
      console.log(block.text);
    }
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var response = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5,
      MaxTokens = 1024,
      Betas = ["mid-conversation-tool-changes-2026-07-01"],
      // 完整的工具集在一开始就声明且永不更改，因此
      // 缓存的前缀保持完整。
      Tools =
      [
          new BetaTool
          {
              Name = "get_weather",
              Description = "Get the current weather for a location.",
              InputSchema = new InputSchema
              {
                  Properties = new Dictionary<string, JsonElement>
                  {
                      ["location"] = JsonSerializer.SerializeToElement(new { type = "string", description = "City name" }),
                  },
                  Required = ["location"],
              },
          },
      ],
      Messages =
      [
          new() { Role = Role.User, Content = "Say OK." },
          // 从此处起撤回 get_weather。该块通过名称引用
          // 该工具而非编辑 `Tools`，因此之前的轮次保持
          // 字节级一致，缓存仍然命中。
          new()
          {
              Role = Role.System,
              Content = new(
              [
                  new BetaRequestToolRemovalBlock
                  {
                      Tool = new BetaToolChangeToolReference { Name = "get_weather" },
                  },
              ]),
          },
      ],
  });

  foreach (var block in response.Content)
  {
      if (block.TryPickText(out var text))
      {
          Console.WriteLine(text.Text);
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Betas:     []anthropic.AnthropicBeta{"mid-conversation-tool-changes-2026-07-01"},
  	// 完整的工具集在一开始就已声明且永不更改，因此
  	// 已缓存的前缀保持完整。
  	Tools: []anthropic.BetaToolUnionParam{
  		{OfTool: &anthropic.BetaToolParam{
  			Name:        "get_weather",
  			Description: anthropic.String("Get the current weather for a location."),
  			InputSchema: anthropic.BetaToolInputSchemaParam{
  				Properties: map[string]any{
  					"location": map[string]any{
  						"type":        "string",
  						"description": "City name",
  					},
  				},
  				Required: []string{"location"},
  			},
  		}},
  	},
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Say OK.")),
  		// 从此处起撤回 get_weather。该块通过名称引用
  		// 该工具而非编辑 Tools，因此先前的轮次保持
  		// 字节级一致，缓存仍然命中。
  		{
  			Role: anthropic.BetaMessageParamRoleSystem,
  			Content: []anthropic.BetaContentBlockParamUnion{
  				anthropic.NewBetaToolRemovalBlock(anthropic.BetaToolChangeToolReferenceParam{
  					Name: "get_weather",
  				}),
  			},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  for _, block := range response.Content {
  	if textBlock, ok := block.AsAny().(anthropic.BetaTextBlock); ok {
  		fmt.Println(textBlock.Text)
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.beta.messages.BetaContentBlockParam;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaMessageParam;
  import com.anthropic.models.beta.messages.BetaRequestToolRemovalBlock;
  import com.anthropic.models.beta.messages.BetaTool;
  import com.anthropic.models.beta.messages.MessageCreateParams;
  // ...
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      // 完整的工具集在一开始就已声明且永不更改，因此
      // 缓存的前缀保持完整。
      BetaTool weatherTool = BetaTool.builder()
          .name("get_weather")
          .description("Get the current weather for a location.")
          .inputSchema(BetaTool.InputSchema.builder()
              .properties(BetaTool.InputSchema.Properties.builder()
                  .putAdditionalProperty("location", JsonValue.from(Map.of(
                      "type", "string",
                      "description", "City name")))
                  .build())
              .addRequired("location")
              .build())
          .build();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addBeta("mid-conversation-tool-changes-2026-07-01")
          .addTool(weatherTool)
          .addUserMessage("Say OK.")
          // 从此处起撤回 get_weather。该块通过名称引用
          // 该工具而非编辑 `tools`，因此之前的轮次保持
          // 字节级一致，缓存仍然命中。
          .addMessage(BetaMessageParam.builder()
              .role(BetaMessageParam.Role.SYSTEM)
              .contentOfBetaContentBlockParams(List.of(
                  BetaContentBlockParam.ofToolRemoval(BetaRequestToolRemovalBlock.builder()
                      .referenceTool("get_weather")
                      .build())))
              .build())
          .build();

      BetaMessage response = client.beta().messages().create(params);
      response.content().stream()
          .flatMap(block -> block.text().stream())
          .forEach(textBlock -> IO.println(textBlock.text()));
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      betas: ['mid-conversation-tool-changes-2026-07-01'],
      // 完整的工具集在一开始就已声明且永不更改，因此
      // 缓存的前缀保持完整。
      tools: [
          [
              'name' => 'get_weather',
              'description' => 'Get the current weather for a location.',
              'input_schema' => [
                  'type' => 'object',
                  'properties' => [
                      'location' => [
                          'type' => 'string',
                          'description' => 'City name',
                      ],
                  ],
                  'required' => ['location'],
              ],
          ],
      ],
      messages: [
          ['role' => 'user', 'content' => 'Say OK.'],
          // 从此处起撤回 get_weather。该块通过名称引用
          // 该工具，而不是编辑 `tools`，因此之前的轮次保持
          // 字节级一致，缓存仍然命中。
          [
              'role' => 'system',
              'content' => [
                  [
                      'type' => 'tool_removal',
                      'tool' => ['type' => 'tool_reference', 'name' => 'get_weather'],
                  ],
              ],
          ],
      ],
  );

  foreach ($response->content as $block) {
      if ($block->type === 'text') {
          echo $block->text, PHP_EOL;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    betas: ["mid-conversation-tool-changes-2026-07-01"],
    # 完整的工具集在一开始就已声明且永不更改，因此
    # 缓存的前缀保持完整。
    tools: [
      {
        name: "get_weather",
        description: "Get the current weather for a location.",
        input_schema: {
          type: "object",
          properties: {
            location: { type: "string", description: "City name" }
          },
          required: ["location"]
        }
      }
    ],
    messages: [
      { role: "user", content: "Say OK." },
      # 从此处起撤回 get_weather。该块通过名称引用
      # 该工具，而不是编辑 `tools`，因此之前的轮次保持
      # 字节级一致，缓存仍然命中。
      {
        role: "system",
        content: [
          {
            type: "tool_removal",
            tool: { type: "tool_reference", name: "get_weather" }
          }
        ]
      }
    ]
  )

  response.content.each do |block|
    puts block.text if block.type == :text
  end
  ```
</CodeGroup>

对话中途的工具变更处于 beta 阶段。要使用它们，请在您的请求中包含 beta 标头 `mid-conversation-tool-changes-2026-07-01`。

## 何时使用对话中途的系统消息

[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)按顺序对请求前缀进行哈希处理：先是 `tools`，然后是 `system`，然后是 `messages`。缓存命中要求前缀与最近的某个请求逐字节完全匹配，直到缓存断点为止。

这种顺序意味着顶层 `system` 字段位于哈希前缀的最开头附近。对它的任何更改，哪怕只是追加一句话，都会产生不同的哈希值，请求将无法命中系统提示及其后每条已缓存消息的缓存。

对话中途的系统消息让您可以改为在消息历史的**末尾**添加指令。新指令之前的所有内容都保持不变，因此现有的缓存条目仍然匹配，只有新消息会作为新输入进行处理。

以下是几种这一点很重要的情况：

* **会话中途的策略或角色变更。** 一个长时间运行的智能体会话在数十个已缓存的轮次之后需要一个新的约束（"从现在开始，将所有 SQL 写成参数化查询"）。将其添加到顶层 `system` 字段会重新处理整个历史记录。
* **必须具有权威性的每轮上下文。** 您希望以系统级别的权重注入一条时效性说明、会话截止时间或工具可用性变更，而它变化过于频繁，不适合放在缓存前缀中。
* **不应堆积的每轮提醒。** 一个运行框架在每批工具结果之后提醒模型（"将相互独立的读取请求一起发出"、"用户已经有一段时间没有收到您的消息了"），并希望模型只看到最新的一份。[轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)只在一个轮次中渲染，之后不产生任何成本，且无需从历史记录中删除任何内容。
* **您的应用程序观察到的状态变化。** 您的应用程序注意到一些 Claude 应当视为运营方级别事实的情况：磁盘上的文件发生了变化、用户切换了自动批准设置、可用工具发生了变化，或者剩余的令牌预算降到了某个阈值以下。
* **不应打断智能体循环的用户输入。** 当 Claude 仍在为上一个请求执行工具时，用户输入了一条后续消息。在下一个工具结果之后将其作为系统消息转达，可以让 Claude 将新输入融入它正在进行的工作中，而不是将其视为需要切换过去的全新请求。请参阅[放置在工具结果之后](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#placement-after-tool-results)。
* **授予持续权限的模式切换。** 会话级别的模式可以使用对话中途的系统消息来授予对某项昂贵功能的持续许可，例如自动启动多智能体工作流，并每隔几个轮次附上简短的提醒，在模式关闭时发出退出通知。有关完整示例，请参阅[构建编排模式](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-effort-example)。

在所有这些情况下，您都可以将指令放在常规的 `user` 消息中，Claude 确实会遵循在用户轮次中到达的指令。区别在于优先级：`user` 消息被视为来自最终用户，而 `system` 消息被视为来自您，即应用程序运营方。当两者冲突时，系统指令优先，因此对于即使最终用户提出不同要求也应当成立的运营方级别事实和约束，请使用 `system` 角色。对话中途的系统消息保留了这种运营方级别的优先级，而无需承担编辑顶层 `system` 字段所带来的缓存未命中成本。

## 工作原理

向 `messages` 数组添加一条 `"role": "system"` 的消息。`content` 可以使用纯字符串或内容块，与 `user` 或 `assistant` 轮次相同。该指令从对话中的该位置起生效。当指令冲突时，较晚的系统消息优先于较早的系统消息，并且对于其后的轮次，对话中途的系统消息优先于顶层 `system` 字段。

对于应当适用于整个对话的指令，您仍然可以设置顶层 `system` 字段。请将对话中途的系统消息保留给那些只在稍后才变得相关的指令，或者您希望在不使缓存前缀失效的情况下添加的指令。

`role: "system"` 消息还可以携带 `output_config.effort`，以便从下一个 `user` 轮次起更改 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)（努力程度）级别。此功能在 Claude API 上的 Claude Fable 5.1、Claude Mythos 5.1 和 Claude Opus 5 上处于 beta 阶段，需要 `mid-conversation-output-config-2026-07-01` beta 标头。请参阅[每条消息的努力程度](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "cache_control": {"type": "ephemeral"},
      "system": "You are a code review assistant. Be concise.",
      "messages": [
        {
          "role": "user",
          "content": "Review process() in utils.py for performance issues."
        },
        {
          "role": "assistant",
          "content": "The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list."
        },
        {
          "role": "user",
          "content": "Now review the calling code that invokes process()."
        },
        {
          "role": "system",
          "content": "From now on, every suggestion must include explicit type annotations."
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages create --transform 'content.#(type=="text").text' --raw-output <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  cache_control:
    type: ephemeral
  system: You are a code review assistant. Be concise.
  messages:
    - role: user
      content: Review process() in utils.py for performance issues.
    - role: assistant
      content: >-
        The list comprehension is fine for small inputs. For large inputs,
        consider a generator to avoid materializing the full list.
    - role: user
      content: Now review the calling code that invokes process().
    - role: system
      content: From now on, every suggestion must include explicit type annotations.
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      # 自动提示缓存：每个请求都会缓存到目前为止的对话，
      # 下一个请求则从缓存中读取未更改的前缀。
      cache_control={"type": "ephemeral"},
      system="You are a code review assistant. Be concise.",
      messages=[
          {
              "role": "user",
              "content": "Review process() in utils.py for performance issues.",
          },
          {
              "role": "assistant",
              "content": "The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list.",
          },
          {
              "role": "user",
              "content": "Now review the calling code that invokes process().",
          },
          # 审阅者在会话中途意识到，所有建议还必须
          # 符合团队严格的类型策略。在此处追加
          # 指令可使之前的轮次保持字节级一致，因此
          # 上一个请求缓存的前缀仍可从缓存中读取。
          {
              "role": "system",
              "content": "From now on, every suggestion must include explicit type annotations.",
          },
      ],
  )

  for block in response.content:
      if block.type == "text":
          print(block.text)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    // 自动提示缓存：每个请求都会缓存到目前为止的对话，
    // 下一个请求则从缓存中读取未更改的前缀。
    cache_control: { type: "ephemeral" },
    system: "You are a code review assistant. Be concise.",
    messages: [
      {
        role: "user",
        content: "Review process() in utils.py for performance issues."
      },
      {
        role: "assistant",
        content:
          "The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list."
      },
      {
        role: "user",
        content: "Now review the calling code that invokes process()."
      },
      // 审阅者在会话中途意识到，所有建议还必须符合
      // 团队严格的类型策略。在此处追加指令可使
      // 先前的轮次保持字节级一致，因此上一个请求
      // 缓存的前缀仍可从缓存中读取。
      {
        role: "system",
        content: "From now on, every suggestion must include explicit type annotations."
      }
    ]
  });

  const textBlock = response.content.find(
    (block): block is Anthropic.TextBlock => block.type === "text"
  );
  console.log(textBlock?.text);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      // 自动提示缓存：每个请求都会缓存到目前为止的对话，
      // 下一个请求则从缓存中读取未更改的前缀。
      CacheControl = new CacheControlEphemeral(),
      System = "You are a code review assistant. Be concise.",
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = "Review process() in utils.py for performance issues."
          },
          new()
          {
              Role = Role.Assistant,
              Content = "The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list."
          },
          new()
          {
              Role = Role.User,
              Content = "Now review the calling code that invokes process()."
          },
          // 审查者在会话中途意识到，所有建议还必须符合
          // 团队的严格类型策略。在此处追加指令可使
          // 先前的轮次保持字节级一致，因此上一个请求
          // 缓存的前缀仍可从缓存中读取。
          new()
          {
              Role = Role.System,
              Content = "From now on, every suggestion must include explicit type annotations."
          }
      ]
  };

  var response = await client.Messages.Create(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	// 自动提示缓存：每个请求都会缓存到目前为止的对话，
  	// 下一个请求会从缓存中读取未更改的前缀。
  	CacheControl: anthropic.NewCacheControlEphemeralParam(),
  	System: []anthropic.TextBlockParam{
  		{Text: "You are a code review assistant. Be concise."},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Review process() in utils.py for performance issues.")),
  		anthropic.NewAssistantMessage(anthropic.NewTextBlock("The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list.")),
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Now review the calling code that invokes process().")),
  		// 审阅者在会话中途意识到，所有建议还必须
  		// 符合团队严格的类型策略。在此处追加指令
  		// 可使之前的轮次保持字节级一致，因此上一个请求
  		// 缓存的前缀仍可从缓存中读取。
  		{
  			Role: anthropic.MessageParamRoleSystem,
  			Content: []anthropic.ContentBlockParamUnion{
  				anthropic.NewTextBlock("From now on, every suggestion must include explicit type annotations."),
  			},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  for _, block := range response.Content {
  	if textBlock, ok := block.AsAny().(anthropic.TextBlock); ok {
  		fmt.Println(textBlock.Text)
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.messages.CacheControlEphemeral;
  // ...
  import com.anthropic.models.messages.MessageParam;
  // ...
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          // 自动提示缓存：每个请求都会缓存到目前为止的对话，
          // 下一个请求则从缓存中读取未更改的前缀。
          .cacheControl(CacheControlEphemeral.builder().build())
          .system("You are a code review assistant. Be concise.")
          .addUserMessage("Review process() in utils.py for performance issues.")
          .addAssistantMessage("The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list.")
          .addUserMessage("Now review the calling code that invokes process().")
          // 审阅者在会话中途意识到，所有建议还必须符合
          // 团队严格的类型策略。在此处追加指令可使
          // 先前的轮次保持字节级一致，因此上一个请求
          // 缓存的前缀仍可从缓存中读取。
          .addMessage(MessageParam.builder()
              .role(MessageParam.Role.SYSTEM)
              .content("From now on, every suggestion must include explicit type annotations.")
              .build())
          .build();

      Message response = client.messages().create(params);
      response.content().stream()
          .flatMap(block -> block.text().stream())
          .forEach(textBlock -> IO.println(textBlock.text()));
  ```

  ```php PHP
  use Anthropic\Messages\CacheControlEphemeral;
  // ...
  $client = new Client();

  $response = $client->messages->create(
      maxTokens: 1024,
      messages: [
          ['role' => 'user', 'content' => 'Review process() in utils.py for performance issues.'],
          ['role' => 'assistant', 'content' => 'The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list.'],
          ['role' => 'user', 'content' => 'Now review the calling code that invokes process().'],
          // 审查者在会话中途意识到，所有建议还必须符合
          // 团队严格的类型策略。在此处追加指令可使
          // 先前的轮次保持字节级一致，因此上一个请求
          // 缓存的前缀仍可从缓存中读取。
          ['role' => 'system', 'content' => 'From now on, every suggestion must include explicit type annotations.']
      ],
      model: 'claude-opus-5',
      // 自动提示缓存：每个请求都会缓存到目前为止的对话，
      // 下一个请求则从缓存中读取未更改的前缀。
      cacheControl: CacheControlEphemeral::with(),
      system: 'You are a code review assistant. Be concise.',
  );

  foreach ($response->content as $block) {
      if ($block->type === 'text') {
          echo $block->text, PHP_EOL;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    # 自动提示缓存：每个请求都会缓存到目前为止的对话，
    # 下一个请求会从缓存中读取未更改的前缀。
    cache_control: { type: "ephemeral" },
    system: "You are a code review assistant. Be concise.",
    messages: [
      { role: "user", content: "Review process() in utils.py for performance issues." },
      { role: "assistant", content: "The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list." },
      { role: "user", content: "Now review the calling code that invokes process()." },
      # 审阅者在会话中途意识到所有建议还必须符合
      # 团队的严格类型策略。在此处追加指令可使
      # 先前的轮次保持字节级一致，因此上一个请求
      # 缓存的前缀仍可从缓存中读取。
      { role: "system", content: "From now on, every suggestion must include explicit type annotations." }
    ]
  )

  response.content.each do |block|
    puts block.text if block.type == :text
  end
  ```
</CodeGroup>

此示例通过顶层 `cache_control` 字段启用了[自动缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#automatic-caching)。提示缓存是选择性启用的：如果请求没有 `cache_control` 字段（无论是自动缓存还是[显式断点](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#explicit-cache-breakpoints)），则不会缓存任何内容，每个请求都要为整个对话支付常规的输入令牌价格。启用缓存后，追加系统消息会使已缓存的轮次保持不变，因此携带新指令的请求仍然从缓存中读取它们，而不是重新处理。缓存还要求对话满足[最小可缓存提示长度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#cache-limitations)；像本例这样简短的示例低于该长度，因此在对话增长之前，`cache_creation_input_tokens` 和 `cache_read_input_tokens` 将保持为 0。

对话中途的系统消息必须紧跟在 `user` 轮次（或以服务器工具结果结尾的 `assistant` 轮次）之后，并且必须是 `messages` 中的最后一个条目，或者紧接着一个 `assistant` 轮次。携带 `tool_result` 块的 `user` 消息也算在内：在智能体循环中，您可以将系统消息放在工具结果之后、Claude 的下一个轮次之前。任何其他位置，包括在 `assistant` 的 `tool_use` 块与回应它的 `tool_result` 之间，都会返回 400 错误。

### 放置在工具结果之后

在[智能体循环](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)中，系统消息放在传递工具结果的 `user` 消息之后。这也是您的应用程序可以转达用户在 Claude 工作期间输入的内容的位置，这样新的上下文就会被吸收，而无需重新开始该轮次：

```json
[
  { "role": "user", "content": "Run the test suite and fix any failures." },
  {
    "role": "assistant",
    "content": [{ "type": "tool_use", "id": "toolu_01", "name": "run_tests", "input": {} }]
  },
  {
    "role": "user",
    "content": [
      { "type": "tool_result", "tool_use_id": "toolu_01", "content": "12 passed, 0 failed" }
    ]
  },
  {
    "role": "system",
    "content": "The user sent the following message while you were working: also update the changelog before you finish."
  }
]
```

请将系统内容表述为上下文，而不是覆盖用户的命令。陈述事实（"收到来自用户的新输入：X"、"剩余令牌预算现在为 Y"），让 Claude 据此行动。Claude 经过训练会抵制那些看起来与用户利益相悖的指令，这种保护同样适用于系统角色，因此诸如"忽略用户所说的话"之类的措辞不如陈述发生了什么变化有效。

此模式用于转达来自对话自身最终用户的输入。请勿使用它来传递工具输出、检索到的文档或其他第三方内容；请将这些内容保留在 `tool_result` 块中（请参阅[限制](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)）。

### 轮次作用域的系统消息

要将 `role: "system"` 消息的作用域限定在当前轮次，请设置其 `clear_at` 字段。它接受以下两个值之一：

* `"never"`（默认值）：该消息在每个包含它的请求中都会在其位置渲染。省略该字段效果相同。
* `"next_user_message"`：该消息是**轮次作用域的**。只有当 `messages` 中它之后没有 `role: "user"` 消息时，其文本才会渲染。在这里，仅携带 `tool_result` 块的用户消息也算作用户消息。一旦存在较晚的用户消息，该消息就会被**清除**：它保留在数组中，但不渲染任何内容，也不消耗输入令牌，在该请求及之后的每个请求中都是如此。

轮次作用域的系统消息处于 beta 阶段。请包含 [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers) `mid-conversation-system-clear-at-2026-08-21`。如果没有它，`clear_at` 会作为未知字段被拒绝。

```json
{
  "role": "system",
  "clear_at": "next_user_message",
  "content": "First privately list what you need next; then request every item that doesn't depend on another's result in this one response."
}
```

主要用途是工具循环中的每轮提醒。每次您希望模型看到提醒时，将其追加在 `tool_result` 消息之后，并将之前的每一份副本保留在原处。模型只会看到最后一条用户消息之后的副本，因此提醒永远不会堆积。`messages` 中较早的内容没有任何变化，因此[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)会持续匹配。在 Claude Fable 5.1 上，这还能使后续的[思考块保持有效](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)：删除较早的提醒会改变这些块之前的对话并导致对话检查失败，而被清除的消息保留在数组中，使该对话保持不变。

以下请求是智能体循环的较晚一步。`messages[3]` 在较早的请求中渲染过，当时它是数组中的最后一条消息。一旦 `messages[5]`（一条较晚的用户消息）存在，`messages[3]` 就会被清除：被清除的消息保留在数组中，因此 `messages[4]` 中思考块之前的对话保持不变，但模型不再看到其文本。`messages[6]` 和 `messages[7]` 都会按顺序渲染。

```json
{
  "model": "claude-fable-5-1",
  "max_tokens": 16000,
  "messages": [
    { "role": "user", "content": "Fix the failing test." },
    {
      "role": "assistant",
      "content": [
        { "type": "thinking", "thinking": "", "signature": "..." },
        {
          "type": "tool_use",
          "id": "toolu_01",
          "name": "read_file",
          "input": { "path": "test_auth.py" }
        }
      ]
    },
    {
      "role": "user",
      "content": [{ "type": "tool_result", "tool_use_id": "toolu_01", "content": "..." }]
    },
    {
      "role": "system",
      "clear_at": "next_user_message",
      "content": "Request independent reads in one turn."
    },
    {
      "role": "assistant",
      "content": [
        { "type": "thinking", "thinking": "", "signature": "..." },
        {
          "type": "tool_use",
          "id": "toolu_02",
          "name": "read_file",
          "input": { "path": "auth.py" }
        },
        {
          "type": "tool_use",
          "id": "toolu_03",
          "name": "read_file",
          "input": { "path": "tokens.py" }
        }
      ]
    },
    {
      "role": "user",
      "content": [
        { "type": "tool_result", "tool_use_id": "toolu_02", "content": "..." },
        {
          "type": "tool_result",
          "tool_use_id": "toolu_03",
          "content": "...",
          "cache_control": { "type": "ephemeral" }
        }
      ]
    },
    {
      "role": "system",
      "clear_at": "next_user_message",
      "content": "Request independent reads in one turn."
    },
    {
      "role": "system",
      "clear_at": "next_user_message",
      "content": "The shell exited with status 137."
    }
  ]
}
```

轮次作用域消息的规则：

* **逐字重新发送被清除的消息。** 被清除的消息仍然是对话历史的一部分。根据当前状态重建它（新的令牌计数、时间戳）、将其作为冗余内容丢弃，或更改其 `clear_at` 值，都属于对较早消息的编辑。提示缓存会从该位置起未命中，并且在 Claude Fable 5.1 上，在它之后生成的每个思考块都会无法通过[对话检查](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)。
* **仅限文本。** `content` 是一个或多个 `text` 块（或一个字符串）。`tool_addition` 和 `tool_removal` 块在轮次作用域消息上会返回 400 错误，`output_config` 也是如此。对于这些内容，请使用不带 `clear_at` 的单独 `role: "system"` 消息。
* **其块上不能有 `cache_control`。** 被清除的消息永远不会成为缓存键的一部分，因此其上的断点永远无法匹配。请像示例中那样，将断点放在前一个用户轮次的最后一个块上。顶层[自动缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#automatic-caching)字段在选择断点时会跳过轮次作用域消息。在清除某条消息的请求中，可重用的缓存前缀在该消息之前的用户轮次处结束，因此只有该消息与新用户消息之间的那一个助手轮次会被重新处理。
* **放置规则仍然适用**，无论是否被清除。轮次作用域消息必须跟在 `user` 轮次（或以服务器工具结果结尾的 `assistant` 轮次）之后，并位于 `assistant` 轮次之前或位于数组末尾，与任何对话中途的系统消息一样。位于数组末尾的消息始终会渲染。紧接着另一条 `user` 消息的消息会返回 400 错误，而不是成为被清除的消息：请将一轮工具调用的所有结果放在一条用户消息中，并将提醒放在其后。
* **助手轮次不会清除它。** 该消息之后的预填充或[暂停](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons#pause-turn)的助手轮次，或服务器端工具循环，都不会添加用户消息，因此该消息在该延续中仍会渲染。要在客户端工具循环中保持提醒可见，请在每条 `tool_result` 消息之后再次追加它。
* **令牌计数遵循实际渲染的内容。** 被清除的消息不会增加 `usage.input_tokens`，也不会增加[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)。
* **导入的历史记录。** 在您一步构建的对话记录中（少样本示例、迁移的对话），如果某条轮次作用域消息之后已经有一个助手轮次和一条用户消息，那么它从第一个请求起就被清除，永远不会渲染。对于您要延续的每轮提醒来说，这是正确的状态。只有在模型应当在每个请求中都看到的消息上才不设置 `clear_at`。

验证错误如下：

```text wrap
messages.3.clear_at: Extra inputs are not permitted
messages.3.clear_at: clear_at is only permitted on role 'system' messages
messages.3.clear_at: Input should be 'next_user_message' or 'never'
messages.3: a turn-scoped system message supports text blocks only (clear_at: 'next_user_message')
messages.3: output_config is not permitted on a turn-scoped system message (clear_at: 'next_user_message')
messages.3.content.0: cache_control is not permitted on a turn-scoped system message (clear_at: 'next_user_message')
```

第一个是没有 beta 标头时返回的错误。在 Amazon Bedrock 和 Google Cloud 上，请按照 [Beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)中的说明传递 beta 值。

通过 SDK 使用时，请在 `messages` 中的 `role: "system"` 条目上设置 `clear_at` 并发送 beta 标头。以下示例在用户轮次之后追加一条轮次作用域的提醒；在下一个请求中，一旦存在较晚的用户消息，该提醒会保留在数组中但不再渲染：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: mid-conversation-system-clear-at-2026-08-21" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-fable-5-1",
      "max_tokens": 4096,
      "messages": [
        {"role": "user", "content": "Draft a short status update on the database migration for the team channel."},
        {"role": "system", "clear_at": "next_user_message", "content": "The reader is on call: keep this reply under 50 words."}
      ]
    }'
  ```

  ```bash CLI
  ant beta:messages create --beta mid-conversation-system-clear-at-2026-08-21 \
    --transform 'content.#(type=="text").text' --raw-output <<'YAML'
  model: claude-fable-5-1
  max_tokens: 4096
  messages:
    - role: user
      content: Draft a short status update on the database migration for the team channel.
    # 回合级提醒：在本回合渲染，一旦出现后续用户消息即清除。
    - role: system
      clear_at: next_user_message
      content: "The reader is on call: keep this reply under 50 words."
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.beta.messages.create(
      model="claude-fable-5-1",
      max_tokens=4096,
      messages=[
          {
              "role": "user",
              "content": "Draft a short status update on the database migration for the team channel.",
          },
          # 回合级提醒：在本回合渲染，一旦出现后续用户消息即清除。
          {
              "role": "system",
              "clear_at": "next_user_message",
              "content": "The reader is on call: keep this reply under 50 words.",
          },
      ],
      betas=["mid-conversation-system-clear-at-2026-08-21"],
  )

  for block in response.content:
      if block.type == "text":
          print(block.text)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-fable-5-1",
    max_tokens: 4096,
    messages: [
      {
        role: "user",
        content: "Draft a short status update on the database migration for the team channel."
      },
      // 回合范围的提醒：在本回合渲染，一旦存在后续用户消息即清除。
      {
        role: "system",
        clear_at: "next_user_message",
        content: "The reader is on call: keep this reply under 50 words."
      }
    ],
    betas: ["mid-conversation-system-clear-at-2026-08-21"]
  });

  for (const block of response.content) {
    if (block.type === "text") {
      console.log(block.text);
    }
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;

  AnthropicClient client = new();

  var response = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = "claude-fable-5-1",
      MaxTokens = 4096,
      Messages =
      [
          new() { Role = Role.User, Content = "Draft a short status update on the database migration for the team channel." },
          // 回合级提醒：在本回合渲染，一旦存在后续用户消息即清除。
          new()
          {
              Role = Role.System,
              ClearAt = ClearAt.NextUserMessage,
              Content = "The reader is on call: keep this reply under 50 words.",
          },
      ],
      Betas = [AnthropicBeta.MidConversationSystemClearAt2026_08_21],
  });

  foreach (var block in response.Content)
  {
      if (block.TryPickText(out var textBlock))
      {
          Console.WriteLine(textBlock.Text);
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.Background(), anthropic.BetaMessageNewParams{
  	Model:     "claude-fable-5-1",
  	MaxTokens: 4096,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Draft a short status update on the database migration for the team channel.")),
  		// 回合级提醒：在本回合渲染，一旦出现后续用户消息即清除。
  		{
  			Role:    anthropic.BetaMessageParamRoleSystem,
  			ClearAt: anthropic.BetaMessageParamClearAtNextUserMessage,
  			Content: []anthropic.BetaContentBlockParamUnion{anthropic.NewBetaTextBlock("The reader is on call: keep this reply under 50 words.")},
  		},
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaMidConversationSystemClearAt2026_08_21},
  })
  if err != nil {
  	log.Fatal(err)
  }

  for _, block := range response.Content {
  	if textBlock, ok := block.AsAny().(anthropic.BetaTextBlock); ok {
  		fmt.Println(textBlock.Text)
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaMessageParam;
  import com.anthropic.models.beta.messages.MessageCreateParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model("claude-fable-5-1")
          .maxTokens(4096L)
          .addBeta(AnthropicBeta.MID_CONVERSATION_SYSTEM_CLEAR_AT_2026_08_21)
          .addUserMessage("Draft a short status update on the database migration for the team channel.")
          // 回合级提醒：在本回合渲染，一旦出现后续用户消息即清除。
          .addMessage(BetaMessageParam.builder()
              .role(BetaMessageParam.Role.SYSTEM)
              .clearAt(BetaMessageParam.ClearAt.NEXT_USER_MESSAGE)
              .content("The reader is on call: keep this reply under 50 words.")
              .build())
          .build();

      BetaMessage response = client.beta().messages().create(params);
      response.content().stream()
          .flatMap(block -> block.text().stream())
          .forEach(textBlock -> IO.println(textBlock.text()));
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaMessageParam;
  use Anthropic\Client;

  $client = new Client();

  $response = $client->beta->messages->create(
      model: 'claude-fable-5-1',
      maxTokens: 4096,
      messages: [
          BetaMessageParam::with(role: 'user', content: 'Draft a short status update on the database migration for the team channel.'),
          // 回合范围的提醒：在本回合渲染，一旦存在后续用户消息即清除。
          BetaMessageParam::with(
              role: 'system',
              clearAt: 'next_user_message',
              content: 'The reader is on call: keep this reply under 50 words.',
          ),
      ],
      betas: [AnthropicBeta::MID_CONVERSATION_SYSTEM_CLEAR_AT_2026_08_21],
  );

  foreach ($response->content as $block) {
      if ($block->type === 'text') {
          echo $block->text, PHP_EOL;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-fable-5-1",
    max_tokens: 4096,
    messages: [
      {role: "user", content: "Draft a short status update on the database migration for the team channel."},
      # 回合级提醒：在本回合渲染，一旦存在后续用户消息即清除。
      {role: "system", clear_at: :next_user_message, content: "The reader is on call: keep this reply under 50 words."}
    ],
    betas: [Anthropic::AnthropicBeta::MID_CONVERSATION_SYSTEM_CLEAR_AT_2026_08_21]
  )

  response.content.each do |block|
    puts block.text if block.type == :text
  end
  ```
</CodeGroup>

## 与提示缓存结合使用

对话中途的系统消息和[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)在设计上就是为了配合使用的：

* **显式启用缓存。** 只有当请求包含 `cache_control` 时才会进行缓存，可以是顶层[自动缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#automatic-caching)字段，也可以是内容块上的[显式断点](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#explicit-cache-breakpoints)。对话中途的系统消息本身不会创建缓存条目，而且如果未启用缓存，就没有可保留的节省。
* **照常缓存稳定前缀。** 将 `cache_control` 放在跨请求保持不变的最后一个块上，无论是顶层 `system` 字段的末尾、工具定义的末尾，还是消息历史中的某个稳定位置。
* **在断点之后追加系统消息。** 由于它位于缓存前缀之后，因此不会改变前缀哈希，缓存仍然命中。
* **对话中途的系统消息本身也是可缓存的。** 一旦它进入对话，就成为稳定历史的一部分。在下一个轮次中，您可以将缓存断点移到它之后（或依靠[自动缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#automatic-caching)来完成），系统消息就会像任何其他轮次一样从缓存中读取。

请避免编辑或删除已经发送的对话中途系统消息。与对较早消息的任何其他更改一样，这会使从该位置起的缓存失效。在 Claude Fable 5.1 上，它还会使之后每个助手轮次中的[思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)失效。对于只应适用于一个轮次的指导，请使用[轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)并将其保留在原处。如果指令需要演变，请追加一条新的系统消息，而不是重写旧的。连续的系统消息是被接受的，并被视为单个系统部分，该部分作为整体遵循相同的放置规则。

## 限制

* **不能作为第一条消息。** 携带内容的 `system` 消息不能是 `messages` 中的第一个条目。对于从一开始就适用的指令，请使用顶层 `system` 字段。
* **放置位置受限。** 携带内容（`text`、`tool_addition` 或 `tool_removal` 块）的 `system` 消息必须紧跟在 `user` 轮次（包括携带 `tool_result` 块的 `user` 轮次）或以服务器工具结果结尾的 `assistant` 轮次之后，并且必须位于 `assistant` 轮次之前或位于数组末尾。它不能位于 `tool_use` 块与其 `tool_result` 之间。放在其他位置会返回 400 错误。`content` 为空且仅设置 [`output_config.effort`](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta) 的消息在其位置不渲染任何内容，可以放在 `messages` 中的任何位置，包括第一个位置或 `assistant` 轮次与 `user` 轮次之间。连续的 `system` 消息会被一起判断，因此在仅设置努力程度的消息旁边添加一条携带文本的消息，会使整个组遵循内容规则。
* **轮次作用域消息仅限文本且需逐字重新发送。** `clear_at: "next_user_message"` 消息不携带 `tool_addition`、`tool_removal`、`output_config` 或 `cache_control`，并且一旦被清除，它必须在之后的请求中逐字节保留在 `messages` 中。请参阅[轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)。
* **不是放置不受信任内容的地方。** Claude 将系统内容视为运营方指令并遵循它。请勿将来自对话外部的文本（例如原始工具输出、检索到的文档或网页内容）直接放在系统消息中；这样做会赋予该文本运营方级别的权限。请将这些数据保留在 `tool_result` 块中，并继续遵循[缓解越狱和提示注入](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks)。

## 相关内容

<CardGroup cols={2}>
  <Card title="提示缓存" icon="bolt" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching">
    缓存的工作原理、断点的放置位置，以及如何读取缓存使用量字段。
  </Card>

  <Card title="缓存诊断" icon="magnifying-glass" href="https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics">
    当您预期的缓存命中没有发生时，准确找出两个请求在何处产生了分歧。
  </Card>

  <Card title="使用 Messages API" icon="message" href="https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages">
    消息结构、多轮对话以及 `system` 字段。
  </Card>

  <Card title="提示最佳实践" icon="text" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices">
    编写有效的提示和系统指令。
  </Card>

  <Card title="使用 Claude 进行工具使用" icon="wrench" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview">
    `tool_use` 和 `tool_result` 块在 `messages` 数组中的结构。
  </Card>
</CardGroup>
