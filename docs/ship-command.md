> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 使用 /ship 和 /steward 让 Pull Request 做好合并准备

> 运行 `/ship` 来创建 GitHub Pull Request、获取审查，并让 Claude 处理反馈和失败的检查；或运行 `/steward`，对已打开的 Pull Request 跳过审查。

`/ship` 是一个 Claude Code 命令，如果您的分支还没有 Pull Request，它会为您的更改创建一个 GitHub Pull Request。Claude 会审查您的更改，并处理反馈和失败的检查，帮助其做好合并准备。

如果您的 Pull Request 已经打开且不需要审查，请输入 [`/steward`](#start-a-watch-with-%2Fsteward)。它会跳过审查，Claude 会同样处理反馈和失败的检查。

> [!NOTE]
>
> 请在终端中、在仓库的本地克隆里，通过 Claude Code CLI 运行这两个命令。云端会话中没有这两个命令。

输入 `/ship` 后，您的更改会经过以下步骤：

![输入 /ship 后所发生过程的示意图。在您的会话中，Claude Code 创建一个 Pull Request，并使用 /code-review 对其进行审查。随后开始监视：Claude 向您的分支推送修复，并响应检查、评论和冲突。当 Pull Request 准备就绪时，由您进行合并。](https://mintcdn.com/claude-code/sMH7RDpTvyWlAkrg/images/ship-flow.svg?fit=max\&auto=format\&n=sMH7RDpTvyWlAkrg\&q=85\&s=ccfa7e5848711d691b8cdf65ac22b26e)

1. **Pull Request**：`/ship` 可以以您的身份[提交您的工作、推送您的分支并创建 Pull Request](#what-%2Fship-commits-and-pushes)
2. **审查**：Claude 使用 [`/code-review`](https://code.claude.com/docs/zh-CN/code-review-local) 审查您的更改
3. **监视**：Claude 会在您的 Claude Code 会话中持续检查该 Pull Request。它会处理失败的检查、合并冲突、审查者的评论以及自身审查的发现，并推送修复。这项持续进行的工作称为监视（watch）
4. **合并**：Pull Request 准备就绪后，由您自己进行合并

`/steward` 会跳过第 1 步和第 2 步，直接开始监视。

## 何时使用 /ship 和 /steward

无论使用哪个命令，Claude 都会处理您的 Pull Request 的反馈和失败的检查。请根据更改所处的阶段在 `/ship` 和 `/steward` 之间进行选择：

* **`/ship`**：您的更改已完成，希望 Claude 先进行审查，再处理随后出现的反馈和失败的检查
* **`/steward`**：您的 Pull Request 已经打开且不需要先审查，或者监视已结束而您想重新开始

两个命令都在您的会话中运行，因此其工作会计入您的用量。

如果您需要其他功能，例如仅进行审查或从云端会话进行修复，请使用以下方式之一：

| 您需要 | 使用 |
| :- | :- |
| 仅进行审查 | [`/code-review`](https://code.claude.com/docs/zh-CN/code-review-local) |
| 在反馈到达时从云端会话进行修复 | [Auto-fix](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md#auto-fix-pull-requests) 或 `/autofix-pr` |
| 面向整个仓库、费用计入组织的 Agent | [Claude Ship](https://code.claude.com/docs/zh-CN/claude-ship.md) |

## /ship 和 /steward 的前提条件

安装或更新以下工具：

* Claude Code v2.1.330 或更高版本。如果缺少 `/ship`，请参阅 [/ship 在您的会话中不可用](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting.md#%2Fship-isn’t-available-in-your-session)
* 对于 GitHub 上的仓库，需要 [GitHub CLI](https://cli.github.com) 2.63 或更高版本，并已通过 `gh auth login` 登录。使用较旧版本时，不会开始任何监视。在 v2.1.331 之前，监视会开始，但 Claude 无法推送修复
* git 2.38 或更高版本，用于将新提交合并到您的分支中。使用较旧的 git 时，Claude 会将这些合并留给您处理，并且[在其他托管平台上](#watch-a-branch-on-another-host)不会开始任何监视

然后准备好您的更改和会话：

* **已完成的更改（用于 `/ship`）**：更改需位于 GitHub 上某个仓库的本地克隆中。它可以是未提交的、已提交在默认分支以外的分支上的，或者已经在您自己打开的 Pull Request 中。如果仓库托管在 GitHub 以外的地方，`/ship` 不会创建 Pull Request。Claude 会审查您的更改，然后可以[监视您的分支](#watch-a-branch-on-another-host)。在 v2.1.331 之前，在这种情况下不会开始任何监视
* **只包含您希望放入 Pull Request 的文件（用于 `/ship`）**：`/ship` 可能会提交您所有未提交的更改，包括新文件。请在 shell 中运行 `git status`。删除每个不希望放入 Pull Request 的新文件，或使用 `git stash push -u -- <file>` 将其暂存
* **这些文件中不含机密信息（用于 `/ship`）**：`/ship` 按文件名（例如 `.env`）查找机密信息，因此请您自己检查文件内容。诸如 `Read(./notes.txt)` 之类的拒绝规则并不能将文件排除在提交之外
* **GitHub 上已打开的 Pull Request（用于 `/steward`）**：它必须是您自己的，并且其分支已在您的本地克隆中检出。对于 GitHub 以外的托管平台，请参阅[在其他托管平台上监视分支](#watch-a-branch-on-another-host)
* **权限模式**：如果希望 Claude 在无人值守的情况下处理 Pull Request，请选择[自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode)。在大多数其他权限模式下，Claude 会在每个权限提示处等待。在 `bypassPermissions` 模式下，不会开始任何监视。在计划模式下，`/ship` 不会启动

在原生 Windows 上，[不会开始任何监视](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting.md#cases-where-no-watch-can-start)。在[后台会话](https://code.claude.com/docs/zh-CN/agent-view.md)中，同样[不会开始任何监视](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting.md#a-session-setting-blocks-a-watch)。在 v2.1.331 之前，监视可以在后台会话中开始。

> [!WARNING]
>
> 仅当您信任所有能够合并到基础分支或推送到您分支的人时，才运行监视。他们中的任何人都可以使用您的凭据在您的计算机上运行程序。

## 运行 /ship 或 /steward

在终端中，于您的克隆中打开 Claude Code，然后在输入框中输入 `/ship` 或 `/steward`。两者都接受[参数](#arguments-for-%2Fship-and-%2Fsteward)，例如 Pull Request 的编号。

### 对已完成的更改运行 /ship

如果您是在 Claude 的协助下完成更改的，请使用同一个会话，因为 Claude 会根据会话中的对话撰写提交信息。任何能看到该 Pull Request 的人都可以阅读这条信息。

> [!NOTE]
>
> `/ship` 会在没有确认提示的情况下继续执行。按下 Enter 后，它可以以您的身份[提交您的工作、推送您的分支并创建 Pull Request](#what-%2Fship-commits-and-pushes)。在提交之前以及推送之前，都会显示一行以 `Esc to cancel` 结尾的文字，说明即将执行的操作。如果您不做任何操作，`/ship` 会在几秒钟后继续。

如果您在推送之前取消，`/ship` 所做的提交会保留。要撤销该提交并保留您的文件，请在 shell 中运行 `git reset HEAD~1`。

### /ship 提交和推送的内容

在 github.com 上且不是 fork 的仓库中，`/ship` 可以在审查之前提交您未提交的更改、推送分支并创建 Pull Request。如果位于默认分支上，它会在新分支上提交，并将您的检出切换到该分支。如果您的工作已提交在其他分支上，`/ship` 会推送该分支并创建 Pull Request。

在以下情况下，`/ship` 会采取不同的处理方式。请以第一个符合的情况为准：

* **该分支已有他人打开的 Pull Request**：`/ship` 不会提交或推送您的工作，也不会开始监视
* **某个未提交文件的名称暗示其包含机密信息，例如 `.env`**：在非公开仓库中，`/ship` 不会提交或推送任何内容。请参阅 [/ship 未提交或推送任何内容](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting.md#%2Fship-committed-or-pushed-nothing)
* **该分支已有您自己打开的 Pull Request**：在可以开始监视且仓库非公开的情况下，`/ship` 可以提交您未提交的更改，然后将这些更改以及您尚未推送的提交推送到该 Pull Request。[在其他情况下](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting.md#your-pull-request-is-already-open)，它会略过其中部分工作。如果随后开始监视，您尚未推送的提交[可以随 Claude 的下一次推送一起推送](#keep-working-while-a-watch-runs)。在 v2.1.331 之前，`/ship` 在这种情况下不会提交或推送
* **仓库是公开的**：`/ship` 不会提交您未提交的更改。如果您位于默认分支以外的分支上，且该分支有自己的提交，`/ship` 会推送这些提交并创建一个不包含您未提交更改的 Pull Request。否则，请在默认分支以外的分支上提交您的工作，然后再次运行 `/ship`
* **您的提交位于默认分支上，且没有未提交的内容**：在非公开仓库中，`/ship` 会为这些提交创建一个新分支，推送该分支，创建 Pull Request，并将您的检出切换到新分支。在 v2.1.331 之前，`/ship` 不会推送任何内容，因此请将这些提交放到默认分支以外的分支上，然后再次运行 `/ship`

如果您需要以下任一项，请自行创建 Pull Request，然后运行 `/ship`：

* 草稿 Pull Request
* 其他基础分支
* 您仓库的 Pull Request 模板
* [GitHub Enterprise 上或来自 fork 的](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting.md#github-enterprise-and-forks) Pull Request

如果 `/ship` 未提交或推送任何内容，且上述情况无法解释原因，请参阅 [/ship 未提交或推送任何内容](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting.md#%2Fship-committed-or-pushed-nothing)。

### 使用 /steward 开始监视

要在不先进行审查的情况下开始监视，请在 Claude Code 输入框中输入 `/steward`。可将其用于您自己已打开的 Pull Request，或在监视结束后重新开始。在让监视无人值守运行之前，请参阅[监视在您的计算机上运行的代码](#code-that-a-watch-runs-on-your-computer)。

`/steward` 与 [PR Steward](https://code.claude.com/docs/zh-CN/pr-steward.md)（由组织设置的 Agent）是不同的。如果 PR Steward 已经在处理您的 Pull Request，则不会开始监视。

## 监视期间 Claude 会做什么

监视是 Claude 对您的 Pull Request 持续进行的工作，它在 `/ship` 的审查之后或您输入 `/steward` 时开始。如果使用 GitHub 以外的托管平台，请参阅[在其他托管平台上监视分支](#watch-a-branch-on-another-host)。Claude 使用您的 `gh` 和 `git` 登录身份，因此它在 GitHub 上的工作会显示在您的名下。

在自动模式下，对于 GitHub 上的 Pull Request，Claude 会在不询问的情况下执行以下操作：

* 处理它判断为问题的审查发现
* 可以发布一条总结其审查的评论
* 向您的分支推送提交
* 将基础分支合并到您的分支中，以解决冲突或使您的分支保持最新
* 处理失败的检查，并重新运行失败的检查
* 回复审查评论、解决讨论串，并重新请求审查

在 GitHub 上，Claude 会读取您以及拥有写入权限的人员的评论。在 github.com 上，它还会读取 Claude 机器人的评论。要让 Claude 读取其他机器人的评论，请输入 `/steward +` 加上机器人的名称，例如 `/steward +my-lint-bot`。

当检查通过且评论都已回复时，Claude 会在会话中告知您。请您自己合并 Pull Request。能否合并取决于您的权限和仓库的分支规则。请参阅[监视何时可能导致合并](#when-a-watch-can-lead-to-a-merge)。

### 在其他托管平台上监视分支

在 GitHub 以外的托管平台上（例如 GitLab、Bitbucket 或 Azure DevOps），Claude 可以监视您的分支最多七天。监视需要 git 2.38 或更高版本，并且无需 GitHub CLI 即可工作。`/ship` 和 `/steward` 在启动时不会提交、推送或打开任何内容。在 v2.1.331 之前，`/ship` 会在这些平台上审查您的更改，但不会启动监视。

请先提交您的更改，然后在 shell 中推送您的分支并设置上游：

```bash theme={null}
git push -u origin <branch>
```

在 gitlab.com、bitbucket.org 或 dev.azure.com 上，接下来请打开您的 Merge Request 或 Pull Request。这样 Claude Code 就可以[使用您的 git 登录身份读取其检查和评论](#checks-and-comments-on-gitlab-bitbucket-and-azure-devops)。然后在 Claude Code 输入框中输入 `/ship`，或输入 `/steward` 以启动不先进行审查的监视。

借助您的 git 登录身份，Claude 可以在自动模式下不经询问执行以下操作：

* 将基础分支合并到您的分支中以解决冲突
* 合并其他人推送到您分支的内容
* 向您的分支推送提交

在这些平台上，监视不会为 Claude 提供回复评论、重新运行检查或读取 CI 日志的命令。要查看 Claude 在这些平台上可以执行的操作，请在监视运行时于输入框中输入 `/ship terms`。

#### GitLab、Bitbucket 和 Azure DevOps 上的检查和评论

在终端会话中，Claude Code 可以读取您已打开的 Merge Request 或 Pull Request 及其检查和评论等详细信息。为此，它会向 git 请求 git 为您的分支推送到的远程所保存的登录凭据。该登录凭据会通过 `GET` 请求发送到 gitlab.com、bitbucket.org 或 dev.azure.com 的 API。系统不会事先询问您。

在首次读取之前会出现类似下面的一行提示，首次读取可能在您运行 `/ship` 或 `/steward` 后立即发出：

```text theme={null}
Claude Code is about to read gitlab.com's web API with the sign-in that git keeps for it. It only reads.
```

读取托管平台的监视会重复这些读取，直到监视结束。要查看这些读取是否适用于您的检出，请在 shell 中运行 `git remote -v`。只有当您的分支推送到的远程的 `(fetch)` 行显示这些平台之一上的 `https` 地址时，才可能发生这些读取。

要阻止这些读取，请在运行 `/ship` 或 `/steward` 之前将远程切换为 SSH，或在设置中拒绝该托管平台。如果监视正在运行，请先输入 `/ship stop`：

* 让您的分支推送到的远程通过 SSH 进行获取，例如在 shell 中运行 `git remote set-url origin git@gitlab.com:<group>/<project>.git`，并将 `origin` 替换为该远程的名称。然后检查 `git remote -v` 的 `(fetch)` 行中没有 `https` 地址
* 将该托管平台添加到 [`sandbox.network.deniedDomains`](https://code.claude.com/docs/zh-CN/settings-reference#sandbox-network-denieddomains)。`about to read` 通知仍会出现，但 Claude Code 不会向 git 请求登录凭据，也不会调用 Web API。监视自身的 git 命令由 Claude Code 在[沙箱](https://code.claude.com/docs/zh-CN/sandboxing.md)之外运行，仍会使用 git 的登录凭据连接该托管平台。启用沙箱后，Claude 也无法从该托管平台获取或向其推送

在监视开始时，托管平台可能会拒绝该登录凭据，或者显示该分支没有您已打开的 Merge Request 或 Pull Request。此时监视可以仅依靠 git 继续进行，其开始时显示的行中会包含 `Claude can't see checks or comments on gitlab.com, so watch those yourself`。如果您在未合并的情况下关闭了您的 Merge Request 或 Pull Request，该监视仍会继续，因此请输入 `/ship stop`。

### 在监视运行时继续工作

监视运行期间，请保持会话打开，保持计算机处于唤醒状态，并保持 Pull Request 的分支处于检出状态。在 v2.1.331 之前，还需要让会话保持在[前台](https://code.claude.com/docs/zh-CN/agent-view.md#send-the-session-to-the-background)。Claude 与您在同一个检出中编辑文件。

* **要开始下一个更改**：请在另一个文件夹（例如仓库的第二个克隆）中工作，并在那里打开另一个 Claude Code 会话
* **要推送您自己的提交**：请指定远程和分支，例如 `git push origin <branch>`。如果您更改 git 的设置，例如使用 `git push -u` 或 `git branch -u`，监视会结束
* **您尚未推送的提交**：它们可能会随 Claude 的下一次推送一起推送上去
* **您未提交的工作**：Claude 可能会编辑您正在处理的文件，并可能提交它。当 Claude 在没有您消息的情况下行动时，Claude Code 会拒绝 Claude 执行以下命令：`git stash`、`git reset`、`git checkout`、`git add -A` 和 `git commit -a`

### 让 Claude 先与您确认

要在 Claude 处理 `/ship` 的审查发现或审查者的评论之前先查看它们，请在 Claude Code 输入框中发送消息说明。要在 Claude 修复任何内容之前查看发现，请在审查运行期间发送该消息。

以下示例要求 Claude 向您展示其发现，并在推送或回复之前先询问：

```text theme={null}
From now on, don't push code changes or reply to reviewers without asking me first. Show me the review findings and wait.
```

该消息是对 Claude 的请求，Claude Code 不会强制执行它。

### 为仓库中的监视保存指令

要告诉 Claude 如何照看某个仓库中的 Pull Request，请在仓库顶层文件夹中保存文件 `.claude/skills/steward/SKILL.md`。[PR Steward 读取同一个文件](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#save-pr-steward-instructions-for-everyone-in-a-repository)，因此一个文件可以同时服务于两者，该章节中有一个示例。

在监视期间，Claude 会读取您检出中的该文件副本，并将其作为指令遵循，例如哪些检查重要以及如何运行测试，但受以下限制：

* **安装和下载**：Claude 可能会跳过安装或下载内容的行，例如该示例中的 `npm install` 行
* **被阻止的命令**：该文件不会改变[监视期间 Claude Code 阻止的内容](#what-claude-code-blocks-during-a-watch)
* **对该文件的更改**：如果您在监视运行时自行添加或编辑该文件，监视会结束。关于其他人的哪些更改可能影响该文件，请参阅[监视在您的计算机上运行的代码](#code-that-a-watch-runs-on-your-computer)

## 检查、停止或重新启动监视

监视是 Claude 对您的 Pull Request 持续进行的工作，它在 `/ship` 的审查之后或您输入 `/steward` 时开始。它在您的会话中运行，因此您可以在 Claude Code 输入框中检查和停止它。

### 检查监视是否正在运行

在 Claude Code 输入框中输入 `/tasks`。审查完成后，在 Claude 未在工作时，正在运行的监视会以名称以 `/steward --tend` 开头的定时任务形式出现在列表中。

### 哪些情况会结束监视

监视结束后，请查找以 `Stopped watching` 开头的行。[监视已停止](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting.md#a-watch-stopped)列出了它给出的原因。

监视会在以下等情况下结束：

| 位置 | 结束监视的情况 |
| :- | :- |
| GitHub Pull Request 上 | 它被合并或关闭、[PR Steward](https://code.claude.com/docs/zh-CN/pr-steward.md) 为其添加了标签，或已过去七天 |
| 其他托管平台上 | 您的分支已从其远程仓库中消失，或不再为其基准分支添加任何内容，或已过去七天。在 Claude Code 能够读取您的 Merge Request 或 Pull Request 的情况下，监视也会在其被合并或关闭时结束。否则，如果您在未合并的情况下将其关闭，[监视可能会继续进行](#checks-and-comments-on-gitlab-bitbucket-and-azure-devops) |
| 键盘操作 | 您在输入框中或在 Claude 工作时按下 `Esc` 或 `Ctrl+C`。按一次即可结束监视，且没有确认 |
| 您的检出中 | 您检出了另一个分支，或更改了 git 的设置 |
| 会话中 | 会话结束、您开始另一个监视或带任务的 `/loop`、您运行了 `/clear`、`/resume` 或 `/branch`，或者在 Claude 下次检查 Pull Request 时会话处于计划模式。在 v2.1.331 之前，监视在计划模式下会继续进行，并在您将会话发送到后台时结束 |
| 失败之后 | 错误、被拒绝的权限提示或用量限制可能导致其提前结束 |

### 停止或重新启动监视

要结束监视，请在运行该监视的会话中，于 Claude Code 输入框中输入 `/ship stop`。如果输入时 Claude 正在工作，监视会在当前轮次结束时结束，因此 Claude 在该轮次中推送或发布的内容仍会到达 Pull Request。要同时停止该轮次，请按 `Esc`。

要重新开始监视，请输入 [`/steward`](#start-a-watch-with-%2Fsteward)。在 GitHub 以外的托管平台上，请加上您指定过的任何基准分支，例如 `/steward origin/main`。

## 决定在哪里运行监视

监视是 Claude 对您的 Pull Request 持续进行的工作，它在 `/ship` 的审查之后或您输入 `/steward` 时开始。Claude 使用您的 `gh` 和 `git` 登录身份工作。在让监视无人值守运行之前，请检查 Claude Code 会阻止什么、哪些代码可能在您的计算机上运行，以及哪些情况仍可能导致合并。

### 监视期间 Claude Code 阻止的内容

当 Claude 在没有您消息的情况下行动时，Claude Code 会阻止用于合并 Pull Request、批准 Pull Request、启用自动合并或强制推送的 `gh` 和 `git` 命令。此时，Claude Code 还会将 Claude 的 `gh` 和 `git` 命令限制在一个简短的列表中，并拒绝监视不需要的工具，例如连接器、`WebFetch` 和 `WebSearch`。要在 Claude Code 中查看这些限制，请在输入框中输入 `/ship terms`。

> [!WARNING]
>
> 监视的限制并不能涵盖所有情况。监视使用您的 `gh` 和 `git` 登录身份，因此如果您仓库的分支规则允许您在无需他人审批的情况下合并 Pull Request，它就可能合并该 Pull Request。请参阅[监视何时可能导致合并](#when-a-watch-can-lead-to-a-merge)。会话中的某项设置（例如权限规则或 hook）可能会[让命令绕过这些限制](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting.md#messages-during-a-watch)。

### 监视在您的计算机上运行的代码

在监视期间，来自以下每个来源的代码都可能使用您的凭据在您的计算机上运行：

| 来源 | 会发生什么 |
| :- | :- |
| 基础分支 | 为了解决冲突或使您的分支保持最新，Claude 会将基础分支合并到您的分支中，包括会自行运行的脚本 |
| 其他人推送到您分支的内容 | Claude 被指示将其合并进来 |
| 来自您分支的程序，例如您的测试 | Claude 会运行它们。它们可以使用您的凭据进行合并或强制推送。Claude Code 只会在会话的权限模式下检查启动它们的命令 |
| 您的 Claude Code hook | 它们会在不询问的情况下运行，并可能运行 Claude 编辑过的文件 |
| [`.claude/skills/steward/SKILL.md`](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests#save-pr-steward-instructions-for-everyone-in-a-repository) | 如果您的检出中有该文件，Claude 会遵循它。任何能够合并到该仓库的人都可以写入该文件 |

如果 Claude Code 发现合并基础分支或其他人推送的内容会更改其设置、`.mcp.json` 或 git 的配置文件，Claude 会将该合并留给您处理。

### 监视何时可能导致合并

以下等情况可能导致合并：

| 情况 | 会发生什么 |
| :- | :- |
| Pull Request 已启用自动合并 | 一旦检查和审查允许，托管平台就会合并。Claude 在监视期间的操作（例如推送修复）可能促成这一点。要在 GitHub 上关闭它，请在 shell 中运行 `gh pr merge --disable-auto <number>` |
| 在终端会话中，您在 Claude 空闲时输入消息 | 在 Claude 回复您的消息期间，[Claude Code 的限制](#what-claude-code-blocks-during-a-watch)处于关闭状态。Claude 处理在此期间到达的后台任务结果时，这些限制也处于关闭状态。在自动模式下，Claude 随后可能被允许进行合并 |
| 在 v2.1.331 之前，您在审查或监视期间将在终端中启动的会话发送到后台，例如按下 `←` | 监视结束。正在进行的工作可能会在那里继续进行，且不受这些限制。要结束该工作，请[停止该会话](https://code.claude.com/docs/zh-CN/agent-view.md#manage-sessions-from-the-shell) |

### 适用于 /ship 和 /steward 的权限规则和 hook

Claude Code 会自行在[沙箱之外](https://code.claude.com/docs/zh-CN/sandboxing.md#what-runs-outside-the-sandbox)运行 `/ship` 自身的提交、推送和 Pull Request 命令。在监视期间，当 Claude 在没有您消息的情况下行动时，它会以 `git -c core.hooksPath=/dev/null push` 的形式编写其 git fetch、合并、提交和推送命令。

无论规则和 hook 位于您自己的设置中还是托管设置中，它们都按以下方式生效：

| 设置 | 对于 `/ship` 自身的提交和推送 | 监视期间 |
| :- | :- | :- |
| 诸如 `Bash(git push *)` 的[拒绝规则](https://code.claude.com/docs/zh-CN/permissions.md#manage-permissions) | `/ship` 将这些步骤留给您处理 | 不会开始监视 |
| 对 `Bash(git push *)` 进行询问的规则 | Claude Code 不会参考它 | 它[不匹配](https://code.claude.com/docs/zh-CN/permissions.md#bash-rule-limits)该形式，因此不会增加权限提示 |
| 对 `Bash(git *)` 进行询问的规则 | Claude Code 不会参考它 | 它会匹配，监视会在每个权限提示处等待，包括在自动模式下 |
| 自动模式的检查，以及诸如 `PreToolUse` 的工具 hook | 它们不会运行 | 它们不适用于 Claude Code 在沙箱之外自行运行的 `gh` 和 `git` 命令，例如读取 Pull Request 的命令 |
| 仓库 hooks 文件夹中的 Git 钩子 | 它们会运行 | 在这些轮次中，它们不会针对 Claude 的 fetch、合并、提交和推送运行 |

## 为您的组织关闭 /ship 和 /steward

要关闭这两个命令，请在[托管设置](https://code.claude.com/docs/zh-CN/managed-settings.md)中的 [`skillOverrides`](https://code.claude.com/docs/zh-CN/skills#override-skill-visibility-from-settings) 下设置这些键。`ship-local` 是 `/ship` 背后的 skill。

以下示例同时关闭两者：

```json theme={null}
{
  "skillOverrides": {
    "ship-local": "off",
    "steward": "off"
  }
}
```

您也可以将 [`CLAUDE_CODE_DISABLE_SHIP`](https://code.claude.com/docs/zh-CN/env-vars#variables) 设置为 `1`，例如在托管设置的 `env` 下设置。需要 Claude Code v2.1.331 或更高版本。

要阻止 `/ship` 自身的推送，请在托管设置中添加诸如 `Bash(git push *)` 的[拒绝规则](https://code.claude.com/docs/zh-CN/permissions.md#manage-permissions)。有了该规则，不会开始任何监视。

## /ship 和 /steward 的参数

在 Claude Code 输入框中，于 `/ship` 或 `/steward` 之后输入参数。要组合多个参数，请将 effort 级别放在最前面，例如 `/ship low 41 +my-lint-bot`。

| 您输入 | 命令 | 作用 |
| :- | :- | :- |
| `low`、`medium`、`high`、`xhigh` 或 `max` | `/ship` | 以该 effort 级别请求审查。默认值为 `high` |
| `41`，或 Pull Request 的 URL | `/ship`、`/steward` | 在 GitHub 上，处理该 Pull Request。对于监视，它必须是您自己的，并且其分支已检出 |
| `+botname` | `/ship`、`/steward` | 在 GitHub 上，同时读取该机器人的评论。重新开始监视时需再次输入 |
| `origin/main`，或其他远程及分支 | `/ship`、`/steward` | 在[GitHub 以外的托管平台](#watch-a-branch-on-another-host)上，指定您的更改要合入的分支。需要 Claude Code v2.1.331 或更高版本 |
| `stop` | `/ship`、`/steward` | 结束监视 |
| `terms` | `/ship`、`/steward` | 显示 Claude 在监视期间可以做什么以及受到哪些限制。在[其他托管平台](#watch-a-branch-on-another-host)上，请在监视运行期间输入 |

## 相关资源

* [排查 `/ship` 和 `/steward` 的问题](https://code.claude.com/docs/zh-CN/ship-command-troubleshooting)：这些命令输出的消息及应对方法
* [在本地审查代码](https://code.claude.com/docs/zh-CN/code-review-local.md)：`/ship` 运行的审查
* [权限模式](https://code.claude.com/docs/zh-CN/permission-modes.md)：自动模式和其他权限模式
