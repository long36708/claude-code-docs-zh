---
title: 任务预算
url: https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets
description: 为 Claude 提供覆盖整个智能体循环的建议性令牌预算，帮助模型在长时间运行的智能体任务中进行自我调节。
featureMetadata:
  status: beta
  betaHeader: task-budgets-2026-03-13
  supportedModels:
    - claude-fable-5-1
    - claude-mythos-5-1
    - claude-fable-5
    - claude-mythos-5
    - claude-opus-5-5
    - claude-opus-5
    - claude-opus-4-8
    - claude-opus-4-7
---

"Task budgets"（任务预算）让您可以告诉 Claude 它在一个完整的 "agentic loop"（智能体循环）中拥有多少令牌，包括思考、工具调用、工具结果和输出。模型会看到一个持续更新的倒计时，并利用它来确定工作的优先级，并在预算消耗时优雅地收尾。

## 何时使用任务预算

任务预算最适合这样的智能体工作流：Claude 在最终确定输出并等待下一次人类响应之前，会进行多次工具调用和决策。在以下情况下使用它们：

* 您希望 Claude 在长周期任务中自我调节令牌消耗。
* 您需要强制执行一个可预测的单任务成本或延迟上限。
* 您希望模型在接近预算时优雅地收尾（总结发现、报告进度），而不是在操作中途被截断。

任务预算与 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)相辅相成：effort 控制 Claude 对每一步推理的深入程度，而任务预算则限制 Claude 在整个智能体循环中可以完成的总工作量。

## 设置任务预算

将 `task_budget` 添加到 `output_config` 中，并包含 beta 请求头：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -N \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: task-budgets-2026-03-13" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5-5",
      "max_tokens": 128000,
      "stream": true,
      "messages": [{
        "role": "user",
        "content": "Review the codebase and propose a refactor plan."
      }],
      "output_config": {
        "effort": "high",
        "task_budget": {"type": "tokens", "total": 64000}
      }
    }'
  ```

  ```bash CLI
  ant beta:messages create --beta task-budgets-2026-03-13 \
    --stream --format jsonl <<'YAML' | jq 'select(.type == "message_delta").usage'
  model: claude-opus-5-5
  max_tokens: 128000
  messages:
    - role: user
      content: Review the codebase and propose a refactor plan.
  output_config:
    effort: high
    task_budget:
      type: tokens
      total: 64000
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  with client.beta.messages.stream(
      model="claude-opus-5-5",
      max_tokens=128000,
      output_config={
          "effort": "high",
          "task_budget": {"type": "tokens", "total": 64000},
      },
      messages=[
          {"role": "user", "content": "Review the codebase and propose a refactor plan."}
      ],
      betas=["task-budgets-2026-03-13"],
  ) as stream:
      response = stream.get_final_message()

  print(response.usage)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const stream = client.beta.messages.stream({
    model: "claude-opus-5-5",
    max_tokens: 128000,
    output_config: {
      effort: "high",
      task_budget: { type: "tokens", total: 64000 }
    },
    messages: [{ role: "user", content: "Review the codebase and propose a refactor plan." }],
    betas: ["task-budgets-2026-03-13"]
  });

  const response = await stream.finalMessage();
  console.log(response.usage);
  ```

  ```csharp C#

  var client = new AnthropicClient();

  var responseUpdates = client.Beta.Messages.CreateStreaming(new MessageCreateParams
  {
      Model = Messages::Model.ClaudeOpus5_5,
      MaxTokens = 128000,
      Messages = [new() { Role = Role.User, Content = "Review the codebase and propose a refactor plan." }],
      OutputConfig = new BetaOutputConfig
      {
          Effort = Effort.High,
          TaskBudget = new BetaTokenTaskBudget { Total = 64000 },
      },
      Betas = ["task-budgets-2026-03-13"],
  });

  var response = await responseUpdates.Aggregate();
  Console.WriteLine(response.Usage);
  ```

  ```go Go
  client := anthropic.NewClient()

  stream := client.Beta.Messages.NewStreaming(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5_5,
  	MaxTokens: 128000,
  	Betas:     []anthropic.AnthropicBeta{"task-budgets-2026-03-13"},
  	Messages: []anthropic.BetaMessageParam{{
  		Role: anthropic.BetaMessageParamRoleUser,
  		Content: []anthropic.BetaContentBlockParamUnion{{
  			OfText: &anthropic.BetaTextBlockParam{Text: "Review the codebase and propose a refactor plan."},
  		}},
  	}},
  	OutputConfig: anthropic.BetaOutputConfigParam{
  		Effort: anthropic.BetaOutputConfigEffortHigh,
  		TaskBudget: anthropic.BetaTokenTaskBudgetParam{
  			Total: 64000,
  		},
  	},
  })

  message := anthropic.BetaMessage{}
  for stream.Next() {
  	event := stream.Current()
  	if err := message.Accumulate(event); err != nil {
  		panic(err)
  	}
  }
  if stream.Err() != nil {
  	panic(stream.Err())
  }

  fmt.Printf("Usage: input_tokens=%d, output_tokens=%d\n", message.Usage.InputTokens, message.Usage.OutputTokens)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5_5)
      .maxTokens(128000L)
      .addUserMessage("Review the codebase and propose a refactor plan.")
      .outputConfig(BetaOutputConfig.builder()
          .effort(BetaOutputConfig.Effort.HIGH)
          .taskBudget(BetaTokenTaskBudget.builder().total(64000L).build())
          .build())
      .addBeta("task-budgets-2026-03-13")
      .build();

  BetaMessageAccumulator accumulator = BetaMessageAccumulator.create();
  try (StreamResponse<BetaRawMessageStreamEvent> stream =
          client.beta().messages().createStreaming(params)) {
      stream.stream().forEach(accumulator::accumulate);
  }

  BetaMessage response = accumulator.message();
  IO.println(response.usage());
  ```

  ```php PHP
  use Anthropic\Beta\Messages\BetaRawMessageDeltaEvent;

  $client = new Client();

  $stream = $client->beta->messages->createStream(
      model: 'claude-opus-5-5',
      maxTokens: 128000,
      messages: [
          ['role' => 'user', 'content' => 'Review the codebase and propose a refactor plan.'],
      ],
      outputConfig: [
          'effort' => 'high',
          'taskBudget' => ['type' => 'tokens', 'total' => 64000],
      ],
      betas: ['task-budgets-2026-03-13'],
  );

  // 最后一个 message_delta 事件包含该请求的累计令牌用量。
  $usage = null;
  foreach ($stream as $event) {
      if ($event instanceof BetaRawMessageDeltaEvent) {
          $usage = $event->usage;
      }
  }

  echo $usage;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  stream = client.beta.messages.stream(
    model: "claude-opus-5-5",
    max_tokens: 128_000,
    messages: [
      { role: "user", content: "Review the codebase and propose a refactor plan." }
    ],
    output_config: {
      effort: :high,
      task_budget: { type: :tokens, total: 64_000 }
    },
    betas: ["task-budgets-2026-03-13"]
  )

  response = stream.accumulated_message

  puts response.usage
  ```
</CodeGroup>

`task_budget` 对象有三个字段：

* `type`：始终为 `"tokens"`。
* `total`：Claude 在整个智能体循环中可以消耗的令牌数量，包括思考、工具调用、工具结果和输出。
* `remaining`（可选）：从先前请求中结转的剩余预算。省略时默认为 `total`。

## 预算倒计时的工作原理

Claude 会在整个对话过程中看到一个由服务端注入的预算倒计时标记。该标记显示当前智能体循环中还剩多少令牌，并随着模型生成思考、工具调用和输出以及处理工具结果而更新。Claude 利用这一信号来控制节奏，并在预算消耗时优雅地收尾。

<Note>
  **倒计时仅对模型可见。** API 响应不包含剩余预算字段：响应的 `usage` 对象中没有 `task_budget` 信息，SDK 也没有相应的访问器。要在客户端跟踪消耗，请按照[测量您当前的用量](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#measure-your-current-usage)中所示，对循环中各请求的令牌用量求和；或者在[跨压缩结转预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#carrying-a-budget-across-compaction-with-remaining)时，通过 `remaining` 传递您自己的数值。
</Note>

<Warning>
  **倒计时反映的是 Claude 在当前智能体循环中已处理的令牌，而不是您在请求之间重新发送的令牌。** 如果您的客户端在每个后续请求中都发送完整的对话历史，那么您在客户端统计的令牌数可能与 Claude 跟踪的预算不同。如果您在重新发送完整历史的同时还递减 `remaining`，模型看到的预算就会被低报，倒计时下降得比应有的更快，导致 Claude 在预算实际允许的范围之前就提前收尾。请设置一个充裕的预算，让模型根据倒计时自我调节，而不是尝试在客户端镜像倒计时。
</Warning>

### 什么算作一个轮次

预算覆盖一个智能体轮次，也称为一个智能体循环：即 Claude 针对一条不包含工具结果的用户消息所做的全部工作。一个轮次可以跨越多个请求。

一条不包含工具结果的用户消息会开启一个新轮次，并获得全新的预算。目前，只要较早轮次的历史仍保留在上下文中，倒计时仍会将其计入。一个常见的情况是在 Claude 结束其轮次之后（例如因为预算耗尽）发送的后续消息：

```json
{ "role": "user", "content": "Continue." }
```

包含 `tool_result` 块的用户消息会延续当前轮次，因为您的客户端正在处理属于该轮次的工具调用：

```json
{
  "role": "user",
  "content": [
    { "type": "tool_result", "tool_use_id": "toolu_01", "content": "<npm audit output>" }
  ]
}
```

即使该消息在工具结果之外还添加了新内容，情况也是如此：

```json
{
  "role": "user",
  "content": [
    { "type": "tool_result", "tool_use_id": "toolu_01", "content": "<npm audit output>" },
    { "type": "text", "text": "Also check the Dockerfile." }
  ]
}
```

轮次期间的服务器端[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold)不会重置预算：该轮次在压缩之前消耗的令牌仍会计入预算。轮次开始之前的令牌不计入，即使轮次开始时的压缩对其进行了总结也是如此。目前，这一排除规则仅适用于跨服务器端压缩延续的预算；较早轮次的历史只要仍保留在上下文中，就仍会被计入。

### 示例详解：跨请求的预算计数

任务预算统计的是 Claude **看到**的内容（思考、工具调用和结果以及文本），而不是您请求负载中的内容。在智能体循环中，您的客户端会在每个请求中重新发送完整对话，因此负载会不断增长，但预算只会按新增内容递减：即 Claude 生成的令牌以及它之前未见过的内容。以下示例是一个由三个请求组成的[智能体轮次](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#what-counts-as-a-turn)：第一个请求携带用户消息，后两个请求各自重新发送历史并附加一个工具结果。

考虑一个设置了 `task_budget: {type: "tokens", total: 100000}` 并带有单个 `bash` 工具的循环。

**请求 1。** 您发送初始请求：

```json
{
  "messages": [
    { "role": "user", "content": "Audit this repo for security issues and report findings." }
  ]
}
```

Claude 进行思考，然后发出一个工具调用，并以 `stop_reason: "tool_use"` 停止：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "thinking",
      "thinking": "I'll start by listing dependencies to look for known-vulnerable packages..."
    },
    {
      "type": "tool_use",
      "id": "toolu_01",
      "name": "bash",
      "input": { "command": "cat package.json && npm audit --json" }
    }
  ]
}
```

假设这条助手消息（思考加工具调用）共生成 5,000 个令牌。Claude 在生成过程中看到的倒计时最终约为 `remaining` ≈ 95,000。

**请求 2。** 您的客户端运行工具，然后重新发送完整历史并附加工具结果：

```json
{
  "messages": [
    { "role": "user", "content": "Audit this repo for security issues and report findings." },
    {
      "role": "assistant",
      "content": [
        { "type": "thinking", "thinking": "I'll start by listing dependencies..." },
        {
          "type": "tool_use",
          "id": "toolu_01",
          "name": "bash",
          "input": { "command": "cat package.json && npm audit --json" }
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01",
          "content": "<2,800 tokens of npm audit output>"
        }
      ]
    }
  ]
}
```

从请求 1 重新发送的消息不会被再次计数，但 2,800 个令牌的工具结果是新内容，会计入预算。Claude 又在思考和第二个工具调用（`grep -rn "eval(" src/`）上消耗了 4,000 个令牌。倒计时最终约为 `remaining` ≈ 88,200。

**请求 3。** 再次重新发送完整历史，并附加第二个工具结果（1,200 个令牌的 grep 输出）。Claude 撰写了一份 6,000 个令牌的最终发现报告，并以 `stop_reason: "end_turn"` 停止。`remaining` ≈ 81,000。

将三个请求并列比较，可以清楚地看出负载大小与预算消耗之间的区别：

| 请求     | 请求负载（您发送的大致输入令牌数）        | 本次请求计入预算的令牌数                             | 之后的预算 `remaining` |
| ------ | ------------------------ | ---------------------------------------- | ----------------- |
| 1      | \~20                     | 5,000（思考 + `tool_use`）                   | \~95,000          |
| 2      | \~7,800（请求 1 的消息 + 工具结果） | 6,800（2,800 工具结果 + 4,000 思考和 `tool_use`） | \~88,200          |
| 3      | \~13,000（完整历史 + 第二个工具结果） | 7,200（1,200 工具结果 + 6,000 `text`）         | \~81,000          |
| **总计** | **各请求共发送约 20,820**       | **计入预算 19,000**                          | 不适用               |

您的客户端发送了三次原始用户消息和两次第一条助手消息，但每条消息只被计数一次。预算消耗了 100,000 个令牌中的 19,000 个，尽管您的客户端传输的累计负载更大，而请求 2 和请求 3 中经过提示缓存的输入则更大。

### 使用 `remaining` 跨压缩结转预算

如果您自己的代码在请求之间压缩或重写消息历史（例如，通过总结较早的消息），服务器将无法记住压缩之前已消耗了多少预算。请在下一个请求中传递 `remaining`，使倒计时从您中断的地方继续，而不是重置为 `total`：

<CodeGroup exclude="shell">
  ```python Python
  # 压缩前消耗的令牌数，在客户端跟踪
  tokens_spent_so_far = 45000

  output_config = {
      "effort": "high",
      "task_budget": {
          "type": "tokens",
          "total": 128000,
          "remaining": 128000 - tokens_spent_so_far,
      },
  }
  ```

  ```typescript TypeScript
  // 压缩前消耗的令牌数，在客户端跟踪
  const tokensSpentSoFar = 45000;

  const outputConfig = {
    effort: "high",
    task_budget: {
      type: "tokens",
      total: 128000,
      remaining: 128000 - tokensSpentSoFar
    }
  };
  ```

  ```csharp C#
  // 压缩前消耗的令牌数，在客户端跟踪
  var tokensSpentSoFar = 45000;

  var outputConfig = new BetaOutputConfig
  {
      Effort = Effort.High,
      TaskBudget = new BetaTokenTaskBudget
      {
          Total = 128000,
          Remaining = 128000 - tokensSpentSoFar,
      },
  };
  ```

  ```go Go
  // 压缩前消耗的令牌数，在客户端跟踪
  tokensSpentSoFar := int64(45000)

  outputConfig := anthropic.BetaOutputConfigParam{
  	Effort: anthropic.BetaOutputConfigEffortHigh,
  	TaskBudget: anthropic.BetaTokenTaskBudgetParam{
  		Total:     128000,
  		Remaining: anthropic.Int(128000 - tokensSpentSoFar),
  	},
  }
  ```

  ```java Java
  // 压缩前消耗的令牌数，在客户端跟踪
  long tokensSpentSoFar = 45000;

  BetaOutputConfig outputConfig = BetaOutputConfig.builder()
      .effort(BetaOutputConfig.Effort.HIGH)
      .taskBudget(BetaTokenTaskBudget.builder()
          .total(128000L)
          .remaining(128000L - tokensSpentSoFar)
          .build())
      .build();
  ```

  ```php PHP
  // 压缩前消耗的令牌数，在客户端跟踪
  $tokensSpentSoFar = 45000;

  $outputConfig = [
      'effort' => 'high',
      'taskBudget' => [
          'type' => 'tokens',
          'total' => 128000,
          'remaining' => 128000 - $tokensSpentSoFar,
      ],
  ];
  ```

  ```ruby Ruby
  # 压缩前消耗的令牌数，在客户端跟踪
  tokens_spent_so_far = 45_000

  output_config = {
    effort: :high,
    task_budget: {
      type: :tokens,
      total: 128_000,
      remaining: 128_000 - tokens_spent_so_far
    }
  }
  ```
</CodeGroup>

在此示例中，压缩之前消耗的令牌是指您迄今为止从历史中移除的所有消息的用量，按照[测量当前用量](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#measure-your-current-usage)中的方法进行测量。请排除仍存在于您所发送消息中的任何内容，包括您添加的任何摘要，因为服务器会自行统计这些令牌。仅在您以这种方式替换历史时才更新此数值；不要在每个请求中递减它。请在每个请求中都传递计算得出的 `remaining`，而不仅仅是执行压缩的那个请求。

对于在每个请求中都重新发送完整未压缩历史的循环，请省略 `remaining`，让服务器跟踪倒计时。

## 在对话中途更改预算

`task_budget` 是一个请求级别的设置。要在任务进行中更改预算（例如，当用户扩大请求范围时延长预算），请在下一次请求的 `output_config` 中设置新的 `task_budget`。请注意对缓存的影响：预算值参与渲染后的提示，因此更改后的值不会匹配在旧值下创建的缓存条目（参见下文的[功能支持](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#feature-support)）。

## 任务预算是建议性的，而非强制执行的

任务预算是一个**软提示，而非硬上限**。如果 Claude 正处于某个操作中途，而中断该操作比完成它更具破坏性，Claude 可能偶尔会超出预算。对总输出令牌的强制限制仍然是 `max_tokens`，达到该限制时会以 `stop_reason: "max_tokens"` 截断响应。

若要对成本或延迟设置硬上限，请将任务预算与合理的 `max_tokens` 值结合使用：

* 使用 `task_budget` 为 Claude 提供一个控制节奏的目标。
* 使用 `max_tokens` 作为防止失控生成的绝对上限。

由于 `task_budget` 跨越整个智能体循环（可能包含多个请求），而 `max_tokens` 限制的是每个单独的请求，因此这两个值是相互独立的；不要求其中一个小于或等于另一个。

<Warning>
  **对任务而言过小的预算可能导致类似拒绝的行为。** 当 Claude 看到一个明显不足以完成所要求工作的预算时（例如，为一个耗时数小时的智能体编码任务设置 20,000 个令牌的预算），它可能会完全拒绝尝试该任务、大幅缩小任务范围，或者提前停止并给出部分结果，而不是开始一项它无法完成的工作。如果您在设置预算后观察到意外的拒绝或过早停止，请先提高预算，再调试其他参数。请根据您实际的任务长度分布来确定预算大小，而不是使用固定的默认值；参见[选择预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#choosing-a-budget)。
</Warning>

## 选择预算

合适的预算取决于您的智能体循环当前完成的工作量。与其猜测，不如先测量您现有的令牌用量，然后在此基础上进行调整。

### 测量您当前的用量

在**不**设置 `task_budget` 的情况下运行一组有代表性的任务样本，并记录 Claude 每个任务消耗的总令牌数。对于智能体循环，请对循环中每个请求的 `usage.output_tokens` 求和，再加上您在请求之间附加的工具结果的令牌数：

<CodeGroup exclude="shell:cURL">
  ```bash CLI
  ant messages create --transform 'usage.output_tokens' <<'YAML'
  model: claude-opus-5-5
  max_tokens: 4096
  messages:
    - role: user
      content: Review the codebase and propose a refactor plan.
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-opus-5-5",
      max_tokens=4096,
      messages=[
          {"role": "user", "content": "Review the codebase and propose a refactor plan."}
      ],
  )

  # 对循环中每个请求的 output_tokens（文本 + 思考 + 工具调用）求和。
  print(response.usage.output_tokens)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-opus-5-5",
    max_tokens: 4096,
    messages: [{ role: "user", content: "Review the codebase and propose a refactor plan." }]
  });

  // 对循环中每个请求的 output_tokens（文本 + 思考 + 工具调用）求和。
  console.log(response.usage.output_tokens);
  ```

  ```csharp C#

  var client = new AnthropicClient();

  var response = await client.Messages.Create(new MessageCreateParams
  {
      Model = Model.ClaudeOpus5_5,
      MaxTokens = 4096,
      Messages = [new() { Role = Role.User, Content = "Review the codebase and propose a refactor plan." }],
  });

  // 对循环中每个请求的 OutputTokens（文本 + 思考 + 工具调用）求和。
  Console.WriteLine(response.Usage.OutputTokens);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5_5,
  	MaxTokens: 4096,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Review the codebase and propose a refactor plan.")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  // 对循环中每个请求的 OutputTokens（文本 + 思考 + 工具调用）求和。
  fmt.Println(response.Usage.OutputTokens)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5_5)
      .maxTokens(4096L)
      .addUserMessage("Review the codebase and propose a refactor plan.")
      .build();

  Message response = client.messages().create(params);
  // 对循环中每个请求的 outputTokens（文本 + 思考 + 工具调用）求和。
  IO.println(response.usage().outputTokens());
  ```

  ```php PHP
  $client = new Client();

  $response = $client->messages->create(
      model: 'claude-opus-5-5',
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Review the codebase and propose a refactor plan.'],
      ],
  );

  // 对循环中每个请求的 outputTokens（文本 + 思考 + 工具调用）求和。
  echo $response->usage->outputTokens . "\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.messages.create(
    model: "claude-opus-5-5",
    max_tokens: 4096,
    messages: [
      { role: "user", content: "Review the codebase and propose a refactor plan." }
    ]
  )

  # 对循环中每个请求的 output_tokens（文本 + 思考 + 工具调用）求和。
  puts response.usage.output_tokens
  ```
</CodeGroup>

在一组有代表性的任务上运行此代码并记录分布情况。从您单任务令牌消耗的 p99 开始，以了解为模型提供任务预算可能会如何改变模型的行为，然后根据需要向上或向下测试。

在所有支持任务预算的模型上（请参阅[功能支持](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#feature-support)），`task_budget.total` 可接受的最小值为 **20,000 个令牌**。更小的值会返回 400 错误。

## 与其他参数的交互

* **`max_tokens`：** 与任务预算相互独立。`max_tokens` 是对每个请求生成令牌的硬性上限，而 `task_budget` 是覆盖整个智能体循环（可能跨越多个请求）的建议性上限。在 `xhigh` 或 `max` effort 下，请将 `max_tokens` 设置为至少 64k，以便为 Claude 在每个请求中的思考和行动留出空间。
* **[Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)：** Effort 控制 Claude 在每个步骤中推理的深度。任务预算控制 Claude 在整个智能体循环中完成的总工作量。两者相辅相成：effort 调节深度，任务预算调节广度。
* **[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)：** 任务预算将思考令牌计入统计，因此自适应思考会随着预算的消耗而减少。
* **[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)：** 预算倒计时标记在每个请求中由服务器端注入，因此在不同请求之间不会匹配。如果您的客户端在每个后续请求中递减 `task_budget.remaining`，更改后的值会使包含它的任何缓存前缀失效。为保留缓存，请在初始请求中设置一次预算，并让模型根据服务器端倒计时自我调节，而不是在客户端修改预算。

## 功能支持

| 模型                | 支持情况                                   |
| ----------------- | -------------------------------------- |
| Claude Fable 5.1  | Beta（设置 `task-budgets-2026-03-13` 请求头） |
| Claude Mythos 5.1 | Beta（设置 `task-budgets-2026-03-13` 请求头） |
| Claude Opus 5.5   | Beta（设置 `task-budgets-2026-03-13` 请求头） |
| Claude Opus 5     | Beta（设置 `task-budgets-2026-03-13` 请求头） |
| Claude Fable 5    | Beta（设置 `task-budgets-2026-03-13` 请求头） |
| Claude Mythos 5   | Beta（设置 `task-budgets-2026-03-13` 请求头） |
| Claude Sonnet 5   | 不支持                                    |
| Claude Opus 4.8   | Beta（设置 `task-budgets-2026-03-13` 请求头） |
| Claude Opus 4.7   | Beta（设置 `task-budgets-2026-03-13` 请求头） |
| Claude Opus 4.6   | 不支持                                    |
| Claude Sonnet 4.6 | 不支持                                    |
| Claude Haiku 4.5  | 不支持                                    |

任务预算在 [Claude Code](https://code.claude.com/docs/zh-CN/overview) 或 Cowork 界面上不受支持。请在[受支持的模型](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#feature-support)上直接通过 Messages API 使用任务预算。

## 后续步骤

<CardGroup>
  <Card title="Effort" icon="gauge" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    控制 Claude 对智能体循环中每一步推理的深入程度。
  </Card>

  <Card title="自适应思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    让 Claude 决定何时以及在多大程度上使用扩展思考。
  </Card>

  <Card title="压缩" icon="arrows-clockwise" href="https://platform.claude.com/docs/zh-CN/build-with-claude/compaction">
    通过服务端压缩管理长时间运行对话中的上下文。
  </Card>

  <Card title="提示缓存" icon="database" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching">
    通过缓存提示前缀来降低重复提示的成本和延迟。
  </Card>
</CardGroup>
