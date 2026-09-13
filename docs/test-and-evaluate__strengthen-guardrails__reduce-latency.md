---
title: 降低延迟
url: https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/reduce-latency
description: 通过选择更快的模型（如 Claude Haiku 4.5）、精简提示和输出令牌以及流式传输响应，来降低 Claude 的响应延迟。
---

"Latency"（延迟）是指模型处理提示并生成输出所需的时间。延迟可能受多种因素影响，例如模型的大小、提示的复杂程度，以及支撑模型和交互点的底层基础设施。

<Note>
  最好先在不受模型或提示限制的情况下设计出一个效果良好的提示，然后再尝试降低延迟的策略。过早地尝试降低延迟可能会妨碍您发现最佳性能的样子。
</Note>

***

## 如何衡量延迟

在讨论延迟时，您可能会遇到几个术语和衡量指标：

* **基线延迟（Baseline latency）：** 这是模型处理提示并生成响应所花费的时间，不考虑每秒的输入和输出令牌数。它提供了对模型速度的总体认识。
* **首令牌时间（Time to first token，TTFT）：** 该指标衡量从发送提示到模型生成响应的第一个令牌所需的时间。当您使用 "streaming"（流式传输）（稍后会详细介绍）并希望为用户提供响应迅速的体验时，这一指标尤为重要。

如需更深入地了解这些术语，请查看[术语表](https://platform.claude.com/docs/zh-CN/about-claude/glossary)。

***

## 如何降低延迟

### 1. 选择合适的模型

降低延迟最直接的方法之一是为您的用例选择合适的模型。Anthropic 提供了[一系列模型](https://platform.claude.com/docs/zh-CN/models/overview)，它们具有不同的能力和性能特征。请考虑您的具体需求，并在速度和输出质量方面选择最符合您需要的模型。

对于速度至关重要的应用，**Claude Haiku 4.5** 在保持高智能水平的同时提供最快的响应时间：

<CodeGroup>
  ```bash cURL
  # 对于时间敏感型应用，请使用 Claude Haiku 4.5
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-haiku-4-5",
      "max_tokens": 100,
      "messages": [{"role": "user", "content": "Summarize this customer feedback in 2 sentences: [feedback text]"}]
    }'
  ```

  ```bash CLI
  # 对于时间敏感的应用，请使用 Claude Haiku 4.5
  ant messages create \
    --model claude-haiku-4-5 \
    --max-tokens 100 \
    --message '{"role": "user", "content": "Summarize this customer feedback in 2 sentences: [feedback text]"}'
  ```

  ```python Python
  client = anthropic.Anthropic()

  # 对于时间敏感的应用，请使用 Claude Haiku 4.5
  message = client.messages.create(
      model="claude-haiku-4-5",
      max_tokens=100,
      messages=[
          {
              "role": "user",
              "content": "Summarize this customer feedback in 2 sentences: [feedback text]",
          }
      ],
  )
  print(message.content[0].text)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // 对于时间敏感的应用，请使用 Claude Haiku 4.5
  const message = await client.messages.create({
    model: "claude-haiku-4-5",
    max_tokens: 100,
    messages: [
      {
        role: "user",
        content: "Summarize this customer feedback in 2 sentences: [feedback text]"
      }
    ]
  });
  const textBlock = message.content.find((block) => block.type === "text");
  console.log(textBlock?.text);
  ```

  ```csharp C#
  AnthropicClient client = new();

  // 对于时间敏感的应用，请使用 Claude Haiku 4.5
  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeHaiku4_5,
      MaxTokens = 100,
      Messages = [
          new()
          {
              Role = Role.User,
              Content = "Summarize this customer feedback in 2 sentences: [feedback text]"
          }
      ]
  };
  var message = await client.Messages.Create(parameters);
  message.Content[0].TryPickText(out var textBlock);
  Console.WriteLine(textBlock?.Text);
  ```

  ```go Go
  client := anthropic.NewClient()

  // 对于时间敏感的应用，请使用 Claude Haiku 4.5
  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeHaiku4_5,
  	MaxTokens: 100,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Summarize this customer feedback in 2 sentences: [feedback text]")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(message.Content[0].Text)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  // 对于时间敏感的应用，请使用 Claude Haiku 4.5
  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_HAIKU_4_5)
      .maxTokens(100L)
      .addUserMessage("Summarize this customer feedback in 2 sentences: [feedback text]")
      .build();
  Message message = client.messages().create(params);
  IO.println(message.content().get(0).text().map(TextBlock::text).orElse(""));
  ```

  ```php PHP
  $client = new Client();

  // 对于时间敏感的应用，请使用 Claude Haiku 4.5
  $message = $client->messages->create(
      maxTokens: 100,
      messages: [['role' => 'user', 'content' => 'Summarize this customer feedback in 2 sentences: [feedback text]']],
      model: 'claude-haiku-4-5',
  );
  echo $message->content[0]->text;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 对于时间敏感的应用，请使用 Claude Haiku 4.5
  message = client.messages.create(
    model: "claude-haiku-4-5",
    max_tokens: 100,
    messages: [{ role: "user", content: "Summarize this customer feedback in 2 sentences: [feedback text]" }]
  )
  puts message.content.first.text
  ```
</CodeGroup>

有关模型指标的更多详细信息，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)页面。

### 2. 优化提示和输出长度

在保持高性能的同时，尽量减少输入提示和预期输出中的令牌数量。模型需要处理和生成的令牌越少，响应速度就越快。

以下是一些帮助您优化提示和输出的技巧：

* **清晰而简洁：** 力求在提示中清晰、简洁地传达您的意图。避免不必要的细节或冗余信息，同时请记住，[Claude 缺乏](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#be-clear-and-direct)关于您用例的上下文，如果指令不清晰，它可能无法做出您预期的逻辑推断。
* **要求更简短的响应：** 直接要求 Claude 保持简洁。如果 Claude 输出的内容过长，请要求 Claude [减少啰嗦](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#be-clear-and-direct)。
  <Tip>
    由于 LLM 计算的是

    [令牌](https://platform.claude.com/docs/zh-CN/about-claude/glossary#tokens)

    而非单词，因此要求精确的字数或字数限制，不如要求段落数或句子数限制那样有效。
  </Tip>
* **设置适当的输出限制：** 使用 `max_tokens` 参数为生成响应的最大长度设置硬性限制。这可以防止 Claude 生成过长的输出。
  <Note>
    当响应达到 

    `max_tokens`

     个令牌时，响应将被截断，可能会在句子中间或单词中间被切断，因此这是一种较为粗暴的技术，可能需要后处理，通常最适用于答案出现在开头的多项选择或简答类响应。
  </Note>
* **尝试调整 temperature：** `temperature` [参数](https://platform.claude.com/docs/zh-CN/api/messages/create)控制输出的随机性。较低的值（例如 0.2）有时会产生更聚焦、更简短的响应，而较高的值（例如 0.8）可能会产生更多样化但可能更长的输出。

在提示清晰度、输出质量和令牌数量之间找到合适的平衡可能需要一些实验。

### 3. 流式传输响应

流式传输是一项允许模型在完整输出完成之前就开始返回响应的功能。这可以显著提升应用程序的感知响应速度，因为用户可以实时看到模型的输出。

启用流式传输后，您可以在模型输出到达时对其进行处理，同时更新用户界面或并行执行其他任务。

请访问[流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)，了解如何为您的用例实现流式传输。

***

## 后续步骤

<CardGroup cols={2}>
  <Card title="减少幻觉" icon="shield" href="https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/reduce-hallucinations">
    通过允许表达不确定性、以直接引用为响应依据以及通过引文验证论断，最大限度地减少 Claude 输出中的幻觉。
  </Card>

  <Card title="流式传输消息" icon="bolt" href="https://platform.claude.com/docs/zh-CN/build-with-claude/streaming">
    通过服务器发送事件（server-sent events）增量流式传输 Messages API 响应，包括文本、工具使用和扩展思考增量。
  </Card>
</CardGroup>
