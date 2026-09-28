---
title: 从终端连接到 Managed Agents 会话
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/sessions-connect
description: 将 ant CLI 连接到 Claude Managed Agents 会话，以实时跟踪其对话记录、发送消息、允许或拒绝工具调用，或在浏览器中打开会话查看器。
---

`ant beta:sessions connect` 会将您的终端连接到现有的 Claude Managed Agents [会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)。它会加载会话的 transcript（对话记录），并在智能体工作时实时跟踪。您也可以介入：发送消息、中断智能体，或者允许或拒绝正在等待批准的工具调用。使用 `--web` 时，它会改为在浏览器中通过 Claude Console 的会话查看器打开该会话。

该命令需要 1.32.0 或更高版本的 CLI。要安装或更新 CLI 并进行身份验证，请参阅 [CLI 快速入门](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/quickstart)。

## 连接到会话

传入您工作区中某个会话的 ID。您可以从创建响应、`ant beta:sessions list` 或 Console 中复制该 ID。

```bash CLI
ant beta:sessions connect sesn_011CZkZAtmR3yMPDzynEDxu7
```

不使用 `--web` 时，该命令需要交互式终端。在脚本中，请改用 `ant beta:sessions:events stream` 和 `ant beta:sessions:events send`。请参阅 [CLI 脚本编写与自动化](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/scripting)。

按 Ctrl+C 断开连接。会话会继续运行，再次连接时会加载其完整历史记录。

## 跟踪和引导会话

终端视图会实时显示对话：消息和工具调用，以及每次调用的持续时间和结果。状态栏会显示会话是正在运行、空闲，还是正在等待您的批准。在 [multiagent](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)（多智能体）会话中，该视图会跟踪会话的主线程，其中包括协调者与其委派的智能体之间交换的消息。

| 按键                | 操作                                                         |
| ----------------- | ---------------------------------------------------------- |
| Enter             | 将您的输入作为 `user.message` 事件发送。Alt+Enter 或 Ctrl+J 可换行。        |
| Esc               | 在智能体运行时中断它（`user.interrupt`）。                              |
| Ctrl+O            | 显示或隐藏详细信息：工具输入和结果、令牌用量以及状态事件。`--verbose`（`-v`）会在启动时显示详细信息。 |
| Page Up、Page Down | 滚动浏览对话记录。向上滚动会暂停跟踪；按 End 可恢复跟踪。                            |
| Ctrl+C            | 断开连接。在空输入行上按 Ctrl+D 也会断开连接。                                |

当某个工具调用正在等待您的批准时，输入行会变为 **Allow tool call?**。这种情况会在 `always_ask` 策略下发生，或在 `auto` 策略下服务器未能做出判定时发生。选择 **Yes**、**No** 或 **No, and tell the agent why**。CLI 会将您的选择作为 [`user.tool_confirmation`](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies#respond-to-confirmation-requests) 事件发送，您输入的任何理由都会作为其 `deny_message`。

如果会话处于 `terminated` 状态或已被删除，则该视图为只读。

## 在浏览器中打开会话查看器

```bash CLI
ant beta:sessions connect sesn_011CZkZAtmR3yMPDzynEDxu7 --web
```

`--web` 会通过 `127.0.0.1` 上的本地服务器提供 Console 的会话查看器，打印其 URL，并在您的浏览器中打开它。添加 `--no-browser` 可跳过打开浏览器。您也可以在浏览器中发送消息、中断智能体，以及允许或拒绝工具调用。与终端视图不同，浏览器查看器会跟踪多智能体会话的每个线程。

该 URL 只能打开一次，且必须在打印后两分钟内打开。重新加载该标签页是可以的，但如果要在其他任何地方打开查看器，请再次运行该命令。您的凭据永远不会离开 CLI：页面只会向本地 `ant` 进程发送请求，由该进程发出 API 请求。服务器会一直运行，直到您按下 Ctrl+C。
