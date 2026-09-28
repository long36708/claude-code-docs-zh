---
title: 压缩与保留思考
url: https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks
description: 在支持保留思考的模型上，按需压缩后保留的轮次中的思考块何时仍然有效，以及如何进行检查。
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

除非您会将思考块发送回具有["preserved thinking"（保留思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)的模型，并在压缩块之后保留轮次，否则可以跳过本页。"Kept turns"（保留的轮次）是指紧随该块之后的轮次：即您未纳入压缩请求的近期轮次（如[保留最近轮次的压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-keep-recent-turns)中所述），或在摘要生成期间到达的轮次（如[后台压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-background)中所述）。

具有保留思考的模型会将较早的思考块与生成它们的对话进行核对。摘要会替换该对话的一部分，但当摘要由 API 编写时，该检查会接受这种替换，因此保留轮次中的思考可以保持有效。

## 保留的思考保持有效的条件

只要以下所有条件都成立，保留轮次中的思考块就会保持有效：

* **压缩请求在具有保留思考的模型上运行。** 此条件涵盖自思考块生成以来的每一次压缩请求，而不仅仅是最近的一次。满足此条件的一种方法是将每个压缩请求都发送到该对话所使用的模型。
* **保留的轮次紧跟在被摘要的消息之后，并且您原样发送它们。** 请完全按照历史记录中的样子发送每条保留的消息。不要在最后一条被摘要的消息与第一条保留的消息之间跳过或添加任何消息。第一条保留的消息的角色还必须与最后一条被摘要的消息不同，并且不能是对话中途的 `role: "system"` 消息。否则，API 会将其合并到最后一条被摘要的消息中。确保第一条保留消息正确的一种方法是，对您已发送过的某个请求的 `messages` 进行完全一致的压缩。这样，保留的轮次就会从 Claude 对该请求的回复开始。
* **`system` 以及未标记 `defer_loading: true` 的 `tools` 不发生变化。** 它们在压缩请求中与生成保留思考的请求中保持一致，并且在后续请求中也保持不变。[更改系统提示或工具](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#change-the-system-prompt-or-tools)介绍了如何安全地更改它们。

如果某个条件不成立，压缩时不会出现任何失败，并且无论如何 API 都会在后续请求中接受该块。在 API 强制执行检查的情况下，失败会出现在第一个发送保留的思考的后续请求上：默认情况下返回 400 错误；如果请求将 `thinking.block_binding.prefix_mismatch_behavior` 设置为 `"drop_block"`，则会丢弃思考块。在 Message Batches API 中，未设置该字段的条目不会失败。在默认应用检查的情况下，API 会改为丢弃这些块。[API 如何处理失效的块](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#mismatch-behavior)描述了这两种结果，[API 何时强制执行检查](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#enforcement)说明了哪些请求会被检查。

## 再次压缩而不破坏较早的思考

您可以[再次压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#compact-again)并保留轮次：新块涵盖旧摘要以及压缩请求中其后的每条消息，而您未纳入该请求的任何轮次都是新块的保留轮次。

[保留的思考保持有效的条件](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)中的第一条会计入自思考块生成以来的每一次压缩，因此，对于经过两次压缩仍被保留的轮次，这两次压缩都必须在具有保留思考的模型上运行。

在思考块生成之前进行的压缩不会对其产生影响。在某个块就位之后生成的思考会绑定到该块，并且在满足条件的后续压缩中保持有效。

## 更改系统提示或工具

后续请求可以使用与压缩请求不同的 `system`、不同的 `tools` 或不同的模型，API 仍会接受该块。此类更改可能会使保留轮次中的思考失效，但不会产生其他影响。

要在不使任何保留思考失效的情况下更改 `system` 或 `tools`，请先压缩整个对话，使其不保留任何轮次。然后在下一个请求中进行更改。

要在不改动 `system` 或 `tools` 的情况下添加指令或更改可用工具，请将更改追加到 `messages` 中，如[在不编辑前缀的情况下进行更改](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#replace-prefix-edits)中所述。

被摘要轮次中的对话中途系统消息也会被摘要，因此在替换之后，其文本指令将不再生效。要让其中某条继续生效，请在保留轮次之后的第一个新 `user` 轮次之后，紧接着用一条 `role: "system"` 消息重新声明它。当压缩请求同时携带 `inline-tools-2026-09-15` 时，这些轮次中的工具更改会自动延续：返回的块会在其 `tool_changes` 字段中记录这些更改的净效果，因此请原样发回该块。如果该块没有 `tool_changes` 字段，请以同样的方式重新声明这些工具更改。放置在块与保留轮次之间的系统消息会破坏这些轮次的思考。

## 检查保留的思考是否有效

压缩响应不会告诉您保留的思考是否有效，而替换后的第一个请求会。要在测试中进行检查：

1. 在开启思考的情况下进行一段简短的对话。使用 API 会对其运行检查的模型（请参阅 [API 何时强制执行检查](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#enforcement)），并在每个步骤中都使用该模型，因为无法读取思考块的模型会直接丢弃它而不报错。
2. 压缩较早的轮次，并至少保留一个包含思考块的轮次。
3. 发送下一个请求，依次包含该块、保留的轮次和一条新的 `user` 消息，并将 `thinking.block_binding.prefix_mismatch_behavior` 设置为 `"error"`。
4. 读取结果。如果返回 200 响应且其 `input_transformations` 数组为空，则表示所有思考块都通过了检查，且没有块被丢弃。如果返回 400 错误并提示该块绑定到了不同的对话，则表示有思考块未通过检查。错误消息以第一个未通过检查的块的路径开头，[API 如何处理失效的块](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#mismatch-behavior)展示了完整的消息。

除了 [`compact-2026-09-04` beta 标头](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#request-a-summary)之外，`prefix_mismatch_behavior` 字段还需要 `thinking-binding-controls-2026-08-01` beta 标头。在默认未开启检查的账户上，设置该字段也会让请求选择启用该检查。

以下程序运行这四个步骤。它会打印保留轮次中包含的思考块数量以及 `input_transformations` 中的条目数量；没有条目表示保留的思考有效：

<CodeGroup exclude="shell">
  ```python Python
  from anthropic.types.beta import BetaMessageParam, BetaThinkingConfigParam

  client = anthropic.Anthropic()

  # Claude Fable 5.1 是首个会将回传的思考内容与对话进行核对的模型。
  MODEL = "claude-fable-5-1"
  BETAS = ["compact-2026-09-04", "thinking-binding-controls-2026-08-01"]
  SYSTEM = "You help plan a recipe app's release. Keep answers short."
  # 设为 "error" 时，未通过检查的 thinking block（思考块）会导致请求失败并返回 400。
  THINKING: BetaThinkingConfigParam = {
      "type": "adaptive",
      "block_binding": {"prefix_mismatch_behavior": "error"},
  }

  # 1. 在开启思考的情况下进行一段简短对话。
  history: list[BetaMessageParam] = [
      {"role": "user", "content": "What are the main entities in the app's data model?"}
  ]
  first = client.beta.messages.create(
      model=MODEL,
      max_tokens=8192,
      system=SYSTEM,
      betas=BETAS,
      thinking=THINKING,
      messages=history,
  )
  history += [
      {"role": "assistant", "content": first.content},
      {
          "role": "user",
          "content": "Testing starts on Tuesday, March 3, 2026, takes 10 weekdays, and pauses on March 9 and March 16. On which date does it end?",
      },
  ]
  second = client.beta.messages.create(
      model=MODEL,
      max_tokens=8192,
      system=SYSTEM,
      betas=BETAS,
      thinking=THINKING,
      messages=history,
  )
  history.append({"role": "assistant", "content": second.content})
  thinking_blocks = sum(block.type == "thinking" for block in second.content)
  print(f"Thinking blocks in the kept turn: {thinking_blocks}")

  # 2. 总结第一轮对话。第二轮不包含在请求中。
  summary = client.beta.messages.create(
      model=MODEL,
      max_tokens=4096,
      system=SYSTEM,
      betas=BETAS,
      thinking=THINKING,
      messages=history[:2],
      compaction={"type": "summarize"},
  )
  if summary.stop_reason != "compaction":
      raise SystemExit(f"No summary: {summary.stop_reason}")

  # 3. 将该思考块放在保留的轮次之前，然后提出下一个问题。
  history = [
      {"role": "assistant", "content": summary.content},
      *history[2:],
      {"role": "user", "content": "Which day should the release go out?"},
  ]
  third = client.beta.messages.create(
      model=MODEL,
      max_tokens=8192,
      system=SYSTEM,
      betas=BETAS,
      thinking=THINKING,
      messages=history,
  )

  # 4. 若返回 200 且没有思考块被丢弃，则说明保留的思考内容通过了检查。
  print(f"Dropped thinking blocks: {len(third.input_transformations)}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // Claude Fable 5.1 是首个会将回传的思考内容与对话进行比对校验的模型。
  const model: Anthropic.Model = "claude-fable-5-1";
  const betas: Anthropic.Beta.AnthropicBeta[] = [
    "compact-2026-09-04",
    "thinking-binding-controls-2026-08-01"
  ];
  const systemPrompt = "You help plan a recipe app's release. Keep answers short.";
  // 设为 "error" 时，未通过校验的思考块会导致请求失败并返回 400。
  const thinking: Anthropic.Beta.Messages.BetaThinkingConfigParam = {
    type: "adaptive",
    block_binding: { prefix_mismatch_behavior: "error" }
  };

  // 1. 在开启思考的情况下进行一段简短对话。
  let history: Anthropic.Beta.Messages.BetaMessageParam[] = [
    { role: "user", content: "What are the main entities in the app's data model?" }
  ];
  const first = await client.beta.messages.create({
    model,
    max_tokens: 8192,
    system: systemPrompt,
    betas,
    thinking,
    messages: history
  });
  history.push(
    { role: "assistant", content: first.content },
    {
      role: "user",
      content:
        "Testing starts on Tuesday, March 3, 2026, takes 10 weekdays, and pauses on March 9 and March 16. On which date does it end?"
    }
  );
  const second = await client.beta.messages.create({
    model,
    max_tokens: 8192,
    system: systemPrompt,
    betas,
    thinking,
    messages: history
  });
  history.push({ role: "assistant", content: second.content });
  const thinkingBlocks = second.content.filter((block) => block.type === "thinking").length;
  console.log(`Thinking blocks in the kept turn: ${thinkingBlocks}`);

  // 2. 总结第一轮对话。第二轮不包含在请求中。
  const summary = await client.beta.messages.create({
    model,
    max_tokens: 4096,
    system: systemPrompt,
    betas,
    thinking,
    messages: history.slice(0, 2),
    compaction: { type: "summarize" }
  });
  if (summary.stop_reason !== "compaction") {
    throw new Error(`No summary: ${summary.stop_reason}`);
  }

  // 3. 将该块放在保留的轮次之前，然后提出下一个问题。
  history = [
    { role: "assistant", content: summary.content },
    ...history.slice(2),
    { role: "user", content: "Which day should the release go out?" }
  ];
  const third = await client.beta.messages.create({
    model,
    max_tokens: 8192,
    system: systemPrompt,
    betas,
    thinking,
    messages: history
  });

  // 4. 返回 200 且没有块被丢弃，说明保留的思考内容通过了校验。
  console.log(`Dropped thinking blocks: ${third.input_transformations?.length ?? 0}`);
  ```

  ```csharp C#
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;
  using Model = Anthropic.Models.Messages.Model;

  AnthropicClient client = new();

  // Claude Fable 5.1 是首个会根据对话校验回传的思考内容的模型。
  const Model ModelId = Model.ClaudeFable5_1;
  AnthropicBeta[] betas = [AnthropicBeta.Compact2026_09_04, AnthropicBeta.ThinkingBindingControls2026_08_01];
  const string SystemPrompt = "You help plan a recipe app's release. Keep answers short.";
  // 设置为 "error" 时，未通过校验的思考块会导致请求失败并返回 400。
  BetaThinkingConfigAdaptive thinking = new()
  {
      BlockBinding = new() { PrefixMismatchBehavior = BetaThinkingPrefixMismatchBehavior.Error },
  };

  // 1. 在启用思考的情况下进行一段简短对话。
  List<BetaMessageParam> history =
  [
      new() { Role = Role.User, Content = "What are the main entities in the app's data model?" },
  ];
  var first = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = ModelId,
      MaxTokens = 8192,
      System = SystemPrompt,
      Betas = [.. betas],
      Thinking = thinking,
      Messages = history,
  });
  history.AddRange(
  [
      new()
      {
          Role = Role.Assistant,
          Content = first.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
      },
      new()
      {
          Role = Role.User,
          Content = "Testing starts on Tuesday, March 3, 2026, takes 10 weekdays, and pauses on March 9 and March 16. On which date does it end?",
      },
  ]);
  var second = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = ModelId,
      MaxTokens = 8192,
      System = SystemPrompt,
      Betas = [.. betas],
      Thinking = thinking,
      Messages = history,
  });
  history.Add(new()
  {
      Role = Role.Assistant,
      Content = second.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
  });
  var thinkingBlocks = second.Content.Count(block => block.TryPickThinking(out _));
  Console.WriteLine($"Thinking blocks in the kept turn: {thinkingBlocks}");

  // 2. 总结第一轮对话。第二轮不放入请求中。
  var summary = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = ModelId,
      MaxTokens = 4096,
      System = SystemPrompt,
      Betas = [.. betas],
      Thinking = thinking,
      Messages = history[..2],
      Compaction = new BetaCompactionConfig(), // type defaults to "summarize"
  });
  if (summary.StopReason != BetaStopReason.Compaction)
  {
      throw new InvalidOperationException($"No summary: {summary.StopReason?.Raw()}");
  }

  // 3. 将该块放在保留的轮次之前，然后提出下一个问题。
  history =
  [
      new()
      {
          Role = Role.Assistant,
          Content = summary.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
      },
      .. history[2..],
      new() { Role = Role.User, Content = "Which day should the release go out?" },
  ];
  var third = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = ModelId,
      MaxTokens = 8192,
      System = SystemPrompt,
      Betas = [.. betas],
      Thinking = thinking,
      Messages = history,
  });

  // 4. 返回 200 且没有块被丢弃，说明保留的思考内容通过了校验。
  Console.WriteLine($"Dropped thinking blocks: {third.InputTransformations?.Count ?? 0}");
  ```

  ```go Go
  ctx := context.Background()
  client := anthropic.NewClient()

  // Claude Fable 5.1 是首个会将回传的思考内容与对话进行比对校验的模型。
  const model = anthropic.ModelClaudeFable5_1
  betas := []anthropic.AnthropicBeta{
  	anthropic.AnthropicBetaCompact2026_09_04,
  	anthropic.AnthropicBetaThinkingBindingControls2026_08_01,
  }
  system := []anthropic.BetaTextBlockParam{{Text: "You help plan a recipe app's release. Keep answers short."}}
  // 设置为 "error" 时，未通过校验的思考块会导致请求失败并返回 400。
  thinking := anthropic.BetaThinkingConfigParamUnion{
  	OfAdaptive: &anthropic.BetaThinkingConfigAdaptiveParam{
  		BlockBinding: anthropic.BetaThinkingBlockBindingParam{
  			PrefixMismatchBehavior: anthropic.BetaThinkingPrefixMismatchBehaviorError,
  		},
  	},
  }

  // 1. 开启思考功能，进行一段简短的对话。
  history := []anthropic.BetaMessageParam{
  	anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("What are the main entities in the app's data model?")),
  }
  first, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  	Model:     model,
  	MaxTokens: 8192,
  	System:    system,
  	Betas:     betas,
  	Thinking:  thinking,
  	Messages:  history,
  })
  if err != nil {
  	log.Fatal(err)
  }
  history = append(history,
  	first.ToParam(),
  	anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Testing starts on Tuesday, March 3, 2026, takes 10 weekdays, and pauses on March 9 and March 16. On which date does it end?")),
  )
  second, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  	Model:     model,
  	MaxTokens: 8192,
  	System:    system,
  	Betas:     betas,
  	Thinking:  thinking,
  	Messages:  history,
  })
  if err != nil {
  	log.Fatal(err)
  }
  history = append(history, second.ToParam())
  thinkingBlocks := 0
  for _, block := range second.Content {
  	if _, ok := block.AsAny().(anthropic.BetaThinkingBlock); ok {
  		thinkingBlocks++
  	}
  }
  fmt.Printf("Thinking blocks in the kept turn: %d\n", thinkingBlocks)

  // 2. 对第一轮进行摘要。第二轮不包含在请求中。
  summary, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  	Model:     model,
  	MaxTokens: 4096,
  	System:    system,
  	Betas:     betas,
  	Thinking:  thinking,
  	Messages:  history[:2],
  	Compaction: anthropic.BetaCompactionConfigUnionParam{
  		OfSummarize: &anthropic.BetaSummarizeCompactionParam{},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  if summary.StopReason != anthropic.BetaStopReasonCompaction {
  	log.Fatalf("No summary: %s", summary.StopReason)
  }

  // 3. 将该块放在保留的轮次之前，然后提出下一个问题。
  history = slices.Concat(
  	[]anthropic.BetaMessageParam{summary.ToParam()},
  	history[2:],
  	[]anthropic.BetaMessageParam{anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Which day should the release go out?"))},
  )
  third, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  	Model:     model,
  	MaxTokens: 8192,
  	System:    system,
  	Betas:     betas,
  	Thinking:  thinking,
  	Messages:  history,
  })
  if err != nil {
  	log.Fatal(err)
  }

  // 4. 返回 200 且没有块被丢弃，说明保留的思考内容通过了校验。
  fmt.Printf("Dropped thinking blocks: %d\n", len(third.InputTransformations))
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaCompactionConfig;
  import com.anthropic.models.beta.messages.BetaContentBlock;
  import com.anthropic.models.beta.messages.BetaMessageParam;
  import com.anthropic.models.beta.messages.BetaStopReason;
  import com.anthropic.models.beta.messages.BetaThinkingBlockBinding;
  import com.anthropic.models.beta.messages.BetaThinkingConfigAdaptive;
  import com.anthropic.models.beta.messages.BetaThinkingPrefixMismatchBehavior;
  import com.anthropic.models.beta.messages.MessageCreateParams;

  // Claude Fable 5.1 是首个会将回传的思考内容与对话进行比对校验的模型。
  static final Model MODEL = Model.CLAUDE_FABLE_5_1;
  static final List<AnthropicBeta> BETAS = List.of(
      AnthropicBeta.COMPACT_2026_09_04,
      AnthropicBeta.THINKING_BINDING_CONTROLS_2026_08_01
  );
  static final String SYSTEM = "You help plan a recipe app's release. Keep answers short.";
  // 设为 "error" 时，未通过校验的思考块会导致请求失败并返回 400。
  static final BetaThinkingConfigAdaptive THINKING = BetaThinkingConfigAdaptive.builder()
      .blockBinding(BetaThinkingBlockBinding.builder()
          .prefixMismatchBehavior(BetaThinkingPrefixMismatchBehavior.ERROR)
          .build())
      .build();

  void main() {
      var client = AnthropicOkHttpClient.fromEnv();

      // 1. 在启用思考的情况下进行一段简短对话。
      var history = new ArrayList<BetaMessageParam>();
      history.add(BetaMessageParam.builder()
          .role(BetaMessageParam.Role.USER)
          .content("What are the main entities in the app's data model?")
          .build());
      var firstParams = MessageCreateParams.builder()
          .model(MODEL)
          .maxTokens(8192)
          .system(SYSTEM)
          .betas(BETAS)
          .thinking(THINKING)
          .messages(history)
          .build();
      var first = client.beta().messages().create(firstParams);
      history.add(first.toParam());
      history.add(BetaMessageParam.builder()
          .role(BetaMessageParam.Role.USER)
          .content("Testing starts on Tuesday, March 3, 2026, takes 10 weekdays, and pauses on March 9 and March 16. On which date does it end?")
          .build());
      var secondParams = MessageCreateParams.builder()
          .model(MODEL)
          .maxTokens(8192)
          .system(SYSTEM)
          .betas(BETAS)
          .thinking(THINKING)
          .messages(history)
          .build();
      var second = client.beta().messages().create(secondParams);
      history.add(second.toParam());
      long thinkingBlocks = second.content().stream().filter(BetaContentBlock::isThinking).count();
      IO.println("Thinking blocks in the kept turn: " + thinkingBlocks);

      // 2. 总结第一轮对话。第二轮不包含在请求中。
      var firstTurn = history.subList(0, 2);
      var summaryParams = MessageCreateParams.builder()
          .model(MODEL)
          .maxTokens(4096)
          .system(SYSTEM)
          .betas(BETAS)
          .thinking(THINKING)
          .messages(firstTurn)
          .compaction(BetaCompactionConfig.builder().build()) // type defaults to "summarize"
          .build();
      var summary = client.beta().messages().create(summaryParams);
      if (!summary.stopReason().map(BetaStopReason.COMPACTION::equals).orElse(false)) {
          throw new IllegalStateException("No summary: " + summary.stopReason().orElseThrow());
      }

      // 3. 将该块放在保留的轮次之前，然后提出下一个问题。
      firstTurn.clear();
      history.addFirst(summary.toParam());
      history.add(BetaMessageParam.builder()
          .role(BetaMessageParam.Role.USER)
          .content("Which day should the release go out?")
          .build());
      var thirdParams = MessageCreateParams.builder()
          .model(MODEL)
          .maxTokens(8192)
          .system(SYSTEM)
          .betas(BETAS)
          .thinking(THINKING)
          .messages(history)
          .build();
      var third = client.beta().messages().create(thirdParams);

      // 4. 返回 200 且没有块被丢弃，说明保留的思考内容通过了校验。
      IO.println("Dropped thinking blocks: " + third.inputTransformations().map(List::size).orElse(0));
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaCompactionConfig;
  use Anthropic\Beta\Messages\BetaMessageParam;
  use Anthropic\Beta\Messages\BetaMessageParam\Role;
  use Anthropic\Beta\Messages\BetaStopReason;
  use Anthropic\Beta\Messages\BetaThinkingBlock;
  use Anthropic\Beta\Messages\BetaThinkingBlockBinding;
  use Anthropic\Beta\Messages\BetaThinkingConfigAdaptive;
  use Anthropic\Beta\Messages\BetaThinkingPrefixMismatchBehavior;

  $client = new Client();

  // Claude Fable 5.1 是首个会将回传的思考内容与对话进行核对的模型。
  const MODEL = Model::CLAUDE_FABLE_5_1;
  const BETAS = [AnthropicBeta::COMPACT_2026_09_04, AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01];
  const SYSTEM = "You help plan a recipe app's release. Keep answers short.";
  // 设置为 "error" 时，未通过核对的 thinking 块会导致请求失败并返回 400。
  $thinking = BetaThinkingConfigAdaptive::with(
      blockBinding: BetaThinkingBlockBinding::with(
          prefixMismatchBehavior: BetaThinkingPrefixMismatchBehavior::ERROR,
      ),
  );

  // 1. 在开启思考的情况下进行一段简短对话。
  $history = [
      BetaMessageParam::with(role: Role::USER, content: "What are the main entities in the app's data model?"),
  ];
  $first = $client->beta->messages->create(
      model: MODEL,
      maxTokens: 8192,
      system: SYSTEM,
      betas: BETAS,
      thinking: $thinking,
      messages: $history,
  );
  $history[] = BetaMessageParam::with(role: Role::ASSISTANT, content: $first->content);
  $history[] = BetaMessageParam::with(
      role: Role::USER,
      content: 'Testing starts on Tuesday, March 3, 2026, takes 10 weekdays, and pauses on March 9 and March 16. On which date does it end?',
  );
  $second = $client->beta->messages->create(
      model: MODEL,
      maxTokens: 8192,
      system: SYSTEM,
      betas: BETAS,
      thinking: $thinking,
      messages: $history,
  );
  $history[] = BetaMessageParam::with(role: Role::ASSISTANT, content: $second->content);
  $thinkingBlocks = count(array_filter($second->content, fn ($block) => $block instanceof BetaThinkingBlock));
  printf("Thinking blocks in the kept turn: %d\n", $thinkingBlocks);

  // 2. 总结第一轮对话。第二轮对话不包含在请求中。
  $summary = $client->beta->messages->create(
      model: MODEL,
      maxTokens: 4096,
      system: SYSTEM,
      betas: BETAS,
      thinking: $thinking,
      messages: array_slice($history, 0, 2),
      compaction: new BetaCompactionConfig(), // type defaults to 'summarize'
  );
  if ($summary->stopReason !== BetaStopReason::COMPACTION->value) {
      throw new RuntimeException("No summary: {$summary->stopReason}");
  }

  // 3. 将该块放在保留的轮次之前，然后提出下一个问题。
  $history = [
      BetaMessageParam::with(role: Role::ASSISTANT, content: $summary->content),
      ...array_slice($history, 2),
      BetaMessageParam::with(role: Role::USER, content: 'Which day should the release go out?'),
  ];
  $third = $client->beta->messages->create(
      model: MODEL,
      maxTokens: 8192,
      system: SYSTEM,
      betas: BETAS,
      thinking: $thinking,
      messages: $history,
  );

  // 4. 返回 200 且没有块被丢弃，说明保留的思考内容有效。
  printf("Dropped thinking blocks: %d\n", count($third->inputTransformations));
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # Claude Fable 5.1 是首个会将回传的思考内容与对话进行核对的模型。
  MODEL = Anthropic::Model::CLAUDE_FABLE_5_1
  BETAS = [
    Anthropic::AnthropicBeta::COMPACT_2026_09_04,
    Anthropic::AnthropicBeta::THINKING_BINDING_CONTROLS_2026_08_01
  ]
  SYSTEM = "You help plan a recipe app's release. Keep answers short."
  # 设为 "error" 时，若 thinking block（思考块）未通过检查，请求将失败并返回 400。
  THINKING = Anthropic::Beta::BetaThinkingConfigAdaptive.new(
    block_binding: Anthropic::Beta::BetaThinkingBlockBinding.new(
      prefix_mismatch_behavior: Anthropic::Beta::BetaThinkingPrefixMismatchBehavior::ERROR
    )
  )

  # 1. 在开启思考的情况下进行一段简短对话。
  history = [{ role: "user", content: "What are the main entities in the app's data model?" }]
  first = client.beta.messages.create(
    model: MODEL,
    max_tokens: 8192,
    system_: SYSTEM,
    betas: BETAS,
    thinking: THINKING,
    messages: history
  )
  history += [
    { role: "assistant", content: first.content },
    {
      role: "user",
      content: "Testing starts on Tuesday, March 3, 2026, takes 10 weekdays, and pauses on March 9 and March 16. On which date does it end?"
    }
  ]
  second = client.beta.messages.create(
    model: MODEL,
    max_tokens: 8192,
    system_: SYSTEM,
    betas: BETAS,
    thinking: THINKING,
    messages: history
  )
  history << { role: "assistant", content: second.content }
  thinking_blocks = second.content.count { it.is_a?(Anthropic::Beta::BetaThinkingBlock) }
  puts "Thinking blocks in the kept turn: #{thinking_blocks}"

  # 2. 对第一轮进行摘要。第二轮不放入请求中。
  summary = client.beta.messages.create(
    model: MODEL,
    max_tokens: 4096,
    system_: SYSTEM,
    betas: BETAS,
    thinking: THINKING,
    messages: history.first(2),
    compaction: { type: "summarize" }
  )
  abort "No summary: #{summary.stop_reason}" unless summary.stop_reason == :compaction

  # 3. 将该块放在保留的轮次之前，然后提出下一个问题。
  history = [
    { role: "assistant", content: summary.content },
    *history.drop(2),
    { role: "user", content: "Which day should the release go out?" }
  ]
  third = client.beta.messages.create(
    model: MODEL,
    max_tokens: 8192,
    system_: SYSTEM,
    betas: BETAS,
    thinking: THINKING,
    messages: history
  )

  # 4. 若返回 200 且没有块被丢弃，则说明保留的思考内容通过了检查。
  puts "Dropped thinking blocks: #{third.input_transformations.length}"
  ```
</CodeGroup>

```text Output wrap
Thinking blocks in the kept turn: 1
Dropped thinking blocks: 0
```

在生产环境中，当某个条件不成立时，`"drop_block"` 可以让请求继续成功，并在 `input_transformations` 中以 `reason: "prefix_binding_mismatch"` 报告每个被丢弃的块。如果某个条目的 `path` 位于保留轮次中，则表示该轮次的思考未能保持有效。[API 如何处理失效的块](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#mismatch-behavior)描述了哪些内容会被丢弃，并说明了如何针对此情况设置告警。
