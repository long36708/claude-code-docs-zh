> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# PR Steward

> 让 PR Steward 向您的 GitHub Pull Request 推送修复，处理失败的检查、合并冲突和审查反馈

PR Steward 是一个处理您团队中处于打开状态的 GitHub Pull Request 的 Agent。它会推送提交来处理审查反馈，并尝试修复失败的检查和解决合并冲突。启用后，您可以打开一个 Pull Request 并开始下一项更改，同时由 PR Steward 处理随后出现的失败检查、冲突和审查评论。该功能目前在 Team 和 Enterprise 计划中处于公测阶段。

> [!TIP]
>
> 要启用 PR Steward，请按照[设置 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-setup.md) 进行操作，这需要您在 Claude 组织中拥有 Owner 角色。

PR Steward 会在 GitHub 上发现问题，在云端会话中进行处理，并在 Pull Request 上回报结果：

![PR Steward 示意图：失败的检查、审查评论或合并冲突会交由云端会话中的 PR Steward 处理，其提交和回复会显示在 GitHub 上。](https://mintcdn.com/claude-code/sMH7RDpTvyWlAkrg/images/pr-steward-flow.svg?fit=max\&auto=format\&n=sMH7RDpTvyWlAkrg\&q=85\&s=4a1de46fbee43c2a00ef78c3dca36e59)

要在某个 Pull Request 上停止 PR Steward，请移除 `claude-pr-steward-watching` 标签或[评论 `@claude stop`](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests#start-or-stop-pr-steward-on-one-pull-request)。

Steward 的设计不会批准或合并 Pull Request。请使用分支保护来要求在任何内容合并之前获得您所需的审批。请参阅[审批和合并方面的限制](https://code.claude.com/docs/zh-CN/pr-steward-security.md#limits-on-approvals-and-merges)。

## PR Steward 在 Pull Request 上执行的操作

当 PR Steward 监视某个 Pull Request 时，它会推送提交并回复评论，而不会事先询问，并且该 Pull Request 会带有 `claude-pr-steward-watching` 标签。请参阅 [PR Steward 向 Pull Request 添加的内容](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#what-pr-steward-adds-to-a-pull-request)。

### PR Steward 何时开始

在 Owner 已将 PR Steward 设置为 **On** 的仓库中，当您打开 Pull Request、重新打开 Pull Request 或将草稿标记为可供审查时，PR Steward 就会开始工作。对于它此前尚未开始处理的打开状态的 Pull Request，它也会在下一次推送时开始工作。

PR Steward 会跳过草稿、来自 fork 的 Pull Request、由 bot 打开的 Pull Request，以及对仓库没有写入权限的作者。Owner 可以[更改仓库的设置](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#choose-when-pr-steward-starts-in-each-repository)，使 PR Steward 也处理草稿，或者通常仅在有人请求时才开始工作。

### PR Steward 修复的内容

PR Steward 处理失败的检查、合并冲突、审查反馈以及 Claude Approvals 检查：

* **失败的检查**：当原因位于 Pull Request 所涉及的代码中时，尝试进行修复
* **合并冲突**：尝试将基础分支合并进来并解决冲突
* **人员的审查评论**：尝试进行小而具体的更改并回复该评论。当请求较大或涉及设计时，可以只回复而不推送
* **审查 bot 的发现**：处理被 [Code Review](https://code.claude.com/docs/zh-CN/code-review.md) 标记为 **Important** 的发现。对于其他 bot，处理 lint 发现和 bug
* **[Claude Approvals](https://code.claude.com/docs/zh-CN/approvals.md) 检查**：如果该检查在等待可通过提交修复的内容（例如未解决的 Code Review 发现），则处理该内容。如果该检查在等待人员的审批，则不做处理。PR Steward 推送后，该检查可能会[再次请求审批](https://code.claude.com/docs/zh-CN/approvals-policy.md#earlier-approvals-after-a-new-push)

### 停止或限制 PR Steward

要在某个 Pull Request 上停止 PR Steward，请移除 `claude-pr-steward-watching` 标签或[评论 `@claude stop`](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests#start-or-stop-pr-steward-on-one-pull-request)。要在您之后打开的 Pull Request 上关闭或限制 PR Steward，请前往 **Settings > Code** 或 **Settings > Claude Code** 中的 [**PR Steward**](https://claude.ai/settings/claude-code#pr-steward)。

## 判断 PR Steward 是否适合您的团队

PR Steward 是 [Claude Ship](https://code.claude.com/docs/zh-CN/claude-ship.md) Agent 之一，与 Code Review 和 Claude Approvals 并列。在启用之前，请将其与 auto-fix 和 `/steward` 进行比较，确认您的组织符合条件，并了解其计费方式。

### PR Steward 与 auto-fix 和 /steward 的区别

[Auto-fix](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md#auto-fix-pull-requests) 和 [`/steward` 命令](https://code.claude.com/docs/zh-CN/ship-command#start-a-watch-with-%2Fsteward)也会处理失败的检查和审查评论。它们在您自己的会话中运行，以您的身份执行操作，并计入您自己的用量。PR Steward 以 [Claude GitHub App](https://github.com/apps/claude) 的身份执行操作，并向您的组织计费。

如果您请求对 PR Steward 正在监视的 Pull Request 执行 auto-fix，该请求会失败。请参阅 [Auto-fix 不可用](https://code.claude.com/docs/zh-CN/pr-steward-troubleshooting.md#auto-fix-is-unavailable)。

### 可用性

PR Steward 适用于 github.com 上的仓库，或在端口 443 上连接的 [GitHub Enterprise Server](https://code.claude.com/docs/zh-CN/github-enterprise-server.md) 实例上的仓库。您需要一个使用 [Team 或 Enterprise 计划](https://code.claude.com/docs/zh-CN/feature-availability.md#features-that-require-a-claude-subscription)的 Claude 组织。

已启用 [Zero Data Retention](https://code.claude.com/docs/zh-CN/zero-data-retention.md)、已应用 [HIPAA 配置](https://code.claude.com/docs/zh-CN/hipaa-setup.md)或使用客户管理的加密密钥的组织无法使用 PR Steward。

### PR Steward 的费用

PR Steward 的用量按 token 计费，从您组织的[使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans)中扣除。在您[设置支出限额](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#set-pr-steward’s-spend-limits)之前，它没有自己的支出限额，设置限额需要 Owner 角色。

## 后续步骤

* [设置 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-setup.md)，需要 Owner 角色
* 在您的 Pull Request 上[使用 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md)
* [PR Steward 可以访问的内容](https://code.claude.com/docs/zh-CN/pr-steward-security.md)
* [PR Steward 故障排除](https://code.claude.com/docs/zh-CN/pr-steward-troubleshooting.md)
