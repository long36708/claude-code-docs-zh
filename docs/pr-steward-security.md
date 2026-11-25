> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# PR Steward 可以访问哪些内容

> 查看 PR Steward 使用的 GitHub 身份、权限和仓库，其会话运行的位置，以及限制批准和合并的因素

[PR Steward](https://code.claude.com/docs/zh-CN/pr-steward.md) 是一个处理您团队中未关闭的 GitHub Pull Request 的 Agent。它以 [Claude GitHub App](https://github.com/apps/claude) 的身份在 GitHub 上执行操作，并在[云端会话](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md)中处理每个 Pull Request。在 GitHub Enterprise Server 上，它以[您的组织创建的应用](https://code.claude.com/docs/zh-CN/github-enterprise-server.md#admin-setup)的身份执行操作。

此处所述的访问权限和限制适用于在 Anthropic 托管环境中运行的会话。有关自托管环境，请参阅[在自托管环境中运行 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#run-pr-steward-in-a-self-hosted-environment)。有关会话数据的处理方式，请参阅[数据使用](https://code.claude.com/docs/zh-CN/data-usage.md)。

## PR Steward 的身份和权限

先查看 PR Steward 会话在 GitHub 上可以访问哪些内容，再查看会话本身是如何设置的。

### PR Steward 的 GitHub 访问权限

PR Steward 会话以其 GitHub App 的身份，使用以下仓库、权限和令牌在 GitHub 上执行操作：

| 主题 | PR Steward 使用的内容 |
| :- | :- |
| 仓库 | 会话从 Pull Request 所在的仓库开始，并对其进行读写。在 github.com 上，GitHub 搜索可以返回您的组织为 PR Steward 添加的其他 github.com 仓库中的结果。会话可以获取 github.com 上的公开仓库 |
| 权限 | 对内容、Pull Request 和 issue 的读写权限。如果您安装的应用授予了相应权限，它还拥有对检查和状态的读取权限以及对 Actions 的写入权限。在 github.com 上，安装还可以授予对工作流的写入权限，这使 PR Steward 能够推送对工作流文件的更改。请参阅该应用的[权限列表](https://code.claude.com/docs/zh-CN/github-actions.md#github-app-permissions) |
| 令牌 | 应用的令牌保存在会话的沙箱之外。默认情况下，会话的 `gh` 和 `git` 请求会经过 Anthropic 的服务，由该服务添加令牌 |

### PR Steward 的会话环境

查看 PR Steward 会话在哪里运行、如何处理权限提示，以及读取哪些评论和指令文件：

| 主题 | 会话的工作方式 |
| :- | :- |
| 代码运行位置 | 在[云端会话](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md)中。请参阅 [PR Steward 运行的环境](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#choose-the-environment-pr-steward-runs-in) |
| 权限提示 | 会话以[自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode)启动，权限请求会被拒绝 |
| 评论 | Anthropic 的服务会向会话发送 Pull Request 作者、拥有仓库写入权限的人员以及机器人的评论和审查。当会话直接读取 Pull Request 时，它也可以看到其他任何人在其中写下的内容 |
| 指令文件 | PR Steward [从默认分支读取这些文件](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#save-pr-steward-instructions-for-everyone-in-a-repository) |

## 对批准和合并的限制

Steward 的设计是不批准也不合并 Pull Request。请使用分支保护，在任何内容合并之前要求获得您所需的批准。

在 Anthropic 托管环境中，对于 PR Steward 会话以其 GitHub App 身份发出的请求，Anthropic 的服务会拒绝其中一些，而将其他请求交由 GitHub 的分支规则处理。

### Anthropic 的服务拒绝的请求

对于以其 GitHub App 身份执行操作的 PR Steward 会话，Anthropic 的服务会拒绝以下请求：

* 批准 Pull Request
* 更改分支保护、规则集或状态，或写入检查运行
* 删除分支
* 推送标签

### 交由 GitHub 分支规则处理的请求

Anthropic 的服务允许诸如以下的请求。在设有规则的分支上，由 GitHub 的分支规则控制这些请求：

* **合并、启用自动合并以及强制推送**：在 GitHub 的分支规则生效之前，会对 Claude 运行的命令执行两项检查
* **推送到 Pull Request 分支以外的分支**：会话可以推送到该仓库中 GitHub 允许 Claude GitHub App 推送的任何分支
* **驳回审查**：要限制谁可以执行此操作，请使用 GitHub 的分支规则，例如经典分支保护规则中的 **Restrict who can dismiss pull request reviews**

对于执行合并、启用自动合并或强制推送的命令，会运行两项检查。权限规则会拒绝执行这些操作的常用命令。[自动模式的检查](https://code.claude.com/docs/zh-CN/permission-modes.md#how-auto-mode-evaluates-actions)会对没有任何规则匹配的命令进行判断。规则匹配的是命令的文本，而检查依赖的是模型的判断，因此以其他方式编写的命令可能会同时通过这两项检查。

要让 GitHub 对 PR Steward 推送的内容强制执行审查，请[使用分支规则要求审查](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#require-reviews-with-branch-rules)，并确保 Claude GitHub App 不在任何规则集的绕过列表中。在经典分支保护规则中，确保它不在 **Allow specified actors to bypass required pull requests** 列表中。

## 相关资源

* [设置 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-setup.md)
* [PR Steward](https://code.claude.com/docs/zh-CN/pr-steward.md)
* [数据使用](https://code.claude.com/docs/zh-CN/data-usage.md)
