---
title: 上下文编辑
url: https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing
description: 通过上下文编辑，在对话上下文不断增长时自动对其进行管理。
---

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

## 概述

<Note>
  对于大多数用例，[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)是管理长时间运行对话中上下文的主要策略。本页介绍的策略适用于您需要对清除哪些内容进行更细粒度控制的特定场景。
</Note>

"Context editing"（上下文编辑）允许您在对话历史不断增长时，有选择地从中清除特定内容。除了优化成本和保持在限制范围内之外，这更关乎主动筛选 Claude 所看到的内容：上下文是一种收益递减的有限资源，不相关的内容会削弱模型的专注度。上下文编辑让您能够在运行时对这种筛选进行细粒度控制。有关上下文管理背后更广泛的原则，请参阅[有效的上下文工程](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)。本页涵盖：

* **工具结果清除** - 最适合大量进行工具使用的智能体工作流，其中旧的工具结果已不再需要
* **思考块清除** - 用于在使用 "extended thinking"（扩展思考）时管理思考块，并提供保留近期思考以保持上下文连续性的选项
* **客户端 SDK 压缩** - 一种基于 SDK 的替代方案，用于基于摘要的上下文管理（通常更推荐服务端压缩）

| 方式      | 运行位置 | 策略                                                                  | 工作原理                                                                                                                                                                                                                                                                                                                                   |
| ------- | ---- | ------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **服务端** | API  | 工具结果清除（`clear_tool_uses_20250919`） 思考块清除（`clear_thinking_20251015`） | 在提示到达 Claude 之前应用。从对话历史中清除特定内容。每种策略都可以独立配置。                                                                                                                                                                                                                                                                                            |
| **客户端** | SDK  | 压缩                                                                  | 在使用 [`tool_runner`](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner) 时，可在 [TypeScript 和 Ruby SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview) 中使用。生成摘要并替换完整的对话历史。请参阅[客户端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#client-side-compaction-sdk)。 |

## 服务端策略

<Note>
  上下文编辑目前处于 beta 阶段，支持工具结果清除和思考块清除。要启用它，请在您的 API 请求中使用 beta 标头 `context-management-2025-06-27`。

  请通过[反馈表单](https://forms.gle/YXC2EKGMhjN1c4L88)分享您对此功能的反馈。
</Note>

### 工具结果清除

`clear_tool_uses_20250919` 策略会在对话上下文增长超过您配置的阈值时清除工具结果。这对于大量进行工具使用的智能体工作流特别有用。较旧的工具结果（如文件内容或搜索结果）在 Claude 处理完之后就不再需要了。

激活后，API 会按时间顺序自动清除最旧的工具结果。API 会将每个被清除的结果替换为占位文本，向 Claude 表明该结果已被移除。默认情况下，仅清除工具结果。您可以选择通过将 `clear_tool_inputs` 设置为 true 来同时清除工具结果和工具调用（即工具使用参数）。

### 思考块清除

`clear_thinking_20251015` 策略用于在启用扩展思考时管理对话中的 `thinking` 块。此策略让您可以控制思考内容的保留：您可以选择保留更多思考块以维持推理的连续性，或者更积极地清除它们以节省上下文空间。

<Tip>
  **默认行为：** 默认值因模型类别而异。

  | 模型类别           | 保留所有先前的思考               | 仅保留最后一轮的思考                |
  | -------------- | ----------------------- | ------------------------- |
  | Opus           | Claude Opus 4.5 及更高版本   | Claude Opus 4.1 及更早版本     |
  | Sonnet         | Claude Sonnet 4.6 及更高版本 | Claude Sonnet 4.5 及更早版本   |
  | Haiku          | （无）                     | 直至 Claude Haiku 4.5 的所有模型 |
  | Fable 和 Mythos | 所有模型                    | （无）                       |

  使用此策略可覆盖默认值。如果您的代码跨多个模型层级运行，请显式设置 `keep`，而不要依赖各模型的默认值。
</Tip>

一个助手对话轮次可能包含多个内容块（例如，在使用工具时）和多个思考块（例如，在使用[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)时）。

### 上下文编辑在服务端进行

上下文编辑在提示到达 Claude 之前于服务端应用。您的客户端应用程序保留完整、未经修改的对话历史。您无需将客户端状态与编辑后的版本同步。请像往常一样继续在本地管理您的完整对话历史。

在 Claude Fable 5.1 上，服务端上下文管理永远不会使思考块失效。客户端对较早轮次的编辑可能会使之后每个助手轮次中的思考块失效。对于在 2026 年 8 月 31 日或之后创建的新账户，重放已失效块的请求将被拒绝，除非您选择丢弃该块。请参阅[保留的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)。

### 上下文编辑与提示缓存

上下文编辑与 "prompt caching"（[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)）的交互方式因策略而异：

* **工具结果清除：** 在清除内容时会使已缓存的提示前缀失效。考虑到这一点，请清除足够多的令牌，使缓存失效变得值得。使用 `clear_at_least` 参数可确保每次至少清除一定数量的令牌。每次清除内容时您都会产生缓存写入成本，但后续请求可以重用新缓存的前缀。

* **思考块清除：** 当思考块被**保留**在上下文中（未被清除）时，提示缓存得以保留，从而实现缓存命中并降低输入令牌成本。当思考块被**清除**时，缓存会在清除发生的位置失效。请根据您是希望优先考虑缓存性能还是上下文窗口可用空间来配置 `keep` 参数。

## 支持的模型

上下文编辑可在所有受支持的 Claude 模型上使用。

## 工具结果清除的用法

启用工具结果清除最简单的方法是仅指定策略类型。所有其他[配置选项](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#configuration-options-for-tool-result-clearing)均使用其默认值：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
      --header "x-api-key: $ANTHROPIC_API_KEY" \
      --header "anthropic-version: 2023-06-01" \
      --header "content-type: application/json" \
      --header "anthropic-beta: context-management-2025-06-27" \
      --data '{
          "model": "claude-opus-5",
          "max_tokens": 4096,
          "messages": [
              {
                  "role": "user",
                  "content": "Search for recent developments in AI"
              }
          ],
          "tools": [
              {
                  "type": "web_search_20250305",
                  "name": "web_search"
              }
          ],
          "context_management": {
              "edits": [
                  {"type": "clear_tool_uses_20250919"}
              ]
          }
      }'
  ```

  ```bash CLI
  ant beta:messages create --beta context-management-2025-06-27 <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  messages:
    - role: user
      content: Search for recent developments in AI
  tools:
    - type: web_search_20250305
      name: web_search
  context_management:
    edits:
      - type: clear_tool_uses_20250919
  YAML
  ```

  ```python Python
  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      messages=[{"role": "user", "content": "Search for recent developments in AI"}],
      tools=[{"type": "web_search_20250305", "name": "web_search"}],
      betas=["context-management-2025-06-27"],
      context_management={"edits": [{"type": "clear_tool_uses_20250919"}]},
  )
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    messages: [
      {
        role: "user",
        content: "Search for recent developments in AI"
      }
    ],
    tools: [
      {
        type: "web_search_20250305",
        name: "web_search"
      }
    ],
    context_management: {
      edits: [{ type: "clear_tool_uses_20250919" }]
    },
    betas: ["context-management-2025-06-27"]
  });
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5,
      MaxTokens = 4096,
      Messages = [
          new() { Role = Role.User, Content = "Search for recent developments in AI" }
      ],
      Tools = [
          new BetaWebSearchTool20250305()
      ],
      ContextManagement = new BetaContextManagementConfig
      {
          Edits = [new BetaClearToolUses20250919Edit()]
      },
      Betas = [AnthropicBeta.ContextManagement2025_06_27]
  };

  var response = await client.Beta.Messages.Create(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 4096,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Search for recent developments in AI")),
  	},
  	Tools: []anthropic.BetaToolUnionParam{
  		{OfWebSearchTool20250305: &anthropic.BetaWebSearchTool20250305Param{}},
  	},
  	ContextManagement: anthropic.BetaContextManagementConfigParam{
  		Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
  			{OfClearToolUses20250919: &anthropic.BetaClearToolUses20250919EditParam{}},
  		},
  	},
  	Betas: []anthropic.AnthropicBeta{
  		anthropic.AnthropicBetaContextManagement2025_06_27,
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.beta.messages.BetaWebSearchTool20250305;
  import com.anthropic.models.beta.messages.BetaContextManagementConfig;
  import com.anthropic.models.beta.messages.BetaClearToolUses20250919Edit;
  import com.anthropic.models.beta.AnthropicBeta;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .addUserMessage("Search for recent developments in AI")
          .addTool(BetaWebSearchTool20250305.builder().build())
          .contextManagement(BetaContextManagementConfig.builder()
              .addEdit(BetaClearToolUses20250919Edit.builder().build())
              .build())
          .addBeta(AnthropicBeta.CONTEXT_MANAGEMENT_2025_06_27)
          .build();

      BetaMessage response = client.beta().messages().create(params);
      IO.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Search for recent developments in AI']
      ],
      model: 'claude-opus-5',
      betas: ['context-management-2025-06-27'],
      tools: [
          ['type' => 'web_search_20250305', 'name' => 'web_search']
      ],
      contextManagement: [
          'edits' => [
              ['type' => 'clear_tool_uses_20250919']
          ]
      ],
  );

  echo $response;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    messages: [
      { role: "user", content: "Search for recent developments in AI" }
    ],
    tools: [
      { type: "web_search_20250305", name: "web_search" }
    ],
    context_management: {
      edits: [
        { type: "clear_tool_uses_20250919" }
      ]
    },
    betas: ["context-management-2025-06-27"]
  )
  puts response
  ```
</CodeGroup>

### 高级配置

您可以通过附加参数自定义工具结果清除的行为：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
      --header "x-api-key: $ANTHROPIC_API_KEY" \
      --header "anthropic-version: 2023-06-01" \
      --header "content-type: application/json" \
      --header "anthropic-beta: context-management-2025-06-27" \
      --data '{
          "model": "claude-opus-5",
          "max_tokens": 4096,
          "messages": [
              {
                  "role": "user",
                  "content": "Create a simple command line calculator app using Python"
              }
          ],
          "tools": [
              {
                  "type": "text_editor_20250728",
                  "name": "str_replace_based_edit_tool",
                  "max_characters": 10000
              },
              {
                  "type": "web_search_20250305",
                  "name": "web_search",
                  "max_uses": 3
              }
          ],
          "context_management": {
              "edits": [
                  {
                      "type": "clear_tool_uses_20250919",
                      "trigger": {
                          "type": "input_tokens",
                          "value": 30000
                      },
                      "keep": {
                          "type": "tool_uses",
                          "value": 3
                      },
                      "clear_at_least": {
                          "type": "input_tokens",
                          "value": 5000
                      },
                      "exclude_tools": ["web_search"]
                  }
              ]
          }
      }'
  ```

  ```bash CLI
  ant beta:messages create --beta context-management-2025-06-27 <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  messages:
    - role: user
      content: Create a simple command line calculator app using Python
  tools:
    - type: text_editor_20250728
      name: str_replace_based_edit_tool
      max_characters: 10000
    - type: web_search_20250305
      name: web_search
      max_uses: 3
  context_management:
    edits:
      - type: clear_tool_uses_20250919
        trigger:
          type: input_tokens
          value: 30000
        keep:
          type: tool_uses
          value: 3
        clear_at_least:
          type: input_tokens
          value: 5000
        exclude_tools:
          - web_search
  YAML
  ```

  ```python Python
  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      messages=[
          {
              "role": "user",
              "content": "Create a simple command line calculator app using Python",
          }
      ],
      tools=[
          {
              "type": "text_editor_20250728",
              "name": "str_replace_based_edit_tool",
              "max_characters": 10000,
          },
          {"type": "web_search_20250305", "name": "web_search", "max_uses": 3},
      ],
      betas=["context-management-2025-06-27"],
      context_management={
          "edits": [
              {
                  "type": "clear_tool_uses_20250919",
                  # 超过阈值时触发清除
                  "trigger": {"type": "input_tokens", "value": 30000},
                  # 清除后保留的工具使用次数
                  "keep": {"type": "tool_uses", "value": 3},
                  # 可选：至少清除这么多令牌
                  "clear_at_least": {"type": "input_tokens", "value": 5000},
                  # 将这些工具排除在清除范围之外
                  "exclude_tools": ["web_search"],
              }
          ]
      },
  )
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    messages: [
      {
        role: "user",
        content: "Create a simple command line calculator app using Python"
      }
    ],
    tools: [
      {
        type: "text_editor_20250728",
        name: "str_replace_based_edit_tool",
        max_characters: 10000
      },
      {
        type: "web_search_20250305",
        name: "web_search",
        max_uses: 3
      }
    ],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_tool_uses_20250919",
          // 超过阈值时触发清除
          trigger: {
            type: "input_tokens",
            value: 30000
          },
          // 清除后保留的工具使用次数
          keep: {
            type: "tool_uses",
            value: 3
          },
          // 可选：至少清除这么多令牌
          clear_at_least: {
            type: "input_tokens",
            value: 5000
          },
          // 将这些工具排除在清除范围之外
          exclude_tools: ["web_search"]
        }
      ]
    }
  });
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5,
      MaxTokens = 4096,
      Messages = [
          new() { Role = Role.User, Content = "Create a simple command line calculator app using Python" }
      ],
      Tools = [
          new BetaToolTextEditor20250728 { MaxCharacters = 10000 },
          new BetaWebSearchTool20250305 { MaxUses = 3 }
      ],
      Betas = [AnthropicBeta.ContextManagement2025_06_27],
      ContextManagement = new BetaContextManagementConfig
      {
          Edits = [
              new BetaClearToolUses20250919Edit
              {
                  Trigger = new BetaInputTokensTrigger(30000),
                  Keep = new BetaToolUsesKeep(3),
                  ClearAtLeast = new BetaInputTokensClearAtLeast(5000),
                  ExcludeTools = ["web_search"]
              }
          ]
      }
  };

  var response = await client.Beta.Messages.Create(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 4096,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Create a simple command line calculator app using Python")),
  	},
  	Tools: []anthropic.BetaToolUnionParam{
  		{OfTextEditor20250728: &anthropic.BetaToolTextEditor20250728Param{
  			MaxCharacters: anthropic.Int(10000),
  		}},
  		{OfWebSearchTool20250305: &anthropic.BetaWebSearchTool20250305Param{
  			MaxUses: anthropic.Int(3),
  		}},
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaContextManagement2025_06_27},
  	ContextManagement: anthropic.BetaContextManagementConfigParam{
  		Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
  			{OfClearToolUses20250919: &anthropic.BetaClearToolUses20250919EditParam{
  				Trigger: anthropic.BetaClearToolUses20250919EditTriggerUnionParam{
  					OfInputTokens: &anthropic.BetaInputTokensTriggerParam{
  						Value: 30000,
  					},
  				},
  				Keep: anthropic.BetaToolUsesKeepParam{
  					Value: 3,
  				},
  				ClearAtLeast: anthropic.BetaInputTokensClearAtLeastParam{
  					Value: 5000,
  				},
  				ExcludeTools: []string{"web_search"},
  			}},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.beta.messages.BetaToolTextEditor20250728;
  import com.anthropic.models.beta.messages.BetaWebSearchTool20250305;
  import com.anthropic.models.beta.messages.BetaContextManagementConfig;
  import com.anthropic.models.beta.messages.BetaClearToolUses20250919Edit;
  import com.anthropic.models.beta.messages.BetaInputTokensTrigger;
  import com.anthropic.models.beta.messages.BetaInputTokensClearAtLeast;
  import com.anthropic.models.beta.messages.BetaToolUsesKeep;
  import com.anthropic.models.beta.AnthropicBeta;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .addUserMessage("Create a simple command line calculator app using Python")
          .addTool(BetaToolTextEditor20250728.builder()
              .maxCharacters(10000L)
              .build())
          .addTool(BetaWebSearchTool20250305.builder()
              .maxUses(3L)
              .build())
          .addBeta(AnthropicBeta.CONTEXT_MANAGEMENT_2025_06_27)
          .contextManagement(BetaContextManagementConfig.builder()
              .addEdit(BetaClearToolUses20250919Edit.builder()
                  .trigger(BetaInputTokensTrigger.builder()
                      .value(30000L)
                      .build())
                  .keep(BetaToolUsesKeep.builder()
                      .value(3L)
                      .build())
                  .clearAtLeast(BetaInputTokensClearAtLeast.builder()
                      .value(5000L)
                      .build())
                  .addExcludeTool("web_search")
                  .build())
              .build())
          .build();

      BetaMessage response = client.beta().messages().create(params);
      IO.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      maxTokens: 4096,
      messages: [
          [
              'role' => 'user',
              'content' => 'Create a simple command line calculator app using Python'
          ]
      ],
      model: 'claude-opus-5',
      betas: ['context-management-2025-06-27'],
      tools: [
          [
              'type' => 'text_editor_20250728',
              'name' => 'str_replace_based_edit_tool',
              'max_characters' => 10000
          ],
          [
              'type' => 'web_search_20250305',
              'name' => 'web_search',
              'max_uses' => 3
          ]
      ],
      contextManagement: [
          'edits' => [
              [
                  'type' => 'clear_tool_uses_20250919',
                  'trigger' => [
                      'type' => 'input_tokens',
                      'value' => 30000
                  ],
                  'keep' => [
                      'type' => 'tool_uses',
                      'value' => 3
                  ],
                  'clear_at_least' => [
                      'type' => 'input_tokens',
                      'value' => 5000
                  ],
                  'exclude_tools' => ['web_search']
              ]
          ]
      ],
  );

  echo $response;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    messages: [
      {
        role: "user",
        content: "Create a simple command line calculator app using Python"
      }
    ],
    tools: [
      {
        type: "text_editor_20250728",
        name: "str_replace_based_edit_tool",
        max_characters: 10000
      },
      {
        type: "web_search_20250305",
        name: "web_search",
        max_uses: 3
      }
    ],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_tool_uses_20250919",
          trigger: {
            type: "input_tokens",
            value: 30000
          },
          keep: {
            type: "tool_uses",
            value: 3
          },
          clear_at_least: {
            type: "input_tokens",
            value: 5000
          },
          exclude_tools: ["web_search"]
        }
      ]
    }
  )
  puts response
  ```
</CodeGroup>

## 思考块清除的用法

启用思考块清除，以便在启用扩展思考时有效地管理上下文和提示缓存：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
      --header "x-api-key: $ANTHROPIC_API_KEY" \
      --header "anthropic-version: 2023-06-01" \
      --header "content-type: application/json" \
      --header "anthropic-beta: context-management-2025-06-27" \
      --data '{
          "model": "claude-opus-5",
          "max_tokens": 16000,
          "messages": [{"role": "user", "content": "Hello"}],
          "context_management": {
              "edits": [
                  {
                      "type": "clear_thinking_20251015",
                      "keep": {
                          "type": "thinking_turns",
                          "value": 2
                      }
                  }
              ]
          }
      }'
  ```

  ```bash CLI
  ant beta:messages create --beta context-management-2025-06-27 <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  messages:
    - role: user
      content: Hello
  context_management:
    edits:
      - type: clear_thinking_20251015
        keep:
          type: thinking_turns
          value: 2
  YAML
  ```

  ```python Python
  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      messages=[{"role": "user", "content": "Hello"}],
      betas=["context-management-2025-06-27"],
      context_management={
          "edits": [
              {
                  "type": "clear_thinking_20251015",
                  "keep": {"type": "thinking_turns", "value": 2},
              }
          ]
      },
  )
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 16000,
    messages: [{ role: "user", content: "Hello" }],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_thinking_20251015",
          keep: {
            type: "thinking_turns",
            value: 2
          }
        }
      ]
    }
  });
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5,
      MaxTokens = 16000,
      Messages = [
          new() { Role = Role.User, Content = "Hello" }
      ],
      Betas = [AnthropicBeta.ContextManagement2025_06_27],
      ContextManagement = new BetaContextManagementConfig
      {
          Edits = [
              new BetaClearThinking20251015Edit
              {
                  Keep = new BetaThinkingTurns(2)
              }
          ]
      }
  };

  var response = await client.Beta.Messages.Create(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 16000,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello")),
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaContextManagement2025_06_27},
  	ContextManagement: anthropic.BetaContextManagementConfigParam{
  		Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
  			{OfClearThinking20251015: &anthropic.BetaClearThinking20251015EditParam{
  				Keep: anthropic.BetaClearThinking20251015EditKeepUnionParam{
  					OfThinkingTurns: &anthropic.BetaThinkingTurnsParam{
  						Value: 2,
  					},
  				},
  			}},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.beta.messages.BetaContextManagementConfig;
  import com.anthropic.models.beta.messages.BetaClearThinking20251015Edit;
  import com.anthropic.models.beta.messages.BetaThinkingTurns;
  import com.anthropic.models.beta.AnthropicBeta;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(16000L)
          .addUserMessage("Hello")
          .addBeta(AnthropicBeta.CONTEXT_MANAGEMENT_2025_06_27)
          .contextManagement(BetaContextManagementConfig.builder()
              .addEdit(BetaClearThinking20251015Edit.builder()
                  .keep(BetaThinkingTurns.builder()
                      .value(2L)
                      .build())
                  .build())
              .build())
          .build();

      BetaMessage response = client.beta().messages().create(params);
      IO.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      maxTokens: 16000,
      messages: [
          ['role' => 'user', 'content' => 'Hello']
      ],
      model: 'claude-opus-5',
      betas: ['context-management-2025-06-27'],
      contextManagement: [
          'edits' => [
              [
                  'type' => 'clear_thinking_20251015',
                  'keep' => [
                      'type' => 'thinking_turns',
                      'value' => 2
                  ]
              ]
          ]
      ],
  );

  echo $response;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 16000,
    messages: [{ role: "user", content: "Hello" }],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_thinking_20251015",
          keep: {
            type: "thinking_turns",
            value: 2
          }
        }
      ]
    }
  )
  puts response
  ```
</CodeGroup>

### 思考块清除的配置选项

`clear_thinking_20251015` 策略支持以下配置：

| 配置选项   | 默认值   | 描述                                                                                                                                                                                               |
| ------ | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `keep` | 因模型而异 | 定义要保留多少个最近的包含思考块的助手轮次。使用 `{type: "thinking_turns", value: N}`（其中 N 必须 > 0）来保留最后 N 个轮次，或使用 `"all"` 来保留所有思考块。Opus 4.5+ 和 Sonnet 4.6+：所有轮次。Fable 和 Mythos 模型：所有轮次。更早的 Opus/Sonnet 以及所有 Haiku：仅最后一轮。 |

**配置示例：**

保留最后 3 个助手轮次的思考块：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
      --header "x-api-key: $ANTHROPIC_API_KEY" \
      --header "anthropic-version: 2023-06-01" \
      --header "content-type: application/json" \
      --header "anthropic-beta: context-management-2025-06-27" \
      --data '{
          "model": "claude-opus-5",
          "max_tokens": 16000,
          "messages": [{"role": "user", "content": "Hello"}],
          "context_management": {
              "edits": [
                  {
                      "type": "clear_thinking_20251015",
                      "keep": {
                          "type": "thinking_turns",
                          "value": 3
                      }
                  }
              ]
          }
      }'
  ```

  ```bash CLI
  ant beta:messages create --beta context-management-2025-06-27 <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  messages:
    - role: user
      content: Hello
  context_management:
    edits:
      - type: clear_thinking_20251015
        keep:
          type: thinking_turns
          value: 3
  YAML
  ```

  ```python Python
  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      messages=[{"role": "user", "content": "Hello"}],
      betas=["context-management-2025-06-27"],
      context_management={
          "edits": [
              {
                  "type": "clear_thinking_20251015",
                  "keep": {"type": "thinking_turns", "value": 3},
              }
          ]
      },
  )
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 16000,
    messages: [{ role: "user", content: "Hello" }],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_thinking_20251015",
          keep: {
            type: "thinking_turns",
            value: 3
          }
        }
      ]
    }
  });
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5,
      MaxTokens = 16000,
      Messages = [
          new() { Role = Role.User, Content = "Hello" }
      ],
      Betas = [AnthropicBeta.ContextManagement2025_06_27],
      ContextManagement = new BetaContextManagementConfig
      {
          Edits = [
              new BetaClearThinking20251015Edit
              {
                  Keep = new BetaThinkingTurns(3)
              }
          ]
      }
  };

  var response = await client.Beta.Messages.Create(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 16000,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello")),
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaContextManagement2025_06_27},
  	ContextManagement: anthropic.BetaContextManagementConfigParam{
  		Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
  			{OfClearThinking20251015: &anthropic.BetaClearThinking20251015EditParam{
  				Keep: anthropic.BetaClearThinking20251015EditKeepUnionParam{
  					OfThinkingTurns: &anthropic.BetaThinkingTurnsParam{
  						Value: 3,
  					},
  				},
  			}},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(16000L)
      .addUserMessage("Hello")
      .addBeta(AnthropicBeta.CONTEXT_MANAGEMENT_2025_06_27)
      .contextManagement(BetaContextManagementConfig.builder()
          .addEdit(BetaClearThinking20251015Edit.builder()
              .keep(BetaThinkingTurns.builder()
                  .value(3L)
                  .build())
              .build())
          .build())
      .build();

  BetaMessage response = client.beta().messages().create(params);
  IO.println(response);
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      maxTokens: 16000,
      messages: [
          ['role' => 'user', 'content' => 'Hello']
      ],
      model: 'claude-opus-5',
      betas: ['context-management-2025-06-27'],
      contextManagement: [
          'edits' => [
              [
                  'type' => 'clear_thinking_20251015',
                  'keep' => [
                      'type' => 'thinking_turns',
                      'value' => 3
                  ]
              ]
          ]
      ],
  );

  echo $response;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 16000,
    messages: [{ role: "user", content: "Hello" }],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_thinking_20251015",
          keep: {
            type: "thinking_turns",
            value: 3
          }
        }
      ]
    }
  )
  puts response
  ```
</CodeGroup>

保留所有思考块（最大化缓存命中）：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
      --header "x-api-key: $ANTHROPIC_API_KEY" \
      --header "anthropic-version: 2023-06-01" \
      --header "content-type: application/json" \
      --header "anthropic-beta: context-management-2025-06-27" \
      --data '{
          "model": "claude-opus-5",
          "max_tokens": 16000,
          "messages": [{"role": "user", "content": "Hello"}],
          "context_management": {
              "edits": [
                  {
                      "type": "clear_thinking_20251015",
                      "keep": "all"
                  }
              ]
          }
      }'
  ```

  ```bash CLI
  ant beta:messages create --beta context-management-2025-06-27 <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  messages:
    - role: user
      content: Hello
  context_management:
    edits:
      - type: clear_thinking_20251015
        keep: all
  YAML
  ```

  ```python Python
  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      messages=[{"role": "user", "content": "Hello"}],
      betas=["context-management-2025-06-27"],
      context_management={
          "edits": [
              {
                  "type": "clear_thinking_20251015",
                  "keep": "all",
              }
          ]
      },
  )
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 16000,
    messages: [{ role: "user", content: "Hello" }],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_thinking_20251015",
          keep: "all"
        }
      ]
    }
  });
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5,
      MaxTokens = 16000,
      Messages = [
          new() { Role = Role.User, Content = "Hello" }
      ],
      Betas = [AnthropicBeta.ContextManagement2025_06_27],
      ContextManagement = new BetaContextManagementConfig
      {
          Edits = [
              new BetaClearThinking20251015Edit
              {
                  Keep = new All()
              }
          ]
      }
  };

  var response = await client.Beta.Messages.Create(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 16000,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello")),
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaContextManagement2025_06_27},
  	ContextManagement: anthropic.BetaContextManagementConfigParam{
  		Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
  			{OfClearThinking20251015: &anthropic.BetaClearThinking20251015EditParam{
  				Keep: anthropic.BetaClearThinking20251015EditKeepUnionParam{
  					OfAll: constant.ValueOf[constant.All](),
  				},
  			}},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(16000L)
      .addUserMessage("Hello")
      .addBeta(AnthropicBeta.CONTEXT_MANAGEMENT_2025_06_27)
      .contextManagement(BetaContextManagementConfig.builder()
          .addEdit(BetaClearThinking20251015Edit.builder()
              .keepAll()
              .build())
          .build())
      .build();

  BetaMessage response = client.beta().messages().create(params);
  IO.println(response);
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      maxTokens: 16000,
      messages: [
          ['role' => 'user', 'content' => 'Hello']
      ],
      model: 'claude-opus-5',
      betas: ['context-management-2025-06-27'],
      contextManagement: [
          'edits' => [
              [
                  'type' => 'clear_thinking_20251015',
                  'keep' => 'all'
              ]
          ]
      ],
  );

  echo $response;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 16000,
    messages: [{ role: "user", content: "Hello" }],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_thinking_20251015",
          keep: "all"
        }
      ]
    }
  )
  puts response
  ```
</CodeGroup>

### 组合策略

您可以同时使用思考块清除和工具结果清除：

<Note>
  使用多种策略时，`clear_thinking_20251015` 策略必须在 `edits` 数组中排在首位。
</Note>

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
      --header "x-api-key: $ANTHROPIC_API_KEY" \
      --header "anthropic-version: 2023-06-01" \
      --header "content-type: application/json" \
      --header "anthropic-beta: context-management-2025-06-27" \
      --data '{
          "model": "claude-opus-5",
          "max_tokens": 16000,
          "messages": [
              {
                  "role": "user",
                  "content": "Search for the latest developments in quantum error correction and summarize the key breakthroughs."
              }
          ],
          "tools": [
              {
                  "type": "web_search_20250305",
                  "name": "web_search",
                  "max_uses": 5
              }
          ],
          "context_management": {
              "edits": [
                  {
                      "type": "clear_thinking_20251015",
                      "keep": {
                          "type": "thinking_turns",
                          "value": 2
                      }
                  },
                  {
                      "type": "clear_tool_uses_20250919",
                      "trigger": {
                          "type": "input_tokens",
                          "value": 50000
                      },
                      "keep": {
                          "type": "tool_uses",
                          "value": 5
                      }
                  }
              ]
          }
      }'
  ```

  ```bash CLI
  ant beta:messages create --beta context-management-2025-06-27 <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  messages:
    - role: user
      content: Search for the latest developments in quantum error correction and summarize the key breakthroughs.
  tools:
    - type: web_search_20250305
      name: web_search
      max_uses: 5
  context_management:
    edits:
      - type: clear_thinking_20251015
        keep:
          type: thinking_turns
          value: 2
      - type: clear_tool_uses_20250919
        trigger:
          type: input_tokens
          value: 50000
        keep:
          type: tool_uses
          value: 5
  YAML
  ```

  ```python Python
  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      messages=[
          {
              "role": "user",
              "content": "Search for the latest developments in quantum error correction and summarize the key breakthroughs.",
          }
      ],
      tools=[
          {
              "type": "web_search_20250305",
              "name": "web_search",
              "max_uses": 5,
          }
      ],
      betas=["context-management-2025-06-27"],
      context_management={
          "edits": [
              {
                  "type": "clear_thinking_20251015",
                  "keep": {"type": "thinking_turns", "value": 2},
              },
              {
                  "type": "clear_tool_uses_20250919",
                  "trigger": {"type": "input_tokens", "value": 50000},
                  "keep": {"type": "tool_uses", "value": 5},
              },
          ]
      },
  )

  print(response)
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 16000,
    messages: [
      {
        role: "user",
        content:
          "Search for the latest developments in quantum error correction and summarize the key breakthroughs."
      }
    ],
    tools: [
      {
        type: "web_search_20250305",
        name: "web_search",
        max_uses: 5
      }
    ],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_thinking_20251015",
          keep: {
            type: "thinking_turns",
            value: 2
          }
        },
        {
          type: "clear_tool_uses_20250919",
          trigger: {
            type: "input_tokens",
            value: 50000
          },
          keep: {
            type: "tool_uses",
            value: 5
          }
        }
      ]
    }
  });

  console.log(response);
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5,
      MaxTokens = 16000,
      Messages = [
          new() { Role = Role.User, Content = "Search for the latest developments in quantum error correction and summarize the key breakthroughs." }
      ],
      Tools = [
          new BetaWebSearchTool20250305 { MaxUses = 5 }
      ],
      Betas = [AnthropicBeta.ContextManagement2025_06_27],
      ContextManagement = new BetaContextManagementConfig
      {
          Edits = [
              new BetaClearThinking20251015Edit
              {
                  Keep = new BetaThinkingTurns(2)
              },
              new BetaClearToolUses20250919Edit
              {
                  Trigger = new BetaInputTokensTrigger(50000),
                  Keep = new BetaToolUsesKeep(5)
              }
          ]
      }
  };

  var response = await client.Beta.Messages.Create(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 16000,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Search for the latest developments in quantum error correction and summarize the key breakthroughs.")),
  	},
  	Tools: []anthropic.BetaToolUnionParam{
  		{OfWebSearchTool20250305: &anthropic.BetaWebSearchTool20250305Param{
  			MaxUses: anthropic.Int(5),
  		}},
  	},
  	Betas: []anthropic.AnthropicBeta{
  		anthropic.AnthropicBetaContextManagement2025_06_27,
  	},
  	ContextManagement: anthropic.BetaContextManagementConfigParam{
  		Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
  			{OfClearThinking20251015: &anthropic.BetaClearThinking20251015EditParam{
  				Keep: anthropic.BetaClearThinking20251015EditKeepUnionParam{
  					OfThinkingTurns: &anthropic.BetaThinkingTurnsParam{
  						Value: 2,
  					},
  				},
  			}},
  			{OfClearToolUses20250919: &anthropic.BetaClearToolUses20250919EditParam{
  				Trigger: anthropic.BetaClearToolUses20250919EditTriggerUnionParam{
  					OfInputTokens: &anthropic.BetaInputTokensTriggerParam{
  						Value: 50000,
  					},
  				},
  				Keep: anthropic.BetaToolUsesKeepParam{
  					Value: 5,
  				},
  			}},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.beta.messages.BetaWebSearchTool20250305;
  import com.anthropic.models.beta.messages.BetaContextManagementConfig;
  import com.anthropic.models.beta.messages.BetaClearThinking20251015Edit;
  import com.anthropic.models.beta.messages.BetaClearToolUses20250919Edit;
  import com.anthropic.models.beta.messages.BetaThinkingTurns;
  import com.anthropic.models.beta.messages.BetaInputTokensTrigger;
  import com.anthropic.models.beta.messages.BetaToolUsesKeep;
  import com.anthropic.models.beta.AnthropicBeta;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(16000L)
          .addUserMessage("Search for the latest developments in quantum error correction and summarize the key breakthroughs.")
          .addTool(BetaWebSearchTool20250305.builder()
              .maxUses(5L)
              .build())
          .addBeta(AnthropicBeta.CONTEXT_MANAGEMENT_2025_06_27)
          .contextManagement(BetaContextManagementConfig.builder()
              .addEdit(BetaClearThinking20251015Edit.builder()
                  .keep(BetaThinkingTurns.builder()
                      .value(2L)
                      .build())
                  .build())
              .addEdit(BetaClearToolUses20250919Edit.builder()
                  .trigger(BetaInputTokensTrigger.builder()
                      .value(50000L)
                      .build())
                  .keep(BetaToolUsesKeep.builder()
                      .value(5L)
                      .build())
                  .build())
              .build())
          .build();

      BetaMessage response = client.beta().messages().create(params);
      IO.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      maxTokens: 16000,
      messages: [
          [
              'role' => 'user',
              'content' => 'Search for the latest developments in quantum error correction and summarize the key breakthroughs.'
          ]
      ],
      model: 'claude-opus-5',
      betas: ['context-management-2025-06-27'],
      tools: [
          [
              'type' => 'web_search_20250305',
              'name' => 'web_search',
              'max_uses' => 5
          ]
      ],
      contextManagement: [
          'edits' => [
              [
                  'type' => 'clear_thinking_20251015',
                  'keep' => [
                      'type' => 'thinking_turns',
                      'value' => 2
                  ]
              ],
              [
                  'type' => 'clear_tool_uses_20250919',
                  'trigger' => [
                      'type' => 'input_tokens',
                      'value' => 50000
                  ],
                  'keep' => [
                      'type' => 'tool_uses',
                      'value' => 5
                  ]
              ]
          ]
      ],
  );

  echo $response;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 16000,
    messages: [
      {
        role: "user",
        content: "Search for the latest developments in quantum error correction and summarize the key breakthroughs."
      }
    ],
    tools: [
      {
        type: "web_search_20250305",
        name: "web_search",
        max_uses: 5
      }
    ],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_thinking_20251015",
          keep: {
            type: "thinking_turns",
            value: 2
          }
        },
        {
          type: "clear_tool_uses_20250919",
          trigger: {
            type: "input_tokens",
            value: 50000
          },
          keep: {
            type: "tool_uses",
            value: 5
          }
        }
      ]
    }
  )
  puts response
  ```
</CodeGroup>

## 工具结果清除的配置选项

| 配置选项                | 默认值           | 描述                                                                        |
| ------------------- | ------------- | ------------------------------------------------------------------------- |
| `trigger`           | 100,000 个输入令牌 | 定义上下文编辑策略何时激活。一旦提示超过此阈值，清除即开始。您可以用 `input_tokens` 或 `tool_uses` 来指定此值。    |
| `keep`              | 3 次工具使用       | 定义清除发生后要保留多少个最近的工具使用/结果对。API 会首先移除最旧的工具交互，保留最近的交互。                        |
| `clear_at_least`    | 无             | 确保每次策略激活时至少清除一定数量的令牌。如果 API 无法清除至少指定的数量，则不会应用该策略。这有助于判断上下文清除是否值得破坏您的提示缓存。 |
| `exclude_tools`     | 无             | 工具名称列表，这些工具的工具使用和结果永远不应被清除。适用于保留重要上下文。                                    |
| `clear_tool_inputs` | `false`       | 控制是否将工具调用参数与工具结果一起清除。默认情况下，仅清除工具结果，同时保持 Claude 原始的工具调用可见。                 |

## 上下文编辑响应

您可以通过 `context_management` 响应字段查看对您的请求应用了哪些上下文编辑，以及有关已清除内容和输入令牌的有用统计信息。

```json Output
{
  "id": "msg_013Zva2CMHLNnXjNJJKqJ2EF",
  "type": "message",
  "role": "assistant",
  "content": [
    // ...
  ],
  "usage": {
    // ...
  },
  "context_management": {
    "applied_edits": [
      // When using `clear_thinking_20251015`
      {
        "type": "clear_thinking_20251015",
        "cleared_thinking_turns": 3,
        "cleared_input_tokens": 15000
      },
      // When using `clear_tool_uses_20250919`
      {
        "type": "clear_tool_uses_20250919",
        "cleared_tool_uses": 8,
        "cleared_input_tokens": 50000
      }
    ]
  }
}
```

对于流式传输响应，上下文编辑包含在最终的 `message_delta` 事件中：

```json Streaming Response
{
  "type": "message_delta",
  "delta": {
    "stop_reason": "end_turn",
    "stop_sequence": null
  },
  "usage": {
    "output_tokens": 1024
  },
  "context_management": {
    "applied_edits": [
      // ...
    ]
  }
}
```

## 令牌计数

[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点支持上下文管理，让您可以预览在应用上下文编辑后您的提示将使用多少令牌。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages/count_tokens \
      --header "x-api-key: $ANTHROPIC_API_KEY" \
      --header "anthropic-version: 2023-06-01" \
      --header "content-type: application/json" \
      --header "anthropic-beta: context-management-2025-06-27" \
      --data '{
          "model": "claude-opus-5",
          "messages": [
              {
                  "role": "user",
                  "content": "Continue our conversation..."
              }
          ],
          "context_management": {
              "edits": [
                  {
                      "type": "clear_tool_uses_20250919",
                      "trigger": {
                          "type": "input_tokens",
                          "value": 30000
                      },
                      "keep": {
                          "type": "tool_uses",
                          "value": 5
                      }
                  }
              ]
          }
      }'
  ```

  ```bash CLI
  cat > request.yaml <<'YAML'
  model: claude-opus-5
  messages:
    - role: user
      content: Continue our conversation...
  context_management:
    edits:
      - type: clear_tool_uses_20250919
        trigger:
          type: input_tokens
          value: 30000
        keep:
          type: tool_uses
          value: 5
  YAML

  ORIGINAL=$(ant beta:messages count-tokens \
    --beta context-management-2025-06-27 \
    --transform context_management.original_input_tokens \
    --raw-output < request.yaml)

  INPUT_TOKENS=$(ant beta:messages count-tokens \
    --beta context-management-2025-06-27 \
    --transform input_tokens --raw-output < request.yaml)

  printf 'Original tokens: %s\n' "$ORIGINAL"
  printf 'After clearing: %s\n' "$INPUT_TOKENS"
  printf 'Savings: %s tokens\n' "$((ORIGINAL - INPUT_TOKENS))"
  ```

  ```python Python
  response = client.beta.messages.count_tokens(
      model="claude-opus-5",
      messages=[{"role": "user", "content": "Continue our conversation..."}],
      betas=["context-management-2025-06-27"],
      context_management={
          "edits": [
              {
                  "type": "clear_tool_uses_20250919",
                  "trigger": {"type": "input_tokens", "value": 30000},
                  "keep": {"type": "tool_uses", "value": 5},
              }
          ]
      },
  )

  print(f"Original tokens: {response.context_management.original_input_tokens}")
  print(f"After clearing: {response.input_tokens}")
  print(
      f"Savings: {response.context_management.original_input_tokens - response.input_tokens} tokens"
  )
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  const response = await anthropic.beta.messages.countTokens({
    model: "claude-opus-5",
    messages: [
      {
        role: "user",
        content: "Continue our conversation..."
      }
    ],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_tool_uses_20250919",
          trigger: {
            type: "input_tokens",
            value: 30000
          },
          keep: {
            type: "tool_uses",
            value: 5
          }
        }
      ]
    }
  });

  console.log(`Original tokens: ${response.context_management?.original_input_tokens}`);
  console.log(`After clearing: ${response.input_tokens}`);
  console.log(
    `Savings: ${
      (response.context_management?.original_input_tokens || 0) - response.input_tokens
    } tokens`
  );
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCountTokensParams
  {
      Model = Messages::Model.ClaudeOpus5,
      Messages = [new() { Role = Role.User, Content = "Continue our conversation..." }],
      Betas = [AnthropicBeta.ContextManagement2025_06_27],
      ContextManagement = new BetaContextManagementConfig
      {
          Edits = [
              new BetaClearToolUses20250919Edit
              {
                  Trigger = new BetaInputTokensTrigger(30000),
                  Keep = new BetaToolUsesKeep(5)
              }
          ]
      }
  };

  var response = await client.Beta.Messages.CountTokens(parameters);

  Console.WriteLine($"Original tokens: {response.ContextManagement?.OriginalInputTokens}");
  Console.WriteLine($"After clearing: {response.InputTokens}");
  Console.WriteLine($"Savings: {(response.ContextManagement?.OriginalInputTokens ?? 0) - response.InputTokens} tokens");
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.CountTokens(context.TODO(), anthropic.BetaMessageCountTokensParams{
  	Model: anthropic.ModelClaudeOpus5,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Continue our conversation...")),
  	},
  	Betas: []anthropic.AnthropicBeta{
  		anthropic.AnthropicBetaContextManagement2025_06_27,
  	},
  	ContextManagement: anthropic.BetaContextManagementConfigParam{
  		Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
  			{OfClearToolUses20250919: &anthropic.BetaClearToolUses20250919EditParam{
  				Trigger: anthropic.BetaClearToolUses20250919EditTriggerUnionParam{
  					OfInputTokens: &anthropic.BetaInputTokensTriggerParam{
  						Value: 30000,
  					},
  				},
  				Keep: anthropic.BetaToolUsesKeepParam{
  					Value: 5,
  				},
  			}},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("Original tokens: %d\n", response.ContextManagement.OriginalInputTokens)
  fmt.Printf("After clearing: %d\n", response.InputTokens)
  fmt.Printf("Savings: %d tokens\n", response.ContextManagement.OriginalInputTokens-response.InputTokens)
  ```

  ```java Java
  import com.anthropic.models.beta.messages.BetaMessageTokensCount;
  import com.anthropic.models.beta.messages.MessageCountTokensParams;
  import com.anthropic.models.beta.messages.BetaContextManagementConfig;
  import com.anthropic.models.beta.messages.BetaClearToolUses20250919Edit;
  import com.anthropic.models.beta.messages.BetaInputTokensTrigger;
  import com.anthropic.models.beta.messages.BetaToolUsesKeep;
  import com.anthropic.models.beta.AnthropicBeta;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCountTokensParams params = MessageCountTokensParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .addUserMessage("Continue our conversation...")
          .addBeta(AnthropicBeta.CONTEXT_MANAGEMENT_2025_06_27)
          .contextManagement(BetaContextManagementConfig.builder()
              .addEdit(BetaClearToolUses20250919Edit.builder()
                  .trigger(BetaInputTokensTrigger.builder()
                      .value(30000L)
                      .build())
                  .keep(BetaToolUsesKeep.builder()
                      .value(5L)
                      .build())
                  .build())
              .build())
          .build();

      BetaMessageTokensCount response = client.beta().messages().countTokens(params);

      IO.println("Original tokens: " + response.contextManagement().get().originalInputTokens());
      IO.println("After clearing: " + response.inputTokens());
      IO.println("Savings: " + (response.contextManagement().get().originalInputTokens() - response.inputTokens()) + " tokens");
  }
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->countTokens(
      messages: [
          ['role' => 'user', 'content' => 'Continue our conversation...']
      ],
      model: 'claude-opus-5',
      betas: ['context-management-2025-06-27'],
      contextManagement: [
          'edits' => [
              [
                  'type' => 'clear_tool_uses_20250919',
                  'trigger' => [
                      'type' => 'input_tokens',
                      'value' => 30000
                  ],
                  'keep' => [
                      'type' => 'tool_uses',
                      'value' => 5
                  ]
              ]
          ]
      ],
  );

  echo "Original tokens: " . $response->contextManagement->originalInputTokens . "\n";
  echo "After clearing: " . $response->inputTokens . "\n";
  echo "Savings: " . ($response->contextManagement->originalInputTokens - $response->inputTokens) . " tokens\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.count_tokens(
    model: "claude-opus-5",
    messages: [
      { role: "user", content: "Continue our conversation..." }
    ],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        {
          type: "clear_tool_uses_20250919",
          trigger: {
            type: "input_tokens",
            value: 30000
          },
          keep: {
            type: "tool_uses",
            value: 5
          }
        }
      ]
    }
  )

  puts "Original tokens: #{response.context_management.original_input_tokens}"
  puts "After clearing: #{response.input_tokens}"
  puts "Savings: #{response.context_management.original_input_tokens - response.input_tokens} tokens"
  ```
</CodeGroup>

```json Output
{
  "input_tokens": 25000,
  "context_management": {
    "original_input_tokens": 70000
  }
}
```

响应同时显示应用上下文管理后的最终令牌计数（`input_tokens`）和任何清除发生之前的原始令牌计数（`original_input_tokens`）。

## 与记忆工具配合使用

上下文编辑可以与[记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)结合使用。当您的对话上下文接近配置的清除阈值时，Claude 会收到自动警告以保存重要信息。这使 Claude 能够在工具结果或上下文从对话历史中被清除之前，将其保存到记忆文件中。

这种组合让您能够：

* **保留重要上下文：** Claude 可以在工具结果被清除之前，将其中的关键信息写入记忆文件
* **维持长时间运行的工作流：** 通过将信息卸载到持久存储，使原本会超出上下文限制的智能体工作流得以运行
* **按需访问信息：** Claude 可以在需要时从记忆文件中查找先前已清除的信息，而不必将所有内容都保留在活动的上下文窗口中

例如，在 Claude 执行大量操作的文件编辑工作流中，随着上下文的增长，Claude 可以将已完成的更改汇总到记忆文件中。当工具结果被清除时，Claude 仍可通过其记忆系统访问这些信息，并能继续有效地工作。

要同时使用这两项功能，请在您的 API 请求中启用它们：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
      --header "x-api-key: $ANTHROPIC_API_KEY" \
      --header "anthropic-version: 2023-06-01" \
      --header "content-type: application/json" \
      --header "anthropic-beta: context-management-2025-06-27" \
      --data '{
          "model": "claude-opus-5",
          "max_tokens": 4096,
          "messages": [
              {
                  "role": "user",
                  "content": "Hello"
              }
          ],
          "tools": [
              {
                  "type": "memory_20250818",
                  "name": "memory"
              }
          ],
          "context_management": {
              "edits": [
                  {"type": "clear_tool_uses_20250919"}
              ]
          }
      }'
  ```

  ```bash CLI
  ant beta:messages create --beta context-management-2025-06-27 <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  messages:
    - role: user
      content: Hello
  tools:
    - type: memory_20250818
      name: memory
  context_management:
    edits:
      - type: clear_tool_uses_20250919
  YAML
  ```

  ```python Python
  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      messages=[{"role": "user", "content": "Hello"}],
      tools=[{"type": "memory_20250818", "name": "memory"}],
      betas=["context-management-2025-06-27"],
      context_management={"edits": [{"type": "clear_tool_uses_20250919"}]},
  )
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    messages: [{ role: "user", content: "Hello" }],
    tools: [
      {
        type: "memory_20250818",
        name: "memory"
      }
    ],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [{ type: "clear_tool_uses_20250919" }]
    }
  });
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5,
      MaxTokens = 4096,
      Messages = [
          new() { Role = Role.User, Content = "Hello" }
      ],
      Tools = [
          new BetaMemoryTool20250818()
      ],
      Betas = [AnthropicBeta.ContextManagement2025_06_27],
      ContextManagement = new BetaContextManagementConfig
      {
          Edits = [new BetaClearToolUses20250919Edit()]
      }
  };

  var response = await client.Beta.Messages.Create(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 4096,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello")),
  	},
  	Tools: []anthropic.BetaToolUnionParam{
  		{OfMemoryTool20250818: &anthropic.BetaMemoryTool20250818Param{}},
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaContextManagement2025_06_27},
  	ContextManagement: anthropic.BetaContextManagementConfigParam{
  		Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
  			{OfClearToolUses20250919: &anthropic.BetaClearToolUses20250919EditParam{}},
  		},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.beta.messages.BetaMemoryTool20250818;
  import com.anthropic.models.beta.messages.BetaContextManagementConfig;
  import com.anthropic.models.beta.messages.BetaClearToolUses20250919Edit;
  import com.anthropic.models.beta.AnthropicBeta;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .addUserMessage("Hello")
          .addTool(BetaMemoryTool20250818.builder().build())
          .addBeta(AnthropicBeta.CONTEXT_MANAGEMENT_2025_06_27)
          .contextManagement(BetaContextManagementConfig.builder()
              .addEdit(BetaClearToolUses20250919Edit.builder().build())
              .build())
          .build();

      BetaMessage response = client.beta().messages().create(params);
      IO.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Hello']
      ],
      model: 'claude-opus-5',
      betas: ['context-management-2025-06-27'],
      tools: [
          [
              'type' => 'memory_20250818',
              'name' => 'memory'
          ]
      ],
      contextManagement: [
          'edits' => [
              ['type' => 'clear_tool_uses_20250919']
          ]
      ],
  );

  echo $response;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    messages: [{ role: "user", content: "Hello" }],
    tools: [
      {
        type: "memory_20250818",
        name: "memory"
      }
    ],
    betas: ["context-management-2025-06-27"],
    context_management: {
      edits: [
        { type: "clear_tool_uses_20250919" }
      ]
    }
  )
  puts response
  ```
</CodeGroup>

有关包括命令和示例在内的完整记忆工具参考，请参阅[记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)。

## 客户端压缩（SDK）

<Warning>
  **Anthropic 推荐使用服务端压缩而非 SDK 压缩。** [服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)可自动处理上下文管理，集成复杂度更低、令牌用量计算更准确，且没有客户端限制。仅当您明确需要在客户端控制摘要过程时才使用 SDK 压缩。

  `compaction_control` 参数在 TypeScript 和 Ruby SDK 中已弃用，并将在未来版本中移除。启用该参数时，SDK 会发出弃用警告。Python SDK 已在 v1.0 中将其移除。要在工具运行器中使用服务端压缩，请在请求的 `context_management` 参数中传入 `compact_20260112` 编辑。
</Warning>

<Note>
  在使用 [`tool_runner` 方法](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner)时，压缩功能可在 [TypeScript 和 Ruby SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview) 中使用。
</Note>

"Compaction"（压缩）是一项 SDK 功能，它通过在令牌用量增长过大时生成摘要来自动管理对话上下文。与清除内容的服务端上下文编辑策略不同，压缩会指示 Claude 对对话历史进行摘要，然后用该摘要替换完整的历史记录。这使 Claude 能够继续处理原本会超出[上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)的长时间运行任务。

### 压缩的工作原理

启用压缩后，SDK 会在每次模型响应后监控令牌用量：

1. **阈值检查：** SDK 将总令牌数计算为 `input_tokens + cache_creation_input_tokens + cache_read_input_tokens + output_tokens`（有关缓存令牌字段，请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)）。
2. **摘要生成：** 当超过阈值时，会以用户轮次的形式注入一个摘要提示，Claude 随后生成一个包裹在 `<summary></summary>` 标签中的结构化摘要。
3. **上下文替换：** SDK 提取摘要并用它替换整个消息历史。
4. **继续：** 对话从摘要处恢复，Claude 从中断的地方继续。

### 使用压缩

在您的 `tool_runner` 调用中添加 `compaction_control`，即可在令牌用量超过阈值时启用自动摘要。

<Tabs>
  <Tab title="cURL">
    <Note>
      压缩在 SDK 的 `tool_runner` 辅助工具中于客户端运行，因此没有直接对应的 HTTP 等效方式。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩。
    </Note>
  </Tab>

  <Tab title="CLI">
    <Note>
      CLI 不包含 `tool_runner` 辅助工具。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩，无需 SDK 端集成。
    </Note>
  </Tab>

  <Tab title="Python">
    <Note>
      在 v1.0 及更高版本中，Python SDK 的工具运行器不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="TypeScript">
    ```typescript TypeScript
    const client = new Anthropic();

    const runner = client.beta.messages.toolRunner({
      model: "claude-opus-5",
      max_tokens: 1024,
      tools: [readFile],
      messages: [{ role: "user", content: "What's in config.json?" }],
      compactionControl: { enabled: true, contextTokenThreshold: 100000 }
    });

    for await (const message of runner) {
      console.log(`Tokens used: ${message.usage.input_tokens}`);
    }
    ```
  </Tab>

  <Tab title="C#">
    <Note>
      C# SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Go">
    <Note>
      Go SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Java">
    <Note>
      Java SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="PHP">
    <Note>
      PHP SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Ruby">
    ```ruby Ruby
    client = Anthropic::Client.new

    runner = client.beta.messages.tool_runner(
      model: "claude-opus-5",
      max_tokens: 1024,
      tools: [ReadFile.new],
      messages: [{ role: "user", content: "What's in config.json?" }],
      compaction_control: { enabled: true, context_token_threshold: 100000 }
    )

    runner.each_message do |message|
      puts "Tokens used: #{message.usage.input_tokens}"
    end
    ```
  </Tab>
</Tabs>

#### 压缩期间会发生什么

随着对话的增长，消息历史不断累积：

**压缩前（接近 100k 令牌）：**

```json
[
  { "role": "user", "content": "Analyze all files and write a report..." },
  { "role": "assistant", "content": "I'll help. Let me start by reading..." },
  {
    "role": "user",
    "content": [{ "type": "tool_result", "tool_use_id": "...", "content": "..." }]
  },
  { "role": "assistant", "content": "Based on file1.txt, I see..." },
  {
    "role": "user",
    "content": [{ "type": "tool_result", "tool_use_id": "...", "content": "..." }]
  },
  { "role": "assistant", "content": "After analyzing file2.txt..." }
  // ... 50 more exchanges like this ...
]
```

当令牌数超过阈值时，SDK 会注入一个摘要请求，Claude 随后生成摘要。然后整个历史记录被替换：

**压缩后（回到约 2–3k 令牌）：**

```json
[
  {
    "role": "assistant",
    "content": "# Task Overview\nThe user requested analysis of directory files to produce a summary report...\n\n# Current State\nAnalyzed 52 files across 3 subdirectories. Key findings documented in report.md...\n\n# Important Discoveries\n- Configuration files use YAML format\n- Found 3 deprecated dependencies\n- Test coverage at 67%\n\n# Next Steps\n1. Analyze remaining files in /src/legacy\n2. Complete final report sections...\n\n# Context to Preserve\nUser prefers markdown format with executive summary first..."
  }
]
```

Claude 会基于此摘要继续工作，就如同它是原始的对话历史一样。

### 配置选项

| 参数                        | 类型      | 必需 | 默认值                                                                                                          | 描述           |
| ------------------------- | ------- | -- | ------------------------------------------------------------------------------------------------------------ | ------------ |
| `enabled`                 | boolean | 是  | -                                                                                                            | 是否启用自动压缩     |
| `context_token_threshold` | number  | 否  | 100,000                                                                                                      | 触发压缩的令牌数     |
| `model`                   | string  | 否  | 与主模型相同                                                                                                       | 用于生成摘要的模型    |
| `summary_prompt`          | string  | 否  | 请参阅[默认摘要提示](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#default-summary-prompt) | 用于生成摘要的自定义提示 |

#### 选择令牌阈值

阈值决定压缩何时发生。较低的阈值意味着更频繁的压缩和更小的上下文窗口。较高的阈值允许更多上下文，但有触及限制的风险。

<Tabs>
  <Tab title="cURL">
    <Note>
      压缩在 SDK 的 `tool_runner` 辅助工具中于客户端运行，因此没有直接对应的 HTTP 等效方式。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩。
    </Note>
  </Tab>

  <Tab title="CLI">
    <Note>
      CLI 不包含 `tool_runner` 辅助工具。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩，无需 SDK 端集成。
    </Note>
  </Tab>

  <Tab title="Python">
    <Note>
      在 v1.0 及更高版本中，Python SDK 的工具运行器不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="TypeScript">
    ```typescript TypeScript
    const client = new Anthropic();

    const runner = client.beta.messages.toolRunner({
      model: "claude-opus-5",
      max_tokens: 1024,
      tools: [readFile],
      messages: [{ role: "user", content: "What's in config.json?" }],
      // 值越低压缩越频繁；任务需要更多上下文时可提高到 150000
      compactionControl: { enabled: true, contextTokenThreshold: 50000 }
    });

    for await (const message of runner) {
      console.log(`Tokens used: ${message.usage.input_tokens}`);
    }
    ```
  </Tab>

  <Tab title="C#">
    <Note>
      C# SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Go">
    <Note>
      Go SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Java">
    <Note>
      Java SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="PHP">
    <Note>
      PHP SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Ruby">
    ```ruby Ruby
    client = Anthropic::Client.new

    runner = client.beta.messages.tool_runner(
      model: "claude-opus-5",
      max_tokens: 1024,
      tools: [ReadFile.new],
      messages: [{ role: "user", content: "What's in config.json?" }],
      # 较低的值会更频繁地压缩；当任务需要更多上下文时可提高到 150000
      compaction_control: { enabled: true, context_token_threshold: 50000 }
    )

    runner.each_message do |message|
      puts "Tokens used: #{message.usage.input_tokens}"
    end
    ```
  </Tab>
</Tabs>

#### 使用不同的模型生成摘要

您可以使用更快或更便宜的模型来生成摘要：

<Tabs>
  <Tab title="cURL">
    <Note>
      压缩在 SDK 的 `tool_runner` 辅助工具中于客户端运行，因此没有直接对应的 HTTP 等效方式。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩。
    </Note>
  </Tab>

  <Tab title="CLI">
    <Note>
      CLI 不包含 `tool_runner` 辅助工具。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩，无需 SDK 端集成。
    </Note>
  </Tab>

  <Tab title="Python">
    <Note>
      在 v1.0 及更高版本中，Python SDK 的工具运行器不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="TypeScript">
    ```typescript TypeScript
    const client = new Anthropic();

    const runner = client.beta.messages.toolRunner({
      model: "claude-opus-5",
      max_tokens: 1024,
      tools: [readFile],
      messages: [{ role: "user", content: "What's in config.json?" }],
      compactionControl: {
        enabled: true,
        contextTokenThreshold: 100000,
        model: "claude-haiku-4-5"
      }
    });

    for await (const message of runner) {
      console.log(`Tokens used: ${message.usage.input_tokens}`);
    }
    ```
  </Tab>

  <Tab title="C#">
    <Note>
      C# SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Go">
    <Note>
      Go SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Java">
    <Note>
      Java SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="PHP">
    <Note>
      PHP SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Ruby">
    ```ruby Ruby
    client = Anthropic::Client.new

    runner = client.beta.messages.tool_runner(
      model: "claude-opus-5",
      max_tokens: 1024,
      tools: [ReadFile.new],
      messages: [{ role: "user", content: "What's in config.json?" }],
      compaction_control: {
        enabled: true,
        context_token_threshold: 100000,
        model: "claude-haiku-4-5"
      }
    )

    runner.each_message do |message|
      puts "Tokens used: #{message.usage.input_tokens}"
    end
    ```
  </Tab>
</Tabs>

#### 自定义摘要提示

您可以针对特定领域的需求提供自定义提示。您的提示应指示 Claude 将其摘要包裹在 `<summary></summary>` 标签中。

<Tabs>
  <Tab title="cURL">
    <Note>
      压缩在 SDK 的 `tool_runner` 辅助工具中于客户端运行，因此没有直接对应的 HTTP 等效方式。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩。
    </Note>
  </Tab>

  <Tab title="CLI">
    <Note>
      CLI 不包含 `tool_runner` 辅助工具。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩，无需 SDK 端集成。
    </Note>
  </Tab>

  <Tab title="Python">
    <Note>
      在 v1.0 及更高版本中，Python SDK 的工具运行器不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="TypeScript">
    ```typescript TypeScript
    const client = new Anthropic();

    const runner = client.beta.messages.toolRunner({
      model: "claude-opus-5",
      max_tokens: 1024,
      tools: [readFile],
      messages: [{ role: "user", content: "What's in config.json?" }],
      compactionControl: {
        enabled: true,
        contextTokenThreshold: 100000,
        summaryPrompt: `Summarize the research conducted so far, including:
    - Sources consulted and key findings
    - Questions answered and remaining unknowns
    - Recommended next steps

    Wrap your summary in <summary></summary> tags.`
      }
    });

    for await (const message of runner) {
      console.log(`Tokens used: ${message.usage.input_tokens}`);
    }
    ```
  </Tab>

  <Tab title="C#">
    <Note>
      C# SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Go">
    <Note>
      Go SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Java">
    <Note>
      Java SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="PHP">
    <Note>
      PHP SDK 包含工具运行器，但不支持客户端 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)：通过在请求的 `context_management` 参数中传入 `compact_20260112` 编辑，它可以与工具运行器配合使用。
    </Note>
  </Tab>

  <Tab title="Ruby">
    ```ruby Ruby
    client = Anthropic::Client.new

    runner = client.beta.messages.tool_runner(
      model: "claude-opus-5",
      max_tokens: 1024,
      tools: [ReadFile.new],
      messages: [{ role: "user", content: "What's in config.json?" }],
      compaction_control: {
        enabled: true,
        context_token_threshold: 100000,
        summary_prompt: <<~PROMPT
          Summarize the research conducted so far, including:
          - Sources consulted and key findings
          - Questions answered and remaining unknowns
          - Recommended next steps

          Wrap your summary in <summary></summary> tags.
        PROMPT
      }
    )

    runner.each_message do |message|
      puts "Tokens used: #{message.usage.input_tokens}"
    end
    ```
  </Tab>
</Tabs>

### 默认摘要提示

内置的摘要提示会指示 Claude 创建一个结构化的延续摘要，其中包括：

1. **任务概述：** 用户的核心请求、成功标准和约束条件。
2. **当前状态：** 已完成的内容、已修改的文件和已生成的产出物。
3. **重要发现：** 技术约束、已做出的决策、已解决的错误以及失败的方法。
4. **后续步骤：** 所需的具体操作、阻碍因素和优先级顺序。
5. **需保留的上下文：** 用户偏好、特定领域的细节以及已做出的承诺。

这种结构使 Claude 能够高效地恢复工作，而不会丢失重要上下文或重复犯错。

<Accordion title="查看完整的默认提示">
  ```text wrap
  You have been working on the task described above but have not yet completed it. Write a continuation summary that will allow you (or another instance of yourself) to resume work efficiently in a future context window where the conversation history will be replaced with this summary. Your summary should be structured, concise, and actionable. Include:

  1. Task Overview
  The user's core request and success criteria
  Any clarifications or constraints they specified

  2. Current State
  What has been completed so far
  Files created, modified, or analyzed (with paths if relevant)
  Key outputs or artifacts produced

  3. Important Discoveries
  Technical constraints or requirements uncovered
  Decisions made and their rationale
  Errors encountered and how they were resolved
  What approaches were tried that didn't work (and why)

  4. Next Steps
  Specific actions needed to complete the task
  Any blockers or open questions to resolve
  Priority order if multiple steps remain

  5. Context to Preserve
  User preferences or style requirements
  Domain-specific details that aren't obvious
  Any promises made to the user

  Be concise but complete—err on the side of including information that would prevent duplicate work or repeated mistakes. Write in a way that enables immediate resumption of the task.

  Wrap your summary in <summary></summary> tags.
  ```
</Accordion>

### 限制

#### 服务端工具

<Warning>
  在使用[网页搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)或[网页抓取](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)等服务端工具时，压缩需要特别注意。
</Warning>

使用服务端工具时，SDK 可能会错误地计算令牌用量，导致压缩在错误的时间触发。

例如，在一次网页搜索操作之后，API 响应可能显示：

```json Output
{
  "usage": {
    "input_tokens": 63000,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 270000,
    "output_tokens": 1400
  }
}
```

SDK 将总用量计算为 63,000 + 0 + 270,000 + 1,400 = 334,400 个令牌。然而，`cache_read_input_tokens` 的值包含了服务端工具进行的多次内部 API 调用所累积的读取量，而非您实际的对话上下文。您真实的上下文长度可能只有 63,000 个 `input_tokens`，但 SDK 看到的是 334k，于是过早地触发了压缩。

**解决方法：**

* 使用[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点获取准确的上下文长度
* 在大量使用服务端工具时避免使用压缩

#### 工具使用的边界情况

当 SDK 在某个工具使用响应尚未完成时触发压缩，它会在生成摘要之前从消息历史中移除该工具使用块。如果仍有需要，Claude 会在从摘要恢复后重新发起该工具调用。

### 监控压缩

了解压缩何时触发有助于您调整阈值并验证预期行为。

<Tabs>
  <Tab title="cURL">
    <Note>
      压缩在 SDK 的 `tool_runner` 辅助工具中于客户端运行，因此没有直接对应的 HTTP 等效方式。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩。
    </Note>
  </Tab>

  <Tab title="CLI">
    <Note>
      CLI 不包含 `tool_runner` 辅助工具。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，它在 Anthropic 的服务器上处理压缩，无需 SDK 端集成。
    </Note>
  </Tab>

  <Tab title="Python">
    <Note>
      在 v1.0 及更高版本中，Python SDK 的工具运行器不支持 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)。
    </Note>
  </Tab>

  <Tab title="TypeScript">
    TypeScript SDK 的 `toolRunner` 支持压缩，但不会记录事件。可通过观察 `runner.params.messages.length` 在轮次之间缩减来检测压缩：

    ```typescript TypeScript
    let prevMsgCount = 0;
    for await (const message of runner) {
      const currMsgCount = runner.params.messages.length;
      if (currMsgCount < prevMsgCount) {
        console.log(`Compaction occurred: ${prevMsgCount} -> ${currMsgCount} messages`);
        console.log(`Input tokens after compaction: ${message.usage.input_tokens}`);
      }
      prevMsgCount = currMsgCount;
    }
    ```
  </Tab>

  <Tab title="C#">
    <Note>
      C# SDK 的工具运行器不支持 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)。
    </Note>
  </Tab>

  <Tab title="Go">
    <Note>
      Go SDK 的工具运行器不支持 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)。
    </Note>
  </Tab>

  <Tab title="Java">
    <Note>
      Java SDK 的工具运行器不支持 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)。
    </Note>
  </Tab>

  <Tab title="PHP">
    <Note>
      PHP SDK 的工具运行器不支持 `compaction_control`。请改用[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)。
    </Note>
  </Tab>

  <Tab title="Ruby">
    Ruby SDK 支持 `on_compact:` 回调，该回调会在压缩发生时触发。将其添加到您的 `compaction_control` 配置中：

    ```ruby Ruby
    client = Anthropic::Client.new

    runner = client.beta.messages.tool_runner(
      model: "claude-opus-5",
      max_tokens: 1024,
      tools: [ReadFile.new],
      messages: [{ role: "user", content: "What's in config.json?" }],
      compaction_control: {
        enabled: true,
        context_token_threshold: 100000,
        on_compact: ->(tokens_before, tokens_after) do
          puts "Compaction occurred: #{tokens_before} -> #{tokens_after} tokens"
        end
      }
    )

    runner.each_message do |message|
      puts "Tokens: #{message.usage.input_tokens}"
    end
    ```
  </Tab>
</Tabs>

### 何时使用压缩

**适合的用例：**

* 处理大量文件或数据源的长时间运行智能体任务
* 积累大量信息的研究工作流
* 具有清晰、可衡量进度的多步骤任务
* 会产生在对话之外持久存在的产出物（文件、报告）的任务

**不太理想的用例：**

* 需要精确回忆对话早期细节的任务
* 大量使用服务端工具的工作流
* 需要在众多变量之间维持精确状态的任务

## 后续步骤

<CardGroup cols={2}>
  <Card title="压缩" icon="arrows-clockwise" href="https://platform.claude.com/docs/zh-CN/build-with-claude/compaction">
    使用服务端压缩管理长对话，这是大多数用例的推荐策略。
  </Card>

  <Card title="提示缓存" icon="database" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching">
    通过缓存提示前缀来降低成本和延迟，并了解上下文编辑如何与缓存交互。
  </Card>
</CardGroup>
