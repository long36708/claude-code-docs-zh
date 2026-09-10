> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 输出样式

> 将 Claude Code 适配用于软件工程之外的用途

输出样式改变 Claude 的响应方式，而不是 Claude 知道什么。它们设置 Claude 的角色、语气和输出格式，用于每个响应。当你在每个回合中不断重新提示相同的语音或格式时，或者当你希望 Claude 充当软件工程师以外的角色时，请使用一个。

自定义输出样式为 Claude 提供你自己的说明，并让你选择是否保留 Claude Code 的内置软件工程说明。当你改变 Claude 的通信方式但仍在编码时（例如总是用图表回答），请保留它们。当 Claude 根本不进行软件工程时（例如写作助手或数据分析师），请省略它们。

有关你的项目、约定或代码库的说明，请改用 [CLAUDE.md](/docs/zh-CN/memory)。

<h2 id="built-in-output-styles">
  内置输出样式
</h2>

Claude Code 的**默认**输出样式是其标准指令集，旨在帮助你高效地完成软件工程任务。

还有四种额外的内置输出样式：

* **Proactive**：Claude 立即执行，做出合理的假设而不是暂停进行常规决策，并倾向于行动而非规划。这提供了比[自动模式](/docs/zh-CN/permission-modes#eliminate-prompts-with-auto-mode)更强的自主执行指导，并且无需更改你的权限模式即可工作，因此你的权限模式仍然决定什么在不询问的情况下运行。

* **Concise**：Claude 以结果开头，跳过前言和叙述，默认保持响应简洁，同时以与默认样式相同的彻底程度完成工程工作。当你要求解释或更多细节时，Claude 会完整回答。Claude 始终保留错误报告、安全警告和破坏性操作确认的完整内容。需要 Claude Code v2.1.237 或更高版本。

* **Explanatory**：在帮助你完成软件工程任务的同时提供教育性的"Insights"。帮助你理解实现选择和代码库模式。

* **Learning**：协作式的边学边做模式，Claude 不仅会在编码时分享"Insights"，还会要求你自己贡献小的、战略性的代码片段。Claude Code 将在你的代码中添加 `TODO(human)` 标记供你实现。

<h2 id="change-your-output-style">
  更改你的输出样式
</h2>

通过以下方式之一选择样式：

* **Terminal**：运行 `/config` 并选择**输出样式**从菜单中选择一种样式。Claude Code 将你的选择保存到[本地项目级别](/docs/zh-CN/settings)的 `.claude/settings.local.json`。
* **VS Code extension**：使用 `/` 打开[命令菜单](/docs/zh-CN/vs-code#use-the-prompt-box)并选择**输出样式**来选择一种样式，包括你的自定义样式。Claude Code 将你的选择保存到 `.claude/settings.local.json`，这是终端菜单写入的同一个文件。需要 Claude Code v2.1.257 或更高版本。
* **Desktop app**：在设置文件中设置 `outputStyle` 字段，例如 `.claude/settings.local.json`，这是终端菜单写入的文件。当你在那里运行 `/config` 时，Claude Code [打开**设置 > Claude Code**](/docs/zh-CN/desktop#what%E2%80%99s-not-available-in-desktop)而不是菜单。

<Note>独立的 `/output-style` 命令在 v2.1.73 中已弃用，在 v2.1.91 中被移除。使用 `/config` 或直接编辑 `outputStyle` 设置。</Note>

要在不使用菜单的情况下设置样式，直接编辑设置文件中的 `outputStyle` 字段：

```json theme={null}
{
  "outputStyle": "Explanatory"
}
```

当你在会话中途切换样式时，Claude 从你的下一条消息开始使用新样式。关于该第一条消息在 prompt caching 中的成本，请参阅[更改输出样式](/docs/zh-CN/prompt-caching#changing-output-style)。在 v2.1.251 之前，新样式仅在你运行 `/clear` 或开始新会话后才会应用。

<h2 id="create-a-custom-output-style">
  创建自定义输出样式
</h2>

自定义输出样式是一个 Markdown 文件：frontmatter 用于元数据，然后是 Claude 的说明。

在 VS Code 扩展中，你也可以从[**输出样式**菜单](/docs/zh-CN/vs-code#use-the-prompt-box)创建文件，而不是手动编写。这需要 Claude Code v2.1.261 或更高版本。

<Steps>
  <Step title="创建一个 Markdown 文件">
    在三个级别之一保存它。文件名成为样式名称，除非你在 frontmatter 中设置 `name`。

    * 用户：`~/.claude/output-styles`
    * 项目：`.claude/output-styles`
    * 托管策略：[托管设置目录](/docs/zh-CN/managed-settings#delivery-mechanisms)内的 `.claude/output-styles`

    项目输出样式从工作目录和仓库根目录之间的每个 `.claude/output-styles/` 加载。当多个这样的嵌套目录定义了同名样式时，Claude Code 使用最接近工作目录的那个。
  </Step>

  <Step title="添加 frontmatter 和说明">
    决定是否保留 Claude Code 的软件工程说明。如果你改变 Claude 的通信方式但仍希望它以相同的方式编码，请设置 `keep-coding-instructions: true`。如果 Claude 不会进行软件工程，请省略它。

    此示例在保留 Claude 编码行为的同时，在每个解释前面加上一个图表：

    ```markdown theme={null}
    ---
    name: Diagrams first
    description: Lead every explanation with a diagram
    keep-coding-instructions: true
    ---

    When explaining code, architecture, or data flow, start with a Mermaid diagram showing the structure, then explain in prose.

    ## Diagram conventions

    Use `flowchart TD` for control flow and `sequenceDiagram` for request paths. Keep diagrams under 15 nodes.
    ```
  </Step>

  <Step title="切换到你的样式">
    在终端中运行 `/config` 并在**输出样式**下选择你的样式。Claude 从你的下一条消息开始使用新样式。在终端中，Claude Code 在启动时读取样式文件，所以如果你在运行会话期间创建或编辑一个样式文件，请重启 Claude Code 以获取更改。
  </Step>
</Steps>

[Plugins](/docs/zh-CN/plugins-reference) 也可以在 `output-styles/` 目录中提供输出样式。

<h3 id="frontmatter">
  Frontmatter
</h3>

输出样式文件支持这些 frontmatter 字段：

| Frontmatter                | 目的                                                                                                             | 默认值     |
| :------------------------- | :------------------------------------------------------------------------------------------------------------- | :------ |
| `name`                     | 输出样式的名称，如果不是文件名                                                                                                | 从文件名继承  |
| `description`              | 输出样式的描述，在 `/config` 选择器中显示                                                                                     | 无       |
| `keep-coding-instructions` | 保留 Claude Code 的内置软件工程说明                                                                                       | `false` |
| `force-for-plugin`         | 仅限 Plugin 输出样式：在启用 plugin 时自动应用此样式，无需要求用户选择它。覆盖用户的 `outputStyle` 设置。如果多个启用的 plugin 设置了此项，Claude Code 使用第一个加载的。 | `false` |

<h2 id="how-output-styles-work">
  输出样式如何工作
</h2>

输出样式改变了 Claude Code 给予 Claude 的指令。

* Claude Code 在每个请求中发送活跃样式的指令。
* 当你[选择除 Default 以外的样式](#change-your-output-style)时，Claude Code 也会在对话期间提醒 Claude 该样式。
* 自定义输出样式排除了 Claude Code 的内置软件工程说明，例如如何限定更改范围、编写注释和验证工作，除非 `keep-coding-instructions` 设置为 `true`。

输出样式适用于主对话和[分支](/docs/zh-CN/sub-agents#fork-the-current-conversation)，分支继承父级的完整对话和系统提示。其他[子代理运行自己的系统提示](/docs/zh-CN/sub-agents#what-loads-at-startup)，所以样式不会改变它们的响应方式。

令牌使用情况取决于样式。样式的指令会增加输入令牌，尽管 prompt caching 在会话中的第一个请求之后会降低这个成本。

内置的 Explanatory 和 Learning 样式在设计上比 Default 产生更长的响应，这会增加输出令牌。Concise 样式则相反，通过指示 Claude 默认保持响应简洁来实现。对于自定义样式，输出令牌使用情况取决于你的指令告诉 Claude 生成什么。

<h2 id="comparisons-to-related-features">
  与相关功能的比较
</h2>

多个功能自定义 Claude Code 的行为方式。输出样式改变 Claude Code 的默认说明并应用于每个响应。其他功能添加说明而不改变默认设置，或将其范围限定为特定任务。

| 功能                          | 工作原理                 | 何时使用                                                                     |
| :-------------------------- | :------------------- | :----------------------------------------------------------------------- |
| 输出样式                        | 改变 Claude Code 的默认说明 | 你想要每个回合都有不同的角色、语气或默认响应格式                                                 |
| [CLAUDE.md](/docs/zh-CN/memory)  | 在系统提示之后添加用户消息        | Claude 应该始终了解你的项目约定和代码库上下文                                               |
| `--append-system-prompt`    | 附加到系统提示而不删除任何内容      | 你想要一次性添加单个调用，作为[启动时的 CLI 标志](/docs/zh-CN/cli-reference#system-prompt-flags)传递 |
| [Agents](/docs/zh-CN/sub-agents) | 使用自己的系统提示、模型和工具运行子代理 | 你想要一个单独作用域的辅助工具来完成专注的任务                                                  |
| [Skills](/docs/zh-CN/skills)     | 在调用时或相关时加载特定于任务的说明   | 你有一个可重用的工作流                                                              |

<h2 id="related-resources">
  相关资源
</h2>

* [Settings](/docs/zh-CN/settings)：`outputStyle` 字段所在的位置以及设置优先级的工作原理
* [Permission modes](/docs/zh-CN/permission-modes)：Proactive 样式与自动模式的比较方式
* [Plugins](/docs/zh-CN/plugins)：打包和分发输出样式以及 skills、hooks 和 agents
* [Debug your configuration](/docs/zh-CN/debug-your-config)：诊断为什么输出样式没有生效
