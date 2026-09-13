---
title: 引导思考
url: https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost
description: 通过努力级别、系统提示指导和逐消息引导来控制 Claude 思考的频率和深度，并了解思考的成本和定价。
---

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

Claude 的思考是自适应的：模型会评估每个请求，并自行决定是否思考以及思考多少。您设定意图，可选地指定 effort（努力级别），模型会在它判断推理有帮助的地方分配推理。

这使得思考非常适合混合了简单请求和复杂请求的工作负载，也适合长周期的智能体工作流——在这类工作流中，合适的推理量会随步骤而变化。

要了解如何开启思考、如何读取思考输出，以及 [Claude Fable 5 和 Claude Mythos 5 上的思考输出](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-output-on-claude-fable-5-and-claude-mythos-5)，请参阅[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)概述。本页介绍 Claude 如何决定何时思考、如何引导该决定，以及由此产生的缓存、成本和定价机制。

## Claude 如何决定何时思考

对模型而言，思考是可选的。在每个请求上，Claude 会权衡输入的复杂度，并决定更深入的推理是否会改善答案。一个简单的事实性问题可能会得到直接回复，完全没有思考块；而一个多步骤数学问题或棘手的调试任务则会触发更深入的推理。

该决定按请求进行。同一对话中可以包含有思考和无思考的轮次，而 Claude 选择不思考的轮次不包含思考块。不要构建假设每个助手轮次都以思考块开头的应用逻辑。

对该决定的主要控制手段是 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 参数，它作为软性指导，决定 Claude 应有多愿意思考以及思考多深；有关每个级别的作用，请参阅本页的[努力级别](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#effort-levels)。

如果您希望 Claude 更少地思考，请先降低努力级别，再考虑基于提示的引导。

思考还会自动与 tool use（工具使用）交错进行：Claude 可以在工具调用之间思考，在决定下一步做什么之前反思每个工具结果（[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)）。您不需要 beta 标头或任何额外配置。

有关思考配置与 effort 参数如何交互的完整说明，请参阅[思考与努力级别](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-effort)。

## 引导 Claude 思考的频率

Claude 是否在某一轮次思考是可以通过提示控制的。Effort 设定整体姿态，但您也可以通过自然语言指导直接塑造该决定——既可以在 system prompt（系统提示）中全局设置，也可以在用户轮次中逐消息设置。

按以下顺序结合使用这两个手段：

1. 设置与您的工作负载在质量和延迟之间的默认平衡相匹配的努力级别。
2. 仅当 Claude 在该级别下的触发行为仍不符合您的需求时，才添加提示指导。

有关配合思考使用的更广泛提示指导，请参阅[利用思考和交错思考能力](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#leverage-thinking-and-interleaved-thinking-capabilities)。

### 努力级别

Effort 是思考的主要引导手段。每个级别为 Claude 思考的频率和深度设定不同的默认值：

| 努力级别       | 思考行为                            |
| ---------- | ------------------------------- |
| `max`      | Claude 始终思考，对思考深度没有限制。          |
| `xhigh`    | Claude 始终深入思考，并进行扩展探索。          |
| `high`（默认） | Claude 几乎总是思考。在复杂任务上提供深度推理。     |
| `medium`   | Claude 使用适度思考。对于简单查询可能跳过思考。     |
| `low`      | Claude 尽量减少思考。对于速度最重要的简单任务跳过思考。 |

此表描述了每个级别如何改变思考行为。有关针对特定工作负载应选择哪个级别的指导（包括按模型的建议），请参阅 effort 页面上的[何时调整 effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#when-to-adjust-the-effort-parameter)。

Effort 在 `output_config.effort` 处设置，而不是在 `thinking` 对象内部；有关各语言的完整示例，请参阅 [Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#basic-usage)。

```json
{
  "model": "claude-opus-5",
  "max_tokens": 4096,
  "output_config": { "effort": "medium" },
  "messages": [{ "role": "user", "content": "..." }]
}
```

级别可用性因模型而异；effort 页面上的 [effort 可用性表](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#effort-levels)是各模型支持哪些级别的权威来源。

### 系统提示指导

系统提示指导会改变对话中每个请求的 Claude 思考阈值。如果 Claude 思考的频率超出您的工作负载所需，请在系统提示中添加如下指导：

```text wrap
Extended thinking adds latency and should only be used when it
will meaningfully improve answer quality, typically for problems
that require multistep reasoning. When in doubt, respond directly.
```

若要反过来鼓励思考，请使用类似这样的短语：

```text wrap
This task involves multistep reasoning. Think carefully before responding.
```

引导效果可能对具体措辞敏感。如果某种表述没有产生您想要的行为，请尝试更直接的变体。

### 逐消息引导

您也可以在用户轮次中逐消息地引导思考，独立于系统提示。在用户消息末尾附加 `"Please think hard before responding."` 会鼓励 Claude 在该轮次思考；`"Answer directly without deliberating."` 则会抑制思考。

当对话中只有部分请求需要扩展推理时，逐消息引导非常有用。例如，智能体框架可以在规划步骤上附加鼓励性短语，在例行确认上附加抑制性短语，而无需触碰系统提示或在轮次之间更改任何请求参数。

### 在您的工作负载上验证引导效果

基于提示的引导会改变模型行为，因此请像对待任何其他提示更改一样对待它：在发布前进行测量。使用有代表性的流量样本分别在有指导和无指导的情况下运行，并比较思考触发的频率（响应中是否存在思考块）、输出令牌用量、延迟，以及在您关心的案例上的答案质量。

<Warning>
  引导 Claude 更少地思考可能会降低受益于推理的任务的质量。降低 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别通常是更好的首选手段，因为它是经过校准的控制，而不是对措辞敏感的指令。在将基于提示的调优部署到生产环境之前，请测量其对您特定工作负载的影响。
</Warning>

## 机制

由 Claude 自行管理思考引出三项机制：轮次验证、提示缓存，以及您如何限定成本。

### 轮次验证

助手轮次不需要以思考块开头。（使用旧版手动思考预算的模型会强制要求启用思考的请求的最后一个助手轮次以思考块开头；请参阅[手动模式下的轮次结构](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#turn-structure-in-manual-mode)。）

对于多轮应用，这意味着您可以按手头已有的任何形式传回对话历史：

* Claude 选择不思考的助手轮次按原样即为有效历史。
* 您可以恢复一个开始时没有思考、或使用了不同思考配置的对话，而无需重写其历史。
* 从混合来源组装的历史不需要在每个助手轮次开头重新插入思考块即可通过验证。

这一放宽是关于验证的，而不是关于您应该发送什么。当您有思考块时，请原样传回它们，尤其是在工具使用期间，因为它们承载着 Claude 工具调用背后的推理。完整规则请参阅[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)概述。

### 提示缓存

保持相同思考配置和努力级别的连续请求会保留 prompt caching（提示缓存）；完整规则请参阅[思考与提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-prompt-caching)。解析后的 effort 值会被渲染到提示中，因此在请求之间更改它会使缓存断点失效，就像在使用旧版 [`budget_tokens`](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#extended-thinking-with-prompt-caching) 参数的模型上更改该参数一样。将 `effort` 显式设置为模型的默认值等同于省略它，不会破坏缓存。

实际结论是：为每个对话选定一个思考配置和一个努力级别并保持不变。如果某些轮次需要更多或更少的思考，请使用[逐消息提示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#tuning-thinking-behavior)进行引导：附加到最新用户消息的指导会保持较早的缓存断点完好，而配置或 effort 更改则不会。

以下示例通过一个您可以自行运行的多轮脚本演示了这种失效：

<Accordion title="Effort 更改会使提示缓存失效">
  <Tabs>
    <Tab title="cURL">
      <Note>
        此工作流不太适合转换为一次性 shell 命令。多轮模式请参阅 SDK 选项卡；逐轮 HTTP 请求遵循[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)页面上的示例。
      </Note>
    </Tab>

    <Tab title="CLI">
      <Note>
        此工作流不太适合转换为一次性 shell 命令。多轮模式请参阅 SDK 选项卡；逐轮 CLI 调用遵循[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)页面上的示例。
      </Note>
    </Tab>

    <Tab title="Python">
      ```python
      import requests

      client = Anthropic()


      def fetch_article_content(url):
          text = requests.get(url).text
          lines = (line.strip() for line in text.splitlines())
          return "\n".join(line for line in lines if line)


      # 获取文章内容
      book_url = "https://www.gutenberg.org/cache/epub/1342/pg1342.txt"
      book_content = fetch_article_content(book_url)
      # 仅使用足够用于缓存的文本（前几章）
      LARGE_TEXT = book_content[:10000]

      # 无系统提示 - 改为在消息中缓存
      MESSAGES = [
          {
              "role": "user",
              "content": [
                  {
                      "type": "text",
                      "text": LARGE_TEXT,
                      "cache_control": {"type": "ephemeral"},
                  },
                  {"type": "text", "text": "Analyze the tone of this passage."},
              ],
          }
      ]

      # 第一次请求 - 建立缓存
      print("First request - establishing cache")
      response1 = client.messages.create(
          model="claude-opus-5",
          max_tokens=16000,
          thinking={"type": "adaptive"},
          messages=MESSAGES,
      )

      print(f"First response usage: {response1.usage}")

      MESSAGES.append({"role": "assistant", "content": response1.content})
      MESSAGES.append({"role": "user", "content": "Analyze the characters in this passage."})

      # 第二次请求 - 相同配置（预期缓存命中）
      print("\nSecond request - same configuration (cache hit expected)")
      response2 = client.messages.create(
          model="claude-opus-5",
          max_tokens=16000,
          thinking={"type": "adaptive"},
          messages=MESSAGES,
      )

      print(f"Second response usage: {response2.usage}")

      MESSAGES.append({"role": "assistant", "content": response2.content})
      MESSAGES.append({"role": "user", "content": "Analyze the setting in this passage."})

      # 第三次请求 - 不同的 effort 级别（预期缓存未命中）
      print("\nThird request - different effort level (cache miss expected)")
      response3 = client.messages.create(
          model="claude-opus-5",
          max_tokens=16000,
          thinking={"type": "adaptive"},
          output_config={"effort": "medium"},
          messages=MESSAGES,
      )

      print(f"Third response usage: {response3.usage}")
      ```
    </Tab>

    <Tab title="TypeScript">
      ```typescript

      const client = new Anthropic();

      async function fetchArticleContent(url: string): Promise<string> {
        const response = await fetch(url);
        const text = await response.text();
        const lines = text.split("\n").map((line) => line.trim());
        return lines.filter((line) => line).join("\n");
      }

      const bookUrl = "https://www.gutenberg.org/cache/epub/1342/pg1342.txt";
      const bookContent = await fetchArticleContent(bookUrl);
      const LARGE_TEXT = bookContent.substring(0, 10000);

      // 无系统提示 - 改为在消息中进行缓存
      const messages: Anthropic.MessageParam[] = [
        {
          role: "user",
          content: [
            {
              type: "text",
              text: LARGE_TEXT,
              cache_control: { type: "ephemeral" }
            },
            {
              type: "text",
              text: "Analyze the tone of this passage."
            }
          ]
        }
      ];

      // 第一次请求 - 建立缓存
      console.log("First request - establishing cache");
      const response1 = await client.messages.create({
        model: "claude-opus-5",
        max_tokens: 16000,
        thinking: { type: "adaptive" },
        messages
      });

      console.log("First response usage: ", response1.usage);

      messages.push(
        { role: "assistant", content: response1.content },
        { role: "user", content: "Analyze the characters in this passage." }
      );

      // 第二次请求 - 相同配置（预期缓存命中）
      console.log("\nSecond request - same configuration (cache hit expected)");
      const response2 = await client.messages.create({
        model: "claude-opus-5",
        max_tokens: 16000,
        thinking: { type: "adaptive" },
        messages
      });

      console.log("Second response usage: ", response2.usage);

      messages.push(
        { role: "assistant", content: response2.content },
        { role: "user", content: "Analyze the setting in this passage." }
      );

      // 第三次请求 - 不同的 effort 级别（预期缓存未命中）
      console.log("\nThird request - different effort level (cache miss expected)");
      const response3 = await client.messages.create({
        model: "claude-opus-5",
        max_tokens: 16000,
        thinking: { type: "adaptive" },
        output_config: { effort: "medium" },
        messages
      });

      console.log("Third response usage: ", response3.usage);
      ```
    </Tab>

    <Tab title="C#">
      ```csharp
      AnthropicClient client = new();

      string bookUrl = "https://www.gutenberg.org/cache/epub/1342/pg1342.txt";
      string bookContent = await FetchArticleContent(bookUrl);
      string largeText = bookContent.Substring(0, Math.Min(10000, bookContent.Length));

      Console.WriteLine("First request - establishing cache");
      var parameters1 = new MessageCreateParams
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 16000,
          Thinking = new ThinkingConfigAdaptive(),
          Messages =
          [
              new()
              {
                  Role = Role.User,
                  Content = new MessageParamContent(new List<ContentBlockParam>
                  {
                      new ContentBlockParam(new TextBlockParam()
                      {
                          Text = largeText,
                          CacheControl = new CacheControlEphemeral(),
                      }),
                      new ContentBlockParam(new TextBlockParam()
                      {
                          Text = "Analyze the tone of this passage."
                      }),
                  })
              }
          ]
      };

      var response1 = await client.Messages.Create(parameters1);
      Console.WriteLine($"First response usage: {response1.Usage}");

      Console.WriteLine("\nSecond request - same configuration (cache hit expected)");
      var parameters2 = new MessageCreateParams
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 16000,
          Thinking = new ThinkingConfigAdaptive(),
          Messages =
          [
              new()
              {
                  Role = Role.User,
                  Content = new MessageParamContent(new List<ContentBlockParam>
                  {
                      new ContentBlockParam(new TextBlockParam()
                      {
                          Text = largeText,
                          CacheControl = new CacheControlEphemeral(),
                      }),
                      new ContentBlockParam(new TextBlockParam()
                      {
                          Text = "Analyze the tone of this passage."
                      }),
                  })
              },
              new()
              {
                  Role = Role.Assistant,
                  Content = response1.Content.Select(block => new ContentBlockParam(block.Json)).ToList()
              },
              new()
              {
                  Role = Role.User,
                  Content = "Analyze the characters in this passage."
              }
          ]
      };

      var response2 = await client.Messages.Create(parameters2);
      Console.WriteLine($"Second response usage: {response2.Usage}");

      Console.WriteLine("\nThird request - different effort level (cache miss expected)");
      var parameters3 = new MessageCreateParams
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 16000,
          Thinking = new ThinkingConfigAdaptive(),
          OutputConfig = new OutputConfig
          {
              Effort = Effort.Medium
          },
          Messages =
          [
              new()
              {
                  Role = Role.User,
                  Content = new MessageParamContent(new List<ContentBlockParam>
                  {
                      new ContentBlockParam(new TextBlockParam()
                      {
                          Text = largeText,
                          CacheControl = new CacheControlEphemeral(),
                      }),
                      new ContentBlockParam(new TextBlockParam()
                      {
                          Text = "Analyze the tone of this passage."
                      }),
                  })
              },
              new()
              {
                  Role = Role.Assistant,
                  Content = response1.Content.Select(block => new ContentBlockParam(block.Json)).ToList()
              },
              new()
              {
                  Role = Role.User,
                  Content = "Analyze the characters in this passage."
              },
              new()
              {
                  Role = Role.Assistant,
                  Content = response2.Content.Select(block => new ContentBlockParam(block.Json)).ToList()
              },
              new()
              {
                  Role = Role.User,
                  Content = "Analyze the setting in this passage."
              }
          ]
      };

      var response3 = await client.Messages.Create(parameters3);
      Console.WriteLine($"Third response usage: {response3.Usage}");

      static async Task<string> FetchArticleContent(string url)
      {
          using HttpClient httpClient = new();
          string content = await httpClient.GetStringAsync(url);
          return content;
      }
      ```
    </Tab>

    <Tab title="Go">
      ```go
      client := anthropic.NewClient()

      bookURL := "https://www.gutenberg.org/cache/epub/1342/pg1342.txt"
      bookContent, err := fetchArticleContent(bookURL)
      if err != nil {
      	log.Fatal(err)
      }

      largeText := bookContent
      if len(largeText) > 10000 {
      	largeText = largeText[:10000]
      }

      // 无系统提示 - 改为在消息中进行缓存
      messages := []anthropic.MessageParam{
      	anthropic.NewUserMessage(
      		anthropic.ContentBlockParamUnion{OfText: &anthropic.TextBlockParam{
      			Text:         largeText,
      			CacheControl: anthropic.NewCacheControlEphemeralParam(),
      		}},
      		anthropic.NewTextBlock("Analyze the tone of this passage."),
      	),
      }

      // 第一次请求 - 建立缓存
      fmt.Println("First request - establishing cache")
      response1, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
      	Model:     anthropic.ModelClaudeOpus5,
      	MaxTokens: 16000,
      	Thinking: anthropic.ThinkingConfigParamUnion{
      		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
      	},
      	Messages: messages,
      })
      if err != nil {
      	log.Fatal(err)
      }
      fmt.Printf("First response usage: %s\n", response1.Usage.RawJSON())

      messages = append(messages, response1.ToParam())
      messages = append(messages, anthropic.NewUserMessage(anthropic.NewTextBlock("Analyze the characters in this passage.")))

      // 第二次请求 - 相同配置（预期缓存命中）
      fmt.Println("\nSecond request - same configuration (cache hit expected)")
      response2, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
      	Model:     anthropic.ModelClaudeOpus5,
      	MaxTokens: 16000,
      	Thinking: anthropic.ThinkingConfigParamUnion{
      		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
      	},
      	Messages: messages,
      })
      if err != nil {
      	log.Fatal(err)
      }
      fmt.Printf("Second response usage: %s\n", response2.Usage.RawJSON())

      messages = append(messages, response2.ToParam())
      messages = append(messages, anthropic.NewUserMessage(anthropic.NewTextBlock("Analyze the setting in this passage.")))

      // 第三次请求 - 不同的 effort 级别（预期缓存未命中）
      fmt.Println("\nThird request - different effort level (cache miss expected)")
      response3, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
      	Model:     anthropic.ModelClaudeOpus5,
      	MaxTokens: 16000,
      	Thinking: anthropic.ThinkingConfigParamUnion{
      		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
      	},
      	OutputConfig: anthropic.OutputConfigParam{
      		Effort: anthropic.OutputConfigEffortMedium,
      	},
      	Messages: messages,
      })
      if err != nil {
      	log.Fatal(err)
      }
      fmt.Printf("Third response usage: %s\n", response3.Usage.RawJSON())
      ```
    </Tab>

    <Tab title="Java">
      ```java
      import com.anthropic.models.messages.CacheControlEphemeral;
      // ...
      void main() throws Exception {
          AnthropicClient client = AnthropicOkHttpClient.fromEnv();

          String bookUrl = "https://www.gutenberg.org/cache/epub/1342/pg1342.txt";
          String bookContent = fetchArticleContent(bookUrl);
          String largeText = bookContent.substring(0, Math.min(10000, bookContent.length()));

          // 第一次请求 - 建立缓存
          IO.println("First request - establishing cache");
          MessageCreateParams params1 = MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5)
              .maxTokens(16000L)
              .thinking(ThinkingConfigAdaptive.builder().build())
              .addUserMessageOfBlockParams(List.of(
                  ContentBlockParam.ofText(TextBlockParam.builder()
                      .text(largeText)
                      .cacheControl(CacheControlEphemeral.builder().build())
                      .build()),
                  ContentBlockParam.ofText(TextBlockParam.builder()
                      .text("Analyze the tone of this passage.")
                      .build())
              ))
              .build();

          Message response1 = client.messages().create(params1);
          IO.println("First response usage: " + response1.usage());

          // 第二次请求 - 相同配置（预期缓存命中）
          IO.println("\nSecond request - same configuration (cache hit expected)");
          MessageCreateParams params2 = MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5)
              .maxTokens(16000L)
              .thinking(ThinkingConfigAdaptive.builder().build())
              .addUserMessageOfBlockParams(List.of(
                  ContentBlockParam.ofText(TextBlockParam.builder()
                      .text(largeText)
                      .cacheControl(CacheControlEphemeral.builder().build())
                      .build()),
                  ContentBlockParam.ofText(TextBlockParam.builder()
                      .text("Analyze the tone of this passage.")
                      .build())
              ))
              .addAssistantMessageOfBlockParams(response1.content().stream()
                  .map(block -> block.toParam())
                  .collect(java.util.stream.Collectors.toList()))
              .addUserMessage("Analyze the characters in this passage.")
              .build();

          Message response2 = client.messages().create(params2);
          IO.println("Second response usage: " + response2.usage());

          // 第三次请求 - 不同的 effort 级别（预期缓存未命中）
          IO.println("\nThird request - different effort level (cache miss expected)");
          MessageCreateParams params3 = MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5)
              .maxTokens(16000L)
              .thinking(ThinkingConfigAdaptive.builder().build())
              .outputConfig(OutputConfig.builder()
                  .effort(OutputConfig.Effort.MEDIUM)
                  .build())
              .addUserMessageOfBlockParams(List.of(
                  ContentBlockParam.ofText(TextBlockParam.builder()
                      .text(largeText)
                      .cacheControl(CacheControlEphemeral.builder().build())
                      .build()),
                  ContentBlockParam.ofText(TextBlockParam.builder()
                      .text("Analyze the tone of this passage.")
                      .build())
              ))
              .addAssistantMessageOfBlockParams(response1.content().stream()
                  .map(block -> block.toParam())
                  .collect(java.util.stream.Collectors.toList()))
              .addUserMessage("Analyze the characters in this passage.")
              .addAssistantMessageOfBlockParams(response2.content().stream()
                  .map(block -> block.toParam())
                  .collect(java.util.stream.Collectors.toList()))
              .addUserMessage("Analyze the setting in this passage.")
              .build();

          Message response3 = client.messages().create(params3);
          IO.println("Third response usage: " + response3.usage());
      }

      String fetchArticleContent(String url) throws Exception {
          HttpClient client = HttpClient.newHttpClient();
          HttpRequest request = HttpRequest.newBuilder()
              .uri(URI.create(url))
              .build();
          HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
          return response.body();
      }
      ```
    </Tab>

    <Tab title="PHP">
      ```php
      function fetchArticleContent($url) {
          $content = file_get_contents($url);
          $lines = explode("\n", $content);
          $cleanedLines = array_filter(array_map('trim', $lines));
          return implode("\n", $cleanedLines);
      }

      $client = new Client();

      $bookUrl = "https://www.gutenberg.org/cache/epub/1342/pg1342.txt";
      $bookContent = fetchArticleContent($bookUrl);
      $largeText = substr($bookContent, 0, 10000);

      echo "First request - establishing cache\n";
      $response1 = $client->messages->create(
          maxTokens: 16000,
          messages: [[
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'text',
                      'text' => $largeText,
                      'cache_control' => ['type' => 'ephemeral']
                  ],
                  [
                      'type' => 'text',
                      'text' => 'Analyze the tone of this passage.'
                  ]
              ]
          ]],
          model: 'claude-opus-5',
          thinking: ['type' => 'adaptive'],
      );

      echo "First response usage: " . json_encode($response1->usage) . "\n";

      echo "\nSecond request - same configuration (cache hit expected)\n";
      $response2 = $client->messages->create(
          maxTokens: 16000,
          messages: [
              [
                  'role' => 'user',
                  'content' => [
                      [
                          'type' => 'text',
                          'text' => $largeText,
                          'cache_control' => ['type' => 'ephemeral']
                      ],
                      [
                          'type' => 'text',
                          'text' => 'Analyze the tone of this passage.'
                      ]
                  ]
              ],
              [
                  'role' => 'assistant',
                  'content' => $response1->content
              ],
              [
                  'role' => 'user',
                  'content' => 'Analyze the characters in this passage.'
              ]
          ],
          model: 'claude-opus-5',
          thinking: ['type' => 'adaptive'],
      );

      echo "Second response usage: " . json_encode($response2->usage) . "\n";

      echo "\nThird request - different effort level (cache miss expected)\n";
      $response3 = $client->messages->create(
          maxTokens: 16000,
          messages: [
              [
                  'role' => 'user',
                  'content' => [
                      [
                          'type' => 'text',
                          'text' => $largeText,
                          'cache_control' => ['type' => 'ephemeral']
                      ],
                      [
                          'type' => 'text',
                          'text' => 'Analyze the tone of this passage.'
                      ]
                  ]
              ],
              [
                  'role' => 'assistant',
                  'content' => $response1->content
              ],
              [
                  'role' => 'user',
                  'content' => 'Analyze the characters in this passage.'
              ],
              [
                  'role' => 'assistant',
                  'content' => $response2->content
              ],
              [
                  'role' => 'user',
                  'content' => 'Analyze the setting in this passage.'
              ]
          ],
          model: 'claude-opus-5',
          thinking: ['type' => 'adaptive'],
          outputConfig: ['effort' => 'medium'],
      );

      echo "Third response usage: " . json_encode($response3->usage) . "\n";
      ```
    </Tab>

    <Tab title="Ruby">
      ```ruby
      require "net/http"
      require "uri"

      def fetch_article_content(url)
        uri = URI.parse(url)
        response = Net::HTTP.get_response(uri)
        text = response.body

        lines = text.split("\n").map(&:strip)
        lines.reject(&:empty?).join("\n")
      end

      client = Anthropic::Client.new

      book_url = "https://www.gutenberg.org/cache/epub/1342/pg1342.txt"
      book_content = fetch_article_content(book_url)
      large_text = book_content[0...10000]

      puts "First request - establishing cache"
      response1 = client.messages.create(
        model: "claude-opus-5",
        max_tokens: 16000,
        thinking: {
          type: "adaptive"
        },
        messages: [{
          role: "user",
          content: [
            {
              type: "text",
              text: large_text,
              cache_control: { type: "ephemeral" }
            },
            {
              type: "text",
              text: "Analyze the tone of this passage."
            }
          ]
        }]
      )

      puts "First response usage: #{response1.usage}"

      puts "\nSecond request - same configuration (cache hit expected)"
      response2 = client.messages.create(
        model: "claude-opus-5",
        max_tokens: 16000,
        thinking: {
          type: "adaptive"
        },
        messages: [
          {
            role: "user",
            content: [
              {
                type: "text",
                text: large_text,
                cache_control: { type: "ephemeral" }
              },
              {
                type: "text",
                text: "Analyze the tone of this passage."
              }
            ]
          },
          {
            role: "assistant",
            content: response1.content
          },
          {
            role: "user",
            content: "Analyze the characters in this passage."
          }
        ]
      )

      puts "Second response usage: #{response2.usage}"

      puts "\nThird request - different effort level (cache miss expected)"
      response3 = client.messages.create(
        model: "claude-opus-5",
        max_tokens: 16000,
        thinking: {
          type: "adaptive"
        },
        output_config: {
          effort: "medium"
        },
        messages: [
          {
            role: "user",
            content: [
              {
                type: "text",
                text: large_text,
                cache_control: { type: "ephemeral" }
              },
              {
                type: "text",
                text: "Analyze the tone of this passage."
              }
            ]
          },
          {
            role: "assistant",
            content: response1.content
          },
          {
            role: "user",
            content: "Analyze the characters in this passage."
          },
          {
            role: "assistant",
            content: response2.content
          },
          {
            role: "user",
            content: "Analyze the setting in this passage."
          }
        ]
      )

      puts "Third response usage: #{response3.usage}"
      ```
    </Tab>
  </Tabs>

  以下是脚本的输出（您看到的数字可能略有不同）：

  ```text Output wrap
  First request - establishing cache
  First response usage: { cache_creation_input_tokens: 3546, cache_read_input_tokens: 0, input_tokens: 15, output_tokens: 1033 }

  Second request - same configuration (cache hit expected)
  Second response usage: { cache_creation_input_tokens: 0, cache_read_input_tokens: 3546, input_tokens: 1062, output_tokens: 1630 }

  Third request - different effort level (cache miss expected)
  Third response usage: { cache_creation_input_tokens: 3546, cache_read_input_tokens: 0, input_tokens: 2706, output_tokens: 1468 }
  ```

  由于缓存断点位于 messages 数组中，将 effort 从默认的 `high` 更改为 `medium` 会使其失效：第三个请求显示 `cache_creation_input_tokens=3546` 和 `cache_read_input_tokens=0`，而第二个请求显示的是完整的缓存读取。
</Accordion>

### 成本控制

您无需设置思考令牌预算。有两个控制手段限定成本：

* `max_tokens` 是请求总输出（思考和响应文本合计）的硬性上限。Claude 绝不会生成超过它的内容。在工具使用循环中，轮次中的每个请求都有自己的 `max_tokens`，因此它不会限定整个轮次的开销。
* `effort` 是关于 Claude 将多少输出分配给思考的软性指导。它塑造行为，但不保证令牌数量。

由于思考计入 `max_tokens`，请将其设置得足够高，为推理和答案都留出空间。一个按无思考响应设定大小的 `max_tokens`，一旦 Claude 开始在困难请求上思考，往往就太小了。

在 `high` 及以上的努力级别下，Claude 可能会进行大量思考，更有可能耗尽预算。如果您在响应中看到 [`stop_reason: "max_tokens"`](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#stopped-at-max-tokens)，您有两种补救办法：

* 提高 `max_tokens`，为模型的思考加答案提供更多空间。
* 降低努力级别，使 Claude 减少思考，为响应文本留出更多预算。

哪一种合适取决于被截断的响应是否需要推理。如果这些请求的质量很重要，请提高上限；如果它们被过度思考了，请降低 effort。

## 定价

思考会产生以下费用：

* Claude 思考时使用的令牌（按输出令牌计费）
* 根据[保留默认值](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)保留在上下文中的先前助手轮次的思考块：在 keep-all 模型上默认为所有轮次，在其他模型上仅为最后一个轮次（按输入令牌计费）
* 标准文本输出令牌

<Note>
  当思考处于活动状态时，会自动包含一个专门的系统提示以支持此功能。
</Note>

无论 `display` 设置如何，您被计费的内容都相同；只有您看到的内容会改变：

|              | `display: "summarized"` | `display: "omitted"`   |
| ------------ | ----------------------- | ---------------------- |
| **输入令牌**     | 您原始请求中的令牌               | 与 summarized 相同        |
| **输出令牌（计费）** | Claude 内部生成的完整思考令牌      | 与 summarized 相同        |
| **输出令牌（可见）** | 摘要后的思考文本                | 零思考令牌（`thinking` 字段为空） |
| **摘要生成**     | 不收费                     | 不适用                    |

<Warning>
  计费的输出令牌数与响应中可见的令牌数**不**匹配。您按完整的思考过程计费，而不是按响应中可见的思考内容计费。
</Warning>

要查看有多少计费输出令牌用于内部推理，请读取响应中的 `usage.output_tokens_details.thinking_tokens`。该值反映模型生成的原始推理（而非正文中返回的摘要文本），并且始终小于或等于 `output_tokens`。从 `output_tokens` 中减去它即可近似得出输出中的非推理部分。在 streaming（流式传输）时，此细分仅出现在最终的 `message_delta` 事件上。

```json
{
  "usage": {
    "input_tokens": 25,
    "output_tokens": 348,
    "output_tokens_details": {
      "thinking_tokens": 312
    }
  }
}
```

`output_tokens` 仍然是用于计费的包含一切的权威总数。`output_tokens_details` 是用于可观测性的只读细分。有关包括基础费率、缓存写入、缓存命中和输出令牌在内的完整定价信息，请参阅[定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

## 后续步骤

<CardGroup cols={3}>
  <Card title="思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    开启思考、读取思考输出，并查看各模型的支持情况。
  </Card>

  <Card title="工具和多轮工作流中的思考" icon="wrench" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-tool-workflows">
    跨工具调用保留思考块，并在多轮对话中管理思考。
  </Card>

  <Card title="Effort" icon="sliders" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    控制 Claude 为每个请求分配多少思考和输出。
  </Card>
</CardGroup>
