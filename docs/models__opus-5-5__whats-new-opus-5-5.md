---
title: Claude Opus 5.5 的新功能
url: https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5
description: Claude Opus 5.5 中的破坏性变更、功能支持和行为差异概述。
---

Claude Opus 5.5 专为长时间运行的智能体编码和知识工作而构建，定价为每百万输入/输出令牌 $4 / $20 美元。有四项"breaking changes"（破坏性变更）会影响已在 Claude Opus 5 上运行的代码：[无法禁用思考](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#thinking-cant-be-disabled)、[强制工具使用会返回错误](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#forced-tool-use-is-not-supported)、[思考块与模型和对话绑定](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#thinking-blocks-are-tied-to-the-model-that-produced-them)，以及在 Claude API 和 Google Cloud 上[不再接受早期的 `computer_20251124` 计算机使用工具](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#computer-20251124-is-not-supported)。前三项同样适用于 Claude Fable 5.1。另有一项变更会改变响应结构，但不会导致任何请求失败：[工具调用之间的文本以 `thinking` 块返回](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#text-between-tool-calls)，在默认的 `display` 设置下，这些块中的文本为空。如果应用程序将这些文本作为进度更新流式传输给用户，那么在它设置一个会返回文本的 `display` 值之前，它在工具调用之间将保持静默。

## 新模型

| 模型              | Claude API ID   | 描述                  |
| --------------- | --------------- | ------------------- |
| Claude Opus 5.5 | claude-opus-5-5 | 适用于长时间运行的智能体编码和知识工作 |

[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)始终开启，[effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)控制思考深度；该模型上的默认值为 `medium`。有关上下文窗口、输出限制、知识截止日期和价格，请参阅 [Claude Opus 5.5 模型页面](https://platform.claude.com/docs/zh-CN/models/opus-5-5/overview)；有关所有当前模型，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。

## 破坏性变更

### 无法禁用思考

在 Claude Opus 5 上，思考默认开启，并且在 effort 为 `high` 或更低时接受 `thinking: {"type": "disabled"}`。在 Claude Opus 5.5 上，思考始终开启：设置了 `thinking: {"type": "disabled"}` 的请求，或使用 `thinking: {"type": "enabled", "budget_tokens": N}` 手动设置预算的请求，都会返回 400 `invalid_request_error`。请省略 `thinking` 字段，或发送与之等效的 `thinking: {"type": "adaptive"}`。此变更不涉及任何 beta 标头。

错误消息如下：

```text wrap
"thinking.type.disabled" is not supported for this model. Use "thinking.type.adaptive" and "output_config.effort" to control thinking behavior.
```

```text wrap
"thinking.type.enabled" is not supported for this model. Use "thinking.type.adaptive" and "output_config.effort" to control thinking behavior.
```

[effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)是控制思考深度、延迟和成本的手段：在您之前禁用思考的地方降低该参数；[优化成本与智能](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#tune-effort)提供了用于选择级别的实测结果。由于每个响应都可能以一个或多个 `thinking` 块开头（在默认的 `display: "omitted"` 下，这些块返回时 `thinking` 字段为空），请按内容块的 `type` 字段而非位置来选择内容块，并在工具使用循环中原样传回 `thinking` 块。已在 Claude Opus 5 上开启思考运行的代码无需更改。请参阅[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)以及迁移指南中的[变更前后对比](https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide#thinking-cant-be-disabled)。

### 不支持强制工具使用

Claude Opus 5.5 不支持"forced tool use"（强制工具使用）。将 `tool_choice` 设置为 `{"type": "any"}` 或 `{"type": "tool", "name": "..."}` 会返回 400 `invalid_request_error`：

```text wrap
tool_choice: type "tool" and "any" are not supported for this model.
```

支持 `tool_choice: {"type": "auto"}`（默认值）和 `{"type": "none"}`，并且相同的验证也适用于[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)端点。如需符合 schema 的 JSON，请保留 `tool_choice: {"type": "auto"}` 并通过[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)设置 `strict: true`，或将 schema 移至[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)。若要让模型调用工具而不是以文本回复，请在提示中说明何时适用该工具。迁移指南展示了[变更前后对比](https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide#forced-tool-use)。

### 思考块与模型和对话绑定

每个思考块都会记录生成它的模型，每个模型都能读取自己的块，但只能读取部分其他模型的块。Claude Opus 5.5 可以读取来自 Claude Opus 5 以及更早的 Opus、Sonnet 和 Haiku 模型的思考块，但不能读取来自 Claude Fable 或 Claude Mythos 模型的思考块。在 Claude API 上，Claude Fable 5.1 和 Claude Mythos 5.1 可以读取来自 Claude Opus 5.5 的思考块；其他模型均不能。从 Claude Opus 5 切换到 Claude Opus 5.5 的对话，或在 Claude API 上从 Claude Opus 5.5 升级到 Claude Fable 5.1 或 Claude Mythos 5.1 的对话，会保留其推理内容。从 Claude Opus 5.5 切换到这两个模型以外的任何模型的对话，或从 Claude Fable 或 Claude Mythos 模型切换到 Claude Opus 5.5 的对话，在切换后的轮次中将不带有先前模型的推理内容。当请求携带目标模型无法读取的块时，API 会在模型看到之前将其丢弃：请求会成功，且被丢弃的块不计费。使用 `thinking-binding-controls-2026-08-01` beta 标头时，丢弃情况会在顶层的 `input_transformations` 数组中报告。请参阅[在对话中途切换模型](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#switching-models)。

API 还会检查 Claude Opus 5.5 思考块之前的任何内容（`system` 提示、`tools` 或更早的消息）自该块生成以来是否发生了变化。与 Claude Fable 5.1 一样，对于在 2026 年 8 月 31 日 00:00 UTC 或之后创建的账户，无论是在 Claude API 还是云平台上，API 都会默认强制执行该检查。在这些账户上，如果请求在发生此类变化后重放某个块，将返回 400 错误。若要改为丢弃受影响的块，请发送 `thinking-binding-controls-2026-08-01` beta 标头，并将 `thinking.block_binding.prefix_mismatch_behavior` 设置为 `"drop_block"`。在较早的账户上，将该字段设置为任一值都会使请求选择启用此检查。请保持对话只追加不修改，这样就不会出现此问题：使用[对话中途的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)来更改指令或工具，而不是进行编辑。请参阅[保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)以及迁移指南中[关于此变更的说明](https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide#thinking-blocks)。

### Claude API 和 Google Cloud 上不支持 `computer_20251124` 计算机使用工具

Claude Opus 5 既接受以 `computer_toolset_20260801` 工具集形式使用的[计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)，也接受在使用 `computer-use-2025-11-24` beta 标头时以早期 `computer_20251124` 工具形式使用的计算机使用。在 Claude API 和 Google Cloud 上，Claude Opus 5.5 仅支持工具集：声明了 `computer_20251124` 工具的请求会返回 400 `invalid_request_error`。该消息会指出被拒绝的类型，然后在 `Did you mean one of` 之后列出模型接受的工具类型（其中包括 `computer_toolset_20260801`）；其开头如下：

```text wrap
'claude-opus-5-5' does not support tool types: computer_20251124.
```

若要迁移 Claude API 或 Google Cloud 上的现有集成，请按照[从 `computer_20251124` 迁移](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#migrate-from-computer-20251124)操作：移除 beta 标头，将 `tools` 条目替换为 `{"type": "computer_toolset_20260801"}`，并更新您的智能体循环以处理成员 `tool_use` 块、批量操作以及结果中的 `toolset_name`。在 Amazon Bedrock 上，早期的 `computer_20251124` 工具在 Claude Opus 5.5 上仍可像在 Claude Opus 5 上一样正常工作，因此无需更改。有关其他平台，请参阅计算机使用工具的[兼容性](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#compatibility)部分。已经使用工具集的集成以及[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)无需更改。迁移指南展示了请求的[变更前后对比](https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide#computer-use-toolset)。

## 功能支持

Claude Opus 5.5 支持[按消息设置的 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)（beta）、[对话中途的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)、[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)、最小可缓存提示为 512 个令牌的[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)、[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)、[Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)、[PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)、[视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)，以及服务器端和客户端[工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。在 Claude API 和 Google Cloud 上，计算机使用需要 `computer_toolset_20260801` 工具集（请参阅[破坏性变更](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#computer-20251124-is-not-supported)）。有关模型可用性，请参阅各功能的页面。

### 快速模式

[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)（研究预览版）仅在 Claude API 上对 Claude Opus 5.5 可用；在 Amazon Bedrock、Claude Platform on AWS、Google Cloud 或 Microsoft Foundry 上不可用。请在使用 `fast-mode-2026-02-01` beta 标头的同时设置 `speed: "fast"`。有关访问权限、支持的模型和定价，请参阅[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)。

### 在消息中定义工具（beta）

使用 `inline-tools-2026-09-15` beta 标头时，对话中途系统消息中的 `tool_addition` 块可以携带完整的工具定义而非引用，因此您可以在对话中途添加工具、更改其 schema，或将服务器工具升级到更新版本，而无需编辑 `tools`，也不会丢失提示缓存。这适用于所有支持在对话中途更改工具的模型，包括 Claude Opus 5.5。请参阅[在消息中定义工具](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#define-tools-in-a-message-beta)。

### 按需压缩（beta）

使用 `compact-2026-09-04` beta 标头时，发送顶层 `compaction` 参数的请求会返回一个经过签名的 `compaction` 块，其中总结了整个对话，之后您可以将其放在最前面发送，以替代被总结的消息。该功能可用于支持压缩的模型，包括 Claude Opus 5.5。您可以自行选择何时压缩，请求可以在后台运行，并且您保留的轮次中的思考块在替换后仍可保持有效（需满足[压缩与保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks#conditions-for-kept-thinking-to-stay-valid)中的条件），这一点在 Claude Opus 5.5 上尤为重要，因为其[思考块与对话绑定](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#thinking-blocks-are-tied-to-the-model-that-produced-them)。有关平台可用性和完整的请求流程，请参阅[按需压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand)。

## 行为差异

Claude Opus 5.5 与 Claude Opus 5 在若干方面存在差异，这些差异无需任何代码更改就会显现。每项差异在[为 Claude Opus 5.5 编写提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5) 中都有相应指导：

* **默认 effort 为 `medium`。** 省略 `effort` 的请求以 `medium` 运行；在 Claude Opus 5 上则以 `high` 运行。请显式设置 `effort` 并重新运行您的参数扫描；请参阅[校准 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5#calibrate-effort)。
* **在给定 effort 级别下每轮思考更多。** 在相同的 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 设置下，该模型每轮的思考往往比 Claude Opus 5 更多，在 `xhigh` 和 `max` 下尤为明显。请重新运行 effort 扫描，而不是沿用原有设置，并在 `max_tokens` 中为思考预留空间。请参阅[校准 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5#calibrate-effort)。
* **工具调用之间的文本以思考块返回。** 模型在工具调用之间编写的简短说明会以[进度更新 `thinking` 块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)而非 `text` 块的形式返回，因此在默认的 `display: "omitted"` 下，将这些内容流式传输给用户的应用程序在工具调用之间会保持静默，且不会出现任何错误。[迁移指南](https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide#text-between-tool-calls)提供了接收这些内容的修复方法，[面向用户的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5#user-facing-progress-updates)介绍了如何请求更多此类更新。
* **更多安全防护类别。** 除网络安全分类器外，该模型还运行生物安全分类器，并且促使模型在响应文本中复现其内部推理的请求可能会以 `reasoning_extraction` 类别被拒绝。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#refusals-and-fallback)和[安全防护拒绝](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5#safeguard-refusals)。
* **对图表、示意图和屏幕截图的解读更加精准。** 该模型无需工具即可更精确地从密集图表和依赖布局的视觉内容中读取数值，因此为早期模型构建的提示端视觉变通方法可能不再需要；对于最密集的输入，图像工具仍能提高准确性。请参阅[用于复杂视觉输入的工具](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5#tools-for-complex-visual-inputs)。

如果您的 Claude Opus 5 集成是在禁用思考的情况下运行的，请结合[破坏性变更](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#thinking-cant-be-disabled)参阅[为禁用思考而编写的提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5#prompts-written-for-thinking-disabled)。有关智能体编码与代码审查、知识工作、沟通、视觉输入和计算机使用方面的能力提升，请参阅[与提示相关的能力](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5#capability-improvements)。

## 拒绝与回退

Claude Opus 5.5 附带安全分类器，[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)中的所有内容均适用。被拒绝的请求会返回 HTTP 200，其中包含 `stop_reason: "refusal"` 以及一个指明政策领域的 [`stop_details`](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response) 对象，因此请处理拒绝并配置回退：通过[服务器端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)（`fallbacks: "default"`，处于 beta 阶段，会在 Anthropic 针对该类别推荐的模型上重试）、[SDK 中间件](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#client-side-fallback)或您自己的重试逻辑在另一个模型上重试。在产生任何输出之前发生的拒绝是否计费取决于其拒绝类别，并且无论哪种情况都会计入您的速率限制；请参阅[拒绝如何计费](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#how-refusals-are-billed)。

## 定价

Claude Opus 5.5 的价格为每百万输入令牌 $4 美元、每百万输出令牌 $20 美元，低于 Claude Opus 5 的 $5 和 $25；5 分钟缓存写入为每百万令牌 $5，1 小时缓存写入为 $8，缓存读取为 $0.20（基础输入价格的 0.05 倍）。[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)为半价：$2 和 $10。有关数据驻留和工具定价，请参阅[定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

## 可用性

Claude Opus 5.5 可在以下平台使用：

* **Claude API：** 面向所有客户，模型 ID 为 `claude-opus-5-5`。
* **AWS：** [Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)，模型 ID 为 `anthropic.claude-opus-5-5`；以及 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)，模型 ID 为 `claude-opus-5-5`。
* **Google Cloud：** [Claude on Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)，模型 ID 为 `claude-opus-5-5`。
* **Microsoft Foundry：** [Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)，模型 ID 为 `claude-opus-5-5`。

## 从 Claude Opus 5 迁移

更新您的模型 ID：

<CodeGroup exclude="shell">
  ```python Python
  model = "claude-opus-5"  # Before
  model = "claude-opus-5-5"  # After
  ```

  ```typescript TypeScript
  let model = "claude-opus-5"; // Before
  model = "claude-opus-5-5"; // After
  ```

  ```csharp C#
  var model = Model.ClaudeOpus5; // Before
  model = Model.ClaudeOpus5_5; // After
  ```

  ```go Go
  model := anthropic.ModelClaudeOpus5  // Before
  model = anthropic.ModelClaudeOpus5_5 // After
  ```

  ```java Java
  Model model = Model.CLAUDE_OPUS_5; // Before
  model = Model.CLAUDE_OPUS_5_5; // After
  ```

  ```php PHP
  $model = Model::CLAUDE_OPUS_5; // Before
  $model = Model::CLAUDE_OPUS_5_5; // After
  ```

  ```ruby Ruby
  model = Anthropic::Model::CLAUDE_OPUS_5 # Before
  model = Anthropic::Model::CLAUDE_OPUS_5_5 # After
  ```
</CodeGroup>

然后移除所有 `thinking: {"type": "disabled"}` 或 `thinking: {"type": "enabled", ...}` 设置，改为选择一个 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别。将 `tool_choice` 类型 `any` 和 `tool` 替换为 `auto` 并配合[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)。如果您在 Claude API 或 Google Cloud 上通过 `computer_20251124` 使用计算机使用功能，请[迁移到工具集](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#migrate-from-computer-20251124)。如果您的界面会显示工具调用之间的文本，还需设置 `thinking.display`；请参阅[工具调用之间的文本在思考块中返回](https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide#text-between-tool-calls)。有关从 Claude Opus 5 及更早模型迁移的分步说明以及完整的检查清单，请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide)。

## 后续步骤

<CardGroup cols={3}>
  <Card title="模型概览" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/overview">
    所有当前 Claude 模型的完整规格和定价。
  </Card>

  <Card title="迁移指南" icon="code" href="https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide">
    将代码从 Claude Opus 5 及更早模型迁移到 Claude Opus 5.5。
  </Card>

  <Card title="为 Claude Opus 5.5 编写提示" icon="terminal" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5">
    Claude Opus 5.5 特有的行为差异和提示模式。
  </Card>

  <Card title="Effort" icon="gauge" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    控制 Claude 在响应时使用的令牌数量，从 low 到 max。
  </Card>

  <Card title="思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    自适应思考的工作原理以及思考块的保留方式。
  </Card>

  <Card title="拒绝与回退" icon="shield" href="https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback">
    处理 `stop_reason: "refusal"` 并在另一个模型上重试。
  </Card>
</CardGroup>
