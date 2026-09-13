---
title: Effort
url: https://platform.claude.com/docs/zh-CN/build-with-claude/effort
description: 使用 effort 参数控制 Claude 在响应时使用的令牌数量，在响应的全面性与令牌效率之间进行权衡。
---

## Compatibility
- [ZDR](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): eligible (excludes [Covered Models](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#model-specific-data-retention-requirements))
- Supported models: `claude-fable-5-1`, `claude-mythos-5-1`, `claude-fable-5`, `claude-mythos-5`, `claude-mythos-preview`, `claude-opus-5`, `claude-opus-4-8`, `claude-opus-4-7`, `claude-opus-4-6`, `claude-opus-4-5-20251101`, `claude-sonnet-5`, `claude-sonnet-4-6`
- Platforms: Claude API, Claude Platform on AWS, Amazon Bedrock, Google Cloud, Microsoft Foundry

effort（努力程度）参数让您可以控制 Claude 在响应请求时花费多少令牌。您可以使用单一模型在响应的全面性与令牌效率之间进行权衡。顶层 effort 参数在所有受支持的模型上均可使用，无需 beta 标头。[按消息设置的 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta) 目前处于 beta 阶段。

<Tip>
  要了解 effort 如何与思考交互以及应使用哪种控制方式，请参阅[思考与 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-effort)。在可使用自适应思考的情况下，effort 是控制思考深度的推荐方式。
</Tip>

## 设置 effort 级别

在请求中设置 `output_config.effort`。以下示例以 `medium` effort 运行一个请求并打印响应文本。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "messages": [{
        "role": "user",
        "content": "Analyze the trade-offs between microservices and monolithic architectures"
      }],
      "output_config": {
        "effort": "medium"
      }
    }'
  ```

  ```bash CLI
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 4096 \
    --output-config '{effort: medium}' \
    --message '{role: user, content: "Analyze the trade-offs between microservices and monolithic architectures"}' \
    --transform 'content.#(type=="text").text' \
    --raw-output
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      messages=[
          {
              "role": "user",
              "content": "Analyze the trade-offs between microservices and monolithic architectures",
          }
      ],
      output_config={"effort": "medium"},
  )

  for block in response.content:
      if block.type == "text":
          print(block.text)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    messages: [
      {
        role: "user",
        content: "Analyze the trade-offs between microservices and monolithic architectures"
      }
    ],
    output_config: {
      effort: "medium"
    }
  });

  const textBlock = response.content.find(
    (block): block is Anthropic.TextBlock => block.type === "text"
  );
  console.log(textBlock?.text);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 4096,
      Messages = [
          new() {
              Role = Role.User,
              Content = "Analyze the trade-offs between microservices and monolithic architectures"
          }
      ],
      OutputConfig = new OutputConfig
      {
          Effort = Effort.Medium
      }
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 4096,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Analyze the trade-offs between microservices and monolithic architectures")),
  	},
  	OutputConfig: anthropic.OutputConfigParam{
  		Effort: anthropic.OutputConfigEffortMedium,
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  for _, block := range response.Content {
  	if textBlock, ok := block.AsAny().(anthropic.TextBlock); ok {
  		fmt.Println(textBlock.Text)
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.messages.OutputConfig;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .addUserMessage("Analyze the trade-offs between microservices and monolithic architectures")
          .outputConfig(OutputConfig.builder()
              .effort(OutputConfig.Effort.MEDIUM)
              .build())
          .build();

      Message response = client.messages().create(params);
      response.content().stream()
          .flatMap(block -> block.text().stream())
          .forEach(textBlock -> IO.println(textBlock.text()));
  }
  ```

  ```php PHP
  $client = new Client();

  $message = $client->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Analyze the trade-offs between microservices and monolithic architectures']
      ],
      model: 'claude-opus-5',
      outputConfig: ['effort' => 'medium'],
  );

  foreach ($message->content as $block) {
      if ($block->type === 'text') {
          echo $block->text, PHP_EOL;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    messages: [
      { role: "user", content: "Analyze the trade-offs between microservices and monolithic architectures" }
    ],
    output_config: {
      effort: "medium"
    }
  )

  message.content.each do |block|
    puts block.text if block.type == :text
  end
  ```
</CodeGroup>

## effort 的工作原理

默认情况下，Claude 使用 high effort，花费所需的尽可能多的令牌以获得出色的结果。您可以将 effort 级别提高到 `max` 以获得绝对最高的能力，或者降低它以更保守地使用令牌，在接受一定能力下降的同时优化速度和成本。

<Tip>
  将 `effort` 设置为 `"high"` 所产生的行为与完全省略 `effort` 参数完全相同。
</Tip>

effort 参数会影响响应中的**所有令牌**，包括：

* 文本响应和解释
* 工具调用和函数参数
* 思考（在启用时）

由于 effort 适用于每一个输出令牌，因此无论是否启用思考，它都会起作用。较低的 effort 也意味着更少、更简洁的工具调用。

### effort 级别

| 级别       | 描述                                                                                                                                                                                                                 | 典型用例                              |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------- |
| `max`    | 绝对最大能力，对令牌花费没有限制。可在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Mythos Preview、Claude Opus 5、Claude Opus 4.8、Claude Opus 4.7、Claude Opus 4.6、Claude Sonnet 5 和 Claude Sonnet 4.6 上使用。 | 需要尽可能最深入的推理和最全面分析的任务              |
| `xhigh`  | 面向长周期工作的扩展能力。可在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Opus 5、Claude Opus 4.8、Claude Opus 4.7 和 Claude Sonnet 5 上使用。                                                             | 令牌预算达数百万的长时间运行的智能体和编码任务（超过 30 分钟） |
| `high`   | 高能力。等同于不设置该参数。                                                                                                                                                                                                     | 复杂推理、困难的编码问题、智能体任务                |
| `medium` | 平衡的方式，适度节省令牌。                                                                                                                                                                                                      | 需要在速度、成本和性能之间取得平衡的智能体任务           |
| `low`    | 最高效。显著节省令牌，但能力有所下降。                                                                                                                                                                                                | 需要最佳速度和最低成本的较简单任务，例如子智能体          |

并非每个支持 `max` 的模型都支持 `xhigh`。

<Note>
  effort 是一种行为信号，而不是严格的令牌预算。在较低的 effort 级别下，Claude 仍会对足够困难的问题进行思考，但对于同一问题，其思考量会少于较高 effort 级别下的思考量。
</Note>

下文中针对各模型的建议在与此表不同之处优先于此表。

### Claude Fable 5.1 的推荐 effort 级别

Claude Fable 5.1 支持全部五个 effort 级别。\*\*从默认值 `high` 开始。\*\*对于对能力最敏感的智能体和编码工作，可提升到 `xhigh` 或 `max`；对于常规或对延迟敏感的工作，一旦您的评估表明质量得以保持，可降低到 `medium` 或 `low`。在 `high` 及以上级别，请设置较大的 `max_tokens`。它是总输出（思考加响应文本）的硬性上限。同样的建议也适用于 Claude Mythos 5.1。请参阅[为 Claude Fable 5.1 编写提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#consider-all-effort-levels)。

Claude Fable 5.1 还支持通过按消息设置的 `output_config` [在对话中途更改 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)，这样可以保留提示缓存。

### Claude Fable 5 的推荐 effort 级别

在 Claude Fable 5 上，effort 是在智能、延迟和成本之间进行权衡的主要控制手段。**对于大多数任务，从默认值 `high` 开始**，对于对能力最敏感的工作负载使用 `xhigh`，对于常规工作则降低到 `medium` 或 `low`。Claude Fable 5 上较低的 effort 设置仍然表现良好，并且通常超过先前模型在 `xhigh` 下的表现。在 `high` 和 `xhigh` 级别，请设置较大的 `max_tokens`。它是总输出（思考加响应文本）的硬性上限。请参阅[成本控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#cost-control)。

如果任务能够完成但耗时超过必要，或者您希望获得更快、更具交互性的工作方式，请降低 effort。同样的建议也适用于 Claude Mythos 5。如需更完整的指导，请参阅[为 Claude Fable 5 编写提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5)。

### Claude Opus 5 的推荐 effort 级别

Claude Opus 5 支持全部五个 effort 级别。**从默认值 `high` 开始**，并根据您的评估进行调整：对于要求较高的编码和智能体工作，提升到 `xhigh`；当任务值得不受限制地花费令牌时，提升到 `max`；在您的评估表明质量得以保持的任何地方，大量使用 `low` 和 `medium` 作为控制令牌成本和响应时间的主要手段。如果您沿用了早期模型的 effort 设置，请在您的评估上重新进行一次 effort 扫描，而不是直接复用这些设置。

effort 控制的是思考量，而不是可见响应的长度：在 Claude Opus 5 上，更改 effort 并不能可靠地缩短响应，因此请改为[通过提示控制长度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#response-length-and-verbosity)。

API 默认值为 `high`。请显式设置 `effort` 以使用不同的级别。您传入的值会覆盖默认值。

在 Claude Opus 5 上，在 `xhigh` 或 `max` effort 下无法禁用思考：在这些级别设置 `thinking: {"type": "disabled"}` 的请求会返回 400 错误。请参阅[effort 与思考](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#effort-with-thinking)。

在以 `xhigh` 或 `max` effort 运行 Claude Opus 5 时，请设置较大的 `max_tokens`，以便模型有足够空间在子智能体和工具调用之间进行思考和行动。从 64k 令牌开始并在此基础上调整是一个合理的默认值。

Claude Opus 5 还支持通过按消息设置的 `output_config` [在对话中途更改 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)，这样可以保留提示缓存。

### Claude Opus 4.8 的推荐 effort 级别

针对 Claude Opus 4.7 的指导同样适用于 Claude Opus 4.8。**对于编码和智能体用例，从 `xhigh` 开始**，对于大多数其他对智能敏感的工作负载使用 `high`，并且只有在您已测量出较低级别在您的评估上能够保持质量时，才降低到 `medium` 或 `low`。

API 默认值为 `high`。请显式设置 `effort` 以使用不同的级别。您传入的值会覆盖默认值。

在以 `xhigh` 或 `max` effort 运行 Claude Opus 4.8 时，请设置较大的 `max_tokens`，以便模型有足够空间在子智能体和工具调用之间进行思考和行动。从 64k 令牌开始并在此基础上调整是一个合理的默认值。

### Claude Opus 4.7 的推荐 effort 级别

**对于编码和智能体用例，从 `xhigh` 开始**，并将 `high` 作为大多数对智能敏感的工作负载的最低级别。对于对成本敏感的工作负载，降低到 `medium`；只有当您的评估表明在 `xhigh` 下仍有可衡量的提升空间时，才提升到 `max`。

API 默认值为 `high`。要使用 `xhigh`，请显式设置 `effort`。您传入的值会覆盖默认值。

| Effort   | 针对 Claude Opus 4.7 的指导                                                     |
| -------- | -------------------------------------------------------------------------- |
| `low`    | 高效，但最适合简短、范围明确的任务。如果您的任务包含多个部分，请将 `low` 与明确的检查清单搭配使用。                      |
| `medium` | 适用于一般工作流程的直接替代选项，在降低成本的同时获得良好的结果。                                          |
| `high`   | 仍需要在智能与令牌消耗之间取得平衡的高级用例。这通常是质量与令牌效率之间的最佳平衡点。                                |
| `xhigh`  | 编码和智能体工作的推荐起点，也适用于探索性任务，例如重复的工具调用、详细的网络搜索和知识库搜索。预计令牌使用量会明显高于 `high`。       |
| `max`    | 留给前沿难题使用。在大多数工作负载上，`max` 会显著增加成本，而质量提升相对较小；在某些结构化输出或对智能不太敏感的任务上，它可能导致过度思考。 |

与 Claude Opus 4.6 相比，Claude Opus 4.7 也更严格地遵循 effort 级别，尤其是在 `low` 和 `medium` 级别。在较低的 effort 级别下，模型会将其工作范围限定在所要求的内容上，而不会做超出要求的事情。如果您在使用 Claude Opus 4.7 处理复杂问题时观察到推理较浅，请提高 effort，而不是通过提示来绕过它。如果您出于延迟考虑必须保持较低的 effort，请添加有针对性的指导，例如"This task involves multistep reasoning. Think carefully before responding."

在以 `xhigh` 或 `max` effort 运行 Claude Opus 4.7 时，请设置较大的 `max_tokens`，以便模型有足够空间在子智能体和工具调用之间进行思考和行动。从 64k 令牌开始并在此基础上调整是一个合理的默认值。

### Claude Sonnet 5 的推荐 effort 级别

Claude Sonnet 5 在 Claude API 和 Claude Code 上默认使用 `high` effort。

* \*\*High effort（默认）：\*\*适用于质量比速度或成本更重要的复杂推理、编码和智能体任务。
* \*\*Xhigh effort：\*\*适用于最困难的编码和智能体任务。请参阅[为 Claude Sonnet 5 编写提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-sonnet-5#calibrating-effort-and-thinking-depth)。
* \*\*Medium effort：\*\*相对于默认值的节省成本的降级选项。与 high effort 下的 Claude Sonnet 4.6 相当。
* \*\*Low effort：\*\*适用于高吞吐量或对延迟敏感的工作负载。适合优先考虑更快响应的聊天和非编码用例。
* \*\*Max effort：\*\*适用于需要绝对最高能力且对令牌花费没有限制的任务。

### Claude Sonnet 4.6 的推荐 effort 级别

Sonnet 4.6 默认使用 `high` effort。使用 Sonnet 4.6 时请显式设置 effort，以避免意外的延迟：

* **Medium effort**（推荐默认值）：对于大多数应用而言，是速度、成本和性能之间的最佳平衡。适用于智能体编码、大量使用工具的工作流程以及代码生成。
* \*\*Low effort：\*\*适用于高吞吐量或对延迟敏感的工作负载。适合优先考虑更快响应的聊天和非编码用例。
* \*\*High effort：\*\*适用于质量比速度或成本更重要的复杂推理和任务。
* \*\*Max effort：\*\*适用于需要绝对最高能力且对令牌花费没有限制的任务。

## effort 与工具使用

在使用工具时，effort 参数既会影响围绕工具调用的解释，也会影响工具调用本身。较低的 effort 级别倾向于：

* 将多个操作合并为更少的工具调用
* 进行更少的工具调用
* 不加铺垫，直接采取行动
* 完成后使用简洁的确认消息

较高的 effort 级别可能会：

* 进行更多的工具调用
* 在采取行动之前解释计划
* 提供详细的变更摘要
* 包含更全面的代码注释

## effort 与思考

`thinking` 参数控制 Claude 在回答之前是否在[思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)中进行思考；`effort` 参数控制 Claude 在整个响应中投入多少工作量，在自适应模式下，这包括它思考的频率和深度。不要将 `adaptive` 作为 `effort` 的值传递：`adaptive` 是一种思考模式，而不是努力程度级别。

在较高的 effort 级别下，Claude 会对大多数请求进行思考，且思考篇幅更长。在较低级别下，对于较简单的问题，它可以完全跳过思考。有关这两种控制方式如何协同工作的完整指导，请参阅[思考与 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-effort)。

在 Claude Opus 4.5（唯一支持 effort 的仅限扩展思考的模型）上，它与 [`budget_tokens`](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking) 配合使用：为您的任务设置 effort 级别，然后根据任务所需的推理深度设置思考令牌预算。

有关各模型的思考可用性，请参阅[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#supported-models)。无论是否启用思考，effort 都会起作用。请参阅[effort 的工作原理](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#how-effort-works)。

## 在对话中途更改 effort

您可以通过两种方式以不同的 effort 级别运行对话的后续轮次。在 Claude Fable 5.1、Claude Mythos 5.1 和 Claude Opus 5 上，使用按消息设置的 effort 更改，这样可以保留提示缓存。在其他模型上，在下一个请求中设置新的顶层值，这会使缓存重新开始。

### 按消息设置的 effort（beta）

按消息设置的 effort 处于 beta 阶段，需要 [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers) `mid-conversation-output-config-2026-07-01`。不支持按消息设置 effort 的模型（包括 Claude Fable 5）会返回 400 错误：`output_config.effort requires a model that supports per-turn effort; this model does not`。

添加一条 `role: "system"` 消息，其 `content` 为空，并在 `output_config.effort` 中指定新级别。新级别从下一个 `user` 轮次开始生效，并一直保持，直到后续消息更改它。该消息之前的所有内容均保持不变，因此缓存的前缀仍然匹配。

以下示例从 `high` 开始，然后针对常规的后续问题降低到 `low`：

<CodeGroup>
  ```bash cURL
  # 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: mid-conversation-output-config-2026-07-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-fable-5-1",
      "max_tokens": 4096,
      "output_config": {"effort": "high"},
      "messages": [
        {"role": "user", "content": "Plan a migration from SQLite to PostgreSQL in three short steps."},
        {"role": "assistant", "content": "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts."},
        {"role": "system", "content": [], "output_config": {"effort": "low"}},
        {"role": "user", "content": "Summarize the plan in one sentence."}
      ]
    }'
  ```

  ```bash CLI
  ant beta:messages create --beta mid-conversation-output-config-2026-07-01 \
    --transform 'content.#(type=="text").text' --raw-output <<'YAML'
  model: claude-fable-5-1
  max_tokens: 4096
  output_config:
    effort: high
  messages:
    - role: user
      content: Plan a migration from SQLite to PostgreSQL in three short steps.
    - role: assistant
      content: "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts."
    # 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
    - role: system
      content: []
      output_config:
        effort: low
    - role: user
      content: Summarize the plan in one sentence.
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.beta.messages.create(
      model="claude-fable-5-1",
      max_tokens=4096,
      output_config={"effort": "high"},
      messages=[
          {
              "role": "user",
              "content": "Plan a migration from SQLite to PostgreSQL in three short steps.",
          },
          {
              "role": "assistant",
              "content": "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts.",
          },
          # 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
          {"role": "system", "content": [], "output_config": {"effort": "low"}},
          {"role": "user", "content": "Summarize the plan in one sentence."},
      ],
      betas=["mid-conversation-output-config-2026-07-01"],
  )

  for block in response.content:
      if block.type == "text":
          print(block.text)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-fable-5-1",
    max_tokens: 4096,
    output_config: { effort: "high" },
    messages: [
      {
        role: "user",
        content: "Plan a migration from SQLite to PostgreSQL in three short steps."
      },
      {
        role: "assistant",
        content:
          "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts."
      },
      // 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
      { role: "system", content: [], output_config: { effort: "low" } },
      { role: "user", content: "Summarize the plan in one sentence." }
    ],
    betas: ["mid-conversation-output-config-2026-07-01"]
  });

  for (const block of response.content) {
    if (block.type === "text") {
      console.log(block.text);
    }
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta;
  using Anthropic.Models.Beta.Messages;

  AnthropicClient client = new();

  var response = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = "claude-fable-5-1",
      MaxTokens = 4096,
      OutputConfig = new() { Effort = Effort.High },
      Messages =
      [
          new() { Role = Role.User, Content = "Plan a migration from SQLite to PostgreSQL in three short steps." },
          new() { Role = Role.Assistant, Content = "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts." },
          // 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
          new()
          {
              Role = Role.System,
              Content = new([]),
              OutputConfig = new() { Effort = BetaSystemMessageOutputConfigEffort.Low },
          },
          new() { Role = Role.User, Content = "Summarize the plan in one sentence." },
      ],
      Betas = [AnthropicBeta.MidConversationOutputConfig2026_07_01],
  });

  foreach (var block in response.Content)
  {
      if (block.TryPickText(out var textBlock))
      {
          Console.WriteLine(textBlock.Text);
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.Background(), anthropic.BetaMessageNewParams{
  	Model:     "claude-fable-5-1",
  	MaxTokens: 4096,
  	OutputConfig: anthropic.BetaOutputConfigParam{
  		Effort: anthropic.BetaOutputConfigEffortHigh,
  	},
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Plan a migration from SQLite to PostgreSQL in three short steps.")),
  		{
  			Role:    anthropic.BetaMessageParamRoleAssistant,
  			Content: []anthropic.BetaContentBlockParamUnion{anthropic.NewBetaTextBlock("1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts.")},
  		},
  		// 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
  		anthropic.NewBetaSystemMessage(anthropic.BetaSystemMessageOutputConfigParam{
  			Effort: anthropic.BetaSystemMessageOutputConfigEffortLow,
  		}),
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Summarize the plan in one sentence.")),
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaMidConversationOutputConfig2026_07_01},
  })
  if err != nil {
  	log.Fatal(err)
  }

  for _, block := range response.Content {
  	if textBlock, ok := block.AsAny().(anthropic.BetaTextBlock); ok {
  		fmt.Println(textBlock.Text)
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaMessageParam;
  import com.anthropic.models.beta.messages.BetaOutputConfig;
  import com.anthropic.models.beta.messages.BetaSystemMessageOutputConfig;
  import com.anthropic.models.beta.messages.MessageCreateParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model("claude-fable-5-1")
          .maxTokens(4096L)
          .addBeta(AnthropicBeta.MID_CONVERSATION_OUTPUT_CONFIG_2026_07_01)
          .outputConfig(BetaOutputConfig.builder()
              .effort(BetaOutputConfig.Effort.HIGH)
              .build())
          .addUserMessage("Plan a migration from SQLite to PostgreSQL in three short steps.")
          .addAssistantMessage("1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts.")
          // 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
          .addMessage(BetaMessageParam.builder()
              .role(BetaMessageParam.Role.SYSTEM)
              .contentOfBetaContentBlockParams(List.of())
              .outputConfig(BetaSystemMessageOutputConfig.builder()
                  .effort(BetaSystemMessageOutputConfig.Effort.LOW)
                  .build())
              .build())
          .addUserMessage("Summarize the plan in one sentence.")
          .build();

      BetaMessage response = client.beta().messages().create(params);
      response.content().stream()
          .flatMap(block -> block.text().stream())
          .forEach(textBlock -> IO.println(textBlock.text()));
  }
  ```

  ```php PHP
  use Anthropic\Beta\AnthropicBeta;
  use Anthropic\Beta\Messages\BetaMessageParam;
  use Anthropic\Beta\Messages\BetaOutputConfig;
  use Anthropic\Beta\Messages\BetaSystemMessageOutputConfig;
  use Anthropic\Client;

  $client = new Client();

  $response = $client->beta->messages->create(
      model: 'claude-fable-5-1',
      maxTokens: 4096,
      outputConfig: BetaOutputConfig::with(effort: 'high'),
      messages: [
          BetaMessageParam::with(role: 'user', content: 'Plan a migration from SQLite to PostgreSQL in three short steps.'),
          BetaMessageParam::with(role: 'assistant', content: '1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts.'),
          // 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
          BetaMessageParam::with(
              role: 'system',
              content: [],
              outputConfig: BetaSystemMessageOutputConfig::with(effort: 'low'),
          ),
          BetaMessageParam::with(role: 'user', content: 'Summarize the plan in one sentence.'),
      ],
      betas: [AnthropicBeta::MID_CONVERSATION_OUTPUT_CONFIG_2026_07_01],
  );

  foreach ($response->content as $block) {
      if ($block->type === 'text') {
          echo $block->text, PHP_EOL;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-fable-5-1",
    max_tokens: 4096,
    output_config: {effort: :high},
    messages: [
      {role: "user", content: "Plan a migration from SQLite to PostgreSQL in three short steps."},
      {role: "assistant", content: "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts."},
      # 仅含 effort 的系统消息：新级别从下一个用户轮次开始生效。
      {role: "system", content: [], output_config: {effort: :low}},
      {role: "user", content: "Summarize the plan in one sentence."}
    ],
    betas: [Anthropic::AnthropicBeta::MID_CONVERSATION_OUTPUT_CONFIG_2026_07_01]
  )

  response.content.each do |block|
    puts block.text if block.type == :text
  end
  ```
</CodeGroup>

仅包含 effort 的系统消息不携带任何文本，因此[对话中途系统消息的放置规则](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)不适用。它可以出现在 `messages` 中的任何位置，包括作为第一个条目，或位于 `assistant` 轮次与下一个 `user` 轮次之间。取值为级别名称（`low`、`medium`、`high`、`xhigh` 和 `max`）。

在 Claude Fable 5.1 上，请优先使用这种形式，而不是在请求之间更改顶层值。顶层更改会使缓存重新开始，并且对模型的引导也不太可靠：它之前的回复是在先前的级别下编写的，而它倾向于与这些回复保持一致。

### 在下一个请求中设置顶层 effort

顶层 `output_config.effort` 适用于整个请求。要以不同的级别运行对话的后续部分，请在下一个请求中设置新值。由于顶层 effort 会影响渲染后的提示，因此在请求之间更改它不会保留早期轮次的缓存前缀。如果您在长会话中依赖[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)，并且您的模型不支持按消息设置 effort，请在开始时选择一个 effort 级别并保持不变。

## 最佳实践

1. \*\*显式设置 effort：\*\*API 默认为 `high`，但合适的起点取决于您的模型和工作负载。
2. \*\*对速度敏感或简单的任务使用 low：\*\*当延迟很重要或任务简单明了时，low effort 可以显著减少响应时间和成本。
3. \*\*测试您的用例：\*\*effort 级别的影响因任务类型而异。在部署之前，请针对您的具体用例评估性能。
4. \*\*考虑动态 effort：\*\*根据任务复杂度调整 effort。简单的查询可能适合 low effort，而智能体编码和复杂推理则受益于 high effort。在同一对话中改变它之前，请先参阅下一条。
5. \*\*在使用缓存的对话中保持顶层 effort 不变：\*\*在请求之间更改顶层 effort 值会使[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)失效，因此请在不同工作负载之间改变它，而不是在依赖缓存命中的对话内部改变它。在支持的模型上，请改用[按消息设置的 effort 更改](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)，这样可以保留缓存。请参阅[思考与提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-prompt-caching)。

## 后续步骤

<CardGroup>
  <Card title="任务预算" icon="gauge" href="https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets">
    为 Claude 提供一个针对完整智能体循环的建议性令牌预算，帮助模型在长时间的智能体任务中进行自我调节。
  </Card>

  <Card title="引导思考" icon="compass" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost">
    了解自适应思考（由 Claude 决定何时思考以及思考多少），并通过 effort 和提示对其进行引导。
  </Card>

  <Card title="思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    了解思考的工作原理、Claude 默认何时进行思考，以及思考如何与 effort 交互。
  </Card>
</CardGroup>
