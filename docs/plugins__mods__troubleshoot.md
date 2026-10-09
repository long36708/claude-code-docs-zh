> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 排查 mod 问题

> 了解为什么 Claude Code mod 不起作用：将症状或消息与其原因匹配，查找拒绝消息，并阅读调试日志。

当 mod 的模块或其中一个 hook 失败时，Claude Code 会跳过它，会话继续进行，因此损坏的 mod 看起来像什么都不做的 mod。首先检查 Claude Code 从 mod 读取了什么以及它在哪里报告问题，然后找到您遇到的症状或消息。

<h2 id="find-out-why-a-mod-does-nothing">
  找出为什么 mod 不起作用
</h2>

当 mod 不起作用时，请检查 Claude Code 从 mod 文件读取的内容，以及它在跳过某些内容时写入的行。对于第一项，在您的 shell 中运行 [`claude plugin validate`](/docs/zh-CN/plugins/mods/create#check-what-claude-code-reads-from-your-mod)，使用 mod 的目录，如 `claude plugin validate ./first-mod`。它可以捕获拼写错误的事件、错误的清单和 Claude Code 无法读取的模块，而无需启动会话。

当模块未加载、hook 被跳过或另一个 mod 拒绝您的 mod 时，Claude Code 会写入一行，其中命名您的 mod。您读取该行的位置取决于会话：

* **热重新加载插件目录的会话**：成绩单中的暗行。这是您使用 `--plugin-dir` 启动的交互式会话，或者是您为 Claude 编写的 mod [启用热重新加载](/docs/zh-CN/plugins/mods/create#ask-claude-for-a-mod) 的会话。
* **任何其他交互式会话，例如运行您从市场安装的 mod 的会话**：[调试日志](#read-the-debug-log) 仅。要获取一个，请使用 `claude --debug` 启动会话。
* **带有 `--plugin-dir` 的 `claude -p` 运行**：stderr，采用默认文本输出格式。另一个 mod 的拒绝仅进入调试日志。

<h2 id="check-whether-mods-can-load">
  检查 mod 是否可以加载
</h2>

要检查您的设置是否允许 mod 加载，而无需安装一个，请在您的 shell 中从不包含 mod 的目录运行 `claude plugin test`。您不需要会话。它打印的消息告诉您状态：

| 消息包含 | 这意味着什么 |
| :- | :- |
| `no hooks module to load` | Mod 可以加载。该命令在此目录中找不到要测试的 mod。 |
| `hooks modules are turned off here` | 一个设置正在阻止您的 mod：您自己的设置中的 `disableAllHooks`，或您的组织的策略 |
| `hooks modules are turned off in this process: the rollout switch served off` | Anthropic 已远程关闭已安装的 mod。 |
| `hooks modules are turned off in this process: the rollout switch was saved off by an earlier session` | 该命令使用了之前某个会话保存的值，该值可能已过时。请启动一次 `claude` 以刷新该值，然后再次运行该命令。 |

组织还可以设置 `allowManagedModsOnly` 以仅允许其自己的 mod，此命令不会报告。在这种情况下，Claude Code 会拒绝您安装的 mod，并且[会有消息说明原因](/docs/zh-CN/plugins/mods/troubleshoot#messages-from-the-built-in-guard)。

<h2 id="the-mod-doesn’t-load">
  mod 不加载
</h2>

mod 添加的任何内容都不会出现：没有命令、没有绘图，也没有行为改变。

<h3 id="your-version-is-too-old">
  您的版本过旧
</h3>

请参阅[应使用哪个版本以及如何检查您的版本](/docs/zh-CN/plugins/mods/overview#turn-mods-on-or-off)。

<h3 id="the-mods-active-line-doesn’t-name-the-mod">
  `mods active` 行不命名 mod
</h3>

mod 添加的任何内容都不会出现，`/plugin` 中的 [`mods active` 行](/docs/zh-CN/plugins/mods/overview#see-which-mods-a-session-loaded) 不命名它。hooks 模块未加载。当 Claude Code 拒绝它时，调试日志有一行以 `hooks module`、mod 的名称和 `not loaded:` 开头，如 `hooks module first-mod@inline not loaded: disableAllHooks in managed settings`，用于使用 `--plugin-dir` 加载的 mod。

读取冒号后的原因。[拒绝消息](#refusal-messages) 部分列出了每一个。如果日志中没有这样的行，请逐一处理此组中的其他条目。

某些设置会阻止 mod，同时让其插件的其余部分继续工作。[启用或关闭 mod](/docs/zh-CN/plugins/mods/overview#turn-mods-on-or-off) 列出了这些设置。

<h3 id="a-claude-p-run-prints-hooks-module-not-loaded">
  `claude -p` 运行打印 `hooks module not loaded`
</h3>

该行以 mod 的名称开头并进入 stderr。hooks 模块被拒绝。非交互式运行没有成绩单，因此消息进入 stderr。

读取冒号后的原因。[拒绝消息](#refusal-messages) 部分列出了每一个。

<h3 id="refusal-messages">
  拒绝消息
</h3>

这些消息中的每一个都遵循调试日志中的 `hooks module`、mod 的名称和 `not loaded:`。

| 消息开头 | 这意味着什么 |
| :- | :- |
| `hooks modules are turned off for installed plugins in this process: the rollout switch served off` | Anthropic 已远程关闭已安装的 mod。 |
| `hooks modules are turned off for installed plugins in this process: the rollout switch was saved off by an earlier session` | 该会话使用了先前会话保存的值，该值可能已过时。请重新启动 Claude Code 以刷新该值。 |
| `disableAllHooks in managed settings` | 您的组织关闭了来自已安装插件的 hooks |
| `only managed plugins and built-in plugins run` | 设置了 `allowManagedHooksOnly`，或在托管设置以外的设置文件中设置了 `disableAllHooks` |
| `installed plugins that are not managed load no hooks module in this mode (--bare)` | 您使用 `--bare` 启动了 Claude Code |
| `another plugin of that name loads first` | 两个插件共享一个名称。使用托管的或首先加载的。 |

<h3 id="messages-from-the-built-in-guard">
  来自内置保护的消息
</h3>

在具有托管设置的机器上，或对于使用 Team 或 Enterprise 计划登录的用户，[内置保护](/docs/zh-CN/plugins/mods/admin#know-what-happens-by-default) 可以拒绝 mod 或其答案之一。每条消息都命名您的组织管理员设置以更改规则的选项。

| 消息包含 | 这意味着什么 | 它出现在哪里 |
| :- | :- | :- |
| `mods are limited to your organization's by policy (allowManagedModsOnly)` | 您的组织仅允许 [其自己的 mod](/docs/zh-CN/plugins/mods/admin#install-your-organizations-mods)，因此您的 mod 被拒绝 | 调试日志，以及 [热重新加载插件目录的会话](#find-out-why-a-mod-does-nothing) 中的会话记录 |
| `tried to lift a deny rule in your settings` | 您的 mod 的 [`tool.check`](/docs/zh-CN/plugins/mods/reference#tools) hook 批准了 `deny` 规则拒绝的调用。该调用保持被拒绝。 | 成绩单和调试日志，会话中每个 mod 一次。在 `claude -p` 运行中，仅调试日志。 |
| `the deny rules in your settings could not be checked for this call, so it is refused` | 保护在检查 mod 批准的调用时失败，因此它拒绝了该调用 | Claude 为被拒绝的调用读取的原因 |

<h3 id="validate-passes-and-lists-no-hooks-line">
  `validate` 通过且不列出 `hooks` 行
</h3>

`hooks/hooks.json` 没有 `modules` 键，或键拼写错误。

添加 `"modules": ["./register.js"]`。

<h3 id="hooks-module-did-not-load">
  `hooks module did not load`
</h3>

该行以 mod 的名称开头，然后是 `hooks module did not load:` 和一个原因，当问题在您的代码中时，它给出文件和行。Claude Code 无法加载模块，例如因为其顶级代码抛出了异常。

修复原因命名的错误。

<h3 id="options-do-not-fit-plugin-json-userconfig">
  `options do not fit plugin.json userConfig`
</h3>

该行以 mod 的名称开头，然后是 `hooks module did not load: options do not fit plugin.json userConfig:` 和一个原因。某个选项未通过其 [`userConfig`](/docs/zh-CN/plugins/components#user-configuration) 字段的验证，例如高于字段 `max` 的数字，或必需字段没有值。

设置或更改值。该行的末尾命名其在 `settings.json` 中的 `pluginConfigs` 条目。

<h3 id="no-mod-loads-in-a-directory-you-opened-for-the-first-time">
  没有 mod 在您首次打开的目录中加载
</h3>

您还没有回答该目录的信任提示。

使用 `claude` 在该目录中启动交互式会话，并接受它打开的信任提示。

<h3 id="no-installed-plugin-loads-at-all">
  没有已安装的插件加载
</h3>

您使用 `--safe-mode` 启动了 Claude Code。

启动时不使用该标志。

<h2 id="a-hook-is-skipped-or-a-mod-is-unloaded">
  hook 被跳过或 mod 被卸载
</h2>

mod 已加载，然后 Claude Code 跳过了其中一个 hooks 或卸载了它。

<h3 id="hook-skipped">
  `hook skipped`
</h3>

该行命名 mod 和事件，然后说 `hook skipped:` 和一个原因，如 `first-mod: tool.call hook skipped: threw Error: boom`。hook 抛出了异常、运行超过了其[时间限制](/docs/zh-CN/plugins/mods/reference#limits)，或返回了错误形状的结果。该行对每个事件和失败类型出现一次，直到 mod 重新加载。

修复错误。调试日志对每次出现都有一行。

<h3 id="no-command-run-hook-answered-it">
  `no command.run hook answered it`
</h3>

您运行了 mod 添加的命令，回复命名了 mod 和命令，如 `first-mod registered /tally but no command.run hook answered it`，然后告诉您添加一个 hook。当命令到达链的末尾而没有答案时，Claude Code 会打印该回复，这发生在两种情况下：

* **没有 hook 回答命令**：模块没有 `command.run` hook，hook 的 [filter](/docs/zh-CN/plugins/mods/events#filter-which-events-a-hook-handles) 命名了不同的命令，或 hook 返回了 `next(e)`
* **Claude Code 跳过了 hook**：[`hook skipped`](#hook-skipped) 列出了原因。将 `focus: false` 传递给 [`$.ui.open`](/docs/zh-CN/plugins/mods/interface#open-a-pane-at-the-right-time) 是到达那里的一种方式。

如果模块已经有回复描述的 hook，请查找命名 `command.run` 的 `hook skipped` 行，它给出了原因。运行命令的 [test](/docs/zh-CN/plugins/mods/test) 失败，原因相同。

<h3 id="it-crashed-the-hooks-worker">
  `it crashed the hooks worker`
</h3>

该行以 mod 的名称开头，如 `first-mod was unloaded: it crashed the hooks worker`。已安装的 mod 共享一个工作线程。工作线程停止响应或崩溃，Claude Code 将其追踪到此 mod 并卸载了它。阻止线程的 hook（例如永不等待的循环）是一个原因。

修复 hook。

<h3 id="its-session-start-ran-again-in-a-fresh-copy">
  `its session.start ran again in a fresh copy`
</h3>

该行以 mod 的名称开头，并指出一个 `$.prompt.submit`、`$.command.run` 或 `$.agent.spawn` 调用，如 `first-mod: its session.start ran again in a fresh copy; the $.prompt.submit call it had already made was not made again`。Claude Code 再次加载了该 mod 的模块（例如在 hooks worker 崩溃并被替换之后），新副本的 [`session.start`](/docs/zh-CN/plugins/mods/reference#session) hook 运行了。该行指出的调用以其首次运行的结果完成，而没有再次运行，因此您的 mod 不会重复提交提示词、运行命令或启动子代理。hook 的其余部分照常运行。

无需修复。

在 v2.1.292 之前，该调用会再运行一次，因此提示词会被提交两次、命令会被运行两次，或子代理会被启动两次。

<h3 id="mods-that-run-in-the-hooks-worker-are-off-for-this-session">
  `mods that run in the hooks worker are off for this session`
</h3>

该行读取 `hooks: mods that run in the hooks worker are off for this session: it crashed 3 times`。工作线程停止了三次，Claude Code 无法将停止追踪到一个 mod，因此它卸载了每个不是内置的 mod，包括您的组织安装的 mod。此行在每个交互式会话中到达成绩单。

运行 `/reload-plugins` 以再次加载它们。

<h2 id="a-tool-call-is-denied">
  工具调用被拒绝
</h2>

mod 已加载，其 hooks 运行，它接触的工具调用被拒绝。

<h3 id="a-hook-changed-this-call’s-input-after-the-model-wrote-it">
  `a hook changed this call's input after the model wrote it`
</h3>

在自动模式下，被拒绝的工具调用给出此原因。hook 在 [服务器端分类器](/docs/zh-CN/permission-modes#server-side-classifier-review) 审查后更改了工具调用的输入，因此该审查不涵盖将运行的内容。hook 可以是 mod 的 [`tool.call`](/docs/zh-CN/plugins/mods/reference#tools) 或 [`turn.step`](/docs/zh-CN/plugins/mods/reference#turns) hook，或 [`PreToolUse`](/docs/zh-CN/hooks#pretooluse) 设置 hook。消息不说明是哪一个。

消息告诉 Claude 再次发出记录的调用。如果也被拒绝，hook 每次都更改输入，因此关闭 mod 或 hook，或离开自动模式并自己批准调用。

<h3 id="a-message-about-the-deny-rules-in-your-settings">
  关于您的设置中的拒绝规则的消息
</h3>

`tried to lift a deny rule in your settings` 和 `the deny rules in your settings could not be checked for this call, so it is refused` 都来自内置保护。

在 [来自内置保护的消息](#messages-from-the-built-in-guard) 中查找它们。

<h2 id="a-drawing-doesn’t-appear-or-respond">
  绘图不出现或不响应
</h2>

mod 已加载，其窗格、带、toast 或控件的行为不符合您的预期。

<h3 id="a-pane-or-band-is-empty-or-shows-claude-code’s-usual-content">
  窗格或带为空或显示 Claude Code 的常规内容
</h3>

您的 hook 返回的 [树](/docs/zh-CN/plugins/mods/interface#build-a-tree-from-elements) 未通过验证。使用 `--plugin-dir` 时，会话记录会显示 `ui.render (Pane) refused:` 及原因，如 `first-mod: ui.render (Pane) refused: Box prop "flexDirection" must be one of row, column, row-reverse, column-reverse; the engine drew its own`。调试日志中有 `a hook returned a tree that does not validate` 及相同的原因。

读取该行上的原因。常见原因是元素不接受的 prop 和应用没有的元素。

<h3 id="a-ui-render-line-says-threw-while-drawn">
  `ui.render` 行显示 `threw while drawn`
</h3>

该行会指明[渲染位置](/docs/zh-CN/plugins/mods/reference#render-sites)，然后显示 `threw while drawn:` 及错误，如 `first-mod: ui.render (ToolUse) threw while drawn: <error>; the engine drew its own`。Claude Code 在绘制您的 [`ui.render`](/docs/zh-CN/plugins/mods/reference#interface) hook 返回的树时，或在根据[您的 hook 传递给 `next` 的 `props`](/docs/zh-CN/plugins/mods/interface#change-what-claude-code-already-draws) 绘制该位置时，遇到了该错误。结尾的 `the engine drew its own` 表示该位置显示 Claude Code 的常规内容。

读取该错误，并修复您的 hook 中导致该错误的值。

在 v2.1.289 之前，会话记录行中出现此错误会导致会话以 [`Claude Code exited after an unrecoverable interface error`](/docs/zh-CN/errors#exited-after-an-unrecoverable-interface-error) 结束。

<h3 id="the-module-failed-without-a-message">
  `the module failed without a message`
</h3>

某个 [`Client`](/docs/zh-CN/plugins/mods/interface#when-a-client-fails) 因一个没有消息的错误而失败，例如 `throw new Error()`。在其位置显示的行类似 `my-mod: Client client/spinner.js: the module failed without a message`。

在您的 `Client` 代码中找到该 throw，并为该错误提供消息。该行随后会显示该消息。

在 v2.1.289 之前，该行改为显示 `Error` 作为原因。

<h3 id="$-ui-open-runs-and-no-pane-appears">
  `$.ui.open` 运行且没有窗格出现
</h3>

调用不是来自用户做的事情，且终端宽度小于[该窗格所需的宽度](/docs/zh-CN/plugins/mods/interface#when-a-pane-waits-for-a-wider-terminal)。

从命令或按钮打开窗格，或检查调用的 `isPlaced` 结果。请参阅 [在正确的时间打开窗格](/docs/zh-CN/plugins/mods/interface#open-a-pane-at-the-right-time)。

<h3 id="a-toast-doesn’t-appear">
  toast 不出现
</h3>

您的 mod 在交互式终端会话中调用了 [`$.ui.toast`](/docs/zh-CN/plugins/mods/api#show-something-without-starting-a-turn)，但您没有看到该 toast。要确认调用已运行，请在[调试日志](#read-the-debug-log)中查找包含您的 mod 名称和 toast 文本的行，如 `$.ui.toast (first-mod): build finished`。然后检查以下原因：

* **缺少该调用对应的行**：查找说明 Claude Code 拒绝该调用原因的行，如 `first-mod: $.ui.toast dropped: timeoutMs is a whole number of ms, 1 to 60000`。
* **某个窗格正在暂缓 toast**：您的 mod 或其他 mod 在打开当前显示的窗格时传递了 [`holdToasts`](/docs/zh-CN/plugins/mods/interface#hold-toasts-behind-a-dialog)。关闭该窗格即可结束暂缓。如果该窗格是您的且需要保持打开，请从其 `$.ui.open` 调用中移除 `holdToasts`，然后重新打开该窗格。
* **toast 位于输入框下方**：在[经典渲染器](/docs/zh-CN/fullscreen#enable-fullscreen-rendering)中，请查看输入框下方的右侧。该处的 toast 是以 mod 名称开头的一行文字，而不是右上角的方框。
* **您的 mod 发出了更新的 toast**：在经典渲染器中，来自您的 mod 的更新 toast 可能会取代正在显示或等待显示的 toast。调试日志中会有针对较早 toast 的另一行：如果该 toast 当时正在显示，该行以 `gave way, cut short` 结尾；如果它从未出现，则以 `gave way, unseen` 结尾。要同时显示两条消息，请将它们放在一个 toast 中。
* **toast 在绘制前超时**：在全屏渲染中，Claude Code 一次最多绘制三个 toast，因此 toast 可能在绘制之前就已超时。调试日志中会有针对该 toast 的另一行，以 `left the stack, never drawn` 结尾。当您的 mod 同时发出多个 toast 时，请将这些消息放在一个 toast 中。

在 v2.1.290 之前，如果某个 toast 是在 Claude Code 为您的 mod 显示上一个 toast 后两秒内发出的，Claude Code 会丢弃该 toast，且调试日志中针对被丢弃 toast 的行会显示 `within 2000ms of the last; dropped`。

<h3 id="hotkeys-do-nothing">
  热键不起作用
</h3>

您的窗格没有键盘焦点。

按 Ctrl+X 然后 Tab，或单击窗格。使用 `focus: true` 从命令打开它。

<h3 id="a-drawing-works-in-the-terminal-and-not-in-the-desktop-app">
  绘图在终端中有效，在桌面应用中无效
</h3>

该位置或元素在那里不可用。

检查 [渲染位置](/docs/zh-CN/plugins/mods/reference#render-sites) 和 [元素](/docs/zh-CN/plugins/mods/reference#elements) 表。

<h2 id="an-edit-or-a-value-is-lost">
  编辑或值丢失
</h2>

mod 运行，您所做的更改或它保留的值不存在。

<h3 id="your-edits-don’t-take-effect">
  您的编辑不生效
</h3>

您正在编辑您安装的插件。Claude Code 运行已安装版本的缓存副本。

使用指向您的工作副本的 `--plugin-dir` 进行开发，如 `claude --plugin-dir ./first-mod`，它在您保存时重新加载。

<h3 id="a-value-resets-when-the-module-reloads">
  模块重新加载时值重置
</h3>

模块级变量在每次重新加载时重新初始化。

[将值保留在 `$.state` 或 `$.store` 中](/docs/zh-CN/plugins/mods/interface#keep-state)。

<h3 id="a-value-resets-after-/clear-/resume-or-/branch">
  值在 `/clear`、`/resume` 或 `/branch` 后重置
</h3>

值重置，或保存的值被其默认值替换。这些命令中的每一个都将 `$.state` 重置为其默认值，`session.start` 不再触发。

[在 `classic.SessionStart` hook 中再次加载保存的值](/docs/zh-CN/plugins/mods/interface#load-a-saved-value-again-after-clear)。

<h2 id="read-the-debug-log">
  阅读调试日志
</h2>

调试日志对 Claude Code 加载或拒绝的每个模块、每个失败的 hook 以及它拒绝的每个结果都有一行，因此当会话记录中没有任何显示时，这是查看的地方。要写入一个，在您的 shell 中使用 `--debug` 启动 Claude Code，或使用 `--debug-file <path>` 选择它的位置：

```bash theme={null}
claude --debug-file ./mod-debug.log --plugin-dir ./first-mod
```

在另一个终端中，跟踪文件并按您的 mod 名称过滤：

```bash theme={null}
tail -f ./mod-debug.log | grep first-mod
```

已加载的 mod 有一行命名它并列出它处理的事件。使用 `--plugin-dir` 加载的 mod 出现在其名称后跟 `@inline` 下：

```text theme={null}
hooks module first-mod@inline loaded (worker, environment 2, tier user); events: session.start,tool.call,command.run,ui.render
```

未验证的绘图计为被拒绝的结果，也会获得一行。要在日志中写入您自己的行，请调用 [`$.ui.log`](/docs/zh-CN/plugins/mods/api#show-something-without-starting-a-turn)，带有第二个参数，如 `$.ui.log('message', { to: 'debug' })`。没有第二个参数，`$.ui.log` 会在会话记录中添加一条暗行。

当您编辑使用 `--plugin-dir` 加载的 mod 时，会话记录为每次重新加载显示一行，命名 mod 并列出其 hook。如果保存破坏了模块，该行会显示 `reload failed, the previous version stays loaded:` 及原因，最后一个可用版本会继续运行，直到 Claude Code 下次重新加载插件，例如当您运行 `/reload-plugins` 时。

<h2 id="next-steps">
  后续步骤
</h2>

* [测试 mod](/docs/zh-CN/plugins/mods/test)：在问题到达会话之前捕获它们
* [排查插件问题](/docs/zh-CN/plugins/troubleshooting)：与安装和加载不特定于 mod 的插件相关的问题
