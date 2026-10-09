> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 使用 mods API

> 从 Claude Code mod 调用 mods API 来添加命令和工具、调用模型、在计时器上运行工作、向其他会话发送消息，以及访问文件和网络。

mods API 是 mod 调用以执行操作的方法集：添加命令和工具、调用模型、在事件之间运行工作，以及访问文件系统、进程和网络。每个 hook 都将其作为第一个参数 `$` 接收，方法按命名空间分组，例如 `$.ui` 和 `$.fs`。[事件](/docs/zh-CN/plugins/mods/events)决定何时运行 hook，mods API 是 hook 运行后调用的内容。

在开始之前，请构建您的[第一个 mod](/docs/zh-CN/plugins/mods/create)。对于每个方法，请参阅 [mods API 方法](/docs/zh-CN/plugins/mods/reference#mods-api-methods)或阅读[您的构建的类型](/docs/zh-CN/plugins/mods/create#get-the-types-for-your-build)。

<h2 id="add-a-command-or-a-tool">
  添加命令或工具
</h2>

mod 可以添加供用户运行的命令和供 Claude 调用的工具。在 [`session.start`](/docs/zh-CN/plugins/mods/reference#session) hook 中注册两者。Claude Code 在第一个提示之前等待该 hook，因此您注册的内容从第一轮开始就可用。

<h3 id="add-a-command">
  添加命令
</h3>

命令是供用户使用的。注册它，然后为其名称处理 [`command.run`](/docs/zh-CN/plugins/mods/reference#commands-and-configuration)。此示例添加了一个 `/standup` 命令，该命令接受可选的天数：

```javascript theme={null}
on('session.start', async ($, e, next) => {
  // 将 /standup 添加到命令列表，并附上用户在那里看到的描述
  await $.command.register({ name: 'standup', description: 'Summarize what changed today', argumentHint: '[days]' })
  return next(e)
})

// 匹配器将 hook 限制为 /standup，因此其他命令不会到达它
on('command.run', { command: 'standup' }, async ($, e) => {
  // e.args 是在命令名称后键入的文本，或空字符串
  return { text: 'Summary for the last ' + (e.args || '1') + ' day(s): ...' }
})
```

会话启动后，`/standup` 及其描述会出现在您键入 `/` 时看到的列表中。`argumentHint` 在您键入命令和空格后显示在提示中，如 `/standup [days]`。当您运行 `/standup 3` 时，第二个 hook 返回 `Summary for the last 3 day(s): ...`，并且成绩单在插件名称后显示该文本。hook 永远不会调用 `next`，因为该命令除了您的行为外没有其他行为。

您返回的 `text` 会打印在成绩单中，Claude 会读取它。要不打印任何内容，如仅打开[窗格](/docs/zh-CN/plugins/mods/interface#pick-where-to-draw)的命令，请返回 `{}`。要让命令在 Claude 工作时运行，请在注册中添加 `immediate: true`。

选择一个没有内置命令使用的名称。在会话中键入 `/` 以查看它们。`$.command.register` 对于已占用的名称会抛出异常，并显示诸如 `"/focus" refused: it is the built-in /focus"` 的消息。抛出异常的 hook 会被跳过，因此您的 `session.start` hook 的其余部分也不会运行。在该 hook 中最后注册命令，或将调用包装在 `try` 和 `catch` 中。

<h3 id="add-a-tool">
  添加工具
</h3>

工具是供 Claude 使用的。使用名称、Claude 读取的描述和其输入的 JSON Schema 注册它。Claude 在由 `mcp__`、您的插件名称、两个下划线和您注册的名称组成的较长名称下看到它。您在 [`tool.call`](/docs/zh-CN/plugins/mods/events#guard-or-change-a-tool-call) hook 中处理其调用，该 hook 被过滤到该完整名称。此示例来自名为 `my-mod` 的插件，注册 `ticket`，因此完整名称是 `mcp__my-mod__ticket`。它为 Claude 提供了一个在问题跟踪器中查找工单的工具：

```javascript theme={null}
on('session.start', async ($, e, next) => {
  await $.tool.register({
    name: 'ticket',
    // Claude 根据此描述决定何时调用该工具
    description: 'Look up a ticket by its id and return its title and status',
    // Claude 必须发送的参数：一个名为 id 的必需字符串
    inputSchema: { type: 'object', properties: { id: { type: 'string' } }, required: ['id'] },
  })
  return next(e)
})

// 完整工具名称是 mcp__、插件名称和注册名称
on('tool.call', { tool: 'mcp__my-mod__ticket' }, async ($, e) => {
  // 工具的参数是 e 的字段，因此 id 是 e.id
  const response = await $.http.fetch('https://tickets.example.com/api/' + encodeURIComponent(e.id))
  // 无论如何都返回结果，以便 Claude 了解查找何时失败
  return { result: response.ok ? response.text : 'Lookup failed with status ' + response.status }
})
```

当您询问工单时，Claude 可以使用其 id 调用 `mcp__my-mod__ticket`。第二个 hook 获取工单并返回响应体，Claude 将其作为工具的结果读取。当服务器以错误状态回答时，Claude 读取 `Lookup failed with status` 和数字。

<Tip>
  当 [MCP 工具搜索](/docs/zh-CN/mcp#scale-with-mcp-tool-search)延迟加载某个已注册的工具时，Claude 能看到其名称，但在搜索该工具之前看不到其描述。如果 Claude 应在每一轮都考虑该工具，请在注册中添加 [`isDeferred: false`](/docs/zh-CN/plugins/mods/reference#tools)，以[预先加载完整工具](/docs/zh-CN/mcp#exempt-a-server-from-deferral)。该字段需要 Claude Code v2.1.293 或更高版本，早期版本会忽略它。
</Tip>

<h2 id="call-a-model">
  调用模型
</h2>

mod 可以向模型发送自己的请求，用于分类或总结文本等小任务。`$.model.complete` 单独发送您的提示词，而 `$.model.fork({ prompt })` 发送当前对话，并将您的提示词附加在末尾。

下表比较了每种请求包含的内容：

| 请求中的内容 | `$.model.complete` | `$.model.fork` |
| :- | :- | :- |
| 模型 | 您传入的 `model` | 会话的模型 |
| 系统提示词 | 一个简短的[归属块](/docs/zh-CN/llm-gateway-protocol#system-prompt-attribution-block)，然后是您传入的 `system`（如果有） | 会话的系统提示词 |
| 消息 | 一条用户消息，即您的 `prompt` | 迄今为止的对话，然后是作为用户消息的您的 `prompt` |
| CLAUDE.md 和其他项目上下文 | 不包含 | 包含，与对话的最后一个请求相同 |
| 工具 | 无 | Claude 的工具，但模型无法调用它们 |

fork 会重复对话的最后一个请求，因此在对话仍处于缓存中时，Claude API 会从[提示缓存](/docs/zh-CN/prompt-caching)中提供其大部分内容。

这两种调用都使用会话的凭据，因此费用计入用户的套餐、API 密钥或云提供商。[您的构建的类型](/docs/zh-CN/plugins/mods/create#get-the-types-for-your-build)记录了每个 `$.model` 方法。

<h3 id="send-one-prompt">
  发送单个提示词
</h3>

向 `$.model.complete` 传入 `model` 和 `prompt`。`prompt` 会成为用户消息。要向模型提供指令（例如角色或输出格式），还需传入 `system`，它会成为系统提示词。

此 hook 通过要求小型模型标记在其后键入的文本来回答 `/triage` 命令（[注册为命令](#add-a-command)）：

```javascript theme={null}
on('command.run', { command: 'triage' }, async ($, e) => {
  const r = await $.model.complete({
    model: 'haiku',
    // 系统提示设置工作，提示携带要标记的文本
    system: 'Reply with one word: bug, feature, or question.',
    prompt: e.args,
    // 一个单词需要很少的令牌，调用在 15 秒后放弃
    maxTokens: 20,
    timeoutMs: 15000,
  })
  // r.text 仅在模型回答时存在，因此首先检查 r.isAnswered
  const label = r.isAnswered ? r.text.trim() : 'unknown'
  return { text: 'Label: ' + label }
})
```

当您运行 `/triage the export button does nothing` 时，mod 将该文本发送到模型并打印其答案，例如 `Label: bug`。当模型不回答时，标签是 `unknown`。

Claude API 失败不会拒绝调用，因此请检查 `r.isAnswered`，并在其为 `false` 时读取 `r.reason`。对于 Claude Code 不会发送的请求，例如您的组织阻止的模型，调用会被拒绝。

[您的构建的类型](/docs/zh-CN/plugins/mods/create#get-the-types-for-your-build)列出了其他选项，例如 `effort`，[限制](/docs/zh-CN/plugins/mods/reference#limits)给出了 `maxTokens` 的默认值。

<h3 id="use-prompt-caching">
  使用提示缓存
</h3>

`$.model.complete` 支持 Claude API 的[提示缓存](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)。API 会缓存请求的开头部分（称为前缀），直到您设置的[缓存断点](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#explicit-cache-breakpoints)为止。当每次调用都以相同的长静态内容（例如指令或参考资料）开头时，请在该内容的末尾设置断点。之后的调用就会从缓存中读取这部分内容，而无需为其支付完整的输入价格。

要设置断点，请将 `prompt` 作为 `{ text }` 块的数组而不是字符串传入，并在静态内容的最后一个块上添加 `cache: true`。Claude Code 会使用 API 的 `cache_control` 字段发送该块。`system` 也接受相同的数组形式。要在两者之间做出选择，请参阅[在 `prompt` 和 `system` 之间选择](#choose-between-prompt-and-system)。

<Note>
  块数组需要 Claude Code v2.1.292 或更高版本。早期版本会拒绝 `prompt` 中的数组，并报出以 `takes { model, prompt } (host check)` 结尾的错误，而 `system` 中的数组则会被排除在请求之外。
</Note>

此版本的 [`/triage` hook](#send-one-prompt) 在要标记的文本之前发送一组较长的标记规则，并在规则之后设置断点。`RULES` 是您自己定义的字符串：

```javascript theme={null}
on('command.run', { command: 'triage' }, async ($, e) => {
  const r = await $.model.complete({
    model: 'haiku',
    prompt: [
      // 每次调用都相同，因此构成缓存的前缀
      { text: RULES, cache: true },
      // 每次调用都会变化，因此放在断点之后
      { text: e.args },
    ],
  })
  return { text: 'Label: ' + (r.isAnswered ? r.text.trim() : 'unknown') }
})
```

TTL 和断点数量有以下限制：

* **TTL**：缓存条目在最后一次使用后持续五分钟。TTL 来自用户的 Claude Code 设置，而不是来自调用。如需一小时，请将 [`subagentPromptCacheTtl`](/docs/zh-CN/prompt-caching#choose-the-ttl-yourself) 设置为 `1h`。
* **每个请求的断点数**：API 接受[最多四个](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#when-to-use-multiple-breakpoints)，再多一个则会在 `r.reason` 中返回 `api-error`

<h4 id="choose-between-prompt-and-system">
  在 `prompt` 和 `system` 之间选择
</h4>

除非您确定请求直接发送到 Claude API，否则请将各调用共享的静态内容放在 `prompt` 的开头：

* **使用 API 密钥或 Claude 订阅直接发送到 Claude API**：两个字段都可以
* **通过 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock)、[Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws)、[Google Cloud's Agent Platform](/docs/zh-CN/google-vertex-ai)、[Microsoft Foundry](/docs/zh-CN/microsoft-foundry) 或 [LLM 网关](/docs/zh-CN/llm-gateway)发送**：使用 `prompt`。Claude Code 会在系统提示词开头添加一个[归属块](/docs/zh-CN/llm-gateway-protocol#system-prompt-attribution-block)，其指纹来自用户消息的开头。`api.anthropic.com` 端点会在缓存之前去除该块。其他端点则会将其作为提示词的一部分接收，因此当 `prompt` 的开头不同时，`system` 中的断点可能无法命中。
* **在其他人运行的 mod 中**：使用 `prompt`，因为您无法选择他们的提供商

在前缀中，`system` 位于 `prompt` 之前，因此 `prompt` 中的断点也会覆盖 `system`，而 `system` 不同的调用将无法命中缓存。

<h4 id="check-for-cache-hits">
  检查缓存命中
</h4>

`$.model.complete` 的结果包含一个 `usage` 对象，其中带有 API 的[缓存字段](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#tracking-cache-performance)。`usage.cache_creation_input_tokens` 统计调用写入缓存的 token 数，`usage.cache_read_input_tokens` 统计调用从缓存读取的 token 数。预期第一次调用会产生写入，而在 TTL 内的后续调用会产生读取。

如果每次调用都在写入而没有任何读取，则说明各调用之间的前缀不同，或者调用间隔超过了 TTL。关于前缀不同的情况，请参阅[在 `prompt` 和 `system` 之间选择](#choose-between-prompt-and-system)。

如果在模型已回答的调用中两个字段都保持为零，则说明没有缓存任何内容。请逐一检查以下原因：

* **前缀太短**：API 不会缓存短于模型[最小长度](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#cache-limitations)的前缀，并且不会返回错误
* **提示缓存已禁用**：当某个 [`DISABLE_PROMPT_CACHING` 变量](/docs/zh-CN/prompt-caching#disable-prompt-caching)适用于该模型时，Claude Code 会移除断点并以不缓存的方式发送文本
* **您的网关去除了 `cache_control`**：网关可能会[移除该字段但仍返回成功](/docs/zh-CN/prompt-caching#where-the-cache-lives)
* **另一个 mod 重写了文本的开头**：此时 Claude Code 会[不带断点发送文本](#what-a-model-complete-hook-receives)

<h3 id="what-a-model-complete-hook-receives">
  `model.complete` hook 接收的内容
</h3>

如果您通过 hook 监听 [`model.complete`](/docs/zh-CN/plugins/mods/reference#mods-api-calls) 事件来检查或更改其他 mod 的请求，请从以下字段读取文本：

* **`e.prompt`**：始终是字符串。当调用方传入数组时，它是各块文本按顺序拼接的结果。
* **`e.system`**：以相同方式构建的字符串，当调用方未传入 `system` 时则不存在
* **`e.promptBlocks` 和 `e.systemBlocks`**：调用方的数组，当调用方为相应字段传入数组时才存在

Claude Code 会发送您的 hook 传给 `next` 的字符串，并使用您随之传入的数组来放置[缓存断点](#use-prompt-caching)。它会保留仍与字符串开头匹配的前导块及其断点，并以不带断点的方式发送字符串的其余部分。例如，`next({ ...e, prompt: e.prompt + NOTE })` 会保留调用方的断点，而更改 `prompt` 开头的 hook 会移除这些断点。

<h2 id="run-work-in-the-background">
  在后台运行工作
</h2>

超越一个事件的工作，例如每分钟检查一次，在您从 `session.start` 启动的计时器上运行。hook 本身为一个事件运行，其自身运行时间有[时间限制](/docs/zh-CN/plugins/mods/reference#limits)。在 `next` 或 mods API 调用上花费的时间不计算，除了 `$.clock.sleep`。`$.clock.every` 和 `$.clock.after` 代替 `setInterval` 和 `setTimeout`，延迟以毫秒为单位：`$.clock.after(5000, fn)` 在五秒后调用 `fn` 一次。每个都返回一个带有 `cancel()` 方法的计时器，`await $.clock.now()` 给出以毫秒为单位的时间。

此 hook 每分钟查找一次拉取请求的检查，并在提示下显示结果。`summarize` 是您自己的函数，将命令的 JSON 输出转换为几个单词：

```javascript theme={null}
on('session.start', async ($, e, next) => {
  // 每 60,000 毫秒调用一次函数，从现在开始一分钟后
  $.clock.every(60_000, async () => {
    const status = await $.process.run(['gh', 'pr', 'checks', '--json', 'state'])
    // 用最新摘要替换提示下的行
    $.ui.status('checks: ' + summarize(status.stdout))
  })
  // 返回而不等待计时器，以便会话立即启动
  return next(e)
})
```

会话照常启动。一分钟后，提示下会出现一行，带有 `⚠`、mod 的名称，然后是 `checks:` 和您的摘要。之后每分钟替换一次。计时器的回调在任何事件之外运行，因此它在轮次之间保持运行，不会启动一个。如果回调抛出异常，错误会进入[调试日志](/docs/zh-CN/plugins/mods/troubleshoot#read-the-debug-log)，计时器在下一个间隔再次运行。

<h3 id="show-something-without-starting-a-turn">
  显示内容而不启动轮次
</h3>

后台工作可以显示用户内容而不启动轮次。这些调用中的每一个都将文本放在不同的位置：

| 调用 | 用户看到的内容 |
| :- | :- |
| `$.ui.status(text)` | 提示下的一行，保持不变直到您更改它。它以 `⚠` 和 mod 的名称开头，如 `⚠ my-mod: checks: 3 passing`。 |
| `$.ui.toast(text)` | 一条带有 mod 名称的 toast 通知，几秒后消失。在[全屏渲染](/docs/zh-CN/fullscreen)中，它是右上角的一个框；在经典渲染器中，它是提示下右侧的一行。 |
| `$.ui.log(text)` | 成绩单中的一条暗线，Claude 不读取。它以 `●` 和 mod 的名称开头，如 `● my-mod: build finished`。 |

<h3 id="start-a-turn-from-a-background-job">
  从后台工作启动轮次
</h3>

当后台工作发现需要 Claude 注意的内容时，它可以通过使用 `$.prompt.submit({ text })` 提交提示来启动轮次。Claude 在命名您的 mod 为发送者的句子后读取文本。要将其作为用户自己的话发送，不带该句子，请添加 `asUser: true`。调用等待直到会话空闲，然后启动新轮次。它在该轮次启动时解析，因此不要在 Claude 工作时运行的处理程序中 `await` 它。

<h3 id="stop-background-work">
  停止后台工作
</h3>

当模块重新加载时，计时器停止。对于 hook 内的长时间运行工作，[`next.signal`](/docs/zh-CN/plugins/mods/reference#the-hook-function) 是一个 `AbortSignal`，当您的 hook 处理的事件被放弃时中止，例如当用户中断时，因此将其传递给任何长时间运行的内容。

<h2 id="send-and-receive-messages-between-sessions">
  在会话之间发送和接收消息
</h2>

mod 可以向您的另一个会话、此会话的子代理之一或其 [agent team](/docs/zh-CN/agent-teams) 中的队友发送纯文本消息，还可以观察到达和离开的消息。

要发送消息，请调用 `$.session.send({ to, text })`，它与 SendMessage 工具进行相同的传递。根据消息的接收者设置 `to`：

* **您的另一个会话**：`{ sessionId }`
* **子代理或队友**：`{ agentId }`，使用来自 `$.agent.list()` 的 id
* **您收到的某条消息的发送者**：该消息来源的字符串地址

调用在消息排队后解析，带有 `{ isDelivered: true }`。当没有传递任何内容时，它使用 `{ isDelivered: false, reason }` 解析，`reason` 说明原因。

此 hook 通过要求您在其后键入的 id 的会话获取状态来回答 `/ping` 命令（[注册为命令](#add-a-command)）：

```javascript theme={null}
on('command.run', { command: 'ping' }, async ($, e) => {
  // e.args 是在 /ping 后键入的会话 id
  const sent = await $.session.send({ to: { sessionId: e.args }, text: 'Status? One line.' })
  // 调用无论如何都解析，因此检查 isDelivered 以了解发生了什么
  if (!sent.isDelivered) $.ui.toast('Not delivered: ' + sent.reason)
  // 空结果在此会话的成绩单中不打印任何内容
  return {}
})
```

当消息排队时，您的会话中不会出现任何内容，其他会话的 Claude 读取 `Status? One line.`。当没有传递任何内容时，toast 通知会给出原因。

`session.receive` 和 `session.send` 让 mod 观察消息。从两者都返回 `next(e)` 以不变地传递每条消息：

| 事件 | 何时触发 | 有用的字段 |
| :- | :- | :- |
| `session.receive` | 消息到达此会话，在 Claude 读取之前 | `e.text` 和 `e.origin.kind`，例如 `peer` 或 `peer-send-message` 用于另一个会话或代理，`task-notification` 或 `scheduled-trigger`。返回 `{ consumed: reason }` 以防止 Claude 读取。 |
| `session.send` | 消息即将离开，来自 SendMessage 工具或 mod | `e.to`、`e.text` 和 `e.origin.kind`，即 `model` 或 `plugin` |

设置为[拒绝入站消息](/docs/zh-CN/cross-session-messaging#control-inbound-messages)的会话在 `session.receive` 触发之前拒绝消息，因此 hook 永远看不到它。为您的批准而保留的消息首先到达 hook，因此 mod 可以读取您尚未批准的消息。hook 的 `next(e)` 在消息未传递时拒绝。

接收消息上的发送者名称是发送者写的任何内容，因此不要基于它做出决定。

<h2 id="reach-files-processes-and-the-network">
  访问文件、进程和网络
</h2>

mod 通过 mods API 访问文件系统、进程和网络，具有与运行 Claude Code 的用户相同的权限。hook 模块本身没有 Node.js API、没有计时器全局变量（如 `setTimeout`），也没有自己的网络或文件访问。标准 JavaScript 和 Web API（如 `URL`、`TextEncoder`、`AbortController` 和 `crypto.subtle`）可用。下面的每个命名空间涵盖一种访问：

| 命名空间 | 它做什么 |
| :- | :- |
| `$.fs` | `read(path)`、`write(path, text)`、`exists(path)`、`stat(path)` 和 `list(path)` 作用于文件和目录 |
| `$.process` | `run(['git', 'status'])` 启动命令并在其退出时解析。`spawn` 流式传输长时间运行命令的输出。 |
| `$.http` | `fetch(url, init)` 通过 `http` 或 `https`。它在读取体后解析为 `{ status, ok, headers, text }`。 |
| `$.store` | 您的插件自己的 JSON 键值存储，在会话之间保留 |
| `$.env` | `get` 和 `set` 环境变量。将名称写为文字字符串。 |
| `$.settings` | `read` 设置文件和托管策略持有的内容 |
| `$.session` | `messages()` 将会话记录作为 `{ role, text, toolUses }` 列表返回。还有工作目录、模型等。[`usage()`](/docs/zh-CN/plugins/mods/reference#mods-api-methods) 返回上下文窗口使用和计划限制。 |
| `$.mcp` | `call` 连接的 MCP 服务器上的工具 |

文件和进程有一些自己的规则：

* **路径**：相对路径相对于会话的工作目录进行解析
* **`$.fs.list`**：将一个目录的条目作为 `{ name, kind, size, isLink }` 返回，不进行递归
* **`$.process.run`**：接受参数列表，不使用 shell。它解析为 `{ exitCode, stdout, stderr }`，无论退出码如何。如果程序无法启动或在超时时仍在运行，它会拒绝，默认为 30 秒，因此将其包装在 `try` 和 `catch` 中。

这些调用中的每一个本身都是一个事件，以其命名空间和方法命名，不带 `$.`，例如 `fs.read` 用于 `$.fs.read`。[链中较早的](/docs/zh-CN/plugins/mods/events#the-order-mods-run-in) mod 可以观察、重写或拒绝您的调用，这是组织限制 mod 到达的方式。

mod 可以在命令已产生输出或已退出之后拒绝您的 `$.process.spawn` 调用，且该命令已执行的任何操作都不会被撤销。此时该调用会拒绝，并返回一条以下列字符串之一加上拒绝方 mod 的原因结尾的消息：

* **`$.process.spawn started, and a plugin withheld its result:`**：拒绝方 mod 尚未将命令的输出读取到末尾。如果命令仍在运行，Claude Code 会将其停止。
* **`$.process.spawn ran, and a plugin withheld its result:`**：拒绝方 mod 已将命令的输出读取到末尾，因此命令已经退出

<h2 id="next-steps">
  后续步骤
</h2>

* [对事件做出反应](/docs/zh-CN/plugins/mods/events)：hook 工具调用、提示和轮次
* [在界面中绘制](/docs/zh-CN/plugins/mods/interface)：在窗格或提示上方显示您的 mod 收集的内容
* [测试 mod](/docs/zh-CN/plugins/mods/test)：在测试中存根这些调用中的任何一个
* [Mods 参考](/docs/zh-CN/plugins/mods/reference)：事件、mods API 方法和限制
