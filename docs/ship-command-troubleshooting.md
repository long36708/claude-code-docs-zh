> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 排除 /ship 和 /steward 的故障

> 查找 `/ship` 或 `/steward` 的消息或症状：命令缺失、未提交或推送任何内容、监视未启动，或监视已停止。

[`/ship`](https://code.claude.com/docs/zh-CN/ship-command) 是一个 Claude Code 命令，它会为您的更改打开一个 GitHub Pull Request，并让 Claude 对其进行审查。随后，Claude 会在您的会话中继续处理该 Pull Request。这项持续的工作称为监视（watch）。`/steward` 会对已经打开的 Pull Request 启动监视。

在第一列中查找所发生的情况，或搜索 Claude Code 所打印消息中的文字：

| 发生的情况 | 前往 |
| :- | :- |
| `/ship` 不在 `/` 菜单中，或 Claude Code 无法识别它 | [您的会话中没有 /ship](#%2Fship-isn’t-available-in-your-session) |
| 您输入了命令，它打印了一条消息但没有启动 | [/ship 或 /steward 打印了一条消息且未启动](#%2Fship-or-%2Fsteward-printed-a-message-and-didn’t-start) |
| 命令已运行，但没有提交或推送任何内容 | [/ship 未提交或推送任何内容](#%2Fship-committed-or-pushed-nothing) |
| Pull Request 已打开，但 Claude 没有跟进它 | [未启动监视](#no-watch-started) |
| Claude 曾在跟进，但停止了 | [监视已停止](#a-watch-stopped) |
| Claude 跟进期间出现了一条消息 | [监视期间的消息](#messages-during-a-watch) |
| Pull Request 与您的预期不符 | [意料之外的 Pull Request 结果](#pull-request-results-that-you-didn’t-expect) |
| 自定义的 `/ship` 或 `/steward` 命令停止工作 | [自定义的 /ship 或 /steward 命令停止加载](#a-custom-%2Fship-or-%2Fsteward-command-stopped-loading) |

## /ship 或 /steward 缺失或未启动

当命令未出现在可选项中，或者它打印了一条消息时，请从这里开始。

### 您的会话中没有 /ship

`/ship` 不在 `/` 菜单中、输入它会启动 `/code-review`，或者您看到 `Unknown command: /ship`。云端会话中没有 `/ship`。

在您自己的计算机上，请执行以下步骤：

1. 在 shell 中运行 `claude update`，然后退出并重新启动 `claude`。
2. 如果 `/ship` 仍然缺失，请再次退出并启动 `claude`，因为新功能[可能需要第二次启动](https://code.claude.com/docs/zh-CN/env-vars.md#first-session-after-an-install-or-upgrade)才会显示。
3. 在 Claude Code 提示符处运行 `/status`。其 `Setting sources` 行列出了已加载的设置。在其中属于您自己的每个[设置文件](https://code.claude.com/docs/zh-CN/settings.md#find-or-create-your-settings-files)中，删除 [`disableBundledSkills`](https://code.claude.com/docs/zh-CN/settings-reference#disablebundledskills) 以及 [`skillOverrides`](https://code.claude.com/docs/zh-CN/skills#override-skill-visibility-from-settings) 下的任何 `ship-local` 条目。要更改托管设置，请联系您的管理员。
4. 检查您的会话是否[跳过了功能标志获取](https://code.claude.com/docs/zh-CN/env-vars.md#features-that-need-feature-flag-fetching)，例如因为设置了 `DISABLE_TELEMETRY` 或您使用了第三方提供商。在这些会话中无法使用 `/ship`。

### skillOverrides 条目关闭了 /ship 或 /steward

`Skill "ship-local" is disabled via skillOverrides`，或针对 `steward` 的相同消息，表示您设置中的某个 [`skillOverrides`](https://code.claude.com/docs/zh-CN/skills#override-skill-visibility-from-settings) 条目关闭了该命令。请删除该条目。要更改托管设置中的条目，请联系您的管理员。

### /ship 或 /steward 打印了一条消息且未启动

查找消息的开头文字：

| 消息 | 处理方法 |
| :- | :- |
| `This session was started by another session` | 在您自己启动的会话中运行它 |
| `Usage: /ship` 或 `Usage: /steward` | 对照[参数](https://code.claude.com/docs/zh-CN/ship-command.md#arguments-for-%2Fship-and-%2Fsteward)检查您输入的内容 |
| `Nothing started. /ship starts only from a line that you type`，或针对 `/steward` 的相同消息 | 在 Claude Code 提示符处亲自输入该命令。它不会通过 `claude -p` 或程序启动 |
| `/ship didn't start:` 或 `/steward didn't start:` | 消息的其余部分会说明需要检查的内容 |
| `/ship is turned off right now`，或针对 `/steward` 的相同消息 | 按照消息所述，在 Claude Code 提示符处运行 `/login`，然后再次运行该命令 |
| `/ship can't start from a queued message` 或 `/steward can't start from a queued message` | 等待 Claude 完成，然后再次运行 |

### GitHub CLI 尚未就绪

`/ship` 和 `/steward` 通过 GitHub CLI `gh` 访问 GitHub：

| 消息 | 处理方法 |
| :- | :- |
| `GitHub CLI (gh) isn't installed` 或 `GitHub CLI (gh) isn't signed in` | 安装它，在 shell 中运行 `gh auth login`，然后再次运行该命令 |
| `gh couldn't tell me about a pull request here` | 括号中的文字说明了失败的内容。要查看 `gh` 自身的错误，请在 shell 中运行 `gh pr view`。一个原因是您检出的仓库的所有远程都不在 `gh` 已登录的 GitHub 主机上。对于 GitHub Enterprise，请在 shell 中运行 `gh auth login --hostname HOST` |

## /ship 未提交或推送任何内容

在某些情况下，`/ship` 会把提交或推送留给您自己完成，并说明原因。其消息可能以 `Nothing committed or pushed`、`Nothing pushed`、`I didn't push anything`、`There is no pull request to watch` 或 `This branch has no open pull request` 开头。

找到您的情况，执行对应行中的步骤，然后再次运行 `/ship`。在这些情况下，如果没有推送任何内容且该分支没有 Pull Request，Claude 会审查您的更改，并且[不会启动监视](#no-watch-started)。

### 公开仓库以及 /ship 不会提交的文件

`/ship` 在提交之前会检查仓库以及每个已更改文件的名称：

| 情况 | 处理方法 |
| :- | :- |
| `For a public repository /ship doesn't commit anything for you` | 在默认分支以外的分支上自行提交您的工作 |
| 对于 `.env` 或 `password.txt` 等已更改的文件，出现 `By its name, this file may hold secrets` 或 `By their names, these files may hold secrets` | 如果其中不包含机密信息，请在默认分支以外的分支上自行提交您的工作。否则，请将其搁置，例如在 shell 中使用 `git stash push -u -- <file>` |
| 消息中列出了并非由您创建的空文件，例如 `.bashrc` 或 `.gitconfig` | 在 Linux 和 WSL2 上，当沙箱化命令运行时，[沙箱](https://code.claude.com/docs/zh-CN/sandboxing.md)会在您的检出目录中保留空文件。沙箱在命令结束之前都需要这些文件，因此在此之前不要暂存、提交或删除它们。停止该命令（例如通过 [`/tasks`](https://code.claude.com/docs/zh-CN/commands#all-commands)），或等待其结束，沙箱就会移除这些文件。如果在没有沙箱化命令运行时这些文件仍然存在，请[删除残留文件](https://code.claude.com/docs/zh-CN/errors.md#stale-sandbox-mask-files-left-by-a-killed-session) |
| 您更改了用于配置 git、CI 或编码 Agent 的文件，例如 `.gitignore`、`CLAUDE.md` 或 `.github/` 下的任何内容 | 在默认分支以外的分支上自行提交您的工作 |

### /ship 无法提交或推送的分支

`/ship` 需要一个它可以提交和推送的分支：

| 情况 | 处理方法 |
| :- | :- |
| `there is nothing here to open one for`：您没有未提交的更改，并且该分支上的每个提交都已在默认分支上 | 进行更改，或检出包含您工作的分支 |
| 默认分支包含您的提交，并且没有未提交的内容 | 将这些提交移到新分支，例如在 shell 中使用 `git switch -c <branch>` |
| 您处于分离 HEAD 状态，或正在进行变基 | 检出一个分支，或完成变基 |

### 禁止提交或推送的拒绝规则

[拒绝规则](https://code.claude.com/docs/zh-CN/permissions.md#manage-permissions)（例如 `Bash(git push *)`）可能会禁止提交、推送或打开 Pull Request。如果设置该规则的人同意，请自行执行这些步骤。

监视同样需要 `git commit` 和 `git push`。如果该规则禁止了其中之一，请在再次运行 `/ship` 之前[将其移除](#a-tool-rule-or-hook-blocks-a-watch)。

### GitHub Enterprise 和复刻

`/ship` 不会在 GitHub Enterprise 上或从复刻打开 Pull Request。请推送分支并自行打开 Pull Request，然后运行 `/ship`。如果从复刻操作，检出目录还需要有一个指向 Pull Request 目标仓库的远程，否则不会启动监视。

## 未启动监视

监视是 Claude 在您的 Pull Request 上持续进行的工作，它在 `/ship` 的审查之后或您输入 `/steward` 时启动。在某些情况下，监视不会启动，Claude 会说明原因。其消息可能包含 `/steward didn't start.`、`I can't watch` 或 `is running on pull request`。

解决原因后，输入 `/steward` 以启动监视而不进行第二次审查；如果该分支仍然没有 Pull Request，则输入 `/ship`。

### 您的 Pull Request 或分支尚未准备好进行监视

首先检查 Pull Request 和您的检出目录：

| 原因 | 处理方法 |
| :- | :- |
| 该分支没有 Pull Request，而 `/steward` 不会打开 Pull Request | 请参阅 [/ship 未提交或推送任何内容](#%2Fship-committed-or-pushed-nothing) |
| 此会话中未检出该 Pull Request 的分支 | 检出该分支 |
| 该 Pull Request 带有 [PR Steward](https://code.claude.com/docs/zh-CN/pr-steward.md) 的某个标签，例如 `claude-pr-steward-watching` | 要接管，请按照 Claude 消息所述，移除该标签并拉取该分支 |

### 会话设置阻止了监视

会话的某些设置会阻止监视启动：

| 原因 | 处理方法 |
| :- | :- |
| 会话处于 `bypassPermissions` 模式 | 按 `Shift+Tab` 切换到计划模式以外的权限模式 |
| [Remote Control](https://code.claude.com/docs/zh-CN/remote-control.md) 已开启 | 运行 `/rc`，然后选择 **Disconnect this session** |
| `your PATH has a relative or empty entry` | 不带 Pull Request 编号的 `/ship` 会审查您的更改，但不会提交、推送或查找 Pull Request。从 `PATH` 中移除空条目或相对条目，例如末尾的 `:`（在 Windows 上为 `;`）。重新启动 `claude`，然后运行 `/ship` |
| 您的 `PATH` 包含检出目录内的某个文件夹。例如，当您激活位于检出目录中的 Python 虚拟环境，或通过 `npx` 或包脚本启动 `claude` 时，就会发生这种情况 | 要在不包含该文件夹的情况下启动会话，如果您使用了虚拟环境，请在 shell 中运行 `deactivate`，然后直接启动 `claude` |

### 工具、规则或 hook 阻止了监视

监视需要 `gh`、`git commit` 和 `git push`，并且在项目的 hook 可能批准命令的情况下不会启动：

| 原因 | 处理方法 |
| :- | :- |
| `gh` 未安装或未登录 | 请参阅 [GitHub CLI 尚未就绪](#the-github-cli-isn’t-ready) |
| 拒绝规则禁止了监视所需的命令，例如 `git push` | Claude 的消息会指出该命令以及包含该规则的设置。如果设置该规则的人同意，请在那里将其移除。要更改托管设置中的规则，请联系您的管理员 |
| 项目设置了一个可能批准命令的 hook，例如检出目录的 `.claude/settings.json` 中针对 shell 命令的 `PreToolUse` 或 `PermissionRequest` [hook](https://code.claude.com/docs/zh-CN/hooks.md) | 消息会指出该文件并说明需要更改的内容 |

### 无法启动监视的情况

在以下情况下，`/ship` 仍然可以审查您的更改：

* 您使用的是原生 Windows
* Pull Request 是由其他人打开的
* 您无法推送到 Pull Request 的目标仓库，并且其他人已发表评论或进行了审查。如果在监视运行期间发生这种情况，监视会结束

## 监视已停止

监视是 Claude 在您的 Pull Request 上持续进行的工作，它在 `/ship` 的审查之后或您输入 `/steward` 时启动。监视结束后，请查找以 `Stopped watching pull request` 开头的行，其中给出了原因。

要重新启动监视，请输入该行给出的命令（如果有的话）。要检查监视是否仍在运行，请在 Claude 空闲时在 Claude Code 提示符处输入 [`/tasks`](https://code.claude.com/docs/zh-CN/ship-command#check-whether-a-watch-is-running)。

查找该行中 "because" 之后的文字：

| 原因 | 发生的情况 |
| :- | :- |
| `its check-in was cancelled` | 您在提示符处或 Claude 工作时按了 `Esc` 或 `Ctrl+C`，或者在 `/tasks` 中停止了该任务。Claude 的消息将其在 Pull Request 上的工作称为 check-in。用于关闭 `/tasks` 列表的 `Esc` 不会停止正在运行的监视 |
| `the watch couldn't go on` | Claude 的回复可能会给出原因。一个原因是该 Pull Request 的分支不再处于检出状态。请在您启动时所在的文件夹中检出该分支，然后输入 `/steward` |
| `a watch doesn't carry over when a session is resumed` | 运行监视的进程已结束，例如因为您关闭了终端。在 shell 中运行 `claude --continue` 会恢复对话，但不会恢复监视 |
| `you moved to another session` | `/clear`、`/resume` 或 `/branch` 可能会导致这种情况 |
| `its check-ins were replaced` | 您启动了 `/loop` |
| `your new command took over this session` | 您再次运行了 `/ship` 或 `/steward` |
| `a deny rule now forbids a command that the watch runs` | 在 Claude Code 提示符处输入 `/permissions` 以查看该规则。请参阅[工具、规则或 hook 阻止了监视](#a-tool-rule-or-hook-blocks-a-watch) |
| `the watch reached its time limit` | 已过去大约七天 |
| `no next check-in was set` | 被拒绝的权限提示、错误或用量限制可能会导致这种情况 |
| `a file that gh, git, or /steward relies on changed` | 请参阅[监视所依赖的文件已更改](#a-file-that-the-watch-relies-on-changed) |

`No pull request is being watched in this session` 表示您输入了 `/ship stop` 或 `/steward stop`，但没有正在运行的监视。请向上滚动查找 `Stopped watching pull request` 行。

### 监视所依赖的文件已更改

当有操作更改了 `gh`、`git` 或 `/steward` 所依赖的文件时，监视会结束。以下文件属于此类：

* **Git 的设置**：`git push -u`、`git branch -u` 和 `git remote add` 会更改它们。VS Code 等编辑器在打开仓库或您切换回仓库时，也可能会向 `.git/config` 添加设置
* **`.claude/settings.local.json`**：当您对权限提示选择 "Yes, and don't ask again" 时，Claude Code 会写入此文件

检查该更改，然后输入 `/steward` 以重新启动监视。

如果您没有更改任何文件，在 Linux 和 WSL2 上的一个原因是在监视期间启动或结束了[沙箱化](https://code.claude.com/docs/zh-CN/sandboxing.md)命令。请停止该命令或等待其结束。

## 监视期间的消息

以下消息会在监视期间出现在您的会话中：

| 消息或您看到的内容 | 处理方法 |
| :- | :- |
| `Claude Code did not run this` 或 `Not run · permission denied` | Claude 尝试了某个步骤，但[监视的限制](https://code.claude.com/docs/zh-CN/ship-command.md#what-claude-code-blocks-during-a-watch)或您的某个规则或 hook 拒绝了它。当监视已经结束时，也会出现此消息。Claude 的消息将这些限制称为“the fence” |
| `In this session there's a way around my limits` | 您会话中的某项设置（例如规则、hook、插件或沙箱）允许某些命令绕过[监视的限制](https://code.claude.com/docs/zh-CN/ship-command.md#what-claude-code-blocks-during-a-watch)。消息后面的文字会指出具体是哪一项。要查看您的规则，请在 Claude Code 提示符处输入 `/permissions`。要在您做出决定期间结束监视，请输入 `/steward stop` |
| 类似 `1 comment not read (no write access, or a bot that you haven't named)` 的行 | Claude [跳过了一条评论](https://code.claude.com/docs/zh-CN/ship-command.md#what-claude-does-during-a-watch)。打开消息中的链接即可阅读。对于您希望 Claude 阅读的机器人评论，请输入 `/steward +botname`。对于您认同的某人的评论，请告诉 Claude 需要更改什么 |
| `gh couldn't say where the pull request's head and base stand` | 一个原因是 GitHub CLI 的版本低于 2.63。在 shell 中运行 `gh --version` 进行检查。更新它，然后输入 `/steward` 以重新启动监视 |
| 会话中出现 `Claude resuming /loop wakeup` 或 `/steward --tend` | 当 Claude 在监视期间检查 Pull Request 时，Claude Code 会输入这些内容。您可以照常继续工作 |

## 意料之外的 Pull Request 结果

当 `/ship` 已运行，而 GitHub 上或您仓库中的某些内容出乎您的意料时，请从这里开始：

| 您看到的内容 | 原因及处理方法 |
| :- | :- |
| Pull Request 的标题是类似 `Update greet.py and 1 more` 的一行 | 当 Claude 无法根据会话的对话撰写标题时（例如在您没有讨论该更改的会话中），`/ship` 会退而使用这样的标题。请在 GitHub 上编辑标题和描述。下次，请在您进行更改的会话中运行 `/ship`，或者先在默认分支以外的分支上自行提交您的工作 |
| Claude 在您看到之前就推送了提交或回复了审查者 | 在自动模式下，或者在您的规则或沙箱允许某个命令的情况下，监视就是这样工作的。请参阅[让 Claude 先与您确认](https://code.claude.com/docs/zh-CN/ship-command.md#ask-claude-to-check-with-you-first) |
| 新的未提交更改没有进入已打开的 Pull Request | Pull Request 打开后，`/ship` 不会提交您的工作。请提交它，然后推送它，例如在 shell 中运行 `git push origin <branch>` |
| `git pull` 或 `git push` 要求指定上游分支 | `/ship` 不会设置上游。请指定远程和分支，例如 `git push origin <branch>`。设置上游会更改 git 的设置，这会[结束正在运行的监视](https://code.claude.com/docs/zh-CN/ship-command.md#keep-working-while-a-watch-runs) |

## 自定义的 /ship 或 /steward 命令停止加载

`Not loaded: a custom /ship` 或 `Not loaded: custom /ship, /steward` 通知与某个自定义 skill 或命令有关。其名称是内置命令保留的名称之一，例如 `ship`、`ship-local` 或 `steward`。请将其重命名，然后重新启动 Claude Code。

请保持仓库中的 `.claude/skills/steward/SKILL.md` 原样，因为 PR Steward 和监视会将其作为指令读取。该通知不会将此文件计算在内，因此提及 `/steward` 的通知指的是另一个 skill 或命令。

## 相关资源

* [`/ship` 和 `/steward`](https://code.claude.com/docs/zh-CN/ship-command)：这些命令的作用、如何运行它们，以及监视期间对 Claude 的限制
* [权限模式](https://code.claude.com/docs/zh-CN/permission-modes.md)：自动模式和其他权限模式
* [Hooks](https://code.claude.com/docs/zh-CN/hooks.md)：`PreToolUse` 和 `PermissionRequest` hook
