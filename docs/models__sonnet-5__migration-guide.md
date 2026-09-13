---
title: 迁移到 Claude Sonnet 5
url: https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide
description: 从早期 Claude 模型迁移到 Claude Sonnet 5：模型 ID、破坏性变更和迁移检查清单。
---

<Note>
  本指南涵盖 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 代码的迁移。如果您使用 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)，则除了更新模型名称之外无需进行任何更改。
</Note>

<Tip>
  **使用 Claude API skill 自动完成迁移。** 在 Claude Code 中，运行 `/claude-api migrate` 以调用内置的 [Claude API skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model)。它适用于以任何当前 Claude 模型作为目标：

  ```text wrap
  /claude-api migrate this project to claude-sonnet-5
  ```

  该 skill 会在您的整个代码库中应用模型 ID 替换，并根据需要处理破坏性参数变更、prefill（预填充）替换以及针对目标模型的 effort（努力程度）校准，然后生成一份需要手动验证的事项清单。在编辑任何文件之前，它会要求您确认迁移范围（整个工作目录、某个子目录或特定的文件列表）。该 skill 还会检测 Amazon Bedrock 和 Claude Platform on AWS 客户端，并针对这些平台调整模型 ID 格式和功能变更。
</Tip>

Claude Sonnet 5 在 Claude 模型家族中提供了速度与智能的最佳组合。它建立在 Claude Sonnet 4.6 的基础之上。

Claude Sonnet 5 是 Claude Sonnet 4.6 的直接替换升级，定价为每百万输入/输出令牌 $2/$10 美元；详情请参阅[定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。对于已在 Claude Sonnet 4.6 上运行的代码，有两项破坏性 API 变更。第一，[adaptive thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)（自适应思考）默认开启，而手动 "extended thinking"（扩展思考）（`thinking: {type: "enabled", budget_tokens: N}`）会返回 400 错误，因此原本不带思考运行的请求现在可能会在第一个 `text` 块之前返回 `thinking` 块，按位置读取内容的代码必须改为按 `type` 选择内容块。第二，设置为非默认值的采样参数（`temperature`、`top_p`、`top_k`）会返回 400 错误。请将自适应思考与 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)配合使用来控制思考深度。Claude Sonnet 5 支持与 Claude Sonnet 4.6 相同的功能集，包括 [1M 令牌 "context window"（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)、[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)、["prompt caching"（提示缓存）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)、[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)、[Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)、[PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)、[视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)，以及全套服务器端和客户端[工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。在 Claude API 和 Google Cloud 上，Claude Sonnet 5 还支持作为稳定版 `computer_toolset_20260801` 工具集的 [computer use](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)（计算机使用），以及用于网页内任务的[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)，这两者 Claude Sonnet 4.6 均不支持；基于早期 `computer_20251124` 版本的现有集成在两个模型上均可继续正常工作，无需更改。要升级现有集成，请参阅[从 `computer_20251124` 迁移](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#migrate-from-computer-20251124)。[Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models) 在 Claude Sonnet 5 上不可用。Claude Sonnet 5 还使用了新的分词器（tokenizer）。

## 从 Claude Sonnet 4.6 迁移到 Claude Sonnet 5

<Note>
  如果您的代码使用的是 Claude Sonnet 4.5 或更早版本，还需应用[从 Claude Sonnet 4.5 及更早的 Sonnet 模型迁移到 Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide#migrating-from-sonnet-45)。这些步骤包含本节未单独涵盖的破坏性变更（助手消息预填充被拒绝、工具参数 JSON 转义差异）。
</Note>

### 更新您的模型名称

```python
# Sonnet 迁移
model = "claude-sonnet-4-6"  # Before
model = "claude-sonnet-5"  # After
```

### 变更内容

以下列表中的第 4 项和第 5 项是破坏性变更。`max_tokens` 仍然是总输出（思考加响应文本）的硬性限制，因此对于在 Claude Sonnet 4.6 上不带思考运行的工作负载，请重新审视该值。

1. **新分词器：** Claude Sonnet 5 使用新的分词器。相同的输入文本产生的令牌数比 Claude Sonnet 4.6 多约 30%。确切的增幅取决于内容。请求、响应和 "streaming"（流式传输）事件保持相同的结构，无需更改代码，但您以令牌计量或预算的任何内容都会发生变化：相同文本的 `usage` 字段和[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)结果会更高，1M 令牌上下文窗口可容纳的文本更少，而针对 Claude Sonnet 4.6 调优的 `max_tokens` 限制可能会截断等效输出。每令牌定价更低（每百万输入/输出令牌 $2/$10 美元，而 Claude Sonnet 4.6 为 $3/$15 美元），但等效请求的成本不会按相同比例直接下降。请针对 Claude Sonnet 5 重新运行令牌计数，而不要复用针对早期模型测得的计数。

2. **128k 最大输出令牌（未变更）：** Claude Sonnet 5 支持最多 128k 输出令牌，与 Claude Sonnet 4.6 相同。现有的 `max_tokens` 值仍然有效。在设定其大小时请考虑新分词器的影响。

3. **助手消息预填充（未变更）：** 在 Claude Sonnet 5 上预填充助手消息会返回 `400` 错误，与 Claude Sonnet 4.6 相同。如果您在迁移到 Claude Sonnet 4.6 时已移除预填充，则无需进一步更改。请改用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)、"system prompt"（系统提示）指令或 `output_config.format`。

4. **自适应思考默认开启：** 在 Claude Sonnet 4.6 上，不带 `thinking` 字段的请求不带思考运行；在 Claude Sonnet 5 上，相同的请求会以[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)运行。要关闭思考，请传入 `thinking: {type: "disabled"}`。手动扩展思考（`thinking: {type: "enabled", budget_tokens: N}`）不受支持，会返回 400 错误。请使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)（默认 `high`）来控制思考深度。

   开启思考后，响应可能会在第一个 `text` 块之前以一个或多个 `thinking` 块开头，在默认的 `display: "omitted"` 下，这些块返回时 `thinking` 字段为空。按位置读取回复的代码（例如 `content[0].text`，或将第一个内容块视为文本的流处理程序）必须改为按 `type` 字段选择内容块，并且工具使用循环必须将 `thinking` 块完整且未经修改地与其工具结果一起传回（请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)）。即使未返回思考文本，思考令牌也会按输出令牌计费。如果您在 Claude Sonnet 4.6 上使用了思考并显示返回的思考文本，请注意 `thinking.display` 在那里默认为 `"summarized"`，而在 Claude Sonnet 5 上默认为 `"omitted"`；请像以下示例那样设置 `display: "summarized"`，以继续接收可读的摘要（请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)）。

   <Tabs>
     <Tab title="Claude Sonnet 5">
       <Note>
         Claude Sonnet 5 默认开启自适应思考。此处显式展示 `thinking` 字段是为了设置 `display: "summarized"`；如果您省略 `thinking`，Claude Sonnet 5 默认会在响应中省略思考内容。有关各模型的默认值，请参阅[各模型拒绝的配置](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#rejected-configurations)。
       </Note>

       <CodeGroup>
         ```bash cURL
         curl https://api.anthropic.com/v1/messages \
           -H "x-api-key: $ANTHROPIC_API_KEY" \
           -H "anthropic-version: 2023-06-01" \
           -H "content-type: application/json" \
           -d '{
             "model": "claude-sonnet-5",
             "max_tokens": 16000,
             "thinking": {
               "type": "adaptive",
               "display": "summarized"
             },
             "output_config": {
               "effort": "high"
             },
             "messages": [
               {
                 "role": "user",
                 "content": "Are there an infinite number of prime numbers such that n mod 4 == 3?"
               }
             ]
           }'
         ```

         ```bash CLI
         ant messages create --transform content --format yaml <<'YAML'
         model: claude-sonnet-5
         max_tokens: 16000
         thinking:
           type: adaptive
           display: summarized
         output_config:
           effort: high
         messages:
           - role: user
             content: Are there an infinite number of prime numbers such that n mod 4 == 3?
         YAML
         ```

         ```python Python
         client = anthropic.Anthropic()

         response = client.messages.create(
             model="claude-sonnet-5",
             max_tokens=16000,
             thinking={"type": "adaptive", "display": "summarized"},
             output_config={"effort": "high"},
             messages=[
                 {
                     "role": "user",
                     "content": "Are there an infinite number of prime numbers such that n mod 4 == 3?",
                 }
             ],
         )

         # 响应包含摘要化的思考块和文本块
         for block in response.content:
             match block.type:
                 case "thinking":
                     print(f"\nThinking summary: {block.thinking}")
                 case "text":
                     print(f"\nResponse: {block.text}")
         ```

         ```typescript TypeScript
         const client = new Anthropic();

         const response = await client.messages.create({
           model: "claude-sonnet-5",
           max_tokens: 16000,
           thinking: {
             type: "adaptive",
             display: "summarized"
           },
           output_config: {
             effort: "high"
           },
           messages: [
             {
               role: "user",
               content: "Are there an infinite number of prime numbers such that n mod 4 == 3?"
             }
           ]
         });

         // 响应包含摘要后的思考块和文本块
         for (const block of response.content) {
           if (block.type === "thinking") {
             console.log(`\nThinking summary: ${block.thinking}`);
           } else if (block.type === "text") {
             console.log(`\nResponse: ${block.text}`);
           }
         }
         ```

         ```csharp C#
         AnthropicClient client = new();

         var response = await client.Messages.Create(new()
         {
             Model = Model.ClaudeSonnet5,
             MaxTokens = 16000,
             Thinking = new ThinkingConfigAdaptive { Display = Display.Summarized },
             OutputConfig = new OutputConfig { Effort = Effort.High },
             Messages =
             [
                 new()
                 {
                     Role = Role.User,
                     Content = "Are there an infinite number of prime numbers such that n mod 4 == 3?",
                 },
             ],
         });

         // 响应包含摘要化的思考块和文本块
         foreach (var block in response.Content)
         {
             if (block.TryPickThinking(out var thinking))
             {
                 Console.WriteLine($"\nThinking summary: {thinking.Thinking}");
             }
             else if (block.TryPickText(out var text))
             {
                 Console.WriteLine($"\nResponse: {text.Text}");
             }
         }
         ```

         ```go Go
         client := anthropic.NewClient()

         response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
         	Model:     anthropic.ModelClaudeSonnet5,
         	MaxTokens: 16000,
         	Thinking: anthropic.ThinkingConfigParamUnion{
         		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{
         			Display: anthropic.ThinkingConfigAdaptiveDisplaySummarized,
         		},
         	},
         	OutputConfig: anthropic.OutputConfigParam{
         		Effort: anthropic.OutputConfigEffortHigh,
         	},
         	Messages: []anthropic.MessageParam{
         		anthropic.NewUserMessage(anthropic.NewTextBlock("Are there an infinite number of prime numbers such that n mod 4 == 3?")),
         	},
         })
         if err != nil {
         	log.Fatal(err)
         }

         // 响应包含摘要化的思考块和文本块
         for _, block := range response.Content {
         	switch block := block.AsAny().(type) {
         	case anthropic.ThinkingBlock:
         		fmt.Printf("\nThinking summary: %s", block.Thinking)
         	case anthropic.TextBlock:
         		fmt.Printf("\nResponse: %s", block.Text)
         	}
         }
         ```

         ```java Java
         import com.anthropic.client.okhttp.AnthropicOkHttpClient;
         import com.anthropic.models.messages.MessageCreateParams;
         import com.anthropic.models.messages.Model;
         import com.anthropic.models.messages.OutputConfig;
         import com.anthropic.models.messages.ThinkingConfigAdaptive;

         void main() {
             var client = AnthropicOkHttpClient.fromEnv();

             var params = MessageCreateParams.builder()
                 .model(Model.CLAUDE_SONNET_5)
                 .maxTokens(16_000)
                 .thinking(ThinkingConfigAdaptive.builder()
                     .display(ThinkingConfigAdaptive.Display.SUMMARIZED)
                     .build())
                 .outputConfig(OutputConfig.builder()
                     .effort(OutputConfig.Effort.HIGH)
                     .build())
                 .addUserMessage("Are there an infinite number of prime numbers such that n mod 4 == 3?")
                 .build();

             var response = client.messages().create(params);

             // 响应包含摘要化的思考块和文本块
             for (var block : response.content()) {
                 block.thinking().ifPresent(thinkingBlock ->
                     IO.println("\nThinking summary: " + thinkingBlock.thinking())
                 );
                 block.text().ifPresent(textBlock ->
                     IO.println("\nResponse: " + textBlock.text())
                 );
             }
         }
         ```

         ```php PHP
         $client = new Client();

         $response = $client->messages->create(
             model: 'claude-sonnet-5',
             maxTokens: 16000,
             thinking: ['type' => 'adaptive', 'display' => 'summarized'],
             outputConfig: ['effort' => 'high'],
             messages: [
                 [
                     'role' => 'user',
                     'content' => 'Are there an infinite number of prime numbers such that n mod 4 == 3?',
                 ],
             ],
         );

         // The response contains summarized thinking blocks and text blocks
         foreach ($response->content as $block) {
             echo match ($block->type) {
                 'thinking' => "\nThinking summary: {$block->thinking}",
                 'text' => "\nResponse: {$block->text}",
                 default => '',
             };
         }
         ```

         ```ruby Ruby
         client = Anthropic::Client.new

         response = client.messages.create(
           model: "claude-sonnet-5",
           max_tokens: 16_000,
           thinking: {type: :adaptive, display: :summarized},
           output_config: {effort: :high},
           messages: [
             {
               role: :user,
               content: "Are there an infinite number of prime numbers such that n mod 4 == 3?"
             }
           ]
         )

         # 响应包含摘要化的思考块和文本块
         response.content.each do |block|
           case block
           in {type: :thinking, thinking:}
             puts "\nThinking summary: #{thinking}"
           in {type: :text, text:}
             puts "\nResponse: #{text}"
           else
           end
         end
         ```
       </CodeGroup>
     </Tab>

     <Tab title="Claude Sonnet 4.6">
       <CodeGroup>
         ```bash cURL
         curl https://api.anthropic.com/v1/messages \
           -H "x-api-key: $ANTHROPIC_API_KEY" \
           -H "anthropic-version: 2023-06-01" \
           -H "content-type: application/json" \
           -d '{
             "model": "claude-sonnet-4-6",
             "max_tokens": 16000,
             "thinking": {
               "type": "enabled",
               "budget_tokens": 10000
             },
             "messages": [
               {
                 "role": "user",
                 "content": "Are there an infinite number of prime numbers such that n mod 4 == 3?"
               }
             ]
           }'
         ```

         ```bash CLI
         ant messages create \
           --transform content --format yaml <<'YAML'
         model: claude-sonnet-4-6
         max_tokens: 16000
         thinking:
           type: enabled
           budget_tokens: 10000
         messages:
           - role: user
             content: Are there an infinite number of prime numbers such that n mod 4 == 3?
         YAML
         ```

         ```python Python
         client = anthropic.Anthropic()

         response = client.messages.create(
             model="claude-sonnet-4-6",
             max_tokens=16000,
             thinking={"type": "enabled", "budget_tokens": 10000},
             messages=[
                 {
                     "role": "user",
                     "content": "Are there an infinite number of prime numbers such that n mod 4 == 3?",
                 }
             ],
         )

         # 响应包含摘要化的思考块和文本块
         for block in response.content:
             match block.type:
                 case "thinking":
                     print(f"\nThinking summary: {block.thinking}")
                 case "text":
                     print(f"\nResponse: {block.text}")
         ```

         ```typescript TypeScript
         const client = new Anthropic();

         const response = await client.messages.create({
           model: "claude-sonnet-4-6",
           max_tokens: 16000,
           thinking: {
             type: "enabled",
             budget_tokens: 10000,
           },
           messages: [
             {
               role: "user",
               content: "Are there an infinite number of prime numbers such that n mod 4 == 3?",
             },
           ],
         });

         // 响应包含摘要化的思考块和文本块
         for (const block of response.content) {
           if (block.type === "thinking") {
             console.log(`\nThinking summary: ${block.thinking}`);
           } else if (block.type === "text") {
             console.log(`\nResponse: ${block.text}`);
           }
         }
         ```

         ```csharp C#
         AnthropicClient client = new();

         var response = await client.Messages.Create(new()
         {
             Model = Model.ClaudeSonnet4_6,
             MaxTokens = 16000,
             Thinking = new ThinkingConfigEnabled(budgetTokens: 10000),
             Messages =
             [
                 new()
                 {
                     Role = Role.User,
                     Content = "Are there an infinite number of prime numbers such that n mod 4 == 3?",
                 },
             ],
         });

         // 响应包含摘要化的思考块和文本块
         foreach (var block in response.Content)
         {
             if (block.TryPickThinking(out var thinking))
             {
                 Console.WriteLine($"\nThinking summary: {thinking.Thinking}");
             }
             else if (block.TryPickText(out var text))
             {
                 Console.WriteLine($"\nResponse: {text.Text}");
             }
         }
         ```

         ```go Go
         client := anthropic.NewClient()

         response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
         	Model:     anthropic.ModelClaudeSonnet4_6,
         	MaxTokens: 16000,
         	Thinking:  anthropic.ThinkingConfigParamOfEnabled(10000),
         	Messages: []anthropic.MessageParam{
         		anthropic.NewUserMessage(anthropic.NewTextBlock("Are there an infinite number of prime numbers such that n mod 4 == 3?")),
         	},
         })
         if err != nil {
         	log.Fatal(err)
         }

         // 响应包含摘要化的思考块和文本块
         for _, block := range response.Content {
         	switch block := block.AsAny().(type) {
         	case anthropic.ThinkingBlock:
         		fmt.Printf("\nThinking summary: %s", block.Thinking)
         	case anthropic.TextBlock:
         		fmt.Printf("\nResponse: %s", block.Text)
         	}
         }
         ```

         ```java Java
         import com.anthropic.client.okhttp.AnthropicOkHttpClient;
         import com.anthropic.models.messages.MessageCreateParams;
         import com.anthropic.models.messages.Model;

         void main() {
             var client = AnthropicOkHttpClient.fromEnv();

             var params = MessageCreateParams.builder()
                 .model(Model.CLAUDE_SONNET_4_6)
                 .maxTokens(16_000)
                 .enabledThinking(10_000)
                 .addUserMessage("Are there an infinite number of prime numbers such that n mod 4 == 3?")
                 .build();

             var response = client.messages().create(params);

             // 响应包含摘要化的思考块和文本块
             for (var block : response.content()) {
                 block.thinking().ifPresent(thinkingBlock ->
                     IO.println("\nThinking summary: " + thinkingBlock.thinking())
                 );
                 block.text().ifPresent(textBlock ->
                     IO.println("\nResponse: " + textBlock.text())
                 );
             }
         }
         ```

         ```php PHP
         $client = new Client();

         $response = $client->messages->create(
             model: 'claude-sonnet-4-6',
             maxTokens: 16000,
             thinking: ['type' => 'enabled', 'budget_tokens' => 10000],
             messages: [
                 [
                     'role' => 'user',
                     'content' => 'Are there an infinite number of prime numbers such that n mod 4 == 3?',
                 ],
             ],
         );

         // 响应包含摘要化的思考块和文本块
         foreach ($response->content as $block) {
             echo match ($block->type) {
                 'thinking' => "\nThinking summary: {$block->thinking}",
                 'text' => "\nResponse: {$block->text}",
                 default => '',
             };
         }
         ```

         ```ruby Ruby
         client = Anthropic::Client.new

         response = client.messages.create(
           model: "claude-sonnet-4-6",
           max_tokens: 16_000,
           thinking: {
             type: :enabled,
             budget_tokens: 10_000
           },
           messages: [
             {
               role: :user,
               content: "Are there an infinite number of prime numbers such that n mod 4 == 3?"
             }
           ]
         )

         # 响应包含摘要化的思考块和文本块
         response.content.each do |block|
           case block
           in {type: :thinking, thinking:}
             puts "\nThinking summary: #{thinking}"
           in {type: :text, text:}
             puts "\nResponse: #{text}"
           else
           end
         end
         ```
       </CodeGroup>
     </Tab>
   </Tabs>

5. **采样参数已移除：** 设置为非默认值的采样参数（`temperature`、`top_p`、`top_k`）不被接受，会返回 400 错误。

6. **网络安全防护措施：** Claude Sonnet 5 是首个具备实时网络安全防护措施的 Sonnet 级模型。涉及被禁止或高风险网络安全主题的请求可能会被拒绝。拒绝以成功的 HTTP 200 响应返回，并带有 `stop_reason: "refusal"`，而不是错误。有关防护措施拦截的内容以及合法安全工作如何申请 Cyber Verification Program，请参阅 [Claude Opus 和 Sonnet 上的实时网络安全防护措施](https://support.claude.com/en/articles/14604842-real-time-cyber-safeguards-on-claude-opus-and-sonnet)。

### 迁移检查清单

* 将模型名称从 `claude-sonnet-4-6` 更新为 `claude-sonnet-5`。
* 针对 Claude Sonnet 5 重新运行[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)。新分词器对相同文本产生的令牌数多约 30%，即使每令牌定价更低，这也可能改变每次请求的成本。确切的增幅取决于内容和工作负载形态。
* 重新审视设定得接近预期输出长度的 `max_tokens` 限制，并在有用的情况下将其提高至 128k 上限（与 Claude Sonnet 4.6 相同）。
* 移除 `thinking: {type: "enabled", budget_tokens: N}` 配置（会返回 400 错误）。自适应思考默认开启；传入 `{type: "disabled"}` 可将其关闭，或使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)控制深度。
* 更新按位置读取内容的响应解析代码，例如 `content[0].text`：开启思考后，`thinking` 块会在 `text` 块之前到达。请改为按 `type` 选择内容块，并在工具使用循环中将 `thinking` 块未经修改地传回；被修改的块会返回 400 错误。
* 确认任何解析 `thinking` 字段的代码仅将其视为显示文本。`thinking.display` 在 Claude Sonnet 5 上默认为 `"omitted"`（在 Claude Sonnet 4.6 上默认为 `"summarized"`），因此思考块到达时 `thinking` 字段为空；设置 `display: "summarized"` 可接收可读的摘要。请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。
* 移除设置为非默认值的 `temperature`、`top_p` 和 `top_k` 参数（它们在 Claude Sonnet 5 上会返回 400 错误）。
* 如果您的工作负载可能涉及网络安全主题，请添加对 `stop_reason: "refusal"` 的处理。
* 在生产部署之前，针对您的典型工作负载重新建立成本基线。
* 对于之前不带思考运行的工作负载，请检查 `max_tokens`。

## 从 Claude Sonnet 4.5 及更早的 Sonnet 模型迁移到 Claude Sonnet 5

如果您要从 Claude Sonnet 4.5 或更早的 Sonnet 模型直接迁移到 Claude Sonnet 5，请应用[从 Claude Sonnet 4.6 迁移到 Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide#migrating-from-claude-sonnet-4-6-to-claude-sonnet-5) 中的变更以及本节中的变更。

<Warning>
  Claude Sonnet 5 的默认 effort 级别为 `high`，而 Sonnet 4.5 没有 effort 参数。迁移时请考虑调整 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)。如果未显式设置，在默认 effort 级别下您可能会遇到更高的延迟。
</Warning>

### 破坏性变更

#### 从 Sonnet 4.5 迁移时

1. **不再支持预填充助手消息**

   <Warning>
     从 Sonnet 4.5 或更早版本迁移时，这是一项破坏性变更。
   </Warning>

   在 Claude Sonnet 4.6 及更高版本模型（包括 Claude Sonnet 5）上，预填充助手消息会返回 `400` 错误。请改用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)、系统提示指令或 `output_config.format`。

   **常见的预填充用例及迁移方式：**

   * **控制输出格式**（强制 JSON/YAML 输出）：使用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)，或对分类任务使用带枚举字段的工具。

   * **消除开场白**（移除"Here is..."之类的短语）：在系统提示中添加直接指令："Respond directly without preamble. Do not start with phrases like 'Here is...', 'Based on...', etc."

   * **避免不当拒绝：** Claude 现在在恰当拒绝方面表现得好得多。在用户消息中给出清晰的提示而不使用预填充应该就足够了。

   * **续写**（恢复被中断的响应）：将续写移至用户消息中："Your previous response was interrupted and ended with `[previous_response]`. Continue from where you left off."

   * **上下文补充 / 角色一致性**（在长对话中刷新上下文）：将之前通过预填充助手消息提供的提醒改为注入到用户轮次中。

2. **工具参数 JSON 转义可能不同**

   <Warning>
     从 Sonnet 4.5 或更早版本迁移时，这是一项破坏性变更。
   </Warning>

   工具参数中的 JSON 字符串转义可能与之前的模型不同。标准 JSON 解析器会自动处理这一点，但自定义的基于字符串的解析可能需要更新。

**扩展思考变更：** 来自 Claude Sonnet 4.5 的 `budget_tokens` 配置（`thinking: {type: "enabled", budget_tokens: N}`）在 Claude Sonnet 5 上不受支持，会返回 400 错误。自适应思考默认开启，因此大多数工作负载完全不需要 `thinking` 配置；请使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)控制思考深度。如果您在 Claude Sonnet 4.5 上不使用扩展思考运行，请传入 `thinking: {type: "disabled"}` 以保留该行为。

#### 从 Claude 3.x 迁移时

3. **移除采样参数**

   <Warning>
     从 Claude 3.x 模型迁移时，这是一项破坏性变更。
   </Warning>

   设置为非默认值的采样参数（`temperature`、`top_p`、`top_k`）在 Claude Sonnet 5 上会返回 400 错误。请将它们从请求中移除，并改用提示来引导模型的行为。

4. **更新工具版本**

   <Warning>
     从 Claude 3.x 模型迁移时，这是一项破坏性变更。
   </Warning>

   更新到最新的工具版本（`text_editor_20250728`、`code_execution_20260521`）。移除任何使用 `undo_edit` 命令的代码。

5. **处理 `refusal` 停止原因**

   更新您的应用程序以[处理 `refusal` 停止原因](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/handle-streaming-refusals)。

6. **针对行为变化更新您的提示**

   Claude 4 模型具有更简洁、直接的沟通风格。请查阅[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)以获取优化指导。

## 从 Claude Haiku 4.5 迁移到 Claude Sonnet 5

Claude Haiku 4.5 与 Claude Sonnet 5 在 API 层面的差异比同一类别内相邻模型之间的差异更大：Claude Haiku 4.5 使用手动[扩展思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)（默认关闭）、200k 令牌上下文窗口以及最多 64k 输出令牌，而 Claude Sonnet 5 默认以[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)运行，默认提供 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)，并支持最多 [128k 输出令牌](https://platform.claude.com/docs/zh-CN/models/overview)。

### 更新您的模型名称

```python
model = "claude-haiku-4-5-20251001"  # Before
model = "claude-sonnet-5"  # After
```

### 变更内容

1. **思考配置：** Claude Haiku 4.5 支持手动扩展思考（`thinking: {type: "enabled", budget_tokens: N}`），并拒绝 `thinking: {type: "adaptive"}`。在 Claude Sonnet 5 上，支持情况正好相反：自适应思考默认开启，而手动扩展思考会返回 400 错误。请移除 `thinking: {type: "enabled", budget_tokens: N}` 配置并依赖默认值，或传入 `thinking: {type: "disabled"}` 以关闭思考。`budget_tokens` 没有直接的替代项；请使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)控制思考深度。Effort 在 Claude Haiku 4.5 上不可用，在 Claude Sonnet 5 上默认为 `high`。

   两类 Claude Haiku 4.5 请求的响应结构都会发生变化。原本不带扩展思考运行的请求现在可能会在第一个 `text` 块之前返回一个或多个 `thinking` 块，因此按位置读取回复的代码（例如 `content[0].text`）必须改为按 `type` 字段选择内容块，并且工具使用循环必须将 `thinking` 块完整且未经修改地与其工具结果一起传回（请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)）。原本使用扩展思考的请求会继续接收 `thinking` 块，但 `thinking.display` 在 Claude Sonnet 5 上默认为 `"omitted"` 而非 `"summarized"`，因此这些块到达时 `thinking` 字段为空；设置 `display: "summarized"` 可继续接收可读的摘要（请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)）。即使未返回思考文本，思考令牌也会按输出令牌计费。

2. **采样参数已移除：** `temperature` 和 `top_p` 在 Claude Haiku 4.5 上可用（一次只能使用一个，不能同时使用）。在 Claude Sonnet 5 上，将 `temperature`、`top_p` 或 `top_k` 设置为非默认值会返回 400 错误。请移除这些参数，并使用提示来引导模型的行为。

3. **助手预填充已移除：** 预填充助手消息在 Claude Haiku 4.5 上可用，但在 Claude Sonnet 5 上会返回 400 错误。请改用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)、系统提示指令或 `output_config.format`。

4. **更大的上下文窗口和输出：** Claude Sonnet 5 默认提供 1M 令牌上下文窗口，高于 Claude Haiku 4.5 的 200k 令牌，并支持最多 128k 输出令牌，高于 64k。Claude Sonnet 5 还使用不同的分词器，因此请重新运行[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)，而不要复用针对 Claude Haiku 4.5 测得的计数。

5. **定价：** Claude Haiku 4.5 的定价为每百万输入/输出令牌 $1/$5 美元。Claude Sonnet 5 的定价为每百万输入/输出令牌 $2/$10 美元。请参阅 [Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

6. **网络安全防护措施：** Claude Sonnet 5 具备实时网络安全防护措施。涉及被禁止或高风险网络安全主题的请求可能会被拒绝，并以成功的 HTTP 200 响应返回，带有 `stop_reason: "refusal"`。有关防护措施拦截的内容以及合法安全工作如何申请 Cyber Verification Program，请参阅 [Claude Opus 和 Sonnet 上的实时网络安全防护措施](https://support.claude.com/en/articles/14604842-real-time-cyber-safeguards-on-claude-opus-and-sonnet)。

### 迁移检查清单

* 将模型名称从 `claude-haiku-4-5-20251001`（或 `claude-haiku-4-5` 别名）更新为 `claude-sonnet-5`。
* 移除 `thinking: {type: "enabled", budget_tokens: N}` 配置（会返回 400 错误）。自适应思考默认开启；传入 `thinking: {type: "disabled"}` 可保留无思考行为，并对原本不带思考运行的工作负载重新审视 `max_tokens`。
* 更新按位置读取内容的响应解析代码，例如 `content[0].text`：开启思考后，`thinking` 块会在 `text` 块之前到达。请改为按 `type` 选择内容块，并在工具使用循环中将 `thinking` 块未经修改地传回；被修改的块会返回 400 错误。
* 如果您的 UI 显示思考内容，请设置 `display: "summarized"`。`thinking.display` 在 Claude Sonnet 5 上默认为 `"omitted"`，否则思考块到达时 `thinking` 字段为空。请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。
* 使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)（默认 `high`）控制思考深度和令牌消耗；它在 Claude Haiku 4.5 上不可用，因此没有现有设置可以沿用。
* 移除 `temperature` 和 `top_p` 设置（非默认值在 Claude Sonnet 5 上会返回 400 错误）。
* 移除任何助手消息预填充（它们在 Claude Sonnet 5 上会返回 400 错误）。
* 针对 Claude Sonnet 5 重新运行[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)，并重新审视 `max_tokens` 限制，您可以将其提高至 128k 上限。
* 如果您的工作负载可能涉及网络安全主题，请添加对 `stop_reason: "refusal"` 的处理。
* 在生产部署之前，针对您的典型工作负载重新建立成本基线；每令牌定价有所不同。
