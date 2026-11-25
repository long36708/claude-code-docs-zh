> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Claude Approvals

> 通过策略文件和 Pull Request 检查，设置 GitHub 仓库中某个路径需要多少人审批，以及 Claude 可以在哪些地方审批

Claude Approvals 是 GitHub Pull Request 上的一项检查，用于报告还有谁需要审批。它遵循一个[策略](https://code.claude.com/docs/zh-CN/approvals-policy.md)，即仓库中的一个文件，该文件可以设置对某个路径的更改需要多少人审批，以及 Claude 可以在哪些地方审批。

> [!TIP]
>
> 要在仓库中启用 Claude Approvals，请按照[设置 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md) 操作。

为了评估 Pull Request，Approvals 会读取策略、找到适用的规则并统计审批数：

![Claude Approvals 如何评估 Pull Request 的示意图。Approvals 读取默认分支上的策略，找到适用的规则并取其中最严格的一条，然后统计审批数。Pull Request 上会出现一项检查和一条评论。如果策略允许，还会出现一条批准审查。](https://mintcdn.com/claude-code/sMH7RDpTvyWlAkrg/images/approvals-flow.svg?fit=max\&auto=format\&n=sMH7RDpTvyWlAkrg\&q=85\&s=29bc7cd0a0a49c1986cab5afdf57ca3b)

使用 Approvals 可以设置一项更改需要多少人审批：高风险更改需要更多人，常规更改需要更少人。例如，计费代码可以要求两个人审批，而测试夹具在 [Code Review](https://code.claude.com/docs/zh-CN/code-review.md) 通过后只需要一个人审批。

> [!NOTE]
>
> 有关 Claude Code 会话中的权限提示，请参阅[配置权限](https://code.claude.com/docs/zh-CN/permissions.md)。

## 在 Pull Request 上找到 Claude Approvals

Approvals 会在其评估的 Pull Request 上添加一项名为 **Claude Approvals** 的检查。检查的标题会说明仍需满足的条件，例如 **Waiting on human review from any person (1/2)**。以 **(observe mode)** 开头的标题表示检查运行时策略未被强制执行。在[观察模式](#set-up-claude-approvals)下，检查会显示您的策略将作出的决定。

Claude 还会发布一条说明原因的评论。在策略允许的情况下，Claude 可以发布一条批准审查：

* **在由 Claude GitHub App 报告检查的 github.com 仓库中**：该审查来自 `claude[bot]`
* **在由 `claude[bot]` 创建的 Pull Request 上**：添加 [Claude Approvals GitHub App](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#pull-requests-that-claude-opens) 后，该审查来自 `claude-approvals[bot]`

有关常见标题以及针对每种标题该如何处理，请参阅[查找您的审批状态及后续操作](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#find-your-approval-status-and-what-to-do-next)。如果 Pull Request 上没有检查，请参阅[未出现 Claude Approvals 检查](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md#no-claude-approvals-check-appeared)。

## 示例策略

策略以规则列表的形式写在 `.claude/approvals.yml` 中。

此策略只有一条规则，要求每项更改都需要两个人审批：

```yaml theme={null}
# .claude/approvals.yml
version: 1
mode: observe
rules:
  - purpose: "A change takes two people, and Claude doesn't approve."
    paths: ["**"]
    verdict: { claude: comments, seats: 2 }
    advisory: [claude/code_review]
```

策略的每个部分各设置一项内容：

* **`mode: observe`**：以[观察模式](#set-up-claude-approvals)启动策略
* **`purpose`**：说明该规则存在的原因
* **`paths`**：指定规则匹配的文件，此处为所有文件
* **`seats: 2`**：要求两个人审批，其中一人[可以是作者](https://code.claude.com/docs/zh-CN/approvals-policy.md#whether-a-pull-request’s-author-counts)
* **`claude: comments`**：表示 Claude 不进行批准
* **`advisory`**：[保留自动 Code Review](#keep-automatic-code-reviews)

您还可以为规则指定条件，条件可以读取有关 Pull Request 的事实以及分类器的回答：

* **有关 Pull Request 的[事实](https://code.claude.com/docs/zh-CN/approvals-policy.md#if)**：例如更改了多少行、由谁创建以及目标分支是哪个
* **[分类器](https://code.claude.com/docs/zh-CN/approvals-classifiers.md)的回答**：模型对某个问题的判断，例如更改的风险有多大

条件无法读取 CI 结果，因此请在 GitHub 中将 CI 保留为单独的必需检查。

## 启用 Approvals 之前

在向仓库添加策略文件之前，请确认您的组织符合条件，以及谁可以启用或关闭 Approvals。然后了解 Approvals 会对合并、Code Review 和计费带来哪些变化。

### 可用性

Approvals 目前作为 [Claude Ship](https://code.claude.com/docs/zh-CN/claude-ship.md) 的一部分，面向 [Team 和 Enterprise](https://code.claude.com/docs/zh-CN/feature-availability.md#features-that-require-a-claude-subscription) 计划的 Claude 组织公开测试。它适用于 github.com，包括 GitHub Enterprise Cloud。合规设置会限制谁可以使用它：

* **不可用**：启用了 [Zero Data Retention](https://code.claude.com/docs/zh-CN/zero-data-retention.md)、应用了 [HIPAA 配置](https://code.claude.com/docs/zh-CN/hipaa-setup.md)或使用客户管理的加密密钥（CMEK）时
* **无法启动 Approvals**：使用 FERPA 或 Access Transparency 设置时

### 谁可以启用或关闭 Approvals

满足[前提条件](https://code.claude.com/docs/zh-CN/approvals-setup.md#prerequisites)后，任何能够将策略文件合并到默认分支的人都可以启用 Approvals。Claude 组织的 Owner 可以为整个组织[关闭 Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md#turn-off-approvals)，此后即使合并了策略文件，它也会保持关闭。

### 未经人工审查的合并

当除作者外没有任何人查看过某个 Pull Request 时，它也可能被合并。请检查您的策略和 GitHub 规则是否存在以下情况：

* **检查不是必需的**：只有在 GitHub 中将 **Claude Approvals** 检查设为必需检查时，它才会阻止合并，并且在观察模式下通常不会阻止
* **策略不要求其他人审批**：作者可以占用一个名额，因此在 `seats: 1` 的情况下，无需其他人查看。在 `seats: 0` 的情况下，由机器人编写的 Pull Request 可以在没有任何人查看的情况下合并
* **GitHub 计入 Claude 的批准**：当 GitHub 分支规则要求一个批准时，Claude 的批准审查可以充当该批准
* **Claude 在没有审查的情况下批准**：带有 `claude: approves` 的规则本身不会启动 Code Review，因此 Claude 可能会批准一项未经任何审查的更改。要先获得审查，请[仅在 Code Review 通过时让 Claude 批准](https://code.claude.com/docs/zh-CN/approvals-setup.md#let-claude-approve-when-code-review-passes)

有关其他情况，以及要求非作者人员审批的规则，请参阅[未经人工审查的合并](https://code.claude.com/docs/zh-CN/approvals-github-rules.md#merges-without-human-review)。

### 保留自动 Code Review

[Code Review](https://code.claude.com/docs/zh-CN/code-review.md) 是另一个 Claude Ship Agent，可以自动审查每个 Pull Request。添加策略文件可能会停止这些审查。当 Pull Request 更改的每个文件都位于某个策略文件的覆盖范围内时，自动审查通常会停止，除非针对这些文件的某条规则指定了 `claude/code_review`。这在[观察模式](#set-up-claude-approvals)下和策略被强制执行时都成立。

要保留自动审查，请[在规则中列出 `claude/code_review`](https://code.claude.com/docs/zh-CN/approvals-policy#advisory)，就像[示例策略](#example-policy)在 `advisory` 下所做的那样。

### Approvals 的计费

对于策略启动的 Code Review 和分类器运行，Approvals 会按 token 从您组织的[使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans)中计费。请参阅[哪些操作会产生费用](https://code.claude.com/docs/zh-CN/approvals-setup.md#what-approvals-costs)。

## 设置 Claude Approvals

要设置 Approvals，请先以观察模式启动策略，在此模式下，**Claude Approvals** 检查会显示您的策略将作出的决定。在观察模式下，审查和分类器运行同样会计费。有关入门策略和具体步骤，请参阅[设置 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md)。

## 相关资源

* [在 Pull Request 上使用 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md)：检查标题和评论命令
* [Approvals 与您的 GitHub 规则](https://code.claude.com/docs/zh-CN/approvals-github-rules.md)：必需审查、CODEOWNERS 和合并队列
* [策略文件参考](https://code.claude.com/docs/zh-CN/approvals-policy.md)：键和示例策略
* [Claude Approvals 故障排除](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md)：未出现检查或策略文件被拒绝时
* [配置权限](https://code.claude.com/docs/zh-CN/permissions.md)：Claude Code 中的权限提示
