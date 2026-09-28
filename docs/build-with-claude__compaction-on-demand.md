---
title: 按需压缩
url: https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand
description: 在您的应用程序选择的时机让 Claude 总结对话，然后从摘要继续。
featureMetadata:
  status: beta
  betaHeader: compact-2026-09-04
  supportedModels:
    - claude-fable-5-1
    - claude-mythos-5-1
    - claude-fable-5
    - claude-mythos-5
    - claude-mythos-preview
    - claude-opus-5-5
    - claude-opus-5
    - claude-opus-4-8
    - claude-opus-4-7
    - claude-opus-4-6
    - claude-sonnet-5
    - claude-sonnet-4-6
  supportedPlatforms:
    Claude API: beta
    Claude Platform on AWS: beta
    Amazon Bedrock: not available
    Google Cloud: beta
    Microsoft Foundry: beta
---

借助 "on-demand compaction"（按需压缩），由您的应用程序决定何时总结对话：您发送一个带有 `compaction` 参数的请求，Claude 会返回一份摘要来代替回复。

## 按需压缩的工作原理

"Compaction"（压缩）请求独立于您的对话轮次。您将当前状态的对话连同 `compaction` 参数一起发送，响应中包含单个 `compaction` 块。该块包含可供您阅读的文本形式的摘要以及一个签名。在后续请求中请原样发送该块。

从那时起，该块将取代它所总结的消息。它位于 `messages` 的最前面，被总结的消息会被移除，您的下一轮紧随其后。Claude 会在这些消息原来的位置看到摘要。

![On-demand compaction（按需压缩）：一个携带四条消息和 compaction 参数的请求返回一个 compaction 块且没有回复；在下一个请求中，该块位于 messages 的最前面，取代这四条消息，后面跟着下一个用户轮次](https://platform.claude.com/docs/images/compaction-on-demand-swap.svg)

## 请求摘要

在请求摘要的请求上，以及之后每个携带已签名块的请求上，都要发送 `compact-2026-09-04` beta 标头。要检查某个模型是否支持按需压缩，请使用该 beta 标头调用 [Models API](https://platform.claude.com/docs/zh-CN/api/beta/models/list)，并读取每个模型的 `capabilities.compaction`。您不能在同一个请求中同时使用 `compaction` 和 `context_management`。

将当前状态的对话与 `"compaction": {"type": "summarize"}` 一起发送。API 会对请求中的每条消息进行一次总结，之后不生成回复，并单独返回该块，`stop_reason` 为 `"compaction"`。请发送与对话其余部分相同的 `system` 提示和 `tools`。总结器会读取它们；如果您在支持 [preserved thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)（保留思考）的模型上保留块之后的轮次，那么只有当 `system` 和 `tools` 一致时，这些轮次中的思考才保持有效。本示例中的对话没有 `system` 提示或工具，因此请求两者都不发送：

<CodeGroup>
  ```bash cURL
  # max_tokens 限制整个调用的令牌总数（含思考部分），因此请预留数千个令牌。
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: compact-2026-09-04" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5-5",
      "max_tokens": 4096,
      "messages": [
        {"role": "user", "content": "I am building a recipe app. Help me name the main entities in the data model."},
        {"role": "assistant", "content": "Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe."},
        {"role": "user", "content": "Good. Now suggest field names for Recipe."}
      ],
      "compaction": {"type": "summarize"}
    }'
  ```

  ```bash CLI
  ant beta:messages create --beta compact-2026-09-04 <<'YAML'
  model: claude-opus-5-5
  # max_tokens 限制整个调用（包括任何思考内容）的令牌数，因此请预留数千个令牌。
  max_tokens: 4096
  messages:
    - role: user
      content: I am building a recipe app. Help me name the main entities in the data model.
    - role: assistant
      content: Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe.
    - role: user
      content: Good. Now suggest field names for Recipe.
  compaction:
    type: summarize
  YAML
  ```

  ```python Python
  from anthropic.types.beta import BetaMessageParam

  client = anthropic.Anthropic()

  history: list[BetaMessageParam] = [
      {
          "role": "user",
          "content": "I am building a recipe app. Help me name the main entities in the data model.",
      },
      {
          "role": "assistant",
          "content": "Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe.",
      },
      {"role": "user", "content": "Good. Now suggest field names for Recipe."},
  ]

  response = client.beta.messages.create(
      model="claude-opus-5-5",
      # max_tokens 限制整个调用（包括任何思考内容）的令牌数，因此请预留数千个令牌。
      max_tokens=4096,
      betas=["compact-2026-09-04"],
      messages=history,
      compaction={"type": "summarize"},
  )
  print(f"Stop reason: {response.stop_reason}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const history: Anthropic.Beta.Messages.BetaMessageParam[] = [
    {
      role: "user",
      content: "I am building a recipe app. Help me name the main entities in the data model."
    },
    {
      role: "assistant",
      content:
        "Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe."
    },
    { role: "user", content: "Good. Now suggest field names for Recipe." }
  ];

  const response = await client.beta.messages.create({
    model: "claude-opus-5-5",
    // max_tokens 限制整个调用的令牌总数（包括思考部分），因此请预留数千个令牌。
    max_tokens: 4096,
    betas: ["compact-2026-09-04"],
    messages: history,
    compaction: { type: "summarize" }
  });
  console.log(`Stop reason: ${response.stop_reason}`);
  ```

  ```csharp C#
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Model = Anthropic.Models.Messages.Model;

  AnthropicClient client = new();

  List<BetaMessageParam> history =
  [
      new()
      {
          Role = Role.User,
          Content = "I am building a recipe app. Help me name the main entities in the data model.",
      },
      new()
      {
          Role = Role.Assistant,
          Content = "Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe.",
      },
      new() { Role = Role.User, Content = "Good. Now suggest field names for Recipe." },
  ];

  var response = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = Model.ClaudeOpus5_5,
      // max_tokens 限制整个调用（包括任何思考内容）的令牌数，因此请预留数千个令牌。
      MaxTokens = 4096,
      Betas = [AnthropicBeta.Compact2026_09_04],
      Messages = history,
      Compaction = new BetaCompactionConfig(), // type defaults to "summarize"
  });

  Console.WriteLine($"Stop reason: {response.StopReason?.Raw()}");
  ```

  ```go Go
  client := anthropic.NewClient()

  history := []anthropic.BetaMessageParam{
  	anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("I am building a recipe app. Help me name the main entities in the data model.")),
  	{
  		Role:    anthropic.BetaMessageParamRoleAssistant,
  		Content: []anthropic.BetaContentBlockParamUnion{anthropic.NewBetaTextBlock("Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe.")},
  	},
  	anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Good. Now suggest field names for Recipe.")),
  }

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model: anthropic.ModelClaudeOpus5_5,
  	// max_tokens 限制整个调用的令牌总数（包括思考部分），因此请预留数千个令牌。
  	MaxTokens: 4096,
  	Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaCompact2026_09_04},
  	Messages:  history,
  	Compaction: anthropic.BetaCompactionConfigUnionParam{
  		OfSummarize: &anthropic.BetaSummarizeCompactionParam{},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println("Stop reason:", response.StopReason)
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaCompactionConfig;
  import com.anthropic.models.beta.messages.MessageCreateParams;

  void main() {
      var client = AnthropicOkHttpClient.fromEnv();

      var params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5_5)
          // max_tokens 限制整个调用（包括任何思考内容）的令牌数，因此请预留数千个令牌。
          .maxTokens(4096)
          .addBeta(AnthropicBeta.COMPACT_2026_09_04)
          .addUserMessage("I am building a recipe app. Help me name the main entities in the data model.")
          .addAssistantMessage("Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe.")
          .addUserMessage("Good. Now suggest field names for Recipe.")
          .compaction(BetaCompactionConfig.builder().build()) // type defaults to "summarize"
          .build();

      var response = client.beta().messages().create(params);
      response.stopReason().ifPresent(reason -> IO.println("Stop reason: " + reason));
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaCompactionConfig;
  use Anthropic\Beta\Messages\BetaMessageParam;
  use Anthropic\Beta\Messages\BetaMessageParam\Role;

  $client = new Client();

  $history = [
      BetaMessageParam::with(
          role: Role::USER,
          content: 'I am building a recipe app. Help me name the main entities in the data model.',
      ),
      BetaMessageParam::with(
          role: Role::ASSISTANT,
          content: 'Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe.',
      ),
      BetaMessageParam::with(role: Role::USER, content: 'Good. Now suggest field names for Recipe.'),
  ];

  $response = $client->beta->messages->create(
      model: Model::CLAUDE_OPUS_5_5,
      // max_tokens 限制整个调用的令牌数（包括任何思考内容），因此请预留数千个令牌。
      maxTokens: 4096,
      betas: [AnthropicBeta::COMPACT_2026_09_04],
      messages: $history,
      compaction: BetaCompactionConfig::with(), // type defaults to 'summarize'
  );

  echo "Stop reason: {$response->stopReason}", PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  history = [
    {
      role: "user",
      content: "I am building a recipe app. Help me name the main entities in the data model."
    },
    {
      role: "assistant",
      content: "Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe."
    },
    { role: "user", content: "Good. Now suggest field names for Recipe." }
  ]

  response = client.beta.messages.create(
    model: Anthropic::Model::CLAUDE_OPUS_5_5,
    # max_tokens 限制整个调用（包括任何思考内容）的令牌数，因此请预留数千个令牌。
    max_tokens: 4096,
    betas: [Anthropic::AnthropicBeta::COMPACT_2026_09_04],
    messages: history,
    compaction: { type: "summarize" }
  )
  puts "Stop reason: #{response.stop_reason}"
  ```
</CodeGroup>

```json Response
{
  "id": "msg_013Zva2CMHLNnXjNJJKqJ2EF",
  "type": "message",
  "role": "assistant",
  "model": "claude-opus-5-5",
  "content": [
    {
      "type": "compaction",
      "content": "Summary of the conversation: the user is designing the data model for a recipe app. The entities agreed so far are Recipe, Ingredient, Step, and RecipeIngredient, which holds the quantity and unit. The user then asked for field names for Recipe.",
      "signature": "EuYBCkQY..."
    }
  ],
  "stop_reason": "compaction",
  "usage": {
    "input_tokens": 0,
    "output_tokens": 0,
    "iterations": [{ "type": "compaction", "input_tokens": 144, "output_tokens": 276 }]
  }
}
```

总结调用使用请求的模型、`system`、`tools`、思考设置和 `max_tokens`。总结器会读取工具定义，但从不运行工具，且响应中不包含思考内容。`max_tokens` 限制的是整个调用，包括模型在编写摘要之前进行的任何思考，因此请预留数千个令牌。[统计压缩用量](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#understanding-usage)说明了该调用的计费方式。

如果最后一个 `assistant` 轮次以尚无结果的工具调用结束，API 会拒绝该请求。请先发送该轮次的工具结果。此外，请不要包含 `stop_sequences`、结构化输出的 `output_config.format`，以及类型为 `any` 或 `tool` 的 `tool_choice`。它们在总结调用中不起任何作用，API 会拒绝它们。对话仍必须能放入模型的 "context window"（上下文窗口），因此请在超出之前进行压缩，而不是之后。

当您对响应进行 "streaming"（流式传输）时，该块会完整到达。您会收到一个携带完整块的 `content_block_start` 事件，然后是 `content_block_stop`，没有 `content_block_delta` 事件。`ping` 事件可能在它们之前或之间到达。

## 从摘要继续

在您的历史记录中，用返回的助手消息替换您发送的消息。保持 `compaction` 块与 API 返回时完全一致，包括其 `signature`。在您发送压缩请求之后进行的任何轮次都原样跟在该块之后，这正是[后台压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-background)所依赖的基础。在之后的每个请求中都将该块放在最前面发送，并附带 beta 标头：

```json
{
  "model": "claude-opus-5-5",
  "max_tokens": 2048,
  "messages": [
    {
      "role": "assistant",
      "content": [
        {
          "type": "compaction",
          "content": "Summary of the conversation: the user is designing the data model for a recipe app. The entities agreed so far are Recipe, Ingredient, Step, and RecipeIngredient, which holds the quantity and unit. The user then asked for field names for Recipe.",
          "signature": "EuYBCkQY..."
        }
      ]
    },
    {
      "role": "assistant",
      "content": "For Recipe, use title, description, servings, prep_minutes, and cook_minutes. Add created_at and updated_at timestamps."
    },
    { "role": "user", "content": "Now do the same for Ingredient." }
  ]
}
```

本示例延续了上面的请求示例，该示例以 `user` 轮次结束；图示展示的是更简单的情况，即在编写摘要期间没有进行任何轮次。这里第二条 `assistant` 消息是对最后一个被总结的 `user` 轮次的回复。它是在编写摘要期间到达的，因此不在被总结的消息之中。这里连续出现两条 `assistant` 消息没有问题，因为该块仍然位于最前面。

API 会将摘要放在该块所在的位置，并将之后的每条消息原样传递给 Claude。请遵循以下规则：

* 将该块放在 `messages` 的最前面，可以作为单独的 `assistant` 消息，也可以作为第一条消息的第一个内容块，无论该消息是 `user` 还是 `assistant` 消息。
* 移除被总结的消息。如果有任何被总结的消息留在该块之前，请求会返回 400 错误（`compaction_block_misplaced`）。
* 每个请求恰好发送一个 `compaction` 块，之后的每个请求都要发送。

<Warning>
  替换过程中有两种错误不会引发报错。如果被总结的消息仍留在该块之后，API 会再次将它们发送给 Claude。如果之后的请求遗漏了该块，Claude 将得不到任何摘要。
</Warning>

阈值压缩的工作方式正好相反：它的块跟在所总结的消息之后，并且 API 会为您丢弃这些消息。请参阅[回传压缩块](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold#passing-compaction-blocks-back)。

在 Python 中，请像本页示例那样使用 `client.beta.messages`。如果您调用 `client.messages` 并自行序列化块，请使用 `to_dict()` 或 `model_dump(exclude_none=True)`：普通的 `model_dump()` 会向块中添加 `citations: null` 和 `text: null`，API 会拒绝这样的块。

如果您保留块之后的轮次并将其思考块传回，保持这些思考有效的条件请参阅[压缩与保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks)。

### 再次压缩

要压缩一个已经以块开头的对话，请再次发送 `compaction`。新块会总结旧摘要及其之后的所有内容。从那时起，只发送最新的块。

## 在循环中压缩

每轮之后，循环会累加上一个响应的输入和输出令牌，因为下一个请求也会发送该回复。当该总数超过限制且仍有下一轮要进行时，循环会使用相同的模型和 `system` 提示发送压缩请求，检查 `stop_reason`，用返回的消息替换其历史记录，并打印它在哪一轮之前进行了压缩。示例中 2,500 个令牌的限制是故意设得很低的，以便短对话也能触发压缩。请将您的限制设置为接近实际的输入预算。

<CodeGroup exclude="shell">
  ```python Python
  from anthropic.types.beta import BetaMessageParam

  client = anthropic.Anthropic()

  # 请将此值设置为接近您实际的输入预算。此处设得较低，以便较短的对话也会触发压缩。
  COMPACT_AT_TOKENS = 2500
  SYSTEM = "You help design a recipe app's data model. Keep answers short."

  QUESTIONS = [
      "What are the main entities in the data model?",
      "Which fields should Recipe have?",
      "Which fields should Ingredient have?",
      "Which fields should RecipeIngredient have?",
      "Which fields should Step have?",
      "Which indexes should these tables have?",
      "Which fields should be required?",
      "Which fields should have default values?",
  ]

  history: list[BetaMessageParam] = []
  for turn, question in enumerate(QUESTIONS, start=1):
      history.append({"role": "user", "content": question})
      response = client.beta.messages.create(
          model="claude-opus-5-5",
          max_tokens=8192,
          system=SYSTEM,
          betas=["compact-2026-09-04"],
          messages=history,
      )
      history.append({"role": "assistant", "content": response.content})

      # 下一个请求也会发送此回复，因此需将其计入。
      conversation_tokens = response.usage.input_tokens + response.usage.output_tokens
      if conversation_tokens > COMPACT_AT_TOKENS and turn < len(QUESTIONS):
          summary = client.beta.messages.create(
              model="claude-opus-5-5",
              max_tokens=4096,
              system=SYSTEM,
              betas=["compact-2026-09-04"],
              messages=history,
              compaction={"type": "summarize"},
          )
          if summary.stop_reason == "compaction":
              history = [{"role": "assistant", "content": summary.content}]
              print(f"Compacted before turn {turn + 1}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // 请将此值设置为接近您实际的输入预算。此处设得较低，以便较短的对话也会触发压缩。
  const compactAtTokens = 2500;
  const systemPrompt = "You help design a recipe app's data model. Keep answers short.";

  const questions = [
    "What are the main entities in the data model?",
    "Which fields should Recipe have?",
    "Which fields should Ingredient have?",
    "Which fields should RecipeIngredient have?",
    "Which fields should Step have?",
    "Which indexes should these tables have?",
    "Which fields should be required?",
    "Which fields should have default values?"
  ];

  let history: Anthropic.Beta.Messages.BetaMessageParam[] = [];
  for (const [index, question] of questions.entries()) {
    const turn = index + 1;
    history.push({ role: "user", content: question });
    const response = await client.beta.messages.create({
      model: "claude-opus-5-5",
      max_tokens: 8192,
      system: systemPrompt,
      betas: ["compact-2026-09-04"],
      messages: history
    });
    history.push({ role: "assistant", content: response.content });

    // 下一个请求也会发送此回复，因此需将其计入。
    const conversationTokens = response.usage.input_tokens + response.usage.output_tokens;
    if (conversationTokens > compactAtTokens && turn < questions.length) {
      const summary = await client.beta.messages.create({
        model: "claude-opus-5-5",
        max_tokens: 4096,
        system: systemPrompt,
        betas: ["compact-2026-09-04"],
        messages: history,
        compaction: { type: "summarize" }
      });
      if (summary.stop_reason === "compaction") {
        history = [{ role: "assistant", content: summary.content }];
        console.log(`Compacted before turn ${turn + 1}`);
      }
    }
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Model = Anthropic.Models.Messages.Model;

  AnthropicClient client = new();

  // 请将此值设置为接近您实际的输入预算。此处设得较低，以便较短的对话也会触发压缩。
  const int CompactAtTokens = 2500;
  const string SystemPrompt = "You help design a recipe app's data model. Keep answers short.";

  string[] questions =
  [
      "What are the main entities in the data model?",
      "Which fields should Recipe have?",
      "Which fields should Ingredient have?",
      "Which fields should RecipeIngredient have?",
      "Which fields should Step have?",
      "Which indexes should these tables have?",
      "Which fields should be required?",
      "Which fields should have default values?",
  ];

  List<BetaMessageParam> history = [];
  foreach (var (index, question) in questions.Index())
  {
      var turn = index + 1;
      history.Add(new() { Role = Role.User, Content = question });
      var response = await client.Beta.Messages.Create(new MessageCreateParams
      {
          Model = Model.ClaudeOpus5_5,
          MaxTokens = 8192,
          System = SystemPrompt,
          Betas = [AnthropicBeta.Compact2026_09_04],
          Messages = history,
      });
      history.Add(new()
      {
          Role = Role.Assistant,
          Content = response.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
      });

      // 下一个请求也会发送此回复，因此需将其计入。
      var conversationTokens = response.Usage.InputTokens + response.Usage.OutputTokens;
      if (conversationTokens > CompactAtTokens && turn < questions.Length)
      {
          var summary = await client.Beta.Messages.Create(new MessageCreateParams
          {
              Model = Model.ClaudeOpus5_5,
              MaxTokens = 4096,
              System = SystemPrompt,
              Betas = [AnthropicBeta.Compact2026_09_04],
              Messages = history,
              Compaction = new BetaCompactionConfig(), // type defaults to "summarize"
          });
          if (summary.StopReason == BetaStopReason.Compaction)
          {
              history =
              [
                  new()
                  {
                      Role = Role.Assistant,
                      Content = summary.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
                  },
              ];
              Console.WriteLine($"Compacted before turn {turn + 1}");
          }
      }
  }
  ```

  ```go Go
  ctx := context.Background()
  client := anthropic.NewClient()

  // 请将此值设置为接近您实际的输入预算。此处设得较低，以便较短的对话也会触发压缩。
  const compactAtTokens = 2500
  system := []anthropic.BetaTextBlockParam{{Text: "You help design a recipe app's data model. Keep answers short."}}

  questions := []string{
  	"What are the main entities in the data model?",
  	"Which fields should Recipe have?",
  	"Which fields should Ingredient have?",
  	"Which fields should RecipeIngredient have?",
  	"Which fields should Step have?",
  	"Which indexes should these tables have?",
  	"Which fields should be required?",
  	"Which fields should have default values?",
  }

  var history []anthropic.BetaMessageParam
  for i, question := range questions {
  	turn := i + 1
  	history = append(history, anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock(question)))
  	response, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  		Model:     anthropic.ModelClaudeOpus5_5,
  		MaxTokens: 8192,
  		System:    system,
  		Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaCompact2026_09_04},
  		Messages:  history,
  	})
  	if err != nil {
  		log.Fatal(err)
  	}
  	history = append(history, response.ToParam())

  	// 下一个请求也会发送此回复，因此需将其计入。
  	conversationTokens := response.Usage.InputTokens + response.Usage.OutputTokens
  	if conversationTokens > compactAtTokens && turn < len(questions) {
  		summary, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  			Model:     anthropic.ModelClaudeOpus5_5,
  			MaxTokens: 4096,
  			System:    system,
  			Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaCompact2026_09_04},
  			Messages:  history,
  			Compaction: anthropic.BetaCompactionConfigUnionParam{
  				OfSummarize: &anthropic.BetaSummarizeCompactionParam{},
  			},
  		})
  		if err != nil {
  			log.Fatal(err)
  		}
  		if summary.StopReason == anthropic.BetaStopReasonCompaction {
  			history = []anthropic.BetaMessageParam{summary.ToParam()}
  			fmt.Printf("Compacted before turn %d\n", turn+1)
  		}
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaCompactionConfig;
  import com.anthropic.models.beta.messages.BetaMessageParam;
  import com.anthropic.models.beta.messages.BetaStopReason;
  import com.anthropic.models.beta.messages.MessageCreateParams;

  // 请将此值设置为接近您实际的输入预算。此处设得较低，以便简短的对话也会触发压缩。
  static final long COMPACT_AT_TOKENS = 2500;
  static final String SYSTEM = "You help design a recipe app's data model. Keep answers short.";

  void main() {
      var client = AnthropicOkHttpClient.fromEnv();

      var questions = List.of(
          "What are the main entities in the data model?",
          "Which fields should Recipe have?",
          "Which fields should Ingredient have?",
          "Which fields should RecipeIngredient have?",
          "Which fields should Step have?",
          "Which indexes should these tables have?",
          "Which fields should be required?",
          "Which fields should have default values?"
      );

      var history = new ArrayList<BetaMessageParam>();
      for (int turn = 1; turn <= questions.size(); turn++) {
          history.add(BetaMessageParam.builder()
              .role(BetaMessageParam.Role.USER)
              .content(questions.get(turn - 1))
              .build());
          var params = MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5_5)
              .maxTokens(8192)
              .system(SYSTEM)
              .addBeta(AnthropicBeta.COMPACT_2026_09_04)
              .messages(history)
              .build();
          var response = client.beta().messages().create(params);
          history.add(response.toParam());

          // 下一个请求也会发送此回复，因此需将其计入。
          long conversationTokens = response.usage().inputTokens() + response.usage().outputTokens();
          if (conversationTokens > COMPACT_AT_TOKENS && turn < questions.size()) {
              var summaryParams = MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5_5)
                  .maxTokens(4096)
                  .system(SYSTEM)
                  .addBeta(AnthropicBeta.COMPACT_2026_09_04)
                  .messages(history)
                  .compaction(BetaCompactionConfig.builder().build()) // type defaults to "summarize"
                  .build();
              var summary = client.beta().messages().create(summaryParams);
              if (summary.stopReason().map(BetaStopReason.COMPACTION::equals).orElse(false)) {
                  history.clear();
                  history.add(summary.toParam());
                  IO.println("Compacted before turn " + (turn + 1));
              }
          }
      }
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaCompactionConfig;
  use Anthropic\Beta\Messages\BetaMessageParam;
  use Anthropic\Beta\Messages\BetaMessageParam\Role;
  use Anthropic\Beta\Messages\BetaStopReason;

  $client = new Client();

  // 请将此值设置为接近您实际的输入预算。此处设得较低，以便简短的对话也会触发压缩。
  const COMPACT_AT_TOKENS = 2500;
  const SYSTEM = "You help design a recipe app's data model. Keep answers short.";

  $questions = [
      'What are the main entities in the data model?',
      'Which fields should Recipe have?',
      'Which fields should Ingredient have?',
      'Which fields should RecipeIngredient have?',
      'Which fields should Step have?',
      'Which indexes should these tables have?',
      'Which fields should be required?',
      'Which fields should have default values?',
  ];

  $history = [];
  foreach ($questions as $index => $question) {
      $turn = $index + 1;
      $history[] = BetaMessageParam::with(role: Role::USER, content: $question);
      $response = $client->beta->messages->create(
          model: Model::CLAUDE_OPUS_5_5,
          maxTokens: 8192,
          system: SYSTEM,
          betas: [AnthropicBeta::COMPACT_2026_09_04],
          messages: $history,
      );
      $history[] = BetaMessageParam::with(role: Role::ASSISTANT, content: $response->content);

      // 下一个请求也会发送此回复，因此需将其计入。
      $conversationTokens = $response->usage->inputTokens + $response->usage->outputTokens;
      if ($conversationTokens > COMPACT_AT_TOKENS && $turn < count($questions)) {
          $summary = $client->beta->messages->create(
              model: Model::CLAUDE_OPUS_5_5,
              maxTokens: 4096,
              system: SYSTEM,
              betas: [AnthropicBeta::COMPACT_2026_09_04],
              messages: $history,
              compaction: new BetaCompactionConfig(), // type defaults to 'summarize'
          );
          if ($summary->stopReason === BetaStopReason::COMPACTION->value) {
              $history = [BetaMessageParam::with(role: Role::ASSISTANT, content: $summary->content)];
              printf("Compacted before turn %d\n", $turn + 1);
          }
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 请将此值设置为接近您实际的输入预算。此处设得较低，以便较短的对话也会触发压缩。
  COMPACT_AT_TOKENS = 2500
  SYSTEM = "You help design a recipe app's data model. Keep answers short."

  questions = [
    "What are the main entities in the data model?",
    "Which fields should Recipe have?",
    "Which fields should Ingredient have?",
    "Which fields should RecipeIngredient have?",
    "Which fields should Step have?",
    "Which indexes should these tables have?",
    "Which fields should be required?",
    "Which fields should have default values?"
  ]

  history = []
  questions.each.with_index(1) do |question, turn|
    history << { role: "user", content: question }
    response = client.beta.messages.create(
      model: Anthropic::Model::CLAUDE_OPUS_5_5,
      max_tokens: 8192,
      system_: SYSTEM,
      betas: [Anthropic::AnthropicBeta::COMPACT_2026_09_04],
      messages: history
    )
    history << { role: "assistant", content: response.content }

    # 下一个请求也会发送此回复，因此需将其计入。
    conversation_tokens = response.usage.input_tokens + response.usage.output_tokens
    if conversation_tokens > COMPACT_AT_TOKENS && turn < questions.length
      summary = client.beta.messages.create(
        model: Anthropic::Model::CLAUDE_OPUS_5_5,
        max_tokens: 4096,
        system_: SYSTEM,
        betas: [Anthropic::AnthropicBeta::COMPACT_2026_09_04],
        messages: history,
        compaction: { type: "summarize" }
      )
      if summary.stop_reason == :compaction
        history = [{ role: "assistant", content: summary.content }]
        puts "Compacted before turn #{turn + 1}"
      end
    end
  end
  ```
</CodeGroup>

对 `stop_reason` 的检查位于代码查找该块之前；[处理缺失的摘要或错误](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#when-no-summary-comes-back)说明了原因。历史记录是被替换，而不是被追加：返回的消息会按照[从摘要继续](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#continue-from-the-summary)中的规则，替换请求所携带的每条消息。当没有返回摘要时，循环会保留其历史记录，并在下一轮之后再次请求。

Python、TypeScript、C#、Go、Java、PHP 和 Ruby 中的 SDK ["tool runner"（工具运行器）](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner)可以为您发送压缩请求。当您决定压缩时，在运行器上调用 `compact_before_next_turn()` (typescript, java, php: `compactBeforeNextTurn()`; csharp, go: `CompactBeforeNextTurn()`)。当前轮次及其工具调用完成后，运行器会发送压缩请求，并用返回的消息替换其历史记录。请使用 `compact-2026-09-04` beta 创建运行器，因为运行器不会自动添加它。

运行器根据自身参数构建压缩请求，并省略 `context_management`。它还会省略 `stop_sequences`、类型为 `any` 或 `tool` 的 `tool_choice`，以及结构化输出的 `output_config.format`，API 会在压缩请求中拒绝这些参数。[请求摘要](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#request-a-summary)解释了原因。运行器会在之后的请求中再次发送它们。Python 1.8.0、TypeScript 0.128.0、C# 12.50.0、Go 1.75.0 和 Java 2.65.0 之前的 SDK 版本也会在压缩请求中发送它们。在这些版本上，设置了其中任何参数的运行器会收到 400 错误。运行器会原样发送[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)。如果 `output_config.task_budget` 设置了 `remaining`，压缩请求将返回 400 错误，因此请不要设置 `remaining`，正如[限制以及与其他功能的交互](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#how-it-fits-with-the-rest-of-the-api)所述。当运行器的 `context_management` 包含压缩编辑时，运行器会拒绝压缩，因此请在一个运行器上只使用一种压缩方式。

### 何时压缩

您可以在任何已完成的轮次之后发送压缩请求，因此由您的代码决定时机。

要估算下一个请求的大小，请像循环那样，将上一个响应 `usage` 中的 `input_tokens` 和 `output_tokens` 相加。使用 ["prompt caching"（提示缓存）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#tracking-cache-performance)时，`input_tokens` 只统计最后一个缓存断点之后的令牌，因此还要加上 `cache_read_input_tokens` 和 `cache_creation_input_tokens`。您也可以将相同的消息发送到 ["token counting"（令牌计数）](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点。

将该数字与您选择的限制进行比较，该限制应低于模型的[上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)。

## 编写您自己的总结提示

如果不提供 `instructions`，API 会使用其自身的总结提示。非空的 `instructions` 字符串（最多 16,384 个字符）会完全替换该提示。例如：

```json
{
  "compaction": {
    "type": "summarize",
    "instructions": "Summarize this recipe app design conversation. Preserve every entity and field name agreed so far, and the user's latest open request. Do not call tools; respond with the summary text only."
  }
}
```

无论是否提供 `instructions`，总结器都会读取整个对话，包括之前的思考。在您的 `instructions` 中，请说明摘要必须保留哪些内容，并告诉模型不要调用工具。总结调用与其他任何请求一样，在相同的安全保障措施下运行。

## 处理缺失的摘要或错误

只有当总结调用以文本正常结束且没有工具调用时，才会生成摘要。否则，响应仍然是 200，但 `content` 为空，因此请在查找该块之前检查 `stop_reason`。该调用仍会计费并在 `usage.iterations` 中报告；如果根本无法进行调用，则用量为零。`stop_reason` 是总结调用结束时的停止原因。在所有情况下，您都可以在没有摘要的情况下继续，并稍后再压缩。

| `stop_reason`                     | 原因               | 处理方法                                |
| --------------------------------- | ---------------- | ----------------------------------- |
| `"max_tokens"`                    | 摘要被截断。           | 使用更大的 `max_tokens` 重新发送。            |
| `"model_context_window_exceeded"` | 没有空间容纳总结提示。      | 使用更短的 `instructions` 或更少的消息重新发送。    |
| `"tool_use"`                      | 模型调用了工具，而不是编写摘要。 | 重新发送，并在 `instructions` 中告诉模型不要调用工具。 |
| `"refusal"`                       | 请求被拒绝。           | 在没有摘要的情况下继续。                        |
| `"end_turn"`                      | 调用没有返回文本。        | 在没有摘要的情况下继续。                        |

总结调用与您的其他请求一样受到相同的安全保障措施约束。出现 `"refusal"` 后，[`stop_details`](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons#refusal) 会标识其背后的策略类别。

### 错误

压缩请求或携带块的请求也可能直接失败。大多数 400 错误都带有一条消息，说明需要移除或重新发送的内容。有些还带有以 `compaction_` 开头的 `error.details.error_code`。参数错误（例如某个字段不能与 `compaction` 组合使用）只带有消息。

| 错误                                                                                      | 原因                                             | 处理方法                                                                                                                               |
| --------------------------------------------------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| 529 `overloaded_error`，`error.details.error_code` 为 `compaction_unavailable`            | 在生成块或读取您传回的块时出现了暂时性服务器问题。                      | 重试该请求。                                                                                                                             |
| 400 `compaction_block_misplaced`                                                        | 被总结的消息仍留在该块之前。                                 | 移除这些消息，使该块位于 `messages` 的最前面。                                                                                                      |
| 400 `compaction_signature_invalid` 或 `compaction_content_mismatch`                      | 块的 `signature` 或 `content` 在 API 返回后被修改。       | 完全按照返回的样子发送该块，包括其 `signature`。                                                                                                     |
| 400                                                                                     | 请求携带了多个 `compaction` 块。                        | 只发送一个，即最新的块。                                                                                                                       |
| 400                                                                                     | 最后一个 `assistant` 轮次以尚无结果的工具调用结束。               | 发送该轮次的工具结果，然后再压缩。                                                                                                                  |
| 400 `compaction_nothing_to_summarize`                                                   | `messages` 中没有 `user` 或 `assistant` 内容，例如列表为空。 | 至少发送一条 `user` 或 `assistant` 消息。                                                                                                    |
| 压缩请求返回 400，消息指出 `compaction` 参数 `requires anthropic-beta: compact-2026-09-04`           | 压缩请求遗漏了 beta 标头。                               | 添加 beta 标头；请参阅[请求摘要](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#request-a-summary)。             |
| 之后携带该块的请求返回 400：一个验证错误，指出 `compaction` 不是预期的内容块类型之一。该消息不会提及标头                           | 该请求遗漏了 beta 标头。                                | 在每个携带该块的请求中添加 beta 标头；请参阅[请求摘要](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#request-a-summary)。  |
| 400 验证错误，例如 `messages.0.content.0.compaction.citations: Extra inputs are not permitted` | 传回的块带有 API 未返回的字段，例如 `citations: null`。        | 完全按照返回的样子发送该块；请参阅[从摘要继续](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#continue-from-the-summary)。 |

## 统计压缩用量

总结调用与其他任何请求一样计费并受 "rate limit"（速率限制）约束，`usage.iterations` 将其报告为 `compaction` 条目。顶层的 `input_tokens` 和 `output_tokens` 为零，因为没有生成回复。要统计一个对话消耗的用量，请对 `usage.iterations` 求和，而不是使用顶层字段。在之后的请求中传回块不会增加压缩成本。

您现在有了一个能够压缩对话并处理缺失摘要的可用循环。有两个页面会改变它的运行方式，并且您可以将它们结合使用：[保留最近轮次的压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-keep-recent-turns)会逐字保留最后几轮，[后台压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-background)让对话在编写摘要期间继续进行。如果您传回思考块并采用其中任一方式，请参阅[压缩与保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks)。

## 限制以及与其他功能的交互

* **阈值压缩和上下文编辑。** 您不能在同一个请求中发送 `compaction` 和 `context_management`。阈值压缩（`compact_20260112`）不能在携带已签名块的请求上运行。
* **提示缓存。** 块上的 `cache_control` 会在摘要之后放置一个断点。
* **对话中途的系统消息和工具变更。** 被总结范围内的 `role: "system"` 消息也会被总结，因此一旦块替换了它们，其中的文本指令就不再生效。如果某条指令仍然重要，请在 `role: "system"` 消息中再次说明。在您下一个新的 `user` 轮次之后立即发送该消息，并从此将其保留在历史记录中。关于[工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes)，以及当您保留块之后的轮次时该消息应放在何处，请参阅[更改系统提示或工具](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#change-the-system-prompt-or-tools)。
* **任务预算。** 不要在带有 `compaction` 的请求中或携带块的请求上发送[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)的 `remaining` 值（`output_config.task_budget.remaining`）。这样做会返回 400 错误。
* **令牌计数。** [令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点会忽略 `compaction` 参数。
* **摘要无法承载的内容。** 被总结消息中的图像、文档、`container_upload` 块和获取的 URL，在块替换它们之后就会消失。请重新说明或重新上传后续轮次仍需要的任何内容。
