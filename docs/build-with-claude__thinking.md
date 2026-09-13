---
title: 思考
url: https://platform.claude.com/docs/zh-CN/build-with-claude/thinking
description: 了解 Claude 的思考如何工作：开启思考、读取思考输出、通过 effort 调节思考深度，以及将思考与工具、缓存和流式传输结合使用。
---

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

一个单次生成答案的模型必须在第一次尝试时就把所有事情做对：没有草稿、没有检查、也无法中途改变方向。对于一个证明、一个棘手的 bug 或一个长时间的智能体任务来说，第一种方法往往不是最好的方法。

"Thinking"（思考）消除了这一限制。当思考处于活动状态时，Claude 会在回答之前用自己的话梳理问题：它会重述所问的内容、尝试各种方法、检查中间结果，并放弃站不住脚的路径。这些推理以 `thinking` 内容块的形式出现在响应之前，Claude 会借助它来生成最终答案。这就是为什么思考能够提升在数学、编码、分析和长时间运行的智能体工作等复杂任务上的表现——在这些任务中，答案的质量取决于中间工作，而这些中间工作否则会被压缩进响应本身或被跳过。

思考是有成本的：Claude 用于推理的令牌按输出令牌计费，即使思考文本没有返回给您也是如此，并且它们与响应文本一起计入 `max_tokens`。本页介绍思考在整个 API 层面的行为：如何开启、如何读取其输出，以及如何管理它与工具、流式传输、缓存和上下文窗口之间的交互。

## 思考如何工作

![思考工作原理示意图：Claude 评估请求并决定是否思考；在工具使用（tool use）场景下，思考可以在工具调用之间反复出现；一个响应先返回 thinking 块，然后返回 text 块](https://platform.claude.com/docs/images/how-thinking-works.svg)

Claude 是否对某个请求进行思考，以及思考的深度，取决于您的思考配置和请求的复杂程度。

以下是思考在响应中的样子：一个或多个 `thinking` 内容块出现在 `text` 块之前。thinking 块仍然是生成的内容，就像它后面的 `text` 块一样，但它与规范响应是分开的。每个 thinking 块还带有一个 `signature` 字段，这是完整推理的加密副本，您需要在多轮对话和工具使用对话中原样传回（请参阅[思考加密](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-encryption)）：

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

您并不总能看到这段文本，而且您看到的永远不是原始的思维链：thinking 块中的文本是 [Claude 推理的摘要](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#summarized-thinking)。思考配置上的 `display` 字段控制是否返回该摘要：`"summarized"` 会返回它，而 `"omitted"`（许多模型上的默认值）会返回 `thinking` 字段为空的 thinking 块。无论哪种方式，该块的计费方式相同，在多轮对话中传回的方式也相同。有关各模型的默认值和详细信息，请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。

如果 Claude 使用工具，思考也可以出现在工具调用之间。请参阅[思考与工具使用](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)。有关完整的响应格式，请参阅 [Messages API 参考](https://platform.claude.com/docs/zh-CN/api/messages/create)。

## 配置思考

在大多数模型上，思考默认开启，或者只需一个参数即可开启。每个模型接受哪种配置以及默认值是什么，列在故障排除页面的[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#supported-models)中。

在 Claude Opus 5、Claude Sonnet 5、Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5 和 Claude Mythos Preview 上，思考已经开启，无需配置。在这些模型上 `display` 默认为 `"omitted"`，因此思考文本在您选择启用之前是隐藏的。通过 `thinking: {"type": "adaptive", "display": "summarized"}` 选择启用，这与下面的请求完全相同，只需替换[模型字符串](https://platform.claude.com/docs/zh-CN/models/overview)。

在 Claude Opus 4.8、Claude Opus 4.7、Claude Opus 4.6 和 Claude Sonnet 4.6 上，思考默认关闭，直到您设置 `thinking: {type: "adaptive"}`，这会让 Claude 根据请求自行决定何时思考以及思考的深度。以下示例就是这样做的，同时设置 `display: "summarized"` 以使思考文本可见，并使用了宽裕的 `max_tokens`：

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
      if block.type == "thinking":
          print(f"\nThinking: {block.thinking}")
      elif block.type == "text":
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
    if (block.type === "thinking") {
      console.log(`\nThinking: ${block.thinking}`);
    } else if (block.type === "text") {
      console.log(`\nResponse: ${block.text}`);
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
      if ($block->type === 'thinking') {
          echo "\nThinking: " . $block->thinking;
      } elseif ($block->type === 'text') {
          echo "\nResponse: " . $block->text;
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
    case block.type
    when :thinking
      puts "\nThinking: #{block.thinking}"
    when :text
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

思考令牌计入 `max_tokens`，因此请将其设置得足够高，为思考和响应文本都留出空间。请参阅调节页面上的[成本控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#cost-control)以及[思考与上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-the-context-window)。

### 关闭思考

在默认开启思考的 Claude Sonnet 5 上，您可以将其关闭：

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

Claude Opus 5 同样默认开启思考，并在 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)（努力程度）为 `high` 或更低时接受 `thinking: {type: "disabled"}`。在 `xhigh` 或 `max` effort 下，思考无法关闭：将 `thinking: {type: "disabled"}` 与这些 effort 级别组合的请求会返回 400 错误。此限制适用于 Claude Opus 5 及更高版本的模型，并在每个请求上强制执行。在禁用思考的情况下，Claude Opus 5 偶尔会以纯文本形式输出工具调用，或在其可见输出中包含内部 XML 标签。有关提示层面的缓解措施，请参阅[在禁用思考的情况下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)。

Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5 和 Claude Mythos Preview 会拒绝 `thinking: {type: "disabled"}`。在这些模型上无法关闭思考。

如果您的模型仅支持 "extended thinking"（扩展思考）（请参阅[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#supported-models)），请改用 `type: "enabled"` 和一个 `budget_tokens` 值进行配置。[扩展思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)页面介绍了该配置。如果任何思考配置返回 400 错误，[思考故障排除](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting)会将每条错误消息与其修复方法对应起来。

## 读取思考输出

### 控制思考显示

思考配置上的 `display` 字段控制思考内容在 API 响应中的返回方式。`display` 在两种模式下都有效：可与 `type: "adaptive"` 或 `type: "enabled"` 一起设置。它接受以下值：

* `"summarized"`：thinking 块包含[摘要思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#summarized-thinking)文本，即 Claude 推理的可读摘要。这是 Claude Opus 4.6、Claude Sonnet 4.6 及更早模型上的默认值。
* `"omitted"`：返回的 thinking 块的 `thinking` 字段为空。`signature` 字段仍然携带加密的完整思考，以保证多轮连续性（请参阅[思考加密](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-encryption)）。这是 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Opus 5、Claude Sonnet 5、Claude Opus 4.8、Claude Opus 4.7 和 [Claude Mythos Preview](https://anthropic.com/glasswing) 上的默认值。
* `"updates"`（测试版）：推理块返回时 `thinking` 字段为空，与 `"omitted"` 相同，而某些模型在工具调用之间写出的简短[进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)会以可读文本形式返回。需要测试版请求头 `thinking-display-updates-2026-08-18`。

当您的应用程序不向用户展示思考内容时，请设置 `display: "omitted"`。主要好处是流式传输时首个文本令牌的到达时间更快：服务器完全跳过思考令牌的流式传输，只传递签名，因此最终文本响应会更早开始流式传输。

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

使用省略思考时请注意以下几点：

* 您仍需为完整的思考令牌付费。省略降低的是延迟，而不是成本。
* 如果您在多轮对话中传回 thinking 块，请原样传回。服务器会解密 `signature` 以重建原始思考用于构建提示（请参阅[保留 thinking 块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)）。您在往返传递的省略块的 `thinking` 字段中放置的任何文本都会被忽略。
* `display` 与 `thinking.type: "disabled"` 一起使用是无效的（没有可显示的内容）。
* 当使用 `thinking.type: "adaptive"` 且模型对简单请求跳过思考时，无论 `display` 如何设置，都不会生成 thinking 块。
* 使用 `display: "omitted"` 进行流式传输时，不会发出 `thinking_delta` 事件。使用 `display: "updates"` 时，只有[进度更新块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)会流式传输 `thinking_delta` 事件。有关事件序列，请参阅[流式传输思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#streaming-thinking)。

<Note>
  无论您设置哪个 `display` 值，`signature` 字段都是相同的。支持在对话的不同轮次之间切换 `display` 值。
</Note>

在 Ruby SDK 中，普通哈希如示例所示使用 `display:`。类型化的 `ThinkingConfigAdaptive` 类将该参数命名为 `display_`（带尾部下划线，以避免遮蔽 Ruby 的 `Kernel#display`）。无论哪种方式，传输字段仍然是 `display`。

### 摘要思考

当 `display` 为 `"summarized"` 时，您收到的思考文本是 Claude 完整思考过程的摘要，而不是原始的思维链。摘要思考在防止滥用的同时提供了思考的全部智能优势。没有任何 `display` 设置会返回原始思维链。

使用摘要思考时请注意以下几点：

* 您需要为原始请求生成的完整思考令牌付费，而不是摘要令牌。计费的输出令牌数与您在响应中看到的令牌数不一致。
* 在 Claude Opus 4.6、Claude Sonnet 4.6 及更早的模型上，思考输出的前几行更为详细，提供了对提示工程特别有帮助的详细推理。[Claude Mythos Preview](https://anthropic.com/glasswing) 从第一个令牌开始就进行摘要，因此其 thinking 块不会显示这种详细的前言。
* 摘要保留了 Claude 思考过程的关键思路，且增加的延迟极小，因此摘要可以在到达时进行流式传输。
* 摘要由与您请求中指定的模型不同的模型处理。思考模型看不到摘要输出。
* 随着 Anthropic 不断改进思考功能，摘要行为可能会发生变化。

<Note>
  在极少数需要访问完整思考输出的情况下，请[联系 Anthropic 销售团队](mailto:sales@anthropic.com)。
</Note>

要查看模型的推理，请读取 `thinking` 块，而不是通过提示要求在响应文本中给出推理。在 Claude Fable 5.1 和 Claude Fable 5 上，试图将模型的内部推理作为响应文本的一部分引出的请求可能会被拒绝，并返回 `stop_details.category: "reasoning_extraction"`。有关字段参考和处理指南，请参阅[拒绝类别](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response)。

### 流式传输思考

思考可与 [streaming](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)（流式传输）配合使用。thinking 块以 `content_block_delta` 事件内的 `thinking_delta` 事件形式流式传输，随后在该块的 `content_block_stop` 之前紧跟一个 `signature_delta` 事件。text 块随后照常流式传输。

![带思考的流式传输（streaming）事件序列示意图：thinking 块打开，仅当 display 设置返回文本时（summarized，或对进度更新块而言为 updates）才流式传输 thinking delta，单个 signature delta 关闭该块，然后流式传输 text delta](https://platform.claude.com/docs/images/how-thinking-streams.svg)

以下示例使用自适应思考流式传输响应，并在思考和文本增量到达时打印它们：

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
          if event.type == "content_block_start":
              print(f"\nStarting {event.content_block.type} block...")
          elif event.type == "content_block_delta":
              if event.delta.type == "thinking_delta":
                  print(event.delta.thinking, end="", flush=True)
              elif event.delta.type == "text_delta":
                  print(event.delta.text, end="", flush=True)
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
    if (event.type === "content_block_start") {
      console.log(`\nStarting ${event.content_block.type} block...`);
    } else if (event.type === "content_block_delta") {
      if (event.delta.type === "thinking_delta") {
        process.stdout.write(event.delta.thinking);
      } else if (event.delta.type === "text_delta") {
        process.stdout.write(event.delta.text);
      }
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
              if (event.contentBlockStart().isPresent()) {
                  var startEvent = event.contentBlockStart().get();
                  var block = startEvent.contentBlock();
                  if (block.isThinking()) {
                      IO.println("\nStarting thinking block...");
                  } else if (block.isText()) {
                      IO.println("\nStarting text block...");
                  }
              } else if (event.contentBlockDelta().isPresent()) {
                  var deltaEvent = event.contentBlockDelta().get();
                  deltaEvent.delta().thinking().ifPresent(td ->
                      IO.print(td.thinking())
                  );
                  deltaEvent.delta().text().ifPresent(td ->
                      IO.print(td.text())
                  );
              }
          });
      }
  }
  ```

  ```php PHP
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
      if ($event->type === 'content_block_start') {
          echo "\nStarting {$event->contentBlock->type} block...\n";
      } elseif ($event->type === 'content_block_delta') {
          if ($event->delta->type === 'thinking_delta') {
              echo $event->delta->thinking;
          } elseif ($event->delta->type === 'text_delta') {
              echo $event->delta->text;
          }
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

要在流式传输后重新组装带有签名的完整 thinking 块，请在有的情况下使用您的 SDK 的消息累积辅助方法（例如 Python 中的 `stream.get_final_message()` 或 TypeScript 中的 `stream.finalMessage()`），而不是自己拼接增量。

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

当设置了 `display: "omitted"` 时，thinking 块打开，到达一个 `signature_delta`，然后该块关闭，没有任何 `thinking_delta` 事件。文本流式传输随即开始：

```sse Output
event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":"","signature":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"signature_delta","signature":"EosnCkYICxIMMb3LzNrMu..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"text","text":""}}
```

使用 `display: "updates"`（测试版）时，推理块的流式传输方式与 `"omitted"` 下相同。每个[进度更新块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)会在它所引出的 `tool_use` 块之前以 `thinking_delta` 事件形式流式传输其文本。在进度更新块打开之前出现几秒钟的停顿是正常的：

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

在 `"updates"` 下，只要某个块的某个 `thinking_delta` 事件携带非空文本，就将该块视为进度更新。

<Note>
  在启用思考的情况下使用流式传输时，您可能会注意到文本有时以较大的块到达，与较小的逐令牌传递交替出现。这是预期行为，尤其是对于思考内容。

  流式传输系统以批次处理内容，这可能会延迟流式传输事件并将其分组为这种"成块"的传递模式。
</Note>

有关一般的流式传输机制，请参阅[流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)。

## 思考与 effort

`thinking` 参数控制 Claude 在回答之前是否在[思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)中进行思考；`effort` 参数控制 Claude 在整个响应中投入多少工作量，在自适应模式下，这包括它思考的频率和深度。不要将 `adaptive` 作为 `effort` 的值传递：`adaptive` 是一种思考模式，而不是努力程度级别。

要了解每个 effort 级别对思考行为的影响，请参阅[调节思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost)页面上的[各级别思考行为表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#effort-levels)。[Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 页面记录了该参数本身，包括每个模型支持哪些级别。在 Claude Opus 4.5（唯一支持 effort 的仅扩展思考模型）上，effort 与 `budget_tokens` 组合使用。请参阅[预算规则与调优](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#budget-rules-and-tuning)。

这两个控制项以这种方式分开后，请选择与您的目标相匹配的那个：

* **在启用思考的工作负载上降低成本或延迟：** 首先降低 `effort`。它会缩减整个响应，包括思考。
* **Claude 思考得太少或太浅：** 提高 `effort`，或参阅调节页面上的[调节 Claude 思考的频率](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#tuning-thinking-behavior)。
* **您需要完全关闭思考：** 在允许的模型上使用 `thinking: {type: "disabled"}`（请参阅[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#supported-models)）。
* **您需要对支出设置硬性上限：** 使用 `max_tokens`。Effort 是软性指导。`max_tokens` 是严格限制。

## 思考与工具使用

思考可与 [tool use](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)（工具使用）配合使用，让 Claude 能够推理工具选择并处理工具结果。有两个约束：

1. **工具选择限制（手动模式）：** 使用手动扩展思考（`thinking: {type: "enabled"}`）的工具使用仅支持 `tool_choice: {"type": "auto"}`（默认值）或 `tool_choice: {"type": "none"}`。使用 `tool_choice: {"type": "any"}` 或 `tool_choice: {"type": "tool", "name": "..."}` 会导致错误，因为这些选项会强制工具使用，这与手动扩展思考不兼容。自适应思考（包括在默认开启思考的模型上）支持强制工具使用，但 Claude Fable 5.1 和 Claude Mythos 5.1 除外（请参阅[响应预填充与强制工具使用](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#limits-and-feature-compatibility)）。
2. **保留 thinking 块：** 当您返回工具结果时，必须将助手消息中的 thinking 块完整且未经修改地传回 API。请参阅[保留 thinking 块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。

**一个工具使用循环是一个助手轮次。** 从模型的角度来看，一个助手轮次直到 Claude 完成其完整响应才算结束，这可能包括多次工具调用和结果。整个序列是一个单一的助手轮次：

```text wrap
User: "What's the weather in Paris?"
Assistant: [thinking] + [tool_use: get_weather]
User: [tool_result: "20°C, sunny"]
Assistant: [text: "The weather in Paris is 20°C and sunny"]
```

整个轮次在单一思考模式下运行：您不能在轮次中途切换思考，包括在工具使用循环期间。在扩展（手动）模式下，API 还会强制要求启用思考的请求的最后一个助手轮次以 thinking 块开头。自适应模式放宽了这一点：没有任何助手轮次需要以 thinking 块开头。

**轮次中途的冲突会优雅降级。** 如果您在轮次中途切换思考（例如，在发送工具调用和返回其结果之间），API 不会报错。相反，它会静默地为该请求禁用思考。为了保持模型质量，API 可能会剥离会造成无效轮次结构的 thinking 块，或在对话历史与启用思考不兼容时禁用思考。要确认思考是否处于活动状态，请检查响应中是否存在 `thinking` 块。

**在轮次之间切换，而不是在轮次之内。** 在每个轮次开始时规划您的思考策略。完成助手轮次，然后为下一个轮次更改思考配置：

```text wrap
User: "What's the weather?"
Assistant: [tool_use] (thinking disabled)
User: [tool_result]
Assistant: [text: "It's sunny"]
User: "What about tomorrow?"
Assistant: [thinking] + [text: "..."] (thinking enabled - new turn)
```

切换思考模式也会使提示缓存失效。请参阅[思考与提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-prompt-caching)。

### 保留 thinking 块

当 Claude 调用工具时，它会暂停构建响应以等待外部信息。当您返回工具结果时，Claude 会继续构建同一个响应，因此其先前的推理必须仍然存在。请将每个 `thinking` 块连同它所伴随的 `tool_use` 块一起，完整且未经修改地传回 API。这很重要，原因有二：

1. **推理连续性：** thinking 块捕获了导致工具请求的逐步推理。包含它们可以让 Claude 从中断的地方继续推理。
2. **上下文维护：** 工具结果在 API 结构中以用户消息的形式出现，但它们是一个连续推理流的一部分。保留 thinking 块可以在多次 API 调用之间维持该流程。

简而言之：

* **必需：** 在工具使用轮次内，传回 thinking 块。
* **推荐：** 跨轮次时，传回所有内容。
* **允许：** 在工具使用之外，省略先前轮次的思考。

您不需要自己修剪旧的思考。在多轮对话中传回所有 thinking 块，API 会自动过滤它们，保留维持模型推理所需的块，并且仅对实际展示给 Claude 的块计费输入令牌。保留哪些先前轮次的块因模型而异。请参阅[各模型的 thinking 块保留](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)。要覆盖默认行为，请使用 [`clear_thinking_20251015` 上下文编辑策略](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#thinking-block-clearing)。

在最新的助手消息中，连续 `thinking` 块的序列必须与模型在原始请求中生成的内容一致：您不能重新排列、编辑或部分删除它们。这包括 [`redacted_thinking` 块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#redacted-thinking-blocks)。

<Note>
  修改过的 thinking 块会被拒绝并返回 400 错误。有关确切消息、常见原因和修复方法，请参阅 [400 错误提示 thinking 块不能被修改](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#error-thinking-blocks-modified)。唯一的例外：放置在[省略](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)块的空 `thinking` 字段中的文本会被忽略而不是被拒绝。
</Note>

有关包含每种 SDK 代码的完整两轮演练，请参阅[工具和多轮工作流中的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-tool-workflows#two-turn-tool-use-round-trip)。它定义了一个工具，接收一个思考加工具使用的响应，并将助手轮次连同工具结果一起回传。

### 交错思考

"Interleaved thinking"（交错思考）让 Claude 能够在工具调用之间进行思考，在对每个工具结果采取行动之前先对其进行推理。借助交错思考，Claude 可以：

* 在决定下一步做什么之前，对工具调用的结果进行推理
* 将多个工具调用串联起来，中间穿插推理步骤
* 根据中间结果做出更细致的决策

<Note>
  连续的工具调用不需要交错思考。无论有没有交错思考，Claude 都可以串联工具调用。交错改变的是 thinking 块在工具调用之间出现的位置，而不是工具调用能否串联。
</Note>

使用自适应思考时，交错思考在每个支持自适应思考的模型上都是自动的。不需要测试版请求头。在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Mythos Preview、Claude Opus 5、Claude Opus 4.8 和 Claude Opus 4.7 上，工具调用之间的推理始终出现在 thinking 块中。Claude Haiku 4.5 不支持交错思考。在使用手动扩展思考的模型上，交错需要测试版请求头，并会改变思考预算的计算方式。[手动模式下的交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#interleaved-thinking)介绍了各模型的规则和特定平台的请求头行为。

使用交错思考时，思考分配可以跨越整个助手轮次，而不是单个响应。交错思考仅支持[通过 Messages API 使用的工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。

有关展示交错思考在双工具工作流中带来哪些变化的实例对比，请参阅[交错思考如何改变流程](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-tool-workflows#how-interleaved-thinking-changes-the-flow)。

### 工具调用之间的进度更新

在 Claude Fable 5.1、Claude Mythos 5.1 和 Claude Fable 5 上，模型可以在工具调用之间写出进度更新。进度更新是一两句话，说明模型刚刚发现了什么以及接下来要做什么，是写给观察智能体的人看的，而不是作为推理。每条进度更新都作为独立的 `thinking` 块返回，带有自己的 `signature`，与同一位置的任何推理块分开。它紧接在它所引出的 `tool_use` 或 `server_tool_use` 块之前。每次工具调用之前最多有一条进度更新，模型可以跳过其中任何一条。进度更新不是[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)：无论工具调用之间是否出现推理块，它们都会出现，并且一个响应可以同时包含两者。

进度更新块包含什么内容取决于 [`display`](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)：

| `display`              | 推理块             | 进度更新块           |
| ---------------------- | --------------- | --------------- |
| `"omitted"`（这些模型上的默认值） | 空 `thinking` 字段 | 空 `thinking` 字段 |
| `"updates"`（测试版）       | 空 `thinking` 字段 | 摘要文本            |
| `"summarized"`         | 摘要文本            | 摘要文本，无法与推理块区分   |

对于保持推理隐藏并在每一步向用户显示状态行的智能体界面，请使用 `display: "updates"`。在该设置下，任何带有非空文本的 `thinking` 块都是进度更新，因此只渲染这些块，不渲染其他内容。它处于测试阶段，需要测试版请求头 `thinking-display-updates-2026-08-18`（在 Amazon Bedrock、Google Cloud 和 Microsoft Foundry 上，请按照[测试版请求头](https://platform.claude.com/docs/zh-CN/api/beta-headers)中的说明传递测试版值）。如果没有它，该值会被拒绝，并返回与未知 `display` 值相同的 400 `invalid_request_error`。

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

在 `"updates"` 下，紧跟在 `tool_result` 之后的响应开头如下所示。第一个块是推理，保持为空，就像在 `"omitted"` 下一样。第二个块携带文本，因此它是进度更新。在 `"summarized"` 下两个块都携带文本，在 `"omitted"` 下两个块都为空。

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

使用进度更新时请注意以下几点：

* 将进度更新块与助手轮次的其余部分一起原样传回，就像任何其他 `thinking` 块一样。
* 您收到的文本是进度更新的摘要，通常是一两句话。不要依赖其长度。进度更新按其完整长度计入 `usage.output_tokens`，而不是摘要的长度。
* 在任何 `display` 值下，进度更新块都可能以空 `thinking` 字段返回。对于空块不渲染任何内容。在 `"updates"` 下，它看起来与空推理块相同，不需要单独处理。
* 当响应在工具调用或工具结果之后不久因 `max_tokens`、`model_context_window_exceeded` 或 `stop_sequence` 而停止时，其最后一个块可能是一个进度更新块，代表模型尚未完成的工作。在 `"updates"` 和 `"summarized"` 下，其文本恰好是 `This part of the response was interrupted before it finished.`，您可以像任何其他更新一样显示它。在 `"omitted"` 下它为空。要继续，请原样传回助手轮次并追加一条新的 `user` 消息（为该轮次中的每个 `tool_use` 块附带一个 `tool_result`）。
* 在[流式传输](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#streaming-thinking)时，预计在进度更新块打开之前会有几秒钟的停顿。请参阅[流式传输思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#streaming-thinking)中的 `"updates"` 跟踪。
* 这些模型在较高 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 下以及在长工具链中写出的进度更新较少。如果您的界面依赖它们，请参阅[请求面向用户的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#ask-for-user-facing-progress-updates)。

### 各模型的 thinking 块保留

先前助手轮次的 thinking 块是否默认保留在上下文中取决于模型：

* **保留所有先前轮次：** Claude Opus 4.5 及更高版本的 Opus 模型、Claude Sonnet 4.6 及更高版本的 Sonnet 模型、Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5 和 Claude Mythos Preview。
* **仅保留最后一个轮次：** 更早的 Opus 和 Sonnet 模型，以及直到 Claude Haiku 4.5 的所有 Haiku 模型。当您传回较旧的 thinking 块时，API 会自动剥离它们。您不需要自己移除它们。

保留带来两个好处：

* **缓存优化：** 保留的 thinking 块在工具使用期间能够实现缓存命中，因为它们与工具结果一起传回，并在整个助手轮次中增量缓存，从而在多步骤工作流中节省令牌。
* **不影响智能：** 保留 thinking 块对模型性能没有负面影响。

代价是上下文使用量：在保留全部的模型上，长对话会消耗更多上下文空间，因为保留的 thinking 块像任何其他对话历史一样计为输入（请参阅[思考与上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-the-context-window)）。在两种机制下该行为都是自动的。不需要代码更改或测试版请求头，您应按照[保留 thinking 块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)中的说明继续传回完整、未经修改的 thinking 块。要在任一方向上覆盖默认行为，请使用 [thinking 块清除](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#thinking-block-clearing)。

**在对话中途切换模型。** 切换模型时请继续原样传回 thinking 块，例如在[分类器拒绝回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)之后。thinking 块只能由生成它的模型或更新的模型读取，API 会忽略或丢弃目标模型无法读取的块。在 Claude Fable 5.1 和 Claude Mythos 5.1 上，方向很重要：它们能读取所有更早模型的 thinking 块，而没有更早的模型能读取它们的块，因此向上切换到它们会保留对话的推理，而向下切换会丢失推理（有关确切列表以及丢弃的块如何计费和报告，请参阅[保留的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-for-model)）。仅在那些忽略而非丢弃块的模型上，为了节省输入令牌才自行剥离先前的 `thinking` 和 `redacted_thinking` 块，并且在兑换[回退额度](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)时绝不要这样做，因为这要求请求体保持不变。

## 保留的思考

Claude 仅在 thinking 块创建时的条件下保留该块，使其在后续轮次中可用。从 Claude Fable 5.1 和 Claude Mythos 5.1 开始，`thinking` 或 `redacted_thinking` 块仅在以下情况下被保留：

* **对于生成它的模型或更新的模型。** 更早的模型无法使用该块，API 会将其从该请求中丢弃。请参阅[仅对于生成它的模型或更新的模型](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-for-model)。
* **在生成它的对话中（仅限 Claude Fable 5.1）。** 如果 `system` 提示、`tools` 或任何更早的消息发生变化，该块将不再有效，API 会拒绝请求或丢弃该块。请参阅[仅在生成它的对话中](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)。

在两个模型上，块的 `signature` 都记录了这两个条件。每当该块在后续请求中返回时（包括发往不同模型的请求），API 都会检查它；Claude Mythos 5.1 仅检查模型条件。

**原样传回块。** 将每个助手轮次按您收到的样子原样发送，包括 thinking 块，让 API 决定模型可以使用哪些块。

### 仅对于生成它的模型或更新的模型

此条件是单向的：Claude Fable 5.1 和 Claude Mythos 5.1 能读取更早模型的 thinking 块，而没有更早的模型能读取它们的块。

* **迁移到 Claude Fable 5.1 或 Claude Mythos 5.1 的对话会保留其推理。** 更早模型的 thinking 块仍然可读，因此模型从切换后的第一个轮次起照常思考。
* **从它们迁移到任何更早模型的对话会丢失推理。** 更早的模型无法读取它们的块，API 会为该请求丢弃这些块，更早的模型会根据可见消息重新推理。如果对话之后以相同的历史返回到 Claude Fable 5.1，它自己的块将再次可读。

完整地说，Claude Fable 5.1 和 Claude Mythos 5.1 能读取彼此生成的 thinking 块，以及由 Claude Opus 5、Claude Fable 5 和 Claude Mythos 5 生成的块，还有由 Claude Opus 4.8 及更早的 Opus 模型、Claude Sonnet 模型和 Claude Haiku 4.5 生成的块。除这两个模型之外，没有任何模型能读取由 Claude Fable 5.1 或 Claude Mythos 5.1 生成的块。

**接收模型无法读取的块会被丢弃。** API 会在提示到达模型之前将其移除。它不计入 `input_tokens`，也不计费。当您在对话中途从 Claude Fable 5.1 回退到较旧的模型时（例如在[分类器拒绝回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)之后），较旧的模型会根据可见对话重新推理。使用[控制测试版请求头](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking-controls)时，丢弃会在 `input_transformations` 中以 `model_binding_mismatch` 报告。没有它时，丢弃是静默的。[服务器端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)以同样的方式丢弃不可读的块。

### 仅在生成它的对话中

来自 Claude Fable 5.1 的 thinking block（思考块）仅在生成它的对话前缀保持不变时才会被保留。它的 `signature` 涵盖了 `system` 提示、`tools` 以及该块之前的消息。Claude Mythos 5.1 会记录相同的 `signature`，但不会运行此检查。

此检查对 2026 年 8 月 31 日或之后创建的新账户强制执行。对于更早创建的账户，API 会在签名中记录该条件，但不会对不匹配采取行动，除非请求设置了 [`thinking.block_binding.prefix_mismatch_behavior`](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking-controls)，该设置会选择启用强制执行。Anthropic 计划在未来的模型上对每个组织强制执行此条件。如果您的账户创建较早，请现在就让您的应用程序兼容：同样的仅追加（append-only）模式可以让[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)保持热状态，并且您可以通过发送 `prefix_mismatch_behavior: "error"` 来针对该检查进行测试。如果您发布的是供他人使用自己的 API 密钥运行的工具或框架，请以这种方式测试：您在新账户上的用户会比您更早受到强制执行。[保留的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)提供了集成检查清单：如何判断您的代码是否编辑了历史记录，以及替代每种编辑的 API 功能。

在强制执行该检查的情况下，针对已更改的前缀重放某个块的请求会被拒绝，并返回 400 `invalid_request_error`：

```text wrap
messages.5.content.0: Invalid `signature` in `thinking` block. The block is bound to a different conversation. Remove the block, or set `thinking.block_binding.prefix_mismatch_behavior` to "drop_block". That setting requires the `thinking-binding-controls-2026-08-01` value in the `anthropic-beta` header.
```

最后一句仅在请求未发送 beta 头时出现。该消息可能以另一句话结尾，指出第一条发生更改的消息。重试相同的请求体会以同样的方式失败。若要改为在不使用已失效推理的情况下继续，请发送 `thinking-binding-controls-2026-08-01` beta 头并将 `prefix_mismatch_behavior` 设置为 `"drop_block"`。API 随后会丢弃失败的块以及对话中其后的每个思考块，并在 `input_transformations` 中将每一个报告为 `prefix_binding_mismatch`。[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点运行相同的检查并返回相同的 400。

会使后续思考块失效的情况：

* 编辑、重新排序或删除较早的消息，包括删除您注入到较早用户轮次中的每轮提醒。
* 在请求之间更改顶层 `system` 提示的内容，或在 `tools` 数组中添加、删除或编辑工具。
* 客户端压缩或截断，即逐字保留最近的助手轮次（包括思考），同时重写它们之前的轮次。
* 较早轮次中的图像或文档 URL 在后续请求中提供了不同的字节。该检查涵盖的是字节而非 URL 字符串，因此同一文件的轮换签名 URL 没有问题。对于您跨轮次引用的内容，请使用 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传一次并发送 `file_id`，或发送 base64。

不会使其失效的情况：

* 从最旧的开始，删除开头连续的一段思考块：对话中的第一个思考块（或最近一次压缩块之后的第一个），然后是下一个，依此类推。从其他任何位置删除思考块都会使其后的每个思考块失效，包括该轮次中的以及之后每个轮次中的。
* 在请求之间更改 `output_config.effort`、`max_tokens` 或其他采样设置。
* `cache_control` 标记，无论您将它们放置或移动到何处。
* 服务器端[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)和[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)：它们不算作编辑，因为该检查比较的是您发送的对话，而不是服务器编辑后的副本。压缩之后，被检查的前缀从压缩块开始。

保持思考块有效的模式：

* **仅追加。** 在 `messages` 末尾添加新消息，并保持较早的轮次逐字节不变。
* \*\*使用[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)\*\*和对话中途工具更改来在中途添加指令或更改工具可用性，而不是编辑顶层 `system` 字段或 `tools` 数组。对于只应适用于一个轮次的提醒，请将其作为[轮次范围的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)发送，并将其保留在历史记录中，而不是稍后删除。这也能保留提示缓存。
* **使用服务器端上下文管理**，而不是自己裁剪历史记录。
* \*\*如果请求因前缀不匹配而被拒绝且您无法修复历史记录，\*\*请使用 beta 头和 `prefix_mismatch_behavior: "drop_block"` 重新发送，或从历史记录中剥离每个 `thinking` 和 `redacted_thinking` 块并重试一次。

当较早的思考被丢弃时，模型会在没有这些块的情况下回答该轮次。反复使自身历史记录失效的客户端每次都会重新启动提示缓存，从而增加成本。

**客户端压缩。** 此检查并不排除在客户端进行压缩。规则更为狭窄：不要在您已重写的前缀之后保留思考块。服务器端[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)是满足该规则的最简单方式。如果您在客户端压缩，请使用以下形式之一：

* **简单压缩（推荐）：** 将对话总结为一条消息，并以该摘要加上新的用户轮次开始下一个请求，不重放任何较早的轮次和较早的思考块。没有较早的思考保留下来，因此不会有任何失败，模型会在压缩后的对话上重新思考。Claude 模型在长周期任务上使用此方案进行训练，对于大多数工作负载，其表现与更复杂的方案相当。与任何压缩一样，它会重置提示缓存。
* **保留尾部压缩：** 总结较旧的轮次并逐字保留最近的轮次。保留轮次的思考块是针对完整历史记录生成的，在摘要之后会失败。从您携带过来的每个轮次中剥离 `thinking` 和 `redacted_thinking`（它们的文本和工具调用可以保留），或设置 `prefix_mismatch_behavior: "drop_block"` 并让 API 丢弃它们。
* **后台压缩：** 在关键路径之外构建摘要，并在对话继续进行时将其换入。在此期间生成的每个轮次都有早于换入的思考。在每个仍携带换入前生成的思考块的请求上发送 `"drop_block"`（或自己剥离这些块；换入后第一个响应上的 `input_transformations` 会准确列出是哪些块），或者同步压缩。

从记录中间剪掉单个轮次会使其后的每个思考块失效，没有任何客户端形式可以避免这一点。对于您正在进行的指令更改，请使用对话中途系统消息；对于选择性删除，请使用服务器端[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)。

### 针对未保留块的控制（beta）

发送 [beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers) `thinking-binding-controls-2026-08-01` 可获得两样东西：每个响应上的 `input_transformations` 数组，列出 API 丢弃的任何思考块；以及思考配置上带有一个字段的 `block_binding` 对象。

| 字段                         | 类型                         | 默认值       | 描述                                                                                                                                                                                                                                         |
| -------------------------- | -------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `prefix_mismatch_behavior` | `"error"` 或 `"drop_block"` | `"error"` | API 对未通过[对话检查](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)的思考块所做的处理。`"error"` 以 400 错误拒绝请求。`"drop_block"` 删除该块以及对话中其后的每个思考块，在 `input_transformations` 中报告每一个，然后继续。两个值都不会改变模型检查，模型检查始终会丢弃。 |

`block_binding` 可与 `thinking.type: "adaptive"` 和 `thinking.type: "enabled"` 一起使用。在没有 beta 头的情况下发送它会返回 400 错误。不运行对话检查的模型会接受该对象并仅报告模型检查的丢弃，因此一个请求体可跨模型使用。在 Amazon Bedrock 和 Google Cloud 上，请按照 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers)中的说明传递 beta 名称。

以下请求选择丢弃而非拒绝。在第一个轮次上没有任何内容可重放，因此 `input_transformations` 返回为空：

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

  ```bash CLI
  ant beta:messages create --beta thinking-binding-controls-2026-08-01 \
    --transform '{content.#(type=="text")#.text,input_transformations}' \
    --format yaml <<'YAML'
  model: claude-fable-5-1
  max_tokens: 16000
  thinking:
    type: adaptive
    block_binding:
      prefix_mismatch_behavior: drop_block
  messages:
    - role: user
      content: What is the greatest common divisor of 1071 and 462?
  YAML
  ```

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

```text Output wrap
The greatest common divisor of 1071 and 462 is 21.
Input transformations: 0
```

**丢弃的块在 `input_transformations` 中报告。** 在 beta 头下，来自支持思考的模型的每个响应都携带此顶层数组。当没有任何内容被丢弃时它为空，且永远不会是 `null`。每个条目指出被丢弃块的位置及其未通过的检查：

```json
{
  "input_transformations": [
    {
      "type": "thinking_dropped",
      "path": "messages.1.content.0",
      "reason": "model_binding_mismatch"
    }
  ]
}
```

`reason` 字段为 `model_binding_mismatch` 或 `prefix_binding_mismatch`。请忽略您不认识其 `type` 或 `reason` 的条目，因为后续检查会添加新值。在[流式传输](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)时，`input_transformations` 随 `message_start` 事件中的 `message` 对象到达。在流中途发生服务器端回退之后，最终的 `message_delta` 事件会再次携带该数组，其中包含实际提供服务的模型的条目。没有 beta 头时该字段不存在。

被篡改或无法解密的签名是另一种失败：它始终返回 400（``Invalid `signature` in `thinking` block``，不带原因子句），且 `prefix_mismatch_behavior` 不适用于它。在[消息批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)中，其块在 `"error"` 下未通过对话检查的项目会解析为 `errored`。

## 思考与提示缓存

[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)与思考以几种特定方式相互作用。以下规则适用于两种思考模式。

**配置更改会使缓存失效。** 思考配置和解析后的 [`effort`](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别会被渲染到提示本身中，因此更改其中任何一项都会开始一个新的缓存前缀。在 `adaptive`、`enabled` 和 `disabled` 之间切换、更改 `budget_tokens` 以及更改 effort 值都会使缓存断点失效：消息级断点始终未命中，工具和系统提示断点也可能未命中，具体取决于模型在何处渲染配置。请将任何思考或顶层 effort 更改视为重新开始缓存。在支持[每消息 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta) 的模型上，在 `messages` 内的 `role: "system"` 消息中携带的 effort 更改会使缓存前缀保持完整。保持相同配置的连续请求会保留缓存，并且将参数显式设置为其默认值等同于省略它。API 在任一[保留思考条件](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking)下丢弃的思考块会从该块的位置起更改缓存前缀。原样传回的块会保持缓存完整。带有用量输出的完整演示位于[引导思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#prompt-caching)页面。

**思考块与工具结果一起缓存。** 在工具使用循环期间，当您发出包含工具结果的后续请求时会发生缓存。此时，之前的对话历史记录（包括其思考块）可以被缓存，并且这些缓存的思考块在从缓存读取时会在您的用量指标中计为输入令牌。即使没有显式的 `cache_control` 标记，这也会自动发生，并且对于常规思考和交错思考的行为相同。权衡之处在于：您在响应中再也看不到的思考块在从缓存读取时仍会计入输入令牌用量。

**先前的块是否在上下文中取决于模型。** [保留默认值](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)决定了这一点。在保留全部的模型上，先前轮次的思考块保持缓存并在上下文中。在仅保留最后一轮的模型上，一旦您发送了不是工具结果的用户消息，所有先前的思考块都会从上下文中剥离。在这些模型上，像这样的对话：

```text wrap
User: ["What's the weather in Paris?"],
Assistant: [thinking_block_1] + [tool_use block 1],
User: [tool_result_1, cache=True],
Assistant: [thinking_block_2] + [text block 2],
User: [Text response, cache=True]
```

会被当作思考块从未存在过一样处理：

```text wrap
User: ["What's the weather in Paris?"],
Assistant: [tool_use block 1],
User: [tool_result_1, cache=True],
Assistant: [text block 2],
User: [Text response, cache=True]
```

在保留全部的模型上，相同的请求会将 `thinking_block_1` 和 `thinking_block_2` 保留在上下文和缓存中。

**降级会从可缓存的历史记录中剥离思考。** 如果思考在轮次中途被禁用，而您在当前工具使用轮次中传递了思考内容，则思考内容会被剥离，并且该请求的思考保持禁用状态（请参阅[优雅降级](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)）。[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)会放大缓存失效的影响，因为思考块可能出现在多个工具调用之间。

<Tip>
  思考密集型任务通常需要比默认的 5 分钟缓存生命周期更长的时间才能完成。请考虑使用 [1 小时缓存时长](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#1-hour-cache-duration)，以在较长的思考会话和多步骤工作流中保持缓存命中。
</Tip>

## 思考与上下文窗口

`max_tokens`（包括 Claude 在当前轮次中生成的所有思考）作为严格限制强制执行。在 Claude 4.5 及更新的模型上，如果输入令牌加上 `max_tokens` 超过上下文窗口大小，API 会接受该请求。如果生成随后达到上下文窗口限制，它会以 `stop_reason: "model_context_window_exceeded"` 停止，而不是返回错误。在更早的模型上，API 会改为返回验证错误。请参阅[处理停止原因](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons)。

思考如何计入窗口取决于它的生成时间：

* **当前轮次的思考**始终计入 `max_tokens`，按输出令牌计费，并占用生成它的轮次的上下文窗口空间。
* **先前轮次的思考**取决于[保留默认值](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)。在[保留所有先前轮次的模型](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)上，先前的思考块保留在上下文中，计入窗口，并像对话历史记录的其余部分一样按输入令牌计费。在仅保留最后一轮的模型上，当您传回较旧的思考块时，API 会自动剥离它们，因此它们不会消耗窗口空间或输入令牌。

在实践中：

* 在保留全部的模型上，请将思考当作普通对话历史记录来规划您的上下文窗口预算，因为它确实如此。长时间的智能体会话会在上下文中累积思考。如果您需要回收空间，请使用[思考块清除](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#thinking-block-clearing)。
* 在仅保留最后一轮的模型上，思考只是每轮次的成本：每个轮次的思考计入该轮次的 `max_tokens`，然后从窗口中移除。

以下图表说明了仅保留最后一轮（剥离）的机制。第一张图显示了多轮对话：每个轮次的思考块在输出中生成，但不会带入后续轮次的输入。

![在剥离先前思考块的模型上的思考示意图：每个轮次的 thinking block（思考块）在输出中生成，不会带入后续轮次的输入](https://platform.claude.com/docs/images/context-window-thinking.svg)

第二张图显示了带有工具使用的相同机制：思考在助手轮次期间与其工具结果一起保留在上下文中，然后在下一个用户轮次时移除。

![在剥离先前思考块的模型上带有 tool use（工具使用）的思考示意图：思考与其 tool result（工具结果）一起保留，然后在下一个用户轮次时丢弃](https://platform.claude.com/docs/images/context-window-thinking-tools.svg)

使用[令牌计数 API](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting) 获取您特定用例的准确计数，尤其是对于包含思考的多轮对话。

## 思考加密

完整的思考内容经过加密并在每个思考块的 `signature` 字段中返回。当您传回思考块时，API 使用签名来验证思考块是由 Claude 生成的。

使用签名时请记住以下几点：

* 只有在[将工具与思考一起使用](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)时才严格需要传回思考块。否则您可以省略先前轮次的思考块。如果您确实传回它们，API 是保留还是剥离它们取决于模型（请参阅[按模型的思考块保留](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)）。使用[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)来配置此行为。
* 传回思考块时，请完全按照您收到的样子传回所有内容，以保持一致性并避免潜在问题。
* 在[流式传输响应](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#streaming-thinking)时，签名作为 `content_block_delta` 事件内的 `signature_delta` 到达，紧接在 `content_block_stop` 事件之前。
* Claude 4 及更高版本模型中的 `signature` 值比之前的模型长得多。
* `signature` 字段是不透明的：不要解释或解析它。
* `signature` 值跨平台兼容（Claude API、[Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 和 [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)）。在一个平台上生成的值可在另一个平台上使用。

## 已编辑的思考块

除了常规的 `thinking` 块之外，当 Claude 的部分推理出于安全原因被编辑时，API 可能会返回 `redacted_thinking` 块。`redacted_thinking` 块在 `data` 字段中包含加密的思考内容，没有可读文本：

```json
{
  "type": "redacted_thinking",
  "data": "..."
}
```

`data` 字段是不透明且加密的。与常规思考块上的 `signature` 字段一样，在使用[工具](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)继续多轮对话时，请将 `redacted_thinking` 块原样传回 API。

<Tip>
  如果您的代码在往返带有工具使用的响应时按类型过滤内容块（例如 `block.type == "thinking"`），请同时包含 `redacted_thinking` 块。仅按 `block.type == "thinking"` 过滤会静默丢弃 `redacted_thinking` 块，并破坏[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)中描述的多轮协议。
</Tip>

<Note>
  `redacted_thinking` 块是在思考出于安全原因被编辑时返回的一种独立的内容块类型。这与 [`display: "omitted"`](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display) 选项不同，后者返回 `thinking` 字段为空的常规 `thinking` 块。
</Note>

## 限制与功能兼容性

### 采样参数

在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Mythos Preview、Claude Opus 5、Claude Opus 4.8、Claude Opus 4.7 和 Claude Sonnet 5 上，非默认的 `temperature`、`top_p` 或 `top_k` 值在每个请求上都会返回 400 错误，无论是否使用思考。在较旧的模型上，该限制仅在思考开启时适用：`temperature` 和 `top_k` 与思考不兼容，而 `top_p` 允许取 0.95 到 1 之间的值。

### 响应预填充与强制工具使用

思考开启时您无法预填充助手响应。强制工具使用（`tool_choice: {"type": "any"}` 或 `{"type": "tool", ...}`）与手动扩展思考不兼容，但可与自适应思考一起使用。例外是 Claude Fable 5.1 和 Claude Mythos 5.1，它们在每个请求上都以 400 错误拒绝强制工具使用。在这些模型上，请改用 `tool_choice: {"type": "auto"}` 搭配[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)或[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)。请参阅[思考与工具使用](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)。

### 输出限制

每个模型接受的 `max_tokens` 最高为此处列出的上限。在 [Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#extended-output-beta) 上，`output-300k-2026-03-24` [beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers)会为列出了批处理上限的模型提高该上限。

| 模型                    | 最大输出令牌数 | 批处理 beta 上限 |
| --------------------- | ------- | ----------- |
| Claude Fable 5.1      | 128k    | —           |
| Claude Mythos 5.1     | 128k    | —           |
| Claude Fable 5        | 128k    | —           |
| Claude Mythos 5       | 128k    | —           |
| Claude Mythos Preview | 128k    | 不可用         |
| Claude Opus 5         | 128k    | 300k        |
| Claude Opus 4.8       | 128k    | 300k        |
| Claude Opus 4.7       | 128k    | 300k        |
| Claude Sonnet 5       | 128k    | 300k        |
| Claude Opus 4.6       | 128k    | 300k        |
| Claude Sonnet 4.6     | 128k    | 300k        |
| Claude Haiku 4.5      | 64k     | 不可用         |
| Claude Sonnet 4.5     | 64k     | 不可用         |
| Claude Opus 4.5       | 64k     | 不可用         |

有关旧版模型的限制，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。

### 长请求

当 `max_tokens` 大于 21,333 时，SDK 要求使用流式传输，以避免长时间运行的请求出现 HTTP 超时。这是客户端验证，而非 API 限制。如果您不需要增量处理事件，请使用 `.stream()` 搭配 `.get_final_message()`（Python）或 `.finalMessage()`（TypeScript）来获取完整的 `Message` 对象，而无需处理单个事件。请参阅[流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming#get-the-final-message-without-handling-events)。思考处于活动状态时，预计响应时间会更长，因为生成思考块会增加处理时间。对于将每个请求的思考推高到大约 32k 令牌以上的工作负载，请使用[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)以避免网络问题：此类请求的运行时间可能长到足以触发系统超时和打开连接数限制。

## 后续步骤

<CardGroup cols={2}>
  <Card title="引导思考" icon="compass" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost">
    通过 effort 级别、系统提示指导和每消息引导来控制 Claude 思考的频率和深度，并了解思考的成本和定价。
  </Card>

  <Card title="工具和多轮工作流中的思考" icon="wrench" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-tool-workflows">
    逐步完成一个正确保留思考块的完整两轮工具使用往返，并了解交错思考如何改变流程。
  </Card>

  <Card title="保留的思考" icon="stack" href="https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking">
    了解您的 Messages API 集成是否编辑了对话历史记录，并用保持较早思考块有效的 API 功能替换每种编辑。
  </Card>

  <Card title="思考故障排除" icon="hammer" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting">
    诊断并修复最常见的思考故障：配置 400 错误、空的或缺失的思考块、max\_tokens 停止以及缓存未命中。
  </Card>

  <Card title="Effort" icon="sliders" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    使用 effort 参数控制 Claude 响应时使用的令牌数量，在响应的全面性和令牌效率之间进行权衡。
  </Card>
</CardGroup>
