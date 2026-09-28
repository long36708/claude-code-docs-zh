---
title: 思考
url: https://platform.claude.com/docs/zh-CN/build-with-claude/thinking
description: 了解 Claude 的思考机制：如何开启思考、读取思考输出、通过 effort 调节思考深度，以及如何将思考与工具、缓存和流式传输结合使用。
---

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

单次作答的模型必须一次就把所有事情做对：没有草稿，没有检查，也不能中途改变方向。对于证明、棘手的 bug 或长时间运行的智能体任务，最初的方法往往不是最好的。

"Thinking"（思考）消除了这一限制。思考处于活动状态时，Claude 会在回答之前用自己的话梳理问题：复述所问的内容，尝试不同方法，检查中间结果，并放弃站不住脚的思路。这些推理以 `thinking` 内容块的形式出现在响应之前，Claude 会借助它们生成最终答案。因此，思考能够提升 Claude 在数学、编程、分析和长时间运行的智能体工作等复杂任务上的表现。在这些任务中，答案的质量取决于中间过程，而没有思考时，这些中间过程要么被压缩进响应本身，要么被直接跳过。

思考是有成本的：Claude 用于推理的令牌按输出令牌计费，即使思考文本没有返回给您也是如此，并且这些令牌会与响应文本一起计入 `max_tokens`。本页介绍思考在整个 API 中的行为：如何开启思考、读取其输出，以及如何处理它与工具、流式传输、缓存和上下文窗口之间的交互。

## 思考的工作原理

![思考工作原理示意图：Claude 评估请求并决定是否思考；使用 tool use（工具使用）时，思考可以在工具调用之间反复出现；一个响应先返回 thinking blocks（思考块），然后返回 text blocks（文本块）](https://platform.claude.com/docs/images/how-thinking-works.svg)

Claude 是否会针对某个请求进行思考，以及思考的深度，取决于您的思考配置和请求的复杂程度。

以下是思考在响应中的样子：一个或多个 `thinking` 内容块出现在 `text` 块之前。思考块与其后的 `text` 块一样，仍然是生成的内容，但它与正式响应是分开的。每个思考块还带有一个 `signature` 字段，这是完整推理的加密副本，在多轮对话和工具使用对话中，您需要原样传回该字段（请参阅[思考加密](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-encryption)）：

```json
{
  "content": [
    {
      "type": "thinking",
      "thinking": "Let me break this down. The question has two parts, so I'll start with the simpler one and use its result to constrain the second...",
      "signature": "WaUjzkypQ2mUEVM36O2Txu...."
    },
    {
      "type": "text",
      "text": "Based on my analysis..."
    }
  ]
}
```

您并不总能看到这段文本，而且您看到的也绝不是原始的思维链：思考块中的文本是 [Claude 推理的摘要](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#summarized-thinking)。思考配置中的 `display` 字段控制是否返回该摘要：`"summarized"` 会返回摘要，而 `"omitted"`（许多模型上的默认值）返回的思考块中 `thinking` 字段为空。无论哪种方式，该块的计费方式相同，在多轮对话中的传回方式也相同。有关各模型的默认值和详细信息，请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。

如果 Claude 使用工具，思考也可能出现在工具调用之间。请参阅[思考与工具使用](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)。有关完整的响应格式，请参阅 [Messages API 参考](https://platform.claude.com/docs/zh-CN/api/messages/create)。

## 配置思考

在大多数模型上，思考默认处于开启状态，或者只需设置一个参数即可开启。每个模型接受哪种配置以及默认值是什么，列在故障排除页面的[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#supported-models)中。

在 Claude Opus 5.5、Claude Opus 5、Claude Sonnet 5、Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5 和 Claude Mythos Preview 上，思考已经开启，无需任何配置。这些模型上的 `display` 默认为 `"omitted"`，因此在您主动选择之前，思考文本是隐藏的。要显示思考文本，请设置 `thinking: {"type": "adaptive", "display": "summarized"}`，也就是将以下请求中的[模型字符串](https://platform.claude.com/docs/zh-CN/models/overview)替换掉即可。

在 Claude Opus 4.8、Claude Opus 4.7、Claude Opus 4.6 和 Claude Sonnet 4.6 上，思考默认关闭，直到您设置 `thinking: {type: "adaptive"}`，这会让 Claude 根据请求决定何时思考以及思考的深度。以下示例就是这样做的，同时设置了 `display: "summarized"` 以使思考文本可见，并使用了较宽裕的 `max_tokens`：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-4-8",
      "max_tokens": 16000,
      "thinking": {
        "type": "adaptive",
        "display": "summarized"
      },
      "messages": [
        {
          "role": "user",
          "content": "What is the greatest common divisor of 1071 and 462?"
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages create \
    --model claude-opus-4-8 \
    --max-tokens 16000 \
    --thinking '{type: adaptive, display: summarized}' \
    --message '{role: user, content: "What is the greatest common divisor of 1071 and 462?"}' \
    --transform content \
    --format yaml
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-opus-4-8",
      max_tokens=16000,
      thinking={"type": "adaptive", "display": "summarized"},
      messages=[
          {
              "role": "user",
              "content": "What is the greatest common divisor of 1071 and 462?",
          }
      ],
  )

  for block in response.content:
      match block.type:
          case "thinking":
              print(f"\nThinking: {block.thinking}")
          case "text":
              print(f"\nResponse: {block.text}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 16000,
    thinking: {
      type: "adaptive",
      display: "summarized"
    },
    messages: [
      {
        role: "user",
        content: "What is the greatest common divisor of 1071 and 462?"
      }
    ]
  });

  for (const block of response.content) {
    switch (block.type) {
      case "thinking":
        console.log(`\nThinking: ${block.thinking}`);
        break;
      case "text":
        console.log(`\nResponse: ${block.text}`);
        break;
    }
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus4_8,
      MaxTokens = 16000,
      Thinking = new ThinkingConfigAdaptive { Display = Display.Summarized },
      Messages = [
          new() {
              Role = Role.User,
              Content = "What is the greatest common divisor of 1071 and 462?"
          }
      ]
  };

  var message = await client.Messages.Create(parameters);

  foreach (var block in message.Content)
  {
      if (block.TryPickThinking(out ThinkingBlock? thinking))
      {
          Console.WriteLine($"\nThinking: {thinking.Thinking}");
      }
      else if (block.TryPickText(out TextBlock? text))
      {
          Console.WriteLine($"\nResponse: {text.Text}");
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus4_8,
  	MaxTokens: 16000,
  	Thinking: anthropic.ThinkingConfigParamUnion{
  		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{
  			Display: anthropic.ThinkingConfigAdaptiveDisplaySummarized,
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("What is the greatest common divisor of 1071 and 462?")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  for _, block := range response.Content {
  	switch v := block.AsAny().(type) {
  	case anthropic.ThinkingBlock:
  		fmt.Printf("\nThinking: %s", v.Thinking)
  	case anthropic.TextBlock:
  		fmt.Printf("\nResponse: %s", v.Text)
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.messages.ThinkingConfigAdaptive;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_4_8)
          .maxTokens(16000L)
          .thinking(ThinkingConfigAdaptive.builder()
              .display(ThinkingConfigAdaptive.Display.SUMMARIZED)
              .build())
          .addUserMessage("What is the greatest common divisor of 1071 and 462?")
          .build();

      Message response = client.messages().create(params);

      response.content().forEach(block -> {
          block.thinking().ifPresent(thinkingBlock ->
              IO.println("\nThinking: " + thinkingBlock.thinking())
          );
          block.text().ifPresent(textBlock ->
              IO.println("\nResponse: " + textBlock.text())
          );
      });
  }
  ```

  ```php PHP
  use Anthropic\Messages\TextBlock;
  use Anthropic\Messages\ThinkingBlock;

  $client = new Client();

  $message = $client->messages->create(
      maxTokens: 16000,
      messages: [
          [
              'role' => 'user',
              'content' => 'What is the greatest common divisor of 1071 and 462?'
          ]
      ],
      model: 'claude-opus-4-8',
      thinking: ['type' => 'adaptive', 'display' => 'summarized'],
  );

  foreach ($message->content as $block) {
      switch (true) {
          case $block instanceof ThinkingBlock:
              echo "\nThinking: " . $block->thinking;
              break;
          case $block instanceof TextBlock:
              echo "\nResponse: " . $block->text;
              break;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-4-8",
    max_tokens: 16000,
    thinking: {
      type: "adaptive",
      display: "summarized"
    },
    messages: [
      {
        role: "user",
        content: "What is the greatest common divisor of 1071 and 462?"
      }
    ]
  )

  message.content.each do |block|
    case block
    when Anthropic::Models::ThinkingBlock
      puts "\nThinking: #{block.thinking}"
    when Anthropic::Models::TextBlock
      puts "\nResponse: #{block.text}"
    end
  end
  ```
</CodeGroup>

运行该示例会先打印摘要后的思考，然后打印答案：

```text Output wrap
Thinking: Use Euclidean algorithm.
1071 = 2*462 + 147
462 = 3*147 + 21
147 = 7*21 + 0
GCD = 21

Response: ## Finding GCD of 1071 and 462

I'll use the **Euclidean algorithm**, repeatedly dividing and taking remainders...
```

思考令牌会计入 `max_tokens`，因此请将其设置得足够高，为思考和响应文本都留出空间。请参阅调节页面上的[成本控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#cost-control)以及[思考与上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-the-context-window)。

### 关闭思考

在思考默认开启的 Claude Sonnet 5 上，您可以将其关闭：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-sonnet-5",
      "max_tokens": 4096,
      "thinking": {"type": "disabled"},
      "messages": [
        {
          "role": "user",
          "content": "Summarize this article in one sentence."
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages create \
    --model claude-sonnet-5 \
    --max-tokens 4096 \
    --thinking '{type: disabled}' \
    --message '{role: user, content: "Summarize this article in one sentence."}'
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-sonnet-5",
      max_tokens=4096,
      thinking={"type": "disabled"},
      messages=[{"role": "user", "content": "Summarize this article in one sentence."}],
  )
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-sonnet-5",
    max_tokens: 4096,
    thinking: { type: "disabled" },
    messages: [{ role: "user", content: "Summarize this article in one sentence." }]
  });
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeSonnet5,
      MaxTokens = 4096,
      Thinking = new ThinkingConfigDisabled(),
      Messages = [
          new() {
              Role = Role.User,
              Content = "Summarize this article in one sentence."
          }
      ]
  };

  var message = await client.Messages.Create(parameters);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeSonnet5,
  	MaxTokens: 4096,
  	Thinking: anthropic.ThinkingConfigParamUnion{
  		OfDisabled: &anthropic.ThinkingConfigDisabledParam{},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Summarize this article in one sentence.")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.messages.ThinkingConfigDisabled;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_SONNET_5)
          .maxTokens(4096L)
          .thinking(ThinkingConfigDisabled.builder().build())
          .addUserMessage("Summarize this article in one sentence.")
          .build();

      Message response = client.messages().create(params);
  }
  ```

  ```php PHP
  $client = new Client();

  $message = $client->messages->create(
      maxTokens: 4096,
      messages: [
          [
              'role' => 'user',
              'content' => 'Summarize this article in one sentence.'
          ]
      ],
      model: 'claude-sonnet-5',
      thinking: ['type' => 'disabled'],
  );
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-sonnet-5",
    max_tokens: 4096,
    thinking: { type: "disabled" },
    messages: [
      {
        role: "user",
        content: "Summarize this article in one sentence."
      }
    ]
  )
  ```
</CodeGroup>

Claude Opus 5 同样默认开启思考，并在 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)（努力程度）为 `high` 或更低时接受 `thinking: {type: "disabled"}`。在 `xhigh` 或 `max` effort 下，思考无法关闭：将 `thinking: {type: "disabled"}` 与这些 effort 级别组合使用的请求会返回 400 错误。此限制会在每个请求上强制执行。在禁用思考的情况下，Claude Opus 5 偶尔可能会以纯文本形式输出工具调用，或在可见输出中包含内部 XML 标签。有关提示层面的缓解措施，请参阅[在禁用思考的情况下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)。

Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Opus 5.5、Claude Mythos Preview 均会拒绝 `thinking: {type: "disabled"}`。这些模型上的思考无法关闭。

如果您的模型仅支持扩展思考（请参阅[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#supported-models)），请改用 `type: "enabled"` 和 `budget_tokens` 值进行配置。[扩展思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)页面介绍了该配置。如果任何思考配置返回 400 错误，[思考故障排除](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting)会将每条错误消息与其修复方法对应起来。

## 读取思考输出

### 控制思考显示

思考配置中的 `display` 字段控制思考内容在 API 响应中的返回方式。`display` 在两种模式下都有效：将其与 `type: "adaptive"` 或 `type: "enabled"` 一起设置即可。它接受以下值：

* `"summarized"`：思考块包含[摘要思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#summarized-thinking)文本，即 Claude 推理过程的可读摘要。这是 Claude Opus 4.6、Claude Sonnet 4.6 及更早模型上的默认值。
* `"omitted"`：返回的思考块中 `thinking` 字段为空。`signature` 字段仍携带加密的完整思考内容，以保证多轮对话的连续性（请参阅[思考加密](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-encryption)）。这是 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Opus 5.5、Claude Opus 5、Claude Sonnet 5、Claude Opus 4.8、Claude Opus 4.7 和 [Claude Mythos Preview](https://anthropic.com/glasswing) 上的默认值。
* `"updates"`（beta）：与 `"omitted"` 一样，返回的推理块中 `thinking` 字段为空，而某些模型在工具调用之间编写的简短[进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)会以可读文本形式返回。需要 beta 标头 `thinking-display-updates-2026-08-18`。

当您的应用程序不向用户展示思考内容时，请设置 `display: "omitted"`。其主要好处是在流式传输时缩短首个文本令牌的到达时间：服务器会完全跳过思考令牌的流式传输，只传送签名，因此最终文本响应能更早开始流式传输。

使用 `display: "omitted"` 时，响应包含 `thinking` 字段为空的 `thinking` 块：

```json Output
{
  "content": [
    {
      "type": "thinking",
      "thinking": "",
      "signature": "EosnCkYICxIMMb3LzNrMu..."
    },
    {
      "type": "text",
      "text": "The answer is 12,231."
    }
  ]
}
```

使用省略的思考时，请注意以下几点：

* 您仍需为完整的思考令牌付费。省略只会降低延迟，不会降低成本。
* 如果您在多轮对话中传回思考块，请原样传回。服务器会解密 `signature` 以重建原始思考，用于构建提示（请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)）。您在往返传递的省略块的 `thinking` 字段中放入的任何文本都会被忽略。
* `display` 与 `thinking.type: "disabled"` 一起使用是无效的（没有可显示的内容）。
* 使用 `thinking.type: "adaptive"` 时，如果模型针对简单请求跳过了思考，则无论 `display` 如何设置，都不会生成思考块。
* 使用 `display: "omitted"` 进行流式传输时，不会流式传输任何思考文本。每个思考块会流式传输一个 `thinking` 字符串为空的 `thinking_delta`，然后是其 `signature_delta`。使用 `display: "updates"` 时，只有[进度更新块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)会流式传输携带文本的 `thinking_delta` 事件。有关事件顺序，请参阅[流式传输思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#streaming-thinking)。

<Note>
  无论您设置哪个 `display` 值，`signature` 字段都是相同的。支持在对话的不同轮次之间切换 `display` 值。
</Note>

在 Ruby SDK 中，普通哈希使用 `display:`，如示例所示。类型化的 `ThinkingConfigAdaptive` 类将该参数命名为 `display_`（带尾随下划线，以避免遮蔽 Ruby 的 `Kernel#display`）。无论哪种方式，传输中的字段名仍然是 `display`。

### 摘要思考

当 `display` 为 `"summarized"` 时，您收到的思考文本是 Claude 完整思考过程的摘要，而不是原始思维链。摘要思考在防止滥用的同时，提供了思考的全部智能优势。没有任何 `display` 设置会返回原始思维链。

使用摘要思考时，请注意以下几点：

* 您需要为原始请求生成的完整思考令牌付费，而不是为摘要令牌付费。计费的输出令牌数与您在响应中看到的令牌数并不一致。
* 在 Claude Opus 4.6、Claude Sonnet 4.6 及更早的模型上，思考输出的前几行会更加详细，提供的详细推理对提示工程特别有帮助。[Claude Mythos Preview](https://anthropic.com/glasswing) 从第一个令牌开始就进行摘要，因此其思考块不会显示这段详细的开头。
* 摘要会保留 Claude 思考过程的关键思路，且只增加极少的延迟，因此摘要可以在生成时即时流式传输。
* 摘要由与您在请求中指定的模型不同的另一个模型处理。思考模型看不到摘要输出。
* 随着 Anthropic 不断改进思考功能，摘要行为可能会发生变化。

<Note>
  在极少数需要访问完整思考输出的情况下，请[联系 Anthropic 销售团队](mailto:sales@anthropic.com)。
</Note>

要查看模型的推理过程，请读取 `thinking` 块，而不是通过提示要求在响应文本中给出推理。在 Claude Fable 5.1、Claude Opus 5.5 和 Claude Fable 5 上，试图让模型在响应文本中输出其内部推理的请求可能会被拒绝，并返回 `stop_details.category: "reasoning_extraction"`。有关字段参考和处理指南，请参阅[拒绝类别](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response)。

### 流式传输思考

思考可与[流式传输](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)配合使用。思考块以 `content_block_delta` 事件中的 `thinking_delta` 事件形式进行流式传输，并在该块的 `content_block_stop` 之前紧跟一个 `signature_delta` 事件。文本块随后照常流式传输。

![带思考的 streaming（流式传输）事件序列示意图：思考块打开，仅当 display 设置返回文本时（summarized，或对于进度更新块为 updates），thinking deltas（思考增量）才携带文本，单个 signature delta（签名增量）关闭该块，然后流式传输 text deltas（文本增量）](https://platform.claude.com/docs/images/how-thinking-streams.svg)

以下示例使用自适应思考流式传输响应，并在思考增量和文本增量到达时将其打印出来：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-4-8",
      "max_tokens": 16000,
      "stream": true,
      "thinking": {
        "type": "adaptive",
        "display": "summarized"
      },
      "messages": [
        {
          "role": "user",
          "content": "What is the greatest common divisor of 1071 and 462?"
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages create \
    --model claude-opus-4-8 \
    --max-tokens 16000 \
    --thinking '{type: adaptive, display: summarized}' \
    --message '{role: user, content: "What is the greatest common divisor of 1071 and 462?"}' \
    --stream \
    --format jsonl
  ```

  ```python Python
  client = anthropic.Anthropic()

  with client.messages.stream(
      model="claude-opus-4-8",
      max_tokens=16000,
      thinking={"type": "adaptive", "display": "summarized"},
      messages=[
          {
              "role": "user",
              "content": "What is the greatest common divisor of 1071 and 462?",
          }
      ],
  ) as stream:
      for event in stream:
          match event.type:
              case "content_block_start":
                  print(f"\nStarting {event.content_block.type} block...")
              case "content_block_delta":
                  delta = event.delta
                  match delta.type:
                      case "thinking_delta":
                          print(delta.thinking, end="", flush=True)
                      case "text_delta":
                          print(delta.text, end="", flush=True)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const stream = client.messages.stream({
    model: "claude-opus-4-8",
    max_tokens: 16000,
    thinking: { type: "adaptive", display: "summarized" },
    messages: [{ role: "user", content: "What is the greatest common divisor of 1071 and 462?" }]
  });

  for await (const event of stream) {
    switch (event.type) {
      case "content_block_start":
        console.log(`\nStarting ${event.content_block.type} block...`);
        break;
      case "content_block_delta":
        switch (event.delta.type) {
          case "thinking_delta":
            process.stdout.write(event.delta.thinking);
            break;
          case "text_delta":
            process.stdout.write(event.delta.text);
            break;
        }
        break;
    }
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus4_8,
      MaxTokens = 16000,
      Thinking = new ThinkingConfigAdaptive { Display = Display.Summarized },
      Messages = [new() { Role = Role.User, Content = "What is the greatest common divisor of 1071 and 462?" }]
  };

  await foreach (var rawEvent in client.Messages.CreateStreaming(parameters))
  {
      if (rawEvent.TryPickContentBlockStart(out var start))
      {
          Console.WriteLine($"\nStarting {start.ContentBlock.Type} block...");
      }
      else if (rawEvent.TryPickContentBlockDelta(out var delta))
      {
          if (delta.Delta.TryPickThinking(out var thinkingDelta))
          {
              Console.Write(thinkingDelta.Thinking);
          }
          else if (delta.Delta.TryPickText(out var textDelta))
          {
              Console.Write(textDelta.Text);
          }
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  stream := client.Messages.NewStreaming(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus4_8,
  	MaxTokens: 16000,
  	Thinking: anthropic.ThinkingConfigParamUnion{
  		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{
  			Display: anthropic.ThinkingConfigAdaptiveDisplaySummarized,
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("What is the greatest common divisor of 1071 and 462?")),
  	},
  })

  for stream.Next() {
  	event := stream.Current()
  	switch eventVariant := event.AsAny().(type) {
  	case anthropic.ContentBlockStartEvent:
  		fmt.Printf("\nStarting %s block...\n", eventVariant.ContentBlock.Type)
  	case anthropic.ContentBlockDeltaEvent:
  		switch deltaVariant := eventVariant.Delta.AsAny().(type) {
  		case anthropic.ThinkingDelta:
  			fmt.Print(deltaVariant.Thinking)
  		case anthropic.TextDelta:
  			fmt.Print(deltaVariant.Text)
  		}
  	}
  }
  if err := stream.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.messages.ThinkingConfigAdaptive;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_4_8)
          .maxTokens(16000L)
          .thinking(ThinkingConfigAdaptive.builder()
              .display(ThinkingConfigAdaptive.Display.SUMMARIZED)
              .build())
          .addUserMessage("What is the greatest common divisor of 1071 and 462?")
          .build();

      try (var streamResponse = client.messages().createStreaming(params)) {
          streamResponse.stream().forEach(event -> {
              switch (event.type().value()) {
                  case CONTENT_BLOCK_START -> {
                      var startEvent = event.asContentBlockStart();
                      var block = startEvent.contentBlock();
                      switch (block.type().value()) {
                          case THINKING -> IO.println("\nStarting thinking block...");
                          case TEXT -> IO.println("\nStarting text block...");
                      }
                  }
                  case CONTENT_BLOCK_DELTA -> {
                      var deltaEvent = event.asContentBlockDelta();
                      deltaEvent.delta().thinking().ifPresent(td ->
                          IO.print(td.thinking())
                      );
                      deltaEvent.delta().text().ifPresent(td ->
                          IO.print(td.text())
                      );
                  }
              }
          });
      }
  }
  ```

  ```php PHP
  use Anthropic\Messages\RawContentBlockDeltaEvent;
  use Anthropic\Messages\RawContentBlockStartEvent;
  use Anthropic\Messages\TextDelta;
  use Anthropic\Messages\ThinkingDelta;

  $client = new Client();

  $stream = $client->messages->createStream(
      maxTokens: 16000,
      messages: [
          ['role' => 'user', 'content' => 'What is the greatest common divisor of 1071 and 462?']
      ],
      model: 'claude-opus-4-8',
      thinking: ['type' => 'adaptive', 'display' => 'summarized'],
  );

  foreach ($stream as $event) {
      switch (true) {
          case $event instanceof RawContentBlockStartEvent:
              echo "\nStarting {$event->contentBlock->type} block...\n";
              break;
          case $event instanceof RawContentBlockDeltaEvent:
              switch (true) {
                  case $event->delta instanceof ThinkingDelta:
                      echo $event->delta->thinking;
                      break;
                  case $event->delta instanceof TextDelta:
                      echo $event->delta->text;
                      break;
              }
              break;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  stream = client.messages.stream(
    model: "claude-opus-4-8",
    max_tokens: 16000,
    thinking: { type: "adaptive", display: "summarized" },
    messages: [
      { role: "user", content: "What is the greatest common divisor of 1071 and 462?" }
    ]
  )

  stream.each do |event|
    case event
    when Anthropic::Streaming::ThinkingEvent
      print event.thinking
    when Anthropic::Streaming::TextEvent
      print event.text
    end
  end
  ```
</CodeGroup>

要在流式传输后重新组装带有签名的完整思考块，请使用您的 SDK 的消息累积辅助工具 `stream.get_final_message()` (typescript: `stream.finalMessage()`; ruby: `stream.accumulated_message`; csharp: `.Aggregate()`; go: `message.Accumulate(event)`; java, php: `MessageAccumulator`)，而不是自己拼接增量。

<Accordion title="完整的流式传输事件跟踪">
  ```sse Output
  event: message_start
  data: {"type": "message_start", "message": {"id": "msg_01...", "type": "message", "role": "assistant", "content": [], "model": "claude-opus-4-8", "stop_reason": null, "stop_sequence": null}}

  event: content_block_start
  data: {"type": "content_block_start", "index": 0, "content_block": {"type": "thinking", "thinking": "", "signature": ""}}

  event: content_block_delta
  data: {"type": "content_block_delta", "index": 0, "delta": {"type": "thinking_delta", "thinking": "I need to find the GCD of 1071 and 462 using the Euclidean algorithm.\n\n1071 = 2 × 462 + 147"}}

  event: content_block_delta
  data: {"type": "content_block_delta", "index": 0, "delta": {"type": "thinking_delta", "thinking": "\n462 = 3 × 147 + 21\n147 = 7 × 21 + 0\n\nSo GCD(1071, 462) = 21"}}

  // Additional thinking deltas...

  event: content_block_delta
  data: {"type": "content_block_delta", "index": 0, "delta": {"type": "signature_delta", "signature": "EqQBCgIYAhIM1gbcDa9GJwZA2b..."}}

  event: content_block_stop
  data: {"type": "content_block_stop", "index": 0}

  event: content_block_start
  data: {"type": "content_block_start", "index": 1, "content_block": {"type": "text", "text": ""}}

  event: content_block_delta
  data: {"type": "content_block_delta", "index": 1, "delta": {"type": "text_delta", "text": "The greatest common divisor of 1071 and 462 is **21**."}}

  // Additional text deltas...

  event: content_block_stop
  data: {"type": "content_block_stop", "index": 1}

  event: message_delta
  data: {"type": "message_delta", "delta": {"stop_reason": "end_turn", "stop_sequence": null}}

  event: message_stop
  data: {"type": "message_stop"}
  ```
</Accordion>

设置 `display: "omitted"` 时，思考块打开，到达一个 `thinking` 字符串为空的 `thinking_delta`，随后是一个 `signature_delta`，然后该块关闭。文本流式传输紧接着开始：

```sse Output
event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":"","signature":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"signature_delta","signature":"EosnCkYICxIMMb3LzNrMu..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"text","text":""}}
```

使用 `display: "updates"`（beta）时，推理块的流式传输方式与 `"omitted"` 下相同。每个[进度更新块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)会在其引出的 `tool_use` 块之前，以 `thinking_delta` 事件的形式流式传输其文本。在进度更新块打开之前出现几秒钟的停顿是正常的：

```sse Output
event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"thinking","thinking":"","signature":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"thinking_delta","thinking":"Confirmed the retry path never refreshes the expired token. Editing auth.py to add the refresh call."}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"signature_delta","signature":"Es8CCkYICxIM..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}

event: content_block_start
data: {"type":"content_block_start","index":2,"content_block":{"type":"tool_use","id":"toolu_01D7FLrfh4GYq7yT1ULFeyMV","name":"edit_file","input":{}}}
```

在 `"updates"` 下，只要某个块的任一 `thinking_delta` 事件携带非空文本，就应将该块视为进度更新。

<Note>
  在启用思考的情况下使用流式传输时，您可能会注意到文本有时以较大的块到达，与较小的逐令牌传送交替出现。这是预期行为，尤其是对于思考内容。

  流式传输系统会分批处理内容，这可能会延迟流式传输事件并将其分组，从而形成这种"块状"传送模式。
</Note>

有关流式传输的一般机制，请参阅[流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)。

## 思考与 effort

`thinking` 参数控制 Claude 在回答之前是否在[思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)中进行思考；`effort` 参数控制 Claude 在整个响应中投入多少工作量，在自适应模式下，这包括它思考的频率和深度。不要将 `adaptive` 作为 `effort` 的值传递：`adaptive` 是一种思考模式，而不是努力程度级别。

要了解每个 effort 级别对思考行为的影响，请参阅[调节思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost)页面上的[各级别思考行为表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#effort-levels)。[Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 页面记录了该参数本身，包括每个模型支持哪些级别。在 Claude Opus 4.5（唯一支持 effort 的仅扩展思考模型）上，effort 与 `budget_tokens` 组合使用。请参阅[预算规则与调优](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#budget-rules-and-tuning)。

这两个控制项以这种方式分开后，请选择与您的目标相匹配的那一个：

* **降低启用思考的工作负载的成本或延迟：** 首先降低 `effort`。它会整体缩减响应，包括思考。
* **Claude 思考得太少或太浅：** 提高 `effort`，或参阅调节页面上的[调节 Claude 的思考频率](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#tuning-thinking-behavior)。
* **您需要完全关闭思考：** 在允许的模型上使用 `thinking: {type: "disabled"}`（请参阅[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#supported-models)）。
* **您需要对支出设置硬性上限：** 使用 `max_tokens`。Effort 是软性指导，`max_tokens` 是严格限制。

## 思考与工具使用

思考可与[工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)配合使用，让 Claude 能够推理工具的选择并处理工具结果。有两项约束：

1. **工具选择限制（手动模式）：** 使用手动扩展思考（`thinking: {type: "enabled"}`）时，工具使用仅支持 `tool_choice: {"type": "auto"}`（默认值）或 `tool_choice: {"type": "none"}`。使用 `tool_choice: {"type": "any"}` 或 `tool_choice: {"type": "tool", "name": "..."}` 会导致错误，因为这些选项会强制使用工具，而这与手动扩展思考不兼容。自适应思考（包括在默认开启思考的模型上）支持强制工具使用，但 Claude Opus 5.5、Claude Fable 5.1 和 Claude Mythos 5.1 除外（请参阅[响应预填充与强制工具使用](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#limits-and-feature-compatibility)）。
2. **保留思考块：** 返回工具结果时，您必须将助手消息中的思考块完整且未经修改地传回 API。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。

**一个工具使用循环就是一个助手轮次。** 从模型的角度来看，在 Claude 完成其完整响应之前，助手轮次不会结束，而完整响应可能包含多次工具调用和结果。以下整个序列是一个助手轮次：

```text wrap
User: "What's the weather in Paris?"
Assistant: [thinking] + [tool_use: get_weather]
User: [tool_result: "20°C, sunny"]
Assistant: [text: "The weather in Paris is 20°C and sunny"]
```

整个轮次以单一思考模式运行：您不能在轮次中途切换思考，包括在工具使用循环期间。在扩展（手动）模式下，API 还会强制要求启用思考的请求中最后一个助手轮次以思考块开头。自适应模式放宽了这一要求：任何助手轮次都不需要以思考块开头。

**轮次中途的冲突会平稳降级。** 如果您在轮次中途切换思考（例如，在发送工具调用和返回其结果之间），API 不会报错，而是会静默地为该请求禁用思考。为了保持模型质量，API 可能会剥离会造成无效轮次结构的思考块，或者在对话历史与启用思考不兼容时禁用思考。要确认思考是否处于活动状态，请检查响应中是否存在 `thinking` 块。

**在轮次之间切换，而不是在轮次内部切换。** 在每个轮次开始时规划您的思考策略。完成助手轮次后，再为下一个轮次更改思考配置：

```text wrap
User: "What's the weather?"
Assistant: [tool_use] (thinking disabled)
User: [tool_result]
Assistant: [text: "It's sunny"]
User: "What about tomorrow?"
Assistant: [thinking] + [text: "..."] (thinking enabled - new turn)
```

切换思考模式还会使提示缓存失效。请参阅[思考与提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-prompt-caching)。

### 保留思考块

当 Claude 调用工具时，它会暂停构建响应以等待外部信息。当您返回工具结果时，Claude 会继续构建同一个响应，因此其先前的推理必须仍然存在。请将每个 `thinking` 块与其所伴随的 `tool_use` 块一起，完整且未经修改地传回 API。这一点很重要，原因有二：

1. **推理连续性：** 思考块记录了导致工具请求的逐步推理。包含这些块可以让 Claude 从中断处继续推理。
2. **上下文维护：** 在 API 结构中，工具结果以用户消息的形式出现，但它们属于同一个连续的推理流程。保留思考块可以在多次 API 调用之间维持这一流程。

简而言之：

* **必需：** 在工具使用轮次内，传回思考块。
* **推荐：** 跨轮次时，传回所有内容。
* **允许：** 在工具使用之外，省略先前轮次的思考。

您无需自行修剪旧的思考。在多轮对话中传回所有思考块，API 会自动过滤它们，保留维持模型推理所需的块，并且只对实际展示给 Claude 的块收取输入令牌费用。保留哪些先前轮次的块因模型而异。请参阅[各模型的思考块保留](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)。要覆盖默认行为，请使用 [`clear_thinking_20251015` 上下文编辑策略](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#thinking-block-clearing)。

在最新的助手消息中，连续 `thinking` 块的顺序必须与模型在原始请求中生成的顺序一致：您不能重新排列、编辑或部分删除它们。这也包括 [`redacted_thinking` 块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#redacted-thinking-blocks)。

<Note>
  经过修改的思考块会被拒绝并返回 400 错误。有关确切的错误消息、常见原因和修复方法，请参阅[400 错误提示思考块不能被修改](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#error-thinking-blocks-modified)。唯一的例外是：放入[省略](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)块的空 `thinking` 字段中的文本会被忽略，而不会被拒绝。
</Note>

有关包含所有 SDK 代码的完整两轮演练，请参阅[工具和多轮工作流中的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-tool-workflows#two-turn-tool-use-round-trip)。该演练定义了一个工具，接收包含思考和工具使用的响应，并将助手轮次与工具结果一起传回。

### 交错思考

"Interleaved thinking"（交错思考）让 Claude 能够在工具调用之间进行思考，在对每个工具结果采取行动之前先对其进行推理。借助交错思考，Claude 可以：

* 在决定下一步做什么之前，对工具调用的结果进行推理
* 将多个工具调用串联起来，并在其间穿插推理步骤
* 根据中间结果做出更细致的决策

<Note>
  连续的工具调用并不需要交错思考。无论是否使用交错思考，Claude 都可以串联工具调用。交错改变的是思考块在工具调用之间出现的位置，而不是工具调用能否串联。
</Note>

使用自适应思考时，交错思考在所有支持自适应思考的模型上都会自动启用，无需 beta 标头。在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Mythos Preview、Claude Opus 5.5、Claude Opus 5、Claude Opus 4.8 和 Claude Opus 4.7 上，工具调用之间的推理始终出现在思考块中。Claude Haiku 4.5 不支持交错思考。在使用手动扩展思考的模型上，交错思考需要 beta 标头，并且会改变思考预算的计算方式。[手动模式下的交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#interleaved-thinking)介绍了各模型的规则以及特定平台的标头行为。

使用交错思考时，思考配额可以覆盖整个助手轮次，而不仅仅是单个响应。交错思考仅支持[通过 Messages API 使用的工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。

有关展示交错思考在双工具工作流中带来哪些变化的实例对比，请参阅[交错思考如何改变流程](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-tool-workflows#how-interleaved-thinking-changes-the-flow)。

### 工具调用之间的进度更新

在 Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5.5、Claude Fable 5 上，模型可以在工具调用之间编写 "progress update"（进度更新）。进度更新是一两句话，说明模型刚刚发现了什么以及接下来要做什么，它是写给观察智能体的人看的，而不是推理内容。每条进度更新都作为独立的 `thinking` 块返回，带有自己的 `signature`，与同一位置的任何推理块分开。它紧挨在其引出的 `tool_use` 或 `server_tool_use` 块之前。每次工具调用之前最多有一条进度更新，并且模型可以跳过其中任何一条。进度更新不是[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)：无论工具调用之间是否出现推理块，进度更新都会出现，并且一个响应可以同时包含两者。

进度更新块包含的内容取决于 [`display`](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)：

| `display`              | 推理块             | 进度更新块           |
| ---------------------- | --------------- | --------------- |
| `"omitted"`（这些模型上的默认值） | `thinking` 字段为空 | `thinking` 字段为空 |
| `"updates"`（beta）      | `thinking` 字段为空 | 摘要文本            |
| `"summarized"`         | 摘要文本            | 摘要文本，与推理块无法区分   |

对于隐藏推理、并在每一步向用户显示状态行的智能体界面，请使用 `display: "updates"`。在该设置下，任何带有非空文本的 `thinking` 块都是进度更新，因此只渲染这些块即可。该功能处于 beta 阶段，需要 beta 标头 `thinking-display-updates-2026-08-18`（在 Amazon Bedrock、Google Cloud 和 Microsoft Foundry 上，请按照 [Beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)中的说明传递 beta 值）。如果没有该标头，该值会被拒绝，并返回与未知 `display` 值相同的 400 `invalid_request_error`。

```json
{
  "model": "claude-fable-5-1",
  "max_tokens": 16000,
  "thinking": { "type": "adaptive", "display": "updates" },
  "tools": [
    {
      "name": "edit_file",
      "description": "Replace the contents of a file in the repository.",
      "input_schema": {
        "type": "object",
        "properties": {
          "path": { "type": "string" },
          "content": { "type": "string" }
        },
        "required": ["path", "content"]
      }
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": "The login test fails after an hour of uptime. Find out why and fix it."
    }
  ]
}
```

在 `"updates"` 下，`tool_result` 之后的响应开头如下所示。第一个块是推理，保持为空，与 `"omitted"` 下相同。第二个块携带文本，因此它是进度更新。在 `"summarized"` 下，两个块都携带文本；在 `"omitted"` 下，两个块都为空。

```json Output
{
  "content": [
    {
      "type": "thinking",
      "thinking": "",
      "signature": "EqMBCkYICxIM..."
    },
    {
      "type": "thinking",
      "thinking": "Confirmed the retry path never refreshes the expired token. Editing auth.py to add the refresh call.",
      "signature": "Es8CCkYICxIM..."
    },
    {
      "type": "tool_use",
      "id": "toolu_01D7FLrfh4GYq7yT1ULFeyMV",
      "name": "edit_file",
      "input": { "path": "auth.py", "content": "..." }
    }
  ]
}
```

使用进度更新时，请注意以下几点：

* 与其他任何 `thinking` 块一样，将进度更新块连同助手轮次的其余部分一起原样传回。
* 您收到的文本是进度更新的摘要，通常为一两句话。不要依赖其长度。进度更新按其完整长度计入 `usage.output_tokens`，而不是按摘要的长度。
* 在任何 `display` 值下，进度更新块都可能以空的 `thinking` 字段返回。对于空块，不要渲染任何内容。在 `"updates"` 下，它看起来与空的推理块相同，无需单独处理。
* 当响应在工具调用或工具结果之后不久因 `max_tokens`、`model_context_window_exceeded` 或 `stop_sequence` 而停止时，其最后一个块可能是一个进度更新块，代表模型尚未完成的工作。在 `"updates"` 和 `"summarized"` 下，其文本恰好为 `This part of the response was interrupted before it finished.`，您可以像显示其他更新一样显示它。在 `"omitted"` 下，它为空。要继续，请将助手轮次原样传回，并追加一条新的 `user` 消息（为该轮次中的每个 `tool_use` 块附上一个 `tool_result`）。
* [流式传输](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#streaming-thinking)时，预计在进度更新块打开之前会有几秒钟的停顿。请参阅[流式传输思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#streaming-thinking)中的 `"updates"` 跟踪。
* 在较高的 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 下以及在较长的工具链中，这些模型编写的进度更新会更少。如果您的界面依赖于进度更新，请参阅[要求提供面向用户的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#ask-for-user-facing-progress-updates)；对于 Claude Opus 5.5，请参阅[面向用户的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5#user-facing-progress-updates)。

### 各模型的思考块保留

先前助手轮次中的思考块是否默认保留在上下文中，取决于模型：

* **保留所有先前轮次：** Claude Opus 4.5 及更高版本的 Opus 模型、Claude Sonnet 4.6 及更高版本的 Sonnet 模型、Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5 和 Claude Mythos Preview。
* **仅保留最后一个轮次：** 更早的 Opus 和 Sonnet 模型，以及截至 Claude Haiku 4.5 的所有 Haiku 模型。当您传回较旧的思考块时，API 会自动将其剥离。您无需自行删除。

保留带来两个好处：

* **缓存优化：** 保留的思考块能够在工具使用期间实现缓存命中，因为它们会随工具结果一起传回，并在整个助手轮次中增量缓存，从而在多步骤工作流中节省令牌。
* **不影响智能：** 保留思考块不会对模型性能产生任何负面影响。

代价是上下文占用：在保留所有轮次的模型上，长对话会消耗更多上下文空间，因为保留的思考块与其他对话历史一样计为输入（请参阅[思考与上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-the-context-window)）。在这两种机制下，该行为都是自动的。无需更改代码或使用 beta 标头，您应继续按照[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)中的说明，传回完整且未经修改的思考块。要在任一方向上覆盖默认行为，请使用[思考块清除](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#thinking-block-clearing)。

**在对话中途切换模型。** 切换模型时（例如在[分类器拒绝回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)之后），请继续原样传回思考块。思考块只能由生成它的模型及某些其他模型读取，API 会忽略或丢弃目标模型无法读取的块。对于 Claude Fable 5.1 和 Claude Mythos 5.1，切换方向很重要：它们可以读取所有更早模型的思考块，而任何更早的模型都无法读取它们的思考块，因此向上切换到这两个模型会保留对话的推理，向下切换则会丢弃推理（请参阅[被丢弃的块如何计费和报告](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#switching-models)）。Claude Opus 5.5 可以读取 Claude Opus 5 的思考块以及更早的 Opus、Sonnet 和 Haiku 模型的思考块，但无法读取 Claude Fable 和 Claude Mythos 模型的思考块；在 Claude API 上，Claude Fable 5.1 和 Claude Mythos 5.1 可以读取 Claude Opus 5.5 的思考块。在 Claude API 上从 Claude Opus 5.5 向上切换到 Claude Fable 5.1 会保留先前轮次的推理；从 Claude Fable 5.1 切换到 Claude Opus 5.5 则会丢弃推理。仅当目标模型会忽略（而非丢弃）先前的 `thinking` 和 `redacted_thinking` 块时，才为节省输入令牌而自行剥离这些块；在兑换[回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)时切勿这样做，因为兑换要求请求体保持不变。

## 保留的思考

"[Preserved thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)"（保留的思考）决定模型能否使用您从先前轮次传回的思考块。从 Claude Fable 5.1 开始，API 会检查请求中每个 `thinking` 或 `redacted_thinking` 块的 `signature`，确认以下两点：

* **生成该块的模型。** 每个模型可以读取自己的思考块，以及一组固定的其他模型的思考块。Claude Fable 5.1 可以读取来自 Claude Opus 5 的块，在 Claude API 上还可以读取来自 Claude Opus 5.5 的块；Claude Opus 5 和 Claude Opus 5.5 都无法读取来自 Claude Fable 5.1 的块。对于当前模型无法读取的块，API 会将其丢弃，既不报错也不计费。请参阅[在对话中途切换模型](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#switching-models)。
* **在该块之前发送的所有内容。** 只有当顶层 `system` 提示、`tools` 以及该块之前的消息保持不变时，该块才保持有效。如果其中任何一项发生变化，该块及之后的所有思考块都将失效，API 会以 400 错误拒绝请求或丢弃失效的块，具体取决于您的选择。请参阅[保持前缀不变](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#prefix-check)。

模型检查适用于所有账户。对于在 2026 年 8 月 31 日 00:00 UTC 或之后创建的账户，API 默认强制执行前缀检查。对于较早的账户，API 仅对设置了 `thinking.block_binding.prefix_mismatch_behavior` 的请求强制执行该检查。无论您的账户创建于何时，都请将您的集成设计为仅追加（append-only）模式，这样同一套代码可以在所有账户上运行，包括默认强制执行检查的较新账户。

要保持思考有效，请将每个助手轮次完全按照收到时的样子传回，并且只在 `messages` 的末尾添加新消息。如果您的代码自行构建 `messages` 数组，保留的思考页面介绍了以下内容：

* [哪些操作算作编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#what-counts-as-an-edit)，以及[如何检查您的代码是否进行了编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#how-to-tell-whether-your-integration-is-impacted)。
* [替代每种常见编辑的 API 功能](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#replace-prefix-edits)：用于新指令和每轮提醒的对话中途系统消息、用于工具变更的 `tool_addition` 和 `tool_removal` 块、用于 effort 变更的每条消息 `output_config`，以及用于裁剪的服务器端压缩和上下文编辑。
* [客户端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#custom-compaction-on-the-client)：哪些模式能保持思考有效，哪些不能。
* [`thinking-binding-controls-2026-08-01` beta 标头](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#preserved-thinking-controls)。它会在思考配置中添加一个 `block_binding.prefix_mismatch_behavior` 字段（`"error"` 或 `"drop_block"`），并在每个响应中添加一个 `input_transformations` 数组。该数组列出 API 丢弃的每个思考块，以及未通过前缀检查但被放行的每个思考块。

## 思考与提示缓存

"Prompt caching"（[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)）与思考之间存在几种特定的交互方式。以下规则在两种思考模式下均适用。

**配置更改会使缓存失效。** 思考配置和解析后的 [`effort`](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)（努力程度）级别会被渲染到提示本身中，因此更改其中任何一项都会开始一个新的缓存前缀。在 `adaptive`、`enabled` 和 `disabled` 之间切换、更改 `budget_tokens` 以及更改 effort 值，都会使缓存断点失效：消息级断点总是会未命中，而工具断点和 "system prompt"（系统提示）断点也可能未命中，具体取决于模型在何处渲染该配置。请将任何思考配置或顶层 effort 的更改视为缓存重新开始。在支持[按消息设置 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta) 的模型上，通过 `messages` 中的 `role: "system"` 消息传递的 effort 更改会保持已缓存的前缀不变。保持相同配置的连续请求会保留缓存，并且将参数显式设置为其默认值等同于省略该参数。在任一[保留思考条件](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking)下被 API 丢弃的思考块，会从该块所在位置起改变已缓存的前缀。原样传回的块会保持缓存完整。包含用量输出的完整演示请参阅[引导思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#prompt-caching)页面。

**思考块会与工具结果一起被缓存。** 在 "tool use"（工具使用）循环中，当您发出包含工具结果的后续请求时，就会发生缓存。此时，之前的对话历史（包括其中的思考块）可以被缓存，并且这些已缓存的思考块在从缓存中读取时，会在您的用量指标中计为输入令牌。即使没有显式的 `cache_control` 标记，这一过程也会自动发生，并且对于常规思考和交错思考的行为相同。需要权衡的是：您在响应中不会再看到的思考块，在从缓存中读取时仍会计入输入令牌用量。

**之前的块是否保留在上下文中取决于模型。** 这由[保留默认行为](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)决定。在保留全部轮次的模型上，之前轮次的思考块会保持缓存并保留在上下文中。在仅保留最后一轮的模型上，一旦您发送了一条不是工具结果的用户消息，所有之前的思考块都会从上下文中移除。在这些模型上，如下所示的对话：

```text wrap
User: ["What's the weather in Paris?"],
Assistant: [thinking_block_1] + [tool_use block 1],
User: [tool_result_1, cache=True],
Assistant: [thinking_block_2] + [text block 2],
User: [Text response, cache=True]
```

会被当作思考块从未存在过一样进行处理：

```text wrap
User: ["What's the weather in Paris?"],
Assistant: [tool_use block 1],
User: [tool_result_1, cache=True],
Assistant: [text block 2],
User: [Text response, cache=True]
```

在保留全部轮次的模型上，同样的请求会将 `thinking_block_1` 和 `thinking_block_2` 保留在上下文和缓存中。

**降级会从可缓存的历史中移除思考内容。** 如果思考在轮次中途被禁用，而您在当前工具使用轮次中传入了思考内容，则该思考内容会被移除，并且该请求的思考将保持禁用状态（请参阅[优雅降级](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)）。[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)会放大缓存失效的影响，因为思考块可能出现在多次工具调用之间。

<Tip>
  思考密集型任务的完成时间通常会超过默认的 5 分钟缓存有效期。请考虑使用 [1 小时缓存时长](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#1-hour-cache-duration)，以便在较长的思考会话和多步骤工作流中保持缓存命中。
</Tip>

## 思考与上下文窗口

`max_tokens` 包含 Claude 在当前轮次中生成的所有思考内容，并作为严格限制执行。在 Claude 4.5 及更新的模型上，如果输入令牌加上 `max_tokens` 超过了 "context window"（上下文窗口）大小，API 仍会接受该请求。如果生成过程随后达到上下文窗口限制，它会以 `stop_reason: "model_context_window_exceeded"` 停止，而不是返回错误。在较早的模型上，API 则会返回验证错误。请参阅[处理停止原因](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons)。

思考内容如何计入上下文窗口取决于其生成时间：

* **当前轮次的思考**始终计入 `max_tokens`，按输出令牌计费，并在生成它的轮次中占用上下文窗口空间。
* **之前轮次的思考**取决于[保留默认行为](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)。在[保留所有之前轮次的模型](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)上，之前的思考块会保留在上下文中，计入上下文窗口，并像对话历史的其余部分一样按输入令牌计费。在仅保留最后一轮的模型上，当您传回较早的思考块时，API 会自动将其移除，因此它们不会占用上下文窗口空间或输入令牌。

实际应用中：

* 在保留全部轮次的模型上，请像对待普通对话历史一样为思考内容规划上下文窗口预算，因为它本质上就是对话历史。长时间的智能体会话会在上下文中不断累积思考内容。如果需要回收空间，请使用[思考块清除](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#thinking-block-clearing)。
* 在仅保留最后一轮的模型上，思考只是每轮的成本：每轮的思考计入该轮的 `max_tokens`，之后便从上下文窗口中移出。

以下图表展示了仅保留最后一轮（移除）的机制。第一张图展示了一个多轮对话：每轮的思考块在输出中生成，但不会被带入后续轮次的输入中。

![在会移除之前思考块的模型上的思考示意图（"context window"（上下文窗口））：每轮的 "thinking block"（思考块）在输出中生成，且不会被带入后续轮次的输入中](https://platform.claude.com/docs/images/context-window-thinking.svg)

第二张图展示了同一机制在工具使用场景下的情况：在助手轮次期间，思考内容与其工具结果一起保留在上下文中，然后在下一个用户轮次时被移除。

![在会移除之前思考块的模型上结合 "tool use"（工具使用）的思考示意图："thinking block"（思考块）与其 "tool result"（工具结果）一起保留，然后在下一个用户轮次时被移除](https://platform.claude.com/docs/images/context-window-thinking-tools.svg)

请使用[令牌计数 API](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting) 获取针对您特定用例的准确计数，尤其是对于包含思考内容的多轮对话。

## 思考加密

完整的思考内容经过加密，并在每个思考块的 `signature` 字段中返回。当您传回思考块时，API 会使用该签名来验证这些思考块是否由 Claude 生成。

使用签名时，请注意以下几点：

* 只有在[结合工具使用思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)时，才严格需要传回思考块。否则，您可以省略之前轮次的思考块。如果您确实传回了它们，API 是保留还是移除它们取决于模型（请参阅[各模型的思考块保留行为](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)）。请使用[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)进行配置。
* 传回思考块时，请完全按照接收时的原样传回所有内容，以保持一致性并避免潜在问题。
* 在 "streaming"（[流式传输](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#streaming-thinking)）响应时，签名会在 `content_block_stop` 事件之前，以 `content_block_delta` 事件中的 `signature_delta` 形式到达。
* 在 Claude 4 及更高版本的模型中，`signature` 值明显比之前的模型更长。
* `signature` 字段是不透明的：请勿解释或解析它。
* `signature` 值在各平台之间兼容（Claude API、[Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 和 [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)）。在一个平台上生成的值可以在另一个平台上使用。

## 已编辑的思考块

除了常规的 `thinking` 块之外，当 Claude 的部分推理内容因安全原因被编辑时，API 可能会返回 `redacted_thinking` 块。`redacted_thinking` 块在 `data` 字段中包含加密的思考内容，不含可读文本：

```json
{
  "type": "redacted_thinking",
  "data": "..."
}
```

`data` 字段是不透明且经过加密的。与常规思考块上的 `signature` 字段一样，在结合[工具](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)继续多轮对话时，请将 `redacted_thinking` 块原样传回 API。

<Tip>
  如果您的代码在往返传递包含工具使用的响应时按类型过滤内容块（例如 `block.type == "thinking"`），请同时包含 `redacted_thinking` 块。仅按 `block.type == "thinking"` 过滤会静默丢弃 `redacted_thinking` 块，并破坏[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)中所述的多轮协议。
</Tip>

<Note>
  `redacted_thinking` 块是一种独立的内容块类型，在思考内容因安全原因被编辑时返回。这与 [`display: "omitted"`](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display) 选项不同，后者返回的是 `thinking` 字段为空的常规 `thinking` 块。
</Note>

## 限制与功能兼容性

### 采样参数

在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Mythos Preview、Claude Opus 5.5、Claude Opus 5、Claude Opus 4.8、Claude Opus 4.7 和 Claude Sonnet 5 上，无论是否使用思考，非默认的 `temperature`、`top_p` 或 `top_k` 值都会在每个请求上返回 400 错误。在较旧的模型上，该限制仅在启用思考时适用：`temperature` 和 `top_k` 与思考不兼容，而 `top_p` 允许设置为 0.95 到 1 之间的值。

### 响应预填充与强制工具使用

启用思考时，您无法预填充助手响应。强制工具使用（`tool_choice: {"type": "any"}` 或 `{"type": "tool", ...}`）与手动扩展思考不兼容，但可与自适应思考配合使用。例外情况是 Claude Opus 5.5、Claude Fable 5.1 和 Claude Mythos 5.1，它们会在每个请求上以 400 错误拒绝强制工具使用。在这些模型上，请改用 `tool_choice: {"type": "auto"}` 配合[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)或[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)。请参阅[思考与工具使用](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)。

### 输出限制

每个模型接受的 `max_tokens` 最高可达此处列出的上限。在 [Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#extended-output-beta) 上，`output-300k-2026-03-24` [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)会为列出了批处理上限的模型提高该上限。

| Model                 | Max output tokens | Batches beta ceiling |
| :-------------------- | :---------------- | :------------------- |
| Claude Fable 5.1      | 128K              | —                    |
| Claude Mythos 5.1     | 128K              | —                    |
| Claude Fable 5        | 128K              | —                    |
| Claude Mythos 5       | 128K              | —                    |
| Claude Mythos Preview | 128K              | Not available        |
| Claude Opus 5.5       | 128K              | 300K                 |
| Claude Opus 5         | 128K              | 300K                 |
| Claude Opus 4.8       | 128K              | 300K                 |
| Claude Opus 4.7       | 128K              | 300K                 |
| Claude Opus 4.6       | 128K              | 300K                 |
| Claude Opus 4.5       | 64K               | Not available        |
| Claude Sonnet 5       | 128K              | 300K                 |
| Claude Sonnet 4.6     | 128K              | 300K                 |
| Claude Sonnet 4.5     | 64K               | Not available        |
| Claude Haiku 4.5      | 64K               | Not available        |

有关旧版模型的限制，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。

### 长请求

当 `max_tokens` 大于 21,333 时，SDK 要求使用流式传输，以避免长时间运行的请求出现 HTTP 超时。这是客户端验证，而非 API 限制。如果您不需要增量处理事件，请使用 `.stream()` (java: `.createStreaming()`; csharp: `.CreateStreaming()`; go: `.NewStreaming()`; php: `->createStream()`) 配合 `.get_final_message()` (typescript: `.finalMessage()`; ruby: `.accumulated_message`; csharp: `.Aggregate()`; go: `message.Accumulate(event)`; java, php: `MessageAccumulator`) 来获取完整的 `Message` 对象，而无需自己从各个事件中组装。请参阅[流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming#get-the-final-message-without-handling-events)。启用思考时，响应时间预计会更长，因为生成思考块会增加处理时间。对于每个请求的思考量超过约 32k 令牌的工作负载，请使用[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)以避免网络问题：此类请求的运行时间可能长到触及系统超时和打开连接数限制。

## 后续步骤

<CardGroup cols={2}>
  <Card title="引导思考" icon="compass" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost">
    通过 effort 级别、系统提示指导和按消息引导，控制 Claude 思考的频率和深度，并了解思考的成本和定价。
  </Card>

  <Card title="工具与多轮工作流中的思考" icon="wrench" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-tool-workflows">
    逐步了解一个正确保留思考块的完整两轮工具使用往返过程，并了解交错思考如何改变该流程。
  </Card>

  <Card title="保留思考" icon="stack" href="https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking">
    了解您的 Messages API 集成是否会编辑对话历史，并将每处编辑替换为能使早期思考块保持有效的 API 功能。
  </Card>

  <Card title="思考故障排除" icon="hammer" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting">
    诊断并修复最常见的思考故障：配置导致的 400 错误、思考块为空或缺失、max\_tokens 停止以及缓存未命中。
  </Card>

  <Card title="Effort" icon="sliders" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    使用 effort 参数控制 Claude 在响应时使用的令牌数量，在响应的详尽程度与令牌效率之间进行权衡。
  </Card>
</CardGroup>
