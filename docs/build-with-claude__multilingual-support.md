---
title: 多语言支持
url: https://platform.claude.com/docs/zh-CN/build-with-claude/multilingual-support
description: Claude 在多种语言的任务中表现出色，相对于英语保持了强大的跨语言性能。
---

## 概述

Claude 展现出强大的多语言能力，在跨语言的 "zero-shot"（零样本）任务中表现尤为出色。该模型在广泛使用的语言和资源较少的语言中均保持一致的相对性能，使其成为多语言应用的可靠选择。

除下表中进行基准测试的语言之外，Claude 还能够处理许多其他语言。请使用与您的具体用例相关的任何语言进行测试。

## 性能数据

下表显示了 Claude 模型在各语言中的零样本 "chain-of-thought"（思维链）评估得分，以相对于英语性能（100%）的百分比表示：

| 语言              | Claude Sonnet 4.51 | Claude Haiku 4.51 |
| --------------- | ------------------ | ----------------- |
| 英语（基准，固定为 100%） | 100%               | 100%              |
| 西班牙语            | 98.2%              | 96.4%             |
| 葡萄牙语（巴西）        | 97.8%              | 96.1%             |
| 意大利语            | 97.9%              | 96.0%             |
| 法语              | 97.5%              | 95.7%             |
| 印度尼西亚语          | 97.3%              | 94.2%             |
| 德语              | 97.0%              | 94.3%             |
| 阿拉伯语            | 97.2%              | 92.5%             |
| 中文（简体）          | 96.9%              | 94.2%             |
| 韩语              | 96.7%              | 93.3%             |
| 日语              | 96.8%              | 93.5%             |
| 印地语             | 96.7%              | 92.4%             |
| 孟加拉语            | 95.4%              | 90.4%             |
| 斯瓦希里语           | 91.1%              | 78.3%             |
| 约鲁巴语            | 79.7%              | 52.7%             |

1 使用 [extended thinking（扩展思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)。

<Note>
  这些指标基于 [MMLU（Massive Multitask Language Understanding，大规模多任务语言理解）](https://en.wikipedia.org/wiki/MMLU) 英语测试集，这些测试集由专业人工译者翻译成另外 14 种语言，详见 [OpenAI 的 simple-evals 代码仓库](https://github.com/openai/simple-evals/blob/main/multilingual_mmlu_benchmark_results.md)。在此评估中使用人工译者可确保高质量的翻译，这对于数字资源较少的语言尤为重要。
</Note>

***

## 设置响应语言

Claude 会从对话中推断响应语言，但对于生产应用，您应明确指定目标语言。最可靠的做法是在 "system prompt"（系统提示）中进行指定，这样可以使该指令在对话的每一轮中保持稳定。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "system": "Always respond in French, regardless of the language the user writes in.",
      "messages": [
        {"role": "user", "content": "How do I reset my password?"}
      ]
    }'
  ```

  ```bash CLI
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --system "Always respond in French, regardless of the language the user writes in." \
    --message '{role: user, content: "How do I reset my password?"}'
  ```

  ```python Python
  client = anthropic.Anthropic()

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      system="Always respond in French, regardless of the language the user writes in.",
      messages=[{"role": "user", "content": "How do I reset my password?"}],
  )

  print(message.content)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const message = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    system: "Always respond in French, regardless of the language the user writes in.",
    messages: [{ role: "user", content: "How do I reset my password?" }]
  });

  console.log(message.content);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      System = "Always respond in French, regardless of the language the user writes in.",
      Messages =
      [
          new() { Role = Role.User, Content = "How do I reset my password?" }
      ]
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	System: []anthropic.TextBlockParam{
  		{Text: "Always respond in French, regardless of the language the user writes in."},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("How do I reset my password?")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(message.Content)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(1024)
      .system("Always respond in French, regardless of the language the user writes in.")
      .addUserMessage("How do I reset my password?")
      .build();

  Message message = client.messages().create(params);
  System.out.println(message.content());
  ```

  ```php PHP
  $client = new Client();

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          ['role' => 'user', 'content' => 'How do I reset my password?']
      ],
      model: 'claude-opus-5',
      system: 'Always respond in French, regardless of the language the user writes in.',
  );

  echo json_encode($message->content, JSON_PRETTY_PRINT), PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    system: "Always respond in French, regardless of the language the user writes in.",
    messages: [
      { role: "user", content: "How do I reset my password?" }
    ]
  )

  puts message.content
  ```
</CodeGroup>

如果您的应用允许用户在运行时选择语言，请将该选择插入到系统提示中，而不是依赖 Claude 从用户消息中推断。若要在两种特定语言之间进行翻译，请同时指明两种语言：`Translate the user's message from German to Korean. Respond with only the translation.`

***

## 最佳实践

处理多语言内容时：

1. **提供清晰的语言上下文：** 尽管 Claude 可以自动检测目标语言，但明确说明所需的输入和输出语言可以提高可靠性。为了增强流畅度，您可以提示 Claude 使用"如同母语者一般的地道表达"。
2. **使用原生文字：** 提交文本时请使用其原生文字而非音译，以获得最佳效果。
3. **考虑文化背景：** 有效的沟通通常需要超越纯粹翻译的文化和地区意识。

另请遵循[提示工程概述](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/overview)中的通用指导，以进一步提升输出质量。

***

## 语言支持注意事项

* Claude 可以处理大多数使用标准 Unicode 字符的世界语言的输入并生成输出。
* 性能因语言而异，在广泛使用的语言中能力尤为突出。
* 即使在数字资源较少的语言中，Claude 仍保持有意义的能力。

## 后续步骤

<CardGroup cols={2}>
  <Card title="提示工程概述" icon="edit" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/overview">
    应用通用提示技巧以提升多语言输出质量。
  </Card>

  <Card title="客户支持智能体" icon="headset" href="https://platform.claude.com/docs/zh-CN/about-claude/use-case-guides/customer-support-chat">
    使用限定语言的系统提示构建本地化的支持聊天机器人。
  </Card>

  <Card title="模型概述" icon="table" href="https://platform.claude.com/docs/zh-CN/models/overview">
    比较各模型层级，在多语言质量与成本和延迟之间取得平衡。
  </Card>

  <Card title="定义成功标准并构建评估" icon="scales" href="https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests">
    在发布之前评估翻译和本地化质量。
  </Card>
</CardGroup>
