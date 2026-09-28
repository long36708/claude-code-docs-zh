---
title: 保留思考
url: https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking
description: 保留思考让模型仅在某个早期轮次的思考块由该模型或更早的模型生成、且该块之前的内容均未更改时，才使用该思考块。
---

"Preserved thinking"（保留思考）是较新 Claude 模型的一项特性，用于防范 "distillation"（蒸馏）。它决定模型能否使用您从早期轮次发回的 "thinking block"（思考块）。从 Claude Fable 5.1 开始，当请求中传回 `thinking` 或 `redacted_thinking` 块时，API 会检查该块的 `signature` 中的两项内容：

* **模型能够读取该块。** 每个模型都能读取自己的思考块，以及一组固定的其他模型的思考块。Claude Fable 5.1 可以读取来自 Claude Opus 5 的块，在 Claude API 上还可以读取来自 Claude Opus 5.5 的块；Claude Opus 5 和 Claude Opus 5.5 都无法读取来自 Claude Fable 5.1 的块。如果当前模型无法读取某个块，API 会从该请求中丢弃它，且不会报错。请参阅[在对话中途切换模型](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#switching-models)。
* **思考块之前的内容均未更改。** 块之前的顶层 `system` 提示、`tools` 和 `messages` 构成它的 "prefix"（前缀）。如果前缀与生成该块时您发送的内容不同，则该块及其后的所有思考块都将失效，API 会根据您的选择，以 400 错误拒绝请求或丢弃失效的块。请参阅[保持前缀不变](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#prefix-check)。

模型检查适用于所有账户。对于在 2026 年 8 月 31 日 00:00 UTC 或之后创建的账户，API 默认强制执行前缀检查。对于较早的账户，API 仅对设置了 `thinking.block_binding.prefix_mismatch_behavior` 的请求强制执行前缀检查。**无论您的账户创建于何时，都请让您的集成采用 "append-only"（仅追加）方式**，这样同一套代码可以在所有账户上运行，包括默认强制执行检查的较新账户。

## 谁需要做出更改

如果您的请求由 Claude Code、claude.ai、Claude Managed Agents 或 Claude Agent SDK 构建，或者您的代码在会话期间保持 `system` 和 `tools` 不变且只向 `messages` 追加内容，则您无需做任何更改。Claude Mythos 5.1 以及 Claude Fable 5.1 之前的模型不运行前缀检查。如果您从不发回思考块，前缀检查就没有可拒绝的内容，模型也无法获得其早期的推理。

如果您的集成在同一对话的两次请求之间执行以下任一操作，请检查您的集成。每一项都链接到相应的替代做法：

* [重建 `system` 提示](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#new-instructions)：日期、模式标志、重新读取的项目说明，或在第一轮之后才连接的插件或 MCP 服务器
* [重新渲染第一条用户消息中的上下文](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#changing-context)
* [清除或缩短旧的工具结果，或重新编码旧图像](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#server-side-trimming)
* [在客户端汇总或丢弃旧轮次](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#custom-compaction-on-the-client)，同时保留近期轮次及其思考
* [在 `tools` 中添加、删除或编辑条目](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#tool-changes)
* [向用户轮次添加提醒](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#per-turn-reminders)，之后又删除或改写它
* [丢弃部分 `thinking` 块而保留后面的块](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#append-assistant-turns-exactly-as-returned)，或删除它们后[又放回去](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#prefix-check)
* [从模板重建已保存的会话](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#faq)，而不是重放已发送的内容

在较早的账户上，除非请求设置了 `prefix_mismatch_behavior`，否则上述操作都不会产生错误，因此使用您自己的密钥运行时没有出错，并不能说明您的代码是否受影响。如果有人使用自己的 API 密钥运行您的工具，那么使用较新账户的用户会比您先遇到 400 错误。要在不改变请求行为的情况下看到他们所看到的情况，请发送 `thinking-binding-controls-2026-08-01` "beta header"（beta 标头）。在较早的账户上，每个响应随后都会标记未通过检查的块，而模型仍会读取它们（请参阅[设置不匹配行为并读取 `input_transformations`](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#preserved-thinking-controls)）。

## 在对话中途切换模型

Claude Fable 5.1 和 Claude Mythos 5.1 可以读取彼此以及更早 Claude 模型生成的思考块。更早的模型都无法读取 Claude Fable 5.1 或 Claude Mythos 5.1 的思考块。

Claude Opus 5.5 可以读取来自 Claude Opus 5 以及更早的 Opus、Sonnet 和 Haiku 模型的思考块，但不能读取来自 Claude Fable 或 Claude Mythos 模型的思考块。在 Claude API 上，Claude Fable 5.1 和 Claude Mythos 5.1 可以读取来自 Claude Opus 5.5 的思考块；其他模型都不能。因此，从 Claude Opus 5 切换到 Claude Opus 5.5 的对话会保留其推理，在 Claude API 上从 Claude Opus 5.5 升级到 Claude Fable 5.1 或 Claude Mythos 5.1 的对话也是如此。而从 Claude Fable 5.1 或 Claude Mythos 5.1 切换到 Claude Opus 5.5，或从 Claude Opus 5.5 切换到这两个模型以外的任何模型的对话，在切换后的轮次中将不带有前一个模型的推理。这些块会被丢弃，而不是被拒绝，如下所述。

* **从更早的模型切换到 Claude Fable 5.1 的对话，或在 Claude API 上从 Claude Opus 5.5 切换过来的对话，会保留其推理。** 更早模型的思考块仍然可读，因此模型从切换后的第一轮起就会照常思考。
* **降级到更早模型的对话会在该请求中失去 Claude Fable 5.1 的推理。** 当路由器将某一轮发送到更便宜的模型时、发生[分类器拒绝回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)之后，或在[服务器端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)期间，就会出现这种情况。API 会在提示到达模型之前删除不可读的块。这些块不计费，也不计入 `input_tokens`。

每次请求都继续发送完整的历史记录（包括思考块），让 API 丢弃当前模型无法读取的内容。API 从不编辑您的 `messages` 数组，因此被丢弃的块仍保留在您的历史记录中。当同一历史记录再次发送给 Claude Fable 5.1 时，其块以及早期模型的思考都会再次可读。只有当您的客户端自行删除这些块时，推理才会永久丢失，例如在切换模型时剥离思考的 "harness"（运行框架），或根据每个模型所用内容重建历史记录的运行框架。

![动画：切换到 Claude Opus 时，该轮会跳过 Claude Fable 5.1 的思考；切换回来后，所有内容都会再次被读取](https://platform.claude.com/docs/images/preserved-thinking-model-switch.svg)

使用 `thinking-binding-controls-2026-08-01` [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)时，响应会在顶层 `input_transformations` 数组中列出每个被丢弃的块，并带有 `reason: "model_binding_mismatch"`：

```json
{
  "input_transformations": [
    {
      "type": "thinking_dropped",
      "path": "messages.3.content.0",
      "reason": "model_binding_mismatch"
    }
  ]
}
```

如果不使用该标头，丢弃会静默进行。此条目并不表示您的集成存在错误，`prefix_mismatch_behavior` 对其也没有影响：当前模型无法读取的块总是会被丢弃。

## 保持前缀不变

在 Claude Fable 5.1 和 Claude Opus 5.5 上，只有当您在思考块之前发送的所有内容在后续请求中保持不变时，该思考块才保持有效。被检查的前缀包含三个部分：

* 顶层 `system` 提示
* `tools` 集合
* 该块之前的每条 `message`

注意：使用服务器端 "[compaction](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)"（压缩）时，被检查的前缀从最近的压缩块开始。

这三个字段之外的请求参数，例如 `effort`、`max_tokens`、`output_config`、`tool_choice` 和 `metadata`，不属于前缀检查范围，`cache_control` 标记也不属于。[哪些算作编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#what-counts-as-an-edit)列出了完整清单。

较早的思考块不在前缀中，但每个思考块都会跨轮次记录在它之前的是哪个思考块。您可以从历史记录的开头（最旧的优先）或末尾删除思考块，也可以全部删除。会导致失败的是中间出现空缺：您保留的思考块必须是原始序列中连续不断的一段，因此从中间删除一个块会使其后的思考块失效。一旦删除某个块，就不要再放回。放回会使该块缺失期间生成的思考块失效。

在会话期间保持 `system` 和 `tools` 不变，并将 `messages` 视为仅追加。同样的做法也能为[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)保持前缀稳定：使思考失效的编辑，正是使缓存重新开始的编辑。

### API 如何处理失效的块

您可以通过 `thinking.block_binding.prefix_mismatch_behavior` 进行选择：

* **`"error"`（默认）：** API 以 400 `invalid_request_error` 拒绝请求，并指出第一个失败的块。
* **`"drop_block"`：** API 丢弃每个失败的块及其后的所有思考块，请求成功。被丢弃的块不计费。模型在该轮回答时不使用被丢弃块中的推理，提示缓存从编辑处重新开始。响应会在 `input_transformations` 中（流式传输时位于 `message_start` 事件上）列出每个被丢弃的块，并带有 `reason: "prefix_binding_mismatch"`。

<Warning>
  `"drop_block"` 会隐藏错误，但不会修复导致错误的编辑。被丢弃的块不计费，但会话的令牌用量仍可能增加，因为 Claude 有时会进行更多思考来重新生成被丢弃的思考。被丢弃的思考块越多，或在长会话中发生丢弃的轮次越多，增幅往往越大。
</Warning>

请统计每个会话中 `input_transformations` 含有 `prefix_binding_mismatch` 条目的响应数量，对其发出告警，并将每处编辑替换为[在不编辑前缀的情况下进行更改](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#replace-prefix-edits)中对应的模式。在 Message Batches API 中，未设置该字段的条目不会失败。在 API 默认强制执行检查的情况下，它会改为丢弃未通过检查的块。如果您希望批处理条目失败，请在其中显式设置 `"error"`。

该字段和 `input_transformations` 数组都需要 `thinking-binding-controls-2026-08-01` [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)。[设置不匹配行为并读取 `input_transformations`](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#preserved-thinking-controls) 展示了各 SDK 中的请求。

400 消息的开头为：

```text wrap
messages.1.content.0: Invalid `signature` in `thinking` block. The block is bound to a different conversation. Remove the block, or set `thinking.block_binding.prefix_mismatch_behavior` to "drop_block".
```

如果请求未发送 beta 标头，消息会继续：

```text wrap
That setting requires the `thinking-binding-controls-2026-08-01` value in the `anthropic-beta` header.
```

消息通常以一句说明更改内容的话结尾，例如 `system` 提示或 `tools` 列表与创建该块时不同。[思考故障排除](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#error-thinking-block-signature)描述了这句话可能指出的内容。

签名被篡改或无法解密是另一种失败。它总是返回 400（``Invalid `signature` in `thinking` block``，且没有关于对话的说明），`prefix_mismatch_behavior` 不适用于这种情况。

#### 在代码中处理错误

这就是本节前面展示的 400 `invalid_request_error`。不要重新发送相同的请求体：它每次都会以同样的方式失败。使用 beta 标头和 `prefix_mismatch_behavior: "drop_block"` 重试一次，并将该选择与会话一起存储，以便之后的每个请求（包括重启之后）也都发送它。如果您无法发送 beta 标头，请从历史记录中一次性删除所有 `thinking` 和 `redacted_thinking` 块，之后不再放回，然后继续。接着修复导致不匹配的编辑。

### 设置不匹配行为并读取 `input_transformations`

`thinking-binding-controls-2026-08-01` [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)会添加：

* 每个响应上的顶层 `input_transformations` 数组
* `thinking` 配置上的 `block_binding` 对象，其唯一字段是 `prefix_mismatch_behavior`

`block_binding` 可与 `thinking.type: "adaptive"` 和 `thinking.type: "enabled"` 一起使用。在不带 beta 标头的情况下发送它会返回 400 错误，其消息以 `block_binding: Extra inputs are not permitted` 结尾。不运行前缀检查的模型会接受该对象，并且只报告模型检查导致的丢弃，因此同一个请求体可以跨模型使用。API 参考将前缀检查称为对话检查（conversation check）。

以下请求选择丢弃而非拒绝。在第一轮中没有需要重放的内容，因此 `input_transformations` 返回为空：

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

在 beta 标头下，来自支持思考的模型的每个响应都带有 `input_transformations`。每个条目通过其 `path`（例如 `messages.1.content.0`）指明一个思考块，并给出 `reason`。条目有两种类型：

* **`thinking_dropped`：** API 在模型读取该块之前将其丢弃，该块不计费。`reason` 为 `prefix_binding_mismatch` 或 `model_binding_mismatch`（请参阅[在对话中途切换模型](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#switching-models)）。
* **`thinking_mismatch_allowed`：** 该块未通过前缀检查，但 API 未对此请求强制执行该检查，因此该块原样到达模型并计费。`reason` 始终为 `prefix_binding_mismatch`。此条目仅出现在 API 默认不强制执行检查的请求上，例如来自较早账户的请求（请参阅[API 何时强制执行检查](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#enforcement)）。将 `prefix_mismatch_behavior` 设置为任一值都会让请求选择加入强制执行检查，因此设置了该字段的请求永远不会收到此条目。

当没有块被丢弃且没有块未通过前缀检查时，该数组为空。请忽略 `type` 或 `reason` 无法识别的条目，因为后续的检查会添加新值。

[流式传输](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)时，该数组位于 `message_start` 事件中的 `message` 对象上。在流式传输中途发生服务器端回退后，最终的 `message_delta` 事件会再次携带该数组，其中包含实际提供服务的模型的条目。在[消息批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)中，如果在显式设置 `"error"` 的情况下某个条目的块未通过前缀检查，该条目会解析为 `errored`。未设置该字段的条目不会失败。在 API 默认强制执行检查的情况下，它会改为丢弃未通过检查的块。[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点会执行相同的前缀检查并返回相同的 400。

### API 何时强制执行检查

API 对新账户在 Claude Fable 5.1 和 Claude Opus 5.5 上强制执行前缀检查。

* **在 2026 年 8 月 31 日 00:00 UTC 或之后创建的账户：** API 会检查 Claude Fable 5.1 和 Claude Opus 5.5 请求，并应用 `"error"`，除非您设置了 `"drop_block"`。新账户的定义同样适用于 Claude API 和云平台。
* **较早的账户：** API 仅对设置了 `prefix_mismatch_behavior` 的请求强制执行检查。设置该字段即可让请求选择加入，因此您无需创建新账户即可看到新账户所看到的情况。对于未设置该字段的请求，API 仍会运行检查，但会让未通过检查的块传递给模型。使用 beta 标头时，响应会在 `input_transformations` 中将每个此类块列为 `thinking_mismatch_allowed`，因此您可以在不改变模型接收内容的情况下找到前缀编辑。

要确定您的账户属于哪一组，请取一个包含思考块的 Claude Fable 5.1 对话，更改该块之前的某些内容，然后在不带 beta 标头或 `block_binding` 字段的情况下将其发送给 Claude Fable 5.1。如果返回提及该标头的 400 响应，则表示您的账户默认强制执行检查。如果返回 200 响应，则表示不强制执行。要进行确认，请带上 beta 标头再次发送相同的请求（仍不带 `block_binding`）：响应会在 `input_transformations` 中将您编辑之后的每个思考块列为 `thinking_mismatch_allowed`。

### 哪些算作编辑

每一行比较两个连续的请求：

| 请求之间的更改                                                                                                                         | 后续思考块                                                                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 在末尾追加消息                                                                                                                         | 有效                                                                                                                                                                                                                                                       |
| 添加一个带有 `defer_loading: true` 且尚未被任何内容引用的工具                                                                                      | 有效                                                                                                                                                                                                                                                       |
| 从历史记录的开头或末尾删除 `thinking` 块，或全部删除                                                                                                | 有效（模型会失去该推理）                                                                                                                                                                                                                                             |
| 更改 `system`、`tools` 和 `messages` 之外的任何请求参数（`effort`、`max_tokens`、`output_config`、`tool_choice`、`metadata`、`thinking.display` 等） | 有效                                                                                                                                                                                                                                                       |
| 添加、移动或删除 `cache_control` 标记                                                                                                     | 有效                                                                                                                                                                                                                                                       |
| 返回相同字节的轮换签名 URL                                                                                                                 | 有效                                                                                                                                                                                                                                                       |
| 服务器端压缩或上下文编辑删除或替换内容                                                                                                             | 有效（检查比较的是您发送的内容，而不是服务器编辑后的副本）                                                                                                                                                                                                                            |
| 保留在原位的已清除[轮次范围系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#per-turn-reminders)             | 有效                                                                                                                                                                                                                                                       |
| 编辑、重新排序或删除任何更早的 `user`、`assistant` 或 `system` 消息                                                                                | 无效，除非在[保留思考的有效条件](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)下，由[按需压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand)生成的签名块替换其所汇总的消息 |
| 使用更改后的值重新渲染您放在第一条用户消息中的上下文                                                                                                      | 所有思考块均无效                                                                                                                                                                                                                                                 |
| 清除或缩短更早的 `tool_result`、重新编码更早的图像，或更改更早的 `tool_use` 输入                                                                           | 之后的所有思考块均无效                                                                                                                                                                                                                                              |
| 向更早的用户轮次添加文本块，或删除您上次添加的文本块                                                                                                      | 无效                                                                                                                                                                                                                                                       |
| 更改顶层 `system` 字符串或块                                                                                                             | 无效                                                                                                                                                                                                                                                       |
| 在 `tools` 中添加、删除、重命名或编辑工具                                                                                                       | 无效                                                                                                                                                                                                                                                       |
| 从历史记录中间删除 `thinking` 块并保留后面的块                                                                                                   | 之后的所有思考块均无效                                                                                                                                                                                                                                              |
| 放回您在更早请求中删除的 `thinking` 块                                                                                                       | 在它缺失期间生成的思考块无效                                                                                                                                                                                                                                           |
| 在下一次请求时返回不同字节的图像或文档 URL                                                                                                         | 无效                                                                                                                                                                                                                                                       |
| 同一条轮次范围消息在后续请求中被删除或改写                                                                                                           | 无效                                                                                                                                                                                                                                                       |

### 检查您的代码是否编辑了前缀

首先，对比您发送的内容。捕获您的集成在几个正常轮次中发送的请求体，其中包括一次压缩或一次工具更改。对于每对连续请求，比较 `system`、`tools` 以及它们共有的 `messages`。除新追加的轮次外，它们应完全相同。

然后通过 API 确认。添加 `thinking-binding-controls-2026-08-01` beta 标头，将 `prefix_mismatch_behavior` 设置为 `"drop_block"`，并在 claude-fable-5-1上通过您的集成运行一个正常的多轮会话。以下示例按照您的集成应有的方式运行两轮：`messages` 只增不减，每个助手轮次都完全按照 API 返回的样子发回（包括 `thinking` 块），并且每个请求都设置了 `block_binding`。每轮结束后，它会打印响应中 `thinking` 块的数量以及被丢弃块的数量：

<CodeGroup>
  ```bash cURL
  # 统计响应中的 thinking 块数量，以及被 API 丢弃的块数量
  COUNTS='"thinking blocks: \([.content[] | select(.type == "thinking")] | length), " +
    "dropped: \(.input_transformations | length)"'

  FIRST=$(curl -s https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: thinking-binding-controls-2026-08-01" \
    -d '{
      "model": "claude-fable-5-1",
      "max_tokens": 16000,
      "thinking": {
        "type": "adaptive",
        "block_binding": { "prefix_mismatch_behavior": "drop_block" }
      },
      "messages": [
        {
          "role": "user",
          "content": "How many positive integers below 500 have exactly 6 positive divisors?"
        }
      ]
    }')
  echo "$FIRST" | jq -r "$COUNTS"

  # 第 2 轮：assistant 轮次按返回时的原样传回，随后附上下一条用户消息
  MESSAGES=$(jq -n --argjson first "$FIRST" '[
    {
      role: "user",
      content: "How many positive integers below 500 have exactly 6 positive divisors?"
    },
    { role: "assistant", content: $first.content },
    { role: "user", content: "How many of those are odd?" }
  ]')

  jq -n --argjson messages "$MESSAGES" '{
    model: "claude-fable-5-1",
    max_tokens: 16000,
    thinking: {
      type: "adaptive",
      block_binding: { prefix_mismatch_behavior: "drop_block" }
    },
    messages: $messages
  }' | curl -s https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: thinking-binding-controls-2026-08-01" \
    -d @- | jq -r "$COUNTS"
  ```

  ```bash CLI
  # 统计响应中的 thinking 块数量，以及被 API 丢弃的块数量
  COUNTS='"thinking blocks: \([.content[] | select(.type == "thinking")] | length), " +
    "dropped: \(.input_transformations | length)"'

  FIRST=$(ant beta:messages create --beta thinking-binding-controls-2026-08-01 \
    --format json <<'YAML'
  model: claude-fable-5-1
  max_tokens: 16000
  thinking:
    type: adaptive
    block_binding:
      prefix_mismatch_behavior: drop_block
  messages:
    - role: user
      content: How many positive integers below 500 have exactly 6 positive divisors?
  YAML
  )
  echo "$FIRST" | jq -r "$COUNTS"

  # 第 2 轮：assistant 轮次按返回时的原样传回，随后附上下一条用户消息
  ant beta:messages create --beta thinking-binding-controls-2026-08-01 \
    --format json <<YAML | jq -r "$COUNTS"
  model: claude-fable-5-1
  max_tokens: 16000
  thinking:
    type: adaptive
    block_binding:
      prefix_mismatch_behavior: drop_block
  messages:
    - role: user
      content: How many positive integers below 500 have exactly 6 positive divisors?
    - role: assistant
      content: $(echo "$FIRST" | jq -c .content)
    - role: user
      content: How many of those are odd?
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  user_turns = [
      "How many positive integers below 500 have exactly 6 positive divisors?",
      "How many of those are odd?",
  ]

  # messages 随轮次增长：每个 assistant 轮次均按返回时的原样传回
  messages = []
  for user_turn in user_turns:
      messages.append({"role": "user", "content": user_turn})
      response = client.beta.messages.create(
          model="claude-fable-5-1",
          max_tokens=16000,
          thinking={
              "type": "adaptive",
              "block_binding": {"prefix_mismatch_behavior": "drop_block"},
          },
          messages=messages,
          betas=["thinking-binding-controls-2026-08-01"],
      )
      messages.append({"role": "assistant", "content": response.content})
      thinking_blocks = sum(block.type == "thinking" for block in response.content)
      dropped = len(response.input_transformations or [])
      print(f"thinking blocks: {thinking_blocks}, dropped: {dropped}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const userTurns = [
    "How many positive integers below 500 have exactly 6 positive divisors?",
    "How many of those are odd?"
  ];

  // messages 随轮次增长：每个 assistant 轮次均按返回时的原样回传
  const messages: Anthropic.Beta.BetaMessageParam[] = [];
  for (const userTurn of userTurns) {
    messages.push({ role: "user", content: userTurn });
    const response = await client.beta.messages.create({
      model: "claude-fable-5-1",
      max_tokens: 16000,
      thinking: {
        type: "adaptive",
        block_binding: { prefix_mismatch_behavior: "drop_block" }
      },
      messages,
      betas: ["thinking-binding-controls-2026-08-01"]
    });
    messages.push({ role: "assistant", content: response.content });
    const thinkingBlocks = response.content.filter((block) => block.type === "thinking");
    const dropped = response.input_transformations ?? [];
    console.log(`thinking blocks: ${thinkingBlocks.length}, dropped: ${dropped.length}`);
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  string[] userTurns =
  [
      "How many positive integers below 500 have exactly 6 positive divisors?",
      "How many of those are odd?",
  ];

  // messages 随轮次增长：每个 assistant 轮次均按返回时的原样回传
  List<BetaMessageParam> messages = [];
  foreach (var userTurn in userTurns)
  {
      messages.Add(new() { Role = Role.User, Content = userTurn });
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
              Messages = messages,
              Betas = [AnthropicBeta.ThinkingBindingControls2026_08_01],
          }
      );
      messages.Add(new()
      {
          Role = Role.Assistant,
          Content = response.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
      });
      var thinkingBlocks = response.Content.Count(block => block.TryPickThinking(out _));
      var dropped = response.InputTransformations?.Count ?? 0;
      Console.WriteLine($"thinking blocks: {thinkingBlocks}, dropped: {dropped}");
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  userTurns := []string{
  	"How many positive integers below 500 have exactly 6 positive divisors?",
  	"How many of those are odd?",
  }

  // messages 随轮次增长：每个 assistant 轮次均按返回时的原样传回
  messages := []anthropic.BetaMessageParam{}
  for _, userTurn := range userTurns {
  	messages = append(messages, anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock(userTurn)))
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
  		Messages: messages,
  		Betas:    []anthropic.AnthropicBeta{anthropic.AnthropicBetaThinkingBindingControls2026_08_01},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}
  	messages = append(messages, response.ToParam())
  	thinkingBlocks := 0
  	for _, block := range response.Content {
  		if block.Type == "thinking" {
  			thinkingBlocks++
  		}
  	}
  	fmt.Printf("thinking blocks: %d, dropped: %d\n", thinkingBlocks, len(response.InputTransformations))
  }
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaContentBlock;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaThinkingBlockBinding;
  import com.anthropic.models.beta.messages.BetaThinkingConfigAdaptive;
  import com.anthropic.models.beta.messages.BetaThinkingPrefixMismatchBehavior;
  import com.anthropic.models.beta.messages.MessageCreateParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      List<String> userTurns = List.of(
          "How many positive integers below 500 have exactly 6 positive divisors?",
          "How many of those are odd?");

      // builder 的消息列表随轮次增长：每个 assistant 轮次都按返回时的原样回传
      MessageCreateParams.Builder conversation = MessageCreateParams.builder()
          .model("claude-fable-5-1")
          .maxTokens(16000L)
          .thinking(BetaThinkingConfigAdaptive.builder()
              .blockBinding(BetaThinkingBlockBinding.builder()
                  .prefixMismatchBehavior(BetaThinkingPrefixMismatchBehavior.DROP_BLOCK)
                  .build())
              .build())
          .addBeta(AnthropicBeta.THINKING_BINDING_CONTROLS_2026_08_01);

      for (String userTurn : userTurns) {
          conversation.addUserMessage(userTurn);
          BetaMessage response = client.beta().messages().create(conversation.build());
          conversation.addMessage(response);
          long thinkingBlocks = response.content().stream()
              .filter(BetaContentBlock::isThinking)
              .count();
          int dropped = response.inputTransformations().map(List::size).orElse(0);
          IO.println("thinking blocks: " + thinkingBlocks + ", dropped: " + dropped);
      }
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaThinkingBlockBinding;
  use Anthropic\Beta\Messages\BetaThinkingConfigAdaptive;
  use Anthropic\Beta\Messages\BetaThinkingPrefixMismatchBehavior;
  use Anthropic\Client;

  $client = new Client();

  $userTurns = [
      'How many positive integers below 500 have exactly 6 positive divisors?',
      'How many of those are odd?',
  ];

  // $messages 随轮次增长：每个 assistant 轮次均按返回时的原样传回
  $messages = [];
  foreach ($userTurns as $userTurn) {
      $messages[] = ['role' => 'user', 'content' => $userTurn];
      $response = $client->beta->messages->create(
          model: 'claude-fable-5-1',
          maxTokens: 16000,
          thinking: BetaThinkingConfigAdaptive::with(
              blockBinding: BetaThinkingBlockBinding::with(
                  prefixMismatchBehavior: BetaThinkingPrefixMismatchBehavior::DROP_BLOCK,
              ),
          ),
          messages: $messages,
          betas: [AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01],
      );
      $messages[] = ['role' => 'assistant', 'content' => $response->content];
      $thinkingBlocks = array_filter($response->content, fn ($block) => $block->type === 'thinking');
      $dropped = $response->inputTransformations ?? [];
      echo 'thinking blocks: ', count($thinkingBlocks), ', dropped: ', count($dropped), PHP_EOL;
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  user_turns = [
    "How many positive integers below 500 have exactly 6 positive divisors?",
    "How many of those are odd?"
  ]

  # messages 随轮次增长：每个 assistant 轮次均按返回时的原样传回
  messages = []
  user_turns.each do |user_turn|
    messages << {role: "user", content: user_turn}
    response = client.beta.messages.create(
      model: "claude-fable-5-1",
      max_tokens: 16_000,
      thinking: {
        type: "adaptive",
        block_binding: {prefix_mismatch_behavior: "drop_block"}
      },
      messages: messages,
      betas: [Anthropic::AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01]
    )
    messages << {role: "assistant", content: response.content}
    thinking_blocks = response.content.count { |block| block.type == :thinking }
    dropped = (response.input_transformations || []).length
    puts "thinking blocks: #{thinking_blocks}, dropped: #{dropped}"
  end
  ```
</CodeGroup>

```text Output wrap
thinking blocks: 1, dropped: 0
thinking blocks: 1, dropped: 0
```

两轮都没有丢弃块，因为之前的内容没有任何更改。请检查第一个响应是否包含 `thinking` 块。使用自适应思考时，有些响应不包含思考块。如果会话中没有任何响应包含思考块，就没有可检查的内容，无论您更改什么，丢弃计数都为 0，因此请再次运行该示例。

在您自己集成的每一轮中记录 `input_transformations`。当 API 丢弃某个块时，条目如下所示：

```json
{
  "input_transformations": [
    {
      "type": "thinking_dropped",
      "path": "messages.1.content.0",
      "reason": "prefix_binding_mismatch"
    }
  ]
}
```

* **在包含 `thinking` 块的会话中每轮都为空：** 您的集成保持了前缀完整。
* **`reason: "prefix_binding_mismatch"`：** 自上一次请求以来，`path` 处的块之前的某些内容发生了更改。对比截至该轮的 `system`、`tools` 和 `messages` 以找到它，或使用 `"error"` 重新发送请求：400 错误通常以一句说明更改内容的话结尾。然后在[在不编辑前缀的情况下进行更改](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#replace-prefix-edits)中找到对应的替代方案。
* **`reason: "model_binding_mismatch"`：** 对话切换到了无法读取早期模型块的模型。这不是前缀编辑。请参阅[在对话中途切换模型](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#switching-models)。

要有意观察失败情况，请从前面的示例发送第三轮，并仅在该请求中添加 `system` 提示，使其与前两个没有 `system` 提示的请求不同。使用 `"drop_block"` 时，丢弃计数不再为 0：响应会为历史记录中的每个思考块列出一个条目，每个条目都带有 `reason: "prefix_binding_mismatch"`。使用 `"error"` 时，请求会返回[API 如何处理失效的块](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#mismatch-behavior)中描述的 400 错误，其最后一句会指出 `system` 提示。在 cURL 和 CLI 选项卡中，删除 `jq` 过滤器即可查看错误正文。如果计数仍为 0，说明没有可检查的内容：请确认模型是 claude-fable-5-1、请求设置了 `block_binding`、您发送的历史记录包含 `thinking` 块，并且前两个请求没有 `system` 提示。

两个普通轮次很少能暴露问题。请针对以下每种情况运行一个会话，并设置 `"error"`，以便回归问题会导致您的 CI 失败：

* 第一次客户端压缩或裁剪
* 在第一轮之后才连接的工具、插件或 MCP 服务器
* 模式或指令更改
* 长工具循环（如果您会添加提醒或缩短旧的工具结果）
* 切换到另一个模型再切换回来
* 保存、重启，并在之后的某天恢复

在较早的账户上，您还可以在不选择加入强制执行检查的情况下观察生产流量：发送 beta 标头，不设置 `block_binding`，并记录 `input_transformations`。仅发送该标头不会改变模型接收的内容。位于您所编辑内容之后的每个思考块都无法通过检查，并各自获得一个 `thinking_mismatch_allowed` 条目，其 `path` 和 `reason` 字段与 `thinking_dropped` 条目相同。编辑之前的块仍会通过。对 `system` 或 `tools` 的编辑位于所有块之前，因此会使请求中的所有思考块都未通过检查。一个条目如下所示：

```json
{
  "input_transformations": [
    {
      "type": "thinking_mismatch_allowed",
      "path": "messages.1.content.0",
      "reason": "prefix_binding_mismatch"
    }
  ]
}
```

请从[较早的账户](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#enforcement)运行以下示例，因为较新的账户会以 400 拒绝其第三个请求。它发送标头但不设置 `block_binding`，并通过第三个请求扩展之前的会话，该请求添加了系统提示，从而有意更改前缀。每轮结束后，它会打印 `thinking` 块的数量和被标记块的数量：

<CodeGroup>
  ```bash cURL
  # 统计响应中的 thinking 块数量，以及未通过前缀检查的块数量
  COUNTS='"thinking blocks: \([.content[] | select(.type == "thinking")] | length), " +
    "flagged: \([.input_transformations[] |
      select(.type == "thinking_mismatch_allowed")] | length)"'

  FIRST=$(curl -s https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: thinking-binding-controls-2026-08-01" \
    -d '{
      "model": "claude-fable-5-1",
      "max_tokens": 16000,
      "thinking": { "type": "adaptive" },
      "messages": [
        {
          "role": "user",
          "content": "How many positive integers below 500 have exactly 6 positive divisors?"
        }
      ]
    }')
  echo "$FIRST" | jq -r "$COUNTS"

  # 第 2 轮：assistant 轮次按返回时的原样传回，随后是下一条用户消息
  MESSAGES=$(jq -n --argjson first "$FIRST" '[
    {
      role: "user",
      content: "How many positive integers below 500 have exactly 6 positive divisors?"
    },
    { role: "assistant", content: $first.content },
    { role: "user", content: "How many of those are odd?" }
  ]')

  SECOND=$(jq -n --argjson messages "$MESSAGES" '{
    model: "claude-fable-5-1",
    max_tokens: 16000,
    thinking: { type: "adaptive" },
    messages: $messages
  }' | curl -s https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: thinking-binding-controls-2026-08-01" \
    -d @-)
  echo "$SECOND" | jq -r "$COUNTS"

  # 第 3 轮：仅此请求添加了系统提示，有意改变前缀
  jq -n --argjson messages "$MESSAGES" --argjson second "$SECOND" '{
    model: "claude-fable-5-1",
    max_tokens: 16000,
    thinking: { type: "adaptive" },
    system: "Answer briefly.",
    messages: ($messages + [
      { role: "assistant", content: $second.content },
      { role: "user", content: "And how many of the odd ones are below 100?" }
    ])
  }' | curl -s https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: thinking-binding-controls-2026-08-01" \
    -d @- | jq -r "$COUNTS"
  ```

  ```bash CLI
  # 统计响应中的 thinking 块数量，以及未通过前缀检查的块数量
  COUNTS='"thinking blocks: \([.content[] | select(.type == "thinking")] | length), " +
    "flagged: \([.input_transformations[] |
      select(.type == "thinking_mismatch_allowed")] | length)"'

  FIRST=$(ant beta:messages create --beta thinking-binding-controls-2026-08-01 \
    --format json <<'YAML'
  model: claude-fable-5-1
  max_tokens: 16000
  thinking:
    type: adaptive
  messages:
    - role: user
      content: How many positive integers below 500 have exactly 6 positive divisors?
  YAML
  )
  echo "$FIRST" | jq -r "$COUNTS"

  # 第 2 轮：assistant 轮次按返回时的原样传回，随后是下一条用户消息
  SECOND=$(ant beta:messages create --beta thinking-binding-controls-2026-08-01 \
    --format json <<YAML
  model: claude-fable-5-1
  max_tokens: 16000
  thinking:
    type: adaptive
  messages:
    - role: user
      content: How many positive integers below 500 have exactly 6 positive divisors?
    - role: assistant
      content: $(echo "$FIRST" | jq -c .content)
    - role: user
      content: How many of those are odd?
  YAML
  )
  echo "$SECOND" | jq -r "$COUNTS"

  # 第 3 轮：仅此请求添加了系统提示，有意改变前缀
  ant beta:messages create --beta thinking-binding-controls-2026-08-01 \
    --format json <<YAML | jq -r "$COUNTS"
  model: claude-fable-5-1
  max_tokens: 16000
  thinking:
    type: adaptive
  system: Answer briefly.
  messages:
    - role: user
      content: How many positive integers below 500 have exactly 6 positive divisors?
    - role: assistant
      content: $(echo "$FIRST" | jq -c .content)
    - role: user
      content: How many of those are odd?
    - role: assistant
      content: $(echo "$SECOND" | jq -c .content)
    - role: user
      content: And how many of the odd ones are below 100?
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  user_turns = [
      "How many positive integers below 500 have exactly 6 positive divisors?",
      "How many of those are odd?",
      "And how many of the odd ones are below 100?",
  ]

  # messages 随轮次增长：每个 assistant 轮次均按返回时的原样回传
  messages = []
  for turn, user_turn in enumerate(user_turns, start=1):
      messages.append({"role": "user", "content": user_turn})
      response = client.beta.messages.create(
          model="claude-fable-5-1",
          max_tokens=16000,
          thinking={"type": "adaptive"},
          # 仅最后一个请求添加了系统提示，这是有意改变前缀
          system="Answer briefly." if turn == len(user_turns) else anthropic.omit,
          messages=messages,
          betas=["thinking-binding-controls-2026-08-01"],
      )
      messages.append({"role": "assistant", "content": response.content})
      thinking_blocks = sum(block.type == "thinking" for block in response.content)
      flagged = sum(
          transformation.type == "thinking_mismatch_allowed"
          for transformation in response.input_transformations or []
      )
      print(f"thinking blocks: {thinking_blocks}, flagged: {flagged}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const userTurns = [
    "How many positive integers below 500 have exactly 6 positive divisors?",
    "How many of those are odd?",
    "And how many of the odd ones are below 100?"
  ];

  // messages 随轮次增长：每个 assistant 轮次都按返回时的原样传回
  const messages: Anthropic.Beta.BetaMessageParam[] = [];
  for (const [turnIndex, userTurn] of userTurns.entries()) {
    messages.push({ role: "user", content: userTurn });
    const response = await client.beta.messages.create({
      model: "claude-fable-5-1",
      max_tokens: 16000,
      thinking: { type: "adaptive" },
      // 仅最后一个请求添加了系统提示，这是有意更改前缀
      system: turnIndex === userTurns.length - 1 ? "Answer briefly." : undefined,
      messages,
      betas: ["thinking-binding-controls-2026-08-01"]
    });
    messages.push({ role: "assistant", content: response.content });
    const thinkingBlocks = response.content.filter((block) => block.type === "thinking");
    const flagged = (response.input_transformations ?? []).filter(
      (transformation) => transformation.type === "thinking_mismatch_allowed"
    );
    console.log(`thinking blocks: ${thinkingBlocks.length}, flagged: ${flagged.length}`);
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  string[] userTurns =
  [
      "How many positive integers below 500 have exactly 6 positive divisors?",
      "How many of those are odd?",
      "And how many of the odd ones are below 100?",
  ];

  // messages 随轮次增长：每个 assistant 轮次都按返回时的原样回传
  List<BetaMessageParam> messages = [];
  for (var turnIndex = 0; turnIndex < userTurns.Length; turnIndex++)
  {
      messages.Add(new() { Role = Role.User, Content = userTurns[turnIndex] });
      var response = await client.Beta.Messages.Create(
          new()
          {
              Model = "claude-fable-5-1",
              MaxTokens = 16000,
              Thinking = new BetaThinkingConfigAdaptive(),
              // 仅最后一个请求添加系统提示，有意改变前缀
              System = turnIndex == userTurns.Length - 1 ? new("Answer briefly.") : null,
              Messages = messages,
              Betas = [AnthropicBeta.ThinkingBindingControls2026_08_01],
          }
      );
      messages.Add(new()
      {
          Role = Role.Assistant,
          Content = response.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
      });
      var thinkingBlocks = response.Content.Count(block => block.TryPickThinking(out _));
      var flagged = response.InputTransformations?.Count(transformation =>
          transformation.TryPickThinkingMismatchAllowed(out _)
      ) ?? 0;
      Console.WriteLine($"thinking blocks: {thinkingBlocks}, flagged: {flagged}");
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  userTurns := []string{
  	"How many positive integers below 500 have exactly 6 positive divisors?",
  	"How many of those are odd?",
  	"And how many of the odd ones are below 100?",
  }

  // messages 随轮次增长：每个 assistant 轮次都按返回时的原样传回
  messages := []anthropic.BetaMessageParam{}
  for i, userTurn := range userTurns {
  	messages = append(messages, anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock(userTurn)))
  	// 仅最后一个请求添加 system prompt（系统提示），有意改变前缀
  	var system []anthropic.BetaTextBlockParam
  	if i == len(userTurns)-1 {
  		system = []anthropic.BetaTextBlockParam{{Text: "Answer briefly."}}
  	}
  	response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  		Model:     "claude-fable-5-1",
  		MaxTokens: 16000,
  		Thinking: anthropic.BetaThinkingConfigParamUnion{
  			OfAdaptive: &anthropic.BetaThinkingConfigAdaptiveParam{},
  		},
  		System:   system,
  		Messages: messages,
  		Betas:    []anthropic.AnthropicBeta{anthropic.AnthropicBetaThinkingBindingControls2026_08_01},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}
  	messages = append(messages, response.ToParam())
  	thinkingBlocks := 0
  	for _, block := range response.Content {
  		if block.Type == "thinking" {
  			thinkingBlocks++
  		}
  	}
  	flagged := 0
  	for _, transformation := range response.InputTransformations {
  		if transformation.Type == "thinking_mismatch_allowed" {
  			flagged++
  		}
  	}
  	fmt.Printf("thinking blocks: %d, flagged: %d\n", thinkingBlocks, flagged)
  }
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaContentBlock;
  import com.anthropic.models.beta.messages.BetaInputTransformation;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaThinkingConfigAdaptive;
  import com.anthropic.models.beta.messages.MessageCreateParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      List<String> userTurns = List.of(
          "How many positive integers below 500 have exactly 6 positive divisors?",
          "How many of those are odd?",
          "And how many of the odd ones are below 100?");

      // 构建器的消息列表随轮次增长：每个 assistant 轮次都按返回时的原样回传
      MessageCreateParams.Builder conversation = MessageCreateParams.builder()
          .model("claude-fable-5-1")
          .maxTokens(16000L)
          .thinking(BetaThinkingConfigAdaptive.builder().build())
          .addBeta(AnthropicBeta.THINKING_BINDING_CONTROLS_2026_08_01);

      for (int turnIndex = 0; turnIndex < userTurns.size(); turnIndex++) {
          if (turnIndex == userTurns.size() - 1) {
              // 只有最后一个请求添加了 system prompt（系统提示），这是有意改变前缀
              conversation.system("Answer briefly.");
          }
          conversation.addUserMessage(userTurns.get(turnIndex));
          BetaMessage response = client.beta().messages().create(conversation.build());
          conversation.addMessage(response);
          long thinkingBlocks = response.content().stream()
              .filter(BetaContentBlock::isThinking)
              .count();
          long flagged = response.inputTransformations().stream()
              .flatMap(List::stream)
              .filter(BetaInputTransformation::isThinkingMismatchAllowed)
              .count();
          IO.println("thinking blocks: " + thinkingBlocks + ", flagged: " + flagged);
      }
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaThinkingConfigAdaptive;
  use Anthropic\Beta\Messages\BetaThinkingMismatchAllowedInputTransformation;
  use Anthropic\Client;

  $client = new Client();

  $userTurns = [
      'How many positive integers below 500 have exactly 6 positive divisors?',
      'How many of those are odd?',
      'And how many of the odd ones are below 100?',
  ];

  // $messages 随轮次增长：每个 assistant 轮次均按返回时的原样传回
  $messages = [];
  foreach ($userTurns as $turnIndex => $userTurn) {
      $messages[] = ['role' => 'user', 'content' => $userTurn];
      $response = $client->beta->messages->create(
          model: 'claude-fable-5-1',
          maxTokens: 16000,
          thinking: BetaThinkingConfigAdaptive::with(),
          // 仅最后一个请求添加系统提示，这会有意改变前缀
          system: $turnIndex === array_key_last($userTurns) ? 'Answer briefly.' : null,
          messages: $messages,
          betas: [AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01],
      );
      $messages[] = ['role' => 'assistant', 'content' => $response->content];
      $thinkingBlocks = array_filter($response->content, fn ($block) => $block->type === 'thinking');
      $flagged = array_filter(
          $response->inputTransformations ?? [],
          fn ($transformation) => $transformation instanceof BetaThinkingMismatchAllowedInputTransformation,
      );
      echo 'thinking blocks: ', count($thinkingBlocks), ', flagged: ', count($flagged), PHP_EOL;
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  user_turns = [
    "How many positive integers below 500 have exactly 6 positive divisors?",
    "How many of those are odd?",
    "And how many of the odd ones are below 100?"
  ]

  # messages 随轮次增长：每个 assistant 轮次均按返回时的原样传回
  messages = []
  user_turns.each_with_index do |user_turn, turn_index|
    messages << {role: "user", content: user_turn}
    # 仅最后一个请求添加了系统提示，这是有意改变前缀
    system_param = (turn_index == user_turns.length - 1) ? {system_: "Answer briefly."} : {}
    response = client.beta.messages.create(
      model: "claude-fable-5-1",
      max_tokens: 16_000,
      thinking: {type: "adaptive"},
      messages: messages,
      betas: [Anthropic::AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01],
      **system_param
    )
    messages << {role: "assistant", content: response.content}
    thinking_blocks = response.content.count { |block| block.type == :thinking }
    flagged = (response.input_transformations || []).count do |transformation|
      transformation.type == :thinking_mismatch_allowed
    end
    puts "thinking blocks: #{thinking_blocks}, flagged: #{flagged}"
  end
  ```
</CodeGroup>

```text Output wrap
thinking blocks: 1, flagged: 0
thinking blocks: 1, flagged: 0
thinking blocks: 1, flagged: 2
```

第三个响应会标记来自之前轮次的每个思考块（在本次运行中每轮一个），因为新的系统提示位于所有这些块之前。模型仍然读取了它们。

请像处理 `prefix_binding_mismatch` 丢弃一样处理这些条目。编辑位于列出的第一个块之前：将截至该块 `path` 的 `system`、`tools` 和 `messages` 与上一次请求进行对比以找到它，然后用[在不编辑前缀的情况下进行更改](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#replace-prefix-edits)中对应的模式替换它。在新账户上，或在任何设置了 `prefix_mismatch_behavior` 的请求上，API 会改为拒绝请求或丢弃未通过检查的块。这些条目是强制执行检查时将被删除内容的下限：使用 `"drop_block"` 时，一个未通过检查的块还会连带删除该轮其余的思考块，而删除一个块也可能导致下一个块未通过检查。当 API 仅记录检查结果时，它会单独判断每个块，并且只在该块本身未通过时才将其列出。

## 在不编辑前缀的情况下进行更改

每种常见的前缀编辑都有一种替代方案，它能向模型提供相同的信息，同时保持之前的字节不变，从而使后续思考保持有效。请在第一列中找到您的代码目前进行的编辑：

| 不要                                          | 改用                                                                                                                                                                                                                                                                                                                                                     | Beta 标头                                                                                                                                                                          |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 重建顶层 `system` 提示                            | [对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#new-instructions)                                                                                                                                                                                                                                               | 无                                                                                                                                                                                |
| 在每次请求时重新渲染第一条用户消息中的上下文（环境、日期、记忆、项目指令）       | 渲染一次并原样重新发送。当内容发生变化时，[将新版本放在最新轮次中](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#changing-context)                                                                                                                                                                                                                       | 无                                                                                                                                                                                |
| 就地清除或缩短旧的 `tool_result` 内容，或重新编码旧图像         | 在首次发送之前（而不是之后）缩短工具结果或缩小图像。如需稍后清除旧结果，请使用 `clear_tool_uses_20250919` [在服务器端裁剪上下文](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#server-side-trimming)                                                                                                                                                                      | `context-management-2025-06-27`                                                                                                                                                  |
| 注入提醒并在下一次请求时删除它                             | [轮次范围系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#per-turn-reminders)（`clear_at: "next_user_message"`）                                                                                                                                                                                                            | `mid-conversation-system-clear-at-2026-08-21`                                                                                                                                    |
| 在 `tools` 中添加或删除条目                          | [`tool_addition` 和 `tool_removal` 块](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#tool-changes)                                                                                                                                                                                                                         | `inline-tools-2026-09-15`（当工具来自通过 MCP 连接器连接的 MCP 服务器时，添加 `mcp-client-2026-09-15`），或较旧的 `mid-conversation-tool-changes-2026-07-01`，后者适用于 Claude API、Amazon Bedrock 和 Google Cloud |
| 更改顶层 `output_config.effort`（会使缓存重新开始，不影响思考） | [按消息设置的 `output_config`](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#effort-changes)                                                                                                                                                                                                                                   | `mid-conversation-output-config-2026-07-01`                                                                                                                                      |
| 在客户端丢弃或汇总旧轮次                                | 使用[按需压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand)保留最近轮次及其思考，使用其他服务器端[压缩或上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#server-side-trimming)，或使用不保留过时思考的[客户端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#custom-compaction-on-the-client) | `compact-2026-09-04`                                                                                                                                                             |
| 字节在请求之间发生变化的图像或文档 URL                       | [来自 Files API 的 `file_id`](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#files-by-id)，或 base64                                                                                                                                                                                                                           | 无                                                                                                                                                                                |

以上所有方案都假定您[完全按照返回的样子发回助手轮次](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#append-assistant-turns-exactly-as-returned)。对话中途系统消息、轮次范围系统消息和工具更改并非在所有模型上都可用：[对话中途系统消息和工具更改](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)列出了接受它们的模型。如果您的代码服务于多个模型，对于不接受它们的模型，请继续编辑顶层 `system` 提示。

要在一个请求中使用多个 beta，请将这些值合并到一个 `anthropic-beta` 标头中。只要某个 beta 在 Amazon Bedrock 和 Google Cloud 上可用，其名称在这些平台上都相同（请参阅 [Beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)）：

```text wrap
anthropic-beta: thinking-binding-controls-2026-08-01,mid-conversation-system-clear-at-2026-08-21,inline-tools-2026-09-15
```

### 完全按照返回的样子发回助手轮次

存储每个响应中的 `content` 数组，并将其原样作为助手轮次发回：包括所有块类型，按接收顺序排列，包括 `thinking` 字段为空的 `thinking` 块。如果序列化器丢弃未知块类型、丢弃空字段或重新排序块，就会编辑之后每一轮的前缀。

在 Claude Fable 5.1 上，`thinking` 字段默认为空，推理由 `signature` 承载，因此跳过空块的序列化器会删除思考。如果它删除了所有思考块，不会有任何失败，但模型在每一轮都会失去其早期推理。如果您自行解析流，即使没有收到思考文本也要保留该块：它会开启，在 `signature_delta` 事件中接收其 `signature`，然后关闭。以空 `signature` 发回的块会失败。

### 使用对话中途系统消息添加指令

有些运行框架会在每次请求时重建顶层 `system` 提示，以携带当前时间、令牌预算、模式标志或新发现的项目上下文。这会使对话中的所有思考块失效。相反，请在会话开始时固定 `system`。当有内容更改时，在 `messages` 中该更改生效的位置追加一条 [`role: "system"` 消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)：

```json
{
  "role": "system",
  "content": "The user switched the workspace to read-only mode. Do not write files until told otherwise."
}
```

模型会以系统提示的权威性对待这条消息，而它之前的所有内容保持不变。在工具循环中，请将该消息放在 `tool_result` 用户消息之后，切勿放在助手的 `tool_use` 与其 `tool_result` 之间（请参阅[限制](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)）。一旦发送，该消息就成为后续思考前缀的一部分：在后续请求中请将其保留在原位。

### 将变化的上下文放在最新轮次中

有些运行框架会在第一条用户消息中放入环境块（工作目录、分支、日期、记忆、项目说明），并在每次请求时重新渲染。当任何值发生变化时，`messages[0]` 就会改变，对话中的所有思考块都会失效。请只渲染一次该块，并按原样重新发送。当某个值发生变化时，请在最新轮次中说明：向您即将发送的用户消息添加一个文本块，或者如果更改来自作为运营方的您，则追加一条[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#new-instructions)。

```json
{
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "Environment update: the current branch is now release-2."
    },
    { "type": "text", "text": "Run the tests again." }
  ]
}
```

一旦发送，该文本块就成为后续思考前缀的一部分：在后续请求中请将其保留在原位。

### 将每轮提醒作为轮次作用域系统消息发送

一种常见的前缀编辑是每轮提示：例如"将独立的读取操作一起请求"或"您已经有一段时间没有向用户更新进展了"这样的一行文字，由您的代码在每批工具结果之后追加。为避免提醒不断堆积，请将每条提示作为带有 `clear_at: "next_user_message"` 的[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)发送，并放在 `tool_result` 用户消息之后。`clear_at` 需要 beta 标头 `mid-conversation-system-clear-at-2026-08-21`。以下 `messages` 数组是两次工具调用及其结果之后的请求。`messages[3]` 是上一次请求的提示，保留在原位，`messages[6]` 是本次请求的副本：

```json
[
  { "role": "user", "content": "Fix the failing test." },
  {
    "role": "assistant",
    "content": [
      { "type": "thinking", "thinking": "", "signature": "..." },
      {
        "type": "tool_use",
        "id": "toolu_01",
        "name": "read_file",
        "input": { "path": "tests/test_auth.py" }
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
    "content": "Request every independent read in one turn."
  },
  {
    "role": "assistant",
    "content": [
      { "type": "thinking", "thinking": "", "signature": "..." },
      {
        "type": "tool_use",
        "id": "toolu_02",
        "name": "read_file",
        "input": { "path": "src/auth.py" }
      }
    ]
  },
  {
    "role": "user",
    "content": [{ "type": "tool_result", "tool_use_id": "toolu_02", "content": "..." }]
  },
  {
    "role": "system",
    "clear_at": "next_user_message",
    "content": "Request every independent read in one turn."
  }
]
```

仅包含 `tool_result` 块的用户消息也算作"下一条用户消息"，因此 `messages[3]` 已被清除。它不会为模型看到的内容增加任何东西，也不消耗输入令牌，但由于它仍在数组中，`messages[4]` 中的思考保持有效。`messages[6]` 是模型在本轮看到的副本。在后续请求中，请将两者都保留在原位，并在下一条 `tool_result` 消息之后追加一个新副本。

### 使用 `tool_addition` 和 `tool_removal` 添加或删除工具

在会话中途编辑 `tools` 数组会使保留的思考块失效。请保持该数组与您首次发送时一致，并通过追加一条携带 `tool_addition` 或 `tool_removal` 块的 `role: "system"` 消息，来更改模型可以使用的工具。这些属于[对话中途的工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes)，需要 beta 标头 `inline-tools-2026-09-15`，该标头在 Claude API 上可用。较旧的 `mid-conversation-tool-changes-2026-07-01` 标头仍适用于通过引用指定工具的变更，可在 Claude API、Amazon Bedrock 和 Google Cloud 上使用。

您可以通过两种方式使用这些块：

* **预先声明所有工具。** 在第一个请求中将会话可能需要的所有工具放入 `tools`，并为模型暂时不应看到的工具设置 `defer_loading: true`。然后使用指定这些工具的 `tool_addition` 和 `tool_removal` 块来启用和停用工具。
* **从快照开始，并随时添加工具。** 在第一个请求中将您已知的工具放入 `tools`。当出现新工具时，在 `tool_addition` 块中定义它，而不是编辑 `tools`。

无论哪种方式，`tools` 都不会改变，因此之前的思考保持有效，提示缓存也仍然命中，下文所述的一种例外情况除外。

例如，在模式切换后撤回一个危险工具：

```json
{
  "role": "system",
  "content": [
    { "type": "tool_removal", "tool": { "type": "tool_reference", "name": "delete_branch" } },
    { "type": "text", "text": "Branch deletion is disabled for the rest of this session." }
  ]
}
```

要启用您使用 `defer_loading: true` 声明的工具，请追加一个指定该工具的 `tool_addition` 块：

```json
{
  "role": "system",
  "content": [
    { "type": "tool_addition", "tool": { "type": "tool_reference", "name": "deploy" } },
    { "type": "text", "text": "Authentication succeeded. Deployment is now available." }
  ]
}
```

有时您无法预先声明某个工具，因为您还不知道它的 schema：例如您的应用程序在运行时发现的工具，或在第一轮之后才连接的 MCP 服务器。请在 `tool_addition` 块中定义它，而不是修改 `tools`。使用 `inline-tools-2026-09-15` 时，该块的 `tool` 可以是 `{"type": "tool_definition", "definition": {...}}`，其中携带的条目与您原本会放入 `tools` 的条目相同：

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

新工具通过 `messages` 到达，`tools` 从不改变，之前的思考保持有效。请在 `tools` 中至少保留一个未设置 `defer_loading: true` 的工具：如果其中所有工具都是延迟加载的，那么您以这种方式定义的第一个工具会导致一次完整的提示缓存未命中。

如果 API 通过 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)为您连接 MCP 服务器，还需发送 `mcp-client-2026-09-15`。它涵盖了 `mcp-client-2025-11-20` 的所有功能，因此请发送它来代替后者。此时该块的 `definition` 可以是针对 `mcp_servers` 中所列服务器的 `mcp_toolset`。当 API 需要获取某个服务器的工具列表时，响应会以该服务器的 `mcp_tool_listing` 块开头。请将其与助手轮次的其余部分一起原样发回，并在之后每个携带该块的请求中持续发送 `mcp-client-2026-09-15`。该块会将工具集固定到该列表，因此 API 不会为此再次联系服务器。这些 MCP 连接器功能在 Claude API 上可用。

如果只有较早的标头，您仍然可以将会话中途得知的工具以 `defer_loading: true` 追加到 `tools`，然后通过 `tool_addition` 块提供它。这样做是安全的，因为在 `tool_addition` 块引用延迟工具之前，前缀检查会忽略它。添加未设置 `defer_loading: true` 的工具会改变前缀，并使之前的思考失效。

携带这些块的 `role: "system"` 消息会成为后续思考前缀的一部分。在后续请求中请将其保留在原位。

### 使用按消息设置的 `output_config` 更改 effort

在请求之间更改顶层 `output_config.effort` 不会使思考失效，因为 effort 不属于前缀。但更改顶层 effort 确实会使提示缓存重新开始。在 Claude Fable 5.1 上，请改用[按消息设置的 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)：追加一条 `content` 为空并带有新级别的 `role: "system"` 消息。它需要 beta 标头 `mid-conversation-output-config-2026-07-01`。

```json
{ "role": "system", "content": [], "output_config": { "effort": "low" } }
```

新级别从下一个 `user` 轮次开始生效。一旦发送，该消息就成为 `messages` 的一部分，因而也是后续思考前缀的一部分：在后续请求中请将其保留在原位，如需再次更改 effort，请再追加一条。

### 在服务器端裁剪上下文

另一种常见的前缀编辑是客户端裁剪：丢弃或汇总最旧的轮次，并逐字保留近期轮次。被保留轮次的思考块是在被删除的历史记录仍存在时生成的，因此它们无法通过检查。服务器端的等效操作不算作编辑，因为检查比较的是您发送的对话：

* [按需压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand)（beta）通过单独的请求返回摘要，该请求可以[在后台运行](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-background)，您发送返回的块来替换它所总结的消息。检查接受这种替换，因此在[保留思考的条件](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)下，您保留的轮次及其思考可以保持有效。[请求摘要](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#request-a-summary)展示了该请求，并列出了所需的 beta 标头。
* [在令牌阈值处进行压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold)会在上下文接近您设置的阈值时，将较早的轮次总结为一个压缩块，被检查的前缀从该块重新开始。其 [`instructions` 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold#custom-summarization-instructions)接受您自己的摘要提示，例如"保留每个股票代码、仓位规模和陈述的假设"。
* [上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)按规则清除旧的工具结果或旧的思考块，从最早的开始。相应策略为 `clear_tool_uses_20250919` 和 `clear_thinking_20251015`。

### 在客户端进行压缩

您仍然可以在客户端进行 "compaction"（压缩）。如果您自己编写摘要，请不要发回在重写之前生成的思考块。如果由 API 通过按需压缩编写摘要，[保留的思考保持有效的条件](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)列出了保留的思考何时保持有效。

#### 简单压缩（推荐）

当对话变得过长时，将整个会话总结为一条用户消息，然后只发送这条消息和下一条指令。之前的内容都不会被重放，因此不会留下任何可能无法通过检查的思考，模型会根据摘要重新进行推理。

![“Simple compaction”（简单压缩）：请求 4 发送完整的历史记录，每个助手轮次都带有思考；请求 5 发送一条用户消息，其中包含第 1 至 4 轮的摘要以及下一条指令，因此不会发送任何先前的思考，也不会检查任何内容](https://platform.claude.com/docs/images/preserved-thinking-simple-compaction.svg)

```json
[
  {
    "role": "user",
    "content": "<summary of the session so far>\n\n<the next instruction>"
  }
]
```

Claude 模型在长周期任务上使用这种方案进行过训练，对于大多数工作负载，它的表现都很好。

#### 保留尾部压缩

"Keep-tail compaction"（保留尾部压缩）会总结较早的轮次，并原样保留最近的轮次，因此模型仍能逐字看到最后几轮交互。如果您自己编写摘要，就会违反规则：保留的助手轮次仍然带有思考块，而这些思考块是在其前面是原始轮次（而非摘要）时生成的。这些块会失败。

要保留这些思考，请让 API 通过按需压缩编写摘要。[保留最近轮次的压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-keep-recent-turns)展示了具体方法，[保留的思考保持有效的条件](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)列出了保留的思考何时保持有效。

本节其余部分介绍您自己编写摘要的情况。

![“Keep-tail compaction”（保留尾部压缩）：历史记录被替换为第 1 和第 2 轮的摘要，后跟原样保留的第 3 至 5 轮；助手第 3 和第 4 轮上的思考是在原始轮次（而非摘要）之后生成的，因此会失败；使用 prefix\_mismatch\_behavior drop\_block 发送的相同请求会成功，API 会丢弃这两个块并在 input\_transformations 中列出它们](https://platform.claude.com/docs/images/preserved-thinking-keep-tail-compaction.svg)

解决方法：保持这些轮次原样不变，并发送 `prefix_mismatch_behavior: "drop_block"`。API 会丢弃过时的思考块，模型会读取保留轮次中的 `text` 和 `tool_use` 块，请求即可成功。

将压缩后的历史记录作为 `messages` 传入，并在 `thinking` 配置上设置 `block_binding`。在以下示例中，`compacted_messages` 是您的压缩步骤生成的数组：摘要消息，后跟按 API 返回原样保留的轮次，包括 `thinking` 块：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: thinking-binding-controls-2026-08-01" \
    -d "{
      \"model\": \"claude-fable-5-1\",
      \"max_tokens\": 16000,
      \"thinking\": {
        \"type\": \"adaptive\",
        \"block_binding\": { \"prefix_mismatch_behavior\": \"drop_block\" }
      },
      \"messages\": $COMPACTED_MESSAGES
    }"
  ```

  ```bash CLI
  ant beta:messages create --beta thinking-binding-controls-2026-08-01 <<YAML
  model: claude-fable-5-1
  max_tokens: 16000
  thinking:
    type: adaptive
    block_binding:
      prefix_mismatch_behavior: drop_block
  messages: $COMPACTED_MESSAGES
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  # compacted_messages：先是摘要消息，然后是按返回原样保留的轮次
  response = client.beta.messages.create(
      model="claude-fable-5-1",
      max_tokens=16000,
      thinking={
          "type": "adaptive",
          "block_binding": {"prefix_mismatch_behavior": "drop_block"},
      },
      messages=compacted_messages,
      betas=["thinking-binding-controls-2026-08-01"],
  )

  print(response.input_transformations)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // compactedMessages：先是摘要消息，然后是按原样返回的保留轮次
  const response = await client.beta.messages.create({
    model: "claude-fable-5-1",
    max_tokens: 16000,
    thinking: {
      type: "adaptive",
      block_binding: { prefix_mismatch_behavior: "drop_block" }
    },
    messages: compactedMessages,
    betas: ["thinking-binding-controls-2026-08-01"]
  });

  console.log(response.input_transformations);
  ```

  ```csharp C#
  AnthropicClient client = new();

  // compactedMessages：先是摘要消息，然后是按返回顺序保留的轮次
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
          Messages = compactedMessages,
          Betas = [AnthropicBeta.ThinkingBindingControls2026_08_01],
      }
  );

  Console.WriteLine(response.InputTransformations?.Count ?? 0);
  ```

  ```go Go
  client := anthropic.NewClient()

  // compactedMessages：先是摘要消息，然后是按原样返回的保留轮次
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
  	Messages: compactedMessages,
  	Betas:    []anthropic.AnthropicBeta{anthropic.AnthropicBetaThinkingBindingControls2026_08_01},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(len(response.InputTransformations))
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

      // compactedMessages：先是摘要消息，然后是按原样返回的保留轮次
      MessageCreateParams params = MessageCreateParams.builder()
          .model("claude-fable-5-1")
          .maxTokens(16000L)
          .thinking(BetaThinkingConfigAdaptive.builder()
              .blockBinding(BetaThinkingBlockBinding.builder()
                  .prefixMismatchBehavior(BetaThinkingPrefixMismatchBehavior.DROP_BLOCK)
                  .build())
              .build())
          .messages(compactedMessages)
          .addBeta(AnthropicBeta.THINKING_BINDING_CONTROLS_2026_08_01)
          .build();

      BetaMessage response = client.beta().messages().create(params);

      IO.println(response.inputTransformations());
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaThinkingBlockBinding;
  use Anthropic\Beta\Messages\BetaThinkingConfigAdaptive;
  use Anthropic\Beta\Messages\BetaThinkingPrefixMismatchBehavior;
  use Anthropic\Client;

  $client = new Client();

  // $compactedMessages：先是摘要消息，然后是按原样返回的保留轮次
  $response = $client->beta->messages->create(
      model: 'claude-fable-5-1',
      maxTokens: 16000,
      thinking: BetaThinkingConfigAdaptive::with(
          blockBinding: BetaThinkingBlockBinding::with(
              prefixMismatchBehavior: BetaThinkingPrefixMismatchBehavior::DROP_BLOCK,
          ),
      ),
      messages: $compactedMessages,
      betas: [AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01],
  );

  var_dump($response->inputTransformations);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # compacted_messages：先是摘要消息，然后是按返回原样保留的轮次
  response = client.beta.messages.create(
    model: "claude-fable-5-1",
    max_tokens: 16_000,
    thinking: {
      type: "adaptive",
      block_binding: {prefix_mismatch_behavior: "drop_block"}
    },
    messages: compacted_messages,
    betas: [Anthropic::AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01]
  )

  puts response.input_transformations
  ```
</CodeGroup>

响应会照常包含新的助手轮次，并且每个被丢弃的块都对应一个 `input_transformations` 条目。对于图中的历史记录，被丢弃的是助手第 3 和第 4 轮上的思考：

```json
{
  "input_transformations": [
    {
      "type": "thinking_dropped",
      "path": "messages.2.content.0",
      "reason": "prefix_binding_mismatch"
    },
    {
      "type": "thinking_dropped",
      "path": "messages.4.content.0",
      "reason": "prefix_binding_mismatch"
    }
  ]
}
```

只要这两个轮次仍在历史记录中，后续请求就要继续发送 `"drop_block"`。模型从此请求开始生成的思考都位于摘要之后，因此保持有效。如果您不想依赖该 beta 标头，另一种方法是在构建压缩后的历史记录时，自行从保留的助手轮次中移除 `thinking` 和 `redacted_thinking` 块。

#### 后台（异步）压缩

后台压缩在对话继续进行的同时，在关键路径之外构建摘要，然后在几个请求之后将其换入。要保留在此期间生成的思考，请让 API 通过按需压缩编写摘要：[后台压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-background)提供了具体步骤，这些思考在与保留的最近轮次相同的[保留思考的条件](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)下保持有效。

您自己构建的摘要会以与保留尾部压缩相同的方式违反规则，只是有所延迟：在构建摘要期间生成的每个助手轮次所带的思考都早于替换，一旦摘要被替换进来，这些思考就会全部失败。如果您使用自己构建的摘要，请像对待保留尾部压缩一样处理替换，从替换开始发送 `"drop_block"`，或者改为同步压缩。

#### 不适用于保留思考的模式

* **从中间删除轮次。** 删除单个轮次会使其后的所有思考块失效，任何压缩方案都无法避免这一点。如果您删除某个轮次是为了更改指令，请改为追加一条[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#new-instructions)。要有选择地移除旧的工具结果或旧的思考，请使用服务器端[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)。
* **在工具调用回合中途进行压缩。** 不要在助手轮次的 `tool_use` 与回应它的 `tool_result` 之间进行压缩。请将该助手轮次连同其完整的思考一起回传，以便模型基于其推理完成该回合。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。

### 通过 ID 引用文件，而不是通过内容会变化的 URL

对于带有 `url` 来源的 `image` 或 `document` 块，检查针对的是获取到的字节，而不是 URL 字符串。内容会变化的 URL 会使后续的思考失效：例如"最新截图"端点，或有人在轮次之间编辑的文档。指向同一文件的轮换签名 URL 则不会。对于您在多个轮次中引用的内容，请使用 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传一次并使用 `file_id`，或者发送 base64。

### 库、代理和网关

库、代理或网关位于他人的历史记录与 API 之间，因此它自身所做的重写都算作编辑，而其用户既看不到也无法修复这些编辑。

* **原样传递您无法识别的内容。** 原样转发调用方的 `anthropic-beta` 值和 `thinking.block_binding`，并将 `input_transformations` 返回给调用方。拒绝未知键的选项模式会阻止您的用户选择 `"drop_block"`。
* **将 `role: "system"` 消息保留在调用方放置的位置。** 将其移入顶层 `system` 字段会更改该请求的 `system`，并使对话中的所有思考块失效。
* **要为某个请求关闭工具使用，请发送 `tool_choice: {"type": "none"}`。** 不要移除 `tools`。
* **不要隐藏 400 错误。** 如果您的代码捕获了该错误、移除了思考并代表调用方重试，请记录这一操作：调用方的历史记录仍然被编辑过，并且模型在之后的每个请求中都会丢失其先前的推理。

## 常见问题

<AccordionGroup>
  <Accordion title="我需要新账户来测试保留的思考吗？">
    不需要。发送 `thinking-binding-controls-2026-08-01` beta 标头并设置 `thinking.block_binding.prefix_mismatch_behavior`。设置该字段会使该请求选择启用强制检查，无论账户创建时间长短。`"error"` 会以与新账户相同的 400 错误拒绝经过编辑的历史，而 `"drop_block"` 会让请求通过，并在 `input_transformations` 中列出被丢弃的内容。要在不强制执行检查的情况下找出较早账户中的前缀编辑，请发送该标头并保持该字段未设置：未通过检查的块仍会到达模型，`input_transformations` 会将它们列为 `thinking_mismatch_allowed`。请参阅[检查您的代码是否编辑了前缀](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#how-to-tell-whether-your-integration-is-impacted)。
  </Accordion>

  <Accordion title="如果思考块之前的任何内容发生变化，哪怕只是一个工具描述，对话就无法使用了吗？">
    不会。失败的是历史记录中位于您所更改位置之后的已有思考，而您可以选择如何处理它们。使用 `prefix_mismatch_behavior: "drop_block"` 时，API 会丢弃这些块，请求会成功：模型在没有这些推理的情况下回答该轮次，提示缓存从编辑处重新开始。使用默认的 `"error"` 时，API 会以 400 错误拒绝请求，直到您撤销编辑或使用 `"drop_block"` 重新发送。请参阅[API 如何处理无效块](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#mismatch-behavior)。[哪些内容算作编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#what-counts-as-an-edit)列出了哪些更改会产生影响。
  </Accordion>

  <Accordion title="在请求之间更改 effort 或其他思考设置会使先前的思考失效吗？">
    不会。`output_config.effort`、`max_tokens` 和 `thinking` 配置不属于被检查的前缀，该前缀仅涵盖 `system`、`tools` 和 `messages`。顶层 effort 的更改会使大部分提示缓存失效。在 Claude Fable 5.1 上，[按消息设置的 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#effort-changes) 更改会保留提示缓存，并作为新的 effort 级别使用，直到再次更改为止。
  </Accordion>

  <Accordion title="我的工具列表会在会话中途发生变化。如何避免使对话失效？">
    不要编辑 `tools`。在会话开始时声明完整的工具集，用 `defer_loading: true` 标记尚不可用的工具，并通过 `tool_addition` 和 `tool_removal` 块提供或撤回它们。如果您只在会话中途才得知某个工具的模式，请在 `tool_addition` 块中定义它（使用 `inline-tools-2026-09-15`，对于 API 的 MCP 连接器所访问的服务器，还需加上 `mcp-client-2026-09-15`），并保持 `tools` 不变。承载这些块的 `role: "system"` 消息会成为后续思考的前缀的一部分，因此之后不要移动、改写或删除它们。请参阅[使用 `tool_addition` 和 `tool_removal` 添加或移除工具](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#tool-changes)。
  </Accordion>

  <Accordion title="我通过总结较早的轮次并原样保留最近的轮次来进行压缩。这样还可行吗？">
    可以，前提是由 API 编写摘要。[按需压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand)（beta 标头 `compact-2026-09-04`）会将较早的轮次总结为一个签名块，您发送该块来替换这些轮次。最近的轮次在[保留思考的条件](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)下保留其思考。

    如果您自己编写摘要，保留轮次的思考将无法通过检查，因为这些块是基于您已替换掉的历史记录生成的。请从您保留下来的轮次中移除 `thinking` 和 `redacted_thinking` 块，并保留其 `text` 和 `tool_use` 块，或者发送 `prefix_mismatch_behavior: "drop_block"`，让 API 丢弃它们。简单压缩不会留下任何可能失败的思考，是推荐的方法：一条摘要消息加上下一个用户轮次，不重放任何先前的轮次。服务器端[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)和[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)不算作编辑。请参阅[在客户端进行压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#custom-compaction-on-the-client)。
  </Accordion>

  <Accordion title="如何处理会在会话中途发生变化的指令文件，例如 AGENTS.md 或 CLAUDE.md？">
    在会话开始时加载一次，并保持顶层 `system` 提示和 `tools` 固定不变。当文件发生变化时，在 `messages` 中的当前位置追加新版本，而不是编辑原始内容。对于来自您（作为运营方）的指令，请使用[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)。对于您视为不可信、不应具有系统提示权限的文件文本，请改为将内容放入下一个 `user` 轮次。请参阅[使用对话中途系统消息添加指令](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#new-instructions)和[限制](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)。
  </Accordion>

  <Accordion title="我可以在重启后或第二天恢复已保存的会话吗？">
    可以。恢复的会话就是一个普通的后续请求：`system`、`tools` 和先前的 `messages` 的内容必须与您上次发送的内容相同。JSON 格式和键的顺序无关紧要，值才重要。请完整保存您发送和接收的内容，并原样重放：渲染后的系统提示、工具定义，以及按返回原样保存的每个助手轮次。不要根据此后可能已发生变化的输入重新渲染，例如日期、更新后的指令文件或新的工具版本。任何新内容都应放在追加的消息中。请参阅[按返回原样回传助手轮次](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#append-assistant-turns-exactly-as-returned)。
  </Accordion>

  <Accordion title="一个已保存的会话现在每个请求都失败。如何让它恢复正常？">
    存储的历史记录中包含编辑，因此重放它不可能成功。从现在起，请使用 `prefix_mismatch_behavior: "drop_block"` 发送该会话，或者一次性移除其 `thinking` 和 `redacted_thinking` 块后继续。只要其之前的内容不再发生变化，模型从那时起生成的思考就会保持有效。然后找出该编辑，以免新会话再遇到同样的问题。请参阅[在代码中处理错误](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#handle-the-error-in-code)。
  </Accordion>

  <Accordion title="我的框架可以将某个轮次路由到非 Claude 模型。这些轮次会使 Claude 先前的思考失效吗？">
    不会，前提是这些轮次追加在现有历史记录之后，且之前的内容没有任何变化：不含思考块的助手消息与其他追加的消息一样。请将另一个模型的输出作为 `text` 和 `tool_use` 内容发送。
  </Accordion>

  <Accordion title="我可以将一个对话的推理带入新的对话吗？">
    不能带入另一个对话。思考块只有在紧跟其生成时所基于的完全相同的 `system`、`tools` 和 `messages` 时才可用。在分叉点之前原样重放该历史记录的分支会保留其思考。从其他任何内容开始的对话都无法使用它，因此请像[简单压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#custom-compaction-on-the-client)那样，从任务状态的摘要开始该对话：目标、已做出的决定、目前为止的文件和结果，以及下一步。
  </Accordion>
</AccordionGroup>

## 后续步骤

<CardGroup cols={2}>
  <Card title="思考故障排除" icon="hammer" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting">
    诊断并修复最常见的思考故障：配置 400 错误、空的或缺失的思考块、max\_tokens 停止以及缓存未命中。
  </Card>

  <Card title="对话中途系统消息和工具变更" icon="messages" href="https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages">
    在对话进行到一半时更改系统指令或工具可用性，而不会使其之前的已缓存前缀失效。
  </Card>

  <Card title="压缩" icon="stack" href="https://platform.claude.com/docs/zh-CN/build-with-claude/compaction">
    服务器端上下文压缩，用于管理接近上下文窗口限制的长对话。
  </Card>

  <Card title="提示缓存" icon="database" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching">
    使用 `cache_control` 缓存提示前缀以降低成本和延迟，可使用自动缓存或带有 5 分钟或 1 小时 TTL 的显式断点。
  </Card>
</CardGroup>
