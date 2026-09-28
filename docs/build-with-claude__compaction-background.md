---
title: 后台压缩
url: https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-background
description: 在对话基于完整历史继续进行的同时请求按需压缩摘要，然后在摘要到达时将该块换入。
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

"Background compaction"（后台压缩），通常也称为"async compaction"（异步压缩），会改变[压缩循环](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#compact-in-a-loop)中的两件事：压缩请求在对话基于完整历史继续进行的同时运行，而替换操作会等到压缩块到达后才进行。[从摘要继续](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#continue-from-the-summary)和[处理缺失的摘要或错误](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#when-no-summary-comes-back)中的内容同样适用，无需更改。

## 工作继续进行时如何替换

压缩请求及其返回的块与循环中的相同。在发送请求和使用其结果之间，您的历史记录会不断增长，而替换操作必须保留这部分增长的内容。

1. 使用当前的历史记录发送[压缩请求](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#request-a-summary)，并记录其中包含的消息数量。
2. 在该请求运行期间，继续基于完整历史进行对话。追加每个新轮次，不要编辑历史记录中已有的任何内容，并且在此请求被替换进来或失败之前，不要启动另一个压缩请求。
3. 当响应到达且 `stop_reason` 为 `"compaction"` 时，从历史记录开头准确删除您发送的那些消息，并将返回的消息放在它们的位置。自步骤 1 以来追加的每个轮次都保留在其后。
4. 在块到达后的第一个请求中发送替换后的历史记录，以便在编写摘要期间产生的思考保持有效。

例如，如果压缩请求包含消息 1 到 5，而对话在其运行期间新增了消息 6 到 8，那么替换后您的历史记录就是该块后跟消息 6 到 8。

![Background compaction（后台压缩）时间线：压缩请求携带消息 1 到 5 发送，同时对话基于完整历史继续进行并新增消息 6 到 8；当块到达时，它替换历史记录开头的消息 1 到 5，历史记录变为该块后跟消息 6 到 8](https://platform.claude.com/docs/images/compaction-background-timeline.svg)

如果响应具有任何其他 `stop_reason`，则表示未生成摘要，这在步骤 2 中算作失败。请保留完整历史记录；[处理缺失的摘要或错误](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#when-no-summary-comes-back)列出了各种原因以及每种情况的处理方法。

## 在后台请求摘要

与任何其他请求一样，压缩请求也会计入您的"rate limit"（速率限制），并且在其运行期间，您的应用程序会同时有两个请求处于打开状态。在替换之前，对话会基于完整历史持续增长，因此请在"context window"（上下文窗口）仍有空间容纳期间到达的轮次时启动压缩请求。

以下程序是[在循环中压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#compact-in-a-loop)中的循环，只是将压缩请求从对话的执行路径中移出。它没有 PHP 版本，因为该示例依赖于同时运行两个请求。高亮显示的行表示它与该循环的不同之处，下面的列表按程序运行的顺序逐一说明这些不同之处。

<CodeGroup exclude="shell, php">
  ```python Python
  from concurrent.futures import Future, ThreadPoolExecutor

  import anthropic
  from anthropic.types.beta import BetaMessage, BetaMessageParam

  client = anthropic.Anthropic()
  executor = ThreadPoolExecutor(max_workers=1)

  # 请将此值设为接近您实际的输入预算。此处设得较低，以便简短对话也会触发压缩。
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


  def swap_in(history: list[BetaMessageParam], summary: BetaMessage, sent: int) -> None:
      if summary.stop_reason == "compaction":
          # 精确替换压缩请求中包含的那些消息。
          # 后续轮次保留在该块之后。
          history[:sent] = [{"role": "assistant", "content": summary.content}]
          print(f"Swapped {sent} messages")


  history: list[BetaMessageParam] = []
  pending: Future[BetaMessage] | None = None
  sent = 0
  for turn, question in enumerate(QUESTIONS, start=1):
      if pending is not None and pending.done():
          swap_in(history, pending.result(), sent)
          pending = None

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
      if (
          conversation_tokens > COMPACT_AT_TOKENS
          and turn < len(QUESTIONS)
          and pending is None
      ):
          sent = len(history)
          pending = executor.submit(
              client.beta.messages.create,
              model="claude-opus-5-5",
              max_tokens=4096,
              system=SYSTEM,
              betas=["compact-2026-09-04"],
              messages=history.copy(),
              compaction={"type": "summarize"},
          )

  # 在保存之前，先换入仍在生成中的摘要
  # 或继续对话。
  if pending is not None:
      swap_in(history, pending.result(), sent)
  executor.shutdown()
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // 请将此值设为接近您实际的输入预算。此处设得较低，以便较短的对话也会触发压缩。
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

  function swapIn(
    history: Anthropic.Beta.Messages.BetaMessageParam[],
    summary: Anthropic.Beta.Messages.BetaMessage,
    sent: number
  ): Anthropic.Beta.Messages.BetaMessageParam[] {
    if (summary.stop_reason !== "compaction") {
      return history;
    }
    console.log(`Swapped ${sent} messages`);
    // 精确替换压缩请求所包含的消息。后续轮次保留在该块之后。
    return [{ role: "assistant", content: summary.content }, ...history.slice(sent)];
  }

  let history: Anthropic.Beta.Messages.BetaMessageParam[] = [];
  let pending: Promise<Anthropic.Beta.Messages.BetaMessage> | undefined;
  let settled = false;
  let sent = 0;
  for (const [index, question] of questions.entries()) {
    const turn = index + 1;
    if (pending && settled) {
      history = swapIn(history, await pending, sent);
      pending = undefined;
      settled = false;
    }

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
    if (conversationTokens > compactAtTokens && turn < questions.length && !pending) {
      sent = history.length;
      pending = client.beta.messages.create({
        model: "claude-opus-5-5",
        max_tokens: 4096,
        system: systemPrompt,
        betas: ["compact-2026-09-04"],
        messages: [...history],
        compaction: { type: "summarize" }
      });
      // 无论结果如何，都将该请求标记为已完成。之后 await 它会返回摘要或抛出异常。
      const markSettled = () => {
        settled = true;
      };
      pending.then(markSettled, markSettled);
    }
  }

  // 在保存或继续对话之前，先换入仍在生成中的摘要。
  if (pending) {
    history = swapIn(history, await pending, sent);
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Model = Anthropic.Models.Messages.Model;

  AnthropicClient client = new();

  // 请将此值设为接近您实际的输入预算。此处设得较低，以便简短对话也会触发压缩。
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

  static List<BetaMessageParam> SwapIn(List<BetaMessageParam> history, BetaMessage summary, int sent)
  {
      if (summary.StopReason != BetaStopReason.Compaction)
      {
          return history;
      }
      Console.WriteLine($"Swapped {sent} messages");
      // 仅替换压缩请求中包含的那些消息。后续轮次保留在该块之后。
      return
      [
          new()
          {
              Role = Role.Assistant,
              Content = summary.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
          },
          .. history[sent..],
      ];
  }

  List<BetaMessageParam> history = [];
  Task<BetaMessage>? pending = null;
  var sent = 0;
  foreach (var (index, question) in questions.Index())
  {
      var turn = index + 1;
      if (pending is { IsCompleted: true })
      {
          history = SwapIn(history, await pending, sent);
          pending = null;
      }

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

      // 下一个请求也会发送此回复，因此请将其计入。
      var conversationTokens = response.Usage.InputTokens + response.Usage.OutputTokens;
      if (conversationTokens > CompactAtTokens && turn < questions.Length && pending is null)
      {
          sent = history.Count;
          pending = client.Beta.Messages.Create(new MessageCreateParams
          {
              Model = Model.ClaudeOpus5_5,
              MaxTokens = 4096,
              System = SystemPrompt,
              Betas = [AnthropicBeta.Compact2026_09_04],
              Messages = [.. history],
              Compaction = new BetaCompactionConfig(), // type defaults to "summarize"
          });
      }
  }

  // 在保存或继续对话之前，请先换入仍在生成中的摘要。
  if (pending is not null)
  {
      history = SwapIn(history, await pending, sent);
  }
  ```

  ```go Go
  ctx := context.Background()
  client := anthropic.NewClient()

  // 请将此值设为接近您实际的输入预算。此处设得较低，以便简短的对话也会触发压缩。
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
  var pending chan *anthropic.BetaMessage
  var sent int
  swapIn := func(summary *anthropic.BetaMessage) {
  	if summary.StopReason != anthropic.BetaStopReasonCompaction {
  		return
  	}
  	fmt.Printf("Swapped %d messages\n", sent)
  	// 精确替换压缩请求所包含的消息。后续轮次保留在该块之后。
  	history = slices.Replace(history, 0, sent, summary.ToParam())
  }

  for i, question := range questions {
  	turn := i + 1
  	// 从 nil channel 接收永远不会成功，因此没有待处理项时会跳过此分支。
  	select {
  	case summary := <-pending:
  		swapIn(summary)
  		pending = nil
  	default:
  	}

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
  	if conversationTokens > compactAtTokens && turn < len(questions) && pending == nil {
  		sent = len(history)
  		pending = make(chan *anthropic.BetaMessage, 1)
  		go func(messages []anthropic.BetaMessageParam, result chan<- *anthropic.BetaMessage) {
  			summary, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  				Model:     anthropic.ModelClaudeOpus5_5,
  				MaxTokens: 4096,
  				System:    system,
  				Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaCompact2026_09_04},
  				Messages:  messages,
  				Compaction: anthropic.BetaCompactionConfigUnionParam{
  					OfSummarize: &anthropic.BetaSummarizeCompactionParam{},
  				},
  			})
  			if err != nil {
  				log.Fatal(err)
  			}
  			result <- summary
  		}(slices.Clone(history), pending)
  	}
  }

  // 在保存或继续对话之前，请先换入仍在生成中的摘要。
  if pending != nil {
  	swapIn(<-pending)
  }
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaCompactionConfig;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaMessageParam;
  import com.anthropic.models.beta.messages.BetaStopReason;
  import com.anthropic.models.beta.messages.MessageCreateParams;

  // 请将此值设置为接近您实际的输入预算。此处设得较低，以便短对话也会触发压缩。
  static final long COMPACT_AT_TOKENS = 2500;
  static final String SYSTEM = "You help design a recipe app's data model. Keep answers short.";

  void swapIn(List<BetaMessageParam> history, BetaMessage summary, int sent) {
      if (!summary.stopReason().map(BetaStopReason.COMPACTION::equals).orElse(false)) {
          return;
      }
      IO.println("Swapped " + sent + " messages");
      // 仅替换压缩请求中包含的那些消息。之后的轮次保留在该块之后。
      history.subList(0, sent).clear();
      history.addFirst(summary.toParam());
  }

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
      CompletableFuture<BetaMessage> pending = null;
      int sent = 0;
      for (int turn = 1; turn <= questions.size(); turn++) {
          if (pending != null && pending.isDone()) {
              swapIn(history, pending.join(), sent);
              pending = null;
          }

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
          if (conversationTokens > COMPACT_AT_TOKENS && turn < questions.size() && pending == null) {
              sent = history.size();
              var summaryParams = MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5_5)
                  .maxTokens(4096)
                  .system(SYSTEM)
                  .addBeta(AnthropicBeta.COMPACT_2026_09_04)
                  .messages(List.copyOf(history))
                  .compaction(BetaCompactionConfig.builder().build()) // type defaults to "summarize"
                  .build();
              pending = client.async().beta().messages().create(summaryParams);
          }
      }

      // 在保存或继续对话之前，请换入仍在生成中的摘要。
      if (pending != null) {
          swapIn(history, pending.join(), sent);
      }
      client.close();
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 请将此值设为接近您实际的输入预算。此处设得较低，以便简短对话也会触发压缩。
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

  def swap_in(history, summary, sent)
    return history unless summary.stop_reason == :compaction

    puts "Swapped #{sent} messages"
    # 仅替换压缩请求所包含的那些消息。后续轮次保留在该块之后。
    [{ role: "assistant", content: summary.content }, *history[sent..]]
  end

  history = []
  pending = nil
  sent = 0
  questions.each.with_index(1) do |question, turn|
    if pending && !pending.alive?
      history = swap_in(history, pending.value, sent)
      pending = nil
    end

    history << { role: "user", content: question }
    response = client.beta.messages.create(
      model: Anthropic::Model::CLAUDE_OPUS_5_5,
      max_tokens: 8192,
      system_: SYSTEM,
      betas: [Anthropic::AnthropicBeta::COMPACT_2026_09_04],
      messages: history
    )
    history << { role: "assistant", content: response.content }

    # 下一个请求也会发送此回复，因此请将其计入。
    conversation_tokens = response.usage.input_tokens + response.usage.output_tokens
    if conversation_tokens > COMPACT_AT_TOKENS && turn < questions.length && pending.nil?
      sent = history.length
      pending = Thread.new(history.dup) do |snapshot|
        client.beta.messages.create(
          model: Anthropic::Model::CLAUDE_OPUS_5_5,
          max_tokens: 4096,
          system_: SYSTEM,
          betas: [Anthropic::AnthropicBeta::COMPACT_2026_09_04],
          messages: snapshot,
          compaction: { type: "summarize" }
        )
      end
    end
  end

  # 在保存或继续对话之前，请先换入仍在生成中的摘要。
  history = swap_in(history, pending.value, sent) if pending
  ```
</CodeGroup>

* **决定何时压缩：** 大小检查还要求当前没有待处理的压缩请求。
* **启动请求：** 在循环等待压缩响应的地方，此版本会记录历史记录中包含的消息数量，使用每种语言自身的并发工具在历史记录的副本上启动请求，然后不等待结果直接进入下一个轮次。
* **检查结果：** 在每个轮次开始时，程序会检查待处理的请求是否已完成。如果已完成，程序会在发送该轮次的请求之前进行替换。
* **进行替换：** 在循环用返回的消息替换整个历史记录的地方，此版本的替换函数仅替换请求所包含的消息（从开头开始计数），并保留此后追加的所有内容。
* **结束循环：** 如果循环结束时压缩请求仍处于待处理状态，程序会等待它完成并进行替换，这样在您保存或继续对话之前，仍在途中的摘要就不会丢失。

`stop_reason` 检查与循环中的相同：没有块的响应会使历史记录保持原样。由于不再有待处理的请求，程序随后可以启动新的压缩请求。

## 在构建摘要期间保持思考有效

在编写摘要期间到达的轮次属于保留的轮次。如果您在支持[保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)的模型上回传思考块，则这些轮次中的思考仅在[保留的思考保持有效的条件](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)成立时才保持有效。
