---
title: 保留思考
url: https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking
description: 修改对话现在会导致错误或块被丢弃；如何检查您的集成是否存在这种情况，以及如何迁移。
---

在 Claude Fable 5.1 上，更改对话中先前的轮次（`system` 提示、`tools` 或任何更早的消息）会影响 API 响应。默认情况下，这会使 API 以错误拒绝该请求，除非您选择改为将受影响的 thinking blocks（思考块）从模型可见的内容中丢弃（`prefix_mismatch_behavior: "drop_block"`）。对于在 2026 年 8 月 31 日 00:00 UTC 或之后创建的新账户，该检查默认强制执行。更多详情请参阅\*[工作原理](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#how-it-works)*和*[受影响的对象](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#who-is-affected)\*。

当您将一个块发回时，API 会使用其 `signature` 来检查先前的对话是否未被更改，以及当前模型是否能够读取该块。设置此检查的目的是，使在一组指令下产生的推理无法在另一组（可能具有对抗性的）指令下被重放。

API 提供了一流的替代方案，用于在对话进行过程中对其进行修改，涵盖了大多数转录编辑的用例：用于新指令的[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)、用于每轮提醒的[轮次作用域系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#per-turn-reminders)、用于添加和移除工具的[对话中途工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes)，以及用于按轮次调整思考深度的[每消息 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)。本页其余部分介绍如何判断您的集成是否受到影响，以及如何将常见的 harness 模式迁移到这些功能。作为额外的好处，保持每个思考块之前的所有内容逐字节不变，也能使前缀对 "prompt caching"（提示缓存）保持稳定，参见[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)。

您是否需要采取任何措施，取决于由什么来管理您的对话历史：

* **您使用官方 Claude 产品或 SDK：** Claude Code、claude.ai、[Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 或 [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview)。这些产品会为您保持前缀完整。

* **您直接调用 Messages API**，无论是从您自己的智能体循环还是任何其他环境。您应检查您的代码，确保 `messages` 数组被视为仅追加（append-only）。以下常见模式会编辑前缀，并使编辑点之后的思考失效：

  * 裁剪或丢弃较早的轮次
  * 在客户端对较早的轮次进行摘要并保留最近的轮次
  * 向较早的轮次注入提醒，并在下一次请求时将其移除
  * 每次请求都重建 `system` 提示（当前时间、令牌预算、模式标志）
  * 在会话中途添加或移除 `tools` 中的条目

## 工作原理

对于新请求，API 会检查：

* **模型相同或更新。** 一个块可被产生它的模型以及之后的模型读取，但不能被更早的模型读取。迁移到更新模型的对话会保留其推理。迁移到更旧模型的对话中，这些块无法通过模型检查，API 会在该请求中丢弃它们。确切的按模型列表请参阅[保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-for-model)。
* **块之前的任何内容都未更改。** 顶层 `system` 提示、`tools` 中的工具集合，以及该块之前的每条消息。使用服务器端压缩时，被检查的前缀从最近的[压缩块](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)开始。
* **更早的思考块链未中断。** 更早的 `thinking` 和 `redacted_thinking` 块不属于前缀的一部分，但每个思考块都会跨轮次记录它之前的那个思考块。您可以从历史的开头移除思考块。从中间移除一个思考块会使其后的每个思考块失效。

未通过模型检查的块总是会被丢弃。对于前缀不匹配，您可以通过 `thinking.block_binding.prefix_mismatch_behavior` 选择如何处理，该字段需要 `thinking-binding-controls-2026-08-01` [beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers)：

* `"drop_block"`：API 移除该块以及对话中其后的每个思考块，请求成功。被丢弃的块不计费。响应会在顶层 `input_transformations` 数组中列出它们（流式传输时位于 `message_start` 事件中）。
* `"error"`：API 以 400 `invalid_request_error` 拒绝请求，并指明第一个未通过检查的块。

默认值为 `"error"`。该头允许您设置此字段，并在响应中添加 `input_transformations`。

## 受影响的对象

Claude Fable 5.1。模型列表请参阅[保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking)。

在 Claude Fable 5.1 上，API 对新账户强制执行该检查。新账户是指在 2026 年 8 月 31 日 00:00 UTC 或之后创建的账户。同样的定义适用于 Claude API 和云平台。之后的模型将对所有用户强制执行该检查。

设置了 `prefix_mismatch_behavior` 的请求无论账户创建时间如何都会选择加入强制执行，这也是您从较旧账户进行测试的方式。要检查您的账户是否默认被强制执行，请在不带 beta 头的情况下发送一个编辑了历史的请求：如果返回指明该头的 400 错误，则表示已强制执行。

<Note>
  如果您维护的是一个供他人使用自己的 API 密钥运行的工具或框架，那么您的新账户用户会比您更早遇到该检查：您自己的密钥很可能属于较旧的账户。请在设置了 `prefix_mismatch_behavior` 的情况下进行测试，以便看到他们将会看到的情况。
</Note>

## 如何判断您的集成是否受到影响

捕获您的集成在几个正常轮次中发送的确切请求体，如果您的产品会进行压缩或工具变更，也请包含这些操作。对于每一对连续的请求，比较 `system`、`tools` 以及 `messages` 的共享部分。在新追加的轮次之前，它们应当逐字节相同。

然后对照 API 进行确认。使用 `thinking-binding-controls-2026-08-01` [beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers)和 `claude-fable-5-1`，将 `thinking.block_binding.prefix_mismatch_behavior` 设置为 `"drop_block"`，并通过您的集成运行一个正常的多轮会话。以下请求是此类会话的第二轮，将第一个响应的助手轮次按原样发回：

```bash
curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: thinking-binding-controls-2026-08-01" \
  -d '{
    "model": "claude-fable-5-1",
    "max_tokens": 16000,
    "thinking": {
      "type": "adaptive",
      "block_binding": { "prefix_mismatch_behavior": "drop_block" }
    },
    "system": "You are a coding agent.",
    "messages": [
      { "role": "user", "content": "Fix the failing test." },
      {
        "role": "assistant",
        "content": [
          { "type": "thinking", "thinking": "", "signature": "EqQBCkYIBxgCKkD..." },
          { "type": "text", "text": "I need to see the test first. Which file is it in?" }
        ]
      },
      { "role": "user", "content": "tests/test_auth.py" }
    ]
  }'
```

之后每个响应都会携带一个顶层 `input_transformations` 数组。请在每一轮记录它：

```json
{
  "input_transformations": [
    {
      "type": "thinking_dropped",
      "path": "messages.1.content.0",
      "reason": "prefix_binding_mismatch"
    }
  ]
}
```

* **每一轮都为空：** 您的集成保持了历史完整。
* **`reason: "prefix_binding_mismatch"`：** 位于 `path` 处的块之前的某些内容在本次请求与上一次请求之间发生了变化。对比该轮次之前的 `system`、`tools` 和 `messages` 以找出变化。
* **`reason: "model_binding_mismatch"`：** 对话迁移到了一个无法读取先前模型块的模型（路由器、回退）。这不是您集成中的 bug。请继续发送这些块，让 API 丢弃当前模型无法读取的内容。

这在任何账户上都有效，因为设置该字段会使请求选择加入强制执行。若要在 CI 中显式失败，请设置 `"error"`。400 错误的开头为：

```text wrap
messages.1.content.0: Invalid `signature` in `thinking` block. The block is bound to a different conversation. Remove the block, or set `thinking.block_binding.prefix_mismatch_behavior` to "drop_block".
```

如果请求中没有 beta 头，消息会继续：``That setting requires the `thinking-binding-controls-2026-08-01` value in the `anthropic-beta` header.`` 消息通常以一句指明发生了什么变化的话结尾，例如 `system` 提示或 `tools` 列表与创建该块时不同。

此错误的所有变体请参阅[思考故障排除](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#error-thinking-block-signature)。

## 什么算作编辑

在两个连续请求之间：

| 请求之间的变化                                                                                                              | 之后的思考块                        |
| -------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| 在末尾追加消息                                                                                                              | 有效                            |
| 添加一个带有 `defer_loading: true` 且尚未被任何内容引用的工具                                                                           | 有效                            |
| 从历史开头移除 `thinking` 块（某一点之前的每个思考块）                                                                                    | 有效                            |
| 更改 `system`、`tools` 和 `messages` 之外的任何请求参数（`max_tokens`、`output_config`、`tool_choice`、`metadata` 等）                  | 有效                            |
| 添加、移动或移除 `cache_control` 标记                                                                                          | 有效                            |
| 返回相同字节的轮换签名 URL                                                                                                      | 有效                            |
| 服务器端压缩或上下文编辑移除或替换内容                                                                                                  | 有效（检查比较的是您发送的内容，而不是服务器编辑后的副本） |
| 保留在原位的已清除[轮次作用域系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#per-turn-reminders) | 有效                            |
| 编辑、重新排序或删除任何更早的 `user`、`assistant` 或 `system` 消息                                                                     | 无效                            |
| 向更早的用户轮次添加文本块，或移除您上次添加的文本块                                                                                           | 无效                            |
| 更改顶层 `system` 字符串或块                                                                                                  | 无效                            |
| 在 `tools` 中添加、移除、重命名或编辑工具                                                                                            | 无效                            |
| 从历史中间移除一个 `thinking` 块并保留之后的块                                                                                        | 对之后的每个思考块均无效                  |
| 在下一次请求时返回不同字节的图像或文档 URL                                                                                              | 无效                            |
| 同一条轮次作用域消息在之后的请求中被删除或改写                                                                                              | 无效                            |

## 更新您的集成

每种模式都用一个 API 功能替换一种历史编辑，该功能对模型具有相同的效果，而不会更改更早的字节。

### 按返回原样追加助手轮次

存储每个响应中的 `content` 数组，并将其原样作为助手轮次发回，所有块类型按接收顺序排列，包括 `thinking` 字段为空的 `thinking` 块。不要通过会丢弃未知块类型或空字段的中间类型进行重新序列化。

### 使用对话中途系统消息添加指令，而不是编辑 `system`

如果您的代码每次请求都重建顶层 `system` 提示（当前时间、令牌预算、模式标志、新发现的项目上下文），对话中的每个思考块都会无法通过检查。请在会话开始时冻结 `system`，当有内容发生变化时，在 `messages` 中该变化生效的位置追加一条 [`role: "system"` 消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)：

```json
{
  "role": "system",
  "content": "The user switched the workspace to read-only mode. Do not write files until told otherwise."
}
```

模型会以系统提示的权威性对待它，并且它之前的所有内容都保持不变。在 Claude Fable 5.1 上不需要 beta 头。在工具循环中，请将其放在 `tool_result` 用户消息之后，绝不要放在助手 `tool_use` 与其 `tool_result` 之间（参见[限制](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)）。

### 将每轮提醒作为轮次作用域系统消息发送

最常见的历史编辑是每轮的提示性提醒：在每批工具结果之后追加一行（"将独立的读取请求合并在一起"、"您已经有一段时间没有向用户更新进展了"），并在下一次请求时移除，以免提醒堆积。移除它就是编辑。

取而代之，请在 `tool_result` 用户消息之后，将该提醒作为带有 `clear_at: "next_user_message"` 的[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)发送（beta 头 `mid-conversation-system-clear-at-2026-08-21`）。以下 `messages` 数组是两轮工具调用之后的请求。`messages[3]` 是上一次请求的提醒，保留在原位，`messages[6]` 是本次请求的副本：

```json
[
  { "role": "user", "content": "Fix the failing test." },
  {
    "role": "assistant",
    "content": [
      { "type": "thinking", "thinking": "", "signature": "..." },
      {
        "type": "tool_use",
        "id": "toolu_01",
        "name": "read_file",
        "input": { "path": "tests/test_auth.py" }
      }
    ]
  },
  {
    "role": "user",
    "content": [{ "type": "tool_result", "tool_use_id": "toolu_01", "content": "..." }]
  },
  {
    "role": "system",
    "clear_at": "next_user_message",
    "content": "Request every independent read in one turn."
  },
  {
    "role": "assistant",
    "content": [
      { "type": "thinking", "thinking": "", "signature": "..." },
      {
        "type": "tool_use",
        "id": "toolu_02",
        "name": "read_file",
        "input": { "path": "src/auth.py" }
      }
    ]
  },
  {
    "role": "user",
    "content": [{ "type": "tool_result", "tool_use_id": "toolu_02", "content": "..." }]
  },
  {
    "role": "system",
    "clear_at": "next_user_message",
    "content": "Request every independent read in one turn."
  }
]
```

仅包含 `tool_result` 的用户消息算作"下一条用户消息"，因此 `messages[3]` 已被清除：它不渲染任何内容，也不消耗输入令牌，但它仍在数组中，因此 `messages[4]` 中的思考保持有效。`messages[6]` 是模型本轮看到的内容。在之后的请求中，将两者保留在原位，并在下一条 `tool_result` 消息之后追加下一个副本。轮次作用域消息仅携带 `text`，不接受 `cache_control`。请将缓存断点放在前一个用户轮次上。参见[轮次作用域系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)。

如果不使用该 beta，请将提醒作为 `text` 块追加在同一用户消息中的 `tool_result` 块之后，并将更早的副本保留在原位。模型会依据最新的那一条行动。

### 使用 `tool_addition` 和 `tool_removal` 更改工具，而不是编辑 `tools`

如果工具集合在会话中途发生变化（某个工具在身份验证后解锁，某个危险工具在模式切换后被撤回），不要编辑 `tools`。请在会话开始时声明完整集合，并使用[对话中途工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes)从该点起提供或撤回某个工具（beta 头 `mid-conversation-tool-changes-2026-07-01`）。尚不可用的工具设置 `defer_loading: true`，并在之后使用 `tool_addition` 块，其形状与此 `tool_removal` 相同：

```json
{
  "role": "system",
  "content": [
    { "type": "tool_removal", "tool": { "type": "tool_reference", "name": "delete_branch" } },
    { "type": "text", "text": "Branch deletion is disabled for the rest of this session." }
  ]
}
```

对于您在会话中途才得知其 schema 的工具（运行时发现的 MCP 服务器），可以将其以 `defer_loading: true` 追加到 `tools` 中，并通过 `tool_addition` 提供。未被引用的延迟加载工具不属于前缀的一部分，因此追加它是安全的。追加常规工具则不安全。

### 尽可能在服务器端裁剪上下文

客户端截断和摘要是第二常见的编辑：丢弃或摘要最早的轮次，并逐字保留最近的轮次。最近轮次的思考块是在您移除的历史仍然存在时产生的，因此它们无法通过检查。服务器端的等效操作不算作编辑，因为检查比较的是您发送的对话：

* [压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)会在上下文接近您设置的阈值时将较早的轮次摘要为一个压缩块，被检查的前缀从该块重新开始。其 [`instructions` 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction#custom-summarization-instructions)接受您自己的摘要提示（"保留每个股票代码、持仓规模和已声明的假设"）。
* [上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)按规则清除旧的工具结果（`clear_tool_uses_20250919`）或按从旧到新的顺序清除旧的思考块（`clear_thinking_20251015`）。

### 客户端自定义压缩

此检查并不禁止客户端压缩。规则更为狭窄：**不要在您已重写的前缀之后保留思考块。**

**简单压缩**是推荐的形式，无需任何更改。当对话变得过长时，将其摘要为一条消息，并以该摘要加上新的用户轮次开始下一次请求，不重放任何更早的轮次或思考块：`messages` 变为 `[{"role": "user", "content": "<summary of the session so far>\n\n<the next instruction>"}]`。没有更早的思考残留，因此不会有任何失败，模型会在压缩后的对话上重新思考。Claude 模型在长周期任务上使用此方案进行训练，对于大多数工作负载，其表现与更复杂的方案相当。与任何压缩一样，它会在压缩点重置提示缓存。

另外两种常见形式按原样编写会失败，各需一处更改：

* **保留尾部压缩**对较早的轮次进行摘要，并逐字保留最近的轮次。被保留轮次的思考块是针对完整历史产生的，因此它们在摘要之后会失败。修复方法：从您携带过来的每个助手轮次中剥离 `thinking` 和 `redacted_thinking`，保留 `text` 和 `tool_use`，或者发送 `prefix_mismatch_behavior: "drop_block"` 让 API 剥离它们。
* **后台压缩**在关键路径之外构建摘要，并在对话继续进行时将其换入，因此在此期间产生的每个轮次的思考都早于该替换。修复方法：在每个仍携带替换前产生的思考块的请求上发送 `"drop_block"`（或自行剥离这些块；替换后第一个响应中的 `input_transformations` 会准确列出是哪些块），或者同步进行压缩。

从转录中间剪除个别轮次会使其后的所有内容失效，没有任何客户端形式可以避免这一点。对于您原本要进行的指令更改，请使用对话中途系统消息；对于选择性移除，请使用服务器端[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)。

不要在工具轮次中间进行压缩：`tool_use` 仍在等待 `tool_result` 的助手轮次应当连同其完整的思考一起发回，以便模型带着其推理完成该轮次（参见[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)）。

### 通过 ID 引用文件，而不是通过内容会变化的 URL

对于带有 `url` 源的 `image` 或 `document` 块，获取到的字节属于被检查前缀的一部分，而 URL 字符串不属于。"最新截图"端点或被编辑过的文档会使之后的思考失效。同一文件的轮换签名 URL 则不会。对于您跨轮次引用的内容，请使用 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传一次并使用 `file_id`，或发送 base64。

### 决定不匹配时的处理方式

一旦您的集成变为仅追加，请为生产环境选择一个 `prefix_mismatch_behavior`。它仅管辖前缀不匹配。当前模型无法读取的块（在路由器切换或[服务器端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)之后）总是会被丢弃，并在发送了 beta 头时在 `input_transformations` 中报告。

* **`"error"`**（默认值），如果前缀不匹配只可能意味着您代码中的 bug。您会在测试中通过 400 错误发现问题，而不是通过被静默丢弃的块。在 Message Batches API 中，未设置时的默认行为是丢弃失败的块而不是使批次项失败；如果您希望项报错，请显式设置 `"error"`。
* **`"drop_block"`**，如果您宁愿丢弃受影响的块而不是失败。请记录 `input_transformations`。

如果您在生产环境中捕获到该 400 错误，重试相同的请求不会清除它。请使用 `prefix_mismatch_behavior: "drop_block"`（以及 beta 头）重试，这会精确移除失败的块，包括 `tool_use` 仍在等待其 `tool_result` 的助手轮次中的任何块。丢弃仅适用于该请求，因此请在会话的剩余部分继续发送 `"drop_block"`（以及 beta 头）。如果不使用该 beta，请从历史中剥离每个 `thinking` 和 `redacted_thinking` 块，保留每个轮次的 `text` 和 `tool_use` 块，然后重试一次。之后修复导致该问题的编辑。

## 本页使用的 API 功能

| 功能                                                                                                                                                                                    | 替代的内容                                           | 状态   | 头                                             |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | ---- | --------------------------------------------- |
| [针对未保留块的控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking-controls)（`thinking.block_binding.prefix_mismatch_behavior`、`input_transformations`） | 在前缀不匹配时选择拒绝或丢弃，并查看丢弃了什么                         | Beta | `thinking-binding-controls-2026-08-01`        |
| [对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)（`messages` 中的 `role: "system"`）                                                 | 重建顶层 `system` 提示                                | 稳定   | 无                                             |
| [轮次作用域系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#per-turn-reminders)（`clear_at: "next_user_message"`）                                          | 注入提醒并在下一次请求时删除                                  | Beta | `mid-conversation-system-clear-at-2026-08-21` |
| [对话中途工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#mid-conversation-tool-changes)（`tool_addition`、`tool_removal`）                   | 编辑 `tools` 数组                                   | Beta | `mid-conversation-tool-changes-2026-07-01`    |
| [压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)（用于自定义摘要提示的 `instructions`）                                                                                  | 客户端对旧轮次的摘要                                      | Beta | `compact-2026-01-12`                          |
| [上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)（`clear_tool_uses_20250919`、`clear_thinking_20251015`）                                               | 客户端删除旧的工具结果或思考                                  | Beta | `context-management-2025-06-27`               |
| [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)（`file_id` 源）                                                                                              | 内容在请求之间变化的 URL                                  | 稳定   | 无                                             |
| [每消息 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)（`role: "system"` 消息上的 `output_config.effort`）                       | 在请求之间更改顶层 effort（保护的是提示缓存而非思考：effort 不属于前缀的一部分） | Beta | `mid-conversation-output-config-2026-07-01`   |

要在一个请求中组合多个头：

```text wrap
anthropic-beta: thinking-binding-controls-2026-08-01,mid-conversation-system-clear-at-2026-08-21,mid-conversation-tool-changes-2026-07-01
```

相同的 beta 名称适用于 Amazon Bedrock 和 Google Cloud。如何通过各 SDK 发送它们，请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers)。

## 检查清单

* 如果由官方 Claude 产品或 SDK（Claude Code、claude.ai、Claude Managed Agents、Claude Agent SDK）管理您的对话历史，到此为止即可。
* 连续的请求体在 `system`、`tools` 和共享的 `messages` 前缀上逐字节相同。
* 在 `prefix_mismatch_behavior: "drop_block"` 下运行的完整会话未记录任何 `prefix_binding_mismatch` 条目。
* 助手轮次按返回原样逐字节发回，包含所有块类型。
* 顶层 `system` 和 `tools` 在会话期间固定不变。变更放在 `role: "system"` 消息以及 `tool_addition` / `tool_removal` 块中。
* 每轮提醒是轮次作用域系统消息（或尾随文本块），每次全新追加且从不移除。
* 上下文通过压缩或上下文编辑进行裁剪，或通过不在重写前缀之后留下任何思考块且从不拆分工具轮次的客户端压缩进行裁剪。
* 跨轮次文件使用 `file_id` 或 base64，而不是可变 URL。
* 已设置生产环境的 `prefix_mismatch_behavior`，并对其 400 错误或丢弃条目进行监控。

## 后续步骤

<CardGroup cols={2}>
  <Card title="思考故障排除" icon="hammer" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting">
    诊断并修复最常见的思考故障：配置 400 错误、空的或缺失的思考块、max\_tokens 停止以及缓存未命中。
  </Card>

  <Card title="对话中途系统消息和工具变更" icon="messages" href="https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages">
    在对话进行到一半时更改系统指令或工具可用性，而不会使其之前的已缓存前缀失效。
  </Card>

  <Card title="压缩" icon="stack" href="https://platform.claude.com/docs/zh-CN/build-with-claude/compaction">
    服务器端上下文压缩，用于管理接近上下文窗口限制的长对话。
  </Card>

  <Card title="提示缓存" icon="database" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching">
    使用 `cache_control` 缓存提示前缀以降低成本和延迟，可使用自动缓存或带有 5 分钟或 1 小时 TTL 的显式断点。
  </Card>
</CardGroup>
