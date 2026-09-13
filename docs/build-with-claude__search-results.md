---
title: 搜索结果
url: https://platform.claude.com/docs/zh-CN/build-with-claude/search-results
description: 通过提供带有来源归属的搜索结果，为 RAG 应用启用自然引用
---

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

搜索结果内容块让 Claude 能够以引用网络搜索结果的相同方式引用您自己的内容：每条引用都携带您提供的来源和标题。请在 RAG（"Retrieval-Augmented Generation"，检索增强生成）应用中使用它们，即 Claude 需要将答案归属到您的文档的场景。

所有[活跃模型](https://platform.claude.com/docs/zh-CN/models/overview)都支持带引用的搜索结果，Claude Haiku 3 除外。无需 beta 标头：搜索结果是标准 Messages API 的一部分。

## 工作原理

搜索结果可以通过两种方式提供：

1. **来自工具调用：** 您的自定义工具返回搜索结果，从而实现动态 RAG 应用
2. **作为顶层内容：** 您直接在用户消息中提供搜索结果，用于预取或缓存的内容

在这两种情况下，当启用 citations（引用）时，Claude 都会自动引用搜索结果。无需特殊提示：提出您的问题，引用就会出现在借鉴了您内容的文本块上。

### 搜索结果架构

搜索结果使用以下结构：

```json
{
  "type": "search_result",
  "source": "https://example.com/article", // Required: Source URL or identifier
  "title": "Article Title", // Required: Title of the result
  "content": [
    // Required: Array of text blocks
    {
      "type": "text",
      "text": "The actual content of the search result..."
    }
  ],
  "citations": {
    // Optional: Citation configuration
    "enabled": true // Enable/disable citations for this result
  }
}
```

### 必填字段

| 字段        | 类型     | 描述                                                 |
| --------- | ------ | -------------------------------------------------- |
| `type`    | string | 必须为 `"search_result"`                              |
| `source`  | string | 内容的来源。任何稳定的字符串均可：URL，或内部标识符，例如 `kb://article-1234` |
| `title`   | string | 搜索结果的描述性标题                                         |
| `content` | array  | 包含实际内容的文本块数组                                       |

### 可选字段

| 字段              | 类型     | 描述                                                                                                                                                                                       |
| --------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `citations`     | object | 带有 `enabled` 布尔字段的引用配置。引用默认禁用；本页的每个示例都显式设置了 `"enabled": true`。一个请求中的所有搜索结果必须使用相同的设置（请参阅[引用控制](https://platform.claude.com/docs/zh-CN/build-with-claude/search-results#citation-control)） |
| `cache_control` | object | 缓存控制设置（例如 `{"type": "ephemeral"}`）                                                                                                                                                       |

`content` 数组中的每一项都必须是一个文本块，包含：

* `type`：必须为 `"text"`
* `text`：实际的文本内容（非空字符串）

搜索结果仅包含文本。`content` 数组内不支持图像和其他媒体。

## 方法 1：来自工具调用的搜索结果

从您的自定义工具返回搜索结果可实现动态 RAG 应用：工具在运行时获取内容，Claude 在响应中引用它。以下示例使用 [`tool_choice`](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/define-tools#forcing-tool-use) 强制进行工具调用，因此检索步骤每次都会运行。

### 示例：知识库工具

<CodeGroup>
  ```bash cURL
  # 工具调用流程需要应用端的搜索逻辑，
  # 这无法转换为一次性的 shell 命令。完整流程请参阅 SDK 选项卡。
  # 包含搜索结果的工具对话的原始结构展示在
  # “结合两种方法”的 cURL 选项卡中；方法 2 展示了顶层结构。
  ```

  ```bash CLI
  # 工具调用流程需要应用端的搜索逻辑，
  # 这无法转换为一次性的 shell 命令。完整流程请参阅 SDK 选项卡。
  # 包含搜索结果的工具对话的原始结构展示在
  # “结合两种方法”的 cURL 选项卡中；方法 2 展示了顶层结构。
  ```

  ```python Python
  from anthropic.types import (
      MessageParam,
      TextBlockParam,
      SearchResultBlockParam,
      ToolResultBlockParam,
  )

  client = Anthropic()

  # 定义一个知识库搜索工具
  knowledge_base_tool = {
      "name": "search_knowledge_base",
      "description": "Search the company knowledge base for information",
      "input_schema": {
          "type": "object",
          "properties": {"query": {"type": "string", "description": "The search query"}},
          "required": ["query"],
      },
  }


  # 处理工具调用的函数
  def search_knowledge_base(query):
      # 在此处编写您的搜索逻辑
      # 以正确的格式返回搜索结果
      return [
          SearchResultBlockParam(
              type="search_result",
              source="https://docs.company.com/product-guide",
              title="Product Configuration Guide",
              content=[
                  TextBlockParam(
                      type="text",
                      text="To configure the product, navigate to Settings > Configuration. The default timeout is 30 seconds, but can be adjusted between 10-120 seconds based on your needs.",
                  )
              ],
              citations={"enabled": True},
          ),
          SearchResultBlockParam(
              type="search_result",
              source="https://docs.company.com/troubleshooting",
              title="Troubleshooting Guide",
              content=[
                  TextBlockParam(
                      type="text",
                      text="If you encounter timeout errors, first check the configuration settings. Common causes include network latency and incorrect timeout values.",
                  )
              ],
              citations={"enabled": True},
          ),
      ]


  # 以列表形式构建对话，从用户的问题开始
  messages = [
      MessageParam(role="user", content="How do I configure the timeout settings?")
  ]

  # 创建一条带有该工具的消息
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      tools=[knowledge_base_tool],
      tool_choice={"type": "tool", "name": "search_knowledge_base"},
      messages=messages,
  )

  # 当 Claude 调用该工具时，提供搜索结果。
  # tool_use 块并不总是排在第一位：需遍历查找。
  tool_use = next((block for block in response.content if block.type == "tool_use"), None)
  if tool_use is not None:
      tool_result = search_knowledge_base(tool_use.input["query"])

      # 将 Claude 的回合以及工具结果追加到当前对话中
      messages.append(MessageParam(role="assistant", content=response.content))
      messages.append(
          MessageParam(
              role="user",
              content=[
                  ToolResultBlockParam(
                      type="tool_result",
                      tool_use_id=tool_use.id,
                      content=tool_result,  # Search results go here
                  )
              ],
          )
      )

      # 将工具结果发送回去
      final_response = client.messages.create(
          model="claude-opus-5",
          max_tokens=1024,
          messages=messages,
      )
      print(final_response)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // 定义一个知识库搜索工具
  const knowledgeBaseTool: Anthropic.Tool = {
    name: "search_knowledge_base",
    description: "Search the company knowledge base for information",
    input_schema: {
      type: "object" as const,
      properties: {
        query: {
          type: "string",
          description: "The search query"
        }
      },
      required: ["query"]
    }
  };

  // 处理工具调用的函数
  function searchKnowledgeBase(query: string) {
    // 在此处编写您的搜索逻辑
    // 以正确的格式返回搜索结果
    return [
      {
        type: "search_result" as const,
        source: "https://docs.company.com/product-guide",
        title: "Product Configuration Guide",
        content: [
          {
            type: "text" as const,
            text: "To configure the product, navigate to Settings > Configuration. The default timeout is 30 seconds, but can be adjusted between 10-120 seconds based on your needs."
          }
        ],
        citations: { enabled: true }
      },
      {
        type: "search_result" as const,
        source: "https://docs.company.com/troubleshooting",
        title: "Troubleshooting Guide",
        content: [
          {
            type: "text" as const,
            text: "If you encounter timeout errors, first check the configuration settings. Common causes include network latency and incorrect timeout values."
          }
        ],
        citations: { enabled: true }
      }
    ];
  }

  // 以列表形式构建对话，从用户的问题开始
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: "How do I configure the timeout settings?" }
  ];

  // 创建一条带有该工具的消息
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    tools: [knowledgeBaseTool],
    tool_choice: { type: "tool", name: "search_knowledge_base" },
    messages
  });

  // 处理工具使用并提供结果。
  // tool_use 块并不总是位于首位：请在 content 数组中查找它。
  const toolUse = response.content.find(
    (block): block is Anthropic.ToolUseBlock => block.type === "tool_use"
  );
  if (toolUse) {
    const input = toolUse.input as { query: string };
    const toolResult = searchKnowledgeBase(input.query);

    // 将 Claude 的回合以及工具结果依次追加到当前对话中
    messages.push({ role: "assistant", content: response.content });
    messages.push({
      role: "user",
      content: [
        {
          type: "tool_result" as const,
          tool_use_id: toolUse.id,
          content: toolResult // Search results go here
        }
      ]
    });

    // 将工具结果发送回去
    const finalResponse = await client.messages.create({
      model: "claude-opus-5",
      max_tokens: 1024,
      messages
    });
    console.log(finalResponse);
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var tools = new List<ToolUnion>
  {
      new ToolUnion(new Tool()
      {
          Name = "search_knowledge_base",
          Description = "Search the company knowledge base for information",
          InputSchema = new InputSchema()
          {
              Properties = new Dictionary<string, JsonElement>
              {
                  ["query"] = JsonSerializer.SerializeToElement(new { type = "string", description = "The search query" }),
              },
              Required = ["query"],
          },
      }),
  };

  // 处理工具调用的函数
  static List<Block> SearchKnowledgeBase(string query)
  {
      // 在此处编写您的搜索逻辑
      // 以正确的格式返回搜索结果
      return
      [
          new SearchResultBlockParam
          {
              Source = "https://docs.company.com/product-guide",
              Title = "Product Configuration Guide",
              Content = [new() { Text = "To configure the product, navigate to Settings > Configuration. The default timeout is 30 seconds, but can be adjusted between 10-120 seconds based on your needs." }],
              Citations = new() { Enabled = true },
          },
          new SearchResultBlockParam
          {
              Source = "https://docs.company.com/troubleshooting",
              Title = "Troubleshooting Guide",
              Content = [new() { Text = "If you encounter timeout errors, first check the configuration settings. Common causes include network latency and incorrect timeout values." }],
              Citations = new() { Enabled = true },
          },
      ];
  }

  // 以列表形式构建对话，从用户的问题开始
  List<MessageParam> messages = [new() { Role = Role.User, Content = "How do I configure the timeout settings?" }];

  // 创建带有该工具的消息
  var response = await client.Messages.Create(new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Tools = tools,
      ToolChoice = new ToolChoiceTool { Name = "search_knowledge_base" },
      Messages = messages,
  });

  // 当 Claude 调用该工具时，提供搜索结果。
  // tool_use 块并不总是排在第一个：找到第一个 tool_use 块。
  foreach (var block in response.Content)
  {
      if (block.TryPickToolUse(out var toolUse))
      {
          var query = toolUse.Input["query"].GetString() ?? "";
          var toolResults = SearchKnowledgeBase(query);

          // 将 Claude 的回合以及工具结果依次追加到当前对话中
          messages.Add(new() { Role = Role.Assistant, Content = response.Content.Select(contentBlock => new ContentBlockParam(contentBlock.Json)).ToList() });
          messages.Add(new()
          {
              Role = Role.User,
              Content = new MessageParamContent(
                  [new ContentBlockParam(new ToolResultBlockParam() { ToolUseID = toolUse.ID, Content = new ToolResultBlockParamContent(toolResults) })]
              ),
          });

          // 将工具结果发送回去
          var finalResponse = await client.Messages.Create(new()
          {
              Model = Model.ClaudeOpus5,
              MaxTokens = 1024,
              Messages = messages,
          });
          Console.WriteLine(finalResponse);
          break;
      }
  }
  ```

  ```go Go
  	client := anthropic.NewClient()

  	knowledgeBaseTool := anthropic.ToolUnionParam{
  		OfTool: &anthropic.ToolParam{
  			Name:        "search_knowledge_base",
  			Description: anthropic.String("Search the company knowledge base for information"),
  			InputSchema: anthropic.ToolInputSchemaParam{
  				Properties: map[string]any{
  					"query": map[string]any{
  						"type":        "string",
  						"description": "The search query",
  					},
  				},
  				Required: []string{"query"},
  			},
  		},
  	}

  	// 在切片中构建对话，从用户的问题开始
  	messages := []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("How do I configure the timeout settings?")),
  	}

  	response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  		Model:      anthropic.ModelClaudeOpus5,
  		MaxTokens:  1024,
  		Tools:      []anthropic.ToolUnionParam{knowledgeBaseTool},
  		ToolChoice: anthropic.ToolChoiceUnionParam{OfTool: &anthropic.ToolChoiceToolParam{Name: "search_knowledge_base"}},
  		Messages:   messages,
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	// tool_use 块并不总是排在第一位：在 content 列表中查找它
  	var toolUse *anthropic.ToolUseBlock
  	for _, block := range response.Content {
  		if variant, ok := block.AsAny().(anthropic.ToolUseBlock); ok {
  			toolUse = &variant
  			break
  		}
  	}

  	if toolUse != nil {
  		var input struct {
  			Query string `json:"query"`
  		}
  		if err := json.Unmarshal(toolUse.Input, &input); err != nil {
  			log.Fatal(err)
  		}
  		toolResults := searchKnowledgeBase(input.Query)

  		// 将 Claude 的回合以及工具结果追加到正在进行的对话中
  		messages = append(messages, response.ToParam())
  		messages = append(messages, anthropic.NewUserMessage(anthropic.ContentBlockParamUnion{
  			OfToolResult: &anthropic.ToolResultBlockParam{
  				ToolUseID: toolUse.ID,
  				Content:   toolResults,
  			},
  		}))

  		// 将工具结果发送回去
  		finalResponse, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  			Model:     anthropic.ModelClaudeOpus5,
  			MaxTokens: 1024,
  			Messages:  messages,
  		})
  		if err != nil {
  			log.Fatal(err)
  		}
  		fmt.Println(finalResponse)
  	}
  // ...
  func searchKnowledgeBase(query string) []anthropic.ToolResultBlockParamContentUnion {
  	return []anthropic.ToolResultBlockParamContentUnion{
  		{OfSearchResult: &anthropic.SearchResultBlockParam{
  			Content: []anthropic.TextBlockParam{
  				{Text: "To configure the product, navigate to Settings > Configuration. The default timeout is 30 seconds, but can be adjusted between 10-120 seconds based on your needs."},
  			},
  			Source:    "https://docs.company.com/product-guide",
  			Title:     "Product Configuration Guide",
  			Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
  		}},
  		{OfSearchResult: &anthropic.SearchResultBlockParam{
  			Content: []anthropic.TextBlockParam{
  				{Text: "If you encounter timeout errors, first check the configuration settings. Common causes include network latency and incorrect timeout values."},
  			},
  			Source:    "https://docs.company.com/troubleshooting",
  			Title:     "Troubleshooting Guide",
  			Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
  		}},
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.messages.ContentBlockParam;
  import com.anthropic.models.messages.CitationsConfigParam;
  // ...
  import com.anthropic.models.messages.MessageParam;
  import com.anthropic.models.messages.Model;
  import com.anthropic.models.messages.SearchResultBlockParam;
  import com.anthropic.models.messages.TextBlockParam;
  import com.anthropic.models.messages.Tool;
  import com.anthropic.models.messages.ToolChoice;
  import com.anthropic.models.messages.ToolChoiceTool;
  import com.anthropic.models.messages.ToolResultBlockParam;
  import com.anthropic.core.JsonValue;
  // ...

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      Tool knowledgeBaseTool = Tool.builder()
          .name("search_knowledge_base")
          .description("Search the company knowledge base for information")
          .inputSchema(Tool.InputSchema.builder()
              .properties(JsonValue.from(Map.of(
                  "query", Map.of(
                      "type", "string",
                      "description", "The search query"
                  )
              )))
              .putAdditionalProperty("required", JsonValue.from(List.of("query")))
              .build())
          .build();

      // 在列表中构建对话，从用户的问题开始
      List<MessageParam> messages = new ArrayList<>();
      messages.add(MessageParam.builder()
          .role(MessageParam.Role.USER)
          .content("How do I configure the timeout settings?")
          .build());

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024L)
          .addTool(knowledgeBaseTool)
          .toolChoice(ToolChoice.ofTool(ToolChoiceTool.builder()
              .name("search_knowledge_base")
              .build()))
          .messages(messages)
          .build();

      Message response = client.messages().create(params);

      // tool_use 块并不总是排在第一位：在 content 列表中查找它
      response.content().stream()
          .flatMap(contentBlock -> contentBlock.toolUse().stream())
          .findFirst()
          .ifPresent(toolUse -> {
              Map<String, JsonValue> input =
                  (Map<String, JsonValue>) toolUse._input().asObject().get();
              List<ToolResultBlockParam.Content.Block> toolResult = searchKnowledgeBase(
                  input.get("query").asStringOrThrow()
              );

              // 将 Claude 的完整回合追加到当前对话中，然后追加工具结果。
              // 仅重建 tool_use 块会丢失 Claude 返回的其他内容块
              // （例如未强制工具调用时的前置文本）——因此应追加
              // 完整回合，与其他语言标签页的做法一致。
              messages.add(MessageParam.builder()
                  .role(MessageParam.Role.ASSISTANT)
                  .contentOfBlockParams(
                      response.content().stream()
                          .map(block -> block.toParam())
                          .toList()
                  )
                  .build());
              messages.add(MessageParam.builder()
                  .role(MessageParam.Role.USER)
                  .contentOfBlockParams(List.of(
                      ContentBlockParam.ofToolResult(
                          ToolResultBlockParam.builder()
                              .toolUseId(toolUse.id())
                              .contentOfBlocks(toolResult)
                              .build()
                      )
                  ))
                  .build());

              // 将工具结果发送回去
              MessageCreateParams finalParams = MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5)
                  .maxTokens(1024L)
                  .messages(messages)
                  .build();

              Message finalResponse = client.messages().create(finalParams);
              System.out.println(finalResponse);
          });
  }

  static List<ToolResultBlockParam.Content.Block> searchKnowledgeBase(String query) {
      return List.of(
          ToolResultBlockParam.Content.Block.ofSearchResult(
              SearchResultBlockParam.builder()
                  .source("https://docs.company.com/product-guide")
                  .title("Product Configuration Guide")
                  .content(List.of(
                      TextBlockParam.builder()
                          .text("To configure the product, navigate to Settings > Configuration. The default timeout is 30 seconds, but can be adjusted between 10-120 seconds based on your needs.")
                          .build()
                  ))
                  .citations(CitationsConfigParam.builder().enabled(true).build())
                  .build()
          ),
          ToolResultBlockParam.Content.Block.ofSearchResult(
              SearchResultBlockParam.builder()
                  .source("https://docs.company.com/troubleshooting")
                  .title("Troubleshooting Guide")
                  .content(List.of(
                      TextBlockParam.builder()
                          .text("If you encounter timeout errors, first check the configuration settings. Common causes include network latency and incorrect timeout values.")
                          .build()
                  ))
                  .citations(CitationsConfigParam.builder().enabled(true).build())
                  .build()
          )
      );
  }
  ```

  ```php PHP
  $client = new Client();

  $knowledgeBaseTool = [
      'name' => 'search_knowledge_base',
      'description' => 'Search the company knowledge base for information',
      'input_schema' => [
          'type' => 'object',
          'properties' => [
              'query' => [
                  'type' => 'string',
                  'description' => 'The search query'
              ]
          ],
          'required' => ['query']
      ]
  ];

  function searchKnowledgeBase($query) {
      return [
          [
              'type' => 'search_result',
              'source' => 'https://docs.company.com/product-guide',
              'title' => 'Product Configuration Guide',
              'content' => [
                  [
                      'type' => 'text',
                      'text' => 'To configure the product, navigate to Settings > Configuration. The default timeout is 30 seconds, but can be adjusted between 10-120 seconds based on your needs.'
                  ]
              ],
              'citations' => ['enabled' => true]
          ],
          [
              'type' => 'search_result',
              'source' => 'https://docs.company.com/troubleshooting',
              'title' => 'Troubleshooting Guide',
              'content' => [
                  [
                      'type' => 'text',
                      'text' => 'If you encounter timeout errors, first check the configuration settings. Common causes include network latency and incorrect timeout values.'
                  ]
              ],
              'citations' => ['enabled' => true]
          ]
      ];
  }

  // 在列表中构建对话，从用户的问题开始
  $messages = [
      ['role' => 'user', 'content' => 'How do I configure the timeout settings?']
  ];

  $response = $client->messages->create(
      maxTokens: 1024,
      messages: $messages,
      model: 'claude-opus-5',
      toolChoice: ['type' => 'tool', 'name' => 'search_knowledge_base'],
      tools: [$knowledgeBaseTool],
  );

  $toolUseBlock = null;
  foreach ($response->content as $block) {
      if ($block->type === 'tool_use') {
          $toolUseBlock = $block;
          break;
      }
  }

  if ($toolUseBlock !== null) {
      $toolResult = searchKnowledgeBase($toolUseBlock->input['query']);

      // 将 Claude 的回合及工具结果依次追加到当前对话中
      $messages[] = ['role' => 'assistant', 'content' => $response->content];
      $messages[] = [
          'role' => 'user',
          'content' => [
              [
                  'type' => 'tool_result',
                  'tool_use_id' => $toolUseBlock->id,
                  'content' => $toolResult
              ]
          ]
      ];

      // 将工具结果发送回去
      $finalResponse = $client->messages->create(
          maxTokens: 1024,
          messages: $messages,
          model: 'claude-opus-5',
      );
      echo $finalResponse;
  } else {
      echo $response;
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  knowledge_base_tool = {
    name: "search_knowledge_base",
    description: "Search the company knowledge base for information",
    input_schema: {
      type: "object",
      properties: {
        query: { type: "string", description: "The search query" }
      },
      required: ["query"]
    }
  }

  def search_knowledge_base(query)
    [
      {
        type: "search_result",
        source: "https://docs.company.com/product-guide",
        title: "Product Configuration Guide",
        content: [
          {
            type: "text",
            text: "To configure the product, navigate to Settings > Configuration. The default timeout is 30 seconds, but can be adjusted between 10-120 seconds based on your needs."
          }
        ],
        citations: { enabled: true }
      },
      {
        type: "search_result",
        source: "https://docs.company.com/troubleshooting",
        title: "Troubleshooting Guide",
        content: [
          {
            type: "text",
            text: "If you encounter timeout errors, first check the configuration settings. Common causes include network latency and incorrect timeout values."
          }
        ],
        citations: { enabled: true }
      }
    ]
  end

  # 在列表中构建对话，从用户的问题开始
  messages = [
    { role: "user", content: "How do I configure the timeout settings?" }
  ]

  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    tools: [knowledge_base_tool],
    tool_choice: { type: "tool", name: "search_knowledge_base" },
    messages: messages
  )

  # tool_use 块并不总是排在第一位：在 content 数组中查找它
  tool_use = response.content.find { |block| block.type == :tool_use }

  if tool_use
    tool_result = search_knowledge_base(tool_use.input[:query])

    # 将 Claude 的回合以及工具结果追加到正在进行的对话中
    messages << { role: "assistant", content: response.content }
    messages << {
      role: "user",
      content: [
        {
          type: "tool_result",
          tool_use_id: tool_use.id,
          content: tool_result
        }
      ]
    }

    # 将工具结果发送回去
    final_response = client.messages.create(
      model: "claude-opus-5",
      max_tokens: 1024,
      messages: messages
    )
    puts final_response
  end
  ```
</CodeGroup>

## 方法 2：作为顶层内容的搜索结果

您也可以直接在用户消息中提供搜索结果。这适用于：

* 来自您的搜索基础设施的预取内容
* 来自先前查询的缓存搜索结果
* 来自外部搜索服务的内容
* 测试和开发

### 示例：直接搜索结果

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [
        {
          "role": "user",
          "content": [
            {
              "type": "search_result",
              "source": "https://docs.company.com/api-reference",
              "title": "API Reference - Authentication",
              "content": [
                {
                  "type": "text",
                  "text": "All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard. Rate limits: 1000 requests per hour for standard tier, 10000 for premium."
                }
              ],
              "citations": {
                "enabled": true
              }
            },
            {
              "type": "search_result",
              "source": "https://docs.company.com/quickstart",
              "title": "Getting Started Guide",
              "content": [
                {
                  "type": "text",
                  "text": "To get started: 1) Sign up for an account, 2) Generate an API key from the dashboard, 3) Install our SDK using pip install company-sdk, 4) Initialize the client with your API key."
                }
              ],
              "citations": {
                "enabled": true
              }
            },
            {
              "type": "text",
              "text": "Based on these search results, how do I authenticate API requests and what are the rate limits?"
            }
          ]
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: search_result
          source: https://docs.company.com/api-reference
          title: API Reference - Authentication
          content:
            - type: text
              text: >-
                All API requests must include an API key in the Authorization
                header. Keys can be generated from the dashboard. Rate limits:
                1000 requests per hour for standard tier, 10000 for premium.
          citations:
            enabled: true
        - type: search_result
          source: https://docs.company.com/quickstart
          title: Getting Started Guide
          content:
            - type: text
              text: >-
                To get started: 1) Sign up for an account, 2) Generate an API
                key from the dashboard, 3) Install our SDK using pip install
                company-sdk, 4) Initialize the client with your API key.
          citations:
            enabled: true
        - type: text
          text: >-
            Based on these search results, how do I authenticate API requests
            and what are the rate limits?
  YAML
  ```

  ```python Python
  from anthropic.types import MessageParam, TextBlockParam, SearchResultBlockParam

  client = Anthropic()

  # 直接在用户消息中提供搜索结果
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          MessageParam(
              role="user",
              content=[
                  SearchResultBlockParam(
                      type="search_result",
                      source="https://docs.company.com/api-reference",
                      title="API Reference - Authentication",
                      content=[
                          TextBlockParam(
                              type="text",
                              text="All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard. Rate limits: 1000 requests per hour for standard tier, 10000 for premium.",
                          )
                      ],
                      citations={"enabled": True},
                  ),
                  SearchResultBlockParam(
                      type="search_result",
                      source="https://docs.company.com/quickstart",
                      title="Getting Started Guide",
                      content=[
                          TextBlockParam(
                              type="text",
                              text="To get started: 1) Sign up for an account, 2) Generate an API key from the dashboard, 3) Install our SDK using pip install company-sdk, 4) Initialize the client with your API key.",
                          )
                      ],
                      citations={"enabled": True},
                  ),
                  TextBlockParam(
                      type="text",
                      text="Based on these search results, how do I authenticate API requests and what are the rate limits?",
                  ),
              ],
          )
      ],
  )

  print(response)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // 直接在用户消息中提供搜索结果
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "search_result" as const,
            source: "https://docs.company.com/api-reference",
            title: "API Reference - Authentication",
            content: [
              {
                type: "text" as const,
                text: "All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard. Rate limits: 1000 requests per hour for standard tier, 10000 for premium."
              }
            ],
            citations: { enabled: true }
          },
          {
            type: "search_result" as const,
            source: "https://docs.company.com/quickstart",
            title: "Getting Started Guide",
            content: [
              {
                type: "text" as const,
                text: "To get started: 1) Sign up for an account, 2) Generate an API key from the dashboard, 3) Install our SDK using pip install company-sdk, 4) Initialize the client with your API key."
              }
            ],
            citations: { enabled: true }
          },
          {
            type: "text" as const,
            text: "Based on these search results, how do I authenticate API requests and what are the rate limits?"
          }
        ]
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  AnthropicClient client = new();

  // 直接在用户消息中提供搜索结果
  var response = await client.Messages.Create(new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = new MessageParamContent(
              [
                  new ContentBlockParam(new SearchResultBlockParam
                  {
                      Source = "https://docs.company.com/api-reference",
                      Title = "API Reference - Authentication",
                      Content = [new() { Text = "All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard. Rate limits: 1000 requests per hour for standard tier, 10000 for premium." }],
                      Citations = new() { Enabled = true },
                  }),
                  new ContentBlockParam(new SearchResultBlockParam
                  {
                      Source = "https://docs.company.com/quickstart",
                      Title = "Getting Started Guide",
                      Content = [new() { Text = "To get started: 1) Sign up for an account, 2) Generate an API key from the dashboard, 3) Install our SDK using pip install company-sdk, 4) Initialize the client with your API key." }],
                      Citations = new() { Enabled = true },
                  }),
                  new ContentBlockParam(new TextBlockParam { Text = "Based on these search results, how do I authenticate API requests and what are the rate limits?" }),
              ]),
          },
      ],
  });

  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.ContentBlockParamUnion{OfSearchResult: &anthropic.SearchResultBlockParam{
  				Content: []anthropic.TextBlockParam{
  					{Text: "All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard. Rate limits: 1000 requests per hour for standard tier, 10000 for premium."},
  				},
  				Source:    "https://docs.company.com/api-reference",
  				Title:     "API Reference - Authentication",
  				Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
  			}},
  			anthropic.ContentBlockParamUnion{OfSearchResult: &anthropic.SearchResultBlockParam{
  				Content: []anthropic.TextBlockParam{
  					{Text: "To get started: 1) Sign up for an account, 2) Generate an API key from the dashboard, 3) Install our SDK using pip install company-sdk, 4) Initialize the client with your API key."},
  				},
  				Source:    "https://docs.company.com/quickstart",
  				Title:     "Getting Started Guide",
  				Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
  			}},
  			anthropic.NewTextBlock("Based on these search results, how do I authenticate API requests and what are the rate limits?"),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.messages.ContentBlockParam;
  import com.anthropic.models.messages.CitationsConfigParam;
  // ...
  import com.anthropic.models.messages.SearchResultBlockParam;
  import com.anthropic.models.messages.TextBlockParam;
  // ...

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024L)
          .addUserMessageOfBlockParams(List.of(
              ContentBlockParam.ofSearchResult(
                  SearchResultBlockParam.builder()
                      .source("https://docs.company.com/api-reference")
                      .title("API Reference - Authentication")
                      .content(List.of(
                          TextBlockParam.builder()
                              .text("All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard. Rate limits: 1000 requests per hour for standard tier, 10000 for premium.")
                              .build()
                      ))
                      .citations(CitationsConfigParam.builder().enabled(true).build())
                      .build()
              ),
              ContentBlockParam.ofSearchResult(
                  SearchResultBlockParam.builder()
                      .source("https://docs.company.com/quickstart")
                      .title("Getting Started Guide")
                      .content(List.of(
                          TextBlockParam.builder()
                              .text("To get started: 1) Sign up for an account, 2) Generate an API key from the dashboard, 3) Install our SDK using pip install company-sdk, 4) Initialize the client with your API key.")
                              .build()
                      ))
                      .citations(CitationsConfigParam.builder().enabled(true).build())
                      .build()
              ),
              ContentBlockParam.ofText(
                  TextBlockParam.builder()
                      .text("Based on these search results, how do I authenticate API requests and what are the rate limits?")
                      .build()
              )
          ))
          .build();

      Message response = client.messages().create(params);
      System.out.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'search_result',
                      'source' => 'https://docs.company.com/api-reference',
                      'title' => 'API Reference - Authentication',
                      'content' => [
                          [
                              'type' => 'text',
                              'text' => 'All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard. Rate limits: 1000 requests per hour for standard tier, 10000 for premium.'
                          ]
                      ],
                      'citations' => ['enabled' => true]
                  ],
                  [
                      'type' => 'search_result',
                      'source' => 'https://docs.company.com/quickstart',
                      'title' => 'Getting Started Guide',
                      'content' => [
                          [
                              'type' => 'text',
                              'text' => 'To get started: 1) Sign up for an account, 2) Generate an API key from the dashboard, 3) Install our SDK using pip install company-sdk, 4) Initialize the client with your API key.'
                          ]
                      ],
                      'citations' => ['enabled' => true]
                  ],
                  [
                      'type' => 'text',
                      'text' => 'Based on these search results, how do I authenticate API requests and what are the rate limits?'
                  ]
              ]
          ]
      ],
      model: 'claude-opus-5',
  );

  echo json_encode($message, JSON_PRETTY_PRINT);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "search_result",
            source: "https://docs.company.com/api-reference",
            title: "API Reference - Authentication",
            content: [
              {
                type: "text",
                text: "All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard. Rate limits: 1000 requests per hour for standard tier, 10000 for premium."
              }
            ],
            citations: { enabled: true }
          },
          {
            type: "search_result",
            source: "https://docs.company.com/quickstart",
            title: "Getting Started Guide",
            content: [
              {
                type: "text",
                text: "To get started: 1) Sign up for an account, 2) Generate an API key from the dashboard, 3) Install our SDK using pip install company-sdk, 4) Initialize the client with your API key."
              }
            ],
            citations: { enabled: true }
          },
          {
            type: "text",
            text: "Based on these search results, how do I authenticate API requests and what are the rate limits?"
          }
        ]
      }
    ]
  )

  puts message
  ```
</CodeGroup>

## Claude 带引用的响应

无论搜索结果以何种方式提供，Claude 在使用其中的信息时都会自动包含引用：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard.",
      "citations": [
        {
          "type": "search_result_location",
          "cited_text": "All API requests must include an API key in the Authorization header. Keys can be generated from the dashboard. Rate limits: 1000 requests per hour for standard tier, 10000 for premium.",
          "source": "https://docs.company.com/api-reference",
          "title": "API Reference - Authentication",
          "search_result_index": 0,
          "start_block_index": 0,
          "end_block_index": 1
        }
      ]
    },
    {
      "type": "text",
      "text": "\n\nTo set this up from scratch, you'll need to "
    },
    {
      "type": "text",
      "text": "sign up for an account, generate an API key from the dashboard, install the SDK using `pip install company-sdk`, and initialize the client with your API key.",
      "citations": [
        {
          "type": "search_result_location",
          "cited_text": "To get started: 1) Sign up for an account, 2) Generate an API key from the dashboard, 3) Install our SDK using pip install company-sdk, 4) Initialize the client with your API key.",
          "source": "https://docs.company.com/quickstart",
          "title": "Getting Started Guide",
          "search_result_index": 1,
          "start_block_index": 0,
          "end_block_index": 1
        }
      ]
    }
  ]
}
```

### 引用字段

每条引用包含：

| 字段                    | 类型            | 描述                                                                               |
| --------------------- | ------------- | -------------------------------------------------------------------------------- |
| `type`                | string        | 对于搜索结果引用，始终为 `"search_result_location"`                                          |
| `source`              | string        | 来自原始搜索结果的来源                                                                      |
| `title`               | string 或 null | 来自原始搜索结果的标题                                                                      |
| `cited_text`          | string        | 被引用块的完整文本，拼接而成。等于 `content[start_block_index:end_block_index]` 的内容连接在一起。不计入输出令牌。 |
| `search_result_index` | integer       | 被引用的搜索结果在请求中所有 `search_result` 块中的从 0 开始的索引，按其出现顺序计算（跨所有消息和工具结果）。                |
| `start_block_index`   | integer       | 搜索结果 `content` 数组中第一个被引用块的从 0 开始的索引。                                             |
| `end_block_index`     | integer       | 搜索结果 `content` 数组中被引用块范围的不包含在内的结束索引。始终大于 `start_block_index`。                    |

块索引标识搜索结果 `content` 数组的一个切片，而 `cited_text` 是该切片的完整文本。文本块是最小的可引用单元：Claude 引用整个块，而不是块内的子字符串。要获得更细粒度的引用，请将您的搜索结果内容拆分为更小的块（请参阅[多个内容块](https://platform.claude.com/docs/zh-CN/build-with-claude/search-results#multiple-content-blocks)）。

## 多个内容块

搜索结果可以在 `content` 数组中包含多个文本块：

```json
{
  "type": "search_result",
  "source": "https://docs.company.com/api-guide",
  "title": "API Documentation",
  "content": [
    {
      "type": "text",
      "text": "Authentication: All API requests require an API key."
    },
    {
      "type": "text",
      "text": "Rate Limits: The API allows 1000 requests per hour per key."
    },
    {
      "type": "text",
      "text": "Error Handling: The API returns standard HTTP status codes."
    }
  ],
  "citations": { "enabled": true }
}
```

引用速率限制块的引用如下所示：

```json
{
  "type": "search_result_location",
  "cited_text": "Rate Limits: The API allows 1000 requests per hour per key.",
  "source": "https://docs.company.com/api-guide",
  "title": "API Documentation",
  "search_result_index": 0,
  "start_block_index": 1,
  "end_block_index": 2
}
```

当此搜索结果被引用时，`start_block_index` 和 `end_block_index` 标识引用覆盖了其中哪些块，而 `cited_text` 恰好包含这些块的文本。将内容拆分为更小、更聚焦的块可为 Claude 提供更精细的引用边界；将内容合并为一个块则意味着每条引用都会返回完整文本。这与 Citations 功能中[自定义内容文档](https://platform.claude.com/docs/zh-CN/build-with-claude/citations#custom-content-documents)所使用的模型相同。

## 高级用法

### 结合两种方法

您可以在同一对话中混合使用两种方法。Claude 会从任一来源进行引用，并且 `search_result_index` 按请求顺序统计所有 `search_result` 块，无论其来源如何。

以下示例重放了一段完整的对话。第一条用户消息携带一个预取的搜索结果，助手轮次调用一个知识库工具，工具结果返回第二个搜索结果。Claude 的回答引用了这两个来源：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "tools": [
        {
          "name": "search_knowledge_base",
          "description": "Search the company knowledge base for information",
          "input_schema": {
            "type": "object",
            "properties": {
              "query": {"type": "string", "description": "The search query"}
            },
            "required": ["query"]
          }
        }
      ],
      "messages": [
        {
          "role": "user",
          "content": [
            {
              "type": "search_result",
              "source": "https://docs.company.com/overview",
              "title": "Product Overview",
              "content": [
                {
                  "type": "text",
                  "text": "Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards."
                }
              ],
              "citations": {"enabled": true}
            },
            {
              "type": "text",
              "text": "What does Acme Dashboard do, and what plans is it available on?"
            }
          ]
        },
        {
          "role": "assistant",
          "content": [
            {
              "type": "text",
              "text": "Let me check the pricing information."
            },
            {
              "type": "tool_use",
              "id": "toolu_01A09q90qw90lq917835lq9",
              "name": "search_knowledge_base",
              "input": {"query": "Acme Dashboard pricing plans"}
            }
          ]
        },
        {
          "role": "user",
          "content": [
            {
              "type": "tool_result",
              "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
              "content": [
                {
                  "type": "search_result",
                  "source": "https://docs.company.com/pricing",
                  "title": "Pricing Plans",
                  "content": [
                    {
                      "type": "text",
                      "text": "Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing."
                    }
                  ],
                  "citations": {"enabled": true}
                }
              ]
            }
          ]
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  tools:
    - name: search_knowledge_base
      description: Search the company knowledge base for information
      input_schema:
        type: object
        properties:
          query:
            type: string
            description: The search query
        required: [query]
  messages:
    - role: user
      content:
        - type: search_result
          source: https://docs.company.com/overview
          title: Product Overview
          content:
            - type: text
              text: >-
                Acme Dashboard is a monitoring tool for distributed systems.
                It supports real-time alerting and custom metric dashboards.
          citations:
            enabled: true
        - type: text
          text: What does Acme Dashboard do, and what plans is it available on?
    - role: assistant
      content:
        - type: text
          text: Let me check the pricing information.
        - type: tool_use
          id: toolu_01A09q90qw90lq917835lq9
          name: search_knowledge_base
          input:
            query: Acme Dashboard pricing plans
    - role: user
      content:
        - type: tool_result
          tool_use_id: toolu_01A09q90qw90lq917835lq9
          content:
            - type: search_result
              source: https://docs.company.com/pricing
              title: Pricing Plans
              content:
                - type: text
                  text: >-
                    Acme Dashboard is available on the Starter plan at $10 per
                    user per month and the Enterprise plan with custom pricing.
              citations:
                enabled: true
  YAML
  ```

  ```python Python
  from anthropic.types import (
      MessageParam,
      SearchResultBlockParam,
      TextBlockParam,
      ToolResultBlockParam,
      ToolUseBlockParam,
  )

  client = Anthropic()

  knowledge_base_tool = {
      "name": "search_knowledge_base",
      "description": "Search the company knowledge base for information",
      "input_schema": {
          "type": "object",
          "properties": {"query": {"type": "string", "description": "The search query"}},
          "required": ["query"],
      },
  }

  # 重放一段以两种方式提供搜索结果的对话：第一条
  # 用户消息携带预先获取的结果，工具结果返回另一个结果
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      tools=[knowledge_base_tool],
      messages=[
          MessageParam(
              role="user",
              content=[
                  SearchResultBlockParam(
                      type="search_result",
                      source="https://docs.company.com/overview",
                      title="Product Overview",
                      content=[
                          TextBlockParam(
                              type="text",
                              text="Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards.",
                          )
                      ],
                      citations={"enabled": True},
                  ),
                  TextBlockParam(
                      type="text",
                      text="What does Acme Dashboard do, and what plans is it available on?",
                  ),
              ],
          ),
          MessageParam(
              role="assistant",
              content=[
                  TextBlockParam(
                      type="text", text="Let me check the pricing information."
                  ),
                  ToolUseBlockParam(
                      type="tool_use",
                      id="toolu_01A09q90qw90lq917835lq9",
                      name="search_knowledge_base",
                      input={"query": "Acme Dashboard pricing plans"},
                  ),
              ],
          ),
          MessageParam(
              role="user",
              content=[
                  ToolResultBlockParam(
                      type="tool_result",
                      tool_use_id="toolu_01A09q90qw90lq917835lq9",
                      content=[
                          SearchResultBlockParam(
                              type="search_result",
                              source="https://docs.company.com/pricing",
                              title="Pricing Plans",
                              content=[
                                  TextBlockParam(
                                      type="text",
                                      text="Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing.",
                                  )
                              ],
                              citations={"enabled": True},
                          )
                      ],
                  )
              ],
          ),
      ],
  )

  print(response)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const knowledgeBaseTool: Anthropic.Tool = {
    name: "search_knowledge_base",
    description: "Search the company knowledge base for information",
    input_schema: {
      type: "object" as const,
      properties: {
        query: { type: "string", description: "The search query" }
      },
      required: ["query"]
    }
  };

  // 重放一段以两种方式提供搜索结果的对话：第一条
  // 用户消息携带预先获取的结果，工具结果返回另一个结果
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    tools: [knowledgeBaseTool],
    messages: [
      {
        role: "user",
        content: [
          {
            type: "search_result" as const,
            source: "https://docs.company.com/overview",
            title: "Product Overview",
            content: [
              {
                type: "text" as const,
                text: "Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards."
              }
            ],
            citations: { enabled: true }
          },
          {
            type: "text" as const,
            text: "What does Acme Dashboard do, and what plans is it available on?"
          }
        ]
      },
      {
        role: "assistant",
        content: [
          { type: "text" as const, text: "Let me check the pricing information." },
          {
            type: "tool_use" as const,
            id: "toolu_01A09q90qw90lq917835lq9",
            name: "search_knowledge_base",
            input: { query: "Acme Dashboard pricing plans" }
          }
        ]
      },
      {
        role: "user",
        content: [
          {
            type: "tool_result" as const,
            tool_use_id: "toolu_01A09q90qw90lq917835lq9",
            content: [
              {
                type: "search_result" as const,
                source: "https://docs.company.com/pricing",
                title: "Pricing Plans",
                content: [
                  {
                    type: "text" as const,
                    text: "Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing."
                  }
                ],
                citations: { enabled: true }
              }
            ]
          }
        ]
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  AnthropicClient client = new();

  // 重放一段以两种方式提供搜索结果的对话：第一条
  // 用户消息携带预取的结果，工具结果返回另一个结果
  var response = await client.Messages.Create(new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Tools =
      [
          new ToolUnion(new Tool()
          {
              Name = "search_knowledge_base",
              Description = "Search the company knowledge base for information",
              InputSchema = new InputSchema()
              {
                  Properties = new Dictionary<string, JsonElement>
                  {
                      ["query"] = JsonSerializer.SerializeToElement(new { type = "string", description = "The search query" }),
                  },
                  Required = ["query"],
              },
          }),
      ],
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = new MessageParamContent(
              [
                  new ContentBlockParam(new SearchResultBlockParam
                  {
                      Source = "https://docs.company.com/overview",
                      Title = "Product Overview",
                      Content = [new() { Text = "Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards." }],
                      Citations = new() { Enabled = true },
                  }),
                  new ContentBlockParam(new TextBlockParam { Text = "What does Acme Dashboard do, and what plans is it available on?" }),
              ]),
          },
          new()
          {
              Role = Role.Assistant,
              Content = new MessageParamContent(
              [
                  new ContentBlockParam(new TextBlockParam { Text = "Let me check the pricing information." }),
                  new ContentBlockParam(new ToolUseBlockParam
                  {
                      ID = "toolu_01A09q90qw90lq917835lq9",
                      Name = "search_knowledge_base",
                      Input = new Dictionary<string, JsonElement>
                      {
                          ["query"] = JsonSerializer.SerializeToElement("Acme Dashboard pricing plans"),
                      },
                  }),
              ]),
          },
          new()
          {
              Role = Role.User,
              Content = new MessageParamContent(
              [
                  new ContentBlockParam(new ToolResultBlockParam()
                  {
                      ToolUseID = "toolu_01A09q90qw90lq917835lq9",
                      Content = new ToolResultBlockParamContent(
                      [
                          new SearchResultBlockParam
                          {
                              Source = "https://docs.company.com/pricing",
                              Title = "Pricing Plans",
                              Content = [new() { Text = "Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing." }],
                              Citations = new() { Enabled = true },
                          },
                      ]),
                  }),
              ]),
          },
      ],
  });

  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  knowledgeBaseTool := anthropic.ToolUnionParam{
  	OfTool: &anthropic.ToolParam{
  		Name:        "search_knowledge_base",
  		Description: anthropic.String("Search the company knowledge base for information"),
  		InputSchema: anthropic.ToolInputSchemaParam{
  			Properties: map[string]any{
  				"query": map[string]any{"type": "string", "description": "The search query"},
  			},
  			Required: []string{"query"},
  		},
  	},
  }

  // 重放一段以两种方式提供搜索结果的对话：第一条
  // 用户消息携带预先获取的结果，工具结果返回另一个结果
  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Tools:     []anthropic.ToolUnionParam{knowledgeBaseTool},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.ContentBlockParamUnion{OfSearchResult: &anthropic.SearchResultBlockParam{
  				Content: []anthropic.TextBlockParam{
  					{Text: "Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards."},
  				},
  				Source:    "https://docs.company.com/overview",
  				Title:     "Product Overview",
  				Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
  			}},
  			anthropic.NewTextBlock("What does Acme Dashboard do, and what plans is it available on?"),
  		),
  		anthropic.NewAssistantMessage(
  			anthropic.NewTextBlock("Let me check the pricing information."),
  			anthropic.ContentBlockParamUnion{OfToolUse: &anthropic.ToolUseBlockParam{
  				ID:    "toolu_01A09q90qw90lq917835lq9",
  				Name:  "search_knowledge_base",
  				Input: map[string]any{"query": "Acme Dashboard pricing plans"},
  			}},
  		),
  		anthropic.NewUserMessage(
  			anthropic.ContentBlockParamUnion{OfToolResult: &anthropic.ToolResultBlockParam{
  				ToolUseID: "toolu_01A09q90qw90lq917835lq9",
  				Content: []anthropic.ToolResultBlockParamContentUnion{
  					{OfSearchResult: &anthropic.SearchResultBlockParam{
  						Content: []anthropic.TextBlockParam{
  							{Text: "Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing."},
  						},
  						Source:    "https://docs.company.com/pricing",
  						Title:     "Pricing Plans",
  						Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
  					}},
  				},
  			}},
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.core.JsonValue;
  import com.anthropic.models.messages.CitationsConfigParam;
  import com.anthropic.models.messages.ContentBlockParam;
  // ...
  import com.anthropic.models.messages.SearchResultBlockParam;
  import com.anthropic.models.messages.TextBlockParam;
  import com.anthropic.models.messages.Tool;
  import com.anthropic.models.messages.ToolResultBlockParam;
  import com.anthropic.models.messages.ToolUseBlockParam;
  // ...

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      Tool knowledgeBaseTool = Tool.builder()
          .name("search_knowledge_base")
          .description("Search the company knowledge base for information")
          .inputSchema(Tool.InputSchema.builder()
              .properties(JsonValue.from(Map.of(
                  "query", Map.of("type", "string", "description", "The search query")
              )))
              .putAdditionalProperty("required", JsonValue.from(List.of("query")))
              .build())
          .build();

      // 重放一段以两种方式提供搜索结果的对话：第一条
      // 用户消息携带预先获取的结果，工具结果返回另一个结果
      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024L)
          .addTool(knowledgeBaseTool)
          .addUserMessageOfBlockParams(List.of(
              ContentBlockParam.ofSearchResult(SearchResultBlockParam.builder()
                  .source("https://docs.company.com/overview")
                  .title("Product Overview")
                  .content(List.of(TextBlockParam.builder()
                      .text("Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards.")
                      .build()))
                  .citations(CitationsConfigParam.builder().enabled(true).build())
                  .build()),
              ContentBlockParam.ofText(TextBlockParam.builder()
                  .text("What does Acme Dashboard do, and what plans is it available on?")
                  .build())
          ))
          .addAssistantMessageOfBlockParams(List.of(
              ContentBlockParam.ofText(TextBlockParam.builder()
                  .text("Let me check the pricing information.")
                  .build()),
              ContentBlockParam.ofToolUse(ToolUseBlockParam.builder()
                  .id("toolu_01A09q90qw90lq917835lq9")
                  .name("search_knowledge_base")
                  .input(JsonValue.from(Map.of("query", "Acme Dashboard pricing plans")))
                  .build())
          ))
          .addUserMessageOfBlockParams(List.of(
              ContentBlockParam.ofToolResult(ToolResultBlockParam.builder()
                  .toolUseId("toolu_01A09q90qw90lq917835lq9")
                  .contentOfBlocks(List.of(
                      ToolResultBlockParam.Content.Block.ofSearchResult(SearchResultBlockParam.builder()
                          .source("https://docs.company.com/pricing")
                          .title("Pricing Plans")
                          .content(List.of(TextBlockParam.builder()
                              .text("Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing.")
                              .build()))
                          .citations(CitationsConfigParam.builder().enabled(true).build())
                          .build())
                  ))
                  .build())
          ))
          .build();

      Message response = client.messages().create(params);
      System.out.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $knowledgeBaseTool = [
      'name' => 'search_knowledge_base',
      'description' => 'Search the company knowledge base for information',
      'input_schema' => [
          'type' => 'object',
          'properties' => [
              'query' => ['type' => 'string', 'description' => 'The search query']
          ],
          'required' => ['query']
      ]
  ];

  // 重放一段以两种方式提供搜索结果的对话：第一条
  // 用户消息携带预取的结果，工具结果返回另一个结果
  $response = $client->messages->create(
      maxTokens: 1024,
      tools: [$knowledgeBaseTool],
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'search_result',
                      'source' => 'https://docs.company.com/overview',
                      'title' => 'Product Overview',
                      'content' => [
                          [
                              'type' => 'text',
                              'text' => 'Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards.'
                          ]
                      ],
                      'citations' => ['enabled' => true]
                  ],
                  [
                      'type' => 'text',
                      'text' => 'What does Acme Dashboard do, and what plans is it available on?'
                  ]
              ]
          ],
          [
              'role' => 'assistant',
              'content' => [
                  ['type' => 'text', 'text' => 'Let me check the pricing information.'],
                  [
                      'type' => 'tool_use',
                      'id' => 'toolu_01A09q90qw90lq917835lq9',
                      'name' => 'search_knowledge_base',
                      'input' => ['query' => 'Acme Dashboard pricing plans']
                  ]
              ]
          ],
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'tool_result',
                      'tool_use_id' => 'toolu_01A09q90qw90lq917835lq9',
                      'content' => [
                          [
                              'type' => 'search_result',
                              'source' => 'https://docs.company.com/pricing',
                              'title' => 'Pricing Plans',
                              'content' => [
                                  [
                                      'type' => 'text',
                                      'text' => 'Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing.'
                                  ]
                              ],
                              'citations' => ['enabled' => true]
                          ]
                      ]
                  ]
              ]
          ]
      ],
      model: 'claude-opus-5',
  );

  echo json_encode($response, JSON_PRETTY_PRINT);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  knowledge_base_tool = {
    name: "search_knowledge_base",
    description: "Search the company knowledge base for information",
    input_schema: {
      type: "object",
      properties: {
        query: { type: "string", description: "The search query" }
      },
      required: ["query"]
    }
  }

  # 重放一段以两种方式提供搜索结果的对话：第一条
  # 用户消息携带预取的结果，工具结果返回另一个结果
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    tools: [knowledge_base_tool],
    messages: [
      {
        role: "user",
        content: [
          {
            type: "search_result",
            source: "https://docs.company.com/overview",
            title: "Product Overview",
            content: [
              {
                type: "text",
                text: "Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards."
              }
            ],
            citations: { enabled: true }
          },
          {
            type: "text",
            text: "What does Acme Dashboard do, and what plans is it available on?"
          }
        ]
      },
      {
        role: "assistant",
        content: [
          { type: "text", text: "Let me check the pricing information." },
          {
            type: "tool_use",
            id: "toolu_01A09q90qw90lq917835lq9",
            name: "search_knowledge_base",
            input: { query: "Acme Dashboard pricing plans" }
          }
        ]
      },
      {
        role: "user",
        content: [
          {
            type: "tool_result",
            tool_use_id: "toolu_01A09q90qw90lq917835lq9",
            content: [
              {
                type: "search_result",
                source: "https://docs.company.com/pricing",
                title: "Pricing Plans",
                content: [
                  {
                    type: "text",
                    text: "Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing."
                  }
                ],
                citations: { enabled: true }
              }
            ]
          }
        ]
      }
    ]
  )

  puts response
  ```
</CodeGroup>

响应引用了两个来源。预取的结果为 `search_result_index: 0`，工具返回的结果为 `search_result_index: 1`，与 `search_result` 块在对话中出现的顺序一致：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Here's what I found about Acme Dashboard:\n\n**What it does:** "
    },
    {
      "type": "text",
      "text": "Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards.",
      "citations": [
        {
          "type": "search_result_location",
          "cited_text": "Acme Dashboard is a monitoring tool for distributed systems. It supports real-time alerting and custom metric dashboards.",
          "source": "https://docs.company.com/overview",
          "title": "Product Overview",
          "search_result_index": 0,
          "start_block_index": 0,
          "end_block_index": 1
        }
      ]
    },
    {
      "type": "text",
      "text": "\n\n**Available plans:** "
    },
    {
      "type": "text",
      "text": "Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing.",
      "citations": [
        {
          "type": "search_result_location",
          "cited_text": "Acme Dashboard is available on the Starter plan at $10 per user per month and the Enterprise plan with custom pricing.",
          "source": "https://docs.company.com/pricing",
          "title": "Pricing Plans",
          "search_result_index": 1,
          "start_block_index": 0,
          "end_block_index": 1
        }
      ]
    }
  ]
}
```

### 与其他内容类型混合

在用户消息中，`search_result` 块可以与任何其他内容块并列。方法 2 的示例将搜索结果与一个 `text` 问题配对，图像或文档块也可以以同样的方式加入。

工具结果则更为严格：如果 `tool_result` 内容数组中的任何块是 `search_result`，则其所有块都必须是 `search_result`。在同一工具结果中将搜索结果与其他块类型混合会返回验证错误。要在工具来源的搜索结果旁返回辅助文本，请将其作为文本块包含在某个搜索结果的 `content` 数组中，这样它也会变得可引用。

### 缓存控制

在搜索结果块上添加 `cache_control` 以缓存它，供跨请求重复使用。它与 `citations` 并列位于同一块上：

```json
{
  "type": "search_result",
  "source": "https://docs.company.com/guide",
  "title": "User Guide",
  "content": [{ "type": "text", "text": "..." }],
  "citations": { "enabled": true },
  "cache_control": { "type": "ephemeral" }
}
```

有关最小可缓存长度和其他要求，请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)。

### 引用控制

默认情况下，搜索结果的引用是禁用的。您可以通过显式设置 `citations` 配置来启用引用：

```json
{
  "type": "search_result",
  "source": "https://docs.company.com/guide",
  "title": "User Guide",
  "content": [{ "type": "text", "text": "Important documentation..." }],
  "citations": {
    "enabled": true // Enable citations for this result
  }
}
```

当 `citations.enabled` 设置为 `true` 时，Claude 会将引用参考附加到借鉴了该搜索结果的文本块上。

<Warning>
  引用是全有或全无的：一个请求中的所有搜索结果要么必须全部启用引用，要么必须全部禁用。混合使用不同引用设置的搜索结果会导致错误。
</Warning>

## 最佳实践

### 基于工具的搜索（方法 1）

* **动态内容：** 用于实时搜索和动态 RAG 应用
* **错误处理：** 搜索失败时返回适当的消息
* **结果限制：** 仅返回最相关的结果，以避免上下文溢出

### 顶层搜索（方法 2）

* **预取内容：** 在您已有搜索结果时使用
* **批处理：** 非常适合一次处理多个搜索结果
* **测试：** 非常适合使用已知内容测试引用行为

### 通用最佳实践

1. **有效地组织结果：**

   * 使用清晰、永久的来源 URL
   * 提供描述性标题
   * 将长内容拆分为逻辑文本块，为 Claude 提供更精细的引用边界

2. **保持一致性：**

   * 在整个应用中使用一致的来源格式
   * 确保标题准确反映内容
   * 保持格式一致

3. **优雅地处理错误：** 当搜索失败或没有返回任何内容时，返回一个描述结果的纯文本块（例如 `{"type": "text", "text": "No results found."}`），而不是抛出错误：Claude 会向用户解释空结果，对话继续进行。

## 限制

* 搜索结果内容块可在 Claude API、Amazon Bedrock 和 Google Cloud 上使用。
* 搜索结果中仅支持文本内容（不支持图像或其他媒体）。
* `search_result` 块只能出现在用户消息中（包括工具结果内部）。包含搜索结果的助手消息会被拒绝。
* 当在同一请求中启用了[网络搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)时，必须在所有 `search_result` 块上启用引用。

## 后续步骤

<CardGroup cols={2}>
  <Card title="流式传输拒绝" icon="lock" href="https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/handle-streaming-refusals">
    在流式传输响应中检测并处理拒绝停止原因，并在备用模型上重试被拒绝的请求。
  </Card>

  <Card title="引用" icon="book" href="https://platform.claude.com/docs/zh-CN/build-with-claude/citations">
    让 Claude 的响应以您的源文档为依据。引用会返回支持每项陈述的确切段落，以便您验证答案并向用户展示来源。
  </Card>

  <Card title="网络搜索工具" icon="magnifying-glass" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool">
    让 Claude 能够访问带有引用来源的最新网络内容，并提供可选的动态过滤和域名控制。
  </Card>

  <Card title="Messages API 参考" icon="code" href="https://platform.claude.com/docs/zh-CN/api/messages/create">
    查看完整的 Messages API 文档，包括内容块类型。
  </Card>

  <Card title="提示缓存" icon="database" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching">
    使用 `cache_control` 缓存搜索结果，以降低重复请求的成本和延迟。
  </Card>
</CardGroup>
