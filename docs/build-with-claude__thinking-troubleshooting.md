---
title: 思考功能故障排查
url: https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting
description: 诊断并修复最常见的思考功能故障：配置导致的 400 错误、空的或缺失的思考块、max_tokens 停止以及缓存未命中。
---

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

本页涵盖配置思考功能或往返传递思考块（即在后续请求中将返回的思考块发送回去）时最常见的故障。第一部分列出了每个模型所支持的思考配置以及它会拒绝的配置；其后的各部分均从您观察到的症状出发，以便您可以将错误消息或意外响应直接对应到其原因和修复方法。要了解思考功能的工作原理，请参阅[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)概述。

## 各模型的思考支持情况、默认值及被拒绝的配置

大多数思考配置错误源于请求中的 `thinking.type` 值与模型所支持的值不匹配。在大多数模型上，思考以 `thinking: {type: "adaptive"}` 的形式运行，并且许多模型默认开启。一些较早的模型则使用 [extended thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)（扩展思考），这是一种旧版手动模式，配置为 `thinking: {type: "enabled", budget_tokens: N}`。

"Extended thinking"（扩展思考）（`thinking.type: "enabled"` 搭配 `budget_tokens`）在 Claude 4.6 模型上已被弃用（使用它的请求仍会成功）。Claude 4.7 及更高版本的模型不支持它，并会拒绝使用它的请求，返回 400 错误。在支持思考的 Claude 4.5 及更早版本的模型上，扩展思考是唯一可用的思考模式。Claude Mythos Preview 同时支持两种模式。在两种模式均可用的情况下，请改用[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。

下表列出了每个模型支持的内容、默认值，以及哪些 `thinking.type` 值会被以 400 错误拒绝；任何未列为被拒绝的值均会被接受。

| 模型                    | 思考类型         | 默认值  | 以 400 拒绝                  |
| --------------------- | ------------ | ---- | ------------------------- |
| Claude Fable 5.1      | 仅自适应         | 始终开启 | `"enabled"`、`"disabled"`  |
| Claude Mythos 5.1     | 仅自适应         | 始终开启 | `"enabled"`、`"disabled"`  |
| Claude Fable 5        | 仅自适应         | 始终开启 | `"enabled"`、`"disabled"`  |
| Claude Mythos 5       | 仅自适应         | 始终开启 | `"enabled"`、`"disabled"`  |
| Claude Mythos Preview | 自适应、扩展       | 始终开启 | `"disabled"`              |
| Claude Opus 5         | 仅自适应         | 开启   | `"enabled"`、`"disabled"`2 |
| Claude Opus 4.8       | 仅自适应         | 关闭   | `"enabled"`               |
| Claude Opus 4.7       | 仅自适应         | 关闭   | `"enabled"`               |
| Claude Sonnet 5       | 仅自适应         | 开启   | `"enabled"`               |
| Claude Opus 4.6       | 自适应、扩展（已弃用）1 | 关闭   | 无                         |
| Claude Sonnet 4.6     | 自适应、扩展（已弃用）1 | 关闭   | 无                         |
| Claude Opus 4.5       | 仅扩展          | 关闭   | `"adaptive"`              |
| Claude Haiku 4.5      | 仅扩展          | 关闭   | `"adaptive"`              |
| Claude Sonnet 4.5     | 仅扩展          | 关闭   | `"adaptive"`              |

*1 `enabled` 和 `budget_tokens` 在这些模型上仍然有效，但已弃用；请改用自适应思考。*\
*2 Claude Opus 5 在 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 为 `high` 或更低时接受 `"disabled"`；将其与 effort `xhigh` 或 `max` 组合使用会返回 400 错误。此限制适用于 Claude Opus 5 及更高版本的模型，并在每个请求上强制执行。*

标记为"始终开启"的模型无法关闭思考。标记为"开启"的模型默认进行思考，但接受 `thinking: {type: "disabled"}`。

较早的 Claude 4 模型（Claude Opus 4.1、Claude Sonnet 4 和 Claude Opus 4）仅支持扩展思考。有关其可用性，请参阅[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)。除非获得 Anthropic 明确授权，否则 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5 和 Claude Mythos 5 在[零数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)下不可用。

## 400 错误提示不支持 `"thinking.type.enabled"`

请求失败并返回 400 错误，其消息内容为：

```text wrap
"thinking.type.enabled" is not supported for this model. Use "thinking.type.adaptive" and "output_config.effort" to control thinking behavior.
```

出现这种情况是因为您请求的模型已移除扩展思考（请参阅[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#rejected-configurations)）。

将请求切换为 `thinking: {type: "adaptive"}`，并使用 `effort` 而非 `budget_tokens` 来控制思考深度。[迁移到自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#migrating-to-adaptive-thinking)详细介绍了转换过程。

## 400 错误提示不支持 `"thinking.type.disabled"`

请求失败并返回 400 错误，其消息内容为：

```text wrap
"thinking.type.disabled" is not supported for this model. Thinking defaults to adaptive mode when not specified; use "thinking.type.enabled" with "budget_tokens" for extended thinking.
```

这种情况发生在思考始终开启的模型上：Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5 和 Claude Mythos Preview 会拒绝 `"disabled"`。除 Claude Mythos Preview 外，所有这些模型也会拒绝错误文本中建议的 `"thinking.type.enabled"`。

省略 `thinking` 参数即可；这些模型无需任何配置即会进行思考。如果您的目的是让响应中不包含思考文本，请使用 `display: "omitted"` 而不是禁用思考；请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。

针对 `"disabled"` 的 400 错误也可能出现在 Claude Opus 5 上，该模型仅在 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 为 `high` 或更低时接受 `thinking: {type: "disabled"}`：将其与 effort `xhigh` 或 `max` 组合使用会被拒绝。请降低 effort 级别，或保持思考开启。

## 400 错误提示不支持自适应思考

请求失败并返回 400 错误，其消息内容为：

```text wrap
adaptive thinking is not supported on this model
```

出现这种情况是因为该模型仅支持扩展思考（请参阅[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#rejected-configurations)）。

请改用 `thinking: {type: "enabled", budget_tokens: N}`；有关配置，请参阅[扩展思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)。

## 400 错误提示思考块不能被修改

返回工具结果的请求失败，并返回 400 `invalid_request_error`，其消息包含：

```text wrap
`thinking` or `redacted_thinking` blocks in the latest assistant message cannot be modified
```

在多轮对话和工具使用对话中，您会将之前的助手消息（包括其 `thinking` 和 `redacted_thinking` 块）发送回 API，而 API 会验证它们是否未经修改地到达。当您发送回的助手消息与 API 返回的消息不同时，就会发生此错误，最常见的原因是您的代码按类型过滤内容块并丢弃了 `redacted_thinking` 块，或者重新构建了助手消息而不是原样回传。

请将助手轮次原样回传，包括思考块。有关规则，请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)；有关各 SDK 中的正确代码，请参阅[工具和多轮工作流中的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-tool-workflows#two-turn-tool-use-round-trip)中的完整往返示例。

## 400 错误提示思考块签名无效

向 Claude Fable 5.1 发送的、重放早期思考块的请求失败，并返回 400 `invalid_request_error`，其消息内容为：

```text wrap
messages.{i}.content.{j}: Invalid `signature` in `thinking` block. The block is bound to a different conversation. Remove the block, or set `thinking.block_binding.prefix_mismatch_behavior` to "drop_block".
```

如果请求未发送 `thinking-binding-controls-2026-08-01` beta 标头，消息会附加 ``That setting requires the `thinking-binding-controls-2026-08-01` value in the `anthropic-beta` header.``。消息末尾还可能有一句话指出第一条发生变化的消息。如果消息中完全没有原因说明子句，则表示该块的内容已被修改。请参阅 [400 错误提示思考块不能被修改](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#error-thinking-blocks-modified)。

在 Claude Fable 5.1 上，API [仅在 `system` 提示、`tools` 以及其之前的消息均未更改时](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)才接受重放的思考块。该错误意味着对话中较早的某些内容在请求之间发生了变化：某个轮次被编辑、重新排序或删除，某条按轮次注入的提醒后来被移除，`system` 提示或 `tools` 数组被重新构建，或者客户端压缩保留了最近的轮次及其思考内容原文。此检查对 2026 年 8 月 31 日或之后创建的新账户强制执行，也对任何设置了 `thinking.block_binding.prefix_mismatch_behavior` 的请求强制执行。服务器端[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)和[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)永远不会触发它。

要修复此问题，请保持历史记录仅追加：将早期轮次按发送和接收时的原样传回，使用[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)添加指令而不是编辑 `system` 或 `tools`，并让服务器端[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)或[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)来完成任何裁剪。重试相同的请求体不会清除该错误。要在不使用已失效推理的情况下继续此请求，请发送 `thinking-binding-controls-2026-08-01` beta 标头，并将 `thinking.block_binding.prefix_mismatch_behavior` 设置为 `"drop_block"`。或者，从历史记录中剥离所有 `thinking` 和 `redacted_thinking` 块（至少包括被指出的块及其之后的每一个块，涵盖该轮次及所有后续轮次），保留每个轮次的其他块不变，然后重试一次。

来自目标模型无法读取的模型的块永远不会产生此错误：API 会将其丢弃，并在使用 beta 标头时在 `input_transformations` 中报告。

## 响应中的 thinking 字段为空

响应包含 `thinking` 块，但其 `thinking` 字段为空字符串，只有 `signature` 字段有值。

出现这种情况是因为在较新的模型上 `display` 默认为 `"omitted"`，这会返回不含文本的思考块。

在您的思考配置中设置 `display: "summarized"` 以接收摘要后的思考文本。有关各模型的默认值，请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。如果您只想要某些模型在工具调用之间写出的简短状态行，而不需要推理内容，请改为设置 `display: "updates"`（beta）。请参阅[工具调用之间的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)。

## 某些轮次没有出现思考块

即使已配置思考，某些响应也完全不包含 `thinking` 块。

这在自适应模式下是正常的：对于 Claude 判断为足够简单、可以直接回答的请求，它会跳过思考。

如果您希望更频繁或更深入地进行思考，请提高 `effort` 或通过提示进行引导；请参阅[引导 Claude 的思考频率](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#tuning-thinking-behavior)。

## 文本输出中出现工具调用或 XML 标签

响应偶尔会将工具调用写入其文本中，而不是发出 `tool_use` 块，或者在其可见文本中包含 `<thinking>` 或其他内部 XML 标签。泄漏的工具调用永远不会运行，并且在智能体循环中，泄漏的文本会保留在对话历史中，因此后续轮次也会受到影响。

这种情况发生在 Claude Opus 5 禁用思考时，最常见于搜索等工具密集型工作负载。系统提示中指示模型不要思考或不要推理的规则会增加标签泄漏。

请重新启用思考（默认设置），并改用较低的 `effort` 级别来控制令牌成本。如果您的集成必须保持思考禁用，请应用[在禁用思考的情况下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)中的提示缓解措施。

## 响应以 `stop_reason: "max_tokens"` 停止

响应以 `stop_reason: "max_tokens"` 结束，通常伴随被截断或缺失的文本块。

出现这种情况是因为思考令牌计入 `max_tokens`，因此较长的思考过程可能会在文本响应完成之前耗尽预算。

请提高 `max_tokens` 以便为思考和文本都留出空间，或降低 `effort` 以使 Claude 在思考上花费更少；请参阅[成本控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#cost-control)和[思考与上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-the-context-window)。

## 更改思考设置后缓存命中率下降

在之前命中缓存的请求上，`cache_read_input_tokens` 降为零。

出现这种情况是因为思考配置和 effort 级别（或其默认值）是缓存提示前缀的一部分，因此更改其中任何一项都会开始一个新的前缀：切换思考模式、更改 effort 值以及更改 `budget_tokens` 都会使消息缓存断点失效，并且根据模型渲染配置的位置，还可能使工具和系统提示断点失效。

请在共享同一对话的请求之间保持思考配置和 effort 级别不变；将参数显式设置为其默认值等同于省略它，不会导致失效。请参阅[思考与提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-prompt-caching)。

## 设置 effort 不会改变思考

您更改了 `effort`，但思考频率或深度保持不变。

出现这种情况是因为 effort 仅在自适应模式下是主要的思考调节手段。在仅支持扩展思考的模型上，思考深度由 `budget_tokens` 设置。

请在这些模型上调整 `budget_tokens`，或检查您的模型运行在哪种模式下；请参阅[思考与 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-effort)。在 Claude Opus 4.5（唯一支持 effort 的仅扩展思考模型）上，effort 与预算共同作用；请参阅[预算规则与调优](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#budget-rules-and-tuning)。

## 后续步骤

<CardGroup cols={3}>
  <Card title="思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    概述：什么是思考、如何配置它，以及它如何与工具、缓存和流式传输交互。
  </Card>

  <Card title="错误" icon="book" href="https://platform.claude.com/docs/zh-CN/api/errors">
    完整的错误参考，包括思考配置相关的 400 错误及其确切的服务器消息。
  </Card>

  <Card title="迁移到自适应思考" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#migrating-to-adaptive-thinking">
    将 `budget_tokens` 请求转换为使用 effort 的自适应思考。
  </Card>
</CardGroup>
