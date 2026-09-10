> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Plugins 参考

> Claude Code 插件系统的完整技术参考，包括模式、CLI 命令和组件规范。

<Tip>
  想要安装插件？请参阅 [发现和安装插件](/docs/zh-CN/discover-plugins)。有关创建插件，请参阅 [Plugins](/docs/zh-CN/plugins)。有关分发插件，请参阅 [Plugin marketplaces](/docs/zh-CN/plugin-marketplaces)。
</Tip>

**插件**是一个自包含的组件目录，使用自定义功能扩展 Claude Code。插件组件包括 skills、agents、hooks、MCP servers、LSP servers 和 monitors。

<h2 id="plugin-components-reference">
  插件组件参考
</h2>

<h3 id="skills">
  Skills
</h3>

插件向 Claude Code 添加 skills，创建可由您或 Claude 调用的 `/name` 快捷方式。

**位置**：插件根目录中的 `skills/` 或 `commands/` 目录，或插件根目录中的单个 `SKILL.md` 文件

**文件格式**：Skills 是包含 `SKILL.md` 的目录；commands 是简单的 markdown 文件

**Skill 结构**：

```text theme={null}
skills/
├── pdf-processor/
│   ├── SKILL.md
│   ├── reference.md (optional)
│   └── scripts/ (optional)
└── code-reviewer/
    └── SKILL.md
```

安装插件时会自动发现 Skills 和 commands。

如果插件没有 `skills/` 目录且没有 `skills` manifest 字段，则插件根目录中的 `SKILL.md` 会作为单个 skill 加载。设置 frontmatter `name` 字段来控制 skill 的调用名称。如果没有设置，Claude Code 会回退到安装目录名称，对于从市场安装的插件，这是一个在每次更新时都会改变的版本字符串。对于包含多个 skill 的插件，请使用上面所示的 `skills/` 目录布局。

在插件 skills 和 commands 中，Boolean frontmatter 字段（如 `disable-model-invocation`）接受 `yes`、`no`、`on`、`off`、`1` 和 `0`（任何字母大小写），以及 `true` 和 `false`。在 v2.1.218 之前，Claude Code 仅识别 `true` 和 `false`。

有关完整详情，请参阅 [Skills](/docs/zh-CN/skills)。

<h3 id="agents">
  Agents
</h3>

插件可以为特定任务提供专门的子代理，Claude 可以在适当时自动调用这些代理。

**位置**：插件根目录中的 `agents/` 目录

**文件格式**：描述代理功能的 markdown 文件

**Agent 结构**：

```markdown theme={null}
---
name: agent-name
description: What this agent specializes in and when Claude should invoke it
model: sonnet
effort: medium
maxTurns: 20
disallowedTools: Write, Edit
---

Detailed system prompt for the agent describing its role, expertise, and behavior.
```

插件代理支持 `name`、`description`、`model`、`effort`、`maxTurns`、`tools`、`disallowedTools`、`skills`、`memory`、`background` 和 `isolation` frontmatter 字段。唯一有效的 `isolation` 值是 `"worktree"`。出于安全原因，插件提供的代理不支持 `hooks`、`mcpServers` 和 `permissionMode`。

Claude Code 会加载插件代理，即使其 frontmatter 没有 `name` 或无法解析：

* 没有 `name`：Claude Code 根据文件名命名代理，因此名为 `my-plugin` 的插件中的 `agents/reviewer.md` 会加载为 `my-plugin:reviewer`
* Frontmatter 无法解析：Claude Code 根据文件名命名代理，使用 `Agent from my-plugin plugin` 作为其描述，并忽略文件中的每个字段

相比之下，Claude Code 会跳过其 frontmatter 没有 `name` 或无法解析的项目、用户或托管代理文件。

要查找插件默认 `agents/` 目录中 frontmatter 无法解析的文件，请运行 `claude plugin validate`。您传递的路径取决于插件是否有 manifest，两个示例都使用 `./my-plugin` 作为插件目录：

* 具有 manifest 的插件：`claude plugin validate ./my-plugin`
* 没有 manifest 的插件：`claude plugin validate ./my-plugin/agents`。需要 Claude Code v2.1.233 或更高版本。

启用插件后，代理会在 [@-mention 类型提示](/docs/zh-CN/sub-agents#invoke-subagents-explicitly) 中显示其作用域名称，例如 `my-plugin:code-reviewer`。

有关完整详情，请参阅 [Subagents](/docs/zh-CN/sub-agents)。

<h3 id="hooks">
  Hooks
</h3>

插件可以提供事件处理程序，自动响应 Claude Code 事件。

**位置**：插件根目录中的 `hooks/hooks.json`，或在 plugin.json 中内联

**格式**：具有事件匹配器和操作的 JSON 配置

**Hook 配置**：

```json theme={null}
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}\"/scripts/format-code.sh"
          }
        ]
      }
    ]
  }
}
```

插件 hooks 响应与 [用户定义的 hooks](/docs/zh-CN/hooks) 相同的生命周期事件：

| Event                 | When it fires                                                                                                                                                                                                                                         |
| :-------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SessionStart`        | When a session begins or resumes                                                                                                                                                                                                                      |
| `Setup`               | When you start Claude Code with `--init-only`, or with `--init` or `--maintenance` in `-p` mode. For one-time preparation in CI or scripts                                                                                                            |
| `UserPromptSubmit`    | When you submit a prompt, before Claude processes it                                                                                                                                                                                                  |
| `UserPromptExpansion` | When a user-typed command expands into a prompt, before it reaches Claude. Can block the expansion                                                                                                                                                    |
| `PreToolUse`          | Before a tool call executes. Can block it                                                                                                                                                                                                             |
| `PermissionRequest`   | When a tool call needs a permission decision                                                                                                                                                                                                          |
| `PermissionDenied`    | When auto mode denies a tool call, including denials without a classifier verdict. Use JSON `hookSpecificOutput.retry: true` to tell the model it may retry the denied tool call. Claude Code ignores `retry` when the classifier produced no verdict |
| `PostToolUse`         | After a tool call succeeds                                                                                                                                                                                                                            |
| `PostToolUseFailure`  | After a tool call fails                                                                                                                                                                                                                               |
| `PostToolBatch`       | After a full batch of parallel tool calls resolves, before the next model call                                                                                                                                                                        |
| `Notification`        | When Claude Code sends a notification                                                                                                                                                                                                                 |
| `MessageDisplay`      | While assistant message text is displayed                                                                                                                                                                                                             |
| `SubagentStart`       | When a subagent is spawned                                                                                                                                                                                                                            |
| `SubagentStop`        | When a subagent finishes                                                                                                                                                                                                                              |
| `TaskCreated`         | When a task is being created via `TaskCreate`                                                                                                                                                                                                         |
| `TaskCompleted`       | When a task is being marked as completed                                                                                                                                                                                                              |
| `Stop`                | When Claude finishes responding                                                                                                                                                                                                                       |
| `StopFailure`         | When the turn ends due to an API error                                                                                                                                                                                                                |
| `TeammateIdle`        | When an [agent team](/docs/en/agent-teams) teammate is about to go idle                                                                                                                                                                                    |
| `InstructionsLoaded`  | When a CLAUDE.md or `.claude/rules/*.md` file is loaded into context. Fires at session start and when files are lazily loaded during a session                                                                                                        |
| `ConfigChange`        | When a configuration file changes during a session                                                                                                                                                                                                    |
| `CwdChanged`          | When the working directory changes, for example when Claude executes a `cd` command. Useful for reactive environment management with tools like direnv                                                                                                |
| `DirectoryAdded`      | When a working directory is added mid-session via `/add-dir` or the SDK `register_repo_root` control request                                                                                                                                          |
| `FileChanged`         | When a watched file changes on disk. The `matcher` field specifies which filenames to watch                                                                                                                                                           |
| `WorktreeCreate`      | When a worktree is being created via `--worktree`, `isolation: "worktree"`, or for a background session. Replaces default git behavior                                                                                                                |
| `WorktreeRemove`      | When a worktree is being removed at session exit, when a subagent finishes, or when you delete a background session                                                                                                                                   |
| `PreCompact`          | Before context compaction                                                                                                                                                                                                                             |
| `PostCompact`         | After context compaction completes                                                                                                                                                                                                                    |
| `PreModelSwitch`      | Before Claude Code applies a model switch that you or a client requested. Can block the switch                                                                                                                                                        |
| `PostModelSwitch`     | After the session's model changes, including changes Claude Code makes on its own, such as restoring the model when you resume a session                                                                                                              |
| `Elicitation`         | When an MCP server requests user input during a tool call                                                                                                                                                                                             |
| `ElicitationResult`   | After a user responds to an MCP elicitation, before the response is sent back to the server                                                                                                                                                           |
| `SessionEnd`          | When a session terminates                                                                                                                                                                                                                             |

**Hook 类型**：

* `command`：执行 shell 命令或脚本
* `http`：将事件 JSON 作为 POST 请求发送到 URL
* `mcp_tool`：在配置的 [MCP server](/docs/zh-CN/mcp) 上调用工具
* `prompt`：使用 LLM 评估提示（使用 `$ARGUMENTS` 占位符作为上下文）
* `agent`：运行具有工具的代理验证器以完成复杂验证任务

针对插件自己的 [捆绑 MCP server](#mcp-servers) 的 hooks 必须使用其作用域名称。工具匹配器和 `if` 字段采用作用域工具名称 `mcp__plugin_<plugin-name>_<server-name>__<tool>`，`mcp_tool` hook 的 `server` 字段采用 `plugin:<plugin-name>:<server-name>`。针对裸服务器密钥编写的匹配器永远不会触发。请参阅 [Match MCP tools](/docs/zh-CN/hooks#match-mcp-tools) 和 [Plugin-provided MCP servers](/docs/zh-CN/mcp#plugin-provided-mcp-servers)。

<h3 id="mcp-servers">
  MCP servers
</h3>

插件可以捆绑 Model Context Protocol (MCP) 服务器，以将 Claude Code 与外部工具和服务连接。

**位置**：插件根目录中的 `.mcp.json`，或在 plugin.json 中内联

**格式**：标准 MCP 服务器配置

**MCP 服务器配置**：

```json theme={null}
{
  "mcpServers": {
    "plugin-database": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
      "env": {
        "DB_PATH": "${CLAUDE_PLUGIN_ROOT}/data"
      }
    },
    "plugin-api-client": {
      "command": "npx",
      "args": ["@company/mcp-server", "--plugin-mode"]
    }
  }
}
```

**集成行为**：

* 启用插件时，插件 MCP 服务器会自动启动
* 服务器在 Claude 的工具包中显示为标准 MCP 工具
* 插件服务器可以独立于用户 MCP 服务器进行配置
* 如果您在会话中途运行 [`/reload-plugins`](/docs/zh-CN/discover-plugins#apply-plugin-changes-without-restarting)，Claude Code 会保持配置未更改的服务器的实时连接

<h3 id="lsp-servers">
  LSP servers
</h3>

<Tip>
  想要使用 LSP 插件？从官方市场安装它们：在 `/plugin` Discover 选项卡中搜索"lsp"。本部分记录如何为官方市场未涵盖的语言创建 LSP 插件。
</Tip>

插件可以提供 [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) (LSP) 服务器，以在处理您的代码库时为 Claude 提供 [实时代码智能](/docs/zh-CN/discover-plugins#code-intelligence)。

**位置**：插件根目录中的 `.lsp.json`，或在 `plugin.json` 中内联

**格式**：将语言服务器名称映射到其配置的 JSON 配置

**`.lsp.json` 文件格式**：

```json theme={null}
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

**在 `plugin.json` 中内联**：

```json theme={null}
{
  "name": "my-plugin",
  "lspServers": {
    "go": {
      "command": "gopls",
      "args": ["serve"],
      "extensionToLanguage": {
        ".go": "go"
      }
    }
  }
}
```

**必需字段：**

| 字段                    | 描述                         |
| :-------------------- | :------------------------- |
| `command`             | 要执行的 LSP 二进制文件（必须在 PATH 中） |
| `extensionToLanguage` | 将文件扩展名映射到语言标识符             |

**可选字段：**

| 字段                      | 描述                                                                                          |
| :---------------------- | :------------------------------------------------------------------------------------------ |
| `args`                  | LSP 服务器的命令行参数                                                                               |
| `transport`             | 通信传输：`stdio`（默认）或 `socket`。Claude Code 接受 `socket` 但在 stdio 上运行每个服务器，因此 stdout 协议规则适用于所有服务器 |
| `env`                   | 启动服务器时要设置的环境变量                                                                              |
| `initializationOptions` | 在初始化期间传递给服务器的选项                                                                             |
| `settings`              | 通过 `workspace/didChangeConfiguration` 传递的设置                                                 |
| `workspaceFolder`       | 服务器的工作区文件夹路径                                                                                |
| `startupTimeout`        | 等待服务器启动的最长时间（毫秒）                                                                            |
| `shutdownTimeout`       | 等待正常关闭的最长时间（毫秒）。当超时时间过去时，Claude Code 会终止服务器进程。未设置时，不适用超时                                    |
| `restartOnCrash`        | 服务器崩溃后是否重新启动。默认为 `true`。设置为 `false` 以保持崩溃的服务器停止而不是重新启动它                                     |
| `maxRestarts`           | 放弃前的最大重启尝试次数                                                                                |
| `diagnostics`           | 编辑后是否将诊断推送到 Claude 的上下文中（默认 `true`）。设置为 `false` 以保持代码导航但禁止自动诊断注入。                           |

`restartOnCrash` 和 `shutdownTimeout` 需要 Claude Code v2.1.205 或更高版本。在 v2.1.205 之前，配置架构接受两个选项，但设置其中任何一个都会导致 Claude Code 在启动时完全跳过该 LSP 服务器，原因仅在 `claude --debug` 输出中可见。

**同一扩展名的多个服务器**：当多个启用的 LSP 服务器在 `extensionToLanguage` 中声明相同的文件扩展名时，无论服务器来自一个插件还是来自不同的插件，第一个注册的服务器处理具有该扩展名的文件，其他服务器永远不会启动。`/plugin` 界面显示一个警告，命名其服务器处于活动状态的插件。

**无法初始化的服务器**：Claude Code 会跳过配置无效的服务器，例如缺少 `command` 或 `extensionToLanguage` 的服务器，其他配置的服务器仍会启动。运行 `claude --debug` 以查看服务器被跳过的原因。

被跳过的服务器不会声明其文件扩展名，因此声明相同扩展名的另一个有效服务器（来自同一插件或不同插件）仍会处理这些文件。

**将日志输出发送到 stderr，而不是 stdout**：Claude Code 仅将服务器的 stdout 读取为协议消息，并接受最大 64 KiB 的消息头和最大 32 MiB 的消息体。Claude Code 会断开超过任一限制或向 stdout 写入非协议输出的服务器，并将断开连接计为 `restartOnCrash` 和 `maxRestarts` 的崩溃。当您使用 `--debug` 运行时，Claude Code 会将命名原因的错误写入调试日志。

<Warning>
  **您必须单独安装语言服务器二进制文件。** LSP 插件配置 Claude Code 如何连接到语言服务器，但它们不包括服务器本身。如果您在 `/plugin` Errors 选项卡中看到 `Executable not found in $PATH`，请为您的语言安装所需的二进制文件。
</Warning>

**可用的 LSP 插件：**

| 插件                  | 语言服务器                      | 安装命令                                                                            |
| :------------------ | :------------------------- | :------------------------------------------------------------------------------ |
| `pyright-lsp`       | Pyright (Python)           | `pip install pyright` 或 `npm install -g pyright`                                |
| `typescript-lsp`    | TypeScript Language Server | `npm install -g typescript-language-server typescript`                          |
| `rust-analyzer-lsp` | rust-analyzer              | [参见 rust-analyzer 安装](https://rust-analyzer.github.io/manual.html#installation) |

首先安装语言服务器，然后从市场安装插件。

<h3 id="monitors">
  Monitors
</h3>

插件可以声明后台监视器，Claude Code 在插件处于活动状态时自动启动。每个监视器在会话的生命周期内运行 shell 命令，并将每个 stdout 行作为通知传递给 Claude，以便 Claude 可以对日志条目、状态更改或轮询事件做出反应，而无需被要求自己启动监视。

插件监视器使用与 [Monitor tool](/docs/zh-CN/tools-reference#monitor-tool) 相同的机制，并共享其可用性约束。它们仅在交互式 CLI 会话中运行，以与 [hooks](#hooks) 相同的信任级别在非沙箱环境中运行，并在 Monitor tool 不可用的主机上被跳过。

**位置**：插件根目录中的 `monitors/monitors.json`，或在 `plugin.json` 中内联

**格式**：监视器条目的 JSON 数组

以下 `monitors/monitors.json` 监视部署状态端点和本地错误日志：

```json theme={null}
[
  {
    "name": "deploy-status",
    "command": "\"${CLAUDE_PLUGIN_ROOT}\"/scripts/poll-deploy.sh",
    "description": "Deployment status changes"
  },
  {
    "name": "error-log",
    "command": "tail -F ./logs/error.log",
    "description": "Application error log",
    "when": "on-skill-invoke:debug"
  }
]
```

要内联声明监视器，请在 `plugin.json` 中将 `experimental.monitors` 设置为相同的数组。要从非默认路径加载，请将 `experimental.monitors` 设置为相对路径字符串，例如 `"./config/monitors.json"`。监视器是 [实验性组件](#experimental-components)。

**必需字段：**

| 字段            | 描述                                     |
| :------------ | :------------------------------------- |
| `name`        | 在插件中唯一的标识符。防止插件重新加载或再次调用 skill 时出现重复进程 |
| `command`     | 在会话工作目录中作为持久后台进程运行的 shell 命令           |
| `description` | 正在监视的内容的简短摘要。显示在任务面板和通知摘要中             |

**可选字段：**

| 字段     | 描述                                                                                                    |
| :----- | :---------------------------------------------------------------------------------------------------- |
| `when` | 控制监视器何时启动。`"always"` 在会话启动和插件重新加载时启动它，这是默认值。`"on-skill-invoke:<skill-name>"` 在第一次调度此插件中的命名 skill 时启动它 |

`command` 值支持 [路径替换](#environment-variables) `${CLAUDE_PLUGIN_ROOT}`、`${CLAUDE_PLUGIN_DATA}` 和 `${CLAUDE_PROJECT_DIR}`，以及环境中的任何 `${ENV_VAR}`。如果脚本需要从插件自己的目录运行，请在命令前加上 `cd "${CLAUDE_PLUGIN_ROOT}" && `。

监视器 `command` 不能引用 [`${user_config.*}`](#user-configuration) 值。命令通过 shell 运行，因此 Claude Code 会拒绝监视器并显示 [错误](/docs/zh-CN/errors#plugin-command-references-user-config)，而不是替换该值。监视器进程不会接收 `CLAUDE_PLUGIN_OPTION_<KEY>` 环境变量，因此让监视器脚本从它拥有的配置文件中读取该值。

如果您在会话中途禁用插件，Claude Code 不会停止已在运行的监视器；它们在会话结束时停止。

<h3 id="themes">
  Themes
</h3>

插件可以提供颜色主题，这些主题与内置预设和用户的本地主题一起显示在 `/theme` 中。主题是 `themes/` 中的 JSON 文件，具有 `base` 预设和稀疏的 `overrides` 颜色令牌映射。主题是 [实验性组件](#experimental-components)。

```json theme={null}
{
  "name": "Dracula",
  "base": "dark",
  "overrides": {
    "claude": "#bd93f9",
    "error": "#ff5555",
    "success": "#50fa7b"
  }
}
```

当用户选择插件主题时，Claude Code 会在其配置中保存 `custom:<plugin-name>:<slug>`。插件主题是只读的：当用户在 `/theme` 中按 `Ctrl+E` 时，Claude Code 会将其复制到 `~/.claude/themes/` 中，以便他们可以编辑副本。

***

<h2 id="plugin-installation-scopes">
  Plugin 安装作用域
</h2>

当你安装一个 plugin 时，你选择一个**作用域**来确定 plugin 在哪里可用以及谁可以使用它：

| 作用域       | 设置文件                                        | 用例                                               |
| :-------- | :------------------------------------------ | :----------------------------------------------- |
| `user`    | `~/.claude/settings.json`                   | 在所有项目中可用的个人 plugins（默认）                          |
| `project` | `.claude/settings.json`                     | 通过版本控制共享的团队 plugins                              |
| `local`   | `.claude/settings.local.json`               | 项目特定的 plugins，当 Claude Code 保存设置到其中时被 gitignored |
| `managed` | [Managed settings](/docs/zh-CN/managed-settings) | 托管 plugins（只读，仅更新）                               |

Plugins 使用与其他 Claude Code 配置相同的作用域系统。有关安装说明和作用域标志，请参阅 [Install plugins](/docs/zh-CN/discover-plugins#install-plugins)。有关作用域的完整说明，请参阅 [Configuration scopes](/docs/zh-CN/settings#where-settings-live)。

***

<h2 id="skills-directory-plugins">
  Skills-directory plugins
</h2>

任何 skills 目录下包含 `.claude-plugin/plugin.json` 清单的文件夹都会在下一个会话中作为名为 `<name>@skills-dir` 的 plugin 加载，无需 marketplace，也无需安装步骤。使用 [`plugin init`](#plugin-init) 来搭建一个。与复制的 marketplace 安装不同，该 plugin 是在原地发现的，而不是复制到 plugin 缓存中。

A skills directory tree supports three distinct things:

| What you have                                 | What it is                                               |
| :-------------------------------------------- | :------------------------------------------------------- |
| `<skills-dir>/foo/SKILL.md` with no manifest  | 一个名为 `foo` 的普通 [skill](/docs/zh-CN/skills)                    |
| `<skills-dir>/foo/.claude-plugin/plugin.json` | 一个 plugin `foo@skills-dir`，可以捆绑自己的 skills、agents、hooks 等 |
| `<plugin>/skills/bar/SKILL.md`                | 一个 skill `bar`，打包在 plugin 内部                             |

<h3 id="choose-where-the-plugin-loads-from">
  选择 plugin 从哪里加载
</h3>

| Skills directory        | Scope    | Loads                                                                                    |
| :---------------------- | :------- | :--------------------------------------------------------------------------------------- |
| `~/.claude/skills/`     | personal | 在每个项目中加载，因为该位置仅属于你                                                                       |
| `<cwd>/.claude/skills/` | project  | 仅在你接受该文件夹的工作区 [trust dialog](/docs/zh-CN/permissions#what-runs-before-you-trust-a-folder) 后加载 |

项目范围的 plugin 被检入到仓库中，并到达克隆它的每个协作者。因为该内容来自仓库而不是来自你，它仅在与 `.claude/settings.json` 中的项目允许规则相同的信任门控后加载，所以信任父文件夹或使用 `-p` 运行是不够的，运行代码的组件受到进一步限制：

* 它声明的 MCP servers 会经过与项目 `.mcp.json` 相同的 [per-server approval](/docs/zh-CN/mcp)
* LSP servers 仅在你信任工作区后启动
* [Background monitors](#monitors) 不加载

Personal-scope plugins 没有这些限制。

<Warning>
  Project-scope `@skills-dir` plugins 仅从会话的 [primary working directory](/docs/zh-CN/permissions#working-directories) 的 `.claude/skills/` 加载。它们不会像普通 skills 和 commands 那样 [walk up to the repository root](/docs/zh-CN/skills#discovery-from-parent-and-nested-directories)，所以从子目录启动会错过位于仓库根目录的 plugin。从仓库根目录启动，或在 v2.1.246 或更高版本上 [使用 `/cd` 将会话移动到那里](/docs/zh-CN/permissions#move-the-session-to-another-directory)。
</Warning>

<h3 id="edit-reload-and-disable-a-skills-directory-plugin">
  编辑、重新加载和禁用 skills-directory plugin
</h3>

你对 skill 的 `SKILL.md` 所做的更改会立即在当前会话中生效。对 plugin 的其他组件（如 `hooks/`、`.mcp.json`、`agents/` 和 `output-styles/`）的更改则不会。运行 `/reload-plugins` 或重启 Claude Code 来获取这些更改。参见 [Live change detection](/docs/zh-CN/skills#live-change-detection)。

要停止加载 skills-directory plugin，删除其文件夹或按名称禁用它。没有 `uninstall` 步骤，因为没有从 marketplace 安装任何东西。

```bash theme={null}
claude plugin disable my-tool@skills-dir
```

***

<h2 id="synced-plugins">
  从 claude.ai 同步的插件
</h2>

在 [Cowork](https://claude.com/product/cowork) 和[云会话](/docs/zh-CN/cloud-environments#what-carries-over-from-your-setup)中，Claude Code 会将为你的 claude.ai 账户启用的插件下载到会话自身环境中的 `~/.claude/plugins/synced/` 目录，并将每个插件加载为 `<name>@synced`，没有 marketplace 和没有安装记录。Claude Code 不会在你在自己的终端中启动的会话中加载它们。在该 Cowork 或云环境中，`claude plugin list` 会在 `Synced from claude.ai` 标题下显示下载的副本。在 v2.1.239 之前，Claude Code 将这些插件加载为 `<name>@inline`，这是 `--plugin-dir` 插件使用的身份。

通过 `claude plugin list` 打印的 `<name>@synced` ID 来管理同步的插件：

* **关闭一个插件**：在同步会话中，运行 `claude plugin disable <name>@synced`，或要求 Claude 运行它。Claude Code 会将该选择保存为该环境的用户级 [`enabledPlugins`](/docs/zh-CN/settings-reference#enabledplugins) 中的 `"<name>@synced": false`。要重新打开该插件，在同一会话中运行 `claude plugin enable <name>@synced`。要将插件排除在每个同步会话之外，[为你的 claude.ai 账户关闭它](/docs/zh-CN/desktop#extend-claude-code)。要将其排除在一个项目的每个环境中的同步会话之外，在该项目的已提交 `.claude/settings.json` 中的 `enabledPlugins` 下设置 `"<name>@synced": false`。
* **在 claude.ai 上管理插件本身**：`claude plugin install`、`update` 和 `uninstall` 不适用于同步的插件。要删除一个，为你的 claude.ai 账户关闭该插件；下一个同步会话将在没有它的情况下启动。

当来自任何其他来源的启用插件（例如 marketplace 安装、[skills-directory 插件](#skills-directory-plugins)或 `--plugin-dir` 插件）与同步插件的名称匹配时，Claude Code 会加载该插件并报告同步副本未加载。要改用 claude.ai 副本，请禁用你自己的副本。在 v2.1.239 之前，Claude Code 会加载同步副本而不是同名的 marketplace 安装。

***

<h2 id="plugin-manifest-schema">
  Plugin manifest schema
</h2>

`.claude-plugin/plugin.json` 文件定义了你的 plugin 的元数据和配置。

manifest 是可选的。如果省略，Claude Code 会在[默认位置](#file-locations-reference)自动发现组件，并从目录名称派生 plugin 名称。当你需要提供元数据或自定义组件路径时，使用 manifest。

<h3 id="complete-schema">
  Complete schema
</h3>

```json theme={null}
{
  "name": "plugin-name",
  "displayName": "Plugin Name",
  "version": "1.2.0",
  "description": "Brief plugin description",
  "author": {
    "name": "Author Name",
    "email": "author@example.com",
    "url": "https://github.com/author"
  },
  "homepage": "https://docs.example.com/plugin",
  "repository": "https://github.com/author/plugin",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"],
  "metadata": { "catalogId": "cat-123", "tier": "pro" },
  "skills": "./custom/skills/",
  "commands": ["./custom/commands/special.md"],
  "agents": ["./custom/agents/reviewer.md"],
  "hooks": "./config/hooks.json",
  "mcpServers": "./mcp-config.json",
  "outputStyles": "./styles/",
  "lspServers": "./.lsp.json",
  "experimental": {
    "themes": "./themes/",
    "monitors": "./monitors.json"
  },
  "dependencies": [
    "helper-lib",
    { "name": "secrets-vault", "version": "~2.1.0" }
  ]
}
```

<h3 id="required-fields">
  必需字段
</h3>

如果你包含 manifest，`name` 是唯一必需的字段。

| 字段     | 类型     | 描述                                                                                                                                                                         | 示例                   |
| :----- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------- |
| `name` | string | 唯一标识符，采用 kebab-case，不包含空格、控制字符或双向格式化字符。当[marketplace 条目](/docs/zh-CN/plugin-marketplaces#plugin-entries)以不同的名称列出 plugin 时，marketplace 条目名称是 `enabledPlugins` 键和 `/plugin` 使用的名称 | `"deployment-tools"` |

此名称用于命名空间组件。例如，在 UI 中，名为 `plugin-dev` 的 plugin 的 agent `agent-creator` 将显示为 `plugin-dev:agent-creator`。

<h3 id="unrecognized-fields">
  无法识别的字段
</h3>

Claude Code 忽略它不识别的顶级字段。你可以在 `plugin.json` 中保留来自另一个生态系统的元数据，plugin 仍然会加载。这使得维护一个 manifest 作为 VS Code 或 Cursor 扩展 manifest、npm `package.json` 或 MCPB/DXT bundle manifest 变得实用。

`claude plugin validate` 将无法识别的字段报告为警告，而不是错误。如果一个字段与识别的字段相差一两个字符，警告会建议可能的预期名称。仅具有无法识别字段警告的 plugin 仍然通过验证并在运行时加载。

Claude Code 如何处理值类型错误的识别字段取决于该字段：

* **大多数字段**：plugin 无法加载。例如，`keywords` 值是字符串而不是数组是加载错误，`claude plugin validate` 会将其报告为错误。
* **`experimental` 和 `metadata`**：Claude Code 忽略非对象值，`claude plugin validate` 报告警告。

传递 `--strict` 以将警告视为错误。在 CI 中使用它来捕获拼写错误的字段名称或来自另一个工具的 manifest 中遗留的字段，然后再发布，即使 plugin 在运行时会加载。

```bash theme={null}
claude plugin validate ./my-plugin --strict
```

<h3 id="metadata-fields">
  元数据字段
</h3>

| 字段               | 类型      | 描述                                                                                                                                                                                                                                 | 示例                                                                |
| :--------------- | :------ | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------- |
| `$schema`        | string  | JSON Schema URL，用于编辑器自动完成和验证。Claude Code 在加载时忽略此字段。                                                                                                                                                                                | `"https://json.schemastore.org/claude-code-plugin-manifest.json"` |
| `displayName`    | string  | 在 `/plugin` 选择器和其他 UI 表面中显示的人类可读名称。对于 marketplace 安装的 plugin，[marketplace 条目](/docs/zh-CN/plugin-marketplaces#optional-plugin-fields)上的 `displayName` 优先于此值。当两个地方都未设置显示名称时，用户会看到 `name`。与 `name` 不同，可以包含空格和任何大小写。不用于命名空间或查找。            | `"Deployment Tools"`                                              |
| `version`        | string  | 可选。语义版本。设置此项会将 plugin 固定到该版本字符串，因此用户仅在你提升版本时才会收到更新，除了[`command` 源](/docs/zh-CN/plugin-marketplaces#command-sources)外；请参阅[版本管理](#version-management)。如果也在 marketplace 条目中设置，`plugin.json` 优先。如果省略，版本来自[版本管理](#version-management)中的下一个源。 | `"2.1.0"`                                                         |
| `description`    | string  | plugin 用途的简要说明                                                                                                                                                                                                                     | `"Deployment automation tools"`                                   |
| `author`         | object  | 作者信息                                                                                                                                                                                                                               | `{"name": "Dev Team", "email": "dev@company.com"}`                |
| `homepage`       | string  | 文档 URL                                                                                                                                                                                                                             | `"https://docs.example.com"`                                      |
| `repository`     | string  | 源代码 URL                                                                                                                                                                                                                            | `"https://github.com/user/plugin"`                                |
| `license`        | string  | 许可证标识符                                                                                                                                                                                                                             | `"MIT"`、`"Apache-2.0"`                                            |
| `keywords`       | array   | 发现标签                                                                                                                                                                                                                               | `["deployment", "ci-cd"]`                                         |
| `metadata`       | object  | 自由格式对象，用于你自己的数据，例如权利或目录字段。Claude Code 不读取它，因此值永远不会影响 plugin 行为。Claude Code 忽略非对象值，`claude plugin validate` 将其报告为警告。在 v2.1.222 之前，Claude Code 将该键视为[无法识别的字段](#unrecognized-fields)。                                                 | `{"catalogId": "cat-123"}`                                        |
| `defaultEnabled` | boolean | 当用户未设置 plugin 状态时，plugin 是否以启用状态启动。默认为 `true`。请参阅[默认启用](#default-enablement)。                                                                                                                                                      | `false`                                                           |

<h3 id="default-enablement">
  默认启用
</h3>

在 `plugin.json` 中设置 `defaultEnabled: false` 以发布已禁用安装的 plugin。用户使用 `claude plugin enable <plugin>` 或 `/plugin` 界面将其打开。对于添加成本或用户应该选择加入的范围的 plugin 使用此选项，例如连接到外部服务的 plugin。

`defaultEnabled` 是当没有其他因素决定 plugin 状态时的后备。两件事优先于它：

* **用户的设置**：任何设置范围内 `enabledPlugins` 中的 plugin 条目。一旦写入，它会在 plugin 更新和重新安装中持续存在，因此在后续版本中更改 `defaultEnabled` 不会翻转现有用户。
* **依赖项要求**：当 plugin 被另一个活跃的 plugin 需要时，Claude Code 在安装或启用时为其写入 `true`。这给了它一个显式设置，所以它自己的默认值不再适用。请参阅[启用或禁用具有依赖项的 plugin](/docs/zh-CN/plugin-dependencies#enable-or-disable-a-plugin-with-dependencies)。

同一字段可以出现在 plugin 的 marketplace 条目中，其中它优先于 `plugin.json` 中的值。请参阅[可选 plugin 字段](/docs/zh-CN/plugin-marketplaces#optional-plugin-fields)。

<h3 id="component-path-fields">
  组件路径字段
</h3>

| 字段                      | 类型                    | 描述                                                                                                            | 示例                                                   |
| :---------------------- | :-------------------- | :------------------------------------------------------------------------------------------------------------ | :--------------------------------------------------- |
| `skills`                | string\|array         | 包含 `<name>/SKILL.md` 的自定义 skill 目录。添加到默认 `skills/` 扫描。请参阅[路径行为规则](#path-behavior-rules)了解 marketplace-root 异常 | `"./custom/skills/"`                                 |
| `commands`              | string\|array         | 自定义平面 `.md` skill 文件或目录（替换默认 `commands/`）                                                                     | `"./custom/cmd.md"` 或 `["./cmd1.md"]`                |
| `agents`                | string\|array         | 自定义 agent 文件（替换默认 `agents/`）                                                                                  | `"./custom/agents/reviewer.md"`                      |
| `workflows`             | string\|array         | 自定义[workflow](/docs/zh-CN/workflows) 脚本文件或目录（替换默认 `workflows/`）                                                    | `"./custom/workflows/"`                              |
| `hooks`                 | string\|array\|object | Hook 配置路径或内联配置                                                                                                | `"./my-extra-hooks.json"`                            |
| `mcpServers`            | string\|array\|object | MCP 配置路径或内联配置                                                                                                 | `"./my-extra-mcp-config.json"`                       |
| `outputStyles`          | string\|array         | 自定义输出样式文件/目录（替换默认 `output-styles/`）                                                                           | `"./styles/"`                                        |
| `lspServers`            | string\|array\|object | [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) 配置，用于代码智能（转到定义、查找引用等）       | `"./.lsp.json"`                                      |
| `experimental.themes`   | string\|array         | 颜色主题文件/目录（替换默认 `themes/`）。请参阅[主题](#themes)                                                                    | `"./themes/"`                                        |
| `experimental.monitors` | string\|array         | 后台[Monitor](/docs/zh-CN/tools-reference#monitor-tool) 配置，在 plugin 活跃时自动启动。请参阅[监视器](#monitors)                      | `"./monitors.json"`                                  |
| `userConfig`            | object                | 在启用时提示的用户可配置值。请参阅[用户配置](#user-configuration)                                                                  | 见下文                                                  |
| `channels`              | array                 | 消息注入的频道声明（Telegram、Slack、Discord 风格）。请参阅[频道](#channels)                                                       | 见下文                                                  |
| `dependencies`          | array                 | 此 plugin 需要的其他 plugin，可选择带有 semver 版本约束。请参阅[约束 plugin 依赖项版本](/docs/zh-CN/plugin-dependencies)                      | `[{ "name": "secrets-vault", "version": "~2.1.0" }]` |

<h3 id="experimental-components">
  实验性组件
</h3>

`experimental` 键下的组件 `themes` 和 `monitors` 具有在版本之间可能会改变的 manifest schema，同时它们稳定下来。你声明它们的位置是一个单独的迁移：顶级仍然有效，`claude plugin validate` 发出警告，未来版本将需要 `experimental.*`。

<h3 id="user-configuration">
  用户配置
</h3>

`userConfig` 字段声明当 plugin 启用时 Claude Code 提示用户的值。使用此选项而不是要求用户手动编辑 `settings.json`。

```json theme={null}
{
  "userConfig": {
    "api_endpoint": {
      "type": "string",
      "title": "API endpoint",
      "description": "Your team's API endpoint"
    },
    "api_token": {
      "type": "string",
      "title": "API token",
      "description": "API authentication token",
      "sensitive": true
    }
  }
}
```

键必须是有效的标识符。每个选项支持这些字段：

| 字段            | 必需 | 描述                                                  |
| :------------ | :- | :-------------------------------------------------- |
| `type`        | 是  | `string`、`number`、`boolean`、`directory` 或 `file` 之一 |
| `title`       | 是  | 在配置对话框中显示的标签                                        |
| `description` | 是  | 在字段下方显示的帮助文本                                        |
| `sensitive`   | 否  | 如果为 `true`，掩盖输入并将值存储在安全存储中而不是 `settings.json`       |
| `required`    | 否  | 如果为 `true`，当字段为空时验证失败                               |
| `default`     | 否  | 当用户未提供任何内容时使用的值                                     |
| `multiple`    | 否  | 对于 `string` 类型，允许字符串数组                              |
| `min` / `max` | 否  | `number` 类型的边界                                      |

每个值都可用于在 MCP 和 LSP 服务器配置以及 hook 命令中作为 `${user_config.KEY}` 进行替换。非敏感值也可以在 skill 和 agent 内容中替换。所有值都作为 `CLAUDE_PLUGIN_OPTION_<KEY>` 环境变量导出到 hook 进程，其中 `<KEY>` 是选项键的大写形式。

在 shell 中运行的字段拒绝 `${user_config.*}`：将配置的值替换到 shell 命令中会让 shell 运行该值包含的任何内容，因此组件失败并出现[错误](/docs/zh-CN/errors#plugin-command-references-user-config)。每个被拒绝的字段都有一种替代方式来传递值：

| 被拒绝的字段                                                                          | 如何传递值                                                                                                      |
| :------------------------------------------------------------------------------ | :--------------------------------------------------------------------------------------------------------- |
| Shell 形式的 hook 命令                                                               | 使用带有 `args` 的 [exec 形式](/docs/zh-CN/hooks#exec-form-and-shell-form)，或从 hook 的环境中读取 `CLAUDE_PLUGIN_OPTION_<KEY>` |
| [Monitor](#monitors) 命令                                                         | 从脚本中的配置文件读取值                                                                                               |
| MCP [`headersHelper`](/docs/zh-CN/mcp#use-dynamic-headers-for-custom-authentication) | 从脚本中的配置文件读取值                                                                                               |

在 v2.1.207 之前，这些字段替换了 `${user_config.KEY}` 值；更新依赖此功能的 plugin。

非敏感值存储在用户 `settings.json` 中 [`pluginConfigs`](/docs/zh-CN/settings-reference#pluginconfigs) 键下，作为 `pluginConfigs[<plugin-id>].options`。

在 macOS 上，Claude Code 将敏感值存储在 macOS Keychain 中，当 Keychain 拒绝写入时回退到 `~/.claude/.credentials.json`。在没有支持的 keychain 的平台上，它将它们存储在 `~/.claude/.credentials.json` 中。Keychain 存储与 OAuth 令牌共享，总限制约为 2 KB，因此保持敏感值较小。

Claude Code 仅从三个设置源读取所有 `pluginConfigs` 值：

* **用户设置**：`~/.claude/settings.json`，启用时提示写入的文件
* **`--settings`**：CLI 标志或 SDK 内联设置
* **托管设置**：[组织控制的策略](/docs/zh-CN/permissions#managed-settings)

当多个源设置相同的键时，托管设置优先，然后是 `--settings`，然后是用户设置。你可以从此列表中删除的唯一源是用户设置：传递不包含 `user` 的 [`--setting-sources`](/docs/zh-CN/cli-reference#cli-flags)，Claude Code 会跳过它们。托管设置和 `--settings` 保持你传递的任何内容。SDK 的 [`settingSources`](/docs/zh-CN/agent-sdk/claude-code-features#what-settingsources-does-not-control) 选项设置相同的列表。

项目的 `.claude/settings.json` 或 `.claude/settings.local.json` 中的条目被忽略。两个文件都位于工作区中，因此克隆的存储库可以在那里提供值，这些值会流入 plugin hook 命令、MCP 服务器配置、LSP 命令和监视器命令。在 v2.1.207 之前，这些条目被读取。限制特定于 `pluginConfigs`：[`enabledPlugins`](/docs/zh-CN/settings-reference#enabledplugins) 仍然遵守项目和本地设置。

<h3 id="channels">
  频道
</h3>

`channels` 字段让 plugin 声明一个或多个消息频道，将内容注入到对话中。每个频道绑定到 plugin 提供的 MCP 服务器。

```json theme={null}
{
  "channels": [
    {
      "server": "telegram",
      "userConfig": {
        "bot_token": {
          "type": "string",
          "title": "Bot token",
          "description": "Telegram bot token",
          "sensitive": true
        },
        "owner_id": {
          "type": "string",
          "title": "Owner ID",
          "description": "Your Telegram user ID"
        }
      }
    }
  ]
}
```

`server` 字段是必需的，必须与 plugin 的 `mcpServers` 中的键匹配。可选的每个频道 `userConfig` 使用与顶级字段相同的 schema，让 plugin 在启用时提示输入机器人令牌或所有者 ID。

<h3 id="path-behavior-rules">
  路径行为规则
</h3>

自定义路径是替换还是扩展 plugin 的默认目录取决于该字段：

* **替换默认值**：`commands`、`agents`、`workflows`、`outputStyles`、`experimental.themes`、`experimental.monitors`。例如，当 manifest 指定 `commands` 时，不会扫描默认 `commands/` 目录。要保留默认值并添加更多，请明确列出：`"commands": ["./commands/", "./extras/"]`
* **添加到默认值**：`skills`。默认 `skills/` 目录始终被扫描，`skills` 中列出的目录与其一起加载。异常：对于[源解析为 marketplace 根的 marketplace 条目](/docs/zh-CN/plugin-marketplaces#advanced-plugin-entries)，声明特定子目录会替换默认 `skills/` 扫描
* **自己的合并规则**：[hooks](#hooks)、[MCP 服务器](#mcp-servers) 和 [LSP 服务器](#lsp-servers)。请参阅每个部分了解多个源如何组合

当 plugin 同时具有默认文件夹和匹配的 manifest 键时，Claude Code 在 `claude plugin list` 和 `/plugin` 详细视图中警告被忽略的文件夹。plugin 仍然使用 manifest 路径加载。当 manifest 键指向默认文件夹时，Claude Code 不会发出警告，例如 `"commands": ["./commands/deploy.md"]`，因为该路径明确命名了文件夹。

对于所有路径字段：

* 所有路径必须相对于 plugin 根目录并以 `./` 开头，除了 `skills` 字段也接受 `"."`
  * `"."` 和 `"./"` 都表示 plugin 根目录本身
  * 在 v2.1.221 之前，`"."` 无法通过 manifest 验证，plugin 无法加载，因此使用 `"./"` 来支持早期版本
* 来自自定义路径的组件使用相同的命名和命名空间规则
* 可以将多个路径指定为数组
* skill 路径可以指向直接包含 `SKILL.md` 的目录，例如 `"skills": ["."]` 用于 plugin 根目录
  * Claude Code 从 `SKILL.md` 中的 frontmatter `name` 字段获取 skill 的调用名称，因此无论安装目录的名称如何，名称都保持稳定
  * 如果 frontmatter 中未设置 `name`，Claude Code 会回退到目录基名

具有根目录中的 `SKILL.md`、没有 `skills/` 子目录且没有 `skills` manifest 字段的 plugin 会自动作为单 skill plugin 加载。对于此布局，你不需要在 `plugin.json` 中设置 `"skills": ["./"]`。

**路径示例**：

```json theme={null}
{
  "commands": [
    "./specialized/deploy.md",
    "./utilities/batch-process.md"
  ],
  "agents": [
    "./custom-agents/reviewer.md",
    "./custom-agents/tester.md"
  ]
}
```

<h3 id="environment-variables">
  环境变量
</h3>

Claude Code 提供三个变量用于引用路径：

| 变量                      | 解析为                                                        | 用途                                               |
| :---------------------- | :--------------------------------------------------------- | :----------------------------------------------- |
| `${CLAUDE_PLUGIN_ROOT}` | plugin 安装目录的绝对路径                                           | 与 plugin 捆绑的脚本、二进制文件和配置文件                        |
| `${CLAUDE_PLUGIN_DATA}` | [持久目录](#persistent-data-directory)，在首次引用时创建，在 plugin 更新中存活 | 已安装的依赖项，例如 `node_modules` 或 Python 虚拟环境、生成的代码和缓存 |
| `${CLAUDE_PROJECT_DIR}` | 项目根目录                                                      | 项目本地脚本和配置文件                                      |

所有三个都作为环境变量导出到 hook 进程以及 MCP 和 LSP 服务器子进程。哪些字段内联替换它们取决于 plugin 组件：

| Plugin 组件                 | 占位符解析的字段                                 |
| :------------------------ | :--------------------------------------- |
| Skill 和 agent 内容          | 占位符出现的任何地方                               |
| Hook 和 monitor 命令         | 占位符出现的任何地方                               |
| MCP `stdio` 服务器           | `command`、`args`、`env`                   |
| MCP `http`、`sse`、`ws` 服务器 | `url`、`headers`、`headersHelper`          |
| LSP 服务器                   | `command`、`args`、`env`、`workspaceFolder` |

在 hook 命令中，使用带有 `args` 的 [exec 形式](/docs/zh-CN/hooks#exec-form-and-shell-form)，以便每个路径作为一个参数传递，无需引用。在 shell 形式的 hooks 和 monitor 命令中，用双引号包装变量，如 `"${CLAUDE_PROJECT_DIR}/scripts/server.sh"`。此 shell 形式的 hook 运行与 plugin 捆绑的脚本：

```json theme={null}
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}\"/scripts/process.sh"
          }
        ]
      }
    ]
  }
}
```

`${CLAUDE_PLUGIN_ROOT}` 在 plugin 更新时改变。前一个版本的目录在更新后的宽限期内保留在磁盘上，但将其视为临时的，不要在那里写入状态。请参阅 [plugin 缓存](#plugin-caching-and-file-resolution)了解清理语义。

当 plugin 在会话中期更新时，hook 命令、monitors、MCP 服务器和 LSP 服务器继续使用前一个版本的路径。运行 `/reload-plugins` 将 hooks、MCP 服务器和 LSP 服务器切换到新路径；monitors 需要会话重启。在没有交互式终端的会话中，重新加载会将 plugin MCP 服务器保留在旧路径上，直到下一个会话。

对于具有 `command` 源的 plugin，Claude Code [可以重新加载 plugin 本身](/docs/zh-CN/plugin-marketplaces#when-claude-code-re-runs-the-command)。

MCP 服务器也可以调用 `roots/list` 请求在运行时读取会话的工作目录。请参阅[`roots/list` 返回的内容以及 Claude Code 何时通知服务器更改](/docs/zh-CN/mcp#option-3-add-a-local-stdio-server)。

<h4 id="persistent-data-directory">
  持久数据目录
</h4>

`${CLAUDE_PLUGIN_DATA}` 目录解析为 `~/.claude/plugins/data/{id}/`，其中 `{id}` 是 plugin 标识符，其中 `a-z`、`A-Z`、`0-9`、`_` 和 `-` 之外的字符被替换为 `-`。对于作为 `formatter@my-marketplace` 安装的 plugin，目录是 `~/.claude/plugins/data/formatter-my-marketplace/`。

常见用途是一次安装语言依赖项并在会话和 plugin 更新中重复使用它们。将其用于 Python 依赖项、使用 Yarn 或 pnpm 锁定的依赖项以及生命周期脚本必须运行的包。对于 marketplace 安装的 plugin，你可能根本不需要它：Claude Code 在缓存 plugin 时自动安装符合条件的 [Node.js 包依赖项](#node-js-package-dependencies)。

因为数据目录的生命周期超过任何单个 plugin 版本，仅检查目录存在性无法检测到更新何时更改 plugin 的依赖项 manifest。推荐的模式是将捆绑的 manifest 与数据目录中的副本进行比较，并在它们不同时重新安装。

此 `SessionStart` hook 在首次运行时安装 `node_modules`，并在 plugin 更新包含更改的 `package.json` 时再次安装：

```json theme={null}
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "diff -q \"${CLAUDE_PLUGIN_ROOT}/package.json\" \"${CLAUDE_PLUGIN_DATA}/package.json\" >/dev/null 2>&1 || (cd \"${CLAUDE_PLUGIN_DATA}\" && cp \"${CLAUDE_PLUGIN_ROOT}/package.json\" . && npm install) || rm -f \"${CLAUDE_PLUGIN_DATA}/package.json\""
          }
        ]
      }
    ]
  }
}
```

当存储的副本缺失或与捆绑的副本不同时，`diff` 退出非零，涵盖首次运行和依赖项更改更新。如果 `npm install` 失败，尾部 `rm` 会删除复制的 manifest，以便下一个会话重试。

捆绑在 `${CLAUDE_PLUGIN_ROOT}` 中的脚本可以针对持久化的 `node_modules` 运行：

```json theme={null}
{
  "mcpServers": {
    "routines": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server.js"],
      "env": {
        "NODE_PATH": "${CLAUDE_PLUGIN_DATA}/node_modules"
      }
    }
  }
}
```

当你从最后一个安装 plugin 的范围卸载 plugin 时，数据目录会自动删除。`/plugin` 界面显示目录大小并在删除前提示。CLI 默认删除；传递 [`--keep-data`](#plugin-uninstall) 以保留它。

***

<h2 id="plugin-caching-and-file-resolution">
  Plugin 缓存和文件解析
</h2>

Plugin 可以通过以下两种方式指定：

* 通过 `claude --plugin-dir` 或 `claude --plugin-url`，在会话期间使用。
* 通过 marketplace，为未来的会话安装。

出于安全和验证目的，Claude Code 将 *marketplace* plugin 复制到用户的本地 **plugin 缓存**（`~/.claude/plugins/cache`）中，而不是就地使用它们，除了 [link 模式下的 `command` 源](/docs/zh-CN/plugin-marketplaces#copy-mode-and-link-mode)，Claude Code 通过缓存条目中的链接就地使用这些源。

对于复制的 plugin，每个已安装的版本都是缓存中的一个单独目录，按 marketplace 和 plugin 分组，并以解析的版本命名，包含 plugin 文件和 [Node.js 包依赖](#node-js-package-dependencies) 的自己的副本。从 [release tag](/docs/zh-CN/plugin-dependencies#tag-plugin-releases-for-version-resolution) 解析的依赖会获得一个带有 commit-SHA 后缀的目录名。

当你更新或卸载 plugin 时，Claude Code 会将之前的版本目录标记为孤立，并在大约 14 天后的后台扫描中将其删除。宽限期允许已加载旧版本的并发 Claude Code 会话继续运行而不出错。Claude Code 仅在至少安装了一个 plugin 时才运行扫描；在卸载最后一个 plugin 后，孤立目录会保留在磁盘上，直到你再次安装 plugin。

Claude Code 仅在 plugin 或 marketplace 文件夹不再包含任何目录或符号链接时才将其从缓存中删除。如果你将开发检出符号链接到缓存中作为 plugin 的版本条目，Claude Code 永远不会将该链接标记为孤立，也永远不会删除它或保存它的文件夹。Claude Code 也永远不会在链接的检出中写入其版本跟踪文件。

Claude 的 Glob 和 Grep 工具在搜索期间跳过孤立的版本目录，因此文件结果不包括过时的 plugin 代码。

<h3 id="node-js-package-dependencies">
  Node.js 包依赖
</h3>

当 Claude Code 将 plugin 复制到缓存中时，它也会在那里安装 plugin 的 Node.js 包依赖，以便 plugin 的 hooks 和 MCP 服务器可以加载它们。本节涵盖 plugin 在其自己的 `package.json` 中声明的 npm 和 Bun 包。对于依赖其他 plugin 的 plugin，请参阅 [plugin 依赖版本](/docs/zh-CN/plugin-dependencies)。

Claude Code 在每次创建复制的版本目录时在其中运行安装：当你安装 plugin 时、当 Claude Code 将 plugin 更新到新版本时，以及在会话启动时当启用的 plugin 尚未缓存时（例如在新机器上）。仅当 plugin 的根目录同时包含 `package.json` 和受支持的 lockfile 时，安装才会运行：

| Lockfile                                    | 命令                                               |
| :------------------------------------------ | :----------------------------------------------- |
| `bun.lock` 或 `bun.lockb`                    | `bun install --frozen-lockfile --ignore-scripts` |
| `npm-shrinkwrap.json` 或 `package-lock.json` | `npm ci --ignore-scripts`                        |

如果 plugin 包含多个这些 lockfile，Claude Code 使用第一个匹配项，按顺序检查：`bun.lock`、`bun.lockb`、`npm-shrinkwrap.json`、`package-lock.json`。Claude Code 跳过 `yarn.lock` 和 `pnpm-lock.yaml`，因为 Yarn 和 pnpm 支持绕过 `--ignore-scripts` 的分辨率时间配置钩子。

为了获得最广泛的覆盖范围，请提供 npm lockfile。Claude Code 从用户的 PATH 运行匹配的 lockfile 的包管理器，如果缺少其他 lockfile，不会回退到它。对于通过 npm 源分发的 plugin，使用 `npm-shrinkwrap.json`；npm 从已发布的包中排除 `package-lock.json`。

Claude Code 对此依赖安装进行了约束，以便 plugin 或其包中的任何代码在安装期间都不会执行，并限制其运行时间：

* **冻结分辨率：** Bun 和 npm 安装 lockfile 精确指定的内容，当 `package.json` 和 lockfile 不一致时失败而不是重新分辨版本。
* **无生命周期脚本：** `--ignore-scripts` 防止 `preinstall`、`install` 和 `postinstall` 脚本运行，因此在这些脚本中构建本机模块的依赖会下载但在此安装期间不会编译。
* **60 秒超时：** Claude Code 停止运行时间较长的安装并将其视为失败。

获取 npm 源 plugin 本身会在此依赖安装运行之前运行启用了生命周期脚本的 `npm install`。

失败或跳过的安装永远不会阻止 plugin。当安装失败或 Claude Code 跳过 yarn 或 pnpm lockfile 时，它会在 [debug 输出](#debugging-commands) 中将原因记录为警告。具有 `package.json` 但没有 lockfile 的 plugin 会被跳过而不记录日志条目。超时的安装可能会在缓存副本中留下部分 `node_modules` 树。

你无法关闭自动安装；没有设置或环境变量可以禁用它。在受限网络中，请参阅 [网络访问要求](/docs/zh-CN/network-config#network-access-requirements) 以了解要允许的主机。

对于自动安装无法提供的依赖，例如需要其生命周期脚本来构建的包、Python 依赖或使用 Yarn 或 pnpm 锁定的 plugin，请从 hook 将它们安装到 [持久数据目录](#persistent-data-directory)。

<h3 id="path-traversal-limitations">
  路径遍历限制
</h3>

Claude Code 不允许 plugin 引用其自己目录之外的文件。它拒绝解析到 plugin 根目录之外的组件路径，无论该路径是在 `plugin.json` 中声明还是在 [marketplace 条目](/docs/zh-CN/plugin-marketplaces#plugin-entries) 中声明。这涵盖指向 plugin 外部的路径（如 `../shared-utils`）和导向 plugin 外部的符号链接，除了 [一个 marketplace 内的链接](#share-files-within-a-marketplace-with-symlinks)。

在 macOS 和 Linux 上，Claude Code 也拒绝包含反斜杠的组件路径，即使该路径保留在 plugin 内。因此，使用反斜杠路径声明的组件仅在 Windows 上加载。使用正斜杠编写组件路径，例如 `./commands/deploy.md`。

当 Claude Code 拒绝路径时，它会报告 [`path escapes plugin directory`](/docs/zh-CN/errors#path-escapes-plugin-directory) 错误，并在没有该组件的情况下加载 plugin。

Claude Code 在安装 plugin 时也不会将 plugin 目录之外的文件复制到缓存中，因此当复制的 plugin 内的脚本读取 plugin 根目录上方的路径时，它也找不到这些文件。

<h3 id="share-files-within-a-marketplace-with-symlinks">
  使用符号链接在 marketplace 内共享文件
</h3>

如果你的 plugin 需要与同一 marketplace 的其他部分共享文件，你可以在 plugin 目录内创建符号链接。当 plugin 被复制到缓存中时，符号链接的处理方式取决于其目标的解析位置：

* **在 plugin 自己的目录内：** 符号链接在缓存中被保留为相对符号链接，因此它在运行时继续解析到复制的目标。
* **在同一 marketplace 内的其他位置：** 符号链接被解引用。目标的内容被复制到缓存中以代替它。这允许元 plugin 的 `skills/` 目录链接到 marketplace 中其他 plugin 定义的技能。
* **在 marketplace 外：** 符号链接出于安全原因被跳过。这防止 plugin 将任意主机文件（如系统路径）拉入缓存。

对于使用 `--plugin-dir` 安装的 plugin、来自本地路径的 plugin 或 来自 [copy 模式下的 `command` 源](/docs/zh-CN/plugin-marketplaces#copy-mode-and-link-mode) 的 plugin，仅保留解析到 plugin 自己目录内的符号链接。所有其他的都被跳过。

以下命令创建从 marketplace plugin 内部到由兄弟 plugin 定义的共享技能的链接。在 Windows 上，从提升的命令提示符使用 `mklink /D` 或启用开发者模式：

```bash theme={null}
ln -s ../../shared-plugin/skills/foo ./skills/foo
```

***

<h2 id="plugin-directory-structure">
  Plugin 目录结构
</h2>

<h3 id="standard-plugin-layout">
  标准 plugin 布局
</h3>

一个完整的 plugin 遵循以下结构：

```text theme={null}
enterprise-plugin/
├── .claude-plugin/           # 元数据目录（可选）
│   └── plugin.json             # plugin 清单
├── skills/                   # Skills
│   ├── code-reviewer/
│   │   └── SKILL.md
│   └── pdf-processor/
│       ├── SKILL.md
│       └── scripts/
├── commands/                 # Skills 作为平面 .md 文件
│   ├── status.md
│   └── logs.md
├── agents/                   # Subagent 定义
│   ├── security-reviewer.md
│   ├── performance-tester.md
│   └── compliance-checker.md
├── workflows/                # Workflow 脚本
│   └── release-audit.js
├── output-styles/            # 输出样式定义
│   └── terse.md
├── themes/                   # 颜色主题定义
│   └── dracula.json
├── monitors/                 # 后台监视器配置
│   └── monitors.json
├── hooks/                    # Hook 配置
│   ├── hooks.json           # 主 hook 配置
│   └── security-hooks.json  # 其他 hooks
├── bin/                      # 添加到 PATH 的 plugin 可执行文件
│   └── my-tool               # 在 Bash tool 中可作为裸命令调用
├── settings.json            # plugin 的默认设置
├── .mcp.json                # MCP 服务器定义
├── .lsp.json                # LSP 服务器配置
├── scripts/                 # Hook 和实用脚本
│   ├── security-scan.sh
│   ├── format-code.py
│   └── deploy.js
├── LICENSE                  # 许可证文件
└── CHANGELOG.md             # 版本历史
```

<Warning>
  `.claude-plugin/` 目录包含 `plugin.json` 文件。所有其他目录（commands/、agents/、skills/、workflows/、output-styles/、themes/、monitors/、hooks/）必须位于 plugin 根目录，而不是在 `.claude-plugin/` 内部。
</Warning>

plugin 根目录中的 `CLAUDE.md` 文件不会作为项目上下文加载。Plugins 通过 skills、agents 和 hooks 而不是 CLAUDE.md 来贡献上下文。要提供加载到 Claude 上下文中的说明，请将其放在 [skill](#skills) 中。

<h3 id="file-locations-reference">
  文件位置参考
</h3>

| 组件            | 默认位置                         | 用途                                                                                                                                                                               |
| :------------ | :--------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **清单**        | `.claude-plugin/plugin.json` | Plugin 元数据和配置（可选）                                                                                                                                                                |
| **Skills**    | `skills/`                    | 具有 `<name>/SKILL.md` 结构的 Skills                                                                                                                                                  |
| **Commands**  | `commands/`                  | 作为平面 Markdown 文件的 Skills。新 plugins 请使用 `skills/`                                                                                                                                 |
| **Agents**    | `agents/`                    | Subagent Markdown 文件                                                                                                                                                             |
| **Workflows** | `workflows/`                 | [Workflow](/docs/zh-CN/workflows) 脚本文件                                                                                                                                                |
| **输出样式**      | `output-styles/`             | 输出样式定义                                                                                                                                                                           |
| **主题**        | `themes/`                    | 颜色主题定义                                                                                                                                                                           |
| **Hooks**     | `hooks/hooks.json`           | Hook 配置                                                                                                                                                                          |
| **MCP 服务器**   | `.mcp.json`                  | MCP 服务器定义                                                                                                                                                                        |
| **LSP 服务器**   | `.lsp.json`                  | 语言服务器配置                                                                                                                                                                          |
| **监视器**       | `monitors/monitors.json`     | 后台监视器配置                                                                                                                                                                          |
| **可执行文件**     | `bin/`                       | 添加到 Bash tool 的 `PATH` 中的可执行文件，在 plugin 启用时可作为裸命令调用。如果您 [通过 claude.ai 组织设置分发 plugin](/docs/zh-CN/plugin-marketplaces#keep-executables-out-of-the-top-level-bin-directory)，则不能在其中包含此目录 |
| **设置**        | `settings.json`              | 启用 plugin 时应用的默认配置。仅支持 [`agent`](/docs/zh-CN/sub-agents) 和 [`subagentStatusLine`](/docs/zh-CN/statusline#subagent-status-lines) 键                                                          |

***

<h2 id="cli-commands-reference">
  CLI 命令参考
</h2>

Claude Code 提供 CLI 命令用于非交互式插件管理，适用于脚本和自动化。

<h3 id="plugin-init">
  plugin init
</h3>

在 `~/.claude/skills/<name>/` 处搭建一个新插件。在下一个 Claude Code 会话中，它会自动加载为 `<name>@skills-dir`，并在 `/plugin` 和 `claude plugin list` 中显示，无需安装步骤。

请参阅 [Skills-directory plugins](#skills-directory-plugins) 了解范围和信任要求。

```bash theme={null}
claude plugin init <name> [options]
```

**参数：**

* `<name>`：插件名称。成为技能命名空间和 `~/.claude/skills/` 下的目录名称，因此不能包含空格或路径分隔符。

**选项：**

| 选项                       | 描述                                                                           | 默认值                     |
| :----------------------- | :--------------------------------------------------------------------------- | :---------------------- |
| `--description <text>`   | 清单描述                                                                         |                         |
| `--author <name>`        | 作者名称                                                                         | `git config user.name`  |
| `--author-email <email>` | 作者电子邮件                                                                       | `git config user.email` |
| `--with <components...>` | 同时搭建组件文件夹。有效值：`skills`、`agents`、`hooks`、`mcp`、`lsp`、`output-style`、`channel` |                         |
| `-f, --force`            | 覆盖目标处现有的 `.claude-plugin/`                                                   |                         |
| `-h, --help`             | 显示命令帮助                                                                       |                         |

**别名：** `new`

每个 `--with` 值都会为该组件添加一个启动文件，准备好编辑：

| 组件             | 搭建内容                                                                                             |
| :------------- | :----------------------------------------------------------------------------------------------- |
| `skills`       | 一个额外的命名空间 `<name>:example` 技能，与默认技能并列                                                            |
| `agents`       | 一个 `agents/` 子代理定义                                                                               |
| `hooks`        | 一个 `hooks/hooks.json`，包含示例事件处理程序                                                                 |
| `mcp`          | 一个 `.mcp.json`，包含 HTTP 和 stdio 服务器示例                                                             |
| `lsp`          | 一个 `.lsp.json` 语言服务器示例                                                                           |
| `output-style` | 一个 `output-styles/<name>.md`，在插件启用时自动应用                                                          |
| `channel`      | 一个基于 MCP 的 [channel](/docs/zh-CN/channels)：一个 stdio 服务器（`server.ts`）、其 `.mcp.json` 和一个 `package.json` |

搭建的插件使用 `@skills-dir` 源而不是市场。管理员可以通过 `strictKnownMarketplaces` 或在 [managed settings](/docs/zh-CN/plugin-marketplaces#managed-marketplace-restrictions) 中添加 `{"source": "skills-dir"}` 到 `blockedMarketplaces` 来阻止此源。当被阻止时，`plugin init` 在写入前失败。

**示例：**

```bash theme={null}
# 搭建最小插件
claude plugin init my-helper

# 搭建带有技能和钩子文件夹的插件
claude plugin init my-helper --with skills hooks

# 覆盖现有搭建
claude plugin init my-helper --force
```

<h3 id="plugin-install">
  plugin install
</h3>

从可用市场安装插件。

```bash theme={null}
claude plugin install <plugin> [options]
```

**参数：**

* `<plugin>`：插件名称或 `plugin-name@marketplace-name` 用于特定市场

**选项：**

| 选项                     | 描述                                                                                                                                                                                                                                                                                                                      | 默认值    |
| :--------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :----- |
| `-s, --scope <scope>`  | 安装范围：`user`、`project` 或 `local`                                                                                                                                                                                                                                                                                         | `user` |
| `--config <key=value>` | 设置插件清单中声明的 [`userConfig`](#user-configuration) 选项。重复该标志以设置多个选项                                                                                                                                                                                                                                                          |        |
| `-y, --yes`            | 接受插件市场声明的命令，无需确认提示：生成具有 [`command` source](/docs/zh-CN/plugin-marketplaces#command-sources) 的插件的命令，或验证存档下载的 [`headersHelper`](/docs/zh-CN/plugin-marketplaces#authenticate-archive-downloads)。接受 `headersHelper` 需要 Claude Code v2.1.238 或更高版本。Claude Code 仍会首先打印命令。当 stdin 或 stdout 不是 TTY 时需要。在 Claude Code 会话内无效，因此从您自己的终端运行命令 |        |
| `-h, --help`           | 显示命令帮助                                                                                                                                                                                                                                                                                                                  |        |

范围决定了已安装插件添加到哪个设置文件。例如，`--scope project` 写入 .claude/settings.json 中的 `enabledPlugins`，使插件对克隆项目存储库的每个人都可用。

**示例：**

```bash theme={null}
# 安装到用户范围（默认）
claude plugin install formatter@my-marketplace

# 安装到项目范围（与团队共享）
claude plugin install formatter@my-marketplace --scope project

# 安装到本地范围（不与团队共享）
claude plugin install formatter@my-marketplace --scope local
```

<h3 id="plugin-uninstall">
  plugin uninstall
</h3>

删除已安装的插件。

```bash theme={null}
claude plugin uninstall <plugin> [options]
```

**参数：**

* `<plugin>`：插件名称或 `plugin-name@marketplace-name`

**选项：**

| 选项                    | 描述                                                            | 默认值    |
| :-------------------- | :------------------------------------------------------------ | :----- |
| `-s, --scope <scope>` | 从范围卸载：`user`、`project` 或 `local`                              | `user` |
| `--keep-data`         | 保留插件的 [persistent data directory](#persistent-data-directory) |        |
| `--prune`             | 同时删除没有其他插件需要的自动安装依赖项。请参阅 [plugin prune](#plugin-prune)        |        |
| `-y, --yes`           | 跳过 `--prune` 确认提示。当 stdin 或 stdout 不是 TTY 时需要                 |        |
| `-h, --help`          | 显示命令帮助                                                        |        |

**别名：** `remove`、`rm`

默认情况下，从最后剩余的范围卸载也会删除插件的 `${CLAUDE_PLUGIN_DATA}` 目录。使用 `--keep-data` 保留它，例如在测试新版本后重新安装时。

<Note>
  当来自不同市场的已安装插件共享一个名称时，`plugin-name@marketplace-name` 形式仅卸载来自命名市场的插件。在 v2.1.212 之前，限定形式可能会匹配并卸载来自不同市场的同名插件。
</Note>

<h3 id="plugin-prune">
  plugin prune
</h3>

删除不再被任何已安装插件需要的自动安装插件依赖项。Claude Code 为满足另一个插件的 [`dependencies`](/docs/zh-CN/plugin-dependencies) 字段而拉入的依赖项会被删除；您直接安装的插件永远不会被触及。

```bash theme={null}
claude plugin prune [options]
```

**选项：**

| 选项                    | 描述                                 | 默认值    |
| :-------------------- | :--------------------------------- | :----- |
| `-s, --scope <scope>` | 在范围处修剪：`user`、`project` 或 `local`  | `user` |
| `--dry-run`           | 列出将被删除的内容而不实际删除                    |        |
| `-y, --yes`           | 跳过确认提示。当 stdin 或 stdout 不是 TTY 时需要 |        |
| `-h, --help`          | 显示命令帮助                             |        |

**别名：** `autoremove`

该命令列出孤立的依赖项并在删除前请求确认。要在一个步骤中删除插件并清理其依赖项，请运行 `claude plugin uninstall <plugin> --prune`。

<h3 id="plugin-enable">
  plugin enable
</h3>

启用已禁用的插件。当目标从市场安装并声明 [dependencies](/docs/zh-CN/plugin-dependencies) 时，Claude Code 在同一范围内以传递方式启用它们。该命令在 [Enable or disable a plugin with dependencies](/docs/zh-CN/plugin-dependencies#enable-or-disable-a-plugin-with-dependencies) 列出的条件下失败。

```bash theme={null}
claude plugin enable <plugin> [options]
```

**参数：**

* `<plugin>`：插件名称或 `plugin-name@marketplace-name`

**选项：**

| 选项                    | 描述                                                        | 默认值  |
| :-------------------- | :-------------------------------------------------------- | :--- |
| `-s, --scope <scope>` | 启用范围：`user`、`project` 或 `local`。省略时，Claude Code 检测安装插件的范围 | 自动检测 |
| `-h, --help`          | 显示命令帮助                                                    |      |

<h3 id="plugin-disable">
  plugin disable
</h3>

禁用插件而不卸载它。当目标从市场安装时，如果另一个启用的插件 [depends on](/docs/zh-CN/plugin-dependencies#enable-or-disable-a-plugin-with-dependencies) 它，该命令会失败。错误消息包含一个链式命令，首先禁用每个依赖项。

```bash theme={null}
claude plugin disable [plugin] [options]
```

**参数：**

* `[plugin]`：插件名称或 `plugin-name@marketplace-name`。使用 `--all` 时可选

**选项：**

| 选项                    | 描述                                                        | 默认值  |
| :-------------------- | :-------------------------------------------------------- | :--- |
| `-a, --all`           | 禁用所有启用的插件。不能与 `--scope` 组合                                |      |
| `-s, --scope <scope>` | 禁用范围：`user`、`project` 或 `local`。省略时，Claude Code 检测安装插件的范围 | 自动检测 |
| `-h, --help`          | 显示命令帮助                                                    |      |

<h3 id="plugin-update">
  plugin update
</h3>

将插件更新到最新版本。

```bash theme={null}
claude plugin update <plugin> [options]
```

**参数：**

* `<plugin>`：插件名称或 `plugin-name@marketplace-name`

**选项：**

| 选项                    | 描述                                                                                                                                                                                                                                                                                                                      | 默认值    |
| :-------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :----- |
| `-s, --scope <scope>` | 更新范围：`user`、`project`、`local` 或 `managed`                                                                                                                                                                                                                                                                               | `user` |
| `-y, --yes`           | 接受插件市场声明的命令，无需确认提示：生成具有 [`command` source](/docs/zh-CN/plugin-marketplaces#command-sources) 的插件的命令，或验证存档下载的 [`headersHelper`](/docs/zh-CN/plugin-marketplaces#authenticate-archive-downloads)。接受 `headersHelper` 需要 Claude Code v2.1.238 或更高版本。Claude Code 仍会首先打印命令。当 stdin 或 stdout 不是 TTY 时需要。在 Claude Code 会话内无效，因此从您自己的终端运行命令 |        |
| `-h, --help`          | 显示命令帮助                                                                                                                                                                                                                                                                                                                  |        |

<Note>
  Claude Code 根据您已安装的插件解析裸插件名称。当来自不同市场的已安装插件共享该名称时，Claude Code 拒绝更新并列出要运行的限定 `plugin-name@marketplace-name` 命令。在 v2.1.246 之前，Claude Code 仅接受限定形式并拒绝裸名称为未找到。
</Note>

***

<h3 id="plugin-list">
  plugin list
</h3>

列出已安装的插件及其版本、源市场和启用状态。

```bash theme={null}
claude plugin list [options]
```

**选项：**

| 选项            | 描述                     | 默认值 |
| :------------ | :--------------------- | :-- |
| `--json`      | 输出为 JSON               |     |
| `--available` | 包括市场中的可用插件。需要 `--json` |     |
| `-h, --help`  | 显示命令帮助                 |     |

在交互式会话中，`/plugin list` 打印类似的列表内联，但仅涵盖市场安装的插件：

* 从技能目录加载的插件出现在 `/plugin` 界面和 `claude plugin list` 中，但不出现在内联 `/plugin list` 输出中。
* 在 Claude Code v2.1.239 或更高版本上，[从 claude.ai 同步的插件](#synced-plugins) 在您在同步会话下载它们的环境中运行 `claude plugin list` 时出现。它们不出现在内联 `/plugin list` 输出中。
* 使用 `--plugin-dir` 或 `--plugin-url` 为会话加载的插件出现在 `/plugin` 界面中，仅当相同标志在子命令前时才出现在 `claude plugin list` 中，如 `claude --plugin-dir <dir> plugin list`。仅标志名称标识其位置，因此裸 `claude plugin list` 无法找到它们，不同于同步插件和技能目录插件，其固定目录 Claude Code 扫描。

交互式形式接受 `--enabled` 或 `--disabled` 以仅显示该状态中的插件，以及 `ls` 作为 `list` 的简写。

<h3 id="plugin-details">
  plugin details
</h3>

显示插件的组件清单和预计令牌成本。输出列出插件贡献的所有组件，分组为 Skills、Agents、Hooks、MCP 服务器和 LSP 服务器，以及它为每个会话添加多少令牌的估计。Skills 组包括 `skills/` 和 `commands/` 条目。

```bash theme={null}
claude plugin details <name>
```

**参数：**

* `<name>`：插件名称或 `plugin-name@marketplace-name`

**选项：**

| 选项           | 描述     | 默认值 |
| :----------- | :----- | :-- |
| `-h, --help` | 显示命令帮助 |     |

输出为每个组件显示两个成本数字：

* **Always-on：** 插件的列表文本（如技能描述、代理描述和命令名称）添加到每个会话的令牌，无论任何组件是否触发。
* **On-invoke：** 组件触发时的成本。按组件显示，而不是作为插件总计，因为典型会话仅调用组件的子集。

此示例显示具有两个技能的插件的输出外观：

```
dependency-guard 1.2.0
  Dependency analysis for Claude Code sessions
  Source: dependency-guard@example-marketplace

Component inventory
  Skills (2)  scan-dependencies, review-changes
  Agents (0)
  Hooks (1)  SessionStart  (harness-only — no model context cost)
  MCP servers (0)
  LSP servers (0)

Projected token cost
  Always-on:   ~180 tok   added to every session

Per-component (rounded)
  component            always-on  on-invoke
  scan-dependencies        ~100      ~2400
  review-changes            ~80      ~1800

  On-invoke cost is paid each time a skill or agent fires.
  Token counts are estimates and may differ from actual usage.
```

always-on 总计通过您的活跃模型的 `count_tokens` API 计算。按组件的数字按比例从该总计缩放。如果 API 无法访问，该命令回退到基于字符的估计。

<h3 id="plugin-validate">
  plugin validate
</h3>

在发布前检查插件或市场的语法和架构错误。

当验证通过时命令退出 0，失败时退出 1，验证运行本身失败时退出 2，例如当您传递的路径不可读时。

```bash theme={null}
claude plugin validate <path> [options]
```

**参数：**

* `<path>`：插件目录或市场目录的路径。请参阅 [Validate a plugin or a directory without a manifest](/docs/zh-CN/plugin-marketplaces#validate-a-plugin-or-a-directory-without-a-manifest) 了解插件运行涵盖的文件。

**选项：**

| 选项           | 描述                                                                                  | 默认值 |
| :----------- | :---------------------------------------------------------------------------------- | :-- |
| `--strict`   | 将警告视为错误，在警告时退出 1。在 CI 中使用以捕获运行时容忍的问题，例如 [unrecognized fields](#unrecognized-fields) |     |
| `--json`     | 将验证报告输出为一个 JSON 对象，具有相同的退出代码。需要 Claude Code v2.1.259 或更高版本                          |     |
| `-h, --help` | 显示命令帮助                                                                              |     |

使用 `--json`，Claude Code 将报告写入 stdout 作为一个 JSON 对象，具有这些顶级字段：

* `success`：退出代码给出的相同判决
* `strict`：运行是否将警告视为错误
* `target`：Claude Code 验证的已解析路径
* `manifest`：清单自己的结果，或 `null` 用于 [run without a manifest](/docs/zh-CN/plugin-marketplaces#validate-a-plugin-or-a-directory-without-a-manifest)
* `contents`：按文件结果，每个命名其 `file` 并携带 `errors`、`warnings` 和 `notes` 数组

在退出 2 时，该命令不向 stdout 写入任何内容；错误消息转到 stderr。

在交互式会话中，`/plugin validate <path>` 内联运行相同的检查。

<h3 id="plugin-tag">
  plugin tag
</h3>

为插件创建发布 git 标签。默认情况下，该命令标记当前目录中的插件；传递路径以标记其他地方的插件。请参阅 [Tag plugin releases](/docs/zh-CN/plugin-dependencies#tag-plugin-releases-for-version-resolution)。

```bash theme={null}
claude plugin tag [path] [options]
```

**参数：**

* `[path]`：插件目录的路径。默认为当前目录。

**选项：**

| 选项                    | 描述                      | 默认值      |
| :-------------------- | :---------------------- | :------- |
| `--push`              | 创建标签后将其推送到远程            |          |
| `--dry-run`           | 打印将被标记的内容而不创建标签         |          |
| `-f, --force`         | 即使工作树脏或标签已存在也创建标签       |          |
| `-m, --message <msg>` | 标签注释消息。使用 `%s` 作为版本的占位符 |          |
| `--remote <name>`     | 使用 `--push` 推送到的远程      | `origin` |
| `-h, --help`          | 显示命令帮助                  |          |

***

<h2 id="debugging-and-development-tools">
  调试和开发工具
</h2>

<h3 id="debugging-commands">
  调试命令
</h3>

使用 `claude --debug` 查看插件加载详情：

这会显示：

* 正在加载哪些插件
* 插件清单中的任何错误
* Skill、agent 和 hook 注册
* MCP 服务器初始化

<h3 id="common-issues">
  常见问题
</h3>

| 问题                                  | 原因                         | 解决方案                                                                                                                                                                                                                                                                                                     |
| :---------------------------------- | :------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 插件未加载                               | 无效的 `plugin.json`          | 运行 `claude plugin validate ./my-plugin` 或 `/plugin validate ./my-plugin`，其中 `./my-plugin` 是你的插件目录，以检查 `plugin.json`、`hooks/hooks.json` 以及插件默认目录中的 skills、agents 和 commands 的前置元数据是否存在语法和模式错误。参见 [验证插件或没有清单的目录](/docs/zh-CN/plugin-marketplaces#validate-a-plugin-or-a-directory-without-a-manifest) 了解运行涵盖的内容 |
| Skills 未显示                          | 目录结构错误                     | 确保 `skills/` 或 `commands/` 在插件根目录，而不是在 `.claude-plugin/` 内                                                                                                                                                                                                                                               |
| Hooks 未触发                           | 脚本不可执行                     | 运行 `chmod +x script.sh`                                                                                                                                                                                                                                                                                  |
| MCP 服务器失败                           | 缺少 `${CLAUDE_PLUGIN_ROOT}` | 对所有插件路径使用变量                                                                                                                                                                                                                                                                                              |
| 路径错误                                | 使用了绝对路径                    | 使路径相对，以 `./` 开头；参见 [路径行为规则](#path-behavior-rules)，其中涵盖了 `skills` 字段的 `"."` 例外                                                                                                                                                                                                                            |
| LSP `Executable not found in $PATH` | 语言服务器未安装                   | 安装二进制文件（例如，`npm install -g typescript-language-server typescript`）                                                                                                                                                                                                                                       |

<h3 id="example-error-messages">
  示例错误消息
</h3>

**清单验证错误**：

* `Invalid JSON syntax: Unexpected token } in JSON at position 142`：检查是否缺少逗号、多余逗号或未引用的字符串
* `Plugin <name> has an invalid manifest file at .claude-plugin/plugin.json. Validation errors: name: Invalid input: expected string, received undefined`：缺少必需字段
* `Plugin <name> has a corrupt manifest file at .claude-plugin/plugin.json. JSON parse error: ...`：JSON 语法错误。在 v2.1.246 之前，Claude Code 也会为保存为带有字节顺序标记 (BOM) 的 UTF-8 的 `plugin.json` 产生此错误，即使 JSON 在其他方面有效。

**插件加载错误**：

* `Warning: No commands found in plugin my-plugin custom directory: ./cmds. Expected .md files or SKILL.md in subdirectories.`：命令路径存在但不包含有效的命令文件
* `Plugin directory not found at path: ./plugins/my-plugin. Check that the marketplace entry has the correct path.`：marketplace.json 中的 `source` 路径指向不存在的目录
* `Plugin my-plugin has conflicting manifests: both plugin.json and marketplace entry specify components.`：删除重复的组件定义或在 marketplace 条目中删除 `strict: false`

<h3 id="hook-troubleshooting">
  Hook 故障排除
</h3>

**Hook 脚本未执行**：

1. 检查脚本是否可执行：`chmod +x ./scripts/your-script.sh`
2. 验证 shebang 行：第一行应为 `#!/bin/bash` 或 `#!/usr/bin/env bash`
3. 检查路径是否使用 `${CLAUDE_PLUGIN_ROOT}`：`"command": "\"${CLAUDE_PLUGIN_ROOT}\"/scripts/your-script.sh"`
4. 手动测试脚本：`./scripts/your-script.sh`

**Hook 未在预期事件上触发**：

1. 验证事件名称正确（区分大小写）：`PostToolUse`，而不是 `postToolUse`
2. 检查匹配器模式是否与你的工具匹配：`"matcher": "Write|Edit"` 用于文件操作
3. 确认 hook 类型有效：`command`、`http`、`mcp_tool`、`prompt` 或 `agent`

<h3 id="mcp-server-troubleshooting">
  MCP 服务器故障排除
</h3>

**服务器未启动**：

1. 检查命令是否存在且可执行
2. 验证所有路径是否使用 `${CLAUDE_PLUGIN_ROOT}` 变量
3. 检查 MCP 服务器日志：`claude --debug` 显示初始化错误
4. 在 Claude Code 外手动测试服务器

**服务器工具未显示**：

1. 确保服务器在 `.mcp.json` 或 `plugin.json` 中正确配置
2. 验证服务器是否正确实现 MCP 协议
3. 检查调试输出中的连接超时

<h3 id="directory-structure-mistakes">
  目录结构错误
</h3>

**症状**：插件加载但组件（skills、agents、hooks）缺失。

**正确结构**：组件必须在插件根目录，而不是在 `.claude-plugin/` 内。只有 `plugin.json` 属于 `.claude-plugin/`。

**调试检查清单**：

1. 运行 `claude --debug` 并查找"loading plugin"消息
2. 检查每个组件目录是否在调试输出中列出
3. 验证文件权限允许读取插件文件

***

<h2 id="distribution-and-versioning-reference">
  分发和版本管理参考
</h2>

<h3 id="version-management">
  版本管理
</h3>

Claude Code 使用插件的版本作为缓存键，以确定是否有可用的更新。当你运行 `/plugin update` 或自动更新触发时，Claude Code 会计算当前版本，如果与已安装的版本匹配，则跳过更新。

对于除 `command` 之外的每种源类型，Claude Code 从以下第一个设置的项中解析版本：

1. 插件 `plugin.json` 中的 `version` 字段
2. 插件在 `marketplace.json` 中的市场条目中的 `version` 字段
3. 插件源的 git 提交 SHA，适用于 git 托管市场中的 `github`、`url`、`git-subdir` 和相对路径源
4. SHA-256 摘要，适用于 [`archive` 源](/docs/zh-CN/plugin-marketplaces#zip-archives)：市场条目中的 `sha256` 固定值，或当你未设置固定值时下载文件的摘要。Claude Code 将其缩短为前 12 个字符
5. `unknown`，适用于 `npm` 源或不在 git 仓库内的本地目录

对于 [`command` 源](/docs/zh-CN/plugin-marketplaces#command-sources)，Claude Code 始终从命令生成的内容中派生版本：单独的 12 字符内容哈希，或在设置了版本时附加到 `plugin.json` 版本作为 `<version>-<hash>`。Claude Code 忽略命令源的市场条目中的 `version` 字段。因此，命令的哈希输出发生变化会产生新版本，即使编写的版本字符串保持不变。在 [link mode](/docs/zh-CN/plugin-marketplaces#copy-mode-and-link-mode) 中，哈希覆盖打印目录的真实路径及其顶级条目，而不是文件内容。

对于这些源类型，这为你提供了三种版本控制插件的方式：

| 方法            | 如何操作                                                                                           | 更新行为                                                             | 最适合                       |
| :------------ | :--------------------------------------------------------------------------------------------- | :--------------------------------------------------------------- | :------------------------ |
| **显式版本**      | 在 `plugin.json` 中设置 `"version": "2.1.0"`                                                       | 用户仅在你更新此字段时获得更新。推送新提交而不更新它没有效果，`/plugin update` 报告"已是最新版本"。      | 具有稳定发布周期的已发布插件            |
| **提交 SHA 版本** | 从 `plugin.json` 和市场条目中都省略 `version`                                                            | 每当源的已解析提交发生变化时，用户获得更新                                            | 正在积极开发的内部或团队插件            |
| **摘要版本**      | 使用 [`archive` 源](/docs/zh-CN/plugin-marketplaces#zip-archives) 并从 `plugin.json` 和市场条目中都省略 `version` | 使用 `sha256` 固定值时，当你更改固定值时用户获得更新。没有固定值时，每当托管 zip 文件的字节发生变化时用户获得更新 | 作为 zip 文件发布到静态服务器或工件仓库的插件 |

如果你使用显式版本，请遵循 [semantic versioning](https://semver.org)（`MAJOR.MINOR.PATCH`）：对于破坏性更改，增加 MAJOR；对于新功能，增加 MINOR；对于错误修复，增加 PATCH。在 `CHANGELOG.md` 中记录更改。

***

<h2 id="see-also">
  另请参阅
</h2>

* [Plugins](/docs/zh-CN/plugins) - 教程和实际用法
* [Plugin marketplaces](/docs/zh-CN/plugin-marketplaces) - 创建和管理市场
* [Skills](/docs/zh-CN/skills) - Skill 开发详情
* [Subagents](/docs/zh-CN/sub-agents) - Agent 配置和功能
* [Hooks](/docs/zh-CN/hooks) - 事件处理和自动化
* [MCP](/docs/zh-CN/mcp) - 外部工具集成
* [Settings](/docs/zh-CN/settings) - Plugins 的配置选项
