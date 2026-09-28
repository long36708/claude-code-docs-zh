---
title: 保留最近轮次的压缩
url: https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-keep-recent-turns
description: 使用按需压缩总结对话中较早的轮次，并在摘要之后逐字发送最近的轮次。
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

"Keep-tail compaction"（保留尾部压缩）会在摘要之后逐字保留对话的最后几个轮次。它改变了[压缩循环](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#compact-in-a-loop)中的两件事：哪些消息进入压缩请求，以及您在压缩块之后发送什么。[从摘要继续](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#continue-from-the-summary)中的所有内容均原样适用。

## 选择要保留的轮次

没有任何参数用于设置保留哪些轮次。您需要在历史记录中选择一个切分点：切分点之前的消息进入压缩请求，从切分点开始的消息则被保留。

保留的轮次会以完整长度发回给 Claude，因此您保留得越多，压缩释放的空间就越少。

请将切分点放在没有未完成工具调用的位置，使每个工具调用及其结果位于同一侧。如果您发送的消息以一个 `assistant` 轮次结尾，且该轮次中的工具调用尚无结果，API 会拒绝该压缩请求。

## 压缩较早的轮次，并在压缩块之后发送其余轮次

要逐字保留最近轮次组成的尾部，请将这些轮次排除在压缩请求之外。API 会总结发送给它的每一条消息，因此请只发送较早的轮次，然后将压缩块放在您保留的轮次前面。

请按照历史记录中的原样发送保留的轮次，包括思考块。两个请求都携带 beta 标头，如[请求摘要](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#request-a-summary)中所述。

在以下示例中，历史记录包含两个轮次，切分点保留第二个轮次。压缩请求携带第一个轮次：

```json
{
  "model": "claude-opus-5-5",
  "max_tokens": 4096,
  "messages": [
    {
      "role": "user",
      "content": "I am building a recipe app. Help me name the main entities in the data model."
    },
    {
      "role": "assistant",
      "content": "Start with Recipe, Ingredient, and Step. Add a RecipeIngredient entry that holds the quantity and unit for each ingredient in a recipe."
    }
  ],
  "compaction": { "type": "summarize" }
}
```

下一个请求首先发送返回的压缩块，然后是原样的保留轮次，最后是新的 `user` 消息。[从摘要继续](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#continue-from-the-summary)展示了一个以压缩块开头的请求。

以下程序是[在循环中压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#compact-in-a-loop)中的循环，经过修改以保留最后两个轮次。高亮显示的行表示它与原循环的不同之处。

<CodeGroup exclude="shell">
  ```python Python
  from anthropic.types.beta import BetaMessageParam

  client = anthropic.Anthropic()

  # 请将此值设为接近您实际的输入预算。此处设得较低，以便短对话也会触发压缩。
  COMPACT_AT_TOKENS = 2500
  SYSTEM = "You help design a recipe app's data model. Keep answers short."
  KEEP_TURNS = 2

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
      if conversation_tokens > COMPACT_AT_TOKENS and KEEP_TURNS < turn < len(QUESTIONS):
          # 一个轮次包含一条用户消息和一条助手回复，
          # 因此保留的轮次以用户消息开头。
          split = -2 * KEEP_TURNS
          older, recent = history[:split], history[split:]
          summary = client.beta.messages.create(
              model="claude-opus-5-5",
              max_tokens=4096,
              system=SYSTEM,
              betas=["compact-2026-09-04"],
              messages=older,
              compaction={"type": "summarize"},
          )
          if summary.stop_reason == "compaction":
              history = [{"role": "assistant", "content": summary.content}, *recent]
              print(f"Kept {len(recent) // 2} turns after the block")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // 请将此值设置为接近您实际的输入预算。此处设得较低，以便简短的对话也会触发压缩。
  const compactAtTokens = 2500;
  const systemPrompt = "You help design a recipe app's data model. Keep answers short.";
  const keepTurns = 2;

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
    if (conversationTokens > compactAtTokens && turn > keepTurns && turn < questions.length) {
      // 一个轮次包含一条用户消息和一条助手回复，因此保留的轮次以用户消息开头。
      const older = history.slice(0, -2 * keepTurns);
      const recent = history.slice(-2 * keepTurns);
      const summary = await client.beta.messages.create({
        model: "claude-opus-5-5",
        max_tokens: 4096,
        system: systemPrompt,
        betas: ["compact-2026-09-04"],
        messages: older,
        compaction: { type: "summarize" }
      });
      if (summary.stop_reason === "compaction") {
        history = [{ role: "assistant", content: summary.content }, ...recent];
        console.log(`Kept ${recent.length / 2} turns after the block`);
      }
    }
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Model = Anthropic.Models.Messages.Model;

  AnthropicClient client = new();

  // 请将此值设为接近您实际的输入预算。此处设得较低，以便简短的对话也会触发压缩。
  const int CompactAtTokens = 2500;
  const string SystemPrompt = "You help design a recipe app's data model. Keep answers short.";
  const int KeepTurns = 2;

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
      if (conversationTokens > CompactAtTokens && turn > KeepTurns && turn < questions.Length)
      {
          // 一个轮次包含一条用户消息和一条助手回复，因此保留的轮次以用户消息开头。
          var older = history[..^(2 * KeepTurns)];
          var recent = history[^(2 * KeepTurns)..];
          var summary = await client.Beta.Messages.Create(new MessageCreateParams
          {
              Model = Model.ClaudeOpus5_5,
              MaxTokens = 4096,
              System = SystemPrompt,
              Betas = [AnthropicBeta.Compact2026_09_04],
              Messages = older,
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
                  .. recent,
              ];
              Console.WriteLine($"Kept {recent.Count / 2} turns after the block");
          }
      }
  }
  ```

  ```go Go
  ctx := context.Background()
  client := anthropic.NewClient()

  // 请将此值设置为接近您实际的输入预算。此处设得较低，以便简短的对话也会触发压缩。
  const compactAtTokens = 2500
  system := []anthropic.BetaTextBlockParam{{Text: "You help design a recipe app's data model. Keep answers short."}}
  const keepTurns = 2

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
  	if conversationTokens > compactAtTokens && turn > keepTurns && turn < len(questions) {
  		// 一轮对话包含一条用户消息和一条助手回复，因此保留的轮次以用户消息开头。
  		split := len(history) - 2*keepTurns
  		older, recent := history[:split], history[split:]
  		summary, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  			Model:     anthropic.ModelClaudeOpus5_5,
  			MaxTokens: 4096,
  			System:    system,
  			Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaCompact2026_09_04},
  			Messages:  older,
  			Compaction: anthropic.BetaCompactionConfigUnionParam{
  				OfSummarize: &anthropic.BetaSummarizeCompactionParam{},
  			},
  		})
  		if err != nil {
  			log.Fatal(err)
  		}
  		if summary.StopReason == anthropic.BetaStopReasonCompaction {
  			history = slices.Replace(history, 0, split, summary.ToParam())
  			fmt.Printf("Kept %d turns after the block\n", len(recent)/2)
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
  static final int KEEP_TURNS = 2;

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
          if (conversationTokens > COMPACT_AT_TOKENS && turn > KEEP_TURNS && turn < questions.size()) {
              // 一个轮次包含一条用户消息和一条助手回复，因此保留的轮次以用户消息开头。
              var older = history.subList(0, history.size() - 2 * KEEP_TURNS);
              var summaryParams = MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5_5)
                  .maxTokens(4096)
                  .system(SYSTEM)
                  .addBeta(AnthropicBeta.COMPACT_2026_09_04)
                  .messages(older)
                  .compaction(BetaCompactionConfig.builder().build()) // type defaults to "summarize"
                  .build();
              var summary = client.beta().messages().create(summaryParams);
              if (summary.stopReason().map(BetaStopReason.COMPACTION::equals).orElse(false)) {
                  older.clear();
                  history.addFirst(summary.toParam());
                  IO.println("Kept " + (history.size() - 1) / 2 + " turns after the block");
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
  const KEEP_TURNS = 2;

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
      if ($conversationTokens > COMPACT_AT_TOKENS && $turn > KEEP_TURNS && $turn < count($questions)) {
          // 一个轮次包含一条用户消息和一条助手回复，因此保留的轮次以用户消息开头。
          $older = array_slice($history, 0, -2 * KEEP_TURNS);
          $recent = array_slice($history, -2 * KEEP_TURNS);
          $summary = $client->beta->messages->create(
              model: Model::CLAUDE_OPUS_5_5,
              maxTokens: 4096,
              system: SYSTEM,
              betas: [AnthropicBeta::COMPACT_2026_09_04],
              messages: $older,
              compaction: new BetaCompactionConfig(), // type defaults to 'summarize'
          );
          if ($summary->stopReason === BetaStopReason::COMPACTION->value) {
              $history = [BetaMessageParam::with(role: Role::ASSISTANT, content: $summary->content), ...$recent];
              printf("Kept %d turns after the block\n", intdiv(count($recent), 2));
          }
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 请将此值设置为接近您实际的输入预算。此处设得较低，以便简短的对话也会触发压缩。
  COMPACT_AT_TOKENS = 2500
  SYSTEM = "You help design a recipe app's data model. Keep answers short."
  KEEP_TURNS = 2

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
    if conversation_tokens > COMPACT_AT_TOKENS && turn > KEEP_TURNS && turn < questions.length
      # 一个轮次包含一条用户消息和一条助手回复，因此保留的轮次以用户消息开头。
      older, recent = history[...-2 * KEEP_TURNS], history.last(2 * KEEP_TURNS)
      summary = client.beta.messages.create(
        model: Anthropic::Model::CLAUDE_OPUS_5_5,
        max_tokens: 4096,
        system_: SYSTEM,
        betas: [Anthropic::AnthropicBeta::COMPACT_2026_09_04],
        messages: older,
        compaction: { type: "summarize" }
      )
      if summary.stop_reason == :compaction
        history = [{ role: "assistant", content: summary.content }, *recent]
        puts "Kept #{recent.length / 2} turns after the block"
      end
    end
  end
  ```
</CodeGroup>

* **选择切分点：** 该程序保留最后两个轮次，其中一个轮次是指一条 `user` 消息及对它的回复。程序在距末尾四条消息处拆分历史记录，因此保留的轮次以一条 `user` 消息开头。
* **决定何时压缩：** 大小检查还要求对话的轮次多于程序保留的轮次，因此较早的部分永远不会为空。
* **压缩请求：** 原循环发送整个历史记录，而此版本只发送较早的消息。
* **替换：** 原循环用返回的消息替换整个历史记录，而此版本的新历史记录是返回的消息后跟保留的轮次。

`stop_reason` 检查以及替换之后的每个请求都与原循环相同。

## 保持保留轮次中的思考有效

如果您在支持[保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)的模型上发回思考块，则保留轮次中的思考仅在[保留的思考保持有效的条件](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)成立时才保持有效，其中一个条件限制了切分点可以落在的位置。

该程序的切分点位于一条回复与下一条 `user` 消息之间，满足该条件。在您已发出的某个请求的末尾进行切分同样满足该条件：精确压缩该请求的 `messages`，并保留您的历史记录自那以后新增的所有内容。
