---
title: 上下文窗口
url: https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows
description: 了解上下文窗口的工作原理、扩展思考和工具使用如何计入上下文窗口，以及如何随着对话增长管理上下文。
---

随着对话的增长，您最终会接近上下文窗口的限制。对于长时间运行的对话和智能体工作流，[服务器端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)是上下文管理的主要策略。

## 上下文窗口的工作原理

"Context window"（上下文窗口）是指语言模型在生成响应时可以参考的所有文本，包括响应本身。这与语言模型训练所用的大型数据语料库不同，而是代表模型的"工作记忆"。较大的上下文窗口允许模型处理更复杂和冗长的提示，但更多的上下文并不自动意味着更好。随着令牌数量的增长，准确性和召回率会下降，这种现象被称为 *context rot*（上下文腐化）。这使得精心管理上下文中的内容与可用空间的大小同样重要。

<Tip>
  有关长上下文为何会退化以及如何通过工程手段应对的更多信息，请参阅 [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)。
</Tip>

下图展示了 API 请求的标准上下文窗口行为1：

![对话轮次在 context window（上下文窗口）中累积直至对话接近令牌限制的示意图](https://platform.claude.com/docs/images/context-window.svg)

*1 诸如 [claude.ai](https://claude.ai/) 之类的聊天界面也可以按滚动的"先进先出"方式管理上下文窗口。*

* **渐进式令牌累积：** 随着对话逐轮推进，每条用户消息和助手响应都会在上下文窗口中累积，之前的轮次会被完整保留。

* **上下文窗口容量：** 上下文窗口（[最多 1M 令牌，取决于模型](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows#context-window-sizes-by-model)）容纳对话历史以及 Claude 生成的新输出。

* **输入-输出流程：** 每一轮包括：

  * **输入阶段：** 包含所有之前的对话历史以及当前的用户消息
  * **输出阶段：** 生成文本响应，该响应将成为下一轮输入的一部分

请求中的所有内容都计入上下文窗口：系统提示、`messages` 中的每条消息（包括工具结果、图像和文档）以及您的工具定义。Claude 为该轮生成的输出（包括其扩展思考）也计入其中。每个响应都会在其 `usage` 字段中报告该请求消耗的内容。如果您使用 [prompt caching（提示缓存）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)，输入计数会拆分为 `input_tokens`、`cache_read_input_tokens` 和 `cache_creation_input_tokens`，这三者都计入窗口。要在发送请求之前进行估算，请使用[令牌计数 API](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)。

## 各模型的上下文窗口大小

Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Opus 5、Claude Opus 4.8、Claude Opus 4.7、Claude Opus 4.6、Claude Sonnet 5、Claude Sonnet 4.6 和 [Claude Mythos Preview](https://anthropic.com/glasswing) 拥有 1M 令牌的上下文窗口。对其中任何一个模型的单个请求最多可生成 128k 输出令牌（`max_tokens`）。其他 Claude 模型（包括 Claude Sonnet 4.5）拥有 200k 令牌的上下文窗口。

对于每个拥有 1M 令牌上下文窗口的模型，1M 是默认值：您不需要 beta 标头，长上下文请求按[标准定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#long-context-pricing)计费。

单个请求最多可包含 600 张图像或 PDF 页面（对于拥有 200k 令牌上下文窗口的模型为 100）。如果您发送大量图像或大型文档，可能会在达到令牌限制之前先达到[请求大小限制](https://platform.claude.com/docs/zh-CN/api/overview#request-size-limits)。

请参阅[模型比较](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)表，了解各模型的上下文窗口大小列表。

## 启用思考时的上下文窗口

使用[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)时，所有输入和输出令牌（包括思考令牌）都计入上下文窗口限制，在多轮情况下有一些细微差别。

思考令牌是您的 `max_tokens` 参数的子集，按输出令牌计费，并计入速率限制。使用 [adaptive thinking（自适应思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)时，Claude 会动态确定其思考分配，因此思考令牌的使用量因请求而异。

之前助手轮次中的思考块是否保留在上下文窗口中取决于模型。在 Claude Opus 4.5 及更高版本的 Opus 模型、Claude Sonnet 4.6 及更高版本的 Sonnet 模型、Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5 和 Claude Mythos Preview 上，API 默认保留之前的思考块，它们像任何其他输入令牌一样计入上下文窗口。在更早的 Opus 和 Sonnet 模型以及所有 Haiku 模型上，当您将之前的思考块传回时，API 会自动将其从对话历史中剥离，从而为对话内容保留令牌容量。有关各模型的默认设置，请参阅[各模型的思考块保留](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-block-preservation-by-model)。要在任一方向上覆盖默认设置，请使用[思考块清除](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#thinking-block-clearing)。

下图展示了在会剥离之前思考块的模型上启用思考时令牌的管理方式：

![在会剥离之前 thinking blocks（思考块）的模型上启用思考的示意图：每轮的思考块在输出中生成，不会带入后续轮次的输入](https://platform.claude.com/docs/images/context-window-thinking.svg)

* **剥离思考块：** 在会剥离之前思考块的模型上，思考块（以深灰色显示）在每轮的输出阶段生成，但不会作为输入令牌带入后续轮次。您无需自行剥离思考块：如果您将它们传回，Claude API 会自动剥离它们。
* **计费：** 思考令牌在生成时作为输出令牌计费一次。在保留之前思考块的模型上，保留的块随后成为后续请求输入的一部分，并像其余对话历史一样按输入令牌计费。

<Note>
  您可以在[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)指南中阅读有关上下文窗口和思考的更多信息。
</Note>

## 启用思考和工具使用时的上下文窗口

下图展示了在会剥离之前思考块的模型上将思考与 tool use（工具使用）结合时令牌的管理方式：

![思考与 tool use（工具使用）结合的示意图：思考块与其工具结果一起保留，然后在会剥离之前思考块的模型上于下一个用户轮次被丢弃](https://platform.claude.com/docs/images/context-window-thinking-tools.svg)

<Steps>
  <Step title="第一轮架构">
    * **输入组件：** 工具配置和用户消息
    * **输出组件：** 思考 + 文本响应 + 工具使用请求
    * **令牌计算：** 所有输入和输出组件都计入上下文窗口，所有输出组件都按输出令牌计费。
  </Step>

  <Step title="工具结果处理（第 2 轮）">
    * **输入组件：** 第一轮中的每个块以及 `tool_result`。您必须将思考块与相应的工具结果一起返回。这是您必须返回思考块的唯一情况。
    * **输出组件：** 在工具结果传回给 Claude 之后，Claude 仅以文本响应（在下一条 `user` 消息之前不会有额外的思考，除非启用了[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)）。
    * **令牌计算：** 所有输入和输出组件都计入上下文窗口，所有输出组件都按输出令牌计费。
  </Step>

  <Step title="新的用户轮次（第 3 轮）">
    * **输入组件：** 所有输入以及上一轮的输出都会被带入。已完成的工具使用周期中的思考块不再需要保留在上下文中：在会剥离之前思考块的模型上，当您将其传回时 API 会自动丢弃它；在保留之前思考块的模型上，它会保留，除非您使用[思考块清除](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#thinking-block-clearing)将其清除。这也是您添加下一个 `user` 轮次的位置。
    * **输出组件：** 由于在工具使用周期之外有一个新的 `user` 轮次，Claude 会生成一个新的思考块并从那里继续。
    * **令牌计算：** 在会剥离之前思考块的模型上，之前的思考令牌不再计入上下文窗口。所有其他之前的块仍计入上下文窗口，当前 `assistant` 轮次中的思考块也是如此。
  </Step>
</Steps>

* **思考与工具使用结合时的注意事项：**

  * 当您提交工具结果时，必须包含伴随该工具请求的完整且未经修改的思考块，包括其签名。
  * API 使用加密签名来验证思考块的真实性。如果您修改了思考块，API 会返回错误。

<Note>
  大多数当前的 Claude 模型支持[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)，这使 Claude 能够在工具调用之间进行思考，包括在收到工具结果之后。在具有自适应思考的模型上它是自动的；Claude Opus 4.5、Claude Sonnet 4.5 和更早的 Claude 4 模型需要 `interleaved-thinking-2025-05-14` beta 标头，而 Claude Haiku 4.5 不支持它。

  有关将工具与思考结合使用的更多信息，请参阅[思考与工具使用](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-with-tool-use)。
</Note>

要减少工具定义本身消耗的上下文，请参阅[管理工具上下文](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/manage-tool-context)，或使用[工具搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)延迟加载工具定义。

## 上下文感知

Claude Sonnet 5、Claude Sonnet 4.6、Claude Sonnet 4.5 和 Claude Haiku 4.5 具有 **context awareness（上下文感知）：** 这些模型在整个对话过程中跟踪其剩余的上下文窗口（即其"令牌预算"）。这使模型能够根据剩余空间管理长时间运行的任务，而不是猜测还剩多少令牌。上下文感知是自动的：您无需启用任何内容，也永远不需要自己发送本节中显示的标签。API 会注入它们。

### 工作原理

在每个请求的系统提示中，API 会告知 Claude 其总上下文窗口：

```xml
<budget:token_budget>200000</budget:token_budget>
```

该预算与您的请求可用的上下文窗口相匹配：Claude Sonnet 5 和 Claude Sonnet 4.6 为 1M 令牌，Claude Sonnet 4.5 和 Claude Haiku 4.5 为 200k 令牌。本节中的示例展示的是拥有 200k 令牌上下文窗口的模型。

每次工具调用后，API 会向 Claude 提供其剩余容量的更新：

```xml
<system_warning>Token usage: 35000/200000; 165000 remaining</system_warning>
```

图像令牌包含在这些预算中。

Claude Opus 4.7 及更高版本的 Opus 模型、Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5 和 Claude Mythos 5 不会收到这些注入的标签。在这些模型上，您可以通过[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)（处于 beta 阶段）为模型提供明确的预算。

<Tip>
  对于跨多个会话的智能体，请设计您的状态工件，以便在新会话开始时能够快速恢复上下文。[记忆工具的多会话模式](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool#multisession-software-development-pattern)介绍了一种具体方法。另请参阅 [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)。
</Tip>

有关使用上下文感知的提示指导，请参阅[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#context-awareness-and-multiwindow-workflows)。

## 使用压缩管理上下文

如果您的对话经常接近上下文窗口限制，请使用[服务器端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)。压缩会在服务器上自动总结对话的早期部分，使对话能够超越上下文窗口限制继续进行。它以 beta 形式适用于 Claude 4.6 及更高版本的模型以及 [Claude Mythos Preview](https://anthropic.com/glasswing)。

对于更专门的需求，[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)提供了额外的策略：

* **工具结果清除：** 在智能体工作流中清除旧的工具结果
* **思考块清除：** 在使用扩展思考时管理思考块

缓存的提示前缀仍然占用上下文窗口：[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)改变的是您为这些令牌支付的费用，而不是它们是否计入。

## 上下文窗口溢出行为

如果仅输入就已超过模型的上下文窗口，API 会在所有模型上返回 400 `invalid_request_error`（"prompt is too long"）。

在 Claude 4.5 及更新的模型上，如果输入令牌加上 `max_tokens` 超过上下文窗口大小，API 会接受该请求。如果生成随后达到上下文窗口限制，它会以 `stop_reason: "model_context_window_exceeded"` 停止。在更早的模型上，API 会改为返回[验证错误](https://platform.claude.com/docs/zh-CN/api/errors)。要在这些模型上选择启用 `model_context_window_exceeded` 行为，请使用 `model-context-window-exceeded-2025-08-26` beta 标头。详情请参阅[停止原因与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons)。

要保持在上下文窗口限制之内，请在向 Claude 发送消息之前使用[令牌计数 API](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting) 估算令牌使用量。

## 后续步骤

<CardGroup cols={2}>
  <Card title="压缩" icon="stack" href="https://platform.claude.com/docs/zh-CN/build-with-claude/compaction">
    服务器端上下文压缩，用于管理接近上下文窗口限制的长对话。
  </Card>

  <Card title="上下文编辑" icon="edit" href="https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing">
    通过上下文编辑在对话上下文增长时自动进行管理。
  </Card>

  <Card title="模型比较表" icon="scales" href="https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison">
    请参阅模型比较表，了解各模型的上下文窗口大小和输入/输出令牌定价列表。
  </Card>

  <Card title="思考" icon="settings" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    为 Claude 提供针对复杂任务的增强推理能力，并控制思考内容的返回方式。
  </Card>
</CardGroup>
