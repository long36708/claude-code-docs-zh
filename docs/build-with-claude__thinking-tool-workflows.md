---
title: 工具与多轮工作流中的思考
url: https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-tool-workflows
description: 逐步演示一个完整的两轮工具使用往返流程，正确保留思考块，并了解交错思考如何改变流程。
---

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

本页逐步演示一个启用思考的完整两轮 tool use（工具使用）往返流程：Claude 进行思考、请求工具调用、接收结果并完成回答，且在每一步都正确处理 thinking blocks（思考块）。完整规则位于[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)页面的[思考与工具使用](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)和[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)部分；本页展示这些规则在可运行代码中的应用。

## 本演示所应用的规则

每个链接都指向思考页面上的完整说明：

* [在手动模式下将工具选择限制为 `auto` 或 `none`](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)：强制工具使用的 `tool_choice` 选项在手动 extended thinking（扩展思考）（`thinking: {type: "enabled"}`）下会返回错误；自适应思考支持强制工具使用。
* [每个助手轮次保持一种思考配置](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)：一个工具使用循环是一个助手轮次，因此只能在轮次之间更改配置。
* [完整且不加修改地传回思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)：当您返回工具结果时，助手消息中的思考块必须随之一起传回。
* [按接收到的原样回传助手消息](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)：重建消息或过滤掉 `redacted_thinking` 块会触发 400 错误。

示例使用自适应思考；在仅支持扩展思考的模型上，请替换为 `thinking: {type: "enabled", budget_tokens: N}`。往返规则完全相同。

## 逐步演示两轮工具使用往返流程

该示例定义了一个 `get_weather` 工具，让 Claude 思考并请求工具调用，然后返回工具结果，同时按接收到的原样回传助手轮次（包括思考块）。

<Steps>
  <Step title="发起第一个请求并提供可用工具">
    发送一个启用自适应思考并定义了工具的请求。除了 `thinking` 参数之外，这是一个标准的[工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)请求：

    <CodeGroup>
      ```bash CLI
      ant messages create --transform content <<'YAML'
      model: claude-opus-4-8
      max_tokens: 16000
      thinking:
        type: adaptive
      tools:
        - name: get_weather
          description: Get current weather for a location
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
          content: "What's the weather in Paris?"
      YAML
      ```

      ```python Python

      client = anthropic.Anthropic()

      weather_tool = {
          "name": "get_weather",
          "description": "Get current weather for a location",
          "input_schema": {
              "type": "object",
              "properties": {"location": {"type": "string", "description": "City name"}},
              "required": ["location"],
          },
      }

      # 第一次请求 - Claude 以思考内容和工具请求进行响应
      response = client.messages.create(
          model="claude-opus-4-8",
          max_tokens=16000,
          thinking={"type": "adaptive"},
          tools=[weather_tool],
          messages=[{"role": "user", "content": "What's the weather in Paris?"}],
      )
      print(response)
      ```

      ```typescript TypeScript
      const client = new Anthropic();

      const weatherTool: Anthropic.Tool = {
        name: "get_weather",
        description: "Get current weather for a location",
        input_schema: {
          type: "object",
          properties: {
            location: { type: "string", description: "City name" }
          },
          required: ["location"]
        }
      };

      // 第一次请求 - Claude 以思考和工具请求进行响应
      const response = await client.messages.create({
        model: "claude-opus-4-8",
        max_tokens: 16000,
        thinking: {
          type: "adaptive"
        },
        tools: [weatherTool],
        messages: [{ role: "user", content: "What's the weather in Paris?" }]
      });
      console.log(response);
      ```

      ```csharp C#
      AnthropicClient client = new();

      var weatherTool = new ToolUnion(new Tool()
      {
          Name = "get_weather",
          Description = "Get current weather for a location",
          InputSchema = new InputSchema()
          {
              Properties = new Dictionary<string, JsonElement>
              {
                  ["location"] = JsonSerializer.SerializeToElement(new { type = "string", description = "City name" }),
              },
              Required = ["location"],
          },
      });

      var parameters = new MessageCreateParams
      {
          Model = Model.ClaudeOpus4_8,
          MaxTokens = 16000,
          Thinking = new ThinkingConfigAdaptive(),
          Tools = [weatherTool],
          Messages = [new() { Role = Role.User, Content = "What's the weather in Paris?" }]
      };

      var message = await client.Messages.Create(parameters);
      Console.WriteLine(message);
      ```

      ```go Go
      client := anthropic.NewClient()

      weatherTool := anthropic.ToolUnionParam{
      	OfTool: &anthropic.ToolParam{
      		Name:        "get_weather",
      		Description: anthropic.String("Get current weather for a location"),
      		InputSchema: anthropic.ToolInputSchemaParam{
      			Properties: map[string]any{
      				"location": map[string]any{
      					"type":        "string",
      					"description": "City name",
      				},
      			},
      			Required: []string{"location"},
      		},
      	},
      }

      response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
      	Model:     anthropic.ModelClaudeOpus4_8,
      	MaxTokens: 16000,
      	Thinking: anthropic.ThinkingConfigParamUnion{
      		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
      	},
      	Tools: []anthropic.ToolUnionParam{weatherTool},
      	Messages: []anthropic.MessageParam{
      		anthropic.NewUserMessage(anthropic.NewTextBlock("What's the weather in Paris?")),
      	},
      })
      if err != nil {
      	log.Fatal(err)
      }
      fmt.Println(response)
      ```

      ```java Java
      import com.anthropic.models.messages.ThinkingConfigAdaptive;
      // ...
          AnthropicClient client = AnthropicOkHttpClient.fromEnv();

          MessageCreateParams params = MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_4_8)
              .maxTokens(16000L)
              .thinking(ThinkingConfigAdaptive.builder().build())
              .addTool(Tool.builder()
                  .name("get_weather")
                  .description("Get current weather for a location")
                  .inputSchema(Tool.InputSchema.builder()
                      .properties(JsonValue.from(Map.of(
                          "location", Map.of("type", "string", "description", "City name")
                      )))
                      .required(List.of("location"))
                      .build())
                  .build())
              .addUserMessage("What's the weather in Paris?")
              .build();

          Message response = client.messages().create(params);
          IO.println(response);
      ```

      ```php PHP
      $client = new Client();

      $weatherTool = [
          'name' => 'get_weather',
          'description' => 'Get current weather for a location',
          'input_schema' => [
              'type' => 'object',
              'properties' => [
                  'location' => ['type' => 'string', 'description' => 'City name']
              ],
              'required' => ['location']
          ]
      ];

      $message = $client->messages->create(
          maxTokens: 16000,
          messages: [
              ['role' => 'user', 'content' => "What's the weather in Paris?"]
          ],
          model: 'claude-opus-4-8',
          thinking: ['type' => 'adaptive'],
          tools: [$weatherTool],
      );
      echo $message;
      ```

      ```ruby Ruby
      client = Anthropic::Client.new

      weather_tool = {
        name: "get_weather",
        description: "Get current weather for a location",
        input_schema: {
          type: "object",
          properties: {
            location: { type: "string", description: "City name" }
          },
          required: ["location"]
        }
      }

      message = client.messages.create(
        model: "claude-opus-4-8",
        max_tokens: 16000,
        thinking: {
          type: "adaptive"
        },
        tools: [weather_tool],
        messages: [
          { role: "user", content: "What's the weather in Paris?" }
        ]
      )
      puts message
      ```
    </CodeGroup>
  </Step>

  <Step title="捕获要回传的 content 数组">
    在 Claude 选择进行思考的运行中，您应该会在响应内容中看到 `thinking`、`text` 和 `tool_use` 块（对于较简单的请求，自适应模式可能会跳过思考块）。请保持此 content 数组完整无缺：下一步会将其原样发回。

    <Note>
      要看到类似此输出的思考文本，请在请求中添加 `display: "summarized"`。在 display 默认为 omitted 的模型上（包括 claude-opus-4-8），`thinking` 字段否则会以空字符串返回，仅填充 `signature`。无论哪种情况，都请原样回传 content 数组；请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。
    </Note>

    ```json Output
    {
      "content": [
        {
          "type": "thinking",
          "thinking": "The user wants to know the current weather in Paris. I have access to a function `get_weather`...",
          "signature": "BDaL4VrbR2Oj0hO4XpJxT28J5T...."
        },
        {
          "type": "text",
          "text": "I can help you get the current weather information for Paris. Let me check that for you"
        },
        {
          "type": "tool_use",
          "id": "toolu_01CswdEQBMshySk6Y9DFKrfq",
          "name": "get_weather",
          "input": {
            "location": "Paris"
          }
        }
      ]
    }
    ```
  </Step>

  <Step title="返回工具结果，并原样回传助手轮次">
    在您这一侧运行工具，然后发送第二个请求，向对话追加两条消息。第一条是按接收到的原样回传的助手内容，因此思考块与 `tool_use` 块一起保持不变。第二条是携带 `tool_result` 的用户消息。

    每个示例都是一个独立的脚本：它重复第一个请求，然后立即使用刚刚收到的响应发送后续请求。

    <CodeGroup>
      ```bash CLI
      # 第一轮：将 assistant 内容数组（thinking 和 tool_use
      # 块，签名保持完整）写入文件。将模型生成的文本
      # 通过文件传递，可避免其之后处于 shell 展开位置。
      ant messages create --transform content --format jsonl \
        > assistant_content.json <<'YAML'
      model: claude-opus-4-8
      max_tokens: 16000
      thinking:
        type: adaptive
      tools:
        - name: get_weather
          description: Get current weather for a location
          input_schema:
            type: object
            properties:
              location:
                type: string
                description: City name
            required: [location]
      messages:
        - role: user
          content: What's the weather in Paris?
      YAML

      # 第二轮：jq 用捕获的文件填充两个 null 占位符，
      # 使这些块原样作为 assistant 消息返回。thinking
      # 块必须与 tool_use 块一同返回。带引号的定界符可防止
      # shell 展开正文中的任何内容。
      jq --slurpfile blocks assistant_content.json '
        .messages[1].content = $blocks[0] |
        .messages[2].content[0].tool_use_id =
          ($blocks[0][] | select(.type == "tool_use") | .id)
      ' <<'JSON' | ant messages create
      {
        "model": "claude-opus-4-8",
        "max_tokens": 16000,
        "thinking": {"type": "adaptive"},
        "tools": [{
          "name": "get_weather",
          "description": "Get current weather for a location",
          "input_schema": {
            "type": "object",
            "properties": {
              "location": {"type": "string", "description": "City name"}
            },
            "required": ["location"]
          }
        }],
        "messages": [
          {"role": "user", "content": "What's the weather in Paris?"},
          {"role": "assistant", "content": null},
          {"role": "user", "content": [{
            "type": "tool_result",
            "tool_use_id": null,
            "content": "Current temperature: 88°F"
          }]}
        ]
      }
      JSON
      ```

      ```python Python

      client = anthropic.Anthropic()
      weather_tool = {
          "name": "get_weather",
          "description": "Get current weather for a location",
          "input_schema": {
              "type": "object",
              "properties": {"location": {"type": "string", "description": "City name"}},
              "required": ["location"],
          },
      }
      response = client.messages.create(
          model="claude-opus-4-8",
          max_tokens=16000,
          thinking={"type": "adaptive"},
          tools=[weather_tool],
          messages=[{"role": "user", "content": "What's the weather in Paris?"}],
      )
      # 提取 tool_use 块以获取其 ID，用于 tool_result
      tool_use_block = next(block for block in response.content if block.type == "tool_use")

      # 调用您实际的天气 API，此处是您实际 API 调用的位置
      # 假设这是我们得到的返回结果
      weather_data = {"temperature": 88}

      # 第二次请求 - 包含 assistant 轮次和 tool_result
      continuation = client.messages.create(
          model="claude-opus-4-8",
          max_tokens=16000,
          thinking={"type": "adaptive"},
          tools=[weather_tool],
          messages=[
              {"role": "user", "content": "What's the weather in Paris?"},
              # 按原样回传 assistant 内容。当存在 thinking
              # 块时，它必须与 tool_use 块一同提供。
              {"role": "assistant", "content": response.content},
              {
                  "role": "user",
                  "content": [
                      {
                          "type": "tool_result",
                          "tool_use_id": tool_use_block.id,
                          "content": f"Current temperature: {weather_data['temperature']}°F",
                      }
                  ],
              },
          ],
      )
      print(continuation)
      ```

      ```typescript TypeScript
      const client = new Anthropic();

      const weatherTool: Anthropic.Tool = {
        name: "get_weather",
        description: "Get current weather for a location",
        input_schema: {
          type: "object",
          properties: {
            location: { type: "string", description: "City name" }
          },
          required: ["location"]
        }
      };

      const response = await client.messages.create({
        model: "claude-opus-4-8",
        max_tokens: 16000,
        thinking: {
          type: "adaptive"
        },
        tools: [weatherTool],
        messages: [{ role: "user", content: "What's the weather in Paris?" }]
      });

      // 提取 tool_use 块以获取其 ID，用于构造 tool_result
      const toolUseBlock = response.content.find(
        (block): block is Anthropic.ToolUseBlock => block.type === "tool_use"
      );

      // 调用您实际的天气 API，此处放置您实际的 API 调用
      // 假设这是我们得到的返回结果
      const weatherData = { temperature: 88 };

      if (toolUseBlock) {
        // 第二次请求 - 包含 assistant 轮次和 tool_result
        const continuation = await client.messages.create({
          model: "claude-opus-4-8",
          max_tokens: 16000,
          thinking: {
            type: "adaptive"
          },
          tools: [weatherTool],
          messages: [
            { role: "user", content: "What's the weather in Paris?" },
            // 原样回传收到的 assistant 内容。当存在 thinking
            // 块时，它必须与 tool_use 块一同提供。
            { role: "assistant", content: response.content },
            {
              role: "user",
              content: [
                {
                  type: "tool_result" as const,
                  tool_use_id: toolUseBlock.id,
                  content: `Current temperature: ${weatherData.temperature}°F`
                }
              ]
            }
          ]
        });
        console.log(continuation);
      }
      ```

      ```csharp C#
      AnthropicClient client = new();

      var weatherTool = new ToolUnion(new Tool()
      {
          Name = "get_weather",
          Description = "Get current weather for a location",
          InputSchema = new InputSchema()
          {
              Properties = new Dictionary<string, JsonElement>
              {
                  ["location"] = JsonSerializer.SerializeToElement(new { type = "string", description = "City name" }),
              },
              Required = ["location"],
          },
      });

      var parameters = new MessageCreateParams
      {
          Model = Model.ClaudeOpus4_8,
          MaxTokens = 16000,
          Thinking = new ThinkingConfigAdaptive(),
          Tools = [weatherTool],
          Messages = [
              new() { Role = Role.User, Content = "What's the weather in Paris?" }
          ]
      };

      var response = await client.Messages.Create(parameters);

      // 提取 tool_use 块以获取其 ID，用于构建工具结果
      ToolUseBlock? toolUseBlock = null;
      foreach (var block in response.Content)
      {
          if (block.TryPickToolUse(out var toolUse))
          {
              toolUseBlock = toolUse;
              break;
          }
      }

      var weatherData = new { temperature = 88 };

      // 使用工具结果构建后续请求
      var continuationParams = new MessageCreateParams
      {
          Model = Model.ClaudeOpus4_8,
          MaxTokens = 16000,
          Thinking = new ThinkingConfigAdaptive(),
          Tools = [weatherTool],
          Messages = [
              new() { Role = Role.User, Content = "What's the weather in Paris?" },
              // response.Content 包含思考块；必须将它们原样传回
              new() { Role = Role.Assistant, Content = response.Content.Select(block => new ContentBlockParam(block.Json)).ToList() },
              new() { Role = Role.User, Content = new MessageParamContent(new List<ContentBlockParam>
              {
                  new ContentBlockParam(new ToolResultBlockParam()
                  {
                      ToolUseID = toolUseBlock?.ID ?? "",
                      Content = $"Current temperature: {weatherData.temperature}°F"
                  })
              })}
          ]
      };

      var continuation = await client.Messages.Create(continuationParams);
      Console.WriteLine(continuation);
      ```

      ```go Go
      client := anthropic.NewClient()

      weatherTool := anthropic.ToolUnionParam{
      	OfTool: &anthropic.ToolParam{
      		Name:        "get_weather",
      		Description: anthropic.String("Get current weather for a location"),
      		InputSchema: anthropic.ToolInputSchemaParam{
      			Properties: map[string]any{
      				"location": map[string]any{
      					"type":        "string",
      					"description": "City name",
      				},
      			},
      			Required: []string{"location"},
      		},
      	},
      }

      response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
      	Model:     anthropic.ModelClaudeOpus4_8,
      	MaxTokens: 16000,
      	Thinking: anthropic.ThinkingConfigParamUnion{
      		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
      	},
      	Tools: []anthropic.ToolUnionParam{weatherTool},
      	Messages: []anthropic.MessageParam{
      		anthropic.NewUserMessage(anthropic.NewTextBlock("What's the weather in Paris?")),
      	},
      })
      if err != nil {
      	log.Fatal(err)
      }

      var toolUseBlock anthropic.ToolUseBlock
      for _, block := range response.Content {
      	if v, ok := block.AsAny().(anthropic.ToolUseBlock); ok {
      		toolUseBlock = v
      		break
      	}
      }

      weatherData := map[string]int{"temperature": 88}

      continuation, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
      	Model:     anthropic.ModelClaudeOpus4_8,
      	MaxTokens: 16000,
      	Thinking: anthropic.ThinkingConfigParamUnion{
      		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
      	},
      	Tools: []anthropic.ToolUnionParam{weatherTool},
      	Messages: []anthropic.MessageParam{
      		anthropic.NewUserMessage(anthropic.NewTextBlock("What's the weather in Paris?")),
      		response.ToParam(),
      		anthropic.NewUserMessage(
      			anthropic.NewToolResultBlock(toolUseBlock.ID, fmt.Sprintf("Current temperature: %d°F", weatherData["temperature"]), false),
      		),
      	},
      })
      if err != nil {
      	log.Fatal(err)
      }

      fmt.Println(continuation)
      ```

      ```java Java
      import com.anthropic.models.messages.ThinkingConfigAdaptive;
      // ...

      void main() {
          AnthropicClient client = AnthropicOkHttpClient.fromEnv();

          Tool weatherTool = Tool.builder()
              .name("get_weather")
              .description("Get current weather for a location")
              .inputSchema(Tool.InputSchema.builder()
                  .properties(JsonValue.from(Map.of(
                      "location", Map.of("type", "string", "description", "City name")
                  )))
                  .required(List.of("location"))
                  .build())
              .build();

          MessageCreateParams initialParams = MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_4_8)
              .maxTokens(16000L)
              .thinking(ThinkingConfigAdaptive.builder().build())
              .addTool(weatherTool)
              .addUserMessage("What's the weather in Paris?")
              .build();

          Message response = client.messages().create(initialParams);

          ToolUseBlock toolUseBlock = null;
          for (var block : response.content()) {
              if (block.toolUse().isPresent()) {
                  toolUseBlock = block.toolUse().get();
                  break;
              }
          }

          int temperature = 88;

          // 第二次请求：原样回传收到的 assistant 轮次，然后附上工具结果
          MessageCreateParams continuationParams = MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_4_8)
              .maxTokens(16000L)
              .thinking(ThinkingConfigAdaptive.builder().build())
              .addTool(weatherTool)
              .addUserMessage("What's the weather in Paris?")
              .addMessage(response)
              .addUserMessageOfBlockParams(List.of(
                  ContentBlockParam.ofToolResult(
                      ToolResultBlockParam.builder()
                          .toolUseId(toolUseBlock.id())
                          .content("Current temperature: " + temperature + "°F")
                          .build()
                  )
              ))
              .build();

          Message continuation = client.messages().create(continuationParams);
          IO.println(continuation);
      }
      ```

      ```php PHP
      $client = new Client();

      $weatherTool = [
          'name' => 'get_weather',
          'description' => 'Get current weather for a location',
          'input_schema' => [
              'type' => 'object',
              'properties' => [
                  'location' => [
                      'type' => 'string',
                      'description' => 'City name'
                  ]
              ],
              'required' => ['location']
          ]
      ];

      $response = $client->messages->create(
          maxTokens: 16000,
          messages: [
              ['role' => 'user', 'content' => "What's the weather in Paris?"]
          ],
          model: 'claude-opus-4-8',
          thinking: ['type' => 'adaptive'],
          tools: [$weatherTool],
      );

      $toolUseBlock = null;
      foreach ($response->content as $block) {
          if ($block->type === 'tool_use') {
              $toolUseBlock = $block;
              break;
          }
      }

      $weatherData = ['temperature' => 88];

      $continuation = $client->messages->create(
          maxTokens: 16000,
          messages: [
              ['role' => 'user', 'content' => "What's the weather in Paris?"],
              ['role' => 'assistant', 'content' => $response->content],
              ['role' => 'user', 'content' => [
                  [
                      'type' => 'tool_result',
                      'tool_use_id' => $toolUseBlock->id,
                      'content' => "Current temperature: {$weatherData['temperature']}°F"
                  ]
              ]]
          ],
          model: 'claude-opus-4-8',
          thinking: ['type' => 'adaptive'],
          tools: [$weatherTool],
      );

      echo $continuation;
      ```

      ```ruby Ruby
      client = Anthropic::Client.new

      weather_tool = {
        name: "get_weather",
        description: "Get current weather for a location",
        input_schema: {
          type: "object",
          properties: {
            location: { type: "string", description: "City name" }
          },
          required: ["location"]
        }
      }

      response = client.messages.create(
        model: "claude-opus-4-8",
        max_tokens: 16000,
        thinking: {
          type: "adaptive"
        },
        tools: [weather_tool],
        messages: [
          { role: "user", content: "What's the weather in Paris?" }
        ]
      )

      tool_use_block = response.content.find { |block| block.type == :tool_use }

      raise "No tool_use block found" unless tool_use_block

      weather_data = { temperature: 88 }

      continuation = client.messages.create(
        model: "claude-opus-4-8",
        max_tokens: 16000,
        thinking: {
          type: "adaptive"
        },
        tools: [weather_tool],
        messages: [
          { role: "user", content: "What's the weather in Paris?" },
          { role: "assistant", content: response.content },
          { role: "user", content: [
            {
              type: "tool_result",
              tool_use_id: tool_use_block.id,
              content: "Current temperature: #{weather_data[:temperature]}°F"
            }
          ] }
        ]
      )

      puts continuation
      ```
    </CodeGroup>
  </Step>

  <Step title="读取最终响应">
    您应该会看到 Claude 以文本完成该轮次。由于 interleaved thinking（交错思考）在自适应模式下是自动的，因此续写内容也可能在最终文本之前以一个新的思考块开头，详见[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)：

    ```json Output
    {
      "content": [
        {
          "type": "text",
          "text": "Currently in Paris, the temperature is 88°F (31°C)"
        }
      ]
    }
    ```
  </Step>
</Steps>

## 交错思考如何改变流程

交错思考让 Claude 能够在工具调用之间进行思考，在对每个工具结果采取行动之前先对其进行推理。其概念和各模型的可用性在思考页面的[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)部分有介绍；交错改变的是思考块出现的位置，而不是工具调用能否链式进行。以下对比展示了交错思考在双工具工作流中带来的变化：

<AccordionGroup>
  <Accordion title="不使用交错思考的工具使用">
    在没有交错思考的情况下，Claude 在助手轮次开始时思考一次。工具结果之后的后续响应会继续进行，而不会产生新的思考块。

    ```text
    User: "What's the total revenue if we sold 150 units at $50 each,
           and how does this compare to our average monthly revenue?"

    Response 1: [thinking] "I need to calculate 150 * $50, then check the database..."
                [tool_use: calculator] { "expression": "150 * 50" }
      ↓ tool result: "7500"

    Response 2: [tool_use: database_query] { "query": "SELECT AVG(revenue)..." }
                ↑ no thinking block
      ↓ tool result: "5200"

    Response 3: [text] "The total revenue is $7,500, which is 44% above your
                average monthly revenue of $5,200."
                ↑ no thinking block
    ```
  </Accordion>

  <Accordion title="使用交错思考的工具使用">
    启用交错思考后，Claude 可以在收到每个工具结果后进行思考，从而能够在继续之前对中间结果进行推理。

    ```text
    User: "What's the total revenue if we sold 150 units at $50 each,
           and how does this compare to our average monthly revenue?"

    Response 1: [thinking] "I need to calculate 150 * $50 first..."
                [tool_use: calculator] { "expression": "150 * 50" }
      ↓ tool result: "7500"

    Response 2: [thinking] "Got $7,500. Now I should query the database to compare..."
                [tool_use: database_query] { "query": "SELECT AVG(revenue)..." }
                ↑ thinking after receiving calculator result
      ↓ tool result: "5200"

    Response 3: [thinking] "$7,500 vs $5,200 average - that's a 44% increase..."
                [text] "The total revenue is $7,500, which is 44% above your
                average monthly revenue of $5,200."
                ↑ thinking before final answer
    ```
  </Accordion>
</AccordionGroup>

## 后续步骤

<CardGroup cols={3}>
  <Card title="思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    概述：开启思考、读取思考输出，并查看有关工具使用、缓存和流式传输的完整规则。
  </Card>

  <Card title="引导思考" icon="compass" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost">
    通过努力级别和基于提示的指导，引导 Claude 思考的频率和深度。
  </Card>

  <Card title="扩展思考" icon="clock" href="https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking">
    旧模型上的手动思考预算：`budget_tokens` 机制以及向自适应思考的迁移。
  </Card>
</CardGroup>
