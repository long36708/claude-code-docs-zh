---
title: 参考
url: https://platform.claude.com/docs/zh-CN/managed-agents/reference
description: Claude Managed Agents 的事件类型、自托管 worker CLI 标志、支持的 MCP 服务器类型、速率限制以及品牌指南。
---

本页汇集了 Claude Managed Agents 的参考资料。如需面向任务的指南，请点击各节中的链接。有关会话资源的操作，请参阅[会话操作](https://platform.claude.com/docs/zh-CN/managed-agents/session-operations)。

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

## 事件类型

持久化的事件类型字符串遵循 `{domain}.{action}` 命名约定；仅限流式传输的 event deltas（事件增量，参见"事件增量"选项卡）是例外。有关发送、流式传输和列出事件的信息，请参阅[会话事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)。Webhook 事件类型在[订阅 webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks#supported-event-types) 中单独列出，其中一些名称与事件流中的名称不同（例如，`session.status_idled` 而非 `session.status_idle`）。

<Tabs>
  <Tab title="用户事件">
    | 类型                        | 描述                                                                                                                                                            |
    | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `user.message`            | 包含文本、图像或文档内容的用户消息。                                                                                                                                            |
    | `user.interrupt`          | 在执行过程中停止智能体。                                                                                                                                                  |
    | `user.custom_tool_result` | 对智能体发起的自定义工具调用的响应。                                                                                                                                            |
    | `user.tool_confirmation`  | 当权限策略要求确认时，批准或拒绝智能体或 MCP 工具调用。                                                                                                                                |
    | `user.define_outcome`     | 定义一个供智能体努力达成的 [outcome（结果）](https://platform.claude.com/docs/zh-CN/managed-agents/define-outcomes)。                                                           |
    | `user.tool_result`        | 仅适用于使用 `self_hosted` [环境](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)的会话，您的集成负责提供 `agent_toolset` 结果。SDK 辅助工具和 CLI 会自动完成此操作。 |
  </Tab>

  <Tab title="智能体事件">
    | 类型                               | 描述                                                                                                                                               |
    | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
    | `agent.message`                  | 智能体响应内容块。                                                                                                                                        |
    | `agent.thinking`                 | 表示智能体正在通过扩展思考取得进展。这仅是一个进度信号，不携带思考内容。                                                                                                             |
    | `agent.tool_use`                 | 智能体调用预构建的智能体工具（bash、文件操作等）。                                                                                                                      |
    | `agent.tool_result`              | 预构建智能体工具执行的结果。                                                                                                                                   |
    | `agent.mcp_tool_use`             | 智能体调用 MCP 服务器工具。                                                                                                                                 |
    | `agent.mcp_tool_result`          | MCP 工具执行的结果。                                                                                                                                     |
    | `agent.custom_tool_use`          | 智能体调用您的某个自定义工具。请使用 `user.custom_tool_result` 事件进行响应。                                                                                             |
    | `agent.thread_context_compacted` | 对话历史已被压缩以适应上下文窗口。                                                                                                                                |
    | `agent.thread_message_received`  | 在[多智能体](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)会话中，来自另一个线程的消息到达了其事件流携带此事件的线程；在主线程上，表示某个智能体向协调者发送了报告或问题。  |
    | `agent.thread_message_sent`      | 在[多智能体](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)会话中，其事件流携带此事件的线程向另一个线程发送了消息；在主线程上，表示协调者向另一个智能体发送了任务或后续消息。 |

    这些事件中的消息内容可能包含 `redacted` 内容块，即 `{"type": "redacted"}`：这是因 Anthropic 模型策略而被隐去的内容的占位符。该块不携带其他字段。Redacted 块仅出现在平台发出的内容中；包含此类块的用户事件将被拒绝并返回 400 错误。
  </Tab>

  <Tab title="会话事件">
    | 类型                                  | 描述                                                                                                                                          |
    | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
    | `session.status_running`            | 智能体正在积极处理中。                                                                                                                                 |
    | `session.status_idle`               | 智能体已完成当前任务，正在等待输入。包含一个 `stop_reason`，指示智能体停止的原因。                                                                                            |
    | `session.status_rescheduled`        | 发生了瞬时错误，会话正在自动重试。                                                                                                                           |
    | `session.status_terminated`         | 会话已结束，原因可能是不可恢复的错误，或是会话已被归档。                                                                                                                |
    | `session.deleted`                   | 会话已被删除。终止所有活动的事件流；此会话不再发出任何事件。                                                                                                              |
    | `session.updated`                   | 会话更新请求更改了至少一个字段。仅包含已更改的字段。更新将在下一轮生效。                                                                                                        |
    | `session.error`                     | 处理过程中发生错误。包含一个带有 `retry_status` 的类型化 `error` 对象。                                                                                            |
    | `session.usage`                     | 会话累计用量和已跟踪标价成本的快照。携带会话的用量总计以及会话[预算](https://platform.claude.com/docs/zh-CN/managed-agents/budgets)的回显，若会话没有预算则为 `null`。                     |
    | `session.thread_created`            | 已创建一个[多智能体](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)线程。                                              |
    | `session.thread_status_running`     | 某个会话线程开始执行。每个会话都会为其主线程发出此事件；在[多智能体](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)会话中，子线程的状态转换也会交叉发布到主事件流。 |
    | `session.thread_status_idle`        | 某个会话线程完成了其轮次，正在等待输入。包含 `stop_reason`。                                                                                                       |
    | `session.thread_status_rescheduled` | 某个会话线程遇到瞬时错误，正在自动重试。                                                                                                                        |
    | `session.thread_status_terminated`  | 某个会话线程已被归档或遇到终止性错误。                                                                                                                         |
  </Tab>

  <Tab title="Span 事件">
    Span 事件是可观测性标记，用于包裹活动以进行计时和用量跟踪。

    | 类型                                | 描述                                                                                                                                                                                                 |
    | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `span.model_request_start`        | 模型推理调用已开始。                                                                                                                                                                                         |
    | `span.model_request_end`          | 模型推理调用已完成。包含带有令牌计数的 `model_usage`。                                                                                                                                                                 |
    | `span.outcome_evaluation_start`   | [Outcome（结果）](https://platform.claude.com/docs/zh-CN/managed-agents/define-outcomes)评估已开始。                                                                                                         |
    | `span.outcome_evaluation_ongoing` | 正在进行的 [outcome（结果）](https://platform.claude.com/docs/zh-CN/managed-agents/define-outcomes)评估期间的心跳。                                                                                                 |
    | `span.outcome_evaluation_end`     | 一个 [outcome（结果）](https://platform.claude.com/docs/zh-CN/managed-agents/define-outcomes)评估周期已完成。`needs_revision` 结果表示随后还有另一个周期；`satisfied`、`max_iterations_reached`、`failed` 和 `interrupted` 为终止状态。 |
  </Tab>

  <Tab title="系统事件">
    | 类型               | 描述                                                                                                                                                                                                           |
    | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
    | `system.message` | 追加特权的系统级上下文，适用于随附的轮次及所有后续轮次。在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Opus 5 和 Claude Opus 4.8 上受支持。在不受支持的主模型上，该事件将被拒绝并返回 `model_does_not_support_mid_conversation_system`。 |
  </Tab>

  <Tab title="事件增量">
    事件增量是仅限流式传输的预览事件。它们在通过 `event_deltas[]` 参数选择启用的流连接（会话级或按线程）上发出，并且永远不会持久化到会话的事件历史中。有关选择启用、累积和协调这些事件的信息，请参阅[事件增量](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#event-deltas)。

    | 类型            | 描述                                                 |
    | ------------- | -------------------------------------------------- |
    | `event_start` | 一个预览事件已开始生成。携带即将到来的事件的 `type` 和 `id`。仅限流式传输，永不持久化。 |
    | `event_delta` | 预览事件的增量内容，由 `event_id` 标识。仅限流式传输，永不持久化。            |
  </Tab>
</Tabs>

## 自托管 worker

以下是用于驱动 `self_hosted` 环境的预构建 worker 的 `ant beta:worker` CLI 标志。有关设置环境、运行 worker 以及 SDK 辅助工具选项的信息，请参阅[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)。

| 标志                     | 描述                                                                                                                   |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `--environment-id`     | 要轮询工作的环境。也可从 `ANTHROPIC_ENVIRONMENT_ID` 读取。                                                                          |
| `--environment-key`    | 使用此环境对 worker 进行身份验证。也可从 `ANTHROPIC_ENVIRONMENT_KEY` 读取。                                                             |
| `--workdir`            | 下载技能以及工具读写文件的目录。默认为 `.`（当前目录）；系统默认工作目录为 `/workspace`。                                                                |
| `--on-work`            | 为每个已领取的工作项调用的脚本，而非在进程内运行工具。以环境变量的形式接收会话详细信息。                                                                         |
| `--unrestricted-paths` | 允许文件工具读写 `--workdir` 之外的路径。workdir 检查仅是针对文件工具的防护措施，而非沙箱；它不约束 bash。                                                   |
| `--max-idle`           | 会话以 `end_turn` [停止原因](https://platform.claude.com/docs/zh-CN/api/handling-stop-reasons)进入空闲状态后，在关闭之前等待的时长。默认为 `60s`。 |
| `--log-format`         | 日志输出格式。使用 `json` 进行结构化日志采集。默认为 `text`。                                                                               |

CLI worker 不会挂载[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory)：附加了记忆存储的会话仍会运行，但智能体在存储的 `mount_path` 处找不到任何内容，且任何更改都不会同步回存储。要在自托管环境上的会话中使用记忆存储，请改为运行 SDK worker；请参阅[使用记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)。

## 支持的 MCP 服务器类型

Claude Managed Agents 可连接到公开 HTTP 端点的[远程 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/remote-mcp-servers)，或通过 [MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview)连接到私有 MCP 服务器。服务器应支持 MCP 协议的 streamable HTTP 传输；仅支持已弃用的 SSE 传输的服务器仍可通过自动回退正常工作。有关在智能体上声明服务器的信息，请参阅 [MCP 连接器](https://platform.claude.com/docs/zh-CN/managed-agents/mcp-connector)。

有关 MCP 及构建 MCP 服务器的更多信息，请参阅 [MCP 文档](https://modelcontextprotocol.io)。

## 速率限制

Managed Agents 端点按组织进行速率限制：

| 操作                  | 限制            |
| ------------------- | ------------- |
| 创建类端点（例如智能体、会话和环境）  | 每分钟 300 个请求   |
| 读取类端点（例如检索、列出和流式传输） | 每分钟 1,200 个请求 |

组织级别的[支出限制和使用层级速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)同样适用。

## 品牌指南

对于集成 Claude Managed Agents 的合作伙伴，使用 Claude 品牌是可选的。在您的产品中提及 Claude 时：

**允许：**

* "Claude Agent"（下拉菜单的首选）
* "Claude"（当位于已标记为"Agents"的菜单中时）
* "\{YourAgentName} Powered by Claude"（如果您已有现有的智能体名称）

**不允许：**

* "Claude Code" 或 "Claude Code Agent"
* "Claude Cowork" 或 "Claude Cowork Agent"
* Claude Code 品牌的 ASCII 艺术或模仿 Claude Code 的视觉元素

您的产品应保持自己的品牌，不应看起来像是 Claude Code、Claude Cowork 或任何其他 Anthropic 产品。如有关于品牌合规的问题，请联系 Anthropic [销售团队](https://www.anthropic.com/contact-sales)。
