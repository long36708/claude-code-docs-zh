---
title: 迁移到 Claude Mythos 5 和 Claude Fable 5
url: https://platform.claude.com/docs/zh-CN/models/fable-5/migration-guide
description: 从 Claude Mythos Preview、Claude Opus 5 或 Claude Opus 4.8 迁移到 Claude Mythos 5 和 Claude Fable 5：模型 ID、API 变更和迁移清单。
---

<Note>
  本指南涵盖 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 代码的迁移。如果您使用 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)，则除了更新模型名称之外无需进行任何更改。
</Note>

<Tip>
  **使用 Claude API skill 自动完成迁移。** 在 Claude Code 中，运行 `/claude-api migrate` 以调用内置的 [Claude API skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model)。它适用于以任何当前 Claude 模型作为目标：

  ```text wrap
  /claude-api migrate this project to claude-fable-5
  ```

  该 skill 会在您的整个代码库中应用模型 ID 替换，并根据需要处理破坏性参数变更、prefill（预填充）替换以及针对目标模型的 effort（努力程度）校准，然后生成一份需要手动验证的事项清单。在编辑任何文件之前，它会要求您确认迁移范围（整个工作目录、某个子目录或特定的文件列表）。该 skill 还会检测 Amazon Bedrock 和 Claude Platform on AWS 客户端，并针对这些平台调整模型 ID 格式和功能变更。
</Tip>

[Claude Fable 5](https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5) 专为高要求的推理和长周期智能体工作而打造。[Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide) 在其基础上进一步构建。Claude Fable 5 可在 Claude API、[Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)、[Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 和 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上使用。[Claude Mythos 5](https://anthropic.com/glasswing) 具有相同的能力，仅向 Project Glasswing 中经批准的客户提供。

`claude-fable-5` 和 `claude-mythos-5` 共享的基线设置：

* **思考：** "Adaptive thinking"（[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)）始终开启。模型会决定在每个请求中何时思考以及思考多少，无需任何 `thinking` 配置。`thinking: {type: "disabled"}` 和手动扩展思考（`thinking: {type: "enabled", budget_tokens: N}`）都会返回 400 错误。
* **预填充：** 对助手消息进行 "prefill"（预填充）会返回 400 错误。请改用系统提示指令。
* **上下文窗口和输出：** 默认提供 [100 万令牌的 "context window"（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)，每个请求最多可输出 128k 个令牌。
* **定价：** 每百万输入令牌 10 美元，每百万输出令牌 50 美元。请参阅 [Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。
* **数据保留：** 这两个模型都要求 30 天数据保留，除非获得 Anthropic 的明确授权，否则不适用于 "zero data retention"（零数据保留），即 ZDR 安排。两者均被指定为受管控模型（Covered Models）。在 Claude API 上，如果组织的数据保留配置不满足此要求，其向 Claude Fable 5 发出的请求将返回 400 `invalid_request_error`。拥有 ZDR 安排的组织应联系其 Anthropic 客户团队讨论数据保留配置，或按工作区配置数据保留。有关各平台的详细信息，请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。

两个模型的不同之处：

* **可用性：** Claude Fable 5 无需访问审批。Claude Mythos 5 仅向 [Project Glasswing](https://anthropic.com/glasswing) 中经批准的客户提供。
* **安全分类器：** Claude Fable 5 运行 "safety classifiers"（安全分类器），可能会以 `stop_reason: "refusal"` 拒绝请求。Claude Mythos 5 不包含这些分类器。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。
* **Priority Tier：** Claude Fable 5 支持 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models)，但 Claude Mythos 5 不支持。

## 从 Claude Mythos Preview 迁移到 Claude Mythos 5 和 Claude Fable 5

[Claude Mythos 5](https://anthropic.com/glasswing) 是 [Claude Mythos Preview](https://anthropic.com/glasswing)（仅限受邀参与的研究预览版）的受访问限制的后继模型。[Claude Fable 5](https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5) 提供相同的能力，且无需访问审批。本节中的变更同样适用于这两个目标模型。

迁移基本上可以直接替换。Claude Mythos 5 和 Claude Fable 5 使用与 Claude Mythos Preview 相同的 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 和相同的[工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)模式，并且由于这三个模型使用相同的 "tokenizer"（分词器），令牌数量大致不变。需要检查的关键变更是不再可用的功能（在下一节中列出）以及思考输出。如果您迁移到 Claude Fable 5，还需要为安全分类器拒绝做好准备，Claude Mythos Preview 和 Claude Mythos 5 没有这类拒绝；请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。

有关 Claude Mythos Preview 的停用时间表，请参阅[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)。

### 更新您的模型名称

```python
model = "claude-mythos-preview"  # Before
model = "claude-mythos-5"  # After

# 或者，使用功能相同且无需访问审批的模型：
model = "claude-fable-5"  # After
```

### Claude Mythos 5 和 Claude Fable 5 上不可用的功能

1. **扩展思考和思考令牌预算：** `claude-mythos-5` 和 `claude-fable-5` 不支持手动扩展思考（`thinking: {type: "enabled", budget_tokens: N}`），使用时会返回 400 错误。[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)始终开启：模型会决定在每个请求中何时思考以及思考多少，无需任何 `thinking` 配置。`thinking: {type: "disabled"}` 会返回错误。`budget_tokens` 没有直接的替代项：思考是自适应的，而 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)是一个独立的输出级控制，并非思考预算。

   之前（Claude Mythos Preview）：

   <CodeGroup>
     ```bash cURL
     curl https://api.anthropic.com/v1/messages \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -H "content-type: application/json" \
       -d '{
         "model": "claude-mythos-preview",
         "max_tokens": 16000,
         "thinking": {
           "type": "enabled",
           "budget_tokens": 10000
         },
         "messages": [
           {
             "role": "user",
             "content": "..."
           }
         ]
       }'
     ```

     ```bash CLI
     ant messages create <<'YAML'
     model: claude-mythos-preview
     max_tokens: 16000
     thinking:
       type: enabled
       budget_tokens: 10000
     messages:
       - role: user
         content: "..."
     YAML
     ```

     ```python Python
     client.messages.create(
         model="claude-mythos-preview",
         max_tokens=16000,
         thinking={"type": "enabled", "budget_tokens": 10000},
         messages=[{"role": "user", "content": "..."}],
     )
     ```

     ```typescript TypeScript
     await client.messages.create({
       model: "claude-mythos-preview",
       max_tokens: 16000,
       thinking: { type: "enabled", budget_tokens: 10000 },
       messages: [{ role: "user", content: "..." }]
     });
     ```

     ```csharp C#
     using Anthropic;
     using Anthropic.Models.Messages;

     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = "claude-mythos-preview",
         MaxTokens = 16000,
         Thinking = new ThinkingConfigEnabled(budgetTokens: 10000),
         Messages = [new() { Role = Role.User, Content = "..." }]
     };

     var response = await client.Messages.Create(parameters);
     Console.WriteLine(response);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     "claude-mythos-preview",
     	MaxTokens: 16000,
     	Thinking:  anthropic.ThinkingConfigParamOfEnabled(10000),
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("...")),
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response)
     ```

     ```java Java
     AnthropicClient client = AnthropicOkHttpClient.fromEnv();

     MessageCreateParams params = MessageCreateParams.builder()
         .model("claude-mythos-preview")
         .maxTokens(16000L)
         .enabledThinking(10000L)
         .addUserMessage("...")
         .build();

     Message response = client.messages().create(params);
     IO.println(response);
     ```

     ```php PHP
     $client = new Client();

     $message = $client->messages->create(
         maxTokens: 16000,
         messages: [['role' => 'user', 'content' => '...']],
         model: 'claude-mythos-preview',
         thinking: ['type' => 'enabled', 'budget_tokens' => 10000],
     );
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     message = client.messages.create(
       model: "claude-mythos-preview",
       max_tokens: 16000,
       thinking: {
         type: "enabled",
         budget_tokens: 10000
       },
       messages: [
         { role: "user", content: "..." }
       ]
     )
     ```
   </CodeGroup>

   之后（Claude Mythos 5）：

   <CodeGroup>
     ```bash cURL
     curl https://api.anthropic.com/v1/messages \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -H "content-type: application/json" \
       -d '{
         "model": "claude-mythos-5",
         "max_tokens": 16000,
         "messages": [
           {
             "role": "user",
             "content": "..."
           }
         ]
       }'
     ```

     ```bash CLI
     ant messages create <<'YAML'
     model: claude-mythos-5
     max_tokens: 16000
     messages:
       - role: user
         content: "..."
     YAML
     ```

     ```python Python
     client.messages.create(
         model="claude-mythos-5",
         max_tokens=16000,
         messages=[{"role": "user", "content": "..."}],
     )
     ```

     ```typescript TypeScript
     await client.messages.create({
       model: "claude-mythos-5",
       max_tokens: 16000,
       messages: [{ role: "user", content: "..." }]
     });
     ```

     ```csharp C#
     using Anthropic;
     using Anthropic.Models.Messages;

     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = "claude-mythos-5",
         MaxTokens = 16000,
         Messages = [new() { Role = Role.User, Content = "..." }]
     };

     var response = await client.Messages.Create(parameters);
     Console.WriteLine(response);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     "claude-mythos-5",
     	MaxTokens: 16000,
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("...")),
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response)
     ```

     ```java Java
     AnthropicClient client = AnthropicOkHttpClient.fromEnv();

     MessageCreateParams params = MessageCreateParams.builder()
         .model("claude-mythos-5")
         .maxTokens(16000L)
         .addUserMessage("...")
         .build();

     Message response = client.messages().create(params);
     IO.println(response);
     ```

     ```php PHP
     $client = new Client();

     $message = $client->messages->create(
         maxTokens: 16000,
         messages: [['role' => 'user', 'content' => '...']],
         model: 'claude-mythos-5',
     );
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     message = client.messages.create(
       model: "claude-mythos-5",
       max_tokens: 16000,
       messages: [
         { role: "user", content: "..." }
       ]
     )
     ```
   </CodeGroup>

   Claude Fable 5 的变更完全相同，只需将模型名称改为 `claude-fable-5`。

2. **助手预填充：** `claude-mythos-5` 和 `claude-fable-5` 不支持对助手消息进行预填充，使用时会返回 400 错误，与 Claude Mythos Preview 相同。请改用系统提示指令。

3. **思考输出：** 在 `claude-mythos-5` 和 `claude-fable-5` 上，原始思维链永远不会返回，但当 `thinking.display` 设置为 `summarized` 时，思考块仍会包含可读的摘要文本。在同一模型上继续对话时，请原样传回思考块。请参阅 [Claude Fable 和 Claude Mythos 模型上的思考输出](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-output-on-claude-fable-5-and-claude-mythos-5)。

### 令牌计数和计费

`claude-mythos-5` 和 `claude-fable-5` 使用与 `claude-mythos-preview` 相同的分词器（随 Claude Opus 4.7 引入的分词器）。从 `claude-mythos-preview` 迁移时，令牌数量大致不变。与 Claude Opus 4.7 之前的模型相比，相同内容分词后的令牌数量可能多出约 30%，具体因内容和工作负载形态而异。

与 `claude-mythos-preview` 相比，[`/v1/messages/count_tokens`](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting) 为 `claude-mythos-5` 和 `claude-fable-5` 返回的值大致不变。请在您自己的工作负载上重新建立成本和延迟基线。

### 迁移清单

* 将模型名称从 `claude-mythos-preview` 更新为 `claude-mythos-5`，或更新为 `claude-fable-5`（它提供相同的能力且无需访问审批）。
* 移除手动扩展思考配置（`thinking: {type: "enabled", budget_tokens: N}`）。自适应思考始终开启，无需任何 `thinking` 字段。
* 移除所有 `thinking: {type: "disabled"}` 配置。在 `claude-mythos-5` 和 `claude-fable-5` 上禁用思考会返回错误。
* 移除 `budget_tokens`。它没有直接的替代项：思考是自适应的，而 `effort` 参数是一个独立的输出级控制，并非思考预算。
* 确认所有解析 `thinking` 字段的代码仅将其视为显示文本，并在同一模型上继续对话时原样传回思考块。在 `claude-mythos-5` 和 `claude-fable-5` 上，`thinking.display` 默认为 `"omitted"`，与 Claude Mythos Preview 相同。设置 `display: "summarized"` 以接收可读的摘要。请参阅 [Claude Fable 和 Claude Mythos 模型上的思考输出](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-output-on-claude-fable-5-and-claude-mythos-5)。
* 如果您要在较早的模型上重放对话历史，请先从之前的助手轮次中剥离 `thinking` 和 `redacted_thinking` 块。来自 `claude-fable-5` 和 `claude-mythos-5` 的思考块只能由生成它们的模型或更新的模型读取：较早的模型会静默忽略它们，而 Claude Fable 5.1 和 Claude Mythos 5.1 会读取它们，因此当您将对话升级到这些模型时请保留它们（请参阅[在对话中途切换模型](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#switching-models)）。剥离这些块可使发往较早模型的请求保持精简和统一。
* 如果您迁移到 Claude Fable 5，请处理 `stop_reason: "refusal"` 并读取 `stop_details.category` 字段。Claude Fable 5 运行 Claude Mythos Preview 和 Claude Mythos 5 所没有的安全分类器。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。
* 在您自己的工作负载上重新建立令牌数量和成本基线。从 `claude-mythos-preview` 迁移时，令牌数量大致不变。

## 从 Claude Opus 5 迁移到 Claude Mythos 5 和 Claude Fable 5

Claude Fable 5 和 Claude Mythos 5 使用与 Claude Opus 5 相同的 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 和相同的[工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)模式，默认提供相同的 [100 万令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)，以及相同的 [128k 最大输出令牌数](https://platform.claude.com/docs/zh-CN/models/overview)。预填充和采样参数限制以及思考显示行为均沿用 Claude Opus 5，保持不变。需要检查的变更包括始终开启的思考、定价、Priority Tier 和数据保留。

### 更新您的模型名称

```python
model = "claude-opus-5"  # Before
model = "claude-fable-5"  # After

# 或者，对于具有相同功能的 Project Glasswing 模型：
model = "claude-mythos-5"  # After
```

### 变更内容

1. **思考无法再被禁用：** 在 Claude Opus 5 上，思考默认开启，并且在 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别为 `high` 或更低时，可以通过 `thinking: {type: "disabled"}` 关闭。在 `claude-fable-5` 和 `claude-mythos-5` 上，[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)始终开启，在任何 effort 级别下 `thinking: {type: "disabled"}` 都会返回 400 错误。请移除 `thinking: {type: "disabled"}` 配置，改用较低的 effort 级别来控制令牌消耗。

   如果您的 Claude Opus 5 请求禁用了思考，响应结构会发生变化：响应可能在第一个 `text` 块之前以一个或多个 `thinking` 块开头，在默认的 `display: "omitted"`（与 Claude Opus 5 的默认值相同）下，这些块返回时 `thinking` 字段为空。按位置读取回复的代码（例如 `content[0].text`，或将第一个内容块视为文本的流处理程序）必须改为按 `type` 字段选择内容块，并且工具使用循环必须将 `thinking` 块与其工具结果一起完整且不加修改地传回。API 会以 400 错误拒绝经过编辑、重新排序或部分丢弃的思考块（请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)）。即使未返回思考文本，思考令牌也会按输出令牌计费。

2. **定价：** Claude Fable 5 和 Claude Mythos 5 的定价为每百万输入令牌 10 美元、每百万输出令牌 50 美元，而 Claude Opus 5 为 5 美元和 25 美元。请参阅 [Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

3. **Priority Tier：** Claude Opus 5 不支持 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models)，因此不会影响任何现有流量。如果您的组织有 Priority Tier 承诺，Claude Fable 5 支持该承诺；Claude Mythos 5 则不支持。

4. **数据保留：** Claude Fable 5 和 Claude Mythos 5 要求 30 天数据保留，除非获得 Anthropic 的明确授权，否则不适用于零数据保留（ZDR）安排。两者均被指定为受管控模型（Covered Models）。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。

### 迁移清单

* 将模型名称从 `claude-opus-5` 更新为 `claude-fable-5`（或 `claude-mythos-5`）。
* 移除所有 `thinking: {type: "disabled"}` 配置；它在 `claude-fable-5` 和 `claude-mythos-5` 上会返回 400 错误。请改用较低的 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别来控制令牌消耗，并针对在 Claude Opus 5 上禁用思考运行的工作负载重新审视 `max_tokens`。
* 如果这些工作负载按位置读取内容（例如 `content[0].text`），请将其更新为按 `type` 选择内容块：`thinking` 块现在会出现在 `text` 块之前。在工具使用循环中完整且不加修改地传回 `thinking` 块；经过修改的块会返回 400 错误。
* 如果您的组织有零数据保留（ZDR）安排，请在迁移前确认资格：除非获得 Anthropic 的明确授权，否则这些模型不适用于 ZDR。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。
* 在您自己的工作负载上重新建立成本基线。令牌数量大致不变；每令牌定价有所不同，并且之前禁用思考运行的工作负载现在会产生思考令牌，这些令牌按输出令牌计费。

## 从 Claude Opus 4.8 迁移到 Claude Mythos 5 和 Claude Fable 5

<Note>
  如果您的代码使用的是 Claude Opus 4.7 或更早版本，请先应用[迁移到 Claude Opus 5.5](https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide) 中与您当前模型相关的迁移章节来处理 API 级别的变更，然后再应用本节中剩余的差异。
</Note>

迁移基本上可以直接替换。Claude Fable 5 和 Claude Mythos 5 使用与 Claude Opus 4.8 相同的 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 和相同的[工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)模式，默认提供相同的 [100 万令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)，以及相同的 [128k 最大输出令牌数](https://platform.claude.com/docs/zh-CN/models/overview)。由于这些模型使用相同的分词器，令牌数量大致不变。需要检查的关键变更包括始终开启的[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)、思考输出、安全分类器拒绝（仅限 Claude Fable 5）以及定价。

### 更新您的模型名称

```python
model = "claude-opus-4-8"  # Before
model = "claude-fable-5"  # After

# 或者，对于具有相同功能的 Project Glasswing 模型：
model = "claude-mythos-5"  # After
```

### 变更内容

本节中的各项描述了在您替换模型 ID 后值得检查的 API 和行为差异。除非另有说明，这些差异同样适用于 `claude-fable-5` 和 `claude-mythos-5`。

1. **自适应思考始终开启：** [自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)是 `claude-fable-5` 和 `claude-mythos-5` 上唯一的思考模式。模型会决定在每个请求中何时思考以及思考多少，无需任何 `thinking` 配置。`thinking: {type: "disabled"}` 会返回错误。请使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)来控制思考深度。

   需要检查的行为变化：在 Claude Opus 4.8 上，不带 `thinking` 字段的请求在不思考的情况下运行；在 `claude-fable-5` 和 `claude-mythos-5` 上，同样的请求会以自适应思考运行。`max_tokens` 仍然是总输出（思考加响应文本）的硬性上限，因此请针对在 Claude Opus 4.8 上不思考运行的工作负载重新审视该值。请参阅[成本控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#cost-control)。响应还可能在第一个 `text` 块之前以一个或多个 `thinking` 块开头，因此按位置读取回复的代码（例如 `content[0].text`，或将第一个内容块视为文本的流处理程序）必须改为按 `type` 字段选择内容块。即使思考文本未返回给您，思考令牌也会按输出令牌计费，因此在 Claude Opus 4.8 上不思考运行的工作负载每个请求会产生更多输出令牌，此外还有每令牌价格的差异。

   如果您运行工具使用循环，在返回工具结果时，请将每个助手响应中的 `thinking` 块完整且不加修改地传回 API，包括 `thinking` 字段为空的块。请按接收时的原样回传助手消息，而不是按类型过滤其内容块或重新构建它：API 会以 400 错误拒绝经过编辑、重新排序或部分丢弃的思考块。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。

   之前（Claude Opus 4.8）：

   <CodeGroup>
     ```bash cURL
     curl https://api.anthropic.com/v1/messages \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -H "content-type: application/json" \
       -d '{
         "model": "claude-opus-4-8",
         "max_tokens": 16000,
         "thinking": {
           "type": "adaptive"
         },
         "output_config": {
           "effort": "high"
         },
         "messages": [
           {
             "role": "user",
             "content": "..."
           }
         ]
       }'
     ```

     ```bash CLI
     ant messages create <<'YAML'
     model: claude-opus-4-8
     max_tokens: 16000
     thinking:
       type: adaptive
     output_config:
       effort: high
     messages:
       - role: user
         content: "..."
     YAML
     ```

     ```python Python
     client.messages.create(
         model="claude-opus-4-8",
         max_tokens=16000,
         thinking={"type": "adaptive"},
         output_config={"effort": "high"},
         messages=[{"role": "user", "content": "..."}],
     )
     ```

     ```typescript TypeScript
     await client.messages.create({
       model: "claude-opus-4-8",
       max_tokens: 16000,
       thinking: { type: "adaptive" },
       output_config: { effort: "high" },
       messages: [{ role: "user", content: "..." }]
     });
     ```

     ```csharp C#
     using Anthropic;
     using Anthropic.Models.Messages;

     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = "claude-opus-4-8",
         MaxTokens = 16000,
         Thinking = new ThinkingConfigAdaptive(),
         OutputConfig = new OutputConfig { Effort = Effort.High },
         Messages = [new() { Role = Role.User, Content = "..." }]
     };

     var response = await client.Messages.Create(parameters);
     Console.WriteLine(response);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     "claude-opus-4-8",
     	MaxTokens: 16000,
     	Thinking: anthropic.ThinkingConfigParamUnion{
     		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
     	},
     	OutputConfig: anthropic.OutputConfigParam{
     		Effort: anthropic.OutputConfigEffortHigh,
     	},
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("...")),
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response)
     ```

     ```java Java
     AnthropicClient client = AnthropicOkHttpClient.fromEnv();

     MessageCreateParams params = MessageCreateParams.builder()
         .model("claude-opus-4-8")
         .maxTokens(16000L)
         .thinking(ThinkingConfigAdaptive.builder().build())
         .outputConfig(OutputConfig.builder()
             .effort(OutputConfig.Effort.HIGH)
             .build())
         .addUserMessage("...")
         .build();

     Message response = client.messages().create(params);
     IO.println(response);
     ```

     ```php PHP
     $client = new Client();

     $message = $client->messages->create(
         maxTokens: 16000,
         messages: [['role' => 'user', 'content' => '...']],
         model: 'claude-opus-4-8',
         thinking: ['type' => 'adaptive'],
         outputConfig: ['effort' => 'high'],
     );
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     message = client.messages.create(
       model: "claude-opus-4-8",
       max_tokens: 16000,
       thinking: {
         type: "adaptive"
       },
       output_config: {
         effort: "high"
       },
       messages: [
         { role: "user", content: "..." }
       ]
     )
     ```
   </CodeGroup>

   之后（Claude Fable 5）：

   <CodeGroup>
     ```bash cURL
     curl https://api.anthropic.com/v1/messages \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -H "content-type: application/json" \
       -d '{
         "model": "claude-fable-5",
         "max_tokens": 16000,
         "output_config": {
           "effort": "high"
         },
         "messages": [
           {
             "role": "user",
             "content": "..."
           }
         ]
       }'
     ```

     ```bash CLI
     ant messages create <<'YAML'
     model: claude-fable-5
     max_tokens: 16000
     output_config:
       effort: high
     messages:
       - role: user
         content: "..."
     YAML
     ```

     ```python Python
     client.messages.create(
         model="claude-fable-5",
         max_tokens=16000,
         output_config={"effort": "high"},
         messages=[{"role": "user", "content": "..."}],
     )
     ```

     ```typescript TypeScript
     await client.messages.create({
       model: "claude-fable-5",
       max_tokens: 16000,
       output_config: { effort: "high" },
       messages: [{ role: "user", content: "..." }]
     });
     ```

     ```csharp C#
     using Anthropic;
     using Anthropic.Models.Messages;

     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = "claude-fable-5",
         MaxTokens = 16000,
         OutputConfig = new OutputConfig { Effort = Effort.High },
         Messages = [new() { Role = Role.User, Content = "..." }]
     };

     var response = await client.Messages.Create(parameters);
     Console.WriteLine(response);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     "claude-fable-5",
     	MaxTokens: 16000,
     	OutputConfig: anthropic.OutputConfigParam{
     		Effort: anthropic.OutputConfigEffortHigh,
     	},
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("...")),
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response)
     ```

     ```java Java
     AnthropicClient client = AnthropicOkHttpClient.fromEnv();

     MessageCreateParams params = MessageCreateParams.builder()
         .model("claude-fable-5")
         .maxTokens(16000L)
         .outputConfig(OutputConfig.builder()
             .effort(OutputConfig.Effort.HIGH)
             .build())
         .addUserMessage("...")
         .build();

     Message response = client.messages().create(params);
     IO.println(response);
     ```

     ```php PHP
     $client = new Client();

     $message = $client->messages->create(
         maxTokens: 16000,
         messages: [['role' => 'user', 'content' => '...']],
         model: 'claude-fable-5',
         outputConfig: ['effort' => 'high'],
     );
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     message = client.messages.create(
       model: "claude-fable-5",
       max_tokens: 16000,
       output_config: {
         effort: "high"
       },
       messages: [
         { role: "user", content: "..." }
       ]
     )
     ```
   </CodeGroup>

   Claude Mythos 5 的变更完全相同，只需将模型名称改为 `claude-mythos-5`。

2. **扩展思考和思考预算（未变）：** `claude-fable-5` 和 `claude-mythos-5` 不支持手动扩展思考（`thinking: {type: "enabled", budget_tokens: N}`），使用时会返回 400 错误，与 Claude Opus 4.8 相同。`budget_tokens` 没有直接的替代项：思考是自适应的，而 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)是一个独立的输出级控制，并非思考预算。

3. **助手预填充（未变）：** `claude-fable-5` 和 `claude-mythos-5` 不支持对助手消息进行预填充，使用时会返回 400 错误，与 Claude Opus 4.8 相同。请改用系统提示指令。

4. **思考输出：** 在 `claude-fable-5` 和 `claude-mythos-5` 上，原始思维链永远不会返回，但当 `thinking.display` 设置为 `summarized` 时，思考块仍会包含可读的摘要文本。在同一模型上继续对话时，请原样传回思考块。请参阅 [Claude Fable 和 Claude Mythos 模型上的思考输出](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-output-on-claude-fable-5-and-claude-mythos-5)。

5. **安全分类器和 `refusal` 停止原因（仅限 Claude Fable 5）：** `claude-fable-5` 会对请求以及在响应生成过程中运行安全分类器。Claude Mythos 5 不包含这些分类器。当分类器拒绝请求时，Messages API 会以成功的 HTTP 200 响应返回 `stop_reason: "refusal"`，而不是错误。`stop_details.category` 字段会报告触发的是哪个分类器，类别包括 `"cyber"`、`"bio"` 和 `"reasoning_extraction"` 等；当拒绝不对应任何命名类别时，该字段为 `null`。完整类别集请参阅[拒绝类别表](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response)。

   在产生任何输出之前发生的拒绝，如果其类别为 `"bio"`、`"frontier_llm"` 或 `"reasoning_extraction"`，则会计费。在产生任何输出之前发生的其他类别或 `null` 类别的拒绝不计费（[拒绝如何计费](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#how-refusals-are-billed)）。在 2026 年 9 月 24 日之前，Claude Fable 5 上在产生任何输出之前发生的拒绝均不计费。当分类器在流式传输过程中触发时，输入和已流式传输的输出会被计费；请丢弃部分输出。

   要自动在另一个模型上重新运行被拒绝的请求，请传入可选启用的 `fallbacks` 参数，该参数在 Claude API 上处于 beta 阶段。该参数在 Message Batches API 以及 Amazon Bedrock、Google Cloud 和 Microsoft Foundry 上不可用；在这三个平台上，请在客户端执行重试，或使用 SDK 的拒绝回退中间件。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。

6. **从 `high` effort 开始：** [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)的默认值仍为 `high`。在 Claude Opus 4.8 上，对于编码和高自主性工作，建议显式设置 `xhigh`。在 `claude-fable-5` 和 `claude-mythos-5` 上，大多数任务请使用 `high` 作为默认值，并将 `xhigh` 保留给对能力最敏感的工作负载。较低的 effort 设置仍然表现良好，并且通常超过之前模型在 `xhigh` 下的表现。如果任务能够完成但耗时超出必要，请降低 effort。请参阅[为 Claude Fable 5 编写提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5#consider-all-effort-levels)。

7. **更低的提示缓存最小值：** `claude-fable-5` 和 `claude-mythos-5` 上的最小可缓存提示长度为 512 个令牌，低于 Claude Opus 4.8 上的 1,024 个令牌。在 Claude Opus 4.8 上因过短而无法缓存的提示现在可以创建缓存条目，无需更改代码。有关各模型的最小值，请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#cache-limitations)。

### 迁移清单

* 如果您的组织有零数据保留（ZDR）安排，请在迁移前确认资格。`claude-fable-5` 和 `claude-mythos-5` 要求 30 天数据保留，除非获得 Anthropic 的明确授权，否则不适用于 ZDR。在 Claude API 上，不满足此要求的 `claude-fable-5` 请求会返回 400 `invalid_request_error`。Claude Opus 4.8 可在 ZDR 下使用。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。
* 将模型名称从 `claude-opus-4-8` 更新为 `claude-fable-5`（或 `claude-mythos-5`）。
* 移除所有 `thinking: {type: "disabled"}` 配置。在 `claude-fable-5` 和 `claude-mythos-5` 上禁用思考会返回错误，并且不带 `thinking` 字段的请求会以自适应思考运行。
* 更新按位置读取内容的响应解析代码（例如 `content[0].text`）：由于自适应思考始终开启，`thinking` 块会出现在 `text` 块之前。请改为按 `type` 选择内容块，并在工具使用循环中完整且不加修改地传回 `thinking` 块；经过修改的块会返回 400 错误。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。
* 如果您在之前的迁移中已移除手动扩展思考和助手预填充，则无需任何操作：两者在 `claude-fable-5` 和 `claude-mythos-5` 上仍不受支持。
* 确认所有解析 `thinking` 字段的代码仅将其视为显示文本，并在同一模型上继续对话时原样传回思考块。在 `claude-fable-5` 和 `claude-mythos-5` 上，`thinking.display` 默认为 `"omitted"`，与 Claude Opus 4.8 相同。设置 `display: "summarized"` 以接收可读的摘要。请参阅 [Claude Fable 和 Claude Mythos 模型上的思考输出](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-output-on-claude-fable-5-and-claude-mythos-5)。
* 如果您要在较早的模型上重放对话历史，请先从之前的助手轮次中剥离 `thinking` 和 `redacted_thinking` 块。来自 `claude-fable-5` 和 `claude-mythos-5` 的思考块只能由生成它们的模型或更新的模型读取：较早的模型会静默忽略它们，而 Claude Fable 5.1 和 Claude Mythos 5.1 会读取它们，因此当您将对话升级到这些模型时请保留它们（请参阅[在对话中途切换模型](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#switching-models)）。剥离这些块可使发往较早模型的请求保持精简和统一。例外情况是兑换[回退额度](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)，这要求按照该功能的确切规则回传请求体。
* 如果您迁移到 Claude Fable 5，请处理 `stop_reason: "refusal"` 并读取 `stop_details.category` 字段。要自动在另一个模型上重新运行被拒绝的请求，请考虑使用可选启用的 `fallbacks` 参数（beta）。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。
* 重新评估您的 `effort` 设置。大多数任务请从 `high` 开始，包括在 Claude Opus 4.8 上以 `xhigh` 运行的工作负载。
* 在您自己的工作负载上重新建立成本和延迟基线。从 `claude-opus-4-8` 迁移时，令牌数量大致不变；每令牌定价有所不同，并且思考令牌按输出令牌计费，因此之前不思考运行的工作负载每个请求会产生更多输出令牌。
