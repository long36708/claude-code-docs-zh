---
title: 迁移到 Claude Fable 5.1 和 Claude Mythos 5.1
url: https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide
description: 从 Claude Fable 5、Claude Mythos 5、Claude Opus 5 或 Claude Opus 4.8 迁移到 Claude Fable 5.1 和 Claude Mythos 5.1：模型 ID、破坏性变更和迁移检查清单。
---

<Note>
  本指南涵盖 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 代码的迁移。如果您使用 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)，则除了更新模型名称之外无需进行任何更改。
</Note>

<Tip>
  **使用 Claude API skill 自动完成迁移。** 在 Claude Code 中，运行 `/claude-api migrate` 以调用内置的 [Claude API skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model)。它适用于以任何当前 Claude 模型作为目标：

  ```text wrap
  /claude-api migrate this project to claude-fable-5-1
  ```

  该 skill 会在您的整个代码库中应用模型 ID 替换，并根据需要处理破坏性参数变更、prefill（预填充）替换以及针对目标模型的 effort（努力程度）校准，然后生成一份需要手动验证的事项清单。在编辑任何文件之前，它会要求您确认迁移范围（整个工作目录、某个子目录或特定的文件列表）。该 skill 还会检测 Amazon Bedrock 和 Claude Platform on AWS 客户端，并针对这些平台调整模型 ID 格式和功能变更。
</Tip>

[Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1) 接替 Claude Fable 5，输入和输出价格相同，缓存读取成本仅为四分之一。它可在 Claude API、[Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)、[Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 和 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上使用。[Claude Mythos 5.1](https://anthropic.com/glasswing) 具有相同的能力，仅向 Project Glasswing 中获得批准的客户提供。有关行为差异和提示模式，请参阅[为 Claude Fable 5.1 编写提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1)。

`claude-fable-5-1` 和 `claude-mythos-5-1` 共享的基线设置：

* **思考：**[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)始终开启，与 Claude Fable 5 相同。模型自行决定何时思考以及思考多少。无需 `thinking` 配置。`thinking: {type: "disabled"}` 和手动 "extended thinking"（扩展思考）（`thinking: {type: "enabled", budget_tokens: N}`）都会返回 400 错误。
* \*\*预填充：\*\*预填充助手消息会返回 400 错误，与 Claude Fable 5 相同。请改用 "system prompt"（系统提示）指令。
* \*\*工具选择：\*\*支持 `{type: "auto"}`（默认）和 `{type: "none"}`。使用 `{type: "any"}` 或 `{type: "tool", name: "..."}` 强制工具调用会返回 400 错误。请参阅[破坏性变更](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-breaking-changes)。
* \*\*跨模型保留的思考：\*\*Claude Fable 5.1 可读取来自 Claude Opus 5、Claude Fable 5、Claude Mythos 5 及更早 Claude 模型的思考块。这些模型都无法读取 Claude Fable 5.1 的思考块。请参阅[破坏性变更](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-breaking-changes)。
* \*\*上下文窗口和输出：\*\*默认提供 [1M 令牌的 "context window"（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)，每个请求最多 128k 输出令牌。
* \*\*定价：\*\*每百万输入令牌 10 美元，每百万输出令牌 50 美元，与 Claude Fable 5 相同。提示缓存读取为每百万令牌 0.25 美元，是 Claude Fable 5 费率的四分之一。请参阅 [Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。
* \*\*数据保留：\*\*两个模型都要求 30 天数据保留，除非获得 Anthropic 明确授权，否则不适用于零数据保留（ZDR）安排，并且被指定为受管辖模型（Covered Models），与 Claude Fable 5 和 Claude Mythos 5 相同。在 Claude API 上，来自未启用 30 天保留的组织或工作区的请求会返回 400 `invalid_request_error`。拥有 ZDR 安排的组织应联系其 Anthropic 客户团队，或按工作区配置保留策略。有关各平台的详细信息，请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。

两个模型的不同之处：

* \*\*可用性：\*\*Claude Fable 5.1 不需要访问审批。Claude Mythos 5.1 仅向 [Project Glasswing](https://anthropic.com/glasswing) 中获得批准的客户提供。请联系您的 Anthropic 客户团队以获取访问权限。
* \*\*安全分类器：\*\*Claude Fable 5.1 运行的安全分类器覆盖与 Claude Fable 5 相同的 `stop_details` 类别。被拒绝的请求会返回 `stop_reason: "refusal"` 以及 `stop_details.category`，并且可以通过 `fallbacks` 参数或客户端重试回退到另一个模型。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。
* \*\*优先级层级：\*\*两个模型都不支持[优先级层级（Priority Tier）](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models)。Claude Fable 5 支持。

## 从 Claude Fable 5 迁移到 Claude Fable 5.1

迁移基本上是直接替换。API 接口、限制、每令牌定价、分词器、始终开启的自适应思考、拒绝处理以及 `stop_details` 类别都与 Claude Fable 5 一致。变化之处在于：强制工具选择会返回 400 错误；思考块仅为生成它们的模型或更新的模型保留，并且仅在生成它们的对话中保留；缓存读取成本更低；智能体循环行为在三个方面有所不同。相同的变更也适用于 [Claude Mythos 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#migrating-from-claude-mythos-5-to-claude-mythos-5-1)，但思考块的对话检查除外，Claude Mythos 5.1 不运行该检查。

### 更新您的模型名称

```python
model = "claude-fable-5"  # Before
model = "claude-fable-5-1"  # After

# 或者，对于具有相同功能的 Project Glasswing 模型：
model = "claude-mythos-5-1"  # After
```

### 破坏性变更

1. \*\*不支持强制工具选择：\*\*Claude Fable 5 接受 `tool_choice` 的 `auto`、`none`、`any` 和 `tool`。在 `claude-fable-5-1` 上，`{type: "any"}` 和 `{type: "tool", name: "..."}` 会返回 400 `invalid_request_error`：

   ```text wrap
   tool_choice: type "tool" and "any" are not supported for this model.
   ```

   该检查适用于 Messages API、Message Batches API 和[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点。

   之前（Claude Fable 5）：

   <CodeGroup>
     ```bash cURL
     curl -sS https://api.anthropic.com/v1/messages \
       -H "content-type: application/json" \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -d @- <<'EOF'
     {
       "model": "claude-fable-5",
       "max_tokens": 16000,
       "tools": [
         {
           "name": "record_summary",
           "description": "Record the structured summary of the document.",
           "input_schema": {
             "type": "object",
             "properties": {"summary": {"type": "string"}},
             "required": ["summary"]
           }
         }
       ],
       "tool_choice": {"type": "tool", "name": "record_summary"},
       "messages": [
         {"role": "user", "content": "Summarize: The meeting moved to Thursday."}
       ]
     }
     EOF
     ```

     <MultiFileExample language="cli" label="CLI">
       ```bash CLI
       ant messages create < request.yaml
       ```

       <File filename="request.yaml">
         ```yaml
         model: claude-fable-5
         max_tokens: 16000
         tools:
           - name: record_summary
             description: Record the structured summary of the document.
             input_schema:
               type: object
               properties:
                 summary:
                   type: string
               required: [summary]
         tool_choice:
           type: tool
           name: record_summary
         messages:
           - role: user
             content: "Summarize: The meeting moved to Thursday."
         ```
       </File>
     </MultiFileExample>

     ```python Python
     client = anthropic.Anthropic()

     record_summary_tool = {
         "name": "record_summary",
         "description": "Record the structured summary of the document.",
         "input_schema": {
             "type": "object",
             "properties": {"summary": {"type": "string"}},
             "required": ["summary"],
         },
     }

     response = client.messages.create(
         model="claude-fable-5",
         max_tokens=16000,
         tools=[record_summary_tool],
         tool_choice={"type": "tool", "name": "record_summary"},
         messages=[{"role": "user", "content": "Summarize: The meeting moved to Thursday."}],
     )
     print(response.content)
     ```

     ```typescript TypeScript
     const client = new Anthropic();

     const response = await client.messages.create({
       model: "claude-fable-5",
       max_tokens: 16000,
       tools: [
         {
           name: "record_summary",
           description: "Record the structured summary of the document.",
           input_schema: {
             type: "object",
             properties: { summary: { type: "string" } },
             required: ["summary"]
           }
         }
       ],
       tool_choice: { type: "tool", name: "record_summary" },
       messages: [{ role: "user", content: "Summarize: The meeting moved to Thursday." }]
     });

     console.log(response.content);
     ```

     ```csharp C#
     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = Model.ClaudeFable5,
         MaxTokens = 16000,
         Tools = [
             new ToolUnion(new Tool()
             {
                 Name = "record_summary",
                 Description = "Record the structured summary of the document.",
                 InputSchema = new InputSchema()
                 {
                     Properties = new Dictionary<string, JsonElement>
                     {
                         ["summary"] = JsonSerializer.SerializeToElement(new { type = "string" }),
                     },
                     Required = ["summary"],
                 },
             }),
         ],
         ToolChoice = new ToolChoiceTool { Name = "record_summary" },
         Messages = [
             new() { Role = Role.User, Content = "Summarize: The meeting moved to Thursday." }
         ]
     };

     var message = await client.Messages.Create(parameters);
     Console.WriteLine(message);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     anthropic.ModelClaudeFable5,
     	MaxTokens: 16000,
     	Tools: []anthropic.ToolUnionParam{
     		{OfTool: &anthropic.ToolParam{
     			Name:        "record_summary",
     			Description: anthropic.String("Record the structured summary of the document."),
     			InputSchema: anthropic.ToolInputSchemaParam{
     				Properties: map[string]any{
     					"summary": map[string]any{"type": "string"},
     				},
     				Required: []string{"summary"},
     			},
     		}},
     	},
     	ToolChoice: anthropic.ToolChoiceUnionParam{OfTool: &anthropic.ToolChoiceToolParam{Name: "record_summary"}},
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("Summarize: The meeting moved to Thursday.")),
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response.RawJSON())
     ```

     ```java Java

     void main() {
         AnthropicClient client = AnthropicOkHttpClient.fromEnv();

         MessageCreateParams params = MessageCreateParams.builder()
             .model(Model.CLAUDE_FABLE_5)
             .maxTokens(16000L)
             .addTool(Tool.builder()
                 .name("record_summary")
                 .description("Record the structured summary of the document.")
                 .inputSchema(InputSchema.builder()
                     .properties(JsonValue.from(Map.of("summary", Map.of("type", "string"))))
                     .required(List.of("summary"))
                     .build())
                 .build())
             .toolChoice(ToolChoice.ofTool(ToolChoiceTool.builder()
                 .name("record_summary")
                 .build()))
             .addUserMessage("Summarize: The meeting moved to Thursday.")
             .build();

         Message response = client.messages().create(params);
         IO.println(response);
     }
     ```

     ```php PHP
     $client = new Client();

     $message = $client->messages->create(
         maxTokens: 16000,
         messages: [
             ['role' => 'user', 'content' => 'Summarize: The meeting moved to Thursday.']
         ],
         model: 'claude-fable-5',
         toolChoice: ['type' => 'tool', 'name' => 'record_summary'],
         tools: [
             [
                 'name' => 'record_summary',
                 'description' => 'Record the structured summary of the document.',
                 'input_schema' => [
                     'type' => 'object',
                     'properties' => [
                         'summary' => ['type' => 'string']
                     ],
                     'required' => ['summary']
                 ]
             ]
         ],
     );

     echo $message;
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     message = client.messages.create(
       model: Anthropic::Model::CLAUDE_FABLE_5,
       max_tokens: 16000,
       tools: [
         {
           name: "record_summary",
           description: "Record the structured summary of the document.",
           input_schema: {
             type: "object",
             properties: { summary: { type: "string" } },
             required: ["summary"]
           }
         }
       ],
       tool_choice: { type: "tool", name: "record_summary" },
       messages: [
         { role: "user", content: "Summarize: The meeting moved to Thursday." }
       ]
     )
     puts message
     ```
   </CodeGroup>

   之后（Claude Fable 5.1）：将 `tool_choice` 保持为 `auto`，在指令中指明工具名称，并设置 `strict: true` 以使调用符合您的模式。（在 [CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek) 组织中，[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)（包括 `strict: true`）在 Claude Fable 模型上不可用，此时仅依赖指令。）例如：

   <CodeGroup>
     ```bash cURL
     curl -sS https://api.anthropic.com/v1/messages \
       -H "content-type: application/json" \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -d @- <<'EOF'
     {
       "model": "claude-fable-5-1",
       "max_tokens": 16000,
       "tools": [
         {
           "name": "record_summary",
           "description": "Record the structured summary of the document.",
           "strict": true,
           "input_schema": {
             "type": "object",
             "properties": {"summary": {"type": "string"}},
             "required": ["summary"],
             "additionalProperties": false
           }
         }
       ],
       "tool_choice": {"type": "auto"},
       "messages": [
         {"role": "user", "content": "Summarize: The meeting moved to Thursday. Call the record_summary tool with your result."}
       ]
     }
     EOF
     ```

     <MultiFileExample language="cli" label="CLI">
       ```bash CLI
       ant messages create < request.yaml
       ```

       <File filename="request.yaml">
         ```yaml
         model: claude-fable-5-1
         max_tokens: 16000
         tools:
           - name: record_summary
             description: Record the structured summary of the document.
             strict: true
             input_schema:
               type: object
               properties:
                 summary:
                   type: string
               required: [summary]
               additionalProperties: false
         tool_choice:
           type: auto
         messages:
           - role: user
             content: "Summarize: The meeting moved to Thursday. Call the record_summary tool with your result."
         ```
       </File>
     </MultiFileExample>

     ```python Python
     client = anthropic.Anthropic()

     record_summary_tool = {
         "name": "record_summary",
         "description": "Record the structured summary of the document.",
         "strict": True,
         "input_schema": {
             "type": "object",
             "properties": {"summary": {"type": "string"}},
             "required": ["summary"],
             "additionalProperties": False,
         },
     }

     response = client.messages.create(
         model="claude-fable-5-1",
         max_tokens=16000,
         tools=[record_summary_tool],
         tool_choice={"type": "auto"},
         messages=[
             {
                 "role": "user",
                 "content": "Summarize: The meeting moved to Thursday. Call the record_summary tool with your result.",
             }
         ],
     )
     print(response.content)
     ```

     ```typescript TypeScript
     const client = new Anthropic();

     const response = await client.messages.create({
       model: "claude-fable-5-1",
       max_tokens: 16000,
       tools: [
         {
           name: "record_summary",
           description: "Record the structured summary of the document.",
           strict: true,
           input_schema: {
             type: "object",
             properties: { summary: { type: "string" } },
             required: ["summary"],
             additionalProperties: false
           }
         }
       ],
       tool_choice: { type: "auto" },
       messages: [
         {
           role: "user",
           content:
             "Summarize: The meeting moved to Thursday. Call the record_summary tool with your result."
         }
       ]
     });

     console.log(response.content);
     ```

     ```csharp C#
     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = "claude-fable-5-1",
         MaxTokens = 16000,
         Tools = [
             new ToolUnion(new Tool()
             {
                 Name = "record_summary",
                 Description = "Record the structured summary of the document.",
                 Strict = true,
                 InputSchema = new InputSchema(new Dictionary<string, JsonElement>
                 {
                     ["properties"] = JsonSerializer.SerializeToElement(new Dictionary<string, object>
                     {
                         ["summary"] = new { type = "string" },
                     }),
                     ["required"] = JsonSerializer.SerializeToElement(new[] { "summary" }),
                     ["additionalProperties"] = JsonSerializer.SerializeToElement(false),
                 }),
             }),
         ],
         ToolChoice = new ToolChoiceAuto(),
         Messages = [
             new() { Role = Role.User, Content = "Summarize: The meeting moved to Thursday. Call the record_summary tool with your result." }
         ]
     };

     var message = await client.Messages.Create(parameters);
     Console.WriteLine(message);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     "claude-fable-5-1",
     	MaxTokens: 16000,
     	Tools: []anthropic.ToolUnionParam{
     		{OfTool: &anthropic.ToolParam{
     			Name:        "record_summary",
     			Description: anthropic.String("Record the structured summary of the document."),
     			Strict:      anthropic.Bool(true),
     			InputSchema: anthropic.ToolInputSchemaParam{
     				Properties: map[string]any{
     					"summary": map[string]any{"type": "string"},
     				},
     				Required: []string{"summary"},
     				ExtraFields: map[string]any{
     					"additionalProperties": false,
     				},
     			},
     		}},
     	},
     	ToolChoice: anthropic.ToolChoiceUnionParam{OfAuto: &anthropic.ToolChoiceAutoParam{}},
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("Summarize: The meeting moved to Thursday. Call the record_summary tool with your result.")),
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response.RawJSON())
     ```

     ```java Java

     void main() {
         AnthropicClient client = AnthropicOkHttpClient.fromEnv();

         MessageCreateParams params = MessageCreateParams.builder()
             .model("claude-fable-5-1")
             .maxTokens(16000L)
             .addTool(Tool.builder()
                 .name("record_summary")
                 .description("Record the structured summary of the document.")
                 .inputSchema(InputSchema.builder()
                     .properties(JsonValue.from(Map.of("summary", Map.of("type", "string"))))
                     .putAdditionalProperty("required", JsonValue.from(List.of("summary")))
                     .putAdditionalProperty("additionalProperties", JsonValue.from(false))
                     .build())
                 .strict(true)
                 .build())
             .toolChoice(ToolChoice.ofAuto(ToolChoiceAuto.builder().build()))
             .addUserMessage("Summarize: The meeting moved to Thursday. Call the record_summary tool with your result.")
             .build();

         Message response = client.messages().create(params);
         IO.println(response);
     }
     ```

     ```php PHP
     $client = new Client();

     $message = $client->messages->create(
         maxTokens: 16000,
         messages: [
             ['role' => 'user', 'content' => 'Summarize: The meeting moved to Thursday. Call the record_summary tool with your result.']
         ],
         model: 'claude-fable-5-1',
         toolChoice: ['type' => 'auto'],
         tools: [
             [
                 'name' => 'record_summary',
                 'description' => 'Record the structured summary of the document.',
                 'strict' => true,
                 'input_schema' => [
                     'type' => 'object',
                     'properties' => [
                         'summary' => ['type' => 'string']
                     ],
                     'required' => ['summary'],
                     'additionalProperties' => false
                 ]
             ]
         ],
     );

     echo $message;
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     message = client.messages.create(
       model: "claude-fable-5-1",
       max_tokens: 16000,
       tools: [
         {
           name: "record_summary",
           description: "Record the structured summary of the document.",
           strict: true,
           input_schema: {
             type: "object",
             properties: { summary: { type: "string" } },
             required: ["summary"],
             additionalProperties: false
           }
         }
       ],
       tool_choice: { type: "auto" },
       messages: [
         { role: "user", content: "Summarize: The meeting moved to Thursday. Call the record_summary tool with your result." }
       ]
     )
     puts message
     ```
   </CodeGroup>

   请参阅[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)和[强制工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/define-tools#forcing-tool-use)。如果您强制使用工具只是为了获得符合模式的 JSON，请改用 [JSON 输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs#json-outputs)（`output_config.format`）。

   如果是您的应用程序（而非用户）要求在多轮对话的当前轮次中进行特定的工具调用，请在最新的 `user` 轮次之后追加一条[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)。指明工具名称，说明本轮次必须进行该调用，并告诉 Claude 以该调用开始其响应。由于该消息是追加的，而不是写入顶层 `system` 提示中，因此较早的轮次保持字节级一致，并保留其[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)命中：

   <CodeGroup>
     ```bash cURL
     curl -sS https://api.anthropic.com/v1/messages \
       -H "content-type: application/json" \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -d @- <<'EOF'
     {
       "model": "claude-fable-5-1",
       "max_tokens": 16000,
       "system": "You are a customer support assistant for an online electronics store.",
       "tools": [
         {
           "name": "search_help_center",
           "description": "Search the help center for policy and troubleshooting articles.",
           "strict": true,
           "input_schema": {
             "type": "object",
             "properties": {"query": {"type": "string"}},
             "required": ["query"],
             "additionalProperties": false
           }
         }
       ],
       "messages": [
         {"role": "user", "content": "My headphones from order A1234 arrived yesterday."},
         {"role": "assistant", "content": "Thanks for confirming. How can I help with order A1234?"},
         {"role": "user", "content": "I opened the box. Can I still return them?"},
         {
           "role": "system",
           "content": "Tool-use requirement for the current turn: the application requires a call to the search_help_center tool in your response to the user's latest message. Begin your response with the search_help_center tool call. Do not reply with text only."
         }
       ]
     }
     EOF
     ```

     <MultiFileExample language="cli" label="CLI">
       ```bash CLI
       ant messages create < request.yaml
       ```

       <File filename="request.yaml">
         ```yaml
         model: claude-fable-5-1
         max_tokens: 16000
         system: You are a customer support assistant for an online electronics store.
         tools:
           - name: search_help_center
             description: Search the help center for policy and troubleshooting articles.
             strict: true
             input_schema:
               type: object
               properties:
                 query:
                   type: string
               required: [query]
               additionalProperties: false
         messages:
           - role: user
             content: My headphones from order A1234 arrived yesterday.
           - role: assistant
             content: Thanks for confirming. How can I help with order A1234?
           - role: user
             content: I opened the box. Can I still return them?
           - role: system
             content: >-
               Tool-use requirement for the current turn: the application requires a call
               to the search_help_center tool in your response to the user's latest message.
               Begin your response with the search_help_center tool call. Do not reply with
               text only.
         ```
       </File>
     </MultiFileExample>

     ```python Python
     client = anthropic.Anthropic()

     search_help_center_tool = {
         "name": "search_help_center",
         "description": "Search the help center for policy and troubleshooting articles.",
         "strict": True,
         "input_schema": {
             "type": "object",
             "properties": {"query": {"type": "string"}},
             "required": ["query"],
             "additionalProperties": False,
         },
     }

     response = client.messages.create(
         model="claude-fable-5-1",
         max_tokens=16000,
         system="You are a customer support assistant for an online electronics store.",
         tools=[search_help_center_tool],
         messages=[
             {
                 "role": "user",
                 "content": "My headphones from order A1234 arrived yesterday.",
             },
             {
                 "role": "assistant",
                 "content": "Thanks for confirming. How can I help with order A1234?",
             },
             {"role": "user", "content": "I opened the box. Can I still return them?"},
             # 应用程序要求在给出任何政策相关回答之前
             # 先查询帮助中心。将该要求作为系统消息追加，
             # 可使之前的对话轮次保持不变。
             {
                 "role": "system",
                 "content": "Tool-use requirement for the current turn: the application requires a call to the search_help_center tool in your response to the user's latest message. Begin your response with the search_help_center tool call. Do not reply with text only.",
             },
         ],
     )
     print(response.content)
     ```

     ```typescript TypeScript
     const client = new Anthropic();

     const response = await client.messages.create({
       model: "claude-fable-5-1",
       max_tokens: 16000,
       system: "You are a customer support assistant for an online electronics store.",
       tools: [
         {
           name: "search_help_center",
           description: "Search the help center for policy and troubleshooting articles.",
           strict: true,
           input_schema: {
             type: "object",
             properties: { query: { type: "string" } },
             required: ["query"],
             additionalProperties: false
           }
         }
       ],
       messages: [
         { role: "user", content: "My headphones from order A1234 arrived yesterday." },
         { role: "assistant", content: "Thanks for confirming. How can I help with order A1234?" },
         { role: "user", content: "I opened the box. Can I still return them?" },
         // 应用程序要求在给出任何政策相关回答之前
         // 先查询帮助中心。将该要求作为系统消息追加，
         // 可使先前的对话轮次保持不变。
         {
           role: "system",
           content:
             "Tool-use requirement for the current turn: the application requires a call to the search_help_center tool in your response to the user's latest message. Begin your response with the search_help_center tool call. Do not reply with text only."
         }
       ]
     });

     console.log(response.content);
     ```

     ```csharp C#
     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = "claude-fable-5-1",
         MaxTokens = 16000,
         System = "You are a customer support assistant for an online electronics store.",
         Tools = [
             new ToolUnion(new Tool()
             {
                 Name = "search_help_center",
                 Description = "Search the help center for policy and troubleshooting articles.",
                 Strict = true,
                 InputSchema = new InputSchema(new Dictionary<string, JsonElement>
                 {
                     ["properties"] = JsonSerializer.SerializeToElement(new Dictionary<string, object>
                     {
                         ["query"] = new { type = "string" },
                     }),
                     ["required"] = JsonSerializer.SerializeToElement(new[] { "query" }),
                     ["additionalProperties"] = JsonSerializer.SerializeToElement(false),
                 }),
             }),
         ],
         Messages = [
             new() { Role = Role.User, Content = "My headphones from order A1234 arrived yesterday." },
             new() { Role = Role.Assistant, Content = "Thanks for confirming. How can I help with order A1234?" },
             new() { Role = Role.User, Content = "I opened the box. Can I still return them?" },
             // 应用程序要求在给出任何政策相关回答之前，
             // 先查询帮助中心。将该要求作为系统消息追加，
             // 可使之前的对话轮次保持不变。
             new()
             {
                 Role = Role.System,
                 Content = "Tool-use requirement for the current turn: the application requires a call to the search_help_center tool in your response to the user's latest message. Begin your response with the search_help_center tool call. Do not reply with text only."
             }
         ]
     };

     var message = await client.Messages.Create(parameters);
     Console.WriteLine(message);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     "claude-fable-5-1",
     	MaxTokens: 16000,
     	System: []anthropic.TextBlockParam{
     		{Text: "You are a customer support assistant for an online electronics store."},
     	},
     	Tools: []anthropic.ToolUnionParam{
     		{OfTool: &anthropic.ToolParam{
     			Name:        "search_help_center",
     			Description: anthropic.String("Search the help center for policy and troubleshooting articles."),
     			Strict:      anthropic.Bool(true),
     			InputSchema: anthropic.ToolInputSchemaParam{
     				Properties: map[string]any{
     					"query": map[string]any{"type": "string"},
     				},
     				Required: []string{"query"},
     				ExtraFields: map[string]any{
     					"additionalProperties": false,
     				},
     			},
     		}},
     	},
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("My headphones from order A1234 arrived yesterday.")),
     		anthropic.NewAssistantMessage(anthropic.NewTextBlock("Thanks for confirming. How can I help with order A1234?")),
     		anthropic.NewUserMessage(anthropic.NewTextBlock("I opened the box. Can I still return them?")),
     		// 应用程序要求在给出任何政策相关回答之前
     		// 先查询帮助中心。将该要求作为系统消息追加，
     		// 可使之前的对话轮次保持不变。
     		{
     			Role: anthropic.MessageParamRoleSystem,
     			Content: []anthropic.ContentBlockParamUnion{
     				anthropic.NewTextBlock("Tool-use requirement for the current turn: the application requires a call to the search_help_center tool in your response to the user's latest message. Begin your response with the search_help_center tool call. Do not reply with text only."),
     			},
     		},
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response.RawJSON())
     ```

     ```java Java

     void main() {
         AnthropicClient client = AnthropicOkHttpClient.fromEnv();

         MessageCreateParams params = MessageCreateParams.builder()
             .model("claude-fable-5-1")
             .maxTokens(16000L)
             .system("You are a customer support assistant for an online electronics store.")
             .addTool(Tool.builder()
                 .name("search_help_center")
                 .description("Search the help center for policy and troubleshooting articles.")
                 .inputSchema(InputSchema.builder()
                     .properties(JsonValue.from(Map.of("query", Map.of("type", "string"))))
                     .putAdditionalProperty("required", JsonValue.from(List.of("query")))
                     .putAdditionalProperty("additionalProperties", JsonValue.from(false))
                     .build())
                 .strict(true)
                 .build())
             .addUserMessage("My headphones from order A1234 arrived yesterday.")
             .addAssistantMessage("Thanks for confirming. How can I help with order A1234?")
             .addUserMessage("I opened the box. Can I still return them?")
             // 应用程序要求在给出任何政策相关回答之前
             // 先查询帮助中心。将该要求作为系统消息追加，
             // 可使之前的对话轮次保持不变。
             .addMessage(MessageParam.builder()
                 .role(MessageParam.Role.SYSTEM)
                 .content("Tool-use requirement for the current turn: the application requires a call to the search_help_center tool in your response to the user's latest message. Begin your response with the search_help_center tool call. Do not reply with text only.")
                 .build())
             .build();

         Message response = client.messages().create(params);
         IO.println(response);
     }
     ```

     ```php PHP
     $client = new Client();

     $message = $client->messages->create(
         maxTokens: 16000,
         messages: [
             ['role' => 'user', 'content' => 'My headphones from order A1234 arrived yesterday.'],
             ['role' => 'assistant', 'content' => 'Thanks for confirming. How can I help with order A1234?'],
             ['role' => 'user', 'content' => 'I opened the box. Can I still return them?'],
             // 应用程序要求在给出任何政策相关回答之前
             // 先查询帮助中心。将该要求作为系统消息追加，
             // 可使之前的对话轮次保持不变。
             ['role' => 'system', 'content' => 'Tool-use requirement for the current turn: the application requires a call to the search_help_center tool in your response to the user\'s latest message. Begin your response with the search_help_center tool call. Do not reply with text only.']
         ],
         model: 'claude-fable-5-1',
         system: 'You are a customer support assistant for an online electronics store.',
         tools: [
             [
                 'name' => 'search_help_center',
                 'description' => 'Search the help center for policy and troubleshooting articles.',
                 'strict' => true,
                 'input_schema' => [
                     'type' => 'object',
                     'properties' => [
                         'query' => ['type' => 'string']
                     ],
                     'required' => ['query'],
                     'additionalProperties' => false
                 ]
             ]
         ],
     );

     echo $message;
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     message = client.messages.create(
       model: "claude-fable-5-1",
       max_tokens: 16000,
       system: "You are a customer support assistant for an online electronics store.",
       tools: [
         {
           name: "search_help_center",
           description: "Search the help center for policy and troubleshooting articles.",
           strict: true,
           input_schema: {
             type: "object",
             properties: { query: { type: "string" } },
             required: ["query"],
             additionalProperties: false
           }
         }
       ],
       messages: [
         { role: "user", content: "My headphones from order A1234 arrived yesterday." },
         { role: "assistant", content: "Thanks for confirming. How can I help with order A1234?" },
         { role: "user", content: "I opened the box. Can I still return them?" },
         # 应用程序要求在给出任何政策相关回答之前
         # 先查询帮助中心。将该要求作为系统消息追加，
         # 可使之前的对话轮次保持不变。
         {
           role: "system",
           content: "Tool-use requirement for the current turn: the application requires a call to the search_help_center tool in your response to the user's latest message. Begin your response with the search_help_center tool call. Do not reply with text only."
         }
       ]
     )
     puts message
     ```
   </CodeGroup>

   在后续请求中，将 `role: "system"` 消息保留在历史记录中，与任何其他轮次一样。对话中途系统消息不需要 beta 标头。对于不得调用工具的轮次，`tool_choice: {"type": "none"}` 仍然有效。

2. \*\*思考块仅为生成它们的模型或更新的模型保留：\*\*每个 `thinking` 块都会记录是哪个模型生成了它。Claude Fable 5.1 可读取自己的思考块，以及来自 Claude Mythos 5.1、Claude Opus 5、Claude Fable 5、Claude Mythos 5 和更早 Claude 模型的思考块。从这些模型中的任何一个迁移到 `claude-fable-5-1` 的对话会保留其先前的推理。该条件是单向的：除 Claude Mythos 5.1 外，这些模型都无法读取 Claude Fable 5.1 的思考块。

   在 Claude Fable 5.1 上运行的对话可能会通过路由器切换、客户端重试或[分类器拒绝回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)（包括[服务器端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)）落到较旧的模型上。API 会在该模型看到之前移除其无法读取的思考块，请求会成功，并且您不会为被丢弃的输入令牌付费。目标模型会在没有该推理的情况下重新规划，这可能会增加切换后第一轮的成本和延迟。要查看丢弃了什么，请发送 `thinking-binding-controls-2026-08-01` [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)：响应随后会携带一个 `input_transformations` 数组，以 `reason: "model_binding_mismatch"` 标明每个被丢弃的块。请参阅[保留的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-for-model)。

3. \*\*编辑较早的轮次会使思考块失效：\*\*来自 Claude Fable 5.1 的每个 `thinking` 块仅对其之前的 `system` 提示、`tools` 和对话历史有效。如果由 Claude Code、claude.ai、[Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 或 [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview) 管理您的对话历史，它已经保持该前缀完整。如果您的代码自行构建 `messages` 数组，则本条适用于您，[保留的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)是完整的集成指南。在强制执行该检查的情况下，在上述任何内容发生更改后仍将该块发回的请求会被拒绝并返回 400 错误：

   ```text wrap
   messages.5.content.0: Invalid `signature` in `thinking` block. The block is bound to a different conversation. Remove the block, or set `thinking.block_binding.prefix_mismatch_behavior` to "drop_block". That setting requires the `thinking-binding-controls-2026-08-01` value in the `anthropic-beta` header.
   ```

   对于 2026 年 8 月 31 日或之后创建的新账户，API 会强制执行该检查。对于更早创建的账户，API 会记录不匹配但不采取行动，除非请求设置了 `thinking.block_binding.prefix_mismatch_behavior`，这会选择加入强制执行。Anthropic 计划在未来的模型上对每个账户强制执行该检查，因此请现在就让您的应用程序兼容：相同的模式可以保持提示缓存处于热状态，并且您可以通过发送 `prefix_mismatch_behavior` 从任何账户针对该检查进行测试。如果您发布的是人们使用自己的 "API key"（API 密钥）运行的工具或框架，请在发布前以这种方式进行测试：您的密钥很可能属于较旧的账户，而使用新账户的用户会比您先遇到该检查。要查看您自己的账户是否默认强制执行，请在不带 beta 标头的情况下发送一个编辑历史记录的请求：如果返回的 400 错误中提到了该标头，则表示已强制执行。

   该错误对于该请求体是永久性的：自动重试循环无法清除它。要在没有失效推理的情况下继续而不是失败，请从历史记录中剥离 `thinking` 块并重试一次，或者发送 `thinking-binding-controls-2026-08-01` [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)并将 `prefix_mismatch_behavior` 设置为 `"drop_block"`（默认为 `"error"`）。使用 `"drop_block"` 时，API 会丢弃不匹配的块以及对话中其后的每个思考块，并在响应的 `input_transformations` 数组中以 `reason: "prefix_binding_mismatch"` 报告每一个：

   <CodeGroup>
     ```bash cURL
     curl https://api.anthropic.com/v1/messages \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -H "anthropic-beta: thinking-binding-controls-2026-08-01" \
       -H "content-type: application/json" \
       -d '{
         "model": "claude-fable-5-1",
         "max_tokens": 16000,
         "thinking": {
           "type": "adaptive",
           "block_binding": {
             "prefix_mismatch_behavior": "drop_block"
           }
         },
         "messages": [
           {
             "role": "user",
             "content": "What is the greatest common divisor of 1071 and 462?"
           }
         ]
       }'
     ```

     <MultiFileExample language="cli" label="CLI">
       ```bash CLI
       ant beta:messages create \
         --beta thinking-binding-controls-2026-08-01 \
         --transform '{content.#(type=="text")#.text,input_transformations}' \
         --format yaml < request.yaml
       ```

       <File filename="request.yaml">
         ```yaml
         model: claude-fable-5-1
         max_tokens: 16000
         thinking:
           type: adaptive
           block_binding:
             prefix_mismatch_behavior: drop_block
         messages:
           - role: user
             content: What is the greatest common divisor of 1071 and 462?
         ```
       </File>
     </MultiFileExample>

     ```python Python
     client = anthropic.Anthropic()

     response = client.beta.messages.create(
         model="claude-fable-5-1",
         max_tokens=16000,
         thinking={
             "type": "adaptive",
             "block_binding": {"prefix_mismatch_behavior": "drop_block"},
         },
         messages=[
             {
                 "role": "user",
                 "content": "What is the greatest common divisor of 1071 and 462?",
             }
         ],
         betas=["thinking-binding-controls-2026-08-01"],
     )

     for block in response.content:
         if block.type == "text":
             print(block.text)

     print(f"Input transformations: {len(response.input_transformations or [])}")
     ```

     ```typescript TypeScript
     const client = new Anthropic();

     const response = await client.beta.messages.create({
       model: "claude-fable-5-1",
       max_tokens: 16000,
       thinking: {
         type: "adaptive",
         block_binding: { prefix_mismatch_behavior: "drop_block" }
       },
       messages: [
         { role: "user", content: "What is the greatest common divisor of 1071 and 462?" }
       ],
       betas: ["thinking-binding-controls-2026-08-01"]
     });

     for (const block of response.content) {
       if (block.type === "text") {
         console.log(block.text);
       }
     }
     console.log(`Input transformations: ${response.input_transformations?.length ?? 0}`);
     ```

     ```csharp C#
     using Anthropic.Models.Beta;
     using Anthropic.Models.Beta.Messages;

     AnthropicClient client = new();

     var response = await client.Beta.Messages.Create(
         new()
         {
             Model = "claude-fable-5-1",
             MaxTokens = 16000,
             Thinking = new BetaThinkingConfigAdaptive
             {
                 BlockBinding = new()
                 {
                     PrefixMismatchBehavior = BetaThinkingPrefixMismatchBehavior.DropBlock,
                 },
             },
             Messages =
             [
                 new()
                 {
                     Role = Role.User,
                     Content = "What is the greatest common divisor of 1071 and 462?",
                 },
             ],
             Betas = [AnthropicBeta.ThinkingBindingControls2026_08_01],
         }
     );

     foreach (var block in response.Content)
     {
         if (block.TryPickText(out var textBlock))
         {
             Console.WriteLine(textBlock.Text);
         }
     }

     Console.WriteLine($"Input transformations: {response.InputTransformations?.Count ?? 0}");
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
     	Model:     "claude-fable-5-1",
     	MaxTokens: 16000,
     	Thinking: anthropic.BetaThinkingConfigParamUnion{
     		OfAdaptive: &anthropic.BetaThinkingConfigAdaptiveParam{
     			BlockBinding: anthropic.BetaThinkingBlockBindingParam{
     				PrefixMismatchBehavior: anthropic.BetaThinkingPrefixMismatchBehaviorDropBlock,
     			},
     		},
     	},
     	Messages: []anthropic.BetaMessageParam{
     		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("What is the greatest common divisor of 1071 and 462?")),
     	},
     	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaThinkingBindingControls2026_08_01},
     })
     if err != nil {
     	log.Fatal(err)
     }

     for _, block := range response.Content {
     	if textBlock, ok := block.AsAny().(anthropic.BetaTextBlock); ok {
     		fmt.Println(textBlock.Text)
     	}
     }
     fmt.Printf("Input transformations: %d\n", len(response.InputTransformations))
     ```

     ```java Java
     import com.anthropic.models.beta.AnthropicBeta;
     import com.anthropic.models.beta.messages.BetaMessage;
     import com.anthropic.models.beta.messages.BetaThinkingBlockBinding;
     import com.anthropic.models.beta.messages.BetaThinkingConfigAdaptive;
     import com.anthropic.models.beta.messages.BetaThinkingPrefixMismatchBehavior;
     import com.anthropic.models.beta.messages.MessageCreateParams;

     void main() {
         AnthropicClient client = AnthropicOkHttpClient.fromEnv();

         MessageCreateParams params = MessageCreateParams.builder()
             .model("claude-fable-5-1")
             .maxTokens(16000L)
             .addBeta(AnthropicBeta.THINKING_BINDING_CONTROLS_2026_08_01)
             .thinking(BetaThinkingConfigAdaptive.builder()
                 .blockBinding(BetaThinkingBlockBinding.builder()
                     .prefixMismatchBehavior(BetaThinkingPrefixMismatchBehavior.DROP_BLOCK)
                     .build())
                 .build())
             .addUserMessage("What is the greatest common divisor of 1071 and 462?")
             .build();

         BetaMessage response = client.beta().messages().create(params);

         response.content().stream()
             .flatMap(block -> block.text().stream())
             .forEach(textBlock -> IO.println(textBlock.text()));
         IO.println("Input transformations: "
             + response.inputTransformations().map(List::size).orElse(0));
     }
     ```

     ```php PHP
     use Anthropic\Beta\AnthropicBeta;
     use Anthropic\Beta\Messages\BetaThinkingBlockBinding;
     use Anthropic\Beta\Messages\BetaThinkingConfigAdaptive;
     use Anthropic\Beta\Messages\BetaThinkingPrefixMismatchBehavior;
     use Anthropic\Client;

     $client = new Client();

     $response = $client->beta->messages->create(
         model: 'claude-fable-5-1',
         maxTokens: 16000,
         thinking: BetaThinkingConfigAdaptive::with(
             blockBinding: BetaThinkingBlockBinding::with(
                 prefixMismatchBehavior: BetaThinkingPrefixMismatchBehavior::DROP_BLOCK,
             ),
         ),
         messages: [
             ['role' => 'user', 'content' => 'What is the greatest common divisor of 1071 and 462?'],
         ],
         betas: [AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01],
     );

     foreach ($response->content as $block) {
         if ($block->type === 'text') {
             echo $block->text, PHP_EOL;
         }
     }

     echo 'Input transformations: ', count($response->inputTransformations ?? []), PHP_EOL;
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     response = client.beta.messages.create(
       model: "claude-fable-5-1",
       max_tokens: 16_000,
       thinking: {
         type: "adaptive",
         block_binding: {prefix_mismatch_behavior: "drop_block"}
       },
       messages: [
         {role: "user", content: "What is the greatest common divisor of 1071 and 462?"}
       ],
       betas: [Anthropic::AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01]
     )

     response.content.each do |block|
       puts block.text if block.type == :text
     end

     puts "Input transformations: #{response.input_transformations&.length || 0}"
     ```
   </CodeGroup>

   [令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点运行相同的检查。有关响应结构和 "streaming"（流式传输）中的位置，请参阅[针对未保留块的控制（beta）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking-controls)。

   会使后续思考块失效的模式，以及替代做法：

   * 编辑、重新排序或移除较早的轮次。这包括删除旧的工具结果、从对话记录中间剪掉轮次，以及在摘要之后逐字保留最近轮次及其思考块的客户端压缩（包括在几轮之后换入其摘要的后台压缩）。请改用服务器端[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)或[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)（对于旧的工具结果使用[工具结果清除](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#tool-result-clearing)），或[在服务器上裁剪上下文](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-trim-context)中的某种客户端压缩形式。
   * 注入您不持久保存的内容，例如在 `tool_result` 块之后追加并在下一个请求中移除的每轮提醒。请改为将提醒作为[轮次范围的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)发送，并将其保留在历史记录中。
   * 在同一对话的请求之间重建顶层 `system` 提示或 `tools` 数组，例如为了更新当前日期或添加或移除工具。请改为追加一条[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)，携带新指令（"The current date is 2026-09-14."）或 `tool_addition` 和 `tool_removal` 块。
   * 在后续请求中提供不同字节的图像或文档 URL。该检查覆盖的是字节而非 URL 字符串，因此同一文件的轮换签名 URL 没有问题。对于您跨轮次引用的内容，请使用 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传一次并发送 `file_id`，或发送 base64。

   每种替代做法还能使较早的轮次保持字节级一致，并保留编辑历史记录、`system` 提示或 `tools` 数组会丢失的[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)命中。

   仍然有效的模式：

   * 仅追加的历史记录：添加轮次，并将较早的轮次按发送和接收时的原样传回，包括追加的 `role: "system"` 消息。
   * 从较早的助手轮次中移除思考块，从最旧的开始。
   * 更改 `effort`、`max_tokens` 或 `system`、`tools` 和 `messages` 之外的任何其他请求参数，以及添加或移动 `cache_control` 标记。
   * 服务器端压缩和上下文编辑，包括[思考块清除](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#thinking-block-clearing)。它们不算作编辑，因为该检查比较的是您发送时的对话。

   要检查现有集成：

   1. 捕获它在几个正常轮次中发送的确切请求体，如果您的产品有压缩或工具更改，也包括在内。对于每对连续请求，比较 `system` 提示、`tools` 数组和 `messages` 的共享前缀。在新追加的轮次之前，它们应该是字节级一致的。
   2. 使用 `thinking-binding-controls-2026-08-01` beta 标头和 `prefix_mismatch_behavior: "drop_block"` 针对 `claude-fable-5-1` 运行一个正常的多轮会话，并在每个响应上记录 `input_transformations`。每轮都是空数组意味着历史记录完整。带有 `reason: "prefix_binding_mismatch"` 的条目意味着自上一个请求以来，`path` 处的块之前的某些内容发生了更改。带有 `reason: "model_binding_mismatch"` 的条目意味着对话切换了模型，这不是您代码中的错误。这在任何账户上都有效，因为设置该字段会使请求选择加入强制执行。在 CI 中，请改为设置 `"error"`，以便编辑会导致运行失败。
   3. 选择生产设置。如果前缀不匹配只可能意味着您代码中的错误，请保留默认的 `"error"`；或者设置 `"drop_block"` 以丢弃受影响的块而不是失败。无论哪种方式，都要监控 400 错误或 `input_transformations` 条目。

   偶尔丢弃一次思考块（例如在压缩边界处）影响很小。在每个请求上都使先前思考失效的集成每次都会重新启动提示缓存，这可能会增加每个任务的成本（请参阅[保持对话历史仅追加](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#keep-the-conversation-history-append-only)）。

### 行为变更

1. \*\*长智能体循环中的并行工具调用更少：\*\*在长时间运行的循环中，如果下一批独立读取仅由任务隐含（自定义编码智能体、bash 加编辑器框架、计算机使用），Claude Fable 5.1 可能每轮只发出一个工具调用。每个额外的轮次都会消耗令牌、一次往返和实际时间。在每条用户消息之后追加一句批处理指令，作为[轮次范围的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)（`clear_at: "next_user_message"`，beta），或者在没有 beta 的情况下，放在 `tool_result` 块之后的文本块中，并在后续请求中将较早的副本保留在历史记录中。请参阅[在智能体循环中批量处理独立工具调用](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#batch-independent-tool-calls-in-agent-loops)。

2. \*\*工具调用之间的进度消息更少：\*\*与 Claude Fable 5 相比，Claude Fable 5.1 在长工具序列期间写的状态更新更少，其智能体编码摘要也更短。如果您的界面渲染这些更新，请将 `thinking.display` 设置为 `"updates"`（beta）或 `"summarized"`，并明确提示要求它们。请参阅[工具调用之间的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)和[请求面向用户的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#ask-for-user-facing-progress-updates)。

3. \*\*低努力级别下的搜索和检索调用更少：\*\*在 `low` 努力级别下，Claude Fable 5.1 比 Claude Fable 5 更常凭记忆回答，而不是调用搜索或检索工具。如果您的产品依赖低努力级别下的检索，请为这些请求提高努力级别，或告诉模型何时搜索。请参阅[低努力级别下的搜索触发](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#search-triggering-at-low-effort)。

有关散文密度、聊天格式、摘要中的引用和文件编辑方面的差异（这些不影响 API 集成），请参阅[相对于 Claude Fable 5 的变化](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#changed-from-claude-fable-5)。

### 建议的变更

这些变更不是必需的，但每一项都能降低成本或延迟，或消除一种故障模式：

1. \*\*在对话中途更改努力级别（beta）：\*\*在 Claude Fable 5 上，`output_config.effort` 是请求级别的，在请求之间更改它会丢弃较早轮次的缓存前缀。在 `claude-fable-5-1` 上，仅携带 `output_config` 的 `role: "system"` 消息可以为困难步骤提高努力级别或为常规步骤降低努力级别，而不会使[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)失效：

   <CodeGroup>
     ```bash cURL
     # 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
     curl https://api.anthropic.com/v1/messages \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -H "anthropic-beta: mid-conversation-output-config-2026-07-01" \
       -H "content-type: application/json" \
       -d '{
         "model": "claude-fable-5-1",
         "max_tokens": 4096,
         "output_config": {"effort": "high"},
         "messages": [
           {"role": "user", "content": "Plan a migration from SQLite to PostgreSQL in three short steps."},
           {"role": "assistant", "content": "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts."},
           {"role": "system", "content": [], "output_config": {"effort": "low"}},
           {"role": "user", "content": "Summarize the plan in one sentence."}
         ]
       }'
     ```

     <MultiFileExample language="cli" label="CLI">
       ```bash CLI
       ant beta:messages create \
         --beta mid-conversation-output-config-2026-07-01 \
         --transform 'content.#(type=="text").text' \
         --raw-output < request.yaml
       ```

       <File filename="request.yaml">
         ```yaml
         model: claude-fable-5-1
         max_tokens: 4096
         output_config:
           effort: high
         messages:
           - role: user
             content: Plan a migration from SQLite to PostgreSQL in three short steps.
           - role: assistant
             content: "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts."
           # Effort-only system message: the new level takes effect from the next user turn.
           - role: system
             content: []
             output_config:
               effort: low
           - role: user
             content: Summarize the plan in one sentence.
         ```
       </File>
     </MultiFileExample>

     ```python Python
     client = anthropic.Anthropic()

     response = client.beta.messages.create(
         model="claude-fable-5-1",
         max_tokens=4096,
         output_config={"effort": "high"},
         messages=[
             {
                 "role": "user",
                 "content": "Plan a migration from SQLite to PostgreSQL in three short steps.",
             },
             {
                 "role": "assistant",
                 "content": "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts.",
             },
             # 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
             {"role": "system", "content": [], "output_config": {"effort": "low"}},
             {"role": "user", "content": "Summarize the plan in one sentence."},
         ],
         betas=["mid-conversation-output-config-2026-07-01"],
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
       output_config: { effort: "high" },
       messages: [
         {
           role: "user",
           content: "Plan a migration from SQLite to PostgreSQL in three short steps."
         },
         {
           role: "assistant",
           content:
             "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts."
         },
         // 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
         { role: "system", content: [], output_config: { effort: "low" } },
         { role: "user", content: "Summarize the plan in one sentence." }
       ],
       betas: ["mid-conversation-output-config-2026-07-01"]
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
         OutputConfig = new() { Effort = Effort.High },
         Messages =
         [
             new() { Role = Role.User, Content = "Plan a migration from SQLite to PostgreSQL in three short steps." },
             new() { Role = Role.Assistant, Content = "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts." },
             // 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
             new()
             {
                 Role = Role.System,
                 Content = new([]),
                 OutputConfig = new() { Effort = BetaSystemMessageOutputConfigEffort.Low },
             },
             new() { Role = Role.User, Content = "Summarize the plan in one sentence." },
         ],
         Betas = [AnthropicBeta.MidConversationOutputConfig2026_07_01],
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
     	OutputConfig: anthropic.BetaOutputConfigParam{
     		Effort: anthropic.BetaOutputConfigEffortHigh,
     	},
     	Messages: []anthropic.BetaMessageParam{
     		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Plan a migration from SQLite to PostgreSQL in three short steps.")),
     		{
     			Role:    anthropic.BetaMessageParamRoleAssistant,
     			Content: []anthropic.BetaContentBlockParamUnion{anthropic.NewBetaTextBlock("1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts.")},
     		},
     		// 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
     		anthropic.NewBetaSystemMessage(anthropic.BetaSystemMessageOutputConfigParam{
     			Effort: anthropic.BetaSystemMessageOutputConfigEffortLow,
     		}),
     		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Summarize the plan in one sentence.")),
     	},
     	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaMidConversationOutputConfig2026_07_01},
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
     import com.anthropic.models.beta.messages.BetaOutputConfig;
     import com.anthropic.models.beta.messages.BetaSystemMessageOutputConfig;
     import com.anthropic.models.beta.messages.MessageCreateParams;

     void main() {
         AnthropicClient client = AnthropicOkHttpClient.fromEnv();

         MessageCreateParams params = MessageCreateParams.builder()
             .model("claude-fable-5-1")
             .maxTokens(4096L)
             .addBeta(AnthropicBeta.MID_CONVERSATION_OUTPUT_CONFIG_2026_07_01)
             .outputConfig(BetaOutputConfig.builder()
                 .effort(BetaOutputConfig.Effort.HIGH)
                 .build())
             .addUserMessage("Plan a migration from SQLite to PostgreSQL in three short steps.")
             .addAssistantMessage("1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts.")
             // 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
             .addMessage(BetaMessageParam.builder()
                 .role(BetaMessageParam.Role.SYSTEM)
                 .contentOfBetaContentBlockParams(List.of())
                 .outputConfig(BetaSystemMessageOutputConfig.builder()
                     .effort(BetaSystemMessageOutputConfig.Effort.LOW)
                     .build())
                 .build())
             .addUserMessage("Summarize the plan in one sentence.")
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
     use Anthropic\Beta\Messages\BetaOutputConfig;
     use Anthropic\Beta\Messages\BetaSystemMessageOutputConfig;
     use Anthropic\Client;

     $client = new Client();

     $response = $client->beta->messages->create(
         model: 'claude-fable-5-1',
         maxTokens: 4096,
         outputConfig: BetaOutputConfig::with(effort: 'high'),
         messages: [
             BetaMessageParam::with(role: 'user', content: 'Plan a migration from SQLite to PostgreSQL in three short steps.'),
             BetaMessageParam::with(role: 'assistant', content: '1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts.'),
             // 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
             BetaMessageParam::with(
                 role: 'system',
                 content: [],
                 outputConfig: BetaSystemMessageOutputConfig::with(effort: 'low'),
             ),
             BetaMessageParam::with(role: 'user', content: 'Summarize the plan in one sentence.'),
         ],
         betas: [AnthropicBeta::MID_CONVERSATION_OUTPUT_CONFIG_2026_07_01],
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
       output_config: {effort: :high},
       messages: [
         {role: "user", content: "Plan a migration from SQLite to PostgreSQL in three short steps."},
         {role: "assistant", content: "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts."},
         # 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
         {role: "system", content: [], output_config: {effort: :low}},
         {role: "user", content: "Summarize the plan in one sentence."}
       ],
       betas: [Anthropic::AnthropicBeta::MID_CONVERSATION_OUTPUT_CONFIG_2026_07_01]
     )

     response.content.each do |block|
       puts block.text if block.type == :text
     end
     ```
   </CodeGroup>

   该值适用于随后的用户轮次以及之后的每个轮次，直到另一条 `role: "system"` 消息更改它。仅接受命名级别（`low`、`medium`、`high`、`xhigh`、`max`），并且需要 `mid-conversation-output-config-2026-07-01` beta 标头。请参阅[每条消息的努力级别](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)。

2. \*\*使用对话中途系统消息更改指令和工具：\*\*要在会话中途更改指令或工具，请追加一条 [`role: "system"` 消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)，对于工具更改使用 `tool_addition` 和 `tool_removal` 块（beta 标头 `mid-conversation-tool-changes-2026-07-01`，并在会话开始时在 `tools` 中声明完整的工具集）。这会保留较早轮次的提示缓存命中，并使对话历史保持仅追加。当当前轮次必须运行特定工具时，同样的消息可替代强制 `tool_choice`（请参阅[破坏性变更](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-breaking-changes)）。对于仅适用于一个轮次的提醒，请将其作为单独的纯文本 `role: "system"` 消息发送，并带有 `clear_at: "next_user_message"`（[轮次范围的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)，beta 标头 `mid-conversation-system-clear-at-2026-08-21`），并将其保留在历史记录中：它在下一条用户消息之后停止渲染，清除后不消耗令牌。携带 `tool_addition` 或 `tool_removal` 块的消息不能是轮次范围的。

3. \*\*对拒绝使用 `fallbacks: "default"`：\*\*继续处理 `stop_reason: "refusal"` 并在响应内容之前读取 `stop_details.category`。要自动在另一个模型上重新运行被拒绝的请求，请设置 `fallbacks: "default"`（beta，`server-side-fallback-2026-07-01` 标头）。`"default"` 会在 Anthropic 为该类别推荐的模型上重试被拒绝的请求。Claude Fable 5.1 允许的回退目标是 Claude Opus 4.8（`claude-opus-4-8`）和 Claude Opus 5（`claude-opus-5`）。显式的 `fallbacks` 列表可以指定其中任一个。回退模型不会收到 Claude Fable 5.1 的思考块。如果您自行构建重试，[回退额度](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)按与 Claude Fable 5 相同的条款适用。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。

4. **从 `high` 努力级别开始并进行扫描测试：**[努力参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)的默认值为 `high`，并且支持全部五个级别。保持 Claude Fable 5 的指导：大多数工作使用 `high`，`medium` 作为值得测试的成本控制手段。Claude Fable 5.1 相对于 Claude Fable 5 的提升在 `xhigh` 和 `max` 级别最大，但这些级别也会增加思考时间和首次响应时间，因此请在对能力最敏感的任务以及您的评估显示有提升的地方才升级到这些级别。请在您自己的评估上重新进行扫描测试，而不是沿用为 Claude Fable 5 调优的设置。请参阅 [Claude Fable 5.1 的推荐努力级别](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#recommended-effort-levels-for-claude-fable-5-1)。

5. \*\*在服务器上裁剪上下文，或以不携带过时思考的形式进行压缩：\*\*如果您的代码在客户端截断或摘要较旧的轮次，最简单的修复方法是将该工作移至服务器端[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)或[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)。两者都不算作编辑，因为[历史检查](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-preserved-thinking)比较的是您发送时的对话，因此它们移除的任何内容都不会使后续思考块失效，并且压缩的 [`instructions` 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction#custom-summarization-instructions)接受您自己的摘要提示。如果您将压缩保留在客户端，请从三种形式中选择一种：

   * \*\*简单压缩（推荐）：\*\*用一条摘要消息加上新的用户轮次替换整个历史记录，不重放任何其他内容。不会携带任何思考块，因此不会有任何失败。Claude 模型使用此方案在长周期任务上进行训练，对于大多数工作负载，其表现与更复杂的方案相当。
   * \*\*保留尾部压缩：\*\*如果您在摘要之后逐字保留最近的轮次，请从这些轮次中剥离 `thinking` 和 `redacted_thinking` 块（文本和工具调用可以保留），或设置 `prefix_mismatch_behavior: "drop_block"`。否则，它们的思考是针对完整历史记录生成的，在摘要之后会失败。
   * \*\*后台压缩：\*\*如果您在关键路径之外构建摘要并稍后换入，则在此期间生成的每个轮次都携带早于换入的思考。在每个仍携带换入前生成的思考块的请求上发送 `"drop_block"`（或自行剥离这些块；换入后第一个响应上的 `input_transformations` 会准确列出是哪些块），或者同步压缩。

   不要从对话记录中间剪掉单个轮次：这会使之后的每个思考块失效，并且没有任何客户端形式可以避免这一点。对于您要进行的指令更改，请使用[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)；对于选择性移除，请使用服务器端[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)。请参阅[传回压缩块](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction#passing-compaction-blocks-back)。

### 迁移清单

* 将模型名称从 `claude-fable-5` 更新为 `claude-fable-5-1`（或从 `claude-mythos-5` 更新为 `claude-mythos-5-1`）。
* 替换强制 `tool_choice`（`{type: "any"}` 或 `{type: "tool", ...}`）。它会返回 400 错误。请使用 `{type: "auto"}` 加上明确的指令和 `strict: true` 工具，或使用 JSON 输出。将指令放在 `user` 轮次中，或者当您的应用程序要求必须进行该调用时，放在对话中途的 `role: "system"` 消息中。
* 在每一轮中继续原样传回 `thinking` 块，包括空块。Claude Fable 5.1 可以读取来自 Claude Opus 5、Claude Fable 5、Claude Mythos 5 及更早模型的块。将对话从 Claude Fable 5.1 迁移到更早的模型会丢弃其块（Claude Mythos 5.1 可以读取它们）。
* 如果您的代码自行构建 `messages` 数组，请检查它是否[编辑了较早的轮次](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-preserved-thinking)：使用 `thinking-binding-controls-2026-08-01` beta 标头和 `prefix_mismatch_behavior: "drop_block"` 运行一次会话，记录 `input_transformations`，并修复每一个 `prefix_binding_mismatch`。模型切换后出现的 `model_binding_mismatch` 条目属于预期情况。
* 保持对话历史仅追加（append-only）：在会话开始时冻结 `system` 和 `tools`，并将会话中途的更改移至 `role: "system"` 消息以及 `tool_addition` / `tool_removal` 块；将每轮提醒作为您永不移除的轮次范围系统消息发送；在服务器端裁剪上下文，或从您跨客户端摘要携带的任何轮次中剥离 thinking 块；并通过 `file_id` 引用跨轮次文件。
* 选择一个生产环境的 `prefix_mismatch_behavior`（默认为 `"error"`，或 `"drop_block"`）并对其进行监控。如果您维护一个由他人使用其自己的 API 密钥运行的工具，请在设置该字段的情况下进行测试：即使您的账户未被强制执行，新账户默认也会被强制执行。
* 检查智能体循环中是否存在每轮仅调用一个工具的行为，并添加批处理指令。
* 如果您的界面在工具调用之间渲染进度文本，请将 `thinking.display` 设置为 `"updates"`（beta）或 `"summarized"`，并通过提示请求更新。
* 如果您在请求之间更改 effort，请将该更改移至[按消息设置 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta) 的 `role: "system"` 消息（beta）中，以保持缓存命中。
* 处理 `stop_reason: "refusal"` 并读取 `stop_details.category`。考虑使用 `fallbacks: "default"`（beta）。
* 通过一次全新的扫描重新评估 `effort`，从 `high` 开始，并在您自己的工作负载上重新建立成本和延迟基线。令牌数量大致不变。提示缓存读取的费用是 Claude Fable 5 费率的四分之一。

## 从 Claude Opus 5 迁移到 Claude Fable 5.1

Claude Fable 5.1 使用与 Claude Opus 5 相同的 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 和[工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)模式。它默认保留 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)、[128k 最大输出令牌](https://platform.claude.com/docs/zh-CN/models/overview)、512 令牌的提示缓存最小值，以及[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)支持。预填充限制、采样参数限制以及 `thinking.display` 的 `"omitted"` 默认值也同样延续。请应用[从 Claude Fable 5 迁移到 Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#migrating-from-claude-fable-5-to-claude-fable-5-1) 中的所有内容，以及以下内容。

### 更新您的模型名称

```python
model = "claude-opus-5"  # Before
model = "claude-fable-5-1"  # After

# 或者，对于具有相同功能的 Project Glasswing 模型：
model = "claude-mythos-5-1"  # After
```

### 变更内容

1. **思考不能再被禁用：** Claude Opus 5 在 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别为 `high` 或更低时接受 `thinking: {type: "disabled"}`。在 `claude-fable-5-1` 和 `claude-mythos-5-1` 上，[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)始终开启，`thinking: {type: "disabled"}` 在任何 effort 级别下都会返回 400 错误。请移除该字段，使用较低的 effort 级别控制令牌消耗，并针对原先在禁用思考情况下运行的工作负载重新审视 `max_tokens`。

2. **不支持强制工具选择：** Claude Opus 5 接受 `tool_choice` 的 `any` 和 `tool`。`claude-fable-5-1` 会返回 400 错误。请参阅[破坏性变更](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-breaking-changes)。

3. **跨模型保留的思考：** Claude Fable 5.1 可以读取 Claude Opus 5 的 thinking 块：从 `claude-opus-5` 迁移到 `claude-fable-5-1` 的对话会保留其推理。Claude Opus 5 无法读取 Claude Fable 5.1 的块。Claude Fable 5.1 的块还会[在较早轮次发生变化时失效](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-preserved-thinking)：如果您的代码在请求之间编辑较早的消息、重建 `system` 或 `tools`，或在客户端进行压缩，Claude Opus 5 不会提出异议，但 `claude-fable-5-1` 会拒绝或丢弃之后的每一个 thinking 块。在切换流量之前，请运行该部分中的三步检查。请参阅[破坏性变更](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-breaking-changes)。

4. **工具调用之间的文本以 thinking 块返回：** 在 Claude Opus 5 上，模型在工具调用之间写入的文本以 `text` 块返回。在 `claude-fable-5-1` 上，与 Claude Fable 5 一样，这些叙述以进度更新 `thinking` 块的形式返回，每次工具调用之前一个。在 `thinking.display` 默认值 `"omitted"` 下，它们不携带可读文本。如果您的界面渲染这些叙述，请设置 `display: "updates"`（beta）以文本形式接收进度更新而推理保持隐藏，或设置 `"summarized"` 以同时接收两者。然后在 `tool_use` 块之间渲染非空的 `thinking` 块。请参阅[工具调用之间的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)。

5. **安全分类器和回退路由：** Claude Fable 5.1 运行的安全分类器覆盖与 Claude Fable 5 相同的 `stop_details` 类别，比 Claude Opus 5 仅限网络安全的分类器范围更广。预计 `stop_details.category` 的值会超出 `"cyber"`，例如 `"bio"` 和 `"reasoning_extraction"`；完整集合请参阅[拒绝类别表](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response)。有关 `fallbacks` 配置和允许的目标，请参阅[对拒绝使用 `fallbacks: "default"`](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-recommended-changes)。

6. **定价：** 每百万输入令牌 10 美元，每百万输出令牌 50 美元，而 Claude Opus 5 分别为 5 美元和 25 美元。提示缓存读取为每百万令牌 0.25 美元，是 Claude Opus 5 费率的一半。请参阅 [Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

7. **数据保留：** Claude Fable 5.1 和 Claude Mythos 5.1 要求 30 天数据保留，除非获得 Anthropic 明确授权，否则不适用于零数据保留（ZDR）安排，并被指定为受管辖模型（Covered Models）。Claude Opus 5 可在 ZDR 下使用。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。

### 迁移清单

* 如果您的组织有零数据保留（ZDR）安排，请首先确认资格：除非获得 Anthropic 明确授权，否则这些模型不适用于 ZDR。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。
* 将模型名称从 `claude-opus-5` 更新为 `claude-fable-5-1`（或 `claude-mythos-5-1`）。
* 移除任何 `thinking: {type: "disabled"}` 配置：它在 `claude-fable-5-1` 上会返回 400 错误。使用较低的 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别控制令牌消耗，并重新审视 `max_tokens`。
* 将强制 `tool_choice`（`any` 或 `tool`）替换为 `auto` 加上明确的指令（`user` 轮次或对话中途系统消息）和 `strict: true` 工具，或替换为 JSON 输出。
* 如果您的界面渲染工具调用之间的文本，请设置 `display: "updates"`（beta）或 `"summarized"`，并渲染非空的 `thinking` 块。
* 应用 [Claude Fable 5 清单](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#migration-checklist-fable-5-1-from-fable-5)中的保留思考、历史编辑、行为、effort 和回退相关条目。
* 在您自己的工作负载上重新建立成本基线。令牌数量大致不变。每令牌定价有所不同。

## 从 Claude Opus 4.8 或更早版本迁移到 Claude Fable 5.1

首先应用[从 Claude Opus 4.8 迁移到 Claude Mythos 5 和 Claude Fable 5](https://platform.claude.com/docs/zh-CN/models/fable-5/migration-guide#migrating-from-claude-opus-48)，以处理自 Claude Opus 4.8 以来的 API 级别变更。它涵盖自适应思考、思考输出、拒绝、effort、缓存最小值、定价和数据保留。然后应用[从 Claude Fable 5 迁移到 Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#migrating-from-claude-fable-5-to-claude-fable-5-1) 中的剩余差异。如果使用的是 Claude Opus 4.7 或更早版本，请从相应的[迁移到 Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide) 部分开始。

### 更新您的模型名称

```python
model = "claude-opus-4-8"  # Before
model = "claude-fable-5-1"  # After

# 或者，使用具有相同功能的 Project Glasswing 模型：
model = "claude-mythos-5-1"  # After
```

### 迁移清单

* 如果您的组织有零数据保留（ZDR）安排，请首先确认资格：除非获得 Anthropic 明确授权，否则这些模型不适用于 ZDR。Claude Opus 4.8 可在 ZDR 下使用。
* 将模型名称从 `claude-opus-4-8` 更新为 `claude-fable-5-1`（或 `claude-mythos-5-1`）。
* 移除任何 `thinking: {type: "disabled"}` 配置并重新审视 `max_tokens`。不带 `thinking` 字段的请求将以自适应思考运行。
* 将强制 `tool_choice`（`any` 或 `tool`）替换为 `auto` 加上明确的指令（`user` 轮次或对话中途系统消息）和 `strict: true` 工具，或替换为 JSON 输出。
* 原样传回 `thinking` 块，并将其文本视为仅供显示。Claude Fable 5.1 可以读取 Claude Opus 4.8 的 thinking 块：迁移到 `claude-fable-5-1` 的对话会保留其较早的推理。Claude Opus 4.8 无法读取 Claude Fable 5.1 的块。
* 如果您的代码自行构建 `messages` 数组，请检查它是否[编辑了较早的轮次](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-preserved-thinking)。为 Claude Opus 4.8 及更早版本编写的集成通常会截断旧轮次、剥离或重建较早的消息，或在每次请求时刷新 `system` 提示，而 Claude Opus 4.8 从未提出异议。在 `claude-fable-5-1` 上，上述每一种做法都会使之后的 thinking 块失效。
* 处理 `stop_reason: "refusal"`，读取 `stop_details.category`，并考虑使用 `fallbacks: "default"`（beta）。
* 应用 [Claude Fable 5 清单](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#migration-checklist-fable-5-1-from-fable-5)中的保留思考、历史编辑、行为、按消息设置 effort 和进度更新相关条目。
* 重新评估 `effort`（从 `high` 开始），检查接近 512 令牌缓存最小值的提示，并重新建立成本和延迟基线。每令牌定价有所不同。

## 从 Claude Mythos 5 迁移到 Claude Mythos 5.1

[Claude Mythos 5.1](https://anthropic.com/glasswing) 是 Claude Fable 5.1 的访问受限对应版本。在切换模型 ID 之前，请与您的 Anthropic 客户团队确认您组织的访问权限。

API 级别的差异与[从 Claude Fable 5 迁移到 Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#migrating-from-claude-fable-5-to-claude-fable-5-1) 一致：强制工具选择会返回 400 错误，并且 thinking 块仅为生成它们的模型或更新的模型保留（Claude Mythos 5.1 可以读取 Claude Mythos 5 的块，反之则不行）。与 Claude Fable 5.1 不同，Claude Mythos 5.1 不运行对话检查，因此编辑较早的轮次不会[使 thinking 块失效](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-preserved-thinking)，但仍会重启提示缓存。

### 更新您的模型名称

```python
model = "claude-mythos-5"  # Before
model = "claude-mythos-5-1"  # After
```

### 迁移清单

* 将模型名称从 `claude-mythos-5` 更新为 `claude-mythos-5-1`。
* 将强制 `tool_choice`（`any` 或 `tool`）替换为 `auto` 加上明确的指令（`user` 轮次或对话中途系统消息）和 `strict: true` 工具，或替换为 JSON 输出。
* 在处理响应内容之前，处理 `stop_reason: "refusal"` 并读取 `stop_details.category`。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。
* 在每一轮中继续原样传回 `thinking` 块，包括空块。
* 如果您的代码自行构建 `messages` 数组，请保持对话历史仅追加，以保持提示缓存处于热状态。Claude Mythos 5.1 不运行[对话检查](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)，因此编辑不会使其 thinking 块失效。
* 应用 [Claude Fable 5 部分](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#migrating-from-claude-fable-5-to-claude-fable-5-1)中的行为和建议变更，但历史编辑相关条目除外，这些条目不适用于 Claude Mythos 5.1。
* 通过一次全新的扫描重新评估 `effort`，并重新建立成本和延迟基线。提示缓存读取的费用是 Claude Mythos 5 费率的四分之一。
