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

  此功能适用于 Claude Fable 5.1、[Claude Mythos 5.1](https://anthropic.com/glasswing)、Claude Fable 5、[Claude Mythos 5](https://anthropic.com/glasswing)、Claude Opus 5.5、Claude Opus 4.8 和 Claude Opus 5。对话中途的系统消息不需要 beta 标头。此功能不适用于 Claude Sonnet 5，在该模型上请改用顶层 `system` 字段。

  [对话中途的工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes)在相同的模型上处于 beta 阶段。在 Claude API 上，发送 `inline-tools-2026-09-15` beta 头，该头也涵盖了[在 `tool_addition` 块内定义工具](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#define-tools-in-a-message-beta)。[以这种方式添加 MCP 服务器](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#add-an-mcp-server-mid-conversation-beta)需要第二个 beta 头 `mcp-client-2026-09-15`，该头在 Claude API 上可用。`mid-conversation-tool-changes-2026-07-01` 头适用于通过引用指定工具名称的变更，在 Claude API、Amazon Bedrock 和 Google Cloud 上均可使用。

  [轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)（`clear_at`）处于 beta 阶段，需要 `mid-conversation-system-clear-at-2026-08-21` beta 标头，支持的模型和平台与对话中途的系统消息相同。
</Note>

## 对话中途的工具变更

`tools` 数组在经过哈希的请求前缀中的位置比顶层 `system` 字段还要靠前，因此编辑它会使整个对话的[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)失效。对话中途的工具变更是对话中途系统消息在工具方面的对应功能。您无需在对话的整个生命周期内固定工具列表，而是可以在轮次之间更改向模型提供的工具：预先在 `tools` 中声明完整的工具集，然后使用 `tool_addition` 和 `tool_removal` 块，从对话中的某个特定位置开始向模型提供某个工具或撤回该工具。`tools` 数组本身从不改变，因此缓存前缀保持完整。对话中途的工具变更处于 beta 阶段，在 Claude API 上使用 `inline-tools-2026-09-15` beta 标头。

`tool_addition` 和 `tool_removal` 是 `role: "system"` 消息的 `content` 数组中的内容块，它们可以与同一消息中的 `text` 块混合使用。该消息遵循任何对话中途系统消息的放置规则，在暂停的轮次之后有一个额外的限制（参见[限制](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)），并且该变更从对话中的那个点开始生效。每个块的 `tool` 字段引用一个工具，而不是定义一个工具：`{"type": "tool_reference", "name": "..."}` 指代请求的 `tools` 数组中声明的工具，而 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)工具可以通过 `mcp_tool_reference`（`server_name` 和 `name`）单独引用，也可以通过 `mcp_toolset_reference`（`server_name`）作为整个工具集引用。引用一个未在 `tools` 中声明的名称会返回 400 错误（在 Claude API 上，`error.details.error_code` 设置为 `tool_reference_unresolved`）。`tool_addition` 块也可以[携带工具的完整定义](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#define-tools-in-a-message-beta)，而 `mid-conversation-tool-changes-2026-07-01` 头不支持这种方式。

在 `tools` 中声明的每个工具从对话开始时就会提供给模型，除非它是以 `defer_loading: true` 声明的，这会使其保持隐藏状态，直到某个 `tool_addition` 块将其显现出来。`tool_addition` 也可以重新提供先前被 `tool_removal` 撤回的工具。

以下请求在 `tools` 中声明了 `get_weather`，然后在第一个用户轮次之后使用 `tool_removal` 块将其撤回。该请求发送了 `inline-tools-2026-09-15` beta 标头。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: inline-tools-2026-09-15" \
    -d '{
      "model": "claude-opus-5-5",
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
  ant beta:messages create --beta inline-tools-2026-09-15 \
    --transform 'content.#(type=="text").text' --raw-output <<'YAML'
  model: claude-opus-5-5
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
      model="claude-opus-5-5",
      max_tokens=1024,
      betas=["inline-tools-2026-09-15"],
      # 完整的工具集在开头一次性声明且不再更改，因此
      # 缓存的前缀保持不变。
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
          # 该工具，而不是编辑 `tools`，因此之前的轮次保持
          # 字节级一致，缓存仍能命中。
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
    model: "claude-opus-5-5",
    max_tokens: 1024,
    betas: ["inline-tools-2026-09-15"],
    // 完整的工具集在开头一次性声明且从不更改，因此
    // 缓存的前缀保持不变。
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
      // 从此处起撤回 get_weather。该块通过名称引用该
      // 工具，而非编辑 `tools`，因此之前的轮次保持
      // 字节级一致，缓存仍能命中。
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
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var response = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5_5,
      MaxTokens = 1024,
      Betas = [AnthropicBeta.InlineTools2026_09_15],
      // 完整的工具集在开头一次性声明且不再更改，因此
      // 缓存前缀（cached prefix）保持不变。
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
          // 该工具，而不是编辑 `Tools`，因此之前的轮次保持
          // 字节级一致，缓存仍能命中。
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
  	Model:     anthropic.ModelClaudeOpus5_5,
  	MaxTokens: 1024,
  	Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaInlineTools2026_09_15},
  	// 完整的工具集在开头一次性声明且不再更改，因此
  	// 缓存前缀保持不变。
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
  		// 从此处起撤回 get_weather。该块按名称引用
  		// 该工具，而不是编辑 Tools，因此之前的轮次保持
  		// 字节级一致，缓存仍能命中。
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
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaContentBlockParam;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaMessageParam;
  import com.anthropic.models.beta.messages.BetaRequestToolRemovalBlock;
  import com.anthropic.models.beta.messages.BetaTool;
  import com.anthropic.models.beta.messages.MessageCreateParams;
  // ...
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      // 完整的工具集在开头一次性声明且不再更改，因此
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
          .model(Model.CLAUDE_OPUS_5_5)
          .maxTokens(1024)
          .addBeta(AnthropicBeta.INLINE_TOOLS_2026_09_15)
          .addTool(weatherTool)
          .addUserMessage("Say OK.")
          // 从此处起撤回 get_weather。该块按名称引用
          // 该工具，而不是编辑 `tools`，因此之前的轮次保持
          // 字节级一致，缓存仍能命中。
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
  use Anthropic\Beta\AnthropicBeta;
  // ...

  $client = new Client();

  $response = $client->beta->messages->create(
      model: Model::CLAUDE_OPUS_5_5,
      maxTokens: 1024,
      betas: [AnthropicBeta::INLINE_TOOLS_2026_09_15],
      // 完整的工具集在开头一次性声明且不再更改，因此
      // 缓存的前缀保持不变。
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
          // 逐字节一致，缓存仍能命中。
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
    model: Anthropic::Model::CLAUDE_OPUS_5_5,
    max_tokens: 1024,
    betas: [Anthropic::AnthropicBeta::INLINE_TOOLS_2026_09_15],
    # 完整的工具集在开头一次性声明且不再更改，因此
    # 缓存的前缀保持不变。
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
      # 从此处起撤回 get_weather。该块按名称引用
      # 该工具，而不是编辑 `tools`，因此之前的轮次保持
      # 字节级一致，缓存仍能命中。
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

### 在消息中定义工具（beta）

使用 `inline-tools-2026-09-15` beta 标头时，`tool_addition` 块可以按值定义工具，即携带其完整定义，而不是按引用指定工具名称。这样，您就可以通过追加一条 `role: "system"` 消息，引入一个在对话开始时未知的工具，或者其 schema 之后会发生变化的工具。`tools` 数组和之前的每条消息都与发送时完全一致，因此提示缓存仍会命中，只有追加的消息会作为新输入进行处理。唯一的例外是 `tools` 数组中没有任何非延迟工具的情况，下面的规则中会介绍。该标头也涵盖按引用添加和移除工具，因此您无需同时发送 `mid-conversation-tool-changes-2026-07-01`。

将定义包装在类型为 `tool_definition` 的 `tool` 对象中。`definition` 是一个 `tools` 条目，例如自定义工具或 Anthropic 定义的客户端或服务器工具，并带有其常规配置，包括 `cache_control` 和 `defer_loading`。在 beta 期间，某些工具类型（包括计算机使用工具）暂时还不能在消息中定义，会返回说明这一点的 400 错误；请在 `tools` 中声明这些工具，并按引用添加它们。例如，要在对话中途定义一个自定义工具：

```json
{
  "role": "system",
  "content": [
    {
      "type": "tool_addition",
      "tool": {
        "type": "tool_definition",
        "definition": {
          "name": "db_query",
          "description": "Run a read-only SQL query against the analytics database.",
          "input_schema": {
            "type": "object",
            "properties": { "sql": { "type": "string" } },
            "required": ["sql"]
          }
        }
      }
    }
  ]
}
```

从该位置开始，模型可以像调用在 `tools` 中声明的工具一样调用该工具。再次发送相同的定义不会产生任何变化，因此客户端可以安全地重新发送它，例如在重试时。

以下请求在 `tools` 中保留 `get_weather`，并在第一个用户轮次之后定义 `db_query`：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: inline-tools-2026-09-15" \
    -d '{
      "model": "claude-opus-5-5",
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
          "content": "How many orders shipped yesterday?"
        },
        {
          "role": "system",
          "content": [
            {
              "type": "tool_addition",
              "tool": {
                "type": "tool_definition",
                "definition": {
                  "name": "db_query",
                  "description": "Run a read-only SQL query against the analytics database.",
                  "input_schema": {
                    "type": "object",
                    "properties": {"sql": {"type": "string"}},
                    "required": ["sql"]
                  }
                }
              }
            }
          ]
        }
      ]
    }'
  ```

  ```bash CLI
  ant beta:messages create --beta inline-tools-2026-09-15 <<'YAML'
  model: claude-opus-5-5
  max_tokens: 1024
  # 在 `tools` 中至少保留一个非延迟加载的工具，这样稍后定义的工具
  # 不会改变渲染后提示的开头部分。
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
      content: How many orders shipped yesterday?
    # 从此处开始按值定义 db_query。`tools` 和
    # 之前的消息与发送时完全一致，因此缓存仍会命中。
    - role: system
      content:
        - type: tool_addition
          tool:
            type: tool_definition
            definition:
              name: db_query
              description: Run a read-only SQL query against the analytics database.
              input_schema:
                type: object
                properties:
                  sql:
                    type: string
                required:
                  - sql
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.beta.messages.create(
      model="claude-opus-5-5",
      max_tokens=1024,
      betas=["inline-tools-2026-09-15"],
      # 在 `tools` 中至少保留一个非延迟加载的工具，这样之后定义的工具
      # 不会改变渲染后提示的开头部分。
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
          {"role": "user", "content": "How many orders shipped yesterday?"},
          # 从此处起按值定义 db_query。`tools` 和
          # 之前的消息与发送时完全一致，因此缓存仍会命中。
          {
              "role": "system",
              "content": [
                  {
                      "type": "tool_addition",
                      "tool": {
                          "type": "tool_definition",
                          "definition": {
                              "name": "db_query",
                              "description": "Run a read-only SQL query against the analytics database.",
                              "input_schema": {
                                  "type": "object",
                                  "properties": {"sql": {"type": "string"}},
                                  "required": ["sql"],
                              },
                          },
                      },
                  },
              ],
          },
      ],
  )

  for block in response.content:
      if block.type == "tool_use":
          print(block.name, block.input)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-opus-5-5",
    max_tokens: 1024,
    betas: ["inline-tools-2026-09-15"],
    // 在 `tools` 中至少保留一个非延迟加载的工具，这样之后定义的工具
    // 不会改变渲染后提示的开头部分。
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
      { role: "user", content: "How many orders shipped yesterday?" },
      // 从此处起按值定义 db_query。`tools` 和
      // 之前的消息与发送时完全一致，因此缓存仍能命中。
      {
        role: "system",
        content: [
          {
            type: "tool_addition",
            tool: {
              type: "tool_definition",
              definition: {
                name: "db_query",
                description: "Run a read-only SQL query against the analytics database.",
                input_schema: {
                  type: "object",
                  properties: { sql: { type: "string" } },
                  required: ["sql"]
                }
              }
            }
          }
        ]
      }
    ]
  });

  for (const block of response.content) {
    if (block.type === "tool_use") {
      console.log(block.name, JSON.stringify(block.input));
    }
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var response = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5_5,
      MaxTokens = 1024,
      Betas = [AnthropicBeta.InlineTools2026_09_15],
      // 在 `Tools` 中至少保留一个非延迟加载（non-deferred）的工具，这样之后定义的工具
      // 不会改变渲染后提示的开头部分。
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
          new() { Role = Role.User, Content = "How many orders shipped yesterday?" },
          // 从此处开始按值定义 db_query。`Tools` 和
          // 之前的消息与发送时完全一致，因此仍能命中缓存。
          new()
          {
              Role = Role.System,
              Content = new(
              [
                  new BetaRequestToolAdditionBlock
                  {
                      Tool = new BetaToolChangeToolDefinitionParam
                      {
                          Definition = new BetaTool
                          {
                              Name = "db_query",
                              Description = "Run a read-only SQL query against the analytics database.",
                              InputSchema = new InputSchema
                              {
                                  Properties = new Dictionary<string, JsonElement>
                                  {
                                      ["sql"] = JsonSerializer.SerializeToElement(new { type = "string" }),
                                  },
                                  Required = ["sql"],
                              },
                          },
                      },
                  },
              ]),
          },
      ],
  });

  foreach (var block in response.Content)
  {
      if (block.TryPickToolUse(out var toolUse))
      {
          Console.WriteLine($"{toolUse.Name} {JsonSerializer.Serialize(toolUse.Input)}");
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5_5,
  	MaxTokens: 1024,
  	Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaInlineTools2026_09_15},
  	// 在 Tools 中至少保留一个非延迟加载的工具，这样之后定义的工具
  	// 不会改变渲染后提示的开头部分。
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
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("How many orders shipped yesterday?")),
  		// 从此处开始按值定义 db_query。Tools 和
  		// 之前的消息与发送时完全一致，因此缓存仍会命中。
  		{
  			Role: anthropic.BetaMessageParamRoleSystem,
  			Content: []anthropic.BetaContentBlockParamUnion{
  				anthropic.NewBetaToolAdditionBlock(anthropic.BetaToolChangeToolDefinitionParam{
  					Definition: anthropic.BetaToolUnionParam{OfTool: &anthropic.BetaToolParam{
  						Name:        "db_query",
  						Description: anthropic.String("Run a read-only SQL query against the analytics database."),
  						InputSchema: anthropic.BetaToolInputSchemaParam{
  							Properties: map[string]any{
  								"sql": map[string]any{"type": "string"},
  							},
  							Required: []string{"sql"},
  						},
  					}},
  				}),
  			},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  for _, block := range response.Content {
  	if toolUse, ok := block.AsAny().(anthropic.BetaToolUseBlock); ok {
  		fmt.Println(toolUse.Name, toolUse.Input)
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaContentBlockParam;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaRequestToolAdditionBlock;
  import com.anthropic.models.beta.messages.BetaTool;
  import com.anthropic.models.beta.messages.MessageCreateParams;
  // ...

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

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

      BetaTool dbQueryTool = BetaTool.builder()
          .name("db_query")
          .description("Run a read-only SQL query against the analytics database.")
          .inputSchema(BetaTool.InputSchema.builder()
              .properties(BetaTool.InputSchema.Properties.builder()
                  .putAdditionalProperty("sql", JsonValue.from(Map.of("type", "string")))
                  .build())
              .addRequired("sql")
              .build())
          .build();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5_5)
          .maxTokens(1024)
          .addBeta(AnthropicBeta.INLINE_TOOLS_2026_09_15)
          // 在 `tools` 中至少保留一个非延迟加载的工具，这样之后定义的工具
          // 不会改变渲染后提示的开头部分。
          .addTool(weatherTool)
          .addUserMessage("How many orders shipped yesterday?")
          // 从此处起按值定义 db_query。`tools` 和
          // 之前的消息与发送时完全一致，因此缓存仍能命中。
          .addSystemMessageOfBetaContentBlockParams(List.of(
              BetaContentBlockParam.ofToolAddition(BetaRequestToolAdditionBlock.builder()
                  .definitionTool(dbQueryTool)
                  .build())))
          .build();

      BetaMessage response = client.beta().messages().create(params);
      response.content().stream()
          .flatMap(block -> block.toolUse().stream())
          .forEach(toolUse -> IO.println(toolUse.name() + " " + toolUse._input()));
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaToolUseBlock;
  // ...

  $client = new Client();

  $response = $client->beta->messages->create(
      model: Model::CLAUDE_OPUS_5_5,
      maxTokens: 1024,
      betas: [AnthropicBeta::INLINE_TOOLS_2026_09_15],
      // 在 `tools` 中至少保留一个非延迟加载的工具，这样之后定义的工具
      // 不会改变渲染后提示的开头部分。
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
          ['role' => 'user', 'content' => 'How many orders shipped yesterday?'],
          // 从此处开始按值定义 db_query。`tools` 和
          // 之前的消息与发送时完全一致，因此缓存仍会命中。
          [
              'role' => 'system',
              'content' => [
                  [
                      'type' => 'tool_addition',
                      'tool' => [
                          'type' => 'tool_definition',
                          'definition' => [
                              'name' => 'db_query',
                              'description' => 'Run a read-only SQL query against the analytics database.',
                              'input_schema' => [
                                  'type' => 'object',
                                  'properties' => ['sql' => ['type' => 'string']],
                                  'required' => ['sql'],
                              ],
                          ],
                      ],
                  ],
              ],
          ],
      ],
  );

  foreach ($response->content as $block) {
      if ($block instanceof BetaToolUseBlock) {
          echo $block->name, ' ', json_encode($block->input), PHP_EOL;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: Anthropic::Model::CLAUDE_OPUS_5_5,
    max_tokens: 1024,
    betas: [Anthropic::AnthropicBeta::INLINE_TOOLS_2026_09_15],
    # 在 `tools` 中至少保留一个非延迟（non-deferred）工具，这样之后定义的工具
    # 不会改变渲染后提示的开头部分。
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
      { role: "user", content: "How many orders shipped yesterday?" },
      # 从此处起按值定义 db_query。`tools` 和
      # 之前的消息与发送时完全一致，因此缓存仍会命中。
      {
        role: "system",
        content: [
          {
            type: "tool_addition",
            tool: {
              type: "tool_definition",
              definition: {
                name: "db_query",
                description: "Run a read-only SQL query against the analytics database.",
                input_schema: {
                  type: "object",
                  properties: { sql: { type: "string" } },
                  required: ["sql"]
                }
              }
            }
          }
        ]
      }
    ]
  )

  response.content.each do |block|
    puts "#{block.name} #{block.input}" if block.is_a?(Anthropic::Beta::BetaToolUseBlock)
  end
  ```
</CodeGroup>

响应的 `content` 中包含针对新工具的 `tool_use` 块，例如：

```json
{
  "type": "tool_use",
  "id": "toolu_01A09q90qw90lq917835lq9",
  "name": "db_query",
  "input": {
    "sql": "SELECT COUNT(*) FROM orders WHERE shipped_at::date = CURRENT_DATE - 1"
  }
}
```

要更改工具的 schema，或将服务器工具迁移到更新的版本，请以相同名称发送不同的定义。新定义从该位置开始替换之前的定义。如果某个定义复用了另一种类型工具的名称，则会返回 400 错误，且 `error.details.error_code` 设置为 `tool_name_conflict`。同一工具的更新版本不算作不同类型。`tool_removal` 仍然接受引用，被移除的工具之后可以再次定义或重新提供。

根据定义的渲染位置，有以下几条规则：

* **预先声明您已知的内容。** 在第一个请求时就已知的工具应放在 `tools` 中；如果模型暂时不应看到它，则使用 `defer_loading: true` 并在之后通过 `tool_addition` 引用。只有在第一个请求时未知或之后会发生变化的工具才按值定义。
* **在 `tools` 中至少保留一个非延迟工具。** 如果对话的 `tools` 数组中没有非延迟工具，请求仍会被接受，但它按值定义的第一个工具会改变渲染后提示的开头部分，这会导致该请求出现一次完整的缓存未命中。[工具搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)算作非延迟工具。
* **带日期的工具类型保留各自的 beta 标头。** 如果您按值定义的服务器工具需要其自身的 beta 标头，请在对话中之后的每个请求上都发送该标头。
* **`cache_control` 放在块上或定义中，不能两者都放，** 并且计入请求的断点数量限制。延迟定义不能携带 `cache_control`。

当超出以下任一限制时，请求会返回 400 错误，且 `error.details.error_code` 设置为 `available_tools_limit_exceeded`：

* 在任意消息之后可用的延迟工具超过 10,000 个。
* 在任意消息之后可用的、在第一条用户消息之后定义的工具超过 10,000 个。
* 在第一条用户消息之后发送、且在任意消息之后仍然可用的工具定义总计超过 4 MB（4,194,304 字节）。
* 渲染后的工具文本大于 4 MB（4,194,304 字节）。

### 在对话中途添加 MCP 服务器（beta）

要在对话进行中添加 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)服务器，请将 `mcp-client-2026-09-15` beta 标头与 `inline-tools-2026-09-15` 一起发送。这样，`tool_addition` 块中的 `definition` 就可以是 `mcp_toolset`，从而无需编辑 `tools` 即可使服务器的工具可用。像往常一样在 `mcp_servers` 中列出服务器的连接详细信息，然后在服务器变为可用的位置追加该工具集：

```json
{
  "role": "system",
  "content": [
    {
      "type": "tool_addition",
      "tool": {
        "type": "tool_definition",
        "definition": { "type": "mcp_toolset", "mcp_server_name": "calendar" }
      }
    }
  ]
}
```

`mcp_toolset` 对象与您放在 `tools` 中的对象相同，包括 `default_config` 和 `configs`。`tool_addition` 块从不包含服务器 URL 或令牌，这些信息保留在 `mcp_servers` 中。

以下请求将 `get_weather` 保留在 `tools` 中，在 `mcp_servers` 中列出日历服务器，并在第一个用户轮次之后添加该服务器的工具集：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: inline-tools-2026-09-15,mcp-client-2026-09-15" \
    -d '{
      "model": "claude-opus-5-5",
      "max_tokens": 1024,
      "mcp_servers": [
        {
          "type": "url",
          "url": "https://mcp.example.com/calendar",
          "name": "calendar",
          "authorization_token": "YOUR_TOKEN"
        }
      ],
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
          "content": "What'\''s on my calendar tomorrow?"
        },
        {
          "role": "system",
          "content": [
            {
              "type": "tool_addition",
              "tool": {
                "type": "tool_definition",
                "definition": {
                  "type": "mcp_toolset",
                  "mcp_server_name": "calendar"
                }
              }
            }
          ]
        }
      ]
    }'
  ```

  ```bash CLI
  ant beta:messages create \
    --beta inline-tools-2026-09-15,mcp-client-2026-09-15 <<'YAML'
  model: claude-opus-5-5
  max_tokens: 1024
  mcp_servers:
    - type: url
      url: https://mcp.example.com/calendar
      name: calendar
      authorization_token: YOUR_TOKEN
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
      content: What's on my calendar tomorrow?
    # 从此处开始，使日历服务器的工具可供使用。
    # 该块仅指明服务器名称，从不包含 URL 或令牌。
    - role: system
      content:
        - type: tool_addition
          tool:
            type: tool_definition
            definition:
              type: mcp_toolset
              mcp_server_name: calendar
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.beta.messages.create(
      model="claude-opus-5-5",
      max_tokens=1024,
      betas=["inline-tools-2026-09-15", "mcp-client-2026-09-15"],
      mcp_servers=[
          {
              "type": "url",
              "url": "https://mcp.example.com/calendar",
              "name": "calendar",
              "authorization_token": "YOUR_TOKEN",
          },
      ],
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
          {"role": "user", "content": "What's on my calendar tomorrow?"},
          # 从此处开始，使日历服务器的工具可供使用。
          # 该块仅指明服务器名称，从不包含 URL 或令牌。
          {
              "role": "system",
              "content": [
                  {
                      "type": "tool_addition",
                      "tool": {
                          "type": "tool_definition",
                          "definition": {
                              "type": "mcp_toolset",
                              "mcp_server_name": "calendar",
                          },
                      },
                  },
              ],
          },
      ],
  )

  # 响应以日历服务器的 mcp_tool_listing 块开头，
  # 因此请检查每个块的类型，而不是直接读取 content[0]。
  for block in response.content:
      match block.type:
          case "mcp_tool_listing":
              print(block.mcp_server_name, [tool.name for tool in block.tools])
          case "text":
              print(block.text)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-opus-5-5",
    max_tokens: 1024,
    betas: ["inline-tools-2026-09-15", "mcp-client-2026-09-15"],
    mcp_servers: [
      {
        type: "url",
        url: "https://mcp.example.com/calendar",
        name: "calendar",
        authorization_token: "YOUR_TOKEN"
      }
    ],
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
      { role: "user", content: "What's on my calendar tomorrow?" },
      // 从此处开始，使日历服务器的工具可供使用。
      // 该块仅指明服务器名称，从不包含 URL 或令牌。
      {
        role: "system",
        content: [
          {
            type: "tool_addition",
            tool: {
              type: "tool_definition",
              definition: { type: "mcp_toolset", mcp_server_name: "calendar" }
            }
          }
        ]
      }
    ]
  });

  // 响应以日历服务器的 mcp_tool_listing 块开头，
  // 因此请检查每个块的类型，而不是直接读取 content[0]。
  for (const block of response.content) {
    switch (block.type) {
      case "mcp_tool_listing":
        console.log(
          block.mcp_server_name,
          block.tools.map((tool) => tool.name)
        );
        break;
      case "text":
        console.log(block.text);
        break;
    }
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var response = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5_5,
      MaxTokens = 1024,
      Betas = [AnthropicBeta.InlineTools2026_09_15, AnthropicBeta.McpClient2026_09_15],
      McpServers =
      [
          new BetaRequestMcpServerUrlDefinition
          {
              Url = "https://mcp.example.com/calendar",
              Name = "calendar",
              AuthorizationToken = "YOUR_TOKEN",
          },
      ],
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
          new() { Role = Role.User, Content = "What's on my calendar tomorrow?" },
          // 从此处开始，使日历服务器的工具可供使用。
          // 该块仅指明服务器名称，从不包含 URL 或令牌。
          new()
          {
              Role = Role.System,
              Content = new(
              [
                  new BetaRequestToolAdditionBlock
                  {
                      Tool = new BetaToolChangeToolDefinitionParam
                      {
                          Definition = new BetaMcpToolset("calendar"),
                      },
                  },
              ]),
          },
      ],
  });

  // 响应以日历服务器的 mcp_tool_listing 块开头，
  // 因此请检查每个块的类型，而不是直接读取 Content[0]。
  foreach (var block in response.Content)
  {
      if (block.TryPickMcpToolListing(out var listing))
      {
          Console.WriteLine($"{listing.McpServerName} {JsonSerializer.Serialize(listing.Tools.Select(tool => tool.Name))}");
      }
      else if (block.TryPickText(out var text))
      {
          Console.WriteLine(text.Text);
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5_5,
  	MaxTokens: 1024,
  	Betas: []anthropic.AnthropicBeta{
  		anthropic.AnthropicBetaInlineTools2026_09_15,
  		anthropic.AnthropicBetaMCPClient2026_09_15,
  	},
  	MCPServers: []anthropic.BetaRequestMCPServerURLDefinitionParam{
  		{
  			URL:                "https://mcp.example.com/calendar",
  			Name:               "calendar",
  			AuthorizationToken: anthropic.String("YOUR_TOKEN"),
  		},
  	},
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
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("What's on my calendar tomorrow?")),
  		// 从此处开始，使日历服务器的工具可供使用。
  		// 该块仅指明服务器名称，绝不包含 URL 或令牌。
  		{
  			Role: anthropic.BetaMessageParamRoleSystem,
  			Content: []anthropic.BetaContentBlockParamUnion{
  				anthropic.NewBetaToolAdditionBlock(anthropic.BetaToolChangeToolDefinitionParam{
  					Definition: anthropic.BetaToolUnionParam{OfMCPToolset: &anthropic.BetaMCPToolsetParam{
  						MCPServerName: "calendar",
  					}},
  				}),
  			},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  // 响应以日历服务器的 mcp_tool_listing 块开头，
  // 因此请检查每个块的类型，而不是直接读取 Content[0]。
  for _, block := range response.Content {
  	switch variant := block.AsAny().(type) {
  	case anthropic.BetaMCPToolListingBlock:
  		var toolNames []string
  		for _, tool := range variant.Tools {
  			toolNames = append(toolNames, tool.Name)
  		}
  		fmt.Println(variant.MCPServerName, toolNames)
  	case anthropic.BetaTextBlock:
  		fmt.Println(variant.Text)
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaContentBlockParam;
  import com.anthropic.models.beta.messages.BetaMcpTool;
  import com.anthropic.models.beta.messages.BetaMcpToolset;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaRequestMcpServerUrlDefinition;
  import com.anthropic.models.beta.messages.BetaRequestToolAdditionBlock;
  import com.anthropic.models.beta.messages.BetaTool;
  import com.anthropic.models.beta.messages.MessageCreateParams;
  // ...

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

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
          .model(Model.CLAUDE_OPUS_5_5)
          .maxTokens(1024)
          .addBeta(AnthropicBeta.INLINE_TOOLS_2026_09_15)
          .addBeta(AnthropicBeta.MCP_CLIENT_2026_09_15)
          .addMcpServer(BetaRequestMcpServerUrlDefinition.builder()
              .url("https://mcp.example.com/calendar")
              .name("calendar")
              .authorizationToken("YOUR_TOKEN")
              .build())
          .addTool(weatherTool)
          .addUserMessage("What's on my calendar tomorrow?")
          // 从此处开始，使日历服务器的工具可供使用。
          // 该块仅指明服务器名称，从不包含 URL 或令牌。
          .addSystemMessageOfBetaContentBlockParams(List.of(
              BetaContentBlockParam.ofToolAddition(BetaRequestToolAdditionBlock.builder()
                  .definitionTool(BetaMcpToolset.builder()
                      .mcpServerName("calendar")
                      .build())
                  .build())))
          .build();

      BetaMessage response = client.beta().messages().create(params);

      // 响应以日历服务器的 mcp_tool_listing 块开头，
      // 因此请检查每个块的类型，而不是直接读取第一个块。
      for (var block : response.content()) {
          switch (block.type().value()) {
              case MCP_TOOL_LISTING -> {
                  var listing = block.asMcpToolListing();
                  var toolNames = listing.tools().stream().map(BetaMcpTool::name).toList();
                  IO.println(listing.mcpServerName() + " " + toolNames);
              }
              case TEXT -> IO.println(block.asText().text());
          }
      }
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaMCPTool;
  use Anthropic\Beta\Messages\BetaMCPToolListingBlock;
  use Anthropic\Beta\Messages\BetaTextBlock;
  // ...

  $client = new Client();

  $response = $client->beta->messages->create(
      model: Model::CLAUDE_OPUS_5_5,
      maxTokens: 1024,
      betas: [
          AnthropicBeta::INLINE_TOOLS_2026_09_15,
          AnthropicBeta::MCP_CLIENT_2026_09_15,
      ],
      mcpServers: [
          [
              'type' => 'url',
              'url' => 'https://mcp.example.com/calendar',
              'name' => 'calendar',
              'authorization_token' => 'YOUR_TOKEN',
          ],
      ],
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
          ['role' => 'user', 'content' => "What's on my calendar tomorrow?"],
          // 从此处开始，使日历服务器的工具可供使用。
          // 该块仅指明服务器名称，从不包含 URL 或令牌。
          [
              'role' => 'system',
              'content' => [
                  [
                      'type' => 'tool_addition',
                      'tool' => [
                          'type' => 'tool_definition',
                          'definition' => [
                              'type' => 'mcp_toolset',
                              'mcp_server_name' => 'calendar',
                          ],
                      ],
                  ],
              ],
          ],
      ],
  );

  // 响应以日历服务器的 mcp_tool_listing 块开头，
  // 因此请检查每个块的类型，而不是直接读取 content[0]。
  foreach ($response->content as $block) {
      switch (true) {
          case $block instanceof BetaMCPToolListingBlock:
              $toolNames = array_map(fn (BetaMCPTool $tool) => $tool->name, $block->tools);
              echo $block->mcpServerName, ' ', json_encode($toolNames), PHP_EOL;
              break;
          case $block instanceof BetaTextBlock:
              echo $block->text, PHP_EOL;
              break;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: Anthropic::Model::CLAUDE_OPUS_5_5,
    max_tokens: 1024,
    betas: [
      Anthropic::AnthropicBeta::INLINE_TOOLS_2026_09_15,
      Anthropic::AnthropicBeta::MCP_CLIENT_2026_09_15
    ],
    mcp_servers: [
      {
        type: "url",
        url: "https://mcp.example.com/calendar",
        name: "calendar",
        authorization_token: "YOUR_TOKEN"
      }
    ],
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
      { role: "user", content: "What's on my calendar tomorrow?" },
      # 从此处开始，使日历服务器的工具可供使用。
      # 该块仅指明服务器名称，从不包含 URL 或令牌。
      {
        role: "system",
        content: [
          {
            type: "tool_addition",
            tool: {
              type: "tool_definition",
              definition: { type: "mcp_toolset", mcp_server_name: "calendar" }
            }
          }
        ]
      }
    ]
  )

  # 响应以日历服务器的 mcp_tool_listing 块开头，
  # 因此请检查每个块的类型，而不是直接读取 content[0]。
  response.content.each do |block|
    case block
    when Anthropic::Beta::BetaMCPToolListingBlock
      puts "#{block.mcp_server_name} #{block.tools.map(&:name)}"
    when Anthropic::Beta::BetaTextBlock
      puts block.text
    end
  end
  ```
</CodeGroup>

使用 `mcp-client-2026-09-15` 时，如果 API 为某个响应获取了服务器的工具列表，该响应会以 `mcp_tool_listing` 块开头，每个被获取的服务器对应一个块。如果您的代码读取 `content[0]`，请跳过这些块。请原样发回助手消息（包括此块），并在每个携带该消息的请求上持续发送 `mcp-client-2026-09-15`。之后的请求将使用记录下来的列表，而不会再次向服务器查询。要自行固定工具集，请将该列表复制到 `mcp_toolset` 的 `tools` 字段中，如[固定 MCP 服务器的工具列表](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#pin-mcp-tool-list)中所述。

`mcp-client-2026-09-15` 包含 `mcp-client-2025-11-20` 的所有功能，因此您无需同时发送两者。这些功能可在 Claude API 上使用。使用 MCP 连接器的请求仍遵循其[数据保留](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#data-retention)条款。

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

`role: "system"` 消息还可以携带 `output_config.effort`，从下一个 `user` 轮次开始更改 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别。此功能在 Claude API 和 Google Cloud 上的 Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5.5、Claude Opus 5 上处于 beta 阶段，需要 `mid-conversation-output-config-2026-07-01` beta 标头。请参阅[按消息设置 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5-5",
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
  model: claude-opus-5-5
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
      model="claude-opus-5-5",
      max_tokens=1024,
      # 自动 prompt caching（提示缓存）：每个请求都会缓存截至目前的对话，
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
          # 通过团队严格的类型检查策略。将该
          # 指令追加在此处可使之前的轮次保持字节级一致，因此
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
    model: "claude-opus-5-5",
    max_tokens: 1024,
    // 自动 prompt caching（提示缓存）：每个请求都会缓存截至目前的对话，
    // 下一个请求则从缓存中读取未变化的前缀。
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
      // 团队严格的类型检查策略。在此处追加该指令可使
      // 之前的轮次保持字节级一致，因此上一个请求缓存的前缀
      // 仍可从缓存中读取。
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
      Model = Model.ClaudeOpus5_5,
      MaxTokens = 1024,
      // 自动 prompt caching（提示缓存）：每个请求都会缓存截至目前的对话，
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
          // 审阅者在会话中途意识到，所有建议还必须符合
          // 团队严格的类型检查策略。将该指令追加到此处可使
          // 之前的轮次保持字节级一致，因此上一个请求缓存的前缀
          // 仍可从缓存中读取。
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
  	Model:     anthropic.ModelClaudeOpus5_5,
  	MaxTokens: 1024,
  	// 自动 prompt caching（提示缓存）：每个请求都会缓存截至目前的对话，
  	// 下一个请求则从缓存中读取未改变的前缀。
  	CacheControl: anthropic.NewCacheControlEphemeralParam(),
  	System: []anthropic.TextBlockParam{
  		{Text: "You are a code review assistant. Be concise."},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Review process() in utils.py for performance issues.")),
  		anthropic.NewAssistantMessage(anthropic.NewTextBlock("The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list.")),
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Now review the calling code that invokes process().")),
  		// 审阅者在会话中途意识到，所有建议还必须
  		// 通过团队严格的类型检查策略。将该指令追加
  		// 在此处可使之前的轮次保持字节级一致，因此上一个请求
  		// 缓存的前缀仍会从缓存中读取。
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
          .model(Model.CLAUDE_OPUS_5_5)
          .maxTokens(1024)
          // 自动 prompt caching（提示缓存）：每个请求都会缓存截至目前的对话，
          // 下一个请求则从缓存中读取未改变的前缀。
          .cacheControl(CacheControlEphemeral.builder().build())
          .system("You are a code review assistant. Be concise.")
          .addUserMessage("Review process() in utils.py for performance issues.")
          .addAssistantMessage("The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list.")
          .addUserMessage("Now review the calling code that invokes process().")
          // 审阅者在会话中途意识到，所有建议还必须符合
          // 团队严格的类型检查策略。在此处追加该指令可使
          // 之前的轮次保持字节级一致，因此上一个请求缓存的前缀
          // 仍可从缓存中读取。
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
          // 审阅者在会话中途意识到，所有建议还必须符合
          // 团队严格的类型检查策略。将该指令追加到此处可使
          // 之前的轮次保持字节级一致，因此上一个请求缓存的前缀
          // 仍可从缓存中读取。
          ['role' => 'system', 'content' => 'From now on, every suggestion must include explicit type annotations.']
      ],
      model: 'claude-opus-5-5',
      // 自动提示缓存：每个请求都会缓存截至目前的对话，
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
    model: "claude-opus-5-5",
    max_tokens: 1024,
    # 自动 prompt caching（提示缓存）：每个请求都会缓存截至目前的对话，
    # 下一个请求则从缓存中读取未改变的前缀。
    cache_control: { type: "ephemeral" },
    system: "You are a code review assistant. Be concise.",
    messages: [
      { role: "user", content: "Review process() in utils.py for performance issues." },
      { role: "assistant", content: "The list comprehension is fine for small inputs. For large inputs, consider a generator to avoid materializing the full list." },
      { role: "user", content: "Now review the calling code that invokes process()." },
      # 审阅者在会话中途意识到，所有建议还必须符合
      # 团队严格的类型检查策略。在此处追加该指令可使
      # 之前的轮次保持字节级一致，因此上一个请求缓存的前缀
      # 仍可从缓存中读取。
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

其主要用途是在工具循环中发送每轮提醒。每当您希望模型看到提醒时，就在 `tool_result` 消息之后追加该提醒，并将之前的每一份副本都保留在原处。模型只会看到最后一条用户消息之后的副本，因此提醒永远不会堆积。`messages` 中较早的内容都没有改变，因此[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)会持续匹配。在 Claude Fable 5.1 和 Claude Opus 5.5 上，这还能使之后的[思考块保持有效](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)：删除较早的提醒会改变这些块之前的对话内容，导致对话检查失败，而被清除的消息仍保留在数组中，不会改变该对话内容。

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

* **原样重新发送被清除的消息。** 被清除的消息仍然是对话历史的一部分。根据当前状态重建它（新的令牌计数、时间戳）、将其视为冗余而删除，或更改其 `clear_at` 值，都属于对较早消息的编辑。提示缓存会从该位置开始未命中，并且在 Claude Fable 5.1 和 Claude Opus 5.5 上，在其之后生成的每个思考块都会无法通过[对话检查](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)。
* **仅限文本。** `content` 是一个或多个 `text` 块（或一个字符串）。在轮次作用域消息上使用 `tool_addition` 和 `tool_removal` 块会返回 400 错误，使用 `output_config` 也是如此。对于这些内容，请使用不带 `clear_at` 的单独 `role: "system"` 消息。
* **其块上不能有 `cache_control`。** 被清除的消息永远不会成为缓存键的一部分，因此其上的断点永远无法匹配。请改为将断点放在前一个用户轮次的最后一个块上，如示例所示。顶层[自动缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#automatic-caching)字段在选择断点时会跳过轮次作用域消息。在清除某条消息的请求中，可复用的已缓存前缀在该消息之前的用户轮次处结束，因此只有该消息与新用户消息之间的那一个助手轮次会被重新处理。
* **放置规则仍然适用**，无论消息是否已被清除。与任何对话中途的系统消息一样，轮次作用域消息必须跟在 `user` 轮次（或以服务器工具结果结尾的 `assistant` 轮次）之后，并且位于 `assistant` 轮次之前或位于数组末尾。位于数组末尾的消息始终会渲染。如果其后直接跟着另一条 `user` 消息，则会返回 400 错误，而不是成为已清除的消息：请将一轮工具调用的所有结果放在一条用户消息中，并将提醒放在其后。
* **助手轮次不会清除它。** 该消息之后的预填充或[暂停](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons#pause-turn)的助手轮次，或者服务器端工具循环，都不会添加用户消息，因此该消息在这种延续中仍会渲染。要在客户端工具循环中始终让提醒可见，请在每条 `tool_result` 消息之后再次追加它。
* **令牌计数以实际渲染的内容为准。** 被清除的消息不会增加 `usage.input_tokens` 或[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)。
* **导入的历史记录。** 在您一次性构建的对话记录中（少样本示例、迁移过来的对话），如果某条轮次作用域消息之后已经有一个助手轮次和一条用户消息，那么它从第一个请求起就会被清除，永远不会渲染。对于您要沿用的每轮提醒来说，这正是正确的状态。只有对于模型应在每个请求中都看到的消息，才不设置 `clear_at`。

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
    # 轮次级提醒：仅在本轮渲染，之后出现新的用户消息时即清除。
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
          # 轮次范围的提醒：在本轮中渲染，待后续出现用户消息后即清除。
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
      // 轮次范围的提醒：仅在本轮渲染，待出现后续用户消息后即清除。
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
          // 轮次范围的提醒：在本轮中渲染，待后续出现用户消息后即清除。
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
  		// 轮次级提醒：仅在本轮渲染，之后出现新的用户消息时即清除。
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
          // 轮次级提醒：仅在本轮渲染，之后出现新的用户消息时即清除。
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
          // 轮次范围的提醒：在本轮中渲染，待后续出现用户消息后即清除。
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
      # 轮次范围的提醒：在本轮中渲染，待后续出现用户消息后即清除。
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

避免编辑或删除已经发送的对话中途系统消息。与对早期消息的任何其他更改一样，这会使从该位置开始的缓存失效。在 Claude Fable 5.1 和 Claude Opus 5.5 上，这还会使之后每个助手轮次中的[思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)失效。对于只应适用于单个轮次的指导，请使用[轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)并将其保留在原位。如果指令需要演变，请追加一条新的系统消息，而不是重写旧消息。连续的系统消息是被接受的，并被视为单个系统部分，该部分作为一个整体遵循相同的放置规则。

## 限制

* **不能作为第一条消息。** 携带内容的 `system` 消息不能是 `messages` 中的第一个条目。对于从一开始就适用的指令，请使用顶层 `system` 字段。
* **放置位置受到限制。** 携带内容（`text`、`tool_addition` 或 `tool_removal` 块）的 `system` 消息必须紧跟在 `user` 轮次（包括携带 `tool_result` 块的 `user` 轮次）或以服务器工具结果结尾的 `assistant` 轮次之后，并且必须位于 `assistant` 轮次之前或作为数组的结尾。它不能位于 `tool_use` 块与其 `tool_result` 之间。将其放在其他位置会返回 400 错误。有一个例外：紧跟在[暂停](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons#pause-turn)的 `assistant` 轮次（以服务器工具结果结尾的轮次）之后时，不接受 `tool_addition` 和 `tool_removal` 块，但接受 `text` 块；请先恢复暂停的轮次，然后在下一条 `system` 消息中发送工具变更。`content` 为空、仅设置 [`output_config.effort`](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta) 的消息在其位置不渲染任何内容，可以放在 `messages` 中的任何位置，包括第一个位置，或 `assistant` 轮次与 `user` 轮次之间。连续的 `system` 消息会被一并判断，因此在仅设置 effort 的消息旁边添加一条携带文本的消息，会使整组消息都遵循内容规则。
* **轮次范围的消息仅限文本，且需原样重新发送。** `clear_at: "next_user_message"` 消息不携带 `tool_addition`、`tool_removal`、`output_config` 或 `cache_control`，并且一旦被清除，在之后的请求中必须逐字节保留在 `messages` 中。请参阅[轮次范围的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)。
* **不适合放置不受信任的内容。** Claude 将系统内容视为运营者指令并予以遵循。请勿将来自对话之外的文本（例如原始工具输出、检索到的文档或网页内容）直接放入系统消息中；这样做会赋予该文本运营者级的权限。请将这些数据保留在 `tool_result` 块中，并继续遵循[缓解越狱和提示注入](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks)中的指导。

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
