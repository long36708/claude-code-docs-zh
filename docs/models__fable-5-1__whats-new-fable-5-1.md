---
title: Claude Fable 5.1 的新特性
url: https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1
description: Claude Fable 5.1 和 Claude Mythos 5.1 的新功能、破坏性变更和能力改进概览。
---

Claude Fable 5.1 在 Claude Fable 5 的基础上进行了扩展，输入和输出价格保持不变，缓存读取成本降至四分之一，并带来了更强的长时间运行的智能体编码、多步骤研究，以及文档、电子表格和幻灯片处理能力。对于大多数工作负载，请从 Claude Opus 5 开始（请参阅[选择模型](https://platform.claude.com/docs/zh-CN/about-claude/models/choosing-a-model)）。当您需要高要求的推理和长周期智能体工作，或者在 Claude Opus 5 上以更高 effort 运行的评估仍然达不到要求时，请使用 Claude Fable 5.1。Claude Mythos 5.1 仅向 [Project Glasswing](https://anthropic.com/glasswing) 参与者提供相同的能力。

如果您已经在调用 Claude Fable 5，有三项变更是破坏性的：[强制工具使用会返回错误](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#forced-tool-use-is-not-supported)、[早期模型无法读取其思考块](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#thinking-blocks-are-tied-to-the-model-that-produced-them)，以及[编辑早期轮次会使思考块失效](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#editing-earlier-turns-invalidates-thinking-blocks)。有五项是新增的：[按消息设置 effort](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#change-effort-mid-conversation-beta)（测试版）、[轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#turn-scoped-system-messages-beta)（测试版）、[工具调用之间可读的进度更新](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#progress-updates-between-tool-calls-beta)（`display: "updates"`，测试版）、[更低的缓存读取价格](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#pricing)，以及[内容溯源](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#content-provenance)。

## 模型

| 模型                | Claude API ID     | 描述                                            | 可用性                                                         |
| ----------------- | ----------------- | --------------------------------------------- | ----------------------------------------------------------- |
| Claude Fable 5.1  | claude-fable-5-1  | Claude Fable 5 的继任者，适用于长时间运行的智能体编码、知识工作和研究    | 所有客户，可在 Claude API 和合作伙伴平台上使用                               |
| Claude Mythos 5.1 | claude-mythos-5-1 | 与 Claude Fable 5.1 能力相同。Claude Mythos 5 的继任者。 | 仅限 [Project Glasswing](https://anthropic.com/glasswing) 参与者 |

Claude Fable 5.1 和 Claude Mythos 5.1 共享相同的规格和定价：

* **上下文窗口和输出：** [1M 令牌的 "context window"（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)（默认值和最大值），整个窗口范围内均按标准的每令牌价格计费，最大输出令牌数为 128k。
* **思考：** "adaptive thinking"（自适应思考）始终开启，详见[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)控制思考深度。
* **定价：** 与 Claude Fable 5 相同，但[缓存读取价格更低](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#pricing)。
* **分词器：** 与 Claude Fable 5 相同（随 Claude Opus 4.7 引入）。与 Claude Opus 4.7 之前的模型相比，相同文本产生的令牌数大约多 30%。请参阅[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)。

有关所有当前模型，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。

## 破坏性变更

### 不支持强制工具使用

Claude Fable 5.1 和 Claude Mythos 5.1 不支持强制 "tool use"（工具使用）。将 `tool_choice` 设置为 `{"type": "any"}` 或 `{"type": "tool", "name": "..."}` 会返回 400 `invalid_request_error`：

```text wrap
tool_choice: type "tool" and "any" are not supported for this model.
```

`tool_choice: {"type": "auto"}`（默认值）和 `{"type": "none"}` 保持不变。相同的验证也适用于[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点。

这些模型的思考始终开启，而强制工具调用会跳过思考。模型会转而将其推演过程写入工具参数中，从而降低参数质量。如需符合 schema 的 JSON，请保持 `tool_choice: {"type": "auto"}` 并通过[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)设置 `strict: true`，或将 schema 移至[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)。要让模型调用工具而不是以文本回复，请在提示中说明该工具何时适用（例如，"使用 `get_weather` 工具来回答"）。Claude Fable 5.1 能够可靠地遵循明确的工具指令。

### 早期模型无法读取 Claude Fable 5.1 的思考块

每个思考块都会记录是哪个模型生成了它，并且仅在一个方向上被保留：Claude Fable 5.1 可以读取早期模型的思考块，而没有任何早期模型可以读取 Claude Fable 5.1 的思考块。迁移到 Claude Fable 5.1 的对话（从 Claude Opus 5、Claude Fable 5 或任何更早的 Claude 模型）会保留其推理。从 Claude Fable 5.1 迁移到上述任何模型的对话，在那些模型上运行的轮次中会丢失推理。

当请求携带目标模型无法读取的块时（例如，在对话中途切换模型的路由器或回退机制），API 会在模型看到该块之前将其丢弃。被丢弃的块不计入 `input_tokens`，也不会计费。使用 `thinking-binding-controls-2026-08-01` 测试版请求头时，丢弃操作会在顶层 `input_transformations` 数组中报告。不使用该请求头时，丢弃是静默的。请参阅[保留的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-for-model)。

### 编辑早期轮次会使思考块失效

修改 Claude Fable 5.1 思考块之前的任何内容（`system` 提示、`tools` 或更早的消息）会导致下一次请求出错，或者如果您选择启用该行为，则该块会被丢弃。Claude Mythos 5.1 不执行此检查。Claude Code、claude.ai、[Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 和 [Claude Agent SDK](https://code.claude.com/docs/zh-CN/agent-sdk/overview) 会为您保持该前缀完整。如果您的代码自行构建 `messages` 数组，请在迁移前进行检查：[保留的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)详细介绍了该检查及每种修复方法。该检查对 2026 年 8 月 31 日或之后创建的新账户强制执行。对于更早创建的账户，API 会记录不匹配情况，但仅在请求设置了 `thinking.block_binding.prefix_mismatch_behavior` 时才会据此采取行动。

以下模式会使之后的每个思考块失效：

* 编辑、重新排序或删除早期轮次，同时保留后续轮次。
* 向早期轮次注入按请求变化的文本（提醒或状态行），并在下一次请求时将其删除。
* 在同一对话的请求之间重建顶层 `system` 提示或 `tools` 数组。
* 在后续请求中提供不同字节内容的图像或文档 URL（该检查针对的是字节而非 URL，因此同一文件的轮换签名 URL 没有问题）。

以下操作可保持后续块有效：删除开头连续的一段思考块（从最旧的开始）、让服务器端压缩或上下文编辑裁剪历史记录、移动 `cache_control` 标记，以及在请求之间更改 `effort`。从连续段开头以外的任何位置删除思考块，会使其后的每个思考块失效。

在强制执行该检查的情况下，重放已失效块的请求会被拒绝并返回 400，其消息为 `The block is bound to a different conversation`。若要改为丢弃该块并继续，请发送 `thinking-binding-controls-2026-08-01` 测试版请求头并设置 `thinking.block_binding.prefix_mismatch_behavior: "drop_block"`。丢弃操作会在 `input_transformations` 中以 `reason: "prefix_binding_mismatch"` 报告。

要在长会话中保持思考有效，请将对话视为仅追加（append-only）。使用[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)添加指令（如果只应适用于一个轮次，则使用[轮次作用域](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#turn-scoped-system-messages-beta)的消息），并使用[对话中途工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes)来更改工具，而不是编辑 `system` 或 `tools`。使用服务器端[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)或[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)来裁剪上下文，这些不算作编辑。这些模式还能使 "prompt cache"（提示缓存）保持热状态，详见[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)。要了解您的集成是否编辑了历史记录，请使用 `prefix_mismatch_behavior: "drop_block"` 运行一次会话并记录 `input_transformations`：[迁移指南](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-preserved-thinking)提供了三步检查法。完整规则请参阅[保留的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)。

## 新功能

### 在对话中途更改 effort（测试版）

在 Claude Fable 5.1 上，您可以在对话中途更改 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别，而不会使提示缓存失效。在困难步骤时提高它，在常规步骤时降低它。按消息设置 effort 处于测试阶段：请包含 `mid-conversation-output-config-2026-07-01` 测试版请求头。Claude Fable 5.1、Claude Mythos 5.1 和 Claude Opus 5 在 Claude API 上支持此功能。

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

详情请参阅[按消息设置 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)。

### 轮次作用域的系统消息（测试版）

[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)可以限定作用于单个轮次。在 `role: "system"` 消息上设置 `clear_at: "next_user_message"`，其文本在当前轮次中具有 "system prompt"（系统提示）的权威性，一旦存在后续的 `user` 消息便停止渲染。该消息保留在 `messages` 中，您继续原样将其发回，因此对话中更早的内容不会发生任何变化。[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)继续匹配，后续的[思考块保持有效](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)，并且已清除的消息不消耗输入令牌。将其用于工具循环中的按轮次提醒（"在运行更多代码之前检查您的收件箱"、"用户看不到该工具输出"），而不是将文本注入历史记录并在下一次请求时删除。轮次作用域的系统消息处于测试阶段：请包含 `mid-conversation-system-clear-at-2026-08-21` 测试版请求头。请参阅[轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)。

```json
{
  "role": "system",
  "clear_at": "next_user_message",
  "content": "Results have landed in your inbox. Check it before running more code."
}
```

### 工具调用之间的进度更新（测试版）

与 Claude Fable 5 一样，Claude Fable 5.1 会在工具调用之间写下简短的进度更新，说明它发现了什么以及接下来要做什么，不过数量更少（请参阅[相对于 Claude Fable 5 的变化](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#changed-from-claude-fable-5)）。每条更新都作为独立的 `thinking` 块紧接在工具调用之前到达。在默认的 `thinking.display` 为 `"omitted"` 的情况下，这些块与推理一样返回为空，因此一个较长的智能体轮次在您的用户看来可能是静默的。新增的是 `display: "updates"` 选项：配合 `thinking-display-updates-2026-08-18` 测试版请求头进行设置，即可以文本形式接收进度更新，同时推理保持隐藏。此时任何带有非空文本的 `thinking` 块都是可以向用户展示的状态行。`"summarized"` 也会返回它们，但与摘要化的推理混合在一起。请参阅[工具调用之间的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)。

### 内容溯源

Claude Fable 5.1 和 Claude Mythos 5.1 生成的文本在该模型可用的每个平台上都带有 Anthropic 的统计文本水印。Claude 生成的受支持的图像和视频文件（例如通过[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)生成的），当您通过 Claude API 上的 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 检索时，会带有已签名的 [C2PA](https://c2pa.org/) 内容凭证（Content Credentials）。

水印不会改变输出的含义、质量或可读性。它不添加任何令牌或隐藏字符，不携带有关您或您组织的任何信息，也不需要对您的请求或响应进行任何更改。有关背景信息，请参阅 [Claude 如何标记 AI 生成的内容](https://support.claude.com/en/articles/16266773-how-claude-marks-ai-generated-content)和 [Claude 的文本水印如何工作](https://www.anthropic.com/news/claude-text-watermark)。

## 行为差异

### 相对于 Claude Fable 5 的变化

Claude Fable 5.1 与 Claude Fable 5 在若干方面存在差异，这些差异无需任何代码更改就会显现。每一项在 [Claude Fable 5.1 提示指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1)中都有对应的提示修复方法：

* **并行工具调用更加多变。** 在 Claude Fable 5 会批量发出多个工具调用的地方，Claude Fable 5.1 可能每轮只发出一个工具调用。这会出现在下一批独立读取仅被隐含暗示的长智能体循环中：自定义编码智能体、bash 加编辑器的运行框架、计算机使用。额外的轮次会消耗令牌、往返次数和实际耗时，但不会降低答案质量。明确指出要获取多项内容的请求仍会并行运行。请添加[在智能体循环中批量处理独立工具调用](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#batch-independent-tool-calls-in-agent-loops)中的单行批处理指令。
* **长时间工具运行期间的进度更新更少。** 模型在工具调用之间写的面向用户的文本更少，尤其是在较高 effort 下。将 `thinking.display` 设置为 `"updates"`（测试版）以接收它确实写下的[进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)，并删除任何告诉它将发现保留到最终响应的提示行。如果您的 UI 依赖于叙述，请明确要求提供开场语、定期更新和结尾回顾。请参阅[要求提供面向用户的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#ask-for-user-facing-progress-updates)。
* **在 `low` effort 下更常凭记忆回答。** 在最低 effort 级别下，模型调用搜索或检索工具的频率更低。对于需要最新信息的轮次，请提高 effort（包括[在对话中途](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#change-effort-mid-conversation-beta)），或添加[低 effort 下的搜索触发](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#search-triggering-at-low-effort)中的验证提示。
* **部分场景下行文更密集。** 在某些情况下，其行文比 Claude Fable 5 更密集，句子更长，段落分隔更少。请参阅[写作密度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#writing-density)。
* **聊天中的格式更少。** 该模型使用粗体、标题和列表的频率低于早期 Claude 模型，因此为那些模型编写的反格式化规则可能会抑制内容所需的结构。请参阅[聊天中的格式](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#formatting-in-chat)。
* **摘要中未标注的引用。** 在总结文档时，模型更有可能复述来源中的段落而不将其标注为引用。请参阅[引用检索到的来源](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#quoting-retrieved-sources)。
* **为小改动重写整个文件。** 在编辑文本文件时，模型更有可能重写整个文件而不是进行有针对性的编辑。结果通常相同，但重写会消耗更多输出令牌和时间。请参阅[优先使用有针对性的编辑而非整文件重写](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#prefer-targeted-edits-over-whole-file-rewrites)。

### 相对于 Claude Fable 5 未变的部分

以下 Messages API 行为从 Claude Fable 5 原样延续：

* 自适应思考始终开启。带 `budget_tokens` 的 `thinking: {"type": "enabled"}` 和 `thinking: {"type": "disabled"}` 都会返回 400 错误。请省略 `thinking` 或发送 `{"type": "adaptive"}`。
* `thinking.display` 默认为 `"omitted"`。`"summarized"` 可用，原始思维链永远不会返回。
* 工具调用之间的推理出现在思考块中而非文本中，并且[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)是自动的，无需测试版请求头。
* 预填充助手响应会返回 400 错误。
* 非默认的 `temperature`、`top_p` 或 `top_k` 值会返回 400 错误。
* 最小可缓存提示长度为 512 个令牌。
* 支持[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)和工具变更。

## 能力改进

Claude Fable 5.1 在 Claude Fable 5 的基础上有所改进，且在较高 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别下差距最大。提升集中在六个领域：

* **长会话中的智能体编码**，包括多文件功能、大型重构和迁移、调试，以及跨越持续数小时会话的代码审查。
* **涉及文档、电子表格和幻灯片的知识工作**，将一项分析从最初的问题推进到成品文档、带实时公式的电子表格，或从空白页构建的幻灯片。
* **研究和搜索**，在多步骤网络研究和会对发现进行跟进的深度研究任务上准确率更高。
* **视觉**，读取密集的图表、申报文件和嵌套在 PDF 中的表格，包括在图表上使用裁剪和缩放工具。
* **长上下文工作**，在完整的 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)范围内进行推理并关联细节。
* **计算机使用**，更可靠地操作浏览器和桌面应用程序，并从失败的步骤中恢复。

多语言性能与 Claude Fable 5 相当。

## 拒绝、回退和计费

Claude Fable 5.1 包含安全分类器，覆盖与 Claude Fable 5 相同的 `stop_details` 类别，并且[拒绝和回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)中的所有内容均适用。它可能返回 `stop_reason: "refusal"`，因此请处理拒绝并配置回退。

* **拒绝：** 被拒绝的请求返回 HTTP 200，带有 `stop_reason: "refusal"` 和一个指明所触发策略领域的 [`stop_details`](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response) 对象。
* **回退：** 使用[服务器端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)、[SDK 中间件](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#client-side-fallback)或您自己的重试机制，在另一个模型上重试被拒绝的请求。`fallbacks: "default"`（测试版）会在 Anthropic 为该类别推荐的模型上重试被拒绝的请求。Claude Fable 5.1 允许的回退目标是 Claude Opus 4.8 和 Claude Opus 5。
* **计费：** 对于在任何输出之前到达的拒绝，您不会被计费；并且对于 Claude Fable 5.1，[回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)会退还切换模型的提示缓存成本。

## 定价

Claude Fable 5.1 和 Claude Mythos 5.1 的定价与 Claude Fable 5 相同，缓存读取除外（价格以美元计）：

| 基础输入       | 5 分钟缓存写入      | 1 小时缓存写入   | 缓存读取         | 输出         |
| ---------- | ------------- | ---------- | ------------ | ---------- |
| $10 / MTok | $12.50 / MTok | $20 / MTok | $0.25 / MTok | $50 / MTok |

在这些模型上，缓存读取（命中和刷新）的成本是基础输入价格的 0.025 倍，而其他 Claude 模型为 0.1 倍。重复读取已缓存前缀的长智能体会话只需支付 Claude Fable 5 费率的四分之一。缓存写入和 [512 令牌的最小可缓存提示长度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#cache-limitations)保持不变。

[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)为每百万输入令牌 5 美元，每百万输出令牌 25 美元。有关数据驻留和工具定价，请参阅[定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

## 可用性

Claude Fable 5.1 可在以下平台使用：

* **Claude API：** 所有客户，模型 ID 为 `claude-fable-5-1`。
* **AWS：** [Amazon Bedrock 中的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)，模型 ID 为 `anthropic.claude-fable-5-1`；以及 [AWS 上的 Claude Platform](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)，模型 ID 为 `claude-fable-5-1`。
* **Google Cloud：** [Google Cloud 上的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)，模型 ID 为 `claude-fable-5-1`。
* **Microsoft Foundry：** [Microsoft Foundry 中的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)，运行在 Anthropic 基础设施上。

Claude Mythos 5.1 仅向 [Project Glasswing](https://anthropic.com/glasswing) 中获批的客户提供。如需访问权限，请联系您的 Anthropic、AWS 或 Google Cloud 客户团队。

Claude Fable 5.1 和 Claude Mythos 5.1 采用 30 天数据保留期，除非获得 Anthropic 明确授权，否则不提供零数据保留。两者均为[受保护模型（Covered Models）](https://support.claude.com/en/articles/15425695)，与 Claude Fable 5 和 Claude Mythos 5 一样。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。

## 从 Claude Fable 5 迁移

要从 Claude Fable 5 迁移，请更新您的模型 ID：

<CodeGroup exclude="shell">
  ```python Python
  model = "claude-fable-5"  # Before
  model = "claude-fable-5-1"  # After
  ```

  ```typescript TypeScript
  let model = "claude-fable-5"; // Before
  model = "claude-fable-5-1"; // After
  ```

  ```csharp C#
  var model = "claude-fable-5"; // Before
  model = "claude-fable-5-1"; // After
  ```

  ```go Go
  model := "claude-fable-5"  // Before
  model = "claude-fable-5-1" // After
  ```

  ```java Java
  String model = "claude-fable-5"; // Before
  model = "claude-fable-5-1"; // After
  ```

  ```php PHP
  $model = 'claude-fable-5'; // Before
  $model = 'claude-fable-5-1'; // After
  ```

  ```ruby Ruby
  model = "claude-fable-5" # Before
  model = "claude-fable-5-1" # After
  ```
</CodeGroup>

然后检查以下事项：

1. 删除任何类型为 `any` 或 `tool` 的 `tool_choice`。将 schema 强制执行移至配合 `tool_choice: {"type": "auto"}` 的[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)，或移至[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)。
2. 原样传回思考块，并保持历史记录仅追加。如果您的代码自行构建 `messages` 数组，请运行[历史编辑检查](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#fable-5-1-preserved-thinking)：将您当前注入并删除的按轮次提醒移至[轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#turn-scoped-system-messages-beta)，将 `system` 和 `tools` 的更改移至对话中途系统消息，在服务器端裁剪上下文或从您跨客户端摘要携带的轮次中剥离思考块，然后选择一个生产环境的 [`prefix_mismatch_behavior`](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking-controls) 并监控 `input_transformations`。
3. 从默认值（`high`）重新调整 effort，并考虑[在对话中途更改它](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#change-effort-mid-conversation-beta)，而不是在整个会话中保持一个级别。
4. 在智能体循环中，留意在 Claude Fable 5 会批量发出多个工具调用的地方是否变为每轮一个工具调用，并添加 [Claude Fable 5.1 提示指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1)中的按轮次说明。
5. 重新运行您的评估。拒绝处理、回退、回退抵扣和令牌计数原样延续。缓存读取成本更低（请参阅[定价](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#pricing)），默认行为在[相对于 Claude Fable 5 的变化](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#changed-from-claude-fable-5)下列出的方面有所不同。

有关分步说明（包括从 Claude Opus 5 及更早模型迁移），请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide)。

## 后续步骤

<CardGroup cols={3}>
  <Card title="模型概览" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/overview">
    每个当前 Claude 模型的规格和定价。
  </Card>

  <Card title="迁移指南" icon="code" href="https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide">
    从 Claude Fable 5、Claude Opus 5 及更早模型迁移。
  </Card>

  <Card title="Claude Fable 5.1 提示指南" icon="terminal" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1">
    Claude Fable 5.1 特有的提示模式。
  </Card>
</CardGroup>
