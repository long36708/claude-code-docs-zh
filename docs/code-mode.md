> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 使用 Code Mode 通过代码运行工具调用

> 试用 Code Mode，这是一项实验性功能，Claude 会在其中编写 JavaScript，在一个步骤中进行多次工具调用。

Code Mode 改变了 Claude 使用其[工具](https://code.claude.com/docs/zh-CN/tools-reference.md)的方式。不使用 Code Mode 时，Claude 会直接请求 Bash、Read 或 Edit 等工具调用，并在决定下一步操作之前读取每个结果。使用 Code Mode 时，Claude 会编写一段 JavaScript 来进行这些工具调用，然后由 Claude Code 在您的会话中运行这段代码。

> [!NOTE]
>
> Code Mode 是一项实验性功能，默认处于关闭状态。它可能会在任何版本中发生变化或被移除。如需分享反馈，请运行 `/feedback`。

## Code Mode 的工作原理

在 Code Mode 会话中，您仍然通过提示词描述任务，Claude 会编写 JavaScript 来执行该任务。一段代码可以并行运行命令、在下一次调用中使用上一次的结果，并返回 Claude 需要读取的那部分输出。

例如，如果您让 Claude 提高重试次数上限，Claude 可能会编写以下代码来查找文件、编辑文件并运行测试。最后一行将文件路径和测试输出返回给 Claude：

```javascript theme={null}
const files = await $`rg -l "function retryPolicy" src`.text();
const target = files.trim().split("\n")[0];
await edit(target, "maxAttempts: 3", "maxAttempts: 5");
({ edited: target, tests: $`npm test -- retryPolicy`.nothrow() });
```

在对话中，您看到的是工具调用（例如编辑的 diff），而不是代码。要阅读代码，请按 `Ctrl+O` 打开[会话记录查看器](https://code.claude.com/docs/zh-CN/interactive-mode.md#transcript-viewer)。每段代码在其中显示为一个折叠的 **Code** 行，并附有如何展开的提示。

### 何时使用 Code Mode

对于需要大量相关工具调用的任务，可以尝试使用 Code Mode，例如：

* 需要在许多文件中重复进行的更改
* 需要在 Claude 读取之前先过滤结果的搜索
* 可以同时运行的多个独立命令

### Code Mode 的限制

在启用 Code Mode 之前，请确认您可以接受以下限制：

* **恢复会话**：只能在终端中、并且在 Code Mode 开启的情况下恢复 Code Mode 会话
* **自动模式**：自动模式的分类器[审查的是代码，而不是每个工具调用](#permission-checks)
* **长时间运行的代码**：除非[代码可以在后台运行](#long-running-code)，否则您需要等待代码运行完成，并且代码会在十分钟后停止
* **Hook**：当 `PreToolUse` hook 无法运行或超时时，您可能会看到[工具调用被拒绝](#tool-call-refused-by-a-hook)；`UserPromptSubmit` hook 可能会导致[代码输出被阻止](#code-output-withheld-by-a-hook)
* **不可用的功能**：无法使用 [Remote Control](https://code.claude.com/docs/zh-CN/remote-control.md)、[Claude in Chrome](https://code.claude.com/docs/zh-CN/chrome.md) 或[计算机使用](https://code.claude.com/docs/zh-CN/computer-use.md)
* **移动会话**：无法将会话移到 Desktop 应用，也无法将其转为[后台会话](https://code.claude.com/docs/zh-CN/agent-view.md)，并且 `/fork` 无法启动后台副本

## 要求

* **版本**：Claude Code v2.1.323 或更高版本，任一[发布频道](https://code.claude.com/docs/zh-CN/setup.md#configure-release-channel)均可。旧版本会在不启用 Code Mode 的情况下启动，可能附带也可能不附带原因 `it isn't open to your account yet`
* **账户**：Claude 订阅或 Console 账户。无法通过网关、Amazon Bedrock、Claude Platform on AWS、Google Cloud 的 Agent Platform 或 Microsoft Foundry 使用 Code Mode
* **组织**：您的组织未使用 [HIPAA 配置](https://code.claude.com/docs/zh-CN/hipaa-setup.md)或[零数据保留](https://code.claude.com/docs/zh-CN/zero-data-retention.md)
* **模型**：Claude Opus 5.5、Sonnet 5.5、Haiku 5.5 或 Fable 5.2
* **使用入口**：终端中的交互式 `claude` 会话。Code Mode 不适用于 `-p`、Agent SDK、VS Code 面板、Desktop 应用或云端会话
* **安装**：Claude Code 的[原生构建版本](https://code.claude.com/docs/zh-CN/setup.md)
* **Windows**：已安装 [Git for Windows](https://git-scm.com/downloads/win)

此外，在某些设置开启时也无法启动 Code Mode，例如 API 密钥助手、agent team、OpenTelemetry 导出或 `DISABLE_TELEMETRY`。有关每种情况的解决方法，请参阅 [Code Mode 无法启动](#code-mode-doesnt-start)。

## 试用 Code Mode

启用 Code Mode，给 Claude 一个任务，然后查看 Claude 编写的代码。

**1. 检查您的版本**

运行 `claude --version`。如果版本低于 v2.1.323，请运行 `claude update`。如果您使用的是 `stable` 频道，且更新后未达到 v2.1.323，请等待 `stable` 跟进，或[切换到 `latest`](https://code.claude.com/docs/zh-CN/setup#configure-release-channel)。

**2. 在开启 Code Mode 的情况下启动 Claude Code**

将 [`CLAUDE_CODE_EXPERIMENTAL_CODE_MODE`](https://code.claude.com/docs/zh-CN/env-vars#variables) 环境变量设置为 `1`。如需试用 Code Mode，请在 shell 中设置它。如需为每个新会话启用 Code Mode，请在用户设置中设置它。

> [tab] macOS, Linux, WSL
>
> ```bash theme={null}
> CLAUDE_CODE_EXPERIMENTAL_CODE_MODE=1 claude
> ```

> [tab] Windows PowerShell
>
> ```powershell theme={null}
> $env:CLAUDE_CODE_EXPERIMENTAL_CODE_MODE = "1"
> claude
> ```

> [tab] settings.json
>
> 将此 `env` 块保存到 `~/.claude/settings.json` 中，然后启动 `claude`：
>
> ```json theme={null}
> {
>   "env": {
>     "CLAUDE_CODE_EXPERIMENTAL_CODE_MODE": "1"
>   }
> }
> ```
>
> Claude Code [会忽略项目设置文件中的此变量](https://code.claude.com/docs/zh-CN/settings-reference.md#variables-claude-code-ignores-in-env)，因此您克隆的仓库无法替您启用 Code Mode。

**3. 确认您要使用 Code Mode**

在您的第一个 Code Mode 会话之前，Claude Code 会要求您确认。选择 **Yes, turn on Code Mode**。Claude Code 会重新启动，并在会话顶部附近显示以下这行内容：

```text theme={null}
Code Mode (experimental) is on · this session can be resumed only with Code Mode on
```

如果您看到的是其他内容或没有看到这行内容，请参阅 [Code Mode 无法启动](#code-mode-doesnt-start)。

**4. 给 Claude 一个需要大量工具调用的任务**

在您熟悉的项目中，发送如下提示词：

```text theme={null}
Read every Markdown file in this project and list the links that point to local files that don't exist.
```

**5. 阅读 Claude 编写的代码**

按 `Ctrl+O` 并展开一个 **Code** 行，查看工具调用背后的 JavaScript。再次按 `Ctrl+O` 返回对话。

> [!WARNING]
>
> 如需稍后恢复 Code Mode 会话，请在运行 `claude --continue` 或 `claude --resume` 时保持 `CLAUDE_CODE_EXPERIMENTAL_CODE_MODE` 设置为 `1`。如果 Code Mode 处于关闭状态，Claude Code 会拒绝恢复该会话。在未启用 Code Mode 的情况下启动的会话，即使设置了该变量，也会在不启用 Code Mode 的情况下恢复。

### 关闭 Code Mode

从 shell 和 `~/.claude/settings.json` 中移除 `CLAUDE_CODE_EXPERIMENTAL_CODE_MODE`，然后启动新会话。您也可以保留该变量，并在任一位置将其值改为 `0`。只要其中一处为 `0`，即使另一处设置为 `1`，Code Mode 也会保持关闭。

### 为您的组织禁用 Code Mode

将值为 `0` 的 `CLAUDE_CODE_EXPERIMENTAL_CODE_MODE` 添加到您的[托管设置](https://code.claude.com/docs/zh-CN/managed-settings.md)中，这样您组织中的任何人都无法使用 Code Mode：

```json theme={null}
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_CODE_MODE": "0"
  }
}
```

正处于 Code Mode 会话中的用户会继续留在该会话中，直到他们退出或重新启动 Claude Code。此后，用户将在不启用 Code Mode 的情况下启动，并且在 `0` 生效期间无法恢复其 Code Mode 会话。您无法通过托管设置为用户启用 Code Mode。

## 权限与长时间运行的代码

您的权限模式决定了 Claude 的代码如何被检查，以及代码运行期间您能否与 Claude 交谈。

### 权限检查

Claude 的代码进行的每个工具调用都会经过您的 `deny` 和 `ask` [权限规则](https://code.claude.com/docs/zh-CN/permissions.md)以及您的 [`PreToolUse` hook](https://code.claude.com/docs/zh-CN/hooks#pretooluse)。您无需为 `Bash` 等工具更改规则或匹配器。除了规则和 hook 之外，在自动模式下，分类器会审查代码；在会进行询问的模式下，您需要回复提示：

* **[自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode)**：分类器会在每段代码运行之前对其进行审查，而不是审查每个工具调用。它看不到运行时计算出的值，例如从搜索输出中获取的路径，因此对于必须加以控制的调用，请保留规则或 hook。当[先前的代码仍在运行而新代码开始运行时，您仍可能收到提示](#permission-prompt-with-a-countdown)
* **其他[权限模式](https://code.claude.com/docs/zh-CN/permission-modes.md)**：代码在未经审查的情况下开始运行，对于在该模式下会提示您的每个工具调用，您都会收到提示

如需检查代码本身，请编写一个匹配 `REPL`（运行 Claude 代码的工具）的 `PreToolUse` hook，并从 `tool_input.code` 中读取 JavaScript。

> [!WARNING]
>
> Bash 规则匹配的是命令文本，因此无论是否使用 Code Mode，Claude 都可以[以规则不匹配的形式](https://code.claude.com/docs/zh-CN/permissions.md#bash-rule-limits)运行被拒绝的命令。如需不依赖命令文本的限制，请参阅[沙箱隔离](https://code.claude.com/docs/zh-CN/sandboxing.md)。

### 长时间运行的代码

当代码运行耗时较长的构建或测试时，Claude 要么等待它完成，要么让它在后台继续运行：

* **等待**：Claude 在代码完成后才回复，代码会在十分钟后停止，这包括您在权限提示上花费的时间
* **后台**：您可以继续与 Claude 交谈，代码最多可以运行两个小时

如需在后台运行代码，请以自动模式或 `bypassPermissions` 模式启动会话。如果 Claude 仍然在等待，请检查您当前的模式以及启动会话的方式：

* **您切换到了其他模式**：按 `Shift+Tab`，直到回到您启动时所用的模式
* **您以其他模式启动，或从 `claude --resume` 列表中选择了该会话**：如果您可以[使用自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode)，请退出，然后在 Code Mode 开启的情况下运行 `claude --continue --permission-mode auto`
* **您向 `--tools` 传递了工具名称**：以自动模式重新启动，不使用该标志

## 故障排除

### Code Mode 无法启动

会话顶部附近出现 `Code Mode (experimental) isn't available` 表示 Claude Code 在未启用 Code Mode 的情况下启动。原因显示在同一行中，`/status` 也会显示该原因。在下表中找到对应原因，进行相应更改，然后启动新会话。如果原因未在表中列出，请检查[要求](#requirements)。

| 原因包含 | 解决方法 |
| :- | :- |
| `it isn't open to your account yet` | 如果您的版本低于 v2.1.323，请[更新](#try-code-mode)。否则，请检查网络连接并重新启动。如果该原因仍然存在，请改日再试 |
| `it only works with the Anthropic API` | 移除通过云提供商、网关或自定义 `ANTHROPIC_BASE_URL` 路由请求的设置 |
| `it doesn't work with an API key helper` | 移除 `apiKeyHelper`、`ANTHROPIC_AUTH_TOKEN`，以及 `ANTHROPIC_CUSTOM_HEADERS` 中的任何 `Authorization` 或 `X-Api-Key` 标头。然后登录或设置 `ANTHROPIC_API_KEY` |
| `it only works with Opus 5.5, Sonnet 5.5 and later models` | 使用[受支持的模型](#requirements)启动，例如使用 `--model opus`。Haiku 5.5 和 Fable 5.2 也可以使用 |
| `it doesn't work in background jobs` | 在前台启动，不使用 `--remote-control` |
| `it needs settings that Claude Code downloads from Anthropic` | 取消设置[关闭功能标志获取](https://code.claude.com/docs/zh-CN/env-vars.md#features-that-need-feature-flag-fetching)的变量，例如 `DISABLE_TELEMETRY` |
| `OpenTelemetry is on` | 如果您不需要 [OpenTelemetry 导出](https://code.claude.com/docs/zh-CN/monitoring-usage.md)，请取消设置 `CLAUDE_CODE_ENABLE_TELEMETRY` 和 `ENABLE_BETA_TRACING_DETAILED` |
| `it doesn't yet work with an installed mod` | 关闭检查工具调用的 [mod](https://code.claude.com/docs/zh-CN/plugins/mods/overview.md) |
| `it can't be used together with agent teams yet` | 取消设置 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` |
| `this launch's tool options` | 不使用 `--restricted` 启动，将 `REPL` 添加到 `--tools`，并从 `--disallowedTools` 和您的 `deny` 规则中移除 `REPL` |
| `couldn't save your answer` | 使 `~/.claude.json` 可写 |
| `your organization's settings don't allow it` | 向您的管理员咨询相关[托管设置](#block-code-mode-for-your-organization) |

如果您没有看到任何关于 Code Mode 的行，请运行 `claude --version`。如果您的版本低于 v2.1.323，请进行更新。然后检查 shell 或用户设置中的 `CLAUDE_CODE_EXPERIMENTAL_CODE_MODE` 是否为 `1`，并从这些位置以及任何 `--settings` 文件中移除所有 `0`。

### 带倒计时的权限提示

在自动模式下，如果先前的代码仍在运行时 Claude 又启动了新代码，先前的代码将失去分类器审查。随后，Claude Code 会就先前代码中剩余的、需要审查的调用询问您。在未连接 IDE 的情况下，提示会显示倒计时：

```text theme={null}
Claude Code will automatically block this request in 1:59, to avoid holding up an unattended session
```

请在倒计时结束前作答。否则该调用将被拒绝，并且之后本应显示倒计时的调用将在不提示的情况下被拒绝。如需再次看到这些提示，请向 Claude 发送一条消息。

### 工具调用被 hook 拒绝

如果 Claude 报告工具调用被拒绝，请让它提供错误文本。以下错误表示 `PreToolUse` hook 未给出决定，即使未设置 [`onFailure`](https://code.claude.com/docs/zh-CN/hooks#block-the-action-when-a-hook-fails)，这也会拒绝该调用：

* **`the hook failed`**：对于来自 Claude 代码的调用，您的 hook 无法启动、被终止、以 126、127 或大于 128 的代码退出，或者返回了 HTTP 错误或无效输出。请修复该 hook
* **`the hook did not finish`**：您的 hook 在处理来自 Claude 代码的调用时超时，或者在 Claude 已在会话中运行过代码后，在处理任何其他工具调用时超时。请增大该 hook 的 [`timeout`](https://code.claude.com/docs/zh-CN/hooks#common-fields)

如需在 `command` 或 `http` hook 失败或超时时允许调用继续进行，请在该 hook 上设置 `"onFailure": "continue"`。如果 `command` hook 以 1 等普通代码退出，调用会继续进行，除非您设置了 `"onFailure": "block"`。

### 代码输出被 hook 拦截

如果在您预期看到 Claude 代码输出的地方，Claude 报告了某个 hook 的阻止原因，请检查您的 [`UserPromptSubmit`](https://code.claude.com/docs/zh-CN/hooks#userpromptsubmit) hook。当[代码可以在后台运行](#long-running-code)时，该事件上的 `command`、`http` 和 `mcp_tool` hook 也会在代码输出上运行。如需让输出通过，请修改该 hook，使其在输入中的 `source` 为 `poll_event` 时不进行阻止。

## 相关资源

* [权限模式](https://code.claude.com/docs/zh-CN/permission-modes.md)：将自动模式与会提示您的模式进行比较
* [Hooks 参考](https://code.claude.com/docs/zh-CN/hooks.md)：编写 `PreToolUse` 和 `UserPromptSubmit` hook
* [托管设置](https://code.claude.com/docs/zh-CN/managed-settings.md)：在整个组织中应用统一配置
