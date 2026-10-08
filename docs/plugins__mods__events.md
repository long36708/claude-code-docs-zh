> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 使用 mod 响应事件

> 从 mod 处理 Claude Code 事件：观察、重写或回答工具调用、提示和轮次，过滤 hook 处理的事件，并为其他 mod 做计划。

hook 是一个事件处理程序：Claude Code 在命名事件发生时运行的函数。Claude Code 在即将采取行动的每个点触发事件，例如当它运行工具、提交提示、向模型发送请求或启动或结束会话时。你的 hook 在 Claude Code 采取行动之前运行，因此它可以观察事件、重写事件或代替 Claude Code 回答事件。你使用 [`on(eventName, handler)`](/docs/zh-CN/plugins/mods/reference#the-hook-function) 注册 hook。

在开始之前，请构建你的[第一个 mod](/docs/zh-CN/plugins/mods/create)。对于每个事件及其确切字段，请参阅[参考](/docs/zh-CN/plugins/mods/reference#events)或阅读[你的构建的类型](/docs/zh-CN/plugins/mods/create#get-the-types-for-your-build)。

<h2 id="how-a-hook-handles-an-event">
  hook 如何处理事件
</h2>

hook 位于事件和 Claude Code 对其采取的行动之间，因此它可以观察事件、重写事件或自己回答事件。它接收三个参数：[mods API](/docs/zh-CN/plugins/mods/api) 作为 `$`、事件作为 `e` 和下一个处理程序作为 `next`。事件的处理程序形成中间件链。`next(e)` 调用下一个处理程序，这是另一个 mod 的 hook 或链末端的 Claude Code 自己的行为，它解析为结果。你的 hook 对 `next` 做什么决定了它做以下三件事中的哪一件。

<h3 id="observe-an-event">
  观察事件
</h3>

要观察事件而不改变它，请执行你的工作并返回 `next(e)`。此 hook 记录 Claude 即将使用的每个工具：

```javascript theme={null}
on('tool.call', async ($, e, next) => {
  // 在工具运行之前运行
  $.ui.log('Claude is about to use ' + e.tool)
  // 原样传递事件
  return next(e)
})
```

在每个工具运行之前，转录中会出现一条暗线，例如 `● my-mod: Claude is about to use Bash`，其中 `my-mod` 是你的插件的名称。工具的运行方式与没有 mod 时相同。

要在事件后采取行动，请 `await next(e)`、执行你的工作并返回结果。此 hook 在每个工具运行后记录它：

```javascript theme={null}
on('tool.call', async ($, e, next) => {
  // 让工具运行，并等待其结果
  const result = await next(e)
  // 在工具运行后运行
  $.ui.log(e.tool + ' finished')
  // 原样返回结果
  return result
})
```

该行现在出现在每个工具完成后。Claude 读取相同的结果，因为 hook 返回 `next(e)` 解析的内容。

<h3 id="rewrite-an-event">
  重写事件
</h3>

要更改 Claude Code 作用的内容，例如提示词的文本，请使用修改后的事件副本调用 `next`。事件本身是不可变的：它被深度冻结，对字段赋值会抛出错误。此 hook 在发送前修剪每个提示词：

```javascript theme={null}
on('prompt.submit', async ($, e, next) => {
  // 传递事件的副本，其文本已更改
  return next({ ...e, text: e.text.trim() })
})
```

后续处理程序和 Claude Code 接收修剪后的提示，永远看不到原始提示。你也可以更改结果：`await next(e)`，然后返回替换了字段的结果副本。

<h3 id="answer-an-event">
  回答事件
</h3>

要自己处理事件，请返回结果而不调用 `next`。这会短路链，因此后续 mod 和 Claude Code 自己的行为不会运行。此 hook 拒绝每个 Bash 命令：

```javascript theme={null}
on('tool.call', { tool: 'Bash' }, async () => {
  // 没有调用 next，所以命令永远不会运行
  return { deny: 'Bash is turned off in this project. Use the file tools.' }
})
```

当 Claude 尝试 Bash 命令时，命令不会运行，Claude 将 `deny` 文本读作工具的结果。每个事件都有自己的结果形状，[事件参考](/docs/zh-CN/plugins/mods/reference#events)列出了这些。

<h3 id="filter-which-events-a-hook-handles">
  过滤 hook 处理的事件
</h3>

要仅为某些事件运行 hook，请将过滤器作为第二个参数传递给 `on`。Claude Code 将过滤器称为 matcher。它是一个对象，其字段与事件的字段进行比较，只有当每个字段都匹配时，hook 才会运行。字段可以是值、允许值的数组或正则表达式。

此示例中的每一行都为更窄的工具调用集合注册相同的函数 `hook`：

```javascript theme={null}
// 字符串匹配一个值：仅 Bash 调用
on('tool.call', { tool: 'Bash' }, hook)
// 数组匹配其中任何值：Edit 调用和 Write 调用
on('tool.call', { tool: ['Edit', 'Write'] }, hook)
// 正则表达式按模式匹配：来自一个 MCP 服务器的每个工具
on('tool.call', { tool: /^mcp__github__/ }, hook)
```

`hook` 为 Bash、Edit 或 Write 调用各运行一次，为名称以 `mcp__github__` 开头的工具调用运行一次。对任何其他工具（如 Read）的调用都不匹配这三个中的任何一个，因此 `hook` 不会为它运行。

事件名称可以是通配符。`'classic.*'` 匹配每个[设置 hook 事件](#hook-the-settings-hook-events)。`'*'` 匹配除[遥测事件](/docs/zh-CN/plugins/mods/reference#telemetry)之外的每个事件，遥测事件需使用其自身的名称和 `{ to: 'collector' }` 过滤器。

为每个 matcher 注册一次事件。如果你为 `session.start` 调用 `on` 两次而没有 matcher，模块将无法加载，错误为 `on("session.start") is registered twice without a matcher`。将你的 mod 在会话启动时执行的所有操作放在一个 hook 中。

<h2 id="hook-what-claude-is-doing">
  对 Claude 正在执行的操作设置 hook
</h2>

处理这些事件，可以在工具调用、提示词或轮次发生时查看或更改它们。有关每个事件以及 hook 可以返回的内容，请参阅[事件参考](/docs/zh-CN/plugins/mods/reference#events)。

<h3 id="guard-or-change-a-tool-call">
  拦截或更改工具调用
</h3>

`tool.call` hook 会看到 Claude 即将使用的每个工具，因此可以拒绝调用、更改其参数或放行。`tool.call` 在 Claude Code 即将运行某个工具时触发，包括子代理发起的调用以及对 MCP 工具的调用。`e.tool` 是工具名称，工具的参数是 `e` 的字段，例如 Bash 的 `e.command`。调用 `next(e)` 时，Claude Code 会先运行权限检查，然后运行该工具。

以下 hook 会拒绝执行强制推送的 Bash 命令，并告诉 Claude 原因：

```javascript theme={null}
// The matcher limits the hook to Bash calls, so e.command is the shell command
on('tool.call', { tool: 'Bash' }, async ($, e, next) => {
  if (/git push .*--force/.test(e.command)) {
    // Returning without calling next answers the event, so the command never runs
    return { deny: 'Force pushes are not allowed in this repository. Push to a new branch instead.' }
  }
  // Every other command goes on to the permission check and then to Bash
  return next(e)
})
```

当 Claude 尝试执行 `git push --force` 时，该命令不会运行，也不会出现权限提示，因为该 hook 从未调用 `next`。Claude 会将 `deny` 文本作为工具的结果读取，因此请将其写成 Claude 可以据此采取行动的指令。其他所有 Bash 命令的运行方式与没有该 mod 时相同。

要在工具运行之后执行操作，请 `await next(e)`，完成您的工作，然后返回 `next` 给您的结果。以下 hook 使用 [`$.ui.log`](/docs/zh-CN/plugins/mods/api#show-something-without-starting-a-turn) 记录 Claude 更改的每个 `.mdx` 文件，该方法会在会话记录中添加一行 Claude 不会读取的暗色文本：

```javascript theme={null}
on('tool.call', { tool: ['Edit', 'Write'] }, async ($, e, next) => {
  // Wait for the permission check and the tool, and keep what they produced
  const result = await next(e)
  // A refused call comes back as { deny }, and a failed one has isError set
  const changed = !result.deny && !result.isError
  if (changed && e.file_path.endsWith('.mdx')) $.ui.log('Claude changed ' + e.file_path)
  // Return the result as it came, so Claude reads what the tool returned
  return result
})
```

在 Claude 编辑或写入 `.mdx` 文件后，会话记录中会出现一行暗色文本，注明该文件名。对于其他类型的文件，或者被拒绝或失败的调用，不会记录任何内容。Claude 对该调用的认知不会改变，因为该 hook 返回的是它收到的结果。

要更改调用，请将更改后的参数传给 `next`。要重试调用，请再次调用 `next(e)`：如果 hook 在第一次结果中看到 `isError`，可以再次运行该工具并返回那次的结果。要自行响应调用，请在不调用 `next` 的情况下返回一个带有 `result` 字段的对象，例如 `{ result: 'Skipped by my-mod' }`。这样做时，不会出现权限提示，工具也不会运行，因此您返回的结果就是 Claude 对所发生情况的全部了解。

您组织的[托管设置](/docs/zh-CN/server-managed-settings)中的 hook 会在任何 mod 的 `tool.call` hook 之前运行，并且其中任一 hook 的阻止都是最终决定。

<h4 id="hold-a-tool-call-until-the-user-decides">
  暂缓工具调用，直到用户做出决定
</h4>

hook 可以暂停工具调用，并在其继续之前询问用户如何处理。`tool.call` hook 可以在调用 `next` 或返回之前执行 `await`，在此之前工具调用会一直处于挂起状态。要向用户提出问题，请调用 `$.ui.ask`。它会在 Claude 向您提问时使用的对话框中，将您的问题显示在您的选项编号列表上方，并解析为用户选择的标签。在您的选项之后，对话框会添加一行用于输入其他答案，以及一行 **Chat about this**。

此示例中的 `RISKY` 模式匹配 `rm -r`、`rm -rf`、`git reset --hard` 以及带有 `--force` 的 `git push`，但不匹配其他写法，例如 `git push -f`。此模块会在运行与该模式匹配的 Bash 命令之前进行询问：

```javascript theme={null}
const RISKY = /\brm\s+-rf?\b|\bgit\s+reset\s+--hard\b|\bgit\s+push\b.*--force/

export function register(on) {
  on('tool.call', { tool: 'Bash' }, async ($, e, next) => {
    // Let every other command through without a question
    if (!RISKY.test(e.command)) return next(e)
    // Start from the safe answer, so a question nobody answers refuses the command
    let answer = 'Refuse'
    try {
      // The tool call waits here until the user picks one of the two labels
      answer = await $.ui.ask('Run this command? ' + e.command, ['Run it', 'Refuse'])
    } catch {
      // The user dismissed the question, or this is a claude -p run with nobody to ask
    }
    if (answer !== 'Run it') {
      // Answer without calling next, so the command doesn't run
      return { deny: 'The user declined this command. Ask before trying a different approach.' }
    }
    return next(e)
  })
}
```

当 Claude 尝试执行诸如 `rm -rf build` 之类的命令时，会出现包含该命令的问题，命令会等待回答：

* **用户选择 Run it**：hook 调用 `next(e)`，之后仍会运行常规的权限检查
* **用户选择 Refuse**：命令不会运行，Claude 会读取 `deny` 文本
* **用户输入答案**：`$.ui.ask` 解析为输入的文本。hook 会将其与 `Run it` 比较，因此任何其他文本都会拒绝该命令。
* **无人回答**：当用户关闭问题或选择 **Chat about this** 时，以及在 `claude -p` 运行中，`$.ui.ask` 会 reject，因此 `catch` 块会将答案保留为 `Refuse`

请将等待保持在诸如 `$.ui.ask` 之类的 mods API 调用内部，因为这段时间不计入 hook 的[时间限制](/docs/zh-CN/plugins/mods/reference#limits)。等待您自己的 promise 所花费的时间则会计入。Claude Code 会跳过超时的 hook，因此被暂缓的命令将会运行。

<h4 id="approve-or-refuse-a-tool-call-before-the-user-is-asked">
  在询问用户之前批准或拒绝工具调用
</h4>

要决定某个工具调用是否可以运行，请处理 [`tool.check`](/docs/zh-CN/plugins/mods/reference#tools)，这是 Claude Code 做出该决定的事件。它在权限规则和设置 hook 做出决定之后触发，`next(e)` 会解析为它们的决定：`allow`、`ask` 或 `deny`。您的 hook 返回该决定或另一个决定。`e.input` 保存工具的参数，例如 Bash 的 `command`。

对于固定的命令或路径，请使用诸如 `Bash(npm test)` 之类的[权限规则](/docs/zh-CN/permissions#permission-rule-syntax)，无需编写代码。当决定取决于当下的实际情况（例如当前的 Git 分支或另一个 hook 记录的值）时，请处理 `tool.check`。

以下 hook 会在当前分支为 `main` 时拒绝 `git push`：

```javascript theme={null}
on('tool.check', { tool: 'Bash' }, async ($, e, next) => {
  // What the permission rules and settings hooks decided: 'allow', 'ask', or 'deny'
  const decided = await next(e)
  if (!e.input.command.includes('git push')) return decided
  const branch = await $.process.run(['git', 'branch', '--show-current'])
  if (branch.stdout.trim() !== 'main') return decided
  return { decision: 'deny', reason: 'Push from a branch other than main' }
})
```

在 `main` 上，即使有规则允许 `git push`，该 hook 也会返回 `deny`。在其他分支上以及对于其他命令，调用会得到与没有该 mod 时相同的决定。

该 hook 匹配的是命令文本，因此请将其视为对 Claude 的提醒。要为所有人阻止向 `main` 推送，请在您的 Git 托管平台上保护该分支。

hook 可以返回 `allow`、`ask` 或 `deny`，因此它也可以批准被托管设置之外的 `PreToolUse` hook 阻止的调用。[使用 hook 扩展权限](/docs/zh-CN/permissions#extend-permissions-with-hooks)列出了哪些决定优先于 mod。

<h3 id="rewrite-or-add-to-a-prompt">
  改写或补充提示词
</h3>

`prompt.submit` hook 会在轮次开始之前看到每个提示词，因此可以改写文本或向其中添加内容。`e.text` 是输入的内容。

| 要执行的操作 | 返回的内容 |
| :- | :- |
| 改写提示词。会话记录中的消息会显示新文本。 | `next({ ...e, text: newText })` |
| 在提示词之后添加只有 Claude 读取的文本 | `next({ ...e, context: [...(e.context ?? []), extraText] })` |
| 阻止发送提示词 | `{ drop: 'the reason' }` |

以下 hook 会在提示词提及 Pull Request 时，为 Claude 添加当前分支名称：

```javascript theme={null}
on('prompt.submit', async ($, e, next) => {
  // Pass on a prompt that doesn't mention a pull request as it is
  if (!/\bPR\b|pull request/i.test(e.text)) return next(e)
  const git = await $.process.run(['git', 'branch', '--show-current'])
  // Outside a git repository the command fails, so there's no branch to add
  if (git.exitCode !== 0) return next(e)
  // Keep any context an earlier hook added, and add one more line for Claude
  return next({ ...e, context: [...(e.context ?? []), 'Current branch: ' + git.stdout.trim()] })
})
```

当您发送诸如 `open a PR for this change` 之类的提示词时，您的消息在会话记录中看起来不变，而 Claude 还会在其后读取诸如 `Current branch: feature/auth` 的一行。未提及 Pull Request 的提示词会原样通过，且不会运行 `git`。

[其他事件](/docs/zh-CN/plugins/mods/reference#prompts-and-what-claude-reads)涵盖了 Claude 读取的其余内容：`prompt.section` 用于系统提示词的每个部分，`prompt.context` 用于随第一条消息发送的上下文，`skill.prompt` 用于 skill 的文本。这些 hook 产生的文本如果在请求之间发生变化，会[使提示缓存失效](/docs/zh-CN/prompt-caching)。

<h3 id="follow-a-turn">
  跟踪轮次
</h3>

轮次是 Claude 为回应一个提示词所做的全部工作。处理 `turn.start`、`turn.step` 和 `turn.complete` 来跟踪一个轮次：

| 事件 | 触发时机 | hook 可以执行的操作 |
| :- | :- | :- |
| `turn.start` | 轮次开始 | 观察。`e.turnId` 在另外两个事件中标识该轮次。 |
| `turn.step` | Claude Code 即将向模型发送一个请求。包含工具调用的轮次会有多个请求。子代理的请求会设置 `e.agentId`。 | 读取每个请求的 token 用量、通过 `next({ ...e, model })` 将其发送到不同的模型，或在不调用模型的情况下直接响应 |
| `turn.complete` | 轮次结束，包括用户中断的轮次，此时 `e.isAborted` 为 `true`。`e.answer` 是 Claude 的最终文本，`e.durationMs` 是所用时间，`e.usage` 是该轮次的 token 总量。子代理的轮次触发该事件时会设置 `e.agentId`。 | 观察，或返回一个带有 `text` 字段的对象（例如 `{ text: 'Done in 12 seconds' }`），以在回答下方显示一行 |

请将 `turn.step` hook 编写为异步生成器，因为该事件是流式的。`yield* next(e)` 会在响应流式传输时将其转发，并求值为完成后的结果。以下 hook 会记录 Claude API 从[提示缓存](/docs/zh-CN/prompt-caching)中为每个请求提供了多少内容：

```javascript theme={null}
// function* makes the hook a generator, which can pass the response on piece by piece
on('turn.step', async function* ($, e, next) {
  // Send the request, forward each piece as it arrives, and keep the finished result
  const result = yield* next(e)
  // Skip a result that reports no token counts
  if (result.usage) {
    $.ui.log('cache read ' + result.usage.cache_read_input_tokens + ' · wrote ' + result.usage.cache_creation_input_tokens)
  }
  // Return the result unchanged, so the turn continues as usual
  return result
})
```

Claude 的回复会像没有该 mod 时一样以流式方式显示在屏幕上。每个请求完成后，会话记录中会出现一行暗色文本，给出从缓存读取的 token 数和写入缓存的 token 数。包含工具调用的轮次会有多个请求，因此会添加多行。

`result.usage` 保存 Claude API 为某个请求报告的 token 计数，以及作出回答的 `model`：`input_tokens`、`output_tokens`、`cache_read_input_tokens` 和 `cache_creation_input_tokens`。该 hook 也会针对子代理的请求运行，因此如果您只想处理主对话，请检查 `e.agentId`。

<h3 id="hook-the-settings-hook-events">
  处理设置 hook 事件
</h3>

设置 hook 是您在设置文件中配置的命令、HTTP、提示词和 Agent hook。每个[设置 hook 事件](/docs/zh-CN/hooks#hook-events)（例如 `Stop`、`SessionEnd` 或 `PostToolUse`）同时也是一个名为 `classic.` 后跟该设置 hook 事件名称的事件，例如 `classic.Stop`。`e` 是设置 hook 在 stdin 上接收的 JSON，包括 `transcript_path`。

以下 hook 使用 `Stop`（在 Claude 完成回复时触发）来记录会话的会话记录保存位置：

```javascript theme={null}
on('classic.Stop', async ($, e, next) => {
  // e has the same fields a Stop hook in a settings file reads from stdin
  $.ui.log('Transcript saved at ' + e.transcript_path)
  // Pass the event on, so Stop hooks in your settings files still run
  return next(e)
})
```

每次 Claude 完成回复时，会话记录中会出现一行暗色文本，给出会话记录文件的路径。该 hook 返回 `next(e)`，因此它只观察该事件，不会改变轮次结束的方式。

<h2 id="run-alongside-other-mods">
  与其他 mod 一起运行
</h2>

多个 mod 可以处理同一事件，其中任何一个都可能失败。如果您的 mod 阻止工具调用，请检查它在链中的位置以及当其 hook 失败时会发生什么。

<h3 id="the-order-mods-run-in">
  mod 运行的顺序
</h3>

同一事件上的 hooks 形成一个中间件链。每个 mod 的 `next` 调用以下 mod 的 hook，最后的 `next` 到达 Claude Code 自己的行为。第一个 mod 是最外层的：它在其他 mod 之前看到事件，在它们之后看到结果，并决定其他 mod 是否运行。后续 mod 无法阻止早期 mod 看到事件。

Claude Code 按每个 mod 的来源对链进行排序：

1. 内置保护 `sec-default@builtin`，一个内置于 Claude Code 的 mod，`/plugin` 列为 `cc-plugin-sec-default`，其中[它加载](/docs/zh-CN/plugins/mods/admin#know-what-happens-by-default)，你的组织在 [`prependPlugins`](/docs/zh-CN/plugins/mods/admin#install-your-organizations-mods) 中列出的 mod，然后是任何其他计为你的组织的 mod，不在 `appendPlugins` 中
2. 你安装的 mod
3. 你的组织在 `appendPlugins` 中列出的 mod
4. 其他内置于 Claude Code 的 mod

在你安装的 mod 中，mod 在它在清单中的 `dependencies` 下列出的 mod 之前运行。在一个模块中，hooks 按 `register` 调用 `on` 的顺序运行。

<h4 id="where-settings-hooks-run-in-the-order">
  设置 hooks 在顺序中运行的位置
</h4>

在设置文件中配置的 `PreToolUse` hooks 也在工具调用期间运行，在 mod 链中的固定点：

* **来自托管设置的 `PreToolUse` hooks**：在第一个 mod 的 `tool.call` hook 之前运行，其中任一 hook 的阻止都是最终决定，因此没有 mod 看到调用。
* **来自每个其他设置文件和插件的 `hooks/hooks.json` 的 `PreToolUse` hooks**：在最后一个 mod 调用 `next` 后运行，作为 Claude Code 自己的行为的一部分。回答 `tool.call` 而不调用 `next` 的 mod 会阻止它们运行，调用 `next` 的 mod 在它返回的结果中看到它们的决定。

[`tool.check`](#approve-or-refuse-a-tool-call-before-the-user-is-asked) 在这些 hook 和权限规则做出决定后触发，因此其上的 hook 可以批准第二组中的 hook 所阻止的调用。

<h3 id="handle-a-hook-that-fails">
  处理失败的 hook
</h3>

失败的 hook 不会破坏会话，你可以决定接下来会发生什么。当没有 `.catch` 处理程序的 hook 抛出、超时或返回错误形状的结果时，接下来会发生什么取决于它是否调用了 `next`：

* **它在调用 `next` 之前失败**：Claude Code 跳过它，下一个处理程序代替运行
* **它在 `next` 解析后失败**：该结果成立，没有任何东西运行第二次

一行命名 mod、事件和原因，例如 `my-mod: tool.call hook skipped: threw Error: boom`。你读取它的位置取决于会话，如[找出 mod 为什么不做任何事](/docs/zh-CN/plugins/mods/troubleshoot#find-out-why-a-mod-does-nothing)列出的。其绘图不验证的 `ui.render` hook 的报告方式不同，如[从元素构建树](/docs/zh-CN/plugins/mods/interface#build-a-tree-from-elements)所述。

要使阻止调用的 hook 失败关闭，请添加一个 `.catch` 错误处理程序来代替回答。这里，`guard` 是你的 hook 函数：

```javascript theme={null}
// on 返回一个注册，.catch 将处理程序附加到该 hook
on('tool.call', { tool: 'Bash' }, guard).catch(async ($, e, next) => {
  // next.error.kind 是 'throw' 或 'timeout'，说明 guard 如何失败
  return { deny: 'The command guard failed, so this command was not run: ' + next.error.kind }
})
```

当 `guard` 工作时，处理程序永远不会运行。当 `guard` 在 Bash 调用上抛出或超时时，Claude Code 使用相同的事件调用处理程序。处理程序返回 `{ deny }`，所以命令不会运行，Claude 读取末尾带有 `throw` 或 `timeout` 的文本。没有处理程序，Claude Code 会跳过 `guard` 并运行命令。处理程序自身有更短的[时间限制](/docs/zh-CN/plugins/mods/reference#limits)。

<h2 id="next-steps">
  后续步骤
</h2>

* [使用 mods API](/docs/zh-CN/plugins/mods/api)：添加命令和工具、调用模型并在计时器上运行工作
* [在界面中绘制](/docs/zh-CN/plugins/mods/interface)：在窗格中或提示上方显示你的 hooks 收集的内容
* [测试 mod](/docs/zh-CN/plugins/mods/test)：从测试中触发这些事件中的任何一个
* [Mods 参考](/docs/zh-CN/plugins/mods/reference)：每个事件、每个 mods API 方法和限制
