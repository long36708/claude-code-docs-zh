---
title: Claude Opus 5 的新特性
url: https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5
description: Claude Opus 5 中新功能和行为变更的概述。
---

Claude Opus 5 相比 Claude Opus 4.8 是一次跨越式的改进，在深度推理、智能体与长周期任务以及测试时计算扩展方面提升最大。本页总结了 Claude Opus 5 的所有新特性，包括对话中途工具变更，以及针对在 Claude Opus 4.8 上运行的代码的两项破坏性变更：思考默认开启，以及仅在 effort 为 `high` 或更低时才能禁用思考。

## 新模型

| 模型            | API 模型 ID       | 描述               |
| ------------- | --------------- | ---------------- |
| Claude Opus 5 | `claude-opus-5` | 适用于复杂的智能体编码和企业工作 |

Claude Opus 5 拥有 [1M 令牌的 "context window"（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)（1M 令牌既是默认值也是最大值；没有更小的上下文变体）、128k 最大输出令牌，并且默认开启[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。Claude Opus 5 不支持 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models)。

有关完整的定价和规格，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。

## 新功能

### 对话中途工具变更（测试版）

您可以在对话的轮次之间添加或移除工具，同时保留提示缓存，而无需在整个会话生命周期内重复发送固定的工具列表。对话中途工具变更目前处于测试阶段：请在您的请求中包含 `mid-conversation-tool-changes-2026-07-01` 测试版请求头。有关用法，请参阅[对话中途工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes)。

### 默认回退模式

`fallbacks` 参数支持新的 `"default"` 模式，该模式按拒绝类别应用 Anthropic 推荐的回退模型，而不是由您自行维护的模型列表。整个 `fallbacks` 参数处于测试阶段。请使用 `server-side-fallback-2026-07-01` 测试版请求头，它同时支持 `"default"` 模式和显式模型列表（较早的 `server-side-fallback-2026-06-01` 请求头仅接受显式列表）。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。

### 更低的提示缓存最小长度

Claude Opus 5 上可缓存的最小提示长度为 512 个令牌，低于 Claude Opus 4.8 上的 1,024 个令牌。在 Claude Opus 4.8 上因过短而无法缓存的提示，现在无需任何代码更改即可创建缓存条目。有关各模型的最小值，请参阅 ["Prompt caching"（提示缓存）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#cache-limitations)。

### 快速模式

[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)（研究预览版）仅在 Claude API 上对 Claude Opus 5 可用；目前在 Amazon Bedrock、Claude Platform on AWS、Google Cloud 或 Microsoft Foundry 上不可用。Claude Opus 5 的快速模式定价为每百万输入令牌 10 美元，每百万输出令牌 50 美元。有关访问权限、支持的模型和定价，请参阅[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)。

## 行为变更

### 思考默认开启

在 Claude Opus 4.8 上，除非您设置 `thinking: {"type": "adaptive"}`，否则请求在不思考的情况下运行。在 Claude Opus 5 上，相同的请求默认开启[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)：模型在每一轮自行决定何时思考以及思考多少，而 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)是控制思考深度的手段。传输值保持不变；`thinking: {"type": "adaptive"}` 仍然有效，且等同于默认值。

对于在 Claude Opus 4.8 上不思考运行的代码而言，这是一项破坏性变更。响应可能在第一个 `text` 块之前以一个或多个 `thinking` 块开头，在默认的 `display: "omitted"` 下这些块以空的 `thinking` 字段返回，因此读取 `content[0].text` 或将第一个流式传输的内容块视为文本的代码必须改为按 `type` 字段选择内容块。工具使用循环必须将 `thinking` 块完整且未经修改地与其工具结果一起传回；请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。

思考令牌按输出令牌计费，并计入 `max_tokens`（总输出的硬性上限，包括思考和响应文本），因此对于在 Claude Opus 4.8 上不思考运行的工作负载，请重新审视 `max_tokens` 并重新确定成本基线。

API 保留了禁用思考的选项，但受禁用思考的 [effort 限制](https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5#disabling-thinking-requires-effort-high-or-below)约束。

### Effort 更加重要

Claude Opus 5 比任何早期 Opus 模型都更可靠地将额外的 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 转化为更好的结果，因此您选择的 effort 级别更具分量。完整的阶梯均可使用：`low`、`medium`、`high`、`xhigh` 和 `max`，其中 `max` 是用于最深度推理的最高层级。从默认值 `high` 开始，并根据您的评估向任一方向调整：在质量保持不变的情况下降低级别以节省令牌和延迟，或为最苛刻的工作提高级别。在以 `xhigh` 或 `max` effort 运行时，请设置较大的 `max_tokens`，以便模型有空间在子智能体和工具调用之间进行思考和行动。

此请求将 effort 一直调高到 `max`：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 64000,
      "stream": true,
      "output_config": {
        "effort": "max"
      },
      "messages": [
        {
          "role": "user",
          "content": "Explain why the sum of two even numbers is always even."
        }
      ]
    }'
  ```

  ```bash CLI
  # 64k 的 max_tokens 可能超出非流式传输的时间限制；请对事件进行流式传输。
  ant messages create --stream --format jsonl <<'YAML'
  model: claude-opus-5
  max_tokens: 64000
  output_config:
    effort: max
  messages:
    - role: user
      content: Explain why the sum of two even numbers is always even.
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  with client.messages.stream(
      model="claude-opus-5",
      max_tokens=64000,
      output_config={"effort": "max"},
      messages=[
          {
              "role": "user",
              "content": "Explain why the sum of two even numbers is always even.",
          }
      ],
  ) as stream:
      response = stream.get_final_message()

  print(response)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const stream = client.messages.stream({
    model: "claude-opus-5",
    max_tokens: 64000,
    output_config: {
      effort: "max"
    },
    messages: [
      {
        role: "user",
        content: "Explain why the sum of two even numbers is always even."
      }
    ]
  });

  const response = await stream.finalMessage();
  console.log(response);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 64000,
      OutputConfig = new OutputConfig
      {
          Effort = Effort.Max
      },
      Messages = [new() { Role = Role.User, Content = "Explain why the sum of two even numbers is always even." }]
  };

  var response = await client.Messages.CreateStreaming(parameters).Aggregate();
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  stream := client.Messages.NewStreaming(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 64000,
  	OutputConfig: anthropic.OutputConfigParam{
  		Effort: anthropic.OutputConfigEffortMax,
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Explain why the sum of two even numbers is always even.")),
  	},
  })

  response := anthropic.Message{}
  for stream.Next() {
  	event := stream.Current()
  	if err := response.Accumulate(event); err != nil {
  		log.Fatal(err)
  	}
  }
  if err := stream.Err(); err != nil {
  	log.Fatal(err)
  }

  fmt.Println(response)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(64000L)
      .outputConfig(OutputConfig.builder()
          .effort(OutputConfig.Effort.MAX)
          .build())
      .addUserMessage("Explain why the sum of two even numbers is always even.")
      .build();

  MessageAccumulator accumulator = MessageAccumulator.create();
  try (var streamResponse = client.messages().createStreaming(params)) {
      streamResponse.stream().forEach(accumulator::accumulate);
  }

  Message response = accumulator.message();
  IO.println(response);
  ```

  ```php PHP
  $client = new Client();

  $stream = $client->messages->createStream(
      maxTokens: 64000,
      messages: [
          ['role' => 'user', 'content' => 'Explain why the sum of two even numbers is always even.']
      ],
      model: Model::CLAUDE_OPUS_5,
      outputConfig: ['effort' => Effort::MAX],
  );

  $accumulator = MessageAccumulator::forMessages();
  foreach ($stream as $event) {
      $accumulator->accumulate($event);
  }

  echo $accumulator->message();
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.messages.stream(
    model: Anthropic::Model::CLAUDE_OPUS_5,
    max_tokens: 64000,
    output_config: {
      effort: :max
    },
    messages: [
      { role: "user", content: "Explain why the sum of two even numbers is always even." }
    ]
  ).accumulated_message

  puts response
  ```
</CodeGroup>

思考在 Claude Opus 5 上[默认开启](https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5#thinking-on-by-default)，因此不需要 `thinking` 字段。

### 禁用思考需要 effort 为 `high` 或更低

在 Claude Opus 5 上，仅当 effort 级别为 `high` 或更低时才接受 `thinking: {"type": "disabled"}`。在 effort 为 `xhigh` 或 `max` 时设置 `thinking: {"type": "disabled"}` 会返回 400 错误。此规则对发往 Claude Opus 5 及后续模型的每个请求强制执行。这是相对于 Claude Opus 4.8 的一项破坏性变更，在 Claude Opus 4.8 上禁用思考与 effort 级别无关。如果您的 Claude Opus 4.8 请求在 effort 为 `xhigh` 或 `max` 时禁用了思考，请要么保持禁用思考并将 effort 设置为 `high` 或更低，要么保持 effort 级别并移除 `thinking` 字段。

在禁用思考的情况下，Claude Opus 5 偶尔可能会将工具调用写入其文本输出而不是发出 `tool_use` 块，或在其可见响应中包含内部 XML 标签。在可能的情况下，请保持思考开启并通过较低的 effort 级别控制令牌成本；对于必须保持禁用思考的集成，请参阅[在禁用思考的情况下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)以了解提示方面的缓解措施。

### 模型行为差异

除了这些 API 变更之外，Claude Opus 5 的行为与 Claude Opus 4.8 有所不同，即使不更改任何代码您也可能会注意到。默认的面向用户的响应和书面交付物篇幅更长。在智能体会话中，模型更频繁地向用户叙述其进展。在多智能体框架中，它更乐于委派给子智能体。它还会在未被告知的情况下验证自己的工作，因此请移除从早期模型沿用下来的验证指令（"包含最终验证步骤"、"使用子智能体进行验证"）；这些指令会导致 Claude Opus 5 过度验证。有关调整上述每种行为的提示模式，请参阅[提示 Claude Opus 5](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5)。

## 能力提升

与 Claude Opus 4.8 相比，Claude Opus 5 是一次跨越式的改进而非渐进式改进，并且以 [Claude Fable 5](https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5) 一半的成本提供前沿智能。提升最大的方面包括：

* **深度推理**，在长问题链中持续进行多步分析。
* **智能体编码和长周期任务**，在长时间的工具使用循环中保持专注于任务，并完成多文件功能、更大规模的重构以及端到端的功能工作，而不留下存根或占位符。
* **测试时计算扩展**，将额外的 effort（最高至 `max` 级别）转化为更好的结果。
* **较低 effort 级别下的效率**，`low` 和 `medium` [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 以远低于更高设置的令牌和延迟产出强劲的质量。
* **代码审查和缺陷发现**，每次审查以高比率发现真实缺陷且误报很少，并在较低 effort 级别下保持准确。
* **视觉**，理解图表、文档和示意图，并复现 UI 和前端视觉效果，在获得可迭代分析、裁剪和验证其工作的工具时表现最强。
* **长上下文工作**，拥有 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)（既是默认值也是最大值），并在整个窗口范围内保持一致的指令遵循、工具调用和推理。
* **办公和文档任务**，生成和编辑包含非平凡公式的复杂多工作表电子表格，并制作结构良好的幻灯片。
* **多智能体协调**，运行子智能体团队，采用有效的编写者-验证者模式，且智能体相互覆盖工作的情况很少。

有关充分发挥这些能力的提示模式，请参阅[提示 Claude Opus 5](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#capability-improvements)。

## 定价

Claude Opus 5 的定价为每百万输入令牌 5 美元，每百万输出令牌 25 美元，与 Claude Opus 4.8 相同。由于思考默认开启且思考令牌按输出令牌计费，在 Claude Opus 4.8 上不思考运行的工作负载在相同的每令牌费率下每个请求可能产生更多输出令牌；请参阅[成本控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#cost-control)。

有关完整定价（包括批处理、提示缓存和快速模式费率），请参阅[定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

## 可用性

Claude Opus 5 可在以下平台使用：

* **Claude API：** 对所有客户可用，模型 ID 为 `claude-opus-5`。
* **AWS：** 通过 [Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 提供，模型 ID 为 `anthropic.claude-opus-5`，以及通过 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 提供。在 Amazon Bedrock 上，Claude Opus 5 也可通过 `bedrock-runtime` 上的 `InvokeModel` API 访问，由相同的基础设施提供服务；[Claude on Amazon Bedrock（旧版）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)集成未将其包含在其 ARN 版本化模型 ID 表中。
* **Google Cloud：** 通过 [Claude on Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 提供，模型 ID 为 `claude-opus-5`。
* **Microsoft Foundry：** 通过 [Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 提供。

Claude Opus 4.8 在所有这些平台上仍然可用。

## 迁移指南

要从 Claude Opus 4.8 迁移，请更新您的模型 ID：

<CodeGroup exclude="shell">
  ```python Python
  model = "claude-opus-4-8"  # Before
  model = "claude-opus-5"  # After
  ```

  ```typescript TypeScript
  let model = "claude-opus-4-8"; // Before
  model = "claude-opus-5"; // After
  ```

  ```csharp C#
  var model = Model.ClaudeOpus4_8; // Before
  model = Model.ClaudeOpus5; // After
  ```

  ```go Go
  model := anthropic.ModelClaudeOpus4_8 // Before
  model = anthropic.ModelClaudeOpus5    // After
  ```

  ```java Java
  Model model = Model.CLAUDE_OPUS_4_8; // Before
  model = Model.CLAUDE_OPUS_5; // After
  ```

  ```php PHP
  $model = Model::CLAUDE_OPUS_4_8; // Before
  $model = Model::CLAUDE_OPUS_5; // After
  ```

  ```ruby Ruby
  model = Anthropic::Model::CLAUDE_OPUS_4_8 # Before
  model = Anthropic::Model::CLAUDE_OPUS_5 # After
  ```
</CodeGroup>

然后查看[行为变更](https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5#behavior-changes)下的两项破坏性变更：思考默认开启（响应可能以 `thinking` 块开头，因此请按 `type` 选择内容块），以及在 effort 为 `xhigh` 或 `max` 时禁用思考会返回 400 错误。有关分步说明和完整检查清单，请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-4-8-to-claude-opus-5)。

## 后续步骤

<CardGroup cols={3}>
  <Card title="模型概览" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/overview">
    所有当前 Claude 模型的完整规格和定价。
  </Card>

  <Card title="提示 Claude Opus 5" icon="terminal" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5">
    Claude Opus 5 特有的行为差异和提示模式。
  </Card>

  <Card title="Effort" icon="gauge" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    控制 Claude 响应时使用的令牌数量，从 low 到 max。
  </Card>

  <Card title="思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    思考在默认开启时如何工作，以及何时可以禁用。
  </Card>

  <Card title="任务预算" icon="database" href="https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets">
    为 Claude 提供一个建议性的令牌预算，以便其据此安排工作节奏。
  </Card>

  <Card title="迁移指南" icon="code" href="https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide">
    从先前 Claude 版本迁移到最新 Claude 模型的指南。
  </Card>

  <Card title="快速模式" icon="bolt" href="https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode">
    以高级定价从 Claude Opus 模型获得更高的每秒输出令牌数。
  </Card>
</CardGroup>
